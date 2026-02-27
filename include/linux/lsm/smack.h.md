## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ LSM (Linux Security Module)؟

تخيل إن الـ kernel هو مدير مبنى كبير — كل ما يحصل فيه (فتح ملف، إرسال packet، إنشاء process) لازم يعدي على "بوابة أمان". الـ **Linux Security Module (LSM)** هو إطار عمل داخل الـ kernel بيسمح بوضع أكثر من "حارس أمن" على نفس البوابة في نفس الوقت — كل واحد بأسلوبه الخاص.

الـ LSM مش security policy بحد ذاته — هو **hook framework**: بيوفر نقاط ربط (hooks) على كل عملية حساسة في الـ kernel، وأي security module عايز يشتغل بيسجّل نفسه على الـ hooks دي.

---

### ما هو الـ Smack؟

الـ **Smack (Simplified Mandatory Access Control Kernel)** هو أحد الـ LSM implementations — نظام تحكم إجباري (Mandatory Access Control) مبسّط مقارنةً بالـ SELinux. فكرته الأساسية: كل process وكل ملف وكل resource في النظام عنده **label** (وسم/تصنيف)، والنظام بيسمح أو بيمنع العمليات بناءً على قواعد بين الـ labels دي.

**مثال بسيط:**
- process عنده label `"web-server"` — يقدر يقرأ من ملفات label `"web-content"` بس.
- لو حاول يفتح ملف label `"secret-config"` — الـ Smack بيرفض.

الـ label في Smack ما هو إلا string قصير (أقل من 256 حرف)، بس داخلياً في الـ kernel يُمثَّل كـ pointer لـ struct اسمه `smack_known`.

---

### دور ملف `include/linux/lsm/smack.h`

هنا بتجي الفكرة الأذكى في الـ LSM الجديد: الـ kernel أحياناً محتاج **يمرر هوية أمنية** (security identity) بين الـ subsystems المختلفة — مثلاً من الـ network stack للـ audit subsystem، أو من الـ VFS لنظام الـ IPC.

المشكلة: كل LSM بيعرّف هوية المستخدم بطريقته:
- الـ SELinux: رقم `u32` اسمه `secid`.
- الـ AppArmor: pointer لـ `struct aa_label`.
- الـ **Smack**: pointer لـ `struct smack_known`.
- الـ BPF LSM: رقم `u32`.

لو كل subsystem في الـ kernel اتعامل مع كل LSM بشكل منفصل، الـ code هيتحول لـ سباغيتي. الحل؟

**`struct lsm_prop`** — container موحّد بيحتوي على هوية كل LSM في مكان واحد. وكل ملف في مجلد `include/linux/lsm/` بيعرّف **القطعة الخاصة بـ LSM معين** داخل هذا الـ container.

ملف `smack.h` تحديداً بيعرّف:

```c
struct smack_known;  /* forward declaration — التعريف الكامل في security/smack/smack.h */

struct lsm_prop_smack {
#ifdef CONFIG_SECURITY_SMACK
    struct smack_known *skp;  /* pointer to the Smack label entry */
#endif
};
```

لو الـ Smack مش مفعّل في الـ kernel (`CONFIG_SECURITY_SMACK` مش موجود) — الـ struct تبقى فاضية تماماً، zero overhead.

---

### كيف تترابط الأجزاء مع بعض؟

```
include/linux/security.h
  ├── #include <linux/lsm/selinux.h>   → struct lsm_prop_selinux { u32 secid; }
  ├── #include <linux/lsm/smack.h>     → struct lsm_prop_smack  { struct smack_known *skp; }
  ├── #include <linux/lsm/apparmor.h>  → struct lsm_prop_apparmor { struct aa_label *label; }
  └── #include <linux/lsm/bpf.h>       → struct lsm_prop_bpf   { u32 secid; }

struct lsm_prop {
    struct lsm_prop_selinux  selinux;
    struct lsm_prop_smack    smack;    ← القطعة اللي smack.h بيعرّفها
    struct lsm_prop_apparmor apparmor;
    struct lsm_prop_bpf      bpf;
};
```

الـ `struct lsm_prop` ده بيتمرر في دوال زي:
- `security_cred_getlsmprop()` — جيب الهوية الأمنية لـ credentials
- `security_inode_getlsmprop()` — جيب الهوية الأمنية لـ inode
- `security_task_getlsmprop_obj()` — جيب هوية task
- `security_lsmprop_to_secctx()` — حوّل الهوية لـ text context

---

### القصة الكاملة (لماذا هذا التصميم؟)

قديماً كان في مفهوم `secid` — رقم `u32` واحد بيمثّل الهوية الأمنية. المشكلة: مش كل LSM بيشتغل بـ u32، والـ Smack لازم يعدّي pointer، مش رقم. لو اتعملت mapping من pointer لـ u32 وبالعكس كل مرة، هيكون overhead وتعقيد.

الحل الجديد: `struct lsm_prop` بيجمع هويات كل الـ LSMs المفعّلة في struct واحدة. كل LSM بيحط قطعته الخاصة — والـ Smack بيحط pointer مباشر لـ `struct smack_known` اللي فيه كل المعلومات عن الـ label (الاسم، الـ secid، قواعد الوصول، الـ CIPSO network label).

---

### الملفات المهمة في هذا الـ Subsystem

| الملف | الدور |
|---|---|
| `include/linux/lsm/smack.h` | تعريف `lsm_prop_smack` (هذا الملف) |
| `include/linux/lsm/selinux.h` | تعريف `lsm_prop_selinux` |
| `include/linux/lsm/apparmor.h` | تعريف `lsm_prop_apparmor` |
| `include/linux/lsm/bpf.h` | تعريف `lsm_prop_bpf` |
| `include/linux/security.h` | يجمع كل الـ lsm headers ويعرّف `struct lsm_prop` والـ API |
| `include/linux/lsm_hooks.h` | تعريف الـ hooks اللي كل LSM بيسجّل عليها |
| `include/uapi/linux/lsm.h` | واجهة الـ userspace مع الـ LSM |
| `security/smack/smack.h` | التعريف الكامل لـ `struct smack_known` والـ Smack internals |
| `security/smack/smack_lsm.c` | تنفيذ الـ hooks الخاصة بالـ Smack (الـ core logic) |
| `security/smack/smack_access.c` | منطق الـ access control بين الـ labels |
| `security/smack/smackfs.c` | الـ filesystem interface (securityfs) لإدارة الـ Smack labels والـ rules |
| `security/smack/smack_netfilter.c` | تكامل الـ Smack مع الـ netfilter لـ packet labeling |
| `Documentation/admin-guide/LSM/Smack.rst` | الـ user-facing documentation |
## Phase 2: شرح الـ LSM (Linux Security Module) Framework وموقع Smack فيه

### المشكلة اللي بيحلها الـ LSM Framework

الـ Linux kernel في الأصل بيعتمد على **DAC (Discretionary Access Control)** — يعني التحكم في الوصول بيعتمد على الـ UID/GID والـ permissions bits اللي بتشوفها في `ls -l`. ده النموذج الكلاسيكي اللي Unix اتبنى عليه.

المشكلة إن DAC مش كفاية في بيئات الـ embedded أو enterprise اللي محتاجة **MAC (Mandatory Access Control)** — يعني:
- الـ root نفسه يكون مقيد بـ policy.
- العمليات تتعزل عن بعض بغض النظر عن الـ UID.
- تطبيق سياسة أمان مركزية مش بتعتمد على برامج userspace تتصرف صح.

المشكلة التانية: كل نظام أمان (SELinux, AppArmor, Smack) كان هيحتاج يعدل في جوف الـ kernel نفسه — الـ VFS، الـ networking، الـ IPC — وده كان هيخلي الـ kernel مرتبط بـ implementation واحدة.

الحل كان إنهم يعملوا **abstraction layer** يفصل الـ policy logic عن الـ kernel core.

---

### الحل: الـ LSM Framework

الـ **LSM (Linux Security Module)** هو hook-based framework اتضاف للـ kernel سنة 2003. الفكرة بسيطة وعبقرية:

> في كل نقطة حساسة في الـ kernel (فتح ملف، fork عملية، إرسال packet)، الـ kernel بيستدعي دالة `security_*()`، اللي بدورها بتروح تنادي على كل الـ LSMs المسجلة بالترتيب. لو أي واحد رفض → العملية بترفض.

الـ kernel core مش عارف أي LSM شغّال — بس بيمشي الـ hooks.

---

### تشبيه من الواقع (مع Mapping كامل)

تخيل **مطار دولي** فيه أنظمة أمان متعددة:

| عنصر المطار | المقابل في الـ Kernel |
|---|---|
| الراكب | Process / Task |
| الـ passport | `struct cred` (credentials) |
| الـ boarding pass | الـ security label (Smack label) |
| بوابة الأمان بتاعة الجوازات | LSM hook في الـ VFS (مثلاً `security_inode_permission`) |
| جهاز X-ray الأمتعة | LSM hook في الـ networking (`security_socket_sendmsg`) |
| نظام الجوازات (أمريكي) | SELinux — بيشتغل بـ secid (رقم) يرمز لـ context |
| نظام التأشيرات البريطاني | Smack — بيشتغل بـ string labels مباشرة |
| نظام AppArmor | AppArmor — profile-based، بيتعامل مع paths |
| الـ IATA (التنسيق بين كل الأنظمة) | LSM Framework نفسه |
| قانون دولي ثابت لكل المطارات | الـ kernel hooks (دايماً موجودة بغض النظر عن النظام) |

الـ LSM بيضمن إن كل الأنظمة دي تتسأل في ترتيب محدد، وإن أي رفض = رفض نهائي.

---

### الصورة الكبيرة: Architecture

```
User Space
    │
    │  syscall: open("/etc/shadow", O_RDONLY)
    ▼
┌─────────────────────────────────────────────────────┐
│                  Kernel VFS Layer                   │
│  vfs_open() → inode_permission()                   │
└──────────────────────┬──────────────────────────────┘
                       │ calls security hook
                       ▼
┌─────────────────────────────────────────────────────┐
│           security_inode_permission()               │
│         (generic LSM dispatch function)             │
└──────────────────────┬──────────────────────────────┘
                       │ iterates static_calls_table
                       ▼
        ┌──────────────────────────────┐
        │   lsm_static_calls_table     │
        │  .inode_permission[0] ──────►│ capability_inode_permission()
        │  .inode_permission[1] ──────►│ smack_inode_permission()
        │  .inode_permission[2] ──────►│ selinux_inode_permission()
        └──────────────────────────────┘
                       │
          كل واحد بيرجع 0 (allow) أو -EACCES (deny)
          أي deny → الـ syscall يفشل فوراً
```

---

### أين يقع Smack في الـ LSM Framework؟

الـ **Smack (Simplified Mandatory Access Control Kernel)** هو واحد من الـ LSMs المدمجة في الـ kernel. هو مصمم خصيصاً للبيئات اللي محتاجة:
- سياسة بسيطة وسريعة (IoT، embedded automotive).
- تشغيل على أنظمة محدودة الموارد.
- استخدام **string labels** مقروءة بدل أرقام أو binary policies.

---

### الـ Core Abstraction: الـ `lsm_prop_smack` و `smack_known`

الملف `include/linux/lsm/smack.h` بيعرّف بيت واحد بس:

```c
/* Public interface: what Smack exposes to the rest of the kernel */
struct lsm_prop_smack {
#ifdef CONFIG_SECURITY_SMACK
    struct smack_known *skp;  /* pointer into the global label list */
#endif
};
```

الـ `struct lsm_prop_smack` هو الـ **public face** بتاع Smack — اللي بيظهره للـ kernel subsystems التانية. بس الحقيقة الحلوة في `struct smack_known` الداخلية:

```c
/* Internal: the real Smack label object */
struct smack_known {
    struct list_head        list;          /* global list of all labels */
    struct hlist_node       smk_hashed;   /* hash table for fast lookup */
    char                   *smk_known;    /* the actual label string e.g. "System" */
    u32                     smk_secid;    /* numeric alias for legacy interfaces */
    struct netlbl_lsm_secattr smk_netlabel; /* CIPSO wire label for networking */
    struct list_head        smk_rules;    /* access rules for this subject */
    struct mutex            smk_rules_lock;
};
```

**الـ `smack_known` هو قلب الـ Smack بالكامل.** كل label في النظام = entry واحد في الـ global list. الـ pointer للـ entry ده هو اللي بيتخزن في كل مكان (inode، task، socket).

---

### الـ Blob System: كيف Smack بيخزن بياناته في كل Object

الـ LSM Framework مش بيدي كل LSM struct منفصل لكل `inode` أو `cred`. بدل كده، بيعمل **blob** — buffer جنب الـ object الأصلي، كل LSM بياخد منه offset.

ده مفهوم مهم جداً، نشرحه بالرسمة:

```
struct inode في الذاكرة:
┌──────────────────────────────────────────────────────┐
│  i_ino, i_mode, i_uid, i_gid ...  (inode fields)   │
│  ...                                                 │
│  void *i_security  ────────────────────────────────► │
└──────────────────────────────────────────────────────┘
                                                       │
                                    ┌──────────────────▼──────────────────┐
                                    │         security blob               │
                                    │  [offset 0] → SELinux data (u32)   │
                                    │  [offset 4] → Smack inode_smack ptr │
                                    │  [offset 8] → AppArmor data         │
                                    └─────────────────────────────────────┘

// Smack بيوصل لبياناته عن طريق:
static inline struct inode_smack *smack_inode(const struct inode *inode)
{
    return inode->i_security + smack_blob_sizes.lbs_inode;
    // يعني: ابدأ من الـ blob، واجري بـ offset محدد لـ Smack
}
```

