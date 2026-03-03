## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

**الـ `pm_runtime.h`** جزء من **POWER MANAGEMENT CORE** — المسؤول عنه Rafael J. Wysocki، وبيتبع الـ mailing list `linux-pm@vger.kernel.org`. الملف موجود في `include/linux/pm_*.h` اللي بتشملها قائمة الـ MAINTAINERS صراحةً.

---

### الفكرة الكبيرة — قبل أي كود

#### المشكلة اللي بيحلها

تخيل عندك لابتوب فيه WiFi card. لو الـ WiFi شغال طول الوقت حتى لو مش بتبعت أو بتستقبل أي بيانات — هتخسر battery بلاش. المنطقي إنك **تطفي الـ WiFi أو تحطه في وضع توفير الطاقة** لما مفيش استخدام، وتصحيه تاني لما حد يحتاجه.

ده بالظبط اللي بيعمله **Runtime PM (Power Management)** — بس مش بس للـ WiFi، لكل device في النظام: USB controllers، GPU، audio chips، sensors، كل حاجة.

---

### القصة كاملة — تخيل المشكلة

**المشكلة:** عندك نظام Linux شغال على موبايل أو embedded device. فيه 20 device مختلف (USB hub, camera, touchscreen, GPS, ...). لو كل device اشتغل بـ full power طول اليوم — البطارية هتخلص في ساعات.

**الحل البدائي:** driver كل device يعمل `suspend` و`resume` يدوي. بس مين يعرف امتى الـ device مش بيُستخدم؟ مين يضمن إن مفيش حد تاني بيستخدمه دلوقتي؟ مين يدير التسلسل لو device A يعتمد على device B؟

**الحل الصح:** **Runtime PM Framework** — طبقة في الـ kernel بتتكلم مع كل driver وبتقوله "انت دلوقتي مش محتاج، روح نام" أو "فيه طلب جديد، صحى". الـ driver بس بيكتب الـ callbacks، والـ framework بيدير الباقي.

---

### هدف الملف `pm_runtime.h`

الملف ده هو **الـ public API** بتاع الـ Runtime PM Framework. بيوفر:

1. **Macros لتعريف الـ PM ops** — زي `DEFINE_RUNTIME_DEV_PM_OPS()` اللي بتخلي الـ driver يحدد إيه اللي يحصل عند `suspend` و`resume` و`idle`.

2. **Functions للتحكم في الـ usage counter** — الـ kernel بيعمل "reference counting" للـ device. لما حد يستخدمه بيعمل `get` (يزيد العداد)، ولما يخلص بيعمل `put` (يقلل العداد). لما العداد يوصل صفر — الـ device ممكن ينام.

3. **Functions للـ suspend/resume** — زي `pm_runtime_suspend()` و`pm_runtime_resume()` وكمان الـ async versions زي `pm_request_resume()`.

4. **الـ autosuspend mechanism** — بدل ما الـ device ينام فوراً لما العداد يوصل صفر، بينتظر delay معين (مثلاً 2000ms) عشان لو جه request تاني قريب ميصحيش ويرجع ينام، ده بيوفر overhead.

5. **Guards للـ locking** — `DEFINE_GUARD(pm_runtime_active, ...)` اللي بتضمن إن الـ device صاحي طول فترة استخدامك ليه وبتقلل العداد أوتوماتيك لما تخرج من الـ scope.

6. **Helper functions** — زي `pm_runtime_enabled()` و`pm_runtime_suspended()` للاستعلام عن حالة الـ device.

---

### حالات الـ Device (State Machine)

```
               usage_count > 0 أو طلب resume
    ┌──────────────────────────────────────────┐
    │                                          │
    ▼                                          │
RPM_ACTIVE ──── usage_count == 0 ────► RPM_SUSPENDING
    ▲                                          │
    │                                          ▼
    │                                    RPM_SUSPENDED
    │                                          │
    └──── runtime_resume callback ──── RPM_RESUMING ◄──┘
```

- **`RPM_ACTIVE`** — الـ device شغال بالكامل
- **`RPM_SUSPENDING`** — بيتنفذ الـ `runtime_suspend` callback دلوقتي
- **`RPM_SUSPENDED`** — الـ device نايم
- **`RPM_RESUMING`** — بيتنفذ الـ `runtime_resume` callback دلوقتي
- **`RPM_BLOCKED`** — الـ runtime PM اتعطل بسبب system sleep

---

### الـ Usage Counter — القلب النابض

```
pm_runtime_get_sync(dev)   → counter++، صحّي الـ device لو نايم
pm_runtime_put(dev)        → counter--، لو وصل 0 → قيّم النوم
pm_runtime_get_noresume()  → counter++ بس (من غير صحيان)
pm_runtime_put_noidle()    → counter-- بس (من غير تقييم)
```

الـ counter ده atomic (`atomic_t usage_count` في `struct dev_pm_info`) — بيتضمن thread-safety.

---

### الـ Autosuspend — الذكاء الإضافي

بعض الـ devices بتصحى وتنام كتير جداً في وقت قصير (زي USB mouse). كل صحيان ونوم ليه overhead (وقت + طاقة). الـ autosuspend بيقول:

> "متنامش فوراً، استنى مثلاً 2 ثانية. لو في الـ 2 ثانية دول جه request تاني — معتبره إنك لسه صاحي."

```c
pm_runtime_set_autosuspend_delay(dev, 2000); /* 2000ms */
pm_runtime_use_autosuspend(dev);
/* ... بدل pm_runtime_put() استخدم: */
pm_runtime_put_autosuspend(dev);
```

---

### إزاي الـ Driver بيستخدمه — المثال العملي

```c
/* في probe() */
pm_runtime_set_active(dev);       /* حالة ابتدائية: active */
pm_runtime_enable(dev);           /* فعّل الـ runtime PM */
pm_runtime_set_autosuspend_delay(dev, 1000);
pm_runtime_use_autosuspend(dev);

/* لما تيجي تتكلم مع الـ hardware */
ret = pm_runtime_resume_and_get(dev);  /* صحّي + زود العداد */
if (ret < 0)
    return ret;
/* ... تعامل مع الـ hardware ... */
pm_runtime_put_autosuspend(dev);       /* قلل العداد + autosuspend */

/* في remove() */
pm_runtime_disable(dev);
```

---

### الفرق بين `CONFIG_PM` و `CONFIG_PM_SLEEP`

| Guard | ما بيحميه |
|---|---|
| `CONFIG_PM` | الـ Runtime PM كله |
| `CONFIG_PM_SLEEP` | الـ System Sleep (suspend-to-RAM, hibernate) |

لما `CONFIG_PM` متفعلش — كل الـ functions بتبقى stubs فارغة أو بترجع قيم ثابتة، عشان الـ drivers يشتغلوا بدون تعديل على أنظمة من غير PM.

---

### الملفات المرتبطة اللي لازم تعرفها

| الملف | الدور |
|---|---|
| `include/linux/pm_runtime.h` | **الملف ده** — الـ public API |
| `include/linux/pm.h` | تعريف `struct dev_pm_ops`، `struct dev_pm_info`، `enum rpm_status` |
| `include/linux/device.h` | تعريف `struct device` اللي فيها `power` field من نوع `dev_pm_info` |
| `drivers/base/power/runtime.c` | **الـ implementation** — فيه الـ core logic كله |
| `drivers/base/power/main.c` | التكامل مع الـ system sleep |
| `drivers/base/power/generic_ops.c` | تنفيذ `pm_generic_runtime_suspend/resume` |
| `drivers/base/power/sysfs.c` | الـ `/sys/devices/.../power/` interface |
| `drivers/base/power/qos.c` | الـ PM QoS (latency constraints) |
| `drivers/base/power/wakeirq.c` | الـ dedicated wake IRQ للـ runtime PM |
| `Documentation/power/runtime_pm.rst` | التوثيق الرسمي التفصيلي |

---

### ملفات الـ Subsystem (Core + Headers + Implementation)

**Core Headers:**
- `include/linux/pm_runtime.h` ← الملف ده
- `include/linux/pm.h`
- `include/linux/pm_domain.h`
- `include/linux/pm_qos.h`
- `include/linux/pm_wakeirq.h`
- `include/linux/pm_clock.h`

**Core Implementation:**
- `drivers/base/power/runtime.c` ← أهم ملف
- `drivers/base/power/main.c`
- `drivers/base/power/common.c`
- `drivers/base/power/generic_ops.c`
- `drivers/base/power/qos.c`
- `drivers/base/power/sysfs.c`
- `drivers/base/power/wakeup.c`
- `drivers/base/power/wakeirq.c`
- `drivers/base/power/clock_ops.c`

**Hardware Drivers** (كل driver فيه `runtime_suspend`/`runtime_resume` callbacks):
- `drivers/usb/` — USB devices
- `drivers/net/` — network adapters
- `drivers/gpu/drm/` — graphics
- `drivers/i2c/` — I2C devices
- وغيرها من آلاف الـ drivers في الـ kernel
## Phase 2: شرح الـ Runtime PM Framework

### المشكلة اللي بيحلها الـ Subsystem

تخيل عندك SoC زي Qualcomm Snapdragon فيه عشرات الـ peripherals: USB controller، I2C، SPI، GPU، camera ISP، DSP، إلخ. كل واحد فيهم ياكل power حتى لو مش بيشتغل. لو سبتهم كلهم ON طول الوقت، البطارية تخلص في ساعات.

**الـ System Sleep** (suspend-to-RAM) بيحل مشكلة الـ power لما الجهاز idle خالص — لكنه مش practical لو عايز توفر power وأنت شغال. الـ USB controller مش لازم يكون شغال لو مفيش USB device متوصل. الـ I2C bus مش لازم يكون active بين كل transaction وتاني.

**المشكلة الأساسية**: كل driver كان بيعمل power management بطريقته الخاصة — بعضهم بيعمل gate clocks يدوي، بعضهم بيحط hardware في low-power state بطريقة ad-hoc، ومفيش coordination بين الـ parent والـ child devices. النتيجة: كود مكرر، bugs، وconsumption عالي.

---

### الحل — Runtime PM Framework

الـ kernel بيقدم **Runtime PM** كـ generic framework يعمل الآتي:

1. يتتبع إمتى كل device بيتستخدم فعلاً عن طريق **usage counter** (`usage_count`).
2. لما الـ counter يوصل صفر (device مش بيتستخدم)، يستدعي callback الـ driver عشان يحط الـ device في low-power state.
3. لما حد يطلب الـ device تاني، يصحيه تلقائياً قبل ما يوديه للـ driver.
4. يضمن إن الـ parent مش بيتوقف قبل كل الـ children.
5. يدعم **autosuspend** — delay قبل الـ suspend عشان تتجنب thrashing لو الـ device هيتستخدم تاني بسرعة.

الـ framework بيقدم abstraction واحد يشتغل مع أي device بدل ما كل driver يعمل الكلام ده من الأول.

---

### الـ Big Picture Architecture

```
 ┌─────────────────────────────────────────────────────────────────┐
 │                        User Space                               │
 │          (لا علاقة مباشرة — الـ PM بيحصل شفاف)                │
 └───────────────────────────────┬─────────────────────────────────┘
                                 │
 ┌───────────────────────────────▼─────────────────────────────────┐
 │                    Driver Code                                   │
 │   pm_runtime_get_sync(dev)  ──► استخدم الـ device               │
 │   pm_runtime_put_autosuspend(dev) ──► خلصت من الـ device        │
 └───────────────────────────────┬─────────────────────────────────┘
                                 │
 ┌───────────────────────────────▼─────────────────────────────────┐
 │              Runtime PM Core  (drivers/base/power/runtime.c)    │
 │                                                                  │
 │   ┌─────────────────────────────────────────────────────────┐   │
 │   │  usage_count (atomic_t) — عداد الاستخدام                │   │
 │   │  child_count (atomic_t) — عدد الـ children الـ active   │   │
 │   │  runtime_status (rpm_status) — الحالة الحالية           │   │
 │   │  disable_depth — عمق الـ disable                        │   │
 │   │  autosuspend_delay — التأخير قبل الـ auto-suspend       │   │
 │   │  last_busy — آخر وقت استخدام (لحساب الـ autosuspend)   │   │
 │   │  suspend_timer (hrtimer) — timer للـ autosuspend        │   │
 │   │  work (work_struct) — async operations                  │   │
 │   └─────────────────────────────────────────────────────────┘   │
 │                                                                  │
 │   Logic: idle? → check counter → 0? → call driver suspend       │
 │          get?  → check status → suspended? → call driver resume  │
 └───────────────────┬─────────────────────┬───────────────────────┘
                     │                     │
        ┌────────────▼──────┐   ┌──────────▼────────────┐
        │  PM Domain        │   │  Bus/Subsystem Layer   │
        │  (GENPD)          │   │  (i2c_bus_type, etc.)  │
        │  ops.runtime_*    │   │  ops.runtime_*         │
        └────────────┬──────┘   └──────────┬─────────────┘
                     │                     │
        ┌────────────▼─────────────────────▼─────────────┐
        │              Driver Callbacks                   │
        │  .runtime_suspend → cut clocks, power down      │
        │  .runtime_resume  → restore clocks, init HW    │
        │  .runtime_idle    → check if safe to suspend   │
        └─────────────────────────────────────────────────┘
```

---

### Consumers وProviders

| Layer | الدور | مثال |
|-------|-------|------|
| **Driver Code** | بيستخدم `pm_runtime_get/put` | `drivers/usb/host/xhci.c` |
| **Runtime PM Core** | بيدير الـ state machine | `drivers/base/power/runtime.c` |
| **Bus Type** | بيستدعي driver callbacks | `drivers/i2c/i2c-core.c` |
| **PM Domain (GENPD)** | power domains مشتركة | `drivers/base/power/domain.c` |
| **Clock Framework (CCF)** | تشغيل/إيقاف الـ clocks | `drivers/clk/` |
| **Regulator Framework** | تشغيل/إيقاف الـ voltages | `drivers/regulator/` |

الـ **GENPD (Generic Power Domain)** هو subsystem تاني — بيجمع devices في power domain واحدة، لو كل الـ devices في الـ domain وقفت يقدر يوقف الـ domain كلها.

الـ **CCF (Common Clock Framework)** هو subsystem تاني — الـ runtime suspend callback عادةً بيعمل `clk_disable_unprepare()` عشان يوفر power.

---

### التشبيه الحقيقي — مكتب وشغالين

تخيل إنك مدير مكتب (الـ Runtime PM Core) وعندك موظفين (devices):

| عنصر في التشبيه | المقابل الحقيقي |
|-----------------|----------------|
| المكتب | الـ system |
| كل موظف | كل `struct device` |
| عداد المهام على مكتب الموظف | `usage_count` |
| الموظف شاغل | `RPM_ACTIVE` |
| الموظف مش شاغل ومش موجود | `RPM_SUSPENDED` |
| الموظف بيرجع للمكتب | `RPM_RESUMING` |
| الموظف بيمشي من المكتب | `RPM_SUSPENDING` |
| قاعدة: مدير مش يمشي قبل موظفيه | parent device لا يتوقف قبل children |
| الـ timer على المكتب — "لو 5 دقايق مفيش شغل امشي" | `autosuspend_delay` |
| آخر مهمة خلصت | `last_busy` |
| صحيت الموظف عشان يعمل مهمة | `pm_runtime_get_sync()` |
| خلصت المهمة وبلغته | `pm_runtime_put_autosuspend()` |
| الموظف بيقرر بنفسه لو ممكن يمشي | `runtime_idle` callback |
| صاحب الشركة قرر إن المكتب دايماً مفتوح | `pm_runtime_disable()` |

التشبيه دقيق لأن:
- الـ `usage_count` بالضبط زي عداد المهام — لو وصل صفر يبدأ يفكر في الـ suspend.
- الـ `child_count` بيمنع الـ parent من الـ suspend لو في children active — زي إن المدير مش يقدر يمشي لو في موظف لسه شغال.
- الـ `autosuspend_delay` بيتجنب إن الموظف يمشي ويرجع بسرعة (thrashing).
- الـ `pm_runtime_block_if_disabled()` زي قاعدة ثابتة تمنع إغلاق المكتب وقت الذروة.

---

### الـ Core Abstraction — الـ Usage Counter وState Machine

**الفكرة المحورية**: كل device عنده **reference count** (`usage_count`). الـ framework يتدخل فقط لما الـ counter يوصل صفر.

```
        pm_runtime_get_sync()          pm_runtime_put() / pm_runtime_put_autosuspend()
              │ usage_count++                  │ usage_count--
              │                               │
              ▼                               ▼
         ┌─────────┐    runtime_resume    ┌─────────┐
         │ ACTIVE  │◄────────────────────│RESUMING │
         │usage > 0│                     └─────────┘
         └────┬────┘
              │ usage_count drops to 0
              │ (idle check passes)
              ▼
         ┌──────────┐   runtime_suspend  ┌───────────┐
         │SUSPENDING│───────────────────►│ SUSPENDED │
         └──────────┘                   └─────┬─────┘
                                              │ pm_runtime_resume() called
                                              └──────────► RPM_RESUMING
```

**الـ State Machine** معمولة من `enum rpm_status`:

```c
enum rpm_status {
    RPM_INVALID   = -1,  /* حالة غير صالحة */
    RPM_ACTIVE    = 0,   /* شغال كامل */
    RPM_RESUMING,        /* في الطريق للـ active */
    RPM_SUSPENDED,       /* موقوف */
    RPM_SUSPENDING,      /* في الطريق للـ suspend */
    RPM_BLOCKED,         /* محجوب (system sleep) */
};
```

---

### الـ dev_pm_info — قلب الـ Framework

كل device عنده `dev->power` من نوع `struct dev_pm_info`. ده الـ struct اللي الـ Runtime PM Core بيقرأ منه ويكتب فيه:

```c
struct dev_pm_info {
    /* ── Runtime PM fields (inside #ifdef CONFIG_PM) ── */
    atomic_t        usage_count;      /* عداد الاستخدام — الأهم */
    atomic_t        child_count;      /* عدد الـ children الـ active */
    unsigned int    disable_depth:3;  /* عمق الـ disable (يتزيد بكل pm_runtime_disable) */

    enum rpm_status runtime_status;   /* الحالة الحالية */
    enum rpm_status last_status;      /* آخر حالة قبل الـ block */
    enum rpm_request request;         /* الطلب الـ pending */

    struct hrtimer  suspend_timer;    /* timer للـ autosuspend */
    u64             timer_expires;    /* موعد الـ timer */
    u64             last_busy;        /* آخر وقت استخدام */
    int             autosuspend_delay;/* التأخير بالـ ms */

    struct work_struct work;          /* async operations تتعمل في pm_wq */
    wait_queue_head_t wait_queue;     /* للـ synchronous operations */

    bool            use_autosuspend:1;/* هل الـ autosuspend مفعّل؟ */
    bool            irq_safe:1;       /* callbacks تشتغل من interrupt context؟ */
    bool            no_callbacks:1;   /* device مش محتاج callbacks */
    bool            ignore_children:1;/* تجاهل الـ children للـ suspend */
    bool            runtime_auto:1;   /* autosuspend مفعّل من sysfs */
    bool            memalloc_noio:1;  /* لا IO allocations في الـ path */
    unsigned int    links_count;      /* عدد الـ device links */
};
```

---

### العلاقة بين الـ Structs

