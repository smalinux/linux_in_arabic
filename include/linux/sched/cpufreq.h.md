## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **CPU Frequency Scaling Framework** — اللي بتشوفه في `MAINTAINERS` تحت اسم `CPU FREQUENCY SCALING FRAMEWORK`، بيتمنتنه Rafael J. Wysocki و Viresh Kumar تحت الـ `linux-pm@vger.kernel.org` mailing list.

---

### القصة — ليه الـ file ده موجود أصلاً؟

تخيل عندك موبايل، والـ CPU بيشتغل بأقصى سرعة طول الوقت — البطارية هتخلص في ساعتين، والموبايل هيتحرق. الحل هو إنك **تخفض سرعة الـ CPU لما الشغل قليل، وترفعها لما الضغط عالي**. ده اللي بيعمله الـ **CPUFreq subsystem** — بيتحكم في تردد الـ CPU (وبالتالي الاستهلاك) ديناميكياً.

بس في مشكلة: **مين الأكثر دراية بإن الـ CPU محتاج أكتر أو أقل؟**

الإجابة هي الـ **Scheduler** — لأنه هو اللي شايف كل الـ tasks، شايف مين شغال ومين واقف، وعارف نسبة الـ utilization (الاستخدام) على كل CPU core في الوقت الفعلي.

الـ **CPUFreq governors** (زي `schedutil`) هم الطرف التاني — هم المسؤولين عن اتخاذ قرار "ارفع التردد أو خفضه" بناءً على المعلومات دي.

**المشكلة:** الـ Scheduler والـ CPUFreq governor كودين منفصلين تماماً. عايزينهم يتكلموا مع بعض من غير ما يكونوا tightly coupled.

**الحل:** الـ file ده — `include/linux/sched/cpufreq.h` — هو الـ **interface contract** (العقد) بينهم.

---

### الـ File بيعمل إيه بالظبط؟

الـ file ده صغير جداً (39 سطر) لكن دوره محوري — هو بيعرّف:

#### 1. الـ `SCHED_CPUFREQ_IOWAIT` Flag

```c
#define SCHED_CPUFREQ_IOWAIT  (1U << 0)
```

لما task بتصحى من انتظار I/O (زي قراءة disk أو network)، الـ scheduler بيحط الـ flag ده في الـ `flags` parameter. الـ governor لما يشوفه بيعرف "الـ task ده كان فاضي بسبب I/O مش بسبب إنه مش محتاج CPU" — فيرفع التردد بسرعة (**IO wait boost**).

#### 2. الـ `struct update_util_data`

```c
struct update_util_data {
    void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
};
```

ده الـ **callback hook** — الـ governor بيسجّل function pointer هنا، والـ scheduler بيناديها كل ما يحتاج يُبلّغ الـ governor بـ utilization جديدة. ده الـ Observer Pattern بالظبط.

#### 3. الـ Registration Functions

```c
void cpufreq_add_update_util_hook(int cpu, struct update_util_data *data, ...);
void cpufreq_remove_update_util_hook(int cpu);
```

- **الـ `add`**: الـ governor (مثلاً `schedutil`) بيستخدمها لما بيتفعّل — بيقول "أنا موجود على الـ CPU ده، كلمني لو في أي تغيير في الـ utilization".
- **الـ `remove`**: لما الـ governor بيتوقف أو الـ CPU بيروح offline.

#### 4. الـ Helper Math Functions

```c
static inline unsigned long map_util_freq(unsigned long util,
                                          unsigned long freq,
                                          unsigned long cap)
{
    return freq * util / cap;  // scale frequency by utilization ratio
}

static inline unsigned long map_util_perf(unsigned long util)
{
    return util + (util >> 2);  // add 25% headroom on top of utilization
}
```

- **`map_util_freq`**: تحويل الـ utilization لـ target frequency. لو الـ CPU بيشتغل 75% من طاقته، والـ max frequency هو 2 GHz → الـ target هو 1.5 GHz.
- **`map_util_perf`**: بتضيف **25% headroom** فوق الـ utilization. ده عشان الـ CPU ميوصلش لـ 100% قبل ما التردد يترفع — بيحسب مقدماً.

---

### الصورة الكاملة — رحلة الـ Utilization

```
┌─────────────────────────────────────────────────────┐
│                    Scheduler                        │
│  يشوف tasks، يحسب util على كل CPU                 │
│  → cpufreq_update_util() [kernel/sched/cpufreq.c]  │
└───────────────────────┬─────────────────────────────┘
                        │  يستدعي الـ callback المسجّل
                        │  (struct update_util_data)
                        ▼
┌─────────────────────────────────────────────────────┐
│              schedutil Governor                     │
│  [kernel/sched/cpufreq_schedutil.c]                │
│  - بيحسب الـ target frequency                      │
│  - map_util_freq() + map_util_perf()               │
│  - يطلب من الـ cpufreq driver رفع/خفض التردد      │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│              CPUFreq Driver                         │
│  [drivers/cpufreq/]                                │
│  - بيكلم الـ hardware فعلياً                       │
│  - Intel P-states / ARM big.LITTLE / etc.          │
└─────────────────────────────────────────────────────┘
```

**الـ `sched/cpufreq.h` هو الخط الفاصل بين الـ Scheduler والـ Governor** — من غيره، كل طرف مش عارف الطرف التاني.

---

### الـ `cpufreq_this_cpu_can_update()`

```c
bool cpufreq_this_cpu_can_update(struct cpufreq_policy *policy);
```

الـ Governor مش دايماً ممكن يعمل frequency update من أي CPU — بعض الـ hardware بس بتقبل الأوامر من الـ CPU اللي بيـmanage الـ policy. الـ function دي بتتحقق إن الـ CPU الحالي مسموحله يعمل update.

---

### الـ Files المهمة في الـ Subsystem

| الـ File | الدور |
|---|---|
| `include/linux/sched/cpufreq.h` | **Interface** بين الـ scheduler والـ governors (الـ file ده) |
| `include/linux/cpufreq.h` | الـ core CPUFreq API — `struct cpufreq_policy`، الـ governors، الـ drivers |
| `kernel/sched/cpufreq.c` | تنفيذ الـ hook registration functions |
| `kernel/sched/cpufreq_schedutil.c` | الـ `schedutil` governor — الـ governor الرسمي المبني جوه الـ scheduler |
| `drivers/cpufreq/` | الـ hardware drivers (intel_pstate، acpi-cpufreq، إلخ) |
| `kernel/sched/sched.h` | الـ internal scheduler definitions |
| `Documentation/admin-guide/pm/cpufreq.rst` | الـ user-facing documentation |
| `Documentation/cpu-freq/` | مزيد من التوثيق الفني |
## Phase 2: شرح الـ Scheduler-CPUFreq Interface Framework

### المشكلة اللي بيحلها الـ Subsystem ده

الـ CPU الحديث مش بيشتغل بفريكونسي ثابتة — ليه جدول أوبي بيسموه **OPP (Operating Performance Points)**، يعني combinations من voltage وfrequency ممكن يتنقل بينها.

المشكلة الأساسية: **مين يقرر امتى يرفع أو يخفض الفريكونسي؟**

قبل الـ schedutil، الحل كان governors زي `ondemand` و`conservative` بيعملوا polling كل فترة — بيسألوا "الـ CPU شغال قد إيه؟" بشكل دوري. الـ overhead ده مش كافي ومش دقيق لأنه:

- الـ sampling بيحصل متأخر — اللحظة اللي بتعرف فيها الـ CPU مشحون، يمكن خلص الشغل.
- الـ scheduler نفسه عنده المعلومة الأدق — هو اللي بيعرف الـ utilization في real-time.

الحل: **اربط الـ CPUFreq بالـ scheduler مباشرة** — خلي الـ scheduler يقول للـ cpufreq governor على طول لما يلاقي شغل جديد.

---

### الحل: الـ Scheduler-CPUFreq Hook

الـ kernel حل الموضوع بواسطة **hook خفيف جداً** — بدل ما الـ governor يعمل polling، الـ scheduler نفسه بيستدعي callback مسجل لما يحدث أي تغيير في utilization.

الـ header الأساسي `include/linux/sched/cpufreq.h` بيعرّف الـ interface ده — هو مش الـ cpufreq subsystem كله، هو بالظبط **نقطة التقاء** بين الـ scheduler وبين الـ cpufreq governors.

---

### التشبيه الحقيقي

تخيل **مصنع** فيه:
- **مدير إنتاج** (الـ scheduler) — عارف كل ثانية كام عامل شغال وكام مستني شغل.
- **مدير طاقة** (الـ cpufreq governor) — مسؤول عن كم موتور هيشتغل بأي سرعة.

**الطريقة القديمة (ondemand):** مدير الطاقة بيجي كل 10 دقائق يسأل مدير الإنتاج "إيه الحال؟" ← بطيء وغلط.

**الطريقة الجديدة (schedutil):** مدير الإنتاج عنده زر — أي ما يجي أوردر جديد أو يخلص أوردر، يضغط الزر ده على طول ومدير الطاقة بيعرف في نفس اللحظة.

الـ `struct update_util_data` ده هو الزر بالظبط — pointer لـ callback مسجل جنب كل CPU.

**ماب كامل للتشبيه:**

| عنصر المصنع | المقابل في الـ kernel |
|---|---|
| مدير الإنتاج | الـ scheduler (scheduler hotpath) |
| مدير الطاقة | الـ cpufreq governor (schedutil) |
| الزر | `struct update_util_data` + callback |
| الأوردر الجديد | task جديد يتضاف للـ runqueue |
| سرعة الموتور | CPU frequency (kHz) |
| عدد العمال الشغالين | CPU utilization (PELT signal) |
| قواعد تغيير السرعة | `sugov_tunables` (rate_limit_us) |

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Linux Scheduler                          │
│                                                                 │
│  task wakeup / tick / load_balance                              │
│         │                                                       │
│         ▼                                                       │
│  cpufreq_update_util(rq, flags)   ← في scheduler hotpath       │
│         │                                                       │
│         │  per-CPU RCU pointer lookup                           │
│         ▼                                                       │
│  cpufreq_update_util_data[cpu]->func(data, time, flags)        │
└────────────────────┬────────────────────────────────────────────┘
                     │  callback registered by governor
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    schedutil Governor                           │
│                                                                 │
│  sugov_update_single_freq() / sugov_update_shared_freq()       │
│         │                                                       │
│         ├── map_util_freq(util, freq, cap)  ← inline math      │
│         ├── map_util_perf(util)             ← 25% headroom      │
│         │                                                       │
│         ▼                                                       │
│  Fast switch?                                                   │
│    ├── YES: cpufreq_driver_fast_switch()  ← direct, in IRQ ctx │
│    └── NO:  irq_work → kthread → cpufreq_driver_target()      │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                  cpufreq Driver (HW specific)                   │
│   e.g., cpufreq-dt, qcom-cpufreq-hw, arm_big_little            │
│                                                                 │
│   clk_set_rate() / regmap write / SCMI message                 │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     ▼
              ┌──────────────┐
              │  CPU Hardware │
              │  PLL / DVFS  │
              └──────────────┘
```

---

### الـ Core Abstraction: `struct update_util_data`

```c
struct update_util_data {
    void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
};
```

ده أبسط struct في الملف بس ده قلب الـ framework كله.

**الفكرة:** كل CPU عنده pointer واحد من النوع ده محفوظ في per-CPU variable:

```c
DEFINE_PER_CPU(struct update_util_data __rcu *, cpufreq_update_util_data);
```

لما الـ scheduler يعمل `cpufreq_update_util()`:
1. يعمل `rcu_dereference()` للـ per-CPU pointer.
2. لو فيه governor مسجل → يستدعي `->func()`.
3. لو مفيش → يرجع بدون أي overhead.

**الـ `__rcu` annotation** مهمة — لأن الـ hook بيتسجل/بيتشال وقت hotplug، والـ scheduler بيشتغل من IRQ/softirq context — فلازم RCU read-side protection.

---

### الـ Registration API

```c
// Governor يسجل نفسه لما يبدأ على CPU معين
void cpufreq_add_update_util_hook(int cpu,
                                  struct update_util_data *data,
                                  void (*func)(...));

