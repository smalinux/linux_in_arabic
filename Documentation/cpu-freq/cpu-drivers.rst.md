## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem ده؟

**الـ CPUFreq subsystem** — ده الجزء اللي بيتحكم في سرعة الـ CPU أثناء تشغيل الـ Linux kernel. اسمه الكامل هو **CPU Frequency Scaling**، وهو جزء من منظومة إدارة الطاقة في الـ kernel.

الملف ده (`cpu-drivers.rst`) هو **دليل المطور اللي عايز يكتب driver جديد لـ CPUFreq**. يعني مش بيشرح ازاي تستخدم الـ feature، لكن بيشرح ازاي تعمل الـ plugin اللي يخلي الـ kernel يعرف يتحكم في سرعة chip معينة.

---

### القصة الكاملة: ليه أصلاً الـ CPU Frequency Scaling موجود؟

تخيل معايا إنك عندك لابتوب. لما بتشتغل على Word أو بتتكلم على Zoom، الـ CPU مش محتاج يشتغل بأقصى سرعته. لو شغّلته بأقصى سرعة طول الوقت هيحصل حاجتين وحشتين:

1. **البطارية هتخلص بسرعة** — لأن الـ CPU بيستهلك كهرباء أكتر كل ما السرعة زادت.
2. **الحرارة هترتفع** — الـ CPU هيسخن، والمروحة هتعمل ضوضاء، وممكن يتحرق.

المنطق بسيط: **اشتغل بتاع للشغل اللي محتاجه بس، ومتبذلش طاقة زيادة.**

وده بالظبط اللي بيعمله الـ **CPUFreq subsystem** — بيخلي الـ kernel يرفع سرعة الـ CPU وقت الضغط الكبير، ويخفضها وقت الهدوء.

---

### مين اللي بيتحكم في السرعة فعلاً؟

في 3 طبقات في الموضوع ده:

```
+---------------------------+
|       User / Apps         |  ← بيطلبوا شغل
+---------------------------+
           |
+---------------------------+
|    CPUFreq Governor       |  ← بيقرر: "هل نرفع السرعة؟"
|  (ondemand, schedutil...) |    زي مدير الموارد البشرية
+---------------------------+
           |
+---------------------------+
|    CPUFreq Core           |  ← بينسق ويرسل الأوامر
|   (drivers/cpufreq/       |    زي المدير التنفيذي
|    cpufreq.c)             |
+---------------------------+
           |
+---------------------------+
|    CPUFreq Driver         |  ← بيكلم الـ Hardware فعلاً
|  (intel_pstate, amd-pstate|    زي العامل اللي بيحرك الماكينة
|   acpi-cpufreq, ...)      |
+---------------------------+
           |
+---------------------------+
|       CPU Hardware        |  ← الـ chip نفسه
+---------------------------+
```

الملف ده بيتكلم عن **الطبقة التحتانية: الـ CPUFreq Driver**.

---

### الـ CPUFreq Driver — مهمته إيه؟

الـ driver هو الوحيد اللي **بيكلم الـ hardware فعلاً**. الـ core والـ governor بيعرفوش أي حاجة عن الـ chip — هما بس بيقولوا "عايزين تردد X". الـ driver هو اللي بيعرف:

- الـ CPU ده بيدعم تردد كام؟ (مثلاً: 800 MHz, 1.2 GHz, 1.8 GHz, 2.4 GHz)
- ازاي يغير التردد ده؟ (عن طريق register معين، أو BIOS interface، أو ACPI)
- قد إيه الوقت اللي بياخده التغيير؟ (transition latency)

---

### المشكلة اللي الـ Driver بيحلها

تخيل إن Qualcomm عملوا chip جديدة اسمها Snapdragon X. الـ Linux kernel مش هيعرف ازاي يتحكم في سرعتها تلقائياً. محتاج حد يكتب **driver مخصص** بيقول للـ kernel:

- "الـ chip دي بتدعم الترددات دي بالظبط: 300, 600, 900, 1200 MHz"
- "عشان تغير التردد، اكتب القيمة دي في الـ register ده"
- "التغيير بياخد 200 nanoseconds"

ده بالظبط اللي الـ driver بيعمله — وده بالظبط اللي الـ documentation ده بيشرح ازاي تكتبه.

---

### مكونات الـ Driver: الـ `struct cpufreq_driver`

أهم struct في الموضوع كله هي `cpufreq_driver`. لما تكتب driver جديد، بتملي الـ struct دي بـ function pointers وبتسجلها مع الـ core:

```c
struct cpufreq_driver {
    char  name[CPUFREQ_NAME_LEN];  /* اسم الـ driver */
    u16   flags;                    /* خصائص */

    /* الإلزاميين */
    int   (*init)(struct cpufreq_policy *policy);    /* تهيئة لكل policy */
    int   (*verify)(struct cpufreq_policy_data *);   /* تحقق من صحة الإعدادات */

    /* اختار واحد من دول لتغيير التردد */
    int   (*target_index)(struct cpufreq_policy *policy, unsigned int index);
    unsigned int (*fast_switch)(struct cpufreq_policy *, unsigned int freq);
    int   (*setpolicy)(struct cpufreq_policy *policy);

    /* اختياريين */
    unsigned int (*get)(unsigned int cpu);           /* اقرأ التردد الحالي */
    int   (*suspend)(struct cpufreq_policy *policy);
    int   (*resume)(struct cpufreq_policy *policy);
    void  (*exit)(struct cpufreq_policy *policy);
    /* ... وغيرهم */
};
```

---

### طرق تغيير التردد: الفرق بينهم

| الطريقة | الاستخدام | ملاحظة |
|---|---|---|
| `target_index()` | لما بيدك index في جدول الترددات | الأكثر شيوعاً |
| `fast_switch()` | من داخل الـ scheduler مباشرة | بدون sleep، لازم تكون سريعة جداً |
| `setpolicy()` | الـ CPU نفسه بيتحكم في التردد | زي Intel Longrun |
| `target()` | deprecated | الطريقة القديمة |

---

### الـ Frequency Table: قائمة الترددات

معظم الـ CPUs مش بتتحكم في سرعة حرة — بتختار من قائمة محددة. الـ `cpufreq_frequency_table` هي مصفوفة بيحطها الـ driver بكل الترددات المتاحة:

```c
struct cpufreq_frequency_table {
    unsigned int  flags;        /* CPUFREQ_ENTRY_INVALID أو 0 */
    unsigned int  driver_data;  /* بيانات خاصة بالـ driver */
    unsigned int  frequency;    /* التردد بالـ kHz */
};
/* آخر entry لازم يكون CPUFREQ_TABLE_END */
```

الـ core بيتحقق من الجدول ده تلقائياً، ومفيش لازم الـ entries تكون مرتبة — بس لو كانت مرتبة، الـ DVFS هيكون أسرع.

---

### خطوات كتابة Driver جديد: الملخص

```
1. في module_init() → تحقق إن الـ kernel شغّال على الـ CPU الصح
2. امليء struct cpufreq_driver بالـ function pointers
3. سجّل الـ driver: cpufreq_register_driver(&my_driver)
4. في .init() → امليء policy->cpuinfo بالـ min/max freq والـ latency
5. في .verify() → تحقق إن الـ policy settings صحيحة
6. في .target_index() → غيّر التردد فعلاً في الـ hardware
7. في .exit() → نظّف الـ resources
```

---

### الـ Intermediate Frequency: ليه بنحتاجه؟

بعض الـ CPUs مش ممكن تنقل من تردد لتردد مباشرة — محتاج تمر بـ **تردد وسيط مستقر** أولاً. مثلاً، لو عايز تروح من 800 MHz لـ 2.4 GHz، لازم تعدي على 1.2 GHz كمحطة وسطى.

الـ driver بيعمل ده عن طريق:
- `get_intermediate()` → يرجّع التردد الوسيط المطلوب
- `target_intermediate()` → يوصّل الـ CPU للتردد الوسيط
- بعدين الـ core يكمّل بنفسه لـ `target_index()`

---

### الملفات المهمة في الـ Subsystem ده

#### الـ Core والـ Headers
| الملف | الدور |
|---|---|
| `/drivers/cpufreq/cpufreq.c` | قلب الـ subsystem كله — الـ core logic |
| `/drivers/cpufreq/freq_table.c` | helper functions للـ frequency table |
| `/include/linux/cpufreq.h` | كل الـ structs والـ macros والـ API |
| `/include/linux/sched/cpufreq.h` | واجهة الـ scheduler مع الـ CPUFreq |

#### الـ Governors (من بيقرر السياسة)
| الملف | الدور |
|---|---|
| `/drivers/cpufreq/cpufreq_ondemand.c` | يرفع السرعة عند الضغط |
| `/drivers/cpufreq/cpufreq_conservative.c` | يرفع بتدريج |
| `/drivers/cpufreq/cpufreq_performance.c` | أقصى سرعة دايماً |
| `/drivers/cpufreq/cpufreq_powersave.c` | أدنى سرعة دايماً |
| `/kernel/sched/cpufreq*.c` | schedutil governor — الـ scheduler نفسه بيتحكم |

#### Hardware Drivers (أمثلة)
| الملف | الدور |
|---|---|
| `/drivers/cpufreq/intel_pstate.c` | Intel modern CPUs |
| `/drivers/cpufreq/amd-pstate.c` | AMD modern CPUs |
| `/drivers/cpufreq/acpi-cpufreq.c` | ACPI-based platforms |
| `/drivers/cpufreq/cpufreq-dt.c` | Device Tree platforms (ARM) |
| `/drivers/cpufreq/longrun.c` | مثال على setpolicy (Transmeta) |
| `/drivers/cpufreq/qcom-cpufreq-hw.c` | Qualcomm CPUs |
| `/drivers/cpufreq/apple-soc-cpufreq.c` | Apple Silicon |

#### Documentation
| الملف | الدور |
|---|---|
| `/Documentation/cpu-freq/cpu-drivers.rst` | **الملف ده** — guide لكتابة driver |
| `/Documentation/cpu-freq/core.rst` | شرح الـ core والـ notifiers |
| `/Documentation/cpu-freq/cpufreq-stats.rst` | إحصائيات استخدام الترددات |
| `/Documentation/admin-guide/pm/cpufreq.rst` | دليل المستخدم من الـ sysfs |

---

### ملاحظة للقارئ

الـ file ده مش source code — هو **وثيقة هندسية** بتشرح الـ contract بين الـ driver والـ core. لو عندك chip جديدة وعايز تضيف دعم لها في الـ Linux، تبدأ من هنا. الـ APIs اللي بتتكلم عنها موجودة في `include/linux/cpufreq.h`، والـ implementation الفعلية للـ core موجودة في `drivers/cpufreq/cpufreq.c`.
## Phase 2: شرح الـ CPUFreq Framework

### المشكلة — ليه الـ CPUFreq موجود أصلاً؟

الـ CPU الحديث مش بيشتغل بـ frequency ثابتة. عنده عشرات الـ **operating points** — كل واحد فيهم عبارة عن جوز (voltage, frequency). لما تشيل الـ frequency بتوفر طاقة لأن الـ dynamic power بيتحسب بالمعادلة دي:

```
P = C × V² × f
```

يعني لو وصّلت الـ frequency لنص، الطاقة بتتقل بشكل كبير — وبتوفر حرارة أيضاً.

المشكلة إن كل CPU/SoC بيعمل ده بطريقة مختلفة:
- بعضهم بيكلم الـ PMIC عبر I2C عشان يغير الـ voltage.
- بعضهم بيكتب في register معين في الـ clock controller.
- بعضهم (زي Intel P-states) بيعمل كل ده internally بـ MSR واحد.
- بعضهم بيفوّض الموضوع كله للـ firmware عبر SCMI أو PSCI.

لو كل driver عمل الـ logic بتاعته من الصفر، هيبقى عندنا تكرار هائل وأي governor جديد هيحتاج يتعامل مع كل driver على حدا. **الـ CPUFreq framework** جه عشان يحل الـ separation of concerns دي.

---

### الحل — فصل "القرار" عن "التنفيذ"

الـ framework بيقسم المسؤولية على ٣ طبقات:

| الطبقة | المسؤولية | المثال |
|--------|-----------|--------|
| **Governor** | بيقرر إيه الـ frequency المناسبة دلوقتي | schedutil, ondemand, performance |
| **CPUFreq Core** | بينسق، بيبعت notifications، بيحمي الـ policy | `kernel/cpufreq.c` |
| **Driver** | بيعرف إزاي يوصل للـ hardware فعلياً | `drivers/cpufreq/arm_scmi.c` |

الـ governor مش محتاج يعرف إنك شغال على Cortex-A55 أو Cortex-X4. والـ driver مش محتاج يفكر في policy أو power management. الـ core بيربطهم ببعض.

---

### الـ Big Picture Architecture

```
User Space
    │
    │  /sys/devices/system/cpu/cpu0/cpufreq/
    │  (scaling_governor, scaling_min_freq, scaling_max_freq, ...)
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│                    CPUFreq Core                         │
│                  (kernel/cpufreq.c)                     │
│                                                         │
│  ┌──────────────┐        ┌────────────────────────┐    │
│  │  cpufreq_    │        │   Notifier Chain        │    │
│  │  policy      │◄──────►│ (thermal, perf, trace)  │    │
│  │  (per-group) │        └────────────────────────┘    │
│  └──────┬───────┘                                       │
│         │                                               │
│  ┌──────▼────────────────────────────────────────┐     │
│  │              Governor Interface               │     │
│  │   init / exit / start / stop / limits        │     │
│  └──────┬────────────────────────────────────────┘     │
│         │                                               │
└─────────┼───────────────────────────────────────────────┘
          │
    ┌─────▼─────────────────────────────────────────┐
    │              Governors                        │
    │  schedutil │ ondemand │ conservative │ perf   │
    │  (يقرروا الـ target frequency)               │
    └─────┬─────────────────────────────────────────┘
          │  target_index() / fast_switch() / setpolicy()
          │
    ┌─────▼─────────────────────────────────────────┐
    │         cpufreq_driver (Platform Driver)      │
    │                                               │
    │  ┌─────────────────────────────────────┐     │
    │  │       cpufreq_frequency_table       │     │
    │  │  [300MHz][600MHz][1.2GHz][1.8GHz]  │     │
    │  └─────────────────────────────────────┘     │
    └─────┬─────────────────────────────────────────┘
          │
    ┌─────▼──────────────────────────────────────────┐
    │              Hardware Layer                    │
    │  Clock Controller / PMIC / SCMI Firmware      │
    │  MSR (x86) / AVS (ARM) / OPP Table            │
    └────────────────────────────────────────────────┘
```

---

### أمثلة Real-World لـ Drivers

| Driver | المنصة | آلية التحكم |
|--------|--------|------------|
| `cpufreq-dt.c` | ARM Generic | Device Tree + OPP table + clk API |
| `arm_scmi.c` | ARM Cortex-M/A مع firmware | SCMI protocol over mailbox |
| `intel_pstate.c` | Intel x86 | MSR_IA32_PERF_CTL |
| `qcom-cpufreq-hw.c` | Snapdragon | CPRF hardware block |
| `longrun.c` | Transmeta | CPU-internal hardware governor |

---

### الـ Real-World Analogy — نظام تكييف المبنى

تخيل مبنى إداري كبير فيه نظام تكييف مركزي:

| عنصر الـ Analogy | المقابل في الـ CPUFreq |
|------------------|----------------------|
| **مستشعرات الحرارة في الغرف** | CPU load metrics (idle time, scheduler signals) |
| **نظام التحكم المركزي (BMS)** | الـ CPUFreq Core |
| **خوارزمية الترشيد** (قرار: هنرفع ولا نخفض التبريد؟) | الـ Governor (schedutil / ondemand) |
| **الـ Policy** (حدود مسموح بيها: 18°C - 24°C) | `cpufreq_policy` (min_freq / max_freq) |
| **ميكانيكي التكييف** (بيعرف إزاي يتحكم في كل وحدة بتاعتها) | الـ `cpufreq_driver` |
| **وحدة التكييف نفسها** (Carrier، Daikin، Trane) | الـ Hardware (PMIC، Clock Controller) |
| **سجل الأعطال والإشعارات** | الـ Notifier Chain (thermal، cpufreq_freqs) |

الخوارزمية (governor) مش محتاجة تعرف إن الوحدة Carrier أو Daikin — هي بس بتقول "ارفع التبريد لـ level 3". الميكانيكي (driver) هو اللي يعرف إن الـ Carrier محتاجة signal 0-10V وإن الـ Daikin محتاجة BACnet protocol.

نفس الـ schedutil governor بيشتغل على ARM Cortex-A و Intel Core i7 من غير ما يغير سطر واحد — لأن الـ driver هو اللي بيتعامل مع الـ hardware فعلياً.

---

### الـ Core Abstraction — الـ `cpufreq_policy`

ده القلب. كل اللي بيحصل في الـ CPUFreq بيتمحور حوالين الـ **`struct cpufreq_policy`**.

```c
struct cpufreq_policy {
    cpumask_var_t   cpus;          /* CPUs online الشايلين الـ policy دي */
    cpumask_var_t   related_cpus;  /* CPUs بتشارك نفس clock/voltage rail */

    unsigned int    cpu;           /* الـ CPU اللي بيدير الـ policy */

    struct cpufreq_cpuinfo cpuinfo; /* hardware limits (min/max/latency) */

    unsigned int    min;           /* الـ governor مش هيطلب أقل من كده */
    unsigned int    max;           /* الـ governor مش هيطلب أكتر من كده */
    unsigned int    cur;           /* الـ frequency الحالية */

    struct cpufreq_governor *governor; /* مين بيتحكم دلوقتي */
    struct cpufreq_frequency_table *freq_table; /* الـ OPPs المتاحة */

    bool    fast_switch_possible;  /* ممكن نغير الـ freq من الـ scheduler? */
    bool    fast_switch_enabled;   /* الـ governor فعّلها؟ */

    struct rw_semaphore rwsem;     /* حماية concurrent access */
    spinlock_t  transition_lock;   /* حماية أثناء الـ transition نفسه */

    struct freq_constraints constraints; /* QoS limits من subsystems تانية */
    /* ... */
};
```

**الـ policy هي وحدة التحكم الأساسية** — مش الـ CPU. لو عندك SoC بـ 4 cores بيشاركوا نفس الـ voltage rail، هيبقى عندك policy واحدة بس لل4 cores دول. الـ `related_cpus` mask بيحدد مين فيهم.

#### علاقة الـ Structs ببعض

```
cpufreq_driver
      │
      │ .init() → ينشئ
      ▼
cpufreq_policy ◄──────── cpufreq_governor
      │                       │
      │ .freq_table            │ .start() / .stop()
      ▼                       │ .limits()
cpufreq_frequency_table        │
  [0] { freq=300000, flags=0 } │
  [1] { freq=600000, flags=0 } │ (calls)
  [2] { freq=1200000, flags=BOOST } │
  [3] { freq=~0, flags=END }   ▼
                          target_index(policy, idx)
                               │
                               ▼
                          cpufreq_driver.target_index()
                               │
                               ▼
                          Hardware (CLK/PMIC/SCMI)
```

