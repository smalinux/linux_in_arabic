## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem ده؟

الملف ده جزء من **CPUFreq subsystem** — وده واحد من أهم أجزاء kernel Linux اللي بتتعامل مع **إدارة أداء المعالج وتوفير الطاقة**. موجود في MAINTAINERS تحت entry اسمه `CPU FREQUENCY SCALING` وبيشمل كل حاجة من `drivers/cpufreq/` لـ `include/linux/cpufreq.h` لـ `kernel/sched/cpufreq*.c`.

---

### القصة الكاملة — ELI5

#### المشكلة الأصلية

تخيل عندك مكيف هواء في البيت. لو شغّلته على أقصى طاقة طول النهار — فاتورة الكهرباء هتكون ضخمة جداً حتى لو الأوضة باردة أصلاً. الحل المنطقي إنك تشغّله قوي لما الحر شديد، وتخففه لما الجو اتعدل. المعالج (CPU) بالضبط نفس الموضوع.

المعالجات الحديثة مش بتشتغل بسرعة ثابتة — هي بتدعم **P-states** (Performance States) يعني مستويات مختلفة من (تردد × جهد كهربي). كل مستوى أعلى = أسرع في التنفيذ = استهلاك طاقة أكتر = حرارة أعلى. كل مستوى أدنى = أبطأ = توفير طاقة = حرارة أقل.

السؤال اللي بيطرحه الـ CPUFreq هو: **إمتى تشغّل المعالج بأقصى سرعة؟ وإمتى تخففه؟**

---

#### الـ CPUFreq — المدير اللي بيوازن بين الأداء والطاقة

**الـ CPUFreq subsystem** هو الطبقة في Linux kernel اللي بتتولى قرار تغيير سرعة المعالج تلقائياً وبذكاء حسب الحمل الفعلي على الجهاز.

بيتكون من **3 طبقات**:

```
┌────────────────────────────────────────────┐
│              User Space / sysfs            │
│   /sys/devices/system/cpu/cpufreq/policy0/ │
└────────────────────┬───────────────────────┘
                     │
┌────────────────────▼───────────────────────┐
│            CPUFreq Core                    │
│  (cpufreq.c) — البنية التحتية والواجهات    │
└──────────┬─────────────────────┬───────────┘
           │                     │
┌──────────▼──────────┐ ┌────────▼──────────┐
│  Scaling Governors  │ │  Scaling Drivers  │
│  (خوارزميات القرار) │ │  (Hardware talk)  │
│  schedutil, ondemand│ │  intel_pstate,    │
│  conservative, ...  │ │  acpi-cpufreq,... │
└─────────────────────┘ └───────────────────┘
```

---

#### الطبقة الأولى — الـ Core

**الـ core** (`drivers/cpufreq/cpufreq.c`) هو القلب. بيعمل حاجتين أساسيتين:
- بيحتفظ بـ **`struct cpufreq_policy`** لكل CPU (أو لمجموعة CPUs بتشارك نفس الـ hardware interface).
- بيوفر **واجهة sysfs** عشان المستخدم أو أي tool تقدر تشوف وتتحكم في الإعدادات.

الـ **`struct cpufreq_policy`** هو الكيان المحوري — بيمثل "السياسة" المطبقة على مجموعة CPUs بتشارك نفس تحكم hardware في الـ P-state. فيه:
- `min` / `max` — حدود التردد المسموح بيها
- `cur` — التردد الحالي
- `governor` — الـ governor المرتبط بالـ policy دي
- `freq_table` — جدول الترددات المتاحة من الـ hardware

---

#### الطبقة التانية — الـ Governors (خبراء القرار)

الـ **Scaling Governor** هو اللي بيقرر: *"الـ CPU محتاج تردد قد إيه دلوقتي؟"*

| Governor | الفلسفة | الاستخدام |
|---|---|---|
| `performance` | دايماً أقصى تردد مسموح | Servers / Gaming |
| `powersave` | دايماً أدنى تردد مسموح | توفير طاقة أقصى |
| `userspace` | المستخدم هو اللي يقرر | Testing / Manual control |
| `schedutil` | بيسأل الـ scheduler مباشرة | الأحدث والأذكى — الموصى بيه |
| `ondemand` | بيقيس CPU load بشكل دوري | قديم لكن شائع |
| `conservative` | زي ondemand بس بتدريج أبطأ | Battery-powered devices |

الـ **`schedutil`** هو الأذكى: بدل ما يقيس الـ load بنفسه، بيأخذ بيانات **PELT** (Per-Entity Load Tracking) مباشرة من الـ CPU scheduler، وبيحسب التردد المطلوب بالمعادلة:

```
f = 1.25 × f_0 × util / max
```

الـ `1.25` ده هامش أمان (25%) عشان الـ CPU ميعدّيش طاقته قبل ما نرفع التردد.

---

#### الطبقة التالتة — الـ Drivers (المتحدثون مع الـ Hardware)

الـ **Scaling Driver** هو اللي يعرف يكلم الـ hardware فعلاً. كل بنية معالج ليها driver مختلف:

- **`acpi-cpufreq`** — للـ x86 اللي بيستخدم ACPI P-states
- **`intel_pstate`** — حالة خاصة: Intel بيعمل bypass كامل للـ governors وبيتحكم هو بنفسه في التردد من خلال MSR registers
- **`amd-pstate`** — نفس الفكرة لـ AMD
- **`cpufreq-dt`** — للـ ARM boards اللي بتوصف الترددات في Device Tree

---

#### حالة خاصة — الـ intel_pstate

الـ `intel_pstate` بيعمل "bypass" للـ governor layer كله. بدل ما يقبل أوامر من governor، هو بنفسه بيسجل callback مع الـ scheduler وبيقرر التردد مباشرة، لأن Intel بيقدر يقيس أداء الـ workload من داخل الـ hardware نفسه بدقة أعلى مما أي software algorithm يعمله.

---

#### الـ Policy Objects وعلاقتها بالـ CPUs

في المعالجات الحديثة، غالباً الـ cores في نفس الـ cluster بيشتركوا في نفس الـ voltage rail — يعني مستحيل ترفع تردد core واحد من غير ما الباقين يتأثروا. عشان كده الـ CPUFreq بيمثل المجموعة دي كـ **policy object واحد** وليس policy per CPU.

```
Core 0 ─┐
Core 1 ─┼──► cpufreq_policy (policy0) ──► Hardware P-state Control
Core 2 ─┤
Core 3 ─┘
```

---

#### الـ Frequency Boost (Turbo / Core Performance Boost)

فوق الـ P-states العادية، في ميزة اسمها **Frequency Boost** (Turbo Boost على Intel، Core Performance Boost على AMD). دي بتخلي الـ hardware يرفع التردد فوق الحد المستدام لفترة قصيرة لو:
- درجة الحرارة مسمحة
- الباقة مش شغالة بكامل طاقتها

الـ CPUFreq بيتحكم فيها عبر:
- ملف `/sys/devices/system/cpu/cpufreq/boost` (1 = مفعّل، 0 = معطّل)
- أو عبر الـ driver-specific interface زي اللي في `intel_pstate`

---

### الـ sysfs Interface — باب المستخدم

أهم مسار هو:

```
/sys/devices/system/cpu/cpufreq/policyX/
├── affected_cpus           # الـ CPUs اللي الـ policy دي بتأثر عليهم
├── cpuinfo_min_freq        # أدنى تردد يدعمه الـ hardware (kHz)
├── cpuinfo_max_freq        # أقصى تردد يدعمه الـ hardware (kHz)
├── cpuinfo_transition_latency  # وقت التبديل بين P-states (ns)
├── scaling_governor        # الـ governor الحالي (قابل للتغيير)
├── scaling_min_freq        # الحد الأدنى اللي البرنامج بيطبقه
├── scaling_max_freq        # الحد الأقصى اللي البرنامج بيطبقه
├── scaling_cur_freq        # التردد الحالي المطلوب من الـ driver
└── cpuinfo_cur_freq        # التردد الفعلي اللي الـ hardware بيشتغل بيه
```

الفرق بين `scaling_cur_freq` و`cpuinfo_cur_freq`: الأولى هي آخر طلب بعته الـ driver للـ hardware، التانية هي اللي الـ hardware فعلاً شغّاله (ودي أدق لكن مش دايماً متاحة).

---

### الملفات اللي المبتدئ لازم يعرفها

**الـ Core والـ Headers:**

| الملف | الدور |
|---|---|
| `drivers/cpufreq/cpufreq.c` | الـ core الرئيسي — كل الـ framework |
| `include/linux/cpufreq.h` | تعريف `struct cpufreq_policy`, `struct cpufreq_driver`, `struct cpufreq_governor` |
| `include/linux/sched/cpufreq.h` | الواجهة بين الـ scheduler والـ CPUFreq (`update_util_data`, `SCHED_CPUFREQ_IOWAIT`) |
| `kernel/sched/cpufreq.c` | glue بين الـ scheduler والـ governors |
| `kernel/sched/cpufreq_schedutil.c` | تنفيذ الـ schedutil governor |
| `drivers/cpufreq/freq_table.c` | helper functions للتعامل مع frequency tables |
| `drivers/cpufreq/cpufreq_stats.c` | إحصائيات الوقت في كل P-state |
| `drivers/cpufreq/cpufreq_governor.c` | shared code بين الـ governors (ondemand/conservative) |

**الـ Governors:**

| الملف | الدور |
|---|---|
| `drivers/cpufreq/cpufreq_performance.c` | performance governor |
| `drivers/cpufreq/cpufreq_powersave.c` | powersave governor |
| `drivers/cpufreq/cpufreq_userspace.c` | userspace governor |
| `drivers/cpufreq/cpufreq_ondemand.c` | ondemand governor |
| `drivers/cpufreq/cpufreq_conservative.c` | conservative governor |

**أهم الـ Hardware Drivers:**

| الملف | الدور |
|---|---|
| `drivers/cpufreq/intel_pstate.c` | Intel — يعمل bypass للـ governors |
| `drivers/cpufreq/amd-pstate.c` | AMD — نفس نهج intel_pstate |
| `drivers/cpufreq/acpi-cpufreq.c` | x86 عبر ACPI P-states |
| `drivers/cpufreq/cpufreq-dt.c` | ARM boards عبر Device Tree |
| `drivers/cpufreq/qcom-cpufreq-hw.c` | Qualcomm SoCs |

**التوثيق ذات الصلة:**

| الملف | الدور |
|---|---|
| `Documentation/admin-guide/pm/cpufreq.rst` | الملف الحالي — نظرة عامة للمستخدم والـ admin |
| `Documentation/admin-guide/pm/cpufreq_drivers.rst` | دليل كتابة scaling driver جديد |
| `Documentation/admin-guide/pm/intel_pstate.rst` | تفاصيل intel_pstate |
| `Documentation/admin-guide/pm/amd-pstate.rst` | تفاصيل amd-pstate |

---

### خلاصة في جملة واحدة

**الـ `cpufreq.rst`** هو الدليل الشامل اللي بيشرح للـ admin والمطور إزاي Linux بيقرر يرفع أو يخفض سرعة المعالج تلقائياً — مين بيقرر (الـ governor)، مين بينفذ (الـ driver)، وإزاي تتحكم في ده كله من الـ sysfs.
## Phase 2: شرح الـ CPUFreq Framework

---

### المشكلة — ليه الـ CPUFreq موجود أصلاً؟

الـ CPU الحديث مش بيشتغل بتردد واحد ثابت. عنده مجموعة من الـ **Operating Performance Points (OPPs)** — كل نقطة فيهم بتجمع بين تردد معين وجهد معين (voltage). الـ ARM بيسميها P-states في مصطلحات ACPI.

المشكلة المزدوجة:

| الهدف | النتيجة |
|-------|---------|
| تردد عالي + جهد عالي | أداء أعلى + استهلاك طاقة أعلى + حرارة أعلى |
| تردد منخفض + جهد منخفض | استهلاك طاقة أقل + أداء أقل |

**بدون إدارة** → إما تشغّل الـ CPU بأقصى تردد دايماً وتحرق بطارية الجهاز/ترفع الحرارة، أو تشغّله بأدنى تردد دايماً وتضيّع الأداء. الحل هو تغيير التردد ديناميكياً حسب الحمل الفعلي على الـ CPU.

ده بيحتاج:
1. واجهة للتحدث مع الـ hardware وتغيير الـ P-state.
2. خوارزمية لتقدير الحمل على الـ CPU وتحديد التردد المناسب.
3. إطار عمل يربط الاثنين ببعض ويوفّر واجهة موحدة.

**الـ CPUFreq subsystem** هو ده الإطار ده.

---

### الحل — الـ kernel بيعمل إيه؟

الـ kernel قسّم المشكلة لـ 3 طبقات منفصلة تماماً:

```
┌─────────────────────────────────────────────────────────┐
│                    User Space / sysfs                   │
│          /sys/devices/system/cpu/cpufreq/policyX/       │
└─────────────────────────┬───────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│                   CPUFreq Core                          │
│  - Policy management   - sysfs interface                │
│  - Notifier chain      - CPU hotplug integration        │
└──────────┬──────────────────────────┬───────────────────┘
           │                          │
┌──────────▼──────────┐   ┌──────────▼──────────────────┐
│  Scaling Governors  │   │      Scaling Drivers         │
│  (What freq to use?)│   │  (How to set the freq?)      │
│                     │   │                              │
│  - schedutil        │   │  - cpufreq-dt (ARM generic)  │
│  - ondemand         │   │  - intel_pstate              │
│  - conservative     │   │  - acpi-cpufreq              │
│  - performance      │   │  - imx6q-cpufreq             │
│  - powersave        │   │  - qcom-cpufreq-hw            │
│  - userspace        │   │  - ...                       │
└─────────────────────┘   └──────────────────────────────┘
           │                          │
           └──────────┬───────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│              Hardware (CPU P-state registers)            │
│         ACPI / DT OPP table / MSR / DVFS controller     │
└─────────────────────────────────────────────────────────┘
```

الفصل ده مهم جداً: governor لا يعرف إيه الـ hardware، والـ driver لا يعرف إيه الخوارزمية. الـ core هو اللي بيوصّلهم.

---

### الـ Analogy الحقيقية — مسئول الطاقة في المصنع

تخيّل مصنع فيه:

- **الآلات (CPU cores)** — بتشتغل بسرعات مختلفة (تردد منخفض = تكلفة أقل، تردد عالي = إنتاج أكتر).
- **مدير الإنتاج (Scaling Governor)** — بيراقب كمية الطلبات الواردة ويقرر: "محتاجين ماكينات تشتغل بـ 70% من طاقتها دلوقتي".
- **مهندس الكهرباء (Scaling Driver)** — هو اللي يعرف فعلاً إزاي يغيّر جهد وتردد كل ماكينة. بيتكلم مع لوحة التحكم (hardware registers).
- **مدير المصنع (CPUFreq Core)** — هو اللي بينسّق. بياخد قرار المدير وبيحوّله لأوامر للمهندس. وكمان بيدير السجلات (sysfs) ويلبّي طلبات الإدارة العليا (user space).
- **السياسة الموحدة (cpufreq_policy)** — لو 4 ماكينات بتتحكم فيهم بنفس السويتش الكهربائي، بنعاملهم كـ policy واحدة.

| عنصر الـ analogy | مقابله في الـ kernel |
|-----------------|----------------------|
| مدير الإنتاج | `struct cpufreq_governor` |
| مهندس الكهرباء | `struct cpufreq_driver` |
| مدير المصنع | CPUFreq Core (`drivers/cpufreq/cpufreq.c`) |
| السويتش المشترك | `struct cpufreq_policy` |
| طلبات الإنتاج | scheduler utilization callbacks |
| لوحة التحكم | ACPI tables / Device Tree OPPs / MSR registers |
| ماكينة واحدة أو مجموعة | logical CPU أو cluster |

---

### الـ Big Picture — Architecture كامل

```
                     ┌──────────────────────────┐
                     │       CPU Scheduler       │
                     │  (CFS / RT / Deadline)    │
                     │                           │
                     │  cpufreq_update_util()    │◄── per-CPU utilization
                     └────────────┬──────────────┘     update callbacks
                                  │ SCHED_CPUFREQ_IOWAIT flag
                                  │ util, time, flags
                                  ▼
                     ┌──────────────────────────┐
                     │   update_util_data.func   │ ◄── registered by governor
                     │   (governor callback)     │     via cpufreq_add_update_util_hook()
                     └────────────┬──────────────┘
                                  │ decides target freq
                                  ▼
          ┌───────────────────────────────────────────┐
          │              CPUFreq Core                  │
          │                                           │
          │  ┌─────────────────────────────────────┐  │
          │  │        cpufreq_policy               │  │
          │  │  .min / .max / .cur  (kHz)          │  │
          │  │  .cpus (cpumask - online)            │  │
          │  │  .related_cpus (online+offline)      │  │
          │  │  .governor → cpufreq_governor        │  │
          │  │  .freq_table → cpufreq_freq_table[]  │  │
          │  │  .driver_data (opaque)               │  │
          │  │  .kobj → /sys/.../policyX/           │  │
          │  └─────────────────────────────────────┘  │
          │                                           │
          │  cpufreq_driver_target()                  │
          │  cpufreq_driver_fast_switch()             │
          └───────────────────┬───────────────────────┘
                              │
                              ▼
          ┌───────────────────────────────────────────┐
          │           cpufreq_driver                  │
          │                                           │
          │  .init()          → setup OPP table       │
          │  .verify()        → sanitize limits       │
          │  .target_index()  → set P-state by index  │
          │  .fast_switch()   → from scheduler ctx    │
          │  .get()           → read current freq     │
          │  .set_boost()     → Turbo/CPB control     │
          └───────────────────┬───────────────────────┘
                              │
                              ▼
          ┌───────────────────────────────────────────┐
          │         Hardware Interface                 │
          │  ARM: DVFS controller / DT OPPs / clk API │
          │  x86: MSR_IA32_PERF_CTL / ACPI _PSS       │
          │  Qualcomm: CPUCP firmware / QoS bus        │
          └───────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الـ `cpufreq_policy`

الـ **`struct cpufreq_policy`** هي الـ central data structure في كل الـ subsystem. كل حاجة بتدور حواليها.

```c
struct cpufreq_policy {
    /* CPUs sharing the same HW P-state interface */
    cpumask_var_t   cpus;           /* online CPUs in this policy */
    cpumask_var_t   related_cpus;   /* online + offline */
    cpumask_var_t   real_cpus;      /* related + physically present */

    unsigned int    cpu;            /* the "managing" CPU (must be online) */

    struct cpufreq_cpuinfo cpuinfo; /* HW limits: min/max freq, transition latency */

    unsigned int    min;            /* current lower limit (kHz) */
    unsigned int    max;            /* current upper limit (kHz) */
    unsigned int    cur;            /* current frequency (kHz) */

    struct cpufreq_governor  *governor;     /* attached governor */
    void                     *governor_data;

    struct cpufreq_frequency_table *freq_table;  /* available P-states */

    struct kobject  kobj;           /* /sys/devices/system/cpu/cpufreq/policyX */
    struct rw_semaphore rwsem;      /* protects policy reads/writes */

    bool    fast_switch_possible;   /* driver supports in-scheduler freq change */
    bool    fast_switch_enabled;    /* governor enabled fast switch */

    /* Thermal integration */
    struct thermal_cooling_device *cdev;

    /* QoS constraints from PM framework */
    struct freq_constraints constraints;
    struct freq_qos_request *min_freq_req;
    struct freq_qos_request *max_freq_req;

