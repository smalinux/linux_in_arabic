## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: Linux Security Module (LSM) Framework

الـ file ده جزء من **Linux Security Module (LSM) framework** — ده النظام اللي بيخلي الـ kernel يدعم أكتر من نظام أمان في نفس الوقت (زي SELinux, AppArmor, Smack) من غير ما يتعدل كود الـ kernel الأساسي.

---

### قصة المشكلة — ليه الـ LSM اتعمل أصلاً؟

**تخيل معايا السيناريو ده:**

في الـ Linux kernel القديم، لو عايز تضيف نظام أمان زي SELinux (اللي بيعمل mandatory access control)، كنت لازم تعدل في كود الـ kernel نفسه في كل مكان بيحصل فيه قرار أمان — لما حد بيفتح ملف، لما process بتعمل fork، لما socket بيتعمل، إلخ.

الناتج؟ كود kernel متشابك ومليان `#ifdef CONFIG_SELINUX` في كل حتة. وكل نظام أمان جديد (AppArmor, TOMOYO, Smack...) كان بيحتاج تعديل مختلف في نفس الأماكن.

**الحل الأنيق:**

بدل كده، اتعمل **نظام hook** — الـ kernel في كل نقطة قرار أمني بيقول "أنا هعمل كذا، في حد عايز يعترض؟" لو في LSM مسجل نفسه، بيتصل بيه ويسمع رأيه. زي بالظبط **interceptor pattern**.

---

### الـ `lsm_hook_defs.h` — هو إيه بالظبط؟

الـ file ده مش header تقليدي بـ `#ifndef` guards — ده **X-macro table**. يعني قائمة من الـ `LSM_HOOK(...)` macros، كل سطر فيها بيعرّف hook واحد.

الفكرة الذكية: نفس الـ file بيتضمن (`#include`) أكتر من مرة في أماكن مختلفة، وفي كل مرة الـ macro `LSM_HOOK` بيتعرّف بشكل مختلف عشان ينتج data structure أو function declarations مختلفة.

**مثال عملي — نفس الـ file بيتستخدم بـ 3 طرق مختلفة:**

```c
/* الاستخدام الأول: بناء union بكل function pointers */
union security_list_options {
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) RET (*NAME)(__VA_ARGS__);
    #include "lsm_hook_defs.h"
    #undef LSM_HOOK
};

/* الاستخدام الثاني: بناء جدول الـ static calls */
struct lsm_static_calls_table {
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) \
        struct lsm_static_call NAME[MAX_LSM_COUNT];
    #include <linux/lsm_hook_defs.h>
    #undef LSM_HOOK
};

/* الاستخدام الثالث: توليد function stubs في security.c */
#define LSM_HOOK(RET, DEFAULT, NAME, ...) \
    RET security_##NAME(__VA_ARGS__) { ... }
#include "lsm_hook_defs.h"
#undef LSM_HOOK
```

**الـ file لوحده بيولد الـ kernel ABI كله للـ security hooks من مصدر واحد** — Single Source of Truth.

---

### إيه اللي جوا الـ File؟

الـ file فيه **~250 hook** موزعة على أقسام:

| القسم | عدد الـ Hooks (تقريبي) | أمثلة |
|---|---|---|
| **Binder (Android IPC)** | 4 | `binder_transaction`, `binder_transfer_file` |
| **Process/Credentials** | 15+ | `bprm_check_security`, `cred_prepare`, `capable` |
| **Filesystem/Superblock** | 20+ | `sb_mount`, `sb_remount`, `sb_kern_mount` |
| **Inode Operations** | 40+ | `inode_permission`, `inode_setxattr`, `inode_create` |
| **File Operations** | 15+ | `file_open`, `file_permission`, `mmap_file` |
| **Task/Scheduling** | 15+ | `task_kill`, `task_setnice`, `task_prctl` |
| **IPC (msg/shm/sem)** | 15+ | `msg_queue_msgsnd`, `shm_shmat` |
| **Network (socket/sk)** | 30+ | `socket_connect`, `socket_bind`, `sk_alloc_security` |
| **XFRM/IPsec** | 10+ | `xfrm_policy_alloc_security`, `xfrm_state_alloc` |
| **Keys** | 4 | `key_alloc`, `key_permission` |
| **Audit** | 4 | `audit_rule_init`, `audit_rule_match` |
| **BPF** | 10+ | `bpf_prog_load`, `bpf_map_create` |
| **Perf Events** | 4 | `perf_event_open`, `perf_event_read` |
| **io_uring** | 4 | `uring_override_creds`, `uring_cmd` |
| **Block Device** | 3 | `bdev_alloc_security`, `bdev_setintegrity` |

---

### شكل الـ Hook — كيف بيشتغل؟

كل hook بيتعرّف بالشكل ده:

```c
LSM_HOOK(return_type, default_value, hook_name, args...)
```

- **`return_type`**: نوع القيمة المرجعة — `int` للـ hooks اللي ممكن ترفض، `void` للـ hooks الإخبارية
- **`default_value`**: القيمة اللو مفيش LSM اتسجل — عادةً `0` (success) أو `LSM_RET_VOID`
- **`hook_name`**: اسم الـ hook
- **`args...`**: البارامترات اللي بتتبعت للـ LSM

**مثال على hook بيرفض:**
```c
LSM_HOOK(int, 0, file_open, struct file *file)
/*
 * لما process بتفتح ملف:
 * - الـ kernel بيستدعي security_file_open()
 * - لو SELinux مسجل وقال -EACCES، الـ open بيفشل
 * - default = 0 (مسموح) لو مفيش LSM
 */
```

**مثال على hook إخباري فقط:**
```c
LSM_HOOK(void, LSM_RET_VOID, bprm_committed_creds,
         const struct linux_binprm *bprm)
/*
 * بعد ما الـ credentials اتغيرت لـ exec جديد
 * الـ LSM بيتعلم بس — مش بيقدر يرفض
 */
```

---

### رحلة الـ Hook من التعريف للتنفيذ

```
lsm_hook_defs.h
      │
      ├──► security/security.c
      │    (بيعمل security_file_open() وبيستدعي كل LSMs المسجلة)
      │
      ├──► include/linux/lsm_hooks.h
      │    (بيعرّف union security_list_options و struct lsm_static_calls_table)
      │
      └──► security/selinux/hooks.c
           security/apparmor/lsm.c
           security/smack/smack_lsm.c
           (كل LSM بيسجل callbacks بتاعته باستخدام LSM_HOOK_INIT)
```

**Flow التنفيذ الفعلي:**

```
app calls open(2)
    │
    ▼
vfs_open() in fs/open.c
    │
    ▼
security_file_open(file)    ← generated from lsm_hook_defs.h
    │
    ├── calls selinux_file_open()   → checks SELinux policy
    ├── calls apparmor_file_open()  → checks AppArmor profile
    └── returns 0 (OK) or -EACCES (denied)
```

---

### ليه الـ Static Calls مش Function Pointers عادية؟

الـ kernel قديماً كان بيستخدم `struct security_operations` بـ function pointers عادية — كل call بيعمل indirect branch. بعدين اتحول لـ **static calls** لأسباب:

1. **Performance**: الـ indirect branch أبطأ بكتير من الـ direct call (خصوصاً مع Spectre mitigations)
2. **Stacking**: دلوقتي ممكن أكتر من LSM يشتغل في نفس الوقت (SELinux + AppArmor معاً)
3. **Security**: الـ `__randomize_layout` على الـ structs بيصعّل exploitation

---

### الفايلات المهمة في الـ Subsystem

**Core Framework:**

| الملف | الدور |
|---|---|
| `include/linux/lsm_hook_defs.h` | **الملف ده** — تعريف كل الـ hooks كـ X-macros |
| `include/linux/lsm_hooks.h` | الـ structs اللي بتتبنى من الـ hook defs |
| `include/linux/security.h` | الـ API العام اللي الـ kernel بيستخدمه |
| `include/uapi/linux/lsm.h` | الـ IDs والـ constants اللي userspace بيشوفها |
| `security/security.c` | الـ dispatcher — بيستدعي كل الـ LSMs المسجلة |

**LSM Implementations:**

| الملف | الـ LSM |
|---|---|
| `security/selinux/hooks.c` | SELinux |
| `security/apparmor/lsm.c` | AppArmor |
| `security/smack/smack_lsm.c` | Smack |
| `security/tomoyo/tomoyo.c` | TOMOYO |
| `security/landlock/setup.c` | Landlock |
| `security/bpf/hooks.c` | BPF LSM |
| `security/ipe/hooks.c` | IPE (Integrity Policy Enforcement) |
| `security/commoncap.c` | Capabilities (always loaded first) |

**Testing:**

| الملف | الدور |
|---|---|
| `tools/testing/selftests/lsm/` | Userspace tests للـ LSM framework |

---

### خلاصة بجملة واحدة

الـ `lsm_hook_defs.h` هو **الـ master list** للنقاط الأمنية في الـ kernel — كل نقطة فيه بتمثل سؤال "هل مسموح بكده؟" بيتسأله الـ kernel لكل الـ security modules المسجلة قبل ما يكمل أي عملية حساسة.
## Phase 2: شرح الـ LSM (Linux Security Module) Framework

---

### المشكلة اللي بيحلّها الـ LSM

الـ kernel بطبيعته بيتعامل مع security بشكل بسيط — DAC (Discretionary Access Control) عن طريق uid/gid و permissions bits. ده كافي في بيئات بسيطة، لكن في البيئات اللي بتحتاج:

- **Mandatory Access Control (MAC)**: زي SELinux و AppArmor — كل process ليها label، والـ policy بتقول مين يوصل لإيه.
- **Integrity measurement**: زي IMA — بتتأكد إن الملفات ما اتغيرتش.
- **Capability brokering**: زي Smack — بتدي labels للـ objects والـ subjects.

المشكلة إن كل نظام أمان بيحتاج يتدخل في نقاط مختلفة جوه الـ kernel: لما process بتفتح file، لما بتعمل `exec`، لما socket بيتعمل، إلخ. لو حطّينا كل نظام أمان hardcoded جوه الـ kernel code، هيبقى:

1. المنتجات المختلفة (Android / SELinux, Ubuntu / AppArmor, RHEL / SELinux) مش هتقدر تشغّل kernel واحد.
2. تضارب بين الـ security modules.
3. صعوبة إضافة نظام أمان جديد من غير ما تخش في كل الـ kernel subsystems.

---

### الحل: الـ Hook-Based Plugin Architecture

الـ LSM framework بيحل المشكلة دي بطريقة أنيقة جداً — **hook points** مدفونة في الـ kernel code في كل نقطة أمنية حساسة. كل LSM بيسجّل نفسه كـ "plugin" وبيقدم callbacks لأي hooks يهمّوه. الـ kernel بينادي الـ hooks دي بالترتيب عند كل قرار أمني.

الفكرة الجوهرية:
```
kernel code → يوصل نقطة أمنية → ينادي LSM hook → كل LSM مسجّل بيرد
```

لو أي LSM رجع error ← الـ kernel يرفض العملية.

---

### تشبيه واقعي: نظام الأمن في المطار

تخيل المطار الدولي:

| مكوّن الـ Airport | المقابل في LSM |
|---|---|
| المطار نفسه (البنية التحتية) | الـ Linux kernel |
| نقاط التفتيش (Security Checkpoints) | الـ LSM hook points |
| إدارة المطار (Airport Authority) | الـ LSM framework نفسه |
| شركات الأمن المتعاقدة (Guards) | كل LSM: SELinux, AppArmor, Smack |
| المسافر بالـ passport | الـ `struct cred` |
| الحقيبة | الـ `struct file` / `struct inode` |
| تصريح الدخول (المنطقة X) | الـ security label/context |

**المطار (kernel)** ما بيعرفش تفاصيل سياسة كل شركة أمن. هو بس بيقول: "في نقطة التفتيش دي، سألوا كل شركات الأمن المتعاقدة. لو أي شركة قالت لأ → ممنوع."

**شركات الأمن (LSMs)** بتقدر تشتغل بالتوازي. الواحدة بتقيّم passport والتانية بتشيل الـ x-ray على الحقيبة. كلهم لازم يوافقوا عشان المسافر يعدّي.

**تصريح الدخول (security label)** ما بيتشافش من المسافر (process) نفسه — شركات الأمن هي اللي بتعيّنه وبتقيّمه.

**إدارة المطار (LSM framework)** بتحدد: ترتيب شركات الأمن، وفين نقاط التفتيش، وإزاي تتجمع ردودهم.

---

### البنية الكبيرة (Architecture)

```
╔══════════════════════════════════════════════════════════════════════╗
║                        User Space                                    ║
║   open()  execve()  socket()  mount()  kill()  ioctl()  mmap()      ║
╚═══════════════╤══════════════════════════════════════════════════════╝
                │  syscall
╔══════════════╧══════════════════════════════════════════════════════╗
║                    Kernel VFS / Networking / IPC                    ║
║                                                                     ║
║   vfs_open()          vfs_mkdir()         sock_create()            ║
║       │                   │                    │                    ║
║       ▼                   ▼                    ▼                    ║
║  ┌─────────┐        ┌─────────┐         ┌──────────┐               ║
║  │ LSM Hook│        │ LSM Hook│         │ LSM Hook │               ║
║  │file_open│        │inode_   │         │socket_   │               ║
║  │         │        │mkdir    │         │create    │               ║
║  └────┬────┘        └────┬────┘         └────┬─────┘               ║
╚═══════╪═════════════════╪══════════════════╪═══════════════════════╝
        │                 │                  │
╔═══════╪═════════════════╪══════════════════╪══════════════╗
║       │    LSM Framework (security/security.c)            ║
║       ▼                 ▼                  ▼              ║
║  ┌─────────────────────────────────────────────────┐     ║
║  │        static_calls_table                       │     ║
║  │  .file_open[0] → SELinux hook                  │     ║
║  │  .file_open[1] → AppArmor hook                 │     ║
║  │  .inode_mkdir[0] → SELinux hook                │     ║
║  │  ...                                           │     ║
║  └─────────────────────────────────────────────────┘     ║
║       │                 │                  │              ║
║       ▼                 ▼                  ▼              ║
║  ┌─────────┐      ┌──────────┐      ┌──────────┐         ║
║  │ SELinux │      │AppArmor  │      │  Smack   │         ║
║  │  LSM    │      │  LSM     │      │  LSM     │         ║
║  └─────────┘      └──────────┘      └──────────┘         ║
╚══════════════════════════════════════════════════════════╝
        │
╔═══════╧══════════════════╗
║   Security Blobs         ║
║  inode->i_security       ║
║  file->f_security        ║
║  cred->security          ║
║  sock->sk_security       ║
╚══════════════════════════╝
```

---

### الـ Core Abstraction: الـ `LSM_HOOK` Macro

الملف `lsm_hook_defs.h` هو **قلب الـ framework**. هو ما بيتضمّنش كود قابل للتنفيذ — هو بس **قائمة تعريفية** لكل الـ hooks موجودة في الـ kernel.

الفكرة الجنية: نفس الملف ده بيتضمّن **3 مرات** بأشكال مختلفة:

```c
// المرة الأولى: لبناء struct security_hook_heads (hlist لكل hook)
struct security_hook_heads {
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) struct hlist_head NAME;
    #include <linux/lsm_hook_defs.h>
    #undef LSM_HOOK
};

// المرة التانية: لبناء union security_list_options (function pointers)
union security_list_options {
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) RET (*NAME)(__VA_ARGS__);
    #include "lsm_hook_defs.h"
    #undef LSM_HOOK
    void *lsm_func_addr;
};

// المرة التالتة: لبناء جدول الـ static calls
struct lsm_static_calls_table {
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) \
        struct lsm_static_call NAME[MAX_LSM_COUNT];
    #include <linux/lsm_hook_defs.h>
    #undef LSM_HOOK
} __packed __randomize_layout;
```

ده **X-Macro pattern** — تقنية C قوية جداً: تعرّف البيانات مرة واحدة وبتستخدمها لتوليد structs وenums وcode تلقائياً.

---

### تشريح الـ `LSM_HOOK` Macro

```c
LSM_HOOK(int, 0, file_open, struct file *file)
```

| الحقل | القيمة | المعنى |
|---|---|---|
| `RET` | `int` | نوع القيمة المرجعة |
| `DEFAULT` | `0` | القيمة لو مفيش LSM بيimplementها (0 = allow) |
| `NAME` | `file_open` | اسم الـ hook |
| `args...` | `struct file *file` | الـ arguments اللي بتتبعت للـ hook |

لما `DEFAULT = 0` ده معناه: لو مفيش LSM مسجّل للـ hook ده، الكيرنل بيسمح بالعملية افتراضياً.

لما `DEFAULT = -EOPNOTSUPP`: الـ hook مش موجود في الـ capability الافتراضية — لازم LSM يشيّله explicitly.

مثال:
```c
// default 0 = allow بالـ default
LSM_HOOK(int, 0, inode_permission, struct inode *inode, int mask)

// default -EOPNOTSUPP = مش موجود في الـ base kernel
LSM_HOOK(int, -EOPNOTSUPP, inode_getsecurity, struct mnt_idmap *idmap,
         struct inode *inode, const char *name, void **buffer, bool alloc)
```

---

### الـ Structs الأساسية وعلاقاتها ببعض

```
┌──────────────────────────────────────────────────────┐
│                   struct lsm_info                    │
│  (يعرّف LSM كامل — يتوضع في .lsm_info.init section) │
│                                                      │
│  .id      ──────────────► struct lsm_id              │
│                             .name = "selinux"        │
│                             .id   = LSM_ID_SELINUX   │
│  .blobs   ──────────────► struct lsm_blob_sizes      │
│                             .lbs_inode = 128         │
│                             .lbs_cred  = 32          │
│  .init    = selinux_init()                           │
│  .order   = LSM_ORDER_MUTABLE                        │
└──────────────────────────────────────────────────────┘
          │
          │ (عند init)
          ▼
┌──────────────────────────────────────────────────────┐
│              struct security_hook_list[]              │
│  (array من الـ hooks اللي الـ LSM ده بيsupportها)   │
│                                                      │
│  [0]: .scalls ──► static_calls_table.file_open       │
│       .hook   = { .file_open = selinux_file_open }   │
│       .lsmid  ──► lsm_id (selinux)                  │
│                                                      │
│  [1]: .scalls ──► static_calls_table.inode_perm      │
│       .hook   = { .inode_permission = selinux_..}    │
│       .lsmid  ──► lsm_id (selinux)                  │
└──────────────────────────────────────────────────────┘
          │
          │ security_add_hooks()
          ▼
┌──────────────────────────────────────────────────────┐
│          struct lsm_static_calls_table               │
│  (الجدول المركزي - ro_after_init)                    │
│                                                      │
│  .file_open[0] = {.key, .trampoline, .hl, .active}   │
│  .file_open[1] = {.key, .trampoline, .hl, .active}   │
│  .inode_permission[0] = { ... }                      │
│  .socket_create[0]    = { ... }                      │
│  ...                                                 │
└──────────────────────────────────────────────────────┘
```

---

### الـ Static Calls Optimization

ده مفهوم مهم جداً — الـ LSM framework في الـ kernels الحديثة ما بيستخدمش indirect function calls عادية (اللي بتكون بطيئة بسبب Spectre mitigations). بدلاً من كده بيستخدم **static calls**:

```
// Call عادي (بطيء بعد Spectre patches):
(*func_ptr)(args);   // ← indirect branch، CPU speculation problem

// Static call (سريع):
static_call(lsm_file_open)(file);  // ← direct jump، patched at runtime
```

الـ `struct lsm_static_call` بيحتفظ بـ:
- `key`: الـ static call key (بيتعدّل عند register الـ LSM)
- `trampoline`: الكود اللي بينقل للـ hook الحقيقي
- `hl`: مين صاحب الـ hook ده
- `active`: static key لتفعيل/تعطيل الـ hook بكفاءة

---

### فئات الـ Hooks الموجودة في `lsm_hook_defs.h`

#### 1. Binder Hooks (Android IPC)
```c
LSM_HOOK(int, 0, binder_set_context_mgr, const struct cred *mgr)
LSM_HOOK(int, 0, binder_transaction, const struct cred *from, const struct cred *to)
LSM_HOOK(int, 0, binder_transfer_file, const struct cred *from,
         const struct cred *to, const struct file *file)
```
الـ Android بيستخدم Binder كـ IPC mechanism أساسي. الـ hooks دي بتخلي SELinux يتحكم في مين يقدر يتكلم مع مين عبر Binder.

#### 2. Task و Credential Hooks
```c
LSM_HOOK(int, 0, task_alloc, struct task_struct *task, u64 clone_flags)
LSM_HOOK(int, 0, cred_prepare, struct cred *new, const struct cred *old, gfp_t gfp)
LSM_HOOK(void, LSM_RET_VOID, cred_transfer, struct cred *new, const struct cred *old)
```
لما process بيعمل `fork()` أو `exec()` الـ LSM لازم يعمل copy للـ security context الخاص بيه. الـ `struct cred` هو المكان اللي بيتحفظ فيه identity الـ process — الـ LSM بيضيف security blob فيه.

