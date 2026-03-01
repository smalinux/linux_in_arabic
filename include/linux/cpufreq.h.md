## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: CPU Frequency Scaling Framework

**الـ maintainers**: Rafael J. Wysocki و Viresh Kumar
**الـ mailing list**: linux-pm@vger.kernel.org
**الحالة**: Maintained

---

### القصة — ليش الموضوع ده موجود أصلاً؟

تخيل إنك بتشغّل لاب توب. لما بتشوف يوتيوب وبتعمل scrolling، الـ CPU محتاج طاقة متوسطة. لما بتشغّل لعبة ثقيلة، محتاج أقصى طاقة. لما الشاشة مقفلة ومحدش شايل، محتاجك توفّر أكبر قدر من البطارية — يعني تخفّض الـ frequency لأقل حاجة ممكنة.

الـ CPU مش بيشتغل على سرعة واحدة ثابتة. عنده جدول أسعار من الـ frequencies، مثلاً: 400 MHz، 800 MHz، 1.2 GHz، 2.4 GHz. كل frequency بيصاحبها **voltage** معين — كلما زدت السرعة، زاد الـ voltage والحرارة والاستهلاك.

المشكلة: مين اللي يقرر امتى نروح لأي frequency؟ ومين اللي يكلّم الـ hardware عشان يغيّر السرعة فعلاً؟

ده بالظبط اللي **الـ CPU Frequency Scaling Framework** بيحله — إطار عمل في الـ Linux kernel بيتوسط بين:
- **الـ governors** (صانعو القرار — مين يقرر السرعة المطلوبة؟)
- **الـ drivers** (المنفّذون — مين يكلّم الـ hardware فعلاً؟)
- **الـ policy** (السياسة — ما هو النطاق المسموح بيه؟)

---

### الـ DVFS — الفكرة الجوهرية

**DVFS** = Dynamic Voltage and Frequency Scaling. الفكرة إن الـ CPU:
1. يشتغل على أدنى frequency تكفي للشغل الحالي
2. يخفّض الـ voltage معاها (لأن voltage أقل = طاقة أقل)
3. لما يجي شغل أكتر، يرفع الاتنين مع بعض فوراً

المعادلة: `Power ∝ C × V² × f`
- **C** = capacitance (ثابت للـ chip)
- **V** = voltage
- **f** = frequency

يعني لو خفّضت الـ frequency بالنص وخفّضت الـ voltage معاها، الاستهلاك بينخفض بشكل كبير.

---

### ما هو الـ `cpufreq.h`؟

ده الـ **header الرئيسي** للـ subsystem كله. بيعرّف:

1. **`struct cpufreq_policy`** — قلب الموضوع. بتمثّل "السياسة" لمجموعة CPUs بتشارك نفس الـ clock. بتحتوي على:
   - نطاق الـ frequencies المسموح بيه (`min` / `max` بالـ kHz)
   - الـ frequency الحالية (`cur`)
   - الـ governor المسؤول
   - الـ CPUs المرتبطة بيها (cpumask)
   - flags للـ fast switch والـ boost والـ DVFS

2. **`struct cpufreq_driver`** — الـ interface اللي كل hardware driver لازم يملأه. بيحتوي على function pointers زي `init`, `verify`, `target_index`, `fast_switch`, إلخ.

3. **`struct cpufreq_governor`** — الـ interface للـ governor اللي بيقرر السرعة المطلوبة.

4. **`struct cpufreq_frequency_table`** — جدول الـ frequencies المتاحة في الـ hardware.

5. **الـ notifier interface** — عشان أجزاء تانية من الـ kernel (زي الـ thermal subsystem) تتعلم لما الـ frequency اتغيّرت.

---

### الثلاث طبقات — الصورة الكاملة

```
┌─────────────────────────────────────────────────────┐
│              User Space / sysfs                      │
│   /sys/devices/system/cpu/cpu0/cpufreq/              │
│   scaling_governor, scaling_min_freq, scaling_max_freq│
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│           GOVERNOR (صانع القرار)                     │
│  performance │ powersave │ ondemand │ schedutil       │
│                                                      │
│  "الشغل كتير؟ اطلب frequency عالية"                 │
│  "الشغل قليل؟ اطلب frequency منخفضة"                │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│           CORE FRAMEWORK  (cpufreq.c + cpufreq.h)    │
│                                                      │
│  • يتحقق إن الـ frequency المطلوبة في النطاق        │
│  • يبعت notifications للمشتركين                     │
│  • يدير الـ policy و الـ sysfs                      │
│  • يختار الـ frequency الأنسب من الجدول             │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│           DRIVER (المنفّذ)                            │
│  intel_pstate │ amd-pstate │ acpi-cpufreq │ scmi      │
│  cpufreq-dt  │ qcom-cpufreq │ ...                    │
│                                                      │
│  "يكلّم الـ hardware فعلاً ويغيّر السرعة"           │
└─────────────────────────────────────────────────────┘
```

---

### الـ Governors المتاحة — مين يقرر؟

| Governor | الاستراتيجية |
|---|---|
| **performance** | دايماً أعلى frequency |
| **powersave** | دايماً أدنى frequency |
| **ondemand** | يرفع فوراً لما الـ load يعلى |
| **conservative** | يرفع وينزل تدريجياً |
| **userspace** | المستخدم يتحكم يدوياً |
| **schedutil** | مبني على إحصائيات الـ scheduler مباشرة (الأحدث والأذكى) |

---

### الـ Policy — المساحة المسموح بيها

**الـ policy** مش governor. الـ policy هي الـ constraint:
- `min` و `max`: النطاق المسموح بيه (قد يُضيَّق بسبب حرارة أو QoS)
- الـ governor بيختار أي frequency *جوا* النطاق ده

مثلاً: لو المستخدم حدّد `scaling_max_freq = 1.6 GHz` والـ governor طلب 2.4 GHz، الـ core بيضرب في 1.6 GHz.

---

### الـ Fast Switch — لما السرعة مهمة

الـ governors التقليدية بتغيّر الـ frequency من خلال work queue = بطيء.

**الـ fast_switch** بيخلّي الـ governor يغيّر الـ frequency مباشرة من الـ scheduler's hotpath (من جوا interrupt context) بدون أي locking ثقيل — ده critical لـ latency-sensitive workloads.

---

### القصة الكاملة: ماذا يحدث لما تشغّل لعبة؟

1. **الـ scheduler** يلاحظ إن الـ CPU utilization وصل 90%
2. يبلّغ الـ **schedutil governor** عن طريق `update_util_data->func()`
3. الـ governor يحسب الـ frequency المطلوبة: `target = util/cap × max_freq`
4. يطلب من الـ **core** يغيّر الـ frequency
5. الـ core يبحث في الـ **frequency table** عن أقرب frequency متاحة
6. يتحقق إن الـ frequency في نطاق الـ `policy->min/max`
7. يبعت **CPUFREQ_PRECHANGE** notification للـ notifiers (زي الـ thermal)
8. يطلب من الـ **driver** (`target_index()` أو `fast_switch()`) يغيّر السرعة فعلاً
9. الـ driver يكلّم الـ hardware (عبر ACPI, SCMI, registers, إلخ)
10. يبعت **CPUFREQ_POSTCHANGE** notification

---

### الـ Inefficient Frequencies — تفصيلة مهمة

بعض الـ CPUs عندها frequencies بتاكل طاقة كتير لأداء قليل (inefficient operating points). الـ framework بيسمح للـ driver يعلّم frequencies بـ `CPUFREQ_INEFFICIENT_FREQ` وبعدين الـ governor مع flag الـ `CPUFREQ_RELATION_E` يتجنبها لو ممكن.

---

### الـ Thermal Integration

لو الـ CPU بدأ يسخن، الـ **thermal subsystem** بيقلّل الـ `policy->max` عن طريق الـ **freq_constraints** و **QoS** mechanism — يعني الـ governor مش ممكن يطلب frequency فوق الحد الحراري. الـ `cpufreq_policy` عنده `struct thermal_cooling_device *cdev` للتكامل ده.

---

### الملفات الأساسية للـ Subsystem

**الـ Core (قلب الـ framework):**
| الملف | الدور |
|---|---|
| `include/linux/cpufreq.h` | الـ header الرئيسي — كل الـ structs والـ interfaces |
| `drivers/cpufreq/cpufreq.c` | تنفيذ الـ core logic (init, policy management, notifiers) |
| `drivers/cpufreq/cpufreq_stats.c` | إحصائيات وقت الإقامة على كل frequency |
| `drivers/cpufreq/cpufreq_governor.c` | كود مشترك بين الـ governors |
| `include/linux/sched/cpufreq.h` | الـ interface بين الـ scheduler والـ cpufreq |

**الـ Governors:**
| الملف | الدور |
|---|---|
| `drivers/cpufreq/cpufreq_performance.c` | governor الأداء الكامل |
| `drivers/cpufreq/cpufreq_powersave.c` | governor توفير الطاقة |
| `drivers/cpufreq/cpufreq_ondemand.c` | governor الطلب الفوري |
| `drivers/cpufreq/cpufreq_conservative.c` | governor التدرج |
| `kernel/sched/cpufreq_schedutil.c` | governor الـ scheduler (الأحدث والأفضل) |
| `kernel/sched/cpufreq.c` | ربط الـ scheduler بالـ cpufreq hooks |

**الـ Hardware Drivers (عينات):**
| الملف | الدور |
|---|---|
| `drivers/cpufreq/intel_pstate.c` | Intel modern CPUs |
| `drivers/cpufreq/amd-pstate.c` | AMD modern CPUs |
| `drivers/cpufreq/acpi-cpufreq.c` | ACPI generic P-states |
| `drivers/cpufreq/cpufreq-dt.c` | ARM DeviceTree-based CPUs |
| `drivers/cpufreq/scmi-cpufreq.c` | SCMI firmware interface |
| `drivers/cpufreq/qcom-cpufreq-hw.c` | Qualcomm Snapdragon |

**الـ Related Headers:**
| الملف | الدور |
|---|---|
| `include/linux/pm_opp.h` | الـ OPP table (frequency/voltage pairs) |
| `include/linux/pm_qos.h` | الـ QoS constraints (الحدود القادمة من thermal وغيره) |
| `drivers/cpufreq/cpufreq-dt.h` | helpers للـ device-tree based drivers |
## Phase 2: شرح الـ CPUFreq Framework

### المشكلة — ليه الـ CPUFreq موجود أصلًا؟

الـ CPU الحديث مش بيشتغل بتردد واحد ثابت. الـ hardware عنده القدرة إنه يغير الـ **clock frequency** والـ **voltage** في نفس الوقت — العملية دي اسمها **DVFS (Dynamic Voltage and Frequency Scaling)**. لو الـ CPU اشتغل دايمًا بأقصى تردد، هيستهلك طاقة تانية وحرارة تانية. لو اشتغل بأدنى تردد، هيكون بطيء.

المشكلة الحقيقية:
- كل chip مختلف — Qualcomm Snapdragon, MediaTek Dimensity, Intel, ARM Cortex-A كل واحد عنده mechanism مختلف لتغيير التردد.
- محتاجين جهة تقرر *امتى* نغير التردد وجهة تنفذ *إزاي* نغيره.
- محتاجين subsystems تانية زي الـ thermal management والـ scheduler يعرفوا التردد الحالي.

من غير CPUFreq، كل driver محتاج يعمل كل ده لوحده — فوضى كاملة.

---

### الحل — الـ CPUFreq Framework

الـ kernel فصل المسألة لـ 3 طبقات مستقلة:

| الطبقة | المسؤولية |
|--------|-----------|
| **cpufreq core** | التنسيق العام، الـ sysfs، الـ notifiers |
| **cpufreq driver** | التعامل المباشر مع الـ hardware لتغيير التردد |
| **cpufreq governor** | اتخاذ قرار *أي* تردد المفروض نستخدم |

---

### Analogy — وزير الكهرباء، المحطة، والمستشار

تخيل دولة عندها شبكة كهرباء:

- **الـ governor** = المستشار الاقتصادي — بيقول "الطلب قليل دلوقتي، خففوا الإنتاج" أو "في ذروة، اشتغلوا بالطاقة الكاملة". بيشوف الـ load ويقرر.
- **الـ cpufreq core** = وزير الكهرباء — بياخد القرار من المستشار، بيعمل الـ validation، بيبعت notifications للجهات المهتمة، وبيحول الأمر لأمر تنفيذي.
- **الـ driver** = محطة توليد الكهرباء — بتنفذ الأمر على أرض الواقع، بترفع أو تخفض التوليد الفعلي.
- **الـ cpufreq_policy** = العقد المحدد للمحطة — فيه الحد الأدنى والأقصى المسموح، التردد الحالي، ومن المرتبطين بنفس المحطة.
- **الـ freq_table** = فئات الإنتاج المتاحة — المحطة مش بتشتغل بأي رقم، في قيم محددة فقط (OPPs).
- **الـ notifier chain** = نشرة إخبارية — كل جهة مهتمة (thermal، scheduler) بتستقبل إشعار لما التردد بيتغير.

الـ mapping الكامل:

```
Governor  →  يقرر target freq  →  cpufreq_policy (العقد)
Core      →  validate + notify  →  notifier_block list
Driver    →  target_index() / fast_switch()  →  HW registers
freq_table →  الترددات المسموح بها فقط
OPP table →  لكل تردد، الجهد المناسب (Voltage)
```

---

### معمارية الـ Framework — Big Picture

```
┌─────────────────────────────────────────────────────────────┐
│                    Userspace / sysfs                         │
│   /sys/devices/system/cpu/cpu0/cpufreq/                     │
│   scaling_governor | scaling_min_freq | scaling_cur_freq    │
└────────────────────────┬────────────────────────────────────┘
                         │ read/write
┌────────────────────────▼────────────────────────────────────┐
│                   CPUFreq Core                               │
│                                                             │
│  ┌──────────────┐   ┌───────────────┐   ┌───────────────┐  │
│  │cpufreq_policy│   │ notifier chain│   │ freq QoS      │  │
│  │  .min/.max   │   │ PRECHANGE     │   │ freq_constraints│ │
│  │  .cur        │   │ POSTCHANGE    │   │ min_freq_req  │  │
│  │  .cpus mask  │   │ POLICY_CREATE │   │ max_freq_req  │  │
│  └──────┬───────┘   └───────────────┘   └───────────────┘  │
│         │                                                    │
│  ┌──────▼───────────────────────────────────────────────┐   │
│  │              Governor Interface                       │   │
│  │  init() → start() → limits() → stop() → exit()      │   │
│  └──────┬───────────────────────────────────────────────┘   │
└─────────│───────────────────────────────────────────────────┘
          │ cpufreq_driver_target() / fast_switch()
┌─────────▼───────────────────────────────────────────────────┐
│              CPUFreq Driver (platform-specific)              │
│                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │   init()    │  │ target_index │  │  fast_switch()   │   │
│  │  verify()   │  │  get()       │  │  adjust_perf()   │   │
│  └──────┬──────┘  └──────┬───────┘  └────────┬─────────┘   │
└─────────│────────────────│────────────────────│─────────────┘
          │                │                    │
┌─────────▼────────────────▼────────────────────▼─────────────┐
│                    Hardware Layer                            │
│                                                             │
│   PLL registers | SCMI/PSCI firmware | voltage regulators  │
│   Clock controller (clk framework) | OPP table             │
└─────────────────────────────────────────────────────────────┘

Consumers (subsystems that interact with CPUFreq):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Scheduler (schedutil governor)    ← يقرر التردد بناءً على CPU utilization
  Thermal framework                 ← يحط ceiling على التردد لما الحرارة ترتفع
  Energy Model (em_perf_domain)     ← بيستخدم OPP data لحساب الـ power
  PM QoS                            ← يحدد min/max freq constraints
```

---

### الـ Core Abstraction — الـ `cpufreq_policy`

الـ **`struct cpufreq_policy`** هو محور الـ framework كله. مش بيمثل CPU واحد — بيمثل **مجموعة CPUs بتشارك نفس الـ clock domain**.

```c
struct cpufreq_policy {
    cpumask_var_t   cpus;         /* CPUs online sharing this clock */
    cpumask_var_t   related_cpus; /* online + offline CPUs */
    cpumask_var_t   real_cpus;    /* physically present CPUs */

    unsigned int    cpu;          /* the "managing" CPU for this policy */

    struct clk     *clk;          /* the actual clock (clk framework) */
    struct cpufreq_cpuinfo cpuinfo; /* hw limits: min/max freq, latency */

    unsigned int    min;          /* current lower bound (kHz) */
    unsigned int    max;          /* current upper bound (kHz) */
    unsigned int    cur;          /* current frequency (kHz) */

    struct cpufreq_governor  *governor;    /* decision maker */
    struct cpufreq_frequency_table *freq_table; /* allowed freqs */

    /* Frequency transition synchronization */
    bool            transition_ongoing;
    spinlock_t      transition_lock;
    wait_queue_head_t transition_wait;

    /* QoS constraints from other subsystems */
    struct freq_constraints  constraints;
    struct freq_qos_request *min_freq_req;
    struct freq_qos_request *max_freq_req;

    /* Fast switch path (no sleeping, no notifiers) */
    bool            fast_switch_possible;
    bool            fast_switch_enabled;

    /* Thermal cooling integration */
    struct thermal_cooling_device *cdev;

    /* sysfs representation */
    struct kobject  kobj;

    /* Stats tracking */
    struct cpufreq_stats *stats;
};
```

#### ليه `cpumask` وسط الـ policy؟

على ARM big.LITTLE أو Intel Hybrid (P+E cores)، ممكن يكون عندك كل cluster بيشارك PLL واحد. لو CPU0 و CPU1 و CPU2 كلهم على نفس الـ clock، مش ممكن تغير تردد CPU0 من غير ما تأثر على البقية. الـ policy بيعبر عن الـ clock domain ده — مجموعة CPUs بتتحكم فيها بـ policy واحدة.

```
SoC مثال: ARM Cortex-A55 cluster
─────────────────────────────────
CPU0 ─┐
CPU1 ─┤─── shared PLL ──► policy_0 { cpus: {0,1,2,3} }
CPU2 ─┤
CPU3 ─┘

CPU4 ─┐
CPU5 ─┤─── shared PLL ──► policy_4 { cpus: {4,5,6,7} }
CPU6 ─┤
CPU7 ─┘
```

---

### الـ `cpufreq_driver` — واجهة الـ Hardware

```c
struct cpufreq_driver {
    char    name[CPUFREQ_NAME_LEN];
    u16     flags;

    /* إلزامي: تهيئة الـ policy للـ CPU */
    int     (*init)(struct cpufreq_policy *policy);

    /* إلزامي: التحقق من صحة min/max قبل تطبيقهم */
    int     (*verify)(struct cpufreq_policy_data *policy);

    /* اختار واحد من الاتنين: */

    /* الأبسط: بيحدد policy كاملة (powersave/performance) */
    int     (*setpolicy)(struct cpufreq_policy *policy);

    /* الأكثر استخداماً: بيضبط تردد محدد بـ index في الـ freq_table */
    int     (*target_index)(struct cpufreq_policy *policy,
                            unsigned int index);

    /* Fast path: بدون sleep، بدون notifiers — للـ schedutil */
    unsigned int (*fast_switch)(struct cpufreq_policy *policy,
                                unsigned int target_freq);

    /* لـ SCMI/firmware-based platforms */
    void    (*adjust_perf)(unsigned int cpu,
                           unsigned long min_perf,
                           unsigned long target_perf,
                           unsigned long capacity);

    /* لما محتاج تعبر بتردد وسيط قبل الـ target */
    unsigned int (*get_intermediate)(struct cpufreq_policy *policy,
                                     unsigned int index);
    int          (*target_intermediate)(struct cpufreq_policy *policy,
                                        unsigned int index);

    /* اقرأ التردد الحالي من الـ HW */
    unsigned int (*get)(unsigned int cpu);

    /* Lifecycle */
    int     (*online)(struct cpufreq_policy *policy);
    int     (*offline)(struct cpufreq_policy *policy);
    void    (*exit)(struct cpufreq_policy *policy);
    int     (*suspend)(struct cpufreq_policy *policy);
    int     (*resume)(struct cpufreq_policy *policy);

    /* Energy Model registration */
    void    (*register_em)(struct cpufreq_policy *policy);
};
```

#### الـ Fast Switch Path

الـ **`fast_switch`** هو optimization مهم جداً. في الـ path العادي، تغيير التردد بيمر بـ:
1. acquire rwsem
2. إرسال PRECHANGE notifier
3. تغيير التردد
4. إرسال POSTCHANGE notifier
5. release rwsem

ده ممكن ياخد وقت ويسبب scheduling overhead. لو الـ driver يدعم `fast_switch`، الـ schedutil governor بيقدر يغير التردد مباشرةً من الـ scheduler hot path (تحت RCU read lock بس) من غير أي sleeping أو notifiers.

```
Normal Path:
─────────────
governor → cpufreq_driver_target()
         → down_read(rwsem)
         → CPUFREQ_PRECHANGE notifiers
         → driver->target_index()        ← HW write
         → CPUFREQ_POSTCHANGE notifiers
         → up_read(rwsem)

Fast Switch Path:
─────────────────
schedutil → cpufreq_driver_fast_switch()
          → driver->fast_switch()        ← HW write (direct, no locks)
          → arch_set_freq_scale()        ← update scheduler invariance
```

---

### الـ `cpufreq_governor` — صانع القرار

```c
struct cpufreq_governor {
    char    name[CPUFREQ_NAME_LEN];

    int     (*init)(struct cpufreq_policy *policy);  /* أول ما يتعين */
    void    (*exit)(struct cpufreq_policy *policy);  /* لما يتعزل */
    int     (*start)(struct cpufreq_policy *policy); /* لما الـ policy تبدأ */
    void    (*stop)(struct cpufreq_policy *policy);  /* لما الـ policy توقف */
    void    (*limits)(struct cpufreq_policy *policy); /* لما تتغير الـ min/max */

    u8      flags;
};
```

