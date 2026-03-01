## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: CPU Hotplug

الـ file ده جزء من subsystem اسمه **CPU Hotplug** في الـ Linux kernel، واللي بيتولاه Thomas Gleixner و Peter Zijlstra. الـ subsystem ده مسؤول عن إمكانية تشغيل وإيقاف الـ CPU cores وقت تشغيل النظام من غير ما تعمل reboot.

---

### القصة: ليه محتاج حاجة زي دي أصلاً؟

تخيل عندك سيرفر فيه 128 CPU core. النظام بيشتغل، والـ load خفيف، فالـ power management قرر يطفي 64 core توفيرًا للكهرباء. بعدين جه load تقيل، فلازم يرجع يشغلهم. ده بالظبط اللي بيعمله **CPU Hotplug**.

دلوقتي فيه مشكلة:

كل CPU لما بييجي online، بيحتاج **threads خاصة بيه** تشتغل عليه — مش threads عامة تتوزع على أي CPU، لأ، كل CPU له نسخته هو. مثلاً:
- الـ `ksoftirqd` — بيعالج الـ soft interrupts على كل CPU
- الـ `rcuc` — بيعمل RCU callbacks على كل CPU
- الـ `migration` — بينقل tasks بين الـ CPUs

المشكلة إن لما CPU يتشغل أو يتوقف، لازم كل الـ threads دي تتعمل أو تتوقف معاه أوتوماتيك. وكل subsystem في الـ kernel ليه threads زي دي، والكل محتاج ينفس الدم ده.

---

### الحل: `smpboot.h` و `struct smp_hotplug_thread`

الـ file ده بيقدم **framework** يخلي أي subsystem يقول للـ kernel:

> "أنا عندي thread لازم يشتغل على كل CPU. خد وصفه واتكفل انت بتشغيله وإيقافه مع كل CPU."

بدل ما كل subsystem يكتب كود hotplug من الأول، بيملا **`struct smp_hotplug_thread`** بمعلومات الـ thread، وبيسجله مرة واحدة بـ `smpboot_register_percpu_thread()`.

---

### الـ `struct smp_hotplug_thread` — كل حقل وليه إيه

```c
struct smp_hotplug_thread {
    /* pointer لـ per-CPU storage للـ task_struct بتاع الـ thread على كل CPU */
    struct task_struct * __percpu *store;

    /* ربط الـ struct ده في list داخلية يديرها الـ kernel */
    struct list_head list;

    /* بيسأل: هل الـ thread المفروض يشتغل دلوقتي؟ (بيتنادى مع disabled preemption) */
    int (*thread_should_run)(unsigned int cpu);

    /* الوظيفة الأساسية للـ thread — ده اللي بيتعمل فعلاً */
    void (*thread_fn)(unsigned int cpu);

    /* اختياري: بيتنادى لما الـ thread يتعمل لأول مرة (مش من context الـ thread) */
    void (*create)(unsigned int cpu);

    /* اختياري: بيتنادى لما الـ thread يبدأ يشتغل للمرة الأولى */
    void (*setup)(unsigned int cpu);

    /* اختياري: بيتنادى لما الـ thread يوقف (عند خروج الـ module) */
    void (*cleanup)(unsigned int cpu, bool online);

    /* اختياري: بيتنادى لما الـ CPU يروح offline — الـ thread بيتـ"park" */
    void (*park)(unsigned int cpu);

    /* اختياري: بيتنادى لما الـ CPU يرجع online — الـ thread بيتـ"unpark" */
    void (*unpark)(unsigned int cpu);

    /* لو true، الـ thread بيتكفل بـ parking نفسه من غير ما الـ framework يتدخل */
    bool selfparking;

    /* اسم الـ thread زي "ksoftirqd/%u" */
    const char *thread_comm;
};
```

---

### Lifecycle الـ Thread — من الـ park للـ unpark

```
CPU comes ONLINE
      |
      v
  [create(cpu)]        <-- اختياري، بيتنادى مرة وبس
      |
      v
  [setup(cpu)]         <-- اختياري، لما الـ thread يبدأ يشتغل فعلاً
      |
      v
  thread loop:
    if thread_should_run(cpu):
        thread_fn(cpu)
    else:
        schedule()     <-- ينام لو مفيش شغل

CPU goes OFFLINE
      |
      v
  [park(cpu)]          <-- الـ thread يتوقف مؤقتاً

CPU comes ONLINE again
      |
      v
  [unpark(cpu)]        <-- الـ thread يرجع يشتغل

Module unload / shutdown
      |
      v
  [cleanup(cpu, online)]
```

---

### مثال واقعي: الـ `ksoftirqd`

في `/workspace/external/linux/kernel/softirq.c`:

```c
static struct smp_hotplug_thread softirq_threads = {
    .store          = &ksoftirqd,
    .thread_should_run = ksoftirqd_should_run,
    .thread_fn      = run_ksoftirqd,
    .thread_comm    = "ksoftirqd/%u",
};

// عند الـ init
BUG_ON(smpboot_register_percpu_thread(&softirq_threads));
```

الـ kernel دلوقتي عارف إنه لازم يعمل `ksoftirqd/0`, `ksoftirqd/1`, ... على كل CPU، وييجيهم مع الـ CPU لو اتشغل أو اتوقف.

---

### مين بيستخدم الـ framework ده؟

| الـ Subsystem | الـ Thread | الملف |
|---|---|---|
| Soft IRQ | `ksoftirqd/%u` | `kernel/softirq.c` |
| CPU Hotplug itself | `cpuhp/%u` | `kernel/cpu.c` |
| IRQ Work | `irq_work/%u` | `kernel/irq_work.c` |
| RCU | `rcuc/%u` | `kernel/rcu/tree.c` |
| Stop Machine | `migration/%u` | `kernel/stop_machine.c` |
| Network Backlog | `ksoftirqd` variant | `net/core/dev.c` |
| Idle Injection | `idle_inject/%u` | `drivers/powercap/idle_inject.c` |

---

### الفرق بين Park و Stop

| | **Park** | **Stop** |
|---|---|---|
| متى | CPU بيروح offline مؤقتاً | module بيتشال أو kernel shutdown |
| الـ thread | بيفضل موجود في الـ memory | بيتمسح خالص |
| الرجوع | ممكن برـ unpark | لأ |

---

### الملفات المهمة اللي المفروض تعرفها

**Core files:**

- `include/linux/smpboot.h` — الـ public API والـ struct (الملف ده)
- `kernel/smpboot.c` — الـ implementation الكاملة، فيها الـ thread loop والـ park/unpark logic
- `kernel/smpboot.h` — internal API للـ idle threads وعمليات الـ park/unpark

**Hotplug framework:**

- `include/linux/cpuhotplug.h` — state machine الـ CPU hotplug (CPUHP_OFFLINE → CPUHP_ONLINE)
- `include/linux/cpu.h` — الـ CPU management API
- `kernel/cpu.c` — الـ implementation الرئيسية للـ CPU hotplug

**مستخدمين حقيقيين:**

- `kernel/softirq.c` — `ksoftirqd`
- `kernel/stop_machine.c` — `migration`
- `kernel/rcu/tree.c` — `rcuc`
- `kernel/irq_work.c` — `irq_work`
## Phase 2: شرح الـ SMP Boot / Per-CPU Hotplug Threads Framework

---

### المشكلة اللي بيحلها الـ Subsystem ده

في أي نظام **SMP (Symmetric Multi-Processing)** — يعني معالج بأكتر من core — في خدمات kernel بتحتاج تشتغل على **كل CPU بالظبط**، مش على أي CPU عشوائي.

أمثلة واقعية:

| الخدمة | ليه لازم تبقى per-CPU؟ |
|---|---|
| `ksoftirqd` | كل CPU يعالج الـ softirqs بتاعته هو |
| `migration` thread | نقل tasks بين CPUs يحتاج thread على كل CPU |
| `watchdog` thread | مراقبة كل CPU بشكل مستقل |
| RCU callbacks | كل CPU يعالل callbacks بتاعته |

**المشكلة المركبة:** لما CPU بيتوعش (online) أو بيتقفل (offline) عن طريق **CPU Hotplug**، لازم كل الـ threads دي تتعمل/تتوقف/تتعلق بشكل منتظم ومتزامن — من غير ما كل driver يكتب نفس الـ boilerplate من الصفر.

---

### الحل اللي بيقدمه الـ Kernel

الـ kernel بيقدم **smpboot framework** — طبقة abstraction بتتولى:

1. إنشاء kthread واحد لكل CPU (per-CPU kthread) لكل "نوع" thread مسجّل
2. إدارة دورة حياة الـ thread كاملة مع الـ CPU hotplug events تلقائيًا
3. توفير lifecycle callbacks للـ driver (setup, cleanup, park, unpark) من غير ما يشيل هم الـ scheduling loop

الـ driver بس بيملي `struct smp_hotplug_thread` بالـ function pointers اللي محتاجها، والـ framework يتكفل بالباقي.

---

### تشبيه من الواقع (مع mapping كامل)

تخيل **شركة فروع** (زي فودافون مثلاً) بتفتح فروع جديدة وبتقفل فروع في مدن مختلفة:

| عنصر الشركة | المقابل في الـ Kernel |
|---|---|
| الشركة الأم (HR system) | `smpboot framework` نفسه |
| كل مدينة فيها فرع | كل CPU في الـ system |
| الوظيفة (مثلاً محاسب) | `struct smp_hotplug_thread` |
| الموظف في كل فرع | الـ kthread على كل CPU |
| فتح فرع جديد | CPU online (hotplug up) |
| تعليق نشاط الفرع مؤقتاً | CPU park (offline مؤقت) |
| استئناف الفرع | CPU unpark (online تاني) |
| إغلاق الفرع نهائياً | CPU offline نهائي / unregister |
| تعيين موظف + تدريبه | `create` callback + `setup` callback |
| إجازة إجبارية | `park` callback |
| عودة من الإجازة | `unpark` callback |
| إنهاء الخدمة | `cleanup` callback |
| قائمة الوظائف المعتمدة | `hotplug_threads` list |

**الـ HR system (framework)** بيعرف إمتى فرع بيفتح وإمتى بيقفل، وبيتعامل مع كل موظف بالوظيفة دي في الفرع ده تلقائياً. الـ driver (صاحب الوظيفة) بس يحدد "إيه اللي المحاسب المفروض يعمله".

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kernel Subsystems                        │
│                                                                 │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│   │  scheduler   │  │  RCU core    │  │  watchdog / softirq  │ │
│   │  (migration) │  │  (callbacks) │  │  (ksoftirqd etc.)    │ │
│   └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │
│          │                 │                      │             │
│          └─────────────────┼──────────────────────┘             │
│                            │  smpboot_register_percpu_thread()  │
│                            ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              smpboot framework                          │   │
│   │                                                         │   │
│   │   hotplug_threads list:                                 │   │
│   │   ┌────────────────┐  ┌────────────────┐               │   │
│   │   │smp_hotplug_thd │→ │smp_hotplug_thd │→ ...          │   │
│   │   │ (migration)    │  │ (watchdog)     │               │   │
│   │   └────────────────┘  └────────────────┘               │   │
│   │                                                         │   │
│   │   Per-CPU kthreads (one per registered type per CPU):   │   │
│   │   CPU0: [migration/0] [watchdog/0] [ksoftirqd/0] ...   │   │
│   │   CPU1: [migration/1] [watchdog/1] [ksoftirqd/1] ...   │   │
│   │   CPU2: [migration/2] [watchdog/2] [ksoftirqd/2] ...   │   │
│   └──────────────────────────┬──────────────────────────────┘   │
│                              │ hooks into                       │
│                              ▼                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              CPU Hotplug Subsystem (cpuhp)               │   │
│   │                                                         │   │
│   │  CPU online  ──► smpboot_create_threads()               │   │
│   │               ──► smpboot_unpark_threads()              │   │
│   │  CPU offline ──► smpboot_park_threads()                 │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

> **ملاحظة:** الـ **CPU Hotplug subsystem** هو subsystem مستقل بيدير state machine لكل CPU (offline → online وعكسه). الـ smpboot framework بيشترك في الـ hotplug callbacks دي عشان يعرف يدير الـ threads.

---

### الـ Core Abstraction

الـ central idea هي `struct smp_hotplug_thread`:

```c
struct smp_hotplug_thread {
    /* per-CPU storage: مؤشر لـ task_struct لكل CPU */
    struct task_struct * __percpu *store;

    /* entry في الـ global hotplug_threads list */
    struct list_head list;

    /*
     * thread_should_run: بيتسأل كل دورة في الـ loop
     * لو رجع 0 → thread_fn مش هتتشتغل، الـ thread بيخد sleep
     * لو رجع 1 → thread_fn بتشتغل فوراً
     * بيتكال وسط preempt_disable() — مهم جداً!
     */
    int  (*thread_should_run)(unsigned int cpu);

    /* الشغل الفعلي اللي المفروض الـ thread يعمله */
    void (*thread_fn)(unsigned int cpu);

    /* Lifecycle callbacks */
    void (*create)(unsigned int cpu);       /* بعد إنشاء الـ kthread مباشرة */
    void (*setup)(unsigned int cpu);        /* أول ما الـ thread يبدأ يشتغل */
    void (*cleanup)(unsigned int cpu, bool online); /* قبل ما الـ thread يوقف */
    void (*park)(unsigned int cpu);         /* لما الـ CPU بيتعمله offline */
    void (*unpark)(unsigned int cpu);       /* لما الـ CPU بيرجع online */

    /* لو الـ thread بيدير parking بنفسه (نادر) */
    bool selfparking;

    /* اسم الـ thread (هيظهر في ps زي "migration/0") */
    const char *thread_comm;
};
```

