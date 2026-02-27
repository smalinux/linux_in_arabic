## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

الملف `include/linux/lsm/bpf.h` جزء من subsystem اسمه **BPF [SECURITY & LSM]** — وده موجود في `MAINTAINERS` تحت عنوان:

> BPF (Security Audit and Enforcement using BPF)

المسؤولون عنه: KP Singh وMatt Bobrowski من Google، والـ mailing list هو `bpf@vger.kernel.org`.

---

### الصورة الكبيرة — قبل أي كود

#### المشكلة اللي بتتحل

تخيل إن عندك بواب في عمارة. البواب ده مش بواب واحد — في الحقيقة في **أكتر من بواب** بيشتغلوا مع بعض في نفس الوقت:

- **SELinux** — البواب الحكومي الرسمي، بيعرف كل شخص برقم تصريح (`secid` يعني `u32`).
- **Smack** — البواب البسيط، بيعرف الناس بـ label مكتوبة.
- **AppArmor** — البواب الذكي، بيشيل pointer لـ struct معقدة اسمها `aa_label`.
- **BPF LSM** — البواب الـ programmable، اللي انت بنفسك بتكتبله القواعد وقت التشغيل.

المشكلة: لما أي subsystem تاني في الكيرنل (زي الـ audit أو الـ networking) عايز يسأل "مين اللي بيعمل العملية دي؟ وإيه مستواه الأمني؟" — لازم يتكلم مع **كل البوابين دول مع بعض**.

عشان كده، الكيرنل عمل struct واحدة شاملة اسمها **`lsm_prop`** جوا `include/linux/security.h`:

```c
struct lsm_prop {
    struct lsm_prop_selinux  selinux;   /* SELinux's secid */
    struct lsm_prop_smack    smack;     /* Smack's label pointer */
    struct lsm_prop_apparmor apparmor;  /* AppArmor's aa_label pointer */
    struct lsm_prop_bpf      bpf;      /* BPF LSM's secid */
};
```

الـ struct دي زي **كارنيه موحد** بيتجمع فيه تعريف كل LSM للـ subject أو الـ object الأمني.

---

#### دور الملف `bpf.h` تحديداً

الملف ده بسيط **جداً** — ده مش عيب، ده تصميم متعمد:

```c
struct lsm_prop_bpf {
#ifdef CONFIG_BPF_LSM
    u32 secid;
#endif
};
```

**الـ BPF LSM** بيمثل هويته الأمنية بـ `u32` واحدة — نفس فكرة SELinux تماماً. الـ `secid` ده رقم فريد بيعرّف الـ security context الخاص بالـ BPF program اللي شايل القرار الأمني.

اللي بيميز الملف ده عن باقي الـ headers في `include/linux/lsm/`:

| LSM | نوع البيانات | الملف |
|---|---|---|
| SELinux | `u32 secid` | `lsm/selinux.h` |
| **BPF** | **`u32 secid`** | **`lsm/bpf.h`** |
| Smack | `struct smack_known *skp` | `lsm/smack.h` |
| AppArmor | `struct aa_label *label` | `lsm/apparmor.h` |

كل LSM بيقول "أنا عايز أحجز إيه في الكارنيه المشترك" — والـ BPF LSM بيقول "أنا محتاج `u32` بس."

---

#### ليه الـ BPF LSM موجود أصلاً؟

الـ LSM التقليدية (SELinux, AppArmor) بتتحمّل في الكيرنل وقت الـ boot وقواعدها محددة. لكن الـ **BPF LSM** بيخليك تكتب **security policy** بنفسك كـ BPF program وتحمّله وقت التشغيل بدون إعادة تشغيل الكيرنل.

مثال حقيقي: شركة زي Google عايزة تمنع أي process من فتح file معين بناءً على شرط ديناميكي — مش محتاجة تكتب kernel module أو تغير SELinux policy. بتكتب BPF program بيتعلق بـ LSM hook مثلاً `bpf_lsm_file_open` وبيقول "لو الـ filename كذا → ارفض."

الـ `secid` في `lsm_prop_bpf` هو الـ identifier اللي بيربط الـ subject (process مثلاً) بالـ BPF security context بتاعه.

---

#### القصة بالكامل خطوة خطوة

```
Process A بتعمل open() على file

         ↓
   VFS Layer بتسأل LSM framework:
   "هل مسموح؟"

         ↓
   security_inode_permission() في security.h
   تجمع lsm_prop للـ subject والـ object

         ↓
   lsm_prop {
     selinux:  { secid: 1234 }
     smack:    { skp: 0xffff... }
     apparmor: { label: 0xffff... }
     bpf:      { secid: 42 }   ← جاي من lsm_prop_bpf
   }

         ↓
   كل LSM مسجّل يشوف النتيجة بتاعته
   لو أي واحد قال "لا" → EACCES
```

---

### الملفات المهمة اللي المفروض تعرفها

#### الـ Core — البنية التحتية

| الملف | الدور |
|---|---|
| `include/linux/lsm/bpf.h` | **الملف ده** — تعريف `lsm_prop_bpf` |
| `include/linux/security.h` | يجمع كل `lsm_prop_*` في struct واحدة `lsm_prop` |
| `include/linux/bpf_lsm.h` | API الكامل للـ BPF LSM |
| `kernel/bpf/bpf_lsm.c` | تنفيذ الـ BPF LSM hooks |
| `kernel/bpf/bpf_lsm_proto.c` | الـ helper functions للـ BPF programs |

#### الـ LSM الأخوات — نفس النمط

| الملف | الدور |
|---|---|
| `include/linux/lsm/selinux.h` | نفس الفكرة — `u32 secid` لـ SELinux |
| `include/linux/lsm/smack.h` | `struct smack_known*` لـ Smack |
| `include/linux/lsm/apparmor.h` | `struct aa_label*` لـ AppArmor |

#### الـ Implementation

| الملف | الدور |
|---|---|
| `security/bpf/hooks.c` | يسجّل كل BPF LSM hooks في الكيرنل |
| `security/bpf/Makefile` | بناء الـ BPF LSM module |

#### الـ Documentation

| الملف | الدور |
|---|---|
| `Documentation/bpf/prog_lsm.rst` | شرح رسمي لكيفية كتابة BPF LSM programs |

---

### الخلاصة

الملف `include/linux/lsm/bpf.h` هو أصغر قطعة في puzzle أكبر — بيقول بوضوح: "الـ BPF LSM لما يحتاج يعرّف هوية أمنية، بيستخدم `u32` واحدة اسمها `secid`." الـ `#ifdef CONFIG_BPF_LSM` بيضمن إن لو الـ BPF LSM مش مفعّل في الكيرنل، الـ struct تبقى فاضية تماماً ومش بتأكل ذاكرة.
## Phase 2: شرح الـ LSM (Linux Security Module) Framework — BPF LSM Interface

### المشكلة اللي بيحلها الـ LSM Framework

الـ kernel بطبيعته بيتعامل مع موارد (files, sockets, processes, IPC) وبيحتاج يسمح أو يرفض العمليات عليها. الـ POSIX permissions التقليدية (rwx + UID/GID) كانت كافية للـ UNIX في الـ 70s، بس في الأنظمة الحديثة دي مش كافية:

- **الـ DAC** (Discretionary Access Control) — صاحب الملف هو اللي بيقرر، وده ضعيف في البيئات الـ multi-tenant والـ containers.
- محتاجين **MAC** (Mandatory Access Control) — الـ policy بتيجي من admin مش من المستخدم.
- كل LSM عنده نموذج أمان مختلف: SELinux بيستخدم labels رقمية، AppArmor بيستخدم paths وpointers، Smack بيستخدم string labels.
- مش ممكن تـ hardcode LSM واحد جوه الـ kernel لأن الـ use cases مختلفة.

**السؤال:** إزاي تضيف سياسات أمان مختلفة للـ kernel من غير ما تحول الـ kernel نفسه لـ SELinux-kernel أو AppArmor-kernel؟

---

### الحل: الـ LSM Hook Architecture

الـ kernel بيحط **hooks** (نقاط استدعاء) في كل العمليات الحساسة:

```
open() syscall
    │
    ▼
VFS layer
    │
    ├──► security_inode_permission()   ← LSM hook
    │        │
    │        ├── capability_inode_permission()
    │        ├── selinux_inode_permission()
    │        └── bpf_lsm_inode_permission()
    │
    ▼
actual inode access
```

كل LSM بيـ"register" نفسه عند الـ hookده، والـ kernel بينادي كل الـ LSMs المسجلة بالترتيب. لو أي LSM قال "ارفض"، العملية بترفض.

---

### الـ BPF LSM: المشكلة التانية

حتى بعد الـ LSM framework، كان في مشكلة: لو عايز تعمل policy جديدة، لازم تكتب kernel module أو تعدل الـ kernel. الـ BPF LSM (موجود من kernel 5.7) بيخليك **تكتب policies بـ BPF programs** تتلود في الـ runtime من الـ userspace من غير recompile.

---

### الـ lsm_prop_bpf: دورها في الصورة الكبيرة

```c
/* include/linux/lsm/bpf.h */
struct lsm_prop_bpf {
#ifdef CONFIG_BPF_LSM
    u32 secid;   /* numeric security label assigned by BPF LSM */
#endif
};
```

الـ `lsm_prop_bpf` هي **الجزء الخاص بالـ BPF LSM** من الـ security label الموحد اللي الـ kernel بيحمله مع كل process وكل object.

---

### البنية المعمارية الكاملة

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Space                               │
│   ┌──────────┐   ┌─────────────┐   ┌────────────────────────┐  │
│   │ SELinux  │   │  AppArmor   │   │  BPF prog (via libbpf) │  │
│   │ policy   │   │  profiles   │   │  loaded at runtime     │  │
│   └────┬─────┘   └──────┬──────┘   └───────────┬────────────┘  │
└────────┼────────────────┼──────────────────────┼───────────────┘
         │                │                      │ BPF_PROG_LOAD
─────────┼────────────────┼──────────────────────┼───────────────
         │                │                      ▼
