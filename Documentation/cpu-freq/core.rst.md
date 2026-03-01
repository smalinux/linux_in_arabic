## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف `Documentation/cpu-freq/core.rst` جزء من **CPUFreq subsystem** — وهو النظام المسؤول عن التحكم في سرعة المعالج داخل Linux kernel. الـ maintainers بيديروا كل حاجة في:

- `drivers/cpufreq/`
- `include/linux/cpufreq.h`
- `Documentation/cpu-freq/`
- `kernel/sched/cpufreq*.c`

---

### تخيل المشكلة

تخيل إنك بتشتغل على laptop. لما بتشتغل على Excel وبتكتب ببطء، المعالج مش محتاج يشتغل بأقصى سرعته — ده بيستهلك كهرباء أكتر وبيسخن البطارية بدون أي فايدة. بس لما تفتح لعبة أو تعمل render لفيديو، المعالج محتاج أقصى طاقة ممكنة.

**السؤال:** مين اللي بيقرر امتى المعالج يروح سريع وامتى يرجع بطيء؟ ومين بيعمل التغيير الفعلي في الـ hardware؟ وإيه اللي بيحصل للأجزاء التانية في الـ kernel لما السرعة تتغير؟

ده بالظبط اللي بيحله الـ **CPUFreq core**.

---

### الصورة الكبيرة

الـ CPUFreq core هو **الطبقة الوسطى** — زي مدير المشروع — اللي بيقف بين 3 أطراف:

```
┌─────────────────────────────────────────────────────┐
│                   USERSPACE / sysfs                 │
│          /sys/devices/system/cpu/cpu0/cpufreq/       │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│               CPUFreq Governor                      │
│   (schedutil / ondemand / performance / powersave)  │
│    بيقرر: "عايز تردد كام دلوقتي؟"                  │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│            *** CPUFreq CORE ***                     │
│         drivers/cpufreq/cpufreq.c                   │
│                                                     │
│  - بيديل policy unified interface                   │
│  - بيبعت notifications للـ notifiers               │
│  - بيتحكم في reference counting للـ policies       │
│  - بيعمل validation وbounds checking               │
│  - بيحدّث loops_per_jiffy                          │
└──────────┬──────────────────────────┬───────────────┘
           │                          │
┌──────────▼──────────┐   ┌──────────▼──────────────┐
│  CPUFreq Driver     │   │   CPUFreq Notifiers      │
│  (arch/hw specific) │   │ (thermal, timing, LCD..) │
│  بيعمل التغيير      │   │ بيتسمعوا لما السرعة      │
│  الفعلي في hardware │   │ تتغير أو الـ policy      │
│                     │   │ يتعدل                    │
│ intel_pstate.c      │   │ ACPI thermal             │
│ amd-pstate.c        │   │ loops_per_jiffy update   │
│ arm-cpufreq.c       │   │ LCD ARM drivers          │
└─────────────────────┘   └──────────────────────────┘
```

---

### ليه الـ CPUFreq Core موجود أصلاً؟

**بدونه** كان كل driver hardware لازم يتعامل مع:
- الـ governors بنفسه
- إخطار الـ thermal modules بنفسه
- التعامل مع الـ hotplug بنفسه
- الـ sysfs interface بنفسه

الـ core بيوحّد ده كله — الـ driver بيقول بس "انا عارف أروح من 800MHz لـ 2.4GHz"، والـ core بيتكفل بالباقي.

---

### المفاهيم الأساسية

#### الـ `cpufreq_policy`

ده القلب — بيمثل مجموعة CPUs بتشارك نفس الـ clock domain (زي CPU cores في نفس الـ cluster). فيه:

| الحقل | المعنى |
|-------|--------|
| `min` / `max` | حدود التردد المسموح بيها (بـ kHz) |
| `cur` | التردد الحالي |
| `governor` | الـ governor اللي بيتحكم في السياسة |
| `cpus` | الـ online CPUs المرتبطة بالـ policy دي |
| `cpuinfo` | الحدود الفعلية اللي الـ hardware يقدر يوصلها |
| `freq_table` | جدول الترددات المتاحة |
| `rwsem` | semaphore للـ thread safety |

#### الـ `cpufreq_driver`

الـ struct اللي كل driver hardware بيسجله مع الـ core. بيحتوي على function pointers زي:

```c
struct cpufreq_driver {
    int  (*init)(struct cpufreq_policy *policy);      /* initialize */
    int  (*verify)(struct cpufreq_policy_data *policy); /* validate limits */
    int  (*target_index)(struct cpufreq_policy *policy,
                         unsigned int index);          /* set frequency */
    unsigned int (*fast_switch)(struct cpufreq_policy *policy,
                                unsigned int target_freq); /* fast path */
    unsigned int (*get)(unsigned int cpu);            /* read current freq */
};
```

#### الـ `cpufreq_freqs`

الـ struct اللي بيتبعت مع الـ transition notifications:

```c
struct cpufreq_freqs {
    struct cpufreq_policy *policy;
    unsigned int old;   /* التردد القديم بالـ kHz */
    unsigned int new;   /* التردد الجديد بالـ kHz */
    u8 flags;
};
```

---

### الـ CPUFreq Notifiers — نظام الإخطار

لما السرعة بتتغير، ناس تانية في الـ kernel محتاجين يعرفوا. الـ core بيعمل ده بـ **notifier chains** — زي قائمة subscribers.

**في نوعين:**

##### 1. Policy Notifiers
بيتبعتوا لما policy اتعملت أو اتمسحت:
- `CPUFREQ_CREATE_POLICY` — policy جديدة اتعملت
- `CPUFREQ_REMOVE_POLICY` — policy بتتمسح

**مثال واقعي:** الـ thermal management module محتاج يعرف لما policy جديدة اتضافت عشان يبدأ يراقب درجة حرارة الـ CPUs دي.

##### 2. Transition Notifiers
بيتبعتوا مرتين في كل تغيير تردد:
- `CPUFREQ_PRECHANGE` — قبل تغيير التردد
- `CPUFREQ_POSTCHANGE` — بعد تغيير التردد

**مثال واقعي:** الـ timing code بيحتاج يعدّل `loops_per_jiffy` (العداد المستخدم في `udelay()`) قبل أو بعد التغيير عشان الـ delays تفضل صح.

---

### الـ OPP — Operating Performance Points

الـ core بيدعم **OPP** وهو نظام بيوصف الأزواج الممكنة (تردد، جهد كهربائي). يعني مش بس السرعة، لكن كمان الـ voltage اللي المعالج محتاجه عند كل سرعة.

الـ `dev_pm_opp_init_cpufreq_table()` بتحول معلومات الـ OPP لجدول ترددات يقدر الـ cpufreq driver يستخدمه مباشرة.

```c
soc_pm_init()
{
    /* translate OPP info into cpufreq table */
    r = dev_pm_opp_init_cpufreq_table(dev, &freq_table);
    if (!r)
        policy->freq_table = freq_table;
}
```

---

### القصة الكاملة: تغيير واحد في السرعة

```
1. الـ governor (مثلاً schedutil) يلاحظ ضغط عالي على الـ CPU
2. بيطلب من الـ core: "عايز 2.4GHz"
3. الـ core يبعت CPUFREQ_PRECHANGE notification لكل subscribers
4. timing code يحفظ القيمة القديمة ويجهز للتغيير
5. الـ core يستدعي driver->target_index()
6. الـ hardware driver يغير الـ PLL أو يكلم الـ firmware
7. الـ core يبعت CPUFREQ_POSTCHANGE notification
8. timing code يحدّث loops_per_jiffy بالقيمة الجديدة
9. الـ stats module يسجل الانتقال
10. sysfs يعكس القيمة الجديدة للـ userspace
```

---

### الملفات المهمة في الـ Subsystem

#### الـ Core

| الملف | الدور |
|-------|-------|
| `drivers/cpufreq/cpufreq.c` | الـ core نفسه — منطق الـ policy، الـ notifiers، الـ reference counting |
| `include/linux/cpufreq.h` | كل الـ structs والـ macros والـ API |
| `include/linux/sched/cpufreq.h` | الـ interface مع الـ scheduler |

#### الـ Governors

| الملف | الدور |
|-------|-------|
| `drivers/cpufreq/cpufreq_ondemand.c` | بيرفع السرعة عند الحاجة |
| `drivers/cpufreq/cpufreq_conservative.c` | أبطأ في الاستجابة |
| `kernel/sched/cpufreq_schedutil.c` | مرتبط بالـ scheduler مباشرة |

#### الـ Hardware Drivers (أمثلة)

| الملف | الدور |
|-------|-------|
| `drivers/cpufreq/intel_pstate.c` | Intel P-state driver |
| `drivers/cpufreq/amd-pstate.c` | AMD P-state driver |
| `drivers/cpufreq/cpufreq-dt.c` | Generic device-tree based driver |
| `drivers/cpufreq/arm_big_little.c` | ARM big.LITTLE |

#### الـ Documentation

| الملف | المحتوى |
|-------|---------|
| `Documentation/cpu-freq/core.rst` | **الملف ده** — الـ core والـ notifiers |
| `Documentation/cpu-freq/cpu-drivers.rst` | كيف تكتب driver جديد |
| `Documentation/cpu-freq/cpufreq-stats.rst` | إحصائيات الـ transitions |
| `Documentation/power/opp.rst` | نظام الـ OPP |
| `Documentation/admin-guide/pm/cpufreq.rst` | للـ sysadmins |

#### الـ Thermal Integration

| الملف | الدور |
|-------|-------|
| `drivers/thermal/cpufreq_cooling.c` | بيستخدم الـ CPUFreq كـ cooling device |
## Phase 2: شرح الـ CPUFreq Framework

### المشكلة — ليه الـ CPUFreq موجود أصلاً؟

الـ CPU مش بيشتغل بسرعة ثابتة في الأنظمة الحديثة. الـ hardware عنده القدرة إنه يشتغل على ترددات متعددة — كل تردد بيتوافق مع **voltage** معين (ده اللي بيتسمى **DVFS: Dynamic Voltage and Frequency Scaling**). الهدف:

- لما الـ CPU مشغول → ارفع التردد عشان تخلص الشغل بسرعة.
- لما الـ CPU فاضي → خفّض التردد (والـ voltage) عشان توفّر طاقة وتقلل الحرارة.

**المشكلة الحقيقية:** كل SoC ليه hardware مختلف (PLL مختلف، voltage regulators مختلفة، OPP tables مختلفة). لو كل جزء في الكيرنل اللي محتاج يعمل frequency scaling اشتغل مباشرة مع الـ hardware، هيبقى في chaos — duplication، race conditions، وكود مش portable خالص.

**الـ CPUFreq** جه يحل ده بإنه يعمل **abstraction layer** موحد يفصل:
- **مين يقرر التردد** (الـ governor).
- **مين يعرف الترددات المتاحة** (الـ driver).
- **مين يحتاج يعرف إن التردد اتغير** (الـ notifiers).

---

### الحل — الكيرنل بيعمل إيه؟

الكيرنل بيقسم المسؤولية لـ 3 طبقات:

| الطبقة | المسؤولية |
|--------|-----------|
| **CPUFreq Core** (`drivers/cpufreq/cpufreq.c`) | التنسيق، الـ policy management، الـ reference counting، الـ sysfs، الإشعارات |
| **CPUFreq Driver** (مثلاً `cpufreq-dt.c`, `intel_pstate.c`) | يعرف الترددات المتاحة، وينفّذ التغيير الفعلي على الـ hardware |
| **CPUFreq Governor** (مثلاً `schedutil`, `ondemand`) | يقرر أي تردد يُستخدم بناءً على حِمل الشغل |

---

### الـ Big Picture — معمارية الـ CPUFreq

```
+------------------------------------------------------+
|                  User Space / sysfs                  |
|   /sys/devices/system/cpu/cpu0/cpufreq/              |
|   scaling_governor, scaling_cur_freq, scaling_max... |
+---------------------------+--------------------------+
                            |
+---------------------------v--------------------------+
|                   CPUFreq Core                       |
|   drivers/cpufreq/cpufreq.c                          |
|                                                      |
|  +----------------+    +------------------------+    |
|  | Policy Manager |    |  Notifier Chain        |    |
|  | cpufreq_policy |    |  - Policy notifiers    |    |
|  +-------+--------+    |  - Transition notifiers|    |
|          |             +------------------------+    |
|  +-------v--------+                                  |
|  | Governor Glue  |  (init/start/stop/limits)        |
|  +-------+--------+                                  |
|          |                                           |
+----------+-------------------------------------------+
           |
+----------v-------------------------------------------+
|              CPUFreq Governors                       |
|  +------------+  +-----------+  +----------------+  |
|  | schedutil  |  |  ondemand |  |  performance   |  |
|  | (scheduler |  | (sampling)|  | (always max)   |  |
|  |  driven)   |  |           |  |                |  |
|  +-----+------+  +-----+-----+  +-------+--------+  |
|        |               |                |            |
+--------+---------------+----------------+-----------+
         |
+--------v-----------------------------------------------+
|              CPUFreq Driver (Platform-specific)        |
|  +-----------------+  +-------------+  +----------+   |
|  | cpufreq-dt      |  | intel_pstate|  | arm_big_ |   |
|  | (Device Tree    |  | (Intel P    |  | little   |   |
|  |  generic driver)|  |  states)    |  |          |   |
|  +--------+--------+  +------+------+  +----+-----+   |
|           |                  |              |          |
+-----------+------------------+--------------+---------+
            |
+-----------v------------------------------------------+
|               Hardware Layer                         |
|  +----------+   +--------------+   +-------------+  |
|  |   PLL /  |   |   Voltage    |   |  OPP Table  |  |
|  |   Clock  |   |  Regulator   |   |  (DT / ACPI)|  |
|  +----------+   +--------------+   +-------------+  |
+------------------------------------------------------+

Consumers (يستقبلون إشعارات):
  - Thermal subsystem (ACPI/thermal cooling)
  - Timing code (loops_per_jiffy)
  - LCD drivers (ARM)
  - Scheduler (schedutil hook)
```

---

### الـ Real-World Analogy — نظام تكييف المبنى

تخيل مبنى كبير فيه نظام تكييف مركزي:

| الـ analogy | المقابل في CPUFreq |
|------------|-------------------|
| **وحدة التحكم المركزية** (BMS) | **CPUFreq Core** — يوزّع الأوامر، ما بيشتغلش مباشرة مع الـ hardware |
| **مستشعرات الحرارة والإشغال** في كل غرفة | **CPUFreq Governor** — يراقب الحِمل ويقرر: "محتاجين تبريد أكتر؟" |
| **مهندس التشغيل** اللي عارف تفاصيل الـ compressor ومحركات الهواء | **CPUFreq Driver** — هو الوحيد اللي يعرف إزاي يضبط الـ hardware الحقيقي |
| **جدول الترددات المتاحة** (الكومبريسور يشتغل على 30%, 60%, 100%) | **`cpufreq_frequency_table`** — الترددات اللي الـ hardware يقدر يشتغل بيها |
| **لافتة تعلن إن التبريد هيقل** عشان ناس تستعد | **CPUFreq Notifiers** — يخبر الـ subsystems التانية قبل وبعد أي تغيير |
| **السياسة** (لا تبرّد أقل من 20°C ولا أكتر من 24°C) | **`cpufreq_policy`** — الـ min/max/governor الخاصة بـ CPU cluster معين |

