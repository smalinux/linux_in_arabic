## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ LSM؟

**Linux Security Module (LSM)** هو framework جوه الـ Linux kernel بيخلي أنظمة أمان متعددة تشتغل جنب بعض من غير ما يتعارضوا. الـ kernel بنفسه مش مبيعملش قرارات أمان بنفسه — هو بيوفر نقاط ربط (**hooks**) في أماكن حساسة زي فتح ملف، عمل socket connection، أو تحميل kernel module. كل LSM (زي SELinux، AppArmor، Smack) بيسجل نفسه في الـ hooks دي ويقرر: "هل مسموح أو لأ؟"

### القصة اللي الـ `lsm_audit.h` بيحلها

تخيل عندك 3 نظام أمان: SELinux، AppArmor، Smack. كل واحد فيهم لما بيرفض عملية بيحتاج يكتب سجل (audit log) زي:

```
type=AVC msg=audit(1234.567:89): avc: denied { read } for pid=1234
comm="bash" name="secret.txt" dev="sda1" ino=5678
saddr=192.168.1.1 dport=8080
```

المشكلة: إزاي كل LSM يكتب المعلومات دي؟ لو كل واحد كتب الكود من الأول — فيها تكرار ضخم ومش consistent. الحل؟ **ملف مشترك واحد**.

**الـ `lsm_audit.h`** هو الـ header اللي بيعرّف الـ data structures المشتركة اللي أي LSM يقدر يملأها لما بيرفض عملية، وبعدين function واحدة مشتركة تكتب الـ audit log بشكل موحد.

### الصورة الكبيرة بالتشبيه

تخيل الـ kernel زي بوابة أمان في مطار. كل راكب (process) بيطلب عمل حاجة (access resource). في البوابة في موظف أمان (LSM). لما الموظف يقول "ممنوع دخول":

1. بيملأ استمارة موحدة فيها: اسم الراكب، رقم جواز السفر، البوابة اللي وقف عندها، السبب
2. بيرفعها للسجل الرسمي للمطار (الـ audit subsystem)

الـ `lsm_audit.h` هو تصميم الاستمارة دي — **`struct common_audit_data`**.

### ليه ده مهم؟

بدون الـ `lsm_audit.h`:
- كل LSM محتاج يعرف ازاي يطبع IP addresses، inode numbers، capability names بنفسه
- الـ audit logs مش consistent بين الـ LSMs المختلفة
- كود مكرر في كل LSM

معاه:
- LSM بيملأ `struct common_audit_data` بالبيانات المناسبة
- بيستدعي `common_lsm_audit()` وخلاص
- الـ kernel يتكفل بباقي التفاصيل

### الـ `struct common_audit_data` — قلب الموضوع

```c
struct common_audit_data {
    char type;          /* نوع الـ resource المتنازع عليه */
    union {
        struct path path;        /* لو ملف */
        struct lsm_network_audit *net; /* لو network connection */
        int cap;                 /* لو Linux capability */
        int ipc_id;             /* لو IPC (shared memory/semaphore) */
        struct task_struct *tsk; /* لو process تاني */
        char *kmod_name;        /* لو kernel module */
        struct file *file;      /* لو file descriptor */
        /* ... إلخ */
    } u;
    /* بيانات خاصة بكل LSM */
    union {
        struct selinux_audit_data *selinux_audit_data;
        struct smack_audit_data   *smack_audit_data;
        struct apparmor_audit_data *apparmor_audit_data;
    };
};
```

**الـ `type` field** بيحدد إيه اللي في الـ `union u`:

| النوع | القيمة | المعنى |
|-------|--------|--------|
| `LSM_AUDIT_DATA_PATH` | 1 | طريق ملف |
| `LSM_AUDIT_DATA_NET` | 2 | network connection |
| `LSM_AUDIT_DATA_CAP` | 3 | Linux capability |
| `LSM_AUDIT_DATA_IPC` | 4 | IPC resource |
| `LSM_AUDIT_DATA_TASK` | 5 | process آخر |
| `LSM_AUDIT_DATA_KEY` | 6 | kernel keyring |
| `LSM_AUDIT_DATA_KMOD` | 8 | kernel module |
| `LSM_AUDIT_DATA_INODE` | 9 | inode مباشر |
| `LSM_AUDIT_DATA_FILE` | 12 | file struct |
| `LSM_AUDIT_DATA_IBPKEY` | 13 | InfiniBand pkey |
| `LSM_AUDIT_DATA_IBENDPORT` | 14 | InfiniBand endpoint |
| `LSM_AUDIT_DATA_LOCKDOWN` | 15 | kernel lockdown |
| `LSM_AUDIT_DATA_ANONINODE` | 17 | anonymous inode |
| `LSM_AUDIT_DATA_NLMSGTYPE` | 18 | netlink message |

### الـ Structs الفرعية

**الـ `struct lsm_network_audit`** — لما العملية المرفوضة عبارة عن network packet:

```c
struct lsm_network_audit {
    int netif;          /* رقم الـ network interface */
    const struct sock *sk; /* الـ socket */
    u16 family;         /* AF_INET أو AF_INET6 */
    __be16 dport;       /* destination port */
    __be16 sport;       /* source port */
    union {
        struct { __be32 daddr; __be32 saddr; } v4;  /* IPv4 */
        struct { struct in6_addr daddr; struct in6_addr saddr; } v6; /* IPv6 */
    } fam;
};
```

**الـ `struct lsm_ioctlop_audit`** — لما العملية ioctl على ملف:

```c
struct lsm_ioctlop_audit {
    struct path path;   /* مسار الملف */
    u16 cmd;           /* رقم أمر الـ ioctl */
};
```

**الـ `struct lsm_ibpkey_audit`** و **`struct lsm_ibendport_audit`** — لـ InfiniBand networking (شبكات HPC عالية السرعة).

### الـ Functions المعرفة

```c
/* تحويل IPv4 packet لبيانات audit */
int ipv4_skb_to_auditdata(struct sk_buff *skb,
        struct common_audit_data *ad, u8 *proto);

/* تحويل IPv6 packet لبيانات audit */
int ipv6_skb_to_auditdata(struct sk_buff *skb,
        struct common_audit_data *ad, u8 *proto);

/* الـ function الرئيسية — بتكتب الـ audit record */
void common_lsm_audit(struct common_audit_data *a,
    void (*pre_audit)(struct audit_buffer *, void *),
    void (*post_audit)(struct audit_buffer *, void *));

/* helper لكتابة البيانات المشتركة في buffer موجود */
void audit_log_lsm_data(struct audit_buffer *ab,
            const struct common_audit_data *a);
```

الـ `common_lsm_audit` بتقبل callback functions — الـ `pre_audit` و`post_audit` — عشان كل LSM يضيف بياناته الخاصة قبل وبعد البيانات المشتركة.

### تدفق العمل — رسم توضيحي

```
Process يطلب فتح ملف
         |
         v
   [VFS Layer]
         |
         v
   security_inode_open()   <-- LSM hook
         |
         v
   SELinux/AppArmor/Smack
   يفحصوا الـ policy
         |
    [مرفوض!]
         |
         v
   يملأ struct common_audit_data:
     .type = LSM_AUDIT_DATA_PATH
     .u.path = <مسار الملف>
     .selinux_audit_data = <تفاصيل SELinux>
         |
         v
   common_lsm_audit(ad, pre_cb, post_cb)
         |
         v
   audit_log_start()  --> audit_log_format() --> audit_log_end()
         |
         v
   يُكتب في /var/log/audit/audit.log
```

### الـ Conditional Compilation

الـ header بيتعامل بذكاء مع حالة إيقاف الـ audit:

```c
#ifdef CONFIG_AUDIT
/* الـ functions الحقيقية */
void common_lsm_audit(...);
#else
/* stubs فارغة — مش بيعمل حاجة */
static inline void common_lsm_audit(...) { }
#endif
```

لو الـ kernel اتبنى من غير `CONFIG_AUDIT`، الـ calls بتتحول لـ no-ops من غير overhead.

### الملفات المرتبطة

| الملف | الدور |
|-------|-------|
| `include/linux/lsm_audit.h` | الـ header — تعريف الـ structs والـ API |
| `security/lsm_audit.c` | الـ implementation — كود الـ functions |
| `include/linux/security.h` | الـ LSM framework API الرئيسي |
| `include/linux/lsm_hooks.h` | تعريف الـ hooks وهياكل الـ LSM |
| `include/linux/lsm_hook_defs.h` | قائمة كل الـ hooks بالـ X-macro pattern |
| `include/uapi/linux/lsm.h` | الـ IDs الرسمية لكل LSM |
| `security/security.c` | الـ LSM framework core |
| `security/lsm_init.c` | تهيئة الـ LSMs عند البوت |
| `security/selinux/avc.c` | مثال SELinux لاستخدام الـ audit |
| `security/smack/smack_access.c` | مثال Smack لاستخدام الـ audit |
| `security/apparmor/audit.c` | مثال AppArmor لاستخدام الـ audit |
| `include/linux/audit.h` | الـ audit subsystem API |

### الـ Subsystem

**الـ `lsm_audit.h`** ينتمي لـ **SECURITY SUBSYSTEM** — مش لـ SELinux أو AppArmor بالتحديد، لكن للـ framework المشترك اللي كل الـ LSMs بتقوم عليه. المشرفون هم Paul Moore وJames Morris.
## Phase 2: شرح الـ LSM Audit Framework

### المشكلة — ليه الـ Framework ده موجود؟

الـ Linux Security Modules (LSM) هو framework بيخلي الـ kernel يدعم أكتر من security policy في نفس الوقت — زي SELinux، AppArmor، وSMAck. كل واحد منهم بيقرر يسمح أو يرفض عمليات kernel زي فتح file أو network connection.

بس المشكلة: لما LSM بيرفض عملية، اللي بيحصل بعدين؟ لو ما فيش logging، المسؤول ما يعرفش ليه application فشل، ومش عارف يعمل diagnose للسياسة الأمنية.

الـ audit subsystem الأصلي في الـ kernel بيعمل logging لأحداث syscall-level. لكن كل LSM عنده context خاص بيه — SELinux بيحتاج يلوج AVC decisions، AppArmor بيحتاج profile violations، SMACK بيحتاج label decisions. لو كل LSM عمل logging بطريقته، هيكون فيه:

- **تكرار كود** لنفس المعلومات (path، network info، inode)
- **عدم consistency** في format الـ audit records
- **صعوبة** في كتابة tools زي `auditd` تقدر تفهم records من كل LSM

**الـ `lsm_audit.h`** بيحل المشكلة دي من خلال توفير **common audit data structure** وشارد logging functions تشتغل مع أي LSM.

---

### الحل — النهج اللي الـ Kernel اتاخده

الـ framework اشتغل على مبدأ **separation of concerns**:

1. **Common data collection**: الـ `struct common_audit_data` بتجمع كل المعلومات العامة عن الحدث (path، network، inode، capability...)
2. **LSM-specific extension**: كل LSM بيضيف data خاصة بيه جوه union منفصل
3. **Shared logging function**: الـ `common_lsm_audit()` بتعمل format للـ common data وبتستدعي hooks من كل LSM للـ LSM-specific parts

---

### الـ Big Picture Architecture

```
  User Space
  ─────────────────────────────────────────────────────
  auditd daemon ← /proc/kmsg أو netlink ← kernel audit records

  Kernel Space
  ─────────────────────────────────────────────────────

  ┌──────────────────────────────────────────────────┐
  │              System Call Layer                    │
  │  open(), connect(), execve(), ioctl(), ...        │
  └───────────────────┬──────────────────────────────┘
                      │ LSM hooks fired here
                      ▼
  ┌──────────────────────────────────────────────────┐
  │           LSM Hook Infrastructure                 │
  │  security_file_open()                             │
  │  security_socket_connect()                        │
  │  security_capable()                               │
  └───┬───────────────┬───────────────┬──────────────┘
      │               │               │
      ▼               ▼               ▼
  ┌────────┐    ┌──────────┐    ┌──────────┐
  │SELinux │    │AppArmor  │    │  SMACK   │
  │  avc_  │    │  aa_     │    │  smk_    │
  │ audit()│    │ audit()  │    │ audit()  │
  └───┬────┘    └────┬─────┘    └────┬─────┘
      │              │               │
      └──────────────┴───────────────┘
                     │
                     │  كل واحد بيبني struct common_audit_data
                     ▼
  ┌──────────────────────────────────────────────────┐
  │          lsm_audit Framework                      │
  │  ┌────────────────────────────────────────────┐  │
  │  │       struct common_audit_data             │  │
  │  │  type: LSM_AUDIT_DATA_PATH / NET / CAP ... │  │
  │  │  union u { path, net, cap, inode, ... }    │  │
  │  │  union { selinux_data, smack_data, aa_data}│  │
  │  └────────────────────────────────────────────┘  │
  │                                                   │
  │  common_lsm_audit(a, pre_audit, post_audit)       │
  │       │                                           │
  │       ├─ يستدعي pre_audit()  ← LSM-specific       │
  │       ├─ يعمل format للـ common data              │
  │       └─ يستدعي post_audit() ← LSM-specific       │
  └───────────────────────┬──────────────────────────┘
                          │
                          ▼
  ┌──────────────────────────────────────────────────┐
  │           Kernel Audit Subsystem                  │
  │  audit_log_start() → audit_buffer                 │
  │  audit_log_format() → يكتب strings               │
  │  audit_log_end()   → يبعت لـ auditd              │
  └──────────────────────────────────────────────────┘
```

---

### التشبيه من الواقع — مركز الشرطة والتقارير

تخيل إن فيه **مركز شرطة** (الـ kernel) فيه أقسام مختلفة: قسم المرور، قسم الجرائم الإلكترونية، قسم الآداب. كل قسم بيتعامل مع نوع مختلف من المخالفات.

لما بيحصل حادثة، لازم يتعمل تقرير يتبعت لـ **مكتب المدعي العام** (auditd).

**بدون الـ framework:**
- قسم المرور بيكتب تقرير بصيغته الخاصة
- قسم الجرائم الإلكترونية بكتب بصيغة تانية
- مكتب المدعي العام مش قادر يفهم أو يعالج التقارير

**مع الـ framework:**
- فيه **استمارة موحدة** (الـ `struct common_audit_data`) فيها حقول مشتركة: اسم المتهم، وقت الحادثة، الموقع
- كل قسم بيملا الاستمارة الموحدة + بيضيف تفاصيل خاصة بالقسم في مكان محجوز ليه
- **كاتب موحد** (`common_lsm_audit`) بيأخد الاستمارة، بيشيل المعلومات المشتركة، وبيطلب من القسم يضيف تفاصيله

**الـ mapping التفصيلي:**