┌────────┼────────────────┼──────────────────────────────────────┐
│        ▼                ▼            ┌──────────────────────┐   │
│  ┌──────────────────────────────┐    │    BPF Verifier      │   │
│  │   LSM Framework (security.c) │    │  (safety check)      │   │
│  │                              │    └──────────┬───────────┘   │
│  │  ┌────────────────────────┐  │               │               │
│  │  │  lsm_static_calls_     │◄─┼───────────────┘               │
│  │  │  table (per hook)      │  │   attach BPF prog to hook     │
│  │  │  [0]=capability        │  │                               │
│  │  │  [1]=SELinux           │  │                               │
│  │  │  [2]=BPF LSM  ◄────────┼──┼── bpf_lsm_* callbacks        │
│  │  │  [3]=AppArmor          │  │                               │
│  │  └────────────────────────┘  │                               │
│  │                              │                               │
│  │  security_hook invocation:   │                               │
│  │    for each active call →    │                               │
│  │      call static_call()      │                               │
│  └──────────────────────────────┘                               │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              lsm_prop (per-object security label)       │    │
│  │  ┌──────────────────┐  ┌───────────────────────────┐    │    │
│  │  │ lsm_prop_selinux │  │     lsm_prop_apparmor     │    │    │
│  │  │   u32 secid      │  │   struct aa_label *label  │    │    │
│  │  └──────────────────┘  └───────────────────────────┘    │    │
│  │  ┌──────────────────┐  ┌───────────────────────────┐    │    │
│  │  │  lsm_prop_smack  │  │      lsm_prop_bpf         │    │    │
│  │  │  smack_known *skp│  │      u32 secid  ◄──────── │────┼──┐ │
│  │  └──────────────────┘  └───────────────────────────┘    │  │ │
│  └─────────────────────────────────────────────────────────┘  │ │
└───────────────────────────────────────────────────────────────┼─┘
                                                                │
                              BPF LSM assigns numeric IDs to    │
                              subjects/objects, stored here ────┘
```

---

### الـ lsm_prop Family: المقارنة

الـ LSMs المختلفة بتمثل الـ security label بأشكال مختلفة:

| LSM | الـ struct | نوع الـ label | التفسير |
|-----|-----------|--------------|---------|
| SELinux | `lsm_prop_selinux` | `u32 secid` | رقم بيشير لـ context في internal table |
| BPF LSM | `lsm_prop_bpf` | `u32 secid` | رقم بيشير لـ security context معرّف بـ BPF |
| AppArmor | `lsm_prop_apparmor` | `struct aa_label *` | pointer لـ struct معقد بيحمل profiles |
| Smack | `lsm_prop_smack` | `struct smack_known *` | pointer لـ entry في global label list |

**الـ BPF LSM اختار نفس تمثيل الـ SELinux** — `u32 secid` — لأن:
1. البساطة: رقم واحد يمثل الـ context.
2. الـ BPF programs مقيّدة في المعالجة (لا recursion، لا dynamic allocation) فـ pointer للـ struct معقد مش safe.
3. الـ BPF LSM بيخزن الـ context data في الـ BPF maps — الـ `secid` هو الـ key.

---

### الـ CONFIG_BPF_LSM Guard: ليه؟

```c
struct lsm_prop_bpf {
#ifdef CONFIG_BPF_LSM
    u32 secid;
#endif
};
```

لو `CONFIG_BPF_LSM` مش enabled، الـ struct بتبقى **empty struct** — حجمها صفر في الـ C (technically 1 byte بس الـ compiler optimizes). ده يعني:

- الـ `lsm_prop` الـ aggregate اللي بيضم كل الـ `lsm_prop_*` بالـ compile time بيكون بالضبط حجم الـ LSMs المفعّلة فقط.
- مفيش memory overhead لـ LSMs مش موجودة.

---

### الـ `secid` في الـ BPF LSM: بيشتغل إزاي؟

الـ BPF LSM بيستخدم الـ `secid` كـ **handle** للـ security context:

```
BPF LSM flow:
─────────────

1. Process/object يتخلق
        │
        ▼
2. bpf_lsm_task_alloc() hook يتنادى
        │
        ▼
3. BPF program يحدد الـ context المناسب
   ويخزن معلوماته في BPF map
        │
        ▼
4. يحط الـ secid (key في الـ map)
   في lsm_prop_bpf.secid
        │
        ▼
5. عند أي access check:
   bpf_lsm_inode_permission() مثلاً
        │
        ▼
6. BPF program يقرأ subject.secid + object.secid
   يبحث في البوليسي (BPF map)
        │
        ▼
7. يرجع ALLOW أو DENY
```

---

### التمثيل بـ analogy: نظام الـ Badge في مبنى كبير

تخيل مبنى شركة كبير فيه:
- **دواب كتير** (resources: files, sockets, processes).
- **أنظمة أمان متعددة** شغالة بالتوازي.

| المبنى | الـ LSM Framework |
|--------|-----------------|
| الدرع الأمني عند كل باب | الـ LSM hook عند كل syscall |
| جهاز بصمة + كاميرا + badge scanner | capability + SELinux + BPF LSM |
| كل نظام له database منفصلة | كل LSM له `lsm_prop_*` منفصلة |
| رقم الـ badge | الـ `u32 secid` في الـ BPF LSM |
| قاعدة بيانات الـ badges | الـ BPF map اللي فيه الـ security contexts |
| تحديث صلاحيات بدون تغيير الـ hardware | تحميل BPF program جديد بدون reboot |
| الباب بيفتح لو كل الأنظمة وافقت | الـ access بيتسمح لو كل LSMs وافقت |

الـ `lsm_prop_bpf.secid` هو رقم الـ badge — بيعرّف الشخص (أو العملية)، والـ BPF program هو الـ database engine اللي بيقرأ الـ access rules من الـ BPF maps.

---

### الـ Static Call Optimization

مفهوم مهم في الـ LSM Framework الحديث: **الـ static calls** بدل الـ function pointers العادية.

قبلاً الـ LSM hooks كانت linked list من function pointers — كل hook call كان بيعمل:
1. قراءة الـ list header (cache miss).
2. indirection للـ pointer.
3. branch prediction miss.

الحل الحديث (`lsm_static_calls_table`):

```c
struct lsm_static_calls_table {
    /* لكل hook: array بحجم MAX_LSM_COUNT من static calls */
    struct lsm_static_call file_permission[MAX_LSM_COUNT];
    struct lsm_static_call inode_permission[MAX_LSM_COUNT];
    /* ... */
};
```

الـ `MAX_LSM_COUNT` بيتحسب بـ compile time عبر الـ `lsm_count.h`:

```c
#define MAX_LSM_COUNT \
    COUNT_LSMS(
        CAPABILITIES_ENABLED   /* = 1, لو enabled */
        SELINUX_ENABLED        /* = 1, لو enabled */
        BPF_LSM_ENABLED        /* = 1, لو enabled */
        /* ... */
    )
```

**الـ BPF LSM** (`BPF_LSM_ENABLED`) بيضيف واحد للـ count ده — يعني كل hook array بيتمد بـ slot إضافي للـ BPF callbacks.

---

### الـ `__randomize_layout` في الـ LSM Structs

```c
struct security_hook_list {
    struct lsm_static_call *scalls;
    union security_list_options hook;
    const struct lsm_id *lsmid;
} __randomize_layout;
```

الـ `__randomize_layout` جزء من **RANDSTRUCT** kernel hardening — بيعمل randomize لترتيب الـ fields في الـ struct عند كل kernel build. ده بيصعّب هجمات الـ memory corruption اللي بتعتمد على معرفة الـ offsets.

---

### الـ LSM Framework: بيمتلك إيه؟ وبيفوّض إيه؟

#### ما بيملكه الـ LSM Framework:
- تحديد **مواقع الـ hooks** (ده fixed في الـ kernel source).
- ترتيب استدعاء الـ LSMs (الـ stacking order).
- الـ **`lsm_prop` structure** — الـ container الموحد للـ security labels.
- الـ static call infrastructure.
- الـ API الموحد (`security_*` functions) اللي الـ kernel يتعامل معاه.

#### ما بيفوّضه للـ BPF LSM تحديداً:
- **تعريف الـ policy** — الـ BPF programs هي اللي بتحدد "مين يعمل إيه".
- **تحديد الـ secid** لكل process/object.
- **الـ BPF maps** — storage الـ policy والـ context data.
- **runtime updates** — تغيير البوليسي من غير reboot.

```
┌─────────────────────────────────────────────────┐
│              LSM Framework Owns                 │
│  • hook locations in kernel code                │
│  • calling convention (return value semantics)  │
│  • lsm_prop_bpf struct definition               │
│  • stacking order between LSMs                  │
└─────────────────────┬───────────────────────────┘
                      │ delegates to
                      ▼
┌─────────────────────────────────────────────────┐
│              BPF LSM Owns                       │
│  • BPF programs (the actual policy logic)       │
│  • BPF maps (context & policy storage)          │
│  • secid assignment to subjects/objects         │
│  • runtime policy updates via bpftool/libbpf    │
└─────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: الـ lsm_prop

الفكرة المحورية للـ LSM framework هي **الـ `lsm_prop`** (أو `lsm_ctx` في بعض السياقات) — وهي struct بتجمع فيها كل الـ per-LSM labels في هيكل واحد:

```c
/* المفهوم (مش بالضبط الكود لكن بيوضح الفكرة) */
struct lsm_prop {
    struct lsm_prop_selinux selinux;   /* u32 secid */
    struct lsm_prop_smack   smack;     /* struct smack_known * */
    struct lsm_prop_apparmor apparmor; /* struct aa_label * */
    struct lsm_prop_bpf     bpf;       /* u32 secid */
};
```

كل object في الـ kernel (task, inode, socket, ...) بيحمل `lsm_prop` خاصة بيه — كل LSM بيكتب في الـ slot بتاعه فقط. ده بيخلي الـ LSMs **تشتغل بالتوازي بدون تداخل**.

