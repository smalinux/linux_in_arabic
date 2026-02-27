## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **Linux Security Module (LSM)** framework — الـ subsystem اللي بيسمح للـ kernel يشتغل مع أكتر من نظام أمان في نفس الوقت (SELinux، AppArmor، Smack، BPF LSM).

---

### الصورة الكبيرة — قبل أي كود

#### تخيل المشكلة

تخيل إنك بتدير مستشفى كبير. في المستشفى ده في **3 أنظمة تعريف مختلفة**:
- نظام SELinux بيعرف كل موظف بـ **رقم وظيفي** (u32).
- نظام AppArmor بيعرف كل موظف بـ **ملف شخصي كامل** (struct pointer).
- نظام Smack بيعرف كل موظف بـ **بطاقة تصنيف أمني** (struct pointer).

لما حد بره المستشفى — مثلاً شركة تأمين — يسأل "مين اللي عمل الـ operation دي؟" محتاج جواب موحد يفهمه بغض النظر عن أي نظام تعريف الجواب جاي منه.

الـ LSM framework محتاج **طريقة موحدة** يعبر بيها عن "هوية الأمان" لأي process أو inode أو socket — حتى لو كل LSM بيمثل الهوية دي بشكل مختلف.

---

#### الحل: الـ `lsm_prop` و الـ per-LSM structs

الـ kernel عمل **struct مجمع** اسمه `lsm_prop` جوا `include/linux/security.h`:

```c
struct lsm_prop {
    struct lsm_prop_selinux  selinux;   /* SELinux's secid (u32) */
    struct lsm_prop_smack    smack;     /* Smack's label pointer */
    struct lsm_prop_apparmor apparmor;  /* AppArmor's aa_label pointer */
    struct lsm_prop_bpf      bpf;       /* BPF LSM's secid (u32) */
};
```

كل LSM عنده **field خاص بيه** جوا الـ struct ده. الـ file اللي بندرسه —
`include/linux/lsm/selinux.h` — بيعرّف **الـ field الخاص بـ SELinux** بس:

```c
struct lsm_prop_selinux {
#ifdef CONFIG_SECURITY_SELINUX
    u32 secid;   /* SELinux security identifier — عدد صحيح 32-bit */
#endif
};
```

---

### ليه الـ `secid` بالتحديد؟

الـ SELinux بيمثل أي "security context" (زي `system_u:system_r:kernel_t:s0`) بـ **رقم integer صغير** اسمه **secid**. الـ secid ده هو اختصار للـ context الكامل — زي رقم الكود البريدي بدل ما تكتب العنوان كامل.

لما الـ kernel محتاج يتحقق من صلاحيات process أو يسجلها في الـ audit log، بيشيل الـ secid من الـ `lsm_prop_selinux` ويبعته لـ SELinux عشان يترجمه.

---

### ليه الـ `#ifdef CONFIG_SECURITY_SELINUX`؟

لو الـ kernel اتبنى من غير SELinux support، الـ struct هيبقى **فاضي تماماً** — مش هياخد أي memory. ده جزء من فلسفة الـ LSM: كل حاجة optional وبتتحسب في الـ compile time.

---

### القصة الكاملة بالترتيب

```
Process بيعمل open() على file
        │
        ▼
Kernel يستدعي security_inode_getlsmprop()
        │
        ▼
كل LSM مفعّل يحط بياناته في lsm_prop
  ┌─────────────────────────────┐
  │ lsm_prop.selinux.secid = 42 │  ← SELinux
  │ lsm_prop.smack.skp = 0xABC  │  ← Smack
  │ lsm_prop.apparmor.label=... │  ← AppArmor
  └─────────────────────────────┘
        │
        ▼
Kernel يبعت lsm_prop للـ audit أو network أو IPC
        │
        ▼
كل subsystem يسحب الـ field الخاص بالـ LSM اللي هو عارفه
```

---

### أهمية الـ file ده

الـ file صغير جداً (17 سطر) لكنه **أساسي**:

- بيفصل الـ "contract" بين SELinux وباقي الـ kernel.
- لو SELinux غير طريقة تعريف الهوية (مثلاً من u32 لـ u64)، التغيير هيبقى **هنا بس** من غير ما تتأثر باقي أجزاء الـ kernel.
- بيحقق **compile-time zero-cost**: لو SELinux مش مفعّل، الـ struct فاضي تماماً.

---

### الفرق بين الـ LSM per-prop headers

| الـ Header | الـ LSM | طريقة التعريف |
|---|---|---|
| `lsm/selinux.h` | SELinux | `u32 secid` — رقم بسيط |
| `lsm/smack.h` | Smack | `struct smack_known *skp` — pointer لـ label |
| `lsm/apparmor.h` | AppArmor | `struct aa_label *label` — pointer لـ label |
| `lsm/bpf.h` | BPF LSM | `u32 secid` — رقم بسيط |

الـ SELinux والـ BPF LSM بيستخدموا الـ u32 لأن كلهم بيخزنوا الـ contexts في **internal tables** ويرجعوا index عليها.

---

### الملفات المرتبطة اللي المفروض تعرفها

| الملف | الدور |
|---|---|
| `include/linux/lsm/selinux.h` | **الملف ده** — تعريف `lsm_prop_selinux` |
| `include/linux/security.h` | تعريف `lsm_prop` الكامل + كل الـ security API |
| `include/linux/lsm/smack.h` | نفس الفكرة لـ Smack |
| `include/linux/lsm/apparmor.h` | نفس الفكرة لـ AppArmor |
| `include/linux/lsm/bpf.h` | نفس الفكرة لـ BPF LSM |
| `include/linux/lsm_hook_defs.h` | تعريفات الـ hooks اللي كل LSM بيطبقها |
| `include/linux/lsm_hooks.h` | الـ infrastructure لتسجيل الـ LSM hooks |
| `security/selinux/` | الـ SELinux implementation الكاملة |
| `include/uapi/linux/lsm.h` | الـ userspace API للـ LSM |
| `include/linux/lsm_audit.h` | دعم الـ audit للـ LSM |

---

### ملفات الـ Subsystem الأساسية

**Core (LSM Framework):**
- `security/security.c` — الـ core dispatcher
- `security/lsm_syscalls.c` — الـ system calls
- `include/linux/security.h` — الـ public API

**Headers (per-LSM props):**
- `include/linux/lsm/selinux.h`
- `include/linux/lsm/smack.h`
- `include/linux/lsm/apparmor.h`
- `include/linux/lsm/bpf.h`

**SELinux Implementation:**
- `security/selinux/hooks.c` — الـ hook implementations
- `security/selinux/avc.c` — الـ Access Vector Cache
- `security/selinux/ss/` — الـ security server (policy engine)

**Testing:**
- `tools/testing/selftests/lsm/` — الـ selftests
## Phase 2: شرح الـ LSM (Linux Security Module) Framework وموقع SELinux فيه

---

### المشكلة اللي بيحلها الـ LSM

الـ kernel بطبيعته بيتحكم في كل عملية وصول للموارد — ملفات، processes، network sockets، IPC. الـ access control التقليدي بيعتمد على **DAC (Discretionary Access Control)**: المالك بيحدد الصلاحيات عبر `chmod`/`chown`، وأي process بـ `root` بيقدر يعمل أي حاجة.

المشكلة؟

- لو process اتاختُرق وعنده `root`، اللعبة خلصت.
- الـ DAC مش كافي في بيئات multi-tenant أو embedded بتتطلب **MAC (Mandatory Access Control)**: السياسة بيحددها الـ admin مسبقاً وما ينفعش أي process يتجاوزها حتى لو `root`.

بس كل نظام MAC له فلسفته الخاصة:
- **SELinux** بيشتغل بـ security labels و type enforcement.
- **AppArmor** بيشتغل بـ pathname-based profiles.
- **Smack** بيشتغل بـ simplified label-based access.
- **BPF LSM** بيشتغل بـ eBPF programs.

المعضلة: إيه الطريقة اللي تخلي الـ kernel يدعم كل دول **في نفس الوقت** من غير ما يبقى كود الـ kernel نفسه متشابك بـ `#ifdef CONFIG_SELINUX` في كل حتة؟

---

### الحل: الـ LSM Framework

الـ **LSM (Linux Security Module) Framework** هو طبقة abstraction في الـ kernel بتفصل بين:

1. **النقاط الحساسة في الـ kernel** (زي لما process بتحاول تفتح ملف أو ترسل packet).
2. **منطق صح الصلاحية الفعلي** اللي بيتقرر في الـ LSM نفسه (SELinux أو AppArmor إلخ).

بدلاً من ما الـ kernel يعمل hardcoded check لـ SELinux، هو بيعمل **hook call** — زي callback function — في كل نقطة حساسة. كل LSM مسجّل في الـ framework بيربط نفسه بالـ hooks دي. لو SELinux مفعّل، الـ hook بتاعته بيتنفذ. لو Smack مفعّل كمان، hook بتاعه بيتنفذ بعده.

النتيجة: الـ kernel core نظيف، والـ security logic معزول تماماً في الـ LSM.

---

### تشبيه من الواقع — مفصّل

تخيل **مطار دولي** فيه security checkpoints:

| عنصر المطار | المقابل في الـ Kernel |
|---|---|
| المسافر بيعدي من بوابة | Process بتعمل syscall (مثلاً `open()`) |
| الـ checkpoint نفسه | الـ LSM hook point في كود الـ kernel |
| قواعد الجواز السفري (دولة A) | SELinux policy |
| قواعد الفيزا (دولة B) | AppArmor profile |
| ضابط الأمن اللي بيفحص | الـ security hook function المسجلة |
| ID الراكب (رقم) | الـ `secid` — الـ u32 في `lsm_prop_selinux` |
| الـ boarding pass الموحد | الـ `lsm_prop` struct اللي بيجمع كل LSMs |

كل checkpoint (hook) ممكن يكون فيه أكتر من ضابط (أكتر من LSM). كل واحد بيفحص بـ قواعده هو. لو أي واحد رفض، العملية بتتوقف.

الـ `secid` هو زي رقم الجواز — رقم واحد بيعرّف المسافر لجهة أمنية معينة (SELinux). مش اسمه، مش تفاصيله — بس رقم فريد تقدر تبحث عنه في الـ policy database.

---

### الـ Big Picture — Architecture

