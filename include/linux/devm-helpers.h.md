## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي الفايل ده جزء منه

الفايل ده جزء من **Device Resource Management Helpers** subsystem — وده موثق في `MAINTAINERS` تحت اسم "DEVICE RESOURCE MANAGEMENT HELPERS"، ومينتينيره هم Hans de Goede و Matti Vaittinen.

---

### المشكلة اللي بيحلها الفايل ده

تخيل إنك عندك مطعم (الـ **driver**)، وعندك موظفين بيشتغلوا في الكواليس (الـ **work items** و **IRQs**). لما المطعم بيقفل (الـ driver بيتشال `remove()`)، لازم كل الموظفين يوقفوا شغلهم وميعملوش أي حاجة جديدة قبل ما تقفل الباب بالمفتاح.

المشكلة: في Linux، عندنا طريقتين لتنظيف الـ resources:

**١. الطريقة اليدوية (Manual):** أنت بنفسك تكتب كود في `remove()` يلغي كل حاجة.

**٢. الطريقة التلقائية (devm = Device-Managed):** الـ kernel يعمل cleanup تلقائي لما الـ device يتشال.

لو خلطت الاتنين مع بعض — يحصل كارثة:

```
مثال حقيقي للمشكلة:
IRQ (devm-managed) → بيشتغل بعد remove()
    ↓
بيشيدوا work item في الـ WQ (manually-managed)
    ↓
الـ WQ اتكلينت في remove() بالفعل!
    ↓
USE-AFTER-FREE أو KERNEL PANIC
```

يعني: الـ IRQ اشتغل بعد ما الـ remove() خلص، وبعث شغل لـ workqueue اتمسحت بالفعل. ده race condition نادر، بيحصل بالصدفة، وصعب جداً تعيد إنتاجه.

---

### الحل اللي بيقدمه `devm-helpers.h`

الفايل ده بيقدم **wrapper functions** تجمع بين الـ `INIT_WORK` / `INIT_DELAYED_WORK` التقليدية وبين نظام الـ `devm` — علشان الـ work items تتلغى تلقائياً لما الـ device يتشال، بالظبط زي ما الـ IRQs بتتلغى.

النتيجة: كل حاجة بقت devm-managed، مفيش خلط، مفيش race conditions.

---

### الـ Workflow بالتفصيل

```
Driver Probe (probe()):
    ↓
devm_work_autocancel(dev, &w, my_worker_fn)
    ├─ INIT_WORK(&w, my_worker_fn)       ← تجهيز الـ work item
    └─ devm_add_action(dev, cancel_work_sync, &w)  ← تسجيل cleanup

    [الـ driver بيشتغل عادي]

Driver Remove (remove()):
    ↓
devm cleanup يحصل تلقائياً:
    └─ cancel_work_sync(&w)  ← بيستنى لحد ما الـ worker يخلص وبيمنع أي شغل جديد
```

---

### الـ Functions الموجودة في الفايل

| Function | الوظيفة |
|---|---|
| `devm_work_autocancel()` | بيعمل `INIT_WORK` وبيضمن إن الـ work يتلغى تلقائياً عند الـ detach |
| `devm_delayed_work_autocancel()` | نفس الفكرة لكن للـ `delayed_work` (اللي بيشتغل بعد delay) |
| `devm_work_drop()` | الـ cleanup callback الداخلية — بتعمل `cancel_work_sync()` |
| `devm_delayed_work_drop()` | الـ cleanup callback للـ delayed work — بتعمل `cancel_delayed_work_sync()` |

---

### قصة واقعية: Battery Charger Driver

تخيل driver لـ charger chip (زي `max17042_battery.c` اللي بيستخدم الفايل ده):

1. الـ chip بتبعت **IRQ** لما البطارية تتغير.
2. الـ IRQ handler بيبعث **work item** يقرأ بيانات البطارية ويبلغ النظام.
3. لما الـ device يتشال (مثلاً kernel module unload أو device unplug):
   - لو الـ IRQ devm-managed والـ WQ manual → race condition ممكن يحصل.
   - لو استخدمنا `devm_work_autocancel()` → كل حاجة بتتلغى بنفس الترتيب الصح.

---

### ملاحظة مهمة عن التوقيت

الـ comment في أول الفايل بيحذر صراحةً:

> "devm based cancellation may be performed some time **after** the `remove()` is ran."

يعني الـ devm cleanup مش بيحصل في نفس لحظة `remove()` — بيحصل بعدها بشوية. وده بالظبط سبب المشكلة لو خلطت devm بـ manual.

---

### الفايلات ذات الصلة

**الـ Core:**
- `include/linux/devm-helpers.h` — الفايل نفسه (الـ helpers)
- `include/linux/device/devres.h` — الـ devm infrastructure الأساسية (`devm_add_action`, `devres_alloc`, إلخ)
- `drivers/base/devres.c` — الـ implementation الفعلي لنظام الـ devm

**الـ Headers المعتمدة:**
- `include/linux/workqueue.h` — تعريفات `work_struct`, `delayed_work`, `INIT_WORK`, `cancel_work_sync`
- `include/linux/workqueue_types.h` — الـ types الأساسية للـ workqueue

**أمثلة من الـ Drivers اللي بتستخدمه:**
- `drivers/power/supply/max17042_battery.c` — battery fuel gauge
- `drivers/power/supply/sbs-battery.c` — smart battery
- `drivers/iio/adc/xilinx-ams.c` — ADC driver
- `drivers/extcon/extcon-gpio.c` — GPIO-based connector detection
- `drivers/hwmon/raspberrypi-hwmon.c` — RPi hardware monitor
- `drivers/hid/intel-ish-hid/ipc/ipc.c` — Intel ISH HID

---

### الملفات اللي تتكون منها الـ Subsystem دي

```
DEVICE RESOURCE MANAGEMENT (devm) Subsystem:
├── include/linux/devm-helpers.h        ← الفايل بتاعنا (work/delayed_work helpers)
├── include/linux/device/devres.h       ← core devm API (devm_add_action, devm_kmalloc...)
├── drivers/base/devres.c               ← implementation كامل لنظام الـ devres
├── include/linux/device.h              ← يـ include الـ devres.h ضمنياً
└── include/linux/workqueue.h           ← تعريفات الـ work items المستخدمة هنا
```
## Phase 2: شرح الـ devm-helpers (Managed Work Queues) Framework

---

### المشكلة اللي الـ subsystem بيحلها

في أي driver بيشتغل على embedded Linux، في دايمًا مرحلتين أساسيتين:
- **probe()**: لما الـ driver بيتربط بالـ device ويحجز الـ resources.
- **remove()**: لما الـ device بيتفصل أو الـ driver بيتحمّل، والـ resources لازم تترجع.

المشكلة الكلاسيكية إن الـ developer لازم يعمل cleanup يدوي لكل حاجة حجزها — IRQ، memory، timers، work queues. لو نسي حاجة أو عمل الترتيب غلط، النتيجة:

- **use-after-free**: الـ work item بيشتغل بعد ما الـ device اتشال من الذاكرة.
- **resource leak**: resource مش اترجعت خالص.
- **race condition**: الـ IRQ اشتغل بعد الـ remove() وscheduled work على queue اتمسحت.

الـ header ده بيحل بالذات المشكلة التالية:

> الـ IRQ كتير بيتعمله `devm_request_irq()` — يعني بيتحرر أوتوماتيك بعد الـ remove(). لكن الـ Work Queue بيتعمل cleanup يدوي *داخل* الـ remove(). المشكلة إن في فترة زمنية بعد الـ remove() وقبل ما devm يحرر الـ IRQ، الـ IRQ ممكن يشتغل ويعمل `queue_work()` على work اتمسح فعلًا.

```
remove() runs                    devm IRQ freed
     |                                |
     v                                v
[manual WQ cancelled] ---gap---  [IRQ fires HERE => use-after-free!]
```

---

### الحل اللي الـ kernel بيستخدمه

الـ framework ده جزء من نظام أكبر اسمه **devres** (Device Resource Management). الفكرة:

> ربط كل resource بـ `struct device`، وعمل cleanup list تتنفذ أوتوماتيك لما الـ device بيتفصل — بالترتيب العكسي لترتيب التسجيل.

الـ `devm-helpers.h` بيضيف فوق ده:
- `devm_work_autocancel()` — تسجّل `work_struct` بحيث يتعمله `cancel_work_sync()` أوتوماتيك.
- `devm_delayed_work_autocancel()` — نفس الفكرة لكن مع `delayed_work`.

بكده الـ work queue بيتعامل زي أي resource تاني مربوط بعمر الـ device.

---

### الـ real-world analogy — مطعم بشيف واحد

تخيل إنك صاحب مطعم وعندك:

| عنصر في المطعم | المقابل في الكود |
|---|---|
| الـ مطعم نفسه | `struct device` |
| الـ شيف | الـ `work_func_t` (الـ worker function) |
| ورقة الأوردرات (queue) | `workqueue_struct` |
| أوردر معلّق | `work_struct` |
| أوردر بموعد | `delayed_work` |
| عقد تأمين على الشيف | `devm_add_action()` |

لما المطعم بيقفل (device detached):
- بدون devm: لازم تتصل بالشيف وتقوله "وقف كل حاجة" يدوي — ولو في أوردر لسه بيتحضر وانت مسحت القائمة، هيحضّره ويحطه فين؟
- مع devm: العقد بيشتغل أوتوماتيك — `cancel_work_sync()` بتستنّى لحد ما الأوردر الحالي يخلص، وبعدين تقفل الكل بأمان.

**التفاصيل الدقيقة في الـ analogy:**

| جزئية في المطعم | المعنى الفعلي |
|---|---|
| العقد "بيشتغل بعد القفل" | الـ devres list بتتنفذ بعد الـ remove() |
| "استنّى الأوردر الحالي يخلص" | `cancel_work_sync()` بيـblock لحد ما الـ work ينهي |
| "لا تقبل أوردرات جديدة" | النتيجة النهائية بعد الـ cancel |
| ترتيب إغلاق المطبخ | الـ devres بتتحرر LIFO (آخر حاجة اتسجلت، أول حاجة بتتحرر) |

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Driver Code                              │
│                                                                 │
│  probe() {                                                      │
│    devm_work_autocancel(dev, &w, my_worker);  ◄── registers     │
│    devm_delayed_work_autocancel(dev, &dw, my_timer_worker);     │
│  }                                                              │
└────────────────────────────┬────────────────────────────────────┘
                             │ calls
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   devm-helpers.h                                │
│                                                                 │
│  devm_work_autocancel()                                         │
│    ├─ INIT_WORK(w, worker)        ◄── workqueue init            │
│    └─ devm_add_action(dev,                                      │
│           devm_work_drop, w)      ◄── registers cleanup         │
│                                                                 │
│  devm_delayed_work_autocancel()                                 │
│    ├─ INIT_DELAYED_WORK(w, worker)                              │
│    └─ devm_add_action(dev,                                      │
│           devm_delayed_work_drop, w)                            │
└────────────────────────────┬────────────────────────────────────┘
                             │ delegates to
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                  devres subsystem (device.h)                    │
│                                                                 │
│  struct device {                                                │
│    ...                                                          │
│    struct list_head  devres_head;  ◄── linked list of resources │
│    ...                                                          │
│  }                                                              │
│                                                                 │
│  devm_add_action() → allocates devres node → appends to list   │
└────────────────────────────┬────────────────────────────────────┘
                             │ at device detach
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│               devres release (LIFO order)                       │
│                                                                 │
│  devm_work_drop(res)          → cancel_work_sync(res)           │
│  devm_delayed_work_drop(res)  → cancel_delayed_work_sync(res)   │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction

الـ abstraction المركزية هنا هي:

> **ربط عمر الـ operation (work/delayed_work) بعمر الـ device** عن طريق تسجيل cleanup callback في الـ devres list.

مش بس تحجز resource وتحررها — ده نظام بيضمن إن الـ cancellation بتحصل **بعد** أي تنفيذ حالي لأن `cancel_work_sync()` و`cancel_delayed_work_sync()` blocking calls — يعني بينتظروا لحد ما الـ work function تخلص لو هي شغّالة في اللحظة دي.

---

### الـ Subsystems اللي محتاج تفهمها الأول

#### 1. devres (Device Resource Management)
الـ framework اللي بيدير الـ resource list الخاصة بكل `struct device`. كل `devm_*` function بتتكل عليه. بيحتفظ بـ linked list من الـ cleanup callbacks وبينفذها بالترتيب العكسي لما الـ device بيتفصل.

#### 2. workqueue subsystem
الـ kernel mechanism اللي بيسمح بتشغيل functions في context منفصل (kernel threads). في نوعين أساسيين هنا:

- **`struct work_struct`**: شغلة بتتنفذ "دلوقتي" لما الـ kernel يقدر.
- **`struct delayed_work`**: زي الـ `work_struct` بس بيتأجل بـ timer قبل الـ execution.

---

### هيكل الـ Structs

```
struct work_struct
┌──────────────────────────────────┐
│ atomic_long_t  data              │  ← flags + pool pointer packed together
│ struct list_head entry           │  ← position in the queue
│ work_func_t func                 │  ← the actual callback
│ [lockdep_map if DEBUG]           │
└──────────────────────────────────┘

struct delayed_work
┌──────────────────────────────────┐
│ struct work_struct work          │  ← embeds work_struct
│ struct timer_list timer          │  ← fires after delay, queues the work
│ struct workqueue_struct *wq      │  ← which queue to use
│ int cpu                          │  ← preferred CPU
└──────────────────────────────────┘

devres node (internal, kernel/base/devres.c)
┌──────────────────────────────────┐
│ struct list_head entry           │  ← links into device->devres_head
│ dr_release_t release             │  ← = devm_work_drop or devm_delayed_work_drop
│ void *data[]                     │  ← points to the work_struct
└──────────────────────────────────┘
```

**العلاقة بين الكل:**

```
struct device
    │
    └──► devres_head (list)
              │
              ├──► devres node [release=devm_work_drop, data=&w]
              │
              └──► devres node [release=devm_delayed_work_drop, data=&dw]

عند الـ device detach:
    devres_release_all() traverses the list LIFO
        → calls devm_work_drop(&w)         → cancel_work_sync(&w)
        → calls devm_delayed_work_drop(&dw) → cancel_delayed_work_sync(&dw)
```

---

### إيه اللي الـ subsystem بيمتلكه vs بيفوّضه للـ driver

| المسؤولية | مين يتحكم فيها |
|---|---|
| تسجيل الـ cleanup callback | **devm-helpers** — عبر `devm_add_action()` |
| تنفيذ الـ cleanup عند الـ detach | **devres subsystem** — تلقائي |
| الـ `cancel_work_sync()` الفعلي | **workqueue subsystem** |
| تحديد الـ worker function | **الـ driver** |
| تحديد متى يعمل `queue_work()` | **الـ driver** — الـ helpers مش بتعمل queue، بس بتضمن الـ cancel |
| الـ LIFO ordering للـ cleanup | **devres** — بيضمن إن الـ resources المرتبطة ببعض تتحرر بالترتيب الصح |

---

### مثال عملي: sensor driver بيقرأ بشكل دوري

```c
struct my_sensor {
    struct delayed_work poll_work;
    void __iomem *base;
};

/* Worker function — reads sensor and reschedules itself */
static void sensor_poll(struct work_struct *w)
{
    struct my_sensor *s = container_of(to_delayed_work(w),
                                       struct my_sensor, poll_work);
    u32 val = readl(s->base + SENSOR_REG);
    pr_info("sensor: %u\n", val);

    /* reschedule after 1 second */
    schedule_delayed_work(&s->poll_work, HZ);
}

static int my_sensor_probe(struct platform_device *pdev)
{
    struct my_sensor *s;
    int ret;

    s = devm_kzalloc(&pdev->dev, sizeof(*s), GFP_KERNEL);
    if (!s)
        return -ENOMEM;

    s->base = devm_platform_ioremap_resource(pdev, 0);
    if (IS_ERR(s->base))
        return PTR_ERR(s->base);

    /*
     * Register the delayed work with devres.
     * No need for manual cancel in remove() —
     * devres will call cancel_delayed_work_sync() automatically.
     */
    ret = devm_delayed_work_autocancel(&pdev->dev, &s->poll_work, sensor_poll);
    if (ret)
        return ret;

    /* Start polling */
    schedule_delayed_work(&s->poll_work, HZ);

    return 0;
}

/* No remove() needed — devres handles everything */
```

**ملاحظة مهمة**: الـ `devm_delayed_work_autocancel()` بس بتعمل `INIT_DELAYED_WORK` وبتسجل الـ cancel. الـ `schedule_delayed_work()` الـ driver بيعملها لما يريد، الـ cleanup بتحصل أوتوماتيك.

---

### تحذير الـ LIFO والـ ordering

لأن الـ devres بتتحرر LIFO، الترتيب في الـ probe() مهم جدًا:

```c
/* probe() */
devm_request_irq(...);          /* registered first  → freed LAST  */
devm_delayed_work_autocancel(); /* registered second → freed FIRST */

/* عند الـ detach — LIFO: */
/* 1. cancel_delayed_work_sync()  — الـ work بيتوقف */
/* 2. free_irq()                  — الـ IRQ بيتحرر */
```

ده بالظبط الترتيب الصح — لأنك عايز توقف الـ work الأول قبل ما تحرر الـ IRQ اللي كان بيشغّله.

لو عملت العكس (سجلت الـ work بعد الـ IRQ)، هيتحرر الـ IRQ الأول وبعدين تحاول تـcancel الـ work — وفي اللحظة دي IRQ جديد ممكن يكون شغّل work قبل ما الـ cancel يحصل.

---

### ملخص

الـ `devm-helpers.h` هو glue layer صغير لكنه بيحل مشكلة real race condition في driver lifecycle. بدل ما تكتب cleanup code يدوي في كل driver، بتسجّل الـ work items كـ managed resources وبيتحرروا أوتوماتيك بالترتيب الصح مع باقي الـ device resources.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Flags, Enums, وـ Config Options — Cheatsheet

#### `enum work_bits` — بتات الـ `data` field في `work_struct`

| Bit | الاسم | المعنى |
|-----|-------|---------|
| 0 | `WORK_STRUCT_PENDING_BIT` | الـ work item في انتظار التنفيذ |
| 1 | `WORK_STRUCT_INACTIVE_BIT` | الـ work item غير نشط حالياً |
| 2 | `WORK_STRUCT_PWQ_BIT` | الـ `data` بيشاور على `pwq` (per-workqueue pool) |
| 3 | `WORK_STRUCT_LINKED_BIT` | الـ work item الجاي مربوط بيه |
| 4 | `WORK_STRUCT_STATIC_BIT` | (بس لو `CONFIG_DEBUG_OBJECTS_WORK`) static initializer |

#### `enum work_flags` — القيم الفعلية (masks)

| Flag | القيمة | الاستخدام |
|------|--------|-----------|
| `WORK_STRUCT_PENDING` | `1 << 0` | بيتضبط لما الـ work item يتحط في الـ queue |
| `WORK_STRUCT_INACTIVE` | `1 << 1` | الـ work متأخر أو متوقف |
| `WORK_STRUCT_PWQ` | `1 << 2` | الـ data pointer بيشاور على pool worker queue |
| `WORK_STRUCT_LINKED` | `1 << 3` | يربط work items مع بعض |
| `WORK_STRUCT_STATIC` | `0` أو `1 << 4` | debug فقط |

#### `enum wq_misc_consts` — ثوابت متنوعة

| Constant | القيمة | المعنى |
|----------|--------|---------|
| `WORK_NR_COLORS` | `16` | عدد ألوان الـ flush |
| `WORK_CPU_UNBOUND` | `NR_CPUS` | مش مقيد بـ CPU معين |
| `WORK_BUSY_PENDING` | `1 << 0` | الـ work في الـ queue |
| `WORK_BUSY_RUNNING` | `1 << 1` | الـ work بيتنفذ دلوقتي |
| `WORKER_DESC_LEN` | `32` | أقصى طول للـ worker description |

#### `enum wq_affn_scope` — نطاق الـ CPU affinity للـ unbound workqueues

| Value | المعنى |
|-------|---------|
| `WQ_AFFN_DFL` | الـ default بتاع النظام |
| `WQ_AFFN_CPU` | pool واحد لكل CPU |
| `WQ_AFFN_SMT` | pool واحد لكل SMT thread group |
| `WQ_AFFN_CACHE` | pool واحد لكل LLC (Last Level Cache) |
| `WQ_AFFN_NUMA` | pool واحد لكل NUMA node |
| `WQ_AFFN_SYSTEM` | pool واحد على مستوى النظام كله |

#### Config Options المهمة

| Config | الأثر |
|--------|-------|
| `CONFIG_DEBUG_OBJECTS_WORK` | بيضيف `WORK_STRUCT_STATIC_BIT` وتحقق runtime من الـ work objects |
| `CONFIG_LOCKDEP` | بيضيف `lockdep_map` جوه `work_struct` لتتبع locking order |

---

### 1. الـ Structs المهمة

#### `struct work_struct`

**الغرض:** الوحدة الأساسية للعمل المؤجل (deferred work) في الـ kernel. بتعرّف شغلة هتتنفذ في context منفصل بعدين.

| Field | النوع | الشرح |
|-------|-------|-------|
| `data` | `atomic_long_t` | بيخزن الـ flags + pointer للـ pool أو الـ pwq — كل حاجة في كلمة واحدة |
| `entry` | `struct list_head` | بيربط الـ work item في قائمة الـ queue الخاصة بالـ worker pool |
| `func` | `work_func_t` | الـ callback اللي هيتنفذ — signature: `void fn(struct work_struct *)` |
| `lockdep_map` | `struct lockdep_map` | (بس لو `CONFIG_LOCKDEP`) بيساعد lockdep يتتبع الـ locking |

**الارتباطات:** بيُدمج جوه `struct delayed_work` و`struct rcu_work` و`struct execute_work`.

---

#### `struct delayed_work`

**الغرض:** work item بيتأخر تنفيذه بـ timer — يعني بتقوله "نفّذ الشغلة دي بعد X milliseconds".

| Field | النوع | الشرح |
|-------|-------|-------|
| `work` | `struct work_struct` | الـ work item الأساسي — أول field عشان `container_of` |
| `timer` | `struct timer_list` | الـ timer اللي لما يخلص بيحط الـ `work` في الـ queue |
| `wq` | `struct workqueue_struct *` | الـ workqueue اللي الـ timer هيحط الـ work فيه |
| `cpu` | `int` | الـ CPU اللي الـ timer شغال عليه وهيحط الـ work منه |

