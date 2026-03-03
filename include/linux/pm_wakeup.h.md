## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

الملف `include/linux/pm_wakeup.h` هو الـ **public interface** الخاص بنظام **Wakeup Sources** في Linux kernel. ده جزء من subsystem أكبر اسمه **Power Management Core** — اللي بيديره Rafael J. Wysocki ومتاح على قائمة `linux-pm@vger.kernel.org`.

---

### القصة الكبيرة — تخيل معايا

#### المشكلة

تخيل إنك شايل laptop وبيعمل suspend — يعني بيدخل في نوم خفيف عشان يوفر battery. السؤال: **إيمتى المفروض يصحى؟**

- لما تضغط زرار الـ power؟ أيوه.
- لما USB keyboard بتيجي packet؟ أيوه.
- لما الـ Wi-Fi يستقبل magic packet؟ ممكن أيوه.
- لما timer داخلي بيخلص؟ ممكن لأ — المفروض يكمل نايم.

الكيرنل محتاج يعرف: **مين الأجهزة اللي مسموحالها تصحي الجهاز؟** ومين الأجهزة اللي لسه "صاحية" وبتمنع الـ suspend يحصل أصلاً؟

#### الحل — الـ Wakeup Source

كل device أو كل event ممكن يصحي الجهاز — بيتسجل كـ **`wakeup_source`**. ده زي بطاقة هوية بتقول: "أنا مسموحلي أصحي الجهاز."

لما كيرنل يحاول يعمل suspend، بيعمل check: في أي `wakeup_source` active دلوقتي؟ لو أيوه — suspend بيتأجل أو بينلغي. لو لأ — الجهاز ينام.

---

### الـ Subsystem

**POWER MANAGEMENT CORE** — المسؤول عنه في MAINTAINERS:
```
M: Rafael J. Wysocki <rafael@kernel.org>
F: drivers/base/power/
F: include/linux/pm_wakeup.h
F: include/linux/pm*.h
```

---

### هدف الملف

`pm_wakeup.h` بيعمل حاجتين:

1. **يعرّف الـ `struct wakeup_source`** — الـ data structure الأساسية اللي بتمثل أي مصدر ممكن يصحي الجهاز.
2. **يعلن عن الـ API** — الدوال اللي بيستخدمها كل driver عشان يسجل نفسه كـ wakeup source، أو يقول "أنا صاحي دلوقتي"، أو "أنا خلصت ورجعت هادي".

---

### الـ `struct wakeup_source` — بالتفصيل

```c
struct wakeup_source {
    const char      *name;           // اسم المصدر — ظاهر في sysfs
    int              id;             // رقم تعريفي فريد
    struct list_head entry;          // عنصر في قائمة كل الـ wakeup sources
    spinlock_t       lock;           // حماية من race conditions
    struct wake_irq *wakeirq;        // الـ IRQ المخصص للإيقاظ (اختياري)
    struct timer_list timer;         // timer لإلغاء التنشيط تلقائياً
    unsigned long    timer_expires;  // متى ينتهي الـ timer

    ktime_t total_time;              // إجمالي وقت النشاط
    ktime_t max_time;                // أطول فترة نشاط متواصلة
    ktime_t last_time;              // آخر مرة اتلمس فيها
    ktime_t start_prevent_time;      // بداية منع الـ autosleep
    ktime_t prevent_sleep_time;      // إجمالي وقت منع الـ autosleep

    unsigned long event_count;       // عدد الأحداث اللي اتسجلت
    unsigned long active_count;      // عدد مرات التنشيط
    unsigned long relax_count;       // عدد مرات إلغاء التنشيط
    unsigned long expire_count;      // عدد مرات انتهاء الـ timer
    unsigned long wakeup_count;      // عدد مرات إلغاء suspend بسببه

    struct device   *dev;            // الـ device المرتبط به في sysfs
    bool active:1;                   // هل هو نشط دلوقتي؟
    bool autosleep_enabled:1;        // هل الـ autosleep شغال؟
};
```

الـ statistics دي مش مجرد زينة — بتتعرض في `/sys/kernel/debug/wakeup_sources` وبيستخدمها المطورين لتشخيص مشاكل الـ suspend مثلاً "مين اللي بيمنع الجهاز ينام؟"

---

### الـ API — كيف بيستخدمه الـ Driver

#### دورة حياة الـ Wakeup Source

```
Driver Load
    │
    ▼
wakeup_source_register(dev, "my-device")
    │  ← بتسجل الـ device كمصدر إيقاظ محتمل
    │
    ▼
device_set_wakeup_capable(dev, true)
    │  ← بتقول "الجهاز ده قادر يصحي النظام"
    │
    ▼
device_wakeup_enable(dev)
    │  ← بتقول "وريح، مسموح له فعلاً"
    │
    ▼
[Normal Operation]
    │
    ├─► pm_stay_awake(dev)      ← "أنا شغال، متناموش!"
    │
    ├─► pm_relax(dev)           ← "أنا خلصت، ممكن تناموا"
    │
    └─► pm_wakeup_event(dev, msec) ← "في حدث، استنوا X ms قبل ما تناموا"
    │
    ▼
Driver Unload
    │
    ▼
wakeup_source_unregister(ws)
```

#### الفرق بين الدوال

| الدالة | الهدف |
|--------|--------|
| `pm_stay_awake(dev)` | تنشيط الـ wakeup source — يمنع الـ suspend |
| `pm_relax(dev)` | إلغاء التنشيط — يسمح بالـ suspend |
| `pm_wakeup_event(dev, msec)` | ينشط لـ `msec` milliseconds ثم يريّح تلقائياً |
| `pm_wakeup_hard_event(dev)` | نفس السابق بس بـ "hard" flag — يلغي suspend جارٍ |
| `__pm_stay_awake(ws)` | نفس `pm_stay_awake` بس بيشتغل على الـ wakeup_source مباشرةً |
| `__pm_relax(ws)` | نفس `pm_relax` بس direct على الـ wakeup_source |

---

### الفرق بين `CONFIG_PM_SLEEP` و بدونه

الملف عنده قسمين:

**مع `CONFIG_PM_SLEEP`:** كل الدوال بتشتغل فعلياً — بيسجل الـ sources، بيتحكم في الـ suspend.

**بدون `CONFIG_PM_SLEEP`:** stub functions فاضية — النظام مش بيدعم suspend أصلاً، فمفيش داعي لأي منطق.

ده pattern شائع جداً في الكيرنل عشان يخلي الـ drivers تشتغل على embedded systems من غير PM.

---

### الملفات اللي المفروض تعرفها

#### الـ Core Files

| الملف | الدور |
|--------|--------|
| `drivers/base/power/wakeup.c` | التنفيذ الفعلي لكل الدوال المعلنة في الـ header |
| `drivers/base/power/wakeup_stats.c` | تصدير الـ statistics لـ sysfs |
| `drivers/base/power/wakeirq.c` | إدارة الـ dedicated wake IRQs |
| `drivers/base/power/main.c` | الـ PM core — بيتعامل مع suspend/resume |

#### الـ Headers المرتبطة

| الملف | الدور |
|--------|--------|
| `include/linux/pm_wakeup.h` | **هذا الملف** — الـ wakeup source interface |
| `include/linux/pm_wakeirq.h` | API خاص بالـ wake IRQs |
| `include/linux/pm.h` | الـ PM data structures الأساسية (struct dev_pm_info) |
| `include/linux/pm_runtime.h` | Runtime PM — نوم جزئي للأجهزة أثناء عمل النظام |
| `include/linux/pm_domain.h` | Power domains — مجموعات من الأجهزة بتصحى وتنام سوا |

#### أين بيتسجل الـ device

الـ `struct device` في `include/linux/device.h` بيحتوي على:
```c
struct dev_pm_info {
    bool        can_wakeup;      // قادر يصحي؟
    bool        wakeup_path;     // في مسار الإيقاظ؟
    bool        out_band_wakeup; // إيقاظ خارج النطاق؟
    struct wakeup_source *wakeup; // الـ wakeup source المرتبط به
    ...
};
```

---

### مثال حقيقي — USB Keyboard

```
مستخدم يضغط زرار على keyboard أثناء suspend
        │
        ▼
USB interrupt handler بيشتغل
        │
        ▼
pm_wakeup_hard_event(keyboard_dev)
        │  ← بيعمل: pm_wakeup_dev_event(dev, 0, hard=true)
        │
        ▼
pm_wakeup_ws_event(ws, 0, true)
        │
        ▼
__pm_wakeup_event ← يرفع combined_event_count
        │
        ▼
kernel suspend loop بيشوف: events_check_enabled && new events
        │
        ▼
suspend يتلغى — الجهاز يصحى
```

---

### القصة الكاملة

فكر في الكيرنل زي manager شركة. الـ suspend زي "إغلاق المكتب آخر اليوم". قبل ما يقفل الباب، بيسأل كل موظف (device): "خلصت شغلك؟"

- لو موظف قال "لأ لسه" (`pm_stay_awake`) — المكتب فضل مفتوح.
- لما قال "خلصت" (`pm_relax`) — الـ manager بيشطبه.
- لما كل الموظفين بلغوا "خلصنا" — الـ manager يقفل الباب (suspend).
- لو حد من بره طرق الباب (wakeup event) — الـ manager يفتح تاني.

الـ `wakeup_source` هو "بطاقة حضور" كل موظف. والـ statistics (total_time, event_count, etc.) هي سجل الحضور والانصراف.
## Phase 2: شرح الـ Wakeup Sources Framework

### المشكلة اللي بيحلها الـ Subsystem

الـ Linux kernel عنده نظام كامل لإدارة الطاقة (Power Management). الهدف الأساسي إنه يقدر يحط الجهاز في حالة نوم (suspend) عشان يوفر طاقة — سواء كان suspend-to-RAM أو hibernate أو autosleep.

المشكلة الحقيقية: **إزاي الـ kernel يعرف إنه آمن يدخل في الـ sleep؟**

لو فيه event بيتعالج دلوقتي — زي packet شبكة واصل، أو touch screen بيُعالَج، أو USB data بييجي — ودخل في الـ sleep وسط الأحداث دي، هيكون كارثة: بيانات اتضاعت، user لقا الجهاز مش بيرد، أو أحيانًا hardware corruption.

المشكلة بتنقسم لجزئين:
1. **Race condition أثناء الـ suspend:** الـ kernel بدأ يدخل suspend، وفي نفس الوقت جه wakeup event — لو ما اتعاملناش معه صح، هيُتجاهل.
2. **من يقرر التوقيت؟** مش الـ kernel core اللي يعرف إن driver معين لسه شغال على event — الـ driver نفسه هو اللي يعرف.

---

### الحل: الـ Wakeup Sources Framework

الكيرنل بيعمل **abstraction** اسمه `wakeup_source`. الفكرة:

- كل entity (device أو subsystem) اللي ممكن يمنع الـ sleep بيمتلك `wakeup_source` object.
- لما بيبدأ يشتغل على event، بيقول للكيرنل: "أنا صاحي" (`pm_stay_awake`).
- لما يخلص: "خلصت" (`pm_relax`).
- الـ PM core مش هيسمح بالـ suspend طالما في أي `wakeup_source` نشط.

ده بيحل الـ race condition بـ **atomic counter** (`combined_event_count`) بيجمع اتنين في واحد: عدد الأحداث المسجلة + عدد الأحداث اللي لسه بتتعالج. الـ suspend بييحصل بس لو العدادين اتأكدوا.

---

### التشبيه الحقيقي — مطعم بيقفل

تخيل مطعم بيقفل في نص الليل:

| عنصر الـ Analogy | المقابل في الـ Kernel |
|---|---|
| المطعم | الـ system كلها |
| مدير المطعم | الـ PM core (`kernel/power/`) |
| النادل | الـ driver |
| الزبون جالس بياكل | الـ wakeup event بيتعالج |
| الزبون دخل → النادل رفع إشارة "مشغول" | `pm_stay_awake()` |
| الزبون دفع وخرج → النادل خفض الإشارة | `pm_relax()` |
| كام إشارة "مشغول" مرفوعة | `combined_event_count` — الـ in-progress bits |
| المدير بيشوف صفر إشارات → بيقفل | الـ PM core يرى `inpr == 0` → يكمل الـ suspend |
| زبون دخل في آخر لحظة قبل ما الباب يقفل | race condition: wakeup event وصل أثناء suspend |
| النادل لازم يرفع الإشارة قبل ما يدخل الزبون | الـ driver لازم يستدعي `pm_stay_awake` قبل ما يبدأ يعالج الـ event |
| لو المدير لاقى إشارة مرفوعة، ما يقفلش | `pm_wakeup_pending()` بترجع true → الـ suspend بيتأجل |

الـ mapping هنا مش سطحي:
- "صفر إشارات" = `inpr == 0` و `cnt == saved_count` — الـ event counter ما اتغيرش من وقت ما الـ PM core سجل القيمة القديمة.
- "الباب" = حالة `events_check_enabled` — لما الـ PM يبدأ يجهز للـ suspend، بيمكّن الـ check ده.
- "الزبون جه مبكر قبل الإشارة" = driver لازم يضمن استدعاء `pm_stay_awake` قبل أي معالجة.

---

### Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        User Space                               │
  │   /sys/power/wakeup_count    /sys/devices/.../power/wakeup      │
  └────────────────┬───────────────────────┬────────────────────────┘
                   │                       │ sysfs
  ─────────────────┼───────────────────────┼──────────────────────────
  │                ▼                       ▼                         │
  │     ┌──────────────────────┐  ┌──────────────────────────────┐  │
  │     │   kernel/power/      │  │   drivers/base/power/        │  │
  │     │   suspend.c          │  │   wakeup.c  ← CORE FILE      │  │
  │     │   autosleep.c        │  │                              │  │
  │     │                      │  │  wakeup_sources (list)       │  │
  │     │  pm_wakeup_pending() │◄─┤  combined_event_count        │  │
  │     │  (abort suspend?)    │  │  events_check_enabled        │  │
  │     └──────────────────────┘  └────────────┬─────────────────┘  │
  │              PM Core                        │                    │
  ─────────────────────────────────────────────┼────────────────────
                                               │
              ┌──────────────────────┬─────────┴──────────────────┐
              │                      │                            │
              ▼                      ▼                            ▼
  ┌─────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
  │  struct device  │   │  struct device        │   │  Virtual / Software  │
  │  (e.g., USB)    │   │  (e.g., RTC alarm)    │   │  wakeup_source       │
  │                 │   │                        │   │  (e.g., eventpoll)   │
  │ dev.power.wakeup│   │ dev.power.wakeup       │   │                      │
  │   = &ws_usb     │   │   = &ws_rtc            │   │  ws_epoll            │
  └────────┬────────┘   └──────────┬─────────────┘   └──────────┬───────────┘
           │                       │                             │
           ▼                       ▼                             ▼
  ┌─────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
  │ wakeup_source   │   │ wakeup_source         │   │ wakeup_source        │
  │ name="usb0"     │   │ name="rtc0"           │   │ name="epoll"         │
  │ active=true     │   │ active=false          │   │ active=false         │
  │ wakeirq=...     │   │ wakeirq=NULL          │   │ wakeirq=NULL         │
  └─────────────────┘   └──────────────────────┘   └──────────────────────┘
       ▲ pm_stay_awake()            ▲ pm_wakeup_event(dev, 500ms)
       │ pm_relax()                 │
  USB driver ISR                 RTC alarm ISR
```

---

### الـ Core Abstraction: `struct wakeup_source`

ده أهم struct في الـ subsystem كله. هو بيمثل **"أي حاجة ممكن تمنع الـ sleep"**.

```c
struct wakeup_source {
    const char          *name;           /* اسم المصدر — ظاهر في sysfs */
    int                  id;             /* unique ID من wakeup_ida */
    struct list_head     entry;          /* عنصر في global list */
    spinlock_t           lock;           /* حماية الـ state */
    struct wake_irq     *wakeirq;        /* IRQ مخصص للـ wakeup (optional) */
    struct timer_list    timer;          /* timer للـ timeout-based wakeup */
    unsigned long        timer_expires;  /* موعد انتهاء الـ timer */