| عنصر في التشبيه | المقابل الحقيقي |
|---|---|
| الاستمارة الموحدة | `struct common_audit_data` |
| حقول مشتركة (متهم، وقت، مكان) | `type` + `union u` (path, net, cap...) |
| تفاصيل خاصة بكل قسم | `union { selinux_data, smack_data, aa_data }` |
| الكاتب الموحد | `common_lsm_audit()` |
| `pre_audit` callback | القسم بيكتب مقدمة التقرير |
| `post_audit` callback | القسم بيضيف تفاصيل إضافية في النهاية |
| مكتب المدعي العام | kernel audit subsystem + auditd |
| الإرسال للمكتب | `audit_log_end()` |

---

### الـ Core Abstraction — ما هو المحور الأساسي؟

المحور الأساسي هو **الـ `struct common_audit_data`** اللي بيعمل **tagged union** لكل أنواع الأحداث الأمنية الممكنة في الـ kernel.

```c
struct common_audit_data {
    char type;   /* discriminator — نوع الحدث */

    union {      /* بيان الحدث — يختلف حسب النوع */
        struct path     path;       /* LSM_AUDIT_DATA_PATH    */
        struct dentry  *dentry;     /* LSM_AUDIT_DATA_DENTRY  */
        struct inode   *inode;      /* LSM_AUDIT_DATA_INODE   */
        struct lsm_network_audit *net; /* LSM_AUDIT_DATA_NET  */
        int             cap;        /* LSM_AUDIT_DATA_CAP     */
        int             ipc_id;     /* LSM_AUDIT_DATA_IPC     */
        struct task_struct *tsk;    /* LSM_AUDIT_DATA_TASK    */
        struct {
            key_serial_t key;
            char *key_desc;
        } key_struct;               /* LSM_AUDIT_DATA_KEY     */
        char           *kmod_name;  /* LSM_AUDIT_DATA_KMOD    */
        struct lsm_ioctlop_audit *op; /* LSM_AUDIT_DATA_IOCTL_OP */
        struct file    *file;       /* LSM_AUDIT_DATA_FILE    */
        struct lsm_ibpkey_audit *ibpkey;  /* InfiniBand pkey  */
        struct lsm_ibendport_audit *ibendport; /* IB endport  */
        int             reason;     /* LSM_AUDIT_DATA_LOCKDOWN */
        const char     *anonclass;  /* LSM_AUDIT_DATA_ANONINODE */
        u16             nlmsg_type; /* LSM_AUDIT_DATA_NLMSGTYPE */
    } u;

    union {      /* بيانات خاصة بكل LSM */
        struct smack_audit_data    *smack_audit_data;
        struct selinux_audit_data  *selinux_audit_data;
        struct apparmor_audit_data *apparmor_audit_data;
    };
};
```

**ليه tagged union ومش polymorphism؟**

الـ kernel مش بيستخدم C++ وبالتالي مفيش virtual functions. الـ tagged union هنا بيعطي نفس التأثير بطريقة C خالصة — بتعرف النوع من الـ `type` field وبعدين بتوصل للـ member الصح من الـ union.

---

### الـ Structs التفصيلية — ازاي كل حاجة بتربط ببعض؟

```
struct common_audit_data
├── char type  ──────────────────── discriminator
│                                  (18 قيمة مختلفة)
│
├── union u  ────────────────────── event-specific data
│   │
│   ├── struct path ─────────────── VFS path
│   │   ├── struct vfsmount *mnt   (mounted filesystem)
│   │   └── struct dentry *dentry  (directory entry)
│   │
│   ├── struct lsm_network_audit * ─── network event
│   │   ├── int netif              (interface index)
│   │   ├── const struct sock *sk  (socket)
│   │   ├── u16 family             (AF_INET / AF_INET6)
│   │   ├── __be16 dport / sport   (ports)
│   │   └── union fam
│   │       ├── struct { __be32 daddr, saddr } v4
│   │       └── struct { in6_addr daddr, saddr } v6
│   │
│   ├── struct lsm_ioctlop_audit * ─── ioctl event
│   │   ├── struct path path
│   │   └── u16 cmd
│   │
│   ├── struct lsm_ibpkey_audit * ──── InfiniBand pkey
│   │   ├── u64 subnet_prefix
│   │   └── u16 pkey
│   │
│   └── struct lsm_ibendport_audit * ─ InfiniBand endport
│       ├── const char *dev_name
│       └── u8 port
│
└── union (anonymous) ───────────── LSM-specific extension
    ├── struct smack_audit_data    *smack_audit_data
    ├── struct selinux_audit_data  *selinux_audit_data
    └── struct apparmor_audit_data *apparmor_audit_data
```

---

### الـ API الرئيسي — ازاي الـ Framework بيتستخدم؟

#### `common_lsm_audit()`

```c
void common_lsm_audit(struct common_audit_data *a,
    void (*pre_audit)(struct audit_buffer *, void *),
    void (*post_audit)(struct audit_buffer *, void *));
```

دي الـ function الرئيسية. الـ LSM بيستدعيها لما يريد يسجل حدث. هي بتعمل:

1. تفتح audit buffer (`audit_log_start`)
2. تستدعي `pre_audit` — بيكتب الـ LSM-specific header (مثلاً SELinux بيكتب `avc: denied`)
3. تستدعي `audit_log_lsm_data()` — بتكتب الـ common data (path, network, cap...)
4. تستدعي `post_audit` — بيكتب الـ LSM-specific footer
5. تغلق الـ buffer (`audit_log_end`)

#### `audit_log_lsm_data()`

```c
void audit_log_lsm_data(struct audit_buffer *ab,
                         const struct common_audit_data *a);
```

بتعمل format للـ `union u` بناءً على قيمة الـ `type`. مثلاً لو النوع `LSM_AUDIT_DATA_PATH` هتكتب `path=/etc/shadow`.

#### Helper functions للـ Network

```c
int ipv4_skb_to_auditdata(struct sk_buff *skb,
                           struct common_audit_data *ad, u8 *proto);

int ipv6_skb_to_auditdata(struct sk_buff *skb,
                           struct common_audit_data *ad, u8 *proto);
```

بيملوا الـ `lsm_network_audit` struct من الـ `sk_buff` (الـ network packet). ده بيعفي الـ LSM من parse الـ IP headers بنفسه.

---

### مثال عملي — SELinux بيرفض open()

```c
/* داخل SELinux — لما بيرفض file access */
static void selinux_audit_rule_free(void *lsm_rule)
{
    struct common_audit_data ad;
    struct selinux_audit_data sad;

    /* 1. ابدأ بـ common data */
    ad.type = LSM_AUDIT_DATA_PATH;
    ad.u.path = file->f_path;   /* الـ path للـ file */

    /* 2. ضيف الـ SELinux-specific data */
    sad.tclass = SECCLASS_FILE;
    sad.requested = FILE__READ;
    sad.denied = FILE__READ;
    ad.selinux_audit_data = &sad;

    /* 3. استدعي الـ framework */
    common_lsm_audit(&ad, avc_audit_pre_callback, avc_audit_post_callback);
}

/* الـ pre_callback بتكتب: "avc: denied { read } for" */
/* الـ common framework بتكتب: "path=/etc/shadow" */
/* الـ post_callback بتكتب: "scontext=... tcontext=... tclass=file" */
```

**النتيجة في الـ audit log:**
```
type=AVC msg=audit(1234567890.123:456): avc: denied { read } for
  pid=1234 comm="httpd" path="/etc/shadow"
  scontext=system_u:system_r:httpd_t:s0
  tcontext=system_u:object_r:shadow_t:s0
  tclass=file permissive=0
```

---

### الـ InfiniBand Support — ليه موجود هنا؟

الـ `lsm_ibpkey_audit` و`lsm_ibendport_audit` بيدعموا audit لشبكات InfiniBand، وهي تقنية high-speed networking بتُستخدم في HPC (High Performance Computing) و data centers.

InfiniBand بيستخدم **partition keys (pkeys)** بدل IP addresses لتقسيم الشبكة. SELinux و SMACK بيدعموا policy على مستوى IB network، وبالتالي محتاجين يعملوا audit لـ pkey و port violations.

---

### الـ Compile-time Guards

```c
#ifdef CONFIG_AUDIT
/* الـ real implementations */
void common_lsm_audit(...);
void audit_log_lsm_data(...);
#else
/* stubs فاضية لو audit مش مفعّل */
static inline void common_lsm_audit(...) {}
static inline void audit_log_lsm_data(...) {}
#endif
```

لو `CONFIG_AUDIT=n` في الـ kernel config، كل calls للـ framework بتتحول لـ no-ops. ده بيضمن إن الـ LSM code مش محتاج `#ifdef` في كل مكان.

كذلك:
```c
#ifdef CONFIG_KEYS
    struct { key_serial_t key; char *key_desc; } key_struct;
#endif
```

الـ key audit data موجودة بس لو `CONFIG_KEYS=y` — ده نموذج على **conditional compilation** مبني على features مش على architecture.

---

### الـ Framework يملك إيه؟ وبيفوّض إيه؟

| المسؤولية | صاحبها |
|---|---|
| تعريف `struct common_audit_data` | **lsm_audit framework** |
| تعريف نوع الحدث (type discriminator) | **lsm_audit framework** |
| Format الـ common fields (path, net, cap...) | **lsm_audit framework** |
| فتح وغلق الـ audit buffer | **lsm_audit framework** |
| Parse الـ IP packet إلى audit data | **lsm_audit framework** |
| كتابة الـ LSM-specific header/footer | **كل LSM لنفسه** (pre/post callbacks) |
| تعريف `selinux_audit_data` / `smack_audit_data` | **كل LSM لنفسه** |
| تحديد متى يتعمل audit | **كل LSM لنفسه** |
| إرسال الـ record لـ userspace | **kernel audit subsystem** |

---

### الاعتماد على Subsystems تانية

- **Audit Subsystem** (`linux/audit.h`): الـ lsm_audit framework مبني فوقيه. الـ audit subsystem هو اللي بيدير الـ `audit_buffer` وبيبعت records لـ userspace عن طريق netlink socket.
- **VFS** (Virtual Filesystem): الـ `struct path` و`struct dentry` و`struct inode` كلها من الـ VFS layer — الـ abstraction layer اللي بيخلي الـ kernel يتعامل مع كل أنواع filesystems بنفس الطريقة.
- **Network Stack**: الـ `struct sock` و`struct sk_buff` من الـ network subsystem — كل packet في الـ kernel محتاز كـ `sk_buff`.
- **IPC Subsystem**: الـ `ipc_id` بيرجع لـ IPC objects زي shared memory و message queues.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Type Constants — Cheatsheet

الـ `common_audit_data.type` بيحدد إيه اللي جوا الـ union. الجدول ده بيوضح كل قيمة:

| Constant | Value | المحتوى في الـ union | الاستخدام |
|---|---|---|---|
| `LSM_AUDIT_DATA_PATH` | 1 | `struct path path` | عمليات على ملفات عبر VFS path كامل |
| `LSM_AUDIT_DATA_NET` | 2 | `struct lsm_network_audit *net` | عمليات شبكة (TCP/UDP/IPv4/IPv6) |
| `LSM_AUDIT_DATA_CAP` | 3 | `int cap` | Linux capability checks |
| `LSM_AUDIT_DATA_IPC` | 4 | `int ipc_id` | IPC objects (shared memory, semaphores) |
| `LSM_AUDIT_DATA_TASK` | 5 | `struct task_struct *tsk` | عمليات بين processes |
| `LSM_AUDIT_DATA_KEY` | 6 | `key_struct { key_serial_t, char* }` | kernel keyring (requires `CONFIG_KEYS`) |
| `LSM_AUDIT_DATA_NONE` | 7 | — | لا يوجد بيانات إضافية |
| `LSM_AUDIT_DATA_KMOD` | 8 | `char *kmod_name` | تحميل kernel modules |
| `LSM_AUDIT_DATA_INODE` | 9 | `struct inode *inode` | عمليات على inode مباشرةً |
| `LSM_AUDIT_DATA_DENTRY` | 10 | `struct dentry *dentry` | عمليات على dentry |
| `LSM_AUDIT_DATA_IOCTL_OP` | 11 | `struct lsm_ioctlop_audit *op` | IOCTL calls على devices |
| `LSM_AUDIT_DATA_FILE` | 12 | `struct file *file` | عمليات على open file descriptor |
| `LSM_AUDIT_DATA_IBPKEY` | 13 | `struct lsm_ibpkey_audit *ibpkey` | InfiniBand partition keys |
| `LSM_AUDIT_DATA_IBENDPORT` | 14 | `struct lsm_ibendport_audit *ibendport` | InfiniBand end ports |
| `LSM_AUDIT_DATA_LOCKDOWN` | 15 | `int reason` | kernel lockdown checks |
| `LSM_AUDIT_DATA_NOTIFICATION` | 16 | `int reason` | LSM notifications |
| `LSM_AUDIT_DATA_ANONINODE` | 17 | `const char *anonclass` | anonymous inodes (مثلاً `io_uring`) |
| `LSM_AUDIT_DATA_NLMSGTYPE` | 18 | `u16 nlmsg_type` | Netlink message types |

---

### الـ Config Options المؤثرة على الملف

| Option | الأثر |
|---|---|
| `CONFIG_AUDIT` | لو مش enabled، `common_lsm_audit` و`audit_log_lsm_data` بيبقوا empty stubs |
| `CONFIG_IPV6` | بيفعّل `ipv6_skb_to_auditdata()` |
| `CONFIG_KEYS` | بيفعّل حقل `key_struct` جوا الـ union |
| `CONFIG_SECURITY_SMACK` | بيفعّل `smack_audit_data*` في الـ LSM union |
| `CONFIG_SECURITY_SELINUX` | بيفعّل `selinux_audit_data*` في الـ LSM union |
| `CONFIG_SECURITY_APPARMOR` | بيفعّل `apparmor_audit_data*` في الـ LSM union |

---

### الـ Structs المهمة

#### 1. `struct common_audit_data`

**الغرض:** ده الـ struct الرئيسي في الملف كله — بيمثل **سجل audit واحد** بيتبنيه أي LSM عشان يوصف عملية انتهكت سياسة الأمان. بيتمرر لـ `common_lsm_audit()` اللي بتكتبه في الـ audit log.

```c
struct common_audit_data {
    char type;          /* نوع البيانات — بيحدد أي branch في الـ union */
    union {
        struct path path;               /* LSM_AUDIT_DATA_PATH  */
        struct dentry *dentry;          /* LSM_AUDIT_DATA_DENTRY */
        struct inode *inode;            /* LSM_AUDIT_DATA_INODE  */
        struct lsm_network_audit *net;  /* LSM_AUDIT_DATA_NET   */
        int cap;                        /* LSM_AUDIT_DATA_CAP   */
        int ipc_id;                     /* LSM_AUDIT_DATA_IPC   */
        struct task_struct *tsk;        /* LSM_AUDIT_DATA_TASK  */
        struct { key_serial_t key; char *key_desc; } key_struct; /* KEY */
        char *kmod_name;                /* LSM_AUDIT_DATA_KMOD  */
        struct lsm_ioctlop_audit *op;   /* LSM_AUDIT_DATA_IOCTL_OP */
        struct file *file;              /* LSM_AUDIT_DATA_FILE  */
        struct lsm_ibpkey_audit *ibpkey;    /* IBPKEY  */
        struct lsm_ibendport_audit *ibendport; /* IBENDPORT */
        int reason;                     /* LOCKDOWN / NOTIFICATION */
        const char *anonclass;          /* LSM_AUDIT_DATA_ANONINODE */
        u16 nlmsg_type;                 /* LSM_AUDIT_DATA_NLMSGTYPE */
    } u;
    union {
        struct smack_audit_data    *smack_audit_data;    /* Smack only */
        struct selinux_audit_data  *selinux_audit_data;  /* SELinux only */
        struct apparmor_audit_data *apparmor_audit_data; /* AppArmor only */
    }; /* anonymous union — مباشرةً accessible */
};
```

