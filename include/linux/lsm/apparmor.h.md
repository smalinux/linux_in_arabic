## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **Linux Security Module (LSM) Framework** — الـ subsystem اللي بيتم maintain ته تحت **SECURITY SUBSYSTEM** في `MAINTAINERS`، بيشمل كل الـ security policies في الـ kernel.

---

### القصة من الأول

تخيل إنك شغال في شركة كبيرة وعندك **حارس أمن واحد** عند الباب. كل واحد بييجي لازم يوريه **بطاقة تعريف**. المشكلة إن في الشركة ناس بتستخدم بطاقات اتحادية زي **SELinux**، وناس بتستخدم بطاقات خاصة زي **AppArmor**، وناس بتستخدم بطاقات برمجية زي **BPF LSM**، وناس بتستخدم بطاقات بسيطة زي **Smack**.

الحارس (= kernel subsystems زي networking, filesystem, IPC) مش مهمه يعرف كل أنواع البطاقات دي. محتاج بس **محفظة واحدة موحدة** يقدر يحطها في جيبه ويشيل فيها أي بطاقة من أي نوع.

هنا بييجي دور **`struct lsm_prop`** — المحفظة دي. وكل بطاقة من البطاقات دي عندها **slot** جوا المحفظة.

الـ file اللي بنتكلم عنه `include/linux/lsm/apparmor.h` هو **البطاقة الخاصة بـ AppArmor** — بيعرّف الـ slot بتاعها جوا المحفظة المشتركة دي.

---

### AppArmor إيه أصلاً؟

**AppArmor** (Application Armor) هو نظام أمان بيشتغل على مبدأ **Mandatory Access Control (MAC)**. بدل ما تقول "المستخدم ده معندوش صلاحية"، بتقول "البرنامج ده مسموحله يعمل كذا بس".

مثلاً: برنامج الـ browser مسموحله يقرأ من `/home/user/Downloads` بس — حتى لو اتعمله exploit، مش هيقدر يوصل لـ `/etc/passwd`.

كل process شغال على Linux ممكن يكون عنده **`aa_label`** — ده زي بطاقة الهوية الخاصة بـ AppArmor اللي بتقول "البرنامج ده عنده profile اسمه كذا، ومسموحله بكذا وكذا".

---

### الهدف من الـ File ده

الـ file `include/linux/lsm/apparmor.h` بيعمل حاجة واحدة بسيطة جداً:

```c
struct aa_label;  /* forward declaration بس — مش محتاجين التفاصيل هنا */

struct lsm_prop_apparmor {
#ifdef CONFIG_SECURITY_APPARMOR
    struct aa_label *label;  /* pointer للـ label الخاص بـ AppArmor */
#endif
};
```

بيعرّف **`struct lsm_prop_apparmor`** — ده الـ slot الخاص بـ AppArmor جوا المحفظة المشتركة `struct lsm_prop`.

لو الـ kernel اتبُني **من غير** AppArmor (يعني `CONFIG_SECURITY_APPARMOR` مش موجود)، الـ struct بيبقى **فاضي تماماً** — مبياخدش أي memory.

---

### ليه محتاجين الـ File ده؟ المشكلة الأصلية

قديماً الـ kernel كان بيحتفظ بـ **`secid`** — رقم بسيط بيمثل هوية الـ process الأمنية. المشكلة إن:

- **SELinux** بيحتاج `u32 secid` (رقم بحت)
- **AppArmor** بيحتاج `struct aa_label *` (pointer لـ struct معقد)
- **Smack** بيحتاج `struct smack_known *` (pointer لـ struct تاني)
- **BPF LSM** بيحتاج `u32 secid` تاني

مستحيل تخزنهم في نفس الـ field. الحل: اعمل **struct كبير** فيها field لكل LSM، واللي مش مفعّل ميبقاش موجود خالص.

```
struct lsm_prop {
    ┌─────────────────────┐
    │ lsm_prop_selinux    │  ← u32 secid (أو فاضي)
    ├─────────────────────┤
    │ lsm_prop_smack      │  ← pointer (أو فاضي)
    ├─────────────────────┤
    │ lsm_prop_apparmor   │  ← pointer لـ aa_label (أو فاضي)  ← ده اللي بنتكلم عنه
    ├─────────────────────┤
    │ lsm_prop_bpf        │  ← u32 secid (أو فاضي)
    └─────────────────────┘
}
```

---

### مين بيستخدم الـ `lsm_prop_apparmor`؟

الـ `struct lsm_prop` الكاملة (اللي بتشمل الـ apparmor slot) بيتم استخدامها في:

- **`security_inode_getlsmprop()`** — لما الـ filesystem محتاج يعرف هوية الـ inode الأمنية
- **`security_cred_getlsmprop()`** — لما محتاجين هوية الـ process الأمنية من الـ credentials
- **`security_current_getlsmprop_subj()`** — هوية الـ process الحالي
- **الـ audit subsystem** — لتسجيل الأحداث الأمنية في الـ audit log

---

### العلاقة بين الـ Files

```
include/linux/security.h
    ├── #include <linux/lsm/apparmor.h>   ← الـ file بتاعنا
    ├── #include <linux/lsm/selinux.h>
    ├── #include <linux/lsm/smack.h>
    └── #include <linux/lsm/bpf.h>
            ↓
    struct lsm_prop {
        lsm_prop_selinux  selinux;
        lsm_prop_smack    smack;
        lsm_prop_apparmor apparmor;   ← جزء من الـ struct الموحدة
        lsm_prop_bpf      bpf;
    }
```

---

### الـ Files المهمة في الـ Subsystem ده

| الـ File | الدور |
|---------|-------|
| `include/linux/lsm/apparmor.h` | **الـ file بتاعنا** — الـ slot الخاص بـ AppArmor في الـ lsm_prop |
| `include/linux/lsm/selinux.h` | نفس الفكرة لـ SELinux — بس بـ `u32 secid` |
| `include/linux/lsm/smack.h` | نفس الفكرة لـ Smack — بـ `struct smack_known *` |
| `include/linux/lsm/bpf.h` | نفس الفكرة لـ BPF LSM — بـ `u32 secid` |
| `include/linux/security.h` | **القلب** — بيجمع كل الـ slots في `struct lsm_prop` |
| `include/linux/lsm_hooks.h` | بيعرّف كل الـ hooks اللي LSMs بتستخدمها |
| `include/linux/lsm_hook_defs.h` | الـ macro definitions للـ hooks |
| `security/apparmor/lsm.c` | تنفيذ الـ AppArmor LSM hooks |
| `security/apparmor/label.c` | إدارة الـ `aa_label` structures |
| `security/apparmor/policy.c` | الـ profiles وتحميل الـ policies |
| `security/apparmor/domain.c` | الـ domain transitions (تغيير الـ label عند exec) |
| `include/uapi/linux/lsm.h` | الـ user-space API للـ LSM framework |
## Phase 2: شرح الـ LSM (Linux Security Module) Framework وموقع AppArmor فيه

---

### المشكلة اللي الـ LSM Framework بيحلها

قبل الـ LSM، لو عايز تضيف security policy جديدة للـ kernel — مثلاً SELinux أو AppArmor — كنت لازم **تعدل في kernel source code مباشرة**. كل security model جديد كان بيضيف `if` conditions وشروط متفرقة في أماكن زي `fs/`, `net/`, `ipc/` إلخ.

النتيجة؟
- **kernel bloat**: كود security متناثر في كل مكان
- **conflict**: مينفعش تشغّل SELinux وAppArmor في نفس الوقت أصلاً بشكل منظم
- **unmaintainable**: كل security model بيتعدل بشكل مختلف

الـ **LSM Framework** ظهر كـ **generic hook mechanism** — بدل ما يبقى فيه كود security مبعثر، اتعمل نقاط hook محددة (مثل باب) في كل مكان حساس في الـ kernel، والـ security modules تشترك فيها.

---

### الحل — فكرة الـ Hook

الـ kernel بيعرف إن في عمليات حساسة أمنياً: فتح ملف، إنشاء socket، exec process، إلخ.

الـ LSM بيضيف **hook calls** في هذه النقاط. كل `security_*()` function هي hook point. لو مفيش LSM مسجّل، بترجع default (مسموح عادةً). لو فيه LSM (أو أكتر)، بيستدعي callbacks بتاعتهم بالترتيب.

```c
/* مثال: لما process تحاول تفتح ملف */
int security_file_open(struct file *file)
{
    /* بيمر على كل LSM مسجّل ويسأله: هل مسموح؟ */
    return call_int_hook(file_open, file);
}
```

---

### تشبيه من الواقع — مطار فيه أكتر من Checkpoint

تخيل مطار فيه رحلات دولية:

| مكوّن الـ LSM | المقابل في المطار |
|---|---|
| **Kernel** | المطار نفسه — الرحلات هتعدي على أي حال |
| **Hook point** | نقطة تفتيش (Checkpoint) — كل مسافر لازم يعدي |
| **LSM (مثل AppArmor)** | موظف الجوازات — بيقرر يعدّي أو لأ |
| **security_hook_list** | قائمة الموظفين على كل checkpoint |
| **lsm_static_call** | إزاي بيتم استدعاء الموظف (intercom مباشر أو تليفون) |
| **lsm_blob** | ختمة الجواز الخاصة بكل دولة — بيانات خاصة بكل LSM |
| **`aa_label`** | هوية المسافر في نظام AppArmor تحديداً — ماله علاقة بـ SELinux |
| **`lsm_prop_apparmor`** | الجيب اللي فيه ختمة AppArmor فقط |

الفرق المهم: لو فيه موظفَي جوازات (AppArmor + SELinux معاً)، المسافر لازم **يعجب الاتنين** — مش يعجب واحد بس.

---

### البنية الكبيرة — أين يجلس الـ LSM في الـ Kernel؟

```
 ┌─────────────────────────────────────────────────────┐
 │                   User Space                        │
 │   (process يطلب open() أو connect() أو exec() ...)  │
 └───────────────────────┬─────────────────────────────┘
                         │  syscall
                         ▼
 ┌─────────────────────────────────────────────────────┐
 │               Kernel Core Subsystems                │
 │  ┌──────────┐  ┌──────────┐  ┌────────────────────┐│
 │  │ VFS/FS   │  │ Network  │  │  IPC / Memory      ││
 │  │          │  │ Stack    │  │                    ││
 │  └────┬─────┘  └────┬─────┘  └────────┬───────────┘│
 │       │             │                 │             │
 │   security_*()  security_*()      security_*()      │
 │       │             │                 │             │
 └───────┼─────────────┼─────────────────┼─────────────┘
         │             │                 │
         └──────────┬──┘─────────────────┘
                    ▼
 ┌──────────────────────────────────────────────────────┐
 │              LSM Framework Layer                     │
 │                                                      │
 │   static_calls_table (hook dispatch table)           │
 │   ┌──────────────────────────────────────────────┐   │
 │   │ .file_open  → [AppArmor_cb][SELinux_cb][...] │   │
 │   │ .socket_connect → [AppArmor_cb][SELinux_cb]  │   │
 │   │ .task_kill  → [Yama_cb][AppArmor_cb]         │   │
 │   └──────────────────────────────────────────────┘   │
 └───────────────┬──────────────────────────────────────┘
                 │
    ┌────────────┼─────────────┐
    ▼            ▼             ▼
 ┌──────────┐ ┌──────────┐ ┌──────────┐
 │AppArmor  │ │SELinux   │ │  Yama    │
 │(aa_label)│ │(sid/ctx) │ │          │
 └──────────┘ └──────────┘ └──────────┘
```

