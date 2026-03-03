## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي الملف ده جزء منه

الـ `pm_wakeirq.h` جزء من **Power Management Core** في Linux kernel. المنتِج المسؤول عنه هو Rafael J. Wysocki، والـ mailing list هو `linux-pm@vger.kernel.org`. الـ pattern في MAINTAINERS هو `include/linux/pm_*`، يعني كل ملفات الـ PM header تحت نفس المظلة.

---

### الصورة الكبيرة — قبل أي كود

#### المشكلة اللي بيحلها

تخيل معايا إنك شايل موبايل. لما الموبايل بيتحول لـ sleep وبعدين ييجيله رسالة — مين اللي صحّاه؟ الـ **interrupt**. الـ interrupt ده هو اللي بيقول للنظام "فيه حاجة محتاج انتباه".

في Linux، كل device ممكن يبقى في حالة **suspend** (نايم) لتوفير الطاقة. لكن المشكلة: لو الـ device نايم وجه له data أو event — مين اللي يصحّيه؟

الإجابة هي **Wake IRQ** — الـ interrupt المسؤول عن إيقاظ الـ device من النوم.

---

#### السيناريو الواقعي — قصة مكتملة

تخيل عندك **UART controller** (دايرة سيريال) جوه SoC على embedded Linux:

1. النظام مشغّل، الـ UART شغّال عادي، بيستقبل بيانات.
2. مفيش بيانات لفترة — kernel قرر يدخل الـ UART في **runtime suspend** لتوفير الطاقة.
3. فجأة، جهاز خارجي بعت بيانات على الـ UART.
4. هنا المشكلة: الـ UART نايم، مش قادر يستقبل.

**الحل؟** بعض الـ hardware عندها زرّين:
- **زر شغل عادي (device IO IRQ):** اللي بيجي لما تيجي بيانات فعلاً.
- **زر صحيان (dedicated wake IRQ):** زر منفصل، مهمته الوحيدة إنه يصحّي الـ device.

الـ `pm_wakeirq.h` هو الـ API اللي بيسمح لـ driver يقول للـ kernel: "الـ IRQ رقم X هو اللي هيصحّيني."

---

### ليه الملف ده مهم؟

- **بدونه:** الـ driver لازم يعمل كل حاجة بنفسه — يسجّل الـ interrupt، يدير الـ enable/disable وقت suspend/resume، يتعامل مع race conditions. ده كود متكرر ومليان bugs.
- **بيه:** الـ driver بيعمل `dev_pm_set_wake_irq(dev, irq)` مرة واحدة، وكل التعقيد يتعمل أوتوماتيك جوه الـ PM core.

---

### الفرق بين نوعين من الـ Wake IRQ

| النوع | الـ Function | الاستخدام |
|---|---|---|
| **Shared Wake IRQ** | `dev_pm_set_wake_irq()` | نفس الـ IRQ اللي بيستخدمه الـ device عادةً، بس بيتفعّل كـ wakeup source |
| **Dedicated Wake IRQ** | `dev_pm_set_dedicated_wake_irq()` | الـ hardware عنده wire منفصل تماماً للصحيان فقط |
| **Dedicated Reverse** | `dev_pm_set_dedicated_wake_irq_reverse()` | نفس الـ dedicated، بس ترتيب الـ enable/disable معكوس لبعض الـ hardware quirks |
| **Device-managed** | `devm_pm_set_wake_irq()` | نفس الـ shared، بس بيتنظف أوتوماتيك لما الـ driver بيتفصل |

---

### الـ CONFIG Guard وليه موجود

الملف فيه `#ifdef CONFIG_PM` بشكل صريح:

```c
#ifdef CONFIG_PM
extern int dev_pm_set_wake_irq(struct device *dev, int irq);
/* ... */
#else
static inline int dev_pm_set_wake_irq(struct device *dev, int irq)
{
    return 0;  /* no-op when PM is disabled */
}
#endif
```

**الـ** `CONFIG_PM` guard ده بيخلي الكود يشتغل حتى لو PM مش مفعّل في الـ kernel config — الـ functions بتتحول لـ stubs فاضية، فالـ driver ما يحتاجش يعمل `#ifdef` في كل حتة.

---

### الملفات المرتبطة اللي لازم تعرفها

#### الـ Core Implementation
| الملف | الدور |
|---|---|
| `/drivers/base/power/wakeirq.c` | الـ implementation الفعلي لكل الـ functions |
| `/drivers/base/power/wakeup.c` | إدارة الـ wakeup sources الكاملة |
| `/drivers/base/power/main.c` | الـ PM core — suspend/resume callbacks |

#### الـ Headers المرتبطة
| الملف | الدور |
|---|---|
| `include/linux/pm_wakeirq.h` | **ملفنا** — API declarations |
| `include/linux/pm_wakeup.h` | يعرّف `struct wakeup_source` و `struct wake_irq` |
| `include/linux/pm.h` | الـ PM interface الرئيسي، `struct dev_pm_ops` |
| `include/linux/pm_runtime.h` | الـ runtime PM API — `pm_runtime_resume()` وغيره |

#### الـ Internal Header
| الملف | الدور |
|---|---|
| `drivers/base/power/power.h` | الـ internal structures مش expose للخارج، بما فيهم `struct wake_irq` الكامل |

---

### مين اللي بيستخدم الـ API ده؟

الـ drivers اللي فيها hardware wake capability:
- **UART drivers** — `drivers/tty/serial/`
- **Touchscreen/input drivers** — `drivers/input/`
- **Wi-Fi/Bluetooth** — لما الموبايل يجيله نوتيفيكيشن وهو sleeping
- **USB controllers** — الـ wakeup من remote device

---

### ملخص الـ Flow بالصورة

```
Driver Probe:
  device_init_wakeup(dev, true)         ← الـ device ممكن يكون wakeup source
  dev_pm_set_dedicated_wake_irq(dev, irq) ← سجّل الـ wake IRQ

System Suspend:
  PM core → dev_pm_enable_wake_irq()   ← فعّل الـ wake IRQ
  device → suspend()

Wake Event:
  Hardware asserts wake IRQ pin
  handle_threaded_wake_irq() fires
    ↓
  pm_wakeup_event() أو pm_runtime_resume()

System Resume:
  PM core → dev_pm_disable_wake_irq()  ← عطّل الـ wake IRQ
  device → resume()

Driver Remove:
  dev_pm_clear_wake_irq(dev)           ← نظّف
```
## Phase 2: شرح الـ pm_wakeirq Framework

### المشكلة اللي بتحلّها الـ subsystem دي

لما الـ system بيدخل في حالة suspend (إيقاف مؤقت لتوفير الطاقة)، لازم يفضل في حالة استعداد لاستقبال أي حدث خارجي ممكن "يصحّيه" تاني — زي ضغطة زرار أو رسالة شبكة أو حركة على touch screen.

المشكلة الجوهرية:
- كل device عنده interrupt عادي (IO interrupt) بيشتغل لما بيجيله بيانات أو حدث طبيعي.
- لكن في حالة suspend، الـ interrupt ده ممكن يبقى معطّل أو غير كافي عشان يصحّي الـ system.
- بعض الـ hardware عنده **dedicated wake interrupt** منفصل تماماً — يعني IRQ تاني مخصوص بس للـ wake-up.
- لازم يكون في آلية موحدة تعرف تتعامل مع الاتنين: الـ IO interrupt العادي المُستخدَم كـ wake source، والـ dedicated wake interrupt المنفصل.

بدون الـ wakeirq framework، كل driver كان لازم يعمل كل الـ logic ده بنفسه — enable/disable الـ IRQ قبل وبعد الـ suspend، ويتأكد إن الـ wakeup_source اتسجّل صح. ده كود مكرر وعرضة للأخطاء.

---

### الحل — الـ approach اللي الـ kernel بياخده

الـ kernel عمل abstraction layer صغيرة ومحكمة اسمها **wakeirq framework** جوا `drivers/base/power/`.

الـ framework دي:
1. بتمثّل الـ wake interrupt في struct اسمه `wake_irq` مربوط بالـ `device`.
2. بتتحكم **أوتوماتيك** في enable/disable الـ IRQ بناءً على حالة الـ runtime PM.
3. بتدعم **نوعين** من الـ wake IRQs:
   - **Shared wake IRQ**: نفس الـ IO interrupt بيتستخدم كـ wakeup source.
   - **Dedicated wake IRQ**: IRQ منفصل بالكامل مخصوص للـ wake-up بس.
4. بتتكامل مع الـ `wakeup_source` infrastructure — اللي هي الـ subsystem المسؤولة عن تتبع مين مانع الـ system من النوم.

---

### التشبيه الواقعي — المنبّه والحارس

تخيّل مبنى فيه موظفين نايمين في الليل (الـ system في suspend):

- **الـ IO interrupt العادي** زي تليفون المكتب — بيرن لما في عمل، لكن في الليل الكل نايم ومحدش بيسمعه.
- **الـ dedicated wake IRQ** زي حارس أمن عنده جرس طوارئ منفصل — مهمته الوحيدة إنه يصحّي الكل لو في طارئ.
- **الـ wakeirq framework** هو بروتوكول إدارة المبنى اللي بيقول: "لما الليل يجي (suspend)، فعّل جرس الطوارئ (enable_irq_wake). لما الصبح يجي (resume)، وقّف الجرس وافتح تليفون المكتب تاني."

| عنصر في التشبيه | الـ kernel concept المقابل |
|---|---|
| مبنى فيه موظفين نايمين | الـ system في حالة suspend |
| تليفون المكتب العادي | الـ IO interrupt (dev IO irq) |
| حارس الأمن بجرس الطوارئ | الـ dedicated wake IRQ |
| بروتوكول إدارة المبنى | الـ wakeirq framework |
| فتح/إغلاق جرس الطوارئ | `enable_irq_wake()` / `disable_irq_wake()` |
| صحيان الموظفين | `pm_runtime_resume()` |
| تسجيل إن الجرس اشتغل | `pm_wakeup_event()` |

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Space                                │
│  /sys/devices/.../power/wakeup   (enable/disable wake from user) │
└────────────────────────────┬────────────────────────────────────┘
                             │  sysfs read/write
┌────────────────────────────▼────────────────────────────────────┐
│                    PM Core (kernel/power/)                        │
│                                                                   │
│   ┌─────────────────┐        ┌──────────────────────────────┐   │
│   │  wakeup_source  │        │   suspend/resume state machine│   │
│   │  infrastructure  │        │  (dpm_suspend / dpm_resume)  │   │
│   │ (pm_wakeup.h)   │        └──────────────┬───────────────┘   │
│   └────────┬────────┘                        │                    │
└────────────│────────────────────────────────│───────────────────┘
             │                                │
┌────────────▼────────────────────────────────▼───────────────────┐
│              pm_wakeirq Framework (drivers/base/power/)           │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   struct wake_irq                        │    │
│  │  ┌──────────┐  ┌────────┐  ┌──────────────────────┐    │    │
│  │  │ dev *    │  │  irq   │  │       status          │    │    │
│  │  │          │  │(irq nr)│  │ ALLOCATED | MANAGED   │    │    │
│  │  │          │  │        │  │ REVERSE   | ENABLED   │    │    │
│  │  └──────────┘  └────────┘  └──────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Public API:                                                      │
│  dev_pm_set_wake_irq()           ← shared IO irq as wake         │
│  dev_pm_set_dedicated_wake_irq() ← separate dedicated wake irq   │
│  dev_pm_set_dedicated_wake_irq_reverse() ← reverse enable order  │
│  dev_pm_clear_wake_irq()         ← cleanup                       │
│  devm_pm_set_wake_irq()          ← devm managed version          │
│                                                                   │
│  Internal (called by RPM core):                                   │
│  dev_pm_arm_wake_irq()           ← on suspend: enable_irq_wake   │
│  dev_pm_disarm_wake_irq()        ← on resume:  disable_irq_wake  │
│  dev_pm_enable_wake_irq_check()  ← called from rpm_suspend()     │
│  dev_pm_disable_wake_irq_check() ← called from rpm_resume()      │
│  dev_pm_enable_wake_irq_complete()← post-suspend enable (REVERSE)│
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                   IRQ Subsystem (kernel/irq/)                     │
│   enable_irq() / disable_irq_nosync()                            │
│   enable_irq_wake() / disable_irq_wake()                         │
│   request_threaded_irq()                                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                  Hardware / GIC / PMIC                            │
│   Physical IRQ line from device (e.g. UART, touchscreen, WiFi)   │
└─────────────────────────────────────────────────────────────────┘

Consumers (drivers that use the API):
  drivers/tty/serial/   → UART driver uses dev_pm_set_wake_irq()
  drivers/input/        → touchscreen with dedicated wake IRQ
  drivers/net/          → WiFi that can wake on WLAN packet
  drivers/mfd/          → PMIC with dedicated wakeirq line
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ `struct wake_irq` هو القلب:

```c
/* من drivers/base/power/power.h */
struct wake_irq {
    struct device  *dev;     /* الـ device صاحب الـ wakeirq */
    unsigned int    status;  /* flags بتوصف حالة الـ IRQ */
    int             irq;     /* رقم الـ IRQ */
    const char     *name;    /* اسم (للـ dedicated IRQ بس) */
};
```

**الـ status flags بالتفصيل:**

| Flag | قيمة | معناه |
|---|---|---|
| `WAKE_IRQ_DEDICATED_ALLOCATED` | BIT(0) | الـ IRQ اتعمله `request_threaded_irq()` — يعني dedicated |
| `WAKE_IRQ_DEDICATED_MANAGED` | BIT(1) | الـ RPM core بيدير enable/disable بتاعه أوتوماتيك |
| `WAKE_IRQ_DEDICATED_REVERSE` | BIT(2) | enable الـ IRQ *بعد* ما الـ `runtime_suspend()` callback يخلص (مش قبله) |
| `WAKE_IRQ_DEDICATED_ENABLED` | BIT(3) | الـ IRQ enabled حالياً |

**ربط الـ `wake_irq` بالـ `device`:**

الـ `struct device` عنده جوا الـ `dev_pm_info`:

```c
/* simplified من include/linux/pm.h */
struct dev_pm_info {
    spinlock_t          lock;       /* يحمي الـ wakeirq */
    struct wake_irq    *wakeirq;    /* pointer للـ wake_irq struct */
    struct wakeup_source *wakeup;   /* الـ wakeup_source المرتبط بالـ device */
    bool                can_wakeup; /* هل الـ device قادر أصلاً يعمل wakeup؟ */
    /* ... */
};
```

```
struct device
   └── power (struct dev_pm_info)
           ├── wakeirq ──────────────► struct wake_irq
           │                               ├── dev (back-pointer)
           │                               ├── irq (IRQ number)
           │                               ├── status (flags)
           │                               └── name (dedicated only)
           └── wakeup ───────────────► struct wakeup_source
                                           ├── wakeirq (same wake_irq ptr)
                                           ├── active
                                           ├── event_count
                                           └── ...
```

لاحظ إن الـ `wakeup_source` كمان عنده pointer للـ `wake_irq` — ده لأن الـ `wakeup_source` هو اللي بيتتبع إن الـ device "صحّى الـ system" فعلاً، والـ `wake_irq` هو الآلية الـ hardware اللي بتشغّله.

> **مصطلح مهم — الـ Runtime PM:** قبل ما تكمل، الـ wakeirq framework بيتكامل بشكل أساسي مع الـ **Runtime PM** (اختصار: RPM) — وهو الـ subsystem المسؤول عن إدارة power state للـ device أثناء تشغيل الـ system (مش فقط عند الـ system suspend). الـ RPM بيعمل `rpm_suspend()` و`rpm_resume()` للـ device لما مش محتاجه.

