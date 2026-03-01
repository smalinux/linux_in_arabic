## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem اللي ينتمي له الملف ده؟

**الـ `intel_pstate`** جزء من subsystem اسمه **CPU Performance Scaling (CPUFreq)** في الـ Linux kernel. ده الـ subsystem اللي بيتحكم في سرعة المعالج — بيسرّعه لما الشغل يزيد، وبيبطّئه لما الشغل يقل، عشان توفر طاقة أو توفر حرارة.

الملف `Documentation/admin-guide/pm/intel_pstate.rst` هو **الدليل الشامل** بتاع الـ driver ده — مش كود، لكنه بيشرح كل حاجة عن طريقة تشغيله وضبطه وفهمه.

---

### القصة الكاملة — ليه محتاجين الملف ده أصلاً؟

#### تخيل معايا السيناريو ده:

عندك لاب توب Intel. المعالج فيه بيقدر يشتغل بسرعات مختلفة — مثلاً من 800 MHz لـ 5.0 GHz. لو خلّيت المعالج يشتغل بأقصى سرعة دايماً، البطارية هتخلص في ساعة وهيبقى ساخن جداً. لو قلّصته على أبطأ سرعة دايماً، الجهاز هيبقى بطيء وما يصلوش.

الحل إن في **driver** بيقرر في كل لحظة إيه السرعة المناسبة بناءً على مدى اشتغال المعالج. ده بالظبط دور **`intel_pstate`**.

بس المشكلة إن Intel مش بس عندها "سرعات" ثابتة زي أي معالج تاني — عندها حاجة أعمق اسمها **P-states** (Performance States). الـ P-state مش مجرد رقم تردد، ده تمثيل داخلي للهاردوير بيلخّص الجهد الكهربي والتردد معاً. وعشان كده `intel_pstate` بيتعامل مع المعالج بطريقة مختلفة تماماً عن الـ drivers التانية زي `acpi-cpufreq`.

---

### المشكلة اللي بيحلّها `intel_pstate`

#### الـ Driver التقليدي (`acpi-cpufreq`) — المشكلة

الـ `acpi-cpufreq` بيشتغل من قايمة P-states جاهزة مكتوبة في جداول الـ BIOS (ACPI `_PSS` objects). المشكلة:

- القايمة دي **ناقصة** — الـ Turbo range (أعلى سرعات مؤقتة) ممثّلة بـ item واحد بس بدل ما يكون فيه تمثيل كامل لكل سرعات الـ Turbo.
- ده بيخلي الـ governors يقرروا إن الـ Turbo هو "أعلى بـ 1 MHz من الحد العادي" فقط — فيُقلّوا من استخدامه إلا في أعلى الأحمال جداً.
- النتيجة: أداء أقل من الممكن في كتير من الأوقات.

#### الحل — `intel_pstate`

`intel_pstate` بيتكلم مع المعالج **مباشرة** عبر الـ MSRs (Model-Specific Registers) أو عبر الـ HWP feature بدل ما يعتمد على جداول الـ BIOS. ده بيديه:

1. **صورة كاملة** لكل الـ P-states المتاحة بما فيها الـ Turbo range كاملاً.
2. قدرة إن المعالج نفسه يختار السرعة المثلى (HWP mode) لما بيديله hints بس.
3. تحكم أدق وأسرع في الأداء والطاقة.

---

### أوضاع التشغيل الثلاثة — ببساطة

```
+---------------------------+
|      intel_pstate         |
+---------------------------+
        |           |
   Active Mode   Passive Mode
        |           |
   +----+----+      |
   |         |   يتصرف زي
  HWP      no-HWP  أي Driver
   |         |   عادي مع
  hardware  software  CPUFreq
  decides   decides   governors
```

**Active Mode مع HWP:**
المعالج نفسه بيختار السرعة. `intel_pstate` بس بيديله "تلميحات" (hints) زي: "أنا أفضّل الأداء" أو "أنا أفضّل توفير طاقة" عبر **EPP** (Energy-Performance Preference) أو **EPB** (Energy-Performance Bias) registers.

**Active Mode بدون HWP:**
الـ driver نفسه بيحسب السرعة المناسبة كل 10ms باستخدام feedback من registers الهاردوير (مش تقديرات نظرية).

**Passive Mode:**
`intel_pstate` بيتصرف زي أي driver عادي — بيستقبل أوامر من الـ CPUFreq governors (زي `schedutil`) وبينفّذها على الهاردوير.

---

### الـ Turbo Boost — مش سحر

الـ **Turbo P-states** هي سرعات عالية جداً المعالج يقدر يوصلها لفترة قصيرة بس (مش دايمة) لأن:
- الطاقة محدودة.
- الحرارة محدودة.

```
P-state Range:
|=====Non-Turbo======|====Turbo Range====|
0.8 GHz         3.0 GHz            5.0 GHz
                  ^
             turbo threshold
```

`intel_pstate` بيعرف الـ turbo threshold الحقيقي من الهاردوير مباشرة — مش من الـ BIOS — فبيقدر يتحكم فيه بدقة عبر الـ `no_turbo` attribute في sysfs.

---

### دعم الـ Hybrid Processors (مثال: Intel Alder Lake)

المعالجات الحديثة زي Alder Lake عندها **نوعين من الكور**:
- **P-cores** (Performance cores) — أسرع وأكثر استهلاكاً للطاقة.
- **E-cores** (Efficiency cores) — أبطأ وأوفر طاقة.

`intel_pstate` بيحل المشكلة دي بطريقتين:

1. **Capacity-Aware Scheduling (CAS):** بيقول للـ scheduler "الـ P-core عنده capacity أعلى" فالـ scheduler بيبعت المهام الثقيلة عليه والخفيفة على الـ E-core — وده بيوفر طاقة.

2. **Energy-Aware Scheduling (EAS):** لو `schedutil` هو الـ governor والـ SMT معطّل، `intel_pstate` بيسجّل **Energy Model** للمعالج يخلي الـ scheduler يفضّل الـ E-cores للمهام الخفيفة عشان أوفر طاقة.

---

### الواجهة مع المستخدم (sysfs)

كل الإعدادات في `/sys/devices/system/cpu/intel_pstate/`:

| الـ Attribute | الوظيفة |
|---|---|
| `max_perf_pct` | أقصى P-state مسموح به كـ % من الـ maximum |
| `min_perf_pct` | أدنى P-state مسموح به كـ % من الـ maximum |
| `no_turbo` | تعطيل/تفعيل الـ Turbo |
| `status` | تغيير الوضع: active / passive / off |
| `hwp_dynamic_boost` | رفع أدنى سرعة مؤقتاً لما task ينتهي من I/O wait |
| `energy_efficiency` | توفير طاقة إضافي لبعض موديلات Kaby/Coffee Lake |

---

### Kernel Command Line Options المهمة

| الـ Option | المعنى |
|---|---|
| `intel_pstate=disable` | لا تسجّل الـ driver خالص |
| `intel_pstate=passive` | ابدأ في الـ passive mode |
| `intel_pstate=no_hwp` | لا تفعّل الـ HWP حتى لو المعالج يدعمه |
| `intel_pstate=active` | أجبره على الـ active mode |
| `intel_pstate=no_cas` | عطّل الـ Capacity-Aware Scheduling |

---

### الملفات المرتبطة اللي المبرمج لازم يعرفها

**الكود الأساسي:**
- `/workspace/external/linux/drivers/cpufreq/intel_pstate.c` — الكود الكامل للـ driver
- `/workspace/external/linux/drivers/cpufreq/cpufreq.c` — الـ CPUFreq core framework
- `/workspace/external/linux/drivers/cpufreq/acpi-cpufreq.c` — البديل المعتمد على ACPI

**الـ Headers:**
- `/workspace/external/linux/include/linux/cpufreq.h` — تعريفات `cpufreq_policy`, `cpufreq_governor`, إلخ
- `/workspace/external/linux/include/linux/sched/cpufreq.h` — الربط بين الـ scheduler والـ cpufreq

**التوثيق:**
- `/workspace/external/linux/Documentation/admin-guide/pm/cpufreq.rst` — الـ CPUFreq العام (لازم تقراه الأول)
- `/workspace/external/linux/Documentation/admin-guide/pm/intel_pstate.rst` — الملف الحالي

**الـ Governors المصاحبة:**
- `/workspace/external/linux/drivers/cpufreq/cpufreq_performance.c`
- `/workspace/external/linux/drivers/cpufreq/cpufreq_powersave.c`
- `/workspace/external/linux/drivers/cpufreq/cpufreq_ondemand.c`

---

### ملخص الصورة الكبيرة

**الـ `intel_pstate`** هو الـ driver المتخصص لمعالجات Intel من Sandy Bridge وما بعدها. بدل ما يعتمد على جداول BIOS محدودة زي `acpi-cpufreq`، بيتكلم مع الهاردوير مباشرة عبر MSRs أو HWP registers. بيقدر يشتغل في وضعين: active (هو أو الهاردوير نفسه يختار السرعة) أو passive (يستجيب لأوامر الـ CPUFreq governors التقليدية). على المعالجات الهجينة (P-cores + E-cores)، بيمكّن الـ CAS وEAS عشان يوزّع المهام بشكل أوفر للطاقة. كل إعداداته موجودة في sysfs وبعض خياراته تُمرَّر عبر kernel command line.
## Phase 2: شرح الـ intel_pstate Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

الـ CPU مش بيشتغل بتردد ثابت — هو عنده **P-states** (Performance States) مختلفة، كل واحدة فيها تردد وجهد كهربي مختلف. كل ما P-state أعلى، كل ما الأداء أعلى والاستهلاك أعلى.

المشكلة الأساسية: **مين المسؤول يختار الـ P-state المناسب؟**

الـ Linux kernel عنده framework عام اسمه **CPUFreq** — هو بيتعامل مع كل أنواع الـ CPUs (ARM، AMD، Intel). الـ CPUFreq بيعتمد على:
- **scaling driver** — بيكلم الـ hardware الفعلي
- **scaling governor** — بيقرر أيه الـ frequency المطلوبة (ondemand، schedutil، إلخ)

المشكلة مع Intel: بروسيسورات Sandy Bridge وبعدها ادخلت مفهوم أعمق من مجرد "اختار frequency من جدول". الـ P-state في Intel مش مجرد رقم frequency — هو **ratio يُكتب في MSR** (Model Specific Register)، والـ hardware نفسه ممكن يأخذ القرار. فالـ CPUFreq التقليدي مش كافي.

لذلك اتعمل `intel_pstate` — driver متخصص لبروسيسورات Intel بيتعامل مع:
1. الـ **P-state representation** الحقيقي حسب Intel spec (مش مجرد kHz)
2. الـ **HWP** (Hardware-managed P-states) — الـ CPU نفسه يختار
3. الـ **Turbo Boost** — P-states فوق الحد الأساسي
4. الـ **EPP/EPB** — hints للـ hardware يفضل performance ولا energy

---

### الحل — الـ kernel بيتعامل إزاي؟

`intel_pstate` بيحل المشكلة بطريقتين (modes):

**Active Mode:** الـ driver بيتجاوز الـ governors layer خالص — هو اللي بيقرر P-state مباشرةً (أو بيسيب الـ HWP hardware يقرر مع hints).

**Passive Mode:** الـ driver بيتصرف زي أي CPUFreq driver عادي — الـ generic governors بيكلموه وهو بيترجم طلباتهم لـ MSR writes.

---

### تشبيه من الواقع — المسؤول والمدير

تخيل مصنع فيه:
- **المدير العام (CPUFreq core)** — بيحدد السياسة العامة (min/max freq)
- **مشرف الإنتاج (scaling governor)** — بيقرر احتياج الإنتاج (schedutil، ondemand)
- **المشغّل (scaling driver)** — بيكلم الآلة الفعلية

في الـ **passive mode**: المدير → المشرف → المشغّل → الآلة. سلسلة طبيعية.

في الـ **active mode**: `intel_pstate` بيكون **مشرف + مشغّل في نفس الوقت** — بيتجاوز المشرف الخارجي ويقرر هو. وفي حالة الـ HWP، الآلة نفسها (الـ CPU) بتشتغل **autonomously** — المشغّل بس بيديها hints وهي تقرر.

التشبيه مطابق للكود:
- المدير العام = `struct cpufreq_policy` + CPUFreq core
- المشرف = `struct cpufreq_governor` (schedutil, ondemand)
- المشغّل = `struct cpufreq_driver` — هنا `intel_pstate` يملأ الـ callbacks
- الآلة = Intel CPU hardware + MSRs + HWP engine
- الـ hints = EPP (Energy Performance Preference) register

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Space                               │
│  /sys/devices/system/cpu/intel_pstate/  (global attrs)          │
│  /sys/devices/system/cpu/cpuX/cpufreq/  (per-policy attrs)      │
│  max_perf_pct, min_perf_pct, no_turbo, status, EPP              │
└───────────────────────────┬─────────────────────────────────────┘
                            │ sysfs read/write
┌───────────────────────────▼─────────────────────────────────────┐
│                    CPUFreq Core                                  │
│  struct cpufreq_policy { min, max, cur, governor, driver_data } │
│  - manages policy lifecycle (online/offline)                     │
│  - coordinates between governor and driver                       │
│  - exposes sysfs interface                                       │
└──────────┬────────────────────────────────────┬─────────────────┘
           │                                    │
           │  PASSIVE MODE                      │  ACTIVE MODE
           ▼                                    ▼
┌──────────────────────┐            ┌───────────────────────────┐
│  Generic Governor    │            │  intel_pstate internal     │
│  (schedutil/ondemand)│            │  P-state algorithm         │
│  - reads CPU util    │            │  "powersave" / "performance"│
│  - calls driver->    │            │  - reads APERF/MPERF MSRs  │
│    target_index()    │            │  - called from sched hook  │
└──────────┬───────────┘            └────────────┬──────────────┘
           │                                     │
           └──────────────┬──────────────────────┘
                          │
           ┌──────────────▼──────────────────────┐
           │          intel_pstate Driver         │
           │  struct cpufreq_driver {             │
           │    .init   = intel_pstate_cpu_init   │
           │    .verify = intel_pstate_verify     │
           │    .setpolicy (active) OR            │
           │    .target_index (passive)           │
           │    .adjust_perf (HWP fast path)      │
           │  }                                   │
           └──────────────┬──────────────────────┘
                          │
           ┌──────────────▼──────────────────────┐
           │         Intel CPU Hardware           │
           │                                     │
           │  MSR_IA32_PERF_CTL  ← no-HWP write  │
           │  MSR_HWP_REQUEST    ← HWP write      │
           │  MSR_HWP_CAPABILITIES (read limits)  │
           │  APERF/MPERF counters (read util)    │
           │  EPP register (energy hint)          │
           │                                     │
           │  ┌────────────────────────────────┐  │
           │  │   HWP Engine (hardware FSM)    │  │
           │  │   reads hints → picks P-state  │  │
           │  └────────────────────────────────┘  │
           └─────────────────────────────────────┘
```

---

### الـ Operation Modes بالتفصيل

#### Active Mode — بدون HWP

```
CPU Scheduler
    │
    │ utilization update callback (كل 10ms)
    ▼
intel_pstate_update_util()
    │
    │ يحسب: APERF/MPERF ratio = actual freq / max freq
    │ يختار P-state بناءً على الـ load
    ▼
intel_pstate_set_pstate()
    │
    │ write to MSR_IA32_PERF_CTL
    ▼
CPU Hardware changes frequency
```

الـ `APERF` (Actual Performance Frequency counter) و`MPERF` (Maximum Performance Frequency counter) هما MSRs بيعدوا:
- **MPERF**: يزيد بثبات مع كل cycle (زي ساعة مرجعية)
- **APERF**: يزيد بالنسبة للـ actual performance (يتأثر بـ C-states والـ throttling)

النسبة `APERF/MPERF` تدي الـ driver الـ effective utilization بدقة.

#### Active Mode — مع HWP

```
intel_pstate initialization
    │
    │ يكتب EPP/EPB في MSR_HWP_REQUEST
    │ يحدد min_perf و max_perf في نفس الـ register
    ▼
HWP Hardware Engine يشتغل autonomously
    │
    │ scheduler callback (لو اشتغل)
    │ → بس لتحديث scaling_cur_freq في sysfs
    │ → مش لاختيار P-state
    ▼
CPU يختار P-state لوحده حسب الـ hints
```

في HWP + performance: الـ driver يكتب EPP = 0 (focus on performance فقط)
في HWP + powersave: الـ driver يكتب EPP = whatever firmware set

#### Passive Mode

```
schedutil governor
    │
    │ يحسب الـ target frequency من utilization
    │ يستدعي cpufreq_driver_fast_switch() OR target_index()
    ▼
intel_pstate (as "intel_cpufreq")
    │
    │ يترجم الـ frequency لـ P-state ratio
    │ write MSR_IA32_PERF_CTL or MSR_HWP_REQUEST
    ▼
Hardware
```

---

### الـ P-state Representation

ده نقطة جوهرية: Intel P-states **مش frequencies مباشرة**.

P-state = **ratio** يُحسب كالتالي:
```
P-state ratio = Target Frequency / 100 MHz (base clock)
```

مثلاً CPU عنده base 100MHz:
- P-state 26 → 2600 MHz (2.6 GHz)
- P-state 40 → 4000 MHz (4.0 GHz) [turbo]

الـ driver بيعمل mapping:
```c
/* P-state → frequency (kHz) */
freq_kHz = p_state_ratio * base_freq_100mhz * 1000

/* frequency → P-state */
p_state = freq_kHz / (base_freq_100mhz * 1000)
```

الـ CPUFreq core بيتكلم بالـ kHz، لكن الـ hardware بيتكلم بالـ ratio. الـ `intel_pstate` هو المترجم.

---

### الـ Turbo P-states

```
P-state range على Core i7 مثلاً:

P-state:  [8]...[26]...[27]...[40]
           ↑      ↑      ↑      ↑
          min   base   turbo  max-turbo
          800MHz 2.6G  2.7G    4.0G
                  │←──────────────────┤
                  │    turbo range    │
                  │                  │
              non-turbo          one-core
              max (sustainable)   max
```

**الـ turbo threshold** = الـ P-state الأعلى قبل ما ندخل الـ turbo range. الـ turbo P-states:
- **مش sustainable** — مضمونش الـ CPU يفضل فيها للأبد (thermal/power limits)
- بيتحكم فيهم الـ hardware نفسه حسب عدد الـ cores الشاغلة
- كلما cores أكتر شغّالة، كلما الـ max turbo P-state أقل

الـ `no_turbo` sysfs attribute بيمنع الـ driver من استخدامهم خالص.

---

### الـ Struct Relationships

```
struct cpufreq_driver (intel_pstate registers this)
┌─────────────────────────────────────┐
│  name: "intel_pstate" OR            │
│        "intel_cpufreq"              │
│  .init    → intel_pstate_cpu_init() │
│  .verify  → intel_pstate_verify()   │
│  .setpolicy (active mode)  OR       │
│  .target_index (passive mode)       │
│  .adjust_perf → HWP fast switch     │
│  .register_em → Energy Model reg    │
└─────────────┬───────────────────────┘
              │ per-CPU
              ▼
struct cpufreq_policy (واحد per logical CPU في intel_pstate)
┌─────────────────────────────────────┐
│  cpu: 0                             │
│  min: 800000 kHz                    │
│  max: 4000000 kHz                   │
│  cur: 2600000 kHz                   │
│  governor: &schedutil_governor      │ ← passive mode
│         OR &intel_pstate_gov        │ ← active mode
│  driver_data: → intel_pstate data   │
│  cpuinfo.max_freq: max turbo freq   │
│  cpuinfo.min_freq: min P-state freq │
└─────────────────────────────────────┘

struct cpufreq_governor (active mode: intel_pstate provides its own)
┌─────────────────────────────────────┐
│  name: "powersave" OR "performance" │
│  NOTE: مش نفس الـ generic governors │
│  .start → register sched callbacks  │
│  .stop  → unregister callbacks      │
│  .limits → update HWP request regs  │
└─────────────────────────────────────┘
```

---

### الـ Energy Performance Preference (EPP)

ده concept مهم خاص بالـ HWP. الـ `MSR_HWP_REQUEST` بيحتوي على:

```
MSR_HWP_REQUEST layout:
 ┌──────────────┬──────────────┬──────────────┬──────────────┐
 │  min_perf    │  max_perf    │  desired_perf│     EPP      │
 │  [7:0]       │  [15:8]      │  [23:16]     │  [31:24]     │
 └──────────────┴──────────────┴──────────────┴──────────────┘