#### 3. Filesystem Hooks
```c
// Super Block (whole filesystem)
LSM_HOOK(int, 0, sb_mount, const char *dev_name, const struct path *path, ...)
LSM_HOOK(int, 0, sb_kern_mount, const struct super_block *sb)

// Inode (individual files)
LSM_HOOK(int, 0, inode_permission, struct inode *inode, int mask)
LSM_HOOK(int, 0, inode_create, struct inode *dir, struct dentry *dentry, umode_t mode)
LSM_HOOK(int, -EOPNOTSUPP, inode_init_security, struct inode *inode, ...)

// File (open file descriptor)
LSM_HOOK(int, 0, file_open, struct file *file)
LSM_HOOK(int, 0, file_permission, struct file *file, int mask)
```
فارق جوهري:
- **inode_permission**: يتنادى لما الـ kernel بيحتاج يتأكد من permissions على الـ inode نفسه
- **file_permission**: يتنادى على الـ open file descriptor — ممكن يبقى أكثر تقييداً من الـ inode

#### 4. Network Hooks
```c
LSM_HOOK(int, 0, socket_create, int family, int type, int protocol, int kern)
LSM_HOOK(int, 0, socket_connect, struct socket *sock, struct sockaddr *address, int addrlen)
LSM_HOOK(int, 0, socket_sock_rcv_skb, struct sock *sk, struct sk_buff *skb)
```
ده بيخلي LSM زي SELinux يعمل network labeling — كل socket ليها security context، وكل packet لازم يتطابق معه. ده الأساس لـ Labeled IPsec و CIPSO.

#### 5. IPC Hooks (SysV IPC)
```c
LSM_HOOK(int, 0, msg_queue_msgsnd, struct kern_ipc_perm *perm, struct msg_msg *msg, int msqflg)
LSM_HOOK(int, 0, shm_shmat, struct kern_ipc_perm *perm, char __user *shmaddr, int shmflg)
LSM_HOOK(int, 0, sem_semop, struct kern_ipc_perm *perm, struct sembuf *sops, ...)
```
الـ SysV IPC (shared memory, message queues, semaphores) ليه hooks منفصلة تماماً عن الـ filesystem.

#### 6. BPF Hooks
```c
LSM_HOOK(int, 0, bpf, int cmd, union bpf_attr *attr, unsigned int size, bool kernel)
LSM_HOOK(int, 0, bpf_prog_load, struct bpf_prog *prog, union bpf_attr *attr, ...)
LSM_HOOK(int, 0, bpf_token_cmd, const struct bpf_token *token, enum bpf_cmd cmd)
```
BPF بقى قوي جداً لدرجة إنه بحاجة لـ security hooks خاصة بيه. LSM BPF نفسه بيستخدم الـ framework اللي بيحميه!

#### 7. Kernel Integrity Hooks
```c
LSM_HOOK(int, 0, kernel_load_data, enum kernel_load_data_id id, bool contents)
LSM_HOOK(int, 0, kernel_read_file, struct file *file, enum kernel_read_file_id id, bool contents)
LSM_HOOK(int, 0, locked_down, enum lockdown_reason what)
```
الـ IMA (Integrity Measurement Architecture) بيستخدم الـ hooks دي عشان يتأكد إن الـ kernel modules والـ firmware موقّعة وسليمة.

---

### الـ Security Blobs: إزاي كل LSM بيخزّن بياناته

كل LSM بيحتاج يحط بيانات خاصة بيه على الـ kernel objects (inodes, files, processes...). الـ framework بيوفّرله **blob** — مساحة ميموري جنب كل object.

```c
struct lsm_blob_sizes {
    unsigned int lbs_cred;       // مساحة في struct cred
    unsigned int lbs_file;       // مساحة في struct file
    unsigned int lbs_inode;      // مساحة في struct inode
    unsigned int lbs_sock;       // مساحة في struct sock
    unsigned int lbs_superblock; // مساحة في struct super_block
    unsigned int lbs_task;       // مساحة في struct task_struct
    // ...
};
```

```
struct inode {
    ...
    void *i_security;   // ← الـ blob
}

تخطيط الـ blob في الميموري لما في 2 LSMs:
┌────────────────────────┐  ← i_security
│  SELinux inode_ctx     │  (128 bytes)
│  (sid, sclass, etc.)  │
├────────────────────────┤  ← offset: 128
│  AppArmor inode_data  │  (16 bytes)
└────────────────────────┘
```

الـ framework بيحسب الـ offsets ده تلقائياً أثناء boot ولما كل LSM بيسجّل `lsm_blob_sizes`. كل LSM بيوصل لـ blob بتاعه عن طريق offset ثابت معروفله.

---

### مثال كامل: رحلة `open()` عبر الـ LSM

```c
// User space
fd = open("/etc/shadow", O_RDONLY);

// Kernel: sys_open → do_sys_open → do_filp_open → vfs_open
int vfs_open(const struct path *path, struct file *file)
{
    // ... VFS checks (inode existence, basic perms) ...

    // 1) تتنادى أول ما الـ inode بيتحوّل لـ file object
    error = security_file_open(file);  // LSM hook dispatcher
    if (error)
        return error;

    // ... باقي الكود ...
}

// في security/security.c:
int security_file_open(struct file *file)
{
    // بينادي كل LSM مسجّل على hook ده بالترتيب
    // لو أي واحد رجع error → تقف
    int ret = call_int_hook(file_open, file);
    ...
    return ret;
}

// في SELinux:
static int selinux_file_open(struct file *file)
{
    struct file_security_struct *fsec;
    struct inode_security_struct *isec;

    // بياخد الـ security blob من الـ file
    fsec = selinux_file(file);
    isec = inode_security(file_inode(file));

    // بيعمل AVC check (Access Vector Cache)
    return avc_has_perm(&selinux_state,
                        fsec->sid, isec->sid,
                        isec->sclass, FILE__OPEN, &ad);
}
```

---

### الـ Hook Registration: إزاي LSM بيسجّل نفسه

```c
// في security/selinux/hooks.c

static struct security_hook_list selinux_hooks[] __ro_after_init = {
    LSM_HOOK_INIT(binder_set_context_mgr, selinux_binder_set_context_mgr),
    LSM_HOOK_INIT(binder_transaction,     selinux_binder_transaction),
    LSM_HOOK_INIT(file_open,              selinux_file_open),
    LSM_HOOK_INIT(inode_permission,       selinux_inode_permission),
    LSM_HOOK_INIT(socket_connect,         selinux_socket_connect),
    // ... وهكذا لمئات الـ hooks
};

// الـ macro بيتحوّل لـ:
// { .scalls = static_calls_table.file_open,
//   .hook   = { .file_open = selinux_file_open },
//   .lsmid  = &selinux_lsmid }

static int __init selinux_init(void)
{
    security_add_hooks(selinux_hooks,
                       ARRAY_SIZE(selinux_hooks),
                       &selinux_lsmid);
    return 0;
}

// يتسجّل في الـ initcall system:
DEFINE_LSM(selinux) = {
    .id    = &selinux_lsmid,
    .flags = LSM_FLAG_LEGACY_MAJOR | LSM_FLAG_EXCLUSIVE,
    .blobs = &selinux_blob_sizes,
    .init  = selinux_init,
};
```

الـ `DEFINE_LSM` macro بيحط الـ `struct lsm_info` في section خاص في الـ binary `.lsm_info.init`، والـ framework بيلاقيهم أثناء boot ويشغّلهم.

---

### اللي الـ LSM Framework بيملكه مقابل اللي بيفوّضه

| الـ LSM Framework يملك | الـ LSM Plugin (SELinux مثلاً) يملك |
|---|---|
| تحديد نقاط الـ hook | logic الـ policy بتاعته |
| ترتيب استدعاء الـ LSMs | تفسير الـ labels والـ contexts |
| توزيع الـ security blobs | تخزين وتحديث الـ security state |
| تجميع نتائج الـ hooks | قرار السماح/الرفض |
| الـ sysfs interface | audit logging |
| أمان الـ static calls | تهيئة الـ policy من الـ userspace |
| تعريف جميع hook signatures | اختيار الـ hooks اللي يهمّوه |

---

### الـ CONFIG Guards في `lsm_hook_defs.h`

```c
#ifdef CONFIG_SECURITY_NETWORK
LSM_HOOK(int, 0, socket_create, ...)
LSM_HOOK(int, 0, socket_connect, ...)
#endif

#ifdef CONFIG_SECURITY_NETWORK_XFRM
LSM_HOOK(int, 0, xfrm_policy_alloc_security, ...)
#endif

#ifdef CONFIG_KEYS
LSM_HOOK(int, 0, key_alloc, ...)
#endif

#ifdef CONFIG_BPF_SYSCALL
LSM_HOOK(int, 0, bpf, ...)
#endif
```

ده مهم جداً للـ embedded Linux — الـ hooks دي بس بتتبني لو الـ feature المقابلة enabled في الـ config. على ARM embedded device مش محتاج networking أو BPF، الـ kernel بيبقى أصغر وأسرع.

---

### الـ lsm_context و lsm_prop: واجهة الـ Userspace

```c
// security context: التمثيل النصي للـ label
struct lsm_context {
    char *context;  // "system_u:system_r:kernel_t:s0" (SELinux)
    u32   len;
    int   id;       // مين الـ LSM اللي أنتجه
};

// lsm_prop: تمثيل داخلي composite من كل الـ LSMs
struct lsm_prop {
    struct lsm_prop_selinux  selinux;
    struct lsm_prop_smack    smack;
    struct lsm_prop_apparmor apparmor;
    struct lsm_prop_bpf      bpf;
};
```

الـ hooks زي `inode_getlsmprop` و `cred_getlsmprop` بتملّي الـ `lsm_prop` بالـ security identifiers من كل LSM. الـ `lsmprop_to_secctx` بيحوّلها لنص يفهمه الـ userspace.

الـ hook:
```c
LSM_HOOK(void, LSM_RET_VOID, inode_getlsmprop,
         struct inode *inode, struct lsm_prop *prop)
```

ده بيتنادى لما حاجة زي `ls -Z` تحتاج تعرف الـ security label لـ file.

---

### ملاحظة على الـ `__randomize_layout`

```c
struct lsm_static_call { ... } __randomize_layout;
struct lsm_static_calls_table { ... } __packed __randomize_layout;
struct security_hook_list { ... } __randomize_layout;
```

الـ `__randomize_layout` بيشوّش ترتيب الـ fields في الـ struct بشكل عشوائي عند كل compile. ده جزء من RANDSTRUCT — hardening ضد الـ exploits اللي بتعتمد على معرفة offset ثابت لـ field معيّن في الـ kernel struct. الـ LSM framework بيحميه بالكامل بالطريقة دي لأنه هو نفسه بنية أمنية.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### الـ Config Options اللي بتتحكم في الـ hooks المتاحة

| Config Option | الـ Hooks المحمية بيها |
|---|---|
| `CONFIG_SECURITY_PATH` | `path_unlink`, `path_mkdir`, `path_rmdir`, `path_mknod`, `path_post_mknod`, `path_truncate`, `path_symlink`, `path_link`, `path_rename`, `path_chmod`, `path_chown`, `path_chroot` |
| `CONFIG_SECURITY_NETWORK` | كل socket/unix/sk/inet/sctp/tun/mptcp hooks |
| `CONFIG_SECURITY_INFINIBAND` | `ib_pkey_access`, `ib_endport_manage_subnet`, `ib_alloc_security` |
| `CONFIG_SECURITY_NETWORK_XFRM` | كل `xfrm_*` hooks |
| `CONFIG_KEYS` | `key_alloc`, `key_permission`, `key_getsecurity`, `key_post_create_or_update` |
| `CONFIG_AUDIT` | `audit_rule_init`, `audit_rule_known`, `audit_rule_match`, `audit_rule_free` |
| `CONFIG_BPF_SYSCALL` | كل `bpf_*` و `bpf_map_*` و `bpf_prog_*` و `bpf_token_*` hooks |
| `CONFIG_PERF_EVENTS` | `perf_event_open`, `perf_event_alloc`, `perf_event_read`, `perf_event_write` |
| `CONFIG_IO_URING` | `uring_override_creds`, `uring_sqpoll`, `uring_cmd`, `uring_allowed` |
| `CONFIG_WATCH_QUEUE` | `post_notification` |
| `CONFIG_KEY_NOTIFICATIONS` | `watch_key` |

#### الـ LSM Flags

| Flag | القيمة | المعنى |
|---|---|---|
| `LSM_FLAG_LEGACY_MAJOR` | `BIT(0)` | LSM قديم بنمط major واحد (SELinux مثلاً) |
| `LSM_FLAG_EXCLUSIVE` | `BIT(1)` | مش ممكن يشتغل مع LSM تاني في نفس الوقت |
| `LSM_RET_VOID` | `((void) 0)` | الـ default value للـ hooks اللي بترجع void |

#### الـ enum lsm_order

| قيمة | المعنى |
|---|---|
| `LSM_ORDER_FIRST = -1` | capabilities فقط — بتيجي الأول دايماً |
| `LSM_ORDER_MUTABLE = 0` | الترتيب العادي القابل للتغيير |
| `LSM_ORDER_LAST = 1` | integrity فقط — بتيجي الأخير دايماً |

#### الـ enum lsm_integrity_type

| قيمة | المعنى |
|---|---|
| `LSM_INT_DMVERITY_SIG_VALID` | التوقيع على dm-verity صالح |
| `LSM_INT_DMVERITY_ROOTHASH` | الـ root hash بتاع dm-verity |
| `LSM_INT_FSVERITY_BUILTINSIG_VALID` | توقيع fs-verity المدمج صالح |

#### الـ enum lsm_event

| قيمة | المعنى |
|---|---|
| `LSM_POLICY_CHANGE` | السياسة الأمنية اتغيرت |
| `LSM_STARTED_ALL` | كل الـ LSMs اتشغلت |

#### الـ enum lockdown_reason — أهم القيم

| قيمة | الفئة | المعنى |
|---|---|---|
| `LOCKDOWN_MODULE_SIGNATURE` | integrity | منع تحميل modules من غير توقيع |
| `LOCKDOWN_KEXEC` | integrity | منع kexec |
| `LOCKDOWN_INTEGRITY_MAX` | فاصل | كل اللي قبله integrity |
| `LOCKDOWN_KCORE` | confidentiality | منع قراءة `/proc/kcore` |
| `LOCKDOWN_PERF` | confidentiality | تقييد perf events |
| `LOCKDOWN_CONFIDENTIALITY_MAX` | فاصل | كل اللي قبله confidentiality |

#### الـ LSM_SETID Flags (للـ task_fix_setuid/setgid)

| Flag | القيمة | المعنى |
|---|---|---|
| `LSM_SETID_ID` | 1 | `setuid` أو `setgid` |
| `LSM_SETID_RE` | 2 | `setreuid` أو `setregid` |
| `LSM_SETID_RES` | 4 | `setresuid` أو `setresgid` |
| `LSM_SETID_FS` | 8 | `setfsuid` أو `setfsgid` |

#### الـ bprm->unsafe Flags

| Flag | القيمة | المعنى |
|---|---|---|
| `LSM_UNSAFE_SHARE` | 1 | الـ process بيشارك memory مع process تاني |
| `LSM_UNSAFE_PTRACE` | 2 | تحت ptrace |
| `LSM_UNSAFE_NO_NEW_PRIVS` | 4 | `no_new_privs` مفعّل |

---

### 1. الـ Structs المهمة

#### `union security_list_options`

الـ union ده بيحتوي على **pointer** لكل hook function ممكنة في النظام. بيتبني باستخدام macro expansion:

```c
union security_list_options {
    /* بيتوسّع لـ function pointer لكل hook */
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) RET (*NAME)(__VA_ARGS__);
    #include "lsm_hook_defs.h"
    #undef LSM_HOOK
    void *lsm_func_addr; /* raw address لأغراض الـ static call */
};
```

**الغرض:** تخزين الـ callback الخاص بـ LSM واحد لـ hook واحد. الـ union بيضمن إن كل الـ function pointers بتبدأ من نفس العنوان، وده بيسهّل الـ static call dispatch.

**الـ fields:** كل hook اسمه field في الـ union، زي `hook.inode_permission` أو `hook.file_open`. الـ field الأخير `lsm_func_addr` بيُستخدم كـ raw pointer للـ static call machinery.

---

#### `struct lsm_static_call`

```c
struct lsm_static_call {
    struct static_call_key *key;       /* مفتاح الـ static call */
    void *trampoline;                  /* الـ trampoline المرتبط بالـ key */
    struct security_hook_list *hl;     /* الـ hook list اللي بيخدمه */
    struct static_key_false *active;   /* enabled لو فيه LSM مسجّل */
} __randomize_layout;
```

**الغرض:** تمثيل **نقطة استدعاء واحدة** لـ hook واحد لـ LSM واحد. بدل ما الـ kernel يعمل indirect call عبر قائمة، بيستخدم static call مباشر في الـ code للـ performance. الـ `__randomize_layout` بيخلي layout الـ struct عشوائي في كل compile لصعوبة الـ exploitation.

**العلاقة مع غيره:**
- بيشير لـ `security_hook_list` عبر `hl`
- الـ `key` هو الـ static call key اللي بيتحوّل لـ direct call بعد الـ patching

---

#### `struct lsm_static_calls_table`

```c
struct lsm_static_calls_table {
    /* لكل hook: array بحجم MAX_LSM_COUNT من الـ static calls */
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) \
        struct lsm_static_call NAME[MAX_LSM_COUNT];
    #include <linux/lsm_hook_defs.h>
    #undef LSM_HOOK
} __packed __randomize_layout;
```

**الغرض:** **الجدول الرئيسي** للـ kernel. بيحتوي على array من الـ `lsm_static_call` لكل hook. الـ array بتتملى من الآخر للأول — أي إن أول LSM مسجّل بياخد آخر slot، وده بيخلي الـ dispatch يبدأ من أول slot مليان بشكل sequential.

**الـ `__packed`:** بيمنع الـ padding بين الـ fields.

**الـ `extern`:** الـ instance الوحيدة في الـ kernel:
```c
extern struct lsm_static_calls_table static_calls_table __ro_after_init;
```
الـ `__ro_after_init` بيخلي الجدول read-only بعد ما الـ init ينتهي — حماية ضد تعديل الـ hooks runtime.

---

#### `struct lsm_id`

```c
struct lsm_id {
    const char *name; /* اسم الـ LSM، زي "selinux" أو "apparmor" */
    u64 id;           /* رقم من uapi/linux/lsm.h */
};
```

**الغرض:** هوية الـ LSM. الـ `id` بيتطابق مع الـ UAPI constants اللي userspace بيستخدمها في `lsm_get_self_attr()`.

---

#### `struct security_hook_list`

```c
struct security_hook_list {
    struct lsm_static_call *scalls; /* pointer لـ array الـ static calls للـ hook ده */
    union security_list_options hook; /* الـ callback نفسه */
    const struct lsm_id *lsmid;     /* هوية الـ LSM المالك */
} __randomize_layout;
```

**الغرض:** يمثّل **تسجيل hook واحد من LSM واحد**. لما LSM بيسجّل نفسه، بيوفّر array من `security_hook_list` — كل عنصر فيها hook واحد.

**كيف بيتبنى:**
```c
/* الـ macro LSM_HOOK_INIT بيسهّل الإنشاء */
#define LSM_HOOK_INIT(NAME, HOOK) \
    { \
        .scalls = static_calls_table.NAME, \
        .hook = { .NAME = HOOK } \
    }
```

---

#### `struct lsm_blob_sizes`

```c
struct lsm_blob_sizes {
    unsigned int lbs_cred;         /* حجم blob في struct cred */
    unsigned int lbs_file;         /* حجم blob في struct file */
    unsigned int lbs_ib;           /* حجم blob في InfiniBand */
    unsigned int lbs_inode;        /* حجم blob في struct inode */
    unsigned int lbs_sock;         /* حجم blob في struct sock */
    unsigned int lbs_superblock;   /* حجم blob في super_block */
    unsigned int lbs_ipc;          /* حجم blob في kern_ipc_perm */
    unsigned int lbs_key;          /* حجم blob في struct key */
    unsigned int lbs_msg_msg;      /* حجم blob في struct msg_msg */
    unsigned int lbs_perf_event;   /* حجم blob في perf_event */
    unsigned int lbs_task;         /* حجم blob في task_struct */
    unsigned int lbs_xattr_count;  /* عدد slots للـ xattrs */
    unsigned int lbs_tun_dev;      /* حجم blob للـ TUN device */
    unsigned int lbs_bdev;         /* حجم blob للـ block device */
    unsigned int lbs_bpf_map;      /* حجم blob في bpf_map */
    unsigned int lbs_bpf_prog;     /* حجم blob في bpf_prog */
    unsigned int lbs_bpf_token;    /* حجم blob في bpf_token */
};
```

**الغرض:** كل LSM محتاج مساحة خاصة بيه في الـ kernel objects (inode, cred, file, إلخ). الـ framework بيجمع الـ sizes دي من كل LSM ويحسب الـ offset لكل واحد — زي memory allocator. بدل ما كل LSM يحجز memory منفصلة، بتبقى embedded في الـ object نفسه.

---

#### `struct lsm_info`