    void    *driver_data;           /* opaque data for the driver */
};
```

**ليه الـ `cpumask`؟** على كتير من المعالجات ARM (Cortex-A big.LITTLE / DynamIQ) الـ CPU cores بتتشارك نفس الـ voltage domain أو الـ PLL. يعني مش ممكن تغير تردد core واحد بمعزل عن الباقيين. فـ 4 cores ممكن يكونوا كلهم محكوم عليهم بـ policy واحدة.

```
  ARM Cortex-A55 cluster          ARM Cortex-A78 cluster
  ┌──────────────────────┐        ┌──────────────────────┐
  │ cpu0 cpu1 cpu2 cpu3  │        │    cpu4   cpu5        │
  │  └───────────────────┤        │  └────────────────────┤
  │    policy0 (LITTLE)  │        │    policy1 (BIG)      │
  │    shared PLL/DVFS   │        │    shared PLL/DVFS    │
  └──────────────────────┘        └──────────────────────┘
```

---

### الـ `cpufreq_driver` — واجهة الـ Hardware

```c
struct cpufreq_driver {
    char    name[CPUFREQ_NAME_LEN];
    u16     flags;

    /* mandatory */
    int     (*init)(struct cpufreq_policy *policy);
        /* called once per new policy: populate freq_table, set cpuinfo limits,
           set related_cpus cpumask */

    int     (*verify)(struct cpufreq_policy_data *policy);
        /* sanitize min/max before applying — must clamp to HW limits */

    /* choose ONE of these two approaches: */

    /* approach 1: governor-driven (most drivers) */
    int     (*target_index)(struct cpufreq_policy *policy, unsigned int index);
        /* set CPU to freq_table[index] */

    unsigned int (*fast_switch)(struct cpufreq_policy *policy,
                                unsigned int target_freq);
        /* same as target_index but callable from scheduler context (no sleeping) */

    /* approach 2: driver manages everything (intel_pstate) */
    int     (*setpolicy)(struct cpufreq_policy *policy);
        /* driver bypasses governor and manages freq itself */

    /* optional helpers */
    unsigned int (*get)(unsigned int cpu);   /* read actual HW frequency */
    int     (*suspend)(struct cpufreq_policy *policy);
    int     (*resume)(struct cpufreq_policy *policy);
    int     (*set_boost)(struct cpufreq_policy *policy, int state);
    void    (*register_em)(struct cpufreq_policy *policy); /* Energy Model */
};
```

**الـ `flags` المهمة:**

| Flag | معناه |
|------|-------|
| `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` | كل policy ليها governor tunables منفصلة (multi-cluster ARM) |
| `CPUFREQ_ASYNC_NOTIFICATION` | الـ driver بيبعت POSTCHANGE notification من خارج `target()` |
| `CPUFREQ_IS_COOLING_DEV` | سجّل الـ driver كـ thermal cooling device تلقائياً |
| `CPUFREQ_NO_AUTO_DYNAMIC_SWITCHING` | امنع governors اللي بتغيّر التردد ديناميكياً (زي ondemand) |

---

### الـ `cpufreq_governor` — الخوارزمية

```c
struct cpufreq_governor {
    char    name[CPUFREQ_NAME_LEN];

    int     (*init)(struct cpufreq_policy *policy);
        /* allocate governor-private data, add sysfs tunables */

    void    (*exit)(struct cpufreq_policy *policy);
        /* free resources */

    int     (*start)(struct cpufreq_policy *policy);
        /* register per-CPU update_util_data hooks with the scheduler
           → from this point, scheduler calls governor on every util change */

    void    (*stop)(struct cpufreq_policy *policy);
        /* unregister hooks */

    void    (*limits)(struct cpufreq_policy *policy);
        /* called when min/max policy limits change */

    struct list_head governor_list;
    u8      flags;
};
```

الـ governor بيتواصل مع الـ scheduler عن طريق:

```c
/* sched/cpufreq.h */
struct update_util_data {
    void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
};

/* governor calls this during ->start() for each CPU */
void cpufreq_add_update_util_hook(int cpu,
                                  struct update_util_data *data,
                                  void (*func)(...));
```

الـ `func` دي هي الـ callback اللي الـ scheduler بيستدعيها عند كل event (task enqueue/dequeue, scheduler tick, ...). **ده هو نقطة التواصل الحرجة بين الـ scheduler والـ CPUFreq.**

---

### الـ `cpufreq_frequency_table` — جدول الـ P-states

```c
struct cpufreq_frequency_table {
    unsigned int flags;       /* CPUFREQ_BOOST_FREQ, CPUFREQ_INEFFICIENT_FREQ */
    unsigned int driver_data; /* opaque index for the driver (e.g. OPP index) */
    unsigned int frequency;   /* kHz */
};
```

الـ driver بيملّي الجدول ده في `->init()`. الـ core والـ governor بيستخدموه لاختيار التردد. الـ relation flags بتحدد إزاي تتعامل مع target freq مش موجودة في الجدول:

```c
#define CPUFREQ_RELATION_L   0  /* lowest freq >= target */
#define CPUFREQ_RELATION_H   1  /* highest freq <= target */
#define CPUFREQ_RELATION_C   2  /* closest to target */
#define CPUFREQ_RELATION_E   BIT(2) /* prefer efficient freq if available */
```

---

### علاقة الـ Structs ببعضها — الصورة الكاملة

```
cpufreq_policy
├── cpumask: cpus, related_cpus, real_cpus
├── cpufreq_cpuinfo: min_freq, max_freq, transition_latency
├── min / max / cur  (user-visible limits)
├── cpufreq_governor *governor ──────────────► cpufreq_governor
│                                               ├── init / exit
│                                               ├── start / stop
│                                               └── limits
├── cpufreq_frequency_table *freq_table ─────► [{ flags, driver_data, freq_kHz }, ...]
│                                               (NULL-terminated with CPUFREQ_TABLE_END)
├── kobject kobj ────────────────────────────► /sys/.../cpufreq/policyX/
├── freq_constraints + freq_qos_request ─────► PM QoS subsystem
├── thermal_cooling_device *cdev ────────────► Thermal subsystem
└── driver_data (void *) ────────────────────► driver-private state

cpufreq_driver (global singleton, one at a time)
├── init(policy)      → fills policy->freq_table, policy->cpuinfo
├── verify(policy)    → clamps min/max
├── target_index()    → talks to HW
├── fast_switch()     → talks to HW from scheduler ctx
└── get()             → reads actual HW freq
```

---

### الـ Initialization Flow — إزاي بيتعمل كل ده؟

```
CPU registration
      │
      ▼
cpufreq_core_init() ──► create /sys/devices/system/cpu/cpufreq/
      │
      ▼
cpufreq_register_driver(driver)
      │
      ▼
for each CPU seen so far:
  cpufreq_online(cpu)
      │
      ├── policy = cpufreq_policy_alloc()   ← new policy object
      │     └── kobject_init()              ← creates policyX/ in sysfs
      │
      ├── driver->init(policy)
      │     └── driver fills: policy->freq_table
      │                       policy->cpuinfo.{min,max}_freq
      │                       policy->related_cpus cpumask
      │
      ├── cpufreq_set_policy()
      │     └── governor->init(policy)
      │           └── governor allocates private data, adds tunables to sysfs
      │
      └── cpufreq_start_governor(policy)
            └── governor->start(policy)
                  └── cpufreq_add_update_util_hook()  ← registers with scheduler
                        for each online CPU in policy
```

---

### ما يملكه الـ Core vs. ما يُفوّضه للـ Drivers

| المسؤولية | CPUFreq Core | Scaling Driver | Scaling Governor |
|-----------|-------------|----------------|------------------|
| إنشاء policy object | نعم | لا | لا |
| تحديد الـ CPUs المشتركة في policy | لا | **نعم** (في `->init`) | لا |
| تحديد min/max freq المتاحة | لا | **نعم** (في `->init`) | لا |
| التحدث مع الـ hardware | لا | **نعم** | لا |
| تقدير حمل الـ CPU | لا | لا | **نعم** |
| اختيار التردد المناسب | لا | لا | **نعم** |
| واجهة sysfs للـ policy | **نعم** | جزئياً (attrs إضافية) | جزئياً (tunables) |
| notifier chain عند تغيير التردد | **نعم** | لا | لا |
| التكامل مع CPU hotplug | **نعم** | لا | لا |
| التكامل مع PM QoS | **نعم** | لا | لا |
| thermal cooling registration | **نعم** (إذا driver طلب) | يطلبها بـ flag | لا |

---

### الـ Fast Switch Path — المسار السريع من الـ Scheduler

**مشكلة:** الـ scheduler بيشتغل في interrupt context أو kernel thread context. الـ normal path لتغيير التردد بيستخدم workqueue (process context). ده بيضيف latency.

**الحل:** لو الـ driver يدعم `->fast_switch()` والـ governor يدعم fast switching:

```
Scheduler tick
      │
      ▼
governor update_util callback (in scheduler context)
      │
      ├── if (policy->fast_switch_enabled)
      │     └── cpufreq_driver_fast_switch(policy, target_freq)
      │           └── driver->fast_switch()   ← direct HW write, no sleep
      │
      └── else
            └── queue work to workqueue
                  └── cpufreq_driver_target() → driver->target_index()
```

الـ `fast_switch_possible` بيحدده الـ driver في `->init()`. الـ `fast_switch_enabled` بيحدده الـ governor في `->init()`. لازم الاتنين يكونوا `true` عشان المسار السريع يشتغل.

---

### الـ intel_pstate الاستثناء

الـ **`intel_pstate`** driver بيكسر الـ layering الاعتيادي. بدل ما يعتمد على governor خارجي:

```
Normal flow:    Governor ──► Core ──► Driver ──► HW
intel_pstate:   Driver (with built-in algorithm) ──► HW
```

بيعمل ده عن طريق:
- تسجيل `->setpolicy()` بدل `->target_index()`.
- الـ core لما يلاقي `->setpolicy()` موجودة، ما بيربطش governor.
- الـ driver نفسه بيسجّل `update_util_data` hooks مع الـ scheduler مباشرة.

ده منطقي لأن Intel processors بيوفروا hardware feedback (HWP - Hardware P-states) اللي صعب تعبّر عنه بـ governor عام.

---

### الـ Frequency Boost (Turbo)

الـ **Turbo Boost** (Intel) / **Core Performance Boost** (AMD) هو القدرة على تشغيل بعض الـ cores بتردد أعلى من الـ TDP المستدام لفترة قصيرة.

```
Normal max: 3.5 GHz  (sustainable)
Boost max:  4.8 GHz  (temporary, when thermals allow)
```

الـ CPUFreq بيدير ده عن طريق:
- الـ `boost` file في `/sys/devices/system/cpu/cpufreq/` (global on/off).
- الـ `CPUFREQ_BOOST_FREQ` flag في `cpufreq_frequency_table` entries.
- الـ `driver->set_boost()` callback.

---

### الـ sysfs Interface — ربط الـ Userspace

كل policy بتظهر كـ directory في sysfs:

```
/sys/devices/system/cpu/cpufreq/
├── policy0/
│   ├── affected_cpus          ← online CPUs in this policy
│   ├── related_cpus           ← all CPUs (online + offline)
│   ├── cpuinfo_min_freq       ← HW minimum (read-only)
│   ├── cpuinfo_max_freq       ← HW maximum (read-only)
│   ├── cpuinfo_transition_latency  ← ns to switch P-states
│   ├── scaling_min_freq       ← user lower limit (read-write)
│   ├── scaling_max_freq       ← user upper limit (read-write)
│   ├── scaling_cur_freq       ← last requested freq
│   ├── cpuinfo_cur_freq       ← actual HW freq (if readable)
│   ├── scaling_governor       ← current governor (read-write)
│   ├── scaling_available_governors
│   ├── scaling_available_frequencies
│   └── scaling_driver
│
├── policy4/
│   └── ... (BIG cluster)
│
└── boost                      ← global Turbo on/off
```

الـ `/sys/devices/system/cpu/cpuY/cpufreq` → symlink لـ policyX الخاصة بـ cpuY.

الـ `freq_attr` struct هي الآلية اللي بتعرّف كل attribute:

```c
struct freq_attr {
    struct attribute attr;          /* sysfs attribute (name, permissions) */
    ssize_t (*show)(struct cpufreq_policy *, char *);
    ssize_t (*store)(struct cpufreq_policy *, const char *, size_t);
};
```

---

### الـ Governors بالتفصيل — مقارنة عملية

| Governor | خوارزمية تقدير الحمل | context التشغيل | مناسب لـ |
|----------|---------------------|----------------|---------|
| **performance** | لا يقدّر — يختار max دايماً | on limit change | servers، benchmarks |
| **powersave** | لا يقدّر — يختار min دايماً | on limit change | إيقاف كامل للـ boost |
| **userspace** | user space يقرر | on write to sysfs | testing، real-time custom |
| **schedutil** | PELT من الـ CFS scheduler | scheduler context | الـ default الموصى به |
| **ondemand** | idle time measurement | workqueue (async) | legacy systems |
| **conservative** | idle time measurement (steps) | workqueue (async) | battery-powered devices |

**الـ schedutil formula:**
```
f = 1.25 × f_0 × util / max
```
- الـ 1.25 headroom معناها إن لو الـ CPU عنده 80% load، بنطلب 100% من التردد — عشان ما نتأخرش على load spikes.
- الـ `SCHED_CPUFREQ_IOWAIT` flag → IO-wait boost: ارفع التردد لـ max فوراً لو كانت task خرجت من I/O wait.

**الـ notifier chain (subsystem آخر مرتبط):**
الـ **Notifier Chain** في الـ kernel هي آلية event broadcasting. الـ CPUFreq بيبعت notifiers لما يتغير التردد (`CPUFREQ_PRECHANGE` / `CPUFREQ_POSTCHANGE`) — drivers تانية أو subsystems ممكن تسجّل نفسها تسمع الأحداث دي.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags, Enums, وConfig Options — Cheatsheet

#### `cpufreq_driver` Flags (u16 bitmask)

| Flag | Value | المعنى |
|------|-------|--------|
| `CPUFREQ_NEED_UPDATE_LIMITS` | `BIT(0)` | الـ driver محتاج يتنبّه لأي تغيير في الحدود حتى لو الـ target frequency ما اتغيّر |
| `CPUFREQ_CONST_LOOPS` | `BIT(1)` | الـ `loops_per_jiffy` مش بيتأثر بتغيير الـ frequency |
| `CPUFREQ_IS_COOLING_DEV` | `BIT(2)` | الـ core يسجّل الـ driver تلقائيًا كـ thermal cooling device |
| `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` | `BIT(3)` | كل policy ليها governor directory مستقل في sysfs |
| `CPUFREQ_ASYNC_NOTIFICATION` | `BIT(4)` | الـ POSTCHANGE notification بتيجي من بره الـ `->target()` |
| `CPUFREQ_NEED_INITIAL_FREQ_CHECK` | `BIT(5)` | الـ core يتحقق إن الـ CPU شغّال على frequency من الـ table |
| `CPUFREQ_NO_AUTO_DYNAMIC_SWITCHING` | `BIT(6)` | يمنع استخدام governors بتعمل dynamic switching |

#### `cpufreq_governor` Flags (u8 bitmask)

| Flag | Value | المعنى |
|------|-------|--------|
| `CPUFREQ_GOV_DYNAMIC_SWITCHING` | `BIT(0)` | الـ governor بيغيّر الـ frequency بنفسه ديناميكيًا |
| `CPUFREQ_GOV_STRICT_TARGET` | `BIT(1)` | لازم يُطبَّق الـ target frequency بالظبط بدون تقريب |

#### Frequency Relation Flags

| Macro | المعنى |
|-------|--------|
| `CPUFREQ_RELATION_L` | أدنى frequency فوق أو تساوي الـ target |
| `CPUFREQ_RELATION_H` | أعلى frequency تحت أو تساوي الـ target |
| `CPUFREQ_RELATION_C` | أقرب frequency للـ target |
| `CPUFREQ_RELATION_E` | flag إضافي: اختار efficient frequency لو متاحة |

#### Notifier Events

| Macro | القيمة | متى بيتبعت |
|-------|--------|-----------|
| `CPUFREQ_PRECHANGE` | 0 | قبل تغيير الـ frequency |
| `CPUFREQ_POSTCHANGE` | 1 | بعد تغيير الـ frequency |
| `CPUFREQ_CREATE_POLICY` | 0 | عند إنشاء policy جديدة |
| `CPUFREQ_REMOVE_POLICY` | 1 | عند حذف policy |

#### ACPI Shared Type

| Macro | القيمة | المعنى |
|-------|--------|--------|
| `CPUFREQ_SHARED_TYPE_NONE` | 0 | مفيش coordination |
| `CPUFREQ_SHARED_TYPE_HW` | 1 | الـ hardware هو اللي بيعمل الـ coordination |
| `CPUFREQ_SHARED_TYPE_ALL` | 2 | كل الـ dependent CPUs لازم تعمل set |
| `CPUFREQ_SHARED_TYPE_ANY` | 3 | أي CPU من الـ dependent CPUs تقدر تعمل set |

#### `cpufreq_table_sorting` Enum

| Value | المعنى |
|-------|--------|
| `CPUFREQ_TABLE_UNSORTED` | الجدول مش مرتب |
| `CPUFREQ_TABLE_SORTED_ASCENDING` | مرتب تصاعدي |
| `CPUFREQ_TABLE_SORTED_DESCENDING` | مرتب تنازلي |

#### Frequency Table Entry Flags

| Macro | المعنى |
|-------|--------|
| `CPUFREQ_BOOST_FREQ` | الـ entry ده بيمثّل boost frequency |
| `CPUFREQ_INEFFICIENT_FREQ` | الـ frequency دي غير كفوءة من ناحية الطاقة |
| `CPUFREQ_ENTRY_INVALID` | الـ entry مش صالح للاستخدام |
| `CPUFREQ_TABLE_END` | نهاية الجدول |

#### Scheduler Interface Flag

| Macro | المعنى |
|-------|--------|
| `SCHED_CPUFREQ_IOWAIT` | الـ CPU كان waiting على I/O — يطلع الـ frequency للـ max فورًا (IO-wait boost) |

---

### الـ Structs المهمة

#### 1. `struct cpufreq_policy`

**الغرض:** الـ object الأساسي في الـ CPUFreq subsystem. بيمثّل مجموعة CPUs بتشارك نفس hardware P-state control interface. كل الـ governors والـ drivers بيشتغلوا من خلاله.

```c
struct cpufreq_policy {
    cpumask_var_t    cpus;          /* Online CPUs فقط بتتشارك الـ policy دي */
    cpumask_var_t    related_cpus;  /* Online + Offline CPUs */
    cpumask_var_t    real_cpus;     /* Related + موجودة فعلًا في الـ system */

    unsigned int     cpu;           /* رقم الـ CPU المسؤول عن إدارة الـ policy */
    struct clk      *clk;           /* الـ clock المرتبط بالـ policy */

    struct cpufreq_cpuinfo cpuinfo; /* hardware limits: min/max/latency */

    unsigned int     min;           /* الحد الأدنى المسموح به (kHz) */
    unsigned int     max;           /* الحد الأقصى المسموح به (kHz) */
    unsigned int     cur;           /* الـ frequency الحالية (kHz) */
    unsigned int     suspend_freq;  /* الـ frequency أثناء الـ suspend */

    struct cpufreq_governor *governor; /* الـ governor الحالي المرتبط بالـ policy */
    void            *governor_data;   /* بيانات خاصة بالـ governor */
    char             last_governor[]; /* اسم آخر governor اتستخدم */

    struct cpufreq_frequency_table *freq_table; /* جدول الـ P-states المتاحة */
    enum cpufreq_table_sorting freq_table_sorted;

    struct rw_semaphore rwsem;       /* الـ lock الرئيسي للـ policy */
    spinlock_t       transition_lock;/* lock أثناء تغيير الـ frequency */
    wait_queue_head_t transition_wait;
    bool             transition_ongoing;

    bool             fast_switch_possible; /* الـ driver يدعم fast switch */
    bool             fast_switch_enabled;  /* الـ governor فعّل الـ fast switch */
    bool             boost_enabled;        /* الـ boost مفعّل للـ policy دي */
    bool             dvfs_possible_from_any_cpu; /* DVFS ينفع من أي CPU */

    struct freq_constraints  constraints;  /* QoS frequency constraints */
    struct freq_qos_request *min_freq_req;
    struct freq_qos_request *max_freq_req;