---

### الـ `cpufreq_driver` — عقد الـ Driver مع الـ Core

```c
struct cpufreq_driver {
    char    name[CPUFREQ_NAME_LEN];
    u16     flags;          /* CPUFREQ_HAVE_GOVERNOR_PER_POLICY, ... */
    void   *driver_data;    /* private data للـ driver */

    /* إجباري: تهيئة الـ policy لأول مرة */
    int     (*init)(struct cpufreq_policy *policy);

    /* إجباري: تأكد إن الـ policy values معقولة */
    int     (*verify)(struct cpufreq_policy_data *policy);

    /* اختار واحدة بس من الـ 4 دول: */
    int     (*setpolicy)(struct cpufreq_policy *policy);       /* hardware governor */
    int     (*target_index)(struct cpufreq_policy *policy,
                            unsigned int index);               /* الأفضل */
    int     (*target)(struct cpufreq_policy *policy,
                      unsigned int target_freq,
                      unsigned int relation);                  /* deprecated */
    unsigned int (*fast_switch)(struct cpufreq_policy *policy,
                                unsigned int target_freq);     /* scheduler context */

    /* للـ two-step transitions */
    unsigned int (*get_intermediate)(struct cpufreq_policy *policy,
                                     unsigned int index);
    int          (*target_intermediate)(struct cpufreq_policy *policy,
                                        unsigned int index);

    /* اختياريات */
    unsigned int (*get)(unsigned int cpu);      /* قرأ الـ frequency الحالية */
    int     (*bios_limit)(int cpu, unsigned int *limit);
    void    (*exit)(struct cpufreq_policy *policy);
    int     (*suspend)(struct cpufreq_policy *policy);
    int     (*resume)(struct cpufreq_policy *policy);
    void    (*ready)(struct cpufreq_policy *policy);

    bool    boost_enabled;
    int     (*set_boost)(struct cpufreq_policy *policy, int state);
    void    (*register_em)(struct cpufreq_policy *policy);  /* Energy Model */
};
```

---

### الـ Switching Callbacks — أيهم تستخدم؟

ده قرار مهم بيأثر على الـ performance والـ correctness:

```
                    ┌─────────────────────────────────┐
                    │  الـ Hardware بيدير الـ scaling   │
                    │  بنفسه في range معين؟            │
                    └──────────────┬──────────────────┘
                                   │
                        Yes ───────┘──────── No
                        │                    │
                        ▼                    ▼
                   setpolicy()      ┌────────────────────┐
              (Transmeta LongRun)   │ بتحتاج fast switch  │
                                    │ من scheduler?      │
                                    └────────┬───────────┘
                                             │
                                  Yes ───────┘──────── No
                                  │                    │
                                  ▼                    ▼
                            fast_switch()       target_index()
                       (schedutil, no sleep)   (الأكثر شيوعاً)
                                               + optionally
                                               get_intermediate()
                                               target_intermediate()
```

#### الـ fast_switch() — متى وليه؟

الـ scheduler (schedutil governor) بيعمل decisions من **interrupt context أو RCU read-side critical section**. لو هنغير الـ frequency من هنا، مش ممكن نعمل:
- `mutex_lock()` / `sleep()` / `kmalloc()`
- أي call ممكن يعمل reschedule

الـ `fast_switch()` callback مصمم يبقى **atomic-safe**، بيكتب في register مباشرة أو بيبعت IPI للـ core المناسب. الـ driver اللي بيطلع قيمة للـ `fast_switch_possible` بيضمن إن كل الـ CPUs اللي شايلين الـ policy يقدروا يغيروا الـ frequency بدون race conditions.

#### الـ Two-Step Transition — get_intermediate + target_intermediate

بعض الـ platforms مش بتقدر تتنقل من frequency لأخرى مباشرة بدون مرور بنقطة آمنة وسطية. مثال: لو هتغير الـ PLL source، لازم تعدي بـ bypass clock مؤقتاً.

```
current_freq ──► intermediate_freq ──► target_freq
  1.8 GHz            400 MHz            600 MHz
(PLL locked)    (bypass/safe clock)  (PLL relocked)
```

الـ core بيبعت notifications للـ `intermediate_freq` أيضاً حتى الـ consumers (زي الـ bus scaling) يعدّلوا نفسهم.

---

### الـ Frequency Table — `cpufreq_frequency_table`

```c
struct cpufreq_frequency_table {
    unsigned int  flags;        /* CPUFREQ_BOOST_FREQ, CPUFREQ_INEFFICIENT_FREQ */
    unsigned int  driver_data;  /* بيانات خاصة بالـ driver (OPP index مثلاً) */
    unsigned int  frequency;    /* بالـ kHz */
};
```

**الـ Sentinel Pattern:** الـ table بتتنهي بـ entry فيها `frequency = CPUFREQ_TABLE_END (~1u)`. الـ entries اللي فيها `CPUFREQ_ENTRY_INVALID (~0u)` بيتخطوها لما بتبحث عن valid frequency.

مثال table لـ Cortex-A55:

```c
static struct cpufreq_frequency_table a55_freq_table[] = {
    { .frequency = 300000,  .driver_data = 0 },  /* 300 MHz */
    { .frequency = 600000,  .driver_data = 1 },  /* 600 MHz */
    { .frequency = 900000,  .driver_data = 2 },  /* 900 MHz */
    { .frequency = CPUFREQ_ENTRY_INVALID },       /* skip - unstable OPP */
    { .frequency = 1200000, .driver_data = 4 },  /* 1.2 GHz */
    { .frequency = 1800000, .flags = CPUFREQ_BOOST_FREQ,
                             .driver_data = 5 },  /* 1.8 GHz - boost only */
    { .frequency = CPUFREQ_TABLE_END },
};
```

الـ macros للـ iteration:

```c
struct cpufreq_frequency_table *pos;
unsigned int idx;

/* iterate كل الـ entries (بما فيها invalid) */
cpufreq_for_each_entry(pos, driver_freq_table) {
    pr_info("freq: %u kHz\n", pos->frequency);
}

/* iterate الـ valid entries فقط مع index */
cpufreq_for_each_valid_entry_idx(pos, driver_freq_table, idx) {
    pr_info("[%u] freq: %u kHz\n", idx, pos->frequency);
}
```

**ليه بنستخدم index بدل pointer arithmetic؟**

لأن `target_index()` callback بياخد `index` مش pointer. والـ core بيحتاج يقدر يعمل lookup سريع بالـ index ده في الـ freq_table. عمل `(pos - table)` ممكن يكون costly على بعض الـ architectures وبيكسر الـ abstraction.

---

### الـ CPUFREQ_RELATION flags — إزاي الـ Core بيختار الـ Frequency؟

لما الـ governor بيطلب frequency معينة، الـ core بيحتاج يلاقي أقرب entry في الـ table. الـ relation بيحدد الاتجاه:

| Flag | المعنى | الاستخدام |
|------|--------|-----------|
| `CPUFREQ_RELATION_L` | أقل frequency عند أو فوق الـ target | conservative، "لا تروح تحت كده" |
| `CPUFREQ_RELATION_H` | أعلى frequency عند أو تحت الـ target | ondemand، "لا تتجاوز كده" |
| `CPUFREQ_RELATION_C` | أقرب frequency للـ target | schedutil في بعض الحالات |
| `CPUFREQ_RELATION_E` (bit flag) | فضّل الـ efficient frequencies | يتـ OR مع أي من الـ 3 فوق |

الـ `CPUFREQ_INEFFICIENT_FREQ` flag في الـ frequency table بيسمح للـ core يتجنب frequencies معينة لو الـ governor طلب كده — مثلاً Cortex-X بيكون أقل كفاءة عند 1GHz مقارنة بـ 800MHz أو 1.2GHz.

---

### الـ verify() Callback — الـ Sanity Check

لما user يكتب:
```bash
echo 500000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
```

الـ core بيبني `cpufreq_policy_data` وبيبعتها لـ `driver->verify()`. الـ driver لازم يتأكد إن:
1. على الأقل frequency valid واحدة موجودة بين `policy->min` و `policy->max`.
2. الـ values موجودة ضمن الـ hardware limits الفعلية.

```c
/* أبسط implementation */
static int my_verify(struct cpufreq_policy_data *policy)
{
    /* يأكد إن min/max داخل حدود الـ hardware */
    cpufreq_verify_within_cpu_limits(policy);
    return 0;
}

/* inline في cpufreq.h */
static inline void cpufreq_verify_within_limits(
        struct cpufreq_policy_data *policy,
        unsigned int min, unsigned int max)
{
    policy->max = clamp(policy->max, min, max);
    policy->min = clamp(policy->min, min, policy->max);
    /* يضمن max >= min دايماً */
}
```

لاحظ إن الـ `verify()` بياخد `cpufreq_policy_data` مش `cpufreq_policy` الكاملة — ده intentional، لأن الـ callback مش المفروض يحدّث الـ `freq_table` أو أي state آخر، فقط يصحح الـ min/max.

---

### الـ Notifier Chain — الـ CPUFreq كـ Event Bus

الـ CPUFreq بيبعت events للـ subsystems التانية في نقطتين:

```
CPUFREQ_PRECHANGE  ──►  [ Thermal / Bus Scaling / cpufreq-stats ]  ──►  driver sets freq
                                                                          │
CPUFREQ_POSTCHANGE ◄──  [ Thermal / Bus Scaling / cpufreq-stats ]  ◄──  done
```

**الـ CPUFREQ_PRECHANGE** بيجي قبل التغيير عشان الـ dependent subsystems (زي الـ memory bus scaling) يجهزوا نفسهم.

**الـ CPUFREQ_POSTCHANGE** بيجي بعد ما التغيير حصل فعلاً عشان يحدّثوا state بتاعتهم.

الـ driver بيستخدم `cpufreq_freq_transition_begin()` و `cpufreq_freq_transition_end()` وبيسيب للـ core يبعت الـ notifications. لو الـ driver بيتعامل مع الـ notifications بنفسه (زي بعض الـ async drivers)، بيرفع `CPUFREQ_ASYNC_NOTIFICATION` flag.

---

### الـ Driver Flags — إشارات للـ Core

```c
/* الـ Core يفضل يستدعي الـ driver حتى لو الـ target frequency ما اتغيرتش */
#define CPUFREQ_NEED_UPDATE_LIMITS      BIT(0)

/* الـ loops_per_jiffy مش بتتأثر بالـ frequency transition */
#define CPUFREQ_CONST_LOOPS             BIT(1)

/* سجّل الـ driver كـ thermal cooling device تلقائياً */
#define CPUFREQ_IS_COOLING_DEV          BIT(2)

/* كل policy ليها governor settings مستقلة (multi-cluster) */
#define CPUFREQ_HAVE_GOVERNOR_PER_POLICY BIT(3)

/* الـ driver بيبعت POSTCHANGE notification بنفسه */
#define CPUFREQ_ASYNC_NOTIFICATION      BIT(4)

/* تأكد إن الـ CPU شغال على frequency موجودة في الـ table */
#define CPUFREQ_NEED_INITIAL_FREQ_CHECK BIT(5)

/* الـ driver مش متوافق مع governors بتعمل dynamic switching */
#define CPUFREQ_NO_AUTO_DYNAMIC_SWITCHING BIT(6)
```

**الـ `CPUFREQ_HAVE_GOVERNOR_PER_POLICY`** مهم جداً للـ big.LITTLE و heterogeneous SoCs. بدونه، الـ governor settings بتتشارك بين كل الـ policies. مع الـ flag ده، كل cluster (Cortex-A55 cluster vs Cortex-X1 cluster) بيبقى له governor tunables خاصة بيه في:

```
/sys/devices/system/cpu/cpu0/cpufreq/schedutil/  ← LITTLE cluster
/sys/devices/system/cpu/cpu4/cpufreq/schedutil/  ← big cluster
```

---

### الـ CPUFreq بيملك إيه ومش بيملك إيه؟

| الـ Core يملك (owns) | الـ Driver يملك (owns) |
|---------------------|----------------------|
| الـ policy lifecycle (create / destroy) | معرفة الـ voltage/frequency pairs |
| الـ transition locking والـ serialization | التواصل مع الـ clock controller / PMIC |
| بعت الـ notifier events | ملء الـ `freq_table` |
| الـ sysfs interface | قراءة الـ current frequency من الـ HW |
| اختيار الـ best frequency من الـ table | دعم الـ boost mode أو لأ |
| الـ QoS constraints enforcement | الـ suspend/resume sequence |
| الـ thermal cooling device registration | الـ intermediate frequency transitions |
| الـ cpufreq-stats | الـ BIOS/firmware frequency limits |

الـ core هو الـ policy enforcer — بيتأكد دايماً إن `min <= requested <= max` قبل ما يعدّي أي request للـ driver. الـ driver بيثق في الـ core يعمل ده وبيركز على التنفيذ الفعلي.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### Driver Flags — `cpufreq_driver.flags` (u16)

| Flag | Value | المعنى |
|------|-------|--------|
| `CPUFREQ_NEED_UPDATE_LIMITS` | `BIT(0)` | الـ driver محتاج يتعمل call حتى لو الـ target_freq ما اتغيرش، بس الـ min/max اتغيروا |
| `CPUFREQ_CONST_LOOPS` | `BIT(1)` | الـ `loops_per_jiffy` مش بتتأثر بالـ frequency transitions |
| `CPUFREQ_IS_COOLING_DEV` | `BIT(2)` | الـ core يسجّل الـ driver أوتوماتيك كـ thermal cooling device |
| `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` | `BIT(3)` | منصات بـ multiple clock-domains، كل policy ليها governor مستقل |
| `CPUFREQ_ASYNC_NOTIFICATION` | `BIT(4)` | الـ driver بيبعت POSTCHANGE notification برّا الـ `->target()` |
| `CPUFREQ_NEED_INITIAL_FREQ_CHECK` | `BIT(5)` | الـ core يتحقق إن الـ CPU شغّال بـ freq موجودة في الـ freq_table |
| `CPUFREQ_NO_AUTO_DYNAMIC_SWITCHING` | `BIT(6)` | يمنع استخدام governors بيعملوا dynamic switching |

#### Governor Flags — `cpufreq_governor.flags` (u8)

| Flag | Value | المعنى |
|------|-------|--------|
| `CPUFREQ_GOV_DYNAMIC_SWITCHING` | `BIT(0)` | الـ governor بيغيّر الـ frequency لوحده ديناميكياً |
| `CPUFREQ_GOV_STRICT_TARGET` | `BIT(1)` | الـ target frequency لازم يتحدد بالظبط بدون تقريب |

#### Frequency Table Entry Flags

| Flag | Value | المعنى |
|------|-------|--------|
| `CPUFREQ_BOOST_FREQ` | `BIT(0)` | الـ entry ده boost frequency (أعلى من الحد العادي) |
| `CPUFREQ_INEFFICIENT_FREQ` | `BIT(1)` | الـ frequency دي inefficient، يُفضّل تجنبها |

#### Special Frequency Values

| Constant | Value | الاستخدام |
|----------|-------|-----------|
| `CPUFREQ_ENTRY_INVALID` | `~0u` | يعطّل entry في الـ table من غير ما يحذفه |
| `CPUFREQ_TABLE_END` | `~1u` | علامة نهاية الـ frequency table |

#### Policy Constants

| Constant | Value | المعنى |
|----------|-------|--------|
| `CPUFREQ_POLICY_UNKNOWN` | `0` | ما اتحددش policy بعد |
| `CPUFREQ_POLICY_POWERSAVE` | `1` | وضع توفير الطاقة (لـ setpolicy drivers) |
| `CPUFREQ_POLICY_PERFORMANCE` | `2` | وضع الأداء الأقصى (لـ setpolicy drivers) |

#### Relation Constants (لاختيار الـ frequency المناسبة)

| Constant | المعنى |
|----------|--------|
| `CPUFREQ_RELATION_L` | أقل frequency >= target ("lowest, no lower than") |
| `CPUFREQ_RELATION_H` | أعلى frequency <= target ("highest, no higher than") |
| `CPUFREQ_RELATION_C` | الأقرب للـ target |
| `CPUFREQ_RELATION_E` | flag إضافي: حاول تاخد efficient frequency |
| `CPUFREQ_RELATION_LE/HE/CE` | combinations من الأساسي + efficient |

#### Notifier Events

| Notifier Type | Events |
|---------------|--------|
| `CPUFREQ_TRANSITION_NOTIFIER` | `CPUFREQ_PRECHANGE`, `CPUFREQ_POSTCHANGE` |
| `CPUFREQ_POLICY_NOTIFIER` | `CPUFREQ_CREATE_POLICY`, `CPUFREQ_REMOVE_POLICY` |

#### ACPI Shared Type Constants

| Constant | Value | المعنى |
|----------|-------|--------|
| `CPUFREQ_SHARED_TYPE_NONE` | 0 | مفيش مشاركة |
| `CPUFREQ_SHARED_TYPE_HW` | 1 | الـ HW بيعمل الـ coordination لوحده |
| `CPUFREQ_SHARED_TYPE_ALL` | 2 | كل الـ CPUs المرتبطة لازم تعمل set للـ freq |
| `CPUFREQ_SHARED_TYPE_ANY` | 3 | أي CPU منهم تقدر تعمل set للـ freq |

#### Config Options المهمة

| Config | الأثر |
|--------|-------|
| `CONFIG_CPU_FREQ` | يفعّل الـ subsystem كله |
| `CONFIG_CPU_FREQ_STAT` | يضيف إحصائيات الـ transitions في sysfs |
| `CONFIG_CPU_FREQ_GOV_SCHEDUTIL` | يفعّل الـ schedutil governor |
| `CONFIG_CPU_THERMAL` | بيشتغل مع `CPUFREQ_IS_COOLING_DEV` |

---

### الـ Structs المهمة

#### 1. `struct cpufreq_driver`

**الغرض:** الـ interface الرئيسي اللي أي driver لازم يملأه ويسجّله مع الـ cpufreq core. ده "عقد" الـ driver مع الـ kernel.