```c
struct lsm_info {
    const struct lsm_id *id;          /* هوية الـ LSM */
    enum lsm_order order;             /* ترتيب التهيئة */
    unsigned long flags;              /* LSM_FLAG_* */
    struct lsm_blob_sizes *blobs;     /* حجم الـ security blobs المطلوبة */
    int *enabled;                     /* مؤشر لمتغير التفعيل (CONFIG_LSM) */
    int (*init)(void);                /* دالة التهيئة الرئيسية */
    /* initcalls في مراحل مختلفة من الـ boot */
    int (*initcall_pure)(void);
    int (*initcall_early)(void);
    int (*initcall_core)(void);
    int (*initcall_subsys)(void);
    int (*initcall_fs)(void);
    int (*initcall_device)(void);
    int (*initcall_late)(void);
};
```

**الغرض:** **الـ registration record** الكامل لكل LSM. بيتوضع في section خاص في الـ kernel image:

```c
#define DEFINE_LSM(lsm) \
    static struct lsm_info __lsm_##lsm \
        __used __section(".lsm_info.init") \
        __aligned(sizeof(unsigned long))
```

الـ kernel بيمشي على section `.lsm_info.init` في الـ boot ويشغّل كل LSM بحسب `order` وترتيب `CONFIG_LSM`.

---

#### `struct lsm_context`

```c
struct lsm_context {
    char *context; /* نص الـ security context، مثلاً "system_u:object_r:httpd_t:s0" */
    u32  len;      /* طول الـ string */
    int  id;       /* رقم الـ LSM المالك */
};
```

**الغرض:** تمرير الـ security context بين الـ kernel والـ userspace. الـ `id` بيحدد مين فسّر الـ context ده (SELinux؟ Smack؟).

---

#### `struct lsm_prop`

```c
struct lsm_prop {
    struct lsm_prop_selinux selinux;
    struct lsm_prop_smack   smack;
    struct lsm_prop_apparmor apparmor;
    struct lsm_prop_bpf     bpf;
};
```

**الغرض:** بديل أحدث للـ `secid` — بيحتوي على معلومات من كل LSM في struct واحدة. الـ hooks اللي بتنتهي بـ `getlsmprop` أو `lsmprop_to_secctx` بتستخدمه بدل الـ `u32 secid` القديم.

---

### 2. مخطط العلاقات بين الـ Structs

```
                    ┌─────────────────────────────────────┐
                    │         struct lsm_info              │
                    │  .id ──────────────────────────┐    │
                    │  .order                         │    │
                    │  .flags                         │    │
                    │  .blobs ─────────────┐          │    │
                    │  .enabled            │          │    │
                    │  .init()             │          │    │
                    └─────────────────────┼──────────┼────┘
                                          │          │
                    ┌─────────────────────▼──┐   ┌───▼─────────────┐
                    │   struct lsm_blob_sizes │   │  struct lsm_id  │
                    │  .lbs_cred              │   │  .name          │
                    │  .lbs_file              │   │  .id            │
                    │  .lbs_inode             │   └───┬─────────────┘
                    │  .lbs_sock              │       │
                    │  ... etc               │       │ (used by)
                    └─────────────────────────┘       │
                                                      ▼
                    ┌─────────────────────────────────────────┐
                    │       struct security_hook_list          │
                    │  .scalls ──────────────────────────┐    │
                    │  .hook (union security_list_options)│    │
                    │  .lsmid ───────────────────────────┘────┤→ lsm_id
                    └────────────────────────────────────┬────┘
                                                         │
                                         .scalls points to:
                                                         │
                    ┌────────────────────────────────────▼────┐
                    │      struct lsm_static_calls_table       │
                    │  (global: static_calls_table)            │
                    │                                          │
                    │  .inode_permission[0..MAX_LSM_COUNT-1]   │
                    │  .file_open[0..MAX_LSM_COUNT-1]          │
                    │  .task_kill[0..MAX_LSM_COUNT-1]          │
                    │  ... (one array per hook)               │
                    │                                          │
                    │  each slot is struct lsm_static_call:    │
                    │    .key ─────────────────────────────────┤→ static_call_key
                    │    .trampoline                            │
                    │    .hl ──────────────────────────────────┤→ security_hook_list
                    │    .active ──────────────────────────────┤→ static_key_false
                    └──────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────────────┐
   │              union security_list_options                      │
   │  بيحتوي على pointer لكل hook ممكن:                            │
   │  .binder_set_context_mgr  → int (*)(const struct cred*)      │
   │  .inode_permission        → int (*)(struct inode*, int)      │
   │  .file_open               → int (*)(struct file*)            │
   │  ... (250+ hook pointers)                                    │
   │  .lsm_func_addr           → void* (raw)                     │
   └──────────────────────────────────────────────────────────────┘
```

---

### 3. مخطط الـ Lifecycle — من التسجيل للاستدعاء

```
BOOT TIME
─────────────────────────────────────────────────────────────────

1. الـ linker يجمع كل struct lsm_info في section ".lsm_info.init"
   ┌──────────────────────────────────┐
   │  .lsm_info.init section          │
   │  [selinux_lsm_info]              │
   │  [smack_lsm_info]                │
   │  [apparmor_lsm_info]             │
   │  [bpf_lsm_info]                  │
   └──────────────────────────────────┘

2. early_security_init() / security_init()
   └─ يمشي على الـ section
   └─ بيفرز بحسب lsm_order
   └─ بيستدعي lsm_info->init() لكل LSM

3. داخل init() — الـ LSM بيستدعي security_add_hooks():
   ┌──────────────────────────────────────────────────────┐
   │  security_add_hooks(hooks_array, count, &my_lsm_id)  │
   │    └─ لكل hook في الـ array:                          │
   │       ├─ يحدد الـ slot المناسب في static_calls_table │
   │       ├─ يكتب الـ callback في lsm_static_call.hl     │
   │       └─ يفعّل static_key_false                      │
   └──────────────────────────────────────────────────────┘

4. __ro_after_init يقفّل static_calls_table
   └─ مش ممكن حد يعدّل الـ hooks بعد كده

RUNTIME
─────────────────────────────────────────────────────────────────

5. kernel code بيستدعي security_inode_permission(inode, mask)
   └─ الـ static call بيروح مباشرة للـ trampoline
   └─ الـ trampoline بيلف على الـ active hooks
   └─ كل hook بيرجع int، لو حد رجع error → بيوقف

6. الـ object بيتحرر
   └─ الـ hook الموافق (مثلاً inode_free_security) بيتستدعى
   └─ الـ LSM بيحرر الـ security blob الخاص بيه
```

---

### 4. مخطط الـ Hook Groups والـ Call Flow

#### 4.1 دورة حياة الـ inode

```
inode_alloc_security(inode)           ← عند إنشاء الـ inode
    → LSM يخصص security blob
    → يحط الـ label (مثلاً SELinux type)
         │
         ▼
inode_init_security(inode, dir, qstr, xattrs, count)
    → يحدد الـ xattr المناسب للـ security label
    → يكتب في xattrs array للحفظ على disk
         │
         ▼
inode_permission(inode, mask)         ← عند كل read/write/exec
    → يتحقق من الـ policy
    → 0 = مسموح، -EACCES = مرفوض
         │
         ▼
inode_setxattr / inode_getxattr       ← تعديل/قراءة attributes
inode_setattr / inode_getattr         ← تعديل/قراءة metadata
         │
         ▼
inode_free_security(inode)            ← عند حذف الـ inode
    → LSM يحرر الـ security blob
    ↓
inode_free_security_rcu(inode_security)  ← نسخة RCU-safe
```

#### 4.2 دورة حياة الـ file

```
file_alloc_security(file)     ← عند open()
    → خصص blob للـ file object
         │
         ▼
file_open(file)               ← بعد الـ open مباشرة
    → تحقق من صلاحية الفتح
         │
         ▼
file_permission(file, mask)   ← عند كل read()/write()
    → تحقق من الـ mask
         │
         ▼
file_ioctl(file, cmd, arg)    ← عند ioctl()
         │
         ▼
file_release(file)            ← بعد آخر close()
file_free_security(file)      ← تحرير الـ blob
```

#### 4.3 دورة حياة الـ task/cred

```
fork() / clone():
    task_alloc(task, clone_flags)
        → خصص security blob للـ task الجديد
    cred_prepare(new_cred, old_cred, gfp)
        → انسخ الـ credentials مع الـ security info

setuid() / setgid():
    task_fix_setuid(new, old, flags)
    task_fix_setgid(new, old, flags)
        → تحقق / عدّل الـ security context

execve():
    bprm_creds_for_exec(bprm)
        → هيّئ الـ credentials للـ exec
    bprm_creds_from_file(bprm, file)
        → خذ credentials من الـ binary نفسه
    bprm_check_security(bprm)
        → تحقق نهائي قبل الـ exec
    bprm_committing_creds(bprm)
        → بدأ تطبيق الـ credentials الجديدة
    bprm_committed_creds(bprm)
        → اكتمل تطبيق الـ credentials

task_kill(task, siginfo, sig, cred)
    → هل مسموح ترسل signal لـ task ده؟

exit():
    task_free(task)
    cred_free(cred)
        → حرّر الـ security blobs
```

#### 4.4 دورة حياة الـ socket (مع CONFIG_SECURITY_NETWORK)

```
socket(family, type, protocol):
    socket_create(family, type, protocol, kern)
    socket_post_create(sock, family, type, protocol, kern)
    sk_alloc_security(sk, family, priority)

bind(sock, addr):
    socket_bind(sock, address, addrlen)

connect(sock, addr):
    socket_connect(sock, address, addrlen)

listen(sock):
    socket_listen(sock, backlog)

accept(sock):
    socket_accept(sock, newsock)
    inet_csk_clone(newsk, req)
    sock_graft(sk, parent)

send/recv:
    socket_sendmsg(sock, msg, size)
    socket_recvmsg(sock, msg, size, flags)
    socket_sock_rcv_skb(sk, skb)

close:
    sk_free_security(sk)
```

#### 4.5 Call Flow العام من الـ syscall للـ hook

```
userspace: open("/etc/passwd", O_RDONLY)
    │
    ▼
sys_open() / do_filp_open()
    │
    ▼
security_inode_permission(inode, MAY_READ)
    │  [static call dispatch — no vtable lookup]
    ▼
lsm_static_calls_table.inode_permission[N] (last active)
    │
    ▼
lsm_static_calls_table.inode_permission[N-1]
    │
    ▼  ...
    ▼
selinux_inode_permission(inode, mask)
    → يبحث عن الـ SID للـ subject والـ object
    → يستعلم الـ AVC cache
    → 0 أو -EACCES
    │
    ▼ (لو كل الـ hooks رجعوا 0)
do_filp_open() يكمل
    │
    ▼
file_alloc_security(file)
    ▼
security_file_open(file)
    ▼
fd يرجع للـ userspace
```

---

### 5. الـ Hook Categories — تصنيف شامل

| الفئة | عدد الـ Hooks | أمثلة |
|---|---|---|
| **Binder IPC** | 4 | `binder_set_context_mgr`, `binder_transaction` |
| **Process / Capabilities** | 7 | `ptrace_access_check`, `capable`, `capget`, `capset` |
| **Execution (bprm)** | 5 | `bprm_creds_for_exec`, `bprm_check_security` |
| **Filesystem (sb)** | 12 | `sb_alloc_security`, `sb_mount`, `sb_umount` |
| **Path** | 12 | `path_unlink`, `path_mkdir`, `path_chmod` |
| **Inode** | 35+ | `inode_permission`, `inode_create`, `inode_setxattr` |
| **File** | 14 | `file_open`, `file_permission`, `mmap_file` |
| **Task / Cred** | 20+ | `task_alloc`, `cred_prepare`, `task_kill` |
| **IPC** | 15 | `ipc_permission`, `msg_queue_*`, `shm_*`, `sem_*` |
| **Network** | 30+ | `socket_create`, `socket_connect`, `sk_alloc_security` |
| **XFRM (IPSec)** | 9 | `xfrm_policy_alloc_security`, `xfrm_state_alloc` |
| **Keys** | 4 | `key_alloc`, `key_permission` |
| **BPF** | 10 | `bpf_map_create`, `bpf_prog_load`, `bpf_token_create` |
| **Audit** | 4 | `audit_rule_init`, `audit_rule_match` |
| **Perf Events** | 4 | `perf_event_open`, `perf_event_alloc` |
| **IO Uring** | 4 | `uring_override_creds`, `uring_cmd` |
| **Security Context** | 6 | `secid_to_secctx`, `lsmprop_to_secctx` |
| **Misc** | 5 | `locked_down`, `initramfs_populated`, `bdev_alloc_security` |

---

### 6. الـ Locking Strategy

#### 6.1 الـ static_calls_table — بدون locks في الـ runtime

الـ `static_calls_table` بيتحدد **مرة واحدة فقط** وقت الـ boot بواسطة `security_add_hooks()`. بعد ما الـ init ينتهي، الـ table بتبقى **read-only** بسبب `__ro_after_init`:

```c
extern struct lsm_static_calls_table static_calls_table __ro_after_init;
```

ده معناه:
- **مفيش lock** محتاج في الـ runtime — الـ table immutable
- الـ static calls بتتحوّل لـ direct jumps في الـ code — zero overhead
- لو حاولت تعدّل الـ table بعد الـ boot → kernel panic (page protection)

#### 6.2 الـ security blobs — بتتبع locks الـ object المالك

الـ security blob المدمج في الـ object (inode, cred, إلخ) بيتحمى بنفس lock الـ object:

| الـ Object | الـ Lock |
|---|---|
| `struct inode` | `inode->i_lock` (spinlock) |
| `struct cred` | الـ cred بنفسها immutable بعد الـ commit — `get_cred/put_cred` |
| `struct file` | `file->f_lock` أو الـ reference counting |
| `struct sock` | `sock->sk_lock` |
| `struct kern_ipc_perm` | `kern_ipc_perm->lock` |
| `struct task_struct` | `task_lock(task)` أو `rcu_read_lock()` |

#### 6.3 الـ RCU في الـ hook dispatch

الـ LSM hooks بتتستدعي في contexts مختلفة — بعضها تحت RCU:

```c
/* hook زي ده بيتستدعى في RCU context */
LSM_HOOK(void, LSM_RET_VOID, inode_free_security_rcu, void *inode_security)
```

الـ hooks العادية مش بتحتاج `rcu_read_lock()` لأن الـ static call table نفسها immutable. لكن الـ hooks اللي بتوصل لـ objects ممكن تتحرر بتحتاج تشتغل تحت RCU أو بعد ما تاخد reference.

#### 6.4 ترتيب الـ Locks (Lock Ordering)

```
(من الأعلى للأتحت — مش ينفع تاخد lock تحت وانت شايل lock فوقه)

1. task_lock(task)       أو  rcu_read_lock()
2. inode->i_lock
3. file->f_lock
4. sock->sk_lock
5. kern_ipc_perm->lock
6. security blob lock (لو LSM عنده lock خاص بيه)
```

#### 6.5 ملاحظة مهمة على الـ Hook Ordering

الـ `lsm_order` بيحدد الترتيب اللي بيه الـ hooks بتتستدعى:

```
LSM_ORDER_FIRST (capabilities)     ← أول حاجة بتتستدعى
      ↓
LSM_ORDER_MUTABLE (SELinux, Smack, AppArmor, BPF LSM, إلخ)
      ↓
LSM_ORDER_LAST (integrity / IMA)   ← آخر حاجة بتتستدعى
```

لو أي hook رجع error، الـ framework بيوقف الـ chain مش بيكمّل — **fail-closed** behavior بشكل افتراضي للـ hooks اللي بترجع `int`.
## Phase 4: شرح الـ Functions

---

### ملخص شامل — Cheatsheet

الـ `lsm_hook_defs.h` مش فيه functions بالمعنى التقليدي — هو **قاموس macro-driven** بيعرّف كل الـ hooks اللي ممكن LSM يسجّلها. كل سطر فيه هو `LSM_HOOK(RET, DEFAULT, NAME, args...)` بيتحوّل لـ:

1. **Function pointer** جوه `union security_list_options`
2. **`struct hlist_head NAME`** جوه `struct security_hook_heads`
3. **`struct lsm_static_call NAME[MAX_LSM_COUNT]`** جوه `struct lsm_static_calls_table`
4. **`security_NAME()` wrapper** في `security.h` بيدور على الـ hook ويشغّله

---

### جدول الـ Hooks حسب الـ Category

#### Binder IPC Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `binder_set_context_mgr` | `int` | 0 | السماح/رفض task يبقى context manager |
| `binder_transaction` | `int` | 0 | التحقق من IPC transaction بين processes |
| `binder_transfer_binder` | `int` | 0 | التحقق من نقل binder object |
| `binder_transfer_file` | `int` | 0 | التحقق من نقل file descriptor عبر binder |

#### Process Tracing Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `ptrace_access_check` | `int` | 0 | التحقق قبل ما task يعمل ptrace على child |
| `ptrace_traceme` | `int` | 0 | التحقق من طلب child إنه يتتتبع من parent |

#### Capabilities Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `capget` | `int` | 0 | قراءة capabilities لـ task |
| `capset` | `int` | 0 | تعديل capabilities لـ task |
| `capable` | `int` | 0 | التحقق من capability معينة |

#### System Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `quotactl` | `int` | 0 | التحقق من quota syscall |
| `quota_on` | `int` | 0 | تفعيل quota على dentry |
| `syslog` | `int` | 0 | التحقق من وصول syslog |
| `settime` | `int` | 0 | التحقق من تعديل system time |
| `vm_enough_memory` | `int` | 0 | التحقق من توفر memory كافية |

#### Binary Execution Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `bprm_creds_for_exec` | `int` | 0 | تعيين credentials قبل exec |
| `bprm_creds_from_file` | `int` | 0 | تعيين credentials من الـ file نفسه |
| `bprm_check_security` | `int` | 0 | فحص أمني أخير قبل exec |
| `bprm_committing_creds` | `void` | — | لحظة تطبيق credentials الجديدة |
| `bprm_committed_creds` | `void` | — | بعد تطبيق credentials |

#### Filesystem Context Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `fs_context_submount` | `int` | 0 | submount context security |
| `fs_context_dup` | `int` | 0 | نسخ fs_context |
| `fs_context_parse_param` | `int` | -ENOPARAM | تحليل mount parameter |

#### Superblock Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `sb_alloc_security` | `int` | 0 | تخصيص security blob للـ superblock |
| `sb_delete` | `void` | — | قبل حذف superblock |
| `sb_free_security` | `void` | — | تحرير security blob |
| `sb_free_mnt_opts` | `void` | — | تحرير mount options |
| `sb_eat_lsm_opts` | `int` | 0 | استخلاص LSM options من mount string |
| `sb_mnt_opts_compat` | `int` | 0 | فحص توافق mount options |
| `sb_remount` | `int` | 0 | التحقق عند remount |
| `sb_kern_mount` | `int` | 0 | التحقق عند kernel mount |
| `sb_show_options` | `int` | 0 | عرض mount options في /proc/mounts |
| `sb_statfs` | `int` | 0 | التحقق من statfs |
| `sb_mount` | `int` | 0 | التحقق من mount |
| `sb_umount` | `int` | 0 | التحقق من umount |
| `sb_pivotroot` | `int` | 0 | التحقق من pivot_root |
| `sb_set_mnt_opts` | `int` | 0 | تطبيق security mount options |
| `sb_clone_mnt_opts` | `int` | 0 | نسخ mount options لـ superblock جديد |
| `move_mount` | `int` | 0 | التحقق من نقل mount point |

#### Dentry Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `dentry_init_security` | `int` | -EOPNOTSUPP | تهيئة security context لـ dentry جديد |
| `dentry_create_files_as` | `int` | 0 | تحديد credentials لإنشاء الملفات |

#### Path Hooks (CONFIG_SECURITY_PATH)

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `path_unlink` | `int` | 0 | التحقق من حذف ملف |
| `path_mkdir` | `int` | 0 | التحقق من إنشاء directory |
| `path_rmdir` | `int` | 0 | التحقق من حذف directory |
| `path_mknod` | `int` | 0 | التحقق من إنشاء device node |
| `path_post_mknod` | `void` | — | بعد إنشاء node |
| `path_truncate` | `int` | 0 | التحقق من truncate |
| `path_symlink` | `int` | 0 | التحقق من إنشاء symlink |
| `path_link` | `int` | 0 | التحقق من إنشاء hard link |
| `path_rename` | `int` | 0 | التحقق من rename |
| `path_chmod` | `int` | 0 | التحقق من chmod |
| `path_chown` | `int` | 0 | التحقق من chown |
| `path_chroot` | `int` | 0 | التحقق من chroot |
| `path_notify` | `int` | 0 | التحقق من fanotify/inotify |