**الارتباطات:**
- بيحتوي على `work_struct` (embedding)
- بيشاور على `workqueue_struct` بالـ pointer
- الـ timer بيستخدم `delayed_work_timer_fn` كـ callback

---

#### `struct rcu_work`

**الغرض:** work item بيتنفذ بعد RCU grace period — يعني بعد ما كل الـ readers الحاليين يخلصوا.

| Field | النوع | الشرح |
|-------|-------|-------|
| `work` | `struct work_struct` | الـ work item الفعلي |
| `rcu` | `struct rcu_head` | الـ RCU callback — لما الـ grace period يخلص بيحط الـ work |
| `wq` | `struct workqueue_struct *` | الـ workqueue المستهدف |

---

#### `struct workqueue_attrs`

**الغرض:** بيحدد خصائص الـ unbound workqueue — الـ CPU affinity والـ priority.

| Field | النوع | الشرح |
|-------|-------|-------|
| `nice` | `int` | الـ nice level للـ worker threads |
| `cpumask` | `cpumask_var_t` | الـ CPUs المسموح للـ workers تشتغل عليها |
| `__pod_cpumask` | `cpumask_var_t` | internal: subset من `cpumask` للـ per-pod pools |
| `affn_strict` | `bool` | لو `true` الـ workers مش هيتنقلوا برا الـ `__pod_cpumask` |
| `affn_scope` | `enum wq_affn_scope` | بيحدد نطاق الـ CPU pods |
| `ordered` | `bool` | لو `true` الـ work items بتتنفذ واحد واحد بالترتيب |

---

#### `struct device` (مستخدم في devm-helpers)

**الغرض:** بيمثل جهاز في الـ driver model. الـ `devm_` functions بتربط resources بالـ lifetime بتاعته.

الـ `devm-helpers.h` بيهتم بـ:
- `dev->devres_head` — قائمة الـ devm resources المربوطة بالـ device
- لما الـ device يتفصل، الـ devres framework بيمشي على القائمة دي ويستدعي كل الـ release callbacks

---

### 2. علاقات الـ Structs — ASCII Diagrams

#### Embedding Chain

```
struct delayed_work
┌─────────────────────────────────────┐
│  work  ← struct work_struct         │
│  ┌───────────────────────────────┐  │
│  │  data  (atomic_long_t)        │  │
│  │  entry (list_head) ───────────┼──┼──► [worker pool queue]
│  │  func  (work_func_t) ─────────┼──┼──► driver_callback()
│  │  lockdep_map (optional)       │  │
│  └───────────────────────────────┘  │
│  timer ← struct timer_list          │
│      └── delayed_work_timer_fn() ───┼──► queue_work() after delay
│  wq ──────────────────────────────────► struct workqueue_struct
│  cpu (int)                          │
└─────────────────────────────────────┘
```

#### devm Resource Chain

```
struct device
┌──────────────────────────────┐
│  devres_head (list_head)     │
│      │                       │
│      ▼                       │
│  ┌─────────────────────┐     │
│  │ devres entry #1     │     │
│  │  release = devm_    │     │
│  │  delayed_work_drop  │     │
│  │  data ──────────────┼─────┼──► struct delayed_work *w
│  └─────────────────────┘     │
│      │                       │
│      ▼                       │
│  ┌─────────────────────┐     │
│  │ devres entry #2     │     │
│  │  release = devm_    │     │
│  │  work_drop          │     │
│  │  data ──────────────┼─────┼──► struct work_struct *w
│  └─────────────────────┘     │
└──────────────────────────────┘
```

#### العلاقة الكاملة بين الـ types

```
                    devm-helpers.h
                         │
           ┌─────────────┴─────────────┐
           │                           │
devm_delayed_work_autocancel()   devm_work_autocancel()
           │                           │
     uses: │                     uses: │
  INIT_DELAYED_WORK()           INIT_WORK()
  devm_add_action()             devm_add_action()
           │                           │
     drop: │                     drop: │
devm_delayed_work_drop()        devm_work_drop()
           │                           │
  cancel_delayed_work_sync()    cancel_work_sync()
           │                           │
    struct delayed_work          struct work_struct
           │
    struct work_struct (embedded)
```

---

### 3. Lifecycle Diagrams

#### Lifecycle بتاع `devm_delayed_work_autocancel`

```
driver probe()
    │
    ▼
devm_delayed_work_autocancel(dev, &dw, my_worker)
    │
    ├─► INIT_DELAYED_WORK(&dw, my_worker)
    │       │
    │       ├── INIT_WORK(&dw.work, my_worker)      ← func = my_worker
    │       └── init_timer(&dw.timer, ...)           ← timer ready
    │
    └─► devm_add_action(dev, devm_delayed_work_drop, &dw)
            │
            └── [تسجيل cleanup callback في devres list بتاع الـ device]
    │
    ▼
[driver يشتغل عادي]
    │
    ▼
queue_delayed_work(wq, &dw, delay)   ← driver يطلب تنفيذ مؤجل
    │
    ▼
[بعد delay] timer fires → queue_work() → my_worker() يتنفذ
    │
    ▼
driver remove() / device detach
    │
    ▼
devres unwind (devm cleanup)
    │
    ▼
devm_delayed_work_drop(&dw)
    │
    ▼
cancel_delayed_work_sync(&dw)
    │
    ├── [لو الـ timer لسه شغال: يوقفه]
    ├── [لو الـ work في الـ queue: يشيله]
    └── [لو الـ work بيتنفذ: ينتظر لحد ما يخلص]
    │
    ▼
[مفيش races — الـ work مش هيشتغل تاني]
```

#### Lifecycle بتاع `devm_work_autocancel`

```
driver probe()
    │
    ▼
devm_work_autocancel(dev, &w, my_worker)
    │
    ├─► INIT_WORK(&w, my_worker)
    │       ├── data = WORK_DATA_INIT()   ← WORK_STRUCT_NO_POOL
    │       ├── INIT_LIST_HEAD(&w.entry)
    │       └── w.func = my_worker
    │
    └─► devm_add_action(dev, devm_work_drop, &w)
    │
    ▼
[driver يشتغل — IRQ أو أي كود بيعمل queue_work(wq, &w)]
    │
    ▼
my_worker() يتنفذ في worker thread context
    │
    ▼
driver remove()
    │
    ▼
devm_work_drop(&w)
    │
    ▼
cancel_work_sync(&w)
    ├── [لو pending: يشيله من الـ queue]
    └── [لو running: ينتظر لحد ما يخلص]
```

---

### 4. Call Flow Diagrams

#### `devm_delayed_work_autocancel` — call flow كامل

```
driver_probe()
  │
  └─► devm_delayed_work_autocancel(dev, w, worker_fn)
        │
        ├─► INIT_DELAYED_WORK(w, worker_fn)           [macro]
        │     │
        │     ├─► INIT_WORK(&w->work, worker_fn)       [macro]
        │     │     └─► __INIT_WORK_KEY(w, fn, 0, key)
        │     │           ├── __init_work(w, 0)         [debug check]
        │     │           ├── w->data = WORK_DATA_INIT()
        │     │           ├── INIT_LIST_HEAD(&w->entry)
        │     │           └── w->func = worker_fn
        │     │
        │     └─► __init_timer(&w->timer,
        │               delayed_work_timer_fn,
        │               TIMER_IRQSAFE)
        │
        └─► devm_add_action(dev, devm_delayed_work_drop, w)
              └─► [يضيف entry في dev->devres_head]
                    entry->release = devm_delayed_work_drop
                    entry->data    = w


[لاحقاً — عند الـ remove/detach]

devres_release_all(dev)
  └─► devm_delayed_work_drop(w)          [الـ release callback]
        └─► cancel_delayed_work_sync(w)
              ├─► del_timer_sync(&w->timer)
              └─► cancel_work_sync(&w->work)
                    └─► [ينتظر الـ worker ينهي لو بيشتغل]
```

#### `devm_work_autocancel` — call flow كامل

```
driver_probe()
  │
  └─► devm_work_autocancel(dev, w, worker_fn)
        │
        ├─► INIT_WORK(w, worker_fn)
        │     └─► w->func = worker_fn
        │         w->data = WORK_DATA_INIT()
        │         INIT_LIST_HEAD(&w->entry)
        │
        └─► devm_add_action(dev, devm_work_drop, w)
              └─► [يسجل cleanup في devres list]


[لاحقاً — عند الـ remove/detach]

devres_release_all(dev)
  └─► devm_work_drop(w)
        └─► cancel_work_sync(w)
              └─► [ينتظر الـ worker ينتهي أو يلغيه من الـ queue]
```

#### المشكلة اللي بيحلها الـ devm-helpers — Race Condition Scenario

```
[بدون devm_work_autocancel — المشكلة]

IRQ fires ──────────────────────────────────────────────────┐
                                                             │
driver remove() starts                                       │
  │                                                          ▼
  │                                               schedule_work(&w)
  ▼                                                          │
[remove() finishes]          ┌─────────────────────────────-┘
  │                          │   w في الـ queue
  ▼                          ▼   لكن remove() خلص
[resources freed]    my_worker() بيتنفذ
                       └── بيوصل resources اتحذفت ← UAF/crash!

[مع devm_work_autocancel — الحل]

driver remove()
  │
  ▼
devres cleanup (بعد remove() مباشرة)
  │
  ▼
cancel_work_sync(&w)     ← بينتظر لو الـ work شغال
  │
  ▼
[resources يتحذفوا بأمان — مفيش work هيشتغل تاني]
```

---

### 5. Locking Strategy

#### مفيش locks في devm-helpers.h نفسه

الـ `devm-helpers.h` ملوش locks خاصة بيه. الـ locking بيجي من طبقتين تحته:

#### الطبقة الأولى — devres locking (في `device.h` / `devres.c`)

```
struct device
  └── spinlock_t devres_lock    ← بيحمي dev->devres_head list

devm_add_action()
  └── يمسك devres_lock
        └── يضيف entry في القائمة
        └── يسيب devres_lock
```

#### الطبقة التانية — workqueue locking (في `workqueue.c`)

**الـ `cancel_work_sync` / `cancel_delayed_work_sync`:**

```
Internal WQ locks (spinlocks داخل worker pool)
    ├── pool->lock      ← بيحمي الـ worklist والـ worker state
    └── work->data      ← atomic — PENDING bit بيتضبط atomically
```

#### Lock Ordering في الـ cancel

```
cancel_delayed_work_sync(dw):
  1. del_timer_sync(&dw->timer)    ← ينتظر الـ timer callback تخلص (بدون lock)
  2. cancel_work_sync(&dw->work)
       ├── pool->lock مش مرئي للـ driver
       ├── بيستخدم WORK_STRUCT_PENDING bit atomically
       └── بيستخدم completion/wait_event لينتظر الـ running work
```

#### القاعدة الأساسية للـ devm

```
[devm cleanup order — LIFO]

الـ devres list بتتفك من آخر resource لأول resource
(زي stack)

لو عندك:
  devm_request_irq()          ← registered first
  devm_work_autocancel()      ← registered second

الـ cleanup هيبقى:
  cancel_work_sync()          ← first (LIFO)
  free_irq()                  ← second

ده بيضمن إن الـ work اتلغى قبل ما الـ IRQ يتحرر
→ مفيش IRQ يطلق work جديد بعد الـ cleanup
```

#### ملخص الـ Locking