EPP values:
  0   = performance (ignore energy)
  128 = balance_performance (default)
  192 = balance_power
  255 = power (max energy saving)
```

User space بيكتب string في sysfs:
```
echo "balance_power" > /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
```

الـ `intel_pstate` بيترجمه لـ integer ويكتبه في الـ MSR.

---

### Support for Hybrid Processors (P-core + E-core)

البروسيسورات الـ hybrid (زي Intel Alder Lake+) فيها:
- **P-cores** (Performance cores) — أسرع، أكل طاقة أكتر
- **E-cores** (Efficiency cores) — أبطأ، أقل استهلاك

`intel_pstate` بيتعامل معاهم إزاي؟

```
Hybrid CPU Package:
┌─────────────────────────────────────────┐
│  P-core cluster        E-core cluster   │
│  HWP_max = 180         HWP_max = 120    │
│  capacity = 1024       capacity = 682   │
│      (180*1024/180)        (120*1024/180)│
└─────────────────────────────────────────┘

intel_pstate يبلغ scheduler بـ:
- CPU capacity لكل core
- Frequency invariance data

Capacity-Aware Scheduling (CAS):
- scheduler يحط الـ tasks الثقيلة على P-cores
- tasks الخفيفة تروح E-cores لتوفير طاقة
- مش بيعتمد على priority — بيعتمد على capacity
```

الـ `arch_scale_cpu_capacity()` بيرجع الـ capacity relative للأقوى CPU. الـ `intel_pstate` هو اللي بيملأ الـ table دي.

---

### P-State Limits Coordination

في نفس الوقت، الـ P-state بيتحكم فيه من مستويين:

```
Global limits (sysfs: /sys/devices/system/cpu/intel_pstate/)
┌─────────────────────────────────────┐
│  max_perf_pct = 80  (80% of max)    │
│  min_perf_pct = 20  (20% of max)    │
└───────────────┬─────────────────────┘
                │ applies to ALL CPUs
                ▼
Per-policy limits (sysfs: /sys/devices/system/cpu/cpuX/cpufreq/)
┌─────────────────────────────────────┐
│  scaling_max_freq = 3500000 kHz     │
│  scaling_min_freq = 1000000 kHz     │
└───────────────┬─────────────────────┘
                │ applies to THIS CPU only
                ▼
Effective P-state = max(global_min, per_policy_min)
                  to min(global_max, per_policy_max)
```

في الـ HWP mode، القيم دي بتتكتب مباشرةً في `MSR_HWP_REQUEST` (min_perf و max_perf fields). في الـ non-HWP mode، الـ driver بيطبقهم manually كل مرة قبل ما يكتب الـ P-state.

---

### intel_pstate مقارنةً بـ acpi-cpufreq

ده سؤال شائع — امتى تستخدم أيه؟

| الجانب | intel_pstate | acpi-cpufreq |
|--------|-------------|--------------|
| P-states المتاحة | كل الـ range (turbo كامل) | محدود بـ ACPI _PSS table |
| Turbo representation | دقيق — ratio حقيقي | 1 MHz فوق non-turbo (تقريبي) |
| HWP support | نعم (native) | لأ |
| سهولة التحكم | sysfs + EPP | sysfs فقط |
| governor compatibility | active: محدود / passive: كل الـ governors | كل الـ governors |
| دقة scaling_cur_freq | من APERF/MPERF (حقيقي) | آخر freq طلبها الـ governor |

المشكلة الرئيسية مع `acpi-cpufreq`: الـ turbo range كله ممثل بـ entry واحد (+1 MHz فوق الـ base). الـ governors اللي بيحسبوا proportionally هيفتحوا الـ turbo بس عند 99%+ load — وده خسارة أداء.

`intel_pstate` بيكشف الـ turbo range الكامل لأي governor → أداء أحسن بشكل افتراضي.

---

### ما الذي يمتلكه intel_pstate مقابل ما يفوّضه

**intel_pstate يمتلك:**
- الـ mapping بين P-state ratios والـ frequencies (processor-specific)
- قراءة الـ MSR capabilities (min/max P-states, turbo threshold)
- كتابة الـ P-state في `MSR_IA32_PERF_CTL` أو `MSR_HWP_REQUEST`
- إدارة الـ EPP/EPB hints
- الـ utilization update callbacks (APERF/MPERF reading)
- كشف الـ hybrid topology وتسجيل الـ CPU capacities
- الـ Energy Model registration (لو CONFIG_ENERGY_MODEL مفعّل)

**intel_pstate يفوّض للـ CPUFreq core:**
- الـ sysfs policy interface (scaling_max_freq, scaling_min_freq, إلخ)
- الـ policy lifecycle (online/offline CPUs)
- الـ notifier chain (للـ thermal cooling وغيره)
- الـ statistics (cpufreq-stats)

**intel_pstate يفوّض للـ hardware (HWP mode):**
- اختيار الـ P-state الفعلي لحظياً
- التعامل مع الـ thermal throttling
- التوازن بين الـ cores داخل نفس الـ package

---

### مفاهيم من subsystems تانية تحتاجها

- **MSR (Model Specific Registers)** — registers خاصة بكل CPU model، بتتعامل معاها عبر `rdmsr`/`wrmsr` instructions. الـ `intel_pstate` بيستخدمها لقراءة capabilities وكتابة P-state requests.

- **CPU Scheduler (sched)** — الـ `intel_pstate` بيسجّل `update_util` hook مع الـ scheduler. الـ scheduler بيستدعيه عند كل task switch أو timer tick لتحديث الـ P-state.

- **ACPI** — الـ `_PSS` objects بتحدد الـ P-states المتاحة لـ `acpi-cpufreq`. الـ `intel_pstate` بيتجاهل الـ _PSS عادةً ويقرأ من الـ MSRs مباشرةً.

- **Energy Model (EM)** — framework في الـ kernel بيسمح للـ scheduler يختار CPU بناءً على الـ energy cost. الـ `intel_pstate` بيبني model اصطناعي (abstract cost) للـ hybrid processors.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options المهمة

#### Config Options

| Option | الأثر |
|---|---|
| `CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE` | لو set → default algorithm هو `performance`، غير كده → `powersave` |
| `CONFIG_ACPI` | يفعّل دعم ACPI _PSS و _PPC وحدود الـ P-states من firmware |
| `CONFIG_ACPI_CPPC_LIB` | يفعّل CPPC لحساب scaling factor وضبط ITMT priorities |
| `CONFIG_ENERGY_MODEL` | يفعّل تسجيل Energy Model للـ hybrid processors بدون SMT |

#### Kernel Command Line Options

| Option | المعنى |
|---|---|
| `intel_pstate=disable` | لا تسجّل الـ driver خالص |
| `intel_pstate=active` | ابدأ في الـ active mode |
| `intel_pstate=passive` | ابدأ في الـ passive mode (يستخدم `intel_cpufreq` كاسم driver) |
| `intel_pstate=no_hwp` | لا تفعّل HWP حتى لو الـ CPU يدعمه |
| `intel_pstate=hwp_only` | سجّل الـ driver بس لو HWP متاح |
| `intel_pstate=force` | استخدمه حتى لو `acpi-cpufreq` هو الـ preferred |
| `intel_pstate=per_cpu_perf_limits` | حدود P-state منفصلة لكل CPU، لا يعرض global attrs |
| `intel_pstate=support_acpi_ppc` | خذ حدود `_PPC` من ACPI في الاعتبار |
| `intel_pstate=no_cas` | لا تفعّل Capacity-Aware Scheduling على الـ hybrid |

#### Enum: `energy_perf_value_index`

| القيمة | الـ Index | الـ EPP value (0–255) |
|---|---|---|
| `EPP_INDEX_DEFAULT` | 0 | firmware default |
| `EPP_INDEX_PERFORMANCE` | 1 | `HWP_EPP_PERFORMANCE` (0) |
| `EPP_INDEX_BALANCE_PERFORMANCE` | 2 | 0x80 (أو per-gen override) |
| `EPP_INDEX_BALANCE_POWERSAVE` | 3 | `HWP_EPP_BALANCE_POWERSAVE` |
| `EPP_INDEX_POWERSAVE` | 4 | `HWP_EPP_POWERSAVE` (0xFF) |

#### Enum: Operation Mode (غير explicit، لكن مُمثَّل بـ string في sysfs)

| القيمة | الدلالة |
|---|---|
| `"active"` | الـ driver يستخدم `intel_pstate` driver struct |
| `"passive"` | الـ driver يستخدم `intel_cpufreq` driver struct |
| `"off"` | الـ driver مش مسجّل |

#### Fixed-Point Arithmetic Macros (Cheatsheet)

| Macro | الوظيفة |
|---|---|
| `FRAC_BITS = 8` | 8 bits للكسر في الـ fixed-point العادي |
| `EXT_BITS = 6` | 6 bits إضافية في الـ extended precision |
| `int_tofp(X)` | تحويل integer → fixed-point (`X << 8`) |
| `fp_toint(X)` | تحويل fixed-point → integer (`X >> 8`) |
| `mul_fp(x, y)` | ضرب fixed-point |
| `div_fp(x, y)` | قسمة fixed-point |
| `mul_ext_fp(x, y)` | ضرب extended precision (14-bit fraction) |
| `div_ext_fp(x, y)` | قسمة extended precision |

#### Hybrid Scaling Factors

| الجيل | الـ Factor |
|---|---|
| `ADL` (Alder Lake) | 78741 |
| `MTL` (Meteor Lake) | 80000 |
| `LNL` (Lunar Lake) | 86957 |
| Core (standard) | 100000 |

#### HWP Status Bits (MSR_HWP_STATUS)

| Bit | المعنى |
|---|---|
| `BIT(0)` | `HWP_GUARANTEED_PERF_CHANGE_STATUS` |
| `BIT(3)` | `HWP_HIGHEST_PERF_CHANGE_STATUS` |

#### HWP Interrupt Request Bits (MSR_HWP_INTERRUPT)

| Bit | المعنى |
|---|---|
| `BIT(0)` | `HWP_GUARANTEED_PERF_CHANGE_REQ` |
| `BIT(2)` | `HWP_HIGHEST_PERF_CHANGE_REQ` |

#### Sysfs Global Attributes

| الـ Attribute | R/W | الوظيفة |
|---|---|---|
| `status` | RW | active / passive / off |
| `no_turbo` | RW | 0 = turbo مسموح، 1 = ممنوع |
| `max_perf_pct` | RW | الحد الأقصى من الـ turbo max (%) |
| `min_perf_pct` | RW | الحد الأدنى من الـ turbo max (%) |
| `turbo_pct` | RO | نسبة الـ turbo range (غائب على hybrid) |
| `num_pstates` | RO | عدد الـ P-states (غائب على hybrid) |
| `hwp_dynamic_boost` | RW | I/O boost ديناميكي (HWP فقط) |
| `energy_efficiency` | RW | فعّل/عطّل DEC (Kaby/Coffee Lake Desktop فقط) |

---

### الـ Structs المهمة

#### 1. `struct sample`

**الغرض:** تخزين قراءة أداء واحدة بين callback وtالتالية.

```c
struct sample {
    int32_t core_avg_perf;  /* APERF/MPERF ratio — الأداء الفعلي المتوسط */
    int32_t busy_scaled;    /* busy fraction * 100 — للـ trace */
    u64 aperf;              /* الفرق في APERF منذ آخر sample */
    u64 mperf;              /* الفرق في MPERF منذ آخر sample */
    u64 tsc;                /* الفرق في TSC منذ آخر sample */
    u64 time;               /* وقت الـ sample من الـ scheduler */
};
```

**مدمج في** `struct cpudata` مباشرةً.

---

#### 2. `struct pstate_data`

**الغرض:** حدود وقيم الـ P-state لكل CPU.

```c
struct pstate_data {
    int current_pstate;       /* الـ P-state المطلوب حاليًا */
    int min_pstate;           /* أدنى P-state ممكن (من MSR) */
    int max_pstate;           /* أعلى P-state غير turbo (guaranteed) */
    int max_pstate_physical;  /* الـ physical max بدون TDP limits */
    int perf_ctl_scaling;     /* factor للـ PERF_CTL → frequency */
    int scaling;              /* الـ factor الفعلي (HWP قد يختلف) */
    int turbo_pstate;         /* أعلى turbo P-state (highest HWP perf) */
    unsigned int min_freq;    /* min_pstate × scaling بالـ kHz */
    unsigned int max_freq;    /* max_pstate × scaling بالـ kHz */
    unsigned int turbo_freq;  /* turbo_pstate × scaling بالـ kHz */
};
```

على الـ hybrid processors: `scaling != perf_ctl_scaling` لأن HWP يعرض performance levels أكتر من P-states.

---

#### 3. `struct vid_data`

**الغرض:** بيانات الجهد (VID) لمنصات Atom فقط — تُستخدم في `atom_get_val()`.

```c
struct vid_data {
    int min;         /* VID عند min_pstate (fixed-point) */
    int max;         /* VID عند max_pstate (fixed-point) */
    int turbo;       /* VID للـ turbo state */
    int32_t ratio;   /* (max-min) / (max_pstate - min_pstate) */
};
```

---

#### 4. `struct global_params`

**الغرض:** الإعدادات العالمية للـ driver (متزامنة بـ mutex).

```c
struct global_params {
    bool no_turbo;        /* هل turbo ممنوع حاليًا؟ */
    bool turbo_disabled;  /* هل turbo ممنوع من الـ BIOS (MSR)؟ */
    int max_perf_pct;     /* الحد الأقصى % (0-100) */
    int min_perf_pct;     /* الحد الأدنى % (0-100) */
};
```

الـ singleton: `static struct global_params global;`

---

#### 5. `struct cpudata`

**الغرض:** الـ per-CPU instance — أهم struct في الـ driver.

```c
struct cpudata {
    int cpu;                          /* رقم الـ CPU */
    unsigned int policy;              /* CPUFREQ_POLICY_PERFORMANCE أو POWERSAVE */
    struct update_util_data update_util; /* hook للـ scheduler callback */
    bool update_util_set;             /* هل الـ hook مسجّل؟ */

    struct pstate_data pstate;        /* P-state limits والقيم */
    struct vid_data vid;              /* VID (Atom فقط) */

    u64 last_update;                  /* وقت آخر update */
    u64 last_sample_time;             /* وقت آخر sample */
    u64 aperf_mperf_shift;           /* shift للـ KNL (10) */
    u64 prev_aperf;                   /* آخر قراءة APERF */
    u64 prev_mperf;                   /* آخر قراءة MPERF */
    u64 prev_tsc;                     /* آخر قراءة TSC */
    struct sample sample;             /* بيانات آخر sample */

    int32_t min_perf_ratio;           /* الحد الأدنى الفعلي (global + policy) */
    int32_t max_perf_ratio;           /* الحد الأقصى الفعلي (global + policy) */

    /* ACPI */
    struct acpi_processor_performance acpi_perf_data;
    bool valid_pss_table;

    /* HWP / EPP */
    unsigned int iowait_boost;        /* boost fraction للـ I/O waits */
    s16 epp_powersave;                /* EPP محفوظ عند التحول لـ performance */
    s16 epp_policy;                   /* آخر policy تم ضبط EPP عليه */
    s16 epp_default;                  /* قيمة EPP الافتراضية من الـ firmware */
    s16 epp_cached;                   /* آخر قيمة EPP مكتوبة */
    u64 hwp_req_cached;               /* cache لـ MSR_HWP_REQUEST */
    u64 hwp_cap_cached;               /* cache لـ MSR_HWP_CAPABILITIES */
    u64 last_io_update;               /* وقت آخر I/O wake event */

    /* Hybrid */
    unsigned int capacity_perf;       /* highest HWP perf للـ CAS */
    unsigned int sched_flags;         /* flags من الـ scheduler */
    u32 hwp_boost_min;                /* الـ min المؤقت أثناء I/O boost */
    bool suspended;                   /* هل الـ CPU في suspend؟ */
    bool pd_registered;               /* هل perf domain مسجّل؟ */
    struct delayed_work hwp_notify_work; /* workqueue للـ HWP interrupt */
};
```

الـ global array: `static struct cpudata **all_cpu_data;`

---

#### 6. `struct pstate_funcs`

**الغرض:** vtable لاختلافات الـ CPU models (Core vs Atom vs KNL).

```c
struct pstate_funcs {
    int (*get_max)(int cpu);               /* max non-turbo P-state */
    int (*get_max_physical)(int cpu);      /* physical max بدون TDP */
    int (*get_min)(int cpu);               /* min P-state */
    int (*get_turbo)(int cpu);             /* turbo P-state */
    int (*get_scaling)(void);              /* scaling factor (system-wide) */
    int (*get_cpu_scaling)(int cpu);       /* scaling factor (per-CPU, hybrid) */
    int (*get_aperf_mperf_shift)(void);    /* shift للـ KNL */
    u64 (*get_val)(struct cpudata*, int);  /* الـ MSR value للـ P-state */
    void (*get_vid)(struct cpudata *);     /* VID data (Atom فقط) */
};
```

الـ singleton: `static struct pstate_funcs pstate_funcs;`

الـ implementations:
- `core_funcs` → Sandy Bridge وما بعده
- `silvermont_funcs` → Atom Silvermont
- `airmont_funcs` → Atom Airmont
- `knl_funcs` → Xeon Phi KNL/KNM

---

### رسم علاقات الـ Structs (ASCII)

```
all_cpu_data[]
  │
  ├─► [0] struct cpudata ──────────────────────────────────────────┐
  │         │                                                       │
  │         ├── struct pstate_data (pstate)                        │
  │         │     ├── min_pstate / max_pstate / turbo_pstate        │
  │         │     ├── min_freq / max_freq / turbo_freq              │
  │         │     └── scaling / perf_ctl_scaling                    │
  │         │                                                       │
  │         ├── struct vid_data (vid)         [Atom only]           │
  │         │     └── min / max / turbo / ratio                     │
  │         │                                                       │
  │         ├── struct sample (sample)                              │
  │         │     ├── aperf / mperf / tsc                           │
  │         │     ├── core_avg_perf                                 │
  │         │     └── busy_scaled                                   │
  │         │                                                       │
  │         ├── struct update_util_data (update_util)               │
  │         │     └── func ptr → intel_pstate_update_util()         │
  │         │                 or intel_pstate_update_util_hwp()     │
  │         │                                                       │
  │         └── struct delayed_work (hwp_notify_work)               │
  │                   └── → intel_pstate_notify_work()              │
  │                                                                 │
  ├─► [1] struct cpudata ◄─────────────────────────────────────────┘
  └─► [N] struct cpudata


struct global_params (global)           struct pstate_funcs (pstate_funcs)
  ├── no_turbo                             ├── get_max()
  ├── turbo_disabled                       ├── get_min()
  ├── max_perf_pct                         ├── get_turbo()
  └── min_perf_pct                         ├── get_scaling()
                                           ├── get_val()
                                           └── get_vid()
                                                  ▲
                              ┌───────────────────┼───────────────────┐
                         core_funcs        silvermont_funcs       knl_funcs


intel_pstate_driver ──► struct cpufreq_driver
  ├── &intel_pstate  (active mode)
  │     ├── setpolicy = intel_pstate_set_policy
  │     ├── verify   = intel_pstate_verify_policy
  │     ├── init     = intel_pstate_cpu_init
  │     ├── online   = intel_pstate_cpu_online
  │     └── offline  = intel_pstate_cpu_offline
  │
  └── &intel_cpufreq (passive mode)
        ├── target      = intel_cpufreq_target
        ├── fast_switch = intel_cpufreq_fast_switch
        ├── adjust_perf = intel_cpufreq_adjust_perf
        └── init        = intel_cpufreq_cpu_init
```

---

### دورة حياة الـ Driver (Lifecycle Diagram)

```
boot / insmod
     │
     ▼
intel_pstate_init()
     │
     ├─ x86_match_cpu(hwp_support_ids) ─── CPU يدعم HWP?
     │       │Yes                               │No
     │       ▼                                  ▼
     │  hwp_active = true              x86_match_cpu(intel_pstate_cpu_ids)
     │  copy_cpu_funcs(&core_funcs)         │ Found?
     │  default_driver = &intel_pstate      │Yes → copy_cpu_funcs()
     │                                      │No  → return -ENODEV
     │
     ├─ vzalloc(all_cpu_data)
     ├─ intel_pstate_sysfs_expose_params()
     │
     ▼