// Governor يشيل نفسه (عند stop أو hotplug)
void cpufreq_remove_update_util_hook(int cpu);
```

**مثال حقيقي من schedutil:**

```c
// في sugov_start():
for_each_cpu(cpu, policy->cpus) {
    struct sugov_cpu *sg_cpu = &per_cpu(sugov_cpu, cpu);
    // struct sugov_cpu بيحتوي على update_util كأول field
    // ده pattern الـ container_of
    cpufreq_add_update_util_hook(cpu, &sg_cpu->update_util,
                                 sugov_update_single_freq);
}
```

الـ `struct sugov_cpu` بيستخدم pattern مهم:

```c
struct sugov_cpu {
    struct update_util_data update_util;  // لازم يكون أول field
    struct sugov_policy    *sg_policy;
    unsigned int            cpu;
    bool                    iowait_boost_pending;
    unsigned int            iowait_boost;
    u64                     last_update;
    unsigned long           util;
    unsigned long           bw_min;
};
```

لما الـ callback بيتستدعي بـ `struct update_util_data *data`، الـ governor يعمل:
```c
struct sugov_cpu *sg_cpu = container_of(data, struct sugov_cpu, update_util);
```
فيوصل لكل المعلومات المحيطة.

---

### الـ Utility Math Functions

#### `map_util_freq()`

```c
static inline unsigned long map_util_freq(unsigned long util,
                                          unsigned long freq,
                                          unsigned long cap)
{
    return freq * util / cap;
}
```

**معناها:** لو الـ CPU شغال `util` من أصل `cap` (الـ capacity عند الفريكونسي الحالية `freq`)، إيه الفريكونسي المطلوبة؟

**مثال عددي:**
- `util = 512` (نص الطاقة المتاحة)
- `freq = 1800 MHz` (الفريكونسي الحالي)
- `cap = 1024` (الـ capacity الكاملة عند 1800 MHz)
- النتيجة: `1800 * 512 / 1024 = 900 MHz`

يعني الـ CPU محتاج بس 900 MHz دلوقتي.

#### `map_util_perf()`

```c
static inline unsigned long map_util_perf(unsigned long util)
{
    return util + (util >> 2);
}
```

**معناها:** زود الـ utilization بـ 25% كـ headroom.

`util >> 2` = `util / 4` = 25%

**ليه؟** لأن الـ utilization signal بيجي من **PELT (Per-Entity Load Tracking)** اللي بيعمل exponential moving average — يعني دايماً متأخر شوية عن الواقع. الـ 25% headroom بيعوض التأخر ده ويضمن إن الفريكونسي المختارة كافية للـ burst القادم.

**مثال:**
- `util = 512` → `map_util_perf(512) = 512 + 128 = 640`
- دلوقتي بنحسب الفريكونسي على أساس 640 مش 512.

---

### الـ SCHED_CPUFREQ_IOWAIT Flag

```c
#define SCHED_CPUFREQ_IOWAIT (1U << 0)
```

ده flag بيتبعت في الـ `flags` parameter للـ callback.

**السيناريو:** task كان نايم على I/O (disk read مثلاً) وصحي — الـ scheduler يعمل `cpufreq_update_util()` بالـ flag ده.

**ردة فعل schedutil:** بيعمل **iowait boost** — بيرفع الفريكونسي لفوق مؤقتاً حتى الـ task يكمل اللي عايز يعمله بعد رجوع الـ I/O، لأن غالباً بعد رجوع الـ I/O الـ task هيبقى CPU-bound لثانية.

```c
// في sugov_iowait_boost():
if (flags & SCHED_CPUFREQ_IOWAIT) {
    sg_cpu->iowait_boost_pending = true;
    // boost يبدأ من IOWAIT_BOOST_MIN = SCHED_CAPACITY_SCALE / 8
}
```

---

### `cpufreq_this_cpu_can_update()`

```c
bool cpufreq_this_cpu_can_update(struct cpufreq_policy *policy)
{
    return cpumask_test_cpu(smp_processor_id(), policy->cpus) ||
        (policy->dvfs_possible_from_any_cpu &&
         rcu_dereference_sched(*this_cpu_ptr(&cpufreq_update_util_data)));
}
```

**ليه محتاجينها؟**

في ARM big.LITTLE أو SoCs فيها CPUs بيشاركوا نفس الـ PLL:
- CPUs 0-3 بيشتغلوا على نفس الفريكونسي (نفس الـ `cpufreq_policy`).
- لو CPU-1 لاقى إنه محتاج فريكونسي أعلى، هل يقدر يطلب ده نيابة عن الكلاستر كله؟

الإجابة:
- **لو `dvfs_possible_from_any_cpu = true`**: أيوه، أي CPU يقدر يطلب تغيير الفريكونسي.
- **لو لأ**: الـ CPU اللي شايل الـ policy هو بس اللي يعمل التغيير.

الشرط الثاني `rcu_dereference_sched(...)` بيتأكد إن الـ CPU الحالي لسه مش بيعمل offline — لأنه لو بيعمل offline، شيل الـ hook منه مسبقاً.

---

### إيه اللي الـ Subsystem ده بيمتلكه وإيه اللي بيفوضه

| المسؤولية | مين بيتكفل بيها |
|---|---|
| تسجيل/إزالة الـ hook | `sched/cpufreq.c` (الـ framework) |
| حفظ الـ per-CPU pointer بأمان (RCU) | `sched/cpufreq.c` |
| حساب `util` من الـ runqueue | الـ scheduler (PELT في `kernel/sched/pelt.c`) |
| قرار الفريكونسي المطلوبة | الـ governor (schedutil) |
| تطبيق تغيير الفريكونسي فعلياً | الـ cpufreq driver |
| تعريف OPPs المتاحة | الـ driver + DT/ACPI |
| الـ thermal throttling | الـ thermal subsystem عبر `freq_qos` |

---

### الـ Subsystems التانية المرتبطة

- **PELT (Per-Entity Load Tracking)** — الـ algorithm اللي بيحسب `util` داخل الـ scheduler، بيستخدم exponential moving average على الـ CPU usage التاريخي.
- **PM QoS** — بيسمح لـ drivers تانية (زي thermal) تفرض حدود min/max على الفريكونسي بشكل asynchronous.
- **RCU (Read-Copy-Update)** — الـ synchronization mechanism المستخدم للـ per-CPU pointer لأن الـ hook بيتشال/بيتضاف من context غير scheduler hotpath.
- **IRQ Work** — الـ mechanism اللي بيستخدمه schedutil لو الـ fast switch مش متاح، بيأجل تغيير الفريكونسي خارج الـ scheduler lock.

---

### رسمة العلاقة بين الـ Structs

```
per_cpu(cpufreq_update_util_data, N)   [per-CPU, RCU-protected]
           │
           │ points to
           ▼
┌──────────────────────────┐
│   struct update_util_data │   ← embedded في sugov_cpu
│   +func (callback ptr)   │
└──────────┬───────────────┘
           │ container_of
           ▼
┌────────────────────────────────────┐
│        struct sugov_cpu            │
│  +update_util                      │
│  +sg_policy ──────────────────┐   │
│  +cpu                          │   │
│  +iowait_boost_pending         │   │
│  +iowait_boost                 │   │
│  +util                         │   │
│  +bw_min                       │   │
└────────────────────────────────┼───┘
                                 │ points to
                                 ▼
                    ┌────────────────────────────┐
                    │    struct sugov_policy      │
                    │  +policy ──────────────┐   │
                    │  +next_freq            │   │
                    │  +last_freq_update_time│   │
                    │  +freq_update_delay_ns │   │
                    │  +irq_work             │   │
                    └────────────────────────┼───┘
                                             │ points to
                                             ▼
                              ┌──────────────────────────┐
                              │   struct cpufreq_policy   │
                              │  +cpus (cpumask)          │
                              │  +min / max / cur (kHz)   │
                              │  +fast_switch_enabled     │
                              │  +dvfs_possible_from_any  │
                              │  +governor                │
                              │  +driver_data             │
                              └──────────────────────────┘
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Config Options — Cheatsheet

| الاسم | القيمة | المعنى |
|---|---|---|
| `SCHED_CPUFREQ_IOWAIT` | `(1U << 0)` | يشير إن الـ CPU كان blocked على I/O — يخلي الـ governor يرفع الـ frequency |
| `CONFIG_CPU_FREQ` | build option | لو مش موجود، كل الـ API بيبقى stubs فارغة |

---

### الـ Structs المهمة

#### 1. `struct update_util_data`

**الغرض:** ده الـ hook بين الـ scheduler والـ cpufreq governor. الـ scheduler بيحتفظ بـ pointer منه per-CPU، وبيستدعي الـ `func` كل ما يتغير الـ CPU utilization.

```c
struct update_util_data {
    void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
    //   ^callback بتاع الـ governor           ^timestamp بالـ ns   ^SCHED_CPUFREQ_IOWAIT
};
```

| الـ Field | النوع | الشرح |
|---|---|---|
| `func` | function pointer | الـ callback اللي الـ governor بيسجله — بيتنادى من الـ scheduler مباشرة |

**ملاحظة مهمة:** الـ struct صغير عمدًا — الـ governor بيعمل `container_of` عشان يوصل للـ governor-private data.

---

#### 2. `struct cpufreq_policy` (من `include/linux/cpufreq.h`)

**الغرض:** يمثل الـ frequency policy لمجموعة CPUs بتشارك نفس الـ clock domain.

| الـ Field | النوع | الشرح |
|---|---|---|
| `cpus` | `cpumask_var_t` | الـ online CPUs في الـ policy |
| `related_cpus` | `cpumask_var_t` | online + offline CPUs |
| `real_cpus` | `cpumask_var_t` | related CPUs اللي physically present |
| `cpu` | `unsigned int` | الـ CPU المسؤول عن إدارة الـ policy |
| `min` / `max` | `unsigned int` | حدود الـ frequency بالـ kHz |
| `cur` | `unsigned int` | الـ frequency الحالية بالـ kHz |
| `governor` | `struct cpufreq_governor *` | الـ governor الحالي (schedutil, ondemand, إلخ) |
| `governor_data` | `void *` | private data بتاع الـ governor |
| `fast_switch_possible` | `bool` | الـ driver يقدر يغير الـ frequency من أي CPU |
| `fast_switch_enabled` | `bool` | الـ governor فعّل الـ fast path |
| `dvfs_possible_from_any_cpu` | `bool` | DVFS ممكن cross-policy |
| `rwsem` | `struct rw_semaphore` | يحمي الـ policy structure |
| `transition_lock` | `spinlock_t` | يحمي الـ transition_ongoing flag |
| `transition_wait` | `wait_queue_head_t` | الـ tasks بتنتظر نهاية الـ transition |
| `freq_table` | `struct cpufreq_frequency_table *` | جدول الـ frequencies المتاحة |
| `stats` | `struct cpufreq_stats *` | إحصائيات الـ frequency |
| `cdev` | `struct thermal_cooling_device *` | للـ thermal throttling |
| `boost_enabled` | `bool` | هل الـ boost مفعّل per-policy |

---

### علاقات الـ Structs — ASCII Diagram