---

### الـ Internal Engine: `smpboot_thread_fn`

ده الـ thread loop اللي بيشغله الـ framework لكل مسجّل thread — الـ driver مش بيشوفه، بس لازم تفهمه:

```c
static int smpboot_thread_fn(void *data)
{
    struct smpboot_thread_data *td = data;   /* cpu + status + ht */
    struct smp_hotplug_thread  *ht = td->ht;

    while (1) {
        set_current_state(TASK_INTERRUPTIBLE); /* استعداد للنوم */
        preempt_disable();

        /* طلب إيقاف نهائي؟ */
        if (kthread_should_stop()) {
            if (ht->cleanup && td->status != HP_THREAD_NONE)
                ht->cleanup(td->cpu, cpu_online(td->cpu));
            kfree(td);
            return 0;
        }

        /* طلب park (CPU offline مؤقت)؟ */
        if (kthread_should_park()) {
            if (ht->park && td->status == HP_THREAD_ACTIVE) {
                ht->park(td->cpu);
                td->status = HP_THREAD_PARKED;
            }
            kthread_parkme(); /* يعلق هنا لحد ما يتعمله unpark */
            continue;
        }

        /* State machine: أول تشغيل أو رجوع من park */
        switch (td->status) {
        case HP_THREAD_NONE:
            if (ht->setup) ht->setup(td->cpu);
            td->status = HP_THREAD_ACTIVE;
            continue;
        case HP_THREAD_PARKED:
            if (ht->unpark) ht->unpark(td->cpu);
            td->status = HP_THREAD_ACTIVE;
            continue;
        }

        /* الحالة العادية: شغل ولا نوم؟ */
        if (!ht->thread_should_run(td->cpu)) {
            preempt_enable_no_resched();
            schedule(); /* نوم لحد ما حاجة توقظه */
        } else {
            ht->thread_fn(td->cpu); /* شغل! */
        }
    }
}
```

**الـ state machine الداخلية:**

```
                    ┌──────────────┐
                    │ HP_THREAD_   │ ← أول ما الـ thread يبدأ
                    │    NONE      │
                    └──────┬───────┘
                           │ setup() callback
                           ▼
                    ┌──────────────┐
           ┌───────►│ HP_THREAD_   │◄──────────────┐
           │        │   ACTIVE     │               │
           │        └──┬───────────┘               │
           │           │ park() callback            │ unpark() callback
           │  thread_  │                            │
           │  fn()     ▼                            │
           │    ┌──────────────┐                   │
           └────┤ HP_THREAD_   │───────────────────┘
                │   PARKED     │
                └──────────────┘
                      │
                      │ kthread_stop()
                      ▼
                 cleanup() → thread exits
```

---

### هياكل البيانات مع بعضها

```
hotplug_threads (global list)
        │
        ├── smp_hotplug_thread  [migration]
        │       ├── store → per_cpu ptr → [CPU0: task*] [CPU1: task*] [CPU2: task*]
        │       ├── thread_should_run()
        │       ├── thread_fn()
        │       ├── setup() / cleanup() / park() / unpark()
        │       └── list → next ht
        │
        └── smp_hotplug_thread  [watchdog]
                ├── store → per_cpu ptr → [CPU0: task*] [CPU1: task*] [CPU2: task*]
                ├── thread_should_run()
                ├── thread_fn()
                └── list → NULL


per_cpu kthread على CPU0 يشيل:
┌─────────────────────────────────┐
│   task_struct  [migration/0]    │
│   + smpboot_thread_data         │
│       ├── cpu   = 0             │
│       ├── status= HP_THREAD_ACTIVE │
│       └── ht   → smp_hotplug_thread [migration] │
└─────────────────────────────────┘
```

---

### الـ Lifecycle كامل: من Register لـ Unregister

```
Driver calls:
smpboot_register_percpu_thread(&my_ht)
        │
        ├── lock cpus_read + smpboot_threads_lock
        ├── for_each_online_cpu(cpu):
        │       ├── __smpboot_create_thread(my_ht, cpu)
        │       │       ├── kzalloc smpboot_thread_data
        │       │       ├── kthread_create_on_cpu(smpboot_thread_fn, td, cpu, name)
        │       │       ├── kthread_park(tsk)  ← يبدأ parked
        │       │       ├── wait_task_inactive() ← ضمان إن الـ thread فعلاً اتعلق
        │       │       └── ht->create(cpu) ← callback اختياري
        │       └── smpboot_unpark_thread(my_ht, cpu)
        │               └── kthread_unpark(tsk) ← يبدأ يشتغل
        └── list_add(&my_ht->list, &hotplug_threads)


لما CPU يتعمله offline:
        smpboot_park_threads(cpu)
                └── for_each ht in hotplug_threads (reverse order):
                        kthread_park(tsk on cpu)
                        ← smpboot_thread_fn يشوف kthread_should_park()
                        ← بيكال ht->park(cpu)
                        ← td->status = HP_THREAD_PARKED
                        ← kthread_parkme() يعلق


لما CPU يرجع online:
        smpboot_unpark_threads(cpu)
                └── for_each ht in hotplug_threads:
                        kthread_unpark(tsk on cpu)
                        ← smpboot_thread_fn يصحى
                        ← case HP_THREAD_PARKED: ht->unpark(cpu)
                        ← td->status = HP_THREAD_ACTIVE


Driver calls:
smpboot_unregister_percpu_thread(&my_ht)
        ├── list_del(&my_ht->list)
        └── smpboot_destroy_threads(my_ht)
                └── for_each_possible_cpu(cpu):
                        kthread_stop_put(tsk)
                        ← ht->cleanup(cpu, online)
```

---

### الـ create vs setup: فرق مهم

| | `create` | `setup` |
|---|---|---|
| **متى بيتكال** | بعد `kthread_create_on_cpu()` مباشرة، والـ thread لسه parked | أول تشغيل فعلي للـ thread على الـ CPU |
| **السياق** | من thread خارجي (مش thread نفسه) | من جوه `smpboot_thread_fn` نفسه |
| **على أنهي CPU** | CPU المنشئ، مش بالضرورة target CPU | نفس الـ CPU المخصص للـ thread |
| **مثال استخدام** | migration thread بيحتاج يعمل scheduler setup قبل ما يبدأ | watchdog بيحتاج يسجل timer |

---

### إيه اللي الـ Framework بيمتلكه vs إيه اللي بيفوضه للـ Driver

**الـ Framework بيمتلك:**
- الـ scheduling loop الداخلية (`smpboot_thread_fn`)
- إنشاء وإيقاف الـ kthreads
- ربط كل thread بـ CPU بعينه (`kthread_create_on_cpu` + `kthread_set_per_cpu`)
- التنسيق مع الـ CPU Hotplug events (park/unpark التلقائي)
- الـ `smpboot_thread_data` (cpu, status, pointer للـ ht)
- الـ `hotplug_threads` global list وحمايتها بـ mutex

**الـ Framework بيفوض للـ Driver:**
- متى المفروض الـ thread يشتغل (`thread_should_run`)
- إيه الشغل الفعلي (`thread_fn`)
- أي initialization خاص (`create`, `setup`)
- أي cleanup خاص (`cleanup`, `park`, `unpark`)
- اسم الـ thread (`thread_comm`)
- per-CPU storage لـ task pointers (`store`)

---

### مثال واقعي: تسجيل Watchdog Thread

```c
/* من kernel/watchdog.c - مبسط للفهم */
static DEFINE_PER_CPU(struct task_struct *, softlockup_watchdog);

static struct smp_hotplug_thread watchdog_threads = {
    .store              = &softlockup_watchdog,
    .thread_should_run  = watchdog_should_run,  /* لو في pending resets */
    .thread_fn          = watchdog,             /* يعمل touch على الـ timer */
    .thread_comm        = "watchdog",           /* اسمه watchdog/0, watchdog/1 */
    .setup              = watchdog_enable,      /* يشغل hrtimer للـ CPU ده */
    .cleanup            = watchdog_cleanup,     /* يوقف hrtimer */
    .park               = watchdog_disable,     /* CPU offline: وقف المراقبة */
    .unpark             = watchdog_enable,      /* CPU online: ابدأ المراقبة */
};

/* تسجيل مرة واحدة — الـ framework يتكفل بكل CPU */
smpboot_register_percpu_thread(&watchdog_threads);
```

النتيجة: في `ps aux` هتلاقي:
```
PID   NAME
  9   [watchdog/0]
 10   [watchdog/1]
 11   [watchdog/2]
 12   [watchdog/3]
```

كل thread مربوط بـ CPU بعينه، ولو CPU2 اتعمله offline، `watchdog/2` بيتعمله park تلقائياً.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Flags — Cheatsheet

#### enum حالات الـ thread (مُعرَّف في `kernel/smpboot.c`)

| القيمة | الاسم | المعنى |
|--------|-------|---------|
| `0` | `HP_THREAD_NONE` | الـ thread اتعمل بس لسه ما اتـ setup |
| `1` | `HP_THREAD_ACTIVE` | شغال، الـ `setup()` اتنفذ |
| `2` | `HP_THREAD_PARKED` | متوقف مؤقتاً (CPU offline)، الـ `park()` اتنفذ |

#### config options ذات صلة

| الـ Config | الأثر |
|------------|-------|
| `CONFIG_GENERIC_SMP_IDLE_THREAD` | يفعّل إدارة الـ idle threads عبر `smpboot.c` (fork + reuse on hotplug) |
| `CONFIG_SMP` | الـ smpboot infrastructure كله يحتاجه |
| `CONFIG_HOTPLUG_CPU` | يفعّل مسارات park/unpark عند CPU online/offline |

#### الـ Locking Constants

| المتغير | النوع | يحمي |
|---------|-------|-------|
| `smpboot_threads_lock` | `struct mutex` | قائمة `hotplug_threads` وكل عمليات create/destroy/park |
| `cpu_hotplug_lock` (عبر `cpus_read_lock`) | `percpu_rwsem` | يمنع تغيير CPU topology أثناء register/unregister |

---

### 1. الـ Structs المهمة

---

#### `struct smp_hotplug_thread`

**الغرض:** الـ descriptor اللي بيعرّف نوع الـ per-CPU thread المرتبط بالـ hotplug. الـ driver أو الـ subsystem بيملأه وبيسجّله مرة واحدة، والـ core بعدين بيعمل thread على كل CPU.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `store` | `struct task_struct * __percpu *` | مصفوفة per-CPU بتخزن pointer للـ `task_struct` لكل CPU — الـ core يكتب فيها |
| `list` | `struct list_head` | بيربط الـ descriptor في قائمة `hotplug_threads` العالمية |
| `thread_should_run` | `int (*)(unsigned int cpu)` | **إجبارية** — بتتحقق هل في شغل؟ بتتكال مع preemption disabled |
| `thread_fn` | `void (*)(unsigned int cpu)` | **إجبارية** — الـ body الفعلي للـ thread، بتتكال كل ما `thread_should_run` ترجع non-zero |
| `create` | `void (*)(unsigned int cpu)` | **اختيارية** — بتتكال بعد ما الـ thread يتعمل ويتـ park (مش من context الـ thread) |
| `setup` | `void (*)(unsigned int cpu)` | **اختيارية** — بتتكال أول مرة الـ thread يبقى active بعد `HP_THREAD_NONE` |
| `cleanup` | `void (*)(unsigned int cpu, bool online)` | **اختيارية** — بتتكال عند stop، الـ `online` flag بتوضح حالة الـ CPU |
| `park` | `void (*)(unsigned int cpu)` | **اختيارية** — بتتكال لما الـ CPU يروح offline |
| `unpark` | `void (*)(unsigned int cpu)` | **اختيارية** — بتتكال لما الـ CPU يرجع online من الـ PARKED state |
| `selfparking` | `bool` | لو `true`، الـ core مش بيعمل `kthread_park/unpark` تلقائياً — الـ thread مسؤول عن نفسه |
| `thread_comm` | `const char *` | اسم الـ thread (بيظهر في `ps`، مثال: `"watchdog/%u"`) |

---

#### `struct smpboot_thread_data`

**الغرض:** الـ cookie الخاص بكل CPU — بيتخزن كـ `void *data` في الـ kthread ويتمرر لـ `smpboot_thread_fn`.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `cpu` | `unsigned int` | رقم الـ CPU اللي الـ thread مربوط بيه |
| `status` | `unsigned int` | الحالة الحالية: `HP_THREAD_NONE` / `HP_THREAD_ACTIVE` / `HP_THREAD_PARKED` |
| `ht` | `struct smp_hotplug_thread *` | back-pointer للـ descriptor اللي عمل الـ thread ده |

---

#### `struct task_struct` (مُستخدَم، مش مُعرَّف هنا)

ده الـ process descriptor القياسي في الـ kernel. في سياق smpboot:
- بيتعمل عبر `kthread_create_on_cpu()`
- بيتخزن في `*per_cpu_ptr(ht->store, cpu)`
- الـ smpboot بيعمله `kthread_set_per_cpu()` عشان يبقى مربوط بـ CPU معين

---

### 2. علاقات الـ Structs — ASCII Diagram