**الربط بالـ structs التانية:**
- بيضم `lsm_network_audit` بالإشارة عشان البيانات الشبكية كبيرة
- بيضم `lsm_ioctlop_audit` بالإشارة عشان فيه `struct path` جواه
- بيضم `lsm_ibpkey_audit` و `lsm_ibendport_audit` بالإشارة للـ InfiniBand
- بيتمرر لـ `audit_buffer` الخاص بالـ audit subsystem

---

#### 2. `struct lsm_network_audit`

**الغرض:** بيحمل كل المعلومات الشبكية اللي محتاجها لـ audit event واحد. بيدعم IPv4 و IPv6 في نفس الوقت عبر union.

```c
struct lsm_network_audit {
    int netif;          /* network interface index (if_index) */
    const struct sock *sk;  /* الـ socket المعنية — قد تكون NULL */
    u16 family;         /* AF_INET أو AF_INET6 */
    __be16 dport;       /* destination port — big-endian */
    __be16 sport;       /* source port — big-endian */
    union {
        struct { __be32 daddr; __be32 saddr; } v4;  /* IPv4 addresses */
        struct { struct in6_addr daddr; struct in6_addr saddr; } v6; /* IPv6 */
    } fam;
};
```

الـ macros `v4info` و`v6info` بيختصروا الوصول:
```c
#define v4info fam.v4
#define v6info fam.v6
/* استخدام: ad.u.net->v4info.daddr */
```

**الربط:** بيتم الإشارة إليه من `common_audit_data.u.net`. الدوال `ipv4_skb_to_auditdata()` و`ipv6_skb_to_auditdata()` بتملأ هذا الـ struct من الـ `sk_buff`.

---

#### 3. `struct lsm_ioctlop_audit`

**الغرض:** بيسجل عملية `ioctl` — بيجمع الـ device path مع الـ command number.

```c
struct lsm_ioctlop_audit {
    struct path path;   /* الـ path الكامل للـ device file */
    u16 cmd;            /* رقم الـ ioctl command */
};
```

**الربط:** بيتم الإشارة إليه من `common_audit_data.u.op`. الـ `struct path` جواه بيشاور على `vfsmount` و`dentry`.

---

#### 4. `struct lsm_ibpkey_audit`

**الغرض:** بيسجل عمليات InfiniBand المتعلقة بالـ partition keys — جزء من RDMA security.

```c
struct lsm_ibpkey_audit {
    u64 subnet_prefix;  /* الـ subnet prefix للـ InfiniBand fabric */
    u16 pkey;           /* partition key value */
};
```

---

#### 5. `struct lsm_ibendport_audit`

**الغرض:** بيسجل عمليات InfiniBand المتعلقة بالـ end ports.

```c
struct lsm_ibendport_audit {
    const char *dev_name;   /* اسم الـ RDMA device */
    u8 port;                /* رقم الـ port */
};
```

---

### رسم علاقات الـ Structs

```
                    ┌─────────────────────────────────────────────┐
                    │         common_audit_data                   │
                    │  ┌──────────────────────────────────────┐   │
                    │  │  char type  (LSM_AUDIT_DATA_*)       │   │
                    │  └──────────────────────────────────────┘   │
                    │  union u {                                   │
                    │    struct path path ──────────────────────┐  │
                    │    struct dentry *dentry                  │  │
                    │    struct inode  *inode                   │  │
                    │    struct lsm_network_audit *net ─────────┼──┼──► lsm_network_audit
                    │    int cap / ipc_id / reason              │  │       ├─ netif
                    │    struct task_struct *tsk                │  │       ├─ sock *sk
                    │    key_struct { key_serial_t, char* }     │  │       ├─ family
                    │    char *kmod_name                        │  │       ├─ dport/sport
                    │    struct lsm_ioctlop_audit *op ──────────┼──┼──► lsm_ioctlop_audit
                    │    struct file *file                      │  │       ├─ struct path
                    │    struct lsm_ibpkey_audit *ibpkey ───────┼──┼──► lsm_ibpkey_audit
                    │    struct lsm_ibendport_audit *ibendport ─┼──┼──► lsm_ibendport_audit
                    │    const char *anonclass                  │  │
                    │    u16 nlmsg_type                         │  │
                    │  }                                        │  │
                    │  union (anonymous — LSM private) {        │  │
                    │    smack_audit_data    *                  │  │
                    │    selinux_audit_data  *                  │  │
                    │    apparmor_audit_data *                  │  │
                    │  }                                        │  │
                    └───────────────────────────────────────────┼──┘
                                                                │
                    struct path ◄───────────────────────────────┘
                    ├─ struct vfsmount *mnt
                    └─ struct dentry  *dentry


    lsm_network_audit.fam union:
    ┌──────────────────────────────────┐
    │ family == AF_INET  → fam.v4      │
    │   ├─ __be32 daddr                │
    │   └─ __be32 saddr                │
    │ family == AF_INET6 → fam.v6      │
    │   ├─ struct in6_addr daddr       │
    │   └─ struct in6_addr saddr       │
    └──────────────────────────────────┘
```

---

### دورة حياة سجل الـ Audit

```
LSM hook يُستدعى (مثلاً: selinux_inode_permission)
        │
        ▼
┌─────────────────────────────────┐
│  تعريف common_audit_data على   │
│  الـ stack (stack allocation)  │
│  struct common_audit_data ad;  │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  تحديد النوع وملء الـ union    │
│  ad.type = LSM_AUDIT_DATA_PATH │
│  ad.u.path = ...               │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  ملء الـ LSM private union     │
│  ad.selinux_audit_data = &sad  │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  استدعاء common_lsm_audit(&ad, │
│    pre_audit_cb, post_audit_cb) │
└────────────┬────────────────────┘
             │
             ├── [CONFIG_AUDIT enabled]
             │         │
             │         ▼
             │   ┌──────────────────────────────┐
             │   │ audit_log_start() → ab        │
             │   │ pre_audit_cb(ab, &ad)         │
             │   │ audit_log_lsm_data(ab, &ad)   │
             │   │   ├─ يقرأ ad.type            │
             │   │   └─ يكتب البيانات المناسبة  │
             │   │ post_audit_cb(ab, &ad)        │
             │   │ audit_log_end(ab)             │
             │   └──────────────────────────────┘
             │
             └── [CONFIG_AUDIT disabled]
                       │
                       ▼
                   stub function — no-op
                       │
                       ▼
             common_audit_data يتحرر تلقائياً
             (stack frame destruction)
```

---

### Call Flow Diagrams

#### Flow 1: شبكي — SELinux يرفض اتصال TCP

```
SELinux hook: selinux_socket_connect()
    │
    ├─ ينشئ common_audit_data ad على الـ stack
    ├─ ad.type = LSM_AUDIT_DATA_NET
    ├─ ينشئ lsm_network_audit net
    ├─ ad.u.net = &net
    ├─ يملأ net.family, net.dport, net.sport
    │
    └─► common_lsm_audit(&ad, avc_audit_pre_callback, avc_audit_post_callback)
            │
            ├─► audit_log_start(current->audit_context, GFP_ATOMIC, AUDIT_AVC)
            │       └─ يرجع audit_buffer *ab
            │
            ├─► pre_callback(ab, &ad)    [SELinux: يكتب avc: denied {connect}]
            │
            ├─► audit_log_lsm_data(ab, &ad)
            │       ├─ يقرأ ad.type == LSM_AUDIT_DATA_NET
            │       ├─ يطبع: laddr=... lport=... faddr=... fport=...
            │       └─ يطبع: netif=eth0
            │
            ├─► post_callback(ab, &ad)   [SELinux: يكتب الـ SID/context]
            │
            └─► audit_log_end(ab)        [يرسل الرسالة لـ auditd]
```

#### Flow 2: ملف — AppArmor يرفض قراءة ملف

```
AppArmor hook: apparmor_file_permission()
    │
    ├─ ad.type = LSM_AUDIT_DATA_PATH
    ├─ ad.u.path = file->f_path
    ├─ ad.apparmor_audit_data = &aad
    │
    └─► common_lsm_audit(&ad, aa_audit_pre, NULL)
            │
            ├─► audit_log_start(...)
            ├─► pre_callback: يكتب "apparmor=DENIED operation=open"
            ├─► audit_log_lsm_data(ab, &ad)
            │       ├─ type == PATH
            │       └─ audit_log_d_path(ab, "name=", &ad.u.path)
            └─► audit_log_end(ab)
```

#### Flow 3: شبكي — ملء البيانات من sk_buff

```
LSM hook يحصل على struct sk_buff *skb
    │
    ├─ ينشئ common_audit_data ad
    ├─ ad.type = LSM_AUDIT_DATA_NET
    │
    ├─► ipv4_skb_to_auditdata(skb, &ad, &proto)
    │       ├─ يقرأ ip_hdr(skb)->saddr → ad.u.net->v4info.saddr
    │       ├─ يقرأ ip_hdr(skb)->daddr → ad.u.net->v4info.daddr
    │       ├─ proto == IPPROTO_TCP → يقرأ tcp_hdr(skb)->source/dest
    │       └─ يملأ ad.u.net->sport, dport
    │
    └─► common_lsm_audit(&ad, pre_cb, post_cb)
```

---

### استراتيجية الـ Locking

ملف `lsm_audit.h` نفسه **ما بيعرّفش locks**، لكن استخدامه بيتبع قواعد محددة:

#### 1. الـ `common_audit_data` نفسها — No Lock Needed

الـ struct دي دايماً على الـ **stack** وبتتمرر local لـ call chain واحدة. مفيش sharing بين threads، فمفيش حاجة لـ lock.

```c
/* مثال — SELinux اvc.c */
struct common_audit_data ad;   /* على الـ stack — thread-local */
/* ... ملء البيانات ... */
common_lsm_audit(&ad, ...);    /* synchronous call */
/* ad يتحرر هنا */
```

#### 2. الـ `audit_buffer` — Protected by Audit Subsystem

الـ `audit_buffer *ab` اللي بتيجي من `audit_log_start()` محمية داخلياً. الـ LSM بيستخدمها بشكل sequential ومفيش اشتراك.

#### 3. الـ `lsm_network_audit` — RCU / Lock-free Read

لما بيتم القراءة من الـ `sock *sk` جوا `lsm_network_audit`:
- الـ sk_buff بيكون مملوك لـ current context
- الـ socket stats بتتقرأ مرة واحدة وبتتخزن في الـ struct
- `GFP_ATOMIC` بيتستخدم في الـ contexts اللي ممنوع فيها sleep (softirq, BH)

#### 4. ترتيب الـ Locks (Lock Ordering) — الـ Audit Layer

```
task_lock / rcu_read_lock
    └─► audit_context lock (internal to audit.c)
            └─► audit_buffer allocation (GFP_ATOMIC or GFP_KERNEL)
                    └─► common_lsm_audit() [لا تمسك أي lock خارجي]
```

**القاعدة الذهبية:** `common_lsm_audit()` بتتستدعى **بعد** ما يتقرر الـ access decision — يعني بعد ما يتحرر أي lock خاص بالـ LSM policy نفسها، عشان تجنب deadlock مع الـ audit subsystem.

#### 5. الـ `GFP_ATOMIC` vs `GFP_KERNEL`

| السياق | الـ gfp flag المستخدم |
|---|---|
| network hooks (softirq) | `GFP_ATOMIC` |
| filesystem hooks (process context) | `GFP_KERNEL` |
| socket operations (mixed) | `GFP_ATOMIC` (آمن في كل الحالات) |

الـ `audit_log_start()` بيتقالها الـ flag ده عشان تحدد إزاي تعمل allocate للـ buffer، وده بيتحكم فيه الـ LSM اللي بيستدعي.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ Data Structures

| الـ Struct | الغرض |
|---|---|
| `common_audit_data` | الـ container الرئيسي لكل بيانات الـ audit event في الـ LSM |
| `lsm_network_audit` | بيانات الـ network (IP, port, socket) للـ audit |
| `lsm_ioctlop_audit` | بيانات الـ ioctl operation (path + cmd number) |
| `lsm_ibpkey_audit` | بيانات الـ InfiniBand partition key |
| `lsm_ibendport_audit` | بيانات الـ InfiniBand end port |

#### الـ Functions / APIs

| الـ Function | الـ Config Guard | الغرض |
|---|---|---|
| `ipv4_skb_to_auditdata()` | `CONFIG_AUDIT` | يملأ الـ `common_audit_data` من IPv4 packet |
| `ipv6_skb_to_auditdata()` | `CONFIG_AUDIT && CONFIG_IPV6` | يملأ الـ `common_audit_data` من IPv6 packet |
| `common_lsm_audit()` | `CONFIG_AUDIT` (stub بدونه) | ينشئ ويرسل الـ audit record كامل |
| `audit_log_lsm_data()` | `CONFIG_AUDIT` (stub بدونه) | يكتب بيانات الـ `common_audit_data` في audit buffer موجود |

---

### Group 1: الـ Data Structures

هذه الـ structs هي الـ building blocks اللي كل الـ LSM modules بتستخدمها عشان تجمع بيانات الـ security event قبل ما ترسلها للـ audit subsystem.

---

#### `struct lsm_network_audit`

```c
struct lsm_network_audit {
    int netif;           /* network interface index */
    const struct sock *sk; /* associated socket */
    u16 family;          /* AF_INET or AF_INET6 */
    __be16 dport;        /* destination port (big-endian) */
    __be16 sport;        /* source port (big-endian) */
    union {
        struct {
            __be32 daddr; /* IPv4 dest addr */
            __be32 saddr; /* IPv4 src addr */
        } v4;
        struct {
            struct in6_addr daddr; /* IPv6 dest addr */
            struct in6_addr saddr; /* IPv6 src addr */
        } v6;
    } fam;
};
```

**الـ struct ده** بيتعمل fill من الـ skb أو من الـ socket مباشرةً، وبيتضاف للـ `common_audit_data` عن طريق الـ `net` pointer في الـ union.
الـ macros `v4info` و `v6info` بتختصر الوصول لـ `fam.v4` و `fam.v6`.

---

#### `struct lsm_ioctlop_audit`

```c
struct lsm_ioctlop_audit {
    struct path path; /* vfsmount + dentry of the target file */
    u16 cmd;          /* ioctl command number */
};
```

