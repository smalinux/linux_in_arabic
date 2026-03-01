## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem ده؟

الملف ده جزء من **CPUFreq subsystem** في Linux kernel — وده الـ subsystem المسؤول عن التحكم في تردد (سرعة) الـ CPU وجهد التشغيل بتاعه ديناميكيًا أثناء تشغيل النظام. الـ subsystem ده بيسمح للـ kernel إنه يرفع سرعة الـ CPU لما في حِمل عالي، ويخفضها لما الجهاز هادي — عشان يوفر استهلاك الطاقة ويطوّل عمر البطارية.

الملف الموثَّق هنا (`cpufreq-stats.rst`) بيشرح **cpufreq-stats driver** — وهو driver مخصص بالكامل للإحصائيات: بيراقب ويسجّل كل تغييرات تردد الـ CPU ويعرضها للمستخدم عبر الـ **sysfs**.

---

### القصة: ليه محتاجين إحصائيات؟

تخيّل عندك لاب توب وعايز تعرف:
- الـ CPU بتاعك بيشتغل بأعلى سرعة قد إيه من وقت التشغيل؟
- بيفضل في وضع التوفير قد إيه؟
- بيتنقل بين الترددات كتير ولا نادر؟

من غير إحصائيات، مش هتعرف تجاوب على السؤال ده. هنا بييجي دور الـ **cpufreq-stats driver** — بيشتغل جنب أي **cpufreq_driver** موجود على أي CPU، وبيسجّل كل انتقال تردد بيحصل، وبيعرض البيانات دي في ملفات read-only تحت `/sys/devices/system/cpu/cpuX/cpufreq/stats/`.

---

### الهدف من الملف

الملف `cpufreq-stats.rst` هو **توثيق للمستخدمين** (user-facing documentation) بيشرح:
1. إزاي تفهم الإحصائيات اللي بيقدمها الـ driver ده.
2. معنى كل ملف في الـ sysfs directory بتاعت الـ stats.
3. إزاي تفعّل الـ driver ده في الـ kernel configuration.

مش بيشرح كود — بيشرح **الواجهة** اللي المستخدم والـ developer بيتعاملوا معاها.

---

### الإحصائيات اللي بيقدمها الـ Driver

الـ driver بيعرض 3 ملفات رئيسية في `/sys/devices/system/cpu/cpuX/cpufreq/stats/`:

| الملف | النوع | الوصف |
|---|---|---|
| `time_in_state` | read-only | الوقت اللي قضاه الـ CPU في كل تردد (بوحدة 10ms) |
| `total_trans` | read-only | إجمالي عدد مرات الانتقال بين الترددات |
| `trans_table` | read-only | مصفوفة ثنائية الأبعاد: كام مرة انتقل من تردد X لتردد Y |
| `reset` | write-only | بيصفّر العدادات بدون ما تعيد تشغيل الجهاز |

#### مثال واقعي على `time_in_state`:
```
3600000 2089    ← قضى 20.89 ثانية عند 3.6GHz
3400000 136     ← قضى 1.36 ثانية عند 3.4GHz
2800000 172488  ← قضى 1724.88 ثانية عند 2.8GHz (وضع التوفير الأساسي)
```

#### مثال واقعي على `trans_table`:
```
From  :    To
      :  3600000  3400000  2800000
3600000:        0        5        0
3400000:        4        0        2
2800000:        0        0        0
```
الـ entry [3600000][3400000] = 5 يعني: الـ CPU انتقل من 3.6GHz لـ 3.4GHz خمس مرات.

---

### ليه ده مفيد؟

- **Developers** بيستخدموه عشان يفهموا سلوك الـ governor بتاعهم — هل بيتغير التردد كتير ولا لأ.
- **Power engineers** بيستخدموه عشان يحسّنوا استهلاك الطاقة — الـ CPU بيقعد في الـ low freq قد إيه؟
- **Testing** — تقدر تعمل `reset` قبل الاختبار وتقيس بعده بدون ما تعيد تشغيل الجهاز.
- **الـ governor comparison** — جرّب governor A، اتفرّج على الإحصائيات، جرّب governor B، قارن.

---

### إزاي بيشتغل الـ Driver (بالبساطة)

```
CPU يغيّر تردده
       ↓
cpufreq core يستدعي cpufreq_stats_record_transition()
       ↓
الـ driver يسجّل:
  - الوقت اللي قضاه في التردد القديم (time_in_state)
  - الانتقال من التردد القديم للجديد (trans_table)
  - يزوّد total_trans بـ 1
       ↓
المستخدم يقرأ من sysfs → يشوف الإحصائيات
```

الـ driver ذكي — بيستخدم **deferred reset** عشان يتجنب الـ race conditions: لما تكتب في `reset`، مش بيصفّر فورًا، بل بيحط flag ويصفّر في أول انتقال تردد جاي.

---

### الـ Kernel Configuration

لتفعيل الـ driver:

```
Config Main Menu
  → Power management options (ACPI, APM)
    → CPU Frequency scaling
      → [*] CPU Frequency scaling          (CONFIG_CPU_FREQ)
      → [*]   CPU frequency translation statistics  (CONFIG_CPU_FREQ_STAT)
```

---

### الملفات المكوّنة للـ Subsystem

#### Core:
| الملف | الدور |
|---|---|
| `drivers/cpufreq/cpufreq.c` | الـ CPUFreq core — اللي يربط كل حاجة ببعض |
| `drivers/cpufreq/cpufreq_stats.c` | الـ stats driver نفسه — بيسجّل الإحصائيات |
| `drivers/cpufreq/freq_table.c` | إدارة جدول الترددات المدعومة |

#### Headers:
| الملف | الدور |
|---|---|
| `include/linux/cpufreq.h` | تعريف `struct cpufreq_policy`، `struct cpufreq_stats`، وكل الـ APIs |

#### Governors (من يقرر متى يغير التردد):
| الملف | الدور |
|---|---|
| `drivers/cpufreq/cpufreq_ondemand.c` | governor بيرفع التردد عند الحاجة |
| `drivers/cpufreq/cpufreq_conservative.c` | governor أكثر تدرجًا في رفع التردد |
| `drivers/cpufreq/cpufreq_performance.c` | دايمًا أعلى تردد |
| `drivers/cpufreq/cpufreq_powersave.c` | دايمًا أقل تردد |
| `drivers/cpufreq/intel_pstate.c` | governor مخصص لـ Intel processors |

#### Hardware Drivers (من يُنفّذ تغيير التردد فعليًا):
| الملف | الدور |
|---|---|
| `drivers/cpufreq/acpi-cpufreq.c` | تغيير التردد عبر ACPI |
| `drivers/cpufreq/amd-pstate.c` | AMD P-State driver |
| `drivers/cpufreq/cpufreq-dt.c` | Generic driver لأجهزة ARM وغيرها عبر Device Tree |

#### Documentation:
| الملف | الدور |
|---|---|
| `Documentation/cpu-freq/core.rst` | شرح الـ CPUFreq core وآلية الـ notifiers |
| `Documentation/cpu-freq/cpu-drivers.rst` | إزاي تكتب cpufreq driver جديد |
| `Documentation/cpu-freq/cpufreq-stats.rst` | **الملف الحالي** — شرح إحصائيات الـ sysfs |
## Phase 2: شرح الـ cpufreq-stats Framework

### المشكلة اللي بتحلها

الـ CPU مش بيشتغل على تردد ثابت — الـ governor (زي `schedutil` أو `ondemand`) بيغيّر التردد باستمرار حسب الـ load. السؤال العملي هو: **إيه اللي بيحصل فعلاً؟** هل الـ CPU بيقضي معظم وقته على أعلى تردد أو على الأقل؟ كام مرة اتغيّر التردد؟

من غير قياس، مش ممكن تعرف:
- هل الـ governor بيشتغل صح؟
- هل في power waste لأن الـ CPU بيقعد على high freq من غير لازمه؟
- إيه الـ transition pattern بين الترددات المختلفة؟

الـ `cpufreq-stats` جه عشان يجاوب على الأسئلة دي من غير ما يكون جزء من أي driver معين — هو **observer مستقل** يراقب كل policy ويسجّل التاريخ.

---

### الـ Solution — إزاي الـ kernel حلّها؟

الكرنل أضاف `cpufreq_stats` كـ **layer مستقل** يتضمّن في الـ `cpufreq_policy`. أي cpufreq driver بيشتغل، الـ stats layer بيتفعّل تلقائياً لأنه متعلّق بالـ policy مش بالـ driver.

الـ design مبني على:
1. **الـ stats بتتسجّل على كل `cpufreq_policy`** — مش على كل CPU منفرداً، لأن الـ CPUs اللي بتشارك clock بتتحكم فيهم بـ policy واحدة.
2. **الـ hooks موجودة في الـ cpufreq core** — لما يحصل transition، الـ core بيعمل call لـ `cpufreq_stats_record_transition()` تلقائياً.
3. **الـ sysfs interface** — الـ stats بتظهر في `/sys/devices/system/cpu/cpuX/cpufreq/stats/` كـ read-only files.

---

### الـ Analogy الواقعية — Flight Data Recorder

تخيّل الـ CPU زي طيارة. الـ governor هو الـ autopilot اللي بيقرر الـ engine throttle (التردد). الـ `cpufreq-stats` هو الـ **Flight Data Recorder (FDR)** — الصندوق الأسود.

| الـ FDR | الـ cpufreq-stats |
|---------|-----------------|
| بيسجّل altitude على مدار الرحلة | `time_in_state` — كام وقت على كل تردد |
| بيحسب كام مرة تغيّر الـ throttle | `total_trans` — عدد التغييرات الكلي |
| بيسجّل من أي altitude لأي altitude | `trans_table` — مصفوفة التحولات |
| مش بيتدخل في قرارات الـ autopilot | الـ stats driver مش بيأثر على الـ governor |
| بيتفعّل من بداية الرحلة | من وقت load الـ module أو آخر reset |
| ممكن تعمل reset للسجل | الـ `reset` attribute |

---

### الـ Big Picture Architecture

```
+----------------------------------------------------------+
|                    User Space                            |
|   cat /sys/devices/system/cpu/cpu0/cpufreq/stats/        |
|        time_in_state | total_trans | trans_table          |
+------------------------|---------------------------------+
                         | sysfs read/write
+------------------------v---------------------------------+
|               cpufreq-stats sysfs layer                  |
|   show_time_in_state()  show_total_trans()               |
|   show_trans_table()    store_reset()                     |
+------------------------|---------------------------------+
                         | reads from
+------------------------v---------------------------------+
|            struct cpufreq_stats (per policy)             |
|  +------------------+  +----------------------------+   |
|  | total_trans: 20  |  | time_in_state[]: u64 array  |  |
|  | last_index: 0    |  | [0]=2089, [1]=136, ...      |  |
|  | last_time: ns    |  +----------------------------+   |
|  | max_state: 5     |  +----------------------------+   |
|  | state_num: 5     |  | freq_table[]: uint array    |  |
|  | reset_pending: 0 |  | [0]=3600000, [1]=3400000,...|  |
|  +------------------+  +----------------------------+   |
|                        +----------------------------+   |
|                        | trans_table[NxN]: uint[]    |  |
|                        | [i*N+j] = count(i->j)       |  |
|                        +----------------------------+   |
+------------------------|---------------------------------+
                         ^ written by
+------------------------|---------------------------------+
|            cpufreq_stats_record_transition()             |
|   called by cpufreq CORE on every freq change            |
+------------------------|---------------------------------+
                         ^ triggered by
+----------------------------------------------------------+
|              cpufreq Governors                           |
|   schedutil / ondemand / conservative / userspace        |
+----------------------------------------------------------+
                         |
+------------------------v---------------------------------+
|              cpufreq Drivers (Hardware)                  |
|   intel_pstate / arm-cpufreq / qcom-cpufreq / ...        |
|   (any driver works — stats is driver-agnostic)          |
+----------------------------------------------------------+
```

---

### الـ Core Abstraction — struct cpufreq_stats

الـ central idea هو إن كل `cpufreq_policy` بيحمل **pointer لـ `struct cpufreq_stats`** — اللي هو الـ state machine اللي بيراقب الانتقالات:

```c
struct cpufreq_stats {
    unsigned int   total_trans;    /* عدد التحولات الكلي */
    unsigned long long last_time;  /* nanoseconds — آخر مرة اتحدثنا فيها */
    unsigned int   max_state;      /* عدد الترددات المخصصة في الـ arrays */
    unsigned int   state_num;      /* عدد الترددات الفعلية (unique) */
    unsigned int   last_index;     /* index التردد الحالي في freq_table */
    u64           *time_in_state;  /* array: وقت (ns) لكل تردد */
    unsigned int  *freq_table;     /* array: قيم الترددات بالـ kHz */
    unsigned int  *trans_table;    /* matrix NxN: trans[i][j] = عدد مرات i->j */
    unsigned int   reset_pending;  /* flag: في reset منتظر؟ */
    unsigned long long reset_time; /* توقيت الـ reset */
};
```

**الـ Memory Layout الفعلي:**

الكود بيعمل `kzalloc` واحد لكل الـ arrays:

```c
/* من cpufreq_stats_create_table() */
alloc_size = count * sizeof(u64)           /* time_in_state */
           + count * sizeof(unsigned int)  /* freq_table */
           + count * count * sizeof(int);  /* trans_table (NxN matrix) */

stats->time_in_state = kzalloc(alloc_size, GFP_KERNEL);
stats->freq_table    = (unsigned int *)(stats->time_in_state + count);
stats->trans_table   = stats->freq_table + count;
```

```
Allocated block:
+----------------+----------------+-----------------------+
| time_in_state  |  freq_table    |     trans_table       |
|  [N * u64]     |  [N * uint]    |   [N*N * uint]        |
+----------------+----------------+-----------------------+
^                ^                ^
stats->time_in_state  stats->freq_table  stats->trans_table
```

---

### الـ struct cpufreq_policy — ربط الكل ببعض

الـ `cpufreq_policy` هو الـ central struct في الـ cpufreq subsystem. كل الـ CPUs اللي بتشارك clock بيتحكم فيهم بـ policy واحدة:

```c
struct cpufreq_policy {
    cpumask_var_t   cpus;           /* CPUs Online sharing this policy */
    unsigned int    cur;            /* التردد الحالي بالـ kHz */
    struct cpufreq_frequency_table *freq_table; /* الترددات المتاحة من الـ driver */
    struct kobject  kobj;           /* sysfs node */
    struct cpufreq_stats *stats;    /* <-- pointer للـ stats struct */
    /* ... */
};
```

```
cpufreq_policy
+-------------------------+
| cpus: {cpu0, cpu4}      |
| cur: 3600000 kHz        |
| freq_table: -----+      |
| kobj: (sysfs)    |      |      cpufreq_frequency_table[]
| stats: ----------+---+  | --> +----------+----------+
+------------------+   |  |    | 3600000  | 3400000  | ...
                    |  |  |    +----------+----------+
                    |  +--+
                    v
              cpufreq_stats
              +---------------+
              | total_trans   |
              | time_in_state |
              | trans_table   |
              +---------------+
```

---

### إزاي بيتسجل الـ Transition — الـ Hot Path

لما الـ governor يقرر يغيّر التردد، الـ cpufreq core بيستدعي `cpufreq_stats_record_transition()`:

```c
void cpufreq_stats_record_transition(struct cpufreq_policy *policy,
                                     unsigned int new_freq)
{
    struct cpufreq_stats *stats = policy->stats;
    int old_index, new_index;

    if (unlikely(!stats))
        return;

    /* لو في reset منتظر، نعمله دلوقتي قبل أي تسجيل جديد */
    if (unlikely(READ_ONCE(stats->reset_pending)))
        cpufreq_stats_reset_table(stats);

    old_index = stats->last_index;
    new_index = freq_table_get_index(stats, new_freq);

    /* skip لو نفس التردد أو index مش معروف */
    if (unlikely(old_index == -1 || new_index == -1 || old_index == new_index))
        return;

    /* نضيف الوقت اللي قضاه على التردد القديم */
    cpufreq_stats_update(stats, stats->last_time);

    /* نحدّث الـ state */
    stats->last_index = new_index;
    stats->trans_table[old_index * stats->max_state + new_index]++;
    stats->total_trans++;
}
```

**مثال عملي — تتبع الـ transitions:**

```
Initial state: cur = 3600000 kHz, last_index = 0, last_time = T0

Event 1: transition to 2800000 kHz at T1
  - time_in_state[0] += T1 - T0   (وقت على 3600MHz)
  - trans_table[0 * 5 + 4]++      (0->4, يعني 3600->2800)
  - total_trans = 1
  - last_index = 4, last_time = T1

Event 2: transition to 3400000 kHz at T2
  - time_in_state[4] += T2 - T1   (وقت على 2800MHz)
  - trans_table[4 * 5 + 1]++      (4->1, يعني 2800->3400)
  - total_trans = 2
  - last_index = 1, last_time = T2
```

---

### الـ time_in_state — إزاي بيحسب الوقت الحالي

الـ `time_in_state[i]` مش بيتحدّث في real-time — بيتحدث فقط لما يحصل transition. لما المستخدم يعمل `cat time_in_state`، الـ `show_time_in_state()` بتضيف الـ "live time" لأخر تردد:

```c
/* لو ده التردد الحالي (last_index) */
if (i == stats->last_index)
    time += local_clock() - stats->last_time;
    /* نضيف الوقت من آخر transition لحد دلوقتي */
```

الـ `local_clock()` هي high-resolution monotonic clock — بترجع nanoseconds. الـ output بيتحول لـ `clock_t` units (10ms) عن طريق `nsec_to_clock_t()`.

---

### الـ Deferred Reset — تصميم lock-free

الـ reset مش بيحصل مباشرة لما المستخدم يكتب في `/sys/.../stats/reset`. الـ `store_reset()` بس بيعمل:

```c
WRITE_ONCE(stats->reset_time, local_clock());  /* سجّل توقيت الـ reset */
smp_wmb();                                      /* memory barrier */
WRITE_ONCE(stats->reset_pending, 1);           /* ارفع الـ flag */
```

الـ reset الفعلي بيحصل في `cpufreq_stats_record_transition()` — على الـ CPU اللي بيملك الـ policy. كده بنتجنّب race conditions من غير locks على الـ hot path.

```
User writes to reset sysfs
        |
        v
store_reset():
  reset_time = now
  smp_wmb()
  reset_pending = 1
        |
        | (no actual reset yet)
        |
Next freq transition happens:
        |
        v
cpufreq_stats_record_transition():
  if reset_pending:
      cpufreq_stats_reset_table()  <-- reset يحصل هنا
          memset(time_in_state, 0, ...)
          memset(trans_table, 0, ...)
          total_trans = 0
          last_time = now
          reset_pending = 0
```

---

### الـ trans_table — الـ 2D Matrix في Memory

الـ matrix بتتخزن كـ flat array بالـ row-major order:

```
N = 5 (5 ترددات)
trans_table[i * N + j] = عدد مرات الانتقال من freq[i] إلى freq[j]

Memory layout:
Index:  0  1  2  3  4  | 5  6  7  8  9  | 10 11 12 13 14 | ...
Freq:   3.6->3.6 3.6->3.4 3.6->3.2 ... | 3.4->3.6 3.4->3.4 ...
                                          ^row 1

trans_table[0*5+1] = trans_table[1] = انتقالات 3.6GHz -> 3.4GHz
```

الـ output اللي بنشوفه:

```
   From  :    To
         : 3600000  3400000  3200000  3000000  2800000
  3600000:        0        5        0        0        0
  3400000:        4        0        2        0        0
```

يعني:
- حصل 5 انتقالات من 3.6GHz إلى 3.4GHz
- حصل 4 انتقالات من 3.4GHz إلى 3.6GHz

---

### إيه اللي الـ cpufreq-stats بيملكه vs إيه اللي بيفوّضه

| المسؤولية | المالك |
|-----------|--------|
| تسجيل وقت كل تردد | **cpufreq-stats** |
| عدّ الـ transitions | **cpufreq-stats** |
| الـ sysfs interface للـ stats | **cpufreq-stats** |
| قرار تغيير التردد | **Governor** (schedutil/ondemand/...) |
| الترددات المتاحة (freq_table) | **cpufreq Driver** (intel_pstate/arm-cpufreq) |
| تنفيذ التغيير على الهاردوير | **cpufreq Driver** |
| ربط الـ CPUs بـ policy | **cpufreq Core** |
| استدعاء `record_transition` | **cpufreq Core** |

---

### الـ Sysfs Integration — ربط الـ Attributes

```c
static struct attribute *default_attrs[] = {
    &total_trans.attr,    /* ro */
    &time_in_state.attr,  /* ro */
    &reset.attr,          /* wo */
    &trans_table.attr,    /* ro */
    NULL
};

static const struct attribute_group stats_attr_group = {
    .attrs = default_attrs,
    .name  = "stats"       /* سيظهر كـ subdirectory */
};
```

الـ `sysfs_create_group(&policy->kobj, &stats_attr_group)` بيعمل:
```
/sys/devices/system/cpu/cpu0/cpufreq/
                                     └── stats/
                                              ├── time_in_state  (r--)
                                              ├── total_trans    (r--)
                                              ├── trans_table    (r--)
                                              └── reset          (-w-)
```

الـ `cpufreq_freq_attr_ro()` و`cpufreq_freq_attr_wo()` هي macros بتعمل `struct freq_attr` بالـ show/store functions المناسبة.

---

### الـ Lifecycle — من init لـ free

```
cpufreq_policy_alloc()
        |
        v
cpufreq_stats_create_table(policy)
  - count valid freq entries
  - kzalloc(struct cpufreq_stats)
  - kzalloc(time_in_state + freq_table + trans_table)
  - fill freq_table from policy->freq_table
  - set last_time = local_clock()
  - set last_index = current freq index
  - policy->stats = stats
  - sysfs_create_group(...)
        |
        | [system running — transitions being recorded]
        |
        v
cpufreq_stats_free_table(policy)
  - sysfs_remove_group(...)
  - kfree(stats->time_in_state)  /* يحرر كل الـ arrays */
  - kfree(stats)
  - policy->stats = NULL
```

---

### مثال كامل — قراءة وتفسير

على ARM board مع 5 ترددات (2.8GHz لـ 3.6GHz):

```bash
$ cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state
3600000 2089     # قضى 20.89 ثانية على 3.6GHz (2089 * 10ms)
3400000 136      # 1.36 ثانية على 3.4GHz
3200000 34       # 0.34 ثانية على 3.2GHz
3000000 67       # 0.67 ثانية على 3.0GHz
2800000 172488   # قضى 1724.88 ثانية على 2.8GHz — أغلب الوقت idle

$ cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans
20               # 20 تغيير تردد بس!

$ cat /sys/devices/system/cpu/cpu0/cpufreq/stats/trans_table
From  :    To
      : 3600000 3400000 3200000 3000000 2800000
3600000:       0       5       0       0       0  # من 3.6 راح 3.4 خمس مرات
3400000:       4       0       2       0       0  # من 3.4 راح 3.6 أربع مرات
3200000:       0       1       0       2       0
3000000:       0       0       1       0       3
2800000:       0       0       0       2       0
```

**القراءة:** الـ CPU بيقضي 99%+ من وقته على أقل تردد (2.8GHz) — الـ system mostly idle. لما يحتاج performance، الـ governor بيرفعه على 3.6GHz مباشرة مش تدريجياً.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags, Enums & Config Options — Cheatsheet

#### Config Options

| Option | القيمة | الوظيفة |
|---|---|---|
| `CONFIG_CPU_FREQ` | bool | يفعّل الـ cpufreq subsystem بالكامل — شرط أساسي |
| `CONFIG_CPU_FREQ_STAT` | bool | يفعّل driver الإحصائيات — بدونه الـ stats stubs فارغة |

#### الـ `enum cpufreq_table_sorting`

| القيمة | المعنى |
|---|---|
| `CPUFREQ_TABLE_UNSORTED` | الجدول مش مرتّب — الـ stats تتجنّب duplicates يدويًا |
| `CPUFREQ_TABLE_SORTED_ASCENDING` | مرتّب تصاعديًا |
| `CPUFREQ_TABLE_SORTED_DESCENDING` | مرتّب تنازليًا |

> الـ `cpufreq_stats_create_table()` بتستخدم القيمة دي عشان تتجنّب تسجيل نفس الـ frequency أكتر من مرة في الـ `freq_table`.

#### Driver Flags المهمة (في `cpufreq_driver.flags`)

| Flag | Bit | الأثر على الـ stats |
|---|---|---|
| `CPUFREQ_ASYNC_NOTIFICATION` | 4 | الـ driver بيبعت notifications برّه `->target()` — الـ stats لازم تتعامل مع transitions غير متزامنة |
| `CPUFREQ_NEED_UPDATE_LIMITS` | 0 | core بيستدعي الـ driver حتى لو الـ freq ماتغيّرش |
| `CPUFREQ_HAVE_GOVERNOR_PER_POLICY` | 3 | كل policy ليها governor منفصل |

#### Sysfs Attributes ومستوياتها

| الملف | الصلاحية | الـ Macro |
|---|---|---|
| `time_in_state` | 0444 (ro) | `cpufreq_freq_attr_ro` |
| `total_trans` | 0444 (ro) | `cpufreq_freq_attr_ro` |
| `trans_table` | 0444 (ro) | `cpufreq_freq_attr_ro` |
| `reset` | 0200 (wo) | `cpufreq_freq_attr_wo` |

---

### الـ Structs المهمة

#### 1. `struct cpufreq_stats`

الـ struct الرئيسي اللي بيخزّن كل إحصائيات الـ CPU لـ policy واحدة. موجود في `drivers/cpufreq/cpufreq_stats.c`.

```c
struct cpufreq_stats {
    unsigned int total_trans;        /* عدد الانتقالات الكلي */
    unsigned long long last_time;    /* وقت آخر transition بالـ ns */
    unsigned int max_state;          /* عدد الـ freq entries في الذاكرة المخصصة */
    unsigned int state_num;          /* عدد الـ unique valid frequencies */
    unsigned int last_index;         /* index الـ frequency الحالية */
    u64 *time_in_state;              /* مصفوفة: وقت في كل frequency (ns) */
    unsigned int *freq_table;        /* مصفوفة: قيم الـ frequencies (kHz) */
    unsigned int *trans_table;       /* مصفوفة 2D: عدد الانتقالات بين كل freq */

    /* Deferred reset */
    unsigned int reset_pending;      /* flag: في reset معلّق؟ */
    unsigned long long reset_time;   /* وقت طلب الـ reset بالـ ns */
};
```

**الحقول بالتفصيل:**

| الحقل | النوع | الوظيفة |
|---|---|---|
| `total_trans` | `unsigned int` | counter إجمالي لكل transition حصلت |
| `last_time` | `unsigned long long` | timestamp من `local_clock()` لحظة آخر update |
| `max_state` | `unsigned int` | حجم الـ allocation — عدد entries الـ freq table |
| `state_num` | `unsigned int` | الـ frequencies الـ unique الفعلية (≤ `max_state`) |
| `last_index` | `unsigned int` | index الـ frequency الشغّالة دلوقتي في `freq_table[]` |
| `time_in_state` | `u64 *` | `time_in_state[i]` = nanoseconds أُمضت في `freq_table[i]` |
| `freq_table` | `unsigned int *` | قيم الـ frequencies بالـ kHz — copied من الـ policy |
| `trans_table` | `unsigned int *` | `trans_table[i * max_state + j]` = transitions من freq[i] لـ freq[j] |
| `reset_pending` | `unsigned int` | يُقرأ بـ `READ_ONCE` — atomic flag للـ deferred reset |
| `reset_time` | `unsigned long long` | وقت طلب الـ reset — يُقرأ بـ `READ_ONCE` بعد `smp_rmb()` |

**ملاحظة مهمة على الـ memory layout:**
الـ `time_in_state` و`freq_table` و`trans_table` بيتخصصوا في **كتلة واحدة متصلة** من الذاكرة:

```
[ time_in_state: count × u64 ][ freq_table: count × u32 ][ trans_table: count² × u32 ]
       ↑
   stats->time_in_state = kzalloc(alloc_size)
   stats->freq_table    = (unsigned int *)(stats->time_in_state + count)
   stats->trans_table   = stats->freq_table + count
```

---

#### 2. `struct cpufreq_policy`