الـ governors الموجودة في الـ kernel:

| Governor | الوصف |
|----------|-------|
| **schedutil** | بيستخدم CPU utilization من الـ scheduler مباشرةً — الأذكى |
| **ondemand** | بيقيس الـ CPU load كل فترة ويضبط التردد |
| **conservative** | زي ondemand بس بيغير التردد بخطوات صغيرة |
| **powersave** | دايمًا عند أدنى تردد |
| **performance** | دايمًا عند أقصى تردد |
| **userspace** | بيسمح لـ userspace يحدد التردد يدوياً |

الـ flag **`CPUFREQ_GOV_DYNAMIC_SWITCHING`** يفرق بين governors بتغير التردد لوحدها (schedutil, ondemand) وبين governors بتحدد policy ثابتة (powersave, performance).

---

### الـ Frequency Table والـ OPP

الـ **`struct cpufreq_frequency_table`** هي قائمة بالترددات اللي الـ hardware يقدر يشتغل بيها فعلاً:

```c
struct cpufreq_frequency_table {
    unsigned int flags;      /* CPUFREQ_BOOST_FREQ | CPUFREQ_INEFFICIENT_FREQ */
    unsigned int driver_data; /* private to the driver */
    unsigned int frequency;   /* kHz */
};
```

الـ flag **`CPUFREQ_INEFFICIENT_FREQ`** مهم جداً — بعض الترددات موجودة في الـ table بس استهلاك الطاقة فيها مش كويس بالنسبة للأداء اللي بيديه (مثلاً، نفس الأداء عند تردد أعلى بجهد أقل). الـ `CPUFREQ_RELATION_E` flag بيطلب من الـ core إنه يتجنب الترددات دي لو في بديل أكفأ.

#### ربط الـ OPP بالـ CPUFreq

الـ **OPP (Operating Performance Point)** هو pair من (frequency, voltage). كل entry في الـ freq_table ليها OPP مقابل فيه الجهد المطلوب. الـ `pm_opp.h` بيعرّف ده.

```
freq_table[0]  →  300 MHz  →  OPP: {300MHz, 750mV}
freq_table[1]  →  600 MHz  →  OPP: {600MHz, 850mV}
freq_table[2]  →  1000 MHz →  OPP: {1GHz,  950mV}
freq_table[3]  →  1200 MHz →  OPP: {1.2GHz, 1050mV} BOOST
```

لما الـ driver بيطلب تغيير التردد، الـ OPP framework بيضبط الجهد الصح قبل/بعد رفع التردد.

---

### الـ Notifier System

الـ CPUFreq بيستخدم نظام الـ **notifier chains** (من `linux/notifier.h`) عشان يبلّغ أي subsystem مهتم بتغييرات التردد.

في نوعين من الـ notifiers:

**1. Transition Notifier** — بيتبعت لما التردد بيتغير فعلاً:
```c
#define CPUFREQ_PRECHANGE   (0)   /* قبل التغيير */
#define CPUFREQ_POSTCHANGE  (1)   /* بعد التغيير */
```
البيانات اللي بتتبعت هي `struct cpufreq_freqs` اللي فيها `.old` و `.new` frequency.

**2. Policy Notifier** — بيتبعت لما الـ policy نفسها بتتغير:
```c
#define CPUFREQ_CREATE_POLICY  (0)
#define CPUFREQ_REMOVE_POLICY  (1)
```

```
cpufreq_freq_transition_begin()
    → notify CPUFREQ_PRECHANGE
    → [driver changes frequency]
cpufreq_freq_transition_end()
    → notify CPUFREQ_POSTCHANGE
    → cpufreq_stats_record_transition()
```

مثال على مشترك: الـ `loops_per_jiffy` المستخدم في `udelay()` لازم يتحسب من جديد لما التردد يتغير — ده بيحصل عبر الـ POSTCHANGE notifier.

---

### الـ Relation Flags — إزاي نختار التردد من الـ Table

لما الـ governor يطلب تردد معين، الـ core محتاج يختار أقرب entry في الـ freq_table. الـ `CPUFREQ_RELATION_*` بيحدد إزاي:

```
Target: 750 MHz
Table: [300, 600, 1000, 1200] MHz

CPUFREQ_RELATION_L → 1000 MHz (lowest freq AT or ABOVE target)
CPUFREQ_RELATION_H →  600 MHz (highest freq AT or BELOW target)
CPUFREQ_RELATION_C →  600 MHz (closest freq to target)

مع CPUFREQ_RELATION_E → يتجنب INEFFICIENT_FREQ entries
```

الـ implementation بيفرق بين الـ table المرتبة ascending أو descending:

```c
/* للـ ascending table: ابحث من الأدنى للأعلى */
static inline int cpufreq_table_find_index_al(...) { ... }

/* للـ descending table: ابحث من الأعلى للأدنى */
static inline int cpufreq_table_find_index_dl(...) { ... }
```

---

### الـ QoS Integration

الـ **PM QoS (Power Management Quality of Service)** — ده subsystem منفصل — بيسمح لأي كود في الـ kernel يطلب minimum أو maximum frequency. مثلاً:
- الـ thermal framework بيضع ceiling على التردد لما الحرارة ترتفع.
- بعض الـ drivers بيطلب minimum frequency عشان تضمن latency معينة.

الـ `cpufreq_policy` بيحتوي على:
```c
struct freq_constraints  constraints;   /* الـ QoS framework object */
struct freq_qos_request *min_freq_req;  /* request من الـ governor/sysfs */
struct freq_qos_request *max_freq_req;  /* request من الـ governor/sysfs */
```

الـ core بيضبط الـ min وmax في الـ policy بناءً على القيم اللي الـ QoS system بيطلع بيها.

---

### الـ Driver Flags — تحكم في سلوك الـ Core

| Flag | المعنى |
|------|--------|
| `CPUFREQ_NEED_UPDATE_LIMITS` | استدعِ الـ driver حتى لو التردد المطلوب لم يتغير (الـ min/max تغيرت) |
| `CPUFREQ_CONST_LOOPS` | `loops_per_jiffy` مش محتاج يتحدث لما التردد يتغير |
| `CPUFREQ_IS_COOLING_DEV` | سجّل الـ driver تلقائياً كـ thermal cooling device |
| `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` | كل policy عندها instance منفصل من الـ governor (مهم للـ multi-cluster) |
| `CPUFREQ_ASYNC_NOTIFICATION` | الـ POSTCHANGE notification بيجي من خارج `->target()` |
| `CPUFREQ_NEED_INITIAL_FREQ_CHECK` | تأكد إن التردد الابتدائي موجود في الـ table |
| `CPUFREQ_NO_AUTO_DYNAMIC_SWITCHING` | لا تسمح بـ governors بيغيروا التردد ديناميكياً |

---

### ما الـ Core يمتلكه vs. ما يفوّضه للـ Driver

| المسؤولية | Owner |
|-----------|-------|
| الـ sysfs interface (`/sys/cpufreq/`) | Core |
| الـ notifier chains | Core |
| اختيار التردد من الـ freq_table | Core |
| الـ policy lifecycle (create/destroy) | Core |
| الـ QoS aggregation | Core |
| الـ statistics (time-in-state) | Core |
| الـ governor management | Core |
| كتابة الـ PLL/clock registers | Driver |
| قراءة التردد الحالي من الـ HW | Driver |
| تحديد OPP table | Driver |
| voltage scaling (عبر OPP framework) | Driver |
| intermediate frequency transitions | Driver |
| firmware communication (SCMI/PSCI) | Driver |
| قرار متى نغير التردد | Governor |
| حساب الـ CPU utilization | Governor (schedutil ← Scheduler) |

---

### الـ Intermediate Frequency — لماذا؟

بعض الـ SoCs عندها قيد hardware: عشان تعمل transition من تردد منخفض جداً لتردد عالي جداً، لازم تمر بتردد وسيط أولاً (مثلاً بسبب PLL lock time أو voltage ramp-up sequence).

الـ driver بيعبّر عن ده بـ:
```c
unsigned int (*get_intermediate)(struct cpufreq_policy *policy, unsigned int index);
int          (*target_intermediate)(struct cpufreq_policy *policy, unsigned int index);
```

الـ core بيعمل التسلسل ده:
```
1. get_intermediate() → ايه التردد الوسيط؟
2. target_intermediate() → اضبط التردد الوسيط
3. [send notifications for intermediate freq]
4. target_index() → اضبط التردد النهائي
5. [send notifications for final freq]
```

---

### ربط الـ Energy Model

الـ `register_em` callback في الـ driver بيسمح للـ driver يسجل نفسه مع الـ **Energy Model framework** (EM). الـ EM بيبني model للطاقة لكل performance domain بناءً على الـ OPP data، والـ EAS (Energy Aware Scheduling) في الـ scheduler بيستخدم الـ model ده عشان يوزع الـ tasks على الـ CPUs بطريقة توفر الطاقة.

```
CPUFreq Driver
    ↓ register_em()
Energy Model (em_perf_domain)
    ↓ يستخدم OPP data (freq + power per freq)
EAS Scheduler
    ↓ يختار CPU الأكفأ طاقةً لكل task
```

الـ `cpufreq_ready_for_eas()` function بتتحقق إن كل الـ perf domains جاهزة قبل ما الـ EAS يشتغل.

---

### الـ Thermal Cooling Integration

لما الـ driver يشيل الـ flag `CPUFREQ_IS_COOLING_DEV`، الـ core بيسجل الـ policy تلقائياً كـ **thermal cooling device**. الـ thermal framework لما تتجاوز الحرارة حد معين بيستدعي الـ cooling device عشان يخفض التردد. النتيجة محفوظة في:

```c
struct thermal_cooling_device *cdev;  /* inside cpufreq_policy */
```

---

### ملخص — الـ Flow الكامل لتغيير تردد CPU