    struct kobject   kobj;           /* sysfs representation */
    struct list_head policy_list;    /* ربط بقية الـ policies */
    struct cpufreq_stats *stats;     /* إحصائيات الـ transitions */
    struct thermal_cooling_device *cdev; /* thermal cooling integration */
    void            *driver_data;    /* بيانات خاصة بالـ driver */
};
```

**الارتباطات:**
- يحتوي على pointer لـ `cpufreq_governor` (العلاقة: policy تستخدم governor)
- يحتوي على `cpufreq_cpuinfo` بالـ embedding (مش بالـ pointer)
- يشير لـ `cpufreq_frequency_table` (array)
- يشير لـ `cpufreq_stats` للإحصائيات
- يشير لـ `thermal_cooling_device` للتحكم الحراري

---

#### 2. `struct cpufreq_driver`

**الغرض:** بيعرّف الـ hardware-specific backend. هو الطبقة اللي بتتكلم مع الـ hardware مباشرةً. بيُسجَّل مرة واحدة فقط في الوقت نفسه.

```c
struct cpufreq_driver {
    char  name[CPUFREQ_NAME_LEN]; /* اسم الـ driver (مثلاً "acpi-cpufreq") */
    u16   flags;                  /* bitmask من CPUFREQ_* flags */
    void *driver_data;            /* بيانات داخلية للـ driver */

    /* إلزامي لكل الـ drivers */
    int (*init)(struct cpufreq_policy *policy);   /* تهيئة الـ policy */
    int (*verify)(struct cpufreq_policy_data *);  /* التحقق من صحة الـ policy */

    /* اختار واحد من الاتنين: */
    int (*setpolicy)(struct cpufreq_policy *);    /* للـ drivers اللي بتتحكم بنفسها */
    int (*target_index)(struct cpufreq_policy *, unsigned int index); /* الأحدث */
    int (*target)(struct cpufreq_policy *, unsigned int, unsigned int); /* deprecated */

    /* Fast path: من scheduler context مباشرةً */
    unsigned int (*fast_switch)(struct cpufreq_policy *, unsigned int target_freq);
    void (*adjust_perf)(unsigned int cpu, unsigned long min_perf,
                        unsigned long target_perf, unsigned long capacity);

    /* Intermediate frequency support */
    unsigned int (*get_intermediate)(struct cpufreq_policy *, unsigned int index);
    int (*target_intermediate)(struct cpufreq_policy *, unsigned int index);

    unsigned int (*get)(unsigned int cpu);         /* اقرأ الـ frequency الحالية */
    void (*update_limits)(struct cpufreq_policy *);/* firmware notification */
    int  (*bios_limit)(int cpu, unsigned int *limit);

    /* دورة حياة الـ CPU */
    int  (*online)(struct cpufreq_policy *);
    int  (*offline)(struct cpufreq_policy *);
    void (*exit)(struct cpufreq_policy *);
    int  (*suspend)(struct cpufreq_policy *);
    int  (*resume)(struct cpufreq_policy *);
    void (*ready)(struct cpufreq_policy *);  /* بعد اكتمال التهيئة */

    struct freq_attr **attr;       /* sysfs attributes خاصة بالـ driver */
    bool  boost_enabled;
    int  (*set_boost)(struct cpufreq_policy *, int state);
    void (*register_em)(struct cpufreq_policy *); /* Energy Model integration */
};
```

**الارتباطات:**
- يستقبل `cpufreq_policy *` في معظم callbacks
- الـ core بيحتفظ بـ global pointer للـ driver المسجَّل حاليًا

---

#### 3. `struct cpufreq_governor`

**الغرض:** بيعرّف خوارزمية اختيار الـ P-state. بينفّذ القرار: "انتي بحاجة لـ frequency كام؟"

```c
struct cpufreq_governor {
    char name[CPUFREQ_NAME_LEN];  /* مثلاً: "schedutil", "ondemand" */

    int  (*init)(struct cpufreq_policy *);   /* تهيئة بيانات الـ governor */
    void (*exit)(struct cpufreq_policy *);   /* تنظيف البيانات */
    int  (*start)(struct cpufreq_policy *);  /* بدء عمل الـ governor (register util hooks) */
    void (*stop)(struct cpufreq_policy *);   /* إيقاف الـ governor */
    void (*limits)(struct cpufreq_policy *); /* الـ limits اتغيّرت */

    ssize_t (*show_setspeed)(struct cpufreq_policy *, char *buf);
    int     (*store_setspeed)(struct cpufreq_policy *, unsigned int freq);

    struct list_head governor_list; /* ربطه بقائمة الـ governors المسجَّلة */
    struct module   *owner;
    u8               flags;         /* CPUFREQ_GOV_* flags */
};
```

**الارتباطات:**
- الـ `cpufreq_policy` بيشير إليه بـ `policy->governor`
- بيُسجَّل في global list ويُربط بـ `policy` عند التفعيل

---

#### 4. `struct cpufreq_cpuinfo`

**الغرض:** hardware hard limits. القيم دي بتيجي من الـ hardware ومش المفروض تتغيّر.

```c
struct cpufreq_cpuinfo {
    unsigned int max_freq;           /* أعلى frequency ممكنة (kHz) */
    unsigned int min_freq;           /* أدنى frequency ممكنة (kHz) */
    unsigned int transition_latency; /* وقت التبديل بين P-states (nanoseconds) */
};
```

**ملاحظة:** ده embedded جوه `cpufreq_policy` مش pointer.

---

#### 5. `struct cpufreq_policy_data`

**الغرض:** نسخة مؤقتة من policy بيتبعتها للـ `->verify()` callback للتأكد من صحة القيم قبل تطبيقها.

```c
struct cpufreq_policy_data {
    struct cpufreq_cpuinfo          cpuinfo;
    struct cpufreq_frequency_table *freq_table;
    unsigned int cpu;
    unsigned int min; /* kHz */
    unsigned int max; /* kHz */
};
```

**ملاحظة:** الـ `->verify()` مسموح له يعدّل `min` و`max` فقط، مش الـ `freq_table`.

---

#### 6. `struct cpufreq_freqs`

**الغرض:** بيوصف transition في الـ frequency. بيتبعت في الـ notifier chain قبل وبعد التغيير.

```c
struct cpufreq_freqs {
    struct cpufreq_policy *policy; /* الـ policy اللي بتعمل transition */
    unsigned int old;              /* الـ frequency القديمة (kHz) */
    unsigned int new;              /* الـ frequency الجديدة (kHz) */
    u8 flags;                      /* نسخة من driver flags */
};
```

---

#### 7. `struct cpufreq_frequency_table`

**الغرض:** يمثّل entry واحدة في جدول الـ P-states المتاحة. الجدول ده array بتنتهي بـ `CPUFREQ_TABLE_END`.

```c
struct cpufreq_frequency_table {
    unsigned int flags;       /* CPUFREQ_BOOST_FREQ أو CPUFREQ_INEFFICIENT_FREQ */
    unsigned int driver_data; /* بيانات خاصة بالـ driver، الـ core بيتجاهلها */
    unsigned int frequency;   /* الـ frequency بالـ kHz */
};
```

---

#### 8. `struct update_util_data`

**الغرض:** الواجهة بين الـ scheduler والـ CPUFreq governor. الـ scheduler بيستدعي `->func` على كل CPU على أحداث مهمة (task enqueue/dequeue، scheduler tick).

```c
struct update_util_data {
    void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
};
```

**الـ flags** بيحتوي على `SCHED_CPUFREQ_IOWAIT` لو الـ CPU كان waiting على I/O.

---

#### 9. `struct freq_attr`

**الغرض:** sysfs attribute خاصة بالـ CPUFreq policy directory.

```c
struct freq_attr {
    struct attribute attr;
    ssize_t (*show)(struct cpufreq_policy *, char *);
    ssize_t (*store)(struct cpufreq_policy *, const char *, size_t count);
};
```

---

#### 10. `struct gov_attr_set` و`struct governor_attr`

**الغرض:** بيديروا الـ sysfs attributes الخاصة بالـ governor (الـ tunables).

```c
struct gov_attr_set {
    struct kobject   kobj;        /* sysfs object */
    struct list_head policy_list; /* قائمة الـ policies اللي بتستخدم الـ governor ده */
    struct mutex     update_lock;
    int              usage_count;
};

struct governor_attr {
    struct attribute attr;
    ssize_t (*show)(struct gov_attr_set *, char *buf);
    ssize_t (*store)(struct gov_attr_set *, const char *buf, size_t count);
};
```

---

### Struct Relationship Diagram

```
                    ┌─────────────────────────────────────────┐
                    │         cpufreq_driver (global)          │
                    │  name, flags, callbacks                   │
                    │  init/verify/target_index/fast_switch     │
                    └──────────────────┬──────────────────────┘
                                       │ يُعطى policy في كل callback
                                       │
               ┌───────────────────────▼────────────────────────────┐
               │              cpufreq_policy                         │
               │  cpus (online), related_cpus, real_cpus             │
               │  min, max, cur (kHz)                                │
               │  rwsem (reader/writer lock)                         │
               │  transition_lock (spinlock)                         │
               │  fast_switch_possible/enabled                       │
               │  boost_enabled, dvfs_possible_from_any_cpu          │
               │                                                     │
               │  ┌──────────────────┐  ┌───────────────────────┐   │
               │  │ cpufreq_cpuinfo  │  │ cpufreq_frequency_    │   │
               │  │ (embedded)       │  │ table[] (pointer)     │   │
               │  │ max_freq         │  │ flags | driver_data   │   │
               │  │ min_freq         │  │ frequency (kHz)       │   │
               │  │ transition_latency│  │ [entry] ... [END]    │   │
               │  └──────────────────┘  └───────────────────────┘   │
               │                                                     │
               │  governor ──────────────► cpufreq_governor          │
               │                          name, flags                │
               │                          init/exit/start/stop       │
               │                          limits/show_setspeed       │
               │                          governor_list ──► ...      │
               │                                                     │
               │  stats ─────────────────► cpufreq_stats             │
               │  cdev ──────────────────► thermal_cooling_device    │
               │  constraints ───────────► freq_constraints (QoS)    │
               │  kobj ───────────────────► sysfs policyX/           │
               └─────────────────────────────────────────────────────┘
                              ▲
                              │ بيُستدعى من
               ┌──────────────┴────────────────┐
               │       update_util_data         │
               │  func(data, time, flags)       │
               │  ← registered by governor      │
               │    per-CPU in scheduler        │
               └────────────────────────────────┘
```

---

### Lifecycle Diagram — Policy Object

```
[CPU Registration / Driver Registration]
              │
              ▼
  cpufreq_core checks: policy pointer set?
              │
      NO ─────┴─────► Create new cpufreq_policy
                              │
                              ├─► alloc cpumasks (cpus, related_cpus, real_cpus)
                              ├─► init rwsem, spinlocks, wait queues
                              ├─► create sysfs kobject (policyX/)
                              │
                              ▼
                    driver->init(policy)
                              │
                              ├─► set cpuinfo.min_freq, cpuinfo.max_freq
                              ├─► set cpuinfo.transition_latency
                              ├─► set freq_table[]
                              ├─► set policy->cpus mask
                              │
                              ▼
                   Attach scaling governor:
                    governor->init(policy)
                              │
                              ├─► alloc governor private data
                              ├─► add governor sysfs tunables
                              │
                              ▼
                    governor->start(policy)
                              │
                              ├─► cpufreq_add_update_util_hook() per CPU
                              │   (registers scheduler callback)
                              │
                              ▼
                   ┌─── POLICY ACTIVE ───┐
                   │  scheduler events   │
                   │  → util hook called │
                   │  → governor decides │
                   │  → driver sets freq │
                   └─────────────────────┘
                              │
              [CPU hotplug / driver unregister]
                              │
                              ▼
                    governor->stop(policy)
                              │
                              ├─► cpufreq_remove_update_util_hook() per CPU
                              │
                              ▼
                    governor->exit(policy)
                              │
                              ├─► free governor private data
                              │
                              ▼
                    driver->exit(policy)
                              │
                              ├─► cleanup hardware state
                              │
                              ▼
                   destroy sysfs kobject
                   free cpumasks
                   free cpufreq_policy
```

---

### Lifecycle Diagram — Frequency Transition

```
governor decides new frequency needed
              │
              ▼
   ┌─ fast_switch_enabled? ──────────────────────────────────────┐
   │ YES (scheduler context)           NO (async/workqueue)      │
   │                                                              │
   ▼                                                              ▼
driver->fast_switch(policy, target_freq)      cpufreq_freq_transition_begin()
   │                                                    │
   ├─► set hardware register directly         ├─► set transition_ongoing = true
   ├─► returns actual frequency set           ├─► send CPUFREQ_PRECHANGE notifier
   │                                          ├─► wait_queue if concurrent transition
   │                                          │
   │                                          ▼
   │                                  driver->target_index(policy, index)
   │                                          │
   │                                          ├─► optional: get_intermediate()
   │                                          ├─► optional: target_intermediate()
   │                                          └─► set hardware to new P-state
   │                                                    │
   │                                          cpufreq_freq_transition_end()
   │                                                    │
   │                                          ├─► send CPUFREQ_POSTCHANGE notifier
   │                                          ├─► update policy->cur
   │                                          └─► clear transition_ongoing
   │                                                    │
   └────────────────────────────────────────────────────┘
                              │
                              ▼
                    cpufreq_stats_record_transition()
```

---

### Call Flow Diagrams

#### A. مسار الـ schedutil governor في الـ scheduler

```
scheduler event (task enqueue/dequeue/tick)
    │
    ▼
cpufreq_update_util(rq, flags)           ← kernel/sched/cpufreq.c
    │
    ▼
update_util_data->func(data, time, flags) ← hook registered by governor
    │
    ▼ (schedutil governor)
sugov_update_single() or sugov_update_shared()
    │
    ├─► flags & SCHED_CPUFREQ_IOWAIT?
    │       YES → target = policy->max (IO-wait boost)
    │       NO  → f = 1.25 * f_0 * util / max
    │
    ├─► rate_limit_us check (too soon? skip)
    │
    ▼
sugov_set_next_freq()
    │
    ├─ fast_switch_enabled?
    │       YES → cpufreq_driver_fast_switch(policy, freq)
    │                 └─► driver->fast_switch(policy, freq)
    │                           └─► write hardware register
    │
    │       NO  → schedule work on sugov_work
    │                 └─► cpufreq_driver_target(policy, freq, relation)
    │                           └─► __cpufreq_driver_target()
    │                                   └─► driver->target_index(policy, idx)
    │                                             └─► write hardware register
    ▼
done
```

#### B. مسار تسجيل الـ driver

```
module_init() في scaling driver
    │
    ▼
cpufreq_register_driver(driver)
    │
    ├─► check: هل في driver مسجَّل بالفعل؟ → EBUSY
    ├─► copy driver to global cpufreq_driver pointer
    │
    ▼
لكل CPU مسجَّل بالفعل:
cpufreq_online(cpu)
    │
    ├─► policy موجودة؟
    │       NO  → alloc_cpufreq_policy()
    │               └─► init rwsem, locks, kobject
    │
    ├─► driver->init(policy)
    │       └─► set freq_table, cpuinfo.{min,max,latency}, cpus mask
    │
    ├─► driver->verify(policy_data)
    │       └─► clamp min/max to hardware limits
    │
    ├─► cpufreq_add_dev()
    │       └─► create sysfs policyX/ with all attributes
    │
    ├─► attach_governor(policy)
    │       ├─► governor->init(policy)
    │       └─► governor->start(policy)
    │               └─► cpufreq_add_update_util_hook() per online CPU
    │
    └─► driver->ready(policy)  ← optional post-init hook
```

#### C. مسار تغيير الـ governor من sysfs

```
user writes to /sys/.../policyX/scaling_governor
    │
    ▼
store_scaling_governor() [sysfs store handler]
    │
    ├─► down_write(policy->rwsem)
    │
    ├─► cpufreq_stop_governor(policy)
    │       └─► old_governor->stop(policy)
    │               └─► cpufreq_remove_update_util_hook() per CPU
    │
    ├─► old_governor->exit(policy)
    │       └─► free old governor data + remove old tunables from sysfs
    │
    ├─► policy->governor = new_governor
    │
    ├─► new_governor->init(policy)
    │       └─► alloc new governor data + add tunables to sysfs
    │
    ├─► cpufreq_start_governor(policy)
    │       └─► new_governor->start(policy)
    │               └─► cpufreq_add_update_util_hook() per online CPU
    │
    └─► up_write(policy->rwsem)
```

---

### Locking Strategy

#### الـ Locks الموجودة في `cpufreq_policy`

| Lock | النوع | بيحمي إيه |
|------|-------|-----------|
| `policy->rwsem` | `rw_semaphore` | كل بيانات الـ policy: governor، freq limits، cpus mask |
| `policy->transition_lock` | `spinlock_t` | الـ `transition_ongoing` flag وحالة الـ transition |
| `policy->transition_wait` | `wait_queue_head_t` | الـ tasks اللي بتستنى اكتمال transition |

#### قواعد الـ rwsem

```
القراءة (down_read):
  - أي كود عايز يقرأ min/max/cur/governor بدون تعديل
  - الـ governors في الـ normal path

الكتابة (down_write):
  - تغيير الـ governor
  - CPU hotplug (online/offline)
  - تعديل الـ frequency limits من sysfs
  - driver unregistration
```

**الـ Guards المعرَّفة:**
```c
DEFINE_GUARD(cpufreq_policy_write, struct cpufreq_policy *,
             down_write(&_T->rwsem), up_write(&_T->rwsem))

DEFINE_GUARD(cpufreq_policy_read, struct cpufreq_policy *,
             down_read(&_T->rwsem), up_read(&_T->rwsem))
```

#### الـ transition_lock

```c
/* spinlock - يُستخدم من interrupt/scheduler context */
spin_lock(&policy->transition_lock);
    policy->transition_ongoing = true;
    policy->transition_task = current;
spin_unlock(&policy->transition_lock);

/* ... do the frequency change ... */

spin_lock(&policy->transition_lock);
    policy->transition_ongoing = false;
    policy->transition_task = NULL;
spin_unlock(&policy->transition_lock);
wake_up_all(&policy->transition_wait);
```

#### Lock Ordering (ترتيب الأقفال)

```
1. cpufreq_driver global lock (mutex داخل الـ core)
       ↓
2. policy->rwsem (write)
       ↓
3. policy->transition_lock (spinlock)
       ↓