```
struct device
    │
    ├── struct dev_pm_info   power           ← الـ state machine data
    │       ├── atomic_t usage_count
    │       ├── atomic_t child_count
    │       ├── enum rpm_status runtime_status
    │       ├── struct hrtimer suspend_timer
    │       ├── struct work_struct work  ──────► pm_wq (dedicated workqueue)
    │       ├── int autosuspend_delay
    │       ├── u64 last_busy
    │       └── struct dev_pm_qos *qos  ──────► QoS constraints
    │
    └── struct dev_pm_domain *pm_domain       ← اختياري: PM domain
            └── struct dev_pm_ops ops
                    ├── runtime_suspend()
                    ├── runtime_resume()
                    └── runtime_idle()

struct bus_type / struct device_driver
    └── struct dev_pm_ops pm
            ├── runtime_suspend()   ← الـ driver بيملأ دول
            ├── runtime_resume()
            └── runtime_idle()
```

---

### الـ Flags — التحكم في السلوك

```c
/* flags بتتبعت في كل call */
#define RPM_ASYNC      0x01  /* لا تنتظر — ابعت work للـ pm_wq */
#define RPM_NOWAIT     0x02  /* لو في state change جارية، ارجع بـ error فوراً */
#define RPM_GET_PUT    0x04  /* تعديل الـ usage_count مع العملية */
#define RPM_AUTO       0x08  /* استخدم الـ autosuspend_delay */
#define RPM_TRANSPARENT 0x10 /* نجح حتى لو الـ runtime PM معطل */
```

مثال: `pm_runtime_put_autosuspend()` بتعمل:
```c
pm_runtime_mark_last_busy(dev);                    // سجل آخر استخدام
__pm_runtime_suspend(dev, RPM_GET_PUT | RPM_ASYNC | RPM_AUTO);
//                        ^^^^^^^^^^^^  ^^^^^^^^^  ^^^^^^^^
//                  نقص الـ counter   async op   استخدم delay
```

---

### الـ Autosuspend — تجنب الـ Thrashing

**المشكلة**: لو device بيتستخدم كل 50ms، وعملنا suspend بعد كل استخدام، هنضيع وقت وenergy في resume/suspend cycles.

**الحل**: `autosuspend_delay` — الـ device مش بيتوقف إلا لو مر وقت معين من آخر استخدام.

```c
/* الـ driver في probe() */
pm_runtime_set_autosuspend_delay(dev, 50); /* 50ms */
pm_runtime_use_autosuspend(dev);

/* في كل operation */
pm_runtime_get_sync(dev);
/* ... do work ... */
pm_runtime_mark_last_busy(dev);        /* سجل إن احنا استخدمناه دلوقتي */
pm_runtime_put_autosuspend(dev);       /* هيستنى 50ms قبل ما يوقف */
```

الـ `last_busy` بيتحسب بـ `ktime_get_mono_fast_ns()`. الـ `pm_runtime_autosuspend_expiration()` بترجع الوقت اللي المفروض ينتظره:
```
expiration = last_busy + autosuspend_delay_ns
```
لو الوقت ده جاي، الـ hrtimer بيشتغل ويبعت `RPM_REQ_AUTOSUSPEND`.

---

### الـ Device Links — Parent/Child Coordination

**اللي بيملكه الـ Runtime PM Core**:
- الـ `usage_count` tracking.
- الـ state machine والانتقالات بينها.
- الـ `child_count` — يزيد لما child device يبقى active، وده بيمنع الـ parent من الـ suspend.
- تنسيق الـ device links عبر `pm_runtime_get_suppliers()` و`pm_runtime_put_suppliers()`.

**اللي بيفوضه للـ Drivers**:
- إيه اللي بيحصل للـ hardware وقت الـ suspend (cut clocks؟ power gate؟ save registers؟).
- هل الـ device فعلاً جاهز للـ suspend (الـ `runtime_idle` callback).
- إزاي تعمل resume بشكل صح (restore state).

---

### مثال عملي — I2C Driver

```c
/* في probe() */
static int mydrv_probe(struct platform_device *pdev)
{
    /* بعد ما نجهز كل حاجة */
    pm_runtime_set_autosuspend_delay(&pdev->dev, 1000); /* 1 second */
    pm_runtime_use_autosuspend(&pdev->dev);
    pm_runtime_set_active(&pdev->dev);   /* ابدأ بحالة active */
    pm_runtime_enable(&pdev->dev);       /* فعّل الـ framework */
}

/* في كل I/O operation */
static int mydrv_transfer(struct mydrv *drv, ...)
{
    int ret;

    /* صحي الـ device وزود الـ usage counter */
    ret = pm_runtime_resume_and_get(&drv->dev);
    if (ret < 0)
        return ret;

    /* ... do the actual transfer ... */

    /* سجل إننا استخدمناه دلوقتي وانقص الـ counter */
    pm_runtime_mark_last_busy(&drv->dev);
    pm_runtime_put_autosuspend(&drv->dev);
    return 0;
}

/* الـ PM callbacks */
static int mydrv_runtime_suspend(struct device *dev)
{
    struct mydrv *drv = dev_get_drvdata(dev);
    clk_disable_unprepare(drv->clk);   /* وقف الـ clock */
    return 0;
}

static int mydrv_runtime_resume(struct device *dev)
{
    struct mydrv *drv = dev_get_drvdata(dev);
    return clk_prepare_enable(drv->clk); /* شغل الـ clock */
}

static const struct dev_pm_ops mydrv_pm_ops = {
    DEFINE_RUNTIME_DEV_PM_OPS(mydrv_pm_ops,
                              mydrv_runtime_suspend,
                              mydrv_runtime_resume,
                              NULL)
};
```

---

### الـ devm Variants — الـ Cleanup التلقائي

الـ `devm_pm_runtime_enable()` بتوفر cleanup تلقائي عند الـ remove:

```c
/* بدل */
pm_runtime_enable(dev);
/* وفي remove() */
pm_runtime_disable(dev);

/* استخدم */
devm_pm_runtime_enable(dev); /* بيعمل disable تلقائي لما يتعمل unbind */
```

الـ `devm_pm_runtime_get_noresume()` بتبقى مفيدة لما عايز تضمن إن الـ device مش هيتوقف وقت الـ probe.

---

### الـ Guards — C Scoped Locking للـ PM

الـ framework بيستخدم الـ guard mechanism الجديد في الـ kernel:

```c
/* الـ guard بيعمل get_sync وبعد الـ scope بيعمل put */
scoped_guard(pm_runtime_active, dev) {
    /* الـ device مضمون active هنا */
    do_something_with_device(dev);
} /* pm_runtime_put() هنا تلقائياً */

/* الـ conditional guard */
if (ACQUIRE(pm_runtime_active_try, guard)(dev)) {
    /* نجح الـ resume */
} else {
    /* الـ device في error state أو PM disabled */
}
```

---

### ملخص — إيه اللي بيملكه الـ Runtime PM Core

| مسؤولية | الـ Core | الـ Driver |
|----------|----------|-----------|
| تتبع الـ usage_count | ✓ | |
| تتبع الـ child_count | ✓ | |
| State machine transitions | ✓ | |
| Async work dispatch | ✓ | |
| Autosuspend timer | ✓ | |
| Device link coordination | ✓ | |
| Clock gating | | ✓ |
| Register save/restore | | ✓ |
| Power rail control | | ✓ |
| Idle decision | | ✓ (runtime_idle) |
| Hardware-specific wakeup | | ✓ |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### RPM Flags (بيتعدوا لـ `__pm_runtime_*` functions)

| Flag | Value | الوظيفة |
|------|-------|---------|
| `RPM_ASYNC` | `0x01` | الـ request بيتنفذ asynchronously عن طريق workqueue |
| `RPM_NOWAIT` | `0x02` | متستناش لو في state change جاري — ارجع فورًا |
| `RPM_GET_PUT` | `0x04` | زوّد/نقّص الـ `usage_count` مع العملية |
| `RPM_AUTO` | `0x08` | استخدم الـ `autosuspend_delay` بدل suspend فوري |
| `RPM_TRANSPARENT` | `0x10` | انجح حتى لو runtime PM معطّل |

#### enum `rpm_status` — الحالة الحالية للجهاز

| القيمة | المعنى |
|--------|--------|
| `RPM_INVALID` | `-1` — حالة غير صالحة (مش متهيّأ) |
| `RPM_ACTIVE` | `0` — الجهاز شغّال بالكامل |
| `RPM_RESUMING` | الـ `runtime_resume()` callback شغّال دلوقتي |
| `RPM_SUSPENDED` | الـ `runtime_suspend()` callback خلّص — الجهاز نايم |
| `RPM_SUSPENDING` | الـ `runtime_suspend()` callback شغّال دلوقتي |
| `RPM_BLOCKED` | runtime PM اتمنع بسبب system sleep |

#### enum `rpm_request` — نوع الـ request الـ pending

| القيمة | المعنى |
|--------|--------|
| `RPM_REQ_NONE` | مفيش request |
| `RPM_REQ_IDLE` | شغّل `runtime_idle()` |
| `RPM_REQ_SUSPEND` | شغّل `runtime_suspend()` |
| `RPM_REQ_AUTOSUSPEND` | suspend بعد انتهاء الـ autosuspend_delay |
| `RPM_REQ_RESUME` | شغّل `runtime_resume()` |

#### DPM Driver Flags (بيتحطوا وقت probe)

| Flag | الوظيفة |
|------|---------|
| `DPM_FLAG_NO_DIRECT_COMPLETE` | منعِش الـ direct-complete optimization |
| `DPM_FLAG_SMART_PREPARE` | خد return value الـ `prepare()` بجدية |
| `DPM_FLAG_SMART_SUSPEND` | متصحيش الجهاز من runtime suspend وقت system suspend |
| `DPM_FLAG_MAY_SKIP_RESUME` | السماح بـ skip لـ noirq/early resume callbacks |

#### Config Options المهمة

| Config | الأثر |
|--------|-------|
| `CONFIG_PM` | يفعّل كل runtime PM — بدونه كل الفنكشنز stubs |
| `CONFIG_PM_SLEEP` | يفعّل system sleep (suspend/hibernate) callbacks |
| `CONFIG_PM_CLK` | يضيف clock management في `pm_subsys_data` |
| `CONFIG_PM_GENERIC_DOMAINS` | يضيف `domain_data` pointer في `pm_subsys_data` |

---

### 1. الـ Structs المهمة

#### `struct dev_pm_info` — قلب runtime PM

ده الـ struct الأساسي اللي موجود جوّا كل `struct device` كـ `dev->power`. بيحتوي على كل حالة runtime PM للجهاز.

| الحقل | النوع | الوظيفة |
|-------|-------|---------|
| `usage_count` | `atomic_t` | عداد الاستخدام — لما بيوصل صفر يبدأ يفكر في suspend |
| `child_count` | `atomic_t` | عدد الأطفال اللي شغّالين — لو > 0 ممنوع suspend |
| `disable_depth` | `unsigned int:3` | عمق التعطيل — لو > 0 فـ runtime PM معطّل |
| `runtime_status` | `enum rpm_status` | الحالة الحالية (ACTIVE/SUSPENDED/...) |
| `last_status` | `enum rpm_status` | آخر حالة قبل system sleep |
| `request` | `enum rpm_request` | الـ request الـ pending في الـ work |
| `work` | `struct work_struct` | الـ work item للـ async operations |
| `suspend_timer` | `struct hrtimer` | timer الـ autosuspend |
| `timer_expires` | `u64` | وقت انتهاء الـ autosuspend timer |
| `wait_queue` | `wait_queue_head_t` | للـ synchronous waits |
| `autosuspend_delay` | `int` | تأخير الـ autosuspend بالـ milliseconds |
| `last_busy` | `u64` | آخر وقت استخدام (monotonic nanoseconds) |
| `active_time` | `u64` | إجمالي وقت النشاط |
| `suspended_time` | `u64` | إجمالي وقت النوم |
| `accounting_timestamp` | `u64` | نقطة البداية للحساب الحالي |
| `runtime_error` | `int` | آخر error من callback |
| `links_count` | `unsigned int` | عدد روابط الـ supplier |
| `wakeirq` | `struct wake_irq *` | الـ IRQ المسؤول عن الصحيان |
| `irq_safe` | `bool` | الـ callbacks ممكن تتنفذ من interrupt context |
| `use_autosuspend` | `bool` | هل autosuspend مفعّل |
| `no_callbacks` | `bool` | الجهاز ملوش runtime PM callbacks |
| `ignore_children` | `bool` | تجاهل الأطفال عند قرار الـ suspend |
| `memalloc_noio` | `bool` | منع الـ I/O في memory allocation |
| `runtime_auto` | `bool` | autosuspend مفعّل من userspace |
| `request_pending` | `bool` | في work مجدول في الـ workqueue |
| `deferred_resume` | `bool` | resume اتطلب وإحنا في suspend |
| `needs_force_resume` | `bool` | محتاج force resume بعد system resume |
| `idle_notification` | `bool` | idle notification شغّال دلوقتي |
| `timer_autosuspends` | `bool` | الـ timer هيعمل autosuspend مش suspend عادي |
| `qos` | `struct dev_pm_qos *` | متطلبات الـ QoS (latency constraints) |
| `subsys_data` | `struct pm_subsys_data *` | بيانات الـ subsystem (clocks, domain) |

#### `struct dev_pm_ops` — الـ callbacks

ده الـ vtable اللي الـ driver بيملّيه بالـ callbacks.

| الحقل | الوظيفة |
|-------|---------|
| `runtime_suspend` | اوقف تشغيل الجهاز (قطع power/clocks) |
| `runtime_resume` | صحّي الجهاز واعمله initialize |
| `runtime_idle` | قرّر إذا كان الجهاز يتعمله suspend |
| `suspend` / `resume` | system sleep callbacks |
| `freeze` / `thaw` | hibernation callbacks |
| `poweroff` / `restore` | hibernate + restore callbacks |
| `prepare` / `complete` | قبل وبعد كل نوع suspend |
| `suspend_noirq` ... `restore_noirq` | الـ noirq variants |

#### `struct dev_pm_domain` — الـ PM Domain

بيوفّر callbacks بديلة عن driver/bus callbacks لمجموعة أجهزة في نفس الـ power domain.

| الحقل | الوظيفة |
|-------|---------|
| `ops` | الـ PM callbacks للـ domain |
| `start` | لما user يحتاج يبدأ الجهاز |
| `detach` | لما الجهاز بيتشال من الـ domain |
| `activate` | قبل probe |
| `sync` | بعد نجاح probe |
| `dismiss` | بعد فشل probe أو بعد removal |
| `set_performance_state` | تغيير حالة الأداء |

#### `struct pm_subsys_data` — بيانات الـ Subsystem

| الحقل | الوظيفة |
|-------|---------|
| `lock` | `spinlock_t` يحمي الـ struct |
| `refcount` | عدد المراجع |
| `clock_list` | قائمة الـ clocks (لو `CONFIG_PM_CLK`) |
| `clock_mutex` | mutex للـ clock operations |
| `domain_data` | بيانات الـ PM domain (لو `CONFIG_PM_GENERIC_DOMAINS`) |

#### `struct pm_message` (= `pm_message_t`)

wrapper بسيط حوالين `int event` — بيحمل نوع transition (SUSPEND, FREEZE, ...).

---

### 2. Struct Relationship Diagram

```
struct device
    │
    └─► dev->power  [struct dev_pm_info]
            │
            ├─► usage_count     (atomic_t)
            ├─► child_count     (atomic_t)
            ├─► runtime_status  (enum rpm_status)
            ├─► request         (enum rpm_request)
            │
            ├─► work ──────────────────────────► pm_wq (workqueue_struct)
            │                                    (global PM workqueue)
            ├─► suspend_timer   (hrtimer)
            ├─► wait_queue      (wait_queue_head_t)
            ├─► wakeirq ───────────────────────► struct wake_irq
            │
            ├─► subsys_data ───────────────────► struct pm_subsys_data
            │                                         │
            │                                         ├─► clock_list
            │                                         └─► domain_data ──► struct pm_domain_data
            │
            └─► qos ───────────────────────────► struct dev_pm_qos

struct device
    │
    └─► dev->driver
            └─► driver->pm ────────────────────► struct dev_pm_ops
                                                   ├─► runtime_suspend()
                                                   ├─► runtime_resume()
                                                   └─► runtime_idle()

struct device
    │
    └─► dev->pm_domain ────────────────────────► struct dev_pm_domain
                                                   ├─► ops (dev_pm_ops)
                                                   └─► start/detach/activate/...

struct device_link
    ├─► supplier ──────────────────────────────► struct device (الـ supplier)
    └─► consumer ──────────────────────────────► struct device (الـ consumer)
         (pm_runtime_get_suppliers/put_suppliers تعمل get/put على الـ supplier)
```

---

### 3. Lifecycle Diagrams

#### Lifecycle الـ Driver مع Runtime PM

```
Driver Probe
    │
    ├─[1]─► pm_runtime_set_active(dev)
    │        أو pm_runtime_set_suspended(dev)
    │        (تحديد الحالة الابتدائية قبل enable)
    │
    ├─[2]─► pm_runtime_set_autosuspend_delay(dev, ms)   [اختياري]
    │
    ├─[3]─► pm_runtime_use_autosuspend(dev)             [اختياري]
    │
    ├─[4]─► pm_runtime_enable(dev)
    │        (ينقص dev->power.disable_depth)
    │        (لو وصل 0 → runtime PM أصبح active)
    │
    │   ══════════ الجهاز شغّال دلوقتي ══════════
    │
    │   [كل مرة Driver يستخدم الـ Hardware]
    ├───► pm_runtime_get_sync(dev)    ← بيزوّد usage_count وبيصحّي
    │        │
    │        ▼
    │     [شغّل hardware]
    │        │
    │        ▼
    └───► pm_runtime_put_autosuspend(dev)  ← بينقّص usage_count
              │
              ▼ (لو usage_count == 0)
           يجدول autosuspend بعد الـ delay
              │
              ▼
           dev->power.ops->runtime_suspend()
              │
              ▼
           RPM_SUSPENDED

Driver Remove
    │
    ├─[1]─► pm_runtime_dont_use_autosuspend(dev)
    ├─[2]─► pm_runtime_disable(dev)
    │        (يزوّد disable_depth، ينتظر operations جارية)
    └─[3]─► pm_runtime_put_noidle(dev)   [لو عمل get_noresume في probe]
```

#### Lifecycle الـ `devm` variant

```
Driver Probe
    │
    └─► devm_pm_runtime_enable(dev)
         │
         ▼
       يسجّل devres cleanup ──► عند remove تلقائيًا:
                                 pm_runtime_dont_use_autosuspend(dev)
                                 pm_runtime_disable(dev)
```

---

### 4. Call Flow Diagrams

#### pm_runtime_get_sync — المسار الكامل

```
driver calls pm_runtime_get_sync(dev)
    │
    └─► __pm_runtime_resume(dev, RPM_GET_PUT)
            │
            ├─ [check] disable_depth > 0? → return 1 (treated as active)
            ├─ [check] runtime_status == RPM_ACTIVE?
            │    └─ yes → atomic_inc(usage_count) → return 1
            │
            ├─ atomic_inc(&dev->power.usage_count)    ← RPM_GET_PUT
            │
            ├─ [RPM_ASYNC not set] → synchronous path
            │
            ├─ [acquire dev->power.lock]  ← spinlock
            │
            ├─ rpm_resume(dev, rpmflags)
            │     │
            │     ├─ [check] RPM_SUSPENDED?  → proceed
            │     ├─ [check] RPM_RESUMING?   → wait for completion
            │     ├─ [check] parent suspended? → resume parent first (recursive!)
            │     │
            │     ├─ set runtime_status = RPM_RESUMING
            │     │
            │     ├─ [release lock]
            │     │
            │     ├─ pm_domain ops->runtime_resume(dev)
            │     │    OR bus->pm->runtime_resume(dev)
            │     │       └─► driver's ops->runtime_resume(dev)
            │     │               └─► hardware registers/clocks
            │     │
            │     ├─ [acquire lock]
            │     │
            │     └─ set runtime_status = RPM_ACTIVE
            │
            └─ return 0 (success)
```