```
Scheduler detects high load
        │
        ▼
schedutil governor calculates target_freq
        │
        ├─── fast_switch enabled? ──YES──► driver->fast_switch()
        │                                        │
        │                                        ▼
        │                               arch_set_freq_scale()
        │
        └─── NO ──► cpufreq_driver_target()
                          │
                          ▼
                    validate against min/max/QoS
                          │
                          ▼
                    cpufreq_freq_transition_begin()
                    → CPUFREQ_PRECHANGE notifiers
                          │
                          ▼
                    cpufreq_frequency_table_target()
                    → find index in freq_table
                          │
                          ▼
                    driver->target_index(policy, idx)
                    → write PLL registers
                    → OPP framework adjusts voltage
                          │
                          ▼
                    cpufreq_freq_transition_end()
                    → CPUFREQ_POSTCHANGE notifiers
                    → update stats
                    → arch_set_freq_scale()
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet: الـ Flags والـ Enums والـ Config Options

#### Driver Flags (`cpufreq_driver.flags` — u16)

| Flag | Bit | المعنى |
|------|-----|--------|
| `CPUFREQ_NEED_UPDATE_LIMITS` | 0 | استدعاء الـ driver حتى لو الـ target_freq ماتغيرش، لأن الـ min/max اتغيروا |
| `CPUFREQ_CONST_LOOPS` | 1 | الـ `loops_per_jiffy` مش بيتأثر بتغيير التردد |
| `CPUFREQ_IS_COOLING_DEV` | 2 | سجّل الـ driver أوتوماتيك كـ thermal cooling device |
| `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` | 3 | كل policy ليها governor منفصل (multi-cluster) |
| `CPUFREQ_ASYNC_NOTIFICATION` | 4 | الـ driver هيبعت POSTCHANGE notification من برّه الـ `target()` |
| `CPUFREQ_NEED_INITIAL_FREQ_CHECK` | 5 | تحقق إن الـ CPU شغّال على تردد موجود في الـ freq_table |
| `CPUFREQ_NO_AUTO_DYNAMIC_SWITCHING` | 6 | امنع الـ governors اللي ليها `dynamic_switching` |

#### Governor Flags (`cpufreq_governor.flags` — u8)

| Flag | Bit | المعنى |
|------|-----|--------|
| `CPUFREQ_GOV_DYNAMIC_SWITCHING` | 0 | الـ governor بيغير التردد بنفسه ديناميكيًا |
| `CPUFREQ_GOV_STRICT_TARGET` | 1 | اضبط التردد بالظبط على الـ target بدون تقريب |

#### Frequency Table Flags (`cpufreq_frequency_table.flags`)

| Flag | Value | المعنى |
|------|-------|--------|
| `CPUFREQ_BOOST_FREQ` | `(1<<0)` | التردد ده boost frequency |
| `CPUFREQ_INEFFICIENT_FREQ` | `(1<<1)` | التردد ده غير كفء (تجنّبه لو في بديل) |
| `CPUFREQ_ENTRY_INVALID` | `~0u` | الـ entry ده ignored (placeholder) |
| `CPUFREQ_TABLE_END` | `~1u` | نهاية الجدول |

#### Relation Flags (لاختيار التردد من الجدول)

| Macro | Value | المعنى |
|-------|-------|--------|
| `CPUFREQ_RELATION_L` | 0 | أقل تردد أكبر من أو يساوي الـ target |
| `CPUFREQ_RELATION_H` | 1 | أعلى تردد أقل من أو يساوي الـ target |
| `CPUFREQ_RELATION_C` | 2 | أقرب تردد للـ target |
| `CPUFREQ_RELATION_E` | `BIT(2)` | flag: خد efficient frequency لو متاح |

#### ACPI Shared Type

| Macro | Value | المعنى |
|-------|-------|--------|
| `CPUFREQ_SHARED_TYPE_NONE` | 0 | مفيش coordination |
| `CPUFREQ_SHARED_TYPE_HW` | 1 | الـ HW بيعمل coordination بنفسه |
| `CPUFREQ_SHARED_TYPE_ALL` | 2 | لازم كل الـ CPUs المرتبطة تضبط التردد |
| `CPUFREQ_SHARED_TYPE_ANY` | 3 | أي CPU مرتبطة تقدر تضبط التردد |

#### Notifier Events

| Macro | List | المعنى |
|-------|------|--------|
| `CPUFREQ_PRECHANGE` | TRANSITION | قبل تغيير التردد |
| `CPUFREQ_POSTCHANGE` | TRANSITION | بعد تغيير التردد |
| `CPUFREQ_CREATE_POLICY` | POLICY | policy اتخلقت |
| `CPUFREQ_REMOVE_POLICY` | POLICY | policy اتشالت |

#### Table Sorting Enum

| Value | المعنى |
|-------|--------|
| `CPUFREQ_TABLE_UNSORTED` | الجدول مش مرتب |
| `CPUFREQ_TABLE_SORTED_ASCENDING` | مرتب تصاعدي |
| `CPUFREQ_TABLE_SORTED_DESCENDING` | مرتب تنازلي |

#### Config Options المهمة

| Config | الوظيفة |
|--------|---------|
| `CONFIG_CPU_FREQ` | تفعيل subsystem الـ cpufreq بالكامل |
| `CONFIG_CPU_FREQ_STAT` | تفعيل إحصائيات التردد في sysfs |
| `CONFIG_CPU_FREQ_GOV_SCHEDUTIL` | تفعيل الـ schedutil governor |
| `CONFIG_CPU_THERMAL` | دعم التحكم الحراري عبر cpufreq |

---

### 1. الـ Structs المهمة

#### `struct cpufreq_cpuinfo`

**الغرض:** معلومات ثابتة عن قدرات الـ CPU الحقيقية — بيملاها الـ driver مرة واحدة أثناء الـ init.

| Field | النوع | الشرح |
|-------|-------|--------|
| `max_freq` | `unsigned int` | أقصى تردد مدعوم بالـ kHz |
| `min_freq` | `unsigned int` | أدنى تردد مدعوم بالـ kHz |
| `transition_latency` | `unsigned int` | زمن التبديل بين الترددات بالـ nanoseconds |

**الارتباط:** متضمنة داخل `cpufreq_policy` و `cpufreq_policy_data` — ده يخلي الـ core يعرف الحدود الأصلية للـ hardware.

---

#### `struct cpufreq_policy`

**الغرض:** القلب النابض لنظام cpufreq. كل مجموعة CPUs بتشارك clock واحد بتتمثل في policy واحدة.

| Field | النوع | الشرح |
|-------|-------|--------|
| `cpus` | `cpumask_var_t` | الـ CPUs الـ online فقط في المجموعة |
| `related_cpus` | `cpumask_var_t` | كل الـ CPUs (online + offline) |
| `real_cpus` | `cpumask_var_t` | موجودة فعلاً في النظام (present) |
| `cpu` | `unsigned int` | الـ CPU المسؤول عن الـ policy (لازم online) |
| `clk` | `struct clk *` | مؤشر للـ clock source |
| `cpuinfo` | `struct cpufreq_cpuinfo` | بيانات الـ hardware الثابتة |
| `min` | `unsigned int` | أدنى تردد مسموح بيه حاليًا (kHz) |
| `max` | `unsigned int` | أقصى تردد مسموح بيه حاليًا (kHz) |
| `cur` | `unsigned int` | التردد الحالي (kHz) |
| `suspend_freq` | `unsigned int` | التردد أثناء الـ suspend |
| `policy` | `unsigned int` | POWERSAVE أو PERFORMANCE (لو setpolicy) |
| `governor` | `struct cpufreq_governor *` | مؤشر للـ governor المفعّل |
| `governor_data` | `void *` | بيانات خاصة بالـ governor |
| `freq_table` | `struct cpufreq_frequency_table *` | جدول الترددات المدعومة |
| `freq_table_sorted` | `enum cpufreq_table_sorting` | حالة ترتيب الجدول |
| `constraints` | `struct freq_constraints` | قيود QoS على الـ min/max |
| `min_freq_req` | `struct freq_qos_request *` | QoS request للحد الأدنى |
| `max_freq_req` | `struct freq_qos_request *` | QoS request للحد الأقصى |
| `kobj` | `struct kobject` | كائن sysfs الخاص بالـ policy |
| `rwsem` | `struct rw_semaphore` | حماية الـ policy (قراءة/كتابة) |
| `fast_switch_possible` | `bool` | الـ driver يدعم fast switch |
| `fast_switch_enabled` | `bool` | الـ governor فعّل fast switch |
| `strict_target` | `bool` | مضبوط من `CPUFREQ_GOV_STRICT_TARGET` |
| `efficiencies_available` | `bool` | في ترددات غير كفؤة متعلّمة في الجدول |
| `transition_delay_us` | `unsigned int` | الحد الأدنى بين تغييرات التردد |
| `dvfs_possible_from_any_cpu` | `bool` | DVFS ممكن من أي CPU تانية |
| `boost_enabled` | `bool` | الـ boost مفعّل لهذه الـ policy |
| `boost_supported` | `bool` | الـ hardware يدعم boost |
| `transition_ongoing` | `bool` | في تغيير تردد جاري |
| `transition_lock` | `spinlock_t` | حماية `transition_ongoing` |
| `transition_wait` | `wait_queue_head_t` | انتظار نهاية الـ transition |
| `stats` | `struct cpufreq_stats *` | إحصائيات وقت الإقامة في كل تردد |
| `driver_data` | `void *` | بيانات خاصة بالـ driver |
| `cdev` | `struct thermal_cooling_device *` | الـ cooling device المرتبط |
| `nb_min` / `nb_max` | `struct notifier_block` | notifiers لتغييرات QoS |
| `cached_target_freq` | `unsigned int` | تردد مكاش مؤخرًا |
| `cached_resolved_idx` | `unsigned int` | index مكاش في الجدول |

---

#### `struct cpufreq_policy_data`

**الغرض:** نسخة مؤقتة من بيانات الـ policy بتتبعت للـ `driver->verify()` فقط — لا يحق للـ driver يعدّل فيها غير `min` و`max`.

| Field | النوع | الشرح |
|-------|-------|--------|
| `cpuinfo` | `struct cpufreq_cpuinfo` | معلومات الـ hardware |
| `freq_table` | `struct cpufreq_frequency_table *` | جدول الترددات (read-only) |
| `cpu` | `unsigned int` | رقم الـ CPU |
| `min` / `max` | `unsigned int` | الحدود القابلة للتعديل من الـ driver |

---

#### `struct cpufreq_freqs`

**الغرض:** بيانات حدث تغيير التردد، بتتبعت للـ notifiers قبل وبعد التغيير.

| Field | النوع | الشرح |
|-------|-------|--------|
| `policy` | `struct cpufreq_policy *` | الـ policy اللي فيها التغيير |
| `old` | `unsigned int` | التردد القديم (kHz) |
| `new` | `unsigned int` | التردد الجديد (kHz) |
| `flags` | `u8` | flags الـ driver (نسخة منها للـ notifier) |

---

#### `struct cpufreq_driver`

**الغرض:** واجهة الـ hardware driver — بيسجّله الـ driver عند التحميل. الـ core بيستدعي الـ function pointers الموجودة فيه.

| Field | النوع | الشرح |
|-------|-------|--------|
| `name` | `char[16]` | اسم الـ driver |
| `flags` | `u16` | combination من `CPUFREQ_*` flags |
| `driver_data` | `void *` | بيانات خاصة بالـ driver |
| `init` | `(*)(policy)` | تهيئة الـ policy (إلزامي) |
| `verify` | `(*)(policy_data)` | التحقق من صحة الـ min/max (إلزامي) |
| `setpolicy` | `(*)(policy)` | استخدمه لو الـ driver بيتحكم في الـ policy بنفسه |
| `target` | `(*)(policy, freq, relation)` | ضبط تردد محدد — deprecated |
| `target_index` | `(*)(policy, index)` | ضبط تردد باستخدام index في الجدول (المفضّل) |
| `fast_switch` | `(*)(policy, freq)` | تغيير التردد سريعًا من scheduler context |
| `adjust_perf` | `(*)(cpu, min, target, capacity)` | بديل `fast_switch` للـ drivers اللي بتتعامل مع performance levels |
| `get_intermediate` | `(*)(policy, index)` | رجّع تردد وسيط قبل القفز للـ target |
| `target_intermediate` | `(*)(policy, index)` | اضبط الـ CPU على التردد الوسيط |
| `get` | `(*)(cpu)` | ارجع التردد الحالي من الـ hardware |
| `update_limits` | `(*)(policy)` | تحديث الحدود عند إشعار الـ firmware |
| `bios_limit` | `(*)(cpu, limit)` | ارجع الحد العلوي اللي الـ BIOS بيفرضه |
| `online` | `(*)(policy)` | CPU رجعت online |
| `offline` | `(*)(policy)` | CPU هتاخد offline |
| `exit` | `(*)(policy)` | تدمير الـ policy |
| `suspend` | `(*)(policy)` | الجهاز هيدخل suspend |
| `resume` | `(*)(policy)` | الجهاز رجع من suspend |
| `ready` | `(*)(policy)` | الـ driver خلص init بالكامل |
| `attr` | `struct freq_attr **` | مصفوفة attributes لـ sysfs |
| `boost_enabled` | `bool` | boost مفعّل على مستوى الـ driver |
| `set_boost` | `(*)(policy, state)` | تفعيل/تعطيل الـ boost |
| `register_em` | `(*)(policy)` | تسجيل الـ energy model |

---

#### `struct cpufreq_governor`

**الغرض:** واجهة الـ frequency scaling governor — بيقرر إيه التردد المناسب بناءً على حمل الـ CPU.

| Field | النوع | الشرح |
|-------|-------|--------|
| `name` | `char[16]` | اسم الـ governor (مثلاً "schedutil") |
| `init` | `(*)(policy)` | ربط الـ governor بالـ policy |
| `exit` | `(*)(policy)` | فك الربط |
| `start` | `(*)(policy)` | ابدأ الـ governor (بعد الـ init) |
| `stop` | `(*)(policy)` | وقّف الـ governor |
| `limits` | `(*)(policy)` | الحدود اتغيرت — حدّث التردد |
| `show_setspeed` | `(*)(policy, buf)` | sysfs show |
| `store_setspeed` | `(*)(policy, freq)` | sysfs store |
| `governor_list` | `struct list_head` | ربط الـ governor في قائمة الـ governors المسجّلين |
| `owner` | `struct module *` | الـ module المالك |
| `flags` | `u8` | `CPUFREQ_GOV_*` flags |

---

#### `struct cpufreq_frequency_table`

**الغرض:** entry واحدة في جدول الترددات المدعومة — الجدول مصفوفة من الـ entries متنهاش بـ `CPUFREQ_TABLE_END`.

| Field | النوع | الشرح |
|-------|-------|--------|
| `flags` | `unsigned int` | BOOST, INEFFICIENT, أو لا شيء |
| `driver_data` | `unsigned int` | بيانات خاصة بالـ driver (مش بيستخدمها الـ core) |
| `frequency` | `unsigned int` | التردد بالـ kHz، أو INVALID، أو TABLE_END |

---

#### `struct freq_attr`

**الغرض:** sysfs attribute خاص بالـ cpufreq — بيعرض أو بيخزن معلومات التردد.

| Field | النوع | الشرح |
|-------|-------|--------|
| `attr` | `struct attribute` | الـ attribute الأساسي (اسم، permissions) |
| `show` | `(*)(policy, buf)` | اقرأ القيمة من الـ policy |
| `store` | `(*)(policy, buf, count)` | اكتب قيمة جديدة للـ policy |

---

#### `struct gov_attr_set`

**الغرض:** مجموعة attributes مشتركة بين كل الـ policies اللي بتستخدم نفس الـ governor (في حالة `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` مش مضبوطة).

| Field | النوع | الشرح |
|-------|-------|--------|
| `kobj` | `struct kobject` | كائن sysfs |
| `policy_list` | `struct list_head` | الـ policies المرتبطة |
| `update_lock` | `struct mutex` | حماية الـ policy_list |
| `usage_count` | `int` | عدد الـ policies المستخدمة |

---

#### `struct governor_attr`

**الغرض:** sysfs attribute خاص بالـ governor — للـ tunable parameters زي `rate_limit_us` في schedutil.

| Field | النوع | الشرح |
|-------|-------|--------|
| `attr` | `struct attribute` | الـ attribute الأساسي |
| `show` | `(*)(attr_set, buf)` | اقرأ القيمة |
| `store` | `(*)(attr_set, buf, count)` | اكتب قيمة |

---

### 2. رسم علاقات الـ Structs (ASCII)

```
                    ┌──────────────────────────────────────┐
                    │        cpufreq_driver                │
                    │  name, flags, driver_data            │
                    │  init(), verify(), target_index()    │
                    │  fast_switch(), adjust_perf()        │
                    │  get(), online(), offline(), exit()  │
                    │  attr[]  ──────────────► freq_attr[] │
                    └─────────────┬────────────────────────┘
                                  │ registers with core
                                  │ cpufreq_register_driver()
                                  ▼
                    ┌──────────────────────────────────────┐
                    │         cpufreq_policy               │◄─── per CPU cluster
                    │                                      │
                    │  cpu (managing CPU)                  │
                    │  cpus ──────► cpumask (online)       │
                    │  related_cpus► cpumask (all)         │
                    │  real_cpus ──► cpumask (present)     │
                    │                                      │
                    │  cpuinfo ───► cpufreq_cpuinfo        │
                    │               (min/max/latency)      │
                    │                                      │
                    │  min, max, cur  (kHz limits)         │
                    │                                      │
                    │  freq_table ─► cpufreq_frequency_table[]
                    │                [freq|flags|drv_data] │
                    │                [freq|flags|drv_data] │
                    │                [TABLE_END]           │
                    │                                      │
                    │  governor ──► cpufreq_governor       │
                    │               init/start/stop/limits │
                    │                                      │
                    │  governor_data (governor's private)  │
                    │                                      │
                    │  constraints ► freq_constraints (QoS)│
                    │  min_freq_req► freq_qos_request      │
                    │  max_freq_req► freq_qos_request      │
                    │                                      │
                    │  stats ─────► cpufreq_stats          │
                    │  cdev ──────► thermal_cooling_device │
                    │  clk ───────► clk                    │
                    │  kobj ──────► kobject (sysfs)        │
                    │  rwsem  (rw_semaphore)               │
                    │  transition_lock (spinlock)          │
                    │  policy_list ──► list_head           │
                    │  nb_min, nb_max (notifier_blocks)    │
                    └──────────────────────────────────────┘
                                  ▲
                    ┌─────────────┘
                    │  passed to verify()
                    │
          ┌─────────────────────────┐
          │   cpufreq_policy_data   │
          │  cpuinfo, freq_table    │
          │  cpu, min, max          │
          └─────────────────────────┘

          ┌─────────────────────────┐
          │    cpufreq_freqs        │
          │  policy ──► policy      │
          │  old, new (kHz)         │
          │  flags                  │
          └─────────────────────────┘
               │ passed to notifiers
               ▼
          ┌─────────────────────────┐
          │   notifier_block chain  │
          │  (PRECHANGE/POSTCHANGE) │
          └─────────────────────────┘

          ┌─────────────────────────┐
          │     gov_attr_set        │
          │  kobj (sysfs)           │
          │  policy_list ──► policies
          │  update_lock (mutex)    │
          └─────────────────────────┘
               │ contains
               ▼
          ┌─────────────────────────┐
          │    governor_attr        │
          │  attr, show(), store()  │
          └─────────────────────────┘
```

---

### 3. Lifecycle Diagrams

#### Driver Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    Driver Lifecycle                             │
└─────────────────────────────────────────────────────────────────┘

  module_init()
      │
      ▼
  cpufreq_register_driver(driver)
      │  ── validates flags
      │  ── stores as global cpufreq_driver
      │  ── for each CPU: calls driver->init(policy)
      │         │
      │         ▼
      │    driver->init(policy):
      │       - يملي policy->cpuinfo (min/max/latency)
      │       - يحدد policy->freq_table
      │       - يضبط policy->min, max
      │       - يسجل الـ clk
      │
      ▼
  [driver active — handles target_index / fast_switch calls]
      │
      │  CPU hotplug offline:
      ▼
  driver->offline(policy)  [optional]
      │
      ▼
  driver->exit(policy)     [cleanup policy data]
      │
      │  module_exit():
      ▼
  cpufreq_unregister_driver(driver)
      │  ── removes from system
      │  ── calls exit on remaining policies
      ▼
  [done]
```

#### Policy Lifecycle

```
  CPU comes online (hotplug or boot)
      │
      ▼
  cpufreq_add_dev()
      │
      ├─► alloc cpufreq_policy
      ├─► driver->init(policy)          ← driver fills hardware info
      ├─► cpufreq_frequency_table_cpuinfo() ← fills policy->cpuinfo from table
      ├─► cpufreq_table_validate_and_sort()
      ├─► notify CPUFREQ_CREATE_POLICY
      ├─► cpufreq_start_governor(policy)
      │       ├─► governor->init(policy)
      │       └─► governor->start(policy)
      ├─► kobject_init_and_add()        ← يظهر في sysfs
      └─► policy active
              │
              │  [frequency change requests]
              │
              ▼
         [see call flow below]
              │
              │  CPU goes offline:
              ▼
         cpufreq_stop_governor(policy)
              ├─► governor->stop(policy)
              └─► governor->exit(policy)
         notify CPUFREQ_REMOVE_POLICY
         driver->offline(policy)  or  driver->exit(policy)
         kobject_put()
         [policy freed if no more CPUs, else kept inactive]
```

#### Governor Lifecycle

```
  insmod governor_module
      │
      ▼
  cpufreq_register_governor(governor)
      │  ── adds to governor_list
      ▼
  [available for policies]

  policy selects governor (sysfs write or default):
      │
      ▼
  cpufreq_start_governor(policy)
      ├─► governor->init(policy)
      │       ── allocates governor-private data
      │       ── links gov_attr_set to policy
      └─► governor->start(policy)
              ── يبدأ الـ sampling / scheduler hooks

  [limits change]:
      └─► governor->limits(policy)

  [policy deactivated or governor changed]:
      ├─► cpufreq_stop_governor()
      │       ├─► governor->stop(policy)
      │       └─► governor->exit(policy)
      └─► [new governor starts if replacing]

  rmmod governor_module:
      └─► cpufreq_unregister_governor(governor)
```

---

### 4. Call Flow Diagrams

#### تغيير التردد العادي (target_index path)

```
  governor decides new freq needed
      │
      ▼
  cpufreq_driver_target(policy, target_freq, relation)
      │
      ▼
  __cpufreq_driver_target(policy, target_freq, relation)
      │
      ├─► cpufreq_frequency_table_target()
      │       ── يحدد الـ index في freq_table حسب relation
      │       ── RELATION_L: أقل تردد ≥ target
      │       ── RELATION_H: أعلى تردد ≤ target
      │       ── RELATION_C: أقرب تردد
      │       ── RELATION_E: يفضل efficient freq
      │
      ├─► cpufreq_freq_transition_begin(policy, freqs)
      │       ├─► spin_lock(transition_lock)
      │       ├─► policy->transition_ongoing = true
      │       ├─► spin_unlock
      │       └─► notify CPUFREQ_PRECHANGE → all notifier_blocks
      │
      ├─► driver->target_index(policy, index)
      │       ── الـ hardware بيتغير فعلاً هنا
      │       ── (مثلاً: كتابة في MMIO register أو SCMI message)
      │
      └─► cpufreq_freq_transition_end(policy, freqs, failed)
              ├─► notify CPUFREQ_POSTCHANGE → all notifier_blocks
              ├─► cpufreq_stats_record_transition()
              ├─► spin_lock(transition_lock)
              ├─► policy->transition_ongoing = false
              ├─► wake_up(transition_wait)
              └─► spin_unlock
```

#### Fast Switch Path (من scheduler context)

```
  schedutil governor, scheduler tick
      │
      ▼
  cpufreq_driver_fast_switch(policy, target_freq)
      │  ── NO notifiers, NO sleeping locks
      │  ── runs in IRQ-disabled context
      ▼
  driver->fast_switch(policy, target_freq)
      │  ── بيكتب في hardware register مباشرة
      ▼
  returns new freq (as set by hardware)
      │
      ▼
  arch_set_freq_scale(cpus, cur_freq, max_freq)
      └─► يحدّث الـ frequency invariance scale للـ scheduler
```

#### adjust_perf Path (schedutil مع performance levels)

```
  schedutil، عارف الـ performance level المطلوب
      │
      ▼
  cpufreq_driver_adjust_perf(cpu, min_perf, target_perf, capacity)
      │
      ▼
  driver->adjust_perf(cpu, min_perf, target_perf, capacity)
      │  ── يرسل performance hints للـ firmware (مثلاً SCMI perf)
      │  ── بدون تحويل لـ kHz
      ▼
  [hardware adjusts internally]
```

#### Intermediate Frequency Path

```
  driver يريد مرور بتردد وسيط (مثلاً لاستقرار PLL)
      │
      ▼
  __cpufreq_driver_target()
      │
      ├─► driver->get_intermediate(policy, index)
      │       ── يرجع التردد الوسيط (أو 0 لو مش محتاجه)
      │
      ├─► [لو رجع != 0]:
      │       ├─► notify PRECHANGE (old → intermediate)
      │       ├─► driver->target_intermediate(policy, index)
      │       └─► notify POSTCHANGE (old → intermediate)
      │
      ├─► notify PRECHANGE (intermediate → new)
      ├─► driver->target_index(policy, index)
      └─► notify POSTCHANGE (intermediate → new)
```

#### Policy Limits Update Path

```
  QoS request changed  /  sysfs write  /  firmware notification
      │
      ▼
  cpufreq_update_policy(cpu)  or  refresh_frequency_limits(policy)
      │
      ├─► down_write(policy->rwsem)
      ├─► driver->verify(policy_data)
      │       ── يتحقق إن min/max في حدود الـ hardware
      │       ── يعدّل لو لزم
      ├─► policy->min = policy_data->min
      ├─► policy->max = policy_data->max
      ├─► governor->limits(policy)
      │       ── الـ governor بيطبق الحدود الجديدة
      │       ── قد يطلب تغيير تردد فوري
      └─► up_write(policy->rwsem)
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks الموجودة في الـ Policy

```
┌────────────────────────────────────────────────────────────────┐
│ Lock               │ النوع            │ بيحمي إيه             │
├────────────────────┼──────────────────┼───────────────────────┤
│ policy->rwsem      │ rw_semaphore     │ كل بيانات الـ policy  │
│ policy->transition_lock │ spinlock    │ transition_ongoing    │
│ gov_attr_set->update_lock │ mutex     │ policy_list في governor│
└────────────────────────────────────────────────────────────────┘
```

#### قواعد الـ `rwsem`

```
  قراءة من الـ policy:
      down_read(&policy->rwsem)
          ... read fields ...
      up_read(&policy->rwsem)

  كتابة / تعديل / CPU hotplug:
      down_write(&policy->rwsem)
          ... modify fields ...
      up_write(&policy->rwsem)

  الـ Kernel بيوفر macros جاهزة:
      guard(cpufreq_policy_write)(policy)  ← scoped write lock
      guard(cpufreq_policy_read)(policy)   ← scoped read lock
```

#### ترتيب الـ Locks (Lock Ordering)

```
  لازم تمسك الـ locks بالترتيب ده دايمًا:

  1. policy->rwsem  (أعلى مستوى — sleepable)
      │
      └─► 2. policy->transition_lock  (spinlock — non-sleepable)
                  │
                  └─► [لا locks تانية جوّاه]

  3. gov_attr_set->update_lock  (mutex — مستقل)
```

#### الـ Fast Switch — بدون Locks

**الـ** `fast_switch` **مش بيستخدم أي locks** — بيتنفذ في سياق الـ scheduler بدون أي sleeping أو notification chains. ده بيتطلب إن الـ driver يضمن:
- الـ `freq_table` مش بتتعدّل أثناء التشغيل.
- الـ hardware يقبل الكتابة من أي CPU في الـ policy.

#### الـ `transition_lock` بالتفصيل

```c
/* بيحمي transition_ongoing flag */
spin_lock(&policy->transition_lock);
if (policy->transition_ongoing) {
    /* في transition جاري — انتظر */
    spin_unlock(&policy->transition_lock);
    wait_event(policy->transition_wait,
               !policy->transition_ongoing);
    spin_lock(&policy->transition_lock);
}
policy->transition_ongoing = true;
policy->transition_task = current;
spin_unlock(&policy->transition_lock);
```

#### الـ `gov_attr_set->update_lock`

**الـ** `mutex` **ده** بيحمي قائمة الـ policies في حالة الـ governor المشترك (لما `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` مش مضبوطة). بيتمسك أثناء:
- `gov_attr_set_get()` — إضافة policy جديدة
- `gov_attr_set_put()` — إزالة policy
- sysfs `store()` operations على الـ governor tunables
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Policy Access & Reference Counting

| Function | الغرض |
|---|---|
| `cpufreq_cpu_get_raw(cpu)` | جيب الـ policy بدون ref-count |
| `cpufreq_cpu_policy(cpu)` | جيب الـ policy مع lock بس بدون kobject ref |
| `cpufreq_cpu_get(cpu)` | جيب الـ policy وزود الـ kobject refcount |
| `cpufreq_cpu_put(policy)` | نزّل الـ refcount وافرج عن الـ policy |
| `policy_is_inactive(policy)` | هل الـ policy مفيهاش CPUs online؟ |
| `policy_is_shared(policy)` | هل أكتر من CPU بيشارك الـ policy؟ |

#### Frequency Query

| Function | الغرض |
|---|---|
| `cpufreq_get(cpu)` | قراءة الـ freq الحالية من الـ hardware |
| `cpufreq_quick_get(cpu)` | قراءة `policy->cur` بسرعة (cached) |
| `cpufreq_quick_get_max(cpu)` | قراءة `policy->max` |
| `cpufreq_get_hw_max_freq(cpu)` | قراءة `cpuinfo.max_freq` |
| `cpufreq_get_pressure(cpu)` | قراءة الـ thermal pressure الخاصة بالـ cpufreq |

#### Driver Registration

| Function | الغرض |
|---|---|
| `cpufreq_register_driver(drv)` | سجّل الـ cpufreq driver |
| `cpufreq_unregister_driver(drv)` | شيل تسجيل الـ driver |
| `cpufreq_driver_test_flags(flags)` | اختبر flags الـ driver الحالي |
| `cpufreq_get_current_driver()` | اسم الـ driver الحالي |
| `cpufreq_get_driver_data()` | جيب `driver_data` الـ private |

#### Frequency Switching (Driver-Facing)

| Function | الغرض |
|---|---|
| `cpufreq_driver_target(policy, freq, rel)` | غيّر الـ freq مع notifiers (مع lock) |
| `__cpufreq_driver_target(policy, freq, rel)` | غيّر الـ freq بدون lock الخارجي |
| `cpufreq_driver_fast_switch(policy, freq)` | fast-path من الـ scheduler thread |
| `cpufreq_driver_adjust_perf(cpu, min, target, cap)` | ضبط الأداء بدون freq lookup |
| `cpufreq_driver_resolve_freq(policy, freq)` | حوّل freq إلى أقرب قيمة في الـ table |
| `cpufreq_policy_transition_delay_us(policy)` | minimum delay بين transitions |

#### Transition Notifications

| Function | الغرض |
|---|---|
| `cpufreq_freq_transition_begin(policy, freqs)` | أبعت `CPUFREQ_PRECHANGE` notifier |
| `cpufreq_freq_transition_end(policy, freqs, failed)` | أبعت `CPUFREQ_POSTCHANGE` notifier |
| `cpufreq_register_notifier(nb, list)` | سجّل في transition أو policy notifier chain |
| `cpufreq_unregister_notifier(nb, list)` | إلغاء تسجيل الـ notifier |

#### Governor Management

| Function | الغرض |
|---|---|
| `cpufreq_register_governor(gov)` | سجّل governor في الـ kernel |
| `cpufreq_unregister_governor(gov)` | شيل تسجيل الـ governor |
| `cpufreq_start_governor(policy)` | ابدأ الـ governor على الـ policy |
| `cpufreq_stop_governor(policy)` | وقّف الـ governor |
| `cpufreq_default_governor()` | جيب الـ governor الافتراضي |
| `cpufreq_fallback_governor()` | جيب الـ fallback governor |
| `sugov_is_governor(policy)` | هل schedutil هو الـ governor الحالي؟ |

#### Governor Attribute Set

| Function | الغرض |
|---|---|
| `gov_attr_set_init(attr_set, list_node)` | initialize الـ attr_set لأول policy |
| `gov_attr_set_get(attr_set, list_node)` | أضف policy جديدة للـ attr_set |
| `gov_attr_set_put(attr_set, list_node)` | شيل policy من الـ attr_set |

#### Frequency Table Helpers

| Function | الغرض |
|---|---|
| `cpufreq_frequency_table_cpuinfo(policy)` | اضبط `cpuinfo.min/max_freq` من الـ table |
| `cpufreq_frequency_table_verify(policy)` | validate وclamp الـ min/max |
| `cpufreq_generic_frequency_table_verify(policy)` | generic version للـ verify |
| `cpufreq_frequency_table_target(policy, freq, min, max, rel)` | جيب index من الـ table |
| `cpufreq_frequency_table_get_index(policy, freq)` | جيب index لـ freq محددة |
| `cpufreq_table_validate_and_sort(policy)` | validate الـ table وSort لو لزم |
| `cpufreq_table_count_valid_entries(policy)` | عدّ الـ entries الصحيحة |
| `cpufreq_table_set_inefficient(policy, freq)` | علّم freq بـ `CPUFREQ_INEFFICIENT_FREQ` |
| `cpufreq_table_index_unsorted(policy, freq, min, max, rel)` | بحث في table مش مرتبة |

#### Power Management & Suspend

| Function | الغرض |
|---|---|
| `cpufreq_suspend()` | suspend كل الـ policies |
| `cpufreq_resume()` | استأنف كل الـ policies |
| `cpufreq_generic_suspend(policy)` | اضبط الـ `suspend_freq` قبل النوم |

#### Boost

| Function | الغرض |
|---|---|
| `cpufreq_boost_enabled()` | هل الـ boost مفعّل globally؟ |
| `cpufreq_boost_set_sw(policy, state)` | فعّل/عطّل الـ boost عن طريق الـ software |

#### Misc Helpers

| Function | الغرض |
|---|---|
| `cpufreq_verify_within_limits(policy, min, max)` | clamp min/max في الـ policy_data |
| `cpufreq_verify_within_cpu_limits(policy)` | نفس الفكرة بس بـ cpuinfo limits |
| `cpufreq_enable_fast_switch(policy)` | فعّل الـ fast_switch_enabled flag |
| `cpufreq_disable_fast_switch(policy)` | عطّل الـ fast_switch_enabled flag |
| `cpufreq_policy_apply_limits(policy)` | طبّق min/max على الـ cur freq |
| `cpufreq_supports_freq_invariance()` | هل الـ kernel يدعم freq invariance؟ |
| `cpufreq_scale(old, div, mult)` | حساب `old * mult / div` بأمان على 32/64-bit |
| `cpufreq_show_cpus(mask, buf)` | اكتب الـ cpumask في الـ sysfs buffer |
| `get_cpu_idle_time(cpu, wall, io_busy)` | جيب wقت الـ idle للـ governor sampling |
| `cpufreq_update_policy(cpu)` | أعد تطبيق الـ policy |
| `cpufreq_update_limits(cpu)` | بلّغ الـ driver بتغيير الـ firmware limits |
| `refresh_frequency_limits(policy)` | أعد حساب الـ freq limits من الـ QoS |
| `arch_freq_get_on_cpu(cpu)` | اقرأ الـ freq من الـ arch-specific register |
| `arch_set_freq_scale(cpus, cur, max)` | اضبط الـ freq invariance scale للـ sched |
| `parse_perf_domain(cpu, list_name, cell_name, args)` | parse الـ OPP domain من الـ DT |
| `of_perf_domain_get_sharing_cpumask(pcpu, ...)` | احسب الـ CPUs اللي بيشاركوا نفس OPP domain |
| `cpufreq_generic_get(cpu)` | قراءة الـ freq من الـ clk framework |
| `cpufreq_generic_init(policy, table, latency)` | initialize policy بـ freq table وlatency |
| `cpufreq_register_em_with_opp(policy)` | سجّل الـ energy model من الـ OPP table |
| `cpufreq_ready_for_eas(cpu_mask)` | هل الـ policy جاهز لـ EAS؟ |
| `have_governor_per_policy()` | هل الـ driver يدعم per-policy governors؟ |
| `has_target_index()` | هل الـ driver عنده `target_index` callback؟ |
| `get_governor_parent_kobj(policy)` | جيب الـ kobject الأب لـ sysfs directory الـ governor |
| `disable_cpufreq()` | عطّل الـ cpufreq subsystem كله |

---

### Group 1: Policy Reference Counting & Access

الـ **`cpufreq_policy`** بتتاخد وبترجع عبر مجموعة functions بتحمي الـ object من الـ race conditions. الـ kobject reference counting بيضمن إن الـ policy متتفراش وفي استخدامها.

---

#### `cpufreq_cpu_get_raw`

```c
struct cpufreq_policy *cpufreq_cpu_get_raw(unsigned int cpu);
```

بترجع الـ policy الخاصة بالـ CPU من الـ per-CPU data structure مباشرة بدون ما تزود الـ refcount ولا تاخد أي lock. بتستخدمها في contexts إنت متأكد فيها إن الـ policy موجودة ومحمية بـ lock تاني.

- **`cpu`**: رقم الـ CPU.
- **Return**: pointer للـ policy أو `NULL` لو مفيش.
- **Key details**: لا lock ولا refcount — dangerous في most contexts. بتتستخدم داخليًا في الـ core أو لما تكون تحت `policy->rwsem`.

---

#### `cpufreq_cpu_policy`

```c
struct cpufreq_policy *cpufreq_cpu_policy(unsigned int cpu);
```

بترجع الـ policy وبتضمن إنها valid من خلال فحص `policy->cpus`، بس بدون زيادة kobject refcount. الفرق عن `cpufreq_cpu_get_raw` إنها بتتحقق من الـ validity بشكل أكبر.

- **`cpu`**: رقم الـ CPU.
- **Return**: pointer للـ policy أو `NULL`.
- **Key details**: مش هتحتاج تعمل `cpufreq_cpu_put` بعديها.

---

#### `cpufreq_cpu_get`

```c
struct cpufreq_policy *cpufreq_cpu_get(unsigned int cpu);
```

**الـ standard way** للحصول على policy. بتزود الـ kobject refcount عبر `kobject_get()` فبتضمن إن الـ policy متتفراش طول ما إنت شاغلها. لازم تعمل `cpufreq_cpu_put` بعد الاستخدام.

- **`cpu`**: رقم الـ CPU.
- **Return**: pointer للـ policy مع refcount مزوّد، أو `NULL`.
- **Key details**: safe للاستخدام من أي context. بتبقى محتاجها لما تعمل access من خارج الـ cpufreq core (مثلًا thermal drivers).
- **Who calls it**: thermal cooling devices، OPP framework، external consumers.

---

#### `cpufreq_cpu_put`

```c
void cpufreq_cpu_put(struct cpufreq_policy *policy);
```

بتنزّل الـ kobject refcount. لو الـ refcount وصل صفر بتشغّل cleanup الـ kobject. **لازم تتكلم مرة واحدة بالظبط مقابل كل `cpufreq_cpu_get`**.

- **`policy`**: الـ policy اللي خدتها بـ `cpufreq_cpu_get`.
- **Return**: void.
- **Key details**: بتستخدم الـ `DEFINE_FREE(put_cpufreq_policy, ...)` macro للـ scope-based cleanup.

---

#### `policy_is_inactive` / `policy_is_shared`

```c
static inline bool policy_is_inactive(struct cpufreq_policy *policy);
static inline bool policy_is_shared(struct cpufreq_policy *policy);
```

**`policy_is_inactive`**: بتفحص `cpumask_empty(policy->cpus)` — يعني مفيش CPUs online على الـ policy دي.

**`policy_is_shared`**: بتفحص لو في أكتر من CPU في `policy->cpus` — يعني الـ clock domain بيتشارك. مهمة لـ governors يحتاجوا يعرفوا هل لازم ينسقوا.

---

### Group 2: Frequency Query Functions

بتيجي الـ query functions دي في مستويين: **slow path** (يتكلم الـ hardware) و**fast path** (يقرأ الـ cached value).

---

#### `cpufreq_get`

```c
unsigned int cpufreq_get(unsigned int cpu);
```

بتقرأ الـ **current frequency** من الـ hardware عبر `cpufreq_driver->get()` callback. بطيئة نسبيًا لأنها بتتكلم مع الـ hardware.

- **`cpu`**: رقم الـ CPU.
- **Return**: الـ freq بالـ kHz أو `0` لو حصل error أو مفيش driver.
- **Key details**: بتاخد الـ `policy->rwsem` للقراءة. مش مناسبة للـ hot path.

---

#### `cpufreq_quick_get` / `cpufreq_quick_get_max`

```c
unsigned int cpufreq_quick_get(unsigned int cpu);
unsigned int cpufreq_quick_get_max(unsigned int cpu);
```

**`cpufreq_quick_get`**: بترجع `policy->cur` — الـ cached frequency. أسرع بكتير من `cpufreq_get` لأنها مش بتتكلم الـ hardware.

**`cpufreq_quick_get_max`**: بترجع `policy->max`.

- **Return**: الـ freq بالـ kHz أو `0`.
- **Key details**: بتاخد `policy->rwsem` read lock بس. مناسبة لـ userspace informational reads.

---

#### `cpufreq_get_hw_max_freq`

```c
unsigned int cpufreq_get_hw_max_freq(unsigned int cpu);
```

بترجع `cpuinfo.max_freq` — الـ **hardware maximum** بغض النظر عن الـ policy limits الحالية. مهمة لـ thermal drivers لما يحتاجوا يعرفوا الـ ceiling الحقيقي.

---

#### `cpufreq_get_pressure`

```c
static inline unsigned long cpufreq_get_pressure(int cpu);
```

بتقرأ الـ `cpufreq_pressure` الـ per-CPU variable بـ `READ_ONCE`. الـ pressure دي بيضبطها الـ thermal framework لما يعمل frequency capping. الـ scheduler بيستخدمها لحساب الـ capacity.

---

### Group 3: Driver Registration & Management

الـ **`cpufreq_driver`** هو الـ heart بتاعت الـ subsystem. التسجيل بيحصل مرة واحدة وبيعمل enumerate لكل الـ CPUs.

---

#### `cpufreq_register_driver`

```c
int cpufreq_register_driver(struct cpufreq_driver *driver_data);
```

الـ **أهم function** في الـ subsystem. بتسجّل الـ platform cpufreq driver وتبدأ الـ policy creation لكل CPU. بتمر بـ flow معقد:

```
cpufreq_register_driver()
  ├── تفحص الـ mandatory callbacks (init, verify)
  ├── تسجّل الـ driver في الـ cpufreq_driver pointer
  ├── subsys_interface_register()
  │     └── لكل CPU: cpufreq_add_dev()
  │              └── cpufreq_online()
  │                   ├── driver->init(policy)
  │                   ├── cpufreq_table_validate_and_sort()
  │                   ├── cpufreq_start_governor()
  │                   └── driver->ready(policy)
  └── ترجع 0 أو error code
```

- **`driver_data`**: pointer للـ `cpufreq_driver` struct المعبّي.
- **Return**: `0` أو negative errno.
- **Key details**: لازم `driver->init` و`driver->verify` موجودين. الـ function دي blocking وبتاخد وقت.
- **Who calls it**: كل cpufreq platform drivers من الـ module init أو الـ probe function.

---

#### `cpufreq_unregister_driver`

```c
void cpufreq_unregister_driver(struct cpufreq_driver *driver_data);
```

بتعكس `cpufreq_register_driver`. بتعمل offline لكل الـ policies وبتمسح الـ sysfs entries. blocking حتى ما تخلص كل الـ cleanup.

- **`driver_data`**: نفس الـ pointer اللي اتسجّل بيه.
- **Key details**: بتتكلم `driver->exit` لكل policy. بعدها الـ `cpufreq_driver` global pointer بيبقى `NULL`.

---

#### `cpufreq_driver_test_flags`

```c
bool cpufreq_driver_test_flags(u16 flags);
```

بتفحص لو الـ driver الحالي عنده الـ flags دي متحطة. مفيدة بدل ما تاخد reference للـ driver وتفحصه يدويًا.

---

#### `cpufreq_get_current_driver` / `cpufreq_get_driver_data`

```c
const char *cpufreq_get_current_driver(void);
void *cpufreq_get_driver_data(void);
```

accessors بسيطة للـ driver metadata. الأولى بترجع اسم الـ driver، الثانية بترجع `driver->driver_data` الـ private.

---

#### `cpufreq_thermal_control_enabled`

```c
static inline int cpufreq_thermal_control_enabled(struct cpufreq_driver *drv);
```

بتفحص لو الـ `CONFIG_CPU_THERMAL` ممكّن **و**الـ driver عنده `CPUFREQ_IS_COOLING_DEV` flag. بيستخدمها الـ cpufreq core لما يقرر يسجّل الـ policy كـ thermal cooling device تلقائيًا.

---

### Group 4: Frequency Switching — Driver-Facing API

الـ group دي هي الـ **hot path** بتاع الـ cpufreq subsystem. الـ governor بيكلّم الـ functions دي عشان يغيّر الـ frequency.

---

#### `cpufreq_driver_target`

```c
int cpufreq_driver_target(struct cpufreq_policy *policy,
                          unsigned int target_freq,
                          unsigned int relation);
```

الـ **standard way** لطلب تغيير الـ frequency. بتاخد الـ `policy->rwsem` write lock وتكلّم `__cpufreq_driver_target`.

- **`policy`**: الـ policy المستهدفة.
- **`target_freq`**: الـ freq المطلوبة بالـ kHz.
- **`relation`**: `CPUFREQ_RELATION_L/H/C` مع optional `CPUFREQ_RELATION_E`.
- **Return**: `0` أو negative errno.
- **Key details**: بتبعت الـ PRECHANGE/POSTCHANGE notifiers عبر الـ transition begin/end. blocking function.

---

#### `__cpufreq_driver_target`

```c
int __cpufreq_driver_target(struct cpufreq_policy *policy,
                            unsigned int target_freq,
                            unsigned int relation);
```

نفس `cpufreq_driver_target` بس **بدون** أخد الـ lock — caller لازم يكون شايل الـ `rwsem` write lock. بيستخدمها الـ core internally وبتستخدمها `cpufreq_policy_apply_limits`.

```
__cpufreq_driver_target()
  ├── clamp target_freq في policy->min .. policy->max
  ├── لو target_freq == policy->cur → skip
  ├── لو driver->target_index:
  │     ├── حوّل الـ freq لـ index
  │     ├── cpufreq_freq_transition_begin()
  │     ├── driver->target_index(policy, index)
  │     └── cpufreq_freq_transition_end()
  └── لو driver->target (deprecated):
        └── driver->target(policy, freq, relation)
```

---

#### `cpufreq_driver_fast_switch`

```c
unsigned int cpufreq_driver_fast_switch(struct cpufreq_policy *policy,
                                        unsigned int target_freq);
```

الـ **scheduler fast path**. بيتكلمها الـ `schedutil` governor من الـ scheduler context — يعني ممكن تيجي من interrupt context أو مع الـ rq lock محجوز. **لا locks، لا notifiers، لا sleeping**.

- **`policy`**: الـ policy المستهدفة.
- **`target_freq`**: الـ freq المطلوبة بالـ kHz.
- **Return**: الـ freq الحقيقية اللي اتحطت أو `0` لو فشل.
- **Key details**: يشترط إن `policy->fast_switch_enabled == true`. الـ driver لازم يكون implementّ الـ `driver->fast_switch` callback. مش بتبعت notifiers.
- **Who calls it**: `schedutil` governor فقط.

---

#### `cpufreq_driver_adjust_perf`

```c
void cpufreq_driver_adjust_perf(unsigned int cpu,
                                unsigned long min_perf,
                                unsigned long target_perf,
                                unsigned long capacity);
```

بديل لـ `fast_switch` للـ drivers اللي بتشتغل بـ **performance levels** (مش kHz). بيمرر hints للـ hardware عن الـ minimum performance المطلوب والـ target. الـ `adjust_perf` callback موجود فقط لو الـ `fast_switch` موجود.

- **`cpu`**: CPU number.
- **`min_perf`**: أدنى مستوى أداء مقبول.
- **`target_perf`**: المستوى المستهدف.
- **`capacity`**: الـ capacity الكلية للـ CPU (للـ normalization).
- **Who calls it**: `schedutil` governor في الـ fast path.

---

#### `cpufreq_driver_resolve_freq`

```c
unsigned int cpufreq_driver_resolve_freq(struct cpufreq_policy *policy,
                                         unsigned int target_freq);
```

بتحوّل الـ target frequency لأقرب قيمة موجودة في الـ freq table. بتستخدم `CPUFREQ_RELATION_L` وبتعمل cache للنتيجة في `policy->cached_target_freq` و`policy->cached_resolved_idx` لتسريع الطلبات المتكررة.

---

#### `cpufreq_policy_transition_delay_us`

```c
unsigned int cpufreq_policy_transition_delay_us(struct cpufreq_policy *policy);
```

بتحسب الـ minimum delay (بالـ microseconds) بين frequency transitions. بترجع `policy->transition_delay_us` لو موجود، وإلا بتحسب من الـ `transition_latency`. الـ governor بيستخدمها عشان معيدش الطلب بسرعة أكبر من قدرة الـ hardware.

---

### Group 5: Transition Notifications

الـ **notifier chain** بتضمن إن أي subsystem محتاج يعرف بتغيير الـ frequency (مثلًا `loops_per_jiffy` recalculation) بياخد الـ notification في الوقت الصح.

---

#### `cpufreq_freq_transition_begin`

```c
void cpufreq_freq_transition_begin(struct cpufreq_policy *policy,
                                   struct cpufreq_freqs *freqs);
```

لازم تتكلم **قبل** ما تبعت أي أوامر للـ hardware بتغيير الـ frequency. بتبعت `CPUFREQ_PRECHANGE` لكل الـ listeners. بتعمل synchronization مع أي transitions موجودة عبر `policy->transition_lock` و`policy->transition_wait`.

- **`policy`**: الـ policy الجاري تغييرها.
- **`freqs`**: struct فيها `old` و`new` frequencies وغيرهم.
- **Key details**: بتضبط `policy->transition_ongoing = true`. لو الـ driver بيستخدم `CPUFREQ_ASYNC_NOTIFICATION` ممكن الـ caller يكون different thread.

---

#### `cpufreq_freq_transition_end`

```c
void cpufreq_freq_transition_end(struct cpufreq_policy *policy,
                                 struct cpufreq_freqs *freqs,
                                 int transition_failed);
```

لازم تتكلم **بعد** ما الـ hardware يخلص التغيير. بتبعت `CPUFREQ_POSTCHANGE`. لو `transition_failed != 0`، الـ `freqs->new` بيتفرض يكون نفس `freqs->old`.

- **`transition_failed`**: `0` لو نجح، non-zero لو فشل.
- **Key details**: بتضبط `policy->transition_ongoing = false` وبتصحّي الـ waiters. بتحدّث `policy->cur`.

---

#### `cpufreq_register_notifier` / `cpufreq_unregister_notifier`

```c
int cpufreq_register_notifier(struct notifier_block *nb, unsigned int list);
int cpufreq_unregister_notifier(struct notifier_block *nb, unsigned int list);
```

التسجيل في إحدى الـ notifier chains:
- `CPUFREQ_TRANSITION_NOTIFIER`: للـ frequency changes (PRECHANGE/POSTCHANGE).
- `CPUFREQ_POLICY_NOTIFIER`: لإنشاء/حذف الـ policies (CREATE_POLICY/REMOVE_POLICY).

- **`nb`**: الـ `notifier_block` بـ callback function.
- **`list`**: إيه الـ chain؟ `CPUFREQ_TRANSITION_NOTIFIER` أو `CPUFREQ_POLICY_NOTIFIER`.
- **Return**: `0` أو negative errno.

---

### Group 6: Governor Registration & Lifecycle

الـ **`cpufreq_governor`** struct بيحدد الـ algorithm المسؤول عن اختيار الـ target frequency.

---

#### `cpufreq_register_governor`

```c
int cpufreq_register_governor(struct cpufreq_governor *governor);
```

بتضيف الـ governor لـ `cpufreq_governor_list`. بعدها الـ governor يبقى متاح للـ userspace لما يكتب اسمه في `scaling_governor` sysfs file. الـ macro الـ `cpufreq_governor_init` بيوظّف الـ function دي تلقائيًا.

- **`governor`**: الـ governor struct المحتوي على الـ callbacks.
- **Return**: `0` أو `-EBUSY` لو اسم مكرر.

---

#### `cpufreq_unregister_governor`

```c
void cpufreq_unregister_governor(struct cpufreq_governor *governor);
```

بتشيل الـ governor من الـ list. لو في policies بتستخدمه، بيبقى لازم تتعمل fallback. الـ macro الـ `cpufreq_governor_exit` بيوظّفها.

---

#### `cpufreq_start_governor`

```c
int cpufreq_start_governor(struct cpufreq_policy *policy);
```

بتبدأ الـ governor على الـ policy:
1. بتكلّم `governor->init(policy)` لو محتاج initialization.
2. بتكلّم `governor->start(policy)`.
3. لو `CPUFREQ_GOV_STRICT_TARGET` موجود، بتضبط `policy->strict_target = true`.

- **Return**: `0` أو error من الـ governor callbacks.
- **Who calls it**: الـ cpufreq core لما يعمل policy online أو لما يتغير الـ governor.

---

#### `cpufreq_stop_governor`

```c
void cpufreq_stop_governor(struct cpufreq_policy *policy);
```

بتوقّف الـ governor بـ `governor->stop(policy)`. بتتكلم قبل offline أو قبل تغيير الـ governor.

---

#### `cpufreq_policy_apply_limits`

```c
static inline void cpufreq_policy_apply_limits(struct cpufreq_policy *policy);
```

بتفحص لو الـ `policy->cur` خرج عن الـ min/max limits وبتعمل `__cpufreq_driver_target` بالـ relation المناسبة (`CPUFREQ_RELATION_HE` للـ max, `CPUFREQ_RELATION_LE` للـ min). بتتكلم من `governor->limits()` callback.

---

#### `gov_attr_set_init` / `gov_attr_set_get` / `gov_attr_set_put`

```c
void gov_attr_set_init(struct gov_attr_set *attr_set, struct list_head *list_node);
void gov_attr_set_get(struct gov_attr_set *attr_set, struct list_head *list_node);
unsigned int gov_attr_set_put(struct gov_attr_set *attr_set, struct list_head *list_node);
```

**الـ `gov_attr_set`** هو الـ shared sysfs kobject للـ governor. لما الـ governor بيشتغل على multiple policies (مثلًا `schedutil` على كل الـ clusters)، الـ attr_set بيجمعهم تحت kobject واحد أو منفصل بناءً على `CPUFREQ_HAVE_GOVERNOR_PER_POLICY`.

- **`init`**: بتعمل initialize الـ kobject وبتضيف أول policy.
- **`get`**: بتزود الـ usage count وبتضيف policy جديدة.
- **`put`**: بتنزل الـ usage count. لو وصل صفر بترجع `0` وبتعمل cleanup للـ kobject.

---

### Group 7: Frequency Table Helpers — Core Search Functions

الـ frequency table search هو الـ hot code path اللي الـ governor بيمر بيه في كل decision. الـ kernel بيوفّر 6 specialized search functions (ascending/descending × lower/higher/closest) وبيوحّدهم في interface واحد.

---

#### `cpufreq_frequency_table_target`

```c
static inline int cpufreq_frequency_table_target(struct cpufreq_policy *policy,
                                                  unsigned int target_freq,
                                                  unsigned int min,
                                                  unsigned int max,
                                                  unsigned int relation);
```

الـ **central dispatcher** للبحث في الـ frequency table. بتفحص الـ `freq_table_sorted` flag وبتوجّه الطلب للـ function المناسبة.

```
cpufreq_frequency_table_target()
  ├── لو UNSORTED → cpufreq_table_index_unsorted()
  ├── strip RELATION_E flag واحتفظ بـ efficiencies flag
  ├── switch(relation):
  │     case RELATION_L → find_index_l()
  │     case RELATION_H → find_index_h()
  │     case RELATION_C → find_index_c()
  ├── لو النتيجة مش في الـ limits وكانت efficiencies=true:
  │     efficiencies = false → retry
  └── رجّع الـ index
```

- **`relation`**: combination من `CPUFREQ_RELATION_L/H/C` مع optional `CPUFREQ_RELATION_E`.
- **Return**: index في الـ freq_table أو negative لو فشل.
- **Key details**: الـ retry mechanism بيضمن إنك دايمًا بتلاقي frequency حتى لو مفيش efficient frequency في الـ range.

---

#### `cpufreq_table_find_index_al` / `cpufreq_table_find_index_dl`

```c
/* Ascending  */ static inline int cpufreq_table_find_index_al(...);
/* Descending */ static inline int cpufreq_table_find_index_dl(...);
```

**"L" = Lowest freq at or above target** — بتدور على أصغر frequency أكبر من أو تساوي الـ target.

- الـ ascending version: بتمشي للأمام وبترجع أول freq >= target.
- الـ descending version: أكتر complexity لأن القيم رايحة للتحت، فبتحتاج تتتبع الـ best candidate.
- **`efficiencies`**: لو `true`، بتتخطى الـ entries المعلّمة بـ `CPUFREQ_INEFFICIENT_FREQ`.

---

#### `cpufreq_table_find_index_ah` / `cpufreq_table_find_index_dh`

```c
/* Ascending  */ static inline int cpufreq_table_find_index_ah(...);
/* Descending */ static inline int cpufreq_table_find_index_dh(...);
```

**"H" = Highest freq at or below target** — بتدور على أكبر frequency أقل من أو تساوي الـ target. بيستخدمها الـ `CPUFREQ_RELATION_H` relation — يعني "اديني أعلى performance ملوش على الـ target".

---

#### `cpufreq_table_find_index_ac` / `cpufreq_table_find_index_dc`

```c
/* Ascending  */ static inline int cpufreq_table_find_index_ac(...);
/* Descending */ static inline int cpufreq_table_find_index_dc(...);
```

**"C" = Closest freq** — بتحسب `|freq - target|` وبترجع الأقرب. لو كانوا متساويين في البُعد، بتختار اللي أعلى (conservative). بيستخدمها `CPUFREQ_RELATION_C`.

---

#### `find_index_l` / `find_index_h` / `find_index_c`

```c
static inline int find_index_l(struct cpufreq_policy *policy,
                                unsigned int target_freq,
                                unsigned int min, unsigned int max,
                                bool efficiencies);
```

الـ **intermediate dispatchers** اللي بيعملوا `clamp_val(target_freq, min, max)` الأول وبعدين بيختاروا ascending أو descending variant بناءً على `policy->freq_table_sorted`.

---

#### `cpufreq_table_find_index_l` / `_h` / `_c`

```c
static inline int cpufreq_table_find_index_l(struct cpufreq_policy *policy,
                                              unsigned int target_freq,
                                              bool efficiencies);
```

الـ **public API** — بتستخدم `policy->min` و`policy->max` كـ limits تلقائيًا. الـ governor بيستخدمها مباشرة لو هو عارف إنه عايز L أو H أو C بشكل قطعي.

---

#### `cpufreq_is_in_limits`

```c
static inline bool cpufreq_is_in_limits(struct cpufreq_policy *policy,
                                        unsigned int min, unsigned int max,
                                        int idx);
```

بتفحص إن الـ frequency عند الـ index المعطى موجودة فعلًا في نطاق [min, max]. بتستخدمها `cpufreq_frequency_table_target` للتحقق من صحة النتيجة قبل الـ retry.

---

#### `cpufreq_table_set_inefficient`

```c
static inline int cpufreq_table_set_inefficient(struct cpufreq_policy *policy,
                                                 unsigned int frequency);
```

بتعلّم frequency محددة بـ `CPUFREQ_INEFFICIENT_FREQ` flag وبتضبط `policy->efficiencies_available = true`. الـ platform driver بيستخدمها لما يعرف إن فيه frequencies مش كفؤة (مثلًا intermediate OPP levels مش موجودة في الـ silicon بشكل optimal).

- **Key details**: فقط على sorted tables. بترجع `-EINVAL` لو الـ table unsorted أو الـ frequency مش موجودة.

---

#### `cpufreq_frequency_table_cpuinfo`

```c
int cpufreq_frequency_table_cpuinfo(struct cpufreq_policy *policy);
```

بتمشي على الـ freq_table وبتحسب `cpuinfo.min_freq` و`cpuinfo.max_freq` تلقائيًا. بتتكلم من `driver->init()` بعد ما الـ driver يملّي الـ freq_table.

---

#### `cpufreq_frequency_table_verify` / `cpufreq_generic_frequency_table_verify`

```c
int cpufreq_frequency_table_verify(struct cpufreq_policy_data *policy);
int cpufreq_generic_frequency_table_verify(struct cpufreq_policy_data *policy);
```

بتتكلم من `driver->verify()` callback. بتضمن إن الـ `policy->min` و`policy->max` موجودين فعلًا في الـ freq_table. الـ generic version بتكلّم `cpufreq_verify_within_cpu_limits` الأول وبعدين بتضمن إن في على الأقل frequency واحدة valid في الـ [min, max] range.

---

#### `cpufreq_frequency_table_get_index`

```c
int cpufreq_frequency_table_get_index(struct cpufreq_policy *policy,
                                      unsigned int freq);
```

بتعمل linear search في الـ table وبترجع الـ index للـ frequency الـ exact. بترجع negative لو مش موجودة.

---

#### `cpufreq_table_count_valid_entries`

```c
static inline int cpufreq_table_count_valid_entries(const struct cpufreq_policy *policy);
```

بتعدّ كل الـ entries اللي مش `CPUFREQ_ENTRY_INVALID`. مفيدة للـ stats وللـ governor initialization.

---

#### `cpufreq_table_index_unsorted`

```c
int cpufreq_table_index_unsorted(struct cpufreq_policy *policy,
                                  unsigned int target_freq, unsigned int min,
                                  unsigned int max, unsigned int relation);
```

بتعمل linear scan على table مش مرتبة. أبطأ من الـ sorted versions. الـ drivers النادرة اللي مش بتقدر ترتّب الـ table بتلجأ ليها.

---

#### `cpufreq_table_validate_and_sort`

```c
int cpufreq_table_validate_and_sort(struct cpufreq_policy *policy);
```

بتتحقق من صحة الـ freq_table (مفيش duplicates، مفيش entries غلط) وبتفرز الـ entries وبتحدد `policy->freq_table_sorted`. بتتكلم من الـ core أثناء `cpufreq_online`.

---

### Group 8: Suspend / Resume

---

#### `cpufreq_suspend` / `cpufreq_resume`

```c
void cpufreq_suspend(void);
void cpufreq_resume(void);
```

بتعمل suspend/resume لكل الـ policies على مستوى الـ system. `cpufreq_suspend` بتوقّف كل الـ governors ومش بتسمح بأي frequency changes جديدة. `cpufreq_resume` بتعيد تشغيل الـ governors.

- **Who calls them**: الـ `syscore_ops` المسجّلين في الـ cpufreq core.

---

#### `cpufreq_generic_suspend`

```c
int cpufreq_generic_suspend(struct cpufreq_policy *policy);
```

بتضبط الـ frequency على `policy->suspend_freq` قبل الـ system sleep. الـ drivers اللي محتاجة frequency محددة أثناء الـ suspend بتستخدمها في الـ `driver->suspend()` callback.

---

### Group 9: Boost Support

---

#### `cpufreq_boost_enabled`

```c
bool cpufreq_boost_enabled(void);
```

بترجع الـ global boost state. بتقرأ الـ `boost_enabled` variable في الـ cpufreq core.

---

#### `cpufreq_boost_set_sw`

```c
int cpufreq_boost_set_sw(struct cpufreq_policy *policy, int state);
```

بتفعّل/تعطّل الـ boost لـ policy عن طريق إضافة/إزالة الـ boost frequencies من الـ table وإعادة validate الـ policy. بتتكلم لما الـ `set_boost` driver callback مش موجود.

---

### Group 10: Generic Driver Helpers

---

#### `cpufreq_generic_get`

```c
unsigned int cpufreq_generic_get(unsigned int cpu);
```

تنفيذ generic لـ `driver->get()`. بيستخدم الـ `policy->clk` ويكلّم `clk_get_rate()` مباشرة. الـ drivers اللي عندهم `clk` pointer بيستخدموا الـ function دي بدل ما يعملوا implementation خاصة بيهم.

---

#### `cpufreq_generic_init`

```c
void cpufreq_generic_init(struct cpufreq_policy *policy,
                          struct cpufreq_frequency_table *table,
                          unsigned int transition_latency);
```

بتعمل initialize الـ policy بأبسط طريقة:
1. بتضبط `policy->freq_table = table`.
2. بتكلّم `cpufreq_frequency_table_cpuinfo()` لحساب min/max.
3. بتضبط `cpuinfo.transition_latency`.
4. بتضبط `policy->cpus` على كل الـ CPUs اللي بيشاركوا الـ policy (من الـ topology).

الـ driver بيكلّمها من `driver->init()` لو مش محتاج أي حاجة custom.

---

### Group 11: Verify & Scale Helpers

---

#### `cpufreq_verify_within_limits`

```c
static inline void cpufreq_verify_within_limits(struct cpufreq_policy_data *policy,
                                                 unsigned int min,
                                                 unsigned int max);
```

بتعمل `clamp` للـ `policy->max` في [min, max]، وبعدين `clamp` للـ `policy->min` في [min, policy->max]. ترتيب الـ clamping مهم — max الأول، بعدين min. لازم الـ driver يكلّمها من `driver->verify()`.

---

#### `cpufreq_verify_within_cpu_limits`

```c
static inline void cpufreq_verify_within_cpu_limits(struct cpufreq_policy_data *policy);
```

wrapper بيكلّم `cpufreq_verify_within_limits` بـ `cpuinfo.min_freq` و`cpuinfo.max_freq`. الـ drivers اللي مش عندهم hardware-specific limits بتستخدمها مباشرة.

---

#### `cpufreq_scale`

```c
static inline unsigned long cpufreq_scale(unsigned long old, u_int div, u_int mult);
```

بتحسب `old * mult / div` بأمان على كل من الـ 32-bit و64-bit architectures. على 32-bit بتستخدم `u64` intermediate وبتعمل `do_div`. على 64-bit بتعمل cast للـ mult لـ u64 قبل الضرب. بيستخدمها الـ governor لحساب الـ scaling factors.

---

#### `arch_set_freq_scale`

```c
static __always_inline void arch_set_freq_scale(const struct cpumask *cpus,
                                                unsigned long cur_freq,
                                                unsigned long max_freq);
```

weak symbol. الـ architecture بتعمل override ليه لو بتدعم **frequency invariance** في الـ scheduler. بتضبط الـ per-CPU `arch_freq_scale` variable اللي الـ scheduler بيستخدمه لتعديل الـ CPU capacity بناءً على الـ current frequency.

- **`cpus`**: الـ CPUs اللي بيتأثروا بالتغيير.
- **`cur_freq`**: الـ current frequency.
- **`max_freq`**: الـ maximum possible frequency.
- **Default**: empty implementation (no-op) لو الـ arch مش supportive.

---

#### `arch_freq_get_on_cpu`

```c
extern int arch_freq_get_on_cpu(int cpu);
```

بتقرأ الـ current frequency من الـ arch-specific performance counter أو MSR register. مش كل architectures بتدعمها — بترجع negative لو مش مدعومة.

---

### Group 12: OPP / DT / Energy Model Helpers

---

#### `parse_perf_domain`

```c
static inline int parse_perf_domain(int cpu, const char *list_name,
                                    const char *cell_name,
                                    struct of_phandle_args *args);
```

بتجيب الـ `device_node` للـ CPU من الـ DT وبتعمل `of_parse_phandle_with_args()` لتحديد الـ performance domain. بتستخدمها `of_perf_domain_get_sharing_cpumask` داخليًا.

---

#### `of_perf_domain_get_sharing_cpumask`

```c
static inline int of_perf_domain_get_sharing_cpumask(int pcpu,
                                                      const char *list_name,
                                                      const char *cell_name,
                                                      struct cpumask *cpumask,
                                                      struct of_phandle_args *pargs);
```

بتمشي على كل الـ possible CPUs وبتشوف مين بيشارك نفس الـ OPP/performance domain زي الـ `pcpu` بمقارنة الـ `of_phandle_args`. بترجع الـ cpumask مليانة الـ CPUs المتشاركة. الـ drivers بيستخدموها في `driver->init()` لبناء `policy->cpus`.

---

#### `cpufreq_register_em_with_opp`

```c
static inline void cpufreq_register_em_with_opp(struct cpufreq_policy *policy);
```

بتكلّم `dev_pm_opp_of_register_em()` لتسجيل الـ **Energy Model** للـ CPU cluster من الـ OPP table في الـ DT. لازم تتكلم من الـ `driver->register_em()` callback بعد ما الـ policy تتهيّأ بالكامل وقبل ما الـ governor يبدأ.

---

#### `cpufreq_ready_for_eas`

```c
bool cpufreq_ready_for_eas(const struct cpumask *cpu_mask);
```

بتفحص لو الـ cpufreq policy لـ الـ CPU mask معينة جاهزة لـ **Energy Aware Scheduling (EAS)**. بتضمن إن الـ Energy Model متسجّل وإن الـ governor المناسب شغّال (عادةً `schedutil`).

---

### Group 13: sysfs & Utility Functions

---

#### `cpufreq_show_cpus`

```c
ssize_t cpufreq_show_cpus(const struct cpumask *mask, char *buf);
```

بتكتب الـ cpumask في الـ sysfs buffer بالـ format المعتاد (أرقام CPUs مفصولة بـ spaces). بتستخدمها الـ `related_cpus` و`affected_cpus` sysfs attributes.

---

#### `get_cpu_idle_time`

```c
u64 get_cpu_idle_time(unsigned int cpu, u64 *wall, int io_busy);
```

بترجع الـ total idle time للـ CPU بالـ nanoseconds. الـ sampling governors (مثلًا `ondemand`، `conservative`) بيستخدموها لحساب نسبة الـ load. الـ `wall` pointer بيتملى بـ total wall clock time. الـ `io_busy` parameter بيحدد هل الـ iowait بيتحسب كـ idle أو busy.

---

#### `have_governor_per_policy`

```c
bool have_governor_per_policy(void);
```

بترجع `true` لو الـ driver عنده `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` flag. الـ sysfs code بيستخدمها لتحديد إيه الـ parent kobject للـ governor directory.

---

#### `get_governor_parent_kobj`

```c
struct kobject *get_governor_parent_kobj(struct cpufreq_policy *policy);
```

بترجع الـ kobject الأب لـ sysfs directory الخاصة بالـ governor. لو الـ driver بيدعم per-policy governors، بترجع `&policy->kobj`. وإلا بترجع الـ global cpufreq kobject.

---

#### `cpufreq_enable_fast_switch` / `cpufreq_disable_fast_switch`

```c
void cpufreq_enable_fast_switch(struct cpufreq_policy *policy);
void cpufreq_disable_fast_switch(struct cpufreq_policy *policy);
```

الـ `governor->start()` بيكلّم `cpufreq_enable_fast_switch` لو بيريد يستخدم الـ fast path (schedutil بيعمل كده). الـ `governor->stop()` بيكلّم `cpufreq_disable_fast_switch`. الـ function بتفحص `policy->fast_switch_possible` الأول.

---

#### `cpufreq_update_policy` / `cpufreq_update_limits`

```c
void cpufreq_update_policy(unsigned int cpu);
void cpufreq_update_limits(unsigned int cpu);
```

**`cpufreq_update_policy`**: بتعيد تطبيق الـ policy كاملة — بتكلّم `driver->verify()` وبتبعت الـ notifiers. بتتكلم من IRQ context عبر `policy->update` work item.

**`cpufreq_update_limits`**: بتبلّغ الـ driver إن الـ firmware غيّر الـ limits (مثلًا ACPI notification) عن طريق استدعاء `driver->update_limits()`.

---

#### `refresh_frequency_limits`

```c
void refresh_frequency_limits(struct cpufreq_policy *policy);
```

بتعيد حساب الـ effective min/max للـ policy من الـ `freq_qos` constraints. بتتكلم لما الـ QoS requests تتغير.

---

#### `disable_cpufreq`

```c
void disable_cpufreq(void);
```

بتعطّل الـ cpufreq subsystem كله — بتضبط flag يمنع أي frequency changes مستقبلية. بتتكلم في حالات panic أو boot failures.

---

#### `cpufreq_supports_freq_invariance`

```c
bool cpufreq_supports_freq_invariance(void);
```

بترجع `true` لو الـ cpufreq driver يدعم frequency invariance — يعني بيكلّم `arch_set_freq_scale` بعد كل transition. الـ scheduler بيستخدمها لمعرفة لو الـ task capacity calculations موثوقة.

---

#### `has_target_index`

```c
bool has_target_index(void);
```

بتفحص لو الـ driver الحالي عنده `target_index` callback. بيستخدمها الـ intermediate frequency code لمعرفة هل يتم دعم two-step transitions.

---

### Macros Summary

#### Iterator Macros

```c
/* بتمشي على كل الـ entries */
cpufreq_for_each_entry(pos, table)

/* بتمشي مع index */
cpufreq_for_each_entry_idx(pos, table, idx)

/* بتتخطى CPUFREQ_ENTRY_INVALID */
cpufreq_for_each_valid_entry(pos, table)

/* بتتخطى INVALID مع index */
cpufreq_for_each_valid_entry_idx(pos, table, idx)

/* بتتخطى INVALID والـ INEFFICIENT_FREQ لو efficiencies=true */
cpufreq_for_each_efficient_entry_idx(pos, table, idx, efficiencies)
```

#### sysfs Attribute Macros

```c
cpufreq_freq_attr_ro(_name)      /* read-only: 0444 */
cpufreq_freq_attr_rw(_name)      /* read-write: 0644 */
cpufreq_freq_attr_wo(_name)      /* write-only: 0200 */
cpufreq_freq_attr_ro_perm(_name, _perm) /* custom perms */
define_one_global_ro(_name)      /* kobj_attribute read-only */
define_one_global_rw(_name)      /* kobj_attribute read-write */
```

#### Governor Registration Macros

```c
/* بيعمل module_init يكلّم cpufreq_register_governor */
cpufreq_governor_init(__governor)

/* بيعمل module_exit يكلّم cpufreq_unregister_governor */
cpufreq_governor_exit(__governor)
```

#### Locking Guards

```c
/* RAII-style write lock على policy->rwsem */
DEFINE_GUARD(cpufreq_policy_write, struct cpufreq_policy *, ...)

/* RAII-style read lock على policy->rwsem */
DEFINE_GUARD(cpufreq_policy_read, struct cpufreq_policy *, ...)

/* scope-based cleanup بيكلّم cpufreq_cpu_put تلقائيًا */
DEFINE_FREE(put_cpufreq_policy, struct cpufreq_policy *, ...)
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

**الـ** `cpufreq` بيكتب في `/sys/kernel/debug/` لما يكون `CONFIG_DEBUG_FS=y`.

```bash
# اعرض كل مدخلات الـ cpufreq في debugfs
ls /sys/kernel/debug/cpufreq/

# اقرأ الـ stats التفصيلية لكل policy
cat /sys/kernel/debug/cpufreq/policy0/stats/time_in_state
cat /sys/kernel/debug/cpufreq/policy0/stats/total_trans
cat /sys/kernel/debug/cpufreq/policy0/stats/trans_table
```

**تفسير الـ** `time_in_state`:
```
# كل سطر = <freq_kHz> <time_in_jiffies>
1200000 14523
1400000 8921
1800000 3012
# الـ CPU قضى معظم الوقت على 1.2GHz → governor سلبي جداً أو thermal throttling
```

**الـ** `trans_table` بيدي matrix الانتقالات بين الترددات — مفيد لتشخيص oscillation بين ترددين.

---

#### 2. مدخلات الـ sysfs

**الـ** `cpufreq_policy` بتكشف نفسها عبر kobject في `/sys/devices/system/cpu/cpuN/cpufreq/`:

| المسار | المحتوى | ملاحظة |
|--------|---------|---------|
| `scaling_cur_freq` | التردد الحالي (kHz) | من `policy->cur` |
| `scaling_min_freq` | الحد الأدنى (kHz) | `policy->min` |
| `scaling_max_freq` | الحد الأقصى (kHz) | `policy->max` |
| `cpuinfo_min_freq` | أدنى تردد هاردوير | `cpuinfo.min_freq` |
| `cpuinfo_max_freq` | أعلى تردد هاردوير | `cpuinfo.max_freq` |
| `cpuinfo_transition_latency` | زمن الانتقال (ns) | `cpuinfo.transition_latency` |
| `scaling_governor` | الـ governor الحالي | `policy->governor->name` |
| `scaling_available_governors` | كل الـ governors المتاحة | — |
| `scaling_available_frequencies` | جدول الترددات | `freq_table` |
| `related_cpus` | كل الـ CPUs المرتبطة | `policy->related_cpus` |
| `affected_cpus` | الـ CPUs الأونلاين فقط | `policy->cpus` |
| `boost` | حالة الـ boost | `policy->boost_enabled` |

```bash
# اقرأ كل مدخلات policy واحدة دفعة واحدة
for f in /sys/devices/system/cpu/cpu0/cpufreq/*; do
    echo "=== $f ==="; cat "$f" 2>/dev/null
done

# تحقق إن الـ cur_freq بيتطابق مع الـ hardware
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq  # من الـ driver مباشرة
```

**الفرق المهم**: `scaling_cur_freq` من الـ governor، `cpuinfo_cur_freq` من `driver->get()` — لو مختلفين فيه bug في الـ transition.

**الـ** `freq_qos` في sysfs:
```bash
# QoS limits مفروضة من thermal أو userspace
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
# لو القيم مش اللي أنت حاطها، فيه notifier بيغيرها (thermal أو policy QoS)
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

**الـ** `cpufreq` عنده tracepoints جاهزة في `include/trace/events/power.h`.

```bash
# فعّل الـ cpufreq tracepoints
cd /sys/kernel/debug/tracing

# اعرض الـ events المتاحة
grep -r cpufreq available_events

# فعّل أهم الـ events
echo 1 > events/power/cpu_frequency/enable
echo 1 > events/power/cpu_frequency_limits/enable
echo 1 > events/power/cpu_idle/enable

# شغّل الـ tracing
echo 1 > tracing_on
sleep 5
echo 0 > tracing_on

# اقرأ النتيجة
cat trace | grep cpu_freq
```

**مثال على output**:
```
# TASK-PID CPU# TIMESTAMP FUNCTION
kworker/0:1-42  [000] 1234.567890: cpu_frequency: state=1800000 cpu_id=0
kworker/0:1-42  [000] 1234.570012: cpu_frequency: state=1400000 cpu_id=0
# انتقالات متكررة = governor oscillation أو thermal thrashing
```

**الـ function tracer للـ driver**:
```bash
# تتبع دوال الـ cpufreq core
echo function > current_tracer
echo 'cpufreq_*' > set_ftrace_filter
echo '__cpufreq_driver_target' >> set_ftrace_filter
echo 'cpufreq_freq_transition_*' >> set_ftrace_filter
echo 1 > tracing_on
```

**الـ function_graph لفهم call chain كامل**:
```bash
echo function_graph > current_tracer
echo 'cpufreq_driver_target' > set_graph_function
cat trace
```

---

#### 4. الـ printk والـ Dynamic Debug

**تفعيل الـ dynamic debug** لسبسيستم الـ cpufreq بدون recompile:

```bash
# فعّل كل debug messages في ملفات الـ cpufreq
echo 'file drivers/cpufreq/* +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/power/cpufreq.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل module محدد (مثلاً cpufreq-dt)
echo 'module cpufreq_dt +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p=print, f=function name, l=line number, m=module, t=thread ID

# اتابع الـ kernel log
dmesg -w | grep -i cpufreq
journalctl -kf | grep -i cpufreq
```

**الـ printk level**:
```bash
# رفع مستوى الـ kernel log لأعلى درجة
echo 8 > /proc/sys/kernel/printk
# أو
dmesg -n 8
```

---

#### 5. أهم الـ Kernel Config Options للـ Debugging

| الـ Config | الغرض |
|-----------|--------|
| `CONFIG_CPU_FREQ_DEBUG` | يضيف `pr_debug()` إضافية في الـ core |
| `CONFIG_CPU_FREQ_STAT` | يفعّل `/sys/.../stats/` — `time_in_state` و `trans_table` |
| `CONFIG_DEBUG_FS` | مطلوب لـ debugfs entries |
| `CONFIG_TRACING` | مطلوب للـ ftrace |
| `CONFIG_TRACE_IRQFLAGS` | يساعد في اكتشاف مشاكل الـ spinlock في transition |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في `policy->rwsem` |
| `CONFIG_PROVE_LOCKING` | تحقق صارم من ترتيب الـ locks |
| `CONFIG_KASAN` | اكتشاف memory corruption في بيانات الـ driver |
| `CONFIG_UBSAN` | اكتشاف undefined behavior في حسابات الـ freq |
| `CONFIG_DEBUG_SPINLOCK` | تحقق من `transition_lock` |
| `CONFIG_ENERGY_MODEL` | مطلوب لـ EAS debugging مع الـ cpufreq |
| `CONFIG_CPU_THERMAL` | فعّل thermal cooling عبر `CPUFREQ_IS_COOLING_DEV` |
| `CONFIG_CPU_FREQ_GOV_SCHEDUTIL` | governor الـ schedutil مع `sugov_is_governor()` |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CPU_FREQ|CPUFREQ' | grep -v '^#'
```

---

#### 6. أدوات الـ Subsystem المتخصصة

**الأداة الرئيسية: `cpupower`**

```bash
# اعرض معلومات الـ frequency كاملة
cpupower frequency-info

# اعرض الـ governor الحالي لكل الـ CPUs
cpupower -c all frequency-info -g

# غيّر الـ governor للـ testing
cpupower frequency-set -g performance
cpupower frequency-set -g schedutil

# اضبط تردد ثابت للـ debugging
cpupower frequency-set -f 1800MHz
cpupower frequency-set --min 1200MHz --max 2400MHz

# اعرض إحصائيات الـ idle
cpupower idle-info
cpupower monitor -m Mperf   # قياس الـ freq الفعلية عبر MSR
```

**الأداة: `turbostat`** (x86 فقط)

```bash
# يقيس الـ freq الفعلية من الـ hardware counters مش من الـ sysfs
turbostat --show CPU,Avg_MHz,Busy%,Bzy_MHz,TSC_MHz,IRQ,POLL
# Avg_MHz = الـ freq المتوسطة الفعلية
# Bzy_MHz = الـ freq وقت الشغل بس
# لو Avg_MHz << Bzy_MHz → الـ CPU كثير idle
# لو Bzy_MHz < scaling_max_freq → thermal throttling
```

**الأداة: `perf`**

```bash
# قياس الـ frequency scaling events
perf stat -e power:cpu_frequency,power:cpu_frequency_limits sleep 10

# تتبع transitions
perf record -e power:cpu_frequency -a sleep 5
perf script | head -50
```

**الـ `devlink`**: مش مستخدم في cpufreq مباشرة، بس لو الـ driver بيستخدم firmware interface:

```bash
# لو الـ platform بيستخدم SCMI أو firmware للـ freq scaling
cat /sys/bus/platform/devices/*/cpufreq/*/scaling_cur_freq
# تحقق من الـ firmware interface
ls /sys/firmware/
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | السبب | الحل |
|--------------------|-------|------|
| `cpufreq: __cpufreq_driver_target: -22` | الـ driver رفض الـ target freq | تحقق من `verify()` callback و `policy->min/max` |
| `cpufreq: Failed to start governor` | الـ governor's `init()` أو `start()` فشلت | `dmesg` للتفاصيل، تحقق من `CONFIG_CPU_FREQ_GOV_*` |
| `cpufreq: CPU X not found in policy` | الـ CPU مش في الـ `cpumask` | مشكلة hotplug، تحقق من `policy->related_cpus` |
| `cpufreq: Failed to register driver` | `cpufreq_register_driver()` فشلت | driver مكرر؟ تحقق من `cpufreq_get_current_driver()` |
| `BUG_ON` after `CPUFREQ_NEED_INITIAL_FREQ_CHECK` | الـ CPU بيشتغل بتردد مش في الـ freq table | الـ hardware أو الـ bootloader حطوا تردد غريب |
| `cpufreq: transition_task != current` | thread تاني حاول يعمل transition | race condition في الـ driver، تحقق من `transition_lock` |
| `cpufreq: governor already registered` | الـ governor اتسجّل مرتين | module conflict |
| `cpufreq: Failed to change frequency` | `target_index()` أو `setpolicy()` رجعت error | تحقق من الـ hardware state والـ voltage regulators |
| `cpufreq: out of table freq X` | الـ get() بيرجع تردد مش في الـ freq_table | الـ hardware تغير تردده من برّه | راجع الـ BIOS/firmware |
| `cpufreq: __cpufreq_driver_target called with cpu X not in policy` | خطأ في الـ caller | الـ governor بيستهدف CPU غلط |

---

#### 8. أماكن استراتيجية لـ `dump_stack()` و`WARN_ON()`

```c
/* في driver->target_index() لو الـ freq مش بتتغير */
int my_driver_target_index(struct cpufreq_policy *policy, unsigned int index)
{
    unsigned int target_freq = policy->freq_table[index].frequency;

    WARN_ON(target_freq < policy->min || target_freq > policy->max);

    if (set_hw_freq(target_freq) < 0) {
        pr_err("Failed to set freq %u kHz\n", target_freq);
        dump_stack();  /* كشف من استدعى الـ target */
        return -EIO;
    }
    return 0;
}

/* تحقق من transition state */
WARN_ON(policy->transition_ongoing && current != policy->transition_task);

/* في fast_switch — تحقق إننا مش في context ممنوع */
WARN_ON(!policy->fast_switch_enabled);
WARN_ON(policy->transition_ongoing);

/* تحقق من rwsem */
WARN_ON(!rwsem_is_locked(&policy->rwsem));  /* لو المفروض نكون locked */

/* في governor limits() — تحقق من QoS constraints */
WARN_ON(policy->min > policy->max);
WARN_ON(policy->cur < policy->min || policy->cur > policy->max);
```

**أماكن مفيدة لـ** `dump_stack()`:
- في `cpufreq_freq_transition_begin()` لفهم كل من بيطلق الـ transition
- في `cpufreq_register_driver()` لفهم ترتيب التسجيل
- في `notifier_call` لفهم من استدعى الـ notifier chain

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

```bash
# 1. قارن الـ sysfs مع الـ hardware counter (x86)
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# vs MSR 0x198 (IA32_PERF_STATUS) upper 16 bits = current ratio
rdmsr -p 0 0x198   # يتطلب rdmsr tool

# 2. تحقق من BIOS/ACPI frequency override
cat /proc/acpi/processor/CPU0/throttling
cat /sys/devices/system/cpu/cpu0/acpi_cppc/cur_perf  # لو ACPI CPPc

# 3. تحقق من thermal state
cat /sys/class/thermal/thermal_zone*/temp
cat /sys/class/thermal/cooling_device*/cur_state
# لو cur_state > 0 → الـ thermal بيحد الـ freq

# 4. ARM: تحقق من OPP table
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
# يجب أن تطابق الـ OPP table في الـ Device Tree

# 5. قارن الـ driver get() مع الـ sysfs
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq  # driver->get()
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq  # governor value
# لو مختلفين بعد transition → driver->get() مش بيقرأ صح
```

---

#### 2. تقنيات الـ Register Dump

**على x86 (Intel/AMD):**

```bash
# اقرأ الـ MSRs المتعلقة بالـ freq scaling
# IA32_PERF_STATUS (0x198): Current P-state
rdmsr -p 0 -u 0x198
# Bits [15:8] = current HWP ratio → multiply by 100MHz للحصول على الـ freq

# IA32_PERF_CTL (0x199): Target P-state
rdmsr -p 0 -u 0x199

# IA32_HWP_REQUEST (0x774): HWP min/max/desired
rdmsr -p 0 -u 0x774

# IA32_HWP_STATUS (0x777): HWP feedback
rdmsr -p 0 -u 0x777

# AMD: MSRC0010064 P-state
rdmsr -p 0 0xC0010064

# تقرأ كل MSRs لكل الـ CPUs
for cpu in $(seq 0 $(($(nproc) - 1))); do
    echo "CPU $cpu: $(rdmsr -p $cpu -u 0x198)"
done
```

**على ARM/ARM64:**

```bash
# استخدم devmem2 لقراءة clock registers مباشرة
# مثال على Rockchip RK3399:
devmem2 0xFF760000 w  # CRU base address
# أو عبر /dev/mem
dd if=/dev/mem bs=4 count=1 skip=$((0xFF760000/4)) 2>/dev/null | xxd

# قراءة عبر io tool
io -4 -r 0xFF760000  # اقرأ 4 bytes من الـ register

# لو الـ SCMI firmware:
cat /sys/bus/platform/devices/firmware:scmi/protocol@13/*/cur_perf_level
```

**الـ `/proc/cpuinfo` للـ verification**:

```bash
# تحقق من الـ MHz الفعلي كما يراه الـ kernel
grep 'cpu MHz' /proc/cpuinfo
# لو مختلف عن scaling_cur_freq → مشكلة في الـ tsc أو driver->get()
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**نقاط القياس على الـ Hardware:**

```
+------------------+         +------------------+
|   SoC / CPU      |         |  PMIC / VRM      |
|                  |         |                  |
| I2C/SPI SCL -----+----→----+  SCL             |
| I2C/SPI SDA -----+----↔----+  SDA             |
|                  |         |                  |
| PWM / GPIO  -----+----→----+  EN / CTRL       |
|                  |         |                  |
| VOUT sense  -----+----←----+  Voltage sense   |
+------------------+         +------------------+
          ↑
     قيس هنا الـ
    voltage ramps
```

**إعداد الـ Logic Analyzer:**

```
- Sample rate: >= 10x أعلى clock للـ I2C/SPI (مثلاً 4MHz لـ 400kHz I2C)
- Trigger: على أول edge في الـ I2C transaction للـ PMIC
- Channels:
  CH1: I2C SCL
  CH2: I2C SDA
  CH3: CPU VDD (عبر current probe)
  CH4: GPIO DVFS_REQ (لو موجود)

- Decode I2C لتشوف الـ voltage registers بتتكتب بكام
- قيس الـ delay بين I2C write للـ voltage وبدء الـ freq change
```

**الـ Oscilloscope لقياس الـ voltage ramp:**

```bash
# الـ kernel بيدي latency في:
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_transition_latency
# هذا هو الـ transition_latency بالـ nanoseconds

# قارنه بالـ oscilloscope reading:
# لو الـ hardware أبطأ من الـ latency القيمة → kernel بيفترض إنك وصلت بس لأ
# → هيعدي على تردد قديم بـ voltage جديدة = خطر stability
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | نمط الـ Log | التشخيص |
|---------|-----------|---------|
| Thermal Throttling | `thermal thermal_zone0: critical temperature reached` + freq تنزل | `cat /sys/class/thermal/thermal_zone*/temp` |
| PMIC voltage failure | `cpufreq: Failed to change frequency` متكرر على ترددات عالية | قيس الـ VDD بالـ oscilloscope تحت load |
| Clock source جاف | `cpufreq: CPU freq reading failed` | تحقق من الـ CLK tree عبر `cat /sys/kernel/debug/clk/clk_summary` |
| Frequency مش بتتغير | `scaling_cur_freq` ثابت رغم governor | `cpuinfo_cur_freq` مختلف؟ → driver->get() مكسور |
| Out-of-table boot freq | `BUG_ON` عند `CPUFREQ_NEED_INITIAL_FREQ_CHECK` | الـ bootloader أو BIOS حط تردد برّه الـ table |
| OPP mismatch | `cpu cpu0: OPP not found` | الـ DT OPP table مش متطابق مع الـ hardware |
| Voltage regulator timeout | `regulator: timeout waiting for voltage` | الـ `transition_latency` أقل من الـ PMIC settling time |

---

#### 5. الـ Device Tree Debugging

**التحقق إن الـ DT يطابق الـ Hardware:**

```bash
# اعرض الـ OPP table المحملة
cat /sys/bus/cpu/devices/cpu0/opp_table  # لو `CONFIG_PM_OPP`
# أو
cat /sys/kernel/debug/opp/cpu0/opp_summary

# تحقق من الـ DT nodes المحملة
cat /proc/device-tree/cpus/cpu@0/operating-points-v2 | xxd
# أو الأوضح:
dtc -I fs /proc/device-tree/ 2>/dev/null | grep -A 20 'operating-points'

# قارن مع الـ driver الـ freq table
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
```

**مثال على DT صح للـ OPP:**

```dts
/* Device Tree: OPP table للـ cpufreq */
cpu_opp_table: opp-table {
    compatible = "operating-points-v2";
    opp-shared;  /* مهم: كل CPUs في نفس الـ policy بيشاركوا */

    opp-1200000000 {
        opp-hz = /bits/ 64 <1200000000>;  /* 1.2GHz */
        opp-microvolt = <1000000>;         /* 1.0V */
        clock-latency-ns = <300000>;       /* 300µs transition */
    };
    opp-1800000000 {
        opp-hz = /bits/ 64 <1800000000>;
        opp-microvolt = <1100000>;
        clock-latency-ns = <300000>;
        turbo-mode;  /* boost freq */
    };
};
```

**تشخيص مشاكل الـ DT:**

```bash
# لو الـ driver مش بيلاقي الـ OPP
dmesg | grep -i 'opp\|operating-point'

# تحقق من الـ phandle references
grep -r 'operating-points-v2' /proc/device-tree/ 2>/dev/null

# تحقق من الـ cpu DT node
cat /proc/device-tree/cpus/cpu@0/compatible
cat /proc/device-tree/cpus/cpu@0/clocks  # reference للـ clock

# لو الـ DT clock مش موجود → cpufreq مش هيشتغل
dmesg | grep 'Failed to get cpu clock\|clk_get'

# تحقق من الـ parse_perf_domain (مستخدمة في cpufreq.h للـ EAS)
dmesg | grep 'energy-performance-available-preferences\|performance-domain'
```

---

### Practical Commands

#### الـ Software Debugging كاملاً

```bash
#!/bin/bash
# cpufreq_debug.sh — سكريبت شامل لتشخيص الـ cpufreq

echo "=== CPUFreq Subsystem Debug Report ==="
date

echo -e "\n--- Kernel Version ---"
uname -r

echo -e "\n--- CPUFreq Driver ---"
cat /sys/bus/cpu/drivers/cpufreq/*/uevent 2>/dev/null || \
    cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver

echo -e "\n--- All Policies ---"
for policy in /sys/devices/system/cpu/cpufreq/policy*/; do
    echo "Policy: $policy"
    for attr in scaling_governor scaling_cur_freq scaling_min_freq \
                scaling_max_freq cpuinfo_min_freq cpuinfo_max_freq \
                cpuinfo_transition_latency related_cpus boost; do
        val=$(cat "$policy/$attr" 2>/dev/null)
        [ -n "$val" ] && printf "  %-35s = %s\n" "$attr" "$val"
    done
done

echo -e "\n--- Frequency Stats ---"
for policy in /sys/devices/system/cpu/cpufreq/policy*/; do
    name=$(basename $policy)
    echo "=== $name ==="
    cat "$policy/stats/time_in_state" 2>/dev/null | \
        awk '{printf "  %7d MHz: %s jiffies\n", $1/1000, $2}'
    echo "  Total transitions: $(cat $policy/stats/total_trans 2>/dev/null)"
done

echo -e "\n--- Thermal Zones ---"
for zone in /sys/class/thermal/thermal_zone*/; do
    temp=$(cat "$zone/temp" 2>/dev/null)
    type=$(cat "$zone/type" 2>/dev/null)
    printf "  %-30s = %d.%d°C\n" "$type" $((temp/1000)) $((temp%1000/100))
done

echo -e "\n--- Thermal Cooling Devices ---"
for dev in /sys/class/thermal/cooling_device*/; do
    type=$(cat "$dev/type" 2>/dev/null)
    cur=$(cat "$dev/cur_state" 2>/dev/null)
    max=$(cat "$dev/max_state" 2>/dev/null)
    [ "$cur" != "0" ] && echo "  ACTIVE: $type = $cur/$max"
done

echo -e "\n--- /proc/cpuinfo MHz ---"
grep 'cpu MHz' /proc/cpuinfo | sort -u

echo -e "\n--- Recent CPUFreq Kernel Messages ---"
dmesg | grep -i cpufreq | tail -20
```

```bash
# تشغيل الـ ftrace لمدة 10 ثواني
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo 1 > events/power/cpu_frequency/enable
echo 1 > events/power/cpu_frequency_limits/enable
echo 1 > tracing_on
sleep 10
echo 0 > tracing_on

# تلخيص الـ transitions
awk '/cpu_frequency/ {
    match($0, /state=([0-9]+) cpu_id=([0-9]+)/, arr)
    count[arr[2]][arr[1]]++
}
END {
    for (cpu in count)
        for (freq in count[cpu])
            printf "CPU%s: %7d kHz → %d times\n", cpu, freq, count[cpu][freq]
}' trace | sort -k1,1 -k2,2n
```

```bash
# مقارنة الـ hardware freq مع الـ kernel (x86 فقط)
echo "=== Hardware vs Kernel Frequency ==="
for cpu in $(seq 0 $(($(nproc)-1))); do
    kernel_freq=$(cat /sys/devices/system/cpu/cpu$cpu/cpufreq/scaling_cur_freq 2>/dev/null)
    hw_freq=$(rdmsr -p $cpu -u 0x198 2>/dev/null | awk '{printf "%d", (($1 >> 8) & 0xFF) * 100000}')
    printf "CPU%d: kernel=%7d kHz  hardware≈%7d kHz  %s\n" \
        $cpu $kernel_freq $hw_freq \
        "$([ "$kernel_freq" != "$hw_freq" ] && echo '⚠ MISMATCH' || echo 'OK')"
