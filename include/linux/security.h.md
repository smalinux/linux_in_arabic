## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف `include/linux/security.h` ينتمي لـ **SECURITY SUBSYSTEM** (Linux Security Module framework — LSM)، المسؤول عنه Paul Moore وJames Morris وSerge Hallyn، وقناته `linux-security-module@vger.kernel.org`.

---

### الفكرة من البداية — قبل أي كود

تخيّل إنك بتبني مبنى ضخم (Linux kernel). المبنى ده فيه أبواب كتير: باب للـ files، باب للـ network، باب للـ processes، باب للـ memory. كل ما حد حاول يفتح باب، المبنى لازم يسأل: "إيه ده مسموح ولا لأ؟"

المشكلة القديمة: مين يقرر "مسموح ولا لأ"؟ في الأول كان الـ kernel بيعمل القرار ده بنفسه بقواعد ثابتة (DAC — Discretionary Access Control). بس الشركات والحكومات والـ security researchers قالوا: احنا عايزين نغيّر القواعد دي. فيه ناس عايزين SELinux، وفيه ناس عايزين AppArmor، وفيه ناس عايزين Smack.

**الحل:** بدل ما الـ kernel يقفل نفسه على نظام أمان واحد، عمل **plugin system** اسمه LSM — Linux Security Module framework. وملف `security.h` ده هو الـ **public contract** بين الـ kernel والـ security modules. يعني: كل مرة الـ kernel محتاج يعمل قرار أمني، بيندي الـ LSM framework وبيقوله "إيه رأيك؟".

---

### القصة عشان تتخيل المشكلة

**بدون LSM:**
```
process يعمل open("/etc/shadow")
    ↓
kernel يشوف: هل الـ UID = 0؟ (أو الـ permissions السماح؟)
    ↓
يرفض أو يوافق — قواعد ثابتة مش قابلة للتغيير
```

**مع LSM (security.h):**
```
process يعمل open("/etc/shadow")
    ↓
kernel يندي security_inode_permission()  ← من security.h
    ↓
الـ LSM framework يمرر الطلب لكل module مسجّل (SELinux, AppArmor, ...)
    ↓
لو أي module قال "ارفض" → الـ kernel يرفض
    ↓
لو كلهم وافقوا → الـ kernel يكمل
```

---

### الهدف من الملف

`include/linux/security.h` هو الـ **single interface** اللي الـ kernel بيستخدمه للتواصل مع أي LSM. بيعمل حاجتين:

1. **لما `CONFIG_SECURITY` مفعّل:** بيعلن عن دوال زي `security_inode_permission()` اللي بتروح للـ LSMs الحقيقية.
2. **لما `CONFIG_SECURITY` مش مفعّل:** بيعمل `static inline` stubs بترجع 0 (مسموح) فوراً بدون أي overhead — الـ compiler يحذفها خالص.

ده معناه إن الـ kernel code العادي مش محتاج يعرف هل فيه LSM مركّب ولا لأ. بيكلم `security_*()` براحته وهو مش دارٍ.

---

### ليه ده مهم؟

| بدون security.h | مع security.h |
|---|---|
| الـ kernel لازم يعرف كل LSM بالاسم | الـ kernel ما يعرفش ولا module |
| تغيير سياسة الأمان = patch في الـ kernel | تغيير LSM = module جديد بس |
| مستحيل تشغّل أكتر من نظام أمان | ممكن تشغّل SELinux + BPF LSM مع بعض |

---

### التغطية — إيه اللي بيحميه security.h

الملف ده بيغطي **كل نقاط التقاطع الأمنية** في الـ kernel:

| المنطقة | أمثلة من الملف |
|---|---|
| **Capabilities** | `security_capable()`, `security_capset()` |
| **Process/Credentials** | `security_task_alloc()`, `security_prepare_creds()`, `security_task_kill()` |
| **Filesystem** | `security_inode_permission()`, `security_inode_create()`, `security_file_open()` |
| **Superblock/Mount** | `security_sb_mount()`, `security_sb_umount()`, `security_sb_remount()` |
| **IPC** | `security_msg_queue_msgsnd()`, `security_shm_shmat()`, `security_sem_semop()` |
| **Network** | `security_socket_bind()`, `security_socket_connect()`, `security_sk_alloc()` |
| **XFRM (IPsec)** | `security_xfrm_policy_alloc()`, `security_xfrm_state_alloc()` |
| **Keys** | `security_key_alloc()`, `security_key_permission()` |
| **BPF** | `security_bpf()`, `security_bpf_prog_load()` |
| **Audit** | `security_audit_rule_match()` |
| **Perf Events** | `security_perf_event_open()`, `security_perf_event_read()` |
| **io_uring** | `security_uring_override_creds()`, `security_uring_cmd()` |
| **Kernel integrity** | `security_kernel_load_data()`, `security_kernel_read_file()` |
| **Lockdown** | `security_locked_down()` مع `enum lockdown_reason` |
| **InfiniBand** | `security_ib_pkey_access()` |

---

### الـ Lockdown — ميزة خاصة

الملف بيعرّف `enum lockdown_reason` بأسباب كتير زي:
- `LOCKDOWN_MODULE_SIGNATURE` — منع تحميل modules غير موقّعة
- `LOCKDOWN_KEXEC` — منع kexec
- `LOCKDOWN_DEBUGFS` — إخفاء debugfs

الـ lockdown مقسّم لمرحلتين: **integrity** (منع تعديل الـ kernel) و**confidentiality** (منع قراءة بيانات الـ kernel).

---

### الـ lsm_prop و lsm_context

```c
/* كل LSM عنده "بصمة أمنية" لكل كيان */
struct lsm_prop {
    struct lsm_prop_selinux selinux;  /* SELinux security ID */
    struct lsm_prop_smack   smack;    /* Smack label */
    struct lsm_prop_apparmor apparmor;/* AppArmor context */
    struct lsm_prop_bpf     bpf;      /* BPF LSM data */
};

/* النص الممثّل للسياق الأمني */
struct lsm_context {
    char *context;  /* e.g. "unconfined_u:unconfined_r:unconfined_t" */
    u32   len;
    int   id;       /* أي LSM صنع السياق ده */
};
```

ده بيخلّي الـ network stack مثلاً يقدر يبعت السياق الأمني مع الـ packets (SELinux CIPSO/CALIPSO).

---

### الملفات ذات الصلة اللي المقروء المفروض يعرفها

**الـ Core Framework:**

| الملف | الدور |
|---|---|
| `include/linux/security.h` | الملف ده — الـ public API |
| `include/linux/lsm_hooks.h` | تعريف `security_hook_list`, `lsm_info`, `lsm_blob_sizes` — اللي كل LSM بيسجّل نفسه بيه |
| `include/linux/lsm_hook_defs.h` | قائمة كل الـ hooks بماكرو `LSM_HOOK(...)` |
| `include/uapi/linux/lsm.h` | الـ IDs (LSM_ID_SELINUX=101 ...) وـ `lsm_ctx` للـ userspace |
| `security/security.c` | التنفيذ الفعلي لكل دوال `security_*()` |
| `security/lsm_init.c` | تسجيل وتهيئة الـ LSMs |
| `security/commoncap.c` | تنفيذ دوال الـ capabilities (`cap_capable()` إلخ) |
| `security/lsm_audit.c` | كتابة audit logs للأحداث الأمنية |

**الـ LSM Modules (كل واحد فيهم subsystem مستقل):**

| الملف | الدور |
|---|---|
| `security/selinux/` | SELinux — الأقوى والأكثر تعقيداً |
| `security/apparmor/` | AppArmor — مشهور في Ubuntu |
| `security/smack/` | Smack — بسيط للـ embedded |
| `security/bpf/` | BPF LSM — hooks قابلة للبرمجة |
| `security/lockdown/lockdown.c` | Lockdown LSM |
| `security/yama/yama_lsm.c` | Yama — حماية ptrace |
| `security/landlock/` | Landlock — sandbox للـ userspace |
| `security/integrity/` | IMA/EVM — integrity measurement |

**الـ securityfs (جزء من الملف):**

| الملف | الدور |
|---|---|
| `security/inode.c` | تنفيذ `securityfs_create_file/dir()` — virtual FS تحت `/sys/kernel/security/` |

---

### صورة تخطيطية للعلاقة

```
kernel subsystems (fs, net, ipc, mm, ...)
         |
         | يستدعي security_*() من security.h
         ↓
  [ LSM Framework — security/security.c ]
         |
         | يوزع الطلب على كل module مسجّل
    ┌────┴───────────────────────────────┐
    ↓         ↓          ↓          ↓
SELinux   AppArmor    Smack      BPF LSM
(101)      (104)       (102)      (109)
    └────┬───────────────────────────────┘
         |
         | لو أي واحد رفض → -EACCES أو -EPERM
         ↓
  kernel يكمّل أو يرفض الـ operation
```

---

### ملاحظة عملية

لما بتكتب LSM جديد، بتسجّل نفسك بـ `DEFINE_LSM(mymod)` في `lsm_hooks.h`، وبتملأ جدول الـ hooks اللي انت عايزها. الـ `security.h` بيضمن إن الـ kernel يوصلك في الوقت الصح.
## Phase 2: شرح الـ Linux Security Module (LSM) Framework

---

### المشكلة اللي بيحلها الـ LSM

الـ Linux kernel زمان كان فيه نظام أمان واحد بس: الـ **DAC (Discretionary Access Control)** — يعني permissions الـ Unix التقليدية (owner/group/other + rwx). كمان فيه الـ **POSIX Capabilities** اللي بتقسم صلاحيات الـ root لأجزاء أصغر.

المشكلة إن الـ DAC مش كافي لبيئات زي:
- **SELinux** — الـ NSA محتاجة mandatory access control حقيقي على كل object في النظام.
- **AppArmor** — Canonical عايزة path-based confinement لكل process.
- **Smack** — Simplified Mandatory Access Control يشتغل في embedded systems.
- **BPF-LSM** — تعريف policy بـ eBPF programs في runtime.

لو كل LSM اتبنى مباشرة في core kernel code، الكود هيبقى chaos — كل subsystem هيلاقي نفسه بيعمل `#ifdef CONFIG_SELINUX` في كل مكان. وكمان مستحيل تشغّل أكتر من LSM واحد في نفس الوقت.

---

### الحل — الـ Hook-based Plugin Architecture

الـ LSM framework بيعمل **security hooks** في كل نقطة حساسة في الكيرنل — قبل ما أي عملية تحصل على resource مهم. كل hook بيسأل السؤال: "هل مسموح بالعملية دي؟"

الفكرة الجوهرية:
- الـ **core kernel** بيحدد *أين* يتم التحقق — بس مش *كيف*.
- الـ **LSM implementation** (SELinux مثلاً) بيحدد *كيف* يتم التحقق.
- الـ `security_*` functions في `security.h` هي الواجهة الوحيدة اللي بيكلمها الكيرنل.

لو `CONFIG_SECURITY` مش مفعّل، كل الـ `security_*` functions بتتحول لـ `static inline` بترجع `0` أو تعمل fallback للـ capabilities — zero overhead تماماً.

---

### تشبيه حقيقي — Airport Security Checkpoint

تخيل مطار فيه checkpoints أمنية عند كل بوابة مهمة:

| المطار | الـ LSM |
|--------|---------|
| بوابات الدخول في كل مكان حساس | الـ security hooks في الكيرنل |
| شركة الأمن اللي بتشغّل الـ checkpoints | الـ LSM المفعّل (SELinux / AppArmor) |
| قواعد السفر الدولي العامة | الـ DAC + Capabilities (دايماً موجودة) |
| بطاقة الهوية بتاعت المسافر | الـ security label (secid / lsm_prop) |
| تذكرة السفر + التأشيرة | الـ credentials (struct cred) |
| قرار: يعدي أو يترفض | return value من الـ hook (0 أو -EACCES) |
| مدير الأمن اللي بيضع القواعد | LSM policy (SELinux policy file) |
| تقرير الحادثة الأمني | الـ audit log |

النقطة الأعمق: المطار (الكيرنل) مش لازم يعرف أنظمة أمان شركة الأمن بالتفصيل — بس لازم يعرف *وين* يحط الـ checkpoints. شركة الأمن (LSM) تتغير من شركة لشركة بدون ما تغيّر بنية المطار.

---

### الـ Big Picture Architecture

```
User Space
  │  open("/etc/passwd", O_RDONLY)
  │
  ▼
┌─────────────────────────────────────────────────────┐
│                   System Call Layer                  │
│  sys_openat() → do_filp_open() → vfs_open()         │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼ security_file_open(file)   ← security hook call
┌─────────────────────────────────────────────────────┐
│              LSM Framework (security/security.c)     │
│                                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │           security_hook_heads               │    │
│  │  .file_open ──► [cap] ──► [selinux] ──► ... │    │
│  │  .inode_permission ──► [cap] ──► [apparmor] │    │
│  │  .task_kill ──► [selinux] ──► [bpf]         │    │
│  └─────────────────────────────────────────────┘    │
│         │ iterates all registered LSMs              │
└─────────┬───────────────────────────────────────────┘
          │
    ┌─────┴──────┬─────────────┬───────────────┐
    ▼            ▼             ▼               ▼
┌────────┐ ┌─────────┐ ┌──────────┐ ┌──────────────┐
│CapabH. │ │ SELinux │ │AppArmor  │ │  BPF-LSM     │
│commoncap│ │security/│ │security/ │ │  (eBPF progs)│
│.c      │ │selinux/ │ │apparmor/ │ │              │
└────────┘ └─────────┘ └──────────┘ └──────────────┘
    │            │             │               │
    └────────────┴─────────────┴───────────────┘
                 │  all must return 0 to allow
                 ▼
         Action allowed or denied

Kernel Objects with Security Blobs:
  struct inode     → inode->i_security   (per-LSM blob)
  struct file      → file->f_security
  struct cred      → cred->security
  struct sock      → sk->sk_security
  struct super_block → sb->s_security
```

---

### الـ Core Abstractions

#### 1. الـ `security_hook_heads` — قائمة الـ Hooks

الكيرنل بيعرّف قائمة من الـ hook points، كل واحدة منهم `hlist_head`. كل LSM بيضيف functions لها عند startup:

```c
/* في security/security.c — مش معرّف في security.h مباشرة */
struct security_hook_heads {
    struct hlist_head binder_set_context_mgr;
    struct hlist_head file_open;
    struct hlist_head inode_permission;
    struct hlist_head task_kill;
    /* ... مئات من الـ hooks */
};
```

لما بيتعمل `security_file_open(file)`:
```c
int security_file_open(struct file *file)
{
    /* يلف على كل LSM مسجّل وينادي hook بتاعه */
    /* لو أي واحد رجع != 0، يوقف ويرجع الـ error */
    return call_int_hook(file_open, 0, file);
}
```

الـ `call_int_hook` macro بيلف على الـ `hlist` ومينادي الـ callback بتاع كل LSM بالترتيب. **First failure wins** — لو SELinux رفض، AppArmor مش هيتنادى.

---

#### 2. الـ `struct lsm_prop` — الـ Aggregated Security Identity

```c
/* في security.h */
struct lsm_prop {
    struct lsm_prop_selinux selinux;  /* u32 secid */
    struct lsm_prop_smack   smack;    /* struct smack_known* */
    struct lsm_prop_apparmor apparmor; /* struct aa_label* */
    struct lsm_prop_bpf     bpf;      /* u32 secid */
};
```

ده الـ "security identity" الموحد اللي بيمثّل object أو subject من منظور كل الـ LSMs المفعّلة في نفس الوقت. مش secid واحد زي زمان، دلوقتي كل LSM عنده الـ representation الخاصة بيه.

```
struct lsm_prop
┌──────────────────────────────────┐
│ selinux: { secid = 1234 }        │  ← SELinux SID
│ smack:   { skp = 0xffff8001... } │  ← Smack label pointer
│ apparmor:{ label = 0xffff8002... }│  ← AppArmor label pointer
│ bpf:     { secid = 0 }           │  ← BPF secid (لو مفعّل)
└──────────────────────────────────┘
```

الكيرنل بيعمل `lsmprop_init()` اللي بتعمل `memset` للـ struct كله بـ 0.
`lsmprop_is_set()` بتتحقق إن فيه أي قيمة مش صفر — يعني في identity معيّنة.

---

#### 3. الـ `struct lsm_context` — الـ Human-Readable Security Label

```c
struct lsm_context {
    char *context; /* e.g. "system_u:system_r:httpd_t:s0" */
    u32   len;
    int   id;      /* LSM_ID_SELINUX, LSM_ID_APPARMOR, etc. */
};
```

ده الـ text representation للـ security label — اللي بتشوفه لو عملت `ls -Z` في SELinux. الـ `security_lsmprop_to_secctx()` بتحوّل `lsm_prop` لـ `lsm_context`.

---

#### 4. الـ `struct lsm_ctx` — الـ Userspace API Context

```c
/* في uapi/linux/lsm.h */
struct lsm_ctx {
    __u64 id;       /* LSM_ID_SELINUX = 101 */
    __u64 flags;
    __u64 len;      /* total struct size including ctx */
    __u64 ctx_len;  /* size of ctx[] only */
    __u8  ctx[];    /* flexible array — the actual label string/binary */
};
```

ده الـ struct اللي بيتنقل بين kernel وuser space عبر الـ syscalls الجديدة `lsm_get_self_attr` / `lsm_set_self_attr`.

---

#### 5. الـ Security Blobs — Private LSM Data على كل Object

كل LSM محتاج يخزن data خاصة بيه على كل kernel object. الـ framework بيدير ده عن طريق **blob** embedded في كل object:

```
struct inode
┌─────────────────────────┐
│ i_ino, i_mode, i_uid... │
│ i_security  ────────────┼──► [SELinux inode_security_struct]
└─────────────────────────┘    [AppArmor aa_inode_ctx]
                               ← مبنيين متتالي في heap واحد
                                  بيديره LSM framework

struct cred
┌─────────────────────────┐
│ uid, gid, cap_effective │
│ security ───────────────┼──► [SELinux task_security_struct]
└─────────────────────────┘    [AppArmor task_ctx]
```

الـ `security_inode_alloc(inode)` بتخصص الـ blob.
الـ `security_inode_free(inode)` بتحرره.
كل LSM بيسجّل الـ blob size بتاعه عند init — الـ framework بيوزّع shared buffer واحد.

---

### تصنيف الـ Hooks حسب المجال

الـ LSM framework بيغطي كل الـ subsystems:

| المجال | أمثلة على الـ Hooks |
|--------|---------------------|
| **Filesystem** | `security_inode_permission`, `security_inode_create`, `security_file_open`, `security_sb_mount` |
| **Task/Process** | `security_task_kill`, `security_task_setnice`, `security_cred_prepare`, `security_bprm_check` |
| **Network** | `security_socket_connect`, `security_sock_rcv_skb`, `security_inet_conn_request` |
| **IPC** | `security_ipc_permission`, `security_msg_queue_msgsnd`, `security_shm_shmat` |
| **Kernel Integrity** | `security_kernel_load_data`, `security_kernel_read_file`, `security_locked_down` |
| **Keys** | `security_key_alloc`, `security_key_permission` |
| **BPF** | `security_bpf`, `security_bpf_prog_load`, `security_bpf_map_create` |
| **Audit** | `security_audit_rule_match`, `security_audit_rule_init` |

---

### Pre-Hook vs Post-Hook Pattern

الـ framework عنده pattern مهم: بعض العمليات ليها hook قبل وبعد:

```c
/* قبل العملية — قرار السماح أو الرفض */
int security_inode_setattr(...);     /* return 0 or -EPERM */

/* بعد العملية — update security state */
void security_inode_post_setattr(...); /* void, لا ترجع error */
```

الـ post hooks موجودة عشان LSM يقدر يحدّث الـ state الداخلي بتاعه بعد ما العملية نجحت فعلاً — زي تحديث الـ audit trail أو إعادة حساب الـ label.

---

