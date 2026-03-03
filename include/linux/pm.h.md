## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

الـ `include/linux/pm.h` هو **العقد الرئيسي** لكل Power Management في الـ Linux kernel. ده الـ header اللي بيحدد إيه اللي المفروض يحصل لأي device لما الجهاز ييجي ينام أو يصحى.

---

### القصة من الأول

تخيل عندك لابتوب وأنت بتشتغل عليه. ضغطت على زرار السليب. إيه اللي المفروض يحصل؟

- الـ Wi-Fi card المفروض تبطل ترسل وتستقبل.
- الـ USB controller المفروض يوقف الـ DMA.
- الـ display المفروض يتقفل.
- الـ NVMe المفروض يحفظ حالته.

كل driver من دول بيعرف device بتاعه بس — مش بيعرف حاجة عن التاني. فمين اللي بيقولهم كلهم في نفس الوقت "اتحضروا للنوم"؟

الإجابة: الـ **PM core**، وبالتحديد ده بيبدأ من الـ `pm.h`.

---

### الـ Subsystem

الملف ده جزء من **POWER MANAGEMENT CORE** ومسجل في الـ `MAINTAINERS` تحت:

```
POWER MANAGEMENT CORE
M:  Rafael J. Wysocki <rafael@kernel.org>
L:  linux-pm@vger.kernel.org
F:  drivers/base/power/
F:  include/linux/pm.h
F:  include/linux/pm_*
```

---

### الفلسفة الأساسية — طبقات فوق بعض

الـ PM في Linux مش "الـ driver يتعامل مع النظام مباشرة". في **طبقتين**:

```
┌─────────────────────────────────────────────────┐
│           System / User Space                   │
│        (echo mem > /sys/power/state)            │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│            PM Core  (kernel/power/)             │
│   يبعت رسالة: SUSPEND / HIBERNATE / RESUME      │
└──────────────────────┬──────────────────────────┘
                       │ يستدعي
┌──────────────────────▼──────────────────────────┐
│   Subsystem Callbacks (bus, class, PM domain)   │
│     (PCI, USB, platform, ACPI, genpd ...)       │
└──────────────────────┬──────────────────────────┘
                       │ يستدعي
┌──────────────────────▼──────────────────────────┐
│       Driver Callbacks (dev_pm_ops)             │
│    .suspend() .resume() .runtime_suspend() ...  │
└─────────────────────────────────────────────────┘
```

الـ `pm.h` بيعرّف **الـ contract** بين الطبقات دي كلها.

---

### أهم ما في الملف

#### 1. الـ `struct dev_pm_ops` — قلب الموضوع

ده struct فيه pointers لـ callbacks. كل driver بيملى الـ callbacks اللي يهمه. الـ PM core بيستدعيهم في الترتيب الصح.

```c
struct dev_pm_ops {
    int (*prepare)(struct device *dev);   /* قبل أي حاجة */
    void (*complete)(struct device *dev); /* بعد الـ resume */

    /* System Sleep: Suspend to RAM */
    int (*suspend)(struct device *dev);
    int (*resume)(struct device *dev);
    int (*suspend_late)(struct device *dev);
    int (*resume_early)(struct device *dev);
    int (*suspend_noirq)(struct device *dev); /* بعد قفل الـ IRQs */
    int (*resume_noirq)(struct device *dev);

    /* Hibernation (Suspend to Disk) */
    int (*freeze)(struct device *dev);
    int (*thaw)(struct device *dev);
    int (*poweroff)(struct device *dev);
    int (*restore)(struct device *dev);
    /* ... و late/noirq variants لكل دي */

    /* Runtime PM — لما الـ device idle أثناء الشغل */
    int (*runtime_suspend)(struct device *dev);
    int (*runtime_resume)(struct device *dev);
    int (*runtime_idle)(struct device *dev);
};
```

**لماذا كل هذه الـ callbacks؟** لأن فيه فرق بين:
- **Suspend to RAM (S3)**: الـ RAM شغال، الـ CPU وقف — الـ device يحفظ حالته في الـ hardware registers.
- **Hibernate (S4)**: كل حاجة اتكتبت على الـ disk — الـ device يحفظ حالته في الـ RAM قبل كده.
- **Runtime PM**: الـ device idle وأنت لسه شغال — وفّر طاقة بدون ما المستخدم يحس.

#### 2. تسلسل الـ Suspend — القصة بالترتيب

لما تضغط Sleep:

```
prepare()          ← حضّر، منع أي children جديدة
    ↓
suspend()          ← وقّف الـ I/O والـ DMA
    ↓
suspend_late()     ← آخر خطوة قبل قفل الـ IRQs
    ↓
suspend_noirq()    ← مع الـ IRQs مقفولة (خطر!)
    ↓
[النظام نايم]
    ↓
resume_noirq()     ← أول خطوة والـ IRQs لسه مقفولة
    ↓
resume_early()     ← قبل فتح الـ IRQs
    ↓
resume()           ← اشتغل تاني
    ↓
complete()         ← نظّف اللي عمله prepare()
```

#### 3. الـ `pm_message_t` والـ `PM_EVENT_*`

الـ PM core بيبعت "رسالة" لكل device بتقوله إيه الـ transition اللي بتحصل:

```c
typedef struct pm_message {
    int event; /* PM_EVENT_SUSPEND, PM_EVENT_FREEZE, PM_EVENT_HIBERNATE ... */
} pm_message_t;
```

الـ constants دي زي:
| Event | المعنى |
|-------|--------|
| `PM_EVENT_SUSPEND` | نوم عادي (S3 - RAM) |
| `PM_EVENT_FREEZE` | تجميد قبل snapshot الـ hibernate |
| `PM_EVENT_HIBERNATE` | الـ snapshot اتحفظ، إقفل |
| `PM_EVENT_RESUME` | صحّى |
| `PM_EVENT_THAW` | فك التجميد |
| `PM_EVENT_RESTORE` | استرجع من الـ hibernate image |

#### 4. الـ `struct dev_pm_info` — الـ state الـ per-device

كل device في الـ kernel عنده `struct device`، وجوه field اسمه `power` من نوع `dev_pm_info`. ده بيحتوي على:

```c
struct dev_pm_info {
    pm_message_t    power_state;      /* الحالة الحالية */
    bool            can_wakeup:1;     /* قادر يصحّي النظام؟ */
    bool            is_suspended:1;   /* متوقف دلوقتي؟ */
    atomic_t        usage_count;      /* Runtime PM reference count */
    enum rpm_status runtime_status;   /* RPM_ACTIVE / RPM_SUSPENDED / ... */
    int             autosuspend_delay; /* تأخير قبل الـ auto suspend */
    u64             last_busy;        /* آخر مرة اتستخدم */
    /* ... والمزيد */
};
```

#### 5. الـ `struct dev_pm_domain`

الـ **PM Domain** هو تجميع منطقي لـ devices اشتراكوا في نفس مصدر الطاقة. مثال: كل الـ peripherals على نفس الـ power rail في SoC. لما تقطع الـ rail، كلهم بيتقطعوا. الـ domain بيتحكم فيهم كوحدة واحدة.

```c
struct dev_pm_domain {
    struct dev_pm_ops ops;             /* callbacks للـ domain كله */
    int (*start)(struct device *dev);
    void (*detach)(struct device *dev, bool power_off);
    int (*activate)(struct device *dev);
    int (*set_performance_state)(struct device *dev, unsigned int state);
};
```

#### 6. الـ Runtime PM Status

```c
enum rpm_status {
    RPM_ACTIVE,     /* شغال بكامل طاقته */
    RPM_RESUMING,   /* بيصحى */
    RPM_SUSPENDED,  /* نايم */
    RPM_SUSPENDING, /* بيدخل نوم */
    RPM_BLOCKED,    /* ممنوع ينام */
};
```

#### 7. Macros للـ Driver Writers

الـ kernel بيوفر macros بتسهل كتابة الـ drivers:

```c
/* أسهل طريقة: نفس الـ callback لـ suspend وـ hibernate */
DEFINE_SIMPLE_DEV_PM_OPS(my_pm_ops, my_suspend, my_resume);

/* لو عايز runtime PM كمان */
DEFINE_RUNTIME_DEV_PM_OPS(my_pm_ops, my_suspend, my_resume, my_idle);

/* لو الـ callbacks مع IRQs مقفولة */
DEFINE_NOIRQ_DEV_PM_OPS(my_pm_ops, my_suspend, my_resume);
```

والـ `pm_sleep_ptr()` و `pm_ptr()` بيخلوا الـ callback يبقى `NULL` لو الـ config مش موجود (مثلاً `CONFIG_PM_SLEEP=n`)، بدل ما تكتب `#ifdef` في كل حتة.

---

### الملفات المهمة في الـ Subsystem

**الـ Headers:**
| الملف | الدور |
|-------|-------|
| `include/linux/pm.h` | الـ core types والـ callbacks (ملفنا) |
| `include/linux/pm_runtime.h` | الـ Runtime PM API |
| `include/linux/pm_wakeup.h` | الـ wakeup sources |
| `include/linux/pm_domain.h` | الـ Generic PM Domains (genpd) |
| `include/linux/pm_qos.h` | الـ QoS constraints (latency, bandwidth) |
| `include/linux/pm_wakeirq.h` | الـ wake IRQs |
| `include/linux/pm_clock.h` | Clock management في الـ PM |

**الـ Core Implementation:**
| الملف | الدور |
|-------|-------|
| `drivers/base/power/main.c` | الـ DPM (Device PM) — التسلسل الرئيسي |
| `drivers/base/power/runtime.c` | الـ Runtime PM engine |
| `drivers/base/power/wakeup.c` | الـ wakeup sources tracking |
| `drivers/base/power/generic_ops.c` | الـ `pm_generic_*` implementations |
| `drivers/base/power/sysfs.c` | `/sys/devices/*/power/` interface |
| `drivers/base/power/qos.c` | الـ per-device QoS |
| `drivers/base/power/wakeirq.c` | الـ dedicated wake IRQs |

**الـ System-Level:**
| الملف | الدور |
|-------|-------|
| `kernel/power/suspend.c` | الـ S3 (suspend to RAM) flow |
| `kernel/power/hibernate.c` | الـ S4 (hibernate) flow |
| `kernel/power/main.c` | `/sys/power/state` entry point |

---

### لماذا هذا التعقيد؟

لأن الـ hardware مش بسيط. مثال حقيقي:

عندك USB device متوصل بـ USB hub متوصل بـ USB controller. لما تعمل suspend:
1. لازم الـ USB device ينام الأول.
2. بعدين الـ hub.
3. بعدين الـ controller.

ولما تعمل resume، العكس بالظبط. الـ `dpm_list` في الـ kernel بيرتب الـ devices دي بالترتيب الصح، والـ `drivers/base/power/main.c` بيمشي عليهم.

لو في أي device فشل في الـ suspend، الـ PM core بيعمل rollback ويصحّي اللي نام بالفعل.

---

### ملفات مهمة للقارئ

- `Documentation/power/runtime_pm.rst` — شرح Runtime PM بالتفصيل
- `Documentation/driver-api/pm/devices.rst` — كيف تكتب PM callbacks في driver
- `Documentation/power/basic-pm-debugging.rst` — debugging الـ suspend issues
- `include/linux/device.h` — تعريف `struct device` اللي فيه `power` field
## Phase 2: شرح الـ Power Management (PM) Framework

---

### المشكلة اللي بيحلها الـ PM Framework

أي نظام embedded أو mobile أو حتى server محتاج يتحكم في استهلاك الطاقة. من غير framework موحّد، كل driver هيعمل power management بطريقته الخاصة — فيه chaos كامل:

- **Driver A** بيوقّف الـ clock لما مفيش requests
- **Driver B** مش عارف إن الـ parent device وقف، فبيحاول يبعت data على bus مش شغّال
- **Driver C** مش عنده أي suspend logic خالص
- لما الـ system يقرر يروح sleep، مش في ضمان إن كل الـ devices اتعملها suspend بالترتيب الصح

النتيجة: **data corruption، kernel panics، أو battery drain كامل**.

الـ Linux PM framework بيحل ده بتعريف **contract موحّد** بين الـ PM core وكل driver في النظام.

---

### الحل اللي بيقدمه الـ Kernel

الـ kernel بيتعامل مع power management على **مستويين مختلفين**:

| المستوى | الوصف |
|---------|-------|
| **System Sleep** | الـ system كله بيروح sleep (suspend-to-RAM, hibernation) |
| **Runtime PM** | device بعينه بيتوقف أو بيشتغل بناءً على إذا محتاج أو لأ |

الـ framework بيعرّف **vtable** من callbacks (`struct dev_pm_ops`) كل driver بيملّيها، والـ PM core بيستدعيها في الوقت الصح وبالترتيب الصح.

---

### الـ Real-World Analogy

تخيل **فندق كبير** فيه مئات الأوض (الـ devices):

- **مدير الفندق** هو الـ **PM core** — هو اللي بيقرر إمتى الفندق يفضل شغّال أو يدخل في وضع توفير طاقة
- **كل أوضة** عندها **إجراءات موحّدة** — إزاي تتقفل، إزاي تتاح للنزيل تاني
- **خدمة الغرف** هي الـ **bus/subsystem layer** — بتتوسط بين المدير والأوضة
- **نزيل** عنده زرار "لا تزعجني" → هو الـ **`usage_count`** في runtime PM
- **Night mode للفندق كله** → هو **system suspend**
- **أوضة بعينها ما فيهاش نزيل** → الـ device بيدخل **runtime suspend** تلقائيًا

الـ mapping التفصيلي:

| الفندق | الـ Kernel |
|--------|-----------|
| مدير الفندق | PM core (`kernel/power/`) |
| نظام إجراءات الأوضة | `struct dev_pm_ops` |
| خدمة الغرف (intermediary) | Bus type / PM domain callbacks |
| النزيل (user of room) | `usage_count` (rpm_get/put) |
| Night mode للفندق كله | System suspend/hibernate |
| أوضة فاضية → وفّر طاقة | Runtime suspend (`runtime_idle` → `runtime_suspend`) |
| Wake-up call | `runtime_resume` / wakeup source |
| قائمة الأوضة بالترتيب | `dpm_list` (ordered suspend/resume) |

---

### الـ Big Picture Architecture

```
 ┌─────────────────────────────────────────────────────────┐
 │                    Userspace / systemd                   │
 │          echo mem > /sys/power/state                    │
 └────────────────────────┬────────────────────────────────┘
                          │
 ┌────────────────────────▼────────────────────────────────┐
 │               PM Core  (kernel/power/)                  │
 │  dpm_suspend_start() → dpm_suspend() → dpm_suspend_end()│
 │  dpm_resume_start()  → dpm_resume()  → dpm_resume_end() │
 └──────┬─────────────────┬──────────────────┬─────────────┘
        │                 │                  │
        ▼                 ▼                  ▼
 ┌────────────┐  ┌─────────────────┐  ┌───────────────┐
 │ PM Domain  │  │   Bus Type      │  │  Device Class │
 │(genpd/PSCI)│  │(PCI/USB/I2C/..)│  │ (block/net/..)│
 │dev_pm_domain│ │  bus->pm ops   │  │ class->pm ops │
 └─────┬──────┘  └────────┬────────┘  └───────┬───────┘
       │                  │                   │
       └──────────────────▼───────────────────┘
                          │
              ┌───────────▼──────────────┐
              │    struct dev_pm_ops     │  ← هنا في كل driver
              │  .suspend / .resume      │
              │  .runtime_suspend        │
              │  .runtime_resume         │
              │  .freeze / .thaw / ...   │
              └──────────────────────────┘
                          │
              ┌───────────▼──────────────┐
              │      struct device       │
              │    .power (dev_pm_info)  │
              └──────────────────────────┘
```

---

### الـ Core Abstraction: `struct dev_pm_ops`

ده الـ **vtable** الأساسي في الـ PM framework. كل device في الكيرنل بيعرف نفسه للـ PM core من خلال الـ `dev_pm_ops`.

```c
struct dev_pm_ops {
    /* === System Sleep Callbacks === */
    int (*prepare)(struct device *dev);    /* قبل أي suspend — lock children */
    void (*complete)(struct device *dev);  /* بعد أي resume — unlock */

    /* Suspend-to-RAM path */
    int (*suspend)(struct device *dev);         /* quiesce device */
    int (*suspend_late)(struct device *dev);    /* late stage, IRQs still on */
    int (*suspend_noirq)(struct device *dev);   /* IRQs off — final stage */

    int (*resume_noirq)(struct device *dev);    /* IRQs still off */
    int (*resume_early)(struct device *dev);    /* early wake */
    int (*resume)(struct device *dev);          /* fully restore device */

    /* Hibernation path (snapshot to disk) */
    int (*freeze)(struct device *dev);          /* before snapshot */
    int (*freeze_late)(struct device *dev);
    int (*freeze_noirq)(struct device *dev);

    int (*thaw_noirq)(struct device *dev);      /* after failed/completed snapshot */
    int (*thaw_early)(struct device *dev);
    int (*thaw)(struct device *dev);

    int (*poweroff)(struct device *dev);        /* after snapshot saved → power off */
    int (*poweroff_late)(struct device *dev);
    int (*poweroff_noirq)(struct device *dev);

    int (*restore_noirq)(struct device *dev);   /* after restoring from snapshot */
    int (*restore_early)(struct device *dev);
    int (*restore)(struct device *dev);

    /* === Runtime PM Callbacks === */
    int (*runtime_suspend)(struct device *dev); /* device idle → low power */
    int (*runtime_resume)(struct device *dev);  /* device needed → full power */
    int (*runtime_idle)(struct device *dev);    /* should we suspend? check here */
};
```

**كل callback ده اختياري** — لو الـ driver مش محتاجه يسيبه `NULL` والـ PM core هيعمل الـ default behavior.

---

### الـ Suspend Phases بالترتيب

الـ PM core بيعمل الـ suspend في **3 مراحل** لكل path — مش callback واحد. ده مهم جدًا:

```
System Suspend (S3 - suspend-to-RAM):
=====================================

dpm_prepare()       → calls .prepare()        for all devices
dpm_suspend()       → calls .suspend()        for all devices
dpm_suspend_late()  → calls .suspend_late()   for all devices
dpm_suspend_noirq() → calls .suspend_noirq()  for all devices (IRQs disabled)

                    [CPU sleeps here]

dpm_resume_noirq()  → calls .resume_noirq()   for all devices (IRQs still off)
dpm_resume_early()  → calls .resume_early()   for all devices
dpm_resume()        → calls .resume()         for all devices
dpm_complete()      → calls .complete()       for all devices


Hibernation (S4 - suspend-to-disk):
====================================

prepare → freeze → freeze_late → freeze_noirq
    [snapshot taken]
thaw_noirq → thaw_early → thaw → complete   (if snapshot OK, continue)
    [OR]
prepare → poweroff → poweroff_late → poweroff_noirq
    [machine powers off]
    [machine boots, loads snapshot]
restore_noirq → restore_early → restore → complete
```