intel_pstate_register_driver(default_driver)
     │
     ├─ global.max_perf_pct = 100
     ├─ global.no_turbo = turbo_is_disabled()
     ├─ cpufreq_register_driver()
     │       │
     │       ▼  [لكل CPU]
     │   intel_pstate_cpu_init() / intel_cpufreq_cpu_init()
     │       │
     │       ├─ intel_pstate_init_cpu()
     │       │     ├─ kzalloc(struct cpudata)
     │       │     ├─ intel_pstate_hwp_enable()   [HWP]
     │       │     └─ intel_pstate_get_cpu_pstates()
     │       │           ├─ pstate_funcs.get_min/max/turbo()
     │       │           ├─ __intel_pstate_get_hwp_cap()  [HWP]
     │       │           └─ intel_pstate_hybrid_hwp_adjust() [hybrid]
     │       │
     │       └─ cpuinfo.min/max_freq ← pstate data
     │
     ▼
[Driver Running]


CPU goes OFFLINE
     │
     ▼
intel_pstate_cpu_offline()
  or intel_cpufreq_cpu_offline()
     │
     ├─ intel_pstate_clear_update_util_hook()
     ├─ intel_pstate_hwp_offline()  [HWP]
     │     ├─ disable HWP interrupt
     │     ├─ set hwp_min = hwp_max = lowest_perf
     │     └─ hybrid_clear_cpu_capacity()
     └─ intel_pstate_set_min_pstate()  [non-HWP]


CPU goes ONLINE (again)
     │
     ▼
intel_pstate_cpu_online()
     ├─ intel_pstate_hwp_reenable()
     ├─ cpu->suspended = false
     └─ hybrid_update_capacity()


suspend
     │
     ▼
intel_pstate_suspend()
  ├─ cpu->suspended = true
  └─ intel_pstate_disable_hwp_interrupt()

resume
     │
     ▼
intel_pstate_resume()
  ├─ restore power_ctl_ee_state
  └─ intel_pstate_hwp_reenable()  [if suspended & HWP]


sysfs write "off" / "passive" / "active"
     │
     ▼
intel_pstate_update_status()
  ├─ cpufreq_unregister_driver()
  ├─ intel_pstate_driver_cleanup()  → kfree(all_cpu_data[cpu])
  └─ intel_pstate_register_driver() [إن كان active/passive]
```

---

### Call Flow Diagrams

#### A. Active Mode — P-state Selection (non-HWP, powersave)

```
CPU scheduler (every tick)
     │
     ▼
cpufreq_update_util()
     │
     ▼
update_util_data.func()
  = intel_pstate_update_util(data, time, flags)
     │
     ├─ check: smp_processor_id() == cpu->cpu?  [local only]
     ├─ iowait_boost calculation
     ├─ delta_ns < 10ms? → return (throttle)
     │
     ▼
intel_pstate_sample(cpu, time)
     ├─ rdmsrq(MSR_IA32_APERF)
     ├─ rdmsrq(MSR_IA32_MPERF)
     ├─ rdtsc()
     ├─ cpu->sample.aperf = aperf - prev_aperf
     └─ intel_pstate_calc_avg_perf()
           └─ core_avg_perf = APERF/MPERF (extended fixed-point)

     │
     ▼
intel_pstate_adjust_pstate(cpu)
     │
     ▼
get_target_pstate(cpu)
     ├─ busy_frac = mperf/tsc  (actual CPU busy fraction)
     ├─ busy_frac = max(busy_frac, iowait_boost)
     ├─ target = turbo_max * 1.25 * busy_frac
     └─ smooth: if avg_pstate > target → target += (avg-target)/2

     │
     ▼
intel_pstate_prepare_request(cpu, target_pstate)
     └─ clamp(target, min_perf_ratio, max_perf_ratio)

     │
     ▼
intel_pstate_update_pstate(cpu, pstate)
     └─ wrmsrq(MSR_IA32_PERF_CTL, pstate_funcs.get_val(cpu, pstate))
```

#### B. Active Mode — HWP with Dynamic Boost (I/O path)

```
CPU scheduler detects I/O wakeup
     │
     ▼
intel_pstate_update_util_hwp(data, time, SCHED_CPUFREQ_IOWAIT)
     │
     ├─ cpu->sched_flags |= IOWAIT
     └─ [local CPU] intel_pstate_update_util_hwp_local(cpu, time)
           │
           ├─ two consecutive IOWAIT ticks? → do_io = true
           └─ intel_pstate_hwp_boost_up(cpu)
                 │
                 ├─ read hwp_req_cached
                 ├─ boost levels: (min+guaranteed)/2 → guaranteed → turbo
                 └─ wrmsrq(MSR_HWP_REQUEST, boosted_value)

[after hwp_boost_hold_time_ns = 3ms of idle]
     │
     ▼
intel_pstate_hwp_boost_down(cpu)
     └─ wrmsrq(MSR_HWP_REQUEST, hwp_req_cached)  [restore]
```

#### C. Passive Mode — Governor → Driver → Hardware

```
schedutil governor
     │
     ▼
intel_cpufreq_target(policy, target_freq, relation)
     │
     ├─ intel_pstate_freq_to_hwp_rel(cpu, freq, relation)
     │     └─ freq / cpu->pstate.scaling
     │
     ▼
intel_cpufreq_update_pstate(policy, target_pstate, fast_switch=false)
     │
     ├─ intel_pstate_prepare_request(cpu, pstate)
     │     └─ clamp(pstate, min_perf_ratio, max_perf_ratio)
     │
     ├─ [HWP]  intel_cpufreq_hwp_update(cpu, min, max, desired, false)
     │           └─ wrmsrq_on_cpu(MSR_HWP_REQUEST, value)
     │
     └─ [!HWP] intel_cpufreq_perf_ctl_update(cpu, pstate, false)
                 └─ wrmsrq_on_cpu(MSR_IA32_PERF_CTL, get_val())

[Fast Switch from scheduler context]
     │
     ▼
intel_cpufreq_fast_switch(policy, target_freq)
     └─ intel_cpufreq_update_pstate(policy, pstate, fast_switch=true)
           └─ wrmsrq(MSR_HWP_REQUEST, value)  [local CPU, no IPI]
```

#### D. HWP Capability Change Notification

```
Hardware fires HWP interrupt
     │
     ▼
notify_hwp_interrupt()  [called from thermal interrupt handler]
     │
     ├─ rdmsrq_safe(MSR_HWP_STATUS)
     ├─ check status_mask bits
     ├─ raw_spin_lock(hwp_notify_lock)
     ├─ cpumask_test_cpu(this_cpu, hwp_intr_enable_mask)?
     └─ schedule_delayed_work(&cpudata->hwp_notify_work, 10ms)

     │
     ▼  [workqueue context, after 10ms]
intel_pstate_notify_work(work)
     │
     ├─ intel_pstate_update_max_freq(cpudata)
     │     ├─ intel_pstate_get_hwp_cap(cpu)
     │     │     └─ rdmsrq(MSR_HWP_CAPABILITIES)
     │     └─ refresh_frequency_limits(policy)
     │
     ├─ hybrid_update_capacity(cpudata)
     └─ wrmsrq(MSR_HWP_STATUS, 0)  [ack interrupt]
```

#### E. Driver Registration / Mode Switch

```
sysfs: echo "passive" > status
     │
     ▼
store_status() [holds intel_pstate_driver_lock]
     │
     ▼
intel_pstate_update_status("passive", 7)
     │
     ├─ cpufreq_unregister_driver(intel_pstate)
     ├─ intel_pstate_sysfs_hide_hwp_dynamic_boost()
     └─ intel_pstate_register_driver(&intel_cpufreq)
           │
           ├─ global reset (max_perf_pct=100, etc.)
           ├─ intel_pstate_driver = &intel_cpufreq
           ├─ cpufreq_register_driver(&intel_cpufreq)
           │     └─ intel_cpufreq_cpu_init() × N CPUs
           └─ hybrid_init_cpu_capacity_scaling()
```

---

### استراتيجية الـ Locking

الـ driver يستخدم ثلاث آليات تزامن مختلفة بحسب السياق:

#### الـ Mutexes

| الـ Mutex | يحمي | متى يُكتسب |
|---|---|---|
| `intel_pstate_driver_lock` | `intel_pstate_driver` pointer، تسجيل/إلغاء تسجيل الـ driver، global mode changes | sysfs handlers، `intel_pstate_update_status()` |
| `intel_pstate_limits_lock` | `global.min/max_perf_pct`، EPP writes، `intel_pstate_set_policy()` | sysfs `store_*` handlers، `intel_pstate_set_policy()` |
| `hybrid_capacity_lock` | `hybrid_max_perf_cpu`، `capacity_perf` fields في `cpudata`، x86 scale-invariance info | `hybrid_update_capacity()`، `hybrid_refresh_cpu_capacity_scaling()` |

#### الـ Spinlocks

| الـ Spinlock | يحمي | ملاحظات |
|---|---|---|
| `hwp_notify_lock` (raw) | `hwp_intr_enable_mask` cpumask | يُكتسب من interrupt context في `notify_hwp_interrupt()` |

#### الـ READ_ONCE / WRITE_ONCE (Lockless)

- **`cpu->hwp_req_cached`**: يُقرأ بـ `READ_ONCE` من `intel_pstate_hwp_boost_up/down()` اللي تشتغل من scheduler context بدون lock. يُكتب بـ `WRITE_ONCE` من `intel_pstate_hwp_set()` اللي تشتغل تحت `intel_pstate_limits_lock`.
- **`cpu->hwp_cap_cached`**: يُكتب بـ `WRITE_ONCE` من `__intel_pstate_get_hwp_cap()`. يُقرأ بـ `READ_ONCE` من أماكن متعددة.
- **`global.no_turbo`**: يُقرأ بـ `READ_ONCE` من scheduler path بدون lock.
- **`all_cpu_data[cpu]`**: يُكتب بـ `WRITE_ONCE` عند الـ init والـ cleanup.

#### قواعد ترتيب الـ Locks (Lock Ordering)

```
intel_pstate_driver_lock
     └── intel_pstate_limits_lock
             └── hybrid_capacity_lock
                     └── hwp_notify_lock (raw spinlock, IRQ-safe)
```

**لا يجوز** اكتساب lock أعلى في الـ hierarchy وأنت ماسك lock أدنى.

#### الـ Policy R/W Semaphore (من CPUFreq Core)

الـ `cpufreq_policy` له semaphore خاص به يُمسكه الـ core. الـ `store_energy_performance_preference()` يعمل مع الـ policy R/W semaphore ممسوك. الـ `intel_pstate_limits_lock` يُكتسب تحته مباشرةً بدون مشكلة لأنه دائمًا في الترتيب الأدنى.

#### خلاصة اللي يحتاج lock ومالوش:

| العملية | الآلية |
|---|---|
| قراءة/كتابة `global.no_turbo` | `intel_pstate_driver_lock` + `intel_pstate_limits_lock` للكتابة، `READ_ONCE` للقراءة من fast path |
| كتابة `MSR_HWP_REQUEST` في I/O boost | `READ_ONCE(hwp_req_cached)` بدون lock (scheduler hot path) |
| تغيير EPP | `intel_pstate_limits_lock` |
| تغيير mode (active/passive/off) | `intel_pstate_driver_lock` |
| قراءة HWP capabilities | بدون lock (per-CPU، يُكتب من CPU نفسه أو workqueue) |
| hybrid capacity update | `hybrid_capacity_lock` |
| HWP interrupt mask | `hwp_notify_lock` (raw spinlock) |
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ Functions والـ APIs

#### Fixed-Point Math Helpers

| Function | Signature | الغرض |
|---|---|---|
| `mul_fp` | `int32_t mul_fp(int32_t x, int32_t y)` | ضرب fixed-point (8 bits fraction) |
| `div_fp` | `int32_t div_fp(s64 x, s64 y)` | قسمة fixed-point (8 bits fraction) |
| `ceiling_fp` | `int ceiling_fp(int32_t x)` | تقريب لأعلى fixed-point |
| `mul_ext_fp` | `u64 mul_ext_fp(u64 x, u64 y)` | ضرب extended fixed-point (14 bits fraction) |
| `div_ext_fp` | `u64 div_ext_fp(u64 x, u64 y)` | قسمة extended fixed-point (14 bits fraction) |

#### P-State Initialization & Discovery

| Function | الغرض |
|---|---|
| `intel_pstate_init` | Entry point — تهيئة الـ driver كله |
| `intel_pstate_init_cpu` | تهيئة per-CPU state |
| `intel_pstate_get_cpu_pstates` | قراءة P-state limits من MSRs |
| `copy_cpu_funcs` | نسخ model-specific callbacks للـ `pstate_funcs` |
| `intel_pstate_msrs_not_valid` | تحقق من MSRs صحيحة |
| `intel_pstate_setup` | early_param parser لـ cmdline |

#### HWP Management

| Function | الغرض |
|---|---|
| `intel_pstate_hwp_enable` | تفعيل HWP على CPU معين |
| `intel_pstate_hwp_set` | كتابة min/max/EPP في `MSR_HWP_REQUEST` |
| `intel_pstate_hwp_offline` | تهيئة HWP قبل offline للـ CPU |
| `intel_pstate_hwp_reenable` | إعادة تفعيل HWP بعد resume/online |
| `__intel_pstate_get_hwp_cap` | قراءة `MSR_HWP_CAPABILITIES` |
| `intel_pstate_get_hwp_cap` | قراءة HWP cap + حساب الـ frequencies |
| `intel_pstate_hwp_is_enabled` | فحص `MSR_PM_ENABLE` bit 0 |

#### HWP Interrupt / Notification

| Function | الغرض |
|---|---|
| `intel_pstate_enable_hwp_interrupt` | تفعيل HWP guaranteed-perf-change interrupt |
| `intel_pstate_disable_hwp_interrupt` | تعطيل HWP interrupt + cancel pending work |
| `notify_hwp_interrupt` | ISR handler لـ HWP notification |
| `intel_pstate_notify_work` | work handler يُنفَّذ في process context |

#### EPP (Energy Performance Preference)

| Function | الغرض |
|---|---|
| `intel_pstate_get_epp` | قراءة EPP من `MSR_HWP_REQUEST` bits [31:24] |
| `intel_pstate_set_epp` | كتابة EPP في `MSR_HWP_REQUEST` |
| `intel_pstate_get_energy_pref_index` | ترجمة EPP رقم → index في `energy_perf_strings` |
| `intel_pstate_set_energy_pref_index` | ضبط EPP من sysfs index أو raw value |
| `intel_pstate_update_epp_defaults` | ضبط الـ EPP defaults حسب الـ CPU gen |

#### P-State Selection (Active Mode without HWP)

| Function | الغرض |
|---|---|
| `intel_pstate_sample` | قراءة APERF/MPERF/TSC وحساب الـ delta |
| `intel_pstate_calc_avg_perf` | حساب `APERF/MPERF` ratio |
| `get_target_pstate` | حساب الـ P-state المطلوب من الـ utilization |
| `intel_pstate_prepare_request` | clamp الـ target بحدود min/max |
| `intel_pstate_update_pstate` | كتابة P-state جديد في `MSR_IA32_PERF_CTL` |
| `intel_pstate_adjust_pstate` | الـ main powersave algorithm — sample + adjust |
| `intel_pstate_update_util` | utilization callback (scheduler context) |

#### HWP Boost (iowait boost)

| Function | الغرض |
|---|---|
| `intel_pstate_hwp_boost_up` | رفع HWP min بصورة تدريجية عند IO |
| `intel_pstate_hwp_boost_down` | تخفيض HWP min بعد hold_time |
| `intel_pstate_update_util_hwp_local` | معالجة iowait boost محلياً على نفس CPU |
| `intel_pstate_update_util_hwp` | utilization callback لـ HWP mode |

#### Passive Mode (intel_cpufreq)

| Function | الغرض |
|---|---|
| `intel_cpufreq_target` | target callback — normal path |
| `intel_cpufreq_fast_switch` | fast_switch callback — scheduler context |
| `intel_cpufreq_adjust_perf` | EAS adjust_perf callback |
| `intel_cpufreq_update_pstate` | منطق الـ update المشترك بين target/fast_switch |
| `intel_cpufreq_hwp_update` | كتابة min/max/desired في `MSR_HWP_REQUEST` |
| `intel_cpufreq_perf_ctl_update` | كتابة `MSR_IA32_PERF_CTL` |
| `intel_cpufreq_verify_policy` | verify + update limits في passive mode |
| `intel_cpufreq_cpu_init` | init callback — passive mode |
| `intel_cpufreq_cpu_exit` | exit callback — cleanup freq-qos |
| `intel_cpufreq_cpu_offline` | offline callback |
| `intel_cpufreq_suspend` | suspend callback |

#### Hybrid Processor Support

| Function | الغرض |
|---|---|
| `hybrid_get_capacity_perf` | قراءة highest HWP perf لتمثيل الـ capacity |
| `hybrid_set_cpu_capacity` | ضبط `arch_scale_cpu_capacity` لكل CPU |
| `hybrid_clear_cpu_capacity` | إعادة ضبط capacity لـ 1 (flat) |
| `hybrid_update_cpu_capacity_scaling` | إعادة حساب أكبر CPU performance وتحديث الكل |
| `hybrid_refresh_cpu_capacity_scaling` | تحديث كامل مع mutex |
| `hybrid_init_cpu_capacity_scaling` | تفعيل CAS عند init أو mode switch |
| `hybrid_update_capacity` | تحديث capacity لـ CPU واحد بعد HWP cap تغيير |
| `intel_pstate_set_itmt_prio` | ضبط ITMT priority من CPPC |
| `hybrid_register_perf_domain` | تسجيل EM perf domain لـ CPU |
| `hybrid_register_all_perf_domains` | تسجيل كل الـ perf domains |
| `intel_pstate_hybrid_hwp_adjust` | تصحيح HWP perf levels على hybrid CPUs |
| `hwp_get_cpu_scaling` | حساب scaling factor للـ CPU type |

#### Sysfs Interface

| Function / Attr | الوصف |
|---|---|
| `show_status` / `store_status` | قراءة/كتابة "active"/"passive"/"off" |
| `show_no_turbo` / `store_no_turbo` | تفعيل/تعطيل turbo |
| `show_max_perf_pct` / `store_max_perf_pct` | global max performance limit % |
| `show_min_perf_pct` / `store_min_perf_pct` | global min performance limit % |
| `show_turbo_pct` | نسبة turbo range |
| `show_num_pstates` | عدد الـ P-states |
| `show_hwp_dynamic_boost` / `store_hwp_dynamic_boost` | تفعيل/تعطيل iowait boost |
| `show_energy_efficiency` / `store_energy_efficiency` | كنترول MSR_IA32_POWER_CTL EE bit |
| `show_energy_performance_preference` / `store_energy_performance_preference` | قراءة/كتابة EPP |
| `show_base_frequency` | الـ guaranteed (non-turbo) frequency |
| `intel_pstate_sysfs_expose_params` | إنشاء `/sys/devices/system/cpu/intel_pstate/` |
| `intel_pstate_sysfs_remove` | حذف الـ sysfs entries |

#### Driver Registration / Lifecycle

| Function | الغرض |
|---|---|
| `intel_pstate_register_driver` | تسجيل الـ driver مع CPUFreq core |
| `intel_pstate_driver_cleanup` | حذف per-cpu data + reset driver pointer |
| `intel_pstate_update_status` | معالجة write لـ `status` sysfs attr |
| `intel_pstate_show_status` | قراءة الـ status الحالي |
| `intel_pstate_cpu_init` | CPUFreq init callback (active mode) |
| `intel_pstate_cpu_online` | online callback |
| `intel_pstate_cpu_offline` | offline callback (active mode) |
| `intel_pstate_cpu_exit` | exit callback |
| `intel_pstate_set_policy` | setpolicy callback (active mode) |
| `intel_pstate_verify_policy` | verify callback |
| `intel_pstate_suspend` | suspend callback |
| `intel_pstate_resume` | resume callback |
| `intel_pstate_update_limits` | update_limits callback |

#### ACPI Helpers

| Function | الغرض |
|---|---|
| `intel_pstate_init_acpi_perf_limits` | تسجيل ACPI `_PSS` performance data |
| `intel_pstate_exit_perf_limits` | إلغاء تسجيل ACPI performance data |
| `intel_pstate_acpi_pm_profile_server` | فحص لو الـ PM profile server |
| `intel_pstate_platform_pwr_mgmt_exists` | فحص لو firmware يتحكم في P-states |
| `intel_pstate_request_control_from_smm` | طلب SMM يتخلى عن تحكم P-states |
| `intel_pstate_get_cppc_guaranteed` | قراءة CPPC guaranteed performance |
| `intel_pstate_cppc_get_scaling` | حساب scaling factor من CPPC nominal data |

#### Model-Specific Callbacks

| Function | يُستخدم في |
|---|---|
| `core_get_min_pstate` | Core CPUs — MSR_PLATFORM_INFO bits [47:40] |
| `core_get_max_pstate` | Core CPUs — حساب max مع TDP |
| `core_get_max_pstate_physical` | Core CPUs — MSR_PLATFORM_INFO bits [15:8] |
| `core_get_turbo_pstate` | Core CPUs — MSR_TURBO_RATIO_LIMIT |
| `core_get_val` | Core CPUs — بناء PERF_CTL value |
| `core_get_scaling` | Core CPUs — ثابت 100000 KHz |
| `atom_get_min_pstate` | Atom — MSR_ATOM_CORE_RATIOS bits [14:8] |
| `atom_get_max_pstate` | Atom — MSR_ATOM_CORE_RATIOS bits [22:16] |
| `atom_get_turbo_pstate` | Atom — MSR_ATOM_CORE_TURBO_RATIOS bits [6:0] |
| `atom_get_val` | Atom — PERF_CTL مع VID |
| `atom_get_vid` | Atom — قراءة VID min/max/turbo |
| `silvermont_get_scaling` | Silvermont — MSR_FSB_FREQ table |
| `airmont_get_scaling` | Airmont — MSR_FSB_FREQ table أكبر |
| `knl_get_turbo_pstate` | Knights Landing — bits [15:8] في MSR_TURBO_RATIO_LIMIT |
| `knl_get_aperf_mperf_shift` | KNL — shift=10 للتعويض عن counting difference |
| `hwp_get_cpu_scaling` | HWP — hybrid-aware scaling factor |

---

### Group 1: Fixed-Point Math Helpers

الـ driver يتجنب floating-point في kernel context تماماً، فيستخدم fixed-point arithmetic مع precision مُعرَّفة بـ macros:

- **`FRAC_BITS = 8`** → القسمة العادية، القيمة الحقيقية = `raw >> 8`
- **`EXT_FRAC_BITS = 14`** → extended precision لحسابات الـ `APERF/MPERF`

#### `mul_fp`

```c
static inline int32_t mul_fp(int32_t x, int32_t y)
```

بتضرب قيمتين fixed-point معاً وترجع النتيجة مع تصحيح الـ scale. بتعمل الضرب في `int64_t` عشان تتجنب overflow، وبعدين بتعمل right shift بـ FRAC_BITS عشان ترجع للـ scale الأصلي.

- **`x`, `y`**: قيم fixed-point (fraction = 8 bits)
- **Return**: نتيجة الضرب fixed-point
- **Key detail**: لازم cast لـ `int64_t` قبل الضرب عشان `int32_t * int32_t` ممكن يتجاوز 32-bit

#### `div_fp`

```c
static inline int32_t div_fp(s64 x, s64 y)
```

بتقسم `x` على `y` بدقة fixed-point. بتعمل left shift لـ x بـ FRAC_BITS قبل القسمة عشان تحافظ على الدقة. بتستخدم `div64_s64` من `<asm/div64.h>` عشان الـ 64-bit division مش native في 32-bit kernels.

- **`x`**: البسط
- **`y`**: المقام — لو صفر، undefined behavior
- **Return**: حاصل القسمة fixed-point

#### `ceiling_fp`

```c
static inline int ceiling_fp(int32_t x)
```

بترجع أصغر integer أكبر من أو يساوي `x` (fixed-point). بتفحص لو في fractional bits تخلي التقريب لأعلى. مستخدمة في `atom_get_val` لحساب VID.

#### `mul_ext_fp` / `div_ext_fp`

```c
static inline u64 mul_ext_fp(u64 x, u64 y)
static inline u64 div_ext_fp(u64 x, u64 y)
```

نفس فكرة `mul_fp`/`div_fp` بس بـ extended precision (14 bits). بتُستخدم في حسابات `APERF/MPERF` ratio اللي بتحتاج دقة أعلى. `get_avg_frequency` و`intel_pstate_calc_avg_perf` بيستخدموهم.

---

### Group 2: Initialization & Boot

#### `intel_pstate_init` — device_initcall

```c
static int __init intel_pstate_init(void)
```

الـ entry point الأساسي للـ driver. بيتشغل مرة واحدة وقت boot. الخطوات:

```
1. فحص vendor == Intel
2. intel_pstate_platform_pwr_mgmt_exists() → لو firmware يتحكم، exit
3. x86_match_cpu(hwp_support_ids) → فحص HWP support
   ├─ HWP موجود:
   │   ├─ hwp_check_epp() → فحص EPP feature
   │   ├─ intel_pstate_hwp_is_enabled() → هل BIOS فعّل HWP؟
   │   ├─ copy_cpu_funcs(&core_funcs)
   │   └─ hwp_active = true
   └─ HWP مش موجود:
       └─ x86_match_cpu(intel_pstate_cpu_ids) → بحث في قائمة الـ supported CPUs