### الـ Lockdown Mechanism

```c
enum lockdown_reason {
    LOCKDOWN_NONE,
    LOCKDOWN_MODULE_SIGNATURE,   /* منع load unsigned modules */
    LOCKDOWN_DEV_MEM,            /* منع /dev/mem access */
    LOCKDOWN_KEXEC,              /* منع kexec */
    LOCKDOWN_DEBUGFS,            /* منع debugfs */
    LOCKDOWN_BPF_WRITE_USER,     /* منع BPF من write لـ user mem */
    /* ... */
    LOCKDOWN_INTEGRITY_MAX,      /* كل اللي فوق = حماية integrity */
    LOCKDOWN_KCORE,              /* منع قراءة /proc/kcore */
    LOCKDOWN_KPROBES,            /* منع kprobes */
    LOCKDOWN_BPF_READ_KERNEL,    /* منع BPF من قراءة kernel mem */
    /* ... */
    LOCKDOWN_CONFIDENTIALITY_MAX,/* كل اللي فوق = حماية confidentiality */
};
```

فيه تقسيم واضح:
- **Integrity lockdown**: حاجات تخلي userland يعدّل kernel code.
- **Confidentiality lockdown**: حاجات تخلي userland يقرأ data سرية من الكيرنل.

`security_locked_down(LOCKDOWN_DEBUGFS)` بيتنادى قبل أي access لـ debugfs لما يكون النظام في secure boot mode.

---

### مقارنة ما يملكه الـ Framework مقابل ما يفوّضه للـ LSM

| الـ LSM Framework يملك | الـ LSM Implementation تفوّض إليها |
|------------------------|------------------------------------|
| **تحديد أماكن الـ hooks** في الكيرنل | **منطق الـ policy** (مين يوصل لإيه) |
| **ترتيب استدعاء الـ LSMs** (stacking) | **تعريف الـ security labels** والمعناها |
| **إدارة الـ blobs** (alloc/free/layout) | **خزن الـ state** على كل object |
| **الواجهة مع userspace** (procattr, selfattr syscalls) | **تحميل وتفسير الـ policy** |
| **الـ default fallback** (capabilities) | **الـ audit messages** التفصيلية |
| **تسجيل الـ LSMs** وترتيبها | **اتخاذ القرار** الفعلي في كل hook |

---

### الـ Stacking — تشغيل أكتر من LSM

من kernel 4.x و بشكل كامل في kernel 5.x+، ممكن تشغّل أكتر من LSM في نفس الوقت:

```
CONFIG_LSM="lockdown,yama,selinux,bpf"
```

الترتيب مهم — الـ framework بينادي بالترتيب ده. **أول واحد يرجع error يوقف السلسلة**.

```
security_file_open(file)
        │
        ▼
   lockdown check  →  0 (pass)
        │
        ▼
   yama check      →  0 (pass)
        │
        ▼
   selinux check   →  -EACCES ← STOP, access denied
        │
   apparmor check  ← NEVER REACHED
```

---

### الـ `struct dm_verity_digest` والـ Integrity LSMs

```c
struct dm_verity_digest {
    const char *alg;        /* e.g. "sha256" */
    const u8   *digest;     /* hash bytes */
    size_t      digest_len;
};

enum lsm_integrity_type {
    LSM_INT_DMVERITY_SIG_VALID,      /* dm-verity signature valid */
    LSM_INT_DMVERITY_ROOTHASH,       /* dm-verity root hash */
    LSM_INT_FSVERITY_BUILTINSIG_VALID, /* fs-verity signature */
};
```

ده للـ integration مع الـ **IMA (Integrity Measurement Architecture)** و**EVM** subsystems — اللي هما LSMs متخصصة في verification إن الـ files مش اتعدّل:
- **IMA**: بيقيس ويحتفظ بـ hash لكل file يتقرأ.
- **EVM**: بيحمي الـ extended attributes من الـ tampering.
- **dm-verity**: block-level integrity للـ read-only partitions (زي Android system partition).

`security_bdev_setintegrity()` بيخلي الـ block device يخبر الـ LSMs بـ integrity info بتاعه — مثلاً لما يـ mount partition محميّ بـ dm-verity.

---

### مسار حياة الـ Security Decision — مثال كامل

```
Process A يعمل open("/etc/shadow", O_RDONLY)
                │
                ▼
    sys_openat → do_filp_open → vfs_open
                │
                ▼
    security_inode_permission(inode, MAY_READ)
                │
     ┌──────────┴──────────────────────┐
     │                                 │
     ▼                                 ▼
DAC check (Unix perm)          SELinux avc_has_perm()
uid=1000, inode owner=0           subject: user_t
mode=640, no read for other        object:  shadow_t
     │                             operation: read
     ▼                                 │
 return -EACCES              policy says: DENY
                                         │
                                         ▼
                                   return -EACCES
                │
                ▼
        open() fails → EACCES returned to userspace
        audit: avc: denied { read } for pid=1234 ...
```

---

### الـ securityfs — /sys/kernel/security/

الـ framework بيوفر filesystem خاص للـ LSMs تعرض interfaces بتاعتها:

```bash
/sys/kernel/security/
├── selinux/
│   ├── enforce          # 0=permissive, 1=enforcing
│   ├── policy           # binary policy
│   └── load             # load new policy
├── apparmor/
│   ├── profiles         # list of loaded profiles
│   └── .load            # load new profile
└── ima/
    └── policy           # IMA measurement policy
```

`securityfs_create_file()` و`securityfs_create_dir()` هي الـ API اللي كل LSM بيستخدمها يعمل entries هنا.

---

### ملخص العلاقات بين الـ Structs

```
                    ┌──────────────────┐
                    │   struct cred    │
                    │  uid, gid, caps  │
                    │  security ───────┼──► LSM blob
                    └────────┬─────────┘
                             │ attached to
                    ┌────────▼─────────┐
                    │ struct task_struct│
                    │  cred, comm, pid │
                    └────────┬─────────┘
                             │ does
                    ┌────────▼─────────┐
                    │  security_*()    │  ← LSM hook call
                    │  framework API   │
                    └────────┬─────────┘
                             │ queries
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
       ┌────────────┐ ┌────────────┐ ┌──────────────┐
       │lsm_prop    │ │lsm_context │ │  lsm_ctx     │
       │(kernel     │ │(text label │ │  (userspace  │
       │ aggregate) │ │ string)    │ │   API blob)  │
       └────────────┘ └────────────┘ └──────────────┘
       │selinux.secid│ │"httpd_t:s0"│ │id=101(SELNX) │
       │smack.skp   │ │            │ │ctx[]="httpd.."│
       │apparmor.lbl│ │            │ │              │
       └────────────┘ └────────────┘ └──────────────┘
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — الـ Flags والـ Enums والـ Config Options

#### CAP_OPT — خيارات دالة `capable`

| Macro | قيمة | المعنى |
|---|---|---|
| `CAP_OPT_NONE` | `0x0` | بدون خيارات |
| `CAP_OPT_NOAUDIT` | `BIT(1)` | ما تعمل audit للطلب |
| `CAP_OPT_INSETID` | `BIT(2)` | الاستدعاء جاي من دالة setid |

#### LSM_SETID — flags لـ `task_security_ops`

| Macro | قيمة | المعنى |
|---|---|---|
| `LSM_SETID_ID` | `1` | setuid/setgid — id0 هو uid أو gid |
| `LSM_SETID_RE` | `2` | setreuid/setregid — id0=real, id1=eff |
| `LSM_SETID_RES` | `4` | setresuid/setresgid — id0=real, id1=eff, id2=saved |
| `LSM_SETID_FS` | `8` | setfsuid/setfsgid |

#### LSM_PRLIMIT — flags لـ `security_task_prlimit`

| Macro | قيمة | المعنى |
|---|---|---|
| `LSM_PRLIMIT_READ` | `1` | عملية قراءة الـ limit |
| `LSM_PRLIMIT_WRITE` | `2` | عملية كتابة الـ limit |

#### LSM_UNSAFE — أسباب خطورة الـ bprm

| Macro | قيمة | المعنى |
|---|---|---|
| `LSM_UNSAFE_SHARE` | `1` | shared VM |
| `LSM_UNSAFE_PTRACE` | `2` | tracee under ptrace |
| `LSM_UNSAFE_NO_NEW_PRIVS` | `4` | no_new_privs مفعّل |

#### LSM_FLAG

| Macro | قيمة | المعنى |
|---|---|---|
| `LSM_FLAG_SINGLE` | `0x0001` | الطلب لـ LSM واحد بس |
| `SECURITY_LSM_NATIVE_LABELS` | `1` | superblock له labels أصلية |

#### LSM_ID — معرّفات الـ LSMs

| Macro | قيمة | الـ LSM |
|---|---|---|
| `LSM_ID_UNDEF` | 0 | غير معرّف |
| `LSM_ID_CAPABILITY` | 100 | Capabilities |
| `LSM_ID_SELINUX` | 101 | SELinux |
| `LSM_ID_SMACK` | 102 | Smack |
| `LSM_ID_TOMOYO` | 103 | TOMOYO |
| `LSM_ID_APPARMOR` | 104 | AppArmor |
| `LSM_ID_YAMA` | 105 | Yama |
| `LSM_ID_LOADPIN` | 106 | LoadPin |
| `LSM_ID_SAFESETID` | 107 | SafeSetID |
| `LSM_ID_LOCKDOWN` | 108 | Lockdown |
| `LSM_ID_BPF` | 109 | BPF-LSM |
| `LSM_ID_LANDLOCK` | 110 | Landlock |
| `LSM_ID_IMA` | 111 | IMA |
| `LSM_ID_EVM` | 112 | EVM |
| `LSM_ID_IPE` | 113 | IPE |

#### LSM_ATTR — سمات الـ LSM API لـ userspace

| Macro | قيمة | المعنى |
|---|---|---|
| `LSM_ATTR_UNDEF` | 0 | غير معرّف |
| `LSM_ATTR_CURRENT` | 100 | السياق الحالي |
| `LSM_ATTR_EXEC` | 101 | سياق exec |
| `LSM_ATTR_FSCREATE` | 102 | سياق إنشاء ملفات |
| `LSM_ATTR_KEYCREATE` | 103 | سياق إنشاء مفاتيح |
| `LSM_ATTR_PREV` | 104 | السياق السابق |
| `LSM_ATTR_SOCKCREATE` | 105 | سياق إنشاء sockets |

#### `enum lockdown_reason` — cheatsheet كامل

| القيمة | النوع | المعنى |
|---|---|---|
| `LOCKDOWN_NONE` | — | مافيش lockdown |
| `LOCKDOWN_MODULE_SIGNATURE` | integrity | توقيع الـ modules |
| `LOCKDOWN_DEV_MEM` | integrity | الوصول لـ /dev/mem |
| `LOCKDOWN_EFI_TEST` | integrity | EFI test interface |
| `LOCKDOWN_KEXEC` | integrity | kexec |
| `LOCKDOWN_HIBERNATION` | integrity | hibernate |
| `LOCKDOWN_PCI_ACCESS` | integrity | PCI BAR |
| `LOCKDOWN_IOPORT` | integrity | I/O port |
| `LOCKDOWN_MSR` | integrity | MSR registers |
| `LOCKDOWN_ACPI_TABLES` | integrity | ACPI custom tables |
| `LOCKDOWN_DEVICE_TREE` | integrity | device tree overlay |
| `LOCKDOWN_PCMCIA_CIS` | integrity | PCMCIA CIS |
| `LOCKDOWN_TIOCSSERIAL` | integrity | serial port settings |
| `LOCKDOWN_MODULE_PARAMETERS` | integrity | module params |
| `LOCKDOWN_MMIOTRACE` | integrity | MMIO tracing |
| `LOCKDOWN_DEBUGFS` | integrity | debugfs |
| `LOCKDOWN_XMON_WR` | integrity | xmon write |
| `LOCKDOWN_BPF_WRITE_USER` | integrity | BPF كتابة userspace |
| `LOCKDOWN_DBG_WRITE_KERNEL` | integrity | debugger kernel write |
| `LOCKDOWN_RTAS_ERROR_INJECTION` | integrity | RTAS error injection |
| `LOCKDOWN_INTEGRITY_MAX` | — | حد integrity |
| `LOCKDOWN_KCORE` | confidentiality | /proc/kcore |
| `LOCKDOWN_KPROBES` | confidentiality | kprobes |
| `LOCKDOWN_BPF_READ_KERNEL` | confidentiality | BPF قراءة kernel |
| `LOCKDOWN_DBG_READ_KERNEL` | confidentiality | debugger kernel read |
| `LOCKDOWN_PERF` | confidentiality | perf events |
| `LOCKDOWN_TRACEFS` | confidentiality | tracefs |
| `LOCKDOWN_XMON_RW` | confidentiality | xmon read/write |
| `LOCKDOWN_XFRM_SECRET` | confidentiality | XFRM secrets |
| `LOCKDOWN_CONFIDENTIALITY_MAX` | — | حد confidentiality |

#### `enum lsm_event`

| القيمة | المعنى |
|---|---|
| `LSM_POLICY_CHANGE` | تغيّر الـ policy |
| `LSM_STARTED_ALL` | كل الـ LSMs اشتغلت |

#### `enum lsm_integrity_type`

| القيمة | المعنى |
|---|---|
| `LSM_INT_DMVERITY_SIG_VALID` | توقيع dm-verity صالح |
| `LSM_INT_DMVERITY_ROOTHASH` | root hash لـ dm-verity |
| `LSM_INT_FSVERITY_BUILTINSIG_VALID` | توقيع fs-verity المدمج صالح |

#### `enum kernel_load_data_id` — أنواع البيانات اللي الكيرنل بيحمّلها

**الـ** enum اتبنى تلقائياً من `__kernel_read_file_id` — بيغطي: `UNKNOWN`, `FIRMWARE`, `MODULE`, `KEXEC_IMAGE`, `KEXEC_INITRAMFS`, `POLICY`, `X509_CERTIFICATE`, وغيرها حسب kernel version.

#### Config Options الأساسية

| Config | تأثيره |
|---|---|
| `CONFIG_SECURITY` | يفعّل نظام LSM كله |
| `CONFIG_SECURITY_NETWORK` | hooks الشبكة |
| `CONFIG_SECURITY_NETWORK_XFRM` | hooks الـ XFRM/IPsec |
| `CONFIG_SECURITY_PATH` | hooks عمليات path |
| `CONFIG_SECURITY_INFINIBAND` | hooks الـ InfiniBand |
| `CONFIG_SECURITY_SELINUX` | يفعّل SELinux |
| `CONFIG_SECURITY_SMACK` | يفعّل Smack |
| `CONFIG_SECURITY_APPARMOR` | يفعّل AppArmor |
| `CONFIG_BPF_LSM` | يفعّل BPF-LSM |
| `CONFIG_MMU` | يفعّل `mmap_min_addr` |
| `CONFIG_KEYS` | hooks الـ keyring |
| `CONFIG_AUDIT` | hooks الـ audit rules |
| `CONFIG_SECURITYFS` | filesystem `/sys/kernel/security` |
| `CONFIG_BPF_SYSCALL` | hooks الـ BPF |
| `CONFIG_PERF_EVENTS` | hooks الـ perf |
| `CONFIG_IO_URING` | hooks الـ io_uring |
| `CONFIG_WATCH_QUEUE` | hooks الـ watch notifications |

---

### 1. الـ Structs المهمة

#### `struct lsm_ctx` — (من `uapi/linux/lsm.h`)

**الغرض:** حاوية LSM context بتُرسل لـ userspace عبر syscalls زي `getselfattr` / `setselfattr`.

```c
struct lsm_ctx {
    __u64 id;        /* LSM_ID_XXX — رقم الـ LSM */
    __u64 flags;     /* flags خاصة بالـ LSM */
    __u64 len;       /* الحجم الكلي للـ struct + البادينج + ctx */
    __u64 ctx_len;   /* حجم ctx بالضبط */
    __u8  ctx[];     /* قيمة الـ context (string أو binary) */
};
```

| Field | المعنى |
|---|---|
| `id` | يحدد الـ LSM المسؤول (SELinux=101, AppArmor=104...) |
| `flags` | LSM-specific، صفر لو مش مستخدم |
| `len` | `sizeof(lsm_ctx) + padding + ctx_len` |
| `ctx_len` | حجم `ctx` بالبايت |
| `ctx` | الـ context الفعلي — string مـnull-terminated أو binary |

**الإشكالية:** `len` لازم تشمل أي padding جاي بعد `ctx`، غلطة شائعة عند التطبيق.

---

#### `struct lsm_context` — (من `security.h` نفسه)

**الغرض:** تمثيل داخلي للـ security context جوّه الكيرنل — البديل الداخلي لـ `lsm_ctx`.

```c
struct lsm_context {
    char *context;  /* الـ string المقدّم من الـ LSM module */
    u32   len;      /* طول الـ string */
    int   id;       /* الـ LSM المسؤول عن هذا الـ context */
};
```

| Field | المعنى |
|---|---|
| `context` | pointer لـ string مخصّص من الـ LSM نفسه |
| `len` | طول الـ string بالبايت |
| `id` | `LSM_ID_XXX` — يحدد من أنشأ هذا الـ context |

**المستخدم في:** `security_secid_to_secctx()`, `security_lsmprop_to_secctx()`, `security_release_secctx()`.

---

#### `struct lsm_prop` — (من `security.h`)

**الغرض:** الـ union الموحّد اللي بيحمل الـ security properties لكل LSM في نفس الوقت — بديل `secid` القديم.

```c
struct lsm_prop {
    struct lsm_prop_selinux  selinux;   /* u32 secid */
    struct lsm_prop_smack    smack;     /* struct smack_known* skp */
    struct lsm_prop_apparmor apparmor;  /* struct aa_label* label */
    struct lsm_prop_bpf      bpf;       /* u32 secid */
};
```

| Field | النوع الحقيقي (لو config مفعّل) | المعنى |
|---|---|---|
| `selinux.secid` | `u32` | رقم الـ SID في SELinux |
| `smack.skp` | `struct smack_known *` | pointer لـ label entry في قائمة Smack |
| `apparmor.label` | `struct aa_label *` | pointer لـ AppArmor label |
| `bpf.secid` | `u32` | رقم الـ secid في BPF-LSM |

**API المرتبط:**
- `lsmprop_init(prop)` — يعمل `memset` بصفر
- `lsmprop_is_set(prop)` — بيتحقق إن فيه قيمة مش صفر

---

#### `struct dm_verity_digest` — (من `security.h`)

**الغرض:** بيحمل معلومات الـ digest الخاص بـ dm-verity لما الـ LSM يحتاج يتحقق من integrity الـ block device.

```c
struct dm_verity_digest {
    const char *alg;        /* اسم الـ algorithm (مثلاً "sha256") */
    const u8   *digest;     /* bytes الـ hash */
    size_t      digest_len; /* طول الـ digest بالبايت */
};
```

**المستخدم في:** `security_bdev_setintegrity()` مع `LSM_INT_DMVERITY_ROOTHASH`.

---

### 2. رسم علاقات الـ Structs

```
                    ┌─────────────────────────────────────┐
                    │           struct lsm_prop            │
                    │  ┌──────────────────────────────┐   │
                    │  │ lsm_prop_selinux: u32 secid  │   │
                    │  │ lsm_prop_smack: *smack_known │   │
                    │  │ lsm_prop_apparmor: *aa_label │   │
                    │  │ lsm_prop_bpf: u32 secid      │   │
                    │  └──────────────────────────────┘   │
                    └──────────────────┬──────────────────┘
                                       │ يُملأ بواسطة
                    ┌──────────────────▼──────────────────┐
                    │   security_cred_getlsmprop()        │
                    │   security_inode_getlsmprop()       │
                    │   security_ipc_getlsmprop()         │
                    │   security_task_getlsmprop_obj()    │
                    └──────────────────┬──────────────────┘
                                       │ يُحوَّل بواسطة
                    ┌──────────────────▼──────────────────┐
                    │         struct lsm_context           │
                    │   char *context  (string)           │
                    │   u32   len                         │
                    │   int   id  (LSM_ID_XXX)            │
                    └──────────────────┬──────────────────┘
                                       │ يُرسل لـ userspace
                    ┌──────────────────▼──────────────────┐
                    │           struct lsm_ctx             │
                    │   __u64 id  __u64 flags             │
                    │   __u64 len __u64 ctx_len           │
                    │   __u8  ctx[]  (flexible array)     │
                    └─────────────────────────────────────┘

         ┌──────────────────────────────────────────────────┐
         │               Kernel Objects                      │
         │                                                   │
         │  struct cred ──────────────►  lsm_prop           │
         │      │                         (security blob)   │
         │      │ owns                                       │
         │  struct task_struct                               │
         │      │                                           │
         │  struct inode ─────────────►  lsm_prop           │
         │      │                         (security blob)   │
         │      │                                           │
         │  struct kern_ipc_perm ──────►  lsm_prop          │
         │      │                         (security blob)   │
         │                                                   │
         │  struct super_block ────────►  void* security    │
         │  struct file        ────────►  void* security    │
         │  struct sock        ────────►  void* security    │
         │  struct key         ────────►  void* security    │
         │  struct bpf_map     ────────►  void* security    │
         │  struct bpf_prog    ────────►  void* security    │
         │  struct perf_event  ────────►  void* security    │
         └──────────────────────────────────────────────────┘

         ┌──────────────────────────────────────────────────┐
         │          XFRM Security                           │
         │                                                   │
         │  struct xfrm_state                               │
         │      └──► struct xfrm_sec_ctx ◄──────────────┐  │
         │                   │                           │  │
         │  struct xfrm_policy ──────────────────────────┘  │
         │      │                                           │
         │  security_xfrm_policy_lookup(ctx, fl_secid)      │
         │  security_xfrm_state_pol_flow_match(x, xp, flic) │
         └──────────────────────────────────────────────────┘