**الـ struct ده** بيستخدمه الـ LSM لما يـ audit على `ioctl()` call — بيسجل الـ file اللي اتعمل عليه الـ ioctl والـ command number.

---

#### `struct lsm_ibpkey_audit`

```c
struct lsm_ibpkey_audit {
    u64 subnet_prefix; /* IB subnet prefix */
    u16 pkey;          /* partition key */
};
```

**الـ struct ده** خاص بـ InfiniBand security — بيسجل الـ subnet prefix والـ partition key اللي اتعمل عليها الـ access decision.

---

#### `struct lsm_ibendport_audit`

```c
struct lsm_ibendport_audit {
    const char *dev_name; /* IB device name string */
    u8 port;              /* port number on the IB device */
};
```

**الـ struct ده** بيسجل الـ InfiniBand end port اللي اتعمل عليه الـ security check.

---

#### `struct common_audit_data`

```c
struct common_audit_data {
    char type;   /* LSM_AUDIT_DATA_* constant — determines union branch */
    union {
        struct path path;               /* LSM_AUDIT_DATA_PATH / FILE */
        struct dentry *dentry;          /* LSM_AUDIT_DATA_DENTRY */
        struct inode *inode;            /* LSM_AUDIT_DATA_INODE */
        struct lsm_network_audit *net;  /* LSM_AUDIT_DATA_NET */
        int cap;                        /* LSM_AUDIT_DATA_CAP */
        int ipc_id;                     /* LSM_AUDIT_DATA_IPC */
        struct task_struct *tsk;        /* LSM_AUDIT_DATA_TASK */
        struct { key_serial_t key; char *key_desc; } key_struct; /* KEY */
        char *kmod_name;                /* LSM_AUDIT_DATA_KMOD */
        struct lsm_ioctlop_audit *op;   /* LSM_AUDIT_DATA_IOCTL_OP */
        struct file *file;              /* LSM_AUDIT_DATA_FILE */
        struct lsm_ibpkey_audit *ibpkey;       /* LSM_AUDIT_DATA_IBPKEY */
        struct lsm_ibendport_audit *ibendport; /* LSM_AUDIT_DATA_IBENDPORT */
        int reason;                     /* LSM_AUDIT_DATA_LOCKDOWN */
        const char *anonclass;          /* LSM_AUDIT_DATA_ANONINODE */
        u16 nlmsg_type;                 /* LSM_AUDIT_DATA_NLMSGTYPE */
    } u;
    union {
        struct smack_audit_data    *smack_audit_data;    /* CONFIG_SECURITY_SMACK */
        struct selinux_audit_data  *selinux_audit_data;  /* CONFIG_SECURITY_SELINUX */
        struct apparmor_audit_data *apparmor_audit_data; /* CONFIG_SECURITY_APPARMOR */
    }; /* anonymous union — LSM-specific private data */
};
```

**ده الـ struct المركزي** اللي كل الـ LSM hooks بتملأه قبل ما تـ call `common_lsm_audit()`. الـ `type` field هو الـ discriminant اللي بيحدد أي branch في الـ union `u` صالح للاستخدام. الـ anonymous union التانية بتخلي كل LSM يحط private data خاص بيه (زي الـ SID في SELinux) جنب الـ generic data.

**الـ type constants:**

| الـ Constant | القيمة | الـ Union Branch المستخدم |
|---|---|---|
| `LSM_AUDIT_DATA_PATH` | 1 | `u.path` |
| `LSM_AUDIT_DATA_NET` | 2 | `u.net` |
| `LSM_AUDIT_DATA_CAP` | 3 | `u.cap` |
| `LSM_AUDIT_DATA_IPC` | 4 | `u.ipc_id` |
| `LSM_AUDIT_DATA_TASK` | 5 | `u.tsk` |
| `LSM_AUDIT_DATA_KEY` | 6 | `u.key_struct` |
| `LSM_AUDIT_DATA_NONE` | 7 | لا شيء |
| `LSM_AUDIT_DATA_KMOD` | 8 | `u.kmod_name` |
| `LSM_AUDIT_DATA_INODE` | 9 | `u.inode` |
| `LSM_AUDIT_DATA_DENTRY` | 10 | `u.dentry` |
| `LSM_AUDIT_DATA_IOCTL_OP` | 11 | `u.op` |
| `LSM_AUDIT_DATA_FILE` | 12 | `u.file` |
| `LSM_AUDIT_DATA_IBPKEY` | 13 | `u.ibpkey` |
| `LSM_AUDIT_DATA_IBENDPORT` | 14 | `u.ibendport` |
| `LSM_AUDIT_DATA_LOCKDOWN` | 15 | `u.reason` |
| `LSM_AUDIT_DATA_NOTIFICATION` | 16 | — |
| `LSM_AUDIT_DATA_ANONINODE` | 17 | `u.anonclass` |
| `LSM_AUDIT_DATA_NLMSGTYPE` | 18 | `u.nlmsg_type` |

---

### Group 2: الـ Packet Parsing Functions

الـ functions دي بتـ parse الـ `sk_buff` وبتملأ الـ `lsm_network_audit` جوا الـ `common_audit_data` — بيتعملوا call من الـ LSM hooks اللي بتشتغل على network packets.

---

#### `ipv4_skb_to_auditdata()`

```c
int ipv4_skb_to_auditdata(struct sk_buff *skb,
                           struct common_audit_data *ad,
                           u8 *proto);
```

**الـ function دي** بتـ extract الـ IPv4 header من الـ `skb` وبتملأ الـ `ad->u.net` بالـ source/destination addresses والـ ports. لو الـ `proto` مش NULL، بترجع الـ IP protocol number (TCP/UDP/etc).

**الـ Parameters:**

| الـ Parameter | النوع | الشرح |
|---|---|---|
| `skb` | `struct sk_buff *` | الـ packet buffer — لازم يحتوي على IPv4 header صالح |
| `ad` | `struct common_audit_data *` | الـ audit data struct اللي هيتملى — الـ `ad->u.net` لازم يكون pointer على `lsm_network_audit` معمول allocate قبل كده |
| `proto` | `u8 *` | optional output — الـ transport layer protocol number (TCP=6, UDP=17, إلخ)، أو NULL لو مش محتاجه |

**الـ Return Value:** `0` في حالة النجاح، أو negative error code لو الـ skb مش IPv4 أو البيانات ناقصة (زي `skb_network_header` مش صالح).

**Key Details:**
- **لا يحتاج lock** — بيشتغل read-only على الـ skb data.
- بيستخدم `ip_hdr(skb)` للوصول للـ header.
- لو الـ protocol هو TCP أو UDP، بيـ parse الـ transport header كمان عشان يجيب الـ ports.
- بيتعمل call من الـ netfilter hooks أو الـ socket hooks في الـ LSM modules زي SELinux.
- الـ function دي موجودة بس لو `CONFIG_AUDIT` enabled.

**Pseudocode:**
```
ipv4_skb_to_auditdata(skb, ad, proto):
    iph = ip_hdr(skb)
    if iph == NULL: return -EINVAL

    ad->u.net->v4info.saddr = iph->saddr
    ad->u.net->v4info.daddr = iph->daddr
    ad->u.net->family = AF_INET

    if proto != NULL: *proto = iph->protocol

    if iph->protocol == IPPROTO_TCP:
        th = tcp_hdr(skb)
        ad->u.net->sport = th->source
        ad->u.net->dport = th->dest
    elif iph->protocol == IPPROTO_UDP:
        uh = udp_hdr(skb)
        ad->u.net->sport = uh->source
        ad->u.net->dport = uh->dest

    return 0
```

---

#### `ipv6_skb_to_auditdata()`

```c
int ipv6_skb_to_auditdata(struct sk_buff *skb,
                           struct common_audit_data *ad,
                           u8 *proto);
```

**نفس منطق** `ipv4_skb_to_auditdata()` بالظبط لكن للـ IPv6. بيـ parse الـ IPv6 header ويملأ `ad->u.net->v6info` بالـ source/destination addresses، ويـ walk الـ extension headers للوصول للـ transport layer.

**الـ Parameters:**

| الـ Parameter | النوع | الشرح |
|---|---|---|
| `skb` | `struct sk_buff *` | الـ packet buffer — لازم IPv6 |
| `ad` | `struct common_audit_data *` | نفس الـ `common_audit_data` — الـ `ad->u.net` لازم initialized |
| `proto` | `u8 *` | output للـ next header protocol (بعد extension headers) |

**الـ Return Value:** `0` للنجاح، negative لو الـ skb مش IPv6 أو الـ header parsing فشل.

**Key Details:**
- بيستخدم `ipv6_hdr(skb)` للـ base header.
- بيـ walk الـ extension headers باستخدام `ipv6_skip_exthdr()` للوصول للـ transport protocol الفعلي.
- متاح بس لو `CONFIG_IPV6` enabled (wrapped في `IS_ENABLED(CONFIG_IPV6)`).
- زي الـ IPv4 version: no locking، read-only على الـ skb.
- بيتعمل call من نفس الـ LSM network hooks.

---

### Group 3: الـ Core Audit Emission Functions

الـ functions دي هي اللي بتبني وبترسل الـ audit records فعلياً للـ audit daemon.

---

#### `common_lsm_audit()`

```c
void common_lsm_audit(struct common_audit_data *a,
                       void (*pre_audit)(struct audit_buffer *, void *),
                       void (*post_audit)(struct audit_buffer *, void *));
```

**دي الـ function الرئيسية** في الـ header ده. بتأخذ الـ `common_audit_data` المعمول fill وبتنشئ audit record كامل وبترسله. بتدي الـ LSM المرونة إنه يضيف data قبل وبعد الـ generic data عن طريق الـ callback functions.

**الـ Parameters:**

| الـ Parameter | النوع | الشرح |
|---|---|---|
| `a` | `struct common_audit_data *` | الـ audit data المجمّع — الـ `type` بيحدد إيه اللي هيتـ log |
| `pre_audit` | `void (*)(struct audit_buffer *, void *)` | callback بيتعمل call **قبل** الـ generic data — بيتعمل pass الـ `audit_buffer` والـ pointer للـ `a` نفسها — ممكن تكون NULL |
| `post_audit` | `void (*)(struct audit_buffer *, void *)` | callback بيتعمل call **بعد** الـ generic data — نفس signature — ممكن تكون NULL |

**الـ Return Value:** void — ما بترجعش error.

**Key Details:**

- **الـ locking:** بتـ call `audit_log_start()` اللي بياخد الـ internal audit locks. الـ caller ما يحتاجش يعمل لocking على الـ `a` struct نفسه عادةً لأنه stack-allocated في الـ hook.
- **الـ rate limiting:** الـ audit subsystem نفسه بيعمل rate limiting — لو الـ audit buffer full، الـ record ممكن تتـ drop.
- **الـ GFP context:** بتـ call `audit_log_start()` اللي ممكن يـ sleep في certain contexts — لازم تتأكد إن الـ caller مش في atomic context لو الـ audit يحتاج allocation. في الـ practice، الـ LSM hooks بتتعمل call من process context.
- الـ `pre_audit` callback بيستخدمه SELinux مثلاً عشان يـ log الـ AVC denial data (SID, class, permissions) قبل الـ generic object info.
- الـ `post_audit` callback نادر الاستخدام لكن متاح للـ LSM-specific trailing data.
- لو `CONFIG_AUDIT` مش enabled، الـ function بتبقى **empty inline stub** — zero overhead.

**Pseudocode Flow:**

```
common_lsm_audit(a, pre_audit, post_audit):
    /* check if audit is enabled and event should be logged */
    if !audit_enabled: return

    ab = audit_log_start(current->audit_context,
                         GFP_KERNEL, AUDIT_AVC)
    if ab == NULL: return  /* rate-limited or OOM */

    /* 1. LSM-specific prefix data */
    if pre_audit != NULL:
        pre_audit(ab, a)

    /* 2. generic object data based on a->type */
    audit_log_lsm_data(ab, a)

    /* 3. LSM-specific suffix data */
    if post_audit != NULL:
        post_audit(ab, a)

    audit_log_end(ab)  /* sends record to audit daemon */
```

**مثال عملي — SELinux AVC denial:**

```c
/* في selinux/avc.c */
struct common_audit_data ad;
struct selinux_audit_data sad;

/* 1. initialize الـ generic data */
ad.type = LSM_AUDIT_DATA_PATH;
ad.u.path = file->f_path;

/* 2. initialize الـ SELinux-specific data */
sad.ssid = ssid;
sad.tsid = tsid;
sad.tclass = tclass;
sad.denied = denied_perms;
ad.selinux_audit_data = &sad;

/* 3. emit the record */
common_lsm_audit(&ad, avc_audit_pre_callback,
                      avc_audit_post_callback);
/* النتيجة في /var/log/audit/audit.log:
   type=AVC msg=audit(...): avc: denied { read }
   for pid=1234 comm="cat" path="/etc/shadow"
   dev="sda1" ino=12345 ... */
```

---

#### `audit_log_lsm_data()`

```c
void audit_log_lsm_data(struct audit_buffer *ab,
                         const struct common_audit_data *a);
```

**الـ function دي** هي اللي بتعمل فعلياً الـ serialization للـ `common_audit_data` في الـ audit buffer. بتـ switch على `a->type` وبتـ format الـ appropriate fields كـ key=value pairs في الـ audit record.

**الـ Parameters:**

| الـ Parameter | النوع | الشرح |
|---|---|---|
| `ab` | `struct audit_buffer *` | الـ buffer الجاري الكتابة فيه — ناتج من `audit_log_start()` |
| `a` | `const struct common_audit_data *` | الـ data اللي هتتـ serialize — const لأن الـ function بس بتقرأ |

**الـ Return Value:** void.

**Key Details:**

- **بتشتغل** كجزء من الـ `common_lsm_audit()` flow، لكن exposed كـ API مستقل عشان الـ LSM modules اللي عايزة تضيف data لـ audit buffer موجود من قبل (زي في الـ netlink audit events).
- بحسب الـ `a->type`، بتـ log data مختلفة:
  - `LSM_AUDIT_DATA_PATH`: بتـ log `path=` و `dev=` و `ino=`
  - `LSM_AUDIT_DATA_NET`: بتـ log `saddr=`, `daddr=`, `sport=`, `dport=`, `netif=`
  - `LSM_AUDIT_DATA_CAP`: بتـ log `capability=`
  - `LSM_AUDIT_DATA_IPC`: بتـ log `key=` للـ IPC object
  - `LSM_AUDIT_DATA_TASK`: بتـ log `pid=` و `comm=` للـ task
  - `LSM_AUDIT_DATA_KMOD`: بتـ log `kmod=` للـ module name
  - `LSM_AUDIT_DATA_INODE`: بتـ log `dev=` و `ino=`
  - `LSM_AUDIT_DATA_IBPKEY`: بتـ log `pkey=` و `subnet=`
  - `LSM_AUDIT_DATA_LOCKDOWN`: بتـ log `lockdown_reason=`