```c
struct cpufreq_driver {
    char    name[CPUFREQ_NAME_LEN]; /* اسم الـ driver (max 16 chars) */
    u16     flags;                  /* مجموعة flags تتحكم في سلوك الـ core */
    void   *driver_data;            /* بيانات خاصة بالـ driver */

    /* إلزامي: initialization لكل policy */
    int    (*init)(struct cpufreq_policy *policy);
    /* إلزامي: validate قيم الـ policy قبل التطبيق */
    int    (*verify)(struct cpufreq_policy_data *policy);

    /* اختار واحد من الأربعة دول */
    int    (*setpolicy)(struct cpufreq_policy *policy);
    int    (*target)(struct cpufreq_policy *, unsigned int freq, unsigned int rel); /* deprecated */
    int    (*target_index)(struct cpufreq_policy *policy, unsigned int index);
    unsigned int (*fast_switch)(struct cpufreq_policy *policy, unsigned int target_freq);

    /* للتحول عبر intermediate frequency */
    unsigned int (*get_intermediate)(struct cpufreq_policy *policy, unsigned int index);
    int          (*target_intermediate)(struct cpufreq_policy *policy, unsigned int index);

    /* اختيارية */
    unsigned int (*get)(unsigned int cpu);           /* اقرأ الـ freq الحالية */
    int          (*bios_limit)(int cpu, unsigned int *limit);
    int          (*online)(struct cpufreq_policy *policy);
    int          (*offline)(struct cpufreq_policy *policy);
    void         (*exit)(struct cpufreq_policy *policy);
    int          (*suspend)(struct cpufreq_policy *policy);
    int          (*resume)(struct cpufreq_policy *policy);
    void         (*ready)(struct cpufreq_policy *policy);
    void         (*update_limits)(struct cpufreq_policy *policy);
    void         (*adjust_perf)(unsigned int cpu, unsigned long min_perf,
                                unsigned long target_perf, unsigned long capacity);
    void         (*register_em)(struct cpufreq_policy *policy); /* Energy Model */
    int          (*set_boost)(struct cpufreq_policy *policy, int state);

    struct freq_attr **attr;    /* قائمة sysfs attributes */
    bool    boost_enabled;      /* هل الـ boost frequencies مفعّلة؟ */
};
```

**العلاقات:**
- يُملأ بالـ driver ويُمرَّر لـ `cpufreq_register_driver()`
- الـ core يحتفظ بمؤشر عالمي للـ driver المسجّل
- يحتوي على pointers لـ `cpufreq_policy` في كل callback

---

#### 2. `struct cpufreq_policy`

**الغرض:** تمثّل سياسة الـ frequency لمجموعة CPUs بتشارك نفس الـ clock/voltage rail. هي الـ object المركزي اللي الـ core والـ driver والـ governor كلهم يشتغلوا عليه.

```c
struct cpufreq_policy {
    /* CPUs اللي بتتحكم فيها الـ policy دي */
    cpumask_var_t   cpus;          /* online CPUs فقط */
    cpumask_var_t   related_cpus;  /* online + offline */
    cpumask_var_t   real_cpus;     /* related + present */

    unsigned int    cpu;           /* الـ CPU اللي بيدير الـ policy */
    struct clk     *clk;           /* pointer للـ clock */

    struct cpufreq_cpuinfo cpuinfo; /* حدود الـ hardware */

    /* الحدود الحالية للـ policy */
    unsigned int    min;            /* kHz */
    unsigned int    max;            /* kHz */
    unsigned int    cur;            /* الـ frequency الحالية */
    unsigned int    suspend_freq;   /* freq وقت الـ suspend */

    unsigned int    policy;         /* POWERSAVE أو PERFORMANCE */
    struct cpufreq_governor *governor;
    void           *governor_data;

    struct cpufreq_frequency_table *freq_table;
    enum cpufreq_table_sorting freq_table_sorted;

    /* locking */
    struct rw_semaphore rwsem;
    spinlock_t      transition_lock;
    wait_queue_head_t transition_wait;

    bool    fast_switch_possible;
    bool    fast_switch_enabled;
    bool    transition_ongoing;

    /* QoS constraints */
    struct freq_constraints  constraints;
    struct freq_qos_request *min_freq_req;
    struct freq_qos_request *max_freq_req;

    /* sysfs */
    struct kobject  kobj;
    struct list_head policy_list;

    /* للـ driver */
    void           *driver_data;
    struct cpufreq_stats *stats;
    struct thermal_cooling_device *cdev;
};
```

**العلاقات:**
- يشير لـ `cpufreq_governor`
- يشير لـ `cpufreq_frequency_table` (array)
- يُنشأ بواسطة الـ core ويُمرَّر لكل callbacks الـ driver

---

#### 3. `struct cpufreq_cpuinfo`

**الغرض:** بيانات الـ hardware الثابتة للـ CPU — الحدود الفيزيائية للـ frequency.

```c
struct cpufreq_cpuinfo {
    unsigned int max_freq;           /* kHz */
    unsigned int min_freq;           /* kHz */
    unsigned int transition_latency; /* nanoseconds — وقت التحول بين frequencies */
};
```

**العلاقات:** embedded مباشرة داخل `cpufreq_policy.cpuinfo`. الـ driver يملأها في `->init()`.

---

#### 4. `struct cpufreq_policy_data`

**الغرض:** نسخة مبسّطة من الـ policy تُستخدم فقط في `->verify()` callback. الـ core يمررها بدل الـ policy الكاملة عشان يمنع الـ driver من تعديل حاجات مش المفروض يعدّلها.

```c
struct cpufreq_policy_data {
    struct cpufreq_cpuinfo          cpuinfo;
    struct cpufreq_frequency_table *freq_table;
    unsigned int    cpu;
    unsigned int    min;  /* kHz — الـ driver يقدر يعدّل ده */
    unsigned int    max;  /* kHz — الـ driver يقدر يعدّل ده */
};
```

**العلاقات:** يشير لـ `cpufreq_cpuinfo` وجدول الـ frequencies. يتحول منه بواسطة helper `cpufreq_verify_within_limits()`.

---

#### 5. `struct cpufreq_frequency_table`

**الغرض:** array بيحدد كل الـ frequencies المتاحة على الـ hardware. كل عنصر = operating point واحد.

```c
struct cpufreq_frequency_table {
    unsigned int flags;        /* CPUFREQ_BOOST_FREQ أو CPUFREQ_INEFFICIENT_FREQ */
    unsigned int driver_data;  /* بيانات خاصة للـ driver، الـ core مش بيستخدمها */
    unsigned int frequency;    /* kHz — أو CPUFREQ_ENTRY_INVALID أو CPUFREQ_TABLE_END */
};
```

**العلاقات:** مؤشر على الـ array ده موجود في `cpufreq_policy.freq_table`. الـ core بيتحقق منه أوتوماتيك.

---

#### 6. `struct cpufreq_freqs`

**الغرض:** بيانات عملية الـ transition نفسها، بتتبعت في الـ notifiers قبل وبعد التغيير.

```c
struct cpufreq_freqs {
    struct cpufreq_policy *policy;  /* الـ policy اللي بتتغير */
    unsigned int old;               /* الـ frequency القديمة (kHz) */
    unsigned int new;               /* الـ frequency الجديدة (kHz) */
    u8 flags;                       /* flags من الـ driver */
};
```

**العلاقات:** يُنشأ مؤقتاً في `cpufreq_freq_transition_begin()` وينتهي في `cpufreq_freq_transition_end()`.

---

#### 7. `struct cpufreq_governor`

**الغرض:** يمثّل governor معين (زي schedutil أو ondemand). الـ governor هو اللي يقرر أي frequency يختار ضمن حدود الـ policy.

```c
struct cpufreq_governor {
    char    name[CPUFREQ_NAME_LEN];
    int    (*init)(struct cpufreq_policy *policy);
    void   (*exit)(struct cpufreq_policy *policy);
    int    (*start)(struct cpufreq_policy *policy);
    void   (*stop)(struct cpufreq_policy *policy);
    void   (*limits)(struct cpufreq_policy *policy);
    ssize_t (*show_setspeed)(struct cpufreq_policy *policy, char *buf);
    int     (*store_setspeed)(struct cpufreq_policy *policy, unsigned int freq);
    struct list_head governor_list;
    struct module   *owner;
    u8      flags;  /* CPUFREQ_GOV_DYNAMIC_SWITCHING, CPUFREQ_GOV_STRICT_TARGET */
};
```

**العلاقات:** مؤشر عليه موجود في `cpufreq_policy.governor`. يتسجّل بـ `cpufreq_register_governor()`.

---

#### 8. `struct freq_attr`

**الغرض:** تعريف sysfs attribute خاصة بالـ driver، مثل عرض frequencies متاحة أو الـ current governor.

```c
struct freq_attr {
    struct attribute attr;
    ssize_t (*show)(struct cpufreq_policy *, char *);
    ssize_t (*store)(struct cpufreq_policy *, const char *, size_t count);
};
```

**العلاقات:** الـ driver يملأ `cpufreq_driver.attr` بـ NULL-terminated array من `freq_attr*`.

---

### رسم العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────────┐
│                    cpufreq_driver                               │
│  name, flags, driver_data                                       │
│  .init / .verify / .target_index / .fast_switch / .setpolicy   │
│  .get / .exit / .suspend / .resume / .ready                     │
│  .attr ──────────────────────────────► freq_attr[]              │
└───────────────────────┬─────────────────────────────────────────┘
                        │ creates & fills
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    cpufreq_policy                               │
│  cpu, min, max, cur, policy, flags                              │
│  cpuinfo ────────────────────────────► cpufreq_cpuinfo          │
│                                         (min_freq, max_freq,    │
│                                          transition_latency)    │
│  governor ───────────────────────────► cpufreq_governor         │
│                                         (init/start/stop/limits)│
│  freq_table ─────────────────────────► cpufreq_frequency_table[]│
│                                         [0] flags|data|freq     │
│                                         [1] flags|data|freq     │
│                                         [N] CPUFREQ_TABLE_END   │
│  constraints ────────────────────────► freq_constraints         │
│  min_freq_req, max_freq_req ─────────► freq_qos_request         │
│  cdev ───────────────────────────────► thermal_cooling_device   │
│  stats ──────────────────────────────► cpufreq_stats            │
│  kobj ───────────────────────────────► kobject (sysfs)          │
│  policy_list ◄──────────────────────── list of all policies     │
│  rwsem (rw_semaphore)                                           │
│  transition_lock (spinlock_t)                                   │
└─────────────────────────────────────────────────────────────────┘
                        │ passed to
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                 cpufreq_policy_data                             │
│  (read-only view used only in ->verify() callback)             │
│  cpuinfo, freq_table, cpu, min, max                            │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────┐       ┌──────────────────────────────────┐
│   cpufreq_freqs      │       │      gov_attr_set                │
│  policy* ────────────┼──────►│  (shared governor tunables)      │
│  old (kHz)           │       │  kobj, policy_list, update_lock  │
│  new (kHz)           │       └──────────────────────────────────┘
│  flags               │
└──────────────────────┘
```

---

### دورة حياة الـ Driver (Lifecycle Diagram)

```
 MODULE LOAD / __init
        │
        ▼
 فحص إن الـ CPU/chipset مناسب
        │
        ▼
 ملء struct cpufreq_driver
 (name, flags, callbacks)
        │
        ▼
 cpufreq_register_driver()
        │
        ▼
 ┌──────────────────────────────────────┐
 │          لكل CPU جديد               │
 │                                      │
 │  driver->init(policy)                │
 │    ├── ملء policy->cpuinfo           │
 │    ├── ملء policy->min / max / cur   │
 │    ├── ملء policy->cpus (cpumask)    │
 │    └── ربط policy->freq_table        │
 │                                      │
 │  driver->verify(policy_data)         │
 │    └── تصحيح min/max إذا لزم        │
 │                                      │
 │  governor->start(policy)             │
 │                                      │
 │  driver->ready(policy)   [optional]  │
 │                                      │
 │  ┌─── RUNNING ───────────────────┐   │
 │  │  governor يطلب تغيير freq     │   │
 │  │  driver->target_index(policy) │   │
 │  │  driver->fast_switch(policy)  │   │
 │  └───────────────────────────────┘   │
 │                                      │
 │  CPU HOTPLUG / SUSPEND:              │
 │    driver->suspend(policy)           │
 │    driver->resume(policy)            │
 │    driver->offline(policy)           │
 │    driver->online(policy)            │
 │                                      │
 │  CPU_POST_DEAD:                      │
 │    driver->exit(policy)              │
 └──────────────────────────────────────┘
        │
        ▼
 MODULE UNLOAD
        │
        ▼
 cpufreq_unregister_driver()
```

---

### Frequency Transition Flow — target_index مع intermediate

```
governor يقرر تغيير الـ freq
        │
        ▼
cpufreq_driver_target()         ← entry point من الـ governor
        │
        ▼
__cpufreq_driver_target()
        │
        ├── driver->get_intermediate(policy, index)
        │       │
        │       ├── returns 0  ──────────────────────────────────────┐
        │       │                                                     │
        │       └── returns intermediate_freq                         │
        │                   │                                         │
        │                   ▼                                         │
        │           cpufreq_freq_transition_begin()                   │
        │           [PRECHANGE notifier → intermediate_freq]          │
        │                   │                                         │
        │                   ▼                                         │
        │           driver->target_intermediate(policy, index)        │
        │           [hardware switches to intermediate]               │
        │                   │                                         │
        │                   ▼                                         │
        │           cpufreq_freq_transition_end()                     │
        │           [POSTCHANGE notifier → intermediate_freq]         │
        │                   │                                         │
        │                   ▼                                         │
        ├───────────────────┘                                         │
        │                                                             │
        ▼                                                             │
cpufreq_freq_transition_begin()  ◄───────────────────────────────────┘
[PRECHANGE notifier → target_freq]
        │
        ▼
driver->target_index(policy, index)
[hardware switches to freq_table[index].frequency]
        │
        ├── success ──────────────────────────────────┐
        │                                              │
        └── error: restore to policy->restore_freq    │
                                                       │
                                                       ▼
                                        cpufreq_freq_transition_end()
                                        [POSTCHANGE notifier]
```

---

### fast_switch Flow (Scheduler Context)

```
scheduler (IRQ context)
        │
        ▼
cpufreq_driver_fast_switch(policy, target_freq)
        │
        │  [NO sleeping allowed here]
        │  [NO notifiers sent]
        │
        ▼
driver->fast_switch(policy, target_freq)
        │
        ▼
يكتب مباشرة في register الـ hardware
        │
        ▼
يرجع الـ frequency الفعلية اللي اتحددت
```

---

### setpolicy Flow

```
user يكتب في sysfs (governor أو scaling_min/max_freq)
        │
        ▼
cpufreq_set_policy()
        │
        ▼
driver->verify(policy_data)
        │
        ▼
driver->setpolicy(policy)
        │
        ├── policy->policy == CPUFREQ_POLICY_PERFORMANCE
        │       └── اضبط hardware على أعلى freq ضمن [min, max]
        │
        └── policy->policy == CPUFREQ_POLICY_POWERSAVE
                └── اضبط hardware على أقل freq ضمن [min, max]
```

---

### Driver Registration Flow

```
driver code
        │
        ▼
cpufreq_register_driver(driver)
        │
        ├── يتحقق من إن الـ callbacks الإلزامية موجودة (init, verify)
        ├── يتحقق من إن في على الأقل واحد من:
        │     setpolicy / target / target_index / fast_switch
        │
        ├── يسجّل الـ driver عالمياً (cpufreq_driver global ptr)
        │
        └── لكل CPU موجود:
                │
                ▼
            cpufreq_add_dev()
                │
                ▼
            driver->init(policy)
                │
                ▼
            cpufreq_add_policy_cpu()
                │
                ▼
            governor->start(policy)
```

---

### استراتيجية الـ Locking

#### الـ Locks المستخدمة

| Lock | النوع | يحمي ماذا؟ |
|------|-------|------------|
| `policy->rwsem` | `rw_semaphore` | كل بيانات الـ `cpufreq_policy` struct |
| `policy->transition_lock` | `spinlock_t` | حالة الـ transition (`transition_ongoing`, `transition_task`) |
| `policy->transition_wait` | `wait_queue_head_t` | الـ threads اللي بتستنى انتهاء الـ transition |
| `cpufreq_driver_lock` | `rwlock_t` (global, في الـ core) | الـ global driver pointer |
| `gov_attr_set->update_lock` | `mutex` | بيانات الـ governor المشتركة بين policies |

#### قواعد الـ Locking

```
قراءة بيانات الـ policy:
    down_read(&policy->rwsem)
    ... قراءة آمنة ...
    up_read(&policy->rwsem)

كتابة / تعديل الـ policy (hotplug / policy change):
    down_write(&policy->rwsem)
    ... تعديل آمن ...
    up_write(&policy->rwsem)

أثناء الـ frequency transition:
    spin_lock(&policy->transition_lock)
    policy->transition_ongoing = true
    policy->transition_task = current
    spin_unlock(&policy->transition_lock)

    ... التحول الفعلي ...

    spin_lock(&policy->transition_lock)
    policy->transition_ongoing = false
    spin_unlock(&policy->transition_lock)
    wake_up(&policy->transition_wait)
```

#### ترتيب الـ Locks (Lock Ordering) — لتجنب الـ deadlock

```
1. cpufreq_driver_lock  (أعلى مستوى — global)
        │
        ▼
2. policy->rwsem        (per-policy semaphore)
        │
        ▼
3. policy->transition_lock  (spinlock — أسرع، لكن لفترة قصيرة جداً)
```

**قاعدة ذهبية:** مش مسموح تاخد `transition_lock` وانت شايل `rwsem` write lock في نفس الوقت لفترة طويلة — الـ spinlock للـ atomic operations بس.

#### الـ fast_switch والـ Locking

الـ `fast_switch` بيشتغل من **scheduler context (IRQ context)** — ممنوع منعاً باتاً استخدام أي semaphore أو mutex. الـ driver لازم يكتب على الـ hardware register مباشرة بدون أي lock يتسبب في sleep.

```
fast_switch context:
    ❌ mutex_lock()
    ❌ down_read/write() (semaphore)
    ✅ READ_ONCE() / WRITE_ONCE()
    ✅ atomic operations
    ✅ spinlock_t (إذا كان ضروري جداً)
```

#### الـ CPUFREQ_ASYNC_NOTIFICATION Flag والـ Locking

لو الـ driver بيرفع `CPUFREQ_ASYNC_NOTIFICATION`:

- الـ POSTCHANGE notification بتيجي من thread منفصل
- الـ core ما بيعملش wait للـ transition تكمل قبل ما يعمل notify
- الـ driver مسؤول عن synchronization الـ notifications الخاصة بيه

---

### Frequency Table Helpers — Cheatsheet

| Helper | الاستخدام |
|--------|-----------|
| `cpufreq_frequency_table_cpuinfo(policy)` | يملأ `policy->cpuinfo.min/max_freq` من الـ table |
| `cpufreq_frequency_table_verify(policy_data)` | يتحقق إن في freq واحدة على الأقل في [min, max] |
| `cpufreq_generic_frequency_table_verify()` | مثل السابق + يتحقق من الـ entries |
| `cpufreq_frequency_table_get_index(policy, freq)` | يجيب index الـ freq في الـ table |
| `cpufreq_verify_within_limits(policy, min, max)` | يعمل clamp للـ policy->min/max في [min, max] |
| `cpufreq_verify_within_cpu_limits(policy)` | نفس السابق بس بحدود الـ cpuinfo |

#### Iteration Macros

```c
/* كل الـ entries بما فيها INVALID */
cpufreq_for_each_entry(pos, table)