    /* Statistics — بتظهر في debugfs */
    ktime_t total_time;          /* المدة الكلية اللي كان فيها active */
    ktime_t max_time;            /* أطول مدة active متواصلة */
    ktime_t last_time;           /* آخر مرة اتلمس */
    ktime_t start_prevent_time;  /* وقت بداية منع الـ autosleep */
    ktime_t prevent_sleep_time;  /* المدة الكلية لمنع الـ autosleep */

    unsigned long event_count;   /* عدد الأحداث المُبلَّغ عنها */
    unsigned long active_count;  /* عدد مرات التفعيل */
    unsigned long relax_count;   /* عدد مرات إلغاء التفعيل */
    unsigned long expire_count;  /* عدد مرات انتهاء الـ timer */
    unsigned long wakeup_count;  /* عدد مرات كاد يمنع suspend */

    struct device *dev;          /* الـ device المرتبط (للـ sysfs) */
    bool active:1;               /* هل هو نشط دلوقتي؟ */
    bool autosleep_enabled:1;    /* هل الـ autosleep شغال؟ */
};
```

#### العلاقة مع `struct device` و `struct dev_pm_info`

```
struct device
└── struct dev_pm_info  power
    ├── bool can_wakeup:1          ← هل الـ hardware قادر hardware-wakeup؟
    ├── struct wakeup_source *wakeup ← مؤشر للـ wakeup_source المرتبط
    ├── bool wakeup_path:1         ← هل الـ device جزء من مسار الـ wakeup؟
    ├── bool out_band_wakeup:1     ← wakeup من خارج نطاق الـ PM callbacks
    └── struct wake_irq *wakeirq   ← IRQ مخصص للـ wakeup
```

**الـ `can_wakeup`** مش نفسه `wakeup != NULL`:
- `can_wakeup = true`: الـ hardware قادر يصحّي — لكن ممكن يكون معطل من user space.
- `wakeup != NULL`: الـ device فعلًا enabled كـ wakeup source، وعنده `wakeup_source` مسجل.

---

### دورة حياة الـ Wakeup Source

```
  Driver Probe
      │
      ▼
  device_set_wakeup_capable(dev, true)
      │   ← يضع can_wakeup=true، يضيف sysfs attributes
      ▼
  device_wakeup_enable(dev)   ← أو device_init_wakeup(dev, true)
      │   ← يستدعي wakeup_source_register()
      │   ← يعمل wakeup_source_create() → kzalloc + ida_alloc
      │   ← يضيف لـ global wakeup_sources list
      │   ← يضع dev->power.wakeup = ws
      ▼
  [Device Active]
      │
      ├─── IRQ/Event arrives
      │        │
      │        ▼
      │    pm_stay_awake(dev)  أو  __pm_stay_awake(ws)
      │        │   ← ws->active = true
      │        │   ← atomic_inc(&combined_event_count) — in-progress bits
      │        │
      │        ▼
      │    [Processing event...]
      │        │
      │        ▼
      │    pm_relax(dev)  أو  __pm_relax(ws)
      │        │   ← ws->active = false
      │        │   ← يحدث statistics (total_time, max_time...)
      │        │   ← atomic_add(MAX_IN_PROGRESS, &combined_event_count)
      │        │     ↑ ده بيزود الـ "registered events" counter بـ 1
      │        │       وبيخصم 1 من الـ "in-progress" counter
      │        │
      │        ▼
      │    [Back to idle]
      │
      └─── Timer-based wakeup: pm_wakeup_event(dev, msec)
               │   ← يسجل event + يضع timer
               ▼
           pm_wakeup_timer_fn() fires after msec
               │
               ▼
           wakeup_source_deactivate(ws)
```

---

### الـ Combined Event Counter — التفصيل

ده أذكى جزء في الـ framework. الـ kernel بيستخدم **atomic واحد** يحمل عددين:

```c
static atomic_t combined_event_count = ATOMIC_INIT(0);

#define IN_PROGRESS_BITS   (sizeof(int) * 4)   /* = 16 bits */
#define MAX_IN_PROGRESS    ((1 << IN_PROGRESS_BITS) - 1)  /* = 0xFFFF */
```

```
  Bit layout of combined_event_count (32-bit):
  ┌─────────────────────┬──────────────────────┐
  │  bits [31..16]      │  bits [15..0]        │
  │  registered_count   │  in_progress_count   │
  └─────────────────────┴──────────────────────┘
```

- **`wakeup_source_activate`**: بيعمل `atomic_inc` → يزود الـ in-progress بـ 1.
- **`wakeup_source_deactivate`**: بيعمل `atomic_add(MAX_IN_PROGRESS + 1)` → يزود الـ registered بـ 1 وينقص الـ in-progress بـ 1.

**ليه؟** عشان الـ two operations تكون **atomically consistent**. لو اتقسموا لاتنين atomic ops، ممكن الـ PM core يشوف حالة intermediate غلط.

```c
/* pm_wakeup_pending() — بيقرر هل نكمل suspend ولا لأ */
bool pm_wakeup_pending(void)
{
    unsigned int cnt, inpr;
    split_counters(&cnt, &inpr);

    /* لو في events بتتعالج (inpr > 0) → لا تنام */
    /* لو events جديدة اتسجلت من آخر مرة PM سجل (cnt != saved_count) → لا تنام */
    ret = (cnt != saved_count || inpr > 0);
}
```

---

### الفرق بين طريقتي الإبلاغ عن الـ Wakeup Event

الـ framework بيدي الـ drivers خيارين:

#### الطريقة الأولى: `pm_stay_awake` / `pm_relax`
لما الـ driver بيعالج الـ event هو بنفسه ويعرف بالظبط امتى بيخلص:

```c
/* في الـ ISR */
pm_stay_awake(dev);      /* أنا بدأت أشتغل */

/* بعد ما يخلص المعالجة */
pm_relax(dev);           /* خلصت */
```

**مثال:** USB HID driver — بيقرأ الـ packet كله ويحوله لـ input events، بعدين بيعمل `pm_relax`.

#### الطريقة الثانية: `pm_wakeup_event(dev, msec)`
لما الـ driver ما يعرفش بالظبط امتى هيخلص — بيحجز وقت بالـ milliseconds:

```c
/* في الـ ISR */
pm_wakeup_event(dev, 500);   /* أعطيني 500ms عشان أخلص */
/* الـ kernel بيضع timer، بعد 500ms يعمل pm_relax تلقائيًا */
```

**مثال:** RTC alarm driver — يُبلّغ عن الـ alarm، ويطلب 500ms عشان الـ user space يقرأ الـ event.

الـ `hard` flag في `pm_wakeup_ws_event`:
```c
void pm_wakeup_ws_event(struct wakeup_source *ws, unsigned int msec, bool hard)
{
    /* ...  */
    if (hard)
        pm_system_wakeup();  /* يصحّي من suspend-to-idle فورًا */
}
```

`hard=true` → يستدعي `pm_system_wakeup()` اللي بيزود `pm_abort_suspend` بـ 1 و بيصحّي الـ `s2idle` فورًا.

---

### الـ Wakeup Path

الـ `wakeup_path` flag مختلف عن الـ `wakeup_source`:

```
  ┌───────────────────────────────────────────────────────┐
  │   Wakeup Path — المسار من الـ wakeup device للـ CPU  │
  │                                                       │
  │  [RTC] → [I2C Controller] → [SoC Power Domain]       │
  │    ↑           ↑                    ↑                 │
  │ wakeup_source  wakeup_path=true    wakeup_path=true   │
  │ (مصدر الـ wakeup) (وسيط — لازم يفضل شغال)           │
  └───────────────────────────────────────────────────────┘
```

`device_set_wakeup_path(dev)` بيقول للـ PM إن الـ device ده جزء من المسار اللي لازم يفضل powered أثناء الـ suspend عشان الـ wakeup يوصل للـ CPU.

`out_band_wakeup` بيُستخدم لـ devices اللي بتصحي من خارج الـ PM callback path — زي device بيولّد interrupt مباشرة على GPIO.

---

### الـ devm Integration

```c
static inline int devm_device_init_wakeup(struct device *dev)
{
    device_init_wakeup(dev, true);
    /* بيسجل cleanup function تلقائيًا تُستدعى عند driver unbind */
    return devm_add_action_or_reset(dev, device_disable_wakeup, dev);
}
```

**الـ devm subsystem** — resource management subsystem بيضمن إن الـ resources اتسابها بعد كده لو الـ probe فشل أو الـ driver اتفصل. معرفة الـ devm ضرورية هنا.

```
  driver_probe_device()
       │
       ├── devm_device_init_wakeup(dev)
       │       ├── device_init_wakeup(dev, true)
       │       │       ├── device_set_wakeup_capable(dev, true)
       │       │       └── device_wakeup_enable(dev)
       │       │               └── wakeup_source_register(dev, name)
       │       └── devm_add_action(dev, device_disable_wakeup)
       │
       ▼
  [Driver Working...]
       │
       ▼ (on driver removal or error)
  device_disable_wakeup() called automatically
       └── device_init_wakeup(dev, false)
               ├── device_wakeup_disable(dev)
               │       └── wakeup_source_unregister(ws)
               └── device_set_wakeup_capable(dev, false)
```

---

### ما الـ Subsystem بيمتلكه vs ما بيفوّضه للـ Drivers

| المهمة | المالك |
|---|---|
| تتبع الـ wakeup sources في global list | PM core (`wakeup_sources` list) |
| الـ atomic counter لإدارة الـ race conditions | PM core (`combined_event_count`) |
| قرار إلغاء أو السماح بالـ suspend | PM core (`pm_wakeup_pending`) |
| إدارة الـ statistics (total_time, event_count...) | PM core |
| sysfs exposure لكل wakeup source | PM core (`wakeup_source_sysfs_add`) |
| تعريف "متى" يبدأ/ينتهي الـ wakeup event | الـ Driver |
| استدعاء `pm_stay_awake` / `pm_relax` | الـ Driver |
| تحديد `can_wakeup` عند الـ probe | الـ Driver |
| تسجيل الـ wakeirq (optional) | الـ Driver + wakeirq subsystem |
| تنفيذ الـ suspend/resume callbacks | الـ Driver + PM domain |

---

### مثال عملي — Touch Screen Driver على ARM

```c
/* في probe() */
int ts_probe(struct platform_device *pdev)
{
    struct ts_data *ts = ...;

    /* 1. أعلن إن الـ hardware قادر يصحّي */
    device_init_wakeup(&pdev->dev, true);
    /* أو بـ devm: devm_device_init_wakeup(&pdev->dev) */

    /* 2. اربط الـ wakeup IRQ */
    ts->irq = platform_get_irq(pdev, 0);
    request_irq(ts->irq, ts_interrupt, IRQF_TRIGGER_FALLING, "ts", ts);
}

/* في الـ ISR */
irqreturn_t ts_interrupt(int irq, void *dev_id)
{
    struct ts_data *ts = dev_id;

    /* أعلن إننا صاحيين — امنع الـ sleep */
    pm_stay_awake(&ts->pdev->dev);

    /* قرأنا الـ coordinates من الـ hardware */
    ts_read_coords(ts);

    /* أرسل الـ input events */
    input_report_abs(ts->input, ABS_X, ts->x);
    input_report_abs(ts->input, ABS_Y, ts->y);
    input_sync(ts->input);

    /* خلصنا — اسمح بالـ sleep */
    pm_relax(&ts->pdev->dev);

    return IRQ_HANDLED;
}

/* في suspend() */
int ts_suspend(struct device *dev)
{
    /* الـ wakeup_source هيتعطل تلقائيًا من الـ PM core */
    /* لو device_may_wakeup(dev) → الكيرنل يُفعّل الـ wakeirq */
    if (device_may_wakeup(dev))
        enable_irq_wake(ts->irq);  /* الـ IRQ يصحّي الـ system */
    return 0;
}
```

**الـ `device_may_wakeup`** — بيتحقق من اتنين:
```c
static inline bool device_may_wakeup(struct device *dev)
{
    return dev->power.can_wakeup && !!dev->power.wakeup;
    /* can_wakeup: hardware capable */
    /* wakeup != NULL: user enabled it via sysfs */
}
```

---

### الـ Sysfs Interface

```
/sys/devices/platform/touchscreen/power/
├── wakeup              ← "enabled" أو "disabled" (read/write)
├── wakeup_count        ← عدد مرات أوقف suspend
├── wakeup_active_count ← عدد مرات اتفعّل
├── wakeup_total_time_ms← المدة الكلية active
└── wakeup_max_time_ms  ← أطول مدة active

/sys/power/wakeup_count  ← الـ global counter — userspace بيقراه قبل suspend
                            ولو اتغير → في wakeup event → يعيد المحاولة
```

الـ userspace (زي `systemd-sleep`) بيعمل:

```bash
# اقرأ الـ current count
count=$(cat /sys/power/wakeup_count)

# حاول تكتب نفس الـ count — لو نجح: مفيش events جديدة → امشي في الـ suspend
echo $count > /sys/power/wakeup_count && echo mem > /sys/power/state
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags, Config Options & Macros — Cheatsheet

#### Config Options

| Option | المعنى |
|---|---|
| `CONFIG_PM_SLEEP` | يفعّل كل الـ wakeup management الحقيقي — بدونه كل الـ APIs بتبقى stubs فاضية |
| `CONFIG_PM` | يفعّل الـ runtime PM fields جوا `dev_pm_info` |
| `CONFIG_PM_CLK` | يضيف clock management جوا `pm_subsys_data` |
| `CONFIG_PM_GENERIC_DOMAINS` | يضيف domain_data جوا `pm_subsys_data` |

#### DPM Driver Flags (بيتضبطوا وقت probe)

| Flag | القيمة | المعنى |
|---|---|---|
| `DPM_FLAG_NO_DIRECT_COMPLETE` | `BIT(0)` | ما تعملش direct-complete optimization |
| `DPM_FLAG_SMART_PREPARE` | `BIT(1)` | خد في الاعتبار return value من `->prepare` |
| `DPM_FLAG_SMART_SUSPEND` | `BIT(2)` | تجنب resume من runtime suspend |
| `DPM_FLAG_MAY_SKIP_RESUME` | `BIT(3)` | سمح بـ skip الـ noirq/early resume callbacks |

#### Macros & Iterators

| Macro | الاستخدام |
|---|---|
| `for_each_wakeup_source(ws)` | يمشي على كل الـ wakeup sources المسجلة — thread-safe عبر read-lock داخلي |

#### Inline Helper Functions — Summary

| الدالة | السياق | الهدف |
|---|---|---|
| `device_can_wakeup(dev)` | always | هل الـ device capable أصلاً؟ (`can_wakeup`) |
| `device_may_wakeup(dev)` | always | capable **و** مفعّل (`wakeup != NULL`) |
| `device_wakeup_path(dev)` | PM_SLEEP | هل الـ device في مسار الـ wakeup؟ |
| `device_set_wakeup_path(dev)` | PM_SLEEP | ضع الـ device في مسار الـ wakeup |
| `device_out_band_wakeup(dev)` | PM_SLEEP | هل يستخدم out-of-band wakeup mechanism؟ |
| `device_set_out_band_wakeup(dev)` | PM_SLEEP | فعّل الـ out-of-band wakeup |
| `device_awake_path(dev)` | always | alias لـ `device_wakeup_path` |
| `device_set_awake_path(dev)` | always | alias لـ `device_set_wakeup_path` |
| `__pm_wakeup_event(ws, msec)` | always | يبعت wakeup event بـ `hard=false` |
| `pm_wakeup_event(dev, msec)` | always | يبعت wakeup event للـ device |
| `pm_wakeup_hard_event(dev)` | always | يبعت hard wakeup event فوري (`msec=0, hard=true`) |
| `device_init_wakeup(dev, enable)` | always | يعمل init كامل للـ wakeup capability |
| `devm_device_init_wakeup(dev)` | always | نسخة resource-managed — بتعمل auto-disable عند remove |