الـ `lsm_prop_bpf` — رغم بساطتها (field واحد `u32 secid`) — هي **نقطة تقاطع** بين الـ static kernel infrastructure والـ dynamic BPF policy engine، وده اللي بيخلي الـ BPF LSM قوي: infrastructure ثابتة + policy ديناميكية.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### نظرة عامة على الملف

الملف `include/linux/lsm/bpf.h` صغير جداً — بس دوره محوري. هو بيعرّف الـ **glue struct** اللي بتربط بين الـ **LSM (Linux Security Module)** subsystem والـ **BPF LSM** module. الملف كله struct واحد بحقل واحد مشروط بـ `CONFIG_BPF_LSM`.

---

### 0. الـ Config Options والـ Flags

| الـ Config / النوع | القيمة | الأثر |
|---|---|---|
| `CONFIG_BPF_LSM` | `y` / `n` | لو `y` → الـ `secid` field موجودة؛ لو `n` → الـ struct فاضية تماماً |
| `u32` | unsigned 32-bit integer | نوع الـ `secid` — يكفي لتمثيل كل الـ security IDs في الكرنل |

> **ملاحظة:** الملف ما فيهوش enums ولا flags إضافية — التصميم مقصود يكون minimal.

---

### 1. الـ Structs المهمة

#### `struct lsm_prop_bpf`

**الغرض:** حاوية بيانات الـ security property الخاصة بـ BPF LSM. بتتضمن الـ **security ID** (`secid`) اللي بيمثل الـ security context اللي BPF programs بتحدده لـ subjects أو objects في النظام.

```c
struct lsm_prop_bpf {
#ifdef CONFIG_BPF_LSM
    u32 secid;  /* BPF security ID — assigned by BPF LSM programs */
#endif
};
```

| الحقل | النوع | الوصف |
|---|---|---|
| `secid` | `u32` | معرّف الـ security context — بيتعيّن من BPF LSM programs، وبيُستخدم في قرارات الـ access control |

**الارتباطات:**
- **الـ `struct lsm_prop_bpf`** بتكون عادةً embedded داخل struct أكبر اسمه `struct lsm_prop` (موجود في `include/linux/lsm_hooks.h` أو `include/linux/security.h`) — اللي بيجمّع الـ security properties من كل الـ LSM modules النشطة.
- الـ `secid` بيُستخدم من الـ BPF programs عبر الـ `bpf_lsm_*` helpers لتمييز الـ subjects (processes) والـ objects (files، sockets...).

---

### 2. رسم علاقات الـ Structs (ASCII)

```
┌─────────────────────────────────────────────┐
│              struct lsm_prop                │
│  (الـ union/container الرئيسي للـ LSM)      │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │        struct lsm_prop_bpf           │   │
│  │  #ifdef CONFIG_BPF_LSM              │   │
│  │      u32 secid;  ◄──────────────────┼───┼── BPF program يكتب هنا
│  │  #endif                             │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │      struct lsm_prop_selinux         │   │
│  │      (SELinux context — مثال)        │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  ┌──────────────────────────────────────┐   │
│  │      struct lsm_prop_smack           │   │
│  │      (Smack label — مثال)            │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
              │
              │ embedded في
              ▼
┌─────────────────────────────────────────────┐
│           struct task_security_struct        │
│           struct inode_security_struct       │
│           struct file_security_struct        │
│           ... (per-object security blobs)    │
└─────────────────────────────────────────────┘
```

---

### 3. دورة حياة الـ `secid` (Lifecycle Diagram)

```
BOOT
  │
  ▼
┌─────────────────────────────────────────────┐
│  bpf_lsm_init()                             │
│  → يسجّل BPF LSM في الـ LSM framework       │
│  → يخصص الـ security blob slots             │
└────────────────────┬────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────┐
│  إنشاء object (task / inode / socket...)    │
│  → alloc_security_blob()                    │
│  → struct lsm_prop_bpf.secid = 0 (default) │
└────────────────────┬────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────┐
│  BPF program يُشغَّل عند LSM hook            │
│  مثال: bpf_lsm_task_alloc()                │
│  → يقرأ / يكتب secid عبر map أو helper     │
│  → struct lsm_prop_bpf.secid = <value>     │
└────────────────────┬────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────┐
│  قرار الـ access control                    │
│  → LSM hook يُقارن الـ secid               │
│  → BPF program يرفض أو يسمح بالعملية       │
└────────────────────┬────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────┐
│  تدمير الـ object                           │
│  → free_security_blob()                     │
│  → struct lsm_prop_bpf تُحرَّر مع الـ blob  │
└─────────────────────────────────────────────┘
```

---

### 4. Call Flow Diagrams

#### تدفق قراءة الـ `secid` من BPF program

```
BPF program يتشغل عند hook
  │
  ▼
bpf_lsm_<hook_name>()
  → يجيب الـ security blob للـ object
    → lsm_prop = security_blob(object)
      → lsm_prop->bpf.secid   ◄── هنا بيقرأ الـ u32
        → يعمل قرار (allow / deny)
          → يرجع 0 أو -EPERM للـ kernel
```

#### تدفق الـ LSM hook الكامل مع BPF

```
syscall (مثلاً: open())
  │
  ▼
security_inode_permission()          ← LSM core
  │
  ├─► call_int_hook(inode_permission, ...)
  │     │
  │     ├─► selinux_inode_permission()   ← SELinux hook
  │     │
  │     └─► bpf_lsm_inode_permission()  ← BPF LSM hook
  │               │
  │               ▼
  │         تشغيل الـ BPF programs المرتبطة بالـ hook
  │               │
  │               ▼
  │         قراءة / مقارنة lsm_prop_bpf.secid
  │               │
  │               ▼
  │         إرجاع 0 (allow) أو -EPERM (deny)
  │
  ▼
نتيجة syscall للـ userspace
```

---

### 5. استراتيجية الـ Locking

الـ `struct lsm_prop_bpf` نفسها **ما فيهاش locks** — بالتصميم. الـ locking بيحصل على مستوى أعلى:

| المورد المحمي | الـ Lock المستخدم | السبب |
|---|---|---|
| الـ `secid` قراءةً وكتابةً | الـ RCU (Read-Copy-Update) أو الـ lock الخاص بالـ object المضيف | الـ secid بيتقرأ كتير وبيتكتب نادراً |
| الـ security blob بتاع الـ task | `task_lock(task)` عند الكتابة | حماية الـ `task_struct` بشكل عام |
| الـ security blob بتاع الـ inode | `inode->i_lock` | الـ inode له lock خاص بيه |
| تسجيل الـ BPF programs على الـ hooks | `bpf_lsm_mutex` (internal) | منع race conditions عند attach/detach |

**ترتيب الـ Locks (Lock Ordering):**
```
task_lock
  └─► inode->i_lock
        └─► (لا locks داخل lsm_prop_bpf نفسها)
```

> الـ `secid` هو مجرد `u32` — الـ atomic read/write عليه على معظم architectures طبيعي بدون lock إضافي، بس الـ consistency مع بقية الـ security state بتتضمن locks على المستوى الأعلى.

---

### خلاصة

| العنصر | التفصيل |
|---|---|
| الـ struct الوحيد | `struct lsm_prop_bpf` |
| الحقل الوحيد | `u32 secid` — مشروط بـ `CONFIG_BPF_LSM` |
| الغرض الأساسي | ربط الـ BPF LSM بالـ LSM property system |
| الـ Locking | inherited من الـ container object |
| التصميم | minimal by design — BPF LSM يحتاج ID واحد بس |
## Phase 4: شرح الـ Functions

### ملخص عام

الملف `include/linux/lsm/bpf.h` هو header صغير جداً — مش فيه functions خالص. الـ file بيعرّف struct واحدة بس هي `lsm_prop_bpf`، وده جزء من نظام الـ **LSM (Linux Security Module)** اللي بيوفر interface موحّد بين الـ security subsystems والـ BPF LSM تحديداً.

---

### جدول الـ APIs والـ Structures

| العنصر | النوع | الغرض | مشروط بـ |
|--------|-------|--------|-----------|
| `struct lsm_prop_bpf` | struct | تخزين الـ security context الخاص بالـ BPF LSM | `CONFIG_BPF_LSM` |
| `lsm_prop_bpf.secid` | `u32` field | معرّف الـ security context (32-bit opaque ID) | `CONFIG_BPF_LSM` |

> **ملاحظة:** الملف ده **لا يحتوي على functions** — ده تصميم متعمّد. الـ struct دي جزء من البنية التحتية للـ LSM prop system اللي بتتجمع مع structs تانية (مثل `lsm_prop_selinux`, `lsm_prop_smack`) في `struct lsm_prop` الكبيرة.

---

### Category: Data Structure Definition

#### الغرض من المجموعة

الـ LSM subsystem في Linux بيستخدم نموذج **stacking** — يعني ممكن أكتر من LSM يشتغل في نفس الوقت (SELinux + BPF LSM + Smack مثلاً). عشان كل LSM يخزن الـ security context بتاعه بشكل isolated، الـ kernel بيعرّف struct صغيرة لكل LSM منفصل، وبعدين بيجمّعهم في union أو struct واحدة كبيرة.

الـ `lsm_prop_bpf` هي الـ per-LSM storage الخاص بـ **BPF LSM**.

---

### `struct lsm_prop_bpf`

```c
/* include/linux/lsm/bpf.h */
struct lsm_prop_bpf {
#ifdef CONFIG_BPF_LSM
    u32 secid;  /* opaque 32-bit security identifier */
#endif
};
```

#### ما هي وبتعمل إيه

الـ `lsm_prop_bpf` هي container صغيرة بتخزن فيها الـ **secid** — وهو 32-bit identifier بيمثل الـ security context اللي الـ BPF LSM حدده لـ object معين (process, socket, file, إلخ).

الـ **secid** ده opaque value يعني مش بيتفسّر من خارج الـ BPF LSM نفسه — الـ subsystems التانية بتشيله وبتمرّره بس.