4. vzalloc(all_cpu_data) → تخصيص pointer array لكل CPU
5. intel_pstate_sysfs_expose_params() → إنشاء /sys entries
6. تحديث epp_values وhybrid_scaling_factor حسب CPU model
7. intel_pstate_register_driver(default_driver)
```

- **Return**: `0` success، `-ENODEV` إذا مش مدعوم
- **Context**: init thread فقط، `__init` → يُحذف من memory بعد boot
- **Locking**: `intel_pstate_driver_lock` mutex حول التسجيل

#### `intel_pstate_setup` — early_param

```c
static int __init intel_pstate_setup(char *str)
```

يُعالج `intel_pstate=<option>` من الـ kernel cmdline. بيتشغل قبل `intel_pstate_init`. بيضبط flags زي `no_load`, `no_hwp`, `force_load`, `per_cpu_limits`, `no_cas`.

#### `intel_pstate_init_cpu`

```c
static int intel_pstate_init_cpu(unsigned int cpunum)
```

بينشئ أو يُعيد تهيئة `struct cpudata` لـ CPU معين. لو أول مرة، بيعمل `kzalloc` وبيفعّل HWP إذا كان active. لو CPU راجع من suspend/offline، بيعمل `intel_pstate_hwp_reenable`.

```c
cpu = kzalloc(sizeof(*cpu), GFP_KERNEL);
cpu->epp_default = -EINVAL;  /* يُعبأ لاحقاً */
if (hwp_active)
    intel_pstate_hwp_enable(cpu);
intel_pstate_get_cpu_pstates(cpu);
```

- **`cpunum`**: رقم الـ CPU
- **Return**: `0` أو `-ENOMEM`
- **Context**: CPUFreq policy init — process context

#### `intel_pstate_get_cpu_pstates`

```c
static void intel_pstate_get_cpu_pstates(struct cpudata *cpu)
```

يملأ `cpu->pstate` بالـ min/max/turbo P-states وحساب الـ frequencies. بيستدعي `pstate_funcs` callbacks. على HWP: بيقرأ `MSR_HWP_CAPABILITIES` وبيعمل `intel_pstate_hybrid_hwp_adjust` لو hybrid.

```
perf_ctl_scaling = pstate_funcs.get_scaling()
max_pstate_physical = pstate_funcs.get_max_physical()
min_pstate = pstate_funcs.get_min()

if (hwp_active):
    __intel_pstate_get_hwp_cap(cpu)        /* MSR_HWP_CAPABILITIES */
    cpu->pstate.scaling = get_cpu_scaling()
    intel_pstate_hybrid_hwp_adjust(cpu)    /* لو hybrid */
else:
    cpu->pstate.max_pstate = get_max()
    cpu->pstate.turbo_pstate = get_turbo()

intel_pstate_set_min_pstate(cpu)           /* اكتب min في PERF_CTL */
```

#### `copy_cpu_funcs`

```c
static void __init copy_cpu_funcs(struct pstate_funcs *funcs)
```

بتنسخ model-specific function pointers من جدول `intel_pstate_cpu_ids` للـ global `pstate_funcs`. بيتشغل مرة واحدة. كل الـ callbacks بعد كده بتكون indirect calls عبر `pstate_funcs`.

---

### Group 3: Model-Specific P-State Discovery (Core & Atom)

#### Core CPU Functions

##### `core_get_min_pstate`

```c
static int core_get_min_pstate(int cpu)
```

يقرأ `MSR_PLATFORM_INFO` bits [47:40] — الـ LFM (Lowest Frequency Mode) ratio. القيمة بتُضرب في `perf_ctl_scaling` (100 MHz) عشان تجيب الـ frequency.

##### `core_get_max_pstate_physical`

```c
static int core_get_max_pstate_physical(int cpu)
```

يقرأ `MSR_PLATFORM_INFO` bits [15:8] — الـ HFM (Highest Frequency Mode) ratio بدون TDP adjustment. الفرق عن `core_get_max_pstate` إنه ما بيراعيش Configurable TDP.

##### `core_get_max_pstate`

```c
static int core_get_max_pstate(int cpu)
```

أكثر تعقيداً من `core_get_max_pstate_physical`. بيفحص TDP configuration:

```
rdmsr(MSR_PLATFORM_INFO) → max_pstate
core_get_tdp_ratio() → tdp_ratio

if tdp_ratio <= 0: return max_pstate  /* لا Configurable TDP */

if hwp_active: return tdp_ratio       /* HWP: استخدم TDP ratio مباشرة */

/* بدون HWP: فحص TAC (Turbo Activation Ratio) */
rdmsr(MSR_TURBO_ACTIVATION_RATIO) → tar
if (tdp_ratio - 1 == tar): max_pstate = tar
return max_pstate
```

##### `core_get_turbo_pstate`

```c
static int core_get_turbo_pstate(int cpu)
```

يقرأ `MSR_TURBO_RATIO_LIMIT` bits [7:0] — أعلى one-core turbo ratio. بيضمن إن الـ turbo >= non-turbo max.

##### `core_get_val`

```c
static u64 core_get_val(struct cpudata *cpudata, int pstate)
```

ببساطة بيستدعي `get_perf_ctl_val(pstate)` اللي بيضع الـ ratio في bits [15:8] من `MSR_IA32_PERF_CTL`. لو `no_turbo` flag مضبوط وفيه IDA support، بيضع bit 32 للـ IDA disable.

#### Atom CPU Functions

##### `atom_get_val`

```c
static u64 atom_get_val(struct cpudata *cpudata, int pstate)
```

فرق عن Core: الـ Atom بيحتاج VID (Voltage ID) كمان في نفس الـ register.

```
val = get_perf_ctl_val(pstate)

/* حساب VID بـ linear interpolation */
vid_fp = vid.min + (pstate - min_pstate) * vid.ratio
vid_fp = clamp(vid_fp, vid.min, vid.max)
vid = ceiling_fp(vid_fp)

if pstate > max_pstate: vid = vid.turbo  /* Turbo VID ثابت */

return val | vid
```

##### `silvermont_get_scaling` / `airmont_get_scaling`

```c
static int silvermont_get_scaling(void)
static int airmont_get_scaling(void)
```

بيقرأ `MSR_FSB_FREQ` عشان يحدد الـ base clock. الـ Silvermont عنده 5 قيم، الـ Airmont عنده 9. النتيجة بالـ KHz (مثلاً 100000 = 100 MHz).

---

### Group 4: HWP Management

#### `intel_pstate_hwp_enable`

```c
static void intel_pstate_hwp_enable(struct cpudata *cpudata)
```

يُفعّل HWP لـ CPU واحد. بيكتب `MSR_PM_ENABLE = 0x1`. بيُعطّل أولاً الـ HWP interrupt ثم بيُعيد تفعيله بشكل صحيح. بيضبط الـ EPP defaults.

- **Side effect**: بعد كتابة `MSR_PM_ENABLE = 1`، الـ BIOS/MSRs المتعلقة بالـ P-state بتتغير
- **يُستدعى من**: `intel_pstate_init_cpu` (first time) أو `intel_pstate_hwp_reenable` (resume)

#### `intel_pstate_hwp_set`

```c
static void intel_pstate_hwp_set(unsigned int cpu)
```

الـ function المحورية لتحديث `MSR_HWP_REQUEST`. بتُستدعى كل مرة يتغير فيها الـ policy أو الـ limits.

```
max = cpu_data->max_perf_ratio
min = cpu_data->min_perf_ratio
if policy == PERFORMANCE: min = max  /* لا variability */

value = rdmsr(MSR_HWP_REQUEST)
value[HWP_MIN] = min
value[HWP_MAX] = max

/* EPP handling */
if policy تغير:
    if → PERFORMANCE:
        epp_powersave = current_epp
        epp = 0  /* performance EPP */
    elif → POWERSAVE:
        epp = epp_powersave (القيمة المحفوظة)
    كتابة EPP في bits [31:24]

wrmsrq_on_cpu(cpu, MSR_HWP_REQUEST, value)
```

- **Locking**: يُستدعى تحت `intel_pstate_limits_lock`
- **يُستدعى من**: `intel_pstate_set_policy`

#### `__intel_pstate_get_hwp_cap`

```c
static void __intel_pstate_get_hwp_cap(struct cpudata *cpu)
```

يقرأ `MSR_HWP_CAPABILITIES` ويخزن:
- `hwp_cap_cached`: القيمة الخام (يُستخدم `WRITE_ONCE`)
- `pstate.max_pstate` = `HWP_GUARANTEED_PERF(cap)` — bits [15:8]
- `pstate.turbo_pstate` = `HWP_HIGHEST_PERF(cap)` — bits [7:0]

الـ `WRITE_ONCE`/`READ_ONCE` هنا مهمة لأن `hwp_cap_cached` ممكن يُقرأ من contexts تانية (IRQ handler مثلاً).

#### `intel_pstate_hwp_offline`

```c
static void intel_pstate_hwp_offline(struct cpudata *cpu)
```

قبل ما الـ CPU يروح offline، لازم نُعيد الـ HWP لحالة آمنة عشان ما يأثرش على الـ sibling threads.

```
intel_pstate_disable_hwp_interrupt(cpu)

/* استرجاع cached EPP (مش performance=0) */
value[EPP] = cpu->epp_cached
cpu->epp_policy = CPUFREQ_POLICY_UNKNOWN  /* سيُعاد ضبطه عند online */

/* clear desired perf */
value &= ~HWP_DESIRED_PERF(~0L)

/* اضبط max = min = lowest perf */
min_perf = HWP_LOWEST_PERF(hwp_cap_cached)
value[HWP_MAX] = min_perf
value[HWP_MIN] = min_perf
value[EPP] = HWP_EPP_POWERSAVE  /* أقصى توفير للطاقة */

wrmsrq_on_cpu(MSR_HWP_REQUEST, value)

/* تحديث hybrid capacity scaling */
if cpu == hybrid_max_perf_cpu:
    hybrid_update_cpu_capacity_scaling()
hybrid_clear_cpu_capacity(cpu->cpu)
```

---

### Group 5: HWP Interrupt & Notification

#### `intel_pstate_enable_hwp_interrupt`

```c
static void intel_pstate_enable_hwp_interrupt(struct cpudata *cpudata)
```

بيُفعّل HWP interrupt لاستقبال إشعار لما الـ guaranteed performance يتغير (مثلاً لما thermal throttling يحصل أو turbo الـ budget يتغير).

```
INIT_DELAYED_WORK(&cpudata->hwp_notify_work, intel_pstate_notify_work)
cpumask_set_cpu(cpu, &hwp_intr_enable_mask)
interrupt_mask = HWP_GUARANTEED_PERF_CHANGE_REQ  /* bit 0 */
if X86_FEATURE_HWP_HIGHEST_PERF_CHANGE:
    interrupt_mask |= HWP_HIGHEST_PERF_CHANGE_REQ  /* bit 2 */
wrmsrq_on_cpu(MSR_HWP_INTERRUPT, interrupt_mask)
wrmsrq_on_cpu(MSR_HWP_STATUS, 0)  /* clear pending */
```

- **Locking**: `hwp_notify_lock` raw spinlock (IRQ context safe)

#### `notify_hwp_interrupt`

```c
void notify_hwp_interrupt(void)
```

الـ ISR — بيتشغل في interrupt context. بيُقرأ `MSR_HWP_STATUS`، لو في change نافع، بيُجدول `hwp_notify_work` بـ 10ms delay عشان يُعالج في process context.

```
rdmsrq_safe(MSR_HWP_STATUS, &value)
if !(value & status_mask): return  /* ما فيش تغيير */

raw_spin_lock_irqsave(&hwp_notify_lock)
if cpu in hwp_intr_enable_mask:
    schedule_delayed_work(&cpudata->hwp_notify_work, 10ms)
else:
    wrmsrq_safe(MSR_HWP_STATUS, 0)  /* ack بدون معالجة */
raw_spin_unlock_irqrestore()
```

#### `intel_pstate_notify_work`

```c
static void intel_pstate_notify_work(struct work_struct *work)
```

يُعالج في process context بعد الـ ISR. يُحدّث الـ max_freq وينفّذ `hybrid_update_capacity`، ثم يمسح `MSR_HWP_STATUS` لتفعيل interrupts جديدة.

---

### Group 6: EPP (Energy-Performance Preference)

#### `intel_pstate_get_epp`

```c
static s16 intel_pstate_get_epp(struct cpudata *cpu_data, u64 hwp_req_data)
```

يستخرج EPP من `MSR_HWP_REQUEST` bits [31:24]. لو `hwp_req_data == 0`، بيقرأ الـ MSR أولاً. لو الـ CPU بيدعم `X86_FEATURE_EPB` بدل `X86_FEATURE_HWP_EPP`، بيقرأ EPB وبيحوله.

- **Return**: قيمة EPP كـ `s16`، أو `< 0` error

#### `intel_pstate_set_epp`

```c
static int intel_pstate_set_epp(struct cpudata *cpu, u32 epp)
```

يكتب EPP في `MSR_HWP_REQUEST` bits [31:24]. بيستخدم `READ_ONCE(cpu->hwp_req_cached)` عشان يتجنب race مع `intel_pstate_hwp_boost_up/down` اللي ممكن يكتبوا نفس الـ MSR.

```c
value = READ_ONCE(cpu->hwp_req_cached);
value &= ~GENMASK_ULL(31, 24);   /* clear EPP field */
value |= (u64)epp << 24;          /* set new EPP */
WRITE_ONCE(cpu->hwp_req_cached, value);
ret = wrmsrq_on_cpu(cpu->cpu, MSR_HWP_REQUEST, value);
if (!ret) cpu->epp_cached = epp;
```

- **Locking**: يُستدعى تحت `intel_pstate_limits_lock`

#### `intel_pstate_update_epp_defaults`

```c
static void intel_pstate_update_epp_defaults(struct cpudata *cpudata)
```

عند أول تفعيل HWP، بيحدد الـ `epp_default` وممكن يُحدّث `epp_values[EPP_INDEX_BALANCE_PERFORMANCE]` حسب الـ CPU generation أو firmware defaults. مهم على AlderLake وما بعده حيث العلاقة بين EPP والـ turbo frequency مختلفة.

---

### Group 7: P-State Selection Algorithm (Active Mode — No HWP)

#### `intel_pstate_sample`

```c
static inline bool intel_pstate_sample(struct cpudata *cpu, u64 time)
```

بيقرأ APERF و MPERF و TSC بأقل فترة ممكنة (مع `local_irq_save` لضمان التتابع). بيحسب الـ delta من القراءة السابقة.

```
local_irq_save()
aperf = rdmsr(MSR_IA32_APERF)
mperf = rdmsr(MSR_IA32_MPERF)
tsc   = rdtsc()
if prev_mperf == mperf: return false  /* CPU في idle */
local_irq_restore()

sample.aperf = aperf - prev_aperf  /* delta */
sample.mperf = mperf - prev_mperf
sample.tsc   = tsc   - prev_tsc

intel_pstate_calc_avg_perf(cpu)    /* APERF/MPERF ratio */
return true
```

- **Return**: `false` إذا CPU كان idle (MPERF ما تغيرش)
- **Context**: scheduler callback — على نفس CPU دايماً

#### `get_target_pstate`

```c
static inline int32_t get_target_pstate(struct cpudata *cpu)
```

قلب الـ `powersave` algorithm. بيحسب الـ P-state المناسب بناءً على:

```
busy_frac = mperf_delta << aperf_mperf_shift / tsc_delta
            /* MPERF/TSC = fraction of max frequency actually used */