---

### النوعان من الـ Wake IRQs — فرق عملي

#### 1. الـ Shared IO Wake IRQ

```
[Device] ──── (single IRQ line) ───► [GIC] ───► Kernel
               ↑
         نفس الـ IRQ بيعمل الاتنين:
         - data I/O أثناء التشغيل الطبيعي
         - wake-up أثناء الـ suspend
```

```c
/* مثال: UART driver */
static int my_uart_probe(struct platform_device *pdev)
{
    int irq = platform_get_irq(pdev, 0);

    device_init_wakeup(&pdev->dev, true);  /* سجّل الـ device كـ wakeup capable */
    dev_pm_set_wake_irq(&pdev->dev, irq);  /* استخدم نفس الـ IRQ كـ wake source */
    /* ... */
}
```

لما `dev_pm_set_wake_irq()` تتبع الـ code:

```c
int dev_pm_set_wake_irq(struct device *dev, int irq)
{
    struct wake_irq *wirq = kzalloc(sizeof(*wirq), GFP_KERNEL);
    wirq->dev = dev;
    wirq->irq = irq;
    /* status = 0 → مش dedicated → مش هيتعمله request_threaded_irq */
    dev_pm_attach_wake_irq(dev, wirq);  /* يربطه بالـ device */
    return 0;
}
```

لاحظ: **مفيش** `request_threaded_irq()` هنا — الـ IRQ اتعمله register من قبل من الـ driver نفسه.

#### 2. الـ Dedicated Wake IRQ

```
[Device] ──── (IO IRQ line) ──────────► [GIC] ─► Kernel (data IRQ)
         └─── (WAKE IRQ line, separate) ► [GIC] ─► handle_threaded_wake_irq()
```

```c
/* مثال: touchscreen مع dedicated wake line */
static int my_ts_probe(struct platform_device *pdev)
{
    int io_irq   = platform_get_irq(pdev, 0);  /* الـ data IRQ */
    int wake_irq = platform_get_irq(pdev, 1);  /* الـ wake IRQ المنفصل */

    /* setup الـ IO IRQ بشكل عادي */
    request_irq(io_irq, my_ts_irq_handler, 0, "ts-io", priv);

    device_init_wakeup(&pdev->dev, true);

    /* الـ framework يعمل كل حاجة للـ wake IRQ */
    dev_pm_set_dedicated_wake_irq(&pdev->dev, wake_irq);
}
```

لما `dev_pm_set_dedicated_wake_irq()` تتبع الـ code:

```c
static int __dev_pm_set_dedicated_wake_irq(struct device *dev, int irq, unsigned int flag)
{
    struct wake_irq *wirq = kzalloc(sizeof(*wirq), GFP_KERNEL);
    wirq->name = kasprintf(GFP_KERNEL, "%s:wakeup", dev_name(dev));
    wirq->dev  = dev;
    wirq->irq  = irq;

    /* منع الـ spurious wakeups */
    irq_set_status_flags(irq, IRQ_DISABLE_UNLAZY);

    /* الـ framework نفسه بيعمل request للـ IRQ بـ handler خاص */
    request_threaded_irq(irq, NULL, handle_threaded_wake_irq,
                         IRQF_ONESHOT | IRQF_NO_AUTOEN,  /* مش بيتـ enable أوتوماتيك */
                         wirq->name, wirq);

    dev_pm_attach_wake_irq(dev, wirq);
    wirq->status = WAKE_IRQ_DEDICATED_ALLOCATED | flag;
}
```

**الـ handler الخاص `handle_threaded_wake_irq`:**

```c
static irqreturn_t handle_threaded_wake_irq(int irq, void *_wirq)
{
    struct wake_irq *wirq = _wirq;

    /* حالة 1: الـ system في suspend → الـ IRQ هو signal للـ wake-up */
    if (irqd_is_wakeup_set(irq_get_irq_data(irq))) {
        pm_wakeup_event(wirq->dev, 0);  /* أبلّغ الـ PM core إن فيه wakeup event */
        return IRQ_HANDLED;
    }

    /* حالة 2: الـ system شغّال لكن الـ device في runtime suspend → استيقظ */
    pm_runtime_resume(wirq->dev);  /* صحّي الـ device نفسه */
    return IRQ_HANDLED;
}
```

---

### دورة حياة الـ Wake IRQ — من الـ probe للـ suspend والـ resume

```
Driver probe()
     │
     ├── device_init_wakeup(dev, true)
     │        └── device_set_wakeup_capable() + device_wakeup_enable()
     │
     ├── dev_pm_set_dedicated_wake_irq(dev, irq)
     │        ├── kzalloc(wake_irq)
     │        ├── request_threaded_irq(..., IRQF_NO_AUTOEN)  ← disabled initially
     │        ├── dev_pm_attach_wake_irq()
     │        │       ├── dev->power.wakeirq = wirq
     │        │       └── device_wakeup_attach_irq()  ← ربط بالـ wakeup_source
     │        └── wirq->status = DEDICATED_ALLOCATED
     │
     ▼
Normal operation (device active):
     wakeirq is DISABLED (IRQ_NO_AUTOEN → IRQ enabled only when needed)

     ▼
rpm_suspend() called (device going idle):
     ├── dev_pm_enable_wake_irq_check(dev, can_change_status=true)
     │       ├── sets DEDICATED_MANAGED flag (first time)
     │       └── enable_irq(wirq->irq)  ← الـ IRQ يتفعّل
     │
     └── dev_pm_enable_wake_irq_complete()  ← لو REVERSE flag مضبوط

     ▼
System suspend (dpm_suspend path):
     dev_pm_arm_wake_irq(wirq)
         ├── if device_may_wakeup():
         │       ├── enable_irq(wirq->irq)  [if dedicated & not enabled]
         │       └── enable_irq_wake(wirq->irq)  ← يخلي الـ GIC يصحّي الـ system

     ▼
System in suspend — waiting...
     [Wake event occurs on IRQ line]
         └── handle_threaded_wake_irq()
                 └── pm_wakeup_event()  ← يبلّغ PM core

     ▼
System resume (dpm_resume path):
     dev_pm_disarm_wake_irq(wirq)
         ├── disable_irq_wake(wirq->irq)
         └── disable_irq_nosync(wirq->irq)  [if dedicated & was enabled just for arm]

     ▼
rpm_resume() called:
     dev_pm_disable_wake_irq_check(dev, cond_disable)
         └── disable_irq_nosync(wirq->irq)

     ▼
Driver remove() / devm cleanup:
     dev_pm_clear_wake_irq(dev)
         ├── device_wakeup_detach_irq()
         ├── dev->power.wakeirq = NULL
         ├── free_irq(wirq->irq, wirq)  [if DEDICATED_ALLOCATED]
         └── kfree(wirq)
```

---

### الـ REVERSE Flag — لماذا موجود؟

في بعض الـ hardware، الـ sequence الطبيعية:

```
[enable wake IRQ] → [run runtime_suspend() callback] → [device powers down]
```

بيسبب مشكلة: الـ IRQ بيتفعّل قبل ما الـ device يتوقف، فممكن يجيله edge spurious وهو بيتوقف.

الـ REVERSE flag بيعكس الـ sequence:

```
[run runtime_suspend() callback] → [device powers down] → [enable wake IRQ]
```

```c
/* driver يحتاج REVERSE */
dev_pm_set_dedicated_wake_irq_reverse(dev, irq);
/* → status |= WAKE_IRQ_DEDICATED_REVERSE */

/* في rpm_suspend(): */
dev_pm_enable_wake_irq_check(dev, true);
    /* لأن REVERSE مضبوط → مش هيعمل enable_irq() هنا */

/* بعد ما الـ runtime_suspend() callback يخلص: */
dev_pm_enable_wake_irq_complete(dev);
    /* هنا بس يعمل enable_irq() */
```

---

### الـ Framework بيملك إيه vs. بيفوّض لمين؟

| المسؤولية | مين بيعملها؟ |
|---|---|
| تخصيص وإدارة الـ `wake_irq` struct | الـ wakeirq framework |
| ربط الـ `wake_irq` بالـ `device` و`wakeup_source` | الـ wakeirq framework |
| إدارة enable/disable الـ IRQ عند الـ suspend/resume | الـ wakeirq framework |
| تسجيل الـ `request_threaded_irq` للـ dedicated IRQ | الـ wakeirq framework |
| استدعاء `enable_irq_wake()` / `disable_irq_wake()` | الـ wakeirq framework |
| تحديد متى الـ device "قادر يعمل wakeup" (`can_wakeup`) | الـ driver (عبر `device_init_wakeup`) |
| تحديد متى الـ device "مسموحله يعمل wakeup" (`wakeup`) | User space عبر sysfs أو الـ driver |
| الـ actual wake-up logic للـ device نفسه (restore state) | الـ driver عبر `runtime_resume()` callback |
| تسجيل الـ wakeup_source statistics | الـ wakeup_source infrastructure |

---

### مثال كامل — UART على ARM SoC

```c
/* drivers/tty/serial/my_uart.c */

static int my_uart_probe(struct platform_device *pdev)
{
    struct my_uart *uart;
    int irq;

    uart = devm_kzalloc(&pdev->dev, sizeof(*uart), GFP_KERNEL);
    irq  = platform_get_irq(pdev, 0);

    /* register الـ IRQ للـ UART data I/O */
    devm_request_irq(&pdev->dev, irq, my_uart_irq_handler,
                     IRQF_SHARED, "my-uart", uart);

    /* علّم الـ device إنه wakeup capable وفعّله */
    device_init_wakeup(&pdev->dev, true);

    /*
     * استخدم نفس الـ UART IRQ كـ wake source
     * لو جاله بيانات وهو suspend → يصحّي الـ system
     */
    devm_pm_set_wake_irq(&pdev->dev, irq);
    /* devm → عند remove() هيتعمل dev_pm_clear_wake_irq() أوتوماتيك */

    return 0;
}

/*
 * لو المستخدم عمل:
 * echo enabled > /sys/bus/platform/devices/my-uart/power/wakeup
 *
 * → device_may_wakeup() ترجع true
 * → لما الـ system ييجي يعمل suspend:
 *      dev_pm_arm_wake_irq() → enable_irq_wake(uart_irq)
 * → الـ GIC في ARM بيخلّي الـ IRQ ده قادر يصحّي الـ SoC من الـ WFI
 */
```

---

### ملاحظة على الـ CONFIG_PM Guard

الـ header `pm_wakeirq.h` بيعمل:

```c
#ifdef CONFIG_PM
extern int dev_pm_set_wake_irq(...);
/* ... */
#else
static inline int dev_pm_set_wake_irq(...) { return 0; }
/* ... */
#endif
```

لما `CONFIG_PM` مش مفعّل (يعني kernel مش فيه power management — نادر جداً في embedded):
- كل الـ functions بتبقى **stubs** ترجع 0 أو void.
- الـ driver code مش محتاج `#ifdef` — بيكتب نفس الـ code وهو بيشتغل صح في الحالتين.
- ده **compile-time polymorphism** — نمط أساسي في الـ kernel لتجنب الـ `#ifdef` في الـ drivers.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags & Enums — Cheatsheet

#### الـ `WAKE_IRQ_*` Status Flags (من `drivers/base/power/power.h`)

| Flag | Value | المعنى |
|------|-------|--------|
| `WAKE_IRQ_DEDICATED_ALLOCATED` | `BIT(0)` | الـ dedicated wake IRQ اتخصص فعلاً بـ `request_threaded_irq()` |
| `WAKE_IRQ_DEDICATED_MANAGED` | `BIT(1)` | الـ kernel بقى مسؤول يـ enable/disable الـ IRQ تلقائياً عند الـ suspend/resume |
| `WAKE_IRQ_DEDICATED_REVERSE` | `BIT(2)` | الـ enable بيحصل **بعد** تشغيل `->runtime_suspend()` مش قبله |
| `WAKE_IRQ_DEDICATED_ENABLED` | `BIT(3)` | الـ IRQ ده حالياً مـ enabled في الـ hardware |
| `WAKE_IRQ_DEDICATED_MASK` | `BIT(0)\|BIT(1)\|BIT(2)` | mask بيجمع أول 3 bits لتسهيل الـ check |

#### الـ DPM Driver Flags (من `include/linux/pm.h`)

| Flag | Value | الاستخدام |
|------|-------|-----------|
| `DPM_FLAG_NO_DIRECT_COMPLETE` | `BIT(0)` | امنع الـ direct-complete optimization |
| `DPM_FLAG_SMART_PREPARE` | `BIT(1)` | خد return value الـ `->prepare()` بجدية |
| `DPM_FLAG_SMART_SUSPEND` | `BIT(2)` | متصحيش الـ device من runtime suspend |
| `DPM_FLAG_MAY_SKIP_RESUME` | `BIT(3)` | ممكن تتخطى الـ noirq/early resume callbacks |

#### الـ `CONFIG_PM` Options

| Config | الأثر على `pm_wakeirq.h` |
|--------|--------------------------|
| `CONFIG_PM=y` | يفعّل كل الـ API الحقيقي (declarations فقط في الـ header) |
| `CONFIG_PM=n` | كل الـ functions بتتحول لـ stubs بترجع `0` أو فاضية |
| `CONFIG_PM_SLEEP` | يفعّل `device_wakeup_attach_irq()` و `device_wakeup_detach_irq()` |

---

### الـ Structs المهمة

#### 1. `struct wake_irq` — النجم الأساسي

**الغرض:** يمثّل الـ wake-up interrupt المرتبط بـ device معين. ممكن يكون shared مع الـ IO interrupt أو dedicated منفصل.

```c
/* من drivers/base/power/power.h */
struct wake_irq {
    struct device  *dev;     /* الـ device اللي بيمتلك الـ wake IRQ ده */
    unsigned int    status;  /* bitmask من WAKE_IRQ_DEDICATED_* flags */
    int             irq;     /* رقم الـ IRQ في الـ kernel */
    const char     *name;    /* اسم الـ IRQ — بس موجود في الـ dedicated case */
};
```

| Field | النوع | الشرح |
|-------|-------|-------|
| `dev` | `struct device *` | pointer للـ device المالك — بيُستخدم في الـ IRQ handler عشان يـ resume الـ device |
| `status` | `unsigned int` | bitmask من الـ flags اللي فوق — بيتحكم في سلوك الـ enable/disable |
| `irq` | `int` | رقم الـ IRQ سواء كان shared أو dedicated |
| `name` | `const char *` | string ديناميكية بتتعمل بـ `kasprintf` — بس في حالة الـ dedicated IRQ عشان تيجي في `/proc/interrupts` |

**الـ `name` بيتعمل كده:**
```c
wirq->name = kasprintf(GFP_KERNEL, "%s:wakeup", dev_name(dev));
/* مثال: "i2c-0:wakeup" */
```

---

#### 2. `struct dev_pm_info` — البيت اللي بيسكن فيه الـ `wake_irq`

**الغرض:** هو الـ `dev->power` struct اللي فيه كل حالة الـ PM للـ device.

الـ fields الأهم المرتبطة بالـ wakeirq:

| Field | النوع | الشرح |
|-------|-------|-------|
| `wakeirq` | `struct wake_irq *` | pointer للـ `wake_irq` المرتبط — `NULL` لو مفيش |
| `lock` | `spinlock_t` | بيحمي `wakeirq` و باقي الـ RPM state |
| `can_wakeup` | `bool:1` | الـ device registered كـ wakeup source؟ |
| `wakeup` | `struct wakeup_source *` | الـ wakeup source المرتبط (CONFIG_PM_SLEEP) |
| `runtime_status` | `enum rpm_status` | الحالة الحالية: ACTIVE/SUSPENDED/SUSPENDING/RESUMING |

---

#### 3. `struct device` — الجد الأكبر

الـ `pm_wakeirq.h` بيتعامل دايماً مع `struct device *dev` كـ entry point. الـ `dev->power` هو الـ `struct dev_pm_info`.

```
struct device
    └── struct dev_pm_info  power
            ├── spinlock_t  lock
            ├── struct wake_irq  *wakeirq  ←── ده اللي بنبني عليه
            └── struct wakeup_source  *wakeup
```

---

### Struct Relationship Diagram

```
┌─────────────────────────────────┐
│         struct device           │
│  ┌──────────────────────────┐   │
│  │    struct dev_pm_info    │   │
│  │    (dev->power)          │   │
│  │  ┌────────────────────┐  │   │
│  │  │  spinlock_t lock   │  │   │
│  │  ├────────────────────┤  │   │
│  │  │  wake_irq *wakeirq │──┼───┼──────────┐
│  │  ├────────────────────┤  │   │          │
│  │  │  wakeup_source     │  │   │          ▼
│  │  │  *wakeup           │  │   │  ┌────────────────┐
│  │  └────────────────────┘  │   │  │  struct wake_irq│
│  └──────────────────────────┘   │  │                │
└─────────────────────────────────┘  │  device *dev ──┼──► (back-ref)
                                      │  unsigned status│
                                      │  int irq        │
                                      │  const char*name│
                                      └────────────────┘
                                               │
                                               │ irq number
                                               ▼
                                     ┌──────────────────┐
                                     │  Linux IRQ Core  │
                                     │  (irq descriptor)│
                                     │  threaded handler│
                                     └──────────────────┘
```

---

### Lifecycle Diagrams

#### الـ Shared Wake IRQ (IO interrupt = wake interrupt)

```
Driver probe()
     │
     ├─► device_init_wakeup(dev, true)    ← يعمل wakeup_source
     │
     └─► dev_pm_set_wake_irq(dev, irq)
              │
              ├─ kzalloc(wake_irq)
              ├─ wirq->dev = dev
              ├─ wirq->irq = irq
              │  [name = NULL — shared mode]
              │
              └─► dev_pm_attach_wake_irq(dev, wirq)
                       │
                       ├─ spin_lock(dev->power.lock)
                       ├─ dev->power.wakeirq = wirq
                       ├─ device_wakeup_attach_irq(dev, wirq)
                       └─ spin_unlock

              الـ status = 0 (مش dedicated)


              [ Device in use — no wakeirq action ]


System suspend path
     │
     └─► dev_pm_arm_wake_irq(wirq)
              │
              ├─ device_may_wakeup(dev)?  ← check sysfs "power/wakeup"
              └─ enable_irq_wake(irq)     ← يقول للـ hardware: ده يصحّي الـ system


System resume path
     │
     └─► dev_pm_disarm_wake_irq(wirq)
              └─ disable_irq_wake(irq)


Driver remove() / unbind
     │
     └─► dev_pm_clear_wake_irq(dev)
              │
              ├─ spin_lock(dev->power.lock)
              ├─ device_wakeup_detach_irq(dev)
              ├─ dev->power.wakeirq = NULL
              ├─ spin_unlock
              │
              ├─ [WAKE_IRQ_DEDICATED_ALLOCATED؟] ← لأ في الحالة دي
              ├─ kfree(wirq->name)               ← NULL — مش هيعمل حاجة
              └─ kfree(wirq)
```

---

#### الـ Dedicated Wake IRQ (interrupt منفصل للـ wakeup)

```
Driver probe()
     │
     └─► dev_pm_set_dedicated_wake_irq(dev, irq)
              │
              └─► __dev_pm_set_dedicated_wake_irq(dev, irq, flag=0)
                       │
                       ├─ kzalloc(wake_irq)
                       ├─ kasprintf(name, "%s:wakeup", dev_name(dev))
                       ├─ wirq->dev = dev
                       ├─ wirq->irq = irq
                       │
                       ├─ irq_set_status_flags(IRQ_DISABLE_UNLAZY)
                       │   └─ منع الـ spurious wake events عند التعطيل
                       │
                       ├─ request_threaded_irq(irq,
                       │      primary=NULL,
                       │      thread=handle_threaded_wake_irq,
                       │      flags=IRQF_ONESHOT|IRQF_NO_AUTOEN,
                       │      name, wirq)
                       │   └─ الـ IRQ بيبدأ DISABLED (NO_AUTOEN)
                       │
                       ├─► dev_pm_attach_wake_irq(dev, wirq)
                       │
                       └─ wirq->status = WAKE_IRQ_DEDICATED_ALLOCATED | flag


Runtime PM path (rpm_suspend):
     │
     ├─► dev_pm_enable_wake_irq_check(dev, can_change_status=true)
     │        │
     │        ├─ MANAGED set؟ → enable_irq() مباشرة
     │        └─ مش set؟ + can_change_status؟ → set MANAGED → enable_irq()
     │             └─ wirq->status |= WAKE_IRQ_DEDICATED_ENABLED
     │
     └─► [device suspends]

Runtime PM path (rpm_resume):
     │
     └─► dev_pm_disable_wake_irq_check(dev, cond_disable=false)
              │
              ├─ MANAGED set؟ → disable_irq_nosync(irq)
              └─ wirq->status &= ~WAKE_IRQ_DEDICATED_ENABLED


Wake event fires while suspended:
     │
     └─► handle_threaded_wake_irq(irq, wirq)
              │
              ├─ irqd_is_wakeup_set(irq)?      ← هل الـ system بيعمل suspend؟
              │    └─ YES → pm_wakeup_event(dev, 0)   ← abort system suspend
              │             return IRQ_HANDLED
              │
              └─ NO → pm_runtime_resume(wirq->dev)    ← صحّي الـ device فقط
                       return IRQ_HANDLED


System suspend with dedicated IRQ:
     │
     └─► dev_pm_arm_wake_irq(wirq)
              │
              ├─ device_may_wakeup(dev)?
              ├─ DEDICATED_ALLOCATED && !DEDICATED_ENABLED?
              │    └─ enable_irq(irq)          ← enable الـ IRQ لو مش شغال
              └─ enable_irq_wake(irq)           ← اجعله يصحّي الـ system


Driver remove():
     │
     └─► dev_pm_clear_wake_irq(dev)
              │
              ├─ spin_lock → wakeirq=NULL → spin_unlock
              │
              ├─ WAKE_IRQ_DEDICATED_ALLOCATED؟ ← YES
              │    └─ free_irq(wirq->irq, wirq)
              │
              ├─ kfree(wirq->name)
              └─ kfree(wirq)
```

---

#### الـ Reverse Enable Order (`dev_pm_set_dedicated_wake_irq_reverse`)

```
Normal order:           Reverse order (WAKE_IRQ_DEDICATED_REVERSE):
─────────────           ──────────────────────────────────────────
rpm_suspend():          rpm_suspend():
  enable_irq()            run ->runtime_suspend() FIRST
  run ->runtime_suspend()   THEN enable_irq()  ← dev_pm_enable_wake_irq_complete()
```

الـ reverse مفيد لما الـ device نفسه محتاج يعمل حاجة قبل ما الـ wake IRQ يبدأ يراقب — زي ما بعض الـ peripherals بتحتاج تدخل low-power mode قبل ما تبدأ تـ assert الـ wake line.

---

### Call Flow Diagrams

#### 1. تسجيل الـ Wake IRQ من الـ Driver

```
driver probe()
  └─► dev_pm_set_wake_irq(dev, irq)           [public API — pm_wakeirq.h]
        └─► dev_pm_attach_wake_irq(dev, wirq)  [internal — wakeirq.c]
              ├─► spin_lock_irqsave(&dev->power.lock)
              ├─► dev->power.wakeirq = wirq
              ├─► device_wakeup_attach_irq(dev, wirq)  [wakeup.c — PM_SLEEP]
              └─► spin_unlock_irqrestore()
```

#### 2. تسجيل الـ Dedicated Wake IRQ

```
driver probe()
  └─► dev_pm_set_dedicated_wake_irq(dev, irq)
        └─► __dev_pm_set_dedicated_wake_irq(dev, irq, flag=0)
              ├─► kzalloc + kasprintf(name)
              ├─► irq_set_status_flags(IRQ_DISABLE_UNLAZY)    [irq core]
              ├─► request_threaded_irq(..., handle_threaded_wake_irq, ...)
              │     └── kernel IRQ core: registers threaded handler, starts DISABLED
              ├─► dev_pm_attach_wake_irq(dev, wirq)
              └─► wirq->status = ALLOCATED | flag
```

#### 3. الـ RPM Suspend Flow مع الـ Wake IRQ

```
pm_runtime_put(dev)
  └─► rpm_suspend(dev)                          [pm_runtime.c]
        ├─► dev_pm_enable_wake_irq_check(dev, true)
        │     ├─ wirq->status & DEDICATED_MASK?  ← بيفحص لو dedicated
        │     ├─ MANAGED set? → enable_irq(wirq->irq)
        │     └─ else: set MANAGED → enable_irq(wirq->irq)
        │           wirq->status |= DEDICATED_ENABLED
        │
        └─► ops->runtime_suspend(dev)           ← driver callback
              └─ [لو REVERSE]: dev_pm_enable_wake_irq_complete(dev)
                                  └─ enable_irq() هنا بدل فوق
```

#### 4. الـ RPM Resume Flow

```
pm_runtime_get(dev)  أو  wake IRQ fires
  └─► rpm_resume(dev)                           [pm_runtime.c]
        ├─► dev_pm_disable_wake_irq_check(dev, false)
        │     ├─ wirq->status & DEDICATED_MASK?
        │     ├─ MANAGED set?
        │     └─ disable_irq_nosync(wirq->irq)
        │           wirq->status &= ~DEDICATED_ENABLED
        │
        └─► ops->runtime_resume(dev)            ← driver callback
```

#### 5. الـ System Suspend/Resume (الـ Sleep path)

```
system suspend (e.g., echo mem > /sys/power/state)
  └─► device_wakeup_arm_wake_irqs()             [wakeup.c — PM_SLEEP]
        └─► dev_pm_arm_wake_irq(wirq)            [wakeirq.c]
              ├─ device_may_wakeup(dev)?
              ├─ [dedicated + not enabled] → enable_irq(wirq->irq)
              └─ enable_irq_wake(wirq->irq)      ← IRQ hardware wakeup flag

  ── system asleep ──

  wake IRQ fires
    └─► handle_threaded_wake_irq(irq, wirq)
          ├─ irqd_is_wakeup_set? → YES
          └─ pm_wakeup_event(wirq->dev, 0)       ← يصحّي الـ system

system resume
  └─► device_wakeup_disarm_wake_irqs()          [wakeup.c]
        └─► dev_pm_disarm_wake_irq(wirq)
              ├─ disable_irq_wake(wirq->irq)
              └─ [dedicated + not managed-enabled] → disable_irq_nosync()
```

---

### Locking Strategy

#### القفل الوحيد: `dev->power.lock` (spinlock)

كل الـ access على `dev->power.wakeirq` محمي بـ `spin_lock_irqsave(&dev->power.lock)`.

| العملية | القفل المستخدم | السبب |
|--------|---------------|-------|
| `dev_pm_attach_wake_irq()` | `spin_lock_irqsave` | ممكن يتنادى من IRQ context |
| `dev_pm_clear_wake_irq()` | `spin_lock_irqsave` | نفس السبب |
| `dev_pm_enable_wake_irq_check()` | **المستدعي** مسؤول | بيتنادى من `rpm_suspend()` اللي شايل القفل |
| `dev_pm_disable_wake_irq_check()` | **المستدعي** مسؤول | نفس الكلام |
| `wirq->status` mutations | محمي بـ `dev->power.lock` | لازم تشيله قبل ما تبدل الـ status |

#### Lock Ordering

```
الـ rule: مفيش قفل تاني يتاخد وإنت شايل dev->power.lock
           الـ irq core functions (enable_irq / disable_irq_nosync) بيتنادوا
           من OUTSIDE القفل في معظم الحالات
```

```
CORRECT ✓                          WRONG ✗
──────────────────────             ──────────────────────────────
spin_lock(dev->power.lock)         spin_lock(dev->power.lock)
  dev->power.wakeirq = wirq          enable_irq(wirq->irq)  ← ممكن يـ deadlock
spin_unlock(dev->power.lock)               لأن enable_irq ممكن يأخد lock تاني
enable_irq(wirq->irq)  ← outside
```

#### الـ `disable_irq_nosync` vs `disable_irq`

الـ code بيستخدم `disable_irq_nosync()` عمداً في `dev_pm_disable_wake_irq_check()` عشان:
- مش بينتظر انتهاء الـ IRQ handler الشغّالة حالياً
- آمن يتنادى من داخل الـ atomic/PM context
- لو استخدم `disable_irq()` ممكن يحصل deadlock لو الـ IRQ handler نفسه حاول يعمل PM operation

#### الـ `IRQ_DISABLE_UNLAZY` Flag

```c
irq_set_status_flags(irq, IRQ_DISABLE_UNLAZY);
```

ده بيمنع الـ "lazy disable" — بعض الـ IRQ controllers بيأخروا تعطيل الـ IRQ. الـ wake IRQ محتاج يتعطل فوراً عشان ما يصحيش الـ system بعد ما المفروض يكون اتعطل.

---

### ملخص — من يستدعي إيه؟