لو `CONFIG_BPF_LSM` مش معمول `=y`، الـ struct بتبقى **فاضية تماماً** (zero-size struct في C)، وده بيخلي الـ compiler يـeliminate أي overhead لو BPF LSM مش موجود.

#### الـ Field الوحيد

| Field | Type | الوصف |
|-------|------|--------|
| `secid` | `u32` | الـ security label الخاص بالـ BPF LSM. بيتخصص من الـ BPF LSM map أو BPF program عند الـ object creation. قيمته `0` بتعني "no label". |

#### Key Details

**الـ Conditional Compilation (`#ifdef CONFIG_BPF_LSM`):**

```c
struct lsm_prop_bpf {
#ifdef CONFIG_BPF_LSM
    u32 secid;
#endif
};
```

ده pattern قياسي في الـ LSM prop headers. لو `CONFIG_BPF_LSM=n`:
- الـ struct بتبقى empty
- الـ compiler مش بيضيف أي padding أو overhead
- الكود اللي بيستخدم `lsm_prop_bpf.secid` بيطلع compile error لو حاول يوصله من غير guard

**الـ Placement في الـ LSM Stack:**

الـ `lsm_prop_bpf` بتتضمّن جوه `struct lsm_prop` الكبيرة (في `include/linux/lsm_hooks.h` أو مشابه):

```
struct lsm_prop {
    struct lsm_prop_selinux  selinux;
    struct lsm_prop_smack    smack;
    struct lsm_prop_bpf      bpf;    /* ← هنا */
    /* ... */
};
```

**الـ secid vs. secctx:**

| المفهوم | النوع | الاستخدام |
|---------|-------|-----------|
| `secid` | `u32` | in-kernel opaque ID — سريع، مش human-readable |
| `secctx` | `char *` | string representation — للـ userspace والـ audit |

الـ `secid` هو الأسرع للمقارنة والـ enforcement لأنه مجرد integer comparison.

**من بيستخدم الـ struct دي:**

- الـ **BPF LSM programs** نفسها — بتقرأ وبتكتب الـ secid عبر BPF maps أو helper functions
- الـ **LSM hook implementations** — زي `bpf_lsm_task_alloc_security()` اللي بتـalloc وبتـfill الـ struct
- الـ **security_secid_to_secctx()** path — لما محتاج تحوّل الـ secid لـ string للـ audit أو netlabel

---

### الـ Design Pattern: Per-LSM Property Structs

ده pattern معمّد في الـ LSM subsystem الحديث (بعد الـ stacking refactor). الفكرة:

```
include/linux/lsm/
├── bpf.h          ← struct lsm_prop_bpf  { u32 secid; }
├── selinux.h      ← struct lsm_prop_selinux { u32 secid; }
├── smack.h        ← struct lsm_prop_smack { ... }
└── ...
```

كل LSM عنده header منفصل بيعرّف الـ per-LSM data. ده بيحقق:

1. **Zero overhead** لو الـ LSM مش مـenabled (empty struct)
2. **Type safety** — مش ممكن تعمل type confusion بين LSMs
3. **Clean separation** بين الـ LSM implementations
4. **Compile-time enforcement** — الـ `#ifdef` بيضمن إن الكود مش بيستخدم fields مش موجودة

---

### مقارنة مع الـ Legacy `secid` Model

| | Legacy (قبل الـ stacking) | الـ LSM Prop Model الحالي |
|--|--------------------------|--------------------------|
| التخزين | `u32 secid` في كل struct مباشرة | `struct lsm_prop` بيجمع per-LSM structs |
| الـ multi-LSM support | صعب — field واحد لكل الـ LSMs | كل LSM عنده field منفصل |
| الـ overhead | Fixed `u32` دايماً | Zero لو LSM مش enabled |
| الـ type safety | Weak — كل LSMs بتشارك نفس الـ field | Strong — per-LSM typed fields |

---

### الـ BPF LSM Context Flow (للتوضيح)

```
BPF Program يـassign label
         │
         ▼
    lsm_prop_bpf.secid = <value>
         │
         ▼
    lsm_prop (composite struct)
         │
         ├── secid passed to LSM hooks
         ├── compared during access checks
         └── converted to string for audit via
             security_secid_to_secctx()
```
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem اللي بنتكلم عنه هو `linux/lsm/bpf.h` — وده الـ interface بين الـ **Linux Security Module (LSM)** والـ **BPF (Berkeley Packet Filter)**. الـ struct الوحيدة فيه هي `lsm_prop_bpf` اللي بتحمل `secid` (u32) لما `CONFIG_BPF_LSM` يكون enabled. الـ debugging هنا بيشمل جانبين: هل الـ BPF LSM شغال صح؟ وهل الـ `secid` بيتعمل assign صح لكل process/socket؟

---

### Software Level

#### 1. debugfs Entries

الـ BPF LSM مش بيكتب entries مباشرة في debugfs، لكن الـ BPF subsystem العام بيكتب:

```bash
# شوف كل الـ BPF programs المحملة
ls /sys/kernel/debug/bpf/

# اعرض info عن برنامج BPF معين (استبدل ID بالرقم الفعلي)
cat /sys/kernel/debug/bpf/prog/<ID>/xlated_prog

# شوف الـ maps المرتبطة بالـ BPF LSM
ls /sys/kernel/debug/bpf/maps/
```

للـ LSM بالتحديد:

```bash
# اعرض الـ LSM hooks المفعلة
cat /sys/kernel/debug/lsm

# مثال على output متوقع:
# capability,yama,selinux,bpf
```

تفسير الـ output: لو `bpf` مش موجود في القائمة، يبقى `CONFIG_BPF_LSM` مش enabled أو مش في `lsm=` boot parameter.

#### 2. sysfs Entries

```bash
# تحقق من الـ LSMs المفعلة في النظام
cat /sys/kernel/security/lsm
# Output مثال: lockdown,capability,yama,bpf

# شوف حالة الـ BPF JIT (مهم لأداء الـ BPF LSM)
cat /proc/sys/net/core/bpf_jit_enable
# 0 = disabled, 1 = enabled, 2 = enabled + debug

# قراءة الـ BPF unprivileged access
cat /proc/sys/kernel/unprivileged_bpf_disabled
# 0 = allowed, 1 = disabled, 2 = disabled permanently
```

#### 3. ftrace — Tracepoints والـ Events

```bash
# فعّل الـ tracing للـ BPF LSM hooks
echo 1 > /sys/kernel/debug/tracing/events/bpf/enable

# فعّل events خاصة بالـ LSM
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_enter_bpf/enable

# تتبع الـ security hooks مباشرة
echo 'security_bpf*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# أوقف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

الـ functions المهمة للـ trace:
- `security_bpf` — عند load/create برنامج BPF
- `security_bpf_map` — عند access لـ BPF map
- `security_bpf_prog` — عند access لـ BPF program

#### 4. printk والـ Dynamic Debug

```bash
# فعّل الـ dynamic debug لكل ملفات الـ LSM/BPF
echo 'file security/bpf*.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/bpf/bpf_lsm.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لكل الـ LSM subsystem
echo 'module bpf +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ rules الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep bpf

# رفع مستوى الـ printk مؤقتاً
echo 8 > /proc/sys/kernel/printk
```

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_BPF_LSM` | يفعّل الـ BPF LSM أصلاً — شرط أساسي |
| `CONFIG_DEBUG_INFO_BTF` | يضيف BTF info للـ BPF programs (ضروري للـ CO-RE debugging) |
| `CONFIG_BPF_EVENTS` | يفعّل الـ BPF tracepoints |
| `CONFIG_BPF_JIT_ALWAYS_ON` | يجبر الـ JIT (يكشف مشاكل الـ JIT) |
| `CONFIG_SECURITY` | الـ base للـ LSM framework |
| `CONFIG_SECURITYFS` | يوفر `/sys/kernel/security/` |
| `CONFIG_LSM_MMAP_MIN_ADDR` | debugging للـ mmap security |
| `CONFIG_DEBUG_BPF_ENABLE_KFUNC_REG` | يكشف مشاكل تسجيل الـ kfuncs |
| `CONFIG_BPF_SYSCALL` | يفعّل الـ BPF syscall (شرط) |
| `CONFIG_KALLSYMS` | يسمح برؤية الـ symbols في الـ trace |

```bash
# تحقق من الـ config الحالية
grep -E 'CONFIG_BPF_LSM|CONFIG_DEBUG_INFO_BTF|CONFIG_BPF_EVENTS' /boot/config-$(uname -r)
```

#### 6. bpftool — الأداة الأساسية

```bash
# اعرض كل البرامج المحملة
bpftool prog list

# اعرض البرامج من نوع LSM
bpftool prog list | grep lsm

# مثال على output:
# 42: lsm  name bpf_prog_name  tag abcdef123456  gpl
#         loaded_at 2026-02-27T10:00:00+0000  uid 0
#         xlated 128B  jited 96B  memlock 4096B

# اعرض الـ bytecode المترجم
bpftool prog dump xlated id 42

# اعرض الـ JITted code
bpftool prog dump jited id 42

# تحقق من الـ BTF info
bpftool btf dump id 42

# اعرض الـ maps المرتبطة
bpftool map list

# تحقق من LSM hooks المتاحة للـ attach
bpftool feature probe | grep lsm

# شوف الـ prog attached لـ hook معين
bpftool prog show pinned /sys/fs/bpf/my_lsm_prog
```

#### 7. جدول الـ Error Messages

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `bpf: LSM framework does not have CONFIG_BPF_LSM enabled` | الـ kernel مش مبني بـ BPF LSM | إعادة build الـ kernel مع `CONFIG_BPF_LSM=y` |
| `libbpf: prog 'xxx': BPF program load failed: Permission denied` | الـ program بيحاول عمل حاجة مش مسموح بيها | تحقق من الـ CAP_BPF و CAP_SYS_ADMIN |
| `bpf: Tracing/LSM programs require CAP_BPF and CAP_PERFMON` | permissions ناقصة | شغّل كـ root أو أضف الـ capabilities |
| `bpf: Cannot attach program to hook` | الـ hook مش متاح أو BPF LSM مش في قائمة الـ LSMs | أضف `lsm=...,bpf` لـ kernel cmdline |
| `invalid argument: LSM hook not found` | اسم الـ hook غلط أو مش مدعوم | تحقق من الـ kernel version والـ hook name |
| `secid: 0 assigned` | الـ BPF LSM مش بيعمل assign صح للـ secid | تحقق من `security_bpf_prog()` implementation |
| `bpf: LRU map update failed` | الـ LRU map فاضت | زود حجم الـ map أو راجع الـ logic |
| `cannot attach lsm to hook ... prog type mismatch` | نوع الـ program غلط | استخدم `BPF_PROG_TYPE_LSM` |