```
per-CPU (RCU protected)
┌─────────────────────────────────────────┐
│  cpu_rq(cpu)->update_util_data          │
│         │                               │
│         ▼                               │
│  struct update_util_data  ◄─────────────┼── governor registers this
│  ┌──────────────────────┐               │
│  │  func ptr ──────────►│ governor cb   │
│  └──────────────────────┘               │
│         │  (container_of)               │
│         ▼                               │
│  governor private struct                │
│  (e.g. struct sugov_cpu in schedutil)  │
│  ┌──────────────────────┐               │
│  │  update_util_data ud │               │
│  │  sugov_policy *sg_p ─┼──────────────►│
│  └──────────────────────┘               │
└─────────────────────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │  struct cpufreq_policy  │
                    │  ┌───────────────────┐  │
                    │  │ cpus (cpumask)    │  │
                    │  │ min / max / cur   │  │
                    │  │ governor ─────────┼──┼──► struct cpufreq_governor
                    │  │ governor_data     │  │
                    │  │ freq_table ───────┼──┼──► cpufreq_frequency_table[]
                    │  │ rwsem             │  │
                    │  │ transition_lock   │  │
                    │  │ stats ────────────┼──┼──► struct cpufreq_stats
                    │  │ cdev ─────────────┼──┼──► thermal_cooling_device
                    │  └───────────────────┘  │
                    └─────────────────────────┘
```

---

### الـ Inline Functions — شرح سريع

#### `map_util_freq(util, freq, cap)`

```c
static inline unsigned long map_util_freq(unsigned long util,
                                          unsigned long freq,
                                          unsigned long cap)
{
    return freq * util / cap;
    // يحسب الـ frequency المطلوبة proportional لـ utilization
    // مثال: util=512, freq=2400MHz, cap=1024 → 1200MHz
}
```

**المعادلة:** `target_freq = max_freq × (utilization / capacity)`

#### `map_util_perf(util)`

```c
static inline unsigned long map_util_perf(unsigned long util)
{
    return util + (util >> 2);
    // يضيف 25% margin فوق الـ utilization
    // util >> 2 = util / 4 = 25%
    // مثال: util=800 → 800 + 200 = 1000
}
```

**السبب:** الـ utilization المقاسة دايمًا أقل من الحقيقية بسبب latency في الـ sampling، فبنضيف margin عشان ما نتأخرش في رفع الـ frequency.

---

### الـ API Functions — جدول كامل

| الدالة | متى تتنادى | من مين |
|---|---|---|
| `cpufreq_add_update_util_hook(cpu, data, func)` | عند تفعيل الـ governor | الـ governor (schedutil مثلًا) |
| `cpufreq_remove_update_util_hook(cpu)` | عند إيقاف الـ governor | الـ governor |
| `cpufreq_this_cpu_can_update(policy)` | داخل الـ util callback | الـ governor عشان يعرف يبعت request |
| `map_util_freq(util, freq, cap)` | في الـ frequency selection | الـ governor |
| `map_util_perf(util)` | قبل `map_util_freq` | الـ governor لتطبيق الـ 25% margin |

---

### Lifecycle Diagram — دورة حياة الـ Hook

```
                  GOVERNOR START
                       │
                       ▼
         cpufreq_add_update_util_hook(cpu, data, func)
                       │
                       │  [يكتب في per-CPU pointer باستخدام RCU]
                       │
                       ▼
              ┌─────────────────┐
              │  HOOK ACTIVE    │
              │                 │
              │ scheduler       │
              │   │             │
              │   │ (per tick   │
              │   │  or task    │
              │   │  enqueue)   │
              │   ▼             │
              │ update_util_data│
              │   →func(data,   │
              │         time,   │
              │         flags)  │
              │                 │
              │ governor decides│
              │ new frequency   │
              └────────┬────────┘
                       │
                  GOVERNOR STOP
                       │
                       ▼
         cpufreq_remove_update_util_hook(cpu)
                       │
                       │  [يكتب NULL في per-CPU pointer]
                       │
                       ▼
              HOOK REMOVED (RCU grace period)
```

---

### Call Flow Diagram — مسار تحديث الـ Frequency

```
scheduler (tick / task wakeup)
  │
  │  [يلاحظ تغيير في utilization]
  │
  ▼
cpufreq_update_util(rq, flags)          ← entry point في scheduler
  │
  ├─ [flags |= SCHED_CPUFREQ_IOWAIT]    ← لو كان blocked على I/O
  │
  ▼
data->func(data, time, flags)           ← الـ callback المسجل
  │   (per-CPU RCU dereference)
  │
  ▼  [مثال: schedutil governor]
sugov_update_single() / sugov_update_shared()
  │
  ├─ map_util_perf(util)                ← +25% margin
  │
  ├─ map_util_freq(util, max, cap)      ← util → kHz
  │
  ├─ cpufreq_this_cpu_can_update()?     ← هل ممكن من هنا؟
  │    │
  │    ├─ YES + fast_switch_enabled:
  │    │    cpufreq_driver_fast_switch() ← مباشر من الـ scheduler context
  │    │
  │    └─ NO / slow path:
  │         irq_work_queue()            ← defer لـ irq context
  │              │
  │              ▼
  │         cpufreq_update_policy()
  │              │
  │              ▼
  │         driver->target() / target_index()
  │              │
  │              ▼
  │         hardware register write (voltage + frequency)
  │
  └─ [update stats, notify thermal]
```

---

### لماذا `cpufreq_this_cpu_can_update()`؟

الـ function دي بتتحقق من حاجتين:

1. **`dvfs_possible_from_any_cpu`**: لو الـ policy بتاع CPU آخر، هل ممكن نغير الـ frequency منه؟
2. **`fast_switch_enabled`**: لو مش fast switch، محتاجين نكون على الـ policy CPU نفسه عشان نتجنب الـ race condition.

```c
// التطبيق الفعلي في cpufreq/core.c
bool cpufreq_this_cpu_can_update(struct cpufreq_policy *policy)
{
    return policy->dvfs_possible_from_any_cpu ||
           cpumask_test_cpu(smp_processor_id(), policy->cpus);
}
```

---

### استراتيجية الـ Locking

#### الـ Locks في هذا الملف وعلاقاتها

| الـ Lock | النوع | يحمي إيه | من الـ `cpufreq.h` |
|---|---|---|---|
| `policy->rwsem` | `rw_semaphore` | كل الـ policy structure عند القراءة/الكتابة | نعم |
| `policy->transition_lock` | `spinlock_t` | `transition_ongoing` flag فقط | نعم |
| per-CPU RCU pointer | RCU | الـ `update_util_data *` per-CPU | ضمني في الـ design |

#### قواعد الـ Locking

```
┌─────────────────────────────────────────────────────────┐
│  SCHEDULER HOT PATH (no sleeping allowed)               │
│                                                         │
│  rcu_read_lock()                                        │
│    data = rcu_dereference(per_cpu(util_update, cpu))    │
│    if (data)                                            │
│        data->func(data, time, flags)    ← governor cb  │
│  rcu_read_unlock()                                      │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  GOVERNOR REGISTRATION (can sleep)                      │
│                                                         │
│  cpufreq_add_update_util_hook():                        │
│    rcu_assign_pointer(per_cpu(util_update, cpu), data)  │
│    synchronize_rcu()  ← ينتظر لحد ما كل readers        │
│                           يخلصوا قبل ما نكمل           │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  GOVERNOR REMOVAL (can sleep)                           │
│                                                         │
│  cpufreq_remove_update_util_hook():                     │
│    rcu_assign_pointer(per_cpu(util_update, cpu), NULL)  │
│    synchronize_rcu()  ← critical! ينتظر عشان يضمن      │
│                           إن الـ func مش بتتنفذ        │
│                           بعد ما نرجع                   │
└─────────────────────────────────────────────────────────┘
```

#### ترتيب الـ Locks (Lock Ordering) — عشان نتجنب deadlock

```
policy->rwsem (write)          ← الأعلى في الـ hierarchy
    │
    └──► policy->transition_lock (spin)
              │
              └──► RCU read lock (per-CPU hook pointer)
                        │
                        └──► governor internal lock
```

**القاعدة:** الـ RCU read lock ممكن يتاخد من الـ scheduler context اللي مش بيتعامل مع الـ rwsem أصلًا — ده بيخلي الـ hot path خالي من أي sleeping lock.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

| Function / Macro | النوع | الغرض |
|---|---|---|
| `cpufreq_add_update_util_hook` | function | تسجيل callback بتاع الـ governor على CPU معين |
| `cpufreq_remove_update_util_hook` | function | إزالة الـ callback من الـ CPU |
| `cpufreq_this_cpu_can_update` | function | check إذا الـ CPU الحالي مسموحله يعمل frequency update |
| `map_util_freq` | static inline | تحويل utilization لـ frequency مطلوبة |
| `map_util_perf` | static inline | تضخيم الـ utilization بـ 25% لـ headroom |
| `SCHED_CPUFREQ_IOWAIT` | macro/flag | flag بيدل إن الـ wakeup سببه I/O wait |
| `struct update_util_data` | struct | container للـ callback بتاع الـ governor |

---

### التصنيفات

```
Registration ──► cpufreq_add_update_util_hook
                 cpufreq_remove_update_util_hook

Runtime Query ──► cpufreq_this_cpu_can_update

Math Helpers ──► map_util_freq
                 map_util_perf

Data Types ──► struct update_util_data
               SCHED_CPUFREQ_IOWAIT
```

---

### Group 1: Registration — ربط الـ Governor بالـ Scheduler

الـ scheduler بيعرف إمتى utilization الـ CPU اتغيرت، لكنه مش هو اللي بيقرر الـ frequency — ده شغل الـ governor. فأحتاج لـ **hook mechanism** ربط الاتنين ببعض بدون tight coupling.

الـ governor بيسجّل نفسه عن طريق `cpufreq_add_update_util_hook`، وبيديه function pointer. كل ما الـ scheduler يلاحظ تغيير في الـ utilization، بيستدعي الـ hook ده على الـ CPU المناسب.

---

#### `cpufreq_add_update_util_hook`

```c
void cpufreq_add_update_util_hook(int cpu,
                                   struct update_util_data *data,
                                   void (*func)(struct update_util_data *data,
                                                u64 time,
                                                unsigned int flags));
```

**الـ function بتعمل إيه:**
بتربط `func` بالـ `cpu` المحدد عن طريق تخزين الـ pointer في الـ per-CPU variable الخاص بالـ hook. الـ scheduler بعد كده بيستدعي الـ `func` دي في كل مرة يحسب فيها utilization جديد للـ CPU. الـ `data` pointer بيعدّي للـ callback عشان الـ governor يقدر يوصل لـ state الخاصة بيه.

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `cpu` | `int` | رقم الـ CPU اللي هيُسجَّل عليه الـ hook |
| `data` | `struct update_util_data *` | الـ context اللي بيتعدّى للـ callback — الـ governor بيضم الـ struct ده جوا struct أكبر |
| `func` | function pointer | الـ callback اللي الـ scheduler هيستدعيه عند كل utilization update |

**الـ `func` signature:**

| Parameter | الشرح |
|---|---|
| `data` | نفس الـ pointer اللي اتسجّل — الـ governor بيعمله `container_of` للوصول لـ state بتاعته |
| `time` | الوقت الحالي بالـ nanoseconds (من `ktime_get()`) |
| `flags` | مثلاً `SCHED_CPUFREQ_IOWAIT` لو الـ wakeup سببه I/O |

**Return value:** `void`

**Key details:**
- **لازم يُستدعى** من context بيمسك الـ `cpufreq_policy` rwsem أو ما يعادله، لأن الـ hook بيُخزَّن في per-CPU variable محمي.
- لو في hook موجود بالفعل على الـ CPU، بيعمل `WARN_ON` ويُعيّن الجديد — مفيش stacking للـ hooks.
- الـ implementation الفعلية في `kernel/sched/cpufreq.c`.

**Who calls it:**
الـ governors زي `schedutil` و `ondemand` بيستدعوها في `governor_start()` أو ما يعادله عند بدء التشغيل على CPU معين.

---

#### `cpufreq_remove_util_hook`