```
┌──────────────────────────────────────────────────────────┐
│                  Driver (consumer)                       │
│  dev_pm_set_wake_irq()                                   │
│  dev_pm_set_dedicated_wake_irq()                         │
│  dev_pm_set_dedicated_wake_irq_reverse()                 │
│  devm_pm_set_wake_irq()          ← auto-cleanup variant  │
│  dev_pm_clear_wake_irq()                                 │
└──────────────────────────┬───────────────────────────────┘
                           │ يبني wake_irq ويحطه في dev->power
                           ▼
┌──────────────────────────────────────────────────────────┐
│               PM Runtime Core (rpm_*)                    │
│  dev_pm_enable_wake_irq_check()   ← من rpm_suspend       │
│  dev_pm_disable_wake_irq_check()  ← من rpm_resume        │
│  dev_pm_enable_wake_irq_complete() ← REVERSE mode        │
└──────────────────────────┬───────────────────────────────┘
                           │ يتحكم في enable/disable الـ IRQ
                           ▼
┌──────────────────────────────────────────────────────────┐
│               PM Sleep Core (suspend/resume)             │
│  dev_pm_arm_wake_irq()    ← من device_wakeup_arm_*       │
│  dev_pm_disarm_wake_irq() ← من device_wakeup_disarm_*    │
└──────────────────────────┬───────────────────────────────┘
                           │ يـ arm/disarm الـ wakeup للـ system sleep
                           ▼
                    IRQ Hardware Layer
              (enable_irq_wake / disable_irq_wake)
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Public API (exported via `EXPORT_SYMBOL_GPL`)

| Function | Category | الغرض |
|---|---|---|
| `dev_pm_set_wake_irq` | Registration | ربط الـ IO IRQ بالـ wake source |
| `dev_pm_set_dedicated_wake_irq` | Registration | ربط IRQ مستقل للـ wakeup (ترتيب enable عادي) |
| `dev_pm_set_dedicated_wake_irq_reverse` | Registration | ربط IRQ مستقل للـ wakeup (enable بعد runtime_suspend) |
| `dev_pm_clear_wake_irq` | Cleanup | فك ربط الـ wake IRQ وتحرير الموارد |
| `devm_pm_set_wake_irq` | Registration | نفس `dev_pm_set_wake_irq` بس managed (auto-cleanup) |

#### Internal / Subsystem-only

| Function | من يستدعيها | الغرض |
|---|---|---|
| `dev_pm_attach_wake_irq` | داخلي فقط | إلحاق الـ `wake_irq` struct بالـ device |
| `handle_threaded_wake_irq` | IRQ subsystem | threaded IRQ handler للـ dedicated wakeirq |
| `__dev_pm_set_dedicated_wake_irq` | داخلي فقط | implementation مشترك لـ dedicated wake IRQ |
| `dev_pm_enable_wake_irq_check` | `rpm_suspend/resume` | تفعيل الـ wake IRQ بشكل lazy وآمن |
| `dev_pm_disable_wake_irq_check` | `rpm_suspend/resume` | تعطيل الـ wake IRQ بشكل conditional |
| `dev_pm_enable_wake_irq_complete` | `rpm_suspend` | تفعيل الـ wake IRQ بعد `->runtime_suspend()` |
| `dev_pm_arm_wake_irq` | PM sleep core | تجهيز الـ IRQ كـ wakeup source قبل السليب |
| `dev_pm_disarm_wake_irq` | PM sleep core | إلغاء تجهيز الـ wakeup بعد الاستيقاظ |
| `devm_pm_clear_wake_irq` | devm framework | wrapper للـ cleanup callback |

---

### البنية الأساسية — `struct wake_irq`

```c
/* drivers/base/power/power.h */
struct wake_irq {
    struct device  *dev;     /* الـ device المالك */
    unsigned int    status;  /* bitmask من الـ flags التالية */
    int             irq;     /* رقم الـ IRQ */
    const char     *name;    /* اسم الـ IRQ (للـ dedicated فقط) */
};

/* Status flags */
#define WAKE_IRQ_DEDICATED_ALLOCATED  BIT(0) /* IRQ allocated via request_threaded_irq */
#define WAKE_IRQ_DEDICATED_MANAGED    BIT(1) /* enable/disable managed by RPM core */
#define WAKE_IRQ_DEDICATED_REVERSE    BIT(2) /* enable يحصل بعد ->runtime_suspend() */
#define WAKE_IRQ_DEDICATED_MASK       (BIT(0)|BIT(1)|BIT(2))
#define WAKE_IRQ_DEDICATED_ENABLED    BIT(3) /* الـ IRQ مفعّل حالياً */
```

الـ **`wake_irq`** هو قلب المنظومة دي. بيتخزن في `dev->power.wakeirq` ويمثّل إما:
- **Shared IRQ**: نفس الـ IRQ بتاع الـ IO (لا يتخصص handler).
- **Dedicated IRQ**: IRQ منفصل عنده `handle_threaded_wake_irq` كـ handler.

---

### Group 1: Registration — ربط الـ Wake IRQ بالـ Device

#### `dev_pm_set_wake_irq`

```c
int dev_pm_set_wake_irq(struct device *dev, int irq);
```

بتربط الـ IO interrupt العادي بتاع الـ device كـ wake source. مش بتعمل `request_irq` لأن الـ IRQ بالفعل معمول له `request_irq` من الـ driver. بس بتعمل `kzalloc` لـ `wake_irq` struct وتخزّنه في `dev->power.wakeirq`.

**Parameters:**
- `dev` — الـ device المراد ربطه.
- `irq` — رقم الـ IRQ البتاع الـ IO. لازم `>= 0`.

**Return:** `0` عند النجاح، أو سالب على error (`-EINVAL`, `-ENOMEM`, `-EEXIST`).

**Key details:**
- بتاخد `dev->power.lock` (spinlock) عشان الكتابة في `dev->power.wakeirq` تبقى atomic.
- بتستدعي `device_wakeup_attach_irq()` اللي بتربط الـ `wake_irq` بالـ `wakeup_source` الخاصة بالـ device.
- الـ `wirq->status` بيبقى `0` (مش dedicated)، يعني الـ runtime PM core مش هيديره بـ enable/disable.
- لو `dev->power.wakeirq` موجود بالفعل، بترجع `-EEXIST` مع warning.

**من يستدعيها:** الـ driver في `probe()` بعد `device_init_wakeup()`.

**مثال واقعي:**
```c
/* UART driver probe */
device_init_wakeup(dev, true);
dev_pm_set_wake_irq(dev, uart->irq);
```

---

#### `dev_pm_set_dedicated_wake_irq`

```c
int dev_pm_set_dedicated_wake_irq(struct device *dev, int irq);
```

بتربط IRQ منفصل (غير IRQ الـ IO) كـ wake source مع threaded handler. الـ use case: hardware زي بعض الـ touchscreens أو الـ Bluetooth chips عندها pin مخصص للـ wakeup منفصل عن الـ data lines.

**Parameters:**
- `dev` — الـ device.
- `irq` — رقم الـ dedicated wakeup IRQ.

**Return:** `0` أو سالب على error.

**Key details:**
- بتستدعي `__dev_pm_set_dedicated_wake_irq(dev, irq, 0)` — flag = 0 يعني ترتيب enable عادي.
- بتعمل `request_threaded_irq` بـ `IRQF_ONESHOT | IRQF_NO_AUTOEN` — الـ IRQ مش هيتفعّل تلقائياً.
- بتعمل `irq_set_status_flags(irq, IRQ_DISABLE_UNLAZY)` لمنع الـ spurious wake events.
- الـ `wirq->status` بيبقى `WAKE_IRQ_DEDICATED_ALLOCATED`.

**من يستدعيها:** الـ driver في `probe()`.

---

#### `dev_pm_set_dedicated_wake_irq_reverse`

```c
int dev_pm_set_dedicated_wake_irq_reverse(struct device *dev, int irq);
```

نفس `dev_pm_set_dedicated_wake_irq` لكن بفرق واحد مهم: بتضيف flag **`WAKE_IRQ_DEDICATED_REVERSE`** اللي بيخلّي الـ RPM core يفعّل الـ wake IRQ **بعد** تنفيذ `->runtime_suspend()` callback، مش قبلها.

**Parameters:** نفس السابقة.

**Return:** `0` أو سالب.

**Key details:**
- الـ use case: لو الـ `->runtime_suspend()` callback نفسه بيحتاج يشوف إن الـ wake IRQ اتفعّل أو لأ — يعني الـ device logic نفسها بتحدد التسلسل.
- بتستدعي `__dev_pm_set_dedicated_wake_irq(dev, irq, WAKE_IRQ_DEDICATED_REVERSE)`.

---

#### `__dev_pm_set_dedicated_wake_irq` (internal)

```c
static int __dev_pm_set_dedicated_wake_irq(struct device *dev, int irq,
                                            unsigned int flag);
```

الـ implementation المشترك للـ dedicated wake IRQ functions. بتعمل كل الشغل الفعلي.

**Pseudocode:**
```
allocate wake_irq struct
allocate name = "{dev_name}:wakeup"
irq_set_status_flags(irq, IRQ_DISABLE_UNLAZY)   // prevent lazy disable races
request_threaded_irq(irq, NULL,
    handle_threaded_wake_irq,
    IRQF_ONESHOT | IRQF_NO_AUTOEN,
    name, wirq)
dev_pm_attach_wake_irq(dev, wirq)
wirq->status = WAKE_IRQ_DEDICATED_ALLOCATED | flag
return 0

on error:
    free_irq → kfree(name) → kfree(wirq)
```

**Key details:**
- `IRQF_NO_AUTOEN` ضروري جداً — الـ device بيبدأ في `RPM_SUSPENDED` state، فلو الـ IRQ اتفعّل تلقائياً، أول `pm_runtime_get()` هيحاول يعمل disable لـ IRQ مش enabled أصلاً.
- `IRQ_DISABLE_UNLAZY` بيمنع الـ kernel من تأجيل الـ disable، وده مهم عشان لو جه wake event بعد قرار الـ suspend بشوية، مش يعمل spurious wakeup.

---

#### `devm_pm_set_wake_irq`

```c
int devm_pm_set_wake_irq(struct device *dev, int irq);
```

نسخة الـ device-managed من `dev_pm_set_wake_irq`. بتعمل نفس الشغل، بس بتسجّل `devm_pm_clear_wake_irq` كـ cleanup action عن طريق `devm_add_action_or_reset()`.

**Parameters:** نفس `dev_pm_set_wake_irq`.

**Return:** `0` أو سالب.

**Key details:**
- لو فشل تسجيل الـ devm action، بيعمل rollback تلقائي (`_or_reset`).
- مش فيه `devm_pm_set_dedicated_wake_irq` — الـ dedicated variant محتاج يتعامل معاها يدوياً.
- الـ `devm_pm_clear_wake_irq` هو مجرد wrapper بيستدعي `dev_pm_clear_wake_irq(dev)`.

---

### Group 2: Cleanup — تحرير الـ Wake IRQ

#### `dev_pm_clear_wake_irq`

```c
void dev_pm_clear_wake_irq(struct device *dev);
```

بتفكّ ربط الـ wake IRQ عن الـ device وبتحرر كل الموارد. مصممة تتسمّى بأمان حتى لو مفيش wake IRQ متسجّل.

**Parameters:**
- `dev` — الـ device المراد تنظيفه.

**Return:** void.

**Key details:**
- بتاخد `dev->power.lock` عشان تعمل detach atomic.
- بتستدعي `device_wakeup_detach_irq(dev)` لفصل الـ `wake_irq` عن الـ `wakeup_source`.
- بتحط `dev->power.wakeirq = NULL` قبل تحرير الـ spinlock — يعني أي thread تاني هيشوف NULL فوراً.
- بعد تحرير الـ lock، لو الـ IRQ كان dedicated (`WAKE_IRQ_DEDICATED_ALLOCATED`): بتستدعي `free_irq()`.
- بعدين بتعمل `kfree(wirq->name)` و `kfree(wirq)`.
- **مهم:** آمنة تتسمّى من driver مش عنده wake IRQ — بتـ return مباشرة لو `wirq == NULL`.

**من يستدعيها:** الـ driver في `remove()`، أو تلقائياً عن طريق الـ devm framework.

**مثال واقعي:**
```c
static int my_driver_remove(struct platform_device *pdev)
{
    dev_pm_clear_wake_irq(&pdev->dev);
    device_init_wakeup(&pdev->dev, false);
    return 0;
}
```

---

### Group 3: IRQ Handler — معالجة الـ Wake Interrupt

#### `handle_threaded_wake_irq` (internal)

```c
static irqreturn_t handle_threaded_wake_irq(int irq, void *_wirq);
```

الـ threaded IRQ handler الخاص بالـ dedicated wake IRQ. بيشتغل في kernel thread context (مش في hardirq context) — وده مهم لأن الـ device قد يحتاج يعمل power up ويعيد تحميل state.

**Parameters:**
- `irq` — رقم الـ IRQ.
- `_wirq` — pointer لـ `struct wake_irq` (الـ private data اللي اتدت في `request_threaded_irq`).

**Return:** دايماً `IRQ_HANDLED`.

**Pseudocode:**
```
if (irqd_is_wakeup_set(irq_get_irq_data(irq))):
    # الـ system في حالة suspend أو بيحاول يعمل suspend
    pm_wakeup_event(wirq->dev, 0)   # abort the suspend
    return IRQ_HANDLED

# الـ system شغّال عادي، الـ device جاهلو يتوقف وبعدين اتبعّت له signal
pm_runtime_resume(wirq->dev)       # synchronous resume
return IRQ_HANDLED
```

**Key details:**
- `irqd_is_wakeup_set()`: بتشيك لو الـ IRQ مضبوط كـ wakeup interrupt على مستوى الـ IRQ chip (يعني احنا في سياق suspend).
- `pm_wakeup_event()` بـ timeout = 0: بتسجّل wakeup event وبتعمل abort للـ suspend الجاري. مش بتعمل resume.
- `pm_runtime_resume()`: بتعمل synchronous resume — مش async ولا nowait. الـ comment في الكود بيوضّح إن ده مقصود عشان نضمن إن الـ device اتوقف قبل ما نرجع.
- مش بتعيد إرسال الـ missed IRQs — افتراض إن `pm_runtime_resume()` callback بتعرف تتعامل مع الـ situation.

---

### Group 4: Runtime PM Integration — التكامل مع الـ RPM Core

الـ functions دي بتتسمّى من الـ RPM state machine داخل `drivers/base/power/runtime.c`. مش من الـ drivers مباشرة.

#### `dev_pm_enable_wake_irq_check`

```c
void dev_pm_enable_wake_irq_check(struct device *dev, bool can_change_status);
```

بتفعّل الـ dedicated wake IRQ بشكل lazy وآمن أثناء الـ RPM suspend/resume cycle. الـ "lazy" هنا يعني إن الـ IRQ مش بيتفعّل في أول مرة (`can_change_status = true`) إلا لو الـ state machine قررت كده.

**Parameters:**
- `dev` — الـ device.
- `can_change_status` — لو `true`: يعني احنا في السياق اللي يسمح بتغيير الـ `WAKE_IRQ_DEDICATED_MANAGED` flag (يعني أول مرة في `rpm_suspend()`).

**Return:** void.

**Key details:**
- لو `WAKE_IRQ_DEDICATED_MANAGED` مش set و `can_change_status = true`: بتضيف الـ flag وتكمل.
- لو `WAKE_IRQ_DEDICATED_REVERSE` مضبوط و `can_change_status = true`: مش بتفعّل الـ IRQ هنا — بتتركه لـ `dev_pm_enable_wake_irq_complete()`.
- بتستدعي `enable_irq()` وبتضيف `WAKE_IRQ_DEDICATED_ENABLED` للـ status.
- الـ caller لازم يمسك `dev->power.lock` أو يكون disabled الـ RPM.

**من يستدعيها:** `rpm_suspend()`, `rpm_resume()`, `pm_runtime_force_suspend()`, `pm_runtime_force_resume()`.

**Flow diagram:**
```
dev_pm_enable_wake_irq_check()
├── wirq == NULL or !DEDICATED_MASK → return
├── DEDICATED_MANAGED set? → goto enable
├── can_change_status? → set MANAGED flag → goto enable
└── return (first call, not yet managed)