- لو `CONFIG_AUDIT` مش enabled، بتبقى **empty inline stub**.
- **لا تعمل** `audit_log_start()` أو `audit_log_end()` — دي مسؤولية الـ caller.

**Pseudocode Flow:**

```
audit_log_lsm_data(ab, a):
    switch a->type:
        case LSM_AUDIT_DATA_PATH:
            audit_log_format(ab, " path=")
            audit_log_untrustedstring(ab, a->u.path.dentry->d_name.name)
            /* log dev, ino from dentry */

        case LSM_AUDIT_DATA_NET:
            net = a->u.net
            if net->family == AF_INET:
                audit_log_format(ab,
                    " saddr=%pI4 daddr=%pI4 sport=%d dport=%d",
                    &net->v4info.saddr, &net->v4info.daddr,
                    ntohs(net->sport), ntohs(net->dport))
            elif net->family == AF_INET6:
                audit_log_format(ab,
                    " saddr=%pI6 daddr=%pI6 ...",
                    &net->v6info.saddr, ...)
            if net->netif:
                log interface name

        case LSM_AUDIT_DATA_CAP:
            audit_log_format(ab, " capability=%d", a->u.cap)

        case LSM_AUDIT_DATA_INODE:
            /* log dev:ino */

        case LSM_AUDIT_DATA_KMOD:
            audit_log_format(ab, " kmod=")
            audit_log_untrustedstring(ab, a->u.kmod_name)

        case LSM_AUDIT_DATA_IBPKEY:
            audit_log_format(ab, " pkey=0x%x subnet_prefix=%llx",
                             a->u.ibpkey->pkey,
                             a->u.ibpkey->subnet_prefix)

        /* ... etc */
```

---

### Group 4: الـ No-op Stubs (بدون CONFIG_AUDIT)

لما `CONFIG_AUDIT` مش enabled، الـ `common_lsm_audit()` و `audit_log_lsm_data()` بيتبقوا **empty static inline functions**:

```c
static inline void common_lsm_audit(struct common_audit_data *a,
    void (*pre_audit)(struct audit_buffer *, void *),
    void (*post_audit)(struct audit_buffer *, void *))
{
    /* intentionally empty */
}

static inline void audit_log_lsm_data(struct audit_buffer *ab,
                                        const struct common_audit_data *a)
{
    /* intentionally empty */
}
```

**الهدف:** zero overhead على الـ systems اللي مش بتستخدم audit — الـ compiler بيـ inline الـ empty functions وبيـ eliminate الـ call sites كلها. الـ LSM hook code اللي بيـ fill الـ `common_audit_data` ممكن يفضل موجود عادةً (لأن الـ struct نفسه مش guarded بـ CONFIG_AUDIT)، لكن الـ emit call بيتحول لـ no-op.

---

### الـ Big Picture — Data Flow

```
LSM Hook (e.g. selinux_file_permission)
    │
    ├─► ملأ common_audit_data على الـ stack
    │       ad.type = LSM_AUDIT_DATA_FILE
    │       ad.u.file = file
    │       ad.selinux_audit_data = &sad
    │
    └─► common_lsm_audit(&ad, pre_cb, post_cb)
            │
            ├─► audit_log_start()  ──► audit_buffer *ab
            │
            ├─► pre_cb(ab, &ad)   ──► "avc: denied { read } for"
            │
            ├─► audit_log_lsm_data(ab, &ad)
            │       switch(ad.type):
            │           FILE ──► path= dev= ino=
            │
            ├─► post_cb(ab, &ad)  ──► LSM-specific suffix
            │
            └─► audit_log_end(ab) ──► netlink msg ──► auditd
```

**الـ `ipv4/ipv6_skb_to_auditdata()`** بتتعمل call قبل الـ flow ده لما الـ event متعلق بـ network packet:

```
Network Hook (e.g. selinux_ip_postroute)
    │
    ├─► ipv4_skb_to_auditdata(skb, &ad, &proto)
    │       يملأ ad.u.net->v4info من الـ IP header
    │
    └─► common_lsm_audit(&ad, ...)
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بالـ LSM/Audit

الـ **audit subsystem** و الـ **LSM framework** بيكشفوا معلومات مهمة جوه `/sys/kernel/debug/` وكمان `/proc/`.

| المسار | المحتوى | كيفية القراءة |
|--------|---------|---------------|
| `/sys/kernel/debug/audit/` | بيانات الـ audit kernel-side | `ls /sys/kernel/debug/audit/` |
| `/proc/audit_rules` | الـ rules النشطة حالياً | `cat /proc/audit_rules` (deprecated — استخدم `auditctl -l`) |
| `/proc/net/netlink` | الـ netlink sockets للـ audit daemon | `cat /proc/net/netlink \| grep -i audit` |
| `/sys/kernel/security/` | الـ LSM security attributes | `ls /sys/kernel/security/` |
| `/sys/kernel/security/lsm` | الـ LSMs المحملين حالياً | `cat /sys/kernel/security/lsm` |

```bash
# شوف الـ LSMs المفعلين
cat /sys/kernel/security/lsm

# مثال output
selinux,lockdown,capability

# تأكد إن audit daemon شغال ومتصل بالـ kernel
cat /proc/net/netlink | head -5
# Output:
# sk               Eth Pid    Groups   Rmem     Wmem     Dump     Locks     Drops     Inode
# ffff...          9   1234   00000003 0        0        0        2         0         12345
```

---

#### 2. مدخلات الـ sysfs المتعلقة بالـ LSM/Audit

| المسار | الوصف |
|--------|-------|
| `/sys/kernel/security/lsm` | قائمة الـ LSMs المحملين بالترتيب |
| `/sys/fs/selinux/enforce` | حالة الـ SELinux (enforcing/permissive) |
| `/sys/fs/selinux/deny_unknown` | سلوك الـ unknown contexts |
| `/sys/fs/selinux/status` | `mmap`-able status page للـ userspace |
| `/sys/fs/smackfs/load` | الـ Smack rules المحملة |
| `/sys/fs/smackfs/cipso` | الـ CIPSO labels |
| `/sys/fs/apparmor/profiles` | الـ AppArmor profiles المحملة |
| `/sys/kernel/tracing/events/syscalls/` | syscall tracepoints للـ audit |

```bash
# SELinux: شوف الـ enforce mode
cat /sys/fs/selinux/enforce
# 1 = enforcing, 0 = permissive

# AppArmor: شوف الـ profiles المحملة
cat /sys/kernel/security/apparmor/profiles

# تأكد إن الـ audit enabled في الـ kernel
cat /proc/sys/kernel/audit
# أو
sysctl kernel.audit
```

---

#### 3. استخدام الـ ftrace مع LSM/Audit

الـ **ftrace** بيسمح بتتبع الـ LSM hooks و audit events بشكل دقيق.

##### تفعيل الـ Tracepoints المتعلقة بالـ Audit

```bash
# اعرف الـ events المتاحة للـ audit
ls /sys/kernel/tracing/events/audit/

# audit:audit_rule_change     - تغييرات الـ rules
# audit:audit_config_change   - تغييرات الـ config
# audit:audit_log_start       - بداية الـ log record

# فعّل كل الـ audit events
echo 1 > /sys/kernel/tracing/events/audit/enable

# أو فعّل event معين
echo 1 > /sys/kernel/tracing/events/audit/audit_rule_change/enable

# اقرأ الـ trace buffer
cat /sys/kernel/tracing/trace
```

##### تتبع الـ LSM Hook Functions مباشرة

```bash
# ابحث عن الـ LSM security hooks المتاحة للـ tracing
grep -r "lsm_audit\|common_lsm_audit" /sys/kernel/tracing/available_filter_functions 2>/dev/null

# فعّل الـ function tracer على common_lsm_audit
echo common_lsm_audit > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل العملية اللي بتختبرها، بعدين
cat /sys/kernel/tracing/trace | head -50

# وقّف الـ tracing
echo 0 > /sys/kernel/tracing/tracing_on
echo nop > /sys/kernel/tracing/current_tracer
```

##### تتبع الـ LSM Security Checks بالـ function_graph

```bash
# استخدم function_graph لشوف الـ call stack كاملاً
echo function_graph > /sys/kernel/tracing/current_tracer
echo 'security_*' > /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on

# مثال output
# 1) | security_file_open() {
# 1) | selinux_file_open() {
# 1) | avc_has_perm() {
# 1)   0.450 us    | avc_lookup();
# 1)   1.234 us    | }
```

---

#### 4. الـ printk و Dynamic Debug للـ LSM/Audit

##### تفعيل الـ Dynamic Debug

```bash
# فعّل الـ debug messages لملف lsm_audit.c
echo 'file security/lsm_audit.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لكل الـ security subsystem
echo 'module selinux +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module apparmor +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug entries النشطة
cat /sys/kernel/debug/dynamic_debug/control | grep security

# فعّل مع الـ stack trace (+ps)
echo 'file security/lsm_audit.c +ps' > /sys/kernel/debug/dynamic_debug/control
```

##### الـ printk Levels المفيدة

```bash
# غيّر الـ loglevel لشوف الـ KERN_DEBUG messages
dmesg -n 7
# أو
echo 7 > /proc/sys/kernel/printk

# تابع الـ audit messages في real-time
dmesg -w | grep -E 'audit:|avc:|lsm:'

# مثال output للـ SELinux AVC denial
# audit: type=1400 audit(1700000000.123:456): avc:  denied  { read } for
#   pid=1234 comm="myapp" name="secret.txt" dev="sda1" ino=98765
#   scontext=user_u:user_r:user_t:s0
#   tcontext=system_u:object_r:secret_file_t:s0
#   tclass=file permissive=0
```

---

#### 5. الـ Kernel Config Options المهمة للـ Debugging

| الخيار | الوصف |
|--------|-------|
| `CONFIG_AUDIT` | تفعيل الـ audit framework نفسه — **أساسي** |
| `CONFIG_AUDIT_GENERIC` | الـ audit على platforms غير x86 |
| `CONFIG_AUDITSYSCALL` | تتبع الـ system calls عبر audit |
| `CONFIG_SECURITY` | الـ LSM framework الأساسي |
| `CONFIG_SECURITY_LSM_DEBUG` | debug messages للـ LSM core |
| `CONFIG_SECURITY_SELINUX_DEBUG` | debug إضافي لـ SELinux |
| `CONFIG_SECURITY_SELINUX_DEVELOP` | يسمح التبديل بين enforcing/permissive runtime |
| `CONFIG_SECURITY_SMACK_BRINGUP` | permissive mode لـ Smack rules معينة |
| `CONFIG_SECURITY_APPARMOR_DEBUG` | debug messages لـ AppArmor |
| `CONFIG_DEBUG_LIST` | اكتشاف الـ list corruption في الـ audit rules |
| `CONFIG_LOCKDEP` | اكتشاف الـ deadlocks في الـ LSM spinlocks |
| `CONFIG_KASAN` | اكتشاف الـ memory corruption في الـ audit buffers |
| `CONFIG_PROVE_LOCKING` | تحقق من صحة الـ locking |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_AUDIT|CONFIG_SECURITY'
# أو
grep -E 'CONFIG_AUDIT|CONFIG_SECURITY' /boot/config-$(uname -r)
```

---

#### 6. أدوات الـ Audit Subsystem المتخصصة

##### الأداة الرئيسية: auditctl

```bash
# شوف الـ audit rules الحالية
auditctl -l

# شوف الـ audit status
auditctl -s
# Output:
# enabled 1
# failure 1
# pid 1234
# rate_limit 0
# backlog_limit 8192
# lost 0
# backlog 0
# backlog_wait_time 15000000

# شوف الـ audit events مباشرة
ausearch -ts today | head -30

# فلتر على AVC denials (SELinux)
ausearch -m avc -ts today

# فلتر على LSM events
ausearch -m MAC_POLICY_LOAD,MAC_STATUS -ts today
```

##### أدوات SELinux المتخصصة

```bash
# شوف الـ AVC denials واقتراح الـ policy
audit2allow -a

# شوف الـ context الخاص بملف
ls -Z /path/to/file
stat -c '%C' /path/to/file  # security context

# شوف الـ context الخاص بـ process
ps -eZ | grep myapp

# تحقق من الـ policy المحملة
seinfo -t | head -20  # types
seinfo -r             # roles
seinfo -u             # users

# شوف الـ access vector cache
cat /sys/fs/selinux/avc/cache_stats
# lookups:   1234567
# hits:      1200000
# misses:    34567
# allocations: 34567
# reclaims:  34000
# frees:     34000
```

##### أدوات AppArmor المتخصصة

```bash
# شوف الـ profiles المحملة وحالتها
aa-status

# مثال output
# 34 profiles are loaded.
# 34 profiles are in enforce mode.
# 0 profiles are in complain mode.

# شوف الـ AppArmor log events
grep 'apparmor="DENIED"' /var/log/kern.log | tail -20

# حوّل إلى complain mode لـ debugging
aa-complain /usr/bin/myapp
```

---

#### 7. جدول الـ Error Messages الشائعة

| الرسالة | المعنى | الحل |
|---------|--------|------|
| `avc: denied { read } for pid=...` | SELinux رفض عملية read | استخدم `audit2allow` لتوليد الـ policy rule المناسب |
| `apparmor="DENIED" operation="file_perm"` | AppArmor رفض الوصول | غيّر الـ profile لـ complain mode وأضف الـ rule المناسب |
| `audit: backlog limit exceeded` | الـ audit buffer امتلأ | زوّد `backlog_limit` عبر `auditctl -b 16384` |
| `audit: audit_lost=N` | ضاعت N رسائل audit | زوّد الـ buffer أو قلل الـ rate limit |
| `audit: type=1400` | LSM denial event (SELinux AVC) | راجع الـ scontext و tcontext في الـ message |
| `audit: type=1400 permissive=1` | SELinux في permissive mode — تسجيل فقط | إما ده متعمد أو الـ SELinux مش enforcing |
| `audit: type=1700` | تغيير في الـ config | شوف أي عملية غيّرت الـ audit config |
| `lsm: security_<hook>: -EACCES` | رفض LSM hook عام | فعّل dynamic debug وشوف أي LSM رفض |
| `key lookup failed` | فشل الـ key في `LSM_AUDIT_DATA_KEY` | تحقق من الـ keyring والـ permissions |
| `netlabel: labeling failure` | مشكلة في الـ NetLabel/CIPSO labeling | راجع `netlabelctl` output |

---

#### 8. مواضع الـ dump_stack() و WARN_ON() الاستراتيجية

الـ **`common_lsm_audit()`** هي نقطة التجمع الرئيسية — أي denial بيمر منها قبل ما يتسجل.