```
User Space
    │
    │  syscall (open, connect, exec, ...)
    ▼
┌─────────────────────────────────────────────────────┐
│                   Kernel Core                        │
│                                                      │
│  VFS / Net / IPC / MM / ...                          │
│       │                                              │
│       │  <hook points — security_*(...)>             │
│       ▼                                              │
│  ┌─────────────────────────────────────────────┐    │
│  │         LSM Framework (security.c)           │    │
│  │                                              │    │
│  │  static_calls_table                          │    │
│  │  ┌──────────┬──────────┬──────────┐          │    │
│  │  │  slot[0] │  slot[1] │  slot[2] │ per hook │    │
│  │  └────┬─────┴────┬─────┴──────────┘          │    │
│  └───────┼──────────┼───────────────────────────┘    │
│          │          │                                 │
│          ▼          ▼                                 │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│   │ SELinux  │  │  Smack   │  │AppArmor  │           │
│   │  hooks   │  │  hooks   │  │  hooks   │           │
│   └──────────┘  └──────────┘  └──────────┘           │
│                                                      │
│   Security Blobs (in cred, inode, file, sock, ...)   │
│   ┌───────────────────────────────────────────┐      │
│   │ cred->security → [SELinux blob][Smack blob]│      │
│   └───────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────┘
```

---

### الـ Core Abstractions

#### 1. الـ `lsm_prop` و `lsm_prop_selinux` — هوية الـ security

```c
/* include/linux/lsm/selinux.h */
struct lsm_prop_selinux {
#ifdef CONFIG_SECURITY_SELINUX
    u32 secid;   /* SELinux security identifier — opaque integer */
#endif
};

/* include/linux/security.h */
struct lsm_prop {
    struct lsm_prop_selinux selinux;
    struct lsm_prop_smack   smack;
    struct lsm_prop_apparmor apparmor;
    struct lsm_prop_bpf     bpf;
};
```

الـ `lsm_prop` هو الـ **unified security identity** للـ kernel. أي entity (process، socket، network packet) لما تحتاج تعبّر عن هويتها الأمنية لأكتر من LSM، بتستخدم `lsm_prop`. كل LSM عنده حقله الخاص جواها.

الـ **`secid`** هو قلب التصميم في SELinux:
- عدد صحيح `u32` بيمثل **security context** كامل (مثلاً `system_u:system_r:httpd_t:s0`).
- الـ SELinux kernel module بيحتفظ بـ mapping داخلي بين الـ `secid` وبين الـ context string الحقيقية.
- مش بتشوف الـ string في الـ kernel paths الحساسة — بس الـ integer. ده بيخلي المقارنة والتخزين سريعين جداً.

#### 2. الـ Hook System — آلية الاستدعاء

```c
/* lsm_hooks.h */
union security_list_options {
    /* لكل hook مُعرَّف في lsm_hook_defs.h، في function pointer هنا */
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) RET (*NAME)(__VA_ARGS__);
    #include "lsm_hook_defs.h"
    #undef LSM_HOOK
    void *lsm_func_addr;
};

struct security_hook_list {
    struct lsm_static_call *scalls;  /* pointer لـ static call slots */
    union security_list_options hook; /* الـ callback الفعلي */
    const struct lsm_id *lsmid;      /* مين اللي سجّل الـ hook ده */
} __randomize_layout;
```

الـ `__randomize_layout` مهم: بيخلي الـ struct layout عشوائياً بين boots عشان يصعّل الـ exploitation.

#### 3. الـ Static Calls Table — الأداء

```c
struct lsm_static_calls_table {
    #define LSM_HOOK(RET, DEFAULT, NAME, ...) \
        struct lsm_static_call NAME[MAX_LSM_COUNT];
    #include <linux/lsm_hook_defs.h>
    #undef LSM_HOOK
} __packed __randomize_layout;
```

الـ **Static Calls** (مش function pointers عادية) ده optimization مهم جداً:

- الـ function pointer العادي: بيعمل indirect branch → CPU branch predictor بيعاني → performance hit.
- الـ **static call**: بيكتب `CALL <address>` مباشرة في الـ code عند الـ boot، وممكن يتعدّل بعدين بـ `text_poke`. بيديك أداء قريب من الـ direct call مع مرونة الـ runtime update.

```
Static Call Mechanism:

  security_file_open()
       │
       ▼
  [CALL 0xXXXXXXXX]  ← patched at boot
       │
       ├─► selinux_file_open()   (slot[0])
       │
       └─► smack_file_open()    (slot[1])
```

#### 4. الـ Blob System — تخزين الـ security data

```c
struct lsm_blob_sizes {
    unsigned int lbs_cred;       /* bytes needed in struct cred */
    unsigned int lbs_file;       /* bytes needed in struct file */
    unsigned int lbs_inode;      /* bytes needed in struct inode */
    unsigned int lbs_sock;       /* bytes needed in struct sock */
    unsigned int lbs_superblock;
    unsigned int lbs_task;
    /* ... */
};
```

كل LSM بيحتاج يخزن data خاصة بيه في الـ kernel objects (مثلاً SELinux محتاج يخزن الـ `secid` في كل `inode`، AppArmor محتاج profile pointer). بدلاً من إضافة fields لكل struct في الـ kernel، الـ LSM framework بيعمل **dynamic blob allocation**:

```
struct inode
┌────────────────────┐
│  ... inode fields  │
├────────────────────┤
│  void *i_security  │──► ┌──────────────┬─────────────┐
│                    │    │ SELinux data │ AppArmor data│
└────────────────────┘    └──────────────┴─────────────┘
                           ▲              ▲
                     offset 0      offset = selinux blob size
```

الـ `lbs_*` fields بتحدد الحجم. الـ framework بيجمعهم عند الـ boot ويعمل allocation واحد.

#### 5. تسجيل الـ LSM — `lsm_info` و `DEFINE_LSM`

```c
struct lsm_info {
    const struct lsm_id *id;       /* اسم وـ ID */
    enum lsm_order order;          /* capabilities أول، integrity آخر */
    unsigned long flags;
    struct lsm_blob_sizes *blobs;  /* كام bytes محتاج في كل object */
    int *enabled;
    int (*init)(void);             /* دالة الـ init الرئيسية */
    /* initcalls لـ phases مختلفة من الـ boot */
    int (*initcall_early)(void);
    int (*initcall_core)(void);
    int (*initcall_device)(void);
    /* ... */
};

/* الـ macro اللي بيسجّل الـ LSM في special ELF section */
#define DEFINE_LSM(lsm)                         \
    static struct lsm_info __lsm_##lsm          \
        __used __section(".lsm_info.init")      \
        __aligned(sizeof(unsigned long))
```

الـ `DEFINE_LSM` بيحط الـ `lsm_info` struct في section اسمه `.lsm_info.init` في الـ ELF binary. الـ kernel عند الـ boot بيـ iterate على الـ section دي ويجمع كل الـ LSMs المسجلة تلقائياً — زي `initcall` mechanism بالظبط.

---

### العلاقة بين الـ Structs

```
lsm_info
┌──────────────────┐
│ id → lsm_id      │──► { name: "selinux", id: LSM_ID_SELINUX }
│ order            │
│ blobs →          │──► lsm_blob_sizes { lbs_cred=X, lbs_inode=Y, ... }
│ init()           │
└──────────────────┘

             registers hooks via security_add_hooks()
                          │
                          ▼
security_hook_list[]
┌──────────────────────────────────┐
│ scalls → static_calls_table.hook │──► lsm_static_call[MAX_LSM_COUNT]
│ hook.file_open = selinux_file_open│        ├─ key (static_call_key)
│ lsmid → { "selinux", id }        │        ├─ trampoline
└──────────────────────────────────┘        ├─ hl (back-pointer)
                                            └─ active (jump_label)

At runtime: security_file_open(file)
    └─► iterates static_calls_table.file_open[]
          ├─► selinux_file_open()   → checks secid in file->f_security
          └─► smack_file_open()    → checks smack label
```

---

### الـ lsm_context — تمثيل الـ security label كـ string

```c
/* security.h */
struct lsm_context {
    char *context;  /* "system_u:object_r:httpd_t:s0" مثلاً */
    u32   len;
    int   id;       /* أي LSM أنشأ الـ context ده؟ */
};
```

ده بيُستخدم لما الـ userspace يطلب يعرف الـ security context بصيغة human-readable (عبر `/proc/*/attr/current` أو `getxattr`). الـ kernel بيحوّل الـ `secid` → string هنا فقط، مش في الـ hot paths.

---

### إيه اللي الـ LSM Framework بيملكه، وإيه اللي بيفوّضه للـ LSM

| المسؤولية | مين بيعملها |
|---|---|
| تحديد نقاط الـ hooks في الـ kernel | الـ LSM Framework (hardcoded في kernel code) |
| تسجيل الـ callbacks على الـ hooks | الـ LSM نفسه (SELinux, AppArmor, ...) |
| ترتيب استدعاء الـ LSMs | الـ LSM Framework حسب `lsm_order` |
| تخصيص مساحة الـ security blobs | الـ LSM Framework بناءً على `lsm_blob_sizes` |
| **تعريف معنى** الـ `secid` | SELinux نفسه — Framework ما يعرفش إيه معناه |
| تحميل الـ security policy | الـ LSM نفسه (SELinux policy database) |
| تحويل `secid` لـ string context | الـ LSM نفسه |
| القرار: allow/deny | الـ LSM نفسه بناءً على policy |
| إبلاغ الـ audit subsystem | الـ LSM نفسه (بيستخدم الـ audit framework) |

> الـ **audit subsystem** هو kernel subsystem مسؤول عن تسجيل الأحداث الأمنية — يعني لو SELinux رفض عملية، بيبعت event للـ audit daemon في userspace.

---

### مثال حقيقي — ماذا يحدث لما تعمل `open("/etc/shadow", O_RDONLY)`؟

```
Process (evil_app, secid=1234)
    │
    │  open("/etc/shadow")
    ▼
sys_open()
    │
    ▼
do_filp_open()
    │
    ├─► vfs_open()
    │       │
    │       └─► security_file_open(file)   ← hook point
    │                   │
    │                   ├─► selinux_file_open(file)
    │                   │       ├─ جيب secid الـ process: 1234 (httpd_t)
    │                   │       ├─ جيب secid الـ file: 5678 (shadow_t)
    │                   │       ├─ ابحث في الـ policy: هل httpd_t مسموح يقرأ shadow_t؟
    │                   │       └─ NO → return -EACCES
    │                   │
    │                   └─► (Smack/AppArmor hooks لو مفعلين كمان)
    │
    └─► -EACCES returned to userspace
```

الـ `secid` هو مفتاح الـ lookup في الـ SELinux policy. بدونه، كل operation كانت هتحتاج تعمل string comparison على الـ security context الكاملة — بطيء جداً.

---

### ليه `secid` هو `u32` وبس؟

ده **تصميم متعمد**. الـ SELinux في الـ kernel paths الحساسة (file open, socket connect, exec, ...) بيتعامل بس مع `u32`. ده بيضمن:

1. **السرعة**: مقارنة `u32` vs مقارنة string.
2. **العزل**: الـ kernel الـ core مش محتاج يعرف أي حاجة عن الـ SELinux policy format.
3. **الـ ABI stability**: الـ interface بين الـ LSM framework وبين الباقي بيبقى ثابت حتى لو SELinux غيّر الـ policy format الداخلي.

الـ translation من `secid` لـ human-readable string بيحصل فقط في:
- `procfs` (`/proc/*/attr/`)
- `xattr` operations
- Audit log messages