| المورد | الـ Lock | من بيمسكه |
|--------|----------|-----------|
| `dev->devres_head` | `dev->devres_lock` (spinlock) | devres core |
| `work_struct->data` PENDING bit | atomic operations | workqueue core |
| worker pool worklist | `pool->lock` (spinlock) | workqueue core |
| timer | internal timer lock | timer core |
| "waiting for running work" | `work_completion` lockdep class | `cancel_work_sync` داخلياً |

**الـ driver نفسه مش محتاج يمسك أي lock** — كل الـ locking بيحصل داخل `cancel_work_sync` و`devm_add_action`.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs

| Function / Macro | النوع | الغرض |
|---|---|---|
| `devm_delayed_work_autocancel()` | `static inline int` | تهيئة `delayed_work` مع auto-cancel عند detach |
| `devm_work_autocancel()` | `static inline int` | تهيئة `work_struct` مع auto-cancel عند detach |
| `devm_delayed_work_drop()` | `static inline void` | cleanup callback — يستدعي `cancel_delayed_work_sync()` |
| `devm_work_drop()` | `static inline void` | cleanup callback — يستدعي `cancel_work_sync()` |
| `devm_add_action()` | macro → `__devm_add_action()` | يسجل cleanup action مربوطة بـ lifetime الـ device |
| `INIT_DELAYED_WORK()` | macro | تهيئة `delayed_work` struct وربطها بـ worker function |
| `INIT_WORK()` | macro | تهيئة `work_struct` وربطها بـ worker function |
| `cancel_delayed_work_sync()` | kernel API | يلغي الـ delayed work وينتظر لو بتشتغل |
| `cancel_work_sync()` | kernel API | يلغي الـ work وينتظر لو بتشتغل |

---

### التصنيف المنطقي للـ Functions

الملف `devm-helpers.h` صغير بس مهم — بيقدم **wrapper layer** فوق الـ workqueue API الاعتيادي، هدفها الوحيد هو ضمان إن الـ work items اتلغت تلقائيًا لما الـ driver بيتعمله detach، بدل ما المبرمج يعمل الـ cancellation يدويًا في `remove()`.

الـ functions بتنقسم لـ 3 فئات:

1. **Cleanup Callbacks (Drop Functions)** — `devm_delayed_work_drop`, `devm_work_drop`
2. **Registration / Init Functions** — `devm_delayed_work_autocancel`, `devm_work_autocancel`
3. **Underlying Infrastructure** — `devm_add_action`, `INIT_WORK`, `INIT_DELAYED_WORK`, `cancel_*_sync`

---

### المشكلة اللي بيحلها الملف ده

قبل ما نشرح الـ functions، لازم نفهم الـ race condition اللي بيمنعها الكود ده:

```
remove() بيتشغل
    │
    ├── WQ manually cancelled  ← تم
    │
    └── devm IRQs لسه شغالين  ← IRQ بيطلق
                                    │
                                    └── يجدول work item جديد
                                            │
                                            └── CRASH — resources اترمت!
```

**الـ devm_helpers بتحل ده** بإنها بتربط إلغاء الـ work بـ devres teardown، مش بـ `remove()` مباشرة، فالترتيب بيتضمن إن الـ IRQs والـ WQs بيتلغوا في نفس الـ devres sequence.

---

### الفئة الأولى: Cleanup Callbacks

#### `devm_delayed_work_drop()`

```c
static inline void devm_delayed_work_drop(void *res)
{
    cancel_delayed_work_sync(res);
}
```

**الـ function دي** هي الـ cleanup action اللي بتتسجل في الـ devres framework. لما الـ device بيتعمله detach، الـ devres core بيستدعيها تلقائيًا ويديها الـ pointer للـ `delayed_work`. بتستدعي `cancel_delayed_work_sync()` اللي بتوقف الـ delayed work وبتـ block لحد ما أي instance شغال بيخلص.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `res` | `void *` | pointer للـ `struct delayed_work` — الـ devres core بيمرره كـ `void *` |

**Return value:** void — مفيش return.

**Key details:**
- الـ `cancel_delayed_work_sync()` بتعمل **flush** للـ pending timer وبتنتظر لو الـ work function بتشتغل على أي CPU.
- **Locking:** آمنة تتنادى من أي context لأن `cancel_delayed_work_sync()` نفسها بتتعامل مع الـ locking داخليًا.
- **Caller context:** بيتنادى من الـ devres teardown path — process context فقط لأن `_sync` بتعمل sleep.
- **Side effect:** بعد ما ترجع، مضمون إن الـ work مش شغالة ومش هتتشغل تاني من غير ما حد يجدولها يدويًا.

---

#### `devm_work_drop()`

```c
static inline void devm_work_drop(void *res)
{
    cancel_work_sync(res);
}
```

**الـ function دي** نظيرة `devm_delayed_work_drop()` بس للـ `work_struct` العادي بدون timer. بتضمن إن الـ work item اتلغى وخلص قبل ما الـ devres teardown يكمل.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `res` | `void *` | pointer للـ `struct work_struct` |

**Return value:** void.

**Key details:**
- `cancel_work_sync()` بتشيل الـ work من الـ queue لو لسه pending، أو بتنتظر لو شغالة.
- Process context only بسبب الـ sleep المحتمل.

---

### الفئة التانية: Registration / Init Functions

#### `devm_delayed_work_autocancel()`

```c
static inline int devm_delayed_work_autocancel(struct device *dev,
                                               struct delayed_work *w,
                                               work_func_t worker)
{
    INIT_DELAYED_WORK(w, worker);
    return devm_add_action(dev, devm_delayed_work_drop, w);
}
```

**الـ function دي** بتعمل خطوتين atomically: أولًا بتهيئ الـ `delayed_work` struct وتربطه بالـ worker function، وثانيًا بتسجل `devm_delayed_work_drop` كـ cleanup action في الـ devres system للـ device. النتيجة إن أي وقت الـ device بيتعمله detach، الـ kernel بيستدعي الـ drop function تلقائيًا.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ device اللي الـ work هيتبع lifetime بتاعها |
| `w` | `struct delayed_work *` | الـ work item — الـ caller مسؤول عن allocate الـ memory |
| `worker` | `work_func_t` | الـ callback function اللي هتشتغل لما الـ work يتشغل — signature: `void (*)(struct work_struct *)` |

**Return value:**
- `0` — نجح تسجيل الـ devres action.
- سالب (error code) — فشل `devm_add_action()` في الـ allocation، الـ work مش مسجل للـ auto-cancel.

**Key details:**
- لو `devm_add_action()` فشلت، الـ `INIT_DELAYED_WORK()` اتعملت بالفعل — الـ work struct متهيأ بس مش مربوط بالـ devres. الـ driver لازم يتعامل مع الـ error.
- مفيش **locking** خاص في الـ function نفسها، بس `devm_add_action()` بتستخدم داخليًا الـ devres list lock.
- **Caller context:** process context — الـ function دي بتتنادى عادةً في `probe()`.
- **Memory:** الـ `devm_add_action()` بتعمل `kmalloc` داخليًا لـ devres node — ممكن تفشل لو memory ضيقة.

**Pseudocode flow:**

```
devm_delayed_work_autocancel(dev, w, worker):
    │
    ├── INIT_DELAYED_WORK(w, worker)
    │       ├── init timer_list في w->timer
    │       ├── اربط delayed_work_timer_fn كـ timer callback
    │       └── اربط worker كـ w->work.func
    │
    └── devm_add_action(dev, devm_delayed_work_drop, w)
            ├── kmalloc(devres node)
            ├── سجل (devm_delayed_work_drop, w) في devres list
            └── return 0 أو -ENOMEM
```

**مثال عملي:**

```c
static void my_poll_work(struct work_struct *work)
{
    struct my_dev *priv = container_of(work, struct my_dev, poll_work.work);
    /* read sensor, schedule next poll */
    queue_delayed_work(system_wq, &priv->poll_work, HZ);
}

static int my_probe(struct platform_device *pdev)
{
    struct my_dev *priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    int ret;

    /* بدل INIT_DELAYED_WORK + manual cancel في remove() */
    ret = devm_delayed_work_autocancel(&pdev->dev, &priv->poll_work, my_poll_work);
    if (ret)
        return ret;

    queue_delayed_work(system_wq, &priv->poll_work, HZ);
    return 0;
    /* مفيش remove() محتاج يعمل cancel_delayed_work_sync() */
}
```

---

#### `devm_work_autocancel()`

```c
static inline int devm_work_autocancel(struct device *dev,
                                       struct work_struct *w,
                                       work_func_t worker)
{
    INIT_WORK(w, worker);
    return devm_add_action(dev, devm_work_drop, w);
}
```

**الـ function دي** نفس `devm_delayed_work_autocancel()` بالظبط بس للـ `work_struct` العادي بدون timer delay. بتهيئ الـ work وتسجل `devm_work_drop` كـ cleanup action في الـ devres.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ device اللي الـ work مربوط بـ lifetime بتاعها |
| `w` | `struct work_struct *` | الـ work item |
| `worker` | `work_func_t` | الـ callback function |

**Return value:**
- `0` — نجح.
- سالب — فشل `devm_add_action()`.

**Key details:**
- نفس كل تفاصيل `devm_delayed_work_autocancel()` ما عدا إن ده `work_struct` مش `delayed_work`، فمفيش timer.
- **Caller context:** process context فقط (probe time).
- لو محتاج تجدول الـ work بـ delay، استخدم `devm_delayed_work_autocancel()` بدل ده.

**Pseudocode flow:**

```
devm_work_autocancel(dev, w, worker):
    │
    ├── INIT_WORK(w, worker)
    │       ├── init data field
    │       ├── init list_head entry
    │       └── اربط worker كـ w->func
    │
    └── devm_add_action(dev, devm_work_drop, w)
            ├── kmalloc(devres node)
            ├── سجل (devm_work_drop, w) في devres list
            └── return 0 أو -ENOMEM
```

**مثال عملي:**

```c
static void my_irq_work(struct work_struct *work)
{
    struct my_dev *priv = container_of(work, struct my_dev, irq_work);
    /* process IRQ data in process context */
}

static int my_probe(struct platform_device *pdev)
{
    struct my_dev *priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    int ret;

    ret = devm_work_autocancel(&pdev->dev, &priv->irq_work, my_irq_work);
    if (ret)
        return ret;

    /* devm_request_irq — الـ IRQ والـ WQ كلاهما managed بالـ devres */
    return devm_request_irq(&pdev->dev, irq, my_irq_handler, 0, "my_dev", priv);
}
```

---

### الفئة التالتة: Underlying Infrastructure

دي مش defined في الملف ده بالظبط، بس الـ functions دي هي اللي بيعتمد عليها الـ helpers، فلازم نفهمها عشان الصورة تكتمل.

#### `devm_add_action()` (macro → `__devm_add_action()`)

```c
#define devm_add_action(dev, action, data) \
    __devm_add_action(dev, action, data, #action)

int __devm_add_action(struct device *dev,
                      void (*action)(void *),
                      void *data,
                      const char *name);
```

**الـ macro ده** بيسجل arbitrary cleanup function في الـ devres system. لما الـ device بيتعمله detach (أو بيتعمله `devm_release_action()` يدوي)، الـ kernel بيستدعي `action(data)` تلقائيًا. الـ `#action` stringify trick بتحط اسم الـ function كـ debug name في الـ devres node.