#### Inode Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `inode_alloc_security` | `int` | 0 | تخصيص security blob لـ inode |
| `inode_free_security` | `void` | — | تحرير security blob |
| `inode_free_security_rcu` | `void` | — | تحرير في RCU context |
| `inode_init_security` | `int` | -EOPNOTSUPP | تهيئة security xattrs لـ inode جديد |
| `inode_init_security_anon` | `int` | 0 | تهيئة anonymous inode |
| `inode_create` | `int` | 0 | التحقق من إنشاء ملف |
| `inode_post_create_tmpfile` | `void` | — | بعد إنشاء tmpfile |
| `inode_link` | `int` | 0 | التحقق من hard link |
| `inode_unlink` | `int` | 0 | التحقق من حذف |
| `inode_symlink` | `int` | 0 | التحقق من symlink |
| `inode_mkdir` | `int` | 0 | التحقق من mkdir |
| `inode_rmdir` | `int` | 0 | التحقق من rmdir |
| `inode_mknod` | `int` | 0 | التحقق من mknod |
| `inode_rename` | `int` | 0 | التحقق من rename |
| `inode_readlink` | `int` | 0 | التحقق من readlink |
| `inode_follow_link` | `int` | 0 | التحقق من متابعة symlink |
| `inode_permission` | `int` | 0 | التحقق من permission (الأهم) |
| `inode_setattr` | `int` | 0 | التحقق من setattr |
| `inode_post_setattr` | `void` | — | بعد setattr |
| `inode_getattr` | `int` | 0 | التحقق من getattr |
| `inode_xattr_skipcap` | `int` | 0 | تخطي capability check لـ xattr معين |
| `inode_setxattr` | `int` | 0 | التحقق من كتابة xattr |
| `inode_post_setxattr` | `void` | — | بعد كتابة xattr |
| `inode_getxattr` | `int` | 0 | التحقق من قراءة xattr |
| `inode_listxattr` | `int` | 0 | التحقق من list xattrs |
| `inode_removexattr` | `int` | 0 | التحقق من حذف xattr |
| `inode_post_removexattr` | `void` | — | بعد حذف xattr |
| `inode_file_setattr` | `int` | 0 | setattr عبر file descriptor |
| `inode_file_getattr` | `int` | 0 | getattr عبر file descriptor |
| `inode_set_acl` | `int` | 0 | التحقق من set ACL |
| `inode_post_set_acl` | `void` | — | بعد set ACL |
| `inode_get_acl` | `int` | 0 | التحقق من get ACL |
| `inode_remove_acl` | `int` | 0 | التحقق من حذف ACL |
| `inode_post_remove_acl` | `void` | — | بعد حذف ACL |
| `inode_need_killpriv` | `int` | 0 | هل محتاج نمسح setuid/setgid |
| `inode_killpriv` | `int` | 0 | مسح setuid/setgid privileges |
| `inode_getsecurity` | `int` | -EOPNOTSUPP | قراءة security attribute |
| `inode_setsecurity` | `int` | -EOPNOTSUPP | كتابة security attribute |
| `inode_listsecurity` | `int` | 0 | list security attributes |
| `inode_getlsmprop` | `void` | — | استخراج LSM property |
| `inode_copy_up` | `int` | 0 | نسخ inode في overlay |
| `inode_copy_up_xattr` | `int` | -EOPNOTSUPP | نسخ xattr في overlay |
| `inode_setintegrity` | `int` | 0 | تعيين integrity metadata |
| `kernfs_init_security` | `int` | 0 | تهيئة security لـ kernfs node |

#### File Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `file_permission` | `int` | 0 | التحقق من read/write/exec على file مفتوح |
| `file_alloc_security` | `int` | 0 | تخصيص security blob لـ file |
| `file_release` | `void` | — | عند release (قبل free) |
| `file_free_security` | `void` | — | تحرير security blob |
| `file_ioctl` | `int` | 0 | التحقق من ioctl |
| `file_ioctl_compat` | `int` | 0 | التحقق من compat ioctl |
| `mmap_addr` | `int` | 0 | التحقق من mmap على عنوان معين |
| `mmap_file` | `int` | 0 | التحقق من mmap ملف |
| `file_mprotect` | `int` | 0 | التحقق من mprotect |
| `file_lock` | `int` | 0 | التحقق من file lock |
| `file_fcntl` | `int` | 0 | التحقق من fcntl |
| `file_set_fowner` | `void` | — | تسجيل owner للـ async signal |
| `file_send_sigiotask` | `int` | 0 | التحقق من إرسال SIGIO |
| `file_receive` | `int` | 0 | التحقق من استلام file عبر Unix socket |
| `file_open` | `int` | 0 | التحقق عند open |
| `file_post_open` | `int` | 0 | بعد open (للـ audit) |
| `file_truncate` | `int` | 0 | التحقق من truncate عبر file descriptor |

#### Task & Credentials Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `task_alloc` | `int` | 0 | تخصيص security blob عند fork |
| `task_free` | `void` | — | تحرير security blob |
| `cred_alloc_blank` | `int` | 0 | تخصيص cred فارغة |
| `cred_free` | `void` | — | تحرير cred security blob |
| `cred_prepare` | `int` | 0 | نسخ cred لـ fork/exec |
| `cred_transfer` | `void` | — | نقل security بين creds |
| `cred_getsecid` | `void` | — | استخراج secid من cred |
| `cred_getlsmprop` | `void` | — | استخراج LSM property من cred |
| `kernel_act_as` | `int` | 0 | السماح للـ kernel بالعمل كـ secid |
| `kernel_create_files_as` | `int` | 0 | إنشاء ملفات بـ security context آخر |
| `kernel_module_request` | `int` | 0 | التحقق من طلب تحميل module |
| `kernel_load_data` | `int` | 0 | التحقق من تحميل binary data |
| `kernel_post_load_data` | `int` | 0 | بعد تحميل binary data |
| `kernel_read_file` | `int` | 0 | التحقق من قراءة ملف للـ kernel |
| `kernel_post_read_file` | `int` | 0 | بعد قراءة ملف |
| `task_fix_setuid` | `int` | 0 | التحقق من setuid |
| `task_fix_setgid` | `int` | 0 | التحقق من setgid |
| `task_fix_setgroups` | `int` | 0 | التحقق من setgroups |
| `task_setpgid` | `int` | 0 | التحقق من setpgid |
| `task_getpgid` | `int` | 0 | التحقق من getpgid |
| `task_getsid` | `int` | 0 | التحقق من getsid |
| `current_getlsmprop_subj` | `void` | — | LSM property للـ current subject |
| `task_getlsmprop_obj` | `void` | — | LSM property لـ task كـ object |
| `task_setnice` | `int` | 0 | التحقق من setnice |
| `task_setioprio` | `int` | 0 | التحقق من setioprio |
| `task_getioprio` | `int` | 0 | التحقق من getioprio |
| `task_prlimit` | `int` | 0 | التحقق من prlimit |
| `task_setrlimit` | `int` | 0 | التحقق من setrlimit |
| `task_setscheduler` | `int` | 0 | التحقق من تعديل scheduling |
| `task_getscheduler` | `int` | 0 | التحقق من قراءة scheduling |
| `task_movememory` | `int` | 0 | التحقق من move_pages |
| `task_kill` | `int` | 0 | التحقق من إرسال signal |
| `task_prctl` | `int` | -ENOSYS | التحقق من prctl |
| `task_to_inode` | `void` | — | ربط task بـ /proc inode |
| `userns_create` | `int` | 0 | التحقق من إنشاء user namespace |

#### IPC Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `ipc_permission` | `int` | 0 | التحقق من IPC permissions |
| `ipc_getlsmprop` | `void` | — | استخراج LSM property من IPC object |
| `msg_msg_alloc_security` | `int` | 0 | تخصيص security لـ message |
| `msg_msg_free_security` | `void` | — | تحرير security لـ message |
| `msg_queue_alloc_security` | `int` | 0 | تخصيص security لـ message queue |
| `msg_queue_free_security` | `void` | — | تحرير security لـ message queue |
| `msg_queue_associate` | `int` | 0 | ربط message queue |
| `msg_queue_msgctl` | `int` | 0 | التحقق من msgctl |
| `msg_queue_msgsnd` | `int` | 0 | التحقق من msgsnd |
| `msg_queue_msgrcv` | `int` | 0 | التحقق من msgrcv |
| `shm_alloc_security` | `int` | 0 | تخصيص security لـ shared memory |
| `shm_free_security` | `void` | — | تحرير security لـ shared memory |
| `shm_associate` | `int` | 0 | ربط shared memory segment |
| `shm_shmctl` | `int` | 0 | التحقق من shmctl |
| `shm_shmat` | `int` | 0 | التحقق من shmat |
| `sem_alloc_security` | `int` | 0 | تخصيص security لـ semaphore |
| `sem_free_security` | `void` | — | تحرير security لـ semaphore |
| `sem_associate` | `int` | 0 | ربط semaphore set |
| `sem_semctl` | `int` | 0 | التحقق من semctl |
| `sem_semop` | `int` | 0 | التحقق من semop |

#### Netlink & Misc Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `netlink_send` | `int` | 0 | التحقق من إرسال netlink message |
| `d_instantiate` | `void` | — | بعد ربط dentry بـ inode |

#### Security Context Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `getselfattr` | `int` | -EOPNOTSUPP | قراءة attributes للـ current task |
| `setselfattr` | `int` | -EOPNOTSUPP | كتابة attributes للـ current task |
| `getprocattr` | `int` | -EINVAL | قراءة /proc/*/attr |
| `setprocattr` | `int` | -EINVAL | كتابة /proc/*/attr |
| `ismaclabel` | `int` | 0 | هل الاسم ده MAC label |
| `secid_to_secctx` | `int` | -EOPNOTSUPP | تحويل secid لـ string |
| `lsmprop_to_secctx` | `int` | -EOPNOTSUPP | تحويل lsm_prop لـ context |
| `secctx_to_secid` | `int` | 0 | تحويل string لـ secid |
| `release_secctx` | `void` | — | تحرير security context |
| `inode_invalidate_secctx` | `void` | — | إلغاء صلاحية cached context |
| `inode_notifysecctx` | `int` | 0 | تحديث security context لـ inode |
| `inode_setsecctx` | `int` | 0 | كتابة security context على dentry |
| `inode_getsecctx` | `int` | -EOPNOTSUPP | قراءة security context من inode |

#### Network Hooks (CONFIG_SECURITY_NETWORK)

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `unix_stream_connect` | `int` | 0 | التحقق من Unix stream connect |
| `unix_may_send` | `int` | 0 | التحقق من Unix datagram send |
| `socket_create` | `int` | 0 | التحقق من socket() syscall |
| `socket_post_create` | `int` | 0 | بعد إنشاء socket |
| `socket_socketpair` | `int` | 0 | التحقق من socketpair() |
| `socket_bind` | `int` | 0 | التحقق من bind() |
| `socket_connect` | `int` | 0 | التحقق من connect() |
| `socket_listen` | `int` | 0 | التحقق من listen() |
| `socket_accept` | `int` | 0 | التحقق من accept() |
| `socket_sendmsg` | `int` | 0 | التحقق من sendmsg() |
| `socket_recvmsg` | `int` | 0 | التحقق من recvmsg() |
| `socket_getsockname` | `int` | 0 | التحقق من getsockname() |
| `socket_getpeername` | `int` | 0 | التحقق من getpeername() |
| `socket_getsockopt` | `int` | 0 | التحقق من getsockopt() |
| `socket_setsockopt` | `int` | 0 | التحقق من setsockopt() |
| `socket_shutdown` | `int` | 0 | التحقق من shutdown() |
| `socket_sock_rcv_skb` | `int` | 0 | التحقق من استقبال packet |
| `socket_getpeersec_stream` | `int` | -ENOPROTOOPT | قراءة peer security context (stream) |
| `socket_getpeersec_dgram` | `int` | -ENOPROTOOPT | قراءة peer security context (dgram) |
| `sk_alloc_security` | `int` | 0 | تخصيص security blob لـ sock |
| `sk_free_security` | `void` | — | تحرير security blob |
| `sk_clone_security` | `void` | — | نسخ security blob |
| `sk_getsecid` | `void` | — | استخراج secid من sock |
| `sock_graft` | `void` | — | ربط sock بـ socket |
| `inet_conn_request` | `int` | 0 | SYN packet وصل |
| `inet_csk_clone` | `void` | — | نسخ sock بعد accept |
| `inet_conn_established` | `void` | — | connection اتأسس |
| `secmark_relabel_packet` | `int` | 0 | re-label packet |
| `secmark_refcount_inc` | `void` | — | زيادة refcount على secmark |
| `secmark_refcount_dec` | `void` | — | تقليل refcount على secmark |
| `req_classify_flow` | `void` | — | تصنيف flow لـ request_sock |
| `tun_dev_alloc_security` | `int` | 0 | تخصيص security لـ TUN device |
| `tun_dev_create` | `int` | 0 | التحقق من إنشاء TUN device |
| `tun_dev_attach_queue` | `int` | 0 | إضافة queue لـ TUN |
| `tun_dev_attach` | `int` | 0 | ربط sock بـ TUN |
| `tun_dev_open` | `int` | 0 | فتح TUN device |
| `sctp_assoc_request` | `int` | 0 | SCTP association request |
| `sctp_bind_connect` | `int` | 0 | SCTP bind/connect |
| `sctp_sk_clone` | `void` | — | نسخ SCTP sock |
| `sctp_assoc_established` | `void/int` | — | SCTP association اتأسس |
| `mptcp_add_subflow` | `int` | 0 | إضافة MPTCP subflow |

#### InfiniBand Hooks (CONFIG_SECURITY_INFINIBAND)

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `ib_pkey_access` | `int` | 0 | التحقق من وصول InfiniBand pkey |
| `ib_endport_manage_subnet` | `int` | 0 | إدارة subnet عبر endport |
| `ib_alloc_security` | `int` | 0 | تخصيص security لـ IB object |

#### XFRM/IPsec Hooks (CONFIG_SECURITY_NETWORK_XFRM)

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `xfrm_policy_alloc_security` | `int` | 0 | تخصيص security لـ XFRM policy |
| `xfrm_policy_clone_security` | `int` | 0 | نسخ XFRM policy security |
| `xfrm_policy_free_security` | `void` | — | تحرير XFRM policy security |
| `xfrm_policy_delete_security` | `int` | 0 | حذف XFRM policy |
| `xfrm_state_alloc` | `int` | 0 | تخصيص security لـ XFRM state |
| `xfrm_state_alloc_acquire` | `int` | 0 | تخصيص state من acquire event |
| `xfrm_state_free_security` | `void` | — | تحرير state security |
| `xfrm_state_delete_security` | `int` | 0 | حذف XFRM state |
| `xfrm_policy_lookup` | `int` | 0 | البحث في XFRM policies |
| `xfrm_state_pol_flow_match` | `int` | 1 | مطابقة state مع policy وflow |
| `xfrm_decode_session` | `int` | 0 | استخراج secid من packet |

#### Key Management Hooks (CONFIG_KEYS)

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `key_alloc` | `int` | 0 | تخصيص security لـ key |
| `key_permission` | `int` | 0 | التحقق من عملية على key |
| `key_getsecurity` | `int` | 0 | قراءة security string لـ key |
| `key_post_create_or_update` | `void` | — | بعد إنشاء/تحديث key |

#### Audit Hooks (CONFIG_AUDIT)

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `audit_rule_init` | `int` | 0 | تهيئة LSM audit rule |
| `audit_rule_known` | `int` | 0 | هل الـ rule تحتوي على LSM field |
| `audit_rule_match` | `int` | 0 | مطابقة event مع rule |
| `audit_rule_free` | `void` | — | تحرير audit rule |

#### BPF Hooks (CONFIG_BPF_SYSCALL)

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `bpf` | `int` | 0 | التحقق من bpf() syscall |
| `bpf_map` | `int` | 0 | وصول لـ BPF map عبر fd |
| `bpf_prog` | `int` | 0 | وصول لـ BPF program عبر fd |
| `bpf_map_create` | `int` | 0 | إنشاء BPF map |
| `bpf_map_free` | `void` | — | تحرير BPF map |
| `bpf_prog_load` | `int` | 0 | تحميل BPF program |
| `bpf_prog_free` | `void` | — | تحرير BPF program |
| `bpf_token_create` | `int` | 0 | إنشاء BPF token |
| `bpf_token_free` | `void` | — | تحرير BPF token |
| `bpf_token_cmd` | `int` | 0 | التحقق من bpf command عبر token |
| `bpf_token_capable` | `int` | 0 | التحقق من capability عبر token |

#### Misc Runtime Hooks

| Hook | Return | Default | الغرض |
|------|--------|---------|-------|
| `locked_down` | `int` | 0 | التحقق من kernel lockdown |
| `perf_event_open` | `int` | 0 | التحقق من perf_event_open |
| `perf_event_alloc` | `int` | 0 | تخصيص security لـ perf event |
| `perf_event_read` | `int` | 0 | التحقق من قراءة perf event |
| `perf_event_write` | `int` | 0 | التحقق من كتابة perf event |
| `uring_override_creds` | `int` | 0 | التحقق من تغيير creds في io_uring |
| `uring_sqpoll` | `int` | 0 | التحقق من إنشاء sqpoll thread |
| `uring_cmd` | `int` | 0 | التحقق من io_uring command |
| `uring_allowed` | `int` | 0 | هل io_uring مسموح به |
| `initramfs_populated` | `void` | — | بعد تحميل initramfs |
| `bdev_alloc_security` | `int` | 0 | تخصيص security لـ block device |
| `bdev_free_security` | `void` | — | تحرير security لـ block device |
| `bdev_setintegrity` | `int` | 0 | تعيين integrity metadata لـ block device |
| `post_notification` | `int` | 0 | بعد watch_queue notification |
| `watch_key` | `int` | 0 | التحقق من مراقبة key |

---

### Groups التفصيلية مع الشرح العميق

---

### Group 1: Binder IPC Hooks

**الـ Android Binder** هو الـ IPC الأساسي في Android. الـ LSM hooks دي بتتدخل في كل عملية تواصل بين processes.

#### `binder_set_context_mgr`

```c
LSM_HOOK(int, 0, binder_set_context_mgr, const struct cred *mgr)
```

بيتشيّل لما process بتحاول تبقى الـ Binder context manager (زي servicemanager في Android). الـ LSM بتقرر هل الـ credentials المقدّمة مؤهلة لهذا الدور. الـ default هو 0 (السماح)، اللي بيعني إن غياب LSM policy ما بيمنعش العملية.

- **`mgr`**: الـ `struct cred` للـ task اللي بيطلب يبقى context manager
- **Return**: 0 للسماح، error code للرفض
- **Caller**: `binder_ioctl()` في `drivers/android/binder.c` عند `BINDER_SET_CONTEXT_MGR`

#### `binder_transaction`

```c
LSM_HOOK(int, 0, binder_transaction, const struct cred *from,
         const struct cred *to)
```

بيتشيّل عند كل Binder transaction. بيدي الـ LSM فرصة تتحقق من إن الـ caller مسموحله يتواصل مع الـ target. ده أهم hook لـ mandatory access control في بيئات Android.

- **`from`**: credentials الـ caller
- **`to`**: credentials الـ target process
- **Return**: 0 للسماح، `-EPERM` أو غيره للرفض
- **Caller**: `binder_transaction()` في kernel binder driver

#### `binder_transfer_binder`

```c
LSM_HOOK(int, 0, binder_transfer_binder, const struct cred *from,
         const struct cred *to)
```

بيتشيّل لما process بتنقل binder object لـ process تانية. مختلف عن `binder_transaction` لأن ده نقل ملكية object مش مجرد call.

#### `binder_transfer_file`

```c
LSM_HOOK(int, 0, binder_transfer_file, const struct cred *from,
         const struct cred *to, const struct file *file)
```

بيتشيّل لما file descriptor بيتبعت عبر Binder. الـ LSM بتتحقق من إن الـ receiver مسموحله يمتلك هذا الـ `file`.

- **`file`**: الـ file descriptor اللي بيتنقل

---

### Group 2: Process Tracing Hooks

#### `ptrace_access_check`

```c
LSM_HOOK(int, 0, ptrace_access_check, struct task_struct *child,
         unsigned int mode)
```

بيتشيّل لما task بتحاول تعمل `ptrace(PTRACE_ATTACH, ...)` على process تانية. ده بعد الفحص التقليدي لـ `CAP_SYS_PTRACE`. الـ LSM بتضيف layer تانية من الفحص يعتمد على security labels.

- **`child`**: الـ process اللي هيتم trace عليها
- **`mode`**: `PTRACE_MODE_*` flags (READ, ATTACH, إلخ)
- **Return**: 0 للسماح
- **Locking**: بيتشيّل وهو ماسك `tasklist_lock` أو في RCU read-side

#### `ptrace_traceme`

```c
LSM_HOOK(int, 0, ptrace_traceme, struct task_struct *parent)
```

بيتشيّل لما child تطلب من parent يعملها trace عبر `PTRACE_TRACEME`. بيدي الـ LSM فرصة ترفض العلاقة دي حتى لو الـ task طالباها بنفسها.

---

### Group 3: Capabilities Hooks

#### `capget`

```c
LSM_HOOK(int, 0, capget, const struct task_struct *target,
         kernel_cap_t *effective, kernel_cap_t *inheritable,
         kernel_cap_t *permitted)
```

بيتشيّل عند `capget()` syscall. الـ LSM ممكن تعدّل الـ sets المُرجَعة أو ترفض العملية. الـ `commoncap` LSM بيستخدمه.

- **`target`**: الـ task اللي بنقرأ capabilities بتاعتها
- **`effective/inheritable/permitted`**: الـ output sets، ممكن الـ LSM تعدّلها

#### `capset`

```c
LSM_HOOK(int, 0, capset, struct cred *new, const struct cred *old,
         const kernel_cap_t *effective, const kernel_cap_t *inheritable,
         const kernel_cap_t *permitted)
```

