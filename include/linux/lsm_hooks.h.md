## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem ده؟

الـ file ده جزء من **Linux Security Module (LSM) Framework** — الـ subsystem اللي بيسمح لـ Linux إنه يشتغل مع أكتر من نظام أمان في نفس الوقت بدون ما يعدّل الـ kernel core.

المسؤولين عنه في `MAINTAINERS`:
- **Paul Moore** و **James Morris** و **Serge Hallyn**
- الـ mailing list: `linux-security-module@vger.kernel.org`

---

### الفكرة بالبساطة — قصة حارس البوابة

تخيّل إنك بتبني فندق ضخم (ده الـ Linux kernel). الفندق فيه أبواب كتير — باب الغرف، باب المطبخ، باب الخزينة. كل ما حد عايز يدخل أي مكان، لازم يمر على **حارس**.

المشكلة: فيه أنواع مختلفة من الحراسة:
- **SELinux**: حارس عسكري صارم جداً بيحكم على كل حاجة بـ labels.
- **AppArmor**: حارس بيشتغل بـ profiles — كل برنامج ليه قواعده.
- **Smack**: حارس بسيط بيعمل labels على الأشياء.
- **BPF LSM**: حارس مرن — أنت بنفسك بتكتب قواعده.

قبل الـ LSM Framework، كانت الـ kernel بتكتب الحراسة hardcoded جوّاها. يعني لو عايز تغيّر نظام الأمان، محتاج تعدّل الـ kernel نفسها — كارثة.

الـ LSM جه وقال: **"أنا هحط hooks في كل الأماكن الحساسة، وكل نظام أمان يسجّل نفسه فيّا، وأنا اللي هنادي عليهم."**

---

### ما الهدف من `lsm_hooks.h` تحديداً؟

الـ file ده هو **عقد التسجيل** بين الـ LSM Framework وأي نظام أمان عايز يشتغل جوّا الـ kernel. بيعرّف:

| البنية / الـ Macro | الغرض |
|---|---|
| `union security_list_options` | Union بيجمع كل الـ hook function pointers الممكنة |
| `struct lsm_static_call` | يمثّل static call واحد لـ hook معيّن لـ LSM واحد |
| `struct lsm_static_calls_table` | جدول كل الـ static calls لكل الـ hooks |
| `struct lsm_id` | اسم وID الـ LSM |
| `struct security_hook_list` | ربط الـ hook بالـ LSM اللي سجّله |
| `struct lsm_blob_sizes` | حجم الـ security data اللي كل LSM محتاجه يضمّه في الـ objects |
| `struct lsm_info` | وصف كامل لـ LSM — اسمه، ترتيبه، وكيف يتبدأ |
| `LSM_HOOK_INIT` | Macro لتسجيل hook بسهولة |
| `DEFINE_LSM` / `DEFINE_EARLY_LSM` | Macros لتعريف LSM وربطه بـ section خاص في الـ binary |
| `security_add_hooks()` | الـ function اللي أي LSM بيناديها عشان يسجّل الـ hooks بتاعته |

---

### قصة تخيّلية: إزاي SELinux بيسجّل نفسه؟

```
Boot time:
  ┌─────────────────────────────────────────┐
  │  kernel يلاقي .lsm_info.init section   │
  │  فيه: SELinux, AppArmor, BPF LSM, ...  │
  └────────────────┬────────────────────────┘
                   │
                   ▼
  ┌─────────────────────────────────────────┐
  │  LSM Framework ينادي selinux_init()    │
  │  SELinux ينادي security_add_hooks()    │
  │  ويقول: "أنا عندي callback لـ         │
  │  file_open, inode_permission, ..."     │
  └────────────────┬────────────────────────┘
                   │
                   ▼
  ┌─────────────────────────────────────────┐
  │  Framework يحط الـ callbacks في        │
  │  static_calls_table                    │
  │  (مرتبة من الآخر للأول في كل hook)    │
  └─────────────────────────────────────────┘

Run time — مثلاً process عايز يفتح file:
  kernel → security_file_open()
         → يجري static call
         → selinux_file_open()  ← SELinux يقرر
         → apparmor_file_open() ← AppArmor يقرر
         → لو أي واحد رفض: EACCES
```

---

### الحل الذكي: Static Calls بدل Function Pointers عادية

الـ **static calls** (`struct lsm_static_call`) جوهر التحسين. بدل ما كل مرة تعمل indirect call من جوز function pointer (بطيء + مش آمن من Spectre)، الـ kernel بتحوّل الـ call لـ direct jump بعد ما الـ LSMs اتبدأت.

الجدول `lsm_static_calls_table` بيتبني **بالمقلوب** — من آخر LSM لأوله — عشان لما الـ kernel تجري الـ calls، تبدأ من أول LSM مسجّل وتكمّل للآخر. ده بيخلي الـ overhead قريب من الصفر.

---

### الـ `lsm_blob_sizes` — مساحة الـ Security Data

كل object في الـ kernel (inode, file, cred, task...) محتاج يشيل **security data** خاصة بكل LSM. بدل ما كل LSM يخصص memory منفصل، الـ framework بيجمع الاحتياجات كلها في struct واحد `lsm_blob_sizes` ويخصص مساحة مدمجة واحدة.

```
cred struct:
 ┌────────────────┬──────────────┬───────────────┐
 │  kernel data  │ SELinux blob │ AppArmor blob │
 └────────────────┴──────────────┴───────────────┘
                 ↑ lbs_cred offset للـ SELinux
                                ↑ lbs_cred offset للـ AppArmor
```

---

### ليه ده مهم جداً؟

- **الأمان قابل للتوسيع** بدون تعديل الـ kernel core.
- **أكتر من LSM** ممكن يشتغلوا **معاً** في نفس الوقت (stacking).
- الـ kernel بتقدر تشتغل بـ SELinux + AppArmor + BPF LSM في نفس الوقت.
- الـ overhead ضئيل جداً بسبب الـ static calls.

---

### الـ Files المكوّنة للـ Subsystem

#### Core Headers
| الـ File | الدور |
|---|---|
| `include/linux/lsm_hooks.h` | **الـ file ده** — عقود التسجيل والـ structs |
| `include/linux/lsm_hook_defs.h` | تعريف كل الـ hooks بالـ LSM_HOOK macro (468 سطر) |
| `include/linux/security.h` | الواجهة العامة اللي الـ kernel تستخدمها |
| `include/uapi/linux/lsm.h` | الـ IDs والتعريفات اللي تتعرض لـ userspace |

#### Core Implementation
| الـ File | الدور |
|---|---|
| `security/security.c` | التنفيذ الفعلي لكل `security_*()` functions |
| `security/lsm_init.c` | Initialization الـ LSM framework |
| `security/lsm_audit.c` | Audit support |
| `security/lsm_syscalls.c` | الـ syscalls الخاصة بالـ LSM |

#### LSM Implementations (الحراس الفعليين)
| الـ File | الـ LSM |
|---|---|
| `security/selinux/` | SELinux — الأقوى والأكثر انتشاراً |
| `security/apparmor/lsm.c` | AppArmor — مستخدم في Ubuntu |
| `security/smack/` | Smack — للـ embedded systems |
| `security/landlock/` | Landlock — للـ unprivileged sandboxing |
| `security/bpf/hooks.c` | BPF LSM — قابل للبرمجة في runtime |
| `security/yama/` | Yama — يحمي من ptrace |
| `security/ipe/` | IPE — Integrity Policy Enforcement |
| `security/commoncap.c` | Linux capabilities — الـ base لكل شيء |

#### Testing
| الـ File | الدور |
|---|---|
| `tools/testing/selftests/lsm/` | الـ selftests الخاصة بالـ LSM |

---

### ملفات مهمة للقارئ يعرفها

- **`include/linux/lsm_hook_defs.h`**: قرّاه الأول — فيه كل الـ hooks بنمط `LSM_HOOK(ret, default, name, args)`. هو المصدر الحقيقي للـ hooks.
- **`security/security.c`**: هنا بتلاقي إزاي كل `security_file_open()` وأخواتها بتنادي الـ hooks المسجّلة.
- **`security/selinux/hooks.c`**: مثال واقعي على LSM كامل بيستخدم `DEFINE_LSM` و`LSM_HOOK_INIT`.
## Phase 2: شرح الـ Linux Security Module (LSM) Framework

---

### المشكلة اللي بيحلها الـ LSM

في بداية الـ Linux، الـ security model كان بسيط جداً: **DAC (Discretionary Access Control)** — يعني الـ owner بتاع الـ file هو اللي يقرر مين يقدر يقرأه. ده كان كافي للـ desktop، بس في بيئات زي الـ military، الـ banking، أو الـ embedded systems اللي فيها multi-tenant، الـ DAC مش بيكفي.

المشكلة الحقيقية: عايزين نضيف **MAC (Mandatory Access Control)** — نظام زي SELinux أو AppArmor — من غير ما نكسر الـ mainline kernel، ومن غير ما كل vendor يعمل fork منفصل.

**قبل LSM**: كل security vendor (NSA بـ SELinux، Immunix بـ AppArmor، إلخ) كان بيحتاج يعمل patch ضخم في الـ kernel source مباشرة. النتيجة: كل نسخة kernel فيها patches متعارضة، مش ممكن تشغل أكتر من نظام واحد في نفس الوقت.

**السؤال الجوهري**: إزاي نخلي الـ kernel يدعم أنظمة security مختلفة ومتعددة في نفس الوقت من غير ما الـ core kernel يعرف أي حاجة عن تفاصيلهم؟

---

### الحل: Hook-Based Pluggable Security

الـ LSM framework (تم تقديمه في kernel 2.6، سنة 2003) حل المشكلة بنفس الفكرة اللي بيستخدمها الـ VFS مع filesystems: **abstraction layer + hooks**.

الـ kernel بيحدد مجموعة من **security-sensitive checkpoints** في الكود — كل مرة حاجة حساسة بتحصل (فتح ملف، execve، bind socket، إلخ) — وبيوصل للـ LSM framework قبل ما يكمل العملية. الـ LSM بيسأل كل LSM مسجّل: "هل تسمح بده؟". لو أي واحد رفض، الـ syscall بترجع error.

---

### Analogy: بوابات الأمن في المطار

تخيل مطار كبير فيه رحلات كتير (= syscalls). المطار ده عنده gates متعددة (= LSM hooks). كل مسافر (= process) لازم يعدي من كل gate قبل ما يوصل للطيارة.

| الـ Analogy | الـ LSM Concept |
|---|---|
| المطار نفسه | الـ Linux Kernel |
| الـ gates بالترتيب | الـ security_hook_list array |
| كل جهاز X-ray في الـ gate | LSM واحد (SELinux، AppArmor، إلخ) |
| الأمتعة والجواز | الـ security blob المرفق بالـ inode/cred/task |
| بطاقة الصعود | الـ security context / label |
| ممنوع الصعود | `return -EACCES` |
| كل الأجهزة لازم توافق | all LSMs must return 0 to allow |
| رئيس الأمن الثابت | الـ Capabilities LSM (دايماً أول واحد) |
| ضابط الخروج الأخير | الـ Integrity LSM (دايماً آخر واحد) |

الـ analogy مش surface-level: زي الأجهزة في المطار، كل LSM بيشوف **نفس الـ object** (الـ struct inode مثلاً) بس بيقيّمه من منظور مختلف. SELinux بيتحقق من الـ label، AppArmor بيتحقق من الـ path policy، Capabilities بيتحقق من الـ privilege bits. لو أي واحد رفض = رحلة ملغاة.

---

### Big Picture Architecture

```
 ┌─────────────────────────────────────────────────────────┐
 │                    User Space                           │
 │  App: open("/etc/shadow", O_RDONLY)                     │
 └───────────────────┬─────────────────────────────────────┘
                     │  syscall
 ┌───────────────────▼─────────────────────────────────────┐
 │                  VFS Layer                              │
 │  do_filp_open() → ... → inode_permission()              │
 └───────────────────┬─────────────────────────────────────┘
                     │  calls security_inode_permission()
 ┌───────────────────▼─────────────────────────────────────┐
 │             security.c  (LSM Multiplexer)               │
 │                                                         │
 │  ┌─────────────────────────────────────────────────┐   │
 │  │         static_calls_table.inode_permission      │   │
 │  │   [slot0: capabilities_inode_permission]         │   │
 │  │   [slot1: selinux_inode_permission     ]         │   │
 │  │   [slot2: apparmor_inode_permission    ]         │   │
 │  │   [slot3: <empty>                      ]         │   │
 │  └──────────────┬──────────────────────────────────┘   │
 └──────────────────┼──────────────────────────────────────┘
                    │  calls each active slot in sequence
         ┌──────────┼──────────────┐
         ▼          ▼              ▼
  ┌────────────┐ ┌──────────┐ ┌──────────────┐
  │Capabilities│ │ SELinux  │ │  AppArmor    │
  │   LSM      │ │   LSM    │ │    LSM       │
  │(always 1st)│ │(MAC)     │ │(path-based)  │
  └────────────┘ └──────────┘ └──────────────┘
         │          │              │
         └──────────┴──────────────┘
               all must return 0
               else → -EACCES
```

---

### الـ Core Abstraction: الـ Hook Table + الـ Security Blob

الـ LSM framework بيقوم على فكرتين أساسيتين:

#### 1. الـ Hook Table (لمن يُنفَّذ؟)

الـ `lsm_hook_defs.h` بيعرّف كل الـ hooks بـ macro واحد:

```c
LSM_HOOK(RET_TYPE, DEFAULT_VALUE, hook_name, args...)
```

ده الـ macro بيتوسّع في أماكن مختلفة:
- في `union security_list_options`: بيعمل function pointer لكل hook
- في `struct lsm_static_calls_table`: بيعمل array من الـ static calls لكل hook

مثال: hook واحد زي `inode_permission`:
```c
// في lsm_hook_defs.h
LSM_HOOK(int, 0, inode_permission, struct inode *inode, int mask)

// يتوسع إلى:
// في union security_list_options:
int (*inode_permission)(struct inode *inode, int mask);

// في struct lsm_static_calls_table:
struct lsm_static_call inode_permission[MAX_LSM_COUNT];
```

#### 2. الـ Security Blob (أين يُخزَّن الـ state؟)

كل kernel object حساس (inode, cred, task, file, socket) عنده **blob** — مساحة مخصصة لكل LSM يخزن فيها بياناته الخاصة. الـ `struct lsm_blob_sizes` بيوصف حجم الـ blob اللي كل LSM محتاجه:

```c
struct lsm_blob_sizes {
    unsigned int lbs_cred;       /* حجم الـ blob في struct cred */
    unsigned int lbs_file;       /* حجم الـ blob في struct file */
    unsigned int lbs_inode;      /* حجم الـ blob في struct inode */
    unsigned int lbs_sock;       /* حجم الـ blob في struct sock */
    unsigned int lbs_superblock; /* حجم الـ blob في struct super_block */
    unsigned int lbs_task;       /* حجم الـ blob في struct task_struct */
    // ... إلخ
};
```

لما SELinux بيتسجل، بيقول "أنا محتاج X bytes في كل inode". لما AppArmor بيتسجل، بيقول "أنا محتاج Y bytes". الـ kernel بيجمعهم ويحجز مساحة كافية للاتنين معاً.

---

### الـ Key Structs وعلاقتها ببعض

```
┌─────────────────────────────────────────────────────────┐
│                    struct lsm_info                      │
│  .id ──────────────► struct lsm_id { name, id }         │
│  .order (FIRST/MUTABLE/LAST)                            │
│  .flags (EXCLUSIVE, LEGACY_MAJOR)                       │
│  .blobs ───────────► struct lsm_blob_sizes              │
│  .init() ← دي الـ function اللي بتشغل الـ LSM          │
└──────────────────────────┬──────────────────────────────┘
                           │  DEFINE_LSM() يضعها في
                           ▼  section ".lsm_info.init"
                    [linker collects all LSMs]
                           │
                           │  security_add_hooks() يتستدعى
                           ▼
┌─────────────────────────────────────────────────────────┐
│               struct security_hook_list                  │
│  .scalls ──────────► &static_calls_table.hook_name      │
│  .hook ─────────────► union security_list_options        │
│  .lsmid ────────────► struct lsm_id                     │
└──────────────────────────┬──────────────────────────────┘
                           │  يتكتب فيه
                           ▼
┌─────────────────────────────────────────────────────────┐
│           struct lsm_static_calls_table                  │
│                                                         │
│  .inode_permission[0] ─► struct lsm_static_call         │
│  .inode_permission[1]    {                              │
│  .inode_permission[2]      .key  → static_call_key      │
│  .file_open[0]             .trampoline → asm stub        │
│  .file_open[1]             .hl  → security_hook_list    │
│  .task_kill[0]             .active → static_key_false   │
│  ... (one array           }                             │
│       per hook)                                         │
└─────────────────────────────────────────────────────────┘
```