---

### الـ Core Abstraction — فكرة الـ Security Blob

كل process (task)، ملف، inode، socket — في الـ kernel عنده مساحة اسمها **security blob**: void pointer أو offset في struct، الـ LSM بيخزن فيها بياناته الخاصة.

مثلاً لـ `struct cred`:
```c
struct cred {
    /* ... uid, gid, capabilities ... */
    void *security;   /* ← هنا الـ LSM يخزن بياناته */
};
```

الـ AppArmor بيخزن هنا `struct aa_label *` — وده هو جوهر الملف اللي بندرسه.

---

### الـ Structs المحورية — كيف تترابط مع بعض

```
struct lsm_info              ← تعريف الـ LSM نفسه (AppArmor كـ module)
    │
    ├─ lsm_id { name, id }   ← اسم وID مميز (LSM_ID_APPARMOR)
    ├─ lsm_blob_sizes        ← كم byte يحتاج AppArmor في كل blob type
    └─ init()                ← دالة التهيئة

                    ┌─────────────────────────────────────┐
                    │       static_calls_table             │
                    │  .file_open[0] = lsm_static_call    │
                    │       ├─ key  → static_call_key     │
                    │       ├─ hl   → security_hook_list  │
                    │       │         ├─ hook.file_open   │
                    │       │         │   = apparmor_file_open()
                    │       │         └─ lsmid → &apparmor_lsmid
                    │       └─ active → static_key_false  │
                    └─────────────────────────────────────┘

struct lsm_prop_apparmor   ← الملف اللي بندرسه
    └─ aa_label *label     ← pointer للـ label الخاص بالـ AppArmor
                              (مثلاً: /usr/bin/firefox → profile "firefox")
```

---

### الـ `lsm_prop_apparmor` — ليه موجودة؟

ده struct بسيط جداً — سطرين كود فعلياً:

```c
struct lsm_prop_apparmor {
#ifdef CONFIG_SECURITY_APPARMOR
    struct aa_label *label;  /* pointer to AppArmor security label */
#endif
};
```

**ليه ده مهم؟**

الـ LSM framework بيحتاج طريقة لنقل security context بين subsystems من غير ما يخلي أي subsystem يعرف تفاصيل الـ LSM الداخلية. الـ `lsm_prop_apparmor` هو **interface struct** — يعني:

- الـ VFS مش بيعرف إيه هو `aa_label` ولا هياكله الداخلية
- بس بيعرف يمسك `lsm_prop_apparmor` ويديه للـ AppArmor
- الـ AppArmor هو الوحيد اللي بيفك محتواه

ده مبدأ **opacity** — الـ consumer بيمرر البيانات من غير ما يفهمها.

في الـ kernel، مثلاً في syscall path، لما يحتاج يجيب الـ security context للـ current task:

```c
/* kernel code عام — مش عارف إيه هو lsm_prop_apparmor جوا */
struct lsm_prop prop;
security_current_getlsmprop_subj(&prop);  /* LSM framework يملاها */

/* بعدين يعدّيها للـ AppArmor عبر interface */
```

---

### الـ `aa_label` — قلب AppArmor

الـ `struct aa_label` (forward declared في الملف) هو الـ **security context** في AppArmor. بيمثل:

- مجموعة الـ **profiles** اللي الـ task شاغلها (ممكن يكون confined بأكتر من profile — stacked profiles)
- كل profile بيحدد: إيه الملفات المسموح قراءتها، إيه الـ capabilities المسموحة، إيه الـ network operations المتاحة

```
aa_label
    └─ vec[] → { aa_profile_1, aa_profile_2, ... }
                     │
                     └─ rules: allow /tmp/r, deny /etc/w
                                allow net tcp, deny cap sys_admin
```

---

### الـ Static Call Mechanism — ليه مش function pointer عادي؟

ده مفهوم متقدم يستحق شرح:

**الـ Function pointer العادي** بيتطلب indirect branch — بطيء على الـ modern CPUs بسبب branch prediction misses، وخطر أمني بسبب Spectre/Meltdown.

**الـ static_call** (مكتبة kernel خاصة) بيحوّل الـ indirect call لـ **direct call** في runtime بعد ما الـ LSMs اتسجّلت. لما مفيش LSM، الـ call بتروح لـ NOP function. لما LSM واحد بس، بتروح مباشرة ليه. لو أكتر من LSM، بتعملهم loop.

```
/* compile time: */
DEFINE_STATIC_CALL(lsm_file_open, lsm_default_file_open);

/* runtime بعد تسجيل AppArmor: */
/* بيتعدل الـ machine code مباشرة (text patching) */
/* call apparmor_file_open  ← direct call بدون indirection */
```

هذا يعني الـ overhead لو مفيش LSM = **صفر تقريباً** (NOP or single direct call).

---

### الـ `lsm_blob_sizes` — إزاي كل LSM بيحجز مساحته

لما الـ kernel يـ boot، كل LSM بيعلن كم byte يحتاج في كل نوع blob:

```c
/* AppArmor بيقول: */
static struct lsm_blob_sizes apparmor_blob_sizes __ro_after_init = {
    .lbs_cred  = sizeof(struct aa_task_ctx),  /* لكل task credentials */
    .lbs_file  = sizeof(struct aa_file_ctx),  /* لكل مفتوح file */
    .lbs_inode = sizeof(struct aa_inode_sec), /* لكل inode */
    /* ... */
};
```

الـ LSM framework بيجمع كل الأحجام، ويحجز memory واحدة (blob) لكل object، وكل LSM بياخد offset معين جواه.

```
struct task's security blob:
┌─────────────────────────────────────┐
│ offset 0:  AppArmor data (16 bytes) │
│ offset 16: SELinux data (8 bytes)   │
│ offset 24: Smack data   (4 bytes)   │
└─────────────────────────────────────┘
```

---

### اللي الـ LSM Framework بيمتلكه vs اللي بيفوّضه للـ Drivers/Modules

| الـ LSM Framework يمتلك | الـ LSM Module (AppArmor) يمتلك |
|---|---|
| تعريف hook points (أكتر من 200 hook) | تنفيذ callbacks لكل hook يهمه |
| ترتيب استدعاء الـ LSMs | logic القرار (مسموح/مرفوض) |
| حجز وإدارة security blobs | تفسير محتوى الـ blob |
| الـ static_calls dispatch | الـ `aa_label` وكل هياكل الـ profiles |
| الـ audit infrastructure المشترك | الـ audit data الخاصة بـ AppArmor |
| stacking (تجميع نتايج كل LSMs) | policy language و profile loading |

---

### رحلة الـ `open()` syscall مع AppArmor

```
user: open("/etc/passwd", O_RDONLY)
         │
         ▼
sys_open()
         │
         ▼
do_filp_open()
         │
         ▼
vfs_open()
         │
         ├─→ security_file_open(file)   ← hook point
         │         │
         │         ▼
         │   call_int_hook(file_open, file)
         │         │
         │         ├─→ apparmor_file_open(file)
         │         │       │
         │         │       ├─ aa_current_label()   ← يجيب aa_label من cred blob
         │         │       ├─ يطابق الملف بالـ profile rules
         │         │       └─ return 0 (OK) or -EACCES
         │         │
         │         └─→ selinux_file_open(file) ← لو SELinux شغّال كمان
         │
         └─ لو أي LSM رجّع error → open() fails
```

---

### لماذا `CONFIG_SECURITY_APPARMOR` يلف الـ struct؟

```c
struct lsm_prop_apparmor {
#ifdef CONFIG_SECURITY_APPARMOR
    struct aa_label *label;
#endif
};
```

لو AppArmor مش compiled في الـ kernel، الـ struct هيبقى **فاضي تماماً** — zero size. ده يضمن إن الـ code اللي بيستخدم `lsm_prop_apparmor` يشتغل على kernels بدون AppArmor من غير أي overhead أو compile errors.

الـ code اللي بيتعامل مع الـ struct بيعمل نفس الـ guard:

```c
/* في أي مكان تاني في الـ kernel */
#ifdef CONFIG_SECURITY_APPARMOR
    prop.apparmor.label = aa_get_label(label);
#endif
```

---

### خلاصة — الصورة الكاملة

الـ **LSM Framework** هو layer بين الـ kernel core وأي security implementation. بيوفر:
1. **Hook points** — نقاط تدخل معلومة ومحددة
2. **Security blobs** — مساحة private storage لكل LSM في كل kernel object
3. **Stacking** — تشغيل أكتر من LSM معاً بترتيب
4. **Static calls** — dispatch mechanism عالي الأداء بدون overhead الـ indirect calls

الـ **`lsm_prop_apparmor`** تحديداً هو الـ **interface struct** اللي بيسمح لأجزاء الـ kernel المختلفة تنقل الـ AppArmor security context (الـ `aa_label`) من غير ما تكون مرتبطة بتفاصيل الـ AppArmor الداخلية — مبدأ **information hiding** على مستوى الـ kernel.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### ملخص الملف

**الـ file** `include/linux/lsm/apparmor.h` صغير جداً — 17 سطر فقط — لكنه جزء من نظام أكبر هو **LSM prop system** اللي بيوفر interface موحد بين الـ kernel subsystems والـ LSM modules المختلفة. الملف نفسه بيعرّف struct واحد بس: `lsm_prop_apparmor`.

---

### ### 1. الـ Flags والـ Enums والـ Config Options

#### Config Options المتعلقة بالملف

| Option | النوع | الأثر |
|--------|-------|--------|
| `CONFIG_SECURITY_APPARMOR` | `bool` | لو مفعّل، الـ `label` field بيتضاف للـ struct. لو مش مفعّل، الـ struct فاضي تماماً |
| `CONFIG_SECURITY_SELINUX` | `bool` | بيتحكم في `lsm_prop_selinux.secid` — مجاور لكن مستقل |
| `CONFIG_SECURITY_SMACK` | `bool` | بيتحكم في `lsm_prop_smack.skp` — مجاور لكن مستقل |
| `CONFIG_BPF_LSM` | `bool` | بيتحكم في `lsm_prop_bpf.secid` — مجاور لكن مستقل |

#### الـ `label_flags` enum (من `security/apparmor/include/label.h`)

الـ `aa_label` اللي بيشير إليه الـ struct بيحمل flags من الـ enum ده:

| Flag | القيمة | المعنى |
|------|--------|---------|
| `FLAG_HAT` | `0x001` | الـ profile ده "hat" — نوع خاص من الـ sub-profile |
| `FLAG_UNCONFINED` | `0x002` | الـ label مش مقيّد (unconfined) — كل الـ profiles فيه unconfined |
| `FLAG_NULL` | `0x004` | الـ profile ده null learning profile |
| `FLAG_IX_ON_NAME_ERROR` | `0x008` | fallback لـ inherit-execute لو name lookup فشل |
| `FLAG_IMMUTIBLE` | `0x010` | ممنوع تغيير أو استبدال الـ label |
| `FLAG_USER_DEFINED` | `0x020` | profile بصلاحيات أقل (user-based) |
| `FLAG_NO_LIST_REF` | `0x040` | الـ list مش بتحتفظ بـ reference للـ profile |
| `FLAG_NS_COUNT` | `0x080` | بيحمل namespace reference count |
| `FLAG_IN_TREE` | `0x100` | الـ label موجود في الـ RB-tree |
| `FLAG_PROFILE` | `0x200` | الـ label ده profile (مش مجرد compound label) |
| `FLAG_EXPLICIT` | `0x400` | static label صريح |
| `FLAG_STALE` | `0x800` | الـ label اتاستبدل أو اتشال |
| `FLAG_RENAMED` | `0x1000` | فيه renaming جوا الـ label |
| `FLAG_REVOKED` | `0x2000` | فيه revocation جوا الـ label |
| `FLAG_DEBUG1/2` | `0x4000/0x8000` | للـ debugging |

#### Print Flags (لـ `aa_label_snxprint`)

| Flag | القيمة | المعنى |
|------|--------|---------|
| `FLAGS_NONE` | `0` | بدون خيارات |
| `FLAG_SHOW_MODE` | `1` | اعرض الـ mode مع الـ label |
| `FLAG_VIEW_SUBNS` | `2` | اعرض من منظور sub-namespace |
| `FLAG_HIDDEN_UNCONFINED` | `4` | خبّي الـ unconfined entries |
| `FLAG_ABS_ROOT` | `8` | استخدم absolute root في العرض |

---

### ### 2. الـ Structs المهمة

#### `struct lsm_prop_apparmor`

```c
/* from include/linux/lsm/apparmor.h */
struct lsm_prop_apparmor {
#ifdef CONFIG_SECURITY_APPARMOR
    struct aa_label *label;  /* pointer to the AppArmor label for this subject/object */
#endif
};
```

| Field | النوع | الغرض |
|-------|-------|--------|
| `label` | `struct aa_label *` | مؤشر للـ label اللي بيمثل هوية الـ process أو الـ object في AppArmor. بيكون `NULL` لو النظام مش مفعّل |

**الـ struct ده** بيظهر فقط كـ member في الـ `struct lsm_prop` الأكبر.

---

#### `struct lsm_prop` (من `include/linux/security.h`)

الـ container الحقيقي اللي بيجمع كل الـ LSM modules:

```c
struct lsm_prop {
    struct lsm_prop_selinux selinux;  /* SELinux: u32 secid */
    struct lsm_prop_smack   smack;    /* Smack: pointer to smack_known */
    struct lsm_prop_apparmor apparmor; /* AppArmor: pointer to aa_label */
    struct lsm_prop_bpf     bpf;      /* BPF LSM: u32 secid */
};
```

ده الـ struct اللي بيتبعت بين الـ kernel subsystems (audit, networking, IPC) والـ LSM layer.

---

#### `struct aa_label` (من `security/apparmor/include/label.h`)

الـ struct الأساسي اللي بيشير إليه `lsm_prop_apparmor.label`:

```c
struct aa_label {
    struct kref       count;    /* reference count — يُدار بـ kref_get/put */
    struct rb_node    node;     /* position in the labelset RB-tree */
    struct rcu_head   rcu;      /* for RCU-safe freeing */
    struct aa_proxy  *proxy;    /* إلو لو الـ label اتاستبدل */
    __counted char   *hname;    /* human-readable name (e.g. "/usr/bin/firefox") */
    long              flags;    /* label_flags بitmask */
    u32               secid;    /* numeric security ID for this label */
    int               size;     /* عدد الـ profiles في الـ vec[] */
    u64               mediates; /* bitmask: أي classes بيطبّق عليها الـ label */
    union {
        struct {
            struct aa_profile  *profile[2];         /* لو FLAG_PROFILE set */
            struct aa_ruleset **rules;               /* الـ rules المرتبطة */
        };
        struct aa_profile **vec;                     /* compound label vector */
    };
};
```

| Field | الغرض |
|-------|--------|
| `count` | reference counting آمن — الـ label بيتشال من الـ tree لما يوصل لـ 0 |
| `node` | بيخلي الـ label يتوجد في `aa_labelset.root` (RB-tree) |
| `rcu` | بيسمح بـ RCU-safe freeing بعد ما الـ refcount يوصل لـ 0 |
| `proxy` | لما label يتاستبدل، الـ proxy بيشير للـ label الجديد |
| `hname` | اسم human-readable زي `"/usr/sbin/cupsd"` |
| `flags` | من `label_flags` — بيحدد طبيعة الـ label |
| `secid` | رقم مرتبط بالـ label للاستخدام في الـ audit وكده |
| `size` | عدد الـ profiles في الـ compound label |
| `mediates` | bitmask بيحدد أي security classes (file, network, ...) بيطبّق عليها |
| `vec[]` | مصفوفة الـ profiles اللي بتكوّن الـ compound label |

---

#### `struct aa_proxy` (من `security/apparmor/include/label.h`)

```c
struct aa_proxy {
    struct kref         count;  /* reference count */
    struct aa_label __rcu *label; /* RCU pointer to the current/replacement label */
};
```

الـ proxy ده بيحل مشكلة الـ stale pointers: لما label يتاستبدل، أي حد محتفظ بـ pointer للـ proxy ممكن يوصل للـ label الجديد عبر الـ `label` field.

---

#### `struct aa_labelset` (من `security/apparmor/include/label.h`)

```c
struct aa_labelset {
    rwlock_t     lock;  /* protects the RB-tree */
    struct rb_root root; /* RB-tree of all labels in this namespace */
};
```

كل **namespace** في AppArmor عندها `labelset` واحد بيحتوي كل الـ labels الخاصة بيها.

---

#### `struct lsm_context` (من `include/linux/security.h`)

```c
struct lsm_context {
    char *context;  /* النص: e.g. "unconfined" أو "/usr/bin/firefox (enforce)" */
    u32   len;      /* طول الـ string */
    int   id;       /* رقم الـ LSM module اللي أنتجه */
};
```

ده الـ output لما حاجة زي `getxattr("security.apparmor")` تطلب الـ context كـ text.

---

### ### 3. مخطط العلاقات بين الـ Structs

```
include/linux/security.h
┌─────────────────────────────────────────────────────┐
│                   struct lsm_prop                   │
│  ┌──────────────────┐  ┌──────────────────────────┐ │
│  │ lsm_prop_selinux │  │   lsm_prop_apparmor      │ │
│  │  .secid: u32     │  │   .label ────────────────┼─┼──► struct aa_label
│  └──────────────────┘  └──────────────────────────┘ │        │
│  ┌──────────────────┐  ┌──────────────────────────┐ │        │ .proxy
│  │  lsm_prop_smack  │  │     lsm_prop_bpf         │ │        ▼
│  │  .skp: *smack_kn │  │     .secid: u32          │ │  struct aa_proxy
│  └──────────────────┘  └──────────────────────────┘ │    │ .label (RCU)
└─────────────────────────────────────────────────────┘    ▼
                                                     struct aa_label (new)

struct aa_label
├── .node ──────────────────────────────► aa_labelset.root (RB-tree)
├── .proxy ─────────────────────────────► struct aa_proxy
├── .vec[0..size-1] ────────────────────► struct aa_profile[]
│       └── .ns ────────────────────────► struct aa_ns
│                 └── .labels ──────────► struct aa_labelset
│                           └── .root ──► RB-tree (يحتوي .node)
└── .rules[] ───────────────────────────► struct aa_ruleset[]
```

---

### ### 4. دورة حياة الـ `aa_label`

#### Creation → Registration → Usage → Teardown

```
1. CREATION
   ──────────
   aa_label_alloc(size, proxy, gfp)
     │ kzalloc() لتخصيص الـ struct + vec[]
     │ kref_init(&label->count) = 1
     │ label->proxy = proxy (أو NULL)
     └─► aa_label_init(label, size, gfp)
           │ يحجز hname إذا لزم
           └─► label جاهز لكن خارج الـ tree

2. REGISTRATION
   ─────────────
   aa_label_insert(labelset, label)
     │ write_lock(&ls->lock)
     │ يبحث في RB-tree (بالـ hname)
     │ لو موجود: يرجع الموجود + يزوّد ref
     │ لو مش موجود: rb_insert_color()
     │              FLAG_IN_TREE يتعمل set
     └── write_unlock(&ls->lock)

3. USAGE
   ──────
   أي كود بيحتاج الـ label:
     aa_get_label(label)          // kref_get
       ↓
   يستخدم label->flags, vec[], mediates
       ↓
   aa_put_label(label)            // kref_put → aa_label_kref

4. REPLACEMENT (لو policy اتحدّثت)
   ──────────────────────────────
   aa_label_replace(old, new)
     │ write_lock(&ls->lock)
     │ يشيل old من الـ tree
     │ يضيف new للـ tree
     │ old->proxy->label = new    (RCU assign)
     │ FLAG_STALE يتعمل set على old
     └── write_unlock(&ls->lock)

5. TEARDOWN
   ─────────
   aa_put_label(label)
     │ kref_put(&label->count, aa_label_kref)
     │ لو count وصل 0:
     │   aa_label_remove(label)   // يشيله من الـ tree لو لسه فيه
     │   aa_label_destroy(label)  // يحرر hname وغيره
     └── kfree(label)             // عبر RCU callback
```

---

### ### 5. Call Flow: من الـ Process لحد الـ `lsm_prop_apparmor`

#### مثال: process بيعمل `open()` على ملف

```
sys_openat()
  │
  ▼
security_inode_permission(inode, mask)       [include/linux/security.h]
  │  بيكال كل الـ LSM hooks المسجّلة
  │
  ├──► apparmor_inode_permission()           [security/apparmor/file.c]
  │      │ بياخد الـ current task
  │      │ label = aa_get_newest_label(      [يجيب أحدث label من الـ proxy]
  │      │           aa_current_raw_label())
  │      │
  │      ├── label_for_each_confined(i, label, profile)
  │      │      aa_profile_file_perm(profile, file, mask)
  │      │        → يقارن الـ permission بالـ rules في profile
  │      │
  │      └── aa_put_label(label)
  │
  └──► [other LSM hooks: SELinux, Smack, ...]

إرجاع 0 (success) أو -EACCES
```

#### مثال: audit event بيسجّل هوية الـ subject

```
audit_log_start()
  │
  ▼
security_current_getlsmprop_subj(&prop)      [include/linux/security.h]
  │  بيملا struct lsm_prop بالبيانات من كل LSM
  │
  ├──► apparmor_current_getlsmprop_subj()    [security/apparmor/lsm.c]
  │      prop->apparmor.label =
  │        aa_get_newest_label(
  │          aa_current_raw_label())
  │
  ├──► selinux_current_getlsmprop_subj()
  │      prop->selinux.secid = ...
  │
  └──► [smack, bpf ...]

audit_log_subj_ctx(ab, &prop)
  │
  ▼
security_lsmprop_to_secctx(&prop, &ctx, ...)
  │
  ▼
apparmor_lsmprop_to_secctx()
  │  بيحوّل prop->apparmor.label لـ string
  │  زي "/usr/bin/firefox (enforce)"
  └──► aa_label_snxprint(str, size, ns, label, flags)
```