```
 ┌─────────────────────────────────────────────────────────┐
 │              hotplug_threads  (LIST_HEAD)                │
 └──────┬──────────────────────────────────────────────────┘
        │ list_head
        ▼
 ┌─────────────────────────────┐      ┌──────────────────────────┐
 │   smp_hotplug_thread (A)    │─────▶│  smp_hotplug_thread (B)  │─▶ ...
 │                             │ list │                          │
 │  store ──────────────────────────────────────┐              │
 │  thread_should_run()        │               │              │
 │  thread_fn()                │               │              │
 │  create / setup / cleanup   │               │              │
 │  park / unpark              │               │              │
 │  selfparking                │               │              │
 │  thread_comm                │               │              │
 └─────────────────────────────┘               │              │
                                               ▼              │
                              ┌────────────────────────────┐  │
                              │  per-CPU store array       │  │
                              │  [cpu0] ──▶ task_struct ◀──┼──┘ (cpu0 of B)
                              │  [cpu1] ──▶ task_struct    │
                              │  [cpu2] ──▶ task_struct    │
                              └────────────────────────────┘
                                        │
                                        │ (each task_struct has)
                                        ▼
                              ┌──────────────────────────┐
                              │   smpboot_thread_data    │
                              │   cpu    = N             │
                              │   status = HP_THREAD_*   │
                              │   ht ────────────────────┼──▶ smp_hotplug_thread
                              └──────────────────────────┘
                              (stored as kthread data pointer)
```

---

### 3. Lifecycle Diagram — من التسجيل للحذف

```
[DRIVER / SUBSYSTEM]
        │
        │  يعبي struct smp_hotplug_thread
        │
        ▼
smpboot_register_percpu_thread(ht)
        │
        ├── cpus_read_lock()          ← يمنع CPU hotplug أثناء التسجيل
        ├── mutex_lock(smpboot_threads_lock)
        │
        ├── for_each_online_cpu(cpu):
        │       ├── __smpboot_create_thread(ht, cpu)
        │       │       ├── kzalloc smpboot_thread_data  [status=HP_THREAD_NONE]
        │       │       ├── kthread_create_on_cpu(smpboot_thread_fn, td, cpu, name)
        │       │       ├── kthread_set_per_cpu(tsk, cpu)
        │       │       ├── kthread_park(tsk)            ← يبدأ parked
        │       │       ├── get_task_struct(tsk)
        │       │       ├── *per_cpu_ptr(ht->store, cpu) = tsk
        │       │       └── ht->create(cpu)  [اختياري، بعد ما يتـ park فعلاً]
        │       │
        │       └── smpboot_unpark_thread(ht, cpu)
        │               └── kthread_unpark(tsk)  ← يبدأ يشتغل
        │
        ├── list_add(&ht->list, &hotplug_threads)
        ├── mutex_unlock(smpboot_threads_lock)
        └── cpus_read_unlock()

━━━━━━━━━━━━━━━━ [CPU يجي ONLINE لاحقاً] ━━━━━━━━━━━━━━━━━
smpboot_create_threads(cpu)       ← بيتكال من CPU hotplug path
        └── __smpboot_create_thread(ht, cpu)  (لكل ht في القائمة)

smpboot_unpark_threads(cpu)       ← بعدين بيتكال
        └── smpboot_unpark_thread(ht, cpu)  (forward order)

━━━━━━━━━━━━━━━━ [CPU يروح OFFLINE] ━━━━━━━━━━━━━━━━━━━━━
smpboot_park_threads(cpu)
        └── smpboot_park_thread(ht, cpu)  (reverse order ← مهم للـ ordering)
                └── kthread_park(tsk)
                        → thread يحس بـ kthread_should_park()
                        → ht->park(cpu)  [اختياري]
                        → status = HP_THREAD_PARKED
                        → kthread_parkme()

━━━━━━━━━━━━━━━━ [UNREGISTER] ━━━━━━━━━━━━━━━━━━━━━━━━━━━
smpboot_unregister_percpu_thread(ht)
        ├── cpus_read_lock()
        ├── mutex_lock(smpboot_threads_lock)
        ├── list_del(&ht->list)
        ├── smpboot_destroy_threads(ht)
        │       └── for_each_possible_cpu(cpu):
        │               ├── kthread_stop_put(tsk)
        │               │     → thread يحس بـ kthread_should_stop()
        │               │     → ht->cleanup(cpu, online)  [اختياري]
        │               │     → kfree(td)
        │               └── *per_cpu_ptr(ht->store, cpu) = NULL
        ├── mutex_unlock(smpboot_threads_lock)
        └── cpus_read_unlock()
```

---

### 4. Call Flow Diagrams

#### 4.1 — Main Loop: `smpboot_thread_fn`

```
smpboot_thread_fn(td)
  │
  └── loop forever:
        │
        ├── set_current_state(TASK_INTERRUPTIBLE)
        ├── preempt_disable()
        │
        ├─[kthread_should_stop?]─YES──▶ cleanup(cpu, online)? → kfree(td) → return 0
        │
        ├─[kthread_should_park?]─YES──▶ if ACTIVE: park(cpu) → PARKED
        │                                kthread_parkme()  ← ينام هنا
        │                                continue (لما يصحى يشوف stop مرة تانية)
        │
        ├─[status == HP_THREAD_NONE]──▶ setup(cpu)? → status=ACTIVE → continue
        │
        ├─[status == HP_THREAD_PARKED]─▶ unpark(cpu)? → status=ACTIVE → continue
        │
        └─[ACTIVE]:
              ├─[thread_should_run(cpu) == 0]──▶ preempt_enable_no_resched() → schedule()
              └─[thread_should_run(cpu) != 0]──▶ set RUNNING → thread_fn(cpu)
```

#### 4.2 — Registration Flow

```
smpboot_register_percpu_thread(ht)
  │
  ├── cpus_read_lock()
  ├── mutex_lock(&smpboot_threads_lock)
  │
  └── for_each_online_cpu(cpu):
        │
        ├── __smpboot_create_thread(ht, cpu)
        │     │
        │     ├── already exists? → return 0 (idempotent)
        │     ├── kzalloc_node(smpboot_thread_data, node=cpu_to_node(cpu))
        │     │     td->cpu = cpu
        │     │     td->ht  = ht
        │     │
        │     ├── kthread_create_on_cpu(smpboot_thread_fn, td, cpu, ht->thread_comm)
        │     │     → kernel/kthread.c: ينشئ task_struct مربوط بالـ CPU
        │     │
        │     ├── kthread_set_per_cpu(tsk, cpu)
        │     ├── kthread_park(tsk)         ← يبدأ في حالة parked
        │     ├── get_task_struct(tsk)      ← refcount+1
        │     ├── store[cpu] = tsk
        │     │
        │     └── ht->create(cpu)?
        │           wait_task_inactive(tsk, TASK_PARKED)  ← ضمان
        │           ht->create(cpu)
        │
        └── smpboot_unpark_thread(ht, cpu)
              └── !selfparking → kthread_unpark(tsk)
                    → الـ thread يصحى → loop → setup() → ACTIVE
```

#### 4.3 — CPU Offline Flow

```
[CPU hotplug: CPU N going OFFLINE]
        │
        ▼
smpboot_park_threads(cpu=N)
        │
        ├── mutex_lock(&smpboot_threads_lock)
        ├── list_for_each_entry_REVERSE(cur, &hotplug_threads, list)
        │       └── smpboot_park_thread(cur, N)
        │             └── kthread_park(tsk)
        │
        └── mutex_unlock

   [داخل سياق كل thread]
        │
   smpboot_thread_fn detects kthread_should_park()
        ├── if status == HP_THREAD_ACTIVE:
        │     ht->park(cpu)      ← park callback
        │     td->status = HP_THREAD_PARKED
        └── kthread_parkme()    ← ينام
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks المستخدمة

| الـ Lock | النوع | يحمي ماذا؟ |
|----------|-------|------------|
| `smpboot_threads_lock` | `struct mutex` | قائمة `hotplug_threads` + عمليات create/destroy/park/unpark |
| `cpu_hotplug_lock` (عبر `cpus_read_lock/unlock`) | `percpu_rwsem` | يمنع تغيير CPU topology (online/offline) أثناء register/unregister |
| `preempt_disable` داخل `smpboot_thread_fn` | preemption guard | يضمن إن الـ `thread_should_run` بيتكال على نفس الـ CPU بدون interrupt |

#### ترتيب الـ Locks (Lock Ordering)

```
[أعلى في الـ hierarchy → يُأخد أولاً]

  cpu_hotplug_lock  (cpus_read_lock)
        ↓
  smpboot_threads_lock  (mutex)
        ↓
  preempt_disable  (في thread loop فقط)
```

**مهم:** لازم دايماً تأخد `cpus_read_lock` قبل `smpboot_threads_lock` — الكود في `smpboot_register_percpu_thread` و `smpboot_unregister_percpu_thread` بيطبق ده بالظبط.

#### ليه الـ park بـ reverse order؟

```c
list_for_each_entry_reverse(cur, &hotplug_threads, list)
    smpboot_park_thread(cur, cpu);
```

عشان لو في dependencies بين الـ threads (مثلاً thread B يعتمد على thread A)، التسجيل بيحطهم بالترتيب (A ثم B)، فالإيقاف بيكون عكسي (B أول ثم A) — نفس مبدأ destructor ordering.

#### الـ `selfparking` Flag

لو الـ thread بيعمل park لنفسه (مثلاً watchdog اللي بيراقب الـ CPU قبل ما يروح offline)، بيتحدد `selfparking = true` وبكده:
- الـ core مش بيعمل `kthread_park/unpark` تلقائياً
- الـ thread نفسه مسؤول عن استدعاء `kthread_parkme()` في الوقت الصح

#### حماية الـ `per_cpu store`

الـ `*per_cpu_ptr(ht->store, cpu)` بيتكتب فيه بس لما الـ `smpboot_threads_lock` مأخود — ده بيضمن إن ما فيش race بين create وdestroy.

---

### ملخص العلاقات

```
[Driver/Subsystem]
      │  يعمل ويملأ
      ▼
struct smp_hotplug_thread    ──────────────────────────────────┐
  │  (descriptor ثابت، عمره مع الـ module)                     │
  │                                                             │
  │ list ──▶ hotplug_threads global list                        │
  │                                                             │
  │ store[cpu] ──▶ task_struct (kthread per CPU)                │
  │                    │                                        │
  │                    │ kthread data pointer                   │
  │                    ▼                                        │
  │              smpboot_thread_data                            │
  │                ├── cpu                                      │
  │                ├── status  (NONE/ACTIVE/PARKED)             │
  │                └── ht ──────────────────────────────────────┘
  │
  └── callbacks: thread_should_run → thread_fn
                 create → setup → [park ↔ unpark] → cleanup
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ API — Cheatsheet

#### Functions العامة (Exported)

| Function | الملف | الـ Export | الغرض |
|---|---|---|---|
| `smpboot_register_percpu_thread` | `kernel/smpboot.c` | `EXPORT_SYMBOL_GPL` | تسجيل thread descriptor وإنشاء threads على كل CPU أونلاين |
| `smpboot_unregister_percpu_thread` | `kernel/smpboot.c` | `EXPORT_SYMBOL_GPL` | إلغاء التسجيل وإيقاف كل الـ threads |

#### Functions الداخلية (Internal)

| Function | الغرض |
|---|---|
| `smpboot_thread_fn` | الـ main loop بتاع كل percpu thread |
| `__smpboot_create_thread` | إنشاء thread واحد على CPU معين |
| `smpboot_create_threads` | إنشاء threads لكل registered descriptors على CPU واحد |
| `smpboot_unpark_thread` | تحرير (unpark) thread واحد على CPU |
| `smpboot_unpark_threads` | تحرير كل threads على CPU معين |
| `smpboot_park_thread` | إيقاف مؤقت (park) thread واحد على CPU |
| `smpboot_park_threads` | إيقاف مؤقت كل threads على CPU معين |
| `smpboot_destroy_threads` | إيقاف نهائي وتدمير كل threads لـ descriptor معين |

#### Functions الـ Idle Thread (CONFIG_GENERIC_SMP_IDLE_THREAD)

| Function | الغرض |
|---|---|
| `idle_thread_get` | جلب idle task_struct لـ CPU معين |
| `idle_thread_set_boot_cpu` | ربط current task بـ boot CPU كـ idle thread |
| `idle_init` | إنشاء idle thread لـ CPU معين لو مش موجود |
| `idle_threads_init` | تهيئة idle threads لكل CPUs possible ما عدا boot CPU |

---

### Struct المركزية

```c
struct smp_hotplug_thread {
    struct task_struct * __percpu *store;      /* per-cpu pointer للـ task */
    struct list_head               list;        /* ربط في hotplug_threads list */
    int  (*thread_should_run)(unsigned int cpu);/* هل يشتغل؟ — preempt disabled */
    void (*thread_fn)(unsigned int cpu);        /* الـ work الفعلي */
    void (*create)(unsigned int cpu);           /* optional — بعد إنشاء الـ thread */
    void (*setup)(unsigned int cpu);            /* optional — أول مرة يشتغل */
    void (*cleanup)(unsigned int cpu, bool online); /* optional — عند الإيقاف */
    void (*park)(unsigned int cpu);             /* optional — عند offline */
    void (*unpark)(unsigned int cpu);           /* optional — عند online */
    bool selfparking;                           /* الـ thread بيعمل park لنفسه */
    const char *thread_comm;                    /* اسم الـ thread */
};
```