```c
/* في security/lsm_audit.c — نقطة مراقبة رئيسية */
void common_lsm_audit(struct common_audit_data *a,
    void (*pre_audit)(struct audit_buffer *, void *),
    void (*post_audit)(struct audit_buffer *, void *))
{
    /* هنا: أضف WARN_ON لتتبع من بيطلب الـ audit */
    WARN_ON(!a);  /* يجب أن يكون a صحيحاً دائماً */

    /* لو محتاج stack trace لكل denial */
    if (a->type == LSM_AUDIT_DATA_PATH)
        dump_stack();  /* أضف مؤقتاً لـ debugging */
    ...
}

/* نقطة مراقبة في ipv4_skb_to_auditdata */
int ipv4_skb_to_auditdata(struct sk_buff *skb,
    struct common_audit_data *ad, u8 *proto)
{
    /* WARN_ON لو الـ skb بدون header */
    WARN_ON(!skb_network_header(skb));
    ...
}
```

##### مواضع استراتيجية للـ WARN_ON

```c
/* 1. تحقق من نوع الـ audit data صحيح */
WARN_ON(a->type < LSM_AUDIT_DATA_PATH || a->type > LSM_AUDIT_DATA_NLMSGTYPE);

/* 2. تحقق من الـ net audit data مش NULL لما النوع NET */
WARN_ON(a->type == LSM_AUDIT_DATA_NET && !a->u.net);

/* 3. تحقق من الـ op data مش NULL لما النوع IOCTL_OP */
WARN_ON(a->type == LSM_AUDIT_DATA_IOCTL_OP && !a->u.op);

/* 4. في الـ LSM hook نفسه — تحقق من الـ context */
WARN_ON_ONCE(in_interrupt() && audit_enabled);
```

---

### Policy/Configuration Level Debugging
*(بديل الـ Hardware Level — لأن الـ lsm_audit.h هو security subsystem header)*

---

#### 1. التحقق من حالة الـ LSM Policy تتطابق مع الـ Kernel State

```bash
# SELinux: قارن الـ policy المحملة مع المتوقع
sestatus -v
# Output:
# SELinux status:                 enabled
# SELinuxfs mount:                /sys/fs/selinux
# SELinux mount point:            /sys/fs/selinux
# Current mode:                   enforcing
# Mode from config file:          enforcing
# Policy MLS status:              enabled
# Policy deny_unknown status:     denied
# Memory protection checking:     actual (secure)
# Max kernel policy version:      33

# تحقق من الـ policy hash (هل اتغيرت؟)
cat /sys/fs/selinux/policyvers
sha256sum /sys/fs/selinux/policy

# AppArmor: تحقق من الـ profiles المحملة
cat /sys/kernel/security/apparmor/profiles | sort

# Smack: تحقق من الـ rules
cat /sys/fs/smackfs/load2 | head -20
```

---

#### 2. تفريغ الـ Audit Rules و Policy State

##### dump الـ Audit Rules

```bash
# صدّر الـ rules الحالية
auditctl -l
# مثال output:
# -w /etc/passwd -p wa -k passwd_changes
# -a always,exit -F arch=b64 -S open -F auid>=1000 -k user_file_access

# صدّر الـ rules بصيغة قابلة للحفظ
auditctl -l > /tmp/current-audit-rules.txt

# احصل على raw audit policy
augenrules --check 2>&1

# شوف الـ SELinux policy rules لـ type معين
sesearch --allow -t shadow_t -s sshd_t
```

##### تفريغ الـ SELinux AVC State

```bash
# شوف الـ AVC cache بالتفصيل
cat /sys/fs/selinux/avc/cache_stats

# تفريغ الـ AVC entries (kernel log)
echo 1 > /sys/fs/selinux/avc/cache_threshold  # lower threshold to force dump

# احصل على الـ security contexts لكل process
ps auxZ | awk '{print $1, $2, $NF}' | head -20
```

---

#### 3. تقنيات الـ Audit Log Analysis

##### تتبع الـ LSM_AUDIT_DATA_NET events

```bash
# شوف الـ network-related denials (lsm_network_audit)
ausearch -m avc -ts today | grep 'netif\|sport\|dport'

# مثال output يعكس struct lsm_network_audit
# type=AVC msg=audit(1700000000.123:789): avc: denied { node_bind } for
#   pid=5678 comm="myserver"
#   laddr=192.168.1.100 lport=8080
#   scontext=system_u:system_r:myserver_t:s0
#   tcontext=system_u:object_r:node_t:s0 tclass=tcp_socket

# شوف الـ IOCTL-related denials (lsm_ioctlop_audit)
ausearch -m avc | grep 'ioctl\|ioctlcmd'
```

##### استخدام aureport للتحليل

```bash
# ملخص كل الـ events
aureport --summary

# أكثر الـ files طلباً للـ access
aureport -f --summary

# أكثر الـ users تسبباً في denials
aureport -u --failed --summary

# timeline للـ LSM events
aureport -l -ts today -te now
```

---

#### 4. مشاكل الـ LSM Policy الشائعة وأنماطها في الـ Kernel Log

| المشكلة | النمط في الـ Log | التشخيص |
|---------|----------------|---------|
| Context مش موجود في الـ policy | `avc: denied { } scontext=unconfined_u` | الـ context مش معرّف في الـ policy |
| ملف مش مـ labeled | `tcontext=system_u:object_r:unlabeled_t` | الـ filesystem محتاج `restorecon` |
| Policy قديمة مش متزامنة | `netlabel: domain lookup failed` | أعد تحميل الـ policy |
| Audit daemon مش شغال | لا logs في `/var/log/audit/` | `systemctl status auditd` |
| Backlog فاض | `audit: backlog limit exceeded` | زوّد `backlog_limit` في `auditd.conf` |
| SELinux في wrong mode | `permissive=1` في كل الـ denials | `setenforce 1` لـ enforce |
| IB (InfiniBand) pkey denial | `lsm_ibpkey_audit: pkey=0xffff` | راجع الـ IB security policy |

```bash
# مثال شامل: تشخيص لماذا process اتمنع
# 1. ابدأ بـ ausearch
ausearch -c "myapp" -m avc -ts today

# 2. لو مفيش log، تحقق إن audit شغال
systemctl status auditd
auditctl -s | grep 'enabled\|pid'

# 3. تحقق من الـ SELinux mode
getenforce

# 4. لو permissive لكن بتشتغل
# غيّر enforcing وراقب
setenforce 1
dmesg -w | grep avc &
./myapp
```

---

#### 5. تتبع الـ LSM Policy Loading و Changes

```bash
# راقب تغييرات الـ audit policy في real-time
ausearch -m AUDIT_CONFIG_CHANGE,MAC_POLICY_LOAD,MAC_STATUS -ts recent

# مثال output لما SELinux policy اتحملت
# type=MAC_POLICY_LOAD msg=audit(1700000000.456:123):
#   policy loaded auid=0 ses=1

# مثال لما SELinux mode اتغير
# type=MAC_STATUS msg=audit(1700000000.789:124):
#   enforcing=1 old_enforcing=0 auid=0 ses=1

# شوف تاريخ تغييرات الـ audit config
aureport --config --summary -ts 'this-month'
```

---

### Practical Commands — Ready to Copy

---

#### Script شامل لتشخيص LSM/Audit سريع

```bash
#!/bin/bash
# lsm-audit-debug.sh — Quick LSM/Audit Diagnostic Script

echo "=== LSM Stack ==="
cat /sys/kernel/security/lsm

echo -e "\n=== Audit Status ==="
auditctl -s 2>/dev/null || echo "auditd not running or auditctl not available"

echo -e "\n=== SELinux Status ==="
if [ -f /sys/fs/selinux/enforce ]; then
    mode=$(cat /sys/fs/selinux/enforce)
    echo "SELinux: $([ $mode -eq 1 ] && echo ENFORCING || echo PERMISSIVE)"
    cat /sys/fs/selinux/avc/cache_stats
else
    echo "SELinux not enabled"
fi

echo -e "\n=== AppArmor Status ==="
if [ -f /sys/module/apparmor/parameters/enabled ]; then
    aa-status --profiled 2>/dev/null | head -5
else
    echo "AppArmor not enabled"
fi

echo -e "\n=== Recent AVC Denials (last 10) ==="
ausearch -m avc -ts recent 2>/dev/null | tail -40

echo -e "\n=== Audit Lost Events ==="
auditctl -s 2>/dev/null | grep 'lost\|backlog'

echo -e "\n=== Active Audit Rules ==="
auditctl -l 2>/dev/null

echo -e "\n=== Kernel Debug Config ==="
grep -E 'CONFIG_AUDIT|CONFIG_SECURITY' /boot/config-$(uname -r) 2>/dev/null | grep '=y'
```

```bash
# تشغيل الـ script
chmod +x lsm-audit-debug.sh && sudo ./lsm-audit-debug.sh
```

---

#### تتبع الـ common_lsm_audit بالـ ftrace

```bash
#!/bin/bash
# trace-lsm-audit.sh — Trace LSM audit function calls

TRACE_DIR=/sys/kernel/tracing

# صفّر الـ trace
echo 0 > $TRACE_DIR/tracing_on
echo > $TRACE_DIR/trace

# اضبط الـ filter
echo 'common_lsm_audit' > $TRACE_DIR/set_ftrace_filter 2>/dev/null || \
echo "WARNING: common_lsm_audit not traceable (may be inlined)"

# شغّل الـ function tracer
echo function > $TRACE_DIR/current_tracer
echo 1 > $TRACE_DIR/tracing_on

echo "Tracing for 5 seconds..."
sleep 5

echo 0 > $TRACE_DIR/tracing_on

echo "=== LSM Audit Trace ==="
cat $TRACE_DIR/trace | grep -v '^#' | head -50

# تنظيف
echo nop > $TRACE_DIR/current_tracer
echo > $TRACE_DIR/set_ftrace_filter
```

---

#### مثال تفسير audit record كامل

```bash
# مثال record من ausearch:
# ----
# time->Thu Nov 23 12:00:00 2023
# type=AVC msg=audit(1700000000.000:500):
#   avc:  denied  { write } for
#   pid=2345
#   comm="nginx"
#   path="/var/log/myapp/app.log"
#   dev="sda1"
#   ino=123456
#   scontext=system_u:system_r:httpd_t:s0
#   tcontext=system_u:object_r:var_log_t:s0
#   tclass=file
#   permissive=0
# ----

# التفسير:
# - comm="nginx"         → العملية المخطئة
# - path=...app.log      → الملف المطلوب (lsm_audit DATA_PATH)
# - scontext=httpd_t     → context الـ subject (من LSM_AUDIT_DATA_TASK)
# - tcontext=var_log_t   → context الـ object (الملف)
# - tclass=file          → نوع الـ object
# - permissive=0         → تم رفض الطلب فعلاً (مش مجرد log)

# الحل المقترح:
audit2allow -a -M nginx-myapp
semodule -i nginx-myapp.pp
```

---

#### تفعيل الـ Audit Events بناءً على نوع الـ common_audit_data

```bash
# LSM_AUDIT_DATA_NET (2) — مشاكل الشبكة
auditctl -a always,exit -F arch=b64 -S connect -S bind -S accept -k lsm_net

# LSM_AUDIT_DATA_PATH (1) — مشاكل الملفات
auditctl -a always,exit -F arch=b64 -S open -S openat -F auid>=1000 -k lsm_path

# LSM_AUDIT_DATA_CAP (3) — مشاكل الـ capabilities
auditctl -a always,exit -F arch=b64 -S capset -S setuid -k lsm_cap

# LSM_AUDIT_DATA_KEY (6) — مشاكل الـ keyrings
auditctl -a always,exit -F arch=b64 -S add_key -S request_key -k lsm_key

# LSM_AUDIT_DATA_IPC (4) — مشاكل الـ IPC
auditctl -a always,exit -F arch=b64 -S msgget -S semget -S shmget -k lsm_ipc

# شوف الـ rules المضافة
auditctl -l | grep 'lsm_'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — SELinux بيبلوك UART Communication

#### العنوان
SELinux denial بيمنع daemon صناعي من فتح `/dev/ttyS2` على gateway مبني على AM62x.

#### السياق
شركة بتبني **industrial gateway** بيشغّل Modbus RTU فوق UART على TI AM62x. الـ gateway بيتكلم مع PLCs في مصنع. الـ OS هو Yocto-based Linux مع SELinux enabled في **enforcing mode**.

#### المشكلة
بعد update للـ SELinux policy، الـ `modbusd` daemon وقف عن الكلام مع الـ PLC. الـ `/dev/ttyS2` بقى مش accessible. الـ daemon بيرجع `EACCES` من `open()`.

#### التحليل
الـ kernel بيمشي من `security_file_open()` ← LSM hook ← SELinux بيعمل `avc_has_perm()`. لما الـ permission مش موجودة في policy، SELinux بيكلم `common_lsm_audit()` في `security/lsm_audit.c` اللي بيستخدم structures من `lsm_audit.h`:

```c
/* الـ audit record بيتبني هنا */
struct common_audit_data ad;
ad.type = LSM_AUDIT_DATA_PATH;  /* type = 1 */
ad.u.path = file->f_path;       /* path لـ /dev/ttyS2 */
```

`common_lsm_audit()` بيتحقق إن `CONFIG_AUDIT` enabled، وبعدين بيبني الـ audit record عبر `audit_log_lsm_data()`. الـ record بيظهر في kernel log بيوضح الـ denial.

```bash
# الـ denial بيظهر كده في /var/log/audit/audit.log
type=AVC msg=audit(...): avc: denied { open } for
  pid=312 comm="modbusd"
  path="/dev/ttyS2"
  dev="tmpfs" ino=47
  scontext=system_u:system_r:modbus_t:s0
  tcontext=system_u:object_r:tty_device_t:s0
  tclass=chr_file permissive=0
```

الـ `ad.type = LSM_AUDIT_DATA_PATH` هو اللي بيخلي `audit_log_lsm_data()` يطبع الـ `path` من `ad.u.path`.

#### الحل

```bash
# 1. استخرج الـ denial واعمل policy module
ausearch -m avc -ts recent | audit2allow -M modbus_uart

# 2. راجع الـ module الناتج
cat modbus_uart.te
# allow modbus_t tty_device_t:chr_file { open read write ioctl };

# 3. install الـ module
semodule -i modbus_uart.pp

# 4. تحقق إن الـ denial اتحل
runcon system_u:system_r:modbus_t:s0 cat /dev/ttyS2
```

#### الدرس المستفاد
الـ `LSM_AUDIT_DATA_PATH` في `common_audit_data.type` هو اللي بيحدد إيه اللي بيتطبع في الـ audit record. فهم الـ `ad.type` بيساعدك تفسر الـ denial بسرعة وتعمل policy صح.

---

### السيناريو 2: Android TV Box على Allwinner H616 — AppArmor بيبلوك Widevine HDMI

#### العنوان
AppArmor denial بيمنع Widevine DRM daemon من الوصول لـ HDMI ioctl على H616 TV box.

#### السياق
منتج **Android TV box** بيشغّل custom Linux مع AppArmor. الـ SoC هو Allwinner H616. الـ Widevine DRM daemon محتاج يعمل ioctl على `/dev/dri/card0` عشان يعمل HDCP handshake مع الشاشة عبر HDMI.

#### المشكلة
المحتوى الـ protected بيفشل في العرض. الـ Widevine بيرجع خطأ `EACCES` من ioctl call. الكاميرا والصوت شغالين، المشكلة في HDMI فقط.

#### التحليل
لما `wvd` process بيعمل `ioctl(fd, DRM_IOCTL_MODE_GETRESOURCES, ...)` على `/dev/dri/card0`، الـ LSM hook `security_file_ioctl()` بيتفعل. AppArmor بيرفض الـ operation وبيكلم `common_lsm_audit()`.

الـ `common_audit_data` بيتبني بـ:

```c
/* في كود AppArmor */
struct common_audit_data ad;
struct lsm_ioctlop_audit ioctlop;