الـ `lsm_blob_sizes` بيحدد حجم الـ blob اللي كل LSM محتاجه في كل نوع object:

```c
struct lsm_blob_sizes {
    unsigned int lbs_cred;       /* size for cred blob */
    unsigned int lbs_file;       /* size for file blob */
    unsigned int lbs_inode;      /* size for inode blob */
    unsigned int lbs_sock;       /* size for socket blob */
    unsigned int lbs_superblock; /* size for superblock blob */
    unsigned int lbs_task;       /* size for task blob */
    /* ... etc */
};
```

---

### الـ Structs اللي Smack بيخزنها في كل Object

```
┌─────────────────────────────────────────────────────────────────┐
│                        Smack Security Objects                   │
│                                                                 │
│  task_struct                                                    │
│  └── cred->security ──► task_smack {                          │
│                            smk_task*    → smack_known "System" │
│                            smk_forked*  → smack_known "System" │
│                            smk_rules    (per-task overrides)   │
│                         }                                       │
│                                                                 │
│  inode                                                          │
│  └── i_security ──────► inode_smack {                         │
│                            smk_inode* → smack_known "Object"  │
│                            smk_task*  → creator's label        │
│                            smk_mmap*  → mmap domain label      │
│                         }                                       │
│                                                                 │
│  sock                                                           │
│  └── sk_security ─────► socket_smack {                        │
│                            smk_out* → outbound label           │
│                            smk_in*  → inbound label            │
│                            smk_packet* → TCP peer label        │
│                         }                                       │
│                                                                 │
│  super_block                                                    │
│  └── s_security ──────► superblock_smack {                    │
│                            smk_root*    → root dir label       │
│                            smk_floor*   → floor label          │
│                            smk_hat*     → hat label            │
│                            smk_default* → default label        │
│                         }                                       │
└─────────────────────────────────────────────────────────────────┘
             │
             │  كلهم بيشاوروا على ──►
             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Global smack_known_list                       │
│                                                                 │
│  smack_known_floor ──► smack_known { "Floor",  secid=1, ... } │
│  smack_known_hat   ──► smack_known { "Hat",    secid=2, ... } │
│  smack_known_star  ──► smack_known { "*",      secid=3, ... } │
│  smack_known_web   ──► smack_known { "Web",    secid=4, ... } │
│  "System"          ──► smack_known { "System", secid=5, ... } │
│  "App1"            ──► smack_known { "App1",   secid=6, ... } │
│         ...                                                     │
│                                                                 │
│  Hash table: smack_known_hash[16] للـ fast lookup              │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Access Control Model: كيف Smack بيقرر

الـ Smack بيعتمد على نموذج **subject → object** مع access type:

```
Process (subject label: "App1") wants to read File (object label: "Config")

smk_access(subject="App1", object="Config", access=MAY_READ, audit_info)
    │
    ├── بيدور في smk_rules list بتاعة "App1"
    │   على entry بيقول: App1 → Config : r (read allowed)
    │
    ├── لو لقى rule تسمح → return 0 (allow)
    └── لو ملقاش → return -EACCES (deny)
```

الـ access types الممكنة في Smack:
```
MAY_READ    (r) - قراءة
MAY_WRITE   (w) - كتابة
MAY_EXEC    (x) - تنفيذ
MAY_APPEND  (a) - إضافة
MAY_TRANSMUTE (t) - تغيير label في directory
MAY_LOCK    (l) - file locking
MAY_BRINGUP (b) - bringup mode (logging فقط)
```

---

### تسجيل الـ LSM في الـ Framework: الـ `lsm_info` و Static Calls

الـ **LSM Framework** (مش Smack بالذات) بيوفر الـ infrastructure دي:

```c
/* كل LSM بيعرّف نفسه بـ struct lsm_info */
struct lsm_info {
    const struct lsm_id *id;        /* الاسم والـ ID الرسمي */
    enum lsm_order order;           /* ترتيب التنفيذ بين الـ LSMs */
    unsigned long flags;
    struct lsm_blob_sizes *blobs;   /* كام byte محتاج في كل blob */
    int *enabled;
    int (*init)(void);              /* دالة التهيئة */
    /* ... initcalls for different boot stages */
};

/* Smack بيسجل نفسه عن طريق الـ DEFINE_LSM macro */
DEFINE_LSM(smack) = {
    .id   = &smack_lsmid,
    .init = smack_init,
    .blobs = &smack_blob_sizes,
};
```

لما الـ kernel بيبوت، بيمشي على كل الـ `lsm_info` structs في الـ `.lsm_info.init` section ويهيئ كل واحد بالترتيب.

---

### الـ Static Calls Optimization

الـ **static calls** هو مفهوم من subsystem تاني (مهم نفهمه هنا):

> الـ **Static Call** هو آلية بتخلي الـ indirect function call (pointer) يتحول لـ direct call بعد الـ boot، بدل ما يمشي كل مرة في pointer lookup. ده بيقلل الـ overhead خصوصاً في الـ hot paths زي `inode_permission`.

```c
/* كل hook بياخد array من الـ static calls */
struct lsm_static_calls_table {
    /* لكل hook موجود في lsm_hook_defs.h، في array */
    struct lsm_static_call inode_permission[MAX_LSM_COUNT];
    struct lsm_static_call socket_sendmsg[MAX_LSM_COUNT];
    /* ... إلخ */
};