**Parameters:**
| Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ device صاحب الـ devres |
| `action` | `void (*)(void *)` | الـ cleanup callback |
| `data` | `void *` | الـ pointer اللي هيتمرر للـ action عند الـ cleanup |

**Return value:** `0` أو `-ENOMEM`.

---

#### `INIT_WORK()` و `INIT_DELAYED_WORK()`

```c
/* تهيئة work_struct */
#define INIT_WORK(_work, _func) /* ... */

/* تهيئة delayed_work = work_struct + timer_list */
#define INIT_DELAYED_WORK(_work, _func) /* ... */
```

**الـ INIT_WORK** بيعمل zero-init للـ data field، بيعمل `INIT_LIST_HEAD` للـ entry، وبيربط الـ `func` pointer. **الـ INIT_DELAYED_WORK** بيعمل نفس الكلام بس كمان بيهيئ الـ `timer_list` الداخلي بـ `delayed_work_timer_fn` كـ timer callback.

---

### مقارنة: devm vs Manual

| الجانب | Manual (`remove()`) | devm helpers |
|---|---|---|
| مكان الـ cancel | `remove()` callback | devres teardown تلقائي |
| خطر الـ race مع IRQ | موجود — IRQ ممكن يطلق بعد cancel | مضمون — devres بيتعامل مع الترتيب |
| كمية الكود | `cancel_*_sync()` في `probe()` + `remove()` | سطر واحد في `probe()` فقط |
| الـ error handling | سهل التحكم | لازم تتحقق من return value |
| الـ ordering | الـ developer مسؤول | الـ devres framework بيتحكم |

---

### تحذيرات مهمة

- **الـ devm teardown** بيحصل **بعد** `remove()` مش أثناءها، يعني فيه فترة زمنية بين الاتنين — ده هو السبب الأساسي اللي خلى الملف ده موجود أصلًا.
- **مزج devm مع manual** هو أخطر pattern — لو الـ WQ يدوي والـ IRQ devm-managed، الـ IRQ ممكن يطلق بعد `remove()` ويجدول work على WQ اتعمله destroy.
- **الـ `_sync` variants** بتعمل sleep — مش تتنادى من interrupt context أو atomic context أبدًا.
- **الـ `devm_add_action()` ممكن تفشل** — دايمًا تحقق من الـ return value، وإلا الـ work مش هيتلغى تلقائيًا.
## Phase 5: دليل الـ Debugging الشامل

الـ `devm-helpers.h` بيوفر آليات الـ **devm** (device-managed resources) لإلغاء تسجيل الـ work items تلقائياً عند detach الـ driver. المشاكل اللي بتظهر هنا بتكون دايماً في ثلاث حالات: work بيشتغل بعد ما الـ device اتشال، أو resource leak لما devm_add_action بيفشل، أو race condition بين الـ remove() والـ work handler.

---

### Software Level

#### 1. debugfs — المدخلات المهمة

الـ workqueue subsystem عنده entries جاهزة في `/sys/kernel/debug/`:

```bash
# شوف كل الـ workqueues الموجودة مع إحصائياتها
ls /sys/kernel/debug/workqueue/

# اقرأ إحصائيات workqueue معين (مثلاً events)
cat /sys/kernel/debug/workqueue/events/

# شوف الـ worker pools الحالية
cat /sys/kernel/debug/workqueue/pool_ids
```

الـ output بيكون زي ده:

```
events          per-cpu pwq 0: cpus=0 node=0 flags=0x0 nice=0 active=1/256 refcnt=2
                  in-flight: 22:my_work_handler
                  pending: another_work_handler
```

- **active=1/256**: 1 work شغال من أصل max 256
- **in-flight: 22:func_name**: الـ work الحالي والـ PID بتاعه

#### 2. sysfs — المدخلات المهمة

```bash
# شوف الـ device وهل اتحذف أم لا
cat /sys/bus/<bus>/devices/<dev>/uevent

# تحقق من الـ driver المرتبط بالـ device
ls -la /sys/bus/<bus>/devices/<dev>/driver

# شوف الـ devm resources المسجلة (kernel >= 5.4)
cat /sys/kernel/debug/devices_deferred  # لو موجود
```

#### 3. ftrace — Tracepoints المهمة

```bash
# فعّل tracing للـ workqueue events
echo 1 > /sys/kernel/debug/tracing/events/workqueue/enable

# أو tracepoints محددة
echo 1 > /sys/kernel/debug/tracing/events/workqueue/workqueue_queue_work/enable
echo 1 > /sys/kernel/debug/tracing/events/workqueue/workqueue_execute_start/enable
echo 1 > /sys/kernel/debug/tracing/events/workqueue/workqueue_execute_end/enable
echo 1 > /sys/kernel/debug/tracing/events/workqueue/workqueue_activate_work/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

الـ output هيكون:

```
kworker/0:1-12    [000] ....  1234.567: workqueue_queue_work: work struct=0xffff... function=my_worker workqueue=events req_cpu=8 cpu=0
kworker/0:1-12    [000] ....  1234.568: workqueue_execute_start: work struct=0xffff... function=my_worker
```

لو شفت `workqueue_queue_work` بعد ما الـ driver اتحذف → race condition واضحة.

#### 4. printk / Dynamic Debug

```bash
# فعّل dynamic debug للـ devm subsystem
echo 'file drivers/base/devres.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لكل الـ devm operations
echo 'module <module_name> +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شوف إيه اللي enabled حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep devres
```

في الكود نفسه، أضف `pr_debug()` في الأماكن الحرجة:

```c
static void my_worker(struct work_struct *w)
{
    struct my_dev *priv = container_of(w, struct my_dev, work);

    /* يتحقق إن الـ device لسه موجود */
    pr_debug("%s: worker started, dev refcount=%d\n",
             __func__, kref_read(&priv->dev->kobj.kref));
}
```

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_DEBUG_OBJECTS_WORK` | يكتشف double-queue وuse-after-free للـ work_struct |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في الـ cancel_work_sync() |
| `CONFIG_DEBUG_DEVRES` | يطبع كل devm allocations/frees |
| `CONFIG_WQ_WATCHDOG` | يكتشف لو الـ workqueue متعطل (stall) |
| `CONFIG_PROVE_LOCKING` | يتحقق من صحة locking order |
| `CONFIG_KASAN` | يكتشف memory corruption في الـ work handlers |
| `CONFIG_KCSAN` | يكتشف data races بين الـ worker والـ remove() |

```bash
# تأكد إن الـ configs مفعلة في kernel حالي
zcat /proc/config.gz | grep -E 'DEBUG_OBJECTS_WORK|DEBUG_DEVRES|WQ_WATCHDOG'
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# شوف كل الـ work items الـ pending في كل الـ workqueues
cat /proc/sys/kernel/workqueue/

# WQ watchdog timeout (بالثواني)
cat /sys/module/workqueue/parameters/watchdog_thresh

# اعمل dump لكل الـ workqueue states (مفيد في الـ hung tasks)
echo t > /proc/sysrq-trigger  # يطبع الـ workqueue info في dmesg
```

**الـ devlink** مش مباشراً relevant هنا لأن `devm-helpers` بتخدم كل الـ bus types. بس لو الـ driver بيشتغل مع network device:

```bash
devlink dev info <dev_name>
devlink health show
```

#### 7. جدول رسائل الـ Error الشائعة

| رسالة في dmesg | المعنى | الحل |
|----------------|--------|-------|
| `BUG: workqueue leaked lock or atomic` | الـ work handler مات وعنده lock | راجع الـ locking في worker function |
| `WARNING: CPU: X PID: Y at kernel/workqueue.c` | عملت queue لـ work بعد ما الـ wq اتحذف | استخدم `devm_work_autocancel` بدل manual management |
| `INFO: task kworker/...:... blocked for more than 120s` | الـ `cancel_work_sync()` بيستنى شغل مش بيخلص | الـ worker نفسه معلق، شوف لو في deadlock |
| `Modules linked in: ...` مع NULL pointer في worker | الـ driver اتحذف والـ worker لسه بيشتغل | الـ race اللي `devm_work_autocancel` المفروض يمنعه |
| `devm_add_action failed` | فشل kmalloc في تسجيل الـ cleanup action | تحقق من الـ memory pressure |
| `kernel BUG at kernel/workqueue.c:XXXX` | استخدام `INIT_WORK` أكتر من مرة على نفس الـ struct | `devm_work_autocancel` يعمل init مرة واحدة بس |

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
static int devm_work_autocancel(struct device *dev,
                                struct work_struct *w,
                                work_func_t worker)
{
    INIT_WORK(w, worker);

    /* WARN لو الـ device في مرحلة الـ removal */
    WARN_ON(dev->bus && !device_is_registered(dev));

    return devm_add_action(dev, devm_work_drop, w);
}