---

### الـ Structs المهمة

#### 1. `struct wakeup_source`

**الغرض:** الـ struct الأساسي اللي بيمثّل أي **wakeup source** في النظام — سواء كان device أو software entity. أي حاجة تقدر تمنع الـ system من الـ suspend بتمتلك `wakeup_source`.

| الـ Field | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | اسم الـ wakeup source — بيظهر في `/sys/kernel/debug/wakeup_sources` |
| `id` | `int` | معرّف فريد بيتحدده الـ kernel عند التسجيل |
| `entry` | `struct list_head` | ربطه في الـ global wakeup sources list |
| `lock` | `spinlock_t` | يحمي كل الـ fields الجوانية — يُمسك قبل أي تعديل |
| `wakeirq` | `struct wake_irq *` | pointer للـ dedicated wakeup IRQ (اختياري) — IRQ مخصص لإيقاظ الـ system |
| `timer` | `struct timer_list` | الـ timer اللي بيعمل auto-deactivation بعد مدة |
| `timer_expires` | `unsigned long` | وقت انتهاء الـ timer |
| `total_time` | `ktime_t` | مجموع الوقت اللي الـ source قضاه active |
| `max_time` | `ktime_t` | أطول فترة continuous activity |
| `last_time` | `ktime_t` | آخر وقت اتلمس فيه الـ source (monotonic clock) |
| `start_prevent_time` | `ktime_t` | أول لحظة بدأ فيها يمنع الـ autosleep |
| `prevent_sleep_time` | `ktime_t` | مجموع الوقت اللي منع فيه الـ autosleep |
| `event_count` | `unsigned long` | عدد الـ wakeup events اللي اتعملت signaling ليها |
| `active_count` | `unsigned long` | عدد مرات الـ activation |
| `relax_count` | `unsigned long` | عدد مرات الـ deactivation |
| `expire_count` | `unsigned long` | عدد مرات انتهاء الـ timer |
| `wakeup_count` | `unsigned long` | عدد مرات اللي الـ source فيها قدر يمنع الـ suspend |
| `dev` | `struct device *` | pointer للـ device المرتبط — لـ sysfs stats |
| `active` | `bool:1` | هل الـ source active دلوقتي؟ |
| `autosleep_enabled` | `bool:1` | هل الـ autosleep شغال؟ — لتحديث `prevent_sleep_time` |

**الارتباط بـ structs تانية:**
- `dev->power.wakeup` → بيشاور عليه من `struct dev_pm_info`
- `wakeirq` → `struct wake_irq` (defined في `drivers/base/power/wakeirq.c`)
- `entry` → بيخليه جزء من الـ global linked list المحمية بـ `wakeup_source_lock`

---

#### 2. `struct dev_pm_info` (من `include/linux/pm.h`)

**الغرض:** بيتضمن **كل المعلومات المتعلقة بالـ PM** للـ device — موجود في `struct device` تحت `dev->power`.

الـ fields المتعلقة بالـ wakeup:

| الـ Field | السياق | الشرح |
|---|---|---|
| `can_wakeup` | always | هل الـ hardware قادر أصلاً يعمل wakeup؟ |
| `wakeup` | `CONFIG_PM_SLEEP` | pointer لـ `wakeup_source` — NULL يعني مش مفعّل |
| `wakeup_path` | `CONFIG_PM_SLEEP` | الـ device في مسار الـ wakeup (ولو مش هو اللي بيعمله) |
| `out_band_wakeup` | `CONFIG_PM_SLEEP` | يستخدم آلية out-of-band (e.g., GPIO بره الـ IRQ العادي) |
| `should_wakeup` | `!CONFIG_PM_SLEEP` | بديل مبسّط لما الـ PM_SLEEP مش مفعّل |
| `wakeirq` | `CONFIG_PM` | pointer لـ `wake_irq` للـ runtime PM |

---

#### 3. `struct wake_irq` (forward declaration فقط في الملف)

**الغرض:** بيمثّل الـ **dedicated wakeup IRQ** المرتبط بـ device — سواء كان dedicated IRQ أو shared IRQ. بيتعرّف في `drivers/base/power/wakeirq.c`.

---

### رسم علاقات الـ Structs

```
struct device
└── power: struct dev_pm_info
    ├── can_wakeup: bool
    ├── wakeup_path: bool
    ├── out_band_wakeup: bool
    ├── wakeup ──────────────────────► struct wakeup_source
    │                                   ├── name: "usb-device"
    │                                   ├── id: 42
    │                                   ├── lock: spinlock_t
    │                                   ├── entry ────────► [global wakeup_sources list]
    │                                   ├── wakeirq ──────► struct wake_irq
    │                                   ├── timer ────────► kernel timer list
    │                                   ├── total_time, max_time, last_time
    │                                   ├── event_count, active_count, relax_count
    │                                   ├── wakeup_count, expire_count
    │                                   ├── prevent_sleep_time
    │                                   ├── dev ──────────► struct device (back ref for sysfs)
    │                                   ├── active: bool
    │                                   └── autosleep_enabled: bool
    └── wakeirq ─────────────────────► struct wake_irq (runtime PM)


Global State:
wakeup_sources_list (srcu-protected list)
    ├── wakeup_source_1.entry
    ├── wakeup_source_2.entry
    └── wakeup_source_N.entry
```

---

### دورة حياة الـ Wakeup Source

```
                    ┌─────────────────────────────────────┐
                    │           DEVICE PROBE               │
                    │  device_init_wakeup(dev, true)       │
                    │    → device_set_wakeup_capable()     │
                    │    → device_wakeup_enable()          │
                    │      → wakeup_source_register()      │
                    │        → wakeup_source_create()      │
                    │        → wakeup_source_add()         │
                    │          → list_add(global list)     │
                    │          → dev->power.wakeup = ws    │
                    └───────────────┬─────────────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────────────┐
                    │              IDLE                    │
                    │  ws->active = false                  │
                    │  النظام مسموح له يعمل suspend        │
                    └───────────────┬─────────────────────┘
                                    │
                       ┌────────────┴────────────┐
                       │ wakeup event حصل        │
                       ▼                         ▼
        ┌──────────────────────┐   ┌─────────────────────────┐
        │ __pm_stay_awake(ws)  │   │ pm_wakeup_ws_event(ws,  │
        │  → ws->active = true │   │   msec, hard)           │
        │  → active_count++    │   │  → __pm_stay_awake()    │
        │  → event_count++     │   │  → timer_expires = now  │
        │  → يمنع الـ suspend  │   │    + msec               │
        └──────────┬───────────┘   └───────────┬─────────────┘
                   │                           │
                   │                           ▼
                   │               ┌─────────────────────────┐
                   │               │  timer fires after msec  │
                   │               │  → __pm_relax(ws)        │
                   │               └───────────┬─────────────┘
                   │                           │
                   ▼                           ▼
        ┌──────────────────────────────────────────────────┐
        │              __pm_relax(ws)                       │
        │  ws->active = false                               │
        │  relax_count++                                    │
        │  total_time += (now - last_time)                  │
        │  النظام يقدر يعمل suspend تاني                   │
        └──────────────────────────┬───────────────────────┘
                                   │
                                   ▼
                    ┌─────────────────────────────────────┐
                    │           DEVICE REMOVE              │
                    │  device_wakeup_disable(dev)          │
                    │    → wakeup_source_unregister(ws)    │
                    │      → wakeup_source_remove()        │
                    │        → list_del(global list)       │
                    │      → wakeup_source_destroy(ws)     │
                    │      → dev->power.wakeup = NULL      │
                    └─────────────────────────────────────┘
```

---

### Call Flow Diagrams

#### تفعيل الـ Wakeup Source (Driver Level)

```
driver calls device_init_wakeup(dev, true)
  → device_set_wakeup_capable(dev, true)
      → dev->power.can_wakeup = true
  → device_wakeup_enable(dev)
      [wakeup.c in drivers/base/power]
      → wakeup_source_register(dev, dev_name(dev))
          → wakeup_source_create(name)
              → kzalloc(wakeup_source)
              → timer_setup(&ws->timer, ...)
          → wakeup_source_add(ws)
              → spin_lock(&ws->lock)
              → srcu_lock(wakeup_srcu)
              → list_add_rcu(&ws->entry, &wakeup_sources)
              → srcu_unlock(wakeup_srcu)
      → dev->power.wakeup = ws
```

#### إرسال Wakeup Event من Driver

```
driver (IRQ handler) calls pm_wakeup_event(dev, 500)  /* 500ms */
  → pm_wakeup_dev_event(dev, 500, false)
      → ws = dev->power.wakeup
      → pm_wakeup_ws_event(ws, 500, false)
          → spin_lock_irqsave(&ws->lock)
          → __pm_stay_awake(ws)
              → ws->active = true
              → ws->active_count++
              → ws->event_count++
              → ws->last_time = ktime_get()
              → pm_wakeup_update_hit_counts()  /* update wakeup_count */
          → ws->timer_expires = jiffies + msecs_to_jiffies(500)
          → mod_timer(&ws->timer, ws->timer_expires)
          → spin_unlock_irqrestore(&ws->lock)
          /* after 500ms: */
          → timer fires: wakeup_source_timer_fn()
              → __pm_relax(ws)
                  → ws->active = false
                  → ws->relax_count++
                  → ws->total_time += elapsed
                  → ws->max_time = max(max_time, elapsed)
```

#### Hard Wakeup Event (يمنع الـ suspend فوراً)

```
driver calls pm_wakeup_hard_event(dev)
  → pm_wakeup_dev_event(dev, 0, true)
      → pm_wakeup_ws_event(ws, 0, true)
          → __pm_stay_awake(ws)
          → hard=true AND msec=0
              → pm_system_wakeup()  /* signals suspend to abort immediately */
          /* NO timer set — driver responsible for calling pm_relax() */
```

#### التحقق قبل الـ Suspend

```
PM core calls device_may_wakeup(dev)
  → return (dev->power.can_wakeup && dev->power.wakeup != NULL)
      /* true  → device configured as wakeup source */
      /* false → device can be powered down freely */

PM core iterates:
  for_each_wakeup_source(ws)
    → wakeup_sources_walk_start()  /* takes SRCU read lock */
    → wakeup_sources_walk_next(ws) /* advances list safely */
    → wakeup_sources_read_unlock() /* releases SRCU lock */
```

#### devm Version — Auto Cleanup

```
driver calls devm_device_init_wakeup(dev)
  → device_init_wakeup(dev, true)  /* enable wakeup */
  → devm_add_action_or_reset(dev, device_disable_wakeup, dev)
      /* registers cleanup callback */
      /* when device is removed: */
      → device_disable_wakeup(dev)
          → device_init_wakeup(dev, false)
              → device_wakeup_disable(dev)
              → device_set_wakeup_capable(dev, false)
```

---

### استراتيجية الـ Locking

#### الـ Locks الموجودة

| الـ Lock | النوع | ما بيحميه |
|---|---|---|
| `ws->lock` | `spinlock_t` | كل الـ fields الإحصائية والـ state جوا `wakeup_source` نفسه — `active`, `total_time`, `event_count`, إلخ |
| `wakeup_srcu` (global) | `struct srcu_struct` | الـ global `wakeup_sources` linked list — بيسمح بـ lockless read traversal |
| `dev->power.lock` | `spinlock_t` (في `dev_pm_info`) | الـ fields الخاصة بالـ PM في `dev_pm_info` |

#### قواعد الـ Locking

```
القاعدة 1: ws->lock دايماً irqsave
    spin_lock_irqsave(&ws->lock, flags)
    /* لأن الـ wakeup events بتيجي من interrupt context */
    spin_unlock_irqrestore(&ws->lock, flags)

القاعدة 2: SRCU للـ list traversal
    idx = wakeup_sources_read_lock()   /* srcu_read_lock */
    /* traverse list safely — readers don't block writers */
    wakeup_sources_read_unlock(idx)    /* srcu_read_unlock */

القاعدة 3: ترتيب الـ locking (Lock Ordering)
    dev->power.lock
        └── ws->lock   ← دايماً بعد power.lock، مش العكس
                           منعاً للـ deadlock
```

#### في الـ IRQ Context

```
IRQ fires
  → __pm_stay_awake(ws)  ← safe تُستدعى من IRQ
      → spin_lock_irqsave(&ws->lock)  ← disables local IRQs
      → updates fields
      → spin_unlock_irqrestore(&ws->lock)

الـ timer callback
  → يشتغل في softirq context
  → __pm_relax(ws)
      → spin_lock_irqsave(&ws->lock)  ← نفس الحماية
```

#### الفرق بين `__pm_stay_awake` و `pm_stay_awake`

```c
/* النسخة المباشرة — للـ wakeup_source مباشرة */
__pm_stay_awake(ws);   /* لازم تمسك ws->lock بنفسك إذا محتاج */

/* النسخة الـ device-level — بتجيب الـ ws من dev */
pm_stay_awake(dev);
  → ws = dev->power.wakeup;
  → if (ws) __pm_stay_awake(ws);
```

---

### ملاحظات التوافق — `CONFIG_PM_SLEEP` disabled

لما `CONFIG_PM_SLEEP` مش معمول compile:

- **`wakeup_source_register()`** → بترجع `NULL` دايماً
- **`device_wakeup_enable()`** → بتضبط `should_wakeup = true` بس (بدل ما تعمل `wakeup_source` حقيقي)
- **`device_may_wakeup()`** → بتتحقق من `can_wakeup && should_wakeup` (بدل الـ pointer)
- **`pm_stay_awake()` / `pm_relax()`** → stubs فاضية
- كل الـ stats والـ timing → مش موجودة

ده بيخلي الـ driver code نفسه يشتغل على أي kernel config بدون `#ifdef` في الـ driver نفسه.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Registration & Lifecycle

| Function | Signature (مختصر) | الغرض |
|---|---|---|
| `wakeup_source_register` | `(struct device *dev, const char *name) → *ws` | ينشئ ويسجل `wakeup_source` جديد |
| `wakeup_source_unregister` | `(struct wakeup_source *ws)` | يلغي التسجيل ويحرر الـ `ws` |
| `device_set_wakeup_capable` | `(struct device *dev, bool capable)` | يحدد هل الـ device قادر على الـ wakeup |
| `device_wakeup_enable` | `(struct device *dev) → int` | يفعّل الـ wakeup للـ device |
| `device_wakeup_disable` | `(struct device *dev)` | يعطّل الـ wakeup للـ device |
| `device_set_wakeup_enable` | `(struct device *dev, bool enable) → int` | يفعّل أو يعطّل الـ wakeup |
| `device_init_wakeup` | `(struct device *dev, bool enable) → int` | يهيئ الـ wakeup capability والـ enable معاً |
| `devm_device_init_wakeup` | `(struct device *dev) → int` | نسخة `devm` من `device_init_wakeup(true)` |

#### Query / State

| Function | Signature (مختصر) | الغرض |
|---|---|---|
| `device_can_wakeup` | `(struct device *dev) → bool` | هل الـ device قادر على الـ wakeup أصلاً؟ |
| `device_may_wakeup` | `(struct device *dev) → bool` | هل الـ wakeup مفعّل فعلاً؟ |
| `device_wakeup_path` | `(struct device *dev) → bool` | هل الـ device في مسار الـ wakeup؟ |
| `device_awake_path` | `(struct device *dev) → bool` | alias لـ `device_wakeup_path` |
| `device_out_band_wakeup` | `(struct device *dev) → bool` | هل الـ device يدعم out-of-band wakeup؟ |