```c
void cpufreq_remove_update_util_hook(int cpu);
```

**الـ function بتعمل إيه:**
بتمسح الـ hook المسجّل على الـ `cpu` المحدد بتصفير الـ per-CPU pointer. بعد الاستدعاء، أي utilization update للـ CPU ده هيُتجاهل من الـ cpufreq side.

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `cpu` | `int` | رقم الـ CPU اللي هيُزال منه الـ hook |

**Return value:** `void`

**Key details:**
- بيستدعيها الـ governor في `governor_stop()` أو عند CPU hotplug removal.
- مش بتعمل synchronize — الـ caller مسؤول إنه يضمن إن الـ callback مش شغالة في نفس الوقت (عادةً بيتعمل ده بـ `synchronize_rcu()` لو الـ access بيتم بـ RCU).
- لو مفيش hook مسجّل، الاستدعاء بيبقى no-op آمن.

**Who calls it:**
الـ governors عند الـ stop path، وكمان `cpufreq_governor_stop()` في الـ cpufreq core.

---

### Group 2: Runtime Query

#### `cpufreq_this_cpu_can_update`

```c
bool cpufreq_this_cpu_can_update(struct cpufreq_policy *policy);
```

**الـ function بتعمل إيه:**
بترجع `true` لو الـ CPU الحالي (اللي بتنفذ الكود عليه) مسموحله يبعت frequency update request للـ policy دي. ده بيمنع إن أكتر من CPU في نفس الـ policy يبعتوا requests متعارضة في نفس الوقت.

**Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `policy` | `struct cpufreq_policy *` | الـ policy اللي بنتحقق منها — بتشمل الـ CPUs المرتبطين ببعض بنفس الـ clock |

**Return value:** `bool`
- `true`: الـ CPU الحالي هو الـ "managing CPU" للـ policy دي، أو إن الـ policy بتسمح بـ updates من أي CPU مرتبط.
- `false`: الـ CPU الحالي مش المناسب يبعت الـ update دلوقتي.

**Key details:**
- الـ implementation بتتحقق إن `smp_processor_id()` هو `policy->cpu` أو إنه في الـ `policy->cpus` mask في حالة الـ shared policies.
- بتُستدعى من داخل الـ utilization callback قبل ما الـ governor يقرر يبعت request.
- ده **fast-path check** — مفيش locking، بس الـ caller لازم يكون في interrupt-disabled context أو preempt-disabled context عشان الـ `smp_processor_id()` يكون valid.

**Pseudocode:**
```c
bool cpufreq_this_cpu_can_update(struct cpufreq_policy *policy)
{
    /* either this CPU manages the policy alone,
       or it's one of the shared CPUs and it's the designated one */
    return cpumask_test_cpu(smp_processor_id(), policy->cpus);
}
```

**Who calls it:**
الـ `schedutil` governor من جوا الـ utilization callback قبل ما يعمل `cpufreq_driver_fast_switch()` أو يضيف work للـ irq_work.

---

### Group 3: Math Helpers

الـ helpers دول بتعمل الـ mapping بين مفاهيم الـ scheduler (utilization بالـ PELT scale) ومفاهيم الـ cpufreq (frequency بالـ kHz أو percentage).

---

#### `map_util_freq`

```c
static inline unsigned long map_util_freq(unsigned long util,
                                           unsigned long freq,
                                           unsigned long cap)
```

**الـ function بتعمل إيه:**
بتحسب الـ frequency المطلوبة بناءً على الـ utilization الحالي ومقارنته بالـ capacity الكاملة. المعادلة بسيطة: لو الـ CPU مشغول بـ 50% من capacity عند frequency معينة، إذن نحتاج نفس الـ 50% من الـ max frequency.

```
target_freq = freq * util / cap
```

**Parameters:**

| Parameter | الشرح |
|---|---|
| `util` | الـ utilization الحالي — بيجي من الـ PELT signal، scale بتاعه `cap` |
| `freq` | الـ maximum frequency للـ CPU (بالـ kHz عادةً) |
| `cap` | الـ maximum capacity — الـ scale بتاع الـ `util` (عادةً `arch_scale_cpu_capacity()`) |

**Return value:** `unsigned long` — الـ frequency المقترحة بنفس وحدة `freq`.

**Key details:**
- **مفيش overflow protection** — الـ caller مسؤول إن `freq * util` ما يعداش `ULONG_MAX`. في الممارسة، الـ util دايماً أصغر من `cap` فالـ result دايماً ≤ `freq`.
- العملية بتتم بالـ **integer division** — الـ result دايماً يتقرب لأدنى قيمة.
- بتُستدعى من الـ governor لحساب الـ target frequency قبل ما يختار من الـ frequency table.

**مثال عملي:**
```c
/* CPU بـ max freq = 2000 kHz, util = 768, cap = 1024 (SCHED_CAPACITY_SCALE) */
unsigned long target = map_util_freq(768, 2000, 1024);
/* target = 2000 * 768 / 1024 = 1500 kHz */
```

---

#### `map_util_perf`

```c
static inline unsigned long map_util_perf(unsigned long util)
```

**الـ function بتعمل إيه:**
بتضيف 25% فوق الـ utilization الحالي كـ **performance headroom**. الفكرة إن الـ PELT signal بيعكس الـ past utilization، بس الـ workload ممكن يرتفع فجأة. الـ 25% دي buffer عشان الـ governor يبقى proactive مش reactive.

```
result = util + (util >> 2)   /* = util * 1.25 */
```

**Parameters:**

| Parameter | الشرح |
|---|---|
| `util` | الـ utilization الخام من الـ scheduler |

**Return value:** `unsigned long` — الـ utilization بعد الـ boost.

**Key details:**
- الـ `util >> 2` = `util / 4` = 25% زيادة.
- لو `util = 1024` (=cap)، الـ result = `1280` — ده طبيعي لأن الـ governor بعدين بيعمل `min(result, cap)` أو بيختار الـ max frequency مباشرةً.
- **بتُستدعى قبل** `map_util_freq` في الـ governor pipeline عشان يضمن إن ما نقصش في الـ frequency.

**مثال:**
```c
unsigned long util = 600;
unsigned long boosted = map_util_perf(util);
/* boosted = 600 + 150 = 750 */

unsigned long freq = map_util_freq(boosted, max_freq, cap);
/* freq أعلى بـ 25% مما كانت ستكون بدون boost */
```

**الـ pipeline الكاملة في schedutil:**
```
PELT util
    │
    ▼
map_util_perf(util)          ← +25% headroom
    │
    ▼
map_util_freq(boosted, max_freq, cap)   ← convert to frequency
    │
    ▼
cpufreq_driver_fast_switch(policy, target_freq)
```

---

### Group 4: Data Type — `struct update_util_data`

```c
struct update_util_data {
    void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
};
```

**الـ struct بتعمل إيه:**
ده الـ **vtable-style container** اللي بيربط الـ governor callback بالـ scheduler hook infrastructure. الـ struct نفسه صغير جداً — function pointer واحد بس — لكن الـ governors بتعمله embed جوا struct أكبر وبتوصله بـ `container_of`.

**الـ idiomatic pattern:**
```c
/* مثال من schedutil */
struct sugov_cpu {
    struct update_util_data    update_util;  /* لازم يكون أول member */
    struct sugov_policy       *sg_policy;
    unsigned int               cpu;
    /* ... */
};

/* في الـ callback */
static void sugov_update_single(struct update_util_data *hook,
                                u64 time, unsigned int flags)
{
    struct sugov_cpu *sg_cpu = container_of(hook, struct sugov_cpu, update_util);
    /* الآن نقدر نوصل لكل state بتاع الـ governor */
}
```

---

### Group 5: Flags — `SCHED_CPUFREQ_IOWAIT`

```c
#define SCHED_CPUFREQ_IOWAIT    (1U << 0)
```

**الـ flag بيعمل إيه:**
بيتبعت في الـ `flags` parameter للـ callback لما الـ scheduler يلاحظ إن task صحي من I/O wait. ده signal مهم للـ governor إن الـ workload ممكن يكون CPU-bound بعد كده، فيزود الـ frequency استباقياً.

**من بيسته:**
الـ scheduler في `cpufreq_update_util()` بيحط الـ flag ده لو الـ wakeup كان من حالة I/O block.

**من بيستخدمه:**
الـ `schedutil` governor — لما بيشوف الـ flag ده، بيرفع الـ frequency بسرعة حتى لو الـ utilization الحالي لسه واطي.

```c
/* مثال من الـ governor */
if (flags & SCHED_CPUFREQ_IOWAIT)
    /* force boost even if util is low */
    util = max(util, sg_policy->iowait_boost);
```

---

### تدفق الاستدعاء الكامل — End-to-End Flow

```
[Scheduler — CFS/RT]
    │
    │  task wakes up or finishes
    ▼
cpufreq_update_util(rq, flags)
    │
    │  reads per-CPU hook pointer (RCU-protected)
    ▼
update_util_data->func(data, ktime_get_ns(), flags)
    │
    │  (this is the governor's registered callback)
    ▼
[Governor — e.g., schedutil]
    │
    ├─ cpufreq_this_cpu_can_update(policy)?  ──NO──► return
    │         │YES
    │         ▼
    ├─ util = map_util_perf(rq->util)
    │         │
    │         ▼
    ├─ freq  = map_util_freq(util, max_freq, cap)
    │         │
    │         ▼
    └─ cpufreq_driver_fast_switch(policy, freq)
           OR
       irq_work_queue → workqueue → cpufreq_driver_target()
```
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem ده هو **واجهة الـ cpufreq مع الـ scheduler** — بالتحديد الـ hook اللي بيخلي الـ scheduler يبلغ الـ cpufreq governor بالـ utilization updates في real-time. الـ debugging بيتمحور حول ٣ محاور: تسجيل الـ hooks، دقة الـ util، وتوقيت الـ callbacks.

---

### Software Level

#### 1. debugfs Entries

الـ `debugfs` بيعرض معلومات الـ cpufreq governor اللي بيستخدم الـ hook ده.

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# شوف الـ schedutil governor debug info
ls /sys/kernel/debug/sched/

# الـ entry الأهم — util لكل CPU
cat /sys/kernel/debug/sched/debug

# شوف الـ PELT (Per-Entity Load Tracking) لكل CPU
cat /sys/kernel/debug/sched/sched_pelt_tau
```

**قراءة output الـ sched debug:**
```
cpu#0, 2800.000 MHz
  .nr_running                    : 1
  .util_avg                      : 512        # 50% utilization من 1024
  .util_est                      : 600
```

الـ `util_avg` هو اللي بيتبعت للـ `update_util_data->func()` callback.

#### 2. sysfs Entries

```bash
# الـ governor الحالي لكل CPU
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# الـ frequency الحالية
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# الـ rate_limit_us للـ schedutil (throttle على الـ callbacks)
cat /sys/devices/system/cpu/cpu0/cpufreq/schedutil/rate_limit_us

# الـ iowait boost
cat /sys/devices/system/cpu/cpu0/cpufreq/schedutil/iowait_boost_min

# إحصائيات الـ transitions
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans
```

**الـ `rate_limit_us`** مهم جداً — لو كبير معناه الـ callbacks بتتجاهل كتير.

#### 3. ftrace — Tracepoints والـ Events

```bash
# enable الـ cpufreq events
echo 1 > /sys/kernel/debug/tracing/events/power/cpu_frequency/enable
echo 1 > /sys/kernel/debug/tracing/events/power/cpu_frequency_limits/enable

# enable الـ sched util events
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_util_est_cfs/enable
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_util_est_se/enable

# تتبع الـ schedutil update function نفسها
echo 'p:cpufreq_hook cpufreq_add_update_util_hook' > /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/cpufreq_hook/enable

# شغل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اشتغل شوية ثم اقرأ
cat /sys/kernel/debug/tracing/trace | head -50