التعميق: الـ BMS (Core) ما بيلمسش الـ compressor مباشرةً أبداً — هو بيبعت أوامر للمهندس (Driver). المستشعر (Governor) بيقول "غرفة 3 فيها ناس كتير" → الـ BMS يترجمه لأمر للمهندس يرفع الضغط.

---

### الـ Core Abstraction — الـ `cpufreq_policy`

الـ **`cpufreq_policy`** هو القلب التابت للـ framework. كل cluster من الـ CPUs اللي بتشارك نفس الـ clock domain بيكون ليه **policy واحدة**.

```c
struct cpufreq_policy {
    cpumask_var_t   cpus;           /* CPUs online في نفس الـ clock domain */
    cpumask_var_t   related_cpus;   /* كل الـ CPUs (online + offline) */

    unsigned int    cpu;            /* الـ CPU اللي بيدير الـ policy دي */

    struct cpufreq_cpuinfo cpuinfo; /* حدود الـ hardware: min_freq, max_freq, latency */

    unsigned int    min;            /* الحد الأدنى المطلوب (kHz) */
    unsigned int    max;            /* الحد الأقصى المطلوب (kHz) */
    unsigned int    cur;            /* التردد الحالي الفعلي (kHz) */

    struct cpufreq_governor *governor;  /* الـ governor اللي بيتحكم دلوقتي */
    void            *governor_data;     /* بيانات خاصة بالـ governor */

    struct cpufreq_frequency_table *freq_table; /* الترددات المتاحة */

    struct freq_constraints constraints; /* QoS constraints من الـ subsystems */

    bool            fast_switch_possible; /* الـ driver يدعم fast switch؟ */
    bool            fast_switch_enabled;  /* الـ governor فعّله؟ */

    struct rw_semaphore rwsem;      /* حماية الـ policy من race conditions */
    spinlock_t      transition_lock; /* حماية الـ transition state */

    struct kobject  kobj;           /* الـ sysfs entry */
    struct thermal_cooling_device *cdev; /* لو بيُستخدم في thermal cooling */
};
```

**العلاقة بين الـ structs:**

```
cpufreq_policy
     |
     |---> cpufreq_governor   (init/start/stop/limits callbacks)
     |         |
     |         +--> gov_attr_set (sysfs tunables للـ governor)
     |
     |---> cpufreq_driver     (الـ driver الـ hardware-specific)
     |         |
     |         +--> init() / verify() / target_index() / fast_switch() ...
     |
     |---> cpufreq_frequency_table[]
     |         [ {flags, driver_data, frequency_kHz}, ... , {~1u} ]
     |
     |---> freq_constraints    (PM QoS — حدود من الـ thermal أو الـ user)
     |
     +---> cpufreq_stats       (إحصائيات وقت كل تردد)
```

---

### الـ `cpufreq_driver` — الواجهة مع الـ Hardware

```c
struct cpufreq_driver {
    char    name[CPUFREQ_NAME_LEN];
    u16     flags;

    /* إلزامي: تهيئة الـ policy وملء الـ freq_table */
    int     (*init)(struct cpufreq_policy *policy);

    /* إلزامي: التحقق إن الـ min/max معقولة */
    int     (*verify)(struct cpufreq_policy_data *policy);

    /* اختيار واحد بس من التاليين: */

    /* لو الـ driver بيدير كل حاجة بنفسه (مثل intel_pstate) */
    int     (*setpolicy)(struct cpufreq_policy *policy);

    /* أو: الـ core يختار التردد والـ driver ينفّذه بالـ index */
    int     (*target_index)(struct cpufreq_policy *policy,
                            unsigned int index);

    /* Fast path: من scheduler context بدون locks ثقيلة */
    unsigned int (*fast_switch)(struct cpufreq_policy *policy,
                                unsigned int target_freq);

    /* أحدث: يبعت perf hints للـ hardware مباشرةً (CPPC/HW-P states) */
    void    (*adjust_perf)(unsigned int cpu,
                           unsigned long min_perf,
                           unsigned long target_perf,
                           unsigned long capacity);

    /* اختياري: اقرأ التردد الحالي من الـ hardware */
    unsigned int (*get)(unsigned int cpu);

    /* lifecycle */
    int     (*online)(struct cpufreq_policy *policy);
    int     (*offline)(struct cpufreq_policy *policy);
    void    (*exit)(struct cpufreq_policy *policy);
    int     (*suspend)(struct cpufreq_policy *policy);
    int     (*resume)(struct cpufreq_policy *policy);
};
```

**الـ driver flags المهمة:**

| Flag | المعنى |
|------|--------|
| `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` | كل cluster ليه governor منفصل (ARM big.LITTLE) |
| `CPUFREQ_CONST_LOOPS` | التردد ما بيأثرش على `loops_per_jiffy` |
| `CPUFREQ_IS_COOLING_DEV` | سجّل الـ driver كـ thermal cooling device تلقائياً |
| `CPUFREQ_ASYNC_NOTIFICATION` | الـ driver بيبعت POSTCHANGE notification بنفسه |
| `CPUFREQ_NEED_INITIAL_FREQ_CHECK` | تحقق إن التردد الحالي في الـ freq_table |

---

### الـ `cpufreq_governor` — من يقرر التردد؟

```c
struct cpufreq_governor {
    char    name[CPUFREQ_NAME_LEN];

    int     (*init)(struct cpufreq_policy *policy);  /* خصص resources */
    void    (*exit)(struct cpufreq_policy *policy);  /* حرّر resources */
    int     (*start)(struct cpufreq_policy *policy); /* ابدأ القرارات */
    void    (*stop)(struct cpufreq_policy *policy);  /* وقّف القرارات */
    void    (*limits)(struct cpufreq_policy *policy);/* الـ min/max اتغيرت */

    u8      flags;  /* CPUFREQ_GOV_DYNAMIC_SWITCHING | CPUFREQ_GOV_STRICT_TARGET */
};
```

الـ governors الموجودة في الكيرنل:

| Governor | المنطق |
|----------|--------|
| `performance` | دايماً أقصى تردد |
| `powersave` | دايماً أدنى تردد |
| `schedutil` | بيستخدم إشارات الـ scheduler (PELT utilization) مباشرةً عبر `update_util_data` hook |
| `ondemand` | بيعمل sampling دوري للـ CPU idle time |
| `conservative` | زي ondemand بس بيرفع/يخفض تدريجياً |

**ملحوظة عن الـ `schedutil`:** هو بيعتمد على الـ **scheduler subsystem** — تحديداً الـ PELT (Per-Entity Load Tracking). الـ scheduler بيحسب `util_avg` لكل CPU، والـ `schedutil` بيترجمه لتردد مناسب عبر `update_util_data` hook المعرّف في `include/linux/sched/cpufreq.h`:

```c
struct update_util_data {
    void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
};
```

---

### الـ CPUFreq Notifiers — نظام الإشعارات

**مفهوم الـ Notifier Chain:** قبل ما تكمل، الـ notifier chain في الكيرنل هي linked list من الـ `notifier_block` structs. أي كود يقدر يسجّل نفسه فيها، ولما حدث معين يصير، كل المسجّلين بيتبلّغوا بالترتيب.

في CPUFreq في نوعين:

#### 1. Policy Notifiers

بيُبعتوا لما policy تتأسس أو تُحذف:

```
CPUFREQ_CREATE_POLICY  →  policy اتعملت (min, max, governor اتحدد)
CPUFREQ_REMOVE_POLICY  →  policy هتتحذف (cleanup)
```

**من يستخدمها؟** أي driver أو subsystem محتاج يعرف لما CPU cluster يصبح متاح أو offline.

#### 2. Transition Notifiers

بيُبعتوا **مرتين** لكل تغيير في التردد:

```
قبل التغيير:  CPUFREQ_PRECHANGE  (old_freq → new_freq)
              ← المستقبلين يجهّزوا نفسهم (مثلاً: timing code)
              ← الـ driver ينفّذ التغيير الفعلي في الـ hardware

بعد التغيير: CPUFREQ_POSTCHANGE (old_freq → new_freq)
              ← تحديث loops_per_jiffy
              ← إشعار thermal module
```

الـ struct اللي بيتبعت:

```c
struct cpufreq_freqs {
    struct cpufreq_policy *policy;  /* الـ policy المتأثرة */
    unsigned int old;               /* التردد القديم (kHz) */
    unsigned int new;               /* التردد الجديد (kHz) */
    u8 flags;                       /* flags من الـ driver */
};
```

**تسلسل الأحداث لتغيير تردد:**

```
Governor يقرر تردد جديد
         |
         v
cpufreq_driver_target()
         |
         v
cpufreq_freq_transition_begin()
    → يبعت CPUFREQ_PRECHANGE للـ notifiers
    → يضبط transition_ongoing = true
         |
         v
driver->target_index()   ← التغيير الفعلي في الـ hardware (PLL/regulator)
         |
         v
cpufreq_freq_transition_end()
    → يبعت CPUFREQ_POSTCHANGE للـ notifiers
    → يحدّث loops_per_jiffy
    → يضبط transition_ongoing = false
    → يصحّى الـ waiters على transition_wait
```

---

### الـ Frequency Table والـ OPP

الـ **`cpufreq_frequency_table`** هي array بسيطة من الترددات المتاحة:

```c
struct cpufreq_frequency_table {
    unsigned int    flags;        /* CPUFREQ_BOOST_FREQ | CPUFREQ_INEFFICIENT_FREQ */
    unsigned int    driver_data;  /* بيانات خاصة بالـ driver (الـ index في OPP table مثلاً) */
    unsigned int    frequency;    /* التردد بالـ kHz */
    // آخر entry: frequency = CPUFREQ_TABLE_END (~1u)
};
```

الـ **OPP (Operating Performance Point):** هو مفهوم منفصل (subsystem: `drivers/opp/`) يربط كل تردد بالـ voltage المناسب. الـ CPUFreq بيتكامل معه عبر:

```c
// يحوّل الـ OPP table لـ cpufreq_frequency_table جاهزة:
dev_pm_opp_init_cpufreq_table(dev, &policy->freq_table);
```

**مثال عملي (RK3399 - ARM Cortex-A72):**

```c
// الـ DT يعرّف الـ OPP table
// الـ OPP subsystem يقرأها
// الـ cpufreq-dt driver يحوّلها:
static int cpufreq_init(struct cpufreq_policy *policy)
{
    ret = dev_pm_opp_init_cpufreq_table(cpu_dev, &freq_table);
    policy->freq_table = freq_table;
    // النتيجة: [ {0, 0, 408000}, {0, 0, 600000}, ..., {0, 0, 1800000}, {0, 0, ~1u} ]
}
```

---

### الـ Fast Switch Path — التحديث السريع من الـ Scheduler

في الأنظمة الحديثة، الـ `schedutil` governor بيحتاج يغيّر التردد من سياق الـ scheduler مباشرةً (hotpath) من غير ما يدخل في locking ثقيل. الحل:

```
Normal Path:  Governor → cpufreq_driver_target() → locks → notifiers → driver->target_index()
Fast Path:    Governor → cpufreq_driver_fast_switch() → driver->fast_switch() مباشرةً
```

الـ `fast_switch` بيشتغل من **interrupt context أو atomic context**، وبالتالي:
- ما فيش notifiers.
- ما فيش `transition_lock`.
- الـ driver لازم يكون thread-safe على كل CPU في الـ policy.

الشرط: `policy->fast_switch_possible = true` (الـ driver يعلن إنه يدعمه) **و** الـ governor يفعّله بـ `cpufreq_enable_fast_switch()`.

---

### مين يملك إيه — الـ Core vs الـ Driver

| المهمة | المالك |
|--------|--------|
| إدارة دورة حياة الـ policy (إنشاء/حذف) | **Core** |
| الـ sysfs interface | **Core** |
| اختيار الـ governor وتشغيله | **Core** |
| إرسال الـ notifiers | **Core** |
| تحديث `loops_per_jiffy` | **Core** |
| Reference counting للـ policy | **Core** (`cpufreq_cpu_get` / `cpufreq_cpu_put`) |
| معرفة الترددات المتاحة (freq_table) | **Driver** |
| تنفيذ تغيير التردد على الـ hardware | **Driver** |
| التعامل مع الـ voltage regulator | **Driver** |
| تسجيل الـ cooling device في الـ thermal | **Core** (لو `CPUFREQ_IS_COOLING_DEV` set) |
| قرار أي تردد يُستخدم | **Governor** |
| حساب الـ utilization | **Scheduler** (PELT) |

---

### الـ Reference Counting والـ Thread Safety

الـ **`cpufreq_cpu_get(cpu)`** بيجيب الـ policy الخاصة بـ CPU معين ويزوّد الـ reference count للـ kobject. ده بيضمن إن الـ policy ما تتحذفش وفيه حد بيستخدمها (مثلاً: driver يتعمل unload أثناء policy change).

```c
// استخدام صح:
struct cpufreq_policy *policy = cpufreq_cpu_get(cpu);
if (!policy)
    return -ENODEV;

// استخدم الـ policy...

cpufreq_cpu_put(policy);  // لازم تتعمل دايماً
```

الـ `rwsem` في الـ policy:
- **read lock:** أي كود يقرأ من الـ policy.
- **write lock:** تغيير الـ governor، الـ CPU hotplug، تغيير الـ min/max.

الـ `transition_lock` (spinlock) + `transition_wait` (waitqueue): بيضمنوا إن transition واحد بس بيحصل في وقت واحد.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### Flags الخاصة بالـ `cpufreq_driver` (الحقل `flags` من نوع `u16`)

| Flag | القيمة | المعنى |
|------|--------|--------|
| `CPUFREQ_NEED_UPDATE_LIMITS` | `BIT(0)` | الـ driver يحتاج استدعاء حتى لو ما تغيّرت الـ target freq، لأن الـ min/max ممكن تتغيّر |
| `CPUFREQ_CONST_LOOPS` | `BIT(1)` | الـ `loops_per_jiffy` مش بتتأثر بتغيير الـ frequency |
| `CPUFREQ_IS_COOLING_DEV` | `BIT(2)` | الـ core يسجّل الـ driver تلقائياً كـ thermal cooling device |
| `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` | `BIT(3)` | منصات متعددة الـ clock-domain — governor لكل policy بشكل مستقل |
| `CPUFREQ_ASYNC_NOTIFICATION` | `BIT(4)` | الـ driver بيبعت POSTCHANGE من بره الـ `target()` |
| `CPUFREQ_NEED_INITIAL_FREQ_CHECK` | `BIT(5)` | الـ core يتحقق إن الـ CPU شغّال على freq موجودة في الجدول |
| `CPUFREQ_NO_AUTO_DYNAMIC_SWITCHING` | `BIT(6)` | يمنع استخدام governors بتعمل dynamic switching |