#### 8. dump_stack() و WARN_ON()

النقاط الاستراتيجية في الكود:

```c
/* في security/bpf/hooks.c — عند assign الـ secid */
static int bpf_lsm_task_alloc(struct task_struct *task, unsigned long flags)
{
    /* نقطة مهمة: تحقق إن الـ secid اتعمل assign صح */
    WARN_ON(task->security == NULL);
    dump_stack(); /* لو محتاج تعرف مين بيعمل alloc */
    return 0;
}

/* عند الـ map access */
static int bpf_lsm_bpf_map(struct bpf_map *map, fmode_t fmode)
{
    struct lsm_prop_bpf *prop = /* get prop */;
    /* تحقق إن الـ secid valid */
    WARN_ON(prop->secid == 0 && IS_ENABLED(CONFIG_BPF_LSM));
    return 0;
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware بتطابق الـ Kernel State

الـ `lsm_prop_bpf` هو software-only struct، لكن الـ BPF programs ممكن تتفاعل مع hardware من خلال الـ hooks:

```bash
# تحقق إن الـ CPU بيدعم الـ features المطلوبة
grep -E 'bpf|jit' /proc/cpuinfo

# تحقق من الـ memory isolation (مهم لأمان الـ BPF)
cat /proc/sys/vm/mmap_min_addr

# تحقق من الـ JIT hardening (hardware protection)
cat /proc/sys/net/core/bpf_jit_harden
# 0 = none, 1 = jit hardening for unprivileged, 2 = all

# تأكد من الـ Spectre mitigations (مهم لـ BPF security)
cat /sys/devices/system/cpu/vulnerabilities/spectre_v2
```

#### 2. Register Dump Techniques

الـ BPF LSM مش بيتعامل مع hardware registers مباشرة، لكن لو بتـ debug مشكلة في الـ JIT compiler:

```bash
# اعرض الـ JITted code كـ assembly لمقارنته بالـ expected
bpftool prog dump jited id <ID> opcodes

# مثال على output:
# int bpf_prog_name(struct pt_regs *ctx):
# bpf_prog_abcdef123456:
#    0:  endbr64
#    4:  push   %rbp
#    5:  mov    %rsp,%rbp
#   ...

# لو محتاج تقرأ memory مباشرة (للـ advanced debugging)
# استخدم devmem2 مع الـ physical address من /proc/iomem
devmem2 0x<physical_addr> w

# أو من خلال /dev/mem (لو مفتوح في الـ kernel)
dd if=/dev/mem bs=1 skip=$((0x<addr>)) count=4 | xxd
```

#### 3. Logic Analyzer / Oscilloscope Tips

الـ BPF LSM هو software subsystem بحت، لكن في سياق الـ embedded systems:

- لو الـ BPF LSM بيتحكم في hardware access (زي network interfaces)، استخدم **logic analyzer** على الـ bus (PCIe/USB) لتأكيد إن الـ access بيتمنع فعلاً
- استخدم **oscilloscope** لقياس الـ latency اللي بيضيفه الـ BPF LSM hook على الـ interrupt handling
- نقاط القياس المفيدة: الـ IRQ line قبل وبعد تفعيل الـ BPF LSM program

```
IRQ Line:  _____|‾‾‾‾‾|_____
                ↑
         قياس الـ latency هنا
         (وقت الـ BPF hook execution)
```

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | Pattern في الـ Kernel Log | السبب المحتمل |
|---|---|---|
| BPF JIT فشل على architecture معينة | `bpf_jit: not supported on this arch` | الـ CPU مش بيدعم الـ JIT features |
| Memory allocation فشل | `bpf: Failed to allocate BPF program memory` | الـ system ناقص RAM أو الـ memory fragmented |
| Spectre mitigation تعارض | `bpf: Spectre v2 mitigation: ... retpoline` | الـ JIT بيعمل workaround للـ Spectre |
| NX bit مشكلة | `bpf: Failed to allocate executable memory` | الـ hardware/firmware مش بيسمح بـ executable pages |

```bash
# ابحث عن الـ patterns دي في الـ kernel log
dmesg | grep -E 'bpf|lsm|security' | head -50
journalctl -k | grep -iE 'bpf_lsm|security_bpf' | tail -30
```

#### 5. Device Tree Debugging

الـ BPF LSM مش بيستخدم Device Tree مباشرة، لكن لو بتـ debug على embedded system:

```bash
# تحقق إن الـ LSM مُعرّف صح في الـ kernel cmdline
cat /proc/cmdline | grep lsm
# Output المتوقع: ... lsm=lockdown,capability,yama,bpf ...

# تحقق من الـ DT node للـ CPU features المطلوبة
dtc -I fs /sys/firmware/devicetree/base | grep -A5 'bpf\|security'

# تأكد من الـ DT boot parameters
cat /proc/device-tree/chosen/bootargs 2>/dev/null
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للـ Copy-Paste

**1. تشخيص سريع — هل الـ BPF LSM شغال؟**

```bash
#!/bin/bash
echo "=== BPF LSM Quick Diagnosis ==="

echo "[1] Enabled LSMs:"
cat /sys/kernel/security/lsm 2>/dev/null || cat /sys/kernel/debug/lsm 2>/dev/null

echo ""
echo "[2] Kernel Config:"
grep CONFIG_BPF_LSM /boot/config-$(uname -r) 2>/dev/null

echo ""
echo "[3] BPF Programs Loaded (LSM type):"
bpftool prog list 2>/dev/null | grep -A2 lsm

echo ""
echo "[4] BPF JIT Status:"
cat /proc/sys/net/core/bpf_jit_enable

echo ""
echo "[5] Recent BPF/LSM kernel messages:"
dmesg | grep -iE '(bpf|lsm)' | tail -10
```

**مثال على output طبيعي:**

```
=== BPF LSM Quick Diagnosis ===
[1] Enabled LSMs:
lockdown,capability,yama,bpf

[2] Kernel Config:
CONFIG_BPF_LSM=y

[3] BPF Programs Loaded (LSM type):
15: lsm  name enforce_policy  tag a1b2c3d4e5f6  gpl
        loaded_at 2026-02-27T09:00:00+0000  uid 0

[4] BPF JIT Status:
1

[5] Recent BPF/LSM kernel messages:
[    2.345678] bpf: Loading BPF LSM
```

**2. مراقبة الـ secid assignment في real-time**

```bash
# تتبع كل مرة بيتعمل فيها security_bpf_prog_alloc
echo 1 > /sys/kernel/debug/tracing/events/bpf/enable
cat /sys/kernel/debug/tracing/trace_pipe &
TRACE_PID=$!

# شغّل البرنامج اللي بتـ debug
bpftool prog load my_lsm.o /sys/fs/bpf/my_lsm

# أوقف الـ trace
kill $TRACE_PID
echo 0 > /sys/kernel/debug/tracing/events/bpf/enable
```

**3. تأكيد إن الـ CONFIG_BPF_LSM مفعّل وإن `secid` بيشتغل**

```bash
# اكتب BPF program صغير لاختبار الـ secid
cat > /tmp/test_lsm_secid.c << 'EOF'
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

SEC("lsm/bpf")
int BPF_PROG(test_bpf_hook, int cmd, union bpf_attr *attr, unsigned int size)
{
    __u32 secid = bpf_get_current_task_btf()->security; /* simplified */
    bpf_printk("BPF LSM hook called, cmd=%d\n", cmd);
    return 0; /* allow */
}

char LICENSE[] SEC("license") = "GPL";
EOF

# compile وload
clang -O2 -target bpf -c /tmp/test_lsm_secid.c -o /tmp/test_lsm_secid.o
bpftool prog load /tmp/test_lsm_secid.o /sys/fs/bpf/test_lsm

# شوف الـ output
cat /sys/kernel/debug/tracing/trace_pipe | grep test_bpf_hook
```

**مثال على output:**

```
     <bpf>-1234    [001] .... 12345.678901: bpf_trace_printk: BPF LSM hook called, cmd=5
```

الـ `cmd=5` هو `BPF_PROG_LOAD` — يعني الـ hook اشتغل صح عند load برنامج BPF تاني.

**4. قياس تأثير الـ BPF LSM على الأداء**

```bash
# قياس الـ latency قبل وبعد تفعيل BPF LSM program
perf stat -e cycles,instructions,cache-misses \
    bpf syscall benchmark-tool 2>&1 | grep -E 'cycles|instructions'

# أو باستخدام bpftool profiling
bpftool prog profile id <ID> duration 5 cycles instructions
```

**5. فحص الـ secid في process معين**

```bash
# شوف الـ security context لـ process معين
cat /proc/<PID>/attr/current

# شوف الـ security labels لكل الـ processes
ps aux | head -5
for pid in $(ps -eo pid --no-headers | head -10); do
    echo -n "PID $pid: "
    cat /proc/$pid/attr/current 2>/dev/null || echo "N/A"
done
```

**6. تشخيص لما يرفض الـ BPF LSM عملية معينة**