4. scheduler runqueue lock (implicit - fast_switch path)
```

**ملاحظة مهمة:** الـ `fast_switch` path بيتم من scheduler context مع الـ runqueue lock ممسوك، لذلك الـ driver->fast_switch() ممنوع يمسك أي lock ممكن يتنافس مع الـ scheduler. ده السبب في وجود الـ `fast_switch_possible` flag — الـ driver نفسه بيضمن إنه آمن.

#### الـ gov_attr_set lock

```c
struct gov_attr_set {
    struct mutex update_lock; /* يحمي policy_list وusage_count */
};
```

بيتمسك لما يتضاف/يتشال policy من/لقائمة الـ governor، وعند قراءة/كتابة الـ tunables من sysfs.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet لكل الـ APIs المهمة

#### Policy Access & Reference Counting

| Function | الغرض |
|---|---|
| `cpufreq_cpu_get(cpu)` | جيب الـ policy وزود الـ refcount |
| `cpufreq_cpu_get_raw(cpu)` | جيب الـ policy بدون refcount |
| `cpufreq_cpu_policy(cpu)` | جيب الـ policy بدون lock |
| `cpufreq_cpu_put(policy)` | نزّل الـ refcount وحرر الـ kobject |

#### Frequency Query

| Function | الغرض |
|---|---|
| `cpufreq_get(cpu)` | اقرأ الـ frequency الحالية من الـ hardware |
| `cpufreq_quick_get(cpu)` | اقرأ الـ `policy->cur` بدون hardware query |
| `cpufreq_quick_get_max(cpu)` | اقرأ الـ `policy->max` |
| `cpufreq_get_hw_max_freq(cpu)` | اقرأ `cpuinfo.max_freq` |

#### Driver Registration

| Function | الغرض |
|---|---|
| `cpufreq_register_driver(drv)` | سجّل الـ scaling driver |
| `cpufreq_unregister_driver(drv)` | الغِ تسجيل الـ driver |

#### Governor Registration & Control

| Function | الغرض |
|---|---|
| `cpufreq_register_governor(gov)` | سجّل governor جديد في الـ kernel |
| `cpufreq_unregister_governor(gov)` | الغِ تسجيله |
| `cpufreq_start_governor(policy)` | ابدأ الـ governor على الـ policy |
| `cpufreq_stop_governor(policy)` | وقّف الـ governor |
| `cpufreq_default_governor()` | جيب الـ governor الافتراضي |
| `cpufreq_fallback_governor()` | جيب الـ fallback governor |

#### Frequency Targeting (from governors)

| Function | الغرض |
|---|---|
| `cpufreq_driver_target(policy, freq, relation)` | غيّر الـ frequency مع full notification |
| `__cpufreq_driver_target(policy, freq, relation)` | نفس السابق بدون الـ policy lock |
| `cpufreq_driver_fast_switch(policy, freq)` | fast path من scheduler context |
| `cpufreq_driver_adjust_perf(cpu, min, target, cap)` | performance hint للـ driver |
| `cpufreq_driver_resolve_freq(policy, freq)` | جيب أقرب frequency من الـ table |

#### Transition Notifications

| Function | الغرض |
|---|---|
| `cpufreq_freq_transition_begin(policy, freqs)` | أرسل `CPUFREQ_PRECHANGE` notifier |
| `cpufreq_freq_transition_end(policy, freqs, failed)` | أرسل `CPUFREQ_POSTCHANGE` notifier |
| `cpufreq_register_notifier(nb, list)` | سجّل notifier لأحداث الـ CPUFreq |
| `cpufreq_unregister_notifier(nb, list)` | الغِ التسجيل |

#### Frequency Table Helpers

| Function / Macro | الغرض |
|---|---|
| `cpufreq_frequency_table_cpuinfo(policy)` | ملأ `cpuinfo.{min,max}_freq` من الـ table |
| `cpufreq_frequency_table_verify(policy)` | تحقق من صحة الـ policy limits |
| `cpufreq_frequency_table_target(policy, freq, min, max, rel)` | جيب index من الـ table بناءً على الـ relation |
| `cpufreq_frequency_table_get_index(policy, freq)` | جيب index لـ frequency معينة |
| `cpufreq_table_set_inefficient(policy, freq)` | علّم frequency كـ inefficient |
| `cpufreq_for_each_entry(pos, table)` | iterate على كل entries |
| `cpufreq_for_each_valid_entry(pos, table)` | iterate متجاهلاً `CPUFREQ_ENTRY_INVALID` |
| `cpufreq_for_each_efficient_entry_idx(pos, table, idx, eff)` | iterate مع skip للـ inefficient |

#### Scheduler Interface

| Function | الغرض |
|---|---|
| `cpufreq_add_update_util_hook(cpu, data, func)` | سجّل utilization callback في الـ scheduler |
| `cpufreq_remove_update_util_hook(cpu)` | الغِ تسجيل الـ callback |
| `cpufreq_this_cpu_can_update(policy)` | تحقق إذا كان الـ CPU الحالي يقدر يعمل update |
| `map_util_freq(util, freq, cap)` | حوّل utilization لـ target frequency |
| `map_util_perf(util)` | طبّق الـ 25% headroom على الـ utilization |

#### Power Management

| Function | الغرض |
|---|---|
| `cpufreq_suspend()` | suspend كل الـ policies |
| `cpufreq_resume()` | resume كل الـ policies |
| `cpufreq_generic_suspend(policy)` | generic implementation للـ `->suspend()` callback |
| `cpufreq_boost_enabled()` | هل الـ boost مفعّل system-wide؟ |

#### Misc Helpers

| Function | الغرض |
|---|---|
| `cpufreq_update_policy(cpu)` | أعِد تطبيق الـ policy (e.g., بعد تغيير الـ governor) |
| `cpufreq_update_limits(cpu)` | trigger limit update من firmware |
| `refresh_frequency_limits(policy)` | حدّث `policy->min/max` من الـ QoS constraints |
| `cpufreq_enable_fast_switch(policy)` | فعّل الـ fast switch على الـ policy |
| `cpufreq_disable_fast_switch(policy)` | عطّل الـ fast switch |
| `have_governor_per_policy()` | هل الـ driver يستخدم per-policy governor dirs؟ |
| `get_cpu_idle_time(cpu, wall, io_busy)` | جيب الـ idle time للـ CPU (تستخدمها الـ governors) |
| `cpufreq_scale(old, div, mult)` | `old * mult / div` بدون 32-bit overflow |
| `cpufreq_verify_within_limits(policy, min, max)` | clamp الـ policy limits في حدود معينة |
| `cpufreq_verify_within_cpu_limits(policy)` | clamp بحدود الـ hardware |
| `cpufreq_get_pressure(cpu)` | جيب الـ thermal pressure على الـ CPU |

---

### المجموعة الأولى: Registration — تسجيل الـ Driver والـ Governor

هدف هذه المجموعة هو ربط كود الـ driver أو الـ governor بالـ CPUFreq core. التسجيل بيحصل مرة واحدة فقط (في `module_init` أو `core_initcall`)، والـ core يعتمد عليهم لإدارة كل الـ CPUs.

---

#### `cpufreq_register_driver`

```c
int cpufreq_register_driver(struct cpufreq_driver *driver_data);
```

الـ CPUFreq core بيسمح بـ driver واحد بس في كل الـ system. الـ function دي بتسجّل الـ driver وبعدين بتشتغل على كل الـ CPUs الموجودة في النظام وقت التسجيل عبر استدعاء `subsys_interface_register`، اللي بيستدعي بدوره `->init()` callback للـ driver لكل CPU.

**Parameters:**
- `driver_data` — مؤشر لـ `struct cpufreq_driver` المكتمل الإعداد، لازم يكون `name` و`init` و`verify` محددين فيه.

**Return:**
- `0` على النجاح، أو `errno` سالب في حالة الخطأ (مثلاً `-EEXIST` لو في driver مسجّل فعلاً).

**Key details:**
- بياخد `cpufreq_driver_lock` (spinlock) وقت الكتابة.
- لو الـ driver عنده `CPUFREQ_NEED_INITIAL_FREQ_CHECK` flag، الـ core بيتحقق إن كل CPU شغّال على frequency موجودة في الـ table.
- بعد النجاح، الـ `cpufreq_driver` global pointer بيتحدد.

**Who calls it:** الـ scaling driver نفسه من `module_init()` أو `device_initcall()`.

---

#### `cpufreq_unregister_driver`

```c
void cpufreq_unregister_driver(struct cpufreq_driver *driver_data);
```

بتعمل tear-down كامل لكل الـ policies المرتبطة بالـ driver وبتشيل الـ driver من الـ core. بتستدعي `->exit()` callback لكل policy موجودة.

**Parameters:**
- `driver_data` — نفس الـ pointer المسجّل في `cpufreq_register_driver`.

**Return:** لا شيء (void).

**Key details:**
- بتعمل `subsys_interface_unregister` اللي بيدمّر كل الـ policy objects.
- بتصفّر الـ `cpufreq_driver` global pointer.
- بتنتظر انتهاء أي frequency transition جارية قبل ما تكمل.

**Who calls it:** الـ driver من `module_exit()`.

---

#### `cpufreq_register_governor`

```c
int cpufreq_register_governor(struct cpufreq_governor *governor);
```

بتضيف الـ governor لقائمة `cpufreq_governor_list` المتاحة في الـ kernel. الـ governors المسجّلين بيظهروا في `scaling_available_governors` sysfs attribute.

**Parameters:**
- `governor` — مؤشر لـ `struct cpufreq_governor` مع `name` و callbacks مضبوطين.

**Return:** `0` على النجاح، `-EBUSY` لو في governor بنفس الاسم مسجّل.

**Key details:**
- بياخد `cpufreq_governor_mutex`.
- الـ macro التسهيلي `cpufreq_governor_init(__governor)` بيولّد تلقائياً `__init` function بتستدعي هذه الـ function عند `core_initcall`.

**Who calls it:** الـ governor module من `core_initcall()`.

---

#### `cpufreq_unregister_governor`

```c
void cpufreq_unregister_governor(struct cpufreq_governor *governor);
```

بتشيل الـ governor من قائمة الـ governors المتاحة. لو الـ governor ده هو الـ default، الـ core بيتحول للـ fallback governor.

**Parameters:**
- `governor` — نفس الـ pointer المسجّل سابقاً.

**Key details:** الـ macro `cpufreq_governor_exit` بيولّد `module_exit` function تستدعيها.

---

### المجموعة التانية: Policy Access — الوصول لكائنات الـ Policy

الـ policy object هو الوحدة الأساسية في CPUFreq. المجموعة دي بتوفر طرق آمنة للوصول للـ policy مع إدارة صحيحة للـ reference counting والـ locking.

---

#### `cpufreq_cpu_get`

```c
struct cpufreq_policy *cpufreq_cpu_get(unsigned int cpu);
```

بتجيب الـ policy المرتبطة بالـ CPU المحدد، وبتزود الـ kobject reference count بعد الحصول عليها. ده بيضمن إن الـ policy object مش هيتحرر من الميموري طول ما الـ caller شغّال بيه.

**Parameters:**
- `cpu` — رقم الـ logical CPU.

**Return:** مؤشر للـ `cpufreq_policy` أو `NULL` لو الـ CPU مش موجود أو CPUFreq مش مفعّل.

**Key details:**
- بتستخدم `cpufreq_cpu_get_raw` داخلياً وبتزود `kobject_get`.
- اللي بياخد الـ policy لازم يستدعي `cpufreq_cpu_put` بعد ما يخلص.
- الـ DEFINE_FREE macro بتوفر scope-based cleanup: `struct cpufreq_policy *p __free(put_cpufreq_policy) = cpufreq_cpu_get(cpu);`

**Who calls it:** أي kernel code يحتاج يشتغل على policy CPU معين بأمان.

---

#### `cpufreq_cpu_get_raw`

```c
struct cpufreq_policy *cpufreq_cpu_get_raw(unsigned int cpu);
```

بتجيب الـ policy من الـ `per_cpu(cpufreq_cpu_data, cpu)` مباشرة بدون زيادة الـ refcount.

**Parameters:** `cpu` — رقم الـ CPU.

**Return:** مؤشر للـ policy أو `NULL`.

**Key details:** استخدامها في أماكن محدودة جداً، لازم الـ caller يتأكد إن الـ policy مش هتتحرر في نفس الوقت (مثلاً من داخل RCU read-side section أو مع الـ policy rwsem محجوز).

---

#### `cpufreq_cpu_put`

```c
void cpufreq_cpu_put(struct cpufreq_policy *policy);
```

بتنزّل الـ kobject reference count. لو وصل لصفر، الـ policy object بيتحرر.

**Parameters:**
- `policy` — الـ policy المحصول عليها سابقاً من `cpufreq_cpu_get`.

**Who calls it:** كل كود بياخد policy عبر `cpufreq_cpu_get`.

---

#### `policy_is_inactive` / `policy_is_shared`

```c
static inline bool policy_is_inactive(struct cpufreq_policy *policy);
static inline bool policy_is_shared(struct cpufreq_policy *policy);
```

**`policy_is_inactive`:** بتتحقق لو `policy->cpus` فاضية، يعني كل الـ CPUs في الـ policy offline.

**`policy_is_shared`:** بتتحقق لو في أكتر من CPU واحد بيشارك نفس الـ policy (يعني نفس الـ P-state control interface).

**Who calls them:** الـ core و الـ governors قبل اتخاذ قرار frequency.

---

### المجموعة التالتة: Frequency Query — قراءة الـ Frequency

---

#### `cpufreq_get`

```c
unsigned int cpufreq_get(unsigned int cpu);
```

بتقرأ الـ frequency الحالية للـ CPU من الـ hardware مباشرة عبر استدعاء `cpufreq_driver->get(cpu)`.

**Parameters:** `cpu` — رقم الـ CPU.

**Return:** الـ frequency بالـ kHz، أو `0` لو مش ممكن القراءة.

**Key details:**
- بياخد `policy->rwsem` للقراءة أثناء الاستدعاء.
- أبطأ من `cpufreq_quick_get` لأنها بتعمل actual hardware read.
- ممكن تتحقق إن الـ policy->cur صح وتحدّثه.

**Who calls it:** user space عبر `cpuinfo_cur_freq` sysfs attribute، وبعض الـ governors.

---

#### `cpufreq_quick_get`

```c
unsigned int cpufreq_quick_get(unsigned int cpu);
```

بترجع `policy->cur` مباشرة من الـ struct بدون hardware access. أسرع بكتير من `cpufreq_get`.

**Parameters:** `cpu` — رقم الـ CPU.

**Return:** الـ `policy->cur` بالـ kHz أو `0`.

**Who calls it:** الـ scheduler وكل كود يحتاج frequency estimate سريعة.

---

#### `cpufreq_quick_get_max`

```c
unsigned int cpufreq_quick_get_max(unsigned int cpu);
```

بترجع `policy->max`، يعني أقصى frequency مسموح بيها للـ policy.

**Return:** قيمة `policy->max` بالـ kHz.

---

### المجموعة الرابعة: Governor Control — التحكم في الـ Governor

---

#### `cpufreq_start_governor`

```c
int cpufreq_start_governor(struct cpufreq_policy *policy);
```

بتبدأ تشغيل الـ governor المرتبط بالـ policy عبر استدعاء `governor->start(policy)`. ده بيخلي الـ governor يبدأ تسجيل utilization callbacks مع الـ scheduler لكل الـ CPUs الـ online في الـ policy.

**Parameters:**
- `policy` — الـ policy اللي هيتشغّل عليها الـ governor.

**Return:** `0` على النجاح أو `errno` سالب.

**Key details:**
- لازم الـ `governor->init(policy)` يتستدعى الأول (في الـ attach step).
- الـ core بيستدعيها عند: إنشاء policy جديدة، رجوع CPU للـ online بعد ما كانت كل الـ siblings offline، وتغيير الـ governor.

**Pseudocode:**
```
cpufreq_start_governor(policy):
    if governor->start:
        ret = governor->start(policy)
        if ret < 0:
            return ret
    if governor->limits:
        governor->limits(policy)  // apply current limits
    return 0
```

---

#### `cpufreq_stop_governor`

```c
void cpufreq_stop_governor(struct cpufreq_policy *policy);
```

بتوقّف الـ governor عبر استدعاء `governor->stop(policy)`. الـ governor بيشيل كل الـ utilization callbacks المسجّلة مع الـ scheduler.

**Parameters:** `policy` — الـ policy المراد إيقاف الـ governor عليها.

**Key details:**
- بتستدعيها الـ core عند: CPU hotplug (آخر CPU في الـ policy هييجي offline)، تغيير الـ governor، وإلغاء تسجيل الـ driver.
- الترتيب اللازم: `stop` → تعديل → `start` (في حالة restart).

---

### المجموعة الخامسة: Frequency Targeting — تغيير الـ Frequency

دي القلب الحقيقي للـ CPUFreq. الـ governors والـ core بيستخدموا هذه الـ functions لطلب P-state جديد من الـ hardware.

---

#### `cpufreq_driver_target`

```c
int cpufreq_driver_target(struct cpufreq_policy *policy,
                          unsigned int target_freq,
                          unsigned int relation);
```

بتطلب من الـ driver تغيير frequency الـ policy للـ `target_freq`. بتأخد الـ `policy->rwsem` بالـ write mode، وبتبعت `PRECHANGE` و`POSTCHANGE` notifications.

**Parameters:**
- `policy` — الـ policy المراد تغيير frequency-ha.
- `target_freq` — الـ frequency المطلوبة بالـ kHz.
- `relation` — كيفية اختيار الـ frequency من الـ table:
  - `CPUFREQ_RELATION_L` — أقل frequency >= target
  - `CPUFREQ_RELATION_H` — أعلى frequency <= target
  - `CPUFREQ_RELATION_C` — أقرب frequency للـ target
  - يمكن OR مع `CPUFREQ_RELATION_E` لتفضيل الـ efficient frequencies.

**Return:** `0` على النجاح أو `errno` سالب.

**Key details:**
- **لا تستدعيها من scheduler context** — استخدم `cpufreq_driver_fast_switch` بدلاً منها.
- بتمر بـ `__cpufreq_driver_target` اللي بتستدعي `driver->target_index()` أو `driver->target()`.

**Pseudocode:**
```
cpufreq_driver_target(policy, target_freq, relation):
    down_write(policy->rwsem)
    ret = __cpufreq_driver_target(policy, target_freq, relation)
    up_write(policy->rwsem)
    return ret

__cpufreq_driver_target(policy, target_freq, relation):
    target_freq = clamp(target_freq, policy->min, policy->max)
    if driver->target_index:
        idx = cpufreq_frequency_table_target(policy, target_freq, relation)
        cpufreq_freq_transition_begin(policy, freqs)
        if driver->get_intermediate:
            mid = driver->get_intermediate(policy, idx)
            if mid:
                driver->target_intermediate(policy, idx)  // go to mid first
        ret = driver->target_index(policy, idx)
        cpufreq_freq_transition_end(policy, freqs, ret)
    elif driver->target:
        ret = driver->target(policy, target_freq, relation)
    return ret
```

---

#### `cpufreq_driver_fast_switch`

```c
unsigned int cpufreq_driver_fast_switch(struct cpufreq_policy *policy,
                                        unsigned int target_freq);
```

الـ fast path لتغيير الـ frequency من **scheduler context مباشرة** بدون notifiers ولا locks ولا workqueue. ممكنة بس لو الـ driver يدعم `fast_switch_possible` والـ policy عندها `fast_switch_enabled`.

**Parameters:**
- `policy` — الـ policy.
- `target_freq` — الـ frequency المطلوبة.

**Return:** الـ frequency الفعلية اللي اتحدد عليها الـ hardware (بالـ kHz)، أو `0` لو فشل.

**Key details:**
- لا تبعت notifiers (لأن الـ notifier context مش compatible مع scheduler context).
- بتحدّث `policy->cur` مباشرة بعد النجاح.
- **الـ schedutil governor** هو المستخدم الأساسي لها.

---

#### `cpufreq_driver_adjust_perf`

```c
void cpufreq_driver_adjust_perf(unsigned int cpu,
                                unsigned long min_perf,
                                unsigned long target_perf,
                                unsigned long capacity);
```

بديل أكتر تعبيراً من `fast_switch` للـ drivers اللي بتشتغل بمقياس performance داخلي (مش kHz). بتسمح للـ driver يعرف كل من الـ minimum المطلوب والـ target المرغوب فيه، مع الـ max capacity.

**Parameters:**
- `cpu` — الـ CPU رقمه.
- `min_perf` — أقل performance مطلوبة (لـ guarantees).
- `target_perf` — الـ target performance المرغوب فيه.
- `capacity` — أقصى capacity ممكنة للـ CPU.

**Key details:** بتستدعي `cpufreq_driver->adjust_perf(cpu, min_perf, target_perf, capacity)` مباشرة. لو الـ driver مش بيدعمها، الـ fall back لـ `fast_switch` بيحصل تلقائياً.

---

#### `cpufreq_driver_resolve_freq`

```c
unsigned int cpufreq_driver_resolve_freq(struct cpufreq_policy *policy,
                                         unsigned int target_freq);
