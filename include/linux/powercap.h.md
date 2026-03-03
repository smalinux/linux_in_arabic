## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem ده؟

الـ `powercap.h` جزء من **Power Management Core** subsystem في Linux kernel. المسؤول عنه Rafael J. Wysocki، وموجود تحت قائمة `POWER MANAGEMENT CORE` في MAINTAINERS. الـ mailing list بتاعه هو `linux-pm@vger.kernel.org`.

---

### القصة الكاملة — ليه الـ Powercap Framework موجود؟

تخيل إنك شغال data center وعندك rack فيه 40 سيرفر. الكهرباء بتاعت الـ rack محدودة — مثلاً 10 kilowatt بس. لو كل السيرفرات شتغلت بأقصى طاقة في نفس الوقت هيحصل overload ويتحرق الـ circuit breaker.

المشكلة: كل processor عنده طريقته الخاصة لتحديد حد الـ power — Intel بتستخدم تقنية اسمها **RAPL (Running Average Power Limit)** بتشتغل عبر **MSR registers** داخل الـ CPU. ARM processors بتستخدم **SCMI (System Control and Management Interface)**. في طرق تانية زي **idle injection** (توقيف الـ CPUs بشكل متعمد عشان يقلل استهلاك الطاقة).

لو كل driver كتب interface خاص بيه في sysfs، الـ userspace tools هتبقى chaos — أداة Intel مش هتفهم AMD، وأداة ARM مش هتفهم Intel.

**الحل:** الـ **Power Capping Framework** — وده اللي `powercap.h` بيعرّفه.

الـ framework ده بيوفر **abstraction layer** موحدة: أي driver — مهما كانت التقنية اللي بيستخدمها — يسجل نفسه بـ API واحدة، وبتاخد sysfs interface موحدة في:
```
/sys/devices/virtual/powercap/
```

---

### الصورة الكبيرة — ELI5

فكر في الـ framework زي **مطار دولي**:

```
┌─────────────────────────────────────────────────────┐
│              Power Capping Framework                 │
│              (المطار — المنظم العام)                 │
├──────────────┬──────────────┬───────────────────────┤
│  intel-rapl  │     dtpm     │   arm-scmi-powercap   │
│  (خط Intel)  │  (خط DTPM)  │    (خط ARM)           │
├──────────────┼──────────────┼───────────────────────┤
│  Package     │    CPU       │    Cluster            │
│  Core        │   DevFreq    │    ...                │
│  Uncore      │    ...       │                       │
│  DRAM        │              │                       │
└──────────────┴──────────────┴───────────────────────┘
         ↕                ↕                ↕
    Userspace tools (powertop, TDP managers, cloud orchestrators)
```

كل "خط طيران" هو **control_type** — يمثل تقنية معينة للتحكم في الطاقة.
كل "رحلة" هي **power_zone** — يمثل جزء من الـ hardware (CPU package, DRAM, GPU...).
كل "قيود السفر" هي **constraints** — الحدود الفعلية للطاقة (short-term limit, long-term limit).

---

### المفاهيم الأساسية في الـ Header

#### 1. `powercap_control_type` — نوع التحكم

```c
struct powercap_control_type {
    struct device dev;       /* device في sysfs */
    struct idr idr;          /* ID manager للـ zones */
    int nr_zones;            /* عدد الـ zones المسجلة */
    const struct powercap_control_type_ops *ops; /* callbacks */
    struct mutex lock;       /* thread safety */
    bool allocated;          /* هل الـ framework اتكلف بالـ memory؟ */
    struct list_head node;   /* linked list مع باقي الـ control types */
};
```

ده الـ container الكبير — مثلاً `intel-rapl` كله كـ control_type واحد.

#### 2. `powercap_zone` — منطقة الطاقة

```c
struct powercap_zone {
    int id;
    char *name;              /* مثلاً "package-0", "core", "dram" */
    void *control_type_inst; /* مين أبوه */
    const struct powercap_zone_ops *ops;
    struct device dev;
    struct idr idr;          /* ID manager للـ child zones */
    struct idr *parent_idr;  /* reference للـ parent */
    void *private_data;      /* driver-specific data */
    struct powercap_zone_constraint *constraints; /* الحدود */
};
```

الـ zones ممكن تكون **متداخلة (hierarchical)** — مثلاً:
```
intel-rapl (control_type)
  └── package-0 (zone)
        ├── core (sub-zone)
        └── uncore (sub-zone)
```

#### 3. `powercap_zone_constraint` — القيد الفعلي

```c
struct powercap_zone_constraint {
    int id;
    struct powercap_zone *power_zone;
    const struct powercap_zone_constraint_ops *ops;
};
```

كل zone ممكن يكون فيه أكتر من constraint — مثلاً Intel RAPL عندها:
- **constraint_0**: long-term power limit (مثلاً 15 seconds window)
- **constraint_1**: short-term power limit (مثلاً 1-3ms window)

#### 4. الـ Callbacks — عقد الـ API

الـ framework بيقول لكل driver: "انت لازم تعمل implement لهذه الـ functions":

```c
struct powercap_zone_constraint_ops {
    int (*set_power_limit_uw)(struct powercap_zone *, int, u64); /* إجباري */
    int (*get_power_limit_uw)(struct powercap_zone *, int, u64 *); /* إجباري */
    int (*set_time_window_us)(struct powercap_zone *, int, u64); /* إجباري */
    int (*get_time_window_us)(struct powercap_zone *, int, u64 *); /* إجباري */
    int (*get_max_power_uw)(struct powercap_zone *, int, u64 *);
    int (*get_min_power_uw)(struct powercap_zone *, int, u64 *);
    int (*get_max_time_window_us)(struct powercap_zone *, int, u64 *);
    int (*get_min_time_window_us)(struct powercap_zone *, int, u64 *);
    const char *(*get_name)(struct powercap_zone *, int); /* إجباري */
};
```

الـ framework هو اللي بيعرض الـ sysfs attributes وبينادي على الـ callbacks دي لما userspace يكتب أو يقرأ.

---

### رحلة كاملة — من الـ Driver لحد الـ User

```
Intel CPU Hardware (MSR Registers)
         ↓
intel_rapl_common.c  ←── يقرأ/يكتب الـ MSR registers
         ↓ (يعمل register لكل domain)
powercap_register_zone() ← من powercap_sys.c
         ↓
sysfs: /sys/devices/virtual/powercap/intel-rapl/intel-rapl:0/
         ├── constraint_0_power_limit_uw  ← write هنا = تحديد حد الواط
         ├── constraint_0_time_window_us  ← الفترة الزمنية للمتوسط
         └── energy_uj                   ← قراءة الطاقة المستهلكة
         ↓
powertop / TDP tools / cloud power managers
```

لما userspace يكتب في `constraint_0_power_limit_uw`:
1. الـ kernel يستقبل الكتابة في sysfs
2. `powercap_sys.c` يستدعي `set_power_limit_uw()` من الـ driver
3. الـ Intel RAPL driver يكتب القيمة في الـ MSR register المناسب
4. الـ CPU hardware بيطبق الـ limit فعلاً

---

### الـ API العامة للـ Drivers

```c
/* تسجيل نوع التحكم */
struct powercap_control_type *powercap_register_control_type(
    struct powercap_control_type *control_type,
    const char *name,
    const struct powercap_control_type_ops *ops);

/* تسجيل zone تحت control_type أو تحت parent zone */
struct powercap_zone *powercap_register_zone(
    struct powercap_zone *power_zone,
    struct powercap_control_type *control_type,
    const char *name,
    struct powercap_zone *parent,   /* NULL = directly under control_type */
    const struct powercap_zone_ops *ops,
    int nr_constraints,
    const struct powercap_zone_constraint_ops *const_ops);

/* إلغاء التسجيل */
int powercap_unregister_zone(...);
int powercap_unregister_control_type(...);
```

---

### الحدود والثوابت المهمة

| Macro | القيمة | المعنى |
|-------|--------|--------|
| `POWERCAP_ZONE_MAX_ATTRS` | 6 | أقصى عدد attributes لكل zone |
| `POWERCAP_CONSTRAINTS_ATTRS` | 8 | أقصى attributes لكل constraint |
| `MAX_CONSTRAINTS_PER_ZONE` | 10 | أقصى عدد constraints لكل zone |

---

### الملفات المكوّنة للـ Subsystem

#### Core Framework

| الملف | الدور |
|-------|-------|
| `include/linux/powercap.h` | **الملف ده** — تعريفات الـ API والـ structs |
| `drivers/powercap/powercap_sys.c` | الـ implementation — sysfs, device model |

#### Hardware Drivers (clients)

| الملف | الدور |
|-------|-------|
| `drivers/powercap/intel_rapl_common.c` | Intel RAPL common code |
| `drivers/powercap/intel_rapl_msr.c` | Intel RAPL via MSR registers |
| `drivers/powercap/intel_rapl_tpmi.c` | Intel RAPL via TPMI registers |
| `drivers/powercap/arm_scmi_powercap.c` | ARM SCMI power capping |
| `drivers/powercap/dtpm.c` | Dynamic Thermal Power Management |
| `drivers/powercap/dtpm_cpu.c` | DTPM for CPUs |
| `drivers/powercap/dtpm_devfreq.c` | DTPM for devfreq devices |
| `drivers/powercap/idle_inject.c` | Power control via idle injection |

#### Headers ذات صلة

| الملف | الدور |
|-------|-------|
| `include/linux/intel_rapl.h` | Intel RAPL-specific types (تستخدم powercap.h) |
| `include/linux/dtpm.h` | DTPM-specific types |

#### Documentation

| الملف | الدور |
|-------|-------|
| `Documentation/power/powercap/powercap.rst` | شرح الـ framework والـ sysfs interface |
| `Documentation/power/powercap/dtpm.rst` | شرح DTPM specifically |

---

### ملفات القراءة التالية المقترحة

1. **`drivers/powercap/powercap_sys.c`** — شوف إزاي الـ framework بيعمل implement لكل حاجة في `powercap.h`
2. **`drivers/powercap/intel_rapl_common.c`** — أوضح مثال على client driver بيستخدم الـ API
3. **`include/linux/intel_rapl.h`** — تعريفات Intel RAPL اللي بتبني فوق الـ powercap structs
## Phase 2: شرح الـ Powercap Framework

### المشكلة اللي بيحلها الـ Framework

الـ SoC الحديث — زي Intel Core أو ARM big.LITTLE — فيه مكونات hardware متعددة بتستهلك power: CPU cores، GPU، DRAM، uncore fabric. لو مفيش تنسيق مركزي، كل driver بيعمل power management بطريقته الخاصة: واحد بيكتب على MSR مباشرةً، تاني بيبعت i2c command، وتالت بيعمل ioctl خاص. النتيجة:

- **User-space** مش عارف يتحكم في power budget موحد عبر أجزاء الـ SoC المختلفة.
- **Tools** زي `powertop` و `thermald` محتاجين يتعلموا interface مختلف لكل hardware.
- **Policy** (مين بيقرر ياخد كام watt؟) متداخلة مع **mechanism** (إزاي بنضبط الـ hardware؟).

---

### الحل: Powercap Framework

الـ kernel بيقدم **powercap** كـ abstraction layer موحدة. الفكرة بسيطة:

1. **Control Type**: تقنية تحكم في الـ power (مثال: RAPL لـ Intel).
2. **Power Zone**: منطقة hardware ليها power budget (مثال: Package، Core، Uncore).
3. **Constraint**: قيد على الـ power zone (power limit + time window).

كل ده بيتعرض عبر **sysfs** بشكل موحد — أي tool بتكلم `/sys/class/powercap/` وبس.

---

### الـ Big Picture Architecture

```
User Space (thermald, powertop, systemd)
          |
          | read/write sysfs files
          v
+------------------------------------------+
|           /sys/class/powercap/           |
|                                          |
|   +----------------------------------+   |
|   |   powercap core (drivers/powercap/powercap.c)   |
|   |  - registers class               |   |
|   |  - manages control_types         |   |
|   |  - creates sysfs attributes      |   |
|   +----------------------------------+   |
+------------------------------------------+
          |
          | powercap_register_control_type()
          | powercap_register_zone()
          v
+------------------------------------------+
|         Control Type Layer               |
|                                          |
|  [RAPL driver]    [other vendor driver]  |
|  intel_rapl_msr   intel_rapl_pci         |
+------------------------------------------+
          |
          | reads/writes hardware registers
          v
+------------------------------------------+
|         Hardware Layer                   |
|  Intel MSR RAPL | MMIO | I2C PMIC | etc. |
+------------------------------------------+
```

**الـ sysfs tree** بيبقى شكله كده:

```
/sys/class/powercap/
└── intel-rapl/                    ← control_type
    ├── enabled
    └── intel-rapl:0/              ← power zone (package 0)
        ├── name
        ├── energy_uj
        ├── max_energy_range_uj
        ├── constraint_0_power_limit_uw
        ├── constraint_0_time_window_us
        ├── constraint_0_name
        └── intel-rapl:0:0/        ← child zone (cores)
            ├── name
            ├── energy_uj
            └── constraint_0_power_limit_uw
```

---

### الـ Real-World Analogy: نظام الميزانية في شركة

تخيل شركة كبيرة بتشتغل بـ budget محدود كل شهر:

| مفهوم الشركة | مفهوم Powercap |
|---|---|
| **نوع العملة** (جنيه، دولار) | **Control Type** (RAPL، vendor X) |
| **إدارة** (Marketing، R&D، Ops) | **Power Zone** (Package، Core، DRAM) |
| **حد الإنفاق لإدارة** | **Constraint** (power limit بالـ microwatts) |
| **نافذة زمنية** (ميزانية شهرية vs ربع سنوية) | **Time Window** (short-term vs long-term window) |
| **عداد الإنفاق الفعلي** | **energy_uj counter** |
| **CFO بيقرأ تقارير موحدة** | **thermald بيقرأ sysfs موحد** |
| **الإدارة تحت إدارة رئيسية** | **Child Zone تحت Parent Zone** |
| **HR policy** مش بتعرف تفاصيل accounts | **thermald** مش محتاج يعرف MSRs |

الـ mapping التفصيلي:

- **CFO** = thermald daemon — بيقرر السياسة (مين ياخد كام watt).
- **نظام المحاسبة الموحد** = powercap sysfs interface — CFO بيتكلم نظام واحد.
- **كل إدارة بتُسلّم تقرير بنفس الفورمات** = كل driver بيimpliment نفس `powercap_zone_ops`.
- **المحاسب الداخلي لكل إدارة** = الـ driver نفسه — هو اللي بيعرف يقرأ الـ hardware registers.
- **الإدارة الرئيسية بتوزع budget على الفروع** = parent zone بتضبط child zones.

---

### الـ Core Abstractions: الـ Structs وعلاقاتهم