```

---

### 3. Lifecycle Diagrams

#### Lifecycle الـ inode security blob

```
VFS يطلب إنشاء inode
        │
        ▼
security_inode_alloc(inode, gfp)
  → LSM يخصّص blob ويربطه بـ inode->i_security
        │
        ▼
security_inode_init_security(inode, dir, qstr, initxattrs, fs_data)
  → LSM يحدد اسم الـ xattr وقيمته
  → الـ callback initxattrs يكتب الـ xattr على الـ FS
        │
        ▼
[استخدام طبيعي]
  security_inode_permission(inode, mask)
  security_inode_setattr(idmap, dentry, attr)
  security_inode_getxattr / setxattr / removexattr
  security_inode_getlsmprop(inode, prop)
        │
        ▼
inode_free_by_rcu() أو eviction
  security_inode_free(inode)
  → LSM يحرّر الـ blob ويصفّر المؤشر
```

#### Lifecycle الـ task/cred security

```
fork() / clone()
        │
        ├─► security_task_alloc(task, clone_flags)
        │       → يخصّص task security blob
        │
        ├─► security_cred_alloc_blank(cred, gfp)
        │       → blank cred (بدون copy)
        │
        └─► security_prepare_creds(new, old, gfp)
                → ينسخ cred security من old لـ new
                        │
                        ▼
                security_transfer_creds(new, old)
                        │
                        ▼
                [تشغيل exec]
                security_bprm_creds_for_exec(bprm)
                security_bprm_creds_from_file(bprm, file)
                security_bprm_check(bprm)
                security_bprm_committing_creds(bprm)
                security_bprm_committed_creds(bprm)
                        │
                        ▼
                [عمل عادي — checks على cred]
                security_task_kill / setnice / setrlimit ...
                        │
                        ▼
                task_exit / cred_put
                security_task_free(task)
                security_cred_free(cred)
```

#### Lifecycle الـ socket/sk security

```
socket()
  → security_socket_create(family, type, proto, kern)
        │
        ▼
  → security_socket_post_create(sock, ...)
  → security_sk_alloc(sk, family, gfp)
        │
        ▼
bind() → security_socket_bind()
connect() → security_socket_connect()
listen() → security_socket_listen()
accept() → security_socket_accept()
             → security_inet_conn_request(sk, skb, req)
             → security_inet_csk_clone(newsk, req)
             → security_inet_conn_established(sk, skb)
             → security_sock_graft(newsk, parent)
        │
        ▼
sendmsg() → security_socket_sendmsg()
recvmsg() → security_socket_recvmsg()
recv skb → security_sock_rcv_skb(sk, skb)
        │
        ▼
close()
  → security_sk_free(sk)
```

#### Lifecycle الـ key security

```
key_alloc()
  → security_key_alloc(key, cred, flags)
        │
        ▼
key_update() أو key_create()
  → security_key_post_create_or_update(keyring, key, payload, ...)
        │
        ▼
key_permission()
  → security_key_permission(key_ref, cred, need_perm)
        │
        ▼
key_free()
  → security_key_free(key)
```

#### Lifecycle الـ BPF security

```
bpf() syscall
  → security_bpf(cmd, attr, size, kernel)
        │
        ├── BPF_MAP_CREATE
        │     → security_bpf_map_create(map, attr, token, kernel)
        │             │
        │             ▼
        │     [map operations]
        │     → security_bpf_map(map, fmode)
        │             │
        │             ▼
        │     map_free()
        │     → security_bpf_map_free(map)
        │
        └── BPF_PROG_LOAD
              → security_bpf_prog_load(prog, attr, token, kernel)
                      │
                      ▼
              [prog operations]
              → security_bpf_prog(prog)
                      │
                      ▼
              prog_free()
              → security_bpf_prog_free(prog)
```

---

### 4. Call Flow Diagrams

#### Flow: فتح ملف (open syscall)

```
userspace: open("/etc/passwd", O_RDONLY)
        │
        ▼
sys_openat()
  → do_filp_open()
      → path_openat()
          │
          ├─► security_inode_permission(inode, MAY_READ)
          │       → LSM hook: inode_permission
          │         SELinux: avc_has_perm(tsid, isid, FILE__READ)
          │         AppArmor: aa_path_perm(...)
          │         returns 0 or -EACCES
          │
          ├─► security_file_alloc(file)
          │       → يخصّص file->f_security
          │
          └─► security_file_open(file)
                  → LSM hook: file_open
                    فرصة لـ LSM يراجع الـ policy مرة ثانية
                    بعد ما الـ file struct اتبنى
                    returns 0 or -EACCES
```

#### Flow: exec (execve syscall)

```
execve("./prog", argv, envp)
        │
        ▼
do_execveat_common()
  → bprm_mm_init()  [تجهيز الـ bprm]
        │
        ▼
security_bprm_creds_for_exec(bprm)
  → LSM يجهّز creds جديدة للـ executable
  → SELinux: يحدد الـ SID الجديد من file context
        │
        ▼
security_bprm_creds_from_file(bprm, file)
  → capabilities: cap_bprm_creds_from_file
  → يعالج setuid bits وcapabilities
        │
        ▼
security_bprm_check(bprm)
  → فحص أخير على الـ binary نفسه
  → Yama: يتحقق من ptrace
        │
        ▼
[commit_creds()]
security_bprm_committing_creds(bprm)
  → LSM يعمل أي setup قبل تطبيق الـ creds
        │
        ▼
security_bprm_committed_creds(bprm)
  → LSM يكمل أي post-commit setup
  → SELinux: يرسل audit event
```

#### Flow: mount filesystem

```
mount("ext4", "/mnt", "ext4", flags, opts)
        │
        ▼
do_mount()
  → security_sb_mount(dev, path, type, flags, data)
        │
        ▼
vfs_kern_mount()
  → alloc_super()
      → security_sb_alloc(sb)
            → يخصّص sb->s_security
        │
        ▼
  → fill_super() [FS-specific]
        │
        ▼
security_sb_kern_mount(sb)
  → LSM يتحقق إن الـ sb مسموح يتـmount
        │
        ├── [أول مرة]
        │   security_sb_set_mnt_opts(sb, mnt_opts, kern_flags, ...)
        │       → SELinux: يطبّق context= من mount options
        │
        └── [remount]
            security_sb_mnt_opts_compat(sb, mnt_opts)
            security_sb_remount(sb, mnt_opts)

umount():
  security_sb_umount(mnt, flags)
  [free]
  security_sb_delete(sb)
  security_sb_free(sb)
  → يحرّر sb->s_security
```

#### Flow: IPC message queue

```
msgget() / msgsnd() / msgrcv()
        │
        ▼
[إنشاء queue]
security_msg_queue_alloc(msq)
  → يخصّص msq->security
        │
        ▼
[إرسال رسالة]
security_msg_queue_msgsnd(msq, msg, msqflg)
  → security_msg_msg_alloc(msg)   [للرسالة نفسها]
  → LSM يتحقق من السياسة
        │
        ▼
[استقبال رسالة]
security_msg_queue_msgrcv(msq, msg, target, type, mode)
  → LSM يتحقق إن target مسموحله يستقبل
        │
        ▼
[حذف queue]
security_msg_queue_free(msq)
security_msg_msg_free(msg)
```

#### Flow: network packet receive (sk_buff)

```
NIC → driver → netif_receive_skb()
        │
        ▼
ip_rcv() → ip_local_deliver()
        │
        ▼
[TCP/UDP layer]
security_sock_rcv_skb(sk, skb)
  → SELinux: selinux_sock_rcv_skb()
      → netlbl_skbuff_getattr(skb, &secattr)
      → avc_has_perm(sk_sid, skb_sid, ...)
  → returns 0 or -EACCES (drop packet)
        │
        ▼
socket_recvmsg()
  → security_socket_recvmsg(sock, msg, size, flags)
```

#### Flow: lockdown check

```
kernel subsystem يحاول عمل عملية حساسة
(مثلاً: تحميل module بدون توقيع)
        │
        ▼
security_locked_down(LOCKDOWN_MODULE_SIGNATURE)
  → لو CONFIG_SECURITY:
      يمشي على كل الـ LSMs المسجّلين
      → lockdown LSM: يرجع -EPERM لو lockdown مفعّل
  → لو !CONFIG_SECURITY:
      يرجع 0 دايماً (allow)
        │
        ▼
لو -EPERM:
  WARN أو يرفض العملية
```

---

### 5. استراتيجية الـ Locking

#### نظرة عامة

الـ `security.h` نفسه **ما يعرّفش locks** — لكن يرسم الإطار اللي الـ LSMs تشتغل فيه. الـ locking strategy موزّعة على ثلاث طبقات:

#### أ) حماية الـ security blobs

```
Object          الـ Lock المسؤول
─────────────── ──────────────────────────────────────
inode->i_security     inode->i_lock  (spinlock)
                       أو RCU لعمليات القراءة
cred->security         immutable بعد commit_creds()
                       — لا يحتاج lock للقراءة
task->security         task_lock(task) — spinlock
sb->s_security         sb->s_umount  (rwsem)
sk->sk_security        lock_sock(sk) / bh_lock_sock()
```

#### ب) حماية الـ xfrm_sec_ctx

```
struct xfrm_policy / xfrm_state
  → محمية بـ xfrm_state_lock (spinlock)
  → وbـ net->xfrm.xfrm_policy_lock (rwlock)

security_xfrm_policy_alloc()  → يحتاج xfrm_cfg_mutex
security_xfrm_policy_free()   → يحتاج RCU grace period
security_xfrm_state_alloc()   → تحت xfrm_state_lock
```

#### ج) الـ LSM notifier

```c
/* blocking notifiers — يستخدمون blocking_notifier_chain_lock */
call_blocking_lsm_notifier(event, data)
register_blocking_lsm_notifier(nb)
unregister_blocking_lsm_notifier(nb)
```

**الـ** `blocking_notifier_head` محمية بـ `rwsem` — القراءة (notifier call) ممكن تتوازى، الكتابة (register/unregister) exclusive.

#### د) ترتيب الـ Locks (Lock Ordering)

```
[Higher → يُقفل أولاً]

task_lock
    └─► inode->i_lock
            └─► sk->sk_lock (bh_lock_sock)
                    └─► xfrm_state_lock

[لا يجوز عكس الترتيب — deadlock]
```

#### هـ) الـ RCU في الـ LSM

- **الـ** `security_inode_follow_link()` تُستدعى ممكن بـ `rcu=true` — الـ LSM لازم يراعي هذا.
- الـ `cred` بتاتشتغل بـ RCU — `rcu_dereference()` للوصول للـ current cred.
- الـ `lsm_prop` بيتنسخ بالقيمة (stack allocation) — ما يحتاج lock أثناء القراءة بعد الـ copy.

#### و) immutability وthread safety

```c
/* lsm_prop إيمّيوتابل بعد ما يتحوّل لـ lsm_context */
/* لازم تستخدم security_release_secctx() لتحرير الـ context */