موجود في `include/linux/cpufreq.h`. ده الـ struct الأب — بيمثّل سياسة الـ cpufreq لمجموعة من الـ CPUs المشتركة في نفس الـ clock domain.

| الحقل المرتبط بالـ stats | النوع | الوظيفة |
|---|---|---|
| `stats` | `struct cpufreq_stats *` | pointer للإحصائيات — `NULL` لو ماتخصصتش |
| `cur` | `unsigned int` | الـ frequency الحالية بالـ kHz |
| `freq_table` | `struct cpufreq_frequency_table *` | جدول الـ frequencies المدعومة من الـ driver |
| `freq_table_sorted` | `enum cpufreq_table_sorting` | ترتيب الجدول |
| `kobj` | `struct kobject` | الـ sysfs object — بيتعلق عليه الـ stats group |
| `rwsem` | `struct rw_semaphore` | الـ lock الرئيسي للـ policy |

---

#### 3. `struct freq_attr`

موجود في `include/linux/cpufreq.h`. بيعرّف sysfs attribute خاص بالـ cpufreq.

```c
struct freq_attr {
    struct attribute attr;                                    /* اسم + permissions */
    ssize_t (*show)(struct cpufreq_policy *, char *);         /* read handler */
    ssize_t (*store)(struct cpufreq_policy *, const char *,   /* write handler */
                     size_t count);
};
```

الـ stats بتستخدم 4 instances منه: `total_trans`، `time_in_state`، `reset`، `trans_table`.

---

#### 4. `struct attribute_group` (stats_attr_group)

```c
static const struct attribute_group stats_attr_group = {
    .attrs = default_attrs,   /* مصفوفة الـ attributes الأربعة */
    .name  = "stats"          /* اسم الـ subdirectory في sysfs */
};
```

ده بيخلي الـ attributes تظهر في `/sys/.../cpuX/cpufreq/stats/`.

---

### Struct Relationship Diagram

```
struct cpufreq_policy
┌─────────────────────────────────────────────────────────┐
│  cpu           : unsigned int                           │
│  cur           : unsigned int  (current freq kHz)       │
│  freq_table    ──────────────────────────────────────┐  │
│  freq_table_sorted                                   │  │
│  kobj          ──────────────────────┐               │  │
│  rwsem                               │               │  │
│  stats ─────────────────────┐        │               │  │
└─────────────────────────────│────────│───────────────│──┘
                              │        │               │
                              ▼        │               ▼
              struct cpufreq_stats     │   struct cpufreq_frequency_table[]
              ┌────────────────────┐   │   ┌──────────────────────────────┐
              │ total_trans        │   │   │ frequency: unsigned int       │
              │ last_time          │   │   │ driver_data: unsigned int     │
              │ max_state          │   │   │ flags: unsigned int           │
              │ state_num          │   │   └──────────────────────────────┘
              │ last_index         │   │
              │                    │   │
              │ time_in_state[] ───┼───┘ (copied at create time)
              │ freq_table[]    ───┘
              │ trans_table[]      │
              │                    │
              │ reset_pending      │
              │ reset_time         │
              └────────────────────┘
                              │
                              │ (sysfs_create_group)
                              ▼
              struct attribute_group (stats_attr_group)
              ┌────────────────────────────────────────┐
              │ name: "stats"                          │
              │ attrs[]:                               │
              │   ├── total_trans  (ro, freq_attr)     │
              │   ├── time_in_state (ro, freq_attr)    │
              │   ├── trans_table  (ro, freq_attr)     │
              │   └── reset        (wo, freq_attr)     │
              └────────────────────────────────────────┘
                              │
                              ▼
              /sys/devices/system/cpu/cpuX/cpufreq/stats/
```

---

### Lifecycle Diagram

```
╔══════════════════════════════════════════════════════════════════════╗
║                    cpufreq_stats LIFECYCLE                          ║
╚══════════════════════════════════════════════════════════════════════╝

  [CPU/Policy Init]
        │
        ▼
  cpufreq_stats_create_table(policy)
  ┌─────────────────────────────────────────────────────┐
  │  1. count = cpufreq_table_count_valid_entries()     │
  │  2. kzalloc(sizeof(cpufreq_stats))                  │
  │  3. kzalloc(count×u64 + count×u32 + count²×u32)    │
  │     → time_in_state, freq_table, trans_table        │
  │  4. copy unique freqs from policy->freq_table       │
  │  5. last_time  = local_clock()                      │
  │  6. last_index = freq_table_get_index(policy->cur)  │
  │  7. policy->stats = stats                           │
  │  8. sysfs_create_group(&policy->kobj, &stats_attr)  │
  └─────────────────────────────────────────────────────┘
        │
        ▼
  [ACTIVE — CPU running, governor changing freq]
        │
        ├──── [read sysfs] ──────────────────────────────────────────┐
        │     show_time_in_state() / show_total_trans()              │
        │     show_trans_table()                                     │
        │     (reads stats fields, accounts for live elapsed time)   │
        │◄───────────────────────────────────────────────────────────┘
        │
        ├──── [write sysfs reset] ───────────────────────────────────┐
        │     store_reset()                                          │
        │     → WRITE_ONCE(reset_time, local_clock())                │
        │     → smp_wmb()                                            │
        │     → WRITE_ONCE(reset_pending, 1)                         │
        │     (actual clear happens in next transition)              │
        │◄───────────────────────────────────────────────────────────┘
        │
        ├──── [freq transition] ─────────────────────────────────────┐
        │     cpufreq_stats_record_transition(policy, new_freq)      │
        │     → if reset_pending: cpufreq_stats_reset_table()        │
        │     → cpufreq_stats_update(stats, stats->last_time)        │
        │     → trans_table[old×max + new]++                         │
        │     → total_trans++                                        │
        │     → last_index = new_index                               │
        │◄───────────────────────────────────────────────────────────┘
        │
        ▼
  [CPU Offline / Policy Destroy]
        │
        ▼
  cpufreq_stats_free_table(policy)
  ┌─────────────────────────────────────────────────────┐
  │  1. sysfs_remove_group(&policy->kobj, &stats_attr)  │
  │  2. kfree(stats->time_in_state)  (frees whole block) │
  │  3. kfree(stats)                                    │
  │  4. policy->stats = NULL                            │
  └─────────────────────────────────────────────────────┘
```

---

### Call Flow Diagrams

#### تسجيل Transition (المسار الأساسي)

```
governor / cpufreq core
  │
  │  calls target_index() or fast_switch()
  ▼
cpufreq core (cpufreq.c)
  │
  │  __cpufreq_driver_target() → notifies POSTCHANGE
  │  → cpufreq_notify_transition()
  │      → cpufreq_stats_record_transition(policy, new_freq)
  ▼
cpufreq_stats_record_transition(policy, new_freq)
  │
  ├─ [stats == NULL?] → return immediately
  │
  ├─ [reset_pending?]
  │     └─ cpufreq_stats_reset_table(stats)
  │           ├─ memset time_in_state → 0
  │           ├─ memset trans_table   → 0
  │           ├─ total_trans = 0
  │           ├─ WRITE_ONCE(reset_pending, 0)
  │           ├─ smp_rmb()
  │           └─ cpufreq_stats_update(stats, READ_ONCE(reset_time))
  │                 └─ time_in_state[last_index] += cur - reset_time
  │                    last_time = cur_time
  │
  ├─ old_index = stats->last_index
  ├─ new_index = freq_table_get_index(stats, new_freq)
  │
  ├─ [old == -1 || new == -1 || old == new?] → return
  │
  ├─ cpufreq_stats_update(stats, stats->last_time)
  │       └─ time_in_state[last_index] += local_clock() - last_time
  │          last_time = local_clock()
  │
  ├─ last_index = new_index
  ├─ trans_table[old * max_state + new]++
  └─ total_trans++
```

#### قراءة `time_in_state` من sysfs

```
user: cat /sys/…/cpu0/cpufreq/stats/time_in_state
  │
  ▼
VFS → sysfs_read_file()
  │
  ▼
show_time_in_state(policy, buf)
  │
  ├─ pending = READ_ONCE(stats->reset_pending)
  │
  ├─ for i in 0..state_num:
  │     if pending:
  │       if i == last_index:
  │         smp_rmb()
  │         time = local_clock() - READ_ONCE(reset_time)
  │       else:
  │         time = 0
  │     else:
  │       time = time_in_state[i]
  │       if i == last_index:
  │         time += local_clock() - last_time   ← live elapsed
  │
  └─ sprintf "%u %llu\n", freq_table[i], nsec_to_clock_t(time)
```

#### الـ Reset Flow (Deferred)

```
user: echo 1 > reset
  │
  ▼
store_reset(policy, buf, count)          ← context: sysfs write (process context)
  ├─ WRITE_ONCE(reset_time, local_clock())
  ├─ smp_wmb()                            ← memory barrier: reset_time visible before flag
  └─ WRITE_ONCE(reset_pending, 1)
              │
              │  (لا reset فعلي هنا — بيتأجّل)
              │
              ▼
  [next freq transition in cpufreq_stats_record_transition()]
              │
              ▼
  cpufreq_stats_reset_table(stats)        ← context: transition (process/softirq)
  ├─ memset all counters to 0
  ├─ WRITE_ONCE(reset_pending, 0)
  ├─ smp_rmb()
  └─ cpufreq_stats_update(stats, reset_time)
       └─ يحسب الوقت اللي فات من لحظة الـ reset للـ frequency الحالية
```

---

### Locking Strategy

#### الـ Locks المستخدمة

| الـ Lock | النوع | مكان التعريف | بيحمي إيه |
|---|---|---|---|
| `policy->rwsem` | `struct rw_semaphore` | `cpufreq_policy` | كل حقول الـ policy بما فيها `policy->stats` |
| `stats->reset_pending` | atomic via `READ/WRITE_ONCE` | `cpufreq_stats` | الـ deferred reset flag |
| `stats->reset_time` | atomic via `READ/WRITE_ONCE` | `cpufreq_stats` | timestamp الـ reset |

#### لماذا لا يوجد explicit lock على `cpufreq_stats`؟

الـ `cpufreq_stats_record_transition()` بيتنادى من سياق الـ transition اللي ممسك بالـ policy lock بالفعل. الـ sysfs reads بتحصل في process context وبتاخد `down_read(&policy->rwsem)` تلقائيًا عن طريق الـ cpufreq sysfs framework.

الـ reset path استخدم تقنية **lock-free deferred reset**:

```
   Writer (store_reset):                 Reader (record_transition):
   ══════════════════════                ═══════════════════════════
   WRITE_ONCE(reset_time, T)             READ_ONCE(reset_pending)  ← sees 1
   smp_wmb()                 ────────►   smp_rmb()
   WRITE_ONCE(reset_pending, 1)          READ_ONCE(reset_time)     ← sees T
```

الـ `smp_wmb()` في الـ writer بيضمن إن `reset_time` يتكتب قبل `reset_pending`. الـ `smp_rmb()` في الـ reader بيضمن إنه مش هيقرأ `reset_time` القديمة.

#### Lock Ordering

```
policy->rwsem (write)        ← CPU hotplug, policy destroy
    └── policy->rwsem (read) ← sysfs show/store, governor changes
            └── (no inner lock on cpufreq_stats — lockless via READ/WRITE_ONCE + barriers)
```

#### متى بيحصل race وإزاي بيتحل؟

السيناريو: `store_reset()` بيتنفّذ في نفس وقت `show_time_in_state()`.

- الـ `show_*` بتقرأ `reset_pending` أوّل حاجة.
- لو شافت `1`، بتستخدم `reset_time` بدل `time_in_state[]` المخزّن — فالمستخدم بياخد أرقام صح من لحظة الـ reset.
- لو شافت `0`، بتقرأ الـ counters العادية.
- الـ actual wipe بيتأجّل للـ transition — وده بيضمن إن لا writer ولا reader بيعمل `memset` في نفس وقت القراءة.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Public API (exported من `cpufreq_stats.c`)

| Function | Caller Context | Purpose |
|---|---|---|
| `cpufreq_stats_create_table()` | `cpufreq_add_policy_cpu()` — policy init | تنشئ الـ stats struct وتسجّل الـ sysfs group |
| `cpufreq_stats_free_table()` | `cpufreq_remove_dev()` — policy teardown | تحذف الـ stats وتمسح الـ sysfs entries |
| `cpufreq_stats_record_transition()` | `cpufreq_notify_transition()` — frequency switch | تسجّل كل تحوّل في التردد |

#### Internal / Static Functions

| Function | Purpose |
|---|---|
| `cpufreq_stats_update()` | تضيف الوقت المنقضي للـ `time_in_state[last_index]` |
| `cpufreq_stats_reset_table()` | تصفّر كل الإحصاءات فعلياً (deferred reset) |
| `freq_table_get_index()` | تترجم قيمة تردد (kHz) إلى index في `freq_table[]` |

#### Sysfs Show/Store Handlers

| Handler | File | Mode |
|---|---|---|
| `show_total_trans()` | `stats/total_trans` | read-only (0444) |
| `show_time_in_state()` | `stats/time_in_state` | read-only (0444) |
| `show_trans_table()` | `stats/trans_table` | read-only (0444) |
| `store_reset()` | `stats/reset` | write-only (0200) |

---

### المجموعة الأولى: Lifecycle — إنشاء وتدمير الـ Stats

هاتان الدالتان هما نقطة دخول وخروج الـ subsystem كله. بتُستدعى من `cpufreq core` عند رفع أو إزالة الـ policy.

---

#### `cpufreq_stats_create_table`

```c
void cpufreq_stats_create_table(struct cpufreq_policy *policy)
```

الدالة دي بتبني الـ `cpufreq_stats` struct من الصفر وبتوصلها بالـ policy وبتسجّل الـ sysfs attributes. بتحسب عدد الـ valid frequencies في الـ frequency table وبتعمل allocation واحد كبير يكفي الـ `time_in_state[]` و`freq_table[]` و`trans_table[]` كلهم متجاورين في الذاكرة.

**Parameters:**
- `policy` — الـ `cpufreq_policy` اللي هتتصل بيها الـ stats.

**Return value:** `void`. عند أي فشل (alloc أو sysfs) بيتم cleanup صامت.