---

### الـ Static Call Mechanism (تحسين أداء مهم)

**ملاحظة مسبقة**: الـ **static calls** هي kernel mechanism مستقلة تحتاج تفهمها أول. الفكرة: بدل الـ indirect function call عبر pointer (اللي بيكلف branch misprediction penalty على الـ CPU)، الـ static call بيحجز mutable instruction في الـ text segment نفسه، فلما الـ LSM بيتسجّل، الـ kernel بيكتب الـ address مباشرة في الـ instruction stream.

في الـ LSM context:

```c
struct lsm_static_call {
    struct static_call_key *key;    /* الـ key اللي بيتحكم في الـ call site */
    void *trampoline;               /* الـ asm stub */
    struct security_hook_list *hl;  /* رجوع للـ hook اللي سجّل هنا */
    struct static_key_false *active;/* هل الـ slot ده فيه hook مسجّل؟ */
};
```

الـ `__randomize_layout` على كل struct ده يعني الـ kernel compile-time بيعمل shuffle للـ fields (KASLR for structs) — protection ضد heap spray attacks.

**الـ fill order**: لما LSMs بتتسجّل، الـ slots بتتملى من آخر لأول (backwards). الغرض: الـ entry point بيبقى dynamic — لو عندك 2 LSMs، الـ call بيبدأ من slot N-2 مش من slot 0، فبتوفر cost الـ empty slots.

```
   قبل التسجيل:          بعد تسجيل LSM واحد:    بعد تسجيل اتنين:
   [empty][empty][empty]  [empty][empty][selinux]  [empty][apparmor][selinux]
                                        ▲                   ▲
                                   entry point         entry point
                                   (jump here)         (jump here)
```

---

### ترتيب الـ LSMs: `enum lsm_order`

```c
enum lsm_order {
    LSM_ORDER_FIRST = -1,   /* Capabilities فقط */
    LSM_ORDER_MUTABLE = 0,  /* SELinux, AppArmor, Smack, إلخ */
    LSM_ORDER_LAST = 1,     /* Integrity (IMA) فقط */
};
```

- **Capabilities**: دايماً أول — لأنه الـ baseline، بيتحقق من الـ POSIX capabilities
- **MUTABLE LSMs**: بالترتيب اللي في `CONFIG_LSM=` في الـ kernel config
- **Integrity (IMA)**: دايماً آخر — لأنه بيعمل measurement بعد كل قرارات السماح

---

### الـ Hook Categories الرئيسية (من lsm_hook_defs.h)

| الفئة | عدد الـ Hooks تقريباً | أمثلة |
|---|---|---|
| Filesystem / Inode | ~40 | `inode_permission`, `inode_create`, `inode_setxattr` |
| File Operations | ~15 | `file_open`, `file_permission`, `mmap_file` |
| Process / Credentials | ~20 | `task_kill`, `cred_prepare`, `bprm_check_security` |
| Network | ~30 | `socket_connect`, `socket_bind`, `sk_alloc_security` |
| IPC | ~15 | `msg_queue_msgsnd`, `shm_shmat`, `sem_semop` |
| Keys | ~4 | `key_alloc`, `key_permission` |
| BPF | ~10 | `bpf_prog_load`, `bpf_map_create` |
| Audit | ~4 | `audit_rule_init`, `audit_rule_match` |
| Lockdown | ~1 | `locked_down` |
| io_uring | ~4 | `uring_override_creds`, `uring_cmd` |

---

### تسجيل LSM جديد: الخطوات الكاملة

```c
/* 1. تعريف الـ LSM ID */
static const struct lsm_id my_lsm_id = {
    .name = "mylsm",
    .id   = LSM_ID_MYLSM,  /* من uapi/linux/lsm.h */
};

/* 2. تعريف حجم الـ blobs المطلوبة */
static struct lsm_blob_sizes my_blob_sizes = {
    .lbs_inode = sizeof(struct my_inode_sec),
    .lbs_cred  = sizeof(struct my_cred_sec),
};

/* 3. تعريف الـ hooks */
static int my_inode_permission(struct inode *inode, int mask)
{
    struct my_inode_sec *isec = inode->i_security + my_blob_offset;
    /* check policy */
    return 0; /* allow */
}

/* 4. ربط الـ hooks بالـ table */
static struct security_hook_list my_hooks[] = {
    LSM_HOOK_INIT(inode_permission, my_inode_permission),
    LSM_HOOK_INIT(file_open, my_file_open),
};

/* 5. دالة الـ init */
static int __init my_lsm_init(void)
{
    security_add_hooks(my_hooks, ARRAY_SIZE(my_hooks), &my_lsm_id);
    return 0;
}

/* 6. تسجيل في الـ linker section */
DEFINE_LSM(mylsm) = {
    .id    = &my_lsm_id,
    .order = LSM_ORDER_MUTABLE,
    .blobs = &my_blob_sizes,
    .init  = my_lsm_init,
};
```

**الـ `DEFINE_LSM` macro** بيضع الـ `struct lsm_info` في section اسمها `.lsm_info.init`. الـ linker script بيجمع كل الـ sections دي مع بعض، والـ kernel عند الـ boot بيعمل iterate عليهم كلهم ويشغّلهم بالترتيب.

---

### الـ `LSM_HOOK_INIT` macro بالتفصيل

```c
#define LSM_HOOK_INIT(NAME, HOOK) \
    {                             \
        .scalls = static_calls_table.NAME,  /* pointer للـ array في الـ table */ \
        .hook   = { .NAME = HOOK }          /* الـ function pointer نفسه */      \
    }
```

لما `security_add_hooks()` بتتستدعى، بتاخد الـ `security_hook_list` دي وبتحط الـ function pointer في أول `lsm_static_call` slot فاضي في الـ `static_calls_table.NAME` array.

---

### الـ Security Context و `struct lsm_context`

```c
struct lsm_context {
    char *context; /* النص: "system_u:object_r:etc_t:s0" */
    u32   len;
    int   id;      /* id بتاع الـ LSM اللي أنشأ الـ context ده */
};
```

ده المكان اللي بيترجم فيه الـ security label من binary format لـ human-readable string. SELinux بيستخدمه لـ `/proc/PID/attr/current`. `struct lsm_prop` هو الـ container اللي بيجمع الـ secids من كل الـ LSMs:

```c
struct lsm_prop {
    struct lsm_prop_selinux  selinux;
    struct lsm_prop_smack    smack;
    struct lsm_prop_apparmor apparmor;
    struct lsm_prop_bpf      bpf;
};
```

---

### الـ `lsm_get_xattr_slot`: إدارة الـ xattr Slots

```c
static inline struct xattr *lsm_get_xattr_slot(struct xattr *xattrs,
                                                int *xattr_count)
{
    if (unlikely(!xattrs))
        return NULL;
    return &xattrs[(*xattr_count)++]; /* اعطني الـ slot الحالي وزوّد الـ counter */
}
```

لما LSM بيحتاج يخزن security label على inode جديد (مثلاً لما `inode_init_security` hook بيتستدعى أثناء file creation)، الـ `lbs_xattr_count` في الـ `lsm_blob_sizes` بتحدد كام slot xattr الـ LSM محتاجه. الـ framework بيحجز array، وكل LSM بياخد slot بـ `lsm_get_xattr_slot()` — pattern بسيط لتجنب conflicts.

---

### الـ LSM يملك إيه vs. بيفوّض إيه؟

| الـ LSM Framework يملك | بيفوّض للـ LSM Implementation |
|---|---|
| ترتيب الاستدعاء | قرار السماح أو الرفض |
| إدارة الـ static call slots | تعريف الـ policy |
| حساب حجم الـ blobs وتوزيعها | قراءة وكتابة الـ blob |
| الـ `security_*()` wrapper functions في `security.c` | الـ `lsm_*()` implementation functions |
| التأكد إن كل LSMs اتستدعت | معنى الـ security context |
| الـ boot ordering عبر الـ linker sections | كيفية تسجيل الـ audit events |
| `__ro_after_init` protection للـ table | الـ network labeling logic |

---

### ملاحظة على الـ `__ro_after_init`

```c
extern struct lsm_static_calls_table static_calls_table __ro_after_init;
```

الـ `static_calls_table` بعد ما الـ boot ينتهي وكل الـ LSMs تتسجّل، بتبقى **read-only**. ده يعني حتى لو attacker نجح في kernel RCE، مش هيقدر يعدّل الـ hook table ويضيف backdoor hook. ده واحد من أهم الـ hardening features في الـ LSM framework.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Flags, Enums, Config Options — Cheatsheet

#### LSM Flags (في `lsm_hooks.h`)

| Flag | Value | المعنى |
|------|-------|--------|
| `LSM_FLAG_LEGACY_MAJOR` | `BIT(0)` | الـ LSM ده legacy وله major stacking slot |
| `LSM_FLAG_EXCLUSIVE` | `BIT(1)` | بس LSM واحد ممكن يتحمل من النوع ده في نفس الوقت |

#### Enum: `lsm_order`

| القيمة | الرقم | الاستخدام |
|--------|-------|-----------|
| `LSM_ORDER_FIRST` | `-1` | بس لـ `capabilities` — بيتشغل أول حاجة دايمًا |
| `LSM_ORDER_MUTABLE` | `0` | الوضع العادي — الترتيب بيحدده `CONFIG_LSM` |
| `LSM_ORDER_LAST` | `1` | بس لـ `integrity` — بيتشغل آخر حاجة |

#### Enum: `lsm_event` (في `security.h`)

| القيمة | المعنى |
|--------|--------|
| `LSM_POLICY_CHANGE` | حصل تغيير في الـ policy |
| `LSM_STARTED_ALL` | كل الـ LSMs اتحملت وشغالة |

#### Enum: `lsm_integrity_type` (في `security.h`)

| القيمة | المعنى |
|--------|--------|
| `LSM_INT_DMVERITY_SIG_VALID` | توقيع dm-verity صح |
| `LSM_INT_DMVERITY_ROOTHASH` | dm-verity root hash |
| `LSM_INT_FSVERITY_BUILTINSIG_VALID` | توقيع fs-verity built-in صح |

#### Enum: `lockdown_reason` — أهم القيم

| القيمة | النوع | المعنى |
|--------|-------|--------|
| `LOCKDOWN_NONE` | — | مفيش lockdown |
| `LOCKDOWN_MODULE_SIGNATURE` | Integrity | تحقق من توقيع الـ module |
| `LOCKDOWN_KEXEC` | Integrity | منع kexec |
| `LOCKDOWN_HIBERNATION` | Integrity | منع Hibernation |
| `LOCKDOWN_INTEGRITY_MAX` | حد فاصل | ما فوقه confidentiality |
| `LOCKDOWN_KCORE` | Confidentiality | منع قراءة `/proc/kcore` |
| `LOCKDOWN_BPF_READ_KERNEL` | Confidentiality | منع BPF من قراءة kernel memory |
| `LOCKDOWN_CONFIDENTIALITY_MAX` | حد فاصل | نهاية enum |

#### LSM_SETID Flags (في `security.h`)

| Flag | Value | المعنى |
|------|-------|--------|
| `LSM_SETID_ID` | `1` | `setuid` أو `setgid` |
| `LSM_SETID_RE` | `2` | `setreuid` / `setregid` |
| `LSM_SETID_RES` | `4` | `setresuid` / `setresgid` |
| `LSM_SETID_FS` | `8` | `setfsuid` / `setfsgid` |

#### LSM_PRLIMIT Flags

| Flag | Value | المعنى |
|------|-------|--------|
| `LSM_PRLIMIT_READ` | `1` | عملية قراءة لـ rlimit |
| `LSM_PRLIMIT_WRITE` | `2` | عملية كتابة لـ rlimit |

#### LSM_UNSAFE Flags (bprm)

| Flag | Value | المعنى |
|------|-------|--------|
| `LSM_UNSAFE_SHARE` | `1` | بيشارك VM مع thread تاني |
| `LSM_UNSAFE_PTRACE` | `2` | تحت ptrace |
| `LSM_UNSAFE_NO_NEW_PRIVS` | `4` | `no_new_privs` set |

#### Macros مهمة

| Macro | المعنى |
|-------|--------|
| `LSM_RET_VOID` | قيمة default للـ hooks اللي بترجع `void` — تساوي `((void)0)` |
| `LSM_HOOK_INIT(NAME, HOOK)` | بيبني `security_hook_list` entry جاهزة للتسجيل |
| `DEFINE_LSM(lsm)` | بيعمل `lsm_info` struct في section `.lsm_info.init` |
| `DEFINE_EARLY_LSM(lsm)` | زي `DEFINE_LSM` بس في section `.early_lsm_info.init` |

---

### 1. الـ Structs المهمة

---

#### `union security_list_options`

**الغرض:** union بيحتوي على كل function pointers الممكنة للـ LSM hooks، مضاف ليها `lsm_func_addr` كـ generic void pointer.

**الطريقة:** بيُبنى بشكل تلقائي عن طريق X-macro من `lsm_hook_defs.h`:
```c
union security_list_options {
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) RET (*NAME)(__VA_ARGS__);
    #include "lsm_hook_defs.h"
    #undef LSM_HOOK
    void *lsm_func_addr; /* generic access لأي hook pointer */
};
```

**الأهمية:** ده القلب — كل hook في النظام ممكن يتخزن كـ function pointer هنا. الـ `union` معناه إن الـ `lsm_func_addr` field بيوصل لنفس الميموري اللي فيها أي function pointer.

---

#### `struct lsm_static_call`

**الغرض:** بيمثل **static call site واحد** لـ hook واحد لـ LSM واحد. بدل ما الكيرنل يعمل indirect call عن طريق function pointer (بطيء بسبب Spectre mitigations)، بيستخدم **static call** — branch instruction بيتعدل في الميموري وقت runtime.

```c
struct lsm_static_call {
    struct static_call_key *key;       /* المفتاح الخاص بالـ static call */
    void *trampoline;                   /* الـ trampoline code الخاص به */
    struct security_hook_list *hl;     /* مين صاحب الـ hook ده */
    struct static_key_false *active;   /* هل الـ hook ده فعّال؟ */
} __randomize_layout;
```

| Field | الوظيفة |
|-------|---------|
| `key` | بيُعرَّف بـ `STATIC_CALL_KEY` — بيستخدم لتعديل الـ branch destination |
| `trampoline` | الكود اللي بيتنفذ أول ما يتكال الـ static call |
| `hl` | pointer للـ `security_hook_list` اللي سجّل الـ hook ده |
| `active` | `static_key_false` — optimized branch اللي بيتجاهل الـ hook لو `false` |

**ملاحظة مهمة:** `__randomize_layout` معناها إن الـ kernel بيخلط ترتيب الـ fields في الميموري عشان يصعّب exploitation.

---

#### `struct lsm_static_calls_table`

**الغرض:** جدول عملاق بيحتوي على **array من `lsm_static_call`** لكل hook في النظام.

```c
struct lsm_static_calls_table {
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) \
        struct lsm_static_call NAME[MAX_LSM_COUNT];
    #include <linux/lsm_hook_defs.h>
    #undef LSM_HOOK
} __packed __randomize_layout;
```

مثلًا لو عندنا hook اسمه `file_open` وعندنا `MAX_LSM_COUNT = 8`:
- `static_calls_table.file_open[0]` → أول LSM سجّل `file_open`
- `static_calls_table.file_open[7]` → آخر LSM

**الترتيب:** الـ LSMs بتتحط من آخر لأول (backwards) — عشان الأول في الـ array يكون أول حاجة بتتنفذ.

**المتغير:** `extern struct lsm_static_calls_table static_calls_table __ro_after_init` — بيبقى read-only بعد init.

---

#### `struct lsm_id`

**الغرض:** بيعرّف هوية الـ LSM بشكل ثابت.

```c
struct lsm_id {
    const char *name;  /* اسم الـ LSM (مثلًا "selinux", "apparmor") */
    u64 id;            /* ID رقمي من uapi/linux/lsm.h */
};
```