**ليه كل ده التعقيد؟**

الـ `noirq` phase بتخلّي الـ suspend يتم بعد ما الـ interrupts اتعطّلت — ده ضروري لـ devices زي UART أو GPIO اللي ممكن تولّد interrupt في آخر لحظة. الـ `late` phase بتيح لـ devices تعمل عمليات تانية بعد ما معظم الـ devices اتوقفت.

---

### الـ Runtime PM في التفاصيل

الـ Runtime PM هو الجزء اللي بيشتغل **طول الوقت** حتى لو الـ system مش بيروح sleep.

```
Device States (enum rpm_status):
=================================

RPM_ACTIVE      ← device شغّال 100%
RPM_SUSPENDING  ← بيتعمله suspend دلوقتي
RPM_SUSPENDED   ← في low-power state
RPM_RESUMING    ← بيرجع يشتغل
RPM_BLOCKED     ← ممنوع يتعمله suspend (parent suspended أو disabled)
RPM_INVALID     ← uninitialized
```

الـ **`usage_count`** هو القلب بتاع الـ runtime PM:

```c
/* Driver needs device → increment usage_count */
pm_runtime_get(dev);      /* async */
pm_runtime_get_sync(dev); /* sync — waits for resume if suspended */

/* Driver done → decrement, may trigger idle check */
pm_runtime_put(dev);
pm_runtime_put_sync(dev);

/* When usage_count hits 0 → PM core calls .runtime_idle() */
/* If idle() returns 0 → PM core queues .runtime_suspend() */
```

الـ `autosuspend` بيضيف **delay** — الـ device مش بيتوقف فور ما الـ usage_count يوصل لـ 0، بينتظر `autosuspend_delay` milliseconds أول.

---

### الـ `struct dev_pm_info` — الـ Per-Device State

كل `struct device` عنده field اسمه `.power` من نوع `struct dev_pm_info`. ده هو الـ **state machine** الخاص بالـ PM لكل device:

```c
struct dev_pm_info {
    pm_message_t     power_state;      /* current PM message */
    bool             can_wakeup;       /* can this device wake the system? */
    bool             async_suspend;    /* suspend this device asynchronously? */
    bool             in_dpm_list;      /* is it registered with PM core? */
    bool             is_prepared;      /* prepare() was called */
    bool             is_suspended;     /* suspend() completed */
    bool             is_noirq_suspended;
    bool             is_late_suspended;
    bool             direct_complete;  /* skip suspend/resume (optimization) */
    u32              driver_flags;     /* DPM_FLAG_* */

    /* Runtime PM fields */
    atomic_t         usage_count;      /* active users */
    atomic_t         child_count;      /* active children */
    enum rpm_status  runtime_status;   /* current RPM state */
    enum rpm_request request;          /* pending request */
    int              autosuspend_delay;/* ms before auto-suspend */
    u64              last_busy;        /* last time device was active */
    u64              active_time;      /* total time active */
    u64              suspended_time;   /* total time suspended */

    struct hrtimer   suspend_timer;    /* for autosuspend delay */
    struct work_struct work;           /* async RPM work */
    wait_queue_head_t wait_queue;      /* waiters for state change */

    struct wakeup_source *wakeup;      /* wakeup event tracking */
    struct pm_subsys_data *subsys_data;/* for bus/PM domain use */
    struct dev_pm_qos    *qos;         /* QoS constraints */
};
```

---

### الـ `struct dev_pm_domain` — الـ Power Domains

الـ **PM domain** مفهوم مهم جدًا في الـ embedded Linux:

```
مثال: Allwinner A64 SoC

  ┌─────────────────────────────────────┐
  │          Power Domain: VCC-PL       │
  │  ┌─────────┐  ┌──────────────────┐  │
  │  │  GPIO   │  │   UART (PL2-5)   │  │
  │  └─────────┘  └──────────────────┘  │
  └─────────────────────────────────────┘

  ┌─────────────────────────────────────┐
  │          Power Domain: VCC-DSP      │
  │  ┌─────────┐  ┌──────────────────┐  │
  │  │  Audio  │  │      DSP Core    │  │
  │  └─────────┘  └──────────────────┘  │
  └─────────────────────────────────────┘
```

لو الـ Power Domain اتعمله power off، كل الـ devices جوه اتوقفت تلقائيًا. الـ `struct dev_pm_domain` بيسمح للـ PM framework إنه يتعامل مع ده:

```c
struct dev_pm_domain {
    struct dev_pm_ops  ops;      /* overrides driver callbacks! */
    int  (*start)(struct device *dev);
    void (*detach)(struct device *dev, bool power_off);
    int  (*activate)(struct device *dev);  /* before probe */
    void (*sync)(struct device *dev);      /* after probe */
    void (*dismiss)(struct device *dev);   /* after failed probe */
    int  (*set_performance_state)(struct device *dev, unsigned int state);
};
```

لما device ينتمي لـ PM domain، الـ PM core بيستدعي `domain->ops` **بدل** الـ driver ops مباشرة. الـ domain هو اللي بيقرر إذا كان هيستدعي الـ driver أو لأ.

---

### الـ `pm_message_t` وأنواع الـ Events

الـ `pm_message_t` هو **token** بسيط بيتبعت مع كل suspend/resume callback عشان الـ driver يعرف إيه اللي بيحصل:

```c
typedef struct pm_message {
    int event;  /* PM_EVENT_* bitmask */
} pm_message_t;

/* أهم الـ events */
#define PM_EVENT_SUSPEND    0x0002  /* going to RAM sleep */
#define PM_EVENT_HIBERNATE  0x0004  /* going to disk */
#define PM_EVENT_FREEZE     0x0001  /* creating hibernation snapshot */
#define PM_EVENT_RESUME     0x0010  /* waking up */
#define PM_EVENT_THAW       0x0020  /* after snapshot created */
#define PM_EVENT_RESTORE    0x0040  /* after snapshot restored */

/* Compound events */
#define PM_EVENT_SLEEP      (PM_EVENT_SUSPEND | PM_EVENT_HIBERNATE)
#define PM_EVENT_AUTO       0x0400  /* runtime PM triggered */
```

الـ helper macros:
```c
#define PMSG_IS_AUTO(msg)     /* هل ده runtime PM event؟ */
#define PMSG_NO_WAKEUP(msg)   /* هل ممنوع enable wakeup؟ (freeze/quiesce) */
```

---

### الـ DPM Flags — تحكم في سلوك الـ PM Core

الـ driver بيقدر يضبط `driver_flags` في `dev_pm_info` من `probe()`:

```c
/* أمثلة */
dev->power.driver_flags |= DPM_FLAG_NO_DIRECT_COMPLETE;
/* يمنع الـ PM core من skip الـ suspend/resume callbacks */

dev->power.driver_flags |= DPM_FLAG_SMART_SUSPEND;
/* لو الـ device runtime-suspended، متصحيهوش عشان system suspend */

dev->power.driver_flags |= DPM_FLAG_MAY_SKIP_RESUME;
/* لو suspend كان "smart"، ممكن نعدي الـ resume callbacks */
```

---

### الـ Helper Macros للـ Drivers

الـ kernel بيوفّر macros جاهزة عشان الـ drivers تعرّف الـ `dev_pm_ops` بسهولة:

```c
/* الأبسط — نفس callbacks لـ suspend-to-RAM وهيبرنيشن */
DEFINE_SIMPLE_DEV_PM_OPS(my_drv_pm_ops, my_suspend, my_resume);

/* لو محتاج runtime PM كمان */
static const struct dev_pm_ops my_pm = {
    SYSTEM_SLEEP_PM_OPS(my_suspend, my_resume)
    RUNTIME_PM_OPS(my_rt_suspend, my_rt_resume, my_rt_idle)
};

/* لو عايز callbacks تتنادى وهي IRQs مقفولة */
DEFINE_NOIRQ_DEV_PM_OPS(my_noirq_pm, my_suspend, my_resume);
```

الـ `pm_sleep_ptr()` و`pm_ptr()` هم macros بيخلّوا الـ pointer `NULL` لو الـ CONFIG_PM_SLEEP أو CONFIG_PM مش enabled — ده يمنع الـ compiler errors وبيوفّر مساحة في binary:

```c
#define pm_ptr(_ptr)       PTR_IF(IS_ENABLED(CONFIG_PM),       (_ptr))
#define pm_sleep_ptr(_ptr) PTR_IF(IS_ENABLED(CONFIG_PM_SLEEP), (_ptr))
```

---

### الـ dpm_list — الترتيب مهم جدًا

الـ PM core بيحتفظ بـ **`dpm_list`** — قائمة مرتبة بكل الـ devices. الترتيب ده ضروري:

```
Suspend:  Parent device يتوقف بعد الـ children
Resume:   Parent device يشتغل قبل الـ children

مثال:
  USB Host Controller (parent)
    └── USB Hub
         └── USB Keyboard

Suspend order:  Keyboard → Hub → USB Controller
Resume order:   USB Controller → Hub → Keyboard
```

الـ `enum dpm_order` بيتحكم في ترتيب device في الـ list لو اتعمله `device_move()`:

```c
enum dpm_order {
    DPM_ORDER_NONE,
    DPM_ORDER_DEV_AFTER_PARENT,
    DPM_ORDER_PARENT_BEFORE_DEV,
    DPM_ORDER_DEV_LAST,
};
```

---

### الـ `pm_subsys_data` — Bus-Level State

الـ `pm_subsys_data` هو جزء private للـ bus أو subsystem، محمي بـ spinlock:

```c
struct pm_subsys_data {
    spinlock_t      lock;
    unsigned int    refcount;

    /* لو CONFIG_PM_CLK enabled */
    unsigned int    clock_op_might_sleep;
    struct mutex    clock_mutex;
    struct list_head clock_list;  /* قائمة الـ clocks المربوطة بالـ device */

    /* لو CONFIG_PM_GENERIC_DOMAINS enabled */
    struct pm_domain_data *domain_data;  /* link to PM domain */
};
```

ده بيسمح لـ bus layers زي I2C أو SPI إنها تحتفظ بـ state خاص بيها (زي قائمة الـ clocks) من غير ما تاخد locks غلط.

---

### الـ PM Core يملك vs. الـ Driver يعمل

| مسؤولية الـ PM Core | مسؤولية الـ Driver |
|--------------------|-------------------|
| ترتيب suspend/resume بين الـ devices | quiesce الـ HW في `suspend()` |
| استدعاء الـ callbacks في الوقت الصح | save/restore registers |
| إدارة الـ `dpm_list` | تحديد الـ wakeup sources |
| تتبع الـ `rpm_status` | تعريف `runtime_idle` policy |
| إدارة الـ `usage_count` و `child_count` | إعادة initialize الـ HW في `resume()` |
| الـ `direct_complete` optimization | تحديد `driver_flags` في `probe()` |
| الـ locking (`device_pm_lock`) | الـ device-specific clock management |
| تشغيل الـ `async_suspend` threads | الـ wakeup IRQ registration |

---

### رسم العلاقة بين الـ Structs

```
struct device
  └── .power  ──────────────────────→ struct dev_pm_info
                                            ├── runtime_status (rpm_status)
                                            ├── usage_count
                                            ├── child_count
                                            ├── request (rpm_request)
                                            ├── suspend_timer (hrtimer)
                                            ├── work (work_struct)
                                            ├── wait_queue
                                            ├── wakeup ──────────→ struct wakeup_source
                                            ├── wakeirq ─────────→ struct wake_irq
                                            ├── subsys_data ─────→ struct pm_subsys_data
                                            │                           ├── clock_list
                                            │                           └── domain_data ──→ struct pm_domain_data
                                            └── qos ─────────────→ struct dev_pm_qos

struct device
  └── .pm_domain ──────────────────→ struct dev_pm_domain
                                          ├── ops (dev_pm_ops)  ← overrides driver!
                                          ├── start()
                                          ├── detach()
                                          ├── activate()
                                          └── set_performance_state()

struct bus_type / device_type / class
  └── .pm ─────────────────────────→ struct dev_pm_ops  ← subsystem-level callbacks

struct device_driver
  └── .pm ─────────────────────────→ struct dev_pm_ops  ← driver-level callbacks
```

---

### Priority Stack — مين بياخد الأولوية؟

لما الـ PM core يريد يعمل suspend، بيبص على:

```
1. dev->pm_domain->ops   ← PM domain (أعلى أولوية)
         ↓ (يستدعي manually لو عايز)
2. dev->type->pm         ← device type ops
         ↓
3. dev->class->pm        ← class ops
         ↓
4. dev->bus->pm          ← bus ops
         ↓ (يستدعي manually)
5. dev->driver->pm       ← driver ops (أقل أولوية)
```

لو الـ `pm_domain` موجود، بياخد الأولوية ومسؤوليته إنه هو اللي يستدعي الـ lower layers لو حب — مش الـ PM core.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### Config Options المؤثرة على الكود

| Option | الأثر |
|---|---|
| `CONFIG_PM` | يفعّل runtime PM كامل — بدونه كتير من الـ fields في `dev_pm_info` بتتشال |
| `CONFIG_PM_SLEEP` | يفعّل system suspend/hibernate — callbacks الـ sleep مش بتتعرّفش من غيره |
| `CONFIG_PM_CLK` | يضيف `clock_list` و`clock_mutex` في `pm_subsys_data` |
| `CONFIG_PM_GENERIC_DOMAINS` | يضيف `domain_data` في `pm_subsys_data` |
| `CONFIG_VT_CONSOLE_SLEEP` | يفعّل `pm_vt_switch_required()` لـ VT console |
| `CONFIG_CXL_SUSPEND` | يفعّل `cxl_mem_active()` للـ CXL memory |

#### **PM_EVENT_** — أحداث الـ Power State

| Macro | القيمة | المعنى |
|---|---|---|
| `PM_EVENT_INVALID` | `-1` | حالة غير صالحة |
| `PM_EVENT_ON` | `0x0000` | الجهاز شغّال عادي |
| `PM_EVENT_FREEZE` | `0x0001` | تجميد قبل أخذ snapshot للـ hibernate |
| `PM_EVENT_SUSPEND` | `0x0002` | النظام رايح ينام (suspend-to-RAM) |
| `PM_EVENT_HIBERNATE` | `0x0004` | الصورة اتحفظت، وقت الـ poweroff |
| `PM_EVENT_QUIESCE` | `0x0008` | تجميد قبل restore من صورة hibernate |
| `PM_EVENT_RESUME` | `0x0010` | النظام صحي من النوم |
| `PM_EVENT_THAW` | `0x0020` | فك التجميد بعد إنشاء الصورة |
| `PM_EVENT_RESTORE` | `0x0040` | استعادة حالة الجهاز من صورة hibernate |
| `PM_EVENT_RECOVER` | `0x0080` | فشل في إنشاء/استعادة الصورة |
| `PM_EVENT_USER` | `0x0100` | modifier — طلب من userspace |
| `PM_EVENT_REMOTE` | `0x0200` | modifier — remote wakeup |
| `PM_EVENT_AUTO` | `0x0400` | modifier — تلقائي من الـ subsystem |
| `PM_EVENT_POWEROFF` | `0x0800` | النظام رايح يُغلق |

الـ combined events دي بتتعمل بالـ OR:

| Macro | التركيب |
|---|---|
| `PM_EVENT_SLEEP` | `SUSPEND \| HIBERNATE` |
| `PM_EVENT_USER_SUSPEND` | `USER \| SUSPEND` |
| `PM_EVENT_AUTO_SUSPEND` | `AUTO \| SUSPEND` |
| `PM_EVENT_AUTO_RESUME` | `AUTO \| RESUME` |

#### **rpm_status** — حالة الجهاز في Runtime PM

| القيمة | المعنى |
|---|---|
| `RPM_INVALID` (`-1`) | حالة غير محددة أو مش initialized |
| `RPM_ACTIVE` (`0`) | الجهاز شغّال بالكامل |
| `RPM_RESUMING` | `runtime_resume()` بتتنفذ دلوقتي |
| `RPM_SUSPENDED` | `runtime_suspend()` خلصت بنجاح |
| `RPM_SUSPENDING` | `runtime_suspend()` بتتنفذ دلوقتي |
| `RPM_BLOCKED` | الجهاز اتمنع من الـ suspend |

#### **rpm_request** — أنواع الطلبات

| القيمة | المعنى |
|---|---|
| `RPM_REQ_NONE` | مفيش طلب معلّق |
| `RPM_REQ_IDLE` | شغّل `runtime_idle()` |
| `RPM_REQ_SUSPEND` | شغّل `runtime_suspend()` |
| `RPM_REQ_AUTOSUSPEND` | suspend بعد `autosuspend_delay` ms |
| `RPM_REQ_RESUME` | شغّل `runtime_resume()` |

#### **DPM_FLAG_** — Driver Flags لـ System Suspend

| Flag | Bit | المعنى |
|---|---|---|
| `DPM_FLAG_NO_DIRECT_COMPLETE` | 0 | إجبار تشغيل callbacks حتى لو الجهاز runtime-suspended |
| `DPM_FLAG_SMART_PREPARE` | 1 | اعتبر القيمة الموجبة من `prepare()` |
| `DPM_FLAG_SMART_SUSPEND` | 2 | متحاولش تصحّي الجهاز من runtime suspend أثناء system suspend |
| `DPM_FLAG_MAY_SKIP_RESUME` | 3 | ممكن تتخطى الـ noirq وearly resume callbacks |

#### **dpm_order** — ترتيب الـ dpm_list

| القيمة | المعنى |
|---|---|
| `DPM_ORDER_NONE` | ما تغيّرش الترتيب |
| `DPM_ORDER_DEV_AFTER_PARENT` | الجهاز بعد أبوه |
| `DPM_ORDER_PARENT_BEFORE_DEV` | الأب قبل الجهاز |
| `DPM_ORDER_DEV_LAST` | الجهاز في الآخر |

#### **PD_FLAG_** — Power Domain Attach Flags

| Flag | Bit | المعنى |
|---|---|---|
| `PD_FLAG_NO_DEV_LINK` | 0 | ما تعملش device-link للـ PM domain |
| `PD_FLAG_DEV_LINK_ON` | 1 | power-on الـ supplier عند إنشاء الـ link |
| `PD_FLAG_REQUIRED_OPP` | 2 | ربط required OPPs |
| `PD_FLAG_ATTACH_POWER_ON` | 3 | power-on الـ domain أثناء الـ attach |
| `PD_FLAG_DETACH_POWER_OFF` | 4 | power-off الـ domain أثناء الـ detach |

---

### الـ Structs المهمة

#### 1. `pm_message_t` (alias لـ `struct pm_message`)

**الغرض:** wrapper بسيط بيحمل نوع حدث الـ power transition.

```c
typedef struct pm_message {
    int event;  /* one of PM_EVENT_* values */
} pm_message_t;
```

| Field | النوع | الدور |
|---|---|---|
| `event` | `int` | قيمة `PM_EVENT_*` بتوصف نوع الـ transition |