**Key details:**
- بتتحقق الأول إن `policy->stats == NULL` — لو موجود بالفعل بترجع فوراً (idempotent).
- الـ allocation layout في الذاكرة:
  ```
  [time_in_state: count × u64] [freq_table: count × uint] [trans_table: count² × uint]
  ```
  كل ده في `kzalloc` واحد بيبدأ من `stats->time_in_state` — يعني `kfree` واحد بيكفي.
- `stats->last_index` بيتضبط على التردد الحالي (`policy->cur`) عن طريق `freq_table_get_index()`.
- `stats->last_time = local_clock()` — timestamp بالـ nanoseconds من آخر تحديث.
- لو `sysfs_create_group()` فشل، بيعمل cleanup ويضبط `policy->stats = NULL`.
- مفيش locking داخلي — الـ cpufreq core بيحمي الـ policy بالـ `rwsem`.

**Pseudocode:**
```
count = cpufreq_table_count_valid_entries(policy)
if count == 0 || policy->stats != NULL: return

stats = kzalloc(sizeof(*stats))
stats->time_in_state = kzalloc(count*(sizeof(u64)+sizeof(int)) + count²*sizeof(int))
stats->freq_table     = (uint*)(stats->time_in_state + count)
stats->trans_table    = stats->freq_table + count

fill stats->freq_table[] from policy->freq_table (skip duplicates if UNSORTED)
stats->state_num  = valid unique freqs count
stats->max_state  = count
stats->last_index = freq_table_get_index(stats, policy->cur)
stats->last_time  = local_clock()

policy->stats = stats
sysfs_create_group(&policy->kobj, &stats_attr_group)
```

---

#### `cpufreq_stats_free_table`

```c
void cpufreq_stats_free_table(struct cpufreq_policy *policy)
```

بتحذف كل موارد الـ stats المرتبطة بالـ policy — الـ sysfs group وكل الـ memory.

**Parameters:**
- `policy` — الـ policy المراد تنظيفها.

**Return value:** `void`.

**Key details:**
- بتتحقق إن `policy->stats != NULL` — لو NULL بترجع فوراً (safe to call multiple times).
- بتعمل `sysfs_remove_group()` الأول عشان تشيل الـ attributes من الـ sysfs قبل ما تحرر الذاكرة.
- `kfree(stats->time_in_state)` بيحرر الـ block الكبير اللي فيه `time_in_state + freq_table + trans_table` كلهم.
- `kfree(stats)` بيحرر الـ struct نفسه.
- بعدين `policy->stats = NULL` لمنع double-free.
- بيُستدعى من `cpufreq_remove_dev()` في `cpufreq.c` عند hotplug أو driver unload.

---

### المجموعة التانية: Runtime — تسجيل التحوّلات

---

#### `cpufreq_stats_record_transition`

```c
void cpufreq_stats_record_transition(struct cpufreq_policy *policy,
                                     unsigned int new_freq)
```

ده القلب النابض للـ subsystem. بيُستدعى في كل مرة بيتغير فيها التردد فعلياً. بيسجّل الوقت اللي CPU قضاه في التردد القديم، وبيحدّث الـ transition matrix.

**Parameters:**
- `policy` — الـ policy اللي بيحصل فيها التحوّل.
- `new_freq` — التردد الجديد بالـ kHz.

**Return value:** `void`.

**Key details:**
- `unlikely(!stats)` — guard لو الـ stats مش initialized.
- **Deferred Reset:** لو `reset_pending == 1`، بيستدعي `cpufreq_stats_reset_table()` قبل أي حاجة — ده التصميم الذكي اللي بيتجنب race conditions بين `store_reset()` اللي بيجي من سياق مختلف والـ transition path.
- لو `old_index == new_index` (same freq) أو أي منهم `-1` (freq مش موجود في الجدول) — بيرجع فوراً.
- بيستدعي `cpufreq_stats_update()` لتسجيل الوقت على `old_index`، ثم يحدّث:
  - `stats->last_index = new_index`
  - `stats->trans_table[old_index * max_state + new_index]++`
  - `stats->total_trans++`
- لا يوجد locking هنا — الـ cpufreq core guarantees serialization للـ transitions.

**Pseudocode:**
```
if !stats: return
if reset_pending: cpufreq_stats_reset_table(stats)

old_index = stats->last_index
new_index = freq_table_get_index(stats, new_freq)

if old_index == -1 || new_index == -1 || old_index == new_index: return

cpufreq_stats_update(stats, stats->last_time)   // accrue time on old freq
stats->last_index = new_index
stats->trans_table[old_index * max_state + new_index]++
stats->total_trans++
```

---

### المجموعة التالتة: Internal Helpers

---

#### `cpufreq_stats_update`

```c
static void cpufreq_stats_update(struct cpufreq_stats *stats,
                                 unsigned long long time)
```

بتحسب الفرق بين الوقت الحالي (`local_clock()`) والـ `time` اللي اتبعتلها، وبتضيفه على `time_in_state[last_index]`.

**Parameters:**
- `stats` — الـ stats struct.
- `time` — timestamp بالـ nanoseconds من آخر نقطة تحديث.

**Return value:** `void`.

**Key details:**
- `local_clock()` بيرجع الوقت بالـ nanoseconds من الـ CPU المحلي — سريع وبدون context switch.
- بتحدّث `stats->last_time` بالوقت الحالي بعد الحساب.
- بتُستدعى من `cpufreq_stats_record_transition()` و`cpufreq_stats_reset_table()`.

---

#### `cpufreq_stats_reset_table`

```c
static void cpufreq_stats_reset_table(struct cpufreq_stats *stats)
```

بتنفذ الـ reset الفعلي اللي اتطلبه `store_reset()` بشكل deferred. بتصفّر كل الإحصاءات وبتحسب الوقت اللي انقضى منذ طلب الـ reset.

**Parameters:**
- `stats` — الـ stats struct.

**Return value:** `void`.

**Key details:**
- `memset(time_in_state, 0, count * sizeof(u64))` — تصفير الـ time counters.
- `memset(trans_table, 0, count * count * sizeof(int))` — تصفير الـ transition matrix.
- `total_trans = 0`.
- `WRITE_ONCE(stats->reset_pending, 0)` — يعلن انتهاء الـ reset.
- `smp_rmb()` بعد الـ `WRITE_ONCE` — يضمن إن قراءة `reset_time` مش بتتعمل قبل clear الـ `reset_pending` في الـ memory order.
- بعدين بيستدعي `cpufreq_stats_update(stats, READ_ONCE(stats->reset_time))` عشان يضيف الوقت اللي CPU قضاه في التردد الحالي منذ لحظة الـ reset request — ده بيضمن دقة `time_in_state` للتردد الحالي.

**Deferred Reset Design:**

```
store_reset():                     cpufreq_stats_record_transition():
  WRITE_ONCE(reset_time, now)         if reset_pending:
  smp_wmb()                              cpufreq_stats_reset_table()
  WRITE_ONCE(reset_pending, 1)              // actual clear happens here
```

الـ `smp_wmb()` في `store_reset()` يضمن إن `reset_time` مكتوب قبل `reset_pending = 1`.
الـ `smp_rmb()` في `reset_table()` يضمن إن `reset_time` يُقرأ بعد ما `reset_pending` يُصفّر.

---

#### `freq_table_get_index`

```c
static int freq_table_get_index(struct cpufreq_stats *stats, unsigned int freq)
```

بتعمل linear search في `stats->freq_table[]` للإيجاد الـ index المقابل لقيمة التردد.

**Parameters:**
- `stats` — الـ stats struct.
- `freq` — التردد المطلوب بالـ kHz.

**Return value:** الـ index (0-based) لو وُجد، أو `-1` لو مش موجود.

**Key details:**
- بتكرر من 0 إلى `max_state - 1`.
- الـ freq table صغيرة في الغالب (أقل من 20 entry لأغلب الـ platforms) — linear scan مقبول هنا.
- الـ return value بيتحقق منه `cpufreq_stats_record_transition()` بالـ `unlikely(... == -1)` guard.

---

### المجموعة الرابعة: Sysfs Show Handlers

---

#### `show_total_trans`

```c
static ssize_t show_total_trans(struct cpufreq_policy *policy, char *buf)
```

بتكتب في الـ buffer عدد كل التحوّلات اللي حصلت في هذا الـ CPU.

**Parameters:**
- `policy` — الـ policy owner للـ CPU.
- `buf` — sysfs page buffer.

**Return value:** عدد الـ bytes المكتوبة.

**Key details:**
- لو `reset_pending == 1` بترجع `"0\n"` مباشرة — يعني من وجهة نظر المستخدم الـ reset حصل فوراً حتى لو الـ deferred reset لسه ما انتهاش.
- السبب: الـ reset الفعلي بيحصل في سياق الـ transition، ولو مفيش transitions هتحصل، المستخدم لازم يشوف صفر مش القيمة القديمة.

---

#### `show_time_in_state`

```c
static ssize_t show_time_in_state(struct cpufreq_policy *policy, char *buf)
```

بتكتب سطر لكل تردد مدعوم بالشكل: `<freq_kHz> <time_jiffies>\n`.

**Parameters:**
- `policy` — الـ policy.
- `buf` — sysfs page buffer.

**Return value:** إجمالي الـ bytes المكتوبة.

**Key details:**
- `nsec_to_clock_t(time)` بيحوّل الـ nanoseconds إلى وحدات USER_HZ (10ms per unit) — متوافق مع `/proc` conventions.
- **للـ last_index فقط:** بيضيف الوقت المنقضي منذ آخر تحديث (`local_clock() - last_time`) عشان القيمة تكون live وليست stale.
- لو `reset_pending`:
  - كل الترددات = 0 ما عدا `last_index` (التردد الحالي) = `local_clock() - reset_time`.
  - `smp_rmb()` قبل قراءة `reset_time` لضمان الـ ordering.

---

#### `show_trans_table`

```c
static ssize_t show_trans_table(struct cpufreq_policy *policy, char *buf)
```

بتطبع الـ 2D transition matrix كـ human-readable table.

**Parameters:**
- `policy` — الـ policy.
- `buf` — sysfs page buffer (max `PAGE_SIZE`).

**Return value:** عدد الـ bytes، أو `-EFBIG` لو الجدول أكبر من `PAGE_SIZE`.

**Key details:**
- بتستخدم `sysfs_emit_at(buf, len, ...)` لضمان عدم الـ overflow — أآمن من `sprintf`.
- بتتحقق من `len >= PAGE_SIZE - 1` بعد كل write — لو حصل overflow بتطبع warning مرة واحدة (`pr_warn_once`) وبترجع `-EFBIG`.
- لو `reset_pending`، كل قيم الجدول = 0.
- الـ matrix مخزّنة كـ flat array: `trans_table[i * max_state + j]`.

---

#### `store_reset`

```c
static ssize_t store_reset(struct cpufreq_policy *policy, const char *buf,
                           size_t count)
```

بتعمل reset للإحصاءات بشكل deferred — بتسجّل وقت الـ reset وبترفع الـ `reset_pending` flag.

**Parameters:**
- `policy` — الـ policy.
- `buf` — محتوى الـ write (مُتجاهَل تماماً — أي كتابة بتعمل reset).
- `count` — عدد الـ bytes المكتوبة.

**Return value:** `count` دائماً — أي write ناجح.

**Key details:**
- `WRITE_ONCE(stats->reset_time, local_clock())` — يسجّل لحظة طلب الـ reset.
- `smp_wmb()` — يضمن إن `reset_time` مرئي للـ readers قبل ما يشوفوا `reset_pending = 1`.
- `WRITE_ONCE(stats->reset_pending, 1)` — يرفع الـ flag.
- الـ write handler مكتوب بـ `cpufreq_freq_attr_wo(reset)` ما يعني permissions = 0200 (write-only for root).

---

### المجموعة الخامسة: Sysfs Attribute Registration Macros

---

#### `cpufreq_freq_attr_ro` / `cpufreq_freq_attr_wo`

```c
#define cpufreq_freq_attr_ro(_name)   \
    static struct freq_attr _name =   \
    __ATTR(_name, 0444, show_##_name, NULL)

#define cpufreq_freq_attr_wo(_name)   \
    static struct freq_attr _name =   \
    __ATTR(_name, 0200, NULL, store_##_name)
```

ماكروهات بتعرّف الـ `freq_attr` structs اللي بتربط اسم الـ attribute بالـ show/store handlers.

**Key details:**
- `cpufreq_freq_attr_ro(total_trans)` → ينشئ `struct freq_attr total_trans` بالـ show handler.
- `cpufreq_freq_attr_ro(time_in_state)` → ينشئ `struct freq_attr time_in_state`.
- `cpufreq_freq_attr_ro(trans_table)` → ينشئ `struct freq_attr trans_table`.
- `cpufreq_freq_attr_wo(reset)` → ينشئ `struct freq_attr reset` بالـ store handler فقط.
- كلهم بيتجمعوا في `default_attrs[]` اللي بيتحدد في `stats_attr_group`.

---

### نظرة شاملة — Call Graph

```
cpufreq_add_policy_cpu()               [cpufreq core]
    └─► cpufreq_stats_create_table()
            ├─ kzalloc(stats)
            ├─ kzalloc(time_in_state + freq_table + trans_table)
            ├─ fill freq_table[]
            └─ sysfs_create_group("stats/")

cpufreq_notify_transition()            [cpufreq core — on every freq change]
    └─► cpufreq_stats_record_transition()
            ├─ [if reset_pending] cpufreq_stats_reset_table()
            │       ├─ memset(time_in_state, 0)
            │       ├─ memset(trans_table, 0)
            │       └─ cpufreq_stats_update(reset_time)
            ├─ cpufreq_stats_update(last_time)
            ├─ trans_table[old→new]++
            └─ total_trans++

sysfs read("stats/time_in_state")
    └─► show_time_in_state()
            └─ [per state] nsec_to_clock_t(time_in_state[i] + live_delta)

sysfs write("stats/reset", "1")
    └─► store_reset()
            ├─ WRITE_ONCE(reset_time, local_clock())
            ├─ smp_wmb()
            └─ WRITE_ONCE(reset_pending, 1)

cpufreq_remove_dev()                   [cpufreq core]
    └─► cpufreq_stats_free_table()
            ├─ sysfs_remove_group("stats/")
            ├─ kfree(time_in_state)
            └─ kfree(stats)
```