busy_frac = max(busy_frac, iowait_boost)  /* lو في IO boost */
sample.busy_scaled = busy_frac * 100       /* للـ tracing */

/* الـ target = 125% من max_pstate * busy_frac */
target = turbo_pstate (or max if no_turbo)
target += target >> 2            /* +25% headroom */
target = mul_fp(target, busy_frac)
target = max(target, min_pstate)

/* Correction: لو avg_pstate > target، خد نص الفرق */
avg_pstate = get_avg_pstate(cpu)
if avg_pstate > target:
    target += (avg_pstate - target) >> 1
```

الـ +25% headroom موجود عشان الـ `MPERF/TSC` بيمثل average frequency وفيه latency — بنُبالغ قليلاً في الـ request عشان نُغطي الـ peaks.

#### `intel_pstate_adjust_pstate`

```c
static void intel_pstate_adjust_pstate(struct cpudata *cpu)
```

ينفّذ خطوة كاملة من الـ powersave algorithm: يحسب الـ target، يُجهزه (clamp)، يكتبه للـ hardware، ويُطلق trace events.

#### `intel_pstate_update_util`

```c
static void intel_pstate_update_util(struct update_util_data *data,
                                     u64 time, unsigned int flags)
```

الـ callback المُسجّل مع الـ scheduler عبر `cpufreq_add_update_util_hook`. بيتشغل في scheduler context عند كل utilization update.

```
if smp_processor_id() != cpu->cpu: return  /* remote call — ignored */

/* iowait boost logic */
if IOWAIT flag:
    if delta > TICK_NSEC: iowait_boost = ONE_EIGHTH_FP  /* reset */
    else: iowait_boost <<= 1  /* double حتى 1.0 */
else:
    if delta > TICK_NSEC: iowait_boost = 0  /* clear */
    else: iowait_boost >>= 1  /* decay */

/* لو ما مضاش 10ms من آخر sample، ارجع */
if delta_ns < INTEL_PSTATE_SAMPLING_INTERVAL: return

/* sample + adjust */
if intel_pstate_sample(): intel_pstate_adjust_pstate()
```

- **Context**: scheduler context — لا blocking، لا sleeping
- **Locking**: لا يحتاج mutex لأنه دايماً على نفس CPU

---

### Group 8: HWP Boost (iowait Dynamic Boost)

#### `intel_pstate_hwp_boost_up`

```c
static inline void intel_pstate_hwp_boost_up(struct cpudata *cpu)
```

عند اكتشاف IO-wait activity، بيرفع الـ `HWP_MIN` تدريجياً عبر 3 مستويات:

```
Level 0 (start): min_limit (القيمة الأصلية)
Level 1:         (P1_guaranteed + min) / 2  — نص الطريق
Level 2:         P1_guaranteed              — الـ guaranteed performance
Level 3:         P0_turbo_max              — الـ max turbo
```

كل invocation بيرفع مستوى واحد. الفكرة إن الـ boost يتصاعد تدريجياً مش فجأة.

```c
hwp_req = (hwp_req & ~GENMASK_ULL(7, 0)) | cpu->hwp_boost_min;
wrmsrq(MSR_HWP_REQUEST, hwp_req);  /* على نفس CPU مباشرة */
```

#### `intel_pstate_hwp_boost_down`

```c
static inline void intel_pstate_hwp_boost_down(struct cpudata *cpu)
```

بيفحص لو مضى `hwp_boost_hold_time_ns` (3ms) بدون IO، وبعدين بيرجع الـ HWP request لقيمته الـ cached.

- **السبب من 3ms**: قصير بما يكفي عشان ما يأثرش على workloads زي specpower، وطويل بما يكفي عشان يغطي IO latency

---

### Group 9: Passive Mode (intel_cpufreq)

#### `intel_cpufreq_target`

```c
static int intel_cpufreq_target(struct cpufreq_policy *policy,
                                unsigned int target_freq,
                                unsigned int relation)
```

الـ `target` callback للـ passive mode. بيتشغل من generic governor في process context.

```
cpufreq_freq_transition_begin()
target_pstate = intel_pstate_freq_to_hwp_rel(cpu, target_freq, relation)
target_pstate = intel_cpufreq_update_pstate(policy, target_pstate, false)
cpufreq_freq_transition_end()
```

#### `intel_cpufreq_fast_switch`

```c
static unsigned int intel_cpufreq_fast_switch(struct cpufreq_policy *policy,
                                              unsigned int target_freq)
```

الـ `fast_switch` callback — يُستدعى من `schedutil` في scheduler context. لا يوجد `cpufreq_freq_transition_begin/end` هنا لأن الـ switching أسرع من إشعار الـ notifiers.

- **Return**: الـ frequency الفعلية بعد التقريب للـ P-state الأقرب

#### `intel_cpufreq_adjust_perf`

```c
static void intel_cpufreq_adjust_perf(unsigned int cpunum,
                                      unsigned long min_perf,
                                      unsigned long target_perf,
                                      unsigned long capacity)
```

الـ `adjust_perf` callback للـ EAS. يُستدعى من الـ scheduler مباشرةً. يحسب min/max/target P-states من نسب utilization وbيكتبهم في `MSR_HWP_REQUEST` في fast_switch mode.

```
cap_pstate = (no_turbo ? guaranteed : highest) HWP perf

target_pstate = cap * target_perf / capacity
min_pstate    = cap * min_perf / capacity

/* clamp بحدود الـ policy */
min_pstate = clamp(min_pstate, min_pstate_hw, max_perf_ratio)
max_pstate = min(cap_pstate, max_perf_ratio)

intel_cpufreq_hwp_update(cpu, min_pstate, max_pstate, target_pstate, true)
```

#### `intel_cpufreq_hwp_update`

```c
static void intel_cpufreq_hwp_update(struct cpudata *cpu, u32 min, u32 max,
                                     u32 desired, bool fast_switch)
```

يكتب الـ HWP request register. لو `fast_switch=true` يستخدم `wrmsrq` (على نفس CPU)، لو `false` يستخدم `wrmsrq_on_cpu` (IPI للـ target CPU).

```c
value &= ~HWP_MIN_PERF(~0L);    value |= HWP_MIN_PERF(min);
value &= ~HWP_MAX_PERF(~0L);    value |= HWP_MAX_PERF(max);
value &= ~HWP_DESIRED_PERF(~0L); value |= HWP_DESIRED_PERF(desired);
if (value == prev) return;  /* no change — avoid unnecessary MSR write */
WRITE_ONCE(cpu->hwp_req_cached, value);
```

---

### Group 10: Hybrid Processor & Capacity-Aware Scheduling

#### `intel_pstate_hybrid_hwp_adjust`

```c
static void intel_pstate_hybrid_hwp_adjust(struct cpudata *cpu)
```

على hybrid processors (مثلاً AlderLake P-cores + E-cores)، الـ HWP performance levels ممكن تختلف في scaling عن الـ PERF_CTL ratios. الـ function ده بيحل هذا التناقض.

```
if scaling == perf_ctl_scaling: return  /* لا فرق — مش hybrid */

hwp_is_hybrid = true

/* احسب frequencies بالـ HWP scaling، مقرّبة للـ PERF_CTL grid */
turbo_freq = rounddown(turbo_pstate * scaling, perf_ctl_scaling)
max_freq   = rounddown(max_pstate   * scaling, perf_ctl_scaling)

/* حوّل physical max وmin لـ HWP units */
cpu->pstate.max_pstate_physical = intel_pstate_freq_to_hwp(cpu, phys_freq)
cpu->pstate.min_pstate          = intel_pstate_freq_to_hwp(cpu, min_freq)
```

#### `hybrid_update_cpu_capacity_scaling`

```c
static void hybrid_update_cpu_capacity_scaling(void)
```

يُحدّد `hybrid_max_perf_cpu` (أعلى CPU performance) ويضبط `arch_scale_cpu_capacity` لكل CPU كنسبة من الـ max.

```
for_each_online_cpu:
    if !hybrid_max_perf_cpu: hybrid_get_capacity_perf(cpu)  /* init */
    if cpu->capacity_perf > max_cap_perf:
        max_perf_cpu = cpu

hybrid_max_perf_cpu = max_perf_cpu
hybrid_set_capacity_of_cpus()   /* يُحدّث arch_scale للكل */
```

#### `hybrid_set_cpu_capacity`

```c
static void hybrid_set_cpu_capacity(struct cpudata *cpu)
```

يستدعي `arch_set_cpu_capacity(cpu, capacity_perf, max_capacity_perf, ...)` عشان يُخبر الـ scheduler بـ capacity الـ CPU. الـ scheduler بيستخدم هذه القيمة في CAS وEAS.

#### `hybrid_get_capacity_perf`

```c
static void hybrid_get_capacity_perf(struct cpudata *cpu)
```

لو `no_turbo`: capacity = max_pstate_physical (HFM).
غير كده: capacity = highest HWP perf من `hwp_cap_cached` (الـ turbo max).

#### `hybrid_register_perf_domain`

```c
static bool hybrid_register_perf_domain(unsigned int cpu)
```

(مع `CONFIG_ENERGY_MODEL` فقط) يُسجّل Energy Model perf domain لكل CPU مع 4 states (40%, 60%, 80%, 100% capacity). الـ EM مصطنع (artificial) — لا power numbers حقيقية، بس يكفي لـ EAS.

---

### Group 11: Sysfs Interface

#### `intel_pstate_sysfs_expose_params`

```c
static void __init intel_pstate_sysfs_expose_params(void)
```

يُنشئ `/sys/devices/system/cpu/intel_pstate/` kobject ويُضيف الـ attributes. بعض الـ attributes مشروطة:
- `turbo_pct` و`num_pstates`: مش موجودين على hybrid CPUs
- `max_perf_pct` و`min_perf_pct`: مش موجودين لو `per_cpu_perf_limits`
- `energy_efficiency`: فقط على KabyLake/CoffeeLake desktop

#### `store_no_turbo`

```c
static ssize_t store_no_turbo(struct kobject *a, struct kobj_attribute *b,
                              const char *buf, size_t count)
```

يُضبط `global.no_turbo`. لو BIOS أعطل turbo، بيرفض تفعيله. بعد التغيير، بيستدعي `intel_pstate_update_limits_for_all()` و`arch_set_max_freq_ratio()` عشان يُعلم الـ kernel بحدود الـ frequency الجديدة.

#### `store_status` → `intel_pstate_update_status`

```c
static int intel_pstate_update_status(const char *buf, size_t size)
```

يُتيح التحويل بين الـ modes في runtime:

```
"off":     cpufreq_unregister_driver() + cleanup  (مش مسموح مع HWP)
"active":  unregister لو passive + register intel_pstate
"passive": unregister لو active + hide hwp_boost + register intel_cpufreq
```

**تحذير**: التحويل بيعمل unregister+register من جديد — كل الـ settings بترجع لـ defaults.

#### `store_energy_performance_preference`

```c
static ssize_t store_energy_performance_preference(
    struct cpufreq_policy *policy, const char *buf, size_t count)
```

يقبل strings (`performance`, `balance_performance`, إلخ) أو raw integer 0-255. في active mode: يُحدّث EPP مباشرة. في passive mode: لازم يوقف الـ governor أولاً (`cpufreq_stop_governor`) لأن الـ governor ممكن يكون شغال على نفس الـ MSR.

---

### Group 12: Driver Registration & Lifecycle

#### `intel_pstate_register_driver`

```c
static int intel_pstate_register_driver(struct cpufreq_driver *driver)
```

بيُسجّل الـ driver (إما `intel_pstate` أو `intel_cpufreq`) مع الـ CPUFreq core.

```
if driver == &intel_pstate:
    intel_pstate_sysfs_expose_hwp_dynamic_boost()

/* Reset global state */
memset(&global, 0, sizeof(global))
global.max_perf_pct = 100
global.turbo_disabled = turbo_is_disabled()
global.no_turbo = global.turbo_disabled
arch_set_max_freq_ratio(global.turbo_disabled)

hybrid_clear_max_perf_cpu()  /* reset hybrid state */

intel_pstate_driver = driver
cpufreq_register_driver(driver)

global.min_perf_pct = min_perf_pct_min()  /* بعد الـ register لأنه بيحتاج CPU data */
hybrid_init_cpu_capacity_scaling(refresh)
```

#### `intel_pstate_set_policy` — Active Mode

```c
static int intel_pstate_set_policy(struct cpufreq_policy *policy)
```

الـ `setpolicy` callback للـ active mode. بيُحدّث الـ performance limits وبيختار الـ algorithm:

```
cpu->policy = policy->policy

intel_pstate_update_perf_limits(cpu, policy->min, policy->max)

if policy == PERFORMANCE:
    intel_pstate_clear_update_util_hook()  /* لا scheduler callbacks */
    intel_pstate_set_pstate(max_perf_ratio)  /* اكتب max مباشرة */
else:
    intel_pstate_set_update_util_hook()  /* تفعيل scheduler callbacks */

if hwp_active:
    intel_pstate_hwp_set(cpu)  /* كتابة MSR_HWP_REQUEST */
```

#### `intel_pstate_update_perf_limits`

```c
static void intel_pstate_update_perf_limits(struct cpudata *cpu,
                                            unsigned int policy_min,
                                            unsigned int policy_max)
```

يحسب `min_perf_ratio` و`max_perf_ratio` بعد تطبيق كل الحدود:

```
/* تحويل freq → P-state ratio */
max_policy_perf = policy_max / perf_ctl_scaling
min_policy_perf = policy_min / perf_ctl_scaling

if per_cpu_limits:
    /* Policy limits مباشرة */
    cpu->min_perf_ratio = min_policy_perf
    cpu->max_perf_ratio = max_policy_perf
else:
    /* دمج مع الحدود العالمية */
    global_max = turbo_max * max_perf_pct / 100
    global_min = turbo_max * min_perf_pct / 100

    cpu->min_perf_ratio = max(min_policy_perf, global_min)
    cpu->max_perf_ratio = min(max_policy_perf, global_max)
    /* ضمان min <= max */
```

#### `intel_pstate_set_update_util_hook` / `intel_pstate_clear_update_util_hook`

```c
static void intel_pstate_set_update_util_hook(unsigned int cpu_num)
static void intel_pstate_clear_update_util_hook(unsigned int cpu)
```

بيُسجّل/يُلغي تسجيل utilization callback مع الـ scheduler.

- **Set**: بيستدعي `cpufreq_add_update_util_hook` مع الـ callback المناسب (`intel_pstate_update_util_hwp` أو `intel_pstate_update_util`)
- **Clear**: بيستدعي `cpufreq_remove_update_util_hook` ثم `synchronize_rcu()` لضمان إن الـ callback مش شغّال على أي CPU

#### `intel_pstate_suspend` / `intel_pstate_resume`

```c
static int intel_pstate_suspend(struct cpufreq_policy *policy)
static int intel_pstate_resume(struct cpufreq_policy *policy)
```

- **Suspend**: بيعطّل HWP interrupt ويضبط `cpu->suspended = true`
- **Resume**: بيُعيد ضبط `MSR_IA32_POWER_CTL` حسب `power_ctl_ee_state`، وبيُعيد تفعيل HWP إذا لزم

---

### Group 13: ACPI Integration

#### `intel_pstate_init_acpi_perf_limits`

```c
static void intel_pstate_init_acpi_perf_limits(struct cpufreq_policy *policy)
```

لو `hwp_active`: بيستدعي `intel_pstate_set_itmt_prio` فقط (لـ ITMT priority).

لو بدون HWP وفيه `_PPC` support: بيسجل `acpi_processor_register_performance` ويتحقق إن الـ `_PSS` entries تستخدم `ACPI_ADR_SPACE_FIXED_HARDWARE` (يعني PERF_CTL MSR). لو التحقق نجح، بيضبط `cpu->valid_pss_table = true`.

#### `intel_pstate_platform_pwr_mgmt_exists`

```c
static bool __init intel_pstate_platform_pwr_mgmt_exists(void)
```

يفحص عدة سيناريوهات تمنع الـ driver من العمل:
1. Out-of-Band (OOB) management: فحص `MSR_MISC_PWR_MGMT` bits 8 و18
2. `plat_info` table: HP ProLiant وOracle servers ممكن يكون فيها ACPI-based management
3. `_PSS` + `PCCH` combination على HP servers

---

### Group 14: Helper Utilities

#### `intel_pstate_freq_to_hwp_rel`

```c
static int intel_pstate_freq_to_hwp_rel(struct cpudata *cpu, int freq,
                                        unsigned int relation)
```

يحوّل frequency (KHz) لـ HWP performance level. بيعامل `turbo_freq` و`max_freq` كـ exact matches. الـ `relation` بيحدد التقريب: `CPUFREQ_RELATION_H` (لأسفل)، `CPUFREQ_RELATION_L` (لأعلى)، `CPUFREQ_RELATION_C` (أقرب).

#### `turbo_is_disabled`

```c
static bool turbo_is_disabled(void)
```

يقرأ `MSR_IA32_MISC_ENABLE` ويفحص `MSR_IA32_MISC_ENABLE_TURBO_DISABLE` bit. يُستخدم في `store_no_turbo` وفي `intel_pstate_register_driver`.

#### `min_perf_pct_min`

```c
static int min_perf_pct_min(void)
```

يحسب أقل قيمة مسموح بها لـ `min_perf_pct` كنسبة مئوية = `(min_pstate * 100) / turbo_pstate`.

#### `get_avg_frequency`

```c
static inline int32_t get_avg_frequency(struct cpudata *cpu)
```

يحسب متوسط الـ frequency الفعلي = `core_avg_perf * cpu_khz`. يُستخدم في الـ tracing.

#### `intel_cpufreq_trace`

```c
static void intel_cpufreq_trace(struct cpudata *cpu, unsigned int trace_type, int old_pstate)
```

يطلق `pstate_sample` trace event في الـ passive mode. بيستخدم `trace_type` لتمييز normal path (10) من fast switch (90) في الـ `core_busy` field.

---

### تدفق البيانات من Scheduler لـ Hardware

```
Scheduler Tick
     │
     ▼
intel_pstate_update_util()  [active, no HWP]
     │  أو
intel_pstate_update_util_hwp()  [active, HWP]
     │
     ├─── [no HWP] ──────────────────────────────┐
     │                                            ▼
     │                              intel_pstate_sample()
     │                              intel_pstate_adjust_pstate()
     │                              intel_pstate_prepare_request()
     │                              intel_pstate_update_pstate()
     │                              wrmsrq(MSR_IA32_PERF_CTL)
     │
     └─── [HWP] ─────────────────────────────────┐
                                                  ▼
                                  intel_pstate_hwp_boost_up/down()
                                  wrmsrq(MSR_HWP_REQUEST)  [HWP_MIN only]
```

```
Governor → intel_cpufreq_target()  [passive mode]
     │
     ▼
intel_pstate_freq_to_hwp_rel()
     │
     ▼
intel_cpufreq_update_pstate()
     │
     ├─── [HWP] → intel_cpufreq_hwp_update() → wrmsrq(MSR_HWP_REQUEST)
     └─── [no HWP] → intel_cpufreq_perf_ctl_update() → wrmsrq(MSR_IA32_PERF_CTL)
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs entries

**الـ `intel_pstate`** بيكتب معلومات الـ Energy Model في debugfs لما يشتغل على hybrid processors بدون SMT.

```bash
# Mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# Energy Model directory (hybrid processors only)
ls /sys/kernel/debug/energy_model/

# اقرأ performance domain لـ CPU معين
cat /sys/kernel/debug/energy_model/cpu0/ps:0/power
cat /sys/kernel/debug/energy_model/cpu0/ps:0/freq
cat /sys/kernel/debug/energy_model/cpu0/ps:0/cost

# ftrace tracing directory
ls /sys/kernel/tracing/
```

| debugfs Entry | المحتوى | إزاي تقراه |
|---|---|---|
| `/sys/kernel/debug/energy_model/` | Performance domains للـ EAS | `ls` ثم `cat` كل ملف |
| `/sys/kernel/tracing/trace` | سجل الـ ftrace الحالي | `cat trace` |
| `/sys/kernel/tracing/available_filter_functions` | كل الفنكشنات اللي ممكن تتتريس | `grep -i pstate` |
| `/sys/kernel/tracing/trace_pipe` | stream حي من الـ trace | `cat trace_pipe` |

---

