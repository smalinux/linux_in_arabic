## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

الملف `gpiolib-shared.h` جزء من **GPIO Subsystem** في Linux kernel.
المسؤولون عنه:
- **Linus Walleij** `<linusw@kernel.org>`
- **Bartosz Golaszewski** `<brgl@kernel.org>`
- الـ mailing list: `linux-gpio@vger.kernel.org`

---

### المشكلة اللي بيحلها الملف ده — القصة الكاملة

#### تخيل السيناريو ده

عندك شريحة SoC (نظام على شريحة واحدة) زي Raspberry Pi أو أي embedded board.
في الـ hardware عندك **GPIO pin رقم 5** على الـ GPIO controller.

ده الـ pin بيتحكم في **reset** لجهازين في نفس الوقت:
- `device_A` — بيستخدمه عشان يعمل reset لنفسه
- `device_B` — بيستخدمه برضو عشان يعمل reset لنفسه

في الـ Device Tree (ملف وصف الـ hardware):

```dts
device_a {
    reset-gpios = <&gpio0 5 GPIO_ACTIVE_LOW>;
};

device_b {
    reset-gpios = <&gpio0 5 GPIO_ACTIVE_LOW>;
};
```

**المشكلة الكلاسيكية:** الـ GPIO subsystem في Linux قديمًا بيعامل كل GPIO pin كـ exclusive resource — يعني واحد بس يقدر يطلبه في وقت واحد. لو `device_A` طلب GPIO 5، جه `device_B` يطلبه برضو → **EBUSY** → فشل.

الحل القديم كان `GPIOD_FLAGS_BIT_NONEXCLUSIVE` — إنك تقول "أنا عارف إن ده shared، سيبني أطلبه رغم كده." لكن ده solution خطر لأنه ما بيعملش أي **synchronization** بين الـ consumers. يعني `device_A` يقدر يرفع الـ pin و`device_B` يخفضه في نفس اللحظة → **race condition**.

---

#### الحل الجديد: GPIO Shared Infrastructure

الكود في `gpiolib-shared.h` و`gpiolib-shared.c` جاء بفكرة مختلفة تمامًا:

**بدل ما نديهم الـ GPIO مباشرة، نعمل وسيط — Proxy.**

الفكرة:
1. الـ kernel بيمسح كل الـ Device Tree nodes في بداية التشغيل.
2. بيلاقي كل GPIO pin اتذكر في أكتر من device واحد.
3. لكل consumer، بيعمل **auxiliary device** وهمي اسمه `proxy`.
4. الـ proxy ده هو اللي بيمسك الـ GPIO الحقيقي، مش الـ consumer مباشرة.
5. الـ consumers بيتعاملوا مع الـ proxy، والـ proxy بيتعامل مع الـ GPIO الحقيقي.

كمان بيتحكم في الـ synchronization عبر `struct gpio_shared_desc` اللي فيه:
- إما `mutex` لو الـ GPIO يمكن يـ sleep (slow GPIO expanders على I2C مثلًا)
- أو `spinlock` لو الـ GPIO سريع ومحتاجش sleep (memory-mapped GPIO)

ده الـ core لل **shared GPIO** infrastructure.

---

### شرح ELI5 — تخيل بواب عمارة

تخيل عندك **مفتاح ضوء واحد** في حجرة السلم بيتحكم في ضوء الدور الأول والدور التاني في نفس الوقت.

- **الطريقة القديمة:** كل واحد يمسك المفتاح لوحده — ممكن حد يشغل وحد تاني يطفي في نفس اللحظة → فوضى.
- **الطريقة الجديدة (gpiolib-shared):** في **بواب** (الـ proxy) يقف عند المفتاح. أي حد عايز يعدّل الضوء بيقول للبواب. البواب بيتعامل مع الطلبات بشكل منظّم واحد واحد، وبيحسب كام واحد شايل الضوء (reference counting).

---

### هيكل البيانات الأساسي في الملف

```c
struct gpio_shared_desc {
    struct gpio_desc *desc;    /* مؤشر للـ GPIO الحقيقي */
    bool can_sleep;            /* هل الـ GPIO بطيء (I2C) أو سريع (memory-mapped)? */
    unsigned long cfg;         /* إعدادات الـ pin */
    unsigned int usecnt;       /* عدد الـ consumers اللي بيستخدموه دلوقتي */
    unsigned int highcnt;      /* عدد الـ consumers اللي عايزينه HIGH */
    union {
        struct mutex mutex;    /* للـ GPIO اللي ممكن يـ sleep */
        spinlock_t spinlock;   /* للـ GPIO السريع */
    };
};
```

الـ `usecnt` و`highcnt` بيخليك تعمل **majority vote** أو **reference counting** على الـ GPIO state — لما كل الـ consumers يرفعوا الـ pin، يتشغل. لما آخر واحد يخليه، يتوقف.

---

### الـ Lock Guard الذكي

الملف بيعرّف macro بيعمل auto-locking بناءً على نوع الـ GPIO:

```c
DEFINE_LOCK_GUARD_1(gpio_shared_desc_lock, struct gpio_shared_desc,
    /* acquire */
    if (_T->lock->can_sleep)
        mutex_lock(&_T->lock->mutex);
    else
        spin_lock_irqsave(&_T->lock->spinlock, _T->flags),
    /* release */
    if (_T->lock->can_sleep)
        mutex_unlock(&_T->lock->mutex);
    else
        spin_unlock_irqrestore(&_T->lock->spinlock, _T->flags),
    unsigned long flags)
```

ده بيخلي أي كود يعمل `guard(gpio_shared_desc_lock)(shared_desc)` ويدي للـ scope-based locking تلقائيًا — من `include/linux/cleanup.h`.

---

### الـ API اللي بيصدّره الملف

| الـ Function | الغرض |
|---|---|
| `gpio_device_setup_shared(gdev)` | لما GPIO controller بيتسجل، بيشوف لو عنده shared pins وبيعمل الـ proxy devices |
| `gpio_device_teardown_shared(gdev)` | لما الـ controller بيتشال، بيحذف الـ proxies |
| `gpio_shared_add_proxy_lookup(consumer, con_id, lflags)` | بيضيف lookup table entry تربط الـ consumer بالـ proxy الخاص بيه |
| `devm_gpiod_shared_get(dev)` | الـ proxy driver بيستخدمها عشان يجيب `gpio_shared_desc` المشترك |

---

### متى بيتفعّل الكود ده؟

الكود ده **مفيش** إلا لو `CONFIG_GPIO_SHARED=y` في kernel config. لو مش مفعّل، كل الدوال بتبقى empty inline stubs بترجع `0`.

الـ `gpio_shared_init()` بيتشغل بـ `postcore_initcall` — يعني في بداية التشغيل جدًا قبل ما أي driver يتسجّل، بيمسح كل الـ Device Tree ويبني قائمة الـ shared GPIOs.

---

### الملفات المرتبطة اللي لازم تعرفها

#### Core الـ subsystem

| الملف | الدور |
|---|---|
| `drivers/gpio/gpiolib.c` | اللب الأساسي للـ GPIO subsystem — التسجيل، الطلب، التحرير |
| `drivers/gpio/gpiolib.h` | الـ internal structs زي `gpio_device` و`gpio_desc` |
| `drivers/gpio/gpiolib-shared.c` | التنفيذ الكامل لنظام الـ shared GPIOs |
| `drivers/gpio/gpiolib-shared.h` | **(ملفنا)** — الـ interface والـ structs المشتركة بين الـ shared GPIO components |
| `drivers/gpio/gpiolib-devres.c` | الـ devres (device-managed resources) wrappers للـ GPIO |

#### الـ Headers العامة

| الملف | الدور |
|---|---|
| `include/linux/gpio/consumer.h` | الـ API العام للـ consumers — `gpiod_get()`, `gpiod_set_value()`, إلخ |
| `include/linux/gpio/driver.h` | الـ API للـ GPIO controller drivers |
| `include/linux/gpio/machine.h` | الـ `gpiod_lookup_table` — ربط الـ consumers بالـ GPIO pins |

#### الـ Firmware / DT support

| الملف | الدور |
|---|---|
| `drivers/gpio/gpiolib-of.c` | قراءة GPIO info من Device Tree |
| `drivers/gpio/gpiolib-acpi-core.c` | قراءة GPIO info من ACPI tables |

#### ذات صلة بالـ reset GPIO

| الملف | الدور |
|---|---|
| `drivers/reset/reset-gpio.c` | الـ driver الخاص بـ GPIO-based reset controllers — بيستخدم الـ shared GPIO infrastructure |
| `Documentation/driver-api/gpio/` | Documentation الرسمية للـ subsystem |
## Phase 2: شرح الـ GPIO Shared Framework

### المشكلة — ليه الـ Subsystem ده موجود أصلاً؟

في الـ embedded Linux، الـ GPIO pin الواحد بيبقى hardware line فيزيائية واحدة على الـ SoC. الـ kernel بيفترض تاريخياً إن كل pin بيتملكه **consumer واحد بس** — لما driver بيطلب GPIO عبر `gpiod_get()` الـ pin بيتـ mark كـ `GPIOD_FLAG_REQUESTED` وأي محاولة تانية للطلب عليه بترجع `-EBUSY`.

المشكلة الحقيقية اللي بتظهر في الـ production hardware:

```
  ┌─────────────────────────────────────┐
  │          SoC                        │
  │  GPIO_42 ──────┬──── RESET line     │
  │                │                    │
  │           WiFi Module               │
  │           Audio Codec               │
  │         (same physical pin!)        │
  └─────────────────────────────────────┘
```

يعني **pin reset واحد** بيتحكم في أكتر من device في نفس الوقت. ده بيحصل كتير في:

- **Reset signals** مشتركة بين أكتر من chip على نفس الـ reset line
- **Power enable lines** بتشغّل أكتر من device
- **Interrupt lines** مربوطة على أكتر من مصدر

قبل الـ GPIO Shared framework، الـ workaround المعتاد كان:
1. إن consumer واحد بس يطلب الـ GPIO ويتحكم فيه بالنيابة عن الباقيين — وده بيكسر الـ driver boundaries
2. إن الـ driver يـ skip الـ GPIO request ويتعامل مع الـ register مباشرةً — وده unsupported وهش
3. إضافة `GPIOD_FLAGS_BIT_NONEXCLUSIVE` اللي اتـ deprecate لأنه مكنش safe

---

### الحل — الـ Kernel بيعمل إيه؟

الـ **GPIO Shared framework** (أُضيف في 2025 بواسطة Linaro) بيحل المشكلة بـ **indirection layer** — بدل ما كل consumer يطلب الـ GPIO مباشرةً من الـ GPIO chip، الـ framework بيعمل:

1. **Scan مبكر** للـ firmware nodes (Device Tree) في وقت الـ `postcore_initcall` — قبل ما أي device driver يشتغل — ويحدد كل pin بيظهر في أكتر من consumer node.

2. **Proxy Auxiliary Devices**: لكل consumer من الـ consumers المشتركين، بيتعمل `auxiliary_device` وهمي (proxy) يمثّله. كل proxy بيبقى عنده "مظهر" إنه GPIO controller صغير عنده pin واحد بس.

3. **Shared Descriptor** مشترك: كل الـ proxies بتشاور على `gpio_shared_desc` واحد فيه الـ real GPIO descriptor، وعنده mutex أو spinlock لحماية الـ concurrent access.

4. **Reference Counting**: استخدام `kref` لضمان إن الـ shared_desc مش بيتحرر من الـ memory قبل ما آخر consumer يسيبه.

---

### الـ Real-World Analogy — التشبيه العميق

تخيّل **مبنى مكاتب** فيه **غرفة اجتماعات واحدة** (= الـ GPIO pin):

| المبنى الحقيقي | مقابله في الـ Kernel |
|---|---|
| الغرفة الفيزيائية | الـ `gpio_desc` الحقيقي في `gpio_device->descs[]` |
| نظام الحجز (Booking System) | الـ `gpio_shared_desc` مع الـ lock |
| شركة في المبنى | كل consumer device (WiFi, Codec) |
| بطاقة دخول الشركة للغرفة | الـ `auxiliary_device` (proxy) لكل consumer |
| مكتب الاستقبال (بيمسك قائمة الحجوزات) | الـ `gpio_shared_entry` |
| قواعد الاستخدام المشترك | الـ `usecnt` و`highcnt` في `gpio_shared_desc` |
| الحارس اللي بيتحكم في الدخول | الـ `mutex`/`spinlock` في `gpio_shared_desc` |

الفكرة العميقة: الغرفة موجودة فيزيائياً مرة واحدة — مش ممكن تنسخها. لكن نظام الحجز بيسمح لأكتر من شركة تستخدمها بشكل منظم. بنفس المنطق، الـ GPIO pin موجود فيزيائياً مرة واحدة، لكن الـ proxy system بتسمح لأكتر من driver يتعامل معه بأمان.

---

### الـ Big Picture Architecture

```
Boot Time: postcore_initcall
════════════════════════════════════════════════════════════

  Device Tree (OF Scan)
  ┌──────────────────────────────────┐
  │ consumer_a { reset-gpios = <&gpio1 42 0> } │
  │ consumer_b { reset-gpios = <&gpio1 42 0> } │
  └──────────────────┬───────────────┘
                     │ gpio_shared_of_traverse()
                     ▼
  gpio_shared_list (Global Linked List)
  ┌────────────────────────────────────────────────┐
  │  gpio_shared_entry (pin: gpio1[42])            │
  │  ├── fwnode → &gpio1 controller node           │
  │  ├── offset = 42                               │
  │  ├── shared_desc → (NULL, until first get)     │
  │  └── refs:                                     │
  │       ├── gpio_shared_ref (consumer_a)         │
  │       │    ├── fwnode → consumer_a node        │
  │       │    ├── con_id = "reset"                │
  │       │    └── adev → proxy_a                  │
  │       └── gpio_shared_ref (consumer_b)         │
  │            ├── fwnode → consumer_b node        │
  │            ├── con_id = "reset"                │
  │            └── adev → proxy_b                  │
  └────────────────────────────────────────────────┘

Runtime: When GPIO chip registers (gpio_device_setup_shared)
════════════════════════════════════════════════════════════

  gpio_device (gpio1)
  ┌─────────────────────────────────────────────┐
  │ descs[42].flags |= GPIOD_FLAG_SHARED        │
  │ gpiod_request_commit(descs[42], "shared")   │  ← pre-claimed!
  └───────────────┬─────────────────────────────┘
                  │ gpio_shared_make_adev()
                  ▼
  Auxiliary Bus
  ┌──────────────────────┐  ┌──────────────────────┐
  │ proxy_a (adev)       │  │ proxy_b (adev)       │
  │ parent = gpio1.dev   │  │ parent = gpio1.dev   │
  │ platform_data = entry│  │ platform_data = entry│
  └──────────┬───────────┘  └──────────┬───────────┘
             │ probe()                  │ probe()
             ▼                          ▼
  consumer_a driver               consumer_b driver
  devm_gpiod_shared_get()         devm_gpiod_shared_get()
             │                          │
             └──────────┬───────────────┘
                        ▼
              gpio_shared_desc (shared!)
              ┌────────────────────────┐
              │ desc    → descs[42]    │
              │ can_sleep = true/false │
              │ usecnt  (# requesters)│
              │ highcnt (# driving H) │
              │ mutex or spinlock      │
              │ kref   (ref count)     │
              └────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ **core abstraction** هي: **"فصل الـ ownership عن الـ access"**.

في الـ GPIO subsystem العادي: الـ ownership = الـ access. من يطلب الـ GPIO يملكه ويتحكم فيه حصرياً.

في الـ GPIO Shared framework: الـ `gpio_device` يملك الـ pin فيزيائياً ويحجزه مسبقاً (`GPIOD_FLAG_SHARED + gpiod_request_commit`). لكن الـ access بيتوزّع عبر `gpio_shared_desc` المشترك اللي بيـ export نفسه لكل consumer عبر proxy.

```
العادي:
consumer_a ──gpiod_get()──► gpio_desc (owned, locked)
consumer_b ──gpiod_get()──► -EBUSY ✗

المشترك:
gpio_device ──────────────► gpio_desc (owned by framework)
                               │
consumer_a ──proxy_a──────► gpio_shared_desc ──► gpio_desc
consumer_b ──proxy_b──────► gpio_shared_desc ──► gpio_desc
```

الـ `gpio_shared_desc` هو الـ single point of truth لحالة الـ pin. الـ consumers لا يتعاملون مع الـ `gpio_desc` مباشرةً أبداً.

---

### الـ Key Data Structures والعلاقات بينها

```
gpio_shared_entry
┌─────────────────────────────────────────────┐
│ fwnode_handle *fwnode  ← GPIO controller   │
│ unsigned int offset    ← pin number        │
│ mutex lock             ← protects refs     │
│ gpio_shared_desc *shared_desc ─────────────┼──┐
│ kref ref               ← refcount         │  │
│ list_head list         ← global list      │  │
│ list_head refs ────────────────────────────┼──┼──► gpio_shared_ref(s)
└─────────────────────────────────────────────┘  │
                                                  │
gpio_shared_desc ◄────────────────────────────────┘
┌─────────────────────────────────────────────┐
│ gpio_desc *desc        ← الـ real GPIO     │
│ bool can_sleep                              │
│ unsigned long cfg      ← current config    │
│ unsigned int usecnt    ← # active users    │
│ unsigned int highcnt   ← # driving high    │
│ union {                                     │
│   struct mutex mutex;    (if can_sleep)     │
│   spinlock_t spinlock;   (if atomic-safe)  │
│ }                                           │
└─────────────────────────────────────────────┘