**الـ PMSG_ macros** بتبني instances جاهزة:
```c
PMSG_SUSPEND  → { .event = PM_EVENT_SUSPEND }
PMSG_RESUME   → { .event = PM_EVENT_RESUME }
```

بتتمرر لكل `dpm_suspend()` / `dpm_resume()` وللـ callbacks في `dev_pm_ops`.

---

#### 2. `struct dev_pm_ops`

**الغرض:** جدول الـ callbacks اللي الـ driver/subsystem بيملأه — هو العقد بين الـ PM core والـ driver.

```c
struct dev_pm_ops {
    /* System sleep */
    int (*prepare)(struct device *dev);
    void (*complete)(struct device *dev);
    int (*suspend)(struct device *dev);
    int (*resume)(struct device *dev);
    int (*freeze)(struct device *dev);
    int (*thaw)(struct device *dev);
    int (*poweroff)(struct device *dev);
    int (*restore)(struct device *dev);

    /* Late/early variants (run with IRQs still on) */
    int (*suspend_late)(struct device *dev);
    int (*resume_early)(struct device *dev);
    int (*freeze_late)(struct device *dev);
    int (*thaw_early)(struct device *dev);
    int (*poweroff_late)(struct device *dev);
    int (*restore_early)(struct device *dev);

    /* noirq variants (run with IRQs disabled) */
    int (*suspend_noirq)(struct device *dev);
    int (*resume_noirq)(struct device *dev);
    int (*freeze_noirq)(struct device *dev);
    int (*thaw_noirq)(struct device *dev);
    int (*poweroff_noirq)(struct device *dev);
    int (*restore_noirq)(struct device *dev);

    /* Runtime PM */
    int (*runtime_suspend)(struct device *dev);
    int (*runtime_resume)(struct device *dev);
    int (*runtime_idle)(struct device *dev);
};
```

**تقسيم الـ callbacks:**

| المجموعة | الـ callbacks | وقت التشغيل |
|---|---|---|
| System Sleep | `suspend/resume/freeze/thaw/poweroff/restore` | IRQs on، قبل تجميد الـ tasks |
| Late/Early | `*_late / *_early` | IRQs on، آخر خطوة قبل noirq |
| noirq | `*_noirq` | **IRQs disabled**، أخطر مرحلة |
| Runtime PM | `runtime_suspend/resume/idle` | عند idle الجهاز ديناميكيًا |

---

#### 3. `struct dev_pm_info`

**الغرض:** الـ state الكامل لـ power management لجهاز واحد. موجود كـ `power` field في `struct device`.

```c
struct dev_pm_info {
    /* --- always present --- */
    pm_message_t    power_state;     /* current PM state message */
    bool            can_wakeup:1;    /* can this device wake the system? */
    bool            async_suspend:1; /* suspend this device asynchronously */
    bool            in_dpm_list:1;   /* on the dpm_list? */
    bool            is_prepared:1;   /* prepare() was called */
    bool            is_suspended:1;  /* suspend() done */
    bool            is_noirq_suspended:1;
    bool            is_late_suspended:1;
    bool            no_pm:1;         /* skip PM entirely */
    bool            early_init:1;    /* device was early-initialized */
    bool            direct_complete:1; /* skip suspend/resume (stay runtime-suspended) */
    u32             driver_flags;    /* DPM_FLAG_* set by driver at probe */
    spinlock_t      lock;            /* protects runtime PM fields */

    /* --- CONFIG_PM_SLEEP --- */
    struct list_head    entry;       /* link in dpm_list */
    struct completion   completion;  /* async suspend/resume sync */
    struct wakeup_source *wakeup;    /* wakeup source, NULL if disabled */
    bool            work_in_progress:1;
    bool            wakeup_path:1;   /* device is in wakeup path */
    bool            syscore:1;       /* syscore device, never runtime-suspended */
    bool            no_pm_callbacks:1;
    bool            smart_suspend:1;
    bool            must_resume:1;
    bool            may_skip_resume:1;
    bool            out_band_wakeup:1;
    bool            strict_midlayer:1;

    /* --- CONFIG_PM (runtime PM) --- */
    struct hrtimer      suspend_timer;    /* autosuspend timer */
    u64                 timer_expires;
    struct work_struct  work;             /* deferred suspend/resume work */
    wait_queue_head_t   wait_queue;       /* waiters for status change */
    struct wake_irq     *wakeirq;         /* dedicated wakeup IRQ */
    atomic_t            usage_count;      /* get/put reference counter */
    atomic_t            child_count;      /* active children count */
    unsigned int        disable_depth:3;  /* pm_runtime_disable() nesting */
    bool                idle_notification:1;
    bool                request_pending:1;
    bool                deferred_resume:1;
    bool                needs_force_resume:1;
    bool                runtime_auto:1;   /* autosuspend enabled via sysfs */
    bool                ignore_children:1;
    bool                no_callbacks:1;   /* skip runtime PM callbacks */
    bool                irq_safe:1;       /* callbacks run in atomic context */
    bool                use_autosuspend:1;
    bool                timer_autosuspends:1;
    bool                memalloc_noio:1;
    unsigned int        links_count;      /* device-links count */
    enum rpm_request    request;          /* pending RPM_REQ_* */
    enum rpm_status     runtime_status;   /* current RPM_* state */
    enum rpm_status     last_status;      /* last status before error */
    int                 runtime_error;
    int                 autosuspend_delay; /* ms before auto-suspend */
    u64                 last_busy;        /* last time device was used */
    u64                 active_time;      /* total time in RPM_ACTIVE */
    u64                 suspended_time;   /* total time in RPM_SUSPENDED */
    u64                 accounting_timestamp;

    /* --- always present at end --- */
    struct pm_subsys_data   *subsys_data; /* owned by subsystem */
    void (*set_latency_tolerance)(struct device *, s32);
    struct dev_pm_qos       *qos;         /* QoS constraints */
    bool                    detach_power_off:1;
};
```

---

#### 4. `struct pm_subsys_data`

**الغرض:** بيانات إضافية بيملكها الـ subsystem (مش الـ PM core)، بتتشارك بين الـ devices على نفس الـ bus.

```c
struct pm_subsys_data {
    spinlock_t      lock;          /* protects refcount and lists */
    unsigned int    refcount;      /* reference count */

    /* CONFIG_PM_CLK */
    unsigned int    clock_op_might_sleep;
    struct mutex    clock_mutex;   /* serialize clock operations */
    struct list_head clock_list;   /* list of PM clocks */

    /* CONFIG_PM_GENERIC_DOMAINS */
    struct pm_domain_data *domain_data; /* genpd-specific data */
};
```

---

#### 5. `struct dev_pm_domain`

**الغرض:** الـ PM domain بيوفر callbacks بديلة للـ subsystem-level callbacks، بيطبّق policy موحّدة على مجموعة devices.

```c
struct dev_pm_domain {
    struct dev_pm_ops   ops;       /* overrides subsystem PM callbacks */
    int  (*start)(struct device *dev);
    void (*detach)(struct device *dev, bool power_off);
    int  (*activate)(struct device *dev);
    void (*sync)(struct device *dev);
    void (*dismiss)(struct device *dev);
    int  (*set_performance_state)(struct device *dev, unsigned int state);
};
```

| Field | وقت الاستدعاء |
|---|---|
| `ops` | بيحل محل callbacks الـ bus type أثناء system suspend/runtime PM |
| `start` | لما userspace أو driver يحتاج يشغّل الجهاز |
| `activate` | قبل تشغيل `probe()` |
| `sync` | بعد نجاح `probe()` |
| `dismiss` | بعد فشل `probe()` أو بعد `remove()` |
| `detach` | لما الجهاز بيتشال من الـ domain |
| `set_performance_state` | لتغيير الـ OPP (Operating Performance Point) |

---

#### 6. `struct wakeup_source` (من `pm_wakeup.h`)

**الغرض:** تتبّع مصادر الـ wakeup، بيحول دون دخول النظام في النوم لما الجهاز محتاج يكون صاحي.

```c
struct wakeup_source {
    const char      *name;
    int             id;
    struct list_head entry;      /* global wakeup_sources list */
    spinlock_t      lock;
    struct wake_irq *wakeirq;
    struct timer_list timer;     /* wakeup timeout timer */
    unsigned long   timer_expires;
    ktime_t         total_time;
    ktime_t         max_time;
    ktime_t         last_time;
    ktime_t         start_prevent_time;
    ktime_t         prevent_sleep_time;
    unsigned long   event_count;
    unsigned long   active_count;
    unsigned long   relax_count;
    unsigned long   expire_count;
    unsigned long   wakeup_count;
    struct device   *dev;        /* sysfs representation */
    bool            active:1;
    bool            autosleep_enabled:1;
};
```

---

### رسوم العلاقات بين الـ Structs

```
struct device
  └─► .power  ──────────────────────────────────────────► struct dev_pm_info
                                                               │
                      ┌────────────────────────────────────────┤
                      │                                         │
                      ▼                                         ▼
            struct wakeup_source              struct pm_subsys_data
              (dev_pm_info.wakeup)              (dev_pm_info.subsys_data)
                      │                                         │
                      │                                  ┌──────┴───────┐
                      ▼                                  ▼               ▼
              struct wake_irq             struct pm_domain_data    clock_list
              (wakeup_source.wakeirq)     (subsys_data.domain_data)

  └─► .pm_domain ─────────────────────────────────────► struct dev_pm_domain
                                                               │
                                                               ▼
                                                       struct dev_pm_ops
                                                       (embedded: domain->ops)

  └─► .driver ──► struct device_driver
                      └─► .pm ─────────────────────────► struct dev_pm_ops
                                                         (driver-level callbacks)

  └─► .bus ────► struct bus_type
                      └─► .pm ─────────────────────────► struct dev_pm_ops
                                                         (bus-level callbacks)
```

**أولوية تنفيذ الـ callbacks (من الأعلى للأدنى):**

```
struct dev_pm_domain .ops    ← الأولوية الأعلى (يلغي الكل)
        │
        ▼ (لو مفيش pm_domain)
struct bus_type .pm          ← الـ bus callbacks
        │
        ▼ (bus بيستدعي)
struct device_driver .pm     ← الـ driver callbacks
```

---

### رسم دورة حياة Runtime PM

```
          Device Registered
                 │
                 ▼
         RPM_ACTIVE (usage_count=0)
                 │
        [no activity, idle timer fires]
                 │
                 ▼
         runtime_idle() called
                 │
         ┌───────┴──────────┐
         │ returns 0         │ returns error
         ▼                   ▼
  RPM_REQ_SUSPEND          stay RPM_ACTIVE
         │
         ▼
  RPM_SUSPENDING
  runtime_suspend() called
         │
    ┌────┴────┐
    │ success  │ error
    ▼          ▼
RPM_SUSPENDED  RPM_ACTIVE
    │
    │ [wakeup event / pm_runtime_get()]
    ▼
RPM_RESUMING
runtime_resume() called
    │
    ▼
RPM_ACTIVE
    │
    │ [pm_runtime_disable()]
    ▼
RPM_BLOCKED   ← disable_depth > 0
```

---

### رسم دورة حياة System Suspend (Suspend-to-RAM)

```
User: echo mem > /sys/power/state
          │
          ▼
  dpm_prepare(PMSG_SUSPEND)
  ┌─ for each device in dpm_list:
  │      dev_pm_ops.prepare(dev)
  │      → sets is_prepared=true
  └─────────────────────────────
          │
          ▼
  dpm_suspend(PMSG_SUSPEND)
  ┌─ for each device:
  │      check direct_complete flag
  │      dev_pm_ops.suspend(dev)
  │      → device quiesces I/O
  └─────────────────────────────
          │
          ▼
  dpm_suspend_late(PMSG_SUSPEND)
  ┌─ dev_pm_ops.suspend_late(dev)
  └─────────────────────────────
          │
          ▼
  dpm_suspend_noirq(PMSG_SUSPEND)   ← IRQs disabled هنا
  ┌─ dev_pm_ops.suspend_noirq(dev)
  │   → device enters low-power state
  │   → configure wakeup if needed
  └─────────────────────────────
          │
          ▼
    [CPU enters sleep]

          │ [wakeup event]
          ▼
  dpm_resume_noirq(PMSG_RESUME)    ← IRQs still disabled
  dpm_resume_early(PMSG_RESUME)
  dpm_resume(PMSG_RESUME)
  dpm_complete(PMSG_RESUME)
  ┌─ dev_pm_ops.complete(dev)
  └─────────────────────────────
          │
          ▼
    System Running Again
```

---

### رسم دورة حياة Hibernation

```
User: echo disk > /sys/power/state

Phase 1: Freeze (snapshot preparation)
  dpm_prepare → dpm_freeze → dpm_freeze_late → dpm_freeze_noirq
                                                      │
                                                      ▼
                                            [hibernation image saved]
                                                      │
Phase 2: Poweroff                                     │
  dpm_poweroff → dpm_poweroff_late → dpm_poweroff_noirq
                                                      │
                                                      ▼
                                              [system powers off]

Resume from hibernation image:
  [kernel loads] → dpm_restore_noirq → dpm_restore_early → dpm_restore
                                                      │
                                                      ▼
                                              dpm_thaw (if image failed)
                                              OR dpm_complete (if success)
```

---

### رسم Call Flow — Runtime PM

```
Driver calls pm_runtime_put(dev)
  │
  ▼
rpm_idle(dev, RPM_ASYNC)
  │  [schedules work or calls directly]
  ▼
dev_pm_ops.runtime_idle(dev)     ← bus-level or domain-level
  │  [if returns 0]
  ▼
rpm_suspend(dev, RPM_ASYNC)
  │  [queues work_struct in dev->power.work]
  ▼
[workqueue thread runs]
  │
  ▼
__rpm_callback()
  │
  ├─► pm_domain->ops.runtime_suspend(dev)   [if pm_domain set]
  │       OR
  └─► bus_type->pm->runtime_suspend(dev)    [bus-level]
            │
            └─► driver->pm->runtime_suspend(dev)  [driver-level]
                      │
                      └─► hardware enters low-power state

[wakeup needed]
pm_runtime_get(dev)
  │
  ▼
rpm_resume(dev)
  │
  ▼
__rpm_callback()
  │
  └─► runtime_resume() chain (reverse order)
            │
            ▼
        dev->power.runtime_status = RPM_ACTIVE
```

---

### رسم Call Flow — System Suspend للـ Subsystem

```
PM core calls dpm_suspend(state)
  │
  ▼
device_suspend(dev)
  │
  ├─[has pm_domain?]─► pm_domain->ops.suspend(dev)
  │                         │
  │                         └─► domain handles driver internally
  │
  └─[no pm_domain]──► bus->pm->suspend(dev)    ← e.g. pci_pm_suspend()
                            │
                            └─► drv->pm->suspend(dev)  ← driver callback
                                      │
                                      ▼
                               device quiesces
                               registers saved
```

---

### استراتيجية الـ Locking

#### الـ Locks في `dev_pm_info`

| Lock | النوع | بيحمي إيه |
|---|---|---|
| `dev_pm_info.lock` | `spinlock_t` | كل الـ runtime PM state fields: `runtime_status`، `request`، `usage_count`، `timer_expires`، `disable_depth` |

**مهم:** الـ `lock` ده بيتأخذ وسط أي عملية runtime PM — يعني الـ callbacks نفسها بتتنفذ **بدون** الـ lock (لأنها ممكن تنام).

#### الـ Locks في `pm_subsys_data`

| Lock | النوع | بيحمي إيه |
|---|---|---|
| `pm_subsys_data.lock` | `spinlock_t` | `refcount` وأي state مشترك بين الـ devices على نفس الـ bus |
| `pm_subsys_data.clock_mutex` | `mutex` | `clock_list` وعمليات الـ PM clocks (ممكن تنام) |

#### `device_pm_lock()` / `device_pm_unlock()`

دي mutex عالمية بتحمي `dpm_list` كاملة — بتتأخذ في `dpm_prepare()` وبتتحرر بعد `dpm_complete()`.

```
device_pm_lock()          ← global dpm_list mutex
  └─ dpm_list protected
       └─ per-device: dev->power.lock  (spinlock, finer grained)
```

#### ترتيب الـ Locks (Lock Ordering)

```
1. device_pm_lock()          [global, outermost]
2. dev->power.lock           [per-device spinlock]
3. wakeup_source.lock        [per-source spinlock]
4. pm_subsys_data.lock       [per-subsystem spinlock]
5. pm_subsys_data.clock_mutex [can sleep, innermost]
```

**قاعدة:** ما تأخدش lock رقم أكبر وأنت شايل lock رقم أصغر إلا بالترتيب ده عشان تتجنب deadlock.

#### الـ `irq_safe` Flag

لو `dev->power.irq_safe = true`، الـ runtime PM callbacks بتتنفذ مع الـ `dev->power.lock` شايلها (بدلاً من تحريرها) — مناسب للـ devices اللي callbacks بتاعتها atomic ومش بتنام.

---

### الـ Macros المساعدة — Cheatsheet

| Macro | الاستخدام |
|---|---|
| `DEFINE_SIMPLE_DEV_PM_OPS(name, suspend, resume)` | أبسط طريقة: نفس الـ fn لـ suspend وhibernate، مفيش runtime PM |
| `DEFINE_NOIRQ_DEV_PM_OPS(name, suspend, resume)` | لما تحتاج فقط noirq callbacks |
| `DEFINE_RUNTIME_DEV_PM_OPS(...)` | كامل: sleep + runtime في struct واحدة |
| `EXPORT_SIMPLE_DEV_PM_OPS(name, ...)` | زي DEFINE بس بيعمل export للـ symbol |
| `EXPORT_GPL_SIMPLE_DEV_PM_OPS(name, ...)` | نفسه بـ GPL license |
| `pm_ptr(ptr)` | بيرجع `NULL` لو `CONFIG_PM` مش مفعّل |
| `pm_sleep_ptr(ptr)` | بيرجع `NULL` لو `CONFIG_PM_SLEEP` مش مفعّل |
| `PMSG_IS_AUTO(msg)` | يتحقق لو الـ event كان automatic |
| `PMSG_NO_WAKEUP(msg)` | يتحقق لو الـ event بيمنع wakeup (FREEZE أو QUIESCE) |
| `suspend_report_result(dev, fn, ret)` | يطبع error في الـ kernel log لو suspend callback فشل |
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheat Sheet

#### Group 1: VT Console / CXL Helpers

| Function | Signature Summary | Purpose |
|---|---|---|
| `pm_vt_switch_required` | `(dev, bool required) → int` | سجّل/الغِ تسجيل حاجة الـ VT switch عند الـ suspend |
| `pm_vt_switch_unregister` | `(dev) → void` | أزِل تسجيل الـ device من قائمة الـ VT switch |
| `cxl_mem_active` | `(void) → bool` | هل في CXL memory نشط يمنع الـ suspend؟ |

#### Group 2: Subsystem Data

| Function | Signature Summary | Purpose |
|---|---|---|
| `dev_pm_get_subsys_data` | `(dev) → int` | احجز/زوِّد مرجع `pm_subsys_data` للـ device |
| `dev_pm_put_subsys_data` | `(dev) → void` | حرّر المرجع — يحذف الـ struct لو refcount وصل صفر |