```

بتجيب الـ frequency الفعلية اللي الـ driver هيضبطها لو طلبنا `target_freq`، من غير ما تعمل فعلاً أي تغيير. مفيدة جداً في الـ governors لمعرفة الـ actual frequency قبل القرار.

**Parameters:**
- `policy` — الـ policy.
- `target_freq` — الـ frequency المطلوب resolve-ha.

**Return:** الـ frequency الفعلية بالـ kHz.

**Key details:** بتستخدم `driver->resolve_freq()` لو موجودة، وإلا بتستخدم `cpufreq_frequency_table_target()`. بتعمل cache للنتيجة في `policy->cached_target_freq` و `policy->cached_resolved_idx`.

---

### المجموعة السادسة: Transition Notifications — إشعارات التحويل

الـ CPUFreq بيوفر notification chain لأي كود يحتاج يعرف لما بيتغير الـ frequency (مثلاً: cpufreq-stats، thermal subsystem، إلخ).

---

#### `cpufreq_freq_transition_begin`

```c
void cpufreq_freq_transition_begin(struct cpufreq_policy *policy,
                                   struct cpufreq_freqs *freqs);
```

بتعلن بداية frequency transition وبتبعت `CPUFREQ_PRECHANGE` notification لكل المشتركين.

**Parameters:**
- `policy` — الـ policy اللي هيتغير frequency-ha.
- `freqs` — struct فيه `old` (الـ frequency الحالية) و`new` (الـ frequency المطلوبة).

**Key details:**
- بتحجز `policy->transition_lock` spinlock.
- بتضبط `policy->transition_ongoing = true`.
- لو في transition تانية جارية (من task تانية)، بتنتظر على `policy->transition_wait`.
- **لازم يتبعها** `cpufreq_freq_transition_end` في نفس سياق التنفيذ.

---

#### `cpufreq_freq_transition_end`

```c
void cpufreq_freq_transition_end(struct cpufreq_policy *policy,
                                 struct cpufreq_freqs *freqs,
                                 int transition_failed);
```

بتعلن نهاية الـ transition وبتبعت `CPUFREQ_POSTCHANGE` notification.

**Parameters:**
- `policy` — الـ policy.
- `freqs` — نفس struct الـ `PRECHANGE`.
- `transition_failed` — `0` لو نجح الـ transition، أي قيمة تانية لو فشل.

**Key details:**
- لو نجح الـ transition، بتحدّث `policy->cur` للـ `freqs->new`.
- بتعمل `wake_up_all(&policy->transition_wait)` عشان تفك أي threads منتظرة.
- بتستدعي `cpufreq_stats_record_transition()` لتسجيل الـ transition في الـ stats.

---

#### `cpufreq_register_notifier` / `cpufreq_unregister_notifier`

```c
int cpufreq_register_notifier(struct notifier_block *nb, unsigned int list);
int cpufreq_unregister_notifier(struct notifier_block *nb, unsigned int list);
```

بيسجّلوا أو يشيلوا notifier block من واحدة من قائمتين:
- `CPUFREQ_TRANSITION_NOTIFIER` — للإشعار بـ frequency transitions.
- `CPUFREQ_POLICY_NOTIFIER` — للإشعار بإنشاء أو حذف الـ policies.

**Parameters:**
- `nb` — الـ notifier block مع الـ `.notifier_call` callback.
- `list` — نوع الـ notifier list.

**Return:** `0` على النجاح أو `errno` سالب.

**Who calls them:** الـ cpufreq-stats، thermal، وأي driver خارجي يحتاج يتابع تغييرات الـ frequency.

---

### المجموعة السابعة: Frequency Table Helpers

الـ scaling drivers غالباً بتوفر جدول frequencies محدود (discrete P-states). المجموعة دي بتسهّل التعامل مع هذه الجداول.

---

#### `cpufreq_frequency_table_cpuinfo`

```c
int cpufreq_frequency_table_cpuinfo(struct cpufreq_policy *policy);
```

بتمشي على الـ `policy->freq_table` وبتملأ `policy->cpuinfo.min_freq` و`policy->cpuinfo.max_freq` بأصغر وأكبر frequency valid موجودة في الجدول.

**Parameters:** `policy` — الـ policy اللي عندها `freq_table` محددة.

**Return:** `0` على النجاح، أو `-EINVAL` لو الـ table فاضية أو كل entries فيها `CPUFREQ_ENTRY_INVALID`.

**Who calls it:** الـ driver من داخل `->init()` callback بعد ما يضبط `policy->freq_table`.

---

#### `cpufreq_frequency_table_verify`

```c
int cpufreq_frequency_table_verify(struct cpufreq_policy_data *policy);
```

بتتحقق إن `policy->min` و`policy->max` موجودين فعلاً في الجدول. لو `policy->min` أكبر من أعلى frequency في الجدول، هيرجع `-EINVAL`.

**Who calls it:** كـ `->verify()` callback للـ driver بدلاً من كتابة verify خاصة.

---

#### `cpufreq_frequency_table_target`

```c
static inline int cpufreq_frequency_table_target(
    struct cpufreq_policy *policy,
    unsigned int target_freq,
    unsigned int min,
    unsigned int max,
    unsigned int relation);
```

قلب عملية اختيار الـ P-state. بتجيب index من `policy->freq_table` للـ frequency الأنسب بناءً على الـ `relation`.

**Parameters:**
- `target_freq` — الـ frequency المطلوبة.
- `min`, `max` — حدود البحث.
- `relation` — قاعدة الاختيار (L, H, C مع or بدون E).

**Return:** الـ index في الـ `freq_table`، أو `-EINVAL`.

**Key details:**
- لو الـ table مرتبة ascending، بتستخدم `find_index_al/ah/ac`.
- لو descending، بتستخدم `find_index_dl/dh/dc`.
- لو الـ `CPUFREQ_RELATION_E` set ومفيش efficient frequency في الحدود، بتعيد المحاولة بدون الـ efficiency filter.

**Pseudocode:**
```
cpufreq_frequency_table_target(policy, target, min, max, relation):
    efficiencies = policy->efficiencies_available && (relation & E)
    relation &= ~E

    if unsorted:
        return cpufreq_table_index_unsorted(...)

retry:
    switch relation:
        L -> idx = find_index_l(policy, target, min, max, efficiencies)
        H -> idx = find_index_h(policy, target, min, max, efficiencies)
        C -> idx = find_index_c(policy, target, min, max, efficiencies)

    if !in_limits(idx) && efficiencies:
        efficiencies = false
        goto retry

    return idx
```

---

#### `cpufreq_table_set_inefficient`

```c
static inline int cpufreq_table_set_inefficient(
    struct cpufreq_policy *policy,
    unsigned int frequency);
```

بتعلّم frequency معينة في الجدول بـ `CPUFREQ_INEFFICIENT_FREQ` flag، يعني الـ governors ممكن تتجنبها لو في alternative frequency أفضل energetically بنفس أو أكبر performance.

**Parameters:**
- `policy` — الـ policy.
- `frequency` — الـ frequency المراد تعليمها كـ inefficient.

**Return:** `0` على النجاح، `-EINVAL` لو الجدول unsorted أو الـ frequency مش موجودة.

**Key details:** بتضبط `policy->efficiencies_available = true` عند النجاح، وده بيفعّل الـ `CPUFREQ_RELATION_E` path في `cpufreq_frequency_table_target`.

**Who calls it:** الـ driver من `->init()` بعد ما يعرف أي P-states inefficient (مثلاً من OPP table أو ACPI data).

---

### المجموعة التامنة: Scheduler Interface — ربط الـ CPUFreq بالـ Scheduler

الـ link ده هو جوهر تصميم `schedutil` governor وكل governor حديث.

---

#### `cpufreq_add_update_util_hook`

```c
void cpufreq_add_update_util_hook(int cpu,
                                  struct update_util_data *data,
                                  void (*func)(struct update_util_data *data,
                                               u64 time,
                                               unsigned int flags));
```

بتسجّل callback في الـ scheduler على الـ `cpu` المحدد. الـ callback ده بيتستدعى من الـ scheduler على كل **utilization update event** (task enqueue/dequeue، scheduler tick، إلخ).

**Parameters:**
- `cpu` — الـ CPU اللي هيتسجّل عليه الـ hook.
- `data` — مؤشر للـ `update_util_data` struct (عادةً جزء من governor's per-cpu data).
- `func` — الـ callback نفسه:
  - `data` — نفس الـ data pointer.
  - `time` — الوقت الحالي بالـ nanoseconds (`ktime_get`).
  - `flags` — أعلام من الـ scheduler، أهمها `SCHED_CPUFREQ_IOWAIT` (بيشير إن task كانت waiting على I/O).

**Key details:**
- لازم تتستدعى من داخل `governor->start()` callback.
- لا يجوز تسجيل أكتر من hook واحد على نفس الـ CPU في نفس الوقت.
- الـ func بتشتغل من **scheduler context** (مع IRQ ممكن تكون disabled).

---

#### `cpufreq_remove_update_util_hook`

```c
void cpufreq_remove_update_util_hook(int cpu);
```

بتشيل الـ utilization update callback المسجّل على الـ `cpu`.

**Parameters:** `cpu` — الـ CPU المراد شيل الـ hook منه.

**Key details:** لازم تتستدعى من داخل `governor->stop()`. بعد الاستدعاء، لازم `synchronize_rcu()` عشان تضمن إن مفيش scheduled callback جارية.

---

#### `cpufreq_this_cpu_can_update`

```c
bool cpufreq_this_cpu_can_update(struct cpufreq_policy *policy);
```

بتتحقق إذا كان الـ CPU الـ current مسموح له يعمل frequency update للـ policy دي من scheduler context.

**Return:** `true` لو الـ CPU ده من ضمن الـ policy CPUs أو لو `dvfs_possible_from_any_cpu` مضبوطة.

**Who calls it:** الـ governor callbacks اللي بتشتغل من الـ scheduler عشان تتجنب race conditions في الـ shared policy scenarios.

---

#### `map_util_freq`

```c
static inline unsigned long map_util_freq(unsigned long util,
                                          unsigned long freq,
                                          unsigned long cap);
```

بتحوّل الـ utilization لـ target frequency بالمعادلة: `freq * util / cap`.

**Parameters:**
- `util` — الـ utilization المقاسة (من PELT).
- `freq` — الـ reference frequency (عادةً `policy->cpuinfo.max_freq`).
- `cap` — الـ maximum capacity.

**الاستخدام الأساسي:** الـ schedutil governor بيستخدمها لتحويل الـ PELT utilization signal لـ target kHz.

---

#### `map_util_perf`

```c
static inline unsigned long map_util_perf(unsigned long util);
```

بتطبّق الـ 25% headroom على الـ utilization: `util + (util >> 2)` أي `util * 1.25`.

ده بيعادل المعامل `1.25` الموجود في المعادلة الرسمية للـ schedutil:
```
f = 1.25 * f_0 * util / max
```

**Who calls it:** الـ schedutil governor قبل ما يمرر الـ util لـ `map_util_freq`.

---

### المجموعة التاسعة: Power Management & Boost

---

#### `cpufreq_suspend` / `cpufreq_resume`

```c
void cpufreq_suspend(void);
void cpufreq_resume(void);
```

**`cpufreq_suspend`:** بتعمل loop على كل الـ policies وبتوقّف الـ governor على كل واحدة، ثم بتستدعي `driver->suspend()` لو موجودة. كمان بتشيل الـ utilization update hooks من الـ scheduler.

**`cpufreq_resume`:** بتعيد تشغيل الـ governors وبتستدعي `driver->resume()`. لو الـ governor بيدعم الـ limits update، بتطلب منه يعيد تطبيق الـ limits.

**Who calls them:** `syscore_ops` من الـ PM infrastructure عند دخول وخروج الـ system suspend.

---

#### `cpufreq_boost_enabled`

```c
bool cpufreq_boost_enabled(void);
```

بترجع `true` لو الـ frequency boost مفعّل system-wide (يعني الـ `/sys/devices/system/cpu/cpufreq/boost` قيمتها 1).

**Who calls it:** الـ scaling drivers اللي بتدعم software-based boost قبل ما تطبّق الـ boost frequencies.

---

### المجموعة العاشرة: sysfs & Limits Management

---

#### `cpufreq_update_policy`

```c
void cpufreq_update_policy(unsigned int cpu);
```

بتعمل policy update كامل للـ CPU المحدد: بتستدعي `driver->verify()` و`governor->limits()`. مفيدة لما تتغير حدود الـ frequency من خارج الـ normal path (مثلاً من ACPI notification).

**Who calls it:** كود الـ ACPI عند تغيير platform limits.

---

#### `refresh_frequency_limits`

```c
void refresh_frequency_limits(struct cpufreq_policy *policy);
```

بتعيد حساب `policy->min` و`policy->max` من الـ QoS constraints (freq_qos_request objects) المرتبطة بالـ policy، وبتطبّقها على الـ governor.

**Who calls it:** الـ QoS notifier blocks (`policy->nb_min` و `policy->nb_max`) لما يتغير أي QoS constraint.

---

#### `cpufreq_enable_fast_switch` / `cpufreq_disable_fast_switch`

```c
void cpufreq_enable_fast_switch(struct cpufreq_policy *policy);
void cpufreq_disable_fast_switch(struct cpufreq_policy *policy);
```

بيفعّلوا أو بيعطّلوا الـ fast switch path على الـ policy. الـ governor بيستدعيهم من `->init()` و`->exit()` على التوالي، لكن بس لو `policy->fast_switch_possible` مضبوطة من الـ driver.

**Key details:**
- `enable`: بتزود `fast_switch_count` global counter.
- `disable`: بتنزّل الـ counter.
- لو الـ counter > 0، الـ fast switch path مفعّل وبيمنع load-balancing أو scheduler migrations تؤثر عليه.

---

#### `cpufreq_verify_within_limits`

```c
static inline void cpufreq_verify_within_limits(
    struct cpufreq_policy_data *policy,
    unsigned int min,
    unsigned int max);
```

بتعمل clamp لـ `policy->max` و`policy->min` بين الحدود المحددة باستخدام `clamp()`. الـ driver بيستخدمها في `->verify()` callback لضمان إن المستخدم مشطّش قيم خارج حدود الـ hardware.

**Who calls it:** الـ driver `->verify()` callback، وعادةً عبر `cpufreq_verify_within_cpu_limits` اللي بتمرر `cpuinfo.min_freq` و`cpuinfo.max_freq` تلقائياً.

---

#### `get_cpu_idle_time`

```c
u64 get_cpu_idle_time(unsigned int cpu, u64 *wall, int io_busy);
```

بتجيب الـ idle time للـ CPU من kernel idle statistics. بتستخدمها الـ governors اللي بتحسب الـ CPU load يدوياً (مثل `ondemand` و`conservative`).

**Parameters:**
- `cpu` — رقم الـ CPU.
- `wall` — pointer بيتملأ بالـ total wall time للـ CPU (كـ reference).
- `io_busy` — لو `1`، بيحسب الـ I/O wait time كـ busy time مش idle.

**Return:** الـ idle time بالـ microseconds.

**Key details:** الفرق بين `wall` والـ return value هو الـ active time. النسبة دي هي تقدير الـ load.

---

#### `cpufreq_scale`

```c
static inline unsigned long cpufreq_scale(unsigned long old,
                                          u_int div,
                                          u_int mult);
```

بتحسب `old * mult / div` بأمان على الـ 32-bit architectures عبر استخدام `u64` intermediate على 32-bit وعملية عادية على 64-bit.

**Who calls it:** الـ governors اللي بتحسب proportional frequencies.

---

### المجموعة الحادية عشر: Governor Attribute Set — إدارة sysfs للـ Governors

الـ governors اللي بيدعمون الـ per-policy tunables بتستخدم `gov_attr_set` لإدارة الـ sysfs kobjects المرتبطة بكل policy.

---

#### `gov_attr_set_init` / `gov_attr_set_get` / `gov_attr_set_put`

```c
void gov_attr_set_init(struct gov_attr_set *attr_set,
                       struct list_head *list_node);
void gov_attr_set_get(struct gov_attr_set *attr_set,
                      struct list_head *list_node);
unsigned int gov_attr_set_put(struct gov_attr_set *attr_set,
                              struct list_head *list_node);
```

- **`gov_attr_set_init`:** بتهيّئ الـ `gov_attr_set` لأول policy. بتعمل kobject_init وبتضيف الـ policy للـ list.
- **`gov_attr_set_get`:** بتستدعى لما policy جديدة بتشارك نفس الـ attr_set. بتزود `usage_count` وبتضيف الـ policy للـ list.
- **`gov_attr_set_put`:** بتستدعى عند إزالة policy من الـ attr_set. لو الـ `usage_count` وصل لصفر، بتعمل kobject_put وبتحرر الـ memory.

**Who calls them:** الـ governor `->init()` و`->exit()` callbacks في governors مثل `schedutil` و`ondemand`.

---

### جدول ملخّص الـ Callbacks في `struct cpufreq_driver`

| Callback | متى يتستدعى | إلزامي؟ |
|---|---|---|
| `->init(policy)` | عند إنشاء policy جديدة | نعم |
| `->verify(policy_data)` | عند كل تغيير لحدود الـ policy | نعم |
| `->target_index(policy, idx)` | لتغيير الـ frequency لـ table index | إما هذا أو `setpolicy` |
| `->setpolicy(policy)` | لتغيير policy كاملة (مثل intel_pstate) | إما هذا أو `target_index` |
| `->fast_switch(policy, freq)` | من scheduler context (fast path) | لا |
| `->adjust_perf(cpu, min, target, cap)` | من scheduler (performance hint) | لا |
| `->get(cpu)` | لقراءة الـ frequency الحالية من hardware | موصى به |
| `->get_intermediate(policy, idx)` | جيب intermediate frequency للتحويل | لا |
| `->target_intermediate(policy, idx)` | اضبط الـ intermediate frequency | لا (مع السابق فقط) |
| `->online(policy)` | عند عودة CPU للـ online | لا |
| `->offline(policy)` | عند ذهاب CPU للـ offline | لا |
| `->exit(policy)` | عند تدمير الـ policy | لا |
| `->suspend(policy)` | عند الـ system suspend | لا |
| `->resume(policy)` | عند الـ system resume | لا |
| `->ready(policy)` | بعد اكتمال تهيئة الـ policy | لا |
| `->set_boost(policy, state)` | لتفعيل/تعطيل الـ boost | لا (لو `boost_supported`) |
| `->register_em(policy)` | لتسجيل مع Energy Model | لا |
| `->update_limits(policy)` | عند تغيير الـ firmware limits | لا |
| `->bios_limit(cpu, limit)` | لقراءة الـ BIOS frequency limit | لا |

---

### جدول ملخّص الـ Callbacks في `struct cpufreq_governor`

| Callback | الغرض |
|---|---|
| `->init(policy)` | هيّئ البيانات الداخلية للـ governor، أضف sysfs attrs |
| `->exit(policy)` | حرر الموارد، أزل sysfs attrs |
| `->start(policy)` | سجّل utilization hooks مع الـ scheduler |
| `->stop(policy)` | أزل الـ utilization hooks |
| `->limits(policy)` | طبّق الـ policy->min/max الجديدة على الـ hardware |
| `->show_setspeed(policy, buf)` | لـ `userspace` governor فقط: اقرأ الـ target freq |
| `->store_setspeed(policy, freq)` | لـ `userspace` governor فقط: اضبط الـ target freq |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs المهمة

الـ CPUFreq بيكتب بيانات في `/sys/kernel/debug/` لما يكون `CONFIG_DEBUG_FS=y`:

```bash
# اقرأ tracing events الخاصة بـ cpufreq
ls /sys/kernel/debug/tracing/events/power/