```bash
# فعّل audit logging للـ LSM denials
auditctl -a always,exit -F arch=b64 -S bpf -k bpf_lsm_audit

# شوف الـ denials
ausearch -k bpf_lsm_audit | tail -20

# أو من dmesg لو audit مش متاح
dmesg | grep -i 'denied\|rejected\|bpf.*lsm' | tail -20
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — BPF-LSM مش مفعّل وال policy بتتجاهل

#### العنوان
BPF security policy بتتكتب بس ما بتشتغل على gateway صناعي

#### السياق
فريق embedded security بيطوّر industrial IoT gateway على **TI AM62x** بيشغّل Linux 6.6. المنتج بيستخدم **BPF-LSM** عشان يحمي الـ UART وSPI اللي بيتكلموا مع حساسات خطيرة. المهندس كتب BPF program يعمل `check_file_open` على device files زي `/dev/ttyS0`.

#### المشكلة
الـ BPF program بيُحمَّل بنجاح، بس الـ access controls ما بتطبَّق. الـ `secid` اللي المفروض يتبعت مع الـ security context بيبقى دايمًا `0`، وكل الـ policy checks بترجع allow حتى للـ processes المحظورة.

#### التحليل
الملف `include/linux/lsm/bpf.h` بيعرّف:

```c
struct lsm_prop_bpf {
#ifdef CONFIG_BPF_LSM
    u32 secid;  /* only exists when BPF_LSM is compiled in */
#endif
};
```

الـ `secid` field **موجودة فقط لو `CONFIG_BPF_LSM=y`**. لو الـ field مش موجودة، أي كود بيحاول يقرأ أو يكتب فيها بيتجاهل أو بيفشل compile-time. لما المهندس فحص الـ `.config`:

```bash
$ grep BPF_LSM /boot/config-$(uname -r)
# CONFIG_BPF_LSM is not set
```

الـ kernel اتبنى من غير `CONFIG_BPF_LSM`، فالـ struct بقت empty تمامًا:

```c
struct lsm_prop_bpf {
    /* empty — zero size on most compilers, or 1 byte padding */
};
```

**الـ BPF program اتحمّل** لأن `BPF_PROG_TYPE_LSM` ممكن يتحمّل حتى من غير BPF-LSM في بعض configs، لكن **ما اتشبكش** بالـ LSM hooks الحقيقية، وال `secid` اللي بتيجي من `lsm_prop_bpf` دايمًا zero.

#### الحل

**خطوة 1** — إعادة بناء الـ kernel مع تفعيل BPF-LSM:

```bash
# في .config أو عبر menuconfig
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_BPF_LSM=y
CONFIG_LSM="lockdown,yama,bpf"  # لازم bpf يكون في القائمة
```

**خطوة 2** — التأكد إن الـ struct بتحتوي الـ `secid` بعد البناء:

```c
/* test build-time check */
#include <linux/lsm/bpf.h>
#include <linux/build_bug.h>

static_assert(sizeof(struct lsm_prop_bpf) == sizeof(u32),
    "BPF_LSM not enabled — secid field missing!");
```

**خطوة 3** — التحقق runtime:

```bash
$ cat /sys/kernel/security/lsm
lockdown,yama,bpf
```

#### الدرس المستفاد
الـ `#ifdef CONFIG_BPF_LSM` داخل `struct lsm_prop_bpf` معناه إن الـ struct بتتغير حجمها وسلوكها بالكامل حسب الـ Kconfig. لازم دايمًا تتأكد إن `CONFIG_BPF_LSM=y` وإن `bpf` موجود في `CONFIG_LSM` قبل ما تعتمد على `secid`.

---

### السيناريو 2: Android TV Box على Allwinner H616 — تعارض LSM stacking مع secid

#### العنوان
SELinux وBPF-LSM مع بعض بيخلّوا الـ `secid` يتكتب بشكل غلط

#### السياق
شركة بتبني Android TV box على **Allwinner H616**. الـ kernel بيشغّل SELinux كـ primary LSM وبيحاولوا يضيفوا BPF-LSM كـ secondary للـ extra monitoring على HDMI content protection paths. المهندس بيحاول يربط `secid` من `lsm_prop_bpf` مع الـ SELinux SID عشان يعرف مين بيفتح `/dev/hdmi_cec`.

#### المشكلة
الـ `secid` اللي بييجي في الـ BPF hook مش بيتطابق مع الـ SELinux SID اللي بيطلعه `id -Z` أو `ps -Z`. الـ policy decisions بتبقى inconsistent.

#### التحليل
`struct lsm_prop_bpf` بيحتوي فقط على:

```c
struct lsm_prop_bpf {
#ifdef CONFIG_BPF_LSM
    u32 secid;
#endif
};
```

الـ `secid` هنا **خاص بـ BPF-LSM subsystem نفسه** — مش نسخة من الـ SELinux SID. الـ LSM stacking في Linux بيخلي كل LSM عنده الـ security blob الخاص بيه. الـ `lsm_prop_bpf.secid` بيتملّى من BPF-LSM context، مش من SELinux.

لما المهندس راح ينظر في الكود اللي بيملّي `secid`:

```c
/* في security/bpf/hooks.c */
void bpf_lsm_secid(struct lsm_prop *prop, u32 *secid)
{
    *secid = prop->lsmblob[bpf_lsm_slot].bpf.secid;
    /* ده BPF secid، مش SELinux SID */
}
```

الـ `u32 secid` في `lsm_prop_bpf` هو identifier **داخلي للـ BPF-LSM**، منفصل تمامًا عن SELinux SIDs.

#### الحل

عشان تربط بين الاتنين، لازم تستخدم `bpf_get_current_task()` وتقرأ الـ SELinux SID من الـ task security struct مباشرة من الـ BPF program:

```c
/* BPF program صح */
SEC("lsm/file_open")
int BPF_PROG(check_hdmi_open, struct file *file)
{
    /* استخدم bpf_get_current_task للوصول لـ SELinux context */
    struct task_struct *task = bpf_get_current_task_btf();
    u32 sid;

    /* اقرأ SELinux SID من task security blob */
    bpf_core_read(&sid, sizeof(sid),
        &task->security /* + selinux offset via BTF */);

    /* مقارنة مع lsm_prop_bpf.secid غلط هنا */
    return 0;
}
```

أو الأبسط: استخدم `bpf_lsm_prop_*` helpers الرسمية اللي بتتعامل مع الـ stacking صح.

#### الدرس المستفاد
الـ `secid` في `lsm_prop_bpf` مش بديل عن SELinux SID وليه namespace منفصل. لو محتاج تعمل cross-LSM correlation، لازم تستخدم الـ BTF-based helpers أو تعتمد على procfs/audit logs للربط.

---

### السيناريو 3: Automotive ECU على i.MX8 — compile error غريب بسبب missing field

#### العنوان
Build failure في automotive BSP بسبب `lsm_prop_bpf.secid` غير موجود

#### السياق
مهندس BSP بيبني kernel لـ **NXP i.MX8QM** في سيارة — الـ ECU بيتحكم في CAN bus وEthernet للـ ADAS. الشركة عندها out-of-tree security module بيستخدم `struct lsm_prop_bpf` مباشرة للـ audit logging.

#### المشكلة
الـ build فجأة بيفشل بعد تغيير الـ `.config` لتعطيل BPF-LSM عشان تقليل attack surface:

```
security/myco_audit.c:47:23: error: 'struct lsm_prop_bpf' has no member named 'secid'
    prop->lsm.bpf.secid = current_sid;
                   ^~~~~
```

#### التحليل
الـ out-of-tree module كان بيافترض إن `secid` موجود دايمًا:

```c
/* كود قديم غلط في security/myco_audit.c */
#include <linux/lsm/bpf.h>

void myco_set_secid(struct lsm_prop *prop, u32 sid)
{
    prop->lsm.bpf.secid = sid;  /* BOOM لو CONFIG_BPF_LSM=n */
}
```

لما بيرجع للـ header:

```c
struct lsm_prop_bpf {
#ifdef CONFIG_BPF_LSM
    u32 secid;   /* conditional! */
#endif
};
```

لو `CONFIG_BPF_LSM` مش مفعّل، الـ struct فاضية وأي وصول لـ `.secid` بيعمل compile error.

#### الحل

الـ out-of-tree module لازم يحترم نفس الـ `#ifdef`:

```c
#include <linux/lsm/bpf.h>

void myco_set_secid(struct lsm_prop *prop, u32 sid)
{
#ifdef CONFIG_BPF_LSM
    prop->lsm.bpf.secid = sid;
#else
    /* BPF_LSM not compiled in — nothing to set */
    (void)sid;
#endif
}

void myco_get_secid(struct lsm_prop *prop, u32 *sid)
{
#ifdef CONFIG_BPF_LSM
    *sid = prop->lsm.bpf.secid;
#else
    *sid = 0;  /* no BPF-LSM context */
#endif
}
```

أو الأفضل: استخدم inline helpers رسمية من `security/bpf/` بدل ما توصل للـ struct directly.

**Kconfig dependency صح:**

```
config MYCO_AUDIT
    depends on BPF_LSM
    help
      Myco audit module requires BPF LSM support.
```

#### الدرس المستفاد
أي كود بيمس `struct lsm_prop_bpf.secid` **لازم** يلف نفسه بـ `#ifdef CONFIG_BPF_LSM`. الـ struct مصمّمة تبقى empty لو BPF-LSM مش موجود — ده اختيار متعمد مش bug.

---

### السيناريو 4: IoT Sensor Node على STM32MP1 — memory layout assumption خاطئة

#### العنوان
Security context بيتكتب في offset غلط بسبب empty struct على STM32MP1

#### السياق
فريق IoT بيبني sensor hub على **STM32MP1** بيجمع بيانات من I2C وSPI sensors وبيبعتها عبر UART لـ cloud gateway. الـ kernel صغير جدًا (minimal config)، والـ security subsystem بسيط. بيحاولوا يضيفوا lightweight auditing بـ BPF.

#### المشكلة
الـ system بيعمل memory corruption غير مفهوم في بعض الأحيان — الـ audit records بتيجي بـ garbage values. الـ corruption بيحصل فقط في الـ release build مش الـ debug build.

#### التحليل
المهندس عنده struct أكبر بتضم `lsm_prop_bpf`:

```c
/* الـ struct في الكود الخاص بيهم */
struct myco_security_ctx {
    u32 pid;
    struct lsm_prop_bpf bpf_ctx;  /* embedded */
    u32 flags;
};
```