بيتشيّل عند `capset()`. الـ LSM بتفحص هل الـ transition المطلوبة مسموحة. الـ `new` cred لسه ما اتطبقتش — لو رجعنا error مش هيحصل تغيير.

#### `capable`

```c
LSM_HOOK(int, 0, capable, const struct cred *cred,
         struct user_namespace *ns, int cap, unsigned int opts)
```

**ده أكتر hook بيتشيّل في الـ kernel.** كل `capable()` و`ns_capable()` call بتمر منه. الـ `commoncap` LSM بيتحقق من الـ capability sets هنا، وباقي الـ LSMs (SELinux, AppArmor) بتضيف policy checks.

```
security_capable()
   └─> call_int_hook(capable, ...)
         └─> selinux_capable()  → check AVC
         └─> apparmor_capable() → check profile
         └─> commoncap_capable() → check kernel_cap_t bits
```

- **`cred`**: الـ credentials اللي بنفحصها
- **`ns`**: الـ user namespace السياق
- **`cap`**: رقم الـ capability (مثلاً `CAP_NET_ADMIN = 12`)
- **`opts`**: `CAP_OPT_NOAUDIT` لتخطي audit، `CAP_OPT_INSETID` لو في setid context
- **Return**: 0 = مسموح، `-EPERM` = مرفوض

---

### Group 4: Binary Execution Hooks

الـ hooks دي بتتشيّل أثناء `execve()` في الترتيب التالي:

```
execve()
  → bprm_creds_for_exec()      ← تهيئة credentials للـ exec
  → bprm_creds_from_file()     ← تعديل credentials من الـ file (suid/sgid)
  → bprm_check_security()      ← فحص أمني نهائي
  → ... kernel prepares mm ...
  → bprm_committing_creds()    ← لحظة تطبيق credentials (لا رجعة)
  → bprm_committed_creds()     ← بعد التطبيق الكامل
```

#### `bprm_creds_for_exec`

```c
LSM_HOOK(int, 0, bprm_creds_for_exec, struct linux_binprm *bprm)
```

الـ hook الأول في exec pipeline. الـ LSM بتحضّر `bprm->cred` بالـ security context المناسب للـ process الجديدة. في SELinux بيحدد هنا الـ domain transition.

- **`bprm`**: الـ `linux_binprm` struct اللي فيه معلومات عن الـ binary
- **Key detail**: `bprm->cred` هو نسخة من الـ current cred، الـ LSM تعدّل عليه هنا

#### `bprm_creds_from_file`

```c
LSM_HOOK(int, 0, bprm_creds_from_file, struct linux_binprm *bprm,
         const struct file *file)
```

بيتشيّل لكل interpreter في الـ exec chain (للـ scripts مثلاً). بيدي الـ LSM فرصة تعدّل credentials بناءً على الـ file المفتوح فعلاً.

#### `bprm_check_security`

```c
LSM_HOOK(int, 0, bprm_check_security, struct linux_binprm *bprm)
```

فحص أمني أخير بعد ما kernel خلّص تحضير الـ `bprm`. لو رجع error هنا الـ exec بيتوقف قبل ما يحصل أي تعديل في الـ process state.

#### `bprm_committing_creds`

```c
LSM_HOOK(void, LSM_RET_VOID, bprm_committing_creds,
         const struct linux_binprm *bprm)
```

بيتشيّل لحظة قبل تطبيق credentials الجديدة. الـ LSM بتعمل هنا أي setup نهائي. مش ممكن ترجع error هنا.

#### `bprm_committed_creds`

```c
LSM_HOOK(void, LSM_RET_VOID, bprm_committed_creds,
         const struct linux_binprm *bprm)
```

بعد ما credentials اتطبقت فعلاً. الـ LSM بتعمل cleanup أو notification. SELinux بيبعث هنا netlink message عن domain transition.

---

### Group 5: Superblock & Filesystem Hooks

#### `sb_alloc_security`

```c
LSM_HOOK(int, 0, sb_alloc_security, struct super_block *sb)
```

بيتشيّل عند إنشاء كل `super_block`. الـ LSM بتخصص هنا security blob في `sb->s_security`. لو فشل بيرجع error وبيمنع mount.

- **Side effect**: بيخصص memory، لازم `sb_free_security` يتشيّل عشان تتحرر

#### `sb_eat_lsm_opts`

```c
LSM_HOOK(int, 0, sb_eat_lsm_opts, char *orig, void **mnt_opts)
```

بيشيّل LSM-specific options من الـ mount option string. مثلاً SELinux بيشيّل `context=`, `fscontext=` إلخ من الـ mount options قبل ما kernel يشوف الـ options التانية.

- **`orig`**: الـ option string (بيتعدّل in-place)
- **`mnt_opts`**: pointer لـ opaque struct بتحتفظ بيه الـ LSM

#### `sb_set_mnt_opts`

```c
LSM_HOOK(int, 0, sb_set_mnt_opts, struct super_block *sb,
         void *mnt_opts, unsigned long kern_flags,
         unsigned long *set_kern_flags)
```

بيطبّق الـ security mount options على الـ superblock. بيتشيّل بعد `sb_alloc_security` وبعد ما الـ options اتحللت. SELinux بيحدد هنا الـ context الخاص بالـ filesystem.

#### `sb_clone_mnt_opts`

```c
LSM_HOOK(int, 0, sb_clone_mnt_opts, const struct super_block *oldsb,
         struct super_block *newsb, unsigned long kern_flags,
         unsigned long *set_kern_flags)
```

بيتشيّل عند bind mount أو remount. بينسخ الـ security context من الـ `oldsb` للـ `newsb`. لازم يتحقق من التوافق.

#### `sb_mount`

```c
LSM_HOOK(int, 0, sb_mount, const char *dev_name, const struct path *path,
         const char *type, unsigned long flags, void *data)
```

فحص أمني قبل `mount()` syscall. بيتشيّل مبكراً في `do_mount()`. الـ SELinux بيتحقق هنا من إن الـ process عندها `mount_t` permission في policy بتاعتها.

---

### Group 6: Inode Hooks — الأهم

#### `inode_permission`

```c
LSM_HOOK(int, 0, inode_permission, struct inode *inode, int mask)
```

**ده أكتر inode hook بيتشيّل** — بيتشيّل في كل path lookup، كل `access()` syscall، كل open. الـ DAC بيكون اتعمل قبله. الـ LSM بيعمل MAC check هنا.

```
inode_permission()
  → generic_permission()  ← DAC check
  → security_inode_permission()
       └─> selinux_inode_permission() → avc_has_perm()
       └─> apparmor_inode_permission() → profile match
```

- **`mask`**: combination of `MAY_READ | MAY_WRITE | MAY_EXEC | MAY_APPEND`
- **Performance**: بيتشيّل بشكل متكرر جداً — الـ SELinux بيعتمد على AVC cache لتجنب overhead

#### `inode_init_security`

```c
LSM_HOOK(int, -EOPNOTSUPP, inode_init_security, struct inode *inode,
         struct inode *dir, const struct qstr *qstr,
         struct xattr *xattrs, int *xattr_count)
```

بيتشيّل عند إنشاء inode جديد. الـ LSM بترجع هنا الـ xattrs اللي لازم تتكتب على الـ inode الجديد (زي `security.selinux`). الـ default هو `-EOPNOTSUPP` يعني لو مافيش LSM شايل الـ hook يبقى مش لازم xattr.

- **`xattrs`**: array الـ LSM بتملي فيها الـ xattr slots
- **`xattr_count`**: عدد الـ slots المملوءة
- **لازم**: الـ LSM تستخدم `lsm_get_xattr_slot()` من `lsm_hooks.h` عشان تملي الـ array بشكل صح مع stacking

#### `inode_setxattr`

```c
LSM_HOOK(int, 0, inode_setxattr, struct mnt_idmap *idmap,
         struct dentry *dentry, const char *name, const void *value,
         size_t size, int flags)
```

بيتشيّل قبل كتابة أي xattr. الـ LSM بتفحص هل الـ process مسموحلها تكتب على `security.*` namespace. SELinux بيمنع كتابة `security.selinux` مباشرة إلا لو عندك `relabelfrom/relabelto` permission.

- **`idmap`**: الـ mount idmap للـ uid/gid mapping
- **`name`**: اسم الـ xattr (مثلاً `"security.selinux"`)

#### `inode_setattr`

```c
LSM_HOOK(int, 0, inode_setattr, struct mnt_idmap *idmap,
         struct dentry *dentry, struct iattr *attr)
```

بيتشيّل قبل تعديل inode attributes (size, timestamps, uid, gid, mode). الـ LSM بتفحص كل نوع تغيير في `attr->ia_valid`.

#### `inode_getsecurity` / `inode_setsecurity`

```c
LSM_HOOK(int, -EOPNOTSUPP, inode_getsecurity, struct mnt_idmap *idmap,
         struct inode *inode, const char *name, void **buffer, bool alloc)

LSM_HOOK(int, -EOPNOTSUPP, inode_setsecurity, struct inode *inode,
         const char *name, const void *value, size_t size, int flags)
```

الـ functions دول للتعامل المباشر مع security attributes (مش عبر xattr interface). الـ default `-EOPNOTSUPP` معناه لو مافيش LSM بيعملها الـ VFS بيتصرف accordingly.

- **`alloc`**: في `getsecurity`، لو `true` الـ LSM بتخصص buffer وتحط pointer فيه

#### `inode_copy_up`

```c
LSM_HOOK(int, 0, inode_copy_up, struct dentry *src, struct cred **new)
```

خاص بـ overlayfs. لما ملف بيتعدّل في الـ upper layer، الـ LSM محتاجة تضبط الـ credentials المستخدمة في الـ copy-up operation.

#### `inode_setintegrity`

```c
LSM_HOOK(int, 0, inode_setintegrity, const struct inode *inode,
         enum lsm_integrity_type type, const void *value, size_t size)
```

بيتشيّل لما IMA أو dm-verity بتقول إن ملف اتتحقق منه. الـ LSM بتحفظ الـ integrity metadata جنب الـ security labels. الـ `enum lsm_integrity_type` بتشمل:
- `LSM_INT_DMVERITY_SIG_VALID`
- `LSM_INT_DMVERITY_ROOTHASH`
- `LSM_INT_FSVERITY_BUILTINSIG_VALID`

---

### Group 7: File Hooks

#### `file_permission`

```c
LSM_HOOK(int, 0, file_permission, struct file *file, int mask)
```

مختلف عن `inode_permission` — ده بيتشيّل على **file مفتوح فعلاً**. بيُعمل عند `read()` و`write()` كل مرة. الـ SELinux بيستخدمه للـ `file__read` و`file__write` AVC checks.

#### `file_open`

```c
LSM_HOOK(int, 0, file_open, struct file *file)
```

بيتشيّل مرة واحدة عند `open()`. ده نقطة مهمة لأن الـ LSM ممكن تتحقق من الـ O_RDONLY/O_RDWR/O_WRONLY flags هنا وتحفظ القرار.

#### `mmap_file`

```c
LSM_HOOK(int, 0, mmap_file, struct file *file, unsigned long reqprot,
         unsigned long prot, unsigned long flags)
```

بيتشيّل عند `mmap()`. الـ LSM بتفحص هل الـ process مسموحلها تعمل executable mapping لـ file معين. SELinux بيستخدم `file__execute` و`file__entrypoint` permissions هنا.

- **`reqprot`**: الـ protection المطلوبة من الـ user
- **`prot`**: الـ protection الفعلية بعد kernel modification
- **`flags`**: الـ MAP_* flags

#### `file_mprotect`

```c
LSM_HOOK(int, 0, file_mprotect, struct vm_area_struct *vma,
         unsigned long reqprot, unsigned long prot)
```

بيتشيّل عند `mprotect()`. الـ LSM ممكن تمنع تحويل mapping من non-exec لـ exec (W^X enforcement).

#### `mmap_addr`

```c
LSM_HOOK(int, 0, mmap_addr, unsigned long addr)
```

بيفحص هل مسموح بعمل mmap على عنوان معين. بيُستخدم لمنع mapping على عناوين منخفضة (null-dereference exploitation). الـ `yama` LSM بيستخدمه.

#### `file_receive`

```c
LSM_HOOK(int, 0, file_receive, struct file *file)
```

بيتشيّل لما file descriptor بيتبعت عبر Unix domain socket (SCM_RIGHTS). الـ LSM بتتحقق من إن الـ receiver مسموحله يمتلك الـ file ده.

---

### Group 8: Task & Credentials Hooks

#### `cred_prepare`

```c
LSM_HOOK(int, 0, cred_prepare, struct cred *new, const struct cred *old,
         gfp_t gfp)
```

بيتشيّل لما kernel بتعمل `prepare_creds()` — في كل `fork()` وقبل كل `exec()`. الـ LSM بتنسخ الـ security blob من الـ `old` cred للـ `new`.

```c
/* مثال SELinux */
static int selinux_cred_prepare(struct cred *new, const struct cred *old, gfp_t gfp)
{
    /* copy the security blob from old to new */
    struct task_security_struct *tsec = selinux_cred(new);
    *tsec = *selinux_cred(old);
    return 0;
}
```

#### `kernel_load_data` / `kernel_post_load_data`

```c
LSM_HOOK(int, 0, kernel_load_data, enum kernel_load_data_id id, bool contents)
LSM_HOOK(int, 0, kernel_post_load_data, char *buf, loff_t size,
         enum kernel_load_data_id id, char *description)
```

بيتشيّلوا لما kernel بتحمّل kernel modules أو firmware أو initrd بدون قراءة من file مباشرة (مثلاً من memory). الـ IMA بيستخدمهم للـ measurement.

- **`id`**: `LOADING_MODULE`, `LOADING_FIRMWARE`, `LOADING_KEXEC_IMAGE`, إلخ
- **`contents`**: هل الـ buffer فيه الـ content الفعلي

#### `task_kill`

```c
LSM_HOOK(int, 0, task_kill, struct task_struct *p,
         struct kernel_siginfo *info, int sig, const struct cred *cred)
```

بيتشيّل قبل إرسال أي signal. الـ LSM بتتحقق من إن الـ sender مسموحله يبعت signal للـ target. SELinux بيطبق `process__signal` و`process__sigkill` permissions.

- **`cred`**: credentials المُرسِل (ممكن تختلف عن current في بعض cases)

#### `task_prctl`

```c
LSM_HOOK(int, -ENOSYS, task_prctl, int option, unsigned long arg2,
         unsigned long arg3, unsigned long arg4, unsigned long arg5)
```

**ده الـ hook الوحيد اللي default بتاعته `-ENOSYS` مش `0`.** معناه لو مافيش LSM مسجّل للـ hook ده `prctl()` بيرجع `-ENOSYS` للـ option ده. الـ LSMs بتستخدمه لإضافة custom prctl options.

#### `userns_create`

```c
LSM_HOOK(int, 0, userns_create, const struct cred *cred)
```

بيتشيّل عند إنشاء user namespace جديدة. الـ LSM ممكن تمنع unprivileged user namespace creation — ده مهم جداً لتقليل attack surface في containers. الـ Yama LSM و AppArmor بيستخدموه.

---

### Group 9: Security Context Hooks

#### `getselfattr` / `setselfattr`

```c
LSM_HOOK(int, -EOPNOTSUPP, getselfattr, unsigned int attr,
         struct lsm_ctx __user *ctx, u32 *size, u32 flags)
LSM_HOOK(int, -EOPNOTSUPP, setselfattr, unsigned int attr,
         struct lsm_ctx *ctx, u32 size, u32 flags)
```

الـ modern API اللي جه مع LSM stacking. بيسمح بقراءة/كتابة attributes من/لـ LSMs متعددة في call واحد. الـ `struct lsm_ctx` بيحتوي على LSM ID مع الـ data.

```c
struct lsm_ctx {
    __u64 id;      /* LSM ID */
    __u64 flags;
    __u64 len;     /* total length */
    __u64 ctx_len; /* length of ctx data */
    __u8  ctx[];   /* variable-length context */
};
```

#### `secid_to_secctx` / `lsmprop_to_secctx`

```c
LSM_HOOK(int, -EOPNOTSUPP, secid_to_secctx, u32 secid, struct lsm_context *cp)
LSM_HOOK(int, -EOPNOTSUPP, lsmprop_to_secctx, struct lsm_prop *prop,
         struct lsm_context *cp)
```

بيحوّلوا الـ internal representation (secid أو lsm_prop) لـ human-readable string. الـ `lsmprop_to_secctx` هو الـ newer version للـ stacked LSMs.

- **`cp`**: الـ output، `struct lsm_context` بيتضمن الـ string والـ length
- **`release_secctx`**: لازم يتشيّل بعد كده عشان يحرر الـ allocated string

#### `inode_invalidate_secctx`

```c
LSM_HOOK(void, LSM_RET_VOID, inode_invalidate_secctx, struct inode *inode)
```

بيتشيّل لما security context للـ inode بيتغير (زي `chcon` في SELinux). الـ LSM بتعمل cache invalidation.

---

### Group 10: Network Hooks

#### `socket_sock_rcv_skb`

```c
LSM_HOOK(int, 0, socket_sock_rcv_skb, struct sock *sk, struct sk_buff *skb)
```

بيتشيّل على كل **incoming packet** قبل ما يوصل للـ socket buffer. ده في الـ hot path للـ networking. الـ SELinux بيستخدم netfilter marks للتحقق من الـ packet security label بدل ما يعمل check هنا.

#### `inet_conn_request`

```c
LSM_HOOK(int, 0, inet_conn_request, const struct sock *sk,
         struct sk_buff *skb, struct request_sock *req)
```

بيتشيّل لما SYN packet بيوصل للـ listening socket. الـ LSM بتقرر هل تقبل الـ connection request. لو رجع error الـ SYN بيتتجاهل.

#### `xfrm_state_pol_flow_match`

```c
LSM_HOOK(int, 1, xfrm_state_pol_flow_match, struct xfrm_state *x,
         struct xfrm_policy *xp, const struct flowi_common *flic)
```

**ده الوحيد اللي default بتاعته `1` مش `0` في غير `task_prctl`.** لأن `1` = match، بيعني لو مافيش LSM يتحقق بيبقى كل حاجة تعتبر match (permissive default).

#### `sctp_assoc_request` / `sctp_assoc_established`

```c
LSM_HOOK(int, 0, sctp_assoc_request, struct sctp_association *asoc,
         struct sk_buff *skb)
LSM_HOOK(int, 0, sctp_assoc_established, struct sctp_association *asoc,
         struct sk_buff *skb)
```

الـ hooks دي للـ SCTP protocol — بيتشيّلوا عند بداية ونهاية association establishment. مهمة للـ environments اللي بتستخدم SCTP (telecom, etc.).

---

### Group 11: BPF Hooks

#### `bpf` / `bpf_prog_load` / `bpf_map_create`

```c
LSM_HOOK(int, 0, bpf, int cmd, union bpf_attr *attr,
         unsigned int size, bool kernel)
LSM_HOOK(int, 0, bpf_prog_load, struct bpf_prog *prog,
         union bpf_attr *attr, struct bpf_token *token, bool kernel)
LSM_HOOK(int, 0, bpf_map_create, struct bpf_map *map,
         union bpf_attr *attr, struct bpf_token *token, bool kernel)
```

الـ BPF hooks جاءت ضرورة لأن BPF بقى أداة قوية جداً تحتاج access control. الـ `bpf_token` هو آلية delegation للـ capabilities — بيسمح لـ unprivileged process تستخدم BPF capabilities معينة بدون `CAP_BPF`.

- **`kernel`**: هل الـ call جاي من kernel نفسه (تجاهل بعض الـ checks)
- **`token`**: لو `NULL` يعني مفيش delegation

#### `bpf_token_capable`

```c
LSM_HOOK(int, 0, bpf_token_capable, const struct bpf_token *token, int cap)
```

بيفحص هل الـ token بيدي الـ capability المطلوبة. ده extension لـ capability model للـ BPF subsystem.

---

### Group 12: Audit Hooks

#### `audit_rule_init`

```c
LSM_HOOK(int, 0, audit_rule_init, u32 field, u32 op, char *rulestr,
         void **lsmrule, gfp_t gfp)
```

بيتشيّل لما audit rule بتتحدد وفيها LSM field (زي `subj_user`, `subj_role` في SELinux). الـ LSM بتحوّل الـ `rulestr` لـ internal representation وتحطه في `*lsmrule`.

#### `audit_rule_match`

```c
LSM_HOOK(int, 0, audit_rule_match, struct lsm_prop *prop, u32 field,
         u32 op, void *lsmrule)
```

بيتشيّل لكل audit event عشان يشوف هل يتطابق مع الـ rule. لو match بيرجع `1`، لو لا بيرجع `0`.

---

### Group 13: Block Device Hooks

#### `bdev_alloc_security` / `bdev_setintegrity`

```c
LSM_HOOK(int, 0, bdev_alloc_security, struct block_device *bdev)
LSM_HOOK(int, 0, bdev_setintegrity, struct block_device *bdev,
         enum lsm_integrity_type type, const void *value, size_t size)
```

الـ `bdev_setintegrity` بيتشيّل لما dm-verity بتتحقق من block device وبتقدّم الـ roothash. الـ LSM بتحفظ الـ integrity status جنب الـ block device security context. مهم جداً في verified boot scenarios.