---

### ملاحظة على الـ Memory Layout

```
stats->time_in_state (kzalloc base ptr)
│
├─ [0 .. count-1]       → u64 time_in_state[count]
├─ [count .. 2*count-1] → unsigned int freq_table[count]
└─ [2*count .. end]     → unsigned int trans_table[count × count]
```

**الـ** `freq_table` **و**`trans_table` **هم مجرد pointers داخل نفس الـ allocation** — تصميم كفء يقلل الـ allocations ويسهّل الـ cleanup.
## Phase 5: دليل الـ Debugging الشامل

### Software Level

#### 1. مدخلات الـ sysfs المتعلقة بالـ cpufreq-stats

الـ **cpufreq-stats** بيعتمد كليًا على الـ sysfs — مفيش debugfs entries خاصة بيه. كل الـ stats موجودة في:

```
/sys/devices/system/cpu/cpuX/cpufreq/stats/
```

| Entry | النوع | الوصف |
|---|---|---|
| `time_in_state` | read-only | الوقت (بوحدة 10ms) اللي اتقضى في كل frequency |
| `total_trans` | read-only | إجمالي عدد الـ frequency transitions |
| `trans_table` | read-only | مصفوفة 2D بعدد الانتقالات من freq_i إلى freq_j |
| `reset` | write-only | بيعمل reset للـ counters بدون reboot |

```bash
# اقرأ الـ stats لكل CPU
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/stats; do
    echo "=== $cpu ==="; cat "$cpu/time_in_state"; done

# اقرأ total transitions لكل CPU
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/stats/total_trans; do
    echo "$cpu: $(cat $cpu)"; done

# اقرأ trans_table لـ CPU0
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/trans_table

# عمل reset للـ stats وابدأ من الصفر
echo 1 > /sys/devices/system/cpu/cpu0/cpufreq/stats/reset
```

**مثال على تفسير `time_in_state`:**

```
3600000 2089    ← اشتغل 20.89 ثانية على 3.6 GHz
3400000 136     ← اشتغل 1.36 ثانية على 3.4 GHz
2800000 172488  ← اشتغل ~28 دقيقة على 2.8 GHz (الـ base freq)
```

**مثال على تفسير `trans_table`:**

```
   From  :    To
         :   3600000   3400000   2800000
   3600000:         0         5         0   ← من 3.6GHz → 3.4GHz خمس مرات
   3400000:         4         0         2
   2800000:         0         0         0
```

لو `trans_table` رجع `-EFBIG`، ده معناه إن عدد الـ frequencies كبير جدًا وجدول الـ transitions أكبر من `PAGE_SIZE` — مش bug في الكود، ده by design.

---

#### 2. مدخلات الـ sysfs المهمة في الـ cpufreq العام

```bash
# الـ governor الحالي
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# الـ frequency الحالية
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# الـ frequencies المتاحة
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies

# حالة الـ policy
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq

# هل الـ stats driver موجود؟
ls /sys/devices/system/cpu/cpu0/cpufreq/stats/ 2>/dev/null || echo "cpufreq-stats not loaded"
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

الـ **ftrace** هو أداتك الأساسية لمتابعة الـ transitions في real time.

```bash
# Enable الـ cpufreq tracepoints
echo 1 > /sys/kernel/debug/tracing/events/power/cpu_frequency/enable
echo 1 > /sys/kernel/debug/tracing/events/power/cpu_frequency_limits/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# ابدأ workload وتابع
cat /sys/kernel/debug/tracing/trace_pipe

# الناتج المتوقع:
# kworker/0:2-123 [000] .... 12345.678901: cpu_frequency: state=3600000 cpu_id=0
# kworker/0:2-123 [000] .... 12345.679001: cpu_frequency: state=2800000 cpu_id=0
```

```bash
# trace فقط cpufreq_stats_record_transition بالـ function tracer
echo function > /sys/kernel/debug/tracing/current_tracer
echo cpufreq_stats_record_transition > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

```bash
# استخدام trace-cmd لتبسيط العملية
trace-cmd record -e power:cpu_frequency sleep 10
trace-cmd report | grep cpu_frequency
```

---

#### 4. الـ printk والـ dynamic debug

الـ driver بيستخدم `pr_debug` في `cpufreq_stats_free_table`:

```c
pr_debug("%s: Free stats table\n", __func__);
```

لتفعيل الـ dynamic debug لهذا الـ module:

```bash
# تفعيل كل الـ debug messages في cpufreq_stats
echo "module cpufreq_stats +p" > /sys/kernel/debug/dynamic_debug/control

# أو تفعيل debug خاص بملف معين
echo "file drivers/cpufreq/cpufreq_stats.c +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ active debug entries
grep cpufreq_stats /sys/kernel/debug/dynamic_debug/control

# تعطيل الـ debug messages
echo "module cpufreq_stats -p" > /sys/kernel/debug/dynamic_debug/control
```

```bash
# زيادة الـ loglevel لرؤية pr_debug
dmesg -n 8
# أو بشكل مؤقت أثناء الـ testing
echo 8 > /proc/sys/kernel/printk
```

---

#### 5. الـ Kernel Config options للـ Debugging

| Config | الوصف | متى تفعّله |
|---|---|---|
| `CONFIG_CPU_FREQ` | الـ cpufreq core — شرط أساسي | دايمًا |
| `CONFIG_CPU_FREQ_STAT` | يفعّل الـ stats driver نفسه | دايمًا للـ debugging |
| `CONFIG_CPU_FREQ_DEBUG` | يفعّل debug logging في الـ cpufreq core | عند مشاكل الـ transitions |
| `CONFIG_TRACING` | الـ ftrace infrastructure | للـ tracing |
| `CONFIG_POWER_EVENT_TRACING` | power tracepoints (cpu_frequency events) | لمتابعة الـ transitions |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل الـ pr_debug dynamically | للـ development |
| `CONFIG_DEBUG_KERNEL` | يفعّل كل الـ kernel debug options | في بيئة الـ dev فقط |
| `CONFIG_LOCKDEP` | يكشف مشاكل الـ locking | لو مشتبه في race conditions |
| `CONFIG_PROVE_LOCKING` | يثبت صحة الـ lock ordering | مع CONFIG_LOCKDEP |
| `CONFIG_KASAN` | يكشف memory bugs | لو مشتبه في memory corruption |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CPU_FREQ|CPUFREQ_STAT"

# أو
grep -E "CPU_FREQ|CPUFREQ_STAT" /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ subsystem

**الـ cpupower tool** هو الأداة الرئيسية لإدارة وتشخيص الـ cpufreq:

```bash
# عرض حالة الـ CPU الكاملة مع الـ frequency info
cpupower frequency-info

# عرض الـ stats بشكل مرتب
cpupower monitor

# عرض الـ governors المتاحة
cpupower frequency-info --governors

# ضبط governor للـ testing
cpupower frequency-set -g performance
cpupower frequency-set -g powersave

# تغيير الـ frequency يدويًا (بعد تفعيل userspace governor)
cpupower frequency-set -g userspace
cpupower frequency-set -f 2800000
```

```bash
# استخدام turbostat لمتابعة الـ frequencies في real time
turbostat --show CPU,Avg_MHz,Busy%,Bzy_MHz,TSC_MHz --interval 1

# استخدام perf لتحليل الـ frequency events
perf stat -e power:cpu_frequency -- sleep 5
perf record -e power:cpu_frequency -a -- sleep 10
perf report
```

---

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ Error | المعنى | الحل |
|---|---|---|
| `cpufreq transition table exceeds PAGE_SIZE. Disabling` | الـ trans_table بقت أكبر من 4KB — بيحصل لما عدد الـ frequencies كبير جدًا (مثلاً Intel مع كل الـ P-states) | اقرأ `time_in_state` و`total_trans` بدل `trans_table`؛ المشكلة by design |
| لا يوجد `/sys/.../cpufreq/stats/` | الـ `CONFIG_CPU_FREQ_STAT` مش مفعّل، أو الـ driver مش محمّل | `modprobe cpufreq_stats` أو اعمل compile مع `CONFIG_CPU_FREQ_STAT=y` |
| `cat: /sys/.../trans_table: File too large` | نفس مشكلة الـ PAGE_SIZE | استخدم `total_trans` للحصول على العدد الإجمالي |
| `time_in_state` بيرجع صفر لكل الـ frequencies | الـ stats اتعملت reset مؤخرًا أو الـ driver اتحمّل للتو | انتظر شوية وحاول تاني؛ تحقق من `total_trans` |
| `total_trans` بيرجع 0 بعد الـ reset | ده سلوك طبيعي — الـ reset_pending flag شغّال | صح تمامًا، انتظر أول transition |
| مش في الـ sysfs أي stats بعد `insmod` | الـ `cpufreq_stats_create_table` فشلت (memory allocation) | تحقق من `dmesg` لـ memory errors؛ تحقق من `kzalloc` failures |

---

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

في الـ source code، الأماكن الأكثر أهمية للـ instrumentation هي:

```c
/* في cpufreq_stats_record_transition — المكان الأهم */
void cpufreq_stats_record_transition(struct cpufreq_policy *policy,
                                     unsigned int new_freq)
{
    struct cpufreq_stats *stats = policy->stats;
    int old_index, new_index;

    /* نقطة 1: stats == NULL بعد ما المفروض يكون initialized */
    if (unlikely(!stats)) {
        WARN_ON_ONCE(!stats); /* كشف لو policy->stats اتحذف بشكل غلط */
        return;
    }

    old_index = stats->last_index;
    new_index = freq_table_get_index(stats, new_freq);

    /* نقطة 2: frequency مش موجودة في الـ freq_table */
    if (unlikely(old_index == -1 || new_index == -1 || old_index == new_index)) {
        if (new_index == -1)
            WARN_ONCE(1, "cpufreq: unknown freq %u for cpu%d\n",
                      new_freq, policy->cpu); /* تحذير لـ unknown frequency */
        return;
    }
    /* ... */
}
```

```c
/* في cpufreq_stats_create_table — تحقق من الـ allocation */
void cpufreq_stats_create_table(struct cpufreq_policy *policy)
{
    /* نقطة 3: لو count == 0 ده مشكلة في الـ freq_table */
    count = cpufreq_table_count_valid_entries(policy);
    WARN_ON_ONCE(count == 0); /* freq_table فاضية = مشكلة في الـ driver */
    /* ... */
}
```

لتطبيق الـ WARN_ON كـ debugging مؤقت بدون تعديل الـ kernel:

```bash
# استخدم kprobe عبر الـ ftrace بدل تعديل الـ source
echo 'p:myprobe cpufreq_stats_record_transition policy=%di new_freq=%si' \
    > /sys/kernel/debug/tracing/kprobe_events

echo 1 > /sys/kernel/debug/tracing/events/kprobes/myprobe/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware بتطابق حالة الـ Kernel

الـ kernel بيقرأ الـ frequency الحالية من الـ hardware، لازم تتحقق إن القيمتين متطابقتين:

```bash
# الـ frequency اللي الـ kernel عارفها
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq  # من الـ driver مباشرة
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq  # من الـ governor

# الـ frequency الفعلية من الـ hardware (x86)
# عبر MSR registers
rdmsr -p 0 0x198  # IA32_PERF_STATUS — الـ current P-state
rdmsr -p 0 0x199  # IA32_PERF_CTL — الـ requested P-state

# الـ TSC-based measurement للـ actual frequency
turbostat --show CPU,Bzy_MHz,Avg_MHz --interval 1 --num_iterations 5
```

```bash
# مقارنة الـ freq في الـ kernel مقابل الـ hardware
watch -n 1 'echo "Kernel: $(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq)"; \
            echo "HW MSR: $(rdmsr 0x198 -d 2>/dev/null || echo N/A)"'
```

**علامات عدم التطابق:**
- `cpuinfo_cur_freq` بيختلف عن `scaling_cur_freq` باستمرار → الـ governor مش عارف يتحكم في الـ HW
- `time_in_state` بيسجّل وقت طويل في freq معينة بس الـ HW عمليًا شغّال على freq تانية → مشكلة في الـ driver أو الـ firmware

---

#### 2. Register Dump Techniques

```bash
# عبر /dev/cpu/X/msr (x86 فقط — يحتاج modprobe msr)
modprobe msr

# قراءة الـ current frequency ratio من IA32_PERF_STATUS
rdmsr -p 0 0x198
# الـ output مثلاً: 0x1a00000000 → الـ bits 15:8 = 0x1a = 26 → 26 × 100MHz = 2.6GHz

# قراءة بالـ devmem2 (للـ MMIO registers — مش شائع في cpufreq)
# devmem2 <physical_address> w

# قراءة الـ ACPI P-state info
cat /proc/acpi/processor/CPU0/performance 2>/dev/null || echo "ACPI not available"

# ARM — قراءة الـ frequency عبر الـ clock framework
cat /sys/kernel/debug/clk/clk_summary | grep -i cpu
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

في الأنظمة المدموجة (embedded) اللي فيها external voltage regulators (PMIC):

- **ابحث عن الـ DVFS signal lines** — عادةً بتبعث pulse عند كل frequency/voltage change
- **قيس الـ Vcore** على الـ oscilloscope — لازم يتغير مع الـ frequency (DVFS)
- **الـ I2C/SPI bus** لو الـ PMIC بيتحكم فيه via software — استخدم logic analyzer لتتبع الأوامر
- **الـ CLK line** للـ CPU — المتوقع تشوف تغيير في الـ frequency بعد ما الـ kernel يبعث طلب

```
Timeline للـ DVFS transition (embedded):
    Kernel write → I2C command to PMIC → Voltage settles → CLK changes
    |←  ~10µs  →|←      ~200µs       →|←   ~100µs    →|←  instant →|
```

---

#### 4. Common Hardware Issues وأنماط الـ Kernel Log