/* مع index */
cpufreq_for_each_entry_idx(pos, table, idx)

/* valid entries فقط (يتخطى CPUFREQ_ENTRY_INVALID) */
cpufreq_for_each_valid_entry(pos, table)

/* valid + مع index */
cpufreq_for_each_valid_entry_idx(pos, table, idx)

/* valid + efficient فقط (يتخطى CPUFREQ_INEFFICIENT_FREQ) */
cpufreq_for_each_efficient_entry_idx(pos, table, idx, efficiencies)
```

**مثال عملي:**

```c
struct cpufreq_frequency_table *pos;
struct cpufreq_frequency_table my_freqs[] = {
    { .frequency = 800000  },  /* 800 MHz */
    { .frequency = 1200000 },  /* 1.2 GHz */
    { .frequency = 1600000, .flags = CPUFREQ_BOOST_FREQ }, /* boost */
    { .frequency = CPUFREQ_TABLE_END },
};

/* iterate على الـ valid entries بس */
cpufreq_for_each_valid_entry(pos, my_freqs) {
    pr_info("Available freq: %u kHz\n", pos->frequency);
}
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Registration & Lifecycle

| Function / Callback | النوع | الغرض |
|---|---|---|
| `cpufreq_register_driver()` | Core API | تسجيل الـ driver مع الـ CPUFreq core |
| `cpufreq_unregister_driver()` | Core API | إلغاء تسجيل الـ driver |
| `cpufreq_driver.init()` | Driver callback | تهيئة الـ policy لأول مرة |
| `cpufreq_driver.exit()` | Driver callback | تنظيف الـ policy عند hotplug |
| `cpufreq_driver.ready()` | Driver callback | يُستدعى بعد اكتمال تهيئة الـ policy |
| `cpufreq_driver.online()` | Driver callback | CPU coming online |
| `cpufreq_driver.offline()` | Driver callback | CPU going offline |

#### Frequency Switching

| Function / Callback | النوع | الغرض |
|---|---|---|
| `cpufreq_driver.target_index()` | Driver callback | ضبط تردد من جدول الترددات (الأساسي) |
| `cpufreq_driver.target()` | Driver callback | ضبط تردد بقيمة مباشرة (deprecated) |
| `cpufreq_driver.fast_switch()` | Driver callback | تبديل سريع من سياق الـ scheduler |
| `cpufreq_driver.setpolicy()` | Driver callback | تفويض التحكم للـ hardware |
| `cpufreq_driver.adjust_perf()` | Driver callback | ضبط الأداء بـ performance levels |
| `cpufreq_driver.get_intermediate()` | Driver callback | إرجاع تردد وسيط مستقر |
| `cpufreq_driver.target_intermediate()` | Driver callback | ضبط الـ CPU للتردد الوسيط |

#### Verification & Policy Helpers

| Function | النوع | الغرض |
|---|---|---|
| `cpufreq_driver.verify()` | Driver callback | التحقق من صحة الـ policy الجديد |
| `cpufreq_verify_within_limits()` | Inline helper | تصحيح min/max داخل حدود معينة |
| `cpufreq_verify_within_cpu_limits()` | Inline helper | تصحيح min/max داخل حدود الـ CPU |

#### Frequency Table Helpers

| Function / Macro | النوع | الغرض |
|---|---|---|
| `cpufreq_frequency_table_cpuinfo()` | Helper | ملء cpuinfo.min/max من الجدول |
| `cpufreq_frequency_table_verify()` | Helper | التحقق من وجود تردد صالح في النطاق |
| `cpufreq_generic_frequency_table_verify()` | Helper | نسخة generic من الـ verify |
| `cpufreq_frequency_table_get_index()` | Helper | البحث عن index لتردد معين |
| `cpufreq_for_each_entry()` | Macro | iterator على كل entries الجدول |
| `cpufreq_for_each_valid_entry()` | Macro | iterator يتخطى INVALID entries |
| `cpufreq_for_each_entry_idx()` | Macro | iterator مع index |
| `cpufreq_for_each_valid_entry_idx()` | Macro | iterator مع index ويتخطى INVALID |
| `cpufreq_for_each_efficient_entry_idx()` | Macro | iterator يتخطى INVALID و INEFFICIENT |

#### Power Management

| Function | النوع | الغرض |
|---|---|---|
| `cpufreq_driver.suspend()` | Driver callback | الاستعداد للـ suspend |
| `cpufreq_driver.resume()` | Driver callback | الاستعادة بعد الـ resume |
| `cpufreq_driver.get()` | Driver callback | قراءة التردد الحالي من الـ hardware |
| `cpufreq_driver.bios_limit()` | Driver callback | قراءة قيود الـ BIOS/HW على التردد |
| `cpufreq_driver.set_boost()` | Driver callback | تفعيل/تعطيل boost frequencies |

#### Transition Notifications

| Function | النوع | الغرض |
|---|---|---|
| `cpufreq_freq_transition_begin()` | Core API | إرسال PRECHANGE notification |
| `cpufreq_freq_transition_end()` | Core API | إرسال POSTCHANGE notification |
| `cpufreq_driver_target()` | Core API | طلب تغيير تردد مع locking |
| `__cpufreq_driver_target()` | Core API | طلب تغيير تردد بدون policy lock |

---

### المجموعة الأولى: Registration & Driver Lifecycle

هذه المجموعة مسؤولة عن تسجيل الـ driver مع الـ CPUFreq core وإدارة دورة حياة الـ policy. تُستدعى مرة واحدة لكل policy (مش لكل CPU)، وهي نقطة الدخول الأساسية لأي driver جديد.

---

#### `cpufreq_register_driver()`

```c
int cpufreq_register_driver(struct cpufreq_driver *driver_data);
```

الـ driver بيستدعيها في `module_init()` (أو `__initcall` level 7) بعد ما يتأكد إن الـ kernel شغال على الـ hardware الصح. بتسجّل `struct cpufreq_driver` مع الـ CPUFreq core، وبعدها الـ core يبدأ يستدعي `.init()` على كل CPU موجود.

**Parameters:**
- `driver_data`: pointer لـ `struct cpufreq_driver` فيها كل الـ callbacks والـ metadata. لازم تفضل valid طول عمر الـ driver.

**Return:** `0` نجاح، أو error code سلبي. أشيع الأخطاء: `-EEXIST` لو driver تاني مسجّل، `-EINVAL` لو الـ struct ناقصة callbacks واجبة.

**Key details:**
- لازم `.init` و `.verify` موجودين دايمًا.
- لازم يكون فيه واحد على الأقل من: `.setpolicy`، `.target`، `.target_index`، `.fast_switch`.
- الـ core بيتقفل على `cpufreq_driver_lock` (rwlock) عند التسجيل.
- بعد التسجيل، الـ core بيعمل probe لكل CPU موجود بيستدعي `.init()`.

**Who calls it:** الـ driver نفسه من `module_init()` أو platform init function.

---

#### `cpufreq_unregister_driver()`

```c
void cpufreq_unregister_driver(struct cpufreq_driver *driver_data);
```

بتُزيل الـ driver من الـ core، وبتستدعي `.exit()` على كل policy نشطة قبل ما تُزيلها. مش بترجع error — الـ caller مسؤول إنه يتأكد من السلامة.

**Parameters:**
- `driver_data`: نفس الـ pointer اللي استخدمته في `cpufreq_register_driver()`.

**Return:** void.

**Key details:**
- بتحصل في `module_exit()`.
- الـ core بيوقف الـ governor الحالي لكل policy قبل الـ cleanup.
- بعدها الـ `.exit()` callback بيتستدعى على كل policy.

**Who calls it:** `module_exit()` أو driver cleanup path.

---

#### `cpufreq_driver.init()` callback

```c
int (*init)(struct cpufreq_policy *policy);
```

الـ callback الأهم في الـ driver. بتتستدعى مرة واحدة لكل policy جديدة (مش لكل CPU). الـ driver لازم يملا الـ policy struct بالمعلومات الأساسية.

**Parameters:**
- `policy`: الـ policy الجديدة المراد تهيئتها، فيها `policy->cpu` رقم الـ CPU الـ managing.

**الـ driver لازم يملا:**

| الحقل | الوصف |
|---|---|
| `policy->cpuinfo.min_freq` | أدنى تردد مدعوم بالـ kHz |
| `policy->cpuinfo.max_freq` | أقصى تردد مدعوم بالـ kHz |
| `policy->cpuinfo.transition_latency` | وقت التبديل بالـ nanoseconds |
| `policy->cur` | التردد الحالي عند التهيئة |
| `policy->min` و `policy->max` | النطاق الافتراضي للـ policy |
| `policy->cpus` | cpumask للـ CPUs المتشاركة في نفس clock/voltage domain |

**Return:** `0` نجاح، error code سلبي يمنع تسجيل الـ policy.

**Key details:**
- الـ `policy->cpus` بيحدد الـ CPUs اللي بتعمل DVFS سوا (يعني بيشتركوا في نفس voltage rail أو clock).
- ممكن تستخدم `cpufreq_frequency_table_cpuinfo()` لملء `cpuinfo.min/max` أوتوماتيكيًا من freq table.
- الـ context: يتستدعى من sleepable context (process context)، يعني ممكن تعمل فيه `kmalloc` و `request_firmware` وغيره.

**Pseudocode flow:**
```
init(policy):
    تحقق من الـ hardware
    احسب أو اقرأ min/max freq من الـ OPP table / device tree
    policy->cpuinfo.min_freq = min
    policy->cpuinfo.max_freq = max
    policy->cpuinfo.transition_latency = TRANSITION_LATENCY_NS
    policy->cur = read_current_freq_from_hw()
    policy->min = min
    policy->max = max
    cpumask_copy(policy->cpus, &shared_cpu_mask)
    policy->freq_table = driver_freq_table
    return 0
```

**Who calls it:** الـ CPUFreq core من `cpufreq_init_policy()` عند إضافة CPU جديد أو تسجيل driver جديد.

---

#### `cpufreq_driver.exit()` callback

```c
void (*exit)(struct cpufreq_policy *policy);
```

بتتستدعى في `CPU_POST_DEAD` phase من الـ CPU hotplug process، أو عند unregister الـ driver. مسؤولة عن تحرير الموارد اللي اتخصصت في `.init()`.

**Parameters:**
- `policy`: الـ policy المراد تنظيفها.

**Return:** void.

**Key details:**
- مقابل `.init()` بالظبط: أي resource اتخصص هناك يتحرر هنا.
- مش بتتستدعى لكل CPU بل مرة واحدة لكل policy.
- يُسمح بـ sleeping هنا.

---

#### `cpufreq_driver.ready()` callback

```c
void (*ready)(struct cpufreq_policy *policy);
```

بتتستدعى بعد ما الـ policy تكتمل تهيئتها بالكامل (بما في ذلك الـ governor). مفيدة لو الـ driver محتاج يعمل حاجة بعد ما كل حاجة اتجهزت.

**Parameters:**
- `policy`: الـ policy المكتملة.

**Return:** void.

**Who calls it:** الـ core من `cpufreq_online()` بعد ما الـ governor يبدأ.

---

#### `cpufreq_driver.online()` و `cpufreq_driver.offline()` callbacks

```c
int  (*online)(struct cpufreq_policy *policy);
void (*offline)(struct cpufreq_policy *policy);
```

**الـ `online()`** بتتستدعى لما CPU يرجع online لـ policy موجودة أصلًا (الـ policy مش محتاجة re-init كاملة). بديل خفيف عن إعادة تشغيل `.init()` بالكامل.

**الـ `offline()`** بتتستدعى قبل ما CPU يروح offline، بتدي الـ driver فرصة يتخلص من موارده الـ per-CPU.

**Return (`online`):** `0` نجاح، error code سلبي.

---

### المجموعة التانية: Frequency Switching Callbacks

دي قلب الـ driver — الـ callbacks المسؤولة عن ضبط التردد الفعلي في الـ hardware. الـ driver لازم يختار واحد أو أكتر من الاستراتيجيات دي حسب قدرات الـ hardware بتاعه.

---

#### `cpufreq_driver.target_index()` callback (الأساسي والمفضّل)

```c
int (*target_index)(struct cpufreq_policy *policy, unsigned int index);
```

الـ callback الأساسي والمفضّل لضبط التردد. الـ core بيحسب الـ index المناسب من الـ freq table وبيمرره للـ driver. الـ driver مش محتاج يتعامل مع الـ relation أو الـ target_freq.

**Parameters:**
- `policy`: الـ policy الحالية.
- `index`: الـ index في `policy->freq_table` للتردد المطلوب. التردد الفعلي هو `freq_table[index].frequency`.

**Return:** `0` نجاح. في حالة فشل، لازم الـ driver يرجع للتردد القديم (`policy->restore_freq`).

**Key details:**
- الـ core بيبعت PRECHANGE/POSTCHANGE notifications أوتوماتيكيًا.
- لو بيستخدم `get_intermediate/target_intermediate`، الـ core بيمشي transition آمن من خلال تردد وسيط.
- في حالة error، لازم يرجع على `policy->restore_freq` وليس على `policy->cur` لأن الـ core بيبعت notification على أساسه.
- بيتستدعى من process context (مش atomic) — يُسمح بـ sleeping.

**Pseudocode flow:**
```
target_index(policy, index):
    new_freq = policy->freq_table[index].frequency
    ret = hw_set_freq(new_freq)
    if ret != 0:
        hw_set_freq(policy->restore_freq)  /* rollback */
        return ret
    return 0
```

**Who calls it:** الـ CPUFreq core من `__cpufreq_driver_target()` بعد ما يحسب الـ index المناسب.

---

#### `cpufreq_driver.target()` callback (Deprecated)

```c
int (*target)(struct cpufreq_policy *policy,
              unsigned int target_freq,
              unsigned int relation);
```

الـ callback القديم — الـ core بيمرر التردد المطلوب مباشرةً والـ relation. الـ driver نفسه مسؤول عن إيجاد التردد المناسب. **deprecated** لصالح `target_index()`.

**Parameters:**
- `policy`: الـ policy الحالية.
- `target_freq`: التردد المستهدف بالـ kHz.
- `relation`: قاعدة الاختيار:
  - `CPUFREQ_REL_L` (= `CPUFREQ_RELATION_L`): اختار أدنى تردد >= target_freq.
  - `CPUFREQ_REL_H` (= `CPUFREQ_RELATION_H`): اختار أعلى تردد <= target_freq.
  - `CPUFREQ_RELATION_C`: اختار أقرب تردد للـ target.

**Constraint:** لازم `policy->min <= new_freq <= policy->max` دايمًا.

**Return:** `0` نجاح، error code سلبي.

**Who calls it:** Core من `__cpufreq_driver_target()` لو `target_index` مش موجود.

---

#### `cpufreq_driver.fast_switch()` callback

```c
unsigned int (*fast_switch)(struct cpufreq_policy *policy,
                            unsigned int target_freq);
```

الـ callback الأسرع — بيتستدعى من سياق الـ scheduler (IRQ-disabled أو RCU read-side critical section). مخصوص للـ governors اللي بتعمل frequency switching مباشرةً من الـ scheduler hotpath زي `schedutil`.

**Parameters:**
- `policy`: الـ policy الحالية.
- `target_freq`: التردد المستهدف بالـ kHz. الـ core بيكون عامل clamping مسبقًا بين `policy->min` و `policy->max`.

**Return:** التردد الجديد الفعلي اللي اتضبط بالـ kHz. لو الـ switching مش تم، يرجع 0.

**Key details:**
- **ممنوع** أي sleeping أو memory allocation أو lock اتكسب sleep-capable.
- الـ core مش بيبعت PRECHANGE/POSTCHANGE notifications من هنا — الـ driver مسؤول لو محتاجها.
- `policy->fast_switch_possible` لازم تكون `true` عشان الـ governor يفعّلها.
- `cpufreq_enable_fast_switch()` بتفعّل هذا المسار، و`cpufreq_disable_fast_switch()` بتوقفه.
- مش كل الـ drivers مطلوبة تنفّذها — الـ governors بترجع لـ `target_index()` لو مش متاحة.

**Who calls it:** الـ schedutil governor مباشرةً من `cpufreq_driver_fast_switch()` في الـ scheduler path.

---

#### `cpufreq_driver.setpolicy()` callback

```c
int (*setpolicy)(struct cpufreq_policy *policy);
```

بديل كامل عن `target_index/target/fast_switch` للـ hardware اللي بيتحكم في التردد بنفسه داخل نطاق محدد (hardware-autonomous DVFS). الـ driver بيحدد النطاق المسموح به فقط.

**Parameters:**
- `policy`: الـ policy اللي فيها:
  - `policy->min`: الحد الأدنى للتردد — يُضبط على الـ hardware.
  - `policy->max`: الحد الأقصى للتردد — يُضبط على الـ hardware.
  - `policy->policy`: الاستراتيجية:
    - `CPUFREQ_POLICY_PERFORMANCE`: يفضّل الأداء.
    - `CPUFREQ_POLICY_POWERSAVE`: يفضّل توفير الطاقة.

**Return:** `0` نجاح، error code سلبي.

**Key details:**
- الـ hardware نفسه بيختار التردد الفعلي داخل النطاق [min, max].
- مثال تطبيقي: `drivers/cpufreq/longrun.c` (Transmeta Crusoe).
- لا يمكن الجمع بين `setpolicy` وأي من `target*` أو `fast_switch`.

---

#### `cpufreq_driver.adjust_perf()` callback

```c
void (*adjust_perf)(unsigned int cpu,
                    unsigned long min_perf,
                    unsigned long target_perf,
                    unsigned long capacity);
```

استبدال متقدم لـ `fast_switch()` للـ drivers اللي بتستخدم performance levels (مش ترددات kHz) داخليًا. بتقدر تمرر hints إضافية للـ hardware.

**Parameters:**
- `cpu`: رقم الـ CPU.
- `min_perf`: الحد الأدنى المطلوب من الـ performance.
- `target_perf`: الـ performance المستهدف.
- `capacity`: الطاقة الاستيعابية الكاملة للـ CPU كـ reference.

**Key details:**
- لازم `fast_switch` يكون موجود برضو لأن الـ core ممكن يرجع له في بعض الحالات.
- بيُستخدم في الـ CPPC / AMDs و Intel ITMT drivers.

---

### المجموعة التالتة: Intermediate Frequency Callbacks

دي آلية للانتقال الآمن بين ترددات — مفيدة لما الانتقال المباشر بين ترددين بعيدين ممكن يسبب instability.

---

#### `cpufreq_driver.get_intermediate()` callback

```c
unsigned int (*get_intermediate)(struct cpufreq_policy *policy,
                                 unsigned int index);
```