enable:
├── !can_change_status OR !REVERSE → enable_irq() + set ENABLED
└── otherwise → skip (REVERSE mode: wait for _complete)
```

---

#### `dev_pm_disable_wake_irq_check`

```c
void dev_pm_disable_wake_irq_check(struct device *dev, bool cond_disable);
```

بتعطّل الـ dedicated wake IRQ أثناء الـ RPM resume cycle. "conditional" يعني ممكن تتجاهل الـ disable لو الـ REVERSE mode مضبوط.

**Parameters:**
- `dev` — الـ device.
- `cond_disable` — لو `true` والـ `WAKE_IRQ_DEDICATED_REVERSE` مضبوط: ارجع بدون disable.

**Return:** void.

**Key details:**
- لو `DEDICATED_MANAGED` مش set: ما يعملش حاجة (الـ IRQ مش managed).
- بتعمل `disable_irq_nosync()` — مش `disable_irq()` — عشان ممكن تتسمّى من context ممكن يعمل deadlock لو استنّت الـ IRQ handler.
- بتشيل `WAKE_IRQ_DEDICATED_ENABLED` من الـ status.

**من يستدعيها:** نفس callers الـ function السابقة.

---

#### `dev_pm_enable_wake_irq_complete`

```c
void dev_pm_enable_wake_irq_complete(struct device *dev);
```

بتكمّل تفعيل الـ wake IRQ بعد تنفيذ `->runtime_suspend()` callback — دي الخطوة التانية في الـ REVERSE mode.

**Parameters:**
- `dev` — الـ device.

**Return:** void.

**Key details:**
- بتتحقق إن `WAKE_IRQ_DEDICATED_MANAGED` و `WAKE_IRQ_DEDICATED_REVERSE` كلاهم مضبوطين.
- بتستدعي `enable_irq()` وبتضيف `WAKE_IRQ_DEDICATED_ENABLED`.
- بتتسمّى من `rpm_suspend()` أو `pm_runtime_force_suspend()` **بعد** نجاح الـ suspend callback.

**السبب في وجودها:**

في الـ REVERSE mode، الـ flow بيبقى:
```
rpm_suspend():
    dev_pm_enable_wake_irq_check(dev, true)  → skip enable (REVERSE)
    driver->runtime_suspend(dev)              → device يدخل sleep
    dev_pm_enable_wake_irq_complete(dev)     → enable IRQ هنا
```

---

### Group 5: Sleep (System Suspend) Integration

الـ functions دي بتتسمّى من `drivers/base/power/wakeup.c` أثناء system-wide suspend.

#### `dev_pm_arm_wake_irq`

```c
void dev_pm_arm_wake_irq(struct wake_irq *wirq);
```

بتجهّز الـ wake IRQ لعمل system wakeup أثناء الـ system suspend. بتعمل `enable_irq_wake()` على الـ IRQ اللي بيخلّي الـ IRQ chip يصحّى الـ CPU من الـ sleep state.

**Parameters:**
- `wirq` — الـ `wake_irq` struct. لو `NULL`، بترجع بهدوء.

**Return:** void.

**Key details:**
- بتتحقق أولاً من `device_may_wakeup()` — لو الـ device مش enabled في sysfs كـ wakeup source: ترجع بدون عمل حاجة.
- لو الـ IRQ كان dedicated (`WAKE_IRQ_DEDICATED_ALLOCATED`) وكان disabled (`!WAKE_IRQ_DEDICATED_ENABLED`): بتعمل `enable_irq()` أولاً عشان الـ `enable_irq_wake()` تشتغل.
- `enable_irq_wake()` بتخلي الـ IRQ controller يبعّت wakeup signal للـ CPU حتى لو هو في power-down state.

**من يستدعيها:** `device_wakeup_arm_wake_irqs()` في `drivers/base/power/wakeup.c` أثناء system suspend.

---

#### `dev_pm_disarm_wake_irq`

```c
void dev_pm_disarm_wake_irq(struct wake_irq *wirq);
```

عكس `dev_pm_arm_wake_irq` — بتلغي الـ wakeup configuration بعد الاستيقاظ من الـ system suspend.

**Parameters:**
- `wirq` — الـ `wake_irq` struct.

**Return:** void.

**Key details:**
- بتعمل `disable_irq_wake()` أولاً.
- لو الـ IRQ كان dedicated وكان disabled قبل الـ arm (`!WAKE_IRQ_DEDICATED_ENABLED`): بتعمل `disable_irq_nosync()` عشان يرجع لحالته الأصلية.
- الـ logic هنا بتحافظ على الـ enabled/disabled state اللي كانت موجودة قبل الـ suspend.

**من يستدعيها:** `device_wakeup_disarm_wake_irqs()` بعد الاستيقاظ.

---

### Group 6: Internal Attachment

#### `dev_pm_attach_wake_irq` (internal)

```c
static int dev_pm_attach_wake_irq(struct device *dev, struct wake_irq *wirq);
```

الـ internal helper اللي بتربط الـ `wake_irq` struct بالـ device فعلياً. كل الـ registration functions العامة بتمر بيها.

**Parameters:**
- `dev` — الـ device.
- `wirq` — الـ struct الجاهزة والمملّية.

**Return:** `0` أو `-EINVAL` أو `-EEXIST`.

**Key details:**
- بتاخد `dev->power.lock` (spinlock with IRQ save) — ده ضروري لأن الـ IRQ بيتمسك وممكن يتشغّل في أي وقت.
- بتعمل `dev_WARN_ONCE()` لو موجود wake IRQ بالفعل قبل ما تبطل.
- بتستدعي `device_wakeup_attach_irq()` اللي بتضيف الـ `wake_irq` pointer في الـ `wakeup_source` الخاصة بالـ device (للربط مع الـ wakeup statistics subsystem).

---

### الـ Big Picture — كيف تتكامل الـ Functions

```
                     ┌─────────────────────────────┐
                     │         Driver probe()       │
                     │  device_init_wakeup(dev, 1)  │
                     │  dev_pm_set_wake_irq(dev, irq)│   ← shared IRQ
                     │      --- OR ---              │
                     │  dev_pm_set_dedicated_wake_  │
                     │  irq(dev, dedicated_irq)     │   ← dedicated IRQ
                     └─────────────┬───────────────┘
                                   │ attach to dev->power.wakeirq
                                   ▼
                     ┌─────────────────────────────┐
                     │   struct wake_irq in memory  │
                     │   (dev, irq, status, name)   │
                     └──────┬──────────────┬────────┘
                            │              │
              ┌─────────────▼──┐    ┌──────▼──────────────────┐
              │  Runtime PM     │    │    System Suspend        │
              │  rpm_suspend()  │    │    device_wakeup_arm_    │
              │  ↓              │    │    wake_irqs()           │
              │  enable_wake_   │    │    ↓                     │
              │  irq_check()    │    │    dev_pm_arm_wake_irq() │
              │                 │    │    ↓                     │
              │  rpm_resume()   │    │    enable_irq_wake()     │
              │  ↓              │    │                          │
              │  disable_wake_  │    │    After resume:         │
              │  irq_check()    │    │    dev_pm_disarm_wake_   │
              └─────────────────┘    │    irq()                 │
                                     └──────────────────────────┘

              ┌──────────────────────────────────────────────┐
              │   Dedicated IRQ fires (handle_threaded_...)  │
              │   ├── System in suspend? → pm_wakeup_event() │
              │   └── Runtime idle?     → pm_runtime_resume()│
              └──────────────────────────────────────────────┘

              ┌────────────────────────┐
              │     Driver remove()    │
              │  dev_pm_clear_wake_irq │
              │  → free_irq (if ded.) │
              │  → kfree(wirq)        │
              └────────────────────────┘
```

---

### ملاحظات مهمة للـ Driver Developer

| الحالة | الـ Function الصح |
|---|---|
| عندك IRQ واحد بس للـ IO والـ wakeup | `dev_pm_set_wake_irq` |
| عندك IRQ منفصل للـ wakeup وتريد enable قبل suspend callback | `dev_pm_set_dedicated_wake_irq` |
| عندك IRQ منفصل للـ wakeup وتريد enable بعد suspend callback | `dev_pm_set_dedicated_wake_irq_reverse` |
| بدك auto-cleanup عند driver removal | `devm_pm_set_wake_irq` |
| لازم تعمل cleanup يدوي | `dev_pm_clear_wake_irq` |

**قاعدة ذهبية:** دايماً استدعي `device_init_wakeup(dev, true)` قبل أي من الـ `dev_pm_set_*` functions، وبعد الـ `dev_pm_clear_wake_irq` استدعي `device_init_wakeup(dev, false)`.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem ده هو **pm_wakeirq** — بيتحكم في الـ wake IRQ اللي بتصحّي الـ device من الـ suspend. الـ bugs فيه بتظهر على شكل: الـ device مش بتصحى (wake failure)، أو بتصحى من غير سبب (spurious wakeup)، أو الـ system مش بيقدر يـ suspend أصلاً.

---

### Software Level

#### 1. debugfs Entries

الـ debugfs بيكشف معلومات الـ wakeup sources والـ IRQ state.

```bash
# Mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# شوف كل الـ wakeup sources النشطة
cat /sys/kernel/debug/wakeup_sources

# شوف معلومات الـ IRQ line نفسه
cat /sys/kernel/debug/irq/<IRQ_NUM>/actions
cat /sys/kernel/debug/irq/<IRQ_NUM>/irqchip_states

# PM runtime state للـ device
cat /sys/kernel/debug/pm_genpd/*/summary        # لو الـ device جوه power domain
ls /sys/kernel/debug/devices/                   # قائمة الـ devices
```

**تفسير output الـ wakeup_sources:**

```
name            active_count event_count  wakeup_count  total_time  ...
mmc0            5            5            2             0ms
usb-wakeup      0            0            0             0ms
```

- `active_count` صفر = الـ wake source مش بيتفعّل خالص → مشكلة في الـ IRQ setup
- `wakeup_count` > 0 = الـ source منع suspend قبل كده

#### 2. sysfs Entries

```bash
# هل الـ device قادرة تصحي النظام؟
cat /sys/devices/<path>/power/wakeup
# output: "enabled" أو "disabled"

# هل الـ device في wakeup_path؟
cat /sys/devices/<path>/power/wakeup_count
cat /sys/devices/<path>/power/wakeup_active_count
cat /sys/devices/<path>/power/wakeup_total_time_ms
cat /sys/devices/<path>/power/wakeup_last_time_ms
cat /sys/devices/<path>/power/wakeup_abort_count

# تفعيل/تعطيل الـ wakeup يدوياً للتجربة
echo enabled  > /sys/devices/<path>/power/wakeup
echo disabled > /sys/devices/<path>/power/wakeup

# الـ IRQ المربوط بالـ device
cat /proc/interrupts | grep <device_name>
```

#### 3. ftrace — Tracepoints والـ Events

```bash
# تفعيل الـ PM tracepoints
cd /sys/kernel/debug/tracing

echo 1 > tracing_on
echo > trace

# تتبع أحداث الـ suspend/resume
echo "power:device_pm_callback_start"  >> set_event
echo "power:device_pm_callback_end"    >> set_event
echo "power:suspend_resume"            >> set_event
echo "irq:irq_handler_entry"           >> set_event
echo "irq:irq_handler_exit"            >> set_event

echo 1 > tracing_on

# شغّل suspend للتجربة
echo mem > /sys/power/state &
sleep 2

cat trace | grep -E "(wakeirq|wake_irq|<device_name>)"
```

**تتبع دالة بعينها:**
```bash
echo "dev_pm_set_wake_irq"             >> set_ftrace_filter
echo "dev_pm_clear_wake_irq"           >> set_ftrace_filter
echo "dev_pm_set_dedicated_wake_irq"   >> set_ftrace_filter
echo function                          > current_tracer
echo 1                                 > tracing_on
cat trace
```

#### 4. printk والـ Dynamic Debug

```bash
# تفعيل dynamic debug لـ wakeirq subsystem
echo "file drivers/base/power/wakeirq.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/base/power/wakeup.c  +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل PM debug messages بشكل عام
echo "module <your_driver> +p" > /sys/kernel/debug/dynamic_debug/control

# تحديد log level
dmesg -n 7       # KERN_DEBUG
dmesg --follow | grep -iE "(wake|irq|pm)"
```

**نقاط مقترحة للـ printk داخل الـ driver:**
```c
/* بعد dev_pm_set_wake_irq مباشرةً */
dev_dbg(dev, "wake irq %d configured, status=%lu\n",
        irq, dev->power.wakeirq ? dev->power.wakeirq->status : -1UL);

/* في suspend callback */
dev_dbg(dev, "suspending, wakeirq active=%d\n",
        device_may_wakeup(dev));
```

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_PM_DEBUG` | يفعّل رسائل PM debug العامة |
| `CONFIG_PM_ADVANCED_DEBUG` | يضيف sysfs entries إضافية للـ devices |
| `CONFIG_PM_SLEEP_DEBUG` | debug للـ suspend/resume path |
| `CONFIG_PM_TRACE_RTC` | يسجّل آخر PM event في الـ RTC لتشخيص hangs |
| `CONFIG_IRQ_DOMAIN_DEBUG` | يعرض IRQ domain mappings في debugfs |
| `CONFIG_GENERIC_IRQ_DEBUGFS` | يعرض `/sys/kernel/debug/irq/` |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في `dev->power.lock` |
| `CONFIG_DEBUG_SPINLOCK` | يكتشف مشاكل الـ spinlock |
| `CONFIG_PROVE_LOCKING` | تحقق من صحة الـ locking order |
| `CONFIG_KASAN` | يكتشف memory corruption في الـ `wake_irq` struct |

```bash
# تأكد أن الـ options مفعّلة
zcat /proc/config.gz | grep -E "CONFIG_PM_DEBUG|CONFIG_PM_SLEEP_DEBUG|CONFIG_GENERIC_IRQ_DEBUGFS"
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# عرض كل الـ IRQ lines ومعدلات حدوثها
watch -n1 "cat /proc/interrupts"

# iرؤية الـ IRQ affinity والـ wake capability
cat /proc/irq/<IRQ_NUM>/smp_affinity
cat /sys/kernel/irq/<IRQ_NUM>/actions
cat /sys/kernel/irq/<IRQ_NUM>/wakeup

# تفعيل/تعطيل wake على IRQ مباشرةً (للتجربة)
echo enabled  > /sys/kernel/irq/<IRQ_NUM>/wakeup
echo disabled > /sys/kernel/irq/<IRQ_NUM>/wakeup

# عرض PM runtime statistics
cat /sys/devices/<path>/power/runtime_status
cat /sys/devices/<path>/power/runtime_active_time
cat /sys/devices/<path>/power/runtime_suspended_time

# أداة pm-utils للتجربة
pm-suspend          # suspend كامل
rtcwake -m mem -s 5 # suspend واستيقظ بعد 5 ثوانٍ عبر RTC
```

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `wake irq already initialized` | استدعاء `dev_pm_set_wake_irq` مرتين على نفس الـ device | في الـ driver، أزل الاستدعاء المكرر أو تحقق بـ `dev->power.wakeirq` أولاً |
| `failed to request dedicated wake-up interrupt` | الـ `request_irq` فشل للـ dedicated wake IRQ | تحقق من صحة رقم الـ IRQ، عدم حجزه، والـ flags |
| `IRQ -EBUSY` | الـ IRQ محجوز بـ driver آخر | راجع `/proc/interrupts` وشوف مين حاجز الـ IRQ |
| `PM: Device ... failed to suspend` | فشل suspend callback للـ device | فعّل `CONFIG_PM_SLEEP_DEBUG` وشوف الـ callback المسبّب |
| `genirq: Flags mismatch irq` | تعارض في الـ IRQ flags عند الـ request | استخدم نفس الـ `IRQF_*` flags في كل `request_irq` على نفس الـ line |
| `PM: Wakeup pending, aborting suspend` | wake event وصل أثناء الـ suspend | طبيعي — لكن لو متكرر بدون سبب، تحقق من الـ spurious IRQs |
| `disable_irq called from IRQ context` | استدعاء خاطئ من داخل IRQ handler | استخدم `disable_irq_nosync` أو أنقل الـ call خارج الـ handler |
| `kernel: WARNING: CPU: ... dev_pm_attach_wake_irq` | WARN_ONCE في الـ attach path | الـ driver يستدعي `dev_pm_set_wake_irq` مرتين |
| `PM: could not update wakeup count` | race condition في الـ wakeup_count | خلل نادر — فعّل `CONFIG_LOCKDEP` |
| `irq N: nobody cared` | الـ IRQ بيحصل لكن مفيش handler | تأكد الـ `request_irq` نجح وإن الـ handler صح |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في dev_pm_set_wake_irq — تحقق من صحة الاستدعاء */
int dev_pm_set_wake_irq(struct device *dev, int irq)
{
    /* تحقق إن device_init_wakeup اتعمل قبل الاستدعاء ده */
    WARN_ON(!device_can_wakeup(dev));

    if (irq < 0)
        return -EINVAL;
    /* ... */
}