#### Group 3: DPM — Device PM List Control

| Function | Signature Summary | Purpose |
|---|---|---|
| `device_pm_lock` | `(void) → void` | خذ الـ dpm_list mutex |
| `device_pm_unlock` | `(void) → void` | حرّر الـ dpm_list mutex |
| `dpm_prepare` | `(state) → int` | استدعِ `->prepare()` لكل الـ devices |
| `dpm_suspend` | `(state) → int` | استدعِ `->suspend()` لكل الـ devices |
| `dpm_suspend_late` | `(state) → int` | استدعِ `->suspend_late()` لكل الـ devices |
| `dpm_suspend_noirq` | `(state) → int` | استدعِ `->suspend_noirq()` — IRQs disabled |
| `dpm_suspend_start` | `(state) → int` | = `dpm_prepare` + `dpm_suspend` |
| `dpm_suspend_end` | `(state) → int` | = `dpm_suspend_late` + `dpm_suspend_noirq` |
| `dpm_resume` | `(state) → void` | استدعِ `->resume()` لكل الـ devices |
| `dpm_resume_early` | `(state) → void` | استدعِ `->resume_early()` لكل الـ devices |
| `dpm_resume_noirq` | `(state) → void` | استدعِ `->resume_noirq()` — IRQs disabled |
| `dpm_complete` | `(state) → void` | استدعِ `->complete()` لكل الـ devices |
| `dpm_resume_start` | `(state) → void` | = `dpm_resume_noirq` + `dpm_resume_early` |
| `dpm_resume_end` | `(state) → void` | = `dpm_resume` + `dpm_complete` |
| `device_pm_wait_for_dev` | `(sub, dev) → int` | خلِّي `sub` ينتظر إن `dev` خلص suspend/resume |
| `dpm_for_each_dev` | `(data, fn) → void` | iterate على كل devices في الـ dpm_list |

#### Group 4: Generic PM Callbacks

| Function | Purpose |
|---|---|
| `pm_generic_prepare` | generic `->prepare()` — يستدعي driver callback |
| `pm_generic_suspend` | generic `->suspend()` |
| `pm_generic_suspend_late` | generic `->suspend_late()` |
| `pm_generic_suspend_noirq` | generic `->suspend_noirq()` |
| `pm_generic_resume` | generic `->resume()` |
| `pm_generic_resume_early` | generic `->resume_early()` |
| `pm_generic_resume_noirq` | generic `->resume_noirq()` |
| `pm_generic_freeze` | generic `->freeze()` للـ hibernation |
| `pm_generic_freeze_noirq` | generic `->freeze_noirq()` |
| `pm_generic_thaw` | generic `->thaw()` بعد فشل hibernation أو creation |
| `pm_generic_thaw_noirq` | generic `->thaw_noirq()` |
| `pm_generic_restore` | generic `->restore()` من الـ hibernation image |
| `pm_generic_restore_early` | generic `->restore_early()` |
| `pm_generic_restore_noirq` | generic `->restore_noirq()` |
| `pm_generic_poweroff` | generic `->poweroff()` بعد حفظ الـ image |
| `pm_generic_poweroff_late` | generic `->poweroff_late()` |
| `pm_generic_poweroff_noirq` | generic `->poweroff_noirq()` |
| `pm_generic_complete` | generic `->complete()` |

#### Group 5: Skip/Optimization Helpers

| Function | Signature Summary | Purpose |
|---|---|---|
| `dev_pm_skip_resume` | `(dev) → bool` | هل ممكن نتخطى الـ resume callbacks؟ |
| `dev_pm_skip_suspend` | `(dev) → bool` | هل ممكن نتخطى الـ suspend callbacks؟ |
| `suspend_report_result` | macro `(dev, fn, ret)` | log error لو الـ suspend callback فشل |
| `__suspend_report_result` | `(function, dev, fn, ret) → void` | الـ underlying function للـ macro |

#### Group 6: Convenience Macros — pm_ptr / pm_sleep_ptr

| Macro | Purpose |
|---|---|
| `pm_ptr(_ptr)` | يرجع `_ptr` لو `CONFIG_PM` مفعّل، وإلا `NULL` |
| `pm_sleep_ptr(_ptr)` | يرجع `_ptr` لو `CONFIG_PM_SLEEP` مفعّل، وإلا `NULL` |

#### Group 7: PM Ops Definition Macros

| Macro | Purpose |
|---|---|
| `DEFINE_SIMPLE_DEV_PM_OPS` | عرّف `dev_pm_ops` بـ suspend/resume فقط |
| `DEFINE_NOIRQ_DEV_PM_OPS` | عرّف `dev_pm_ops` بـ noirq variants فقط |
| `EXPORT_SIMPLE_DEV_PM_OPS` | عرّف وـexport الـ ops بـ sleep guards |
| `EXPORT_GPL_SIMPLE_DEV_PM_OPS` | نفس السابق بـ GPL license |
| `UNIVERSAL_DEV_PM_OPS` | (deprecated) suspend+resume+runtime في ops واحد |
| `SIMPLE_DEV_PM_OPS` | (deprecated) استخدم `DEFINE_SIMPLE_DEV_PM_OPS` |

---

### Group 1: VT Console & CXL Helpers

هذه الـ functions مرتبطة بالـ platform-level checks قبل الـ system suspend — يعني بتحدد هل الـ console أو الـ CXL memory ممكن تأثر على الـ suspend path.

---

#### `pm_vt_switch_required`

```c
/* CONFIG_VT_CONSOLE_SLEEP only */
int pm_vt_switch_required(struct device *dev, bool required);
```

بتسجّل إن الـ `dev` ده محتاج (أو مش محتاج) الـ VT switch عند الـ suspend. الـ PM core بيجمع رأي كل الـ devices المسجّلة ويقرر هل يعمل switch للـ VT قبل النوم ولا لأ.

**Parameters:**
- `dev` — الـ device بتاعك اللي بتسجل عنه.
- `required` — `true` لو محتاج الـ switch، `false` لو مش محتاج.

**Return:** `0` لو نجح، error code سالب لو في مشكلة (memory allocation فشلت مثلاً).

**Key details:** مبني على refcounting داخلي. الـ `pm_vt_switch_unregister` لازم تتعمل في الـ remove path. لو `CONFIG_VT_CONSOLE_SLEEP` مش مفعّل، الـ inline stub بترجع `0` دايماً.

**Caller context:** Driver probe أو init — process context.

---

#### `pm_vt_switch_unregister`

```c
void pm_vt_switch_unregister(struct device *dev);
```

بتشيل الـ device من قائمة الـ VT switch tracking. لازم تتعمل في الـ `.remove` أو الـ `.disconnect` callback.

**Parameters:**
- `dev` — الـ device المسجّل مسبقاً.

**Return:** void.

**Key details:** لو `CONFIG_VT_CONSOLE_SLEEP` مش مفعّل، بتبقى no-op.

---

#### `cxl_mem_active`

```c
bool cxl_mem_active(void);
```

بتفحص لو في CXL memory region نشط حالياً. الـ PM core بتستخدمها عشان تمنع الـ system sleep لو الـ CXL memory لا تزال active — لأن الـ CXL memory بتتأثر بـ power-off وممكن يحصل data loss.

**Return:** `true` لو CXL memory نشط، `false` غير كده.

**Caller context:** الـ suspend entry path — لازم process context.

---

### Group 2: Subsystem Data Management

الـ `pm_subsys_data` هو struct مُشترك بين الـ PM subsystems (clk framework، PM domains). الـ functions دي بتدير الـ lifecycle بتاعته بـ refcounting.

---

#### `dev_pm_get_subsys_data`

```c
int dev_pm_get_subsys_data(struct device *dev);
```

بتتحقق لو `dev->power.subsys_data` موجود — لو لأ، بتعمل `kzalloc` وبتعمّره وبتبدأ الـ `spinlock`. كل call بتزوّد الـ refcount بواحد. الـ PM clock framework والـ PM domains بيستخدموها لتخزين بياناتهم في نفس الـ struct.

**Parameters:**
- `dev` — الـ device اللي محتاج الـ subsys data.

**Return:** `0` لو نجح، `-ENOMEM` لو ما قدرش يعمل allocate.

**Key details:**
- الـ `dev->power.lock` spinlock بيتأخذ أثناء الـ assignment.
- الـ struct بيتحذف تلقائياً لما الـ `refcount` يوصل صفر في `dev_pm_put_subsys_data`.

**Caller context:** عادةً من الـ bus type أو PM domain أثناء الـ device registration.

---

#### `dev_pm_put_subsys_data`

```c
void dev_pm_put_subsys_data(struct device *dev);
```

بتنقص الـ refcount — لو وصل صفر بتعمل `kfree` للـ `pm_subsys_data` وبتعمل `dev->power.subsys_data = NULL`.

**Parameters:**
- `dev` — الـ device اللي عايز تحرر الـ subsys data بتاعته.

**Return:** void.

**Key details:** spinlock بيتأخذ أثناء الـ decrement والـ NULL assignment. لو `subsys_data` NULL من الأول، الـ function بتعمل early return من غير أي عمل.

---

### Group 3: DPM List Lock/Unlock

الـ `dpm_list` هي الـ global list اللي فيها كل الـ devices المسجّلة للـ PM. الـ lock/unlock بيحميها من الـ concurrent access.

---

#### `device_pm_lock` / `device_pm_unlock`

```c
void device_pm_lock(void);
void device_pm_unlock(void);
```

بيأخذوا/بيحرروا الـ `dpm_list_mtx` — الـ mutex اللي بيحمي الـ `dpm_list`. لازم تتاخذوا قبل أي تعديل على الـ list أو traversal خارج الـ PM core نفسه.

**Caller context:** Process context فقط — mutex.

---

### Group 4: DPM Suspend/Resume Pipeline

دي الـ engine الأساسية للـ system sleep. الـ PM core بيستدعيها بالترتيب في الـ suspend/resume paths.

---

#### `dpm_prepare`

```c
int dpm_prepare(pm_message_t state);
```

بتعمل iterate على الـ `dpm_list` من الأول لآخره وبتستدعي `->prepare()` لكل device. لو أي device رجع `-EAGAIN`، الـ PM core بيعيد المحاولة لاحقاً. لو رجع positive number، الـ PM core ممكن يعمل `direct_complete` optimization (اللي بيخلي الـ device يفضل في runtime suspend من غير ما يتحرك).

**Parameters:**
- `state` — الـ `pm_message_t` اللي بيحدد نوع الـ transition (SUSPEND, FREEZE, HIBERNATE...).

**Return:** `0` لو كل الـ devices اتجهزت، error code لو في device فشل.

**Key details:**
- الـ `dpm_list_mtx` لازم يكون مش محجوز عند الـ call.
- في حالة الـ failure، الـ PM core بيعمل rollback — بيستدعي `->complete()` على كل اللي اتحضّروا.

**Pseudocode:**
```c
// Simplified flow
dpm_prepare(state):
    lock(dpm_list_mtx)
    list_for_each_entry(dev, &dpm_list, power.entry):
        unlock(dpm_list_mtx)
        ret = device_prepare(dev, state)  // calls ->prepare()
        if ret < 0 and ret != -EAGAIN:
            // rollback: call complete() on already-prepared
            return ret
        lock(dpm_list_mtx)
    unlock(dpm_list_mtx)
    return 0
```

---

#### `dpm_suspend`

```c
int dpm_suspend(pm_message_t state);
```

بتستدعي `->suspend()` لكل device بعد ما `dpm_prepare` اتنجح. الـ devices بتتعمل بالعكس (آخر device في الـ list أول حاجة) عشان الـ children يتسسبند قبل الـ parents.

**Return:** `0` لو كل حاجة تمام، error لو في device فشل في الـ suspend.

**Key details:** في حالة الـ error، الـ PM core بيعمل partial resume للـ devices اللي اتسسبندت فعلاً.

---

#### `dpm_suspend_late`

```c
int dpm_suspend_late(pm_message_t state);
```

الـ second phase من الـ suspend — بتستدعي `->suspend_late()`. بتعمل نفس الـ reverse-order traversal. بعد الـ `dpm_suspend_late`، الـ system بيكون في حالة شبه-suspended — معظم الـ devices بطّلت I/O.

---

#### `dpm_suspend_noirq`

```c
int dpm_suspend_noirq(pm_message_t state);
```

**الـ last suspend phase.** بتتنفذ مع الـ IRQs disabled (أو الـ interrupts mostratos عبر الـ `disable_irq`). بتستدعي `->suspend_noirq()` لكل device. بعد ما تخلص، المفروض كل الـ devices تكون في low-power state ومستعدة للـ system sleep.

**Key details:** لو الـ device بيولّد wakeup signals، لازم يتكونفجر هنا. الـ irq_safe devices بيتعاملوا بشكل مختلف (مش بيتاخد IRQ disable عنهم).

---

#### `dpm_suspend_start` / `dpm_suspend_end`

```c
int dpm_suspend_start(pm_message_t state);  // = dpm_prepare + dpm_suspend
int dpm_suspend_end(pm_message_t state);    // = dpm_suspend_late + dpm_suspend_noirq
```

Convenience wrappers بتجمع الـ phases مع بعض. الـ `kernel/power/suspend.c` بتستدعيهم مباشرة في الـ system sleep path.

---

#### `dpm_resume_noirq`

```c
void dpm_resume_noirq(pm_message_t state);
```

**أول خطوة في الـ resume** — بتتنفذ مع الـ IRQs disabled. بتستدعي `->resume_noirq()` بالترتيب الأمامي (parents أول). بعدين بتعيد تفعيل الـ IRQs.

---

#### `dpm_resume_early`

```c
void dpm_resume_early(pm_message_t state);
```

بتستدعي `->resume_early()` بالترتيب الأمامي. الـ clocks والـ regulators بيتعملوا enable هنا عادةً قبل الـ full resume.

---

#### `dpm_resume`

```c
void dpm_resume(pm_message_t state);
```

بتستدعي `->resume()` لكل device بالترتيب الأمامي. بعد ما تخلص، الـ devices المفروض تكون functional تاني. الـ error codes من `->resume()` مش بتوقف الـ resume — بس بتتطبع في الـ dmesg.

---

#### `dpm_complete`

```c
void dpm_complete(pm_message_t state);
```

**آخر خطوة في الـ resume pipeline** — بتستدعي `->complete()` لكل device. دي مش بترجع error لأن `->complete()` هو void. بتتأكد إن الـ `power.is_prepared` flag يتعمل clear.

---

#### `dpm_resume_start` / `dpm_resume_end`

```c
void dpm_resume_start(pm_message_t state);  // = dpm_resume_noirq + dpm_resume_early
void dpm_resume_end(pm_message_t state);    // = dpm_resume + dpm_complete
```

Symmetrical مع الـ suspend counterparts.

---

#### `device_pm_wait_for_dev`

```c
int device_pm_wait_for_dev(struct device *sub, struct device *dev);
```

بتخلي الـ device `sub` ينتظر لحد ما الـ device `dev` يخلص الـ PM operation الحالية. مفيدة لما في dependencies غير standard بين الـ devices مش معبّر عنها في الـ device tree.

**Parameters:**
- `sub` — الـ device اللي هينتظر.
- `dev` — الـ device اللي بينتظر يخلص.

**Return:** `0` لو الانتظار اتم بنجاح، `-EBUSY` لو الانتظار مش ممكن.

**Key details:** بتستخدم `dev->power.completion` internally. لازم process context.

---

#### `dpm_for_each_dev`

```c
void dpm_for_each_dev(void *data, void (*fn)(struct device *, void *));
```

بتعمل iterate على كل الـ devices في الـ `dpm_list` وبتستدعي `fn` على كل واحد. خدها زي `list_for_each_entry` بس مع الـ dpm_list mutex.

**Parameters:**
- `data` — pointer بيتمرر لـ `fn` بدون تعديل.
- `fn` — الـ callback اللي بيتستدعى لكل device.

**Caller context:** Process context — بتأخذ mutex داخلياً.

---

### Group 5: Error Reporting

---

#### `__suspend_report_result` / `suspend_report_result`

```c
void __suspend_report_result(const char *function, struct device *dev, void *fn, int ret);

/* macro wrapper: */
#define suspend_report_result(dev, fn, ret) \
    do { __suspend_report_result(__func__, dev, fn, ret); } while (0)
```

لو `ret != 0`، بتطبع `dev_err` message فيها اسم الـ function اللي فشلت واسم الـ device والـ error code. الـ drivers بيستخدموها في الـ suspend callbacks عشان يعملوا standardized logging.

**Parameters:**
- `function` — `__func__` من الـ caller (اسم الـ function).
- `dev` — الـ device اللي فشل فيه الـ suspend.
- `fn` — pointer للـ callback اللي فشل (بيتستخدم في الـ log فقط).
- `ret` — الـ error code.

**Key details:** مش بتوقف التنفيذ — بس logging. الـ macro بيمرر `__func__` تلقائياً.

---

### Group 6: Generic PM Callbacks

الـ `pm_generic_*` functions هي implementations جاهزة للـ `dev_pm_ops` callbacks. الـ bus types والـ PM domains بتستخدمها لما مش محتاجة behavior خاص — بتعمل delegate للـ driver callbacks مباشرة.

**المنطق العام لكل function:**

```c
int pm_generic_suspend(struct device *dev):
    drv = dev->driver
    if drv and drv->pm and drv->pm->suspend:
        return drv->pm->suspend(dev)
    return 0
```

---

#### `pm_generic_prepare`

```c
int pm_generic_prepare(struct device *dev);
```

بتستدعي `dev->driver->pm->prepare(dev)` لو موجود. لو الـ driver مش عنده `->prepare()`، بترجع `0`. بتكون نقطة الدخول للـ subsystem-level `->prepare()` اللي بيستدعي driver callback.

**Return:** return value من الـ driver callback، أو `0` لو مفيش callback.

---

#### `pm_generic_suspend` / `pm_generic_resume`

```c
int pm_generic_suspend(struct device *dev);
int pm_generic_resume(struct device *dev);
```

بيستدعوا `drv->pm->suspend(dev)` و `drv->pm->resume(dev)` على الترتيب. الـ `pm_generic_resume` لو رجع error، الـ PM core بيطبعه بس مش بيوقف الـ resume.

---

#### `pm_generic_suspend_late` / `pm_generic_resume_early`

```c
int pm_generic_suspend_late(struct device *dev);
int pm_generic_resume_early(struct device *dev);
```

نفس المنطق لكن لـ `->suspend_late()` و `->resume_early()` callbacks.

---

#### `pm_generic_suspend_noirq` / `pm_generic_resume_noirq`

```c
int pm_generic_suspend_noirq(struct device *dev);
int pm_generic_resume_noirq(struct device *dev);
```

بيستدعوا الـ `noirq` variants — بيتنفذوا مع الـ IRQs disabled.

---

#### `pm_generic_freeze` / `pm_generic_thaw`