بتحدد التردد الوسيط الآمن اللي المفروض الـ CPU يعدّي عليه قبل ما يوصل للتردد النهائي في `freq_table[index]`.

**Parameters:**
- `policy`: الـ policy الحالية.
- `index`: الـ index الـ target في الـ freq table.

**Return:** التردد الوسيط بالـ kHz. لو رجع `0`، الـ core بيتخطى الـ intermediate step ويستدعي `target_index()` مباشرةً.

**Key details:**
- متاح بس للـ drivers اللي بتستخدم `target_index()` ومش عندها `CPUFREQ_ASYNC_NOTIFICATION`.
- الـ core بيبعت notifications على التردد الوسيط أوتوماتيكيًا — الـ driver مش محتاج.

---

#### `cpufreq_driver.target_intermediate()` callback

```c
int (*target_intermediate)(struct cpufreq_policy *policy,
                           unsigned int index);
```

بتضبط الـ CPU على التردد الوسيط اللي رجعته `get_intermediate()`. بتتستدعى قبل `target_index()`.

**Parameters:**
- `policy`: الـ policy الحالية.
- `index`: نفس الـ index اللي اتبعت لـ `get_intermediate()`.

**Return:** `0` نجاح. في حالة فشل، الـ core لن يستدعي `target_index()`.

**Key details:**
- الـ intermediate state مش محتاج تبعت notifications — الـ core بيعملها.
- لو رجعت error، الـ core بيبعت POSTCHANGE notification للتردد القديم مباشرةً.

**الـ flow الكامل:**
```
Core يطلب تغيير لـ index X:

1. get_intermediate(policy, X) → intermediate_freq
   إذا intermediate_freq != 0:
2.   PRECHANGE notification (old → intermediate_freq)
3.   target_intermediate(policy, X)
4.   POSTCHANGE notification (old → intermediate_freq)
5. PRECHANGE notification (intermediate_freq → new_freq)
6. target_index(policy, X)
7. POSTCHANGE notification (intermediate_freq → new_freq)

إذا intermediate_freq == 0:
2. PRECHANGE notification (old → new_freq)
3. target_index(policy, X)
4. POSTCHANGE notification (old → new_freq)
```

---

### المجموعة الرابعة: Verification Callbacks & Helpers

الـ verify group مسؤول عن ضمان إن الـ policy اللي المستخدم طلبها منطقية وضمن حدود الـ hardware. ده جزء حرج لأنه بيتستدعى كل ما المستخدم يغيّر policy settings عبر sysfs.

---

#### `cpufreq_driver.verify()` callback

```c
int (*verify)(struct cpufreq_policy_data *policy);
```

بتتستدعى لما المستخدم يطلب policy جديدة عبر sysfs. الـ driver لازم يصحح الـ `policy->min` و `policy->max` عشان يكونوا ضمن الحدود المدعومة، وبيضمن وجود تردد صالح واحد على الأقل في النطاق.

**Parameters:**
- `policy`: `struct cpufreq_policy_data` (مش `cpufreq_policy` — نسخة مخففة للـ verification):
  - `.min` و `.max`: النطاق المطلوب — الـ driver ممكن يعدّلهم.
  - `.cpuinfo.min_freq` / `.cpuinfo.max_freq`: الحدود الفعلية للـ hardware.
  - `.freq_table`: pointer للـ freq table.

**Return:** `0` نجاح، error code سلبي لو مش ممكن يحقق أي policy صالحة.

**القاعدة الذهبية:**
```
إذا كان مش ممكن نطاق صالح:
    اول حاجة: زوّد policy->max
    لو ما نفعش: خفّض policy->min
    لازم دايمًا: policy->min <= freq_valid <= policy->max
```

**Key details:**
- الـ `struct cpufreq_policy_data` مش هي `struct cpufreq_policy` الكاملة — دي بالقصد عشان الـ verify callback ميعدّلش حاجة غير `min` و `max`.
- استخدام `cpufreq_verify_within_limits()` أو `cpufreq_frequency_table_verify()` بيسهّل الأمور جدًا.

---

#### `cpufreq_verify_within_limits()` — Inline Helper

```c
static inline void cpufreq_verify_within_limits(
    struct cpufreq_policy_data *policy,
    unsigned int min,
    unsigned int max);
```

بتعمل clamping لـ `policy->max` بين `[min, max]`، وبعدين clamping لـ `policy->min` بين `[min, policy->max]`. بتضمن إن النطاق صالح ومنطقي.

**Parameters:**
- `policy`: الـ policy المراد تصحيحها.
- `min`: الحد الأدنى المسموح به.
- `max`: الحد الأقصى المسموح به.

**Return:** void. بتعدّل `policy->min` و `policy->max` in-place.

**المنطق الداخلي:**
```c
policy->max = clamp(policy->max, min, max);
policy->min = clamp(policy->min, min, policy->max);
```

---

#### `cpufreq_verify_within_cpu_limits()` — Inline Helper

```c
static inline void cpufreq_verify_within_cpu_limits(
    struct cpufreq_policy_data *policy);
```

نسخة مختصرة من `cpufreq_verify_within_limits()` بتستخدم `policy->cpuinfo.min_freq` و `policy->cpuinfo.max_freq` كحدود تلقائيًا.

**Parameters:**
- `policy`: الـ policy المراد تصحيحها.

**Return:** void.

**Who calls it:** عادةً يتستدعى من داخل `.verify()` callback مباشرةً كأول سطر فيه.

---

### المجموعة الخامسة: Power Management Callbacks

---

#### `cpufreq_driver.get()` callback

```c
unsigned int (*get)(unsigned int cpu);
```

بتقرأ التردد الحالي مباشرةً من الـ hardware registers. مهمة للـ accuracy بعد الـ resume أو عند السؤال عبر sysfs.

**Parameters:**
- `cpu`: رقم الـ CPU المطلوب قراءة تردده.

**Return:** التردد الحالي بالـ kHz. يرجع `0` في حالة error.

**Key details:**
- بتتستدعى من `cpufreq_get()` و`cpufreq_quick_get()`.
- الـ `cpufreq_quick_get()` بتقرأ من `policy->cur` بدون استدعاء الـ driver (أسرع لكن ممكن تكون outdated).
- يُفضّل التنفيذ دايمًا لضمان دقة القراءة بعد الـ suspend/resume.

---

#### `cpufreq_driver.suspend()` callback

```c
int (*suspend)(struct cpufreq_policy *policy);
```

بتتستدعى قبل الـ system suspend، **بعد** ما الـ governor يتوقف وبـ **interrupts disabled**. الـ driver ممكن يحتاج يضبط التردد على قيمة معينة للـ suspend أو يحفظ state.

**Parameters:**
- `policy`: الـ policy الحالية.

**Return:** `0` نجاح، error code سلبي.

**Key details:**
- بيتستدعى مع interrupts disabled — ممنوع sleeping.
- `policy->suspend_freq` ممكن يكون محدد للتردد المطلوب عند الـ suspend.

---

#### `cpufreq_driver.resume()` callback

```c
int (*resume)(struct cpufreq_policy *policy);
```

بتتستدعى بعد الـ system resume، **قبل** ما الـ governor يبدأ وبـ **interrupts disabled**.

**Parameters:**
- `policy`: الـ policy الحالية.

**Return:** `0` نجاح، error code سلبي.

**Key details:**
- الـ context: interrupts disabled — ممنوع sleeping.
- بيُستخدم لاستعادة التردد أو إعادة تهيئة الـ hardware بعد الـ resume.

---

#### `cpufreq_driver.bios_limit()` callback

```c
int (*bios_limit)(int cpu, unsigned int *limit);
```

بتُرجع الحد الأقصى للتردد اللي حدده الـ BIOS أو الـ firmware. مفيدة في بيئات ACPI لو الـ BIOS فرض قيودًا على الـ OC أو الـ thermal.

**Parameters:**
- `cpu`: رقم الـ CPU.
- `limit`: pointer لـ unsigned int — بيتملا بالحد الأقصى بالـ kHz.

**Return:** `0` نجاح (و`*limit` فيها القيمة)، error code سلبي لو مفيش limit.

---

#### `cpufreq_driver.set_boost()` callback

```c
int (*set_boost)(struct cpufreq_policy *policy, int state);
```

بتفعّل أو بتعطّل الـ boost frequencies (زي Intel Turbo Boost أو AMD Precision Boost) على مستوى الـ policy.

**Parameters:**
- `policy`: الـ policy المستهدفة.
- `state`: `1` لتفعيل الـ boost، `0` لتعطيله.

**Return:** `0` نجاح، error code سلبي.

**Key details:**
- `driver->boost_enabled` flag بيتبعت مع الـ driver.
- ممكن يتستدعى من sysfs أو من الـ core عند تغيير الـ thermal limits.

---

### المجموعة السادسة: Frequency Table Helpers

معظم الـ drivers عندها عدد محدد من الترددات المدعومة. الـ `cpufreq_frequency_table` بتنظّم الترددات دي، والـ helpers بتسهّل التعامل معاها.

---

#### `struct cpufreq_frequency_table`

```c
struct cpufreq_frequency_table {
    unsigned int flags;       /* CPUFREQ_BOOST_FREQ, CPUFREQ_INEFFICIENT_FREQ */
    unsigned int driver_data; /* driver-specific opaque data */
    unsigned int frequency;   /* kHz, or special value */
};
```

كل entry بيمثّل تردد مدعوم. الـ table بتنتهي بـ entry فيها `frequency = CPUFREQ_TABLE_END`.

**قيم خاصة للـ frequency:**
| القيمة | الوصف |
|---|---|
| `CPUFREQ_TABLE_END` (`~1u`) | نهاية الجدول — لازم آخر entry |
| `CPUFREQ_ENTRY_INVALID` (`~0u`) | entry لازم يتخطاها الـ core |

**قيم الـ flags:**
| Flag | الوصف |
|---|---|
| `CPUFREQ_BOOST_FREQ` | تردد boost (فوق الـ normal max) |
| `CPUFREQ_INEFFICIENT_FREQ` | تردد موجود لكن غير كفؤ طاقيًا |

**ملاحظة:** الـ entries مش لازم تكون مرتبة، لكن لو كانت مرتبة (تصاعديًا أو تنازليًا) الـ core بيحدد `policy->freq_table_sorted` وبيعمل DVFS أسرع لأن الـ binary search أسرع من linear search.

---

#### `cpufreq_frequency_table_cpuinfo()`

```c
int cpufreq_frequency_table_cpuinfo(struct cpufreq_policy *policy);
```

بتملا `policy->cpuinfo.min_freq` و `policy->cpuinfo.max_freq` أوتوماتيكيًا بأدنى وأعلى تردد valid في `policy->freq_table`.

**Parameters:**
- `policy`: الـ policy اللي فيها `freq_table` pointer محدد مسبقًا.

**Return:** `0` نجاح، `-EINVAL` لو الجدول فاضي أو مفيش entries valid.

**Who calls it:** عادةً من `.init()` callback بعد تعيين `policy->freq_table`.

---

#### `cpufreq_frequency_table_verify()`

```c
int cpufreq_frequency_table_verify(struct cpufreq_policy_data *policy);
```

بتتحقق من إن في تردد valid واحد على الأقل ضمن نطاق `[policy->min, policy->max]`، وبتصحّح النطاق لو لزم.

**Parameters:**
- `policy`: الـ policy_data المراد التحقق منها (مش `cpufreq_policy` الكاملة).

**Return:** `0` إذا تم إيجاد تردد صالح، error code سلبي إذا فشل.

**Who calls it:** من `.verify()` callback بدل كتابة logic يدوي.

---

#### `cpufreq_generic_frequency_table_verify()`

```c
int cpufreq_generic_frequency_table_verify(struct cpufreq_policy_data *policy);
```

نسخة generic شاملة — بتستدعي `cpufreq_verify_within_cpu_limits()` أولًا، ثم `cpufreq_frequency_table_verify()`. كتير من الـ simple drivers بتستخدمها مباشرةً كـ `.verify` callback.

```c
/* مثال: استخدامها مباشرةً كـ callback */
static struct cpufreq_driver my_driver = {
    .verify = cpufreq_generic_frequency_table_verify,
    /* ... */
};
```

---

#### `cpufreq_frequency_table_get_index()`

```c
int cpufreq_frequency_table_get_index(struct cpufreq_policy *policy,
                                      unsigned int freq);
```

بتبحث في `policy->freq_table` عن entry بتردد `freq` وبترجع الـ index.

**Parameters:**
- `policy`: الـ policy الحالية.
- `freq`: التردد المطلوب البحث عنه بالـ kHz.

**Return:** الـ index إذا وُجد، `-EINVAL` إذا لم يُوجد.

---

### المجموعة السابعة: Iterator Macros

---

#### `cpufreq_for_each_entry(pos, table)`

```c
#define cpufreq_for_each_entry(pos, table) \
    for (pos = table; pos->frequency != CPUFREQ_TABLE_END; pos++)
```

Loop على كل entries بما فيها الـ INVALID. مفيد لو محتاج تعدّل كل entries بغض النظر عن صلاحيتها.

---

#### `cpufreq_for_each_valid_entry(pos, table)`

```c
#define cpufreq_for_each_valid_entry(pos, table) \
    for (pos = table; pos->frequency != CPUFREQ_TABLE_END; pos++) \
        if (pos->frequency == CPUFREQ_ENTRY_INVALID) \
            continue; \
        else
```

Loop على entries الصالحة فقط، بيتخطى الـ `CPUFREQ_ENTRY_INVALID`. الاستخدام الأشيع.

---

#### `cpufreq_for_each_entry_idx(pos, table, idx)`

```c
#define cpufreq_for_each_entry_idx(pos, table, idx) \
    for (pos = table, idx = 0; pos->frequency != CPUFREQ_TABLE_END; \
        pos++, idx++)
```

زي `cpufreq_for_each_entry` لكن بيوفّر الـ index أيضًا. ضروري لأن pointer subtraction غالي على بعض الـ architectures — ولازم تستخدم الـ macro دي بدل `(pos - table)`.

---

#### `cpufreq_for_each_valid_entry_idx(pos, table, idx)`

```c
#define cpufreq_for_each_valid_entry_idx(pos, table, idx) \
    cpufreq_for_each_entry_idx(pos, table, idx) \
        if (pos->frequency == CPUFREQ_ENTRY_INVALID) \
            continue; \
        else
```

الجمع بين Valid filtering و index tracking. الأكثر استخدامًا في logic إيجاد الـ best matching index.

---

#### `cpufreq_for_each_efficient_entry_idx(pos, table, idx, efficiencies)`

```c
#define cpufreq_for_each_efficient_entry_idx(pos, table, idx, efficiencies) \
    cpufreq_for_each_valid_entry_idx(pos, table, idx) \
        if (efficiencies && (pos->flags & CPUFREQ_INEFFICIENT_FREQ)) \
            continue; \
        else
```

الأكثر تخصصًا — بيتخطى الـ INVALID وكمان بيتخطى الـ `CPUFREQ_INEFFICIENT_FREQ` لو `efficiencies == true`. بيستخدمه الـ core في وظائف البحث عن أفضل تردد كفاءةً (`CPUFREQ_RELATION_E`).

**مثال كامل:**
```c
struct cpufreq_frequency_table *pos;
unsigned int idx;

/* طباعة كل الترددات الصالحة مع index */
cpufreq_for_each_valid_entry_idx(pos, policy->freq_table, idx) {
    pr_info("freq[%u] = %u kHz, flags = 0x%x\n",
            idx, pos->frequency, pos->flags);
}
```

---

### المجموعة الثامنة: Transition Notification Helpers (للـ drivers الـ async)

الـ drivers العادية مش محتاجة تتعامل مع دي لأن الـ core بيتكفل بيها. لكن الـ drivers اللي عندها `CPUFREQ_ASYNC_NOTIFICATION` flag (أي بتعمل POSTCHANGE من thread منفصل) محتاجة تستخدمها يدويًا.

---

#### `cpufreq_freq_transition_begin()`

```c
void cpufreq_freq_transition_begin(struct cpufreq_policy *policy,
                                   struct cpufreq_freqs *freqs);
```

بتبعت PRECHANGE notification لكل المهتمين (الـ governors، الـ stats، الـ cpufreq notifiers chain). بتضبط `transition_ongoing = true` وبتاخد `transition_lock`.

**Parameters:**
- `policy`: الـ policy.
- `freqs`: struct فيها `.old` (التردد الحالي) و`.new` (التردد المستهدف) بالـ kHz.

---

#### `cpufreq_freq_transition_end()`

```c
void cpufreq_freq_transition_end(struct cpufreq_policy *policy,
                                 struct cpufreq_freqs *freqs,
                                 int transition_failed);
```

بتبعت POSTCHANGE notification وبتحدّث `policy->cur` بالتردد الجديد. بتوقف الـ transition وبتحرر الـ `transition_lock`.

**Parameters:**
- `policy`: الـ policy.
- `freqs`: نفس الـ struct اللي اتبعت لـ `begin()`.
- `transition_failed`: `0` نجاح، أي قيمة تانية الـ core بيرجع لـ `.old`.

---

### ملاحظات ختامية على الـ Driver Flags

| Flag | الوصف |
|---|---|
| `CPUFREQ_NEED_UPDATE_LIMITS` | الـ core يستدعي الـ driver حتى لو التردد ما اتغيرش (لو الـ limits اتغيرت) |
| `CPUFREQ_CONST_LOOPS` | `loops_per_jiffy` مش بيتأثر بتغيير التردد |
| `CPUFREQ_IS_COOLING_DEV` | اشترك أوتوماتيكيًا كـ thermal cooling device |
| `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` | كل policy ممكن يكون عندها governor مختلف (multi-cluster SoCs) |
| `CPUFREQ_ASYNC_NOTIFICATION` | الـ driver بيبعت POSTCHANGE بنفسه من خارج الـ target() callback |
| `CPUFREQ_NEED_INITIAL_FREQ_CHECK` | الـ core يتحقق إن تردد الـ boot موجود في الجدول ويصحّحه لو لا |
| `CPUFREQ_NO_AUTO_DYNAMIC_SWITCHING` | يمنع استخدام governors زي `ondemand` الـ dynamic switching |
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. مدخلات الـ debugfs المهمة

**الـ** CPUFreq subsystem بيكتب معلومات تشخيصية في `/sys/kernel/debug/` — لازم يكون `CONFIG_DEBUG_FS=y`.

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف كل entries المتعلقة بالـ cpufreq
ls /sys/kernel/debug/cpufreq/

# اقرأ الـ frequency transitions المسجلة
cat /sys/kernel/debug/cpufreq/policy0/stats/trans_table

