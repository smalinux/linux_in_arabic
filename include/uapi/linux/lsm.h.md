## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: Linux Security Module (LSM) Framework

الـ file ده جزء من **LSM Framework** — الـ subsystem المسؤول عن كل الأمان في الـ Linux kernel. موجود في MAINTAINERS تحت قسم يشمل `include/uapi/linux/lsm.h` و `security/` و `include/linux/security.h`.

---

### المشكلة اللي بيحلها بالقصة

تخيل عندك **بواب** في عمارة. الدور بتاعه إنه يقرر مين يدخل ومين ميدخلش. في Linux، ده الـ kernel بيحميه باستخدام **security modules** زي SELinux و AppArmor و Smack.

بس المشكلة القديمة كانت: **مفيش طريقة موحدة** يقدر بيها البرنامج العادي (userspace) يسأل الـ kernel "انت شغال بأنهي security module؟ وإيه هو الـ security context بتاعي دلوقتي؟"

كل module كان عنده طريقته الخاصة — SELinux بيتكلم عبر `/proc/self/attr/current`، و AppArmor برضو كده، لكن كل واحد بلهجة مختلفة. البرنامج اللي محتاج يعرف هو شغال تحت أنهي module كان لازم يتعامل مع كل واحد على حدة.

الحل جه في **2022** مع Casey Schaufler و Intel: إضافة **3 system calls جديدة** بـ API موحد. الـ header ده هو **عقد الاتفاق** بين الـ kernel وأي برنامج في userspace.

---

### الهدف من الـ File

الـ `include/uapi/linux/lsm.h` هو الـ **UAPI header** — يعني ده الـ interface الرسمي اللي بيتكلم فيه الـ userspace مع الـ kernel بخصوص الأمان.

بيعمل 3 حاجات أساسية:

**1. تعريف `struct lsm_ctx`** — الـ data structure اللي بتحمل معلومات الـ security context:

```c
struct lsm_ctx {
    __u64 id;      /* رقم الـ LSM زي LSM_ID_SELINUX = 101 */
    __u64 flags;   /* flags خاصة بالـ LSM نفسه */
    __u64 len;     /* الحجم الكلي للـ struct + أي data تانية */
    __u64 ctx_len; /* حجم الـ ctx بالظبط */
    __u8 ctx[];    /* الـ context نفسه (string أو binary) */
};
```

ده زي **بطاقة هوية أمنية** — فيها مين أصدرها (الـ id)، وإيه هو المحتوى (الـ ctx).

**2. تعريف `LSM_ID_XXX`** — أرقام تعريف كل LSM:

| Macro | القيمة | الـ LSM |
|---|---|---|
| `LSM_ID_CAPABILITY` | 100 | Linux capabilities الأساسية |
| `LSM_ID_SELINUX` | 101 | SELinux (RHEL, Android) |
| `LSM_ID_SMACK` | 102 | Smack (embedded systems) |
| `LSM_ID_APPARMOR` | 104 | AppArmor (Ubuntu, SUSE) |
| `LSM_ID_YAMA` | 105 | Yama (ptrace protection) |
| `LSM_ID_BPF` | 109 | BPF-based security |
| `LSM_ID_LANDLOCK` | 110 | Landlock (unprivileged sandboxing) |
| `LSM_ID_IMA` | 111 | Integrity Measurement Architecture |
| `LSM_ID_EVM` | 112 | Extended Verification Module |
| `LSM_ID_IPE` | 113 | Integrity Policy Enforcement |

**3. تعريف `LSM_ATTR_XXX`** — أنواع الـ attributes اللي ممكن تسأل عنها:

| Macro | المعنى |
|---|---|
| `LSM_ATTR_CURRENT` | الـ context الحالي للـ process |
| `LSM_ATTR_EXEC` | الـ context اللي هيتطبق لما تعمل exec |
| `LSM_ATTR_FSCREATE` | الـ context لما تعمل ملف جديد |
| `LSM_ATTR_KEYCREATE` | الـ context لما تعمل kernel key |
| `LSM_ATTR_PREV` | الـ context السابق قبل ما يتغير |
| `LSM_ATTR_SOCKCREATE` | الـ context لما تعمل socket |

---

### الـ System Calls اللي بتستخدم الـ Header ده

الـ header ده هو الأساس لـ 3 syscalls جديدة عُرّفت في `security/lsm_syscalls.c`:

```
lsm_list_modules()   → "انهي LSMs شغالة دلوقتي؟"
lsm_get_self_attr()  → "إيه الـ security context بتاعي؟"
lsm_set_self_attr()  → "عايز أغير الـ security context بتاعي"
```

---

### سيناريو عملي

تخيل برنامج container runtime زي **runc** محتاج يعرف:
- هل النظام شغال بـ SELinux ولا AppArmor؟
- إيه هو الـ label الأمني الحالي للـ process؟

قبل الـ header ده، كان لازم يقرأ `/proc/self/attr/current` ويحاول يفسره بطريقة كل module. دلوقتي:

```c
/* الأول اسأل الـ modules الشغالة */
lsm_list_modules(ids, &size, 0);

/* بعدين اجيب الـ context */
lsm_get_self_attr(LSM_ATTR_CURRENT, ctx_buf, &ctx_size, 0);

/* الـ ctx بتاعك هيكون فيه lsm_ctx بـ id = LSM_ID_SELINUX مثلاً */
```

---

### الـ Files المكوّنة للـ Subsystem

#### Core Framework
| الـ File | الدور |
|---|---|
| `security/security.c` | قلب الـ LSM framework، بيوزع الـ hooks على الـ modules |
| `security/lsm_syscalls.c` | الـ 3 system calls الجديدة |
| `include/linux/security.h` | الـ kernel-internal API للـ security |
| `include/linux/lsm_hooks.h` | تعريفات الـ hooks والـ structs الداخلية |
| `include/linux/lsm_hook_defs.h` | قائمة كل الـ hooks بأسمائها وأنواعها |
| `include/linux/lsm_count.h` | عدد الـ LSMs المدعومة |

#### الـ UAPI (واجهة الـ Userspace)
| الـ File | الدور |
|---|---|
| `include/uapi/linux/lsm.h` | **الـ file الحالي** — العقد مع الـ userspace |

#### LSM Implementations
| الـ File | الـ LSM |
|---|---|
| `security/selinux/` | SELinux |
| `security/apparmor/` | AppArmor |
| `security/smack/` | Smack |
| `security/yama/` | Yama |
| `security/landlock/` | Landlock |
| `security/bpf/` | BPF LSM |
| `security/ipe/` | IPE |
| `security/loadpin/` | LoadPin |
| `security/safesetid/` | SafeSetID |

#### Testing
| الـ File | الدور |
|---|---|
| `tools/testing/selftests/lsm/lsm_list_modules_test.c` | اختبار `lsm_list_modules` |
| `tools/testing/selftests/lsm/lsm_get_self_attr_test.c` | اختبار `lsm_get_self_attr` |
| `tools/testing/selftests/lsm/lsm_set_self_attr_test.c` | اختبار `lsm_set_self_attr` |

#### Documentation
| الـ File | الدور |
|---|---|
| `Documentation/admin-guide/LSM/` | دليل إدارة كل LSM |
| `rust/kernel/security.rs` | Rust bindings للـ LSM |
## Phase 2: شرح الـ Linux Security Modules (LSM) Framework

### المشكلة — ليه الـ LSM موجود أصلاً؟

الـ kernel في الأصل عنده نموذج أمان واحد بس: **Unix DAC (Discretionary Access Control)** — يعني permissions من نوع `rwxr-xr--`، و uid/gid. ده كافي للاستخدام العادي، بس مش كافي لبيئات زي:

- **Military-grade systems** محتاجة Mandatory Access Control (MAC) زي SELinux
- **Mobile/desktop** محتاجة profile-based confinement زي AppArmor
- **Embedded systems** محتاجة path-based restrictions بسيطة زي TOMOYO
- **Container runtimes** محتاجة per-process syscall filtering زي Landlock

المشكلة كانت إن كل حل بيحاول يبقى هو الـ security model الوحيد، والـ kernel مش قادر يدعم الاتنين في نفس الوقت — لازم تختار SELinux **أو** AppArmor في الـ kernel config.

فضلاً عن ده، لو حد عايز يضيف security policy جديدة، كان لازم يعدّل الـ kernel source مباشرة في أماكن كتير (فضلاً عن مراجعة الـ Linus Torvalds نفسه لكل تعديل).

---

### الحل — مدخل الـ LSM

الـ **Linux Security Modules (LSM)** هو framework بيوفر **hook-based plugin architecture** جوه الـ kernel.

الفكرة الجوهرية:
1. في نقاط معينة (hooks) جوه الـ kernel (فتح ملف، إنشاء socket، execve...)، الـ kernel بيسأل كل LSM مسجّل: "هل هذا العملية مسموح بيها؟"
2. كل LSM بيرد بـ `0` (مسموح) أو error code (مرفوض).
3. لو أي LSM رفض → العملية اترفضت.

ده بيخلي:
- أكتر من LSM يشتغلوا **في نفس الوقت** (stacking).
- كل LSM معزول عن التاني.
- إضافة LSM جديد من غير تغيير الـ kernel core code.

---

### تشبيه من الواقع — مكتب الجمارك متعدد المراحل

تخيل مطار فيه أكتر من نقطة تفتيش:

| نقطة التفتيش | مقابلها في LSM |
|---|---|
| حارس الباب الخارجي (Capabilities) | `LSM_ID_CAPABILITY` — أول LSM يشتغل دايماً |
| موظف الجوازات (SELinux/AppArmor) | LSM رئيسي بيطبق MAC policy |
| موظف الجمارك الثاني (Yama) | LSM إضافي بيراقب ptrace |
| الكاميرات الذكية (BPF LSM) | LSM بيخليك تكتب policy بنفسك بـ eBPF |

كل مسافر (system call) لازم يعدي **كل** نقاط التفتيش بالترتيب. لو أي واحد رفض → ممنوع.

تطبيق التشبيه على الكود:
- **المطار = الـ kernel** — بيوفر الـ infrastructure
- **نقاط التفتيش = الـ hooks** — زي `security_file_open()` و `security_task_kill()`
- **الموظفين = الـ LSM callbacks** — زي `selinux_file_open()` و `apparmor_file_open()`
- **بطاقة المسافر = `struct lsm_ctx`** — الـ security context اللي بيتبادله الـ LSM مع userspace
- **دليل التفتيش = `struct lsm_static_calls_table`** — الجدول اللي بيحدد مين بيشتغل امتى

---