الـ `store` هو per-cpu pointer array — كل CPU عنده pointer لـ `task_struct` بتاعه.
الـ `list` بيربط الـ descriptor في الـ global `hotplug_threads` list.

---

### Group 1: Registration & Unregistration

الغرض من الـ group ده هو إدارة دورة حياة الـ percpu hotplug threads — من لحظة التسجيل لحد الإلغاء. كل subsystem (زي migration, watchdog, ksoftirqd) بيسجل descriptor واحد بس، والـ smpboot layer بيتكفل بإنشاء thread على كل CPU.

---

#### `smpboot_register_percpu_thread`

```c
int smpboot_register_percpu_thread(struct smp_hotplug_thread *plug_thread);
```

بتسجل الـ `smp_hotplug_thread` descriptor وبتنشئ threads على كل CPU أونلاين في نفس الوقت. بعد الإنشاء، بتعمل unpark للـ threads علشان تبدأ تشتغل. لو أي CPU فشل في الإنشاء، بتعمل rollback وبتدمر كل اللي تم إنشاؤه.

**Parameters:**
- `plug_thread` — pointer لـ descriptor كامل فيه callbacks والـ store وغيره

**Return value:**
- `0` — نجاح
- Error code سالب — فشل في إنشاء thread على أي CPU

**Key details:**
- بتاخد `cpus_read_lock()` علشان تمنع hotplug events أثناء التسجيل
- بتاخد `smpboot_threads_lock` (mutex) علشان تحمي الـ `hotplug_threads` list
- الترتيب مهم: إنشاء ثم unpark ثم إضافة للـ list
- لو فشلت: `smpboot_destroy_threads` بيعمل cleanup لكل ما تم إنشاؤه
- **Caller context:** Process context فقط، ممكن تنام (mutex + GFP_KERNEL)

**Pseudocode flow:**
```
cpus_read_lock()
mutex_lock(smpboot_threads_lock)
for each online cpu:
    ret = __smpboot_create_thread(plug_thread, cpu)
    if ret != 0:
        smpboot_destroy_threads(plug_thread)
        goto out
    smpboot_unpark_thread(plug_thread, cpu)
list_add(&plug_thread->list, &hotplug_threads)
out:
mutex_unlock(smpboot_threads_lock)
cpus_read_unlock()
return ret
```

---

#### `smpboot_unregister_percpu_thread`

```c
void smpboot_unregister_percpu_thread(struct smp_hotplug_thread *plug_thread);
```

بتوقف وبتدمر كل threads المرتبطة بالـ descriptor على كل possible CPUs — حتى اللي offline. بتشيل الـ descriptor من الـ global list علشان hotplug events جديدة ما تلاقيه.

**Parameters:**
- `plug_thread` — نفس الـ descriptor اللي تم تسجيله

**Return value:** `void`

**Key details:**
- بتاخد `cpus_read_lock()` و `smpboot_threads_lock`
- `list_del` أول حاجة — علشان أي CPU hotplug event جديدة ما تحاول تتعامل معاه
- بتشتغل على `for_each_possible_cpu` مش `online` — لأن ممكن يكون في threads parked على offline CPUs
- بتستخدم `kthread_stop_put` — ده stop + put_task_struct في خطوة واحدة
- **Caller context:** Process context (module exit أو subsystem teardown)

---

### Group 2: Thread Lifecycle Management (Internal)

الـ functions دي هي اللي بتتكلم مع الـ kthread API مباشرةً. الـ hotplug callbacks (CPU_UP/CPU_DOWN) بتستدعيها.

---

#### `__smpboot_create_thread`

```c
static int __smpboot_create_thread(struct smp_hotplug_thread *ht, unsigned int cpu);
```

بتنشئ thread واحد على CPU معين. لو الـ thread موجود خلاص (تم إنشاؤه قبل كده)، بترجع `0` فوراً بدون ما تعمل حاجة. بتعمل park للـ thread بعد الإنشاء مباشرةً علشان ما يبدأش قبل ما يكون الـ CPU جاهز.

**Parameters:**
- `ht` — الـ descriptor
- `cpu` — رقم الـ CPU المستهدف

**Return value:**
- `0` — نجاح أو thread موجود خلاص
- `-ENOMEM` — فشل alloc الـ `smpboot_thread_data`
- Error code من `kthread_create_on_cpu`

**Key details:**
- الـ `smpboot_thread_data` بيتعمل alloc على نفس الـ NUMA node بتاع الـ CPU (كفاءة)
- `kthread_create_on_cpu` بيربط الـ thread بـ CPU معين (affinity مباشرة)
- `kthread_set_per_cpu(tsk, cpu)` بيعلم الـ scheduler إن ده percpu thread
- بعد `kthread_park`: بيستنى `wait_task_inactive(tsk, TASK_PARKED)` قبل ما ينادي `ht->create` — ضروري علشان migration thread callback بيتطلب إن الـ task مش على runqueue
- `get_task_struct(tsk)` — زيادة refcount لأن الـ pointer محفوظ في `store`

**Pseudocode flow:**
```
tsk = *per_cpu_ptr(ht->store, cpu)
if tsk != NULL: return 0  // already created

td = kzalloc_node(sizeof(*td), GFP_KERNEL, cpu_to_node(cpu))
if !td: return -ENOMEM

td->cpu = cpu; td->ht = ht

tsk = kthread_create_on_cpu(smpboot_thread_fn, td, cpu, ht->thread_comm)
if IS_ERR(tsk):
    kfree(td)
    return PTR_ERR(tsk)

kthread_set_per_cpu(tsk, cpu)
kthread_park(tsk)
get_task_struct(tsk)
*per_cpu_ptr(ht->store, cpu) = tsk

if ht->create:
    wait_task_inactive(tsk, TASK_PARKED)  // WARN_ON if fails
    ht->create(cpu)

return 0
```

---

#### `smpboot_create_threads`

```c
int smpboot_create_threads(unsigned int cpu);
```

بتمشي على كل registered descriptors في `hotplug_threads` list وبتنشئ thread لكل واحد على الـ CPU الجديد. بتتنادى من الـ hotplug framework لما CPU بيجي online.

**Parameters:**
- `cpu` — الـ CPU الجديد اللي جاي online

**Return value:**
- `0` — نجاح
- Error code — من أول descriptor فشل

**Key details:**
- بتاخد `smpboot_threads_lock` فقط (مش `cpus_read_lock` لأن الـ caller بيمسكه)
- لو فشل descriptor واحد، بتوقف فوراً (fail-fast)
- **Caller context:** CPU hotplug callback (CPUHP_BP_PREPARE_DYN state)

---

#### `smpboot_unpark_thread` / `smpboot_unpark_threads`

```c
static void smpboot_unpark_thread(struct smp_hotplug_thread *ht, unsigned int cpu);
int smpboot_unpark_threads(unsigned int cpu);
```

**الـ `smpboot_unpark_thread`:** بتعمل `kthread_unpark` للـ thread اللي على CPU معين — بس لو `selfparking` مش متفعل. لو الـ thread بيدير parking نفسه، مش محتاجة تتدخل.

**الـ `smpboot_unpark_threads`:** بتمشي forward على الـ list وبتعمل unpark لكل threads على CPU معين. بتتنادى لما CPU بييجي online.

**Key details:**
- الـ `selfparking = true` معناه إن الـ thread نفسه مسؤول عن الـ parking — زي الـ watchdog thread
- **Caller context:** CPU hotplug callback (CPU coming online)

---

#### `smpboot_park_thread` / `smpboot_park_threads`

```c
static void smpboot_park_thread(struct smp_hotplug_thread *ht, unsigned int cpu);
int smpboot_park_threads(unsigned int cpu);
```

**الـ `smpboot_park_thread`:** بتعمل `kthread_park` للـ thread لو موجود ومش selfparking.

**الـ `smpboot_park_threads`:** بتمشي **reverse** على الـ list علشان تعكس ترتيب الـ unpark. ده مهم علشان الـ teardown يكون عكس الـ setup بالظبط. بتتنادى لما CPU بييجي offline.

**Key details:**
- الـ reverse iteration في park vs. forward في unpark ده design مقصود (LIFO teardown)
- **Caller context:** CPU hotplug callback (CPU going offline)

---

#### `smpboot_destroy_threads`

```c
static void smpboot_destroy_threads(struct smp_hotplug_thread *ht);
```

بتدمر كل threads بتاعة descriptor معين على كل possible CPUs. بتستخدم `kthread_stop_put` اللي بيعمل `kthread_stop` + `put_task_struct` في خطوة واحدة — علشان تحرر الـ refcount اللي `get_task_struct` عمله في وقت الإنشاء.

**Key details:**
- بتشتغل على `for_each_possible_cpu` مش `online` — لأن ممكن في threads parked على offline CPUs
- بعد الـ stop: بتعمل `*per_cpu_ptr(ht->store, cpu) = NULL` علشان تمنع dangling pointers
- **Caller context:** بتتنادى من `smpboot_unregister_percpu_thread` أو كـ rollback من `smpboot_register_percpu_thread`

---

### Group 3: الـ Main Thread Loop

ده أهم function في الـ subsystem — هو اللي بيشتغل جوا كل percpu thread.

---

#### `smpboot_thread_fn`

```c
static int smpboot_thread_fn(void *data);
```

الـ main loop بتاع كل percpu hotplug thread. بيتحكم في الـ state machine بتاع الـ thread — من الإنشاء لحد الـ cleanup. بيتحقق من stop/park conditions أول، وبعدين بيعمل state transitions (NONE → ACTIVE → PARKED)، وبعدين بيشغل الـ work لو مطلوب.

**Parameters:**
- `data` — pointer لـ `smpboot_thread_data` (فيه cpu رقم + ht pointer + status)

**Return value:**
- `0` — الـ thread انتهى بشكل طبيعي (بعد `kthread_should_stop`)

**Key details:**
- بيبدأ loop بـ `set_current_state(TASK_INTERRUPTIBLE)` + `preempt_disable()` — علشان يتحقق من conditions بدون race
- **Stop path:** بيعمل cleanup لو الـ status مش `HP_THREAD_NONE`، وبعدين بيحرر الـ `td` memory وبيرجع
- **Park path:** بيعمل `ht->park` بس لو status = `HP_THREAD_ACTIVE`، وبيتحقق من `BUG_ON(td->cpu != smp_processor_id())` — الـ thread لازم يشتغل على نفس الـ CPU
- **NONE → ACTIVE transition:** بيستدعي `ht->setup` مرة واحدة بس (أول مرة)
- **PARKED → ACTIVE transition:** بيستدعي `ht->unpark` وبيغير الـ status
- **Work loop:** لو `thread_should_run` رجع false → بيعمل `schedule()`. لو true → بيشغل `thread_fn` مباشرةً
- الـ `preempt_disable` + فحص الـ conditions + `preempt_enable` ده pattern علشان يمنع race مع hotplug events

**State Machine:**

```
                  [thread created]
                        |
                        v
               HP_THREAD_NONE
                        |
                  ht->setup(cpu)
                        |
                        v
               HP_THREAD_ACTIVE  <----------+
                   |        |               |
           should_run?    park?         unpark
            yes  |  no     |               |
                 |  |      v               |
           thread_fn  schedule  HP_THREAD_PARKED
                              [ht->park called]
```

**Pseudocode flow:**
```
while (1):
    set_current_state(TASK_INTERRUPTIBLE)
    preempt_disable()

    if kthread_should_stop():
        set_running(); preempt_enable()
        if ht->cleanup && status != NONE:
            ht->cleanup(cpu, cpu_online(cpu))
        kfree(td)
        return 0

    if kthread_should_park():
        set_running(); preempt_enable()
        if ht->park && status == ACTIVE:
            ht->park(cpu)
            status = PARKED
        kthread_parkme()
        continue

    BUG_ON(cpu != smp_processor_id())

    switch status:
        case NONE:
            set_running(); preempt_enable()
            ht->setup(cpu) if exists
            status = ACTIVE
            continue
        case PARKED:
            set_running(); preempt_enable()
            ht->unpark(cpu) if exists
            status = ACTIVE
            continue

    if !ht->thread_should_run(cpu):
        preempt_enable_no_resched()
        schedule()
    else:
        set_running(); preempt_enable()
        ht->thread_fn(cpu)
```

---

### Group 4: Idle Thread Management

الـ functions دي شغالة بس لو `CONFIG_GENERIC_SMP_IDLE_THREAD` متفعل. الـ idle thread هو الـ task اللي الـ CPU بيشغله لما مفيش حاجة تانية تشتغل.

---

#### `idle_thread_get`

```c
struct task_struct *idle_thread_get(unsigned int cpu);
```

بترجع الـ `task_struct` بتاع الـ idle thread لـ CPU معين. بتتنادى من الـ hotplug code لما CPU بييجي online علشان يلاقي الـ idle task جاهز.

**Parameters:**
- `cpu` — رقم الـ CPU

**Return value:**
- Pointer لـ `task_struct` — لو موجود
- `ERR_PTR(-ENOMEM)` — لو مش موجود (ما اتعملش init بعد)

**Key details:**
- مافيش لocking — الـ caller مسؤول عن ضمان التزامن
- **Caller context:** CPU hotplug bringup path

---

#### `idle_thread_set_boot_cpu`

```c
void __init idle_thread_set_boot_cpu(void);
```