---

### ### 6. استراتيجية الـ Locking

#### الـ Locks الموجودة وحمايتها

| Lock | النوع | بيحمي إيه |
|------|-------|-----------|
| `aa_labelset.lock` | `rwlock_t` | الـ RB-tree من الـ labelset — قراءة/كتابة |
| `aa_label.count` | `kref` (atomic) | عدد الـ references — atomic بدون lock |
| `aa_proxy.label` | `__rcu` | مؤشر الـ label داخل الـ proxy — محمي بـ RCU |
| `task->cred` | `rcu_read_lock()` | بيانات الـ credentials — لازم تقراها تحت RCU |

#### قواعد الـ Lock Ordering

```
1. لو محتاج تعدّل الـ labelset:
   write_lock(&labelset->lock)
     ... تعديل الـ tree ...
   write_unlock(&labelset->lock)

2. لو بتقرا الـ proxy->label (بدون lock):
   rcu_read_lock()
     label = rcu_dereference(proxy->label)
     aa_get_label(label)   ← لازم قبل ما تطلع من RCU section
   rcu_read_unlock()

3. لو محتاج الـ label من الـ current task:
   rcu_read_lock()
     label = aa_current_raw_label()  ← من task->cred
     label = aa_get_newest_label(label)  ← عبر proxy
     aa_get_label(label)
   rcu_read_unlock()
   ... استخدام ...
   aa_put_label(label)

⚠️ ممنوع: تاخد labelset->lock وإنت شايل rcu_read_lock
⚠️ ممنوع: تعدّل aa_label fields بدون write_lock على الـ labelset
✔ مسموح: تقرا aa_label.flags تحت rcu_read_lock (atomic read)
```

#### مثال كود حقيقي للـ locking pattern

```c
/* get the newest version of the current task's label */
struct aa_label *label;

rcu_read_lock();
/* aa_current_raw_label() reads from task->cred under RCU */
label = aa_get_newest_label(aa_current_raw_label());
rcu_read_unlock();

/* now we hold a reference — safe to use outside RCU */
if (label_unconfined(label))
    goto allow;

/* ... check permissions ... */

aa_put_label(label);  /* release reference — may trigger kref callback */
```

---

### ### 7. الصورة الكاملة: مكانة `lsm_prop_apparmor` في النظام

```
┌─────────────────────────────────────────────────────────────────┐
│                     kernel subsystems                           │
│  [audit]  [networking]  [IPC]  [filesystem]  [binder]          │
└──────────────────┬──────────────────────────────────────────────┘
                   │ يستخدم struct lsm_prop
                   ▼
┌─────────────────────────────────────────────────────────────────┐
│               LSM core (security/security.c)                    │
│  security_*getlsmprop*() → بيكال كل LSM hooks في sequence      │
└──┬──────────────┬─────────────────┬────────────────┬────────────┘
   │              │                 │                │
   ▼              ▼                 ▼                ▼
[AppArmor]   [SELinux]          [Smack]          [BPF LSM]
fills:        fills:             fills:            fills:
.apparmor     .selinux           .smack            .bpf
 .label        .secid             .skp              .secid
   │
   ▼
struct aa_label
 ├── في labelset RB-tree (namespace-scoped)
 ├── بيشير لـ aa_profile[] (السياسة الفعلية)
 └── بيتوصّل عبر aa_proxy لو اتاستبدل
```

**الـ `lsm_prop_apparmor`** هو الجسر الوحيد بين الـ LSM prop system العام وبين عالم AppArmor الداخلي المعقد. حجمه صغير (مؤشر واحد أو zero bytes لو AppArmor مش مفعّل) لكن أهميته كبيرة: بيخلي أي kernel subsystem يقدر يسأل "مين الـ AppArmor label بتاع الـ subject/object ده؟" بطريقة موحدة ومستقلة عن implementation details الـ AppArmor الداخلية.
## Phase 4: شرح الـ Functions

> **ملاحظة:** الملف `include/linux/lsm/apparmor.h` لا يحتوي على أي functions أو APIs قابلة للاستدعاء. هو ملف **header-only** يعرّف struct واحدة فقط مستخدمة كـ glue بين الـ LSM framework والـ AppArmor subsystem.

---

### ملخص المحتوى

| العنصر | النوع | الغرض |
|--------|-------|--------|
| `struct aa_label` | forward declaration | الـ type الأساسي لـ AppArmor label |
| `struct lsm_prop_apparmor` | struct definition | حاوية الـ AppArmor label داخل الـ LSM property |

---

### تحليل الـ Struct

#### `struct lsm_prop_apparmor`

```c
struct lsm_prop_apparmor {
#ifdef CONFIG_SECURITY_APPARMOR
    struct aa_label *label; /* pointer to the task's AppArmor label */
#endif
};
```

الـ struct دي هي الـ **per-LSM slot** اللي بيخزن فيها الـ AppArmor security context الخاص بـ task أو object معين. الـ LSM framework بيستخدم union أو embedded struct من النوع ده لكل LSM مسجّل في الـ kernel، عشان يعزل الـ security data الخاص بكل LSM عن التاني.

**الـ `#ifdef CONFIG_SECURITY_APPARMOR`** معناها إن لو AppArmor مش مفعّل في الـ kernel config، الـ struct بتبقى فاضية تماماً — zero-size struct — ومفيش overhead على الـ memory.

**الـ `struct aa_label *label`** هو pointer للـ `aa_label` اللي بيمثل الـ security profile المرتبط بالـ task أو الـ object. الـ `aa_label` نفسها معرّفة في `security/apparmor/include/label.h` وبتحتوي على مجموعة من الـ profiles (الـ `aa_profile`) مع reference counting.

---

### الـ Forward Declaration

```c
struct aa_label;
```

ده **incomplete type declaration** بيسمح للـ header إنه يستخدم `struct aa_label *` (pointer) من غير ما يحتاج يـ include الـ full definition. ده pattern مهم في الـ kernel لتقليل الـ compile-time dependencies وتجنب الـ circular includes.

---

### ليه مفيش Functions هنا؟

الـ header ده جزء من نظام **LSM stacking** المُدخَل في الـ kernel من kernel 4.15+ وتطوّر في 6.x. الفلسفة إن:

- كل LSM بيعرّف **data struct** خاصة بيه في `include/linux/lsm/<lsm_name>.h`.
- الـ LSM framework بيجمع الـ structs دي في `struct lsm_prop` أو `struct security_blob_sizes` إلخ.
- الـ actual API (دوال زي `apparmor_task_alloc`, `apparmor_file_permission`, إلخ) موجودة في `security/apparmor/` وبتتسجّل عبر `struct security_hook_list`.

يعني الملف ده **data-only interface** — مش API header.
## Phase 5: دليل الـ Debugging الشامل

الـ file الأساسي `include/linux/lsm/apparmor.h` بيعرّف الـ `lsm_prop_apparmor` struct اللي بتحمل pointer لـ `aa_label` — ده الـ bridge بين الـ LSM framework العام وبين الـ AppArmor subsystem. الـ debugging هنا بيشمل مستويين: الـ LSM framework نفسه والـ AppArmor policy engine.

---

### Software Level

#### 1. Debugfs Entries

الـ AppArmor بيكشف entries كتير في `/sys/kernel/debug/apparmor/`:

```bash
# اقرأ الـ profiles المحمّلة
cat /sys/kernel/debug/apparmor/profiles

# اقرأ حالة الـ label cache
cat /sys/kernel/debug/apparmor/label_stats

# اقرأ إحصائيات الـ policy
cat /sys/kernel/debug/apparmor/policy_stats
```

مثال output:
```
/usr/bin/firefox (enforce)
/usr/sbin/cups-browsed (complain)
unconfined
```

**تفسيره:** كل سطر = profile اسمه + mode (enforce/complain/unconfined).

```bash
# قائمة كل الـ namespaces
ls /sys/kernel/debug/apparmor/namespaces/

# اقرأ الـ label لـ process معين
cat /sys/kernel/debug/apparmor/namespaces/default/profiles
```

#### 2. Sysfs Entries

```bash
# تحقق من تفعيل AppArmor
cat /sys/module/apparmor/parameters/enabled
# output: Y = مفعّل

# اقرأ الـ mode الحالي
cat /sys/kernel/security/apparmor/enforce
cat /sys/kernel/security/apparmor/.access

# أهم entries
ls /sys/kernel/security/apparmor/
```

| Entry | الوظيفة |
|-------|---------|
| `/sys/kernel/security/apparmor/profiles` | قائمة الـ profiles المحمّلة |
| `/sys/kernel/security/apparmor/.remove` | تحميل/إزالة profiles |
| `/sys/kernel/security/apparmor/.replace` | استبدال profile |
| `/sys/kernel/security/apparmor/policy/` | الـ policy tree كاملة |
| `/sys/kernel/security/apparmor/features` | الـ features المدعومة |

```bash
# اقرأ الـ aa_label الخاص بـ process معين عبر procfs
cat /proc/<PID>/attr/current         # AppArmor label
cat /proc/<PID>/attr/apparmor/current
cat /proc/<PID>/attr/apparmor/prev   # السابق (بعد exec)
```

#### 3. Ftrace — Tracepoints والـ Events

```bash
# فعّل الـ LSM tracepoints
echo 1 > /sys/kernel/debug/tracing/events/lsm/enable

# AppArmor-specific events (kernel 6.x+)
ls /sys/kernel/debug/tracing/events/apparmor/

# فعّل كل الـ AppArmor events
echo 1 > /sys/kernel/debug/tracing/events/apparmor/enable

# trace hook بعينه — مثلاً file_permission
echo 1 > /sys/kernel/debug/tracing/events/apparmor/apparmor_file_permission/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

**لـ trace الـ label lookup بالذات:**
```bash
# استخدم function tracer على aa_label functions
echo 'aa_label*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

مثال output:
```
 bash-1234  [001] .... 12345.678: aa_label_parse <-apparmor_setprocattr
 bash-1234  [001] .... 12345.679: aa_get_label <-aa_label_parse
```