/* في suspend callback للـ driver */
static int mydrv_suspend(struct device *dev)
{
    /* تحقق إن الـ wakeirq اتسجّل فعلاً */
    WARN_ON(device_may_wakeup(dev) && !dev->power.wakeirq);
    /* ... */
}

/* في الـ IRQ handler — لو بيحصل wakeirq وهو مش متوقع */
static irqreturn_t mydrv_wake_irq_handler(int irq, void *data)
{
    struct wake_irq *wirq = data;
    /* لو الـ device مش مفروض تكون في suspend */
    WARN_ON(!(wirq->status & WAKE_IRQ_DEDICATED_MASK));
    return IRQ_HANDLED;
}
```

---

### Hardware Level

#### 1. التحقق من توافق حالة الـ Hardware مع الـ Kernel

```bash
# هل الـ IRQ line نشط فعلاً؟
cat /proc/interrupts | grep <IRQ_NUM>
# العمود الأول = عدد المرات — لو صفر = مش بيحصل

# هل الـ kernel شايف الـ wakeup source مفعّلاً؟
cat /sys/devices/<path>/power/wakeup          # يجب = "enabled"
cat /sys/devices/<path>/power/wakeup_count    # يزيد مع كل wake event

# هل الـ IRQ مسجّل كـ wake source في الـ IRQ chip؟
cat /sys/kernel/irq/<IRQ_NUM>/wakeup          # يجب = "enabled"

# مقارنة السجلات: الـ kernel يقرأ الـ HW registers؟
# استخدم devmem2 (الخطوة الجاية)
```

#### 2. Register Dump Techniques

```bash
# قراءة register معين عبر devmem2 (لازم تعرف العنوان من الـ datasheet)
# مثال: GPIO controller base address = 0x44E07000، offset لـ IRQ status = 0x2C
devmem2 0x44E0702C          # قراءة 32-bit
devmem2 0x44E0702C w 0x01   # كتابة (تصفير interrupt pending)

# عبر /dev/mem مباشرةً (لو devmem2 مش موجود)
python3 -c "
import mmap, os, struct
fd = os.open('/dev/mem', os.O_RDWR | os.O_SYNC)
mem = mmap.mmap(fd, 4096, offset=0x44E07000)
val = struct.unpack('<I', mem[0x2C:0x30])[0]
print(f'IRQ Status Register: 0x{val:08X}')
mem.close()
os.close(fd)
"

# لو الـ device على I2C/SPI — اقرأ الـ interrupt status register منه
i2cget -y <bus> <addr> <reg>    # مثال: i2cget -y 1 0x68 0x3A

# عبر io utility (من busybox أو util-linux)
io -4 -r 0x44E0702C             # قراءة 32-bit register
```

**مثال عملي لـ GPIO wake IRQ — TI AM335x:**
```bash
# GPIO0 IRQSTATUS_0 register
devmem2 0x44E07028
# لو bit معين = 1 → الـ GPIO line ده triggered interrupt
# لو البت مش اتصفّر = الـ interrupt pending وبيمنع suspend
```

#### 3. Logic Analyzer / Oscilloscope Tips

```
الـ setup المقترح:
┌─────────────┐         ┌──────────────┐
│   Device    │──IRQ───►│ GPIO Header  │──CH1──► Logic Analyzer
│  (sensor)   │         │  (test pad)  │
└─────────────┘         └──────────────┘
                                          CH2 ──► SYS_PWRDN (suspend indicator)
```

- **اربط CH1** على الـ IRQ line (WAKE_IRQ pin)
- **اربط CH2** على signal يدل على الـ suspend state (مثل: PMIC power-good، أو GPIO بيتغير في suspend callback)
- **ابحث عن:**
  - pulses على CH1 وقت ما المفروض تيجي = الـ device بتولّد events
  - CH1 مرتفع دايماً = الـ IRQ line stuck high (short/hardware fault)
  - CH1 مش بيتحرك خالص = الـ device مش بتولّد wake signal

**إعدادات مقترحة:**
```
Trigger:    Rising edge على CH1
Time/div:   10 ms (لتشخيص spurious)، 1 µs (لتشخيص glitches)
Voltage:    1.8V أو 3.3V حسب الـ IO voltage
```

#### 4. Hardware Issues الشائعة وأنماطها في الـ Kernel Log

| المشكلة | Pattern في الـ dmesg | التشخيص |
|---|---|---|
| **IRQ line stuck low** | لا يوجد أي interrupt counter يزيد في `/proc/interrupts` | قيس الـ voltage على الـ pin — يجب 0V normally |
| **IRQ line stuck high** | `irq N: nobody cared` أو suspend يُرفض فوراً | قيس الـ voltage — يجب VCC لما الـ event يحصل فقط |
| **Glitch على الـ IRQ line** | `PM: Wakeup pending` مرات كتير بدون سبب منطقي | Oscilloscope بـ trigger على edge — ابحث عن pulses قصيرة |
| **Wrong IRQ polarity** | الـ driver يشتغل بس مش بيصحى من suspend | غيّر `IRQF_TRIGGER_RISING` لـ `IRQF_TRIGGER_FALLING` أو العكس |
| **Missing pull-up/pull-down** | IRQ عشوائي — بيحصل أو مش بيحصل بدون نظام | قيس الـ voltage لما مفيش event — لازم يكون ثابت (0 أو VCC) |
| **Power domain off أثناء suspend** | wake من suspend مش بيحصل رغم الـ signal | تأكد الـ wake IRQ source في power domain شغّال أثناء suspend |
| **IRQ shared ومش مُتحكَّم فيه** | كل الـ devices المشتركة في الـ IRQ بتصحى | استخدم dedicated IRQ أو `IRQF_NO_SUSPEND` بحذر |

#### 5. Device Tree Debugging

```bash
# عرض الـ DT compiled
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | less

# تحقق من الـ interrupt spec للـ device
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "your-device"

# شوف الـ interrupts property مباشرةً
hexdump -C /sys/firmware/devicetree/base/soc/your-device/interrupts
cat /sys/firmware/devicetree/base/soc/your-device/interrupt-parent

# لو بتستخدم wakeup-source property في DT
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -B5 "wakeup-source"
```

**مثال DT صح لـ dedicated wake IRQ:**
```dts
&i2c1 {
    sensor@68 {
        compatible = "vendor,my-sensor";
        reg = <0x68>;

        /* الـ IO interrupt العادي */
        interrupts = <4 IRQ_TYPE_LEVEL_LOW>;
        interrupt-parent = <&gpio1>;

        /* الـ dedicated wake IRQ — IRQ منفصل للصحيان */
        wakeup-source;
        interrupt-names = "irq", "wakeup";
        interrupts = <4 IRQ_TYPE_LEVEL_LOW>,
                     <5 IRQ_TYPE_EDGE_RISING>;
    };
};
```

**تحقق من التطابق بين DT والـ hardware:**
```bash
# IRQ number في الـ kernel بعد parsing الـ DT
dmesg | grep "irq.*gpio\|gpio.*irq"

# تحقق إن الـ IRQ number المحجوز يطابق ما في DT
cat /proc/interrupts | grep sensor
# رقم الـ IRQ هنا لازم يطابق ما رسمته في DT
```

---

### Practical Commands

#### جاهزة للـ Copy-Paste

**الخطوة 1: تشخيص سريع للـ wake IRQ**
```bash
DEVICE_PATH="/sys/devices/platform/soc/i2c1/i2c-1/1-0068"  # غيّر حسب device-ك

echo "=== Wakeup capability ==="
cat ${DEVICE_PATH}/power/wakeup

echo "=== Wake event stats ==="
for f in wakeup_count wakeup_active_count wakeup_total_time_ms wakeup_last_time_ms wakeup_abort_count; do
    printf "%-35s: %s\n" "$f" "$(cat ${DEVICE_PATH}/power/$f 2>/dev/null || echo N/A)"
done

echo "=== Active wakeup sources ==="
awk '$2 > 0 || $3 > 0' /sys/kernel/debug/wakeup_sources 2>/dev/null | column -t
```

**الخطوة 2: تعقّب الـ IRQ في الـ suspend path**
```bash
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo "power:suspend_resume power:device_pm_callback_start power:device_pm_callback_end" > set_event
echo "irq:irq_handler_entry irq:irq_handler_exit" >> set_event
echo 1 > tracing_on

# في terminal تاني
rtcwake -m mem -s 10

cat /sys/kernel/debug/tracing/trace | grep -E "(suspend|wake|irq)" | head -50
```

**مثال output ومعناه:**
```
# TASK-PID    CPU#  ||||  TIMESTAMP  FUNCTION
#                   ||||
   systemd-1  [000] .... 12345.678: suspend_resume: suspend_enter[1] calling
   kworker    [001] .N.. 12345.690: device_pm_callback_start: i2c-sensor, noirq_suspend
   kworker    [001] .... 12345.695: device_pm_callback_end:   i2c-sensor, noirq_suspend, err=0
   <idle>     [000] dN.. 12346.100: irq_handler_entry: irq=42 name=gpio-wakeup
   <idle>     [000] dN.. 12346.101: irq_handler_exit:  irq=42 ret=handled
   systemd-1  [000] .... 12346.105: suspend_resume: suspend_enter[1] end
```

- `irq_handler_entry` بـ irq=42 أثناء suspend = الـ wake IRQ اشتغل تمام
- لو مفيش `irq_handler_entry` = الـ IRQ مش بيوصل للـ kernel

**الخطوة 3: تحقق شامل من IRQ registration**
```bash
IRQ=42  # غيّر للـ IRQ number بتاعك

echo "=== IRQ $IRQ details ==="
cat /sys/kernel/irq/$IRQ/actions     # اسم الـ handler
cat /sys/kernel/irq/$IRQ/chip_name   # اسم الـ IRQ chip
cat /sys/kernel/irq/$IRQ/type        # edge/level
cat /sys/kernel/irq/$IRQ/wakeup      # enabled/disabled
cat /sys/kernel/irq/$IRQ/hwirq       # hardware IRQ number

echo "=== IRQ $IRQ counts per CPU ==="
grep "^ *$IRQ:" /proc/interrupts
```

**الخطوة 4: اختبار wake من suspend بـ RTC**
```bash
# اصحى بعد 30 ثانية ولاحظ هل wake device اشتغلت
dmesg -c > /dev/null  # امسح القديم
rtcwake -m mem -s 30
dmesg | grep -iE "(wake|irq|pm:|suspend|resume)" | head -30
```

**الخطوة 5: تحقق من الـ Device Tree المحمّل**
```bash
# ابحث عن الـ device بتاعك في الـ DT المحمّل
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    awk '/your-device/,/};/' | \
    grep -E "(interrupt|wakeup|status)"

# تحقق إن wakeup-source property موجودة
find /sys/firmware/devicetree/base -name "wakeup-source" 2>/dev/null
```

**الخطوة 6: تشخيص spurious wakeups**
```bash
# شغّل الأمر ده قبل الـ suspend — بيعدّ الـ wakeups
BEFORE=$(cat /sys/power/wakeup_count)
echo "Wakeup count before: $BEFORE"

# suspend
rtcwake -m mem -s 60 &
sleep 5

AFTER=$(cat /sys/power/wakeup_count)
echo "Wakeup count after: $AFTER"
echo "Wakeups that aborted suspend: $((AFTER - BEFORE))"

# شوف مين السبب
cat /sys/kernel/debug/wakeup_sources | sort -k4 -rn | head -10
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على AM62x — الـ UART مش بيصحّي الـ System

#### العنوان
**Wake-on-UART مش شغال على AM62x industrial gateway**

#### السياق
شركة بتبني industrial IoT gateway بـ TI AM62x SoC. الـ gateway بتدخل في **suspend** بعد فترة idle عشان توفّر طاقة. المفروض إن أي بيانات جاية على UART3 تصحّي الـ system. المنتج في production والعميل بيشكي إن الـ gateway بتتجمّد ومش بترد على الـ sensors.

#### المشكلة
الـ UART driver بيستخدم `dev_pm_set_wake_irq()` بس الـ SoC دي عندها **dedicated wakeup interrupt** مفصول عن الـ UART data interrupt. الـ developer استخدم الـ function الغلط فـ الـ system مش بتصحى لما UART data تيجي أثناء suspend.

#### التحليل
**الـ `pm_wakeirq.h`** بيعرّف الفرق بين نوعين:

```c
/* يستخدم نفس الـ IRQ اللي بيشتغل الـ device عادةً */
extern int dev_pm_set_wake_irq(struct device *dev, int irq);

/* يستخدم IRQ مخصص للـ wakeup بس — مفصول عن الـ data IRQ */
extern int dev_pm_set_dedicated_wake_irq(struct device *dev, int irq);
```

على AM62x، الـ UART3 عنده:
- `IRQ 123` → الـ data/error interrupt العادي (بيتعطّل وقت suspend)
- `IRQ 87`  → الـ dedicated wakeup interrupt من الـ wakeup controller

الـ developer كتب:
```c
/* WRONG: استخدم الـ data IRQ مش الـ wakeup IRQ */
dev_pm_set_wake_irq(&pdev->dev, uart_irq); /* IRQ 123 */
```

لما الـ system يعمل suspend، الـ kernel بيعمل `disable_irq()` على الـ IRQ العادي (123) لأنه مش **dedicated**، فـ الـ UART مش بيقدر يصحّي أي حاجة.

#### الحل

**أولاً: Device Tree**
```dts
/* am62x-gateway.dts */
&uart3 {
    status = "okay";
    /* الـ wakeup-source property بتفعّل الـ wakeup capability */
    wakeup-source;
    interrupts = <GIC_SPI 123 IRQ_TYPE_LEVEL_HIGH>;
    /* IRQ مخصص للـ wakeup من الـ wakeup controller */
    interrupt-names = "wakeup";
    wakeups = <&wkup_ctrl 87>;
};
```

**ثانياً: Driver**
```c
static int am62x_uart_probe(struct platform_device *pdev)
{
    int data_irq, wakeup_irq;

    data_irq   = platform_get_irq(pdev, 0);
    wakeup_irq = platform_get_irq_byname(pdev, "wakeup");

    /* صح: استخدم الـ dedicated wakeup IRQ */
    ret = dev_pm_set_dedicated_wake_irq(&pdev->dev, wakeup_irq);
    if (ret)
        dev_err(&pdev->dev, "failed to set dedicated wake IRQ: %d\n", ret);

    /* باقي الـ initialization */
}
```

**ثالثاً: تحقق**
```bash
# شوف الـ wake IRQ المسجّل
cat /sys/bus/platform/devices/2810000.uart/power/wakeup_irq

# اختبر الـ suspend/wake cycle
echo mem > /sys/power/state &
sleep 2
# أبعت بيانات على UART من device تاني
echo "wake" > /dev/ttyUSB0
# المفروض الـ system يصحّى
```