وده intentionally خارج الـ hot path.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### نظرة عامة على الملف

الملف `include/linux/lsm/selinux.h` هو من أبسط header files في الـ kernel — بالحرف. يحتوي على struct واحدة بس، وfield واحدة مشروطة بـ `CONFIG_SECURITY_SELINUX`. دوره هو تعريف **الـ SELinux security property** اللي بتتعامل معاها الـ LSM (Linux Security Module) framework كـ opaque value.

---

### 0. الـ Config Options والـ Flags

| Option / Macro | النوع | الأثر |
|---|---|---|
| `CONFIG_SECURITY_SELINUX` | Kconfig bool | لو مفعّل، الـ `secid` field بتتضمن في الـ struct. لو مش مفعّل، الـ struct بتبقى فاضية تمامًا. |
| `__LINUX_LSM_SELINUX_H` | Include guard | يمنع double-inclusion للـ header. |

> **ملاحظة مهمة**: مفيش enums، مفيش flags، مفيش constants تانية في الملف ده. كل التعقيد موجود في `security/selinux/` مش هنا.

---

### 1. الـ Structs الموجودة في الملف

#### `struct lsm_prop_selinux`

**الغرض**: تمثّل الـ **SELinux security label** الخاصة بـ subject أو object في الـ kernel. الـ LSM framework بتستخدم الـ struct دي كجزء من الـ `lsm_prop` union الأكبر منها، عشان تخزّن الـ security context بتاع SELinux كـ `u32 secid`.

```c
struct lsm_prop_selinux {
#ifdef CONFIG_SECURITY_SELINUX
    u32 secid;  /* 32-bit security identifier — index into SELinux policy tables */
#endif
};
```

| Field | النوع | الوصف |
|---|---|---|
| `secid` | `u32` | **Security Context ID** — رقم 32-bit بيشير لـ security context معيّن في policy database بتاع SELinux. مش pointer، مش string — مجرد رقم integer. |

**إمتى بتبقى الـ struct فاضية؟**
لو `CONFIG_SECURITY_SELINUX=n`، الـ struct بتتجمّع كـ zero-size struct. الـ LSM framework بتتعامل مع ده بـ `#ifdef` guards في الكود اللي بيستخدمها.

**علاقتها بالـ structs التانية:**

الـ `lsm_prop_selinux` مش بتشتغل لوحدها. هي عضو في `struct lsm_prop` اللي موجودة في `include/linux/lsm/lsm.h` (مش موجودة في الملف ده مباشرة). الـ `lsm_prop` بتكون union أو struct بتجمّع الـ security properties من كل الـ LSMs المفعّلة.

---

### 2. مخطط علاقة الـ Structs (ASCII)

```
include/linux/lsm/selinux.h
┌─────────────────────────────┐
│   struct lsm_prop_selinux   │
│  ┌──────────────────────┐   │
│  │  u32 secid           │   │  ← 32-bit opaque index
│  │  (if SELINUX enabled) │  │
│  └──────────────────────┘   │
└─────────────┬───────────────┘
              │ embedded in
              ▼
┌─────────────────────────────────────────┐
│   struct lsm_prop                       │
│   (include/linux/security.h or lsm.h)  │
│  ┌──────────────────────────────────┐   │
│  │  struct lsm_prop_selinux selinux │   │
│  │  struct lsm_prop_smack   smack   │   │
│  │  struct lsm_prop_apparmor aa     │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
              │ used by
              ▼
┌──────────────────────────────────────────────────┐
│  kernel objects: task_struct, inode, socket, ... │
│  كل object عنده lsm_prop عشان يخزّن security info │
└──────────────────────────────────────────────────┘
```

---

### 3. مخطط الـ Lifecycle

```
Boot / Policy Load
─────────────────
SELinux policy file يتحمل من الـ userspace (load_policy)
    │
    ▼
الـ SELinux engine يبني policy tables
الـ secids بتتخصّص لكل (type, role, user, range) tuple
    │
    ▼
Object Creation (e.g., process fork / file create)
────────────────────────────────────────────────────
kernel يخلق object جديد (task_struct / inode / socket)
    │
    ▼
LSM hook يتنادى (e.g., security_task_alloc)
    │
    ▼
SELinux hook implementation بتحسب الـ secid المناسب
بتعمل lookup في policy tables
    │
    ▼
بتكتب الـ secid في lsm_prop_selinux.secid
    │
    ▼
Object Usage
────────────
أي access check (read / write / exec / connect ...)
    │
    ▼
LSM hook يقرأ lsm_prop_selinux.secid من الـ subject والـ object
    │
    ▼
SELinux بتعمل policy lookup بالـ secids
    │
    ▼
تقرر: ALLOW أو DENY + تسجّل في الـ audit log
    │
    ▼
Object Teardown
───────────────
Object بيتحذف (process exit / file unlink)
الـ secid بيتحرر ضمنيًا مع الـ struct نفسها
(مفيش deallocation منفصل — مجرد u32)
```

---

### 4. مخطط الـ Call Flow

#### سيناريو: process بيحاول يفتح ملف

```
userspace: open("secret_file", O_RDONLY)
    │
    ▼
sys_openat() — kernel syscall entry
    │
    ▼
security_inode_permission()     ← LSM hook dispatcher
    │
    ▼
selinux_inode_permission()      ← SELinux hook implementation
    │
    ├── اقرأ secid من task_struct   → lsm_prop_selinux.secid  (subject SID)
    ├── اقرأ secid من inode         → lsm_prop_selinux.secid  (object SID)
    │
    ▼
avc_has_perm(ssid, tsid, tclass, requested)
    │
    ├── بحث في الـ AVC cache (Access Vector Cache) أولًا
    │     │
    │     ├── HIT  → رجّع القرار المخزّن مباشرة
    │     │
    │     └── MISS → اعمل policy lookup
    │                    │
    │                    ▼
    │               security_compute_av()
    │               بتحسب الـ allow/deny بناءً على الـ policy rules
    │
    ▼
القرار: 0 (allowed) أو -EACCES (denied)
    │
    ▼
sys_openat() يكمل أو يرجع error للـ userspace
```

#### سيناريو: تخصيص الـ secid لـ process جديد

```
fork() / clone()
    │
    ▼
copy_process()
    │
    ▼
security_task_alloc()           ← LSM hook
    │
    ▼
selinux_task_alloc()
    │
    ├── اقرأ الـ secid من parent task
    │
    ▼
بنسخ الـ lsm_prop_selinux.secid من parent لـ child
(الـ child بيورث الـ security context من parent)
    │
    ▼
task_struct جديدة بـ secid صح
```

---

### 5. استراتيجية الـ Locking

الـ `struct lsm_prop_selinux` نفسها مجرد `u32` — مفيش locking على مستوى الـ struct دي مباشرة. الـ locking بيحصل في المستويات اللي فوقيها:

| ما بيتحمى | الـ Lock المستخدم | المكان |
|---|---|---|
| الـ `secid` field في الـ `task_struct` | الـ `task_lock()` أو RCU | عند القراءة/الكتابة على الـ task security info |
| الـ `secid` في الـ `inode` | الـ `inode->i_lock` أو RCU read lock | عند access checks على الـ files |
| الـ AVC cache (Access Vector Cache) | الـ `avc_cache.lock` (spinlock) | داخل `avc_has_perm()` |
| الـ SELinux policy tables | `policy_rwlock` (rwlock) | عند policy reload أو lookup |

**القاعدة الأساسية**:
- الـ `secid` نفسه بيتقرأ بشكل atomic لأنه `u32` على الـ architectures العادية.
- الـ policy tables اللي الـ `secid` بيشير إليها هي اللي محتاجة locking حقيقي.
- الـ AVC بيستخدم RCU عشان يخلّي الـ read path سريع بدون locking.

```
Read path (fast — no lock needed):
    secid (u32) ──→ AVC cache lookup (RCU) ──→ decision

Write path (slow — lock needed):
    policy reload ──→ policy_rwlock (write) ──→ flush AVC ──→ rebuild
```
## Phase 4: شرح الـ Functions

### ملخص الـ API

الملف `include/linux/lsm/selinux.h` هو **header-only** — مفيش functions خالص. محتواه الوحيد هو تعريف struct واحد بس:

| العنصر | النوع | الغرض |
|---|---|---|
| `lsm_prop_selinux` | `struct` | يحمل الـ SELinux security identifier (`secid`) داخل الـ LSM property system |
| `secid` | `u32` | الـ security context identifier الخاص بـ SELinux — موجود بس لو `CONFIG_SECURITY_SELINUX` معمول |

---

### التصنيف: Data Structure Definition

الملف ده جزء من منظومة **LSM (Linux Security Module)** — تحديداً هو الـ SELinux-specific slot جوه الـ `lsm_prop` union اللي بتستخدمها kernel subsystems عشان تبعت security context بين الأجزاء المختلفة (networking, IPC, audit, إلخ).

---

### الـ Struct الوحيد: `lsm_prop_selinux`

```c
struct lsm_prop_selinux {
#ifdef CONFIG_SECURITY_SELINUX
    u32 secid;  /* SELinux security context ID — opaque 32-bit label */
#endif
};
```

#### ما هو الـ `secid`؟

**الـ `secid`** هو رقم `u32` بيمثل security context بشكل compact داخل SELinux. بدل ما تنقل string زي `"system_u:system_r:kernel_t:s0"` في كل مكان، SELinux بيحول الـ context لـ integer صغير عشان:

- الـ performance: مقارنة integer أسرع بكتير من مقارنة string.
- الـ storage: 4 bytes بس في كل `task_struct`، `inode`، `sk_buff`، إلخ.
- الـ atomicity: تحديث `u32` atomic على كل الـ architectures المعروفة.

الـ `secid` نفسه هو index داخل الـ SELinux **SID table** (Security Identifier table) اللي موجودة في `security/selinux/ss/sidtab.c`.

#### الـ `CONFIG_SECURITY_SELINUX` Guard

الـ `#ifdef` ده مهم جداً — لو SELinux مش مفعّل في الـ kernel config، الـ struct بتبقى فاضية تماماً (zero-size في C هي technically undefined behavior، لكن GCC بيعملها zero-byte). ده بيضمن إن الكود اللي بيستخدم `lsm_prop_selinux` يشتغل بدون تعديل سواء كان SELinux موجود أو لا.

---

### السياق الأكبر: الـ LSM Property System

الـ `lsm_prop_selinux` بيتستخدم كجزء من struct أكبر اسمه `lsm_prop` (أو `lsm_context` في بعض الـ kernel versions) اللي بيكون union أو struct بيجمع الـ security data من كل الـ LSMs المفعّلة:

```c
/* مثال مبسط لكيفية استخدام lsm_prop_selinux في السياق الأكبر */
struct lsm_prop {
    struct lsm_prop_selinux selinux;   /* SELinux secid slot */
    /* slots for other LSMs: smack, apparmor, etc. */
};
```