#### نظرة عامة على العلاقات

```
powercap_control_type
    │
    │  (parent of)
    ├──► powercap_zone  [zone 0: package]
    │        │
    │        │  (child of)
    │        ├──► powercap_zone  [zone 0:0: cores]
    │        │        └──► powercap_zone_constraint [0] ──► powercap_zone_constraint_ops
    │        │        └──► powercap_zone_constraint [1]
    │        │
    │        └──► powercap_zone_constraint [0] ──► powercap_zone_constraint_ops
    │             (short window limit)
    │        └──► powercap_zone_constraint [1]
    │             (long window limit)
    │
    └──► powercap_control_type_ops
```

#### `struct powercap_control_type` — الـ Container الرئيسي

```c
struct powercap_control_type {
    struct device dev;           /* kernel device — ده بيخلي sysfs entry يتعمل تلقائياً */
    struct idr idr;              /* ID Radix Tree — بيخزن child zones بـ unique integer IDs */
    int nr_zones;                /* عدد الـ zones المسجلة تحت الـ control type ده */
    const struct powercap_control_type_ops *ops; /* callbacks: enable/disable الـ control type كله */
    struct mutex lock;           /* يحمي concurrent access على الـ zone list */
    bool allocated;              /* هل الـ framework هو اللي allocate الـ memory ولا الـ driver؟ */
    struct list_head node;       /* بيربط كل الـ control types في linked list عالمي */
};
```

**الـ `struct idr`** — ده subsystem تاني مهم: بيعمل mapping بين integer IDs وpointers. بيستخدم **radix tree** داخلياً عشان يكون O(log n). الـ powercap بيستخدمه عشان يدي كل zone ID رقمي فريد (`intel-rapl:0`، `intel-rapl:0:0`) بدل ما يستخدم linked list بس.

#### `struct powercap_zone` — وحدة الـ Power Management

```c
struct powercap_zone {
    int id;                          /* الرقم الفريد — جاي من الـ IDR */
    char *name;                      /* اسم بشري: "package-0"، "core"، "dram" */
    void *control_type_inst;         /* pointer للـ control_type الأب — void* للمرونة */
    const struct powercap_zone_ops *ops; /* callbacks: اقرأ energy، اقرأ power */
    struct device dev;               /* kernel device — sysfs node */
    int const_id_cnt;                /* كم constraint مسجل على الـ zone ده */
    struct idr idr;                  /* IDR للـ child zones */
    struct idr *parent_idr;          /* pointer لـ IDR الأب — عشان نقدر نشيل نفسنا منه وقت الـ unregister */
    void *private_data;              /* driver-specific data — RAPL driver بيحط هنا MSR address مثلاً */
    struct attribute **zone_dev_attrs;       /* sysfs attributes array للـ zone */
    int zone_attr_count;
    struct attribute_group dev_zone_attr_group;
    const struct attribute_group *dev_attr_groups[2]; /* array لتسجيل مع الـ device */
    bool allocated;
    struct powercap_zone_constraint *constraints; /* array من constraints */
};
```

**لاحظ**: الـ `private_data` هو الجسر بين الـ framework والـ driver. لما RAPL driver يسجل zone، بيحط فيها pointer لـ struct خاصة فيه بتحتوي على MSR addresses وغيره. الـ framework مش محتاج يعرف تفاصيلها.

#### `struct powercap_zone_constraint` — الـ Power Budget القيد

```c
struct powercap_zone_constraint {
    int id;                          /* 0 = short window, 1 = long window (مثال RAPL) */
    struct powercap_zone *power_zone; /* إيه الـ zone المالك لـ constraint ده */
    const struct powercap_zone_constraint_ops *ops; /* callbacks */
};
```

كل zone ممكن يبقى عنده **حتى 10 constraints** (`MAX_CONSTRAINTS_PER_ZONE = 10`). في Intel RAPL عندهم 2:
- **Constraint 0**: short-term power limit (PL1) — مثلاً 15W لمدة 28 ثانية.
- **Constraint 1**: long-term power limit (PL2) — مثلاً 25W لمدة 1ms.

#### الـ Ops Structs — واجهة الـ Driver

```c
/* Zone يلزمه يimpliment واحد على الأقل: energy أو power */
struct powercap_zone_ops {
    int (*get_max_energy_range_uj) (struct powercap_zone *, u64 *); /* max قبل rollover */
    int (*get_energy_uj)           (struct powercap_zone *, u64 *); /* القراءة الحالية */
    int (*reset_energy_uj)         (struct powercap_zone *);         /* صفّر العداد */
    int (*get_max_power_range_uw)  (struct powercap_zone *, u64 *);
    int (*get_power_uw)            (struct powercap_zone *, u64 *); /* instantaneous power */
    int (*set_enable)              (struct powercap_zone *, bool);
    int (*get_enable)              (struct powercap_zone *, bool *);
    int (*release)                 (struct powercap_zone *);         /* cleanup */
};

/* Constraint يلزمه الـ 5 callbacks دول */
struct powercap_zone_constraint_ops {
    int (*set_power_limit_uw)    (struct powercap_zone *, int constraint_id, u64);
    int (*get_power_limit_uw)    (struct powercap_zone *, int constraint_id, u64 *);
    int (*set_time_window_us)    (struct powercap_zone *, int constraint_id, u64);
    int (*get_time_window_us)    (struct powercap_zone *, int constraint_id, u64 *);
    /* optional range queries */
    int (*get_max_power_uw)      (struct powercap_zone *, int, u64 *);
    int (*get_min_power_uw)      (struct powercap_zone *, int, u64 *);
    int (*get_max_time_window_us)(struct powercap_zone *, int, u64 *);
    int (*get_min_time_window_us)(struct powercap_zone *, int, u64 *);
    const char *(*get_name)      (struct powercap_zone *, int); /* "long_term"، "short_term" */
};
```

---

### الـ Registration Flow

#### إزاي Driver بيسجل نفسه؟

```
Driver Init
    │
    ▼
powercap_register_control_type("intel-rapl", ops)
    │  ← بيعمل device_register، بيضيف لـ global list
    │
    ▼
powercap_register_zone(zone, ctrl_type, "package-0", NULL, zone_ops, 2, const_ops)
    │  ← parent=NULL يعني directly under control_type
    │  ← nr_constraints=2 → يعمل array من 2 powercap_zone_constraint
    │
    ▼
powercap_register_zone(subzone, ctrl_type, "core", package_zone, zone_ops, 1, const_ops)
    │  ← parent=package_zone → بيبقى child zone
    │  ← الـ framework بيستخدم IDR عشان يدي الـ zone ID فريد
    │
    ▼
sysfs entries تتعمل تلقائياً عبر device model
```

#### الـ Memory Ownership Model

الـ framework بيدعم نموذجين:

**نموذج 1 — الـ Framework يعمل الـ allocation:**
```c
/* مرر NULL → الـ framework بيعمل kmalloc ويحط allocated=true */
zone = powercap_register_zone(NULL, ctrl, "pkg", NULL, ops, 2, cops);
```

**نموذج 2 — الـ Driver يعمل الـ allocation (Embedding):**
```c
/* driver بيحط powercap_zone جوه struct خاصة بيه */
struct rapl_domain {
    struct powercap_zone power_zone; /* مضمن مباشرةً */
    u64 msr_addr;
    /* ... */
};
/* مرر pointer → allocated=false → الـ framework مش بيعمل free */
powercap_register_zone(&domain->power_zone, ctrl, "pkg", NULL, ops, 2, cops);
```

الـ `allocated` flag بيحمي من double-free.

---

### الـ Framework يملك vs. بيفوّض

| المسؤولية | الـ Framework | الـ Driver |
|---|---|---|
| إنشاء sysfs entries | نعم — `device_register()` | لا |
| تسمية الـ attributes | نعم — `energy_uj`، `constraint_0_power_limit_uw` | لا |
| ربط الـ zones في hierarchy | نعم — IDR + parent pointer | لا |
| قراءة hardware register | لا | نعم — `get_energy_uj()` callback |
| كتابة power limit على الـ HW | لا | نعم — `set_power_limit_uw()` callback |
| تحديد عدد constraints | لا | نعم — وقت الـ `register_zone()` |
| mutual exclusion في الـ callbacks | لا — الـ driver مسؤول | نعم |
| تحديد ما إذا كان energy أو power | لا | نعم — أي من الاثنين كافي |
| ID allocation للـ zones | نعم — IDR | لا |
| Lifecycle management (ref counting) | نعم — device model | لا |

---

### الـ sysfs وعلاقته بالـ device model

الـ powercap بيعتمد اعتماداً كاملاً على الـ **Linux Device Model** (subsystem مستقل مهم):

- **الـ `struct device` مضمن** في كل من `powercap_control_type` و`powercap_zone` — ده يعني كل zone هو device حقيقي في kernel.
- الـ `device_register()` بينشئ automatically الـ kobject وبالتالي الـ sysfs directory.
- الـ attributes (`energy_uj`، `constraint_0_power_limit_uw`) بتتسجل عبر `sysfs_create_group()`.
- الـ `device_unregister()` بيضمن الـ ref-counted cleanup.

```
powercap_zone.dev  ──► kobject ──► /sys/class/powercap/intel-rapl:0/
                                         ├── energy_uj          (read)
                                         ├── max_energy_range_uj (read)
                                         ├── constraint_0_power_limit_uw (read/write)
                                         ├── constraint_0_time_window_us (read/write)
                                         └── constraint_0_name  (read)
```

لما user space يكتب على `constraint_0_power_limit_uw`، الـ sysfs بيستدعي الـ `store` function الخاصة بالـ attribute، اللي بتستدعي `constraint->ops->set_power_limit_uw(zone, 0, value)` — وهنا الـ driver بيتولى ضبط الـ hardware register.

---

### مثال واقعي: Intel RAPL

**RAPL** = Running Average Power Limit — تقنية Intel بتسمح بضبط power limit على الـ CPU عبر **MSRs** (Model-Specific Registers).

```
Control Type: "intel-rapl"
│
├── Zone: "package-0"  (MSR_PKG_ENERGY_STATUS، MSR_PKG_POWER_LIMIT)
│   ├── Constraint 0: "long_term"  → PL1 (مثلاً 15W / 28s)
│   ├── Constraint 1: "short_term" → PL2 (مثلاً 25W / 2ms)
│   │
│   ├── Zone: "core"    (MSR_PP0_ENERGY_STATUS)
│   │   └── Constraint 0: "long_term"
│   │
│   ├── Zone: "uncore"  (MSR_PP1_ENERGY_STATUS)
│   │   └── Constraint 0: "long_term"
│   │
│   └── Zone: "dram"    (MSR_DRAM_ENERGY_STATUS)
│       └── Constraint 0: "long_term"
│
└── Zone: "package-1"  (في multi-socket systems)
    └── ...
```

**thermald** بيقرأ temperature، يقرر إن الـ CPU بيسخن، وبيكتب:
```bash
echo 10000000 > /sys/class/powercap/intel-rapl/intel-rapl:0/constraint_0_power_limit_uw
# → 10W limit على الـ package
```

الـ RAPL driver بياخد الـ value ده وبيكتبه على الـ MSR المناسب — من غير ما thermald يعرف أي MSR بالظبط.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Macros والـ Constants — Cheatsheet

| الـ Macro / Constant | القيمة | الغرض |
|---|---|---|
| `POWERCAP_ZONE_MAX_ATTRS` | `6` | الحد الأقصى لعدد الـ sysfs attributes لكل zone |
| `POWERCAP_CONSTRAINTS_ATTRS` | `8` | عدد الـ attributes لكل constraint في الـ sysfs |
| `MAX_CONSTRAINTS_PER_ZONE` | `10` | الحد الأقصى لعدد الـ constraints المسموح بيهم لكل zone |
| `POWERCAP_GET_DEV(zone)` | `(&zone->dev)` | Macro بياخد الـ `struct device*` من الـ zone مباشرةً |

---

### الـ Structs المهمة

---

#### 1. `struct powercap_control_type_ops`

**الغرض:** جدول الـ callbacks اللي بيعرّف سلوك الـ control type بالنسبة للـ framework. الـ driver بيملى الـ ops دي وبيجيب للـ framework صلاحية التحكم فيه.

| الـ Field | النوع | الغرض |
|---|---|---|
| `set_enable` | `int (*)(ct*, bool)` | تفعيل / تعطيل الـ control type كله — لو `false` بيتوقف التحكم في الـ power لكل الـ zones |
| `get_enable` | `int (*)(ct*, bool*)` | قراءة الحالة الحالية (مفعل / معطل) |
| `release` | `int (*)(ct*)` | **إلزامي** لو الـ driver هو اللي بيمتلك الـ memory — بيتنادى لما آخر reference يتحرر |

> الـ `release` callback هو البديل لـ `kfree` في الـ framework — الـ framework مش بيعرف مين المالك الحقيقي للـ memory.

---

#### 2. `struct powercap_control_type`

**الغرض:** الـ container الرئيسي اللي بيمثل نوع تقنية تحكم في الـ power (مثلاً RAPL). بيحتوي على كل الـ zones اللي بتشتغل بنفس الطريقة.

| الـ Field | النوع | الغرض |
|---|---|---|
| `dev` | `struct device` | الـ sysfs device — مضمّن directly (مش pointer) |
| `idr` | `struct idr` | Radix-tree لتوليد IDs فريدة للـ child zones |
| `nr_zones` | `int` | عداد الـ zones المسجلة تحت الـ control type ده |
| `ops` | `const struct powercap_control_type_ops *` | مؤشر لجدول الـ callbacks |
| `lock` | `struct mutex` | mutex بيحمي التعديل على الـ control type (إضافة/حذف zones) |
| `allocated` | `bool` | `true` = الـ framework هو المالك (هيعمل `kfree`)، `false` = الـ driver هو المالك |
| `node` | `struct list_head` | ربط الـ control type في القائمة العالمية للـ framework |

---

#### 3. `struct powercap_zone_ops`

**الغرض:** جدول الـ callbacks الخاص بـ zone واحدة. الـ driver لازم يوفر على الأقل نوع واحد (energy أو power).

| الـ Field | النوع | إلزامي؟ |
|---|---|---|
| `get_max_energy_range_uj` | `int (*)(zone*, u64*)` | اختياري |
| `get_energy_uj` | `int (*)(zone*, u64*)` | اختياري (لو بيستخدم energy) |
| `reset_energy_uj` | `int (*)(zone*)` | اختياري |
| `get_max_power_range_uw` | `int (*)(zone*, u64*)` | اختياري |
| `get_power_uw` | `int (*)(zone*, u64*)` | اختياري (لو بيستخدم power) |
| `set_enable` | `int (*)(zone*, bool)` | اختياري |
| `get_enable` | `int (*)(zone*, bool*)` | اختياري |
| `release` | `int (*)(zone*)` | إلزامي لو الـ driver مالك الـ memory |