---

### Group 14: io_uring Hooks

```c
LSM_HOOK(int, 0, uring_override_creds, const struct cred *new)
LSM_HOOK(int, 0, uring_sqpoll, void)
LSM_HOOK(int, 0, uring_cmd, struct io_uring_cmd *ioucmd)
LSM_HOOK(int, 0, uring_allowed, void)
```

الـ io_uring hooks جاءت بعد ما اكتُشف إن io_uring ممكن يُستخدم لـ privilege escalation لأنه كان بيتجاوز بعض LSM checks. الـ `uring_allowed` بتسمح للـ LSM بمنع io_uring كلياً.

---

### آلية الـ Macro وكيف تُجمَع الـ Hooks

```c
/* الـ LSM_HOOK macro بيتعرّف بأشكال مختلفة حسب السياق */

/* في union security_list_options: */
#define LSM_HOOK(RET, DEFAULT, NAME, ...) RET (*NAME)(__VA_ARGS__);

/* في struct lsm_static_calls_table: */
#define LSM_HOOK(RET, DEFAULT, NAME, ...) \
    struct lsm_static_call NAME[MAX_LSM_COUNT];

/* في struct security_hook_heads (legacy): */
#define LSM_HOOK(RET, DEFAULT, NAME, ...) struct hlist_head NAME;
```

#### Static Call Mechanism

الـ kernel بيستخدم **static calls** بدل الـ function pointers لأداء أفضل:

```
LSM registration (security_add_hooks)
  → copies hook callbacks into lsm_static_calls_table
  → fills from last slot to first (backwards)
  → enables static_key for each active slot

Hook invocation (security_NAME)
  → call_int_hook() macro
  → iterates only active slots (via static_key)
  → aggregates return values
```

الـ static keys بتعني إن الـ CPU branch predictor دايماً يعرف أيه الـ hooks المفعّلة بدون memory indirection.

#### الـ `call_int_hook` Flow

```c
/* Pseudocode for how int hooks are called */
#define call_int_hook(HOOK, ...) ({                    \
    int rc = 0;                                         \
    /* for each active static call slot for HOOK: */   \
    /*   rc = slot->func(__VA_ARGS__);               */ \
    /*   if (rc != 0) break; (first failure wins)   */ \
    rc;                                                 \
})
```

الـ `void` hooks بتستخدم `call_void_hook` وهي نفس الفكرة بدون return value aggregation.

---

### Default Values وما بتعنيه

| Default | الـ Hooks اللي بتستخدمه | المعنى |
|---------|------------------------|--------|
| `0` | معظم الـ hooks | السماح لما مافيش LSM policy |
| `LSM_RET_VOID` | جميع الـ void hooks | لا يوجد return value |
| `-EOPNOTSUPP` | `inode_init_security`, `getselfattr`, `secid_to_secctx`, إلخ | الـ feature مش supported بدون LSM |
| `-ENOPARAM` | `fs_context_parse_param` | الـ parameter مش LSM parameter |
| `-EINVAL` | `getprocattr`, `setprocattr` | error لو مافيش LSM بيعرّف الـ attr |
| `-ENOSYS` | `task_prctl` | الـ option مش معروف |
| `1` | `xfrm_state_pol_flow_match` | match by default (permissive) |
| `-ENOPROTOOPT` | socket peer security hooks | الـ option مش supported |
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بالـ LSM

الـ **LSM framework** بيكشف معلومات مفيدة جداً تحت `/sys/kernel/security/` (مش `debugfs` بالمعنى الضيق، لكن هي النقطة الأساسية للـ LSM).

| المسار | المحتوى | طريقة القراءة |
|--------|---------|--------------|
| `/sys/kernel/security/lsm` | قائمة الـ LSMs الفعّالة مرتبة | `cat /sys/kernel/security/lsm` |
| `/sys/kernel/security/selinux/enforce` | SELinux: 0=permissive, 1=enforcing | `cat /sys/kernel/security/selinux/enforce` |
| `/sys/kernel/security/selinux/policy_capabilities/` | capabilities مفعّلة في الـ policy | `ls /sys/kernel/security/selinux/policy_capabilities/` |
| `/sys/kernel/security/apparmor/profiles` | قائمة الـ AppArmor profiles المحملة | `cat /sys/kernel/security/apparmor/profiles` |
| `/sys/kernel/security/apparmor/.access` | interface لاختبار الـ access | `echo "file /etc/passwd r" > /sys/kernel/security/apparmor/.access` |
| `/proc/self/attr/current` | security label للـ process الحالي | `cat /proc/self/attr/current` |
| `/proc/<pid>/attr/current` | security label لأي process | `cat /proc/1234/attr/current` |
| `/proc/<pid>/attr/exec` | label لحظة الـ exec | `cat /proc/1234/attr/exec` |

```bash
# اعرف إيه الـ LSMs اللي شغّالة دلوقتي
cat /sys/kernel/security/lsm

# اقرأ security context لأي process
cat /proc/$$/attr/current

# للـ SELinux: اعرف الـ context الحالي
cat /proc/self/attr/current
id -Z   # لو selinuxfs متاحة
```

---

#### 2. مدخلات الـ sysfs المهمة

| المسار | الوصف |
|--------|-------|
| `/sys/kernel/security/` | الـ root لكل الـ LSM interfaces |
| `/sys/fs/selinux/` | SELinux-specific controls |
| `/sys/kernel/security/apparmor/` | AppArmor controls |
| `/proc/sys/kernel/perf_event_paranoid` | يتحكم في `perf_event_open` hook |
| `/proc/sys/kernel/kptr_restrict` | يؤثر على visibility الـ kernel pointers |
| `/proc/sys/kernel/dmesg_restrict` | يتحكم في `syslog` hook |
| `/proc/sys/net/core/bpf_jit_enable` | يؤثر على `bpf` hooks |
| `/proc/sys/kernel/unprivileged_bpf_disabled` | يتحكم في BPF access بدون privilege |
| `/proc/sys/user/max_user_namespaces` | يتحكم في `userns_create` hook |

```bash
# اعرف قيمة الـ dmesg_restrict (syslog hook)
cat /proc/sys/kernel/dmesg_restrict

# اعرف الـ BPF restrictions
cat /proc/sys/kernel/unprivileged_bpf_disabled

# اعرف user namespace limit
cat /proc/sys/user/max_user_namespaces
```

---

#### 3. استخدام الـ ftrace مع الـ LSM hooks

الـ **ftrace** هو الأداة الأقوى لتتبع الـ LSM hooks في real-time.

##### تفعيل الـ tracepoints الخاصة بالـ LSM:

```bash
# اتأكد إن debugfs متاحة
mount | grep debugfs
# لو مش موجود:
mount -t debugfs none /sys/kernel/debug

# اعرف الـ LSM-related events المتاحة
ls /sys/kernel/debug/tracing/events/ | grep -i lsm
ls /sys/kernel/debug/tracing/events/ | grep -i security

# تتبع security events الشبكة
echo 1 > /sys/kernel/debug/tracing/events/sock/security_socket_connect/enable

# تتبع كل الـ syscalls اللي ممكن تلاقيها في الـ LSM hooks
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_security/enable
```

##### استخدام الـ function tracer لتتبع دوال الـ LSM:

```bash
# اعرف الدوال المتاحة في الـ LSM framework
grep -i "security_" /sys/kernel/debug/tracing/available_filter_functions | head -30

# trace دالة security_inode_permission (من inode_permission hook)
echo 'security_inode_permission' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'function' > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe

# trace كل دوال الـ security subsystem
echo 'security_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'function_graph' > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# لما تخلص
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo '' > /sys/kernel/debug/tracing/set_ftrace_filter
```

##### trace hook معين (مثال: binder hooks):

```bash
# trace binder security hooks
echo 'security_binder_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'function' > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل التطبيق اللي بيستخدم binder
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

---

#### 4. الـ printk والـ dynamic debug

##### تفعيل الـ dynamic debug للـ LSM:

```bash
# تفعيل كل debug messages في الـ security subsystem
echo 'module selinux +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module apparmor +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug messages لملف معين
echo 'file security/security.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file security/selinux/hooks.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file security/apparmor/lsm.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل الـ LSM-related debug
echo 'file include/linux/lsm_hooks.h +p' > /sys/kernel/debug/dynamic_debug/control

# اعرض الـ messages
dmesg -w | grep -i 'security\|lsm\|selinux\|apparmor'

# أو حدد loglevel
dmesg --level=debug,info | grep -i security
```

##### printk levels مفيدة للـ LSM:

```c
/* لو بتكتب LSM خاص بيك */
pr_debug("LSM: hook %s called for inode %lu\n",
         __func__, inode->i_ino);

/* لـ permission denials */
pr_info("LSM %s: permission denied for pid %d\n",
        lsm_name, current->pid);
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| الـ Config | الوظيفة |
|-----------|---------|
| `CONFIG_SECURITY` | تفعيل الـ LSM framework نفسه |
| `CONFIG_SECURITY_SELINUX` | SELinux |
| `CONFIG_SECURITY_APPARMOR` | AppArmor |
| `CONFIG_SECURITY_SMACK` | Smack LSM |
| `CONFIG_SECURITY_TOMOYO` | TOMOYO LSM |
| `CONFIG_SECURITY_LANDLOCK` | Landlock |
| `CONFIG_SECURITY_PATH` | يفعّل path-based hooks |
| `CONFIG_SECURITY_NETWORK` | يفعّل network hooks |
| `CONFIG_SECURITY_NETWORK_XFRM` | XFRM/IPsec hooks |
| `CONFIG_SECURITY_INFINIBAND` | InfiniBand hooks |
| `CONFIG_DEBUG_LIST` | يكشف corruption في الـ hook lists |
| `CONFIG_KALLSYMS` | لازمة لـ ftrace ولقراءة الـ stack traces |
| `CONFIG_FTRACE` | الـ function tracer |
| `CONFIG_KPROBES` | لتسجيل probes على الـ hooks |
| `CONFIG_BPF_EVENTS` | بيخلي BPF tracing يشتغل |
| `CONFIG_LSM_MMAP_MIN_ADDR` | حماية الـ mmap للـ low addresses |
| `CONFIG_AUDIT` | يفعّل audit hooks |
| `CONFIG_AUDITSYSCALL` | audit على مستوى الـ syscall |
| `CONFIG_HAVE_ARCH_SECCOMP_FILTER` | seccomp integration |
| `CONFIG_INTEGRITY` | integrity subsystem |
| `CONFIG_IMA` | Integrity Measurement Architecture |
| `CONFIG_EVM` | Extended Verification Module |
| `CONFIG_KEYS` | key management hooks |
| `CONFIG_KEY_NOTIFICATIONS` | watch_key hook |

```bash
# اعرف إيه الـ configs المفعّلة في الـ kernel الحالي
grep -E 'CONFIG_SECURITY|CONFIG_LSM|CONFIG_AUDIT' /boot/config-$(uname -r)
# أو
zcat /proc/config.gz | grep -E 'CONFIG_SECURITY|CONFIG_LSM'
```

---

#### 6. أدوات الـ LSM الخاصة (devlink وغيرها)

##### أدوات SELinux:

```bash
# اعرف الـ policy version والـ status
sestatus -v

# اعرف الـ AVC denials (Access Vector Cache)
ausearch -m AVC,USER_AVC -ts recent
# أو
grep 'avc: denied' /var/log/audit/audit.log | tail -20

# تحويل الـ denial لـ policy module
audit2allow -a -M mymodule
semodule -i mymodule.pp

# اختبار context معين
runcon -l s0 -t httpd_t ls /var/www/

# اعرض contexts
ls -Z /etc/passwd
ps -Z -p 1234
```

##### أدوات AppArmor:

```bash
# اعرض الـ profiles والـ status
aa-status

# ضبط profile على complain mode للـ debugging
aa-complain /usr/sbin/nginx

# راقب الـ denials
aa-notify -s 1 -v

# اعرض الـ denials من الـ log
grep 'apparmor="DENIED"' /var/log/syslog
```

##### أدوات Landlock:

```bash
# مفيش tool خاص، لكن ابحث في الـ kernel log
dmesg | grep -i landlock

# مثال على كود الـ debugging
# strace على process يستخدم landlock
strace -e trace=landlock_create_ruleset,landlock_add_rule,landlock_restrict_self ./myapp
```

##### أدوات الـ audit subsystem (مرتبط بـ audit hooks):

```bash
# اعرض الـ audit rules
auditctl -l

# ضيف rule لتتبع file access
auditctl -w /etc/passwd -p rwxa -k passwd_access

# راقب الـ LSM-related audit events
ausearch -k passwd_access
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|-------|------|
| `avc: denied { read } for pid=... comm="..." path="..."` | SELinux رفض الـ read access | افتح audit.log، استخدم `audit2allow` لإنشاء policy |
| `apparmor="DENIED" operation="open" profile="..."` | AppArmor رفض الـ operation | ضيف rule في الـ profile أو استخدم `aa-complain` للتشخيص |
| `smack: SMACK access denied: ...` | Smack رفض الـ access | تحقق من الـ labels باستخدام `smackfs` |
| `LSM: security_capable: returning EPERM` | الـ process مش عنده الـ capability المطلوبة | تحقق من الـ capabilities بـ `capsh --print` |
| `security_inode_permission: permission denied` | hook رفض الـ inode access | افحص الـ LSM policy للملف المعني |
| `security_bprm_check: denied` | LSM رفض الـ exec | افحص الـ executable context وهل الـ policy بتسمح بيه |
| `security_socket_connect: denied` | LSM رفض الـ socket connection | افحص network policy في SELinux/AppArmor |
| `bpf: Security Violation!` | `bpf` hook رفض الـ BPF operation | تحقق من `unprivileged_bpf_disabled` وlsm policy |
| `key_alloc: security check failed` | رفض إنشاء kernel key | افحص keyring context وLSM policy |
| `tomoyo: Access denied ...` | TOMOYO رفض عملية | افحص الـ TOMOYO policy في `/sys/kernel/security/tomoyo/` |
| `integrity: ...: appraisal failed` | IMA appraisal فشل | افحص الـ xattrs وكذلك الـ IMA policy |
| `EVM: ... hmac_add_misc_data failed` | EVM failed to verify integrity | افحص الـ security.evm xattr للملف |
| `audit: type=1400 ...` | LSM audit event | افحص الـ full audit record في audit.log |
| `Warning: possible circular locking dependency` | deadlock في الـ LSM hook order | افحص الـ lockdep output وراجع الـ hook call order |

---

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

لو بتكتب LSM module أو بتعدّل في الـ security framework، دي أهم النقاط:

```c
/* 1. عند فشل التحقق من الـ hook registration */
int security_add_hooks(struct security_hook_list *hooks, int count,
                       const struct lsm_id *lsmid)
{
    WARN_ON(!lsmid);               /* لو lsm_id فاضية */
    WARN_ON(count <= 0);           /* لو count غلط */
    /* ... */
}

/* 2. في dentry_init_security لو return value غير متوقع */
static int my_lsm_dentry_init_security(struct dentry *dentry, int mode,
                                        const struct qstr *name,
                                        const char **xattr_name,
                                        struct lsm_context *cp)
{
    if (WARN_ON(!cp)) {
        dump_stack();
        return -EINVAL;
    }
    /* ... */
}

/* 3. عند inode_free_security لو الـ blob بيُحرّر أكتر من مرة */
static void my_lsm_inode_free_security(struct inode *inode)
{
    struct my_lsm_inode_sec *sec = inode->i_security;
    WARN_ON(!sec);  /* لو اتحرر قبل كده */
    /* ... */
}

/* 4. عند cred_prepare لو memory allocation فشل */
static int my_lsm_cred_prepare(struct cred *new, const struct cred *old,
                                 gfp_t gfp)
{
    struct my_blob *blob = kzalloc(sizeof(*blob), gfp);
    if (WARN_ON_ONCE(!blob))
        return -ENOMEM;
    /* ... */
}

/* 5. عند التحقق من الـ task_kill hook */
static int my_lsm_task_kill(struct task_struct *p,
                              struct kernel_siginfo *info,
                              int sig, const struct cred *cred)
{
    WARN_ON(!cred);
    if (sig == SIGKILL)
        dump_stack(); /* لتتبع من طلب الـ SIGKILL */
    /* ... */
}
```

---

### Hardware Level

---

#### 1. التحقق من تطابق حالة الـ Hardware مع حالة الـ Kernel

الـ LSM framework مش بيتعامل مع hardware مباشرة، لكن في interfaces مهمة:

```bash
# تحقق من الـ hardware security features المتاحة
grep -E 'smep|smap|umip|pku|la57' /proc/cpuinfo

# تحقق من تفعيل NX/XD bit (مرتبط بـ mmap_file hook)
dmesg | grep -i 'NX\|Execute Disable'

# تحقق من TPM (مرتبط بـ IMA/EVM اللي بتستخدم bdev_setintegrity hook)
ls /dev/tpm*
cat /sys/class/tpm/tpm0/tpm_version_major

# تحقق من IOMMU (مرتبط بـ ib_pkey_access hooks)
dmesg | grep -i 'IOMMU\|AMD-Vi\|Intel-IOMMU'
```

---

#### 2. Register Dump Techniques

بالنسبة للـ LSM، الـ "registers" المهمة هي registers الـ CPU للـ memory protection:

```bash
# اقرأ CR4 register (يحتوي على SMEP/SMAP bits) باستخدام rdmsr
# لازم kernel module أو rdmsr tool
modprobe msr
rdmsr 0x4   # CR4 equivalent via MSR

# استخدام /dev/mem لقراءة أماكن معينة (يتطلب root وCAP_SYS_RAWIO)
# تنبيه: ممكن يكون مقيّد بالـ kernel_read_file hook أو /dev/mem restrictions
dd if=/dev/mem bs=1 count=16 skip=$((0x1000)) 2>/dev/null | xxd

# استخدام devmem2 للـ MMIO registers (مثال: لـ TrEE/secure boot registers)
# devmem2 <address> [type [data]]
devmem2 0xFED40000 b   # مثال: TPM locality 0

# قراءة الـ MSRs المتعلقة بالـ security features
rdmsr 0x3A   # IA32_FEATURE_CONTROL (Secure Boot, VMX)
rdmsr 0x140  # IA32_XSS
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ LSM hooks هي software-only، لكن في حالات التكامل مع hardware security:

- **بالنسبة للـ TPM** (مستخدم مع IMA/EVM عبر `bdev_setintegrity`): تقدر تراقب الـ SPI أو LPC bus للتحقق من الـ TPM commands.
- **بالنسبة للـ Secure Boot**: راقب الـ SPI flash interface لقراءة الـ UEFI variables.
- **بالنسبة للـ InfiniBand** (عبر `ib_pkey_access`): استخدم الـ IB port analyzer لمراقبة الـ pkey exchanges.

```
Logic Analyzer Signals for TPM SPI:
  CS   ──┐ ┌──────────────────┐ ┌──
  CLK  ──┘ └──┐ ┌──┐ ┌──┐ ┌──┘ └──
  MOSI  ... [TPM_ACCESS_REG] [CMD] ...
  MISO  ... [RESP_STATUS]    [DATA] ...
```

---

#### 4. Hardware Issues → Kernel Log Patterns

| المشكلة | الـ Pattern في الـ Kernel Log |
|---------|------------------------------|
| TPM غير مستجيب (بيأثر على IMA) | `tpm tpm0: A TPM error ... occurred` |
| Secure Boot key غير موجود (kernel_read_file) | `MODSIGN: Modules are not signed` أو `Lockdown: ...` |
| IOMMU fault (يؤثر على ib_pkey_access) | `DMAR: DRHD ... fault addr ...` |
| Hardware RNG failure (يؤثر على key generation) | `random: crng init done` متأخر جداً |
| PCIe AER error مع IB devices | `pcieport ... AER: Corrected error ...` |
| Memory ECC error (بيأثر على الـ security blobs) | `EDAC ... CE ... on ...` |

```bash
# راقب TPM errors
dmesg | grep -i tpm

# راقب IOMMU faults
dmesg | grep -i 'DMAR\|iommu\|IOMMU'

# راقب Secure Boot / lockdown messages
dmesg | grep -i 'lockdown\|modsign\|secure.boot'
```

---

#### 5. Device Tree Debugging للـ LSM

الـ LSM نفسه مش بيتحكم في الـ DT مباشرة، لكن الـ hardware اللي بيستخدم LSM hooks (زي TPM وInfiniBand) ممكن يكون معرّف في الـ DT:

```bash
# تحقق من وجود TPM node في الـ DT
ls /proc/device-tree/ | grep tpm
cat /proc/device-tree/tpm@0/compatible 2>/dev/null

# تحقق من الـ DT overlay اللي بيعرّف security devices
dtc -I fs /proc/device-tree | grep -A5 tpm

# تحقق من الـ ACPI tables المتعلقة بـ security (بديل DT على x86)
ls /sys/firmware/acpi/tables/ | grep -E 'TPM|TCPA|DRTM'

# اقرأ الـ ACPI TPM table
acpidump -n TPMI 2>/dev/null | xxd | head -30
```