#### Flags الخاصة بالـ `cpufreq_governor` (الحقل `flags` من نوع `u8`)

| Flag | القيمة | المعنى |
|------|--------|--------|
| `CPUFREQ_GOV_DYNAMIC_SWITCHING` | `BIT(0)` | الـ governor بيغيّر الـ frequency ديناميكياً بنفسه (زي schedutil) |
| `CPUFREQ_GOV_STRICT_TARGET` | `BIT(1)` | لازم الـ driver يضبط الـ freq بالضبط على الـ target |

#### Flags الخاصة بالـ `cpufreq_frequency_table` (الحقل `flags`)

| Flag | القيمة | المعنى |
|------|--------|--------||
| `CPUFREQ_BOOST_FREQ` | `(1 << 0)` | الـ frequency دي boost — مش مفعّلة بشكل افتراضي |
| `CPUFREQ_INEFFICIENT_FREQ` | `(1 << 1)` | الـ frequency دي مش كفء — ممكن تتخطّى لو الـ relation = E |

#### قيم خاصة في الـ `cpufreq_frequency_table`

| الثابت | القيمة | المعنى |
|--------|--------|--------|
| `CPUFREQ_ENTRY_INVALID` | `~0u` | الـ entry مش صالح |
| `CPUFREQ_TABLE_END` | `~1u` | نهاية الجدول |

#### الـ Relation Flags (بتحدد ازاي تختار الـ freq من الجدول)

| الثابت | المعنى |
|--------|--------|
| `CPUFREQ_RELATION_L` | أدنى freq أكبر من أو تساوي الـ target |
| `CPUFREQ_RELATION_H` | أعلى freq أقل من أو تساوي الـ target |
| `CPUFREQ_RELATION_C` | أقرب freq للـ target |
| `CPUFREQ_RELATION_E` | `BIT(2)` — يُفضّل الـ freq الكفء (efficient) لو متاح |
| `CPUFREQ_RELATION_LE/HE/CE` | تركيبة من L/H/C مع E |

#### الـ Notifier Events

| النوع | الثابت | القيمة | المعنى |
|-------|--------|--------|--------|
| Policy | `CPUFREQ_CREATE_POLICY` | 0 | تم إنشاء policy جديدة |
| Policy | `CPUFREQ_REMOVE_POLICY` | 1 | تم حذف الـ policy |
| Transition | `CPUFREQ_PRECHANGE` | 0 | قبل تغيير الـ freq |
| Transition | `CPUFREQ_POSTCHANGE` | 1 | بعد تغيير الـ freq |

#### الـ Config Options المهمة

| Config | المعنى |
|--------|--------|
| `CONFIG_CPU_FREQ` | يفعّل الـ CPUFreq core كله |
| `CONFIG_CPU_FREQ_STAT` | إحصائيات وقت الإقامة لكل freq |
| `CONFIG_CPU_FREQ_GOV_SCHEDUTIL` | الـ schedutil governor |
| `CONFIG_CPU_THERMAL` | دعم الـ thermal cooling |
| `CONFIG_PM_OPP` | دعم الـ OPP لإنشاء جداول الـ freq |

#### الـ Shared Type (ACPI فقط)

| الثابت | القيمة | المعنى |
|--------|--------|--------|
| `CPUFREQ_SHARED_TYPE_NONE` | 0 | مفيش تنسيق مطلوب |
| `CPUFREQ_SHARED_TYPE_HW` | 1 | الـ HW بيعمل التنسيق بنفسه |
| `CPUFREQ_SHARED_TYPE_ALL` | 2 | كل الـ CPUs لازم تضبط الـ freq |
| `CPUFREQ_SHARED_TYPE_ANY` | 3 | أي CPU يقدر يضبط الـ freq |

---

### الـ Structs المهمة

#### 1. `struct cpufreq_policy`

**الغرض:** ده العقد الأساسي في نظام CPUFreq — بيمثّل سياسة تحكم في الـ frequency لمجموعة CPUs بتشارك نفس الـ clock domain.

```c
struct cpufreq_policy {
    cpumask_var_t   cpus;          /* CPUs أونلاين بس */
    cpumask_var_t   related_cpus;  /* أونلاين + أوف لاين */
    cpumask_var_t   real_cpus;     /* موجودة فعلاً في النظام */

    unsigned int    cpu;           /* الـ CPU المسؤول عن الـ policy */
    struct clk     *clk;           /* الـ clock المرتبط */
    struct cpufreq_cpuinfo cpuinfo;/* min/max/latency من الـ hardware */

    unsigned int    min;           /* أقل freq مسموح (kHz) */
    unsigned int    max;           /* أعلى freq مسموح (kHz) */
    unsigned int    cur;           /* الـ freq الحالية (kHz) */
    unsigned int    suspend_freq;  /* الـ freq وقت الـ suspend */

    struct cpufreq_governor *governor; /* الـ governor الحالي */
    void           *governor_data;    /* بيانات خاصة بالـ governor */

    struct freq_constraints  constraints; /* QoS constraints */
    struct freq_qos_request *min_freq_req;
    struct freq_qos_request *max_freq_req;

    struct cpufreq_frequency_table *freq_table; /* جدول الـ frequencies */
    enum cpufreq_table_sorting freq_table_sorted;

    struct list_head policy_list;  /* لقائمة كل الـ policies */
    struct kobject   kobj;         /* sysfs entry */

    struct rw_semaphore rwsem;     /* الـ lock الرئيسي للـ policy */

    bool fast_switch_possible;     /* الـ driver يدعم fast switch */
    bool fast_switch_enabled;      /* الـ governor فعّله */
    bool strict_target;            /* يضبط الـ freq بالضبط */
    bool efficiencies_available;   /* فيه freqs كفء في الجدول */
    bool dvfs_possible_from_any_cpu;

    spinlock_t      transition_lock;    /* يحمي الـ transition state */
    wait_queue_head_t transition_wait;
    bool            transition_ongoing;

    struct cpufreq_stats *stats;    /* إحصائيات الـ freq */
    void           *driver_data;   /* بيانات خاصة بالـ driver */
    struct thermal_cooling_device *cdev; /* للـ thermal mitigation */
    struct notifier_block nb_min, nb_max; /* QoS notifiers */
};
```

**الحقول الجوهرية:**

| الحقل | الوظيفة |
|-------|---------|
| `cpus` | الـ CPUs الأونلاين المشتركة في الـ policy |
| `related_cpus` | كل الـ CPUs المرتبطة (أونلاين وأوف لاين) |
| `min / max` | حدود الـ freq المقبولة حالياً |
| `cur` | الـ freq الفعلية الحالية |
| `governor` | بوينتر للـ governor المسؤول عن قرار التردد |
| `freq_table` | جدول الـ frequencies المدعومة من الـ hardware |
| `rwsem` | الـ lock الرئيسي — read لمن يقرأ، write لمن يعدّل أو يحذف |
| `transition_lock` | spinlock يحمي عملية التغيير الجارية |
| `fast_switch_possible` | يحدد لو الـ governor يقدر يتجاوز الـ notifiers |

---

#### 2. `struct cpufreq_driver`

**الغرض:** بيمثّل الـ driver الخاص بالمنصة — هو اللي يعرف كيف يتكلم مع الـ hardware فعلياً.

```c
struct cpufreq_driver {
    char    name[CPUFREQ_NAME_LEN]; /* اسم الـ driver */
    u16     flags;                  /* سلوكيات خاصة */
    void   *driver_data;            /* بيانات داخلية */

    /* إلزامي */
    int     (*init)(struct cpufreq_policy *policy);
    int     (*verify)(struct cpufreq_policy_data *policy);

    /* اختار واحدة من الاتنين */
    int     (*setpolicy)(struct cpufreq_policy *policy);   /* للـ hardware-managed */
    int     (*target_index)(struct cpufreq_policy *policy, unsigned int index);

    /* fast path — بدون notifiers */
    unsigned int (*fast_switch)(struct cpufreq_policy *policy, unsigned int target_freq);
    void         (*adjust_perf)(unsigned int cpu, unsigned long min_perf,
                                unsigned long target_perf, unsigned long capacity);

    /* لتغيير freq بمرحلتين */
    unsigned int (*get_intermediate)(struct cpufreq_policy *policy, unsigned int index);
    int          (*target_intermediate)(struct cpufreq_policy *policy, unsigned int index);

    unsigned int (*get)(unsigned int cpu);        /* يقرأ الـ freq الحالية */
    void         (*update_limits)(struct cpufreq_policy *policy);

    /* دورة حياة الـ CPU */
    int  (*online)(struct cpufreq_policy *policy);
    int  (*offline)(struct cpufreq_policy *policy);
    void (*exit)(struct cpufreq_policy *policy);
    int  (*suspend)(struct cpufreq_policy *policy);
    int  (*resume)(struct cpufreq_policy *policy);
    void (*ready)(struct cpufreq_policy *policy);  /* بعد اكتمال التهيئة */

    struct freq_attr **attr;  /* sysfs attributes */
    bool boost_enabled;
    int  (*set_boost)(struct cpufreq_policy *policy, int state);
    void (*register_em)(struct cpufreq_policy *policy); /* Energy Model */
};
```

**الحقول الجوهرية:**

| الحقل | الوظيفة |
|-------|---------|
| `init` | يُهيئ الـ policy — يملأ الـ freq_table والـ cpuinfo |
| `verify` | يتحقق ويصحح حدود الـ freq قبل التطبيق |
| `target_index` | يضبط الـ freq بناءً على index في الجدول (الطريقة الحديثة) |
| `fast_switch` | يغير الـ freq بدون notifiers من scheduler hotpath |
| `adjust_perf` | بديل fast_switch بمعلومات أكتر عن الأداء |
| `get_intermediate` | يُرجع freq وسيطة آمنة قبل القفز لـ target |

---

#### 3. `struct cpufreq_governor`

**الغرض:** يمثّل الـ policy لاتخاذ قرار أي freq تُختار — هو الوسيط بين متطلبات النظام وقدرات الـ driver.

```c
struct cpufreq_governor {
    char   name[CPUFREQ_NAME_LEN];
    int    (*init)(struct cpufreq_policy *policy);   /* تهيئة موارد الـ governor */
    void   (*exit)(struct cpufreq_policy *policy);   /* تحرير الموارد */
    int    (*start)(struct cpufreq_policy *policy);  /* ابدأ ضبط الـ freq */
    void   (*stop)(struct cpufreq_policy *policy);   /* وقّف الضبط */
    void   (*limits)(struct cpufreq_policy *policy); /* الحدود تغيّرت */
    ssize_t (*show_setspeed)(struct cpufreq_policy *policy, char *buf);
    int    (*store_setspeed)(struct cpufreq_policy *policy, unsigned int freq);
    struct list_head governor_list;  /* في قائمة الـ governors المسجّلة */
    struct module   *owner;
    u8               flags;
};
```

---

#### 4. `struct cpufreq_freqs`

**الغرض:** حزمة المعلومات اللي بتتمرر للـ notifiers وقت تغيير الـ freq.

```c
struct cpufreq_freqs {
    struct cpufreq_policy *policy; /* الـ policy اللي بتتغير */
    unsigned int old;              /* الـ freq القديمة (kHz) */
    unsigned int new;              /* الـ freq الجديدة (kHz) */
    u8 flags;                      /* flags من الـ driver */
};
```

---

#### 5. `struct cpufreq_cpuinfo`

**الغرض:** معلومات الـ hardware الثابتة — الحدود الفيزيائية للـ CPU.

```c
struct cpufreq_cpuinfo {
    unsigned int max_freq;          /* أعلى freq يدعمها الـ hardware (kHz) */
    unsigned int min_freq;          /* أقل freq يدعمها الـ hardware (kHz) */
    unsigned int transition_latency; /* وقت التحول بالنانوثانية */
};
```

---

#### 6. `struct cpufreq_frequency_table`

**الغرض:** جدول كل الـ frequencies المدعومة من الـ hardware — array تنتهي بـ `CPUFREQ_TABLE_END`.

```c
struct cpufreq_frequency_table {
    unsigned int flags;       /* BOOST أو INEFFICIENT */
    unsigned int driver_data; /* بيانات خاصة بالـ driver */
    unsigned int frequency;   /* التردد (kHz) */
};
```

---

#### 7. `struct cpufreq_policy_data`

**الغرض:** نسخة مبسّطة من الـ policy — بتُمرَّر للـ `verify()` callback فقط، ومش بيُسمح للـ driver يعدّل فيها غير الـ min/max.

```c
struct cpufreq_policy_data {
    struct cpufreq_cpuinfo         cpuinfo;
    struct cpufreq_frequency_table *freq_table;
    unsigned int cpu;
    unsigned int min; /* in kHz */
    unsigned int max; /* in kHz */
};
```

---

#### 8. `struct gov_attr_set`

**الغرض:** مجموعة الـ sysfs attributes للـ governor — مشتركة بين كل الـ policies اللي بتستخدم نفس الـ governor.

```c
struct gov_attr_set {
    struct kobject   kobj;
    struct list_head policy_list; /* الـ policies بتاعته */
    struct mutex     update_lock;
    int              usage_count;
};
```

---

#### 9. `struct freq_attr` و `struct governor_attr`

**الغرض:** بيعرّفوا الـ sysfs attributes — كل attribute عنده show و store.

```c
struct freq_attr {          /* attribute خاص بالـ policy */
    struct attribute attr;
    ssize_t (*show)(struct cpufreq_policy *, char *);
    ssize_t (*store)(struct cpufreq_policy *, const char *, size_t count);
};

struct governor_attr {      /* attribute خاص بالـ governor */
    struct attribute attr;
    ssize_t (*show)(struct gov_attr_set *attr_set, char *buf);
    ssize_t (*store)(struct gov_attr_set *attr_set, const char *buf, size_t count);
};
```

---

### رسم علاقات الـ Structs