done
```

```bash
# فحص OPP table و DT
echo "=== OPP Table ==="
cat /sys/kernel/debug/opp/cpu*/opp_summary 2>/dev/null || \
    cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies

echo -e "\n=== Clock Tree (cpufreq relevant) ==="
cat /sys/kernel/debug/clk/clk_summary 2>/dev/null | grep -i 'cpu\|pll\|dvfs' | head -30

echo -e "\n=== QoS Frequency Limits ==="
for cpu in $(seq 0 $(($(nproc)-1))); do
    min=$(cat /sys/devices/system/cpu/cpu$cpu/cpufreq/scaling_min_freq 2>/dev/null)
    max=$(cat /sys/devices/system/cpu/cpu$cpu/cpufreq/scaling_max_freq 2>/dev/null)
    hw_min=$(cat /sys/devices/system/cpu/cpu$cpu/cpufreq/cpuinfo_min_freq 2>/dev/null)
    hw_max=$(cat /sys/devices/system/cpu/cpu$cpu/cpufreq/cpuinfo_max_freq 2>/dev/null)
    [ "$min" != "$hw_min" ] || [ "$max" != "$hw_max" ] && \
        echo "CPU$cpu: QoS is LIMITING freq (min=$min/$hw_min max=$max/$hw_max)"