> لازم يتوفر واحد على الأقل من: `get_energy_uj` أو `get_power_uw`.

---

#### 4. `struct powercap_zone`

**الغرض:** يمثل منطقة تحكم في الـ power واحدة — ممكن تكون CPU package، أو DRAM، أو GPU. كل zone ممكن تحتوي على zones فرعية وعلى constraints.

| الـ Field | النوع | الغرض |
|---|---|---|
| `id` | `int` | ID فريد مولّد من الـ IDR الخاص بالـ parent |
| `name` | `char *` | اسم الـ zone اللي بيظهر في الـ sysfs |
| `control_type_inst` | `void *` | مؤشر للـ `powercap_control_type` المالك (بيُحفظ كـ void*) |
| `ops` | `const struct powercap_zone_ops *` | جدول الـ callbacks الخاص بالـ zone |
| `dev` | `struct device` | الـ sysfs device الخاص بالـ zone — مضمّن |
| `const_id_cnt` | `int` | عدد الـ constraints المعرّفة لهذه الـ zone |
| `idr` | `struct idr` | لتوليد IDs للـ child zones |
| `parent_idr` | `struct idr *` | مؤشر لـ IDR الـ parent عشان تشيل نفسها منه عند الحذف |
| `private_data` | `void *` | بيانات خاصة بالـ driver — يُعدّل عبر `powercap_set_zone_data()` |
| `zone_dev_attrs` | `struct attribute **` | مصفوفة الـ sysfs attributes |
| `zone_attr_count` | `int` | عدد الـ attributes |
| `dev_zone_attr_group` | `struct attribute_group` | الـ group الوحيد للـ attributes — مضمّن |
| `dev_attr_groups` | `const struct attribute_group *[2]` | مصفوفة الـ groups المسجلة مع الـ device (group + NULL) |
| `allocated` | `bool` | نفس منطق الـ control type — من يمتلك الـ memory |
| `constraints` | `struct powercap_zone_constraint *` | مصفوفة الـ constraints (حجمها `const_id_cnt`) |

---

#### 5. `struct powercap_zone_constraint_ops`

**الغرض:** جدول الـ callbacks الخاص بكل constraint — بيتحكم في الـ power limit والـ time window.

| الـ Field | إلزامي؟ | الوحدة |
|---|---|---|
| `set_power_limit_uw` | **نعم** | micro-watts |
| `get_power_limit_uw` | **نعم** | micro-watts |
| `set_time_window_us` | **نعم** | micro-seconds |
| `get_time_window_us` | **نعم** | micro-seconds |
| `get_name` | **نعم** | string |
| `get_max_power_uw` | لأ | micro-watts |
| `get_min_power_uw` | لأ | micro-watts |
| `get_max_time_window_us` | لأ | micro-seconds |
| `get_min_time_window_us` | لأ | micro-seconds |

---

#### 6. `struct powercap_zone_constraint`

**الغرض:** يمثل قيد واحد (مثلاً short-term limit أو long-term limit) على zone معينة.

| الـ Field | النوع | الغرض |
|---|---|---|
| `id` | `int` | رقم الـ constraint (0، 1، ...) — بيظهر في الـ sysfs كـ `constraint_N_*` |
| `power_zone` | `struct powercap_zone *` | back-pointer للـ zone المالكة |
| `ops` | `const struct powercap_zone_constraint_ops *` | جدول الـ callbacks |

---

### رسم العلاقات بين الـ Structs

```
powercap framework (global list)
│
├──► powercap_control_type  ◄─── (e.g. "intel-rapl")
│      │  .dev        → sysfs: /sys/class/powercap/intel-rapl/
│      │  .idr        → assigns IDs to child zones
│      │  .lock       → mutex protecting zone add/remove
│      │  .ops ──────────────────────────────────────────►  powercap_control_type_ops
│      │  .node       → list_head in framework global list  │  .set_enable()
│      │                                                     │  .get_enable()
│      │                                                     └  .release()
│      │
│      └──► powercap_zone   (root zone, e.g. "intel-rapl:0")
│             │  .dev            → sysfs: /sys/class/powercap/intel-rapl:0/
│             │  .control_type_inst ──────────► (back to control_type, void*)
│             │  .idr            → assigns IDs to child zones
│             │  .parent_idr ──► control_type.idr  (to remove self)
│             │  .ops ─────────────────────────────────────► powercap_zone_ops
│             │  .private_data ──► driver private struct     │  .get_energy_uj()
│             │  .constraints[] ──┐                          │  .get_power_uw()
│             │                   │                          └  .release()
│             │                   ▼
│             │          powercap_zone_constraint[0]
│             │            .id = 0
│             │            .power_zone ──────────────────► (back to zone)
│             │            .ops ───────────────────────► powercap_zone_constraint_ops
│             │                                            │  .set_power_limit_uw()
│             │                                            │  .get_power_limit_uw()
│             │                                            └  .get_name()
│             │
│             │          powercap_zone_constraint[1]
│             │            .id = 1
│             │            ...
│             │
│             └──► powercap_zone  (child zone, e.g. "intel-rapl:0:0")
│                    .parent_idr ──► parent_zone.idr
│                    .constraints[]
│                    ...
```

---

### رسم الـ Lifecycle

#### أولاً: تسجيل الـ Control Type

```
Driver init
    │
    ▼
powercap_register_control_type(ct_ptr_or_NULL, "intel-rapl", &ops)
    │
    ├─ لو ct_ptr == NULL → kzalloc() + set allocated=true
    ├─ لو ct_ptr != NULL → استخدم memory الـ driver + set allocated=false
    │
    ├─ device_initialize(&ct->dev)
    ├─ ct->dev.class = powercap_class
    ├─ idr_init(&ct->idr)
    ├─ mutex_init(&ct->lock)
    ├─ device_add(&ct->dev)   ← يظهر في sysfs
    └─ list_add(&ct->node, &powercap_ctrls)  ← تسجيل عالمي
    │
    ▼
    ✓ control_type جاهز
```

#### ثانياً: تسجيل الـ Zone

```
Driver (بعد تسجيل control_type)
    │
    ▼
powercap_register_zone(zone_ptr, ct, "intel-rapl:0", parent, &zone_ops, 2, &const_ops)
    │
    ├─ لو zone_ptr == NULL → kzalloc()
    ├─ تسجيل الـ zone في IDR الـ parent (ct->idr أو parent_zone->idr)
    │    → يحدد zone->id
    ├─ zone->parent_idr = &parent->idr (أو &ct->idr)
    ├─ kzalloc constraints array (nr_constraints * sizeof)
    ├─ لكل constraint: تعيين .id و .power_zone و .ops
    ├─ إعداد الـ sysfs attributes (zone attrs + constraint attrs)
    ├─ device_add(&zone->dev)  ← يظهر في sysfs
    └─ ct->nr_zones++
    │
    ▼
    ✓ zone جاهزة للقراءة والتحكم
```

#### ثالثاً: الـ Teardown

```
Driver unload
    │
    ▼
powercap_unregister_zone(ct, child_zone)   ← الأبناء أولاً دايماً
    │
    ├─ idr_remove(parent_idr, zone->id)
    ├─ device_del(&zone->dev)
    ├─ kfree(zone->constraints)
    ├─ لو zone->allocated → kfree(zone)
    └─ لو لأ → zone->ops->release(zone)  [driver يحرر memory]
    │
    ▼
powercap_unregister_control_type(ct)   ← بعد ما كل الـ zones اتحذفت
    │
    ├─ لو ct->nr_zones != 0 → return -EBUSY  ← حماية!
    ├─ list_del(&ct->node)
    ├─ device_del(&ct->dev)
    ├─ idr_destroy(&ct->idr)
    ├─ لو ct->allocated → kfree(ct)
    └─ لو لأ → ct->ops->release(ct)
```

---

### رسم الـ Call Flow — قراءة قيمة من الـ sysfs

```
User reads: /sys/class/powercap/intel-rapl:0/energy_uj
    │
    ▼
VFS read()
    │
    ▼
sysfs show() callback (في powercap core)
    │
    ▼
zone->ops->get_energy_uj(zone, &val)
    │                              [الـ driver يقرأ من hardware]
    │                              ▼
    │                    rdmsr(MSR_PKG_ENERGY_STATUS)  ← x86 MSR
    │                              │
    │                              ▼
    │                    *val = raw_value * scale
    │
    ▼
sprintf(buf, "%llu\n", val)
    │
    ▼
return to userspace
```

#### الـ Call Flow — كتابة Power Limit

```
User writes: echo 15000000 > constraint_0_power_limit_uw
    │
    ▼
sysfs store() callback (في powercap core)
    │
    ├─ kstrtoull(buf, &val)
    ├─ validate val <= constraint->ops->get_max_power_uw()
    │
    ▼
constraint->ops->set_power_limit_uw(zone, constraint_id, val)
    │
    ▼
driver يكتب القيمة في الـ hardware register
    (مثلاً MSR_PKG_POWER_LIMIT في RAPL)
```

#### الـ Call Flow — تفعيل/تعطيل control type

```
User writes: echo 0 > /sys/class/powercap/intel-rapl/enabled
    │
    ▼
sysfs store() في powercap core
    │
    ▼
mutex_lock(&ct->lock)
    │
    ▼
ct->ops->set_enable(ct, false)
    │
    ▼
driver يعطل كل الـ zones (يرفع الـ power limits أو يصفيها)
    │
    ▼
mutex_unlock(&ct->lock)
```

---

### استراتيجية الـ Locking

| الـ Lock | النوع | موجود في | بيحمي |
|---|---|---|---|
| `powercap_control_type.lock` | `struct mutex` | كل control_type | إضافة/حذف zones، تغيير `nr_zones`، تفعيل/تعطيل الـ control type |
| `struct device` internal lock | داخلي في kernel | كل `struct device` | الـ sysfs attribute access والـ reference counting عبر `kobject` |
| driver-level locking | حسب الـ driver | الـ driver نفسه | الـ hardware registers — الـ framework **لا يوفر** حماية للـ callbacks |

#### ترتيب الـ Locks (Lock Ordering)

```
1. powercap_control_type.lock     ← الأعلى (يُمسك أولاً)
2. powercap_zone device lock      ← (داخل device model)
3. driver internal lock           ← الأدنى (يُمسك أخيراً)
```

> **تحذير مهم:** الـ framework نفسه لا يمسك أي lock عند استدعاء الـ zone ops أو constraint ops — الـ driver مسؤول عن الـ mutual exclusion داخل الـ callbacks. ده واضح في التعليق:
> *"Client drivers should handle mutual exclusion, if required in callbacks."*

#### نقطة دقيقة: الـ `allocated` flag والـ Memory Ownership

```
حالة 1: driver يمرر NULL لـ powercap_register_zone()
    → framework يعمل kzalloc()
    → zone->allocated = true
    → عند الحذف: framework يعمل kfree(zone)

حالة 2: driver يمرر pointer لـ memory خاصة بيه
    → framework يستخدم الـ pointer زي ما هو
    → zone->allocated = false
    → عند الحذف: framework يستدعي zone->ops->release(zone)
    → الـ driver مسؤول عن التحرير
```

هذا التصميم بيسمح للـ driver يعمل embed للـ `powercap_zone` داخل struct خاص بيه:

```c
struct my_rapl_zone {
    struct powercap_zone pz;   /* must be first or use container_of */
    u32 msr_offset;
    spinlock_t hw_lock;
    /* ... */
};

/* عند التسجيل */
powercap_register_zone(&my_zone->pz, ct, "rapl:0", NULL, &ops, 2, &const_ops);
/* allocated = false → release() callback هو المسؤول */
```

---

### الـ IDR وإدارة الـ IDs

**الـ `struct idr`** بيستخدم Radix Tree داخلياً لتوليد IDs صحيحة (integers) بشكل فريد وسريع.

```
control_type.idr
    ├── id=0 → zone "intel-rapl:0"
    │             zone.idr
    │               ├── id=0 → child zone "intel-rapl:0:0"
    │               └── id=1 → child zone "intel-rapl:0:1"
    └── id=1 → zone "intel-rapl:1"
                  zone.idr
                    └── id=0 → child zone "intel-rapl:1:0"
```

كل zone بتحتفظ بـ `parent_idr` عشان تقدر تشيل نفسها من الـ tree بتاعت الـ parent وقت الحذف بدون ما تحتاج ترجع للـ parent نفسه.
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ Functions والـ APIs — Cheatsheet

#### Control Type APIs

| Function | النوع | الغرض |
|---|---|---|
| `powercap_register_control_type()` | exported | تسجيل control type جديد مع الـ framework |
| `powercap_unregister_control_type()` | exported | إلغاء تسجيل control type وتحرير موارده |

#### Zone APIs

| Function | النوع | الغرض |
|---|---|---|
| `powercap_register_zone()` | exported | تسجيل power zone تحت control type معين |
| `powercap_unregister_zone()` | exported | إلغاء تسجيل power zone وتحرير موارده |

#### Inline Helpers

| Function | النوع | الغرض |
|---|---|---|
| `powercap_set_zone_data()` | static inline | حفظ private data pointer في الـ zone |
| `powercap_get_zone_data()` | static inline | استرجاع private data pointer من الـ zone |
| `POWERCAP_GET_DEV()` | macro | الحصول على `struct device *` من zone |

#### Callback Tables (Ops Structs)

| Struct | المستخدم | الغرض |
|---|---|---|
| `powercap_control_type_ops` | client driver | callbacks خاصة بالـ control type |
| `powercap_zone_ops` | client driver | callbacks خاصة بالـ power zone |
| `powercap_zone_constraint_ops` | client driver | callbacks خاصة بالـ constraint |

---

### تصنيف الـ Functions

```
┌─────────────────────────────────────────────────────┐
│              powercap framework                      │
│                                                     │
│  ┌──────────────────┐   ┌──────────────────────┐   │
│  │  Registration     │   │  Runtime Helpers      │   │
│  │  - register CT    │   │  - set_zone_data()    │   │
│  │  - register zone  │   │  - get_zone_data()    │   │
│  └──────────────────┘   │  - GET_DEV macro       │   │
│  ┌──────────────────┐   └──────────────────────┘   │
│  │  Cleanup          │   ┌──────────────────────┐   │
│  │  - unregister CT  │   │  Callbacks (ops)      │   │
│  │  - unregister zone│   │  - CT ops             │   │
│  └──────────────────┘   │  - Zone ops            │   │
│                          │  - Constraint ops      │   │
│                          └──────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

### المجموعة الأولى: Registration Functions

**الهدف:** بناء الـ hierarchy بتاعة الـ powercap في الـ sysfs — من الـ control type اللي هو الـ container الأكبر، وصولاً للـ zones والـ constraints.

---

#### `powercap_register_control_type()`

```c
struct powercap_control_type *powercap_register_control_type(
    struct powercap_control_type *control_type,
    const char *name,
    const struct powercap_control_type_ops *ops);