# الملفات المفيدة:
# cpu_frequency        → كل تغيير في الـ frequency
# cpu_frequency_limits → تغيير حدود الـ min/max
# cpu_idle             → حالة الـ idle
```

```bash
# اقرأ الـ cpufreq stats عبر sysfs مش debugfs
# (الـ debugfs للـ cpufreq نفسه محدود — معظم البيانات في sysfs)
cat /sys/kernel/debug/tracing/events/power/cpu_frequency/format
```

---

#### 2. مدخلات الـ sysfs المهمة

كل الـ attributes دي تحت `/sys/devices/system/cpu/cpufreq/policyX/`:

| الملف | القراءة | الكتابة | الوصف |
|-------|---------|---------|-------|
| `scaling_governor` | اسم الـ governor الحالي | اسم governor جديد | غير الـ governor |
| `scaling_cur_freq` | التردد الحالي (kHz) | — | آخر P-state طلبه الـ driver |
| `cpuinfo_cur_freq` | التردد الفعلي من الـ HW (kHz) | — | ما يقوله الـ hardware فعلاً |
| `cpuinfo_max_freq` | أقصى تردد ممكن (kHz) | — | حد الـ hardware |
| `cpuinfo_min_freq` | أدنى تردد ممكن (kHz) | — | حد الـ hardware |
| `scaling_max_freq` | الحد العلوي المسموح (kHz) | قيمة جديدة | يمكن تعديله |
| `scaling_min_freq` | الحد السفلي المسموح (kHz) | قيمة جديدة | يمكن تعديله |
| `scaling_available_governors` | قائمة الـ governors المتاحة | — | للمعرفة فقط |
| `scaling_available_frequencies` | قائمة الترددات المتاحة (kHz) | — | إن كانت discrete |
| `cpuinfo_transition_latency` | زمن التبديل بين P-states (ns) | — | مهم للـ governor tuning |
| `scaling_driver` | اسم الـ driver المستخدم | — | للتعرف على الـ driver |
| `affected_cpus` | الـ CPUs الـ online في هذه الـ policy | — | |
| `related_cpus` | كل الـ CPUs (online + offline) | — | |
| `bios_limit` | حد الـ BIOS للتردد (kHz) | — | موجود فقط إن دعمه الـ driver |

```bash
# اطبع حالة كل الـ policies دفعة واحدة
for policy in /sys/devices/system/cpu/cpufreq/policy*/; do
    echo "=== $policy ==="
    for f in scaling_governor scaling_cur_freq cpuinfo_cur_freq \
              scaling_max_freq scaling_min_freq cpuinfo_max_freq; do
        printf "  %-35s = %s\n" "$f" "$(cat $policy$f 2>/dev/null || echo N/A)"
    done
done
```

**الـ boost control:**

```bash
# قراءة حالة الـ boost (0=معطل، 1=مفعّل)
cat /sys/devices/system/cpu/cpufreq/boost

# تعطيل الـ boost
echo 0 > /sys/devices/system/cpu/cpufreq/boost

# إعادة التفعيل
echo 1 > /sys/devices/system/cpu/cpufreq/boost
```

**الـ stats (إن كان `CONFIG_CPU_FREQ_STAT=y`):**

```bash
# إحصائيات الوقت الذي أمضاه كل CPU في كل تردد
cat /sys/devices/system/cpu/cpufreq/policy0/stats/time_in_state
cat /sys/devices/system/cpu/cpufreq/policy0/stats/total_trans
cat /sys/devices/system/cpu/cpufreq/policy0/stats/trans_table
```

---

#### 3. الـ ftrace: Tracepoints والـ Events

الـ CPUFreq بيستخدم الـ `power` tracing subsystem:

```bash
# فعّل الـ tracing للـ cpu_frequency events
cd /sys/kernel/debug/tracing

# شوف الـ events المتاحة
ls events/power/ | grep cpu

# فعّل تسجيل تغييرات الـ frequency
echo 1 > events/power/cpu_frequency/enable
echo 1 > events/power/cpu_frequency_limits/enable

# فعّل أيضاً الـ cpu_idle عشان تشوف العلاقة
echo 1 > events/power/cpu_idle/enable

# ابدأ الـ tracing
echo 1 > tracing_on

# اشتغّل شوية ثم اقرأ النتائج
cat trace | head -50

# أو استخدم trace_pipe للـ real-time
cat trace_pipe &
# ... run workload ...
kill %1
```

**مثال على الـ output:**

```
     kworker/0:2-123   [000] d..2  1234.567890: cpu_frequency: state=1200000 cpu_id=0
     kworker/0:2-123   [000] d..2  1234.568100: cpu_frequency: state=2400000 cpu_id=0
```

الـ `state` هنا هو التردد بالـ kHz.

**استخدام الـ function tracer للـ governor:**

```bash
# تتبع دوال الـ cpufreq core
echo function > current_tracer
echo 'cpufreq_*' > set_ftrace_filter
echo 1 > tracing_on
cat trace | grep cpufreq | head -30
```

**استخدام الـ perf لتتبع الـ frequency:**

```bash
# record frequency changes with perf
perf record -e power:cpu_frequency -a sleep 10
perf script
```

---

#### 4. الـ printk والـ Dynamic Debug

**تفعيل الـ dynamic debug للـ cpufreq:**

```bash
# تفعيل كل رسائل الـ debug في الـ cpufreq core
echo 'file drivers/cpufreq/cpufreq.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل الـ governor بتاعك (مثلاً schedutil)
echo 'file kernel/sched/cpufreq_schedutil.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل الـ ondemand governor
echo 'file drivers/cpufreq/cpufreq_ondemand.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل ملفات الـ cpufreq دفعة واحدة
echo 'module cpufreq +p' > /sys/kernel/debug/dynamic_debug/control

# إيقاف الـ debug messages
echo 'file drivers/cpufreq/cpufreq.c -p' > /sys/kernel/debug/dynamic_debug/control
```

**قراءة الـ kernel log بعد التفعيل:**

```bash
dmesg -w | grep -i cpufreq
# أو
journalctl -k -f | grep -i cpufreq
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ Config Option | الوصف |
|-------------------|-------|
| `CONFIG_CPU_FREQ` | الـ CPUFreq core نفسه (لازم يكون مفعّل) |
| `CONFIG_CPU_FREQ_STAT` | إحصائيات الترددات في sysfs |
| `CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE` | governor افتراضي للاختبار |
| `CONFIG_CPU_FREQ_GOV_SCHEDUTIL` | تفعيل schedutil governor |
| `CONFIG_CPU_FREQ_GOV_ONDEMAND` | تفعيل ondemand governor |
| `CONFIG_CPU_FREQ_GOV_CONSERVATIVE` | تفعيل conservative governor |
| `CONFIG_CPU_FREQ_GOV_USERSPACE` | للتحكم اليدوي في الاختبار |
| `CONFIG_DEBUG_FS` | لازم يكون مفعّل لرؤية /sys/kernel/debug |
| `CONFIG_TRACING` | لتفعيل الـ ftrace |
| `CONFIG_PM_DEBUG` | رسائل debug للـ power management |
| `CONFIG_PM_ADVANCED_DEBUG` | معلومات تفصيلية إضافية في sysfs |
| `CONFIG_THERMAL_GOV_STEP_WISE` | مهم لفهم العلاقة بين thermal و cpufreq |
| `CONFIG_X86_ACPI_CPUFREQ_CPB` | دعم AMD Core Performance Boost على x86 |
| `CONFIG_CPUFREQ_ARCH_CUR_FREQ` | الحصول على التردد الحقيقي من الـ arch |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CPU_FREQ|CPUFREQ' | grep -v '^#'
# أو
grep -E 'CPU_FREQ|CPUFREQ' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**cpupower (الأداة الرئيسية):**

```bash
# معلومات شاملة عن كل CPUs
cpupower frequency-info

# عرض التردد الحالي لكل CPU
cpupower -c all frequency-info | grep 'current CPU'

# تغيير الـ governor لكل CPUs
cpupower -c all frequency-set -g performance

# تحديد تردد ثابت (يتطلب userspace governor)
cpupower frequency-set -f 2400MHz

# عرض إحصائيات الـ idle
cpupower -c all idle-info
```

**turbostat (لـ Intel/AMD x86):**

```bash
# مراقبة الترددات الفعلية والـ P-states
turbostat --interval 1

# عرض فقط الـ frequency columns
turbostat --show Busy%,Avg_MHz,Bzy_MHz,TSC_MHz

# تشغيل أمر مع المراقبة
turbostat -- sleep 10
```

**perf stat للـ frequency scaling:**

```bash
# مراقبة الـ IPC وعلاقتها بالتردد
perf stat -e cycles,instructions,cpu-migrations sleep 5

# مراقبة الـ power events
perf stat -e power/energy-pkg/,power/energy-cores/ sleep 5
```

**i7z (للـ Intel):**

```bash
# عرض تفصيلي للـ C-states وـ P-states
i7z
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|----------------------|--------|-------|
| `cpufreq: __cpufreq_driver_target: cpu: X, target: Y, policy min: Z` | الـ target frequency خارج نطاق الـ policy | تحقق من `scaling_min_freq` و`scaling_max_freq` |
| `Failed to change cpufreq: -22` | `EINVAL` — قيمة غير صالحة للتردد | تأكد أن التردد المطلوب ضمن `scaling_available_frequencies` |
| `cpufreq: transition timeout!` | الـ driver تجاوز وقت التبديل بين الترددات | مشكلة في الـ hardware أو الـ driver، راجع `transition_ongoing` |
| `cpufreq: governor failed, falling back` | الـ governor فشل في الـ init أو الـ start | تحقق من `dmesg` لمعرفة الـ governor المشكوك فيه |
| `cpufreq: No governor specified` | لا يوجد governor محدد عند boot | أضف `cpufreq.default_governor=schedutil` لـ cmdline |
| `cpufreq: Driver not registering` | الـ scaling driver فشل في التسجيل | تحقق من الـ ACPI/BIOS أو kernel module loading |
| `acpi-cpufreq: Error getting cpufreq from BIOS` | ACPI _PSS/_PPC غير متاح أو خاطئ | راجع ACPI tables بـ `acpidump` |
| `cpufreq: bios_limit: limiting frequency` | الـ BIOS يفرض حد على التردد | راجع `bios_limit` في sysfs — قد تحتاج تعديل BIOS |
| `CPUFREQ: Transition rate limit exceeded` | استدعاءات تغيير التردد أسرع من المسموح | زيادة `rate_limit_us` للـ schedutil |

---

#### 8. نقاط وضع الـ dump_stack() والـ WARN_ON()

أماكن استراتيجية في الـ kernel لإضافة assertions عند تطوير أو debug الـ subsystem:

```c
/* في cpufreq_set_policy() — تحقق أن الـ policy limits منطقية */
WARN_ON(policy->min > policy->max);

/* في cpufreq_driver_target() — تحقق أن الـ cpu online */
WARN_ON(!cpu_online(policy->cpu));

/* في governor's ->start() callback — تحقق من الـ init */
WARN_ON(!policy->governor);

/* في الـ frequency transition callback */
WARN_ON(policy->transition_ongoing &&
        policy->transition_task != current);

/* لتتبع من يستدعي تغيير التردد في سياق غير متوقع */
if (unlikely(in_interrupt())) {
    dump_stack();
    pr_err("cpufreq: freq change from interrupt context!\n");
}
```

---

### Hardware Level

#### 1. التحقق أن حالة الـ Hardware تطابق حالة الـ Kernel

```bash
# قارن التردد المُبلَّغ عنه من الـ kernel
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq

# مع التردد الفعلي من الـ hardware
cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_cur_freq

# استخدم turbostat للتردد الفعلي المبني على MSR
turbostat --show Avg_MHz,Bzy_MHz -i 1

# على ARM — تحقق من AMU (Activity Monitor Unit) إن كان متاحاً
cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_avg_freq
```

**قارن P-states المتاحة:**

```bash
# من الـ kernel
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies

# من ACPI (المصدر الأصلي)
cat /proc/acpi/processor/CPU0/throttling 2>/dev/null
# أو
acpidump -n _PSS | acpixtract -a  # يحتاج acpica-tools
```

---

#### 2. Register Dump Techniques

**على x86 — قراءة MSR registers:**

```bash
# المتطلبات
modprobe msr

# قراءة MSR_IA32_PERF_CTL (0x199) — التردد المطلوب حالياً
rdmsr -a 0x199

# قراءة MSR_IA32_PERF_STATUS (0x198) — التردد الفعلي
rdmsr -a 0x198

# قراءة MSR_PLATFORM_INFO (0xCE) — الحدود الأساسية للتردد
rdmsr 0xCE

# HWP request (إن كان مفعّل — Intel Skylake+)
rdmsr -a 0x774  # HWP_REQUEST
rdmsr -a 0x771  # HWP_CAPABILITIES
rdmsr -a 0x773  # HWP_STATUS
```

**تفسير MSR_IA32_PERF_STATUS (0x198):**

```
bits [15:8] = Current P-state ratio
التردد الفعلي = ratio × bus_clock (عادة 100 MHz)

مثال: القيمة 0x1800 → ratio = 0x18 = 24 → 24 × 100 = 2400 MHz
```

**على ARM — قراءة CPUFREQ registers عبر devmem:**

```bash
# قراءة سجلات من عنوان فيزيائي (مثلاً DVFS controller)
devmem2 0xFD000000 w  # عنوان مثال — يختلف حسب الـ SoC

# أو عبر /dev/mem
dd if=/dev/mem bs=4 count=1 skip=$((0xFD000000/4)) | xxd
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**نقاط القياس:**

```
┌─────────────────────────────────────────────────────────┐
│  نقاط مثالية للـ logic analyzer / oscilloscope          │
│                                                          │
│  1. VID pins (Voltage ID) بين الـ CPU والـ VRM          │
│     → تشوف متى وبكام بيتغير الـ voltage                 │
│                                                          │
│  2. I2C/SPI bus للـ PMIC (Power Management IC)          │
│     → تتبع أوامر تغيير الـ voltage                      │
│                                                          │
│  3. CLK output من الـ PLL                               │
│     → قياس التردد الفعلي مباشرة                         │
│                                                          │
│  4. SVID bus (Intel) أو SVI2/SVI3 bus (AMD)             │
│     → بروتوكول تغيير الـ voltage/frequency              │
└─────────────────────────────────────────────────────────┘
```

**إعدادات الـ logic analyzer:**

- Sample rate: على الأقل 10x أعلى تردد متوقع للـ bus
- Protocol decode: I2C, SPI, أو SVID حسب الـ hardware
- Trigger: على أول تغيير في VID pins
- قارن timestamp الـ logic analyzer مع الـ ftrace timestamps

---

#### 4. مشاكل الـ Hardware الشائعة وـ Kernel Log Patterns

| المشكلة | نمط في الـ Kernel Log | التشخيص |
|---------|----------------------|---------|
| الـ VRM لا يستجيب لتغيير الـ voltage | `cpufreq: transition timeout` متكرر | قياس VID pins بالـ oscilloscope |
| الـ BIOS يقيّد التردد | `bios_limit: X kHz` في dmesg | قرأ `bios_limit` في sysfs، راجع BIOS settings |
| ACPI _PSS table خاطئ | `acpi-cpufreq: Failed` أو ترددات غريبة | `acpidump` + `acpixtract` + `iasl -d` |
| تداخل thermal throttling | التردد ينخفض فجأة بدون سبب واضح | `cat /sys/class/thermal/*/temp` + دمجه مع ftrace |
| P-states لا تُطبّق بشكل صحيح | الفرق كبير بين `scaling_cur_freq` و`cpuinfo_cur_freq` | راجع MSR 0x198 مباشرة |
| CPU clock unstable | رسائل `clocksource` خطأ أو drift عالي | `dmesg | grep clocksource` |

---

#### 5. Device Tree Debugging (للـ ARM/Embedded)

**التحقق من الـ DT يطابق الـ hardware:**

```bash
# شوف الـ OPP table المُحمّلة من الـ DT
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies

# قارن مع الـ DT المُجمَّع
# استخرج الـ DTB المحمّل حالياً
dtc -I fs /proc/device-tree > /tmp/current.dts 2>/dev/null
grep -A 20 'opp-table' /tmp/current.dts

# أو مباشرة من sysfs
find /sys/firmware/devicetree/base -name "opp-hz" | while read f; do
    echo "$f: $(cat $f | xxd | head -2)"
done
```

**فحص الـ clk hierarchy:**

```bash
# شوف شجرة الـ clock
cat /sys/kernel/debug/clk/clk_summary | grep -i cpu

# أو
cat /sys/kernel/debug/clk/*/clk_rate | head -20
```

**التحقق من الـ regulator (للـ DVFS):**

```bash
# شوف الـ voltage regulator المرتبط بالـ CPU
cat /sys/kernel/debug/regulator/regulator_summary 2>/dev/null | grep -i cpu

# أو بالـ sysfs
ls /sys/class/regulator/
cat /sys/class/regulator/regulator.*/name
cat /sys/class/regulator/regulator.*/microvolts
```

**مشاكل شائعة في الـ DT:**

```
المشكلة: الـ driver يعرف ترددات أقل من المتوقع
السبب: opp-table في الـ DT ناقصة أو محدودة بـ opp-supported-hw
الحل: `grep -r opp-supported-hw /proc/device-tree` وقارن مع chip version

المشكلة: CPUFREQ لا يعمل خالص على ARM
السبب: compatible string غلط أو missing clocks/regulators في DT
الحل: dmesg | grep -E 'cpufreq|opp|dvfs|clk|regulator'
```

---

### Practical Commands

#### 1. أوامر جاهزة للنسخ

**الحالة الكاملة للـ CPUFreq:**

```bash
#!/bin/bash
# cpufreq-status.sh — snapshot شامل

echo "===== CPUFREQ STATUS ====="
echo "Kernel: $(uname -r)"
echo "Date: $(date)"
echo ""

echo "--- Driver & Boost ---"
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_driver
cat /sys/devices/system/cpu/cpufreq/boost 2>/dev/null && echo "(boost file exists)" || echo "(no boost file)"

echo ""
echo "--- Per-Policy Info ---"
for policy in /sys/devices/system/cpu/cpufreq/policy*/; do
    name=$(basename $policy)
    gov=$(cat $policy/scaling_governor 2>/dev/null)
    cur=$(cat $policy/scaling_cur_freq 2>/dev/null)
    hw_cur=$(cat $policy/cpuinfo_cur_freq 2>/dev/null)
    min=$(cat $policy/scaling_min_freq 2>/dev/null)
    max=$(cat $policy/scaling_max_freq 2>/dev/null)
    cpus=$(cat $policy/affected_cpus 2>/dev/null)
    echo "[$name] CPUs=$cpus governor=$gov cur=${cur}kHz hw_cur=${hw_cur}kHz range=[${min}-${max}]kHz"
done

echo ""
echo "--- Stats (if available) ---"
for policy in /sys/devices/system/cpu/cpufreq/policy*/; do
    if [ -f $policy/stats/time_in_state ]; then
        echo "$(basename $policy) time_in_state:"
        cat $policy/stats/time_in_state
    fi
done
```

**تفعيل الـ ftrace monitoring:**

```bash
#!/bin/bash
# cpufreq-trace.sh — سجّل تغييرات التردد لمدة N ثانية

SECONDS_TO_TRACE=${1:-10}
TRACE_DIR=/sys/kernel/debug/tracing

echo 0 > $TRACE_DIR/tracing_on
echo > $TRACE_DIR/trace

# فعّل الـ events
echo 1 > $TRACE_DIR/events/power/cpu_frequency/enable
echo 1 > $TRACE_DIR/events/power/cpu_frequency_limits/enable

echo 1 > $TRACE_DIR/tracing_on
echo "Tracing for $SECONDS_TO_TRACE seconds..."
sleep $SECONDS_TO_TRACE
echo 0 > $TRACE_DIR/tracing_on

echo "=== Frequency Changes ==="
cat $TRACE_DIR/trace | grep cpu_frequency | head -100

echo ""
echo "=== Summary ==="
cat $TRACE_DIR/trace | grep cpu_frequency | awk '{print $NF}' | sort | uniq -c | sort -rn

# Cleanup
echo 0 > $TRACE_DIR/events/power/cpu_frequency/enable
echo 0 > $TRACE_DIR/events/power/cpu_frequency_limits/enable
```

**مقارنة scaling_cur_freq مع cpuinfo_cur_freq:**