# وقف
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**تتبع الـ map_util_freq بـ function_graph:**
```bash
echo 'schedutil_freq_to_util' > /sys/kernel/debug/tracing/set_graph_function
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 2
cat /sys/kernel/debug/tracing/trace
```

**مثال output:**
```
 1)   0.312 us    |  cpufreq_driver_resolve_freq();
 1)   0.124 us    |  map_util_freq();   /* util=600 freq=2800000 cap=1024 */
```

#### 4. printk والـ Dynamic Debug

```bash
# تفعيل dynamic debug للـ cpufreq subsystem
echo 'file drivers/cpufreq/*.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/sched/cpufreq*.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل للـ schedutil governor
echo 'file kernel/sched/cpufreq_schedutil.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ log
dmesg -w | grep -E 'cpufreq|schedutil|util'

# تفعيل verbose للـ PELT
echo 'file kernel/sched/pelt.c +p' > /sys/kernel/debug/dynamic_debug/control
```

**في الكود مباشرة** لو محتاج تضيف debug مؤقت:
```c
/* في schedutil أو أي governor بيستخدم update_util_data */
void my_governor_update(struct update_util_data *data,
                        u64 time, unsigned int flags)
{
    unsigned long util = /* ... */;
    unsigned long freq = map_util_freq(util, max_freq, capacity);

    /* debug: اطبع كل update */
    pr_debug_ratelimited("cpu%d util=%lu freq=%lu flags=%u iowait=%d\n",
                         cpu, util, freq, flags,
                         !!(flags & SCHED_CPUFREQ_IOWAIT));
}
```

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_CPU_FREQ_DEBUG` | يضيف debug printks في cpufreq core |
| `CONFIG_SCHED_DEBUG` | يفعل `/proc/sched_debug` و debugfs entries |
| `CONFIG_DEBUG_KERNEL` | يفعل كل الـ debug infrastructure |
| `CONFIG_LOCKDEP` | يكشف lock ordering bugs في الـ hook registration |
| `CONFIG_PROVE_LOCKING` | يثبت correctness الـ spinlocks في الـ per-cpu hooks |
| `CONFIG_FUNCTION_TRACER` | يفعل ftrace للـ function tracing |
| `CONFIG_KPROBES` | يفعل dynamic probing على الـ functions |
| `CONFIG_TRACEPOINTS` | يفعل الـ static tracepoints |
| `CONFIG_CPU_FREQ_STAT` | يضيف `/sys/.../stats/` entries |
| `CONFIG_CPU_FREQ_GOV_SCHEDUTIL` | يفعل الـ schedutil governor |

```bash
# تحقق إن الـ configs مفعلة
grep -E 'CONFIG_(CPU_FREQ|SCHED)_DEBUG|CONFIG_LOCKDEP' /proc/config.gz | zcat
# أو
zcat /proc/config.gz | grep -E 'CPU_FREQ|SCHED_DEBUG'
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# cpupower — أقوى أداة لـ cpufreq
cpupower frequency-info
cpupower monitor -m Mperf   # real hardware frequency
cpupower idle-info

# turbostat — قراءة الـ frequency الحقيقية من MSRs
turbostat --interval 1 --show CPU,Avg_MHz,Busy%,Bzy_MHz

# perf لقياس الـ frequency scaling events
perf stat -e power:cpu_frequency -a sleep 5

# perf trace لتتبع الـ syscalls + frequency changes
perf trace -e power:* sleep 5

# stress-ng لاختبار الـ governor response
stress-ng --cpu 4 --timeout 10s &
watch -n 0.5 cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
```

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|----------------------|--------|-------|
| `cpufreq: __cpufreq_add_dev: ->get() failed` | الـ driver مش قادر يقرأ الـ frequency الحالية | تحقق من الـ cpufreq driver init |
| `cpufreq: governor change failed` | فشل تغيير الـ governor لـ schedutil | تأكد `CONFIG_CPU_FREQ_GOV_SCHEDUTIL=y` |
| `WARNING: CPU: X PID: Y at kernel/sched/cpufreq.c` | الـ hook اتسجل مرتين على نفس الـ CPU | `cpufreq_remove_update_util_hook()` قبل الـ add |
| `BUG: spinlock lockup` في cpufreq | الـ callback بياخد وقت طويل | راجع الـ governor callback — ممنوع يـ sleep |
| `cpufreq: Failed to register notifier` | مشكلة في الـ notifier chain | راجع الـ initialization order |
| `util out of range` (custom WARN) | `util > cap` — حالة غير متوقعة | راجع حساب الـ PELT، قد يكون CPU hotplug race |
| `schedutil: CPU X missed deadline` | الـ rate_limit_us كبير جداً | قلل `rate_limit_us` في sysfs |

#### 8. نقاط استراتيجية لـ dump_stack() وـ WARN_ON()

```c
/* في cpufreq_add_update_util_hook — تحقق من double registration */
void cpufreq_add_update_util_hook(int cpu, struct update_util_data *data,
    void (*func)(struct update_util_data *data, u64 time, unsigned int flags))
{
    /* WARN لو في hook موجود بالفعل */
    WARN_ON(per_cpu(cpufreq_update_util_data, cpu) != NULL);

    data->func = func;
    rcu_assign_pointer(per_cpu(cpufreq_update_util_data, cpu), data);
}

/* في map_util_freq — تحقق من cap != 0 */
static inline unsigned long map_util_freq(unsigned long util,
                                          unsigned long freq,
                                          unsigned long cap)
{
    WARN_ON_ONCE(cap == 0);      /* division by zero protection */
    WARN_ON_ONCE(util > cap);    /* util يفوق الـ capacity */
    return freq * util / cap;
}

/* في map_util_perf — تحقق من overflow */
static inline unsigned long map_util_perf(unsigned long util)
{
    /* util + util/4 — لو util قريب من ULONG_MAX هيحصل overflow */
    WARN_ON_ONCE(util > (ULONG_MAX - (util >> 2)));
    return util + (util >> 2);
}

/* في الـ governor callback — تحقق من الـ IOWAIT flag */
void my_governor_update(struct update_util_data *data,
                        u64 time, unsigned int flags)
{
    /* لو iowait والـ utilization صفر — حالة مريبة */
    WARN_ON_ONCE((flags & SCHED_CPUFREQ_IOWAIT) && util == 0);
}
```

---

### Hardware Level

#### 1. التحقق من مطابقة الـ Hardware State للـ Kernel State

الفكرة: الـ kernel بيحسب frequency مطلوبة، لكن هل الـ hardware نفذها فعلاً؟

```bash
# قارن الـ requested frequency بالـ actual
# الـ requested (kernel)
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# الـ actual من hardware (عبر MSR أو perf)
turbostat --interval 1 --show CPU,Avg_MHz,Bzy_MHz,Busy% --num_iterations 3

# لو الـ Bzy_MHz != scaling_cur_freq → مشكلة hardware
# أسباب شائعة: thermal throttling، power limits، firmware override
```

#### 2. Register Dump Techniques

```bash
# قراءة MSR مباشرة (Intel)
# MSR_IA32_PERF_STATUS = 0x198 → الـ frequency الحالية
rdmsr -a 0x198   # على كل الـ CPUs

# MSR_IA32_PERF_CTL = 0x199 → الـ requested P-state
rdmsr -a 0x199

# تحويل القيمة: bits[15:8] = ratio → multiply by bus_clock (100MHz)
# مثال: 0x1E00 → ratio=30 → 3000 MHz

# AMD: MSRC001_0063 → P-state status
rdmsr -a 0xC0010063

# لو rdmsr مش موجود
modprobe msr
apt install msr-tools  # أو yum install msr-tools

# عبر /dev/cpu/N/msr
python3 -c "
import struct
with open('/dev/cpu/0/msr', 'rb') as f:
    f.seek(0x198)
    val = struct.unpack('Q', f.read(8))[0]
    print(f'P-state current: {hex(val)}, freq: {(val >> 8) & 0xff}00 MHz')
"

# devmem2 للـ MMIO registers (platform-specific)
# مثال على Rockchip: قراءة الـ CRU (Clock & Reset Unit)
devmem2 0xFF760000 w  # base address الـ CRU على RK3399
```

#### 3. Logic Analyzer / Oscilloscope Tips

لتتبع تغييرات الـ frequency على الـ hardware:

```
مشكلة: كيف تعرف إن الـ CPU غير الـ frequency فعلاً؟

الحل ١: استخدم GPIO كـ marker
  - في بداية الـ governor callback: اعمل GPIO high
  - في النهاية: GPIO low
  - قيس بالـ oscilloscope → هتشوف latency التنفيذ

الحل ٢: راقب Vcore voltage rail
  - لما الـ frequency بتتغير، الـ voltage بيتغير معاها (DVFS)
  - استخدم differential probe على الـ Vcore
  - تغيير الـ voltage = confirmation إن الـ P-state اتغير

الحل ٣: Power rail monitoring
  - INA226/INA3221 على الـ VDD_CPU rail
  - لما الـ utilization يرتفع → الـ power يرتفع → الـ freq بتعلى
```

#### 4. Hardware Issues وـ Kernel Log Patterns

| المشكلة | الـ Kernel Log Pattern | السبب |
|---------|----------------------|-------|
| Thermal throttling | `thermal thermal_zone0: cpu0 is above trip temp` | الـ CPU وصل للـ thermal limit، بيتجاهل طلبات رفع الـ freq |
| Power capping | `RAPL: power limit exceeded` | الـ BIOS/firmware بيحد الـ power → الـ freq بتنزل |
| P-state limits من BIOS | `cpufreq: policy min/max clamped` | الـ BIOS قيّد الـ P-states المتاحة |
| IRQ affinity conflicts | `irq N: no longer affine to CPU X` | الـ CPU اتغير load بسبب IRQ rebalancing |
| Broken ACPI _PSS table | `ACPI: processor: Failed to get throttling info` | جدول الـ P-states في الـ firmware غلط |
| Turbo disabled | `CPU X: No turbo` | الـ turbo boost اتعطل من BIOS أو kernel param |

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT nodes المرتبطة بالـ cpufreq
# الـ operating-points-v2 هو اللي بيحدد الـ P-states
dtc -I fs /sys/firmware/devicetree/base | grep -A 20 'operating-points'

# شوف الـ OPP table للـ CPU
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies

# تحقق من الـ DT cpu node
ls /sys/firmware/devicetree/base/cpus/cpu@0/

# الـ properties المهمة في الـ DT
# compatible = "operating-points-v2"
# opp-hz = الـ frequency
# opp-microvolt = الـ voltage المقابل

# لو الـ OPP table ناقصة frequencies
dmesg | grep -i 'opp\|operating'

# مقارنة الـ DT frequencies بالـ hardware actual limits
diff <(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies | tr ' ' '\n' | sort -n) \
     <(dtc -I fs /sys/firmware/devicetree/base | grep 'opp-hz' | awk '{print $3}' | sort -n)
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

**1. تشخيص سريع شامل:**
```bash
#!/bin/bash
echo "=== CPUFreq Governor ==="
for cpu in /sys/devices/system/cpu/cpu[0-9]*/cpufreq/scaling_governor; do
    echo "$cpu: $(cat $cpu)"
done

echo ""
echo "=== Current Frequencies ==="
for cpu in /sys/devices/system/cpu/cpu[0-9]*/cpufreq/scaling_cur_freq; do
    echo "$cpu: $(cat $cpu) kHz"
done

echo ""
echo "=== Schedutil Rate Limit ==="
cat /sys/devices/system/cpu/cpu0/cpufreq/schedutil/rate_limit_us 2>/dev/null || echo "Not using schedutil"

echo ""
echo "=== Frequency Transitions ==="
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans 2>/dev/null
```

**2. مراقبة الـ util callbacks في real-time:**
```bash
# تشغيل trace للـ cpu_frequency events
trace-cmd record -e power:cpu_frequency -e sched:sched_util_est_cfs \
    -p function_graph -g schedutil_update_single \
    stress-ng --cpu 2 --timeout 5s