```
                    ┌─────────────────────────────────────┐
                    │        cpufreq_driver               │
                    │  name, flags, init, verify,         │
                    │  target_index, fast_switch, ...     │
                    └──────────────┬──────────────────────┘
                                   │  يسجّل نفسه في الـ core
                                   │  cpufreq_register_driver()
                                   ▼
         ┌──────────────────────────────────────────────────────┐
         │                 cpufreq_policy                       │
         │  cpu, min, max, cur, suspend_freq                    │
         │  ┌───────────────┐    ┌──────────────────────────┐  │
         │  │ cpufreq_cpuinfo│    │ cpufreq_frequency_table[]│  │
         │  │ min/max/latency│    │ freq, flags, driver_data │  │
         │  └───────────────┘    └──────────────────────────┘  │
         │  ┌─────────────────┐   ┌───────────────────────────┐ │
         │  │ cpumask (cpus)  │   │ freq_constraints + QoS    │ │
         │  │ related_cpus    │   │ min_freq_req, max_freq_req│ │
         │  │ real_cpus       │   └───────────────────────────┘ │
         │  └─────────────────┘                                 │
         │  ┌─────────────────────────────┐                     │
         │  │ cpufreq_governor  *governor │                     │
         │  │  init, exit, start, stop    │                     │
         │  │  limits, flags              │                     │
         │  └─────────────────────────────┘                     │
         │  struct kobject kobj  ──► /sys/devices/system/cpu/   │
         │  struct rw_semaphore rwsem                           │
         │  spinlock_t transition_lock                          │
         │  struct cpufreq_stats *stats                         │
         │  struct thermal_cooling_device *cdev                 │
         └──────────────────────────────────────────────────────┘
                    │                         │
                    │                         │
                    ▼                         ▼
         ┌─────────────────┐        ┌──────────────────────┐
         │  cpufreq_freqs  │        │    gov_attr_set       │
         │  policy*        │        │  kobj, policy_list,  │
         │  old, new, flags│        │  update_lock         │
         └─────────────────┘        └──────────────────────┘
              (notifiers)                (sysfs governor)
```

---

### رسم دورة حياة الـ Policy

```
  [CPU Hotplug / Boot]
         │
         ▼
  cpufreq_add_dev()
         │
         ├─► تخصيص struct cpufreq_policy
         │
         ├─► driver->init(policy)
         │     ├── يملأ policy->cpuinfo (min/max/latency)
         │     ├── يبني freq_table
         │     └── يضبط policy->min/max/cur
         │
         ├─► cpufreq_freq_transition_begin/end()
         │     (تهيئة الـ freq الأولية)
         │
         ├─► تسجيل الـ kobject في sysfs
         │     /sys/devices/system/cpu/cpufreq/policy<N>/
         │
         ├─► cpufreq_start_governor(policy)
         │     ├── governor->init(policy)
         │     └── governor->start(policy)
         │
         ├─► إشعار الـ notifiers: CPUFREQ_CREATE_POLICY
         │
         │   [التشغيل الطبيعي]
         │     governor يراقب الـ load
         │     └── cpufreq_driver_target() / fast_switch()
         │           ├── CPUFREQ_PRECHANGE notifiers
         │           ├── driver->target_index(policy, idx)
         │           └── CPUFREQ_POSTCHANGE notifiers
         │
         ▼   [CPU Hotplug Out / Shutdown]
  cpufreq_remove_dev()
         │
         ├─► إشعار الـ notifiers: CPUFREQ_REMOVE_POLICY
         │
         ├─► cpufreq_stop_governor(policy)
         │     ├── governor->stop(policy)
         │     └── governor->exit(policy)
         │
         ├─► driver->offline(policy)  [إن وُجد]
         │
         ├─► driver->exit(policy)
         │
         └─► تحرير struct cpufreq_policy
```

---

### رسم دورة حياة الـ Driver

```
  [Module Load / Boot]
         │
         ▼
  cpufreq_register_driver(driver)
         │
         ├── يتحقق مش فيه driver مسجّل
         ├── يضيف الـ driver في المتغير العالمي cpufreq_driver
         ├── يستدعي subsys_interface_register()
         │     └── لكل CPU موجود: cpufreq_add_dev()
         │           └── ينشئ policy جديدة
         │
         [التشغيل]
         │
         ▼
  cpufreq_unregister_driver(driver)
         │
         ├── يُزيل كل الـ policies (cpufreq_remove_dev لكل CPU)
         ├── يُصفّر cpufreq_driver
         └── Module Unload آمن
```

---

### رسم دورة حياة الـ Governor

```
  [Module Load]
         │
         ▼
  cpufreq_register_governor(governor)
         │
         └── يُضيف الـ governor في قائمة cpufreq_governor_list

  [عند تغيير الـ governor عبر sysfs]
         │
         ▼
  cpufreq_set_policy()
         │
         ├─► cpufreq_stop_governor(policy)
         │     ├── old_gov->stop(policy)
         │     └── old_gov->exit(policy)
         │
         ├─► policy->governor = new_gov
         │
         └─► cpufreq_start_governor(policy)
               ├── new_gov->init(policy)
               └── new_gov->start(policy)

  [عند تغيير الـ limits]
         │
         ▼
  governor->limits(policy)   ← الـ governor يُعيد ضبط الـ freq
```

---

### Call Flow Diagrams

#### تغيير الـ Frequency (المسار الاعتيادي)

```
governor decides new freq needed
  │
  ▼
cpufreq_driver_target(policy, target_freq, relation)
  │
  ├── down_read(policy->rwsem)
  │
  ├── cpufreq_driver_resolve_freq(policy, target_freq)
  │     └── يبحث في freq_table حسب relation (L/H/C/E)
  │
  ├── cpufreq_freq_transition_begin(policy, &freqs)
  │     ├── spinlock: transition_ongoing = true
  │     └── srcu_notifier_call_chain(CPUFREQ_PRECHANGE)
  │           └── كل notifier مسجّل يستقبل الحدث
  │
  ├── cpufreq_driver->target_index(policy, index)
  │     └── الـ driver يكتب في registers الـ hardware
  │           مثلاً: writel(freq, clk_reg)
  │
  ├── cpufreq_freq_transition_end(policy, &freqs, failed)
  │     ├── تحديث policy->cur
  │     ├── srcu_notifier_call_chain(CPUFREQ_POSTCHANGE)
  │     │     └── مثلاً: loops_per_jiffy يتحدث
  │     └── spinlock: transition_ongoing = false
  │           wake_up(transition_wait)
  │
  └── up_read(policy->rwsem)
```

#### المسار السريع (Fast Switch) — من الـ Scheduler

```
schedutil governor (في scheduler hotpath)
  │
  ▼
cpufreq_driver_fast_switch(policy, target_freq)
  │
  ├── مفيش rwsem ولا notifiers
  │
  ├── cpufreq_driver->fast_switch(policy, target_freq)
  │     └── الـ driver يكتب مباشرة في الـ hardware
  │
  └── policy->cur = new_freq  (بشكل atomic)
```

#### الـ Notifier Registration وإرسال الأحداث

```
driver/subsystem يريد الاشتراك في أحداث الـ freq
  │
  ▼
cpufreq_register_notifier(nb, CPUFREQ_TRANSITION_NOTIFIER)
  │
  └── يُضاف nb في cpufreq_transition_notifier_list

عند حدوث تغيير في الـ freq:
  cpufreq_freq_transition_begin()
    └── srcu_notifier_call_chain(cpufreq_transition_notifier_list,
                                 CPUFREQ_PRECHANGE, &freqs)
          └── nb->notifier_call(nb, CPUFREQ_PRECHANGE, &freqs)
                مثلاً: thermal module يُعدّل limits

  cpufreq_freq_transition_end()
    └── srcu_notifier_call_chain(cpufreq_transition_notifier_list,
                                 CPUFREQ_POSTCHANGE, &freqs)
          └── nb->notifier_call(nb, CPUFREQ_POSTCHANGE, &freqs)
                مثلاً: تحديث loops_per_jiffy
```

#### بناء جدول الـ Freq من OPP

```
المنصة عندها OPP table في device tree
  │
  ▼
dev_pm_opp_init_cpufreq_table(dev, &freq_table)
  │
  ├── يقرأ OPP entries من الـ framework
  ├── يبني struct cpufreq_frequency_table[]
  └── يُرجع البوينتر

driver->init(policy):
  policy->freq_table = freq_table;

عند الانتهاء:
dev_pm_opp_free_cpufreq_table(dev, &freq_table)
```

---

### استراتيجية الـ Locking

#### الـ `rwsem` في `cpufreq_policy`

ده الـ lock الأساسي لحماية بنية الـ policy.

| من يمسك | النوع | متى |
|---------|-------|-----|
| governor, driver (قراءة فقط) | `down_read` | عند قراءة min/max/cur/freq_table |
| cpufreq core (تعديل الـ policy) | `down_write` | عند تغيير governor، تعديل min/max، hotplug |
| CPU hotplug (حذف الـ policy) | `down_write` | عند إزالة CPU من النظام |

```c
/* Guards مُعرّفة في الـ header للاستخدام الآمن */
DEFINE_GUARD(cpufreq_policy_write, struct cpufreq_policy *,
             down_write(&_T->rwsem), up_write(&_T->rwsem))

DEFINE_GUARD(cpufreq_policy_read, struct cpufreq_policy *,
             down_read(&_T->rwsem), up_read(&_T->rwsem))
```

#### الـ `transition_lock` (spinlock)

بيحمي حالة الـ transition الجارية — لأن الـ fast_switch بيحصل من scheduler hotpath ومش ممكن يستخدم mutex.

| ما يحميه | طبيعة الـ lock |
|----------|----------------|
| `transition_ongoing` | spinlock |
| `transition_task` | spinlock |
| إشارة `transition_wait` | spinlock قبل wake_up |

#### الـ `gov_attr_set->update_lock` (mutex)

بيحمي قائمة الـ `policy_list` في الـ governor — لما فيه أكتر من policy بتستخدم نفس الـ governor.

#### ترتيب الـ Locks (Lock Ordering)

```
cpus_read_lock()              ← CPU hotplug lock (الأعلى)
  └── policy->rwsem (write)   ← تعديل الـ policy
        └── policy->transition_lock (spin) ← أثناء التحول
              └── gov_attr_set->update_lock ← governor tunables
```

> **قاعدة صارمة:** الـ `transition_lock` spinlock لا يُمسك أبداً وأنت ماسك الـ `rwsem` write بشكل قد يسبب deadlock مع الـ scheduler — الـ fast_switch path مُصمّم بشكل يتجنب هذا تماماً.

#### الـ Reference Counting

الـ policy بتتحمى من الحذف أثناء الاستخدام عبر:

```c
/* يزيد refcount على الـ kobject */
struct cpufreq_policy *cpufreq_cpu_get(unsigned int cpu);

/* ينقص الـ refcount — لو وصل صفر، الـ policy تتحرر */
void cpufreq_cpu_put(struct cpufreq_policy *policy);

/* Scope-based cleanup (C cleanup attribute) */
DEFINE_FREE(put_cpufreq_policy, struct cpufreq_policy *,
            if (_T) cpufreq_cpu_put(_T))
```

هذا يضمن إن الـ driver مش هيُفرغ من الذاكرة وفيه كود لسه بيستخدمه.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs (Cheatsheet)

#### Policy Reference Counting

| Function | الغرض |
|---|---|
| `cpufreq_cpu_get(cpu)` | احصل على reference لـ policy الـ CPU وزود الـ refcount |
| `cpufreq_cpu_put(policy)` | حرر الـ reference وقلل الـ refcount |

#### Policy Notifiers

| Function / Macro | الغرض |
|---|---|
| `cpufreq_register_notifier(nb, list)` | سجّل notifier في قائمة policy أو transition |
| `cpufreq_unregister_notifier(nb, list)` | أزل notifier من القائمة |
| `CPUFREQ_POLICY_NOTIFIER` | الـ list ID لـ policy notifiers |
| `CPUFREQ_TRANSITION_NOTIFIER` | الـ list ID لـ transition notifiers |

#### OPP Table Helpers

| Function | الغرض |
|---|---|
| `dev_pm_opp_init_cpufreq_table(dev, table)` | حوّل OPP entries إلى `cpufreq_frequency_table` |
| `dev_pm_opp_free_cpufreq_table(dev, table)` | حرر الـ table المخصصة من `init_cpufreq_table` |

#### Notification Phases & Events

| Constant | السياق | المعنى |
|---|---|---|
| `CPUFREQ_CREATE_POLICY` | Policy Notifier | policy اتأسست لأول مرة |
| `CPUFREQ_REMOVE_POLICY` | Policy Notifier | policy بتتحذف |
| `CPUFREQ_PRECHANGE` | Transition Notifier | قبل تغيير الـ frequency |
| `CPUFREQ_POSTCHANGE` | Transition Notifier | بعد اكتمال تغيير الـ frequency |

---

### Group 1: Reference Counting — Policy Lifecycle Management

الـ CPUFreq core بيحافظ على reference counting لكل `cpufreq_policy` عشان يضمن إن الـ driver مش هيتـ unload وإن الـ policy مش هتتحرر من الـ memory وفيه كود شغال عليها.

---

#### `cpufreq_cpu_get`

```c
struct cpufreq_policy *cpufreq_cpu_get(unsigned int cpu);
```

**الـ function دي بتعمل إيه:**
بتجيب الـ `cpufreq_policy` المرتبطة بـ `cpu` المحدد، وبتزود الـ reference count عليها. ده بيضمن إن الـ policy مش هتتحرر طول ما الكود شغال عليها، وكمان بيضمن إن الـ cpufreq driver موجود ومرجّعه صح.

**Parameters:**
- `cpu` — رقم الـ logical CPU (من 0 لـ `nr_cpu_ids - 1`)

**Return value:**
- بترجع pointer لـ `struct cpufreq_policy` لو نجحت
- بترجع `NULL` لو الـ CPU مش موجود أو مفيش policy مسجلة أو الـ driver مش موجود

**Key details:**
- بتعمل `get_online_cpus()` / `rcu_read_lock()` داخليًا لحماية الوصول
- بتزود reference على الـ `kobject` المرتبط بالـ policy عشان تمنع الـ free
- لازم يقابلها `cpufreq_cpu_put()` بدون استثناء

**Who calls it:**
بيستدعيها أي kernel subsystem محتاج يتعامل مع policy معينة — مثلًا thermal drivers, scheduler code, أو cpufreq governors.

---

#### `cpufreq_cpu_put`

```c
void cpufreq_cpu_put(struct cpufreq_policy *policy);
```

**الـ function دي بتعمل إيه:**
بتحرر الـ reference اللي `cpufreq_cpu_get()` عملتها. بتقلل الـ reference count على الـ policy object، وبيسمح للـ core إنه يحرر الـ policy لو الـ count وصل صفر.

**Parameters:**
- `policy` — الـ pointer اللي رجعته `cpufreq_cpu_get()` من قبل

**Return value:**
- `void` — مفيش return

**Key details:**
- لازم تتعمل في نفس الـ context اللي اتعمل فيه `cpufreq_cpu_get()`
- عدم الاستدعاء = memory leak وإمكانية blocking لـ driver unload

**Who calls it:**
نفس الـ caller اللي استدعى `cpufreq_cpu_get()` بعد ما خلص شغله على الـ policy.

---

### Group 2: Notifier Registration — Policy & Transition Notifiers

الـ CPUFreq subsystem بيوفر نظام notifiers قياسي بيسمح لأجزاء تانية في الكيرنل إنها تتنبه لأحداث زي تغيير الـ policy أو تغيير الـ frequency. في نوعين من الـ notifiers: **policy notifiers** و**transition notifiers**.

---

#### `cpufreq_register_notifier`