**الاستخدام:** كل LSM بيعرّف `lsm_id` global static واحدة، وبيبعتها لـ `security_add_hooks()`.

---

#### `struct security_hook_list`

**الغرض:** بيمثل تسجيل hook واحد من LSM واحد. ده الـ "entry" اللي بيحطه الـ LSM في الـ framework.

```c
struct security_hook_list {
    struct lsm_static_call *scalls;        /* pointer لأول slot في الجدول */
    union security_list_options hook;      /* الـ function pointer الفعلي */
    const struct lsm_id *lsmid;           /* مين صاحب الـ hook ده */
} __randomize_layout;
```

| Field | الوظيفة |
|-------|---------|
| `scalls` | pointer لـ `static_calls_table.HOOKNAME` — أول slot في الـ array |
| `hook` | الـ callback الفعلي — function pointer في الـ union |
| `lsmid` | مرجع لـ `lsm_id` الخاص بالـ LSM اللي سجّل |

**بناءه:** الـ macro `LSM_HOOK_INIT` بيبسّط الكلام:
```c
/* بدل ما تكتب كل ده يدويًا */
static struct security_hook_list selinux_hooks[] = {
    LSM_HOOK_INIT(file_open, selinux_file_open),
    LSM_HOOK_INIT(inode_permission, selinux_inode_permission),
    /* ... */
};
```

---

#### `struct lsm_blob_sizes`

**الغرض:** لما أكتر من LSM بيشتغلوا مع بعض (**stacking**)، كل واحد محتاج يخزن بيانات خاصة بيه في kernel objects زي `inode`, `cred`, `file`, إلخ. الـ struct ده بيحدد **كام byte** كل LSM محتاجه.

```c
struct lsm_blob_sizes {
    unsigned int lbs_cred;        /* bytes في struct cred */
    unsigned int lbs_file;        /* bytes في struct file */
    unsigned int lbs_ib;          /* bytes لـ InfiniBand security */
    unsigned int lbs_inode;       /* bytes في struct inode */
    unsigned int lbs_sock;        /* bytes في struct sock */
    unsigned int lbs_superblock;  /* bytes في struct super_block */
    unsigned int lbs_ipc;         /* bytes في IPC objects */
    unsigned int lbs_key;         /* bytes في struct key */
    unsigned int lbs_msg_msg;     /* bytes في struct msg_msg */
    unsigned int lbs_perf_event;  /* bytes لـ perf events */
    unsigned int lbs_task;        /* bytes في task security */
    unsigned int lbs_xattr_count; /* عدد xattr slots */
    unsigned int lbs_tun_dev;     /* bytes لـ TUN devices */
    unsigned int lbs_bdev;        /* bytes لـ block devices */
    unsigned int lbs_bpf_map;     /* bytes لـ BPF maps */
    unsigned int lbs_bpf_prog;    /* bytes لـ BPF programs */
    unsigned int lbs_bpf_token;   /* bytes لـ BPF tokens */
};
```

**المثال العملي:** SELinux ممكن يحتاج 16 bytes في كل `inode`، AppArmor يحتاج 8 bytes. الـ framework بيجمعهم ويخصص blob واحد بيحتوي على الاتنين.

---

#### `struct lsm_info`

**الغرض:** ده الـ "registration record" الكامل لأي LSM — بيحتوي على كل اللي الـ framework محتاجه عشان يعرف يشغّل الـ LSM.

```c
struct lsm_info {
    const struct lsm_id *id;          /* هوية الـ LSM */
    enum lsm_order order;             /* أول / عادي / آخر */
    unsigned long flags;              /* LSM_FLAG_* */
    struct lsm_blob_sizes *blobs;     /* كام ميموري محتاجه لكل object type */
    int *enabled;                     /* pointer لـ variable الـ enable/disable */
    int (*init)(void);                /* الـ init function الرئيسي */
    /* initcalls بترتيب مراحل الكيرنل: */
    int (*initcall_pure)(void);
    int (*initcall_early)(void);
    int (*initcall_core)(void);
    int (*initcall_subsys)(void);
    int (*initcall_fs)(void);
    int (*initcall_device)(void);
    int (*initcall_late)(void);
};
```

**الإضافة في الميموري:** بيتحط في section خاصة باستخدام `DEFINE_LSM`:
```c
DEFINE_LSM(selinux) = {
    .id = &selinux_lsmid,
    .init = selinux_init,
    .blobs = &selinux_blob_sizes,
};
```
ده بيحط الـ `lsm_info` في `.lsm_info.init` section — الكيرنل بيلف عليها في وقت الـ boot.

---

#### `struct lsm_context` (في `security.h`)

**الغرض:** بيمثل security context نصي (زي `"system_u:system_r:httpd_t"` في SELinux).

```c
struct lsm_context {
    char *context;  /* النص المقروء للـ security label */
    u32 len;        /* طول النص */
    int id;         /* ID الـ LSM اللي أنتج الـ context ده */
};
```

---

#### `struct lsm_prop` (في `security.h`)

**الغرض:** aggregate struct بيحتوي على security properties من كل الـ LSMs المفعّلة.

```c
struct lsm_prop {
    struct lsm_prop_selinux selinux;
    struct lsm_prop_smack smack;
    struct lsm_prop_apparmor apparmor;
    struct lsm_prop_bpf bpf;
};
```

بيُستخدم في hooks زي `inode_getlsmprop`, `cred_getlsmprop` عشان يجيب الـ properties من كل الـ LSMs في مكان واحد.

---

### 2. Struct Relationship Diagram

```
                      ┌─────────────────────┐
                      │    struct lsm_info  │
                      │  (DEFINE_LSM macro) │
                      │  .lsm_info.init ELF │
                      └──────────┬──────────┘
                                 │ .id
                    ┌────────────▼────────────┐
                    │      struct lsm_id      │
                    │  .name = "selinux"      │
                    │  .id = LSM_ID_SELINUX   │
                    └────────────────────────-┘
                                 ▲
          .lsmid                 │               .lsmid
    ┌─────────────────┐          │          ┌─────────────────┐
    │security_hook_list│─────────┘          │security_hook_list│
    │  (file_open)    │                     │ (inode_perm)    │
    │  .scalls ──────►│──────────────────────►  .scalls ─────►│
    │  .hook.file_open│      points into    │  .hook.inode_.. │
    └─────────────────┘  lsm_static_calls   └─────────────────┘
                                │ table
                    ┌───────────▼─────────────────────────────┐
                    │   struct lsm_static_calls_table          │
                    │   (global: static_calls_table)           │
                    │                                          │
                    │  .file_open[0] ── .file_open[1] ── ...  │
                    │  .inode_perm[0] ─ .inode_perm[1] ─ ...  │
                    │  .task_kill[0] ── .task_kill[1] ── ...   │
                    │  (MAX_LSM_COUNT slots per hook)          │
                    └──────────────────────────────────────────┘
                                │ each cell is:
                    ┌───────────▼─────────────────────────────┐
                    │   struct lsm_static_call                 │
                    │   .key ──────► static_call_key           │
                    │   .trampoline► asm trampoline code       │
                    │   .hl ───────► security_hook_list (back) │
                    │   .active ───► static_key_false          │
                    └─────────────────────────────────────────-┘

    ┌─────────────────────────────────────────────────────────┐
    │               struct lsm_info                           │
    │   .blobs ──────────────────────────────────────────────►│
    │                                                         │
    │   struct lsm_blob_sizes                                 │
    │   .lbs_cred = 16                                        │
    │   .lbs_inode = 32                                       │
    │   .lbs_file = 8                                         │
    │   ...                                                   │
    └─────────────────────────────────────────────────────────┘

    union security_list_options
    ┌──────────────────────────────────────────┐
    │ int (*file_open)(struct file *file)      │─── field في security_hook_list.hook
    │ int (*inode_permission)(...)             │
    │ int (*task_kill)(...)                    │
    │ void *lsm_func_addr  ◄── generic access  │
    └──────────────────────────────────────────┘
```

---

### 3. Lifecycle Diagrams

#### A. LSM Registration Lifecycle

```
Boot Time
    │
    ▼
ELF sections (.lsm_info.init, .early_lsm_info.init) loaded in memory
    │
    ▼
kernel iterates over lsm_info structs (ordered by lsm_order)
    │
    ├── LSM_ORDER_FIRST   → capabilities (أول حاجة)
    ├── LSM_ORDER_MUTABLE → selinux, apparmor, smack (ترتيبها من CONFIG_LSM)
    └── LSM_ORDER_LAST    → integrity (آخر حاجة)
    │
    ▼
لكل lsm_info:
  1. تخصيص blob sizes (lsm_blob_sizes) — بيُضاف لكل struct kernel
  2. استدعاء lsm_info.init()
      │
      └── داخل init():
          ┌──────────────────────────────────────────┐
          │ selinux_init() مثلًا:                    │
          │   ① تجهيز internal data structures       │
          │   ② بناء security_hook_list array        │
          │       LSM_HOOK_INIT(file_open,            │
          │                    selinux_file_open)     │
          │   ③ استدعاء security_add_hooks(hooks,    │
          │                                count,    │
          │                                &lsm_id)  │
          └──────────────────────────────────────────┘
    │
    ▼
security_add_hooks():
  - لكل security_hook_list في الـ array:
    ① ابحث عن أول slot فاضي في static_calls_table.HOOKNAME
    ② حط الـ function pointer في lsm_static_call
    ③ فعّل الـ active static_key
    ④ patch الـ static call trampoline بعنوان الـ function الجديدة
    │
    ▼
static_calls_table هيبقى __ro_after_init (read-only بعد init)
    │
    ▼
Runtime: الـ hooks جاهزة للاستدعاء بأعلى performance ممكن
```

#### B. Hook Invocation Lifecycle (Runtime)

```
userspace syscall (مثلًا open())
    │
    ▼
kernel: security_file_open(file)   ← في security/security.c
    │
    ▼
يلف على static_calls_table.file_open[]
من آخر slot مفعّل لأول slot مفعّل
    │
    ├── slot[1] active? YES → static_call(slot[1].key)(file)
    │       │
    │       └── يروح لـ apparmor_file_open(file) مثلًا
    │               ↓ return 0 (مسموح)
    │
    ├── slot[0] active? YES → static_call(slot[0].key)(file)
    │       │
    │       └── يروح لـ selinux_file_open(file)
    │               ↓ return -EACCES (مرفوض!)
    │
    └── نتيجة أول error بتوقف الباقي
    │
    ▼
بيرجع النتيجة للـ VFS
```

#### C. Blob Allocation Lifecycle

```
LSM Registration Phase:
    lsm_info.blobs → lsm_blob_sizes { lbs_inode = 16 }
    │
    ▼
kernel accumulates all LSM blob sizes:
    total_inode_blob = selinux(16) + apparmor(8) + smack(4) = 28 bytes
    │
    ▼
Runtime — inode_alloc_security() hook:
    ① alloc 28 bytes كـ security blob
    ② يحط pointer في inode->i_security
    ③ SELinux بياخد bytes 0-15
    ④ AppArmor بياخد bytes 16-23
    ⑤ Smack بياخد bytes 24-27
    │
    ▼
كل LSM بيوصل لـ blob بتاعه عن طريق:
    selinux_inode(inode) = inode->i_security + selinux_blob_offset
    │
    ▼
inode_free_security() hook:
    ① كل LSM بيعمل cleanup لـ blob بتاعه
    ② kfree(inode->i_security)
```

---

### 4. Call Flow Diagrams

#### A. LSM Hook Definition Flow (X-Macro Pattern)

```
lsm_hook_defs.h:
  LSM_HOOK(int, 0, file_open, struct file *file)
        │
        │ يتضمن في union security_list_options:
        ├── int (*file_open)(struct file *file);
        │
        │ يتضمن في struct lsm_static_calls_table:
        ├── struct lsm_static_call file_open[MAX_LSM_COUNT];
        │
        │ يولّد static call declaration:
        └── DECLARE_STATIC_CALL(lsm_file_open, ...)
```

#### B. DEFINE_LSM Macro Flow

```
في selinux/hooks.c:
  DEFINE_LSM(selinux) = { .id = &selinux_lsmid, .init = selinux_init }
        │
        ▼
  static struct lsm_info __lsm_selinux
      __used
      __section(".lsm_info.init")
      __aligned(sizeof(unsigned long))
        │
        ▼
  ELF linker script يجمع كل lsm_info structs في نطاق محدد
  __start_lsm_info ... __end_lsm_info
        │
        ▼
  security_init() في boot:
    for (lsm = __start_lsm_info; lsm < __end_lsm_info; lsm++) {
        if (!*lsm->enabled) continue;
        lsm_early_lsm(lsm);  /* or ordered call */
    }
```

#### C. security_add_hooks() Flow

```
LSM calls:
  security_add_hooks(selinux_hooks, ARRAY_SIZE(selinux_hooks), &selinux_lsmid)
        │
        ▼
  for each hook in selinux_hooks[]:
    hook.lsmid = lsmid;           /* ربط الـ hook بالـ LSM */
        │
        ▼
    lsm_static_call = hook.scalls + next_free_slot
        │
        ▼
    static_call_update(lsm_static_call->key, hook.hook.lsm_func_addr)
        │                    ↑ يعمل patch للـ instruction في الميموري
        ▼
    static_key_enable(lsm_static_call->active)
        │                    ↑ يفعّل الـ branch لهذا الـ slot
        ▼
  الـ hook جاهز — أي استدعاء لاحق بيروح مباشرة للـ function
```

#### D. lsm_get_xattr_slot() Flow

```
inode_init_security hook → LSM يحتاج يكتب xattr:

  xattrs = alloc_xattr_array(xattr_count)     /* array من struct xattr */
        │
        ▼
  slot = lsm_get_xattr_slot(xattrs, &count)
        │
        ├── xattrs == NULL? → return NULL (no xattrs requested)
        │
        └── return &xattrs[count++]           /* slot التالي المتاح */
              │
              ▼
        LSM يملأ:
          slot->name = "security.selinux"
          slot->value = selinux_context
          slot->value_len = ctx_len
```

#### E. Static Call vs Function Pointer — Performance Flow

```
قديمًا (function pointer):
  security_file_open(file)
    → load hook_list_head           (cache miss محتمل)
    → load function pointer         (cache miss)
    → CALL *rax                     (indirect → retpoline overhead!)

حديثًا (static call):
  security_file_open(file)
    → لو hook مش active: JMP next  (single branch — fast)
    → لو hook active:
      CALL selinux_file_open        (direct call — no retpoline!)
       الـ instruction نفسها بتتعدل في الميموري وقت الـ registration
```

---

### 5. Locking Strategy

الـ `lsm_hooks.h` نفسه ما فيهوش lock definitions صريحة، لكن الـ framework بيعتمد على استراتيجيات محددة:

#### A. `__ro_after_init` للـ static_calls_table

```c
extern struct lsm_static_calls_table static_calls_table __ro_after_init;
```

| الآلية | التفاصيل |
|--------|---------|
| `__ro_after_init` | بعد انتهاء الـ kernel init، الصفحات اللي فيها الجدول بتبقى read-only على مستوى الـ page table. أي محاولة كتابة = page fault. |
| الهدف | منع أي code (حتى kernel code) من تعديل الـ hook table بعد الـ boot |
| الـ Registration | بيحصل فقط خلال `security_init()` قبل ما الـ ro_after_init يتفعّل |

#### B. الـ `static_key_false` في `lsm_static_call`

```c
struct static_key_false *active;
```

- الـ `static_key` بيستخدم آلية خاصة للـ locking عند التعديل: بيوقف كل الـ CPUs (stop_machine) عشان يعمل patch للـ instructions بأمان.
- ده مش بيحصل في الـ runtime العادي — بس في الـ init.

#### C. الـ Hook Execution (Lockless)

بعد الـ init، تنفيذ الـ hooks بيكون **lockless بالكامل**:
- الـ `static_call` هو direct branch instruction — مفيش قراءة لأي pointer.
- الـ `static_key_false` branch هو nop أو jmp — بيتقرر statically في الـ CPU.
- مفيش spinlock ولا mutex في الـ hot path.

#### D. الـ Blob Access Locking

| الـ Object | الـ Lock |
|------------|---------|
| `inode->i_security` | `inode->i_lock` (spinlock) لو الـ inode مشارك |
| `cred->security` | الـ cred نفسها immutable بعد ما بتتعمل — مفيش lock محتاج |
| `file->f_security` | `file_lock` في بعض السياقات |
| `task security blob` | task_lock أو RCU حسب السياق |