# شوف الـ pm_qos constraints على الـ frequency
cat /sys/kernel/debug/pm_qos/cpu_dma_latency
```

| Entry | المحتوى | كيف تقراه |
|---|---|---|
| `/sys/kernel/debug/cpufreq/` | directory لكل policy | `ls` ثم `cat` الملفات الجوانية |
| `trans_table` | matrix بيوضح الانتقالات بين الترددات | كل صف = من تردد، كل عمود = إلى تردد |
| `/sys/kernel/debug/clk/` | شجرة الـ clocks (لو الـ driver بيستخدم CCF) | `cat /sys/kernel/debug/clk/clk_summary` |

---

#### 2. مدخلات الـ sysfs المهمة

**الـ** sysfs interface هو الأساس لتشخيص الـ cpufreq driver — كل policy ليها directory تحت `/sys/devices/system/cpu/`.

```bash
# شوف الـ policy المرتبطة بـ CPU0
ls /sys/devices/system/cpu/cpu0/cpufreq/

# الـ driver المستخدم حالياً
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver

# التردد الحالي (بالـ kHz)
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# الـ freq table اللي الـ driver سجلها
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies

# الحد الأدنى والأقصى اللي الـ hardware بيسمح بيه
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_min_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq

# الـ transition latency (nanoseconds)
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_transition_latency

# الـ governor الحالي
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# الـ governors المتاحة
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors

# إحصائيات الوقت في كل تردد
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state

# لو الـ driver بيدعم boost
cat /sys/devices/system/cpu/cpu0/cpufreq/boost
```

| Sysfs File | المعنى | علامة المشكلة |
|---|---|---|
| `scaling_driver` | اسم الـ driver المسجّل | لو فاضي = الـ driver مش loaded |
| `scaling_cur_freq` | التردد اللي الـ kernel شايفه | اختلافه عن `cpuinfo_cur_freq` = مشكلة sync |
| `cpuinfo_cur_freq` | التردد الفعلي من الـ hardware | لو 0 = الـ `.get()` callback فاشل |
| `scaling_available_frequencies` | الـ freq_table اللي الـ driver سجلها | لو فاضية = `freq_table` مش متسجلة صح |
| `cpuinfo_transition_latency` | وقت الانتقال | `CPUFREQ_ETERNAL` = الـ driver مش بيحدده |

---

#### 3. استخدام الـ ftrace

**الـ** ftrace بيساعد تتتبع مسار تنفيذ الـ frequency transitions خطوة بخطوة.

```bash
# تفعيل الـ tracing
mount -t tracefs none /sys/kernel/tracing

# شوف الـ events المتاحة للـ cpufreq
ls /sys/kernel/tracing/events/power/ | grep cpu

# تفعيل الـ events الأساسية
echo 1 > /sys/kernel/tracing/events/power/cpu_frequency/enable
echo 1 > /sys/kernel/tracing/events/power/cpu_frequency_limits/enable

# تفعيل الـ function tracing لـ functions معينة في الـ driver
echo cpufreq_driver_target > /sys/kernel/tracing/set_ftrace_filter
echo cpufreq_freq_transition_begin >> /sys/kernel/tracing/set_ftrace_filter
echo cpufreq_freq_transition_end >> /sys/kernel/tracing/set_ftrace_filter

echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل حِمل لتحريك الـ frequency
stress-ng --cpu 4 --timeout 5s

echo 0 > /sys/kernel/tracing/tracing_on

# اقرأ النتائج
cat /sys/kernel/tracing/trace
```

**مثال على output متوقع:**

```
# TASK-PID    CPU#  IRQS-OFF  TIMESTAMP  FUNCTION
kworker/0:2-123  [000]  ....  1234.567890: cpu_frequency: state=1800000 cpu_id=0
kworker/0:2-123  [000]  ....  1234.567910: cpu_frequency: state=2400000 cpu_id=0
```

**الـ** `state` = التردد الجديد بالـ kHz — لو مش بيتغير يبقى الـ `target_index` أو `target` callback مش شغال.

---

#### 4. تفعيل الـ printk و Dynamic Debug

```bash
# تفعيل dynamic debug لكل الـ cpufreq subsystem
echo "module cpufreq +p" > /sys/kernel/debug/dynamic_debug/control

# أو لـ file معين داخل الـ kernel
echo "file drivers/cpufreq/cpufreq.c +p" > /sys/kernel/debug/dynamic_debug/control

# لو الـ driver في module منفصل (مثلاً acpi-cpufreq)
echo "module acpi_cpufreq +pflmt" > /sys/kernel/debug/dynamic_debug/control
# p = print, f = function name, l = line, m = module, t = thread ID

# شوف الـ messages في dmesg
dmesg -w | grep -i cpufreq

# لو محتاج verbose من الـ early boot
# أضف في kernel cmdline:
# dyndbg="module cpufreq +p"
```

لو بتكتب driver جديد، استخدم:

```c
/* بدل pr_debug عادي، استخدم dev_dbg مع device pointer */
dev_dbg(policy->cpu_dev, "target_index called: index=%u freq=%u kHz\n",
        index, freq_table[index].frequency);

/* أو pr_debug لو مفيش device */
pr_debug("cpufreq: [%s] transitioning cpu%u to %u kHz\n",
         cpufreq_driver->name, policy->cpu, target_freq);
```

---

#### 5. الـ Kernel Config Options للـ Debugging

```bash
# شوف الـ options المفعّلة في الكيرنل الحالي
grep -E "CONFIG_CPU_FREQ|CONFIG_DEBUG" /boot/config-$(uname -r) | grep -v "^#"
```

| Config Option | الوظيفة | متى تستخدمه |
|---|---|---|
| `CONFIG_CPU_FREQ_DEBUG` | يفعّل debug messages أساسية في الـ cpufreq core | أول خطوة في الـ debugging |
| `CONFIG_CPU_FREQ_STAT` | يفعّل إحصائيات `time_in_state` و`trans_table` | لتحليل سلوك الـ governor |
| `CONFIG_DEBUG_FS` | يفعّل الـ debugfs بالكامل | ضروري لمعظم أدوات الـ debugging |
| `CONFIG_FTRACE` | يفعّل الـ function tracing | لتتبع مسار الكود |
| `CONFIG_TRACEPOINTS` | يفعّل الـ tracepoints | للـ `cpu_frequency` events |
| `CONFIG_LOCKDEP` | يكتشف الـ deadlocks في الـ rwsem/spinlock | لو في kernel panic أو hang أثناء transition |
| `CONFIG_PROVE_LOCKING` | يتحقق من ترتيب الـ locks | مفيد لاكتشاف races في الـ driver |
| `CONFIG_CPU_THERMAL` | يربط الـ cpufreq بالـ thermal framework | للـ thermal throttling debugging |
| `CONFIG_PM_DEBUG` | يفعّل debug الـ suspend/resume | لمشاكل الـ suspend/resume في الـ driver |
| `CONFIG_DEBUG_OBJECTS` | يتتبع lifetime الـ objects | لاكتشاف use-after-free في الـ policy |

---

#### 6. أدوات الـ cpupower

**الـ** `cpupower` هو الأداة الرسمية للتعامل مع CPUFreq من الـ userspace.

```bash
# تثبيت الأداة
apt install linux-tools-$(uname -r)   # Debian/Ubuntu
dnf install kernel-tools              # Fedora/RHEL

# شوف معلومات الـ frequency لكل الـ CPUs
cpupower frequency-info

# مثال على output:
# analyzing CPU 0:
#   driver: acpi-cpufreq
#   CPUs which run at the same hardware frequency: 0
#   CPUs which need to have their frequency coordinated by software: 0
#   maximum transition latency:  0.97 ms
#   hardware limits: 400 MHz - 3.60 GHz
#   available frequency steps:  3.60 GHz, 2.80 GHz, ...
#   available cpufreq governors: conservative ondemand userspace powersave performance schedutil
#   current policy: freq should be within 400 MHz and 3.60 GHz.
#   current CPU frequency: 1.60 GHz (asserted by call to hardware)

# تغيير الـ governor للاختبار
cpupower frequency-set -g userspace

# تحديد تردد ثابت للاختبار
cpupower frequency-set -f 1800MHz

# شوف idle states (مرتبط بالـ cpufreq)
cpupower idle-info

# مراقبة الـ frequency في real-time
watch -n 0.5 "cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq"

# استخدام turbostat للمراقبة الدقيقة
turbostat --show CPU,Avg_MHz,Busy%,Bzy_MHz,TSC_MHz --interval 1
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `cpufreq: __cpufreq_add_dev: ->get() failed` | الـ `.get()` callback رجع error أو 0 | تحقق من إن الـ hardware accessible وإن الـ register read صح |
| `cpufreq: Failed to register driver` | `cpufreq_register_driver()` فشل | الـ driver ده مسجّل قبل كده، أو مفيش memory — شوف الـ return code |
| `cpufreq: __target_index: Failed to change cpu frequency` | `target_index()` رجع error | تحقق من الـ hardware communication (SPI/I2C/regulator) ومن صلاحيات الـ PMIC |
| `cpufreq: CPU0: cpu_exit() failed` | الـ `.exit()` callback فيه مشكلة | راجع الـ cleanup code، ممكن double-free |
| `cpufreq: Inaccessible CPU frequencies` | الـ cpuinfo.min/max_freq = 0 أو مش متحددين | في `.init()` callback، بالتأكيد الـ freq table مش متحطتش صح |
| `cpufreq: BIOS frequency change requested target` | الـ BIOS يتحكم في الـ frequency | disable P-state control from BIOS أو استخدم `intel_pstate=disable` |
| `cpufreq: __cpufreq_driver_target: cpu(0) is not present` | الـ CPU offline أثناء transition | راجع الـ hotplug handling في الـ driver |
| `BUG_ON()` في `cpufreq_frequency_table_verify` | الـ freq_table محتوية على entries غلط | تأكد من إن آخر entry فيها `CPUFREQ_TABLE_END` |
| `cpufreq: governor failed to transition` | الـ governor مش قادر يبدأ | ممكن conflict بين `fast_switch_possible` والـ governor |
| `cpufreq: transition timeout!` | الـ `transition_ongoing` مش اتـclear | الـ `cpufreq_freq_transition_end()` مش اتنادى — لازم يتنادى دايماً حتى لو فيه error |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في .init() — تحقق من صحة الـ freq_table */
int my_driver_init(struct cpufreq_policy *policy)
{
    /* تحقق من إن الـ table مش فاضية */
    WARN_ON(!policy->freq_table);

    if (!policy->cpuinfo.max_freq) {
        pr_err("cpufreq: [%s] max_freq not set!\n", policy->driver_data);
        dump_stack(); /* اعرف مين استدعى init بهذا الحال */
        return -EINVAL;
    }
    return 0;
}

/* في .target_index() — قبل الكتابة على الـ hardware */
int my_driver_target_index(struct cpufreq_policy *policy, unsigned int index)
{
    /* تحقق من صحة الـ index */
    WARN_ON(index >= num_freq_entries);

    /* تحقق من إن الـ transition مش recursive */
    WARN_ON(policy->transition_ongoing &&
            policy->transition_task == current);

    /* ... باقي الكود */
}

/* في .verify() — تحقق من الـ policy limits */
int my_driver_verify(struct cpufreq_policy_data *policy)
{
    WARN_ON(policy->min > policy->max);
    cpufreq_verify_within_cpu_limits(policy);
    return 0;
}

/* في .exit() — تحقق من عدم وجود leak */
void my_driver_exit(struct cpufreq_policy *policy)
{
    struct my_driver_data *data = policy->driver_data;
    WARN_ON(!data); /* لو NULL يبقى init فيه bug */
    kfree(data);
    policy->driver_data = NULL; /* امنع use-after-free */
}
```

**أماكن لازم دايماً تحط فيها WARN_ON:**
- بعد `cpufreq_freq_transition_begin()` مباشرة لو الـ hardware write فشل
- في الـ `fast_switch()` لو التردد المطلوب خارج الـ policy limits
- في `get_intermediate()` لو الـ intermediate freq أعلى من `cpuinfo.max_freq`

---

### Hardware Level

---

#### 1. التحقق من إن الـ Hardware State متطابق مع الـ Kernel State

```bash
# قارن التردد اللي الـ kernel شايفه مع التردد الفعلي
# scaling_cur_freq = ما الكيرنل شايفه (من .get() callback)
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# cpuinfo_cur_freq = التردد الفعلي من الـ hardware (لو الـ driver بيدعمه)
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq

# استخدم turbostat للقراءة من الـ MSR registers مباشرة (x86)
turbostat --show CPU,Bzy_MHz,TSC_MHz --interval 1

# على ARM، استخدم perf لحساب التردد الفعلي
perf stat -e cycles,task-clock -a -- sleep 1
# Bzy_MHz = cycles / task-clock = التردد الفعلي
```

**لو `scaling_cur_freq` ≠ `cpuinfo_cur_freq`:**
- الـ `.get()` callback بيقرأ register غلط
- الـ hardware مش بيستجاب للـ frequency change command

---

#### 2. تقنيات الـ Register Dump

```bash
# على x86 — اقرأ الـ MSR المسؤول عن الـ P-state (IA32_PERF_CTL)
# MSR 0x199 = IA32_PERF_CTL (كتابة = request freq)
# MSR 0x198 = IA32_PERF_STATUS (قراءة = current freq)
modprobe msr
rdmsr -p 0 0x198   # اقرأ current P-state لـ CPU0
rdmsr -p 0 0x199   # اقرأ requested P-state

# على أي معمارية — استخدم devmem2 للوصول للـ MMIO registers
# (مثلاً لو الـ frequency controller على عنوان MMIO معروف)
devmem2 0xFE000000 w   # اقرأ 32-bit من العنوان ده

# أو عبر /dev/mem (لو CONFIG_STRICT_DEVMEM=n)
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    f.seek(0xFE000000)
    val = struct.unpack('<I', f.read(4))[0]
    print(f'Register value: 0x{val:08X}')
"

# على ARM/ARM64 — استخدم io (من package busybox أو devmem2)
io -4 -r 0x11800000   # قراءة 32-bit register

# مثال عملي: قراءة DVFS register على Rockchip
devmem2 0xFF760000 w   # CRU (Clock Reset Unit) base address
```

---

#### 3. نصائح الـ Logic Analyzer و Oscilloscope

**لو الـ CPUFreq بيتحكم في جهد الـ CPU عبر PMIC (Power Management IC):**

```
Oscilloscope Setup:
┌─────────────────────────────────────────────────────┐
│  Channel 1: CPU VDD rail (probe at PMIC output)     │
│  Channel 2: I2C SDA (PMIC control interface)        │
│  Channel 3: I2C SCL                                 │
│  Trigger: Rising edge on Ch2 (I2C transaction)      │
│                                                     │
│  ما تشوفه عند frequency change:                    │
│  1. I2C transaction لتغيير voltage (Ch2/Ch3)       │
│  2. Voltage يتغير على Ch1 (بعد ~1-5ms)             │
│  3. بعدين الـ frequency تتغير                       │
└─────────────────────────────────────────────────────┘
```

**نقاط القياس المهمة:**
- **VDD_CPU rail**: تأكد إن الجهد بيوصل للقيمة المطلوبة قبل رفع التردد
- **Clock output**: قس الـ frequency مباشرة على الـ clock line بـ frequency counter
- **PMIC SDA/SCL**: تأكد من صحة الـ I2C protocol

**Logic Analyzer tips:**
```
- فعّل I2C decoder لقراءة الـ PMIC commands مباشرة
- ابحث عن الـ slave address للـ PMIC في datasheet
- كل voltage step لازم يسبق frequency step بوقت = transition_latency
- لو الـ voltage والـ frequency بيتغيروا في نفس الوقت = خطر instability
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | نمط الـ Kernel Log | السبب المحتمل |
|---|---|---|
| الـ PMIC مش بيرد | `i2c i2c-0: Transaction timeout` | I2C bus مش متحضّر قبل cpufreq init |
| الـ voltage مش بيتغير | `regulator: Failed to set voltage` | الـ regulator مش مفعّل في الـ DT أو الـ PMIC firmware غلط |
| تردد غير مستقر | `cpu0: Core temperature above threshold` | الـ voltage مش كافي لهذا التردد — راجع OPP table |
| الكيرنل يعمل hang عند frequency change | timeout ثم watchdog reset | الـ target_index() بلوكت في hardware access |
| الـ frequency مش بتتغير خالص | لا messages | الـ governor مش مفعّل أو الـ setpolicy() مش شغال |
| instability عند أعلى OPP | random crashes, `kernel BUG at ...` | الـ voltage أقل من minimum لهذا الـ OPP |
| الـ boost مش شغال | `cpufreq: Boost is not supported` | الـ `.set_boost` callback مش متعرّف في الـ driver |

---

#### 5. تشخيص الـ Device Tree

```bash
# تحقق من الـ DT المحمّل فعلاً (compiled DT في الذاكرة)
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "cpu@0"

# أو استخدم fdtdump على الـ DTB مباشرة
fdtdump /boot/dtb-$(uname -r)/platform.dtb | grep -A 30 "cpufreq"

# تحقق من الـ OPP table المرتبطة بالـ CPU
cat /proc/device-tree/cpus/cpu@0/operating-points-v2 | xxd | head
# القيمة دي هي phandle للـ OPP table

# شوف الـ OPP table عبر sysfs
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
# المفروض تتطابق مع الـ opp-hz values في الـ DT

# تحقق من الـ clock المرتبط بالـ CPU في الـ DT
cat /proc/device-tree/cpus/cpu@0/clocks | xxd

# تحقق من الـ regulator
cat /proc/device-tree/cpus/cpu@0/cpu-supply | xxd
```

**الأشياء اللي لازم تتطابق بين الـ DT والـ Hardware:**

```
DT: cpu@0 {
    operating-points-v2 = <&cpu_opp_table>;   ← OPP table phandle
    clocks = <&ccu CLK_CPU>;                  ← clock source
    clock-names = "cpu";
    cpu-supply = <&vdd_cpu>;                  ← voltage regulator
};

cpu_opp_table: opp-table {
    opp@1008000000 {
        opp-hz = /bits/ 64 <1008000000>;      ← بالـ Hz (مش kHz!)
        opp-microvolt = <1100000>;             ← الجهد بالـ µV
    };
};
```

**لو `scaling_available_frequencies` فاضية** رغم وجود OPP table في الـ DT:
```bash
# تحقق من إن الـ OPP library قرأت الـ DT صح
dmesg | grep -i opp
# لو فيه "no OPPs found" يبقى مشكلة في الـ phandle أو الـ compatible string
```

---

### Practical Commands

---

#### سيناريو 1: الـ Driver مش بيتسجّل

```bash
# شوف لو الـ module محمّل
lsmod | grep cpufreq
# أو لو built-in
cat /proc/modules | grep cpufreq

# شوف دمسج بعد load الـ module
dmesg | tail -50 | grep -iE "cpufreq|error|fail"

# جرب تحمّل الـ module يدوياً مع verbose
modprobe -v my_cpufreq_driver

# لو الـ driver registered بنجاح هتلاقي:
# [    2.345678] cpufreq: my_driver: registered successfully
# لو فشل:
# [    2.345678] cpufreq: Driver my_driver already registered
```