الـ subsystems اللي بتستخدم `secid` بشكل مباشر أو غير مباشر:

| الـ Subsystem | الاستخدام |
|---|---|
| **Networking** (`net/netlabel/`) | label الـ network packets بالـ secid |
| **Audit** (`kernel/audit.c`) | تسجيل الـ secid في audit records |
| **IPC** (`ipc/`) | حماية الـ message queues والـ shared memory |
| **NFS** (`fs/nfs/`) | نقل الـ security context مع الـ file operations |
| **Perf** (`kernel/events/`) | التحقق من صلاحيات الـ perf events |

---

### ملاحظات للـ Kernel Developer

1. **الـ secid مش stable عبر reboots** — كل boot بيتعمله re-assignment من الـ policy loader. ماتحفظوش على disk.

2. **التحويل بين secid وstring** بيتم عبر:
   - `security_secid_to_secctx()` — من secid لـ human-readable string
   - `security_secctx_to_secid()` — العكس

3. **الـ `u32` size** كافي لأن SELinux بيستخدم 32-bit SIDs داخلياً — المساحة الكلية `2^32` SID أكبر من أي policy حقيقية.

4. الملف ده **لا يعرّف أي function** — وظيفته فقط تعريف الـ data shape اللي بيتشارك فيها SELinux مع باقي الـ kernel عبر الـ LSM interface.
## Phase 5: دليل الـ Debugging الشامل

الـ `selinux.h` تحت `include/linux/lsm/` بتعرّف واجهة بسيطة جداً — struct واحد (`lsm_prop_selinux`) بيحتوي على قيمة واحدة هي الـ **secid** (نوعها `u32`). لكن الـ secid ده بيتعامل معاه من كل مكان في الـ kernel — من الـ networking للـ IPC للـ filesystem. الـ debugging هنا بيكون على مستوى الـ SELinux policy والـ audit وتتبع الـ secid عبر الـ subsystems.

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ SELinux بيعرض معلوماته في `/sys/fs/selinux/` (مش `/sys/kernel/debug/` التقليدي لأنه بيستخدم `selinuxfs` خاص بيه).

```bash
# اقرأ حالة الـ SELinux (enforcing=1, permissive=0, disabled)
cat /sys/fs/selinux/enforce

# اقرأ الـ policy version المحملة
cat /sys/fs/selinux/policyvers

# شوف الـ MLS/MCS support
cat /sys/fs/selinux/mls

# قائمة الـ booleans المتاحة وقيمتها
ls /sys/fs/selinux/booleans/
cat /sys/fs/selinux/booleans/httpd_can_network_connect

# ترجمة secid → security context (عن طريق الـ kernel API مش مباشرة)
# استخدم seinfo أو sesearch من الـ userspace
```

**الـ selinuxfs** بيتماونت عادةً على `/sys/fs/selinux` ومن خلاله تقدر:

| المسار | المحتوى | الاستخدام |
|--------|---------|-----------|
| `/sys/fs/selinux/enforce` | 0 أو 1 | تحقق من وضع الـ enforcement |
| `/sys/fs/selinux/policyvers` | رقم الـ version | تتأكد إن الـ policy متوافقة |
| `/sys/fs/selinux/status` | struct بحالة النظام | يقدر userspace يقراه بـ mmap |
| `/sys/fs/selinux/booleans/*` | قيم الـ booleans | تغيير سلوك الـ policy ديناميكياً |
| `/sys/fs/selinux/access` | واجهة الـ access check | اختبار صلاحية قبل تطبيقها |
| `/sys/fs/selinux/context` | تحقق من صحة context | validate context string |
| `/sys/fs/selinux/deny_unknown` | سلوك الـ unknown classes | debugging policy gaps |

```bash
# ماونت الـ selinuxfs لو مش متماونت
mount -t selinuxfs selinuxfs /sys/fs/selinux

# اقرأ الـ status struct (بيتحدث atomically)
xxd /sys/fs/selinux/status
```

---

#### 2. مدخلات الـ sysfs

```bash
# الـ SELinux parameters في /proc
cat /proc/sys/fs/suid_dumpable        # بيتأثر بـ SELinux
cat /proc/$PID/attr/current           # الـ security context بتاع الـ process
cat /proc/$PID/attr/exec              # context بعد execve
cat /proc/$PID/attr/fscreate          # context للـ files الجديدة
cat /proc/$PID/attr/keycreate         # context للـ keys الجديدة
cat /proc/$PID/attr/sockcreate        # context للـ sockets الجديدة

# شوف الـ context بتاع ملف
ls -Z /path/to/file
ps -eZ | grep httpd    # context الـ processes

# الـ netlabel (لو مستخدم)
cat /sys/fs/selinux/netlabel/status
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

الـ SELinux بيوفر tracepoints مهمة:

```bash
# شوف كل الـ events المتعلقة بـ SELinux
ls /sys/kernel/debug/tracing/events/selinux/

# فعّل كل أحداث الـ SELinux
echo 1 > /sys/kernel/debug/tracing/events/selinux/enable

# أو اختار حدث معين مثلاً الـ access vector cache
echo 1 > /sys/kernel/debug/tracing/events/selinux/selinux_audited/enable

# فعّل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اقرأ النتائج
cat /sys/kernel/debug/tracing/trace

# تتبع دالة معينة في SELinux
echo 'security_compute_av' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# تتبع الـ secid lookup
echo 'selinux_inode_getsecid' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'selinux_task_getsecid' >> /sys/kernel/debug/tracing/set_ftrace_filter
```

```bash
# مثال على output الـ ftrace
# <...>-1234  [001] ....  123.456789: security_compute_av <-selinux_inode_permission
# <...>-1234  [001] ....  123.456790: selinux_inode_getsecid <-security_inode_getsecid
```

**تفسير الـ output**: العمود الأول اسم الـ process والـ PID، التاني الـ CPU، الرابع الـ timestamp، الخامس اسم الدالة والسهم بيوضح من فين اتنادت.

---

#### 4. الـ printk والـ Dynamic Debug

```bash
# فعّل الـ dynamic debug للـ SELinux
echo 'module selinux +p' > /sys/kernel/debug/dynamic_debug/control

# أو بشكل أدق — فعّل في ملف معين
echo 'file security/selinux/hooks.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file security/selinux/avc.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file security/selinux/ss/services.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف إيه الـ dynamic debug entries المتاحة للـ SELinux
grep selinux /sys/kernel/debug/dynamic_debug/control

# رفع الـ loglevel عشان تشوف كل الرسائل
dmesg -n 8
# أو
echo 8 > /proc/sys/kernel/printk

# متابعة الـ kernel messages live
dmesg -w | grep -i selinux
journalctl -k -f | grep -i selinux
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

```bash
# شوف الـ config الحالي
zcat /proc/config.gz | grep -i selinux
# أو
grep -i selinux /boot/config-$(uname -r)
```

| الـ Config Option | الوظيفة | متى تفعّله |
|------------------|---------|-----------|
| `CONFIG_SECURITY_SELINUX` | تفعيل SELinux أصلاً | دايماً في الـ debugging |
| `CONFIG_SECURITY_SELINUX_DEVELOP` | وضع الـ permissive التلقائي عند الـ boot | أثناء تطوير الـ policy |
| `CONFIG_SECURITY_SELINUX_AVC_STATS` | إحصائيات الـ AVC cache | تحليل الـ performance |
| `CONFIG_SECURITY_SELINUX_CHECKREQPROT` | تحقق من الـ protection المطلوبة | debugging الـ mmap |
| `CONFIG_SECURITY_SELINUX_SIDTAB_HASH_BITS` | حجم الـ hash table للـ SIDs | tuning الـ performance |
| `CONFIG_AUDIT` | تفعيل الـ audit framework | لازم لأي debugging |
| `CONFIG_AUDIT_TREE` | الـ audit watches | تتبع الوصول للملفات |
| `CONFIG_NETLABEL` | الـ NetLabel support | debugging الـ network labels |
| `CONFIG_SECURITY_NETWORK_XFRM` | الـ IPsec + SELinux | debugging الـ network policies |

```bash
# لو بتبني kernel جديد، ضيف في .config
CONFIG_SECURITY_SELINUX=y
CONFIG_SECURITY_SELINUX_DEVELOP=y
CONFIG_SECURITY_SELINUX_AVC_STATS=y
CONFIG_AUDIT=y
CONFIG_AUDIT_TREE=y
```

---

#### 6. أدوات الـ SELinux الخاصة

```bash
# ============ أدوات الـ userspace الأساسية ============

# شوف الـ AVC denials في الـ audit log
ausearch -m AVC -ts recent
ausearch -m AVC,USER_AVC -ts today | audit2why

# حوّل الـ denials لـ policy rules
ausearch -m AVC -ts recent | audit2allow -m mymodule

# compileوحمّل الـ policy module الجديد
checkmodule -M -m -o mymodule.mod mymodule.te
semodule_package -o mymodule.pp -m mymodule.mod
semodule -i mymodule.pp

# تحقق من الـ policy الحالية
seinfo -t | grep httpd          # كل الـ types المتعلقة بـ httpd
sesearch --allow -s httpd_t     # كل الـ allow rules لـ httpd_t
sesearch --allow -t httpd_sys_content_t  # مين يوصل للـ content ده

# ============ تتبع الـ secid مباشرة ============

# اقرأ الـ SID بتاع process
cat /proc/$PID/attr/current
# مثال output: system_u:system_r:httpd_t:s0

# اقرأ الـ SID بتاع socket
ss -Z | grep httpd

# ============ الـ AVC Cache Statistics ============
cat /sys/fs/selinux/avc/cache_stats
# output:
# lookups hits misses allocations reclaims frees
# 12453   12100  353    353         100       253

# تحليل: hits/lookups ratio مفروض > 90%، لو أقل راجع الـ policy

# ============ أدوات الـ netlabel ============
netlabelctl map list       # قائمة الـ netlabel mappings
netlabelctl unlbl list     # الـ unlabeled traffic
netlabelctl cipso list     # الـ CIPSO configurations

# ============ أدوات الـ audit ============
auditctl -l                # شوف الـ audit rules
auditctl -w /etc/passwd -p wa -k passwd_changes  # watch ملف
ausearch -k passwd_changes  # ابحث عن الـ events
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|--------------------|--------|------|
| `avc: denied { read } for pid=... scontext=... tcontext=... tclass=file` | الـ SELinux منع عملية قراءة | حلل بـ `audit2why` وأضف rule أو صلح الـ context |
| `avc: denied { write } for ... tclass=dir` | منع الكتابة في directory | تحقق من الـ context بـ `ls -Z` وصلحه بـ `restorecon` |
| `SELinux: initialized (dev ..., type ...)` | filesystem تم تهيئته بـ SELinux | معلوماتي — مش error |
| `SELinux: unrecognized netlink message type` | netlink message بـ type غير معروف | راجع الـ policy وأضف `netlink_*` rules |
| `SELinux: policy capability booleans not supported` | الـ kernel بيدعم booleans أقل من الـ policy | طابق version الـ kernel مع الـ policy |
| `SELinux: Context ... is not valid` | context string غير صالح | تحقق من الـ policy بـ `seinfo` و `chcon` |
| `avc: denied { name_bind } for ... tclass=tcp_socket` | منع الـ bind على port | أضف `port_type` للـ policy أو استخدم `semanage port` |
| `avc: denied { connectto } for ... tclass=unix_stream_socket` | منع الاتصال بـ unix socket | راجع الـ `domain_auto_trans` rules |
| `SELinux: Could not load policy` | فشل تحميل الـ policy عند الـ boot | تحقق من `/etc/selinux/config` وسلامة الـ policy files |
| `LSM: Unable to initialize SELinux` | فشل تهيئة الـ SELinux في الـ boot | تحقق من الـ kernel config و boot parameters |
| `SELinux: mls_level_isvalid: out of range level` | MLS level خارج النطاق المسموح | راجع الـ `seinfo --sensitivity` وتأكد من الـ clearance |
| `avc: denied { sigchld } for ... tclass=process` | منع الـ signal بين processes | أضف `signal` permissions في الـ policy |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

المواضع الأهم في كود الـ SELinux للـ debugging:

```c
/* في security/selinux/avc.c — تتبع الـ AVC misses */
static void avc_audit_pre_callback(struct audit_buffer *ab, void *a)
{
    struct common_audit_data *ad = a;
    /* هنا تقدر تضيف WARN_ON لو الـ secid = 0 (secid غير مهيأ) */
    WARN_ON(ad->selinux_audit_data->ssid == 0);
}