```c
int pm_generic_freeze(struct device *dev);
int pm_generic_thaw(struct device *dev);
```

الـ hibernation-specific variants. الـ `freeze` بتوقف الـ device بدون تغيير الـ power state — لازم تحفظ الـ device state في الـ RAM. الـ `thaw` بترجعه لحالة ما قبل الـ freeze.

---

#### `pm_generic_freeze_noirq` / `pm_generic_thaw_noirq`

```c
int pm_generic_freeze_noirq(struct device *dev);
int pm_generic_thaw_noirq(struct device *dev);
```

الـ noirq phase من الـ freeze/thaw cycle.

---

#### `pm_generic_poweroff` / `pm_generic_restore`

```c
int pm_generic_poweroff(struct device *dev);
int pm_generic_restore(struct device *dev);
```

بعد ما الـ hibernation image اتحفظ، الـ `poweroff` بتوقف الـ devices قبل ما الـ machine يتطفي. الـ `restore` بتشغّل الـ devices تاني بعد restore الـ hibernation image.

---

#### `pm_generic_poweroff_late` / `pm_generic_poweroff_noirq`

```c
int pm_generic_poweroff_late(struct device *dev);
int pm_generic_poweroff_noirq(struct device *dev);
```

الـ late وـnoirq phases من الـ poweroff sequence.

---

#### `pm_generic_restore_early` / `pm_generic_restore_noirq`

```c
int pm_generic_restore_early(struct device *dev);
int pm_generic_restore_noirq(struct device *dev);
```

الـ early وـnoirq phases من الـ restore sequence.

---

#### `pm_generic_complete`

```c
void pm_generic_complete(struct device *dev);
```

بتستدعي `drv->pm->complete(dev)` لو موجود. الـ void return — مفيش error handling هنا.

---

### Group 7: Skip Optimization Helpers

---

#### `dev_pm_skip_resume`

```c
bool dev_pm_skip_resume(struct device *dev);
```

بتحدد هل ممكن نتخطى الـ resume callbacks (early + noirq) للـ device ده. بترجع `true` لو:
- الـ `DPM_FLAG_MAY_SKIP_RESUME` مضبوط، **و**
- مفيش حاجة تستلزم الـ resume (مثلاً مفيش pending wakeup).

**Key details:** مرتبطة بـ `dev->power.may_skip_resume` اللي بيتضبط من الـ subsystem. الـ PM core بتستدعيها في الـ `dpm_resume_noirq` path.

---

#### `dev_pm_skip_suspend`

```c
bool dev_pm_skip_suspend(struct device *dev);
```

**Symmetric** للـ `dev_pm_skip_resume`. بترجع `true` لو الـ device ممكن يفضل في الـ runtime suspend حالته من غير ما نستدعي أي suspend callback — ده الـ `direct_complete` optimization.

**Key details:** بتعتمد على `DPM_FLAG_SMART_SUSPEND` وإن الـ device كان في `RPM_SUSPENDED` قبل الـ system suspend.

---

### Group 8: PM Ops Definition Macros

الـ macros دي بتبسّط تعريف وتصدير `dev_pm_ops` structures بشكل portable — بتتعامل مع الـ `CONFIG_PM` و `CONFIG_PM_SLEEP` guards تلقائياً.

---

#### `pm_ptr` و `pm_sleep_ptr`

```c
#define pm_ptr(_ptr)       PTR_IF(IS_ENABLED(CONFIG_PM), (_ptr))
#define pm_sleep_ptr(_ptr) PTR_IF(IS_ENABLED(CONFIG_PM_SLEEP), (_ptr))
```

لو الـ CONFIG مش مفعّل، بيرجعوا `NULL` عشان الـ compiler يعرف يـdead-code-eliminate الـ function تماماً. الـ `PTR_IF` هو:
```c
#define PTR_IF(cond, ptr)  ((cond) ? (ptr) : NULL)
```

**الاستخدام الشائع:**
```c
static const struct dev_pm_ops my_pm_ops = {
    SYSTEM_SLEEP_PM_OPS(my_suspend, my_resume)
    RUNTIME_PM_OPS(my_rt_suspend, my_rt_resume, NULL)
};

/* لو CONFIG_PM_SLEEP مش enabled، my_suspend/my_resume بيبقوا NULL */
```

---

#### `SYSTEM_SLEEP_PM_OPS` / `LATE_SYSTEM_SLEEP_PM_OPS` / `NOIRQ_SYSTEM_SLEEP_PM_OPS`

```c
#define SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn) \
    .suspend   = pm_sleep_ptr(suspend_fn), \
    .resume    = pm_sleep_ptr(resume_fn),  \
    .freeze    = pm_sleep_ptr(suspend_fn), \
    .thaw      = pm_sleep_ptr(resume_fn),  \
    .poweroff  = pm_sleep_ptr(suspend_fn), \
    .restore   = pm_sleep_ptr(resume_fn),
```

بتعبئ الـ system sleep callbacks (suspend+hibernate+restore) بـ function واحدة للـ suspend وواحدة للـ resume. كل الـ pointers محمية بـ `pm_sleep_ptr`.

**الـ LATE variant** بتعبئ `suspend_late/resume_early/freeze_late/thaw_early/poweroff_late/restore_early`.

**الـ NOIRQ variant** بتعبئ الـ noirq set.

---

#### `RUNTIME_PM_OPS`

```c
#define RUNTIME_PM_OPS(suspend_fn, resume_fn, idle_fn) \
    .runtime_suspend = suspend_fn, \
    .runtime_resume  = resume_fn,  \
    .runtime_idle    = idle_fn,
```

مفيش `pm_ptr` هنا! الـ runtime PM callbacks مش محتاجة guards عشان الـ runtime PM مستقل عن الـ sleep — الـ device framework بيتعامل مع `NULL` callbacks.

---

#### `DEFINE_SIMPLE_DEV_PM_OPS`

```c
#define DEFINE_SIMPLE_DEV_PM_OPS(name, suspend_fn, resume_fn) \
    _DEFINE_DEV_PM_OPS(name, suspend_fn, resume_fn, NULL, NULL, NULL)
```

**الأكثر استخداماً.** بتعرّف `const struct dev_pm_ops name` بـ system sleep callbacks فقط (suspend+resume لكل الـ transitions). الـ runtime PM callbacks بتبقى NULL.

**مثال:**
```c
static int my_drv_suspend(struct device *dev) { /* ... */ return 0; }
static int my_drv_resume(struct device *dev)  { /* ... */ return 0; }

DEFINE_SIMPLE_DEV_PM_OPS(my_drv_pm_ops, my_drv_suspend, my_drv_resume);

static struct platform_driver my_driver = {
    .driver = {
        .pm = pm_ptr(&my_drv_pm_ops),
    },
};
```

---

#### `DEFINE_NOIRQ_DEV_PM_OPS`

```c
#define DEFINE_NOIRQ_DEV_PM_OPS(name, suspend_fn, resume_fn) \
const struct dev_pm_ops name = { \
    NOIRQ_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn) \
}
```

لو الـ driver محتاج callbacks تتنفذ وقت الـ IRQs disabled فقط — مفيد للـ drivers اللي بتتعامل مع shared IRQ lines.

---

#### `EXPORT_SIMPLE_DEV_PM_OPS` / `EXPORT_GPL_SIMPLE_DEV_PM_OPS`

```c
#define EXPORT_SIMPLE_DEV_PM_OPS(name, suspend_fn, resume_fn) \
    EXPORT_DEV_SLEEP_PM_OPS(name) = { \
        SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn) \
    }
```

زي `DEFINE_SIMPLE_DEV_PM_OPS` بس بيعمل export للـ symbol كمان. لو `CONFIG_PM_SLEEP` مش مفعّل، الـ symbol بيتعرّف بس بيتجاهل (`__maybe_unused` static).

**اختار:** `EXPORT_GPL_SIMPLE_DEV_PM_OPS` للـ GPL modules، `EXPORT_SIMPLE_DEV_PM_OPS` للباقي.

---

#### `SET_SYSTEM_SLEEP_PM_OPS` / `SET_RUNTIME_PM_OPS` (deprecated-safe wrappers)

```c
#ifdef CONFIG_PM_SLEEP
#define SET_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn) \
    SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn)
#else
#define SET_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn)
#endif
```

الـ `SET_*` variants هي الـ guarded version — بتحط فراغ لو الـ CONFIG مش مفعّل. استخدامها في الـ `SIMPLE_DEV_PM_OPS` و `UNIVERSAL_DEV_PM_OPS` (اللي اتنوا deprecated).

**الفرق الجوهري:**
- `SYSTEM_SLEEP_PM_OPS` + `pm_sleep_ptr` = compile always, NULL at runtime لو disabled.
- `SET_SYSTEM_SLEEP_PM_OPS` = compile away completely لو disabled.

الـ modern approach هو الأول (مع `pm_sleep_ptr`) عشان أحسن للـ dead-code elimination.

---

### الصورة الكاملة — PM Transition Flow

```
System Suspend:
┌──────────────────────────────────────────────────────┐
│  dpm_suspend_start()                                  │
│    ├─ dpm_prepare()    → calls ->prepare() per dev   │
│    └─ dpm_suspend()    → calls ->suspend() per dev   │
│  dpm_suspend_end()                                    │
│    ├─ dpm_suspend_late()   → ->suspend_late()        │
│    └─ dpm_suspend_noirq()  → ->suspend_noirq()       │
│                              [IRQs disabled]          │
│  <<< machine enters sleep >>>                        │
│  dpm_resume_start()                                  │
│    ├─ dpm_resume_noirq()  → ->resume_noirq()         │
│    │                        [IRQs disabled]           │
│    └─ dpm_resume_early()  → ->resume_early()         │
│  dpm_resume_end()                                    │
│    ├─ dpm_resume()    → calls ->resume() per dev     │
│    └─ dpm_complete()  → calls ->complete() per dev   │
└──────────────────────────────────────────────────────┘

Hibernation adds:
  Suspend path: freeze / freeze_late / freeze_noirq
                → save image →
                poweroff / poweroff_late / poweroff_noirq
  Resume path:  restore_noirq / restore_early / restore
                thaw_noirq / thaw_early / thaw
```

---

### جدول الـ dev_pm_ops Callbacks — متى تُستدعى

| Callback | متى | IRQs | ممكن يُسبّب rollback؟ |
|---|---|---|---|
| `prepare` | أول خطوة suspend | enabled | نعم |
| `suspend` | بعد prepare | enabled | نعم |
| `suspend_late` | بعد كل suspend | enabled | نعم |
| `suspend_noirq` | آخر suspend phase | disabled | نعم |
| `resume_noirq` | أول resume phase | disabled | لا |
| `resume_early` | بعد resume_noirq | enabled | لا |
| `resume` | بعد resume_early | enabled | لا (errors logged only) |
| `complete` | آخر resume phase | enabled | N/A (void) |
| `freeze` | قبل hibernation image | enabled | نعم |
| `freeze_noirq` | بعد freeze_late | disabled | نعم |
| `thaw` | بعد فشل/نجاح image creation | enabled | لا |
| `poweroff` | بعد حفظ image | enabled | نعم |
| `restore` | بعد load hibernation image | enabled | لا |
| `runtime_suspend` | device idle (runtime PM) | varies | نعم |
| `runtime_resume` | device needed (runtime PM) | varies | نعم |
| `runtime_idle` | device appears idle | varies | N/A |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs — المدخلات المهمة وإزاي تقراها

**الـ PM subsystem** بيكتب بيانات كتير في `/sys/kernel/debug/` وفي `/sys/kernel/debug/pm_debug/`:

```bash
# شوف كل مدخلات الـ PM في debugfs
ls /sys/kernel/debug/pm_debug/

# الـ suspend stats — بيوضح عدد مرات الـ suspend النجاح والفشل
cat /sys/kernel/debug/suspend_stats

# الـ wakeup sources — مين صحّى الـ system
cat /sys/kernel/debug/wakeup_sources

# الـ power domains hierarchy
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary

# الـ device power states في الـ DPM list
cat /sys/kernel/debug/device_component/*/
```

**تفسير output الـ suspend_stats:**

```
success:          5       # عدد مرات الـ suspend الناجحة
fail:             2       # فشل في أي مرحلة
failed_freeze:    0       # فشل في مرحلة الـ freeze (hibernation)
failed_prepare:   1       # فشل في dev_pm_ops.prepare()
failed_suspend:   0       # فشل في dev_pm_ops.suspend()
failed_suspend_noirq: 0
failed_suspend_late:  1
last_failed_dev:  0000:00:1f.3  # آخر device فشل
last_failed_errno: -16   # EBUSY
last_failed_step: prepare
```

---

#### 2. sysfs — المدخلات الأساسية

كل device في `/sys/bus/.../devices/<device>/power/` ليه الـ attributes دي:

| المدخل | القيم | الوصف |
|--------|-------|-------|
| `control` | `auto` / `on` | تحكم في runtime PM — `on` بيعطله |
| `runtime_status` | `active` / `suspended` / `suspending` / `resuming` | الحالة الحالية لـ `rpm_status` |
| `runtime_enabled` | `enabled` / `disabled` / `forbidden` | هل runtime PM شغال؟ |
| `runtime_suspended_time` | microseconds | الوقت الكلي في الـ suspend |
| `runtime_active_time` | microseconds | الوقت الكلي في الـ active |
| `runtime_usage` | integer | قيمة `usage_count` في `dev_pm_info` |
| `autosuspend_delay_ms` | milliseconds | قيمة `autosuspend_delay` |
| `wakeup` | `enabled` / `disabled` | هل الـ wakeup مفعّل؟ |
| `wakeup_count` | integer | عدد مرات الـ wakeup |
| `wakeup_abort_count` | integer | عدد مرات إلغاء الـ suspend بسببه |
| `async` | `enabled` / `disabled` | الـ `async_suspend` flag |

```bash
# مثال عملي — شوف device معين
DEV=/sys/bus/platform/devices/my_device
cat $DEV/power/runtime_status
cat $DEV/power/runtime_usage
cat $DEV/power/control

# عطّل runtime PM مؤقتاً للـ debug
echo on > $DEV/power/control

# شغّل runtime PM تاني
echo auto > $DEV/power/control
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

**الـ PM subsystem** عنده tracepoints جاهزة في `power` category:

```bash
# شوف كل events متاحة
ls /sys/kernel/debug/tracing/events/power/

# Events المهمة:
# device_pm_callback_start  — بداية تنفيذ أي callback في dev_pm_ops
# device_pm_callback_end    — نهاية التنفيذ + الـ error code
# rpm_suspend               — بداية runtime_suspend
# rpm_resume                — بداية runtime_resume
# rpm_idle                  — استدعاء runtime_idle
# suspend_resume            — مراحل الـ system suspend كاملة
# wakeup_source_activate    — تفعيل wakeup source
# wakeup_source_deactivate  — إلغاء تفعيل wakeup source
```

```bash
# تفعيل كل PM events
echo 1 > /sys/kernel/debug/tracing/events/power/enable

# أو events محددة
echo 1 > /sys/kernel/debug/tracing/events/power/device_pm_callback_start/enable
echo 1 > /sys/kernel/debug/tracing/events/power/device_pm_callback_end/enable
echo 1 > /sys/kernel/debug/tracing/events/power/rpm_suspend/enable
echo 1 > /sys/kernel/debug/tracing/events/power/rpm_resume/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# افعل الـ suspend
echo mem > /sys/power/state

# وقف الـ trace وشوف النتيجة
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

**مثال على output:**

```
kworker/0:1-45   [000]  device_pm_callback_start: my_device, ops: suspend, event: 2
kworker/0:1-45   [000]  device_pm_callback_end:   my_device, ops: suspend, error: 0
kworker/0:1-45   [000]  rpm_suspend: my_device flags=0x4
```

**الـ event=2** يساوي `PM_EVENT_SUSPEND`، **error=0** يعني نجاح.

---

#### 4. printk / dynamic debug

```bash
# تفعيل debug messages لـ PM core كله
echo "file drivers/base/power/*.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ runtime PM بالذات
echo "file drivers/base/power/runtime.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لـ device معين (باسم الـ driver)
echo "module my_driver +p" > /sys/kernel/debug/dynamic_debug/control

# شوف إيه اللي مفعّل حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep power

# تفعيل messages مع الـ function name والـ line number
echo "file drivers/base/power/main.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
# p = print, f = function name, l = line number, m = module name, t = thread id
```

**في الكود:** لو بتكتب driver، استخدم:

```c
#define pr_fmt(fmt) "my_driver PM: " fmt
#include <linux/dev_printk.h>

/* في suspend callback */
static int my_suspend(struct device *dev)
{
    dev_dbg(dev, "suspending, power_state=%d\n",
            dev->power.power_state.event);
    dev_dbg(dev, "runtime_status=%d usage=%d\n",
            dev->power.runtime_status,
            atomic_read(&dev->power.usage_count));
    return 0;
}
```

---

#### 5. Kernel Config Options للـ Debugging

| الـ Config | الوصف |
|-----------|-------|
| `CONFIG_PM_DEBUG` | يفعّل `/sys/power/pm_debug_messages` ويكثّر الـ printk في PM core |
| `CONFIG_PM_SLEEP_DEBUG` | debug messages إضافية لـ system suspend/resume |
| `CONFIG_PM_ADVANCED_DEBUG` | يضيف attributes إضافية في sysfs مثل `pm_qos_resume_latency_us` |
| `CONFIG_PM_TRACE` | يكتب PM trace في الـ RTC لكشف سبب فشل الـ resume بعد hard reset |
| `CONFIG_PM_TRACE_RTC` | نفس السابق مع الحفظ في RTC register |
| `CONFIG_DEBUG_LOCK_ALLOC` | يكشف lock ordering bugs في PM callbacks |
| `CONFIG_LOCKDEP` | يتبع كل lock acquisition — مهم لكشف deadlocks في PM paths |
| `CONFIG_DEBUG_MUTEXES` | debug إضافي لـ `struct mutex` المستخدم في `pm_subsys_data` |
| `CONFIG_PM_CLK` | تفعيل دعم clock management في PM (يظهر `clock_list` في `pm_subsys_data`) |
| `CONFIG_PM_GENERIC_DOMAINS` | تفعيل generic power domains debugging |
| `CONFIG_PM_GENERIC_DOMAINS_DEBUG` | يضيف `/sys/kernel/debug/pm_genpd/` كاملاً |
| `CONFIG_PM_OPP` | Operating Performance Points debugging |
| `CONFIG_SUSPEND_FREEZER` | debug task freezing أثناء suspend |

```bash
# تفعيل PM debug messages في runtime
echo 1 > /sys/power/pm_debug_messages

# شوف الـ PM trace بعد resume فاشل
cat /sys/kernel/debug/suspend_stats
```

---

#### 6. أدوات خاصة بالـ Subsystem

**powertop** — يشوف consumption وأسباب الـ wakeups:

```bash
powertop --auto-tune          # optimize automatically
powertop --html=report.html   # HTML report
powertop --time=30            # monitor 30 seconds
```