```bash
# لو TPM مش شغّال رغم وجوده في الـ DT
dmesg | grep -E 'tpm|tcg'

# تحقق من إن الـ kernel recognize الـ device
ls -la /sys/class/tpm/

# تحقق من الـ DT compatible string
cat /proc/device-tree/spi@xxx/tpm@0/compatible
# المتوقع: "tcg,tpm_tis-spi" أو "atmel,attpm20p"
```

---

### Practical Commands

---

#### سيناريو 1: تشخيص رفض الـ permission من LSM

```bash
# الخطوة 1: اعرف الـ LSM اللي رفض
dmesg | tail -50 | grep -E 'denied|DENIED|permission|security'

# الخطوة 2: لو SELinux
ausearch -m AVC -ts recent --just-one
# مثال output:
# type=AVC msg=audit(1234567890.123:456): avc: denied { read }
# for pid=1234 comm="nginx" name="secret.conf" dev="sda1" ino=12345
# scontext=system_u:system_r:httpd_t:s0
# tcontext=system_u:object_r:admin_home_t:s0 tclass=file permissive=0

# الخطوة 3: فهم الـ context
ls -Z /path/to/secret.conf     # اعرف الـ file context
ps -Z -p 1234                   # اعرف الـ process context
id -Z                           # اعرف الـ current user context

# الخطوة 4: إصلاح مؤقت للتشخيص (permissive mode)
setenforce 0   # تحويل لـ permissive مؤقتاً
# شغّل التطبيق وتحقق إنه اشتغل
setenforce 1   # رجّع enforcing

# الخطوة 5: إصلاح دائم
chcon -t httpd_sys_content_t /path/to/secret.conf
# أو باستخدام semanage لإصلاح دائم حتى بعد relabel
semanage fcontext -a -t httpd_sys_content_t "/path/to/secret.conf"
restorecon -v /path/to/secret.conf
```

#### سيناريو 2: تتبع LSM hooks باستخدام ftrace بشكل كامل

```bash
#!/bin/bash
# سكريبت كامل لتتبع الـ LSM hooks

DEBUGFS=/sys/kernel/debug/tracing

# إعداد الـ tracer
echo nop > $DEBUGFS/current_tracer
echo 0 > $DEBUGFS/tracing_on
echo '' > $DEBUGFS/set_ftrace_filter

# تحديد الـ hooks اللي عايز تتتبعها
cat >> $DEBUGFS/set_ftrace_filter << 'EOF'
security_inode_permission
security_file_permission
security_bprm_check
security_socket_connect
security_capable
EOF

# تفعيل الـ function graph tracer
echo function_graph > $DEBUGFS/current_tracer
echo 1 > $DEBUGFS/tracing_on

# تشغيل الأمر اللي عايز تتتبعه
$@

# إيقاف الـ tracing
echo 0 > $DEBUGFS/tracing_on

# عرض النتائج
cat $DEBUGFS/trace | head -100
```

```
# مثال output من الـ function_graph tracer:
# CPU  DURATION           FUNCTION CALLS
# |    |   |              |   |   |   |
# 0)   2.345 us    |  security_inode_permission() {
# 0)   0.123 us    |    selinux_inode_permission();
# 0)   2.100 us    |  }
# 0)   1.234 us    |  security_file_permission();
```

#### سيناريو 3: استخدام audit لتتبع الـ LSM events

```bash
# ضبط audit rules لتتبع LSM-relevant syscalls
auditctl -a always,exit -F arch=b64 -S open -S read -S write \
         -F exit=-EACCES -k lsm_deny
auditctl -a always,exit -F arch=b64 -S execve -k lsm_exec

# مراقبة real-time
ausearch -k lsm_deny --start recent -i | tail -f

# مثال output:
# ----
# type=SYSCALL msg=audit(02/27/2026 10:00:00.123:1234):
#   arch=x86_64 syscall=open success=no exit=-13(Permission denied)
#   a0=0x7fff... a1=O_RDONLY a2=0 a3=0
#   items=1 ppid=1000 pid=1234 auid=1000 uid=1000
#   gid=1000 euid=1000 ... comm=cat exe=/usr/bin/cat
# type=AVC msg=audit(...):
#   avc: denied { read } for pid=1234 comm="cat"
#   scontext=... tcontext=... tclass=file
```

#### سيناريو 4: تشخيص BPF security hooks

```bash
# تحقق من الـ BPF restrictions
cat /proc/sys/kernel/unprivileged_bpf_disabled
# 0 = مسموح للجميع
# 1 = root فقط (أو CAP_BPF)
# 2 = ممنوع تماماً حتى لـ root

# تتبع الـ bpf hook
echo 'security_bpf' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# جرّب BPF command
bpftool prog list

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# اعرض loaded BPF programs مع security info
bpftool prog show
bpftool map show
```

#### سيناريو 5: تشخيص network LSM hooks

```bash
# تتبع الـ socket hooks
echo 'security_socket_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# جرّب connection
curl -s https://example.com > /dev/null &

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep security_socket

# لو SELinux: تحقق من network contexts
netstat -Z    # اعرض security context لكل socket
ss -Z         # نفس الكلام باستخدام ss

# لو AppArmor: اعرض network rules
cat /sys/kernel/security/apparmor/profiles | grep -A5 "network"
```

#### سيناريو 6: فحص الـ integrity hooks

```bash
# تحقق من IMA policy
cat /sys/kernel/security/ima/policy

# اعرض IMA measurements
cat /sys/kernel/security/ima/ascii_runtime_measurements | head -20
# مثال output:
# 10 <sha256_hash> ima-ng sha256:<hash> /usr/bin/ls

# تحقق من EVM status
cat /sys/kernel/security/evm

# فحص xattrs للـ integrity
getfattr -n security.ima /usr/bin/ls
getfattr -n security.evm /usr/bin/ls

# تشغيل IMA في verbose mode
echo 'file security/integrity/ima/ima_main.c +p' > \
     /sys/kernel/debug/dynamic_debug/control
dmesg | grep -i ima
```

#### سيناريو 7: تشخيص الـ inode_init_security hook

```bash
# تفعيل debug لرؤية الـ xattr assignment عند إنشاء files
echo 'file security/selinux/hooks.c +p' > \
     /sys/kernel/debug/dynamic_debug/control

# إنشاء file وراقب الـ log
touch /tmp/test_security_file
dmesg | tail -10 | grep -i security

# اعرض الـ security xattrs للـ file
getfattr -d -m security /tmp/test_security_file
# مثال output:
# # file: tmp/test_security_file
# security.selinux="unconfined_u:object_r:user_tmp_t:s0"

# تحقق من الـ inode security blob size
cat /sys/kernel/debug/lsm/inode_blob_size 2>/dev/null || \
    grep -r 'inode' /sys/kernel/security/*/blob_sizes 2>/dev/null
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — SELinux بيمنع قراءة `/dev/ttyS2`

#### العنوان
**الـ SELinux hook بيمنع daemon صناعي من الوصول لـ UART على TI AM62x**

#### السياق
شركة بتبني industrial gateway بيشغّل Linux على TI AM62x (Sitara). الـ gateway بيقرأ بيانات من sensor عبر UART (`/dev/ttyS2`). الـ userspace daemon اسمه `modbus_d` بيشتغل كـ systemd service.

#### المشكلة
بعد تفعيل SELinux في الـ kernel config، الـ daemon بدأ يطلع error:

```
modbus_d: open /dev/ttyS2: Permission denied
```

مع إن الـ Unix permissions (`crw-rw---- root dialout`) صح وإن الـ daemon user عضو في `dialout` group.

#### التحليل

الـ VFS بتاعة الـ kernel لما حد بعمل `open()` على `/dev/ttyS2`، بيعدي عبر:

```c
/* security/security.c */
int security_file_open(struct file *file)
{
    /* يستدعي كل hook مسجّل في القائمة */
    return call_int_hook(file_open, 0, file);
}
```

الـ hook ده معرّف في `lsm_hook_defs.h`:

```c
LSM_HOOK(int, 0, file_open, struct file *file)
```

الـ default value هنا `0` (allow)، بس لو SELinux مسجّل hook، بيُستدعى:

```c
/* security/selinux/hooks.c */
static int selinux_file_open(struct file *file)
{
    /* بيتحقق من السياق الأمني للعملية مقارنةً بالـ inode label */
    return file_open_perm(file);
}
```

السبب: الـ `modbus_d` process context هو `system_u:system_r:modbus_t:s0`، لكن `/dev/ttyS2` label مش بيسمح لـ `modbus_t` بـ `open`.

للتشخيص:

```bash
# شوف الـ AVC denials
ausearch -m avc -ts recent | grep modbus

# أو مباشرة من dmesg
dmesg | grep "avc: denied" | grep ttyS2

# تحقق من label الملف
ls -lZ /dev/ttyS2

# تحقق من context الـ process
ps -eZ | grep modbus_d
```

الناتج:
```
type=AVC msg=audit(...): avc: denied { open } for pid=1234
  comm="modbus_d" path="/dev/ttyS2"
  scontext=system_u:system_r:modbus_t:s0
  tcontext=system_u:object_r:tty_device_t:s0
  tclass=chr_file
```

#### الحل

**الحل الصح**: تكتب SELinux policy module تديله الصلاحية:

```bash
# اعمل policy module
cat > modbus_uart.te << 'EOF'
module modbus_uart 1.0;

require {
    type modbus_t;
    type tty_device_t;
    class chr_file { open read write ioctl };
}

# السماح لـ modbus_t بفتح/قراءة/كتابة الـ tty device
allow modbus_t tty_device_t:chr_file { open read write ioctl };
EOF

checkmodule -M -m -o modbus_uart.mod modbus_uart.te
semodule_package -o modbus_uart.pp -m modbus_uart.mod
semodule -i modbus_uart.pp
```

أو لو عايز تعمل label مخصص للـ device:

```bash
# في ملف .fc لتعريف label مخصص لـ /dev/ttyS2
echo '/dev/ttyS2    --    system_u:object_r:modbus_device_t:s0' >> modbus_uart.fc
```

#### الدرس المستفاد

الـ `file_open` hook في `lsm_hook_defs.h` يعمل **بعد** الـ DAC checks (Unix permissions). تفعيل SELinux أو أي MAC module بيضيف طبقة أمان مستقلة تماماً عن الـ Unix permissions. مهندس kernel أو BSP لازم يعمل SELinux policy audit قبل shipment أي industrial product.

---

### السيناريو 2: Android TV Box على Allwinner H616 — `bpf_prog_load` hook بيكسّر OTA Update

#### العنوان
**الـ LSM hook بتاع BPF بيمنع تحميل BPF program أثناء OTA update على H616**

#### السياق
TV box Android بيشغّل kernel 5.15 على Allwinner H616. الـ OTA update client بيستخدم BPF program لتتبع network bandwidth أثناء التحميل. المنتج بيستخدم Android's Verified Boot + custom LSM خفيف (شبيه Landlock).

#### المشكلة
الـ OTA update بيفشل في مرحلة التحميل مع:

```
bpf(BPF_PROG_LOAD, ...): Operation not permitted
```

مع إن الـ OTA client بيشتغل بـ `CAP_SYS_ADMIN`.

#### التحليل

الـ `BPF_PROG_LOAD` syscall بتعدي عبر:

```c
LSM_HOOK(int, 0, bpf_prog_load, struct bpf_prog *prog,
         union bpf_attr *attr,
         struct bpf_token *token,
         bool kernel)
```

وكمان:

```c
LSM_HOOK(int, 0, bpf, int cmd,
         union bpf_attr *attr,
         unsigned int size,
         bool kernel)
```

الـ framework بيستدعي الاتنين. الـ custom LSM الخاص بالشركة بيعمل hook على `bpf_prog_load` وبيرفض أي BPF program من processes مش في whitelist، حتى لو عندها `CAP_SYS_ADMIN`.

```c
/* custom_lsm.c - مثال مبسّط للـ hook الخاص بالشركة */
static int custom_bpf_prog_load(struct bpf_prog *prog,
                                union bpf_attr *attr,
                                struct bpf_token *token,
                                bool kernel)
{
    /* رفض أي BPF من context مش في الـ approved list */
    if (!is_approved_context(current_cred()))
        return -EPERM;
    return 0;
}
```

للتشخيص:

```bash
# شغّل kernel tracing على LSM hooks
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_bpf/enable

# أو استخدم ftrace لتتبع الـ hook
echo 'security_bpf_prog_load' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace

# تحقق من الـ LSMs المفعّلة
cat /sys/kernel/security/lsm
```

#### الحل

في الـ custom LSM، أضف الـ OTA client process context لـ whitelist:

```c
/* custom_lsm.c */
static const char * const approved_contexts[] = {
    "u:r:system_server:s0",
    "u:r:update_engine:s0",   /* أضف OTA update context */
    NULL
};

static int custom_bpf_prog_load(struct bpf_prog *prog,
                                union bpf_attr *attr,
                                struct bpf_token *token,
                                bool kernel)
{
    char ctx_buf[128];
    /* استخرج الـ security context للـ process الحالي */
    security_current_getlsmprop_subj(...);

    if (is_in_approved_list(ctx_buf))
        return 0;
    return -EPERM;
}
```

أو استخدم `bpf_token` mechanism اللي المـ kernel بيوفّره:

```c
LSM_HOOK(int, 0, bpf_token_capable, const struct bpf_token *token, int cap)
```

بتنشئ token بصلاحيات محددة وتمرّره للـ OTA client بدل إعطاءه `CAP_SYS_ADMIN` كامل.

#### الدرس المستفاد

الـ BPF hooks في `lsm_hook_defs.h` — خصوصاً `bpf_prog_load` و`bpf_token_capable` — بتسمح بضبط دقيق جداً لمين يقدر يحمّل BPF programs. في embedded Android، الـ custom LSM لازم يراعي كل الـ system services اللي بتستخدم BPF (network stats، OTA، cgroup management).

---

### السيناريو 3: IoT Sensor على STM32MP1 — `kernel_load_data` بيمنع تحميل firmware للـ co-processor

#### العنوان
**الـ IMA/LSM hook بيرفض تحميل firmware للـ Cortex-M4 co-processor على STM32MP157**

#### السياق
STM32MP157 SoC بيحتوي على Cortex-A7 (بيشغّل Linux) وCortex-M4 (real-time co-processor). Linux بيحمّل الـ firmware للـ M4 عبر remoteproc framework. المنتج بيستخدم IMA (Integrity Measurement Architecture) لضمان سلامة الـ firmware.

#### المشكلة
بعد تفعيل IMA مع policy تستلزم توقيع كل الـ firmware، الـ remoteproc بدأ يطلع:

```
remoteproc remoteproc0: Failed to load firmware 'm4_firmware.elf': -EACCES
```

#### التحليل

`lsm_hook_defs.h` بيعرّف:

```c
LSM_HOOK(int, 0, kernel_load_data, enum kernel_load_data_id id, bool contents)
LSM_HOOK(int, 0, kernel_read_file, struct file *file,
         enum kernel_read_file_id id, bool contents)
LSM_HOOK(int, 0, kernel_post_read_file, struct file *file, char *buf,
         loff_t size, enum kernel_read_file_id id)
```

الـ remoteproc framework لما بيحمّل الـ firmware بيستخدم `request_firmware()` اللي بتعدي عبر:

```c
/* kernel/kexec_file.c و drivers/base/firmware_loader/main.c */
security_kernel_read_file(file, READING_FIRMWARE, true);
```

الـ IMA hook بيتحقق من signature الملف. الـ `m4_firmware.elf` مش موقّع بـ IMA key، فالـ hook بيرجع `-EACCES`.

```bash
# شوف IMA log
cat /sys/kernel/security/ima/ascii_runtime_measurements | grep m4_firmware

# شوف سبب الرفض
dmesg | grep "IMA: appraise"
# المتوقع:
# IMA: appraise integrity: Missing IMA xattr (m4_firmware.elf)

# تحقق من الـ IMA policy
cat /sys/kernel/security/ima/policy
```

الـ policy اللي سبّبت المشكلة:

```
appraise func=FIRMWARE_CHECK appraise_type=imasig
```

#### الحل

**الحل 1**: وقّع الـ firmware بـ IMA key:

```bash
# توليد key pair
openssl genrsa -out ima_signing_key.pem 2048
openssl req -new -x509 -key ima_signing_key.pem -out ima_cert.pem -days 365

# توقيع الـ firmware
evmctl ima_sign --key ima_signing_key.pem m4_firmware.elf

# تحقق من الـ xattr
getfattr -n security.ima m4_firmware.elf
```

**الحل 2**: لو الـ firmware بيُحمَّل من tmpfs (مش من disk)، عدّل الـ IMA policy لاستثناء remoteproc:

```
# /etc/ima/ima-policy
dont_appraise func=FIRMWARE_CHECK obj_type=remoteproc_firmware_t
appraise func=FIRMWARE_CHECK appraise_type=imasig
```

**الحل 3**: في بيئة development، استخدم:

```c
LSM_HOOK(int, 0, kernel_load_data, enum kernel_load_data_id id, bool contents)
```

الـ `id` بيحدد نوع البيانات — `LOADING_FIRMWARE` بيسمح لك تعمل exception محدود.

#### الدرس المستفاد

الـ `kernel_load_data` و`kernel_read_file` hooks بيشملوا كل أنواع البيانات اللي الـ kernel بيحمّلها: modules، firmware، kexec images. في SoCs زي STM32MP1 اللي فيها heterogeneous processing، لازم تفكر في signing pipeline للـ co-processor firmware من أول يوم في الـ BSP design.

---

### السيناريو 4: Automotive ECU على i.MX8QM — `ptrace_access_check` بيعطّل debugging في production

#### العنوان
**الـ Yama LSM hook بيمنع GDB من الـ attach على ECU process على NXP i.MX8QM**

#### السياق
شركة Tier-1 automotive بتبني ECU على NXP i.MX8QM. أثناء field debugging لمشكلة intermittent في CAN bus handler process، المهندس حاول يعمل `gdb --pid` لـ attach على الـ process المشتغل. الـ ECU بيشغّل Yama LSM مع `ptrace_scope=1`.

#### المشكلة

```bash
$ gdb --pid 1847
Attaching to process 1847
ptrace: Operation not permitted.
```

مع إن المهندس بيشتغل بـ `root`.

#### التحليل

الـ hook المسؤول:

```c
LSM_HOOK(int, 0, ptrace_access_check, struct task_struct *child,
         unsigned int mode)
```

الـ Yama LSM بيعمل override على الـ default behavior (اللي default value له `0` يعني allow):

```c
/* security/yama/yama_lsm.c */
static int yama_ptrace_access_check(struct task_struct *child,
                                    unsigned int mode)
{
    int rc = 0;

    switch (ptrace_scope) {
    case YAMA_SCOPE_RELATIONAL:
        /* scope=1: بيسمح بس لو child عمل PR_SET_PTRACER
           أو لو tracer هو ancestor */
        if (!task_is_descendant(current, child) &&
            !ptracer_exception_found(current, child) &&
            !ns_capable(task_user_ns(child), CAP_SYS_PTRACE))
            rc = -EPERM;
        break;
    /* ... */
    }
    return rc;
}
```

حتى مع `root`، في `scope=1` لازم أو الـ process يعمل `PR_SET_PTRACER` أو يكون هناك ancestor relationship.

```bash
# تحقق من الـ scope الحالي
cat /proc/sys/kernel/yama/ptrace_scope
# output: 1

# شوف الـ kernel log
dmesg | grep ptrace

# تحقق من LSMs المفعّلة
cat /sys/kernel/security/lsm
# output: yama,selinux
```

#### الحل

**للـ debugging المؤقت في field**:

```bash
# خفّض الـ scope مؤقتاً (بيتطلب CAP_SYS_PTRACE في user namespace)
echo 0 > /proc/sys/kernel/yama/ptrace_scope

# أو السماح للـ target process بالـ ptrace من أي process
# من داخل الـ process نفسه أو من script launch:
prctl(PR_SET_PTRACER, PR_SET_PTRACER_ANY, 0, 0, 0);
```

**للـ production-safe debugging**:

```bash
# استخدم gdbserver على الـ target
gdbserver :1234 --attach 1847

# من الـ host تاعك
gdb -ex "target remote ECU_IP:1234" can_handler
```

**في الـ BSP**، لو محتاج debugging في production بشكل منضبط:

```c
/* في الـ init script، ابعت SIGSTOP للـ process
   واعمل له ancestor-chain صح */
/* أو استخدم hook التاني: */
LSM_HOOK(int, 0, ptrace_traceme, struct task_struct *parent)
/* الـ child process بيطلب هو إنه يتـ trace */
```

#### الدرس المستفاد

الـ `ptrace_access_check` hook في `lsm_hook_defs.h` بيكون enabled في كل distribution تقريباً عبر Yama LSM. في automotive ECUs، الـ `ptrace_scope` لازم يكون `2` أو `3` في production (منع الـ ptrace تماماً)، مع وجود alternative debugging path عبر `gdbserver` أو hardware debuggers (JTAG/J-Link).

---

### السيناريو 5: Custom Board Bring-Up على RK3562 — `sb_mount` hook بيمنع mount الـ overlayfs لـ container runtime

#### العنوان
**الـ AppArmor/SELinux `sb_mount` hook بيمنع Docker من تشغيل containers على RK3562-based board**

#### السياق
شركة بتعمل bring-up لبورد مبني على Rockchip RK3562. الـ board بيشغّل Ubuntu-based Linux لـ edge AI inference. الفريق حاول يشغّل Docker containers لـ model serving.

#### المشكلة

```bash
$ docker run --rm hello-world
docker: Error response from daemon:
  OCI runtime create failed:
  container_linux.go:380:
  starting container process caused:
  process_linux.go:545:
  container init caused: rootfs_linux.go:76:
  mounting "/var/lib/docker/overlay2/.../merged" to rootfs at "/":
  mount /var/lib/docker/overlay2/.../merged:/var/lib/docker/overlay2/.../merged
  (via /proc/self/fd/6), flags: 0x5001: permission denied: unknown