/* في security/selinux/hooks.c — لما يفشل الـ inode relabeling */
static int selinux_inode_setxattr(...)
{
    /* نقطة مهمة: لما الـ new_sid = SECSID_NULL */
    if (unlikely(new_sid == SECSID_NULL)) {
        WARN_ONCE(1, "SELinux: invalid sid from setxattr\n");
        dump_stack();
    }
}

/* في security/selinux/ss/services.c — تتبع الـ policy reload */
static void selinux_policy_commit(struct selinux_policy *newpolicy)
{
    /* نقطة حرجة — بعد الـ commit مباشرة */
    pr_debug("SELinux: policy reloaded, version=%u\n",
             newpolicy->policydb.policyvers);
}
```

```bash
# تفعيل الـ WARN_ON outputs في الـ log
dmesg | grep -i "WARNING:"
dmesg | grep -i "Call Trace"

# لو عايز تعمل panic على أي WARN (للـ CI/testing)
echo 1 > /proc/sys/kernel/panic_on_warn
```

---

### Hardware Level — Policy & Audit Level Debugging

> الـ SELinux module برمجي بالكامل، مفيش hardware مباشر ليه. لكن الـ "hardware level" هنا بيعني التحقق من أن حالة الـ policy والـ audit تعكس الـ state الفعلي للنظام.

#### 1. التحقق من أن حالة الـ Policy تطابق الـ System State

```bash
# ============ تحقق من الـ enforcing state ============
# الـ kernel state
cat /sys/fs/selinux/enforce

# الـ userspace state (من /etc/selinux/config)
cat /etc/selinux/config | grep SELINUX=

# المفروض يتطابقوا — لو مختلفين في بعض الـ scenarios، في مشكلة
# مقارنة الـ policy المحملة مع الـ policy على الـ disk
semodule -l | sort > /tmp/loaded_modules.txt
ls /etc/selinux/targeted/modules/active/modules/ | sort > /tmp/disk_modules.txt
diff /tmp/loaded_modules.txt /tmp/disk_modules.txt

# ============ تحقق من الـ file contexts ============
# قارن الـ context الفعلي مع المتوقع من الـ policy
matchpathcon /var/www/html/index.html    # context المتوقع
ls -Z /var/www/html/index.html           # context الفعلي

# لو مختلفين:
restorecon -v /var/www/html/index.html   # صلح الـ context

# تحقق من كل الـ filesystem
restorecon -Rnv /var/www/   # -n = dry run, -v = verbose
```

#### 2. تقنيات الـ "Register Dump" — الـ Policy State Dump

في SELinux مفيش register dump بالمعنى الحرفي، لكن في ما يعادله:

```bash
# ============ dump الـ policy الكاملة ============
# استخرج الـ binary policy من الـ kernel
cp /sys/fs/selinux/policy /tmp/current_policy.bin

# حلّل الـ binary policy
seinfo /tmp/current_policy.bin --stats

# قارن الـ policy المحملة مع الـ expected policy
seinfo /etc/selinux/targeted/policy/policy.33 --stats

# ============ dump الـ AVC cache ============
# الإحصائيات
cat /sys/fs/selinux/avc/cache_stats

# ============ dump الـ SID table ============
# مش متاح مباشرة لكن عبر الـ audit
ausearch -m AVC | awk '{print $NF}' | sort -u | head -20

# ============ dump الـ netlink state ============
cat /proc/net/netlink  # شوف الـ SELinux netlink sockets
```

#### 3. أدوات الـ Audit بدل Logic Analyzer

الـ audit subsystem هو "logic analyzer" الـ SELinux:

```bash
# ============ setup audit rules شاملة ============
# راقب كل AVC events
auditctl -a always,exit -F arch=b64 -S all -k selinux_all

# راقب تغييرات الـ policy
auditctl -w /etc/selinux/ -p wa -k selinux_policy
auditctl -w /sys/fs/selinux/enforce -p wa -k selinux_enforce

# ============ تحليل الـ audit trail ============
# شوف كل الـ AVC denials في الـ 24 ساعة الأخيرة
ausearch -m AVC -ts today --format text

# رسّم نمط الـ denials (أكتر object class بتتحجب)
ausearch -m AVC -ts today | \
  grep -o 'tclass=[^ ]*' | sort | uniq -c | sort -rn

# ============ الـ audit2why التفصيلي ============
ausearch -m AVC -ts recent | audit2why --all
# بيشرح ليه اتحجبت وإيه الـ policy rule الناقصة
```

#### 4. مشاكل شائعة وأنماط الـ Kernel Log

| المشكلة | النمط في الـ Log | التشخيص |
|---------|----------------|---------|
| خدمة مش بتشتغل بعد الـ update | `avc: denied { execute }` لـ context قديم | `restorecon -Rv /usr/sbin/` |
| web server مش بيوصل للـ network | `avc: denied { name_connect } ... tclass=tcp_socket` | `setsebool -P httpd_can_network_connect 1` |
| container runtime بيفشل | `avc: denied { sys_admin }` لـ container domain | راجع الـ `container_selinux` policy |
| policy reload بيبطّئ النظام | كتير من `security_compute_av` في الـ ftrace | راجع حجم الـ AVC cache و `CONFIG_SECURITY_SELINUX_SIDTAB_HASH_BITS` |
| /proc files مش بتتقرأ | `avc: denied { read } ... tclass=proc_type` | أضف `proc_type` في الـ policy |
| systemd services بتفشل بعد selinux enable | `avc: denied { dyntransition }` | راجع الـ `type_transition` rules |

#### 5. تحقق من الـ Device Tree (المعادل في SELinux: الـ File Contexts)

```bash
# الـ file_contexts بمثابة "الـ Device Tree" في SELinux
# بتحدد إيه الـ label المفروض على كل path

# شوف الـ file_contexts المحملة
cat /etc/selinux/targeted/contexts/files/file_contexts | head -30

# اختبر matching
matchpathcon /var/log/httpd/access.log

# شوف الـ process contexts المتوقعة
cat /etc/selinux/targeted/contexts/initrc_context
cat /etc/selinux/targeted/contexts/default_context

# تحقق من الـ port contexts
semanage port -l | grep http

# لو في mismatch بين الـ runtime context والـ policy:
semanage fcontext -l | grep "/var/www"   # contexts في الـ policy
ls -Z /var/www                            # contexts الفعلية
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**سيناريو 1: خدمة بتتحجب من SELinux**

```bash
# الخطوة 1: اعرف إيه اللي بيتحجب
ausearch -m AVC -ts recent -c httpd | audit2why
# output مثال:
# type=AVC msg=audit(1234567890.123:456): avc: denied { name_connect }
#   for pid=1234 comm="httpd" dest=3306 scontext=system_u:system_r:httpd_t:s0
#   tcontext=system_u:object_r:mysqld_port_t:s0 tclass=tcp_socket
#
# Was caused by:
#   The boolean httpd_can_network_connect_db was set incorrectly.
#
# Fix:
#   setsebool -P httpd_can_network_connect_db 1

# الخطوة 2: طبّق الـ fix
setsebool -P httpd_can_network_connect_db 1

# الخطوة 3: تحقق
getsebool httpd_can_network_connect_db
# output: httpd_can_network_connect_db --> on
```

**سيناريو 2: ملفات بـ context غلط**

```bash
# اكتشاف
ls -Z /var/www/html/
# system_u:object_r:user_home_t:s0 index.html  ← غلط! المفروض httpd_sys_content_t

# تشخيص
matchpathcon /var/www/html/index.html
# /var/www/html/index.html  system_u:object_r:httpd_sys_content_t:s0

# إصلاح
restorecon -v /var/www/html/index.html
# Relabeled /var/www/html/index.html from user_home_t to httpd_sys_content_t

# تحقق
ls -Z /var/www/html/index.html
# system_u:object_r:httpd_sys_content_t:s0 index.html  ✓
```

**سيناريو 3: تتبع مصدر الـ secid**

```bash
# شوف الـ secid بتاع process معين عن طريق /proc
PID=$(pgrep httpd | head -1)
cat /proc/$PID/attr/current
# system_u:system_r:httpd_t:s0

# حوّل الـ context ده لـ secid (بستخدم الـ kernel API عبر userspace tool)
python3 -c "
import ctypes
lib = ctypes.CDLL('libselinux.so.1')
lib.getpidcon.restype = ctypes.c_int
context = ctypes.c_char_p()
lib.getpidcon($PID, ctypes.byref(context))
print('Context:', context.value.decode())
"

# أو أسهل:
ps -eZ | grep httpd
# system_u:system_r:httpd_t:s0 1234 ? 00:00:01 httpd
```

**سيناريو 4: تحليل الـ AVC cache performance**

```bash
# اقرأ الإحصائيات
cat /sys/fs/selinux/avc/cache_stats
# lookups hits misses allocations reclaims frees
# 1000000  950000  50000  50000      20000    30000

# احسب الـ hit rate
awk 'NR==2 {printf "Hit rate: %.1f%%\n", ($2/$1)*100}' \
  /sys/fs/selinux/avc/cache_stats
# Hit rate: 95.0%

# لو الـ hit rate < 90%، زود حجم الـ cache
# في الـ kernel config:
# CONFIG_SECURITY_SELINUX_AVC_STATS=y
# وحجم الـ hash: Security options → SELinux AVC hash size
```