gpio_shared_ref
┌─────────────────────────────────────────────┐
│ fwnode_handle *fwnode  ← consumer node     │
│ gpiod_flags flags      ← requested flags  │
│ char *con_id           ← "reset", etc.    │
│ int dev_id             ← IDA allocated    │
│ mutex lock                                  │
│ auxiliary_device adev  ← the proxy dev   │
│ gpiod_lookup_table *lookup                 │
│ bool is_reset_gpio                          │
│ list_head list         ← in entry->refs   │
└─────────────────────────────────────────────┘
```

---

### الـ Locking Strategy — ليه mutex وspinlock في نفس الوقت؟

ده قرار design مهم جداً. الـ `gpio_shared_desc` بيحتوي على:

```c
struct gpio_shared_desc {
    struct gpio_desc *desc;
    bool can_sleep;
    unsigned long cfg;
    unsigned int usecnt;
    unsigned int highcnt;
    union {
        struct mutex mutex;      /* للـ GPIO chips اللي can_sleep */
        spinlock_t spinlock;     /* للـ GPIO chips اللي atomic-safe */
    };
};
```

الـ **`can_sleep` flag** بيتحدد من الـ GPIO chip driver نفسه:
- لو الـ chip محتاج I2C أو SPI عشان يتحكم في الـ GPIO (زي GPIO expanders) → `can_sleep = true` → لازم `mutex` لأن مش ممكن تـ sleep في atomic context
- لو الـ chip access عبر memory-mapped registers (زي GPIO controllers جوه الـ SoC) → `can_sleep = false` → ينفع `spinlock`

الـ **`DEFINE_LOCK_GUARD_1`** macro في الـ header بيوفّر **scoped locking** — بيستخدم الـ cleanup mechanism (من `linux/cleanup.h`) عشان الـ lock يتحرر أوتوماتيكاً في نهاية الـ scope:

```c
/* من gpiolib-shared.h */
DEFINE_LOCK_GUARD_1(gpio_shared_desc_lock, struct gpio_shared_desc,
    /* acquire: */
    if (_T->lock->can_sleep)
        mutex_lock(&_T->lock->mutex);
    else
        spin_lock_irqsave(&_T->lock->spinlock, _T->flags),
    /* release: */
    if (_T->lock->can_sleep)
        mutex_unlock(&_T->lock->mutex);
    else
        spin_unlock_irqrestore(&_T->lock->spinlock, _T->flags),
    unsigned long flags)
```

الـ `lockdep_assert_held` في `gpio_shared_lockdep_assert()` بيتأكد في وقت الـ debug إن الـ caller ماسك الـ lock الصح قبل ما يعمل أي عملية على الـ shared descriptor.

---

### الـ Lifecycle الكامل من البداية للنهاية

#### 1. Boot Scan (postcore_initcall)

```c
/* من gpiolib-shared.c */
static int __init gpio_shared_init(void)
{
    ret = gpio_shared_of_scan();    /* امسح كل الـ DT nodes */
    gpio_shared_free_exclusive();   /* احذف الـ entries اللي مش shared فعلاً */
}
postcore_initcall(gpio_shared_init);
```

الـ `gpio_shared_of_traverse()` بيمشي على كل node في الـ Device Tree وبيدوّر على properties إسمها `*-gpios` أو `*-gpio`. لو لقى نفس الـ GPIO controller + offset في أكتر من node، بيخلق `gpio_shared_entry` ويحطه في `gpio_shared_list`.

#### 2. GPIO Chip Registration (gpio_device_setup_shared)

لما GPIO chip driver بيشتغل ويسجّل نفسه، الـ framework بيـ intercept ده:

```c
int gpio_device_setup_shared(struct gpio_device *gdev)
{
    /* لو ده proxy device → علّم الـ descriptor وارجع */
    /* لو ده الـ real GPIO chip → خلق الـ proxy devices */
    __set_bit(GPIOD_FLAG_SHARED, &desc->flags);
    gpiod_request_commit(desc, "shared");  /* احجز الـ pin للـ framework */
    gpio_shared_make_adev(gdev, entry, ref);  /* خلق auxiliary device لكل consumer */
}
```

#### 3. Consumer Driver Probe

لما consumer driver بيـ probe ويطلب `devm_gpiod_get(dev, "reset", GPIOD_OUT_LOW)`، الـ framework بيـ redirect الطلب لـ proxy:

```c
/* gpio_shared_add_proxy_lookup يضيف machine lookup entry */
lookup->table[0] = GPIO_LOOKUP(proxy_key, 0, con_id, lflags);
gpiod_add_lookup_table(ref->lookup);
```

ثم `devm_gpiod_shared_get()` بيرجع الـ `gpio_shared_desc` المشترك:

```c
struct gpio_shared_desc *devm_gpiod_shared_get(struct device *dev)
{
    entry = dev_get_platdata(dev);  /* الـ entry حاطته في platform_data */

    scoped_guard(mutex, &entry->lock) {
        if (entry->shared_desc) {
            kref_get(&entry->ref);          /* consumer تاني → زوّد الـ refcount */
            shared_desc = entry->shared_desc;
        } else {
            shared_desc = gpiod_shared_desc_create(entry);  /* أول consumer */
            kref_init(&entry->ref);
        }
    }

    devm_add_action_or_reset(dev, gpiod_shared_put, entry);  /* cleanup أوتوماتيكي */
    return shared_desc;
}
```

#### 4. Release (devm cleanup)

لما consumer driver بيتـ unload، الـ devm cleanup بيـ call `gpiod_shared_put()` → `kref_put()`. لما آخر consumer يسيب الـ shared_desc، الـ `gpio_shared_release()` بيـ free الـ `gpio_shared_desc`.

---

### الـ Reset GPIO — Special Case مهم

**الـ reset-gpio subsystem** (من `CONFIG_RESET_GPIO`) بيخلق `auxiliary_device` ديناميكياً لكل device عنده `reset-gpios` property. المشكلة إن الـ GPIO Shared framework بيعمل الـ scan مرة واحدة في وقت الـ boot قبل ما الـ reset-gpio device يتخلق.

الحل: لما الـ framework يلاقي `reset-gpios` property أثناء الـ scan، بيخلق **secondary shared ref** مسبقاً للـ reset-gpio device:

```c
if (strcmp(prop->name, "reset-gpios") == 0) {
    ret = gpio_shared_setup_reset_proxy(entry, args.args[1]);
}
```

وعند الـ probe، `gpio_shared_dev_is_reset_gpio()` بيتحقق إن الـ consumer الجديد هو فعلاً الـ reset-gpio device المتوقع عبر مقارنة الـ firmware references.

---

### الـ Framework يملك إيه مقابل إيه بيفوّضه للـ Drivers؟

| **الـ GPIO Shared Framework يملك** | **بيفوّض للـ Drivers** |
|---|---|
| الـ scan الـ firmware للـ shared pins | تحديد `can_sleep` (الـ GPIO chip driver) |
| إنشاء وإدارة الـ proxy auxiliary devices | منطق الـ set/get الفعلي للـ GPIO value |
| الـ `gpio_shared_desc` وحماية الـ concurrent access | قرار متى يرفع/يخفض الـ GPIO (الـ consumer driver) |
| الـ reference counting عبر `kref` | تفسير الـ `usecnt` و`highcnt` لتحديد الـ state |
| الـ machine lookup tables للـ proxy devices | سلوك الـ GPIO chip hardware layer |
| الـ lifecycle management (setup/teardown) | اختيار mutex أو spinlock (مبني على `can_sleep`) |

---

### Subsystems الأخرى اللي الـ Framework بيعتمد عليها

- **Auxiliary Bus** (`linux/auxiliary_bus.h`): مكانيزم عشان kernel subsystem يخلق "virtual devices" من غير hardware حقيقي — هنا بيُستخدم لخلق الـ proxy devices.
- **SRCU** (Sleepable RCU): مستخدم في الـ `gpio_device` لحماية الـ pointer للـ `gpio_chip` — مهم لأن الـ chip ممكن يتنزع (unbind) في أي وقت.
- **Pinctrl Subsystem**: لو `CONFIG_PINCTRL` مفعّل، الـ GPIO subsystem بيـ coordinate مع الـ pinctrl لتحديد الـ pin mux. الـ GPIO Shared framework مش بيتدخل في ده مباشرةً.
- **Device Tree / OF** (`linux/of.h`): المصدر الوحيد الحالي للـ shared GPIO detection. الـ ACPI path مش supported بعد.
- **IDA** (ID Allocator): عشان كل `gpio_shared_ref` يبقى عنده unique ID يستخدمه في تسمية الـ auxiliary device.
- **lockdep** (`linux/lockdep.h`): الـ `lockdep_register_key()` في كل ref بيضمن إن الـ lock validator يعرف يفرق بين locks المختلفة حتى لو نفس النوع — ده important لأن كل ref عنده mutex مستقل.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Config Options والـ Flags — Cheatsheet

#### Config Options

| Config | الأثر |
|--------|-------|
| `CONFIG_GPIO_SHARED` | يفعّل كامل منظومة الـ shared GPIO — بدونه كل الـ API بتبقى stubs ترجع 0 |
| `CONFIG_GPIOLIB` | الـ core library للـ GPIO — بدونه معظم الـ API معطّل |
| `CONFIG_GPIOLIB_IRQCHIP` | يدمج الـ IRQ chip جوه الـ gpiolib |
| `CONFIG_PINCTRL` | يضيف الـ `pin_ranges` list جوه `gpio_device` |
| `CONFIG_GPIO_CDEV` | يضيف الـ `debounce_period_us` للـ `gpio_desc` |
| `CONFIG_OF_DYNAMIC` | يضيف الـ `hog` pointer للـ `gpio_desc` |
| `CONFIG_LOCKDEP` | يخلّي `gpiochip_add_data` يسجّل static lock class keys |

#### الـ `gpio_desc->flags` Bit Numbers

| Bit | Symbol | المعنى |
|-----|--------|--------|
| 0 | `GPIOD_FLAG_REQUESTED` | الـ line اتطلب من consumer |
| 1 | `GPIOD_FLAG_IS_OUT` | الاتجاه: output |
| 2 | `GPIOD_FLAG_EXPORT` | مصدَّر لـ userspace |
| 3 | `GPIOD_FLAG_SYSFS` | مصدَّر عبر `/sys/class/gpio` |
| 6 | `GPIOD_FLAG_ACTIVE_LOW` | المنطق معكوس (active-low) |
| 7 | `GPIOD_FLAG_OPEN_DRAIN` | open-drain |
| 8 | `GPIOD_FLAG_OPEN_SOURCE` | open-source |
| 9 | `GPIOD_FLAG_USED_AS_IRQ` | متوصّل بـ IRQ |
| 10 | `GPIOD_FLAG_IRQ_IS_ENABLED` | الـ IRQ مفعَّل |
| 11 | `GPIOD_FLAG_IS_HOGGED` | الـ line محجوز بـ hog |
| 12 | `GPIOD_FLAG_TRANSITORY` | الـ value ممكن يتضيّع في sleep/reset |
| 13 | `GPIOD_FLAG_PULL_UP` | pull-up مفعَّل |
| 14 | `GPIOD_FLAG_PULL_DOWN` | pull-down مفعَّل |
| 15 | `GPIOD_FLAG_BIAS_DISABLE` | bias معطَّل صراحةً |
| 16 | `GPIOD_FLAG_EDGE_RISING` | CDEV يرصد rising edge |
| 17 | `GPIOD_FLAG_EDGE_FALLING` | CDEV يرصد falling edge |
| 18 | `GPIOD_FLAG_EVENT_CLOCK_REALTIME` | timestamps من REALTIME clock |
| 19 | `GPIOD_FLAG_EVENT_CLOCK_HTE` | timestamps من hardware timestamp engine |
| 20 | `GPIOD_FLAG_SHARED` | الـ line مشترَك بين أكثر من consumer |
| 21 | `GPIOD_FLAG_SHARED_PROXY` | الـ line virtual proxy لـ physically shared pin |

#### الـ `gpio_shared_desc->cfg` و`lflags`

**الـ** `cfg` field بيخزّن الـ `gpio_lookup_flags` اللي بتتضمّن:

| Flag | المعنى |
|------|--------|
| `GPIO_ACTIVE_HIGH` | منطق طبيعي |
| `GPIO_ACTIVE_LOW` | منطق معكوس |
| `GPIO_OPEN_DRAIN` | open-drain |
| `GPIO_OPEN_SOURCE` | open-source |
| `GPIO_PULL_UP` / `GPIO_PULL_DOWN` | bias |
| `GPIO_TRANSITORY` | transitory |

---

### الـ Structs المهمة

#### 1. `struct gpio_shared_desc`

**الغرض:** الـ state الخاص بـ GPIO line مشترَك (shared) بين أكثر من consumer — بيتعامل مع الـ reference counting والـ locking والـ configuration.

```c
struct gpio_shared_desc {
    struct gpio_desc *desc;       /* المؤشر للـ gpio_desc الأصلي */
    bool can_sleep;               /* هل الـ chip يحتاج sleeping context؟ */
    unsigned long cfg;            /* gpio_lookup_flags — electrical config */
    unsigned int usecnt;          /* عدد الـ consumers الحاليين */
    unsigned int highcnt;         /* عدد من طلبوا high value */
    union {
        struct mutex mutex;       /* لو can_sleep == true */
        spinlock_t spinlock;      /* لو can_sleep == false */
    };
};
```

**الـ key fields:**

| Field | النوع | الدور |
|-------|-------|-------|
| `desc` | `struct gpio_desc *` | الـ GPIO line الفعلي المشترَك |
| `can_sleep` | `bool` | يحدد نوع الـ lock المستخدَم |
| `cfg` | `unsigned long` | الـ electrical flags (active-low, open-drain...) |
| `usecnt` | `unsigned int` | reference counter — عدد الـ consumers |
| `highcnt` | `unsigned int` | كام consumer طالب الـ output يبقى high |
| `mutex` / `spinlock` | union | الـ lock — mutex لو sleepable، spinlock لو atomic |

**الارتباط بالـ structs الأخرى:** الـ `gpio_shared_desc` بيحتوي pointer لـ `gpio_desc`، والـ `gpio_desc` بيحتوي pointer لـ `gpio_device`.

---

#### 2. `struct gpio_desc`

**الغرض:** الـ descriptor الخاص بـ GPIO line واحد — ده اللي بيتبادله الـ kernel بدل الـ integer القديم.

```c
struct gpio_desc {
    struct gpio_device *gdev;             /* الـ chip اللي بيملك الـ line */
    unsigned long flags;                  /* bitmask of GPIOD_FLAG_* */
    struct gpio_desc_label __rcu *label;  /* اسم الـ consumer (RCU-protected) */
    const char *name;                     /* اسم الـ line من الـ firmware */
    /* CONFIG_OF_DYNAMIC: */
    struct device_node *hog;              /* node بيعمل hog للـ line */
    /* CONFIG_GPIO_CDEV: */
    unsigned int debounce_period_us;      /* debounce بالـ microseconds */
};
```

**الارتباط:** `gpio_desc` ← `gpio_device` ← `gpio_chip`

---

#### 3. `struct gpio_device`

**الغرض:** الـ internal state container لـ GPIO controller — بيعيش حتى بعد ما الـ chip يتشال لو userspace لسه بيستخدمه.

```c
struct gpio_device {
    struct device dev;                          /* Linux device model */
    struct cdev chrdev;                         /* character device */
    int id;                                     /* رقم الـ chip */
    struct module *owner;
    struct gpio_chip __rcu *chip;               /* RCU-protected pointer للـ chip */
    struct gpio_desc *descs;                    /* array of ngpio descriptors */
    unsigned long *valid_mask;                  /* bitmask للـ lines الصالحة */
    struct srcu_struct desc_srcu;               /* SRCU للـ descriptors */
    unsigned int base;                          /* deprecated global GPIO base */
    u16 ngpio;                                  /* عدد الـ lines */
    bool can_sleep;                             /* inherited من الـ chip */
    const char *label;
    void *data;                                 /* driver-private data */
    struct list_head list;                      /* قائمة كل الـ gpio_devices */
    struct raw_notifier_head line_state_notifier;
    rwlock_t line_state_lock;
    struct workqueue_struct *line_state_wq;
    struct blocking_notifier_head device_notifier;
    struct srcu_struct srcu;                    /* SRCU يحمي الـ chip pointer */
    /* CONFIG_PINCTRL: */
    struct list_head pin_ranges;
};
```

---

#### 4. `struct gpio_chip`

**الغرض:** الـ abstraction للـ GPIO controller من ناحية الـ driver — بيحتوي الـ ops callbacks.

```c
struct gpio_chip {
    const char *label;
    struct gpio_device *gpiodev;          /* الـ gpio_device المقابل */
    struct device *parent;
    struct fwnode_handle *fwnode;
    struct module *owner;