static void my_worker(struct work_struct *w)
{
    struct my_priv *priv = container_of(w, struct my_priv, work);

    /* WARN_ON لو الـ device مش موجود */
    if (WARN_ON(!priv->dev))
        return;

    /* dump_stack هنا لو محتاج تعرف مين schedule الـ work ده */
    /* dump_stack(); */  /* uncomment للـ debugging فقط */
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يتطابق مع الـ Kernel State

الـ `devm-helpers` نفسه مش بيتكلم مباشرةً مع الـ hardware، بس الـ work handlers بتاعة الـ driver بتعمل كده. تحقق من:

```bash
# شوف هل الـ device لسه موجود فعلاً على الـ bus
lspci -vvv | grep -A 20 "<vendor_id>"       # PCI
lsusb -v | grep -A 20 "<vendor_id>"         # USB
ls /sys/bus/i2c/devices/                    # I2C
ls /sys/bus/spi/devices/                    # SPI

# قارن الـ state في الـ kernel مع الـ hardware
cat /sys/bus/<bus>/devices/<dev>/power/runtime_status
```

#### 2. Register Dump Techniques

```bash
# قرا registers عن طريق /dev/mem (محتاج CONFIG_DEVMEM=y)
devmem2 0xFEDC0000 w   # اقرأ word من عنوان معين

# لو مش موجود devmem2، استخدم dd
dd if=/dev/mem bs=4 count=1 skip=$((0xFEDC0000/4)) 2>/dev/null | xxd

# لو الـ driver بيستخدم ioremap، شوف الـ mappings الحالية
cat /proc/iomem | grep -i "<device_name>"

# لـ PCI devices
lspci -s <BDF> -xxx    # dump كل الـ config space
setpci -s <BDF> 0x10.l  # اقرأ BAR0
```

#### 3. Logic Analyzer / Oscilloscope

الـ devm work helpers غالباً بتيجي مع:

- **IRQ-triggered work**: استخدم logic analyzer على الـ IRQ line وراقب التوقيت بين IRQ assertion وتنفيذ الـ worker.
- **I2C/SPI transactions داخل الـ worker**: راقب الـ bus lines بالـ oscilloscope وتأكد إن الـ transactions مش بتبدأ بعد الـ remove().
- **نقطة القياس**: على الـ CS (chip select) لـ SPI أو الـ SCL/SDA لـ I2C.
- **الـ trigger**: edge على الـ IRQ line، trace لمدة 100ms+ عشان تشوف لو في activity بعد الـ remove.

```
IRQ Line:    ______|‾|_____________|‾|______
                   ↑               ↑
              IRQ fires        IRQ after remove? (bug!)

Worker exec: ________|‾‾‾‾|_________|‾‾? (should not happen)
```

#### 4. Hardware Issues الشائعة → Kernel Log Patterns

| المشكلة الـ HW | Pattern في dmesg | السبب |
|----------------|------------------|-------|
| الـ device اتشال فجأة (hot-unplug) | `Unable to handle kernel paging request` في worker | الـ work بيشتغل بعد unmap الـ registers |
| IRQ stuck high | `irq X: nobody cared` | الـ worker اللي المفروض يـ clear الـ IRQ اتلغى |
| Power glitch | `kworker: BUG: scheduling while atomic` | الـ device reset أثناء تنفيذ الـ worker |
| Spurious IRQ | `workqueue: ...: was not idle` | الـ IRQ بيتنفر من hardware والـ work queue ممتلية |

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT node للـ device
ls /sys/firmware/devicetree/base/
cat /sys/firmware/devicetree/base/<path_to_node>/compatible

# قارن مع الـ DT source
dtc -I fs /sys/firmware/devicetree/base/ 2>/dev/null | grep -A 10 "<node_name>"

# تحقق من الـ interrupts في الـ DT
cat /sys/firmware/devicetree/base/<node>/interrupts | xxd

# شوف إيه الـ interrupts الـ registered فعلاً
cat /proc/interrupts | grep <driver_name>
```

---

### Practical Commands

#### سكريبت شامل لـ devm Work Debugging

```bash
#!/bin/bash
# devm-work-debug.sh — يساعد في تشخيص مشاكل devm work helpers

DEV_NAME="${1:-my_device}"   # اسم الـ device أو الـ driver

echo "=== Workqueue Status ==="
# شوف كل الـ work items الـ pending
cat /sys/kernel/debug/workqueue/* 2>/dev/null | head -50

echo ""
echo "=== Active kworkers ==="
ps aux | grep kworker

echo ""
echo "=== Recent dmesg for device ==="
dmesg --since "1 hour ago" | grep -i "$DEV_NAME"

echo ""
echo "=== devm resources (if available) ==="
cat /sys/kernel/debug/devices_deferred 2>/dev/null | head -20
```

#### تفعيل Workqueue Debugging الكامل

```bash
# 1. فعّل WQ watchdog
echo 30 > /sys/module/workqueue/parameters/watchdog_thresh

# 2. فعّل كل الـ WQ tracepoints
for event in workqueue_queue_work workqueue_execute_start workqueue_execute_end; do
    echo 1 > /sys/kernel/debug/tracing/events/workqueue/$event/enable
done

# 3. فعّل tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 4. افعل الـ operation اللي بتسبب المشكلة (مثلاً: insmod ثم rmmod)
modprobe <driver_name>
sleep 1
modprobe -r <driver_name>

# 5. اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace | grep -A 2 -B 2 "my_worker"

# 6. أوقف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

#### التحقق من Race Condition بالـ ftrace

```bash
# راقب لو في work بيتنفذ بعد device removal
# قبل rmmod:
echo 'workqueue_queue_work' > /sys/kernel/debug/tracing/set_event
echo 1 > /sys/kernel/debug/tracing/tracing_on

# بعد rmmod:
# افتح trace_pipe وشوف لو في events من الـ driver اتحذف
timeout 5 cat /sys/kernel/debug/tracing/trace_pipe | \
    grep "<driver_work_function_name>"

# النتيجة المتوقعة الصحيحة: لا output بعد الـ rmmod
# النتيجة اللي بتدل على bug: events بعد الـ rmmod
```

**مثال output صح (بدون race):**

```
# اللي المفروض تشوفه بعد rmmod: سكوت تام
```

**مثال output فيه race condition:**

```
kworker/2:1-456  [002] ....  9999.001: workqueue_execute_start: function=my_driver_worker
```

ده معناه إن الـ work اتنفذ بعد الـ remove، وده بالظبط اللي `devm_work_autocancel` المفروض يمنعه.

#### تشخيص فشل devm_add_action

```bash
# فعّل DEBUG_DEVRES
# في kernel command line أو /etc/default/grub:
# GRUB_CMDLINE_LINUX="... devres.debug=1"

# أو في runtime لو CONFIG_DYNAMIC_DEBUG مفعل:
echo 'file drivers/base/devres.c +p' > /sys/kernel/debug/dynamic_debug/control

# بعدين اعمل modprobe وشوف الـ output
dmesg | grep -E 'devres|devm'
```

**مثال output:**

```
[   12.345] devres: allocated 32 bytes for devm_work_drop at drivers/mydrv.c:45
[   12.346] devres: released devm_work_drop (my_driver remove)
```

لو مشفتش السطر التاني → الـ resource اتلغى قبل ما الـ remove يشتغل (bug).
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Kernel Oops في industrial gateway بعد `rmmod`

#### العنوان
**Use-after-free في UART driver على RK3562 بسبب خلط devm مع manual cancel**

#### السياق
شركة بتبني industrial gateway بيشغّل Modbus RTU على UART2 في RK3562. الـ driver بيقرأ بيانات من sensors عبر IRQ، وبيعالجها في `work_struct` عادي (مش devm). الـ product بيتشغّل في مصنع وبيعمل hot-unplug للـ module أثناء الشغل.

#### المشكلة
بعد `rmmod uart_modbus.ko`، الـ system بيـcrash بـ:

```
BUG: unable to handle kernel NULL pointer dereference at 0000000000000018
Call Trace:
  process_one_work+0x...
  modbus_rx_worker+0x...       ← worker بيحاول يوصل لـ struct اتحذف
```

الـ crash مش consistent — بيحصل تقريباً 1 من كل 20 مرة `rmmod`.

#### التحليل
الـ driver الأصلي كان زي كده:

```c
static int modbus_probe(struct platform_device *pdev)
{
    struct modbus_dev *mdev = devm_kzalloc(&pdev->dev, sizeof(*mdev), GFP_KERNEL);

    /* IRQ registered with devm — سيُلغى تلقائياً بعد remove() */
    devm_request_irq(&pdev->dev, irq, modbus_irq_handler, 0, "modbus", mdev);

    /* Work registered manually — مش devm */
    INIT_WORK(&mdev->rx_work, modbus_rx_worker);
}

static int modbus_remove(struct platform_device *pdev)
{
    /* بيلغي الـ work */
    cancel_work_sync(&mdev->rx_work);
    return 0;
}
```

المشكلة: الـ `devm_request_irq` بيتلغى **بعد** ما `remove()` يخلص. يعني:

```
remove() runs:
  cancel_work_sync(&rx_work)    ← OK, work cancelled
  return 0                      ← remove() خلص

devm cleanup runs (بعد remove):
  free_irq(...)                 ← IRQ لسه شغال!
  ↓
  IRQ fires between remove() and devm cleanup
  → modbus_irq_handler() → schedule_work(&rx_work)
  → worker runs → mdev already freed → OOPS
```

الـ header نفسه يشرح الخطر ده صراحةً في الـ comment:

```c
/*
 * IRQ scheduling a work in a queue is typical example where IRQs are
 * often devm-managed and WQs are manually cleaned at remove(). If IRQs
 * are not manually freed at remove() ... we have a period of time after
 * remove() - and before devm managed IRQs are freed - where new IRQ may
 * fire and schedule a work item which won't be cancelled.
 */
```

#### الحل
استخدام `devm_work_autocancel` من `devm-helpers.h` بدل `INIT_WORK` يدوياً:

```c
#include <linux/devm-helpers.h>

static int modbus_probe(struct platform_device *pdev)
{
    struct modbus_dev *mdev = devm_kzalloc(&pdev->dev, sizeof(*mdev), GFP_KERNEL);

    /* Work الآن devm-managed — سيُلغى بعد IRQ مباشرة في devm cleanup */
    ret = devm_work_autocancel(&pdev->dev, &mdev->rx_work, modbus_rx_worker);
    if (ret)
        return ret;

    /* IRQ أيضاً devm-managed */
    devm_request_irq(&pdev->dev, irq, modbus_irq_handler, 0, "modbus", mdev);

    return 0;
}