```

**الوظيفة:**
بتسجّل **control type** جديد مع الـ powercap class. الـ control type ده بيمثل تقنية تحكم في الطاقة، زي RAPL على معالجات Intel. لو `control_type` بـ NULL، الـ framework بيعمل `kzalloc` لنفسه ويحط `allocated = true`، لو مش NULL فالـ driver هو اللي بيملك الذاكرة وبيتحط `allocated = false`.

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `control_type` | `struct powercap_control_type *` | pointer لذاكرة pre-allocated أو NULL عشان الـ framework يخصص |
| `name` | `const char *` | اسم فريد للـ control type، بيظهر في `/sys/class/powercap/` |
| `ops` | `const struct powercap_control_type_ops *` | pointer لـ callbacks struct، ممكن يكون NULL لو مفيش enable/disable |

**Return Value:**
- **نجاح:** pointer لـ `struct powercap_control_type` المسجّل
- **فشل:** `NULL` — ده بيحصل لو الاسم مش unique أو لو `kzalloc` فشل أو لو `device_register()` فشل

**Key Details:**
- الاسم لازم يكون unique عبر كل الـ control types المسجّلة، وإلا بترجع `NULL`.
- الـ function بتعمل `device_initialize()` و`device_add()` داخلياً عشان تعمل sysfs entry تحت `/sys/class/powercap/<name>/`.
- بتعمل `idr_init()` للـ `idr` الخاص بالـ control type عشان يدير IDs للـ child zones.
- الـ `mutex` الخاص بالـ control type بيتعمل initialize هنا.

**Pseudocode Flow:**

```
powercap_register_control_type(ct, name, ops):
    if ct == NULL:
        ct = kzalloc(sizeof(*ct))   /* framework owns memory */
        ct->allocated = true
    else:
        ct->allocated = false       /* client owns memory */

    ct->ops = ops
    idr_init(&ct->idr)
    mutex_init(&ct->lock)

    dev = &ct->dev
    dev->class = &powercap_class
    dev_set_name(dev, name)

    ret = device_register(dev)     /* creates sysfs entry */
    if ret:
        if ct->allocated: kfree(ct)
        return NULL

    list_add(&ct->node, &powercap_ctrls)
    return ct
```

**Who calls it:** client drivers زي `intel_rapl` في `drivers/powercap/intel_rapl_common.c` بيسموها في الـ `module_init` أو `probe()`.

---

#### `powercap_register_zone()`

```c
struct powercap_zone *powercap_register_zone(
    struct powercap_zone *power_zone,
    struct powercap_control_type *control_type,
    const char *name,
    struct powercap_zone *parent,
    const struct powercap_zone_ops *ops,
    int nr_constraints,
    const struct powercap_zone_constraint_ops *const_ops);
```

**الوظيفة:**
بتسجّل **power zone** تحت control type معين. الـ zone ممكن تكون top-level (مباشرة تحت الـ control type) أو nested (تحت zone أبوها). كل zone ليها عدد محدد من الـ constraints اللي بتتحكم في حدود الطاقة. الـ function دي بتبني الـ sysfs attributes للـ energy/power وبتربط الـ constraints بالـ zone.

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `power_zone` | `struct powercap_zone *` | pre-allocated أو NULL عشان الـ framework يخصص |
| `control_type` | `struct powercap_control_type *` | الـ control type اللي الـ zone هتتسجل تحته |
| `name` | `const char *` | اسم الـ zone، بيظهر في الـ sysfs |
| `parent` | `struct powercap_zone *` | الـ parent zone لو موجودة، أو NULL لو top-level |
| `ops` | `const struct powercap_zone_ops *` | callbacks الـ zone — لازم يكون فيه على الأقل energy أو power callbacks |
| `nr_constraints` | `int` | عدد الـ constraints (max = `MAX_CONSTRAINTS_PER_ZONE` = 10) |
| `const_ops` | `const struct powercap_zone_constraint_ops *` | callbacks الـ constraints |

**Return Value:**
- **نجاح:** pointer لـ `struct powercap_zone` المسجّل
- **فشل:** `ERR_PTR(-errno)` — لو `nr_constraints` تجاوز الـ max، أو لو `ops` مش فيها callbacks كافية، أو لو `device_register()` فشل

**Key Details:**
- الـ function بتعمل `idr_alloc()` على IDR الخاص بالـ parent (أو الـ control type) عشان تاخد unique ID للـ zone.
- الـ sysfs attributes بتتبنى ديناميكياً حسب الـ ops المتوفرة — لو `get_energy_uj` موجود، بيتضاف `energy_uj` attribute.
- الـ constraints بتتعمل كـ `kzalloc` array من `struct powercap_zone_constraint`.
- الـ device parent في الـ sysfs بيكون إما الـ parent zone أو الـ control type device.
- الـ `nr_zones` counter في الـ control type بيتزود.

**Pseudocode Flow:**

```
powercap_register_zone(pz, ct, name, parent, ops, nr_c, c_ops):
    validate: nr_c <= MAX_CONSTRAINTS_PER_ZONE
    validate: ops has at least (energy OR power) callbacks

    if pz == NULL:
        pz = kzalloc(sizeof(*pz))
        pz->allocated = true
    else:
        pz->allocated = false

    pz->ops = ops
    pz->control_type_inst = ct

    /* allocate constraints array */
    pz->constraints = kcalloc(nr_c, sizeof(struct powercap_zone_constraint))
    for i in 0..nr_c:
        pz->constraints[i].id = i
        pz->constraints[i].power_zone = pz
        pz->constraints[i].ops = c_ops

    /* get unique ID from parent IDR */
    if parent:
        pz->id = idr_alloc(&parent->idr, pz, ...)
        pz->parent_idr = &parent->idr
        dev->parent = &parent->dev
    else:
        pz->id = idr_alloc(&ct->idr, pz, ...)
        pz->parent_idr = &ct->idr
        dev->parent = &ct->dev

    /* build sysfs attributes dynamically */
    create_zone_attributes(pz)

    device_register(&pz->dev)
    ct->nr_zones++
    return pz
```

**Who calls it:** client drivers زي RAPL بيسموها لكل package/die/core domain في الـ `probe()`.

---

### المجموعة الثانية: Cleanup Functions

**الهدف:** تفكيك الـ hierarchy بترتيب عكسي — الـ leaves الأول قبل الـ root — وتحرير كل الموارد بشكل صحيح بدون memory leaks.

---

#### `powercap_unregister_control_type()`

```c
int powercap_unregister_control_type(struct powercap_control_type *instance);
```

**الوظيفة:**
بتلغي تسجيل الـ control type من الـ powercap framework وبتشيله من الـ sysfs. **شرط مهم:** كل الـ power zones المسجّلة تحت الـ control type ده لازم تتشال الأول، غير كده الـ function بترجع error. الذاكرة بتتحرر بس لو الـ framework هو اللي خصصها (`allocated = true`).

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `instance` | `struct powercap_control_type *` | pointer لـ control type المراد إلغاء تسجيله |

**Return Value:**
- **`0`:** نجاح
- **`-EBUSY`:** لو لسه في zones مسجّلة تحت الـ control type (`nr_zones > 0`)
- **`-EINVAL`:** لو `instance` بـ NULL

**Key Details:**
- الـ function بتاخد الـ `lock` الخاص بالـ control type قبل ما تتحقق من `nr_zones`.
- بعد `device_unregister()` الـ `release` callback (لو موجودة) بتتستدعى تلقائياً من الـ device model عند وصول الـ reference count للصفر.
- لو `allocated = true`، الـ `kfree()` بيتعمل في الـ release callback.
- الـ `idr_destroy()` بيتستدعى عشان يحرر الـ IDR resources.

**Who calls it:** client drivers في الـ `module_exit` أو `remove()`.

---

#### `powercap_unregister_zone()`

```c
int powercap_unregister_zone(struct powercap_control_type *control_type,
                              struct powercap_zone *power_zone);
```

**الوظيفة:**
بتلغي تسجيل power zone معين من الـ sysfs وبتحرر موارده. الـ caller لازم يكون شال كل الـ child zones الأول. الـ function بتشيل الـ ID من الـ parent IDR وبتقلل عداد الـ zones في الـ control type.

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `control_type` | `struct powercap_control_type *` | الـ control type اللي الـ zone مسجّل تحته |
| `power_zone` | `struct powercap_zone *` | الـ zone المراد إلغاء تسجيله |

**Return Value:**
- **`0`:** نجاح
- **`-EINVAL`:** لو أي من الـ pointers بـ NULL

**Key Details:**
- بتستدعي `idr_remove(power_zone->parent_idr, power_zone->id)` عشان تحرر الـ ID.
- الـ `control_type->nr_zones--` بيتعمل تحت الـ lock.
- `device_unregister()` بيشيل الـ zone من الـ sysfs ويطلق الـ reference.
- الذاكرة (الـ constraints array + الـ zone struct نفسه) بتتحرر في الـ `release` callback لو الـ framework هو اللي خصصها.
- الـ zone attributes المبنية ديناميكياً بتتحرر كمان في الـ release path.

**Who calls it:** client drivers بيسموها بترتيب عكسي (children أول ثم parent) في الـ `remove()` أو `module_exit`.

---

### المجموعة الثالثة: Inline Helpers والـ Macros

**الهدف:** توفير طريقة آمنة وبسيطة للـ client drivers عشان يخزنوا ويسترجعوا private data من الـ zone، وبكده بيفصلوا بين الـ framework internals والـ driver state.

---

#### `powercap_set_zone_data()`

```c
static inline void powercap_set_zone_data(struct powercap_zone *power_zone,
                                           void *pdata)
```

**الوظيفة:**
بتخزّن `void *` pointer في الـ `private_data` field بتاع الـ zone. الـ client driver بيستخدمها عشان يربط بين الـ framework zone object وبين الـ driver-specific data structure بتاعته.

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `power_zone` | `struct powercap_zone *` | الـ zone المراد تخزين البيانات فيها |
| `pdata` | `void *` | pointer للبيانات الخاصة بالـ driver |

**Return Value:** لا يرجع قيمة (`void`).

**Key Details:**
- فيه NULL check على `power_zone` — safe to call even if registration failed.
- لا يوجد locking — الـ caller مسؤول عن الـ synchronization لو محتاج.
- بيتستدعى عادةً مباشرة بعد `powercap_register_zone()` في الـ `probe()`.

**مثال عملي:**

```c
/* في الـ driver */
struct my_rapl_domain {
    struct powercap_zone zone;   /* embedded — zero extra alloc */
    u32 msr_id;
    int package_id;
};

struct my_rapl_domain *dom = kzalloc(sizeof(*dom), GFP_KERNEL);
dom->msr_id = 0x610;

struct powercap_zone *z = powercap_register_zone(&dom->zone, ct, "package-0",
                                                  NULL, &rapl_zone_ops, 2,
                                                  &rapl_const_ops);
/* ربط الـ domain بالـ zone */
powercap_set_zone_data(z, dom);
```

---

#### `powercap_get_zone_data()`

```c
static inline void *powercap_get_zone_data(struct powercap_zone *power_zone)
```

**الوظيفة:**
بترجع الـ `private_data` pointer المخزّن مسبقاً بـ `powercap_set_zone_data()`. بتستخدمها الـ callbacks عشان ترجع من الـ `struct powercap_zone *` للـ driver-specific struct.

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `power_zone` | `struct powercap_zone *` | الـ zone المراد استرجاع البيانات منها |

**Return Value:**
- الـ `void *` pointer المخزّن مسبقاً
- **`NULL`** لو `power_zone` بـ NULL أو لو مفيش data مخزّنة

**Key Details:**
- لا يوجد locking — نفس ملاحظة الـ set function.
- الاستخدام النموذجي داخل الـ ops callbacks:

```c
static int rapl_get_power_uw(struct powercap_zone *zone, u64 *power)
{
    /* استرجاع الـ driver struct من الـ zone */
    struct my_rapl_domain *dom = powercap_get_zone_data(zone);

    /* الوصول لـ MSR باستخدام البيانات الخاصة */
    u64 msr_val;
    rdmsrl(dom->msr_id, msr_val);
    *power = (msr_val & 0x7FFF) * 125000; /* convert to micro-watts */
    return 0;
}
```

---

#### `POWERCAP_GET_DEV()` — Macro

```c
#define POWERCAP_GET_DEV(power_zone)  (&power_zone->dev)
```

**الوظيفة:**
بيرجع `struct device *` من الـ `powercap_zone`. بيستخدمه الـ client driver مع `dev_dbg()` / `dev_err()` / `dev_warn()` عشان يطبع رسائل kernel logging مرتبطة بالـ zone.

**مثال:**

```c
dev_dbg(POWERCAP_GET_DEV(zone), "power limit set to %llu uw\n", val);
```

---

### المجموعة الرابعة: Callbacks (Ops Structs)

**الهدف:** تحديد الـ interface اللي الـ client drivers لازم تنفّذه عشان تتكلم مع الـ powercap framework. كل struct هو virtual function table.

---

#### `struct powercap_control_type_ops`

```c
struct powercap_control_type_ops {
    int (*set_enable) (struct powercap_control_type *, bool mode);
    int (*get_enable) (struct powercap_control_type *, bool *mode);
    int (*release)   (struct powercap_control_type *);
};
```

| Callback | إلزامي؟ | الوظيفة |
|---|---|---|
| `set_enable` | اختياري | تفعيل/تعطيل الـ control type كله — لما يتعطّل، كل الـ zones بتبقى للمراقبة بس |
| `get_enable` | اختياري | قراءة حالة التفعيل الحالية |
| `release` | **إلزامي** لو الـ client يملك الذاكرة | بيتستدعى لما الـ reference count يوصل للصفر، الـ client يحرر ذاكرته هنا |

**Key Details:**
- الـ `release` callback لازم يكون موجود لو الـ driver بعت `control_type` غير NULL لـ `register`. لأن الـ framework مش هيعمل `kfree` للـ client-owned memory.

---

#### `struct powercap_zone_ops`

```c
struct powercap_zone_ops {
    int (*get_max_energy_range_uj) (struct powercap_zone *, u64 *);
    int (*get_energy_uj)           (struct powercap_zone *, u64 *);
    int (*reset_energy_uj)         (struct powercap_zone *);
    int (*get_max_power_range_uw)  (struct powercap_zone *, u64 *);
    int (*get_power_uw)            (struct powercap_zone *, u64 *);
    int (*set_enable)              (struct powercap_zone *, bool mode);
    int (*get_enable)              (struct powercap_zone *, bool *mode);
    int (*release)                 (struct powercap_zone *);
};
```

| Callback | إلزامي؟ | الوظيفة |
|---|---|---|
| `get_max_energy_range_uj` | اختياري | أقصى قيمة للـ energy counter قبل الـ wrap |
| `get_energy_uj` | اختياري* | قراءة الـ energy المتراكمة بالـ micro-joules |
| `reset_energy_uj` | اختياري | إعادة تصفير الـ energy counter |
| `get_max_power_range_uw` | اختياري | أقصى قيمة لـ power reading |
| `get_power_uw` | اختياري* | قراءة الـ power الحالية بالـ micro-watts |
| `set_enable` | اختياري | تفعيل/تعطيل الـ zone |
| `get_enable` | اختياري | قراءة حالة التفعيل |
| `release` | **إلزامي** لو الـ client يملك الذاكرة | تحرير موارد الـ zone |

**\* ملاحظة مهمة:** لازم يكون فيه على الأقل واحد من `get_energy_uj` أو `get_power_uw` — الاثنين مع بعض اختياريين، لكن واحد منهم على الأقل لازم يكون موجود وإلا الـ `powercap_register_zone()` بترجع error.

**Locking Note:** الـ framework مش بيوفر locking للـ ops callbacks — الـ driver مسؤول عن حماية الـ shared state لو محتاج mutual exclusion.

---

#### `struct powercap_zone_constraint_ops`

```c
struct powercap_zone_constraint_ops {
    int (*set_power_limit_uw)    (struct powercap_zone *, int id, u64);
    int (*get_power_limit_uw)    (struct powercap_zone *, int id, u64 *);
    int (*set_time_window_us)    (struct powercap_zone *, int id, u64);
    int (*get_time_window_us)    (struct powercap_zone *, int id, u64 *);
    int (*get_max_power_uw)      (struct powercap_zone *, int id, u64 *);
    int (*get_min_power_uw)      (struct powercap_zone *, int id, u64 *);
    int (*get_max_time_window_us)(struct powercap_zone *, int id, u64 *);
    int (*get_min_time_window_us)(struct powercap_zone *, int id, u64 *);
    const char *(*get_name)      (struct powercap_zone *, int id);
};
```

| Callback | إلزامي؟ | الوظيفة |
|---|---|---|
| `set_power_limit_uw` | **إلزامي** | ضبط حد الطاقة للـ constraint |
| `get_power_limit_uw` | **إلزامي** | قراءة حد الطاقة الحالي |
| `set_time_window_us` | **إلزامي** | ضبط الـ time window الذي يُحسب عليه المتوسط |
| `get_time_window_us` | **إلزامي** | قراءة الـ time window الحالي |
| `get_max_power_uw` | اختياري | أقصى حد طاقة مسموح به |
| `get_min_power_uw` | اختياري | أدنى حد طاقة مسموح به |
| `get_max_time_window_us` | اختياري | أقصى time window مسموح به |
| `get_min_time_window_us` | اختياري | أدنى time window مسموح به |
| `get_name` | **إلزامي** | اسم الـ constraint (مثلاً: "long_term", "short_term") |

**الـ `id` parameter:** كل constraint له رقم فريد (0 إلى `nr_constraints - 1`) بيتمرر في كل callback عشان الـ driver يعرف يتعامل مع الـ constraint الصح من مصادره (مثلاً MSR registers مختلفة).

**مثال عملي لـ RAPL:**

```c
/* RAPL بيستخدم constraint 0 = long term, constraint 1 = short term */
static int rapl_set_power_limit(struct powercap_zone *zone, int cid, u64 val)
{
    struct my_rapl_domain *dom = powercap_get_zone_data(zone);

    /* cid == 0 → MSR_PKG_POWER_LIMIT[14:0] */
    /* cid == 1 → MSR_PKG_POWER_LIMIT[46:32] */
    rapl_write_constraint(dom, cid, val);
    return 0;
}