```c
int cpufreq_register_notifier(struct notifier_block *nb, unsigned int list);
```

**الـ function دي بتعمل إيه:**
بتسجل `notifier_block` في قائمة notifiers محددة داخل الـ CPUFreq core. الـ notifier هيتنادى تلقائيًا لما الحدث المناسب يحصل (policy change أو frequency transition).

**Parameters:**
- `nb` — الـ `notifier_block` المحتوي على الـ callback function (`notifier_call`) والـ priority
- `list` — نوع الـ notifier:
  - `CPUFREQ_POLICY_NOTIFIER` — للتنبه لأحداث الـ policy (create/remove)
  - `CPUFREQ_TRANSITION_NOTIFIER` — للتنبه لأحداث تغيير الـ frequency

**Return value:**
- `0` لو نجحت
- قيمة سالبة (error code) لو فشلت

**Key details:**
- بتستخدم `blocking_notifier_chain_register()` للـ policy notifiers
- بتستخدم `srcu_notifier_chain_register()` للـ transition notifiers (عشان safe مع الـ RCU)
- الـ `nb->notifier_call` لازم يرجع `NOTIFY_OK` أو `NOTIFY_DONE` أو `NOTIFY_BAD`

**Who calls it:**
Drivers أو subsystems محتاجة تعرف بتغييرات الـ policy زي ACPI thermal driver، أو بتغييرات الـ frequency زي timing code.

**Pseudocode flow:**
```
cpufreq_register_notifier(nb, list):
    if list == CPUFREQ_POLICY_NOTIFIER:
        blocking_notifier_chain_register(&cpufreq_policy_notifier_list, nb)
    elif list == CPUFREQ_TRANSITION_NOTIFIER:
        srcu_notifier_chain_register(&cpufreq_transition_notifier_list, nb)
    else:
        return -EINVAL
    return 0
```

---

#### `cpufreq_unregister_notifier`

```c
int cpufreq_unregister_notifier(struct notifier_block *nb, unsigned int list);
```

**الـ function دي بتعمل إيه:**
بتشيل الـ `notifier_block` من قائمة الـ notifiers المحددة. لازم تتستدعى قبل ما الـ module اللي سجّل الـ notifier يتـ unload.

**Parameters:**
- `nb` — نفس الـ `notifier_block` اللي اتسجل قبل كده
- `list` — نفس قيمة الـ `list` اللي اتستخدمت في التسجيل

**Return value:**
- `0` لو نجحت
- قيمة سالبة لو فشلت (مثلًا لو الـ `nb` مش موجود في القائمة)

**Key details:**
- لازم تتعمل في module cleanup path (`module_exit` أو `remove()`)
- عدم الاستدعاء = crash لو الـ module اتـ unload والـ notifier لسه registered

**Who calls it:**
نفس الـ driver أو subsystem اللي استدعى `cpufreq_register_notifier()` وقت cleanup.

---

### Group 3: Policy Notifier Callbacks — Events & Data

لما بيحصل حدث على policy، الـ core بيشغّل كل الـ notifiers المسجلة في `cpufreq_policy_notifier_list` مع تحديد الـ phase والـ data.

---

#### Notifier Callback Signature (Policy)

```c
static int my_policy_notifier(struct notifier_block *nb,
                               unsigned long event, void *data)
{
    struct cpufreq_policy *policy = data;

    switch (event) {
    case CPUFREQ_CREATE_POLICY:
        /* policy just created, policy->min / policy->max valid */
        break;
    case CPUFREQ_REMOVE_POLICY:
        /* policy about to be destroyed */
        break;
    }
    return NOTIFY_OK;
}
```

**الـ event values:**

| Event | المعنى |
|---|---|
| `CPUFREQ_CREATE_POLICY` | الـ policy اتأسست لأول مرة — بيتبعت مرة واحدة |
| `CPUFREQ_REMOVE_POLICY` | الـ policy على وشك تتحذف — فرصة لتنظيف الـ state |

**الـ `data` pointer:**
بيشاور على `struct cpufreq_policy` اللي فيها:
- `policy->min` — أدنى frequency مسموح بيها (kHz)
- `policy->max` — أعلى frequency مسموح بيها (kHz)
- `policy->cpu` — الـ CPU الـ policy مسجلة عليه

---

### Group 4: Transition Notifier Callbacks — Frequency Change Events

الـ transition notifiers بيتشغلوا مرتين لكل CPU أونلاين في الـ policy وقت تغيير الـ frequency — مرة قبل التغيير ومرة بعده.

---

#### Notifier Callback Signature (Transition)

```c
static int my_transition_notifier(struct notifier_block *nb,
                                   unsigned long event, void *data)
{
    struct cpufreq_freqs *freq = data;

    switch (event) {
    case CPUFREQ_PRECHANGE:
        /* freq->old still active, freq->new is the target */
        break;
    case CPUFREQ_POSTCHANGE:
        /* freq->new is now the active frequency */
        break;
    }
    return NOTIFY_OK;
}
```

**الـ `struct cpufreq_freqs` fields:**

| Field | النوع | المعنى |
|---|---|---|
| `policy` | `struct cpufreq_policy *` | الـ policy صاحبة التغيير |
| `old` | `unsigned int` | الـ frequency القديمة (kHz) |
| `new` | `unsigned int` | الـ frequency الجديدة (kHz) |
| `flags` | `unsigned int` | flags خاصة بالـ cpufreq driver |

**Key details:**
- الـ `loops_per_jiffy` بيتـ update تلقائيًا من الـ core في `CPUFREQ_POSTCHANGE` — مش محتاج الـ notifier يعمل ده
- الـ transition notifiers بتستخدم SRCU (Sleepable RCU) — يعني الـ callback ممكن تعمل sleep
- بيتشغلوا لكل CPU أونلاين في الـ policy — مش مرة واحدة للـ policy كلها

---

### Group 5: OPP Table Helpers — Frequency Table Generation

الـ OPP (Operating Performance Points) layer بيوفر abstraction للـ voltage/frequency pairs. الـ helpers دي بتحول الـ OPP data لـ format مناسب للـ CPUFreq core.

---

#### `dev_pm_opp_init_cpufreq_table`

```c
int dev_pm_opp_init_cpufreq_table(struct device *dev,
                                   struct cpufreq_frequency_table **table);
```

**الـ function دي بتعمل إيه:**
بتقرأ كل الـ OPP entries المسجلة للـ `dev` وبتحولها لـ `cpufreq_frequency_table` جاهزة للاستخدام في `policy->freq_table`. بتعمل `kmalloc` للـ table داخليًا وبترجع الـ pointer عن طريق الـ `table` argument.

**Parameters:**
- `dev` — الـ device اللي ليه OPPs مسجلة (عادةً الـ CPU device)
- `table` — pointer لـ pointer هيتملى بعنوان الـ table الجديدة

**Return value:**
- `0` لو نجحت والـ `*table` فيها الـ pointer للـ table
- قيمة سالبة (مثلًا `-ENOMEM`) لو الـ allocation فشلت

**Key details:**
- **ممنوع استدعاؤها من interrupt context** — بتعمل memory allocation
- متاحة بس لو `CONFIG_CPU_FREQ` و`CONFIG_PM_OPP` اتفعّلوا معًا
- الـ table بتتـ allocate وملكيتها بتنتقل للـ caller — لازم `dev_pm_opp_free_cpufreq_table()` تتعمل عليها

**Who calls it:**
Platform initialization code — عادةً في `soc_pm_init()` أو `probe()` function للـ cpufreq driver.

**Example usage:**
```c
soc_pm_init(void)
{
    struct cpufreq_frequency_table *freq_table;
    int r;

    /* populate OPPs via dev_pm_opp_add() calls first */
    r = dev_pm_opp_init_cpufreq_table(dev, &freq_table);
    if (!r)
        policy->freq_table = freq_table;
    /* ... */
}
```

---

#### `dev_pm_opp_free_cpufreq_table`

```c
void dev_pm_opp_free_cpufreq_table(struct device *dev,
                                    struct cpufreq_frequency_table **table);
```

**الـ function دي بتعمل إيه:**
بتحرر الـ `cpufreq_frequency_table` اللي `dev_pm_opp_init_cpufreq_table()` عملتها وبتـ set الـ pointer على `NULL` لمنع الـ use-after-free.

**Parameters:**
- `dev` — نفس الـ device اللي اتبعت لـ `init_cpufreq_table()`
- `table` — pointer لـ pointer الـ table — هيتـ set على `NULL` بعد التحرير

**Return value:**
- `void`

**Key details:**
- لازم تتعمل في cleanup path (مثلًا `exit()` أو `remove()` للـ driver)
- بعد الاستدعاء `*table == NULL` — safe ضد double-free

**Who calls it:**
نفس الكود اللي استدعى `dev_pm_opp_init_cpufreq_table()` في cleanup path.

---

### Group 6: CPUFreq Core Internal Flow (Context)

عشان تفهم الـ functions دي في سياقها الصح، الـ flow العام للـ core هو:

```
CPU Online / Driver Registration
        |
        v
cpufreq_add_device()
        |
        +--> alloc cpufreq_policy
        |
        +--> driver->init(policy)   ← driver fills freq_table, min, max
        |
        +--> notify CPUFREQ_CREATE_POLICY  ← policy notifiers fire here
        |
        v
Governor starts / Frequency changes begin
        |
        v
cpufreq_driver_target() / __cpufreq_driver_target()
        |
        +--> notify CPUFREQ_PRECHANGE   ← transition notifiers fire
        +--> driver->target() / driver->target_index()
        +--> notify CPUFREQ_POSTCHANGE  ← transition notifiers fire
        +--> update loops_per_jiffy
        |
        v
CPU Offline / Driver Unregistration
        |
        +--> notify CPUFREQ_REMOVE_POLICY  ← policy notifiers fire
        +--> free cpufreq_policy
```

الـ `cpufreq_cpu_get/put` بيتستخدموا في أي نقطة في الـ flow ده عشان أي external code يحتاج يمسك reference على الـ policy بـ safety.
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بالـ CPUFreq

الـ **CPUFreq core** بيكتب معلوماته في `/sys/kernel/debug/cpufreq/` لو اتفعّل `CONFIG_DEBUG_FS`:

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/cpufreq/<cpu>/` | معلومات policy لكل CPU |

```bash
# اعرض كل ملفات debugfs المتعلقة بـ cpufreq
ls /sys/kernel/debug/cpufreq/
```

الـ **sched** بيكتب معلومات عن الـ frequency invariance في:

```bash
cat /sys/kernel/debug/sched/debug
```

---

#### 2. مدخلات الـ sysfs الأساسية

الـ **CPUFreq sysfs interface** موجود في `/sys/devices/system/cpu/cpu<N>/cpufreq/`:

| المسار | الوظيفة | كيف تقرأه |
|--------|---------|-----------|
| `/sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq` | التردد الحالي بالـ kHz | `cat` |
| `/sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq` | أدنى تردد مسموح | `cat` |
| `/sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq` | أقصى تردد مسموح | `cat` |
| `/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor` | الـ governor الحالي | `cat` |
| `/sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors` | الـ governors المتاحة | `cat` |
| `/sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies` | الترددات المتاحة | `cat` |
| `/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq` | التردد الحقيقي من الـ hardware | `cat` |
| `/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_min_freq` | أدنى تردد للـ CPU | `cat` |
| `/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq` | أقصى تردد للـ CPU | `cat` |
| `/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_transition_latency` | وقت الانتقال بالـ ns | `cat` |
| `/sys/devices/system/cpu/cpu0/cpufreq/related_cpus` | الـ CPUs المرتبطة بنفس الـ policy | `cat` |
| `/sys/devices/system/cpu/cpu0/cpufreq/affected_cpus` | الـ CPUs المتأثرة بتغيير الـ policy | `cat` |

```bash
# اعرض كل المعلومات دفعة واحدة لكل CPU
for cpu in /sys/devices/system/cpu/cpu*/cpufreq; do
    echo "=== $cpu ==="
    for f in scaling_cur_freq scaling_governor scaling_min_freq scaling_max_freq; do
        printf "  %-30s = %s\n" "$f" "$(cat $cpu/$f 2>/dev/null || echo N/A)"
    done
done
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

الـ **CPUFreq core** بيوفر tracepoints جاهزة في `power` event group:

```bash
# شوف كل events المتعلقة بالـ cpufreq
ls /sys/kernel/tracing/events/power/

# الـ events الأساسية:
# cpu_frequency          — بيتفعل بعد كل تغيير تردد
# cpu_frequency_limits   — بيتفعل لما تتغير الـ min/max limits
```

```bash
# فعّل تتبع تغييرات التردد على كل الـ CPUs
echo 1 > /sys/kernel/tracing/events/power/cpu_frequency/enable
echo 1 > /sys/kernel/tracing/events/power/cpu_frequency_limits/enable
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل workload
stress-ng --cpu 4 --timeout 10s

# اقرأ النتائج
cat /sys/kernel/tracing/trace
echo 0 > /sys/kernel/tracing/tracing_on
```

**مثال على output:**

```
stress-ng-cpu-1234  [002] .... 12345.678901: cpu_frequency: state=2400000 cpu_id=0
stress-ng-cpu-1234  [002] .... 12345.679100: cpu_frequency: state=3600000 cpu_id=0
```

**التفسير:** كل سطر بيقولك الـ CPU اتحوّل لأي تردد (بالـ kHz) وامتى بالضبط.

---

#### 4. الـ printk والـ Dynamic Debug

**تفعيل dynamic debug** للـ CPUFreq core أثناء التشغيل:

```bash
# فعّل كل debug messages في cpufreq.c
echo 'file drivers/cpufreq/cpufreq.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل ملفات الـ cpufreq subsystem
echo 'file drivers/cpufreq/* +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل معلومات الـ notifiers
echo 'module cpufreq +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ debug messages مباشرة
dmesg -w | grep -i cpufreq
```

**لو عايز تفعّل في وقت الـ boot** عبر kernel command line:

```
dyndbg="file drivers/cpufreq/cpufreq.c +p"
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ Config | الوظيفة |
|-----------|---------|
| `CONFIG_CPU_FREQ` | تفعيل الـ CPUFreq subsystem أساساً |
| `CONFIG_CPU_FREQ_DEBUG` | رسائل debug إضافية (قديم، بديله dynamic_debug) |
| `CONFIG_CPU_FREQ_STAT` | إحصائيات الوقت في كل تردد في sysfs |
| `CONFIG_DEBUG_FS` | يفتح `/sys/kernel/debug/` |
| `CONFIG_TRACING` | يفعّل الـ ftrace infrastructure |
| `CONFIG_POWER_TRACER` | tracepoints خاصة بالـ power events |
| `CONFIG_PM_OPP` | دعم الـ OPP table — لازم لـ `dev_pm_opp_init_cpufreq_table` |
| `CONFIG_PM_OPP_DEBUG` | رسائل debug للـ OPP layer |
| `CONFIG_KPROBES` | يسمح بـ dynamic probing على أي function |
| `CONFIG_LOCKDEP` | يكشف deadlocks في الـ policy locking |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CPU_FREQ|PM_OPP|POWER_TRACE'
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الأداة `cpupower`** — أهم أداة userspace للـ CPUFreq:

```bash
# معلومات شاملة عن الـ CPU frequency
cpupower frequency-info