في الـ debug build: `CONFIG_BPF_LSM=y`، فالـ struct layout بيبقى:
```
offset 0:  pid     (4 bytes)
offset 4:  secid   (4 bytes — من lsm_prop_bpf)
offset 8:  flags   (4 bytes)
total: 12 bytes
```

في الـ release build: `CONFIG_BPF_LSM=n`، فالـ `lsm_prop_bpf` بتبقى empty:
```
offset 0:  pid     (4 bytes)
offset 4:  (empty struct — 0 or 1 byte depending on compiler)
offset 4:  flags   (4 bytes)
total: 8 bytes
```

الكود بيكتب `flags` على offset 8 دايمًا بس الـ struct اتغيرت، فبيكتب out-of-bounds.

```c
/* كود المشكلة */
void write_flags(struct myco_security_ctx *ctx, u32 f)
{
    /* لو offsetof(myco_security_ctx, flags) اتغير → corruption */
    *((u32 *)((char *)ctx + 8)) = f;  /* hardcoded offset خطر */
}
```

#### الحل

**لا تعتمد على hardcoded offsets أبدًا:**

```c
void write_flags(struct myco_security_ctx *ctx, u32 f)
{
    ctx->flags = f;  /* compiler يحسب الـ offset الصح */
}
```

**أضف compile-time check لاكتشاف التغيير:**

```c
#include <linux/build_bug.h>

/* لو الـ config اتغير والـ layout اتكسر، يظهر error واضح */
#ifdef CONFIG_BPF_LSM
static_assert(offsetof(struct myco_security_ctx, flags) == 8,
    "layout changed — update serialization code");
#else
static_assert(offsetof(struct myco_security_ctx, flags) == 4,
    "layout changed — update serialization code");
#endif
```

#### الدرس المستفاد
الـ `struct lsm_prop_bpf` بتغيّر حجمها حسب `CONFIG_BPF_LSM`. أي struct بتضمّها لازم تفترض إن layout هتتغير بين configs، وأي serialization أو wire protocol لازم يتبنى على `offsetof()` مش hardcoded numbers.

---

### السيناريو 5: Custom Board Bring-Up على RK3562 — BPF-LSM hook بيتفعّل بدري قبل الـ subsystem يتجهّز

#### العنوان
Kernel panic عند boot على RK3562 بسبب access لـ `secid` قبل BPF-LSM initialization

#### السياق
مهندس بيعمل bring-up لـ custom board على **Rockchip RK3562** في smart display product. الـ kernel بيحتوي على BPF-LSM مفعّل وعنده custom early-boot security hook بيحاول يسجّل كل process creation من أول `init`.

#### المشكلة
الـ kernel بيعمل panic أثناء الـ boot بعد تفعيل BPF-LSM:

```
BUG: kernel NULL pointer dereference at 0x00000004
Call trace:
  bpf_lsm_task_alloc+0x34
  security_task_alloc+0x58
  copy_process+0x3a4
  kernel_clone+0x8c
```

#### التحليل
الـ `struct lsm_prop_bpf` بيتعمل allocate وتهيئة أثناء الـ `security_task_alloc`. الـ BPF-LSM slot في الـ task security blob لازم يكون initialized قبل ما أي hook يحاول يوصله.

```c
struct lsm_prop_bpf {
#ifdef CONFIG_BPF_LSM
    u32 secid;  /* must be initialized before first access */
#endif
};
```

الـ custom hook بتاع المهندس اتسجّل على `task_alloc` لكن بيشتغل **قبل** ما BPF-LSM يخلّص تهيئة الـ slot الخاص بيه في الـ early boot sequence:

```c
/* early boot hook — خطأ */
static int myco_early_task_alloc(struct task_struct *task,
                                  unsigned long clone_flags)
{
    struct lsm_prop *prop = task->security;
    /* هنا prop->lsm.bpf.secid → NULL dereference */
    u32 sid = prop->lsm.bpf.secid;
    myco_log_task_create(task, sid);
    return 0;
}
```

المشكلة إن `task->security` نفسه مش initialized بعد على الـ early processes (قبل `security_add_hooks` يخلّص).

#### الحل

**خطوة 1** — تأخير التسجيل لما بعد LSM initialization كاملة:

```c
static int __init myco_security_init(void)
{
    /* يتسجّل بعد ما security_init() خلّصت */
    security_add_hooks(myco_hooks, ARRAY_SIZE(myco_hooks), &myco_lsm_id);
    return 0;
}
/* لازم يكون late_initcall مش security_initcall */
late_initcall(myco_security_init);
```

**خطوة 2** — إضافة null check قبل الوصول:

```c
static int myco_task_alloc(struct task_struct *task,
                            unsigned long clone_flags)
{
    struct lsm_prop *prop;
    u32 sid = 0;

#ifdef CONFIG_BPF_LSM
    prop = task->security;
    if (prop)  /* guard against early boot */
        sid = prop->lsm.bpf.secid;
#endif

    myco_log_task_create(task, sid);
    return 0;
}
```

**خطوة 3** — تتبّع الـ boot sequence:

```bash
# تأكد إن LSM slots اتهيئت
$ dmesg | grep -E "LSM:|bpf_lsm"
[    0.012345] LSM: initializing lsm=lockdown,yama,bpf
[    0.023456] bpf_lsm: initializing BPF LSM slot 2
```

#### الدرس المستفاد
الـ `secid` في `lsm_prop_bpf` لا يوجد فعليًا ويبقى valid إلا بعد إتمام `security_init()` وتهيئة الـ task security blob. الـ early boot hooks لازم تتعامل مع الـ security props بحذر شديد، وتحط guards ضد uninitialized state.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**LWN.net** هو المرجع الأول لمتابعة تطور الـ kernel — الروابط دي من نتائج بحث حقيقية:

| المقال | الوصف |
|--------|-------|
| [BPF and security](https://lwn.net/Articles/946389/) | نظرة شاملة على استخدام BPF-LSM لمنع هجمات على الـ runtime |
| [Impedance matching for BPF and LSM](https://lwn.net/Articles/813261/) | مناقشة KRSI وكيف يتكامل BPF مع الـ LSM hooks |
| [Kernel runtime security instrumentation](https://lwn.net/Articles/798157/) | أول مقال يشرح فكرة KRSI كـ patch set |
| [MAC and Audit policy using eBPF (KRSI)](https://lwn.net/Articles/809645/) | الـ RFC الأولي لـ KRSI وكيف يستخدم LSM hooks |
| [New BPF map and BTF security LSM hooks](https://lwn.net/Articles/928879/) | إضافة hooks جديدة لـ `bpf_map_create_security` و `bpf_btf_load_security` |
| [bpf-lsm: Check return values of security modules](https://lwn.net/Articles/917290/) | التحقق من return values في الـ BPF LSM programs |
| [Add return value range check for BPF LSM](https://lwn.net/Articles/981664/) | تحسينات على range check لـ return values |
| [bpf-lsm: Extend interoperability with IMA](https://lwn.net/Articles/886575/) | تكامل BPF-LSM مع IMA لتقييم الملفات |
| [BPF signing LSM hook change rejected](https://lwn.net/Articles/1042625/) | رفض تغيير لـ hook خاص بـ BPF signing |
| [New file mode and LSM hooks for eBPF object permission control](https://lwn.net/Articles/735950/) | hooks قديمة للتحكم في صلاحيات BPF objects |

---

### التوثيق الرسمي للـ Kernel

**الملفات دي في source tree** مباشرة أو على `docs.kernel.org`:

```
Documentation/bpf/prog_lsm.rst        ← الشرح الرسمي لـ BPF LSM programs
Documentation/security/lsm.rst        ← معمارية LSM العامة
Documentation/security/lsm-development.rst ← دليل تطوير LSM modules
include/linux/lsm/bpf.h               ← الملف الحالي — يعرّف struct lsm_prop_bpf
include/linux/bpf_lsm.h               ← الـ interface الرئيسي لـ BPF LSM
include/linux/lsm_hook_defs.h         ← تعريفات كل LSM hooks
security/bpf/hooks.c                  ← تنفيذ BPF LSM hooks
```

**روابط على docs.kernel.org:**

- [LSM BPF Programs — The Linux Kernel documentation](https://docs.kernel.org/bpf/prog_lsm.html)
- [BPF Documentation](https://docs.kernel.org/bpf/)
- [Linux Security Module Development](https://www.kernel.org/doc/html/v5.5/security/lsm-development.html)

---

### Kernel Commits المهمة

**الـ commits دي أدخلت أو غيّرت الكود المتعلق بـ BPF LSM:**

| الـ Commit | الوصف |
|-----------|-------|
| `fc611f47f218` | إدخال BPF LSM في kernel 5.7 (الـ hook infrastructure) |
| `641cd7b06c91` | إضافة `BPF_PROG_TYPE_LSM` كـ program type |
| `CONFIG_BPF_LSM` | الـ Kconfig option اللي بيفعّل الـ BPF LSM instrumentation |

**الـ struct `lsm_prop_bpf`** نفسها جزء من refactoring الأحدث لـ LSM properties — الـ `secid` field بتظهر بس لما `CONFIG_BPF_LSM` يكون enabled.

للبحث عن الـ commits بالتفصيل:

```bash
git log --all --oneline -- include/linux/lsm/bpf.h
git log --all --oneline --grep="lsm_prop_bpf"
git log --all --oneline --grep="BPF_LSM"
```

---

### مناقشات الـ Mailing List

- [RFC PATCH: LSM Single calls in secid hooks](https://www.mail-archive.com/audit@vger.kernel.org/msg01513.html) — نقاش على mailing list الـ `audit` حول الـ `secid` hooks في LSM
- **الـ mailing lists المتعلقة:** `linux-security-module@vger.kernel.org` و `bpf@vger.kernel.org`

للبحث في الأرشيف:

```bash
# ابحث في lore.kernel.org
https://lore.kernel.org/bpf/?q=lsm_prop_bpf
https://lore.kernel.org/linux-security-module/?q=BPF+LSM+secid
```

---

### Kernelnewbies.org

صفحات الـ kernel release notes بتشرح الفيتشرز الجديدة بالتفصيل:

| الصفحة | الأهمية |
|--------|---------|
| [Linux 5.7](https://kernelnewbies.org/Linux_5.7) | **أهم صفحة** — هنا دخل BPF LSM (KRSI) لأول مرة |
| [Linux 5.8](https://kernelnewbies.org/Linux_5.8) | تحسينات على BPF LSM بعد أول release |
| [Linux 6.0](https://kernelnewbies.org/Linux_6.0) | إضافة cgroup_sock LSM flavor لـ BPF LSM |
| [Linux 5.4](https://kernelnewbies.org/Linux_5.4) | الـ BPF infrastructure اللي مهّد الطريق |
| [Linux 5.16](https://kernelnewbies.org/Linux_5.16) | تحسينات إضافية على BPF security |

---

### مصادر تقنية إضافية

- [Cloudflare Blog: Live-patching security vulnerabilities with eBPF LSM](https://blog.cloudflare.com/live-patch-security-vulnerabilities-with-ebpf-lsm/) — مثال حقيقي من production
- [eBPF Docs: BPF_PROG_TYPE_LSM](https://docs.ebpf.io/linux/program-type/BPF_PROG_TYPE_LSM/) — توثيق تفصيلي لـ program type
- [eBPF Tutorial 19: Security Detection using LSM](https://eunomia.dev/tutorials/19-lsm-connect/) — tutorial عملي
- [Linux Kernel Driver DB: CONFIG_BPF_LSM](https://cateee.net/lkddb/web-lkddb/BPF_LSM.html) — تفاصيل الـ Kconfig option
- [GitHub: lsm.c selftest](https://github.com/torvalds/linux/blob/master/tools/testing/selftests/bpf/progs/lsm.c) — كود الـ selftests في الـ kernel

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول المفيدة:** الفصول الخاصة بـ kernel interfaces والـ security — الكتاب متاح مجاناً على [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- **ملاحظة:** الكتاب قديم (2005) — مش بيغطي BPF LSM، لكن أساسيات الـ kernel architecture مهمة.

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول المفيدة:**
  - Chapter 17: Devices and Modules — لفهم الـ subsystem architecture
  - Chapter 19: Portability — لفهم الـ `#ifdef CONFIG_*` patterns زي اللي في `lsm_prop_bpf`
- **الأهمية:** بيشرح كيف الـ kernel بيبني abstractions فوق hardware/software features اختيارية.

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- مفيد لفهم الـ kernel configuration system والـ `CONFIG_*` options
- بيشرح كيف features زي `CONFIG_BPF_LSM` بتتحكم في compile-time inclusion

---

### Search Terms للبحث عن معلومات أكثر

```
# للبحث في Google/DuckDuckGo
"lsm_prop_bpf" linux kernel
"struct lsm_prop" secid BPF
BPF LSM KRSI kernel runtime security instrumentation
BPF_PROG_TYPE_LSM hook secid
linux security module BPF secid propagation
CONFIG_BPF_LSM kernel documentation
bpf lsm hook "u32 secid"
lsm_prop union BPF kernel

# للبحث في lore.kernel.org
https://lore.kernel.org/bpf/
https://lore.kernel.org/linux-security-module/

# للبحث في kernel source
grep -r "lsm_prop_bpf" linux/
grep -r "CONFIG_BPF_LSM" linux/security/
```
## Phase 8: Writing simple module

### الفكرة

الـ `include/linux/lsm/bpf.h` بيعرّف struct واحدة بس هي `lsm_prop_bpf` اللي بتحمل `secid` — الـ security ID الخاص بالـ BPF LSM. الـ struct دي بتتملى لما الكيرنل بيستدعي `security_bpf()` عشان يتحقق من permissions قبل ما process أي عملية BPF syscall. هنعمل **kprobe** على الدالة `security_bpf` عشان نطبع معلومات عن كل محاولة BPF syscall وهي بتعدي على الـ LSM layer.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_bpf_lsm.c
 * Hook security_bpf() to log every BPF syscall crossing the LSM gate.
 */

/* ---------- includes ---------- */
#include <linux/kernel.h>      /* pr_info, printk macros */
#include <linux/module.h>      /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/sched.h>       /* current, task_comm_len */
#include <linux/uaccess.h>     /* not needed here but common guard */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("LSM BPF Watcher");
MODULE_DESCRIPTION("kprobe on security_bpf() to observe BPF LSM gate crossings");

/*
 * الـ includes دي ضرورية:
 * - linux/kprobes.h: بتعرّف struct kprobe والـ API بتاعتها.
 * - linux/sched.h: بتديك وصول لـ `current` (الـ task_struct للعملية الحالية).
 */

/* ---------- kprobe handler ---------- */

/*
 * security_bpf() signature (from security/security.c):
 *   int security_bpf(int cmd, union bpf_attr *attr, unsigned int size);
 *
 * pre_handler يتنفذ قبل ما security_bpf نفسها تشتغل.
 * الـ regs بيحمل قيم الـ registers في وقت الاستدعاء:
 *   - regs->di = cmd  (أول argument على x86-64)
 *   - regs->si = attr (تاني argument)
 *   - regs->dx = size (تالت argument)
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Extract cmd (first arg: which BPF command is being called) */
    int cmd = (int)regs->di;

    /*
     * current->comm: اسم الـ process اللي استدعت BPF syscall.
     * current->pid:  الـ PID بتاعها.
     * cmd:           رقم أمر BPF زي BPF_PROG_LOAD=5, BPF_MAP_CREATE=0, إلخ.
     */
    pr_info("[bpf_lsm_watch] process='%s' pid=%d  BPF cmd=%d  => hitting LSM gate\n",
            current->comm,
            current->pid,
            cmd);

    /* returning 0 means "let the probed function run normally" */
    return 0;
}

/*
 * الـ handler_pre شغلته قبل الدالة الأصلية:
 * بنطبع اسم الـ process والـ PID ورقم أمر BPF اللي طلبته،
 * ده بيساعد في مراقبة مين بيعمل BPF operations على النظام.
 */

/* ---------- post handler (optional) ---------- */

/*
 * post_handler بيتنفذ بعد ما security_bpf ترجع بنجاح.
 * بنستخدمه عشان نطبع رسالة إضافية للتأكيد.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* regs->ax holds the return value on x86-64 */
    long ret = (long)regs->ax;

    pr_info("[bpf_lsm_watch] security_bpf() returned %ld for pid=%d (%s)\n",
            ret, current->pid, current->comm);
}

/*
 * الـ post_handler بيوضح إيه اللي الـ LSM قررهـ:
 * لو ret == 0 يعني LSM سمح بالعملية،
 * لو ret < 0 (زي EPERM) يعني رفضها.
 */

/* ---------- kprobe struct ---------- */

static struct kprobe kp = {
    .symbol_name    = "security_bpf",   /* الدالة اللي هنعمل probe عليها */
    .pre_handler    = handler_pre,
    .post_handler   = handler_post,
};

/*
 * الـ struct kprobe بتحمل اسم الـ symbol المستهدف والـ handlers.
 * الكيرنل بيحول الاسم لعنوان في الذاكرة تلقائيًا وقت التسجيل.
 */

/* ---------- module init ---------- */

static int __init bpf_lsm_watch_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[bpf_lsm_watch] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[bpf_lsm_watch] kprobe planted on security_bpf @ %p\n",
            kp.addr);
    return 0;
}

/*
 * register_kprobe بتحط breakpoint على عنوان security_bpf في الذاكرة.
 * لو فشلت (مثلًا الـ symbol مش موجود أو محمي) بترجع error code سالب.
 */

/* ---------- module exit ---------- */

static void __exit bpf_lsm_watch_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[bpf_lsm_watch] kprobe removed from security_bpf\n");
}

/*
 * unregister_kprobe ضروري جدًا في exit:
 * لو منزلناش الـ probe قبل تفريغ الـ module، الكيرنل هيكال handler_pre
 * اللي عنوانه بقى invalid وده بيسبب kernel panic.
 */

module_init(bpf_lsm_watch_init);
module_exit(bpf_lsm_watch_exit);
```

---

### Makefile

```makefile
obj-m += kprobe_bpf_lsm.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### تشغيل والمخرجات

```bash
# بناء وتحميل الـ module
sudo make
sudo insmod kprobe_bpf_lsm.ko

# شغّل أي أمر BPF عشان تشوف الـ output
sudo bpftool prog list

# راقب الـ kernel log
sudo dmesg | grep bpf_lsm_watch
```

**مثال على المخرجات:**

```
[bpf_lsm_watch] kprobe planted on security_bpf @ ffffffffc0a12340
[bpf_lsm_watch] process='bpftool' pid=4821  BPF cmd=5  => hitting LSM gate
[bpf_lsm_watch] security_bpf() returned 0 for pid=4821 (bpftool)
[bpf_lsm_watch] process='bpftool' pid=4821  BPF cmd=0  => hitting LSM gate
[bpf_lsm_watch] security_bpf() returned 0 for pid=4821 (bpftool)
```

- الـ `cmd=5` يعني `BPF_PROG_LOAD`
- الـ `cmd=0` يعني `BPF_MAP_CREATE`
- الـ `returned 0` يعني الـ LSM سمح بالعملية

```bash
# إزالة الـ module
sudo rmmod kprobe_bpf_lsm
```

---

### ليه security_bpf تحديدًا؟

الـ `struct lsm_prop_bpf` اللي في الهيدر بتتستخدم جوه الـ BPF LSM subsystem لتخزين الـ `secid` الخاص بكل task. الدالة `security_bpf()` هي **نقطة الدخول الوحيدة** اللي بيعدي منها كل BPF syscall قبل ما الكيرنل ينفذه — يعني hooking عليها بيديك رؤية كاملة لكل BPF activity على النظام دون ما تحتاج تعدّل كود الكيرنل نفسه.