**سيناريو 5: debugging الـ secid = 0 (uninitialized)**

```bash
# فعّل ftrace لتتبع متى بيتعمل lookup بـ secid = 0
echo 'selinux_secid_to_secctx' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل العملية المشبوهة
systemctl start suspicious-service

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace | grep selinux_secid_to_secctx

# نظّف
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

**سيناريو 6: تفعيل الـ permissive mode مؤقتاً للـ debugging**

```bash
# اجعل النظام permissive مؤقتاً (بدون restart)
setenforce 0
cat /sys/fs/selinux/enforce  # يطبع 0

# أو اجعل domain معين permissive بس (أأمن)
semanage permissive -a httpd_t

# شغّل العملية وجمّع كل الـ denials
ausearch -m AVC -ts recent | audit2allow -M httpd_custom

# طبّق الـ rules الجديدة
semodule -i httpd_custom.pp

# رجّع الـ enforcing
setenforce 1
# أو شيل الـ permissive domain
semanage permissive -d httpd_t
```

**سيناريو 7: تدقيق شامل لـ selinuxfs state**

```bash
#!/bin/bash
# سكريبت audit شامل للـ SELinux state

echo "=== SELinux Status ==="
sestatus

echo -e "\n=== Enforce Mode ==="
cat /sys/fs/selinux/enforce

echo -e "\n=== Policy Version ==="
cat /sys/fs/selinux/policyvers

echo -e "\n=== AVC Cache Stats ==="
cat /sys/fs/selinux/avc/cache_stats

echo -e "\n=== Recent AVC Denials (last 10) ==="
ausearch -m AVC -ts recent 2>/dev/null | tail -20

echo -e "\n=== Loaded Policy Modules ==="
semodule -l | wc -l
echo "modules loaded"

echo -e "\n=== Permissive Domains ==="
semanage permissive -l 2>/dev/null || echo "none"

echo -e "\n=== Boolean Changes from Default ==="
semanage boolean -l | grep "on   on\|off  off" -v | head -20
```
## Phase 6: سيناريوهات من الحياة العملية

الـ file المحوري هنا هو `include/linux/lsm/selinux.h` — بيعرّف struct واحدة بس:

```c
struct lsm_prop_selinux {
#ifdef CONFIG_SECURITY_SELINUX
    u32 secid; /* SELinux security identifier — رقم بيمثل security context */
#endif
};
```

الـ `secid` ده هو المحور اللي بتتعامل معاه كل السيناريوهات دي — رقم `u32` بيعمل map لـ security context زي `system_u:system_r:kernel_t:s0` جوه الـ SELinux policy database.

---

### السيناريو الأول: Industrial Gateway على AM62x — UART blocked بسبب secid خاطئ

#### العنوان
**الـ UART console مش شغّال بعد تفعيل SELinux على gateway صناعي**

#### السياق
شركة بتبني industrial gateway على **TI AM62x** بـ Linux 6.x. الـ product بيجمع بيانات من sensors عبر UART وبيبعتها على cloud. قرروا يفعّلوا SELinux لمتطلبات IEC 62443 (أمان الأنظمة الصناعية). بعد التفعيل، الـ UART daemon مش بيقدر يكتب على `/dev/ttyS0`.

#### المشكلة
الـ daemon بيطلع في `/var/log/audit/audit.log`:

```
type=AVC msg=audit(1700000001.123:42): avc: denied { write } for
pid=312 comm="sensor-daemon" name="ttyS0" dev="tmpfs" ino=88
scontext=system_u:system_r:sensor_daemon_t:s0
tcontext=system_u:object_r:tty_device_t:s0
tclass=chr_file permissive=0
```

#### التحليل
الـ kernel لما بيعمل access check على `/dev/ttyS0`، الـ LSM hook بيشتغل وبيجيب الـ `secid` الخاص بالـ process من struct `lsm_prop_selinux`:

```c
struct lsm_prop_selinux {
#ifdef CONFIG_SECURITY_SELINUX
    u32 secid; /* هنا بيتخزن secid الخاص بـ sensor_daemon_t */
#endif
};
```

الـ SELinux engine بياخد الـ `secid` ده وبيعمل lookup في الـ AVC (Access Vector Cache) — بيلاقي إن مفيش rule بتسمح لـ `sensor_daemon_t` بالـ `write` على `tty_device_t`.

المشكلة مش في الـ struct نفسها — المشكلة إن الـ policy مش بتغطي الـ UART permission.

#### الحل

**خطوة 1: تشغيل permissive مؤقت للتشخيص**

```bash
# على الـ target
setenforce 0
# لو الـ daemon اشتغل → المشكلة في الـ policy مش في hardware
```

**خطوة 2: استخدام audit2allow لتوليد الـ policy**

```bash
grep "sensor_daemon" /var/log/audit/audit.log | audit2allow -M sensor_uart
semodule -i sensor_uart.pp
```

**خطوة 3: policy rule صح**

```
# في ملف sensor_daemon.te
allow sensor_daemon_t tty_device_t:chr_file { read write open ioctl };
allow sensor_daemon_t serial_device_t:chr_file { read write open ioctl };
```

**خطوة 4: verify الـ secid بعد تطبيق الـ policy**

```bash
# شوف الـ context الصح
ps -Z | grep sensor-daemon
# المفروض تلاقي: system_u:system_r:sensor_daemon_t:s0

# verify الـ secid عبر seinfo
seinfo -t sensor_daemon_t -x
```

#### الدرس المستفاد
الـ `secid` في `lsm_prop_selinux` هو مجرد رقم — معناه الحقيقي بييجي من الـ policy. لازم تكتب الـ policy قبل ما تطلّع SELinux enforcing في production. على الـ AM62x industrial boards، افتح الـ UART permissions من أول يوم في الـ bring-up.

---

### السيناريو التاني: Android TV Box على Allwinner H616 — secid mismatch بعد OTA update

#### العنوان
**الـ media service مش بيقدر يفتح `/dev/video0` بعد OTA**

#### السياق
**Android TV box** على **Allwinner H616** بـ Linux-based Android. بعد OTA update، الـ media streaming service بيتوقف فجأة. الـ product بيستخدم SELinux بالـ Android policy. الـ update غيّرت الـ binary بس ما غيّرتش الـ SELinux policy.

#### المشكلة
الـ binary الجديد اتبنى من context جديد بـ file context مختلف، فالـ `secid` المحمّل في `lsm_prop_selinux` للـ process بقى مختلف عن اللي الـ policy بتتوقعه:

```
avc: denied { read } for pid=1205 comm="mediaserver"
scontext=u:r:mediaserver_updated_t:s0
tcontext=u:object_r:video_device_t:s0
tclass=chr_file
```

الـ OTA حملّت binary جديد بـ label `mediaserver_updated_t` بدل `mediaserver_t`.

#### التحليل

لما الـ kernel بيعمل `exec` للـ mediaserver، الـ SELinux بيحدد الـ `secid` الجديد للـ process بناءً على:

1. الـ `secid` الخاص بـ parent process
2. الـ file context (label) للـ executable على disk

```c
/* الـ struct بتتملّى لما process بتبدأ */
struct lsm_prop_selinux {
#ifdef CONFIG_SECURITY_SELINUX
    u32 secid; /* بيتحدد من file context + transition rules */
#endif
};
```

الـ OTA غيّرت الـ binary لكن ما عملتش `restorecon` على الـ file — فالـ file label اتغير وبالتالي الـ `secid` اللي بييجي من الـ type transition اتغير.

#### الحل

**خطوة 1: افحص الـ file context**

```bash
ls -Z /system/bin/mediaserver
# هتلاقي: u:object_r:mediaserver_updated_exec_t:s0  ← label غلط
```

**خطوة 2: صلّح الـ label**

```bash
restorecon /system/bin/mediaserver
ls -Z /system/bin/mediaserver
# المفروض: u:object_r:mediaserver_exec_t:s0  ← label صح
```

**خطوة 3: لو محتاج label جديد، زوّد الـ policy**

```
# في file_contexts
/system/bin/mediaserver  u:object_r:mediaserver_exec_t:s0

# في mediaserver.te
domain_auto_trans(init_t, mediaserver_exec_t, mediaserver_t);
```

**خطوة 4: في الـ OTA script نفسه، اضيف restorecon**

```bash
# في OTA post-install script
restorecon -R /system/bin/
restorecon -R /vendor/bin/
```

#### الدرس المستفاد
الـ `secid` الخاص بأي process بييجي من الـ file label للـ executable — أي OTA بتغير binary لازم تعمل `restorecon` أو تضمن إن الـ file_contexts متوافق. على الـ Allwinner H616 TV boxes، ده سبب شائع لـ media pipeline failures بعد updates.

---

### السيناريو التالت: IoT Sensor على STM32MP1 — secid=0 بيعمل kernel panic

#### العنوان
**الـ kernel يعمل panic بسبب invalid secid في LSM hook**

#### السياق
فريق بيعمل bring-up لـ **IoT sensor node** على **STM32MP1** (Cortex-A7). البورد بتشغّل Linux بـ minimal rootfs. المطوّر فعّل `CONFIG_SECURITY_SELINUX` في الـ kernel config لكن منسيش يحط الـ `selinux=1` في kernel cmdline ومنبنيش الـ policy image.

#### المشكلة
الـ kernel بيعمل boot وبعدين يـ panic في early userspace لما أي process بتحاول تعمل socket:

```
BUG: kernel NULL pointer dereference at virtual address 00000000
PC is at selinux_socket_create+0x2c/0x80
```

#### التحليل

الـ kernel compile الـ SELinux code لأن `CONFIG_SECURITY_SELINUX=y`، وبالتالي الـ struct اتبنت:

```c
struct lsm_prop_selinux {
#ifdef CONFIG_SECURITY_SELINUX
    u32 secid; /* موجودة في memory — لكن قيمتها صفر لأن SELinux مش initialized */
#endif
};
```

لما الـ SELinux مش loaded (مفيش policy)، الـ `secid` بييجي `0` اللي هو invalid context. الـ SELinux hooks بتحاول تعمل dereference على pointer جاي من lookup بـ `secid=0` — بيطلع NULL.

بالتفصيل — الـ LSM بيعمل `selinux_inode_alloc_security()` → بيبدأ يتعامل مع الـ `secid` قبل ما الـ policy تتحمّل → crash.

#### الحل

**الخيار الأول: disable SELinux في الـ config لو مش محتاجه**

```bash
# في kernel config
CONFIG_SECURITY_SELINUX=n
```

**الخيار التاني: فعّل SELinux صح**

```bash
# في bootloader environment (U-Boot على STM32MP1)
setenv bootargs "console=ttySTM0,115200 root=/dev/mmcblk0p2 rw selinux=1 security=selinux"
saveenv
```

**وحط الـ policy image في الـ rootfs**

```bash
# على الـ build machine
mkdir -p /etc/selinux/targeted/policy/
cp policy.33 /etc/selinux/targeted/policy/
cp /etc/selinux/targeted/contexts/files/file_contexts .