# راقب التردد كل ثانية
watch -n 1 cpupower frequency-info

# اضبط الـ governor
cpupower frequency-set -g performance

# اضبط تردد ثابت
cpupower frequency-set -f 2400MHz

# اعرض الإحصائيات لو CONFIG_CPU_FREQ_STAT مفعّل
cpupower frequency-stats
```

**الأداة `turbostat`** — لمراقبة الترددات الحقيقية مع C-states:

```bash
turbostat --interval 1 --show CPU,Avg_MHz,Bzy_MHz,TSC_MHz
```

**الأداة `perf`** — لربط الـ frequency بالـ performance:

```bash
# راقب أحداث تغيير التردد
perf stat -e power:cpu_frequency sleep 5
perf record -e power:cpu_frequency -a sleep 5
perf report
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---------------------|--------|------|
| `cpufreq: __cpufreq_set_policy: Trying to set policy for cpu that doesn't exist` | CPU غير موجود أو offline | تأكد إن الـ CPU online قبل ضبط الـ policy |
| `cpufreq: cpufreq_set_policy: <X> not in cpufreq list` | الـ governor المطلوب مش مبنيّ | حمّل الـ module المناسب أو ابنيه في الـ kernel |
| `cpufreq: __cpufreq_driver_target: Failed, driver unavailable` | الـ driver اتفكّ أو مش شغال | افحص `lsmod | grep cpufreq` |
| `cpufreq: Failed to register notifier` | الـ notifier مش قادر يسجل نفسه | افحص الـ memory أو conflict في الـ notifier chain |
| `cpufreq: no or improper frequency table` | الـ frequency table فاضية أو غلط | افحص `dev_pm_opp_init_cpufreq_table` وقيم الـ OPP |
| `cpufreq: __target_index: Failed to change cpu frequency` | الـ driver فشل في تغيير التردد | افحص الـ hardware، راجع driver logs |
| `cpufreq: CPU <N>: cpufreq_add_dev failed` | فشل تسجيل الـ policy للـ CPU | افحص الـ memory، راجع الـ driver probe |
| `cpufreq: governor initialization failed` | فشل بدء تشغيل الـ governor | افحص `dmesg` لرسائل أكثر من نفس الوقت |
| `opp: dev_pm_opp_init_cpufreq_table: no OPPs` | مفيش OPP entries متاحة | افحص الـ Device Tree أو ACPI _PSS tables |

---

#### 8. أماكن وضع `dump_stack()` و`WARN_ON()`

**النقاط الاستراتيجية في `drivers/cpufreq/cpufreq.c`:**

```c
/* في cpufreq_set_policy() — لو الـ policy جاي بقيم غريبة */
if (policy->min > policy->max) {
    WARN_ON(1);  /* هنا هتعرف مين طلب policy خاطئة */
    dump_stack();
}

/* في __cpufreq_driver_target() — لو الـ driver رجع error */
ret = cpufreq_driver->target_index(policy, index);
if (ret) {
    WARN_ON_ONCE(ret);  /* مرة واحدة بس عشان ما يملاش الـ log */
}

/* في cpufreq_notify_transition() — لو الـ notifier فشل */
ret = blocking_notifier_call_chain(&cpufreq_transition_notifier_list, ...);
if (ret == NOTIFY_BAD) {
    WARN(1, "cpufreq: notifier returned NOTIFY_BAD\n");
}

/* في cpufreq_cpu_get() — لو حصل race condition */
if (WARN_ON(!policy))
    dump_stack();  /* هتعرف مين استدعى get قبل ما الـ policy تتسجل */
```

---

### Hardware Level

---

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

**الـ kernel بيشوف التردد من الـ hardware عبر `cpuinfo_cur_freq`:**

```bash
# قارن التردد اللي الـ kernel طلبه مع اللي الـ hardware بيشتغل بيه فعلاً
for cpu in $(ls /sys/devices/system/cpu/ | grep '^cpu[0-9]'); do
    scaling=$(cat /sys/devices/system/cpu/$cpu/cpufreq/scaling_cur_freq 2>/dev/null)
    cpuinfo=$(cat /sys/devices/system/cpu/$cpu/cpufreq/cpuinfo_cur_freq 2>/dev/null)
    echo "$cpu: scaling=$scaling kHz  cpuinfo=$cpuinfo kHz"
done
```

**لو الفرق كبير** — ده بيدل على إن:
- الـ hardware مش بيطبق الـ P-state المطلوب
- في Turbo Boost أو hardware throttling شغال
- الـ driver مش بيقرأ التردد صح

---

#### 2. الـ Register Dump

**على x86 — قراءة MSRs الخاصة بالـ frequency:**

```bash
# قرأ الـ P-state الحالي (IA32_PERF_STATUS - MSR 0x198)
rdmsr -a 0x198

# قرأ الـ P-state المطلوب (IA32_PERF_CTL - MSR 0x199)
rdmsr -a 0x199

# قرأ الـ TSC frequency info
rdmsr 0xCE  # MSR_PLATFORM_INFO على Intel
```

**على ARM — عبر `devmem2` أو `/dev/mem`:**

```bash
# تأكد إن CONFIG_STRICT_DEVMEM=n أو استخدم /dev/mem بحذر
devmem2 0xFD000000 w  # عنوان الـ Clock Control Register (مثال على RK3399)

# أو عبر io utility
io -4 0xFD000000
```

**ملاحظة:** العناوين بتختلف حسب الـ SoC — ارجع لـ datasheet.

---

#### 3. منطق أنالوجر / لوجيك أنالايزر

**للتحقق من تغييرات التردد على الـ hardware:**

- **الـ oscilloscope:** قيس على Pin الـ CLK الخارج للـ CPU أو الـ PLL output
- **الـ logic analyzer:** راقب الـ I2C/SPI bus لو الـ regulator بيتحكم في الـ voltage عبرهم (DVFS)

**نقاط القياس المهمة:**
1. الـ VDD_CPU voltage — بيتغير قبل أو بعد التردد؟ (DVFS ordering)
2. الـ PLL lock signal — بياخد قد إيه؟ قارنه بـ `cpuinfo_transition_latency`
3. الـ clock output frequency — قيسها مباشرة وقارنها بما الـ kernel بيقوله

**للـ DVFS debugging:**

```
CPU Voltage (VDD_CPU)
  1.2V ─────┐          ┌─────
            │          │
  1.0V      └──────────┘
                 ↑
            هنا المشكلة: التردد اتغير قبل الـ voltage يتثبت
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | نمط الـ log | التشخيص |
|---------|------------|---------|
| الـ PLL مش بيـ lock | `cpufreq: Failed to change cpu frequency` بعد timeout | افحص الـ transition_latency، زوّد الوقت |
| الـ voltage regulator بطيء | تأخير كبير في الانتقال بين الترددات | افحص الـ regulator ramp rate في DT |
| thermal throttling | `CPU <N>: HARDWARE ERROR` + تردد بينزل وحده | افحص `cat /sys/class/thermal/thermal_zone*/temp` |
| OPP table غلط | `opp: no OPPs` أو تردد مش موجود في الـ table | راجع الـ DT أو ACPI tables |
| عدم توافق الـ policy | `cpufreq_get: could not get hardware cpu frequency` | الـ driver مش قادر يقرأ التردد من الـ hardware |

---

#### 5. الـ Device Tree Debugging

الـ **OPP table** في الـ Device Tree هي مصدر الـ `dev_pm_opp_init_cpufreq_table`:

```bash
# اعرض الـ DT المحمّل على النظام
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 'opp-table'

# أو مباشرة
ls /sys/firmware/devicetree/base/cpus/cpu@0/

# اقرأ الـ OPP entries
for opp in /sys/firmware/devicetree/base/cpus/cpu@0/opp-table/opp*; do
    echo "=== $opp ==="
    cat $opp/opp-hz 2>/dev/null | od -An -tu8
    cat $opp/opp-microvolt 2>/dev/null | od -An -tu4
done
```

**مثال DT صح:**

```dts
cpu0: cpu@0 {
    compatible = "arm,cortex-a53";
    operating-points-v2 = <&cpu0_opp_table>;
};

cpu0_opp_table: opp-table-0 {
    compatible = "operating-points-v2";
    opp-shared;

    opp-1200000000 {
        opp-hz = /bits/ 64 <1200000000>;
        opp-microvolt = <1100000>;
    };
    opp-1800000000 {
        opp-hz = /bits/ 64 <1800000000>;
        opp-microvolt = <1250000>;
    };
};
```

**تحقق إن الـ DT بيطابق الـ hardware specs:**

```bash
# قارن الـ OPPs في الـ DT مع الترددات المتاحة في sysfs
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
```

---

### Practical Commands

---

#### أوامر جاهزة للنسخ

**1. Snapshot سريع لحالة الـ CPUFreq:**

```bash
#!/bin/bash
echo "=== CPUFreq Status Snapshot ==="
echo "Date: $(date)"
echo ""
for cpu in $(ls /sys/devices/system/cpu/ | grep '^cpu[0-9]' | sort -V); do
    base="/sys/devices/system/cpu/$cpu/cpufreq"
    [ -d "$base" ] || continue
    echo "--- $cpu ---"
    printf "  Governor:    %s\n" "$(cat $base/scaling_governor)"
    printf "  Cur Freq:    %s kHz\n" "$(cat $base/scaling_cur_freq)"
    printf "  Min/Max:     %s / %s kHz\n" "$(cat $base/scaling_min_freq)" "$(cat $base/scaling_max_freq)"
    printf "  HW Cur:      %s kHz\n" "$(cat $base/cpuinfo_cur_freq 2>/dev/null || echo N/A)"
done
```

**2. تفعيل الـ ftrace لمراقبة تغييرات التردد:**

```bash
#!/bin/bash
TRACE=/sys/kernel/tracing

# نظّف القديم
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace

# فعّل الـ events
echo 1 > $TRACE/events/power/cpu_frequency/enable
echo 1 > $TRACE/events/power/cpu_frequency_limits/enable

# ابدأ التتبع
echo 1 > $TRACE/tracing_on
echo "Tracing for 10 seconds..."
sleep 10
echo 0 > $TRACE/tracing_on

# اعرض النتائج
echo "=== Frequency Transitions ==="
cat $TRACE/trace | grep cpu_frequency | head -50
```

**مثال output وتفسيره:**

```
     kworker/0:1-45    [000] .... 1234.567: cpu_frequency: state=1800000 cpu_id=0
     kworker/0:1-45    [000] .... 1234.889: cpu_frequency: state=2400000 cpu_id=0
     kworker/0:1-45    [000] .... 1235.001: cpu_frequency: state=3600000 cpu_id=0
```

- `state=1800000` → التردد الجديد = 1800 MHz
- `cpu_id=0` → الـ CPU اللي اتأثر
- الفرق الزمني بين السطرين = وقت الـ transition

**3. مراقبة الـ notifiers عبر kprobe:**

```bash
# تتبع كل استدعاء لـ cpufreq_notify_transition
echo 'p:myprobe cpufreq_notify_transition policy=%di state=%si' \
    > /sys/kernel/tracing/kprobe_events

echo 1 > /sys/kernel/tracing/events/kprobes/myprobe/enable
echo 1 > /sys/kernel/tracing/tracing_on
sleep 5
cat /sys/kernel/tracing/trace
```

**4. مراقبة الـ dynamic debug output:**

```bash
# فعّل الـ debug ثم راقب الـ dmesg
echo 'file drivers/cpufreq/cpufreq.c +p' > /sys/kernel/debug/dynamic_debug/control
dmesg -w | grep -E 'cpufreq|opp'
```

**5. اختبار تغيير الـ governor وقياس التأثير:**

```bash
# احفظ الـ governor الحالي
OLD_GOV=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor)

# جرّب كل governor
for gov in $(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors); do
    cpupower frequency-set -g $gov > /dev/null 2>&1
    printf "Governor: %-15s  Freq: %s kHz\n" \
        "$gov" \
        "$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq)"
    sleep 1
done

# رجّع الـ governor القديم
cpupower frequency-set -g $OLD_GOV
```

**6. التحقق من الـ OPP table:**

```bash
# اعرض الترددات والـ policy info
cpupower frequency-info --policy
cpupower frequency-info --hwlimits
cpupower frequency-info --freq

# لو cpupower مش متاح:
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies \
    | tr ' ' '\n' \
    | sort -n \
    | awk '{printf "  %d MHz\n", $1/1000}'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ CPU يتجمد عند تغيير الـ frequency

#### العنوان
Deadlock في الـ CPUFreq transition notifier على gateway صناعي

#### السياق
شركة بتبني industrial IoT gateway بناءً على **RK3562**. الجهاز بيشتغل كـ Modbus-to-MQTT bridge. في production، الـ device بيتجمد كل بضع ساعات. الـ watchdog بيعمل reset. اللوج بيوقف عند:

```
INFO: rcu_preempt self-detected stall on CPU 0
```

#### المشكلة
فريق الـ BSP عمل custom notifier بيتكلم مع SPI-connected power monitor (INA226) لما الـ CPU بيغير الـ frequency. الـ notifier بيعمل `mutex_lock` على نفس الـ mutex اللي الـ SPI driver بيمسكه أثناء الـ transfer — وده بيعمل deadlock.

#### التحليل
الـ `core.rst` بيوضح إن الـ **CPUFreq transition notifiers** بيتبعتوا **مرتين** لكل CPU online في الـ policy:
- مرة بـ `CPUFREQ_PRECHANGE`
- مرة بـ `CPUFREQ_POSTCHANGE`

الـ notifier callback بيتنادى من جوه `cpufreq_notify_transition()` في `drivers/cpufreq/cpufreq.c`، اللي بيتنادى من الـ driver نفسه أثناء عملية الـ frequency switch — يعني في context حساس جداً. لو الـ callback عمل أي operation بتاخد sleep أو بتمسك mutex مشترك مع أي path تاني بيشتغل في نفس الوقت، الـ deadlock حتمي.

```c
/* الـ callback اللي السبب في المشكلة */
static int power_monitor_cpufreq_notifier(struct notifier_block *nb,
                                           unsigned long event, void *data)
{
    struct cpufreq_freqs *freqs = data;

    if (event == CPUFREQ_POSTCHANGE) {
        mutex_lock(&spi_bus_mutex); /* DEADLOCK هنا */
        ina226_read_power(freqs->new);
        mutex_unlock(&spi_bus_mutex);
    }
    return NOTIFY_OK;
}
```

#### الحل
الحل إنك تفصل الـ SPI read عن الـ notifier باستخدام **work queue**:

```c
static struct work_struct power_update_work;
static unsigned long pending_freq;

static void power_update_worker(struct work_struct *work)
{
    /* هنا safe تمسك mutex لأنك في process context */
    mutex_lock(&spi_bus_mutex);
    ina226_read_power(pending_freq);
    mutex_unlock(&spi_bus_mutex);
}

static int power_monitor_cpufreq_notifier(struct notifier_block *nb,
                                           unsigned long event, void *data)
{
    struct cpufreq_freqs *freqs = data;

    if (event == CPUFREQ_POSTCHANGE) {
        pending_freq = freqs->new;
        schedule_work(&power_update_work); /* non-blocking */
    }
    return NOTIFY_OK;
}
```

#### الدرس المستفاد
الـ transition notifiers بيشتغلوا في context حساس — أي blocking operation جوّاهم ممكن يسبب deadlock أو stall. استخدم work queues أو atomic operations بس جوّا الـ notifier callbacks.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ thermal throttle مش شغال صح

#### العنوان
الـ policy notifier مش بيشوف الـ frequency limits الجديدة من الـ thermal governor

#### السياق
منتج **Android TV box** بالـ **Allwinner H616**. الـ SoC بيوصل لـ 95°C تحت load عالي (4K HEVC decoding). الـ thermal daemon المفروض يخفض الـ max frequency من 1.8GHz لـ 1.2GHz، بس الـ CPU فاضل شغال بـ 1.8GHz. المستخدم شايف frame drops وحرارة مش بتتراجع.

#### المشكلة
فريق الـ Android port عمل custom thermal module بيسجل **policy notifier** بس بيقرأ الـ `policy->max` في `CPUFREQ_CREATE_POLICY` بس — مش في كل تغيير. الـ `CPUFREQ_REMOVE_POLICY` مش بيعمل cleanup صح، فالـ stale value فاضل محفوظ.

#### التحليل
الـ `core.rst` بيوضح إن الـ **policy notifiers** بيتبعتوا في حالتين بس:
- `CPUFREQ_CREATE_POLICY`: لما الـ policy اتعملت
- `CPUFREQ_REMOVE_POLICY`: لما الـ policy اتشالت

الـ policy notifier **مش** الأداة الصح لمتابعة تغييرات الـ limits الجارية. لمتابعة `policy->max` و `policy->min` الـ متغيرين، المفروض تستخدم **transition notifier** أو `cpufreq_register_notifier` مع `CPUFREQ_POLICY_NOTIFIER` وتقرأ القيم في كل event.

```c
/* الكود الغلط */
static int thermal_cpufreq_notifier(struct notifier_block *nb,
                                     unsigned long event, void *data)
{
    struct cpufreq_policy *policy = data;

    /* بيقرأ القيمة مرة واحدة بس عند الإنشاء */
    if (event == CPUFREQ_CREATE_POLICY) {
        cached_max_freq = policy->max; /* stale بعد كده */
    }
    return NOTIFY_OK;
}
```

الحل:

```c
static int thermal_cpufreq_notifier(struct notifier_block *nb,
                                     unsigned long event, void *data)
{
    struct cpufreq_policy *policy = data;

    /* تابع كل تغيير في الـ policy */
    if (event == CPUFREQ_CREATE_POLICY ||
        event == CPUFREQ_REMOVE_POLICY) {
        /* هنا تعمل register/unregister للـ transition notifier */
        update_thermal_limits(policy, event);
    }
    return NOTIFY_OK;
}

/* transition notifier لمتابعة كل تغيير فعلي */
static int thermal_freq_notifier(struct notifier_block *nb,
                                  unsigned long event, void *data)
{
    struct cpufreq_freqs *freqs = data;

    if (event == CPUFREQ_POSTCHANGE) {
        /* هنا الـ policy->max محدّث دايماً */
        enforce_thermal_budget(freqs->policy->max, freqs->new);
    }
    return NOTIFY_OK;
}
```

#### الدرس المستفاد
الـ policy notifiers للإنشاء والحذف بس. لو محتاج تتابع تغييرات الـ frequency أو الـ limits الجارية، استخدم transition notifiers.

---

### السيناريو 3: Custom Board Bring-up على STM32MP1 — الـ OPP table مش بتتحمّل

#### العنوان
`dev_pm_opp_init_cpufreq_table` بترجع `-ENOMEM` في early boot

#### السياق
مهندس embedded بيعمل bring-up لـ custom board بالـ **STM32MP1** (Cortex-A7 + Cortex-M4). الـ board مخصصة لـ medical device. الـ kernel بيبوت بشكل طبيعي لكن CPUFreq driver بيفشل في الـ init:

```
stm32-cpufreq stm32-cpufreq: failed to get OPP table: -12
cpufreq: failed to init cpufreq for cpu 0: -12
```

#### المشكلة
الـ `soc_pm_init()` بتتنادى من `__init` context قبل ما الـ memory allocator يبقى جاهز لكل الـ subsystems. الـ `dev_pm_opp_init_cpufreq_table` بتحاول تعمل `kzalloc` لجدول الـ OPP لكن الـ memory pressure في early boot بيخلي الـ allocation تفشل.

#### التحليل
الـ `core.rst` بيوضح صراحةً:

> **Do not use this function in interrupt context.**

والـ example المذكور:

```c
soc_pm_init()
{
    /* Do things */
    r = dev_pm_opp_init_cpufreq_table(dev, &freq_table);
    if (!r)
        policy->freq_table = freq_table;
    /* Do other things */
}
```

الـ function دي محتاجة `CONFIG_CPU_FREQ` و `CONFIG_PM_OPP` مفعّلين مع بعض. المشكلة إن الـ board's DT كان عنده `operating-points-v2` table بـ voltage values مش موجودة على الـ regulator — فالـ OPP core رفض يسجّل الـ OPPs وجدول الـ frequency فضل فاضي.

فحص الـ DT:

```bash
# تحقق من الـ OPP table في الـ DT
dtc -I dtb -O dts /boot/stm32mp157-custom.dtb | grep -A 20 operating-points
```

الـ output كشف إن voltage `1250000` مكانش موجود في الـ regulator's `regulator-allowed-modes`.

#### الحل
تصحيح الـ DT:

```dts
cpu0_opp_table: opp-table {
    compatible = "operating-points-v2";
    opp-shared;

    opp-650000000 {
        opp-hz = /bits/ 64 <650000000>;
        opp-microvolt = <1200000>; /* صح — موجود في الـ regulator */
        clock-latency-ns = <300000>;
    };

    opp-800000000 {
        opp-hz = /bits/ 64 <800000000>;
        opp-microvolt = <1350000>; /* صح */
        clock-latency-ns = <300000>;
    };
};
```

وتأكيد الـ kernel config:

```bash
grep -E "CONFIG_CPU_FREQ|CONFIG_PM_OPP" /boot/config-$(uname -r)
# المفروض يطلع:
# CONFIG_CPU_FREQ=y
# CONFIG_PM_OPP=y
```

#### الدرس المستفاد
`dev_pm_opp_init_cpufreq_table` مش بس محتاج الـ Kconfig — هي محتاجة إن كل الـ OPP entries في الـ DT تكون valid ومتوافقة مع الـ regulators الفعلية على الـ board.

---

### السيناريو 4: Automotive ECU على i.MX8 — الـ loops_per_jiffy بيتحسب غلط بعد suspend/resume

#### العنوان
timing drift في الـ CAN bus handler بعد الـ CPU يرجع من deep sleep

#### السياق
**Automotive ECU** بالـ **i.MX8QM** بيشتغل كـ gateway بين CAN bus و Ethernet. بعد الـ system يرجع من **suspend to RAM**، الـ CAN frame timing بيبقى مش دقيق — الـ frames بتوصل متأخرة أو بتتعمل timeout. المشكلة بتأثر على الـ AUTOSAR stack.

#### المشكلة
لما الـ CPUFreq driver بيغير الـ frequency بعد الـ resume، الـ kernel بيحدّث `loops_per_jiffy` تلقائياً. لكن custom CAN timing code كان بيكاش قيمة `loops_per_jiffy` في الـ module init ومش بيقرأها تاني.

#### التحليل
الـ `core.rst` بيوضح صراحةً:

> Additionally, the kernel "constant" **loops_per_jiffy** is updated on frequency changes here.

يعني الـ CPUFreq core (في `cpufreq.c`) بيحدّث `loops_per_jiffy` في كل مرة الـ frequency بتتغير. الـ code اللي كاش القيمة دي:

```c
/* الكود الغلط في الـ CAN driver */
static unsigned long cached_lpj;

static int __init can_timing_init(void)
{
    cached_lpj = loops_per_jiffy; /* بيكاش مرة واحدة بس */
    return 0;
}

static void can_udelay(unsigned int usecs)
{
    /* بيستخدم القيمة القديمة بعد ما الـ frequency اتغيرت */
    __delay(usecs * cached_lpj / 1000000);
}
```

#### الحل
الصح إنك تقرأ `loops_per_jiffy` مباشرة أو تستخدم `udelay()` من الـ kernel اللي هي بتعمل ده تلقائياً:

```c
/* الحل: استخدم الـ kernel delay functions مباشرة */
static void can_udelay(unsigned int usecs)
{
    udelay(usecs); /* بتقرأ loops_per_jiffy الـ محدّث تلقائياً */
}
```

أو لو محتاج custom implementation، سجّل transition notifier لتحديث الـ cached value:

```c
static int can_cpufreq_notifier(struct notifier_block *nb,
                                 unsigned long event, void *data)
{
    if (event == CPUFREQ_POSTCHANGE) {
        /* حدّث الـ cached value بعد كل تغيير */
        cached_lpj = loops_per_jiffy;
    }
    return NOTIFY_OK;
}

static struct notifier_block can_cpufreq_nb = {
    .notifier_call = can_cpufreq_notifier,
};

static int __init can_timing_init(void)
{
    cached_lpj = loops_per_jiffy;
    cpufreq_register_notifier(&can_cpufreq_nb, CPUFREQ_TRANSITION_NOTIFIER);
    return 0;
}
```

#### الدرس المستفاد
الـ `loops_per_jiffy` مش constant — الـ CPUFreq core بيحدثها في كل تغيير frequency. أي driver بيعمل timing يجب إنه يقرأها ديناميكياً أو يستخدم الـ kernel's `udelay()`/`ndelay()` مباشرة.

---

### السيناريو 5: IoT Sensor Hub على AM62x — الـ cpufreq_cpu_get بيرجع NULL وبيوقع الـ driver

#### العنوان
NULL pointer dereference في custom power management driver على AM62x

#### السياق
**IoT sensor hub** بالـ **TI AM62x** (Cortex-A53). الجهاز بيجمع بيانات من 8 I2C sensors وبيرفعها على MQTT. الـ system بيـcrash بـ kernel oops في custom power manager:

```
BUG: kernel NULL pointer dereference, address: 0000000000000018
CPU: 0 PID: 234 Comm: power_mgr
Call Trace:
 power_mgr_set_budget+0x4c/0xa0 [custom_pm]
 ...
```

#### المشكلة
الـ custom power manager بيعمل `cpufreq_cpu_get(cpu)` في بداية كل budget cycle، لكن مش بيتحقق من الـ return value قبل ما يستخدمه. لو الـ CPUFreq driver لسه مش registered أو الـ CPU offline، الـ function بترجع NULL.

#### التحليل
الـ `core.rst` بيشرح الـ reference counting:

> Reference counting of the cpufreq policies is done by **cpufreq_cpu_get** and **cpufreq_cpu_put**, which make sure that the cpufreq driver is correctly registered with the core, and will not be unloaded until `cpufreq_put_cpu` is called. That also ensures that the respective cpufreq policy doesn't get freed while being used.

يعني `cpufreq_cpu_get()` ممكن ترجع NULL لو:
- الـ CPUFreq driver مش registered لسه
- الـ CPU المطلوب offline
- الـ policy اتحذف

الكود الغلط:

```c
static void power_mgr_set_budget(int cpu, unsigned long budget_mw)
{
    struct cpufreq_policy *policy;

    policy = cpufreq_cpu_get(cpu);
    /* crash هنا لو policy == NULL */
    pr_info("Current max: %u kHz\n", policy->max);

    /* ... باقي الكود */
    cpufreq_cpu_put(policy);
}
```

#### الحل
دايماً تتحقق من الـ return value وتعمل cleanup صح:

```c
static void power_mgr_set_budget(int cpu, unsigned long budget_mw)
{
    struct cpufreq_policy *policy;
    unsigned int target_freq;

    policy = cpufreq_cpu_get(cpu);
    if (!policy) {
        /* الـ CPUFreq driver مش جاهز أو الـ CPU offline */
        pr_debug("CPUFreq policy not available for cpu%d\n", cpu);
        return;
    }

    pr_info("Current max: %u kHz, min: %u kHz\n", policy->max, policy->min);

    /* احسب الـ target frequency بناءً على الـ budget */
    target_freq = calc_freq_from_budget(policy, budget_mw);

    /* حرر الـ reference — لازم دايماً */
    cpufreq_cpu_put(policy);

    /* اعمل frequency change من بره الـ policy lock */
    cpufreq_update_policy(cpu);
}
```

كمان تأكد من الـ probe order في الـ DT:

```dts
/* تأكد إن الـ cpufreq node بيتـprobe قبل الـ custom power manager */
&cpu0 {
    operating-points-v2 = <&cpu0_opp_table>;
};

&custom_power_mgr {
    /* اربطه بالـ CPU node صراحةً */
    cpus = <&cpu0>;
};
```

وفحص الـ state في runtime:

```bash
# تحقق إن الـ CPUFreq جاهز
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
# المفروض يطلع: cpufreq-dt أو ti-cpufreq

# تحقق من الـ policy
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies
```

#### الدرس المستفاد
الـ `cpufreq_cpu_get()` ممكن ترجع NULL في أي وقت — مش بس في الـ init. كل استخدام لازم يتحقق من الـ NULL، ولازم يكون فيه `cpufreq_cpu_put()` مقابل كل `get` ناجح، حتى في paths الخطأ.
## Phase 7: مصادر ومراجع

### مصادر رسمية — Kernel Documentation

| المسار | الوصف |
|--------|-------|
| `Documentation/cpu-freq/core.rst` | الـ core الرئيسي للـ CPUFreq وشرح الـ notifiers |
| `Documentation/cpu-freq/cpu-drivers.rst` | دليل كتابة driver جديد للـ CPUFreq |
| `Documentation/cpu-freq/governors.rst` | شرح كل الـ governors المتاحة |
| `Documentation/power/opp.rst` | الـ Operating Performance Points (OPP) API |
| `Documentation/admin-guide/pm/cpufreq.rst` | دليل المستخدم للـ CPU performance scaling |
| `include/linux/cpufreq.h` | تعريفات الـ structs والـ macros والـ notifier chains |
| `drivers/cpufreq/cpufreq.c` | الكود الأساسي للـ CPUFreq core |