```bash
#!/bin/bash
# freq-drift-check.sh — اكشف الفرق بين الـ requested وـ actual frequency

while true; do
    for policy in /sys/devices/system/cpu/cpufreq/policy*/; do
        req=$(cat $policy/scaling_cur_freq 2>/dev/null)
        act=$(cat $policy/cpuinfo_cur_freq 2>/dev/null)
        if [ -n "$req" ] && [ -n "$act" ]; then
            diff=$((act - req))
            pct=$((diff * 100 / req))
            printf "$(basename $policy): requested=%dkHz actual=%dkHz diff=%+d%%\n" $req $act $pct
        fi
    done
    sleep 1
    echo "---"
done
```

**تغيير الـ governor وقياس الأثر:**

```bash
# حفظ الـ governor الحالي
OLD_GOV=$(cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor)

# التبديل إلى performance للاختبار
cpupower -c all frequency-set -g performance

# تشغيل workload وقياس الأداء
time your_benchmark_command

# الرجوع للـ governor الأصلي
cpupower -c all frequency-set -g $OLD_GOV
```

---

#### 2. تفسير الـ Output

**مثال على output من `cpupower frequency-info`:**

```
analyzing CPU 0:
  driver: acpi-cpufreq
  CPUs which run at the same hardware frequency: 0
  CPUs which need to have their frequency coordinated by software: 0
  maximum transition latency: 10.0 us
  hardware limits: 400 MHz - 3.60 GHz
  available frequency steps:  3.60 GHz, 3.40 GHz, 3.20 GHz, ...
  available cpufreq governors: conservative ondemand userspace powersave performance schedutil
  current policy: frequency should be within 400 MHz and 3.60 GHz.
                  The governor "schedutil" may decide which speed to use
                  within this policy.
  current CPU frequency: 1.20 GHz (asserted by call to hardware)
  cpufreq stats: 3.60 GHz:0.09%, 3.40 GHz:0.12%, ...  (407)
```

**كيف تقرأ الـ output:**
- **driver: acpi-cpufreq** → الـ driver المستخدم — لو كان `intel_pstate` الـ governors لا تنطبق بنفس الطريقة
- **maximum transition latency** → مهم للـ `schedutil rate_limit_us` — اضبطه على 1.5x هذه القيمة
- **hardware limits** → الحدود الفيزيائية — لا يمكن تجاوزها
- **current CPU frequency (asserted by call to hardware)** → التردد الفعلي، مش المطلوب

**مثال على output من `turbostat`:**

```
Package Core CPU Avg_MHz Busy%  Bzy_MHz TSC_MHz IRQ  POLL%
-       -    -   1247    34.6   3601    3600    2345  0.00
0       0    0   2100    58.3   3601    3600     890  0.00
0       0    4   800     22.1   3600    3600     432  0.00
```

- **Avg_MHz** → متوسط التردد (مع حساب الـ idle)
- **Busy_MHz** → التردد لما الـ CPU يكون busy فعلاً
- **TSC_MHz** → تردد الـ TSC = الـ base clock الاسمي
- لو `Bzy_MHz > TSC_MHz` → الـ Turbo Boost شتغال
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ `schedutil` بيسبب latency spikes في Modbus polling

#### العنوان
الـ CPUFreq governor غلط بيخلي الـ real-time Modbus polling يتأخر في industrial gateway

#### السياق
شركة بتعمل industrial IoT gateway بالـ RK3562 (Cortex-A53 quad-core). الجهاز بيشتغل Linux 6.6 وبيعمل Modbus RTU polling على UART كل 10ms لـ PLC خارجي. الـ gateway بيستخدم `schedutil` كـ default governor لأنه الـ distro default.

#### المشكلة
المهندسون لاحظوا إن الـ Modbus responses بتتأخر أحياناً من 10ms لـ 40-80ms بشكل عشوائي. الـ PLC بيعتبرها timeout وبيقطع الاتصال. المشكلة مش ثابتة — بتحصل بس لما الـ CPU يكون idle لفترة وبعدين يتحمّل فجأة.

#### التحليل
الـ `schedutil` governor بيشتغل بالمعادلة:
```
f = 1.25 * f_0 * util / max
```
لما الـ CPU يكون idle، الـ governor بيحط التردد على minimum (مثلاً 408 MHz على RK3562). لما الـ Modbus task يصحّى، في latency لحد ما الـ P-state يتغير للأعلى — وده الـ `cpuinfo_transition_latency`. على RK3562 ده ممكن يوصل لـ 200-500µs بس في الـ `schedutil` الـ `rate_limit_us` بيمنع التغيير المتكرر:

```
rate_limit_us = 1.5 * transition_latency  (default)
```

يعني لو الـ transition latency = 300µs، الـ `rate_limit_us` = 450µs. في الـ real-time context، الـ CPU بيفضل شغال بأقل تردد لحظات كافية تعمل missed deadlines.

```bash
# شوف الـ current governor والـ transition latency
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
# schedutil

cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_transition_latency
# 300000  (300µs in nanoseconds)

cat /sys/devices/system/cpu/cpufreq/policy0/schedutil/rate_limit_us
# 450
```

المشكلة إن الـ task wake-up + frequency ramp-up + UART interrupt handling = latency أكبر من الـ polling tolerance.

#### الحل

**Option 1 — غيّر الـ governor لـ `performance`:**
```bash
echo performance > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```
ده بيثبّت التردد على `scaling_max_freq` دايماً — مفيش P-state transitions، مفيش latency من الـ frequency scaling.

**Option 2 — لو محتاج power saving في وقت idle، استخدم `ondemand` مع aggressive settings:**
```bash
echo ondemand > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
echo 95 > /sys/devices/system/cpu/cpufreq/policy0/ondemand/up_threshold
echo 10000 > /sys/devices/system/cpu/cpufreq/policy0/ondemand/sampling_rate
```

**Option 3 — في `/etc/rc.local` أو systemd unit لضمان الـ setting بعد كل boot:**
```bash
#!/bin/bash
for policy in /sys/devices/system/cpu/cpufreq/policy*; do
    echo performance > "$policy/scaling_governor"
done
```

**للتحقق من الـ fix:**
```bash
# قيس الـ actual CPU frequency أثناء الـ polling
watch -n 0.1 cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_cur_freq
```

#### الدرس المستفاد
الـ `schedutil` ممتاز لـ general-purpose systems لكن في الـ real-time industrial workloads، الـ frequency transition latency نفسها تبقى مشكلة. الـ `performance` governor هو الصح لأي تطبيق محتاج deterministic response time. الـ `cpuinfo_transition_latency` في sysfs هو الـ indicator الأول تشوفه.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ thermal throttling بيوقف الـ boost

#### العنوان
الـ video playback بيتقطع على Android TV box لأن الـ `boost` اتعطل من الـ thermal governor

#### السياق
Android TV box رخيص بالـ Allwinner H616 (Cortex-A53 quad-core @ 1.512 GHz). بيشغّل 4K HEVC decode. الـ user شايف stuttering كل 30-60 ثانية بالضبط، خصوصاً لما الـ ambient temperature عالي.

#### المشكلة
الـ decoder بيطلب burst performance. الـ `boost` file موجود (الـ H616 driver بيدعمه) وبيكون مفعّل. لكن الـ thermal daemon بيكتب `0` في الـ `boost` file لما temperature تعدي threshold. وبعدين مش بيرجّعه. النتيجة: الـ frequency بتتقيد على `scaling_max_freq` أقل من القيمة المتوقعة، والـ decoder يفضل يطلب CPUs أقل من اللي يقدر يديها.

#### التحليل
```bash
# شوف الـ boost status
cat /sys/devices/system/cpu/cpufreq/boost
# 0  ← الـ thermal daemon كتب ده

# شوف الـ max freq المسموح بيه
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq
# 1008000  ← محدود بسبب thermal

cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_max_freq
# 1512000  ← الـ hardware maximum الحقيقي
```

الفرق بين `cpuinfo_max_freq` و`scaling_max_freq` هو الـ clue. الـ `scaling_max_freq` مش بس عن الـ boost — الـ thermal governor ممكن يعمل `scaling_max_freq` أقل من الـ hardware maximum حتى من غير `boost=0`.

الـ `boost` file بيتحكم في الـ permission للـ frequencies فوق الـ sustainable threshold. لما بيبقى `0`:

```
cpuinfo_max_freq = 1512 MHz (hardware absolute max)
scaling_max_freq = capped to non-boost range (e.g., 1008 MHz)
```

الـ video decoder بيتأثر لأن الـ P-state selection algorithm (الـ governor) بيشوف `scaling_max_freq` كـ ceiling.

#### الحل

**Debug أولاً:**
```bash
# شوف الـ thermal zones
cat /sys/class/thermal/thermal_zone*/temp
# 78000 (78°C) ← فوق الـ throttle threshold

# شوف إيه اللي بيغيّر الـ boost
grep -r "boost" /etc/thermal* /etc/android/thermal* 2>/dev/null
```

**Fix للـ hardware:**
1. تحسين الـ cooling — thermal paste أحسن، heatsink أكبر
2. تعديل الـ thermal trip points في الـ Device Tree:

```dts
/* في الـ H616 DT */
cpu_thermal: cpu-thermal {
    polling-delay-passive = <100>;
    polling-delay = <1000>;
    trips {
        cpu_hot: cpu-hot {
            temperature = <85000>;  /* رفعناه من 75000 */
            hysteresis = <5000>;
            type = "passive";
        };
        cpu_crit: cpu-crit {
            temperature = <110000>;
            hysteresis = <5000>;
            type = "critical";
        };
    };
};
```

**Fix مؤقت للـ test:**
```bash
# أعد تفعيل الـ boost يدوياً (مش recommended في production)
echo 1 > /sys/devices/system/cpu/cpufreq/boost

# أو ضع ceiling أعلى
echo 1512000 > /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq
```

#### الدرس المستفاد
الـ `boost` file مش بس عن الـ performance — هو السيف الأول اللي الـ thermal management بيستخدمه. الفرق بين `cpuinfo_max_freq` و`scaling_max_freq` و`cpuinfo_cur_freq` بيقولك بالضبط فين التقييد. دايماً ارسم الـ complete picture: hardware ceiling → boost gate → thermal cap → governor selection → actual frequency.

---

### السيناريو 3: Custom Board Bring-up على STM32MP1 — الـ CPUFreq مش بيشتغل خالص

#### العنوان
الـ CPUFreq لا بيظهر في sysfs ولا بيشتغل على custom STM32MP157 board جديد

#### السياق
مهندس Embedded بيعمل bring-up لـ custom board بالـ STM32MP157 (Cortex-A7 dual-core). الـ board جاهز، الـ kernel بيبوت، لكن مفيش `/sys/devices/system/cpu/cpufreq/` خالص. الـ CPU شغال بتردد واحد ثابت.

#### المشكلة
الـ CPUFreq core محتاج:
1. scaling driver مسجّل
2. الـ driver بيعرف يـ initialize الـ policy للـ CPU
3. الـ DT/ACPI بيوصف الـ OPPs (Operating Performance Points)

بدون الـ OPP table، الـ driver مش هيعرف الـ available P-states وهيفشل في الـ `->init()` callback.

#### التحليل
```bash
# شوف الـ kernel messages
dmesg | grep -i cpufreq
# cpufreq: driver registration failed: -22

dmesg | grep -i opp
# opp: _opp_add: device_node not found

# شوف لو في policy objects
ls /sys/devices/system/cpu/cpufreq/
# ls: cannot access '/sys/devices/system/cpu/cpufreq/': No such file or directory
```

الـ STM32MP1 بيستخدم `stm32mp1-cpufreq` driver اللي بيعتمد على `dev_pm_opp` framework. الـ OPPs لازم تكون في الـ Device Tree تحت الـ CPU node.

الـ Device Tree الخاطئ:
```dts
cpus {
    cpu0: cpu@0 {
        compatible = "arm,cortex-a7";
        device_type = "cpu";
        /* OPP table مش موجودة هنا! */
    };
};
```

الـ Device Tree الصح:
```dts
cpus {
    cpu0: cpu@0 {
        compatible = "arm,cortex-a7";
        device_type = "cpu";
        operating-points-v2 = <&cpu0_opp_table>;
        clocks = <&rcc CK_MPU>;
    };
};

cpu0_opp_table: opp-table {
    compatible = "operating-points-v2";
    opp-shared;

    opp-650000000 {
        opp-hz = /bits/ 64 <650000000>;
        opp-microvolt = <1200000>;
    };
    opp-800000000 {
        opp-hz = /bits/ 64 <800000000>;
        opp-microvolt = <1350000>;
    };
    opp-850000000 {
        opp-hz = /bits/ 64 <850000000>;
        opp-microvolt = <1350000>;
    };
};
```

#### الحل
1. أضف الـ OPP table للـ DT
2. تأكد إن الـ `CONFIG_CPU_FREQ` و`CONFIG_CPU_FREQ_STAT` و`CONFIG_ARM_STM32_CPUFREQ` enabled في الـ kernel config

```bash
# بعد fix الـ DT وإعادة البناء
grep -i cpufreq /boot/config-$(uname -r)
# CONFIG_CPU_FREQ=y
# CONFIG_ARM_STM32_CPUFREQ=y

# بعد البوت تاني
ls /sys/devices/system/cpu/cpufreq/policy0/
# affected_cpus  cpuinfo_max_freq  cpuinfo_min_freq  scaling_governor  ...

cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies
# 650000 800000 850000
```

**Debug إضافي:**
```bash
# شوف لو الـ driver اتسجّل
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_driver
# stm32mp1-cpufreq
```

#### الدرس المستفاد
الـ CPUFreq core محتاج ثلاثة أشياء: driver + policy + hardware description (OPPs). غياب أي منهم بيمنع إنشاء الـ `policyX` directories في sysfs. الـ `dmesg | grep -i cpufreq` و`dmesg | grep -i opp` هما أول خطوة في أي bring-up. الـ `cpuinfo_min_freq` و`cpuinfo_max_freq` بيتملّوا من الـ OPP table مباشرة.

---

### السيناريو 4: IoT Sensor Node على AM62x — الـ `conservative` governor بيصرف بطارية أكتر من المتوقع

#### العنوان
الـ battery life على AM62x IoT node أقل بـ 40% من الـ theoretical estimate رغم استخدام power-saving governor

#### السياق
شركة بتصنع IoT environmental sensor بالـ TI AM62x (Cortex-A53). الجهاز بيشتغل على بطارية وبيرسل sensor readings كل 30 ثانية عبر LoRa. المهندس اختار `conservative` governor ظناً إنه الأمثل للـ battery.

#### المشكلة
الـ `conservative` governor بيغيّر التردد في steps صغيرة (`freq_step` = 5% by default). المشكلة إن الـ sampling rate الافتراضي بيخلي الـ governor يشتغل كتير، وكل run بيرفع التردد خطوة. في الـ bursty workload (نوم طويل + بث سريع)، الـ governor مش بيوصل للـ min frequency بسرعة كافية، والـ CPU بيفضل في intermediate frequencies بين الـ bursts.

#### التحليل
```bash
# شوف الـ current settings
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
# conservative

cat /sys/devices/system/cpu/cpufreq/policy0/conservative/freq_step
# 5   ← 5% per step

cat /sys/devices/system/cpu/cpufreq/policy0/conservative/sampling_rate
# 20000  ← 20ms

cat /sys/devices/system/cpu/cpufreq/policy0/conservative/down_threshold
# 20   ← لازم load < 20% عشان ينزل
```

مع `freq_step=5` والـ AM62x عنده مثلاً 10 OPPs، يحتاج 10 samples × 20ms = 200ms عشان يوصل من max لـ min. في الـ 30-second sleep، ده مش مشكلة نظرياً. لكن في الـ LoRa transmission (burst)، الـ CPU بيطلع لـ max وبعدين ياخد 200ms عشان يرجع. لو الـ next sleep = 30s، فـ 200ms waste مقبول. لكن لو في frequent interrupts أو background tasks، الـ frequency بتفضل elevated.

```bash
# تابع الـ frequency بشكل real-time
watch -n 0.1 "cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq"

# شوف الـ cpuinfo_cur_freq (الحقيقي من الـ hardware)
cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_cur_freq
```

#### الحل
لهذا الـ use case، الـ `powersave` governor أو الـ `userspace` governor مع manual control أفضل بكتير:

**Option 1 — استخدم `powersave` للـ baseline:**
```bash
echo powersave > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```
هيثبّت التردد على `scaling_min_freq` دايماً. لو الـ LoRa burst محتاج performance، ارفع الـ limit مؤقتاً من الـ application.

**Option 2 — `userspace` governor مع application control:**
```bash
echo userspace > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor

# في الـ application (C pseudocode):
// قبل الـ LoRa transmission
write("/sys/devices/system/cpu/cpufreq/policy0/scaling_setspeed", "1000000"); // 1GHz

// بعد الـ transmission، ارجع للـ minimum
write("/sys/devices/system/cpu/cpufreq/policy0/scaling_setspeed", "200000"); // 200MHz
```

**Option 3 — لو محتاج تفضل بـ `conservative`، شدّد الـ settings:**
```bash
# خلّي الـ down_threshold أعلى (انزل بسرعة أكبر)
echo 40 > /sys/devices/system/cpu/cpufreq/policy0/conservative/down_threshold

# زوّد الـ freq_step عشان الـ ramp-down أسرع
echo 15 > /sys/devices/system/cpu/cpufreq/policy0/conservative/freq_step

# خفف الـ sampling_rate
echo 50000 > /sys/devices/system/cpu/cpufreq/policy0/conservative/sampling_rate
```

#### الدرس المستفاد
الـ `conservative` governor مش دايماً الأوفر في الـ power في الـ bursty workloads. اسمه "conservative" يعني conservative في التغيير مش conservative في الـ power. في الـ IoT sleep/wake patterns، الـ `powersave` أو `userspace` أوضح وأكثر predictability. دايماً measure الـ actual current consumption بـ power analyzer، مش تتوقع من اسم الـ governor.

---

### السيناريو 5: Automotive ECU على i.MX8MP — الـ `ondemand` بيتسبب في non-deterministic behavior في Safety-Critical System

#### العنوان
الـ AUTOSAR stack على i.MX8MP بيفشل في الـ timing validation بسبب الـ CPUFreq `ondemand` governor

#### السياق
شركة automotive بتعمل ECU بالـ NXP i.MX8M Plus (Cortex-A53 quad-core + Cortex-M7). الـ Linux بيشتغل على الـ A53 وبيدير CAN bus communication وبيعمل SOME/IP middleware. الـ safety validation team طلبت timing guarantees لـ specific CAN message response times (< 5ms).

#### المشكلة
الـ timing tests بيفشلوا بشكل non-reproducible. أحياناً الـ response time = 2ms وأحياناً = 8ms. الفريق اشتبه في الـ kernel scheduler، لكن الجذر كان الـ `ondemand` governor.

#### التحليل
الـ `ondemand` governor بيشتغل عبر workqueue (asynchronous):
- بيقيس الـ CPU idle time
- بيحسب الـ load
- لو فوق `up_threshold` (default 80%)، بيطلب `scaling_max_freq`
- الـ workqueue run نفسه بيأثر على الـ idle time measurement

```bash
# شوف الـ current up_threshold
cat /sys/devices/system/cpu/cpufreq/policy0/ondemand/up_threshold
# 80

cat /sys/devices/system/cpu/cpufreq/policy0/ondemand/sampling_rate
# 20000  ← 20ms sampling

cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_transition_latency
# 200000  ← 200µs
```

المشكلة:
1. الـ CAN message يوصل → interrupt handler يصحّى الـ task
2. الـ CPU لسه شغال على min frequency (e.g., 800MHz بدل 1.6GHz)
3. الـ `ondemand` لسه ماشافش إن الـ load ارتفع (الـ sampling rate = 20ms)
4. الـ response يحصل على 800MHz → أبطأ بنسبة 50%
5. في الـ next sampling cycle، الـ governor يشوف الـ load ويطلع للـ max