#### pm_runtime_put_autosuspend — المسار الكامل

```
driver calls pm_runtime_put_autosuspend(dev)
    │
    ├─► pm_runtime_mark_last_busy(dev)
    │     └─► WRITE_ONCE(dev->power.last_busy, ktime_get_mono_fast_ns())
    │
    └─► __pm_runtime_put_autosuspend(dev)
          └─► __pm_runtime_suspend(dev, RPM_GET_PUT | RPM_ASYNC | RPM_AUTO)
                │
                ├─ atomic_add_unless(&usage_count, -1, 0)   ← RPM_GET_PUT
                │   └─ [نقّص لو مش بالفعل 0]
                │
                ├─ [check] usage_count > 0? → return (مش وقت suspend)
                │
                ├─ [RPM_ASYNC] → queue_pm_work(&dev->power.work)
                │                 └─► pm_wq workqueue
                │                       └─► pm_runtime_work()
                │                             └─► rpm_suspend(dev, RPM_AUTO)
                │
                └─ [في الـ workqueue] rpm_autosuspend_expiration()
                      ├─ [وقت انتهى؟] → rpm_suspend()
                      │     ├─► dev->pm_domain->ops.runtime_suspend()
                      │     │     OR bus->pm->runtime_suspend()
                      │     │           └─► driver ops->runtime_suspend()
                      │     │                   └─► hardware power-down
                      │     └─► set runtime_status = RPM_SUSPENDED
                      └─ [لسه وقت؟] → hrtimer_start(suspend_timer, remaining)
```

#### pm_runtime_idle — مسار الـ idle check

```
driver calls pm_runtime_idle(dev)
    │
    └─► __pm_runtime_idle(dev, 0)
            │
            └─► rpm_idle(dev, 0)
                    │
                    ├─ [check] usage_count > 0? → return -EAGAIN
                    ├─ [check] child_count > 0? → return -EBUSY
                    ├─ [check] disable_depth > 0? → return -EACCES
                    ├─ [check] runtime_status != RPM_ACTIVE? → return -EAGAIN
                    │
                    ├─ set idle_notification = true
                    │
                    ├─► dev->pm_domain->ops.runtime_idle()
                    │     OR bus->pm->runtime_idle()
                    │           └─► driver ops->runtime_idle()
                    │                   [returns 0 → OK to suspend]
                    │
                    ├─ idle_notification = false
                    │
                    └─ [use_autosuspend?]
                         ├─ yes → rpm_autosuspend(dev) → set timer
                         └─ no  → rpm_suspend(dev)
```

#### pm_runtime_force_suspend — مسار system sleep

```
system sleep code
    │
    └─► pm_runtime_force_suspend(dev)   [بيتستخدم كـ .suspend callback]
            │
            ├─ pm_runtime_disable(dev)
            │   └─ زوّد disable_depth → امنع runtime PM
            │
            ├─ [check] pm_runtime_status_suspended(dev)? → skip callback, done
            │
            ├─► dev->pm_domain->ops.runtime_suspend(dev)
            │     OR dev->driver->pm->runtime_suspend(dev)
            │           └─► hardware power-down
            │
            └─ set runtime_status = RPM_SUSPENDED
                set needs_force_resume = true

...later on resume...
    └─► pm_runtime_force_resume(dev)
            │
            ├─ [check] needs_force_resume? → call runtime_resume()
            │
            ├─► dev->driver->pm->runtime_resume(dev)
            │
            └─ pm_runtime_enable(dev)
                └─ نقّص disable_depth → فعّل runtime PM تاني
```

---

### 5. State Machine Diagram

```
                    ┌──────────────────────────────────────────┐
                    │           RPM_ACTIVE                     │
                    │   usage_count > 0: الجهاز مستخدم         │
                    └──────────┬─────────────────┬─────────────┘
                               │                 ▲
              usage_count→0   │                 │ pm_runtime_resume()
              pm_runtime_idle()│                 │ pm_runtime_get_sync()
                               ▼                 │
                    ┌──────────────────────┐     │
                    │  rpm_idle() runs     │     │
                    │  driver idle check   │     │
                    └──────────┬───────────┘     │
                               │ returns 0        │
                               ▼                 │
                    ┌──────────────────────────────────────────┐
                    │          RPM_SUSPENDING                  │
                    │  driver->runtime_suspend() running       │
                    └──────────┬───────────────────────────────┘
                               │ success
                               ▼
                    ┌──────────────────────────────────────────┐
                    │          RPM_SUSPENDED                   │
                    │   الجهاز نايم — no power/clocks          │
                    └──────────┬───────────────────────────────┘
                               │ pm_runtime_get() / wakeup IRQ
                               ▼
                    ┌──────────────────────────────────────────┐
                    │          RPM_RESUMING                    │
                    │  driver->runtime_resume() running        │
                    └──────────┬───────────────────────────────┘
                               │ success
                               └──────────────────────────────► RPM_ACTIVE


Special state: disable_depth > 0
    ┌──────────────────────────────────────────────────────────┐
    │  runtime PM DISABLED — pm_runtime_enable/disable         │
    │  pm_runtime_active() returns true regardless of status   │
    │  all requests blocked                                     │
    └──────────────────────────────────────────────────────────┘
```

---

### 6. Autosuspend Flow Diagram

```
pm_runtime_put_autosuspend(dev)
    │
    ├─► mark_last_busy: last_busy = now_ns
    │
    └─► __pm_runtime_suspend(dev, GET_PUT | ASYNC | AUTO)
              │
              ▼
        [work item queued on pm_wq]
              │
              ▼
        pm_runtime_autosuspend_expiration(dev)
              │
              ├─ expiration = last_busy + autosuspend_delay_ns
              │
              ├─ [now >= expiration] ──────────────────► rpm_suspend() NOW
              │
              └─ [now < expiration] ──────► hrtimer_start(suspend_timer,
                                                           remaining_time)
                                                │
                                                ▼
                                          [timer fires]
                                                │
                                                ▼
                                          rpm_suspend(dev, RPM_AUTO)
                                                │
                                                ▼
                                          driver->runtime_suspend()
```

---

### 7. Device Links و Supplier/Consumer

```
Consumer Device ──── device_link ────► Supplier Device
    │                                       │
    │ pm_runtime_get_suppliers(consumer)    │
    │   └─► pm_runtime_get_sync(supplier) ─┘
    │       (صحّي الـ supplier قبل الـ consumer)
    │
    │ pm_runtime_put_suppliers(consumer)
    │   └─► pm_runtime_put(supplier)
    │       (نقّص usage_count الـ supplier)
    │
    └── pm_runtime_new_link(dev)   ← إضافة link جديد
        pm_runtime_drop_link(link) ← حذف link
        pm_runtime_release_supplier(link) ← تحرير الـ supplier
```

الـ `links_count` في `dev_pm_info` بيعدّ عدد روابط الـ supplier النشطة.

---

### 8. Locking Strategy

#### الـ Lock الأساسي: `dev->power.lock` (spinlock)

**بيحمي:**
- `runtime_status`
- `request`
- `usage_count` (بشكل غير مباشر — atomic لكن التغييرات المركبة تحتاج lock)
- `child_count`
- `disable_depth`
- `request_pending`
- `deferred_resume`
- `idle_notification`
- `timer_expires`
- `runtime_error`

**طريقة الاستخدام:**
```c
/* داخل الـ rpm_* functions */
spin_lock_irqsave(&dev->power.lock, flags);   /* IRQs disabled */
    /* ... critical section ... */
spin_unlock_irqrestore(&dev->power.lock, flags);
```

لما الـ `irq_safe` flag مضبوط على الجهاز → بيستخدم `spin_lock` بدون `irqsave` لأن الـ caller نفسه بيكون جاي من interrupt.

#### الـ `wait_queue` — للـ Synchronous Operations

لما `__pm_runtime_suspend/resume` بتتنفذ بشكل synchronous وفي operation جارية:
```
wait_event(dev->power.wait_queue,
           dev->power.runtime_status != RPM_SUSPENDING &&
           dev->power.runtime_status != RPM_RESUMING)
```

#### Lock Ordering (ترتيب الـ Locks)

```
child lock → parent lock    (مش العكس!)
    │
    └─ لما بنصحّي parent أثناء resume الـ child،
       بنـ release الـ child lock الأول
       ثم بنـ acquire الـ parent lock
```

قاعدة: **child يتلاك قبل parent** — ولو محتاج locks الاثنين، حرر الـ child lock الأول عشان تمسك الـ parent.

#### الـ `pm_subsys_data->lock` (spinlock)

بيحمي الـ `refcount` والـ `clock_list` الخاصين بالـ subsystem (bus/PM domain).

#### الـ `pm_subsys_data->clock_mutex` (mutex)

بيحمي عمليات الـ clock اللي ممكن تنام (`clock_op_might_sleep`).

#### ملخص Locking

| ما بيتحمى | الـ Lock المستخدم | نوعه |
|-----------|------------------|------|
| runtime_status, request, flags | `dev->power.lock` | spinlock + irqsave |
| parent/child coordination | `wait_queue` + `lock` | spinlock |
| subsystem refcount | `pm_subsys_data->lock` | spinlock |
| clock list operations | `pm_subsys_data->clock_mutex` | mutex |
| last_busy (autosuspend) | `WRITE_ONCE` / `READ_ONCE` | lock-free atomic |

---

### 9. الـ Guards و RAII Pattern

الـ header بيوفّر `DEFINE_GUARD` macros لتسهيل الـ get/put pattern:

```c
/* RAII style: يعمل get_sync عند الدخول، put عند الخروج */
DEFINE_GUARD(pm_runtime_active, struct device *,
             pm_runtime_get_sync(_T), pm_runtime_put(_T));

/* الاستخدام: */
guard(pm_runtime_active)(dev);
/* ... استخدم الـ hardware ... */
/* عند نهاية الـ scope: pm_runtime_put(dev) أتوماتيك */
```

| Guard | Get | Put |
|-------|-----|-----|
| `pm_runtime_noresume` | `get_noresume` | `put_noidle` |
| `pm_runtime_active` | `get_sync` | `put` |
| `pm_runtime_active_auto` | `get_sync` | `put_autosuspend` |
| `pm_runtime_active_try` | `get_active(TRANSPARENT)` | conditional |
| `pm_runtime_active_try_enabled` | `resume_and_get` | conditional |

الـ `PM_RUNTIME_ACQUIRE*` macros بتغلّف الـ `ACQUIRE()` للـ conditional guards — لازم تستخدم `PM_RUNTIME_ACQUIRE_ERR()` بعديها عشان تتحقق من النجاح.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Category 1: Core Low-Level Functions (exported from `drivers/base/power/runtime.c`)

| Function | Signature Summary | Purpose |
|---|---|---|
| `__pm_runtime_idle` | `(dev, rpmflags) → int` | تشغّل idle check وممكن تعمل suspend |
| `__pm_runtime_suspend` | `(dev, rpmflags) → int` | تعمل runtime-suspend للـ device |
| `__pm_runtime_resume` | `(dev, rpmflags) → int` | تعمل runtime-resume للـ device |
| `pm_schedule_suspend` | `(dev, delay_ms) → int` | تجدول suspend بعد delay معيّن |
| `pm_runtime_get_if_active` | `(dev) → int` | تزوّد الـ usage_count لو الـ device active |
| `pm_runtime_get_if_in_use` | `(dev) → int` | تزوّد الـ usage_count لو فيه استخدام فعلي |
| `__pm_runtime_set_status` | `(dev, status) → int` | تسيت الـ runtime_status يدوياً |
| `pm_runtime_barrier` | `(dev)` | تنتظر أي عملية PM جارية |
| `pm_runtime_force_suspend` | `(dev) → int` | تجبر الـ device على suspend (لـ system sleep) |
| `pm_runtime_force_resume` | `(dev) → int` | تجبر الـ device على resume (لـ system resume) |

#### Category 2: Enable / Disable / Allow / Forbid

| Function | Purpose |
|---|---|
| `pm_runtime_enable` | تفعّل runtime PM للـ device |
| `pm_runtime_disable` | تعطّل runtime PM (تزوّد disable_depth) |
| `__pm_runtime_disable` | نسخة lower-level مع خيار check_resume |
| `pm_runtime_allow` | تسمح بـ runtime PM من userspace |
| `pm_runtime_forbid` | تمنع runtime PM من userspace |
| `pm_runtime_block_if_disabled` | تبلوك لو كان runtime PM معطّل |
| `pm_runtime_unblock` | تلغي الـ block |

#### Category 3: Inline Wrappers (synchronous)

| Function | Flags passed internally | Behaviour |
|---|---|---|
| `pm_runtime_idle` | `0` | sync idle check |
| `pm_runtime_suspend` | `0` | sync suspend |
| `pm_runtime_resume` | `0` | sync resume |
| `pm_runtime_autosuspend` | `RPM_AUTO` | mark busy + sync autosuspend |
| `pm_runtime_get_sync` | `RPM_GET_PUT` | inc usage + sync resume |
| `pm_runtime_resume_and_get` | via `pm_runtime_get_active` | sync resume ثم inc usage لو نجحت |
| `pm_runtime_put_sync` | `RPM_GET_PUT` | dec usage + sync idle check |
| `pm_runtime_put_sync_suspend` | `RPM_GET_PUT` | dec usage + sync suspend |
| `pm_runtime_put_sync_autosuspend` | `RPM_GET_PUT\|RPM_AUTO` | mark busy + dec + sync autosuspend |

#### Category 4: Inline Wrappers (asynchronous)

| Function | Flags passed internally | Behaviour |
|---|---|---|
| `pm_request_idle` | `RPM_ASYNC` | async idle check |
| `pm_request_resume` | `RPM_ASYNC` | async resume |
| `pm_request_autosuspend` | `RPM_ASYNC\|RPM_AUTO` | mark busy + async autosuspend |
| `pm_runtime_get` | `RPM_GET_PUT\|RPM_ASYNC` | inc usage + async resume |
| `pm_runtime_put` | `RPM_GET_PUT\|RPM_ASYNC` | dec usage + async idle check |
| `pm_runtime_put_autosuspend` | via `__pm_runtime_put_autosuspend` | mark busy + dec + async autosuspend |

#### Category 5: Usage Counter Bare Ops

| Function | Operation |
|---|---|
| `pm_runtime_get_noresume` | `atomic_inc(usage_count)` — بدون resume |
| `pm_runtime_put_noidle` | `atomic_add_unless(usage_count, -1, 0)` — بدون idle check |

#### Category 6: Status Query (inline predicates)

| Function | Condition |
|---|---|
| `pm_runtime_suspended` | `runtime_status == RPM_SUSPENDED && !disable_depth` |
| `pm_runtime_active` | `runtime_status == RPM_ACTIVE \|\| disable_depth` |
| `pm_runtime_status_suspended` | `runtime_status == RPM_SUSPENDED` (regardless of enable) |
| `pm_runtime_enabled` | `!disable_depth` |
| `pm_runtime_blocked` | `last_status == RPM_BLOCKED` |
| `pm_runtime_has_no_callbacks` | `no_callbacks == true` |
| `pm_runtime_is_irq_safe` | `irq_safe == true` |

#### Category 7: Autosuspend Config

| Function | Purpose |
|---|---|
| `pm_runtime_use_autosuspend` | تفعّل autosuspend mechanism |
| `pm_runtime_dont_use_autosuspend` | تعطّل autosuspend mechanism |
| `__pm_runtime_use_autosuspend` | lower-level مع `bool use` |
| `pm_runtime_set_autosuspend_delay` | تحدد delay بالـ milliseconds |
| `pm_runtime_autosuspend_expiration` | تحسب وقت انتهاء الـ autosuspend delay |
| `pm_runtime_mark_last_busy` | تحدّث `last_busy` بالـ ktime monotonic |

#### Category 8: Device Links / Suppliers

| Function | Purpose |
|---|---|
| `pm_runtime_get_suppliers` | تزوّد usage_count لكل supplier |
| `pm_runtime_put_suppliers` | تنقص usage_count لكل supplier |
| `pm_runtime_new_link` | تسجّل device link جديد |
| `pm_runtime_drop_link` | تحذف device link |
| `pm_runtime_release_supplier` | تحرّر الـ supplier من device link |

#### Category 9: Devm (managed) APIs

| Function | Purpose |
|---|---|
| `devm_pm_runtime_enable` | تفعّل runtime PM مع auto-disable عند الـ detach |
| `devm_pm_runtime_set_active_enabled` | تسيت active + enable مع devm |
| `devm_pm_runtime_get_noresume` | تزوّد usage_count مع devm unwind |

#### Category 10: Misc Helpers

| Function | Purpose |
|---|---|
| `pm_runtime_no_callbacks` | تعلّم الـ device إنه ملوش PM callbacks |
| `pm_runtime_irq_safe` | تسمح بتشغيل PM callbacks من interrupt context |
| `pm_suspend_ignore_children` | تتجاهل child_count عند قرار الـ suspend |
| `pm_runtime_set_memalloc_noio` | تربط الـ device بـ GFP_NOIO memory allocation |
| `pm_runtime_set_active` | تسيت status = RPM_ACTIVE يدوياً |
| `pm_runtime_set_suspended` | تسيت status = RPM_SUSPENDED يدوياً |
| `pm_runtime_need_not_resume` | تسأل لو الـ device محتاجش resume بعد system suspend |
| `pm_runtime_suspended_time` | ترجع الوقت الكلي اللي قضاه الـ device في suspended state |
| `queue_pm_work` | تطرح work item في pm_wq |
| `pm_generic_runtime_suspend` | generic implementation لـ runtime_suspend callback |
| `pm_generic_runtime_resume` | generic implementation لـ runtime_resume callback |

#### Category 11: Guards (scoped locking patterns)

| Guard | Acquire | Release |
|---|---|---|
| `pm_runtime_noresume` | `get_noresume` | `put_noidle` |
| `pm_runtime_active` | `get_sync` | `put` |
| `pm_runtime_active_auto` | `get_sync` | `put_autosuspend` |
| `pm_runtime_active_try` | `get_active(RPM_TRANSPARENT)` | `put` |
| `pm_runtime_active_try_enabled` | `resume_and_get` | `put` |

---

### Group 1: Core Low-Level Engine Functions

الـ functions دي هي الـ heart الحقيقي لـ runtime PM framework. بتتعامل مباشرة مع الـ `dev->power` struct، بتأخد **rpmflags** اللي بتتحكم في السلوك (sync/async، increment/decrement، autosuspend، transparent).

الـ **RPM flags** المتاحة:

| Flag | Value | معناه |
|---|---|---|
| `RPM_ASYNC` | `0x01` | نفّذ بشكل async عبر pm_wq |
| `RPM_NOWAIT` | `0x02` | متستنيش لو في state change جارية |
| `RPM_GET_PUT` | `0x04` | dec/inc الـ usage_count كجزء من العملية |
| `RPM_AUTO` | `0x08` | استخدم autosuspend_delay بدل الـ suspend الفوري |
| `RPM_TRANSPARENT` | `0x10` | انجح (بدون عمل حاجة) لو runtime PM معطّل |