بتسجل الـ `current` task (اللي بيشتغل على boot CPU أثناء الـ init) كـ idle thread لـ boot CPU. بتتنادى مرة واحدة بس أثناء الـ kernel init.

**Key details:**
- `__init` — بيتحذف بعد الـ boot
- بتستخدم `smp_processor_id()` علشان تعرف رقم الـ boot CPU

---

#### `idle_init`

```c
static __always_inline void idle_init(unsigned int cpu);
```

بتنشئ idle thread لـ CPU معين لو مش موجود. بتستخدم `fork_idle(cpu)` اللي بيعمل fork من init task مع affinity للـ CPU المحدد.

**Parameters:**
- `cpu` — رقم الـ CPU

**Key details:**
- `__always_inline` — علشان ماتعملش call overhead في الـ init path
- لو `fork_idle` فشل، بتطبع error وبتكمل (مش بتوقف الـ boot)
- **Caller context:** `idle_threads_init` فقط

---

#### `idle_threads_init`

```c
void __init idle_threads_init(void);
```

بتعمل init لـ idle threads على كل possible CPUs ما عدا boot CPU (اللي اتعمله `idle_thread_set_boot_cpu` قبل كده). بتتنادى مرة واحدة من `smp_init` أثناء الـ kernel boot.

**Key details:**
- `__init` — code بيتحذف بعد الـ boot
- بتستثني boot CPU من الـ loop لأنه معمول بالفعل
- **Caller context:** Single-threaded init phase (قبل إطلاق الـ secondary CPUs)

---

### Global State

```c
/* الـ list اللي فيه كل registered descriptors */
static LIST_HEAD(hotplug_threads);

/* بيحمي الـ list من concurrent access */
static DEFINE_MUTEX(smpboot_threads_lock);

/* الـ idle threads (لو CONFIG_GENERIC_SMP_IDLE_THREAD) */
static DEFINE_PER_CPU(struct task_struct *, idle_threads);
```

الـ `smpboot_threads_lock` بيتاخد دايماً **بعد** `cpus_read_lock` علشان يتجنب deadlock مع الـ hotplug framework.

---

### Thread Status Enum

```c
enum {
    HP_THREAD_NONE   = 0,  /* thread اتعمل لكن لسه ماعملش setup */
    HP_THREAD_ACTIVE,      /* thread شغال وعمل setup */
    HP_THREAD_PARKED,      /* thread متوقف مؤقتاً (CPU offline) */
};
```

---

### Locking Order Summary

| Lock | من وين بياخده | الغرض |
|---|---|---|
| `cpus_read_lock()` | `register` / `unregister` | يمنع hotplug events أثناء التعديل |
| `smpboot_threads_lock` (mutex) | كل function بيعدل في الـ list | يحمي `hotplug_threads` list |
| `preempt_disable` (في thread loop) | `smpboot_thread_fn` | يمنع race عند فحص stop/park/run conditions |

---

### مثال عملي — ksoftirqd

الـ ksoftirqd بيستخدم الـ smpboot API بالشكل ده:

```c
static struct smp_hotplug_thread softirq_threads = {
    .store          = &ksoftirqd,          /* per-cpu task_struct pointer */
    .thread_should_run = ksoftirqd_should_run, /* هل في softirqs؟ */
    .thread_fn      = run_ksoftirqd,       /* شغل الـ softirqs */
    .thread_comm    = "ksoftirqd/%u",      /* الاسم في ps */
};

/* في __init */
smpboot_register_percpu_thread(&softirq_threads);
```

الـ smpboot layer بييجي بالـ `ksoftirqd/0`, `ksoftirqd/1`, ... وبيتكفل بكل الـ lifecycle management — hotplug، park/unpark، cleanup — من غير ما الـ softirq code يعرف حاجة عن كل ده.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem ده هو **smpboot** — المسؤول عن إنشاء وإدارة الـ per-CPU hotplug threads اللي بتتعامل مع CPU online/offline. المشاكل هنا بتظهر في أوقات الـ CPU hotplug، أو لما thread مش بيتشغل على الـ CPU الصح، أو لما الـ park/unpark cycle بتتعطل.

---

### Software Level

#### 1. Debugfs Entries

الـ smpboot نفسه مش بيعمل entries في debugfs مباشرةً، لكن الـ kthread subsystem والـ CPU hotplug بيعملوا:

```bash
# شوف كل الـ kthreads اللي شغالة per-CPU
ls /sys/kernel/debug/sched/

# الـ CPU hotplug state machine debug
cat /sys/kernel/debug/cpu_hotplug_state 2>/dev/null || echo "not available"

# ftrace للـ hotplug events — الأهم هنا
ls /sys/kernel/debug/tracing/events/cpuhp/
```

#### 2. Sysfs Entries

```bash
# حالة كل CPU — online/offline
cat /sys/devices/system/cpu/cpu*/online

# عدد الـ CPUs المتاحة
cat /sys/devices/system/cpu/possible
cat /sys/devices/system/cpu/present
cat /sys/devices/system/cpu/online

# الـ hotplug state لكل CPU
cat /sys/devices/system/cpu/cpu1/hotplug/state
cat /sys/devices/system/cpu/cpu1/hotplug/fail

# مثال على output
# /sys/devices/system/cpu/cpu1/hotplug/state → 214  (CPUHP_ONLINE)
# /sys/devices/system/cpu/cpu1/hotplug/fail  → -1   (no failure)
```

#### 3. Ftrace — Tracepoints المهمة

```bash
# فعّل الـ CPU hotplug events
echo 1 > /sys/kernel/debug/tracing/events/cpuhp/enable

# فعّل kthread events لمتابعة park/unpark
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_kthread_stop/enable
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_kthread_stop_ret/enable
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_wakeup/enable

# تابع thread_fn calls باستخدام function tracer
echo function > /sys/kernel/debug/tracing/current_tracer
echo smpboot_thread_fn > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# أو استخدم function_graph لرؤية call tree كامل
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo smpboot_thread_fn > /sys/kernel/debug/tracing/set_graph_function
cat /sys/kernel/debug/tracing/trace
```

مثال على output لما CPU بيتوقف:

```
     migration/1-21    [001] d...  123.456: cpuhp_enter: cpu: 1 target: 0 step: CPUHP_AP_SMPBOOT_THREADS
     migration/1-21    [001] d...  123.457: cpuhp_exit:  cpu: 1 state: 0 step: CPUHP_AP_SMPBOOT_THREADS ret: 0
```

#### 4. Printk / Dynamic Debug

```bash
# فعّل dynamic debug للـ smpboot module
echo 'file kernel/smpboot.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ kthread بالكامل
echo 'file kernel/kthread.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف اللي فعّال دلوقتي
cat /sys/kernel/debug/dynamic_debug/control | grep smpboot

# أو عن طريق kernel cmdline
# dyndbg="file kernel/smpboot.c +p"
```

الـ messages اللي هتظهر في dmesg:

```
[  5.123] SMP: fork_idle() failed for CPU 3        ← idle thread فشل
[  5.456] CPU 2 is now offline                     ← hotplug event
```

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_DEBUG_HOTPLUG_CPU0` | تقدر تعمل offline لـ CPU0 للاختبار |
| `CONFIG_HOTPLUG_CPU` | لازم يكون enabled عشان الـ smpboot threads تشتغل |
| `CONFIG_GENERIC_SMP_IDLE_THREAD` | بيفعّل `idle_threads_init()` |
| `CONFIG_DEBUG_PREEMPT` | بيكتشف preemption violations في `thread_should_run` |
| `CONFIG_LOCKDEP` | بيتتبع الـ mutex `smpboot_threads_lock` |
| `CONFIG_PROVE_LOCKING` | بيكتشف deadlocks في الـ hotplug lock chain |
| `CONFIG_KTHREAD_DEBUG` | debug للـ kthread park/unpark states |
| `CONFIG_SCHEDSTATS` | إحصائيات scheduling للـ smpboot threads |
| `CONFIG_SCHED_DEBUG` | `/proc/sched_debug` يبين حالة الـ threads |

#### 6. أدوات خاصة بالـ Subsystem

```bash
# شوف كل الـ smpboot threads الشغالة دلوقتي
ps aux | grep -E 'ksoftirqd|migration|kworker|watchdog|cpuhp'

# مثال output:
# root         14  0.0  0.0      0     0 ?  S  Jan01   0:00 [ksoftirqd/0]
# root         18  0.0  0.0      0     0 ?  S  Jan01   0:00 [migration/0]
# root         21  0.0  0.0      0     0 ?  S  Jan01   0:00 [cpuhp/0]

# شوف الـ CPU affinity للـ thread — لازم يكون pinned على CPU واحد
taskset -p $(pgrep -f 'migration/1')
# output: pid 18's current affinity mask: 2  (binary: 10 → CPU1)

# شوف الـ thread state
cat /proc/$(pgrep -f 'migration/1')/status | grep -E 'State|Cpus_allowed'
# State:  S (sleeping)      ← parked في TASK_PARKED
# Cpus_allowed: 00000002    ← CPU1 بس

# /proc/sched_debug لو CONFIG_SCHED_DEBUG مفعّل
cat /proc/sched_debug | grep -A5 migration
```

#### 7. جدول رسائل الـ Error الشائعة

| رسالة في dmesg | المعنى | الحل |
|---|---|---|
| `SMP: fork_idle() failed for CPU N` | `fork_idle()` رجع error لما كان بيعمل idle thread | تأكد إن الـ memory مش full، و`CONFIG_GENERIC_SMP_IDLE_THREAD` مفعّل |
| `WARNING: CPU: N PID: M at kernel/smpboot.c:124` | الـ `BUG_ON(td->cpu != smp_processor_id())` اتفجر — الـ thread شغّال على CPU غلط | race condition في الـ CPU affinity، راجع `kthread_set_per_cpu()` calls |
| `kthread_create_on_cpu failed` | فشل إنشاء الـ per-CPU thread — ممكن memory أو CPU offline | راجع `dmesg` للـ `-ENOMEM`، وتأكد إن الـ CPU online قبل الـ register |
| `WARNING: at kernel/smpboot.c:202` | `wait_task_inactive()` رجع 0 — الـ thread مش وصل لـ TASK_PARKED | الـ thread مش بيستجاب للـ park signal، ابحث عن busy loop في `thread_fn` |
| `CPU N is now offline` + hang | الـ park function بتفضل blocked | الـ `park()` callback بتاعتك محتاج يرجع بسرعة بدون wait |
| `BUG: scheduling while atomic` | حد استدعى schedule() وهو في preempt-disabled context داخل `thread_should_run` | الـ callback ده بيتنادى مع preemption disabled، متعملش blocking ops |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في __smpboot_create_thread() — بعد kthread_create_on_cpu */
tsk = kthread_create_on_cpu(smpboot_thread_fn, td, cpu, ht->thread_comm);
if (IS_ERR(tsk)) {
    /* هنا عارف إن الـ thread creation فشل */
    WARN(1, "smpboot: failed to create thread for CPU%u: %ld\n",
         cpu, PTR_ERR(tsk));
    dump_stack(); /* عشان نشوف مين طلب الـ register */
}

/* في smpboot_thread_fn() — تحقق من CPU pinning */
WARN_ON_ONCE(td->cpu != smp_processor_id());
/* الأصلي BUG_ON — ممكن تحوله WARN_ON في dev kernel عشان مش fatal */

/* في park callback بتاعك — لو بتشك إن الـ state مش متزامن */
static void my_thread_park(unsigned int cpu)
{
    WARN_ON(!cpu_online(cpu)); /* المفروض CPU لسه online وقت الـ park */
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

```bash
# قارن الـ CPUs اللي الـ BIOS بيشوفها مع اللي الـ kernel شايفها
# ACPI MADT table بيديك عدد الـ CPUs الفعلي
dmesg | grep -E 'ACPI|APIC|CPU.*found'

# مثال:
# [    0.000] ACPI: 4 LAPIC NMI (acpi_id[0xff] high edge lint[0x1])
# [    0.000] x86: Booting SMP configuration:
# [    0.000] .... node  #0, CPUs:  #1 #2 #3

# تأكد إن عدد الـ CPUs في /sys يطابق
nproc --all  # عدد الـ CPUs الكلي (possible)
nproc        # عدد الـ CPUs الشغالة (online)

# لو في CPU مش بيظهر:
cat /sys/devices/system/cpu/possible  # 0-3
cat /sys/devices/system/cpu/present   # 0-3
cat /sys/devices/system/cpu/online    # 0,2-3  ← CPU1 offline مثلاً
```

#### 2. Register Dump Techniques

```bash
# Local APIC registers — مهمة لـ CPU bring-up
# بتحتاج /dev/mem أو devmem2

# LAPIC base عادةً على 0xFEE00000
devmem2 0xFEE00020 w   # LAPIC ID register
devmem2 0xFEE00030 w   # LAPIC Version register
devmem2 0xFEE00300 w   # ICR (Interrupt Command Register) — للـ IPI

# مثال output:
# /dev/mem opened.
# Memory mapped at address 0x7f8a2d3f2000.
# Read at address 0xFEE00020 (0x7f8a2d3f2020): 0x01000000
# → LAPIC ID = 1 (bits 24-31)

# MSR registers للـ CPU capabilities
rdmsr 0x1B    # IA32_APIC_BASE MSR
# output: c000000fee00900
# → bit 11 = 1: APIC enabled
# → bits 12-35: APIC base address = 0xFEE00000