struct lsm_static_call {
    struct static_call_key *key;
    void *trampoline;
    struct security_hook_list *hl;
    struct static_key_false *active;  /* هل الـ slot ده فيه LSM؟ */
};
```

الـ calls بتتملى من الآخر للأول — يعني لو عندك 2 LSMs، الـ slot الأعلى فيه الأول يتنفذ. ده بيخلي الـ dispatch يبدأ مباشرة من أول slot مفعّل.

---

### الـ `lsm_prop_smack` والـ `lsm_prop_*` Family: الـ Public Interface

الملف الأصلي `include/linux/lsm/smack.h` بيحدد `lsm_prop_smack` اللي هي جزء من نمط عام:

```
include/linux/lsm/
├── smack.h    → struct lsm_prop_smack { struct smack_known *skp; }
├── selinux.h  → struct lsm_prop_selinux { u32 secid; }
├── apparmor.h → struct lsm_prop_apparmor { struct aa_label *label; }
└── bpf.h      → struct lsm_prop_bpf { u32 secid; }
```

كل LSM بيعرّض **minimal public struct** — اللي الـ kernel subsystems التانية (زي الـ networking أو الـ audit) ممكن تستخدمه من غير ما تعرف تفاصيل الـ LSM الداخلية. ده الـ encapsulation الحقيقي.

المقارنة:
| LSM | الـ identifier اللي بيستخدمه |
|---|---|
| **Smack** | `struct smack_known *` — pointer للـ label object نفسه |
| **SELinux** | `u32 secid` — رقم يمثل الـ security context |
| **AppArmor** | `struct aa_label *` — pointer للـ profile label |
| **BPF LSM** | `u32 secid` — رقم |

Smack اختار الـ **pointer approach** عن قصد — بدل ما يحتاج lookup كل مرة من رقم لـ label object، الـ pointer بيوديك مباشرة.

---

### ما اللي الـ LSM Framework بيمتلكه vs ما بيفوّضه للـ LSM

| المسؤولية | اللي بيعمله الـ LSM Framework | اللي بيعمله Smack (أو أي LSM) |
|---|---|---|
| **Hook registration** | بيوفر `security_add_hooks()` و الـ static call table | بيسجل hooks بتاعته في الـ table |
| **Blob allocation** | بيحسب الـ offsets وبيخصص الذاكرة في كل object | بيقول محتاج كام byte (`lsm_blob_sizes`) |
| **Dispatch** | بيستدعي الـ LSMs بالترتيب في كل `security_*()` call | لا دخل له — بس بيستجيب لما يتنادى عليه |
| **Policy logic** | لا علاقة له | بيشيل كل الـ policy (label lookup، rule check) |
| **Label storage** | بيوفر الـ blob slots | بيحدد الـ struct (`smack_known`, etc.) |
| **Audit** | بيوفر الـ `common_audit_data` infrastructure | بيملي الـ Smack-specific audit fields |
| **Network labeling** | لا علاقة له | بيتكامل مع NetLabel/CIPSO بنفسه |
| **Ordering** | بيطبق الـ order (FIRST/MUTABLE/LAST) | بيحدد ترتيبه (`LSM_ORDER_MUTABLE`) |

---

### ملخص: الـ Core Abstraction

الـ **LSM Framework** قدّم abstraction مفادها:

> "الـ kernel بيعرف فين نقاط القرار الأمني، بس مش بيعرف (ومش محتاج يعرف) إيه القرار."

الـ Smack بيملي الفراغ ده بـ:
- **Label per object**: كل inode، task، socket عنده label نصي.
- **Rule per subject**: كل label عنده قائمة بالـ labels التانية اللي مسموح يوصلها وبأي صلاحية.
- **Global label list**: مرجع واحد مشترك في النظام — ما يتعملش نسخ، بس pointers.

الجمال في `struct lsm_prop_smack` إنها pointer واحد بس — خفيفة، مباشرة، ومعاها كل المعلومات اللي الـ consumer محتاجها.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### مقدمة سريعة

الـ `include/linux/lsm/smack.h` هو header صغير جداً — بيعرّف بس `struct lsm_prop_smack` اللي بتربط الـ LSM framework العام بـ Smack. الـ implementation الحقيقية موجودة في `security/smack/smack.h`. الاتنين بيشتغلوا مع بعض، فالشرح هيكون شامل للملفين.

---

### 0. الـ Flags والـ Enums والـ Config Options

#### Config Options الأساسية

| Config | الأثر |
|--------|-------|
| `CONFIG_SECURITY_SMACK` | يفعّل Smack كلياً — بدونه `lsm_prop_smack.skp` مش موجود |
| `CONFIG_SECURITY_SMACK_NETFILTER` | يفعّل SECMARK labeling عبر netfilter بدل port labeling |
| `CONFIG_SECURITY_SMACK_BRINGUP` | يفعّل وضع bringup للـ debug — يسمح بـ `smack_unconfined` |
| `CONFIG_SECURITY_SMACK_APPEND_SIGNALS` | يخلي إرسال signal يتطلب `MAY_APPEND` بدل `MAY_WRITE` |
| `CONFIG_IPV6` | يفعّل دعم IPv6 host labeling |
| `CONFIG_AUDIT` | يفعّل audit logging في `smk_audit_info` |
| `CONFIG_KEYS` | يفعّل label للـ keyring objects |

#### Macros الـ Labeling

| Macro | القيمة | الغرض |
|-------|--------|-------|
| `SMK_LABELLEN` | 24 | الحد الأقصى التاريخي لطول الـ label (23 + null) |
| `SMK_LONGLABEL` | 256 | الحد الأقصى الحديث للـ label الطويل |
| `SMK_CIPSOLEN` | 24 | أقصى bytes للـ CIPSO level data |
| `SMACK_HASH_SLOTS` | 16 | عدد buckets في hash table الـ labels |
| `SMK_NUM_ACCESS_TYPE` | 7 | عدد أنواع الوصول: `r w x a t l b` |

#### Superblock Flags

| Flag | القيمة | المعنى |
|------|--------|--------|
| `SMK_SB_INITIALIZED` | `0x01` | الـ superblock اتهيأ بـ Smack labels |
| `SMK_SB_UNTRUSTED` | `0x02` | الـ filesystem مش موثوق — يطبّق `smk_default` |

#### Socket / NetLabel States

| ثابت | القيمة | الحالة |
|------|--------|--------|
| `SMK_NETLBL_UNSET` | 0 | لسه ما اتعملش labeling |
| `SMK_NETLBL_UNLABELED` | 1 | socket بدون label على الشبكة |
| `SMK_NETLBL_LABELED` | 2 | socket عليها CIPSO/netlabel label |
| `SMK_NETLBL_REQSKB` | 3 | محتاج skb عشان يكتمل الـ labeling |

#### Inode Flags

| Flag | القيمة | المعنى |
|------|--------|--------|
| `SMK_INODE_INSTANT` | `0x01` | الـ inode اتوضع label عليه فعلاً |
| `SMK_INODE_TRANSMUTE` | `0x02` | الـ directory في وضع transmute (الـ children يورثوا label الـ directory) |
| `SMK_INODE_CHANGED` | `0x04` | اتعمله transmute (غير مستخدم حالياً) |
| `SMK_INODE_IMPURE` | `0x08` | شارك في transaction بين labels مختلفة |

#### Access Permission Bits

| Bit | الاسم | المعنى |
|-----|-------|--------|
| `MAY_READ` | r | قراءة |
| `MAY_WRITE` | w | كتابة |
| `MAY_EXEC` | x | تنفيذ |
| `MAY_APPEND` | a | إضافة فقط |
| `MAY_TRANSMUTE` | `0x1000` | t — يتحكم في directory labeling |
| `MAY_LOCK` | `0x2000` | l — قفل الملف |
| `MAY_BRINGUP` | `0x4000` | b — log عند استخدام هذه القاعدة |
| `MAY_DELIVER` | w أو a | إرسال signal (حسب config) |

#### Mount Options Enum

```c
enum {
    Opt_error      = -1,  /* خطأ في parsing */
    Opt_fsdefault  =  0,  /* smackfsdef=LABEL */
    Opt_fsfloor    =  1,  /* smackfsfloor=LABEL */
    Opt_fshat      =  2,  /* smackfshat=LABEL */
    Opt_fsroot     =  3,  /* smackfsroot=LABEL */
    Opt_fstransmute=  4,  /* smackfstransmute=LABEL */
};
```

#### CIPSO Constants

| ثابت | القيمة | الغرض |
|------|--------|-------|
| `SMACK_CIPSO_DOI_DEFAULT` | 3 | الـ DOI الافتراضي (تاريخي) |
| `SMACK_CIPSO_DOI_INVALID` | -1 | قيمة غير صالحة |
| `SMACK_CIPSO_DIRECT_DEFAULT` | 250 | category رقم لـ direct mapping |
| `SMACK_CIPSO_MAPPED_DEFAULT` | 251 | category رقم لـ mapped labels |
| `SMACK_CIPSO_MAXLEVEL` | 255 | أقصى level في CIPSO 2.2 |
| `SMACK_CIPSO_MAXCATNUM` | 184 | أقصى عدد categories (23 × 8 bits) |

#### Ptrace Policy

| ثابت | القيمة | السلوك |
|------|--------|--------|
| `SMACK_PTRACE_DEFAULT` | 0 | تطبيق access rules العادية |
| `SMACK_PTRACE_EXACT` | 1 | يشترط تطابق الـ label بالضبط |
| `SMACK_PTRACE_DRACONIAN` | 2 | يشترط امتياز كامل لأي ptrace |

---

### 1. الـ Structs المهمة

#### `struct lsm_prop_smack` — نقطة التقاطع بين الـ LSM وSmack

```c
/* in include/linux/lsm/smack.h */
struct lsm_prop_smack {
#ifdef CONFIG_SECURITY_SMACK
    struct smack_known *skp;   /* pointer لـ label entry في قائمة Smack العالمية */
#endif
};
```

- **الغرض**: واجهة Smack للـ LSM framework العام — بتُضمَّن في `struct lsm_prop` اللي بتُخزَّن في السياق المشترك بين الـ LSMs.
- **المهم**: لو `CONFIG_SECURITY_SMACK` مش موجود، البنية فاضية تماماً — zero-size struct.
- **الاتصال**: بتشير إلى `struct smack_known` الموجودة في القائمة العالمية `smack_known_list`.

---

#### `struct smack_known` — قلب النظام

```c
struct smack_known {
    struct list_head        list;           /* ربطها بـ smack_known_list العالمية */
    struct hlist_node       smk_hashed;     /* ربطها بـ smack_known_hash[] للبحث السريع */
    char                   *smk_known;      /* نص الـ label (e.g., "User", "_", "*") */
    u32                     smk_secid;      /* رقم تعريف للـ components اللي ما بتدعمش LSM */
    struct netlbl_lsm_secattr smk_netlabel; /* بيانات CIPSO/netlabel للإرسال على الشبكة */
    struct list_head        smk_rules;      /* قائمة access rules لهذا الـ subject */
    struct mutex            smk_rules_lock; /* يحمي smk_rules من التعديل المتزامن */
};
```

- **الغرض**: يمثّل label واحد في نظام Smack — الـ labels مش strings عادية، دي objects ثابتة في الذاكرة بتُعاد استخدامها.
- **مهم جداً**: الـ entries بتُضاف بس، ما بتُحذفش أبداً — الـ pointers تظل صالحة للأبد.
- البنية موجودة في قائمتين في نفس الوقت: `smack_known_list` (ordered) و `smack_known_hash[]` (للبحث بـ O(1)).

---

#### `struct task_smack` — الـ Label الخاص بالـ Task

```c
struct task_smack {
    struct smack_known *smk_task;       /* label الوصول الحالي للـ task */
    struct smack_known *smk_forked;     /* label وقت الـ fork (للرجوع إليه) */
    struct smack_known *smk_transmuted; /* label بعد transmutation */
    struct list_head    smk_rules;      /* قواعد وصول خاصة بهذا الـ task */
    struct mutex        smk_rules_lock; /* يحمي smk_rules */
    struct list_head    smk_relabel;    /* labels مسموح للـ task أنه ينتقل إليها */
};
```

- **الغرض**: بيانات Smack المرتبطة بكل task — بتُخزَّن في blob في `struct cred`.
- **smk_relabel**: قائمة `smack_known_list_elem` — الـ task ممنوع ينتقل لأي label تاني غير اللي في القائمة دي.
- **الوصول**: `smack_cred(cred)` — بتحسب offset في `cred->security` بناءً على `smack_blob_sizes.lbs_cred`.

---

#### `struct inode_smack` — الـ Label الخاص بالـ Inode

```c
struct inode_smack {
    struct smack_known *smk_inode; /* label الـ filesystem object نفسه */
    struct smack_known *smk_task;  /* label الـ task اللي أنشأ الـ inode */
    struct smack_known *smk_mmap;  /* label نطاق الـ mmap */
    int                 smk_flags; /* SMK_INODE_* flags */
};
```

- **الغرض**: بيانات Smack لكل inode في النظام.
- **smk_mmap**: لو task حاول يعمل mmap لملف، الـ label ده بيُستخدم للتحقق من الصلاحية.
- **الوصول**: `smack_inode(inode)` → `inode->i_security + smack_blob_sizes.lbs_inode`.

---

#### `struct socket_smack` — الـ Label الخاص بالـ Socket

```c
struct socket_smack {
    struct smack_known *smk_out;    /* label الـ traffic الصادر */
    struct smack_known *smk_in;     /* label الـ traffic الوارد */
    struct smack_known *smk_packet; /* label peer الـ TCP (بيُملأ عند الاتصال) */
    int                 smk_state;  /* SMK_NETLBL_* — حالة الـ netlabel */
};
```

- **الغرض**: تحكم في حركة الشبكة — الـ label الصادر بيُرسَل في الـ packet، والوارد بيُتحقق منه.
- **smk_packet**: بيُملأ من CIPSO header للـ TCP peer، أو من قوائم `smk_net4addr_list`.

---

#### `struct superblock_smack` — الـ Label الخاص بالـ Filesystem

```c
struct superblock_smack {
    struct smack_known *smk_root;    /* label جذر الـ filesystem */
    struct smack_known *smk_floor;   /* الحد الأدنى للوصول */
    struct smack_known *smk_hat;     /* label خاص للـ privileged access */
    struct smack_known *smk_default; /* label افتراضي لملفات جديدة */
    int                 smk_flags;   /* SMK_SB_* flags */
};
```

- **الغرض**: يتحكم في labels خاصة بالـ filesystem كلها — بيُحدَّد عند mount عبر options زي `smackfsdef=`.

---

#### `struct smack_rule` — قاعدة الوصول

```c
struct smack_rule {
    struct list_head    list;        /* ربطها بقائمة rules */
    struct smack_known *smk_subject; /* من؟ — الـ label اللي بيطلب الوصول */
    struct smack_known *smk_object;  /* لمن؟ — الـ label المطلوب الوصول إليه */
    int                 smk_access;  /* بتاه إيه؟ — bitmask من MAY_* */
};
```

- **الغرض**: تمثّل قاعدة واحدة في سياسة Smack: "subject يقدر يعمل access على object".
- **التخزين**: القاعدة بتُخزَّن في `smack_known.smk_rules` للـ subject — وبيستخدم `smack_rule_cache` (kmem_cache) لتسريع الـ allocation.

---

#### `struct smk_net4addr` / `struct smk_net6addr` — Host Labeling

```c
struct smk_net4addr {
    struct list_head  list;
    struct in_addr    smk_host;   /* عنوان الشبكة */
    struct in_addr    smk_mask;   /* قناع الشبكة */
    int               smk_masks; /* حجم الـ mask بالـ bits */
    struct smack_known *smk_label; /* label المستخدم لهذا النطاق */
};
```

- **الغرض**: تعيين Smack label لـ IP ranges — بيُستخدم لتحديد label الـ traffic الوارد من IP معين.
- القائمة العالمية: `smk_net4addr_list` و `smk_net6addr_list`.

---

#### `struct smk_port_label` — Port Labeling (IPv6 فقط)

```c
struct smk_port_label {
    struct list_head    list;
    struct sock        *smk_sock;       /* الـ socket المرتبطة */
    unsigned short      smk_port;       /* رقم الـ port */
    struct smack_known *smk_in;         /* label الوارد */
    struct smack_known *smk_out;        /* label الصادر */
    short               smk_sock_type;  /* نوع الـ socket */
    short               smk_can_reuse;  /* هل ممكن إعادة الاستخدام؟ */
};
```

- **الغرض**: ربط Smack label بـ port number — يُفعَّل فقط لو `SMACK_IPV6_PORT_LABELING` موجود.

---

#### `struct smack_known_list_elem` — عنصر في قائمة Labels

```c
struct smack_known_list_elem {
    struct list_head    list;
    struct smack_known *smk_label;
};
```

- **الغرض**: wrapper خفيف لاستخدام `smack_known` في قوائم متعددة — بيُستخدم في `task_smack.smk_relabel` و `smack_onlycap_list`.

---

#### `struct smack_audit_data` + `struct smk_audit_info`

```c
struct smack_audit_data {
    const char *function; /* اسم الـ hook اللي اتنادى */
    char       *subject;  /* نص subject label */
    char       *object;   /* نص object label */
    char       *request;  /* نص الـ permissions المطلوبة */
    int         result;   /* النتيجة: 0 مسموح، غير ذلك مرفوض */
};

struct smk_audit_info {
#ifdef CONFIG_AUDIT
    struct common_audit_data a;   /* البيانات العامة للـ LSM audit */
    struct smack_audit_data sad;  /* البيانات الخاصة بـ Smack */
#endif
};
```

- **الغرض**: جمع بيانات الـ audit في مكان واحد — لو `CONFIG_AUDIT` مش موجود، البنية فاضية وكل الـ inline functions لا تعمل شيء.

---

### 2. مخطط علاقات الـ Structs (ASCII)

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    Global Label Registry                     │
  │                                                             │
  │  smack_known_list (doubly linked)                           │
  │  smack_known_hash[16] (hlist for fast lookup)               │
  └─────────────────────────────────────────────────────────────┘
              │ contains many
              ▼
  ┌───────────────────────────────────┐
  │         struct smack_known        │
  │  ┌─────────────────────────────┐  │
  │  │ smk_known: "User"          │  │
  │  │ smk_secid: 42              │  │
  │  │ smk_netlabel: CIPSO attrs  │  │
  │  │ smk_rules ──────────────────┼──┼──► list of smack_rule
  │  │ smk_rules_lock (mutex)     │  │
  │  └─────────────────────────────┘  │
  └───────────────────────────────────┘
        ▲              ▲              ▲              ▲
        │              │              │              │
  ┌─────┴──────┐ ┌─────┴──────┐ ┌────┴───────┐ ┌───┴────────────┐
  │task_smack  │ │inode_smack │ │socket_smack│ │superblock_smack│
  │ smk_task ──┘ │ smk_inode─┘ │ smk_out ───┘ │ smk_root ──────┘
  │ smk_forked─┘ │ smk_task ─┘ │ smk_in ────┘ │ smk_floor ─────┘
  │ smk_transmut─┘ smk_mmap ─┘ │ smk_packet─┘ │ smk_hat ───────┘
  │ smk_rules    │             │ smk_state    │ smk_default ───┘
  │ smk_relabel──┼─► list of   └─────────────┘ └────────────────┘
  └──────────────┘   smack_known_list_elem

  struct smack_rule:
  ┌─────────────────────────────┐
  │ smk_subject ──► smack_known │
  │ smk_object  ──► smack_known │
  │ smk_access: MAY_READ | ...  │
  └─────────────────────────────┘

  struct lsm_prop_smack (in lsm/smack.h):
  ┌──────────────────────────────┐
  │ skp ──────────────────────────┼──► smack_known
  └──────────────────────────────┘
  (embedded in struct lsm_prop used by LSM framework)

  struct smk_net4addr / smk_net6addr:
  ┌─────────────────────────┐
  │ smk_host: 192.168.1.0   │
  │ smk_mask: /24           │
  │ smk_label ──────────────┼──► smack_known
  └─────────────────────────┘

  struct smk_port_label (IPv6 only):
  ┌─────────────────────────┐
  │ smk_port: 8080          │
  │ smk_sock ──► struct sock│
  │ smk_in ─────────────────┼──► smack_known
  │ smk_out ────────────────┼──► smack_known
  └─────────────────────────┘
```

---

### 3. دورة الحياة — Label

```
  Boot / Module Init
        │
        ▼
  smack_initcall()
        │
        ├──► تسجيل built-in labels:
        │    smack_known_floor ("_")
        │    smack_known_hat ("^")
        │    smack_known_huh ("?")
        │    smack_known_star ("*")
        │    smack_known_web ("@")
        │    (كلهم بيدخلوا smack_known_list و smack_known_hash)
        │
        ├──► init_smk_fs() — تهيئة /sys/fs/smackfs
        │
        └──► smack_nf_ip_init() — (لو NETFILTER موجود)

  Runtime — Label Import (مثلاً: xattr على ملف)
        │
        ▼
  smk_import_entry(label_string, len)
        │
        ├──► smk_find_entry() — ابحث في smack_known_hash
        │         │
        │    موجود؟ ──► return existing smack_known (share it)
        │         │
        │    مش موجود؟
        │         ▼
        │    allocate smack_known + smk_known string
        │    smk_insert_entry() ──► list_add_tail() + hlist_add_head()
        │    smack_populate_secattr() ──► smk_netlbl_mls()
        │
        └──► return smack_known* (لا يُحذف أبداً)

  Teardown
        │
        └──► Labels لا تُحذف أبداً في runtime
             فقط عند unload الـ module (غير مدعوم عملياً)
```