done

echo -e "\n=== cpupower Summary ==="
cpupower frequency-info 2>/dev/null | head -30
```

```bash
# تتبع مشكلة transition_ongoing deadlock
# لو الـ system هانج أثناء freq change:
echo 1 > /proc/sysrq-trigger  # SysRq-t يطبع كل الـ threads
# أو
echo t > /proc/sysrq-trigger
dmesg | grep -A 5 'transition_task\|cpufreq'
# ابحث عن thread عالق في:
# wait_event(policy->transition_wait, !policy->transition_ongoing)
```

```bash
# استخدام turbostat لمراقبة real-time (x86)
turbostat \
    --show CPU,Avg_MHz,Busy%,Bzy_MHz,TSC_MHz,PkgWatt,CorWatt,IRQ \
    --interval 1 \
    -- sleep 30
# Avg_MHz << Bzy_MHz → كثير C-states
# Bzy_MHz ثابت تحت الـ max → thermal أو power capping
# PkgWatt عالي → check TDP limits
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Rockchip RK3562 — Industrial Gateway يرفض يشتغل بأقل من 1.2GHz

#### العنوان
**الـ cpufreq policy مش بتنزل تحت الـ min_freq بسبب `cpufreq_verify_within_limits` غلط**

#### السياق
شركة بتعمل industrial IoT gateway على RK3562 (quad-core Cortex-A53). المنتج مفروض يشتغل في modes مختلفة: **idle mode** بـ 408MHz و**active mode** بـ 1.5GHz. الـ BSP engineer كتب driver خاص بيعمل override للـ `verify()` callback.