# CPUID لمعرفة عدد الـ logical CPUs
cpuid -l 0x1 | grep -i 'logical'
# أو
python3 -c "import cpuinfo; print(cpuinfo.get_cpu_info()['count'])"
```

#### 3. Logic Analyzer / Oscilloscope Tips

لما بتشتغل على embedded system وعندك مشاكل في CPU bring-up:

```
الـ SMP Boot Sequence على wire:

CPU0 (BSP)                    CPU1 (AP)
    |                              |
    |── INIT IPI ────────────────>|
    |── SIPI (Start-up IPI) ────>|  ← measure pulse on LINT0
    |                              |── executes trampoline code
    |<── AP signals ready ─────── |  ← GPIO toggled in trampoline
    |                              |

نقاط القياس:
- LINT0/LINT1 pins على الـ CPU socket
- GPIO pin لو الـ AP bringup code بيقدر يكتب عليه
- JTAG: breakpoint عند smp_callin() في arch/x86/kernel/smpboot.c
```

**JTAG Debug:**
```bash
# لو عندك OpenOCD
openocd -f board/your_board.cfg
# في gdb:
target extended-remote :3333
hbreak smp_callin        # hardware breakpoint عند AP entry
continue
```

#### 4. Hardware Issues الشائعة → Kernel Log Patterns

| مشكلة في الـ Hardware | Pattern في dmesg |
|---|---|
| الـ AP مش بيرد على SIPI | `CPU1: Stuck ?! Send IPI to self` أو timeout في `do_boot_cpu()` |
| LAPIC معطوب أو مش enabled في BIOS | `APIC: Switch to symmetric I/O mode setup` فشل |
| CPU frequency scaling بتسبب مشكلة في timing | `hrtimer: interrupt took N ns` قيم عالية جداً |
| الـ CPU لا يدعم hotplug فعلياً | `CPU1 has no hotplug support` |
| Thermal shutdown أثناء الـ boot | `mce: [Hardware Error]: Machine check events logged` + CPU offline |
| NUMA node configuration غلط | `NUMA: node N configured` + smpboot thread على node غلط |

#### 5. Device Tree Debugging (للـ ARM/embedded)

```bash
# تحقق إن الـ DT بيعرّف الـ CPUs صح
dtc -I fs /proc/device-tree 2>/dev/null | grep -A20 'cpus {'

# مثال DT صحيح:
# cpus {
#     #address-cells = <1>;
#     #size-cells = <0>;
#     cpu@0 { device_type = "cpu"; reg = <0>; };
#     cpu@1 { device_type = "cpu"; reg = <1>; };
# }

# لو CPU مش ظاهر في /sys بس موجود في DT:
dmesg | grep -i 'cpu.*failed\|cpu.*error\|cpu.*unable'

# تحقق إن enable-method صح
dtc -I fs /proc/device-tree/cpus/cpu@1 | grep enable-method
# المفروض: enable-method = "psci"; أو "spin-table";

# PSCI debugging
dmesg | grep -i psci
# [    0.000] psci: probing for conduit method from ACPI.
# [    0.000] psci: PSCIv1.1 detected in firmware.

# لو الـ PSCI مش شغال صح، الـ smpboot threads مش هتتعمل للـ APs
cat /sys/devices/system/cpu/cpu1/hotplug/state
# لو الـ value صغيرة جداً → الـ AP مش وصل لـ CPUHP_ONLINE
```

---

### Practical Commands

#### جاهزة للـ Copy-Paste

```bash
# ===== 1. شوف كل الـ smpboot threads وحالتها =====
for pid in $(pgrep -f 'migration\|ksoftirqd\|kworker\|cpuhp\|watchdog'); do
    comm=$(cat /proc/$pid/comm 2>/dev/null)
    state=$(awk '/^State/{print $2}' /proc/$pid/status 2>/dev/null)
    cpu=$(awk '/^Cpus_allowed_list/{print $2}' /proc/$pid/status 2>/dev/null)
    echo "PID=$pid  COMM=$comm  STATE=$state  CPU=$cpu"
done

# مثال output:
# PID=14  COMM=ksoftirqd/0  STATE=S  CPU=0
# PID=18  COMM=migration/0  STATE=S  CPU=0
# PID=24  COMM=cpuhp/0      STATE=S  CPU=0
# PID=25  COMM=cpuhp/1      STATE=S  CPU=1

# ===== 2. جرب hotplug ووقت الـ threads =====
time echo 0 > /sys/devices/system/cpu/cpu1/online
time echo 1 > /sys/devices/system/cpu/cpu1/online
# لو الوقت > 1sec → في thread بياخد وقت طويل في park/unpark

# ===== 3. تتبع park/unpark cycle بـ ftrace =====
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo function_graph > current_tracer
echo 'smpboot_park_thread
smpboot_unpark_thread
smpboot_create_threads' > set_graph_function
echo 1 > tracing_on
echo 0 > /sys/devices/system/cpu/cpu1/online
echo 1 > /sys/devices/system/cpu/cpu1/online
echo 0 > tracing_on
cat trace | head -60
echo nop > current_tracer

# مثال output:
# 1)               |  smpboot_park_threads() {
# 1)   0.521 us    |    smpboot_park_thread();   /* migration */
# 1)   0.312 us    |    smpboot_park_thread();   /* ksoftirqd */
# 1) + 1.234 us    |  }

# ===== 4. كشف CPU على الـ wrong CPU =====
# شوف لو في thread بيشتغل مش على CPU بتاعه
cat /proc/$(pgrep migration/1)/status | grep -E 'Cpus_allowed|voluntary'
# Cpus_allowed: 00000002  ← لازم bit 1 بس (CPU1)
# voluntary_ctxt_switches: 1234  ← عدد الـ sleep cycles

# ===== 5. فحص LAPIC state (x86) =====
# تحتاج MSR tools مثبتة (apt install msr-tools)
modprobe msr
rdmsr -p 1 0x1B   # IA32_APIC_BASE للـ CPU1
# output: fee00900
# → hex 0xFEE00000 = APIC base (bits 12+)
# → bit 11 (0x800) = 1: APIC enabled
# → bit 8  (0x100) = 1: BSP flag (لو 0 على CPU1 → AP)

# ===== 6. مراقبة hotplug failures في الـ state machine =====
watch -n 1 'for cpu in /sys/devices/system/cpu/cpu[0-9]*; do
    state=$(cat $cpu/hotplug/state 2>/dev/null)
    fail=$(cat $cpu/hotplug/fail 2>/dev/null)
    echo "$(basename $cpu): state=$state fail=$fail"
done'

# output لما كل حاجة تمام:
# cpu0: state=214 fail=-1
# cpu1: state=214 fail=-1
# cpu2: state=214 fail=-1

# output لما في مشكلة:
# cpu1: state=136 fail=136
# → الـ CPU فشل عند step 136 في الـ hotplug state machine

# ===== 7. Dynamic debug للـ smpboot + kthread =====
echo 'file kernel/smpboot.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file kernel/kthread.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
dmesg -w | grep -E 'smpboot|kthread'
# +p = printk, +f = function name, +l = line number, +m = module name, +t = timestamp

# ===== 8. شوف الـ smpboot_threads_lock contention =====
# لازم CONFIG_LOCKSTAT=y
echo 1 > /proc/sys/kernel/lock_stat
echo 0 > /sys/devices/system/cpu/cpu2/online
echo 1 > /sys/devices/system/cpu/cpu2/online
cat /proc/lock_stat | grep -A5 smpboot_threads_lock
echo 0 > /proc/sys/kernel/lock_stat

# مثال output:
# smpboot_threads_lock:  con-bounces: 0  waittime-min: 0.5  ...
# → con-bounces عالية = contention = في threads كتير بتحاول lock

# ===== 9. تتبع thread_fn execution time =====
perf record -g -e sched:sched_switch \
    --filter 'next_comm~migration*' \
    -- sleep 5
perf report

# ===== 10. stack trace للـ smpboot thread عند hang =====
# لو CPU offline بيتعلق:
SYS_PID=$(pgrep cpuhp/0)
kill -SIGUSR1 $SYS_PID 2>/dev/null || echo "use sysrq instead"

# بديل: SysRq
echo t > /proc/sysrq-trigger  # اطبع stack traces لكل threads
dmesg | grep -A20 'migration/1'

# مثال output لما في hang:
# migration/1     D    0    18      2 0x00000000
# Call Trace:
#  schedule+0x74/0xa0
#  kthread_parkme+0x54/0x80    ← عالق في park
#  smpboot_thread_fn+0x98/0x140
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — thread بيموت بصمت عند CPU Hotplug

#### العنوان
**الـ smp_hotplug_thread بيتوقف من غير kernel panic على gateway صناعي**

#### السياق
شركة بتبني industrial gateway بناءً على **RK3562** (quad-core Cortex-A53). الـ gateway بيشتغل كـ Modbus-to-MQTT bridge. الـ firmware engineer كتب kernel module خاص بيعمل polling على UART بشكل دوري عبر per-CPU kthread مسجّل بـ `smpboot_register_percpu_thread()`.

#### المشكلة
لما المنتج بيشتغل تحت حِمل عالي وبيتعمله `echo 0 > /sys/devices/system/cpu/cpu3/online` لتوفير طاقة، الـ thread بيختفي من `ps aux` على CPU3 — لكن لما CPU3 بييجي أونلاين تاني، الـ thread مش بييجي معاه. الـ UART polling على CPU3 بيوقف خالص.

#### التحليل
الـ `struct smp_hotplug_thread` بيوفر callback اسمه `unpark`:

```c
struct smp_hotplug_thread {
    struct task_struct  * __percpu *store;  // per-CPU pointer للـ task
    void (*park)(unsigned int cpu);         // بيتنادى لما CPU يروح offline
    void (*unpark)(unsigned int cpu);       // بيتنادى لما CPU يرجع online
    bool selfparking;                       // لو true، الـ framework مش هو اللي بيعمل park
    ...
};
```

الـ engineer نسي يعمل implement للـ `unpark` callback. لما CPU3 بيرجع online، الـ kernel بيطلب `unpark` على الـ thread بتاعه — لكن لأن الـ pointer بـ `NULL`، المنطق اللي بيعمل re-activate الـ thread ميتنفذش صح. الـ `store` pointer على CPU3 موجود، لكن الـ task مش بيتعمله `wake_up_process()` بشكل صحيح.

```c
/* المشكلة: unpark = NULL، فالـ framework مش بيعمل الـ re-init اللازم */
static struct smp_hotplug_thread uart_poll_thread = {
    .store          = &uart_poll_task,
    .thread_fn      = uart_poll_fn,
    .thread_should_run = uart_should_run,
    .thread_comm    = "uart_poll/%u",
    /* .unpark مش موجود خالص */
};
```

#### الحل
إضافة `unpark` callback بيعمل re-initialize الـ UART state للـ CPU اللي رجع:

```c
static void uart_poll_unpark(unsigned int cpu)
{
    /* re-enable the UART channel assigned to this CPU */
    struct uart_chan *ch = per_cpu_ptr(&uart_channels, cpu);
    ch->active = true;
}

static struct smp_hotplug_thread uart_poll_thread = {
    .store             = &uart_poll_task,
    .thread_fn         = uart_poll_fn,
    .thread_should_run = uart_should_run,
    .park              = uart_poll_park,
    .unpark            = uart_poll_unpark,  /* الإضافة المهمة */
    .thread_comm       = "uart_poll/%u",
};
```

تحقق بـ:
```bash
# لما CPU3 يرجع online
echo 1 > /sys/devices/system/cpu/cpu3/online
ps aux | grep uart_poll
# المفروض يظهر uart_poll/3
```

#### الدرس المستفاد
الـ `park`/`unpark` callbacks في `smp_hotplug_thread` مش اختيارية لو الـ thread بيدير state خارجي. نسيانهم معناه resource leak صامت أو توقف وظيفة حرجة بدون أي error log.

---

### السيناريو 2: Android TV Box على Allwinner H616 — اسم الـ thread بيتعمل format غلط

#### العنوان
**الـ `thread_comm` بيظهر corrupted في `top` على TV box**

#### السياق
فريق بيطور Android TV box بناءً على **Allwinner H616** (quad-core Cortex-A53 + Mali-G31). الـ team كتبت kernel driver لـ hardware video decoder بيستخدم per-CPU threads لتوزيع الـ decode jobs. المنتج وصل للـ QA وبدأوا يشكوا إن اسم الـ thread في `htop` بيظهر كـ `vdec_work` بدل `vdec_work/0`, `vdec_work/1`...إلخ.

#### المشكلة
الـ thread name ميتعملش format صح بالـ CPU index. أحياناً بيظهر `vdec_work/%u` حرفياً أو بيظهر اسم غلط خالص.

#### التحليل
الـ `thread_comm` field في `smp_hotplug_thread` بيتعمله format مع `%u` للـ CPU number أوتوماتيك من الـ kernel عند `smpboot_register_percpu_thread()`. لكن الـ engineer حدد الاسم بشكل غلط:

```c
/* غلط: حجم الاسم بيتجاوز TASK_COMM_LEN وهو 16 بايت */
static struct smp_hotplug_thread vdec_thread = {
    .thread_comm = "video_decoder_worker/%u",  /* 24 حرف قبل الـ format = تجاوز */
    ...
};
```