---

#### `__pm_runtime_idle`

```c
extern int __pm_runtime_idle(struct device *dev, int rpmflags);
```

**الـ function دي** بتنفّذ "idle check" للـ device — بتستدعي الـ `->runtime_idle()` callback لو موجود، ولو رجعت 0 أو مش موجودة، بتقرر إما تعمل autosuspend لو `RPM_AUTO` مفعّل أو تطلب suspend عادي. دي entry point لكل الـ wrappers اللي بتعمل idle check.

**Parameters:**
- `dev` — الـ device المستهدف
- `rpmflags` — combination من `RPM_ASYNC`, `RPM_NOWAIT`, `RPM_GET_PUT`, `RPM_AUTO`, `RPM_TRANSPARENT`

**Return:**
- `0` — نجاح
- `-EINVAL` — خطأ داخلي في الـ runtime PM state machine
- `-EACCES` — runtime PM معطّل (`disable_depth > 0`)
- `-EAGAIN` — usage_count != 0 أو في state change جارية أو الـ device مش في `RPM_ACTIVE`
- `-EBUSY` — `child_count > 0`
- `-EPERM` — resume latency constraint = 0
- `-EINPROGRESS` — suspend بدأت فعلاً
- `-ENOSYS` — `CONFIG_PM` مش مفعّل

**Key details:**
- بتاخد `dev->power.lock` (spinlock) داخلياً
- لو `RPM_ASYNC`: بتطرح `RPM_REQ_IDLE` في `pm_wq` وترجع فوراً
- لو `RPM_GET_PUT`: بتعمل decrement للـ usage_count — لو بقت أكبر من 0 ترجع بدون عمل حاجة

**Who calls it:** الـ wrappers `pm_runtime_idle()`, `pm_request_idle()`, `pm_runtime_put()`, `pm_runtime_put_sync()`

---

#### `__pm_runtime_suspend`

```c
extern int __pm_runtime_suspend(struct device *dev, int rpmflags);
```

**الـ function دي** بتطلب تحويل الـ device لـ `RPM_SUSPENDED` state. بتتحقق من الـ preconditions (usage_count == 0، child_count == 0، مش في RPM_SUSPENDED فعلاً)، ثم بتستدعي الـ `->runtime_suspend()` callback عبر الـ subsystem layers.

**Parameters:**
- `dev` — الـ device المستهدف
- `rpmflags` — نفس flags الـ `__pm_runtime_idle`

**Return:**
- `1` — الـ device كان suspended فعلاً
- `0` — نجاح: الـ device اتحوّل لـ `RPM_SUSPENDED`
- values سالبة — أخطاء متنوعة (زي `__pm_runtime_idle`)

**Key details:**
- لو `RPM_AUTO` مفعّل: بتحسب `autosuspend_expiration` وتحط `hrtimer` بدل ما تعمل suspend فوري
- لو `RPM_ASYNC`: بتطرح `RPM_REQ_SUSPEND` أو `RPM_REQ_AUTOSUSPEND` في `pm_wq`
- الـ callback بيتنفّذ في `RPM_SUSPENDING` state، ولو فشل الـ status بيرجع `RPM_ACTIVE`
- الـ `dev->power.runtime_error` بيتسيت لو الـ callback رجع error

**Pseudocode:**
```c
// داخل __pm_runtime_suspend (مبسّط)
spin_lock(&dev->power.lock);
if (dev->power.disable_depth)       return -EACCES;
if (dev->power.usage_count > 0)     return -EAGAIN;
if (dev->power.child_count > 0)     return -EBUSY;
if (status == RPM_SUSPENDED)        return 1;
if (RPM_AUTO && !expired)           arm_hrtimer(); return 0;
if (RPM_ASYNC)                      queue_pm_work(); return 0;
dev->power.runtime_status = RPM_SUSPENDING;
spin_unlock(&dev->power.lock);
ret = subsys->runtime_suspend(dev);
spin_lock(&dev->power.lock);
if (ret == 0)
    dev->power.runtime_status = RPM_SUSPENDED;
else
    dev->power.runtime_status = RPM_ACTIVE;
spin_unlock(&dev->power.lock);
```

**Who calls it:** `pm_runtime_suspend()`, `pm_runtime_autosuspend()`, `pm_runtime_put_sync_suspend()`, `pm_runtime_put_sync_autosuspend()`, `pm_request_autosuspend()`, `pm_runtime_put_autosuspend()`, `__pm_runtime_put_autosuspend()`

---

#### `__pm_runtime_resume`

```c
extern int __pm_runtime_resume(struct device *dev, int rpmflags);
```

**الـ function دي** بتطلب تحويل الـ device لـ `RPM_ACTIVE` state. لو الـ device suspended، بتستدعي `->runtime_resume()` callback، وبتتعامل مع الـ parent resume recursively لو لزم.

**Parameters:**
- `dev` — الـ device
- `rpmflags` — بالإضافة للـ flags العادية، `RPM_GET_PUT` هنا بيعمل **increment** للـ usage_count

**Return:**
- `1` — الـ device كان active فعلاً
- `0` — نجاح: الـ device اتحوّل لـ `RPM_ACTIVE`
- values سالبة — فشل الـ resume

**Key details:**
- لو `RPM_GET_PUT`: بتعمل `atomic_inc(usage_count)` **قبل** ما تستدعي الـ callback — ده السبب إن `pm_runtime_get_sync` بيفضل الـ usage_count مرفوع حتى لو فشلت
- لو `RPM_ASYNC`: بتطرح `RPM_REQ_RESUME` في `pm_wq`
- بتوقظ الـ parent قبل الـ child تلقائياً
- الـ `RPM_TRANSPARENT` flag: لو runtime PM معطّل، ترجع `0` (success) بدل `-EACCES`

**Who calls it:** كل resume wrappers، وكمان داخلياً من kernel لما يكون في pending work

---

#### `pm_schedule_suspend`

```c
extern int pm_schedule_suspend(struct device *dev, unsigned int delay);
```

**الـ function دي** بتجدول runtime suspend بعد `delay` milliseconds عبر الـ `hrtimer` الخاص بالـ device. مختلفة عن autosuspend لأن الـ delay بتحددها أنت مش `autosuspend_delay`.

**Parameters:**
- `dev` — الـ device
- `delay` — الوقت بالـ milliseconds قبل الـ suspend

**Return:**
- `0` — الـ timer اتحط أو الـ device suspended فعلاً
- أخطاء سالبة — runtime PM disabled أو usage_count != 0

**Key details:** بتسيت `power.timer_autosuspends = 0` عشان تفرّق بينها وبين autosuspend timer

---

#### `pm_runtime_get_if_active`

```c
extern int pm_runtime_get_if_active(struct device *dev);
```

**الـ function دي** بتزوّد الـ usage_count بـ 1 **فقط لو** الـ device في `RPM_ACTIVE` state. مفيدة لما عايز تضمن إن الـ device شغّال فعلاً قبل ما تستخدمه بدون ما تعمل resume.

**Return:**
- `1` — الـ device كان active وتم الـ increment
- `0` — الـ device مش active، مفيش increment
- `-EINVAL` — `CONFIG_PM` مش مفعّل

**Who calls it:** drivers اللي بتستخدم IRQ handlers وعايزه يكون atomic

---

#### `pm_runtime_get_if_in_use`

```c
extern int pm_runtime_get_if_in_use(struct device *dev);
```

**الـ function دي** بتزوّد الـ usage_count بـ 1 **فقط لو** كانت قيمته الحالية أكبر من 0 (يعني في حد تاني بيستخدم الـ device). مفيدة للـ shared resources.

**Return:**
- `> 0` — نجاح: القيمة الجديدة للـ usage_count
- `0` — الـ usage_count كان 0، مفيش increment
- `-EINVAL` — `CONFIG_PM` مش مفعّل

---

#### `__pm_runtime_set_status`

```c
extern int __pm_runtime_set_status(struct device *dev, unsigned int status);
```

**الـ function دي** بتسيت الـ `runtime_status` يدوياً بدون ما تعمل أي callbacks. **بتُستخدم فقط لما runtime PM معطّل** (يعني `disable_depth > 0`)، عادةً في الـ `probe` أو `remove`.

**Parameters:**
- `dev` — الـ device
- `status` — إما `RPM_ACTIVE` أو `RPM_SUSPENDED`

**Return:**
- `0` — نجاح
- `-EINVAL` — محاولة سيت status وruntime PM مفعّل، أو status غلط

**Key details:** لو بتسيت `RPM_ACTIVE`، بتزوّد الـ `child_count` للـ parent تلقائياً. اللي بيستدعيها بيكون مسؤول عن الـ locking الصحيح.

---

### Group 2: Enable / Disable / Control

#### `pm_runtime_enable`

```c
extern void pm_runtime_enable(struct device *dev);
```

**الـ function دي** بتنقص `disable_depth` بواحد. لو وصلت لـ 0، runtime PM بقى مفعّل فعلياً للـ device. لازم يكون فيه matching call مع كل `pm_runtime_disable`.

**Key details:**
- لو الـ `disable_depth` بقت 0 ومحتاج resume pending، بيتبعت فوراً
- بتاخد `power.lock`
- **مش** thread-safe لو اتستدعت من أكتر من thread في نفس الوقت بدون synchronization خارجي

---

#### `__pm_runtime_disable` / `pm_runtime_disable`

```c
extern void __pm_runtime_disable(struct device *dev, bool check_resume);
static inline void pm_runtime_disable(struct device *dev)
{
    __pm_runtime_disable(dev, true);
}
```

**الـ function دي** بتزوّد `disable_depth` بواحد، وبتلغي أي pending PM requests، وبتنتظر أي operations جارية تخلص. لو `check_resume = true`، بتعمل resume للـ device الأول لو في pending resume request.

**Key details:**
- بتستدعي `pm_runtime_barrier` داخلياً لضمان إن مفيش operations جارية
- **كل** استدعاء محتاج matching `pm_runtime_enable`
- من الـ `pm_runtime_disable` public API، `check_resume = true` دايماً

---

#### `pm_runtime_allow` / `pm_runtime_forbid`

```c
extern void pm_runtime_allow(struct device *dev);
extern void pm_runtime_forbid(struct device *dev);
```

**الـ function دول** بتتحكم في runtime PM من **userspace** عبر الـ sysfs file `power/control`. الـ `allow` بتعمل `put` (تنقص usage_count وتسمح بالـ idle)، والـ `forbid` بتعمل `get` (تزوّد usage_count وتمنع الـ suspend).

**Key details:** الـ runtime PM framework بيحسب حالة `runtime_auto` flag — لو `runtime_auto = false` ده معناه `forbid` اتعملت

---

#### `pm_runtime_block_if_disabled` / `pm_runtime_unblock`

```c
extern bool pm_runtime_block_if_disabled(struct device *dev);
extern void pm_runtime_unblock(struct device *dev);
```

**الـ `block_if_disabled`** بتعمل increment لـ `usage_count` لو runtime PM معطّل، عشان تمنع accidental suspend. **الـ `unblock`** بتعمل decrement للـ usage_count وممكن تشغّل idle check.

**Return لـ block_if_disabled:**
- `true` — runtime PM معطّل وتم الـ block (usage_count ++ )
- `false` — runtime PM مفعّل، مفيش block

---

### Group 3: Synchronous Inline Wrappers

#### `pm_runtime_idle`

```c
static inline int pm_runtime_idle(struct device *dev)
{
    return __pm_runtime_idle(dev, 0);
}
```

**الـ function دي** بتعمل idle check بشكل synchronous — بتستدعي `->runtime_idle()` مباشرةً في نفس الـ context. ممكن تبلوك لو الـ device بيعمل state transition دلوقتي.

**Return:** نفس `__pm_runtime_idle` — `0` نجاح، values سالبة أخطاء

**Who calls it:** عادةً drivers بعد ما تخلص من operation وعايزه يعمل idle check فوراً

---

#### `pm_runtime_suspend`

```c
static inline int pm_runtime_suspend(struct device *dev)
{
    return __pm_runtime_suspend(dev, 0);
}
```

**الـ function دي** بتعمل runtime-suspend synchronous بدون increment/decrement للـ usage_count. لازم الـ usage_count يكون 0 قبل الاستدعاء، غير كده ترجع `-EAGAIN`.

---

#### `pm_runtime_resume`

```c
static inline int pm_runtime_resume(struct device *dev)
{
    return __pm_runtime_resume(dev, 0);
}
```

**الـ function دي** بتعمل runtime-resume synchronous بدون تغيير في الـ usage_count. مفيدة لو عايز تضمن إن الـ device شغّال قبل عملية معينة بدون ما تزوّد الـ counter.

---

#### `pm_runtime_autosuspend`

```c
static inline int pm_runtime_autosuspend(struct device *dev)
{
    pm_runtime_mark_last_busy(dev);
    return __pm_runtime_suspend(dev, RPM_AUTO);
}
```

**الـ function دي** بتحدّث `last_busy` بالوقت الحالي وبعدين بتطلب autosuspend. لو autosuspend مش مفعّل، بتعمل suspend عادي فوراً. لو مفعّل، بتحسب لو الـ delay انتهى أو لا.

---

#### `pm_runtime_get_sync`

```c
static inline int pm_runtime_get_sync(struct device *dev)
{
    return __pm_runtime_resume(dev, RPM_GET_PUT);
}
```

**الـ function دي** بتزوّد الـ usage_count وبتعمل synchronous resume. **ملاحظة مهمة:** حتى لو الـ resume فشل، الـ usage_count بيفضل مزوّد. ده بيخلي error handling صعب — الأحسن تستخدم `pm_runtime_resume_and_get` بدلها.

**Return:** نفس `__pm_runtime_resume`، لكن usage_count دايماً incremented

---

#### `pm_runtime_resume_and_get`

```c
static inline int pm_runtime_resume_and_get(struct device *dev)
{
    return pm_runtime_get_active(dev, 0);
}
```

**الـ function دي** بتعمل synchronous resume، ولو نجحت تزوّد الـ usage_count. لو فشلت، **مبتزوّدش** الـ usage_count وبتعمل `put_noidle` للـ decrement. ده behavior أنظف من `get_sync`.

**Pseudocode داخل `pm_runtime_get_active`:**
```c
ret = __pm_runtime_resume(dev, RPM_GET_PUT | rpmflags);
if (ret < 0) {
    pm_runtime_put_noidle(dev); /* undo the increment */
    return ret;
}
return 0;
```

---

#### `pm_runtime_put_sync`

```c
static inline int pm_runtime_put_sync(struct device *dev)
{
    return __pm_runtime_idle(dev, RPM_GET_PUT);
}
```

**الـ function دي** بتنقص الـ usage_count وبتعمل idle check synchronous. لو الـ usage_count وصل 0، بتستدعي `->runtime_idle()` ثم ممكن تعمل suspend.

---

#### `pm_runtime_put_sync_suspend`

```c
static inline int pm_runtime_put_sync_suspend(struct device *dev)
{
    return __pm_runtime_suspend(dev, RPM_GET_PUT);
}
```

**الـ function دي** بتنقص الـ usage_count وبتحاول تعمل suspend مباشرةً **بدون** idle check. لو الـ usage_count لسه أكبر من 0 ترجع `-EAGAIN`.

---

#### `pm_runtime_put_sync_autosuspend`

```c
static inline int pm_runtime_put_sync_autosuspend(struct device *dev)
{
    pm_runtime_mark_last_busy(dev);
    return __pm_runtime_suspend(dev, RPM_GET_PUT | RPM_AUTO);
}
```

**الـ function دي** بتحدّث `last_busy`، بتنقص الـ usage_count، وبتشغّل autosuspend mechanism. لو الـ autosuspend delay لسه ما انتهيش، بتحط hrtimer وترجع فوراً.

---

### Group 4: Asynchronous Inline Wrappers

#### `pm_request_idle`

```c
static inline int pm_request_idle(struct device *dev)
{
    return __pm_runtime_idle(dev, RPM_ASYNC);
}
```

**الـ function دي** بتطرح idle check request في `pm_wq` وترجع فوراً. مفيدة من interrupt context أو من contexts مش مناسب فيها blocking.

---

#### `pm_request_resume`

```c
static inline int pm_request_resume(struct device *dev)
{
    return __pm_runtime_resume(dev, RPM_ASYNC);
}
```

**الـ function دي** بتطرح resume request في `pm_wq` وترجع فوراً. **مش** بتزوّد الـ usage_count — لو محتاج تضمن الـ device يفضل active لازم تستخدم `pm_runtime_get` بدلها.

---

#### `pm_request_autosuspend`

```c
static inline int pm_request_autosuspend(struct device *dev)
{
    pm_runtime_mark_last_busy(dev);
    return __pm_runtime_suspend(dev, RPM_ASYNC | RPM_AUTO);
}
```

**الـ function دي** بتحدّث `last_busy` وبتطرح autosuspend request بشكل async في `pm_wq`. مشابهة لـ `pm_runtime_autosuspend` لكن async.

---

#### `pm_runtime_get`

```c
static inline int pm_runtime_get(struct device *dev)
{
    return __pm_runtime_resume(dev, RPM_GET_PUT | RPM_ASYNC);
}
```

**الـ function دي** بتزوّد الـ usage_count وبتطرح resume request async. ترجع فوراً. مفيدة لما عايز تضمن الـ device هيتفتح لكن مش محتاج تستنى.

**Key detail:** الـ usage_count بيتزوّد **دلوقتي** حتى لو الـ resume لسه ما اتنفّذش — ده بيحمي الـ device من الـ suspend قبل ما الـ resume يخلص.

---

#### `pm_runtime_put`

```c
static inline int pm_runtime_put(struct device *dev)
{
    return __pm_runtime_idle(dev, RPM_GET_PUT | RPM_ASYNC);
}
```

**الـ function دي** بتنقص الـ usage_count وبتطرح idle check request async لو وصلت 0. المقابل الـ async لـ `pm_runtime_put_sync`.

---

#### `pm_runtime_put_autosuspend`

```c
static inline int pm_runtime_put_autosuspend(struct device *dev)
{
    pm_runtime_mark_last_busy(dev);
    return __pm_runtime_put_autosuspend(dev);
}

static inline int __pm_runtime_put_autosuspend(struct device *dev)
{
    return __pm_runtime_suspend(dev, RPM_GET_PUT | RPM_ASYNC | RPM_AUTO);
}
```

**الـ function دي** بتحدّث `last_busy`، بتنقص الـ usage_count، وبتطرح autosuspend request async. ده الـ pattern الشائع في drivers اللي بتستخدم autosuspend.

**Pattern نموذجي في driver:**
```c
/* طلب resume قبل عملية */
pm_runtime_get_sync(dev);
/* do work */
do_hardware_operation(dev);
/* إطلاق مع autosuspend */
pm_runtime_mark_last_busy(dev);
pm_runtime_put_autosuspend(dev);
```

---

### Group 5: Bare Usage Counter Operations

#### `pm_runtime_get_noresume`

```c
static inline void pm_runtime_get_noresume(struct device *dev)
{
    atomic_inc(&dev->power.usage_count);
}
```

**الـ function دي** بتزوّد الـ `usage_count` **فقط** بدون ما تشغّل resume. مفيدة لما عايز تمنع الـ device من الـ suspend أثناء الـ probe أو الـ initialization قبل ما تفعّل runtime PM.