| المشكلة الـ HW | نمط الـ Kernel Log | السبب | الحل |
|---|---|---|---|
| الـ CPU مش بيتجاوز الـ base freq | `trans_table` بيظهر انتقالات لكن `turbostat` بيظهر نفس الـ freq | Turbo Boost معطّل من الـ BIOS أو thermal throttling | تحقق من `dmesg | grep throttl` و`/sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq` |
| الـ stats مش بتتسجّل | `time_in_state` كله صفر رغم الـ load | الـ driver مش بيستقبل notifications | تحقق من إن `cpufreq_stats_record_transition` بيتستدعى عبر kprobe |
| الـ frequency دايمًا fixed | `total_trans = 0` | الـ governor على `performance` أو `powersave` بدون scaling | `cpupower frequency-set -g schedutil` |
| crash عند قراءة `trans_table` | kernel panic أو NULL pointer | `policy->stats` اتحذف بينما حد بيقرأ | race condition — تحقق من `CONFIG_PROVE_LOCKING` |
| `EFBIG` من `trans_table` | `cpufreq transition table exceeds PAGE_SIZE` في `dmesg` | عدد الـ P-states كبير جدًا (> ~64 على x86 حديثة مع Intel HWP) | استخدم `time_in_state` فقط |

```bash
# تحقق من thermal throttling
dmesg | grep -i "thermal\|throttl\|freq\|MHz"

# تحقق من الـ CPU flags للـ turbo support
grep -m1 "flags" /proc/cpuinfo | grep -o "ida\|turbo\|hwp"

# تحقق من الـ max freq مقابل الـ turbo freq
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_max_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
```

---

#### 5. Device Tree Debugging

في الأنظمة المدموجة اللي بتستخدم الـ DT للـ cpufreq:

```bash
# تحقق من الـ DT الفعلي اللي بيشتغل بيه الـ kernel
ls /sys/firmware/devicetree/base/cpus/

# اقرأ الـ OPP table (Operating Performance Points)
# المفروض تطابق الـ freq_table في الـ stats
cat /sys/kernel/debug/opp/opp_summary 2>/dev/null

# تحقق من الـ OPP nodes في الـ DT
find /sys/firmware/devicetree/base -name "opp*" | head -20

# مقارنة الـ DT frequencies مع الـ kernel frequencies
diff <(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies | tr ' ' '\n' | sort) \
     <(grep -r "opp-hz" /sys/firmware/devicetree/base 2>/dev/null | \
       awk '{print $NF}' | sort)
```

**مشاكل شائعة في الـ DT:**

```
مشكلة: الـ scaling_available_frequencies بيظهر frequencies أقل مما في الـ DT
السبب: بعض الـ OPP nodes فيها "status = disabled" أو الـ regulators مش قادرة توفر الـ voltage المطلوب
الحل: تحقق من الـ OPP table في الـ DT وشوف الـ regulator voltage ranges
```

```bash
# تحقق من الـ cpufreq driver المستخدم
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver

# تحقق من الـ DT node للـ CPU
cat /sys/firmware/devicetree/base/cpus/cpu@0/compatible 2>/dev/null
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**فحص سريع للحالة الكاملة:**

```bash
#!/bin/bash
# cpufreq-stats quick health check
echo "=== CPUFreq Stats Health Check ==="

# تحقق من وجود الـ driver
if [ ! -d /sys/devices/system/cpu/cpu0/cpufreq/stats ]; then
    echo "ERROR: cpufreq-stats not loaded"
    echo "Try: modprobe cpufreq_stats  OR check CONFIG_CPU_FREQ_STAT"
    exit 1
fi

echo "Driver: OK"
echo ""

# عرض stats لكل CPU
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/stats; do
    cpu_num=$(echo $cpu | grep -o 'cpu[0-9]*')
    echo "--- $cpu_num ---"
    echo "Total transitions: $(cat $cpu/total_trans)"
    echo "Time in state:"
    cat $cpu/time_in_state | awk '{printf "  %7d MHz: %d × 10ms = %.2f sec\n", $1/1000, $2, $2/100}'
    echo ""
done
```

**Stress test + monitoring:**

```bash
# terminal 1: ابدأ stress test لإجبار الـ CPU على scaling
stress-ng --cpu 4 --timeout 30s &

# terminal 2: راقب الـ transitions في real time
watch -n 0.5 'cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans; \
              cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq'
```

**Reset والـ benchmark نظيف:**

```bash
# Reset كل الـ stats قبل الـ benchmark
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/stats/reset; do
    echo 1 > $cpu
done

# شغّل الـ workload
./my_benchmark

# اقرأ النتائج
for cpu in /sys/devices/system/cpu/cpu*/cpufreq/stats; do
    echo "=== $(basename $(dirname $cpu)) ==="
    cat $cpu/time_in_state
done
```

**ftrace session كاملة:**

```bash
#!/bin/bash
TRACE_DIR=/sys/kernel/debug/tracing

# Setup
echo 0 > $TRACE_DIR/tracing_on
echo > $TRACE_DIR/trace
echo 1 > $TRACE_DIR/events/power/cpu_frequency/enable

# Start
echo 1 > $TRACE_DIR/tracing_on
echo "Tracing for 10 seconds..."
sleep 10
echo 0 > $TRACE_DIR/tracing_on

# Analyze
echo "=== Frequency transitions ==="
grep "cpu_frequency" $TRACE_DIR/trace | \
    awk '{print $NF}' | sort | uniq -c | sort -rn

echo "=== Timeline ==="
grep "cpu_frequency" $TRACE_DIR/trace | head -50

# Cleanup
echo 0 > $TRACE_DIR/events/power/cpu_frequency/enable
```

**تحقق من الـ trans_table بتفصيل:**

```bash
# اقرأ الـ trans_table ولو جاب EFBIG اعرض رسالة واضحة
for i in $(seq 0 $(nproc --all | awk '{print $1-1}')); do
    RESULT=$(cat /sys/devices/system/cpu/cpu${i}/cpufreq/stats/trans_table 2>&1)
    if echo "$RESULT" | grep -q "File too large"; then
        echo "CPU$i: trans_table too large (>PAGE_SIZE) — use total_trans instead"
        echo "  Total transitions: $(cat /sys/devices/system/cpu/cpu${i}/cpufreq/stats/total_trans)"
    else
        echo "=== CPU$i trans_table ==="
        echo "$RESULT"
    fi
done
```

**kprobe لمراقبة `cpufreq_stats_record_transition`:**

```bash
# أضف kprobe
echo 'p:freq_trans cpufreq_stats_record_transition new_freq=%si' \
    > /sys/kernel/debug/tracing/kprobe_events

# فعّل الـ probe
echo 1 > /sys/kernel/debug/tracing/events/kprobes/freq_trans/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# راقب
timeout 5 cat /sys/kernel/debug/tracing/trace_pipe

# نظّف
echo 0 > /sys/kernel/debug/tracing/events/kprobes/freq_trans/enable
echo > /sys/kernel/debug/tracing/kprobe_events
```

**الـ output المتوقع وتفسيره:**

```
# مثال على trace_pipe output:
   kworker/0:1-45  [000] .... 1234.567890: freq_trans: new_freq=0xd6d800
   ↑ thread         ↑CPU  ↑flags ↑timestamp  ↑probe name  ↑0xd6d800 = 14080000 Hz ≈ 14 GHz (غلط)
   # ملاحظة: الـ %si يمثل الـ 32-bit lower register، لازم تحسب الـ sign extension

# الطريقة الصح:
echo 'p:freq_trans cpufreq_stats_record_transition new_freq=%r9:u32' \
    > /sys/kernel/debug/tracing/kprobe_events
# أو استخدم الـ cpu_frequency tracepoint مباشرة بدل الـ kprobe
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — `time_in_state` بيوري أرقام غلط

#### العنوان
**الـ time_in_state بيقول إن الـ CPU شغال على أعلى frequency طول الوقت — وده مش صح**

#### السياق
شركة بتبني industrial gateway على RK3562 لتجميع بيانات من sensors صناعية. الـ gateway بيشتغل 24/7 ومفروض يوفر طاقة. الـ power engineer لاحظ إن consumption أعلى من المتوقع.

#### المشكلة
```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state
# Output:
1800000 98023
1416000 312
1200000 18
600000  4
```
الأرقام بتقول إن الـ CPU بيقضي 99% من وقته على 1800 MHz — بس الـ governor هو `schedutil` والـ workload خفيف.

#### التحليل
الـ `cpufreq_stats_update()` بتتحسب كالآتي:

```c
static void cpufreq_stats_update(struct cpufreq_stats *stats,
                                 unsigned long long time)
{
    unsigned long long cur_time = local_clock();
    // بتضيف الفرق بين cur_time و time على last_index
    stats->time_in_state[stats->last_index] += cur_time - time;
    stats->last_time = cur_time;
}
```

الـ `last_index` بيتحدث بس لما تحصل transition فعلية عبر `cpufreq_stats_record_transition()`:

```c
void cpufreq_stats_record_transition(struct cpufreq_policy *policy,
                                     unsigned int new_freq)
{
    // ...
    old_index = stats->last_index;
    new_index = freq_table_get_index(stats, new_freq);

    // لو old == new → مفيش تسجيل
    if (unlikely(old_index == -1 || new_index == -1 || old_index == new_index))
        return;

    cpufreq_stats_update(stats, stats->last_time);
    stats->last_index = new_index;
    // ...
}
```

المشكلة: الـ cpufreq driver بتاع RK3562 (rockchip-cpufreq) بيعمل `set_target_index` لنفس الـ frequency أحياناً بسبب bug في الـ OPP table — فـ `old_index == new_index` وبيرجع من غير ما يسجل. النتيجة: الـ `last_index` فاضل على أعلى frequency من وقت الـ boot.

```bash
# تأكيد: شوف total_trans
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans
# 14  ← عدد قليل جداً رغم الـ uptime الطويل
```

#### الحل
**خطوة 1: Reset الـ stats وراقب من أول**
```bash
echo 1 > /sys/devices/system/cpu/cpu0/cpufreq/stats/reset
sleep 60
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state
```

**خطوة 2: فحص الـ OPP table في Device Tree**
```bash
# ابحث عن duplicate frequencies
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
```

**خطوة 3: إصلاح الـ DT لو فيه OPP مكرر**
```dts
/* قبل الإصلاح — OPP مكرر */
cpu_opp_table: opp-table {
    opp-1800000000 { opp-hz = /bits/ 64 <1800000000>; };
    opp-1800000000 { opp-hz = /bits/ 64 <1800000000>; }; /* مكرر! */
    opp-1416000000 { opp-hz = /bits/ 64 <1416000000>; };
};

/* بعد الإصلاح */
cpu_opp_table: opp-table {
    opp-1800000000 { opp-hz = /bits/ 64 <1800000000>; };
    opp-1416000000 { opp-hz = /bits/ 64 <1416000000>; };
    opp-1200000000 { opp-hz = /bits/ 64 <1200000000>; };
    opp-600000000  { opp-hz = /bits/ 64 <600000000>;  };
};
```

#### الدرس المستفاد
الـ `cpufreq_stats` بيعتمد على الـ transitions الفعلية — لو الـ driver بيبعت نفس الـ frequency مرتين، الـ `old_index == new_index` check بيمنع التسجيل وبتبقى الإحصائيات مشوهة. دايماً تحقق من `total_trans` كـ sanity check أولاً.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ trans_table بترجع `-EFBIG`

#### العنوان
**قراءة `trans_table` بتفشل بـ `-EFBIG` — المنتج بيشتكي إن monitoring tools مش شغالة**

#### السياق
شركة بتبني Android TV box على Allwinner H616 (quad-core Cortex-A53). الـ team بيستخدم tool مبنية على قراءة `trans_table` لتحليل الأداء. بعد تفعيل custom DVFS table، التool بدأت تفشل.

#### المشكلة
```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/trans_table
# cat: /sys/.../trans_table: File too large
```

#### التحليل
الـ `show_trans_table()` بتتحقق من حجم الـ output:

```c
static ssize_t show_trans_table(struct cpufreq_policy *policy, char *buf)
{
    // ...
    if (len >= PAGE_SIZE - 1) {
        pr_warn_once("cpufreq transition table exceeds PAGE_SIZE. Disabling\n");
        return -EFBIG;  // ← هنا بيفشل
    }
    return len;
}
```

الـ `trans_table` هي matrix بحجم `N × N` حيث N هو عدد الـ frequency states. المساحة المطلوبة:

```
N states → header: ~(N × 10) chars
           rows:   N × (N × 10) chars
           Total ≈ N² × 10 bytes
```

لو الـ H616 عنده OPP table كبيرة (مثلاً 20 frequency):
```
20² × 10 = 4000 bytes للـ data
+ headers + formatting = يتجاوز 4096 (PAGE_SIZE)
```

```c
/* في cpufreq_stats_create_table */
alloc_size += count * count * sizeof(int);  // trans_table: N² integers
```

الـ H616 custom OPP table اللي الـ team ضافوها فيها 18 frequency — الـ matrix بقت 18×18 = 324 entry × formatting = أكبر من PAGE_SIZE.

#### الحل
**خطوة 1: تحقق من عدد الـ frequencies**
```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies | wc -w
# 18 ← كتير جداً
```

**خطوة 2: قلل الـ OPP table في الـ DT**
```dts
/* احذف الـ frequencies المتقاربة جداً اللي مش بتفيد */
cpu_opp_table: opp-table {
    /* احتفظ بنقاط مهمة فقط */
    opp-1800000000 { opp-hz = /bits/ 64 <1800000000>; opp-microvolt = <1100000>; };
    opp-1512000000 { opp-hz = /bits/ 64 <1512000000>; opp-microvolt = <1050000>; };
    opp-1200000000 { opp-hz = /bits/ 64 <1200000000>; opp-microvolt = <950000>;  };
    opp-1008000000 { opp-hz = /bits/ 64 <1008000000>; opp-microvolt = <900000>;  };
    opp-816000000  { opp-hz = /bits/ 64 <816000000>;  opp-microvolt = <850000>;  };
    opp-480000000  { opp-hz = /bits/ 64 <480000000>;  opp-microvolt = <800000>;  };
};
/* 6 states → 6×6 = 36 entries → يسع في PAGE_SIZE براحة */
```

**خطوة 3: بديل برمجي — استخدم `time_in_state` بدل `trans_table`**
```bash
# بيدي نفس المعلومات المهمة من غير حد الـ PAGE_SIZE
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans
```

#### الدرس المستفاد
الـ `trans_table` محدودة بـ `PAGE_SIZE` (4096 bytes) بسبب طبيعة الـ sysfs read. عدد الـ OPP states مش بس بيأثر على الـ stats — بيأثر على قابلية استخدام monitoring tools. خلي عدد الـ OPP states أقل من 12 لضمان إن `trans_table` تشتغل.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — الـ reset بيخلي `time_in_state` يوري قيم عشوائية مؤقتاً