#### المشكلة
الـ gateway دايمًا شايل 1.2GHz حتى وهو idle، والـ power consumption عالي جدًا — بيشكو العميل إن البطارية الاحتياطية بتخلص في 4 ساعات بدل 12.

```bash
# Engineer بيشوف الفريكوانسي
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
1200000

$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
408000

$ cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_min_freq
408000
```

مع إن الـ governor (schedutil) بيطلب 408MHz، الـ CPU مش بينزل.

#### التحليل

الـ driver بتاعه كتب الـ `verify()` callback بالشكل ده:

```c
static int rk3562_cpufreq_verify(struct cpufreq_policy_data *policy)
{
    /* BUG: Engineer hardcoded min to 1.2GHz thinking it's "safe" */
    cpufreq_verify_within_limits(policy, 1200000, 1800000);
    return 0;
}
```

لما بنرجع للـ header:

```c
/* من cpufreq.h السطر 493 */
static inline void cpufreq_verify_within_limits(struct cpufreq_policy_data *policy,
                                                unsigned int min,
                                                unsigned int max)
{
    /* clamp بيخلي policy->max بين min و max */
    policy->max = clamp(policy->max, min, max);
    /* clamp بيخلي policy->min بين min و policy->max */
    policy->min = clamp(policy->min, min, policy->max);
}
```