**Pattern شائع:**
```c
pm_runtime_get_noresume(dev);  /* prevent suspend during probe */
pm_runtime_set_active(dev);
pm_runtime_enable(dev);
/* later, after device is ready */
pm_runtime_put(dev);           /* allow suspend */
```

---

#### `pm_runtime_put_noidle`

```c
static inline void pm_runtime_put_noidle(struct device *dev)
{
    atomic_add_unless(&dev->power.usage_count, -1, 0);
}
```

**الـ function دي** بتنقص الـ `usage_count` بدون ما تشغّل idle check. المقابل لـ `pm_runtime_get_noresume`. الـ `atomic_add_unless` بتضمن إن الـ counter ميقعش تحت 0.

---

### Group 6: Status Query Functions (Predicates)

#### `pm_runtime_suspended`

```c
static inline bool pm_runtime_suspended(struct device *dev)
{
    return dev->power.runtime_status == RPM_SUSPENDED
        && !dev->power.disable_depth;
}
```

**الـ function دي** بترجع `true` لو الـ device في suspended state **وكمان** runtime PM مفعّل. ده يعني لو runtime PM معطّل، الـ device "مش suspended" من وجهة نظر الـ framework.

**Key detail:** النتيجة مش موثوقة إلا لو أنت في `power.lock` أو في conditions مينقلوش فيها الـ status.

---

#### `pm_runtime_active`

```c
static inline bool pm_runtime_active(struct device *dev)
{
    return dev->power.runtime_status == RPM_ACTIVE
        || dev->power.disable_depth;
}
```

**الـ function دي** بترجع `true` لو الـ device active **أو** لو runtime PM معطّل. ده منطقي لأنه لو runtime PM معطّل، الـ device بيتعامل معاه كـ "active دايماً" من وجهة نظر الـ callers.

---

#### `pm_runtime_status_suspended`

```c
static inline bool pm_runtime_status_suspended(struct device *dev)
{
    return dev->power.runtime_status == RPM_SUSPENDED;
}
```

**الـ function دي** بتتحقق من الـ `runtime_status` مباشرةً **بغض النظر** عن `disable_depth`. مفيدة لو عايز تعرف الـ status الفعلي حتى لو runtime PM معطّل.

---

#### `pm_runtime_enabled`

```c
static inline bool pm_runtime_enabled(struct device *dev)
{
    return !dev->power.disable_depth;
}
```

**الـ function دي** بتتحقق لو runtime PM مفعّل (يعني `disable_depth == 0`). بسيطة جداً لكن مهمة للـ drivers اللي محتاجة تعرف الـ state قبل ما تعمل operations.

---

#### `pm_runtime_blocked`

```c
static inline bool pm_runtime_blocked(struct device *dev)
{
    return dev->power.last_status == RPM_BLOCKED;
}
```

**الـ function دي** بتتحقق لو runtime PM enabling اتبلوك. **ملاحظة مهمة:** التعليق في الكود بيقول "Do not call this function outside system suspend/resume code paths" — يعني للاستخدام الداخلي في system sleep paths فقط.

---

#### `pm_runtime_has_no_callbacks`

```c
static inline bool pm_runtime_has_no_callbacks(struct device *dev)
{
    return dev->power.no_callbacks;
}
```

**الـ function دي** بترجع `true` لو الـ device اتعلّم عليه إنه ملوش runtime PM callbacks. الـ framework بيتعامل مع الـ device ده بشكل مختلف — suspend/resume operations بتنجح فوراً بدون استدعاء callbacks.

---

#### `pm_runtime_is_irq_safe`

```c
static inline bool pm_runtime_is_irq_safe(struct device *dev)
{
    return dev->power.irq_safe;
}
```

**الـ function دي** بترجع `true` لو الـ device تم وضع علامة عليه كـ "IRQ-safe". ده بيعني إن الـ `pm_wq` بتستخدم `WQ_UNBOUND` mode خاص وإن الـ callbacks ممكن تتنفّذ من interrupt context.

---

### Group 7: Autosuspend Configuration

#### `pm_runtime_mark_last_busy`

```c
static inline void pm_runtime_mark_last_busy(struct device *dev)
{
    WRITE_ONCE(dev->power.last_busy, ktime_get_mono_fast_ns());
}
```

**الـ function دي** بتحدّث `dev->power.last_busy` بالوقت الحالي من monotonic clock. ده بيُستخدم من الـ autosuspend mechanism لحساب متى آخر مرة الـ device كان busy.

**Key detail:** بتستخدم `WRITE_ONCE` و`ktime_get_mono_fast_ns` اللي safe من interrupt context وبدون locking.

**Who calls it:** drivers بعد كل عملية hardware تخلص، وكمان `pm_runtime_autosuspend`, `pm_runtime_put_autosuspend`, إلخ.

---

#### `pm_runtime_set_autosuspend_delay`

```c
extern void pm_runtime_set_autosuspend_delay(struct device *dev, int delay);
```

**الـ function دي** بتسيت `dev->power.autosuspend_delay` بالـ milliseconds. ممكن تسيتها من driver أو من userspace عبر sysfs.

**Parameters:**
- `delay` — بالـ milliseconds. القيمة السالبة بتعني "لا تعمل autosuspend أبداً". القيمة 0 بتعني suspend فوراً.

**Key detail:** لو autosuspend مش enabled (`use_autosuspend = false`)، الـ delay بيتتجاهل والـ suspend بيحصل فوراً.

---

#### `pm_runtime_autosuspend_expiration`

```c
extern u64 pm_runtime_autosuspend_expiration(struct device *dev);
```

**الـ function دي** بتحسب الـ timestamp اللي المفروض يحصل فيه الـ autosuspend بناءً على `last_busy + autosuspend_delay`. الـ framework بيستخدمها لتحديد لو حان وقت الـ suspend.

**Return:**
- `> 0` — الـ timestamp (بالـ nanoseconds) اللي المفروض يحصل فيه الـ suspend
- `0` — الـ delay انتهى أو autosuspend مش مفعّل

---

#### `__pm_runtime_use_autosuspend` / `pm_runtime_use_autosuspend` / `pm_runtime_dont_use_autosuspend`

```c
extern void __pm_runtime_use_autosuspend(struct device *dev, bool use);

static inline void pm_runtime_use_autosuspend(struct device *dev)
{
    __pm_runtime_use_autosuspend(dev, true);
}

static inline void pm_runtime_dont_use_autosuspend(struct device *dev)
{
    __pm_runtime_use_autosuspend(dev, false);
}
```

**الـ function دي** بتسيت `dev->power.use_autosuspend`. لو بتعطّل autosuspend (`use = false`)، بتحاول تعمل suspend فوري لو الـ usage_count كانت 0.

**Key detail:** التعليق في الكود بيوضح إن `pm_runtime_dont_use_autosuspend` لازم يتعمل في driver exit time إلا لو استخدمت `devm_pm_runtime_enable`.

---

### Group 8: Device Links & Supplier Management

#### `pm_runtime_get_suppliers`

```c
extern void pm_runtime_get_suppliers(struct device *dev);
```

**الـ function دي** بتمشي على كل الـ device links اللي الـ `dev` ده consumer فيها وبتعمل `pm_runtime_get_sync` لكل supplier. بتضمن إن كل الـ suppliers شغّالين قبل ما الـ consumer يتوقظ.

---

#### `pm_runtime_put_suppliers`

```c
extern void pm_runtime_put_suppliers(struct device *dev);
```

**الـ function دي** المقابل لـ `pm_runtime_get_suppliers` — بتعمل `pm_runtime_put` لكل supplier بعد ما الـ consumer خلص شغله.

---

#### `pm_runtime_new_link` / `pm_runtime_drop_link`

```c
extern void pm_runtime_new_link(struct device *dev);
extern void pm_runtime_drop_link(struct device_link *link);
```

**الـ function دول** بتتعاملان مع الـ `dev->power.links_count`. لما link جديد بيتضاف، بيتزوّد الـ counter وبتتأخذ خطوات عشان الـ supplier يفضل active. لما link بيتحذف، بيتنقص الـ counter.

---

#### `pm_runtime_release_supplier`

```c
extern void pm_runtime_release_supplier(struct device_link *link);
```

**الـ function دي** بتحرّر الـ supplier من device link معين — بتعمل `pm_runtime_put` للـ supplier device المرتبط بالـ link ده.

---

### Group 9: Managed (devm) APIs

#### `devm_pm_runtime_enable`

```c
extern int devm_pm_runtime_enable(struct device *dev);
```

**الـ function دي** بتعمل `pm_runtime_enable` وبتسجّل cleanup function بتستدعي `pm_runtime_disable` أوتوماتيك لما الـ device بيتحذف. بتسيت كمان `dont_use_autosuspend` في الـ cleanup.

**Return:** `0` دايماً (لكن في النسخ القديمة كانت ممكن ترجع error)

**Key detail:** بتتعامل مع الـ autosuspend cleanup أوتوماتيك — ده السبب اللي ذكر في تعليق `pm_runtime_use_autosuspend` إنه مش لازم تعمل `dont_use_autosuspend` يدوياً لو استخدمت `devm_pm_runtime_enable`.

---

#### `devm_pm_runtime_set_active_enabled`

```c
int devm_pm_runtime_set_active_enabled(struct device *dev);
```

**الـ function دي** بتعمل `pm_runtime_set_active` + `pm_runtime_enable` معاً، وبتسجّل cleanup function. مفيدة في الـ probe function لو الـ device بدأ active.

---

#### `devm_pm_runtime_get_noresume`

```c
int devm_pm_runtime_get_noresume(struct device *dev);
```

**الـ function دي** بتعمل `pm_runtime_get_noresume` وبتسجّل cleanup بتعمل `pm_runtime_put_noidle` تلقائياً عند الـ detach.

---

### Group 10: Generic Callbacks & Force Suspend/Resume

#### `pm_generic_runtime_suspend` / `pm_generic_runtime_resume`

```c
extern int pm_generic_runtime_suspend(struct device *dev);
extern int pm_generic_runtime_resume(struct device *dev);
```

**الـ function دول** هي الـ generic implementation للـ runtime PM callbacks اللي ممكن الـ bus types تستخدمها. بتستدعي الـ `->runtime_suspend` و`->runtime_resume` من الـ `dev->driver->pm` struct مباشرةً.

---

#### `pm_runtime_force_suspend`

```c
extern int pm_runtime_force_suspend(struct device *dev);
```

**الـ function دي** بتُستخدم كـ system sleep callback (`.suspend`) في الـ `DEFINE_RUNTIME_DEV_PM_OPS` macro. بتجبر الـ device على الـ runtime suspend state عشان الـ system sleep يكمل.

**Pseudocode:**
```c
// pm_runtime_force_suspend (مبسّط)
if (pm_runtime_status_suspended(dev))
    return 0; /* already suspended, nothing to do */
ret = pm_runtime_disable(dev);        /* prevent new PM operations */
pm_runtime_set_suspended(dev);        /* update status */
/* synchronously call runtime_suspend callback */
ret = dev->pm_domain->ops.runtime_suspend(dev);
if (ret)
    pm_runtime_enable(dev);           /* undo on failure */
```

**Key detail:** بتسيت `power.needs_force_resume = 1` عشان الـ `pm_runtime_force_resume` تعرف تعمل resume لما الـ system يصحى.

---

#### `pm_runtime_force_resume`

```c
int pm_runtime_force_resume(struct device *dev);
```

**الـ function دي** (مُعرَّفة بشرط `CONFIG_PM_SLEEP`) المقابل لـ `pm_runtime_force_suspend` — بتعمل resume للـ device بعد system wake-up. بتتحقق من `needs_force_resume` flag الأول.

---

### Group 11: Miscellaneous Helpers

#### `pm_runtime_no_callbacks`

```c
extern void pm_runtime_no_callbacks(struct device *dev);
```

**الـ function دي** بتسيت `dev->power.no_callbacks = 1`. الـ runtime PM framework بعد كده بيتعامل مع الـ device كـ "always ready" ومش محتاج يستدعي أي callbacks. مفيدة للـ virtual devices أو الـ devices اللي الـ PM بيتعمل على مستوى أعلى.

---

#### `pm_runtime_irq_safe`

```c
extern void pm_runtime_irq_safe(struct device *dev);
```

**الـ function دي** بتسيت `dev->power.irq_safe = 1`. ده بيأثر على الـ `pm_wq` work queue — الـ callbacks بتتنفّذ في context مناسب للـ interrupt. كمان بيأثر على الـ parent (لازم الـ parent يكون irq-safe كمان أو يكون marked `no_callbacks`).

**Key detail:** بتستدعي `pm_runtime_get_noresume` على الـ parent عشان تمنعه من الـ suspend لو الـ child irq-safe.

---

#### `pm_suspend_ignore_children`

```c
static inline void pm_suspend_ignore_children(struct device *dev, bool enable)
{
    dev->power.ignore_children = enable;
}
```

**الـ function دي** بتسيت `ignore_children` flag. لو `true`، الـ runtime PM framework بيتجاهل الـ `child_count` عند قرار الـ suspend — يعني الـ device ممكن يتسسبند حتى لو فيه children active.

---

#### `pm_runtime_set_memalloc_noio`

```c
extern void pm_runtime_set_memalloc_noio(struct device *dev, bool enable);
```

**الـ function دي** بتسيت `dev->power.memalloc_noio`. ده بيأثر على الـ memory allocation policy في الـ runtime PM context — بيمنع `GFP_IO` operations من الـ PM callbacks لتجنب deadlocks في memory pressure scenarios.

---

#### `pm_runtime_set_active` / `pm_runtime_set_suspended`

```c
static inline int pm_runtime_set_active(struct device *dev)
{
    return __pm_runtime_set_status(dev, RPM_ACTIVE);
}

static inline int pm_runtime_set_suspended(struct device *dev)
{
    return __pm_runtime_set_status(dev, RPM_SUSPENDED);
}
```

**الـ function دول** بتستدعي `__pm_runtime_set_status` لضبط الـ status يدوياً. **لازم** تتنادوا لما runtime PM معطّل (`disable_depth > 0`) فقط — عادةً في الـ probe قبل `pm_runtime_enable`.

---

#### `pm_runtime_need_not_resume`

```c
bool pm_runtime_need_not_resume(struct device *dev);
```

**الـ function دي** (بشرط `CONFIG_PM_SLEEP`) بتتحقق لو الـ device محتاجش resume بعد system suspend. ده بيسمح للـ system suspend path بتخطّي resume operations للـ devices اللي كانت suspended فعلاً قبل الـ system sleep.

---

#### `pm_runtime_suspended_time`

```c
extern u64 pm_runtime_suspended_time(struct device *dev);
```

**الـ function دي** بترجع الوقت الإجمالي (بالـ nanoseconds) اللي قضاه الـ device في `RPM_SUSPENDED` state. مفيدة لـ power accounting وـ debugging.

---

#### `queue_pm_work`

```c
static inline bool queue_pm_work(struct work_struct *work)
{
    return queue_work(pm_wq, work);
}
```

**الـ function دي** بتطرح work item في الـ `pm_wq` work queue الخاص بالـ runtime PM. الـ `pm_wq` بيتعرف في `drivers/base/power/runtime.c`.

**Return:** `true` لو الـ work اتضاف بنجاح، `false` لو كان موجود فعلاً في الـ queue

---

### Group 12: Scoped Guards (Lock-like Patterns)

الـ guards دي بتستخدم kernel's `DEFINE_GUARD` / `DEFINE_GUARD_COND` macros اللي بتوفّر scoped resource management.

```c
DEFINE_GUARD(pm_runtime_noresume, struct device *,
             pm_runtime_get_noresume(_T), pm_runtime_put_noidle(_T));

DEFINE_GUARD(pm_runtime_active, struct device *,
             pm_runtime_get_sync(_T), pm_runtime_put(_T));

DEFINE_GUARD(pm_runtime_active_auto, struct device *,
             pm_runtime_get_sync(_T), pm_runtime_put_autosuspend(_T));
```

**الـ guards دول** بيوفّروا RAII-style management للـ runtime PM references:

| Guard | Acquire | Release | Use Case |
|---|---|---|---|
| `pm_runtime_noresume` | `get_noresume` | `put_noidle` | منع suspend بدون resume |
| `pm_runtime_active` | `get_sync` | `put` | ضمان device active أثناء code block |
| `pm_runtime_active_auto` | `get_sync` | `put_autosuspend` | كـ السابق مع autosuspend عند الخروج |

#### الـ Conditional Guards

```c
DEFINE_GUARD_COND(pm_runtime_active, _try,
                  pm_runtime_get_active(_T, RPM_TRANSPARENT), _RET == 0)

DEFINE_GUARD_COND(pm_runtime_active, _try_enabled,
                  pm_runtime_resume_and_get(_T), _RET == 0)
```

| Variant | الفرق |
|---|---|
| `_try` | ينجح حتى لو runtime PM معطّل (`RPM_TRANSPARENT`) |
| `_try_enabled` | بيفشل لو runtime PM معطّل |

#### الـ ACQUIRE Macros

```c
#define PM_RUNTIME_ACQUIRE(_dev, _var)          ACQUIRE(pm_runtime_active_try, _var)(_dev)
#define PM_RUNTIME_ACQUIRE_AUTOSUSPEND(_dev, _var) ACQUIRE(pm_runtime_active_auto_try, _var)(_dev)
#define PM_RUNTIME_ACQUIRE_IF_ENABLED(_dev, _var)  ACQUIRE(pm_runtime_active_try_enabled, _var)(_dev)
#define PM_RUNTIME_ACQUIRE_ERR(_var_ptr)           ACQUIRE_ERR(pm_runtime_active, _var_ptr)
```

**Pattern استخدام:**

```c
/* مثال باستخدام PM_RUNTIME_ACQUIRE */
PM_RUNTIME_ACQUIRE(dev, ref);  /* try to get active reference */
if (PM_RUNTIME_ACQUIRE_ERR(&ref))
    return -ENODEV;            /* device not accessible */

/* device guaranteed active here */
do_work_with_device(dev);
/* ref auto-released at scope exit */
```

---

### الـ RPM Flags — تأثيرها على الـ Core Functions

```
              __pm_runtime_idle / __pm_runtime_suspend / __pm_runtime_resume
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ RPM_ASYNC:      queue work item in pm_wq → return immediately   │
│ RPM_NOWAIT:     return -EAGAIN if state change in progress      │
│ RPM_GET_PUT:    inc/dec usage_count as part of operation        │
│ RPM_AUTO:       use autosuspend_delay + hrtimer instead         │
│ RPM_TRANSPARENT: succeed (return 0) if runtime PM disabled      │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ `dev->power` fields الأكثر أهمية

| Field | Type | الوظيفة |
|---|---|---|
| `usage_count` | `atomic_t` | عدد المستخدمين الحاليين للـ device |
| `child_count` | `atomic_t` | عدد الـ children النشطين |
| `disable_depth` | `unsigned int:3` | مستوى التعطيل (0 = enabled) |
| `runtime_status` | `enum rpm_status` | الحالة الحالية (ACTIVE/SUSPENDED/...) |
| `last_status` | `enum rpm_status` | الحالة السابقة |
| `runtime_auto` | `bool` | اتسمحلها userspace suspend؟ |
| `use_autosuspend` | `bool` | autosuspend مفعّل؟ |
| `autosuspend_delay` | `int` | الـ delay بالـ ms |
| `last_busy` | `u64` | آخر timestamp للـ busy |
| `irq_safe` | `bool` | callbacks تشتغل من interrupt؟ |
| `no_callbacks` | `bool` | مفيش callbacks لـ runtime PM |
| `ignore_children` | `bool` | تجاهل child_count في suspend decision |
| `memalloc_noio` | `bool` | فرض GFP_NOIO في PM context |
| `needs_force_resume` | `bool` | محتاج force resume بعد system wake |
| `request` | `enum rpm_request` | الـ pending request في الـ pm_wq |
| `work` | `struct work_struct` | الـ work item للـ async operations |
| `suspend_timer` | `struct hrtimer` | timer للـ autosuspend |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بالـ Runtime PM

الـ kernel بيعمل expose لكل الـ runtime PM state من خلال `/sys/kernel/debug/` (وده بيتطلب `CONFIG_DEBUG_FS=y`).

```bash
# اعرض كل ملفات الـ runtime PM لـ device معين (مثال: USB device)
ls /sys/kernel/debug/usb/