#### Path Marking

| Function | Signature (مختصر) | الغرض |
|---|---|---|
| `device_set_wakeup_path` | `(struct device *dev)` | يضع الـ device في مسار الـ wakeup |
| `device_set_awake_path` | `(struct device *dev)` | alias لـ `device_set_wakeup_path` |
| `device_set_out_band_wakeup` | `(struct device *dev)` | يضع علامة out-of-band wakeup |

#### Runtime Wakeup Events

| Function | Signature (مختصر) | الغرض |
|---|---|---|
| `__pm_stay_awake` | `(struct wakeup_source *ws)` | يُنشّط الـ `ws` مباشرة |
| `pm_stay_awake` | `(struct device *dev)` | يُنشّط `dev->power.wakeup` |
| `__pm_relax` | `(struct wakeup_source *ws)` | يُهدّئ الـ `ws` مباشرة |
| `pm_relax` | `(struct device *dev)` | يُهدّئ `dev->power.wakeup` |
| `pm_wakeup_ws_event` | `(ws, msec, hard)` | يُشير لـ wakeup event على `ws` بـ timeout |
| `pm_wakeup_dev_event` | `(dev, msec, hard)` | يُشير لـ wakeup event على device |
| `__pm_wakeup_event` | `(ws, msec)` | wrapper بدون hard flag |
| `pm_wakeup_event` | `(dev, msec)` | wrapper بدون hard flag |
| `pm_wakeup_hard_event` | `(dev)` | hard wakeup بدون timeout |

#### Iteration

| Function | Signature (مختصر) | الغرض |
|---|---|---|
| `wakeup_sources_read_lock` | `() → int` | يأخذ RCU read lock على قائمة الـ wakeup sources |
| `wakeup_sources_read_unlock` | `(int idx)` | يحرر الـ read lock |
| `wakeup_sources_walk_start` | `() → *ws` | يبدأ الـ iteration على قائمة الـ sources |
| `wakeup_sources_walk_next` | `(*ws) → *ws` | ينتقل للعنصر التالي |
| `for_each_wakeup_source` | macro `(ws)` | macro للـ iteration |

---

### الفئة الأولى: Registration & Lifecycle

هذه الـ functions مسؤولة عن إنشاء وتسجيل وتدمير الـ `wakeup_source` objects، وكذلك ربطها بالـ devices. الـ `wakeup_source` هو الكيان المركزي في subsystem الـ wakeup — كل device يريد منع الـ suspend لازم يكون عنده `wakeup_source` مسجّل.

---

#### `wakeup_source_register`

```c
struct wakeup_source *wakeup_source_register(struct device *dev,
                                              const char *name);
```

**الـ function دي بتعمل إيه:**
بتخصص وتهيئ `struct wakeup_source` جديد، وبتضيفه لقائمة الـ wakeup sources العامة (اللي بتتحكم فيها الـ kernel بـ RCU أو spinlock). لو الـ `dev` مش NULL، بتنشئ كمان `sysfs` entries تحت `/sys/class/wakeup` عشان تعرض الإحصائيات.

**Parameters:**
- `dev` — الـ device المرتبط بالـ source، ممكن يكون NULL لو الـ source مش مرتبط بـ device معين (زي الـ early boot sources).
- `name` — الاسم اللي هيتعرض في `/sys/kernel/debug/wakeup_sources` والـ sysfs.

**Return value:**
- Pointer للـ `wakeup_source` الجديد عند النجاح.
- `NULL` لو فشل الـ allocation أو الـ registration.

**Key details:**
- بتنادي داخلياً على `wakeup_source_create()` ثم `wakeup_source_add()`.
- الـ `id` بيتعيّن atomically من counter عالمي.
- الـ locking: بتأخذ `wakeup_sources_lock` (spinlock) وهي بتضيف الـ source للـ list.
- لو `CONFIG_PM_SLEEP` مش معرّف، بترجع NULL دايماً (stub).

**Who calls it:**
بيناديها `device_wakeup_enable()` اللي بيناديه بدوره device core أو الـ driver مباشرة.

---

#### `wakeup_source_unregister`

```c
void wakeup_source_unregister(struct wakeup_source *ws);
```

**الـ function دي بتعمل إيه:**
بتشيل الـ `wakeup_source` من القائمة العامة، وبتحذف الـ sysfs entries المرتبطة بيه، وبتحرر الذاكرة. لازم يتنادى عليها بعد `__pm_relax` عشان متيجيش في حالة active وانت بتحذفه.

**Parameters:**
- `ws` — الـ `wakeup_source` المراد إلغاء تسجيله. ممكن يكون NULL (بتتحقق داخلياً).

**Return value:** `void`.

**Key details:**
- داخلياً بتنادي `wakeup_source_remove()` ثم `wakeup_source_destroy()`.
- لو الـ `ws` لسه active وقت الـ unregister، الـ kernel بتعمل `WARN_ON` وبتعمل relax قبل الحذف.
- الـ locking: بتأخذ `wakeup_sources_lock` أثناء إزالته من الـ list.

**Who calls it:**
`device_wakeup_disable()` بشكل مباشر، وبشكل غير مباشر `device_init_wakeup(dev, false)`.

---

#### `device_set_wakeup_capable`

```c
void device_set_wakeup_capable(struct device *dev, bool capable);
```

**الـ function دي بتعمل إيه:**
بتحدد هل الـ device hardware قادر أصلاً على توليد wakeup signals. ده الـ flag الأساسي — من غيره مينفعش تفعّل الـ wakeup. بيتحكم في `dev->power.can_wakeup`.

**Parameters:**
- `dev` — الـ device المراد تحديد capability بتاعته.
- `capable` — `true` لو الـ hardware يدعم الـ wakeup، `false` لو لأ.

**Return value:** `void`.

**Key details:**
- في `CONFIG_PM_SLEEP`: بتنادي على `device_wakeup_disable()` أوتوماتيك لو الـ `capable` بقى `false` عشان تمنع الـ inconsistency.
- بتعمل update للـ sysfs `wakeup` attribute لو موجود.
- ده الـ flag اللي `device_can_wakeup()` بيقرأه.

**Who calls it:**
الـ bus drivers في الـ probe أو عند اكتشاف الـ hardware capabilities. مثال: `pci_enable_wake()` والـ USB core.

---

#### `device_wakeup_enable`

```c
int device_wakeup_enable(struct device *dev);
```

**الـ function دي بتعمل إيه:**
بتفعّل الـ wakeup للـ device عن طريق إنشاء `wakeup_source` جديد وتخزينه في `dev->power.wakeup`. لازم الـ device يكون `can_wakeup` أولاً.

**Parameters:**
- `dev` — الـ device المراد تفعيل الـ wakeup بتاعته.

**Return value:**
- `0` عند النجاح.
- `-EINVAL` لو `dev->power.can_wakeup` مش set.
- `-ENODEV` أو error من الـ allocation لو فشل.

**Key details:**
- بتنادي `wakeup_source_register(dev, dev_name(dev))` داخلياً.
- الـ mutex `dev->power.lock` بيتأخذ أثناء تعيين `dev->power.wakeup`.
- لو `dev->power.wakeup` موجود بالفعل، بترجع `-EBUSY` في بعض versions.

**Who calls it:**
`device_init_wakeup(dev, true)` وكذلك `device_set_wakeup_enable(dev, true)`.

---

#### `device_wakeup_disable`

```c
void device_wakeup_disable(struct device *dev);
```

**الـ function دي بتعمل إيه:**
بتعطّل الـ wakeup للـ device عن طريق إزالة وتحرير الـ `wakeup_source` المرتبط بـ `dev->power.wakeup`.

**Parameters:**
- `dev` — الـ device المراد تعطيل الـ wakeup بتاعته.

**Return value:** `void`.

**Key details:**
- بتعمل `dev->power.wakeup = NULL` بعد الـ unregister.
- الـ mutex `dev->power.lock` بيتأخذ أثناء العملية.
- لو الـ `dev->power.wakeup` كان NULL، بترجع بدون عمل حاجة.

**Who calls it:**
`device_init_wakeup(dev, false)` وكذلك `device_set_wakeup_capable(dev, false)`.

---

#### `device_set_wakeup_enable`

```c
int device_set_wakeup_enable(struct device *dev, bool enable);
```

**الـ function دي بتعمل إيه:**
wrapper بسيط — لو `enable` يناديها على `device_wakeup_enable()`، لو لأ يناديها على `device_wakeup_disable()`. سهّلت الكود الـ conditional في الـ drivers.

**Parameters:**
- `dev` — الـ target device.
- `enable` — `true` للتفعيل، `false` للتعطيل.

**Return value:**
- نفس return value الـ `device_wakeup_enable()` في حالة الـ enable.
- `0` في حالة الـ disable.

**Key details:**
- في `!CONFIG_PM_SLEEP`: بتعمل set على `dev->power.should_wakeup` مباشرة بدون إنشاء `wakeup_source`.

**Who calls it:**
الـ sysfs `power/wakeup` write handler، وبعض الـ bus drivers.

---

#### `device_init_wakeup`

```c
static inline int device_init_wakeup(struct device *dev, bool enable);
```

**الـ function دي بتعمل إيه:**
الـ one-stop-shop لتهيئة الـ wakeup في الـ probe. لو `enable=true` بتعمل `set_wakeup_capable(true)` ثم `wakeup_enable()`. لو `enable=false` بتعمل `wakeup_disable()` ثم `set_wakeup_capable(false)`.

**Parameters:**
- `dev` — الـ device المراد تهيئته.
- `enable` — هل يُفعَّل الـ wakeup من الأول.

**Return value:**
- نفس return value الـ `device_wakeup_enable()` في حالة `enable=true`.
- `0` في حالة `enable=false`.

**Key details:**
- inline function — مفيش overhead.
- الترتيب مهم: عند الـ enable، الـ `can_wakeup` لازم يتعيّن قبل `wakeup_enable()`.
- عند الـ disable، الـ `wakeup_disable()` الأول عشان يحرر الـ `wakeup_source` قبل ما يشيل الـ `can_wakeup` flag.

**Pseudocode flow:**
```c
if (enable) {
    dev->power.can_wakeup = true;   // set_wakeup_capable
    return wakeup_source_register() // device_wakeup_enable
}
// else:
wakeup_source_unregister()         // device_wakeup_disable
dev->power.can_wakeup = false;     // set_wakeup_capable
return 0;
```

**Who calls it:**
الـ probe functions في drivers زي keyboard drivers وnetwork adapters وpower button drivers.

---

#### `devm_device_init_wakeup`

```c
static inline int devm_device_init_wakeup(struct device *dev);
```

**الـ function دي بتعمل إيه:**
نسخة الـ resource-managed من `device_init_wakeup(dev, true)`. بعد ما بتفعّل الـ wakeup، بتسجّل cleanup action عن طريق `devm_add_action_or_reset()` عشان `device_disable_wakeup()` يتنادى أوتوماتيك عند الـ unbind أو الـ remove.

**Parameters:**
- `dev` — الـ device المراد تهيئته وإدارته بشكل managed.

**Return value:**
- `0` عند النجاح.
- Error code لو فشل `device_init_wakeup` أو `devm_add_action_or_reset`.
- لو الـ `devm_add_action_or_reset` فشل، بتعمل rollback أوتوماتيك وتعطّل الـ wakeup.

**Key details:**
- بتستخدم `device_disable_wakeup` (الـ static function) كـ cleanup callback.
- الفايدة: مش محتاج تكتب `device_init_wakeup(dev, false)` في الـ remove function.
- الـ `devm` mechanism بتضمن الـ cleanup حتى لو الـ probe فشل في مرحلة لاحقة.

**Who calls it:**
الـ modern drivers اللي بتستخدم الـ devm pattern في الـ probe.

---

#### `device_disable_wakeup` (internal)

```c
static void device_disable_wakeup(void *dev);
```

**الـ function دي بتعمل إيه:**
wrapper صغير بيناديه `devm_add_action_or_reset` كـ cleanup callback. بيحوّل الـ `void *` pointer لـ `struct device *` ثم بيناديه على `device_init_wakeup(dev, false)`.

**Key details:**
- `static` بدون `inline` — ده مقصود عشان تاخد عنوانها لـ `devm_add_action_or_reset`.
- مش مفروض يتنادى عليها مباشرة من الـ drivers.

---

### الفئة التانية: Query & State Functions

الـ functions دي بتقرأ الحالة الحالية للـ device بخصوص الـ wakeup. كلها `static inline` وبتقرأ مباشرة من `dev->power.*` من غير أي locking — ده safe لأن الـ fields دي بتتغير بشكل محكوم فقط.

---

#### `device_can_wakeup`

```c
static inline bool device_can_wakeup(struct device *dev);
```

**الـ function دي بتعمل إيه:**
بترجع قيمة `dev->power.can_wakeup` مباشرة. ده الـ hardware capability flag — بيعبّر عن إمكانية الـ hardware وليس الـ policy.

**Parameters:**
- `dev` — الـ device المراد الاستعلام عنه.

**Return value:**
- `true` لو الـ device hardware يدعم الـ wakeup.
- `false` لو لأ.

**Key details:**
- بيتوفر في كل الـ configs (مع وبدون `CONFIG_PM_SLEEP`).
- لا lock مطلوب — الـ flag ده بيتعيّن فقط عبر `device_set_wakeup_capable()` وده protected.

---

#### `device_may_wakeup`

```c
static inline bool device_may_wakeup(struct device *dev);
```

**الـ function دي بتعمل إيه:**
بتتحقق من شرطين معاً: الـ hardware قادر (`can_wakeup`) **والـ** wakeup مفعّل فعلاً (يعني `dev->power.wakeup != NULL`). ده الـ function الأهم للـ PM core لأنه السؤال الحقيقي: "هل المفروض أفعّل الـ wakeup interrupt لهذا الـ device قبل الـ suspend؟"

**Parameters:**
- `dev` — الـ device المراد الاستعلام عنه.

**Return value:**
- `true` لو الشرطين متحققين.
- `false` لو أي منهم مش متحقق.

**Key details:**
- في `CONFIG_PM_SLEEP`: `dev->power.wakeup` بيكون pointer للـ `wakeup_source`، فالـ `!!` بتحوّله لـ bool.
- في `!CONFIG_PM_SLEEP`: بتقرأ `can_wakeup && should_wakeup` — الـ `should_wakeup` بدل الـ `wakeup` pointer.
- الـ bus suspend callbacks (مثلاً `pci_pm_suspend`) بتنادي على الـ function دي قبل ما تحدد هل تفعّل الـ PME.

---

#### `device_wakeup_path` و `device_awake_path`

```c
static inline bool device_wakeup_path(struct device *dev);
static inline bool device_awake_path(struct device *dev);
```

**الـ function دي بتعمل إيه:**
بترجع `dev->power.wakeup_path`. ده flag بيعبّر عن إن الـ device ده يقع في **مسار** الـ wakeup signal من source وصولاً للـ CPU — حتى لو هو نفسه مش الـ wakeup source. مثال: bridge devices.

`device_awake_path` هي مجرد alias لـ `device_wakeup_path` — موجودة لأسباب تاريخية وتوحيد الـ naming.

**Return value:**
- `true` لو الـ device في مسار الـ wakeup.
- `false` (hardcoded) في `!CONFIG_PM_SLEEP`.

---

#### `device_out_band_wakeup`

```c
static inline bool device_out_band_wakeup(struct device *dev);
```

**الـ function دي بتعمل إيه:**
بترجع `dev->power.out_band_wakeup`. الـ **out-of-band wakeup** معناه إن الـ device عنده آلية wakeup خارج الـ standard PM signaling — زي الـ dedicated wakeup pin المتصل بمسار منفصل عن الـ main bus.

**Return value:**
- `true` لو الـ device يدعم out-of-band wakeup.
- `false` hardcoded في `!CONFIG_PM_SLEEP`.