**المشكلة واضحة:** `cpufreq_verify_within_limits` بتـ clamp الـ `policy->min` بحيث متنزلش تحت الـ `min` اللي بنمرره — وهو هنا 1200000 kHz.

يعني حتى لو schedutil طلب 408MHz، الـ core بيعمل verify وبيلاقي إن policy->min = 408000 < 1200000 فبيرفعه لـ 1200000.

الصح هو إنك تخلي الـ min الفعلي يجي من `policy->cpuinfo.min_freq`:

```c
/* من cpufreq.h السطر 501 */
static inline void
cpufreq_verify_within_cpu_limits(struct cpufreq_policy_data *policy)
{
    cpufreq_verify_within_limits(policy, policy->cpuinfo.min_freq,
                                 policy->cpuinfo.max_freq);
}
```

#### الحل

```c
static int rk3562_cpufreq_verify(struct cpufreq_policy_data *policy)
{
    /* Use hardware limits from cpuinfo, not hardcoded values */
    cpufreq_verify_within_cpu_limits(policy);
    return 0;
}
```

وفي الـ Device Tree:

```dts
/* arch/arm64/boot/dts/rockchip/rk3562-gateway.dts */
&cpu0 {
    operating-points-v2 = <&cpu0_opp_table>;
};

&cpu0_opp_table {
    opp-408000000 {
        opp-hz = /bits/ 64 <408000000>;
        opp-microvolt = <850000>;
    };
    opp-1200000000 {
        opp-hz = /bits/ 64 <1200000000>;
        opp-microvolt = <1000000>;
    };
    opp-1512000000 {
        opp-hz = /bits/ 64 <1512000000>;
        opp-microvolt = <1100000>;
    };
};
```

```bash
# التحقق بعد الإصلاح
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
408000
```

#### الدرس المستفاد
**الـ `cpufreq_verify_within_limits` مش للـ security** — هي بس للـ sanitization. لو hardcoded الـ min فيها، بتكسر قدرة الـ governor على التوفير في الطاقة. دايمًا استخدم `cpufreq_verify_within_cpu_limits` إلا لو عندك سبب hardware حقيقي.

---

### السيناريو الثاني: STM32MP1 — Android TV Box بيحصله Thermal Throttle مش متوقع

#### العنوان
**الـ `CPUFREQ_IS_COOLING_DEV` flag مش set والـ thermal framework مش بيتحكم في الـ CPU freq**

#### السياق
مطور بيعمل Android TV box على STM32MP157 (dual Cortex-A7 + Cortex-M4). الجهاز بيستخدم Mali GPU وبيشغل 4K video decoding. الـ thermal sensor بيسجل 95°C على الـ SoC لكن الـ CPU مش بينزل في الـ frequency.

#### المشكلة

```bash
$ cat /sys/class/thermal/thermal_zone0/temp
95000

# الـ CPU لسه بيشتغل بأقصى فريكوانسي!
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
650000

$ ls /sys/class/thermal/
thermal_zone0  # مفيش cooling_device!
```

الـ thermal framework مش شايف الـ CPU كـ cooling device.

#### التحليل

الـ driver الخاص بالـ STM32MP1 مش حاطط الـ flag:

```c
static struct cpufreq_driver stm32mp1_cpufreq_driver = {
    .name   = "stm32mp1-cpufreq",
    .flags  = CPUFREQ_NEED_INITIAL_FREQ_CHECK,
    /* BUG: Missing CPUFREQ_IS_COOLING_DEV */
    .init   = stm32mp1_cpufreq_init,
    .verify = cpufreq_generic_frequency_table_verify,
    .target_index = stm32mp1_set_target,
    .get    = cpufreq_generic_get,
    .attr   = cpufreq_generic_attr,
};
```

من الـ header:

```c
/* cpufreq.h السطر 447 */
/*
 * Set by drivers that want the core to automatically register the cpufreq
 * driver as a thermal cooling device.
 */
#define CPUFREQ_IS_COOLING_DEV  BIT(2)
```

والـ helper function اللي بتـ check الـ flag:

```c
/* cpufreq.h السطر 487 */
static inline int cpufreq_thermal_control_enabled(struct cpufreq_driver *drv)
{
    return IS_ENABLED(CONFIG_CPU_THERMAL) &&
        (drv->flags & CPUFREQ_IS_COOLING_DEV);
}
```

**لو الـ flag مش set، الـ core مش بيعمل `cpufreq_cooling_register()` تلقائيًا** — يعني الـ thermal framework مش عارف إن الـ CPU ممكن يتبرد بتقليل الـ frequency.

#### الحل

```c
static struct cpufreq_driver stm32mp1_cpufreq_driver = {
    .name   = "stm32mp1-cpufreq",
    .flags  = CPUFREQ_NEED_INITIAL_FREQ_CHECK |
              CPUFREQ_IS_COOLING_DEV,  /* Register as thermal cooling device */
    .init   = stm32mp1_cpufreq_init,
    .verify = cpufreq_generic_frequency_table_verify,
    .target_index = stm32mp1_set_target,
    .get    = cpufreq_generic_get,
    .attr   = cpufreq_generic_attr,
};
```

وفي الـ Device Tree لازم تربط الـ thermal zone بالـ CPU:

```dts
thermal-zones {
    cpu_thermal: cpu-thermal {
        polling-delay-passive = <250>;
        polling-delay = <1000>;
        thermal-sensors = <&stm32_temp>;

        trips {
            cpu_alert: cpu-alert {
                temperature = <85000>;
                hysteresis = <2000>;
                type = "passive";
            };
            cpu_crit: cpu-crit {
                temperature = <95000>;
                hysteresis = <5000>;
                type = "critical";
            };
        };

        cooling-maps {
            map0 {
                trip = <&cpu_alert>;
                cooling-device = <&cpu0 THERMAL_NO_LIMIT THERMAL_NO_LIMIT>;
            };
        };
    };
};
```

```bash
# بعد الإصلاح
$ ls /sys/class/thermal/
cooling_device0  thermal_zone0

$ cat /sys/class/thermal/cooling_device0/type
cpufreq-cpu0

# الـ policy struct بتشيل pointer للـ cooling device
# policy->cdev مش NULL تاني
```

#### الدرس المستفاد
الـ `cpufreq_policy` بيحتفظ بـ `struct thermal_cooling_device *cdev` — بس ده بيتملى بس لو الـ driver عمل set لـ `CPUFREQ_IS_COOLING_DEV`. من غير الـ flag، الـ SoC ممكن يتحرق حرفيًا.

---

### السيناريو الثالث: i.MX8MQ — IoT Sensor Node بيـ Deadlock عند CPU Hotplug

#### العنوان
**`cpufreq_policy->rwsem` deadlock بسبب lock غلط في custom driver callback**

#### السياق
شركة بتعمل industrial IoT sensor node على i.MX8MQ (quad-core Cortex-A53). الـ product بيعمل CPU hotplug عشان توفير الطاقة — بيشغل cores زيادة لما في load عالي. بعد kernel upgrade لـ 6.6، الـ system بيـ hang عند online/offline للـ cores.

#### المشكلة

```bash
$ echo 0 > /sys/devices/system/cpu/cpu3/online
[  42.318745] INFO: task sh:1234 blocked for more than 120 seconds.
[  42.318748] "echo 0 >" is blocked on rwsem:
[  42.318749]  down_write+0x28/0x60
[  42.318750]  cpufreq_offline+0x4c/0x1c0
```

الـ kernel hang — الـ rwsem مـ بيتحررش.

#### التحليل

الـ driver الخاص بيقرأ من الـ policy جوا الـ `offline()` callback بـ طريقة غلط:

```c
static int imx8mq_cpufreq_offline(struct cpufreq_policy *policy)
{
    /* BUG: Trying to acquire read lock while write lock is already held */
    struct cpufreq_policy *p = cpufreq_cpu_get(policy->cpu);
    if (!p)
        return -ENODEV;

    /* Do something with p... */
    cpufreq_cpu_put(p);
    return 0;
}
```

من الـ header، `cpufreq_cpu_get()` بتعمل `down_read()` على الـ rwsem:

```c
/* cpufreq.h السطر 207 */
struct cpufreq_policy *cpufreq_cpu_get(unsigned int cpu);
```

لكن الـ core لما بيستدعي `offline()` callback، هو **أصلًا شايل الـ write lock** على نفس الـ rwsem:

```c
/* من cpufreq.h السطر 100 */
/*
 * - Any routine that will write to the policy structure and/or may take away
 *   the policy altogether (eg. CPU hotplug), will hold this lock in write
 *   mode before doing so.
 */
struct rw_semaphore rwsem;
```

والـ guard macros المعرفة في الـ header:

```c
/* cpufreq.h السطر 171 */
DEFINE_GUARD(cpufreq_policy_write, struct cpufreq_policy *,
             down_write(&_T->rwsem), up_write(&_T->rwsem))

DEFINE_GUARD(cpufreq_policy_read, struct cpufreq_policy *,
             down_read(&_T->rwsem), up_read(&_T->rwsem))
```

**الـ deadlock:** core حاطط write lock → driver callback بيطلب read lock → `rwsem` مش بيسمح read وفيه write pending على نفس الـ thread.

#### الحل

```c
static int imx8mq_cpufreq_offline(struct cpufreq_policy *policy)
{
    /*
     * CORRECT: policy is already passed to us, no need to call
     * cpufreq_cpu_get() which would try to acquire a read lock
     * on a write-locked rwsem → deadlock!
     */
    unsigned int cpu = policy->cpu;
    unsigned int cur_freq = policy->cur;

    /* Use policy directly without re-fetching it */
    dev_dbg(get_cpu_device(cpu), "Going offline at %u kHz\n", cur_freq);

    /* Save state for resume */
    imx8mq_save_freq_state(policy);

    return 0;
}
```

```bash
# بعد الإصلاح
$ echo 0 > /sys/devices/system/cpu/cpu3/online
[  42.318745] CPU3: shutdown
$ echo 1 > /sys/devices/system/cpu/cpu3/online
[  43.128654] CPU3: online
```

#### الدرس المستفاد
الـ `cpufreq_policy->rwsem` مش reentrant — لو الـ core بيستدعي الـ driver callback وهو شايل write lock، الـ driver **ممنوع** يطلب read lock على نفس الـ policy. الـ `policy` pointer اللي بتجيله كافي — ما تعملش `cpufreq_cpu_get()` من جوا callback.

---

### السيناريو الرابع: AM62x — Automotive ECU بيعمل Hard Lock عند Fast Switch

#### العنوان
**`fast_switch_possible` بيتعمل set بدون `fast_switch_enabled` — governor بيـ crash**

#### السياق
شركة automotive بتعمل ECU على TI AM62x (quad-core Cortex-A53) لنظام infotainment. الـ product بيستخدم schedutil governor مع fast frequency switching عشان latency منخفض جدًا (< 10µs). عند اختبار الـ system تحت load عالي (navigation + audio + CAN bus)، الـ kernel بيعمل BUG_ON.

#### المشكلة

```
[  128.441832] kernel BUG at drivers/cpufreq/cpufreq.c:2134!
[  128.441840] Call trace:
[  128.441842]  cpufreq_driver_fast_switch+0x88/0x140
[  128.441843]  sugov_update_shared+0x1cc/0x220
[  128.441844]  update_load_avg+0x344/0x480
```

#### التحليل

الـ driver بيعمل set لـ `fast_switch_possible` في الـ `init()` callback:

```c
static int am62x_cpufreq_init(struct cpufreq_policy *policy)
{
    policy->fast_switch_possible = true;  /* OK: driver supports it */
    /* BUG: policy->fast_switch_enabled is NOT set here */
    /* That's the governor's job — but governor wasn't told to enable it */
    return 0;
}
```

من الـ header، الـ flag semantics:

```c
/* cpufreq.h السطر 110 */
/*
 * Fast switch flags:
 * - fast_switch_possible should be set by the driver if it can
 *   guarantee that frequency can be changed on any CPU sharing the
 *   policy and that the change will affect all of the policy CPUs then.
 * - fast_switch_enabled is to be set by governors that support fast
 *   frequency switching with the help of cpufreq_enable_fast_switch().
 */
bool fast_switch_possible;
bool fast_switch_enabled;
```

**المشكلة:** schedutil بيشوف إن `fast_switch_possible = true` فبيستدعي `cpufreq_enable_fast_switch()` لكن الـ driver نفسه بيـ register الـ `fast_switch` callback بشكل غلط:

```c
static struct cpufreq_driver am62x_cpufreq_driver = {
    .name         = "am62x-cpufreq",
    .fast_switch  = NULL,  /* BUG: fast_switch_possible=true but callback is NULL! */
    .target_index = am62x_set_target,
    /* ... */
};
```

لما schedutil بيستدعي `cpufreq_driver_fast_switch()` من scheduler hotpath:

```c
/* من cpufreq.h السطر 618 */
unsigned int cpufreq_driver_fast_switch(struct cpufreq_policy *policy,
                                        unsigned int target_freq);
```

الـ core بيـ BUG_ON لأن الـ `driver->fast_switch` = NULL.

#### الحل

```c
/* Option 1: Implement fast_switch properly */
static unsigned int am62x_fast_switch(struct cpufreq_policy *policy,
                                       unsigned int target_freq)
{
    /* Must be callable from scheduler hotpath — no sleeping! */
    unsigned int idx;

    idx = cpufreq_table_find_index_l(policy, target_freq, false);
    if (idx < 0)
        return policy->cur;

    /* Write to SoC PLL register directly — no mutex */
    am62x_pll_set_rate(policy->cpu, policy->freq_table[idx].frequency);

    return policy->freq_table[idx].frequency;
}

static struct cpufreq_driver am62x_cpufreq_driver = {
    .name         = "am62x-cpufreq",
    .fast_switch  = am62x_fast_switch,  /* Now consistent with fast_switch_possible */
    .target_index = am62x_set_target,
    /* ... */
};

static int am62x_cpufreq_init(struct cpufreq_policy *policy)
{
    policy->fast_switch_possible = true;
    /* fast_switch_enabled is set by the governor via cpufreq_enable_fast_switch() */
    policy->transition_delay_us = 100; /* hint to governor */
    return 0;
}
```

```bash
# التحقق
$ cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_transition_latency
100000

# schedutil بيستخدم fast_switch path
$ grep fast_switch /proc/cpufreq_debug
fast_switch_enabled: 1
fast_switch_count:   284729
```

#### الدرس المستفاد
الـ `fast_switch_possible` و`fast_switch_enabled` لهم roles مختلفة تمامًا. الـ driver بيـ declare القدرة (`possible`) — الـ governor هو اللي بيـ enable الاستخدام (`enabled`) عبر `cpufreq_enable_fast_switch()`. لازم تـ implement الـ `fast_switch` callback قبل ما تعمل `fast_switch_possible = true`.

---

### السيناريو الخامس: Allwinner H616 — Custom Board بيديه Frequency Jumps غريبة

#### العنوان
**`CPUFREQ_INEFFICIENT_FREQ` flag مش متحسبوش صح — scheduler بيختار أسوأ نقطة عمل**

#### السياق
مطور بيعمل custom board على Allwinner H616 (quad-core Cortex-A53 @ 1.5GHz) لمشغل media مبني على Linux. الـ board بيعمل OPP table معقد من الـ vendor، لكن اللي بيلاحظه المطور إن الـ CPU دايمًا بيقف عند 1.2GHz حتى لو الـ load بسيط — و1.2GHz دي voltage/performance ratio سيئ جدًا مقارنة بـ 1.0GHz.

#### المشكلة

```bash
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
480000 720000 1008000 1200000 1512000

# تحت load خفيف، schedutil بيختار 1200000 بدل 1008000
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
1200000

# energy model
$ cat /sys/devices/system/cpu/cpu0/cpufreq/../energy_model/state2/power
850  # 1008000 kHz
$ cat /sys/devices/system/cpu/cpu0/cpufreq/../energy_model/state3/power
1400 # 1200000 kHz — أعلى بكتير بدون فايدة حقيقية
```

الـ 1.2GHz voltage يرتفع بشكل كبير (1.05V vs 0.95V) مع زيادة طفيفة في الأداء.

#### التحليل

الـ driver مش بيعمل mark لـ 1200MHz كـ inefficient:

```c
static int h616_cpufreq_init(struct cpufreq_policy *policy)
{
    int ret;

    ret = cpufreq_generic_init(policy, h616_freq_table,
                               CPUFREQ_DEFAULT_TRANSITION_LATENCY_NS);
    if (ret)
        return ret;

    /* BUG: 1200MHz is inefficient (bad voltage/perf ratio) but not marked */
    /* policy->efficiencies_available stays false */
    return 0;
}
```

من الـ header، الـ mechanism:

```c
/* cpufreq.h السطر 712 */
#define CPUFREQ_INEFFICIENT_FREQ    (1 << 1)

/* cpufreq.h السطر 1128 */
static inline int
cpufreq_table_set_inefficient(struct cpufreq_policy *policy,
                               unsigned int frequency)
{
    struct cpufreq_frequency_table *pos;

    /* Not supported for unsorted tables */
    if (policy->freq_table_sorted == CPUFREQ_TABLE_UNSORTED)
        return -EINVAL;

    cpufreq_for_each_valid_entry(pos, policy->freq_table) {
        if (pos->frequency == frequency) {
            pos->flags |= CPUFREQ_INEFFICIENT_FREQ;      /* Mark it */
            policy->efficiencies_available = true;         /* Enable E-flag */
            return 0;
        }
    }
    return -EINVAL;
}
```

**لما `efficiencies_available = false`**، الـ `CPUFREQ_RELATION_E` flag في `cpufreq_frequency_table_target()` بيتـ ignore:

```c
/* cpufreq.h السطر 1069 */
static inline int cpufreq_frequency_table_target(...)
{
    bool efficiencies = policy->efficiencies_available &&
                        (relation & CPUFREQ_RELATION_E);  /* false if not available */
    /* ... */
}
```

يعني schedutil مع `CPUFREQ_RELATION_LE` كـ relation بيختار 1200MHz لأنه "أقل تردد فوق الـ target" — بدون اعتبار إنه inefficient.

#### الحل

```c
static int h616_cpufreq_init(struct cpufreq_policy *policy)
{
    int ret;

    ret = cpufreq_generic_init(policy, h616_freq_table,
                               CPUFREQ_DEFAULT_TRANSITION_LATENCY_NS);
    if (ret)
        return ret;

    /*
     * 1200MHz on H616 has poor energy efficiency:
     * voltage jumps from 0.95V (1008MHz) to 1.05V (1200MHz)
     * for only ~19% frequency gain. Mark it inefficient so
     * schedutil with CPUFREQ_RELATION_LE skips it when possible.
     */
    ret = cpufreq_table_set_inefficient(policy, 1200000);
    if (ret)
        dev_warn(get_cpu_device(policy->cpu),
                 "Failed to mark 1200MHz inefficient: %d\n", ret);

    return 0;
}
```