void security_release_secctx(struct lsm_context *cp);
/* يحرّر cp->context المخصّص من الـ LSM */
/* يصفّر الـ struct بعدها */
```

#### ز) جدول ملخّص

| Operation | الـ Lock المطلوب | نوعه |
|---|---|---|
| قراءة `inode->i_security` | `rcu_read_lock()` | RCU |
| تعديل `inode->i_security` | `inode->i_lock` | spinlock |
| قراءة `cred->security` | بدون lock (immutable) | — |
| تعديل `sk->sk_security` | `lock_sock(sk)` | mutex-like |
| `xfrm_sec_ctx` alloc/free | `xfrm_cfg_mutex` | mutex |
| LSM notifier register | `rwsem` (write) | rwsem |
| LSM notifier call | `rwsem` (read) | rwsem |
| `mmap_min_addr` | `sysctl_lock` | spinlock |
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet لكل الـ Functions المهمة

#### Group 1: Initialization & Core Infrastructure

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `security_init` | `int security_init(void)` | تهيئة الـ LSM framework بعد الـ boot |
| `early_security_init` | `int early_security_init(void)` | تهيئة مبكرة قبل الـ init thread |
| `lsmprop_init` | `void lsmprop_init(struct lsm_prop*)` | صفّر struct lsm_prop |
| `lsmprop_is_set` | `bool lsmprop_is_set(struct lsm_prop*)` | اتحقق لو فيه secid مش صفر |
| `lsm_name_to_attr` | `u64 lsm_name_to_attr(const char*)` | حوّل اسم LSM لـ attr ID |
| `lsm_fill_user_ctx` | `int lsm_fill_user_ctx(...)` | ابنِ lsm_ctx في userspace buffer |
| `kernel_load_data_id_str` | `const char* kernel_load_data_id_str(enum)` | اجيب string اسم الـ data ID |

#### Group 2: Notifier & Blocking Hooks

| Function | الغرض |
|---|---|
| `call_blocking_lsm_notifier` | أطلق blocking notifier chain لحدث LSM |
| `register_blocking_lsm_notifier` | سجّل notifier block في الـ chain |
| `unregister_blocking_lsm_notifier` | ألغِ تسجيل notifier block |

#### Group 3: Capabilities (commoncap.c)

| Function | الغرض |
|---|---|
| `cap_capable` | تحقق من capability بدون LSM |
| `cap_capget` | اقرأ capabilities من task |
| `cap_capset` | اكتب capabilities على cred |
| `cap_bprm_creds_from_file` | طبّق setuid/setgid من الـ executable |
| `cap_mmap_addr` | تحقق من mmap_min_addr |
| `cap_task_fix_setuid` | صحّح capabilities بعد setuid |
| `cap_task_prctl` | معالجة PR_SET/GET_SECUREBITS وغيره |
| `cap_vm_enough_memory` | تحقق من حد الـ memory عبر capabilities |
| `cap_inode_setxattr` | تحقق من caps قبل setxattr |
| `cap_inode_removexattr` | تحقق من caps قبل removexattr |
| `cap_inode_need_killpriv` | تحقق لو lazim نمسح setuid bit |
| `cap_inode_killpriv` | امسح setuid/setgid بعد write |

#### Group 4: Binder IPC Security

| Function | الغرض |
|---|---|
| `security_binder_set_context_mgr` | authorize context manager |
| `security_binder_transaction` | تحقق من transaction بين processes |
| `security_binder_transfer_binder` | تحقق من nesting transfer |
| `security_binder_transfer_file` | تحقق من file descriptor transfer |

#### Group 5: Credentials & Task Security

| Function | الغرض |
|---|---|
| `security_task_alloc` | alloc security blob لـ task جديد |
| `security_task_free` | حرّر security blob للـ task |
| `security_cred_alloc_blank` | alloc فاضي لـ cred |
| `security_cred_free` | حرّر security blob للـ cred |
| `security_prepare_creds` | انسخ security label من old لـ new cred |
| `security_transfer_creds` | انقل security state بين creds |
| `security_cred_getlsmprop` | اجيب lsm_prop من cred |
| `security_kernel_act_as` | غيّر security label للـ kernel thread |
| `security_kernel_create_files_as` | خلّي kernel ينشئ ملفات بـ label معينة |

#### Group 6: Process Capabilities & Privileges

| Function | الغرض |
|---|---|
| `security_capable` | الـ main hook لـ capability check |
| `security_capget` | قرأ capabilities من task |
| `security_capset` | كتابة capabilities |
| `security_ptrace_access_check` | تحقق من ptrace permission |
| `security_task_fix_setuid/gid/groups` | LSM validation عند setuid/setgid |
| `security_task_setpgid/getpgid` | تحقق من process group operations |
| `security_task_kill` | تحقق من إرسال signal |
| `security_task_prctl` | hook على prctl syscall |
| `security_task_prlimit` | تحقق من read/write resource limits |
| `security_create_user_ns` | تحقق من إنشاء user namespace |

#### Group 7: Filesystem & Superblock Security

| Function | الغرض |
|---|---|
| `security_sb_alloc/free/delete` | lifetime management لـ superblock blob |
| `security_sb_mount` | تحقق عند mount |
| `security_sb_umount` | تحقق عند umount |
| `security_sb_remount` | تحقق عند remount |
| `security_sb_kern_mount` | تحقق عند kernel-initiated mount |
| `security_sb_set_mnt_opts` | طبّق mount options على superblock |
| `security_sb_clone_mnt_opts` | انسخ mount options بين superblocks |
| `security_sb_eat_lsm_opts` | استخرج LSM options من mount string |
| `security_sb_pivotroot` | تحقق من pivot_root |
| `security_fs_context_submount` | تحقق من submount context |
| `security_move_mount` | تحقق من move_mount |

#### Group 8: Inode Security

| Function | الغرض |
|---|---|
| `security_inode_alloc/free` | lifetime management لـ inode blob |
| `security_inode_init_security` | init xattr label لـ inode جديد |
| `security_inode_init_security_anon` | init security لـ anonymous inode |
| `security_inode_create/link/unlink/symlink/mkdir/rmdir/mknod/rename` | pre-operation permission checks |
| `security_inode_permission` | تحقق من read/write/exec permission |
| `security_inode_setattr/getattr` | تحقق من inode attribute operations |
| `security_inode_setxattr/getxattr/removexattr` | تحقق من xattr operations |
| `security_inode_set_acl/get_acl/remove_acl` | تحقق من POSIX ACL operations |
| `security_inode_getsecurity/setsecurity` | اقرأ/اكتب security xattr |
| `security_inode_getlsmprop` | اجيب lsm_prop من inode |
| `security_inode_need_killpriv/killpriv` | إدارة setuid stripping |
| `security_inode_copy_up/copy_up_xattr` | overlayfs copy-up security |
| `security_inode_setintegrity` | اكتب dm-verity/fsverity integrity data |
| `security_inode_getsecctx/setsecctx` | ادارة security context strings |
| `security_inode_invalidate_secctx` | invalidate cached security context |
| `security_d_instantiate` | hook لما dentry بيتربط بـ inode |
| `security_kernfs_init_security` | init kernfs node security |

#### Group 9: File Security

| Function | الغرض |
|---|---|
| `security_file_alloc/free/release` | lifetime management لـ file blob |
| `security_file_permission` | تحقق من read/write على file |
| `security_file_open` | hook عند open |
| `security_file_post_open` | post-open hook |
| `security_file_truncate` | تحقق عند truncate |
| `security_file_ioctl/ioctl_compat` | تحقق من ioctl commands |
| `security_mmap_file` | تحقق من mmap على file |
| `security_mmap_addr` | تحقق من low address mmap |
| `security_file_mprotect` | تحقق من mprotect |
| `security_file_lock` | تحقق من file locking |
| `security_file_fcntl` | تحقق من fcntl |
| `security_file_set_fowner` | set security عند تغيير file owner |
| `security_file_send_sigiotask` | تحقق من SIGIO delivery |
| `security_file_receive` | تحقق عند passing file via SCM_RIGHTS |

#### Group 10: Kernel Module & Data Loading

| Function | الغرض |
|---|---|
| `security_kernel_module_request` | تحقق من طلب load kernel module |
| `security_kernel_load_data` | تحقق قبل load data blob |
| `security_kernel_post_load_data` | تحقق بعد load data blob |
| `security_kernel_read_file` | تحقق قبل read_file للـ kernel |
| `security_kernel_post_read_file` | تحقق بعد read_file للـ kernel |

#### Group 11: IPC Security (SysV)

| Function | الغرض |
|---|---|
| `security_ipc_permission` | general IPC permission check |
| `security_ipc_getlsmprop` | اجيب lsm_prop من IPC object |
| `security_msg_msg_alloc/free` | lifetime لـ message |
| `security_msg_queue_alloc/free` | lifetime لـ message queue |
| `security_msg_queue_associate/msgctl/msgsnd/msgrcv` | msg queue operation checks |
| `security_shm_alloc/free` | lifetime لـ shared memory |
| `security_shm_associate/shmctl/shmat` | shm operation checks |
| `security_sem_alloc/free` | lifetime لـ semaphore |
| `security_sem_associate/semctl/semop` | semaphore operation checks |

#### Group 12: Security Context Translation

| Function | الغرض |
|---|---|
| `security_secid_to_secctx` | حوّل u32 secid لـ string context |
| `security_lsmprop_to_secctx` | حوّل lsm_prop لـ string context |
| `security_secctx_to_secid` | حوّل string context لـ u32 secid |
| `security_release_secctx` | حرّر lsm_context string |
| `security_getselfattr/setselfattr` | قرأ/اكتب task security attrs |
| `security_getprocattr/setprocattr` | procfs-based security attribute access |
| `security_ismaclabel` | تحقق لو اسم xattr هو MAC label |

#### Group 13: Network Security (`CONFIG_SECURITY_NETWORK`)

| Function | الغرض |
|---|---|
| `security_socket_create/post_create` | تحقق من socket creation |
| `security_socket_bind/connect/listen/accept` | تحقق من socket operations |
| `security_socket_sendmsg/recvmsg` | تحقق من data transfer |
| `security_sock_rcv_skb` | تحقق من incoming packet |
| `security_sk_alloc/free/clone` | lifetime لـ sock security blob |
| `security_sk_classify_flow` | صنّف flow بـ security label |
| `security_inet_conn_request` | تحقق من incoming connection |
| `security_secmark_relabel_packet` | relabel packet secmark |
| `security_unix_stream_connect/may_send` | Unix socket security |
| `security_netlink_send` | تحقق من netlink message |

#### Group 14: XFRM (IPSec) Security (`CONFIG_SECURITY_NETWORK_XFRM`)

| Function | الغرض |
|---|---|
| `security_xfrm_policy_alloc/clone/free/delete` | lifetime لـ XFRM policy |
| `security_xfrm_state_alloc/alloc_acquire/free/delete` | lifetime لـ XFRM state |
| `security_xfrm_policy_lookup` | تحقق من policy match |
| `security_xfrm_state_pol_flow_match` | تحقق من state/policy/flow consistency |
| `security_xfrm_decode_session` | استخرج secid من skb |
| `security_skb_classify_flow` | صنّف skb flow |

#### Group 15: Path-Based Security (`CONFIG_SECURITY_PATH`)

| Function | الغرض |
|---|---|
| `security_path_unlink/mkdir/rmdir/mknod/symlink/link/rename` | pre-operation path checks |
| `security_path_truncate` | تحقق قبل truncate via path |
| `security_path_chmod/chown/chroot` | تحقق من permission changes |
| `security_path_post_mknod` | post-mknod hook |

#### Group 16: Keys Security (`CONFIG_KEYS`)

| Function | الغرض |
|---|---|
| `security_key_alloc/free` | lifetime لـ key security blob |
| `security_key_permission` | تحقق من key access |
| `security_key_getsecurity` | اجيب security context للـ key |
| `security_key_post_create_or_update` | hook بعد create/update key |
| `security_watch_key` | تحقق من watch على key |

#### Group 17: Audit Security

| Function | الغرض |
|---|---|
| `security_audit_rule_init` | ابنِ LSM audit rule |
| `security_audit_rule_known` | تحقق لو الـ rule يستخدم LSM |
| `security_audit_rule_match` | طابق lsm_prop مع audit rule |
| `security_audit_rule_free` | حرّر audit rule |

#### Group 18: BPF Security (`CONFIG_BPF_SYSCALL`)

| Function | الغرض |
|---|---|
| `security_bpf` | تحقق من bpf() syscall |
| `security_bpf_map/bpf_prog` | تحقق من access على map/prog |
| `security_bpf_map_create` | تحقق عند إنشاء map |
| `security_bpf_prog_load` | تحقق عند load program |
| `security_bpf_token_create/cmd/capable` | BPF token security |

#### Group 19: Misc Subsystems

| Function | الغرض |
|---|---|
| `security_perf_event_open/alloc/free/read/write` | perf events security |
| `security_uring_override_creds/sqpoll/cmd/allowed` | io_uring security |
| `security_ib_pkey_access/endport_manage_subnet` | InfiniBand security |
| `security_ib_alloc/free_security` | IB security blob lifetime |
| `security_locked_down` | تحقق من lockdown level |
| `security_initramfs_populated` | hook بعد mount initramfs |
| `securityfs_create_file/dir/symlink/remove` | securityfs VFS helpers |
| `security_bdev_alloc/free/setintegrity` | block device security |
| `security_post_notification` | watch_queue notification security |
| `mmap_min_addr_handler` | sysctl handler لـ mmap_min_addr |

---

### Category 1: Initialization & Core Infrastructure

هذه الـ functions هي نقطة دخول الـ LSM framework كله. بتتشغل في أوائل الـ boot وبتثبّت الـ hooks وبتهيئ الـ security blobs.

---

#### `security_init`

```c
extern int security_init(void);
```

بتعمل initialize للـ LSM framework الكامل — بتسجّل الـ LSMs المبنية في الكيرنل وبتخصص الـ security blobs. بتتسمّى من `start_kernel()` بعد ما يخلص setup الـ memory allocator. لو فشلت، الـ kernel يـ panic.

- **Return:** `0` دايمًا في الـ CONFIG_SECURITY path (failure = panic).
- **Caller:** `start_kernel()` في `init/main.c`.

---

#### `early_security_init`

```c
extern int early_security_init(void);
```

تهيئة مبكرة جدًا — بتتسمّى قبل `security_init` وقبل الـ memory allocator يكون جاهز خالص. بتسجّل الـ LSMs اللي محتاجة تشتغل قبل الـ init process.

- **Return:** `0` عند النجاح.
- **Caller:** `start_kernel()` قبل `setup_arch()`.

---

#### `lsmprop_init`

```c
static inline void lsmprop_init(struct lsm_prop *prop)
{
    memset(prop, 0, sizeof(*prop));
}
```

مجرد `memset` على `struct lsm_prop` — بتصفّر كل الـ secids لكل الـ LSMs (SELinux, Smack, AppArmor, BPF). لازم تتسمى قبل أي استخدام للـ struct.

- **Parameters:**
  - `prop`: pointer لـ struct اللي هيتصفّر.
- **Return:** void.
- **Key:** inline، بدون overhead.

---

#### `lsmprop_is_set`

```c
static inline bool lsmprop_is_set(struct lsm_prop *prop)
```

بتقارن `prop` مع `struct lsm_prop` صفري باستخدام `memcmp`. بتشيل لو في أي LSM حاطت secid غير صفر.

- **Return:** `true` لو في value مضبوطة، `false` لو كله صفر.
- **Usage:** في الـ network code لتحديد لو الـ packet عنده label.

---

#### `lsm_name_to_attr`

```c
extern u64 lsm_name_to_attr(const char *name);
```

بتحوّل اسم LSM (مثلاً `"selinux"`) لـ attribute ID (enum `LSM_ATTR_*`). بتستخدمها الـ userspace-facing APIs زي `getselfattr`.

- **Parameters:**
  - `name`: اسم الـ LSM.
- **Return:** الـ attribute ID، أو `LSM_ATTR_UNDEF` لو مش معروف.

---

#### `lsm_fill_user_ctx`

```c
int lsm_fill_user_ctx(struct lsm_ctx __user *uctx, u32 *uctx_len,
                      void *val, size_t val_len, u64 id, u64 flags);
```

بتبني `struct lsm_ctx` في userspace buffer — بتحسب الـ size المطلوبة وبتعمل `copy_to_user`. بتتعامل مع الـ alignment padding.

- **Parameters:**
  - `uctx`: userspace buffer.
  - `uctx_len`: مؤشر للـ size المتاحة، بتحدّثه بالـ size المستخدمة.
  - `val/val_len`: الـ value الحقيقية (security context string أو blob).
  - `id`: لـ LSM identifier.
  - `flags`: reserved.
- **Return:** `0` أو error code.

---

### Category 2: Notifier Chain (Blocking LSM Notifiers)

---

#### `call_blocking_lsm_notifier`

```c
int call_blocking_lsm_notifier(enum lsm_event event, void *data);
```

بتطلق الـ `blocking_lsm_notifier_chain` — كل الـ registered notifiers بتتنادى بالترتيب. الـ callers ممكن يكونوا في sleepable context.

- **Parameters:**
  - `event`: `LSM_POLICY_CHANGE` أو `LSM_STARTED_ALL`.
  - `data`: event-specific data pointer.
- **Return:** notifier chain return value.
- **Locking:** sleeping allowed — not atomic context.

---

#### `register_blocking_lsm_notifier` / `unregister_blocking_lsm_notifier`

```c
int register_blocking_lsm_notifier(struct notifier_block *nb);
int unregister_blocking_lsm_notifier(struct notifier_block *nb);
```

يسجّل/يلغي تسجيل notifier block في الـ blocking LSM chain. الـ nb لازم يفضل valid طول ما هو registered.

---

### Category 3: Capabilities (commoncap.c)

الـ commoncap functions هي الـ default implementation للـ POSIX capabilities بدون LSM. كتير من الـ security_* stubs بتتكل عليها لما CONFIG_SECURITY=n.

---

#### `cap_capable`

```c
extern int cap_capable(const struct cred *cred, struct user_namespace *ns,
                       int cap, unsigned int opts);
```

بتتحقق لو `cred` عنده الـ `cap` في الـ `ns`. بتمشي في الـ user_namespace hierarchy (من child لـ root ns) وبتتحقق من الـ permitted set.

- **Parameters:**
  - `cred`: الـ credentials اللي بتتحقق منها.
  - `ns`: الـ user namespace الـ target.
  - `cap`: رقم الـ capability (مثلاً `CAP_NET_ADMIN`).
  - `opts`: `CAP_OPT_NOAUDIT`, `CAP_OPT_INSETID`.
- **Return:** `0` لو مسموح، `-EPERM` لو لأ.
- **Key:** هذه هي الـ low-level check — `security_capable` بيناديها لما CONFIG_SECURITY=n.

---

#### `cap_bprm_creds_from_file`

```c
extern int cap_bprm_creds_from_file(struct linux_binprm *bprm, const struct file *file);
```

بتطبّق الـ file capabilities (fscaps) على الـ `bprm->cred` — بتقرأ الـ `security.capability` xattr من الـ executable وبتحسب الـ new effective/permitted/inheritable sets.

- **Flow مبسّط:**
```
read security.capability xattr from file
if setuid/setgid bits set → adjust cred uid/gid
compute new caps = (file_permitted ∩ bounding) ∪ (ambient ∩ inheritable)
check no_new_privs restrictions
write into bprm->cred
```

---

#### `cap_task_fix_setuid`

```c
extern int cap_task_fix_setuid(struct cred *new, const struct cred *old, int flags);
```

بعد setuid، بتصحح الـ capabilities — لو UID تغيّر لـ non-root، بتمسح الـ permitted set. لو تغيّر لـ root، بتضيف capabilities.

- **Parameters:**
  - `flags`: `LSM_SETID_ID`, `LSM_SETID_RE`, `LSM_SETID_RES`, `LSM_SETID_FS`.
- **Caller:** `security_task_fix_setuid`.

---

### Category 4: Binder IPC Security

الـ Android Binder IPC بيستخدم هذه الـ hooks لتطبيق policy الـ MAC (في SELinux مثلاً) على الـ IPC operations.

---

#### `security_binder_set_context_mgr`

```c
int security_binder_set_context_mgr(const struct cred *mgr);
```

بتتحقق لو العملية صاحبة `mgr` مسموحلها تبقى context manager للـ Binder. في SELinux بتتحقق من `binder_set_context_mgr` permission.

---

#### `security_binder_transaction`

```c
int security_binder_transaction(const struct cred *from, const struct cred *to);
```

بتتحقق لو `from` مسموحلها تعمل Binder transaction لـ `to`. بتمنع elevation of privilege عبر Binder.

---

#### `security_binder_transfer_file`

```c
int security_binder_transfer_file(const struct cred *from,
                                  const struct cred *to, const struct file *file);
```

بتتحقق من نقل file descriptor عبر Binder. هذا critical لأنه ممكن يُستخدم لتجاوز DAC checks.

---

### Category 5: Credentials & Task Security

---

#### `security_task_alloc`

```c
int security_task_alloc(struct task_struct *task, u64 clone_flags);
```

بتخصص الـ security blob للـ task الجديد أثناء `fork`/`clone`. بتتسمى من `copy_process` بعد الـ task struct allocation.

- **Parameters:**
  - `task`: الـ task الجديد.
  - `clone_flags`: نفس الـ flags اللي اتبعتت لـ `clone()`.
- **Return:** `0` أو `-ENOMEM` لو allocation فشل.
- **Locking:** لازم تكون safe من concurrent access على parent.

---

#### `security_prepare_creds`

```c
int security_prepare_creds(struct cred *new, const struct cred *old, gfp_t gfp);
```

بتنسخ الـ security label من `old` cred لـ `new`. بتتسمى من `prepare_creds()` — كل syscall اللي بيعمل `cred modification` بيبدأ بيها.

- **Key:** بتمنع privilege escalation أثناء `execve` لو الـ LSM رفض.

---

#### `security_kernel_act_as`

```c
int security_kernel_act_as(struct cred *new, u32 secid);
```

بتخلّي kernel thread يعمل operations بـ security label مختلف (محدد بالـ `secid`). بتستخدمها الـ NFS server وبعض subsystems.

- **Caller:** مثلاً `set_security_override()` في `security/security.c`.

---

### Category 6: Process Privileges & Control

---

#### `security_capable`

```c
int security_capable(const struct cred *cred,
                     struct user_namespace *ns,
                     int cap,
                     unsigned int opts);
```

الـ main entry point للـ capability check في الكيرنل — كل `capable()` و`ns_capable()` macro بتنادي هذه. بتمرر لكل الـ registered LSMs بالترتيب.

- **Parameters:**
  - `cred`: credentials الخاصة بالـ subject.
  - `ns`: الـ user namespace context.
  - `cap`: رقم الـ capability المطلوبة.
  - `opts`: `CAP_OPT_NOAUDIT` لو مش عايز audit denial، `CAP_OPT_INSETID` لو في set-id context.
- **Return:** `0` = allowed، `-EPERM` = denied.
- **Key:** كل الـ LSMs بتشوفها — SELinux بيتحقق من permission في policy بالإضافة للـ DAC check.

---

#### `security_task_kill`

```c
int security_task_kill(struct task_struct *p, struct kernel_siginfo *info,
                       int sig, const struct cred *cred);
```

بتتحقق لو مسموح للـ sender يبعث `sig` للـ `p`. SELinux بيتحقق من `signal` permission بين الـ two processes.

- **Parameters:**
  - `p`: الـ target task.
  - `info`: signal info (source, etc.).
  - `sig`: الـ signal number.
  - `cred`: credentials الـ sender (أو `NULL` للـ kernel).
- **Caller:** `__send_signal()`, `kill_something_info()`.

---

#### `security_task_prctl`

```c
int security_task_prctl(int option, unsigned long arg2, unsigned long arg3,
                        unsigned long arg4, unsigned long arg5);
```

بتتحقق من operations الـ `prctl()` syscall. لو مفيش LSM عارف يتعامل مع الـ option، بترجع `-ENOSYS`، وساعتها `commoncap.c` بيشتغل كـ fallback.

- **Return:** `0` = handled & allowed, `-ENOSYS` = not handled, negative = denied.

---

#### `security_create_user_ns`

```c
int security_create_user_ns(const struct cred *cred);
```

بتتحقق لو العملية مسموحلها تعمل user namespace جديدة. SELinux ممكن يمنعها حسب الـ policy.

---

### Category 7: Filesystem & Superblock Security

---

#### `security_sb_alloc` / `security_sb_free` / `security_sb_delete`

```c
int security_sb_alloc(struct super_block *sb);
void security_sb_free(struct super_block *sb);
void security_sb_delete(struct super_block *sb);
```

**Lifecycle management** للـ security blob المرتبط بالـ superblock.

- `security_sb_alloc`: بتتسمى في `alloc_super()` — بتخصص `sb->s_security`.
- `security_sb_delete`: بتتسمى قبل teardown الـ filesystem — فرصة لـ cleanup الـ references.
- `security_sb_free`: بتتسمى في `destroy_super()` — بتحرر `sb->s_security`.

---

#### `security_sb_mount`

```c
int security_sb_mount(const char *dev_name, const struct path *path,
                      const char *type, unsigned long flags, void *data);
```

بتتحقق من `mount()` syscall — السؤال الكبير: هل العملية مسموحلها تعمل mount هنا؟

- **Caller:** `do_mount()` في `fs/namespace.c`.
- **SELinux:** بتتحقق من `filesystem:mount` permission.

---

#### `security_sb_set_mnt_opts`

```c
int security_sb_set_mnt_opts(struct super_block *sb,
                              void *mnt_opts,
                              unsigned long kern_flags,
                              unsigned long *set_kern_flags);
```

بتطبّق الـ parsed LSM-specific mount options على الـ superblock. مثلاً في SELinux: `context=`, `fscontext=`, `defcontext=`.

- **key detail:** `SECURITY_LSM_NATIVE_LABELS` flag بيقول للـ LSM إنه يستخدم الـ native filesystem labels بدل ما يفرض mount context.

---

#### `security_sb_eat_lsm_opts`

```c
int security_sb_eat_lsm_opts(char *options, void **mnt_opts);
```

بتمشي على mount options string وبتستخرج الـ LSM-specific options (زي `context=system_u:...`) وبتحطها في `mnt_opts`. الـ options اللي بتاخدها بتتشال من الـ string الأصلية.

---

### Category 8: Inode Security (الأكبر والأهم)

---

#### `security_inode_init_security`

```c
int security_inode_init_security(struct inode *inode, struct inode *dir,
                                 const struct qstr *qstr,
                                 initxattrs initxattrs, void *fs_data);