---

### 4. دورة الحياة — Task

```
  fork() / exec()
        │
        ▼
  smack_cred_prepare() أو smack_cred_alloc_blank()
        │
        ├──► allocate task_smack في blob الـ cred الجديدة
        │    smk_task   = parent->smk_task   (copy on fork)
        │    smk_forked = parent->smk_task   (save original)
        │
        ▼
  Task Running
        │
        ├──► write("/proc/.../attr/current", "NewLabel")
        │         │
        │         ▼
        │    smack_setprocattr()
        │         ├── تحقق smk_relabel (هل النقل مسموح؟)
        │         └── smk_task = smk_import_entry("NewLabel")
        │
        └──► access() / open() / connect()
                  │
                  ▼
             smk_curacc() أو smk_access()
                  ├── subject = smk_of_current()
                  ├── object  = smk_of_inode() أو smk_of_task()
                  └── smk_access_entry() → check smk_rules list

  exit()
        │
        ▼
  smack_cred_free()
        └──► smk_destroy_label_list(&tsp->smk_rules)
             kfree task_smack blob
```

---

### 5. دورة الحياة — Inode

```
  inode_alloc_security()
        │
        ▼
  smack_inode() → inode_smack في blob
        │
        ├── smk_inode = smack_net_ambient (initial placeholder)
        │   (SMK_INODE_INSTANT flag مش موجود بعد)
        │
        ▼
  inode_init_security() / getxattr("security.SMACK64")
        │
        ├── smk_import_entry(xattr_value)
        └── smk_inode = result
            SMK_INODE_INSTANT flag يتعمل set
            (لو directory و transmute xattr موجود: SMK_INODE_TRANSMUTE)

  File Access Check
        │
        ▼
  smack_inode_permission()
        ├── subject = smk_of_current()
        ├── object  = smk_of_inode(inode)
        └── smk_curacc(object, MAY_READ | ..., &audit_info)

  inode_free_security()
        └──► blob يتحرر مع الـ inode (managed by LSM framework)
```

---

### 6. مسار Access Check — Call Flow

```
  userspace: open("/secret", O_RDONLY)
        │
        ▼
  VFS: may_open()
        │
        ▼
  security_inode_permission()           [security/security.c]
        │
        ▼
  smack_inode_permission()              [security/smack/smack_lsm.c]
        │
        ├── smk_of_current()
        │       └── smk_of_task(smack_cred(current_cred()))
        │               └── tsp->smk_task  (struct smack_known*)
        │
        ├── smk_of_inode(inode)
        │       └── inode_smack->smk_inode (struct smack_known*)
        │
        ├── smk_curacc(object, MAY_READ, &audit_info)
        │       │
        │       ▼
        │   smk_access(subject, object, MAY_READ, audit)
        │       │
        │       ├── smk_access_entry(subject->smk_known,
        │       │                    object->smk_known,
        │       │                    &subject->smk_rules)
        │       │       └── list_for_each_entry(rule, &smk_rules, list)
        │       │               if rule->smk_object == object
        │       │                   return rule->smk_access
        │       │
        │       ├── لو access مسموح: return 0
        │       │
        │       └── لو مرفوض:
        │               smack_log(subject, object, request, EPERM, audit)
        │               return -EACCES
        │
        └──► EACCES أو 0 يرجع للـ VFS
```

---

### 7. مسار Network Labeling — Call Flow

```
  TCP connect() من task بـ label "Client"
        │
        ▼
  smack_socket_connect()
        │
        ├── socket_smack->smk_out = smk_of_current()  ["Client"]
        │
        └── smack_netlabel(sk, ...)
                │
                ▼
            netlbl_sock_setattr()   [netlabel subsystem]
                │
                ├── CIPSO option يتعمل encode من smk_netlabel
                └── يتحط في الـ IP packet header

  TCP packet وصل للـ server
        │
        ▼
  smack_socket_sock_rcv_skb()
        │
        ├── netlbl_skbuff_getattr() → label من CIPSO header
        │       └── OR smk_net4addr_list lookup بالـ source IP
        │
        ├── socket_smack->smk_packet = label الـ peer
        │
        └── smk_access(peer_label, socket->smk_in, MAY_WRITE, audit)
                └── مرفوض؟ drop الـ packet
```

---

### 8. استراتيجية الـ Locking

#### جدول الـ Locks

| Lock | النوع | يحمي إيه؟ | ترتيب الأخذ |
|------|-------|-----------|------------|
| `smack_known_lock` | `mutex` | `smack_known_list`، `smack_known_hash[]` — إضافة labels جديدة | أول |
| `smack_known.smk_rules_lock` | `mutex` | `smack_known.smk_rules` — قواعد subject label | تاني |
| `task_smack.smk_rules_lock` | `mutex` | `task_smack.smk_rules` — قواعد الـ task الخاصة | تاني (مستقل) |
| `smack_onlycap_lock` | `mutex` | `smack_onlycap_list` — قائمة الـ privileged labels | مستقل |
| RCU | read-side lock | `cred` للـ `smk_of_task_struct_obj()` | أخف من mutex |

#### تفاصيل مهمة

**الـ `smack_known_list` — قراءة بدون lock:**
- الـ labels بتُضاف فقط، ما بتُحذفش.
- القراءة آمنة بدون lock لأن الـ list monotonically growing.
- الإضافة بس بتحتاج `smack_known_lock`.

**الـ `smk_rules` — محمية بـ per-label mutex:**
- كل `smack_known` عنده `smk_rules_lock` منفصل.
- هذا يسمح بـ concurrent access checks لـ subjects مختلفة.

**الـ `cred` — محمية بـ RCU:**
```c
/* مثال من smk_of_task_struct_obj() */
rcu_read_lock();
cred = __task_cred(t);          /* atomic read under RCU */
skp = smk_of_task(smack_cred(cred));
rcu_read_unlock();
```

**Lock Ordering (لتجنب deadlock):**
```
smack_known_lock
    └──► smack_known.smk_rules_lock  (لـ subject label معين)
         (لا تأخذ smack_known_lock وانت شايل smk_rules_lock)
```

**الـ `smack_rule_cache` (kmem_cache):**
- بيُستخدم لتسريع allocation وdeallocation لـ `struct smack_rule`.
- الـ cache نفسه بيُنشأ مرة واحدة عند init — لا يحتاج lock في الاستخدام.

---

### ملاحظة ختامية على `lsm_prop_smack`

الـ `include/linux/lsm/smack.h` بيعرّف interface نظيف جداً:

```c
struct lsm_prop_smack {
#ifdef CONFIG_SECURITY_SMACK
    struct smack_known *skp;
#endif
};
```

الـ pointer `skp` ده بيكون دايماً pointer لـ entry في `smack_known_list` — مش نسخة مستقلة.
لو `CONFIG_SECURITY_SMACK` مش موجود، الـ struct فاضية والـ compiler بيحذف كل الكود المرتبط بيها تلقائياً.
الـ lsm framework بيستخدم `struct lsm_prop` اللي بتضم `lsm_prop_smack` جنب بيانات الـ LSMs التانية في نفس البنية.
## Phase 4: شرح الـ Functions

### ملاحظة مهمة

الـ file `include/linux/lsm/smack.h` مش بيعرّف أي functions — هو بيعرّف **struct واحد بس** اسمه `lsm_prop_smack`، وده جزء من الـ **LSM (Linux Security Module) interface layer** اللي بيخلي الـ Smack subsystem يعرض الـ security label بتاعه للـ subsystems التانية في الـ kernel.

الـ file ده بيمثل **glue header** — مش library functions، مش API calls — ده type definition بيُستخدم جوه الـ `lsm_prop` union اللي بتجمع كل الـ LSM-specific security properties في مكان واحد.

---

### جدول ملخص الـ API

| العنصر | النوع | الوصف |
|--------|-------|-------|
| `struct smack_known` | forward declaration | الـ struct الحقيقي للـ Smack label، معرّف في `security/smack/smack.h` |
| `struct lsm_prop_smack` | struct | الـ Smack-specific slot جوه الـ `lsm_prop` union |
| `lsm_prop_smack::skp` | `struct smack_known *` | pointer للـ Smack label entry في الـ global label list |

---

### الـ Category الوحيدة: Type Definition / LSM Property Slot

#### الغرض من الـ Group

الـ LSM framework في الـ kernel بيستخدم نظام **`lsm_prop`** عشان يخزّن الـ security metadata الخاصة بكل LSM بشكل منفصل ومنظّم. كل LSM (SELinux، Smack، AppArmor، إلخ) بيوفر **struct صغير** بيحط فيه الـ data اللي محتاج يشاركها مع الـ subsystems التانية. الـ Smack بيحتاج يشارك pointer واحد بس — هو الـ `skp` — اللي بيأشر على الـ label entry في الـ **global Smack label list**.

الـ design ده بيحقق:
- **Zero-cost abstraction**: لو `CONFIG_SECURITY_SMACK` مش مفعّل، الـ struct بيبقى فاضي تمامًا.
- **Type safety**: الـ subsystems التانية بتشتغل مع `struct lsm_prop_smack` مش مع `void *`.
- **Encapsulation**: الـ Smack internals (الـ `smack_known` details) محجوبة وراء الـ forward declaration.

---

### `struct lsm_prop_smack`

```c
/* include/linux/lsm/smack.h */
struct lsm_prop_smack {
#ifdef CONFIG_SECURITY_SMACK
    struct smack_known *skp;  /* pointer to the subject's Smack label entry */
#endif
};
```

#### ما هو

ده الـ **Smack-specific slot** اللي بيتحط جوه `lsm_prop` union. الـ `lsm_prop` union بتجمع الـ security properties بتاعة كل الـ LSMs اللي ممكن يكونوا active في نفس الوقت على الـ kernel، وكل LSM بياخد slot باسمه.

#### الـ Field الوحيد: `skp`

```c
struct smack_known *skp;
```

- **النوع**: pointer لـ `struct smack_known` — الـ struct ده معرّف في `security/smack/smack.h` (مش في الـ public headers).
- **المعنى**: بيأشر على **entry في الـ global Smack label list** (`smack_known_list`) اللي بتمثل الـ security label بتاعة الـ subject (process، socket، إلخ).
- **الـ lifetime**: الـ `smack_known` entries بتتحط في الـ list ومبتتمسحش طول عمر الـ kernel — يعني الـ pointer ده دايمًا valid طالما الـ kernel شغّال.
- **الـ immutability**: الـ pointer نفسه ممكن يتغيّر (لو الـ label اتغيّر)، بس الـ `smack_known` object اللي بيأشر عليه مش بيتعدّل بعد الـ creation.

#### الـ `#ifdef CONFIG_SECURITY_SMACK`

لو الـ Smack **مش** مفعّل في الـ kernel config، الـ struct بيبقى **completely empty** (`{}`). ده بيخلي الـ compiler يعمل zero-size optimization، ومحدش محتاج يكتب `#ifdef` في كل مكان بيستخدم الـ `lsm_prop_smack`.

---

### السياق الأشمل: إزاي `lsm_prop_smack` بيتستخدم

#### الـ `lsm_prop` Union

الـ struct ده بيتضمّن جوه `struct lsm_prop` المعرّف في `include/linux/lsm_hook_defs.h` أو `include/linux/security.h`:

```c
struct lsm_prop {
    struct lsm_prop_smack smack;
    /* slots for other LSMs: selinux, apparmor, etc. */
};
```

#### الـ `struct smack_known` (الـ target)

```c
/* security/smack/smack.h — simplified */
struct smack_known {
    struct list_head    list;       /* entry in smack_known_list */
    struct hlist_node   smk_hashed; /* hash table node */
    char                *smk_known; /* the label string (e.g., "System") */
    u32                 smk_secid;  /* unique numeric ID for this label */
    struct netlbl_lsm_secattr smk_netlabel; /* NetLabel attributes */
    struct smack_rule   *smk_rules; /* access rules involving this label */
    /* ... */
};
```

الـ `skp` pointer بيديك access لكل الـ metadata دي بدون ما تعمل string lookup في كل مرة.

#### مثال: إزاي Smack بيستخدم الـ `lsm_prop`

```c
/* في الـ smack_task_getsecid() hook مثلًا */
void smack_task_getsecid_subj(struct task_struct *p, struct lsm_prop *prop)
{
    struct smack_known *skp = smk_of_task_struct_subj(p);
    /* بنحط الـ pointer في الـ Smack slot */
    prop->smack.skp = skp;
}
```

ثم subsystem تاني (زي الـ audit) بيقرأ:

```c
/* في الـ audit subsystem */
void audit_log_lsm(struct lsm_prop *prop)
{
    /* الـ Smack label string جاهز من غير string search */
    if (prop->smack.skp)
        audit_log_format(ab, " smack=%s", prop->smack.skp->smk_known);
}
```

---

### تصميم الـ Header: Forward Declaration بدل Full Include

```c
struct smack_known;  /* forward declaration فقط */
```

الـ header ده **مش بيعمل include** لـ `security/smack/smack.h` — وده مقصود:

- الـ `security/smack/smack.h` فيه الـ Smack internals الكاملة وده ملوش مكان في الـ public kernel API.
- الـ subsystems التانية محتاجة **pointer فقط** — مش محتاجة يعرفوا شكل الـ `smack_known` الداخلي.
- ده بيقلل الـ **compilation dependencies** ويمنع تسريب الـ internals.

---

### خلاصة