#### الدرس المستفاد
`dev_pm_set_wake_irq()` بيعمل الـ IRQ المعطى **wakeup-capable** لكن الـ PM core ممكن يعمله `disable` أثناء suspend لو الـ driver مش عارف ده dedicated. `dev_pm_set_dedicated_wake_irq()` بيخلّي الـ kernel يعرف إن الـ IRQ ده **لازم يفضل enabled** حتى أثناء suspend عشان يعمل wakeup.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ USB مش بيصحّي من الـ Remote

#### العنوان
**USB IR receiver مش بيصحّي الـ TV box من suspend**

#### السياق
TV box رخيصة بـ Allwinner H616 بتشغّل Android. المستخدم بيضغط على الـ remote (USB IR receiver) والـ box مش بتصحّى. الـ box بتصحّى بس لو ضغط على الـ power button المادي.

#### المشكلة
الـ USB IR receiver driver بيستدعي `dev_pm_set_wake_irq()` بعد ما الـ device اتـ enumerate، بس بيعمله بـ IRQ رقم غلط — بياخد الـ IRQ من الـ USB host controller مش من الـ device نفسه.

#### التحليل
الـ flow في `pm_wakeirq.h`:

```c
/*
 * dev_pm_set_wake_irq() → بتربط الـ IRQ بالـ device's wakeup_source
 * لو الـ IRQ غلط → الـ wakeup_source مش هيتـ trigger
 */
extern int dev_pm_set_wake_irq(struct device *dev, int irq);
```

الـ driver الغلط:
```c
static int ir_usb_probe(struct usb_interface *intf, ...)
{
    struct usb_device *udev = interface_to_usbdev(intf);

    /* WRONG: ده IRQ الـ USB host controller مش الـ IR device */
    int hcd_irq = udev->bus->controller->irq; /* IRQ 45 - HCD */
    dev_pm_set_wake_irq(&intf->dev, hcd_irq);
}
```

الـ `dev_pm_set_wake_irq()` بتربط IRQ 45 بـ `intf->dev`، بس لما الـ USB remote يبعت interrupt، الـ signal بييجي على USB endpoint interrupt مش على الـ HCD IRQ مباشرة خلال suspend. على H616، الـ wakeup من USB بيتم عبر **dedicated wakeup line** من الـ USB PHY.

#### الحل

**Device Tree للـ H616:**
```dts
&usb_phy0 {
    /* الـ H616 عنده dedicated wakeup IRQ للـ USB */
    wakeup-source;
    interrupts = <GIC_SPI 105 IRQ_TYPE_LEVEL_HIGH>, /* normal */
                 <GIC_SPI 106 IRQ_TYPE_LEVEL_HIGH>; /* wakeup */
    interrupt-names = "usb", "wakeup";
};
```

**Driver الصح:**
```c
static int ir_usb_probe(struct usb_interface *intf,
                         const struct usb_device_id *id)
{
    struct usb_device *udev = interface_to_usbdev(intf);
    struct device *hcd_dev = udev->bus->controller;
    int wakeup_irq;

    /* جيب الـ dedicated wakeup IRQ من الـ PHY/HCD device */
    wakeup_irq = of_irq_get_byname(hcd_dev->of_node, "wakeup");
    if (wakeup_irq < 0) {
        dev_warn(&intf->dev, "no dedicated wakeup IRQ, using HCD IRQ\n");
        /* fallback: استخدم الـ reverse variant */
        dev_pm_set_dedicated_wake_irq_reverse(&intf->dev,
                                               hcd_dev->driver->irq);
    } else {
        dev_pm_set_dedicated_wake_irq(&intf->dev, wakeup_irq);
    }

    device_init_wakeup(&intf->dev, true);
    return 0;
}

static void ir_usb_disconnect(struct usb_interface *intf)
{
    /* لازم تمسح الـ wake IRQ قبل ما الـ device يتشال */
    dev_pm_clear_wake_irq(&intf->dev);
    device_init_wakeup(&intf->dev, false);
}
```

**تحقق:**
```bash
# شوف الـ wakeup sources المفعّلة
cat /sys/kernel/debug/wakeup_sources

# شوف الـ IRQ المربوط بالـ device
cat /sys/bus/usb/devices/1-1/power/wakeup
# المفروض يطلع: enabled

# فحص الـ IRQ statistics
cat /proc/interrupts | grep -i usb
```

#### الدرس المستفاد
`dev_pm_set_wake_irq()` و `dev_pm_set_dedicated_wake_irq()` الفرق الحقيقي: الأول بيفترض إن الـ IRQ هو نفسه الـ runtime IRQ وممكن يتعطّل، التاني بيضمن إن الـ kernel هيفضّله enabled أثناء suspend. لو IRQ hierarchy معقّدة (زي USB)، ممكن تحتاج `_reverse` variant اللي بيعكس الـ enable/disable order.

---

### السيناريو 3: STM32MP1 — I2C Touchscreen مش بيصحّي الـ Industrial HMI

#### العنوان
**الـ devm_pm_set_wake_irq() بيتسبّب في memory leak أثناء driver unbind/rebind**

#### السياق
industrial HMI panel بـ STM32MP1 بيستخدم I2C touchscreen (FT5336). الـ team بيعمل driver hot-reload أثناء field update. بعد كذا unbind/bind cycle، الـ system بيبدأ يتصرف غريب والـ wakeup مش شغال صح.

#### المشكلة
الـ developer استخدم `dev_pm_set_wake_irq()` (مش الـ `devm_` version) ونسي يعمل `dev_pm_clear_wake_irq()` في الـ `remove()`. الـ `struct wake_irq` بيتـ allocate من جوّا الـ kernel بس مش بيتحرر.

#### التحليل
من `pm_wakeirq.h` في الـ header:

```c
/* النسخة العادية — أنت مسؤول عن الـ cleanup */
extern int dev_pm_set_wake_irq(struct device *dev, int irq);
extern void dev_pm_clear_wake_irq(struct device *dev);

/* النسخة الـ devm — الـ kernel بيعمل cleanup تلقائياً */
extern int devm_pm_set_wake_irq(struct device *dev, int irq);
```

الـ driver الغلط:
```c
static int ft5336_probe(struct i2c_client *client,
                         const struct i2c_device_id *id)
{
    int irq = client->irq;

    /* بيستخدم النسخة العادية مش الـ devm */
    dev_pm_set_wake_irq(&client->dev, irq);
    device_init_wakeup(&client->dev, true);
    return 0;
}

static int ft5336_remove(struct i2c_client *client)
{
    /* نسي dev_pm_clear_wake_irq() ! */
    device_init_wakeup(&client->dev, false);
    return 0;
}
```

كل مرة `probe` بتتعمل، الـ kernel بيـ allocate `struct wake_irq` جديد. الـ pointer القديم بيتـ overwrite بدون ما الـ memory القديمة تتحرر. بعد كذا cycle، الـ `dev->power.wakeirq` بيبقى stale pointer لـ memory قديمة.

#### الحل

**الحل الأمثل — استخدم `devm_pm_set_wake_irq()`:**
```c
static int ft5336_probe(struct i2c_client *client,
                         const struct i2c_device_id *id)
{
    int ret;

    /* devm version بتعمل cleanup تلقائياً لما الـ device يتشال */
    ret = devm_pm_set_wake_irq(&client->dev, client->irq);
    if (ret) {
        dev_err(&client->dev, "failed to set wake IRQ: %d\n", ret);
        return ret;
    }

    device_init_wakeup(&client->dev, true);
    return 0;
}

static int ft5336_remove(struct i2c_client *client)
{
    /* مش محتاج dev_pm_clear_wake_irq() — devm بتتكلم بنفسها */
    device_init_wakeup(&client->dev, false);
    return 0;
}
```

**لو محتاج النسخة العادية لأسباب معيّنة:**
```c
static int ft5336_remove(struct i2c_client *client)
{
    /* لازم تعمل clear قبل disable wakeup */
    dev_pm_clear_wake_irq(&client->dev);
    device_init_wakeup(&client->dev, false);
    return 0;
}
```

**تحقق بعد الإصلاح:**
```bash
# شوف الـ memory usage قبل وبعد unbind/bind cycles
for i in $(seq 1 10); do
    echo "1-0038" > /sys/bus/i2c/drivers/ft5336/unbind
    echo "1-0038" > /sys/bus/i2c/drivers/ft5336/bind
done

# فحص الـ slab memory
cat /proc/slabinfo | grep wake

# تأكد إن الـ wakeup لسه شغال
echo mem > /sys/power/state &
sleep 1
# العب على الـ touchscreen
```

#### الدرس المستفاد
`devm_pm_set_wake_irq()` موجودة في الـ API تحديداً عشان حالات زي دي. قاعدة: لو الـ driver بيدعم hot-plug أو unbind/bind، دايماً استخدم الـ `devm_` variants. لو اضطريت تستخدم `dev_pm_set_wake_irq()`، اتأكد إن `dev_pm_clear_wake_irq()` موجودة في كل exit path في `remove()`.

---

### السيناريو 4: i.MX8M Plus — الـ SPI Accelerometer بيكسّر الـ Suspend على Automotive ECU

#### العنوان
**`dev_pm_set_dedicated_wake_irq_reverse()` مطلوب للـ SPI chain على i.MX8M Plus**

#### السياق
automotive ECU بـ NXP i.MX8M Plus. الـ ECU بيحتوي على IMU (accelerometer/gyro) متوصّل بـ SPI وبيستخدم interrupt line منفصلة للـ wakeup (الـ INT1 pin للـ IMU). المشكلة إن الـ system بيعمل suspend بشكل صح بس بيتعلّق أثناء الـ resume بسبب interrupt storm.

#### المشكلة
الـ IMU بيبعت burst of interrupts لما يـ wake up. الـ `dev_pm_set_dedicated_wake_irq()` بيعمل الـ wakeup IRQ enabled **قبل** الـ parent SPI controller يصحّى، فـ الـ interrupt بييجي والـ SPI master مش ready بعد.

#### التحليل
الـ header بيوفّر تلات variants:

```c
/* 1. نفس الـ runtime IRQ — مش dedicated */
extern int dev_pm_set_wake_irq(struct device *dev, int irq);

/* 2. dedicated: بيـ enable الـ wakeup IRQ قبل ما parents يصحّوا */
extern int dev_pm_set_dedicated_wake_irq(struct device *dev, int irq);

/* 3. dedicated_reverse: بيـ enable الـ wakeup IRQ بعد ما parents يصحّوا */
extern int dev_pm_set_dedicated_wake_irq_reverse(struct device *dev, int irq);
```

الفرق الجوهري:
- `dev_pm_set_dedicated_wake_irq()` → IRQ بيتـ enable في **beginning of resume** (قبل parent devices)
- `dev_pm_set_dedicated_wake_irq_reverse()` → IRQ بيتـ enable في **end of resume** (بعد parent devices)

على SPI device زي IMU، الـ parent SPI controller لازم يصحّى الأول. لو الـ IMU بعت interrupt وانتا في منتصف الـ SPI controller resume، النتيجة interrupt storm أو kernel panic.

#### الحل

**Device Tree:**
```dts
/* imx8mp-ecu.dts */
&spi2 {
    status = "okay";

    imu@0 {
        compatible = "bosch,bmi088";
        reg = <0>;
        spi-max-frequency = <10000000>;

        /* الـ INT1 pin: data interrupt */
        interrupt-parent = <&gpio3>;
        interrupts = <14 IRQ_TYPE_EDGE_RISING>;

        /* الـ INT2 pin: dedicated wakeup */
        wakeup-source;
        wakeup-gpios = <&gpio3 15 GPIO_ACTIVE_HIGH>;
    };
};
```

**Driver:**
```c
static int bmi088_probe(struct spi_device *spi)
{
    struct bmi088_data *data;
    int data_irq, wakeup_irq;

    data_irq   = spi->irq; /* INT1 */
    wakeup_irq = of_irq_get_byname(spi->dev.of_node, "wakeup"); /* INT2 */

    /*
     * نستخدم _reverse لأن الـ SPI controller (parent) لازم يصحّى
     * قبل ما الـ IMU يبدأ يبعت interrupts
     */
    ret = dev_pm_set_dedicated_wake_irq_reverse(&spi->dev, wakeup_irq);
    if (ret) {
        dev_err(&spi->dev, "failed to set wake IRQ: %d\n", ret);
        return ret;
    }

    device_init_wakeup(&spi->dev, true);
    return 0;
}
```

**تحقق:**
```bash
# فحص الـ resume timing
echo 1 > /sys/power/pm_debug_messages

# شوف الـ suspend/resume log
dmesg | grep -E "(PM:|wake_irq|bmi088)"

# تأكد من الـ IRQ order أثناء resume
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
```

#### الدرس المستفاد
الـ `_reverse` suffix مش تفصيلة صغيرة — دي بتحدد متى بالظبط الـ wakeup IRQ يتـ enable أثناء الـ resume path. للـ devices اللي هي children في bus hierarchy (SPI, I2C)، `dev_pm_set_dedicated_wake_irq_reverse()` هو الأأمن لأنه بيضمن إن الـ parent controller صاحي وجاهز قبل ما الـ child يبدأ يتكلم.

---

### السيناريو 5: RK3562 — Custom Board Bring-up، الـ HDMI CEC مش بيصحّي الـ Display

#### العنوان
**Board bring-up: HDMI CEC wakeup مش شغال على RK3562 بسبب missing `dev_pm_clear_wake_irq()`**

#### السياق
bring-up لـ custom board بـ Rockchip RK3562 للـ digital signage. الـ display مفروض يصحّى لما الـ HDMI CEC يستقبل أمر power-on من التليفزيون. الـ engineer يلاقي إن الـ first boot شغال تمام، بس بعد kernel module reload أو driver reset، الـ CEC wakeup بيوقف.

#### المشكلة
الـ HDMI CEC driver بيستدعي `dev_pm_set_dedicated_wake_irq()` في الـ `probe()` بس مش بيستدعي `dev_pm_clear_wake_irq()` في الـ error path داخل `probe()` نفسه. لما `probe()` بيفشل في خطوة لاحقة وبيرجع error، الـ wake IRQ بيفضل مسجّل لـ device موجودة بشكل ناقص.

#### التحليل
الـ buggy probe flow:

```c
static int rk3562_cec_probe(struct platform_device *pdev)
{
    int irq, ret;

    irq = platform_get_irq(pdev, 0);

    /* Step 1: set wake IRQ — نجح */
    ret = dev_pm_set_dedicated_wake_irq(&pdev->dev, irq);
    if (ret)
        return ret;

    /* Step 2: allocate resources */
    priv->clk = devm_clk_get(&pdev->dev, "cec");
    if (IS_ERR(priv->clk)) {
        /* BUG: رجع error بدون ما يعمل clear للـ wake IRQ */
        return PTR_ERR(priv->clk); /* ← wake IRQ لسه مسجّل! */
    }

    /* Step 3: register CEC adapter */
    ret = cec_register_adapter(priv->adap, &pdev->dev);
    if (ret) {
        /* BUG: نفس المشكلة */
        return ret;
    }

    return 0;
}
```

لما الـ module يتـ reload، الـ kernel يحاول يسجّل wake IRQ جديد على device عندها wake IRQ قديم. الـ behavior هنا بيعتمد على الـ kernel version — ممكن يـ override أو يـ fail أو يدي stale state.

#### الحل