trace-cmd report | head -100
```

**3. قياس latency الـ governor response:**
```bash
# قياس الوقت من رفع الـ util لحد تغيير الـ frequency
perf record -e power:cpu_frequency -a -- stress-ng --cpu 1 --timeout 10s
perf script | awk '/cpu_frequency/{print $4, $NF}' | head -20
```

**4. اختبار الـ IOWAIT flag:**
```bash
# أنشئ I/O load واراقب الـ frequency boost
dd if=/dev/zero of=/tmp/test bs=1M count=1000 &
watch -n 0.2 'cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq'
```

**5. تفعيل كل الـ debug دفعة واحدة:**
```bash
#!/bin/bash
# Enable all cpufreq+sched debug events
for event in power/cpu_frequency power/cpu_frequency_limits \
             sched/sched_util_est_cfs sched/sched_util_est_se; do
    echo 1 > /sys/kernel/debug/tracing/events/${event}/enable
    echo "Enabled: $event"
done

echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "Tracing started. Press Enter to stop..."
read

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace > /tmp/cpufreq_trace_$(date +%s).txt
echo "Trace saved."
```

**مثال output مفسَّر:**
```
# الـ trace output
          stress-2345  [002] d..2  1234.567890: cpu_frequency: state=2800000 cpu_id=2
          stress-2345  [002] d..2  1234.567920: cpu_frequency: state=3200000 cpu_id=2
          <idle>-0     [002] d..2  1234.600000: cpu_frequency: state=400000 cpu_id=2

# التفسير:
# السطر 1: CPU2 كان على 2.8GHz
# السطر 2: بعد 30μs رفع لـ 3.2GHz (governor استجاب للـ util ارتفع)
# السطر 3: بعد 32ms الـ CPU خلص شغل → governor نزّل لـ 400MHz (idle)
# الـ 30μs هو latency الـ governor callback → قيمة طبيعية جداً
```

**6. فحص مشاكل الـ map_util_perf:**
```bash
# احسب يدوياً النتيجة المتوقعة وقارنها
python3 -c "
util = int(open('/sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq').read())
# map_util_perf: util + util/4
perf = util + (util >> 2)
print(f'util={util}, map_util_perf={perf} (25% boost applied)')
"
```

**7. تحقق من الـ hook registration:**
```bash
# شوف لو في hook مسجل على الـ CPUs
# يتطلب kernel بـ KPROBES
echo 'p:probe_add cpufreq_add_update_util_hook cpu=%di data=%si' \
    > /sys/kernel/debug/tracing/kprobe_events
echo 'p:probe_remove cpufreq_remove_update_util_hook cpu=%di' \
    >> /sys/kernel/debug/tracing/kprobe_events

echo 1 > /sys/kernel/debug/tracing/events/kprobes/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغل governor switch
echo schedutil > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

cat /sys/kernel/debug/tracing/trace | grep -E 'probe_add|probe_remove'
```

**مثال output:**
```
kworker/0:1-23  [000] ....  100.123: probe_add: (cpufreq_add_update_util_hook+0x0)
                                     cpu=0x0 data=0xffff888012345678
# → الـ hook اتسجل على CPU 0 بـ data pointer صحيح
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ governor مش بيتحدث والـ CPU واقف على أقل تردد

#### العنوان
**الـ schedutil governor مش شايل الـ hook على RK3562 بعد custom driver تم تحميله**

#### السياق
شركة بتبني industrial IoT gateway على Rockchip RK3562. المنتج بيشتغل Linux 6.6 مع `schedutil` كـ CPUfreq governor. الـ board بتشغّل Modbus TCP stack وبتجمع بيانات من 64 sensor عبر RS-485. العميل بيشتكي إن الـ gateway بطيء جداً وإن الـ CPU دايماً شايل تردد 408 MHz (الـ minimum) حتى لو الـ load عالي.

#### المشكلة
الـ engineer راح يتحقق:

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# Output: 408000  ← دايماً minimum!

cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
# Output: schedutil
```

الـ `schedutil` مش بيرفع التردد رغم إن الـ CPU load عالي.

#### التحليل
الـ `schedutil` governor بيشتغل عن طريق الـ hook اللي بيتسجّل بـ `cpufreq_add_update_util_hook()`. الـ hook ده بيتشغّل كل ما الـ scheduler يحسب utilization جديد ويحتاج يقرر هل يرفع التردد أو لأ.

```c
/* من sched/cpufreq.h — هي دي نقطة الدخول */
struct update_util_data {
    void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
};

void cpufreq_add_update_util_hook(int cpu, struct update_util_data *data,
                   void (*func)(struct update_util_data *data, u64 time,
                                unsigned int flags));
```

الـ engineer اكتشف إن custom out-of-tree driver للـ RS-485 controller كان بيعمل `cpufreq_remove_update_util_hook(cpu)` في الـ `probe()` بشكل خاطئ — ظناً منه إنه بيعمل cleanup لـ hook قديم. لكن ده خلى الـ schedutil hook اتشال قبل ما يتسجل من أصله.

```c
/* الكود الخاطئ في الـ RS-485 driver */
static int rs485_probe(struct platform_device *pdev)
{
    int cpu = smp_processor_id();

    /* BUG: بيشيل الـ hook بدون ما يكون هو اللي سجّله */
    cpufreq_remove_update_util_hook(cpu);

    /* ... باقي الـ init ... */
    return 0;
}
```

#### الحل

```bash
# خطوة 1: تتبع الـ hook باستخدام ftrace
echo 'cpufreq_add_update_util_hook' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'cpufreq_remove_update_util_hook' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# تحميل الـ driver ثم راقب
modprobe rs485_custom
cat /sys/kernel/debug/tracing/trace
```

الحل: حذف الـ `cpufreq_remove_update_util_hook()` من `rs485_probe()` لأنه مش مسؤول عنه.

```c
/* الكود المصحح */
static int rs485_probe(struct platform_device *pdev)
{
    /* لا تشيل hook مش أنت اللي سجّلته */
    return rs485_hw_init(pdev);
}
```

#### الدرس المستفاد
الـ `cpufreq_remove_update_util_hook()` مش "safe cleanup" — هو بيشيل أي hook موجود على الـ CPU بغض النظر عن مين سجّله. driver ما يستخدمش الـ hook ما يبقاش له دعوة بيه خالص.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ UI متقطع والـ I/O بيوقف الـ DVFS

#### العنوان
**الـ SCHED_CPUFREQ_IOWAIT flag بيخلي الـ CPU يقفز لأعلى تردد بشكل زائد على H616**

#### السياق
Android TV box بيشتغل Allwinner H616 مع Android 13. الـ UI بتستخدم `schedutil`. المنتج بيعمل playback لـ 4K HEVC. المستخدمين بيشتكوا من stuttering في الـ UI بيحصل بشكل دوري حتى لو الـ CPU load منخفض.

#### المشكلة
```bash
# مراقبة التردد
watch -n 0.1 cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# التردد بيقفز من 480 MHz لـ 1800 MHz وبيرجع بشكل سريع جداً
```

الـ thermal throttling مش هو السبب. الـ GPU load طبيعي. بس الـ CPU بيـ spike بشكل مستمر.

#### التحليل
الـ `SCHED_CPUFREQ_IOWAIT` flag معرّف في الملف:

```c
#define SCHED_CPUFREQ_IOWAIT    (1U << 0)
```

الـ scheduler بيبعت الـ flag ده في `flags` parameter لما task بتخرج من I/O wait. الـ `schedutil` governor لما يشوف الـ flag ده بيطلب أعلى تردد فوراً لأنه بيفترض إن في burst of work قادم.

```c
/* الـ func في update_util_data بتستقبل الـ flags */
void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
/*                                                            ^^^^^^^
 * لو SCHED_CPUFREQ_IOWAIT معموله set، الـ governor بيعمل
 * boost فوري للتردد
 */
```

الـ 4K decoder كان بيعمل I/O requests كتير جداً (reading chunks من الـ eMMC) وكل request بتخلي الـ CPU يـ boost عشان يستقبل الـ data — رغم إن الـ decoding نفسه على الـ VPU مش محتاج CPU load عالي.

```bash
# تأكيد المشكلة
perf stat -e sched:sched_stat_iowait -p $(pgrep mediaserver) sleep 5
# هتلاقي iowait events كتير جداً
```

#### الحل

```bash
# خيار 1: ضبط الـ iowait boost في schedutil
echo 0 > /sys/devices/system/cpu/cpufreq/schedutil/iowait_boost_enable
# مش متاح في كل kernel versions

# خيار 2: تعديل الـ I/O pattern في الـ media pipeline
# بدل ما يعمل reads صغيرة كتير، يعمل buffered reads أكبر
# ده بيقلل عدد iowait events
```

أو تعديل الـ governor لإضافة threshold:

```c
/* في kernel/sched/cpufreq_schedutil.c */
static void sugov_iowait_boost(struct sugov_cpu *sg_cpu, u64 time,
                                unsigned int flags)
{
    /* تجاهل الـ iowait boost لو الـ util أقل من threshold */
    if (sg_cpu->util < sg_cpu->iowait_boost_min_util)
        return;
    /* ... */
}
```

#### الدرس المستفاد
**الـ `SCHED_CPUFREQ_IOWAIT`** flag مصمم لـ latency-sensitive tasks. لو الـ I/O intensive workload مش محتاج CPU boost (زي VPU-based decoding)، لازم تضبط الـ I/O access pattern أو تتحكم في الـ iowait boost behavior بدل ما تسيبه يأثر على الـ thermal و battery life.

---

### السيناريو 3: Automotive ECU على i.MX8QM — الـ map_util_freq بيدي تردد خاطئ بعد OPP table تعديل

#### العنوان
**الـ `map_util_freq()` بيحسب target frequency غلط بعد تعديل الـ OPP table على i.MX8QM**

#### السياق
Automotive ECU بيشغّل i.MX8QM في سيارة كهربائية. الـ ECU بيشغّل AUTOSAR stack فوق Linux. الـ SoC عنده Cortex-A53 cluster بـ OPP table اتعدّل عشان يدعم voltage binning (بعض الـ chips بتشتغل على أعلى تردد). بعد التعديل، الـ system بيخلي الـ CPU يشتغل على 1.2 GHz بدل الـ 1.6 GHz المتوقع في الـ full load.

#### المشكلة
```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq
# 1600000  ← صح

cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
# 1600000  ← صح

cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 1200000  ← غلط! المفروض يكون 1600000 في full load
```

#### التحليل
الـ `schedutil` بيستخدم `map_util_freq()` لتحويل الـ CPU utilization لـ target frequency:

```c
static inline unsigned long map_util_freq(unsigned long util,
                                          unsigned long freq,
                                          unsigned long cap)
{
    return freq * util / cap;
    /*
     * freq = current max freq من الـ OPP table
     * util = CPU utilization (0 to cap)
     * cap  = CPU capacity (عادةً 1024 على ARM)
     */
}
```

المشكلة: بعد تعديل الـ OPP table، الـ `freq` parameter اللي بيتبعت لـ `map_util_freq()` كان لسه بيجيب القيمة من الـ `policy->cpuinfo.max_freq` اللي ما اتحدثتش صح بعد إضافة الـ OPP entries الجديدة.

```bash
# تحقق من الـ OPP table
cat /sys/bus/cpu/devices/cpu0/opp/opp-microvolt
# هتلاقي الـ 1.6 GHz entry موجودة بس الـ cpuinfo.max_freq مش بتعكسها
```

السبب: الـ DT OPP table اتعدّلت لكن الـ `opp-supported-hw` mask مش صح، فالـ kernel بيتجاهل الـ 1.6 GHz entry لبعض الـ chip variants.

```dts
/* الـ DT الخاطئ */
opp-1600000000 {
    opp-hz = /bits/ 64 <1600000000>;
    opp-microvolt = <1000000>;
    opp-supported-hw = <0x02>; /* بس variant 2 — لكن الـ chip ده variant 1 */
};
```

#### الحل

```dts
/* الـ DT المصحح — دعم كل الـ variants */
opp-1600000000 {
    opp-hz = /bits/ 64 <1600000000>;
    opp-microvolt = <1000000>;
    opp-supported-hw = <0x03>; /* variant 1 و 2 */
};
```

```bash
# تحقق بعد الإصلاح
dmesg | grep opp
# [    2.134] cpu0: Found 6 OPPs  ← المفروض يظهر كل الـ OPPs

cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
# 408000 600000 816000 1104000 1200000 1600000  ← الـ 1.6G ظهر
```

#### الدرس المستفاد
الـ `map_util_freq()` بسيطة (`freq * util / cap`) لكن صحتها بتعتمد 100% على الـ `freq` اللي بتستقبلها. لو الـ OPP table مش complete بسبب `opp-supported-hw` خاطئ، الـ governor هيحسب target frequency غلط حتى لو الكود نفسه سليم.

---

### السيناريو 4: IoT Sensor Hub على STM32MP1 — الـ cpufreq_this_cpu_can_update() بيسبب race condition

#### العنوان
**crash في الـ schedutil على STM32MP1 عند CPU hotplug أثناء sensor data burst**

#### السياق
IoT sensor hub بيشتغل STM32MP157 (dual Cortex-A7). الـ product بيجمع بيانات من 200 sensor عبر I2C و SPI. عشان توفير الطاقة، الـ CPU1 بيتعمله hotplug-out لما الـ load منخفض. في لحظات الـ data burst، بيحصل kernel panic متقطع.

#### المشكلة
```
BUG: kernel NULL pointer dereference at 0x00000010
PC is at sugov_update_shared+0x48/0x120
```

الـ crash بيحصل في `sugov_update_shared` اللي بتستدعي `cpufreq_this_cpu_can_update()`.

#### التحليل
الـ `cpufreq_this_cpu_can_update()` موجودة كـ declaration في الملف:

```c
bool cpufreq_this_cpu_can_update(struct cpufreq_policy *policy);
```

الـ implementation في `drivers/cpufreq/cpufreq.c` بتتحقق هل الـ CPU الحالي مؤهل يعمل update للـ policy:

```c
/* Implementation المرتبطة */
bool cpufreq_this_cpu_can_update(struct cpufreq_policy *policy)
{
    return cpumask_test_cpu(smp_processor_id(), policy->cpus) &&
           (policy->governor->flags & CPUFREQ_GOV_DYNAMIC_SWITCHING);
}
```

الـ race condition: لما CPU1 بيتعمله hotplug-out، الـ scheduler على CPU0 ممكن يكون في منتصف الـ `sugov_update_shared()` وبيمسك الـ `policy->cpus` mask قبل ما CPU1 يتشال منه. في نفس الوقت، الـ hotplug code بيعمل `cpufreq_remove_update_util_hook(1)` على CPU1.

```
CPU0 (schedutil update)          CPU1 (hotplug-out)
-------------------------------- ---------------------------
sugov_update_shared()
  read policy->cpus (CPU0+CPU1)
                                 cpufreq_remove_util_hook(1)
  cpufreq_this_cpu_can_update()  policy->cpus = {CPU0 only}
  → panic: stale pointer
```

#### الحل

```bash
# تأكيد الـ race
echo 1 > /sys/kernel/debug/prove_locking
# ثم تشغيل الـ workload + hotplug

# الـ fix في kernel: التأكد من الـ locking في sugov_update_shared
# الـ policy->rwsem لازم يتمسك قبل check الـ cpumask
```

الـ fix الصح: استخدام `cpufreq_this_cpu_can_update()` مع الـ proper locking:

```c
/* في sugov_update_shared - الكود المصحح conceptually */
static void sugov_update_shared(struct update_util_data *hook,
                                 u64 time, unsigned int flags)
{
    struct sugov_cpu *sg_cpu = container_of(hook, struct sugov_cpu, update_util);
    struct sugov_policy *sg_policy = sg_cpu->sg_policy;

    raw_spin_lock(&sg_policy->update_lock); /* لازم يكون قبل الـ check */

    if (!cpufreq_this_cpu_can_update(sg_policy->policy)) {
        raw_spin_unlock(&sg_policy->update_lock);
        return;
    }
    /* ... */
}
```

#### الدرس المستفاد
الـ `cpufreq_this_cpu_can_update()` مصممة تتُستدعى **تحت الـ appropriate lock**. استخدامها بدون الـ `update_lock` في بيئة multi-CPU مع hotplug ممكن يسبب race condition قاتل. على الـ STM32MP1 (dual-core صغير) المشكلة أسهل تتكرر لأن الـ hotplug أسرع.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ map_util_perf() بيدي boost زيادة عن اللازم

#### العنوان
**الـ `map_util_perf()` بيخلي الـ AM62x يشتغل على max freq دايماً في الـ bring-up phase**

#### السياق
engineer بيعمل bring-up لـ custom board على Texas Instruments AM625 (quad Cortex-A53). الـ board مخصصة لـ HMI industrial display. في مرحلة الـ bring-up، الـ engineer شايل إن الـ CPU دايماً على أعلى تردد (1.4 GHz) حتى لو الـ terminal فاضي وما فيش أي workload. ده بيخلي الـ board تسخن وبيصعّب قياسات الـ power.

#### المشكلة
```bash
watch -n 1 cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# 1400000  ← دايماً maximum رغم إن ما فيش حاجة شغّالة

cpupower frequency-info
# current policy: frequency should be within 200 MHz and 1400 MHz
# current CPU frequency: 1.40 GHz (asserted by call to hardware)
```

#### التحليل
الـ `map_util_perf()` موجودة في الملف:

```c
static inline unsigned long map_util_perf(unsigned long util)
{
    return util + (util >> 2);
    /*
     * بيضيف 25% فوق الـ utilization المحسوبة
     * هدفها: margin عشان الـ CPU يوصل للتردد المطلوب قبل ما يحتاجه
     */
}
```

الـ `schedutil` بيستخدمها قبل `map_util_freq()`:

```
util_adjusted = map_util_perf(util)     → util + 25%
target_freq   = map_util_freq(util_adjusted, max_freq, cap)
```

في الـ AM62x bring-up، اكتشف الـ engineer إن الـ CPU capacity (`cap`) اتحسبت غلط بسبب `capacity-dmips-mhz` مش موجودة في الـ DT:

```dts
/* الـ DT الناقص */
cpu0: cpu@0 {
    compatible = "arm,cortex-a53";
    /* capacity-dmips-mhz مش موجودة! */
};
```

لما الـ `cap` بيبقى خاطئ (أصغر من اللازم)، الـ `map_util_freq()` بتحسب:

```
target_freq = max_freq * util_adjusted / cap
            = 1400000 * 270 / 256   ← cap صغير → النتيجة أكبر من max_freq
            → يتكلم على max_freq دايماً
```

```bash
# تحقق من الـ CPU capacity
cat /sys/devices/system/cpu/cpu0/cpu_capacity
# 1024  ← المفروض يكون 1024 على Cortex-A53 symmetric

# لو الـ DT مش حاطط capacity-dmips-mhz، الـ kernel بيحسبها بشكل مختلف
dmesg | grep capacity
```

#### الحل

```dts
/* الـ DT المصحح */
cpu0: cpu@0 {
    compatible = "arm,cortex-a53";
    device_type = "cpu";
    reg = <0x000>;
    capacity-dmips-mhz = <512>; /* من الـ TRM أو قيس بـ Dhrystone */
};

cpu1: cpu@1 {
    compatible = "arm,cortex-a53";
    device_type = "cpu";
    reg = <0x001>;
    capacity-dmips-mhz = <512>;
};
```

```bash
# تحقق بعد الإصلاح
cat /sys/devices/system/cpu/cpu0/cpu_capacity
# 1024