### Big Picture Architecture

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                        User Space                               │
 │   ┌────────────┐  ┌──────────┐  ┌─────────────────────────┐   │
 │   │  process   │  │ semanage │  │  bpftool / landlock API  │   │
 │   └─────┬──────┘  └────┬─────┘  └──────────┬──────────────┘   │
 └─────────┼──────────────┼─────────────────────┼─────────────────┘
           │  syscall     │  /proc/*/attr        │  LSM syscalls
           │              │  /sys/kernel/security │  (lsm_get_self_attr)
 ══════════╪══════════════╪═════════════════════╪══ syscall boundary ══
           ▼              ▼                      ▼
 ┌─────────────────────────────────────────────────────────────────┐
 │                     Kernel Space                                │
 │                                                                 │
 │  ┌──────────────────────────────────────────────────────────┐  │
 │  │              VFS / Networking / IPC / ...                │  │
 │  │                                                          │  │
 │  │  [open file]──►security_file_open()                     │  │
 │  │  [execve]────►security_bprm_check()                     │  │
 │  │  [connect]───►security_socket_connect()                 │  │
 │  └──────────────────────┬───────────────────────────────────┘  │
 │                         │                                       │
 │  ┌──────────────────────▼───────────────────────────────────┐  │
 │  │              LSM Framework (security/security.c)         │  │
 │  │                                                          │  │
 │  │  lsm_static_calls_table (per hook, array of callbacks)  │  │
 │  │  ┌──────────┬──────────┬──────────┬──────────────────┐  │  │
 │  │  │capability│ selinux  │ apparmor │   bpf_lsm   ...  │  │  │
 │  │  └────┬─────┴────┬─────┴────┬─────┴────┬─────────────┘  │  │
 │  └───────┼──────────┼──────────┼───────────┼────────────────┘  │
 │          │          │          │           │                    │
 │  ┌───────▼──┐ ┌─────▼────┐ ┌──▼───────┐ ┌─▼──────────────┐   │
 │  │Capability│ │ SELinux  │ │AppArmor  │ │  BPF LSM       │   │
 │  │  LSM     │ │  LSM     │ │  LSM     │ │  (eBPF prog)   │   │
 │  │(built-in)│ │(built-in)│ │(built-in)│ │                │   │
 │  └──────────┘ └──────────┘ └──────────┘ └────────────────┘   │
 │                                                                 │
 │  ┌──────────────────────────────────────────────────────────┐  │
 │  │  Security Blobs (embedded in kernel objects)             │  │
 │  │  task_struct.security  inode.i_security  file.f_security │  │
 │  └──────────────────────────────────────────────────────────┘  │
 └─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ LSM بيقوم على فكرتين أساسيتين:

#### 1. الـ Hook — نقطة القرار الأمني

في كل عملية حساسة جوه الـ kernel، في استدعاء لـ `security_XXX()` function. ده بيتعمل عن طريق X-macro pattern من `lsm_hook_defs.h`:

```c
// كل سطر في lsm_hook_defs.h بيعرّف hook زي:
LSM_HOOK(int, 0, file_open, struct file *file)
// ده بيولّد:
//   - نوع الـ return (int)
//   - القيمة الافتراضية لو مفيش LSM (0)
//   - اسم الـ hook (file_open)
//   - الـ arguments

// الـ framework بيولّد تلقائياً:
// security_file_open(file) {
//     for each registered LSM:
//         ret = lsm->file_open(file);
//         if (ret != 0) return ret;
//     return 0;
// }
```

#### 2. الـ Security Blob — بيانات LSM مضمّنة في kernel objects

كل kernel object زي `task_struct`, `inode`, `file`, `cred` عنده مكان مخصص لكل LSM يخزن فيه بياناته (labels/contexts):

```c
// مش pointer واحد — ده offset بيتحسب وقت boot
struct lsm_blob_sizes {
    unsigned int lbs_cred;       // حجم blob في struct cred
    unsigned int lbs_file;       // حجم blob في struct file
    unsigned int lbs_inode;      // حجم blob في struct inode
    unsigned int lbs_task;       // حجم blob في struct task_struct
    // ...
};
```

يعني SELinux ممكن يخزن security label جنب AppArmor label في نفس الـ inode من غير ما حد يتعارض مع التاني.

---

### الـ Struct Relationships — علاقة الـ structures ببعض

```
  DEFINE_LSM(selinux) ──────────────────────────────────────────────────────────┐
  (في .lsm_info.init section)                                                   │
                                                                                 ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ struct lsm_info                                                              │
  │   .id ──────────────────────────────────► struct lsm_id                     │
  │                                              .name = "selinux"               │
  │   .order = LSM_ORDER_MUTABLE                 .id   = LSM_ID_SELINUX (101)   │
  │   .blobs ───────────────────────────────► struct lsm_blob_sizes             │
  │                                              .lbs_inode = sizeof(inode_sec) │
  │   .init = selinux_init()                     .lbs_cred  = sizeof(cred_sec)  │
  └──────────────────────────────────────────────────────────────────────────────┘
                │
                │ selinux_init() calls: security_add_hooks(hooks, count, lsmid)
                ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ struct security_hook_list[]    (array, واحد per hook)                       │
  │                                                                              │
  │   [0]: .scalls ──────────────► static_calls_table.file_open[slot_N]         │
  │        .hook.file_open = selinux_file_open                                  │
  │        .lsmid ───────────────► lsm_id { "selinux", 101 }                   │
  │                                                                              │
  │   [1]: .scalls ──────────────► static_calls_table.task_kill[slot_N]         │
  │        .hook.task_kill = selinux_task_kill                                  │
  │        .lsmid ───────────────► lsm_id { "selinux", 101 }                   │
  └──────────────────────────────────────────────────────────────────────────────┘
                │
                ▼
  ┌──────────────────────────────────────────────────────────────────────────────┐
  │ struct lsm_static_calls_table  (global table, __ro_after_init)              │
  │                                                                              │
  │   .file_open[MAX_LSM_COUNT]:                                                 │
  │      [0]: lsm_static_call { key, trampoline, hl=apparmor_hook, active=true }│
  │      [1]: lsm_static_call { key, trampoline, hl=selinux_hook,  active=true }│
  │      [2]: lsm_static_call { key, trampoline, hl=NULL,          active=false}│
  │                                                                              │
  │   .task_kill[MAX_LSM_COUNT]: ...                                             │
  └──────────────────────────────────────────────────────────────────────────────┘
```

**ملاحظة مهمة على الـ static calls**: الـ kernel بيستخدم `static_call` بدل function pointers عادية عشان يتفادى الـ Spectre/Meltdown attacks اللي ممكن تستغل indirect branches. الـ `static_call` بيعمل patching للـ call site مباشرة في الـ code segment وقت runtime.

---

### الـ UAPI — الواجهة مع Userspace

الملف الأصلي `uapi/linux/lsm.h` بيعرّف الـ ABI بين الـ kernel والـ user space:

#### `struct lsm_ctx` — حاوية الـ Security Context

```c
struct lsm_ctx {
    __u64 id;       // مين الـ LSM؟ (LSM_ID_SELINUX = 101)
    __u64 flags;    // LSM-specific flags
    __u64 len;      // الحجم الكلي للـ struct + ctx + padding
    __u64 ctx_len;  // حجم ctx فقط
    __u8  ctx[];    // الـ security label نفسه (flexible array member)
};
```

**ليه كل الـ fields بـ `__u64`؟**
عشان الـ alignment يكون consistent بين 32-bit و 64-bit userspace على نفس الـ kernel. ده مهم خصوصاً في embedded systems اللي ممكن تشغّل 32-bit processes على 64-bit kernel (CONFIG_COMPAT).

**مثال عملي** — قراءة الـ security context بتاع process:

```c
// userspace code
#include <linux/lsm.h>

// syscall: lsm_get_self_attr(attr, ctx, size, flags)
// LSM_ATTR_CURRENT = 100 → الـ current security context

struct lsm_ctx *ctx;
__u32 size = 4096;
ctx = malloc(size);

// لو SELinux شغّال، هيرجع:
// ctx->id      = 101 (LSM_ID_SELINUX)
// ctx->ctx     = "unconfined_u:unconfined_r:unconfined_t:s0\0"
// ctx->ctx_len = strlen(ctx->ctx) + 1
// ctx->len     = sizeof(lsm_ctx) + ctx->ctx_len
```

#### الـ LSM_ATTR_XXX — أنواع الـ Attributes

```
LSM_ATTR_CURRENT  = 100  → الـ label الحالي للـ process
LSM_ATTR_EXEC     = 101  → الـ label اللي هيتطبق بعد exec()
LSM_ATTR_FSCREATE = 102  → الـ label اللي هيتطبق على الملفات الجديدة
LSM_ATTR_KEYCREATE= 103  → الـ label اللي هيتطبق على kernel keys
LSM_ATTR_PREV     = 104  → الـ label السابق (قبل الـ transition)
LSM_ATTR_SOCKCREATE=105  → الـ label اللي هيتطبق على sockets الجديدة
```

ده بيخلي user space تقدر تتحكم في الـ security transitions من غير ما تعرف التفاصيل الداخلية لكل LSM.

#### الـ LSM_FLAG_SINGLE

```c
#define LSM_FLAG_SINGLE 0x0001
```

لما userspace تعمل `lsm_get_self_attr()` وتحدد `LSM_FLAG_SINGLE`، بيجيب context LSM واحد بس (المحدد بالـ `id`). من غيره بيجيب الكل.

---

### الـ LSM IDs — تسجيل رسمي

```c
#define LSM_ID_UNDEF      0    // undefined — مش مسموح يتبعت لـ userspace
#define LSM_ID_CAPABILITY 100  // Unix capabilities (دايماً أول واحد)
#define LSM_ID_SELINUX    101  // Security-Enhanced Linux
#define LSM_ID_SMACK      102  // Simplified Mandatory Access Control Kernel
#define LSM_ID_TOMOYO     103  // path-based MAC (شائع في embedded)
#define LSM_ID_APPARMOR   104  // profile-based MAC (Ubuntu/Android default)
#define LSM_ID_YAMA       105  // ptrace scope restriction
#define LSM_ID_LOADPIN    106  // restrict kernel module loading source
#define LSM_ID_SAFESETID  107  // restrict setuid/setgid transitions
#define LSM_ID_LOCKDOWN   108  // kernel lockdown (Secure Boot)
#define LSM_ID_BPF        109  // eBPF-based custom policies
#define LSM_ID_LANDLOCK   110  // unprivileged sandboxing
#define LSM_ID_IMA        111  // Integrity Measurement Architecture
#define LSM_ID_EVM        112  // Extended Verification Module
#define LSM_ID_IPE        113  // Integrity Policy Enforcement
```

القيم من 1-99 **محجوزة** للاستخدام المستقبلي. ده ABI decision — مش ممكن تغيّر الأرقام دي من غير تكسير الـ userspace.

---

### LSM Stacking — الـ LSMs في نفس الوقت

في الكرنل القديم، كان ممكن تشغّل LSM واحد بس. الكرنل الحديث بيدعم **stacking** — تشغيل أكتر من LSM في نفس الوقت.

```
  kernel command line: lsm=capability,yama,selinux,bpf
                              │         │       │      │
                              ▼         ▼       ▼      ▼
  order of execution:    FIRST      MUTABLE MUTABLE  LAST
  (LSM_ORDER_FIRST)                              (LSM_ORDER_LAST=integrity)
```

الـ `enum lsm_order` بيحدد الترتيب:
- `LSM_ORDER_FIRST = -1`: Capabilities — **دايماً أول** عشان ده الـ baseline
- `LSM_ORDER_MUTABLE = 0`: SELinux/AppArmor/Yama — بيتحدد بالـ config
- `LSM_ORDER_LAST = 1`: IMA/EVM — **دايماً آخر** عشان بيتحقق من integrity

---

### ما بيملكه الـ Framework vs. ما بيفوّضه للـ Drivers

| الـ LSM Framework بيملك | الـ LSM Driver (plugin) بيملك |
|---|---|
| الـ hooks infrastructure والـ dispatch | الـ security policy logic |
| الـ blob allocation في kernel objects | استخدام الـ blob لتخزين labels/contexts |
| الـ ordering بين الـ LSMs | قرار "مسموح / مرفوض" |
| الـ UAPI (`lsm_ctx`, syscalls) | تعريف معنى الـ attributes |
| الـ static_calls optimization | الـ callback implementation |
| الـ `security_add_hooks()` registration | استدعاء `security_add_hooks()` وقت init |
| إدارة الـ blob sizes عبر `lsm_blob_sizes` | تحديد كمية الـ storage المطلوبة |
| منع تعديل الـ hooks بعد boot (`__ro_after_init`) | لا يقدر يعدّل framework data بعد init |

---

### مثال كامل — الـ lifecycle من boot لـ syscall

```
1. BOOT: kernel يقرأ ".lsm_info.init" section
         └─► يلاقي: capability_lsm_info, selinux_lsm_info, apparmor_lsm_info

2. INIT: لكل LSM:
         └─► يحسب blob sizes (lbs_inode += selinux_inode_security size)
         └─► يستدعي LSM->init()
         └─► LSM يستدعي security_add_hooks() → يملأ static_calls_table
         └─► static_calls_table تصبح read-only (__ro_after_init)

3. RUNTIME: process يعمل open("/etc/shadow", O_RDONLY)
            └─► VFS تستدعي security_file_open(file)
            └─► Framework يلف على static_calls_table.file_open[]:
                [0] capability_file_open(file) → 0 (ok)
                [1] selinux_file_open(file)    → -EACCES (مرفوض!)
            └─► ترجع -EACCES للـ process
            └─► open() يفشل بـ "Permission denied"

4. USERSPACE QUERY: process يسأل عن context بتاعه
                    └─► syscall: lsm_get_self_attr(LSM_ATTR_CURRENT, buf, &size, 0)
                    └─► kernel يبني struct lsm_ctx لكل LSM
                        { id=101, ctx="unconfined_t", ctx_len=13, len=48 }
                    └─► يرجعهم لـ userspace
```

---

### ملاحظة على `__counted_by`

```c
__u8 ctx[] __counted_by(ctx_len);
```

الـ `__counted_by` ده **GCC attribute** (جزء من hardening بدأ في kernel 6.1+) بيخلي الـ compiler يعرف إن الـ flexible array member `ctx` حجمه محدد بـ `ctx_len`. ده بيمكّن الـ **-fsanitize=bounds** يتحقق تلقائياً من إن أي access للـ `ctx` في حدود الـ `ctx_len` — مهم جداً لمنع buffer overflows في الـ UAPI code.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### 0. الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### LSM_ID_XXX — معرّفات الـ LSMs

| الماكرو | القيمة | الـ LSM المقصود |
|---|---|---|
| `LSM_ID_UNDEF` | 0 | غير معرّف — محجوز، لا يُستخدم من userspace |
| `LSM_ID_CAPABILITY` | 100 | POSIX Capabilities |
| `LSM_ID_SELINUX` | 101 | SELinux |
| `LSM_ID_SMACK` | 102 | Smack (Simplified Mandatory Access Control) |
| `LSM_ID_TOMOYO` | 103 | TOMOYO Linux |
| `LSM_ID_APPARMOR` | 104 | AppArmor |
| `LSM_ID_YAMA` | 105 | Yama (ptrace restrictions) |
| `LSM_ID_LOADPIN` | 106 | LoadPin (kernel module loading policy) |
| `LSM_ID_SAFESETID` | 107 | SafeSetID (UID/GID transition control) |
| `LSM_ID_LOCKDOWN` | 108 | Lockdown (integrity lockdown) |
| `LSM_ID_BPF` | 109 | BPF LSM |
| `LSM_ID_LANDLOCK` | 110 | Landlock (unprivileged sandboxing) |
| `LSM_ID_IMA` | 111 | IMA (Integrity Measurement Architecture) |
| `LSM_ID_EVM` | 112 | EVM (Extended Verification Module) |
| `LSM_ID_IPE` | 113 | IPE (Integrity Policy Enforcement) |

> القيم من 1 لـ 99 محجوزة للمستقبل. صفر = undefined دايمًا.

#### LSM_ATTR_XXX — أنواع الـ Security Attributes

| الماكرو | القيمة | المعنى |
|---|---|---|
| `LSM_ATTR_UNDEF` | 0 | غير معرّف |
| `LSM_ATTR_CURRENT` | 100 | الـ security context الحالي للـ process |
| `LSM_ATTR_EXEC` | 101 | الـ context اللي هيتطبق على الـ exec القادم |
| `LSM_ATTR_FSCREATE` | 102 | الـ context اللي بيُستخدم لما يتعمل ملف جديد |
| `LSM_ATTR_KEYCREATE` | 103 | الـ context لإنشاء kernel keys |
| `LSM_ATTR_PREV` | 104 | الـ context السابق (قبل exec) |
| `LSM_ATTR_SOCKCREATE` | 105 | الـ context اللي بيُستخدم لإنشاء sockets |

#### LSM_FLAG_XXX — الـ API Flags

| الماكرو | القيمة | المعنى |
|---|---|---|
| `LSM_FLAG_SINGLE` | `0x0001` | الـ request ده لـ LSM واحد بس محدد بالـ `id` |

---

### 1. الـ Structs المهمة

#### `struct lsm_ctx`

**الغرض:** ده الـ struct الوحيد في الملف ده، وهو الـ container الأساسي اللي بيحمل الـ security context بتاع أي LSM في الـ userspace API. بيُستخدم في syscalls زي `lsm_get_self_attr()` و`lsm_set_self_attr()`.

```c
struct lsm_ctx {
    __u64 id;          /* which LSM: LSM_ID_SELINUX, etc. */
    __u64 flags;       /* LSM-specific flags, zero if unused */
    __u64 len;         /* total size: sizeof(lsm_ctx) + padding + extra data */
    __u64 ctx_len;     /* exact byte length of ctx[] */
    __u8  ctx[] __counted_by(ctx_len); /* the actual context value (string or binary) */
};
```

| الحقل | النوع | الشرح |
|---|---|---|
| `id` | `__u64` | معرّف الـ LSM — من `LSM_ID_XXX` |
| `flags` | `__u64` | flags خاصة بالـ LSM المحدد — صفر لو مش مستخدمة |
| `len` | `__u64` | الحجم الكلي للـ struct مع الـ padding وأي data زيادة بعد `ctx` |
| `ctx_len` | `__u64` | حجم `ctx` بالـ bytes بالظبط |
| `ctx` | `__u8[]` | الـ context الفعلي — ممكن يبقى string أو binary |

**قواعد مهمة:**
- **الـ `len`** لازم = `sizeof(struct lsm_ctx)` + padding + extra data بعد `ctx`
- **الـ `ctx_len`** لازم = حجم `ctx` بالظبط
- لو `ctx` هو string → لازم يكون null-terminated و`ctx_len = strlen(ctx) + 1`
- الـ `__counted_by(ctx_len)` annotation بتساعد الـ compiler (و sanitizers) يتحقق من الحدود

**العلاقة بالـ structs التانية:** الـ struct ده هو فقط من الـ uapi layer — بيتبعت من/لـ userspace عبر syscalls. الـ kernel داخليًا عنده structs تانية زي `lsm_id` و`security_hook_list` لكنها مش في الـ uapi.

---

### 2. رسم علاقات الـ Structs

```
  Userspace (application)
        │
        │  syscall: lsm_get_self_attr() / lsm_set_self_attr() / lsm_list_modules()
        ▼
  ┌─────────────────────────────────────────────┐
  │              struct lsm_ctx                 │
  │  ┌──────────┬──────────┬────────┬─────────┐ │
  │  │  id      │  flags   │  len   │ ctx_len │ │
  │  │ (u64)    │ (u64)    │ (u64)  │  (u64)  │ │
  │  ├──────────┴──────────┴────────┴─────────┤ │
  │  │         ctx[] — flexible array          │ │
  │  │  e.g. "unconfined\0" (SELinux label)    │ │
  │  └─────────────────────────────────────────┘ │
  └─────────────────────────────────────────────┘
        │
        │  id field points logically to one of:
        ▼
  ┌────────────────────────────────────┐
  │         LSM_ID_XXX values          │
  │  100=CAPABILITY  101=SELINUX       │
  │  102=SMACK       104=APPARMOR      │
  │  110=LANDLOCK    109=BPF  ...      │
  └────────────────────────────────────┘
        │
        │  kernel-side (NOT uapi):
        ▼
  ┌─────────────────────────────────────────────┐
  │   struct lsm_id  (kernel internal)          │
  │   struct security_hook_list  (kernel)        │
  │   struct lsm_blob_sizes  (kernel)           │
  └─────────────────────────────────────────────┘