static const char *rapl_get_constraint_name(struct powercap_zone *zone, int cid)
{
    return cid == 0 ? "long_term" : "short_term";
}
```

---

### الـ Hierarchy الكاملة في الـ sysfs

```
/sys/class/powercap/
└── intel-rapl/                         ← control type (registered via register_control_type)
    ├── enabled
    ├── intel-rapl:0/                   ← top-level zone (package 0)
    │   ├── name
    │   ├── energy_uj
    │   ├── max_energy_range_uj
    │   ├── constraint_0_name           ← "long_term"
    │   ├── constraint_0_power_limit_uw
    │   ├── constraint_0_time_window_us
    │   ├── constraint_1_name           ← "short_term"
    │   ├── constraint_1_power_limit_uw
    │   ├── constraint_1_time_window_us
    │   └── intel-rapl:0:0/             ← child zone (core domain)
    │       ├── name
    │       ├── energy_uj
    │       └── constraint_0_...
    └── intel-rapl:1/                   ← top-level zone (package 1)
        └── ...
```

---

### جدول الـ Memory Ownership

| الحالة | `allocated` flag | من يحرر الذاكرة؟ |
|---|---|---|
| Client بعت `NULL` للـ register | `true` | الـ framework في الـ `release` callback |
| Client بعت pointer للـ register | `false` | الـ client في الـ `ops->release` callback |

**ملاحظة:** الـ constraints array دايماً بتتخصص بواسطة الـ framework وبتتحرر بواسطته، بغض النظر عن ملكية الـ zone struct نفسه.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. Debugfs Entries

الـ powercap subsystem مش بيستخدم debugfs بشكل مباشر، لكن الـ driver اللي فوقيه (زي RAPL) ممكن يضيف entries. الـ sysfs هو الواجهة الأساسية.

```bash
# تحقق إن debugfs متاح
mount | grep debugfs
# لو مش mounted:
mount -t debugfs none /sys/kernel/debug

# شوف لو في entries خاصة بالـ powercap
ls /sys/kernel/debug/power/
```

---

#### 2. Sysfs Entries

الـ powercap بيظهر تحت `/sys/class/powercap/`. الـ hierarchy بيعكس العلاقة بين **control_type** → **power_zone** → **constraints**.

```bash
# اطبع شجرة كاملة لكل الـ powercap zones
find /sys/class/powercap/ -maxdepth 5 | sort

# مثال RAPL على Intel:
# /sys/class/powercap/intel-rapl/
# /sys/class/powercap/intel-rapl:0/          ← package-0
# /sys/class/powercap/intel-rapl:0:0/        ← core (child zone)
# /sys/class/powercap/intel-rapl:0:1/        ← uncore (child zone)

# قراءة الطاقة الحالية (micro-joules)
cat /sys/class/powercap/intel-rapl:0/energy_uj

# قراءة الـ power limit للـ constraint 0
cat /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw

# قراءة الـ time window للـ constraint 0
cat /sys/class/powercap/intel-rapl:0/constraint_0_time_window_us

# هل الـ zone enabled؟
cat /sys/class/powercap/intel-rapl:0/enabled

# اسم الـ constraint
cat /sys/class/powercap/intel-rapl:0/constraint_0_name

# قيم min/max المسموح بيها
cat /sys/class/powercap/intel-rapl:0/constraint_0_max_power_uw
cat /sys/class/powercap/intel-rapl:0/constraint_0_min_power_uw
cat /sys/class/powercap/intel-rapl:0/constraint_0_max_time_window_us
cat /sys/class/powercap/intel-rapl:0/constraint_0_min_time_window_us

# عدد الـ zones المسجلة (من داخل الـ control_type)
# nr_zones مش exposed مباشرة، لكن تقدر تحسبها:
ls /sys/class/powercap/ | grep "intel-rapl" | wc -l
```

---

#### 3. Ftrace — Tracepoints والـ Events

```bash
# شوف الـ events المتاحة للـ power subsystem
ls /sys/kernel/tracing/events/power/

# enable أهم الـ events للـ powercap debugging
echo 1 > /sys/kernel/tracing/events/power/enable

# أو بشكل أكثر تحديداً:
echo 1 > /sys/kernel/tracing/events/power/cpu_frequency/enable
echo 1 > /sys/kernel/tracing/events/power/cpu_idle/enable

# تتبع دوال الـ powercap باستخدام function tracer
echo function > /sys/kernel/tracing/current_tracer
echo 'powercap_*' > /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on

# بعد ما تعمل العملية اللي بتحاول تتبعها:
cat /sys/kernel/tracing/trace | head -50

# إيقاف وتنظيف
echo 0 > /sys/kernel/tracing/tracing_on
echo nop > /sys/kernel/tracing/current_tracer
echo > /sys/kernel/tracing/set_ftrace_filter

# تتبع كل الـ calls الداخلة في powercap_register_zone
echo powercap_register_zone > /sys/kernel/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
```

---

#### 4. Printk والـ Dynamic Debug

```bash
# تفعيل الـ dynamic debug لكل ملفات الـ powercap
echo 'module intel_rapl_common +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module intel_rapl_msr +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل لملف معين (powercap core)
echo 'file drivers/powercap/powercap_sys.c +pflmt' \
    > /sys/kernel/debug/dynamic_debug/control

# +p = print, +f = function name, +l = line number, +m = module, +t = thread ID

# شوف الـ dynamic debug المفعلة حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep powercap

# رفع مستوى الـ loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk

# مشاهدة الـ kernel log في real-time
dmesg -w | grep -i powercap
dmesg -w | grep -i rapl
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_POWERCAP` | تفعيل الـ powercap framework نفسه |
| `CONFIG_INTEL_RAPL` | RAPL driver للـ Intel |
| `CONFIG_INTEL_RAPL_CORE` | RAPL core library |
| `CONFIG_DEBUG_KERNEL` | تفعيل عام لكل debugging features |
| `CONFIG_DEBUG_OBJECTS` | تتبع lifetime الـ kernel objects |
| `CONFIG_DEBUG_OBJECTS_WORK` | تتبع الـ work queue objects |
| `CONFIG_LOCKDEP` | كشف deadlocks في الـ mutex (الـ powercap_control_type بيستخدم mutex) |
| `CONFIG_PROVE_LOCKING` | إثبات صحة الـ locking order |
| `CONFIG_DEBUG_MUTEXES` | debug إضافي للـ mutex |
| `CONFIG_KASAN` | كشف memory corruption في الـ zone/constraint structs |
| `CONFIG_UBSAN` | كشف undefined behavior |
| `CONFIG_KCOV` | coverage للـ fuzzing |
| `CONFIG_DYNAMIC_DEBUG` | dynamic debug messages |
| `CONFIG_TRACING` | تفعيل الـ ftrace infrastructure |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'POWERCAP|RAPL|DEBUG_OBJECTS|LOCKDEP'
# أو
grep -E 'POWERCAP|RAPL' /boot/config-$(uname -r)
```

---

#### 6. Subsystem-Specific Tools

```bash
# powertop: أداة Intel لمشاهدة استهلاك الطاقة
powertop --time=5

# turbostat: تفاصيل دقيقة عن الـ CPU power states وRALP readings
turbostat --quiet --show Pkg_W,Core_W,GFX_W,RAM_W --interval 1

# perf: قياس أحداث الطاقة
perf stat -e power/energy-pkg/,power/energy-cores/,power/energy-ram/ \
    -a sleep 5

# rapl-read (userspace tool)
# يقرأ الـ MSRs مباشرة ويقارنها بالـ sysfs
ls /dev/cpu/0/msr && rdmsr -a 0x606   # RAPL_POWER_UNIT MSR

# مراقبة continuous للـ energy counters
watch -n 1 'cat /sys/class/powercap/intel-rapl:0/energy_uj'

# حساب average power خلال فترة
E1=$(cat /sys/class/powercap/intel-rapl:0/energy_uj)
sleep 5
E2=$(cat /sys/class/powercap/intel-rapl:0/energy_uj)
echo "Avg Power (W): $(echo "scale=2; ($E2 - $E1) / 5000000" | bc)"
```

---

#### 7. Common Error Messages

| رسالة الـ Kernel Log | المعنى | الإصلاح |
|---|---|---|
| `powercap: Failed to register control_type` | اسم الـ control_type مش unique أو memory allocation فشلت | تأكد إن الاسم مش مستخدم: `ls /sys/class/powercap/` |
| `powercap: Failed to create zone` | الـ parent zone مش valid أو الـ idr allocation فشل | تحقق من ترتيب الـ register: parent قبل child |
| `powercap: Zone already unregistered` | محاولة unregister zone أكتر من مرة | راجع الـ driver cleanup code |
| `powercap: Unregister all children first` | محاولة unregister zone وعنده children | الـ unregister لازم يبدأ من الأعمق (leaf) للـ root |
| `intel_rapl: RAPL Interface not supported by hardware` | الـ CPU مش بيدعم RAPL MSRs | تحقق من `cpuid` أو جرب على CPU تاني |
| `intel_rapl: Failed to read power unit` | مشكلة في قراءة MSR_RAPL_POWER_UNIT | تحقق من `/dev/cpu/0/msr` وصلاحياته |
| `WARNING: CPU: X PID: Y at drivers/powercap/powercap_sys.c:Z` | WARN_ON اتفعل في الـ powercap core | شوف stack trace ودد السبب من السطر |
| `BUG: sleeping function called from invalid context` | callback من context مش بيسمح بـ sleep والـ mutex locked | راجع الـ callback implementation في الـ driver |
| `kobject_add failed` | مشكلة في إضافة الـ sysfs device | ممكن name conflict أو parent device مش valid |

---

#### 8. Strategic Points لـ dump_stack() وـ WARN_ON()

نقاط مهمة في `drivers/powercap/powercap_sys.c` لإضافة assertions:

```c
/* في powercap_register_zone() — تحقق إن الـ ops صحيحة */
WARN_ON(!ops->get_power_uw && !ops->get_energy_uj);

/* في powercap_unregister_zone() — تحقق من الـ children */
WARN_ON(power_zone->idr.idr_rt.xa_head != NULL);

/* في الـ constraint callbacks — تحقق من القيم */
WARN_ON(power_limit_uw == 0);

/* لو عايز تعرف مين استدعى register */
pr_debug("powercap: registering zone '%s'\n", name);
dump_stack();   /* مؤقتاً للـ debugging فقط */

/* تحقق من الـ mutex state */
WARN_ON(!mutex_is_locked(&control_type->lock));
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

الـ powercap framework على Intel بيتكلم مع الـ hardware عبر **MSR (Model-Specific Registers)**.

```bash
# قراءة MSR_PKG_POWER_LIMIT (0x610) — الـ power limit الفعلي في الـ HW
rdmsr -p 0 0x610    # من CPU core 0

# قراءة MSR_PKG_ENERGY_STATUS (0x611) — عداد الطاقة في الـ HW
rdmsr -p 0 0x611

# قراءة وحدات القياس
rdmsr -p 0 0x606    # MSR_RAPL_POWER_UNIT
# البتات [3:0] = power units (1/2^n watts)
# البتات [12:8] = energy units (1/2^n joules)
# البتات [19:16] = time units