#### 2. sysfs entries

**الـ global attributes** كلها تحت `/sys/devices/system/cpu/intel_pstate/`:

```bash
# اقرأ كل الـ global attributes دفعة واحدة
for f in /sys/devices/system/cpu/intel_pstate/*; do
    echo "=== $(basename $f) ===" && cat "$f"
done
```

| sysfs Path | نوع الوصول | المعنى |
|---|---|---|
| `.../intel_pstate/status` | RW | active / passive / off |
| `.../intel_pstate/no_turbo` | RW | 1 = disable turbo P-states |
| `.../intel_pstate/max_perf_pct` | RW | أقصى P-state كنسبة % من الـ turbo max |
| `.../intel_pstate/min_perf_pct` | RW | أدنى P-state كنسبة % |
| `.../intel_pstate/num_pstates` | RO | عدد الـ P-states الكلي |
| `.../intel_pstate/turbo_pct` | RO | % من الـ range اللي turbo |
| `.../intel_pstate/hwp_dynamic_boost` | RW | iowait boost (HWP active mode فقط) |
| `.../intel_pstate/energy_efficiency` | RW | Kaby/Coffee Lake desktop فقط |

**الـ per-policy attributes** تحت `/sys/devices/system/cpu/cpufreq/policy<N>/`:

```bash
# شوف state الـ driver
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_driver
# intel_pstate (active) أو intel_cpufreq (passive)

# شوف الـ governor الحالي
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor

# الـ EPP (HWP only)
cat /sys/devices/system/cpu/cpufreq/policy0/energy_performance_preference
cat /sys/devices/system/cpu/cpufreq/policy0/energy_performance_available_preferences

# Base frequency (HWP only) — فوقيها = turbo range
cat /sys/devices/system/cpu/cpufreq/policy0/base_frequency

# الـ current frequency
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_max_freq
cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_min_freq
```

---

#### 3. ftrace

**الطريقة دي بتستخدمها لما تحتاج تعرف:**
- بيتعمل P-state change بأي تكرار؟
- مين اللي بيستدعي `intel_pstate_set_pstate`؟

```bash
cd /sys/kernel/tracing

# اكتشف كل الفنكشنات المتاحة في intel_pstate
cat available_filter_functions | grep -i pstate

# trace فنكشن واحدة بالظبط
echo intel_pstate_set_pstate > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
sleep 5
echo 0 > tracing_on
cat trace | head -30

# trace أكتر من فنكشن
echo -e "intel_pstate_set_pstate\nintel_pstate_hwp_set_policy_params" > set_ftrace_filter

# trace events الـ power subsystem
echo 1 > events/power/pstate_sample/enable
echo 1 > events/power/cpu_frequency/enable
echo 1 > tracing_on
cat trace_pipe   # stream مباشر

# تنظيف
echo nop > current_tracer
echo > set_ftrace_filter
echo 0 > events/power/pstate_sample/enable
echo 0 > events/power/cpu_frequency/enable
```

**تفسير output الـ `pstate_sample` event:**

```
gnome-terminal--4510 [001] ..s. 1177.680733: pstate_sample:
    core_busy=107   ← % utilization (مقاسة من APERF/MPERF) * 100
    scaled=94       ← core_busy بعد التعديل على الـ idle periods
    from=26         ← P-state القديم
    to=26           ← P-state الجديد (لم يتغير)
    mperf=1143818   ← delta MPERF (max perf clocks)
    aperf=1230607   ← delta APERF (actual perf clocks)
    tsc=29838618    ← delta TSC
    freq=2474476    ← الـ frequency المحسوبة بـ kHz
```

---

#### 4. printk و dynamic debug

```bash
# فعّل dynamic debug لكل ملفات intel_pstate
echo 'file intel_pstate.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل لـ module بالكامل
echo 'module intel_pstate +p' > /sys/kernel/debug/dynamic_debug/control

# شوف اللي فعّال حالياً
grep intel_pstate /sys/kernel/debug/dynamic_debug/control

# تابع الـ kernel messages
dmesg -w | grep -i pstate
journalctl -k -f | grep -i pstate

# رفع الـ loglevel مؤقتاً (الكل يطلع)
echo 8 > /proc/sys/kernel/printk
```

---

#### 5. Kernel Config options للـ Debugging

```bash
# شوف الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(CPU_FREQ|INTEL_PSTATE|X86_MSR|DEBUG|TRACING|ENERGY)'
```

| Config Option | الغرض |
|---|---|
| `CONFIG_X86_MSR` | لازم يكون `=y` عشان تقدر تقرأ MSRs يدوياً |
| `CONFIG_CPU_FREQ_DEBUG` | يفعّل extra debug output في CPUFreq core |
| `CONFIG_CPU_FREQ_STAT` | إحصائيات الـ P-state transitions في sysfs |
| `CONFIG_DEBUG_FS` | لازم لـ debugfs و energy_model directory |
| `CONFIG_FTRACE` | لازم لكل تقنيات الـ function tracing |
| `CONFIG_TRACING` | infrastructure الـ trace events |
| `CONFIG_ENERGY_MODEL` | EAS + energy_model directory في debugfs |
| `CONFIG_ACPI_CPPC_LIB` | لازم لـ CPPC/HWP على بعض platforms |
| `CONFIG_INTEL_IDLE` | بيتفاعل مع intel_pstate في idle states |
| `CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE` | يحدد الـ default governor |

---

#### 6. أدوات خاصة بالـ subsystem

```bash
# cpupower — الأداة الرئيسية
cpupower frequency-info          # كل المعلومات
cpupower frequency-info -p       # P-states المتاحة
cpupower monitor                 # monitor مباشر للـ frequencies
cpupower monitor -m Mperf        # APERF/MPERF monitor
cpupower idle-info               # C-states info

# turbostat — الأشمل للـ Intel
turbostat --interval 1           # كل 1 ثانية
turbostat --show CPU,Bzy_MHz,Avg_MHz,Busy%,PkgWatt

# x86_energy_perf_policy (EPB control)
x86_energy_perf_policy --read    # قرا EPB لكل CPUs
x86_energy_perf_policy performance
x86_energy_perf_policy powersave

# rdmsr / wrmsr (من package msr-tools)
rdmsr -p 0 0x199   # قرا IA32_PERF_CTL للـ CPU 0
rdmsr -p 0 0x771   # HWP_REQUEST MSR
rdmsr -p 0 0x772   # HWP_CAPABILITIES MSR
rdmsr -p 0 0x1a0   # IA32_MISC_ENABLE (بيت 38 = IDA/turbo disable)
rdmsr -p 0 0x1FC   # MSR_POWER_CTL

# perf لـ P-state monitoring
perf stat -e power/energy-pkg/ sleep 1
perf stat -a -e cpu-cycles,cpu-clock sleep 5
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة في dmesg | المعنى | الحل |
|---|---|---|
| `intel_pstate: HWP enabled` | HWP اتفعّل بنجاح | ✓ طبيعي |
| `intel_pstate: CPU model not supported` | الـ CPU مش في القائمة المدعومة + مفيش HWP | جرّب `intel_pstate=passive` أو `acpi-cpufreq` |
| `intel_pstate: overclocking detected` | الـ turbo pstate أعلى من المتوقع | تحقق من BIOS overclocking settings |
| `intel_pstate: Disabling inefficient turbo` | thermal أو power limit أوقف الـ turbo | افحص الحرارة، `turbostat` |
| `intel_pstate: freq is out of limits` | الـ frequency المطلوبة خارج الحدود | راجع `scaling_min_freq` / `scaling_max_freq` |
| `pstate: ACPI _PPC requires ... MHz` | الـ BIOS/firmware قيّد الـ P-state | فعّل `intel_pstate=support_acpi_ppc` أو راجع firmware |
| `intel_pstate: sysfs: invalid EPP value` | كتبت قيمة EPP غلط | القيم الصحيحة: 0-255 أو string من `available_preferences` |
| `intel_pstate: turbo is disabled by user` | `no_turbo=1` مفعّل | `echo 0 > .../intel_pstate/no_turbo` |
| `intel_pstate: Not attaching to` | الـ driver رفض CPU معين | الـ CPU مش supported، راجع supported list في المصدر |
| `hwp_notify: HWP Interrupt` | HWP interrupt وصل (thermal event) | افحص الحرارة وحدود الـ power |

---

#### 8. نقاط استراتيجية لـ `dump_stack()` و `WARN_ON()`

في الكود الأصلي لـ `intel_pstate.c`، النقاط دي مهمة للـ debugging:

```c
/* نقطة 1: قبل كتابة MSR_IA32_PERF_CTL */
static void intel_pstate_set_pstate(struct cpudata *cpu, int pstate)
{
    /* WARN لو الـ pstate خارج الحدود */
    WARN_ON(pstate < cpu->pstate.min_pstate ||
            pstate > cpu->pstate.turbo_pstate);
    /* هنا تحط dump_stack() لو محتاج call trace */
}

/* نقطة 2: في intel_pstate_update_util_hwp() */
/* لو الـ EPP لم يتغير كما هو متوقع */
WARN_ON(rdmsrl_safe(MSR_HWP_REQUEST, &value) < 0);

/* نقطة 3: في intel_pstate_init_cpu() */
/* لو cpudata allocation فشل */
WARN_ON(!all_cpu_data[cpu->cpu]);
```

**للـ debugging اليدوي** — حط في kernel patch مؤقت:

```c
/* في intel_pstate_set_pstate() */
pr_debug("CPU%d: pstate %d -> %d (min=%d max=%d turbo=%d)\n",
         cpu->cpu, cpu->pstate.current_pstate, pstate,
         cpu->pstate.min_pstate, cpu->pstate.max_pstate,
         cpu->pstate.turbo_pstate);
```

---

### Hardware Level

#### 1. التحقق إن الـ hardware state يطابق الـ kernel state

الـ `intel_pstate` يشتغل بكتابة MSRs مباشرة. التحقق بيكون بـ قراءة نفس الـ MSRs:

```bash
# الـ MSRs الأساسية
# IA32_PERF_CTL (0x199) — P-state المطلوب (بدون HWP)
rdmsr -p 0 0x199
# الـ bits [14:8] = الـ P-state المطلوب

# IA32_PERF_STATUS (0x198) — P-state الفعلي الحالي
rdmsr -p 0 0x198
# الـ bits [14:8] = الـ P-state الفعلي

# HWP_REQUEST (0x774) — لما HWP مفعّل
rdmsr -p 0 0x774
# bits [7:0]   = min performance
# bits [15:8]  = max performance
# bits [23:16] = desired performance
# bits [31:24] = EPP

# HWP_CAPABILITIES (0x771)
rdmsr -p 0 0x771
# bits [7:0]   = highest_performance
# bits [15:8]  = guaranteed_performance
# bits [23:16] = efficient_performance
# bits [31:24] = lowest_performance

# APERF (0xE8) و MPERF (0xE7) — للتحقق من الـ actual frequency
rdmsr -p 0 0xE8   # APERF
rdmsr -p 0 0xE7   # MPERF
# actual_freq = base_freq * (APERF_delta / MPERF_delta)

# IA32_MISC_ENABLE (0x1A0)
rdmsr -p 0 0x1A0
# bit 38 = IDA (turbo disable) — لو 1 يبقى turbo disabled من الـ BIOS
```

**مقارنة kernel state بالـ hardware:**

```bash
# kernel يقول الـ current freq
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq

# hardware يقول الـ actual freq (أصدق)
turbostat --show CPU,Avg_MHz,Bzy_MHz --interval 1 --num_iterations 3
```

---

#### 2. Register Dump Techniques

```bash
# devmem2 — للـ MMIO registers (مش شايع في intel_pstate)
# لكن ممكن تستخدمه للـ PCIe power management registers
devmem2 0xFEDC0000 w   # مثال لـ MMIO address

# /dev/cpu/N/msr — القراءة المباشرة للـ MSR
# لازم module msr يكون loaded
modprobe msr
dd if=/dev/cpu/0/msr bs=8 count=1 skip=$((0x198)) 2>/dev/null | xxd

# dump كل المهم دفعة واحدة
for cpu in $(seq 0 $(nproc --all | xargs -I{} expr {} - 1)); do
    echo "=== CPU $cpu ==="
    rdmsr -p $cpu 0x198   # PERF_STATUS
    rdmsr -p $cpu 0x199   # PERF_CTL
    rdmsr -p $cpu 0x774   # HWP_REQUEST (لو مدعوم)
    rdmsr -p $cpu 0x771   # HWP_CAPABILITIES
done 2>/dev/null

# script شامل للـ P-state state
cat << 'EOF' > /tmp/pstate_dump.sh
#!/bin/bash
echo "=== intel_pstate Global State ==="
cat /sys/devices/system/cpu/intel_pstate/status
cat /sys/devices/system/cpu/intel_pstate/no_turbo
cat /sys/devices/system/cpu/intel_pstate/max_perf_pct
cat /sys/devices/system/cpu/intel_pstate/min_perf_pct

echo "=== Per-CPU MSR Dump ==="
for cpu in $(seq 0 $(($(nproc) - 1))); do
    perf_status=$(rdmsr -p $cpu 0x198 2>/dev/null)
    echo "CPU$cpu PERF_STATUS=0x$perf_status current_pstate=$(( 16#${perf_status:-0} >> 8 & 0x7F ))"
done
EOF
chmod +x /tmp/pstate_dump.sh
bash /tmp/pstate_dump.sh
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

للـ `intel_pstate` hardware debugging:

| ما تقيسه | الـ Signal | الهدف |
|---|---|---|
| VRM switching frequency | الـ power rail للـ CPU | تأكيد الـ DVFS بيحصل |
| CPU core voltage | VCC_CORE | تحقق إن الـ voltage بيتغير مع الـ P-state |
| PROCHOT# pin | GPIO/PMBus | thermal throttling signal |
| PLx power limits | PMBus/SVID | هل الـ power limit بيقيّد الـ turbo |

```
CPU Package
    ┌─────────────────────────┐
    │  Core 0  Core 1  ...    │
    │    │       │            │
    │    ▼       ▼            │
    │  P-state via MSR write  │
    │    │                    │
    │    ▼                    │
    │  VRM ←── SVID/PMBus ───┼── Oscilloscope هنا
    │  (Voltage Regulator)    │   لتتبع voltage steps
    └─────────────────────────┘
```

**نصايح عملية:**
- لو عندك PROCHOT# يشتغل كتير → CPU بيتحرق، مش مشكلة في intel_pstate
- قس الـ VCC_CORE بالأسيلوسكوب لما تغيّر الـ governor → لازم تشوف voltage steps
- PMBus interface على boards الـ server بتديك قراءة مباشرة للـ power

---

#### 4. Hardware Issues الشائعة ← kernel log patterns

| المشكلة الـ Hardware | pattern في dmesg / turbostat | التشخيص |
|---|---|---|
| Thermal throttling | `PROCHOT`, `Pkg%pc2` عالي، `Bzy_MHz` أقل من `Avg_MHz` | `turbostat \| grep -E 'PROCHOT\|PkgTmp'` |
| Power limit reached | `PkgWatt` = max طول الوقت، turbo P-state مش بيتوصله | `turbostat --show PkgWatt,CPU,Bzy_MHz` |
| BIOS turbo disabled | `IA32_MISC_ENABLE bit 38 = 1`، `turbo_disabled=1` في kernel | `rdmsr -p 0 0x1A0` وشوف bit 38 |
| ACPI firmware conflict | `_PPC limit`، frequency caps غير متوقعة | `acpidump \| acpixtract -a && iasl -d DSDT.dat` |
| Memory/IO bottleneck | `core_busy` منخفض لكن frequency كمان منخفضة | مش مشكلة في intel_pstate نفسه |
| VRM overtemperature | Sudden freq drop، voltage collapse على الـ scope | افحص VRM heatsink |

---

#### 5. Device Tree Debugging

الـ `intel_pstate` بيشتغل على **x86** فمفيش Device Tree بالمعنى التقليدي، لكن في ما يعادله:

```bash
# تحقق من ACPI tables (الـ "DT" على x86)
# _PSS object — بيعرّف الـ P-states المتاحة
acpidump -n DSDT > /tmp/DSDT.dat
acpixtract -a /tmp/DSDT.dat
iasl -d DSDT.dat

# دور على _PSS في الـ ACPI source
grep -A 20 "_PSS" DSDT.dsl | head -60

# تحقق من ACPI _PPC (performance present capabilities)
# الـ firmware ممكن يقيّد الـ P-states عبر _PPC
cat /sys/bus/acpi/devices/*/power_resources 2>/dev/null

# CPUID — تحقق إن الـ HWP feature موجودة فعلاً في الـ hardware
cpuid | grep -i "hwp\|hdc\|epb\|epp"
# أو
cpuid -1 -l 6 -s 0  # Leaf 6 = Thermal and Power Management

# تأكيد إن HWP اتفعّل في الـ kernel
dmesg | grep -i hwp
cat /proc/cpuinfo | grep -i hwp  # لو مش ظاهر جرب:
rdmsr -p 0 0x770  # IA32_PM_ENABLE — bit 0 = HWP_ENABLE

# FADT profile يأثر على intel_pstate behavior
acpidump -n FACP > /tmp/FACP.dat
acpixtract -a /tmp/FACP.dat
iasl -d FACP.dat
grep -i "preferred" FACP.dsl  # "Enterprise Server" = _PPC limits applied by default
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

**#1 — تشخيص سريع شامل:**

```bash
#!/bin/bash
# quick_pstate_diag.sh — تشغيله كـ root

echo "====== intel_pstate Quick Diagnostic ======"
echo ""

echo "[1] Driver Status:"
cat /sys/devices/system/cpu/intel_pstate/status 2>/dev/null || echo "intel_pstate not active"
echo "Scaling driver: $(cat /sys/devices/system/cpu/cpufreq/policy0/scaling_driver)"
echo "Governor: $(cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor)"

echo ""
echo "[2] Global Limits:"
echo "  no_turbo=$(cat /sys/devices/system/cpu/intel_pstate/no_turbo 2>/dev/null)"
echo "  max_perf_pct=$(cat /sys/devices/system/cpu/intel_pstate/max_perf_pct 2>/dev/null)"
echo "  min_perf_pct=$(cat /sys/devices/system/cpu/intel_pstate/min_perf_pct 2>/dev/null)"
echo "  num_pstates=$(cat /sys/devices/system/cpu/intel_pstate/num_pstates 2>/dev/null)"
echo "  turbo_pct=$(cat /sys/devices/system/cpu/intel_pstate/turbo_pct 2>/dev/null)"

echo ""
echo "[3] HWP Status (CPU0):"
hwp_en=$(rdmsr -p 0 0x770 2>/dev/null)
echo "  HWP Enable MSR (0x770) = 0x${hwp_en:-N/A}"
hwp_cap=$(rdmsr -p 0 0x771 2>/dev/null)
echo "  HWP Capabilities (0x771) = 0x${hwp_cap:-N/A}"
hwp_req=$(rdmsr -p 0 0x774 2>/dev/null)
echo "  HWP Request (0x774) = 0x${hwp_req:-N/A}"