ad.type = LSM_AUDIT_DATA_IOCTL_OP;  /* type = 11 */
ioctlop.path = file->f_path;         /* /dev/dri/card0 */
ioctlop.cmd  = DRM_IOCTL_MODE_GETRESOURCES;
ad.u.op = &ioctlop;
```

الـ `audit_log_lsm_data()` لما بيشوف `LSM_AUDIT_DATA_IOCTL_OP` بيطبع:
- الـ `path` من `ioctlop.path`
- الـ `cmd` hex value

```bash
# الـ denial في /var/log/kern.log
kernel: audit: type=1400 audit(...): apparmor="DENIED"
  operation="ioctl" profile="/usr/bin/wvd"
  name="/dev/dri/card0"
  requested_mask="ioctl"
  denied_mask="ioctl" cmd=0xb64d
```

الـ `cmd=0xb64d` هو `DRM_IOCTL_MODE_GETRESOURCES`.

#### الحل

```bash
# 1. اعرف الـ profile المستخدم
aa-status | grep wvd

# 2. عدّل الـ AppArmor profile
cat >> /etc/apparmor.d/usr.bin.wvd << 'EOF'
  /dev/dri/card0 rw,
  /dev/dri/card0 ioctl -> {
    DRM_IOCTL_MODE_GETRESOURCES,
    DRM_IOCTL_MODE_SETCRTC,
  },
EOF

# 3. reload الـ profile
apparmor_parser -r /etc/apparmor.d/usr.bin.wvd

# 4. اختبر
wvd --test-hdcp
```

#### الدرس المستفاد
الـ `LSM_AUDIT_DATA_IOCTL_OP` و`struct lsm_ioctlop_audit` في `lsm_audit.h` بيخليك تعرف بالضبط أي ioctl command اتمنع. الـ `cmd` field في الـ audit log هو مفتاح التشخيص.

---

### السيناريو 3: IoT Sensor على STM32MP1 — SELinux بيبلوك Network Socket لـ MQTT

#### العنوان
SELinux بيمنع MQTT client من الاتصال بـ broker على port 1883 عبر WiFi على STM32MP1.

#### السياق
**IoT sensor node** مبني على STM32MP1 بيقرأ temperature و humidity وبيبعتها لـ cloud عبر MQTT. الـ OS هو Buildroot مع SELinux. بعد تشديد الـ policy، الـ sensor وقف عن البعت.

#### المشكلة
الـ `mosquitto_pub` بيفشل بـ `Connection refused` أو `EACCES`. الـ WiFi متصل والـ broker شغال. الـ socket creation نفسها بتنجح لكن `connect()` بيفشل.

#### التحليل
لما الـ process بيعمل `connect()` على TCP socket، الـ hook `security_socket_connect()` بيتفعل. SELinux بيرفض، وبيبني:

```c
struct common_audit_data ad;
struct lsm_network_audit net;

ad.type = LSM_AUDIT_DATA_NET;   /* type = 2 */
net.sk     = sk;                 /* الـ socket */
net.family = AF_INET;
net.v4info.daddr = broker_ip;   /* 192.168.1.100 */
net.dport  = htons(1883);
net.sport  = htons(ephemeral);
ad.u.net = &net;
```

`audit_log_lsm_data()` لما بيشوف `LSM_AUDIT_DATA_NET` بيطبع:
- الـ `daddr`, `saddr`, `dport`, `sport`
- الـ `netif` (interface index)

```bash
# الـ denial
type=AVC msg=audit(...): avc: denied { name_connect } for
  pid=441 comm="mosquitto_pub"
  dest=1883
  scontext=system_u:system_r:iot_sensor_t:s0
  tcontext=system_u:object_r:mqtt_port_t:s0
  tclass=tcp_socket permissive=0
```

#### الحل

```bash
# 1. تحقق إن الـ port معرّف في SELinux
semanage port -l | grep 1883
# لو مش موجود:
semanage port -a -t mqtt_port_t -p tcp 1883

# 2. أضف الـ rule لـ iot_sensor_t
cat > iot_mqtt.te << 'EOF'
module iot_mqtt 1.0;
require {
    type iot_sensor_t;
    type mqtt_port_t;
    class tcp_socket { name_connect };
}
allow iot_sensor_t mqtt_port_t:tcp_socket name_connect;
EOF

checkmodule -M -m -o iot_mqtt.mod iot_mqtt.te
semodule_package -o iot_mqtt.pp -m iot_mqtt.mod
semodule -i iot_mqtt.pp

# 3. تحقق من الاتصال
mosquitto_pub -h 192.168.1.100 -t sensor/temp -m "25.3"
```

#### الدرس المستفاد
الـ `struct lsm_network_audit` في `lsm_audit.h` بيخزن كل معلومات الـ network connection اللي بيتمنع. الـ `dport` و`daddr` في الـ audit log بيوضحوا بالضبط أي connection اتمنع، وده بيفرق كتير في debugging الـ network policy على embedded devices.

---

### السيناريو 4: Automotive ECU على i.MX8 — SELinux بيبلوك Kernel Module Load

#### العنوان
SELinux بيمنع vehicle CAN bus driver module من الـ load على i.MX8 ECU في بيئة AUTOSAR.

#### السياق
**Automotive ECU** مبني على i.MX8QM بيشغّل Linux مع SELinux في enforcing mode. الـ ECU محتاج يـ load `flexcan.ko` عند الـ boot عشان يتكلم مع الـ CAN bus للسيارة. الـ module بيتـ load بواسطة `modprobe` من init script.

#### المشكلة
الـ `modprobe flexcan` بيفشل صامت. الـ CAN interfaces مش موجودة. الـ `ip link show` مش بيظهر `can0`.

#### التحليل
الـ `modprobe` بيستخدم `finit_module()` syscall. الـ hook `security_kernel_module_from_file()` بيتفعل. لما SELinux يرفض:

```c
struct common_audit_data ad;
ad.type = LSM_AUDIT_DATA_KMOD;   /* type = 8 */
ad.u.kmod_name = "flexcan";      /* اسم الـ module */
```

`audit_log_lsm_data()` لما بيشوف `LSM_AUDIT_DATA_KMOD` بيطبع اسم الـ module مباشرة. الـ denial:

```bash
type=AVC msg=audit(...): avc: denied { module_load } for
  pid=89 comm="modprobe"
  kmod="flexcan"
  scontext=system_u:system_r:init_t:s0
  tcontext=system_u:system_r:kernel_t:s0
  tclass=system permissive=0
```

في بيئة automotive، الـ `LSM_AUDIT_DATA_KMOD` مهم لأن الـ module loading policy لازم تكون strict جداً — بس الـ CAN driver ضروري.

#### الحل

```bash
# 1. اعمل policy تسمح لـ init_t يلود flexcan
cat > can_module.te << 'EOF'
module can_module 1.0;
require {
    type init_t;
    type kernel_t;
    class system { module_load };
}
allow init_t kernel_t:system module_load;
EOF

# 2. بدل عن كده، استخدم labeled module loading
# ضيف file context لـ flexcan.ko
semanage fcontext -a -t modules_object_t \
    "/lib/modules/.*flexcan\.ko"
restorecon -v /lib/modules/$(uname -r)/kernel/drivers/net/can/flexcan.ko

# 3. load الـ module مع checking
modprobe flexcan
ip link show can0

# 4. على automotive systems، فعّل الـ module signing بدلاً من رفعه في policy
# CONFIG_MODULE_SIG=y في kernel config
```

#### الدرس المستفاد
الـ `LSM_AUDIT_DATA_KMOD` بيخلي الـ audit log يوضح اسم الـ module بالظبط. في automotive، الـ module loading policy لازم تكون دقيقة — مش `allow_module_load` global، بل per-module. الـ audit trail مهم للـ functional safety requirements زي ASIL.

---

### السيناريو 5: Custom Board Bring-up على RK3562 — SELinux Lockdown يمنع Raw I2C Access

#### العنوان
SELinux Lockdown mechanism بيمنع engineer من debugging I2C على RK3562 board في bring-up phase.

#### السياق
Engineer بيعمل **board bring-up** لـ custom hardware مبني على RK3562. الـ board عندها I2C PMIC و EEPROM. الـ engineer بيحاول يستخدم `i2cdetect` و `i2cdump` من userspace عشان يتحقق من الـ I2C devices. الـ kernel compiled مع `CONFIG_SECURITY_LOCKDOWN_LSM=y`.

#### المشكلة
```bash
$ i2cdetect -y 2
Error: Could not open file '/dev/i2c-2': Permission denied
$ i2cdump -y 2 0x36
Error: Could not open file '/dev/i2c-2': Permission denied
```

حتى root مش شغال. الـ `ls -la /dev/i2c-2` بيظهر permissions صح.

#### التحليل
الـ Lockdown LSM — مش SELinux أو AppArmor — هو اللي بيمنع الـ access. الـ hook `security_locked_down()` بيتفعل لما process يحاول يفتح raw I2C device. الـ Lockdown بيبني:

```c
struct common_audit_data ad;
ad.type = LSM_AUDIT_DATA_LOCKDOWN;  /* type = 15 */
ad.u.reason = LOCKDOWN_DEV_MEM;     /* أو LOCKDOWN_RAWIO */
```

الـ `audit_log_lsm_data()` لما بيشوف `LSM_AUDIT_DATA_LOCKDOWN` بيطبع:

```bash
# في kernel log
kernel: Lockdown: i2cdetect: raw I2C access is restricted;
        see man kernel_lockdown.7
type=ANOM_ABEND msg=audit(...):
  op=lockdown-test reason=raw-i2c-access
  scontext=unconfined_u:unconfined_r:unconfined_t:s0
```

الـ `ad.u.reason` بيتحول لـ string في `audit_log_lsm_data()` ويتطبع في الـ audit log.

#### الحل
خيارات مرتبة من الأسهل للأكثر أماناً:

```bash
# الخيار 1: للـ bring-up فقط — disable lockdown مؤقتاً
# في kernel cmdline:
# lockdown=none

# الخيار 2: compile بدون lockdown للـ development build
# CONFIG_SECURITY_LOCKDOWN_LSM=n

# الخيار 3: استخدم kernel module للـ I2C access بدلاً من userspace raw
# اكتب character driver بسيط يعرض PMIC registers

# الخيار 4: استخدم debugfs لو PMIC driver موجود
ls /sys/bus/i2c/devices/2-0036/
cat /sys/bus/i2c/devices/2-0036/voltage