```

#### التحليل

الـ overlayfs mount بيعدي عبر عدة hooks متتالية في `lsm_hook_defs.h`:

```c
/* 1. أول check عند بداية الـ mount */
LSM_HOOK(int, 0, sb_mount, const char *dev_name,
         const struct path *path,
         const char *type,
         unsigned long flags,
         void *data)

/* 2. بعد إنشاء الـ superblock */
LSM_HOOK(int, 0, sb_kern_mount, const struct super_block *sb)

/* 3. لو عايز يعمل move mount */
LSM_HOOK(int, 0, move_mount, const struct path *from_path,
         const struct path *to_path)
```

الـ AppArmor profile لـ Docker daemon بيسمح بـ `mount` لكن بقيود على الـ flags. الـ overlayfs بيستخدم `MS_PRIVATE` وغيره من الـ propagation flags اللي الـ AppArmor profile بيمنعها.

```bash
# شوف AppArmor denials
dmesg | grep apparmor | grep DENIED
# أو
journalctl -k | grep apparmor | grep mount

# تحقق من الـ profile المفعّل لـ dockerd
cat /proc/$(pgrep dockerd)/attr/current

# شوف الـ AppArmor profile
cat /etc/apparmor.d/docker
```

الناتج المتوقع من dmesg:

```
apparmor="DENIED" operation="mount" info="failed flags check"
  name="/var/lib/docker/overlay2/xxx/merged"
  pid=2341 comm="dockerd"
  flags="rw,rprivate" -> denied
```

```bash
# تحقق من الـ mounts اللي AppArmor بيسمح بيها
grep -A20 "mount" /etc/apparmor.d/docker
```

#### الحل

**الحل 1**: تحديث الـ AppArmor profile لـ Docker:

```
# /etc/apparmor.d/docker-custom
profile docker-default flags=(attach_disconnected, mediate_deleted) {
  # السماح بـ overlayfs mounts
  mount fstype=overlay,
  mount options=(rw, rprivate) -> /var/lib/docker/overlay2/**,
  mount options=(rw, rbind) -> /var/lib/docker/overlay2/**,

  # السماح بـ proc وغيره داخل الـ container
  mount fstype=proc -> /proc/,
  mount fstype=tmpfs,
  mount fstype=devtmpfs,
}
```

```bash
# reload الـ profile
apparmor_parser -r /etc/apparmor.d/docker-custom
systemctl restart docker
```

**الحل 2**: لو الـ board bring-up وما تحتاج تقييد صارم:

```bash
# disable الـ AppArmor profile لـ docker مؤقتاً
aa-complain /etc/apparmor.d/docker
# أو
echo 0 > /proc/sys/kernel/apparmor_restrict_unprivileged_userns
```

**الحل 3**: استخدم الـ `sb_set_mnt_opts` hook لفهم إيه اللي بيتمنع:

```c
/* في kernel module للتشخيص */
static int debug_sb_mount(const char *dev_name,
                          const struct path *path,
                          const char *type,
                          unsigned long flags, void *data)
{
    pr_info("LSM: mount attempt: dev=%s type=%s flags=0x%lx\n",
            dev_name ?: "none", type ?: "none", flags);
    return 0; /* بنرصد بس، مش بنمنع */
}
```

#### الدرس المستفاد

الـ `sb_mount` و`move_mount` hooks في `lsm_hook_defs.h` حساسين جداً في بيئات containerization. أثناء board bring-up على RK3562 أو أي SoC جديد، ابدأ بـ AppArmor في `complain mode` لتسجيل الـ denials قبل الـ enforce mode. الـ overlayfs بيحتاج خصوصاً `mount fstype=overlay` permission صريحة في الـ LSM policy، ومش بيكفي إن الـ Unix permissions تمام.

---
## Phase 7: مصادر ومراجع

### التوثيق الرسمي للـ Kernel

| المصدر | الوصف |
|--------|-------|
| [`Documentation/security/lsm.rst`](https://docs.kernel.org/security/lsm.html) | الدليل الرسمي لـ LSM framework — يشرح آلية الـ hooks، الـ stacking، وكيفية كتابة LSM جديد |
| [`Documentation/security/lsm-development.rst`](https://docs.kernel.org/security/lsm-development.html) | دليل تطوير الـ LSM modules |
| [`Documentation/security/SELinux.rst`](https://docs.kernel.org/security/SELinux.html) | توثيق SELinux — أشهر تطبيق للـ LSM hooks |
| [`include/linux/lsm_hook_defs.h`](https://elixir.bootlin.com/linux/latest/source/include/linux/lsm_hook_defs.h) | الملف الأساسي — كل الـ hooks معرَّفة هنا بـ macro واحد |
| [`include/linux/security.h`](https://elixir.bootlin.com/linux/latest/source/include/linux/security.h) | واجهة الـ security subsystem — الـ wrappers الخارجية |
| [`security/security.c`](https://elixir.bootlin.com/linux/latest/source/security/security.c) | التنفيذ الفعلي للـ LSM framework وآلية الـ hook dispatch |

---

### مقالات LWN.net الأساسية

دي أهم المقالات اللي بتغطي تطور الـ LSM hooks من الأول لحد دلوقتي:

#### الأساسيات والتاريخ

- **[Linux Security Modules: General Security Hooks for Linux](https://static.lwn.net/kerneldoc/security/lsm.html)**
  — التوثيق الأصلي اللي كتبه Crispin Cowan وفريقه، بيشرح الفلسفة الأساسية وراء الـ LSM hooks.

- **[Complete coverage in Linux security modules](https://lwn.net/Articles/154277/)**
  — ناقش مشكلة الـ hooks اللي بتتجاوز بعض paths في الـ VFS وتأثيرها على الأمان.

- **[The future of the Linux Security Module API](https://lwn.net/Articles/180194/)**
  — نقاش مبكر حول مستقبل الـ API ومشاكله.

- **[The return of authoritative hooks](https://lwn.net/Articles/273822/)**
  — ناقش فكرة الـ hooks اللي بتقدر تبدّل سلوك الـ kernel مش بس تراقبه.

#### الـ LSM Stacking

- **[Supporting multiple LSMs](https://lwn.net/Articles/426921/)**
  — أول نقاش جدي حول تشغيل أكتر من LSM في نفس الوقت — ليه صعب وإيه المقترحات.

- **[Progress in security module stacking](https://lwn.net/Articles/635771/)**
  — تقرير عن التقدم في موضوع الـ stacking بعد سنين من المحاولات.

- **[LSM stacking and the future](https://lwn.net/Articles/804906/)**
  — عرض من Linux Security Summit 2019 بيناقش use cases زي الـ containers.

- **[LSM: Module stacking for AppArmor](https://lwn.net/Articles/837994/)**
  — تفاصيل محاولة Ubuntu لعمل stacking بين AppArmor وبقية الـ LSMs.

- **[LSM: General module stacking](https://lwn.net/Articles/955464/)**
  — أحدث جهد لعمل stacking شامل لكل الـ major LSMs.

- **[A change in direction for security-module stacking?](https://lwn.net/Articles/970070/)**
  — مقال حديث بيناقش تغيير الاتجاه في حل مشكلة الـ stacking.

#### BPF و LSM

- **[Impedance matching for BPF and LSM](https://lwn.net/Articles/813261/)**
  — بيشرح KRSI (Kernel Runtime Security Instrumentation) وكيف بتتربط الـ BPF programs بالـ LSM hooks في `lsm_hook_defs.h`.

- **[BPF and security](https://lwn.net/Articles/946389/)**
  — نظرة شاملة على استخدام BPF في الأمان وتأثيره على الـ LSM hooks.

#### Hooks جديدة وتطورات

- **[A security-module hook for user-namespace creation](https://lwn.net/Articles/903580/)**
  — ناقش إضافة hook جديد (`userns_create`) — نفس الـ hook الموجود دلوقتي في الملف.

- **[LSM: Three basic syscalls](https://lwn.net/Articles/928790/)**
  — الـ syscalls الجديدة `lsm_get_self_attr` و`lsm_set_self_attr` اللي بتتعامل مع الـ `getselfattr`/`setselfattr` hooks.

- **[Safe LSM (un)loading, and immutable hooks](https://lwn.net/Articles/752779/)**
  — بيشرح ليه الـ hooks بقت immutable وإزاي بيأثر ده على الـ `hlist_head` structure.

- **[Landlock LSM: Unprivileged sandboxing](https://lwn.net/Articles/698226/)**
  — LSM جديد بالكامل بيستخدم الـ hooks الموجودة لعمل sandboxing بدون privileges.

---

### Commits مهمة في الـ Kernel

| الـ Commit | الوصف |
|-----------|-------|
| [`v2.6.0`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/security) | إدخال الـ LSM framework لأول مرة من WireX و Greg Kroah-Hartman |
| [`bbd3662a8348`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bbd3662a8348) | تحويل الـ hooks لـ `hlist` بدل الـ function pointers المباشرة |
| [BPF LSM v39](https://lore.kernel.org/linux-security-module/20200220175657.4619741-1-kpsingh@chromium.org/) | إضافة BPF LSM hooks للـ `lsm_hook_defs.h` |

---

### نقاشات Mailing List

- **[linux-security-module mailing list](https://lore.kernel.org/linux-security-module/)**
  — الأرشيف الكامل لكل النقاشات حول الـ LSM hooks والـ patches.

- **[LSM Infrastructure management of the superblock](https://www.openwall.com/lists/kernel-hardening/2020/08/02/10)**
  — نقاش تقني عميق على kernel-hardening list حول الـ `sb_*` hooks.

- **[Patchwork: linux-security-module project](https://patchwork.kernel.org/project/linux-security-module/list/)**
  — كل الـ patches المقترحة للـ LSM subsystem.

---

### مصادر kernelnewbies.org

- **[Linux 4.2 — LSM Stacking](https://kernelnewbies.org/Linux_4.2)**
  — الإصدار اللي أضاف دعم stacking الأولي للـ LSMs.

- **[Linux 6.3 — LSM improvements](https://kernelnewbies.org/Linux_6.3)**
  — تحسينات على الـ LSM framework في إصدار 6.3.

- **[Linux 6.12 — Security changes](https://kernelnewbies.org/Linux_6.12)**
  — أحدث تغييرات في الـ security subsystem.

- **[KernelGlossary](https://kernelnewbies.org/KernelGlossary)**
  — مصطلحات مفيدة زي LSM, capabilities, secid.

---

### مصادر eLinux.org

- **[Security Working Group](https://elinux.org/Security_Working_Group)**
  — مجموعة العمل بتاعة embedded Linux security — بتتعامل مع LSM في بيئة الـ embedded systems.

- **[Category:Security](https://elinux.org/Category:Security)**
  — كل الصفحات المتعلقة بالأمان في eLinux.

- **[Security Hardware Resources](https://elinux.org/Security_Hardware_Resources)**
  — موارد الـ hardware security اللي بتتعامل مع الـ LSM hooks على مستوى أعمق.

---

### كتب موصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)

**الـ chapter المهم:** Chapter 1 (Introduction) — بيشرح أساسيات الـ kernel subsystems وإزاي الـ security layer بيتعامل مع الـ VFS.

> متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)

| الـ Chapter | الموضوع |
|------------|---------|
| Chapter 13: The Virtual Filesystem | بيفهمك الـ VFS hooks اللي الـ LSM بيتكلم عليها (inode, dentry, file) |
| Chapter 14: The Block I/O Layer | مهم لفهم الـ `bdev_*` hooks الجديدة |
| Chapter 9: An Introduction to Kernel Synchronization | مهم لفهم الـ `hlist_head` locking |

#### Understanding the Linux Kernel — Bovet & Cesati (3rd Edition)

**الأهم:** Chapter 20 (System Calls) وChapter 19 (Process Communication) — بيفهمك الـ context اللي بتشتغل فيه الـ `task_*` و`ipc_*` hooks.

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

**الـ chapter المهم:** Chapter 16 (Kernel Debugging Techniques) وأي section بيتكلم عن kernel security في embedded — مفيد لفهم الـ hooks في بيئات الـ embedded المقيدة.

#### The Art of Linux Kernel Design — Lixiang Yang

بيشرح تصميم الـ LSM framework وعلاقته بالـ VFS بشكل مبسط — مناسب كـ bridge بين LDD3 والـ source code.

---

### أدوات بحث إضافية

#### Bootlin Elixir — Cross-referencer

```
https://elixir.bootlin.com/linux/latest/source/include/linux/lsm_hook_defs.h
```

الأداة دي بتخليك تتتبع كل استخدام لأي hook في الـ kernel كله — مفيد جداً لفهم إمتى بيتم استدعاء كل hook.

#### كيفية البحث في Mailing List

```bash
# ابحث عن نقاشات حول hook معين
https://lore.kernel.org/linux-security-module/?q=inode_permission

# ابحث في patchwork
https://patchwork.kernel.org/project/linux-security-module/list/?q=lsm_hook_defs
```

---

### مصطلحات البحث

لو حابب تلاقي معلومات أكتر، استخدم المصطلحات دي:

```
LSM hook implementation linux kernel
lsm_hook_defs.h security_hook_list
SELinux AppArmor Smack hooks
BPF-LSM KRSI kernel runtime security
linux security stacking multiple LSM
capability hooks linux kernel
linux audit LSM integration
xfrm LSM IPSec security context
landlock LSM unprivileged
```

---

### جدول تلخيصي للمصادر حسب الهدف

| لو هدفك | اقرأ |
|---------|------|
| فهم المبادئ الأساسية | [docs.kernel.org/security/lsm](https://docs.kernel.org/security/lsm.html) + LDD3 |
| كتابة LSM جديد | [lsm-development.rst](https://docs.kernel.org/security/lsm-development.html) + LWN [180194](https://lwn.net/Articles/180194/) |
| فهم الـ stacking | LWN [426921](https://lwn.net/Articles/426921/) + [955464](https://lwn.net/Articles/955464/) |
| استخدام BPF-LSM | LWN [813261](https://lwn.net/Articles/813261/) + [946389](https://lwn.net/Articles/946389/) |
| debug الـ hooks | Bootlin Elixir + patchwork mailing list |
| embedded security | eLinux Security Working Group + Embedded Linux Primer |
## Phase 8: Writing simple module

### الـ Hook المختار: `file_open`

الـ `file_open` LSM hook بيتنادى كل ما process تحاول تفتح file. ده بيخليه مثالي للمراقبة لأنه بيعطيك اسم الـ file، الـ PID، والـ UID قبل ما الـ kernel يكمل العملية.

بنستخدم **kprobe** على الـ `security_file_open` — ده الـ wrapper الـ kernel بيناديه عشان يشغّل كل الـ registered LSM hooks للـ `file_open`. بالـ kprobe احنا مش بنسجّل LSM module حقيقي، بس بنراقب دخول الـ function من غير ما نحتاج كل infrastructure الـ LSM.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * lsm_file_open_probe.c
 *
 * Attaches a kprobe to security_file_open() to log every
 * file-open event: PID, UID, and the file path.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe    */
#include <linux/fs.h>           /* struct file, struct path, d_path  */
#include <linux/dcache.h>       /* dentry helpers used by d_path     */
#include <linux/sched.h>        /* current, task_pid_nr              */
#include <linux/cred.h>         /* current_uid(), from_kuid          */
#include <linux/uidgid.h>       /* kuid_t                            */
#include <linux/uaccess.h>      /* not needed here but conventional  */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Ahmed <ahmed@example.com>");
MODULE_DESCRIPTION("kprobe on security_file_open — log every file open");

/* Buffer size for d_path result */
#define PATH_BUF_SZ 256

/*
 * pre_handler — called just before security_file_open() executes.
 *
 * Prototype of security_file_open:
 *   int security_file_open(struct file *file);
 *
 * On x86_64 the first argument lands in register %rdi,
 * which pt_regs maps to regs->di.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve the first argument: pointer to struct file */
    struct file *file = (struct file *)regs->di;
    char *buf, *path_str;
    kuid_t uid;

    if (!file)
        return 0;

    /* Allocate a temporary buffer for d_path on the kernel heap */
    buf = kmalloc(PATH_BUF_SZ, GFP_ATOMIC);
    if (!buf)
        return 0;

    /*
     * d_path resolves the dentry + vfsmount inside file->f_path
     * into a human-readable absolute path string.
     * It writes backwards from the end of buf, so the returned
     * pointer may not equal buf — it points inside buf.
     */
    path_str = d_path(&file->f_path, buf, PATH_BUF_SZ);

    /* d_path returns ERR_PTR on failure (e.g. path too long) */
    if (IS_ERR(path_str)) {
        path_str = "<error>";
    }

    /* current_uid() returns a kuid_t; convert to raw u32 for printing */
    uid = current_uid();

    pr_info("lsm_probe: pid=%d uid=%u file=%s\n",
            task_pid_nr(current),
            from_kuid(&init_user_ns, uid),
            path_str);

    kfree(buf);
    return 0; /* 0 means "continue normal execution" */
}

/* Define and initialise the kprobe struct */
static struct kprobe kp = {
    .symbol_name = "security_file_open",  /* symbol to probe   */
    .pre_handler = handler_pre,           /* our callback      */
};

/* ---- module_init -------------------------------------------------- */
static int __init lsm_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("lsm_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("lsm_probe: planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ---- module_exit -------------------------------------------------- */
static void __exit lsm_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("lsm_probe: kprobe removed\n");
}

module_init(lsm_probe_init);
module_exit(lsm_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية لأي kernel module: `MODULE_LICENSE`، `module_init`، `module_exit` |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل API الـ kprobe |
| `linux/fs.h` | تعريف `struct file` و `struct path` اللي الـ hook بيستلمهم |
| `linux/dcache.h` | `d_path()` اللي بتحوّل الـ dentry لـ string مقروءة |
| `linux/sched.h` | الـ macro `current` (pointer للـ task الحالي) و `task_pid_nr` |
| `linux/cred.h` | `current_uid()` عشان نجيب الـ UID من الـ credentials |
| `linux/uidgid.h` | `from_kuid()` بتحوّل `kuid_t` لـ `u32` عشان نطبعه |

---

#### الـ `handler_pre` — الـ Callback

**`struct pt_regs *regs`** بيحتوي على حالة الـ registers لحظة إيقاف التنفيذ.
على x86_64، الـ ABI بتحط أول argument في `rdi`، وده بيظهر في `regs->di`، فاحنا بنقرأ منه الـ `struct file *`.

**`GFP_ATOMIC`** في `kmalloc` ضروري لأن الـ kprobe handler ممكن يتنادى في context مش قابل للـ sleep (زي interrupt context أو spinlock held).

**`d_path`** بتاخد `struct path` (اللي جوّاها `dentry` + `vfsmount`) وبترجع absolute path string. الـ pointer الراجع بيكون جوّا الـ buffer مش في أوله، فمش لازم تـ free الـ path_str بشكل منفصل.

**`from_kuid(&init_user_ns, uid)`** بتحوّل الـ kernel-internal UID لرقم مقروء بالنسبة لـ initial user namespace.

---

#### الـ `struct kprobe kp`

الـ `symbol_name` بيقول للـ kprobe subsystem يدوّر على `security_file_open` في الـ kernel symbol table ويحط breakpoint فيه.
الـ `pre_handler` هو الـ callback اللي بيتنادى *قبل* ما الـ function تنفّذ، وده بيسمحلنا نشوف الـ arguments.

---

#### الـ `module_init` / `module_exit`

`register_kprobe` بتطلب من الـ kernel يحط software breakpoint على العنوان المطلوب وبتربط الـ callback بيه. لو رجعت قيمة سالبة يبقى في مشكلة (مثلاً الـ symbol مش موجود أو الـ kprobes مش مفعّلة في الكرنل).

`unregister_kprobe` في الـ exit مهم جداً — لو ما اتنادتش، الـ kernel هيفضل يناول control للـ handler بعد ما الـ module اتمسح من الذاكرة، وده هيسبب kernel panic فوري.

---

### بناء وتشغيل الـ Module

```bash
# Makefile في نفس الـ directory
obj-m += lsm_file_open_probe.o

# بناء
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل
sudo insmod lsm_file_open_probe.ko

# مشاهدة الـ log
sudo dmesg -w | grep lsm_probe

# إزالة
sudo rmmod lsm_file_open_probe
```

مثال على output في `dmesg`:

```
lsm_probe: planted kprobe at security_file_open (ffffffffa1234560)
lsm_probe: pid=1842 uid=1000 file=/home/ahmed/.bashrc
lsm_probe: pid=1842 uid=1000 file=/usr/lib/locale/locale-archive
lsm_probe: pid=203  uid=0    file=/proc/sys/kernel/randomize_va_space
```