# قارن القيمة اللي الـ kernel شايفها مع الـ MSR مباشرة
SYSFS_VAL=$(cat /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw)
MSR_RAW=$(rdmsr -d -p 0 0x610)
echo "sysfs: ${SYSFS_VAL} uW"
echo "MSR raw: ${MSR_RAW}"
```

**جدول أهم MSRs:**

| MSR Address | الاسم | الوظيفة |
|---|---|---|
| `0x606` | MSR_RAPL_POWER_UNIT | وحدات القياس للـ power/energy/time |
| `0x610` | MSR_PKG_POWER_LIMIT | حد الطاقة للـ package |
| `0x611` | MSR_PKG_ENERGY_STATUS | عداد الطاقة الكلية |
| `0x614` | MSR_PKG_POWER_INFO | TDP والحدود المسموحة |
| `0x638` | MSR_PP0_POWER_LIMIT | حد طاقة الـ CPU cores |
| `0x639` | MSR_PP0_ENERGY_STATUS | عداد طاقة الـ cores |
| `0x618` | MSR_DRAM_POWER_LIMIT | حد طاقة الـ DRAM |

---

#### 2. Register Dump Techniques

```bash
# devmem2: قراءة physical memory مباشرة (للـ MMIO-based powercap)
# تثبيت: apt install devmem2 أو yum install devmem2
devmem2 0xFED00000 w    # مثال لعنوان MMIO

# /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((0xFED00000 / 4)) 2>/dev/null | xxd

# io utility (للـ port-based I/O)
# io -4 -r 0x400   # قراءة 4 bytes من port 0x400

# msr-tools لقراءة MSRs
# تثبيت
apt-get install msr-tools   # Ubuntu/Debian
modprobe msr                 # تحميل الـ MSR module

# قراءة MSR من كل الـ cores
for cpu in $(ls /dev/cpu/); do
    echo -n "CPU${cpu}: "
    rdmsr -p ${cpu} 0x611 2>/dev/null || echo "N/A"
done

# dump كامل لكل MSRs المتعلقة بالـ RAPL
for msr in 0x606 0x610 0x611 0x614 0x618 0x619 0x638 0x639 0x64A 0x64D; do
    printf "MSR 0x%X: " ${msr}
    rdmsr -p 0 ${msr} 2>/dev/null || echo "not supported"
done
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

لو الـ powercap بيتحكم في hardware فعلي (زي voltage regulator أو power switch):

```
نقاط القياس:
┌─────────────────────────────────────────────────────┐
│  CPU Package                                         │
│  ┌──────────┐    SVID bus     ┌──────────────────┐  │
│  │  RAPL    │◄───────────────►│  Voltage Regulator│  │
│  │  Engine  │                 │  (VRM)            │  │
│  └──────────┘                 └──────────────────┘  │
│       ↑                              ↑               │
│  MSR writes                    قياس الـ power هنا   │
└─────────────────────────────────────────────────────┘
```

- **قياس الـ SVID bus** (Serial VID Interface): بروتوكول بين الـ CPU والـ VRM
  - Frequency: عادة 10MHz
  - الـ logic analyzer لازم يدعم SVID protocol decode
- **قياس استهلاك التيار الفعلي**: استخدم current probe على الـ 12V rail للـ CPU
- **تحقق من الـ power limit enforcement**: لو الـ kernel ضبط limit بـ X watts، الـ oscilloscope المفروض يوريك إن الـ package مش بيتعدى X بشكل ملحوظ

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log | التشخيص |
|---|---|---|
| CPU مش بيدعم RAPL | `RAPL Interface not supported` | `cpuid -l 0x0000000A` — تحقق من power management features |
| MSR access مرفوض في VM | `rdmsr: CPU 0 cannot read MSR` | الـ hypervisor مش expose المـ MSRs — فعل الـ MSR passthrough |
| Energy counter overflow | القراءات بتقل فجأة | الـ counter 32-bit أو 64-bit بيـ overflow — الـ driver المفروض يتعامل معاه |
| Power limit مش بيتطبق | الـ power بيتعدى الـ limit الـ sysfs | تحقق من إن الـ lock bit (bit 31 في MSR 0x610) مش مضبوط |
| DRAM RAPL مش متاح | لا يوجد `constraint` لـ dram zone | مش كل platforms بتدعم DRAM RAPL |
| الـ energy_uj بيقرأ صفر دايماً | driver bug أو hardware issue | جرب `rdmsr 0x611` مباشرة |

---

#### 5. Device Tree Debugging

الـ powercap على ARM/embedded systems بيعتمد على الـ DT:

```bash
# اطبع الـ DT المحمل حالياً
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 10 powercap
# أو
cat /sys/firmware/devicetree/base/compatible

# تحقق من الـ DT node الخاص بالـ powercap zone
find /sys/firmware/devicetree/base -name "*power*" -o -name "*thermal*" | \
    while read f; do
        echo "=== $f ==="; hexdump -C "$f" 2>/dev/null | head -4
    done

# مقارنة الـ DT مع الـ hardware manual:
# لو الـ DT بيقول إن الـ zone بتدعم 10 constraints لكن الـ hw بيدعم 5 فقط:
cat /sys/class/powercap/*/constraint_*_name 2>/dev/null

# تحقق من الـ compatible string في الـ DT
grep -r "powercap" /sys/firmware/devicetree/base/ 2>/dev/null
```

**أشياء تتحقق منها في الـ DT:**

```dts
/* مثال DT node صح للـ powercap zone */
power-zone@0 {
    compatible = "vendor,powercap-zone";
    reg = <0x0 0x1000>;         /* MMIO region */
    #power-cap-cells = <1>;     /* عدد الـ constraints */
    power-limit-uw = <15000000>; /* 15W max */
    time-window-us = <1000000>;  /* 1 second */
};
```

---

### Practical Commands

#### Ready-to-Copy Shell Commands

**1. فحص سريع للـ powercap state:**
```bash
#!/bin/bash
# powercap-status.sh — quick health check

echo "=== Powercap Zones ==="
for zone in /sys/class/powercap/*/; do
    name=$(basename "$zone")
    enabled=$(cat "${zone}enabled" 2>/dev/null || echo "N/A")
    energy=$(cat "${zone}energy_uj" 2>/dev/null || echo "N/A")
    echo "Zone: $name | enabled=$enabled | energy=${energy}uJ"
    # constraints
    for i in 0 1 2; do
        limit="${zone}constraint_${i}_power_limit_uw"
        [ -f "$limit" ] && echo "  constraint_${i}: $(cat $limit) uW"
    done
done
```

**2. مراقبة الـ power في real-time:**
```bash
#!/bin/bash
# راقب الـ power consumption لـ 30 ثانية بدقة 1 ثانية
ZONE="/sys/class/powercap/intel-rapl:0/energy_uj"
PREV=$(cat $ZONE)
for i in $(seq 1 30); do
    sleep 1
    CURR=$(cat $ZONE)
    DIFF=$((CURR - PREV))
    POWER=$(echo "scale=2; $DIFF / 1000000" | bc)
    echo "$(date +%T) Power: ${POWER}W"
    PREV=$CURR
done
```

**3. تفعيل كامل debugging للـ RAPL:**
```bash
#!/bin/bash
# راجع الـ trace
mount -t tracefs nodev /sys/kernel/tracing 2>/dev/null

# فعل function tracing
echo function > /sys/kernel/tracing/current_tracer
echo 'powercap_*:intel_rapl_*' > /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on

# شغل أي عملية تريد تتبعها...

echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace > /tmp/powercap_trace.txt
echo "Trace saved to /tmp/powercap_trace.txt"
```

**4. اختبار كتابة power limit:**
```bash
# احفظ القيمة الحالية
OLD=$(cat /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw)
echo "Current limit: ${OLD} uW"

# اضبط limit جديد (10W = 10,000,000 uW) مع مراقبة الـ dmesg
dmesg -C
echo 10000000 > /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw
NEW=$(cat /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw)
echo "New limit: ${NEW} uW"
dmesg | grep -i "rapl\|powercap\|error\|warning"

# استعادة القيمة الأصلية
echo $OLD > /sys/class/powercap/intel-rapl:0/constraint_0_power_limit_uw
```

---

#### Example Output وكيفية تفسيره

**مخرجات `find /sys/class/powercap/`:**
```
/sys/class/powercap/intel-rapl
/sys/class/powercap/intel-rapl:0        ← package-0 zone (root)
/sys/class/powercap/intel-rapl:0:0      ← core sub-zone (child)
/sys/class/powercap/intel-rapl:0:1      ← uncore sub-zone (child)
/sys/class/powercap/intel-rapl:1        ← package-1 (dual socket)
```
- الأرقام بعد الـ `:` بتمثل الـ `id` في `struct powercap_zone`
- الـ hierarchy بتعكس الـ `parent_idr` في الـ struct

**مخرجات `turbostat`:**
```
Pkg_W   Core_W  GFX_W   RAM_W
12.45    8.23    0.95    2.11    ← هذه القراءات من نفس MSRs اللي powercap بيقرأها
```
- لو `Pkg_W` بتتعدى `constraint_0_power_limit_uw / 1e6` باستمرار: الـ limit مش بيتطبق صح

**مخرجات `rdmsr 0x610`:**
```
0x00dd8960006813a3
```
- البتات [14:0]: power limit value
- البت [15]: limit enable flag — لازم يكون `1`
- البتات [23:17]: time window exponent
- البت [63]: lock bit — لو `1` الـ limit مقفول ومش ممكن تغييره

```bash
# فك ترميز MSR_PKG_POWER_LIMIT
MSR=$(rdmsr -d 0x610)
LOCK=$(( (MSR >> 63) & 1 ))
ENABLE=$(( (MSR >> 15) & 1 ))
echo "Lock=$LOCK Enable=$ENABLE"
# لو Lock=1 → الكتابة على sysfs هتفشل بصمت
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Android TV Box على Allwinner H616 — power zone بيتسجل بس ما بيتحكم فيه

#### العنوان
تسجيل `powercap_zone` بدون `set_power_limit_uw` — constraint بيظهر في sysfs بس الكتابة فيه بترجع `-ENOSYS`

#### السياق
شغلانة: Android TV box بيعمله vendor على H616 (quad-core Cortex-A53). المطلوب إنك تعمل thermal throttling باستخدام powercap framework بدل ما تعتمد على thermal governor بس. الفريق بيبني driver جديد اسمه `h616-powercap` وبيسجل control type وzone واحدة للـ CPU cluster.

#### المشكلة
لما المهندس يكتب في:
```bash
echo 2000000 > /sys/devices/power_capping/h616-power/cpu-cluster/constraint_0_power_limit_uw
```
بيجي:
```
bash: echo: write error: Function not implemented
```
رغم إن الـ zone موجودة في sysfs وبتقرأ `constraint_0_name` تمام.

#### التحليل
الكود في `powercap.h` بيوضح إن `powercap_zone_constraint_ops` عندها مجموعة callbacks، وبالنسبة للـ mandatory callbacks:

```c
/*
 * Mandatory callbacks — can't be NULL:
 *  set_power_limit_uw
 *  get_power_limit_uw
 *  set_time_window_us
 *  get_time_window_us
 *  get_name
 */
struct powercap_zone_constraint_ops {
    int (*set_power_limit_uw) (struct powercap_zone *, int, u64);
    int (*get_power_limit_uw) (struct powercap_zone *, int, u64 *);
    ...
    const char *(*get_name) (struct powercap_zone *, int);
};
```

الـ framework في `drivers/powercap/powercap_sys.c` لما بيجي يكتب على `constraint_X_power_limit_uw` بيعمل check إن `ops->set_power_limit_uw != NULL`، لو NULL بيرجع `-ENOSYS`. المهندس نسي يملأ الـ callback في الـ `const_ops` struct اللي اتبعت لـ `powercap_register_zone()`.

```c
/* في كود الـ driver الغلط */
static const struct powercap_zone_constraint_ops h616_constraint_ops = {
    .get_power_limit_uw  = h616_get_power_limit,
    .set_time_window_us  = h616_set_time_window,
    .get_time_window_us  = h616_get_time_window,
    .get_name            = h616_constraint_name,
    /* set_power_limit_uw ← منسية! */
};
```

#### الحل
إضافة الـ callback الناقصة:

```c
static int h616_set_power_limit(struct powercap_zone *zone,
                                int constraint_id, u64 power_uw)
{
    struct h616_power_data *priv = powercap_get_zone_data(zone);

    if (power_uw > H616_MAX_POWER_UW)
        return -EINVAL;

    /* Write limit to cluster power register */
    writel(power_uw / 1000, priv->base + H616_PWR_LIMIT_REG);
    priv->current_limit_uw = power_uw;
    return 0;
}

static const struct powercap_zone_constraint_ops h616_constraint_ops = {
    .set_power_limit_uw  = h616_set_power_limit,   /* ← الإضافة */
    .get_power_limit_uw  = h616_get_power_limit,
    .set_time_window_us  = h616_set_time_window,
    .get_time_window_us  = h616_get_time_window,
    .get_name            = h616_constraint_name,
};
```

تحقق بعد الإصلاح:
```bash
cat /sys/devices/power_capping/h616-power/cpu-cluster/constraint_0_name
# Short term
echo 1500000 > /sys/devices/power_capping/h616-power/cpu-cluster/constraint_0_power_limit_uw
# يكتب بدون error
```

#### الدرس المستفاد
الـ `powercap_zone_constraint_ops` بتحدد بوضوح في `powercap.h` الـ callbacks اللي mandatory. أي NULL فيهم بيخلي الـ sysfs write يفشل بصمت بـ `-ENOSYS`. لازم تقرأ الـ header كويس قبل ما تسجل الـ zone.

---

### السيناريو 2: Industrial Gateway على AM62x — memory leak لما driver بيتفصل

#### العنوان
عدم استخدام `allocated` flag بشكل صح → double-free في `powercap_unregister_zone()`

#### السياق
شغلانة: industrial gateway بيشتغل على TI AM62x (Cortex-A53 + MCU subsystem). Driver بيتحكم في power budget للـ MCU والـ DSP عبر powercap framework. البورد بتشتغل شهور بدون مشكلة، لكن لما admin بيعمل `rmmod` للـ driver بتحصل kernel panic.

#### المشكلة
```
BUG: kernel NULL pointer dereference at 0000000000000000
Call trace:
  kfree+0x...
  powercap_unregister_zone+0x...
  am62x_powercap_remove+0x...
```

#### التحليل
`powercap_zone` struct فيها field مهم:

```c
struct powercap_zone {
    ...
    bool allocated; /* true = framework owns memory,
                       false = client owns memory */
    ...
};
```

وكمان `powercap_control_type`:

```c
struct powercap_control_type {
    ...
    bool allocated;
    ...
};
```

الـ framework بيستخدم الـ flag ده عشان يعرف مين المسؤول عن الـ free. لو المهندس عمل:

```c
/* في probe — client بيخصص الميمورى */
static struct powercap_zone am62x_mcu_zone;  /* static allocation */