# الخيار 5: في production، استخدم signed I2C access tool مع capability
# CAP_SYS_RAWIO بيعدي الـ lockdown check في بعض الإعدادات
```

```c
/* إثبات: الـ lockdown check في kernel */
/* security/lockdown/lockdown.c */
static int lockdown_is_locked_down(enum lockdown_reason what)
{
    if (kernel_locked_down >= what) {
        /* هنا بيتبني الـ common_audit_data بـ LSM_AUDIT_DATA_LOCKDOWN */
        return -EPERM;
    }
    return 0;
}
```

#### الدرس المستفاد
الـ `LSM_AUDIT_DATA_LOCKDOWN` و`ad.u.reason` بيوضحوا إن المشكلة مش من SELinux أو AppArmor policy، لكن من الـ Lockdown mechanism نفسه. في bring-up، اعرف الفرق بين أنواع الـ LSM denials عشان توجه التحقيق صح ومتضيعش وقت تعدّل SELinux policy مشكلتها مش فيه.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ LSM audit framework في kernel بشكل مباشر:

| المقال | الأهمية |
|--------|---------|
| [LSM hooks for audit](https://lwn.net/Articles/102224/) | المقال الأصلي اللي ناقش إضافة audit hooks للـ LSM — أساسي لفهم تاريخ `common_audit_data` |
| [Light-weight Auditing Framework](https://lwn.net/Articles/73623/) | شرح إطار الـ audit الخفيف اللي بيستخدمه الـ LSM لتسجيل الأحداث |
| [Light-weight Auditing Framework (variant)](https://lwn.net/Articles/75408/) | نسخة تانية من المقال بتناقش تفاصيل التكامل مع SELinux |
| [LSM stacking and the future](https://lwn.net/Articles/804906/) | بيشرح تحديات دعم أكتر من LSM في نفس الوقت وأثرها على الـ audit records |
| [LSM: Stacking for major security modules](https://lwn.net/Articles/697259/) | يناقش التعديلات المطلوبة في الـ audit subsystem لدعم `selinux_audit_data` + `smack_audit_data` + `apparmor_audit_data` معاً |
| [Progress in security module stacking](https://lwn.net/Articles/635771/) | مراحل التطور نحو دعم stacking وتأثيره على struct `common_audit_data` |
| [Audit: Records for multiple security contexts](https://lwn.net/Articles/1014947/) | أحدث مقال — بيناقش إضافة `AUDIT_MAC_TASK_CONTEXTS` لدعم أكتر من LSM context في record واحد |
| [MAC and Audit policy using eBPF (KRSI)](https://lwn.net/Articles/809645/) | بيشرح كيف BPF LSM programs بتستخدم نفس infrastructure الـ audit |
| [Auditing io_uring](https://lwn.net/Articles/858023/) | مثال عملي على إضافة LSM hooks وبياناتها لـ `common_audit_data` لـ io_uring |
| [selinux: Implement LSM notification system](https://lwn.net/Articles/720991/) | بيشرح `LSM_AUDIT_DATA_NOTIFICATION` اللي اتضاف لـ `common_audit_data` |
| [Adding system calls for Linux security modules](https://lwn.net/Articles/919059/) | syscalls الجديدة للتعامل مع الـ LSM وعلاقتها بالـ audit |
| [A change in direction for security-module stacking?](https://lwn.net/Articles/970070/) | أحدث مناقشة (2024) عن مستقبل الـ LSM stacking والـ audit |
| [Integrity Policy Enforcement LSM (IPE)](https://lwn.net/Articles/969749/) | مثال على LSM جديد بيضيف بياناته الخاصة لإطار الـ audit |

---

### التوثيق الرسمي للـ Kernel

**الـ `Documentation/` paths الأساسية:**

```
Documentation/security/lsm.rst          ← شرح LSM framework والـ hooks
Documentation/security/lsm-development.rst  ← دليل كتابة LSM جديد
Documentation/admin-guide/LSM/          ← توثيق كل LSM مدعوم
Documentation/admin-guide/LSM/index.rst ← فهرس كل الـ LSMs
Documentation/admin-guide/LSM/apparmor.rst
Documentation/admin-guide/LSM/Smack.rst
Documentation/admin-guide/LSM/tomoyo.rst
Documentation/bpf/prog_lsm.rst          ← BPF LSM programs
```

**الروابط الإلكترونية للتوثيق:**

- [Linux Security Module Usage](https://docs.kernel.org/admin-guide/LSM/index.html) — نقطة البداية لفهم كيف بتتعامل مع الـ LSMs
- [Linux Security Module Development](https://docs.kernel.org/security/lsm-development.html) — كيف تكتب LSM وتستخدم `common_lsm_audit()`
- [Linux Security Modules: General Security Hooks](https://docs.kernel.org/security/lsm.html) — كل الـ hooks الموجودة وعلاقتها بالـ audit
- [LSM BPF Programs](https://docs.kernel.org/bpf/prog_lsm.html) — استخدام eBPF مع الـ LSM audit infrastructure
- [Landlock LSM](https://docs.kernel.org/admin-guide/LSM/landlock.html) — مثال على LSM بيستخدم audit framework الحديث

---

### الـ Source Files الأساسية في الـ Kernel

الملفات دي هي المرجع الأول قبل أي حاجة تانية:

```
include/linux/lsm_audit.h     ← الـ header موضوع توثيقنا
security/lsm_audit.c          ← implementation لـ common_lsm_audit() و audit_log_lsm_data()
kernel/audit.c                ← الـ audit subsystem الأساسي
include/linux/audit.h         ← audit buffer API اللي بيستخدمه lsm_audit
security/selinux/avc.c        ← أقدم مستخدم لـ common_lsm_audit
security/smack/smack_access.c ← مثال على smack_audit_data
security/apparmor/audit.c     ← مثال على apparmor_audit_data
```

**الـ GitHub mirror للـ audit kernel:**
- [linux-audit/audit-kernel](https://github.com/linux-audit/audit-kernel) — mirror رسمي لـ audit subsystem في الـ kernel
- [linux-audit/audit-documentation Wiki](https://github.com/linux-audit/audit-documentation/wiki) — توثيق إضافي للـ audit framework

---

### Commits المهمة

**الـ commits دي غيّرت أو أنشأت الـ lsm_audit infrastructure:**

| الحدث | الوصف |
|-------|-------|
| إضافة `lsm_audit.h` بقلم Etienne Basset | أنشأ `common_audit_data` و `common_lsm_audit()` كـ unified API لكل الـ LSMs بدل ما كل واحد يعمل audit code خاص بيه |
| إضافة `LSM_AUDIT_DATA_FILE` | [Patchwork](https://patchwork.kernel.org/patch/9323851/) — أضاف type جديد للـ union في `common_audit_data` لدعم file objects |
| إضافة `audit_log_lsm_data()` | [RFC v2](https://www.mail-archive.com/audit@vger.kernel.org/msg01054.html) / [v6](https://www.mail-archive.com/audit@vger.kernel.org/msg01383.html) / [v7](https://www.mail-archive.com/audit@vger.kernel.org/msg01457.html) — فصل الـ dump logic في helper منفصل |
| دعم RDMA/InfiniBand | إضافة `lsm_ibpkey_audit` و `lsm_ibendport_audit` للـ header |

---

### نقاشات Mailing List

الـ patches والنقاشات دي على mailing list بتوضح كيف الكود اتطور:

- **[PATCH v6 01/26] lsm: Add audit_log_lsm_data() helper** — [mail-archive.com](https://www.mail-archive.com/audit@vger.kernel.org/msg01383.html)
  فصل `dump_common_audit_data()` في helper قابل للاستدعاء المستقل

- **[RFC PATCH v2 02/14] lsm: Add audit_log_lsm_data() helper** — [mail-archive.com](https://www.mail-archive.com/audit@vger.kernel.org/msg01054.html)
  النسخة الأولى من الـ refactoring

- **[PATCH v7 01/28] lsm: Add audit_log_lsm_data() helper** — [mail-archive.com](https://www.mail-archive.com/audit@vger.kernel.org/msg01457.html)
  النسخة النهائية اللي اتدمجت — constified الـ argument

- **lsm,audit,selinux: Introduce LSM_AUDIT_DATA_FILE** — [Patchwork](https://patchwork.kernel.org/patch/9323851/)
  إضافة نوع جديد للـ audit data union

- **Audit: Add record for multiple task security contexts** — [Patchwork](https://patchwork.kernel.org/project/selinux/patch/20220415211801.12667-27-casey@schaufler-ca.com/)
  دعم LSM stacking في الـ audit records

---

### كتب مقترحة

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل:** مش بيغطي الـ LSM audit بشكل مباشر
- **الأهمية:** بيشرح `struct file`, `struct inode`, `struct dentry` اللي بتظهر في `common_audit_data.u`
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17:** Memory Management — بيشرح `struct path` و `struct dentry`
- **الفصل 14:** The Block I/O Layer — context مفيد لفهم الـ inode/file types في الـ audit data
- الأساسي لفهم الـ data structures اللي بتظهر في `union u` في `common_audit_data`

#### Understanding the Linux Kernel — Bovet & Cesati
- **الفصل 1 و 20:** بيغطي security architecture ودور الـ capabilities
- مفيد لفهم `LSM_AUDIT_DATA_CAP` والـ capability checking

#### The Linux Programming Interface — Michael Kerrisk
- **الفصل 38:** بيشرح Linux security model
- مفيد لفهم IPC IDs (`LSM_AUDIT_DATA_IPC`)، network sockets، وkeys

---

### مصادر إضافية

#### Kernel Newbies
- [KernelGlossary](https://kernelnewbies.org/KernelGlossary) — تعريفات LSM وباقي المصطلحات
- [Linux_2_6_18](https://kernelnewbies.org/Linux_2_6_18) — لما اتضافت LSM hooks للـ task scheduling
- [Linux_2_6_15](https://kernelnewbies.org/Linux_2_6_15) — LSM hooks لـ key management (بيتعلق بـ `LSM_AUDIT_DATA_KEY`)
- [FirstKernelPatch](https://kernelnewbies.org/FirstKernelPatch) — لو عايز تساهم في الـ LSM audit code

#### مراجع تانية
- [An Introduction Into Linux Security Modules](https://linux-audit.com/introduction-into-linux-security-modules/) — شرح عملي للـ LSM framework
- [Linux Audit Framework — ArchWiki](https://wiki.archlinux.org/title/Audit_framework) — كيف تقرأ وتفسر audit records اللي بيطلعها `common_lsm_audit()`
- [What You Need to Know About Linux Audit Framework](https://goteleport.com/blog/linux-audit/) — شرح للـ auditd وكيف بيتعامل مع الـ LSM audit messages
- [Linux Security Modules — Wikipedia](https://en.wikipedia.org/wiki/Linux_Security_Modules) — نظرة تاريخية على تطور الـ LSM framework

---

### Search Terms للبحث عن معلومات أكتر

للبحث في الـ kernel mailing lists والمصادر المتخصصة:

```
# للبحث في git log
git log --oneline -- security/lsm_audit.c
git log --oneline -- include/linux/lsm_audit.h
git log --grep="common_audit_data"
git log --grep="LSM_AUDIT_DATA"

# Mailing list searches
lore.kernel.org/linux-security-module/?q=common_audit_data
lore.kernel.org/audit/?q=lsm_audit

# LWN.net search terms
"common_lsm_audit" site:lwn.net
"LSM audit data" site:lwn.net
"lsm_audit" site:lwn.net
"selinux avc audit" site:lwn.net

# General search terms
linux kernel "struct common_audit_data" explanation
linux LSM audit hook selinux smack apparmor
linux kernel audit buffer lsm security context
"audit_log_lsm_data" kernel patch
linux kernel lsm_network_audit ipv6 ipv4
```
## Phase 8: Writing simple module

### الفكرة

**`common_lsm_audit()`** هي الدالة الي بتتصل بيها كل الـ LSM modules عشان تسجّل أي access decision في الـ audit log. هنعمل **kprobe** عليها عشان نراقب كل لحظة بيحاول فيها الـ LSM يسجّل event — ونطبع نوع البيانات الجاية معاها (`type` field في `common_audit_data`) مع اسم الـ process الي استدعى الكود ده.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * lsm_audit_probe.c
 * Hooks common_lsm_audit() via kprobe to log every LSM audit event type.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit          */
#include <linux/kprobes.h>      /* kprobe struct + register/unregister APIs  */
#include <linux/printk.h>       /* pr_info()                                 */
#include <linux/lsm_audit.h>    /* common_audit_data + LSM_AUDIT_DATA_* defs */
#include <linux/sched.h>        /* current, task_comm_len                    */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("LSM Watcher <example@kernel.org>");
MODULE_DESCRIPTION("kprobe on common_lsm_audit to trace LSM audit events");

/* ------------------------------------------------------------------ */
/* Helper: map the numeric type to a human-readable string            */
/* ------------------------------------------------------------------ */
static const char *audit_type_name(char type)
{
    switch (type) {
    case LSM_AUDIT_DATA_PATH:         return "PATH";
    case LSM_AUDIT_DATA_NET:          return "NET";
    case LSM_AUDIT_DATA_CAP:          return "CAP";
    case LSM_AUDIT_DATA_IPC:          return "IPC";
    case LSM_AUDIT_DATA_TASK:         return "TASK";
    case LSM_AUDIT_DATA_KEY:          return "KEY";
    case LSM_AUDIT_DATA_NONE:         return "NONE";
    case LSM_AUDIT_DATA_KMOD:         return "KMOD";
    case LSM_AUDIT_DATA_INODE:        return "INODE";
    case LSM_AUDIT_DATA_DENTRY:       return "DENTRY";
    case LSM_AUDIT_DATA_IOCTL_OP:     return "IOCTL_OP";
    case LSM_AUDIT_DATA_FILE:         return "FILE";
    case LSM_AUDIT_DATA_IBPKEY:       return "IBPKEY";
    case LSM_AUDIT_DATA_IBENDPORT:    return "IBENDPORT";
    case LSM_AUDIT_DATA_LOCKDOWN:     return "LOCKDOWN";
    case LSM_AUDIT_DATA_NOTIFICATION: return "NOTIFICATION";
    case LSM_AUDIT_DATA_ANONINODE:    return "ANONINODE";
    case LSM_AUDIT_DATA_NLMSGTYPE:    return "NLMSGTYPE";
    default:                          return "UNKNOWN";
    }
}

/* ------------------------------------------------------------------ */
/* pre-handler: runs just before common_lsm_audit() executes          */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * First argument of common_lsm_audit(a, pre_audit, post_audit)
     * is `struct common_audit_data *a` — on x86-64 it lives in RDI.
     */
    struct common_audit_data *ad =
            (struct common_audit_data *)regs->di;  /* x86-64: 1st arg = RDI */

    if (!ad)
        return 0;   /* skip if null — shouldn't happen, but be safe */

    pr_info("lsm_audit_probe: comm=%-16s pid=%d audit_type=%s(%d)\n",
            current->comm,          /* name of the running process          */
            task_pid_nr(current),   /* PID of the caller                    */
            audit_type_name(ad->type), /* readable type (PATH/NET/CAP/...)  */
            (int)ad->type);         /* raw numeric value for cross-check    */

    return 0;   /* 0 = continue normal execution, don't skip the probed fn */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "common_lsm_audit",  /* exact exported symbol name */
    .pre_handler = handler_pre,          /* our callback               */
};

/* ------------------------------------------------------------------ */
/* module_init: register the kprobe                                    */
/* ------------------------------------------------------------------ */
static int __init lsm_probe_init(void)
{
    int ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("lsm_audit_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("lsm_audit_probe: planted kprobe at %p\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit: unregister the kprobe                                  */
/* ------------------------------------------------------------------ */
static void __exit lsm_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("lsm_audit_probe: kprobe removed\n");
}

module_init(lsm_probe_init);
module_exit(lsm_probe_exit);
```

---

### Makefile لتجميع الـ module

```makefile
obj-m += lsm_audit_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل قسم

#### الـ Includes

| Header | السبب |
|--------|-------|
| `<linux/kprobes.h>` | بيجيب الـ `struct kprobe` وكل الـ API بتاعة register/unregister |
| `<linux/lsm_audit.h>` | بيعرّف `struct common_audit_data` والـ `LSM_AUDIT_DATA_*` constants الي بنستخدمها في الـ switch |
| `<linux/sched.h>` | بيجيب الـ `current` macro و `task_pid_nr()` عشان نعرف مين استدى الـ function |

---

#### الـ `audit_type_name()` helper

الـ `type` field في `common_audit_data` هو مجرد `char` بيتراوح من 1 لـ 18 — الـ helper ده بيحوّله لاسم مقروء زي `"PATH"` أو `"NET"` عشان الـ log يبقى مفيد من غير ما تفتح الـ header كل شوية.

---

#### الـ `handler_pre()`

**الـ `struct pt_regs *regs`** بيحتوي على قيم الـ registers وقت الاستدعاء. على معمارية **x86-64**، الـ argument الأول بيكون دايماً في `regs->di` (RDI register) — ده هو مؤشر `struct common_audit_data *a`. بنقرأ منه الـ `type` بس (safe read-only) ونطبعه مع اسم الـ process وPID الخاص بيه.

---

#### الـ `struct kprobe kp`

**`symbol_name`** بيخلي الـ kernel يحلّ العنوان وقت الـ `register_kprobe` — مش محتاج تحدد عنوان صلب. **`pre_handler`** بيشتغل مباشرةً قبل ما `common_lsm_audit` تنفّذ أي كود، يعني بنشوف كل الـ arguments سليمة.

---

#### الـ `module_exit`

`unregister_kprobe()` ضروري في الـ exit عشان لو مشيناش الـ kprobe هيفضل موجود في الـ kernel وأي استدعاء لـ `common_lsm_audit` بعد `rmmod` هيدي kernel panic أو use-after-free لأن الـ handler بقى في memory محذوفة.

---

### تشغيل وتتبع النتائج

```bash
# تحميل الـ module
sudo insmod lsm_audit_probe.ko

# مراقبة الـ kernel log
sudo dmesg -w | grep lsm_audit_probe

# مثال مثير للـ NET audit: استخدام curl أو connect
curl http://example.com

# تفريغ الـ module
sudo rmmod lsm_audit_probe
```

**مثال على الـ output المتوقع:**

```
lsm_audit_probe: planted kprobe at ffffffffc0a12340
lsm_audit_probe: comm=sshd             pid=1423 audit_type=NET(2)
lsm_audit_probe: comm=curl             pid=3817 audit_type=NET(2)
lsm_audit_probe: comm=chmod            pid=3821 audit_type=PATH(1)
lsm_audit_probe: comm=python3          pid=3902 audit_type=CAP(3)
lsm_audit_probe: kprobe removed
```

كل سطر بيمثّل قرار LSM واحد على وشك يتسجّل — PATH لما process بتلمس ملف، NET لما socket بتتصل، CAP لما process بتطلب privilege.