```

بتتسمى عند إنشاء inode جديد — بتسأل كل الـ LSMs عن الـ xattr اللي عايزين يحطوه على الـ inode الجديد، بعدين بتناديها عبر الـ callback `initxattrs`.

- **Flow:**
```
allocate xattr_array[]
for each LSM → get security xattr name & value
call initxattrs(inode, xattr_array, fs_data)  /* writes to filesystem */
free xattr_array
```
- **Caller:** الـ VFS functions زي `ext4_create()`.

---

#### `security_inode_permission`

```c
int security_inode_permission(struct inode *inode, int mask);
```

الـ hot path للـ access control — بتتسمى من كل `inode_permission()` call. الـ mask بيكون combination من `MAY_READ`, `MAY_WRITE`, `MAY_EXEC`.

- **Performance:** بتتسمى ملايين المرات في الثانية — كل LSM لازم يكون سريع هنا.
- **Locking:** bتتسمى مع RCU read lock في بعض paths.

---

#### `security_inode_setattr`

```c
int security_inode_setattr(struct mnt_idmap *idmap,
                           struct dentry *dentry, struct iattr *attr);
```

بتتحقق قبل `chown/chmod/utimes`. الـ `ia_valid` في `iattr` بيحدد exactly إيه اللي بيتغيّر.

- **Caller:** `notify_change()` في VFS.

---

#### `security_inode_setxattr`

```c
int security_inode_setxattr(struct mnt_idmap *idmap,
                            struct dentry *dentry, const char *name,
                            const void *value, size_t size, int flags);
```

بتتحقق من `setxattr()` syscall. لو الـ xattr name بتبدأ بـ `"security."` فده مهم جدًا — لأنه بيتحكم في الـ security label مباشرة.

---

#### `security_inode_getsecurity`

```c
int security_inode_getsecurity(struct mnt_idmap *idmap,
                               struct inode *inode, const char *name,
                               void **buffer, bool alloc);
```

بتجيب الـ security xattr لـ inode. لو `alloc=true` بتخصص buffer وبترجع pointer فيه، المُستدعي مسؤول عن تحريره.

- **Return:** size الـ xattr عند النجاح، error code عند الفشل.

---

#### `security_inode_copy_up`

```c
int security_inode_copy_up(struct dentry *src, struct cred **new);
```

خاصة بـ **overlayfs** — لما ملف في lower layer بيتعمله copy-up لـ upper layer. بتعمل override على الـ credentials لضمان الـ security label الصح على الـ new inode.

---

#### `security_inode_setintegrity`

```c
int security_inode_setintegrity(const struct inode *inode,
                                enum lsm_integrity_type type, const void *value,
                                size_t size);
```

بيخلي الـ LSM يخزّن integrity info زي dm-verity digest أو fsverity signature. الـ `type` بيكون واحدة من:
- `LSM_INT_DMVERITY_SIG_VALID`
- `LSM_INT_DMVERITY_ROOTHASH`
- `LSM_INT_FSVERITY_BUILTINSIG_VALID`

---

### Category 9: File Operations Security

---

#### `security_file_open`

```c
int security_file_open(struct file *file);
```

بتتسمى من `do_dentry_open()` — هذا آخر hook قبل ما الـ file يبقى accessible. الفرق عن `inode_permission`: هنا الـ file struct موجود والـ flags معروفة (O_RDONLY, O_WRONLY, etc).

- **SELinux:** بتتحقق من `file:open` permission وبتحفظ الـ permissions في `file->f_security`.

---

#### `security_mmap_file`

```c
int security_mmap_file(struct file *file, unsigned long prot,
                       unsigned long flags);
```

بتتحقق قبل `mmap()`. `prot` بيكون `PROT_READ|PROT_WRITE|PROT_EXEC`. هذا critical لـ W^X enforcement.

- **SELinux/AppArmor:** بتمنع `PROT_EXEC` على ملفات معينة.
- **Caller:** `vm_mmap_pgoff()`.

---

#### `security_mmap_addr`

```c
int security_mmap_addr(unsigned long addr);
```

بتتحقق من mmap على low addresses (أقل من `mmap_min_addr`). بتحمي من null-pointer dereference exploits.

- **No-LSM fallback:** `cap_mmap_addr` بتتحقق من `CAP_SYS_RAWIO`.

---

#### `security_file_receive`

```c
int security_file_receive(struct file *file);
```

بتتسمى لما process بتاخد file descriptor عبر Unix socket `SCM_RIGHTS`. مهمة لمنع privilege escalation عبر file descriptor passing.

---

### Category 10: Kernel Code & Data Loading

---

#### `security_kernel_read_file`

```c
int security_kernel_read_file(struct file *file, enum kernel_read_file_id id,
                              bool contents);
```

بتتسمى في بداية `kernel_read_file()` — لما الكيرنل بيقرأ ملف (kernel module, firmware, kexec image). الـ `contents=false` معناه creds check فقط، `true` معناه الـ file content جاهز للـ verification.

- **IDs المهمة:** `READING_MODULE`, `READING_FIRMWARE`, `READING_KEXEC_IMAGE`, `READING_POLICY`.

---

#### `security_kernel_post_read_file`

```c
int security_kernel_post_read_file(struct file *file, char *buf, loff_t size,
                                   enum kernel_read_file_id id);
```

بتتسمى بعد ما الكيرنل خلص يقرأ الـ file content في `buf`. IMA بتستخدمها لـ verify signature الـ file.

---

#### `security_kernel_load_data`

```c
int security_kernel_load_data(enum kernel_load_data_id id, bool contents);
```

زي `security_kernel_read_file` بس للـ data اللي مجاش من file (مثلاً firmware مجيتش عبر filesystem).

---

### Category 11: IPC Security

---

#### `security_msg_queue_msgsnd`

```c
int security_msg_queue_msgsnd(struct kern_ipc_perm *msq,
                              struct msg_msg *msg, int msqflg);
```

بتتحقق قبل إرسال message لـ queue. SELinux بتتحقق من `msgq:write` permission.

---

#### `security_shm_shmat`

```c
int security_shm_shmat(struct kern_ipc_perm *shp, char __user *shmaddr, int shmflg);
```

بتتحقق قبل attach لـ shared memory segment. بتشيل الـ `SHM_RDONLY` flag لتحديد read/write access.

---

### Category 12: Security Context Translation

---

#### `security_secid_to_secctx`

```c
int security_secid_to_secctx(u32 secid, struct lsm_context *cp);
```

بتحوّل الـ `u32 secid` (internal kernel representation) لـ human-readable string زي `"system_u:system_r:kernel_t:s0"`. الـ string بتتخزن في `cp->context` — المُستدعي لازم يناديها `security_release_secctx()` بعدين.

- **Caller:** netfilter, NFS, audit subsystem.

---

#### `security_lsmprop_to_secctx`

```c
int security_lsmprop_to_secctx(struct lsm_prop *prop, struct lsm_context *cp,
                                int lsmid);
```

زي `security_secid_to_secctx` بس بتاخد `lsm_prop` (اللي بتحتوي على secids لكل الـ LSMs) وبتحوّل الـ specific LSM المحدد بـ `lsmid`.

---

#### `security_release_secctx`

```c
void security_release_secctx(struct lsm_context *cp);
```

بتحرر الـ string اللي خصصتها `security_secid_to_secctx` أو `security_lsmprop_to_secctx`. لازم تتسمى دايمًا بعد الاستخدام.

---

#### `security_getselfattr` / `security_setselfattr`

```c
int security_getselfattr(unsigned int attr, struct lsm_ctx __user *ctx,
                         u32 __user *size, u32 flags);
int security_setselfattr(unsigned int attr, struct lsm_ctx __user *ctx,
                         u32 size, u32 flags);
```

الـ backend لـ `lsm_get_self_attr` و`lsm_set_self_attr` syscalls (kernel 6.x+). بتقرأ/تكتب الـ security attributes للـ current task مباشرة لـ userspace. كل LSM بيحتل `lsm_ctx` entry منفصل.

- **Flow لـ `getselfattr`:**
```
for each active LSM:
    call lsm->getselfattr(attr, &ctx_buf, ...)
    copy lsm_ctx to userspace
    increment count
return count
```

---

### Category 13: Network Security

---

#### `security_socket_create`

```c
int security_socket_create(int family, int type, int protocol, int kern);
```

بتتسمى من `__sock_create()` — أولى الـ checks عند إنشاء socket. SELinux بتتحقق من `socket:create` permission.

---

#### `security_sock_rcv_skb`

```c
int security_sock_rcv_skb(struct sock *sk, struct sk_buff *skb);
```

**Hot path جدًا** — بتتسمى لكل incoming packet قبل الـ deliver للـ socket. SELinux بيتحقق من الـ secmark على الـ skb مع الـ socket policy.

---

#### `security_socket_getpeersec_stream`

```c
int security_socket_getpeersec_stream(struct socket *sock, sockptr_t optval,
                                      sockptr_t optlen, unsigned int len);
```

بتجيب الـ security context لـ peer على TCP connection — بتتسمى من `getsockopt(SO_PEERSEC)`. بتمكّن الـ server من معرفة security label للـ client.

---

#### `security_sk_classify_flow`

```c
void security_sk_classify_flow(const struct sock *sk, struct flowi_common *flic);
```

بتحدد الـ security label للـ network flow بناءً على الـ socket — بتُستخدم في الـ IP routing لاختيار الـ XFRM policy الصح.

---

### Category 14: XFRM (IPSec) Security

---

#### `security_xfrm_state_pol_flow_match`

```c
int security_xfrm_state_pol_flow_match(struct xfrm_state *x,
                                       struct xfrm_policy *xp,
                                       const struct flowi_common *flic);
```

بتتحقق لو الـ XFRM state والـ policy والـ flow كلهم بيتطابقوا من ناحية الـ security label. بترجع `1` لو match، `0` لو لأ.

- **No-LSM:** بترجع `1` دايمًا.

---

#### `security_xfrm_decode_session`

```c
int security_xfrm_decode_session(struct sk_buff *skb, u32 *secid);
```

بتستخرج الـ secid من الـ IPSec-decrypted packet. بتُستخدم في الـ policy lookup بعد decryption.

---

### Category 15: Path Security

---

#### الفرق بين `security_inode_*` و `security_path_*`

| | security_inode_* | security_path_* |
|---|---|---|
| **Context** | يشتغل على inode مباشرة | يشتغل على path (mount + dentry) |
| **وقت التنفيذ** | pre-vfs operation | pre-vfs، بس مع mount context |
| **استخدام** | SELinux, Smack | AppArmor (محتاج mount point) |
| **config** | CONFIG_SECURITY | CONFIG_SECURITY_PATH |

---

#### `security_path_chmod`

```c
int security_path_chmod(const struct path *path, umode_t mode);
```

AppArmor بتستخدمها لـ check chmod — محتاجة الـ full path لأن الـ policy بتاعتها path-based.

---

### Category 16: Keys Security

---

#### `security_key_permission`

```c
int security_key_permission(key_ref_t key_ref, const struct cred *cred,
                            enum key_need_perm need_perm);
```

بتتحقق من access على kernel keyring key. الـ `need_perm` بتكون values زي `KEY_NEED_READ`, `KEY_NEED_WRITE`, `KEY_NEED_SEARCH`.

---

#### `security_key_post_create_or_update`

```c
void security_key_post_create_or_update(struct key *keyring, struct key *key,
                                        const void *payload, size_t payload_len,
                                        unsigned long flags, bool create);
```

بتتسمى بعد create أو update لـ key — فرصة للـ LSM يخزّن info عن الـ key (مثلاً في BPF LSM).

---

### Category 17: Audit Integration

---

#### `security_audit_rule_init`

```c
int security_audit_rule_init(u32 field, u32 op, char *rulestr, void **lsmrule,
                             gfp_t gfp);
```

بتُبنى لما admin يضيف audit rule بـ LSM field (مثلاً `subj_user=unconfined_u`). بتحوّل الـ string rule لـ compiled LSM-specific representation.

---

#### `security_audit_rule_match`

```c
int security_audit_rule_match(struct lsm_prop *prop, u32 field, u32 op,
                              void *lsmrule);
```

بتطابق الـ lsm_prop للـ object الحالي مع الـ compiled rule. بتتسمى لكل audit record لتحديد هل يتكتب في audit log أو لأ.

---

### Category 18: BPF Security

---

#### `security_bpf`

```c
extern int security_bpf(int cmd, union bpf_attr *attr, unsigned int size, bool kernel);
```

بتتحقق من `bpf()` syscall — أول line of defense. SELinux بتتحقق من `bpf:command` permission حسب الـ `cmd`.

---

#### `security_bpf_token_capable`

```c
extern int security_bpf_token_capable(const struct bpf_token *token, int cap);
```

BPF token mechanism (kernel 6.7+) بيسمح لـ unprivileged users يعملوا بعض BPF operations عبر delegate token. هذا الـ hook بيتحقق لو الـ token بيعطي الـ capability المطلوبة.

---

### Category 19: securityfs

---

#### `securityfs_create_file` / `securityfs_create_dir` / `securityfs_remove`

```c
extern struct dentry *securityfs_create_file(const char *name, umode_t mode,
                                             struct dentry *parent, void *data,
                                             const struct file_operations *fops);
extern struct dentry *securityfs_create_dir(const char *name, struct dentry *parent);
extern void securityfs_remove(struct dentry *dentry);
```

بتنشئ/تحذف ملفات ومجلدات في `/sys/kernel/security/`. الـ LSMs بتستخدمها لـ expose interface للـ userspace (مثلاً SELinux expose `/sys/kernel/security/selinux/load` لـ load policy).

- **Return:** `ERR_PTR(-ENODEV)` لو `CONFIG_SECURITYFS=n`.
- **Locking:** بتقفل VFS locks داخليًا.

---

### Category 20: Lockdown, Block Device & Misc

---

#### `security_locked_down`

```c
int security_locked_down(enum lockdown_reason what);
```

بتتحقق لو الـ system في lockdown mode وبتحدد لو الـ operation المطلوبة مسموحة. الـ `lockdown_reason` enum مقسومة قسمين:
- ما قبل `LOCKDOWN_INTEGRITY_MAX`: بتحمي kernel integrity (write access).
- ما قبل `LOCKDOWN_CONFIDENTIALITY_MAX`: بتحمي kernel confidentiality (read access).

```c
/* Example lockdown reasons */
LOCKDOWN_MODULE_SIGNATURE   /* need signed modules */
LOCKDOWN_KEXEC              /* prevent kexec */
LOCKDOWN_DEBUGFS            /* prevent debugfs */
LOCKDOWN_KPROBES            /* prevent kprobes */
LOCKDOWN_BPF_READ_KERNEL    /* prevent BPF reading kernel memory */
```

- **Caller:** BPF verifier, kexec, debugfs, /dev/mem, etc.

---

#### `security_bdev_setintegrity`

```c
int security_bdev_setintegrity(struct block_device *bdev,
                               enum lsm_integrity_type type, const void *value,
                               size_t size);
```

بتخلّي الـ dm-verity driver يبلّغ الـ LSM بـ integrity info للـ block device. IMA/LSM ممكن يستخدموها لـ label الـ filesystem بناءً على الـ verified state.

---

#### `security_initramfs_populated`

```c
extern void security_initramfs_populated(void);
```

بتتسمى بعد ما الكيرنل يفرد الـ initramfs. SELinux بيستخدمها لـ restore labels على الـ initramfs files.

---

### ملاحظات ختامية مهمة

#### الـ Dual API Pattern

كل function في الملف موجودة مرتين:
1. **الـ real implementation** (لما `CONFIG_SECURITY=y`) — في `security/security.c`.
2. **stub inline** (لما `CONFIG_SECURITY=n`) — في الملف ده — بترجع 0 أو بتناديـ cap_* functions كـ fallback.

هذا بيضمن إن الـ callers في بقية الكيرنل مش محتاجين `#ifdef CONFIG_SECURITY` في كل مكان.

#### الـ Config Guards

```
CONFIG_SECURITY          → core hooks
CONFIG_SECURITY_NETWORK  → socket/network hooks
CONFIG_SECURITY_NETWORK_XFRM → IPSec hooks
CONFIG_SECURITY_PATH     → path-based hooks
CONFIG_SECURITY_INFINIBAND → IB hooks
CONFIG_SECURITYFS        → /sys/kernel/security
CONFIG_BPF_SYSCALL + CONFIG_SECURITY → BPF hooks
CONFIG_PERF_EVENTS + CONFIG_SECURITY → perf hooks
CONFIG_IO_URING + CONFIG_SECURITY → io_uring hooks
```

#### الـ lsm_prop vs secid

- **`u32 secid`**: legacy identifier — single LSM, single 32-bit ID.
- **`struct lsm_prop`**: modern — بتحمل secid لكل LSM مرة واحدة (SELinux + Smack + AppArmor + BPF). الـ new code لازم يستخدم `lsm_prop`.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — الـ Entries المهمة

الـ **LSM (Linux Security Module)** بيكتب entries في `/sys/kernel/security/` (وهو الـ `securityfs`) مش في `debugfs` مباشرةً، بس في بعض الـ LSMs زي SELinux وAppArmor بيستخدموا `debugfs` جنب.

| المسار | المحتوى | أزاي تقراه |
|--------|---------|------------|
| `/sys/kernel/security/lsm` | قائمة الـ LSMs الشغّالة | `cat /sys/kernel/security/lsm` |
| `/sys/kernel/security/selinux/` | كل الـ SELinux policy/state | `ls /sys/kernel/security/selinux/` |
| `/sys/kernel/security/selinux/enforce` | وضع الـ enforcing | `cat /sys/kernel/security/selinux/enforce` |
| `/sys/kernel/security/selinux/deny_unknown` | رد الـ unknown access | `cat /sys/kernel/security/selinux/deny_unknown` |
| `/sys/kernel/security/apparmor/` | الـ AppArmor profiles | `ls /sys/kernel/security/apparmor/` |
| `/sys/kernel/security/apparmor/.access` | واجهة الـ access check | — |
| `/sys/kernel/security/tomoyo/` | الـ TOMOYO domain info | `cat /sys/kernel/security/tomoyo/stat` |
| `/sys/kernel/debug/lsm/` | متاح لو `CONFIG_DEBUG_FS=y` | `ls /sys/kernel/debug/lsm/` |

```bash
# قراءة الـ LSMs الشغّالة
cat /sys/kernel/security/lsm
# مثال output:
# lockdown,capability,landlock,yama,selinux

# قراءة enforce mode في SELinux
cat /sys/kernel/security/selinux/enforce
# 1 = enforcing, 0 = permissive

# قراءة الـ policy stats
cat /sys/kernel/security/selinux/policyvers
```

---

#### 2. sysfs — الـ Entries المهمة

| المسار | الوصف |
|--------|-------|
| `/proc/sys/kernel/yama/ptrace_scope` | مستوى حماية الـ ptrace |
| `/proc/sys/vm/mmap_min_addr` | الحد الأدنى لعناوين الـ mmap (من `security.h`) |
| `/proc/sys/fs/suid_dumpable` | تحكم في الـ core dump لـ setuid |
| `/proc/self/attr/current` | الـ security context للـ process الحالي |
| `/proc/self/attr/exec` | الـ context اللي هيتعيّن بعد exec |
| `/proc/self/attr/fscreate` | الـ context للـ file اللي هيتعمل |
| `/proc/self/attr/keycreate` | الـ context للـ key اللي هيتعمل |
| `/proc/self/attr/sockcreate` | الـ context للـ socket اللي هيتعمل |
| `/proc/<pid>/attr/` | نفس العناصر لأي process |

```bash
# تحقق من الـ security context للـ process
cat /proc/self/attr/current
# مثال: unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

# تحقق من الـ mmap_min_addr (تحمي من NULL-deref exploits)
cat /proc/sys/vm/mmap_min_addr
# 65536

# تحقق من الـ ptrace scope
cat /proc/sys/kernel/yama/ptrace_scope
# 0=disabled, 1=restricted, 2=admin-only, 3=no-attach
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

الـ LSM subsystem مش عنده tracepoints مخصوصة في الكود القديم، بس من kernel 5.x+ في tracepoints عبر **BPF LSM** وعبر tracing الـ functions مباشرةً.

```bash
# 1. تفعيل function tracing لـ security hooks
cd /sys/kernel/debug/tracing