**طريقة 1 — استخدم `devm_pm_set_wake_irq()` (أفضل):**
```c
static int rk3562_cec_probe(struct platform_device *pdev)
{
    int irq, ret;

    irq = platform_get_irq(pdev, 0);

    /*
     * devm version بتتأكد إن الـ cleanup بيحصل دايماً
     * سواء probe نجحت أو فشلت
     */
    ret = devm_pm_set_wake_irq(&pdev->dev, irq);
    if (ret)
        return ret;

    priv->clk = devm_clk_get(&pdev->dev, "cec");
    if (IS_ERR(priv->clk))
        return PTR_ERR(priv->clk); /* devm هتعمل cleanup تلقائياً */

    device_init_wakeup(&pdev->dev, true);
    return 0;
}
```

**طريقة 2 — لو لازم تستخدم النسخة العادية:**
```c
static int rk3562_cec_probe(struct platform_device *pdev)
{
    int irq, ret;

    irq = platform_get_irq(pdev, 0);

    ret = dev_pm_set_dedicated_wake_irq(&pdev->dev, irq);
    if (ret)
        return ret;

    priv->clk = devm_clk_get(&pdev->dev, "cec");
    if (IS_ERR(priv->clk)) {
        ret = PTR_ERR(priv->clk);
        goto err_clear_wake;  /* cleanup صريح */
    }

    ret = cec_register_adapter(priv->adap, &pdev->dev);
    if (ret)
        goto err_clear_wake;

    device_init_wakeup(&pdev->dev, true);
    return 0;

err_clear_wake:
    dev_pm_clear_wake_irq(&pdev->dev);
    return ret;
}

static int rk3562_cec_remove(struct platform_device *pdev)
{
    dev_pm_clear_wake_irq(&pdev->dev);
    device_init_wakeup(&pdev->dev, false);
    cec_unregister_adapter(priv->adap);
    return 0;
}
```

**تحقق أثناء bring-up:**
```bash
# فحص حالة الـ wake IRQ قبل وبعد module reload
cat /sys/bus/platform/devices/fdba0000.cec/power/wakeup_irq

# reload الـ module
modprobe -r rk3562_hdmi_cec
modprobe rk3562_hdmi_cec

# تأكد إن الـ wakeup IRQ اتسجّل صح
cat /sys/bus/platform/devices/fdba0000.cec/power/wakeup_irq

# اختبار end-to-end
echo mem > /sys/power/state &
# أبعت CEC power-on command من device تاني
cec-ctl --to 0 --image-view-on
# المفروض الـ display يصحّى
```

#### الدرس المستفاد
الـ `dev_pm_clear_wake_irq()` مش بس للـ `remove()` — لازم تكون في كل **error path** داخل الـ `probe()` لو استخدمت النسخة غير الـ `devm`. أحسن ممارسة في الـ bring-up: استخدم دايماً `devm_pm_set_wake_irq()` لأنها أكثر أماناً في أي سيناريو. لما تشوف wakeup بيشتغل على أول boot بس مش بعد reload، أول حاجة افتكرها: missing `dev_pm_clear_wake_irq()` في error path أو `remove()`.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الصلة |
|--------|-------|
| [genirq / PM: Wakeup interrupts handling rework (related to suspend-to-idle)](https://lwn.net/Articles/610109/) | إعادة هيكلة آلية الـ wakeup IRQs عند الـ suspend-to-idle، الأساس النظري لـ `pm_wakeirq` |
| [PM: Avoid losing wakeup events during suspend](https://lwn.net/Articles/392897/) | **wakeup_count** ومنع ضياع الأحداث — يفهم منه ليش الـ wake IRQ لازم يكون موثوق |
| [PM / Wakeup: Introduce wakeup source objects and event statistics (v2)](https://lwn.net/Articles/405108/) | البنية التحتية لـ wakeup sources اللي بتستخدمها `dev_pm_set_wake_irq` تحت الغطاء |
| [PCI: Add support for PCIe WAKE# interrupt](https://lwn.net/Articles/1032187/) | مثال حديث على استخدام `dev_pm_set_dedicated_wake_irq` في PCIe — يوضح الـ use case الواقعي |

---

### التوثيق الرسمي للـ kernel

```
Documentation/power/suspend-and-interrupts.rst
Documentation/power/runtime_pm.rst
Documentation/driver-api/pm/devices.rst
Documentation/ABI/testing/sysfs-devices-power
```

- **[System Suspend and Device Interrupts](https://www.kernel.org/doc/html/latest/power/suspend-and-interrupts.html)** — يشرح `enable_irq_wake()` و`suspend_device_irqs()` اللي بتتعامل معاهم `pm_wakeirq` في الخلفية.
- **[Runtime Power Management Framework for I/O Devices](https://www.kernel.org/doc/html/v6.13-rc1/power/runtime_pm.html)** — الـ runtime PM lifecycle اللي فيه بتُفعَّل وتُعطَّل الـ wake IRQs تلقائيًا.
- **[Device Power Management Basics](https://www.kernel.org/doc/html/v4.19/driver-api/pm/devices.html)** — المفاهيم الأساسية للـ device PM في الـ kernel.
- **[Linux generic IRQ handling](https://docs.kernel.org/core-api/genericirq.html)** — بنية الـ IRQ framework اللي عليها بيتبنى `pm_wakeirq`.

---

### Commits مهمة في الـ kernel

| الـ Commit / الـ Patch | الوصف |
|------------------------|-------|
| [`4990d4fe327b`](https://patchwork.kernel.org/project/linux-omap/patch/1431560196-5722-3-git-send-email-tony@atomide.com/) | **PM / Wakeirq: Add automated device wake IRQ handling** — الـ commit الأصلي اللي أنشأ `drivers/base/power/wakeirq.c` و`include/linux/pm_wakeirq.h` (Tony Lindgren، 2015) |
| [PM / wakeirq: Fix unbalanced irq enable for wakeirq](https://patchwork.kernel.org/project/linux-pm/patch/20180209161126.18580-1-tony@atomide.com/) | إصلاح خلل الـ unbalanced enable/disable عند الـ runtime suspend — درس مهم في تصميم الـ state machine |
| [PATCH v4: support enabling wake-up irq after runtime_suspend called](https://lore.kernel.org/linux-arm-kernel/20211025070155.2995-2-chunfeng.yun@mediatek.com/T/) | إضافة دعم تفعيل الـ wake IRQ بعد `runtime_suspend` — الأساس لـ `dev_pm_set_dedicated_wake_irq_reverse` |

لعرض كامل تاريخ الملف في الـ GitHub:
```
https://github.com/torvalds/linux/commits/master/drivers/base/power/wakeirq.c
https://github.com/torvalds/linux/commits/master/include/linux/pm_wakeirq.h
```

---

### نقاشات الـ mailing list

- **[drivers/power/wakeirq: Call device_init_wakeup from dev_pm_set_wake_irq](https://linux.kernel.narkive.com/M4I5IpQH/patch-drivers-power-wakeirq-call-device-init-wakeup-from-dev-pm-set-wake-irq)** — نقاش عن تبسيط الـ API بدمج `device_init_wakeup()` داخل `dev_pm_set_wake_irq()`.
- **[PM / wakeirq: Fix unbalanced irq enable — LKML](https://lists.openwall.net/linux-kernel/2018/02/09/607)** — تشخيص خلل حقيقي في الـ wakeirq مع حل Tony Lindgren.
- **[HID: i2c-hid: Use PM subsystem to manage wake irq](https://patchwork.kernel.org/project/linux-input/patch/20220830171332.2.Id4b4bdfe06e2caf2d5a3c9dd4a9b1080c38b539c@changeid/)** — مثال عملي على تحويل درايفر حقيقي لاستخدام `pm_wakeirq` API بدل الكود اليدوي.
- **[elan_i2c: Use PM subsystem to manage wake irq](https://patchew.org/linux/20220830231541.1135813-1-rrangel@chromium.org/20220830171332.1.Id022caf53d01112188308520915798f08a33cd3e@changeid/)** — نفس الفكرة في درايفر touchpad.
- **[Re: PM runtime: enable wake irq after runtime_suspend hook](https://lore.kernel.org/all/YG/o1ERNkcaYAV9y@atomide.com/)** — نقاش مع Tony Lindgren عن الـ ordering بين الـ runtime_suspend وتفعيل الـ wake IRQ.

---

### كتب مرجعية

#### Linux Device Drivers (LDD3)
- **Corbet, Rubini, Kroah-Hartman** — الفصل 10 (*Interrupt Handling*) و الفصل 14 (*The Linux Device Model*).
- متاح مجانًا: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)
- الـ `pm_wakeirq` ظهر بعد إصدار LDD3، بس الفهم الأساسي للـ IRQs و device model ضروري.

#### Linux Kernel Development — Robert Love (3rd Edition)
- الفصل 7 (*Interrupts and Interrupt Handlers*) — يشرح `request_irq()` و `free_irq()` اللي بتعتمد عليهم `pm_wakeirq`.
- الفصل 14 (*The Block I/O Layer*) و الفصل 17 (*Devices and Modules*) — context للـ device lifecycle.

#### Embedded Linux Primer — Christopher Hallinan
- الفصل 15 (*Embedded Linux Power Management*) — مباشر في الموضوع: suspend/resume، wakeup sources، و GPIO wake IRQs.

---

### مصادر kernelnewbies.org و elinux.org

- **[Linux 4.17 - Kernel Newbies](https://kernelnewbies.org/Linux_4.17)** — فيه إضافة `wakeup sysfs node` لعرض حالة الـ IRQ wakeup عبر `/sys/kernel/irq/<n>/wakeup`.
- **[IRQ Handling Subsystem — Kernel Newbies](https://kernelnewbies.org/New_Kernel_Hacking_HOWTO/Subsystems/Exceptions_and_Interrupts_Handling)** — مدخل لفهم الـ IRQ subsystem.
- **[OMAP Power Management — eLinux.org](https://elinux.org/OMAP_Power_Management)** — مثال embedded حقيقي على تهيئة GPIO wakeup IRQs على OMAP، المنصة اللي منها انطلق كود `pm_wakeirq` أصلًا.

---

### مصادر إضافية

- **[Power management/Wakeup triggers — ArchWiki](https://wiki.archlinux.org/title/Power_management/Wakeup_triggers)** — شرح عملي لـ `/sys/devices/.../power/wakeup` و كيف يتحكم فيها الـ driver.
- **[TI AM62x SDK — Wakeup Sources](https://software-dl.ti.com/processor-sdk-linux/esd/AM62X/09_00_00_03/exports/docs/linux/Foundational_Components/Kernel/Kernel_Drivers/Power_Management/pm_wakeup_sources.html)** — توثيق Embedded حقيقي على معالج TI لـ wakeup sources.

---

### مصطلحات البحث للمزيد

```
"pm_wakeirq" site:lore.kernel.org
"dev_pm_set_dedicated_wake_irq" driver example
linux kernel "wakeirq" "runtime_suspend"
enable_irq_wake irq_set_irq_wake difference
linux pm wakeup source vs wake irq
"IRQF_NO_SUSPEND" vs "enable_irq_wake"
linux kernel wakeirq unbalanced enable disable
pm_wakeirq.h device driver implementation
```
## Phase 8: Writing simple module

### الفكرة

**`dev_pm_set_wake_irq`** هي أنسب function نعمل عليها kprobe — بتتسمى لما driver بيسجّل IRQ كـ wakeup source لـ device. ده بيخلينا نعرف مين بيفعّل wakeup capability ونطبع اسم الـ device ورقم الـ IRQ.

---

### الـ Module كامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * wakeirq_probe.c
 * Hooks dev_pm_set_wake_irq via kprobe to log
 * every device that registers a wakeup IRQ.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit  */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe    */
#include <linux/device.h>      /* struct device, dev_name()         */
#include <linux/kernel.h>      /* pr_info                           */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on dev_pm_set_wake_irq to trace wakeup IRQ registration");

/* ------------------------------------------------------------------ */
/* kprobe pre-handler: runs just BEFORE dev_pm_set_wake_irq executes  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * x86-64 calling convention:
     *   arg0 (struct device *dev) → rdi
     *   arg1 (int irq)            → rsi
     *
     * regs_get_kernel_argument() abstracts the ABI across arches.
     */
    struct device *dev = (struct device *)regs_get_kernel_argument(regs, 0);
    int irq            = (int)regs_get_kernel_argument(regs, 1);

    if (dev)
        pr_info("wakeirq_probe: device '%s' registering wakeup IRQ %d\n",
                dev_name(dev), irq);
    else
        pr_info("wakeirq_probe: NULL device, IRQ %d\n", irq);

    return 0; /* 0 = let the original function continue normally */
}

/* ------------------------------------------------------------------ */
/* kprobe definition                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "dev_pm_set_wake_irq", /* function we want to hook */
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init wakeirq_probe_init(void)
{
    int ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("wakeirq_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("wakeirq_probe: planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit wakeirq_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("wakeirq_probe: kprobe removed\n");
}

module_init(wakeirq_probe_init);
module_exit(wakeirq_probe_exit);
```

---

### Makefile

```makefile
obj-m += wakeirq_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | ليه موجود |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kprobes.h` | `struct kprobe` + `register_kprobe` / `unregister_kprobe` |
| `linux/device.h` | `struct device` + `dev_name()` عشان نطبع اسم الـ device |
| `linux/kernel.h` | `pr_info` / `pr_err` |

---

#### الـ `handler_pre`

الـ handler بيتشغّل **قبل** ما `dev_pm_set_wake_irq` تبدأ، فبنقدر نشوف الـ arguments زي ما الـ driver بعتهم.
**الـ `regs_get_kernel_argument`** بتجرّد الـ ABI الخاص بكل architecture عشان الكود يشتغل على x86 و ARM بنفس الطريقة من غير `#ifdef`.

---

#### الـ `struct kprobe`

- **`symbol_name`**: بيقول لـ kprobe framework على أي function يحط الـ breakpoint — الـ kernel بيحوّل الاسم لـ address وقت الـ register.
- **`pre_handler`**: الـ callback اللي بيتنفّذ قبل الـ instruction الأولى في الـ function.

---

#### الـ `module_init`

`register_kprobe` بتحجز الـ breakpoint في الـ kernel وتربط الـ handler بيه.
لو فشلت (مثلاً الـ symbol مش exported أو الـ kernel compiled بدون `CONFIG_KPROBES`) بنرجع الـ error فوراً ومبنكملش الـ init.

---

#### الـ `module_exit`

**لازم** نعمل `unregister_kprobe` في الـ exit عشان الـ kernel يشيل الـ breakpoint ويرجع الـ instruction الأصلية — لو متشلتش هيبقى في dangling pointer لـ handler مش موجود ويحصل kernel panic.

---

### تجربة الـ Module

```bash
# بناء وتحميل
make
sudo insmod wakeirq_probe.ko

# نفرّع device بيسجّل wakeup IRQ (مثلاً USB suspend/resume)
# أو نشوف الـ log مباشرة
sudo dmesg | grep wakeirq_probe

# إزالة الـ module
sudo rmmod wakeirq_probe
```

**مثال output متوقع:**

```
[  42.123456] wakeirq_probe: planted kprobe at dev_pm_set_wake_irq (ffffffff81a3c210)
[  45.789012] wakeirq_probe: device 'i2c-0' registering wakeup IRQ 32
[  45.800100] wakeirq_probe: device '0000:00:14.0' registering wakeup IRQ 9
```

---

### ملاحظة على الـ Safety

`dev_pm_set_wake_irq` بتتسمى في سياق بيجيز الـ sleep، ومفيش lock خاص حساس فيها، فالـ kprobe هنا **آمن** ومش بيأثر على الـ timing بشكل يعطّل الـ PM subsystem.