# الـ runtime PM stats لأي device عام
cat /sys/kernel/debug/pm_genpd/PM_DOMAIN_NAME/summary
```

الـ **`/sys/kernel/debug/pm_genpd/`** بيديك:

| الملف | المحتوى |
|---|---|
| `summary` | قائمة بكل الـ PM domains وحالتها وعدد الـ devices |
| `<domain_name>/devices` | الـ devices المرتبطة بالـ domain دي |

```bash
# مثال عملي
cat /sys/kernel/debug/pm_genpd/summary
# Output:
# domain                          status          slaves
#                                 /device                                 runtime status
# -----------------------------------------------------------------------
# pd_usb                          on
#                                 /sys/devices/platform/usb0              active
```

---

#### 2. مدخلات الـ sysfs المتعلقة بالـ Runtime PM

كل device بيبقى عنده directory في sysfs فيه:

```
/sys/devices/<path-to-device>/power/
├── autosuspend_delay_ms     # قيمة الـ autosuspend_delay بالـ ms
├── control                  # "auto" أو "on"
├── runtime_active_kids      # عدد الـ children الـ active
├── runtime_active_time      # الوقت الكلي الـ device قضاه active (µs)
├── runtime_enabled          # "enabled" / "disabled" / "forbidden" / "unsupported"
├── runtime_status           # "active" / "suspended" / "suspending" / "resuming" / "error"
├── runtime_suspended_time   # الوقت الكلي الـ device قضاه suspended (µs)
├── runtime_usage            # قيمة الـ usage_count الحالية
└── wakeup                   # wakeup capability info
```

```bash
# راجع الـ runtime PM state لـ device معين
DEV=/sys/devices/platform/my-driver.0
cat $DEV/power/runtime_status
cat $DEV/power/runtime_usage
cat $DEV/power/runtime_active_time
cat $DEV/power/runtime_suspended_time
cat $DEV/power/autosuspend_delay_ms

# فرض الـ device يفضل active (disable autosuspend)
echo "on"   > $DEV/power/control

# اسمح للـ kernel يتحكم في الـ device
echo "auto" > $DEV/power/control
```

---

#### 3. الـ ftrace: Tracepoints وإزاي تشغلهم

الـ **runtime PM subsystem** عنده tracepoints جاهزة في `drivers/base/power/trace.c` وكمان تحت `include/trace/events/rpm.h`.

```bash
# شوف الـ tracepoints المتاحة
ls /sys/kernel/tracing/events/rpm/

# الـ events الأساسية:
# rpm_idle        → وقت استدعاء rpm_idle()
# rpm_suspend     → وقت بدء الـ runtime suspend
# rpm_resume      → وقت بدء الـ runtime resume
# rpm_return_int  → القيمة الراجعة من أي callback

# شغّل الـ tracing على runtime PM كله
echo 1 > /sys/kernel/tracing/events/rpm/enable
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل جهازك ولاحظ الـ events
cat /sys/kernel/tracing/trace

# وقف الـ tracing
echo 0 > /sys/kernel/tracing/tracing_on
echo 0 > /sys/kernel/tracing/events/rpm/enable
```

```bash
# لو عايز trace على device بعينها بس (فلترة)
echo 'name == "my-driver.0"' > /sys/kernel/tracing/events/rpm/rpm_suspend/filter
echo 1 > /sys/kernel/tracing/events/rpm/rpm_suspend/enable
```

مثال output مشروح:

```
# TASK-PID   CPU#  TIMESTAMP  FUNCTION
  kworker-42  [001]  123.456:  rpm_suspend: my-driver.0 state=1 flags=0x0
  kworker-42  [001]  123.458:  rpm_return_int: my-driver.0 function=runtime_suspend ret=0
  kworker-42  [001]  125.100:  rpm_resume: my-driver.0 state=3 flags=0x1
```

- `state=1` = `RPM_SUSPENDING`، `state=3` = `RPM_RESUMING`
- `flags=0x1` = `RPM_ASYNC`

---

#### 4. الـ printk والـ dynamic debug

```bash
# تفعيل dynamic debug لكل ملفات runtime PM core
echo "file drivers/base/power/runtime.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/power/generic_ops.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل لأي driver معين
echo "file drivers/mfd/my_chip.c +p" > /sys/kernel/debug/dynamic_debug/control

# شوف ايه الـ dynamic debug الـ active دلوقتي
cat /sys/kernel/debug/dynamic_debug/control | grep "runtime\|rpm"

# تفعيل من kernel command line (في /etc/default/grub)
# GRUB_CMDLINE_LINUX="dyndbg='file drivers/base/power/runtime.c +p'"
```

للـ **printk levels** في حالة boot مبكر:

```bash
# رفع الـ log level مؤقتاً
echo 8 > /proc/sys/kernel/printk

# أو من الـ bootloader
# console=ttyS0,115200 loglevel=8
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ Option | الوظيفة |
|---|---|
| `CONFIG_PM` | الأساس — الـ runtime PM نفسه |
| `CONFIG_PM_SLEEP` | دعم system sleep + `pm_runtime_force_suspend/resume` |
| `CONFIG_PM_SLEEP_DEBUG` | طباعة debug messages أثناء system sleep |
| `CONFIG_PM_DEBUG` | يفعّل `/sys/power/pm_test` وfile الـ pm_print_times |
| `CONFIG_PM_ADVANCED_DEBUG` | يضيف fields زيادة في sysfs (usage_count, child_count) |
| `CONFIG_DEBUG_FS` | لازم عشان `debugfs` |
| `CONFIG_PM_TRACE_RTC` | trace suspend/resume باستخدام RTC |
| `CONFIG_PM_GENERIC_DOMAINS_OF` | debugging لـ OF-based power domains |
| `CONFIG_PM_CLK` | دعم clock management في PM |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_PM|CONFIG_DEBUG"
# أو
grep -E "CONFIG_PM|CONFIG_DEBUG" /boot/config-$(uname -r)
```

---

#### 6. الـ devlink والـ tools المتخصصة

```bash
# استخدام udevadm لمراقبة PM events
udevadm monitor --kernel --property | grep -i "power\|PM"

# powertop: شوف مين بيمنع الـ suspend
sudo powertop --time=10

# cpupower لـ ACPI PM states
sudo cpupower -c all idle-info
sudo cpupower -c all frequency-info

# turbostat بيوري الـ C-states والـ power
sudo turbostat --interval 1

# pm-utils debugging
pm-suspend --debug 2>&1 | tee /tmp/pm-suspend.log
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|---|---|---|
| `Runtime PM sequence error` | الـ `usage_count` أو الـ state غلط | تأكد من كل `get` ليه `put` مقابله |
| `PM: Device X not suspended` | device ما اتعملتلهاش suspend أثناء system sleep | تأكد من `runtime_suspend` callback |
| `PM: Failed to suspend device X` | الـ `runtime_suspend` callback رجع error | اطبع الـ error code في الـ callback |
| `PM: dpm_run_callback(): X returned -EBUSY` | device مش قادر يتعمله suspend دلوقتي | child devices لسه active أو usage_count > 0 |
| `PM: Wakeup pending, aborting suspend` | جه wakeup event أثناء suspend | check الـ wakeup sources |
| `-EACCES` من `pm_runtime_suspend` | الـ runtime PM معطل للـ device | ضمن `pm_runtime_enable()` اتنادى |
| `-EAGAIN` من `pm_runtime_suspend` | `usage_count` مش صفر | مفيش استدعاء `put` كافي |
| `-EBUSY` من `pm_runtime_suspend` | `child_count` مش صفر | child devices لسه active |
| `-EPERM` من `pm_runtime_suspend` | QoS latency = 0 (لا يسمح بالـ suspend) | راجع PM QoS constraints |
| `PM: runtime suspend of X failed` | الـ suspend callback فشل | راجع الـ driver callback |
| `PM: can't remove a linked device` | device لينكها بـ supplier يمنعها | تأكد من ترتيب الـ cleanup |

---

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

```c
/* في runtime_suspend callback: تحقق من state سليم قبل suspend */
int my_runtime_suspend(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);

    /* تحقق إن الـ device مش busy */
    WARN_ON(atomic_read(&priv->pending_ops) != 0);

    if (!priv->initialized) {
        dev_err(dev, "suspend called on uninit device\n");
        dump_stack(); /* اطبع call stack كامل */
        return -EINVAL;
    }
    return 0;
}

/* في runtime_resume callback */
int my_runtime_resume(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);

    /* تحقق مش بنعمل double resume */
    WARN_ON(pm_runtime_active(dev) && !pm_runtime_suspended(dev));
    return 0;
}

/* في الـ probe: تحقق من ترتيب الـ setup */
static int my_probe(struct platform_device *pdev)
{
    /* ... */
    pm_runtime_set_active(&pdev->dev);
    pm_runtime_enable(&pdev->dev);

    /* تحقق إن كل شيء اتضبط صح */
    WARN_ON(!pm_runtime_enabled(&pdev->dev));
    return 0;
}
```

---

### Hardware Level

---

#### 1. التحقق إن الـ hardware state بيطابق الـ kernel state

**الـ runtime_status** في sysfs المفروض يطابق الـ hardware power state الفعلي:

```bash
# راجع الـ kernel state
cat /sys/devices/platform/my-device/power/runtime_status
# المتوقع: "suspended" لو الـ device مش شغال

# تحقق من الـ hardware
# لو device على UART:
# راقب الـ TX/RX lines بـ logic analyzer

# لو device على I2C:
# جرب تعمل probe وشوف لو بيرد
i2cdetect -y 1

# لو device على SPI:
# جرب قراءة register ID
```

مقارنة الـ state:

```
kernel state: "suspended"  → hardware: no power / clock disabled
kernel state: "active"     → hardware: powered / clock enabled / responding
```

---

#### 2. تقنيات الـ Register Dump

```bash
# devmem2 لقراءة registers مباشرة (لو الـ address معروف)
# قراءة register بـ physical address 0xFE200000
devmem2 0xFE200000 w   # قراءة 32-bit word
devmem2 0xFE200004 w

# أو عن طريق /dev/mem
dd if=/dev/mem bs=4 count=1 skip=$((0xFE200000/4)) 2>/dev/null | xxd

# لو عندك MMIO mapped:
# في الكود
void __iomem *base = ioremap(PHYS_ADDR, SIZE);
dev_dbg(dev, "REG0=0x%08x REG1=0x%08x\n",
        readl(base + 0x00), readl(base + 0x04));
iounmap(base);

# لو device بـ debugfs يعمل expose للـ registers:
cat /sys/kernel/debug/regmap/<device-name>/registers
# أو
cat /sys/kernel/debug/regmap/<device-name>/range
```

لو بتستخدم **regmap**:

```bash
# regmap بيعمل dump تلقائي في debugfs
ls /sys/kernel/debug/regmap/
cat /sys/kernel/debug/regmap/1-0050/registers
# Output: offset=0x00 value=0xABCD1234
```

---

#### 3. نصائح Logic Analyzer والـ Oscilloscope

```
Signal                  | ما تراقبه
------------------------|-----------------------------------------
POWER_EN (GPIO)         | يتبع rpm_status: LOW=suspended, HIGH=active
CLK line                | يجب إنه يوقف أثناء suspend
IRQ/WAKE line           | يجب إنه يصحّى الـ device من suspended
SDA/SCL (I2C)           | ما في transactions أثناء suspended
CS# (SPI)               | HIGH (inactive) أثناء suspended
```

**نصائح عملية:**

- **Trigger** على GPIO الـ POWER_EN: ابدأ الـ capture لما يتغير
- قس الـ latency بين rising edge الـ POWER_EN وأول I2C transaction — الـ `autosuspend_delay` المتوقع
- لو الـ CS# بيكون LOW أثناء suspended: bug في الـ driver، الـ `runtime_suspend` مش بتوقف الـ SPI
- استخدم **sigrok / PulseView** مع logic analyzer رخيص (Saleae / fx2lafw)

```bash
# تسجيل GPIO changes أثناء runtime PM transitions
# لو الـ GPIO مكشوف في sysfs
watch -n 0.1 cat /sys/class/gpio/gpio123/value
```

---

#### 4. مشاكل الـ hardware الشائعة وأنماط الـ log

| المشكلة الـ hardware | نمط الـ kernel log | السبب المحتمل |
|---|---|---|
| الـ device مش بيرجع بعد suspend | `PM: Device X failed to resume` + timeout | الـ clock مش اتشغل قبل الـ register access |
| الـ device بيرفض الـ suspend | `runtime suspend of X returned -EIO` | الـ hardware لسه busy / FIFO مش فاضي |
| الـ interrupt مش بيوصل | device فاضل suspended، ما في `rpm_resume` | الـ IRQ handler مش بيعمل `pm_runtime_get` |
| random crashes بعد resume | `BUG: unable to handle kernel paging` | الـ regmap access قبل ما الـ clock يتشغل |
| الـ device بيصحى لوحده | `PM: Wakeup event` متكرر | spurious interrupt أو floating IRQ line |
| power consumption عالي | `runtime_status` = active دايماً | `pm_runtime_put` ناقص في الـ driver |

```bash
# راقب الـ wakeup sources
cat /sys/kernel/debug/wakeup_sources
# أو
cat /proc/sys/kernel/wakeup_count
```

---

#### 5. الـ Device Tree Debugging

```bash
# تحقق من الـ DT compiled
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A10 "my-device"

# تحقق من الـ power-domains في الـ DT
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -B2 -A5 "power-domains"

# قارن مع الـ hardware datasheet:
# الـ power domain reference في الـ DT لازم يطابق الـ SoC TRM
```

مثال DT صح:

```dts
my_device: device@fe200000 {
    compatible = "vendor,my-device";
    reg = <0xfe200000 0x1000>;
    clocks = <&cru CLK_MY_DEVICE>;  /* clock لازم يكون صح */
    power-domains = <&power PD_MY_DEVICE>;  /* PD ID لازم يطابق SoC TRM */
    status = "okay";
};
```

```bash
# تحقق إن الـ power domain اتسجل صح
cat /sys/kernel/debug/pm_genpd/summary | grep my-device

# تحقق من الـ clock
cat /sys/kernel/debug/clk/clk_summary | grep my_device_clk

# لو الـ clock مش شغال:
# runtime_resume هيفشل بـ -ENOENT أو -EPERM
```

---

### Practical Commands

---

#### 1. Script شامل لـ diagnosis سريع

```bash
#!/bin/bash
# runtime_pm_debug.sh - شوف حالة الـ runtime PM لكل الـ devices

echo "=== Runtime PM Status for All Devices ==="
for dev in /sys/devices/*/power /sys/devices/*/*/power /sys/devices/*/*/*/power; do
    [ -f "$dev/runtime_status" ] || continue
    status=$(cat "$dev/runtime_status" 2>/dev/null)
    usage=$(cat "$dev/runtime_usage" 2>/dev/null)
    enabled=$(cat "$dev/runtime_enabled" 2>/dev/null)
    devpath=$(dirname "$dev")
    echo "Device: $(basename $devpath)"
    echo "  Path:    $devpath"
    echo "  Status:  $status"
    echo "  Usage:   $usage"
    echo "  Enabled: $enabled"
    echo ""
done
```

```bash
# تشغيل السكريبت
chmod +x runtime_pm_debug.sh
sudo ./runtime_pm_debug.sh 2>/dev/null | head -100
```

---

#### 2. مراقبة الـ autosuspend

```bash
DEV=/sys/devices/platform/my-driver.0

# شوف الـ delay الحالي
cat $DEV/power/autosuspend_delay_ms

# غير الـ delay (مثال: 5 ثواني)
echo 5000 > $DEV/power/autosuspend_delay_ms

# شوف الـ last_busy timestamp (ns من boot)
# مفيش sysfs مباشر — بس الـ active_time بيدلك
cat $DEV/power/runtime_active_time
cat $DEV/power/runtime_suspended_time

# احسب نسبة الـ suspension
active=$(cat $DEV/power/runtime_active_time)
suspended=$(cat $DEV/power/runtime_suspended_time)
echo "Active: ${active}us, Suspended: ${suspended}us"
```

---

#### 3. فرض suspend/resume يدوي للـ testing

```bash
DEV=/sys/devices/platform/my-driver.0

# فرض الـ kernel يتحكم (auto mode)
echo "auto" > $DEV/power/control

# فرض الـ device يفضل on (امنع الـ suspend)
echo "on" > $DEV/power/control

# راقب التغيير
watch -n 1 "cat $DEV/power/runtime_status $DEV/power/runtime_usage"
```

---

#### 4. تتبع الـ usage_count leaks

```bash
# شوف usage_count لكل الـ devices
for dev in /sys/devices/*/power /sys/devices/*/*/power; do
    [ -f "$dev/runtime_usage" ] || continue
    usage=$(cat "$dev/runtime_usage" 2>/dev/null)
    [ "$usage" -gt "1" ] 2>/dev/null && \
        echo "LEAK? $(dirname $dev): usage=$usage"
done

# الـ output مثال:
# LEAK? /sys/devices/platform/my-device.0: usage=3
# ده معناه في 3 pm_runtime_get بدون put مقابل
```

---

#### 5. تفعيل runtime PM tracing مع فلترة device معين

```bash
DEV_NAME="my-driver.0"

# فعّل الـ tracing
echo 0 > /sys/kernel/tracing/tracing_on
echo "" > /sys/kernel/tracing/trace
echo "function_graph" > /sys/kernel/tracing/current_tracer

# اضبط filter على الـ pm functions
echo "rpm_*" > /sys/kernel/tracing/set_ftrace_filter

# فعّل events
echo 1 > /sys/kernel/tracing/events/rpm/enable
echo 1 > /sys/kernel/tracing/tracing_on

# اعمل الـ action اللي عايزه (مثلاً echo auto للـ device)
echo "auto" > /sys/devices/platform/${DEV_NAME}/power/control

# شوف النتايج
cat /sys/kernel/tracing/trace | grep -i "$DEV_NAME\|rpm_"

# مثال output:
# kworker/u8:2-345  [002] 1234.567890: rpm_suspend: my-driver.0 state=1 flags=0x0
# kworker/u8:2-345  [002] 1234.568001: rpm_return_int: my-driver.0 function=runtime_suspend ret=0
```

---

#### 6. kernel panic debugging لمشاكل الـ PM

```bash
# لو الـ system بيـcrash أثناء runtime PM:
# أضف للـ kernel cmdline:
# no_console_suspend earlyprintk=ttyS0,115200 ignore_loglevel

# أو فعّل kdump
systemctl enable kdump
echo 1 > /proc/sys/kernel/sysrq
echo c > /proc/sysrq-trigger  # crash يدوي للـ test

# بعد الـ kdump: حلل الـ crash
crash /usr/lib/debug/boot/vmlinux-$(uname -r) /var/crash/*/vmcore
# في crash shell:
# bt        → backtrace
# ps        → كل الـ processes
# log       → kernel ring buffer
# struct device.power <addr>  → شوف dev_pm_info
```