---

### الفئة التالتة: Path Marking Functions

---

#### `device_set_wakeup_path` و `device_set_awake_path`

```c
static inline void device_set_wakeup_path(struct device *dev);
static inline void device_set_awake_path(struct device *dev);
```

**الـ function دي بتعمل إيه:**
بتعمل set على `dev->power.wakeup_path = true`. الـ PM core بتنادي دول على الـ ancestors للـ wakeup source device عشان تعرف إنها لازم تفضل شغّالة أثناء الـ suspend.

**Key details:**
- بتتنادى في الـ suspend path عندما الـ `device_may_wakeup(dev)` يرجع `true`.
- الـ `device_set_awake_path` هي alias بالظبط لـ `device_set_wakeup_path`.
- في `!CONFIG_PM_SLEEP`: no-op.

---

#### `device_set_out_band_wakeup`

```c
static inline void device_set_out_band_wakeup(struct device *dev);
```

**الـ function دي بتعمل إيه:**
بتعمل set على `dev->power.out_band_wakeup = true`. بيتنادى عليها من الـ driver أو bus layer لما يكتشف إن الـ device عنده out-of-band wakeup capability.

**Key details:**
- في `!CONFIG_PM_SLEEP`: no-op.

---

### الفئة الرابعة: Runtime Wakeup Event Functions

دي القلب الحقيقي للـ wakeup subsystem — الـ functions دي هي اللي بتشغّل وتهدّي وتشير للـ wakeup events أثناء الـ runtime. الـ PM core بيستخدمها لمنع أو السماح بالـ suspend.

---

#### `__pm_stay_awake`

```c
void __pm_stay_awake(struct wakeup_source *ws);
```

**الـ function دي بتعمل إيه:**
بتنشّط الـ `wakeup_source` مباشرة — ده يعني إن الـ kernel مش هيعدي للـ suspend طالما الـ `ws` ده active. بتزوّد الـ `active_count`، بتسجّل الـ `last_time`، وبتعمل update للـ wakeup counter العالمي.

**Parameters:**
- `ws` — الـ `wakeup_source` المراد تنشيطه. ممكن يكون NULL (بتتحقق داخلياً).

**Return value:** `void`.

**Key details:**
- بتأخذ `ws->lock` (spinlock) أثناء العملية.
- لو الـ `ws` كان active بالفعل، مش بتعمل حاجة تانية (no double-counting).
- بتعمل `ws->active = true` وبتضيف `ws` للـ global `wakeup_count`.
- الـ prefix `__` معناه إن الـ caller مسؤول عن الـ validation — استخدم `pm_stay_awake()` أحسن للـ devices.

**Who calls it:**
`pm_stay_awake()` والـ interrupt handlers المرتبطة بالـ wakeup sources مباشرة.

---

#### `pm_stay_awake`

```c
void pm_stay_awake(struct device *dev);
```

**الـ function دي بتعمل إيه:**
wrapper حول `__pm_stay_awake()` بتاخد الـ `dev->power.wakeup` وتناديه عليه. أنسب للـ drivers لأنها بتتعامل مع الـ `device` مباشرة.

**Parameters:**
- `dev` — الـ device المراد إبقاؤه يقظاً.

**Return value:** `void`.

**Key details:**
- بتأخذ `dev->power.lock` (spinlock) قبل الوصول لـ `dev->power.wakeup` عشان تضمن إنه مش بيتحذف في نفس الوقت.
- لو `dev->power.wakeup == NULL`، بترجع بدون حاجة.

**Who calls it:**
الـ interrupt handlers، الـ input subsystem (keyboard/mouse events)، الـ USB core.

---

#### `__pm_relax`

```c
void __pm_relax(struct wakeup_source *ws);
```

**الـ function دي بتعمل إيه:**
بتهدّئ الـ `wakeup_source` — بتعمل `ws->active = false` وبتعمل update للـ timing statistics (`total_time`، `max_time`)، وبتنقص الـ global wakeup counter. لو الـ counter وصل للـ صفر، ممكن يبدأ الـ autosleep.

**Parameters:**
- `ws` — الـ `wakeup_source` المراد تهدئته.

**Return value:** `void`.

**Key details:**
- بتأخذ `ws->lock` أثناء العملية.
- لو الـ `ws` مش active، بترجع فوراً.
- بتنادي على `pm_autosleep_wakeup_source_deactivated()` لو relevant.
- بتحذف الـ pending timer لو كان في `ws->timer` set.

**Who calls it:**
`pm_relax()` وبعض الـ internal paths بعد انتهاء معالجة الـ wakeup event.

---

#### `pm_relax`

```c
void pm_relax(struct device *dev);
```

**الـ function دي بتعمل إيه:**
wrapper حول `__pm_relax()` للـ device-level interface. نفس فكرة `pm_stay_awake` بالنسبة لـ `__pm_stay_awake`.

**Parameters:**
- `dev` — الـ device اللي انتهى الـ wakeup event بتاعه.

**Key details:**
- بتأخذ `dev->power.lock` قبل الوصول لـ `dev->power.wakeup`.
- الـ pattern الشائع: `pm_stay_awake(dev)` في بداية الـ ISR، `pm_relax(dev)` بعد ما تخلص من معالجة الـ event.

---

#### `pm_wakeup_ws_event`

```c
void pm_wakeup_ws_event(struct wakeup_source *ws, unsigned int msec, bool hard);
```

**الـ function دي بتعمل إيه:**
بتشير لـ wakeup event على `wakeup_source` معين. لو `msec > 0` بتنشّط الـ `ws` لمدة `msec` ميلي ثانية عن طريق timer، وبعدين بتعمل relax أوتوماتيك. لو `hard = true` بتضمن إيقاظ النظام حتى لو كان في منتصف الـ suspend.

**Parameters:**
- `ws` — الـ `wakeup_source` اللي حصل فيه الـ event.
- `msec` — مدة بقاء الـ source نشطاً بالـ milliseconds. `0` يعني تنشيط فوري وتهدئة فورية (one-shot event signal).
- `hard` — لو `true`، بتعمل abort للـ suspend الجاري حتى لو كان المطلوب فقط autosleep.

**Return value:** `void`.

**Key details:**
- بتزوّد `ws->event_count` دايماً.
- لو `msec > 0`: بتنشّط الـ `ws`، بتعمل `mod_timer()` للـ `ws->timer`.
- الـ timer callback بينادي على `__pm_relax()` أوتوماتيك.
- الـ `hard = true` بتتمرر لـ `pm_wakeup_pending()` اللي بتقاطع الـ suspend path.

**Pseudocode flow:**
```c
ws->event_count++;
if (msec > 0) {
    __pm_stay_awake(ws);
    mod_timer(&ws->timer, jiffies + msecs_to_jiffies(msec));
} else {
    // signal only, no hold
    pm_wakeup_pending();  // abort suspend if needed
}
if (hard) {
    // force abort even deep suspend
}
```

---

#### `pm_wakeup_dev_event`

```c
void pm_wakeup_dev_event(struct device *dev, unsigned int msec, bool hard);
```

**الـ function دي بتعمل إيه:**
wrapper حول `pm_wakeup_ws_event()` للـ device-level — بتاخد الـ `dev->power.wakeup` وتناديه عليه.

**Parameters:**
- `dev` — الـ device صاحب الـ event.
- `msec` — نفس الـ `pm_wakeup_ws_event`.
- `hard` — نفس الـ `pm_wakeup_ws_event`.

**Key details:**
- بتأخذ `dev->power.lock` قبل الوصول لـ `dev->power.wakeup`.
- في `!CONFIG_PM_SLEEP`: no-op كاملة.

---

#### `__pm_wakeup_event`

```c
static inline void __pm_wakeup_event(struct wakeup_source *ws, unsigned int msec);
```

**الـ function دي بتعمل إيه:**
shortcut لـ `pm_wakeup_ws_event(ws, msec, false)` — soft wakeup event بدون الـ hard abort flag.

**Key details:**
- inline wrapper بدون أي logic إضافية.
- مناسب للـ normal wakeup events اللي مش محتاجة hard abort.

---

#### `pm_wakeup_event`

```c
static inline void pm_wakeup_event(struct device *dev, unsigned int msec);
```

**الـ function دي بتعمل إيه:**
shortcut لـ `pm_wakeup_dev_event(dev, msec, false)` — الـ API الأبسط والأكتر شيوعاً في الـ drivers لـ normal wakeup events.

**Who calls it:**
أغلب الـ drivers اللي محتاجة ترفع wakeup event — input drivers، network drivers، USB drivers.

---

#### `pm_wakeup_hard_event`

```c
static inline void pm_wakeup_hard_event(struct device *dev);
```

**الـ function دي بتعمل إيه:**
shortcut لـ `pm_wakeup_dev_event(dev, 0, true)` — hard wakeup event فوري بدون timeout. بتضمن abort الـ suspend حتى لو كان في مرحلة متأخرة.

**Key details:**
- `msec = 0` يعني إن الـ source مش هيتنشّط لمدة معينة — بس الـ `hard = true` بيضمن إيقاظ النظام.
- مناسب للـ critical events زي الـ power button أو emergency signals.

---

### الفئة الخامسة: Iteration Functions

هذه الـ functions بتسمح بالـ traversal على قائمة الـ wakeup sources الكاملة، وده مفيد للـ debugging والـ sysfs/debugfs reporting.

---

#### `wakeup_sources_read_lock`

```c
int wakeup_sources_read_lock(void);
```

**الـ function دي بتعمل إيه:**
بتأخذ الـ read lock على قائمة الـ wakeup sources. الـ implementation بتستخدم `srcu_read_lock()` (Sleepable RCU) عشان تسمح بالـ sleeping أثناء الـ iteration في سياق الـ sysfs reads.

**Return value:**
- `int idx` — الـ SRCU index المطلوب تمريره لـ `wakeup_sources_read_unlock()`.

**Key details:**
- لازم يتزاوج مع `wakeup_sources_read_unlock(idx)`.
- الـ SRCU مناسب هنا لأن الـ sysfs readers ممكن يحتاجوا يـ sleep.

---

#### `wakeup_sources_read_unlock`

```c
void wakeup_sources_read_unlock(int idx);
```

**الـ function دي بتعمل إيه:**
بتحرر الـ SRCU read lock اللي أخدته `wakeup_sources_read_lock()`.

**Parameters:**
- `idx` — الـ index اللي رجعه `wakeup_sources_read_lock()`.

---

#### `wakeup_sources_walk_start`

```c
struct wakeup_source *wakeup_sources_walk_start(void);
```

**الـ function دي بتعمل إيه:**
بترجع أول `wakeup_source` في القائمة، أو `NULL` لو القائمة فاضية. بتستخدم `list_entry_rcu()` داخلياً للوصول الآمن.

**Return value:**
- Pointer للأول `wakeup_source`.
- `NULL` لو القائمة فاضية.

---

#### `wakeup_sources_walk_next`

```c
struct wakeup_source *wakeup_sources_walk_next(struct wakeup_source *ws);
```

**الـ function دي بتعمل إيه:**
بترجع الـ `wakeup_source` التالي في القائمة بعد `ws`، أو `NULL` لو وصلنا للنهاية.

**Parameters:**
- `ws` — الـ `wakeup_source` الحالي في الـ iteration.

---

#### `for_each_wakeup_source` (macro)

```c
#define for_each_wakeup_source(ws) \
    for ((ws) = wakeup_sources_walk_start(); \
         (ws); \
         (ws) = wakeup_sources_walk_next((ws)))
```

**الـ macro ده بيعمل إيه:**
بيوفر syntax نظيف للـ iteration على كل الـ registered wakeup sources. بيبدأ بـ `walk_start()` ومستمر لحد ما `walk_next()` يرجع `NULL`.

**مثال استخدام:**
```c
struct wakeup_source *ws;
int idx;

idx = wakeup_sources_read_lock();
for_each_wakeup_source(ws) {
    pr_info("wakeup source: %s, active: %d\n", ws->name, ws->active);
}
wakeup_sources_read_unlock(idx);
```

**Key details:**
- لازم يكون محاط بـ `wakeup_sources_read_lock()/unlock()` للاستخدام الآمن.
- الـ `ws` pointer آمن طالما الـ read lock محجوز.

---

### العلاقة بين الـ Functions — Flow Diagrams

#### Flow تفعيل الـ Wakeup في Probe

```
driver probe()
    └─► device_init_wakeup(dev, true)
            ├─► device_set_wakeup_capable(dev, true)
            │       └─► dev->power.can_wakeup = true
            └─► device_wakeup_enable(dev)
                    └─► wakeup_source_register(dev, name)
                            ├─► wakeup_source_create()   [alloc + init]
                            └─► wakeup_source_add()      [add to global list]
                                    └─► dev->power.wakeup = ws
```

#### Flow حدث الـ Wakeup (ISR Context)

```
IRQ Handler
    └─► pm_stay_awake(dev)
            ├─► spin_lock(dev->power.lock)
            ├─► ws = dev->power.wakeup
            └─► __pm_stay_awake(ws)
                    ├─► ws->active = true
                    ├─► ws->active_count++
                    └─► update global wakeup_count

[event processing...]

work_done:
    └─► pm_relax(dev)
            └─► __pm_relax(ws)
                    ├─► ws->active = false
                    ├─► update total_time, max_time
                    └─► decrement global wakeup_count
                            └─► [if count == 0] → autosleep may proceed
```

#### Flow الـ suspend check

```
PM core tries to suspend
    └─► pm_wakeup_pending()
            └─► checks global wakeup_count > 0?
                    ├─► YES: abort suspend
                    └─► NO: proceed to suspend
                            └─► for each device:
                                    └─► device_may_wakeup(dev)?
                                            └─► enable hardware wakeup interrupt
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بالـ wakeup sources

الـ kernel بيعرض معلومات الـ **wakeup sources** تحت `/sys/kernel/debug/wakeup_sources` (عبر debugfs).

```bash
# Mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# اقرأ كل الـ wakeup sources المسجلة
cat /sys/kernel/debug/wakeup_sources
```

**مثال على الـ output:**

```
name            active_count    event_count    wakeup_count    expire_count    \
active_since    total_time      max_time        last_change     prevent_sleep_time
eventpoll       0               7              0               0               \
0               0               0              0               0
alarmtimer      1               23             5               0               \
1500            320000          15000          1678900          180000
```

**طريقة تفسير الأعمدة:**

| العمود | المعنى |
|---|---|
| `name` | اسم الـ `wakeup_source` (حقل `ws->name`) |
| `active_count` | عدد مرات استدعاء `__pm_stay_awake()` |
| `event_count` | عدد الـ wakeup events المُشار إليها |
| `wakeup_count` | كم مرة أوقف suspend |
| `expire_count` | عدد مرات انتهاء الـ timer قبل الـ relax |
| `active_since` | وقت بدأ `ws->active` (بالـ microseconds) |
| `total_time` | مجموع وقت النشاط (`ws->total_time`) |
| `max_time` | أقصى فترة نشاط متواصل (`ws->max_time`) |
| `prevent_sleep_time` | الوقت الكلي اللي منع فيه autosleep |

---

#### 2. مدخلات الـ sysfs المتعلقة بالـ wakeup

كل device مسجّل فيه `wakeup_source` بيظهر تحت `/sys/bus/.../devices/<dev>/power/`:

```bash
# هل الـ device يقدر يوقظ الجهاز؟
cat /sys/devices/platform/soc/usb0/power/wakeup
# output: enabled | disabled

# عدد الـ wakeup events اللي أطلقها الـ device
cat /sys/devices/platform/soc/usb0/power/wakeup_count

# الوقت اللي كان فيه الـ wakeup source نشط (بالـ ms)
cat /sys/devices/platform/soc/usb0/power/wakeup_active_count
cat /sys/devices/platform/soc/usb0/power/wakeup_total_time_ms
cat /sys/devices/platform/soc/usb0/power/wakeup_max_time_ms
cat /sys/devices/platform/soc/usb0/power/wakeup_last_time_ms