| الجانب | التفصيل |
|--------|---------|
| الـ file type | Type-only header — لا functions، لا macros معقدة |
| الـ struct | `lsm_prop_smack` — Smack's slot في الـ `lsm_prop` union |
| الـ field | `skp` — pointer لـ `smack_known` entry في الـ global label list |
| الـ conditionality | الـ field موجود فقط لو `CONFIG_SECURITY_SMACK=y` |
| الـ design pattern | Forward declaration + opaque pointer = encapsulation بدون coupling |
| الـ callers | Smack LSM hooks، audit subsystem، networking (NetLabel)، IPC security |
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem ده هو **Smack (Simplified Mandatory Access Control Kernel)** — نظام الـ MAC اللي بيستخدم labels بسيطة للتحكم في الـ access بين الـ subjects والـ objects في الـ kernel. الـ interface الأساسي بين Smack والـ subsystems التانية هو `struct lsm_prop_smack` اللي بيحمل pointer على `struct smack_known` — الـ label entry الحقيقية.

---

### Software Level

#### 1. debugfs entries

الـ Smack بيشتغل عبر **smackfs** — filesystem خاص بيتـmount على `/sys/fs/smackfs/` (مش `/sys/kernel/debug/`). ده هو الـ control interface الكامل.

```bash
# mount smackfs لو مش موجود
mount -t smackfs smackfs /sys/fs/smackfs

# قراءة كل الـ labels الموجودة في الـ kernel
cat /sys/fs/smackfs/load2

# قراءة الـ rules الحالية (subject object access)
cat /sys/fs/smackfs/load

# label بتاع الـ process الحالي
cat /proc/self/attr/current

# label بتاع process معين (PID)
cat /proc/1234/attr/current

# الـ ambient network label
cat /sys/fs/smackfs/ambient

# قراءة الـ CIPSO configuration
cat /sys/fs/smackfs/cipso2
cat /sys/fs/smackfs/mapped

# onlycap — مين عنده privilege خاص
cat /sys/fs/smackfs/onlycap

# ptrace rule الحالي
cat /sys/fs/smackfs/ptrace

# قراءة الـ netlabel mappings (IPv4)
cat /sys/fs/smackfs/netlabel

# unconfined label (لو CONFIG_SECURITY_SMACK_BRINGUP مفعّل)
cat /sys/fs/smackfs/unconfined
```

**تفسير output الـ `load2`:**
كل سطر فيه: `subject object rwxatl` — الحروف دي بتمثل:
- `r` = read, `w` = write, `x` = execute
- `a` = append, `t` = transmute, `l` = lock, `b` = bringup

---

#### 2. sysfs entries

الـ Smack مش بيستخدم sysfs بشكل مباشر، بيستخدم **smackfs**. لكن في بعض entries مهمة:

```bash
# الـ LSM المفعّل حالياً
cat /sys/kernel/security/lsm

# blob sizes — بتحدد مساحة الـ security data في كل object
# (مش ملف مباشر، لكن بتشوفه في kernel log وقت boot)
dmesg | grep -i smack

# xattr على الـ files
getfattr -n security.SMACK64 /path/to/file
getfattr -n security.SMACK64EXEC /path/to/file
getfattr -n security.SMACK64MMAP /path/to/file
getfattr -n security.SMACK64TRANSMUTE /path/to/file
getfattr -n security.SMACK64IPIN /path/to/file   # للـ sockets
getfattr -n security.SMACK64IPOUT /path/to/file  # للـ sockets
```

---

#### 3. ftrace — tracepoints وـ events

```bash
# شوف الـ events المتاحة للـ LSM
ls /sys/kernel/tracing/events/lsm/

# enable كل الـ LSM events
echo 1 > /sys/kernel/tracing/events/lsm/enable

# trace الـ smack access checks
# (Smack بيستخدم audit events مش tracepoints dedicated)
# بدل كده، استخدم function tracing على الـ smack functions

# trace دالة smk_access
echo 'smk_access' >> /sys/kernel/tracing/set_ftrace_filter
echo 'smk_curacc' >> /sys/kernel/tracing/set_ftrace_filter
echo 'smk_tskacc' >> /sys/kernel/tracing/set_ftrace_filter
echo 'smack_file_open' >> /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace

# trace كل دوال smack مع return values
echo 'smk_*' >> /sys/kernel/tracing/set_ftrace_filter
echo 'smack_*' >> /sys/kernel/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/tracing/current_tracer
cat /sys/kernel/tracing/trace
```

---

#### 4. printk / dynamic debug

```bash
# تفعيل الـ dynamic debug لكل الـ smack module
echo 'module smack +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug في ملف معين
echo 'file security/smack/smack_access.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file security/smack/smack_lsm.c +p' > /sys/kernel/debug/dynamic_debug/control

# إظهار الـ smack audit logs في الـ kernel log
# Smack بيستخدم audit framework — شغّل الـ audit daemon
auditd -f  # foreground mode

# استعراض الـ denied access events
ausearch -m AVC | grep smack
ausearch -ts recent | grep -i smack

# تفعيل verbose logging في smackfs
echo 3 > /sys/fs/smackfs/logging   # 1=denied, 2=accept, 3=both
cat /sys/fs/smackfs/logging         # قراءة الـ current policy
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوصف |
|--------|-------|
| `CONFIG_SECURITY_SMACK` | تفعيل Smack نفسه — أساسي |
| `CONFIG_SECURITY_SMACK_BRINGUP` | وضع الـ bringup: بيسمح بـ `MAY_BRINGUP` — مفيد لـ debug قبل ما تطبق policy |
| `CONFIG_SECURITY_SMACK_NETFILTER` | Smack مع netfilter/secmark — للـ network labeling |
| `CONFIG_SECURITY_SMACK_APPEND_SIGNALS` | تغيير semantics الـ signal delivery |
| `CONFIG_AUDIT` | لازم يكون مفعّل عشان تشوف الـ smack audit events |
| `CONFIG_SECURITY_PATH` | hook على الـ path-based operations |
| `CONFIG_DEBUG_KERNEL` | تفعيل الـ debugging العام |
| `CONFIG_DEBUG_LIST` | كشف فساد الـ linked lists — مهم لأن Smack بيستخدم lists كتير |
| `CONFIG_LOCKDEP` | كشف deadlocks في `smk_rules_lock` و`smack_known_lock` |
| `CONFIG_KASAN` | كشف memory corruption في الـ label strings |
| `CONFIG_PROVE_LOCKING` | تحقق من صحة استخدام الـ mutexes في Smack |
| `CONFIG_NETLABEL` | الـ NetLabel subsystem — مطلوب للـ network labeling |

---

#### 6. أدوات خاصة بالـ Smack

```bash
# smackctl — الأداة الرسمية للـ Smack policy management
# (من package smackutil/smack-utils)

# قراءة الـ label الحالي للـ process
smackctl get-label $$

# تعيين label لـ executable
smackctl set-label /usr/bin/myapp "mylabel"

# عرض كل الـ rules
smackctl list-rules

# اختبار access
smackctl check-access "subject_label" "object_label" "r"

# netlabel tool — للـ network debugging
netlabelctl cipso list
netlabelctl unlbl list

# audit2smack — تحويل الـ denials لـ rules (مشابه لـ audit2allow في SELinux)
ausearch -m AVC | grep smack | audit2smack
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة في الـ log | المعنى | الحل |
|-----------------|--------|-------|
| `smack: DENIED subj=X obj=Y req=r` | label X محتاجه read access على label Y وملقاش rule | أضف rule: `echo "X Y r" > /sys/fs/smackfs/load2` |
| `smack: BRINGUP subj=X obj=Y req=w` | access بـ `MAY_BRINGUP` — تجربي مش نهائي | حوّله لـ rule دائم أو اشل الـ bringup |
| `smack: no ambient label` | الـ ambient network label مش متحدد | `echo "labelname" > /sys/fs/smackfs/ambient` |
| `smack: xattr length %d invalid` | الـ xattr قيمة غير صالحة على الـ filesystem | امسح الـ xattr وأعد تعيينه |
| `smack: missing label for inode` | inode معندوش label محدد | ظهور بعد mount بدون smack options — حدد `smackfsdef=label` |
| `smack: socket peer label mismatch` | الـ TCP peer label مش متطابق مع الـ expected | فحص الـ CIPSO configuration على الـ wire |
| `smack: CIPSO error %d` | خطأ في الـ CIPSO IP option processing | راجع `netlabelctl cipso list` وتأكد من الـ DOI |
| `smack: netlabel failure %d` | فشل في تطبيق الـ netlabel على الـ socket | تأكد من تحميل `netlabel` module |
| `smack: ptrace access rejected` | process حاول ptrace بدون صلاحية | غيّر `smack_ptrace_rule` أو أضف cap |
| `smack: no access to onlycap` | process عنده cap لكن مش في `onlycap` list | أضف الـ label لـ `/sys/fs/smackfs/onlycap` |

---

#### 8. Strategic Points لـ `dump_stack()` / `WARN_ON()`

النقاط دي في الكود مهمة للـ debugging:

```c
/* في smk_access() — نقطة القرار الأساسية */
int smk_access(struct smack_known *subject, struct smack_known *object,
               int request, struct smk_audit_info *a)
{
    /* WARN_ON هنا لو subject أو object NULL — مش المفروض يحصل */
    WARN_ON(subject == NULL || object == NULL);

    /* dump_stack() هنا بيساعد تعرف مين طلب الـ access */
}

/* في smk_import_entry() — عند إضافة label جديد */
struct smack_known *smk_import_entry(const char *string, int len)
{
    /* WARN_ON لو الـ label string NULL */
    WARN_ON(string == NULL);
}

/* في smack_cred() — عند access للـ credential blob */
static inline struct task_smack *smack_cred(const struct cred *cred)
{
    /* WARN_ON لو offset غلط — يشير لـ blob size mismatch */
    WARN_ON(smack_blob_sizes.lbs_cred == 0);
    return cred->security + smack_blob_sizes.lbs_cred;
}

/* في smk_access_entry() — لما بيبحث في الـ rules list */
/* dump_stack() هنا بيساعد تتبع مشكلة لو الـ rules list اتتلفت */
```

أهم الـ WARN_ON locations:
- عند دخول `smk_access()` بـ NULL pointers
- عند فشل `smk_import_entry()` في allocate
- عند lock contention غير متوقع في `smk_rules_lock`
- عند اكتشاف `smk_flags` قيمة غير متوقعة في `inode_smack`

---

### Hardware Level

#### 1. التحقق من تطابق الـ Hardware State مع الـ Kernel State

الـ Smack هو software-only LSM — مافيش hardware مباشر. لكن لما بيشتغل مع الـ network:

```bash
# تحقق إن الـ CIPSO labels بتتبعت صح على الـ wire
tcpdump -i eth0 -v 'ip[0] & 0xf > 5' | grep -i cipso

# تحقق من الـ NetLabel state
cat /proc/net/netlabel/addr4
cat /proc/net/netlabel/addr6
cat /proc/net/netlabel/protocols
cat /proc/net/netlabel/static

# تحقق من الـ secmark rules (لو NETFILTER مفعّل)
iptables -L -v -n | grep SECMARK
ip6tables -L -v -n | grep SECMARK
```

---

#### 2. Register Dump Techniques

الـ Smack مش بيتعامل مع hardware registers مباشرة. لكن لـ network hardware:

```bash
# فحص الـ NIC capabilities للـ security offloading
ethtool -k eth0 | grep -i offload

# لو بتستخدم /dev/mem لفحص الـ memory (نادر)
# لازم تعرف address الـ smack_known_list في kernel
# من kernel symbols
cat /proc/kallsyms | grep smack_known_list
# ثم
devmem2 <address> l

# أفضل بكتير: استخدم crash utility
echo "smack_known_list" | crash /proc/kcore /usr/lib/debug/boot/vmlinux-$(uname -r)
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

الـ Smack نفسه software — لكن لو بتـdebug مشكلة network labeling:

```
السيناريو: الـ CIPSO IP options مش بتتبعت صح على الـ wire

Logic Analyzer Setup:
- Capture على الـ NIC pins
- Filter على IP protocol
- ابحث في الـ IP options field (offset 20+)
- CIPSO option = 0x86 (134)
- بعده: الـ DOI ثم الـ sensitivity level

Oscilloscope:
- مش مفيد مباشرة مع Smack
- لكن لو في timing issue في الـ netlabel processing
- قيس latency بين packet arrive وـ accept/deny
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | الـ Pattern في الـ log |
|---------|----------------------|
| NIC مش بتحفظ الـ CIPSO options | `smack: netlabel failure -22` متكرر |
| Hardware offload بيشوه الـ IP options | `smack: CIPSO error -14` على الـ receive |
| MTU صغير بيقطع الـ CIPSO header | fragmentation errors مع smack network denials |
| Clock skew في distributed system | inconsistent label verification بين nodes |

```bash
# تشخيص NIC offload مشاكل مع CIPSO
ethtool -K eth0 rx off tx off  # disable offloading مؤقتاً
# لو المشكلة اتحلت، الـ NIC offload هو السبب

# مراقبة الـ ip options processing
perf trace -e net:net_dev_xmit,net:netif_receive_skb 2>&1 | head -50
```

---

#### 5. Device Tree Debugging

الـ Smack مش بيحتاج Device Tree entries. لكن لو في platform-specific security hardware:

```bash
# تحقق من الـ LSM في الـ kernel cmdline (DT أو bootloader)
cat /proc/cmdline | grep -i lsm
# المفروض يظهر: lsm=smack,capability أو ما يشبهه

# لو بتبدأ Smack من DT (نادر — embedded systems)
cat /proc/device-tree/chosen/linux,lsm 2>/dev/null

# تحقق من الـ security xattrs محفوظة صح على الـ filesystem
# (مهم لـ embedded systems مع read-only rootfs)
ls -la --lcontext /
```