#### 4. Printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لكل ملفات AppArmor
echo 'file security/apparmor/* +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل لملف بعينه
echo 'file security/apparmor/label.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع line numbers وfunction names
echo 'file security/apparmor/label.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug المفعّلة حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep apparmor
```

**الـ AppArmor بيستخدم `AA_DEBUG` macro داخلياً:**
```c
/* من security/apparmor/include/lib.h */
#define AA_DEBUG(fmt, args...) \
    pr_debug_ratelimited("AppArmor: " fmt, ##args)
```

لرؤية هذه الرسائل:
```bash
# تأكد إن kernel compiled مع CONFIG_DYNAMIC_DEBUG
grep CONFIG_DYNAMIC_DEBUG /boot/config-$(uname -r)

# أو عبر dmesg
dmesg -w | grep -i apparmor
```

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_SECURITY_APPARMOR` | تفعيل AppArmor أصلاً |
| `CONFIG_SECURITY_APPARMOR_DEBUG` | تفعيل الـ debug messages |
| `CONFIG_SECURITY_APPARMOR_DEBUG_ASSERTS` | تفعيل الـ assertions الداخلية |
| `CONFIG_SECURITY_APPARMOR_HASH` | هاش الـ policy للتحقق |
| `CONFIG_SECURITY_APPARMOR_HASH_DEFAULT` | هاش افتراضي |
| `CONFIG_AUDIT` | لازم مفعّل عشان audit logs |
| `CONFIG_SECURITY_LSM_DEBUG` | debugging للـ LSM framework نفسه |
| `CONFIG_DYNAMIC_DEBUG` | لـ dynamic debug messages |
| `CONFIG_KPROBES` | لـ kprobe على aa_* functions |

```bash
# تحقق من الـ configs المحمّلة
zcat /proc/config.gz | grep -E 'APPARMOR|LSM_DEBUG'
```

#### 6. أدوات خاصة بالـ AppArmor

```bash
# aa-status: يعرض كل الـ profiles والـ processes
aa-status

# aa-audit: يعرض الـ audit log مفسّراً
journalctl -k | grep apparmor

# aa-notify: يراقب الـ denials real-time
aa-notify -s 1 -v

# apparmor_parser: تحليل الـ profile syntax
apparmor_parser --debug /etc/apparmor.d/usr.bin.firefox

# dump الـ label بـ python/strace
strace -e trace=open,read -p <PID> 2>&1 | grep attr

# aa-easyprof: لبناء profiles للـ debugging
aa-easyprof --template=default /usr/bin/myapp
```

**للتحقق من الـ `lsm_prop_apparmor` في الـ process:**
```bash
# الـ label المرتبط بـ process
cat /proc/self/attr/apparmor/current

# كل الـ attr الخاصة بـ AppArmor
ls /proc/self/attr/apparmor/
```

#### 7. رسائل الأخطاء الشائعة

| رسالة في الـ log | المعنى | الحل |
|-----------------|--------|------|
| `apparmor: DENIED operation=...` | الـ profile رفض عملية | راجع الـ profile أو حوّله لـ complain mode |
| `apparmor: audit: type=1400 ... apparmor="DENIED"` | denial مسجّل في audit | `ausearch -m AVC \| aa-decode` |
| `AppArmor: label allocation failed` | نفدت الـ memory للـ aa_label | راجع الـ memory وراجع الـ label cache leak |
| `AppArmor: invalid label string` | format الـ label غلط | تحقق من `/proc/<PID>/attr/apparmor/current` |
| `AppArmor: policy load error` | خطأ في تحميل الـ policy | `apparmor_parser -p /etc/apparmor.d/<profile>` |
| `apparmor: profile not found` | الـ process بيطلب profile مش موجود | حمّل الـ profile أو راجع الـ namespace |
| `apparmor: unable to replace profile` | conflict في الـ profile | `aa-status` وتحقق من الـ profile المتعارض |
| `LSM: Unable to initialize AppArmor` | فشل الـ initialization | تحقق من `CONFIG_SECURITY_APPARMOR` وترتيب الـ LSMs |
| `AppArmor: namespace error` | خطأ في الـ label namespace | راجع `/sys/kernel/security/apparmor/policy/namespaces/` |

#### 8. Strategic Points لـ `dump_stack()` و`WARN_ON()`

**النقاط الاستراتيجية في الـ AppArmor/LSM code:**

```c
/* في security/apparmor/label.c — عند allocate الـ aa_label */
struct aa_label *aa_label_alloc(int size, struct aa_proxy *proxy, gfp_t gfp)
{
    struct aa_label *new;
    new = kzalloc(sizeof(*new) + size * sizeof(struct aa_profile *), gfp);
    WARN_ON(!new);   /* <-- هنا لو الـ allocation فشل */
    ...
}

/* في security/apparmor/domain.c — عند الـ label transition */
int apparmor_bprm_creds_for_exec(struct linux_binprm *bprm)
{
    struct aa_label *label = aa_get_newest_label(aa_current_raw_label());
    WARN_ON(!label);  /* <-- تحقق إن الـ label مش NULL قبل الاستخدام */
    ...
}

/* في security/apparmor/lsm.c — في الـ hook نفسه */
static int apparmor_file_permission(struct file *file, int mask)
{
    struct aa_label *label;
    label = __begin_current_label_crit_section();
    if (WARN_ON(!label)) {  /* <-- الـ label المرتبط بـ lsm_prop_apparmor */
        dump_stack();
        return -EINVAL;
    }
    ...
}
```

**نقاط مهمة للـ `WARN_ON`:**
- عند dereferencing `lsm_prop_apparmor.label` لو `CONFIG_SECURITY_APPARMOR` مفعّل
- عند `aa_label` reference count يوصل صفر قبل الأوان
- عند mismatch بين الـ task's label والـ expected namespace

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يتطابق مع الـ Kernel State

الـ AppArmor هو pure software LSM — مفيش hardware مباشر. لكن في سيناريوهات مهمة:

```bash
# تأكد إن الـ CPU يدعم الـ features المطلوبة للـ LSM
grep -E 'smep|smap|pti' /proc/cpuinfo

# تأكد إن الـ secure boot مش بيعطّل AppArmor loading
mokutil --sb-state

# لو بتشتغل على ARM مع TrustZone
# تحقق إن الـ secure world مش بيتعارض مع الـ LSM hooks
dmesg | grep -i 'trustzone\|tee\|optee'

# على أنظمة virtualization — تأكد إن الـ hypervisor مش بيمنع LSM
dmesg | grep -i 'kvm\|vmx\|svm' | head -5
```

#### 2. Register Dump Techniques

الـ AppArmor مش بيتعامل مع hardware registers مباشرة، لكن ممكن نحتاج نتحقق من:

```bash
# قراءة MSR registers المتعلقة بالـ security (x86)
# SMEP/SMAP bits في CR4
# لازم kernel module أو rdmsr tool

# تثبيت msr-tools
apt-get install msr-tools

# اقرأ CR4 عبر MSR (على x86)
rdmsr 0x3A    # IA32_FEATURE_CONTROL

# تحقق من الـ memory layout للـ security blobs
# عبر /proc/iomem
cat /proc/iomem | grep -i kernel
```

**لقراءة الـ aa_label struct في الـ memory عبر gdb/crash:**
```bash
# استخدم crash utility مع vmcore
crash /usr/lib/debug/boot/vmlinux-$(uname -r) /var/crash/vmcore

# داخل crash:
# > task <PID>
# > struct task_struct <addr>
# > p ((struct task_security_struct *)<cred_security_addr>)->label
```

#### 3. Logic Analyzer / Oscilloscope Tips

الـ AppArmor كـ software LSM مش بيحتاج logic analyzer مباشرة. لكن لو بتـ debug الـ interaction مع hardware-backed security:

- **ARM TrustZone + AppArmor:** استخدم JTAG لمراقبة الـ transitions بين EL0/EL1/EL3
- **TPM + AppArmor:** راقب الـ TPM bus (I2C/SPI) لو بيستخدم AppArmor مع IMA/TPM
- **HSM (Hardware Security Module):** راقب الـ PCIe transactions لو الـ key management مرتبط بـ AppArmor labels

```
Signal Analysis Points:
┌─────────────────────────────────────────────┐
│  User Space Process                         │
│       │ syscall                             │
│       ▼                                     │
│  Kernel Entry (System Call Table)           │
│       │                                     │
│       ▼                                     │
│  LSM Hook Dispatch                          │
│       │ calls apparmor_file_permission()    │
│       ▼                                     │
│  aa_label lookup (lsm_prop_apparmor.label)  │
│       │                                     │
│       ▼                                     │
│  Policy Engine Decision (allow/deny)        │
└─────────────────────────────────────────────┘
```

#### 4. Hardware Issues → Kernel Log Patterns

| المشكلة | Pattern في الـ Log |
|---------|-------------------|
| Memory corruption في الـ aa_label | `BUG: KASAN: use-after-free in aa_label_parse` |
| Stack overflow في الـ LSM hook chain | `BUG: kernel stack overflow` في call stack يحتوي `apparmor_` |
| Race condition في الـ label refcount | `WARNING: CPU: X PID: Y at security/apparmor/label.c:Z` |
| OOM أثناء label allocation | `apparmor: label allocation failed` + `oom_kill` |
| Firmware/Secure Boot تعارض | `AppArmor: unable to open policy` بعد `secure_boot: enabled` |

```bash
# فلتر كل الـ AppArmor related kernel messages
dmesg --level=err,warn | grep -i apparmor
journalctl -k --since "1 hour ago" | grep -i 'apparmor\|lsm'

# تحقق من KASAN/UBSAN errors مرتبطة بـ AppArmor
dmesg | grep -A 10 'KASAN.*apparmor\|apparmor.*KASAN'
```

#### 5. Device Tree Debugging

الـ AppArmor نفسه مش مرتبط بـ Device Tree مباشرة، لكن:

```bash
# لو الـ AppArmor مش بيشتغل على embedded ARM
# تحقق من الـ DT bootargs
cat /proc/device-tree/chosen/bootargs | tr '\0' '\n' | grep -E 'security|apparmor|lsm'

# مثال bootargs صح:
# security=apparmor lsm=landlock,lockdown,yama,integrity,apparmor

# تحقق من الـ kernel command line
cat /proc/cmdline

# تحقق إن الـ LSM order صح
cat /sys/kernel/security/lsm
# output: lockdown,capability,landlock,yama,apparmor
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**1. فحص سريع لحالة AppArmor:**
```bash
#!/bin/bash
echo "=== AppArmor Status ==="
aa-status 2>/dev/null || cat /sys/module/apparmor/parameters/enabled

echo -e "\n=== Loaded LSMs ==="
cat /sys/kernel/security/lsm

echo -e "\n=== Current Process Label ==="
cat /proc/self/attr/apparmor/current

echo -e "\n=== Recent Denials ==="
journalctl -k --since "1 hour ago" | grep 'apparmor.*DENIED' | tail -20
```

**2. Debug label لـ process معين:**
```bash
PID=1234
echo "Label for PID $PID:"
cat /proc/$PID/attr/apparmor/current
echo "Previous label:"
cat /proc/$PID/attr/apparmor/prev 2>/dev/null || echo "N/A"
echo "Task name: $(cat /proc/$PID/comm)"
```

**3. تفعيل full AppArmor debug tracing:**
```bash
#!/bin/bash
# فعّل dynamic debug
echo 'file security/apparmor/* +pflmt' > /sys/kernel/debug/dynamic_debug/control

# فعّل ftrace
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'aa_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# سجّل لـ 5 ثواني
sleep 5
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace > /tmp/apparmor_trace.txt
echo "Trace saved to /tmp/apparmor_trace.txt"
wc -l /tmp/apparmor_trace.txt
```

**4. مراقبة الـ denials real-time:**
```bash
# طريقة 1: auditd
ausearch -m AVC,USER_AVC -i --raw | grep apparmor | tail -f

# طريقة 2: journalctl
journalctl -kf | grep -i 'apparmor.*denied\|apparmor.*audit'

# طريقة 3: dmesg مع filter
dmesg -w | grep --line-buffered 'apparmor'
```

**5. تحليل denial وتوليد الـ fix:**
```bash
# خذ الـ denial message
DENIAL=$(journalctl -k --since "5 min ago" | grep 'apparmor.*DENIED' | head -1)
echo "Denial: $DENIAL"

# استخدم aa-logprof لتحليل تلقائي
aa-logprof

# أو aa-genprof لـ profile جديد
aa-genprof /usr/bin/myapp
```

**6. التحقق من الـ `lsm_prop_apparmor` struct size:**
```bash
# عبر kernel symbols
cat /proc/kallsyms | grep aa_label | head -10

# أو عبر crash/gdb
crash /usr/lib/debug/boot/vmlinux-$(uname -r) /proc/kcore <<'EOF'
p sizeof(struct lsm_prop_apparmor)
p sizeof(struct aa_label)
EOF
```

**7. فحص الـ label cache statistics:**
```bash
cat /sys/kernel/debug/apparmor/label_stats
# output مثال:
# total: 42
# proxy: 40
# stale: 2
# longest: 3 (لأطول label chain)
```

**تفسير الـ output:**
- `total`: عدد الـ aa_label objects الموجودة في الـ cache
- `proxy`: labels مرتبطة بـ proxy (للـ atomic profile replacement)
- `stale`: labels قديمة لسه في الـ cache (pending cleanup)
- `longest`: أعمق label في الـ inheritance chain — لو كبير جداً = مشكلة في الـ label نesting

**8. اختبار policy بدون تطبيقها (dry-run):**
```bash
# parse الـ profile وابحث عن أخطاء
apparmor_parser -p /etc/apparmor.d/usr.bin.firefox

# قارن profile جديد بالقديم
apparmor_parser -K -T /etc/apparmor.d/usr.bin.firefox

# حمّل الـ profile في complain mode للـ debugging
aa-complain /etc/apparmor.d/usr.bin.firefox
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — AppArmor بيمنع daemon من الكتابة على /dev/ttyS0

#### العنوان
**AppArmor label مش متحدد على process بيحاول يكتب على UART في gateway صناعي**

#### السياق
شركة بتعمل industrial IoT gateway على Texas Instruments AM62x. الـ gateway بيشغّل Yocto Linux مع AppArmor مفعّل. في production، فيه daemon اسمه `sensor-collector` بيقرا بيانات من حساسات عبر `/dev/ttyS1` (UART).

#### المشكلة
بعد kernel upgrade من 5.15 لـ 6.6، الـ `sensor-collector` بدأ يطلع error في syslog:

```
kernel: audit: type=1400 audit(1700000000.123:45): apparmor="DENIED"
        operation="open" profile="sensor-collector"
        name="/dev/ttyS1" pid=1234 comm="sensor-collector"
        requested_mask="rw" denied_mask="rw"
```

الـ daemon موقّف تماماً. المنتج في field بيتعطل.

#### التحليل
الـ file الي بندرسه هو `include/linux/lsm/apparmor.h`:

```c
struct lsm_prop_apparmor {
#ifdef CONFIG_SECURITY_APPARMOR
    struct aa_label *label;  /* pointer to the AppArmor label for this task */
#endif
};
```

لما الـ kernel بيعمل security check على `open("/dev/ttyS1")`:

1. الـ kernel بيجيب الـ `lsm_prop_apparmor` من الـ `task_struct` عبر `lsm_prop`.
2. بيشوف الـ `aa_label *label` — لو الـ pointer ده بيشاور على label مش مضيالوش إذن `/dev/ttyS1`، الـ AppArmor بيرفض العملية.
3. في الـ upgrade الجديد، الـ policy loading بيحصل بعد ما الـ daemon بيتشغّل، فالـ process بيتلاقيه مش ليه label (أو ليه label `unconfined` بس الـ profile اتغير).

الـ struct بسيطة — مجرد pointer واحد — بس غيابه أو إنه يشاور على label خاطئ بيوقف كل حاجة.

#### الحل

**1. تشخيص الـ label الحالي للـ process:**
```bash
# شوف الـ label بتاع الـ daemon
cat /proc/$(pgrep sensor-collector)/attr/current
# لو طلع: "unconfined" أو فاضي → فيه مشكلة في profile loading
```

**2. تأكيد إن الـ AppArmor profile اتحمّل صح:**
```bash
aa-status | grep sensor-collector
# لو مش موجود → الـ profile مش loaded
```

**3. تحميل الـ profile يدوياً:**
```bash
apparmor_parser -r /etc/apparmor.d/usr.sbin.sensor-collector
```

**4. الـ profile المطلوب (أضف UART permission):**
```
/usr/sbin/sensor-collector {
    /dev/ttyS1 rw,
    /dev/ttyS[0-9]* rw,
    /var/log/sensor.log w,
}
```

**5. في Yocto، تأكيد إن الـ AppArmor profiles بتتحمّل قبل الـ daemon في systemd:**
```ini
# sensor-collector.service
[Unit]
After=apparmor.service
Requires=apparmor.service
```

#### الدرس المستفاد
الـ `lsm_prop_apparmor` بيحتوي على pointer واحد بس (`aa_label *label`) — لو الـ label ده NULL أو غلط في وقت الـ syscall، كل العمليات بترفض. في embedded systems، لازم تضمن إن الـ AppArmor profiles بتتحمّل **قبل** ما الـ daemons تتشغّل، وإلا الـ label مش بيتعيّن صح.

---

### السيناريو 2: Android TV Box على RK3562 — AppArmor مش compile وسبب kernel panic

#### العنوان
**`CONFIG_SECURITY_APPARMOR` مش مفعّل وكود بيافترض وجود الـ `label` pointer**

#### السياق
فريق بيعمل Android TV box على Rockchip RK3562. بيستخدموا custom kernel مبني على AOSP kernel مع تعديلات. بيحاولوا يضيفوا LSM custom بيتكلم مع AppArmor عبر الـ `lsm_prop_apparmor`.

#### المشكلة
الـ engineer كتب كود في الـ custom LSM بيعمل access مباشر لـ `lsm_prop_apparmor.label` من غير ما يشيك على الـ `#ifdef`:

```c
/* في custom LSM — كود غلط */
static int my_lsm_check(struct task_struct *task)
{
    struct lsm_prop prop;
    security_task_getsecid(task, &prop);

    /* هنا بيوصل struct lsm_prop_apparmor */
    struct aa_label *lbl = prop.lsm[apparmor_blob_sizes.lbs_task].apparmor.label;
    if (lbl) { /* ... */ }
    return 0;
}
```

لما بنى الـ kernel بدون `CONFIG_SECURITY_APPARMOR`، الـ build فضل بيشتغل بس بعدين kernel panic عند runtime لأن الـ struct size اتغيرت.

#### التحليل
الـ header بيوضح المشكلة صراحةً:

```c
struct lsm_prop_apparmor {
#ifdef CONFIG_SECURITY_APPARMOR
    struct aa_label *label;  /* field ده موجود بس لو AppArmor compiled in */
#endif
};
```

لو `CONFIG_SECURITY_APPARMOR` مش defined:
- الـ `struct lsm_prop_apparmor` بتبقى **فاضية تماماً** — size = 0.
- أي code بيحاول يوصل لـ `.label` بيعمل access لـ garbage memory.
- الـ `lsm_prop` الكلي بيتغير في الـ size، فـ offsets كلها بتتغير.
- النتيجة: kernel panic أو silent memory corruption.

#### الحل

**الكود الصح — لازم دايماً تحوّط بـ ifdef:**

```c
/* custom LSM — الطريقة الصح */
static int my_lsm_check(struct task_struct *task)
{
    struct lsm_prop prop;
    security_task_getsecid(task, &prop);

#ifdef CONFIG_SECURITY_APPARMOR
    /* آمن نوصل للـ label بس لو AppArmor compiled in */
    struct aa_label *lbl = prop.lsm[apparmor_blob_sizes.lbs_task].apparmor.label;
    if (lbl) {
        /* process the label */
    }
#endif
    return 0;
}
```

**وتأكيد في Kconfig:**
```kconfig
config MY_CUSTOM_LSM
    bool "My Custom LSM"
    depends on SECURITY_APPARMOR
    help
      This LSM requires AppArmor to be enabled.
```

**للـ RK3562 في defconfig:**
```
CONFIG_SECURITY_APPARMOR=y
CONFIG_MY_CUSTOM_LSM=y
```

#### الدرس المستفاد
الـ `#ifdef CONFIG_SECURITY_APPARMOR` في `lsm_prop_apparmor` مش مجرد تزويق — ده guard حقيقي بيغير الـ ABI. أي code بياعمل access لـ `.label` لازم يكون محاط بنفس الـ `#ifdef`، وإلا تعملت مشكلة silent corruption أو build error في configurations تانية.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — memory overhead من AppArmor label في كل task

#### العنوان
**`aa_label *` في كل task بيرفع memory usage في نظام محدود الـ RAM**

#### السياق
منتج IoT بيشغّل Linux على STM32MP1 (Cortex-A7، 256MB RAM). النظام بيشغّل مئات الـ threads لمعالجة بيانات حساسات. الـ product manager بيطلب تفعيل AppArmor للـ security hardening.

#### المشكلة
بعد تفعيل `CONFIG_SECURITY_APPARMOR=y`، الـ engineer لاحظ زيادة في memory usage:

```bash
# قبل AppArmor
free -m
# Mem: 256MB total, 180MB used, 76MB free

# بعد AppArmor
free -m
# Mem: 256MB total, 218MB used, 38MB free
```

الفرق حوالي 38MB مع 500 thread نشط. الـ system بدأ يعمل OOM kills.

#### التحليل
كل task في الـ kernel بيحمل `lsm_prop` في جزء من الـ `task_struct` أو في الـ security blob. الـ struct هي:

```c
struct lsm_prop_apparmor {
#ifdef CONFIG_SECURITY_APPARMOR
    struct aa_label *label;  /* 8 bytes على 64-bit، pointer فقط */
#endif
};
```

الـ pointer نفسه 8 bytes بس، لكن الـ `aa_label` الي بيشاور عليه بيحتوي على:
- Reference counts
- RCU structures
- `aa_vecinfo` + vector of `aa_profile *` pointers
- String data للـ label name

كل label shared بين processes بنفس الـ profile — بس لو عندك 500 thread كلهم **`unconfined`** بشكل مختلف أو profile مختلف، الـ labels بتتكاثر.

```bash
# شوف كام label موجود
cat /sys/kernel/security/apparmor/profiles | wc -l

# شوف memory الـ AppArmor بيستخدمه
grep -i apparmor /proc/slabinfo
```

#### الحل

**1. استخدام label واحد مشترك لكل threads بنفس الـ policy:**
```bash
# كل threads الـ sensor بيشتغلوا تحت profile واحد
# → label واحد مشترك بدل 500 label
```

**2. في الـ AppArmor policy، استخدام hat أو child profile بدل profiles منفصلة:**
```
/usr/bin/sensor-master {
    ^child_thread {
        /dev/i2c-* rw,
        /sys/bus/i2c/** r,
    }
}
```

**3. لو الـ memory constraint صعب جداً — disable AppArmor وبدّله بـ lighter LSM:**
```
# في kernel config لـ STM32MP1
CONFIG_SECURITY_APPARMOR=n
CONFIG_SECURITY_YAMA=y   # lightweight LSM
```

**4. قياس الـ label memory بـ slab:**
```bash
cat /proc/slabinfo | grep aa_
# aa_label   500   500   256   ...
# كل label حوالي 256 bytes → 500 × 256 = 128KB للـ labels وحدها
```

#### الدرس المستفاد
الـ `aa_label *` في `lsm_prop_apparmor` هو pointer خفيف (8 bytes)، لكن الـ `aa_label` نفسه بيستهلك memory حقيقية. في أنظمة بمئات الـ threads، لازم تتأكد إن الـ labels متشاركة (shared) قدر الإمكان عبر profiles مصممة كويس، مش كل process بـ label منفرد.

---

### السيناريو 4: Automotive ECU على i.MX8 — AppArmor label corruption بسبب Use-After-Free

#### العنوان
**الـ `aa_label *label` pointer بيتقرأ بعد ما الـ label اتحرر — kernel crash في ECU**

#### السياق
شركة automotive بتعمل ECU (Electronic Control Unit) على NXP i.MX8QM بيشغّل Linux مع PREEMPT_RT patch للـ real-time. النظام بيستخدم AppArmor لعزل processes الـ safety-critical. المنتج في validation stage.

#### المشكلة
الـ system بيعمل kernel panic متكرر في stress tests:

```
BUG: KASAN: use-after-free in aa_label_printxattr+0x48/0xc0
Read of size 8 at addr ffff888012345678 by task ecu-monitor/1234
...
Freed by task kworker/0:2:
    aa_label_destroy+0x30/0x80
    aa_put_label+0x1c/0x30
```

#### التحليل
الـ `lsm_prop_apparmor` بيشيل pointer:

```c
struct lsm_prop_apparmor {
#ifdef CONFIG_SECURITY_APPARMOR
    struct aa_label *label;  /* raw pointer — يحتاج reference counting صح */
#endif
};
```

الـ `aa_label` بيستخدم **reference counting** للـ memory management. الـ flow الصح:

```
task creation  →  aa_get_label(label)  →  label->count++
task exit      →  aa_put_label(label)  →  label->count--  →  free if 0
```

في الـ PREEMPT_RT environment، race condition ممكن يحصل:

```
Thread A (ecu-monitor):         Thread B (kworker):
read label pointer          →
                            →   aa_put_label() → count = 0
                            →   aa_label_destroy() → kfree(label)
dereference label pointer   →   USE-AFTER-FREE!
```

المشكلة إن كود بييجي بيقرأ الـ `lsm_prop_apparmor.label` من غير ما يعمل `aa_get_label()` الأول (زيادة الـ refcount).

#### الحل

**الكود الصح — دايماً استخدم RCU أو get/put:**

```c
/* الطريقة الآمنة لقراءة الـ label */
static void safe_read_label(struct task_struct *task)
{
    struct aa_label *label;

    rcu_read_lock();
    /* aa_get_current_label تعمل get_label داخلياً */
    label = aa_get_task_label(task);
    if (label) {
        /* استخدم الـ label هنا بأمان */
        // ... do work ...
        aa_put_label(label);  /* لازم put بعد الاستخدام */
    }
    rcu_read_unlock();
}
```

**لتشخيص المشكلة:**
```bash
# فعّل KASAN في kernel config للـ i.MX8
CONFIG_KASAN=y
CONFIG_KASAN_INLINE=y
CONFIG_PREEMPT_RT=y

# شغّل stress test وراقب
dmesg | grep -i "use-after-free"
```

**في الـ device tree للـ i.MX8QM، مفيش تغيير مطلوب — المشكلة في الكود بس.**

#### الدرس المستفاد
الـ `aa_label *label` في `lsm_prop_apparmor` هو raw pointer لـ reference-counted object. في أي كود بييجي بيوصل للـ label، لازم **دايماً** يستخدم `aa_get_label()` قبل الاستخدام و`aa_put_label()` بعده — خصوصاً في PREEMPT_RT kernels حيث الـ race windows أكبر.

---

### السيناريو 5: Custom Board Bring-Up على Allwinner H616 — AppArmor مش شغّال وكل processes بـ "unconfined"

#### العنوان
**board جديد بيشغّل Android TV OS — AppArmor موجود في kernel لكن مش active**

#### السياق
فريق bring-up بيشتغل على board جديد مبني على Allwinner H616 (Android TV box). القرار إن الـ security team اشترط AppArmor enforcement قبل الـ mass production. الـ `CONFIG_SECURITY_APPARMOR=y` موجود في الـ defconfig، لكن بعد الـ boot كل processes بتبقى `unconfined`.

#### المشكلة
```bash
aa-status
# AppArmor module is loaded.
# 0 profiles are loaded.
# 0 processes have profiles defined.
# 0 processes are in enforce mode.
# 500 processes are unconfined.
```

الـ `lsm_prop_apparmor` موجود في كل task، بس الـ `label` بيشاور على `unconfined` label طول الوقت.

#### التحليل
الـ struct:

```c
struct lsm_prop_apparmor {
#ifdef CONFIG_SECURITY_APPARMOR
    struct aa_label *label;  /* initialized to "unconfined" label at task creation */
#endif
};
```

لما task بيتخلق، الـ kernel بيعمل:
```c
/* في apparmor/lsm.c - apparmor_task_alloc() */
new->label = aa_get_newest_label(&root_ns->unconfined->label);
```

يعني كل process بيبدأ بـ `unconfined` label. الـ transition لـ confined label بيحصل عبر:
1. **exec** مع profile موجود (الطريقة الرئيسية)
2. `aa_change_profile()` يدوياً

المشكلة: الـ AppArmor profiles مش بتتحمّل في الـ boot. السبب في الـ H616 إن الـ init system (Android init) مش بيشغّل `apparmor_parser` أو بيشغّله قبل ما الـ `/sys/kernel/security/apparmor` يكون ready.

#### الحل

**1. تشخيص:**
```bash
# تأكيد إن AppArmor LSM active في kernel
cat /sys/kernel/security/lsm
# المفروض يظهر: lockdown,yama,apparmor,bpf

# تأكيد إن الـ filesystem mounted
ls /sys/kernel/security/apparmor/
# profiles  policy  features  ...

# شوف الـ boot cmdline
cat /proc/cmdline | grep -o 'security=[^ ]*'
# لازم يكون security=apparmor أو LSM=apparmor
```

**2. في الـ bootloader (U-Boot للـ H616)، أضف kernel param:**
```bash
# في /boot/uEnv.txt أو U-Boot env
setenv bootargs "... security=apparmor lsm=landlock,lockdown,yama,apparmor,bpf"
```

**3. تحميل profiles مبكراً في Android init:**
```
# في init.rc
on early-boot
    exec_background /sbin/apparmor_parser --replace /etc/apparmor.d/
```

**4. التحقق إن label اتعيّن صح بعد exec:**
```bash
# بعد تشغيل process
cat /proc/$(pgrep media.codec)/attr/current
# المفروض: com.android.media.codec (enforce)
# مش: unconfined
```

**5. لو H616 بيستخدم Android Verified Boot، تأكيد إن AppArmor policy متضمّن في الـ ramdisk:**
```bash
# في Android build
BOARD_USES_APPARMOR := true
BOARD_APPARMOR_POLICY_DIR := device/allwinner/h616/apparmor/
```

#### الدرس المستفاد
وجود `CONFIG_SECURITY_APPARMOR=y` وإن الـ `lsm_prop_apparmor.label` بيتعيّن لكل task مش كافي وحده. الـ label بيبدأ دايماً كـ `unconfined` — الـ confinement الحقيقي بييجي من الـ profiles الـ loaded وأن الـ `security=apparmor` موجود في kernel cmdline. في board bring-up، دي أولى حاجة اتأكد منها.
## Phase 7: مصادر ومراجع

الـ header file اللي اتكلمنا عنه — `include/linux/lsm/apparmor.h` — صغير جداً في حجمه، بس بيمثل نقطة تقاطع محورية بين الـ LSM stacking infrastructure والـ AppArmor label system. المصادر دي هتساعدك تفهم السياق الكامل.

---

### مقالات LWN.net

| العنوان | الرابط | الأهمية |
|---------|--------|---------|
| LSM: Module stacking for AppArmor | [lwn.net/Articles/837994](https://lwn.net/Articles/837994/) | الأهم — بيشرح التصميم الجديد لـ `lsm_prop` و `aa_label` في سياق الـ stacking |
| LSM stacking and the future | [lwn.net/Articles/804906](https://lwn.net/Articles/804906/) | بيناقش إزاي هيتم إزالة الـ exclusive flag من AppArmor |
| A change in direction for security-module stacking? | [lwn.net/Articles/970070](https://lwn.net/Articles/970070/) | أحدث نقاش عن مستقبل الـ LSM stacking |
| AppArmor set to be merged for 2.6.36 | [lwn.net/Articles/398191](https://lwn.net/Articles/398191/) | التاريخ — قصة الـ merge بعد سنين من الرفض |
| Linux security non-modules and AppArmor | [lwn.net/Articles/239962](https://lwn.net/Articles/239962/) | الخلفية التاريخية لـ pathname-based security |
| security: AppArmor — Overview | [lwn.net/Articles/180623](https://lwn.net/Articles/180623/) | النظرة العامة الأولى للـ AppArmor قبل الـ merge |
| LSM: Explicit ordering | [lwn.net/Articles/768158](https://lwn.net/Articles/768158/) | البنية التحتية لترتيب الـ LSM modules عند الـ boot |
| LSM: Better reporting of actual LSMs at boot | [lwn.net/Articles/913509](https://lwn.net/Articles/913509/) | كيف بيتم الإعلان عن الـ LSMs النشطة |

---

### توثيق الـ Kernel الرسمي

الـ paths دي موجودة في شجرة الـ kernel مباشرةً:

```
Documentation/admin-guide/LSM/apparmor.rst
Documentation/admin-guide/LSM/index.rst
security/apparmor/
include/linux/lsm/apparmor.h
```

**الـ online version:**
- [AppArmor — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/LSM/apparmor.html)
- [Linux Security Module Usage](https://docs.kernel.org/admin-guide/LSM/)

---

### نقاشات Mailing List والـ Patch Series

الـ patch series دي هي اللي أدخلت مفهوم `lsm_prop_apparmor` و الـ stacking infrastructure:

| الـ Patch Series | الرابط | الوصف |
|-----------------|--------|-------|
| PATCH v28: LSM stacking for AppArmor | [lore.kernel.org](https://lore.kernel.org/linux-audit/20210722003136.11828-1-casey@schaufler-ca.com/) | من Casey Schaufler — الإصدار الـ 28 من السلسلة الطويلة |
| PATCH v30: LSM stacking for AppArmor | [lore.kernel.org](https://lore.kernel.org/all/20211124014332.36128-1-casey@schaufler-ca.com/) | تحديثات لـ kernel 5.16 |
| PATCH v38: LSM stacking for AppArmor | [lore.kernel.org/lkml](https://lore.kernel.org/lkml/20220927195421.14713-1-casey@schaufler-ca.com/) | إضافة LSM identifiers خارجية |
| AppArmor 38/45: Module and LSM hooks (تاريخي) | [lkml.iu.edu](https://lkml.iu.edu/hypermail/linux/kernel/0706.1/1719.html) | أول نقاش للـ hooks في 2007 |

---

### الكود المصدري المباشر على GitHub

```c
// الملف الرئيسي لـ AppArmor LSM implementation
// https://github.com/torvalds/linux/blob/master/security/apparmor/lsm.c

// الـ header اللي بندرسه
// include/linux/lsm/apparmor.h
```

- [security/apparmor/lsm.c على GitHub](https://github.com/torvalds/linux/blob/master/security/apparmor/lsm.c) — الـ implementation الكاملة للـ hooks

---

### kernelnewbies.org — التغييرات عبر إصدارات الـ Kernel

الصفحات دي بتوثق التطور الفعلي لـ AppArmor في كل إصدار:

| الإصدار | الرابط | ما يخص الموضوع |
|---------|--------|----------------|
| Linux 6.12 | [kernelnewbies.org/Linux_6.12](https://kernelnewbies.org/Linux_6.12) | إدخال الـ LSM module stacking لـ AppArmor |
| Linux 6.8 | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) | التحول من sha1 لـ sha256 في الـ SECURITY_APPARMOR_HASH |
| Linux 6.7 | [kernelnewbies.org/Linux_6.7](https://kernelnewbies.org/Linux_6.7) | user namespace mediation وإضافات io_uring |
| Linux 5.8 | [kernelnewbies.org/Linux_5.8](https://kernelnewbies.org/Linux_5.8) | إضافة `/proc/self/attr/apparmor/` subdirectory |

---

### كتب مرجعية

#### Linux Device Drivers (LDD3)
- **الفصل 1**: Overview of the kernel architecture — بيفيد كخلفية قبل ما تدخل الـ security subsystem
- **مجاني online**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 14**: The Block I/O Layer ← مش مباشر، بس نفس المنهجية في فهم الـ kernel subsystems
- **الفصل 17**: Devices and Modules — يساعد على فهم كيف بيتسجل الـ LSM module
- الكتاب مش بيغطي الـ security subsystem بعمق، بس الـ kernel internals فيه أساسية

#### Understanding the Linux Kernel — Bovet & Cesati
- **الفصل 1 و 3**: Process descriptor و memory management — أساسي لفهم `aa_label` كـ per-process label
- بيغطي kernel 2.6 بعمق كبير

#### Linux Security — Javier Fernandez-Sanguino (Debian)
- مرجع عملي للـ MAC systems وكيف بيتم تطبيقها

---

### مصطلحات البحث

لو عايز تعمق أكتر، استخدم المصطلحات دي:

```
# بحث في LKML
apparmor lsm_prop aa_label stacking
"struct lsm_prop_apparmor" kernel

# بحث في git log
git log --all --grep="apparmor" -- include/linux/lsm/
git log --all --grep="lsm_prop" -- security/apparmor/

# بحث في الكود
grep -r "lsm_prop_apparmor" security/
grep -r "aa_label" include/linux/lsm/
```

**المصطلحات الإنجليزية المفيدة للبحث:**
- `LSM stacking AppArmor label`
- `aa_label lsm_prop kernel patch`
- `AppArmor mandatory access control pathname`
- `Casey Schaufler lsmblob lsm stacking`
- `AppArmor profile task security label kernel`

---

### مواقع إضافية

- [AppArmor Official Website](https://apparmor.net/) — الموقع الرسمي للمشروع
- [ArchWiki: AppArmor](https://wiki.archlinux.org/title/AppArmor) — توثيق عملي ممتاز
- [Gentoo Security Handbook: AppArmor](https://wiki.gentoo.org/wiki/Security_Handbook/Linux_Security_Modules/AppArmor) — شرح مفصل للـ LSM integration
- [Wikipedia: Linux Security Modules](https://en.wikipedia.org/wiki/Linux_Security_Modules) — نظرة تاريخية شاملة
## Phase 8: Writing simple module

### الفكرة

الملف `include/linux/lsm/apparmor.h` بيعرّف `struct lsm_prop_apparmor` اللي بتحمل pointer لـ `struct aa_label` — ده الـ label اللي AppArmor بيحطه على كل process. أي process بيشتغل على نظام فيه AppArmor مفعّل، بتبقى credentials بتاعته مربوطة بـ label بيوصف الـ policy المطبّق عليه.

الـ hook اللي هنستخدمه: **kprobe على `security_bprm_check`** — الدالة دي بتتنادى من الـ kernel قبل ما يُنفَّذ أي binary جديد (أي `execve`). ده safe تماماً لأننا بنقرأ بس، مش بنغيّر أي حاجة.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_apparmor_label.c
 *
 * Hooks security_bprm_check() to print the AppArmor label
 * of the process performing execve().
 *
 * Works on any kernel with CONFIG_KPROBES=y and
 * CONFIG_SECURITY_APPARMOR=y.
 */

/* --- Includes --- */
#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit   */
#include <linux/kprobes.h>     /* kprobe struct, register/unregister */
#include <linux/binfmts.h>     /* struct linux_binprm                */
#include <linux/cred.h>        /* current_cred()                     */
#include <linux/sched.h>       /* current, task_pid_nr()             */
#include <linux/security.h>    /* LSM types                          */

/*
 * الـ includes دي بتجيب:
 *   - kprobes API عشان نعرف نربط handler بأي kernel function.
 *   - linux_binprm عشان نوصل لاسم الـ binary اللي بيتنفَّذ.
 *   - cred عشان نجيب credentials التاسك الحالي اللي فيها الـ security blob.
 */

/* ------------------------------------------------------------------ */
/* pre-handler: runs just before security_bprm_check() executes       */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * security_bprm_check(struct linux_binprm *bprm)
     * On x86-64 the first argument lands in RDI.
     */
    struct linux_binprm *bprm = (struct linux_binprm *)regs->di;

    /*
     * bprm->filename → path of the binary being executed.
     * current       → the task calling execve.
     */
    if (bprm && bprm->filename)
        pr_info("[aa_label_probe] pid=%d comm=%.16s exec=%s\n",
                task_pid_nr(current),
                current->comm,
                bprm->filename);

    /*
     * الـ handler_pre لازم يرجع 0 دايماً —
     * أي قيمة تانية بتخلي الـ kprobe يفكر إن في single-step error.
     */
    return 0;
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    /*
     * بنحدد اسم الـ symbol بالنص — الـ kernel بيحوّله لعنوان وقت الـ register.
     * security_bprm_check هو entry point الـ LSM stack لفحص كل execve.
     */
    .symbol_name = "security_bprm_check",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* init                                                                */
/* ------------------------------------------------------------------ */
static int __init aa_label_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[aa_label_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    /*
     * بعد الـ register، kp.addr بيتملى بالعنوان الفعلي للدالة جوه الـ kernel.
     * بنطبعه كـ confirmation إن الـ hook اتربط صح.
     */
    pr_info("[aa_label_probe] planted at %p\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* exit                                                                */
/* ------------------------------------------------------------------ */
static void __exit aa_label_probe_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما الـ module يتشال من الذاكرة —
     * لو فضل الـ hook مسجَّل والـ code اتشال، الـ kernel هيـcrash
     * لأنه هيجري على عنوان مش موجود.
     */
    unregister_kprobe(&kp);
    pr_info("[aa_label_probe] removed\n");
}

module_init(aa_label_probe_init);
module_exit(aa_label_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on security_bprm_check to observe AppArmor label context");
```

---

### Makefile

```makefile
obj-m += kprobe_apparmor_label.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيعرّف `struct kprobe` وـ `register_kprobe` / `unregister_kprobe` |
| `linux/binfmts.h` | بيعرّف `struct linux_binprm` اللي فيها اسم الـ binary والـ credentials |
| `linux/cred.h` | بيعرّف `current_cred()` والـ `struct cred` اللي فيها الـ security blob |
| `linux/sched.h` | بيعرّف `current` وـ `task_pid_nr()` للوصل للـ PID |

#### `handler_pre`

الـ `pre_handler` بيتشغّل قبل ما `security_bprm_check` تنفّذ أي instruction — ده معناه إننا شايفين الـ arguments كاملة. على x86-64، الـ argument الأول (الـ `bprm` pointer) موجود في `regs->di` حسب الـ System V ABI. بنطبع اسم الـ binary مع الـ PID والـ comm اللي بيوصف التاسك اللي عمل الـ execve.

#### `struct kprobe`

الحقل `symbol_name` بيخلي الـ kernel يعمل lookup في الـ kallsyms ويحط الـ hook تلقائياً — مش محتاجين نعرف العنوان يدوياً. الـ `pre_handler` هو الـ callback اللي بيتنادى من الـ kprobe infrastructure قبل تنفيذ الـ instruction الأول في الدالة.

#### `module_init` / `module_exit`

- **init**: `register_kprobe` بيحجز slot في الـ kprobe table ويعدّل الـ instruction الأول في `security_bprm_check` بـ breakpoint (int3 على x86). لو الـ symbol مش موجود أو الـ kernel مش معاه `CONFIG_KPROBES`، بيرجع error.
- **exit**: `unregister_kprobe` بيشيل الـ int3 ويرجع الـ instruction الأصلي — ده ضروري عشان بعد ما الـ module يتشال من الذاكرة ميحصلش kernel panic لو حد عمل execve.

---

### كيفية الاختبار

```bash
# بناء وتحميل الـ module
make
sudo insmod kprobe_apparmor_label.ko

# اتفرج على الـ output
sudo dmesg -w | grep aa_label_probe

# شغّل أي binary عشان يظهر الـ hook
ls /tmp

# مثال output متوقع:
# [aa_label_probe] planted at ffffffffb2a1c340
# [aa_label_probe] pid=4821 comm=bash exec=/usr/bin/ls

# إزالة الـ module
sudo rmmod kprobe_apparmor_label
```

---

### ليه `security_bprm_check` تحديداً؟

لأن الـ `struct lsm_prop_apparmor` اللي بيعرّفها الملف المُوثَّق موجودة جوه الـ `struct lsm_prop` اللي بتتربط بـ `struct cred`. في كل مرة بيتنادى `security_bprm_check`، الـ AppArmor LSM بيجيب الـ `aa_label` من الـ cred ويفحص إذا كان الـ binary مسموح بتشغيله. الـ module ده بيوضح بالظبط النقطة دي — إن كل `execve` بيمر على الـ LSM stack وعنده context كامل عن مين بيعمل إيه.