```

---

### 3. دورة حياة الـ `lsm_ctx` — Lifecycle

```
  ┌────────────────────────────────────────────────────────────────┐
  │                     Creation (Kernel Side)                     │
  │                                                                │
  │  syscall lsm_get_self_attr() يُستدعى من userspace             │
  │     │                                                          │
  │     ▼                                                          │
  │  kernel بيحدد كام LSM enabled وكام ctx هيرجع                 │
  │     │                                                          │
  │     ▼                                                          │
  │  kernel بيعمل alloc لكل lsm_ctx مع flexible array ctx         │
  │  بيملا: id ← LSM_ID_XXX                                       │
  │          flags ← LSM-specific                                  │
  │          ctx_len ← حجم الـ label الفعلي                       │
  │          len ← sizeof(lsm_ctx) + ctx_len + padding            │
  │          ctx[] ← الـ label نفسه (e.g. SELinux label)          │
  └────────────────────────────────────────────────────────────────┘
        │
        │  copy_to_user()
        ▼
  ┌────────────────────────────────────────────────────────────────┐
  │                  Usage (Userspace Side)                        │
  │                                                                │
  │  app بتقرأ lsm_ctx من buffer                                  │
  │  بتشيك id عشان تعرف انهي LSM                                  │
  │  لو id == LSM_ID_SELINUX → ctx هو SELinux label string        │
  │  لو id == LSM_ID_APPARMOR → ctx هو AppArmor profile string    │
  │  بتستخدم ctx_len عشان تقرأ الـ bytes الصح                    │
  └────────────────────────────────────────────────────────────────┘
        │
        │  (lsm_set_self_attr): userspace بتبعت lsm_ctx للـ kernel
        ▼
  ┌────────────────────────────────────────────────────────────────┐
  │               Registration / Set (Kernel Side)                 │
  │                                                                │
  │  kernel بيتحقق: id صح؟ flags معروفة؟ len == ctx_len + header؟ │
  │  بيبعت الـ ctx للـ LSM المحدد بالـ id                         │
  │  الـ LSM بيعدّل الـ security attribute للـ process             │
  └────────────────────────────────────────────────────────────────┘
        │
        ▼
  ┌────────────────────────────────────────────────────────────────┐
  │                     Teardown                                   │
  │                                                                │
  │  kernel بيعمل kfree للـ lsm_ctx buffer بعد copy_to_user       │
  │  userspace مسؤولة عن buffer بتاعتها                           │
  └────────────────────────────────────────────────────────────────┘
```

---

### 4. Call Flow Diagrams

#### `lsm_get_self_attr()` — استرجاع الـ security contexts

```
userspace calls lsm_get_self_attr(attr, ctx_buf, size, flags)
  │
  ▼
sys_lsm_get_self_attr()   [kernel/security/lsm_syscalls.c]
  │
  ├─► validates flags (only LSM_FLAG_SINGLE allowed)
  │
  ├─► if LSM_FLAG_SINGLE: checks that exactly one LSM id provided
  │
  ├─► security_lsm_get_self_attr(attr, &ctx_list)
  │     │
  │     ├─► iterates over all active LSMs
  │     │
  │     └─► calls each LSM's .lsm_get_self_attr hook
  │               │
  │               ▼
  │           LSM allocates lsm_ctx:
  │             ctx->id      = LSM_ID_XXX
  │             ctx->flags   = 0 (or LSM specific)
  │             ctx->ctx_len = label_length
  │             ctx->len     = sizeof(*ctx) + ctx_len
  │             memcpy(ctx->ctx, label, ctx_len)
  │
  ├─► calculates total size needed
  │
  ├─► if size too small → return -E2BIG with required size
  │
  └─► copy_to_user(ctx_buf, ctx_list, total_size)
        │
        ▼
      userspace receives array of lsm_ctx structs
      (one per active LSM that supports the attr)
```

#### `lsm_set_self_attr()` — تعديل الـ security context

```
userspace calls lsm_set_self_attr(attr, ctx, size, flags)
  │
  ▼