echo ""
echo "[4] Turbo Enable (MISC_ENABLE):"
misc=$(rdmsr -p 0 0x1A0 2>/dev/null)
echo "  MSR_IA32_MISC_ENABLE = 0x${misc:-N/A}"
# bit 38 turbo disable
turbo_dis=$(( (16#${misc:-0} >> 38) & 1 ))
echo "  Turbo Disabled bit: $turbo_dis (1=disabled in hardware)"

echo ""
echo "[5] EPP (policy0):"
cat /sys/devices/system/cpu/cpufreq/policy0/energy_performance_preference 2>/dev/null

echo ""
echo "[6] Current Frequencies:"
cat /sys/devices/system/cpu/cpufreq/policy*/scaling_cur_freq 2>/dev/null | \
    awk '{printf "  policy%d: %d MHz\n", NR-1, $1/1000}'
```

---

**#2 — تفعيل trace events ومراقبة مباشرة:**

```bash
# فعّل الاثنين
cd /sys/kernel/tracing
echo 1 > events/power/pstate_sample/enable
echo 1 > events/power/cpu_frequency/enable
echo 1 > tracing_on
echo "Tracing for 10 seconds..."
sleep 10
echo 0 > tracing_on

# عرض النتائج مع تفسير
cat trace | grep -E "pstate_sample|cpu_frequency" | head -20
```

**مثال output وتفسيره:**

```
# pstate_sample — Active mode only
gnome-terminal--4510 [001] ..s. 1177.680733: pstate_sample:
    core_busy=107   ← الـ CPU مشغول 107% (scaled utilization)
    scaled=94       ← بعد حساب الـ idle
    from=26 to=26   ← الـ P-state ما اتغيرش (26 = 2600 MHz مثلاً)
    mperf=1143818   ← max perf clocks في الفترة دي
    aperf=1230607   ← actual perf clocks (APERF > MPERF = turbo mode!)
    tsc=29838618    ← TSC ticks
    freq=2474476    ← الـ frequency المحسوبة بـ kHz

# cpu_frequency — active + passive
cat-5235 [002] ..s. 1177.681723: cpu_frequency:
    state=2900000   ← الـ frequency المطلوبة بـ kHz
    cpu_id=2        ← رقم الـ CPU
```

---

**#3 — ftrace function tracing:**

```bash
cd /sys/kernel/tracing

# trace كل الفنكشنات المتعلقة بـ pstate
cat available_filter_functions | grep -i pstate | \
    tee /tmp/pstate_funcs.txt

# فعّل الأهم منهم
echo "intel_pstate_set_pstate
intel_pstate_update_util
intel_pstate_hwp_set_policy_params
intel_pstate_set_energy_pref_index" > set_ftrace_filter

echo function > current_tracer
echo 1 > tracing_on
sleep 3
echo 0 > tracing_on
cat trace | grep -v "^#" | head -30
```

---

**#4 — turbostat لمراقبة شاملة:**

```bash
# كل 1 ثانية، أهم الأعمدة فقط
turbostat \
    --show CPU,Bzy_MHz,Avg_MHz,Busy%,PkgWatt,CoreTmp,POLL,C1,C6 \
    --interval 1

# مثال output
# CPU   Bzy_MHz  Avg_MHz  Busy%  PkgWatt  CoreTmp
#   0     3800     1200    12.5     8.2      45
#   1     3800      950     8.1     8.2      43
# لو Bzy_MHz < max turbo → يبقى في throttling أو power limit
```

---

**#5 — مراقبة EPP وتغييره:**

```bash
# اقرأ EPP لكل الـ CPUs
for policy in /sys/devices/system/cpu/cpufreq/policy*/; do
    echo -n "$(basename $policy): "
    cat "${policy}energy_performance_preference"
done

# غيّر EPP لـ performance
for policy in /sys/devices/system/cpu/cpufreq/policy*/; do
    echo "performance" > "${policy}energy_performance_preference"
done

# القيم المتاحة
cat /sys/devices/system/cpu/cpufreq/policy0/energy_performance_available_preferences
# default performance balance_performance balance_power power
```

---

**#6 — اكتشاف hybrid processor:**

```bash
# شوف لو في core types مختلفة (P-cores vs E-cores)
cat /sys/devices/system/cpu/cpu*/cpufreq/cpuinfo_max_freq | sort -u
# لو في قيمتين مختلفتين → hybrid processor

# شوف capacity لكل CPU (CAS)
for cpu in /sys/devices/system/cpu/cpu[0-9]*/cpu_capacity; do
    echo "$(dirname $cpu | xargs basename): $(cat $cpu)"
done

# شوف لو SMT مفعّل
cat /sys/devices/system/cpu/smt/active
# 1 = SMT active, 0 = SMT off

# لو hybrid بدون SMT → CAS و EAS مفعّلين تلقائياً
# تحقق من energy_model
ls /sys/kernel/debug/energy_model/ 2>/dev/null && \
    echo "EAS Energy Model registered" || \
    echo "No Energy Model"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: سيرفر Intel Xeon بيعمل بـ `acpi-cpufreq` بدل `intel_pstate` وبيوقف الـ turbo

#### العنوان
Industrial Gateway على Xeon E-2300 — الـ turbo مش شغال والـ throughput أقل من المتوقع

#### السياق
عندنا industrial gateway بيشتغل على Intel Xeon E-2388G (Ice Lake) في خط إنتاج. الـ gateway بيعمل deep packet inspection على traffic عالي بيتطلع أحيانًا لـ 10 Gbps. الـ team لاحظت إن الـ CPU مش بيوصل للـ turbo frequencies حتى لو الـ load 100%.

#### المشكلة
الـ `scaling_driver` بيظهر `acpi-cpufreq` مش `intel_pstate`، وبالتالي الـ turbo range بيتعامل معاه كـ 1 MHz فوق الـ base frequency بدل ما يكون الـ full turbo range متاح. الـ governor بيحسب إن 100% load = base + 1 MHz، مش الـ turbo الفعلي.

```bash
# اللي بيظهره الـ sysfs
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
acpi-cpufreq

$ cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq
3200001   # base 3.2 GHz + 1 KHz فقط — الـ turbo الفعلي 5.1 GHz

$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
3200000   # مش بيتعدى الـ base أبدًا
```

#### التحليل
الـ documentation بتشرح المشكلة بالظبط:

> *"the turbo range corresponds to a small fraction of the frequency band it can use (1 MHz vs 1 GHz or more). In consequence, it will only go to the turbo range for the highest loads"*

الـ `acpi-cpufreq` بياخد من `_PSS` objects اللي بتمثّل الـ turbo range كـ item واحد بـ frequency = highest non-turbo + 1 MHz. يعني الـ governor زي `schedutil` بيشوف الـ turbo كـ 3200001 Hz بدل 5100000 Hz، فبيحسب الـ load mapping غلط ومش بيروح للـ turbo إلا لما الـ load يبقى فعلًا فوق 99.9%.

```
ACPI _PSS view:               intel_pstate view:
┌──────────────┐              ┌──────────────┐
│ 3200001 Hz   │ ← turbo all  │ 5100000 Hz   │ ← max turbo
│ 3200000 Hz   │              │ 4800000 Hz   │
│ 2800000 Hz   │              │ ...          │
│ 2400000 Hz   │              │ 3200000 Hz   │ ← base
│ ...          │              │ ...          │
└──────────────┘              └──────────────┘
```

#### الحل
نفرض `intel_pstate` بدل `acpi-cpufreq` عن طريق الـ kernel command line:

```bash
# في /etc/default/grub
GRUB_CMDLINE_LINUX="intel_pstate=active"

# أو لو الـ BIOS بيفضّل acpi-cpufreq
GRUB_CMDLINE_LINUX="intel_pstate=force"

$ sudo update-grub && sudo reboot
```

بعد الـ reboot:

```bash
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
intel_pstate

$ cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq
5100000   # الـ real turbo max

$ cat /sys/devices/system/cpu/intel_pstate/turbo_pct
59        # 59% من الـ P-states هي turbo states
```

#### الدرس المستفاد
لما تلاقي الـ `scaling_driver` بيقول `acpi-cpufreq` على Intel CPU حديث، الـ turbo range مش هيكون متاح بشكل صح لأي governor إلا `performance` و`powersave`. استخدم `intel_pstate=force` لو محتاج الـ full turbo range وانت عارف إن الـ platform مش بيعتمد على `_PSS` للـ thermal management.

---

### السيناريو الثاني: Android TV Box على Core i3-1115G4 — الـ battery بتخلص بسرعة

#### العنوان
Android TV Box مبني على Intel Core i3-1115G4 (Tiger Lake) — الـ thermal throttling مستمر وعمر الـ battery قصير

#### السياق
شركة بتعمل Android TV box بيشتغل على Tiger Lake. الجهاز بيتشحن عن طريق USB-C بـ 45W فقط. المستخدمين بيشتكوا إن الجهاز بيسخن والـ battery بتخلص في 2-3 ساعات بدل 6 ساعات.

#### المشكلة
الـ `energy_performance_preference` على كل CPU ضابط على `performance` (EPP=0) لأن الـ kernel compile بـ `CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE=y`. ده بيخلي الـ HWP يضغط دايمًا على الـ turbo حتى في وقت الـ video playback البسيط.

```bash
$ cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
performance

$ cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_available_preferences
default performance balance_performance balance_power power

$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
performance
```

#### التحليل
الـ documentation بتقول:

> *"In this configuration intel_pstate will write 0 to the processor's Energy-Performance Preference (EPP) knob... any attempts to change the EPP/EPB to a value different from 0 ("performance") via sysfs in this configuration will be rejected."*

يعني طالما الـ `scaling_governor` = `performance`، أي محاولة لتغيير الـ EPP بتتعطل:

```bash
$ echo balance_power > /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
bash: echo: write error: Invalid argument
# رفض لأن الـ HWP performance mode بيلغي أي EPP غير 0
```

كمان الـ documentation بتوضح:

> *"the range of P-states available to the processor's internal P-state selection logic is always restricted to the upper boundary"*

يعني الـ HWP دايمًا بيشتغل من min_perf لـ max_perf بدون قيود، وده بيرفع الـ power consumption.

#### الحل
نغيّر الـ governor لـ `powersave` الأول، وبعدين نضبط الـ EPP:

```bash
# خطوة 1: غيّر الـ governor لـ powersave على كل الـ CPUs
$ for cpu in /sys/devices/system/cpu/cpu*/cpufreq/; do
    echo powersave > "${cpu}scaling_governor"
  done

# خطوة 2: دلوقتي يمكن تغيير الـ EPP
$ for cpu in /sys/devices/system/cpu/cpu*/cpufreq/; do
    echo balance_power > "${cpu}energy_performance_preference"
  done

# خطوة 3: تحقق
$ cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference
balance_power
```

أو لو عايز حل دائم في الـ kernel command line:

```bash
GRUB_CMDLINE_LINUX="intel_pstate=passive"
# في passive mode، schedutil governor مع EAS بيتحكم أحسن في الـ power
```

للـ video playback تحديدًا، الـ `hwp_dynamic_boost` ممكن يتعطل:

```bash
$ echo 0 > /sys/devices/system/cpu/intel_pstate/hwp_dynamic_boost
```

#### الدرس المستفاد
في battery-powered devices على Intel HWP، خد بالك إن `CONFIG_CPU_FREQ_DEFAULT_GOV_PERFORMANCE` بيلغي قدرتك على تغيير الـ EPP. لازم تغيّر الـ governor لـ `powersave` الأول عشان الـ EPP writes تشتغل. في الـ product firmware، افضل تـ compile بـ `CONFIG_CPU_FREQ_DEFAULT_GOV_SCHEDUTIL`.

---

### السيناريو الثالث: Automotive ECU على Core i7-1185G7E — الـ real-time latency غير متوقعة

#### العنوان
Automotive ECU بيشتغل على Intel Core i7-1185G7E (Tiger Lake-E) — spikes في الـ latency وقت تشغيل الـ ADAS algorithms

#### السياق
شركة automotive بتبني ADAS (Advanced Driver Assistance System) ECU على Intel Tiger Lake-E. الـ real-time algorithms لازم تـ respond في أقل من 10ms. الـ system بيشتغل على PREEMPT_RT kernel. المشكلة إن في spikes عشوائية في الـ processing time بتوصل لـ 50ms.

#### المشكلة
الـ `intel_pstate` شغال في الـ active mode بدون HWP (`intel_pstate=no_hwp intel_pstate=active`). في الـ active mode بدون HWP، الـ `powersave` algorithm بيتشغل كل 10ms minimum عن طريق scheduler callbacks. ده بيعمل interruptions في الـ real-time tasks.

```bash
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
intel_pstate

$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
powersave   # المشكلة هنا

# الـ ftrace بيكشف الـ problem
$ cd /sys/kernel/tracing/
$ echo intel_pstate_set_pstate > set_ftrace_filter
$ echo function > current_tracer
$ cat trace | grep -c "intel_pstate_set_pstate"
# آلاف من الـ calls في ثانية واحدة!
```

#### التحليل
الـ documentation بتشرح:

> *"This algorithm is run by the driver's utilization update callback for the given CPU when it is invoked by the CPU scheduler, but not more often than every 10 ms."*

في الـ real-time scenario، الـ P-state selection callback بيتشغل من scheduler context كل 10ms، وده بيعمل:
1. **jitter** في الـ execution time للـ RT tasks
2. **cache pollution** لأن الـ MSR writes بياخدوا cycles
3. تغييرات في الـ P-state بيعمل voltage/frequency transitions بياخدوا وقت

الـ ftrace بيكشف إن `intel_pstate_set_pstate` بيتـ call من `intel_pstate_timer_func` كتير جدًا:

```
ADAS-rt-1234  [003] ..s.  5000.000000: intel_pstate_set_pstate <-intel_pstate_timer_func
ADAS-rt-1234  [003] ..s.  5000.010000: intel_pstate_set_pstate <-intel_pstate_timer_func
# كل 10ms، حتى لو الـ P-state مش بيتغير
```

#### الحل
**الحل المثالي:** ضبّط `performance` governor عشان الـ RT CPUs تبقى على max P-state دايمًا:

```bash
# خصص CPUs 2 و 3 للـ RT tasks وخليهم على performance
$ echo performance > /sys/devices/system/cpu/cpu2/cpufreq/scaling_governor
$ echo performance > /sys/devices/system/cpu/cpu3/cpufreq/scaling_governor

# أو عطّل الـ turbo وخلي الـ P-state ثابت على base
$ echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo
$ echo 100 > /sys/devices/system/cpu/intel_pstate/min_perf_pct
$ echo 100 > /sys/devices/system/cpu/intel_pstate/max_perf_pct
```

**الحل الأمثل في الـ kernel cmdline:**

```bash
# HWP بيخلي الـ CPU hardware يتحكم في الـ P-state بدون scheduler interference
GRUB_CMDLINE_LINUX="intel_pstate=active"
# مع إن HWP enabled by default على Tiger Lake-E
```

بعد التغيير، تحقق من الـ latency:

```bash
$ cyclictest -p 99 -t 1 -n -q --duration=60
# النتايج المتوقعة بعد الـ fix:
# Max latency: 150us بدل 50ms
```

#### الدرس المستفاد
في الـ real-time systems على Intel، الـ `performance` governor في active mode بدون HWP هو الخيار الأمثل لأنه بيختار max P-state مرة واحدة وما بيتدخلش تاني. لو HWP متاح، فعّله وخلي الـ hardware يتحكم — الـ HWP transitions أسرع وأقل impact على الـ RT tasks.

---

### السيناريو الرابع: Custom Board Bring-up على Intel NUC بـ Alder Lake — الـ hybrid CPUs بيدوا نتايج غريبة

#### العنوان
Custom industrial board على Intel Core i9-12900 (Alder Lake) — الـ P-cores بتتجمد على min frequency وبعض الـ tasks بطيئة بشكل غير منطقي

#### السياق
فريق بيعمل bring-up لـ custom industrial computing board بيستخدم Intel Core i9-12900 (Alder Lake Hybrid: 8 P-cores + 16 E-cores). البورد بيشتغل على Ubuntu Server 22.04 مع kernel 5.15. بعض الـ worker threads بطيئة جدًا بشكل غير متوقع.

#### المشكلة
الـ system شغال بـ `intel_pstate=no_cas` في الـ cmdline بسبب مشكلة قديمة مع kernel 5.12. الـ Capacity-Aware Scheduling (CAS) معطّل، وبالتالي الـ scheduler مش عارف الفرق بين P-cores وE-cores، فبيبعت الـ compute-intensive tasks على E-cores والـ light tasks على P-cores.

```bash
$ cat /proc/cmdline
... intel_pstate=no_cas ...

$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
intel_pstate

# تحقق من الـ CPU capacities
$ for cpu in /sys/devices/system/cpu/cpu*/cpu_capacity; do
    echo "$cpu: $(cat $cpu)"
  done
# المفروض يظهر قيم مختلفة لـ P-cores وE-cores
# لكن مع no_cas كل الـ CPUs بيظهروا capacity متساوية
```

#### التحليل
الـ documentation بتشرح:

> *"The capacity-aware scheduling (CAS) support in the CPU scheduler is enabled by intel_pstate by default on hybrid processors without SMT."*

> *"the capacity of each CPU is represented by the ratio of its highest HWP performance level, multiplied by 1024, to the highest HWP performance level of the most performant CPU in the system"*

مع `no_cas`، الـ `cpu_capacity` بيبقى متساوي لكل الـ CPUs، وبالتالي:

```
بدون CAS (no_cas):
┌─────────────┬──────────────┐
│  P-core 0   │ capacity=1024│ ← scheduler مش عارف إنه أقوى
│  P-core 1   │ capacity=1024│
│  E-core 8   │ capacity=1024│ ← scheduler مش عارف إنه أضعف
│  E-core 9   │ capacity=1024│
└─────────────┴──────────────┘

مع CAS (default):
┌─────────────┬──────────────┐
│  P-core 0   │ capacity=1024│ ← max (الـ reference)
│  P-core 1   │ capacity=1024│
│  E-core 8   │ capacity= 640│ ← أقل من الـ P-core
│  E-core 9   │ capacity= 640│
└─────────────┴──────────────┘
```

فالـ heavy tasks بتروح E-cores بالصدفة وتتبطأ.

#### الحل
أزل `intel_pstate=no_cas` من الـ cmdline وتحقق إن الـ kernel version جديد بما يكفي:

```bash
# خطوة 1: أزل no_cas من GRUB
# في /etc/default/grub:
# قبل:  GRUB_CMDLINE_LINUX="... intel_pstate=no_cas ..."
# بعد:  GRUB_CMDLINE_LINUX="..."

$ sudo update-grub && sudo reboot

# خطوة 2: بعد الـ reboot، تحقق من الـ CAS activation
$ for cpu in /sys/devices/system/cpu/cpu*/cpu_capacity; do
    echo "$(basename $(dirname $cpu)): $(cat $cpu)"
  done
# cpu0: 1024  ← P-core
# cpu8: 640   ← E-core

# خطوة 3: تحقق من الـ Energy Model (لو CONFIG_ENERGY_MODEL مفعّل)
$ ls /sys/kernel/debug/energy_model/
# هيظهر directories للـ performance domains
```

للتأكد إن الـ CAS شغال:

```bash
# شغّل task ثقيل وشوف هيروح على أي CPU
$ taskset -c 0-23 stress-ng --cpu 1 &
$ top -d1 | grep stress
# المفروض يظهر على P-core (cpu0-7) مش E-core (cpu8-23)
```

#### الدرس المستفاد
في الـ hybrid Intel processors (Alder Lake وما بعده)، `intel_pstate=no_cas` له تأثير كبير جدًا على الـ performance. الـ CAS هو اللي بيخلي الـ scheduler يفرّق بين P-cores وE-cores. لو عندك مشكلة قديمة خلتك تعطله، راجع الـ kernel version الحالي — المشكلة ممكن تكون اتحلت. دايمًا تحقق من `cpu_capacity` قبل وبعد أي تغيير في الـ cmdline على hybrid systems.

---

### السيناريو الخامس: IoT Edge Server على Intel Atom x6425E — الـ power budget متجاوز في الـ data center

#### العنوان
IoT Edge Server بـ Intel Atom x6425E (Elkhart Lake) — الـ rack power budget بيتجاوز وبيتسبب في thermal shutdown

#### السياق
شركة IoT بتشغّل edge servers في data center. كل rack بيحتوي على 40 server، كل واحد بيشتغل على Intel Atom x6425E (Elkhart Lake). الـ total power budget للـ rack هو 2 kW. في أوقات الـ peak load (بيعالجوا sensor data من آلاف الـ IoT devices)، الـ rack بيسحب 2.4 kW وبيحصل thermal shutdown.

#### المشكلة
كل الـ servers شغّالة بـ `intel_pstate` في الـ active mode بـ `performance` governor. الـ `max_perf_pct` = 100 والـ turbo مفعّل. وقت الـ peak load، كل الـ CPUs بتروح للـ turbo P-states وبترفع الـ power فوق الـ budget.

```bash
# على كل server
$ cat /sys/devices/system/cpu/intel_pstate/max_perf_pct
100

$ cat /sys/devices/system/cpu/intel_pstate/no_turbo
0

$ cat /sys/devices/system/cpu/intel_pstate/turbo_pct
30    # 30% من الـ P-states هي turbo

$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
performance
# بيختار دايمًا max P-state = turbo
```

#### التحليل
الـ documentation بتشرح الـ P-state limits coordination:

> *"All CPUs are affected by the global limits (that is, none of them can be requested to run faster than the global maximum)"*

الـ `max_perf_pct` هو global limit بيأثر على كل الـ CPUs. حسب الـ documentation:

> *"If the no_turbo global attribute is set, the driver is not allowed to use turbo P-states, so the maximum value of scaling_max_freq and scaling_min_freq is limited to the maximum non-turbo P-state frequency."*

الحل: إما نعطّل الـ turbo كليًا أو نقلّل الـ `max_perf_pct` لتحديد power ceiling.

```
Power behavior:
                        turbo zone
┌──────────────────────────────────────┐
│ max_perf_pct=100 + turbo ON          │ ← 2.4 kW (فوق الـ budget)
│ ████████████████████████████████████ │
└──────────────────────────────────────┘

                    non-turbo zone only
┌──────────────────────────────────────┐
│ no_turbo=1 OR max_perf_pct=70        │ ← 1.8 kW (تحت الـ budget)
│ ██████████████████████████░░░░░░░░░░ │
└──────────────────────────────────────┘
```

#### الحل
**الحل الفوري (runtime):**

```bash
#!/bin/bash
# اضبط على كل الـ servers في الـ rack

# خيار 1: عطّل الـ turbo كليًا
echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo

# أو خيار 2: قلّل الـ max_perf_pct لتحديد power ceiling
# (70% من max turbo = roughly non-turbo max على Elkhart Lake)
echo 70 > /sys/devices/system/cpu/intel_pstate/max_perf_pct

# تحقق من التأثير
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
# المفروض يقل من 3000000 Hz (turbo) لـ 2100000 Hz (base)
```

**الحل الدائم والأذكى — dynamic power management script:**

```bash
#!/bin/bash
# /usr/local/bin/power-manager.sh
# يشتغل كـ systemd service

POWER_BUDGET_WATTS=45   # per server (2000W / 40 servers = 50W, نحتاط بـ 10%)
PSTATE_DIR="/sys/devices/system/cpu/intel_pstate"

while true; do
    # اقرأ الـ current power draw (لو عندك RAPL)
    POWER=$(cat /sys/class/powercap/intel-rapl/intel-rapl:0/energy_uj)
    sleep 1
    POWER_NEW=$(cat /sys/class/powercap/intel-rapl/intel-rapl:0/energy_uj)
    WATTS=$(( (POWER_NEW - POWER) / 1000000 ))

    if [ "$WATTS" -gt "$POWER_BUDGET_WATTS" ]; then
        # قلّل الـ max_perf_pct
        CURRENT=$(cat ${PSTATE_DIR}/max_perf_pct)
        NEW=$(( CURRENT - 5 ))
        [ "$NEW" -lt 50 ] && NEW=50
        echo $NEW > ${PSTATE_DIR}/max_perf_pct
    elif [ "$WATTS" -lt $(( POWER_BUDGET_WATTS - 5 )) ]; then
        # زوّد الـ max_perf_pct
        CURRENT=$(cat ${PSTATE_DIR}/max_perf_pct)
        NEW=$(( CURRENT + 5 ))
        [ "$NEW" -gt 100 ] && NEW=100
        echo $NEW > ${PSTATE_DIR}/max_perf_pct
    fi

    sleep 5
done
```

**الحل في الـ kernel cmdline:**

```bash
# في /etc/default/grub على كل server
GRUB_CMDLINE_LINUX="intel_pstate=passive"
# في passive mode، schedutil مع energy-aware scheduling بيدير الـ power أحسن
# وممكن نستخدم cpufreq.off=1 على بعض الـ servers في أوقات الـ peak
```

#### التأكد من الحل

```bash
# تحقق من إن الـ no_turbo اشتغل
$ cat /sys/devices/system/cpu/intel_pstate/no_turbo
1

# لاحظ إن cpuinfo_max_freq ما اتغيرش
$ cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq
3000000   # لسه بيظهر الـ max turbo

# لكن scaling_max_freq اتغير
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
2100000   # non-turbo max فقط

# الـ documentation بتوضح ده:
# "This attribute does not affect the maximum supported frequency value
#  supplied to the CPUFreq core... but it affects the maximum possible
#  value of per-policy P-state limits"
```

#### الدرس المستفاد
الـ `no_turbo` global attribute في `intel_pstate` هو أسرع طريقة لتقليل الـ power consumption في dense deployments. الـ `max_perf_pct` بيدي تحكم أدق. مهم تعرف إن `cpuinfo_max_freq` مش بيتغير مع `no_turbo` — بس `scaling_max_freq` هو اللي بيتقيّد. في الـ IoT edge deployments اللي فيها power budgets صارمة، ادرس الـ RAPL (Running Average Power Limit) interface جنب `intel_pstate` عشان تعمل dynamic power capping دقيق.
## Phase 7: مصادر ومراجع

### الـ Official Kernel Documentation

| المستند | الوصف |
|---------|-------|
| [`Documentation/admin-guide/pm/intel_pstate.rst`](https://www.kernel.org/doc/html/latest/admin-guide/pm/intel_pstate.html) | الـ reference الرسمي والشامل لـ `intel_pstate` — المصدر الأول لأي سؤال |
| [`Documentation/admin-guide/pm/cpufreq.rst`](https://www.kernel.org/doc/html/latest/admin-guide/pm/cpufreq.html) | شرح الـ `CPUFreq` subsystem اللي `intel_pstate` جزء منه |
| [`Documentation/admin-guide/pm/working-state.rst`](https://static.lwn.net/kerneldoc/admin-guide/pm/working-state.html) | نظرة عامة على الـ working-state power management في الكيرنل |

---

### الـ LWN.net Articles

دي أهم المقالات من LWN.net اللي اتعملت بـ search حقيقي:

1. **[Add P state driver for Intel Core Processors](https://lwn.net/Articles/536017/)** — الـ patch الأصلي اللي دخّل `intel_pstate` للكيرنل في 2013، من Dirk Brandewie. لازم تقرأه عشان تفهم القرار الأصلي.

2. **[Intel_pstate: HWP Dynamic performance boost](https://lwn.net/Articles/754564/)** — patch series بتشرح الـ `hwp_dynamic_boost` feature اللي بتزوّد الـ minimum P-state مؤقتاً لما task بعد I/O wait بتبدأ.

3. **[Documentation/cpu-freq: add intel-pstate.txt](https://lwn.net/Articles/578959/)** — أول documentation رسمية اتضافت للـ `intel_pstate` في الكيرنل.

4. **[Van de Ven: Some basics on CPU P states on Intel processors](https://lwn.net/Articles/556484/)** — مقال تاسيسي من Arjan van de Ven بيشرح الـ P-states على Intel hardware من الأول، ضروري للفهم الأعمق.

5. **[Support Intel Turbo Boost Max Technology 3.0](https://lwn.net/Articles/706156/)** — بيتكلم على الـ per-core turbo boost وازاي `intel_pstate` بيتعامل معاه.

---

### الـ Kernel Source & Commits

#### الـ Main Driver File
```
drivers/cpufreq/intel_pstate.c
```
**الـ GitHub:** [torvalds/linux — intel_pstate.c](https://github.com/torvalds/linux/blob/master/drivers/cpufreq/intel_pstate.c)

#### الـ Kconfig
```
drivers/cpufreq/Kconfig.x86
```
بيحتوي على الـ `CONFIG_X86_INTEL_PSTATE` option والـ dependencies.

#### أهم الـ Commits التاريخية

| الـ Feature | المصدر |
|------------|--------|
| أول إضافة للـ driver (Sandy Bridge, 2013) | [spinics.net patch](https://www.spinics.net/lists/cpufreq/msg04232.html) |
| Passive mode with HWP enabled | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-pm/patch/5605271.5WD5kalLAl@kreacher/) |
| HWP Guaranteed change notification | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-pm/patch/20210928164217.950960-1-srinivas.pandruvada@linux.intel.com/) |

---

### الـ Mailing List Discussions

- **[Passive mode with HWP — Patchwork](https://patchwork.kernel.org/project/linux-pm/patch/3955470.QvD6XneCf3@kreacher/)** — نقاش مهم على linux-pm mailing list عن تفعيل الـ passive mode مع HWP وضبط الـ HWP floor.

- **[Force intel_pstate to load when HWP disabled in firmware](https://lore.kernel.org/all/fb6c8a4e284a9b6c043f4ac382387b19bd100976.camel@linux.intel.com/T/)** — lore.kernel.org thread عن سيناريوهات الـ fallback لما الـ HWP متمكّنش في firmware.

- **الـ linux-pm mailing list:** لأي نقاش حديث، ابحث على `lore.kernel.org/linux-pm/` بكلمة `intel_pstate`.

---

### الـ KernelNewbies.org — تتبع التغييرات عبر الإصدارات

الموقع بيوثّق التغييرات في كل kernel release:

| الإصدار | الـ Feature |
|---------|------------|
| [Linux 3.17](https://kernelnewbies.org/Linux_3.17-DriversArch) | إضافة مبكرة لـ features في الـ driver |
| [Linux 4.10](https://kernelnewbies.org/Linux_4.10) | Per-logical-CPU P-state limits + EPP/EPB knobs |
| [Linux 4.12](https://kernelnewbies.org/Linux_4.12) | تحسينات على الـ HWP integration |
| [Linux 5.7](https://kernelnewbies.org/Linux_5.7) | Default governor أصبح `schedutil` مع frequency invariance |
| [Linux 5.9](https://kernelnewbies.org/Linux_5.9) | تحسينات إضافية على الـ passive mode |
| [Linux 5.14](https://kernelnewbies.org/Linux_5.14) | HWP على hybrid processors + Icelake server support |
| [Linux 6.1](https://kernelnewbies.org/Linux_6.1) | Tigerlake support في no-HWP mode |
| [Linux 6.11](https://kernelnewbies.org/Linux_6.11) | Emerald Rapids EPP updates + OOB mode |
| [Linux 6.12](https://kernelnewbies.org/Linux_6.12) | Hybrid CPU capacity scaling + Granite Rapids OOB |

---

### الـ eLinux.org

البحث على elinux.org ما لقاش pages مخصصة لـ `intel_pstate` — الموقع ده أساساً بيتكلم عن embedded Linux وـ ARM systems. المرجع الأنسب للـ Intel power management هو kernel.org وـ LWN.net.

---

### الـ Intel Official References

| المرجع | الرابط |
|--------|--------|
| **Intel SDM Vol. 3** — System Programming Guide (P-states, MSRs, HWP) | [intel.com](https://www.intel.com/content/www/us/en/architecture-and-technology/64-ia-32-architectures-software-developer-system-programming-manual-325384.html) |
| **ACPI Specification** (للـ `_PSS` objects) | [uefi.org](https://uefi.org/sites/default/files/resources/ACPI_6_3_final_Jan30.pdf) |
| **LinuxCon Europe 2015 — Balancing Power and Performance** (Kristen Accardi) | [linuxfound.org slides](https://events.static.linuxfound.org/sites/events/files/slides/LinuxConEurope_2015.pdf) |
| **Lenovo Press — Using P-States with Linux on Intel Servers** | [lenovopress.com](https://lenovopress.lenovo.com/lp1946-using-processor-performance-p-states-with-linux-on-intel-based-servers) |

---

### الـ Recommended Books

#### Linux Device Drivers (LDD3)
- **الفصل الأهم:** Chapter 14 — *The Linux Device Model*
- الكتاب ما بيتكلمش على `intel_pstate` بشكل مباشر، بس بيشرح الـ `sysfs` infrastructure اللي الـ driver بيستخدمها لـ expose الـ attributes.
- **تحميل مجاني:** [lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل الأهم:** Chapter 11 — *Timers and Time Management* (للـ periodic callbacks)
- Chapter 14 — *The Block I/O Layer* (للفهم العام للـ subsystem model)
- الـ `cpufreq` و`intel_pstate` بيتبعوا نفس نمط الـ kernel subsystem design اللي الكتاب بيشرحه.

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل الأهم:** Chapter 15 — *Kernel Initialization*
- مفيد لفهم ازاي الـ drivers بتتسجّل في وقت الـ boot وازاي الـ kernel command line options بتاثّر فيهم زي `intel_pstate=passive`.

---

### الـ ArchWiki (مرجع عملي)

**[CPU frequency scaling — ArchWiki](https://wiki.archlinux.org/title/CPU_frequency_scaling)**
شرح عملي ممتاز بيغطي:
- الفرق بين `intel_pstate` و`acpi-cpufreq`
- ازاي تشوف الـ current governor والـ mode
- الـ tuning options من الـ user space

---

### Search Terms للبحث عن معلومات أكثر

```
intel_pstate site:lwn.net
intel_pstate site:lore.kernel.org
cpufreq HWP hardware managed p-states linux
intel_pstate passive mode schedutil
EPP energy performance preference linux kernel
intel_pstate turbo boost MSR
CONFIG_X86_INTEL_PSTATE kernel config
intel_pstate hybrid processor CAS capacity aware scheduling
intel_pstate energy model EAS
intel_pstate no_hwp command line
```
## Phase 8: Writing simple module

### الفكرة

**`intel_pstate`** بيسجّل utilization update callbacks مع الـ CPU scheduler، وده بيحصل من خلال الـ `cpufreq_transition_notifier` اللي الـ CPUFreq core بيبعث منه events لما الـ frequency بتتغير. هنعمل module بيستخدم **`cpufreq_register_notifier`** عشان نتفرّج على كل تغيير في الـ P-state (frequency) على أي CPU في الجهاز — ده مفيد جداً في تشخيص سلوك `intel_pstate` في الـ active أو passive mode.

---

### الـ Module كامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * intel_pstate_watch.c
 *
 * Monitor every CPUFreq frequency transition on the system.
 * Works with intel_pstate (active/passive) and any other cpufreq driver.
 *
 * Hook used: cpufreq_register_notifier(CPUFREQ_TRANSITION_NOTIFIER)
 * Safe to load/unload at runtime.
 */

#include <linux/module.h>        /* MODULE_LICENSE, module_init/exit        */
#include <linux/kernel.h>        /* pr_info                                 */
#include <linux/cpufreq.h>       /* cpufreq_register/unregister_notifier,
                                    struct cpufreq_freqs, CPUFREQ_POSTCHANGE */
#include <linux/notifier.h>      /* struct notifier_block, NOTIFY_OK        */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("Monitor intel_pstate P-state transitions via cpufreq notifier");

/* -----------------------------------------------------------------------
 * الـ callback اللي بيتنادى كل ما CPU غيّرت الـ P-state (frequency).
 * بييجي في CPUFREQ_PRECHANGE (قبل التغيير) و CPUFREQ_POSTCHANGE (بعده).
 * إحنا بنطبع بس بعد التغيير عشان نشوف الـ old و new معاً.
 * ----------------------------------------------------------------------- */
static int pstate_notifier_call(struct notifier_block *nb,
                                unsigned long event,
                                void *data)
{
    struct cpufreq_freqs *freqs = data; /* frequency info struct */

    /* Only act after the hardware has already changed frequency */
    if (event != CPUFREQ_POSTCHANGE)
        return NOTIFY_OK;

    pr_info("intel_pstate_watch: cpu=%u  %u kHz -> %u kHz  (flags=0x%x)\n",
            freqs->policy->cpu,   /* logical CPU number                     */
            freqs->old,           /* old frequency in kHz                   */
            freqs->new,           /* new frequency in kHz (new P-state)     */
            freqs->flags);        /* CPUFREQ_IS_COOLING_DEV etc.            */

    return NOTIFY_OK; /* always return OK so the chain keeps going */
}

/*
 * struct notifier_block — الـ kernel بيستخدمه عشان يعرف أي function يندهلها.
 * priority=0 معناها ترتيب عادي في الـ notification chain.
 */
static struct notifier_block pstate_nb = {
    .notifier_call = pstate_notifier_call,
    .priority      = 0,
};

/* -----------------------------------------------------------------------
 * module_init: بنسجّل الـ notifier مع الـ CPUFreq transition chain.
 * لو الـ registration فشلت بنرجع error فوراً من غير ما نسيب حاجة معلّقة.
 * ----------------------------------------------------------------------- */
static int __init pstate_watch_init(void)
{
    int ret;

    /* CPUFREQ_TRANSITION_NOTIFIER = the chain for frequency change events */
    ret = cpufreq_register_notifier(&pstate_nb, CPUFREQ_TRANSITION_NOTIFIER);
    if (ret) {
        pr_err("intel_pstate_watch: failed to register notifier (%d)\n", ret);
        return ret;
    }

    pr_info("intel_pstate_watch: loaded — watching all P-state transitions\n");
    return 0;
}

/* -----------------------------------------------------------------------
 * module_exit: لازم نلغي الـ registration قبل ما الـ module يتـunload،
 * لأن الـ kernel ممكن يندهّ الـ callback على pointer اتمسح → kernel panic.
 * ----------------------------------------------------------------------- */
static void __exit pstate_watch_exit(void)
{
    cpufreq_unregister_notifier(&pstate_nb, CPUFREQ_TRANSITION_NOTIFIER);
    pr_info("intel_pstate_watch: unloaded\n");
}

module_init(pstate_watch_init);
module_exit(pstate_watch_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | ماكروهات `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info` / `pr_err` للطباعة في الـ kernel log |
| `linux/cpufreq.h` | `cpufreq_register_notifier`, `struct cpufreq_freqs`, `CPUFREQ_POSTCHANGE` |
| `linux/notifier.h` | `struct notifier_block`, `NOTIFY_OK` |

**الـ `linux/cpufreq.h`** هو اللب — فيه كل الـ API الخاص بـ `intel_pstate` وأي driver تاني.

---

#### الـ `pstate_notifier_call` — الـ Callback

```
event == CPUFREQ_POSTCHANGE
```

الـ CPUFreq core بيبعت `CPUFREQ_PRECHANGE` قبل التغيير و`CPUFREQ_POSTCHANGE` بعده. إحنا بنفلتر `POSTCHANGE` بس عشان في اللحظة دي `freqs->old` و`freqs->new` الاتنين متاحين وواضحين — مش محتاجين نتفرّج على نص الصورة.

**`struct cpufreq_freqs`** بتشيل:
- `policy->cpu` ← رقم الـ logical CPU
- `old` / `new` ← الـ frequency بالـ kHz قبل وبعد التغيير (= P-state mapping)
- `flags` ← `CPUFREQ_IS_COOLING_DEV` لو الـ thermal subsystem هو اللي طلب التغيير

ده بالظبط اللي `intel_pstate` بيغيّره كل ما الـ scheduler يطلب P-state جديد — فإحنا بنشوف كل قرار.

---

#### الـ `struct notifier_block`

الـ kernel بيشيل الـ notifier chains كـ linked list من `notifier_block` structures. كل واحد فيها بيشير لـ function. الـ `priority=0` يخلينا في المنتصف — مش هنقطع على أي حاجة تانية مسجّلة.

---

#### الـ `module_init`

`cpufreq_register_notifier(&pstate_nb, CPUFREQ_TRANSITION_NOTIFIER)` بتضيف الـ `pstate_nb` لقائمة المستمعين على كل frequency transitions في الـ system — سواء من `intel_pstate` أو `acpi-cpufreq` أو أي driver. لو فشلت (مثلاً الـ cpufreq مش مُفعّل) بنرجع الـ error code مباشرة.

---

#### الـ `module_exit`

`cpufreq_unregister_notifier` ضرورية جداً: لو الـ module اتـunload وسابّ الـ `pstate_nb` مسجّلاً، الـ kernel هيحاول ينادي `pstate_notifier_call` على address اتمسح من الـ memory → **use-after-free / kernel panic**. الـ unregister بيشيل الـ pointer من الـ chain بأمان قبل ما الكود يتحرر.

---

### تشغيل وتجربة

```bash
# بناء الـ module (Makefile بسيط مع obj-m)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل
sudo insmod intel_pstate_watch.ko

# تشغيل حِمل على CPU عشان تتحرك الـ P-states
stress-ng --cpu 2 --timeout 5

# مشاهدة الـ output
sudo dmesg | grep intel_pstate_watch

# تفريغ
sudo rmmod intel_pstate_watch
```

**مثال على الـ output المتوقع:**

```
[  142.301] intel_pstate_watch: loaded — watching all P-state transitions
[  142.451] intel_pstate_watch: cpu=0  800000 kHz -> 3600000 kHz  (flags=0x0)
[  142.452] intel_pstate_watch: cpu=2  800000 kHz -> 3600000 kHz  (flags=0x0)
[  147.510] intel_pstate_watch: cpu=0  3600000 kHz -> 800000 kHz  (flags=0x0)
[  147.980] intel_pstate_watch: unloaded
```

الـ `flags=0x1` (CPUFREQ_IS_COOLING_DEV) تظهر لو الـ thermal throttling هو اللي بدّل الـ P-state مش الـ scheduler.