وفي الـ documentation بيقول صراحة:
> "the CPU P-state updates triggered by it can be relatively irregular"

```bash
# أثبت المشكلة بـ tracing
echo 1 > /sys/kernel/debug/tracing/events/power/cpu_frequency/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل الـ CAN test
cat /sys/kernel/debug/tracing/trace | grep cpu_frequency
```

#### الحل

**الحل الصح للـ automotive safety context:**

```bash
# اقفل الـ frequency على max
echo performance > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor

# أو ضع scaling_min_freq = scaling_max_freq
MAX=$(cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_max_freq)
echo $MAX > /sys/devices/system/cpu/cpufreq/policy0/scaling_min_freq
echo $MAX > /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq
```

**في الـ systemd service أو init script:**
```bash
[Unit]
Description=Set CPU performance mode for automotive ECU
After=systemd-udevd.service

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'for p in /sys/devices/system/cpu/cpufreq/policy*; do echo performance > $p/scaling_governor; done'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

**وفي الـ kernel cmdline لضمان أسبق setting:**
```
cpufreq.default_governor=performance
```

**للتحقق من الثبات:**
```bash
# شوف إن الـ cur_freq = max_freq دايماً
while true; do
    cur=$(cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_cur_freq)
    max=$(cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_max_freq)
    if [ "$cur" != "$max" ]; then
        echo "VIOLATION: cur=$cur max=$max"
    fi
    sleep 0.1
done
```

**لو المتطلبات بتحتاج power saving في بعض الأوقات (e.g., parked mode):**
```bash
# من الـ application، غيّر الـ governor حسب الـ vehicle state
echo powersave > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor  # parked
echo performance > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor  # driving
```

#### الدرس المستفاد
الـ `ondemand` governor مذكور في الـ documentation إنه asynchronous وإن الـ P-state updates غير منتظمة. في أي safety-critical أو real-time system، الـ `performance` governor هو الوحيد اللي بيضمن deterministic behavior. الـ `cpuinfo_transition_latency` مش بس عن السرعة — هو جزء من الـ worst-case timing analysis اللي الـ safety engineer لازم يحسبه. الـ `scaling_cur_freq` و`cpuinfo_cur_freq` لازم يكونوا متطابقين في الـ locked-performance mode.
## Phase 7: مصادر ومراجع

### مقالات LWN.net الأساسية

دي أهم المقالات اللي اتكتبت على LWN.net عن الـ CPUFreq subsystem:

| المقال | الموضوع |
|--------|---------|
| [Per-entity load tracking](https://lwn.net/Articles/531853/) | شرح PELT اللي بتستخدمه schedutil — المقال المُشار إليه مباشرة في الـ kernel doc |
| [Improvements in CPU frequency management](https://lwn.net/Articles/682391/) | تغطية إضافة governor الـ schedutil في kernel 4.7 وتكاملها مع الـ scheduler |
| [cpufreq: schedutil governor](https://lwn.net/Articles/678777/) | تغطية مفصلة لتصميم schedutil وفلسفتها كـ replacement للـ ondemand |
| [Toward better CPU load estimation](https://lwn.net/Articles/741171/) | تحسينات PELT وتأثيرها على دقة قرارات الـ cpufreq |
| [CPU frequency governors and remote callbacks](https://lwn.net/Articles/732740/) | ميكانيزم الـ remote CPU utilization updates في الـ governors |
| [cpufreq: introduce a new AMD CPU frequency control mechanism](https://lwn.net/Articles/868671/) | مقدمة لـ amd-pstate driver كبديل لـ acpi-cpufreq على AMD Zen 2+ |
| [Implement AMD Pstate EPP Driver](https://lwn.net/Articles/916147/) | إضافة Energy Performance Preference لـ amd-pstate |
| [amd-pstate preferred core](https://lwn.net/Articles/947712/) | دعم الـ preferred core ranking في amd-pstate |
| [CPUFreq driver using CPPC methods](https://lwn.net/Articles/650781/) | driver الـ ACPI CPPC اللي بيدعم ARM وغيره |
| [sched: Add schedutil overview](https://lwn.net/Articles/840745/) | نظرة شاملة على كيفية استخدام PELT مع schedutil |
| [cpufreq: Specify the default governor on command line](https://lwn.net/Articles/824303/) | إضافة `cpufreq.default_governor=` كـ kernel parameter |
| [1/3 A dynamic cpufreq governor](https://lwn.net/Articles/55587/) | الـ patch الأصلي لـ ondemand governor (2003) |
| [cpufreq core for 2.5](https://lwn.net/Articles/1476/) | تاريخ الـ CPUFreq core في kernel 2.5 |

---

### التوثيق الرسمي في الـ Kernel

#### مسارات `Documentation/` المهمة

```
Documentation/admin-guide/pm/cpufreq.rst      ← الملف الرئيسي (هذا المستند)
Documentation/admin-guide/pm/intel_pstate.rst ← driver الـ Intel
Documentation/admin-guide/pm/amd-pstate.rst   ← driver الـ AMD Zen 2+
Documentation/cpu-freq/core.txt               ← تصميم الـ CPUFreq core
Documentation/cpu-freq/cpu-drivers.txt        ← كيفية كتابة scaling driver
Documentation/cpu-freq/governors.txt          ← توثيق الـ governors القديم
Documentation/cpu-freq/user-guide.txt         ← دليل الاستخدام من userspace
Documentation/scheduler/schedutil.rst         ← توثيق schedutil من زاوية الـ scheduler
```

#### الـ sysfs interface

**الـ** `/sys/devices/system/cpu/cpufreq/policyX/` بيحتوي على:

```
affected_cpus
cpuinfo_cur_freq
cpuinfo_max_freq
cpuinfo_min_freq
cpuinfo_transition_latency
related_cpus
scaling_available_frequencies
scaling_available_governors
scaling_cur_freq
scaling_driver
scaling_governor          ← read-write
scaling_max_freq          ← read-write
scaling_min_freq          ← read-write
scaling_setspeed          ← فعّال بس مع userspace governor
```

---

### التوثيق الرسمي Online

- [CPU Performance Scaling — kernel.org](https://docs.kernel.org/admin-guide/pm/cpufreq.html)
- [Schedutil — kernel.org](https://docs.kernel.org/scheduler/schedutil.html)
- [amd-pstate — kernel.org](https://docs.kernel.org/admin-guide/pm/amd-pstate.html)
- [intel_pstate — kernel.org](https://docs.kernel.org/admin-guide/pm/intel_pstate.html)

---

### Commits مهمة في تاريخ CPUFreq

| الـ Commit / الحدث | الأهمية |
|-------------------|---------|
| `fe7034338ba0` — cpufreq: Add mechanism for registering utilization update callbacks | فتح الباب لـ schedutil بإضافة `cpufreq_update_util()` |
| kernel 4.7 (2016) | إضافة schedutil كأول governor مدمج مع الـ scheduler |
| kernel 5.17 (2022) | إطلاق amd-pstate driver لـ AMD Zen 2+ |
| kernel 6.3 (2023) | amd-pstate أصبح الـ default على AMD |
| kernel 5.7 (2020) | frequency-invariant scheduler accounting على x86 |

للبحث عن commits بالتفصيل:
```bash
git log --oneline -- drivers/cpufreq/ | head -30
git log --oneline -- kernel/sched/cpufreq*.c
```

---

### نقاشات Mailing List

- [schedutil: New governor based on scheduler utilization data — LKML](https://lkml.iu.edu/hypermail/linux/kernel/1603.0/01376.html)
- [schedutil patch v3 — Patchwork](https://patchwork.kernel.org/project/linux-pm/patch/1855303.8Nyy7L51ju@vostro.rjw.lan/)
- [cpufreq: interactive governor — LWN](https://lwn.net/Articles/662209/)
- [LKML Archives — linux-pm](https://lore.kernel.org/linux-pm/) ← الأرشيف الكامل لـ linux-pm mailing list

---

### Kernel Newbies — تغييرات CPUFreq عبر الإصدارات

| الإصدار | التغيير |
|---------|---------|
| [Linux 4.7](https://kernelnewbies.org/Linux_4.7) | إضافة schedutil governor + fast frequency switching في acpi-cpufreq |
| [Linux 4.10](https://kernelnewbies.org/Linux_4.10) | intel_pstate يعمل مع الـ generic cpufreq governors |
| [Linux 4.11](https://kernelnewbies.org/Linux_4.11) | إضافة `cpufreq.off=1` boot option |
| [Linux 4.14](https://kernelnewbies.org/Linux_4.14) | الـ scheduler بيبعت notifications لـ remote CPUs |
| [Linux 5.7](https://kernelnewbies.org/Linux_5.7) | frequency-invariant scheduling على x86 + schedutil default لـ intel_pstate |

---

### eLinux.org

- [Tests:R-CAR-GEN3-CPUFreq](https://elinux.org/Tests:R-CAR-GEN3-CPUFreq) — اختبارات CPUFreq على Renesas R-Car Gen3 (مثال embedded real-world)
- [CELF PM Requirements 2006](https://www.elinux.org/CELF_PM_Requirements_2006) — المتطلبات التاريخية لـ power management في embedded Linux

---

### كتب مُوصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 14**: Linux Device Model — الـ kobject وكيف يُبنى الـ sysfs interface
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)
- الـ CPUFreq نفسه مش موجود صراحةً لأن LDD3 قديم، لكن الـ sysfs infrastructure والـ driver model المشروحين فيه أساسيين لفهم CPUFreq

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل 11**: Timers and Time Management — مهم لفهم الـ sampling في ondemand/conservative
- **الفصل 13**: The Virtual Filesystem — لفهم كيف يعمل الـ sysfs
- **الفصل 14**: The Block I/O Layer — context للـ I/O-wait boosting في schedutil

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل 16**: Kernel Debugging Techniques — بيشمل sysfs inspection وكيف تـ debug مشاكل الـ cpufreq
- مهم للـ embedded developers اللي بيشتغلوا على ARM platforms مع cpufreq drivers

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- يغطي power management infrastructure بالتفصيل مع code walk-through

---

### مصادر إضافية

#### ArchWiki
- [CPU frequency scaling — ArchWiki](https://wiki.archlinux.org/title/CPU_frequency_scaling) — أفضل userspace guide عملي لـ cpufreq tools والـ governors

#### أدوات Userspace
- **cpupower** (جزء من `linux-tools`): الأداة الرئيسية للتحكم في cpufreq
- **cpufrequtils**: مجموعة أدوات قديمة بديلة
- [cpupowerutils — LWN](https://lwn.net/Articles/433002/)

```bash
# عرض المعلومات الحالية
cpupower frequency-info

# تغيير الـ governor
cpupower frequency-set -g schedutil

# تحديد نطاق التردد
cpupower frequency-set -d 1GHz -u 3GHz
```

---

### Search Terms للبحث عن معلومات أكثر

```
cpufreq schedutil PELT frequency scaling linux
linux kernel P-state DVFS governor
cpufreq_policy struct kernel source
acpi-cpufreq driver intel_pstate amd-pstate
linux power management DVFS embedded
cpufreq sysfs interface kernel documentation
schedutil rate_limit_us tuning
ondemand vs schedutil performance comparison
linux frequency boost turbo core cpufreq
ACPI CPPC collaborative processor performance control
```
## Phase 8: Writing simple module

### الهدف

الـ CPUFreq subsystem بيوفر **notifier chain** اسمها `CPUFREQ_TRANSITION_NOTIFIER` — بتتفعّل كل ما الـ kernel بيطلب تغيير تردد الـ CPU. هنعمل module بيـ hook على الـ notifier دي ويطبع معلومات كل transition.

---

### الـ Hook المختار: `cpufreq_register_notifier` مع `CPUFREQ_TRANSITION_NOTIFIER`

الـ notifier دي بتشتغل في كل مرة بيبدأ أو بيخلص الـ transition بين P-states. بتجيب `struct cpufreq_freqs` اللي فيها الـ policy اللي اتغيرت، والتردد القديم والجديد بالـ kHz — مفيدة جداً لمراقبة سلوك الـ governor في real-time.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * cpufreq_watch.c — monitor every CPU frequency transition via notifier
 *
 * Hooks into CPUFREQ_TRANSITION_NOTIFIER and logs old/new freq for each policy.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/cpufreq.h>      /* cpufreq_register_notifier, struct cpufreq_freqs,
                                   CPUFREQ_TRANSITION_NOTIFIER,
                                   CPUFREQ_PRECHANGE, CPUFREQ_POSTCHANGE */
#include <linux/notifier.h>     /* struct notifier_block, notifier_call_chain */

/* ------------------------------------------------------------------ */
/*  Notifier callback — called by cpufreq core before and after        */
/*  every frequency transition.                                        */
/* ------------------------------------------------------------------ */
static int cpufreq_watch_notify(struct notifier_block *nb,
                                unsigned long event,
                                void *data)
{
    struct cpufreq_freqs *freqs = data;  /* cast the opaque pointer */

    /*
     * event is either CPUFREQ_PRECHANGE (before the HW switch)
     * or CPUFREQ_POSTCHANGE (after the HW switch completes).
     * We log only POSTCHANGE so we know the transition actually happened.
     */
    if (event != CPUFREQ_POSTCHANGE)
        return NOTIFY_OK;   /* let the chain continue, nothing to do */

    /*
     * freqs->policy->cpu  — the managing CPU of this policy
     * freqs->old          — previous frequency in kHz
     * freqs->new          — new (requested) frequency in kHz
     *
     * Dividing by 1000 converts kHz → MHz for human-readable output.
     */
    pr_info("cpufreq_watch: CPU%u  %u MHz → %u MHz  (policy min=%u MHz, max=%u MHz)\n",
            freqs->policy->cpu,
            freqs->old / 1000,
            freqs->new / 1000,
            freqs->policy->min / 1000,
            freqs->policy->max / 1000);

    return NOTIFY_OK;  /* always return OK; we don't block transitions */
}

/* ------------------------------------------------------------------ */
/*  The notifier_block that ties the callback into the cpufreq chain  */
/* ------------------------------------------------------------------ */
static struct notifier_block cpufreq_watch_nb = {
    .notifier_call = cpufreq_watch_notify,
    /*
     * .priority defaults to 0 — middle of the chain.
     * Higher values run first; we don't need to be first or last.
     */
};

/* ------------------------------------------------------------------ */
/*  module_init — register with the cpufreq transition notifier chain */
/* ------------------------------------------------------------------ */
static int __init cpufreq_watch_init(void)
{
    int ret;

    /*
     * cpufreq_register_notifier registers our notifier_block
     * with the CPUFREQ_TRANSITION_NOTIFIER chain inside the cpufreq core.
     * From this point on, every freq change will call cpufreq_watch_notify.
     */
    ret = cpufreq_register_notifier(&cpufreq_watch_nb,
                                    CPUFREQ_TRANSITION_NOTIFIER);
    if (ret) {
        pr_err("cpufreq_watch: failed to register notifier (%d)\n", ret);
        return ret;
    }

    pr_info("cpufreq_watch: loaded — watching all CPU frequency transitions\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit — MUST unregister before the module is removed        */
/* ------------------------------------------------------------------ */
static void __exit cpufreq_watch_exit(void)
{
    /*
     * Unregistering is mandatory: if the module is removed while the
     * notifier_block is still in the chain, the next freq transition
     * will call a function that no longer exists → kernel oops / use-after-free.
     */
    cpufreq_unregister_notifier(&cpufreq_watch_nb,
                                CPUFREQ_TRANSITION_NOTIFIER);

    pr_info("cpufreq_watch: unloaded\n");
}

module_init(cpufreq_watch_init);
module_exit(cpufreq_watch_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example Author <author@example.com>");
MODULE_DESCRIPTION("Log every CPUFreq P-state transition via CPUFREQ_TRANSITION_NOTIFIER");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `<linux/module.h>` | الـ macros الأساسية للـ kernel module: `module_init`, `module_exit`, `MODULE_LICENSE` |
| `<linux/kernel.h>` | `pr_info` / `pr_err` للـ kernel logging |
| `<linux/cpufreq.h>` | تعريف `cpufreq_register_notifier`, `struct cpufreq_freqs`, `struct cpufreq_policy`, والـ constants زي `CPUFREQ_POSTCHANGE` |
| `<linux/notifier.h>` | تعريف `struct notifier_block` و`NOTIFY_OK` |

الـ `cpufreq.h` هو المحور — فيه تعريف `struct cpufreq_freqs` اللي بتاخد معلومات الـ transition كلها، و`struct cpufreq_policy` اللي فيها حدود الـ min/max للـ policy دي.

---

#### الـ Callback: `cpufreq_watch_notify`

```c
static int cpufreq_watch_notify(struct notifier_block *nb,
                                unsigned long event, void *data)
```

- **`nb`**: pointer للـ `notifier_block` اللي سجلنا بيه — مش بنستخدمه هنا لأن الـ state كله جاي من `data`.
- **`event`**: إما `CPUFREQ_PRECHANGE` (قبل التغيير) أو `CPUFREQ_POSTCHANGE` (بعد ما الـ hardware اتحول فعلاً).
- **`data`**: الـ `void *` بيشاور على `struct cpufreq_freqs` — فيه الـ policy وتردد ما قبل وما بعد.

بنـ filter على `CPUFREQ_POSTCHANGE` عشان نتأكد إن التغيير اتم فعلاً قبل ما نطبعه — الـ `PRECHANGE` ممكن يتلغى أو يتعدل.

الـ return `NOTIFY_OK` معناه "شوفت الـ event ومش محتاج أوقف الـ chain".

---

#### الـ `notifier_block`

```c
static struct notifier_block cpufreq_watch_nb = {
    .notifier_call = cpufreq_watch_notify,
};
```

الـ `struct notifier_block` هو الـ "entry point" اللي بيسجّله الـ cpufreq core في الـ linked list بتاع الـ notifier chain. أي كود تاني ممكن يضيف الـ `notifier_block` بتاعه وكلهم هياخدوا نفس الـ events بالترتيب حسب الـ `priority`.

---

#### الـ `module_init`

بيستدعي `cpufreq_register_notifier` مع الـ flag `CPUFREQ_TRANSITION_NOTIFIER` عشان يميّز الـ chain دي عن `CPUFREQ_POLICY_NOTIFIER` (اللي بتتعلق بإنشاء وحذف الـ policies مش تغيير التردد).

لو الـ registration فشل (مثلاً `CONFIG_CPU_FREQ` مش enabled)، الـ module بيرجع الـ error ومش بيكمل load.

---

#### الـ `module_exit`

**الـ unregister إجباري** — لو شلنا الـ module وسبنا الـ `notifier_block` في الـ chain، الـ cpufreq core هيحاول يكلم `cpufreq_watch_notify` اللي اتشالت من الـ memory مع الـ module → **use-after-free** وكراش في الـ kernel.

---

### Makefile للـ out-of-tree build

```makefile
obj-m := cpufreq_watch.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ Module

```bash
# بناء الـ module
make

# تحميله
sudo insmod cpufreq_watch.ko

# شغّل workload لتحفيز frequency changes
stress-ng --cpu 4 --timeout 5s

# شوف الـ output
sudo dmesg | grep cpufreq_watch

# إخراجه
sudo rmmod cpufreq_watch
```

**مثال للـ output المتوقع:**

```
[  123.456] cpufreq_watch: loaded — watching all CPU frequency transitions
[  124.100] cpufreq_watch: CPU0  800 MHz → 3600 MHz  (policy min=400 MHz, max=4200 MHz)
[  124.200] cpufreq_watch: CPU4  800 MHz → 3600 MHz  (policy min=400 MHz, max=4200 MHz)
[  129.300] cpufreq_watch: CPU0  3600 MHz → 800 MHz   (policy min=400 MHz, max=4200 MHz)
[  135.000] cpufreq_watch: unloaded
```

كل سطر بيوضح: أنهي CPU core اتأثر، التردد القديم والجديد بالـ MHz، وحدود الـ policy الحالية — ده بيخليك تتابع قرارات الـ governor (ondemand / schedutil / etc.) في real-time.