---

#### سيناريو 2: التردد مش بيتغير

```bash
# خطوة 1: تحقق من الـ governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# لو "performance" يبقى هو اللي بيمنع التغيير عمداً

# خطوة 2: حط الـ governor على userspace وغيّر يدوي
echo userspace > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo 1200000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed

# خطوة 3: تحقق هل اتغير
sleep 0.1
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# خطوة 4: لو مش اتغير، فعّل ftrace وشوف target_index اتنادى؟
echo 1 > /sys/kernel/tracing/events/power/cpu_frequency/enable
echo 1200000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed
cat /sys/kernel/tracing/trace | grep cpu_frequency
```

---

#### سيناريو 3: تشخيص الـ transition_latency

```bash
# شوف الـ latency اللي الـ driver بيعلن عنها
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_transition_latency
# القيمة بالـ nanoseconds — لو 0 أو CPUFREQ_ETERNAL (0xFFFFFFFF) = مش محدد

# قِس الـ latency الفعلية بـ ftrace
echo function_graph > /sys/kernel/tracing/current_tracer
echo my_driver_target_index > /sys/kernel/tracing/set_graph_function
echo 1 > /sys/kernel/tracing/tracing_on
echo 1200000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace
# هتلاقي timing لكل function call — دي الـ actual latency
```

**مثال على output:**

```
 0)               |  my_driver_target_index() {
 0)   0.125 µs    |    regmap_read();
 0)   0.987 µs    |    regmap_write();      /* voltage change */
 0) + 1.234 ms    |    udelay(1000);        /* wait for voltage settle */
 0)   0.456 µs    |    clk_set_rate();      /* actual freq change */
 0) + 1.236 ms    |  } /* total latency */
```

---

#### سيناريو 4: تشخيص مشاكل الـ Suspend/Resume

```bash
# فعّل debug الـ PM
echo 1 > /sys/power/pm_debug_messages

# شوف الـ frequency قبل وبعد الـ suspend
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
echo mem > /sys/power/state    # suspend
# ... بعد الاستيقاظ ...
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
dmesg | grep -iE "cpufreq|suspend|resume" | tail -20

# تحقق من الـ suspend_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
```

---

#### سيناريو 5: تشخيص مشاكل الـ Multicore / Shared Policy

```bash
# شوف إيه الـ CPUs في نفس الـ policy
cat /sys/devices/system/cpu/cpu0/cpufreq/related_cpus
cat /sys/devices/system/cpu/cpu0/cpufreq/affected_cpus

# مثال على output لـ quad-core مع policy مشتركة:
# related_cpus: 0 1 2 3

# تحقق من إن كل الـ CPUs في نفس الـ cluster بيغيروا معاً
for cpu in 0 1 2 3; do
    echo -n "CPU$cpu: "
    cat /sys/devices/system/cpu/cpu$cpu/cpufreq/scaling_cur_freq
done
# المفروض كل القيم متساوية لو shared policy

# لو مختلفة، تحقق من الـ policy->cpus mask في الـ driver:
# في .init():
# cpumask_copy(policy->cpus, topology_sibling_cpumask(policy->cpu));
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: RK3562 — Industrial Gateway — مشكلة `cpuinfo.transition_latency` غلط

#### العنوان
**الـ gateway الصناعي بيتأخر في استجابة الـ I2C بسبب latency مش صح في الـ cpufreq driver**

#### السياق
شركة بتبني industrial gateway على RK3562 بيشتغل بـ Debian embedded. الجهاز بيتكلم مع sensors عن طريق I2C بـ real-time tight timing (100µs window). بعد ما فريق الـ BSP كتب custom cpufreq driver للـ RK3562، لاحظوا إن الـ I2C transactions بتفشل بصورة عشوائية تحت load عالي.

#### المشكلة
الـ governor (schedutil) بيغير الـ frequency في وسط transactions الحساسة. التحقيق أظهر إن الـ `cpuinfo.transition_latency` اتضبطت بقيمة `CPUFREQ_ETERNAL` (يعني 0xFFFFFFFF) بالغلط بدل الـ latency الحقيقية اللي هي 300µs. ده خلى الـ governor يعتقد إن الـ frequency switching "مجاني" فيعملها في أي وقت.

#### التحليل
الـ doc بيقول صراحة في section 1.2:

```
policy->cpuinfo.transition_latency — the time it takes on this CPU to
switch between two frequencies in nanoseconds
```

الـ driver كان بيعمل:

```c
/* WRONG: developer mistakenly set CPUFREQ_ETERNAL */
policy->cpuinfo.transition_latency = CPUFREQ_ETERNAL;
```

الـ schedutil governor بيستخدم الـ `transition_latency` عشان يحسب `rate_limit_us` — الحد الأدنى بين كل frequency switch وتاني. لما الـ latency بـ `CPUFREQ_ETERNAL`، الحساب بيطلع خاطئ وبيسمح بـ switching متكرر جداً.

`cpufreq_verify_within_limits()` مش بتتحقق من الـ latency — هي بس بتتحقق من حدود الـ min/max frequency — فالمشكلة فاتت من غير ما تتشاف في الـ verify callback.

#### الحل

```c
static int rk3562_cpufreq_init(struct cpufreq_policy *policy)
{
    /* 300 microseconds = 300,000 nanoseconds — measured from datasheet */
    policy->cpuinfo.transition_latency = 300 * NSEC_PER_USEC;

    policy->cpuinfo.min_freq = 408000;   /* 408 MHz in kHz */
    policy->cpuinfo.max_freq = 1800000;  /* 1.8 GHz in kHz */

    policy->cur = clk_get_rate(policy->clk) / 1000;
    return 0;
}
```

للتحقق من الأثر:

```bash
# شوف الـ transition_latency اللي الـ core شايفها
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_transition_latency

# حسب الـ rate_limit في schedutil
cat /sys/devices/system/cpu/cpufreq/policy0/schedutil/rate_limit_us
```

#### الدرس المستفاد
**الـ `transition_latency` مش مجرد metadata** — هي بتأثر مباشرة على إيمتى الـ governor يجرؤ يغير الـ frequency. قيمة غلط بتأثر على real-time workloads زي I2C/SPI بدون ما يحصل crash واضح.

---

### السيناريو الثاني: STM32MP1 — IoT Sensor Hub — `policy->cpus` mask غلطان مع SMP

#### العنوان
**الـ cpufreq على STM32MP1 بيخلي الـ Cortex-A7 الاتنين على policies مستقلة بالغلط**

#### السياق
STM32MP1 فيه dual Cortex-A7 بيشتركوا في نفس PLL ونفس voltage rail. فريق الـ BSP كتب cpufreq driver جديد وفي الـ `init()` نسيوا يضبطوا `policy->cpus` بشكل صح. النتيجة: kernel عمل policy منفصلة لكل CPU، وبدأ يحاول يضبط الـ frequency لكل منهم بشكل مستقل — وده مستحيل من ناحية hardware.

#### المشكلة
```
[    3.412] cpufreq: cpu1: policy already exists
[    3.413] cpufreq: Failed to initialize policy for cpu: 1
```

الـ cpu1 بدأ يـ crash عند أول frequency transition لأن الـ PLL اتغير بالنسبله وهو مش عارف.

#### التحليل
الـ documentation بيقول في section 1.2:

```
policy->cpus — Update this with the masks of the (online + offline) CPUs
that do DVFS along with this CPU (i.e. that share clock/voltage rails with it).
```

الـ driver الخاطئ:

```c
static int stm32mp1_cpufreq_init(struct cpufreq_policy *policy)
{
    /* BUG: only setting the current cpu, not its sibling */
    cpumask_set_cpu(policy->cpu, policy->cpus);
    /* ... */
}
```

الصح هو إن الاتنين CPUs على STM32MP1 بيشتركوا في PLL1، فلازم يكونوا في نفس الـ policy:

```c
static int stm32mp1_cpufreq_init(struct cpufreq_policy *policy)
{
    /* Both A7 cores share the same clock — mark them as siblings */
    cpumask_copy(policy->cpus, cpu_possible_mask);
    /* ... */
}
```

ممكن كمان تعمل ده من الـ Device Tree لو الـ driver بيستخدم `cpufreq-dt`:

```dts
/* in DT: mark both CPUs as sharing OPP table */
cpu0: cpu@0 {
    clocks = <&rcc CK_MPU>;
    operating-points-v2 = <&cpu_opp_table>;
};

cpu1: cpu@1 {
    clocks = <&rcc CK_MPU>;
    operating-points-v2 = <&cpu_opp_table>;
    /* cpufreq-dt will auto-detect siblings via shared OPP table */
};
```

#### التحقق

```bash
# لازم تشوف cpu0 و cpu1 في نفس الـ policy
cat /sys/devices/system/cpu/cpufreq/policy0/related_cpus
# Expected: 0 1

# مش المفروض يوجد policy1
ls /sys/devices/system/cpu/cpufreq/
# Expected: policy0 only
```

#### الدرس المستفاد
**الـ `policy->cpus` هي تعريف hardware الحقيقي** — مش تعريف software. أي CPU بيشارك clock أو voltage rail لازم يكون في نفس الـ mask، وإلا الـ core هيحاول يعمل independent DVFS لـ hardware واحد.

---

### السيناريو الثالث: i.MX8MQ — Android TV Box — `fast_switch` بيتسبب في kernel panic

#### العنوان
**الـ schedutil بيكال `fast_switch()` من IRQ context والـ driver بيعمل `mutex_lock()` جواه**

#### السياق
شركة بتبني Android TV box على i.MX8MQ. بعد تفعيل الـ `schedutil` governor، الجهاز بيتعمله kernel panic بشكل عشوائي تحت media playback load:

```
BUG: sleeping function called from invalid context
in_atomic(): 1, irqs_disabled(): 0
mutex_lock+0x...
```

#### المشكلة
الـ developer نسخ كود من الـ `target_index()` callback وحطه في الـ `fast_switch()` callback. الكود ده فيه `mutex_lock()` — وده مش مقبول من scheduler context.

#### التحليل
الـ documentation بيحدد في section 1.6:

```
This function is used for frequency switching from scheduler's context.
Not all drivers are expected to implement it, as sleeping from within
this callback isn't allowed. This callback must be highly optimized to
do switching as fast as possible.
```

الكود الخاطئ:

```c
static unsigned int imx8mq_fast_switch(struct cpufreq_policy *policy,
                                        unsigned int target_freq)
{
    struct imx8mq_data *data = policy->driver_data;

    /* BUG: mutex_lock can sleep — illegal in scheduler context */
    mutex_lock(&data->lock);
    imx8mq_set_freq(data, target_freq);
    mutex_unlock(&data->lock);

    return target_freq;
}
```

الصح: الـ `fast_switch()` لازم تستخدم بس spinlock أو لا تستخدم locking خالص لو الـ hardware register write اتومي:

```c
static unsigned int imx8mq_fast_switch(struct cpufreq_policy *policy,
                                        unsigned int target_freq)
{
    struct imx8mq_data *data = policy->driver_data;
    unsigned long flags;

    /* Use spinlock — safe in atomic context */
    spin_lock_irqsave(&data->slock, flags);
    writel(target_freq / 1000, data->base + FREQ_REG);
    spin_unlock_irqrestore(&data->slock, flags);

    return target_freq;
}
```

لو الـ hardware مش بيدعم atomic frequency switch، الأصح إنك متعملش `fast_switch` خالص:

```c
static struct cpufreq_driver imx8mq_driver = {
    .name         = "imx8mq-cpufreq",
    .init         = imx8mq_cpufreq_init,
    .verify       = cpufreq_generic_frequency_table_verify,
    .target_index = imx8mq_target_index,
    /* fast_switch intentionally omitted — hardware requires sleeping ops */
    .get          = imx8mq_cpufreq_get,
};
```

#### التحقق

```bash
# تحقق إن fast_switch_possible اتضبطت أو لا
cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_transition_latency

# شغل stress test وراقب
stress-ng --cpu 4 --timeout 60 &
dmesg -w | grep -i "cpufreq\|BUG\|panic"
```

#### الدرس المستفاد
**الـ `fast_switch()` context مختلف تماماً عن `target_index()`** — هو scheduler context وممنوع فيه أي sleeping operation. لو مش متأكد، متعملش `fast_switch` والـ core هيرجع لـ `target_index()` تلقائياً.

---

### السيناريو الرابع: AM62x — Automotive ECU — مشكلة الـ `get_intermediate` والـ voltage glitch

#### العنوان
**الـ CPU على AM62x بيعمل voltage glitch أثناء frequency transition بسبب غياب intermediate frequency**

#### السياق
ECU يشتغل على AM62x (TI) في سيارة بيتحكم في CAN bus processing. عند تغيير الـ OPP من 400MHz إلى 1.4GHz، المحرك بيتسجل voltage undershoot قصير بيسبب reset للـ CAN controller. الـ oscilloscope أثبت إن الـ VDD_CORE بينزل لحظياً تحت الـ minimum لـ 1.4GHz في نص الـ transition.

#### المشكلة
الـ driver بيقفز مباشرة من 400MHz إلى 1.4GHz من غير ما يمر بـ intermediate frequency آمنة. الـ PMIC محتاج وقت يرفع الـ voltage قبل ما الـ PLL يقفز للـ high frequency.

#### التحليل
الـ documentation بيشرح في section 1.8:

```
get_intermediate should return a stable intermediate frequency platform wants
to switch to, and target_intermediate() should set CPU to that frequency,
before jumping to the frequency corresponding to 'index'.
Core will take care of sending notifications and driver doesn't have to
handle them in target_intermediate() or target_index().
```

الحل هو إضافة intermediate frequency (مثلاً 800MHz) حيث الـ voltage بيكون كافي للاتنين:

```c
/* Returns safe intermediate freq before big frequency jumps */
static unsigned int am62x_get_intermediate(struct cpufreq_policy *policy,
                                            unsigned int index)
{
    unsigned int target_freq = policy->freq_table[index].frequency;
    unsigned int cur_freq    = policy->cur;

    /* Only use intermediate if jumping more than 400MHz */
    if (abs((int)target_freq - (int)cur_freq) > 400000)
        return 800000; /* 800 MHz — safe at both voltage levels */

    return 0; /* 0 means skip intermediate, go direct */
}

/* Step 1: move to intermediate freq (voltage already raised by PMIC notifier) */
static int am62x_target_intermediate(struct cpufreq_policy *policy,
                                      unsigned int index)
{
    return am62x_set_freq(policy, 800000);
}

/* Step 2: now jump to final target (voltage is stable) */
static int am62x_target_index(struct cpufreq_policy *policy,
                               unsigned int index)
{
    unsigned int freq = policy->freq_table[index].frequency;

    /* Restore to restore_freq on error as doc requires */
    if (am62x_set_freq(policy, freq) < 0) {
        am62x_set_freq(policy, policy->restore_freq);
        return -EIO;
    }
    return 0;
}

static struct cpufreq_driver am62x_driver = {
    .name                = "am62x-cpufreq",
    .init                = am62x_cpufreq_init,
    .verify              = cpufreq_generic_frequency_table_verify,
    .get_intermediate    = am62x_get_intermediate,
    .target_intermediate = am62x_target_intermediate,
    .target_index        = am62x_target_index,
    .get                 = am62x_cpufreq_get,
};
```

#### التحقق

```bash
# راقب الـ frequency steps أثناء الـ transition
trace-cmd record -e power:cpu_frequency &
echo 1400000 > /sys/devices/system/cpu/cpufreq/policy0/scaling_setspeed
trace-cmd report | grep cpu_frequency

# المتوقع تشوف: 400000 -> 800000 -> 1400000
```

#### الدرس المستفاد
**الـ `get_intermediate` / `target_intermediate` موجودين عشان مشاكل hardware حقيقية** — مش مجرد feature اختيارية. أي SoC فيه PMIC بيحتاج رمب-أب للـ voltage قبل frequency jump كبير، محتاج intermediate step عشان يمنع glitches.

---

### السيناريو الخامس: Allwinner H616 — Android TV Box — `CPUFREQ_ENTRY_INVALID` والـ OPP table corruption

#### العنوان
**الـ cpufreq driver على H616 بيستخدم frequencies غير مدعومة من الـ hardware وبيسبب instability**

#### السياق
شركة صينية بتبني Android TV box على Allwinner H616. الـ driver اللي ورثوه بيحمل frequency table فيها 14 OPP من 480MHz لـ 1.8GHz. بعد شهرين من الـ production، بدأوا يلاقوا devices بتعمل hard reset عشوائي عند load عالي. التحقيق أثبت إن H616 chip revisions مختلفة مش كلها بتدعم 1.8GHz — بعضها maximum هو 1.512GHz.

#### المشكلة
الـ driver بيحمل نفس الـ table لكل الـ chip revisions من غير ما يـ invalidate الـ OPPs اللي مش مدعومة.

#### التحليل
الـ documentation بيشرح في section 2:

```
if you want to skip one entry in the table, set the frequency to
CPUFREQ_ENTRY_INVALID.
The entries don't need to be sorted in any particular order, but if they are
cpufreq core will do DVFS a bit quickly for them as search for best match is faster.
```

الحل:

```c
static struct cpufreq_frequency_table h616_freq_table[] = {
    { .frequency = 480000  },
    { .frequency = 600000  },
    { .frequency = 720000  },
    { .frequency = 816000  },
    { .frequency = 1008000 },
    { .frequency = 1200000 },
    { .frequency = 1416000 },
    { .frequency = 1512000 },
    { .frequency = 1608000 }, /* only on rev B+ */
    { .frequency = 1704000 }, /* only on rev B+ */
    { .frequency = 1800000 }, /* only on rev B+ */
    { .frequency = CPUFREQ_TABLE_END },
};

static int h616_cpufreq_init(struct cpufreq_policy *policy)
{
    u32 chip_rev = h616_get_chip_revision();

    /* Invalidate frequencies unsupported by this chip revision */
    if (chip_rev < H616_REV_B) {
        h616_freq_table[8].frequency = CPUFREQ_ENTRY_INVALID;
        h616_freq_table[9].frequency = CPUFREQ_ENTRY_INVALID;
        h616_freq_table[10].frequency = CPUFREQ_ENTRY_INVALID;
    }

    policy->freq_table = h616_freq_table;

    /*
     * Core auto-validates freq_table if policy->freq_table is set.
     * cpufreq_frequency_table_verify() will honor CPUFREQ_ENTRY_INVALID.
     */
    policy->cpuinfo.min_freq = 480000;
    policy->cpuinfo.max_freq = (chip_rev >= H616_REV_B) ? 1800000 : 1512000;
    policy->cpuinfo.transition_latency = 200 * NSEC_PER_USEC;

    return 0;
}