# وفي /etc/selinux/config
SELINUX=enforcing
SELINUXTYPE=targeted
```

**الخيار التالت: permissive mode للـ bring-up**

```bash
# kernel cmdline
enforcing=0 selinux=1
```

#### الدرس المستفاد
الـ `CONFIG_SECURITY_SELINUX=y` من غير policy image = guaranteed crash. الـ `secid=0` في `lsm_prop_selinux` هو علامة إن الـ SELinux initialized لكن مفيش policy محملة. على الـ STM32MP1 IoT boards، لو مش عايز SELinux — disable من الـ config خالص، ماتفعّلوش ناقص.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — secid مش بيتنقل صح عبر IPC

#### العنوان
**الـ inter-process communication بين ECU processes بيتعطل بسبب secid propagation**

#### السياق
**Automotive ECU** على **NXP i.MX8** بيشغّل Linux مع SELinux لمتطلبات ISO 21434 (automotive cybersecurity). الـ system عنده process `can_gateway_t` بتقرأ من CAN bus وبتبعت data لـ `dashboard_t` عبر Unix domain socket. بعد ما الـ SOC upgrade للـ i.MX8M Plus، الـ IPC بدأ يفشل.

#### المشكلة

```
avc: denied { sendmsg } for pid=445 comm="can-gateway"
scontext=system_u:system_r:can_gateway_t:s0
tcontext=system_u:system_r:dashboard_t:s0
tclass=unix_dgram_socket
```

الـ rule موجودة في الـ policy لكن الـ denial بيحصل!

#### التحليل

المشكلة دي دقيقة — الـ i.MX8M Plus بيستخدم kernel version جديدة فيها **multi-LSM support**. الـ `lsm_prop_selinux` بقت جزء من struct أكبر اسمها `lsm_prop`:

```c
/* لما بييجي secid يتنقل عبر socket، الـ kernel بيعمل ده */
struct lsm_prop_selinux {
#ifdef CONFIG_SECURITY_SELINUX
    u32 secid; /* الـ secid اللي بيتبعت مع الـ message */
#endif
};
```

الـ kernel الجديد بيبعت الـ `secid` مع كل message على الـ Unix socket — لكن المشكلة إن الـ socket options اتغيرت. الـ `SO_PEERSEC` بقى بيرجع string بدل ما الـ kernel يعمل comparison داخلي صح.

بالتحقيق اكتشفنا إن الـ `can_gateway` process بتستخدم `SCM_CREDENTIALS` بس مش بتبعت الـ security context — فالـ receiver بيشوف secid مختلف.

#### الحل

**خطوة 1: تحقق من الـ secid اللي بيتنقل**

```bash
# على i.MX8 target
cat /proc/445/attr/current
# system_u:system_r:can_gateway_t:s0

# افحص الـ peer context على الـ socket
cat /proc/446/attr/sockcreate
```

**خطوة 2: في الـ application code، ابعت الـ security context صح**

```c
/* في can_gateway.c — لازم يبعت الـ SCM_SECURITY */
struct msghdr msg = {0};
struct cmsghdr *cmsg;
char buf[256];
struct iovec iov = { .iov_base = data, .iov_len = len };

msg.msg_iov = &iov;
msg.msg_iovlen = 1;
msg.msg_control = buf;
msg.msg_controllen = sizeof(buf);

cmsg = CMSG_FIRSTHDR(&msg);
cmsg->cmsg_level = SOL_SOCKET;
cmsg->cmsg_type = SCM_SECURITY;
/* kernel هيحط الـ secid تلقائياً من lsm_prop_selinux */
```

**خطوة 3: الـ policy rule الصح**

```
# في can_gateway.te
allow can_gateway_t dashboard_t:unix_dgram_socket sendmsg;
allow can_gateway_t dashboard_t:unix_dgram_socket { sendmsg write };
```

**خطوة 4: verify بـ runcon**

```bash
runcon system_u:system_r:can_gateway_t:s0 can-gateway-test
```

#### الدرس المستفاد
على الـ i.MX8 automotive systems، الـ `secid` في `lsm_prop_selinux` بيتنقل مع IPC messages تلقائياً — لكن لازم الـ application يكون مبني على kernel API الصح. لو الـ policy rule موجودة والـ denial لسه بيحصل، اشتبه في الـ secid propagation مش في الـ policy نفسها.

---

### السيناريو الخامس: Custom Board Bring-up على RK3562 — debugging secid في kernel module

#### العنوان
**kernel module مش بيقدر يعمل network socket بسبب secid غير معروف**

#### السياق
فريق bring-up لـ **custom industrial board** على **Rockchip RK3562**. كتبوا kernel module بيعمل raw socket لـ custom industrial protocol. لما فعّلوا SELinux، الـ module بدأ يفشل في `sock_create_kern()`.

#### المشكلة

```
[ 45.123456] SELinux: avc: denied { create } for
comm="kthread" scontext=system_u:system_r:kernel_t:s0
tcontext=system_u:system_r:kernel_t:s0 tclass=rawip_socket
permissive=0
```

غريب — الـ `scontext` والـ `tcontext` نفس الـ `kernel_t` لكن لسه denied!

#### التحليل

الـ kernel module بيشتغل في kernel space — الـ `secid` اللي بيتعامل معاه من `lsm_prop_selinux` هو الـ `secid` الخاص بـ `kernel_t`:

```c
struct lsm_prop_selinux {
#ifdef CONFIG_SECURITY_SELINUX
    u32 secid; /* للـ kernel threads: secid الخاص بـ kernel_t context */
#endif
};
```

المشكلة إن الـ default SELinux policy على Linux ما بتسمحش حتى لـ `kernel_t` بإنشاء `rawip_socket` — ده security measure مقصود. حتى الـ kernel threads محتاجة explicit permission.

بالتحقيق في الـ policy:

```bash
sesearch --allow -s kernel_t -c rawip_socket
# مفيش نتايج → مفيش allow rule
```

#### الحل

**الخيار الأول: اعمل custom SELinux module للـ kernel module**

```
# في industrial_module.te
module industrial_module 1.0;

require {
    type kernel_t;
    class rawip_socket { create bind read write };
}

# السماح لـ kernel_t بإنشاء raw IP socket
allow kernel_t self:rawip_socket { create bind read write setopt getopt };
```

```bash
# compile وload الـ module
checkmodule -M -m -o industrial_module.mod industrial_module.te
semodule_package -o industrial_module.pp -m industrial_module.mod
semodule -i industrial_module.pp
```

**الخيار التاني: استخدم نـ user-space daemon بدل kernel module**

```c
/* بدل kernel module، اعمل user-space process بـ custom_proto_t context */
/* وبالتالي الـ lsm_prop_selinux.secid بتاعه هيكون لـ custom_proto_t */
/* وتكتب الـ policy عليه بسهولة */
```

**خطوة التحقق — اعرف الـ secid الحالي للـ kernel_t**

```bash
# على RK3562 target
echo "kernel_t" | secon -t -
# أو
cat /proc/1/attr/current  # للـ init process

# verify الـ secid رقمياً لو عندك debug access
cat /sys/kernel/security/selinux/checkreqprot
```

**خطوة تفعيل الـ audit للـ debug**

```bash
# فعّل كل الـ audit messages
echo 1 > /sys/kernel/security/selinux/avc/cache_threshold
auditctl -a always,exit -F arch=b32 -S socket
```

#### الدرس المستفاد
الـ `secid` في `lsm_prop_selinux` للـ kernel threads هو `secid` الخاص بـ `kernel_t` — وده context محمي وmحتاج explicit policy rules حتى للـ operations الأساسية. على الـ RK3562 custom boards، لو كاتب kernel module بيعمل network operations، فكّر في نقل الـ logic لـ user-space daemon بـ custom SELinux type أسهل بكتير من تعديل الـ kernel_t policy.
## Phase 7: مصادر ومراجع

### مصادر الـ Kernel الرسمية

| المصدر | الرابط |
|--------|--------|
| **SELinux — Kernel Admin Guide** | [docs.kernel.org/admin-guide/LSM/SELinux.html](https://docs.kernel.org/admin-guide/LSM/SELinux.html) |
| **LSM General Security Hooks** | [docs.kernel.org/security/lsm.html](https://www.kernel.org/doc/html/latest/security/lsm.html) |
| **Linux Security Module Usage** | [docs.kernel.org/admin-guide/LSM/index.html](https://docs.kernel.org/admin-guide/LSM/index.html) |
| **LSM/SELinux secid** | [docs.kernel.org/networking/secid.html](https://docs.kernel.org/networking/secid.html) |

الملف الرئيسي اللي بنوثقه:

```
include/linux/lsm/selinux.h      ← تعريف struct lsm_prop_selinux
security/selinux/                 ← التطبيق الكامل لـ SELinux
include/linux/lsm_hooks.h         ← تعريف الـ LSM hooks
include/linux/security.h          ← الـ API العام للـ LSM
```

### مقالات LWN.net

دي أهم المقالات اللي بتشرح تطور الـ LSM و SELinux في الـ kernel:

| المقال | الأهمية |
|--------|---------|
| [LSM stacking and the future](https://lwn.net/Articles/804906/) | بيشرح إزاي SELinux هو الـ "major LSM" وجهود تشغيل أكتر من LSM في نفس الوقت |
| [A change in direction for security-module stacking?](https://lwn.net/Articles/970070/) | أحدث مقال عن stacking الـ LSMs الكبيرة زي SELinux |
| [Adding system calls for Linux security modules](https://lwn.net/Articles/919059/) | system calls جديدة للتعامل مع أكتر من security context من LSMs مختلفة |
| [The future of the Linux Security Module API](https://lwn.net/Articles/180194/) | مقال تاريخي عن نقاشات إزالة الـ LSM وأن SELinux كان المستخدم الوحيد |
| [selinux: Implement LSM notification system](https://lwn.net/Articles/720991/) | تطبيق نظام الـ notifications في SELinux عبر الـ LSM |
| [V2 Remove SELinux dependencies from linux-audit via LSM](https://lwn.net/Articles/244514/) | جهود فصل الـ SELinux عن audit باستخدام واجهة الـ LSM |
| [The return of loadable security modules?](https://lwn.net/Articles/526983/) | نظرة تاريخية على تطور الـ LSM مع SELinux كمستخدم أساسي |
| [Linux security non-modules and AppArmor](https://lwn.net/Articles/239962/) | نقاش حول الـ LSM API وأهمية SELinux فيه |

### الـ kernelnewbies.org

تغييرات SELinux في إصدارات مختلفة من الـ kernel:

| الإصدار | التغيير |
|---------|---------|
| [Linux 2.6.11](https://kernelnewbies.org/Linux_2_6_11) | من أوائل الإصدارات اللي دعمت SELinux رسمياً |
| [Linux 2.6.15](https://kernelnewbies.org/Linux_2_6_15) | دعم MLS (Multi-Level Security) في SELinux |
| [Linux 2.6.18](https://kernelnewbies.org/Linux_2_6_18) | دمج SELinux مع Netfilter لتحكم أقوى في أمان الشبكة |
| [Linux 2.6.28](https://kernelnewbies.org/Linux_2_6_28) | إضافة "dummy" policy في SELinux |
| [Linux 3.11](https://kernelnewbies.org/Linux_3.11) | دعم Labeled NFS — SELinux كامل على NFS |
| [Linux 6.4](https://kernelnewbies.org/Linux_6.4) | إزالة إمكانية تعطيل SELinux وقت التشغيل |

### الـ eLinux.org

| الصفحة | المحتوى |
|--------|---------|
| [SELinux — eLinux.org](https://elinux.org/SELinux) | صفحة رئيسية عن Embedded SELinux — الـ kernel، الـ userland، BusyBox، والـ policy |
| [Security Presentations](https://elinux.org/Security_Presentations) | عروض تقديمية عن صعوبات SELinux في الـ embedded systems |
| [Getting started with meta-selinux](https://elinux.org/images/4/43/Yps2021.11-selinux.pdf) | دليل تطبيق SELinux في Yocto |

### نقاشات الـ Mailing List

| الموضوع | الرابط |
|---------|--------|
| [PATCH: LSM — Define SELinux function to measure security state](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2238653.html) | نقاش حول قياس حالة الـ SELinux عبر الـ LSM |

للبحث في الـ mailing list archives مباشرة:

```bash
# البحث في lore.kernel.org
https://lore.kernel.org/selinux/
https://lore.kernel.org/linux-security-module/
```

### Commits مهمة في الـ Kernel

```bash
# إيجاد الـ commit اللي أضاف lsm_prop_selinux
git log --oneline --all -- include/linux/lsm/selinux.h