# trace كل security_* functions
echo 'security_*' > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
cat trace | head -50
echo 0 > tracing_on

# 2. trace hook محدد مثلاً security_file_open
echo security_file_open > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
# شغّل العملية المشبوهة هنا
cat trace
echo 0 > tracing_on
echo nop > current_tracer
echo > set_ftrace_filter

# 3. function_graph لرؤية call tree
echo security_inode_permission > set_graph_function
echo function_graph > current_tracer
echo 1 > tracing_on
ls /sensitive/path/
cat trace
echo 0 > tracing_on

# 4. trace events للـ syscall layer (تعطي context للـ LSM decisions)
echo 1 > events/syscalls/sys_enter_open/enable
echo 1 > events/syscalls/sys_exit_open/enable
cat trace_pipe
```

---

#### 4. printk / dynamic debug

```bash
# تفعيل dynamic debug لكل ملفات الـ security subsystem
echo 'file security/*.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file security/selinux/*.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file security/apparmor/*.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل dynamic debug لـ capability module
echo 'file security/commoncap.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# flags: p=print, f=function name, l=line number, m=module name, t=thread ID

# إيقاف
echo 'file security/*.c -p' > /sys/kernel/debug/dynamic_debug/control

# رفع مستوى الـ printk لو الرسائل مش بتظهر
echo 8 > /proc/sys/kernel/printk

# مشاهدة الـ kernel messages في real-time
dmesg -w | grep -i 'avc\|denied\|security\|capability\|lsm'

# في حالة SELinux — رسائل الـ AVC
dmesg | grep 'avc:  denied'
# أو عبر audit daemon
ausearch -m avc -ts today
```

---

#### 5. Kernel Config Options للـ Debugging

| الـ Config | الوصف |
|-----------|-------|
| `CONFIG_SECURITY` | تفعيل الـ LSM framework أصلاً |
| `CONFIG_SECURITY_NETWORK` | LSM hooks للـ network |
| `CONFIG_SECURITY_PATH` | LSM hooks للـ path operations |
| `CONFIG_SECURITY_NETWORK_XFRM` | LSM hooks للـ IPsec/XFRM |
| `CONFIG_SECURITYFS` | تركيب `/sys/kernel/security/` |
| `CONFIG_DEBUG_FS` | يلزم لـ debugfs |
| `CONFIG_SECURITY_SELINUX` | تفعيل SELinux |
| `CONFIG_SECURITY_SELINUX_DEVELOP` | SELinux permissive mode |
| `CONFIG_SECURITY_SELINUX_AVC_STATS` | إحصائيات الـ AVC cache |
| `CONFIG_SECURITY_APPARMOR` | تفعيل AppArmor |
| `CONFIG_SECURITY_APPARMOR_DEBUG` | AppArmor debug messages |
| `CONFIG_SECURITY_APPARMOR_DEBUG_ASSERTS` | AppArmor assertions |
| `CONFIG_SECURITY_SMACK` | تفعيل Smack |
| `CONFIG_SECURITY_TOMOYO` | تفعيل TOMOYO |
| `CONFIG_SECURITY_LANDLOCK` | تفعيل Landlock |
| `CONFIG_SECURITY_YAMA` | تفعيل Yama (ptrace hardening) |
| `CONFIG_BPF_LSM` | BPF-based LSM programs |
| `CONFIG_LSM_MUTABLE_HOOKS` | السماح بتعديل الـ hooks في runtime |
| `CONFIG_AUDIT` | نظام الـ audit (لازم مع SELinux) |
| `CONFIG_AUDITSYSCALL` | audit على مستوى الـ syscall |
| `CONFIG_DEBUG_CREDENTIALS` | تتبع الـ credentials alloc/free |
| `CONFIG_KALLSYMS` | لازم للـ ftrace function filter |

```bash
# تحقق من الـ LSMs المفعّلة في الكرنل الحالي
grep 'CONFIG_SECURITY' /boot/config-$(uname -r) | grep '=y'

# تحقق من ترتيب الـ LSMs
grep CONFIG_LSM= /boot/config-$(uname -r)
# CONFIG_LSM="lockdown,yama,loadpin,safesetid,integrity,selinux,smack,tomoyo,apparmor,bpf"
```

---

#### 6. أدوات مخصصة للـ Subsystem

##### SELinux Tools
```bash
# حالة SELinux
sestatus
getenforce
setenforce 0   # permissive مؤقتاً للـ debug

# تحليل الـ AVC denials
audit2why < /var/log/audit/audit.log
audit2allow -a   # اقتراح rules

# context الـ process والـ file
ps -eZ | grep <process_name>
ls -Z /path/to/file

# البحث في الـ audit log
ausearch -m avc --start recent
ausearch -m avc -c httpd   # لـ process اسمه httpd
```

##### AppArmor Tools
```bash
# حالة الـ profiles
aa-status

# وضع profile في complain mode للـ debug
aa-complain /usr/bin/firefox

# قراءة الـ violations
journalctl -k | grep 'apparmor="DENIED"'
cat /sys/kernel/security/apparmor/profiles
```

##### BPF LSM Tools
```bash
# عرض الـ BPF programs المحمّلة
bpftool prog list | grep lsm

# عرض الـ LSM hooks المتاحة للـ BPF
bpftool prog show -p <prog_id>
```

##### Capability Debugging
```bash
# capabilities للـ process
cat /proc/self/status | grep Cap
capsh --decode=<hex_value>   # فك تشفير الـ capabilities

# مثال كامل
grep CapEff /proc/1/status
# CapEff: 000001ffffffffff
capsh --decode=000001ffffffffff
```

---

#### 7. جدول رسائل الـ Error الشائعة

| الرسالة في الـ dmesg/log | المعنى | الحل |
|--------------------------|--------|------|
| `avc: denied { read } for pid=XXX comm="app" path="/etc/secret" scontext=...` | SELinux منع قراءة file | أضف policy rule أو فحص الـ context بـ `chcon` |
| `avc: denied { execute } for pid=XXX scontext=... tcontext=...:bin_t` | SELinux منع تنفيذ binary | فحص الـ file context: `ls -Z /path` ثم `restorecon` |
| `apparmor="DENIED" operation="open" profile="firefox"` | AppArmor منع عملية `open` | تعديل الـ profile أو استخدام `aa-complain` |
| `WARNING: at security/capability.c:XXX` | violation في الـ capability check | تتبع الـ stack لمعرفة من طلب الـ capability |
| `Oops: general protection fault` بعد LSM hook | bug في LSM hook نفسه أو null pointer | فعّل `CONFIG_DEBUG_CREDENTIALS` وتابع الـ stack trace |
| `LOCKDOWN: kexec: locked down` | الـ lockdown subsystem منع الـ kexec | تعطيل Secure Boot أو تعديل الـ lockdown policy |
| `LOCKDOWN: /dev/mem: Access to physical memory restricted` | الـ LOCKDOWN_DEV_MEM مفعّل | استخدم kernel debugger أو أعد build بدون lockdown |
| `Unable to open initial console` مع LSM errors | الـ LSM منع الـ init console access | فحص الـ policy للـ init context |
| `security_inode_permission: -13` | الـ hook رجع `-EACCES` | تحقق من الـ inode security context |
| `smack: SMACK-MAC for task process` | Smack منع IPC | فحص الـ Smack labels بـ `cat /proc/<pid>/attr/current` |
| `integrity: Unable to open file` | IMA/EVM فشل في فتح الـ file | تحقق من الـ IMA policy وتوقيعات الـ files |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

أهم أماكن تحط فيها debug code لو بتطور LSM أو بتحقق في مشكلة:

```c
/* في security/security.c — بداية كل hook dispatcher */
int security_inode_permission(struct inode *inode, int mask)
{
    /* نقطة مهمة: لو الـ hook بيرفض بشكل غير متوقع */
    int ret;
    ret = call_int_hook(inode_permission, 0, inode, mask);
    if (ret) {
        WARN_ON(ret == -ENOSYS); /* مش المفروض يحصل */
        pr_debug("LSM: inode_permission denied ino=%lu mask=%d ret=%d\n",
                 inode->i_ino, mask, ret);
    }
    return ret;
}

/* في LSM hook نفسه — لو قرار الرفض غير متوقع */
int my_lsm_inode_permission(struct inode *inode, int mask)
{
    struct my_inode_security *isec = inode->i_security;

    WARN_ON(!isec);            /* الـ security blob لازم يكون موجود */
    if (unlikely(condition_should_not_happen)) {
        dump_stack();          /* اطبع الـ call stack */
        return -EPERM;
    }
    return 0;
}

/* في credential operations — تتبع من غيّر الـ creds */
int security_prepare_creds(struct cred *new, const struct cred *old, gfp_t gfp)
{
    WARN_ON(new->security && new->security != old->security);
    /* ... */
}