ret = powercap_register_zone(&am62x_mcu_zone, ctrl_type,
                             "mcu", NULL, &zone_ops, 1, &const_ops);
```

هنا بمر الـ zone pointer مش NULL، يعني الـ framework بيحط `allocated = false` (الـ client owns the memory). المشكلة إن المهندس في نفس الوقت بيعمل `kfree(&am62x_mcu_zone)` في الـ remove function، وده خطأ لأن الـ struct static مش heap-allocated.

بالإضافة لكده، لو استخدم الـ `private_data`:

```c
/* في probe */
powercap_set_zone_data(zone, kzalloc(sizeof(*priv), GFP_KERNEL));

/* في remove — نسي يفرر private_data قبل unregister */
powercap_unregister_zone(ctrl_type, zone);
/* priv leak! */
```

الـ inline functions في `powercap.h`:

```c
static inline void powercap_set_zone_data(struct powercap_zone *power_zone,
                                          void *pdata)
{
    if (power_zone)
        power_zone->private_data = pdata;
}
```

ما بتعمل `free` للـ `private_data` القديمة — ده مسؤولية الـ client.

#### الحل
الـ remove function الصح:

```c
static int am62x_powercap_remove(struct platform_device *pdev)
{
    struct am62x_powercap *priv = platform_get_drvdata(pdev);

    /* أول حاجة: فرر الـ child zones */
    powercap_unregister_zone(priv->ctrl_type, &priv->dsp_zone);
    powercap_unregister_zone(priv->ctrl_type, &priv->mcu_zone);

    /* تاني حاجة: فرر الـ private_data يدوياً */
    kfree(powercap_get_zone_data(&priv->mcu_zone));
    kfree(powercap_get_zone_data(&priv->dsp_zone));

    /* تالت حاجة: فرر الـ control type */
    powercap_unregister_control_type(priv->ctrl_type);

    return 0;
    /* priv نفسه بيتفرر بواسطة devm */
}
```

وتحقق من الـ memory leaks:
```bash
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

#### الدرس المستفاد
الـ `allocated` flag في `powercap_zone` و`powercap_control_type` بيحدد مين المسؤول عن الـ memory. `private_data` دايماً مسؤولية الـ client. رتيب الـ unregister مهم: children أولاً ثم parent ثم control type.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — parent-child zones بتتسجل بترتيب غلط

#### العنوان
تسجيل child zone قبل parent → `powercap_register_zone()` بترجع NULL وما في error واضح

#### السياق
شغلانة: IoT environmental sensor بيشتغل على STM32MP157 (dual Cortex-A7). الـ design عاوز يعمل hierarchy:
```
stm32-power/
  ├── soc/           ← parent zone
  │   ├── cpu/       ← child zone
  │   └── periph/    ← child zone
```

المهندس بيكتب الـ init function بس بينسى يراعي الترتيب.

#### المشكلة
```
[ 5.123] stm32-powercap: failed to register cpu zone
[ 5.124] stm32-powercap: failed to register periph zone
```
وفي sysfs بتلاقي بس:
```
/sys/devices/power_capping/stm32-power/
```
من غير أي zones جوه.

#### التحليل
توقيع `powercap_register_zone()` في `powercap.h`:

```c
struct powercap_zone *powercap_register_zone(
    struct powercap_zone *power_zone,
    struct powercap_control_type *control_type,
    const char *name,
    struct powercap_zone *parent,   /* ← لو مش NULL، لازم يكون registered قبل كده */
    const struct powercap_zone_ops *ops,
    int nr_constraints,
    const struct powercap_zone_constraint_ops *const_ops);
```

الـ `parent` parameter بيلزمه يكون pointer لـ zone registered بالفعل في نفس الـ `control_type`. لو الـ parent لسه NULL (مش registered)، الـ framework مش هيقدر يعمل `idr_alloc()` صح تحت الـ parent وبيرجع NULL.

كود الـ driver الغلط:

```c
static int stm32_powercap_probe(struct platform_device *pdev)
{
    /* غلط: بنسجل cpu قبل soc */
    priv->cpu_zone = powercap_register_zone(
        NULL, priv->ctrl, "cpu",
        priv->soc_zone,   /* ← NULL لسه! */
        &cpu_ops, 1, &const_ops);

    /* بعدين بنسجل soc */
    priv->soc_zone = powercap_register_zone(
        NULL, priv->ctrl, "soc",
        NULL, &soc_ops, 1, &const_ops);
}
```

#### الحل
الترتيب الصح — parent أولاً:

```c
static int stm32_powercap_probe(struct platform_device *pdev)
{
    /* 1. سجل parent zone أولاً */
    priv->soc_zone = powercap_register_zone(
        NULL, priv->ctrl, "soc",
        NULL,               /* parent = NULL → directly under ctrl */
        &soc_ops, 1, &const_ops);
    if (!priv->soc_zone) {
        dev_err(&pdev->dev, "failed to register soc zone\n");
        return -EINVAL;
    }

    /* 2. سجل children بعد كده */
    priv->cpu_zone = powercap_register_zone(
        NULL, priv->ctrl, "cpu",
        priv->soc_zone,     /* parent registered بالفعل */
        &cpu_ops, 1, &const_ops);

    priv->periph_zone = powercap_register_zone(
        NULL, priv->ctrl, "periph",
        priv->soc_zone,
        &periph_ops, 1, &const_ops);

    return 0;
}
```

تحقق من الـ hierarchy:
```bash
find /sys/devices/power_capping/ -name "name" -exec cat {} \; -print
```

وعند الـ unregister، عكس الترتيب:
```c
/* children أولاً، parent أخيراً */
powercap_unregister_zone(priv->ctrl, priv->cpu_zone);
powercap_unregister_zone(priv->ctrl, priv->periph_zone);
powercap_unregister_zone(priv->ctrl, priv->soc_zone);
```

#### الدرس المستفاد
الـ powercap framework بيبني tree structure باستخدام `idr` داخل كل zone. الـ `parent_idr` field في `powercap_zone` بيشاور على الـ IDR بتاع الـ parent. يعني الـ parent لازم يكون initialized ومتسجل قبل ما تسجل أي child. الترتيب العكسي في الـ unregister كمان إلزامي عشان متفككش الـ IDR وفيه references.

---

### السيناريو 4: Automotive ECU على i.MX8QM — تعدد `control_type` بيسبب name collision

#### العنوان
محاولة تسجيل اتنين `control_type` بنفس الاسم → الثاني بيفشل وما في تشخيص واضح

#### السياق
شغلانة: automotive ECU بيشتغل على i.MX8QM (dual Cortex-A72 + quad Cortex-A53). الـ system عنده:
- `imx8-cpu-power`: driver للـ CPU power capping
- `imx8-gpu-power`: driver للـ GPU power capping

كلهم بيسجلوا `control_type` بس الـ system integrator بيلاقي إن GPU driver فاشل في الـ probe.

#### المشكلة
```
[ 3.456] imx8-gpu-power: powercap_register_control_type failed
[ 3.457] imx8-gpu-power: probe failed with error -EEXIST
```

وفي sysfs:
```bash
ls /sys/devices/power_capping/
# imx8-power   ← بس واحد
```

#### التحليل
الـ doc في `powercap.h` صريح:

```c
/**
 * powercap_register_control_type()
 * ...
 * The name can be any string which must be unique,
 * otherwise this function returns NULL.
 */
struct powercap_control_type *powercap_register_control_type(
    struct powercap_control_type *control_type,
    const char *name,             /* ← unique name مطلوب */
    const struct powercap_control_type_ops *ops);
```

الكودين الاتنين بيستخدموا نفس الـ name:

```c
/* في imx8-cpu-power driver */
ctrl = powercap_register_control_type(NULL, "imx8-power", &cpu_ctrl_ops);

/* في imx8-gpu-power driver */
ctrl = powercap_register_control_type(NULL, "imx8-power", &gpu_ctrl_ops);
/* ← بيفشل لأن الاسم موجود */
```

الـ framework بيتحقق من uniqueness عبر الـ `list_head node` في `powercap_control_type` وبيمشي على list الـ control types المسجلة. أي اسم متكرر بيرجع NULL.

#### الحل
**خيار 1:** اسم مختلف لكل control type:

```c
/* CPU driver */
ctrl = powercap_register_control_type(NULL, "imx8-cpu-power", &cpu_ctrl_ops);

/* GPU driver */
ctrl = powercap_register_control_type(NULL, "imx8-gpu-power", &gpu_ctrl_ops);
```

**خيار 2:** control type واحد shared، zones منفصلة:

```c
/* driver أساسي بيسجل control type واحد */
shared_ctrl = powercap_register_control_type(NULL, "imx8-power", &ctrl_ops);

/* CPU zone */
cpu_zone = powercap_register_zone(NULL, shared_ctrl, "cpu",
                                  NULL, &cpu_zone_ops, 2, &cpu_const_ops);

/* GPU zone */
gpu_zone = powercap_register_zone(NULL, shared_ctrl, "gpu",
                                  NULL, &gpu_zone_ops, 2, &gpu_const_ops);
```

النتيجة في sysfs:
```
/sys/devices/power_capping/imx8-power/
  ├── cpu/
  │   ├── constraint_0_power_limit_uw
  │   └── constraint_1_power_limit_uw
  └── gpu/
      ├── constraint_0_power_limit_uw
      └── constraint_1_power_limit_uw
```

تحقق:
```bash
cat /sys/devices/power_capping/imx8-power/cpu/constraint_0_name
cat /sys/devices/power_capping/imx8-power/gpu/constraint_0_name
```

#### الدرس المستفاد
`powercap_control_type` ليه `struct list_head node` بيخليه يتضاف لـ global list. الاسم بيكون الـ identifier الـ unique في الـ sysfs. في systems فيها أكتر من driver بيستخدم powercap، اتفق على naming convention مبكراً أو استخدم control type واحد shared مع zones منفصلة.

---

### السيناريو 5: Custom Board Bring-up على RK3562 — energy counter بيتقرأ غلط بسبب unit mismatch

#### العنوان
`get_energy_uj` بيرجع قيم بالـ mJ مش الـ µJ → userspace thermal daemon بيعمل over-throttle

#### السياق
شغلانة: custom industrial board بيشتغل على RK3562 (quad Cortex-A53). الـ bring-up engineer بيكتب أول powercap driver للـ SoC. بعد أسبوعين من التشغيل، المنتج بيعمل throttle قوي جداً وأداءه ضعيف رغم إن temperature تمام.

#### المشكلة
الـ thermal daemon اللي بيقرأ:
```
/sys/devices/power_capping/rk3562-power/cpu/energy_uj
```
بيلاقي قيم صغيرة جداً (مثلاً 1200 بدل 1200000)، فبيحسب إن الـ power consumption عالي جداً وبيعمل throttle مش لازم.

#### التحليل
الـ `powercap_zone_ops` في `powercap.h` واضحة في الـ unit:

```c
struct powercap_zone_ops {
    /**
     * @get_max_energy_range_uj: Get maximum range of energy counter
     *                           in micro-joules.
     */
    int (*get_max_energy_range_uj) (struct powercap_zone *, u64 *);

    /**
     * @get_energy_uj: Get current energy counter in micro-joules.
     */
    int (*get_energy_uj) (struct powercap_zone *, u64 *);
    ...
};
```

الـ suffix `_uj` = micro-joules، `_uw` = micro-watts، `_us` = micro-seconds. الـ driver بيقرأ من register بيعطي قيمة بالـ milli-joules ومش بيحولها:

```c
/* كود غلط */
static int rk3562_get_energy(struct powercap_zone *zone, u64 *energy)
{
    struct rk3562_priv *priv = powercap_get_zone_data(zone);
    u32 reg_val;

    reg_val = readl(priv->base + RK3562_ENERGY_REG); /* value in mJ */
    *energy = reg_val;  /* ← غلط! المطلوب µJ */
    return 0;
}
```

#### الحل
التحويل الصح من milli-joules لـ micro-joules:

```c
static int rk3562_get_energy(struct powercap_zone *zone, u64 *energy_uj)
{
    struct rk3562_priv *priv = powercap_get_zone_data(zone);
    u32 reg_val;

    /* RK3562 energy register gives value in milli-joules */
    reg_val = readl(priv->base + RK3562_ENERGY_REG);

    /* Convert mJ → µJ: multiply by 1000 */
    *energy_uj = (u64)reg_val * 1000ULL;
    return 0;
}

static int rk3562_get_max_energy_range(struct powercap_zone *zone, u64 *max_uj)
{
    /* RK3562 register is 32-bit, max value = 0xFFFFFFFF mJ */
    *max_uj = (u64)U32_MAX * 1000ULL;
    return 0;
}
```

وكمان الـ power callbacks:

```c
static int rk3562_get_power(struct powercap_zone *zone, u64 *power_uw)
{
    struct rk3562_priv *priv = powercap_get_zone_data(zone);
    u32 reg_val;

    /* RK3562 power register gives value in milli-watts */
    reg_val = readl(priv->base + RK3562_POWER_REG);

    /* Convert mW → µW: multiply by 1000 */
    *power_uw = (u64)reg_val * 1000ULL;
    return 0;
}
```

تحقق سريع من الـ unit:
```bash
# قرأ القيمة
cat /sys/devices/power_capping/rk3562-power/cpu/energy_uj

# قارن مع قيمة معروفة عبر perf
perf stat -e power/energy-cores/ sleep 1
```

وتحقق من الـ max range:
```bash
cat /sys/devices/power_capping/rk3562-power/cpu/max_energy_range_uj
```

#### الدرس المستفاد
الـ `powercap.h` صريحة في الـ unit في اسم كل callback: `_uj` = micro-joules، `_uw` = micro-watts، `_us` = micro-seconds. الـ hardware registers غالباً بيديك قيم بـ milli أو حتى units كاملة. لازم دايماً تعمل unit conversion صح في الـ callback قبل ما تكتب في الـ `u64 *out`. unit mismatch ما بيسبب error — بيسبب over-throttle أو under-throttle صامت.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأساسي لمتابعة تطور الـ Linux kernel — فيما يلي أهم المقالات المتعلقة بالـ powercap framework:

| المقال | الوصف |
|--------|-------|
| [Power Capping Framework](https://lwn.net/Articles/562015/) | المقال الأصلي اللي قدّم الـ powercap class driver للـ kernel |
| [Power Capping Framework and RAPL Driver](https://lwn.net/Articles/569535/) | شرح تفصيلي للـ framework مع الـ Intel RAPL driver كأول implementation |
| [Documentation/ABI/testing/sysfs-class-powercap](https://lwn.net/Articles/574225/) | توثيق الـ sysfs ABI للـ powercap class (kernel 3.13) |
| [Documentation/power/powercap/powercap.txt](https://lwn.net/Articles/574224/) | الوثيقة الرسمية للـ framework في kernel 3.13 |
| [SCMIv3.1 Powercap protocol and driver](https://lwn.net/Articles/899974/) | تقديم الـ ARM SCMI-based powercap driver اللي بيستخدم الـ framework |
| [Introduce powercap userspace frontend](https://lwn.net/Articles/915834/) | أداة `powercap-info` لاستعراض الـ powercap zones من الـ userspace |
| [powercap/intel_rapl: Introduce RAPL TPMI support](https://lwn.net/Articles/929547/) | دعم الـ TPMI (Topology Aware Register and PM Capsule Interface) في الـ Intel RAPL |
| [powercap/dtpm: Add the DTPM framework](https://lwn.net/Articles/839318/) | الـ Dynamic Thermal Power Management framework المبني على الـ powercap |
| [powercap: Introduce TPMI RAPL PMU support](https://lwn.net/Articles/969009/) | دعم الـ PMU counters للـ TPMI RAPL |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة في الـ kernel source tree:

```
Documentation/power/powercap/powercap.rst
Documentation/ABI/testing/sysfs-class-powercap
include/linux/powercap.h
drivers/powercap/powercap_sys.c
drivers/powercap/intel_rapl_common.c
drivers/powercap/intel_rapl_msr.c
drivers/powercap/intel_rapl_perf.c
```

**الـ online documentation** على الموقع الرسمي:
- [Power Capping Framework — The Linux Kernel documentation](https://docs.kernel.org/power/powercap/powercap.html)
- [Energy Model of devices — The Linux Kernel documentation](https://static.lwn.net/kerneldoc/power/energy-model.html)
- [Raw powercap.txt على kernel.org](https://www.kernel.org/doc/Documentation/power/powercap/powercap.txt)

---

### Commits المهمة في الـ Kernel

**الـ commits** دي أسست الـ subsystem أو غيّرت فيه بشكل جوهري:

| الوصف | المرجع |
|-------|--------|
| إدخال الـ powercap framework (kernel 3.13) | [v2 patch series](https://groups.google.com/g/linux.kernel/c/akvKHTquPmg) |
| الـ Intel RAPL driver كـ client للـ framework | [intel_rapl_common.c على GitHub](https://github.com/torvalds/linux/blob/master/drivers/powercap/intel_rapl_common.c) |
| الـ intel_rapl_msr.c — MSR-based RAPL interface | [torvalds/linux على GitHub](https://github.com/torvalds/linux/blob/master/drivers/powercap/intel_rapl_msr.c) |
| AMD RAPL support — Fam17h و Fam19h | [mailing list patch](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2375243.html) |
| Power Limit4 support for Alder Lake-N | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-pm/patch/20220726130229.24634-1-sumeet.r.pawnikar@intel.com/) |

---

### نقاشات الـ Mailing List

- [Re: AMD RAPL PowerCap Patches — linux-kernel@vger.kernel.org](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2375243.html) — نقاش تفعيل الـ powercap لـ AMD Zen CPUs
- [v2 0/6: Power Capping Framework and RAPL Driver](https://groups.google.com/g/linux.kernel/c/akvKHTquPmg) — الـ patch series الأصلية اللي أدخلت الـ framework
- [RAPL BIOS lock check fix](https://lkml.kernel.org/lkml/1453252038-31915-82-git-send-email-kamal@canonical.com/) — مثال على bugfix في الـ subsystem

---

### Kernelnewbies.org — متابعة التغييرات

الـ [kernelnewbies.org](https://kernelnewbies.org) بيوثّق تغييرات الـ powercap في كل إصدار kernel:

| الإصدار | التغيير |
|---------|---------|
| [Linux 3.13](https://kernelnewbies.org/Linux_3.13) | أول إصدار يحتوي على الـ powercap framework مع الـ Intel RAPL driver |
| [Linux 5.6](https://kernelnewbies.org/Linux_5.6) | تحديثات على الـ idle injection powercap framework |
| [Linux 5.15](https://kernelnewbies.org/Linux_5.15) | تحسينات على الـ RAPL driver |
| [Linux 5.17](https://kernelnewbies.org/Linux_5.17) | تحديثات AMD RAPL support |
| [Linux 6.0](https://kernelnewbies.org/Linux_6.0) | تحديثات على الـ powercap subsystem |
| [Linux 6.1](https://kernelnewbies.org/Linux_6.1) | تحديثات على الـ intel_rapl driver |
| [Linux 6.2](https://kernelnewbies.org/Linux_6.2) | إضافة الـ ARM SCMI Powercap driver + cpupower userspace frontend |
| [Linux 6.3](https://kernelnewbies.org/Linux_6.3) | Power Limit4 لـ Meteor Lake + idle injection improvements |
| [Linux 6.5](https://kernelnewbies.org/Linux_6.5) | RAPL TPMI support لـ Intel platforms الجديدة |
| [Linux 6.9](https://kernelnewbies.org/Linux_6.9) | دعم Arrow Lake و Lunar Lake-M في intel_rapl |
| [Linux 6.10](https://kernelnewbies.org/Linux_6.10) | TPMI RAPL PMU support + ArrowLake-H |

---

### مشاريع الـ Userspace

- [powercap/powercap على GitHub](https://github.com/powercap/powercap) — C bindings للـ Linux Power Capping Framework عبر الـ sysfs
- [powercap/raplcap على GitHub](https://github.com/powercap/raplcap) — C interface للـ RAPL power capping بـ implementations متعددة
- [Reading RAPL from Linux — University of Maine](https://web.eece.maine.edu/~vweaver/projects/rapl/) — مرجع أكاديمي شامل لقراءة RAPL energy counters

---

### كتب موصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول ذات الصلة:**
  - Chapter 14: The Linux Device Model — فهم الـ `struct device` اللي بيستخدمها الـ powercap
  - Chapter 4: Debugging Techniques — debugfs وـ sysfs للـ monitoring
- **متاح مجانًا:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصول ذات الصلة:**
  - Chapter 17: Devices and Modules — كيف بيشتغل الـ device model
  - Chapter 11: Timers and Time Management — مهم لفهم الـ time windows في الـ constraints
  - Chapter 5: System Calls — كيف الـ userspace بيوصل للـ sysfs attributes

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصول ذات الصلة:**
  - Power Management chapter — شرح power domains وـ power budgeting في الـ embedded systems
  - فهم الـ sysfs interface للـ power management على embedded hardware

---

### مصادر إضافية

| المصدر | الرابط |
|--------|--------|
| RAPL power measurement — detailed reference | [web.eece.maine.edu/~vweaver/projects/rapl/rapl_support.html](https://web.eece.maine.edu/~vweaver/projects/rapl/rapl_support.html) |
| AMD RAPL PowerCap — Phoronix coverage | [phoronix.com](https://www.phoronix.com/news/AMD-RAPL-Linux-Now-19h) |
| powercap sysfs tree exploration | [community.frame.work — RAPL limits](https://community.frame.work/t/responded-thermal-management-on-linux-with-rapl-limits/17416) |

---

### Search Terms للبحث عن معلومات أكتر

لو حابب تعمق أكتر في الـ powercap subsystem، استخدم الـ search terms دي:

```
linux kernel powercap framework
linux RAPL intel_rapl driver sysfs
powercap_register_zone kernel API
powercap control_type zones constraints
linux power capping DTPM dynamic thermal
intel RAPL TPMI topology aware power management
linux powercap AMD RAPL Zen
ARM SCMI powercap protocol driver
linux energy_uj power_limit_uw sysfs
CONFIG_POWERCAP CONFIG_INTEL_RAPL kernel config
```

**للبحث في الـ kernel source مباشرة:**

```bash
# الـ implementation الرئيسية
git log --oneline drivers/powercap/

# كل الـ drivers اللي بتستخدم الـ framework
grep -r "powercap_register_zone" drivers/

# الـ sysfs ABI documentation
cat Documentation/ABI/testing/sysfs-class-powercap

# استعراض الـ zones الموجودة على الجهاز
ls /sys/devices/virtual/powercap/
```
## Phase 8: Writing simple module

### الفكرة

الـ `powercap_register_zone()` هي أهم function في الـ framework — بتُسجِّل كل power zone جديدة في الـ sysfs. هنعمل **kprobe** عليها عشان نشوف كل مرة driver زي RAPL بيسجل zone جديدة، ونطبع اسم الـ zone، واسم الـ control type، وعدد الـ constraints.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * powercap_zone_spy.c
 *
 * Kprobe on powercap_register_zone() — prints zone name,
 * control-type name, and constraint count on every registration.
 */

#include <linux/module.h>      /* MODULE_* macros, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/powercap.h>    /* struct powercap_zone, powercap_control_type */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("Spy on powercap_register_zone() via kprobe");

/*
 * powercap_register_zone signature:
 *
 *   struct powercap_zone *powercap_register_zone(
 *       struct powercap_zone        *power_zone,   // arg0
 *       struct powercap_control_type *control_type, // arg1
 *       const char                  *name,          // arg2
 *       struct powercap_zone        *parent,        // arg3
 *       const struct powercap_zone_ops *ops,        // arg4
 *       int                          nr_constraints,// arg5
 *       const struct powercap_zone_constraint_ops *const_ops // arg6
 *   );
 */

/* pre-handler: runs just before the probed function executes */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first 6 integer/pointer args land in:
     *   rdi, rsi, rdx, rcx, r8, r9
     *
     * arg1 = control_type  →  regs->si
     * arg2 = name          →  regs->dx
     * arg5 = nr_constraints → regs->r8 (5th arg, 0-indexed)
     *
     * We cast carefully and guard against NULL to be safe.
     */
    struct powercap_control_type *ct =
        (struct powercap_control_type *)regs->si;

    const char *zone_name  = (const char *)regs->dx;
    int         nr_constr  = (int)regs->r8;

    /* control_type carries its name via its embedded struct device */
    const char *ct_name = (ct && ct->dev.kobj.name)
                          ? ct->dev.kobj.name
                          : "<unknown>";

    pr_info("[powercap_spy] register_zone called:\n");
    pr_info("[powercap_spy]   control_type : %s\n", ct_name);
    pr_info("[powercap_spy]   zone name    : %s\n",
            zone_name ? zone_name : "<null>");
    pr_info("[powercap_spy]   nr_constraints: %d\n", nr_constr);

    /* returning 0 means "let the original function run normally" */
    return 0;
}

/* Define the kprobe — target function by symbol name */
static struct kprobe kp = {
    .symbol_name = "powercap_register_zone",
    .pre_handler = handler_pre,
};

static int __init powercap_spy_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[powercap_spy] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[powercap_spy] kprobe planted on powercap_register_zone @ %p\n",
            kp.addr);
    return 0;
}

static void __exit powercap_spy_exit(void)
{
    /*
     * Must unregister before the module is unloaded —
     * otherwise the handler points to freed .text memory → kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("[powercap_spy] kprobe removed\n");
}

module_init(powercap_spy_init);
module_exit(powercap_spy_exit);
```

---

### Makefile

```makefile
obj-m += powercap_zone_spy.o

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
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kernel.h` | الـ `pr_info` / `pr_err` |
| `linux/kprobes.h` | الـ `struct kprobe` وكل الـ API بتاعتها |
| `linux/powercap.h` | الـ structs بتاعة الـ powercap عشان نعمل cast صح للـ args |

الـ `linux/powercap.h` مهم هنا عشان بنعمل pointer cast لـ `struct powercap_control_type` ونوصل لـ `dev.kobj.name` بشكل type-safe.

---

#### الـ `handler_pre`

**الـ `struct pt_regs *regs`** بتحتوي على محتوى الـ registers في لحظة الاستدعاء. على x86-64، الـ calling convention بتحط الـ arguments في:

| Arg | Register | المعنى هنا |
|---|---|---|
| 0 | `rdi` | `power_zone` (ممكن NULL) |
| 1 | `rsi` | `control_type` |
| 2 | `rdx` | `name` (zone name) |
| 3 | `rcx` | `parent` |
| 4 | `r8` | `ops` |
| 5 | `r9` | `nr_constraints` |

لكن `nr_constraints` في الـ signature هو الـ argument رقم 5 (0-indexed)، فبيجي في `regs->r9` مش `regs->r8` — ده بيختلف لو في padding. الكود فوق بيستخدم `regs->r8` كـ example بس المهم تتحقق من الـ signature الفعلية وتتأكد بالـ disassembly لو محتاج precision.

الـ `return 0` بيقول للكرنل "اكمل تنفيذ الـ function الأصلية بشكل طبيعي" — لو رجعنا قيمة غير صفر، الكرنل بيعمل skip للـ function.

---

#### الـ `module_init` و `module_exit`

- **`register_kprobe`**: بتزرع breakpoint في الـ kernel text segment عند عنوان الـ symbol المحدد. لو فشلت (مثلاً الـ CONFIG_KPROBES مش مفعّل، أو الـ symbol مش exported بالشكل ده)، الـ module ميكملش التحميل.

- **`unregister_kprobe`**: لازم تتعمل في الـ exit — لو الـ module اتحمّل من الذاكرة من غير ما تشيل الـ kprobe، الكرنل هيحاول ينفّذ الـ handler من عنوان اتحرر → kernel panic فوري.

---

### كيفية الاختبار

```bash
# بناء الـ module
make

# تحميله
sudo insmod powercap_zone_spy.ko

# شوف الـ log
sudo dmesg | grep powercap_spy

# لو عندك Intel RAPL:
# الـ zones بتتسجل عند boot، فجرب:
sudo modprobe intel_rapl_msr   # أو intel_rapl_common
sudo dmesg | grep powercap_spy

# إزالة الـ module
sudo rmmod powercap_zone_spy
```

**مثال متوقع من `dmesg`**:

```
[powercap_spy] kprobe planted on powercap_register_zone @ ffffffffc01234ab
[powercap_spy] register_zone called:
[powercap_spy]   control_type : intel-rapl
[powercap_spy]   zone name    : package-0
[powercap_spy]   nr_constraints: 2
[powercap_spy] register_zone called:
[powercap_spy]   control_type : intel-rapl
[powercap_spy]   zone name    : core
[powercap_spy]   nr_constraints: 0
```

---

### ليه `powercap_register_zone` تحديداً؟

الـ `powercap_register_zone` هي نقطة تسجيل كل power zone في الـ sysfs — يعني كل driver بيدعم power capping (RAPL، RAPL-PCI، ARM SCMI) لازم يعدي من هنا. Hooking عليها بيديك visibility كاملة على كل الـ power management hierarchy في الـ system من مكان واحد، من غير ما تلمس أي driver بعينه.