#### E. ترتيب الـ Hooks (Lock Ordering Implication)

الـ LSMs بتتنفذ بترتيب محدد (`LSM_ORDER_*`). ده مهم لأن:
- لو LSM الأول رجع error، الباقيين ما بيتشغلوش.
- مفيش two LSMs بيشغّلوا hook بالتوازي لنفس الـ object (الـ kernel serializes الـ calls عبر الـ caller's locks).
- الـ `capabilities` LSM دايمًا أول — لو fail هنا ما بنوصلش للباقيين.

```
Lock Ordering للـ security subsystem:
  task_lock
    └── inode_lock (i_rwsem)
          └── file_lock
                └── [LSM hooks run here — no additional LSM-level lock]
```
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ APIs والـ Functions المهمة

#### جدول الـ Macros

| Macro | الغرض |
|-------|--------|
| `LSM_HOOK(RET, DEFAULT, NAME, ...)` | يعرّف hook واحد في الـ LSM framework — بيُستخدم عشان يولّد structs وstatic calls |
| `LSM_HOOK_INIT(NAME, HOOK)` | بيهيّئ `security_hook_list` entry بشكل مختصر |
| `DEFINE_LSM(lsm)` | بيعرّف `lsm_info` struct في section `.lsm_info.init` |
| `DEFINE_EARLY_LSM(lsm)` | زي `DEFINE_LSM` بس في section `.early_lsm_info.init` |
| `LSM_RET_VOID` | القيمة الافتراضية للـ hooks اللي void return type |
| `LSM_FLAG_LEGACY_MAJOR` | BIT(0) — الـ LSM ده legacy major LSM |
| `LSM_FLAG_EXCLUSIVE` | BIT(1) — الـ LSM ده exclusive، مش بيشارك hooks مع غيره |

#### جدول الـ Functions

| Function | النوع | الغرض |
|----------|-------|--------|
| `security_add_hooks()` | exported kernel function | بتسجّل array من الـ hooks لـ LSM معين |
| `lsm_get_xattr_slot()` | static inline helper | بترجع أول slot فاضي في الـ xattrs array |

#### جدول الـ Structs

| Struct | الغرض |
|--------|--------|
| `lsm_static_call` | entry واحدة في جدول الـ static calls لـ hook محدد |
| `lsm_static_calls_table` | الجدول الكامل لكل static calls لكل الـ hooks |
| `lsm_id` | بيعرّف هوية الـ LSM (اسم + ID رقمي) |
| `security_hook_list` | entry واحدة من hooks اللي LSM بيسجّلها |
| `lsm_blob_sizes` | بيحدد حجم/offset الـ security blob في كل object |
| `lsm_info` | البنية الأساسية اللي بتعرّف LSM كامل للـ framework |
| `union security_list_options` | union بيحتوي على كل function pointers للـ hooks |

---

### التقسيم حسب الفئة المنطقية

---

### فئة 1: Registration — تسجيل الـ LSM ودي الـ Hooks

#### `security_add_hooks()`

```c
extern void security_add_hooks(struct security_hook_list *hooks,
                                int count,
                                const struct lsm_id *lsmid);
```

الدالة دي هي **نقطة التسجيل الوحيدة** اللي LSM بيستخدمها عشان يحط callbacks بتاعته في الـ framework. بتاخد array من الـ `security_hook_list` entries وبتربطها بالـ static calls table الخاصة بكل hook. الـ framework بيملّي الـ static calls من آخر LSM لأول واحد عشان الترتيب يكون صح وقت الاستدعاء.

**Parameters:**
- `hooks` — pointer لأول element في الـ array من `security_hook_list`. كل element بتمثّل hook واحد مع الـ callback بتاعه.
- `count` — عدد الـ entries في الـ array.
- `lsmid` — pointer لـ `lsm_id` struct اللي بيعرّف هوية الـ LSM صاحب الـ hooks دي.

**Return value:** void — مفيش return، أي فشل هنا kernel panic.

**Key details:**
- بتتاخد في وقت الـ init فقط (داخل `__init` context).
- بتعدّل الـ `static_calls_table` اللي هي `__ro_after_init` — يعني بعد الـ init مش هينفع تتعدّل.
- الـ locking يكون implicit لأنها بتتاخد في single-threaded init phase.
- الـ static calls بتتملى من الآخر للأول (backward fill) عشان الـ entry point يكون dynamic حسب عدد الـ LSMs المفعّلة.

**Who calls it:**
- كل LSM في `init()` function بتاعته. مثلاً SELinux يعمل كده:

```c
/* مثال حقيقي من SELinux */
static struct security_hook_list selinux_hooks[] __ro_after_init = {
    LSM_HOOK_INIT(binder_set_context_mgr, selinux_binder_set_context_mgr),
    LSM_HOOK_INIT(ptrace_access_check,    selinux_ptrace_access_check),
    /* ... */
};

static int __init selinux_init(void)
{
    security_add_hooks(selinux_hooks,
                       ARRAY_SIZE(selinux_hooks),
                       &selinux_lsmid);
    return 0;
}
```

**Pseudocode flow:**

```
security_add_hooks(hooks, count, lsmid):
    for i in 0..count:
        hook = &hooks[i]
        hook->lsmid = lsmid
        /* ابحث عن أول slot فاضي في الـ static calls table للـ hook ده */
        for each slot in hook->scalls (backward):
            if slot is empty:
                static_call_update(slot->key, hook->hook.func_addr)
                static_key_enable(slot->active)
                break
```

---

### فئة 2: Initialization Infrastructure — بنية التهيئة

#### الـ Macro `DEFINE_LSM(lsm)`

```c
#define DEFINE_LSM(lsm)                             \
    static struct lsm_info __lsm_##lsm              \
        __used __section(".lsm_info.init")          \
        __aligned(sizeof(unsigned long))
```

**الـ macro ده بيعرّف `lsm_info` struct** في special linker section اسمها `.lsm_info.init`. الـ kernel بوقت boot بيمشي على كل الـ entries في السكشن دي عشان يعرف إيه الـ LSMs اللي المفروض يشغّلها. الترتيب بيتحكم فيه `enum lsm_order`.

**Key details:**
- `__used` — بيمنع الـ compiler من إنه يحذف الـ symbol.
- `__section(".lsm_info.init")` — بيحط الـ struct في section معينة في الـ vmlinux عشان الـ LSM framework يقدر يلاقيها.
- `__aligned(sizeof(unsigned long))` — alignment مضمون عشان الـ framework يقدر يعمل iteration بشكل صح.

**Usage pattern:**

```c
DEFINE_LSM(mymod) = {
    .id       = &mymod_lsmid,
    .init     = mymod_init,
    .blobs    = &mymod_blob_sizes,
    .enabled  = &mymod_enabled,
    .order    = LSM_ORDER_MUTABLE,
};
```

#### الـ Macro `DEFINE_EARLY_LSM(lsm)`

```c
#define DEFINE_EARLY_LSM(lsm)                          \
    static struct lsm_info __early_lsm_##lsm           \
        __used __section(".early_lsm_info.init")       \
        __aligned(sizeof(unsigned long))
```

**نفس `DEFINE_LSM` بس للـ LSMs اللي لازم تتشغّل مبكر** قبل باقي الـ subsystems (زي Yama). الـ section مختلفة `.early_lsm_info.init` عشان الـ framework يعرف يفرّق بينهم.

---

### فئة 3: Hook Initialization — تهيئة الـ Hooks

#### الـ Macro `LSM_HOOK_INIT(NAME, HOOK)`

```c
#define LSM_HOOK_INIT(NAME, HOOK)           \
    {                                       \
        .scalls = static_calls_table.NAME, \
        .hook = { .NAME = HOOK }           \
    }
```

**Convenience macro** بيملّي `security_hook_list` entry بشكل صح ومختصر. بيربط الـ callback بـ slot بتاعه في الـ `static_calls_table` الـ global.

**Key details:**
- `.scalls` بيشاور على الـ array من `lsm_static_call` الخاص بالـ hook ده في الـ global table.
- `.hook` هي الـ union اللي بتحتوي function pointer.
- بيُستخدم دايماً في الـ array الـ `__ro_after_init` الخاص بكل LSM.

**مثال:**

```c
/* لو عندك callback اسمه mymod_capable */
LSM_HOOK_INIT(capable, mymod_capable)
/* ده بيعادل: */
{
    .scalls = static_calls_table.capable,
    .hook   = { .capable = mymod_capable },
}
```

---

### فئة 4: xattr Helper — مساعدة الـ Extended Attributes

#### `lsm_get_xattr_slot()`

```c
static inline struct xattr *lsm_get_xattr_slot(struct xattr *xattrs,
                                                int *xattr_count)
{
    if (unlikely(!xattrs))
        return NULL;
    return &xattrs[(*xattr_count)++];
}
```

**الدالة دي بترجع pointer** للـ slot الأول الفاضي في الـ `xattrs` array وبتزوّد الـ counter بمقدار واحد. الـ LSMs بتستخدمها لما بتحتاج تضيف xattr للـ inode الجديد وقت الـ `inode_init_security` hook.

**Parameters:**
- `xattrs` — pointer لـ array من `struct xattr` اللي الـ VFS خصّصه. لو NULL يبقى مفيش slots.
- `xattr_count` — pointer لـ counter بيتكلم على عدد الـ xattrs المتملّيين حاليًا — بيتزوّد atomically بعد الرجوع.

**Return value:**
- Pointer لـ `struct xattr` slot جاهز للملء.
- NULL لو الـ `xattrs` نفسها NULL (مش مخصّص).

**Key details:**
- `unlikely(!xattrs)` — branch hint لأن الـ case العادي هو إن الـ array موجود.
- الدالة مش thread-safe بذاتها — الـ locking مسؤولية الـ caller (عادة بيكون في context اللي مفيهوش تنافس).
- الـ `lbs_xattr_count` في `lsm_blob_sizes` بيحدد عدد الـ slots المتاحة — الـ LSM لازم يحجز slots كافية فيها.

**Who calls it:**
- أي LSM بيعمل implement للـ `inode_init_security` hook:

```c
/* مثال من SELinux-style LSM */
static int mymod_inode_init_security(struct inode *inode,
                                     struct inode *dir,
                                     const struct qstr *qstr,
                                     struct xattr *xattrs,
                                     int *xattr_count)
{
    struct xattr *slot;

    slot = lsm_get_xattr_slot(xattrs, xattr_count);
    if (!slot)
        return 0; /* no xattrs array → skip */

    slot->name  = XATTR_NAME_MYMOD;
    slot->value = mymod_compute_label(inode);
    slot->value_len = strlen(slot->value);
    return 0;
}
```

---

### فئة 5: Core Data Structures — البنى الأساسية

#### `union security_list_options`

```c
union security_list_options {
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) RET (*NAME)(__VA_ARGS__);
    #include "lsm_hook_defs.h"
    #undef LSM_HOOK
    void *lsm_func_addr;
};
```

**الـ union ده هو الـ container** لكل function pointers الخاصة بكل الـ hooks الممكنة في الـ LSM framework. كل member اسمه اسم الـ hook، وكل `security_hook_list` entry بتحتوي واحد منهم. الـ `lsm_func_addr` member بيسمح بالوصول الـ generic للـ function pointer كـ `void *` عشان الـ framework يقدر يحطها في الـ static call.

**Key details:**
- الـ `lsm_hook_defs.h` بيتضمن مرتين — مرة هنا لتوليد الـ union members، ومرة في `lsm_static_calls_table` لتوليد الـ arrays.
- X-macro pattern — `LSM_HOOK` بيتعرّف بطرق مختلفة في كل مكان بيتضمن فيه `lsm_hook_defs.h`.

---

#### `struct lsm_static_call`

```c
struct lsm_static_call {
    struct static_call_key *key;       /* مفتاح الـ static call */
    void                   *trampoline;/* الـ trampoline المرتبط */
    struct security_hook_list *hl;     /* الـ hook list اللي ملكيته */
    struct static_key_false *active;   /* هل الـ static call ده مفعّل؟ */
} __randomize_layout;
```

**كل instance من الـ struct ده** بيمثّل static call slot واحد لـ hook واحد. الـ framework بيخصص `MAX_LSM_COUNT` slots لكل hook (مش أكتر من عدد الـ LSMs المـcompile-time). الـ `__randomize_layout` بيخلي الـ kernel يعمل KASLR على الـ struct layout.

| Field | الوصف |
|-------|--------|
| `key` | الـ key الخاص بالـ `STATIC_CALL_KEY` — بيُستخدم في `static_call_update()` |
| `trampoline` | الـ code patch target — الـ jump instruction اللي بيتعدّل |
| `hl` | back-pointer للـ `security_hook_list` اللي ملكيته، أو NULL لو فاضي |
| `active` | `static_key_false` — بيُفعَّل لو في LSM مسجّل هنا |

**Key details:**
- الـ static key بيتحكم في الـ jump في بداية كل hook call — لو مفيش LSMs مسجّلة للـ hook ده، الـ call بيتخطى كل الـ slots بدون overhead.
- الـ slots بتتملى من الآخر — آخر LSM في الترتيب بيكون في أعلى index، والـ execution بيبدأ من أول index غير فاضي.

---

#### `struct lsm_static_calls_table`

```c
struct lsm_static_calls_table {
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) \
        struct lsm_static_call NAME[MAX_LSM_COUNT];
    #include <linux/lsm_hook_defs.h>
    #undef LSM_HOOK
} __packed __randomize_layout;
```

**الجدول الكامل** اللي بيحتوي على array من static call slots لكل hook موجود. الـ `__packed` بيمنع padding عشان الـ layout يكون compact. الـ `static_calls_table` instance منه هو `extern __ro_after_init` — يعني read-only بعد نهاية الـ init.

**مثال على الـ layout المنطقي:**

```
static_calls_table:
  .capable[0]   → SELinux capable hook     (active)
  .capable[1]   → AppArmor capable hook    (active)
  .capable[2]   → (empty)
  ...
  .ptrace_access_check[0] → SELinux hook   (active)
  .ptrace_access_check[1] → (empty)
  ...
```

---

#### `struct lsm_id`

```c
struct lsm_id {
    const char *name;  /* اسم الـ LSM — لازم approved من LSM maintainers */
    u64         id;    /* رقم ID من uapi/linux/lsm.h */
};
```

**بتعرّف هوية الـ LSM** — الاسم بيظهر في sysfs وفي الـ audit logs، والـ ID رقمي ثابت معرّف في الـ UAPI عشان الـ userspace يقدر يشتغل بيه بدون اعتماد على strings.

---

#### `struct security_hook_list`

```c
struct security_hook_list {
    struct lsm_static_call      *scalls; /* pointer لأول slot في الجدول */
    union security_list_options  hook;   /* الـ callback function pointer */
    const struct lsm_id         *lsmid;  /* هوية الـ LSM صاحب الـ hook */
} __randomize_layout;
```

**الوحدة الأساسية** لتسجيل hook واحد من LSM واحد. الـ LSM بيعمل array من الـ structs دي (واحد لكل hook بيعمله implement) وبيبعتها لـ `security_add_hooks()`.

**Relationship diagram:**

```
security_hook_list[]  (per-LSM array, __ro_after_init)
        │
        ├── scalls ──→ lsm_static_calls_table.hook_name[]
        │                   ├── [0] → lsm_static_call (active)
        │                   ├── [1] → lsm_static_call (active)
        │                   └── [N] → lsm_static_call (empty)
        │
        ├── hook ────→ LSM callback function pointer
        │
        └── lsmid ───→ lsm_id { .name, .id }
```

---

#### `struct lsm_blob_sizes`

```c
struct lsm_blob_sizes {
    unsigned int lbs_cred;        /* حجم blob في struct cred */
    unsigned int lbs_file;        /* حجم blob في struct file */
    unsigned int lbs_ib;          /* حجم blob في InfiniBand */
    unsigned int lbs_inode;       /* حجم blob في struct inode */
    unsigned int lbs_sock;        /* حجم blob في struct sock */
    unsigned int lbs_superblock;  /* حجم blob في struct super_block */
    unsigned int lbs_ipc;         /* حجم blob في IPC objects */
    unsigned int lbs_key;         /* حجم blob في struct key */
    unsigned int lbs_msg_msg;     /* حجم blob في struct msg_msg */
    unsigned int lbs_perf_event;  /* حجم blob في perf events */
    unsigned int lbs_task;        /* حجم blob في task security */
    unsigned int lbs_xattr_count; /* عدد xattr slots مطلوبة */
    unsigned int lbs_tun_dev;     /* حجم blob في TUN devices */
    unsigned int lbs_bdev;        /* حجم blob في block devices */
    unsigned int lbs_bpf_map;     /* حجم blob في BPF maps */
    unsigned int lbs_bpf_prog;    /* حجم blob في BPF programs */
    unsigned int lbs_bpf_token;   /* حجم blob في BPF tokens */
};
```

**الـ struct ده هو اللي LSM بيعلن بيه** احتياجاته من الـ security blobs. الـ framework بيجمع كل الـ blob sizes من كل الـ LSMs وبيخصص memory متراكمة (stacked blobs). كل LSM بيحصل على offset داخل الـ blob الكلي.

**Key details:**
- الـ `lbs_xattr_count` مختلف — ده مش size بس count عشان الـ VFS يقدر يخصص array كافية لـ xattrs.
- الـ framework بيعمل aggregation وقت boot — بيجمع كل الـ sizes ويعمل allocate واحدة كبيرة لكل object type.
- الـ individual LSM بيوصل للـ blob بتاعه من خلال offset ثابت بيتحسبله وقت الـ init.

---

#### `struct lsm_info`

```c
struct lsm_info {
    const struct lsm_id   *id;               /* اسم ورقم الـ LSM */
    enum lsm_order         order;            /* ترتيب التشغيل */
    unsigned long          flags;            /* LSM_FLAG_* */
    struct lsm_blob_sizes *blobs;            /* متطلبات الـ blob */
    int                   *enabled;          /* هل مفعّل؟ (CONFIG_LSM) */
    int (*init)(void);                       /* دالة الـ init الرئيسية */
    int (*initcall_pure)(void);              /* callback للـ initcall_pure */
    int (*initcall_early)(void);             /* callback للـ early_initcall */
    int (*initcall_core)(void);              /* callback للـ core_initcall */
    int (*initcall_subsys)(void);            /* callback للـ subsys_initcall */
    int (*initcall_fs)(void);               /* callback للـ fs_initcall */
    int (*initcall_device)(void);            /* callback للـ device_initcall */
    int (*initcall_late)(void);              /* callback للـ late_initcall */
};
```

**البنية الأكثر أهمية** في الـ lsm_hooks.h — بتعرّف LSM كامل للـ framework. الـ framework بيقرأ كل الـ `lsm_info` entries من الـ `.lsm_info.init` section وقت الـ boot ويعمل لهم init بالترتيب.

**Fields:**

| Field | الوصف |
|-------|--------|
| `id` | pointer لـ `lsm_id` — الاسم والرقم الفريد |
| `order` | `LSM_ORDER_FIRST` (capabilities فقط)، `LSM_ORDER_MUTABLE` (عادي)، `LSM_ORDER_LAST` (integrity فقط) |
| `flags` | `LSM_FLAG_LEGACY_MAJOR` أو `LSM_FLAG_EXCLUSIVE` |
| `blobs` | طلبات حجم الـ security blobs |
| `enabled` | pointer لـ int — بيُتحكم فيه من خلال `CONFIG_LSM=` kernel parameter |
| `init` | الدالة الرئيسية اللي بتتاخد في وقت مناسب حسب الـ order |
| `initcall_*` | callbacks اختيارية لـ initcall levels مختلفة لو الـ LSM محتاج يعمل حاجات في stages مختلفة |

**Key details:**
- `order` بيتحكم في الترتيب بين الـ LSMs — مش ترتيب الـ hooks نفسها وقت التنفيذ.
- الـ `enabled` pointer مهم — الـ userspace أو الـ admin يقدر يـdisable LSM بـ `lsm=` kernel cmdline.
- الـ multiple `initcall_*` callbacks موجودة عشان بعض الـ LSMs محتاجة تعمل حاجات في stages مختلفة من الـ boot (مثلاً IMA محتاج يـsetup قبل الـ filesystems).

---

### فئة 6: Compile-time Constants — الثوابت وقت الـ Compile

#### `MAX_LSM_COUNT`

معرّف في `lsm_count.h` — بيحسب عدد الـ LSMs المفعّلة في الـ kernel وقت الـ compile. ده بيُستخدم كـ size للـ array في `lsm_static_calls_table`:

```c
/* كل hook عنده array بحجم MAX_LSM_COUNT */
struct lsm_static_call capable[MAX_LSM_COUNT];
```

الحساب بيتم بـ X-macro trick — كل LSM مفعّل بيضيف `1,` في الـ list وبعدين `COUNT_ARGS()` بتعدّ.

**Key details:**
- لو `CONFIG_SECURITY` مش مفعّل → `MAX_LSM_COUNT = 0`.
- الـ LSMs المدعومة حالياً: Capabilities, SELinux, Smack, AppArmor, Tomoyo, Yama, LoadPin, Lockdown, SafeSetID, BPF LSM, Landlock, IMA, EVM, IPE.

---

### الـ Flow الكامل من Boot لـ Hook Call

```
Boot time:
──────────
1. Linker يجمع كل lsm_info structs من .lsm_info.init section
2. LSM framework يمشي عليهم ويرتّبهم حسب order
3. لكل LSM: يستدعي lsm_info.init()
4. init() بتستدعي security_add_hooks(hooks, count, lsmid)
5. security_add_hooks() بتملّي static_calls_table بـ callbacks
   (من آخر slot لأول slot — backward fill)
6. الـ static_key_false المرتبطة بكل slot تتفعّل

Runtime (عند kernel security check):
──────────────────────────────────────
7. الـ kernel يعمل static_call(hook_name)(args...)
8. الـ static_call mechanism يـjump لأول slot مفعّل
9. كل slot بيستدعي الـ LSM callback
10. الـ control بيـfall-through لباقي الـ slots المفعّلة
11. النتيجة بتترجع للـ kernel (عادة: أول non-zero يوقف)
```

---

### ملاحظات على الـ Security و KASLR

- كل struct مهم معمّله `__randomize_layout` — الـ kernel ممكن يغيّر ترتيب الـ fields وقت compile عشان يصعّب exploitation.
- الـ `static_calls_table` هو `__ro_after_init` — بعد الـ boot ما تعملش write على الجدول ده، أي محاولة هتـtrigger fault.
- الـ `security_hook_list` arrays في كل LSM عادةً `__ro_after_init` كمان — الـ hooks مش بتتغيّر at runtime.
## Phase 5: دليل الـ Debugging الشامل

الـ LSM framework بيوفر نقاط تدخل (hooks) في كل مكان حساس في الـ kernel. لما حاجة بتتعطل — permission denied غريبة، LSM مش بيتحمل صح، أو hook مش بيتنفذ — لازم تعرف تتتبع المشكلة من أول ما بتدخل الـ `security_*` call لغاية ما بترجع النتيجة.

---

### Software Level

#### 1. debugfs — المدخل الأول

الـ LSM بيعرض بيانات مهمة عبر `/sys/kernel/debug/lsm/` (لو kernel compiled with `CONFIG_DEBUG_FS`).

| المسار | المحتوى | طريقة القراءة |
|--------|----------|---------------|
| `/sys/kernel/security/lsm` | قائمة الـ LSMs الشغالة | `cat /sys/kernel/security/lsm` |
| `/sys/kernel/debug/lsm/` | مجلد debugging عام | `ls /sys/kernel/debug/lsm/` |
| `/proc/self/attr/current` | الـ security context الخاص بالـ process الحالي | `cat /proc/self/attr/current` |
| `/proc/<pid>/attr/current` | الـ security context لأي process | `cat /proc/1234/attr/current` |
| `/proc/<pid>/attr/exec` | الـ context اللي هيتحول ليه عند exec | `cat /proc/1234/attr/exec` |
| `/proc/<pid>/attr/prev` | الـ context السابق | `cat /proc/1234/attr/prev` |
| `/proc/<pid>/attr/keycreate` | context لإنشاء الـ keys | `cat /proc/1234/attr/keycreate` |
| `/proc/<pid>/attr/sockcreate` | context لإنشاء الـ sockets | `cat /proc/1234/attr/sockcreate` |

```bash
# اقرأ كل الـ LSMs اللي على kernel دلوقتي
cat /sys/kernel/security/lsm
# مثال output: lockdown,capability,selinux,bpf

# شوف الـ security blob sizes لكل LSM
cat /sys/kernel/debug/lsm/blob_sizes 2>/dev/null || echo "not available"
```

---

#### 2. sysfs — الضبط والمراقبة

| المسار | الوظيفة |
|--------|---------|
| `/sys/kernel/security/` | root لكل الـ LSM sysfs entries |
| `/sys/kernel/security/selinux/enforce` | 0=permissive, 1=enforcing |
| `/sys/kernel/security/selinux/deny_unknown` | سلوك الـ unknown classes |
| `/sys/kernel/security/selinux/policyvers` | إصدار الـ policy |
| `/sys/kernel/security/apparmor/profiles` | قائمة الـ AppArmor profiles |
| `/sys/kernel/security/apparmor/.access` | اختبار الـ access يدوياً |
| `/sys/kernel/security/tomoyo/` | الـ TOMOYO entries |

```bash
# اعرف هل SELinux في enforcing أو permissive
cat /sys/kernel/security/selinux/enforce

# شوف كل الـ AppArmor profiles المحملة
cat /sys/kernel/security/apparmor/profiles

# اقرأ الـ LSM blob sizes المخصصة
for f in /sys/kernel/security/lsm/*; do echo "=== $f ==="; cat "$f" 2>/dev/null; done
```

---

#### 3. ftrace — تتبع الـ LSM hooks بدقة

الـ LSM hooks بتتسمى `security_<hook_name>` في الـ kernel. الـ ftrace بيخلي تشوف كل call بالتفصيل.

```bash
# فعّل الـ function tracer للـ LSM security functions
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'security_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل العملية اللي بتتتبعها هنا
sleep 2
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

الـ **tracepoints** المهمة لـ LSM:

```bash
# اعرض كل الـ LSM-related events المتاحة
ls /sys/kernel/debug/tracing/events/lsm/ 2>/dev/null

# فعّل selinux-specific events
echo 1 > /sys/kernel/debug/tracing/events/selinux/enable 2>/dev/null

# استخدم function_graph لشوف call stack كامل
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'security_inode_permission' > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

```bash
# تتبع hook معين: مثلاً security_file_open
echo 'security_file_open' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /proc/self/status  # trigger some file ops
cat /sys/kernel/debug/tracing/trace | head -50
```

---

#### 4. printk و dynamic debug

**الـ dynamic debug** بيخلي تفعّل الـ `pr_debug()` و `dev_dbg()` في runtime بدون recompile.

```bash
# فعّل كل الـ debug messages في ملفات الـ LSM
echo 'file security/security.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file security/selinux/*.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file security/apparmor/*.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل الـ debug في subsystem الـ LSM
echo 'module selinux +p' > /sys/kernel/debug/dynamic_debug/control

# شوف إيه اللي فعّلته
cat /sys/kernel/debug/dynamic_debug/control | grep security

# تابع الـ kernel log في realtime
dmesg -w | grep -E 'LSM|SELinux|AppArmor|avc:|audit'
```

للـ **printk** المباشر في الكود (لما بتعمل development):

```c
/* في hook function بتاعك */
pr_debug("LSM hook: %s called for inode %lu\n",
         __func__, inode->i_ino);

/* للـ critical paths */
pr_info("LSM[%s]: hook %s returned %d\n",
        lsmid->name, __func__, ret);

/* لو محتاج stack trace */
if (ret < 0)
    dump_stack();
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_SECURITY` | يفعّل الـ LSM framework أصلاً |
| `CONFIG_DEBUG_FS` | يفعّل `/sys/kernel/debug/` |
| `CONFIG_SECURITY_SELINUX` | يبني SELinux |
| `CONFIG_SECURITY_SELINUX_DEVELOP` | يفعّل permissive mode |
| `CONFIG_SECURITY_SELINUX_DEBUG` | debugging خاص بـ SELinux |
| `CONFIG_SECURITY_APPARMOR` | يبني AppArmor |
| `CONFIG_SECURITY_APPARMOR_DEBUG` | debugging خاص بـ AppArmor |
| `CONFIG_SECURITY_APPARMOR_DEBUG_ASSERTS` | تفعيل الـ assertions |
| `CONFIG_SECURITY_APPARMOR_DEBUG_MESSAGES` | رسائل تفصيلية |
| `CONFIG_SECURITY_SMACK` | يبني Smack LSM |
| `CONFIG_SECURITY_TOMOYO` | يبني TOMOYO LSM |
| `CONFIG_SECURITY_TOMOYO_MAX_AUDIT_LOG` | حجم الـ audit log |
| `CONFIG_SECURITY_LANDLOCK` | يبني Landlock |
| `CONFIG_SECURITY_LOCKDOWN_LSM` | يبني Lockdown LSM |
| `CONFIG_AUDIT` | يفعّل الـ audit subsystem |
| `CONFIG_AUDITSYSCALL` | audit على مستوى الـ syscalls |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل الـ dynamic debug |
| `CONFIG_KALLSYMS` | لازم لـ stack traces مفهومة |
| `CONFIG_LSM_MUTABLE_HOOKS` | يسمح بتعديل الـ hooks بعد init |
| `CONFIG_STATIC_USERMODEHELPER` | يقيّد الـ usermode helpers |

```bash
# تحقق من الـ configs الشغالة في kernel الحالي
zcat /proc/config.gz | grep -E 'CONFIG_SECURITY|CONFIG_AUDIT|CONFIG_DEBUG'
# أو
cat /boot/config-$(uname -r) | grep -E 'CONFIG_SECURITY|CONFIG_AUDIT'
```

---

#### 6. أدوات خاصة بالـ LSM Subsystem

**للـ SELinux:**
```bash
# اعرض الـ AVC denials الأخيرة
ausearch -m avc -ts recent
# أو
dmesg | grep 'avc:'

# فهم الـ denial وولّد policy
audit2why < /var/log/audit/audit.log
audit2allow -a

# تحقق من الـ context
ls -Z /path/to/file        # file context
ps -eZ | grep myprocess    # process context
id -Z                      # current user context

# seinfo / sesearch (من حزمة setools)
sesearch --allow --source httpd_t --target var_log_t
seinfo -t | grep httpd

# getenforce / setenforce
getenforce        # Enforcing / Permissive / Disabled
setenforce 0      # temporary permissive للـ testing
```

**للـ AppArmor:**
```bash
# شوف الـ profiles المحملة
apparmor_status
aa-status

# تتبع الـ violations
dmesg | grep 'apparmor='
journalctl | grep apparmor

# parse الـ profile
apparmor_parser -p /etc/apparmor.d/usr.sbin.nginx

# complain mode للـ debugging
aa-complain /etc/apparmor.d/usr.sbin.nginx
```

**للـ BPF LSM:**
```bash
# اعرض الـ BPF programs المحملة على LSM hooks
bpftool prog list | grep lsm
bpftool prog show id <ID>
bpftool prog dump xlated id <ID>
```

**لـ lsm_get_xattr_slot debugging:**
```bash
# شوف الـ xattrs الأمنية على ملف
getfattr -d -m security /path/to/file
getfattr -n security.selinux /path/to/file
getfattr -n security.SMACK64 /path/to/file
getfattr -n security.apparmor /path/to/file
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة في الـ log | المعنى | الحل |
|-----------------|--------|------|
| `avc: denied { <perm> } for pid=<N> comm="<cmd>" scontext=<src> tcontext=<tgt>` | SELinux منع عملية | `audit2allow -a` ثم أضف الـ rule |
| `apparmor="DENIED" operation="<op>" profile="<p>" name="<file>"` | AppArmor منع عملية | فعّل complain mode واضبط الـ profile |
| `audit: type=1400 audit(<time>)` | SELinux AVC denial مسجّل في audit | `ausearch -m avc` وحلّل الـ output |
| `LSM: Security Framework initializing` | الـ LSM framework بدأ التهيئة | informational فقط |
| `No LSM has been chosen` | مفيش LSM محدد في kernel cmdline | أضف `lsm=<name>` في boot params |
| `<LSM>: disabled` | LSM معطّل لأن مفيش hook | تحقق من `CONFIG_SECURITY_<LSM>` |
| `security_hook_list: hook <N> has no registered handler` | hook بدون implementation | لازم تفعّل الـ LSM الصح |
| `BUG: unable to handle kernel NULL pointer dereference` في `security_*` | hook pointer فارغ | تحقق من `lsm_static_call.active` |
| `WARNING: possible circular locking dependency` | deadlock محتمل في LSM locks | افحص ترتيب الـ hook calls |
| `integrity: Problem loading X.509 certificate` | مشكلة في IMA/EVM keys | تحقق من الـ keyring setup |
| `SMACK: failed to import label` | Smack label غلط | تحقق من الـ xattr values |
| `tomoyo: out of memory` | TOMOYO audit log امتلأ | زوّد `CONFIG_SECURITY_TOMOYO_MAX_AUDIT_LOG` |
| `landlock: error -22` | Landlock ruleset parameter غلط | تحقق من الـ ABI version |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/*
 * في security_add_hooks() — تحقق إن الـ hooks اتسجلت صح
 */
void security_add_hooks(struct security_hook_list *hooks, int count,
                        const struct lsm_id *lsmid)
{
    WARN_ON(!lsmid);          /* lsm_id لازم يكون valid */
    WARN_ON(count <= 0);      /* لازم يكون فيه hooks */
    /* ... */
}

/*
 * في الـ hook callback بتاعك — تحقق من الـ input
 */
static int my_lsm_inode_permission(struct inode *inode, int mask)
{
    WARN_ON_ONCE(!inode);     /* inode مفروش ميبقاش NULL هنا */
    if (unlikely(!inode->i_security)) {
        pr_err("LSM: inode %lu has no security blob!\n", inode->i_ino);
        dump_stack();         /* خد stack trace لتشوف مين اتصل */
        return -EINVAL;
    }
    /* ... */
}

/*
 * في lsm_get_xattr_slot — تحقق من الـ array bounds
 */
static inline struct xattr *lsm_get_xattr_slot(struct xattr *xattrs,
                                                int *xattr_count)
{
    /* WARN لو بنتعدى الـ lbs_xattr_count المحجوز */
    WARN_ON(*xattr_count >= MAX_LSM_XATTR_SLOTS);
    if (unlikely(!xattrs))
        return NULL;
    return &xattrs[(*xattr_count)++];
}

/*
 * نقاط WARN_ON المنطقية:
 * - لما lsm_static_call.active = true لكن .hl = NULL
 * - لما security_hook_list.lsmid->id مش في uapi/linux/lsm.h
 * - لما lsm_blob_sizes.lbs_* بتتغير بعد init
 */
```

---

### Hardware Level

#### 1. التحقق من تطابق Hardware State مع Kernel State

الـ LSM مش بيتعامل مع hardware مباشرة في معظم الأحيان، لكن بعض الـ LSMs بتعتمد على hardware features:

```bash
# تحقق من الـ CPU security features المتاحة
grep -E 'smep|smap|pku|umip|la57' /proc/cpuinfo

# تحقق من الـ Secure Boot state (يؤثر على Lockdown LSM)
mokutil --sb-state
# أو
cat /sys/firmware/efi/efivars/SecureBoot-*/  2>/dev/null | xxd

# تحقق من الـ TPM (لـ IMA/EVM)
ls /dev/tpm* 2>/dev/null
tpm2_getcap properties-fixed 2>/dev/null | head -20

# تحقق من kernel lockdown mode
cat /sys/kernel/security/lockdown 2>/dev/null
```

| Hardware Feature | الـ LSM المرتبط | علامة التحقق |
|------------------|-----------------|-------------|
| Secure Boot (UEFI) | Lockdown LSM | `mokutil --sb-state` |
| TPM 1.2/2.0 | IMA/EVM | `/dev/tpm0` موجود |
| CPU SMEP/SMAP | Kernel hardening | `/proc/cpuinfo` flags |
| ARM TrustZone | OP-TEE / TEE | `/dev/tee0` |
| Intel TXT | tboot integration | `/sys/kernel/security/` |

---

#### 2. Register Dump Techniques

الـ LSM نفسه مش بيتعامل مع hardware registers مباشرة، لكن لو بتحقق من hardware security boundaries:

```bash
# قراءة physical memory (لو kernel config يسمح)
# devmem2 بيقرأ physical address
devmem2 0xFED40000 w  # مثال: TPM MMIO base

# استخدام /dev/mem (محتاج CONFIG_DEVMEM + no lockdown)
dd if=/dev/mem bs=4 count=1 skip=$((0xFED40000/4)) 2>/dev/null | xxd

# مشاهدة الـ MSR registers المتعلقة بالأمان
rdmsr 0x3A  # IA32_FEATURE_CONTROL (Intel VMX/TXT lock)
# Output: 0x5 = locked + VMX enabled

# CR4 register لمعرفة SMEP/SMAP
# (محتاج kernel module أو crash dump analysis)
crash /proc/kcore /usr/lib/debug/boot/vmlinux-$(uname -r)
# ثم في crash shell:
# rd cr4
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ LSM بطبيعته software-only، لكن لو بتـ debug hardware security modules زي TPM أو Secure Elements:

```
TPM SPI/I2C Bus Analysis:
─────────────────────────
Logic Analyzer Setup:
  - SPI: MOSI، MISO، CLK، CS — sample at 4x the bus clock
  - I2C: SDA، SCL — trigger on START condition

Protocol Decode:
  - ابحث عن TPM2_PCR_Read command: 80 01 00 00 00 0E ...
  - ابحث عن TPM2_PCR_Extend command: 80 01 00 00 00 ...
  - الـ response الناجح بيبدأ بـ: 80 01 00 00 00 0A 00 00 00 00

IMA Measurement Flow:
  kernel → security_inode_permission() → IMA hook
         → TPM extend (PCR[10]) → kernel log
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | النمط في kernel log | التحقق |
|---------|---------------------|--------|
| TPM غير متاح | `integrity: Unable to open file: /etc/keys/x509_ima.der` | `ls /dev/tpm*` |
| Secure Boot disabled مع Lockdown | `Lockdown: <cmd>: <reason> is restricted` | `mokutil --sb-state` |
| EVM key مش محمّل | `EVM: HMAC attrs: 0x1` أو `evm: <file>: EVM not initialized` | `keyctl show` |
| TPM PCR extend فشل | `IMA: Error Communicating to TPM chip` | `tpm2_pcrread` |
| Hardware RNG غير متاح | `random: crng init done` متأخر جداً | `cat /proc/sys/kernel/random/entropy_avail` |

---

#### 5. Device Tree Debugging للـ Embedded Systems

في الـ embedded systems اللي بتشغّل LSM (زي Android kernel):

```bash
# تحقق من الـ DT nodes المتعلقة بالأمان
ls /proc/device-tree/firmware/
cat /proc/device-tree/firmware/android/compatible 2>/dev/null

# على ARM مع TrustZone
ls /proc/device-tree/firmware/optee/
cat /proc/device-tree/firmware/optee/compatible 2>/dev/null
# يفترض يطلع: linaro,optee-tz

# تحقق من الـ memory reservations للـ secure world
cat /proc/iomem | grep -i 'secure\|trustzone\|tee'

# dtc لتحويل الـ DTB للـ human-readable
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A5 'security\|optee\|tpm'
```

```bash
# تحقق إن الـ DT فاعل ومتوافق مع الـ hardware
dmesg | grep -E 'OF:|DT:|device-tree'

# لو الـ TPM node مش موجود في DT
dmesg | grep -i tpm
# لو طلع: "tpm_tis: No TPM chip found" مع وجود hardware:
# → افحص الـ DT node: compatible = "tcg,tpm_tis-spi"
```

---

### Practical Commands

#### أوامر جاهزة للنسخ والتشغيل

**1. فحص سريع للـ LSM state:**
```bash
#!/bin/bash
echo "=== Active LSMs ==="
cat /sys/kernel/security/lsm

echo -e "\n=== SELinux Status ==="
sestatus 2>/dev/null || echo "SELinux not available"

echo -e "\n=== AppArmor Status ==="
apparmor_status 2>/dev/null || echo "AppArmor not available"

echo -e "\n=== Recent AVC Denials (last 20) ==="
dmesg | grep 'avc:' | tail -20

echo -e "\n=== Recent AppArmor Denials (last 20) ==="
dmesg | grep 'apparmor="DENIED"' | tail -20

echo -e "\n=== Security Contexts of running processes ==="
ps -eZ | head -20
```

**2. تتبع hook معين بـ ftrace:**
```bash
#!/bin/bash
HOOK=${1:-security_file_open}   # hook الافتراضي
TRACEDIR=/sys/kernel/debug/tracing

echo 0 > $TRACEDIR/tracing_on
echo > $TRACEDIR/trace                        # امسح الـ buffer
echo function_graph > $TRACEDIR/current_tracer
echo "$HOOK" > $TRACEDIR/set_graph_function
echo stacktrace > $TRACEDIR/trace_options     # stack مع كل call
echo 1 > $TRACEDIR/tracing_on

echo "Tracing $HOOK for 5 seconds..."
sleep 5

echo 0 > $TRACEDIR/tracing_on
echo "=== Results ==="
cat $TRACEDIR/trace | head -100
```

**3. تشخيص selinux denial:**
```bash
#!/bin/bash
PID=${1:-$$}
echo "=== Context of PID $PID ==="
cat /proc/$PID/attr/current

echo -e "\n=== Last 10 AVC denials for this process ==="
COMM=$(cat /proc/$PID/comm 2>/dev/null)
ausearch -m avc -ts today 2>/dev/null | grep "comm=\"$COMM\"" | tail -10

echo -e "\n=== Suggested allow rules ==="
ausearch -m avc -ts today 2>/dev/null | grep "comm=\"$COMM\"" | audit2allow 2>/dev/null
```

**4. فحص الـ BPF LSM programs:**
```bash
#!/bin/bash
echo "=== BPF LSM programs ==="
bpftool prog list type lsm 2>/dev/null || \
    bpftool prog list | grep -A2 'lsm'

echo -e "\n=== LSM Hook attachments ==="
bpftool link list 2>/dev/null | grep lsm
```

**5. مراقبة realtime لكل الـ security events:**
```bash
# terminal منفصل: شغّل الـ tracing
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_execve/enable
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_open/enable
# أضف security tracepoints لو متاحة
ls /sys/kernel/debug/tracing/events/lsm/ 2>/dev/null && \
    echo 1 > /sys/kernel/debug/tracing/events/lsm/enable

# تابع الـ output
cat /sys/kernel/debug/tracing/trace_pipe
```

---

#### قراءة الـ Output وتفسيره

**مثال output لـ AVC denial وتفسيره:**
```
avc:  denied  { read } for  pid=1337 comm="nginx" name="secret.conf"
      dev="sda1" ino=98765 scontext=system_u:system_r:httpd_t:s0
      tcontext=system_u:object_r:shadow_t:s0 tclass=file permissive=0
```

| الحقل | القيمة | التفسير |
|-------|--------|---------|
| `denied { read }` | `read` | الـ permission المرفوض |
| `comm="nginx"` | nginx | البرنامج اللي حاول |
| `scontext=httpd_t` | httpd_t | الـ SELinux type للـ process |
| `tcontext=shadow_t` | shadow_t | الـ SELinux type للـ file |
| `permissive=0` | 0 | enforcing — الـ operation اتمنعت فعلاً |

**الحل:**
```bash
# ولّد rule تسمح لـ httpd_t يقرأ shadow_t
echo 'allow httpd_t shadow_t:file { read };' >> /etc/selinux/targeted/policy/modules/myfix.te
make -f /usr/share/selinux/devel/Makefile myfix.pp
semodule -i myfix.pp
```

**مثال output لـ ftrace function_graph:**
```
 1)               |  security_file_open() {
 1)               |    selinux_file_open() {
 1)               |      selinux_inode_permission() {
 1)   0.342 us    |        avc_has_perm_noaudit();
 1)   1.234 us    |      } /* selinux_inode_permission */
 1)   0.123 us    |      file_has_perm();
 1)   2.891 us    |    } /* selinux_file_open */
 1)   3.445 us    |  } /* security_file_open */
```

**الـ output ده بيوضح:**
- الـ `security_file_open` استغرق `3.445 µs` كلها
- الـ SELinux hook نفسه أخد `2.891 µs`
- الـ `avc_has_perm_noaudit` هو أسرع جزء (cache hit)
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — LSM مش بيتحمّل بالترتيب الصح

#### العنوان
LSM hook مش بيتسجّل صح وبيعمل kernel panic في بداية التشغيل

#### السياق
شركة بتبني **industrial gateway** على **AM62x (TI)** بيشتغل Linux مع SELinux. الفريق محتاج يضيف LSM مخصوص بيمنع process بعينها من الوصول لـ UART السيريال (ttyS2) اللي بيتحكم في PLC. كتبوا الـ LSM جديد، شغّلوه، والـ kernel بيتعلق في البداية.

#### المشكلة
الـ LSM الجديد بيعمل `security_add_hooks()` قبل ما الـ `static_calls_table` تتهيأ لأن الـ `init()` callback بيتشغّل بدري أوي.

#### التحليل
في `lsm_hooks.h`:

```c
/* الـ static_calls_table بيتحمّل بعد كل الـ LSMs يتسجّلوا */
extern struct lsm_static_calls_table static_calls_table __ro_after_init;
```

الـ macro `DEFINE_EARLY_LSM` بيحط الـ `lsm_info` في section `.early_lsm_info.init`:

```c
#define DEFINE_EARLY_LSM(lsm)                       \
    static struct lsm_info __early_lsm_##lsm        \
        __used __section(".early_lsm_info.init")    \
        __aligned(sizeof(unsigned long))
```

المهندس غلط واستخدم `DEFINE_EARLY_LSM` بدل `DEFINE_LSM`. الفرق إن الـ early LSMs بيتشغّلوا قبل ما الـ `static_calls_table` تكون جاهزة. لما `security_add_hooks()` بتحاول تكتب في `static_calls_table.NAME[i]`، الـ pointer غير valid وبيحصل panic.

```c
/*
 * lsm_info.order بيتحكم في التسلسل:
 * FIRST  → capabilities بس
 * MUTABLE → الـ LSMs العادية
 * LAST   → integrity (IMA/EVM)
 */
enum lsm_order {
    LSM_ORDER_FIRST  = -1,
    LSM_ORDER_MUTABLE = 0,
    LSM_ORDER_LAST   = 1,
};
```

الـ `struct lsm_info` فيها `order` field مهمة:

```c
struct lsm_info {
    const struct lsm_id *id;
    enum lsm_order order;       /* لازم MUTABLE للـ LSM العادي */
    unsigned long flags;
    struct lsm_blob_sizes *blobs;
    int *enabled;
    int (*init)(void);          /* ده اللي بيتشغّل عند التهيئة */
    /* ... */
};
```

#### الحل

```c
/* غلط — بيتشغّل بدري أوي */
DEFINE_EARLY_LSM(uart_guard) = {
    .id    = &uart_guard_lsmid,
    .order = LSM_ORDER_MUTABLE,
    .init  = uart_guard_init,
};

/* صح — بيستنى الـ framework يتهيأ */
DEFINE_LSM(uart_guard) = {
    .id    = &uart_guard_lsmid,
    .order = LSM_ORDER_MUTABLE,
    .init  = uart_guard_init,
};
```

للتأكد من ترتيب التحميل:

```bash
# اشوف الـ LSMs المتفعّلة وترتيبها
cat /sys/kernel/security/lsm

# output متوقع
capability,selinux,uart_guard
```

#### الدرس المستفاد
`DEFINE_EARLY_LSM` مخصوصة لـ LSMs زي capabilities اللي محتاجة تتشغّل قبل كل حاجة. أي LSM مخصوص بيحتاج hooks عادية لازم يستخدم `DEFINE_LSM` مع `LSM_ORDER_MUTABLE`.

---

### السيناريو 2: Android TV Box على RK3562 — تعارض blob sizes بين SELinux وـ LSM مخصوص

#### العنوان
Heap corruption بسبب `lsm_blob_sizes` غلط لـ inode blob

#### السياق
فريق بيطوّر **Android TV box** على **RK3562** مع Android kernel. الجهاز بيشغّل SELinux كـ mandatory LSM، والفريق أضاف LSM تاني بيحفظ metadata للـ HDMI content protection. الجهاز بيشغّل وبعد ساعات بيحصل memory corruption عشوائي.

#### المشكلة
الـ LSM الجديد طلب `lbs_inode = sizeof(struct hdmi_sec_blob)` لكن مش بيحسب إن SELinux كمان بيطلب space في نفس الـ inode blob. النتيجة: الاتنين بيكتبوا على نفس الـ offset وبيخربوا بعض.

#### التحليل
الـ `struct lsm_blob_sizes` بيستخدمه كل LSM يعلن عن المساحة اللي محتاجها:

```c
struct lsm_blob_sizes {
    unsigned int lbs_cred;
    unsigned int lbs_file;
    unsigned int lbs_ib;
    unsigned int lbs_inode;      /* المساحة المطلوبة في inode security blob */
    unsigned int lbs_sock;
    unsigned int lbs_superblock;
    unsigned int lbs_ipc;
    unsigned int lbs_key;
    unsigned int lbs_msg_msg;
    unsigned int lbs_perf_event;
    unsigned int lbs_task;
    unsigned int lbs_xattr_count;
    unsigned int lbs_tun_dev;
    unsigned int lbs_bdev;
    unsigned int lbs_bpf_map;
    unsigned int lbs_bpf_prog;
    unsigned int lbs_bpf_token;
};
```

الـ LSM framework بيجمع كل الـ blob sizes من كل LSM ويخصّص memory واحدة متواصلة. كل LSM بياخد offset مختلف جوه الـ blob. لكن الـ LSM الجديد كان بيحسب الـ offset يدوي غلط:

```c
/* غلط — بيفترض إن inode blob بيبدأ من 0 */
static struct my_inode_blob *get_blob(struct inode *inode)
{
    /* inode->i_security ده pointer للـ blob الكامل */
    return (struct my_inode_blob *)inode->i_security; /* خطأ! */
}

/* صح — لازم يستخدم الـ offset اللي حدده الـ framework */
static struct my_inode_blob *get_blob(struct inode *inode)
{
    return (struct my_inode_blob *)
        ((char *)inode->i_security + my_lsm_blob_sizes.lbs_inode);
}
```

الـ framework بيحدّث `lbs_inode` في كل `lsm_blob_sizes` ليكون الـ offset الصح لكل LSM بعد ما يجمع الـ sizes.

#### الحل

```bash
# افحص حجم الـ blob الكلي
cat /sys/kernel/debug/lsm/blob_sizes 2>/dev/null || \
    dmesg | grep -i "lsm.*blob"

# لو مفيش debug interface، قيّد الـ LSMs المتفعّلة وشوف الفرق
# في kernel cmdline:
# lsm=capability,selinux,hdmi_guard
```

```c
/* الطريقة الصحيحة لتسجيل الـ blob size */
static struct lsm_blob_sizes hdmi_guard_blob_sizes __ro_after_init = {
    .lbs_inode = sizeof(struct hdmi_sec_blob),
};

DEFINE_LSM(hdmi_guard) = {
    .id    = &hdmi_guard_lsmid,
    .order = LSM_ORDER_MUTABLE,
    .blobs = &hdmi_guard_blob_sizes,  /* الـ framework هيحدّث lbs_inode للـ offset */
    .init  = hdmi_guard_init,
};
```

#### الدرس المستفاد
الـ `lbs_*` fields في `lsm_blob_sizes` مش sizes ثابتة — الـ framework بيحوّلها لـ offsets بعد التهيئة. لازم دايمًا تستخدم الـ offset اللي بيرجعه الـ framework مش تحسب يدوي.

---

### السيناريو 3: IoT Sensor على STM32MP1 — IMA hook بيبطّل بداية التشغيل

#### العنوان
IMA failing على flash صغيرة بسبب `MAX_LSM_COUNT` أكبر من اللازم

#### السياق
فريق بيبني **IoT sensor node** على **STM32MP1** مع Linux مقلّص. الذاكرة محدودة (256MB RAM). حاولوا يفعّلوا IMA لـ integrity checking على الـ firmware لكن الجهاز بيأخد وقت طويل جدًا في البداية وبييجي kernel warning.

#### المشكلة
الـ `MAX_LSM_COUNT` كبير (14 LSM) رغم إن الجهاز مفعّلش غير 3 LSMs. كل hook بياخد array بحجم `MAX_LSM_COUNT` في `lsm_static_calls_table`، والـ `.rodata` section بقت كبيرة جدًا.

#### التحليل
في `lsm_count.h`:

```c
/* MAX_LSM_COUNT بيتحسب compile-time بناءً على الـ configs المفعّلة */
#define MAX_LSM_COUNT           \
    COUNT_LSMS(                 \
        CAPABILITIES_ENABLED    \  /* دايمًا 1 */
        SELINUX_ENABLED         \  /* لو CONFIG_SECURITY_SELINUX */
        SMACK_ENABLED           \
        APPARMOR_ENABLED        \
        TOMOYO_ENABLED          \
        YAMA_ENABLED            \
        LOADPIN_ENABLED         \
        LOCKDOWN_ENABLED        \
        SAFESETID_ENABLED       \
        BPF_LSM_ENABLED         \
        LANDLOCK_ENABLED        \
        IMA_ENABLED             \
        EVM_ENABLED             \
        IPE_ENABLED)
```

الـ `lsm_static_calls_table` بتاخد كل الـ hooks وتعمل array بحجم `MAX_LSM_COUNT` لكل واحد:

```c
struct lsm_static_calls_table {
    /* لكل hook: array بحجم MAX_LSM_COUNT من lsm_static_call */
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) \
        struct lsm_static_call NAME[MAX_LSM_COUNT];
    #include <linux/lsm_hook_defs.h>
    #undef LSM_HOOK
} __packed __randomize_layout;
```

لو عندك 250 hook وـ MAX_LSM_COUNT = 14:
- كل `lsm_static_call` = ~32 bytes
- الحجم الكلي = 250 × 14 × 32 = **112 KB** في `.rodata`

على STM32MP1 ده كتير.

#### الحل

```bash
# في .config، disable كل LSM مش محتاجه
# بدل ما تحتفظ بـ SELinux + AppArmor + TOMOYO + كل حاجة:

CONFIG_SECURITY_SELINUX=n
CONFIG_SECURITY_SMACK=n
CONFIG_SECURITY_APPARMOR=n
CONFIG_SECURITY_TOMOYO=n
CONFIG_SECURITY_YAMA=n
CONFIG_SECURITY_LOADPIN=n
CONFIG_SECURITY_LOCKDOWN_LSM=n
CONFIG_SECURITY_SAFESETID=n
CONFIG_BPF_LSM=n
CONFIG_SECURITY_LANDLOCK=n
# فعّل بس اللي محتاجه
CONFIG_IMA=y
CONFIG_EVM=y
```

```bash
# افحص حجم الـ static_calls_table
nm vmlinux | grep static_calls_table
size vmlinux | head -5

# قبل التحسين
# .rodata: 2.1MB

# بعد disable الـ LSMs الزيادة
# .rodata: 1.1MB
```

#### الدرس المستفاد
على الأنظمة المدمجة المحدودة الموارد، `MAX_LSM_COUNT` له تأثير مباشر على حجم الكرنل. لازم تـ disable كل LSM مش محتاجه لتقليل الـ `static_calls_table` size.

---

### السيناريو 4: Automotive ECU على i.MX8 — `__randomize_layout` بيكسر out-of-tree LSM

#### العنوان
Out-of-tree LSM module بيـ crash بعد kernel update بسبب struct layout randomization

#### السياق
فريق automotive بيشتغل على **ECU** على **i.MX8QM** لـ advanced driver assistance system. عندهم LSM proprietary بيتحمّل كـ kernel module (out-of-tree) بيتحكم في الوصول لـ CAN bus. بعد upgrade الكرنل من 6.1 لـ 6.6، المديول بيـ crash فورًا عند التحميل.

#### المشكلة
الـ `struct security_hook_list` و `struct lsm_static_call` و `struct lsm_static_calls_table` كلها معمّلها `__randomize_layout__`. الـ module القديم بيعمل struct initialization بالترتيب الـ hardcoded في source، لكن الترتيب في الكرنل الجديد مختلف.

#### التحليل

```c
/*
 * __randomize_layout يخلي GCC يرتّب الـ fields عشوائيًا
 * لكل kernel build — ده security feature ضد exploits
 */
struct security_hook_list {
    struct lsm_static_call *scalls;
    union security_list_options hook;
    const struct lsm_id *lsmid;
} __randomize_layout;          /* الترتيب بيتغيّر مع كل build */

struct lsm_static_call {
    struct static_call_key *key;
    void *trampoline;
    struct security_hook_list *hl;
    struct static_key_false *active;
} __randomize_layout;
```

الـ out-of-tree module كان بيعمل:

```c
/* غلط — positional initialization بيفشل مع __randomize_layout */
struct security_hook_list my_hooks[] = {
    { static_calls_table.file_open,   /* field 0 */
      { .file_open = my_file_open },  /* field 1 */
      &my_lsmid },                    /* field 2 */
};
```

بعد randomization، الترتيب اتغيّر فـ `.scalls` اتحط في غلط offset.

الحل الصح هو استخدام الـ macro `LSM_HOOK_INIT` اللي بيستخدم designated initializers:

```c
/*
 * LSM_HOOK_INIT بيستخدم named fields (.field_name = value)
 * فمش بيتأثر بـ __randomize_layout
 */
#define LSM_HOOK_INIT(NAME, HOOK)           \
    {                                       \
        .scalls = static_calls_table.NAME,  \
        .hook   = { .NAME = HOOK }          \
    }

/* صح — designated initializers */
struct security_hook_list my_hooks[] = {
    LSM_HOOK_INIT(file_open, my_file_open),
    LSM_HOOK_INIT(inode_permission, my_inode_permission),
};
```

#### الحل

```bash
# اتأكد إن CONFIG_RANDSTRUCT مفعّل
grep CONFIG_RANDSTRUCT /boot/config-$(uname -r)
# CONFIG_RANDSTRUCT=y

# لو out-of-tree module، لازم يتـ build مع نفس الـ kernel headers
# والـ RANDSTRUCT_HASHED_SEED لازم يكون نفسه
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# افحص الـ module symbols
modinfo can_guard.ko | grep -E "vermagic|depends"
```

```c
/* في الـ module، دايمًا استخدم LSM_HOOK_INIT */
static struct security_hook_list can_guard_hooks[] __ro_after_init = {
    LSM_HOOK_INIT(file_open,          can_guard_file_open),
    LSM_HOOK_INIT(socket_sendmsg,     can_guard_socket_send),
    LSM_HOOK_INIT(inode_permission,   can_guard_inode_perm),
};
```

#### الدرس المستفاد
`__randomize_layout__` على structs الـ LSM يعني إن أي positional struct initialization هيكسر مع كل kernel build جديد. دايمًا استخدم `LSM_HOOK_INIT` macro اللي بيستخدم designated initializers وبيكون آمن مع أي ترتيب.

---

### السيناريو 5: Custom Board Bring-up على Allwinner H616 — LSM hook بيتنفّذ بس بيرجع error غلط لـ I2C

#### العنوان
LSM مخصوص بيمنع كل I2C writes عشان بيستخدم DEFAULT value غلط في hook definition

#### السياق
فريق بيعمل **custom media board** على **Allwinner H616** (Orange Pi Zero 2). الجهاز فيه sensor I2C على /dev/i2c-1. المهندس كتب LSM بيـ audit الوصول للـ I2C لأغراض logging. بعد تفعيل الـ LSM، كل الـ I2C writes فشلت بـ `EPERM` حتى من root.

#### المشكلة
المهندس عرّف hook في `lsm_hook_defs.h` المخصوص بيرجع `int` لكن بدون ما يفهم معنى الـ `DEFAULT` value في الـ `LSM_HOOK` macro. الـ DEFAULT هو القيمة اللي بتتعمل لما مفيش LSM مسجّل للـ hook ده. لو DEFAULT = 0 ولكن الـ hook code بيرجع `EPERM` دايمًا، هيمنع كل حاجة.

#### التحليل

الـ `union security_list_options` بيتعمل من `lsm_hook_defs.h`:

```c
/*
 * LSM_HOOK(RET, DEFAULT, NAME, ...)
 * RET     = return type
 * DEFAULT = القيمة الافتراضية لو مفيش hook مسجّل
 * NAME    = اسم الـ hook
 */
union security_list_options {
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) RET (*NAME)(__VA_ARGS__);
    #include "lsm_hook_defs.h"
    #undef LSM_HOOK
    void *lsm_func_addr;
};
```

المهندس كتب hook implementation غلط:

```c
/* غلط — بيمنع كل حاجة بدون سبب */
static int i2c_audit_file_open(struct file *file)
{
    const char *name = file->f_path.dentry->d_name.name;

    /* قصد المهندس يـ audit بس I2C files */
    if (strstr(name, "i2c")) {
        pr_info("i2c_audit: access to %s\n", name);
        /* نسي يرجع 0 هنا! */
    }
    return -EPERM; /* ده بيتنفّذ دايمًا! */
}

/* صح */
static int i2c_audit_file_open(struct file *file)
{
    const char *name = file->f_path.dentry->d_name.name;

    if (strstr(name, "i2c"))
        pr_info("i2c_audit: access to %s\n", name);

    return 0; /* السماح — الـ audit بس */
}
```

كمان المهندس مفهمش إن الـ hooks بتشتغل بالتسلسل. لو LSM واحد رجّع error، الـ framework مش بيكمّل:

```
LSM Call Chain للـ file_open hook:
┌─────────────┐    ┌─────────────┐    ┌───────────────┐
│ capability  │ -> │  SELinux    │ -> │  i2c_audit    │
│   ret=0     │    │   ret=0     │    │   ret=-EPERM  │
└─────────────┘    └─────────────┘    └───────────────┘
                                              │
                                    file_open يفشل بـ EPERM
```

#### الحل

```bash
# اتأكد من الـ LSM chain
cat /sys/kernel/security/lsm
# capability,selinux,i2c_audit

# افحص الـ audit logs
dmesg | grep i2c_audit

# للـ debug، disable الـ LSM مؤقتًا من kernel cmdline
# أضف لـ /boot/extlinux/extlinux.conf:
# APPEND root=/dev/mmcblk0p2 lsm=capability,selinux

# اتأكد إن I2C بيشتغل بدون الـ LSM
i2cdetect -y 1
```

```c
/* الـ hook الصح للـ audit بدون منع */
static int i2c_audit_file_open(struct file *file)
{
    const char *path;
    char buf[64];

    path = dentry_path_raw(file->f_path.dentry, buf, sizeof(buf));
    if (!IS_ERR(path) && strstr(path, "i2c"))
        pr_info("i2c_audit: PID %d opened %s\n", current->pid, path);

    /* لازم ترجع 0 لو مش عايز تمنع */
    return 0;
}

/* تسجيل الـ hook */
static struct security_hook_list i2c_audit_hooks[] __ro_after_init = {
    LSM_HOOK_INIT(file_open, i2c_audit_file_open),
};

static int __init i2c_audit_init(void)
{
    security_add_hooks(i2c_audit_hooks,
                       ARRAY_SIZE(i2c_audit_hooks),
                       &i2c_audit_lsmid);
    pr_info("i2c_audit LSM: initialized\n");
    return 0;
}
```

```bash
# بعد الإصلاح، اتأكد إن I2C بيشتغل
i2cdetect -y 1
#      0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
# 00:          -- -- -- -- -- -- -- -- -- -- -- -- --
# 50: 50 -- -- -- -- -- -- --
# ← الـ sensor ظهر على address 0x50

dmesg | grep i2c_audit
# i2c_audit LSM: initialized
# i2c_audit: PID 1234 opened /dev/i2c-1
```

#### الدرس المستفاد
الـ LSM hooks اللي بترجع `int` لازم دايمًا ترجع 0 لو قصدك تـ observe بس مش تمنع. رجوع أي قيمة سالبة هيمنع العملية لكل الـ processes. استخدم `DEFAULT` value في `LSM_HOOK` كمرجع لفهم الـ semantics المتوقعة من الـ hook.
## Phase 7: مصادر ومراجع

### توثيق رسمي للـ Kernel

| المصدر | الرابط |
|--------|--------|
| **LSM General Security Hooks** — التوثيق الرسمي في kernel.org | [docs.kernel.org/security/lsm.html](https://docs.kernel.org/security/lsm.html) |
| **LSM Usage (admin-guide)** — دليل استخدام الـ LSM | [static.lwn.net/kerneldoc/admin-guide/LSM/](https://static.lwn.net/kerneldoc/admin-guide/LSM/) |
| **AppArmor Documentation** | [docs.kernel.org/admin-guide/LSM/apparmor.html](https://docs.kernel.org/admin-guide/LSM/apparmor.html) |
| **ملف `lsm_hooks.h` على GitHub** | [github.com/torvalds/linux/.../lsm_hooks.h](https://github.com/torvalds/linux/blob/master/include/linux/lsm_hooks.h) |
| **Documentation/security/lsm.txt** (نسخة قديمة) | [kernel.org/doc/Documentation/lsm.txt](https://www.kernel.org/doc/Documentation/lsm.txt) |

مسارات التوثيق داخل source tree:

```
Documentation/security/lsm.rst          ← الوصف العام لـ LSM framework
Documentation/security/lsm-development.rst ← دليل كتابة LSM جديد
include/linux/lsm_hooks.h              ← تعريف الـ hooks (الملف المدروس)
include/linux/lsm_hook_defs.h          ← قائمة كل hook بـ X-macro
include/linux/security.h               ← الـ API العام للـ kernel
security/security.c                    ← dispatcher وتسجيل الـ hooks
```

---

### مقالات LWN.net الأساسية

الـ **LWN.net** هو المرجع الأهم لفهم تطور الـ LSM — كل مقال بيشرح مرحلة تطور مختلفة:

| المقال | الأهمية |
|--------|---------|
| [Linux Security Modules: General Security Hooks for Linux](https://static.lwn.net/kerneldoc/security/lsm.html) | التوثيق الأصلي لـ LSM framework — نقطة البداية |
| [Eliminating indirect calls for security modules](https://lwn.net/Articles/979683/) | أحدث تطور: استبدال الـ indirect calls بـ static calls لتحسين الأداء |
| [Reduce overhead of LSMs with static calls](https://lwn.net/Articles/944883/) | المرحلة الأولى من تحويل LSM hooks لـ static calls |
| [A change in direction for security-module stacking?](https://lwn.net/Articles/970070/) | نقاش مهم عن مستقبل الـ LSM stacking |
| [LSM stacking and the future](https://lwn.net/Articles/804906/) | تحليل عميق لمشكلة stacking قبل الحل الحالي |
| [LSM: Module stacking for AppArmor](https://lwn.net/Articles/837994/) | الـ patchset الرئيسي لـ stacking (Casey Schaufler) |
| [Progress in security module stacking](https://lwn.net/Articles/635771/) | تاريخ تطور الـ stacking من v2 إلى v4 kernel |
| [The future of the Linux Security Module API](https://lwn.net/Articles/180184/) | نقاش قديم ومهم عن مسار الـ LSM API |
| [Safe LSM (un)loading, and immutable hooks](https://lwn.net/Articles/752779/) | آلية load/unload آمنة للـ LSM بعد boot |
| [Complete coverage in Linux security modules](https://lwn.net/Articles/154277/) | نقاش coverage الـ hooks — أي عمليات محمية وأيها لا |
| [LSM: Three basic syscalls](https://lwn.net/Articles/928790/) | الـ syscalls الجديدة لـ LSM introspection |
| [A security-module hook for user-namespace creation](https://lwn.net/Articles/903580/) | إضافة hook لـ user namespace — مثال عملي على توسيع الـ API |
| [Rework LSM hooks](https://lwn.net/Articles/97267/) | إعادة هيكلة قديمة وتاريخية للـ hooks |
| [Static calls](https://lwn.net/Articles/771209/) | شرح الـ static call mechanism اللي بتستخدمه الـ LSM |
| [Avoiding retpolines with static calls](https://lwn.net/Articles/815908/) | السبب الأمني وراء استخدام static calls |
| [Impedance matching for BPF and LSM](https://lwn.net/Articles/813261/) | تكامل BPF مع LSM hooks |

---

### Mailing List Discussions

نقاشات مهمة على قوائم البريد الرسمية:

| الموضوع | الرابط |
|---------|--------|
| **[PATCH 00/58] LSM: Module stacking for AppArmor** — أكبر patchset في تاريخ الـ LSM | [lore.kernel.org/selinux/20190602...](https://lore.kernel.org/selinux/20190602165101.25079-14-casey@schaufler-ca.com/t/) |
| **[PATCH v28 00/25] LSM: Module stacking for AppArmor** — النسخة النهائية | [lore.kernel.org/selinux/20210722...](https://lore.kernel.org/selinux/20210722004758.12371-15-casey@schaufler-ca.com/T/) |
| **[RFC] security: replace indirect calls with static calls** — بداية نقاش الـ static calls | [lkml.org/lkml/2020/8/20/2197](https://lkml.org/lkml/2020/8/20/2197) |
| **[PATCH] security: Add LSM hook at fatal signal** — مثال إضافة hook جديد | [openwall.com/lists/kernel-hardening/2021/06/05/2](https://openwall.com/lists/kernel-hardening/2021/06/05/2) |
| **[v13,00/25] LSM: Module stacking for AppArmor** — Patchwork tracking | [patchwork.kernel.org/project/selinux/...](https://patchwork.kernel.org/project/selinux/cover/20191224235939.7483-1-casey@schaufler-ca.com/) |

---

### KernelNewbies.org

| الصفحة | الفائدة |
|--------|---------|
| [KernelGlossary — LSM entry](https://kernelnewbies.org/KernelGlossary) | تعريف مختصر وسريع لمصطلح LSM |
| [Linux_2_6_18 changes](https://kernelnewbies.org/Linux_2_6_18) | تغييرات تاريخية في الـ LSM عند إضافة SELinux improvements |
| [Linux_4.15 changes](https://kernelnewbies.org/Linux_4.15) | تغييرات LSM في kernel 4.15 |
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | تتبع تغييرات الـ LSM عبر كل إصدار kernel |
| [RFC PATCH: Brute LSM](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2020-December/021288.html) | مثال عملي لكتابة LSM جديد من الصفر |

---

### الورقة البحثية الأصلية

الورقة اللي وضعت أساس الـ LSM framework:

> **"Linux Security Module Framework"**
> Chris Wright و Crispin Cowan — Ottawa Linux Symposium 2002
> [kroah.com/linux/talks/ols_2002_lsm_paper/lsm.pdf](http://www.kroah.com/linux/talks/ols_2002_lsm_paper/lsm.pdf)

الورقة دي بتشرح القرارات التصميمية الأصلية: ليه اختاروا hook-based approach، وإزاي قرروا أين يحطوا الـ hooks في الـ kernel.

---

### Wikipedia & General Overviews

| المصدر | الرابط |
|--------|--------|
| Wikipedia: Linux Security Modules | [en.wikipedia.org/wiki/Linux_Security_Modules](https://en.wikipedia.org/wiki/Linux_Security_Modules) |
| Star Lab: A Brief Tour of Linux Security Modules | [starlab.io/blog/a-brief-tour-of-linux-security-modules](https://www.starlab.io/blog/a-brief-tour-of-linux-security-modules/) |
| AccuKnox: LSM hooks explained | [accuknox.com/blog/linux-security-modules-lsm-hooks](https://accuknox.com/blog/linux-security-modules-lsm-hooks) |
| KubeArmor: Introduction to LSMs | [github.com/kubearmor/KubeArmor/wiki/Introduction-to-Linux-Security-Modules](https://github.com/kubearmor/KubeArmor/wiki/Introduction-to-Linux-Security-Modules-(LSMs)) |

---

### الكتب الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)

الكتاب مجاني online، لكن الـ LSM مش موضوعه الأساسي. مفيد لفهم:
- **Chapter 1**: البنية العامة للـ kernel
- **Chapter 5**: Concurrency وأهميتها للـ security hooks

> [lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/) — متاح مجاناً

#### Linux Kernel Development — Robert Love (3rd Edition)

الكتاب الأفضل لفهم السياق اللي بيشتغل فيه الـ LSM:

| الفصل | الصلة بـ LSM |
|-------|-------------|
| **Chapter 3** — Process Management | فهم `task_struct` اللي الـ LSM بيضيفله security blob |
| **Chapter 9** — An Introduction to Kernel Synchronization | الـ RCU اللي بيحمي قوائم الـ hooks |
| **Chapter 17** — Devices and Modules | فهم الـ module infrastructure المستخدمة في الـ LSM |

#### Understanding the Linux Kernel — Bovet & Cesati (3rd Edition)

- **Chapter 20**: System Calls — فهم نقاط الـ hook في syscall layer
- الفصول المتعلقة بـ VFS لفهم الـ inode/file hooks

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

مفيد لفهم:
- تقليل footprint الـ LSM في embedded systems
- اختيار الـ LSM المناسب (SELinux vs AppArmor vs SMACK) لـ embedded

---

### Kernel Source Files المهمة

الملفات دي مرتبطة مباشرة بـ `lsm_hooks.h`:

```
include/linux/lsm_hooks.h        ← الملف المدروس — الـ structures الأساسية
include/linux/lsm_hook_defs.h    ← قائمة كل hook بـ X-macro style
include/linux/security.h         ← الـ wrappers العامة اللي بتستدعي الـ hooks
include/uapi/linux/lsm.h         ← الـ IDs والـ attributes المكشوفة لـ userspace
security/security.c              ← التنفيذ: تسجيل وتنفيذ الـ hooks
security/Kconfig                 ← إعدادات الـ build للـ LSMs
security/selinux/hooks.c         ← مثال: تسجيل hooks في SELinux
security/apparmor/lsm.c          ← مثال: تسجيل hooks في AppArmor
security/bpf/hooks.c             ← مثال: BPF LSM hooks
```

---

### Search Terms للبحث عن معلومات إضافية

استخدم المصطلحات دي في Google أو lore.kernel.org أو LWN:

```
lsm_hooks.h security_hook_list kernel
LSM stacking linux kernel
security_add_hooks linux
lsm_static_call kernel performance
linux security module hook registration
LSM blob management kernel
BPF LSM program kernel
selinux apparmor smack stacking kernel
linux security module audit subsystem
lsm_id kernel uapi
MAX_LSM_COUNT kernel config
```

للبحث في الـ mailing list الرسمي:

```bash
# البحث في lore.kernel.org
https://lore.kernel.org/linux-security-module/

# البحث في LKML
https://lkml.org/lkml/2024/  # استبدل السنة
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `lsm_hooks.h` بيعرّف الـ hook الأساسي اللي بيشتغل وقت كل `execve()` وهو `bprm_check_security`. ده الـ hook اللي بيعدي عليه كل LSM (زي SELinux وAppArmor) عشان يقرر يسمح بالتنفيذ أو لا. هنستخدم **kprobe** على الـ kernel function `security_bprm_check` اللي هي نقطة الدخول اللي بتستدعي الـ hook ده، عشان نطبع اسم الـ binary اللي بيتنفذ مع PID العملية.

---

### الـ Hook المختار

| الـ Hook | الـ Kernel Function | الـ Trigger |
|---|---|---|
| `bprm_check_security` | `security_bprm_check` | كل `execve()` / `execveat()` |

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * lsm_exec_watch.c
 * Kprobe on security_bprm_check — logs every exec attempt
 * with the binary path and the calling PID/comm.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/binfmts.h>     /* struct linux_binprm — holds exec info */
#include <linux/sched.h>       /* current, task_pid_nr */
#include <linux/printk.h>      /* pr_info */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("LSM Watcher");
MODULE_DESCRIPTION("Kprobe on security_bprm_check to log every exec via LSM hook");

/* ---------------------------------------------------------------------------
 * Pre-handler: runs just BEFORE security_bprm_check executes.
 * pt_regs holds the CPU registers at call time — first arg (rdi on x86-64)
 * is the pointer to struct linux_binprm.
 * --------------------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64, the first function argument lives in regs->di.
     * security_bprm_check(struct linux_binprm *bprm) — so regs->di = bprm.
     */
    struct linux_binprm *bprm = (struct linux_binprm *)regs->di;

    if (!bprm || !bprm->filename)
        return 0;

    /*
     * current  → the task that called execve()
     * bprm->filename → full path of the binary being executed
     * task_pid_nr(current) → PID of the calling process
     */
    pr_info("lsm_exec_watch: pid=%d comm=%-16s exec=%s\n",
            task_pid_nr(current),
            current->comm,
            bprm->filename);

    return 0; /* non-zero here would cause a fault — always return 0 */
}

/* ---------------------------------------------------------------------------
 * kprobe descriptor — we attach to security_bprm_check by name.
 * The kernel's kallsyms resolves the symbol to its address automatically.
 * --------------------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "security_bprm_check",
    .pre_handler = handler_pre,
};

/* ---------------------------------------------------------------------------
 * module_init: registers the kprobe.
 * If registration fails (symbol not found, kprobes disabled) we return the
 * error so insmod reports it cleanly.
 * --------------------------------------------------------------------------- */
static int __init lsm_watch_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("lsm_exec_watch: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("lsm_exec_watch: hooked security_bprm_check at %p\n", kp.addr);
    return 0;
}

/* ---------------------------------------------------------------------------
 * module_exit: MUST unregister the kprobe before the module is unloaded.
 * Without this, the kernel still holds a pointer to handler_pre which will
 * disappear from memory → guaranteed kernel panic on next execve().
 * --------------------------------------------------------------------------- */
static void __exit lsm_watch_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("lsm_exec_watch: unhooked security_bprm_check\n");
}

module_init(lsm_watch_init);
module_exit(lsm_watch_exit);
```

---

### Makefile

```makefile
obj-m += lsm_exec_watch.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| الـ Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kprobes.h` | تعريف `struct kprobe` ودوال التسجيل |
| `linux/binfmts.h` | تعريف `struct linux_binprm` اللي فيها بيانات الـ exec |
| `linux/sched.h` | الـ `current` macro و`task_pid_nr()` |
| `linux/printk.h` | `pr_info` / `pr_err` |

**الـ** `binfmts.h` ضروري لأن `struct linux_binprm` هي الـ struct اللي بتحمل كل بيانات العملية الجديدة اللي بتتنفذ، زي اسم الملف والـ credentials.

---

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

**الـ** `pt_regs` بيحتوي على قيم الـ registers لحظة الاستدعاء. على `x86-64`، الـ calling convention بيحط أول argument في `rdi` اللي بيقابله `regs->di` في الكود، وده بيخلينا نعمل cast للـ `bprm` من غير ما نحتاج تعديل في الـ function الأصلية.

الـ `return 0` إجباري — لو رجعنا غير صفر الـ kprobe infrastructure هيعتبره fault ويعمل `SIGSEGV` للـ task.

---

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "security_bprm_check",
    .pre_handler = handler_pre,
};
```

**الـ** `.symbol_name` بيخلي الـ kernel يبحث عن العنوان تلقائياً عن طريق `kallsyms` وقت التسجيل، فمش محتاجين نحدد عنوان hardcoded. الـ `.pre_handler` بيشتغل قبل تنفيذ الـ function الأصلية عشان نشوف الـ arguments قبل ما أي LSM يشوفها.

---

#### الـ `module_init` / `module_exit`

**الـ** `register_kprobe` بيحط **breakpoint** في ذاكرة الـ kernel عند عنوان الـ function المستهدفة. لما الـ CPU يوصل لده البريكبوينت، بيشتغل الـ `handler_pre` الأول. `unregister_kprobe` في الـ exit ضروري جداً لأن لو نسيناه ورفعنا الـ module، الـ kernel هيحاول يستدعي دالة محذوفة من الذاكرة وده بيعمل kernel panic.

---

### تجربة الـ Module

```bash
# بناء الـ module
make

# تحميل الـ module
sudo insmod lsm_exec_watch.ko

# تنفيذ أي أمر عشان نشوف الـ output
ls /tmp

# مشاهدة الـ log
sudo dmesg | tail -20
# المتوقع:
# lsm_exec_watch: pid=1234 comm=bash             exec=/usr/bin/ls

# إزالة الـ module
sudo rmmod lsm_exec_watch
```

---

### مثال على الـ Output

```
[  142.381204] lsm_exec_watch: hooked security_bprm_check at ffffffff813a2c10
[  145.002311] lsm_exec_watch: pid=2847 comm=bash             exec=/usr/bin/ls
[  145.118432] lsm_exec_watch: pid=2848 comm=bash             exec=/usr/bin/cat
[  146.330021] lsm_exec_watch: pid=2849 comm=sshd             exec=/bin/bash
[  147.441102] lsm_exec_watch: unhooked security_bprm_check
```

كل سطر فيه الـ PID والـ comm (اسم العملية الأم) ومسار الـ binary الجديد — بالظبط اللي LSM نفسه بيشوفه لحظة القرار.