sys_lsm_set_self_attr()
  │
  ├─► copy_from_user(kctx, ctx, size)
  │
  ├─► validates:
  │     kctx->len == size?
  │     kctx->ctx_len fits within len?
  │     flags == 0?
  │
  └─► security_lsm_set_self_attr(attr, kctx)
        │
        └─► finds LSM matching kctx->id
              │
              └─► calls LSM's .lsm_set_self_attr hook
                    │
                    ▼
                  LSM updates process security context
                  (e.g. SELinux updates task's sid)
```

#### `lsm_list_modules()` — قائمة الـ LSMs الفعّالة

```
userspace calls lsm_list_modules(ids_buf, size, flags)
  │
  ▼
sys_lsm_list_modules()
  │
  ├─► flags must be 0
  │
  ├─► counts active LSMs
  │
  ├─► if size too small → return -E2BIG
  │
  └─► copy_to_user(ids_buf, lsm_ids[], count * sizeof(__u64))
        │
        ▼
      userspace gets array of LSM_ID_XXX values
      (e.g. [101, 104, 110] = SELinux + AppArmor + Landlock)
```

---

### 5. الـ Locking Strategy

الـ `lsm.h` ده هو uapi header بحت — مفيش locks فيه. لكن على مستوى الـ kernel اللي بيستخدم الـ structs دي:

| المورد | الـ Lock المستخدم | السبب |
|---|---|---|
| قائمة الـ LSMs الفعّالة | لا يوجد lock (read-only بعد boot) | الـ LSMs بتتسجل وقت الـ boot بس، بعدين immutable |
| الـ security blob بتاع الـ task | `task_lock(task)` أو RCU | لحماية الـ security attributes من concurrent access |
| الـ copy_to/from_user | لا lock — بس `mmap_read_lock` لو محتاج | الـ userspace buffer لا يحتاج kernel lock |
| الـ LSM hook calls | الـ LSM نفسه مسؤول (SELinux عنده `sidtab_lock`، AppArmor عنده `aa_lock`) | كل LSM بيحمي data بتاعته بنفسه |

**ملاحظة مهمة:** الـ `lsm_ctx` struct بيتعمله alloc وكلمة بداخل syscall context وبيتحرر قبل ما الـ syscall يرجع — مفيش shared state محتاج synchronization على مستوى الـ struct ده نفسه. الـ locking الحقيقي بيحصل داخل كل LSM على الـ security data الخاصة بيه.
## Phase 4: شرح الـ Functions

> الـ file ده (`uapi/linux/lsm.h`) هو **UAPI header** خالص — مش بيحتوي على functions، بس بيعرّف الـ `struct lsm_ctx` والـ macros اللي بتكوّن العقد بين kernel space و user space في الـ **LSM userspace API**. الـ functions الفعلية اللي بتستخدم الـ definitions دي موجودة في `security/lsm_syscalls.c` — وهنشرحها كلها هنا مع الـ UAPI constructs.

---

### ملخص: كل الـ APIs في جدول

#### الـ Syscalls (كلها في `lsm_syscalls.c`، بتستخدم types من `uapi/linux/lsm.h`)

| Syscall | Signature | الغرض |
|---|---|---|
| `lsm_get_self_attr` | `(attr, ctx, size, flags)` | جيب attributes الـ LSM للـ task الحالي |
| `lsm_set_self_attr` | `(attr, ctx, size, flags)` | اضبط attribute الـ LSM للـ task الحالي |
| `lsm_list_modules` | `(ids, size, flags)` | جيب قائمة الـ LSMs النشطة في الـ kernel |

#### الـ Helper Functions

| Function | Location | الغرض |
|---|---|---|
| `lsm_name_to_attr` | `lsm_syscalls.c` | حوّل اسم attribute نصي لـ ID رقمي |
| `lsm_get_xattr_slot` | `linux/lsm_hooks.h` | جيب slot فاضي في array الـ xattrs |

#### الـ UAPI Constructs (macros + struct)

| Construct | النوع | الغرض |
|---|---|---|
| `struct lsm_ctx` | struct | container الـ LSM context بين kernel و userspace |
| `LSM_ID_*` | macros | تعريف IDs للـ LSMs المعروفة |
| `LSM_ATTR_*` | macros | تعريف أنواع الـ attributes القابلة للقراءة/الكتابة |
| `LSM_FLAG_SINGLE` | macro | flag للـ syscall يقول "رجّع بس اللي ليه LSM معين" |

---

### Category 1: الـ UAPI Data Structure

#### `struct lsm_ctx`

```c
struct lsm_ctx {
    __u64 id;           /* LSM ID: LSM_ID_SELINUX, LSM_ID_APPARMOR, etc. */
    __u64 flags;        /* LSM-specific flags — zero إذا مش مستخدم */
    __u64 len;          /* الحجم الكلي للـ struct + padding + بيانات إضافية */
    __u64 ctx_len;      /* حجم الـ ctx field بالظبط */
    __u8 ctx[] __counted_by(ctx_len); /* الـ context الفعلي — string أو binary */
};
```

**الـ `struct lsm_ctx` هو الـ wire format** اللي بيتتبادل بين user space وكرنل في كل عمليات الـ LSM userspace API. لو الـ LSM بيرجّع أكتر من context (زي SELinux + AppArmor مع بعض)، بيتعمل array من الـ `lsm_ctx` كل واحد يمثل LSM معين.

**تفاصيل حرجة:**

- الـ `len` لازم يساوي `sizeof(struct lsm_ctx) + padding + extra_data`. ده بيخلي الـ parser يعرف يـ jump للـ next struct في الـ array بـ `(void*)ctx + ctx->len`.
- الـ `ctx_len` لازم يساوي حجم الـ `ctx` بالظبط. لو string → `strlen(ctx) + 1` (مع الـ null terminator).
- الـ `__counted_by(ctx_len)` هو annotation لـ **Clang's bounds checker** — بيمنع buffer overread على مستوى الـ static analysis.
- الـ `flags` و `ctx` يتفسروا بس من الـ LSM اللي `id` بتاعته موجود — غيره لازم يعاملهم كـ opaque.

**ASCII — layout في الـ memory:**

```
+--------+--------+--------+--------+------- ... -------+
|  id    | flags  |  len   |ctx_len |  ctx[]            |
| 8 bytes| 8 bytes| 8 bytes| 8 bytes| ctx_len bytes     |
+--------+--------+--------+--------+------- ... -------+
 <----------- len (total struct size) ------------------>
```

**عند array من الـ structs:**

```
[ lsm_ctx#0 | ctx#0 ][ lsm_ctx#1 | ctx#1 ][ lsm_ctx#2 | ctx#2 ]
      ^-- len#0 -->^      ^-- len#1 -->^
```

---

### Category 2: الـ LSM ID Macros

```c
#define LSM_ID_UNDEF        0    /* undefined — مش يتستخدم برا kernel */
#define LSM_ID_CAPABILITY  100
#define LSM_ID_SELINUX     101
#define LSM_ID_SMACK       102
#define LSM_ID_TOMOYO      103
#define LSM_ID_APPARMOR    104
#define LSM_ID_YAMA        105
#define LSM_ID_LOADPIN     106
#define LSM_ID_SAFESETID   107
#define LSM_ID_LOCKDOWN    108
#define LSM_ID_BPF         109
#define LSM_ID_LANDLOCK    110
#define LSM_ID_IMA         111
#define LSM_ID_EVM         112
#define LSM_ID_IPE         113
```

دي الـ **stable numeric IDs** اللي بتعرّف كل LSM في الـ kernel. القيم 1-99 محجوزة للمستقبل. الـ IDs دي بتيجي من kernel في `lsm_list_modules` syscall، وبيتستخدموا كـ `lsm_ctx.id` عشان userspace يعرف مين صاحب الـ context ده.

**الـ `struct lsm_id` في kernel side** (من `lsm_hooks.h`) بتربط الـ ID بالـ name:

```c
struct lsm_id {
    const char *name;  /* "selinux", "apparmor", etc. */
    u64 id;            /* LSM_ID_SELINUX, LSM_ID_APPARMOR, etc. */
};
```

---

### Category 3: الـ LSM Attribute Macros

```c
#define LSM_ATTR_UNDEF       0    /* undefined */
#define LSM_ATTR_CURRENT   100    /* الـ context الحالي للـ task */
#define LSM_ATTR_EXEC      101    /* الـ context بعد exec القادم */
#define LSM_ATTR_FSCREATE  102    /* الـ context للـ files الجديدة */
#define LSM_ATTR_KEYCREATE 103    /* الـ context للـ keys الجديدة */
#define LSM_ATTR_PREV      104    /* الـ context السابق (قبل transition) */
#define LSM_ATTR_SOCKCREATE 105   /* الـ context للـ sockets الجديدة */
```

كل `LSM_ATTR_*` بيمثل **نوع attribute** ممكن تقراه أو تكتبه عبر `lsm_get_self_attr` / `lsm_set_self_attr`. مش كل LSM بيدعم كل attribute — SELinux مثلاً بيدعم كلهم، بينما YAMA مش بيدعم حاجة منهم.

**مقارنة بالـ `/proc/self/attr/` interface القديم:**

| الـ `/proc/self/attr/` | الـ LSM_ATTR_* |
|---|---|
| `/proc/self/attr/current` | `LSM_ATTR_CURRENT` |
| `/proc/self/attr/exec` | `LSM_ATTR_EXEC` |
| `/proc/self/attr/fscreate` | `LSM_ATTR_FSCREATE` |
| `/proc/self/attr/keycreate` | `LSM_ATTR_KEYCREATE` |
| `/proc/self/attr/prev` | `LSM_ATTR_PREV` |
| `/proc/self/attr/sockcreate` | `LSM_ATTR_SOCKCREATE` |

---

### Category 4: الـ Flag Macros

```c
#define LSM_FLAG_SINGLE  0x0001
```

الـ `LSM_FLAG_SINGLE` بيتبعت في `flags` parameter لـ `lsm_get_self_attr`. معناه: "رجّعلي بس الـ context الخاص بالـ LSM اللي `id` بتاعته موجود في الـ `ctx` اللي بعتهولك". من غيره، الـ syscall بيرجّع contexts من كل الـ LSMs النشطة اللي بتدعم الـ attribute ده.

**Use case:** برنامج بيعرف هو عايز SELinux context بس — بيبعت `ctx` فاضي بـ `id = LSM_ID_SELINUX` مع `LSM_FLAG_SINGLE`.

---

### Category 5: الـ Helper Function — `lsm_name_to_attr`

```c
u64 lsm_name_to_attr(const char *name);
```

**بتعمل إيه:** بتحوّل string اسم attribute (زي `"current"` أو `"exec"`) لـ numeric ID من النوع `LSM_ATTR_*`. بتستخدمها الـ `/proc/self/attr/` interface عشان تترجم اسم الـ file للـ ID المناسب قبل ما تعدّي للـ `security_getselfattr` / `security_setselfattr`.

**Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `name` | `const char *` | اسم الـ attribute كـ string ASCII |

**Return value:** الـ `LSM_ATTR_*` المقابل، أو `LSM_ATTR_UNDEF` (0) لو مفيش match.

**Key details:**

- مفيش locking — pure lookup بلا side effects.
- المقارنة بـ `strcmp` — case-sensitive، `"Current"` مش هيشتغل.
- بتتكلمها kernel code اللي بتبري بين الـ procfs attribute names والـ numeric IDs.

**Pseudocode:**

```c
u64 lsm_name_to_attr(const char *name) {
    if (!strcmp(name, "current"))    return LSM_ATTR_CURRENT;
    if (!strcmp(name, "exec"))       return LSM_ATTR_EXEC;
    if (!strcmp(name, "fscreate"))   return LSM_ATTR_FSCREATE;
    if (!strcmp(name, "keycreate"))  return LSM_ATTR_KEYCREATE;
    if (!strcmp(name, "prev"))       return LSM_ATTR_PREV;
    if (!strcmp(name, "sockcreate")) return LSM_ATTR_SOCKCREATE;
    return LSM_ATTR_UNDEF;           /* no match */
}
```

---

### Category 6: الـ Syscalls

#### `sys_lsm_get_self_attr`

```c
SYSCALL_DEFINE4(lsm_get_self_attr,
    unsigned int, attr,          /* LSM_ATTR_* */
    struct lsm_ctx __user *, ctx, /* user buffer للـ output */
    u32 __user *, size,          /* pointer لحجم الـ buffer */
    u32, flags                   /* 0 أو LSM_FLAG_SINGLE */
)
```

**بتعمل إيه:** بترجع الـ LSM security attributes للـ task الحالي (`current`). بتستدعي `security_getselfattr` اللي بتلف على كل الـ LSMs النشطة، وكل واحد بيكتب الـ `lsm_ctx` بتاعه في الـ user buffer.

**Parameters:**

| Parameter | الوصف |
|---|---|
| `attr` | نوع الـ attribute المطلوب: `LSM_ATTR_CURRENT`, `LSM_ATTR_EXEC`, إلخ |
| `ctx` | user buffer هيتملا بـ array من `struct lsm_ctx`. ممكن يكون `NULL` لو عايز تعرف الحجم المطلوب بس |
| `size` | pointer لـ `u32` فيه حجم الـ buffer. بيتعدّل بالحجم الفعلي لو buffer صغير |
| `flags` | `0` = رجّع من كل LSMs، `LSM_FLAG_SINGLE` = رجّع من LSM واحد محدد بـ `ctx->id` |

**Return value:**
- `>= 0` = عدد الـ `lsm_ctx` elements المرجعة (ممكن تكون صفر لو مفيش contexts)
- `-E2BIG` = الـ buffer صغير، وتم تحديث `*size` بالحجم المطلوب
- `-EFAULT` = خطأ في الـ user memory access

**Key details:**

- **Caller context:** user process عبر syscall interface.
- الـ `security_getselfattr` بتعمل iteration على `lsm_idlist` وبتنادي كل LSM hooks بالترتيب.
- لو `ctx == NULL` → بترجع `-E2BIG` وبتحط الحجم المطلوب في `*size` — ده الـ pattern الصح لمعرفة الحجم المطلوب قبل الـ allocation.

**Flow:**

```
user calls lsm_get_self_attr(LSM_ATTR_CURRENT, buf, &size, 0)
    |
    v
security_getselfattr(attr, ctx, size, flags)
    |
    +---> for each active LSM:
    |        lsm->getprocattr(current, attr) → writes lsm_ctx to user buf
    |
    v
returns count of lsm_ctx written
```

---

#### `sys_lsm_set_self_attr`

```c
SYSCALL_DEFINE4(lsm_set_self_attr,
    unsigned int, attr,           /* LSM_ATTR_* */
    struct lsm_ctx __user *, ctx, /* الـ context الجديد */
    u32, size,                    /* حجم الـ ctx buffer */
    u32, flags                    /* reserved, must be 0 */
)
```

**بتعمل إيه:** بتضبط الـ LSM security attribute للـ task الحالي. بتمرر الـ user-provided `lsm_ctx` للـ `security_setselfattr` اللي بتعمل validation وبتستدعي الـ LSM hook المناسب.

**Parameters:**

| Parameter | الوصف |
|---|---|
| `attr` | الـ attribute المطلوب تعديله |
| `ctx` | pointer لـ `struct lsm_ctx` في user space فيه الـ LSM ID والـ context الجديد |
| `size` | حجم الـ `ctx` buffer كاملاً |
| `flags` | محجوز، لازم يكون صفر حالياً |

**Return value:**
- `0` = نجاح
- قيمة سالبة = error code (`-EINVAL`, `-EPERM`, `-EFAULT`, إلخ)

**Key details:**

- بس الـ LSM اللي `ctx->id` بتاعه يقدر يعالج الطلب — الـ kernel بيعمل dispatch بناءً على الـ ID.
- الـ LSM نفسه مسؤول عن التحقق من الـ permissions (مثلاً SELinux بيتحقق إن الـ transition مسموح بيه في الـ policy).
- **Caller context:** user process، ممكن يحتاج capabilities أو permissions معينة حسب الـ LSM.

---

#### `sys_lsm_list_modules`

```c
SYSCALL_DEFINE3(lsm_list_modules,
    u64 __user *, ids,   /* user buffer لاستقبال الـ LSM IDs */
    u32 __user *, size,  /* pointer لحجم الـ buffer بالـ bytes */
    u32, flags           /* reserved, must be 0 */
)
```

**بتعمل إيه:** بترجع list بـ IDs كل الـ LSMs النشطة في الـ kernel، مرتبة حسب أولويتها. بتتيح لـ userspace يعرف إيه الـ security modules الشغالة من غير ما يعتمد على أسامي hardcoded.

**Parameters:**

| Parameter | الوصف |
|---|---|
| `ids` | user buffer من نوع `u64[]` — هيتملا بـ `LSM_ID_*` values للـ LSMs النشطة |
| `size` | pointer لـ `u32`: input = حجم الـ buffer بالـ bytes، output = الحجم المطلوب |
| `flags` | لازم صفر — أي قيمة تانية بترجع `-EINVAL` |

**Return value:**
- `>= 0` = عدد الـ LSMs النشطة (حتى لو buffer كان NULL)
- `-E2BIG` = buffer صغير، `*size` اتعدّل بالحجم الصح
- `-EFAULT` = خطأ في قراءة/كتابة الـ user memory
- `-EINVAL` = `flags != 0`

**Key details:**

- `lsm_active_cnt` و `lsm_idlist` هما globals بيتبنوا وقت الـ boot في `security/security.c`.
- بيعمل `get_user(*size)` أولاً عشان يقرأ الحجم المتاح، وبعدين `put_user(total_size, size)` عشان يحدّثه — حتى لو الـ buffer صغير.
- مفيش locking لأن `lsm_active_cnt` و `lsm_idlist` read-only بعد الـ init (`__ro_after_init`).

**Pseudocode:**

```c
sys_lsm_list_modules(ids, size, flags):
    if flags != 0:
        return -EINVAL

    total_size = lsm_active_cnt * sizeof(u64)

    usize = get_user(*size)           // قرا حجم buffer المستخدم
    put_user(total_size, size)         // حدّث بالحجم المطلوب دايماً

    if usize < total_size:
        return -E2BIG

    for i in 0..lsm_active_cnt:
        put_user(lsm_idlist[i]->id, ids++)  // اكتب كل ID

    return lsm_active_cnt
```

---

### Category 7: الـ Kernel-Internal Helper — `lsm_get_xattr_slot`

```c
/* في linux/lsm_hooks.h */
static inline struct xattr *lsm_get_xattr_slot(struct xattr *xattrs,
                                                int *xattr_count)
```

**بتعمل إيه:** بترجع الـ slot الجاي المتاح في array الـ xattrs اللي بيتستخدمها الـ LSMs عشان تضيف xattrs لـ inode جديدة وقت الـ creation. كل LSM بينادي الـ function دي مرة عشان ياخد slot خاص بيه.

**Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `xattrs` | `struct xattr *` | الـ array المخصصة للـ xattrs — حجمها محدد بـ `lbs_xattr_count` |
| `xattr_count` | `int *` | عداد الـ slots المستخدمة حالياً — بيتزاد بـ 1 |

**Return value:**
- Pointer للـ `struct xattr` الجاي المتاح
- `NULL` لو `xattrs` نفسها كانت `NULL` (الـ caller مش هيبعت xattrs)

**Key details:**

- `inline` — مفيش function call overhead.
- الـ `unlikely(!xattrs)` بيوجّه الـ branch predictor إن الـ NULL case نادر.
- مفيش bounds check — الـ kernel بيفترض إن الـ array حجمها كافي بناءً على `lbs_xattr_count` اللي اتحسب وقت LSM registration.
- **Caller context:** LSM hooks زي `inode_init_security` أثناء إنشاء inodes جديدة.

**مثال استخدام في LSM:**

```c
/* في LSM hook: inode_init_security */
struct xattr *slot = lsm_get_xattr_slot(xattrs, xattr_count);
if (slot) {
    slot->name  = XATTR_NAME_SELINUX;   /* "security.selinux" */
    slot->value = context;
    slot->value_len = clen;
}
```
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem ده هو **Linux Security Modules (LSM)** — الـ framework اللي بيوفر interface موحد بين الـ kernel والـ security modules زي SELinux وAppArmor وSmack. الـ UAPI header بيعرّف `struct lsm_ctx` والـ `LSM_ID_*` constants اللي بتتكلم بيها الـ userspace مع الـ kernel عبر syscalls زي `lsm_get_self_attr()` و`lsm_set_self_attr()`.

---

### Software Level

#### 1. debugfs Entries

الـ LSM مش بيكتب كتير في `/sys/kernel/debug/` بشكل مباشر، بس فيه entries مهمة:

```bash
# شوف الـ LSMs الـ enabled على النظام
cat /sys/kernel/debug/lsm

# لو SELinux enabled:
ls /sys/kernel/debug/selinux/

# أهم entries في SELinux debugfs:
cat /sys/kernel/debug/selinux/ss/sidtab_hash_stats   # hash table stats
cat /sys/kernel/debug/selinux/ss/avtab_hash_stats    # access vector table stats
cat /sys/kernel/debug/selinux/avc/cache_stats        # AVC cache hit/miss

# لو AppArmor:
ls /sys/kernel/debug/apparmor/
cat /sys/kernel/debug/apparmor/profiles              # loaded profiles
```

**تفسير الـ AVC cache stats:**
```
lookups:   1000000   # عدد المرات اللي بحثنا فيها
hits:       980000   # لقيناها في الـ cache
misses:      20000   # محتاجين نحسبها من أول
```
نسبة الـ miss العالية → الـ cache صغير، جرب تزود `avc_cache_threshold`.

#### 2. sysfs Entries

```bash
# الـ LSMs الـ active على النظام (ordered list)
cat /sys/kernel/security/lsm

# مثال على output:
# lockdown,capability,landlock,yama,apparmor,selinux

# SELinux sysfs:
cat /sys/fs/selinux/enforce          # 0=permissive, 1=enforcing
cat /sys/fs/selinux/checkreqprot     # check requested or applied protection
cat /sys/fs/selinux/deny_unknown     # deny access to unknown object classes

# AppArmor sysfs:
cat /sys/module/apparmor/parameters/enabled
cat /sys/module/apparmor/parameters/mode
cat /sys/module/apparmor/parameters/audit

# Yama:
cat /sys/kernel/yama/ptrace_scope    # 0-3

# Lockdown:
cat /sys/kernel/security/lockdown    # none/integrity/confidentiality
```

#### 3. ftrace — Tracepoints وEvents

```bash
# شوف الـ LSM events الـ available
ls /sys/kernel/tracing/events/lsm/

# enable كل الـ LSM events
echo 1 > /sys/kernel/tracing/events/lsm/enable

# أو enable event معين (مثلاً lsm_audit):
echo 1 > /sys/kernel/tracing/events/lsm/lsm_audit/enable

# تتبع syscalls الـ LSM الجديدة:
echo 'syscalls:sys_enter_lsm_get_self_attr' >> /sys/kernel/tracing/set_event
echo 'syscalls:sys_enter_lsm_set_self_attr' >> /sys/kernel/tracing/set_event
echo 'syscalls:sys_enter_lsm_list_modules'  >> /sys/kernel/tracing/set_event

# شغّل الـ trace
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace_pipe

# function tracer على security_ functions:
echo function > /sys/kernel/tracing/current_tracer
echo 'security_*' > /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace_pipe
```

**مثال على output:**
```
     bash-1234  [002] .... security_file_open+0x0/0x40
     bash-1234  [002] .... security_inode_permission+0x0/0x60
```

#### 4. printk و Dynamic Debug

```bash
# تفعيل dynamic debug لكل الـ LSM security.c:
echo 'file security/security.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو كل الـ LSM subsystem:
echo 'module security +p' > /sys/kernel/debug/dynamic_debug/control

# SELinux verbose:
echo 'file security/selinux/* +p' > /sys/kernel/debug/dynamic_debug/control

# AppArmor verbose:
echo 'file security/apparmor/* +p' > /sys/kernel/debug/dynamic_debug/control

# رفع الـ loglevel عشان تشوف KERN_DEBUG:
echo 8 > /proc/sys/kernel/printk

# أو من kernel cmdline:
# loglevel=8 lsm=capability,selinux,apparmor

# تابع الـ audit log:
journalctl -k -f | grep -E 'avc:|apparmor='
```

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_SECURITY` | تفعيل الـ LSM framework أصلاً |
| `CONFIG_SECURITY_SELINUX` | SELinux support |
| `CONFIG_SECURITY_SELINUX_DEVELOP` | يخلي SELinux يبدأ permissive بدل enforcing |
| `CONFIG_SECURITY_SELINUX_DEBUG` | debug messages في SELinux |
| `CONFIG_SECURITY_APPARMOR_DEBUG` | AppArmor debug output |
| `CONFIG_SECURITY_APPARMOR_DEBUG_ASSERTS` | assertions في AppArmor |
| `CONFIG_SECURITY_SMACK_BRINGUP` | Smack bringup mode — يعمل log بدل deny |
| `CONFIG_AUDIT` | لازم يكون enabled عشان LSM audit يشتغل |
| `CONFIG_AUDITSYSCALL` | audit الـ syscalls بما فيهم الـ LSM syscalls |
| `CONFIG_DEBUG_LIST` | تتحقق من سلامة الـ linked lists المستخدمة في الـ hooks |
| `CONFIG_KALLSYMS` | تظهر اسم الـ function في الـ stack traces |
| `CONFIG_SECURITY_LOCKDOWN_LSM_EARLY` | Lockdown يشتغل بدري في الـ boot |
| `CONFIG_LSM` | الـ ordered list من الـ LSMs المحملة (كـ string) |
| `CONFIG_SECURITY_LANDLOCK` | Landlock LSM |

```bash
# تحقق من الـ config الحالي:
zcat /proc/config.gz | grep -E 'CONFIG_SECURITY|CONFIG_AUDIT|CONFIG_LSM'
```

#### 6. devlink وTools الخاصة بالـ Subsystem

```bash
# الـ tool الرئيسي: lsm_get_self_attr syscall (kernel 6.8+)
# مفيش tool standard بس ممكن تكتب C program صغير:

# Python wrapper بسيط:
python3 -c "
import ctypes, os
NR_lsm_list_modules = 460  # x86_64
libc = ctypes.CDLL(None)
# check if syscall exists
ret = libc.syscall(NR_lsm_list_modules, 0, ctypes.c_size_t(0), 0)
print('syscall exists' if ret != -1 or ctypes.get_errno() != 38 else 'not available')
"

# SELinux tools:
sestatus -v           # حالة SELinux التفصيلية
seinfo -t             # كل الـ types
sesearch --allow -s httpd_t  # الـ allow rules للـ httpd_t
ausearch -m avc -ts recent   # آخر AVC denials

# AppArmor tools:
aa-status             # profiles loaded وmodes
aa-complain /usr/sbin/nginx  # حط profile في complain mode
aa-enforce  /usr/sbin/nginx  # رجّع لـ enforce mode
journalctl -k | grep apparmor  # audit log

# Smack:
cat /proc/$$/attr/current          # label الـ process الحالي
ls /sys/fs/smackfs/                # كل الـ smack filesystem

# Landlock (kernel 5.13+):
# مفيش tools رسمية، بس ممكن تستخدم strace:
strace -e trace=landlock_create_ruleset,landlock_add_rule,landlock_restrict_self ./myapp

# lsm_list_modules syscall — اعرف الـ LSMs المحملة:
cat /sys/kernel/security/lsm
```

#### 7. جدول رسائل الأخطاء الشائعة

| الرسالة | المعنى | الحل |
|---------|--------|------|
| `avc: denied { read } for pid=X comm="bash" ...` | SELinux منع قراءة resource | `audit2allow` أو تعديل الـ policy |
| `apparmor="DENIED" operation="open" ...` | AppArmor منع العملية | حط الـ profile في complain mode وراجع الـ log |
| `lsm_get_self_attr: -EINVAL` | الـ attr المطلوب مش supported من الـ LSM | تحقق إن الـ `LSM_ATTR_*` صح وإن الـ LSM support it |
| `lsm_set_self_attr: -EPERM` | Process مش عنده صلاحية تغيير الـ attr | محتاج privilege أو policy تسمح بكده |
| `lsm_list_modules: -E2BIG` | الـ buffer صغير على عدد الـ LSMs | كبّر الـ buffer المرسل للـ syscall |
| `security_inode_permission: -EACCES` | LSM hook رفض الـ access | راجع الـ LSM policy للـ inode المعني |
| `smack: SMACK: mismatch in ...` | Smack label mismatch | راجع الـ labels وقواعد الـ access في `/sys/fs/smackfs/load2` |
| `ima: No key found ...` | IMA مش لاقي key للـ signature verification | تحقق من الـ keyring وإن الـ key محمل |
| `evm: hmac(sha1): EVM HMAC/signature mismatch` | EVM اكتشف tampering في الـ xattrs | الـ file ممكن يكون اتعبث فيه، أو الـ key غلط |
| `lockdown: Operation not permitted` | Lockdown LSM منع العملية | غيّر الـ lockdown level أو disable السيكيورتي Feature |
| `yama: ptrace blocked` | Yama منع الـ ptrace | غيّر `ptrace_scope` أو استخدم root |
| `lsm_ctx: invalid len field` | الـ `len` في `struct lsm_ctx` مش بيطابق الـ actual size | تأكد إن `len = sizeof(struct lsm_ctx) + ctx_len` |

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

```c
/* في security/security.c — عند فشل الـ hook call */
int security_inode_permission(struct inode *inode, int mask)
{
    int ret = call_int_hook(inode_permission, inode, mask);
    WARN_ON(ret < -MAX_ERRNO);   /* ret غير متوقع */
    return ret;
}

/* في lsm_get_self_attr — تحقق من سلامة الـ lsm_ctx */
static int validate_lsm_ctx(struct lsm_ctx *ctx, size_t total_len)
{
    if (WARN_ON(ctx->len > total_len)) {
        dump_stack();
        return -EINVAL;
    }
    if (WARN_ON(ctx->ctx_len > ctx->len - sizeof(*ctx))) {
        dump_stack();
        return -EINVAL;
    }
    return 0;
}

/* عند تسجيل الـ hooks — تحقق من سلامة الـ lsm_id */
void security_add_hooks(struct security_hook_list *hooks,
                        int count, const struct lsm_id *lsmid)
{
    WARN_ON(!lsmid->name);
    WARN_ON(lsmid->id == LSM_ID_UNDEF);
    /* ... */
}
```

**النقاط الاستراتيجية للـ WARN_ON:**
- عند رجوع `lsm_ctx->len` أصغر من `sizeof(struct lsm_ctx)`
- عند رجوع `lsm_ctx->id` بقيمة `LSM_ID_UNDEF` (0) في production
- عند أي hook يرجع قيمة مش في نطاق `-MAX_ERRNO` لـ `0`
- عند تسجيل LSM بـ `id` مكررة

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

الـ LSM هو software-only subsystem، بس في حالات بيتعامل مع الـ hardware بشكل غير مباشر:

```bash
# تأكد إن TPM موجود وشغال (مهم لـ IMA/EVM):
ls /dev/tpm*
cat /sys/class/tpm/tpm0/tpm_version_major
dmesg | grep -i tpm

# تحقق من الـ Secure Boot state (مهم لـ Lockdown LSM):
mokutil --sb-state
# أو:
cat /sys/firmware/efi/efivars/SecureBoot-*/  # غالباً binary

# تحقق من وجود hardware security features:
grep -m1 '' /proc/cpuinfo | grep -E 'smep|smap|pke'

# تأكد إن IMA policy متوافقة مع الـ hardware المتاح:
cat /sys/kernel/security/ima/policy
```

#### 2. Register Dump Techniques

الـ LSM مش بيتعامل مع hardware registers مباشرة، بس في سياق IMA/EVM وTPM:

```bash
# TPM registers (عبر kernel interface):
cat /sys/class/tpm/tpm0/pcrs       # قراءة PCR values
cat /sys/class/tpm/tpm0/caps       # capabilities

# لو محتاج تقرأ memory region معين (مثلاً EFI vars الخاصة بـ Secure Boot):
# بس انتبه: ده خطر على production
sudo devmem2 0xFED40000 b          # مثال: TPM MMIO base (تختلف حسب الـ platform)

# أو عبر /dev/mem (لو enabled في kernel):
sudo dd if=/dev/mem bs=1 skip=$((0xFED40000)) count=16 2>/dev/null | xxd

# أفضل طريقة — استخدم الـ kernel interface:
sudo tpm2_pcrread           # قراءة PCR values عبر tpm2-tools
sudo tpm2_getcap properties-fixed   # TPM properties
```

#### 3. Logic Analyzer / Oscilloscope Tips

الـ LSM نفسه software-only، بس لو بتـ debug مشاكل IMA مع TPM communication:

- **I2C/SPI TPM chips:** استخدم logic analyzer على الـ I2C/SPI bus لو الـ TPM مش بيستجاوب
- **Clock edges:** تأكد إن الـ clock stable، مشكلة الـ clock بتخلي الـ TPM يعيد إرسال أوامر
- **JTAG:** لو بتـ debug kernel + LSM على embedded platform، استخدم JTAG لـ breakpoints على `security_inode_permission`

```
TPM SPI Waveform:
CS  _____|‾‾‾‾‾‾‾|_____
CLK _____|‾|_|‾|_|‾|___
MOSI xxxx| CMD | DATA |xx
MISO xxxx| STS | RESP |xx
```

#### 4. Common Hardware Issues وKernel Log Patterns

| المشكلة | الـ Log Pattern | الحل |
|---------|----------------|------|
| TPM مش موجود/مش شغال | `tpm_tis: TPM is disabled/deactivated` | فعّل الـ TPM من الـ BIOS |
| Secure Boot مفعّل وبيمنع modules | `Lockdown: unsigned module loading is restricted` | sign الـ module أو disable Secure Boot |
| IMA فشل في verification | `integrity: Unable to open file` | تأكد إن الـ IMA keyring محمّل |
| EVM HMAC mismatch بعد disk change | `evm: Error: HMAC mismatch` | أعد حساب الـ HMAC بعد أي تغيير |
| TPM PCR extend فشل | `tpm: A TPM error (0x3) occurred` | الـ TPM busy أو locked |

```bash
# تحقق من الـ TPM errors في dmesg:
dmesg | grep -E 'tpm|ima|evm' | tail -50
```

#### 5. Device Tree Debugging (للـ Embedded Platforms)

```bash
# تحقق إن الـ TPM node موجود في DTB:
dtc -I fs /sys/firmware/devicetree/base | grep -A 20 'tpm'

# مثال على DT node صح للـ TPM:
# tpm@0 {
#     compatible = "tcg,tpm-tis-spi";
#     spi-max-frequency = <5000000>;
#     reg = <0>;
# };

# تحقق إن الـ kernel اتعرف على الـ node:
dmesg | grep -i 'tpm\|of: '

# لو الـ driver مش بيتعرف على الـ device:
ls /sys/bus/spi/devices/          # هل الـ device موجود؟
cat /sys/bus/spi/devices/spi0.0/modalias   # هل الـ compatible صح؟

# تحقق من الـ interrupts:
cat /proc/interrupts | grep tpm

# فرق مهم: لو الـ DT بيقول TPM موجود بس الـ kernel مش شايفه:
# - الـ compatible string ممكن يكون غلط
# - الـ pinmux ممكن يكون مش configured
# - الـ power domain ممكن يكون مش enabled
```

---

### Practical Commands

#### جميع الأوامر جاهزة للـ Copy

```bash
# ===== تحقق سريع من حالة الـ LSM =====
cat /sys/kernel/security/lsm
sestatus 2>/dev/null || echo "SELinux not running"
aa-status 2>/dev/null || echo "AppArmor not running"

# ===== قراءة attributes لـ process معين عبر /proc =====
cat /proc/$$/attr/current          # LSM label الـ process الحالي
cat /proc/$$/attr/exec             # label بعد exec
cat /proc/$$/attr/fscreate         # label الـ files الجديدة
cat /proc/$$/attr/keycreate        # label الـ keys الجديدة
cat /proc/$$/attr/sockcreate       # label الـ sockets الجديدة
cat /proc/$$/attr/prev             # الـ label السابق

# ===== تفعيل LSM tracing =====
echo 1 > /sys/kernel/tracing/events/syscalls/sys_enter_lsm_get_self_attr/enable
echo 1 > /sys/kernel/tracing/events/syscalls/sys_enter_lsm_set_self_attr/enable
echo 1 > /sys/kernel/tracing/events/syscalls/sys_enter_lsm_list_modules/enable
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace_pipe &
TRACE_PID=$!
# شغّل البرنامج هنا...
kill $TRACE_PID
echo 0 > /sys/kernel/tracing/tracing_on

# ===== كود C لاستدعاء lsm_get_self_attr =====
```

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/syscall.h>
#include <linux/lsm.h>   /* struct lsm_ctx, LSM_ATTR_*, LSM_FLAG_* */

/* syscall numbers on x86_64 */
#ifndef __NR_lsm_get_self_attr
#define __NR_lsm_get_self_attr 459
#endif
#ifndef __NR_lsm_list_modules
#define __NR_lsm_list_modules  461
#endif

int main(void)
{
    /* Step 1: اعرف الـ LSMs المحملة */
    __u64 ids[16];
    __u32 count = 16;
    long ret = syscall(__NR_lsm_list_modules, ids, &count, 0);
    if (ret < 0) {
        perror("lsm_list_modules");
        return 1;
    }

    printf("Loaded LSMs (%u):\n", count);
    for (__u32 i = 0; i < count; i++)
        printf("  id=%llu\n", (unsigned long long)ids[i]);

    /* Step 2: اجيب الـ current label */
    /* أول استدعاء: اعرف حجم الـ buffer المطلوب */
    size_t ctx_size = 0;
    ret = syscall(__NR_lsm_get_self_attr,
                  LSM_ATTR_CURRENT,   /* attribute */
                  NULL,               /* buffer = NULL → يرجع الحجم المطلوب */
                  &ctx_size,
                  0);                 /* flags */

    if (ret < 0 && ctx_size == 0) {
        perror("lsm_get_self_attr (size query)");
        return 1;
    }

    /* allocate buffer */
    struct lsm_ctx *ctx = malloc(ctx_size);
    if (!ctx) { perror("malloc"); return 1; }
    memset(ctx, 0, ctx_size);

    /* تاني استدعاء: اجيب الـ actual data */
    ret = syscall(__NR_lsm_get_self_attr,
                  LSM_ATTR_CURRENT,
                  ctx,
                  &ctx_size,
                  0);
    if (ret < 0) { perror("lsm_get_self_attr"); free(ctx); return 1; }

    /* اطبع كل الـ contexts (ممكن يكون فيه أكتر من LSM) */
    struct lsm_ctx *cur = ctx;
    while ((char *)cur < (char *)ctx + ctx_size) {
        printf("LSM id=%llu flags=%llu ctx_len=%llu ctx=\"%.*s\"\n",
               (unsigned long long)cur->id,
               (unsigned long long)cur->flags,
               (unsigned long long)cur->ctx_len,
               (int)cur->ctx_len, (char *)cur->ctx);
        /* الانتقال للـ entry الجاي */
        cur = (struct lsm_ctx *)((char *)cur + cur->len);
    }

    free(ctx);
    return 0;
}
```

```bash
# compile وشغّل:
gcc -o lsm_debug lsm_debug.c && ./lsm_debug

# مثال على output:
# Loaded LSMs (3):
#   id=100   ← LSM_ID_CAPABILITY
#   id=101   ← LSM_ID_SELINUX
#   id=104   ← LSM_ID_APPARMOR
# LSM id=101 flags=0 ctx_len=30 ctx="system_u:system_r:unconfined_t:s0"
# LSM id=104 flags=0 ctx_len=13 ctx="unconfined\n"

# ===== SELinux Debugging =====
# شوف آخر AVC denials:
ausearch -m avc -ts today | aureport --avc
# أو:
dmesg | grep 'avc: denied' | tail -20

# اعرف الـ type enforcement rule المطلوب:
audit2allow -a                     # من كل الـ audit log
audit2allow -i /var/log/audit/audit.log  # من ملف محدد

# تحقق من context ملف معين:
ls -Z /etc/passwd
stat -c '%C' /etc/passwd          # SELinux context

# تحقق من context process:
ps -eZ | grep nginx
id -Z                              # context الـ shell الحالي

# ===== AppArmor Debugging =====
# شوف الـ denials:
journalctl -k | grep 'apparmor="DENIED"' | tail -20
# أو:
dmesg | grep 'apparmor="DENIED"'

# حط profile في complain mode عشان تجمع الـ events:
aa-complain /path/to/profile
# بعد كده جمّع:
journalctl -k | grep apparmor > /tmp/aa_events.txt
# وحوّل لـ profile rules:
cat /tmp/aa_events.txt | aa-logprof

# ===== Smack Debugging =====
cat /proc/$$/attr/current         # label الـ process
ls -Z /path/to/file               # label الـ file (لو smack مفعّل)
cat /sys/fs/smackfs/load2         # كل الـ access rules
cat /sys/fs/smackfs/access2       # فحص access بين labelين

# ===== IMA/EVM Debugging =====
cat /sys/kernel/security/ima/policy    # الـ policy الحالية
cat /sys/kernel/security/ima/ascii_runtime_measurements  # الـ measurements
# تحقق من IMA violations:
dmesg | grep -E 'ima:|evm:' | grep -i 'error\|fail\|mismatch'

# ===== Landlock Debugging =====
# strace لـ Landlock syscalls:
strace -e trace=landlock_create_ruleset,landlock_add_rule,landlock_restrict_self \
       -v ./my_sandboxed_app 2>&1 | head -50

# ===== dynamic debug للـ LSM =====
# شوف الـ debug messages الـ available:
grep 'security/' /sys/kernel/debug/dynamic_debug/control | head -20

# فعّل debug لـ security/security.c:
echo 'file security/security.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p=print, f=function name, l=line number, m=module name, t=thread ID

# تابع الـ output:
dmesg -w | grep 'security'

# ===== فحص struct lsm_ctx في الـ kernel memory (advanced) =====
# باستخدام crash tool بعد kernel panic:
# crash /usr/lib/debug/boot/vmlinux-$(uname -r) /proc/kcore
# (crash)> struct lsm_ctx <address>

# ===== perf لقياس overhead الـ LSM hooks =====
perf stat -e 'syscalls:sys_enter_lsm_get_self_attr,syscalls:sys_enter_lsm_set_self_attr' \
     -p $(pgrep nginx) sleep 10

# قياس overhead الـ security hooks بشكل عام:
perf record -g -e cycles sleep 5
perf report --sort symbol | grep security_
```

**تفسير الـ perf output:**
```
#  Overhead  Command  Symbol
    15.23%   nginx    security_file_open          ← كتير! راجع الـ LSM policy
     8.45%   nginx    security_inode_permission
     2.11%   nginx    selinux_inode_permission
```

لو الـ overhead عالي → راجع الـ AVC cache size أو simplify الـ policy.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — فشل تطبيق userspace في قراءة SELinux context

#### العنوان
تطبيق monitoring على gateway صناعي بيفشل يقرأ security context لـ process حساسة.

#### السياق
**Board**: TI AM62x-based industrial gateway
**Product**: نظام SCADA gateway بيجمع بيانات من sensors عبر Modbus/TCP وبيرفعها للـ cloud.
**OS**: Yocto-based Linux مع SELinux مفعّل.
**التطبيق**: daemon مكتوب بـ C بيعمل audit على كل process بتتكلم مع الـ PLC، وبيستخدم الـ LSM userspace API الجديدة (kernel 6.8+).

#### المشكلة
الـ daemon بيستدعي `lsm_get_self_attr(LSM_ATTR_CURRENT, ctx_buf, &size, 0)` ودايمًا بيرجع `-EINVAL`. التطبيق مش عارف يقرأ الـ SELinux label بتاعته.

#### التحليل
بتبص في الكود اللي بيبني الـ buffer:

```c
/* المبرمج بنى الـ buffer بشكل غلط */
struct lsm_ctx *ctx = malloc(256);
__u64 size = 256;

int ret = syscall(__NR_lsm_get_self_attr,
                  LSM_ATTR_CURRENT,  /* = 100, صح */
                  ctx,
                  &size,
                  0);                /* flags = 0 */
```

**الخطأ**: الـ `struct lsm_ctx` فيه flexible array member:

```c
struct lsm_ctx {
    __u64 id;        /* 8 bytes */
    __u64 flags;     /* 8 bytes */
    __u64 len;       /* 8 bytes — إجمالي حجم الـ struct + ctx + padding */
    __u64 ctx_len;   /* 8 bytes — حجم ctx بس */
    __u8 ctx[] __counted_by(ctx_len);  /* flexible array */
};
/* sizeof(struct lsm_ctx) = 32 bytes بدون الـ ctx */
```

المبرمج بيمرر `size = 256` كـ إجمالي buffer لكن الـ kernel بيتوقع إن `size` تحتوي على الـ buffer size كاملة، وإن `ctx->len` هيتحسب من الـ kernel. المشكلة إنه ما بيحط قيمة أولية في `ctx->len` وبالتالي الـ kernel بيرفض.

الأهم: `LSM_FLAG_SINGLE = 0x0001` — لو المبرمج عايز context من LSM واحد بس (SELinux)، لازم يحدد الـ `id` ويرفع الـ flag.

#### الحل

```c
#include <linux/lsm.h>
#include <sys/syscall.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

#define CTX_BUF_SIZE 512

int get_selinux_context(char *out, size_t out_len) {
    /* allocate enough room for struct + context string */
    uint8_t raw[CTX_BUF_SIZE];
    struct lsm_ctx *ctx = (struct lsm_ctx *)raw;
    __u64 size = CTX_BUF_SIZE;

    memset(raw, 0, CTX_BUF_SIZE);

    /*
     * LSM_FLAG_SINGLE: we want only SELinux context.
     * Must set ctx->id = LSM_ID_SELINUX when using this flag.
     */
    ctx->id = LSM_ID_SELINUX;   /* = 101 */

    int ret = syscall(__NR_lsm_get_self_attr,
                      LSM_ATTR_CURRENT,   /* attribute = 100 */
                      ctx,
                      &size,
                      LSM_FLAG_SINGLE);   /* = 0x0001 */
    if (ret < 0) {
        perror("lsm_get_self_attr");
        return -1;
    }

    /*
     * ctx->ctx_len = length of ctx->ctx (including null terminator
     * for string values per the header comment).
     */
    snprintf(out, out_len, "%.*s", (int)ctx->ctx_len, ctx->ctx);
    return 0;
}
```

**الدرس المستفاد**
`LSM_FLAG_SINGLE` مش optional لما بتعايز LSM معين — من غيره الـ kernel بيحاول يملا contexts لكل LSM نشط وممكن الـ buffer ما يكفيش، أو بيرجع `-EINVAL` لو الـ `id` ما اتضبطش. الـ header واضح: `flags` و `ctx` يتعاملوا بيهم الـ LSM المحدد بـ `id` فقط.

---

### السيناريو 2: Android TV Box على RK3562 — تطبيق NDK بيحاول يقرأ Smack label ويجيب garbage

#### العنوان
**الـ** NDK app على TV box بيقرأ Smack label غلط بسبب سوء فهم `ctx_len` vs `len`.

#### السياق
**Board**: Rockchip RK3562 Android TV box
**Product**: Smart TV box بيشغّل Android 14 مع kernel 6.6، فيه DRM content protection.
**المتطلب**: تطبيق native بيتحقق إن الـ media service شغال بـ Smack label صح قبل ما يفك تشفير المحتوى.

#### المشكلة
التطبيق بيقرأ `ctx->ctx` وبيطبع characters غريبة. الـ label المتوقع هو `"media_rw"` لكن البرنامج بيطبع أحيانًا `"media_rw\x00garbage..."`.

#### التحليل
الكود الغلط:

```c
struct lsm_ctx *ctx = malloc(200);
__u64 size = 200;
syscall(__NR_lsm_get_self_attr, LSM_ATTR_CURRENT, ctx, &size, LSM_FLAG_SINGLE);

/* BUG: using len instead of ctx_len */
printf("Label: %.*s\n", (int)ctx->len, ctx->ctx);
```

من الـ header:

```c
struct lsm_ctx {
    __u64 id;
    __u64 flags;
    __u64 len;      /* حجم الـ struct كله = 32 + ctx + padding */
    __u64 ctx_len;  /* حجم ctx بس */
    __u8 ctx[] __counted_by(ctx_len);
};
```

`ctx->len` هو الـ **total struct size** مش طول الـ string. لو الـ label هو `"media_rw"` (8 chars + null = 9 bytes)، إذن:
- `ctx->ctx_len = 9`
- `ctx->len = 32 + 9 = 41` (أو أكتر لو في padding)

المبرمج استخدم `ctx->len = 41` كـ printf width، فطبع 41 byte من `ctx->ctx` اللي فيها الـ label + باقي buffer.

#### الحل

```c
/* الصح: استخدم ctx_len مش len */
if (ctx->ctx_len > 0 && ctx->ctx[ctx->ctx_len - 1] == '\0') {
    /* string context: ctx_len includes null terminator */
    printf("Smack label: %s\n", (char *)ctx->ctx);
} else {
    /* binary context: print hex */
    for (__u64 i = 0; i < ctx->ctx_len; i++)
        printf("%02x", ctx->ctx[i]);
}
```

**للتحقق من الـ Smack label عبر sysfs (بديل أسرع في debug):**

```bash
# على الجهاز مباشرة
cat /proc/self/attr/current

# أو عبر adb
adb shell cat /proc/$PID/attr/smack/current
```

**الدرس المستفاد**
`len` و `ctx_len` اسمين قريبين لكن معناهم مختلف تمامًا. `len` = total struct footprint (مهم للـ kernel لما بيحسب offset للـ struct التالي في array). `ctx_len` = بيانات الـ context الفعلية. دايمًا استخدم `ctx_len` لما بتتعامل مع القيمة.

---

### السيناريو 3: i.MX8 Automotive ECU — kernel panic عند استخدام `lsm_list_modules` ونسيان إن `LSM_ID_UNDEF = 0`

#### العنوان
Kernel panic في ECU يدعم AUTOSAR بسبب استخدام `LSM_ID_UNDEF` كـ valid module ID.

#### السياق
**Board**: NXP i.MX8QM-based Automotive ECU
**Product**: Gateway ECU في سيارة كهربائية، بيشغّل Linux مع SELinux + Yama + Lockdown.
**الكود**: security monitoring module مكتوب كـ kernel module بيلف على قائمة LSM المفعّلة.

#### المشكلة
الـ module بيلف على نتائج `lsm_list_modules()` ولما بيلاقي `id = 0` بيتعامل معاه كـ valid LSM ويحاول يستدعي attributes عليه، فبيجيب NULL pointer dereference.

#### التحليل
من الـ header:

```c
#define LSM_ID_UNDEF    0   /* zero = undefined, MUST NOT be used */
#define LSM_ID_CAPABILITY   100
#define LSM_ID_SELINUX      101
/* ... */
#define LSM_ID_LOCKDOWN     108
```

الـ header صريح: `"A value of zero/0 is considered undefined and should not be used outside the kernel."` لكن الـ module ما بيعملش validation:

```c
/* BUG: no check for LSM_ID_UNDEF */
for (int i = 0; i < count; i++) {
    __u64 id = lsm_ids[i];
    /* إذا رجع id = 0 لأي سبب، هنا هيوقع panic */
    process_lsm(id);  /* بيعمل lookup في table بيبدأ من index 100 */
}
```

#### الحل

```c
#include <linux/lsm.h>

static const char *lsm_name_by_id(__u64 id) {
    switch (id) {
    case LSM_ID_CAPABILITY: return "capability";
    case LSM_ID_SELINUX:    return "selinux";
    case LSM_ID_SMACK:      return "smack";
    case LSM_ID_TOMOYO:     return "tomoyo";
    case LSM_ID_APPARMOR:   return "apparmor";
    case LSM_ID_YAMA:       return "yama";
    case LSM_ID_LOADPIN:    return "loadpin";
    case LSM_ID_SAFESETID:  return "safesetid";
    case LSM_ID_LOCKDOWN:   return "lockdown";
    case LSM_ID_BPF:        return "bpf";
    case LSM_ID_LANDLOCK:   return "landlock";
    case LSM_ID_IMA:        return "ima";
    case LSM_ID_EVM:        return "evm";
    case LSM_ID_IPE:        return "ipe";
    case LSM_ID_UNDEF:
    default:
        return NULL; /* guard against undefined/future IDs */
    }
}

void audit_active_lsms(__u64 *ids, int count) {
    for (int i = 0; i < count; i++) {
        if (ids[i] == LSM_ID_UNDEF) {
            pr_warn("LSM audit: got LSM_ID_UNDEF at index %d, skipping\n", i);
            continue;
        }
        const char *name = lsm_name_by_id(ids[i]);
        if (!name) {
            pr_warn("LSM audit: unknown LSM id %llu, skipping\n", ids[i]);
            continue;
        }
        pr_info("LSM audit: active LSM [%llu] = %s\n", ids[i], name);
    }
}
```

**الدرس المستفاد**
Reserved range `1-99` موجود في الـ header لأسباب forward-compatibility. دايمًا تعامل مع `id = 0` كـ invalid وعمل defensive check على أي id خارج الـ known range، خصوصًا في automotive context حيث الـ kernel version ممكن يختلف عن الـ userspace headers.

---

### السيناريو 4: Allwinner H616 IoT Sensor Hub — تطبيق Python بيستخدم ctypes لـ LSM API وبيواجه alignment مشاكل

#### العنوان
**الـ** Python IoT daemon على H616 بيعمل crash عشان الـ `struct lsm_ctx` ما اتعملتش alignment صح في ctypes.

#### السياق
**Board**: Allwinner H616 (ARM Cortex-A53, 64-bit)
**Product**: IoT sensor hub بيجمع بيانات من 200+ sensor عبر I2C/SPI، بيشغّل Armbian مع AppArmor.
**الكود**: Python 3 daemon بيستخدم `ctypes` عشان يستدعي `lsm_get_self_attr` مباشرة.

#### المشكلة
الـ daemon بيعمل segfault بشكل عشوائي. أحيانًا بيشتغل، أحيانًا بيموت. المشكلة مش reproducible بثبات.

#### التحليل
الـ Python developer كتب:

```python
import ctypes
import ctypes.util

class LsmCtx(ctypes.Structure):
    _fields_ = [
        ("id",      ctypes.c_uint64),
        ("flags",   ctypes.c_uint64),
        ("len",     ctypes.c_uint64),
        ("ctx_len", ctypes.c_uint64),
        # BUG: flexible array handled as fixed 256-byte buffer
        ("ctx",     ctypes.c_uint8 * 256),
    ]
```

المشكلة: الـ `struct lsm_ctx` في الـ header بيستخدم **flexible array member**:

```c
__u8 ctx[] __counted_by(ctx_len);
```

ده معناه إن الـ struct نفسه `32 bytes` بس، والـ `ctx` بييجي بعده في نفس الـ allocation. لما Python بيعمل `LsmCtx()` بيخصص `sizeof(LsmCtx) = 32 + 256 = 288 bytes` وبيمرر pointer للـ kernel. الـ kernel بيكتب فيه بشكل صح. المشكلة الفعلية: لو الـ kernel رجع context أكبر من 256 bytes (ممكن مع AppArmor labels الطويلة) الـ kernel بيكتب خارج الـ buffer → heap corruption.

#### الحل

```python
import ctypes
import os
import struct

# NR_lsm_get_self_attr on ARM64
NR_LSM_GET_SELF_ATTR = 459

LSM_ATTR_CURRENT = 100
LSM_ID_APPARMOR  = 104
LSM_FLAG_SINGLE  = 0x0001

LSM_CTX_HEADER_SIZE = 32  # 4 x uint64

def get_apparmor_label():
    # Step 1: probe required size by passing size=0
    size = ctypes.c_uint64(0)
    buf  = ctypes.create_string_buffer(LSM_CTX_HEADER_SIZE)

    # pre-fill id for LSM_FLAG_SINGLE
    struct.pack_into("Q", buf, 0, LSM_ID_APPARMOR)  # id field offset=0

    ret = ctypes.CDLL(None).syscall(
        NR_LSM_GET_SELF_ATTR,
        LSM_ATTR_CURRENT,
        buf,
        ctypes.byref(size),
        LSM_FLAG_SINGLE
    )
    # expect -ERANGE with size filled in
    needed = size.value

    # Step 2: allocate exact size and retry
    buf = ctypes.create_string_buffer(needed)
    struct.pack_into("Q", buf, 0, LSM_ID_APPARMOR)
    size = ctypes.c_uint64(needed)

    ret = ctypes.CDLL(None).syscall(
        NR_LSM_GET_SELF_ATTR,
        LSM_ATTR_CURRENT,
        buf,
        ctypes.byref(size),
        LSM_FLAG_SINGLE
    )
    if ret != 0:
        raise OSError(ctypes.get_errno(), "lsm_get_self_attr failed")

    # parse header
    _id, _flags, _len, ctx_len = struct.unpack_from("QQQQ", buf, 0)
    ctx_bytes = buf[LSM_CTX_HEADER_SIZE: LSM_CTX_HEADER_SIZE + ctx_len]
    return ctx_bytes.rstrip(b'\x00').decode()

print(get_apparmor_label())
```

**debug من command line:**

```bash
# تحقق من AppArmor profile للـ process
cat /proc/self/attr/apparmor/current

# تحقق من الـ LSMs المفعّلين
cat /sys/kernel/security/lsm
```

**الدرس المستفاد**
الـ flexible array member في `struct lsm_ctx` مش مجرد syntax — ده contract: الـ struct header دايمًا `32 bytes`، وبعديه الـ data. في أي لغة غير C، لازم تتعامل مع الـ buffer كـ raw bytes وتعمل two-pass: pass أول عشان تعرف الحجم المطلوب، pass تاني بالحجم الصح.

---

### السيناريو 5: STM32MP1 Custom Board Bring-Up — مهندس بيحاول يفهم ليه `lsm_get_self_attr` بيرجع `-ENOSYS`

#### العنوان
Board bring-up على STM32MP1: الـ LSM userspace API مش موجودة لأن الـ kernel قديم.

#### السياق
**Board**: STMicroelectronics STM32MP157 custom industrial board
**Product**: PLC controller بيشغّل OpenSTLinux مع kernel 5.15 (LTS).
**المهندس**: بيحاول يضيف security monitoring feature باستخدام الـ LSM API الجديدة.

#### المشكلة
كل calls لـ `lsm_get_self_attr` و `lsm_set_self_attr` و `lsm_list_modules` بترجع `-ENOSYS` (Function not implemented).

#### التحليل
**الـ** LSM userspace syscalls (`lsm_get_self_attr`, `lsm_set_self_attr`, `lsm_list_modules`) اتضافت في kernel **6.8** (2024).

الـ header file `include/uapi/linux/lsm.h` نفسه موجود من kernel 6.2 تقريبًا، لكن الـ syscalls نفسها ما كانتش موجودة في 5.15. يعني:

```c
/* هيتكمبايل بدون مشاكل */
#include <linux/lsm.h>

/* لكن هيرجع -ENOSYS على kernel 5.15 */
ret = syscall(__NR_lsm_get_self_attr, ...);
```

**كيف تتحقق:**

```bash
# على STM32MP1 board
uname -r
# OUTPUT: 5.15.118-st

# تحقق من syscall numbers
grep -r "lsm_get_self_attr" /usr/include/asm/unistd*.h
# ممكن ما يلاقيش حاجة على kernel headers قديمة

# تحقق من الـ LSMs المفعّلين بالطريقة القديمة
cat /sys/kernel/security/lsm
# OUTPUT: capability,yama,selinux

# الطريقة القديمة لقراءة SELinux context (تشتغل على 5.15)
cat /proc/self/attr/current
```

#### الحل — Compatibility Layer

```c
#include <sys/syscall.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>

/*
 * Fallback: read security context via /proc/self/attr/
 * Works on all kernel versions, no lsm.h syscalls needed.
 */
static int get_current_context_legacy(char *buf, size_t len) {
    FILE *f = fopen("/proc/self/attr/current", "r");
    if (!f) return -1;
    size_t n = fread(buf, 1, len - 1, f);
    fclose(f);
    buf[n] = '\0';
    /* strip trailing newline if present */
    if (n > 0 && buf[n-1] == '\n') buf[n-1] = '\0';
    return 0;
}

/*
 * Try new LSM API first, fall back to legacy /proc interface.
 * Needed for boards running kernel < 6.8.
 */
int get_security_context(char *buf, size_t len) {
#ifdef __NR_lsm_get_self_attr
    uint8_t raw[512] = {0};
    struct lsm_ctx *ctx = (struct lsm_ctx *)raw;
    __u64 size = sizeof(raw);

    int ret = syscall(__NR_lsm_get_self_attr,
                      LSM_ATTR_CURRENT, ctx, &size, 0);
    if (ret == 0) {
        snprintf(buf, len, "%.*s", (int)ctx->ctx_len, ctx->ctx);
        return 0;
    }
    if (errno != ENOSYS) {
        perror("lsm_get_self_attr");
        return -1;
    }
    /* ENOSYS = kernel too old, use legacy path */
#endif
    return get_current_context_legacy(buf, len);
}
```

**لو عايز تطور الـ kernel على STM32MP1:**

```bash
# في kernel config
make menuconfig
# Security options → Enable different security models
# CONFIG_SECURITY_SELINUX=y
# CONFIG_LSM="capability,yama,selinux"

# الـ new LSM syscalls مش موجودة في 5.15 حتى لو فعّلت كل ده
# لازم ترفع الـ kernel لـ 6.8+
```

**الدرس المستفاد**
وجود `include/uapi/linux/lsm.h` في الـ headers لا يضمن وجود الـ syscalls في الـ running kernel. دايمًا اتحقق من `uname -r` وعمل runtime detection بـ `ENOSYS` check. في embedded boards زي STM32MP1 اللي بتستخدم LTS kernels قديمة، لازم يكون عندك fallback عبر `/proc/self/attr/` اللي شغّالة منذ Linux 2.6.
## Phase 7: مصادر ومراجع

### مصادر رسمية في الـ Kernel

| المصدر | الوصف |
|--------|-------|
| `include/uapi/linux/lsm.h` | الـ UAPI header الأساسي — يعرّف `lsm_ctx` وكل الـ `LSM_ID_*` و`LSM_ATTR_*` |
| `Documentation/security/lsm.rst` | وثيقة الـ LSM framework من داخل الـ kernel source tree |
| `Documentation/userspace-api/lsm.rst` | وثيقة الـ LSM userspace API الرسمية — تشرح الـ syscalls الجديدة |
| `Documentation/admin-guide/LSM/` | دليل الـ admin لتفعيل وضبط الـ LSMs المختلفة |
| `security/security.c` | التنفيذ الرئيسي لإطار الـ LSM |
| `include/linux/lsm_hooks.h` | تعريف كل الـ hooks المتاحة للـ LSMs |

**الـ kernel documentation** الرسمية أونلاين:
- [Linux Security Modules — Kernel Docs](https://docs.kernel.org/security/lsm.html)
- [LSM Userspace API](https://docs.kernel.org/userspace-api/lsm.html)
- [LSM Admin Guide](https://docs.kernel.org/admin-guide/LSM/index.html)
- [LSM Development Guide](https://www.kernel.org/doc/html/latest/security/lsm-development.html)

---

### مقالات LWN.net

دي من أهم المصادر لمتابعة تطور الـ LSM من 2003 لحد دلوقتي:

| المقال | الأهمية |
|--------|---------|
| [Adding system calls for Linux security modules](https://lwn.net/Articles/919059/) | يشرح إضافة الـ syscalls الجديدة `lsm_get_self_attr` و`lsm_set_self_attr` و`lsm_list_modules` |
| [LSM: lsm_self_attr syscall for LSM self attributes](https://lwn.net/Articles/908803/) | مناقشة مبكرة لمقترح الـ syscall وتصميم `lsm_ctx` |
| [LSM: Three basic syscalls](https://lwn.net/Articles/928790/) | مناقشة الـ patchset النهائي اللي أضاف الـ 3 syscalls |
| [LSM stacking and the future](https://lwn.net/Articles/804906/) | خريطة طريق الـ LSM stacking ومشاكله التاريخية |
| [LSM: General module stacking](https://lwn.net/Articles/955464/) | آخر تحديث على مسار الـ stacking الشامل |
| [A change in direction for security-module stacking?](https://lwn.net/Articles/970070/) | نقاش 2024 حول مستقبل الـ LSM stacking |
| [Lockdown as a security module](https://lwn.net/Articles/791863/) | كيف أُضيف `LSM_ID_LOCKDOWN` (108) كـ LSM مستقل |
| [The future of the Linux Security Module API](https://lwn.net/Articles/180194/) | مقال تاريخي مهم عن تطور الـ LSM API |
| [Complete coverage in Linux security modules](https://lwn.net/Articles/154277/) | تغطية شاملة مبكرة للـ LSM framework |
| [Supporting multiple LSMs](https://lwn.net/Articles/426921/) | محاولات دعم أكتر من LSM في نفس الوقت |
| [LSM: Module stacking for AppArmor](https://lwn.net/Articles/909688/) | الـ AppArmor (`LSM_ID_APPARMOR` = 104) ودعمه للـ stacking |
| [LSM: Security module stacking](https://lwn.net/Articles/748868/) | مرحلة متوسطة مهمة في تطور الـ stacking |
| [LSM: Module stacking for all](https://lwn.net/Articles/786307/) | مقترح الـ stacking الشامل |

---

### نقاشات Mailing List المهمة

**الـ patchsets اللي أدت لإنشاء `lsm.h` UAPI:**

- [PATCH v36: LSM provide lsm name and id slot mappings — Casey Schaufler](https://lore.kernel.org/linux-security-module/20220609230146.319210-5-casey@schaufler-ca.com/)
  ده من سلسلة patches بدأها Casey Schaufler وأدت في النهاية لإنشاء `lsm.h` وتعريف `LSM_ID_*`

- [v1,4/8: LSM Maintain a table of LSM attribute data — Patchwork](https://patchwork.kernel.org/project/linux-security-module/patch/20221025184519.13231-5-casey@schaufler-ca.com/)
  التطبيق المبكر لجدول الـ LSM attributes

- [v30,14/28: LSM Specify which LSM to display — Patchwork](https://patchwork.kernel.org/project/linux-audit/patch/20211124014332.36128-15-casey@schaufler-ca.com/)
  جزء من مسيرة الـ patches الطويلة نحو الـ multi-LSM API

---

### Kernel Commits الأساسية

الـ commits دي أدخلت الـ `lsm.h` UAPI والـ syscalls المرتبطة بيه — ابحث عنها في `git log --all --oneline -- include/uapi/linux/lsm.h`:

```bash
# ابحث عن تاريخ الـ file في الـ kernel tree
git log --follow --oneline include/uapi/linux/lsm.h

# الـ commits المرتبطة بـ lsm_ctx و LSM_ID definitions
git log --oneline --grep="LSM_ID" --grep="lsm_ctx" --all-match
```

الـ commits المهمة (ابحث عنها في [kernel.org cgit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git)):
- **إضافة `include/uapi/linux/lsm.h`** — Casey Schaufler, Intel Corporation, 2022
- **إضافة `lsm_get_self_attr` syscall** — merged in kernel 6.8
- **إضافة `lsm_set_self_attr` syscall** — merged in kernel 6.8
- **إضافة `lsm_list_modules` syscall** — merged in kernel 6.8
- **إضافة `LSM_ID_IPE` (113)** — أحدث ID أُضيف للـ header

---

### KernelNewbies

- [KernelGlossary — تعريف LSM](https://kernelnewbies.org/KernelGlossary)
- [Linux_2_6_18 — أول kernel version أضاف تغييرات مهمة للـ LSM](https://kernelnewbies.org/Linux_2_6_18)
- [LinuxChanges — متابعة كل تغييرات الـ LSM عبر الإصدارات](https://kernelnewbies.org/LinuxChanges)
- [KernelHacking — نقطة بداية لتطوير modules أمنية](https://kernelnewbies.org/KernelHacking)

---

### eLinux.org

- [Security Working Group](https://elinux.org/Security_Working_Group) — مجموعة العمل المتعلقة بأمن الـ embedded Linux
- [Security Hardware Resources](https://elinux.org/Security_Hardware_Resources) — موارد الأمن على مستوى الـ hardware (TPM, Secure Boot, إلخ)
- [Category:Security](https://elinux.org/Category:Security) — فهرس كل صفحات الأمن في eLinux

---

### كتب موصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل المتعلق:** Appendix B وأي ذكر للـ kernel security
- **تحميل مجاني:** [lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)
- **ملاحظة:** الـ LSM API اتغير كتير من وقت LDD3، استخدمه كمقدمة للـ kernel internals بس

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17:** Memory Management (relevant to `__counted_by` and `__u64` types)
- **الفصل 14:** The Block I/O Layer (لفهم أسلوب التصميم العام)
- الكتاب ده مش بيتكلم عن الـ LSM بشكل مباشر، لكنه بيدي خلفية ممتازة لفهم الـ kernel architecture

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 15:** Kernel Debugging Techniques
- **الفصل 16:** Porting Linux وdiscussion عن الـ security في الـ embedded systems
- **أهميته للـ LSM:** يشرح كيف تُفعّل LSMs زي SELinux أو AppArmor في بيئات الـ embedded

#### The Linux Programming Interface — Michael Kerrisk
- **الجزء العاشر:** Security — شرح ممتاز للـ capabilities و security contexts من الـ user space
- **الأهمية لـ `lsm.h`:** يشرح كيف يتعامل الـ user space مع الـ security attributes

---

### مصادر إضافية

#### Wikipedia
- [Linux Security Modules — Wikipedia](https://en.wikipedia.org/wiki/Linux_Security_Modules)
  تاريخ الـ LSM من 2001 ومقارنة بين الـ LSMs المختلفة

#### GitHub Resources
- [KubeArmor: Introduction to Linux Security Modules](https://github.com/kubearmor/KubeArmor/wiki/Introduction-to-Linux-Security-Modules-(LSMs))
  شرح عملي للـ LSMs مع أمثلة من الـ cloud-native security

#### الـ LSM-specific Documentation
كل LSM من الـ LSMs المعرّفة في `lsm.h` عنده وثيقته الخاصة في `Documentation/admin-guide/LSM/`:

| LSM | المعرّف | الوثيقة |
|-----|---------|---------|
| SELinux | `LSM_ID_SELINUX` = 101 | `Documentation/admin-guide/LSM/SELinux.rst` |
| Smack | `LSM_ID_SMACK` = 102 | `Documentation/admin-guide/LSM/Smack.rst` |
| AppArmor | `LSM_ID_APPARMOR` = 104 | `Documentation/admin-guide/LSM/apparmor.rst` |
| Yama | `LSM_ID_YAMA` = 105 | `Documentation/admin-guide/LSM/Yama.rst` |
| Landlock | `LSM_ID_LANDLOCK` = 110 | `Documentation/userspace-api/landlock.rst` |

---

### Search Terms للبحث عن معلومات أكتر

```
# بحث عام
"LSM_ID" "lsm_ctx" site:lore.kernel.org
"lsm_get_self_attr" OR "lsm_set_self_attr" OR "lsm_list_modules"
"linux/lsm.h" uapi syscall kernel

# بحث في الـ mailing lists
site:lore.kernel.org "include/uapi/linux/lsm.h"
site:lore.kernel.org "LSM stacking" "Casey Schaufler"
site:patchwork.kernel.org "lsm_ctx" OR "LSM_ATTR"

# بحث في الـ LWN
site:lwn.net "LSM stacking"
site:lwn.net "lsm_ctx"
site:lwn.net "LSM userspace API"

# بحث في الـ kernel source
git log --all --oneline --follow -- include/uapi/linux/lsm.h
git log --oneline --grep="LSM_ID_" -- include/uapi/linux/lsm.h
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `uapi/linux/lsm.h` بيعرّف الـ `struct lsm_ctx` والـ `LSM_ID_*` constants اللي بتتعامل معاها الـ syscalls الجديدة زي `lsm_get_self_attr(2)`. أحسن حاجة نعمل kprobe عليها هي الـ `security_getselfattr` — دي الـ kernel function اللي بتتنفذ لما أي process يطلب الـ LSM attributes بتاعته من الـ userspace. هنشوف مين اللي بيعمل الـ call، وإيه الـ LSM attribute اللي بيطلبه (`LSM_ATTR_CURRENT`, إلخ).

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * lsm_ctx_probe.c
 *
 * kprobe on security_getselfattr():
 *   - logs the calling process name & PID
 *   - logs which LSM attribute is being queried (attr argument)
 *   - logs the LSM_FLAG_SINGLE flag if set
 *
 * security_getselfattr() is the kernel side of lsm_get_self_attr(2).
 * It receives an attr selector (LSM_ATTR_XXX), a user pointer to an
 * array of lsm_ctx structs, a size pointer, and flags.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/sched.h>        /* current, task_comm_len           */
#include <linux/uaccess.h>      /* access_ok (not used directly but pulled by security.h) */
#include <uapi/linux/lsm.h>     /* LSM_ATTR_XXX, LSM_FLAG_SINGLE    */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("LSM Observer <observer@kernel-labs.io>");
MODULE_DESCRIPTION("kprobe on security_getselfattr to log LSM context queries");

/* ------------------------------------------------------------------ */
/* Helper: map LSM_ATTR_XXX value to a readable string                 */
/* ------------------------------------------------------------------ */
static const char *attr_name(unsigned int attr)
{
    switch (attr) {
    case LSM_ATTR_CURRENT:    return "CURRENT";
    case LSM_ATTR_EXEC:       return "EXEC";
    case LSM_ATTR_FSCREATE:   return "FSCREATE";
    case LSM_ATTR_KEYCREATE:  return "KEYCREATE";
    case LSM_ATTR_PREV:       return "PREV";
    case LSM_ATTR_SOCKCREATE: return "SOCKCREATE";
    default:                  return "UNKNOWN";
    }
}

/* ------------------------------------------------------------------ */
/* kprobe pre-handler — runs just before security_getselfattr()        */
/*                                                                     */
/* Signature of security_getselfattr (kernel/security/security.c):    */
/*   int security_getselfattr(unsigned int attr,                       */
/*                            struct lsm_ctx __user *uctx,            */
/*                            u32 __user *size,                        */
/*                            u32 flags);                              */
/*                                                                     */
/* On x86-64 the calling convention maps args to registers:           */
/*   attr  → di (regs->di)                                            */
/*   uctx  → si (regs->si)  — userspace ptr, we don't dereference it  */
/*   size  → dx (regs->dx)                                            */
/*   flags → cx (regs->cx)                                            */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    unsigned int attr  = (unsigned int)regs->di;   /* which LSM attribute */
    unsigned int flags = (unsigned int)regs->cx;   /* LSM_FLAG_SINGLE etc */

    pr_info("lsm_ctx_probe: pid=%d comm=%-16s attr=%s(%u) flags=%s\n",
            current->pid,
            current->comm,
            attr_name(attr), attr,
            (flags & LSM_FLAG_SINGLE) ? "SINGLE" : "none");

    return 0; /* 0 = continue normal execution */
}

/* ------------------------------------------------------------------ */
/* kprobe definition                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "security_getselfattr", /* function we want to probe */
    .pre_handler = handler_pre,            /* our callback              */
};