# هل الـ device في مسار الـ wakeup؟
cat /sys/devices/platform/soc/usb0/power/wakeup_path

# تفعيل/تعطيل الـ wakeup من userspace
echo enabled  > /sys/devices/platform/soc/usb0/power/wakeup
echo disabled > /sys/devices/platform/soc/usb0/power/wakeup
```

**ملاحظة:** الـ sysfs attributes دي بتتعامل مع `dev->power.wakeup` اللي هو بالضبط الـ `wakeup_source` المرتبط بالـ device عبر `device_wakeup_enable()`.

---

#### 3. الـ ftrace — tracepoints وأحداث الـ PM

الـ kernel بيوفر tracepoints جاهزة في `trace/events/power.h`:

```bash
# اعرض كل الـ events المتعلقة بالـ PM
ls /sys/kernel/debug/tracing/events/power/

# فعّل الأحداث المهمة للـ wakeup
echo 1 > /sys/kernel/debug/tracing/events/power/wakeup_source_activate/enable
echo 1 > /sys/kernel/debug/tracing/events/power/wakeup_source_deactivate/enable
echo 1 > /sys/kernel/debug/tracing/events/power/suspend_resume/enable

# فعّل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# انتظر أو استنى حدث، ثم اقرأ
cat /sys/kernel/debug/tracing/trace
```

**مثال على output:**

```
     kworker/0:2-45  [000] d... 12345.678901: wakeup_source_activate: alarmtimer state=1
     kworker/0:2-45  [000] d... 12345.679100: wakeup_source_deactivate: alarmtimer state=0
```

**تفسير:** كل سطر بيوضح اسم الـ wakeup source واللحظة اللي اتنشط أو اترخّ فيها.

```bash
# لو عايز تتبع مسار suspend بالكامل
echo 'suspend_resume' > /sys/kernel/debug/tracing/set_event
echo function > /sys/kernel/debug/tracing/current_tracer
echo pm_wakeup_ws_event > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. الـ printk والـ dynamic debug

**تفعيل dynamic debug لسب النظام wakeup:**

```bash
# فعّل كل رسائل drivers/base/power/wakeup.c
echo 'file drivers/base/power/wakeup.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل ملفات pm
echo 'file drivers/base/power/* +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق مما تم تفعيله
cat /sys/kernel/debug/dynamic_debug/control | grep wakeup
```

**رفع مستوى الـ loglevel للـ PM:**

```bash
# رفع loglevel بحيث تشوف رسائل pr_debug
echo 8 > /proc/sys/kernel/printk

# أو من cmdline
# kernel boot param: ignore_loglevel
```

**مثال على رسائل تظهر بعد التفعيل:**

```
PM: wakeup_source_activate: alarmtimer activated, event_count=24
PM: __pm_relax: alarmtimer deactivated
PM: check_wakeup_irqs: wakeup irq detected, aborting suspend
```

---

#### 5. الـ kernel config options للـ debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_PM_SLEEP` | يفعّل subsystem الـ wakeup sources أصلاً |
| `CONFIG_PM_SLEEP_DEBUG` | يفعّل رسائل debug إضافية في مسار الـ suspend/resume |
| `CONFIG_PM_DEBUG` | يفعّل `/sys/power/pm_debug_messages` وغيرها |
| `CONFIG_PM_ADVANCED_DEBUG` | يفعّل معلومات تفصيلية أكتر في sysfs |
| `CONFIG_PM_TEST_SUSPEND` | يسمح باختبار الـ suspend بدون ما ينام الجهاز فعلاً |
| `CONFIG_WAKEUP_SOURCE_DEBUGFS` | يعرض `wakeup_sources` في debugfs |
| `CONFIG_DEBUG_SPINLOCK` | يكشف أخطاء الـ spinlock في `ws->lock` |
| `CONFIG_LOCKDEP` | يكشف deadlocks في قفل الـ wakeup sources |
| `CONFIG_FTRACE` | يفعّل الـ tracepoints بتاعة power |
| `CONFIG_DYNAMIC_DEBUG` | يتيح `pr_debug` at runtime |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_PM|CONFIG_WAKEUP'
```

---

#### 6. أدوات خاصة بالـ subsystem

**أداة `pm-utils` / `systemd`:**

```bash
# اعرض آخر wakeup reason بعد الاستيقاظ
cat /sys/power/pm_wakeup_irq

# اعرض الـ IRQ اللي أيقظ الجهاز
cat /proc/interrupts | grep -i wakeup

# فحص حالة الـ autosleep
cat /sys/power/autosleep

# فحص الـ wakeup count قبل وبعد suspend
cat /sys/power/wakeup_count
```

**أداة `powertop`:**

```bash
# بتعرض الـ wakeup sources وعدد interrupts كل ثانية
powertop --time=10
```

**أداة `rtcwake`** لاختبار wakeup من RTC:

```bash
# نم لمدة 10 ثواني ثم اصحى
rtcwake -m mem -s 10
```

---

#### 7. جدول الـ error messages الشائعة

| رسالة في dmesg | المعنى | الحل |
|---|---|---|
| `PM: Wakeup pending, cannot sleep` | في wakeup event حصل أثناء أو قبل الـ suspend مباشرة | افحص الـ wakeup_sources النشطة بـ `cat /sys/kernel/debug/wakeup_sources` |
| `PM: Some devices failed to suspend` | device فشل في الـ suspend callback | افحص dmesg لاسم الـ device الفاشل |
| `PM: Device X not suspending, no callback` | الـ device مفيش له suspend callback | عادي في بعض الـ drivers، افحص لو محتاج تضيف `.suspend` |
| `PM: Wakeup IRQ X triggered during suspend` | interrupt وصل أثناء الـ suspend | عادي، بس لو متكرر افحص الـ IRQ masking |
| `PM: wakeup source already active: <name>` | الـ wakeup source اتنشط مرتين بدون relax | bug في الـ driver — مكالمة `__pm_stay_awake()` بدون مقابلها `__pm_relax()` |
| `genirq: Flags mismatch irq X` | mismatch في flags الـ wake IRQ | راجع `dev_pm_set_wake_irq()` وتأكد من الـ IRQ flags |
| `PM: Cannot get wakeup_count` | race condition في قراءة الـ wakeup count | أعد المحاولة أو افحص الـ suspend logic |

---

#### 8. أماكن استراتيجية لـ `dump_stack()` و `WARN_ON()`

```c
/* في __pm_stay_awake() — للتحقق إن الـ ws مش NULL */
void __pm_stay_awake(struct wakeup_source *ws)
{
    WARN_ON(!ws);  /* يكشف لو driver نسي يسجل الـ wakeup source */
    ...
}

/* في __pm_relax() — للتحقق من double-relax */
void __pm_relax(struct wakeup_source *ws)
{
    if (!ws->active) {
        WARN_ON(1);  /* relax بدون activate -- bug في الـ driver */
        dump_stack();
    }
    ...
}

/* في wakeup_source_register() — للتحقق من allocation */
struct wakeup_source *wakeup_source_register(struct device *dev,
                                              const char *name)
{
    struct wakeup_source *ws;
    ws = kzalloc(sizeof(*ws), GFP_KERNEL);
    WARN_ON(!ws);  /* OOM في وقت التسجيل */
    ...
}

/* في device_wakeup_enable() — تحقق إن الـ device قادر يكون wakeup */
int device_wakeup_enable(struct device *dev)
{
    WARN_ON(!device_can_wakeup(dev));
    /* dump_stack هنا يوضح من استدعى enable بدون ما يعمل set_wakeup_capable أولاً */
    ...
}
```

---

### Hardware Level

---

#### 1. التحقق إن الـ hardware state مطابق للـ kernel state

```bash
# تحقق إن الـ wakeup source في kernel مطابق للـ hardware
# مثال: GPIO wakeup
cat /sys/class/gpio/gpio<N>/value       # قيمة الـ GPIO الحالية
cat /sys/class/gpio/gpio<N>/edge        # rising/falling/both

# تحقق إن الـ interrupt enabled في الـ hardware
cat /proc/interrupts | grep <irq_num>

# مثال: USB wakeup
cat /sys/bus/usb/devices/usb1/power/wakeup
lsusb -v | grep -i wakeup
```

**مطابقة الـ kernel state مع الـ hardware:**

```
kernel: ws->active == true
  ↕ يجب أن يكون مطابقاً
hardware: IRQ line high أو device يرسل wakeup signal
```

---

#### 2. Register dump بالأدوات

```bash
# devmem2 — قراءة register مباشرة من userspace (يحتاج CONFIG_DEVMEM=y)
# مثال: اقرأ Wake Status Register في SoC
devmem2 0x40020004 w       # address بتاعة الـ wakeup status register

# /dev/mem مع python
python3 -c "
import mmap, os, struct
with open('/dev/mem', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 4096, offset=0x40020000)
    val = struct.unpack('<I', mm[4:8])[0]
    print(hex(val))
    mm.close()
"

# io tool (من package iotools)
io -4 -r 0x40020004         # قراءة 32-bit register

# لو شغال على ARM وفيك PMU/PMIC
i2cdump -y 1 0x68           # dump registers الـ PMIC على I2C bus 1
```

---

#### 3. Logic Analyzer و Oscilloscope

**نقاط القياس المهمة:**

```
┌─────────────────────────────────────────────────────┐
│  أماكن وصل الـ probe                                │
├─────────────────────────────────────────────────────┤
│  1. WAKEUP_N pin على الـ SoC                        │
│  2. IRQ line بين الـ device والـ SoC               │
│  3. Power rail للـ device (3.3V / 1.8V)             │
│  4. CLK line لو الـ device بيستخدم clock gating     │
└─────────────────────────────────────────────────────┘
```

**نصائح عملية:**

- استخدم **Logic Analyzer** (زي Saleae) لالتقاط الـ IRQ pulse وتوقيته بالنسبة للـ suspend/resume
- على الـ Oscilloscope: ابحث عن glitches على الـ wakeup line قبل الـ suspend بـ milliseconds — دي بتسبب spurious wakeups
- وصّل trigger على الـ WAKEUP_N line وابحث عن الـ latency بين pulse الـ hardware وأول رسالة في dmesg

---

#### 4. مشاكل الـ hardware الشائعة وأنماطها في الـ kernel log

| المشكلة | النمط في dmesg | التشخيص |
|---|---|---|
| **Spurious wakeup** من GPIO | `PM: Wakeup pending, cannot sleep` يتكرر باستمرار | افحص الـ GPIO pull-up/pull-down، قيس الـ signal بـ oscilloscope |
| **IRQ لا يصل** للـ SoC | الجهاز مش بيصحى رغم وجود حدث | تحقق من الـ IRQ routing في الـ SoC، افحص الـ IOMUX/pinmux |
| **Power rail** ينزل قبل الأوان | `PM: Some devices failed to resume` مع timeout | راقب الـ power sequence بـ oscilloscope |
| **Shared IRQ** مش بيتمسك | الـ wakeup IRQ بيشارك ويتفوّت | افحص `cat /proc/interrupts` وتحقق من shared handler |
| **RTC wakeup** مش شغال | الجهاز مش بيصحى من الـ RTC | تحقق من battery الـ RTC وافحص `hwclock -r` |

---

#### 5. Device Tree Debugging

```bash
# تحقق إن الـ DT بيوصف الـ wakeup-source صح
dtc -I fs /sys/firmware/devicetree/base | grep -A5 wakeup

# أو استخدم fdtget
fdtget /boot/dtb /soc/usb@101c0000 wakeup-source

# فحص الـ interrupts في الـ DT node
cat /sys/firmware/devicetree/base/soc/usb@101c0000/interrupts | xxd
```

**مثال DT node صحيح لـ device يدعم wakeup:**

```dts
usb_phy: usb@101c0000 {
    compatible = "vendor,usb-phy";
    reg = <0x101c0000 0x1000>;
    interrupts = <GIC_SPI 32 IRQ_TYPE_LEVEL_HIGH>;
    interrupt-parent = <&gic>;

    /* هنا بيقول للـ kernel إن الـ device ده wakeup source */
    wakeup-source;
};
```

**تحقق إن الـ DT مطابق للـ hardware:**

```bash
# تأكد إن رقم الـ IRQ في DT نفسه في /proc/interrupts
grep -i usb /proc/interrupts

# تحقق من الـ pinmux — هل الـ wakeup pin مفعّل؟
cat /sys/kernel/debug/pinctrl/pinctrl-handles
```

---

### Practical Commands

---

#### مجموعة أوامر جاهزة للنسخ والتشغيل

**1. عرض كل الـ wakeup sources مع التفاصيل:**

```bash
#!/bin/bash
echo "=== Active Wakeup Sources ==="
cat /sys/kernel/debug/wakeup_sources | column -t

echo ""
echo "=== Last Wakeup IRQ ==="
cat /sys/power/pm_wakeup_irq 2>/dev/null || echo "not available"

echo ""
echo "=== Wakeup-Enabled Devices ==="
for f in /sys/devices/*/power/wakeup /sys/devices/*/*/power/wakeup; do
    val=$(cat "$f" 2>/dev/null)
    [ "$val" = "enabled" ] && echo "$f"
done
```

**2. مراقبة الـ wakeup events في real-time بالـ ftrace:**

```bash
#!/bin/bash
TDIR=/sys/kernel/debug/tracing

echo nop > $TDIR/current_tracer
echo 0 > $TDIR/tracing_on
echo > $TDIR/trace

echo 1 > $TDIR/events/power/wakeup_source_activate/enable
echo 1 > $TDIR/events/power/wakeup_source_deactivate/enable
echo 1 > $TDIR/events/power/suspend_resume/enable

echo 1 > $TDIR/tracing_on
echo "Tracing started. Press Ctrl+C to stop..."
trap "echo 0 > $TDIR/tracing_on; cat $TDIR/trace" INT
sleep 30
echo 0 > $TDIR/tracing_on
cat $TDIR/trace
```

**مثال output:**

```
     kworker/u4:2-123 [001] d... 5432.100: wakeup_source_activate: alarmtimer state=1
     kworker/u4:2-123 [001] d... 5432.115: wakeup_source_deactivate: alarmtimer state=0
          <idle>-0     [000] d... 5432.200: suspend_resume: suspend_enter[3] begin
          <idle>-0     [000] d... 5432.205: suspend_resume: suspend_enter[3] end
```

**3. تشخيص سريع لمشكلة "الجهاز مش بينام":**

```bash
#!/bin/bash
echo "=== Checking who's preventing sleep ==="

# عرض active wakeup sources فقط
awk 'NR==1 || $5 > 0 {print}' /sys/kernel/debug/wakeup_sources | column -t

echo ""
echo "=== wakeup_count ==="
cat /sys/power/wakeup_count

echo ""
echo "=== pm_abort_suspend indicator ==="
# لو الرقم بيتغير باستمرار، في wakeup events نشطة
prev=$(cat /sys/power/wakeup_count)
sleep 1
curr=$(cat /sys/power/wakeup_count)
echo "Before: $prev  After: $curr"
[ "$curr" -gt "$prev" ] && echo "WARNING: wakeup events are happening!" || echo "No new events"
```

**4. اختبار suspend مع مراقبة الـ wakeup:**

```bash
#!/bin/bash
echo "=== Wakeup count before suspend ==="
COUNT_BEFORE=$(cat /sys/power/wakeup_count)
echo $COUNT_BEFORE

# كتابة الـ count للـ kernel لو هو نفس الـ count الحالي يسمح بالـ suspend
echo $COUNT_BEFORE > /sys/power/wakeup_count && \
    echo mem > /sys/power/state

echo "=== Wakeup count after resume ==="
cat /sys/power/wakeup_count