---

#### 7. التحقق من صحة الـ WARN_ON والـ BUG_ON في سياق الـ PM

```bash
# شوف الـ kernel warning messages المتعلقة بالـ PM
dmesg | grep -i "WARNING\|BUG\|WARN\|runtime\|RPM"

# مثال output يشوش:
# WARNING: CPU: 0 PID: 42 at drivers/base/power/runtime.c:1234 rpm_suspend+0x100
# هنا لازم تراجع السطر 1234 في runtime.c

# فعّل CONFIG_PM_ADVANCED_DEBUG وشوف:
dmesg | grep "runtime_status\|usage_count\|child_count"
```

---

#### 8. مقارنة سريعة: الـ runtime_status في الـ kernel vs الـ sysfs

```
kernel (enum rpm_status)     sysfs (runtime_status)
----------------------------  -----------------------
RPM_ACTIVE        (0)         "active"
RPM_RESUMING      (2)         "resuming"
RPM_SUSPENDED     (3)         "suspended"  ← pm_runtime_suspended() returns true هنا
RPM_SUSPENDING    (1)         "suspending"
RPM_BLOCKED       (4)         "blocked"
```

```bash
# تحقق سريع من كل device عندها error
for dev in /sys/devices/*/power; do
    err=$(cat "$dev/runtime_status" 2>/dev/null)
    [ "$err" = "error" ] && echo "ERROR: $dev"
done
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ UART بيبقى suspended وسط التشغيل

#### العنوان
**الـ UART بيفشل في إرسال بيانات Modbus في gateway صناعي**

#### السياق
شركة بتبني industrial gateway بـ RK3562 لجمع بيانات من حساسات Modbus RTU عبر UART. المنتج بيشتغل 24/7 في مصنع. بعد ساعات من التشغيل، الـ gateway بيبطل يستقبل بيانات فجأة من غير أي reboot أو error واضح في الـ kernel log.

#### المشكلة
الـ UART driver بيعمل `pm_runtime_put_autosuspend()` بعد كل transaction، والـ autosuspend delay معموله 2 ثانية. في حالة عادية كويس. لكن الـ Modbus master بياخد أحياناً أكتر من 2 ثانية بين الـ frames، فالـ UART بيدخل `RPM_SUSPENDED` وبيتقطع الـ clock قبل ما يجي الـ frame الجديد. الـ driver بياخد الـ byte الأول وهو بيعمل `pm_runtime_resume_and_get()` — بس ده بيحصل async، والـ byte اتأكل.

#### التحليل

```c
/* في الـ UART driver — المشكلة هنا */
static irqreturn_t uart_irq_handler(int irq, void *dev_id)
{
    struct uart_port *port = dev_id;

    /*
     * pm_runtime_resume_and_get() calls __pm_runtime_resume(dev, RPM_GET_PUT)
     * هذا synchronous لكن الـ resume callback محتاج وقت لتشغيل الـ clock
     * في هذا الوقت يضيع أول byte
     */
    if (pm_runtime_resume_and_get(port->dev) < 0)
        return IRQ_NONE;

    /* بيجي هنا بعد ما الـ byte اتأكل */
    handle_rx(port);
    pm_runtime_mark_last_busy(port->dev);
    pm_runtime_put_autosuspend(port->dev);
    return IRQ_HANDLED;
}
```

**تتبع المسار:**

1. `pm_runtime_put_autosuspend()` → `pm_runtime_mark_last_busy()` + `__pm_runtime_suspend(dev, RPM_GET_PUT | RPM_ASYNC | RPM_AUTO)`
2. الـ `RPM_AUTO` بيخلي الـ framework يحسب `pm_runtime_autosuspend_expiration()` — لو الوقت فات → suspend
3. الـ `runtime_status` بيبقى `RPM_SUSPENDED` و`disable_depth == 0`
4. `pm_runtime_suspended(dev)` ترجع `true`
5. جه interrupt → `pm_runtime_resume_and_get()` → `pm_runtime_get_active(dev, 0)` → `__pm_runtime_resume(dev, RPM_GET_PUT)` — **synchronous** بس الـ resume callback شغلة كثيرة (enable clock, wait for PLL lock)

#### الحل

```c
/* الحل 1: رفع الـ autosuspend delay */
pm_runtime_set_autosuspend_delay(&pdev->dev, 10000); /* 10 ثانية */

/* الحل 2: استخدام pm_runtime_irq_safe() لو الـ resume خفيف */
pm_runtime_irq_safe(&pdev->dev); /* يتيح resume من interrupt context */

/* الحل 3: تعطيل autosuspend لـ UART في gateway mode */
pm_runtime_dont_use_autosuspend(&pdev->dev);

/* الحل 4 (الأفضل): رفع الـ usage_count أثناء الـ port مفتوح */
static int uart_open(struct tty_struct *tty, struct file *filp)
{
    struct uart_state *state = tty->driver_data;
    /* يمنع الـ suspend طول ما الـ port مفتوح */
    pm_runtime_get_sync(state->uart_port->dev);
    return 0;
}

static void uart_close(struct tty_struct *tty, struct file *filp)
{
    struct uart_state *state = tty->driver_data;
    pm_runtime_put_sync(state->uart_port->dev);
}
```

#### الدرس المستفاد
**الـ `pm_runtime_irq_safe()` مش بديل عن التصميم الصح.** في devices بيتكلم معاها 24/7 زي UART في industrial، الصح إنك ترفع الـ `usage_count` طول ما الـ file descriptor مفتوح، مش تعتمد على autosuspend بين كل frame وفريم.

---

### السيناريو 2: Android TV Box على Allwinner H616 — freeze في suspend بسبب USB

#### العنوان
**الـ system suspend بيتعلق بسبب USB device link مش متسوى صح**

#### السياق
TV box بـ Allwinner H616 بيشغل Android 12. المستخدم بيضغط "sleep" من الريموت. الشاشة بتتقفل بس الـ system بيفضل يسحب تيار عادي — يعني الـ suspend مش بيكمل. الـ dmesg بيوري إن الـ system واقف في `dpm_suspend`.

#### المشكلة
الـ USB hub مربوط بـ device_link على الـ USB controller، لكن الـ hub driver ما عملش `pm_runtime_get_suppliers()` صح. الـ controller بيعمل suspend قبل الـ hub، فالـ hub يلاقي الـ supplier suspended ويعمل `-EACCES` في `runtime_suspend` callback.

#### التحليل

```c
/*
 * pm_runtime_get_suppliers() بتعمل resume لكل supplier links
 * وترفع usage_count عندهم
 * لو ما اتعملتش → الـ supplier (USB controller) ممكن يتعلق
 */
extern void pm_runtime_get_suppliers(struct device *dev);
extern void pm_runtime_put_suppliers(struct device *dev);

/*
 * الـ pm_runtime_force_suspend() اللي بيتسمى أثناء system suspend
 * بيشيل الـ dev من suppliers قبل suspend callback
 */
extern int pm_runtime_force_suspend(struct device *dev);
```

**تتبع المسار:**

```
dpm_suspend
  └─> pm_runtime_force_suspend(hub_dev)
        └─> pm_runtime_get_suppliers(hub_dev)  ← بيحاول يرفع suppliers
              └─> __pm_runtime_resume(controller_dev, ...)
                    └─> runtime_status == RPM_SUSPENDING  ← controller بيتعلق!
                          └─> return -EAGAIN → deadlock
```

الـ `pm_runtime_barrier()` بيستنى كل pending operations تخلص — لكن هنا الـ supplier في `RPM_SUSPENDING` والـ consumer بيحاول يعمله resume في نفس الوقت.

#### الحل

```c
/* في probe الـ USB hub driver */
static int usb_hub_probe(struct usb_interface *intf,
                         const struct usb_device_id *id)
{
    struct device *dev = &intf->dev;
    struct device *controller = /* الـ USB controller */;

    /* إنشاء device_link صريح بـ DL_FLAG_PM_RUNTIME */
    struct device_link *link = device_link_add(dev, controller,
        DL_FLAG_STATELESS | DL_FLAG_PM_RUNTIME | DL_FLAG_RPM_ACTIVE);
    if (!link)
        return -ENOMEM;

    /*
     * pm_runtime_new_link() بيزود links_count
     * الـ framework هيعرف إن الـ hub محتاج الـ controller يبقى active
     */
    pm_runtime_enable(dev);
    return 0;
}

/* في remove */
static void usb_hub_disconnect(struct usb_interface *intf)
{
    device_link_del(link);
    pm_runtime_drop_link(link); /* بيطبق pm_runtime_release_supplier() */
}
```

```bash
# تشخيص سريع
cat /sys/bus/usb/devices/usb1/power/runtime_status
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
# شوف links
ls /sys/bus/usb/devices/usb1/power/
```

#### الدرس المستفاد
**الـ `device_link` مع `DL_FLAG_PM_RUNTIME` مش اختياري في consumer/supplier chains.** الـ framework بيعتمد على `links_count` و`pm_runtime_get_suppliers()` لضمان الترتيب الصح في suspend. غيابه بيعمل race condition مؤلمة جداً.

---

### السيناريو 3: IoT Sensor على STM32MP1 — الـ I2C بيرجع -EACCES في interrupt

#### العنوان
**قراءة sensor من ISR بتفشل بـ `-EACCES` على STM32MP1**

#### السياق
IoT device بـ STM32MP1 بيقيس ضغط وحرارة من sensor I2C. الـ sensor بيعمل interrupt لما تيجي بيانات جديدة. الـ ISR بيحاول يقرأ الـ I2C مباشرة. شغال كويس في بداية التطوير، لكن بعد تفعيل runtime PM للـ I2C controller بدأ يرجع `-EACCES`.

#### المشكلة
الـ I2C controller دخل `RPM_SUSPENDED`. لما الـ ISR شغل وحاول يعمل `pm_runtime_resume_and_get()` — الـ standard resume بيحاول يأخذ spinlock ويعمل schedule. ده مش ممكن من interrupt context. فالـ framework رجع `-EACCES`.

```c
/*
 * من pm_runtime_active():
 * ترجع true لو runtime_status == RPM_ACTIVE أو disable_depth > 0
 * لو RPM_SUSPENDED ومش irq_safe → resume من IRQ مستحيل
 */
static inline bool pm_runtime_active(struct device *dev)
{
    return dev->power.runtime_status == RPM_ACTIVE
        || dev->power.disable_depth;
}

/*
 * pm_runtime_is_irq_safe():
 * بترجع dev->power.irq_safe
 * لازم تتسطب عشان resume يشتغل من IRQ context
 */
static inline bool pm_runtime_is_irq_safe(struct device *dev)
{
    return dev->power.irq_safe;
}
```

#### التحليل

```
ISR fires
  └─> pm_runtime_resume_and_get(i2c_dev)
        └─> pm_runtime_get_active(i2c_dev, 0)
              └─> __pm_runtime_resume(i2c_dev, RPM_GET_PUT)
                    └─> rpm_resume()  ← kernel/power/runtime.c
                          └─> !dev->power.irq_safe → can't schedule
                                └─> return -EACCES
```

الـ `pm_runtime_irq_safe()` لازم تتعمل في `probe()` عشان تسطب `dev->power.irq_safe = 1` ويشتغل الـ resume من IRQ context بدون scheduling.

#### الحل

```c
static int stm32_i2c_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    int ret;

    /* ضروري قبل enable عشان resume يشتغل من IRQ */
    pm_runtime_irq_safe(dev);

    pm_runtime_set_autosuspend_delay(dev, 50); /* 50ms */
    pm_runtime_use_autosuspend(dev);

    ret = devm_pm_runtime_enable(dev);
    if (ret)
        return ret;

    return 0;
}

/* في الـ ISR */
static irqreturn_t sensor_irq(int irq, void *data)
{
    struct sensor_dev *sensor = data;
    struct device *i2c_dev = sensor->i2c_client->dev.parent;

    /* الآن يشتغل من IRQ لأن irq_safe = true */
    if (pm_runtime_resume_and_get(i2c_dev) < 0)
        return IRQ_NONE;

    read_sensor_data(sensor);

    pm_runtime_mark_last_busy(i2c_dev);
    pm_runtime_put_autosuspend(i2c_dev);
    return IRQ_HANDLED;
}
```

**تحذير مهم:** `pm_runtime_irq_safe()` بتعني إن الـ suspend/resume callbacks لازم تشتغل من spinlock context — مش ممكن يعملوا `msleep()` أو أي blocking.

```bash
# تأكد إن irq_safe متسطب
cat /sys/bus/i2c/devices/i2c-0/power/runtime_status
# في debugfs
cat /sys/kernel/debug/pm_genpd/stm32mp1-i2c/summary
```

#### الدرس المستفاد
**`pm_runtime_irq_safe()` مش مجرد hint — هي بتغير سلوك الـ framework كله.** لازم الـ suspend/resume callbacks تتكتب بحيث تشتغل من atomic context. أي `msleep` أو `mutex_lock` جوا الـ callback هيعمل kernel panic.

---

### السيناريو 4: Automotive ECU على i.MX8 — تسريب في الـ usage_count

#### العنوان
**الـ SPI controller مش بيدخل suspend أبداً على i.MX8 في ECU للسيارة**

#### السياق
ECU (Electronic Control Unit) بـ i.MX8 بيتحكم في sensors الفرامل. المشروع محتاج يوفر استهلاك طاقة في idle. الـ engineer لاقي إن الـ SPI controller مش بيعمل suspend رغم إنه مش بيتكلم مع أي device.

#### المشكلة
الـ `usage_count` فضل عند 1 بدل ما يرجع لصفر. جاء من إن driver قديم استخدم `pm_runtime_get_sync()` في error path بس نسي يعمل `pm_runtime_put()` لما الـ operation فشلت.

```c
/*
 * pm_runtime_get_sync() → __pm_runtime_resume(dev, RPM_GET_PUT)
 * بتزود usage_count حتى لو رجعت error
 *
 * من documentation الـ header:
 * "the runtime PM usage counter of @dev remains incremented in all cases,
 *  even if it returns an error code."
 *
 * بالمقابل pm_runtime_resume_and_get() بتعمل put لو فشلت:
 */
static inline int pm_runtime_get_active(struct device *dev, int rpmflags)
{
    int ret;
    ret = __pm_runtime_resume(dev, RPM_GET_PUT | rpmflags);
    if (ret < 0) {
        pm_runtime_put_noidle(dev); /* بترد الـ count لو فشل */
        return ret;
    }
    return 0;
}
```

#### التحليل

```c
/* الكود الخاطئ في SPI driver */
static int spi_transfer(struct spi_device *spi, struct spi_message *msg)
{
    struct spi_controller *ctlr = spi->controller;
    int ret;

    /* usage_count++ حتى لو رجع error */
    ret = pm_runtime_get_sync(ctlr->dev.parent);
    if (ret < 0) {
        /* BUG: مفيش put هنا! usage_count فضل مزوَّد */
        return ret;
    }

    ret = do_spi_transfer(ctlr, msg);

    pm_runtime_mark_last_busy(ctlr->dev.parent);
    pm_runtime_put_autosuspend(ctlr->dev.parent);
    return ret;
}
```

**التشخيص:**

```bash
# شوف الـ usage_count الحالي
cat /sys/bus/spi/devices/spi0/power/runtime_usage
# لازم يكون 0 في idle، لو 1+ في مشكلة

# شوف الـ status
cat /sys/bus/spi/devices/spi0/power/runtime_status
# المتوقع: suspended — لو active وانت مش بتستخدمه = leak
```

#### الحل

```c
/* الكود الصح */
static int spi_transfer(struct spi_device *spi, struct spi_message *msg)
{
    struct spi_controller *ctlr = spi->controller;
    int ret;

    /*
     * pm_runtime_resume_and_get() = pm_runtime_get_active(dev, 0)
     * بتعمل put تلقائياً لو resume فشل → مفيش leak
     */
    ret = pm_runtime_resume_and_get(ctlr->dev.parent);
    if (ret < 0)
        return ret;

    ret = do_spi_transfer(ctlr, msg);

    pm_runtime_mark_last_busy(ctlr->dev.parent);
    pm_runtime_put_autosuspend(ctlr->dev.parent);
    return ret;
}
```

**أو استخدام الـ guard الجديد:**

```c
static int spi_transfer(struct spi_device *spi, struct spi_message *msg)
{
    struct spi_controller *ctlr = spi->controller;
    /* DEFINE_GUARD(pm_runtime_active, ...) هيعمل get_sync في البداية
       وput تلقائياً لما يخرج من الـ scope */
    guard(pm_runtime_active)(ctlr->dev.parent);

    return do_spi_transfer(ctlr, spi->cur_msg);
}
```

#### الدرس المستفاد
**`pm_runtime_get_sync()` deprecated في error paths.** استخدم دايماً `pm_runtime_resume_and_get()` لأنها تعمل `put` تلقائياً لو الـ resume فشل. أو استخدم `DEFINE_GUARD(pm_runtime_active, ...)` عشان الـ compiler يضمنلك الـ balance تلقائياً.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ HDMI مش بيشتغل بعد suspend/resume

#### العنوان
**الشاشة بتتقفل تماماً بعد أول system suspend/resume على AM62x**

#### السياق
فريق bring-up بيشغل custom display board بـ Texas Instruments AM62x. الـ HDMI شغال كويس من أول boot. بعد أول `echo mem > /sys/power/state` والـ resume، الشاشة ميجيش فيها أي صورة. لا kernel panic، لا error — الـ system شاغل بس HDMI صامت.

#### المشكلة
الـ HDMI driver استخدم `pm_runtime_set_active()` في `probe()` من غير ما يعمل `pm_runtime_enable()` الأول. ده خلى الـ `disable_depth` يفضل `> 0`. في الـ system resume، الـ `pm_runtime_force_resume()` بيشيك على `pm_runtime_need_not_resume()` — ولقى إن الـ device "active before suspend" فما عملش resume حقيقي.

```c
/*
 * pm_runtime_set_active() → __pm_runtime_set_status(dev, RPM_ACTIVE)
 * صح بس لازم تجي قبل pm_runtime_enable()
 * لو ما اتعملتش enable → disable_depth يفضل > 0
 *
 * pm_runtime_active() بترجع true لو disable_depth > 0
 * حتى لو الـ HW فعلاً suspended!
 */
static inline bool pm_runtime_active(struct device *dev)
{
    return dev->power.runtime_status == RPM_ACTIVE
        || dev->power.disable_depth; /* ← هنا المشكلة */
}

/*
 * pm_runtime_enabled() بترجع false لو disable_depth > 0
 * يعني الـ runtime PM مش شغال أصلاً
 */
static inline bool pm_runtime_enabled(struct device *dev)
{
    return !dev->power.disable_depth;
}
```

#### التحليل

```
HDMI probe()
  └─> pm_runtime_set_active(dev)   ← status = RPM_ACTIVE, enable لسه ما اتعملش
  └─> /* نسي pm_runtime_enable() */

system suspend
  └─> pm_runtime_force_suspend(hdmi_dev)
        └─> disable_depth > 0 → skip actual suspend → runtime_status unchanged