#### العنوان
**بعد كتابة الـ reset، monitoring script بيقرأ قيم كبيرة جداً قبل ما تتصفر**

#### السياق
فريق بيبني IoT sensor node على STM32MP1 (dual Cortex-A7). بيستخدموا script Python تقرأ `time_in_state` كل 10 ثواني وتبعت البيانات لـ MQTT broker. لاحظوا إن بعض القراءات بترجع قيم ضخمة جداً (overflow-like values) مباشرة بعد الـ reset.

#### المشكلة
```python
# Script بسيطة
with open('/sys/devices/system/cpu/cpu0/cpufreq/stats/reset', 'w') as f:
    f.write('1')
# قراءة فورية بعد الـ reset
with open('/sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state', 'r') as f:
    data = f.read()
    # أحياناً بترجع قيمة ضخمة للـ last_index state
```

#### التحليل
الـ reset مش فوري — بيتأجل (deferred reset) لحين حصول transition:

```c
/* store_reset — بس بيسجل الـ reset_time ويرفع الـ flag */
static ssize_t store_reset(struct cpufreq_policy *policy, const char *buf,
                           size_t count)
{
    WRITE_ONCE(stats->reset_time, local_clock());
    smp_wmb();
    WRITE_ONCE(stats->reset_pending, 1);  // flag بس، مش reset فعلي
    return count;
}
```

الـ `show_time_in_state()` بتتعامل مع الحالتين:

```c
static ssize_t show_time_in_state(struct cpufreq_policy *policy, char *buf)
{
    bool pending = READ_ONCE(stats->reset_pending);
    // ...
    for (i = 0; i < stats->state_num; i++) {
        if (pending) {
            if (i == stats->last_index) {
                smp_rmb();
                // الوقت من reset_time لحد دلوقتي
                time = local_clock() - READ_ONCE(stats->reset_time);
            } else {
                time = 0;  // باقي الـ states = صفر
            }
        } else {
            time = stats->time_in_state[i];
            if (i == stats->last_index)
                time += local_clock() - stats->last_time;
        }
        // ...
    }
}
```

المشكلة دقيقة: لو الـ script قرأت في نافذة زمنية ضيقة بين ما الـ `reset_pending` يتعمله `WRITE_ONCE` وقبل ما الـ smp_rmb يكمل — ممكن `reset_time` يُقرأ قديم على بعض architectures. لكن على STM32MP1 (ARM)، المشكلة الأساسية إن الـ script مش بتستنى الـ reset يكمل فعلاً.

الرسم يوضح التسلسل:
```
reset write → reset_pending=1 → [قراءة من script] → transition تحصل → reset_table فعلي
                                 ↑
                          هنا time_in_state صح (pending=1 محسوب)
                          لكن لو race condition → reset_time قديم → قيمة غلط
```

#### الحل
**خطوة 1: استنى بعد الـ reset قبل القراءة**
```python
import time

def reset_and_read_stats(cpu=0):
    base = f'/sys/devices/system/cpu/cpu{cpu}/cpufreq/stats'

    # اعمل reset
    with open(f'{base}/reset', 'w') as f:
        f.write('1')

    # استنى transition تحصل (schedutil بيتحرك خلال ~10ms)
    time.sleep(0.1)

    with open(f'{base}/time_in_state', 'r') as f:
        return f.read()
```

**خطوة 2: تحقق إن الـ transition حصلت فعلاً**
```python
def safe_reset(cpu=0):
    base = f'/sys/devices/system/cpu/cpu{cpu}/cpufreq/stats'

    # اقرأ total_trans قبل
    with open(f'{base}/total_trans') as f:
        before = int(f.read())

    with open(f'{base}/reset', 'w') as f:
        f.write('1')

    # استنى لحد ما transition تحصل
    for _ in range(50):
        time.sleep(0.01)
        with open(f'{base}/total_trans') as f:
            after = int(f.read())
        if after == 0:  # reset كمل
            break
```

#### الدرس المستفاد
الـ `reset` في `cpufreq_stats` هو **deferred reset** — بيتنفذ فعلاً في `cpufreq_stats_record_transition()` مش فوراً. الـ `show_time_in_state()` بتتعامل مع الحالة الانتقالية صح، لكن userspace code لازم يستوعب إن الـ reset مش atomic ويستنى قليلاً أو يتحقق من `total_trans`.

---

### السيناريو 4: Automotive ECU على i.MX8MP — الـ cpufreq stats مش موجودة بعد الـ boot

#### العنوان
**الـ stats directory مش موجودة على بعض الـ CPUs بعد الـ cold boot**

#### السياق
شركة automotive بتبني ECU على i.MX8MP (quad Cortex-A53). الـ system بيستخدم CPU isolation — بعض الـ cores محجوزة لـ real-time tasks. بعد الـ boot، الـ monitoring tool لاقت إن `/sys/devices/system/cpu/cpu2/cpufreq/stats/` مش موجود.

#### المشكلة
```bash
ls /sys/devices/system/cpu/cpu0/cpufreq/stats/
# time_in_state  total_trans  trans_table  reset  ← موجود

ls /sys/devices/system/cpu/cpu2/cpufreq/stats/
# ls: cannot access '...': No such file or directory
```

#### التحليل
الـ `cpufreq_stats_create_table()` بتُستدعى لما الـ policy تتعمل:

```c
void cpufreq_stats_create_table(struct cpufreq_policy *policy)
{
    unsigned int i = 0, count;
    struct cpufreq_stats *stats;

    count = cpufreq_table_count_valid_entries(policy);
    if (!count)
        return;  // ← لو مفيش valid frequencies → مفيش stats

    /* stats already initialized */
    if (policy->stats)
        return;

    // ...
    policy->stats = stats;
    if (!sysfs_create_group(&policy->kobj, &stats_attr_group))
        return;  // ← success

    /* We failed, release resources */
    policy->stats = NULL;  // ← لو sysfs_create_group فشل
    // ...
}
```

على i.MX8MP مع CPU isolation، الـ cpu2 و cpu3 بيتعمل isolate عن طريق `isolcpus=2,3` في kernel cmdline. ده بيأثر على إزاي الـ cpufreq policy بتتبنى. المشكلة: لو الـ cpufreq driver بيعمل policy مشتركة (shared policy) للـ cluster كله، وكان cpu2 isolated قبل ما الـ policy تتسجل، ممكن `sysfs_create_group` يفشل لأن الـ kobject مش موجود أو في حالة غير متوقعة.

```bash
# تحقق من الـ dmesg
dmesg | grep cpufreq
# [    2.341] cpufreq_stats: sysfs group creation failed for cpu2
```

#### الحل
**خطوة 1: فهم الـ policy topology**
```bash
# على i.MX8MP — كل الـ cores في cluster واحد
cat /sys/devices/system/cpu/cpu0/cpufreq/related_cpus
# 0 1 2 3

# الـ stats موجودة على الـ policy CPU فقط
ls /sys/devices/system/cpu/cpu0/cpufreq/stats/  # ← ده الـ policy CPU
```

**خطوة 2: استخدم الـ policy CPU الصح**
```bash
# ابحث عن الـ cpufreq policy CPU لكل core
for cpu in 0 1 2 3; do
    policy=$(cat /sys/devices/system/cpu/cpu${cpu}/cpufreq/cpuinfo_cur_freq 2>/dev/null && echo "cpu${cpu}" || echo "N/A")
    echo "cpu${cpu}: ${policy}"
done
```

**خطوة 3: إصلاح الـ DT لفصل الـ clusters**
```dts
/* لو محتاج كل CPU يبقى له policy مستقلة */
cpus {
    cpu0: cpu@0 {
        clocks = <&clk IMX8MP_CLK_ARM>;
        /* لكل core OPP table مستقلة */
        operating-points-v2 = <&cpu0_opp_table>;
    };
    cpu2: cpu@2 {
        /* isolated CPU — لا تعمل cpufreq ليه */
        /delete-property/ operating-points-v2;
    };
};
```

**خطوة 4: أو اضبط الـ monitoring على الـ policy CPU فقط**
```bash
# اقرأ stats من الـ policy CPU مش من كل CPU
POLICY_CPU=$(cat /sys/devices/system/cpu/cpufreq/policy0/cpuinfo_cur_freq &>/dev/null && echo "0")
cat /sys/devices/system/cpu/cpu${POLICY_CPU}/cpufreq/stats/time_in_state
```

#### الدرس المستفاد
الـ `cpufreq_stats` بتُنشأ على مستوى الـ **policy** مش الـ **CPU الفردي**. في الـ SMP systems اللي فيها shared cpufreq policy، الـ stats بتظهر على الـ policy CPU بس. الـ CPU isolation بيعقد الأمور أكتر — لازم تفهم الـ topology قبل ما تبني monitoring.

---

### السيناريو 5: Custom Board Bring-up على AM62x — `last_index` بيبقى `-1` وبيمنع تسجيل أي transition

#### العنوان
**الـ total_trans فاضل صفر طول الوقت رغم إن الـ governor شغال**

#### السياق
فريق hardware bring-up بيشتغل على board مبنية على TI AM62x (quad Cortex-A53). الـ board جديدة وبيعملوا bring-up للـ cpufreq. الـ governor شغال والـ frequency بتتغير (مؤكد من oscilloscope)، لكن `total_trans` فاضل صفر.

#### المشكلة
```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans
# 0  ← رغم إن الـ uptime 30 دقيقة والـ governor نشيط

cat /sys/devices/system/cpu/cpu0/cpufreq/stats/time_in_state
# 1400000 0
# 1000000 0
# 800000  0
# 400000  180000
```

كل الوقت على 400MHz وده غلط — المفروض الـ governor يرفع الـ frequency.

#### التحليل
الـ `cpufreq_stats_create_table()` بتحسب `last_index` من `policy->cur`:

```c
void cpufreq_stats_create_table(struct cpufreq_policy *policy)
{
    // ...
    stats->last_time = local_clock();
    stats->last_index = freq_table_get_index(stats, policy->cur);
    // ↑ لو policy->cur مش موجود في freq_table → last_index = -1
    policy->stats = stats;
    // ...
}
```

الـ `freq_table_get_index()`:

```c
static int freq_table_get_index(struct cpufreq_stats *stats, unsigned int freq)
{
    int index;
    for (index = 0; index < stats->max_state; index++)
        if (stats->freq_table[index] == freq)
            return index;
    return -1;  // ← مش لاقيها!
}
```

وفي `cpufreq_stats_record_transition()`:

```c
void cpufreq_stats_record_transition(struct cpufreq_policy *policy,
                                     unsigned int new_freq)
{
    // ...
    old_index = stats->last_index;  // ← -1
    new_index = freq_table_get_index(stats, new_freq);

    if (unlikely(old_index == -1 || new_index == -1 || old_index == new_index))
        return;  // ← بيرجع دايماً لأن old_index = -1!
    // ...
}
```

على AM62x، الـ cpufreq driver بيقرأ `policy->cur` من hardware register مباشرة أثناء الـ init. لو الـ hardware رجّع قيمة مش موجودة في الـ OPP table (مثلاً 399000 KHz بدل 400000 KHz بسبب rounding) → `last_index = -1` → كل الـ transitions بتتجاهل.

```bash
# تحقق
cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq
# 399000  ← مش موجود في scaling_available_frequencies!

cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies
# 400000 800000 1000000 1400000
```

#### الحل
**خطوة 1: تأكيد المشكلة**
```bash
# الـ cur_freq مش في الـ available list
CUR=$(cat /sys/devices/system/cpu/cpu0/cpufreq/cpuinfo_cur_freq)
AVAIL=$(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies)
echo "Current: $CUR"
echo "Available: $AVAIL"
# لو CUR مش في AVAIL → هي المشكلة
```

**خطوة 2: إصلاح الـ cpufreq driver أو OPP table**

الإصلاح في الـ driver (TI k3-cpufreq أو am62x clock driver) — اضبط الـ initial frequency reading:

```c
/* في driver bring-up code */
/* بدل قراءة hardware مباشرة، استخدم أقرب OPP */
policy->cur = cpufreq_driver_resolve_freq(policy, hw_freq);
```

أو إصلاح الـ OPP table في DT لتشمل الـ frequency الفعلية:
```dts
cpu0_opp_table: opp-table {
    compatible = "operating-points-v2";
    opp-400000000 {
        opp-hz = /bits/ 64 <400000000>;
        opp-microvolt = <800000>;
    };
    /* أو لو الـ hardware بيدي 399 MHz فعلاً */
    opp-399000000 {
        opp-hz = /bits/ 64 <399000000>;
        opp-microvolt = <800000>;
    };
};
```

**خطوة 3: Workaround فوري — Force transition**
```bash
# اجبر الـ frequency تتغير لـ OPP معروف وترجع
echo userspace > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo 1000000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed
echo 400000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed
echo schedutil > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# دلوقتي last_index اتحدد → الـ stats هتشتغل
cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans
# 2  ← اشتغلت!
```

**خطوة 4: تحقق بعد الإصلاح**
```bash
# راقب لمدة دقيقة
watch -n 5 'cat /sys/devices/system/cpu/cpu0/cpufreq/stats/total_trans'
# المفروض تشوف الأرقام بتزيد
```