echo "=== Last wakeup IRQ ==="
cat /sys/power/pm_wakeup_irq
```

**5. dynamic debug تفعيل وتعطيل:**

```bash
# تفعيل
echo 'file drivers/base/power/wakeup.c +pflmt' \
    > /sys/kernel/debug/dynamic_debug/control

# تعطيل
echo 'file drivers/base/power/wakeup.c -p' \
    > /sys/kernel/debug/dynamic_debug/control

# تأكيد
grep 'wakeup.c' /sys/kernel/debug/dynamic_debug/control | head -20
```

**6. تفسير output الـ wakeup_sources بالتفصيل:**

```bash
# مثال output مع شرح
cat /sys/kernel/debug/wakeup_sources
```

```
name          active_count  event_count  wakeup_count  expire_count  \
active_since  total_time    max_time     last_change    prevent_sleep_time

# السطر ده يعني:
# - "NETLINK" استيقظ 3 مرات
# - منع suspend مرة واحدة (wakeup_count=1)
# - أطول فترة نشاط كانت 5000 microseconds
NETLINK       3             3            1             0             \
0             12000         5000         98765432       8000
```

**7. فحص wake IRQ مرتبط بـ device:**

```bash
# اعرض الـ wake IRQ المخصص لكل device
for dev in /sys/bus/*/devices/*/power; do
    irq_file="$dev/../wakeirq"
    [ -f "$irq_file" ] && echo "$dev: $(cat $irq_file)"
done

# أو من kernel log بعد boot
dmesg | grep -i 'wake.*irq\|irq.*wake' | head -20
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الجهاز مش بيصحى من Sleep

#### العنوان
UART wakeup مش شغال على industrial gateway بيشتغل بـ RK3562

#### السياق
شركة بتعمل industrial IoT gateway بيشتغل على **RK3562**. الـ gateway بيدخل `suspend-to-RAM` بعد فترة خمول، والمفروض أي بيانات جاية على **UART2** (connected to a 4G modem) تصحيه. المنتج في مرحلة bring-up والـ hardware team قالت إن الـ modem بيبعت بيانات كل دقيقة.

#### المشكلة
الـ gateway بيدخل suspend تمام، بس مش بيصحى أبدًا. الـ modem بيبعت بيانات، الـ RX line بتتحرك، بس الـ system فاضل نايم.

#### التحليل
المهندس راح يشوف `/sys/kernel/debug/wakeup_sources` وقفل:

```bash
cat /sys/kernel/debug/wakeup_sources | grep uart
# مفيش نتيجة — مفيش wakeup_source مسجل للـ UART
```

راح للـ driver الـ RK3562 UART (based on 8250/pl011). لقى إن الـ driver بيعمل:

```c
/* في الـ probe function */
platform_set_drvdata(pdev, sport);
/* مفيش استدعاء لـ device_init_wakeup */
```

يعني الـ `dev->power.can_wakeup` فاضل `false`. لما بيجي وقت الـ suspend، الـ PM core بيشوف:

```c
static inline bool device_can_wakeup(struct device *dev)
{
    return dev->power.can_wakeup; /* returns false */
}
```

فمش بيسمحله يكون wakeup source. حتى لو في `wakeup_source` متسجل، الـ:

```c
static inline bool device_may_wakeup(struct device *dev)
{
    return dev->power.can_wakeup && !!dev->power.wakeup;
    /* false && anything = false */
}
```

فالـ IRQ الخاص بالـ UART مش بيتفعّل كـ wake IRQ.

#### الحل
في الـ probe function للـ UART driver:

```c
static int rk3562_uart_probe(struct platform_device *pdev)
{
    /* ... existing init code ... */

    /* Enable wakeup capability and register wakeup source */
    device_init_wakeup(&pdev->dev, true);

    /* Use devm variant so cleanup is automatic */
    /* OR use devm_device_init_wakeup for managed cleanup */
    return devm_device_init_wakeup(&pdev->dev);
}
```

وفي الـ Device Tree لازم يضيف:

```dts
&uart2 {
    status = "okay";
    wakeup-source;
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart2_xfer>;
    pinctrl-1 = <&uart2_sleep>;
};
```

بعد التعديل:

```bash
cat /sys/kernel/debug/wakeup_sources | grep uart
# uart2    1    0    0    0    0    ...    active=0
echo mem > /sys/power/state
# أرسل بيانات من الـ modem
# الـ system صحى!
```

#### الدرس المستفاد
**الـ `device_init_wakeup`** هي البوابة الأولى — لو منستدعيهاش في الـ probe، مهما عملت في الـ DT أو في الـ PM callbacks، الجهاز مش هيصحى. الـ `device_can_wakeup` هي الـ guard الأول في كل قرار wakeup في الـ kernel.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ Box مش بتنام

#### العنوان
الـ wakeup_source مش بيتحرر وبيمنع الـ autosleep على Allwinner H616 TV Box

#### السياق
منتج **Android TV Box** بيشتغل على **Allwinner H616**. المفروض الـ box تدخل autosleep بعد 10 دقايق من غير استخدام. بس المستخدمين بيشكوا إنها سخنانة طول الليل ومش بتنام أبدًا.

#### المشكلة
الـ system مش بيدخل suspend. فريق الـ SW اشتبه في الـ **HDMI** driver أو الـ **USB** stack.

#### التحليل
أول خطوة:

```bash
# شوف مين بيمنع الـ sleep
cat /sys/kernel/debug/wakeup_sources
```

الناتج:

```
Name            Active  Prevent SleepTime  ...
hdmi_cec_ws     1       active  12340000   ...
```

الـ `hdmi_cec_ws` شغال منذ ساعات. راح للـ CEC driver:

```c
static irqreturn_t cec_irq_handler(int irq, void *dev_id)
{
    struct cec_dev *cec = dev_id;

    __pm_stay_awake(cec->ws); /* activates wakeup_source */

    /* process CEC message */
    schedule_work(&cec->work);

    return IRQ_HANDLED;
}

static void cec_work_handler(struct work_struct *work)
{
    struct cec_dev *cec = container_of(work, struct cec_dev, work);

    process_cec_command(cec);
    /* BUG: missing __pm_relax(cec->ws) */
}
```

**الـ `__pm_relax`** منستدعيهاش أبدًا! يعني الـ `wakeup_source->active` فاضل `true` للأبد. الـ PM core لما بيشوف:

```c
/* في wakeup.c — simplified */
static bool ws_active(struct wakeup_source *ws)
{
    return ws->active; /* always true */
}
```

فالـ autosleep بيلاقي wakeup source شغال ومش بينام.

الـ `prevent_sleep_time` بيتراكم في:

```c
ktime_t prevent_sleep_time; /* struct wakeup_source field */
```

ولو شفنا `/sys/kernel/debug/wakeup_sources` هنلاقي هذا الحقل بالـ ms بالآلاف.

#### الحل

```c
static void cec_work_handler(struct work_struct *work)
{
    struct cec_dev *cec = container_of(work, struct cec_dev, work);

    process_cec_command(cec);

    /* Release wakeup source after processing */
    __pm_relax(cec->ws);
}
```

للتحقق:

```bash
watch -n1 'cat /sys/kernel/debug/wakeup_sources | grep cec'
# active_count يزيد، relax_count كمان يزيد
# prevent_sleep_time وقفت عن الزيادة
```

#### الدرس المستفاد
كل `__pm_stay_awake` لازم يقابله `__pm_relax`. الـ `wakeup_source->active` بيتحول `true` في `__pm_stay_awake` ومش بيرجع `false` إلا بـ `__pm_relax`. أي حد بيكتب driver فيه IRQ + workqueue لازم يضمن إن الـ relax بيحصل بعد إنهاء الشغل مش في الـ IRQ handler نفسه.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — الـ device بيصحى كتير أوي

#### العنوان
الـ I2C sensor بيصحى الـ system كل ثانية بدل ما يصحيه بس لما في بيانات مهمة

#### السياق
IoT sensor node بيستخدم **STM32MP1** مع **I2C accelerometer** (LIS2DH12). الـ design: الـ MPU بيدخل sleep، الـ accelerometer بيصحيه لما يحس بـ threshold motion. بس في الـ field، البطارية بتخلص في ساعتين بدل أسبوعين.

#### المشكلة
الـ system بيصحى كل ثانية تقريبًا. الـ accelerometer interrupt line شغال بس مش المفروض يولد كل ده.

#### التحليل
المهندس شاف الـ wakeup stats:

```bash
cat /sys/kernel/debug/wakeup_sources | grep accel
# wakeup_count: 3600   في ساعة واحدة!
```

الـ `wakeup_count` بيمثل عدد مرات إن الـ wakeup source ممكن يكون قاطع suspend. راح للـ driver:

```c
static irqreturn_t accel_irq_handler(int irq, void *data)
{
    struct accel_dev *accel = data;

    /* Wake up for EVERY interrupt — wrong! */
    pm_wakeup_event(accel->dev, 500); /* 500ms timeout */

    schedule_work(&accel->work);
    return IRQ_HANDLED;
}
```

الـ `pm_wakeup_event` بيستدعي:

```c
static inline void pm_wakeup_event(struct device *dev, unsigned int msec)
{
    pm_wakeup_dev_event(dev, msec, false);
}
```

وده بيستدعي `pm_wakeup_ws_event` بـ `hard=false`، وده بيزود الـ `wakeup_count` ويبدأ timer بـ 500ms. المشكلة: الـ accelerometer بيولد data-ready interrupt كل ثانية (الـ ODR = 1Hz) حتى من غير motion. الـ driver بيعامل كل interrupt كـ wakeup event.

#### الحل

```c
static irqreturn_t accel_irq_handler(int irq, void *data)
{
    struct accel_dev *accel = data;

    /* Read interrupt source register first */
    u8 int_src = accel_read_reg(accel, INT1_SRC);

    /* Only wake system for significant motion events */
    if (int_src & LIS2DH12_IA_MASK) {
        /* Real motion detected — hard wakeup, no timeout */
        pm_wakeup_hard_event(accel->dev);
    }
    /* Data-ready interrupts: just schedule work, no wakeup */

    schedule_work(&accel->work);
    return IRQ_HANDLED;
}
```

الـ `pm_wakeup_hard_event` بيستدعي:

```c
static inline void pm_wakeup_hard_event(struct device *dev)
{
    pm_wakeup_dev_event(dev, 0, true); /* hard=true */
}
```

الـ `hard=true` معناه إن حتى لو الـ suspend بدأ فعلًا، الـ kernel يلغيه فورًا.

#### الدرس المستفاد
**الفرق بين `pm_wakeup_event` و`pm_wakeup_hard_event`**: الأول بيديك timeout يمكن الـ suspend يبدأ خلاله لو مفيش events تانية، والتاني بيلغي الـ suspend فورًا. ومهم جدًا تفلتر الـ IRQs قبل ما تقرر تعمل wakeup event — مش كل interrupt يستحق إنه يصحي الـ system.

---

### السيناريو 4: Automotive ECU على i.MX8 — الـ SPI device مش بيتنظف صح

#### العنوان
Memory leak في الـ wakeup_source عند unload الـ driver على i.MX8 Automotive ECU

#### السياق
**Automotive ECU** بيشتغل على **i.MX8** في سيارة كهربائية. الـ ECU بيتحكم في SPI-connected **CAN transceiver** (MCP2518FD). في بعض الأحيان، الـ ECU بيعمل hot-reload للـ driver لما تتغير configuration. بعد عدة reloads، الـ system بيبدأ يتصرف غريب وفي memory warnings.

#### المشكلة
كل مرة الـ driver بيتعمله reload، الـ memory usage بيزيد. بعد 50 reload، الـ system دخل في low-memory condition.

#### التحليل

```bash
# قبل reload
grep -i mcp /sys/kernel/debug/wakeup_sources
# mcp2518fd    active=0 ...

# بعد 10 reloads
ls /sys/kernel/debug/wakeup_sources | wc -l
# كل مرة بيزيد entry
```

راح للـ driver:

```c
static int mcp2518_probe(struct spi_device *spi)
{
    struct mcp2518_dev *priv;

    priv = devm_kzalloc(&spi->dev, sizeof(*priv), GFP_KERNEL);

    /* Register wakeup source */
    priv->ws = wakeup_source_register(&spi->dev, "mcp2518fd");
    if (!priv->ws)
        return -ENOMEM;

    device_init_wakeup(&spi->dev, true);
    /* ... rest of init ... */
    return 0;
}

static int mcp2518_remove(struct spi_device *spi)
{
    struct mcp2518_dev *priv = spi_get_drvdata(spi);

    /* BUG: missing wakeup_source_unregister */
    device_wakeup_disable(&spi->dev);
    return 0;
}
```

الـ `wakeup_source_register` بيعمل `kzalloc` للـ `struct wakeup_source` وبيضيفها للـ global list. لو `wakeup_source_unregister` متستدعيش، الـ struct بيفضل في الـ list وفي الـ memory.

#### الحل

```c
static int mcp2518_remove(struct spi_device *spi)
{
    struct mcp2518_dev *priv = spi_get_drvdata(spi);

    /* Proper cleanup order */
    device_init_wakeup(&spi->dev, false); /* disable + mark not capable */
    wakeup_source_unregister(priv->ws);  /* free the wakeup_source struct */
    priv->ws = NULL;

    return 0;
}
```

أو الأفضل: استخدام الـ `devm` variant في الـ probe وخلي الـ kernel يتكفل بالـ cleanup:

```c
static int mcp2518_probe(struct spi_device *spi)
{
    /* ... */
    /* devm_device_init_wakeup handles enable + auto-disable on remove */
    ret = devm_device_init_wakeup(&spi->dev);
    if (ret)
        return ret;

    /* devm_add_action already registered for device_disable_wakeup */
    /* But still need to manually handle wakeup_source if registered separately */
    return 0;
}
```

للتحقق بعد الإصلاح:

```bash
# Load/unload 10 times
for i in $(seq 10); do
    modprobe mcp251xfd
    sleep 1
    modprobe -r mcp251xfd
    sleep 1
done
cat /sys/kernel/debug/wakeup_sources | grep mcp
# لازم تكون فاضية أو entry واحدة بس
```

#### الدرس المستفاد
**الـ `devm_device_init_wakeup`** هي الأفضل دايمًا لأنها بتضيف `device_disable_wakeup` تلقائيًا كـ devm action عبر `devm_add_action_or_reset`. لو استخدمت `wakeup_source_register` يدويًا، لازم تعمل `wakeup_source_unregister` يدويًا في الـ remove. مفيش devm wrapper للـ `wakeup_source_register` في الـ base API.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ USB wakeup شغال بس بيصحى الـ system غلط

#### العنوان
الـ `wakeup_path` مش متسجل صح وبيسبب suspend failure على AM62x custom board

#### السياق
فريق bring-up بيشتغل على **custom board** بيستخدم **TI AM62x** SoC. الـ board فيها **USB hub** مربوط بـ USB3 port وتحته **USB Ethernet adapter**. المطلوب: لما يجي network packet على الـ USB Ethernet، الـ system يصحى.

#### المشكلة
الـ USB Ethernet بيستقبل packets، الـ wakeup event بيتعمل، بس الـ system مش بيصحى. الـ suspend بيكتمل بنجاح والـ system بيفضل نايم.

#### التحليل

أول حاجة:

```bash
echo enabled > /sys/bus/usb/devices/1-1/power/wakeup
echo enabled > /sys/bus/usb/devices/1-1.1/power/wakeup
cat /sys/bus/usb/devices/1-1/power/wakeup
# enabled
```

الـ device `may_wakeup` شغال. بس المشكلة تانية. راح يشوف الـ PM core logs:

```bash
echo 1 > /sys/power/pm_debug_messages
echo mem > /sys/power/state
dmesg | grep -i "wakeup\|suspend\|path"
```

لقى:

```
PM: Skipping USB hub 1-1: not in wakeup path
```