**pm-utils / systemd:**

```bash
# systemd suspend stats
systemd-analyze
journalctl -b -1 -u systemd-suspend.service  # آخر suspend log

# شوف wakeup sources من systemd
journalctl | grep -E "(PM: suspend|PM: resume|ACPI)"
```

**rtcwake** — اختبار الـ suspend/resume آلياً:

```bash
# suspend لمدة 30 ثانية ثم resume تلقائي
rtcwake -m mem -s 30

# hibernate ثم resume
rtcwake -m disk -s 60
```

**pm-graph** — يرسم timeline للـ suspend/resume:

```bash
# تثبيت
apt install pm-graph

# تسجيل
sudo sleepgraph -m mem -rtcwake 15

# النتيجة: ملف HTML يوضح كل callback ووقته
```

---

#### 7. رسالة خطأ → السبب → الحل

| رسالة الخطأ | السبب | الحل |
|-------------|-------|------|
| `PM: Device <dev> failed to suspend: error -16` | الـ `suspend()` callback رجع `-EBUSY` — الـ device لسه شغال | تأكد إن الـ device quiescent قبل ما يرجع |
| `PM: Device <dev> failed to prepare: error -11` | الـ `prepare()` رجع `-EAGAIN` — race condition مع child registration | طبيعي — الـ PM core هيعيد المحاولة |
| `PM: late suspend of <dev> failed: -5` | الـ `suspend_late()` رجع `-EIO` — مشكلة hardware | شوف الـ hardware registers والـ clocks |
| `PM: noirq suspend of <dev> failed: -110` | `-ETIMEDOUT` — الـ device ما ردّش | راجع الـ interrupt masking والـ DMA |
| `Runtime PM: device <dev> has refcount > 0` | الـ `usage_count` مش صفر رغم الـ idle | في مكان بتعمل `pm_runtime_get()` من غير `pm_runtime_put()` |
| `PM: failed to freeze some drivers` | driver ما نفّذش الـ `freeze()` صح | فعّل `CONFIG_PM_SLEEP_DEBUG` وشوف مين |
| `PM: Some devices failed to power down` | الـ `poweroff()` callback فشل | تحقق من الـ wakeup source المفتوح |
| `PM: Basic memory bitmaps allocated` | Hibernation بدأ يخصص ذاكرة | طبيعي — مش خطأ |
| `PM: Restoring platform NVS memory` | Restore من hibernation | طبيعي |
| `suspend: PM: Device <dev> wakeup-capable` | الـ device registered كـ wakeup source | طبيعي لو `can_wakeup = true` |
| `PM: pm_trace: RTC time appears invalid` | `CONFIG_PM_TRACE` مفعّل لكن RTC مش شغال | استخدم RTC حقيقي أو عطّل الـ trace |
| `PM: active wakeup source: <name>` | في wakeup source مفتوح منع الـ suspend | `cat /sys/kernel/debug/wakeup_sources` وشوف مين |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
static int my_driver_suspend(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);

    /* تأكد إن usage count صح */
    WARN_ON(atomic_read(&dev->power.usage_count) != 0);

    /* تأكد إن الـ device مش في حالة غلط */
    WARN_ON(dev->power.runtime_status == RPM_SUSPENDING);

    /* تأكد إن الـ driver_flags متظبّطة صح */
    WARN_ON(dev->power.driver_flags & DPM_FLAG_NO_DIRECT_COMPLETE &&
            dev->power.driver_flags & DPM_FLAG_MAY_SKIP_RESUME);

    if (!priv) {
        dev_err(dev, "suspend called with NULL private data\n");
        dump_stack();  /* خد stack trace كامل */
        return -EINVAL;
    }

    /* WARN لو الـ hardware state مش متوقع */
    WARN_ON_ONCE(priv->hw_state != HW_STATE_IDLE);

    return 0;
}

static int my_driver_runtime_suspend(struct device *dev)
{
    /* نقطة مهمة: تأكد إن مفيش pending work */
    WARN_ON(work_pending(&dev->power.work));

    /*
     * لو في حالة خطأ سابقة مسجلة في runtime_error،
     * ده مشكلة — يجب investigate
     */
    if (dev->power.runtime_error) {
        dev_warn(dev, "runtime_error=%d before suspend\n",
                 dev->power.runtime_error);
        dump_stack();
    }
    return 0;
}
```

**نقاط مهمة للـ WARN_ON في PM:**

```c
/* في complete() callback — تأكد من direct_complete */
void my_complete(struct device *dev)
{
    /* لو direct_complete مش set، المفروض كانت في callbacks */
    if (!dev->power.direct_complete)
        WARN_ON(/* hardware state unexpected */);
}

/* في runtime_idle — تأكد من child_count */
int my_runtime_idle(struct device *dev)
{
    WARN_ON(atomic_read(&dev->power.child_count) < 0);
    if (atomic_read(&dev->power.child_count) > 0)
        return -EBUSY;
    return pm_runtime_autosuspend(dev);
}
```

---

### Hardware Level

---

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

الـ `dev_pm_info.runtime_status` المفروض يطابق حالة الـ hardware الفعلية:

```
rpm_status = RPM_ACTIVE   → الـ hardware مفروض powered on وشغال
rpm_status = RPM_SUSPENDED → الـ hardware مفروض في low power mode
```

```bash
# قارن الـ kernel state بالـ hardware
cat /sys/bus/platform/devices/my_dev/power/runtime_status
# لو قال "suspended"، تأكد من الـ hardware registers إن الـ device فعلاً suspended

# شوف الـ power domain state
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
```

**مثال عملي على SoC:**

```
Domain           Status
------------------------------------------
pd_core          on           # core power domain
pd_display       off          # display power domain suspended
pd_audio         on           # audio active
```

---

#### 2. Register Dump Techniques

```bash
# devmem2 — قراءة register مباشرة (يحتاج root)
# مثال: قراءة power control register على عنوان 0x10001000
devmem2 0x10001000 w

# /dev/mem — قراءة raw
dd if=/dev/mem bs=4 count=1 skip=$((0x10001000/4)) 2>/dev/null | xxd

# io tool (من package iotools)
io -4 0x10001000   # قراءة 32-bit register

# لو في MMIO mapped في /proc/iomem
cat /proc/iomem | grep -i "power"

# رؤية الـ ioremap regions
cat /proc/vmallocinfo | grep ioremap
```

**داخل kernel module للـ debugging:**

```c
/* قراءة register وطباعتها في كل مرحلة من PM */
static void dump_hw_pm_registers(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    u32 ctrl = readl(priv->base + PM_CTRL_REG);
    u32 status = readl(priv->base + PM_STATUS_REG);

    dev_dbg(dev, "PM_CTRL=0x%08x PM_STATUS=0x%08x\n", ctrl, status);
    dev_dbg(dev, "  power_state=%s clock_en=%d wakeup_en=%d\n",
            (ctrl & BIT(0)) ? "D0" : "D3",
            !!(ctrl & BIT(4)),
            !!(ctrl & BIT(8)));
}

static int my_suspend(struct device *dev)
{
    dump_hw_pm_registers(dev);  /* قبل */
    /* ... do suspend ... */
    dump_hw_pm_registers(dev);  /* بعد */
    return 0;
}
```

---

#### 3. Logic Analyzer / Oscilloscope

**ما تقيسه:**

| الإشارة | ما تدور عليه |
|---------|-------------|
| VDD core / VDD IO | وقت قطع وإعادة الطاقة مع PM callbacks |
| SUSPEND_ACK pin | تأكيد الـ hardware إنه suspend |
| WAKE_IRQ | الـ wake_irq من `dev_pm_info.wakeirq` |
| I2C / SPI bus | هل الـ device بيتكلم لما المفروض يكون suspended? |
| CLK lines | هل الـ clocks بتتقطع في timing صح؟ |

**نصائح عملية:**

```
1. Trigger على SUSPEND_ACK — قيس الـ latency من suspend_noirq() لحد الـ ACK
2. لو بتستخدم pm_subsys_data.clock_list — قيس CLK disable time
3. قارن الـ hrtimer (suspend_timer في dev_pm_info) بالـ actual hardware response
4. ابحث عن glitches في VDD أثناء الـ direct_complete optimization
```

---

#### 4. Hardware Issues وأنماطها في الـ Kernel Log

| المشكلة الـ Hardware | النمط في الـ Kernel Log |
|--------------------|------------------------|
| الـ power rail بيتأخر في الارتفاع | `resume of <dev> failed: -110` (ETIMEDOUT) |
| الـ clock مش بيتفعّل | `failed to enable clock` أو register reads returning 0xFFFFFFFF |
| الـ wakeup IRQ مش بيوصل | `PM: Wakeup pending, aborting suspend` مش بيظهر رغم trigger |
| الـ device ما بيردّش بعد power-on | `noirq resume of <dev> failed: -5` (EIO) |
| Race بين hardware reset والـ resume | `device <dev> reset during suspend` |
| الـ VDD drop أثناء suspend | `PM: Device <dev> has unknown PM state` |
| الـ autosuspend_delay قصير | `RPM: device <dev> already in suspend state` متكرر |

```bash
# شوف الـ PM errors في الـ kernel log
dmesg | grep -E "PM:|RPM:|suspend|resume" | grep -v "^$"

# فلتر الـ errors بس
dmesg --level=err,warn | grep -iE "pm|power|suspend|resume"

# شوف timing الـ suspend/resume
dmesg -T | grep -E "PM: (suspend|resume)" | head -20
```

---

#### 5. Device Tree Debugging

الـ DT بيأثر على:
- الـ wakeup-source property (يضبط `can_wakeup`)
- الـ power-domains (يربط الـ device بـ `dev_pm_domain`)
- الـ clocks المستخدمة في `pm_subsys_data.clock_list`

```bash
# تحقق إن الـ DT property وصلت للـ kernel
cat /sys/firmware/devicetree/base/soc/my_device/wakeup-source 2>/dev/null \
  && echo "wakeup-source property present" \
  || echo "wakeup-source NOT in DT"

# شوف الـ power domain المرتبط بالـ device
cat /sys/firmware/devicetree/base/soc/my_device/power-domains

# قارن الـ DT clock names بالـ actual clocks
cat /sys/firmware/devicetree/base/soc/my_device/clock-names

# تحقق من الـ wakeup-irqs في DT
cat /sys/firmware/devicetree/base/soc/my_device/wakeup-source
grep -r "wakeup-source" /sys/firmware/devicetree/base/soc/

# شوف الـ compiled DT من kernel
dtc -I fs /sys/firmware/devicetree/base/ 2>/dev/null | \
  grep -A 10 "my_device"
```

**مطابقة DT مع الـ hardware:**

```bash
# تأكد إن الـ PM domain موجود فعلاً
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary | grep my_pd

# لو مش موجود، الـ DT فيه خطأ في اسم الـ power domain
# راجع:
# - compatible string في الـ power controller node
# - phandle reference في الـ device node
```

**Overlay للـ debug:**

```dts
/* أضف هذا كـ overlay لتفعيل wakeup على device موجود */
/dts-v1/;
/plugin/;
&my_device {
    wakeup-source;
};
```

---

### Practical Commands

---

#### كل الأوامر جاهزة للـ copy-paste

**1. فحص شامل لـ device معين:**

```bash
#!/bin/bash
DEV_PATH="/sys/bus/platform/devices/my_device"
DEV_NAME="my_device"

echo "=== PM Status for $DEV_NAME ==="
echo "Runtime Status: $(cat $DEV_PATH/power/runtime_status 2>/dev/null)"
echo "Runtime Enabled: $(cat $DEV_PATH/power/runtime_enabled 2>/dev/null)"
echo "Usage Count: $(cat $DEV_PATH/power/runtime_usage 2>/dev/null)"
echo "Active Time: $(cat $DEV_PATH/power/runtime_active_time 2>/dev/null) us"
echo "Suspended Time: $(cat $DEV_PATH/power/runtime_suspended_time 2>/dev/null) us"
echo "Wakeup: $(cat $DEV_PATH/power/wakeup 2>/dev/null)"
echo "Wakeup Count: $(cat $DEV_PATH/power/wakeup_count 2>/dev/null)"
echo "Control: $(cat $DEV_PATH/power/control 2>/dev/null)"
echo "Async Suspend: $(cat $DEV_PATH/power/async 2>/dev/null)"
```

**2. تفعيل PM tracing كامل:**

```bash
#!/bin/bash
TRACE=/sys/kernel/debug/tracing

# صفّر الـ trace buffer
echo > $TRACE/trace

# فعّل PM events
for event in device_pm_callback_start device_pm_callback_end \
             rpm_suspend rpm_resume rpm_idle \
             suspend_resume wakeup_source_activate; do
    echo 1 > $TRACE/events/power/$event/enable 2>/dev/null
done

# ابدأ الـ trace
echo 1 > $TRACE/tracing_on
echo "Tracing enabled. Do your suspend/resume now."
echo "Then run: echo 0 > $TRACE/tracing_on && cat $TRACE/trace"
```

**3. تحليل الـ trace بعد الـ suspend:**

```bash
# بعد الـ trace
echo 0 > /sys/kernel/debug/tracing/tracing_on

# شوف callbacks اللي استغرقت أكتر من 100ms
cat /sys/kernel/debug/tracing/trace | \
  awk '/device_pm_callback_end/ {
    if ($NF != "error=0") print "FAILED:", $0
  }'

# فلتر errors بس
cat /sys/kernel/debug/tracing/trace | grep "error=[^0]"
```

**4. مراقبة الـ wakeup sources:**

```bash
# مراقبة في real-time
watch -n 1 'cat /sys/kernel/debug/wakeup_sources | \
  awk "NR==1 || \$5>0" | column -t'

# Output مثال:
# name          active_count  event_count  wakeup_count  active_since  total_time  max_time  last_change
# alarm              3             3            0             0          15000        5000      12345
# gpio-keys          1             1            1             0           1000        1000      67890
```

**5. اختبار الـ runtime PM يدوياً:**

```bash
DEV="/sys/bus/platform/devices/my_device"

# تفعيل runtime PM
echo auto > $DEV/power/control

# Force suspend
echo suspend > $DEV/power/state  # (لو متاح)

# أو عبر sysfs في بعض الـ drivers
cat $DEV/power/runtime_status    # المفروض يتغيّر لـ "suspended"

# Force resume
cat $DEV/uevent  # أي access للـ device يعمل runtime resume
```

**6. debugging الـ system suspend كامل:**

```bash
#!/bin/bash
# سجّل كل PM events أثناء suspend to RAM

TRACE=/sys/kernel/debug/tracing
LOG=/tmp/pm_trace_$(date +%s).txt

# Setup
echo > $TRACE/trace
echo 1 > $TRACE/events/power/enable
echo 1 > $TRACE/tracing_on

# Suspend لمدة 10 ثواني
echo mem > /sys/power/state &
sleep 10 &
wait

# Collect
echo 0 > $TRACE/tracing_on
cp $TRACE/trace $LOG
echo "Trace saved to $LOG"

# Analysis
echo "=== Failed callbacks ==="
grep "error=[^0]" $LOG

echo "=== Slowest callbacks (>50ms estimated) ==="
awk '/device_pm_callback_start/{dev=$0} /device_pm_callback_end/{print dev, "->", $0}' $LOG | head -20
```

**7. تشخيص مشكلة الـ usage_count:**

```bash
# لو الـ device ما بيعملش suspend رغم الـ idle
DEV="/sys/bus/platform/devices/my_device"
echo "Usage count: $(cat $DEV/power/runtime_usage)"
# لو > 0، في كود بيعمل pm_runtime_get() من غير pm_runtime_put()

# ابحث في الـ trace
cat /sys/kernel/debug/tracing/trace | grep "my_device" | grep "rpm_"

# فعّل lockdep للـ PM
echo 1 > /proc/sys/kernel/prove_locking  # لو lockdep مفعّل في الكرنل
```

**8. تشخيص hibernate:**

```bash
# شوف الـ hibernation state
cat /sys/power/disk         # الـ backend المستخدم (platform/shutdown/reboot)
cat /sys/power/image_size   # الحجم الأقصى للـ hibernation image

# فعّل hibernation debug
echo 1 > /sys/power/pm_trace

# Hibernate
echo disk > /sys/power/state

# بعد الـ resume، شوف الـ trace
dmesg | grep "PM: Starting trace"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — UART لا يشتغل بعد الـ suspend

#### العنوان
**الـ UART driver ما بيرجعش يشتغل صح بعد system suspend على industrial gateway**

#### السياق
شركة بتعمل industrial IoT gateway بالـ RK3562. الـ gateway بيتكلم مع sensors عن طريق UART. الـ gateway مفروض يعمل suspend-to-RAM كل ساعة لتوفير الطاقة، وبعدين يصحى لما يجي data جديد.

#### المشكلة
بعد أول suspend/resume cycle، الـ UART بيبقى صامت — مش بيستقبل ولا بيبعت. الـ `/dev/ttyS1` موجود، بس الـ data بتضيع.

#### التحليل
الـ engineer بيراجع الـ UART driver ويلاقي إن الـ `dev_pm_ops` معرف كده:

```c
/* الـ driver بيستخدم SIMPLE_DEV_PM_OPS القديم */
static SIMPLE_DEV_PM_OPS(rk_uart_pm_ops, rk_uart_suspend, rk_uart_resume);
```

الـ `SIMPLE_DEV_PM_OPS` — وهو macro موجود في `pm.h` — بيعمل التالي:

```c
/* من pm.h — السطر 437 */
#define SIMPLE_DEV_PM_OPS(name, suspend_fn, resume_fn) \
const struct dev_pm_ops __maybe_unused name = { \
    SET_SYSTEM_SLEEP_PM_OPS(suspend_fn, resume_fn) \
}
```

الـ `SET_SYSTEM_SLEEP_PM_OPS` بيملي فقط `.suspend` و`.resume`، لكن مش `.suspend_late` ولا `.resume_early`. النتيجة: الـ clock manager بيوقف الـ UART clock في الـ `suspend_late` phase قبل ما الـ UART يخلص تهدئته — race condition كلاسيكي.

لما بيبص على `dev_pm_info` في `pm.h`:

```c
/* السطر 666 — dev_pm_info */
struct dev_pm_info {
    pm_message_t  power_state;   /* الـ state الحالي */
    bool          is_suspended:1;
    bool          is_late_suspended:1;
    /* ... */
};
```

بيلاقي إن `is_late_suspended` = true بس `is_suspended` = false، يعني الـ suspend_late اشتغل قبل ما `.suspend` يخلص تسجيل الـ state.

#### الحل
الانتقال من `SIMPLE_DEV_PM_OPS` المهجور لـ `DEFINE_SIMPLE_DEV_PM_OPS` الجديد، مع إضافة `.suspend_late` صريح:

```c
/* الحل: استخدام LATE_SYSTEM_SLEEP_PM_OPS بدل SIMPLE */
static const struct dev_pm_ops rk_uart_pm_ops = {
    SYSTEM_SLEEP_PM_OPS(rk_uart_suspend, rk_uart_resume)
    LATE_SYSTEM_SLEEP_PM_OPS(rk_uart_suspend_late, rk_uart_resume_early)
};

static int rk_uart_suspend_late(struct device *dev)
{
    struct rk_uart *uart = dev_get_drvdata(dev);
    /* flush TX FIFO completely before clocks die */
    rk_uart_wait_tx_empty(uart);
    clk_disable(uart->clk);
    return 0;
}

static int rk_uart_resume_early(struct device *dev)
{
    struct rk_uart *uart = dev_get_drvdata(dev);
    clk_enable(uart->clk);
    rk_uart_reinit(uart); /* restore baud rate, FIFO config */
    return 0;
}
```

للتشخيص السريع:
```bash
# شوف الـ pm status للـ UART device
cat /sys/bus/platform/devices/fe650000.serial/power/runtime_status

# تتبع الـ PM callbacks
echo 1 > /sys/power/pm_debug_messages
dmesg | grep -i "uart\|serial" | grep -i "suspend\|resume"
```

#### الدرس المستفاد
الـ `SIMPLE_DEV_PM_OPS` — وحتى `DEFINE_SIMPLE_DEV_PM_OPS` — بيملوا فقط الـ `.suspend/.resume` الأساسية. لو عندك clocks أو DMA بتتوقف في الـ `late` phase، لازم تعرّف `.suspend_late/.resume_early` بشكل صريح عن طريق `LATE_SYSTEM_SLEEP_PM_OPS`.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — تهنيج عند الـ hibernate

#### العنوان
**الـ HDMI encoder بيعمل kernel panic أثناء hibernation image creation**

#### السياق
منتج Android TV box بالـ Allwinner H616. القرار إن الـ box يدعم suspend-to-disk (hibernation) عشان يحفظ الـ state لو انقطع التيار. الـ HDMI driver كتبه vendor وما فيهوش أي دعم لـ hibernate.

#### المشكلة
لما المستخدم يعمل:
```bash
echo disk > /sys/power/state
```
الـ kernel بيعمل panic في وسط الـ `freeze` phase مع:
```
BUG: sleeping function called from invalid context
```

#### التحليل
الـ engineer بيبص على الـ HDMI driver ويلاقي:

```c
/* vendor driver — المشكلة هنا */
static const struct dev_pm_ops h616_hdmi_pm_ops = {
    .suspend  = h616_hdmi_suspend,
    .resume   = h616_hdmi_resume,
    /* .freeze و .thaw غايبين! */
};
```

الـ PM core لما يشوف `.freeze` = NULL، بيستدعي `pm_generic_freeze()` اللي موجود في `pm.h`:

```c
/* pm.h السطر 849 */
extern int pm_generic_freeze(struct device *dev);
```

الـ `pm_generic_freeze` بيشتغل عادي، لكن المشكلة إن الـ `h616_hdmi_suspend` نفسه بيعمل `mutex_lock` مع GFP_KERNEL allocation في السياق الغلط. لما `freeze` بيتكلم على الـ `suspend` callback عن طريق الـ subsystem، الـ context بيبقى مجمّد (tasks frozen) وأي sleeping call ممنوعة.

ببص على `pm_message_t`:

```c
/* pm.h السطر 60 */
typedef struct pm_message {
    int event;
} pm_message_t;
```

ولو كان الـ driver بيفحص `PMSG_NO_WAKEUP`:

```c
/* pm.h السطر 574 */
#define PMSG_NO_WAKEUP(msg) \
    (((msg).event & (PM_EVENT_FREEZE | PM_EVENT_QUIESCE)) != 0)
```

كان ممكن يتصرف صح. لكن الـ vendor ما بصش على الـ event type خالص.

#### الحل

```c
static int h616_hdmi_freeze(struct device *dev)
{
    struct h616_hdmi *hdmi = dev_get_drvdata(dev);
    /*
     * freeze context: tasks are frozen, no GFP_KERNEL allocations.
     * Just save registers — do NOT touch clocks or send I2C commands.
     */
    h616_hdmi_save_regs(hdmi); /* memcpy to pre-allocated buffer */
    return 0;
}

static int h616_hdmi_thaw(struct device *dev)
{
    struct h616_hdmi *hdmi = dev_get_drvdata(dev);
    h616_hdmi_restore_regs(hdmi);
    return 0;
}

static int h616_hdmi_restore(struct device *dev)
{
    /* full reinit after loading hibernation image */
    return h616_hdmi_resume(dev);
}

static const struct dev_pm_ops h616_hdmi_pm_ops = {
    SYSTEM_SLEEP_PM_OPS(h616_hdmi_suspend, h616_hdmi_resume)
    /* explicit hibernation callbacks */
    .freeze   = h616_hdmi_freeze,
    .thaw     = h616_hdmi_thaw,
    .restore  = h616_hdmi_restore,
    .poweroff = h616_hdmi_suspend, /* same as suspend after image saved */
};
```

#### الدرس المستفاد
الـ `struct dev_pm_ops` في `pm.h` بيفصل بوضوح بين `.suspend` (system sleep) و`.freeze` (hibernation). الـ vendor drivers اللي بتحدد `.suspend` بس وبتسيب `.freeze` = NULL هتورّث الـ PM core إنه يستدعي الـ generic path — اللي ممكن يبقى unsafe لو الـ `.suspend` نفسه بيعمل sleeping allocations.

---

### السيناريو الثاني: IoT Sensor Hub على STM32MP1 — runtime PM بيمنع الـ suspend

#### العنوان
**الـ SPI sensor driver بيخلي الـ system مستحيل يعمل suspend**

#### السياق
IoT environmental sensor hub بالـ STM32MP1. الـ hub بيقرأ من 4 SPI sensors كل 10 ثواني. المطلوب إن النظام يعمل autosuspend بين القراءات.

#### المشكلة
`/sys/power/state` ما بيقبل "mem" ابداً. الـ log بيقول:
```
PM: Some devices failed to suspend, or early wake event detected
```

#### التحليل

```bash
# شوف مين ما عملش suspend
cat /sys/kernel/debug/suspend_stats
cat /sys/kernel/debug/device_component/status
```

الـ engineer بيلاقي إن الـ SPI sensor device عنده `usage_count` = 1 طول الوقت. بيبص على `dev_pm_info`:

```c
/* pm.h السطر 702 */
atomic_t  usage_count;    /* runtime PM reference count */
atomic_t  child_count;    /* active children count */
unsigned int disable_depth:3; /* rpm_disable nesting depth */
```

الـ driver عنده bug:

```c
static int stm32_sensor_probe(struct platform_device *pdev)
{
    /* ... */
    pm_runtime_enable(&pdev->dev);
    pm_runtime_get_sync(&pdev->dev); /* usage_count++ */
    /* BUG: pm_runtime_put() never called! */
    return 0;
}
```

الـ `pm_runtime_get_sync` رفع الـ `usage_count` لـ 1 وما نزلش. لما الـ system بيحاول suspend، الـ PM core بيشوف `usage_count > 0` وبيقول إن الـ device active وما يقدرش يعمل system suspend.

لو الـ driver كان بيستخدم `DPM_FLAG_SMART_SUSPEND`:

```c
/* pm.h السطر 663 */
#define DPM_FLAG_SMART_SUSPEND  BIT(2)
```

الـ PM core كان ممكن يتجاوز الـ runtime state، لكن الـ flag مش set.

#### الحل

```c
static int stm32_sensor_probe(struct platform_device *pdev)
{
    struct stm32_sensor *sensor = ...;

    pm_runtime_enable(&pdev->dev);
    pm_runtime_set_autosuspend_delay(&pdev->dev, 500); /* 500ms */
    pm_runtime_use_autosuspend(&pdev->dev);

    /* set flag: allow system suspend even if runtime-active */
    device_set_flag(&pdev->dev, DPM_FLAG_SMART_SUSPEND);

    return 0;
}

static int stm32_sensor_read(struct stm32_sensor *sensor)
{
    int ret;
    /* wake device, do work, then schedule autosuspend */
    pm_runtime_get_sync(sensor->dev);
    ret = stm32_spi_transfer(sensor);
    pm_runtime_mark_last_busy(sensor->dev);
    pm_runtime_put_autosuspend(sensor->dev); /* usage_count-- after delay */
    return ret;
}

/* runtime_suspend: called when usage_count reaches 0 */
static int stm32_sensor_runtime_suspend(struct device *dev)
{
    struct stm32_sensor *s = dev_get_drvdata(dev);
    clk_disable_unprepare(s->clk);
    return 0;
}

static const struct dev_pm_ops stm32_sensor_pm_ops = {
    DEFINE_RUNTIME_DEV_PM_OPS(stm32_sensor_pm_ops,
        stm32_sensor_runtime_suspend,
        stm32_sensor_runtime_resume,
        NULL)
};
```

```bash
# تحقق من runtime PM status
cat /sys/bus/spi/devices/spi0.0/power/runtime_status
cat /sys/bus/spi/devices/spi0.0/power/usage_count
```

#### الدرس المستفاد
الـ `usage_count` في `dev_pm_info` هو الـ gatekeeper للـ runtime PM. أي `pm_runtime_get*` لازم يقابله `pm_runtime_put*`. الـ `DPM_FLAG_SMART_SUSPEND` موجود بالضبط عشان يخلي الـ system suspend يشتغل حتى لو الـ device كانت runtime-active.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — wakeup source مش شغال

#### العنوان
**الـ CAN bus لا يصحّي الـ ECU من الـ suspend على i.MX8**

#### السياق
Automotive ECU بالـ NXP i.MX8QM. الـ ECU المفروض يدخل في suspend لما مفيش نشاط على الـ CAN bus، ويصحى لما تيجي رسالة CAN جديدة من الـ vehicle network.

#### المشكلة
الـ ECU بيعمل suspend صح، بس الـ CAN message اللي بتيجي من الـ body controller ما بتصحيش.

#### التحليل
الـ engineer بيفحص الـ `dev_pm_info` للـ CAN device:

```bash
cat /sys/bus/platform/devices/5a8d0000.can/power/wakeup
# Output: disabled
```

ببص على `dev_pm_info`:

```c
/* pm.h السطر 666 */
struct dev_pm_info {
    bool  can_wakeup:1;       /* hardware can generate wakeup */
    /* ... */
#ifdef CONFIG_PM_SLEEP
    struct wakeup_source *wakeup; /* NULL means wakeup disabled */
    bool  wakeup_path:1;      /* device is in the wakeup path */
    /* ... */
#endif
};
```

الـ `can_wakeup` = false لأن الـ driver ما استدعاش `device_set_wakeup_capable()`. وبالتالي `wakeup` pointer = NULL، والـ PM core مش بيعتبر الـ device ده wakeup source.

في `suspend_noirq`:

```c
/* الـ PM core بيفحص wakeup_path قبل ما يوقف الـ IRQ */
/* لو wakeup_path = false، الـ wakeup IRQ بيتعطل */
```

الـ CAN controller IRQ بيتعطل قبل الـ system يدخل في الـ low-power state.

#### الحل

```c
static int imx8_can_probe(struct platform_device *pdev)
{
    struct imx8_can *can = ...;
    int ret;

    /* declare hardware wakeup capability */
    device_set_wakeup_capable(&pdev->dev, true);

    /* enable wakeup by default for automotive use case */
    device_wakeup_enable(&pdev->dev);

    return 0;
}

static int imx8_can_suspend(struct device *dev)
{
    struct imx8_can *can = dev_get_drvdata(dev);

    if (device_may_wakeup(dev)) {
        /* configure CAN controller to generate wakeup on RX */
        imx8_can_enable_wakeup_irq(can);
        enable_irq_wake(can->irq);
    } else {
        imx8_can_disable(can);
    }
    return 0;
}

static int imx8_can_resume(struct device *dev)
{
    struct imx8_can *can = dev_get_drvdata(dev);

    if (device_may_wakeup(dev))
        disable_irq_wake(can->irq);

    imx8_can_reinit(can);
    return 0;
}

static const struct dev_pm_ops imx8_can_pm_ops = {
    SYSTEM_SLEEP_PM_OPS(imx8_can_suspend, imx8_can_resume)
};
```

في الـ device tree:
```dts
&flexcan1 {
    status = "okay";
    wakeup-source;   /* mark as system wakeup source */
};
```

```bash
# تفعيل الـ wakeup
echo enabled > /sys/bus/platform/devices/5a8d0000.can/power/wakeup
# تحقق
cat /sys/power/wakeup_count
```

#### الدرس المستفاد
الـ `wakeup` pointer في `dev_pm_info` بيبقى NULL إلا لو الـ driver صرّح بـ `device_set_wakeup_capable()`. بدون ده، الـ `wakeup_path` flag مش هيتضبط، والـ PM core هيعطل الـ wakeup IRQ في الـ `suspend_noirq` phase — اللي في `pm.h` بيكون بعد ما الـ interrupts تتعطل.

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — PM domain مش شغال

#### العنوان
**الـ I2C controller مش بيدخل في low-power أبداً على AM62x custom board**

#### السياق
شركة بتعمل custom industrial board بالـ TI AM62x. الـ board فيها I2C controller بيتكلم مع EEPROM وـ temperature sensors. الـ engineer لاحظ إن الـ I2C clock ثابت وما بيتوقفش حتى لو مفيش transactions.

#### المشكلة
الـ `runtime_status` دايماً "active"، والـ I2C controller مش بيعمل runtime suspend. الاستهلاك أعلى من الـ spec بـ 15mA.

#### التحليل
الـ engineer بيبص على الـ `dev_pm_domain` في `pm.h`:

```c
/* pm.h السطر 751 */
struct dev_pm_domain {
    struct dev_pm_ops  ops;       /* domain-level PM callbacks */
    int (*start)(struct device *dev);
    void (*detach)(struct device *dev, bool power_off);
    int (*activate)(struct device *dev);  /* called before probe */
    void (*sync)(struct device *dev);     /* called after probe */
    void (*dismiss)(struct device *dev);
    int (*set_performance_state)(struct device *dev, unsigned int state);
};
```

الـ AM62x بيستخدم TI Power Sleep Controller (PSC) كـ PM domain. بيفحص:

```bash
cat /sys/bus/platform/devices/20000000.i2c/power/runtime_status
# active

cat /sys/bus/platform/devices/20000000.i2c/power/control
# on   <-- هنا المشكلة!
```

الـ `control` = "on" يعني `pm_runtime_forbid()` اتستدعى في مكان ما. بيفحص الـ driver:

```c
/* bug في الـ AM62 I2C driver */
static int am62_i2c_probe(struct platform_device *pdev)
{
    /* ... */
    pm_runtime_enable(&pdev->dev);
    pm_runtime_get_noresume(&pdev->dev); /* usage_count++ بس من غير resume */
    /* BUG: get_noresume without matching put */
    return 0;
}
```

بالإضافة لده، بيلاقي إن الـ `pm_subsys_data`:

```c
/* pm.h السطر 635 */
struct pm_subsys_data {
    spinlock_t    lock;
    unsigned int  refcount;
#ifdef CONFIG_PM_CLK
    unsigned int  clock_op_might_sleep;
    struct mutex  clock_mutex;
    struct list_head clock_list; /* clocks managed by PM framework */
#endif
    /* ... */
};
```

الـ clock مش مسجلة في `clock_list`، فالـ PM core مش عارف يوقفها تلقائياً.

#### الحل

```c
static int am62_i2c_probe(struct platform_device *pdev)
{
    struct am62_i2c *i2c = ...;
    int ret;

    /* register clock with PM clock framework */
    ret = pm_clk_create(&pdev->dev);
    if (ret)
        return ret;

    ret = pm_clk_add(&pdev->dev, "fck"); /* functional clock */
    if (ret)
        goto err_clk;

    pm_runtime_enable(&pdev->dev);
    pm_runtime_set_autosuspend_delay(&pdev->dev, 200);
    pm_runtime_use_autosuspend(&pdev->dev);

    /* do NOT call pm_runtime_get_noresume without matching put */
    ret = pm_runtime_get_sync(&pdev->dev);
    if (ret < 0)
        goto err_runtime;

    ret = am62_i2c_init(i2c);

    pm_runtime_mark_last_busy(&pdev->dev);
    pm_runtime_put_autosuspend(&pdev->dev); /* correct: put after init */

    return 0;

err_runtime:
    pm_runtime_disable(&pdev->dev);
err_clk:
    pm_clk_destroy(&pdev->dev);
    return ret;
}

static int am62_i2c_runtime_suspend(struct device *dev)
{
    /* pm_clk framework will stop "fck" automatically */
    return pm_clk_suspend(dev);
}

static int am62_i2c_runtime_resume(struct device *dev)
{
    return pm_clk_resume(dev);
}

static const struct dev_pm_ops am62_i2c_pm_ops = {
    SYSTEM_SLEEP_PM_OPS(pm_runtime_force_suspend, pm_runtime_force_resume)
    RUNTIME_PM_OPS(am62_i2c_runtime_suspend, am62_i2c_runtime_resume, NULL)
};
```

```bash
# تحقق من التحسين
cat /sys/bus/platform/devices/20000000.i2c/power/runtime_status
# suspended  (بعد الـ fix)

# شوف الـ clock state
cat /sys/kernel/debug/clk/i2c0_fck/clk_enable_count
```

#### الدرس المستفاد
الـ `pm_subsys_data` في `pm.h` بيحتوي على `clock_list` — اللي لو الـ clocks اتسجلت فيه عن طريق `pm_clk_add()`، الـ PM framework هيوقفها ويشغلها تلقائياً مع كل runtime suspend/resume. بدون التسجيل ده، الـ clocks بتفضل شغالة حتى لو الـ device suspended. والـ `pm_runtime_get_noresume` بدون `put` مقابل هو أكتر bug شايع في الـ board bring-up.
## Phase 7: مصادر ومراجع

### مصادر LWN.net

**الـ LWN.net** هو أهم مصدر لمتابعة تطور الـ Linux kernel، وفيما يلي أهم المقالات المتعلقة بـ power management وبالـ `pm.h`:

| المقالة | الوصف |
|---------|-------|
| [Linux power management: The documentation I wanted to read](https://lwn.net/Articles/505683/) | نظرة شاملة على نظام الـ PM في الـ kernel — نقطة البداية المثالية |
| [Runtime power management](https://lwn.net/Articles/347573/) | شرح تفصيلي لـ runtime PM وإضافة الـ `runtime_suspend/resume/idle` callbacks |
| [Documentation/power/runtime_pm.txt](https://lwn.net/Articles/347575/) | النص الأصلي للـ documentation الذي أُضيف مع الـ runtime PM framework |
| [PM: Introduce core framework for run-time PM of I/O devices (rev. 17)](https://lwn.net/Articles/347574/) | الـ patch الأصلي الذي أضاف الـ runtime PM framework لأول مرة |
| [Run-time PM idea](https://lwn.net/Articles/336729/) | النقاش الأولي على الـ mailing list قبل تنفيذ الـ runtime PM |
| [PM: Using runtime PM callbacks for system suspend/resume](https://lwn.net/Articles/733130/) | كيفية إعادة استخدام الـ `runtime_suspend/resume` في system sleep |
| [PM: Introduce new top level suspend and hibernation callbacks (rev. 3)](https://lwn.net/Articles/274689/) | إضافة الـ `prepare/complete/freeze/thaw/poweroff/restore` callbacks |
| [PM / Sleep: Introduce new phases of device suspend/resume](https://lwn.net/Articles/475730/) | تعديلات على مراحل الـ suspend/resume وتدقيق الترتيب |
| [PM: Separate hibernation code from suspend code](https://lwn.net/Articles/233283/) | فصل كود الـ hibernation عن الـ suspend — تغيير معماري مهم |
| [PM: Hibernation and suspend notifiers](https://lwn.net/Articles/235936/) | إضافة الـ notifiers لإشعار الـ subsystems بأحداث الـ PM |
| [Suspend and hibernation status report](https://lwn.net/Articles/243404/) | تقرير تاريخي عن حالة الـ suspend والـ hibernation |
| [PCI: Runtime power management](https://lwn.net/Articles/346898/) | تطبيق الـ runtime PM على حافلة الـ PCI كمثال عملي |
| [CPU PM notifiers](https://lwn.net/Articles/457614/) | الـ CPU PM notifiers وتأثيرها على منظومة الـ PM |

---

### التوثيق الرسمي للـ kernel

**الـ Documentation/** الرسمي هو المرجع الأساسي لفهم السلوك المضمون:

#### توثيق الـ driver-api

| الملف | الرابط |
|-------|--------|
| `Documentation/driver-api/pm/devices.rst` | [Device Power Management Basics](https://docs.kernel.org/driver-api/pm/devices.html) — شرح كامل لدورة حياة الـ `dev_pm_ops` callbacks |
| `Documentation/driver-api/pm/types.rst` | [Device Power Management Data Types](https://www.kernel.org/doc/html/v4.12/driver-api/pm/types.html) — تعريف `struct dev_pm_ops` و`pm_message_t` |
| `Documentation/driver-api/pm/notifiers.rst` | [Suspend/Hibernation Notifiers](https://static.lwn.net/kerneldoc/driver-api/pm/notifiers.html) — نظام الـ notifiers |

#### توثيق الـ power/

| الملف | الرابط |
|-------|--------|
| `Documentation/power/runtime_pm.rst` | [Runtime PM Framework](https://docs.kernel.org/power/runtime_pm.html) — المرجع الكامل للـ runtime PM |
| `Documentation/power/runtime_pm.txt` | [النص الأصلي على kernel.org](https://www.kernel.org/doc/Documentation/power/runtime_pm.txt) |
| `Documentation/power/basic-pm-debugging.rst` | [Debugging hibernation and suspend](https://www.kernel.org/doc/html/latest/power/basic-pm-debugging.html) |

#### توثيق الـ admin-guide

| الملف | الرابط |
|-------|--------|
| `Documentation/admin-guide/pm/sleep-states.rst` | [System Sleep States](https://static.lwn.net/kerneldoc/admin-guide/pm/index.html) — S0/S1/S2/S3/S4 وما يقابلها في الـ kernel |

---

### توثيق الـ kernel.org الشامل

```
https://docs.kernel.org/driver-api/pm/index.html
https://docs.kernel.org/power/index.html
```

- [CPU and Device Power Management — kernel.org](https://static.lwn.net/kerneldoc/driver-api/pm/index.html)
- [Power Management — admin guide](https://static.lwn.net/kerneldoc/admin-guide/pm/index.html)
- [Runtime PM Framework — kernel.org](https://static.lwn.net/kerneldoc/power/runtime_pm.html)
- [USB Power Management](https://static.lwn.net/kerneldoc/driver-api/usb/power-management.html)
- [PCI Power Management](https://www.kernel.org/doc/html/v5.5/power/pci.html)

---

### مواقع kernelnewbies.org و elinux.org

#### kernelnewbies.org

| الصفحة | الوصف |
|--------|-------|
| [PowerManagement](https://kernelnewbies.org/PowerManagement) | مدخل للـ PM — يشرح APM وACPI والـ Software Suspend |
| [KernelProjects/PowerSaving](https://kernelnewbies.org/KernelProjects/PowerSaving) | مشاريع توفير الطاقة في الـ kernel |
| [Linux 5.12 — DTPM](https://kernelnewbies.org/Linux_5.12) | إضافة الـ Dynamic Thermal Power Management framework |
| [Linux 5.1 — cpuidle governor](https://kernelnewbies.org/Linux_5.1) | الـ governor الجديد الأكثر كفاءة للـ cpuidle |
| [Linux 3.13 — power capping](https://kernelnewbies.org/Linux_3.13) | إطار الـ power capping لأجهزة Intel RAPL |
| [Linux 2.6.24 — link PM](https://kernelnewbies.org/Linux_2_6_24) | الـ Device Initiated Power Management لأقراص SATA |
| [Linux 2.6.26 — PCIe ASPM](https://kernelnewbies.org/Linux_2_6_26) | دعم PCIe Active State Power Management |

#### elinux.org

| الصفحة | الوصف |
|--------|-------|
| [Power Management for Linux](https://elinux.org/Power_Management) | الصفحة الرئيسية للـ PM على eLinux |
| [Power Management Presentations](https://elinux.org/Power_Management_Presentations) | عروض تقديمية من مؤتمرات Embedded Linux |
| [Power Management Framework](https://elinux.org/Power_Management_Framework) | روابط لتوثيق الـ kernel وعروض الـ PM |
| [Static Power Management Specification](https://elinux.org/Static_Power_Management_Specification) | مواصفات الـ suspend/resume للأنظمة المدمجة |
| [Power Management Specification](https://elinux.org/Power_Management_Specification) | تصنيف تقنيات الـ PM: platform suspend، device PM، dynamic PM |
| [Category: Power Management](https://elinux.org/Category:Power_Management) | فهرس كامل لصفحات الـ PM على eLinux |
| [CELF PM Requirements 2006](https://www.elinux.org/CELF_PM_Requirements_2006) | متطلبات الـ PM من منظور مطوري أجهزة Consumer Electronics |
| [Debugging Embedded Linux Kernel PM (PDF)](https://www.elinux.org/images/1/13/Debugging_Embedded_Linux_(Kernel)_Power_Management.pdf) | دليل عملي لتصحيح مشاكل الـ PM في الأنظمة المدمجة |
| [Consolidating Linux PM on ARM (PDF)](https://elinux.org/images/0/09/Elce11_pieralisi.pdf) | توحيد الـ PM على معمارية ARM |
| [PM Domains on SH7372 (PDF)](https://elinux.org/images/e/e6/Elce11_wysocki.pdf) | تطبيق الـ PM domains على معالج SH7372 |

---

### الـ Kernel Source Files ذات الصلة المباشرة

```
include/linux/pm.h           — تعريفات struct dev_pm_ops و pm_message_t
include/linux/pm_runtime.h   — API الـ runtime PM للـ drivers
include/linux/pm_domain.h    — PM domains (genpd)
drivers/base/power/          — تنفيذ الـ PM core في الـ kernel
kernel/power/                — كود الـ system sleep والـ hibernation
```

---

### الكتب المرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)

**المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
**الفصل المهم**: Chapter 14 — The Linux Device Model (يشمل شرح الـ `dev_pm_ops` والـ bus callbacks)
**متاح مجانًا**: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

> ملاحظة: الكتاب قديم (2005) — الـ runtime PM لم يكن موجودًا وقتها، لذا يُستخدم كمدخل للـ device model فقط.

#### Linux Kernel Development, 3rd Edition (Robert Love)

**الفصل المهم**: Chapter 14 — The Block I/O Layer + عمومًا أي فصل يتناول الـ device lifecycle
**الفائدة**: فهم الـ kernel data structures التي يعتمد عليها الـ PM

#### Essential Linux Device Drivers (Sreekrishnan Venkateswaran)

**الفصل المهم**: Chapter 8 — Power Management
يشرح الـ `dev_pm_ops` callbacks بشكل عملي مع أمثلة على الـ suspend/resume في drivers حقيقية.

#### Embedded Linux Primer, 2nd Edition (Christopher Hallinan)

**الفصول المهمة**: الفصول المتعلقة بالـ power management في الأجهزة المدمجة
يغطي الفرق بين الـ static suspend (S3/S4) والـ runtime PM لأجهزة ARM وMIPS.

---

### الـ Mailing Lists والنقاشات التاريخية

نقاشات أساسية على **linux-pm@vger.kernel.org**:

- [Run-time PM idea — النقاش التأسيسي](https://lwn.net/Articles/336729/) — حيث بدأت فكرة الـ runtime PM على الـ mailing list
- [PM: Introduce core framework for run-time PM (rev. 17)](https://lwn.net/Articles/347574/) — الـ patch set الذي قبله الـ Linus

للبحث في أرشيف الـ mailing list مباشرة:
```
https://lore.kernel.org/linux-pm/
```

---

### الـ Git Commits المهمة

للتتبع التاريخي لتطور الـ `pm.h` وما يرتبط به:

```bash
# أول إضافة للـ runtime PM callbacks في dev_pm_ops
git log --all --oneline -- include/linux/pm.h

# البحث في تاريخ الـ pm.h
git log --follow -p include/linux/pm.h | less

# الـ commit الذي أضاف runtime_suspend/resume/idle
git log --oneline --all --grep="runtime PM" -- include/linux/pm.h
```

الـ commits المهمة الموثقة:
- **2009**: إضافة `runtime_suspend`, `runtime_resume`, `runtime_idle` إلى `struct dev_pm_ops`
- **2010**: إضافة `prepare` و`complete` و`freeze` و`thaw` و`poweroff` و`restore`
- **2011**: إضافة الـ `_noirq` و`_early` و`_late` variants لمزيد من التحكم
- **2014**: دمج `SET_RUNTIME_PM_OPS` و`SET_SYSTEM_SLEEP_PM_OPS` helper macros

---

### مصطلحات البحث

للعثور على مزيد من المعلومات، استخدم هذه الـ search terms:

```
linux kernel dev_pm_ops callbacks
linux runtime PM runtime_suspend runtime_resume
linux system sleep suspend to RAM S3 kernel
linux hibernation suspend to disk S4 kernel
linux pm_message_t PMSG_SUSPEND PMSG_FREEZE
linux PM domains genpd generic power domain
linux wakeup sources pm_stay_awake pm_relax
linux autosuspend pm_runtime_set_autosuspend_delay
linux pm_runtime_get_sync pm_runtime_put_autosuspend
linux power management regulator framework
linux ACPI power management S-states D-states
linux cpuidle cpufreq power management
linux device power state D0 D1 D2 D3hot D3cold
```
## Phase 8: Writing simple module

### الفكرة

**`dpm_suspend_start`** هي دالة exported موجودة في `pm.h` بتبدأ دورة الـ suspend الكاملة للـ devices. بدل ما نـ hook الـ function نفسها بـ kprobe — اللي ممكن تتعقد مع الـ PM core — هنستخدم **PM notifier** اللي الـ kernel بيوفره عشان يبلغ الـ subsystems بأحداث الـ suspend/resume. ده أكتر أماناً وأوضح.

الـ **`register_pm_notifier`** بتسجل callback يتنادى عند كل transition في الـ system sleep state، وبترجعلنا الـ `pm_message_t` اللي فيها الـ `event` code اللي يدينا معلومات عن نوع الـ transition.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * pm_spy.c — Watch system PM transitions via PM notifier
 *
 * Hooks into the kernel's PM notifier chain to log every
 * suspend/resume/hibernate event with its PM_EVENT_ code.
 */

#include <linux/module.h>       /* MODULE_*, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/suspend.h>      /* register_pm_notifier, PM_SUSPEND_* events */
#include <linux/notifier.h>     /* notifier_block, NOTIFY_OK */
#include <linux/pm.h>           /* pm_message_t, PM_EVENT_* constants */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("PM Spy Demo");
MODULE_DESCRIPTION("Log system PM transitions via PM notifier chain");

/* ------------------------------------------------------------------ */
/*  Helper: translate PM_EVENT_* value to a human-readable string      */
/* ------------------------------------------------------------------ */
static const char *pm_event_name(unsigned long event)
{
    switch (event) {
    case PM_SUSPEND_PREPARE:  return "SUSPEND_PREPARE";   /* going to suspend */
    case PM_POST_SUSPEND:     return "POST_SUSPEND";       /* returned from suspend */
    case PM_HIBERNATION_PREPARE: return "HIBERNATION_PREPARE";
    case PM_POST_HIBERNATION: return "POST_HIBERNATION";
    case PM_RESTORE_PREPARE:  return "RESTORE_PREPARE";
    case PM_POST_RESTORE:     return "POST_RESTORE";
    default:                  return "UNKNOWN";
    }
}

/* ------------------------------------------------------------------ */
/*  The notifier callback — called by PM core before/after transitions */
/* ------------------------------------------------------------------ */
static int pm_spy_notify(struct notifier_block *nb,
                         unsigned long event,     /* PM_SUSPEND_PREPARE etc. */
                         void *data)              /* always NULL for PM notifier */
{
    /*
     * event   : one of PM_SUSPEND_PREPARE / PM_POST_SUSPEND /
     *           PM_HIBERNATION_PREPARE / PM_POST_HIBERNATION etc.
     *           defined in <linux/suspend.h>
     * data    : not used by the PM notifier chain (always NULL)
     */
    pr_info("pm_spy: PM event [%lu] => %s\n", event, pm_event_name(event));

    /*
     * NOTIFY_OK = we handled it fine, let the rest of the chain continue.
     * لو رجعنا NOTIFY_BAD ممكن نوقف الـ suspend — مش ده هدفنا هنا.
     */
    return NOTIFY_OK;
}

/* ------------------------------------------------------------------ */
/*  The notifier_block — ربط الـ callback بالـ PM chain               */
/* ------------------------------------------------------------------ */
static struct notifier_block pm_spy_nb = {
    .notifier_call = pm_spy_notify,
    .priority      = 0,   /* default priority, run in registration order */
};

/* ------------------------------------------------------------------ */
/*  module_init — register with the PM notifier chain                  */
/* ------------------------------------------------------------------ */
static int __init pm_spy_init(void)
{
    int ret;

    /*
     * register_pm_notifier() بتضيف الـ notifier_block جوا
     * الـ pm_chain_head اللي الـ PM core بيـ iterate عليها
     * قبل وبعد كل system sleep transition.
     */
    ret = register_pm_notifier(&pm_spy_nb);
    if (ret) {
        pr_err("pm_spy: failed to register PM notifier: %d\n", ret);
        return ret;
    }

    pr_info("pm_spy: loaded — watching PM transitions\n");
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit — MUST unregister to avoid a dangling pointer          */
/* ------------------------------------------------------------------ */
static void __exit pm_spy_exit(void)
{
    /*
     * لو ما أزلناش الـ notifier_block قبل ما الـ module يتـ unload،
     * الـ PM core هيحاول يكال الـ callback بعد ما الـ code اتشال من الـ memory
     * → kernel panic. الـ unregister_pm_notifier بتشيله من الـ chain بأمان.
     */
    unregister_pm_notifier(&pm_spy_nb);
    pr_info("pm_spy: unloaded\n");
}

module_init(pm_spy_init);
module_exit(pm_spy_exit);
```

---

### Makefile

```makefile
obj-m += pm_spy.o

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
| `<linux/module.h>` | الـ macros الأساسية للـ module: `MODULE_LICENSE`, `module_init`, `module_exit` |
| `<linux/suspend.h>` | بتعرّف `register_pm_notifier`, `unregister_pm_notifier`, والـ `PM_SUSPEND_*` events |
| `<linux/notifier.h>` | بتعرّف `notifier_block` و `NOTIFY_OK` |
| `<linux/pm.h>` | بتعرّف `pm_message_t` والـ `PM_EVENT_*` constants |

**الـ `<linux/suspend.h>`** هو الـ header الرئيسي هنا لأنه بيجمع الـ PM notifier API كله في مكان واحد.

#### الـ `pm_event_name()` helper

بتحوّل الـ `unsigned long event` القادم من الـ PM core لـ string مقروء. ده مفيد في الـ `pr_info` لأن القيم الرقمية (زي `4` للـ `PM_SUSPEND_PREPARE`) مش واضحة في الـ dmesg.

#### الـ `pm_spy_notify()` callback

**الـ `event`** بييجي من الـ PM core ويكون واحد من الـ `PM_SUSPEND_PREPARE` / `PM_POST_SUSPEND` / `PM_HIBERNATION_PREPARE` إلخ. **الـ `data`** في الـ PM notifier chain بيكون دايماً `NULL` — محتاجينه في الـ signature بس لأن الـ API بيطلبه.

بنرجع **`NOTIFY_OK`** عشان نخلي باقي الـ notifiers في الـ chain تشتغل. لو رجعنا `NOTIFY_BAD` ممكن نوقف الـ suspend نفسه، وده مش المطلوب في module للـ monitoring.

#### الـ `pm_spy_nb` struct

**الـ `notifier_block`** ده هو الـ entry بتاعنا في الـ PM chain. الـ `priority = 0` يعني الـ kernel بيرتبه بالترتيب اللي اتسجل بيه بالنسبة للـ notifiers التانية اللي عندها نفس الـ priority.

#### الـ `pm_spy_init()`

**`register_pm_notifier()`** بتضيف الـ `pm_spy_nb` في الـ `pm_chain_head` اللي الـ kernel بيستخدمها في الـ `pm_notifier_call_chain()`. لو الـ registration فشلت بنرجع الـ error كـ return value من `module_init` عشان الـ `insmod` يعرف إن الـ load فشل.

#### الـ `pm_spy_exit()`

**`unregister_pm_notifier()`** ضرورية جداً. لو الـ module اتـ unload وفضل الـ `notifier_block` مسجل، الـ PM core هيحاول يـ call الـ `pm_spy_notify` من memory اتحررت → **use-after-free** وكراش مضمون.

---

### تجربة الـ module

```bash
# بناء وتحميل الـ module
make
sudo insmod pm_spy.ko

# شوف الـ log
dmesg | grep pm_spy

# جرب suspend
sudo systemctl suspend

# بعد الـ resume
dmesg | grep pm_spy
# هتلاقي:
# pm_spy: PM event [4] => SUSPEND_PREPARE
# pm_spy: PM event [5] => POST_SUSPEND

# إزالة الـ module
sudo rmmod pm_spy
```

---

### مثال على الـ output

```
[  142.331204] pm_spy: loaded — watching PM transitions
[  198.004712] pm_spy: PM event [4] => SUSPEND_PREPARE
[  198.821903] pm_spy: PM event [5] => POST_SUSPEND
[  201.003310] pm_spy: unloaded
```

الـ events رقم `4` و`5` هم `PM_SUSPEND_PREPARE` و`PM_POST_SUSPEND` اللي معرّفين في `<linux/suspend.h>`. ده بيديك visibility كاملة على lifecycle الـ system sleep بدون ما تلمس الـ PM core نفسه.