# اختبار الـ frequency scaling
stress-ng --cpu 1 --timeout 10 &
watch -n 0.5 cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
# هتشوف التردد بيتغير حسب الـ load
```

#### الدرس المستفاد
الـ `map_util_perf()` و `map_util_freq()` مع بعض بيعتمدوا على الـ `cap` (CPU capacity) اللي بيجي من الـ scheduler. لو الـ DT مش حاطط `capacity-dmips-mhz` صح، الـ capacity calculation هتبقى غلط وهتخلي الـ DVFS يشتغل على max frequency دايماً — خصوصاً في الـ bring-up لما الـ board configuration لسه مش اكتملت. دايماً تحقق من الـ `cpu_capacity` sysfs أول خطوة في debug أي مشكلة DVFS.
## Phase 7: مصادر ومراجع

### توثيق الكيرنل الرسمي

| المسار | الوصف |
|--------|-------|
| `Documentation/admin-guide/pm/cpufreq.rst` | شرح كامل لـ CPU Performance Scaling وكيفية عمل governors |
| `Documentation/scheduler/schedutil.rst` | توثيق schedutil governor والعلاقة مع الـ scheduler |
| `Documentation/cpu-freq/` | مجلد كامل بكل وثائق cpufreq القديمة والحديثة |
| `include/linux/sched/cpufreq.h` | الـ header الرئيسي للـ interface بين الـ scheduler وـ cpufreq |
| `kernel/sched/cpufreq.c` | تنفيذ `cpufreq_add_update_util_hook` و`cpufreq_remove_update_util_hook` |
| `kernel/sched/cpufreq_schedutil.c` | تنفيذ schedutil governor كاملاً |

الـ [Schedutil official kernel docs](https://docs.kernel.org/scheduler/schedutil.html) هو أهم مرجع رسمي.

---

### مقالات LWN.net

دي أهم المقالات اللي بتشرح تطور الـ integration بين الـ cpufreq والـ scheduler:

| المقال | الأهمية |
|--------|---------|
| [Scheduler-driven CPU frequency selection (2015)](https://lwn.net/Articles/667281/) | البداية — أول proposal لربط الـ scheduler بالـ cpufreq مباشرة |
| [Teaching the scheduler about power management (2014)](https://lwn.net/Articles/602479/) | شرح ليه الـ cpufreq القديم كان بيحاول يحسب الـ load بطريقة غير مباشرة |
| [cpufreq: schedutil governor (2016)](https://lwn.net/Articles/678777/) | إعلان إضافة schedutil لأول مرة في kernel 4.7 |
| [schedutil enhancements (2016)](https://lwn.net/Articles/679936/) | تفاصيل تحسينات schedutil بعد أول إصدار |
| [Improvements in CPU frequency management (2016)](https://lwn.net/Articles/682391/) | نظرة شاملة على كل التحسينات في 4.7 |
| [CPU frequency governors and remote callbacks (2017)](https://lwn.net/Articles/732740/) | شرح كيف schedutil بقى يعمل remote wakeup callbacks |
| [arm/arm64: frequency- and cpu-invariant accounting (2017)](https://lwn.net/Articles/732028/) | الـ frequency-invariant accounting اللي بيأثر على `map_util_freq` |
| [cpufreq: schedutil: Allow remote wakeups (2017)](https://lwn.net/Articles/716648/) | تفاصيل تقنية لـ remote CPU updates |
| [sched: Consolidate cpufreq updates (2023)](https://lwn.net/Articles/972506/) | أحدث تحسينات في الـ integration |

---

### Commits مهمة في الكيرنل

**الـ commits دي هي اللي أنشأت أو غيرت الـ interface في `sched/cpufreq.h` بشكل مباشر:**

| الـ commit | الوصف |
|-----------|-------|
| [`9bdcb44`](https://github.com/torvalds/linux/commit/9bdcb44e391da5c41b98573bf0305a0e0b1c9569) | إضافة schedutil governor لأول مرة — أول استخدام لـ `update_util_data` |
| [schedutil governor source](https://github.com/torvalds/linux/blob/master/kernel/sched/cpufreq_schedutil.c) | الكود الحالي لـ schedutil اللي بيستخدم الـ hook |

الـ commit الأصلي اللي أضاف `cpufreq_add_update_util_hook` كان جزء من series أضافت آلية callbacks بين الـ scheduler وـ governors.

---

### نقاشات الـ Mailing List

| الرابط | الموضوع |
|--------|---------|
| [PATCH v4 7/7 — schedutil governor](https://lore.kernel.org/lkml/11678919.CQLTrQTYxG@vostro.rjw.lan/) | النقاش الرئيسي على lkml لـ schedutil — Rafael J. Wysocki |
| [PATCH v3 7/7 — schedutil governor](https://lore.kernel.org/lkml/1855303.8Nyy7L51ju@vostro.rjw.lan/) | نسخة سابقة مع تعليقات Peter Zijlstra |
| [Patchwork — schedutil v8](https://patchwork.kernel.org/project/linux-pm/patch/9554623.KpzoB0bi0t@vostro.rjw.lan/) | النسخة النهائية اللي اتقبلت في linux-pm tree |
| [add cpu to update_util_data](https://lore.kernel.org/patchwork/patch/677244/) | نقاش إضافة الـ `cpu` field في الـ callback data |

---

### Kernelnewbies.org

الصفحات دي بتوضح في أي kernel version اتضافت كل feature:

| الصفحة | التغيير المهم |
|--------|--------------|
| [Linux 4.7](https://kernelnewbies.org/Linux_4.7) | إضافة schedutil governor لأول مرة |
| [Linux 4.10](https://kernelnewbies.org/Linux_4.10) | intel_pstate بقى يشتغل مع generic cpufreq governors |
| [Linux 4.14](https://kernelnewbies.org/Linux_4.14) | الـ scheduler بقى يعمل cpufreq updates لـ remote CPUs |
| [Linux 4.11](https://kernelnewbies.org/Linux_4.11) | إضافة `cpufreq.off=1` boot option |
| [Linux 5.7](https://kernelnewbies.org/Linux_5.7) | frequency-invariant accounting على x86 |

---

### eLinux.org

| الصفحة | الفائدة |
|--------|---------|
| [R-CAR-GEN3-CPUFreq Tests](https://elinux.org/Tests:R-CAR-GEN3-CPUFreq) | مثال عملي على التعامل مع cpufreq في embedded systems |
| [CELF PM Requirements 2006](https://www.elinux.org/CELF_PM_Requirements_2006) | المتطلبات التاريخية للـ power management في embedded Linux |
| [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) | مصادر عامة للكيرنل |

---

### كتب مقترحة

#### Linux Device Drivers, 3rd Edition (LDD3)
- الكتاب متاح مجاناً على [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- الـ chapters المتعلقة بـ power management وـ sysfs مفيدة لفهم كيف cpufreq بيعرض نفسه للـ userspace

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 4**: Process Scheduling — فهم الـ scheduler ضروري قبل ما تفهم الـ integration
- **Chapter 14**: The Block I/O Layer — نفس pattern الـ callbacks موجود هنا
- الكتاب بيشرح الـ scheduler بشكل ممتاز وده أساس فهم `update_util_data`

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **Chapter 14**: Kernel Debugging Techniques
- **Chapter 16**: Porting Linux to a Custom Platform — بيشرح كيف بتضيف cpufreq driver لـ custom hardware

---

### توثيق إضافي مفيد

```
Documentation/admin-guide/pm/cpufreq.rst     ← للـ sysfs interface والـ governors
Documentation/admin-guide/pm/schedutil.rst   ← schedutil بالتفصيل
Documentation/scheduler/sched-capacity.rst   ← فهم util و cap
Documentation/scheduler/sched-energy.rst     ← Energy Aware Scheduling
Documentation/cpu-freq/cpu-drivers.rst       ← كتابة cpufreq driver جديد
```

---

### Search Terms للبحث عن مزيد من المعلومات

لو عايز تكمل بحثك استخدم الـ terms دي:

```
schedutil governor linux kernel
cpufreq_update_util scheduler
update_util_data cpufreq hook
SCHED_CPUFREQ_IOWAIT flag
map_util_freq capacity invariant
map_util_perf 1.25 margin
cpufreq_this_cpu_can_update policy
Energy Aware Scheduling EAS cpufreq
frequency invariant accounting arm64
cpufreq remote wakeup callback
```

على الـ kernel mailing list (`lore.kernel.org`):
```
lore.kernel.org/linux-pm cpufreq schedutil
lore.kernel.org/lkml update_util_data
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `cpufreq_add_update_util_hook` بتخلّيك تسجّل callback بيتّشال من الـ scheduler في كل مرة بيحسب utilization جديد لـ CPU معيّن. ده بالضبط زي ما الـ `schedutil` governor بيشتغل — بس هنا بنعمله بنفسنا من module خارجي لمراقبة اللي بيحصل.

الـ hook الأسلم والأكثر استخداماً من الملف ده هو تسجيل `update_util_data` على CPU واحد، وطباعة الـ `util` و`time` و`flags` في كل مرة الـ scheduler بيطلب تحديث.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * util_hook_mod.c
 *
 * Hooks cpufreq update-util callback on CPU 0 and logs
 * scheduler utilization signals via pr_info().
 */

#include <linux/module.h>        /* MODULE_*, module_init/exit */
#include <linux/kernel.h>        /* pr_info */
#include <linux/sched/cpufreq.h> /* update_util_data, cpufreq_add/remove_util_hook */
#include <linux/cpu.h>           /* num_online_cpus */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("Monitor scheduler cpufreq util hook on CPU 0");

/* ------------------------------------------------------------------ */
/* Target CPU to monitor                                               */
/* ------------------------------------------------------------------ */
#define TARGET_CPU  0

/* ------------------------------------------------------------------ */
/* Our hook container — must stay valid while hook is registered       */
/* ------------------------------------------------------------------ */
static struct update_util_data util_hook;

/* ------------------------------------------------------------------ */
/* Callback — called by scheduler every time it updates CPU util       */
/* ------------------------------------------------------------------ */
static void my_util_update(struct update_util_data *data,
                           u64 time,
                           unsigned int flags)
{
    /*
     * flags == SCHED_CPUFREQ_IOWAIT (1<<0) means the CPU is waiting
     * for I/O; otherwise it's a normal compute-utilization update.
     */
    pr_info("util_hook [cpu%d]: time=%llu ns  iowait=%s\n",
            TARGET_CPU,
            time,
            (flags & SCHED_CPUFREQ_IOWAIT) ? "yes" : "no");
}

/* ------------------------------------------------------------------ */
/* init — register the hook                                            */
/* ------------------------------------------------------------------ */
static int __init util_hook_init(void)
{
    /* Make sure CPU 0 is online before registering */
    if (!cpu_online(TARGET_CPU)) {
        pr_err("util_hook: CPU %d is offline, aborting\n", TARGET_CPU);
        return -ENODEV;
    }

    /*
     * cpufreq_add_update_util_hook() links our struct into the
     * per-CPU pointer; after this the scheduler calls my_util_update()
     * directly on every utilization recalculation for CPU 0.
     */
    cpufreq_add_update_util_hook(TARGET_CPU, &util_hook, my_util_update);

    pr_info("util_hook: registered on CPU %d\n", TARGET_CPU);
    return 0;
}

/* ------------------------------------------------------------------ */
/* exit — unregister the hook before module memory is freed            */
/* ------------------------------------------------------------------ */
static void __exit util_hook_exit(void)
{
    /*
     * Must unregister before returning so the scheduler can't call
     * into freed module memory after rmmod completes.
     */
    cpufreq_remove_update_util_hook(TARGET_CPU);
    pr_info("util_hook: unregistered from CPU %d\n", TARGET_CPU);
}

module_init(util_hook_init);
module_exit(util_hook_exit);
```

---

### Makefile

```makefile
obj-m += util_hook_mod.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | ماكروهات `MODULE_*` وتعريفات `module_init/exit` |
| `linux/kernel.h` | `pr_info` و`pr_err` |
| `linux/sched/cpufreq.h` | تعريف `update_util_data` والدوال `cpufreq_add/remove_update_util_hook` |
| `linux/cpu.h` | `cpu_online()` عشان نتأكد إن الـ CPU شغال قبل ما نسجّل |

---

#### الـ `update_util_data` struct

```c
static struct update_util_data util_hook;
```

ده الـ container اللي الكيرنل بيحتفظ بـ pointer ليه داخل الـ per-CPU data. لازم يكون `static` أو allocated عشان يفضل valid طول ما الـ hook مسجّل — لو اتحرر من الـ stack هيحصل use-after-free.

---

#### الـ callback `my_util_update`

```c
static void my_util_update(struct update_util_data *data, u64 time, unsigned int flags)
```

- **`data`**: pointer للـ `util_hook` struct بتاعنا — ممكن تستخدمه لو حبيت تخزّن state إضافي بـ `container_of`.
- **`time`**: الـ timestamp بالـ nanoseconds اللي جت من الـ scheduler — بيفيد لو عايز تحسب الفرق بين invocations.
- **`flags`**: بس `SCHED_CPUFREQ_IOWAIT` لحد دلوقتي، بيقول إيه نوع الضغط اللي على الـ CPU.

الـ callback بيتّشال في **سياق الـ scheduler** (softirq أو task context حسب الحالة) — يعني ممنوع blocking أو sleeping جوّاه.

---

#### الـ `module_init`

```c
cpufreq_add_update_util_hook(TARGET_CPU, &util_hook, my_util_update);
```

الدالة دي بتكتب الـ pointer بتاع الـ callback في الـ per-CPU variable الخاص بـ CPU المطلوب. من اللحظة دي، الـ `schedutil` أو أي code بيستدعي `cpufreq_update_util()` هيمشي عندنا أوتوماتيك.

التحقق من `cpu_online()` أولاً ضروري لأن الـ API مش بيعمل validation داخلي، وعلى CPU offline هيودّي corruption.

---

#### الـ `module_exit`

```c
cpufreq_remove_update_util_hook(TARGET_CPU);
```

لازم يتعمل قبل ما تخلّص `rmmod` — لو الـ scheduler استدعى الـ callback بعد ما الـ module memory اتحررت، هيحصل kernel panic. الـ `remove` بيعمل `NULL` للـ per-CPU pointer، فبعدها أي `cpufreq_update_util()` على الـ CPU ده هيـskip الـ callback بأمان.

---

### تجربة سريعة

```bash
# Build
make

# Insert
sudo insmod util_hook_mod.ko

# Watch logs (Ctrl+C to stop)
sudo dmesg -w | grep util_hook

# Remove
sudo rmmod util_hook_mod
```

مثال ناتج متوقع:

```
[  123.456789] util_hook: registered on CPU 0
[  123.460001] util_hook [cpu0]: time=123460001000 ns  iowait=no
[  123.460512] util_hook [cpu0]: time=123460512000 ns  iowait=yes
...
[  145.001234] util_hook: unregistered from CPU 0
```

---

### ملاحظات أمان

- الكود ده بيشتغل على **CPU 0 فقط** — لو عايز كل الـ CPUs، عمل loop بـ `for_each_online_cpu()` وخزّن array من `update_util_data`.
- الـ callback بيتّشال بتردد عالي جداً (آلاف المرات في الثانية) — في production بدّل الـ `pr_info` بـ tracepoint أو atomic counter عشان ماتغرّقش الـ syslog.
- تأكد إن الـ `CONFIG_CPU_FREQ` مفعّل في الـ kernel config وإلا مش هتلاقي الـ symbols.