الـ `TASK_COMM_LEN` في Linux هو 16 بايت بما فيهم الـ null terminator. لما الـ kernel بيعمل `snprintf(comm, TASK_COMM_LEN, ht->thread_comm, cpu)` الاسم بيتقطع.

```c
/* الـ kernel بيعمل كده داخلياً في kernel/smpboot.c */
snprintf(comm, TASK_COMM_LEN, ht->thread_comm, cpu);
/* "video_decoder_worker/0" → بيتقطع لـ "video_decoder_wo" */
```

#### الحل
تقصير اسم الـ thread عشان يتناسب مع الـ limit:

```c
static struct smp_hotplug_thread vdec_thread = {
    /* max اسم: 16 - 1 (null) - 2 (رقم CPU أقصاه 99) - 1 (/) = 12 حرف للاسم الأساسي */
    .thread_comm = "vdec_work/%u",  /* 9 حروف + /N = OK */
    ...
};
```

تحقق:
```bash
ps -eo pid,comm | grep vdec
# المفروض يظهر: vdec_work/0  vdec_work/1  vdec_work/2  vdec_work/3
cat /proc/$(pgrep -f vdec_work/0)/status | grep Name
```

#### الدرس المستفاد
الـ `thread_comm` في `smp_hotplug_thread` بيتعمله format مع CPU number داخل buffer بحجم `TASK_COMM_LEN` (16 بايت). أي اسم أساسي أطول من 12 حرف بيتقطع على systems رباعية الـ cores أو أكتر.

---

### السيناريو 3: IoT Sensor Hub على STM32MP1 — crash عند module unload

#### العنوان
**kernel panic عند `rmmod` لـ module بيستخدم `smpboot_register_percpu_thread`**

#### السياق
شركة IoT بتبني sensor aggregation hub على **STM32MP157** (dual Cortex-A7 + Cortex-M4). الـ Linux side بيشغّل kernel module لقراءة SPI sensors بشكل دوري. الـ module بيستخدم `smpboot_register_percpu_thread()` لإنشاء per-CPU polling threads. المشكلة ظهرت في الـ field لما الـ OTA update بيعمل `rmmod` للـ module القديم قبل تحميل الجديد.

#### المشكلة
```
BUG: kernel NULL pointer dereference at 0000000000000018
IP: smpboot_thread_fn+0x8c/0x1a0
```

#### التحليل
الـ `smp_hotplug_thread` struct محتاج يتعمله unregister بشكل صحيح قبل ما الـ module يتفكك من الذاكرة. لما الـ struct بيكون stack-allocated أو بيتحرر قبل `smpboot_unregister_percpu_thread()`، الـ kernel بيحاول يوصل للـ `thread_fn` pointer وهو بقى invalid:

```c
/* غلط: الـ struct بتتحرر قبل الـ unregister */
static int __exit spi_sensor_exit(void)
{
    kfree(sensor_ctx);          /* بيحرر الـ struct اللي فيه thread_fn */
    smpboot_unregister_percpu_thread(&sensor_thread); /* crash هنا */
    return 0;
}
```

الـ `smpboot_unregister_percpu_thread()` من `smpboot.h` بتعمل park للـ threads الشغّالة وبتمسح الـ list entry — لكن لو الـ `thread_fn` pointer اتحرر، الـ park sequence بيعمل NULL dereference.

#### الحل
الترتيب الصحيح: unregister الـ thread الأول، وبعدين حرّر الـ resources:

```c
static void __exit spi_sensor_exit(void)
{
    /* أول حاجة: وقّف وامسح الـ threads */
    smpboot_unregister_percpu_thread(&sensor_thread);

    /* بعدين: حرّر الـ resources */
    kfree(sensor_ctx);
}

static int __init spi_sensor_init(void)
{
    sensor_ctx = kzalloc(sizeof(*sensor_ctx), GFP_KERNEL);
    if (!sensor_ctx)
        return -ENOMEM;

    /* الـ register الأخير بعد تجهيز كل الـ resources */
    return smpboot_register_percpu_thread(&sensor_thread);
}
```

#### الدرس المستفاد
قاعدة ثابتة: `smpboot_unregister_percpu_thread()` لازم تتنادى **قبل** أي `kfree()` لأي resource بيستخدمه الـ `thread_fn`. ترتيب الـ init/exit لازم يكون معكوس تماماً.

---

### السيناريو 4: Automotive ECU على i.MX8QM — الـ thread بيشتغل على الـ CPU الغلط

#### العنوان
**الـ per-CPU thread بيشتغل على core خارج الـ real-time cluster على ECU سياراتي**

#### السياق
شركة automotive بتطور ECU على **i.MX8QM** (4x Cortex-A35 + 2x Cortex-A72). الـ design بيخصص A35 cores للـ real-time tasks والـ A72 للـ general purpose. الـ kernel module المسؤول عن CAN bus monitoring بيسجّل `smp_hotplug_thread` ومفروض يشتغل بس على الـ A35 cores (CPU0-CPU3).

#### المشكلة
الـ `ps` بيظهر `can_monitor/4` و`can_monitor/5` شغّالين على الـ A72 cores، وده بيسبب latency spikes في الـ real-time monitoring.

#### التحليل
الـ `smpboot_register_percpu_thread()` بيعمل thread لـ **كل** CPU موجود في الـ system. الـ struct مفيهوش field لـ CPU affinity مباشرةً:

```c
struct smp_hotplug_thread {
    struct task_struct  * __percpu *store;
    struct list_head    list;
    int  (*thread_should_run)(unsigned int cpu);  /* هنا الحل */
    void (*thread_fn)(unsigned int cpu);
    ...
};
```

الـ `thread_should_run` callback هو الآلية الصحيحة للتحكم في أيه CPUs تشغّل الـ thread. لو رجعت `0` لـ CPU معين، الـ thread بيتعمله park أوتوماتيك على اللي مش محتاجه.

```c
/* الحل: استخدم thread_should_run للتصفية */
static int can_monitor_should_run(unsigned int cpu)
{
    /* شغّل بس على A35 cores: CPU0-CPU3 */
    return cpu < 4;
}

static struct smp_hotplug_thread can_monitor_thread = {
    .store             = &can_monitor_task,
    .thread_fn         = can_monitor_fn,
    .thread_should_run = can_monitor_should_run,  /* التصفية هنا */
    .thread_comm       = "can_mon/%u",
};
```

#### الحل
تطبيق الـ CPU mask check داخل `thread_should_run`:

```c
/* بديل: استخدام cpumask صريح */
static cpumask_t rt_cpumask;

static int can_monitor_should_run(unsigned int cpu)
{
    return cpumask_test_cpu(cpu, &rt_cpumask);
}

static int __init can_monitor_init(void)
{
    /* خصّص الـ A35 cores فقط */
    cpumask_setall(&rt_cpumask);
    cpumask_clear_cpu(4, &rt_cpumask);
    cpumask_clear_cpu(5, &rt_cpumask);

    return smpboot_register_percpu_thread(&can_monitor_thread);
}
```

تحقق:
```bash
ps -eo pid,comm,psr | grep can_mon
# المفروض يظهر PSR 0,1,2,3 فقط — مش 4 أو 5
```

#### الدرس المستفاد
الـ `smpboot_register_percpu_thread()` مش بيعمل CPU affinity تلقائياً. الطريقة الوحيدة للتحكم في أيه CPUs تشغّل الـ thread هي `thread_should_run` callback. استخدامه كـ CPU filter هو الـ idiomatic way في الـ kernel.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ `create` callback بيتنادى من context غلط

#### العنوان
**الـ `create` callback بيعمل sleep في non-sleepable context أثناء bring-up على AM62x**

#### السياق
فريق bring-up بيشتغل على board مخصوصة بناءً على **TI AM625** (quad Cortex-A53). الـ board بيها HDMI transmitter متصل بـ I2C. الـ engineer كتب kernel module بيستخدم `smp_hotplug_thread` لعمل periodic HDMI status checking. الـ `create` callback المفروض يعمل initialize للـ I2C channel قبل ما الـ thread يبدأ.

#### المشكلة
عند `insmod`، الـ kernel يطبع:
```
BUG: sleeping function called from invalid context at kernel/locking/mutex.c:580
in_atomic(): 1, irqs_disabled(): 0, non-block: 0
```

#### التحليل
الـ doc في `smpboot.h` بيوضح الفرق بين `create` و`setup`:

```c
/**
 * @create: Optional setup function, called when the thread gets
 *          created (Not called from the thread context)        ← <-- ده هنا المشكلة
 * @setup:  Optional setup function, called when the thread gets
 *          operational the first time                          ← <-- ده الصح
 */
struct smp_hotplug_thread {
    ...
    void (*create)(unsigned int cpu);  /* NOT from thread context */
    void (*setup)(unsigned int cpu);   /* from thread context — can sleep */
    ...
};
```

الـ `create` callback بيتنادى من **CPU hotplug context** اللي ممكن يكون في atomic context أو مع special locking. أما الـ `setup` callback بيتنادى من **جوه الـ thread نفسه**، يعني ممكن يعمل sleep وينادي blocking functions زي I2C transactions.

```c
/* غلط: I2C blocking call في create */
static void hdmi_create(unsigned int cpu)
{
    /* i2c_smbus_read_byte_data ممكن تعمل sleep — crash هنا */
    u8 chip_id = i2c_smbus_read_byte_data(hdmi_client, HDMI_CHIP_ID_REG);
}

/* صح: I2C call في setup من thread context */
static void hdmi_setup(unsigned int cpu)
{
    /* هنا OK لأن الـ thread context يسمح بالـ sleep */
    u8 chip_id = i2c_smbus_read_byte_data(hdmi_client, HDMI_CHIP_ID_REG);
    per_cpu(hdmi_chip_id, cpu) = chip_id;
}

static struct smp_hotplug_thread hdmi_check_thread = {
    .store             = &hdmi_task,
    .thread_fn         = hdmi_check_fn,
    .thread_should_run = hdmi_should_run,
    .create            = NULL,         /* لا تستخدمه لـ blocking ops */
    .setup             = hdmi_setup,   /* الـ blocking init هنا */
    .thread_comm       = "hdmi_chk/%u",
};
```

#### الحل
نقل كل الـ blocking initialization من `create` إلى `setup`:

```c
/* create: بس للـ non-blocking, non-sleeping init */
static void hdmi_create(unsigned int cpu)
{
    /* init الـ spinlock أو الـ atomic variable — بس */
    spin_lock_init(&per_cpu(hdmi_lock, cpu));
}

/* setup: كل الـ I2C/blocking init هنا */
static void hdmi_setup(unsigned int cpu)
{
    struct i2c_client *client = hdmi_get_client(cpu);
    per_cpu(hdmi_chip_id, cpu) = i2c_smbus_read_byte_data(client, 0x00);
    per_cpu(hdmi_ready, cpu) = true;
}
```

تحقق أثناء bring-up:
```bash
dmesg | grep -E "(hdmi_chk|smpboot|sleeping function)"
# لازم تظهر threads من غير أي sleeping function warnings
ps aux | grep hdmi_chk
```

#### الدرس المستفاد
الفرق بين `create` و`setup` في `smp_hotplug_thread` مش مجرد تفصيلة: `create` بيتنادى من hotplug control path (ممكن atomic)، و`setup` بيتنادى من جوه الـ thread نفسه (آمن للـ sleep). أي blocking operation (I2C, SPI, mutex) لازم تروح في `setup` مش `create`.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأهم لمتابعة تطور kernel — الروابط دي مباشرة للموضوع:

| المقال | الأهمية |
|--------|---------|
| [Parallel CPU bringup for x86_64](https://lwn.net/Articles/924933/) | أحدث وأهم مقال — بيشرح إزاى parallel bringup اتعمل فى x86 وعلاقته بـ smpboot |
| [SMP: Boot and CPU hotplug refactoring - Part 1](https://lwn.net/Articles/493537/) | بيشرح إعادة هيكلة كود SMP boot وإزاى percpu threads اتنقلوا للـ core |
| [cpu/hotplug: Core infrastructure for cpu hotplug rework](https://lwn.net/Articles/677686/) | بيوضح المشاكل الأصلية فى notifier mechanism والـ asymmetry اللى أدت لتصميم `smp_hotplug_thread` |
| [Hotplug CPU Infrastructure](https://lwn.net/Articles/75096/) | المقال التاريخى الأول عن CPU hotplug فى kernel |
| [Documentation/cpu-hotplug.txt](https://lwn.net/Articles/537570/) | نسخة محفوظة من docs القديمة بتوضح التطور |
| [Documentation for CPU hotplug support](https://lwn.net/Articles/159561/) | شرح مبكر للـ hotplug وإزاى per-cpu threads بتتعامل معاه |

---

### توثيق kernel الرسمى

**الـ Documentation/** جوه الـ kernel source هو المرجع الأساسى:

```
Documentation/core-api/cpu_hotplug.rst
```

- الصفحة الرسمية: [CPU hotplug in the Kernel](https://docs.kernel.org/core-api/cpu_hotplug.html)
- نسخة v5.16: [kernel.org/doc/html/v5.16/core-api/cpu_hotplug.html](https://www.kernel.org/doc/html/v5.16/core-api/cpu_hotplug.html)
- نسخة v4.14: [kernel.org/doc/html/v4.14/core-api/cpu_hotplug.html](https://www.kernel.org/doc/html/v4.14/core-api/cpu_hotplug.html)

**الـ** source files الأساسية المرتبطة بـ `smpboot.h`:

```
include/linux/smpboot.h       ← الـ header الرئيسى
kernel/smpboot.c              ← التنفيذ الـ generic
arch/x86/kernel/smpboot.c     ← تنفيذ x86 المحدد
```

**الـ** per-cpu kthreads documentation:
- [Reducing OS jitter due to per-cpu kthreads](https://static.lwn.net/kerneldoc/admin-guide/kernel-per-CPU-kthreads.html)

---

### Commits مهمة فى git

#### الـ commit الأصلى — إنشاء البنية التحتية

**الـ commit** اللى أنشأ `smp_hotplug_thread` و `smpboot_register_percpu_thread`:

```
commit f97f8f06a49febbc3cb3635172efbe64ddc79700
Author: Thomas Gleixner <tglx@linutronix.de>
Date:   Mon Aug 13 2012

smpboot: Provide infrastructure for percpu hotplug threads
```

- الرابط: [kernel.googlesource.com — tip/tip 710d60cb](https://kernel.googlesource.com/pub/scm/linux/kernel/git/tip/tip/+/710d60cbf1b312a8075a2158cbfbbd9c66132dcc)
- الرابط على gitea: [smpboot: Provide infrastructure for percpu hotplug threads](https://gitea.lierfang.com/Proxmox-Port/linux-loongson/commit/f97f8f06a49febbc3cb3635172efbe64ddc79700)

#### الـ parallel bringup — تطوير حديث (2023)

**الـ patchset** بتاع David Woodhouse وUsama Arif لـ parallel CPU bringup:

- [PATCH v16 0/8 — Parallel CPU bringup for x86_64](https://lore.kernel.org/lkml/20230321194008.785922-1-usama.arif@bytedance.com/T/)
- [x86/smpboot: Support parallel startup of secondary CPUs](https://patchwork.kernel.org/project/rcu/patch/20230308171328.1562857-10-usama.arif@bytedance.com/)

#### الـ hotplug rework — SMP/hotplug pull request

- [GIT pull — smp/hotplug for 5.3-rc1 by Thomas Gleixner](https://lore.kernel.org/lkml/156257673794.14831.7846075415358186761.tglx@nanos.tec.linutronix.de/)

---

### نقاشات Mailing List

**الـ** patchset الأصلى على LKML:

- [linux.kernel — [Patch 3/7] smpboot: Provide infrastructure for percpu hotplug threads](https://groups.google.com/g/linux.kernel/c/Z-hseRBzqMo)
- [lkml.indiana.edu — tip:smp/hotplug smpboot percpu hotplug threads](https://lkml.indiana.edu/hypermail/linux/kernel/1208.1/02264.html)

**الـ** __percpu annotation fix:

- [PATCH: smpboot: Place the __percpu annotation correctly](https://lore.kernel.org/all/20190424085253.12178-1-bigeasy@linutronix.de/T/)

**الـ** unparking على control CPU:

- [PATCH: smp/hotplug: Move unparking of percpu threads to the control CPU](https://groups.google.com/g/linux.kernel/c/YtsTSuoOhqc/m/pOGOrPNhBQAJ)

**hotplug thread issues** — نقاش عن serialization bug:

- [hotplug thread issues — lore.kernel.org](https://lore.kernel.org/patchwork/patch/514592/)
- [hotplug thread issues — linux.kernel.narkive.com](https://linux.kernel.narkive.com/wC2GgDYz/hotplug-thread-issues)

---

### Kernelnewbies.org

**الـ** kernel version pages اللى بتوضح التغييرات المرتبطة بـ SMP boot:

- [Linux_3.15-DriversArch — kernelnewbies.org](https://kernelnewbies.org/Linux_3.15-DriversArch) — بيتضمن تغييرات hotplug threads
- [Linux_6.11 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.11) — تغييرات SMP حديثة
- [Linux_6.13 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.13)
- [Linux_6.14 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.14)
- [FirstKernelPatch — kernelnewbies.org](https://kernelnewbies.org/FirstKernelPatch) — للمبتدئين اللى عايزين يساهموا فى الكود ده

---

### eLinux.org

**أهم مرجع** من elinux عن SMP bring-up على ARM:

- [SMP Bring-up on ARM SoC — Clement (PDF)](https://elinux.org/images/0/00/Clement-smp-bring-up-on-arm-soc.pdf)
- [Category: Boot Time — elinux.org](https://elinux.org/Category:Boot_Time) — كل اللى يخص تحسين boot time بما فيه SMP

---

### GitHub — Source Code References

**الـ** x86 implementation الحالية:

- [arch/x86/kernel/smpboot.c — torvalds/linux](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/smpboot.c)

---

### كتب موصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)

**الـ LDD3** — متاح مجانًا على:
- [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

الفصول ذات الصلة:
- **Chapter 7** — Time, Delays, and Deferred Work — بيشرح kernel threads وdeferred work
- **Chapter 8** — Allocating Memory — بيشرح per-cpu allocations المستخدمة فى smpboot

#### Linux Kernel Development — Robert Love (3rd Ed.)

| الفصل | الموضوع |
|-------|---------|
| Chapter 7 | Interrupts and Interrupt Handlers |
| Chapter 8 | Bottom Halves and Deferring Work |
| **Chapter 9** | **An Introduction to Kernel Synchronization** — SMP locking |
| **Chapter 17** | **Devices and Modules** — per-cpu variables |

**الـ** chapters الأهم للفهم الكامل لـ `smp_hotplug_thread`.

#### Embedded Linux Primer — Christopher Hallinan (2nd Ed.)

- **Chapter 14** — Linux SMP — بيشرح كيفية بدء الـ secondary CPUs على embedded systems
- **Chapter 16** — Kernel Debugging Techniques — مفيد لـ debug مشاكل hotplug threads

---

### Search Terms للبحث عن معلومات أكثر

استخدم الـ terms دى على Google أو LKML أو git log:

```
smpboot_register_percpu_thread
smp_hotplug_thread kernel
linux kernel percpu hotplug thread park unpark
CPUHP_ONLINE smpboot
kthread_park kthread_unpark smpboot
linux kernel SMP secondary CPU bringup
CONFIG_SMP kernel thread hotplug
smpboot selfparking thread
```

**للبحث فى git log:**

```bash
git log --oneline --all -- kernel/smpboot.c
git log --oneline --all -- include/linux/smpboot.h
git log --grep="smpboot" --oneline
```

**للبحث فى LKML:**

```
site:lore.kernel.org smpboot percpu thread
site:lkml.org smpboot hotplug
```

---

### خلاصة مراجع سريعة

| النوع | الرابط |
|-------|--------|
| Kernel docs (رسمى) | [docs.kernel.org/core-api/cpu_hotplug.html](https://docs.kernel.org/core-api/cpu_hotplug.html) |
| LWN — parallel bringup | [lwn.net/Articles/924933](https://lwn.net/Articles/924933/) |
| LWN — hotplug rework | [lwn.net/Articles/677686](https://lwn.net/Articles/677686/) |
| LWN — SMP refactor | [lwn.net/Articles/493537](https://lwn.net/Articles/493537/) |
| Original commit | [gitea.lierfang.com — f97f8f06](https://gitea.lierfang.com/Proxmox-Port/linux-loongson/commit/f97f8f06a49febbc3cb3635172efbe64ddc79700) |
| LKML patch | [groups.google.com/linux.kernel](https://groups.google.com/g/linux.kernel/c/Z-hseRBzqMo) |
| x86 source | [github.com/torvalds/linux — smpboot.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/smpboot.c) |
| ARM SMP bringup (PDF) | [elinux.org — Clement SMP ARM](https://elinux.org/images/0/00/Clement-smp-bring-up-on-arm-soc.pdf) |
| Phoronix news | [Parallel CPU Bring-Up 2023](https://www.phoronix.com/news/Linux-CPU-Parallel-Bringup-2023) |
## Phase 8: Writing simple module

### الفكرة

**`smpboot_register_percpu_thread`** هي الدالة الأكثر إثارة في الملف — بتسجّل thread بيشتغل على كل CPU بشكل مستقل وبيتحكم فيه الـ CPU hotplug تلقائيًا. هنعمل kprobe على الدالة دي ونطبع اسم الـ thread اللي بيتسجّل وعنوان الـ `smp_hotplug_thread` struct.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * smpboot_probe.c — kprobe on smpboot_register_percpu_thread
 *
 * Prints the thread_comm (name) of every per-cpu smpboot thread
 * that gets registered while this module is loaded.
 */

#include <linux/module.h>       /* MODULE_* macros, module_init/exit        */
#include <linux/kernel.h>       /* pr_info                                   */
#include <linux/kprobes.h>      /* kprobe struct and API                     */
#include <linux/smpboot.h>      /* smp_hotplug_thread definition             */

/* -----------------------------------------------------------------------
 * pre_handler — بيتنادى قبل ما smpboot_register_percpu_thread تشتغل
 *
 * regs->di (x86-64 ABI) بيحتوي على أول argument — يعني pointer للـ
 * smp_hotplug_thread struct اللي بيتسجّل دلوقتي.
 * ----------------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Cast the first argument (RDI on x86-64) to the struct pointer */
    struct smp_hotplug_thread *ht =
        (struct smp_hotplug_thread *)regs->di;

    if (ht && ht->thread_comm)
        pr_info("smpboot_probe: registering per-cpu thread \"%s\" (plug=%px)\n",
                ht->thread_comm, ht);
    else
        pr_info("smpboot_probe: registering per-cpu thread <unknown>\n");

    return 0; /* 0 = let the real function continue normally */
}

/* -----------------------------------------------------------------------
 * kprobe descriptor
 * symbol_name بدل address عشان الكود يشتغل على أي kernel build
 * ----------------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "smpboot_register_percpu_thread",
    .pre_handler = handler_pre,
};

/* -----------------------------------------------------------------------
 * module_init — بيسجّل الـ kprobe
 * ----------------------------------------------------------------------- */
static int __init smpboot_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("smpboot_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("smpboot_probe: kprobe planted at %px (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

/* -----------------------------------------------------------------------
 * module_exit — بيشيل الـ kprobe قبل ما الـ handler يتحذف من الـ memory
 * ----------------------------------------------------------------------- */
static void __exit smpboot_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("smpboot_probe: kprobe removed\n");
}

module_init(smpboot_probe_init);
module_exit(smpboot_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on smpboot_register_percpu_thread — log per-cpu thread registrations");
```

---

### Makefile

```makefile
obj-m += smpboot_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kernel.h` | `pr_info` / `pr_err` |
| `linux/kprobes.h` | `struct kprobe` وكل الـ API بتاعها |
| `linux/smpboot.h` | تعريف `struct smp_hotplug_thread` عشان نعمل cast صح للـ argument |

---

#### `handler_pre`

**الـ `pre_handler`** بيتنادى بعد ما الـ kprobe تستوقف تنفيذ الـ CPU لحظة دخول الدالة المستهدفة، قبل ما أي instruction فيها تشتغل. الـ `pt_regs *regs` بيحتوي على حالة الـ registers في اللحظة دي، فـ `regs->di` هو `RDI` اللي في x86-64 System V ABI بيحمل أول argument — وده بالظبط الـ `struct smp_hotplug_thread *plug_thread` اللي بتاخده `smpboot_register_percpu_thread`. بنعمل cast ليه ونطبع `thread_comm` اللي هو اسم الـ thread المتسجّل.

---

#### `struct kprobe kp`

بنحدد `symbol_name` بدل عنوان hardcoded عشان الـ kernel يعمل resolve للعنوان بنفسه وقت الـ `register_kprobe` — ده بيخلي الكود يشتغل على أي kernel version من غير ما نعدّل. الـ `pre_handler` هو اللي هيشتغل قبل الدالة.

---

#### `module_init`

`register_kprobe` بتزرع breakpoint (أو trap instruction) في الذاكرة عند بداية `smpboot_register_percpu_thread`. لو فشلت (مثلًا الدالة `inline` أو `__kprobes`) بنرجع الـ error فورًا عشان منسيبش الـ module في حالة غلط.

---

#### `module_exit`

`unregister_kprobe` ضروري قبل ما الـ module يتحمل من الـ memory — لو تركنا الـ kprobe من غير ما نشيلها، الـ kernel هيحاول يكمّل تنفيذ الـ `handler_pre` بعد ما الكود اتحذف، وده kernel panic مضمون.

---

### تجربة عملية

```bash
# تحميل الـ module
sudo insmod smpboot_probe.ko

# في terminal تاني — شغّل أي حاجة بتسجّل per-cpu threads (مثلًا modprobe iwlwifi)
# أو شاهد الـ dmesg مباشرة
dmesg | grep smpboot_probe

# تفريغ الـ module
sudo rmmod smpboot_probe
```

**مثال على الـ output:**

```
[  42.301] smpboot_probe: kprobe planted at ffffffff810ab2c0 (smpboot_register_percpu_thread)
[  43.118] smpboot_probe: registering per-cpu thread "watchdog" (plug=ffff888003a14200)
[  43.119] smpboot_probe: registering per-cpu thread "migration" (plug=ffff888003a14280)
```

---

### ملاحظة على الـ architecture

الكود ده مكتوب لـ **x86-64** حيث `regs->di` = `RDI`. على ARM64 الـ first argument في `regs->regs[0]`. لو عايز portable code، استخدم `(void *)regs_get_kernel_argument(regs, 0)` اللي متوفرة في بعض kernel versions.