# تاريخ ملف الـ header
git log --follow -p include/linux/lsm/selinux.h

# البحث عن secid في الـ SELinux code
git log --oneline --all --grep="secid" -- security/selinux/
```

الـ commits الأساسية اللي كونت هذا الـ subsystem:
- إضافة الـ `struct lsm_prop_selinux` كجزء من مبادرة **LSM stacking** لعزل الـ security blob لكل LSM
- تحريك الـ `secid` من الـ shared structures لـ per-LSM structures ضمن تنظيم `lsm_prop`

### كتب مُوصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 14**: الـ Linux Device Model — بيساعد على فهم بنية الـ kernel بشكل عام
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Devices and Modules
- بيشرح الـ kernel architecture اللي بتعتمد عليها الـ security subsystems
- ضروري لفهم إزاي الـ `#ifdef CONFIG_*` بيشتغل في الـ kernel

#### Understanding the Linux Kernel — Bovet & Cesati
- الفصول اللي بتتكلم عن الـ VFS وإدارة الـ processes — ضرورية لفهم نقاط تطبيق SELinux hooks

#### SELinux by Example — Mayer, MacMillan & Caplan
- الكتاب الأشمل عن SELinux من الـ policy لحد الـ kernel implementation
- بيشرح مفهوم الـ `secid` والـ security context بالتفصيل

### مصطلحات للبحث

```
# للبحث في Google أو DuckDuckGo
"lsm_prop_selinux" kernel
"secid" SELinux LSM kernel
"struct lsm_prop" linux kernel
SELinux "security blob" LSM stacking
SELinux secid networking xfrm kernel
linux kernel "lsm/selinux.h"
CONFIG_SECURITY_SELINUX kernel hook
```

```
# للبحث في lore.kernel.org
subject:"selinux" subject:"lsm_prop"
subject:"secid" list:selinux
```

### مراجع إضافية

| المصدر | الرابط |
|--------|--------|
| **SELinux Project الرسمي** | [github.com/SELinuxProject](https://github.com/SELinuxProject) |
| **Linux Security Modules — Wikipedia** | [en.wikipedia.org/wiki/Linux_Security_Modules](https://en.wikipedia.org/wiki/Linux_Security_Modules) |
| **Kernel docs: LSM hooks** | [kernel.org/doc/html/latest/security/lsm.html](https://www.kernel.org/doc/html/latest/security/lsm.html) |
| **Star Lab: A Brief Tour of LSMs** | [starlab.io/blog/a-brief-tour-of-linux-security-modules](https://www.starlab.io/blog/a-brief-tour-of-linux-security-modules/) |
| **Cloudflare: eBPF LSM** | [blog.cloudflare.com/live-patch-security-vulnerabilities-with-ebpf-lsm](https://blog.cloudflare.com/live-patch-security-vulnerabilities-with-ebpf-lsm/) |
## Phase 8: Writing simple module

### الفكرة

الـ `selinux.h` بتعرّف `struct lsm_prop_selinux` اللي فيها حقل واحد بس: `u32 secid`. الـ **secid** ده رقم داخلي بيمثّل **security context** في SELinux — كل process أو file ليه secid بيتترجم لـ label زي `system_u:system_r:kernel_t`. الـ function الأنسب نعمل عليها kprobe هي `security_secid_to_secctx` — دي بتتاخد secid وبترجعله string context، وبتتاخد في كل مرة حاجة في الـ kernel محتاجة تعرف الـ label الحقيقي لـ security context معين.

### الـ Module

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * selinux_secid_probe.c
 *
 * kprobe on security_secid_to_secctx to observe secid lookups.
 * Prints the secid value every time the kernel resolves a secid -> label.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* kprobe API */
#include <linux/lsm/selinux.h>  /* struct lsm_prop_selinux, u32 secid */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs");
MODULE_DESCRIPTION("kprobe on security_secid_to_secctx to trace SELinux secid lookups");

/*
 * security_secid_to_secctx signature (from security/security.c):
 *   int security_secid_to_secctx(u32 secid, char **secdata, u32 *seclen);
 *
 * arg0 = secid  (the raw SELinux security ID — same u32 in lsm_prop_selinux)
 * arg1 = secdata (pointer to receive the label string, e.g. "unconfined_u:...")
 * arg2 = seclen  (pointer to receive string length)
 */

/* pre-handler: runs just before the probed function executes */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * regs_get_kernel_argument() جيبنا الـ argument الأول (secid)
     * من الـ registers حسب calling convention للـ architecture دي.
     * ده نفس الـ secid اللي بيتخزن في lsm_prop_selinux.secid.
     */
    u32 secid = (u32)regs_get_kernel_argument(regs, 0);

    pr_info("selinux_probe: security_secid_to_secctx called, secid=0x%08x (%u)\n",
            secid, secid);
    return 0; /* must return 0 — non-zero aborts the original call */
}

/* post-handler: runs after the probed function returns */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * regs_return_value() بيرجع الـ return value بعد ما الـ function خلصت.
     * لو رجعت 0 يبقى نجح الـ lookup وتم تحويل الـ secid لـ string.
     */
    long ret = regs_return_value(regs);
    pr_info("selinux_probe: security_secid_to_secctx returned %ld\n", ret);
}

/* kprobe struct — بيحدد الـ function اللي هنعمل عليها probe */
static struct kprobe kp = {
    .symbol_name    = "security_secid_to_secctx",
    .pre_handler    = handler_pre,
    .post_handler   = handler_post,
};

static int __init selinux_probe_init(void)
{
    int ret;

    /*
     * register_kprobe بتربط الـ kprobe بالـ function في الـ kernel memory.
     * لو SELinux مش enabled أو الـ symbol مش موجود بترجع error.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("selinux_probe: register_kprobe failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("selinux_probe: planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit selinux_probe_exit(void)
{
    /*
     * unregister_kprobe ضروري في الـ exit عشان نشيل الـ breakpoint
     * من الـ kernel code ونمنع استدعاء handler_pre بعد ما الـ module اتشال —
     * لو مشلناش هينكسر الـ kernel لأن الـ handler address بقى invalid.
     */
    unregister_kprobe(&kp);
    pr_info("selinux_probe: kprobe removed from %s\n", kp.symbol_name);
}

module_init(selinux_probe_init);
module_exit(selinux_probe_exit);
```

### الـ Makefile

```makefile
obj-m += selinux_secid_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | ماكروهات `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info`, `pr_err` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `regs_get_kernel_argument` |
| `linux/lsm/selinux.h` | تعريف `u32 secid` اللي هو نفس اللي في `lsm_prop_selinux` |

#### ليه `security_secid_to_secctx`؟

دي من أكتر الـ functions استخداماً في الـ SELinux subsystem — بتتاخد في audit logging، `/proc/*/attr/current`، وNetlink security queries. كل call عليها معناها إن حاجة في الـ kernel محتاجة تترجم رقم `secid` لـ human-readable label. ده مباشرةً مرتبط بالـ `u32 secid` الموجود في `struct lsm_prop_selinux`.

#### الـ `handler_pre`

`regs_get_kernel_argument(regs, 0)` بيجيب الـ argument الأول (`secid`) من الـ registers قبل ما الـ function تشتغل. الـ `secid` ده نفس الـ `u32` المعرّف في `lsm_prop_selinux.secid` — بيعمل print له بالـ hex والـ decimal عشان يسهّل ربطه بالـ audit logs.

#### الـ `handler_post`

بيطبع الـ return value بعد ما الـ function خلصت. لو رجعت `0` يبقى الـ lookup نجح وتم ملء `secdata` بالـ label string. أي error (زي `-EINVAL` أو `-ENOENT`) يبقى الـ secid مش معروف لـ SELinux.

#### الـ `module_exit` وأهمية `unregister_kprobe`

لو ما عملناش `unregister_kprobe` في الـ exit، الـ kernel هيفضل يستدعي `handler_pre` اللي اتشالت من الـ memory مع الـ module — ده بيعمل kernel panic فوري. الـ unregister بيشيل الـ software breakpoint من الـ kernel code ويرجّع الـ original instruction.

### تجربة الـ Module

```bash
# بناء الـ module
make

# تحميل الـ module (SELinux لازم يكون enabled)
sudo insmod selinux_secid_probe.ko

# شوف الـ output في الـ kernel log
sudo dmesg -w | grep selinux_probe

# أي عملية بتعمل access check هتطلّع سطر زي:
# selinux_probe: security_secid_to_secctx called, secid=0x00000001 (1)
# selinux_probe: security_secid_to_secctx returned 0

# إزالة الـ module
sudo rmmod selinux_secid_probe
```

> **ملاحظة:** الـ module ده محتاج kernel مبني بـ `CONFIG_KPROBES=y` وـ `CONFIG_SECURITY_SELINUX=y`، وأغلب الـ distros (Fedora, RHEL, Ubuntu بـ SELinux) بيوفّروا التنيين.