system resume
  └─> pm_runtime_need_not_resume(hdmi_dev)
        └─> بيشيك: هل كان suspended قبل system sleep؟
              └─> last_status != RPM_SUSPENDED → ترجع true = "no need to resume"
  └─> pm_runtime_force_resume() ما اتعملش!
  └─> HDMI hardware فعلاً powerless بس الـ driver فاكره active
```

```bash
# تشخيص
cat /sys/bus/platform/devices/hdmi/power/runtime_enabled
# لو "disabled" → disable_depth > 0 → المشكلة هنا

cat /sys/bus/platform/devices/hdmi/power/runtime_status
# لو "active" وانت عارف إن الـ HW مش شغال → fake active
```

#### الحل

```c
static int hdmi_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    int ret;

    /* الترتيب الصح */

    /* 1. سطب الـ status قبل enable */
    ret = pm_runtime_set_active(dev);
    if (ret) {
        dev_err(dev, "failed to set active: %d\n", ret);
        return ret;
    }

    /* 2. enable runtime PM — بيخلي disable_depth = 0 */
    ret = devm_pm_runtime_enable(dev);
    if (ret)
        return ret;

    /*
     * devm_pm_runtime_enable() أفضل من pm_runtime_enable()
     * لأنه بيعمل disable تلقائياً لو probe فشل أو driver اتشال
     * بدون leak في disable_depth
     */

    /* 3. تأكد إنه enabled */
    if (!pm_runtime_enabled(dev)) {
        dev_err(dev, "runtime PM not enabled!\n");
        return -EINVAL;
    }

    return hdmi_hw_init(pdev);
}
```

**بديل أكثر أمان باستخدام `devm_pm_runtime_set_active_enabled()`:**

```c
static int hdmi_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    int ret;

    /*
     * devm_pm_runtime_set_active_enabled() = set_active + enable في خطوة واحدة
     * وبيعمل undo تلقائياً لو probe فشل
     */
    ret = devm_pm_runtime_set_active_enabled(dev);
    if (ret)
        return ret;

    return hdmi_hw_init(pdev);
}
```

```bash
# بعد الإصلاح، تحقق من دورة suspend/resume
echo mem > /sys/power/state
# انتظر resume
dmesg | grep -i hdmi | tail -20
cat /sys/bus/platform/devices/hdmi/power/runtime_status
# المتوقع: active
```

#### الدرس المستفاد
**الترتيب في `probe()` حرج جداً: `pm_runtime_set_active()` لازم تجي قبل `pm_runtime_enable()`.** استخدم `devm_pm_runtime_set_active_enabled()` أو `devm_pm_runtime_enable()` عشان الـ cleanup يبقى automatic ومتضمنش memory leak في `disable_depth`. وبعد أي تغيير في PM initialization، اختبر دايماً دورة كاملة suspend/resume مش بس الـ boot.
## Phase 7: مصادر ومراجع

---

### الـ Official Kernel Documentation

الـ documentation الرسمي للـ runtime PM موجود في الـ kernel source مباشرةً:

| الملف | المحتوى |
|---|---|
| `Documentation/power/runtime_pm.rst` | الـ framework الكامل، الـ callbacks، الـ states، الـ locking |
| `Documentation/power/admin-guide/pm/cpuidle.rst` | العلاقة بين الـ runtime PM والـ CPU idle |
| `Documentation/driver-api/pm/devices.rst` | دليل كتابة الـ drivers مع الـ PM |
| `include/linux/pm_runtime.h` | الـ header الأساسي — كل الـ API موجود هنا |
| `drivers/base/power/runtime.c` | التنفيذ الفعلي للـ framework |
| `drivers/base/power/main.c` | الـ system sleep integration |

الـ online version للـ documentation الرسمي:

- [Runtime Power Management Framework for I/O Devices — kernel.org docs](https://static.lwn.net/kerneldoc/power/runtime_pm.html)
- [Runtime Power Management Framework — kernel 5.19 docs](https://docs.kernel.org/5.19/power/runtime_pm.html)
- [Runtime Power Management Framework — infradead mirror](https://www.infradead.org/~mchehab/kernel_docs/power/runtime_pm.html)

---

### الـ LWN.net Articles

**أهم المقالات — مرتبة من الأقدم للأحدث:**

#### 1. النشأة والـ Patch الأصلي

- [Run-time PM idea — المقترح الأول (2008)](https://lwn.net/Articles/336729/)
  - النقاش الأولي لفكرة الـ runtime PM قبل التنفيذ.

- [PM: Introduce core framework for run-time PM of I/O devices](https://lwn.net/Articles/337538/)
  - الـ patch الأول الذي قدّم الـ framework — التعليقات تكشف قرارات التصميم.

- [PM: Introduce core framework — rev. 8](https://lwn.net/Articles/340047/)
  - نسخة متطورة من الـ patch مع ردود المجتمع.

- [PM: Introduce core framework — rev. 17 (النسخة النهائية)](https://lwn.net/Articles/347574/)
  - النسخة التي دخلت الـ kernel في 2.6.32.

#### 2. الـ Documentation والـ Overview

- [Documentation/power/runtime_pm.txt](https://lwn.net/Articles/347575/)
  - الـ documentation الذي أُرسل مع الـ patch الأصلي.

- [Runtime power management — مقالة LWN الشارحة](https://lwn.net/Articles/347573/)
  - شرح مبسط للـ framework وكيفية استخدامه في الـ drivers.

- [Linux power management: The documentation I wanted to read](https://lwn.net/Articles/505683/)
  - مقالة عام 2012 — تشرح الـ usage_count، الـ autosuspend timer، وعلاقة الـ parent/child.

#### 3. الـ Autosuspend والـ Integration

- [USB autosuspend](https://lwn.net/Articles/373550/)
  - كيف يستخدم الـ USB subsystem الـ autosuspend — مثال عملي حقيقي.

- [PCI: Runtime power management](https://lwn.net/Articles/346898/)
  - تطبيق الـ runtime PM على الـ PCI devices.

- [runtime PM support for I2C and SPI client devices](https://lwn.net/Articles/566234/)
  - توسعة الـ framework ليشمل الـ I2C والـ SPI.

- [PM: Using runtime PM callbacks for system suspend/resume](https://lwn.net/Articles/733130/)
  - كيف تُوظّف الـ `pm_runtime_force_suspend/resume` في الـ system sleep.

- [PM: Reconcile different driver options for runtime PM integration with system sleep](https://lwn.net/Articles/1026995/)
  - أحدث مقالة (2024) — توحيد الـ API بين الـ runtime PM والـ system sleep.

- [PM / Runtime: Generic clock manipulation routines for runtime PM](https://lwn.net/Articles/440445/)
  - إضافة الـ clock gating الـ generic للـ runtime PM.

- [vfio/pci: Enable runtime power management support](https://lwn.net/Articles/882561/)
  - مثال حديث على إضافة الـ runtime PM لـ VFIO/PCI.

---

### الـ Mailing List والـ Lore

الـ linux-pm mailing list هو المكان الرئيسي لنقاشات الـ runtime PM:

- [lore.kernel.org — linux-pm](https://lore.kernel.org/linux-pm/)
  - البحث بـ `pm_runtime` يعطي مئات الـ patches والنقاشات.

- [Re: PM / sleep: fix unbalanced pm runtime disable](https://lore.kernel.org/lkml/4160153.ggAvdOZIvP@vostro.rjw.lan/)
  - مثال على نقاش Rafael Wysocki حول bug في الـ pm_runtime disable flow.

- [Re: Allow rpm_resume() to succeed when runtime PM is disabled](https://lore.kernel.org/lkml/CAJZ5v0jV5QS6yxBgK0OHJ_7ivDPs7tL7Ms19dNBTUAYSfKDkCg@mail.gmail.com/)
  - نقاش حول سلوك `rpm_resume()` عند الـ disabled state.

- [RFC: Routine for checking device status during system suspend](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg630119.html)
  - نقاش حول الـ interaction بين الـ runtime PM والـ system suspend.

---

### الـ Key Git Commits

الـ commits الأساسية في تاريخ الـ runtime PM:

| الـ Commit | الوصف |
|---|---|
| دخل في **2.6.32** (Dec 2009) | الـ framework الأصلي بقلم Rafael J. Wysocki |
| دخل في **2.6.33** (Feb 2010) | إضافة الـ autosuspend و `pm_runtime_set_autosuspend_delay` |
| دخل في **3.x** | إضافة `pm_runtime_get_if_in_use` |
| دخل في **5.1** (May 2019) | إضافة `pm_runtime_resume_and_get` كبديل أآمن لـ `pm_runtime_get_sync` |
| دخل في **5.5** (Jan 2020) | إضافة `devm_pm_runtime_enable` |
| دخل في **6.x** | إضافة الـ `DEFINE_GUARD` macros للـ scoped locking |

للبحث في الـ git history:

```bash
git log --oneline --all -- drivers/base/power/runtime.c | head -30
git log --oneline --all -- include/linux/pm_runtime.h | head -30
git log --grep="pm_runtime" --oneline v2.6.31..v2.6.32
```

---

### الـ KernelNewbies.org

- [Linux 2.6.32 — Runtime PM introduction](https://kernelnewbies.org/Linux_2_6_32)
  - الـ changelog يذكر دخول الـ runtime PM framework كأحد أهم features الـ release.

- [Linux 2.6.36 — PM enhancements](https://kernelnewbies.org/Linux_2_6_36)
  - تحسينات على الـ runtime PM بعد الإطلاق.

- [Linux 2.6.37 — Further PM improvements](https://kernelnewbies.org/Linux_2_6_37)
  - مزيد من التحسينات على الـ autosuspend والـ driver integration.

- [Linux 5.5 — devm runtime PM helpers](https://kernelnewbies.org/Linux_5.5)
  - إضافة الـ `devm_pm_runtime_enable` والـ managed helpers.

- [Linux 5.17 — PM updates](https://kernelnewbies.org/Linux_5.17)
  - تحديثات الـ runtime PM في كيرنل 5.17.

---

### الـ eLinux.org Resources

- [Power Management Framework — elinux.org](https://elinux.org/Power_Management_Framework)
  - نظرة عامة على الـ PM framework مع روابط لـ slides وـ presentations.

- [Power Management for Linux — elinux.org](https://elinux.org/Power_Management)
  - الصفحة الرئيسية للـ power management في الـ embedded Linux.

- [Power Management Presentations — elinux.org](https://elinux.org/Power_Management_Presentations)
  - مجموعة presentations من ELC وـ ELCE تتضمن محاضرة Rafael Wysocki عن الـ Runtime PM (LinuxCon 2010).

- [Debugging Embedded Linux Power Management (PDF)](https://www.elinux.org/images/1/13/Debugging_Embedded_Linux_(Kernel)_Power_Management.pdf)
  - دليل عملي لـ debugging مشاكل الـ power management في الـ embedded systems.

- [Power Management Using PM Domains on SH7372 — Wysocki (PDF)](https://elinux.org/images/e/e6/Elce11_wysocki.pdf)
  - محاضرة Wysocki في ELCE 2011 عن الـ PM domains وعلاقتها بالـ runtime PM.

---

### الكتب الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 14: The Linux Device Model**
  - الـ `device` و`device_driver` structures وعلاقتها بالـ PM.
- الكتاب قديم (2005) ولا يغطي الـ runtime PM مباشرة، لكنه ضروري لفهم الـ device model الذي يبنى عليه الـ runtime PM.
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل 14: The Block I/O Layer** — background للـ device management.
- **الفصل 17: Devices and Modules** — الـ device model والـ sysfs.
- الكتاب لا يغطي الـ runtime PM بالتفصيل لكنه يبني الـ foundation.

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل 14: Kernel Debugging Techniques**
- **الفصل 15: Embedded Linux Primer** — يتناول الـ power management للـ embedded systems.
- مفيد جداً لمن يعمل على الـ embedded devices مع الـ runtime PM.

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- يغطي الـ power management subsystem بتفصيل أكبر من LDD3.
- يشرح الـ driver model والـ PM framework بعمق.

---

### الـ Conference Talks

- **Rafael J. Wysocki — Runtime Power Management Framework (LinuxCon 2010)**
  - [Slides (PDF)](https://events.static.linuxfound.org/slides/2010/linuxcon2010_wysocki.pdf)
  - أفضل مصدر لفهم فلسفة التصميم من المؤلف نفسه.

- **ELCE 2011 — PM Domains on SH7372**
  - [elinux.org slides](https://elinux.org/images/e/e6/Elce11_wysocki.pdf)

---

### Search Terms للبحث عن معلومات إضافية

للبحث في الـ mailing lists والـ git history:

```
pm_runtime_get_sync replacement
pm_runtime_resume_and_get vs get_sync
runtime PM autosuspend delay
devm_pm_runtime_enable cleanup
pm_runtime_force_suspend system sleep
RPM_SUSPENDED RPM_ACTIVE state machine
runtime PM device link supplier
pm_wq workqueue priority
rpm_resume rpm_suspend kernel internals
CONFIG_PM runtime power management driver
```

للبحث في الـ kernel source:

```bash
# كل الـ drivers التي تستخدم autosuspend
grep -r "pm_runtime_set_autosuspend_delay" drivers/ --include="*.c" -l

# مثال على driver كامل يستخدم runtime PM
grep -r "devm_pm_runtime_enable" drivers/ --include="*.c" -l | head -5

# الـ runtime PM في الـ sysfs
ls /sys/devices/*/power/runtime_*
cat /sys/devices/pci*/*/power/runtime_status
```

---

### الـ Related Subsystems للمتابعة

| الـ Subsystem | الـ File الأساسي | الصلة بالـ runtime PM |
|---|---|---|
| الـ PM QoS | `include/linux/pm_qos.h` | الـ latency constraints تؤثر على الـ suspend |
| الـ PM Domains | `include/linux/pm_domain.h` | الـ genpd يستخدم الـ runtime PM |
| الـ Device Links | `include/linux/device.h` | الـ supplier/consumer ordering |
| الـ Wakeup Sources | `include/linux/pm_wakeup.h` | منع الـ suspend عند الـ wakeup |
| الـ OPP | `include/linux/pm_opp.h` | تغيير الـ frequency مع الـ runtime PM |
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على الدالة `__pm_runtime_resume` اللي بتتفعّل في كل مرة device بتعمل runtime resume — سواء synchronous أو async. ده بيخلينا نشوف إيه الـ devices اللي بتصحى من الـ suspend في الـ kernel بالاسم والـ usage counter بتاعها.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * pm_resume_probe.c
 *
 * Intercept every __pm_runtime_resume() call and log
 * the device name + current usage_count + rpmflags.
 */

/* Core kernel headers */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>      /* kprobe API */
#include <linux/device.h>       /* struct device, dev_name() */
#include <linux/pm_runtime.h>   /* RPM_GET_PUT, RPM_ASYNC, etc. */
#include <linux/atomic.h>       /* atomic_read() */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Explorer");
MODULE_DESCRIPTION("kprobe on __pm_runtime_resume to trace runtime PM resume events");

/* ------------------------------------------------------------------ */
/* Helper: decode rpmflags bitmask into a readable string              */
/* ------------------------------------------------------------------ */
static void decode_flags(int flags, char *buf, size_t len)
{
    buf[0] = '\0';
    if (flags & RPM_ASYNC)       strlcat(buf, "ASYNC ",       len);
    if (flags & RPM_NOWAIT)      strlcat(buf, "NOWAIT ",      len);
    if (flags & RPM_GET_PUT)     strlcat(buf, "GET_PUT ",     len);
    if (flags & RPM_AUTO)        strlcat(buf, "AUTO ",        len);
    if (flags & RPM_TRANSPARENT) strlcat(buf, "TRANSPARENT ", len);
    if (!flags)                  strlcat(buf, "NONE",         len);
}

/* ------------------------------------------------------------------ */
/* kprobe pre-handler — runs BEFORE __pm_runtime_resume executes       */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * __pm_runtime_resume(struct device *dev, int rpmflags)
     *
     * On x86-64 the calling convention passes:
     *   arg0 (dev)      -> RDI
     *   arg1 (rpmflags) -> RSI
     *
     * regs_get_kernel_argument() is the portable helper for this.
     */
    struct device *dev    = (struct device *)regs_get_kernel_argument(regs, 0);
    int            flags  = (int)           regs_get_kernel_argument(regs, 1);

    char flag_str[64];

    /* Guard against a NULL device pointer (should not happen, but safe) */
    if (!dev)
        return 0;

    decode_flags(flags, flag_str, sizeof(flag_str));

    pr_info("pm_resume_probe: dev=%-32s  usage=%d  flags=[%s]\n",
            dev_name(dev),
            atomic_read(&dev->power.usage_count),
            flag_str);

    return 0;   /* 0 = let the probed function execute normally */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "__pm_runtime_resume",   /* exact kernel symbol name */
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init / module_exit                                           */
/* ------------------------------------------------------------------ */
static int __init pm_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("pm_resume_probe: register_kprobe failed (%d)\n", ret);
        return ret;
    }

    pr_info("pm_resume_probe: planted kprobe at %p (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

static void __exit pm_probe_exit(void)
{
    /*
     * Unregister MUST happen in exit so the handler pointer
     * is no longer live after the module text is freed —
     * otherwise the next resume call would jump into unmapped memory.
     */
    unregister_kprobe(&kp);
    pr_info("pm_resume_probe: kprobe removed\n");
}

module_init(pm_probe_init);
module_exit(pm_probe_exit);
```

---

### Makefile

```makefile
obj-m += pm_resume_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل الـ module
make
sudo insmod pm_resume_probe.ko

# متابعة اللوغ
sudo dmesg -w | grep pm_resume_probe

# تفريغ الـ module
sudo rmmod pm_resume_probe
```

---

### شرح كل جزء

#### `#include` الـ headers

| Header | السبب |
|---|---|
| `linux/kprobes.h` | بيوفّر `struct kprobe` و `register_kprobe` / `unregister_kprobe` |
| `linux/device.h` | تعريف `struct device` و `dev_name()` عشان نطبع اسم الجهاز |
| `linux/pm_runtime.h` | ماكروهات `RPM_*` اللي بنفسّر بيها الـ `rpmflags` |
| `linux/atomic.h` | `atomic_read()` عشان نقرأ `usage_count` بأمان |

#### `decode_flags()`

بتحوّل الـ bitmask اللي بييجي في argument تاني لـ `__pm_runtime_resume` لـ string مقروء. ده بيوضّح هل الـ resume كان async، ولا بيشيل usage count، ولا غير كده.

#### `handler_pre` — الـ callback

**الـ `regs_get_kernel_argument(regs, N)`** هو الطريقة الـ portable لاسترجاع arguments الدالة من الـ registers؛ على x86-64 بيجيب RDI و RSI على التوالي — اللي هما `dev` و `rpmflags` في signature بتاعة `__pm_runtime_resume`. الـ `pr_info` بيطبع اسم الـ device الذي أيقظ نفسه، والـ usage count الحالي، والـ flags اللي طلبت بيها الـ resume.

#### `struct kprobe kp`

الـ `.symbol_name` بيخلي الـ kernel يحسب العنوان تلقائيًا من الـ symbol table بدل ما نحطّه manually — أكثر portable عبر kernel versions مختلفة.

#### `module_init` و `module_exit`

`register_kprobe` في الـ init بيزرع **software breakpoint** على أول byte من `__pm_runtime_resume` — لما أي حد يستدعيها، الـ handler بيتشغّل قبلها. `unregister_kprobe` في الـ exit **إجبارية**: لو مشيناها من غير unregister، الكود بتاع الـ handler هيتفيّل لما الـ module text يتحرر من الذاكرة، وده بيودّي في kernel panic.