الـ PM core لما بيعمل suspend، بيمشي في الـ device tree من الـ leaf لـ root. كل device في الـ path لازم يتعلم إنه `wakeup_path`. الـ code في الـ PM core بيعمل:

```c
/* simplified from drivers/base/power/main.c */
static void dpm_propagate_wakeup_to_parent(struct device *dev)
{
    struct device *parent = dev->parent;

    if (!parent)
        return;

    /* Mark parent as being in the wakeup path */
    if (device_wakeup_path(dev)) {
        device_set_wakeup_path(parent); /* sets parent->power.wakeup_path = true */
    }
}
```

والـ function:

```c
static inline void device_set_wakeup_path(struct device *dev)
{
    dev->power.wakeup_path = true;
}
```

المشكلة: الـ USB Ethernet driver عمل `device_wakeup_enable` على نفسه، بس الـ USB hub driver اللي فوقيه ما عملش `device_set_wakeup_capable` فالـ propagation وقفت عنده.

```bash
cat /sys/bus/usb/devices/1-1/power/can_wakeup
# 0  <-- المشكلة هنا! الـ hub نفسه مش capable
```

#### الحل

الـ USB hub driver لازم يعمل نفسه wakeup capable:

```c
/* في usb hub driver probe */
static int hub_probe(struct usb_interface *intf, const struct usb_device_id *id)
{
    struct usb_device *hdev = interface_to_usbdev(intf);

    /* Hub must be wakeup-capable to propagate child wakeups */
    device_set_wakeup_capable(&hdev->dev, true);

    /* Enable wakeup by default for hubs with downstream wakeup-capable devices */
    if (hub_is_superspeed(hdev))
        device_wakeup_enable(&hdev->dev);

    /* ... rest of probe ... */
}
```

أو من الـ DT/userspace:

```bash
# Enable wakeup capability on the hub
echo enabled > /sys/bus/usb/devices/1-1/power/wakeup

# Verify wakeup path propagation
cat /sys/bus/usb/devices/1-1/power/wakeup_path
# 1  — now it's in the path
```

للتحقق الكامل:

```bash
# Check the full wakeup path
for dev in /sys/bus/usb/devices/*/; do
    echo -n "$dev: wakeup="
    cat "$dev/power/wakeup" 2>/dev/null
    echo -n " wakeup_path="
    cat "$dev/power/wakeup_path" 2>/dev/null
    echo
done

# Now test suspend + remote wakeup
echo mem > /sys/power/state &
sleep 2
# Send network packet to USB Ethernet
ping -c1 192.168.1.x
# System should wake up!
```

#### الدرس المستفاد
الـ **wakeup path** في الـ Linux PM مش بيتسجل تلقائيًا — كل device في الـ chain من الـ leaf لـ root لازم يكون `wakeup_capable` وفي الـ `wakeup_path`. الـ `device_wakeup_path()` و`device_set_wakeup_path()` بيتحكموا في ده. لو أي device في الـ chain مش capable، الـ PM core بيتجاهل الـ wakeup signal من الـ devices اللي تحته. ده بيأثر خصوصًا على الـ USB hubs والـ PCI bridges.
## Phase 7: مصادر ومراجع

### مقدمة

الـ `pm_wakeup.h` هو قلب الـ **wakeup source framework** في الـ Linux kernel. المراجع دي متنوعة — من الـ mailing list اللي اتكتب فيها الكود الأصلي، لحد الكتب اللي بتشرح المبادئ من الأساس.

---

### LWN.net Articles

دي أهم المقالات اللي بتغطي تطور الـ wakeup source infrastructure:

| العنوان | الأهمية |
|---|---|
| [PM / Wakeup: Introduce wakeup source objects and event statistics (v2)](https://lwn.net/Articles/405108/) | الـ patch الأساسي اللي عرّف `struct wakeup_source` |
| [PM / Wakeup: Introduce wakeup source objects (original thread)](https://lwn.net/Articles/404000/) | النقاش الأولاني في الـ mailing list قبل الـ v2 |
| [PM: Avoid losing wakeup events during suspend](https://lwn.net/Articles/392897/) | المشكلة اللي أدت لإنشاء الـ framework — فقدان wakeup events أثناء الـ suspend |
| [An alternative to suspend blockers](https://lwn.net/Articles/416690/) | كيف الـ PM core بيتعامل مع الـ wakeup events عبر `pm_stay_awake()` و `pm_wakeup_event()` |
| [Autosleep and wake locks](https://lwn.net/Articles/479841/) | دمج الـ wakeup sources في kernel 2.6.37 ومقارنتها بالـ Android wake locks |
| [Waking systems from suspend](https://lwn.net/Articles/429925/) | آليات الـ wakeup من suspend بشكل عام |

---

### الـ Official Kernel Documentation

الـ documentation الرسمي موجود في مسارات اتفرقت بين الـ `Documentation/power/` والـ `Documentation/driver-api/pm/`:

```
Documentation/power/runtime_pm.rst          # Runtime PM و wakeup integration
Documentation/power/suspend-and-interrupts.rst  # العلاقة بين IRQs والـ wakeup
Documentation/driver-api/pm/devices.rst     # Device PM API الشامل
Documentation/driver-api/pm/types.rst       # تعريفات الـ types المستخدمة
Documentation/devicetree/bindings/power/wakeup-source.txt  # DT bindings للـ wakeup sources
```

الـ **sysfs interface** للـ wakeup sources موجود على:

```
/sys/class/wakeup/                   # كل wakeup source مسجّل
/sys/devices/.../power/wakeup        # إعداد الـ wakeup per device
/sys/devices/.../power/wakeup_count  # عداد الـ wakeup events
```

---

### الـ Implementation Source Files

الملفات الأساسية في الكورنيل للـ wakeup source framework:

```
include/linux/pm_wakeup.h            # الـ API declarations (الملف ده)
drivers/base/power/wakeup.c          # الـ core implementation
drivers/base/power/wakeup_stats.c    # الـ sysfs statistics
drivers/base/power/wakeirq.c         # الـ dedicated wakeup IRQ support
```

---

### Relevant Kernel Commits

**الـ commit الأساسي** اللي عرّف `struct wakeup_source`:

- **Rafael J. Wysocki** هو المؤلف الرئيسي — ابدأ بالـ `git log --all -- drivers/base/power/wakeup.c` في أي kernel tree
- الـ commit الأصلي كان في سياق kernel **2.6.37** (نهاية 2010)
- الـ `wakeup_source_register()` اتضافت لاحقًا كـ helper تجمع الـ create + add في خطوة واحدة

الـ commits المهمة على الـ [kernel.org git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/drivers/base/power/wakeup.c):

```bash
# لتتبع تاريخ الملف في kernel tree محلي
git log --oneline -- drivers/base/power/wakeup.c
git log --oneline -- include/linux/pm_wakeup.h
```

---

### Mailing List Discussions

النقاشات الحقيقية اللي شكّلت الـ framework:

- **[linux-pm] Wakeup-events implementation** (August 2010):
  [https://lists.linuxfoundation.org/pipermail/linux-pm/2010-August/028290.html](https://lists.linuxfoundation.org/pipermail/linux-pm/2010-August/028290.html)

- **[PATCH] PM / wakeup: Integrate mechanism to abort transitions** — Rafael Wysocki:
  [https://lore.kernel.org/all/6281089.pVH57fyP0i@aspire.rjw.lan/](https://lore.kernel.org/all/6281089.pVH57fyP0i@aspire.rjw.lan/)

- **Re: Understanding the use cases for calling `pm_wakeup_event()`**:
  [https://www.spinics.net/lists/kernel/msg2623540.html](https://www.spinics.net/lists/kernel/msg2623540.html)

- **Re: [PATCH v6] PM / wakeup: show wakeup sources stats in sysfs**:
  [https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2067358.html](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2067358.html)

---

### Kernelnewbies.org

الـ kernel version pages بتوضح متى اتضافت features في الـ wakeup framework:

- **[Linux_4.5](https://kernelnewbies.org/Linux_4.5)** — تغييرات الـ PM في kernel 4.5
- **[Linux_4.9](https://kernelnewbies.org/Linux_4.9)** — تحسينات الـ wakeup source statistics
- **[KernelGlossary](https://kernelnewbies.org/KernelGlossary)** — تعريفات مصطلحات الـ power management
- **[LinuxChanges](https://kernelnewbies.org/LinuxChanges)** — سجل التغييرات عبر الإصدارات

---

### eLinux.org — Embedded Perspective

الـ wakeup sources مهمة جدًا في الـ embedded systems:

- **[Power Management Framework](https://elinux.org/Power_Management_Framework)** — نظرة شاملة على الـ Linux PM framework
- **[Power Management Definition Of Terms](https://elinux.org/Power_Management_Definition_Of_Terms)** — تعريف المصطلحات بما فيها الـ wakeup sources
- **[OMAP Power Management](https://elinux.org/OMAP_Power_Management)** — مثال عملي على تكوين الـ wakeup sources في الـ OMAP SoCs
- **[Static Power Management Specification](https://elinux.org/Static_Power_Management_Specification)** — المتطلبات العامة للـ suspend/resume و wakeup
- **[Power Management Presentations](https://elinux.org/Power_Management_Presentations)** — presentations من مؤتمرات ELC

---

### الكتب الموصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم**: Chapter 14 — The Linux Device Model
- **ملاحظة**: LDD3 قديم نسبيًا (2005) والـ wakeup source framework جه بعده، بس الـ device model concepts لازالت صحيحة
- **متاح مجانًا**: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition (Robert Love)
- **الفصل المهم**: Chapter 14 — The Block I/O Layer (للفهم العام للـ kernel subsystems)
- **ملاحظة**: يشرح architecture الـ kernel بشكل عام اللي بيساعد على فهم الـ PM framework
- الـ spinlocks والـ timers اللي موجودين في `struct wakeup_source` متشروحين بالتفصيل

#### Embedded Linux Primer, 2nd Edition (Christopher Hallinan)
- **الفصل المهم**: Chapter 15 — Open Source U-Boot, وChapter على الـ power management
- **الأهمية**: بيتكلم عن الـ wakeup sources في سياق الـ embedded boards (مثل beaglebone)
- عملي جدًا لفهم ليه `device_init_wakeup()` بتتكال في الـ board drivers

---

### Search Terms للبحث عن مزيد من المعلومات

```
# البحث في الـ kernel source
git log --all --grep="wakeup_source" -- drivers/base/power/
git log --all --grep="pm_stay_awake"

# البحث في الـ mailing lists
site:lore.kernel.org "wakeup_source_register"
site:lore.kernel.org "pm_wakeup" "struct wakeup_source"

# LWN.net
site:lwn.net "wakeup source" suspend
site:lwn.net "pm_stay_awake" OR "pm_relax"

# الـ kernel documentation online
site:kernel.org/doc wakeup source
site:docs.kernel.org power management wakeup
```

**الـ keywords الأساسية**:
- `wakeup_source` — الـ struct الرئيسي
- `pm_stay_awake` / `pm_relax` — الـ API الأساسي
- `suspend blockers` — الاسم القديم (Android era)
- `autosleep` — الـ feature المرتبطة
- `wakeup events framework` — الاسم الرسمي للـ subsystem
- `CONFIG_PM_SLEEP` — الـ Kconfig option اللي بيتحكم في الكود
- `device_may_wakeup` — الـ function الأكتر استخدامًا في الـ drivers
## Phase 8: Writing simple module

### الفكرة

**`wakeup_source_register`** هي دالة exported بتسجل wakeup source جديد في النظام — أي device بيقول "أنا ممكن أصحّي الجهاز من النوم". هنعمل kprobe عليها عشان نعرف مين بيسجل wakeup sources ومتى.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on wakeup_source_register():
 * Logs every wakeup source registration with its name and device.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* kprobe struct, register/unregister */
#include <linux/device.h>       /* struct device, dev_name() */
#include <linux/pm_wakeup.h>    /* wakeup_source_register prototype */
#include <linux/printk.h>       /* pr_info */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs");
MODULE_DESCRIPTION("kprobe on wakeup_source_register to trace wakeup source creation");

/*
 * pre_handler is called just BEFORE wakeup_source_register() executes.
 * Arguments on x86-64:
 *   regs->di = first arg  -> struct device *dev
 *   regs->si = second arg -> const char *name
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct device *dev = (struct device *)regs->di;
    const char    *ws_name = (const char *)regs->si;

    /* Print wakeup source name and the device it belongs to (if any) */
    pr_info("wakeup_source_register: name=\"%s\" dev=%s\n",
            ws_name ? ws_name : "(null)",
            (dev && dev_name(dev)) ? dev_name(dev) : "none");

    return 0; /* 0 = continue normal execution */
}

/* kprobe descriptor — we only need a pre_handler here */
static struct kprobe kp = {
    .symbol_name = "wakeup_source_register",
    .pre_handler = handler_pre,
};

static int __init ws_probe_init(void)
{
    int ret;

    /* Register the kprobe; kernel will resolve the symbol address */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe planted on wakeup_source_register at %p\n", kp.addr);
    return 0;
}

static void __exit ws_probe_exit(void)
{
    /* Must unregister before module memory is freed */
    unregister_kprobe(&kp);
    pr_info("kprobe on wakeup_source_register removed\n");
}

module_init(ws_probe_init);
module_exit(ws_probe_exit);
```

---

### Makefile بسيط

```makefile
obj-m += ws_kprobe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# تحميل الموديول
sudo insmod ws_kprobe.ko

# مشاهدة اللوق
sudo dmesg | grep wakeup_source_register

# فك التحميل
sudo rmmod ws_kprobe
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `<linux/module.h>` | ماكروهات `module_init`, `module_exit`, `MODULE_LICENSE` |
| `<linux/kprobes.h>` | الـ `struct kprobe` وفانكشنات `register/unregister_kprobe` |
| `<linux/device.h>` | الـ `struct device` و`dev_name()` عشان نطبع اسم الـ device |
| `<linux/pm_wakeup.h>` | prototype الدالة اللي بنـ hook عليها |
| `<linux/printk.h>` | `pr_info()` و`pr_err()` |

---

#### الـ `handler_pre`

**الـ `pre_handler`** بيتشال قبل ما `wakeup_source_register` تنفذ، فبنقدر نقرأ الـ arguments من registers قبل ما تتغير. على x86-64 الـ calling convention بيحط أول argument في `rdi` وتاني argument في `rsi`، فبنعمل cast مباشر من `regs->di` و`regs->si`.

الـ null checks (`ws_name ? ws_name : "(null)"`) ضرورية عشان المحتمل إن driver بيستدعي الدالة بـ name=NULL وده مش undefined behavior من ناحيتنا.

---

#### الـ `struct kprobe`

**`symbol_name`** بتخلي الـ kernel يحل العنوان وقت الـ `register_kprobe` بدل ما نحتاج نعرف العنوان الفيزيائي يدوياً. لو الدالة مش exported أو مش موجودة `register_kprobe` بترجع `-ENOENT`.

---

#### الـ `module_exit`

**`unregister_kprobe`** لازمة في الـ exit عشان الـ kprobe handler بيشاور على كود في memory الموديول — لو الموديول اتحط من الـ kernel والـ kprobe لسه مسجلة، أي call بعد كده هيعمل kernel panic. الـ unregister بتضمن إن الـ kernel مش هيستدعي الـ handler بعد ما الـ memory اتحررت.

---

### مثال على الـ output

```
[  123.456789] kprobe planted on wakeup_source_register at ffffffffc0123456
[  124.001234] wakeup_source_register: name="event5" dev=input5
[  124.002100] wakeup_source_register: name="bluetooth" dev=hci0
[  124.900000] wakeup_source_register: name="alarmtimer" dev=none
```

كل سطر بيظهر اسم الـ wakeup source واسم الـ device اللي طلبه، وده بيساعد في فهم مين بيمنع الـ system من الـ sleep.