/* في security_capable — تتبع طلبات الـ capabilities */
int security_capable(const struct cred *cred, struct user_namespace *ns,
                     int cap, unsigned int opts)
{
    if (cap == CAP_SYS_ADMIN && !(opts & CAP_OPT_NOAUDIT))
        pr_debug("LSM: CAP_SYS_ADMIN requested by pid=%d comm=%s\n",
                 current->pid, current->comm);
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق من حالة الـ Hardware مقارنةً بالـ Kernel State

الـ LSM framework نفسه مش بيتعامل مع hardware مباشرةً، لكن المكونات اللي بتأثر عليه في الـ hardware مستوى هي:

| المكوّن | أزاي تتحقق |
|---------|-----------|
| **Secure Boot** | `mokutil --sb-state` أو `bootctl status` |
| **TPM** | `tpm2_getcap handles-pcr` و `tpm2_pcrread` |
| **IOMMU** | `dmesg | grep IOMMU` ثم `cat /sys/kernel/iommu_groups/*/devices/*` |
| **dm-verity** | `dmsetup status <device>` — تشوف الـ hash verification |
| **Secure Memory** | `/proc/iomem` — تحقق من EFI runtime regions |

```bash
# حالة Secure Boot
mokutil --sb-state
# SecureBoot enabled

# حالة TPM
ls /dev/tpm*
tpm2_getcap properties-fixed 2>/dev/null | head -20

# الـ dm-verity — لو system بيستخدم integrity checking
dmsetup status
# example: vda-verity  0 20971520 verity V:10 0 2 0

# الـ IMA (Integrity Measurement Architecture) - يعتمد على hardware TPM
cat /sys/kernel/security/ima/ascii_runtime_measurements | head -5
```

---

#### 2. Register Dump Techniques

الـ LSM نادراً ما بيحتاج register dumps للـ hardware، بس لو في مشكلة في الـ Secure Boot أو TPM:

```bash
# قراءة من /dev/mem (محظور لو LOCKDOWN_DEV_MEM مفعّل)
# أول لازم تتحقق:
cat /sys/kernel/security/selinux/enforce  # لو SELinux

# استخدام devmem2 (لو متاح)
# devmem2 0xFEDC0000 w   # قراءة ACPI register مثلاً

# الطريقة الآمنة — عبر ioport لـ legacy devices
sudo ioperm 0x80 1 1
sudo inb 0x80

# قراءة MSR (محظور لو LOCKDOWN_MSR)
rdmsr 0x3A  # IA32_FEATURE_CONTROL — بيتحكم في VT-x و SMX
# Bit 0 = Lock bit, Bit 1 = VMX in SMX, Bit 2 = VMX outside SMX

# لو الـ lockdown منع الوصول:
dmesg | grep LOCKDOWN
# LOCKDOWN: /dev/mem: Access to physical memory restricted; see man kernel_lockdown.7
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ security subsystem ما بيحتاج logic analyzer في العادة، بس في حالات معينة:

- **TPM communications**: الـ TPM بيتكلم على SPI أو I2C — تقدر تحط logic analyzer على الـ bus وتتحقق من الـ commands والـ responses
- **Secure Boot sequence**: أحياناً بيكون في timing issues في UEFI→kernel handoff يسببوا LSM initialization failures
- **IOMMU DMA operations**: لو في مشكلة في الـ IOMMU اللي بيحميه SELinux/SMACK

```
Logic Analyzer Setup for TPM SPI:
─────────────────────────────────
Channel 0: SPI_CLK   (عادةً 10-33 MHz)
Channel 1: SPI_MOSI  (host → TPM)
Channel 2: SPI_MISO  (TPM → host)
Channel 3: SPI_CS#   (chip select, active low)

Protocol decoder: SPI → TPM 2.0 protocol
Look for: TPM_STS register (0xD40018) status bits
```

---

#### 4. مشاكل الـ Hardware الشائعة → Kernel Log Patterns

| مشكلة الـ Hardware | النمط في الـ Kernel Log | السبب المحتمل |
|--------------------|------------------------|--------------|
| TPM غير موجود أو معطوب | `tpm tpm0: A TPM error occurred` | TPM firmware bug أو مشكلة hardware |
| Secure Boot signature failure | `Lockdown: kexec: Kernel image signature required` | kernel غير موقّع في Secure Boot |
| IOMMU fault | `DMAR: DRHD: handling fault... address: 0xXXX` | DMA من device بيحاول الوصول لمنطقة محمية |
| dm-verity corruption | `device-mapper: verity: <dev>: data block <N> is corrupted` | corruption في الـ storage أو الـ hash tree |
| IMA measurement fail | `integrity: Hmac(sha1): No key` | TPM PCR sealing فشل |
| EFI memory map conflict | `efi: memattr: ! omitting 0x...` | مشكلة في الـ EFI memory attributes |

```bash
# تحقق من أخطاء TPM
dmesg | grep -i tpm
journalctl -k | grep -i 'tpm\|tcg'

# تحقق من أخطاء IOMMU
dmesg | grep -i 'iommu\|dmar\|iova'

# تحقق من أخطاء dm-verity
dmesg | grep -i verity
```

---

#### 5. Device Tree Debugging (للـ Embedded Systems)

في embedded Linux (ARM/RISC-V) اللي فيه LSM:

```bash
# تحقق من الـ DT nodes المرتبطة بالـ security hardware
ls /proc/device-tree/
# أو
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 'tpm\|secure'

# التحقق من الـ chosen node (bootloader بيحط LSM config هنا أحياناً)
cat /proc/device-tree/chosen/linux,lsm 2>/dev/null || echo "not set in DT"

# TPM في DT
find /proc/device-tree -name 'compatible' -exec grep -l tpm {} \;
# ثم
cat /proc/device-tree/spi@.../tpm@.../compatible

# التحقق أن الـ DT والـ hardware متطابقين
# مثلاً لو DT بيقول TPM على SPI1 CS0:
ls /dev/spidev*
# لازم يكون /dev/spidev1.0 موجود
```

```bash
# مشكلة شائعة: LSM مش بيشتغل في embedded
# التحقق أن الـ LSM initialized قبل أي userspace
dmesg | grep -E 'LSM|security.*initialized|selinux.*initialized'
# المفروض تشوف:
# [    0.000000] LSM: initializing lsm=capability,selinux
# [    0.000000] SELinux:  Initializing.
```

---

### Practical Commands

#### كل الأوامر جاهزة للنسخ

##### فحص حالة الـ LSM System

```bash
#!/bin/bash
echo "=== LSM Status ==="
cat /sys/kernel/security/lsm
echo ""

echo "=== SELinux Status ==="
sestatus 2>/dev/null || echo "SELinux not active"

echo "=== AppArmor Status ==="
aa-status 2>/dev/null || cat /sys/kernel/security/apparmor/.ns_name 2>/dev/null || echo "AppArmor not active"

echo "=== Lockdown Status ==="
# هيعطي error لو مش موجود، ده طبيعي
cat /sys/kernel/security/lockdown 2>/dev/null || echo "No lockdown file (check dmesg)"

echo "=== mmap_min_addr ==="
cat /proc/sys/vm/mmap_min_addr

echo "=== Yama ptrace scope ==="
cat /proc/sys/kernel/yama/ptrace_scope 2>/dev/null || echo "Yama not enabled"
```

##### تتبع الـ LSM Hook Denials في Real-Time

```bash
# الطريقة 1: ftrace على security hooks
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo 'security_*' > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
sleep 5
cat trace | grep -v '^#' | head -100
echo 0 > tracing_on
echo nop > current_tracer

# الطريقة 2: audit system (الأكثر شيوعاً في production)
auditctl -a always,exit -F arch=b64 -S open -F exit=-EACCES -k lsm_deny
ausearch -k lsm_deny --start recent
auditctl -d always,exit -F arch=b64 -S open -F exit=-EACCES -k lsm_deny
```

##### تحليل AVC Denial وإنشاء Policy Rule

```bash
# خطوة 1: البحث عن الـ denial
ausearch -m avc -ts today | tail -20

# خطوة 2: فهم السبب
ausearch -m avc -ts today | audit2why

# مثال output:
# type=AVC msg=audit(1706400000.123:456): avc: denied { read } for
#   pid=1234 comm="myapp" name="secret.conf" dev="sda1" ino=5678
#   scontext=system_u:system_r:myapp_t:s0
#   tcontext=system_u:object_r:etc_t:s0
#   tclass=file permissive=0
# Was caused by: Missing type enforcement (TE) allow rule.

# خطوة 3: إنشاء rule (للـ development فقط)
ausearch -m avc -ts today | audit2allow -M mymodule
semodule -i mymodule.pp

# خطوة 4: التحقق
semodule -l | grep mymodule
```

##### تفعيل Dynamic Debug للـ Security Subsystem

```bash
# تفعيل debug messages لكل الـ security files
for f in $(ls /sys/kernel/debug/dynamic_debug/control 2>/dev/null); do
    echo "file security/*.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
    echo "file security/selinux/*.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
    echo "file security/apparmor/*.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
done

# مراقبة الـ output
dmesg -w | grep -E 'security|lsm|selinux|apparmor'

# إيقاف
echo "file security/*.c -p" > /sys/kernel/debug/dynamic_debug/control
```

##### فحص الـ Capabilities لـ Process معين

```bash
#!/bin/bash
PID=${1:-self}
echo "=== Capabilities for PID $PID ==="
STATUS=$(cat /proc/$PID/status 2>/dev/null)

for cap_type in CapInh CapPrm CapEff CapBnd CapAmb; do
    val=$(echo "$STATUS" | grep "^$cap_type:" | awk '{print $2}')
    echo -n "$cap_type: $val => "
    capsh --decode=$val 2>/dev/null || echo "(capsh not installed)"
done

echo ""
echo "=== Security Context ==="
cat /proc/$PID/attr/current 2>/dev/null || echo "N/A"
```

##### فحص الـ IMA Measurements (Integrity)

```bash
# قراءة الـ IMA log
cat /sys/kernel/security/ima/ascii_runtime_measurements | head -20
# Format: PCR-index hash template-hash filename

# فحص الـ IMA policy
cat /sys/kernel/security/ima/policy 2>/dev/null || echo "IMA policy not readable"

# عدد الـ measurements
wc -l /sys/kernel/security/ima/ascii_runtime_measurements

# مثال output:
# 10 sha1:abc123... ima-ng sha256:def456... /usr/bin/bash
# PCR=10, hash=abc123, file=/usr/bin/bash
```

##### تشخيص مشكلة Mount مع LSM

```bash
# لو mount بيفشل بسبب LSM
mount -v /dev/sdb1 /mnt/test 2>&1

# فحص الـ kernel log فوراً
dmesg | tail -20

# لو SELinux يمنع الـ mount
ausearch -m avc -ts recent | grep mount

# تشغيل mount في permissive mode مؤقتاً
setenforce 0
mount /dev/sdb1 /mnt/test
setenforce 1
# لو نجح → مشكلة SELinux policy

# تحقق من الـ LSM mount options
cat /proc/mounts | grep 'context=\|rootcontext='
```

##### مقارنة حالة الـ LSM Hooks قبل وبعد عملية

```bash
# snapshot لحالة الـ AVC cache قبل
cat /sys/kernel/security/selinux/avc/cache_stats > /tmp/avc_before

# تشغيل العملية المشبوهة
./suspicious_operation

# snapshot بعد
cat /sys/kernel/security/selinux/avc/cache_stats > /tmp/avc_after

# مقارنة
diff /tmp/avc_before /tmp/avc_after
# output مثال:
# < lookups: 1000 hits: 950 misses: 50 allocations: 50 reclaims: 0 frees: 0
# > lookups: 1050 hits: 990 misses: 60 allocations: 60 reclaims: 0 frees: 0
# ده معناه 50 lookup جديد، 10 misses (تحققات جديدة)
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Android TV Box على Allwinner H616 — Binder يرفض كل Transaction

#### العنوان
**الـ SELinux يكسر Binder IPC في Android TV box بعد تفعيل Enforcing mode**

#### السياق
بتشتغل على Android TV box بـ Allwinner H616، المنتج بيشغّل Android 12. المشروع في مرحلة bring-up ومحتاجين تفعيل **SELinux** بـ `enforcing` mode قبل الشحن. كل حاجة كانت شغّالة في `permissive` mode.

#### المشكلة
بعد تغيير `SELINUX=enforcing` في `sepolicy`، كل الـ apps اللي بتستخدم **Binder IPC** (زي `surfaceflinger`, `mediaserver`, `audioserver`) بتتوقف فوراً مع:

```
adb logcat | grep "avc:"
# W/Binder  : Binder call failed, -13
# avc: denied { call } for scontext=u:r:mediaserver:s0
#              tcontext=u:r:surfaceflinger:s0 tclass=binder
```

#### التحليل
الـ kernel بيمر على LSM hooks في `include/linux/security.h`:

```c
/* كل binder transaction بتعدي على الـ hook ده */
int security_binder_transaction(const struct cred *from,
                                const struct cred *to);

int security_binder_transfer_binder(const struct cred *from,
                                    const struct cred *to);

int security_binder_transfer_file(const struct cred *from,
                                  const struct cred *to,
                                  const struct file *file);
```

الـ flow بيكون:

```
binder_thread_write()
    └─> binder_transaction()
            └─> security_binder_transaction(from_cred, to_cred)
                    └─> SELinux: selinux_binder_transaction()
                            └─> avc_has_perm(from_sid, to_sid, SECCLASS_BINDER, ...)
                                    └─> DENIED → returns -EACCES (-13)
```

الـ `security_binder_set_context_mgr()` كمان بيتحقق لما `servicemanager` يسجّل نفسه:

```c
int security_binder_set_context_mgr(const struct cred *mgr);
/* لو SELinux ما سمحش → servicemanager نفسه بيفشل يشتغل */
```

#### الحل
1. **تحديد الـ domains المتأثرة:**

```bash
adb shell dmesg | grep "avc: denied" | grep "binder"
# أو
adb shell cat /proc/kmsg | grep binder
```

2. **كتابة sepolicy صح:**

```
# file: device/allwinner/h616/sepolicy/mediaserver.te
allow mediaserver surfaceflinger:binder { call transfer };
allow surfaceflinger mediaserver:binder { call transfer };

# لو في file transfer عبر binder
allow mediaserver surfaceflinger:binder transfer_binder;
```

3. **التحقق قبل الـ enforcing:**

```bash
# شغّل في permissive وجمّع الـ denials الأول
adb shell setenforce 0
adb shell logcat -d | audit2allow -p out/target/product/h616/root/sepolicy
```

#### الدرس المستفاد
الـ `security_binder_*` hooks في `security.h` بتُطبَّق على **كل** binder call — حتى بين system processes. لازم sepolicy تعمل `allow` صريح لكل زوج domain قبل ما تحوّل لـ `enforcing`. استخدم `audit2allow` في permissive mode لتجميع كل الـ denials قبل الشحن.

---

### السيناريو 2: Industrial Gateway على AM62x — kernel module مرفوض بعد تفعيل Secure Boot

#### العنوان
**`security_kernel_load_data` يرفض تحميل out-of-tree driver في gateway صناعي**

#### السياق
بتشتغل على industrial gateway بـ TI AM62x بيشغّل Debian 12. العميل طلب تفعيل **Secure Boot** + **IMA (Integrity Measurement Architecture)** على الـ production image. البورد عندها driver SPI مخصص لـ industrial sensor (out-of-tree module).

#### المشكلة
بعد تفعيل IMA policy، الـ module بيرفض التحميل:

```bash
modprobe industrial_sensor
# modprobe: ERROR: could not insert 'industrial_sensor':
#           Operation not permitted

dmesg | tail
# [  45.123] IMA: Refusing unsigned module load
# [  45.124] ima: appraise_measurement: LookupDigest error
```

#### التحليل
التسلسل في الـ kernel:

```c
/* في load_module() → بيتنادى قبل ما الـ module يُحمَّل */
int security_kernel_load_data(enum kernel_load_data_id id, bool contents);

/* بعد ما الـ buffer اتقرأ في الذاكرة */
int security_kernel_post_load_data(char *buf, loff_t size,
                                   enum kernel_load_data_id id,
                                   char *description);
```

الـ `id` هنا بيساوي `LOADING_MODULE` (من enum `kernel_load_data_id` المُعرَّف في الملف):

```c
/* الـ enum ده بيتبني من kernel_read_file_id بالـ macro ده */
#define __data_id_enumify(ENUM, dummy) LOADING_ ## ENUM,

enum kernel_load_data_id {
    __kernel_read_file_id(__data_id_enumify)
    /* ينتج: LOADING_UNKNOWN, LOADING_FIRMWARE, LOADING_MODULE, ... */
};
```

الـ IMA hook جوا `security_kernel_load_data` بيتحقق إن الـ module موقّع بـ key موجود في kernel keyring. الـ out-of-tree module مش موقّع → `EACCES`.

كمان `security_kernel_module_request()` بيتنادى لو الـ kernel حاول يعمل `request_module()`:

```c
int security_kernel_module_request(char *kmod_name);
/* SELinux بيتحقق من domain → LOADING_MODULE مسموح أو لأ */
```

#### الحل

**الخيار 1: توقيع الـ module:**

```bash
# على build machine — توقيع بـ private key المسجّل في kernel
/usr/src/linux-headers-$(uname -r)/scripts/sign-file \
    sha256 \
    /path/to/signing_key.pem \
    /path/to/signing_cert.pem \
    industrial_sensor.ko

# تحقق من التوقيع
modinfo industrial_sensor.ko | grep sig
```

**الخيار 2: إضافة IMA policy rule للـ module بالاسم:**

```bash
# في /etc/ima/ima-policy
appraise func=MODULE_CHECK appraise_type=imasig|modsig
# أو تخفيف للـ module المحدد (مش production-safe)
dont_appraise func=MODULE_CHECK obj_user=root
```

**الخيار 3: تضمين الـ module في kernel مباشرة (الأكثر أماناً):**

```
# في Kconfig
CONFIG_INDUSTRIAL_SENSOR=y  # بدل =m
```

#### الدرس المستفاد
**الـ `kernel_load_data_id`** و `kernel_read_file_id` في `security.h` بيحكموا **كل** عملية تحميل ديناميكي — firmware، modules، kexec. في production مع IMA، كل module لازم موقّع أو مبني static في الـ kernel. خطّط لده من أول البورد bring-up.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — filesystem mount بيفشل بـ SELinux

#### العنوان
**`security_sb_set_mnt_opts` يمنع mount الـ VFAT partition في embedded IoT device**

#### السياق
بتشتغل على IoT sensor node بـ STM32MP1 (Cortex-A7 + M4). الـ Linux side (A7) محتاج يعمل mount لـ FAT32 partition مشتركة مع الـ M4 عشان تبادل configuration files. الـ image بيشغّل **SELinux** مع custom policy.

#### المشكلة
بعد boot، الـ mount بيفشل:

```bash
mount /dev/mmcblk0p3 /mnt/shared
# mount: /mnt/shared: permission denied.

dmesg | grep "SELinux\|avc"
# avc: denied { mount } for pid=312 comm="mount"
#      scontext=u:r:init:s0 tcontext=u:object_r:vfat:s0
#      tclass=filesystem permissive=0
```

#### التحليل
سلسلة الـ hooks في `security.h`:

```c
/* أول hook بيتنادى عند mount */
int security_sb_mount(const char *dev_name, const struct path *path,
                      const char *type, unsigned long flags, void *data);

/* بعد ما الـ superblock يتعمل */
int security_sb_kern_mount(const struct super_block *sb);

/* تحديد الـ mount options المسموح بيها */
int security_sb_set_mnt_opts(struct super_block *sb,
                              void *mnt_opts,
                              unsigned long kern_flags,
                              unsigned long *set_kern_flags);

/* التحقق من compatibility الـ options مع الـ superblock الموجود */
int security_sb_mnt_opts_compat(struct super_block *sb, void *mnt_opts);
```

الـ `security_sb_mount()` بيتحقق إن الـ process له صلاحية `{ mount }` على الـ filesystem type `vfat`. لو الـ domain مش عنده الـ permission → deny.

الـ flow:

```
do_mount()
    └─> security_sb_mount(dev, path, "vfat", flags, data)
            └─> selinux_sb_mount()
                    └─> avc_has_perm(init_sid, vfat_sid, SECCLASS_FILESYSTEM,
                                     FILESYSTEM__MOUNT, ...)
                                └─> DENIED
```

#### الحل

**تعديل الـ SELinux policy:**

```
# file: sepolicy/init.te
allow init vfat:filesystem { mount unmount getattr };

# تعريف الـ type للـ mount point
# file: sepolicy/file_contexts
/mnt/shared    u:object_r:shared_data_file:s0

# السماح للـ sensor app بالقراءة والكتابة
allow sensor_app shared_data_file:file { read write open };
```

**أو تعيين SELinux context عند الـ mount:**

```bash
mount -t vfat -o context=u:object_r:shared_data_file:s0 \
      /dev/mmcblk0p3 /mnt/shared
```

**للـ fstab:**

```
# /etc/fstab
/dev/mmcblk0p3  /mnt/shared  vfat  defaults,context=u:object_r:shared_data_file:s0  0  0
```

#### الدرس المستفاد
الـ `security_sb_mount()` و `security_sb_set_mnt_opts()` hooks بيتنادوا قبل أي بيانات من الـ filesystem تُقرأ. في embedded systems بـ shared partitions بين cores، لازم الـ SELinux policy تعرف صراحة الـ filesystem types والـ mount points المسموح بيها من أول مرحلة التصميم.

---

### السيناريو 4: Automotive ECU على i.MX8 — lockdown يمنع debugfs أثناء field debug

#### العنوان
**`security_locked_down(LOCKDOWN_DEBUGFS)` يمنع فريق الـ field service من الوصول لبيانات الـ UART**

#### السياق
بتشتغل على automotive ECU بـ i.MX8QM بيتحكم في نظام HVAC. الـ production image عنده **kernel lockdown** مفعّل بـ `integrity` mode (عبر EFI Secure Boot). فريق الـ field service محتاج يقرأ UART debug registers عبر `debugfs` لتشخيص مشكلة في بورد عميل.

#### المشكلة
```bash
# على الـ ECU في الميدان
ls /sys/kernel/debug/
# ls: cannot open directory '/sys/kernel/debug/': Permission denied

# أو لو debugfs متاح ولكن الـ read مرفوض
cat /sys/kernel/debug/imx-uart/uart4/regs
# cat: /sys/kernel/debug/imx-uart/uart4/regs: Operation not permitted
```

#### التحليل
الـ lockdown reason `LOCKDOWN_DEBUGFS` موجود في الـ enum:

```c
enum lockdown_reason {
    LOCKDOWN_NONE,
    /* ... */
    LOCKDOWN_DEBUGFS,        /* ← هنا المشكلة */
    /* ... */
    LOCKDOWN_INTEGRITY_MAX,  /* كل اللي فوقه بيتحجب في integrity mode */
    /* ... */
};
```

الـ hook بيتنادى من `debugfs_file_get()`:

```c
/* في fs/debugfs/file.c */
int security_locked_down(enum lockdown_reason what);
/* what = LOCKDOWN_DEBUGFS → returns -EPERM لو lockdown مفعّل */
```

الـ `lockdown_reasons` array (المُعلن في `security.h`):

```c
extern const char *const lockdown_reasons[LOCKDOWN_CONFIDENTIALITY_MAX+1];
/* lockdown_reasons[LOCKDOWN_DEBUGFS] = "debugfs" */
```

الـ integrity lockdown بيحجب كل اللي قبل `LOCKDOWN_INTEGRITY_MAX` — وده بيشمل `LOCKDOWN_DEBUGFS` كـ write access لكن في بعض implementations كمان الـ read.

#### الحل

**الخيار 1: استخدام `/proc` أو sysfs بدل debugfs (الأصح للـ production):**

```c
/* في الـ UART driver — expose registers عبر sysfs بدل debugfs */
static ssize_t uart_regs_show(struct device *dev,
                               struct device_attribute *attr, char *buf)
{
    struct imx_port *sport = dev_get_drvdata(dev);
    /* قراءة الـ registers وعرضها — مش محتاج debugfs */
    return sprintf(buf, "UCR1: 0x%08x\n", readl(sport->port.membase + UCR1));
}
static DEVICE_ATTR_RO(uart_regs);
```

**الخيار 2: تخفيف الـ lockdown مؤقتاً في field (مع authentication):**

```bash
# لو الـ EFI Secure Boot بيسمح بـ custom MOK
# تغيير lockdown level عبر sysctl (لو مسموح في الـ policy)
echo "integrity" > /sys/kernel/security/lockdown
# أو
echo "none" > /sys/kernel/security/lockdown  # تخفيف كامل للـ debug
```

**الخيار 3: custom debug interface عبر character device:**

```c
/* /dev/ecu_debug — interface مخصص بـ ioctl للـ field service */
static long ecu_debug_ioctl(struct file *f, unsigned int cmd, unsigned long arg)
{
    /* التحقق من credentials الـ field engineer */
    if (!capable(CAP_SYS_RAWIO))
        return -EPERM;
    /* قراءة UART registers وإرجاعها */
}
```

#### الدرس المستفاد
الـ `LOCKDOWN_DEBUGFS` في `lockdown_reason` enum هو قرار **تصميمي** مقصود — debugfs يفضح معلومات kernel داخلية ممكن تُستخدم لتجاوز الـ integrity. في automotive و industrial، صمّم debug interfaces منفصلة عبر sysfs أو character devices من أول مرحلة التصميم، ومتعتمدش على debugfs في production builds.

---

### السيناريو 5: Custom Board Bring-Up على RK3562 — dm-verity يمنع rootfs mount

#### العنوان
**`security_bdev_setintegrity` و `LSM_INT_DMVERITY_ROOTHASH` يوقفوا الـ boot في custom RK3562 board**

#### السياق
بتشتغل على custom board بـ Rockchip RK3562 لمنتج industrial HMI. الـ image مُعدّ لاستخدام **dm-verity** على الـ rootfs partition لضمان integrity. أول boot بعد flash image جديد.

#### المشكلة
الـ board بتوقف عند الـ boot:

```
[    8.432] device-mapper: verity: sha256 hash mismatch
[    8.433] device-mapper: verity: Data device verification failed
[    8.434] EXT4-fs error (device dm-0): ext4_find_entry:1455
[    8.435] Kernel panic - not syncing: VFS: Unable to mount root fs
```

أو لو الـ initramfs شغّال بيحاول recover:

```bash
# من initramfs shell
veritysetup verify /dev/mmcblk0p2 /dev/mmcblk0p3 <roothash>
# Verity device verify failed with code -22
```

#### التحليل
الـ LSM hooks المتعلقة بـ dm-verity في `security.h`:

```c
/* الـ types المسموح بيها لـ integrity */
enum lsm_integrity_type {
    LSM_INT_DMVERITY_SIG_VALID,      /* التوقيع على الـ roothash صحيح */
    LSM_INT_DMVERITY_ROOTHASH,       /* الـ roothash نفسه */
    LSM_INT_FSVERITY_BUILTINSIG_VALID,
};

/* الـ struct بيحمل معلومات الـ digest */
struct dm_verity_digest {
    const char *alg;       /* مثلاً "sha256" */
    const u8 *digest;      /* الـ hash value */
    size_t digest_len;
};

/* بيتنادى من dm-verity عند إنشاء الـ block device */
int security_bdev_setintegrity(struct block_device *bdev,
                                enum lsm_integrity_type type,
                                const void *value,
                                size_t size);
```

الـ flow:

```
dm_verity_init()
    └─> verity_verify_io()
            └─> [hash mismatch detected]
                    └─> security_bdev_setintegrity(bdev,
                                LSM_INT_DMVERITY_ROOTHASH,
                                NULL, 0)  /* NULL = verification failed */
                            └─> IMA/LSM: يرفض mount الـ device
```

لو الـ roothash في الـ kernel command line مختلف عن اللي اتحسب فعلاً من الـ partition:

```bash
# تحقق من الـ cmdline
cat /proc/cmdline | grep verity
# ... dm-verity.roothash=abc123... ← لازم يطابق الـ partition الفعلي
```

#### الحل

**الخطوة 1: حساب الـ roothash الصحيح:**

```bash
# على build machine بعد كل flash
veritysetup format \
    /path/to/rootfs.img \
    /path/to/hashtree.img \
    | grep "Root hash:" | awk '{print $3}'
# مثلاً: a3f9c2b1...
```

**الخطوة 2: تحديث الـ kernel cmdline في U-Boot:**

```bash
# في U-Boot environment
setenv bootargs "... dm-verity.roothash=a3f9c2b1... dm-verity.hashdev=/dev/mmcblk0p3"
saveenv
```

**الخطوة 3: التحقق من الـ DM table:**

```bash
# من initramfs
dmsetup table rootfs
# 0 2097152 verity 1 /dev/mmcblk0p2 /dev/mmcblk0p3 4096 4096 ...
```

**الخطوة 4: لو الـ image نفسه corrupt، إعادة build مع roothash صحيح:**

```bash
# Yocto/OE
bitbake core-image-minimal
# الـ dm-verity setup بيتعمل تلقائياً لو IMAGE_CLASSES += "dm-verity-image"
# والـ roothash بيتضاف للـ wks file أو boot script
```

**اختبار `security_bdev_setintegrity` في الـ kernel:**

```c
/* لو بتكتب driver بيعمل integrity check يدوي */
#include <linux/security.h>

struct dm_verity_digest digest = {
    .alg = "sha256",
    .digest = computed_hash,
    .digest_len = SHA256_DIGEST_SIZE,
};

/* إبلاغ الـ LSM بنجاح الـ verification */
ret = security_bdev_setintegrity(bdev,
        LSM_INT_DMVERITY_ROOTHASH,
        &digest,
        sizeof(digest));
if (ret)
    pr_err("LSM rejected integrity report: %d\n", ret);
```

#### الدرس المستفاد
الـ `security_bdev_setintegrity()` و `lsm_integrity_type` enum هم الجسر بين **dm-verity** و **LSM** — الـ LSM ما بيعملش الـ hash verification بنفسه، بس بيتلقى النتيجة ويقرر يسمح بالـ mount أو لأ. في board bring-up، الـ roothash في الـ cmdline لازم يتحدّث مع **كل** build جديد. استخدم automation script في سكريبت الـ flash عشان تضمن التطابق دايماً.
## Phase 7: مصادر ومراجع

---

### توثيق الـ Kernel الرسمي

| المصدر | الرابط |
|--------|--------|
| **LSM: General Security Hooks for Linux** — التوثيق الرسمي الكامل | [docs.kernel.org/security/lsm.html](https://docs.kernel.org/security/lsm.html) |
| **LSM Development Guide** — دليل كتابة LSM جديد | [docs.kernel.org — lsm-development](https://docs.kernel.org/6.2/security/lsm-development.html) |
| **LSM Admin Guide** — إعداد الـ LSMs من الـ userspace | [static.lwn.net/kerneldoc/admin-guide/LSM/](https://static.lwn.net/kerneldoc/admin-guide/LSM/) |
| **BPF LSM Programs** — استخدام eBPF مع الـ LSM hooks | [docs.kernel.org/bpf/prog_lsm.html](https://docs.kernel.org/bpf/prog_lsm.html) |
| **Landlock LSM** — sandboxing بدون امتيازات | [docs.kernel.org/security/landlock.html](https://docs.kernel.org/security/landlock.html) |
| **Landlock userspace API** | [docs.kernel.org/userspace-api/landlock.html](https://docs.kernel.org/userspace-api/landlock.html) |
| **LSM.txt** — النسخة النصية القديمة من التوثيق | [kernel.org/doc/Documentation/security/LSM.txt](https://www.kernel.org/doc/Documentation/security/LSM.txt) |

---

### مقالات LWN.net

دي أهم المقالات على LWN.net اللي بتشرح تطور الـ LSM framework:

| المقال | الوصف |
|--------|-------|
| [LSM stacking and the future](https://lwn.net/Articles/804906/) | تقرير 2019 من Casey Schaufler عن وضع stacking الـ LSMs وخطط المستقبل |
| [A change in direction for security-module stacking?](https://lwn.net/Articles/970070/) | أحدث مقال عن التغيير في نهج الـ stacking وآلية traversal الـ hooks |
| [LSM: Module stacking for AppArmor](https://lwn.net/Articles/891538/) | تفاصيل تقنية عن stacking الـ AppArmor مع LSMs تانية |
| [LSM: Stacking for major security modules](https://lwn.net/Articles/697259/) | كيف يتم الـ stacking بين الـ major LSMs زي SELinux و Smack |
| [Progress in security module stacking](https://lwn.net/Articles/635771/) | تقدم جهود الـ stacking عبر الزمن |
| [Supporting multiple LSMs](https://lwn.net/Articles/426921/) | المقاربة المبكرة لدعم أكتر من LSM في نفس الـ kernel |
| [Adding system calls for Linux security modules](https://lwn.net/Articles/919059/) | إضافة syscalls جديدة للـ LSM framework |
| [Complete coverage in Linux security modules](https://lwn.net/Articles/154277/) | تغطية شاملة لنقاط الـ hooks في الـ kernel |
| [The future of the Linux Security Module API](https://lwn.net/Articles/180194/) | نقاش قديم عن مستقبل الـ LSM API |
| [The return of loadable security modules?](https://lwn.net/Articles/526983/) | نقاش عن إمكانية تحميل الـ LSMs كـ modules في runtime |
| [Linux security non-modules and AppArmor](https://lwn.net/Articles/239962/) | نقاش حول AppArmor وتصنيفها كـ non-module |
| [Kernel runtime security instrumentation](https://lwn.net/Articles/798157/) | مقال عن KRSI وكيف بيستخدم BPF مع LSM hooks |
| [Landlock LSM: Unprivileged sandboxing](https://lwn.net/Articles/698226/) | شرح Landlock كـ LSM للـ sandboxing بدون root |
| [mmap_min_addr and security modules](https://lwn.net/Articles/342450/) | تفاعل الـ LSM مع حماية الـ mmap |
| [More hooks for kernel events](https://lwn.net/Articles/122764/) | إضافة hooks جديدة لأحداث الـ kernel |
| [selinux: add hooks for key subsystem](https://lwn.net/Articles/186082/) | إضافة SELinux hooks لنظام الـ keys |
| [security: add fault injection to LSM hooks](https://lwn.net/Articles/836902/) | إضافة fault injection لاختبار الـ LSM hooks |
| [Disabling SELinux's runtime disable](https://lwn.net/Articles/927463/) | نقاش إزالة runtime disable في SELinux |
| [Add security hooks to binder](https://lwn.net/Articles/630503/) | إضافة LSM hooks لنظام Binder في Android |

---

### نقاشات الـ Mailing List

| المصدر | الوصف |
|--------|-------|
| [LKML.org](https://lkml.org/) | الأرشيف الرئيسي للـ Linux Kernel Mailing List — ابحث فيه بكلمات زي `LSM hooks` أو `security_ops` |
| [LSM Stacking Patch Series — lore.kernel.org](https://lore.kernel.org/selinux/20190602165101.25079-14-casey@schaufler-ca.com/t/) | سلسلة patches ضخمة من Casey Schaufler لـ stacking الـ LSMs مع AppArmor (58 patch) |
| [Landlock mailing list — kernsec.org](http://kernsec.org/pipermail/linux-security-module-archive/2021-May/027004.html) | إعلان قائمة بريد مخصصة لنقاشات Landlock في الـ userspace |
| [LSM blob and hooks for namespaces — LKML 2026](https://lkml.org/lkml/2026/2/17/962) | نقاش حديث عن إضافة LSM blobs وhooks للـ namespaces |
| [Dynamic LSM hooks loading — Patchwork](https://patchwork.kernel.org/project/linux-security-module/patch/9ffcc4fb12bcdb4785be013b51b5177069cd2686.1520407240.git.sargun@sargun.me/) | patch تجريبي لتحميل LSM hooks ديناميكيًا في runtime |

---

### الورقة البحثية الأصلية للـ LSM

الـ **LSM framework** اتوصف الأول في ورقة بحثية رسمية:

> **"Linux Security Modules: General Security Support for the Linux Kernel"**
> Chris Wright, Crispin Cowan, et al. — **USENIX Security 2002**
> [تحميل PDF من kroah.com](http://www.kroah.com/linux/talks/ols_2002_lsm_paper/lsm.pdf)

الورقة دي هي المرجع الأصلي اللي بيشرح فلسفة التصميم، وليه اتعملت hooks بدل ما الـ SELinux يتدمج مباشرة في الـ kernel.

---

### مصادر kernelnewbies.org

| المصدر | الوصف |
|--------|-------|
| [KernelGlossary — LSM](https://kernelnewbies.org/KernelGlossary) | تعريف مختصر للـ LSM وارتباطه بنظام الأمان |
| [Linux 2.6.18 Changes](https://kernelnewbies.org/Linux_2_6_18) | LSM hooks اتأضيفت لـ subsystems جديدة في 2.6.18 |
| [Linux 2.6.15 Changes](https://kernelnewbies.org/Linux_2_6_15) | إضافة LSM hooks لنظام الـ key management |
| [Linux 2.6.19 Changes](https://kernelnewbies.org/Linux_2_6_19) | دعم NetLabel لـ LSM developers وحذف BSD secure level module |
| [Linux 4.9 Changes](https://kernelnewbies.org/Linux_4.9) | تحسينات على الـ LSM hook implementations |

---

### مصادر elinux.org

| المصدر | الوصف |
|--------|-------|
| [Security Working Group](https://elinux.org/Security_Working_Group) | مجموعة العمل المعنية بأمان Linux في الأجهزة المدمجة |
| [Security Presentations](https://elinux.org/Security_Presentations) | عروض تقديمية عن LSM وأمان الـ embedded Linux |
| [TOMOYO Linux](https://elinux.org/TOMOYO_Linux) | شرح TOMOYO كـ MAC implementation مبني على LSM |
| [Security Hardware Resources](https://elinux.org/Security_Hardware_Resources) | موارد الـ hardware security: TPM، ARM TrustZone مع LSM |
| [SE for Android (PDF)](https://elinux.org/images/b/b1/Se_for_android--smalley.pdf) | شرح SELinux for Android وكيف بيستخدم LSM hooks |

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل المهم**: Chapter 5 — Concurrency and Race Conditions (بيشرح locking اللي بيتعامل معاه الـ LSM)
- **تحميل مجاني**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- الكتاب مش بيتكلم عن LSM مباشرة، بس بيبني الفهم الأساسي لبنية الـ kernel اللي عليها الـ LSM اتبنى

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Devices and Modules — بيشرح كيف الـ kernel بيدعم extensions
- **الفصل 5**: System Calls — مهم لفهم نقاط الـ hook في الـ syscall path
- بيغطي بنية الـ `security_operations` وفكرة الـ security fields في الـ kernel data structures

#### Understanding the Linux Kernel — Bovet & Cesati
- **الفصل 20**: Program Execution — بيشرح نقاط الـ hook في `linux_binprm` اللي بتستخدمها الـ LSM hooks زي `security_bprm_check`
- **الفصل 12**: The Virtual Filesystem — بيشرح الـ inode hooks اللي بيتعامل معاها `security_inode_*`

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 18**: إعداد SELinux وAppArmor على الأنظمة المدمجة
- بيشرح الفرق بين الـ MAC policies وكيف تختار الـ LSM المناسب لجهازك

#### The Linux Programming Interface — Michael Kerrisk
- **الفصل 38**: Writing Secure Privileged Programs — بيشرح الـ capabilities اللي بيراقبها `security_capable()`
- مرجع ممتاز لفهم الـ POSIX capabilities قبل ما تتعمق في LSM hooks

---

### مسارات التوثيق في الـ Kernel Source

```
Documentation/
├── security/
│   ├── lsm.rst              # الوثيقة الرئيسية للـ LSM framework
│   ├── lsm-development.rst  # دليل كتابة LSM جديد
│   ├── landlock.rst         # Landlock LSM
│   ├── SELinux.rst          # SELinux documentation
│   ├── apparmor.rst         # AppArmor documentation
│   └── Smack.rst            # Smack LSM
├── admin-guide/
│   └── LSM/
│       ├── index.rst        # نظرة عامة على إعداد الـ LSMs
│       ├── apparmor.rst
│       ├── landlock.rst
│       ├── SELinux.rst
│       └── Smack.rst
└── bpf/
    └── prog_lsm.rst         # BPF LSM programs
```

الملف الأساسي:
```
include/linux/security.h     # تعريف كل الـ LSM hooks والـ API
security/security.c          # تنفيذ الـ hook dispatch mechanism
security/Kconfig             # خيارات إعداد الـ LSM في kernel config
```

---

### Commits تاريخية مهمة في الـ LSM

| الحدث | التفاصيل |
|-------|---------|
| **2001** — بداية الـ LSM framework | WireX Communications بالتعاون مع Greg Kroah-Hartman وJames Morris |
| **2003** — اندماج في الـ mainline kernel | الـ LSM framework اتدمج رسميًا في kernel 2.6 |
| **2007** — إضافة Yama LSM | أول minor LSM يُعتمد بعد الـ framework الأصلي |
| **Linux 4.2** — beg of LSM blobs | بداية استخدام الـ security blobs في الـ data structures |
| **Linux 5.1** — Landlock (RFC) | أول RFC للـ Landlock LSM |
| **Linux 5.7** — BPF LSM | دعم eBPF programs كـ LSM hooks |
| **Linux 5.13** — Landlock merged | اندماج Landlock رسميًا في الـ mainline |

للبحث عن commits محددة:
```bash
git log --oneline --all -- security/security.c | head -30
git log --oneline --all -- include/linux/security.h | head -30
git log --grep="LSM" --oneline -20
```

---

### مصطلحات البحث

استخدم الكلمات دي للبحث عن معلومات إضافية:

```
linux security module LSM hooks
security_operations struct linux kernel
LSM stacking multiple security modules
SELinux hooks implementation
BPF LSM programs eBPF security
Landlock unprivileged sandboxing linux
linux capabilities security_capable
inode security hooks linux kernel
linux mandatory access control MAC
security_hook_list kernel
lsm_blob_sizes linux kernel
```

للبحث في الـ kernel source مباشرة:
```bash
# كل الـ security hooks المُعرَّفة
grep -r "DEFINE_LSM_HOOK\|LSM_HOOK_INIT" security/

# كل الـ hook calls في الـ kernel
grep -r "security_[a-z_]*(" fs/ mm/ net/ kernel/ | grep -v security.c

# الـ modules المسجّلة
cat /sys/kernel/security/lsm
```

---

### روابط سريعة مرتبة بالأولوية

| الأولوية | المصدر | السبب |
|---------|--------|-------|
| ⭐⭐⭐ | [docs.kernel.org/security/lsm.html](https://docs.kernel.org/security/lsm.html) | التوثيق الرسمي — ابدأ منه |
| ⭐⭐⭐ | [USENIX Paper (PDF)](http://www.kroah.com/linux/talks/ols_2002_lsm_paper/lsm.pdf) | الورقة الأصلية — الفلسفة والتصميم |
| ⭐⭐⭐ | [lwn.net LSM stacking](https://lwn.net/Articles/804906/) | أشمل مقال لفهم تطور الـ framework |
| ⭐⭐ | [docs.kernel.org/bpf/prog_lsm.html](https://docs.kernel.org/bpf/prog_lsm.html) | BPF LSM — الاتجاه الحديث |
| ⭐⭐ | [docs.kernel.org/security/landlock.html](https://docs.kernel.org/security/landlock.html) | Landlock — أحدث LSM في الـ mainline |
| ⭐⭐ | [lwn.net — change in direction](https://lwn.net/Articles/970070/) | أحدث نقاش عن مستقبل الـ stacking |
| ⭐ | [elinux.org Security Working Group](https://elinux.org/Security_Working_Group) | منظور الـ embedded systems |
| ⭐ | [kernelnewbies.org Glossary](https://kernelnewbies.org/KernelGlossary) | تعريفات سريعة للمبتدئين |
## Phase 8: Writing simple module

### الفكرة

**`register_blocking_lsm_notifier()`** — دي الـ function اللي بتسجل **notifier_block** عشان تستقبل أحداث الـ LSM (Linux Security Module) زي `LSM_POLICY_CHANGE`. ده hook آمن ومثير للاهتمام لأنه بيخليك تتابع وقت ما أي LSM بيغيّر الـ policy بتاعته (مثلاً SELinux أو AppArmor).

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * lsm_notify_demo.c
 *
 * Registers a blocking LSM notifier to observe LSM policy-change events.
 * Uses the register_blocking_lsm_notifier() API from linux/security.h.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit macros     */
#include <linux/kernel.h>       /* pr_info()                                   */
#include <linux/notifier.h>     /* struct notifier_block, notifier chain API   */
#include <linux/security.h>     /* register/unregister_blocking_lsm_notifier,
                                   enum lsm_event                              */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("Demo: hook LSM policy-change events via blocking notifier");

/* ------------------------------------------------------------------ */
/* Callback — called synchronously when an LSM fires a policy event   */
/* ------------------------------------------------------------------ */
static int lsm_event_handler(struct notifier_block *nb,
                              unsigned long event,   /* enum lsm_event value  */
                              void *data)            /* event-specific payload */
{
    /*
     * event == LSM_POLICY_CHANGE  → an LSM just changed its policy
     * event == LSM_STARTED_ALL    → all LSMs finished initialization
     *
     * 'data' is currently NULL for both events — reserved for future use.
     */
    switch (event) {
    case LSM_POLICY_CHANGE:
        pr_info("lsm_notify_demo: LSM_POLICY_CHANGE received "
                "(pid=%d comm=\"%s\")\n",
                current->pid, current->comm);
        break;
    case LSM_STARTED_ALL:
        pr_info("lsm_notify_demo: LSM_STARTED_ALL — "
                "all LSMs are now active\n");
        break;
    default:
        pr_info("lsm_notify_demo: unknown LSM event %lu\n", event);
        break;
    }

    /* NOTIFY_OK tells the notifier chain to continue calling other handlers */
    return NOTIFY_OK;
}

/* ------------------------------------------------------------------ */
/* The notifier_block that ties our callback into the LSM chain        */
/* ------------------------------------------------------------------ */
static struct notifier_block lsm_nb = {
    .notifier_call = lsm_event_handler,
    .priority      = 0,   /* default priority — called after higher-priority handlers */
};

/* ------------------------------------------------------------------ */
/* module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init lsm_notify_demo_init(void)
{
    int ret;

    ret = register_blocking_lsm_notifier(&lsm_nb);
    if (ret) {
        pr_err("lsm_notify_demo: failed to register notifier (%d)\n", ret);
        return ret;
    }

    pr_info("lsm_notify_demo: loaded — watching for LSM events\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit lsm_notify_demo_exit(void)
{
    unregister_blocking_lsm_notifier(&lsm_nb);
    pr_info("lsm_notify_demo: unloaded — notifier removed\n");
}

module_init(lsm_notify_demo_init);
module_exit(lsm_notify_demo_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module: `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info()` / `pr_err()` لطباعة الـ kernel log |
| `linux/notifier.h` | `struct notifier_block` وكل الـ API بتاعت notifier chains |
| `linux/security.h` | `register_blocking_lsm_notifier()` و `enum lsm_event` المُعرَّفَين في الملف ده |

الـ `linux/security.h` هو قلب الموضوع — بيعرّف الـ LSM hook API بالكامل وبيوفر الـ `register_blocking_lsm_notifier()` اللي بنستخدمها.

---

#### الـ Callback: `lsm_event_handler`

```c
static int lsm_event_handler(struct notifier_block *nb,
                              unsigned long event,
                              void *data)
```

**الـ `nb`** — pointer للـ `notifier_block` المسجّل، مفيد لو عندك أكتر من notifier تعرف مين اتنادى.
**الـ `event`** — قيمة من `enum lsm_event`: إما `LSM_POLICY_CHANGE` أو `LSM_STARTED_ALL` — بيحدد نوع الحدث اللي وقع.
**الـ `data`** — payload إضافي، حالياً `NULL` للحدثين دول، موجود للتوسعة المستقبلية.

الـ callback بيطبع `current->pid` و `current->comm` عشان تعرف مين (أي process) كان السبب في الـ policy change — معلومة مفيدة جداً لـ audit أو debugging.

الـ return value `NOTIFY_OK` بيقول للـ notifier chain "استكمل وادّي الـ event للـ handlers التانية".

---

#### الـ `notifier_block`

```c
static struct notifier_block lsm_nb = {
    .notifier_call = lsm_event_handler,
    .priority      = 0,
};
```

**الـ `struct notifier_block`** ده اللي بيربط الـ callback بالـ chain — الـ kernel بيمشي على الـ chain ويناديها بالترتيب حسب الـ `priority` (الأعلى أول).
**الـ `priority = 0`** يعني الـ handler ده بيتنادى في مكانه الطبيعي من غير تقديم أو تأخير.

---

#### الـ `module_init`

```c
ret = register_blocking_lsm_notifier(&lsm_nb);
```

**الـ `register_blocking_lsm_notifier()`** — بتضيف الـ `notifier_block` بتاعنا في الـ blocking LSM notifier chain اللي الـ kernel بيستخدمها عشان يُعلم كل الـ listeners لما أي LSM يغيّر الـ policy.
"blocking" يعني الـ notifier chain دي بتتنفذ في سياق قابل للـ sleep (process context)، عكس atomic notifiers اللي محظور فيها الـ sleep.

---

#### الـ `module_exit`

```c
unregister_blocking_lsm_notifier(&lsm_nb);
```

**الـ unregister ضروري جداً في exit** — لو نسيته والـ module اتحمل من الـ memory، الـ kernel هيحاول يناديه تاني في أول policy change وهيحصل **kernel panic** (use-after-free).
دايماً: كل `register` لازم يقابله `unregister` في الـ cleanup path.

---

### طريقة التجربة

```bash
# بناء الـ module (Makefile بسيط في نفس الـ directory)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod lsm_notify_demo.ko

# شوف الـ log
dmesg | tail -5

# عشان تشوف الـ LSM_POLICY_CHANGE عملياً مع SELinux:
sudo setenforce 0   # أو أي تغيير في الـ policy
dmesg | grep lsm_notify_demo

# إزالة الـ module
sudo rmmod lsm_notify_demo
```

**ملحوظة:** الـ `LSM_POLICY_CHANGE` event بيتبعث مثلاً لما `semanage` أو `load_policy` بيغير الـ SELinux policy، أو لما AppArmor profile بيتحمل — يعني في الـ production بتشوف الـ event ده بشكل حقيقي.