---

### Practical Commands

#### مجموعة أوامر شاملة — Copy-Ready

```bash
#!/bin/bash
# === Smack Full Debug Script ===

echo "=== Current LSM ==="
cat /sys/kernel/security/lsm

echo "=== Current Process Label ==="
cat /proc/self/attr/current

echo "=== All Smack Labels ==="
cat /sys/fs/smackfs/load2 2>/dev/null || echo "smackfs not mounted"

echo "=== Smack Access Rules ==="
cat /sys/fs/smackfs/load 2>/dev/null

echo "=== Ambient Label ==="
cat /sys/fs/smackfs/ambient 2>/dev/null

echo "=== Logging Policy ==="
cat /sys/fs/smackfs/logging 2>/dev/null

echo "=== Ptrace Rule ==="
cat /sys/fs/smackfs/ptrace 2>/dev/null

echo "=== Netlabel IPv4 mappings ==="
cat /sys/fs/smackfs/netlabel 2>/dev/null

echo "=== Recent Smack Audit Events ==="
ausearch -m AVC -ts recent 2>/dev/null | grep -i smack | tail -20

echo "=== Smack-related kernel messages ==="
dmesg | grep -i smack | tail -30

echo "=== File Labels Check ==="
getfattr -n security.SMACK64 -R /etc 2>/dev/null | head -20

echo "=== Process Labels ==="
ps -eo pid,label,comm 2>/dev/null | head -20
# أو
for pid in $(ls /proc | grep '^[0-9]'); do
    label=$(cat /proc/$pid/attr/current 2>/dev/null)
    comm=$(cat /proc/$pid/comm 2>/dev/null)
    [ -n "$label" ] && echo "PID=$pid LABEL=$label COMM=$comm"
done
```

---

#### تشخيص Denial محدد

```bash
# خطوة 1: اعرف الـ denial
ausearch -m AVC -ts today | grep smack

# مثال output:
# type=AVC msg=audit(1708900000.123:456): smack: DENIED subj=User obj=System req=r
# → process بـ label "User" حاول يقرأ object بـ label "System" وفشل

# خطوة 2: شوف الـ rules الحالية للـ subject
grep "^User " /sys/fs/smackfs/load2

# خطوة 3: أضف الـ rule المطلوب
echo "User System r" > /sys/fs/smackfs/load2

# خطوة 4: تحقق
grep "^User System" /sys/fs/smackfs/load2
```

---

#### ftrace لتتبع smk_access

```bash
# Setup كامل
cd /sys/kernel/tracing

# صفّر
echo 0 > tracing_on
echo > trace
echo > set_ftrace_filter

# فعّل function tracing للـ smack
echo 'smk_access' > set_ftrace_filter
echo 'smk_curacc' >> set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on

# شغّل الـ workload المشكوك فيه
sleep 5

# اقرأ النتيجة
echo 0 > tracing_on
cat trace | head -50

# مثال output:
#          bash-1234  [001] ..... 12345.678: smk_access <-smack_inode_permission
#          bash-1234  [001] ..... 12345.679: smk_curacc <-smack_file_open
```

---

#### التحقق من الـ Label Inheritance عند fork

```bash
# عملية عملية: تتبع الـ label عبر fork/exec
strace -f -e trace=process -e signal=none sh -c 'cat /proc/self/attr/current' 2>&1

# شوف الـ label قبل وبعد exec
echo "Parent label:"; cat /proc/self/attr/current
bash -c 'echo "Child label:"; cat /proc/self/attr/current'

# لو الـ label اتغير بعد exec بسبب SMACK64EXEC xattr
getfattr -n security.SMACK64EXEC /bin/bash
```

---

#### debug الـ smack_known list في memory

```bash
# باستخدام crash tool (الأقوى)
crash /proc/kcore /boot/vmlinux-$(uname -r) <<'EOF'
# عرض الـ smack_known_list
list smack_known.list -s smack_known.smk_known smack_known_list
EOF

# باستخدام gdb على live kernel (يحتاج CONFIG_GDB_SCRIPTS)
gdb /boot/vmlinux-$(uname -r) /proc/kcore -ex \
  'python import linux.lists; linux.lists.list_for_each("smack_known_list", "struct smack_known", "list")'
```

---

#### Example Output وتفسيره

```
$ cat /sys/fs/smackfs/load2
_       _       rwxatl
*       _       rwxatl
User    System  r
App     Data    rw
```

**التفسير:**
- السطر الأول: label `_` (الـ floor) ليه كل الـ access على نفسه
- السطر الثاني: label `*` (الـ star) بيوصل لـ `_` بكل الـ permissions
- السطر الثالث: `User` process ممكن يقرأ بس objects بـ label `System`
- السطر الرابع: `App` يقدر يقرأ ويكتب `Data`

```
$ ausearch -m AVC -ts recent | grep smack
type=AVC msg=audit(1709000000.001:789):
  smack: DENIED
  subj=untrusted_app
  obj=system_data
  req=w
  result=-13
```

**التفسير:** الـ `untrusted_app` process حاول يكتب على `system_data` object وانرفض. الـ `-13` = `EACCES`. الحل: أضف rule أو راجع الـ design.
## Phase 6: سيناريوهات من الحياة العملية

---

### سيناريو 1: Industrial Gateway على AM62x — عملية IPC بتتعطل بسبب Smack Label مش متعرف

#### العنوان
IPC بين Cortex-A53 و MCU firmware بيفشل بعد enable Smack على gateway صناعي

#### السياق
بنبني industrial IoT gateway على **Texas Instruments AM62x** (Cortex-A53 + M4F MCU). البورد بتشغّل Linux على A53، والـ M4F بيشتغل RTOS. التواصل بيتم عبر RPMsg فوق shared memory. Smack كان مفعّل كـ LSM للحماية.

#### المشكلة
بعد تحديث kernel من 5.15 لـ 6.6، عمليات `/dev/rpmsg_ctrl` بدأت ترجع `EACCES`. الـ systemd service المسؤول عن IPC واقف، والـ gateway مش بيستقبل sensor data من الـ M4F.

```bash
# من system journal
kernel: smack: access denied for task=rpmsg_app smk_in=rpmsg_app smk_out=_ reason=no rule
```

#### التحليل
الـ LSM interface الجديد في الـ kernel بدأ يمرر security context عبر `lsm_prop_smack`:

```c
// include/linux/lsm/smack.h
struct lsm_prop_smack {
#ifdef CONFIG_SECURITY_SMACK
    struct smack_known *skp;  // pointer لـ label entry في القائمة الكونية
#endif
};
```

الـ `struct smack_known` جوا `security/smack/smack.h` بيحتوي على:
```c
struct smack_known {
    struct list_head    list;
    struct hlist_node   smk_hashed;
    char               *smk_known;   // النص الفعلي للـ label
    u32                 smk_secid;
    struct netlbl_lsm_secattr smk_netlabel;
    struct list_head    smk_rules;   // قواعد الـ access
    struct mutex        smk_rules_lock;
};
```

الـ `rpmsg_ctrl` device node الـ xattr بتاعته كان فاضي — ما فيش `security.SMACK64` attribute، فالـ Smack عامله بالـ default label `_` (floor). الـ process label `rpmsg_app` ما عندهوش rule للـ `_` label بعد التحديث، لأن policy قديمة اتمسحت.

#### الحل

```bash
# 1. نتحقق من label الـ device node
getfattr -n security.SMACK64 /dev/rpmsg_ctrl
# لا يوجد output — يعني label غير معين

# 2. نعين label واضح للـ device
setfattr -n security.SMACK64 -v "rpmsg" /dev/rpmsg_ctrl

# 3. نضيف access rule في Smack policy
echo "rpmsg_app rpmsg rw" > /sys/fs/smackfs/load

# 4. نتحقق من القواعد المحملة
cat /sys/fs/smackfs/load | grep rpmsg_app
```

لو بدنا نجيب الـ `skp` لـ process معين للـ debug:
```c
// في kernel module مؤقت للتشخيص
#include <linux/lsm/smack.h>

void debug_task_label(struct task_struct *t) {
    struct lsm_prop prop;
    /* get smack prop for task */
    security_task_getsecid(t, &prop);
    /* prop.smack.skp->smk_known يعطيك النص */
    pr_info("task label: %s\n", prop.smack.skp->smk_known);
}
```

#### الدرس المستفاد
**الـ** `lsm_prop_smack` **بيمسك pointer** مش copy — لما الـ label مش موجود في القائمة أو device ما عندوش xattr، الـ Smack بيرجعله الـ default `_` label. بعد أي kernel upgrade، لازم تتحقق من كل device nodes xattrs وتعيد تحميل الـ Smack policy بشكل صريح.

---

### سيناريو 2: Android TV Box على Allwinner H616 — Container Isolation فشل وسمح بـ label escalation

#### العنوان
تطبيق Android TV بيقدر يوصل لـ HDMI control interface بدون إذن بسبب missing Smack config

#### السياق
**Allwinner H616** TV box بيشغّل Android-based Linux. الـ HDMI CEC interface (`/dev/cec0`) محمي بـ Smack. في الـ userspace، فيه isolated app container بـ label `untrusted_app` المفروض ما يوصلش لـ CEC.

#### المشكلة
تطبيق ثالث (third-party) اكتشف إنه يقدر يتحكم في HDMI CEC — يقدر يغير channel، يطفي الـ TV، حتى يعمل volume control. الـ security team لقت إن الـ cec0 device ما عندوش Smack xattr.

```bash
# التطبيق بيعمل ioctl على /dev/cec0 بنجاح
strace -e trace=open,ioctl app_process /system/app/BadApp.apk
# open("/dev/cec0", O_RDWR) = 5   <-- نجح!
```

#### التحليل

الـ Smack check لما process بتحاول تفتح inode بيمشي كده:

```c
// smack_inode_permission داخل smack_lsm.c بيفحص:
// 1. smk_of_inode(inode) → بيجيب smack_known* من inode->i_security
// 2. smk_of_current() → بيجيب smack_known* للـ task الحالي
// 3. بيعمل smk_access() بينهم

// inode_smack struct بيتخزن في inode->i_security:
struct inode_smack {
    struct smack_known *smk_inode; // label of the fso
    struct smack_known *smk_task;
    struct smack_known *smk_mmap;
    int                 smk_flags;
};
```

لما `/dev/cec0` ما عندوش xattr، الـ `smk_inode` بيتعبى بـ default floor label `_`. قاعدة في الـ policy القديمة كانت بتعطي `untrusted_app` access على `_` للـ read — هذا كان مقصود للـ `/proc` entries، بس انطبق كمان على `cec0`.

```c
// لسم_prop_smack هو اللي بيربط الـ prop بالـ smack_known:
struct lsm_prop_smack {
#ifdef CONFIG_SECURITY_SMACK
    struct smack_known *skp; // لو skp->smk_known == "_" → floor label
#endif
};
```

الـ `skp` للـ `cec0` كان يشاور على الـ `_` label entry، والـ rule `untrusted_app _ r` كانت موجودة للـ `/proc` — فالـ access اتسمح.

#### الحل

```bash
# 1. نعين label خاص لـ HDMI CEC
setfattr -n security.SMACK64 -v "hdmi_cec" /dev/cec0

# 2. نمسح الـ rule العامة اللي بتسمح untrusted_app على _
# Smack ما بيدعم حذف rule مباشر — بنعمل policy reset
echo "" > /sys/fs/smackfs/load

# 3. نعيد تحميل policy بشكل صحيح
# في init script:
echo "system_app hdmi_cec rw" > /sys/fs/smackfs/load
echo "media_server hdmi_cec rw" >> /sys/fs/smackfs/load
# untrusted_app مش في القائمة → blocked

# 4. نتحقق
runcon u:r:untrusted_app:s0 cat /dev/cec0
# يجب أن يرجع: Permission denied
```

#### الدرس المستفاد
**الـ** `smack_known *skp` **في** `lsm_prop_smack` **بيشاور على entry مشتركة** في الـ global list. لو اتنين device nodes عندهم نفس الـ label (زي `_`)، أي rule بتعطي access على الـ label دي بتنطبق على الاتنين. كل device حساس لازم عنده label فريد.

---

### سيناريو 3: Automotive ECU على i.MX8 — Smack CONFIG مش مفعّل بسبب build misconfiguration

#### العنوان
الـ `lsm_prop_smack` فاضي دايمًا في ECU بيشغّل i.MX8 بسبب كون `CONFIG_SECURITY_SMACK` مش موجود في `.config`

#### السياق
نبني **automotive ECU** على **NXP i.MX8QM** بيشغّل Linux مع Smack كـ mandatory access control. الـ BSP team مسؤولة عن الـ kernel build، والـ security team مسؤولة عن الـ policy. في production testing، الفريق الأمني لاحظ إن كل process بتقدر توصل لأي resource.

#### المشكلة
```bash
# test: untrusted_proc المفروض ما يقرأش /etc/can_config (label: can_trusted)
cat /etc/can_config
# نجح! بدون أي error
```

```bash
# فحص الـ LSM المفعّل
cat /sys/kernel/security/lsm
# capability,lockdown,yama
# Smack غير موجود في القائمة!
```

#### التحليل

الـ header في السؤال:
```c
// include/linux/lsm/smack.h
struct lsm_prop_smack {
#ifdef CONFIG_SECURITY_SMACK
    struct smack_known *skp;  // هذا الـ field موجود بس لو CONFIG_SECURITY_SMACK=y
#endif
};
```