    /* ops callbacks */
    int (*request)(struct gpio_chip *gc, unsigned int offset);
    void (*free)(struct gpio_chip *gc, unsigned int offset);
    int (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int (*get)(struct gpio_chip *gc, unsigned int offset);
    int (*set)(struct gpio_chip *gc, unsigned int offset, int value);
    int (*set_config)(struct gpio_chip *gc, unsigned int offset, unsigned long config);
    int (*to_irq)(struct gpio_chip *gc, unsigned int offset);

    int base;
    u16 ngpio;
    bool can_sleep;

    /* CONFIG_GPIOLIB_IRQCHIP: */
    struct gpio_irq_chip irq;
};
```

---

#### 5. `struct gpio_chip_guard`

**الغرض:** RAII guard للـ SRCU read lock على الـ `gpio_chip` pointer — يضمن إن الـ chip موجود طول فترة الاستخدام.

```c
struct gpio_chip_guard {
    struct gpio_device *gdev;
    struct gpio_chip *gc;    /* الـ chip المقفول بـ SRCU */
    int idx;                 /* SRCU read lock index */
};
```

**الاستخدام:**
```c
/* DEFINE_CLASS macro يخلّي استخدامه بالشكل ده: */
CLASS(gpio_chip_guard, guard)(desc);
if (!guard.gc) return -ENODEV;
/* هنا الـ chip مضمون موجود */
/* عند خروج الـ scope، SRCU unlock تلقائي */
```

---

#### 6. `struct gpio_irq_chip`

**الغرض:** يدمج الـ interrupt controller جوه الـ GPIO chip — embedded داخل `struct gpio_chip`.

**الـ key fields:**

| Field | النوع | الدور |
|-------|-------|-------|
| `chip` | `struct irq_chip *` | الـ IRQ chip implementation |
| `domain` | `struct irq_domain *` | translation: hwirq → Linux IRQ |
| `parent_domain` | `struct irq_domain *` | للـ hierarchical IRQ |
| `handler` | `irq_flow_handler_t` | الـ IRQ flow handler |
| `threaded` | `bool` | هل الـ IRQ handling في thread منفصل؟ |
| `initialized` | `bool` | flag للتأكد من الـ init |
| `valid_mask` | `unsigned long *` | الـ lines اللي تقدر تكون IRQs |

---

### مخطط علاقات الـ Structs

```
                    ┌─────────────────────────────────────┐
                    │         gpio_shared_desc             │
                    │  desc ──────────────────────────┐   │
                    │  can_sleep                       │   │
                    │  cfg (lookup_flags)              │   │
                    │  usecnt / highcnt                │   │
                    │  union { mutex | spinlock }      │   │
                    └─────────────────────────────────────┘
                                                       │
                                                       ▼
┌───────────────────────────────────────────────────────────────────┐
│                          gpio_desc                                │
│  gdev ──────────────────────────────────────────────────────┐    │
│  flags (GPIOD_FLAG_SHARED | GPIOD_FLAG_SHARED_PROXY | ...)  │    │
│  label (RCU)                                                 │    │
│  name                                                        │    │
└───────────────────────────────────────────────────────────────────┘
                                                              │
                                                              ▼
┌───────────────────────────────────────────────────────────────────┐
│                         gpio_device                               │
│  dev (struct device)                                              │
│  chrdev (cdev)                                                    │
│  chip ──(RCU)──────────────────────────────────────────────┐     │
│  descs[] ──► [gpio_desc, gpio_desc, ..., gpio_desc]         │     │
│  valid_mask                                                 │     │
│  desc_srcu / srcu                                           │     │
│  line_state_notifier + line_state_lock                      │     │
│  pin_ranges (CONFIG_PINCTRL)                                │     │
└───────────────────────────────────────────────────────────────────┘
                                                             │
                                                             ▼
┌───────────────────────────────────────────────────────────────────┐
│                          gpio_chip                                │
│  gpiodev ──────────────────────────────────────────────────┘     │
│  parent (struct device *)                                         │
│  label / ngpio / base / can_sleep                                 │
│  ops: get, set, direction_input, direction_output, ...            │
│  irq (struct gpio_irq_chip) ──────────────────────────────┐      │
└───────────────────────────────────────────────────────────────────┘
                                                            │
                                                            ▼
                    ┌───────────────────────────────────────┐
                    │         gpio_irq_chip                 │
                    │  chip (irq_chip *)                    │
                    │  domain (irq_domain *)                │
                    │  parent_domain                        │
                    │  handler / threaded                   │
                    └───────────────────────────────────────┘
```

**الـ** `gpio_chip_guard` هو RAII wrapper مؤقّت — بيمسك SRCU read lock على الـ `gpio_device->chip` pointer طول عمره:

```
gpio_desc
  └─► gpio_device
        └─►(srcu RCU)── gpio_chip          ◄── gpio_chip_guard (idx, gc)
                              └─► gpio_irq_chip
```

---

### مخطط دورة حياة الـ gpio_shared_desc

```
devm_gpiod_shared_get(dev)
        │
        ▼
  alloc gpio_shared_desc
  (devm — ربط بعمر الـ device)
        │
        ▼
  gpio_device_setup_shared(gdev)
  ┌─────────────────────────────┐
  │  init usecnt = 0            │
  │  init highcnt = 0           │
  │  set can_sleep from gdev    │
  │  init mutex OR spinlock     │
  └─────────────────────────────┘
        │
        ▼
  ─── consumer يطلب الـ line ───
  lock(gpio_shared_desc)
  usecnt++
  configure desc flags (cfg)
  if first consumer → set direction
  unlock(gpio_shared_desc)
        │
        ▼
  ─── consumer يستخدم الـ line ───
  lock → read/write → unlock
        │
        ▼
  ─── consumer يحرّر الـ line ───
  lock(gpio_shared_desc)
  usecnt--
  if usecnt == 0 → release desc
  unlock(gpio_shared_desc)
        │
        ▼
  gpio_device_teardown_shared(gdev)
  destroy mutex/spinlock
  (devm auto-free)
```

---

### مخطط دورة حياة الـ gpio_chip / gpio_device

```
Driver Probe
     │
     ▼
 fill struct gpio_chip
 (label, ngpio, ops, can_sleep...)
     │
     ▼
 gpiochip_add_data(gc, data)
     │
     ├─► alloc gpio_device
     │      init dev, cdev, srcu, desc_srcu
     │      alloc descs[]
     │
     ├─► (CONFIG_PINCTRL) add_pin_ranges
     │
     ├─► (CONFIG_GPIOLIB_IRQCHIP) setup irqdomain
     │
     ├─► register character device
     │
     └─► device_add() → visible in sysfs
              │
              ▼
         [Runtime: get/set/direction ops عبر gpio_desc]
              │
              ▼
         Driver Remove
              │
              ▼
         gpiochip_remove(gc)
              │
              ├─► unregister cdev + sysfs
              ├─► free irqdomain
              ├─► RCU: gpio_device->chip = NULL
              └─► gpio_device_put()
                      │
                      ▼
                  if no more users → kfree gpio_device
                  else stays alive (userspace holds ref)
```

---

### مخطط Call Flow — طلب shared GPIO

```
consumer driver
  └─► devm_gpiod_shared_get(dev)
        │
        └─► gpio_shared_add_proxy_lookup(consumer, con_id, lflags)
              │   [يضيف lookup entry للـ proxy line]
              │
              └─► gpiod_get(consumer, con_id, flags)   [gpiolib core]
                    │
                    └─► gpiod_find_and_request(...)
                          │
                          ├─► firmware lookup (DT/ACPI/lookup_table)
                          │     └─► returns gpio_desc
                          │
                          └─► gpiod_request(desc, label)
                                │
                                ├─► gpiod_request_commit(desc, label)
                                │     ├─► CLASS(gpio_chip_guard, guard)(desc)
                                │     │     └─► srcu_read_lock(gdev->srcu)
                                │     │         gc = srcu_dereference(gdev->chip)
                                │     │
                                │     ├─► gc->request(gc, offset)  [optional]
                                │     └─► set GPIOD_FLAG_REQUESTED
                                │
                                └─► gpiod_configure_flags(desc, ...)
                                      └─► gpio_do_set_config(desc, config)
                                            └─► gc->set_config(gc, offset, cfg)
```

---

### مخطط Call Flow — set/get على shared GPIO

```
consumer
  └─► gpiod_set_value(desc, value)
        │
        ▼
    [gpiolib checks GPIOD_FLAG_SHARED]
        │
        └─► lock(gpio_shared_desc)   ← mutex أو spin_lock_irqsave حسب can_sleep
              │
              ├─► if output:
              │     highcnt += (value ? 1 : 0)
              │     actual_value = (highcnt > 0) ? 1 : 0
              │     CLASS(gpio_chip_guard, guard)(desc)
              │       └─► srcu_read_lock
              │           gc->set(gc, offset, actual_value)
              │           srcu_read_unlock (auto via guard destructor)
              │
              └─► unlock(gpio_shared_desc)
```

---

### مخطط Call Flow — teardown

```
device remove أو driver unbind
     │
     ▼
gpio_device_teardown_shared(gdev)
     │
     ├─► lock(gpio_shared_desc)
     ├─► if usecnt > 0: WARN — consumers لسه موجودين!
     ├─► unlock + destroy lock
     └─► (devm free gpio_shared_desc)
              │
              ▼
         gpiochip_remove(gc)
              │
              ├─► synchronize_srcu(&gdev->srcu)
              │    [ينتظر أي قارئ active ينتهي]
              │
              ├─► RCU assign: gdev->chip = NULL
              │
              └─► gpio_device_put(gdev)
                   [kref decrement — لو وصل صفر: kfree]
```

---

### استراتيجية الـ Locking

#### جدول شامل

| البيانات المحمية | الـ Lock | نوع الـ Lock | ملاحظات |
|-----------------|---------|-------------|---------|
| `gpio_desc->flags` | `gpio_desc` lock (داخلي) | spinlock | set/clear bits |
| `gpio_device->chip` pointer | `gpio_device->srcu` | SRCU | يحمي RCU dereference |
| `gpio_desc->label` | `gpio_device->desc_srcu` | SRCU | RCU pointer للـ label string |
| `gpio_shared_desc` fields | `gpio_shared_desc->mutex` أو `spinlock` | adaptive | حسب `can_sleep` |
| `gpio_device->line_state_notifier` | `gpio_device->line_state_lock` | rwlock_t | readers كتير، writer نادر |
| قائمة كل الـ gpio_devices | `gpio_devices_lock` (global) | mutex | في gpiolib.c |
| `gpio_desc->hog` | none (init-time only) | — | بيتكتب مرة واحدة |

#### الـ Lock Ordering (ترتيب الأقفال)

```
1. gpio_devices_lock  (global list lock)
       │
       ▼
2. gpio_device->srcu  (SRCU — chip pointer protection)
       │
       ▼
3. gpio_shared_desc lock  (mutex أو spinlock)
       │
       ▼
4. gpio_desc lock  (per-descriptor spinlock)
       │
       ▼
5. hardware register access
```

> **قاعدة أساسية:** الـ SRCU lock لازم يتمسك قبل أي access للـ `gpio_chip` ops — لأن الـ chip ممكن يتشال في أي وقت. الـ `gpio_chip_guard` بيعمل ده تلقائي.

#### الـ `gpio_shared_desc` Lock — Adaptive Pattern

الـ lock الـ adaptive ده من أجمل الـ patterns في الـ kernel:

```c
/* التعريف في gpiolib-shared.h */
DEFINE_LOCK_GUARD_1(gpio_shared_desc_lock, struct gpio_shared_desc,
    /* acquire */
    if (_T->lock->can_sleep)
        mutex_lock(&_T->lock->mutex);
    else
        spin_lock_irqsave(&_T->lock->spinlock, _T->flags),
    /* release */
    if (_T->lock->can_sleep)
        mutex_unlock(&_T->lock->mutex);
    else
        spin_unlock_irqrestore(&_T->lock->spinlock, _T->flags),
    unsigned long flags)  /* extra state للـ spinlock flags */
```

**الاستخدام:**
```c
/* عند دخول الـ scope: lock تلقائي */
scoped_guard(gpio_shared_desc_lock, shared_desc) {
    shared_desc->usecnt++;
    /* ... */
}
/* عند الخروج: unlock تلقائي */
```

#### الـ `lockdep` Assertions

```c
static inline void gpio_shared_lockdep_assert(struct gpio_shared_desc *shared_desc)
{
    if (shared_desc->can_sleep)
        lockdep_assert_held(&shared_desc->mutex);
    else
        lockdep_assert_held(&shared_desc->spinlock);
}
```

**الـ** `lockdep_assert_held` بيكون no-op في الـ production لو `CONFIG_LOCKDEP` معطَّل، لكن في الـ debug builds بيـtrigger WARNING لو الـ lock مش ممسوك — أداة قوية لاكتشاف الـ locking bugs وقت الـ development.

---

### ملخص العلاقات الكاملة (نظرة من الأعلى)

```
Consumer Driver
    │
    │  devm_gpiod_shared_get()
    │  gpio_shared_add_proxy_lookup()
    ▼
gpio_shared_desc ──desc──► gpio_desc ──gdev──► gpio_device
                                                    │
                                          ┌─────────┴──────────┐
                                          │                    │
                                    descs[0..ngpio-1]    chip (RCU)
                                          │                    │
                                    [gpio_desc]           gpio_chip
                                                               │
                                                         ┌─────┴──────┐
                                                         │            │
                                                      ops()     gpio_irq_chip
                                                    (get/set/     (irq_chip,
                                                    direction)    irq_domain)
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Public API (exported / header-declared)

| Function | Defined In | Export | Purpose |
|---|---|---|---|
| `gpio_device_setup_shared()` | `gpiolib-shared.c` | Internal (gpiolib) | Setup proxy devices عند register الـ GPIO chip |
| `gpio_device_teardown_shared()` | `gpiolib-shared.c` | Internal (gpiolib) | Cleanup proxy devices عند unregister الـ GPIO chip |
| `gpio_shared_add_proxy_lookup()` | `gpiolib-shared.c` | Internal (gpiolib) | يضيف machine lookup table لـ proxy consumer |
| `devm_gpiod_shared_get()` | `gpiolib-shared.c` | `EXPORT_SYMBOL_GPL` | الـ consumer بيطلب `gpio_shared_desc` بـ devm |

#### Internal (static) Functions

| Function | Purpose |
|---|---|
| `gpio_shared_init()` | `postcore_initcall` — يبدأ scan الـ firmware nodes |
| `gpio_shared_of_scan()` | Entry point لـ OF tree traversal |
| `gpio_shared_of_traverse()` | Recursive DT traversal — يبني الـ entries/refs |
| `gpio_shared_find_entry()` | يدور على `gpio_shared_entry` بـ fwnode + offset |
| `gpio_shared_make_ref()` | Allocates + initializes `gpio_shared_ref` |
| `gpio_shared_make_adev()` | Creates auxiliary device لكل ref |
| `gpio_shared_remove_adev()` | يحذف auxiliary device |
| `gpio_shared_setup_reset_proxy()` | يضيف ref خاص بـ reset-gpio |
| `gpio_shared_dev_is_reset_gpio()` | يتحقق إن الـ consumer هو reset-gpio proxy |
| `gpio_shared_entry_is_really_shared()` | يحكم إن الـ pin مشترك فعلاً |
| `gpio_shared_free_exclusive()` | يحذف entries الـ non-shared من الـ list |
| `gpiod_shared_desc_create()` | ينشئ `gpio_shared_desc` جديد |
| `gpiod_shared_put()` | devm action — يعمل `kref_put` |
| `gpio_shared_release()` | kref release callback — يحرر `gpio_shared_desc` |
| `gpio_shared_drop_ref()` | يحرر `gpio_shared_ref` كاملاً |
| `gpio_shared_drop_entry()` | يحرر `gpio_shared_entry` كاملاً |
| `gpio_shared_teardown()` | يحرر كل الـ list عند init failure |

#### Inline Helpers (header)

| Function / Macro | Purpose |
|---|---|
| `gpio_shared_lockdep_assert()` | يتحقق إن الـ lock مأخوذ — debug only |
| `DEFINE_LOCK_GUARD_1(gpio_shared_desc_lock, ...)` | Scope-based lock guard يختار mutex أو spinlock |

---

### التصنيف المنطقي للـ Functions

```
┌─────────────────────────────────────────────────────────┐
│  Initialization Layer                                    │
│  gpio_shared_init → gpio_shared_of_scan                 │
│                   → gpio_shared_of_traverse              │
│                     → gpio_shared_find_entry             │
│                     → gpio_shared_make_ref               │
│                     → gpio_shared_setup_reset_proxy      │
│  gpio_shared_free_exclusive                              │
└────────────────────┬────────────────────────────────────┘
                     │ (shared entries list built)
┌────────────────────▼────────────────────────────────────┐
│  GPIO Device Registration Layer                          │
│  gpio_device_setup_shared                                │
│    → gpio_shared_make_adev (per ref)                    │
│  gpio_device_teardown_shared                             │
│    → gpio_shared_remove_adev                            │
└────────────────────┬────────────────────────────────────┘
                     │ (auxiliary devices created)
┌────────────────────▼────────────────────────────────────┐
│  Consumer Acquisition Layer                              │
│  gpio_shared_add_proxy_lookup (called by proxy driver)  │
│  devm_gpiod_shared_get (called by proxy driver)         │
│    → gpiod_shared_desc_create                           │
│    → devm_add_action_or_reset → gpiod_shared_put        │
│                                → gpio_shared_release    │
└─────────────────────────────────────────────────────────┘
```

---

### Group 1: Initialization — بناء قائمة الـ Shared GPIOs

الـ group ده بيتشغّل مرة واحدة فقط في وقت boot عبر `postcore_initcall`. هدفه إنه يمشي على الـ Device Tree كله ويشوف أي GPIO pin بيشير إليه أكتر من consumer واحد، وبكده يبني الـ `gpio_shared_list` اللي هي القاعدة اللي باقي الكود بيشتغل عليها.

---

#### `gpio_shared_init`

```c
static int __init gpio_shared_init(void)
```

**ما بيعمله:** بيعمل kick-off للـ scan على الـ OF tree، لو فشل بيعمل teardown للـ partial state. بعد الـ scan بيشيل الـ entries اللي GPIO فيها مش مشترك فعلاً (`gpio_shared_free_exclusive`).

**Parameters:** لا يوجد.

**Return:** `0` للنجاح، error code من `gpio_shared_of_scan` للفشل.

**Key details:**
- **`postcore_initcall`** — بيتنفذ بعد `core_initcall` وقبل الـ device drivers، يعني الـ `gpio_shared_list` جاهزة قبل ما أي GPIO driver يتسجل.
- الـ `gpio_shared_list` و `gpio_shared_ida` هما global statics — مفيش locking عليهم هنا لأن الـ init single-threaded.

**Pseudocode:**
```
gpio_shared_init():
    ret = gpio_shared_of_scan()
    if ret < 0:
        gpio_shared_teardown()
        pr_err(...)
        return ret
    gpio_shared_free_exclusive()
    return 0
```

---

#### `gpio_shared_of_scan`

```c
static int gpio_shared_of_scan(void)
```

**ما بيعمله:** Entry point للـ OF traversal، بيبدأ من `of_root`. لو CONFIG_OF مش enabled بترجع 0 فاضي.

**Return:** `0` أو error من الـ traversal.

**Key details:** بيتحقق إن `of_root` مش NULL قبل ما يبدأ — لو الـ system مش OF-based بيرجع 0 بهدوء.

---

#### `gpio_shared_of_traverse`

```c
static int gpio_shared_of_traverse(struct device_node *curr)
```

**ما بيعمله:** الـ core logic للـ scan. بيمشي recursively على كل الـ device nodes في الـ DT. لكل property اسمها `*-gpios` أو `*-gpio` أو `gpios` أو `gpio`، بيعمل `of_parse_phandle_with_args` على كل element في الـ array. لو الـ target node هو `gpio-controller`، بيدور على أو ينشئ `gpio_shared_entry` ويضيف `gpio_shared_ref`.

**Parameters:**
- `curr` — الـ `device_node` الحالي اللي بيتم scan عليه.

**Return:** `0` أو `-ENOMEM`.

**Key details:**
- بيتجاهل nodes بـ `gpio_shared_of_node_ignore()`: disabled nodes، `__symbols__`، وـ gpio hogs.
- بيدعم فقط الـ **2-cell GPIO binding** (offset + flags) — الـ 1-cell و 3-cell بيتجاهلهم.
- لو الـ property هي `reset-gpios`، بيستدعي `gpio_shared_setup_reset_proxy()` لإنشاء ref إضافي.
- الـ `con_id` بيتحسب بأخذ اسم الـ property وشيل الـ suffix (`-gpios` أو `-gpio`).

**Pseudocode:**
```
gpio_shared_of_traverse(node):
    if should_ignore(node): return 0

    for each property in node:
        if not GPIO property: continue
        for i in 0..count:
            parse phandle args[i]
            if target is not gpio-controller: continue
            if args_count != 2: continue

            entry = find_or_create_entry(fwnode, offset)
            con_id = prop_name minus suffix
            ref = make_ref(node_fwnode, con_id, flags)
            add ref to entry.refs

            if prop == "reset-gpios":
                setup_reset_proxy(entry, flags)

    for each child:
        gpio_shared_of_traverse(child)
```

---

#### `gpio_shared_find_entry`

```c
static struct gpio_shared_entry *
gpio_shared_find_entry(struct fwnode_handle *controller_node,
                       unsigned int offset)
```

**ما بيعمله:** Linear search في `gpio_shared_list` بيدور على entry بـ نفس الـ `fwnode` ونفس الـ `offset`.

**Parameters:**
- `controller_node` — الـ fwnode بتاع الـ GPIO controller.
- `offset` — الـ hardware offset للـ pin داخل الـ chip.

**Return:** pointer لـ `gpio_shared_entry` أو `NULL`.

**Key details:** No locking — فقط بيتاعمل فيها من سياق single-threaded init.

---

#### `gpio_shared_make_ref`

```c
static struct gpio_shared_ref *gpio_shared_make_ref(struct fwnode_handle *fwnode,
                                                     const char *con_id,
                                                     enum gpiod_flags flags)
```

**ما بيعمله:** Allocates وينشئ `gpio_shared_ref` جديد مع كل الـ fields الخاصة بيه. بيخصص ID من `gpio_shared_ida` لاستخدامه كـ auxiliary device ID. بيعمل `mutex_init_with_key()` باستخدام `lock_class_key` خاص بكل ref عشان lockdep يميّز بين الـ mutexes.

**Parameters:**
- `fwnode` — الـ firmware node بتاع الـ consumer، ممكن يبقى `NULL` لـ reset-gpio proxy.
- `con_id` — الـ connection ID (مثلاً `"enable"`)، ممكن `NULL`.
- `flags` — الـ GPIO flags من الـ DT (`GPIOD_*`).

**Return:** pointer لـ `gpio_shared_ref` أو `NULL` عند فشل الـ allocation.

**Key details:**
- بيستخدم `no_free_ptr()` مع `__free(kfree)` للـ cleanup-on-failure pattern.
- الـ `lockdep_register_key()` ضروري لأن كل ref بيستخدم نفس النوع من الـ lock لكن بـ different nesting context.

---

#### `gpio_shared_setup_reset_proxy`

```c
static int gpio_shared_setup_reset_proxy(struct gpio_shared_entry *entry,
                                          enum gpiod_flags flags)
```

**ما بيعمله:** بينشئ ref ثانوي للـ entry خاص بـ `reset-gpio` device. ده لأن الـ `reset-gpio` auxiliary driver ممكن يتطلب GPIO اللي الـ original consumer بيحكمه بـ reset control.

**Parameters:**
- `entry` — الـ GPIO entry اللي عنده `reset-gpios`.
- `flags` — نفس الـ flags من الـ DT.

**Return:** `0` أو `-ENOMEM`.

**Key details:** لو الـ entry عنده بالفعل reset-gpio ref (`is_reset_gpio == true`)، بيرجع 0 فوراً — idempotent.

---

#### `gpio_shared_entry_is_really_shared`

```c
static bool gpio_shared_entry_is_really_shared(struct gpio_shared_entry *entry)
```

**ما بيعمله:** بيحكم إن الـ GPIO pin مشترك فعلاً بين أكتر من consumer حقيقي. الـ corner case: لو في 2 refs بس وواحد منهم `is_reset_gpio`، الـ pin مش مشترك حقيقياً — الـ reset-gpio ref مجرد secondary proxy.

**Return:** `true` لو مشترك فعلاً.

**Logic:**
```
num == 0 or 1  → false
num > 2        → true
num == 2:
    if one ref is_reset_gpio → false
    else → true
```

---

#### `gpio_shared_free_exclusive`

```c
static void gpio_shared_free_exclusive(void)
```

**ما بيعمله:** بيمشي على الـ `gpio_shared_list` ويشيل أي entry مش مشترك فعلاً (عشان دي exclusive GPIOs وصلتلها بالغلط). بيحرر الـ ref والـ entry.

**Key details:** بيستخدم `list_for_each_entry_safe` لأنه بيعمل `list_del` في نص الـ loop.

---

### Group 2: GPIO Device Registration — ربط الـ Entries بالـ Hardware

الـ group ده بيتعامل مع الـ `gpio_device` lifecycle. بيتشغل من gpiolib core لما GPIO chip بيتسجل أو بيتشال.

---

#### `gpio_device_setup_shared`

```c
int gpio_device_setup_shared(struct gpio_device *gdev)
```

**ما بيعمله:** يتشغل لما `gpio_device` جديد بيتسجل. بيمر بمرحلتين:
1. **Proxy detection:** لو الـ `gdev` ده هو بالفعل proxy (والده هو `adev` من ref موجود) → بيضرب `GPIOD_FLAG_SHARED_PROXY` على `descs[0]` ويرجع.
2. **Controller setup:** لو الـ `gdev` هو الـ GPIO controller الأصلي → لكل pin مشترك بيعمل `gpiod_request_commit()` (يعني بيقفل الـ pin عشان حد تاني ميقدرش ياخده مباشرة) وبينشئ auxiliary device (`adev`) لكل ref.

**Parameters:**
- `gdev` — الـ GPIO device الجديد.

**Return:** `0` أو error code.

**Key details:**
- الـ `gpiod_request_commit(desc, "shared")` بيعمل internal request — الـ pin يبقى owned بالـ shared subsystem.
- لو `list_count_nodes(&entry->refs) <= 1`، الـ pin مش مشترك فعلاً وبيتجاهله.
- لو `gpio_shared_make_adev()` فشل، بيعمل `gpiod_free_commit()` ويرجع error.

**Who calls it:** `gpiolib.c` من `gpiochip_add_data_with_key()` أو ما يقابلها.

**Pseudocode:**
```
gpio_device_setup_shared(gdev):
    // Phase 1: am I a proxy?
    for each entry in shared_list:
        for each ref in entry.refs:
            if gdev.dev.parent == &ref.adev.dev:
                set GPIOD_FLAG_SHARED_PROXY on descs[0]
                return 0

    // Phase 2: am I the controller exposing shared pins?
    for each entry matching gdev.fwnode:
        if refs count <= 1: continue
        desc = &gdev.descs[entry.offset]
        set GPIOD_FLAG_SHARED on desc
        gpiod_request_commit(desc, "shared")
        for each ref:
            gpio_shared_make_adev(gdev, entry, ref)
    return 0
```

---

#### `gpio_device_teardown_shared`

```c
void gpio_device_teardown_shared(struct gpio_device *gdev)
```

**ما بيعمله:** Cleanup mirror لـ `gpio_device_setup_shared`. لكل entry مرتبطة بالـ `gdev`:
1. بيحرر الـ pin بـ `gpiod_free_commit()`.
2. لكل ref: بيشيل الـ lookup table لو موجودة (`gpiod_remove_lookup_table`) وبيحذف الـ auxiliary device.

**Parameters:**
- `gdev` — الـ GPIO device اللي بيتشال.

**Key details:**
- الـ mutex guard على `ref->lock` ضروري عشان الـ `ref->lookup` ممكن يكون بيتعدل concurrently من consumer.
- `kfree(ref->lookup->table[0].key)` مهمة — الـ key بيتعمله `kasprintf` في `gpio_shared_add_proxy_lookup`.

**Who calls it:** `gpiolib.c` من cleanup path.

---

#### `gpio_shared_make_adev`

```c
static int gpio_shared_make_adev(struct gpio_device *gdev,
                                  struct gpio_shared_entry *entry,
                                  struct gpio_shared_ref *ref)
```

**ما بيعمله:** بينشئ `auxiliary_device` للـ ref ده. الـ auxiliary device هو الـ "proxy" اللي من خلاله الـ consumer هيتعرف على الـ GPIO المشترك.

**Parameters:**
- `gdev` — الـ GPIO controller (والده للـ adev).
- `entry` — الـ entry اللي هتتخزن في `platform_data`.
- `ref` — الـ ref اللي عنده `adev` struct.

**Return:** `0` أو error.

**Key details:**
- `adev->name = "proxy"` — الـ auxiliary bus ID بيبقى `<module_name>.proxy.<id>`.
- `adev->dev.platform_data = entry` — الـ proxy driver بيجيب الـ entry بـ `dev_get_platdata()`.
- `adev->dev.parent = gdev->dev.parent` — الـ parent هو الـ GPIO chip's parent, مش الـ GPIO device نفسه.
- محمي بـ `guard(mutex)(&ref->lock)`.

---

#### `gpio_shared_remove_adev`

```c
static void gpio_shared_remove_adev(struct auxiliary_device *adev)
```

**ما بيعمله:** Simple wrapper — بيعمل `auxiliary_device_delete` ثم `auxiliary_device_uninit`. الـ two-step مهم: `delete` بيشيل الـ device من الـ bus (بيتريجر probe/remove)، و`uninit` بيحرر الـ resources.

---

### Group 3: Consumer Acquisition — الـ Proxy Driver بياخد الـ GPIO

الـ group ده هو الـ runtime API اللي الـ proxy driver بيستخدمه عشان يحصل على `gpio_shared_desc` ويقدر يشغّل الـ GPIO.

---

#### `gpio_shared_add_proxy_lookup`

```c
int gpio_shared_add_proxy_lookup(struct device *consumer, const char *con_id,
                                  unsigned long lflags)
```

**ما بيعمله:** لما الـ proxy driver بيقوم، بيستدعي الـ function دي عشان يضيف **machine lookup table** تربط اسمه بالـ GPIO pin الصح. ده ضروري عشان `gpiod_get()` يقدر يلاقي الـ GPIO من خلال الـ proxy device name بدل firmware node مباشرة.

**Parameters:**
- `consumer` — الـ proxy device اللي بيطلب الـ lookup.
- `con_id` — الـ connection ID المطلوب.
- `lflags` — الـ lookup flags (مثلاً `GPIO_ACTIVE_LOW`).

**Return:** `0` أو `-ENOMEM` أو `-ENOENT`.

**Key details:**
- بيمشي على `gpio_shared_list` ويدور على الـ ref اللي `ref->fwnode` بيطابق الـ `consumer`.
- لو الـ consumer هو `reset-gpio` (compatible)، بيستدعي `gpio_shared_dev_is_reset_gpio()` للـ special case.
- لو `ref->lookup` موجود بالفعل (previous call)، بيرجع 0 فوراً — idempotent.
- الـ lookup key هي `"<module_name>.proxy.<id>"` — نفس اسم الـ auxiliary device.
- بيعمل `gpiod_add_lookup_table()` تلقائياً.
- الـ `WARN_ON(1)` في النهاية هو programmer bug indicator — يعني مفيش ref طابق.

**Pseudocode:**
```
gpio_shared_add_proxy_lookup(consumer, con_id, lflags):
    for each entry in shared_list:
        for each ref in entry.refs:
            lock(ref)
            if consumer doesn't match ref: continue
            if con_id mismatch: continue
            if ref.lookup already set: return 0

            key = kasprintf("module.proxy.%u", ref.adev.id)
            lookup = alloc(2 entries)
            lookup.dev_id = dev_name(consumer)
            lookup.table[0] = GPIO_LOOKUP(key, 0, con_id, lflags)
            ref.lookup = lookup
            gpiod_add_lookup_table(lookup)
            return 0

    WARN_ON(1)
    return -ENOENT
```

---

#### `devm_gpiod_shared_get`

```c
struct gpio_shared_desc *devm_gpiod_shared_get(struct device *dev)
```

**ما بيعمله:** الـ **main API** اللي proxy driver بيستخدمه. بيجيب `gpio_shared_desc` للـ GPIO المشترك مع reference counting كامل. لو الـ `shared_desc` موجود بالفعل (consumer تاني سبق وطلبه)، بيعمل `kref_get` فقط. لو لأ، بيعمل `gpiod_shared_desc_create()` وينشئ `kref_init`.

بيربط `gpiod_shared_put()` كـ devm action عشان يتعمل cleanup لما الـ device يتحرر.

**Parameters:**
- `dev` — الـ proxy device (اللي `dev_get_platdata(dev)` بيرجع الـ `gpio_shared_entry`).

**Return:** pointer لـ `gpio_shared_desc` أو `ERR_PTR` (`-ENOENT`, `-EPROBE_DEFER`, `-ENOMEM`).

**Key details:**
- `dev_get_platdata(dev)` بيرجع `gpio_shared_entry` — اتحطت في `adev->dev.platform_data` في `gpio_shared_make_adev`.
- الـ `scoped_guard(mutex, &entry->lock)` بيحمي creation/get الـ `shared_desc`.
- `EXPORT_SYMBOL_GPL` — متاح للـ modules.
- `-EPROBE_DEFER` من `gpiod_shared_desc_create` لو الـ GPIO chip لسه ماتسجلش.

**Who calls it:** GPIO shared proxy driver (مثلاً reset-gpio driver أو أي driver يعمل auxiliary driver للـ shared GPIO).

**Pseudocode:**
```
devm_gpiod_shared_get(dev):
    entry = dev_get_platdata(dev)
    if !entry: return ERR_PTR(-ENOENT)

    lock(entry)
    if entry.shared_desc:
        kref_get(&entry.ref)
        shared_desc = entry.shared_desc
    else:
        shared_desc = gpiod_shared_desc_create(entry)
        if IS_ERR: return ERR_CAST
        kref_init(&entry.ref)
        entry.shared_desc = shared_desc
    unlock(entry)

    devm_add_action_or_reset(dev, gpiod_shared_put, entry)
    return shared_desc
```

---

#### `gpiod_shared_desc_create`

```c
static struct gpio_shared_desc *
gpiod_shared_desc_create(struct gpio_shared_entry *entry)
```

**ما بيعمله:** بينشئ `gpio_shared_desc` جديد ويملاه. بيستخدم `gpio_device_find_by_fwnode()` للوصول للـ `gpio_device`، وبيجيب الـ `gpio_desc` مباشرة من `gdev->descs[offset]`. بيشوف `gpiod_cansleep()` لتحديد نوع الـ lock (mutex أو spinlock).

**Parameters:**
- `entry` — الـ entry اللي عندها الـ fwnode والـ offset.

**Return:** `gpio_shared_desc*` أو `ERR_PTR(-ENOMEM)` أو `ERR_PTR(-EPROBE_DEFER)`.

**Key details:**
- `lockdep_assert_held(&entry->lock)` — لازم يتشغّل وهو ماسك `entry->lock`.
- الـ `gdev` reference محتفظ بيه في `gpio_device_put()` في `gpio_shared_release()`.
- الـ lock type بيتحدد runtime بناءً على `can_sleep`:
  - `can_sleep == true` → `mutex_init` (الـ GPIO controller needs to sleep، مثلاً I2C GPIO expander)
  - `can_sleep == false` → `spin_lock_init` (fast path، local GPIO)

---

### Group 4: Reference Counting & Cleanup — إدارة الـ Lifetime

---

#### `gpiod_shared_put` (devm action)

```c
static void gpiod_shared_put(void *data)
```

**ما بيعمله:** الـ devm action المسجّل بـ `devm_add_action_or_reset`. بيعمل `kref_put` على الـ `gpio_shared_entry`. لو الـ refcount وصل 0، بيستدعي `gpio_shared_release`.

**Parameters:**
- `data` — `gpio_shared_entry *` (cast من `void *`).

**Who calls it:** devm cleanup path لما الـ proxy device يتحرر.

---

#### `gpio_shared_release`

```c
static void gpio_shared_release(struct kref *kref)
```

**ما بيعمله:** الـ kref release callback. بيحرر الـ `gpio_shared_desc` وكل resources بيزمّها:
1. يعمل `gpio_device_put()` على الـ gdev اللي اتحفظ في الـ desc.
2. لو `can_sleep` → `mutex_destroy`.
3. `kfree(shared_desc)`.
4. `entry->shared_desc = NULL`.

**Key details:**
- محمي بـ `guard(mutex)(&entry->lock)`.
- الـ `gpio_device_put()` ضروري لأن `gpiod_shared_desc_create` عمل implicit get بـ `gpio_device_find_by_fwnode`.

---

#### `gpio_shared_drop_ref`

```c
static void gpio_shared_drop_ref(struct gpio_shared_ref *ref)
```

**ما بيعمله:** يحرر `gpio_shared_ref` بالكامل:
- `list_del` من `entry->refs`.
- `mutex_destroy` + `lockdep_unregister_key`.
- `kfree(ref->con_id)`.
- `ida_free` للـ dev_id.
- `fwnode_handle_put` للـ fwnode.
- `kfree(ref)`.

**Key details:** فقط بيتشغّل من init failure path أو من `gpio_shared_free_exclusive`. الـ normal teardown path بيستخدم `gpio_device_teardown_shared`.

---

#### `gpio_shared_drop_entry`

```c
static void gpio_shared_drop_entry(struct gpio_shared_entry *entry)
```

**ما بيعمله:** يحرر `gpio_shared_entry` بالكامل بعد ما كل refs اتحررت.

---

#### `gpio_shared_teardown`

```c
static void __init gpio_shared_teardown(void)
```

**ما بيعمله:** يحرر كل الـ `gpio_shared_list` بالكامل — refs وentries. بيتشغّل فقط لو `gpio_shared_init()` فشلت. الـ `__init` annotation صح هنا رغم إن الاسم "teardown" — لأنه فقط خاص بالـ init failure cleanup وهينزل مع الـ init code من الـ memory بعد boot.

---

### Group 5: Locking Helpers — الـ Lock Infrastructure

---

#### `DEFINE_LOCK_GUARD_1(gpio_shared_desc_lock, ...)`

```c
DEFINE_LOCK_GUARD_1(gpio_shared_desc_lock, struct gpio_shared_desc,
    if (_T->lock->can_sleep)
        mutex_lock(&_T->lock->mutex);
    else
        spin_lock_irqsave(&_T->lock->spinlock, _T->flags),
    if (_T->lock->can_sleep)
        mutex_unlock(&_T->lock->mutex);
    else
        spin_unlock_irqrestore(&_T->lock->spinlock, _T->flags),
    unsigned long flags)
```

**ما بيعمله:** بيعرّف lock guard class اسمه `gpio_shared_desc_lock` يصلح للاستخدام مع `guard()` أو `scoped_guard()`. الـ guard بيختار تلقائياً mutex أو spinlock حسب قيمة `can_sleep` في الـ `gpio_shared_desc`.

**Key details:**
- الـ `unsigned long flags` في آخره هو extra field بيتضاف للـ guard struct — ده متطلب `spin_lock_irqsave` عشان يحفظ الـ IRQ state.
- الـ `_T` في الـ macro بيشير لـ pointer للـ guard struct، و`_T->lock` بيشير للـ `gpio_shared_desc`.
- مثال استخدام:

```c
scoped_guard(gpio_shared_desc_lock, shared_desc) {
    /* critical section — mutex or spinlock held */
}
```

---

#### `gpio_shared_lockdep_assert`

```c
static inline void gpio_shared_lockdep_assert(struct gpio_shared_desc *shared_desc)
```

**ما بيعمله:** Debug assertion تتحقق إن الـ lock الصح ماسك. بيستخدم `lockdep_assert_held()` على الـ mutex أو الـ spinlock حسب `can_sleep`.

**Parameters:**
- `shared_desc` — الـ shared descriptor المراد التحقق من lock بتاعه.

**Key details:**
- No-op في production (`CONFIG_LOCKDEP` مش enabled).
- بيتشغّل في داخل الـ critical sections عشان تثبّت إن الـ caller ماسك الـ lock.

---

#### `gpio_shared_dev_is_reset_gpio`

```c
static bool gpio_shared_dev_is_reset_gpio(struct device *consumer,
                                           struct gpio_shared_entry *entry,
                                           struct gpio_shared_ref *ref)
```

**ما بيعمله:** Special case handler لـ `reset-gpio` auxiliary devices. بيتحقق إن الـ `consumer` هو فعلاً الـ reset-gpio proxy المقابل لـ entry معينة عن طريق مقارنة الـ `reset-gpios` property arguments.

**Parameters:**
- `consumer` — الـ device اللي بيطلب الـ lookup.
- `entry` — الـ GPIO entry المعنية.
- `ref` — الـ secondary ref اللي `is_reset_gpio == true`.

**Return:** `true` لو confirmed reset-gpio match.

**Key details:**
- `lockdep_assert_held(&ref->lock)` — بيتشغّل وهو ماسك `ref->lock`.
- بيعمل nested `guard(mutex)(&real_ref->lock)` على كل real_ref — يعني nested locking. الـ lockdep lock keys المختلفة بتمنع false positive.
- لو matched، بيعمل `fwnode_handle_get(reset_fwnode)` ويحفظه في `ref->fwnode` عشان الـ calls الجاية تمشي في normal path.
- فقط compiled لو `CONFIG_RESET_GPIO`.

---

### الـ Data Structures المستخدمة

```
gpio_shared_list (global list)
    └── gpio_shared_entry (per unique GPIO pin)
            ├── fwnode  → GPIO controller fwnode
            ├── offset  → hardware pin number
            ├── lock    → protects shared_desc
            ├── shared_desc → gpio_shared_desc (created on first get)
            │       ├── desc    → gpio_desc* (actual pin)
            │       ├── can_sleep
            │       ├── cfg / usecnt / highcnt
            │       └── mutex | spinlock
            ├── ref     → kref (counts active consumers)
            └── refs    (list of gpio_shared_ref)
                    └── gpio_shared_ref (per consumer reference)
                            ├── fwnode  → consumer fwnode
                            ├── con_id
                            ├── flags
                            ├── dev_id  → IDA ID for adev
                            ├── lock    → per-ref mutex
                            ├── adev    → auxiliary_device
                            ├── lookup  → gpiod_lookup_table*
                            └── is_reset_gpio
```

### مخطط معمارية الـ Subsystem

```
Boot time (postcore_initcall):
┌─────────────────────────────────────────────┐
│           gpio_shared_init()                │
│  1. gpio_shared_of_scan()                   │
│     └─ gpio_shared_of_traverse(of_root)     │
│         ├─ for each node: scan *-gpios props│
│         ├─ gpio_shared_find_entry() or alloc│
│         └─ gpio_shared_make_ref() per ref   │
│  2. gpio_shared_free_exclusive()            │
│     └─ remove non-shared entries           │
└─────────────────────────────────────────────┘

GPIO Controller probe:
┌─────────────────────────────────────────────┐
│    gpio_device_setup_shared(gdev)           │
│  ├─ if proxy: mark GPIOD_FLAG_SHARED_PROXY  │
│  └─ if real controller:                     │
│      ├─ mark GPIOD_FLAG_SHARED on desc      │
│      ├─ gpiod_request_commit() (lock pin)   │
│      └─ gpio_shared_make_adev() per ref     │
│          └─ creates "gpiolib.proxy.N" adev  │
└─────────────────────────────────────────────┘

Consumer probe (proxy driver):
┌─────────────────────────────────────────────┐
│  gpio_shared_add_proxy_lookup(consumer, ...) │
│  └─ builds gpiod_lookup_table               │
│      └─ gpiod_add_lookup_table()            │
│                                             │
│  devm_gpiod_shared_get(proxy_dev)           │
│  └─ returns gpio_shared_desc with:          │
│      ├─ desc → real gpio_desc               │
│      ├─ can_sleep flag                      │
│      └─ mutex or spinlock for arbitration   │
└─────────────────────────────────────────────┘

GPIO Controller remove:
┌─────────────────────────────────────────────┐
│   gpio_device_teardown_shared(gdev)         │
│   ├─ gpiod_free_commit() per shared pin     │
│   ├─ gpiod_remove_lookup_table() per ref    │
│   └─ gpio_shared_remove_adev() per ref      │
└─────────────────────────────────────────────┘
```

### الـ `gpio_shared_desc` — قلب الـ Runtime

```c
struct gpio_shared_desc {
    struct gpio_desc *desc;    /* الـ descriptor الحقيقي في الـ GPIO chip */
    bool can_sleep;            /* هل الـ GPIO في controller بيحتاج sleep? */
    unsigned long cfg;         /* config cache للـ consumers */
    unsigned int usecnt;       /* عدد الـ active users */
    unsigned int highcnt;      /* عدد اللي طالبين output HIGH */
    union {
        struct mutex mutex;    /* لو can_sleep == true */
        spinlock_t spinlock;   /* لو can_sleep == false */
    };
};
```

الـ `usecnt` و`highcnt` بيتستخدموا في الـ drivers اللي بتستخدم الـ `gpio_shared_desc` مباشرة (زي `reset-gpio`) لتطبيق **shared GPIO arbitration** — الـ pin بيعلى لما واحد على الأقل طالبه high، وبيرجع low لما الـ usecnt يوصل 0.
## Phase 5: دليل الـ Debugging الشامل

الـ `gpiolib-shared.h` بيعرّف الـ infrastructure الخاصة بـ **GPIO Shared** — وهي آلية بتخلي أكتر من consumer يشاركوا نفس الـ GPIO pin. الـ debugging هنا بيشمل مستويين: الـ locking mechanism (mutex/spinlock) والـ shared state (`usecnt`, `highcnt`, `cfg`).

---

### Software Level

#### 1. debugfs Entries

الـ GPIO subsystem بيعمل expose كامل في `debugfs`:

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# اعرض كل الـ GPIO chips والـ lines
cat /sys/kernel/debug/gpio

# اعرض gpiochip معين بالتفصيل
cat /sys/kernel/debug/gpio | grep -A 50 "gpiochip0"
```

**الـ output بيبان كده:**
```
gpiochip0: GPIOs 0-31, parent: platform/fe200000.gpio, pinctrl-bcm2835:
 gpio-4   (                    |sysfs               ) in  hi IRQ ACTIVE LOW
 gpio-17  (                    |shared-consumer-a   ) out lo [SHARED]
 gpio-17  (                    |shared-consumer-b   ) out lo [SHARED]
```

لما الـ line فيها `[SHARED]` يبقى الـ `GPIOD_FLAG_SHARED` (bit 20) مضبوط في `gpio_desc->flags`.

**الـ fields المهمة للـ shared descriptor:**

| Field | المعنى |
|-------|--------|
| `usecnt` | عدد الـ consumers اللي عندهم request حالياً |
| `highcnt` | عدد الـ consumers اللي طالبين HIGH |
| `cfg` | الـ configuration flags المدمجة |
| `can_sleep` | نوع الـ lock: mutex أو spinlock |

#### 2. sysfs Entries

```bash
# اعرض كل الـ GPIO lines المـ exported
ls /sys/class/gpio/

# اعرض direction وvalue لـ GPIO معين
cat /sys/class/gpio/gpio17/direction
cat /sys/class/gpio/gpio17/value

# اعرض الـ gpiochip info
cat /sys/class/gpio/gpiochip0/ngpio
cat /sys/class/gpio/gpiochip0/label
cat /sys/class/gpio/gpiochip0/base

# اعرض الـ device الـ parent
ls -la /sys/bus/platform/devices/ | grep gpio
```

> **ملاحظة:** الـ shared GPIO pins ممكن تظهر برقم واحد بس في sysfs لأن الـ `gpio_desc` واحد بيمثلهم، لكن `usecnt` بيعكس الـ consumers الحقيقيين.

#### 3. ftrace — Tracepoints والـ Events

```bash
# فعّل الـ GPIO tracepoints
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# تابع gpio_value events
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# تابع gpio_direction events
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable

# شغّل الـ tracer وشوف النتايج
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe

# وقّف
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**لـ trace الـ locking في الـ shared descriptor (function tracer):**
```bash
# trace الـ functions الخاصة بـ gpio_shared
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'gpio_shared*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

**مثال output:**
```
     kworker/0:1-45    [000] .... 1234.567890: gpio_device_setup_shared <-gpiochip_add_data_with_key
     consumer-drv-123  [001] .... 1234.568012: devm_gpiod_shared_get <-my_driver_probe
```

#### 4. printk / Dynamic Debug

```bash
# فعّل dynamic debug للـ GPIO subsystem كله
echo 'file drivers/gpio/* +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ gpiolib-shared تحديداً
echo 'file drivers/gpio/gpiolib-shared.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع الـ stack trace
echo 'file drivers/gpio/gpiolib-shared.c +ps' > /sys/kernel/debug/dynamic_debug/control

# اعرض الـ dynamic debug entries النشطة
cat /sys/kernel/debug/dynamic_debug/control | grep gpio
```

**الـ log messages اللي هتظهر:**
```
[ 1234.567] gpio-17 (shared-consumer-a): shared get: usecnt=1 highcnt=0
[ 1234.568] gpio-17 (shared-consumer-b): shared get: usecnt=2 highcnt=1
```

#### 5. Kernel Config Options للـ Debugging

| Config | الغرض |
|--------|--------|
| `CONFIG_GPIO_SHARED` | يفعّل الـ shared GPIO mechanism اصلاً |
| `CONFIG_DEBUG_GPIO` | يفعّل الـ verbose GPIO debugging |
| `CONFIG_DEBUG_MUTEXES` | يفعّل mutex debugging (مهم لـ `can_sleep=true`) |
| `CONFIG_DEBUG_SPINLOCK` | يفعّل spinlock debugging (لـ `can_sleep=false`) |
| `CONFIG_LOCKDEP` | يفعّل lock dependency checker — بيكشف deadlocks |
| `CONFIG_PROVE_LOCKING` | يفعّل full locking correctness proof |
| `CONFIG_LOCK_STAT` | lock contention statistics |
| `CONFIG_DEBUG_LOCK_ALLOC` | يتحقق من الـ lock lifecycle |
| `CONFIG_GPIOLIB_FASTPATH_LIMIT` | يحدد متى تدخل الـ fast path |
| `CONFIG_GPIO_CDEV` | يفعّل character device interface |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(GPIO|DEBUG_MUTEX|DEBUG_SPIN|LOCKDEP|PROVE_LOCK)'
```

#### 6. أدوات خاصة بالـ GPIO Subsystem

```bash
# gpio-tools (من libgpiod)
gpiodetect                          # اعرض كل الـ GPIO chips
gpioinfo gpiochip0                  # اعرض كل الـ lines في chip
gpioinfo gpiochip0 17               # اعرض line معينة
gpioget gpiochip0 17                # اقرأ قيمة الـ GPIO
gpioset gpiochip0 17=1              # اضبط قيمة
gpiomon gpiochip0 17                # راقب الـ edge events

# اعرض الـ GPIO uAPI من kernel
ls /dev/gpiochip*
```

**لفهم الـ shared state عبر `/proc`:**
```bash
# اعرض الـ processes اللي بتستخدم GPIO device
lsof /dev/gpiochip0

# اعرض الـ file descriptors المفتوحة
cat /proc/<pid>/fdinfo/<fd>
```

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `gpio-N (label): gpiochip not found` | الـ `gpio_device` اتحذف قبل ما الـ consumer يخلص | تأكد من الـ devm lifecycle وترتيب الـ probe |
| `gpio-N: sleeping function called from invalid context` | استخدام mutex في atomic context لما `can_sleep=true` | استخدم spinlock path أو غيّر الـ context |
| `BUG: spinlock lockup` | الـ spinlock اتعمل lock ولم يُفك | راجع الـ `DEFINE_LOCK_GUARD_1` flow وتأكد من الـ unlock |
| `WARNING: possible circular locking dependency` | lockdep كشف deadlock محتمل بين shared lock وlock تاني | راجع ترتيب الـ locking في كل الـ code paths |
| `gpio-N: direction refused` | محاولة تغيير direction على shared GPIO | الـ shared GPIO اتجاهه متحكم فيه بـ `cfg` المدمج |
| `devm_gpiod_shared_get: -ENOMEM` | فشل allocate الـ `gpio_shared_desc` | مشكلة في الـ memory، راجع الـ OOM logs |
| `gpio-N: chip is gone` | الـ `gpio_chip` اترفع أثناء الاستخدام | الـ `srcu` guard فشل، راجع الـ module unload order |
| `lockdep: held lock freed!` | الـ `gpio_shared_desc` اتحذف وفيه lock ممسكه | الـ devm teardown ترتيبه غلط |
| `gpio-N: proxy lookup failed` | `gpio_shared_add_proxy_lookup` فشل | تحقق من الـ `con_id` والـ `lflags` |

#### 8. Strategic Points لـ `dump_stack()` و`WARN_ON()`

الأماكن الاستراتيجية داخل الـ implementation:

```c
/* في gpio_device_setup_shared() — تحقق من الـ allocation */
if (WARN_ON(!shared_desc))
    return -ENOMEM;

/* في devm_gpiod_shared_get() — تحقق من الـ state قبل الـ lock */
WARN_ON(shared_desc->usecnt == 0 && shared_desc->highcnt != 0);

/* في gpio_shared_lockdep_assert() — مضمنة في الكود اصلاً */
if (shared_desc->can_sleep)
    lockdep_assert_held(&shared_desc->mutex);
else
    lockdep_assert_held(&shared_desc->spinlock);

/* لو عايز تعرف مين طلب الـ GPIO */
if (shared_desc->usecnt > 1) {
    pr_debug("gpio-%d: %u consumers active, highcnt=%u\n",
             desc_to_gpio(shared_desc->desc),
             shared_desc->usecnt,
             shared_desc->highcnt);
    dump_stack();
}
```

---

### Hardware Level

#### 1. التحقق أن الـ Hardware State بيطابق الـ Kernel State

الـ `gpio_shared_desc` بيحتفظ بـ `highcnt` — عدد الـ consumers اللي طالبين HIGH. الـ pin الفعلي بيكون HIGH لو `highcnt > 0`.

```bash
# اقرأ قيمة الـ GPIO من kernel perspective
gpioget gpiochip0 17

# قارن بالـ highcnt اللي ممكن تشوفه في debugfs
cat /sys/kernel/debug/gpio | grep "gpio-17"

# اقرأ الـ register مباشرة
# لـ BCM2835 (Raspberry Pi): GPFSEL, GPSET, GPCLR, GPLEV
devmem2 0xFE200034 w   # GPLEV0 — GPIO Level Register 0
```

**ASCII diagram للـ shared state vs hardware:**

```
Consumer A ──┐
             ├──► gpio_shared_desc ──► gpio_desc ──► gpio_chip ──► HW Pin
Consumer B ──┘    usecnt=2              flags         .get/.set    GPLEV
                  highcnt=1
                  can_sleep=false
                  (spinlock)
                       │
                       ▼
                 HW value = HIGH  (highcnt > 0)
```

#### 2. Register Dump Techniques

```bash
# باستخدام devmem2
apt install devmem2  # أو build من source

# مثال: BCM2711 (RPi4) GPIO base = 0xFE200000
BASE=0xFE200000

# GPFSEL0: GPIO Function Select 0 (pins 0-9)
devmem2 $((BASE + 0x00)) w

# GPLEV0: GPIO Pin Level 0 (pins 0-31)
devmem2 $((BASE + 0x34)) w

# GPEDS0: GPIO Event Detect Status 0
devmem2 $((BASE + 0x40)) w

# باستخدام /dev/mem مباشرة لو devmem2 مش موجود
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0xFE200000)
    val = struct.unpack('<I', m[0x34:0x38])[0]
    print(f'GPLEV0: 0x{val:08x}')
    for i in range(32):
        print(f'  GPIO{i}: {(val >> i) & 1}')
"

# باستخدام io utility (من package i2c-tools أو standalone)
io -4 -r 0xFE200034
```

#### 3. Logic Analyzer / Oscilloscope Tips

لو عندك shared GPIO بين consumers، المشكلة الشائعة هي **glitch** وقت التسليم بين الـ consumers:

```
Consumer A يحرر:
  highcnt: 2 -> 1  =>  PIN stays HIGH (صح)

Consumer A وConsumer B يحرروا في نفس الوقت بدون lock:
  Race condition:
  ___________
  PIN:       |____  (glitch غلط — المفروض يفضل HIGH)
```

**إعداد الـ Logic Analyzer:**
- Sample rate: minimum 10x أسرع من أسرع edge متوقع
- Trigger: falling edge على الـ shared pin
- Channel 2: IRQ line لو موجودة (لمقارنة التوقيت)
- Channel 3: I2C/SPI clock لو الـ GPIO بيتحكم في power rail

**إعداد الـ Oscilloscope:**
```
Ch1: GPIO Pin — الـ physical voltage (0 أو 3.3V/1.8V)
Ch2: VCC supply — تأكد انها مستقرة
Trigger: Ch1 falling، مستوى 1.5V
Time/div: 1us للـ fast transitions، 100us للـ slow
```

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الهاردوير | الـ Pattern في الـ Kernel Log | التفسير |
|------------------|------------------------------|---------|
| Pin مش بيستجيب للـ set | `gpio-17: direction refused` + pin ثابتة | ممكن short circuit أو hardware override |
| Floating input على shared pin | القيمة بتتغير عشوائياً في debugfs | Pull-up/down مش مضبوط، راجع الـ `cfg` في `gpio_shared_desc` |
| Power sequencing خطأ | `gpiochip not found` اثناء الـ probe | الـ GPIO controller مش powered لما الـ consumer بيـ probe |
| Noise على الـ IRQ shared pin | `irq N: nobody cared` متكرر | edge detection بيـ trigger على noise، ضيف debounce |
| Level mismatch | الـ `highcnt=0` لكن Pin = HIGH | الـ hardware بيـ override، راجع الـ boot firmware/bootloader |

#### 5. Device Tree Debugging

الـ `gpio_shared_add_proxy_lookup` بتحتاج الـ DT يكون صح:

```bash
# اعرض الـ compiled Device Tree
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 5 "gpio"

# أو استخدم fdtdump
fdtdump /boot/dtb/bcm2711-rpi-4-b.dtb | grep -A 10 "gpio"

# تحقق من GPIO lookups المضبوطة
cat /sys/kernel/debug/gpio | grep -E "gpio-[0-9]+"

# اعرض GPIO hogs المضبوطة في DT
cat /sys/kernel/debug/gpio | grep "hogged"

# تحقق من pinmux للـ GPIO shared pins
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -i gpio

# لو CONFIG_OF_DYNAMIC مفعّل، اعرض الـ hog node
cat /sys/firmware/devicetree/base/gpio@*/gpio-line-names 2>/dev/null
```

**DT صح للـ shared GPIO:**
```dts
/* الـ GPIO المشترك بين consumer-a وconsumer-b */
&gpio {
    shared_pin: shared-pin {
        /* pin 17 مشترك */
        gpio-hog;
        gpios = <17 GPIO_ACTIVE_HIGH>;
        output-high;
        line-name = "shared-power-en";
    };
};

&consumer_a {
    power-gpios = <&gpio 17 GPIO_ACTIVE_HIGH>;
};

&consumer_b {
    enable-gpios = <&gpio 17 GPIO_ACTIVE_HIGH>;
};
```

**تحقق من الـ DT بيطابق الـ hardware:**
```bash
# اعرض GPIO number من DT perspective
grep -r "gpio" /sys/firmware/devicetree/base/ 2>/dev/null | head -20

# تحقق من ان الـ pin مش مـ assigned لـ pinctrl function تاني
cat /sys/kernel/debug/pinctrl/*/pins | grep "pin 17"
```

---

### Practical Commands

#### Ready-to-Copy Commands

**1. فحص شامل للـ Shared GPIO State:**
```bash
#!/bin/bash
# gpio-shared-debug.sh — فحص شامل للـ shared GPIO subsystem

echo "=== GPIO Chip Info ==="
gpiodetect

echo ""
echo "=== GPIO Lines State ==="
cat /sys/kernel/debug/gpio

echo ""
echo "=== GPIO Events (ftrace 5 seconds) ==="
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 5
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
echo 0 > /sys/kernel/debug/tracing/events/gpio/enable
```

**2. فعّل Dynamic Debug للـ GPIO Shared:**
```bash
# فعّل debug messages مع line numbers وfunction names
echo 'file drivers/gpio/gpiolib-shared.c +pflmt' \
    > /sys/kernel/debug/dynamic_debug/control

# أو للـ module كله
echo 'module gpiolib +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages
dmesg -w | grep -i "gpio.*shared"
```

**3. راقب الـ Lock Contention:**
```bash
# فعّل lock stats (يحتاج CONFIG_LOCK_STAT=y)
echo 1 > /proc/sys/kernel/lock_stat

# شغّل workload
sleep 10

# اعرض النتايج
cat /proc/lock_stat | grep -E "(gpio|mutex|spin)" | head -20

# وقّف
echo 0 > /proc/sys/kernel/lock_stat
```

**4. تتبع الـ gpio_shared functions عبر الـ ftrace:**
```bash
# trace function مع args (يحتاج CONFIG_FUNCTION_GRAPH_TRACER)
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'gpio_shared*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ consumer driver
modprobe my_gpio_consumer

cat /sys/kernel/debug/tracing/trace | grep gpio_shared
```

**5. كشف الـ Deadlock المحتمل بـ lockdep:**
```bash
# راجع الـ lockdep output بعد الـ probe
dmesg | grep -A 30 "possible circular locking"

# أو
dmesg | grep -E "(WARNING|BUG|lockdep|spinlock)" | head -50
```

**6. اعرض قيم الـ registers مباشرة:**
```bash
# للـ BCM2711 على RPi4
GPIO_BASE=0xFE200000

echo "GPFSEL0 (Function Select 0):"
devmem2 $((GPIO_BASE + 0x00)) w

echo "GPFSEL1 (Function Select 1, pins 10-19):"
devmem2 $((GPIO_BASE + 0x04)) w

echo "GPLEV0 (Level pins 0-31):"
devmem2 $((GPIO_BASE + 0x34)) w

# فسّر الـ GPLEV0
val=$(devmem2 $((GPIO_BASE + 0x34)) w 2>/dev/null | awk '/Read/ {print $NF}')
echo "GPIO 17 value: $(( (val >> 17) & 1 ))"
```

#### Example Output وكيفية تفسيره

**debugfs output مع shared GPIO:**
```
gpiochip0: GPIOs 0-31, parent: platform/fe200000.gpio, pinctrl-bcm2835:
 gpio-17  (                    |consumer-a          ) out hi [SHARED]
```
- **`out hi`** — الـ pin حالياً output وقيمته HIGH
- **`[SHARED]`** — `GPIOD_FLAG_SHARED` مضبوط، يعني `usecnt >= 1`
- لو فيه consumer تاني هتلاقي نفس الـ gpio-17 بـ label تاني في السطر الجاي

**ftrace output للـ shared events:**
```
 consumer-a-234  [001] ....  1001.123456: gpio_value: 17 set 1
 consumer-b-567  [001] ....  1001.124789: gpio_value: 17 set 0
 consumer-b-567  [001] ....  1001.124790: gpio_value: 17 set 1
```
السطر الأخير يعني الـ `highcnt` رجع يكون `> 0` بعد ما consumer-a لسه طالب HIGH، فالـ pin فضل HIGH رغم ان consumer-b طلب LOW — ده الـ behavior الصح للـ shared GPIO.

**lockdep warning:**
```
WARNING: possible circular locking dependency detected
consumer-a/234 is trying to acquire lock:
  (gpio_shared_desc_lock){....}, at: devm_gpiod_shared_get+0x48

but task is already holding lock:
  (&dev->mutex){....}, at: device_lock+0x1c

the existing dependency chain:
  -> #0 (gpio_shared_desc_lock):
  -> #1 (&dev->mutex):
```
ده يعني الـ consumer بيمسك `dev->mutex` وبعدين بيحاول يمسك `gpio_shared_desc_lock` — لو في code path تاني بيعمل العكس هيحصل deadlock. الحل: رتّب الـ locking بنفس الترتيب في كل مكان، أو استخدم `lockdep_assert_held()` اللي موجودة اصلاً في `gpio_shared_lockdep_assert()`.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Deadlock في Industrial Gateway على RK3562

#### العنوان
**GPIO Shared Deadlock** بين driver الـ I2C والـ interrupt handler في industrial gateway

#### السياق
شركة بتبني industrial gateway بيستخدم **RK3562** — الـ gateway ده بيتحكم في 8 sensors عن طريق I2C. الـ GPIO المشترك بيتحكم في enable line لـ power rail بتاع الـ sensors. الـ driver بيستخدم `gpio_shared_desc` مع `mutex` لأن الـ I2C calls ممكن تـ sleep.

#### المشكلة
الـ system بيـ hang بعد تشغيله بساعتين تقريباً. الـ kernel log بيظهر:

```
INFO: possible circular locking dependency detected
gpio_shared_desc_lock -> i2c_lock -> gpio_shared_desc_lock
```

الـ watchdog بيـ trigger وكل حاجة بتـ reboot.

#### التحليل
الـ bug في الـ `DEFINE_LOCK_GUARD_1` اللي في الملف:

```c
DEFINE_LOCK_GUARD_1(gpio_shared_desc_lock, struct gpio_shared_desc,
    if (_T->lock->can_sleep)
        mutex_lock(&_T->lock->mutex);   /* blocking lock — process context only */
    else
        spin_lock_irqsave(&_T->lock->spinlock, _T->flags),
    if (_T->lock->can_sleep)
        mutex_unlock(&_T->lock->mutex);
    else
        spin_unlock_irqrestore(&_T->lock->spinlock, _T->flags),
    unsigned long flags)
```

الـ sequence اللي حصل:

```
Thread A (I2C driver):
  1. يمسك gpio_shared_desc_lock (mutex)
  2. يبعت I2C transaction
  3. ينتظر I2C completion

IRQ Handler:
  4. I2C interrupt يطلع
  5. يحاول يمسك gpio_shared_desc_lock تاني (deadlock!)
```

الـ `can_sleep = true` اتحدد لأن الـ GPIO controller الـ underlying على RK3562 بيحتاج I2C access للـ register read/write — لكن الـ interrupt handler اتـ register بدون `IRQF_NO_SUSPEND` وبيحاول يـ acquire نفس الـ mutex.

الـ `gpio_shared_lockdep_assert` لو اتستخدمت صح كانت هتكتشف الـ violation قبل الـ deadlock:

```c
static inline void gpio_shared_lockdep_assert(struct gpio_shared_desc *shared_desc)
{
    if (shared_desc->can_sleep)
        lockdep_assert_held(&shared_desc->mutex);
    else
        lockdep_assert_held(&shared_desc->spinlock);
}
```

لكن الـ assert مش موجود في الـ IRQ handler — الـ lockdep مش شايف الـ dependency chain كامل.

#### الحل

**الخطوة 1:** تغيير design — الـ IRQ handler مش المفروض يـ access الـ shared GPIO مباشرة. بدل كده نستخدم `work_struct`:

```c
struct my_sensor_dev {
    struct gpio_shared_desc *shared_gpio;
    struct work_struct gpio_work;
    bool gpio_requested_state;
};

static irqreturn_t sensor_irq_handler(int irq, void *data)
{
    struct my_sensor_dev *dev = data;
    /* defer to workqueue — don't touch shared GPIO here */
    dev->gpio_requested_state = true;
    schedule_work(&dev->gpio_work);
    return IRQ_HANDLED;
}

static void sensor_gpio_work(struct work_struct *work)
{
    struct my_sensor_dev *dev =
        container_of(work, struct my_sensor_dev, gpio_work);
    /* now safe to take mutex — we're in process context */
    guard(gpio_shared_desc_lock)(dev->shared_gpio);
    gpiod_set_value_cansleep(dev->shared_gpio->desc,
                             dev->gpio_requested_state);
}
```

**الخطوة 2:** تفعيل `CONFIG_PROVE_LOCKING=y` في kernel config على الـ development board:

```bash
CONFIG_PROVE_LOCKING=y
CONFIG_DEBUG_LOCKDEP=y
CONFIG_LOCK_STAT=y
```

#### الدرس المستفاد
الـ `can_sleep` flag في `gpio_shared_desc` مش بس hint — هو يحدد نوع الـ lock المستخدم. لو الـ GPIO controller بتاعك يحتاج sleepable context (زي I2C-connected GPIO expander)، فـ interrupt handlers مش المفروض يـ acquire الـ `gpio_shared_desc_lock` مباشرة. استخدم workqueue أو threaded IRQ.

---

### السيناريو 2: Race Condition في Android TV Box على Allwinner H616

#### العنوان
**HDMI HPD GPIO** مشترك بين الـ display driver والـ audio driver — `usecnt` corruption

#### السياق
Android TV box ببني على **Allwinner H616**. الـ HDMI Hot-Plug Detect (HPD) GPIO مشترك بين:
- `sun8i-drm` (display driver) — يقرأ GPIO لـ detect connection
- `sun50i-hdmi-audio` (audio driver) — نفس الـ GPIO يـ trigger audio path switch

الـ `CONFIG_GPIO_SHARED=y` اتضاف عشان يـ manage الـ shared access.

#### المشكلة
بعد كذا plugin/unplug للـ HDMI:

```
kernel: gpio-shared: usecnt underflow on GPIO HPD-HDMI!
kernel: WARNING: CPU: 2 PID: 847 at drivers/gpio/gpiolib-shared.c:xxx
```

الـ `usecnt` بيوصل لـ 0 وبعدين بيـ wrap-around لـ `UINT_MAX`.

#### التحليل
الـ `gpio_shared_desc` struct بتتابع الـ users:

```c
struct gpio_shared_desc {
    struct gpio_desc *desc;
    bool can_sleep;
    unsigned long cfg;
    unsigned int usecnt;   /* عدد الـ consumers الحاليين */
    unsigned int highcnt;  /* كام واحد طالب high level */
    union {
        struct mutex mutex;
        spinlock_t spinlock;
    };
};
```

الـ `devm_gpiod_shared_get` بيـ allocate وبيـ register الـ `gpio_shared_desc`. الـ cleanup بيحصل عن طريق الـ `devm` mechanism — لما الـ device بيـ release، الـ `usecnt` المفروض يرجع صفر بشكل نضيف.

المشكلة: الـ audio driver بيـ unbind/rebind عند كل HDMI disconnect عن طريق الـ Android `audioserver` restart. الـ sequence:

```
1. audio driver: devm_gpiod_shared_get()  -> usecnt = 2
2. display driver: يشتغل عادي
3. HDMI unplug:
   - audio driver: unbind -> devm cleanup -> usecnt-- = 1  [OK]
   - audio driver: bind مرة تانية (قبل ما display يـ notice) -> usecnt++ = 2
4. display driver: يعمل cleanup -> usecnt-- = 1  (مش 0!)
5. audio driver: unbind تاني -> usecnt-- = 0  [OK]
6. audio driver: unbind مرة تالتة (bug في Android audioserver) -> usecnt-- = UINT_MAX  [BOOM]
```

الـ `gpio_device_setup_shared` و `gpio_device_teardown_shared` من الـ API بيحموا الـ `gpio_device` لكن مش الـ consumer-level `usecnt` من racing `devm` cleanups.

#### الحل

**الخطوة 1:** إضافة defensive check في الـ consumer code:

```c
static int hdmi_audio_probe(struct platform_device *pdev)
{
    struct hdmi_audio_dev *adev;
    struct gpio_shared_desc *shared;

    shared = devm_gpiod_shared_get(&pdev->dev);
    if (IS_ERR(shared)) {
        dev_err(&pdev->dev, "failed to get shared HPD GPIO: %ld\n",
                PTR_ERR(shared));
        return PTR_ERR(shared);
    }

    /* verify we actually got a valid descriptor */
    if (!shared->desc) {
        dev_err(&pdev->dev, "shared GPIO descriptor is NULL\n");
        return -ENODEV;
    }

    adev->hpd_shared = shared;
    return 0;
}
```

**الخطوة 2:** تحليل الـ race بـ `ftrace`:

```bash
echo 'gpio_shared*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# افعل HDMI unplug
cat /sys/kernel/debug/tracing/trace | grep usecnt
```

**الخطوة 3:** إصلاح الـ audioserver في Android للتأكد من عدم الـ double-unbind.

#### الدرس المستفاد
الـ `usecnt` في `gpio_shared_desc` بيُدار من الـ subsystem، لكن الـ consumer code لازم يكون defensive ويتأكد إن الـ lifecycle بتاعه صحيح. الـ `devm` رائع لكن مش بديل عن الـ clean driver lifecycle management — خصوصاً لما Android framework بتـ rebind drivers بشكل aggressive.

---

### السيناريو 3: غياب CONFIG_GPIO_SHARED في IoT Sensor Node على STM32MP1

#### العنوان
الـ shared GPIO بيـ compile كـ no-op وكل الـ consumers بيكتبوا على نفس الـ GPIO بدون تنسيق

#### السياق
IoT sensor node ببني على **STM32MP1**. فيه GPIO واحد (PD14) بيتحكم في الـ power enable لـ 3V3 rail اللي بيشغّل:
- BME280 (temperature/humidity sensor) عن طريق SPI
- SHT31 (another humidity sensor) عن طريق I2C
- LED indicator

كل device له driver منفصل وكل driver بيعمل `gpiod_get()` على نفس الـ GPIO.

#### المشكلة
عند boot، الـ I2C sensor driver بيقفل الـ power GPIO (يحطه low) قبل ما الـ SPI driver يخلص initialization — النتيجة: الـ SPI sensor بيـ fail بـ timeout errors وبيدخل في retry loop لا نهاية لها.

```
spi0.0: BME280 read failed: -ETIMEDOUT
spi0.0: BME280 read failed: -ETIMEDOUT
i2c-1: SHT31 probe successful
spi0.0: BME280 read failed: -ETIMEDOUT
```

#### التحليل
المشكلة الأساسية: `CONFIG_GPIO_SHARED` **مش مفعّل** في الـ kernel config. لما بتبص في `gpiolib-shared.h`:

```c
#if IS_ENABLED(CONFIG_GPIO_SHARED)

int gpio_device_setup_shared(struct gpio_device *gdev);
void gpio_device_teardown_shared(struct gpio_device *gdev);
int gpio_shared_add_proxy_lookup(struct device *consumer, const char *con_id,
                                 unsigned long lflags);

#else

/* كل الـ functions بتبقى no-ops! */
static inline int gpio_device_setup_shared(struct gpio_device *gdev)
{
    return 0;  /* لا setup، لا tracking، لا coordination */
}

static inline void gpio_device_teardown_shared(struct gpio_device *gdev) { }

static inline int gpio_shared_add_proxy_lookup(struct device *consumer,
                                               const char *con_id,
                                               unsigned long lflags)
{
    return 0;  /* proxy lookup مش شغال */
}

#endif /* CONFIG_GPIO_SHARED */
```

لما `CONFIG_GPIO_SHARED=n`، مفيش `gpio_shared_desc` يتبني، مفيش `usecnt` يتـ track، مفيش lock يحمي الـ shared access. كل driver بيأخذ الـ GPIO لوحده بدون ما يعرف عن التاني.

#### الحل

**الخطوة 1:** تفعيل `CONFIG_GPIO_SHARED` في kernel config:

```bash
# في menuconfig
Device Drivers -> GPIO Support -> [*] Support for shared GPIO lines
# أو مباشرة في .config
echo "CONFIG_GPIO_SHARED=y" >> arch/arm/configs/stm32mp1_defconfig
```

**الخطوة 2:** تحديث الـ Device Tree لتعريف الـ GPIO كـ shared:

```dts
/ {
    power-enable-gpio {
        compatible = "gpio-shared";
        gpios = <&gpiod 14 GPIO_ACTIVE_HIGH>;
    };

    spi-sensor {
        compatible = "bosch,bme280";
        power-gpios = <&power_enable_gpio>;
    };

    i2c-sensor {
        compatible = "sensirion,sht31";
        power-gpios = <&power_enable_gpio>;
    };
};
```

**الخطوة 3:** في كل driver، استخدام `devm_gpiod_shared_get` بدل `gpiod_get`:

```c
static int bme280_probe(struct spi_device *spi)
{
    struct gpio_shared_desc *pwr_gpio;

    pwr_gpio = devm_gpiod_shared_get(&spi->dev);
    if (IS_ERR(pwr_gpio))
        return PTR_ERR(pwr_gpio);

    {
        guard(gpio_shared_desc_lock)(pwr_gpio);
        pwr_gpio->usecnt++;
        /* اشتغل بأمان */
    }
    return 0;
}
```

#### الدرس المستفاد
الـ `#if IS_ENABLED(CONFIG_GPIO_SHARED)` guard في `gpiolib-shared.h` يعني إن كل حاجة بتبقى silent no-op لما الـ config مش مفعّل. مفيش error، مفيش warning — بس مفيش حماية. دايماً افحص `CONFIG_GPIO_SHARED=y` في الـ `.config` قبل ما تعتمد على أي shared GPIO coordination.

---

### السيناريو 4: Interrupt Context والـ spinlock على i.MX8 في Automotive ECU

#### العنوان
استخدام صح لـ `spin_lock_irqsave` مع `gpio_shared_desc` في real-time automotive context

#### السياق
Automotive ECU ببني على **i.MX8QM** بيشتغل على Linux مع PREEMPT_RT patch. الـ ECU بيتحكم في CAN bus transceiver enable line — الـ GPIO ده مشترك بين:
- CAN driver (بيشغّله)
- Power Management controller (بيقفله في low-power mode)

المتطلب: الـ CAN enable GPIO لازم يـ respond في أقل من 100 microseconds من الـ interrupt — ده يعني **مش مقبول** استخدام mutex (بيـ sleep).

#### المشكلة
الـ engineer الجديد غيّر الـ `can_sleep = false` إلى `can_sleep = true` عشان يحل warning مش فاهمه:

```
[ 0.847] gpio-shared: GPIO CAN-EN: using spinlock (non-sleepable)
```

بعد التغيير، الـ system أبدأ يـ hit:

```
BUG: scheduling while atomic: can_tx/847/0x00010001
```

#### التحليل
الـ `DEFINE_LOCK_GUARD_1` في `gpiolib-shared.h` بيختار الـ lock type بناءً على `can_sleep`:

```c
DEFINE_LOCK_GUARD_1(gpio_shared_desc_lock, struct gpio_shared_desc,
    if (_T->lock->can_sleep)
        mutex_lock(&_T->lock->mutex);      /* sleepable — process context only */
    else
        spin_lock_irqsave(&_T->lock->spinlock, _T->flags), /* safe in IRQ context */
    if (_T->lock->can_sleep)
        mutex_unlock(&_T->lock->mutex);
    else
        spin_unlock_irqrestore(&_T->lock->spinlock, _T->flags),
    unsigned long flags)
```

لما `can_sleep = true` والـ CAN driver بيـ call الـ GPIO من interrupt context، الـ kernel بيـ detect "scheduling while atomic" لأن الـ mutex implicitly بيـ allow sleeping.

الـ `gpio_shared_lockdep_assert` كانت هتكتشف ده لو اتستخدمت:

```c
static inline void gpio_shared_lockdep_assert(struct gpio_shared_desc *shared_desc)
{
    if (shared_desc->can_sleep)
        lockdep_assert_held(&shared_desc->mutex);
    else
        lockdep_assert_held(&shared_desc->spinlock);
}
```

#### الحل

**الخطوة 1:** رجوع `can_sleep = false` — ده هو الصح للـ interrupt context:

```c
static int can_transceiver_probe(struct platform_device *pdev)
{
    struct gpio_shared_desc *en_gpio;

    en_gpio = devm_gpiod_shared_get(&pdev->dev);
    if (IS_ERR(en_gpio))
        return PTR_ERR(en_gpio);

    /*
     * can_sleep MUST be false — CAN TX happens in hard IRQ context
     * on i.MX8QM FLEXCAN peripheral. Spinlock ensures < 1us latency.
     */
    if (en_gpio->can_sleep) {
        dev_err(&pdev->dev,
            "CAN enable GPIO must be non-sleepable (check DT)\n");
        return -EINVAL;
    }

    return 0;
}
```

**الخطوة 2:** في الـ interrupt handler، استخدام الـ guard بشكل صح:

```c
static irqreturn_t can_tx_irq(int irq, void *data)
{
    struct can_ecu_dev *edev = data;

    /* spinlock_irqsave يتأخذ تلقائياً عن طريق الـ guard */
    guard(gpio_shared_desc_lock)(edev->en_gpio);

    /* آمن — spinlock محجوز، interrupts معطلة محلياً */
    gpiod_set_value(edev->en_gpio->desc, 1);

    /* الـ lock بيتحرر تلقائياً هنا (scope-based cleanup) */
    return IRQ_HANDLED;
}
```

**الخطوة 3:** تفعيل `CONFIG_DEBUG_ATOMIC_SLEEP=y` في development:

```bash
CONFIG_DEBUG_ATOMIC_SLEEP=y
CONFIG_PROVE_LOCKING=y
```

#### الدرس المستفاد
الـ `can_sleep` flag في `gpio_shared_desc` مش cosmetic — هو بيحدد إيه نوع الـ lock اللي الـ `DEFINE_LOCK_GUARD_1` هيستخدمه. في automotive/real-time systems، الـ GPIO اللي بيتعمله access من IRQ context **لازم** يستخدم spinlock (`can_sleep = false`). الـ warning "using spinlock" ده مش مشكلة — ده هو السلوك الصح.

---

### السيناريو 5: Custom Board Bring-up على AM62x — GPIO Shared Proxy Lookup Failure

#### العنوان
**`gpio_shared_add_proxy_lookup` failure** يمنع multi-driver board bring-up على Texas Instruments AM62x

#### السياق
Custom industrial board ببني على **AM62x** (TI Sitara) لتطبيق factory automation. الـ board فيها GPIO واحد (GPIO0_15) بيتحكم في enable line لـ RS-485 transceiver. الـ GPIO ده مشترك بين:
- الـ primary serial driver (`tty-rs485-primary`)
- الـ redundant serial driver (`tty-rs485-redundant`) — for failover

الـ board جديدة ومفيش reference design.

#### المشكلة
عند boot:

```
tty-rs485-primary: gpio_shared_add_proxy_lookup failed: -ENODEV
tty-rs485-primary: probe failed
tty-rs485-redundant: gpio_shared_add_proxy_lookup failed: -ENODEV
```

الـ RS-485 communication مش شغال خالص.

#### التحليل
الـ `gpio_shared_add_proxy_lookup` من `gpiolib-shared.h`:

```c
int gpio_shared_add_proxy_lookup(struct device *consumer, const char *con_id,
                                 unsigned long lflags);
```

الـ function دي بتربط الـ consumer device بالـ shared GPIO عن طريق الـ GPIO lookup table. الـ `-ENODEV` بيعني إن الـ GPIO device نفسه مش اتسجل في الـ kernel بعد.

الـ investigation خطوة بخطوة:

**الخطوة 1:** فحص الـ Device Tree:

```bash
ls /sys/bus/gpio/devices/
# النتيجة: فاضي! مفيش gpio devices
```

**الخطوة 2:** فحص الـ GPIO device setup:

```c
/* gpiolib-shared.h بيعرف: */
int gpio_device_setup_shared(struct gpio_device *gdev);
```

الـ `gpio_device_setup_shared` بيتـ call من الـ GPIO controller driver عند الـ probe. لو الـ GPIO controller مش اتـ probe، الـ `gpio_device` مش موجود، والـ proxy lookup بيفشل.

**الخطوة 3:** فحص الـ GPIO controller في DT:

```bash
find /sys/firmware/devicetree/base -name "compatible" -exec grep -l "gpio" {} \;
# النتيجة: مفيش gpio controller nodes!
```

**الخطوة 4:** فحص الـ board DTS file — اكتشاف المشكلة:

```dts
/* الـ DTS الغلط — GPIO controller node مش مكتمل */
&main_gpio0 {
    /* status مش محدد — default هو "disabled" في AM62x base DTS! */
    pinctrl-names = "default";
    pinctrl-0 = <&main_gpio0_pins>;
};
```

#### الحل

**الخطوة 1:** إضافة `status = "okay"` للـ GPIO controller node:

```dts
/* board.dts */
&main_gpio0 {
    status = "okay";    /* ده كان ناقص */
    pinctrl-names = "default";
    pinctrl-0 = <&main_gpio0_pins>;
};
```

**الخطوة 2:** تعريف الـ shared GPIO node:

```dts
/ {
    rs485-enable-gpio {
        compatible = "gpio-shared";
        gpios = <&main_gpio0 15 GPIO_ACTIVE_HIGH>;
        gpio-line-names = "RS485-EN";
        status = "okay";
    };
};
```

**الخطوة 3:** في كل driver، استخدام الـ correct connection ID:

```c
static int rs485_primary_probe(struct platform_device *pdev)
{
    struct gpio_shared_desc *en;
    int ret;

    /* register proxy lookup قبل ما تـ get الـ GPIO */
    ret = gpio_shared_add_proxy_lookup(&pdev->dev, "rs485-en", 0);
    if (ret) {
        dev_err(&pdev->dev,
            "proxy lookup failed: %d — is main_gpio0 enabled in DTS?\n",
            ret);
        return ret;
    }

    en = devm_gpiod_shared_get(&pdev->dev);
    if (IS_ERR(en))
        return PTR_ERR(en);

    /* gpio_device_setup_shared اتعمل بالفعل من GPIO controller */
    dev_info(&pdev->dev, "RS485 enable GPIO shared, usecnt=%u\n",
             en->usecnt);
    return 0;
}
```

**الخطوة 4:** التحقق بعد الإصلاح:

```bash
# تأكد من إن GPIO controller اتـ probe
ls /sys/bus/gpio/devices/
# gpiochip0  gpiochip1  ...

# تأكد من الـ shared GPIO
cat /sys/kernel/debug/gpio
# gpio-15 (RS485-EN): output, shared, users=2

# اختبار الـ RS-485
minicom -D /dev/ttyS2 -b 9600
```

#### الدرس المستفاد
الـ `gpio_shared_add_proxy_lookup` بيفتقر على وجود الـ `gpio_device` اللي بيُنشئه `gpio_device_setup_shared` — وده بيحتاج إن الـ GPIO controller node في الـ Device Tree يكون **مفعّل صراحةً** بـ `status = "okay"`. في SoCs زي AM62x اللي الـ base DTS بتاعها بتـ disable كل peripherals by default، ده غلطة board bring-up شائعة جداً. دايماً ابدأ الـ bring-up بالتحقق من `ls /sys/bus/gpio/devices/` قبل أي حاجة تانية.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأول لأي حاجة تخص تطور الـ Linux kernel. الـ GPIO subsystem عنده تغطية ممتازة — الروابط دي كلها من نتائج بحث حقيقي:

| العنوان | الرابط | الأهمية |
|---------|--------|---------|
| GPIO in the kernel: an introduction | [lwn.net/Articles/532714](https://lwn.net/Articles/532714/) | نقطة البداية — شرح شامل للـ gpiolib والـ API |
| GPIO in the kernel: future directions | [lwn.net/Articles/533632](https://lwn.net/Articles/533632/) | تطور الـ API والتوجهات المستقبلية بعد 2.6.21 |
| New descriptor-based GPIO interface | [lwn.net/Articles/565662](https://lwn.net/Articles/565662/) | الإعلان الرسمي عن الـ `gpiod_get()` API الجديدة |
| Documentation: gpiolib: document new interface | [lwn.net/Articles/574055](https://lwn.net/Articles/574055/) | توثيق الـ descriptor interface الجديد في `Documentation/` |
| **gpio: improve support for shared GPIOs** | [**lwn.net/Articles/1039548**](https://lwn.net/Articles/1039548/) | **الأهم لـ `gpiolib-shared.h`** — patch series الـ `gpio_shared_desc` |
| gpio: rework locking and object life-time control | [lwn.net/Articles/960024](https://lwn.net/Articles/960024/) | إعادة هيكلة الـ locking في gpiolib — متعلق بالـ `mutex`/`spinlock` union |
| Add support for software nodes to gpiolib | [lwn.net/Articles/914358](https://lwn.net/Articles/914358/) | إضافة software nodes — بيوضح مرونة الـ lookup system |
| Documentation: gpio: add character device userspace API | [lwn.net/Articles/958322](https://lwn.net/Articles/958322/) | توثيق الـ chardev ABI اللي بيتفاعل مع الـ `gpio_device` |

---

### توثيق الـ Kernel الرسمي

الـ `Documentation/driver-api/gpio/` هو المرجع الرسمي داخل الـ kernel tree:

```
Documentation/driver-api/gpio/index.rst           ← الفهرس الرئيسي
Documentation/driver-api/gpio/intro.rst           ← مقدمة الـ GPIO subsystem
Documentation/driver-api/gpio/consumer.rst        ← واجهة الـ consumer (gpiod_get / devm_gpiod_*)
Documentation/driver-api/gpio/driver.rst          ← واجهة كتابة الـ gpio chip driver
Documentation/driver-api/gpio/board.rst           ← GPIO mappings (device tree / ACPI / platform)
Documentation/driver-api/gpio/legacy.rst          ← الـ integer-based API القديم (deprecated)
Documentation/driver-api/gpio/drivers-on-gpio.rst ← subsystems تعتمد على GPIO
```

روابط مباشرة على docs.kernel.org:

- [GPIO Driver API Index](https://docs.kernel.org/driver-api/gpio/index.html)
- [GPIO Consumer Interface](https://docs.kernel.org/driver-api/gpio/consumer.html) — يشمل `devm_gpiod_get()` اللي بيرجع `gpio_shared_desc`
- [GPIO Driver Interface](https://static.lwn.net/kerneldoc/driver-api/gpio/driver.html)
- [GPIO Mappings (Board / DT / ACPI)](https://static.lwn.net/kerneldoc/driver-api/gpio/board.html)
- [Subsystem drivers using GPIO](https://static.lwn.net/kerneldoc/driver-api/gpio/drivers-on-gpio.html)
- [Legacy GPIO Interfaces](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html)
- [GPIO Character Device Userspace API](https://docs.kernel.org/userspace-api/gpio/chardev.html)
- [Using GPIO Lines in Linux](https://docs.kernel.org/driver-api/gpio/using-gpio.html)

---

### Kernel Commits والـ Patch Series ذات الصلة

الـ **gpiolib-shared.h** وبنية `gpio_shared_desc` والـ `CONFIG_GPIO_SHARED` دخلوا الـ kernel من خلال patch series كتبها **Bartosz Golaszewski** ونُوقشت على `linux-hardening@vger.kernel.org`:

| إصدار الـ Patch | الرابط | الوصف |
|----------------|--------|-------|
| RFC 6/9 | [mail-archive](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg11042.html) | أول نسخة — `gpiolib: support shared GPIOs in core subsystem code` |
| RFC 4/9 | [mail-archive](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg11233.html) | `gpiolib: implement low-level, shared GPIO support` — تعريف `gpio_shared_desc` |
| v2 05/10 | [mail-archive](http://www.mail-archive.com/linux-hardening@vger.kernel.org/msg11312.html) | المراجعة التانية — تحسينات الـ locking |
| v3 05/10 | [lkml.org](https://lkml.org/lkml/2025/10/29/866) | نسخة v3 على LKML |
| v4 05/10 | [mail-archive](http://www.mail-archive.com/linux-hardening@vger.kernel.org/msg11496.html) | **النسخة النهائية** — `Acked-by: Linus Walleij` — المدمجة في الـ mainline |

تتبع الـ history مباشرة بالـ git:

```bash
# تتبع تاريخ الملف
git log --oneline drivers/gpio/gpiolib-shared.h

# الـ commit اللي أضاف الملف للمرة الأولى
git log --diff-filter=A -- drivers/gpio/gpiolib-shared.h

# كل الـ commits المتعلقة بالـ shared GPIO
git log --oneline --grep="gpio.*shared\|shared.*gpio" -- drivers/gpio/
```

---

### نقاشات Mailing List

**الـ linux-gpio@vger.kernel.org** هو الـ mailing list الرسمي للـ GPIO subsystem. أرشيفه على lore.kernel.org:

- [lore.kernel.org/linux-gpio](https://lore.kernel.org/linux-gpio/) — أرشيف كامل
- [PATCH RFC: gpiolib shared GPIOs على linux-hardening](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg11042.html)
- [PATCH v4: final shared GPIO support](http://www.mail-archive.com/linux-hardening@vger.kernel.org/msg11496.html)
- [kernelnewbies: devm_gpiod_get usage discussion](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html)
- [kernelnewbies: GPIO from device tree on platform devices](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html)

---

### Kernelnewbies.org

**الـ kernelnewbies.org** بيوثق التغييرات في كل kernel release. أهم الصفحات للـ GPIO:

| الصفحة | الرابط | ما يخص GPIO |
|--------|--------|------------|
| Linux 2.6.21 | [kernelnewbies.org/Linux_2_6_21](https://kernelnewbies.org/Linux_2_6_21) | أول إضافة الـ GPIO API للـ kernel |
| Linux 2.6.25 | [kernelnewbies.org/Linux_2_6_25](https://kernelnewbies.org/Linux_2_6_25) | توسعة مبكرة للـ GPIO support |
| Linux 6.6 | [kernelnewbies.org/Linux_6.6](https://kernelnewbies.org/Linux_6.6) | تغييرات الـ GPIO في الإصدارات الحديثة |
| Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) | GPIO support لـ chipsets جديدة |
| Linux 6.13 | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) | آخر تغييرات GPIO — Aspeed G7 وغيره |
| LinuxChanges Index | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) | الفهرس الكامل لكل الإصدارات |

---

### eLinux.org

**الـ eLinux.org** مركز موارد الـ embedded Linux — مفيد للجانب العملي:

| المورد | الرابط | الفائدة |
|--------|--------|---------|
| New GPIO Interface for User Space — ELCE 2017 | [elinux.org PDF](https://elinux.org/images/7/74/Elce2017_new_GPIO_interface.pdf) | شرح بصري للـ chardev ABI الجديد |
| Introduction to pin muxing and GPIO — ELC 2021 | [elinux.org PDF](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) | pinmux وعلاقته بالـ GPIO في embedded |
| EBC GPIO via mmap | [elinux.org/EBC_GPIO_via_mmap](https://elinux.org/EBC_GPIO_via_mmap) | الوصول المباشر لـ GPIO registers عبر mmap |
| EBC GPIO Polling and Interrupts | [elinux.org/EBC_gpio_Polling_and_Interrupts](https://elinux.org/EBC_gpio_Polling_and_Interrupts) | GPIO polling والـ interrupts من userspace |

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 6** — Advanced Char Driver Operations: بيشرح الـ ioctl والـ locking الـ patterns اللي بتشوفها في `gpio_shared_desc`
- **الفصل 14** — The Linux Device Model: بيشرح `struct device` وآلية الـ `devm_*` resource management اللي بيستخدمها `devm_gpiod_shared_get()`
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 10** — Kernel Synchronization Methods: شرح `mutex_lock()` و`spin_lock_irqsave()` اللي موجودين حرفياً في الـ `DEFINE_LOCK_GUARD_1` بالملف
- **الفصل 9** — An Introduction to Kernel Synchronization: متى تستخدم mutex ومتى spinlock — سبب الـ `union` في `gpio_shared_desc`
- **الفصل 17** — Devices and Modules: الـ device model وعلاقته بالـ `gpio_device`

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 16** — Kernel Debugging Techniques: بيشمل الـ `lockdep` اللي بتستخدمه `gpio_shared_lockdep_assert()`
- **الفصل 14** — Porting Linux to a Custom Platform: GPIO setup في embedded boards

---

### مصادر إضافية للـ APIs المستخدمة في الملف

الـ `gpiolib-shared.h` بيستخدم ثلاث آليات kernel متقدمة — كل واحدة عندها مصادرها:

#### DEFINE_LOCK_GUARD_1 / linux/cleanup.h
- [LWN: Scope-based resource management in the kernel](https://lwn.net/Articles/934679/)
- [kernel source: include/linux/cleanup.h](https://github.com/torvalds/linux/blob/master/include/linux/cleanup.h)
- [elixir: DEFINE_LOCK_GUARD_1 usages](https://elixir.bootlin.com/linux/latest/A/ident/DEFINE_LOCK_GUARD_1)

#### lockdep / lockdep_assert_held
- [LWN: The kernel lock validator](https://lwn.net/Articles/185666/)
- [Documentation/locking/lockdep-design.rst](https://www.kernel.org/doc/html/latest/locking/lockdep-design.html)

#### spin_lock_irqsave vs mutex — متى تُختار كل واحدة
- [LWN: Unreliable Guide to Locking](https://www.kernel.org/doc/html/latest/kernel-hacking/locking.html)
- الـ `can_sleep` flag في `gpio_shared_desc` هي اللي بتحدد الاختيار وقت الـ runtime

---

### مصطلحات البحث

```
# البحث عن الـ shared GPIO implementation
"gpio_shared_desc" site:github.com
"CONFIG_GPIO_SHARED" linux kernel
"gpio_device_setup_shared" Bartosz Golaszewski
"gpio_shared_add_proxy_lookup" kernel patch

# الـ descriptor interface عموماً
"gpiod_get" "devm_gpiod_get" linux kernel driver
"gpiolib" site:lore.kernel.org

# الـ locking patterns المستخدمة
"DEFINE_LOCK_GUARD_1" linux kernel cleanup.h
"spin_lock_irqsave" "mutex_lock" gpio kernel

# تصفح الكود مباشرة على Elixir
# https://elixir.bootlin.com/linux/latest/source/drivers/gpio/gpiolib-shared.h
# https://elixir.bootlin.com/linux/latest/A/ident/gpio_shared_desc
# https://elixir.bootlin.com/linux/latest/A/ident/devm_gpiod_shared_get

# الـ mailing list archive
linux-gpio@vger.kernel.org lore.kernel.org
```
## Phase 8: Writing simple module

### الفكرة

**`devm_gpiod_shared_get()`** هي الدالة الوحيدة المُصدَّرة بـ `EXPORT_SYMBOL_GPL` في الملف. بتتنفذ في كل مرة driver بيطلب descriptor لـ GPIO مشترك. هنستخدم **kprobe** نعمل hook عليها، ونطبع اسم الـ device اللي طلب الـ GPIO مع عنوان الـ `gpio_shared_desc` اللي اترجع.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_shared_gpio.c
 *
 * Hooks devm_gpiod_shared_get() to log every shared GPIO acquisition.
 * Works on kernels with CONFIG_GPIO_SHARED=y and CONFIG_KPROBES=y.
 */

/* -------- includes -------- */
#include <linux/module.h>       /* MODULE_*, module_init/exit             */
#include <linux/kprobes.h>      /* kprobe, register_kprobe, etc.          */
#include <linux/printk.h>       /* pr_info, pr_warn                       */
#include <linux/device.h>       /* struct device, dev_name()              */
#include <linux/errno.h>        /* error codes                            */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example <example@example.com>");
MODULE_DESCRIPTION("kprobe on devm_gpiod_shared_get — logs shared GPIO acquisition");

/* ------------------------------------------------------------------ */
/*  kprobe pre-handler: runs just before devm_gpiod_shared_get()      */
/* ------------------------------------------------------------------ */

/*
 * devm_gpiod_shared_get() prototype:
 *   struct gpio_shared_desc *devm_gpiod_shared_get(struct device *dev);
 *
 * On x86-64:  first argument is in RDI  → regs->di
 * On ARM64:   first argument is in X0   → regs->regs[0]
 * We use pt_regs helpers provided by kprobes to stay arch-agnostic.
 */
static int pre_handler(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * استخراج الـ argument الأول (struct device *dev).
     * الـ macro ده بيشتغل على x86-64 و ARM64 و RISC-V.
     */
#if defined(CONFIG_X86_64)
    struct device *dev = (struct device *)regs->di;
#elif defined(CONFIG_ARM64)
    struct device *dev = (struct device *)regs->regs[0];
#elif defined(CONFIG_RISCV)
    struct device *dev = (struct device *)regs->a0;
#else
    struct device *dev = NULL;   /* architecture not handled */
#endif

    if (!dev) {
        pr_warn("kprobe_shared_gpio: pre_handler: dev is NULL or arch unsupported\n");
        return 0;
    }

    /*
     * dev_name() بترجع اسم الـ device اللي بيطلب الـ shared GPIO descriptor.
     * ده بيساعد نعرف أنهي proxy auxiliary device بيحاول يحصل على الـ GPIO.
     */
    pr_info("kprobe_shared_gpio: devm_gpiod_shared_get() called by device='%s'\n",
            dev_name(dev));

    return 0; /* 0 = continue executing the original function */
}

/* ------------------------------------------------------------------ */
/*  kprobe post-handler: runs after the function returns              */
/* ------------------------------------------------------------------ */

/*
 * في الـ post_handler الـ function خلصت وراجعت.
 * بنطبع برسالة بسيطة إن الـ call اتكملت — في production ممكن تشيل ده.
 */
static void post_handler(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * الـ return value على x86-64 في RAX، على ARM64 في X0.
     * بنطبعه كعنوان 64-bit لو عايزين نتابع الـ descriptor اللي اترجع.
     */
#if defined(CONFIG_X86_64)
    unsigned long retval = regs->ax;
#elif defined(CONFIG_ARM64)
    unsigned long retval = regs->regs[0];
#elif defined(CONFIG_RISCV)
    unsigned long retval = regs->a0;
#else
    unsigned long retval = 0;
#endif

    if (IS_ERR_VALUE(retval))
        pr_info("kprobe_shared_gpio: devm_gpiod_shared_get() returned error %ld\n",
                (long)retval);
    else
        pr_info("kprobe_shared_gpio: devm_gpiod_shared_get() returned descriptor @ 0x%lx\n",
                retval);
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                  */
/* ------------------------------------------------------------------ */

static struct kprobe kp = {
    /*
     * الاسم الـ symbol بالظبط زي ما هو في /proc/kallsyms.
     * الـ kprobe framework بيحول الاسم لـ address وقت الـ registration.
     */
    .symbol_name = "devm_gpiod_shared_get",
    .pre_handler = pre_handler,
    .post_handler = post_handler,
};

/* ------------------------------------------------------------------ */
/*  module_init / module_exit                                          */
/* ------------------------------------------------------------------ */

static int __init kprobe_shared_gpio_init(void)
{
    int ret;

    /*
     * register_kprobe() بيحجز breakpoint على الـ symbol وبيربط الـ handlers.
     * لو فشل — kernel إما مش فيه CONFIG_KPROBES أو الـ symbol مش موجود.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_warn("kprobe_shared_gpio: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_shared_gpio: planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit kprobe_shared_gpio_exit(void)
{
    /*
     * unregister_kprobe() ضروري في الـ exit عشان نشيل الـ breakpoint
     * من الـ kernel text قبل ما الـ module يتـunload.
     * لو منعملوش → kernel crash لما الـ probe callback يتنفذ بعد unload.
     */
    unregister_kprobe(&kp);
    pr_info("kprobe_shared_gpio: kprobe removed from %s\n", kp.symbol_name);
}

module_init(kprobe_shared_gpio_init);
module_exit(kprobe_shared_gpio_exit);
```

---

### Makefile

```makefile
# Minimal out-of-tree Makefile
obj-m += kprobe_shared_gpio.o

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
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/printk.h` | `pr_info`, `pr_warn` |
| `linux/device.h` | `struct device`, `dev_name()` |
| `linux/errno.h` | `IS_ERR_VALUE()` لفحص الـ return value |

الـ includes دي بتجيب كل اللي المحول محتاجه من غير ما نجيب headers زيادة ترفع وقت الـ compilation.

---

#### الـ pre_handler

**الـ pre_handler** بيتنفذ مباشرةً قبل دخول `devm_gpiod_shared_get()`. بيقرأ الـ argument الأول (`struct device *dev`) من الـ registers بشكل مباشر لأن الـ calling convention بتختلف من architecture لأخرى — x86-64 بتحط أول argument في `RDI`، و ARM64 في `X0`. الـ return value صفر معناه "كمّل تنفيذ الدالة الأصلية".

---

#### الـ post_handler

**الـ post_handler** بيتنفذ بعد رجوع الدالة. بنستخدمه نشوف إيه الـ `gpio_shared_desc *` اللي اترجع — لو فيه error بنطبعه، لو نجح بنطبع العنوان. ده بيساعد في debugging لو مثلاً الـ probe defer وريجع `-EPROBE_DEFER`.

---

#### الـ kprobe struct

الـ struct بيحدد:
- **`symbol_name`**: اسم الـ function اللي هنضع عليها الـ probe — الـ kernel بيحوله لعنوان عن طريق `kallsyms_lookup_name()`.
- **`pre_handler` / `post_handler`**: الـ callbacks المرتبطة بالـ probe.

الـ field `addr` بيتملي تلقائياً من قِبل `register_kprobe()`.

---

#### الـ module_init

`register_kprobe()` بتكتب **int3** (x86) أو **BRK** (ARM64) في بداية الدالة المستهدفة داخل الـ kernel text. من اللحظة دي كل call لـ `devm_gpiod_shared_get()` بيمر على الـ handlers. لو فشل الـ registration — بيبقى غالباً `CONFIG_KPROBES=n` أو الـ symbol مـ`EXPORT`ش.

---

#### الـ module_exit

`unregister_kprobe()` بتشيل الـ breakpoint وبترجع الـ original bytes. دي خطوة إجبارية — لو شيلت الـ module من غير unregister، أي call لـ `devm_gpiod_shared_get()` بعد كده بتنفذ الـ handler اللي اتشالت من الـ memory → **kernel panic**.

---

### تشغيل وملاحظة الـ output

```bash
# بناء الـ module
make

# تحميله
sudo insmod kprobe_shared_gpio.ko

# متابعة الـ log
sudo dmesg -w | grep kprobe_shared_gpio

# تفريغه
sudo rmmod kprobe_shared_gpio
```

مثال على الـ output المتوقع:

```
kprobe_shared_gpio: planted kprobe at devm_gpiod_shared_get (0xffffffff8xxxxxxx)
kprobe_shared_gpio: devm_gpiod_shared_get() called by device='gpio_shared.proxy.0'
kprobe_shared_gpio: devm_gpiod_shared_get() returned descriptor @ 0xffff888xxxxxxxx0
kprobe_shared_gpio: kprobe removed from devm_gpiod_shared_get
```

**الـ `gpio_shared.proxy.N`** هو اسم الـ auxiliary device اللي اتعمل في `gpio_shared_make_adev()` — كل proxy بيمثل consumer واحد للـ GPIO المشترك.