**الـ online version** للـ official docs:
- [General description of the CPUFreq core and CPUFreq notifiers](https://www.kernel.org/doc/html/latest/cpu-freq/core.html)
- [CPU Performance Scaling — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/pm/cpufreq.html)
- [How to Implement a new CPUFreq Processor Driver](https://docs.kernel.org/cpu-freq/cpu-drivers.html)

---

### مقالات LWN.net

الـ LWN هو المرجع الأهم لفهم تطور الـ CPUFreq subsystem عبر الزمن.

| المقال | الرابط |
|--------|--------|
| CPUfreq Documentation (4/4) — توثيق شامل للـ subsystem | [lwn.net/Articles/8642](https://lwn.net/Articles/8642/) |
| cpufreq core for 2.5 — أول patch للـ cpufreq core في 2.5 | [lwn.net/Articles/1476](https://lwn.net/Articles/1476/) |
| 2.5.40 CPUFreq documentation — توثيق مرحلة 2.5.40 | [lwn.net/Articles/11548](https://lwn.net/Articles/11548/) |
| cpufreq: introduce a new AMD CPU frequency control mechanism | [lwn.net/Articles/868671](https://lwn.net/Articles/868671/) |
| CPUFreq driver using CPPC methods | [lwn.net/Articles/650781](https://lwn.net/Articles/650781/) |
| CPU frequency governors and remote callbacks | [lwn.net/Articles/732740](https://lwn.net/Articles/732740/) |
| cpufreq: interactive: New 'interactive' governor | [lwn.net/Articles/662209](https://lwn.net/Articles/662209/) |
| cpufreq: ARM big LITTLE: Add generic cpufreq driver | [lwn.net/Articles/544457](https://lwn.net/Articles/544457/) |
| cpupowerutils — cpufrequtils extended | [lwn.net/Articles/433002](https://lwn.net/Articles/433002/) |

---

### KernelNewbies.org — تغييرات CPUFreq عبر إصدارات الـ Kernel

كل صفحة من دول بتوثّق التغييرات الخاصة بالـ CPUFreq في كل إصدار:

| الإصدار | الـ Link | أبرز التغيير |
|---------|---------|-------------|
| Linux 4.10 | [kernelnewbies.org/Linux_4.10](https://kernelnewbies.org/Linux_4.10) | تشغيل `intel_pstate` مع generic governors |
| Linux 4.11 | [kernelnewbies.org/Linux_4.11](https://kernelnewbies.org/Linux_4.11) | إضافة `cpufreq.off=1` boot option |
| Linux 4.14 | [kernelnewbies.org/Linux_4.14](https://kernelnewbies.org/Linux_4.14) | الـ scheduler يُحدّث policies للـ remote CPUs |
| Linux 5.7 | [kernelnewbies.org/Linux_5.7](https://kernelnewbies.org/Linux_5.7) | frequency invariant scheduler accounting على x86 |
| Linux 4.7 | [kernelnewbies.org/Linux_4.7](https://kernelnewbies.org/Linux_4.7) | إضافة governor جديد للـ dynamic frequency scaling |
| Linux 3.12 | [kernelnewbies.org/Linux_3.12](https://kernelnewbies.org/Linux_3.12) | تغييرات على الـ CPUFreq infrastructure |
| Linux 2.6.24 | [kernelnewbies.org/Linux_2_6_24](https://kernelnewbies.org/Linux_2_6_24) | تطور الـ CPUFreq في حقبة 2.6 |

---

### eLinux.org — Embedded Linux و CPUFreq

| الصفحة | الرابط |
|--------|--------|
| Tests: R-CAR GEN3 CPUFreq — اختبارات عملية للـ CPUFreq على embedded platform | [elinux.org/Tests:R-CAR-GEN3-CPUFreq](https://elinux.org/Tests:R-CAR-GEN3-CPUFreq) |
| CELF PM Requirements 2006 — متطلبات power management التاريخية | [elinux.org/CELF_PM_Requirements_2006](https://www.elinux.org/CELF_PM_Requirements_2006) |
| Linux Kernel Resources — موارد عامة للـ kernel development | [elinux.org/Linux_Kernel_Resources](https://elinux.org/Linux_Kernel_Resources) |

---

### Mailing List Discussions

| الموضوع | الرابط |
|---------|--------|
| `[PATCH] cpufreq: suspend/resume governors with PM notifiers` — نقاش تعليق الـ governors أثناء الـ suspend | [lkml.indiana.edu](https://lkml.indiana.edu/hypermail/linux/kernel/1311.1/04017.html) |
| `[PATCH 0/4] CPUFreq: Implement per policy instances of governors` — تحويل الـ governors لـ per-policy | [lists.gt.net/linux/kernel/1673358](https://lists.gt.net/linux/kernel/1673358) |
| `[PATCH RFC 3/3] cpufreq: cpufreq-cpu0: clk rate-change notifiers` | [linux.kernel.narkive.com](https://linux.kernel.narkive.com/56pVZCZU/patch-rfc-3-3-cpufreq-cpufreq-cpu0-clk-rate-change-notifiers) |

---

### Source Code المباشر على GitHub

```
https://github.com/torvalds/linux/blob/master/drivers/cpufreq/cpufreq.c
https://github.com/torvalds/linux/blob/master/include/linux/cpufreq.h
```

- **الـ cpufreq.c**: بيحتوي على `cpufreq_register_notifier`, `cpufreq_unregister_notifier`, `cpufreq_cpu_get`, `cpufreq_cpu_put`، وكل منطق الـ policy lifecycle.
- **الـ cpufreq.h**: بيعرّف `struct cpufreq_policy`, `struct cpufreq_freqs`, `CPUFREQ_PRECHANGE`, `CPUFREQ_POSTCHANGE`, `CPUFREQ_CREATE_POLICY`, `CPUFREQ_REMOVE_POLICY`.

---

### كتب مُوصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم**: Chapter 14 — The Linux Device Model (بيشرح `kobject`, `sysfs`, والـ power management infrastructure اللي بيعتمد عليها CPUFreq)
- **متاح مجاناً**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل المهم**: Chapter 11 — Timers and Time Management (بيشرح `loops_per_jiffy` اللي بيتحدّث مع كل تغيير في التردد)
- **الفصل المهم**: Chapter 14 — The Block I/O Layer (نفس pattern الـ notifier chains)

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل المهم**: Chapter 15 — Debugging Embedded Linux Applications
- **الفصل المهم**: Chapter 16 — Kernel Debugging Techniques
- بيشرح كيفية التعامل مع الـ CPUFreq على platforms زي ARM وكيفية debug الـ frequency transitions

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- بيغطي بتفصيل عميق الـ power management infrastructure والـ notifier chains في الـ kernel

---

### Search Terms للبحث عن معلومات أكثر

للبحث في الـ LKML (Linux Kernel Mailing List) وغيره:

```
cpufreq notifier CPUFREQ_PRECHANGE CPUFREQ_POSTCHANGE
cpufreq_register_notifier kernel
cpufreq policy governor schedutil
cpufreq OPP dev_pm_opp_init_cpufreq_table
struct cpufreq_policy min max governor
cpufreq_cpu_get cpufreq_cpu_put reference counting
loops_per_jiffy cpufreq update
CPUFREQ_CREATE_POLICY CPUFREQ_REMOVE_POLICY
intel_pstate cpufreq driver
ARM big.LITTLE cpufreq
```

للبحث في **git log** مباشرة:

```bash
# شوف تاريخ الـ core file
git log --oneline drivers/cpufreq/cpufreq.c

# دوّر على commits خاصة بالـ notifiers
git log --oneline --grep="cpufreq.*notif" drivers/cpufreq/

# تغييرات الـ OPP integration
git log --oneline --grep="opp.*cpufreq\|cpufreq.*opp" drivers/opp/
```

---

### ملخص سريع للمراجع الأهم

```
Priority  المرجع
────────  ──────────────────────────────────────────────────────
★★★★★    docs.kernel.org/cpu-freq/core.html  ← نقطة البداية
★★★★★    drivers/cpufreq/cpufreq.c           ← الكود الحقيقي
★★★★☆    lwn.net/Articles/732740             ← governors & callbacks
★★★★☆    lwn.net/Articles/868671             ← AMD modern cpufreq
★★★☆☆    kernelnewbies.org/Linux_4.14        ← schedutil integration
★★★☆☆    elinux.org/Tests:R-CAR-GEN3-CPUFreq ← embedded testing
★★★☆☆    LDD3 Chapter 14                     ← device model foundation
```
## Phase 8: Writing simple module

### الفكرة

هنستخدم **CPUFreq transition notifier** — وده من أكتر الـ notifiers اللي بتوفّر data مفيدة، لأنه بيتبعت مرتين لكل تغيير في الـ frequency: مرة `CPUFREQ_PRECHANGE` ومرة `CPUFREQ_POSTCHANGE`. هنسجّل الـ old والـ new frequency لكل CPU عند كل تغيير.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * cpufreq_watch.c — CPUFreq transition notifier demo module
 *
 * Registers a cpufreq transition notifier and logs every frequency
 * change (old → new) for every CPU in the policy.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit         */
#include <linux/kernel.h>       /* pr_info                                   */
#include <linux/cpufreq.h>      /* cpufreq_register_notifier, cpufreq_freqs */
#include <linux/notifier.h>     /* notifier_block, NOTIFY_OK                 */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("KernelDoc-Demo");
MODULE_DESCRIPTION("CPUFreq transition notifier — log freq changes");

/* ------------------------------------------------------------------ */
/*  الـ callback اللي بيتاستدعى عند كل تغيير في الـ CPU frequency     */
/* ------------------------------------------------------------------ */
static int cpufreq_watch_cb(struct notifier_block *nb,
                             unsigned long event,
                             void *data)
{
    /*
     * data بيوصّل لـ struct cpufreq_freqs اللي فيها:
     *   policy  → pointer للـ policy (فيه cpu mask، min، max)
     *   old     → الـ frequency القديمة (kHz)
     *   new     → الـ frequency الجديدة (kHz)
     *   flags   → flags من الـ driver
     */
    struct cpufreq_freqs *freqs = (struct cpufreq_freqs *)data;

    /* بنطبع بس عند POSTCHANGE عشان الـ transition خلص فعلاً */
    if (event == CPUFREQ_POSTCHANGE) {
        pr_info("cpufreq_watch: CPU%u  %u kHz → %u kHz\n",
                freqs->policy->cpu,   /* رقم الـ CPU الـ policy بتاعه      */
                freqs->old,           /* الـ frequency اللي كانت           */
                freqs->new);          /* الـ frequency الجديدة             */
    }

    return NOTIFY_OK; /* لازم نرجع NOTIFY_OK عشان الـ chain تكمل */
}

/* ------------------------------------------------------------------ */
/*  تعريف الـ notifier_block                                           */
/* ------------------------------------------------------------------ */
static struct notifier_block cpufreq_watch_nb = {
    .notifier_call = cpufreq_watch_cb,
    /*
     * .priority = 0 (default) — بيتنفّذ في الـ normal order
     * لو عايز تتنفّذ قبل الباقيين تعلّيه (مثلاً 100)
     */
};

/* ------------------------------------------------------------------ */
/*  module_init                                                        */
/* ------------------------------------------------------------------ */
static int __init cpufreq_watch_init(void)
{
    int ret;

    /*
     * CPUFREQ_TRANSITION_NOTIFIER → بيسجّل الـ notifier
     * في الـ cpufreq_transition_notifier_list
     * اللي بتتبعت لما الـ driver يغيّر الـ frequency فعلاً
     */
    ret = cpufreq_register_notifier(&cpufreq_watch_nb,
                                    CPUFREQ_TRANSITION_NOTIFIER);
    if (ret) {
        pr_err("cpufreq_watch: failed to register notifier (%d)\n", ret);
        return ret;
    }

    pr_info("cpufreq_watch: loaded — watching all CPU freq transitions\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                        */
/* ------------------------------------------------------------------ */
static void __exit cpufreq_watch_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما الـ module يتـunload
     * عشان لو جه frequency change بعدين هيحاول يكلّم الـ callback
     * اللي اتشالت من الـ memory → kernel panic
     */
    cpufreq_unregister_notifier(&cpufreq_watch_nb,
                                CPUFREQ_TRANSITION_NOTIFIER);

    pr_info("cpufreq_watch: unloaded\n");
}

module_init(cpufreq_watch_init);
module_exit(cpufreq_watch_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kernel.h` | `pr_info` / `pr_err` |
| `linux/cpufreq.h` | `cpufreq_register_notifier`، `struct cpufreq_freqs`، الـ events |
| `linux/notifier.h` | `struct notifier_block`، `NOTIFY_OK` |

---

#### الـ callback — `cpufreq_watch_cb`

الـ kernel بيستدعيه مرتين لكل تغيير: مرة `CPUFREQ_PRECHANGE` (قبل التغيير) ومرة `CPUFREQ_POSTCHANGE` (بعده). احنا بنفلتر على `POSTCHANGE` بس عشان الـ transition خلصت فعلاً والـ new frequency اتطبّقت.

**الـ** `struct cpufreq_freqs` بتوصّلنا بالـ `old` وبالـ `new` frequency بالـ kHz، وكمان بالـ `policy` اللي فيها رقم الـ CPU وحدود الـ min/max. رجوع `NOTIFY_OK` ضروري عشان الـ notifier chain تكمل ومتتوقفش.

---

#### الـ `notifier_block`

الـ `.notifier_call` هو الـ function pointer اللي الـ kernel هيكلّمه. الـ `.priority` بيتركه default (0) عشان مش محتاجين نتنفّذ قبل أو بعد أي notifier تاني.

---

#### الـ `module_init`

بيسجّل الـ notifier في الـ `cpufreq_transition_notifier_list`. لو الـ registration فشلت (مثلاً الـ CONFIG_CPU_FREQ مش مفعّل) بيرجع الـ error ومش بيكمل.

---

#### الـ `module_exit`

الـ unregister ضروري جداً — لو اتشال الـ module من الـ memory من غير ما نشيل الـ notifier، هيبقى في الـ list pointer لـ function مش موجودة، وأول ما يحصل frequency change هيبوظ الـ kernel. الـ `cpufreq_unregister_notifier` بيشيله من الـ list بأمان.

---

### Makefile للـ build

```makefile
obj-m += cpufreq_watch.o

KDIR := /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ module

```bash
# build
make

# load
sudo insmod cpufreq_watch.ko

# شوف الـ logs (استخدم stress أو cpupower عشان تخلي الـ freq يتغيّر)
sudo cpupower frequency-set -f 1GHz
dmesg | grep cpufreq_watch

# unload
sudo rmmod cpufreq_watch
```

**مثال على الـ output:**

```
cpufreq_watch: loaded — watching all CPU freq transitions
cpufreq_watch: CPU0  2400000 kHz → 1000000 kHz
cpufreq_watch: CPU0  1000000 kHz → 2400000 kHz
cpufreq_watch: unloaded
```