لأن `CONFIG_SECURITY_SMACK` مش موجود في الـ `.config`، الـ `struct lsm_prop_smack` بيبقى **empty struct**. بمعنى إن أي كود بيفحص `prop.smack.skp` مش بيعمل compile أصلًا، أو لو compile، الـ struct فيه zero bytes ومفيش فحص أمني بيحصل.

```bash
# فحص الـ kernel config
zcat /proc/config.gz | grep SMACK
# CONFIG_SECURITY_SMACK is not set  ← المشكلة هنا
```

الـ BSP team كانت شغّالة على NXP default defconfig اللي ما بيفعّلش Smack:
```bash
# في arch/arm64/configs/imx8_defconfig
# CONFIG_SECURITY_SMACK is not set
```

#### الحل

```bash
# 1. تعديل kernel config
# في menuconfig:
# Security options → Smack support → [*]

# أو مباشرة في .config:
echo "CONFIG_SECURITY_SMACK=y" >> arch/arm64/configs/imx8_ecu_defconfig
echo "CONFIG_SECURITY_SMACK_BRINGUP=y" >> arch/arm64/configs/imx8_ecu_defconfig

# 2. إضافة Smack لقائمة LSMs المفعّلة في kernel cmdline
# في bootloader environment:
setenv bootargs "${bootargs} lsm=capability,yama,smack"
# أو في kernel config:
# CONFIG_LSM="capability,yama,smack"

# 3. بعد rebuild وflash، نتحقق:
cat /sys/kernel/security/lsm
# capability,lockdown,yama,smack

# 4. نتحقق إن struct بيتبنى صح:
# في kernel source:
grep -r "lsm_prop_smack" include/linux/lsm/smack.h
# يجب أن يحتوي skp field
```

```c
// للتحقق من أن Smack فعلًا active في runtime:
#include <linux/lsm/smack.h>

void verify_smack_active(void) {
#ifdef CONFIG_SECURITY_SMACK
    pr_info("Smack is compiled in, skp field available\n");
#else
    pr_warn("Smack is NOT compiled - lsm_prop_smack is empty!\n");
#endif
}
```

#### الدرس المستفاد
**الـ** `#ifdef CONFIG_SECURITY_SMACK` **جوا** `lsm_prop_smack` **مش مجرد guard** — ده بيعني إن الـ struct ممكن يبقى فاضي تمامًا. لازم الـ CI/CD pipeline يتحقق من `CONFIG_SECURITY_SMACK=y` وإن `smack` موجود في `/sys/kernel/security/lsm` كجزء من الـ boot tests في كل automotive build.

---

### سيناريو 4: Custom Board Bring-Up على RK3562 — Smack Label Corruption بعد كتابة xattr على rootfs مش صح

#### العنوان
الـ `smack_known *skp` بيشاور على label قديم بعد filesystem remount على RK3562 tablet

#### السياق
**RK3562** بيستخدم في tablet صناعي بـ custom Android-like Linux. الـ rootfs على eMMC بـ ext4. بعد OTA update بيعمل remount للـ rootfs، بعض الـ processes بدأت تحصل على `EACCES` رغم إن الـ policy صح.

#### المشكلة
```bash
# بعد OTA reboot:
journalctl -b | grep "smack"
# smack: access denied: task=sensor_daemon smk_in=sensor rw smk_out=sensor_new
# smack: access denied: task=sensor_daemon smk_in=sensor_new rw smk_out=sensor
```

اتنين label مختلفين بيظهروا لنفس الـ process! الـ sensor daemon لا بيقدر يكتب في `/var/sensor/data`.

#### التحليل

الـ Smack بيخزن labels في global list:
```c
struct smack_known {
    struct list_head    list;
    struct hlist_node   smk_hashed;
    char               *smk_known;    // pointer لـ string في heap
    u32                 smk_secid;
    // ...
};
```

والـ `lsm_prop_smack` بتمسك:
```c
struct lsm_prop_smack {
#ifdef CONFIG_SECURITY_SMACK
    struct smack_known *skp;  // pointer للـ entry في الـ global list
#endif
};
```

اللي حصل: الـ OTA script غلط عمل `setfattr` لملفات الـ sensor بـ label `sensor_new` بدل `sensor`، بس ما عملش flush للـ dentry cache. الـ process الشغّالة كانت عندها `skp` يشاور على الـ `sensor` entry. بعد remount، الـ kernel قرأ الـ xattr الجديد `sensor_new` وعمل له entry جديدة في الـ global list. دلوقتي الـ process task label = `sensor` والـ file inode label = `sensor_new` — وما فيش rule بينهم.

```bash
# نتحقق من الـ xattr على الملف
getfattr -n security.SMACK64 /var/sensor/data
# security.SMACK64="sensor_new"

# نتحقق من label الـ process
cat /proc/$(pidof sensor_daemon)/attr/current
# sensor
```

#### الحل

```bash
# الخيار 1: نرجع الـ xattr للقيمة الصح
setfattr -n security.SMACK64 -v "sensor" /var/sensor/data
setfattr -n security.SMACK64 -v "sensor" /var/sensor/

# الخيار 2: لو OTA قصد تغيير الـ label، لازم نحدث الـ policy كمان
echo "sensor sensor_new rw" > /sys/fs/smackfs/load
echo "sensor_new sensor rw" >> /sys/fs/smackfs/load

# الخيار 3: في OTA script، نضمن consistency
# قبل تغيير أي label:
echo "old_label new_label rw" > /sys/fs/smackfs/load  # مؤقت
# بعد تغيير كل الملفات والـ process labels:
# نعمل cleanup للـ rules المؤقتة

# نتحقق من الـ labels الموجودة في الـ global list
cat /sys/fs/smackfs/mapped
```

```c
// في kernel: الـ smk_known string بيتخزن مرة واحدة بس
// لو نفس النص اتسجل مرتين (مثلًا whitespace مختلف)
// بيبقى عندنا entry منفصلة وpointer مختلف

// الـ smk_import function بيتحقق من التكرار:
// قبل إنه يعمل entry جديدة، بيفتش في smack_known_list
// لو ما لقاش → بيعمل kmalloc جديد ويضيفه للـ list
```

#### الدرس المستفاد
**الـ** `smack_known *skp` **مش بيتحدث أوتوماتيك** لما xattr الملف يتغير — الـ inode security context بيتحدث عند الـ open التالي بعد cache invalidation. في OTA scripts، أي تغيير في Smack labels لازم يصاحبه تحديث للـ policy ومراعاة الـ running processes.

---

### سيناريو 5: IoT Sensor Node على STM32MP1 — Smack بيمنع UART data collection بسبب socket label مش متعين صح

#### العنوان
Sensor aggregator daemon على STM32MP1 فاشل يقرأ من UART بسبب socket_smack label conflict

#### السياق
**STM32MP1** (Cortex-A7 + M4) بيشغّل Linux على A7. الـ M4 بيقرأ sensor data وبيبعته عبر UART للـ A7. الـ Linux daemon (`sensor_agg`) بيقرأ من `/dev/ttySTM0` ويحوّل البيانات لـ MQTT. Smack مفعّل مع NetLabel للـ network.

#### المشكلة
```bash
# الـ daemon بيفشل في الفتح:
sensor_agg[342]: open /dev/ttySTM0: Permission denied

# في dmesg:
smack: access denied for sensor_agg label=sensor_agg tty_label=system_u
```

#### التحليل

الـ TTY device (`/dev/ttySTM0`) عنده:
```bash
getfattr -n security.SMACK64 /dev/ttySTM0
# security.SMACK64="system_u"
```

الـ `sensor_agg` process عنده label `sensor_agg`. مفيش rule بين `sensor_agg` و `system_u`.

لكن الـ socket layer كمان ليه دور هنا — الـ `socket_smack`:
```c
struct socket_smack {
    struct smack_known *smk_out;    /* outbound label */
    struct smack_known *smk_in;     /* inbound label */
    struct smack_known *smk_packet; /* TCP peer label */
    int                 smk_state;
};
```

لما الـ daemon بيفتح الـ UART، بيعمل في نفس الوقت socket لـ MQTT. الـ `smk_out` للـ socket بيتعين من `lsm_prop_smack.skp` للـ task. لو الـ NetLabel ما عرفش يعين label للـ outbound packet، الـ Smack بيعتبره `UNLABELED` وبيمنع الـ access.

```c
// في smack_lsm.c — smack_socket_sendmsg:
// بيفحص smk_out == label مسموح له بالـ access على الـ destination
// لو smk_state == SMK_NETLBL_UNSET → بيحاول يعمل netlabel
// لو فشل → EACCES
```

الـ `lsm_prop_smack.skp` للـ task بيكون `sensor_agg`، لكن الـ NetLabel configuration مش عارف يعمل map من `sensor_agg` لـ CIPSO label للـ local loopback — وده ممنع في الـ default Smack netlabel config.

#### الحل

```bash
# الخطوة 1: نصلح label الـ TTY
setfattr -n security.SMACK64 -v "sensor_agg" /dev/ttySTM0
# أو نضيف rule بدل ما نغير الـ label:
echo "sensor_agg system_u rw" > /sys/fs/smackfs/load

# الخطوة 2: نضبط NetLabel للـ loopback
echo "CIPSO 127.0.0.0/8" > /sys/fs/smackfs/netlabel

# الخطوة 3: نتحقق من socket state
# نشغّل الـ daemon مع strace للـ smack calls
strace -f -e trace=open,sendmsg,recvmsg sensor_agg 2>&1 | head -50

# الخطوة 4: في DTS — لو بنستخدم device label من DT
# ما فيش Smack DT property مباشرة، لكن:
# udev rules بتتولى ضبط الـ xattr عند boot
# /etc/udev/rules.d/99-smack.rules:
echo 'KERNEL=="ttySTM0", RUN+="/usr/bin/setfattr -n security.SMACK64 -v sensor_agg %N"' \
    > /etc/udev/rules.d/99-smack.rules
```

```c
// للتحقق من الـ socket label بعد الـ open في kernel module:
#include <linux/lsm/smack.h>

void check_socket_label(struct socket *sock) {
    struct socket_smack *ssp = sock->sk->sk_security;
    /* ssp->smk_out->smk_known يعطيك الـ outbound label */
    pr_info("socket out label: %s, state: %d\n",
            ssp->smk_out->smk_known, ssp->smk_state);
}
```

#### الدرس المستفاد
**الـ** `lsm_prop_smack` **بيربط الـ task security context بالـ socket labels** عبر `socket_smack`. على أنظمة embedded زي STM32MP1 مع NetLabel، لازم تتحقق من الـ `smk_state` للـ socket وتضبط الـ netlabel configuration للـ loopback وإلا حتى الـ local UART access ممكن يفشل. استخدم `udev rules` لضبط الـ Smack xattrs تلقائيًا عند boot بدل ما تعتمد على default labels.
## Phase 7: مصادر ومراجع

### مقدمة

الـ file اللي بندرسه — `include/linux/lsm/smack.h` — صغير جداً (17 سطر)، بس هو interface رسمي بين Smack LSM وباقي الـ kernel subsystems. عشان تفهم السياق الكامل لازم ترجع للمصادر دي.

---

### مقالات LWN.net

أهم مصدر لمتابعة تطور Smack في الـ kernel.