/* ------------------------------------------------------------------ */
/* module_init: register the kprobe                                    */
/* ------------------------------------------------------------------ */
static int __init lsm_ctx_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("lsm_ctx_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("lsm_ctx_probe: planted kprobe on %s (addr=%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit: unregister the kprobe before the module is removed     */
/* ------------------------------------------------------------------ */
static void __exit lsm_ctx_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("lsm_ctx_probe: removed kprobe from %s\n", kp.symbol_name);
}

module_init(lsm_ctx_probe_init);
module_exit(lsm_ctx_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module: `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل الـ API بتاع الـ kprobe framework |
| `linux/sched.h` | عشان نوصل لـ `current->pid` و`current->comm` — بيانات الـ process الحالي |
| `uapi/linux/lsm.h` | بيجيب `LSM_ATTR_XXX` و`LSM_FLAG_SINGLE` — نفس الـ constants اللي بيستخدمها الـ userspace |

**الـ** `uapi/linux/lsm.h` بنضمّه مباشرة عشان نستخدم نفس قيم `LSM_ATTR_*` اللي بيبعتها الـ userspace في الـ syscall، من غير ما نعرّفهم تاني.

---

#### الـ `attr_name()` helper

دي مجرد `switch` بتحوّل الـ integer value بتاع الـ `attr` لـ string مقروء. بتساعدنا في الـ log إننا نشوف "CURRENT" بدل "100".

---

#### الـ `handler_pre` — قلب الـ kprobe

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

**الـ** `pt_regs` هو snapshot من الـ CPU registers لحظة وقوف الـ kprobe. بنقرأ منه الـ arguments بتاعة `security_getselfattr` مباشرة عن طريق الـ x86-64 calling convention (`rdi`, `rsi`, `rdx`, `rcx`).

- **`regs->di`**: أول argument — الـ `attr` (مثلاً `LSM_ATTR_CURRENT = 100`).
- **`regs->cx`**: رابع argument — الـ `flags`؛ بنشيك على `LSM_FLAG_SINGLE` عشان نعرف إذا كان الـ userspace عايز context من LSM واحد بس.
- **`current->comm`**: اسم الـ process اللي طلب الـ syscall — معلومة مفيدة جداً عشان نعرف مين اللي بيسأل عن الـ LSM context.

الـ return value `0` معناه "استمر في تنفيذ الـ function الأصلية بشكل طبيعي" — الـ kprobe مش بيوقف التنفيذ.

---

#### الـ `struct kprobe kp`

```c
static struct kprobe kp = {
    .symbol_name = "security_getselfattr",
    .pre_handler = handler_pre,
};
```

**الـ** `symbol_name` بيخلي الـ kprobe framework يحلّ الـ address أوتوماتيك من الـ kernel symbol table في وقت الـ registration، من غير ما نحتاج نكتب عنوان hardcoded. الـ `pre_handler` هو الـ callback اللي بيتنفذ قبل الـ function الأصلية.

---

#### الـ `module_init` و`module_exit`

- **`register_kprobe(&kp)`**: بيحط breakpoint على الـ function وبيربط الـ handler. لو فشل (مثلاً الـ function مش موجودة أو مش exported كـ symbol) بيرجع error code سالب.
- **`unregister_kprobe(&kp)`**: لازم نعمله في الـ exit عشان لو المودول اتشال من الذاكرة والـ kprobe لسه registered، أي حد يعمل call للـ function بعد كده هيتنفذ الـ handler من memory مش موجودة → **kernel panic**. إزالة الـ kprobe أول ما يتشال المودول هي الضمانة الوحيدة إن كل حاجة تترجع نظيف.

---

### Makefile

```makefile
obj-m += lsm_ctx_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### تجربة المودول

```bash
# بناء وتحميل
make
sudo insmod lsm_ctx_probe.ko

# أي process بيستخدم lsm_get_self_attr(2) هيطلع في الـ log
# مثال: id أو ps على نظام فيه SELinux/AppArmor
id

# مشاهدة الـ log
sudo dmesg | grep lsm_ctx_probe

# مثال للناتج:
# lsm_ctx_probe: planted kprobe on security_getselfattr (addr=ffffffff81a3c210)
# lsm_ctx_probe: pid=1842 comm=id               attr=CURRENT(100) flags=none

# إزالة المودول
sudo rmmod lsm_ctx_probe
```

---

### ملاحظة على الـ CONFIG

لازم الـ kernel يتبنى بـ `CONFIG_KPROBES=y` وبـ `CONFIG_SECURITY=y`. للتحقق:

```bash
grep -E "CONFIG_KPROBES|CONFIG_SECURITY" /boot/config-$(uname -r)
```