#### الدرس المستفاد
الـ `last_index = -1` هو silent killer للـ cpufreq stats — مفيش error message، بس كل الـ transitions بتتجاهل. دايماً في الـ board bring-up تتحقق إن `cpuinfo_cur_freq` موجود في `scaling_available_frequencies`، وإلا الـ stats هتفضل صفر إلى الأبد. ده من أكتر الـ bugs الخفية في بدايات الـ cpufreq bring-up على SoCs جديدة.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| [`Documentation/cpu-freq/cpufreq-stats.rst`](https://www.kernel.org/doc/html/latest/cpu-freq/cpufreq-stats.html) | الوثيقة الرئيسية للـ cpufreq stats في الـ kernel — الـ sysfs interfaces، الـ stats، وطريقة الـ configuration |
| [`Documentation/cpu-freq/index.rst`](https://www.kernel.org/doc/html/latest/cpu-freq/index.html) | الـ index الكامل لوثائق الـ CPUFreq subsystem |
| [`admin-guide/pm/cpufreq.rst`](https://docs.kernel.org/admin-guide/pm/cpufreq.html) | شرح شامل لـ CPU Performance Scaling من منظور الـ admin |
| [`Documentation/ABI/testing/sysfs-devices-system-cpu`](https://www.kernel.org/doc/html/latest/ABI/testing/sysfs-devices-system-cpu.html) | الـ ABI الرسمي لملفات الـ sysfs الخاصة بالـ CPU |

---

### مقالات LWN.net

الـ LWN.net هو المرجع الأساسي لتتبع تطور الـ kernel — الروابط دي بترصد مسيرة الـ cpufreq من الـ stats الأساسية لحد الـ active stats الحديثة.

| المقال | الأهمية |
|--------|---------|
| [Introduce Cpufreq Active Stats](https://lwn.net/Articles/890585/) | patch set بيحل مشكلة إن الـ cpufreq stats بتحسب وقت الـ idle — الـ active stats بتتبع بس الوقت اللي الـ CPU كان فيه شغّال فعلاً |
| [Introduce Active Stats framework with CPU performance statistics](https://lwn.net/Articles/861949/) | الـ framework الأشمل اللي بيتتبع الـ frequency transitions مع الـ idle entry/exit events معاً |
| [Improvements in CPU frequency management](https://lwn.net/Articles/682391/) | مقالة شاملة عن التحسينات في إدارة الـ CPU frequency بما فيها الـ stats |
| [cpufreq: schedutil governor](https://lwn.net/Articles/678777/) | شرح الـ schedutil governor اللي بيستخدم بيانات الـ scheduler بدل الـ polling |
| [CPU frequency governors and remote callbacks](https://lwn.net/Articles/732740/) | تفاصيل آلية الـ remote callbacks في الـ governors |
| [sched: Consolidate cpufreq updates](https://lwn.net/Articles/972506/) | دمج تحديثات الـ cpufreq من الـ scheduler — بيأثر على دقة الـ stats |
| [sched: scheduler-driven CPU frequency selection](https://lwn.net/Articles/667281/) | الأساس النظري لربط الـ scheduler بالـ frequency selection |
| [Frequency-invariant utilization tracking for x86](https://lwn.net/Articles/816388/) | الـ frequency-invariant accounting على x86 وأثره على دقة الـ stats |
| [Scaled statistics using APERF/MPERF in x86](https://lwn.net/Articles/283769/) | استخدام MSRs الـ hardware لإحصاءات الـ frequency أدق |

---

### كود المصدر في الـ Kernel

```
drivers/cpufreq/cpufreq_stats.c     ← الـ driver الأساسي
drivers/cpufreq/cpufreq.c           ← الـ core الي بيستدعي الـ stats notifier
include/linux/cpufreq.h             ← الـ structs والـ macros المشتركة
```

**الـ source على GitHub:**
- [`drivers/cpufreq/cpufreq_stats.c`](https://github.com/torvalds/linux/blob/master/drivers/cpufreq/cpufreq_stats.c) — الكود الكامل للـ stats driver

---

### الـ Kernel Config

| الـ Config | الوصف |
|-----------|-------|
| `CONFIG_CPU_FREQ` | يفعّل الـ CPUFreq subsystem كله |
| `CONFIG_CPU_FREQ_STAT` | يفعّل الـ translation statistics — هو موضوعنا |

**مرجع الـ config:**
- [kernelconfig.io: CONFIG_CPU_FREQ_STAT](https://www.kernelconfig.io/config_cpu_freq_stat)
- [Linux Kernel Driver DataBase: CONFIG_CPU_FREQ_STAT](https://cateee.net/lkddb/web-lkddb/CPU_FREQ_STAT.html)

---

### مناقشات الـ Mailing List والـ Patches

الـ mailing list الرسمي للـ CPUFreq هو **linux-pm@vger.kernel.org**.

| الرابط | الموضوع |
|--------|---------|
| [cpufreq: stats: clear statistics (Patchwork)](https://patchwork.kernel.org/project/linux-pm/patch/20161103204615.85395-1-code@mmayer.net/) | الـ patch اللي أضاف الـ `reset` interface لمسح الإحصاءات بدون reboot |
| [linux-pm archive on lore.kernel.org](https://lore.kernel.org/linux-pm/) | أرشيف الـ mailing list الكامل — ابحث فيه عن `cpufreq_stats` |

---

### صفحات kernelnewbies.org

الروابط دي بترصد التغييرات اللي أثّرت على الـ cpufreq stats عبر إصدارات الـ kernel:

| الصفحة | ما يخص الـ stats |
|--------|-----------------|
| [Linux 4.7 - kernelnewbies](https://kernelnewbies.org/Linux_4.7) | إضافة الـ schedutil governor — بيغير إزاي بيتم الـ frequency selection |
| [Linux 4.10 - kernelnewbies](https://kernelnewbies.org/Linux_4.10) | intel_pstate بقى يشتغل كـ cpufreq driver عادي مع الـ governors |
| [Linux 4.14 - kernelnewbies](https://kernelnewbies.org/Linux_4.14) | الـ schedutil بقى يعمل remote CPU updates |
| [Linux 5.7 - kernelnewbies](https://kernelnewbies.org/Linux_5.7) | frequency-invariant scheduler accounting على x86 |

---

### صفحات elinux.org

| الصفحة | الصلة بالموضوع |
|--------|---------------|
| [Tests: R-CAR-GEN3-CPUFreq](https://elinux.org/Tests:R-CAR-GEN3-CPUFreq) | مثال عملي لاختبار الـ cpufreq على embedded platform — بيتحقق من وجود الـ sysfs directories |
| [CELF PM Requirements 2006](https://www.elinux.org/CELF_PM_Requirements_2006) | متطلبات الـ power management في الـ embedded systems — cpufreq كـ CPU throttling framework |
| [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) | مرجع عام لموارد تطوير الـ kernel |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل ذو الصلة:** الـ sysfs integration وكيف بيشتغل الـ kobject/kset model اللي الـ cpufreq stats بتستخدمه
- **متاح مجاناً:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول ذات الصلة:**
  - Chapter 11: Timers and Time Management — الـ jiffies اللي بيُستخدم في حساب الـ `time_in_state`
  - Chapter 17: Devices and Modules — فهم الـ driver model
- **الأهمية:** بيشرح الـ notifier chains اللي الـ cpufreq stats بتعتمد عليها

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل ذو الصلة:** Chapter 14: Kernel Initialization and Power Management
- **الأهمية:** بيشرح الـ cpufreq في سياق الـ embedded systems وإزاي بتأثر على استهلاك الطاقة

---

### وثائق إضافية مفيدة

| المصدر | الرابط |
|--------|--------|
| ArchWiki — CPU frequency scaling | [wiki.archlinux.org/CPU_frequency_scaling](https://wiki.archlinux.org/title/CPU_frequency_scaling) |
| الـ CPUFreq الرسمي القديم (txt) | [kernel.org/doc/Documentation/cpu-freq/cpufreq-stats.txt](https://www.kernel.org/doc/Documentation/cpu-freq/cpufreq-stats.txt) |
| الـ governors القديم (txt) | [kernel.org/doc/Documentation/cpu-freq/governors.txt](https://www.kernel.org/doc/Documentation/cpu-freq/governors.txt) |
| Fedora — Enabling cpufreq stats | [discussion.fedoraproject.org](https://discussion.fedoraproject.org/t/enabling-cpu-frequency-and-voltage-scaling-statistics-in-the-linux-kernel/76772) |

---

### Search Terms للبحث عن معلومات أكثر

لو عايز تعمق أكتر، استخدم الـ search terms دي:

```
# على LWN.net
site:lwn.net cpufreq stats
site:lwn.net "time_in_state"
site:lwn.net cpufreq_stats active

# على Google / GitHub
linux kernel cpufreq_stats.c notifier
CONFIG_CPU_FREQ_STAT sysfs interface
cpufreq trans_table PAGE_SIZE limitation
cpufreq stats reset sysfs
linux pm cpufreq active stats idle

# على lore.kernel.org
https://lore.kernel.org/linux-pm/?q=cpufreq_stats
```
## Phase 8: Writing simple module

### الفكرة

الـ `cpufreq_stats` driver بيسجّل كل transition في تردد الـ CPU عن طريق `cpufreq_stats_record_transition()`. الـ kernel بيطلق الـ tracepoint الـ **`cpu_frequency`** (معرّف في `include/trace/events/power.h`) في كل مرة يتغير فيها تردد الـ CPU فعلياً. ده هو أنسب نقطة نـhook عليها لأنها:

- بتتفعّل بعد كل transition حقيقي.
- بتجيب الـ `frequency` الجديدة والـ `cpu_id` مباشرة.
- آمنة تماماً للقراءة من أي module.

بدل الـ tracepoint، ممكن نستخدم **kprobe** على `cpufreq_stats_record_transition` لأنها exported وبتستقبل الـ `policy` والـ `new_freq` — وده اللي هنعمله هنا لأنه أوضح وأكثر تعليمياً.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * cpufreq_stats_probe.c
 *
 * Hooks cpufreq_stats_record_transition() via kprobe to log every
 * CPU frequency transition with old and new frequencies.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/cpufreq.h>

/* ------------------------------------------------------------------ */
/* kprobe handler — fires just before cpufreq_stats_record_transition  */
/* ------------------------------------------------------------------ */

/*
 * pre_handler runs before the probed function executes.
 * regs holds the CPU register state at the call site, which lets us
 * read the function arguments: rdi = policy, rsi = new_freq (x86_64).
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct cpufreq_policy *policy;
    unsigned int new_freq;
    unsigned int old_freq;

    /*
     * On x86_64 the first argument (policy) is in rdi,
     * the second argument (new_freq) is in rsi.
     * kernel_ulong_t cast keeps it portable across 32/64-bit.
     */
    policy   = (struct cpufreq_policy *)regs->di;
    new_freq = (unsigned int)regs->si;

    /* Guard against a NULL policy — stats may not be initialised yet */
    if (!policy)
        return 0;

    old_freq = policy->cur; /* current freq before the transition */

    pr_info("cpufreq_probe: CPU%u  %u kHz -> %u kHz  (delta %+d kHz)\n",
            policy->cpu,
            old_freq,
            new_freq,
            (int)new_freq - (int)old_freq);

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                    */
/* ------------------------------------------------------------------ */

/*
 * We name the symbol we want to probe. The kernel resolves its address
 * at registration time using kallsyms.
 */
static struct kprobe kp = {
    .symbol_name = "cpufreq_stats_record_transition",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init / module_exit                                            */
/* ------------------------------------------------------------------ */

static int __init cpufreq_probe_init(void)
{
    int ret;

    /*
     * register_kprobe() patches the first byte of the target function
     * with an INT3 (x86) or BRK (arm64) instruction so our handler
     * gets called before the real function body executes.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("cpufreq_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("cpufreq_probe: hooked cpufreq_stats_record_transition at %p\n",
            kp.addr);
    return 0;
}

static void __exit cpufreq_probe_exit(void)
{
    /*
     * unregister_kprobe() restores the original instruction and waits
     * for any in-flight handler invocations to finish before returning.
     * Skipping this would leave a dangling INT3 pointing at freed code.
     */
    unregister_kprobe(&kp);
    pr_info("cpufreq_probe: unhooked cpufreq_stats_record_transition\n");
}

module_init(cpufreq_probe_init);
module_exit(cpufreq_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on cpufreq_stats_record_transition to log freq transitions");
```

---

### Makefile

```makefile
obj-m += cpufreq_stats_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module (`MODULE_LICENSE`، `module_init`، ...) |
| `linux/kernel.h` | `pr_info` / `pr_err` |
| `linux/kprobes.h` | `struct kprobe`، `register_kprobe`، `unregister_kprobe` |
| `linux/cpufreq.h` | تعريف `struct cpufreq_policy` عشان نقدر نقرأ الـ fields منها |

#### الـ `handler_pre`

الـ pre-handler بيتنفّذ **قبل** جسم الـ function الأصلية. لذلك لما نقرأ `policy->cur` داخله بيكون لسه الـ frequency القديمة، ونقدر نقارنها بـ `new_freq` اللي جاية كـ argument ونطبع الـ delta. الـ `regs->di` و`regs->si` بيمثّلوا الـ argument الأول والتاني على الـ x86_64 ABI (System V).

#### الـ `struct kprobe`

بنحط اسم الـ symbol في `symbol_name` بدل عنوان ثابت — الـ kernel بيحلّ العنوان تلقائياً من `kallsyms` عند `register_kprobe`، وده بيخلّي الـ module portable بين kernel versions مختلفة طالما الـ function موجودة.

#### الـ `module_init`

`register_kprobe` بيحط **software breakpoint** (INT3 على x86) في أول بايت من الـ `cpufreq_stats_record_transition`. من غير الـ registration ما فيش hook وما فيش تأثير على النظام.

#### الـ `module_exit`

`unregister_kprobe` بيرجّع الـ byte الأصلي وبيضمن إن أي handler شغّال خلص قبل ما الـ module يُفرّغ من الذاكرة — لو عملنا `rmmod` من غيره هيحصل kernel panic لأن CPU هيحاول ينفّذ كود في memory اتحررت.

---

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميله
sudo insmod cpufreq_stats_probe.ko

# إجبار frequency transition (يتطلب governor مش ondemand/conservative)
sudo cpupower frequency-set -f 1000000

# أو عبر stress
stress-ng --cpu 1 --timeout 5s

# مشاهدة الـ log
sudo dmesg | grep cpufreq_probe

# مثال على الـ output
# cpufreq_probe: CPU0  2400000 kHz -> 800000 kHz  (delta -1600000 kHz)
# cpufreq_probe: CPU0  800000 kHz -> 3600000 kHz  (delta +2800000 kHz)

# إزالة الـ module
sudo rmmod cpufreq_stats_probe
```

> **ملاحظة:** لو الـ kernel مبني بـ `CONFIG_CPU_FREQ_STAT=n` فالـ `cpufreq_stats_record_transition` مش موجودة وهيفشل الـ `register_kprobe` بـ `-ENOENT`. تأكد من `grep CONFIG_CPU_FREQ_STAT /boot/config-$(uname -r)`.