| العنوان | الأهمية |
|---|---|
| [Smack for simplified access control](https://lwn.net/Articles/244531/) | أول مقال يشرح Smack لما اتقدم للـ mainline — الأساس |
| [SMACK meets the One True Security Module](https://lwn.net/Articles/252562/) | نقاشات الـ merge وموقف المجتمع من LSM stacking |
| [OLS: Smack for embedded devices](https://lwn.net/Articles/292291/) | شرح Casey Schaufler للـ use cases في الـ embedded systems |
| [Smack namespace](https://lwn.net/Articles/645403/) | إضافة namespace support لـ Smack — تطور مهم |
| [LSM stacking and the future](https://lwn.net/Articles/804906/) | مستقبل تشغيل أكثر من LSM معاً وموقع Smack فيه |
| [A change in direction for security-module stacking?](https://lwn.net/Articles/970070/) | أحدث نقاش حول LSM stacking ودور Casey Schaufler |
| [Adding system calls for Linux security modules](https://lwn.net/Articles/919059/) | syscalls جديدة للـ LSM interface بما فيها Smack |
| [The return of loadable security modules?](https://lwn.net/Articles/526983/) | نقاش حول مستقبل الـ LSM framework |

---

### التوثيق الرسمي للـ Kernel

#### داخل شجرة المصدر

```
Documentation/admin-guide/LSM/Smack.rst   ← الدليل الكامل لإعداد Smack
Documentation/admin-guide/LSM/index.rst   ← فهرس كل الـ LSMs
Documentation/security/                   ← وثائق الـ security subsystem
```

#### على الإنترنت

- [Smack — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/LSM/Smack.html) — الوثيقة الرسمية على docs.kernel.org
- [Linux Security Module Usage](https://static.lwn.net/kerneldoc/admin-guide/LSM/) — دليل استخدام الـ LSM framework
- [Smack.txt (أقدم نسخة)](https://www.kernel.org/doc/Documentation/security/Smack.txt) — النسخة القديمة من الوثيقة

---

### الملفات المصدرية الرئيسية في الـ Kernel

```
include/linux/lsm/smack.h          ← الـ interface الرسمي (الملف اللي بندرسه)
security/smack/smack.h             ← الـ internal definitions وتعريف smack_known
security/smack/smack_lsm.c         ← التطبيق الكامل لكل الـ LSM hooks
security/smack/smackfs.c           ← الـ smackfs pseudo-filesystem للـ configuration
security/smack/smack_access.c      ← منطق التحقق من الـ access rules
security/smack/smack_netfilter.c   ← دعم الـ network filtering
```

الـ GitHub mirror:
- [torvalds/linux — security/smack/smack_lsm.c](https://github.com/torvalds/linux/blob/master/security/smack/smack_lsm.c)

---

### Commits مهمة في تاريخ Smack

| الحدث | الـ Kernel Version |
|---|---|
| أول merge لـ Smack في الـ mainline | **Linux 2.6.25** (2008) |
| إضافة Smack لـ IPv6 | 2.6.28 |
| تطوير Smack networking | 2.6.30 |
| إضافة Smack namespace support | 3.14 / 4.2 |
| إعادة هيكلة الـ LSM interface — `lsm_prop` | 6.x |

للبحث عن commits بالاسم:

```bash
git log --oneline --all -- security/smack/
git log --oneline --all -- include/linux/lsm/smack.h
```

---

### نقاشات الـ Mailing List

- [Casey Schaufler — LSM Framework discussion](https://lkml.kernel.org/lkml/281517.30826.qm@web36613.mail.mud.yahoo.com/) — نقاش تاريخي حول الـ LSM interface
- [LSM stacking for AppArmor — Patchwork](https://patchwork.kernel.org/project/linux-security-module/cover/20190602165101.25079-1-casey@schaufler-ca.com/) — patchset كبير من Casey يوضح كيف تتعامل Smack مع الـ stacking
- [LSM: Maintain a table of LSM attribute data](https://patchwork.kernel.org/project/linux-security-module/patch/20221025184519.13231-5-casey@schaufler-ca.com/) — تطور الـ `lsm_prop` structure
- [smack: fix lots of kernel-doc notation](https://lists.openwall.net/linux-kernel/2009/02/18/446) — نقاشات الـ documentation في الـ mailing list

الـ mailing list المخصصة للأمان:
- `linux-security-module@vger.kernel.org` — المكان الرئيسي لنقاشات Smack وكل الـ LSMs

---

### Kernelnewbies.org

الـ kernel release notes بتوثق التغييرات في Smack عبر الـ versions:

| الـ Release | الرابط |
|---|---|
| Linux 2.6.25 (أول merge لـ Smack) | [kernelnewbies.org/Linux_2_6_25](https://kernelnewbies.org/Linux_2_6_25) |
| Linux 2.6.28 | [kernelnewbies.org/Linux_2_6_28](https://kernelnewbies.org/Linux_2_6_28) |
| Linux 2.6.30 | [kernelnewbies.org/Linux_2_6_30](https://kernelnewbies.org/Linux_2_6_30) |
| Linux 3.5 | [kernelnewbies.org/Linux_3.5](https://kernelnewbies.org/Linux_3.5) |
| Linux 3.14 | [kernelnewbies.org/Linux_3.14](https://kernelnewbies.org/Linux_3.14) |
| Linux 4.2 | [kernelnewbies.org/Linux_4.2](https://kernelnewbies.org/Linux_4.2) |
| Linux 5.2 | [kernelnewbies.org/Linux_5.2](https://kernelnewbies.org/Linux_5.2) |
| Linux 6.4 | [kernelnewbies.org/Linux_6.4](https://kernelnewbies.org/Linux_6.4) |
| Linux 6.9 | [kernelnewbies.org/Linux_6.9](https://kernelnewbies.org/Linux_6.9) |

---

### eLinux.org

- [SMACK for Digital TV and application store security (PDF)](https://elinux.org/images/0/06/Buzov-SMACK.pdf) — ورقة بحثية عن استخدام Smack في الـ embedded/DTV systems — مثال عملي ممتاز

---

### كتب مرجعية

#### Linux Device Drivers (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل ذو الصلة:** Chapter 5 — Concurrency and Race Conditions (فهم الـ kernel locking المستخدم في smack_known)
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل ذو الصلة:** Chapter 17 — Devices and Modules (فهم الـ kernel modules وكيف الـ LSM بيتضمن)
- الكتاب ده بيوفر الخلفية الأساسية لفهم كيف الـ kernel hooks بتشتغل

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل ذو الصلة:** Chapter 16 — Kernel Debugging Techniques / Security
- Smack اتصمم أصلاً للـ embedded — الكتاب ده بيشرح البيئة اللي اتولد فيها

#### The Linux Programming Interface — Michael Kerrisk
- **الفصل ذو الصلة:** Chapter 39 — Capabilities / Chapter 38 — Writing Secure Programs
- بيشرح الـ MAC/DAC model اللي Smack بيبني عليه

---

### مصادر إضافية

#### Wikipedia
- [Smack (software) — Wikipedia](https://en.wikipedia.org/wiki/Smack_(software)) — نظرة عامة وتاريخ

#### Ubuntu Community Help
- [SmackConfiguration — Ubuntu Community Help Wiki](https://help.ubuntu.com/community/SmackConfiguration) — دليل عملي لإعداد Smack على Ubuntu

#### Android / Tizen
- [Android kernel common — smack_lsm.c](https://android.googlesource.com/kernel/common/+/android-trusty-3.10/security/smack/smack_lsm.c) — Smack في الـ Android kernel — مثال على الـ embedded use case
- Tizen OS بتستخدم Smack بشكل كبير — ابحث عن "Tizen Smack" لمزيد من الأمثلة العملية

---

### كلمات البحث المفيدة

للبحث عن معلومات أكثر:

```
smack lsm linux kernel
smack_known struct kernel
lsm_prop smack interface
linux mandatory access control embedded
Casey Schaufler smack lsm
smackfs configuration
linux security module stacking smack
tizen smack security policy
smack label xattr linux
```

في الـ kernel source:

```bash
# كل الملفات المتعلقة بـ Smack
find security/smack/ -type f

# كل الأماكن اللي بتستخدم lsm_prop_smack
grep -r "lsm_prop_smack" --include="*.h" --include="*.c"

# كل الأماكن اللي بتستخدم smack_known
grep -r "smack_known" security/smack/

# الـ LSM hooks اللي Smack بتطبقها
grep "smack_" security/smack/smack_lsm.c | grep -v "^[[:space:]]*//"
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `include/linux/lsm/smack.h` بتعرّف `struct lsm_prop_smack` اللي بتحتوي على pointer لـ `struct smack_known` — ده اللي بيمثّل الـ **Smack label** الخاصة بأي process أو object.

الـ hook الأنسب للـ demo هو `smack_task_kill` — وظيفته إنه Smack بيقرر فيها هل process مسموح لها ترسل signal لـ process تانية. بنستخدم **kprobe** على الـ kernel function دي عشان نطبع الـ label بتاعة الـ subject (المُرسِل) والـ signal.

---

### الـ Makefile

```makefile
obj-m += smack_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### الـ Module

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * smack_probe.c — kprobe on smack_task_kill
 * Prints the Smack subject label and signal number on every kill attempt.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/sched.h>
#include <linux/signal.h>

/*
 * lsm_prop_smack.h defines struct lsm_prop_smack { struct smack_known *skp; }
 * We need struct smack_known to read the label string smk_known.
 * We copy the minimal definition here to avoid pulling in all of smack.h
 * (which is internal to security/smack/).
 */
struct smack_known_min {
    /* list and hash fields omitted — we only need the label */
    struct list_head  list;       /* embedded list for the global label list */
    struct hlist_node smk_hashed; /* hash-table node */
    char             *smk_known;  /* NUL-terminated label string */
};

/*
 * smack_task_kill prototype (from security/smack/smack_lsm.c):
 *
 *   int smack_task_kill(struct task_struct *p,
 *                       struct kernel_siginfo *info,
 *                       int sig,
 *                       const struct cred *cred);
 *
 * p    — target task (receiver of the signal)
 * info — signal info (can be SEND_SIG_NOINFO)
 * sig  — signal number (e.g. 9 for SIGKILL)
 * cred — credentials of the sender (subject)
 */

/* pre-handler: runs just before smack_task_kill executes */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Pull args from registers — matches the x86-64 System V ABI:
     *   rdi = p (task_struct*)
     *   rsi = info (kernel_siginfo*)
     *   rdx = sig (int)
     *   rcx = cred (const struct cred*)
     */
    struct task_struct    *target = (struct task_struct *)regs->di;
    int                    sig    = (int)regs->dx;
    const struct cred     *cred   = (const struct cred *)regs->cx;

    /*
     * cred->security holds the LSM blob. For Smack the first slot in
     * the blob is a struct smack_known pointer (the subject label).
     * We read it as a pointer-to-pointer and dereference one level.
     *
     * NOTE: This is a best-effort introspection — it works when Smack
     * is the only LSM or the first stacked LSM.  In production, use
     * security_cred_getsecid() + secid_to_secctx() instead.
     */
    struct smack_known_min *skp = NULL;

    if (cred && cred->security)
        skp = *(struct smack_known_min **)cred->security;

    pr_info("smack_probe: pid=%d comm=%-16s -> pid=%d sig=%d label=%s\n",
            current->pid,
            current->comm,
            target ? task_pid_nr(target) : -1,
            sig,
            (skp && skp->smk_known) ? skp->smk_known : "<unknown>");

    return 0; /* 0 = keep executing the original function */
}

/* post-handler: runs after smack_task_kill returns (optional, left minimal) */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* ax holds the return value on x86-64 */
    long ret = (long)regs->ax;
    if (ret != 0)
        pr_info("smack_probe: smack_task_kill DENIED (ret=%ld)\n", ret);
}

/* kprobe descriptor — we probe by symbol name, no manual address needed */
static struct kprobe kp = {
    .symbol_name = "smack_task_kill",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

static int __init smack_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("smack_probe: register_kprobe failed: %d\n", ret);
        pr_err("smack_probe: is CONFIG_SECURITY_SMACK=y and kprobes enabled?\n");
        return ret;
    }

    pr_info("smack_probe: planted kprobe at %pS\n", kp.addr);
    return 0;
}

static void __exit smack_probe_exit(void)
{
    /*
     * Unregister MUST happen in exit — leaving a kprobe pointing at a
     * handler that no longer exists causes an immediate kernel panic on
     * the next probe hit.
     */
    unregister_kprobe(&kp);
    pr_info("smack_probe: kprobe removed\n");
}

module_init(smack_probe_init);
module_exit(smack_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Demo");
MODULE_DESCRIPTION("kprobe on smack_task_kill — logs Smack label on signal send");
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
```

**الـ** `kprobes.h` بتجيب `struct kprobe` وكل functions التسجيل `register_kprobe` / `unregister_kprobe`. من غيرها مش هينفع نزرع probe.

```c
#include <linux/sched.h>
```

محتاجينها عشان `struct task_struct` و `current` و `task_pid_nr()` — ده اللي بنستخدمه نطبع معلومات الـ process.

---

#### الـ `struct smack_known_min`

**الـ** `smack_known` الأصلية معرّفة في `security/smack/smack.h` اللي هي internal header مش exported. بنعرّف نسخة مبسّطة بس بالـ fields اللي محتاجينها (`smk_known` اللي هو الـ label string) عشان نقدر نقرأها بأمان.

---

#### الـ `handler_pre`

ده اللي بيتنفّذ **قبل** ما `smack_task_kill` تشتغل. بياخد الـ arguments من registers (x86-64 ABI):

| Register | Argument |
|----------|----------|
| `rdi`    | `task_struct *p` (الـ target) |
| `rsi`    | `kernel_siginfo *info` |
| `rdx`    | `int sig` |
| `rcx`    | `const struct cred *cred` |

من `cred->security` بنقرأ pointer الـ Smack label الخاصة بالـ sender وبنطبعها مع الـ signal number. ده بيديك visibility على كل محاولة إرسال signal في النظام وإيه الـ Smack label اللي بتحاول.

---

#### الـ `handler_post`

بيتنفّذ **بعد** ما الـ function ترجع. بنتحقق من الـ return value — لو مش صفر معناه إن Smack رفض الـ operation. مش إلزامي بس بيكمّل الصورة.

---

#### الـ `struct kprobe kp`

```c
.symbol_name = "smack_task_kill",
```

**الـ** kernel بيحوّل الـ symbol name لـ address وقت التسجيل تلقائيًا عن طريق `kallsyms` — مش محتاج تحدد عنوان يدوي.

---

#### الـ `module_init` / `module_exit`

**الـ** `register_kprobe` بتزرع الـ breakpoint في memory وقت الـ load. الـ `unregister_kprobe` في `exit` ضروري جدًا — من غيرها لو الـ module اتشيل وحصل signal، الـ kernel هيجري handler مش موجود في memory وهيحصل **kernel panic**.

---

### تجربة الـ Module

```bash
# تأكد إن Smack مفعّل في الـ kernel
grep CONFIG_SECURITY_SMACK /boot/config-$(uname -r)
# لازم يطلع: CONFIG_SECURITY_SMACK=y

# بناء وتحميل
make
sudo insmod smack_probe.ko

# بعث signal لأي process عشان تشتغل الـ probe
kill -0 $$

# شوف الـ output
sudo dmesg | grep smack_probe

# إزالة الـ module
sudo rmmod smack_probe
```

**مثال output متوقع:**

```
smack_probe: planted kprobe at smack_task_kill+0x0/0x60
smack_probe: pid=1234 comm=bash             -> pid=1234 sig=0 label=_
smack_probe: kprobe removed
```

الـ label `_` هي الـ Smack default label (floor label) اللي بتتبعت لأي process مش ليها label صريحة.