static int h616_cpufreq_verify(struct cpufreq_policy_data *policy)
{
    /* Helper handles CPUFREQ_ENTRY_INVALID automatically */
    return cpufreq_frequency_table_verify(policy, h616_freq_table);
}
```

للتحقق إن الـ invalid entries اتأخدت بجدية:

```bash
# شوف الـ available frequencies — المفروض مش تشوف 1608/1704/1800 على rev A
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies

# اتحقق من الـ max
cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_max_freq

# loop عبر كل frequency وراقب الاستقرار
for freq in $(cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_frequencies); do
    echo $freq > /sys/devices/system/cpu/cpufreq/policy0/scaling_setspeed
    sleep 0.5
    echo "Tested: $freq kHz — OK"
done
```

#### الدرس المستفاد
**`CPUFREQ_ENTRY_INVALID` مش بس تحسين — هو آلية safety حقيقية** للـ chip variants. لما عندك SoC بيتعمل في revisions مختلفة (زي H616 Rev A vs B)، الـ `init()` callback هو المكان الصح تعمل فيه runtime detection وتـ invalidate الـ OPPs اللي مش safe، بدل ما تعمل separate driver لكل revision.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأول لمتابعة تطور kernel — الروابط دي تغطي رحلة الـ cpufreq من البداية للحاضر:

| المقال | الأهمية |
|--------|----------|
| [CPUfreq core for 2.5](https://lwn.net/Articles/1476/) | أول إضافة للـ cpufreq core في kernel 2.5.19 |
| [CPUFreq Documentation (4/4)](https://lwn.net/Articles/8642/) | التوثيق الأصلي للـ CPUFreq subsystem |
| [2.5.40 CPUFreq documentation](https://lwn.net/Articles/11548/) | توثيق مرحلة kernel 2.5.40 |
| [cpufreq governor interface](https://lwn.net/Articles/22080/) | واجهة الـ governor وآلية التكامل مع الـ driver |
| [scheduler-based cpu frequency scaling](https://lwn.net/Articles/643184/) | الأساس النظري لـ schedutil governor |
| [CPUFreq driver using CPPC methods](https://lwn.net/Articles/650781/) | driver الـ CPPC للـ ARM/ACPI platforms |
| [CPUFreq driver using CPPC methods (v2)](https://lwn.net/Articles/648214/) | نسخة مُحدَّثة من patch الـ CPPC driver |
| [cpufreq: interactive: New 'interactive' governor](https://lwn.net/Articles/662209/) | الـ interactive governor اللي استخدمه Android |
| [CPU frequency governors and remote callbacks](https://lwn.net/Articles/732740/) | تحديث الـ policy للـ remote CPUs من الـ scheduler |
| [cpufreq: Specify the default governor on command line](https://lwn.net/Articles/824303/) | تحديد الـ governor من الـ boot parameters |
| [cpufreq: introduce a new AMD CPU frequency control mechanism](https://lwn.net/Articles/868671/) | الـ amd-pstate driver والـ fast_switch |
| [PM / devfreq: Add cpu based scaling support to passive governor](https://lwn.net/Articles/894263/) | تكامل الـ cpufreq مع الـ devfreq subsystem |
| [cpufreq: ARM big LITTLE: Add generic cpufreq driver](https://lwn.net/Articles/544457/) | الـ big.LITTLE cpufreq driver للـ ARM |

---

### التوثيق الرسمي للـ Kernel

**الـ Documentation/** في الـ kernel source هو المرجع الأدق والأحدث:

```
Documentation/cpu-freq/
├── cpu-drivers.rst       ← الملف اللي بندرسه — كيفية كتابة driver جديد
├── core.rst              ← معمارية الـ cpufreq core
├── governors.rst         ← شرح الـ governors المتاحة
├── index.rst             ← فهرس القسم كله
└── user-guide.rst        ← دليل المستخدم والـ sysfs interface

Documentation/admin-guide/pm/
└── cpufreq.rst           ← CPU Performance Scaling الشامل
```

**الـ Online:**
- [docs.kernel.org — cpu-freq index](https://docs.kernel.org/cpu-freq/index.html)
- [docs.kernel.org — cpu-drivers](https://docs.kernel.org/cpu-freq/cpu-drivers.html)
- [docs.kernel.org — CPU Performance Scaling](https://docs.kernel.org/admin-guide/pm/cpufreq.html)
- [Legacy CPUFreq Drivers Documentation (v5.9)](https://www.kernel.org/doc/html/v5.9/admin-guide/pm/cpufreq_drivers.html)

---

### Source Code المهم في الـ Kernel

**الملفات دي** هي القلب التقني — لازم تقرأها جنب التوثيق:

```
drivers/cpufreq/
├── cpufreq.c             ← الـ core: cpufreq_register_driver(), policy management
├── freq_table.c          ← helpers: cpufreq_frequency_table_verify(), iterators
├── cpufreq_governor.c    ← الـ base governor infrastructure
├── longrun.c             ← reference implementation لـ setpolicy() (deprecated)
├── cppc_cpufreq.c        ← CPPC driver مثال على target_index()
├── intel_pstate.c        ← أكبر driver — يدعم setpolicy و fast_switch
└── amd-pstate.c          ← أحدث driver — fast_switch و boost
```

**على GitHub:**
- [drivers/cpufreq/ — torvalds/linux](https://github.com/torvalds/linux/tree/master/drivers/cpufreq)
- [drivers/cpufreq/cpufreq.c](https://github.com/torvalds/linux/blob/master/drivers/cpufreq/cpufreq.c)
- [drivers/cpufreq/cppc_cpufreq.c](https://github.com/torvalds/linux/blob/master/drivers/cpufreq/cppc_cpufreq.c)

---

### Commits مهمة في تاريخ الـ CPUFreq

الـ commits دي غيّرت شكل الـ driver API:

| الـ Feature | الوصف |
|------------|-------|
| `target_index()` introduction | استبدال `target()` القديمة بـ index-based API أبسط وأكثر أماناً |
| `fast_switch()` addition | إضافة callback مخصص للـ scheduler context بدون sleeping |
| `get_intermediate()` / `target_intermediate()` | آلية الـ intermediate frequency للتحول الآمن بين الـ P-states |
| `CPUFREQ_ASYNC_NOTIFICATION` flag | السماح للـ hardware بتغيير الـ frequency بشكل مستقل |
| `policy->freq_table` auto-verification | الـ core بيتحقق تلقائياً من صحة الـ frequency table |

**ابحث في git log بالأمر ده:**
```bash
git log --oneline drivers/cpufreq/cpufreq.c | head -30
git log --oneline --all --grep="target_index" drivers/cpufreq/
```

---

### Mailing List

**الـ mailing list الرسمي** للـ cpufreq و power management:

- **القائمة:** `linux-pm@vger.kernel.org`
- **الأرشيف:** [lore.kernel.org/linux-pm](https://lore.kernel.org/linux-pm/)
- **نقاش تاريخي:** [Mailing List Archive — cpufreq: Processor Clocking Control driver](https://lists.archive.carbon60.com/linux/kernel/1165875)

للبحث في الأرشيف عن موضوع معين:
```
https://lore.kernel.org/linux-pm/?q=cpufreq+driver+target_index
```

---

### صفحات kernelnewbies.org

**الـ kernelnewbies.org** بيوثق الـ changes في كل release — مفيد جداً لفهم إيه اللي اتغير ومتى:

| الصفحة | الـ Feature المتعلق بالـ cpufreq |
|--------|----------------------------------|
| [Linux 4.7](https://kernelnewbies.org/Linux_4.7) | إضافة schedutil governor |
| [Linux 4.9](https://kernelnewbies.org/Linux_4.9) | تحسينات الـ schedutil وتكامله مع الـ scheduler |
| [Linux 4.10](https://kernelnewbies.org/Linux_4.10) | intel_pstate كـ generic cpufreq driver مع governors عادية |
| [Linux 4.11](https://kernelnewbies.org/Linux_4.11) | إضافة `cpufreq.off=1` boot option |
| [Linux 4.14](https://kernelnewbies.org/Linux_4.14) | الـ scheduler يُحدِّث الـ cpufreq policy للـ remote CPUs |
| [Linux 5.7](https://kernelnewbies.org/Linux_5.7) | intel_pstate يستخدم schedutil افتراضياً |
| [Linux 3.7 Driver/Arch](https://kernelnewbies.org/Linux_3.7_DriverArch) | تغييرات بنية الـ cpufreq drivers |

---

### صفحات elinux.org

**الـ elinux.org** مركّز على الـ embedded systems:

| الصفحة | المحتوى |
|--------|---------|
| [Tests:R-CAR-GEN3-CPUFreq](https://elinux.org/Tests:R-CAR-GEN3-CPUFreq) | اختبار الـ cpufreq على R-Car Gen3 — مثال عملي على الـ sysfs interface |
| [CELF PM Requirements 2006](https://www.elinux.org/CELF_PM_Requirements_2006) | متطلبات الـ power management للـ embedded systems وكيف أثّرت على الـ cpufreq |
| [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) | روابط عامة لموارد الـ kernel development |

---

### كتب مُوصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **متاح مجاناً:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- **الفصول ذات الصلة:**
  - Chapter 14: The Linux Device Model — فهم `struct bus_type`, `struct device_driver`
  - Chapter 1: An Introduction to Device Drivers — دورة حياة الـ module وتسجيل الـ driver

> **ملاحظة:** LDD3 قديم (kernel 2.6) لكن الـ concepts الأساسية لا تزال سارية. الـ cpufreq API تغيّر كثيراً بعده.

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول ذات الصلة:**
  - Chapter 11: Timers and Time Management — فهم الـ latency وقياس الوقت
  - Chapter 17: Devices and Modules — `module_init()`, `__initcall`, تسجيل الـ drivers
  - Chapter 14: The Block I/O Layer — نموذج للـ subsystem مع policy وqueue

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل ذو الصلة:**
  - Chapter 16: Kernel Debugging Techniques — debugging الـ cpufreq driver
  - أقسام الـ power management في الـ embedded platforms

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل ذو الصلة:**
  - Chapter 14: Power Management — شرح مفصّل للـ DVFS وتكامله مع الـ scheduler

---

### مصطلحات للبحث

استخدم المصطلحات دي للعثور على معلومات أكثر:

```
# للبحث في Google / DuckDuckGo:
cpufreq_driver target_index implementation
cpufreq_register_driver example driver
linux cpufreq fast_switch scheduler context
cpufreq frequency table helpers CPUFREQ_TABLE_END
linux DVFS driver P-state
cpufreq_verify_within_limits usage
struct cpufreq_driver flags CPUFREQ_ASYNC_NOTIFICATION
linux cpufreq get_intermediate target_intermediate stable frequency
amd-pstate driver fast_switch boost
intel_pstate setpolicy CPUFREQ_POLICY_PERFORMANCE

# للبحث في lore.kernel.org:
cpufreq: introduce target_index
cpufreq: add fast_switch callback
cpufreq: get_intermediate
```

---

### ملخص سريع للمراجع حسب الأولوية

```
1. docs.kernel.org/cpu-freq/cpu-drivers.html    ← ابدأ هنا (الملف اللي بندرسه)
2. docs.kernel.org/admin-guide/pm/cpufreq.html  ← الصورة الكاملة
3. drivers/cpufreq/cpufreq.c                    ← الـ core implementation
4. drivers/cpufreq/cppc_cpufreq.c               ← مثال target_index() حقيقي
5. lwn.net/Articles/643184/                     ← الأساس النظري للـ schedutil
6. lwn.net/Articles/868671/                     ← amd-pstate وآخر تطورات
7. lore.kernel.org/linux-pm/                    ← النقاشات الحية
```
## Phase 8: Writing simple module

### الفكرة

هنعمل module بيستخدم **`cpufreq_register_notifier()`** مع **`CPUFREQ_TRANSITION_NOTIFIER`** عشان نراقب كل مرة الـ kernel بيغير frequency الـ CPU. ده من أكتر الأشياء عملية في الـ cpufreq subsystem لأن الـ governor (زي `schedutil` أو `ondemand`) بيعمل transition كتير جداً وممكن تشوف الـ old/new frequency في real-time.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * cpufreq_spy.c — Monitor CPU frequency transitions via notifier chain
 *
 * Hooks into CPUFREQ_TRANSITION_NOTIFIER to log every frequency change
 * the kernel applies to any CPU policy.
 */

#include <linux/module.h>       /* module_init, module_exit, MODULE_* macros  */
#include <linux/kernel.h>       /* pr_info, pr_err                             */
#include <linux/cpufreq.h>      /* cpufreq_register_notifier, cpufreq_freqs   */
#include <linux/notifier.h>     /* notifier_block, NOTIFY_OK                  */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("Spy on CPU frequency transitions using cpufreq notifier");

/* ------------------------------------------------------------------ */
/* Callback — called by cpufreq core before and after each transition  */
/* ------------------------------------------------------------------ */

static int cpufreq_spy_notifier(struct notifier_block *nb,
                                unsigned long event,
                                void *data)
{
    /*
     * data هنا pointer لـ struct cpufreq_freqs
     * اللي فيها الـ policy، الـ old freq، والـ new freq (بالـ kHz).
     */
    struct cpufreq_freqs *freqs = data;

    /*
     * CPUFREQ_PRECHANGE:  core لسه مغيرش الـ freq — فرصة للـ logging قبل التغيير.
     * CPUFREQ_POSTCHANGE: core خلص من التغيير — نأكد إن التغيير حصل فعلاً.
     */
    if (event == CPUFREQ_PRECHANGE) {
        pr_info("cpufreq_spy: [PRE ] CPU%u  %u kHz -> %u kHz\n",
                freqs->policy->cpu,   /* first CPU of the shared policy   */
                freqs->old,           /* current (about to change) freq   */
                freqs->new);          /* requested target freq            */
    } else if (event == CPUFREQ_POSTCHANGE) {
        pr_info("cpufreq_spy: [POST] CPU%u  settled at %u kHz\n",
                freqs->policy->cpu,
                freqs->new);
    }

    /* NOTIFY_OK = عملنا اللي علينا، استمر في الـ chain */
    return NOTIFY_OK;
}

/* ------------------------------------------------------------------ */
/* notifier_block — الـ struct اللي بيربط الـ callback بالـ chain     */
/* ------------------------------------------------------------------ */

static struct notifier_block cpufreq_spy_nb = {
    .notifier_call = cpufreq_spy_notifier,
    /*
     * priority = 0 (default) —
     * معناه الـ notifier بتاعنا هيتنادى بعد اللي ليه priority أعلى.
     * كافي للـ monitoring من غير ما يأثر على اللي قبله.
     */
};

/* ------------------------------------------------------------------ */
/* module_init — بنسجل الـ notifier في الـ TRANSITION chain           */
/* ------------------------------------------------------------------ */

static int __init cpufreq_spy_init(void)
{
    int ret;

    /*
     * cpufreq_register_notifier() بتضيف الـ notifier_block بتاعنا
     * في الـ blocking notifier chain الخاصة بالـ frequency transitions.
     * CPUFREQ_TRANSITION_NOTIFIER = الـ chain اللي بتتنادى قبل وبعد كل freq change.
     */
    ret = cpufreq_register_notifier(&cpufreq_spy_nb,
                                    CPUFREQ_TRANSITION_NOTIFIER);
    if (ret) {
        pr_err("cpufreq_spy: failed to register notifier: %d\n", ret);
        return ret;
    }

    pr_info("cpufreq_spy: loaded — watching all CPU frequency transitions\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit — لازم نشيل الـ notifier قبل ما الـ module يتنزل       */
/* ------------------------------------------------------------------ */

static void __exit cpufreq_spy_exit(void)
{
    /*
     * لو مشلناش الـ notifier هنا، الـ core هيحاول يكلم الـ callback
     * بعد ما الـ module اتنزل من الـ memory → kernel panic مضمون.
     * cpufreq_unregister_notifier() بتشيله من الـ chain بأمان.
     */
    cpufreq_unregister_notifier(&cpufreq_spy_nb,
                                CPUFREQ_TRANSITION_NOTIFIER);
    pr_info("cpufreq_spy: unloaded\n");
}

module_init(cpufreq_spy_init);
module_exit(cpufreq_spy_exit);
```

---

### Makefile للـ build

```makefile
obj-m += cpufreq_spy.o

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
| `linux/module.h` | الماكروهات الأساسية لأي kernel module |
| `linux/kernel.h` | الـ `pr_info` / `pr_err` للـ logging |
| `linux/cpufreq.h` | تعريف `struct cpufreq_freqs` ودالتي الـ register/unregister |
| `linux/notifier.h` | تعريف `struct notifier_block` والـ `NOTIFY_OK` |

#### الـ `cpufreq_spy_notifier` callback

**الـ `event`** بييجي بقيمتين: `CPUFREQ_PRECHANGE` قبل التغيير و`CPUFREQ_POSTCHANGE` بعده. ده بيخلينا نشوف الـ intention والـ result بشكل منفصل، وهو نفس الـ flow اللي الـ `cpufreq_driver` بيتعامل معاه جوه الـ core.

**الـ `struct cpufreq_freqs *freqs`** جوه الـ data pointer فيها الـ `policy->cpu` (أول CPU في الـ shared policy)، الـ `old` (الـ freq الحالية بالـ kHz)، والـ `new` (الـ target frequency). ده بيديك real-time visibility على قرارات الـ governor.

#### الـ `notifier_block`

**الـ `struct notifier_block`** هو الـ "handle" اللي الـ kernel chain بتستخدمه عشان تعرف تنادي callback بتاعك. الـ `priority = 0` معناه الـ order الافتراضي في الـ chain.

#### الـ `module_init`

**`cpufreq_register_notifier()`** هي الـ function الـ exported اللي الـ kernel بيوفرها عشان أي كود (driver أو module) يقدر يتابع الـ transitions من غير ما يلمس الـ cpufreq core code مباشرة. ده الـ decoupling pattern الصح.

#### الـ `module_exit`

**لازم** تعمل `cpufreq_unregister_notifier()` في الـ exit لأن الـ chain بتيجي في الـ memory الـ kernel الثابتة، ولو الـ module اتنزل من غير ما يشيل نفسه، الـ function pointer هيبقى dangling pointer وأي freq transition بعد كده هتعمل crash.

---

### تجربة عملية

```bash
# بناء الـ module
make

# تحميل الـ module
sudo insmod cpufreq_spy.ko

# راقب الـ output (استخدم stress أو أي load عشان تفرس الـ governor يغير الـ freq)
sudo dmesg -w | grep cpufreq_spy

# مثال على الـ output
# [PRE ] CPU0  800000 kHz -> 2400000 kHz
# [POST] CPU0  settled at 2400000 kHz

# إزالة الـ module
sudo rmmod cpufreq_spy
```

> ملحوظة: محتاج kernel compiled مع `CONFIG_CPU_FREQ=y` وفيه governor نشط زي `ondemand` أو `schedutil`.