الـ OPP table في الـ Device Tree كمان ممكن تعمل hint:

```dts
&cpu0_opp_table {
    opp-1008000000 {
        opp-hz = /bits/ 64 <1008000000>;
        opp-microvolt = <950000>;
    };
    opp-1200000000 {
        opp-hz = /bits/ 64 <1200000000>;
        opp-microvolt = <1050000>;
        /* Note: some drivers read opp-level for efficiency hints */
    };
    opp-1512000000 {
        opp-hz = /bits/ 64 <1512000000>;
        opp-microvolt = <1100000>;
    };
};
```

```bash
# التحقق بعد الإصلاح
# تحت نفس الـ load الخفيف
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
1008000  # schedutil اختار الأكفأ الآن

# الـ 1200MHz بيتاخد بس لو مفيش خيار تاني
$ echo schedutil > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
$ stress-ng --cpu 1 --cpu-load 60 &
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
1008000  # still efficient, not 1200MHz

$ stress-ng --cpu 4 --cpu-load 90 &
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
1512000  # jumped directly to 1512MHz, skipped 1200MHz
```

#### الدرس المستفاد
الـ `cpufreq_table_set_inefficient()` + `CPUFREQ_INEFFICIENT_FREQ` flag هي آلية قوية بيتم تجاهلها كتير. لازم تحلل الـ OPP table بدقة وتحسب الـ performance-per-watt لكل نقطة تشغيل. نقطة بـ voltage/performance ratio سيئ مش بس مش مفيدة — هي ضارة لأن الـ governor بيختارها بدل ما يقفز لنقطة أفضل.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ** LWN.net هو المرجع الأول لمتابعة تطور الـ kernel — الروابط دي بتغطي رحلة الـ cpufreq من الـ 2.5 kernel لحد modern governors.

| المقال | الوصف |
|--------|-------|
| [CPUfreq Documentation (4/4)](https://lwn.net/Articles/8642/) | التوثيق الأصلي للـ cpufreq في زمن kernel 2.5 — الأساسيات من أول ما اتكتبت |
| [cpufreq core for 2.5](https://lwn.net/Articles/1476/) | إعلان إضافة الـ cpufreq core للـ kernel 2.5 — بداية الـ subsystem |
| [2.5.40 CPUFreq documentation](https://lwn.net/Articles/11548/) | توثيق مرحلة 2.5.40 مع تفاصيل الـ driver interface |
| [cpufreq: schedutil governor](https://lwn.net/Articles/678777/) | مقال بيشرح الـ schedutil — الـ governor اللي بيستخدم بيانات الـ scheduler مباشرة |
| [Improvements in CPU frequency management](https://lwn.net/Articles/682391/) | تحسينات الـ cpufreq في kernel 4.7 مع الـ schedutil |
| [CPU frequency governors and remote callbacks](https://lwn.net/Articles/732740/) | مناقشة آلية الـ remote DVFS وإزاي الـ governors بتتعامل مع CPUs مختلفة |
| [CPUFreq driver using CPPC methods](https://lwn.net/Articles/650781/) | الـ CPPC (Collaborative Processor Performance Control) — البديل الـ ACPI-based |
| [cpufreq: introduce a new AMD CPU frequency control mechanism](https://lwn.net/Articles/868671/) | الـ amd-pstate driver — آلية تحكم جديدة لـ AMD CPUs بدل الـ ACPI P-states |
| [cpufreq: ARM big LITTLE cpufreq driver](https://lwn.net/Articles/544457/) | الـ driver بتاع ARM big.LITTLE — مثال حي على multi-cluster policy |
| [cpufreq: Upstream Android's Interactive governor](https://lwn.net/Articles/700608/) | نقاش حول الـ interactive governor اللي Android كانت بتستخدمه |

---

### التوثيق الرسمي للـ kernel

**الـ** `Documentation/` في source tree هي المرجع الأدق والأحدث دايمًا.

#### مسارات مهمة في الـ kernel source

```
Documentation/cpu-freq/
├── core.rst              # شرح الـ cpufreq core وتصميمه
├── cpu-drivers.rst       # كيفية كتابة cpufreq driver
├── governors.rst         # شرح الـ governors المتاحة وكيفية الاختيار
├── frequency-plans.rst   # مفهوم الـ frequency tables
└── index.rst             # نقطة البداية الرئيسية

Documentation/admin-guide/pm/
├── cpufreq.rst           # دليل المستخدم — sysfs interface وكيفية الضبط
├── cpufreq-drivers.rst   # قائمة الـ drivers المتاحة مع نظرة عامة
└── amd-pstate.rst        # توثيق الـ amd-pstate driver تفصيليًا
```

#### روابط الـ kernel docs المباشرة

- [CPU Performance Scaling — docs.kernel.org](https://docs.kernel.org/admin-guide/pm/cpufreq.html)
- [Schedutil Governor Documentation](https://docs.kernel.org/scheduler/schedutil.html)
- [AMD P-State Driver Documentation](https://docs.kernel.org/admin-guide/pm/amd-pstate.html)
- [CPU Performance Scaling (v4.14)](https://www.kernel.org/doc/html/v4.14/admin-guide/pm/cpufreq.html)

---

### Commits مهمة في تاريخ الـ cpufreq

**الـ** commits دي هي نقاط تحول في تاريخ الـ subsystem — فهمها بيوضح ليه الـ API اتصمم بالشكل ده.

| الـ commit / الـ patch | التأثير |
|-------------------------|---------|
| [schedutil: New governor based on scheduler utilization data](https://lore.kernel.org/lkml/1855303.8Nyy7L51ju@vostro.rjw.lan/) | إضافة الـ schedutil governor — Rafael Wysocki |
| [cpufreq: schedutil v4 patch](https://lore.kernel.org/lkml/11678919.CQLTrQTYxG@vostro.rjw.lan/) | آخر iteration قبل الـ merge في kernel 4.7 |
| [cpufreq: governor: Get rid of governor events](https://lore.kernel.org/lkml/2126576.ycyxhAHC04@vostro.rjw.lan/) | تبسيط الـ governor lifecycle — إزالة الـ events القديمة |
| [schedutil: New governor based on scheduler](https://patchwork.kernel.org/project/linux-pm/patch/1842158.0Xhak3Uaac@vostro.rjw.lan/) | الـ patchwork thread الكامل مع الـ review |

#### إزاي تلاقي commits بنفسك

```bash
# ابحث في git log عن تغييرات الـ cpufreq
git log --oneline -- include/linux/cpufreq.h

# شوف من اتغير في الـ cpufreq core
git log --oneline drivers/cpufreq/cpufreq.c | head -30

# ابحث عن commit معين
git log --all --grep="cpufreq: schedutil"
```

---

### نقاشات الـ mailing list

**الـ** Linux PM mailing list هي المكان اللي بتتقرر فيه الـ design decisions.

- [linux-pm@vger.kernel.org archive](https://lore.kernel.org/linux-pm/) — الـ archive الرسمي لكل نقاشات الـ power management
- [CPUFreq and The Scheduler Revolution — Linux Foundation slides](http://events17.linuxfoundation.org/sites/events/files/slides/cpufreq_and_scheduler_0.pdf) — عرض Rafael Wysocki عن تكامل الـ cpufreq مع الـ scheduler
- [EAS dependency on schedutil — mailing list](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1787233.html) — نقاش ربط الـ Energy Aware Scheduling بالـ schedutil

#### البحث في الـ lore.kernel.org

```
# ابحث عن موضوع معين
https://lore.kernel.org/linux-pm/?q=cpufreq+fast+switch

# شوف كل الـ threads بخصوص الـ schedutil
https://lore.kernel.org/linux-pm/?q=schedutil
```

---

### مواقع الـ embedded Linux

#### kernelnewbies.org — تغييرات الـ cpufreq عبر الإصدارات

**الـ** kernelnewbies بيوثق التغييرات المهمة في كل إصدار kernel — مفيد جدًا لفهم "إيه اللي اتغير ومتى".

| الإصدار | التغيير المهم في الـ cpufreq |
|---------|------------------------------|
| [Linux 4.7](https://kernelnewbies.org/Linux_4.7) | إضافة الـ schedutil governor — نقطة تحول كبيرة |
| [Linux 4.9](https://kernelnewbies.org/Linux_4.9) | تحسينات الـ schedutil وتكاملها مع الـ scheduler |
| [Linux 4.10](https://kernelnewbies.org/Linux_4.10) | intel_pstate بيشتغل كـ generic cpufreq driver |
| [Linux 4.11](https://kernelnewbies.org/Linux_4.11) | إضافة `cpufreq.off=1` boot option |
| [Linux 4.14](https://kernelnewbies.org/Linux_4.14) | الـ scheduler بيحدّث الـ cpufreq policy على remote CPUs |
| [Linux 5.7](https://kernelnewbies.org/Linux_5.7) | frequency invariant scheduling على x86 — intel_pstate → schedutil |
| [Linux 3.12](https://kernelnewbies.org/Linux_3.12) | إصلاح oscillation في الـ ondemand governor |

#### elinux.org — الـ embedded perspective

- [Tests: R-CAR-GEN3-CPUFreq](https://elinux.org/Tests:R-CAR-GEN3-CPUFreq) — كيفية اختبار الـ cpufreq على Renesas R-Car Gen3 — مثال حي على الـ sysfs verification
- [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) — مرجع عام بيحتوي على روابط لـ cpufreq وغيره من الـ subsystems

---

### الكتب الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)

**الكتاب** ده free online ومتاح على [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/).

- **Chapter 1** — Introduction: فهم الـ kernel subsystems بشكل عام
- **Chapter 14** — The Linux Device Model: الـ `kobject`, `sysfs`, و `kobj_attribute` — نفس الـ building blocks اللي الـ cpufreq بيستخدمها في الـ `struct cpufreq_policy`
- **Chapter 6** — Advanced Char Driver Operations: الـ `wait_queue_head_t` اللي موجودة في الـ `cpufreq_policy` لـ `transition_wait`

#### Linux Kernel Development, 3rd Edition — Robert Love

أفضل كتاب لفهم الـ kernel internals بشكل متوازن.

- **Chapter 4** — Process Scheduling: فهم الـ PELT وكيف الـ schedutil بتقرأ منه
- **Chapter 13** — The Virtual Filesystem: فهم الـ sysfs model الـ cpufreq بيقوم عليه
- **Chapter 17** — Devices and Modules: الـ driver model والـ kobject hierarchy

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan

- **Chapter 16** — Kernel Debugging Techniques: بيشمل مثال على debug الـ cpufreq
- **Chapter 15** — Kernel Initialization: فهم الـ `core_initcall` اللي بيستخدمها `cpufreq_governor_init`

#### Professional Linux Kernel Architecture — Wolfgang Mauerer

- **Chapter 18** — بيشرح الـ power management subsystem بشكل عميق بما فيه الـ cpufreq

---

### مصادر إضافية متخصصة

#### أدوات الـ userspace

- [cpupower documentation](https://www.kernel.org/doc/html/latest/admin-guide/pm/cpufreq.html) — الأداة الرسمية للتعامل مع الـ cpufreq من الـ userspace
- [cpupowerutils on LWN](https://lwn.net/Articles/433002/) — الـ cpufrequtils الموسعة

#### الـ ACPI والـ CPPC

- [ACPI Specification](https://uefi.org/specifications) — الفصل الخاص بـ P-states وCPPC في ACPI 5.0+
- [CPUFreq driver using CPPC](https://lwn.net/Articles/650781/) — تفاصيل الـ CPPC implementation في الـ kernel

#### ArchWiki — مرجع عملي ممتاز

- [CPU frequency scaling — ArchWiki](https://wiki.archlinux.org/title/CPU_frequency_scaling) — دليل عملي شامل للـ governors والـ drivers والـ sysfs interface

---

### مسارات الـ sysfs للفهم العملي

```bash
# شوف الـ cpufreq policy الحالية
ls /sys/devices/system/cpu/cpu0/cpufreq/

# الـ governor الحالي
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# الـ frequencies المتاحة
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies

# الـ driver الـ active
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver

# الـ stats — بيعكس الـ cpufreq_stats في الـ cpufreq_policy
ls /sys/devices/system/cpu/cpu0/cpufreq/stats/
```

---

### كلمات البحث للعثور على مزيد من المعلومات

```
# بحث تقني في الـ kernel
cpufreq_policy rwsem locking
cpufreq fast_switch schedutil
cpufreq_driver target_index implementation
DVFS linux kernel scheduler integration
cpufreq thermal cooling device integration
cpufreq QoS freq_constraints linux
cpufreq boost freq linux kernel
linux cpufreq energy model EAS
cpufreq adjust_perf callback
cpufreq inefficient frequency table

# بحث عام
linux DVFS dynamic voltage frequency scaling
linux P-states C-states power management
intel_pstate vs acpi-cpufreq comparison
amd-pstate CPPC linux driver
schedutil governor tuning linux
```

---

### ملخص أهم الروابط

| النوع | الرابط |
|-------|--------|
| التوثيق الرسمي | [docs.kernel.org/admin-guide/pm/cpufreq.html](https://docs.kernel.org/admin-guide/pm/cpufreq.html) |
| schedutil LWN | [lwn.net/Articles/678777](https://lwn.net/Articles/678777/) |
| تحسينات الـ cpufreq LWN | [lwn.net/Articles/682391](https://lwn.net/Articles/682391/) |
| remote callbacks LWN | [lwn.net/Articles/732740](https://lwn.net/Articles/732740/) |
| AMD pstate LWN | [lwn.net/Articles/868671](https://lwn.net/Articles/868671/) |
| linux-pm mailing list | [lore.kernel.org/linux-pm](https://lore.kernel.org/linux-pm/) |
| elinux CPUFreq test | [elinux.org/Tests:R-CAR-GEN3-CPUFreq](https://elinux.org/Tests:R-CAR-GEN3-CPUFreq) |
| ArchWiki | [CPU frequency scaling](https://wiki.archlinux.org/title/CPU_frequency_scaling) |
| amd-pstate docs | [docs.kernel.org/admin-guide/pm/amd-pstate.html](https://docs.kernel.org/admin-guide/pm/amd-pstate.html) |
## Phase 8: Writing simple module

### الـ Hook المختار: `CPUFREQ_TRANSITION_NOTIFIER`

اخترنا الـ **cpufreq transition notifier** لأنه آمن تماماً ومفيد — بيتشغل كل مرة الـ kernel بيغير تردد CPU، وبيوفر بيانات واضحة (التردد القديم والجديد وإيه الـ CPU المتأثر).

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * cpufreq_watch.c - Monitor CPU frequency transitions via notifier chain.
 *
 * Registers a CPUFREQ_TRANSITION_NOTIFIER to log every frequency change
 * (old_freq -> new_freq) for each CPU policy.
 */

/* Core kernel headers */
#include <linux/module.h>       /* MODULE_* macros, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/notifier.h>     /* notifier_block, notifier chain API */
#include <linux/cpufreq.h>      /* cpufreq_freqs, CPUFREQ_PRECHANGE/POSTCHANGE,
                                   cpufreq_register_notifier,
                                   cpufreq_unregister_notifier,
                                   CPUFREQ_TRANSITION_NOTIFIER */
#include <linux/cpu.h>          /* cpu_possible_mask */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example Author <author@example.com>");
MODULE_DESCRIPTION("Log CPU frequency transitions using the cpufreq notifier chain");

/*
 * cpufreq_transition_callback - called by cpufreq core before and after
 * every frequency change.
 *
 * @nb:     pointer to our notifier_block (embedded in a larger struct if needed)
 * @event:  CPUFREQ_PRECHANGE or CPUFREQ_POSTCHANGE
 * @data:   pointer to struct cpufreq_freqs carrying policy, old, new, flags
 *
 * Returns NOTIFY_OK to let the notifier chain continue to the next callback.
 */
static int cpufreq_transition_callback(struct notifier_block *nb,
                                       unsigned long event,
                                       void *data)
{
    /* data is always struct cpufreq_freqs* for CPUFREQ_TRANSITION_NOTIFIER */
    struct cpufreq_freqs *freqs = (struct cpufreq_freqs *)data;

    /*
     * freqs->policy->cpu  : the managing CPU for this policy
     * freqs->old          : frequency before transition (kHz)
     * freqs->new          : frequency after  transition (kHz)
     *
     * We only log POSTCHANGE to avoid duplicate lines per transition;
     * PRECHANGE fires before the hardware switch, POSTCHANGE fires after.
     */
    if (event == CPUFREQ_POSTCHANGE) {
        pr_info("cpufreq_watch: CPU%u  %u kHz -> %u kHz  (%.1f MHz -> %.1f MHz)\n",
                freqs->policy->cpu,
                freqs->old,
                freqs->new,
                freqs->old / 1000.0f,   /* convert kHz to MHz for readability */
                freqs->new / 1000.0f);
    }

    /* NOTIFY_OK = success, chain continues to other registered notifiers */
    return NOTIFY_OK;
}

/*
 * notifier_block registration struct.
 * .notifier_call : our callback function above.
 * .priority      : 0 = default priority (runs after higher-priority notifiers).
 */
static struct notifier_block cpufreq_watch_nb = {
    .notifier_call = cpufreq_transition_callback,
    .priority      = 0,
};

/* ------------------------------------------------------------------ */
/*                          module_init / module_exit                  */
/* ------------------------------------------------------------------ */

static int __init cpufreq_watch_init(void)
{
    int ret;

    /*
     * cpufreq_register_notifier() adds our notifier_block to the
     * CPUFREQ_TRANSITION_NOTIFIER chain inside the cpufreq core.
     * From this point on, every frequency transition calls our callback.
     */
    ret = cpufreq_register_notifier(&cpufreq_watch_nb,
                                    CPUFREQ_TRANSITION_NOTIFIER);
    if (ret) {
        pr_err("cpufreq_watch: failed to register notifier, err=%d\n", ret);
        return ret;
    }

    pr_info("cpufreq_watch: loaded — monitoring all CPU frequency transitions\n");
    return 0;
}

static void __exit cpufreq_watch_exit(void)
{
    /*
     * Must unregister before the module is unloaded; otherwise the kernel
     * would call a function pointer that no longer exists in memory,
     * causing an instant kernel panic on the next frequency change.
     */
    cpufreq_unregister_notifier(&cpufreq_watch_nb,
                                CPUFREQ_TRANSITION_NOTIFIER);

    pr_info("cpufreq_watch: unloaded — notifier removed\n");
}

module_init(cpufreq_watch_init);
module_exit(cpufreq_watch_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ `MODULE_*` macros و`module_init`/`module_exit` |
| `linux/notifier.h` | تعريف `struct notifier_block` وكل دوال الـ chain |
| `linux/cpufreq.h` | `struct cpufreq_freqs`، الـ `CPUFREQ_*` constants، ودوال register/unregister |
| `linux/cpu.h` | ماكروهات related to CPU topology (مستخدمة indirectly عبر cpufreq) |

---

#### الـ Callback: `cpufreq_transition_callback`

الـ `data` بيجي كـ `void *` لأن نفس الـ notifier API بتستخدمه لأنواع مختلفة من الـ events، فبنعمل cast لـ `struct cpufreq_freqs *` اللي بيحتوي على `policy` (بيدينا رقم الـ CPU) و`old`/`new` (الترددات بالـ kHz).

بنفلتر على `CPUFREQ_POSTCHANGE` بس عشان نطبع السطر مرة واحدة بعد ما الـ hardware فعلاً غير التردد، مش قبله.

---

#### الـ `notifier_block` struct

الـ `priority = 0` معناه إن الـ kernel هيشغل الـ callbacks الأعلى priority الأول — إحنا مش محتاجين نتدخل قبل حد، بس نراقب. الـ `.notifier_call` هو الـ function pointer اللي الـ kernel هيستدعيه مباشرة.

---

#### `module_init` و `module_exit`

**الـ init** بيسجل الـ `notifier_block` في الـ `CPUFREQ_TRANSITION_NOTIFIER` chain اللي بتديرها الـ cpufreq core، فمن اللحظة دي كل تغيير تردد بيعدي علينا.

**الـ exit** لازم يلغي التسجيل قبل ما الـ module يتشال من الـ memory — لو ما عملناش ده، الـ kernel هيفضل عنده pointer لـ function اتمسحت، وأول frequency transition بعدها هتحصل kernel panic فورية.

---

### كيفية البناء والتشغيل

```bash
# Makefile بسيط
# obj-m += cpufreq_watch.o

make -C /lib/modules/$(uname -r)/build M=$PWD modules

sudo insmod cpufreq_watch.ko

# شوف الـ log
dmesg | grep cpufreq_watch

# لتوليد transitions على سيستم بيه cpufreq governor ديناميكي:
# شغّل load مثلاً: stress --cpu 4 &
# أو اكتب تردد يدوي: echo 1200000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed

sudo rmmod cpufreq_watch
```

**مثال على الـ output في dmesg:**

```
[  123.456] cpufreq_watch: loaded — monitoring all CPU frequency transitions
[  124.112] cpufreq_watch: CPU0  800000 kHz -> 1800000 kHz  (800.0 MHz -> 1800.0 MHz)
[  124.113] cpufreq_watch: CPU4  800000 kHz -> 2400000 kHz  (800.0 MHz -> 2400.0 MHz)
[  130.009] cpufreq_watch: CPU0  1800000 kHz -> 800000 kHz  (1800.0 MHz -> 800.0 MHz)
```

---

### ملاحظات أمان

- الـ callback بيتشغل في **process context** (الـ `CPUFREQ_TRANSITION_NOTIFIER` هو blocking notifier chain) فممكن تعمل `pr_info` بأمان.
- **ما تستخدمش** `GFP_KERNEL` allocation أو mutex جوا الـ callback إلا لو متأكد من الـ context — ولو محتاج، الـ blocking chain بتسمح بده.
- الـ module مش بيعدل أي policy أو تردد — بيراقب بس، فآمن تماماً على أي system.