/* remove() مش محتاج cancel_work_sync بعد كده */
```

الـ devm cleanup بيلغي كل الـ resources بالترتيب العكسي للتسجيل. لأن `devm_work_autocancel` اتسجّل **قبل** `devm_request_irq`، هيتنفّذ **بعده** في الـ cleanup — فالـ IRQ بيتحرر الأول، تم بعدين الـ work يتلغى.

#### الدرس المستفاد
لو الـ IRQ والـ work كلاهما devm، لازم ترتيب التسجيل يكون: **work أولاً، IRQ ثانياً** — عشان الـ cleanup يلغي IRQ قبل الـ work.

---

### السيناريو 2: SPI sensor driver على STM32MP1 بيـhang عند `unbind`

#### العنوان
**`cancel_delayed_work_sync` بيـblock إلى الأبد في IMU driver**

#### السياق
بورد STM32MP1 في IoT sensor node. الـ driver بيقرأ IMU (accelerometer) عبر SPI كل 100ms باستخدام `delayed_work`. الـ engineer بيعمل `echo -n > /sys/bus/spi/drivers/imu_driver/spi0.0/driver/unbind` لتغيير الـ sampling rate.

#### المشكلة
الـ unbind بيـblock تماماً — الـ terminal بيفضل في حالة waiting ومفيش response. الـ system مش hung لكن الـ unbind process مش بترجع أبداً.

#### التحليل
الـ driver كان عنده logic غلط في الـ worker نفسه:

```c
static void imu_poll_worker(struct work_struct *work)
{
    struct imu_dev *imu = container_of(work, struct imu_dev, poll_dwork.work);

    /* قراءة SPI */
    imu_read_spi(imu);

    /* Worker بيعيد جدولة نفسه دايماً */
    schedule_delayed_work(&imu->poll_dwork, msecs_to_jiffies(100));
}
```

الـ `remove()` كان بيعمل:

```c
static int imu_remove(struct platform_device *pdev)
{
    cancel_delayed_work_sync(&imu->poll_dwork);
    return 0;
}
```

المشكلة: الـ `cancel_delayed_work_sync` بيـwait للـ work الحالي يخلص. لكن الـ worker نفسه بيعمل `schedule_delayed_work` من جوّه — فالـ cancel بيحاول يلغي work ثم في نفس الوقت بيتجدول work جديد، وده ممكن يعمل race condition.

لو الـ driver استخدم `devm_delayed_work_autocancel`:

```c
static inline void devm_delayed_work_drop(void *res)
{
    cancel_delayed_work_sync(res);  /* ده اللي بيتنفّذ في devm cleanup */
}
```

الـ `cancel_delayed_work_sync` بيضمن إن الـ work مش شغال وقت الـ cancel، لكن لو الـ worker schedule نفسه تاني، الـ pending work الجديد مش بيتلغى لأن `cancel_delayed_work_sync` بتلغي الـ work الحالي بس.

#### الحل
تصحيح الـ worker عشان يفحص flag قبل ما يعيد جدولة نفسه:

```c
static void imu_poll_worker(struct work_struct *work)
{
    struct imu_dev *imu = container_of(work, struct imu_dev, poll_dwork.work);

    imu_read_spi(imu);

    /* فقط أعد الجدولة لو الـ device لسه موجود */
    if (!imu->removing)
        schedule_delayed_work(&imu->poll_dwork, msecs_to_jiffies(100));
}
```

ثم في الـ probe:

```c
static int imu_probe(struct platform_device *pdev)
{
    imu->removing = false;

    /* devm يضمن cancel بعد unbind */
    return devm_delayed_work_autocancel(&pdev->dev,
                                        &imu->poll_dwork,
                                        imu_poll_worker);
}
```

وفي الـ remove:

```c
static int imu_remove(struct platform_device *pdev)
{
    imu->removing = true;   /* أوقف إعادة الجدولة */
    /* devm_delayed_work_drop سيُستدعى تلقائياً بعدين */
    return 0;
}
```

#### الدرس المستفاد
`devm_delayed_work_autocancel` مش بتحل مشكلة الـ self-rescheduling workers. لازم الـ worker يكون aware بالـ removal state. الـ devm helper بيتكفل بالـ cancel عند الـ cleanup، لكن الـ logic الداخلي للـ worker مسؤولية الـ developer.

---

### السيناريو 3: Android TV Box على Allwinner H616 — HDMI hotplug crash

#### العنوان
**Race condition في HDMI HPD handler على H616 بيعمل panic عند فصل الشاشة**

#### السياق
TV box بيشغّل Android 13 على Allwinner H616. الـ HDMI driver بيـdetect hotplug عبر GPIO IRQ ويبعت event للـ userspace. المنتج في إيد المستخدم ومن الطبيعي إنه يفصل ويوصّل الشاشة.

#### المشكلة
عند فصل الشاشة أثناء أو بعد `unbind` للـ driver (مثلاً عند suspend)، الـ system بيعمل:

```
kernel BUG at kernel/workqueue.c:1672!
invalid opcode: 0000 [#1] PREEMPT SMP
```

#### التحليل
الـ driver الأصلي:

```c
struct hdmi_dev {
    struct work_struct hpd_work;  /* بيتعامل مع HPD events */
    int hpd_irq;
};

static int hdmi_probe(struct platform_device *pdev)
{
    INIT_WORK(&hdmi->hpd_work, hdmi_hpd_worker);

    hdmi->hpd_irq = devm_request_irq(&pdev->dev, irq,
                                      hdmi_irq_handler,
                                      IRQF_TRIGGER_BOTH,
                                      "hdmi-hpd", hdmi);
    return 0;
}

static int hdmi_remove(struct platform_device *pdev)
{
    /* المطوّر نسي cancel_work_sync هنا! */
    return 0;
}
```

الـ sequence اللي بيعمل crash:

```
1. hdmi_remove() يتنفّذ → يرجع 0 فوراً (مفيش cancel)
2. devm cleanup: free_irq(hpd_irq) ← IRQ اتحرر
3. لكن BETWEEN خطوة 1 و 2: HPD IRQ fires
4. hdmi_irq_handler → schedule_work(&hdmi->hpd_work)
5. hdmi_hpd_worker يتنفّذ → يحاول يوصل لـ hdmi struct → اتـfree بالفعل
   → BUG_ON
```

الحل باستخدام `devm_work_autocancel`:

```c
static int hdmi_probe(struct platform_device *pdev)
{
    int ret;

    /* سجّل الـ work أولاً */
    ret = devm_work_autocancel(&pdev->dev,
                               &hdmi->hpd_work,
                               hdmi_hpd_worker);
    if (ret)
        return ret;

    /* سجّل الـ IRQ ثانياً — سيُلغى قبل الـ work في cleanup */
    return devm_request_irq(&pdev->dev, irq,
                            hdmi_irq_handler,
                            IRQF_TRIGGER_BOTH,
                            "hdmi-hpd", hdmi);
}
```

ترتيب الـ devm cleanup (LIFO):

```
devm cleanup order:
  1. free_irq(hpd_irq)           ← IRQ اتحرر — مفيش IRQs جديدة
  2. cancel_work_sync(hpd_work)  ← work اتلغى بأمان
  3. kfree(hdmi)                 ← struct اتحرر
```

#### الدرس المستفاد
**ترتيب التسجيل في devm = عكس ترتيب التحرير**. دايماً سجّل الـ work قبل الـ IRQ عشان الـ IRQ يتحرر أولاً في الـ cleanup.

---

### السيناريو 4: i.MX8 automotive ECU — delayed work بيشغّل بعد driver removal

#### العنوان
**`devm_delayed_work_autocancel` بيحل مشكلة watchdog timer في CAN bus driver**

#### السياق
ECU في سيارة بيستخدم i.MX8QM. الـ driver بيعمل health check لـ CAN bus كل 500ms باستخدام `delayed_work`. لو الـ CAN silent لأكتر من 2 ثانية، الـ driver بيعمل reset للـ peripheral. الـ system بيعمل partial shutdown لبعض الـ subsystems أثناء الـ driving.

#### المشكلة
أثناء الـ partial shutdown، الـ CAN driver بيتـremove لكن الـ watchdog work بيفضل شغال ويحاول يعمل reset لـ peripheral مش موجود:

```
[  342.1] can_watchdog_worker: Cannot access CAN registers — device removed
[  342.1] Oops: general protection fault
```

#### التحليل
الـ driver القديم:

```c
static int can_probe(struct platform_device *pdev)
{
    INIT_DELAYED_WORK(&can->watchdog_dwork, can_watchdog_worker);
    schedule_delayed_work(&can->watchdog_dwork, msecs_to_jiffies(500));
    return 0;
}

static int can_remove(struct platform_device *pdev)
{
    /* المطور استخدم cancel_delayed_work بدل _sync! */
    cancel_delayed_work(&can->watchdog_dwork);
    return 0;
}
```

الفرق المهم:
- `cancel_delayed_work` — بتطلب cancel لكن مش بتستنى الـ worker يخلص
- `cancel_delayed_work_sync` — بتستنى الـ worker الشغال يخلص قبل ما ترجع

بعد `can_remove()` رجع، ممكن يكون الـ worker لسه شغال:

```
can_remove():
  cancel_delayed_work(&watchdog) ← request cancel (non-blocking)
  return 0                       ← رجعت وسيب الـ worker شغال!

devm cleanup:
  kfree(can)                    ← can struct اتحذف

can_watchdog_worker (لسه شغال!):
  can->regs->ctrl = RESET;      ← use-after-free → OOPS
```

#### الحل
```c
#include <linux/devm-helpers.h>

static int can_probe(struct platform_device *pdev)
{
    int ret;

    /* devm_delayed_work_autocancel تستخدم cancel_delayed_work_sync داخلياً */
    ret = devm_delayed_work_autocancel(&pdev->dev,
                                       &can->watchdog_dwork,
                                       can_watchdog_worker);
    if (ret)
        return ret;

    /* ابدأ الـ watchdog */
    schedule_delayed_work(&can->watchdog_dwork, msecs_to_jiffies(500));
    return 0;
}

/* remove() مش محتاج يعمل أي حاجة للـ work */
static int can_remove(struct platform_device *pdev)
{
    return 0;
}
```

الـ `devm_delayed_work_drop` اللي بتتنفّذ في cleanup:

```c
static inline void devm_delayed_work_drop(void *res)
{
    cancel_delayed_work_sync(res);  /* blocking — بتستنى الـ worker يخلص */
}
```

#### الدرس المستفاد
الفرق بين `cancel_delayed_work` و`cancel_delayed_work_sync` critical في السياقات الـ safety-critical. `devm_delayed_work_autocancel` دايماً بتستخدم الـ `_sync` version — ده مقصود وضروري.

---

### السيناريو 5: AM62x custom board bring-up — driver بيـfail silently

#### العنوان
**`devm_add_action` بيرجع error والـ developer مش بيـcheck الـ return value**

#### السياق
bring-up بورد custom بيستخدم AM62x (Texas Instruments) مع I2C temperature sensor. الـ driver بيستخدم `devm_work_autocancel` لأول مرة. الـ engineer شايف إن الـ driver بيـload بدون error، لكن عند `rmmod` الـ system بيعمل crash.

#### المشكلة
```
[   5.2] i2c_temp_sensor: probe succeeded
[  42.8] rmmod i2c_temp
[  42.9] BUG: scheduling while atomic
[  42.9] Call Trace: cancel_work_sync ← بيتنفّذ من context غلط
```

#### التحليل
الـ probe code:

```c
static int temp_probe(struct i2c_client *client)
{
    struct temp_dev *dev = devm_kzalloc(&client->dev,
                                         sizeof(*dev), GFP_KERNEL);

    INIT_WORK(&dev->poll_work, temp_poll_worker);

    /* المطور نسي يـcheck الـ return value! */
    devm_work_autocancel(&client->dev, &dev->poll_work, temp_poll_worker);

    /* ده بيعمل INIT_WORK مرة تانية — يبطل التسجيل السابق */
    INIT_WORK(&dev->poll_work, temp_poll_worker);

    return 0;
}
```

الـ `devm_work_autocancel` هي:

```c
static inline int devm_work_autocancel(struct device *dev,
                                       struct work_struct *w,
                                       work_func_t worker)
{
    INIT_WORK(w, worker);                        /* خطوة 1: init الـ work */
    return devm_add_action(dev, devm_work_drop, w); /* خطوة 2: سجّل cleanup */
}
```

المشكلة المزدوجة:
1. الـ engineer عمل `INIT_WORK` يدوياً بعد الـ `devm_work_autocancel` — ده reset الـ work struct وكسر الـ devm cleanup pointer.
2. لو `devm_add_action` فشل (ممكن لو الـ memory ضيّق)، الـ return value اتجاهل، والـ work registered بدون cleanup.

يعني ممكن ييجي crash من مسارين:
```
Path A: devm_add_action fails silently
  → work never registered for cleanup
  → cancel_work_sync not called at remove
  → worker accesses freed memory → crash

Path B: INIT_WORK called after devm_work_autocancel
  → devm_work_drop pointer still points to old (now reinit'd) work
  → cancel_delayed_work_sync on wrong state → BUG
```

#### الحل
```c
static int temp_probe(struct i2c_client *client)
{
    struct temp_dev *dev = devm_kzalloc(&client->dev,
                                         sizeof(*dev), GFP_KERNEL);
    int ret;

    /* استخدم devm_work_autocancel بس — لا INIT_WORK يدوي قبلها أو بعدها */
    ret = devm_work_autocancel(&client->dev,
                               &dev->poll_work,
                               temp_poll_worker);
    if (ret) {
        dev_err(&client->dev, "Failed to init devm work: %d\n", ret);
        return ret;
    }

    schedule_work(&dev->poll_work);
    return 0;
}
```

فحص بسيط بعد الـ probe:

```bash
# تأكد إن الـ devm actions اتسجلت
cat /sys/kernel/debug/devices/<device>/devres
# أو
echo -n > /sys/bus/i2c/drivers/i2c_temp/1-0048/driver/unbind && dmesg | tail -5
```

#### الدرس المستفاد
- دايماً check الـ return value من `devm_work_autocancel` و`devm_delayed_work_autocancel` — الـ `devm_add_action` اللي بيتنادى جوّاهم ممكن يفشل.
- لا تعمل `INIT_WORK` يدوياً على نفس الـ `work_struct` بعد ما تعمل `devm_work_autocancel` — الـ helper بيعمل الـ init جواه.
- الـ double-init للـ work struct من أخطر الأخطاء في الـ workqueue code.
## Phase 7: مصادر ومراجع

### مصادر رسمية — Kernel Documentation

| المصدر | الرابط |
|--------|--------|
| **Devres - Managed Device Resource** (الوثيقة الرسمية) | [docs.kernel.org](https://docs.kernel.org/driver-api/driver-model/devres.html) |
| **Devres RST source** على kernel.org | [kernel.org/doc](https://www.kernel.org/doc/Documentation/driver-api/driver-model/devres.rst) |
| **Driver implementer's API guide** | [docs.kernel.org/driver-api](https://docs.kernel.org/driver-api/index.html) |
| **Concurrency Managed Workqueue (cmwq)** | [docs.kernel.org/core-api/workqueue](https://docs.kernel.org/core-api/workqueue.html) |
| **Driver Model index** | [docs.kernel.org/driver-api/driver-model](https://docs.kernel.org/driver-api/driver-model/index.html) |

الملف المباشر في الـ kernel source:
```
include/linux/devm-helpers.h
Documentation/driver-api/driver-model/devres.rst
```

---

### مقالات LWN.net

دي أهم المقالات اللي بتشرح الـ devres / devm framework من الأساس:

| المقال | الوصف |
|--------|--------|
| [Managed device resources](https://lwn.net/Articles/215861/) | المقال الأصلي اللي قدّم الـ devres API في الـ kernel |
| [The managed resource API](https://lwn.net/Articles/222860/) | تفاصيل تقنية أعمق عن الـ API بعد الـ merge |
| [Device resource management](https://lwn.net/Articles/215996/) | نقاش حول إدارة الـ resources في الـ device drivers |
| [Kernel development (devres context)](https://lwn.net/Articles/215235/) | السياق العام لتطوير الـ devres في الـ kernel |
| [drm_device managed resources](https://lwn.net/Articles/812802/) | مثال متقدم على توسيع الـ devm pattern في الـ DRM subsystem |
| [Devm helpers for regulator get and enable](https://lwn.net/Articles/904557/) | مثال حديث على إضافة devm helpers جديدة للـ regulators |

---

### مناقشات Mailing List

- **[LKML: devm_add_action_or_reset patch](https://linux.kernel.narkive.com/t2eaUpTx/patch-1-2-devm-add-helper-devm-add-action-or-reset)**
  الـ patch اللي أضاف `devm_add_action_or_reset()` — وهي الـ function الأساسية اللي بتتبنى عليها `devm_delayed_work_autocancel()` و `devm_work_autocancel()`.

- **[LKML: Add device-managed allocate workqueue](https://lkml.org/lkml/2026/2/23/228)**
  نقاش حديث جداً (فبراير 2026) حول إضافة devm-managed workqueue allocation — موضوع مرتبط مباشرة بـ `devm-helpers.h`.

- **[LKML: devm_delayed_work_autocancel return code](https://lkml.org/lkml/2025/11/3/45)**
  نقاش عن الـ error handling الصحيح لـ `devm_delayed_work_autocancel()`.

- **[LKML archive](https://lkml.org/)** — ابحث بـ: `devm_work_autocancel`, `devm_delayed_work_autocancel`, `devm-helpers`

---

### KernelNewbies

| الرابط | الموضوع |
|--------|---------|
| [KernelMemoryAllocation](https://kernelnewbies.org/KernelMemoryAllocation) | شرح الـ devm_kzalloc وأخواتها مع مقارنة بالـ manual allocation |
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | تتبع التغييرات في الـ kernel عبر الإصدارات |
| [Linux_2_6_21](https://kernelnewbies.org/Linux_2_6_21) | الإصدار اللي دخل فيه الـ devres framework لأول مرة |
| [Mailing list discussion: devm_kzalloc on probe fail](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2020-April/020730.html) | نقاش عملي حول سلوك الـ devm عند فشل الـ probe |

---

### مصادر تعليمية إضافية

| المصدر | الوصف |
|--------|--------|
| [Deferred work — linux-kernel-labs](https://linux-kernel-labs.github.io/refs/heads/master/labs/deferred_work.html) | شرح عملي للـ workqueues والـ deferred work مع أمثلة |
| [Device Model — linux-kernel-labs](https://linux-kernel-labs.github.io/refs/heads/master/labs/device_model.html) | الـ device model كاملاً مع الـ devres |
| [Devres PDF — Eli Billauer, Haifa Linux Club](http://www.haifux.org/lectures/323/haifux-devres.pdf) | presentation ممتاز بيشرح الـ devres من الألف للياء |
| [Devm functions correct usage — blog](https://vthakkar1994.wordpress.com/2015/08/24/devm-functions-and-their-correct-usage/) | أمثلة عملية وأخطاء شائعة |

---

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل الأساسي**: Chapter 3 — *Char Drivers* (resource cleanup patterns)
- **الفصل 14**: *The Linux Device Model* — فيه شرح الـ `struct device` والـ driver lifecycle
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: *Devices and Modules* — بيشرح الـ driver model والـ resource management
- **الفصل 7**: *Interrupts and Interrupt Handlers* — مرتبط بموضوع الـ IRQ + devm

#### Mastering Linux Device Driver Development — John Madieu (Packt)
- الكتاب الأحدث اللي بيغطي الـ devm APIs الحديثة بشكل تفصيلي
- بيشرح `devm_request_irq()`, `devm_add_action()`, وpattern الـ cleanup

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 16**: *Kernel Initialization* — context مهم لفهم الـ driver lifecycle

---

### مسارات البحث المقترحة

للي عايز يتعمق أكتر، دي أهم الـ search terms:

```
# في git log على الـ kernel source
git log --all --oneline -- include/linux/devm-helpers.h
git log --grep="devm_work_autocancel"
git log --grep="devm_delayed_work_autocancel"

# في LKML
site:lkml.org "devm-helpers"
site:lkml.org "devm_add_action" workqueue cancel
site:lkml.org "devm_work_autocancel"

# في kernel source
grep -r "devm_work_autocancel" drivers/
grep -r "devm_delayed_work_autocancel" drivers/
```

**الـ search terms** للبحث العام:
- `linux kernel devm managed resource cleanup`
- `devres linux kernel driver detach automatic`
- `devm_add_action custom cleanup kernel`
- `linux workqueue cancel driver remove devm`
- `devm helpers mixing manual resource management bug`
## Phase 8: Writing simple module

### الفكرة

**`devm_add_action`** هي الدالة اللي بتربط cleanup callback بـ device lifetime — وهي اللي بيتكل عليها كل من `devm_work_autocancel` و`devm_delayed_work_autocancel`. هنعمل kprobe على `devm_add_action_or_reset` (الـ exported الأكثر استخداماً) عشان نشوف كل مرة driver بيسجّل devm action ومين الـ device اللي بتتسجل عليه وإيه الـ callback.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * devm_action_probe.c
 *
 * kprobe on devm_add_action_or_reset() to log every devm-managed
 * cleanup action registered by any driver at runtime.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit       */
#include <linux/kprobes.h>      /* kprobe, register_kprobe, ...           */
#include <linux/device.h>       /* struct device, dev_name()              */
#include <linux/printk.h>       /* pr_info()                              */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs Project");
MODULE_DESCRIPTION("kprobe on devm_add_action_or_reset to trace devm actions");

/*
 * pre_handler — بيتنفذ قبل أول instruction في الدالة المستهدفة.
 * الـ regs بتحتوي على قيم الـ registers عند نقطة الـ probe.
 * على x86_64:
 *   rdi = arg0 = struct device *dev
 *   rsi = arg1 = void (*action)(void *)
 *   rdx = arg2 = void *data
 */
static int devm_action_pre(struct kprobe *kp, struct pt_regs *regs)
{
    struct device *dev;
    void (*action)(void *);
    void *data;

#ifdef CONFIG_X86_64
    dev    = (struct device *)regs->di;   /* rdi — first argument  */
    action = (void (*)(void *))regs->si;  /* rsi — second argument */
    data   = (void *)regs->dx;            /* rdx — third argument  */
#elif defined(CONFIG_ARM64)
    dev    = (struct device *)regs->regs[0]; /* x0 */
    action = (void (*)(void *))regs->regs[1]; /* x1 */
    data   = (void *)regs->regs[2];          /* x2 */
#else
    /* fallback — just log that we were called */
    pr_info("devm_probe: called (arch not decoded)\n");
    return 0;
#endif

    /* dev_name() بترجع اسم الـ device زي "0000:00:1f.3" */
    pr_info("devm_probe: dev='%s' action=%pS data=%px\n",
            dev ? dev_name(dev) : "(null)",
            action,   /* %pS يطبع symbol name + offset */
            data);

    return 0; /* 0 = continue normal execution */
}

/* الـ kprobe struct — بيحدد الـ symbol اللي هنضع عليه الـ probe */
static struct kprobe kp = {
    .symbol_name = "devm_add_action_or_reset",
    .pre_handler = devm_action_pre,
};

static int __init devm_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("devm_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("devm_probe: kprobe planted on %s @ %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit devm_probe_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما الـ module يتانلود —
     * لو سبنا الـ probe شغّال والـ module اتانلود،
     * الـ pre_handler هيبقى pointer لكود اتحذف → kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("devm_probe: kprobe removed\n");
}

module_init(devm_probe_init);
module_exit(devm_probe_exit);
```

---

### Makefile

```makefile
obj-m += devm_action_probe.o

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
| `linux/module.h` | ماكرو `module_init`/`module_exit` والـ `MODULE_*` |
| `linux/kprobes.h` | `struct kprobe` و`register_kprobe`/`unregister_kprobe` |
| `linux/device.h` | `struct device` و`dev_name()` |
| `linux/printk.h` | `pr_info()` و`pr_err()` |

#### الـ pre_handler

الـ callback بياخد `struct pt_regs *regs` اللي بيمثل حالة الـ CPU registers عند نقطة الـ probe. بنستخرج منه الـ arguments على حسب الـ calling convention للـ architecture — على x86_64 الـ arguments الأول بيتحطوا في `rdi, rsi, rdx, rcx, r8, r9`.

`%pS` في `pr_info` هو format specifier خاص بالـ kernel بيطبع اسم الـ symbol من الـ symbol table — يعني بدل ما تطبع عنوان، بتطبع مثلاً `devm_work_drop+0x0/0x10`.

#### الـ kprobe struct

**`symbol_name`** بيخلي الـ kernel يبحث عن الـ symbol في `kallsyms` ويحسب العنوان تلقائياً — مش محتاج تعرف العنوان hardcoded. **`pre_handler`** بيتشغّل قبل تنفيذ الـ instruction الأول في الدالة، وهو الوقت المثالي لقراءة الـ arguments.

#### الـ init / exit

في الـ `init` بنسجّل الـ kprobe وبنطبع العنوان اللي اتحسب. في الـ `exit` لازم نعمل `unregister_kprobe` قبل ما الـ module يتحذف من الـ memory — لأن الـ kernel لو استدعى الـ handler بعد التحذف هيتقابل بكود مش موجود ويحصل kernel panic.

---

### تشغيل وقراءة الـ output

```bash
# بناء وتحميل الـ module
make
sudo insmod devm_action_probe.ko

# مشاهدة الـ output في real-time
sudo dmesg -w | grep devm_probe

# مثال على ما ستراه لما أي driver يعمل probe:
# devm_probe: dev='0000:00:02.0' action=devm_work_drop+0x0/0x10 data=00000000abcd1234
# devm_probe: dev='i2c-0'        action=devm_delayed_work_drop+0x0/0x10 data=...

# إزالة الـ module
sudo rmmod devm_action_probe
```

**الـ `action=devm_work_drop+0x0/0x10`** يأكد إن الـ driver استخدم `devm_work_autocancel` — لأن `devm_work_drop` هو بالظبط الـ callback اللي بيتسجّل.
