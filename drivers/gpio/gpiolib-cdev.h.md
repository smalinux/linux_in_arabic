## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **GPIO UAPI** subsystem — اللي بيتكلم عنه في `MAINTAINERS` تحت اسم "GPIO UAPI"، Maintainer بتاعه هو Bartosz Golaszewski. الـ subsystem ده هو اللي بيوفر الـ **character device interface** للـ userspace عشان يتكلم مع الـ GPIO pins.

---

### الصورة الكبيرة — ليه GPIO أصلاً موجود؟

تخيل إن عندك Raspberry Pi أو أي embedded board. فيه pins صغيرة على الجنب — بعضها بتشغل LED، بعضها بتقرأ button، بعضها بتكلم sensor. الـ pins دي اسمها **GPIO (General Purpose Input/Output)** — يعني pin عام ممكن تعمله input أو output زي ما تحب.

**المشكلة القديمة:**
من زمان، اللي عايز يتحكم في GPIO pin من الـ userspace (برنامجك العادي مش kernel driver) كان لازم يعمل كده:

```
echo 17 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio17/direction
echo 1 > /sys/class/gpio/gpio17/value
```

الـ sysfs interface دي كانت **كارثة**: بطيئة، مش atomic، مش بتقدر تتحكم في أكتر من pin في نفس الوقت، ومفيش أي أمان لو برنامجين اتفقوا على نفس الـ pin.

**الحل الجديد — الـ Character Device:**
الـ kernel عمل interface جديد: كل GPIO chip بتظهر كـ `/dev/gpiochipN` — زي `/dev/gpiochip0`. الـ userspace بيفتح الملف ده، وبيبعت `ioctl()` calls عشان يطلب pins، يقرأ، يكتب، أو حتى يستنى على events (زي rising/falling edge).

---

### الـ `gpiolib-cdev.h` — إيه دوره؟

الـ file ده بسيط جداً — هو مجرد **internal header** داخل الـ `drivers/gpio/` directory. بيعلن عن **فانكشنين بس**:

```c
/* Register a GPIO chip as a character device under /dev/gpiochipN */
int gpiolib_cdev_register(struct gpio_device *gdev, dev_t devt);

/* Remove the character device when the chip is removed */
void gpiolib_cdev_unregister(struct gpio_device *gdev);
```

الـ header ده هو **الواجهة الداخلية** بين جزئين في الـ GPIO subsystem:

- **`gpiolib.c`** — اللي بيسجّل GPIO chips وبيديرها
- **`gpiolib-cdev.c`** — اللي بينفذ الـ `/dev/gpiochipN` character device

يعني `gpiolib.c` لما بيجي يسجل chip جديدة، بيقول لـ `gpiolib-cdev.c`: "اعمل device entry في `/dev/`" عن طريق `gpiolib_cdev_register()`.

---

### القصة كاملة — الـ Journey من الـ Hardware للـ Userspace

```
Hardware GPIO Chip (e.g. BCM2835 on RPi)
          |
          v
   gpio_chip struct    <-- الـ driver بيملي ده
          |
          v
   gpio_device struct  <-- الـ gpiolib.c بيعمله
          |
     +----+----+
     |         |
     v         v
gpiolib.c   gpiolib-cdev.c
(core)      (character device)
                |
                v
         /dev/gpiochip0     <-- userspace بيشوفه
                |
                v
     ioctl(fd, GPIO_GET_CHIPINFO_IOCTL, ...)
     ioctl(fd, GPIO_V2_GET_LINE_IOCTL, ...)
     read(fd, ...)   <-- لو بتستنى events
```

لما driver بيعمل `gpiochip_add()`:
1. **`gpiolib.c`** بيخلق `gpio_device` وبيعمل `dev_t` (major/minor number)
2. بيكلم `gpiolib_cdev_register()` من الـ `gpiolib-cdev.h`
3. **`gpiolib-cdev.c`** بيعمل `cdev_add()` فـ `/dev/gpiochip0` بتظهر
4. الـ userspace دلوقتي يقدر يفتح `/dev/gpiochip0` ويبعت `ioctl()` calls

لما الـ chip بتتشال (hotplug أو module unload):
1. **`gpiolib.c`** بيكلم `gpiolib_cdev_unregister()`
2. الـ character device بتتمسح من `/dev/`

---

### ليه الـ header ده مهم رغم إنه صغير؟

الـ **separation of concerns** ده مهم جداً:

| الملف | المسؤولية |
|-------|-----------|
| `gpiolib.c` | إدارة الـ GPIO chips والـ descriptors |
| `gpiolib-cdev.c` | الـ character device وكل الـ ioctl handlers |
| `gpiolib-cdev.h` | الـ interface بينهم (فانكشنين بس) |

الـ header ده بيقول: "أنا مش بيعرف التفاصيل، بس عارف إزاي أسجّل وأمسح device" — وده مبدأ الـ **encapsulation**.

---

### الملفات المرتبطة اللي المفروض تعرفها

**Core Files:**

| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpiolib.c` | الـ core — بيدير كل GPIO chips والـ descriptors |
| `drivers/gpio/gpiolib.h` | Internal header — بيعرف `struct gpio_device`, `struct gpio_desc` |
| `drivers/gpio/gpiolib-cdev.c` | التنفيذ الفعلي للـ character device وكل الـ ioctl handlers |
| `drivers/gpio/gpiolib-cdev.h` | الملف ده — الـ interface بين `gpiolib.c` و `gpiolib-cdev.c` |

**UAPI (Userspace API):**

| الملف | الدور |
|-------|-------|
| `include/uapi/linux/gpio.h` | الـ structs والـ ioctl definitions اللي الـ userspace بيستخدمها |

**Supporting Files:**

| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpiolib-sysfs.c` | الـ legacy sysfs interface القديم |
| `drivers/gpio/gpiolib-of.c` | دعم الـ Device Tree لاكتشاف GPIO chips |
| `drivers/gpio/gpiolib-acpi-core.c` | دعم الـ ACPI (للـ x86) |
| `drivers/gpio/gpiolib-devres.c` | الـ devres helpers للـ managed resources |
| `include/linux/gpio/driver.h` | الـ API اللي GPIO chip drivers بتستخدمه |

**Tools:**

| الملف | الدور |
|-------|-------|
| `tools/gpio/` | userspace tools زي `gpioinfo`, `gpioset`, `gpioget` |
## Phase 2: شرح الـ GPIO Subsystem Framework

### المشكلة — ليه الـ GPIO Subsystem موجود أصلاً؟

في أي SoC زي Raspberry Pi أو STM32 أو i.MX8، في المئات من الـ GPIO lines. المشكلة مش في الـ GPIO نفسها — المشكلة في الـ **chaos** اللي كان موجود قبل الـ gpiolib:

- كل driver بيتكلم مع الـ hardware بشكل مختلف
- الـ userspace مكانش يقدر يتحكم في الـ GPIO إلا عن طريق `/sys/class/gpio` — واجهة قديمة وضعيفة
- مفيش abstraction موحد — لو شركة غيّرت الـ SoC، كل الـ drivers لازم تتغير
- مفيش طريقة لـ request/release الـ GPIO lines بأمان — أي driver يقدر يـ"يسرق" line من driver تاني

الـ **gpiolib** جه عشان يحل ده. هو الـ framework الجوهري اللي بيوفر abstraction موحد للـ GPIO في الـ Linux kernel.

الملف اللي بنشتغل عليه (`gpiolib-cdev.h`) هو جزء صغير لكن مهم جداً من الصورة الكبيرة — هو الـ interface بين الـ gpiolib وبين الـ **character device** اللي بيسمح لـ userspace إنه يتكلم مع الـ GPIO.

---

### الحل — الـ Kernel بيعمل إيه؟

الـ kernel قسّم الموضوع لطبقات واضحة:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Userspace                                  │
│          /dev/gpiochip0   /dev/gpiochip1  ...                   │
└──────────────────────────┬──────────────────────────────────────┘
                           │  ioctl() / read() / poll()
┌──────────────────────────▼──────────────────────────────────────┐
│              GPIO Character Device Layer                        │
│         gpiolib-cdev.c   ←   gpiolib-cdev.h (واجهتنا)          │
│   gpiolib_cdev_register()     gpiolib_cdev_unregister()         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                   gpiolib Core                                  │
│            gpio_device    gpio_desc    gpio_array               │
│         gpiod_get()   gpiod_set_value()   gpiod_direction_*()   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│              GPIO Chip Driver (Provider)                        │
│                    struct gpio_chip                             │
│         .get()   .set()   .direction_input()   .to_irq()        │
│                                                                 │
│   [pl061-gpio]  [gpio-mxc]  [gpio-pca953x]  [gpio-ftdi-elan]   │
└──────────────────────────┬──────────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────────┐
│                     Hardware                                    │
│          SoC GPIO Controller / I2C Expander / SPI Expander      │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Real-World Analogy — فندق الـ GPIO

تخيل **فندق كبير** فيه مئات الأوض:

| عنصر الفندق | المقابل في الـ GPIO Subsystem |
|---|---|
| الفندق نفسه | `struct gpio_device` — الـ container الكاملة |
| رقم الأوضة | رقم الـ GPIO line (0 .. ngpio-1) |
| كل أوضة | `struct gpio_desc` — descriptor لـ GPIO واحدة |
| الريسبشن | gpiolib core — بيمنع حجز نفس الأوضة مرتين |
| الإنتركم بين النزيل والمدير | `/dev/gpiochipN` — الـ cdev interface |
| طاقم التشغيل (صيانة، تنظيف) | `struct gpio_chip` callbacks — الـ hardware-specific ops |
| شركة إدارة الفنادق | الـ gpiolib framework نفسه |
| نزيل بيحجز أوضة | driver بيعمل `gpiod_request()` |
| نزيل خارجي بيتصل بالهاتف | userspace بيفتح `/dev/gpiochip0` ويعمل `ioctl()` |

لما نزيل بيطلب أوضة، الريسبشن بيتأكد إنها مش محجوزة (mutex)، وبيسجّل اسمه كـ consumer في الـ label. لو النزيل سافر (driver unloaded) والأوضة لسه مستخدمة من زبون تاني (userspace)، الفندق بيفضل شغّال — ده بالظبط اللي بيعمله `gpio_device` كـ reference-counted container.

---

### الـ Core Abstractions — إيه هي الـ structs الأساسية؟

#### 1. `struct gpio_chip` — الـ Hardware Provider

ده الـ struct اللي كل driver بيـ"يملاه" عشان يقول للـ kernel "أنا عارف أتحكم في الـ hardware ده":

```c
struct gpio_chip {
    const char      *label;       /* اسم الـ chip زي "gpio-pl061" */
    struct gpio_device *gpiodev;  /* pointer للـ wrapper */
    u16             ngpio;        /* عدد الـ GPIO lines */
    int             base;         /* أول رقم GPIO (deprecated) */
    bool            can_sleep;    /* هل الـ callbacks تقدر تنام؟ */

    /* الـ ops اللي الـ driver لازم يـimplementها */
    int  (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)(struct gpio_chip *gc, unsigned int offset);
    int  (*set)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int  (*set_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int  (*set_config)(struct gpio_chip *gc, unsigned int offset, unsigned long config);
    int  (*to_irq)(struct gpio_chip *gc, unsigned int offset);

    struct gpio_irq_chip irq;  /* إذا الـ GPIO chip فيها IRQ support */
};
```

الـ driver بيعمل `gpiochip_add_data()` عشان يسجّل الـ chip.

---

#### 2. `struct gpio_device` — الـ Runtime Container

ده الـ **internal state container** اللي بيعيش حتى لو الـ hardware driver اتشال:

```c
struct gpio_device {
    struct device       dev;       /* Linux device model integration */
    struct cdev         chrdev;    /* character device — /dev/gpiochipN */
    int                 id;        /* رقم الـ chip (N في gpiochipN) */
    struct module       *owner;    /* يمنع unload الـ module وفيه GPIOs active */

    struct gpio_chip __rcu *chip;  /* pointer للـ hw chip — RCU protected */
    struct gpio_desc    *descs;    /* array بحجم ngpio من descriptors */
    unsigned long       *valid_mask; /* bitmask للـ GPIOs الصالحة */
    struct srcu_struct  desc_srcu; /* SRCU لحماية الـ descriptors */

    u16                 ngpio;     /* عدد الـ GPIOs */
    bool                can_sleep;
    const char          *label;

    /* notification system للـ line state changes */
    struct raw_notifier_head  line_state_notifier;
    rwlock_t                  line_state_lock;
    struct workqueue_struct   *line_state_wq;

    /* notification للـ cdev waitqueues لما الـ device بيتشال */
    struct blocking_notifier_head device_notifier;

    struct srcu_struct  srcu;      /* SRCU لحماية الـ chip pointer نفسه */
    struct list_head    list;      /* linked list لكل gpio_devices */
};
```

**ملاحظة مهمة**: الـ `gpio_chip` بيتشال لو الـ module unloaded، لكن `gpio_device` بيفضل موجود طالما في reference واحدة عليه (مثلاً userspace فاتح `/dev/gpiochip0`). ده الـ **separation of lifetime** الأساسي في التصميم.

---

#### 3. `struct gpio_desc` — الـ Per-Line Descriptor

كل GPIO line عندها `gpio_desc` خاص بيها. ده مش الـ hardware — ده الـ **logical representation**:

```c
struct gpio_desc {
    struct gpio_device  *gdev;    /* parent device */
    unsigned long       flags;    /* bitmask من الـ flags */

    struct gpio_desc_label __rcu *label;  /* consumer name (RCU protected) */
    const char          *name;    /* اسم الـ line من DT/ACPI */

    unsigned int        debounce_period_us;  /* للـ cdev edge detection */
};
```

الـ flags بتحدد حالة الـ line:

```
Bit 0  GPIOD_FLAG_REQUESTED     ← الـ line محجوزة
Bit 1  GPIOD_FLAG_IS_OUT        ← output mode
Bit 6  GPIOD_FLAG_ACTIVE_LOW    ← logic inversion
Bit 9  GPIOD_FLAG_USED_AS_IRQ   ← متصلة بـ IRQ
Bit 16 GPIOD_FLAG_EDGE_RISING   ← cdev edge detection (rising)
Bit 17 GPIOD_FLAG_EDGE_FALLING  ← cdev edge detection (falling)
Bit 18 GPIOD_FLAG_EVENT_CLOCK_REALTIME ← timestamps في الـ events
Bit 19 GPIOD_FLAG_EVENT_CLOCK_HTE      ← hardware timestamps
```

---

### كيف الـ Structs بتتربط ببعض — ASCII Diagram

```
gpio_device ──────────────────────────────────────────────┐
│ dev (struct device)                                      │
│ chrdev (struct cdev)  ──────► /dev/gpiochip0             │
│ id = 0                                                   │
│ ngpio = 32                                               │
│                                                          │
│ chip ──────────────────────────────────► gpio_chip       │
│  (RCU pointer,                          │ label          │
│   can become NULL                       │ ngpio = 32     │
│   if driver unloads)                    │ .get()         │
│                                         │ .set()         │
│                                         │ .direction_*() │
│                                         │ .to_irq()      │
│                                         │                │
│ descs ──────────────────────────────────┘                │
│  [0] gpio_desc ─► gdev, flags, label, name               │
│  [1] gpio_desc ─► gdev, flags, label, name               │
│  [2] gpio_desc ─► gdev, flags, label, name               │
│  ...                                                     │
│  [31] gpio_desc ─► gdev, flags, label, name              │
│                                                          │
│ list ────────────────────────────────► gpio_device[1]    │
│                                           │              │
│                                        gpio_device[2]    │
│                                           │              │
│                                          ...             │
└──────────────────────────────────────────────────────────┘
```

---

### الـ `gpiolib-cdev.h` — دوره في الصورة الكبيرة

الملف نفسه بسيط جداً:

```c
/* gpiolib-cdev.h */

int  gpiolib_cdev_register(struct gpio_device *gdev, dev_t devt);
void gpiolib_cdev_unregister(struct gpio_device *gdev);
```

بس وراء هاتين الفانكشنين، في architecture كامل:

- **`gpiolib_cdev_register()`**: بتاخد `gpio_device` وبتعمل `cdev_add()` عشان تربط الـ `chrdev` field بالـ `/dev/gpiochipN`. من النهارده، الـ userspace يقدر يفتح الـ device ويعمل `ioctl()` عليه.
- **`gpiolib_cdev_unregister()`**: بتشيل الـ cdev وبتـ notify أي userspace process فاتح الـ fd إن الـ device راح (`device_notifier`).

الـ cdev layer جوّاه بتـ implement الـ **GPIO UAPI v2** اللي بيسمح بـ:

| Operation | IOCTL |
|---|---|
| اقرأ معلومات الـ chip | `GPIO_GET_CHIPINFO_IOCTL` |
| اقرأ معلومات line واحدة | `GPIO_V2_GET_LINEINFO_IOCTL` |
| اطلب GPIO lines | `GPIO_V2_GET_LINE_IOCTL` |
| اقرأ/اكتب values | `GPIO_V2_LINE_SET_VALUES_IOCTL` |
| اتابع edge events | `read()` / `poll()` على الـ line fd |

---

### الـ Concurrency Model — إزاي الـ Kernel بيحمي الـ Data؟

ده من أصعب الجزئيات — لأن الـ GPIO بتتكلم فيها من:
- Kernel drivers (kernel context)
- Interrupt handlers (atomic context)
- Userspace (process context)

الـ kernel بيستخدم ثلاث mechanisms:

#### أ) SRCU — Sleepable Read-Copy-Update

الـ **SRCU** (يحتاج فهم الـ RCU subsystem الأول — وهو mechanism لـ lock-free reads بيسمح بـ concurrent readers بدون blocking):

```c
struct gpio_device {
    struct gpio_chip __rcu *chip;  /* محمية بـ SRCU */
    struct srcu_struct srcu;       /* SRCU domain للـ chip pointer */
    struct srcu_struct desc_srcu;  /* SRCU domain للـ descriptors */
};
```

الـ `gpio_chip_guard` class بتستخدمها:

```c
/* لما تيجي تقرأ الـ chip pointer بأمان */
scoped_guard(gpio_chip_guard, desc) {
    /* هنا الـ chip pointer صاحي مش NULL */
    gc->set(gc, offset, value);
}
/* بعد الـ scope، الـ SRCU read lock اتحل تلقائياً */
```

#### ب) RCU — للـ descriptor labels

```c
struct gpio_desc {
    struct gpio_desc_label __rcu *label;  /* RCU protected */
};
```

القراءة تحتاج `rcu_dereference()`، والكتابة تحتاج `rcu_assign_pointer()` + `kfree_rcu()`.

#### ج) rwlock — للـ line state notifications

```c
struct gpio_device {
    rwlock_t line_state_lock;  /* يحمي line_state_notifier */
};
```

---

### الـ Pinctrl Integration — علاقة الـ GPIO بالـ Pin Control

الـ **Pinctrl subsystem** (هو الـ framework المسؤول عن تكوين الـ physical pins على الـ SoC — mux, pull-up, drive strength) بيتكامل مع الـ GPIO framework:

```c
struct gpio_device {
#ifdef CONFIG_PINCTRL
    struct list_head pin_ranges;  /* الـ SoC pins اللي الـ GPIO controller بيديرها */
#endif
};
```

لما driver بيعمل `gpiod_request()`:
1. الـ gpiolib core بيطلب من الـ pinctrl إنه يـ"يحجز" الـ pin ويـ configure إنه GPIO
2. بعدين بيعمل `gpio_chip->request()` callback
3. ده بيضمن إن مفيش اتنين subsystems بيتصارعوا على نفس الـ physical pin

---

### الـ IRQ Integration — الـ GPIO كـ Interrupt Source

الـ `gpio_chip` بيحمل `struct gpio_irq_chip` جوّاه (لو `CONFIG_GPIOLIB_IRQCHIP` مفعّل):

```c
struct gpio_irq_chip {
    struct irq_chip     *chip;         /* الـ irq_chip implementation */
    struct irq_domain   *domain;       /* mapping من GPIO hwirq لـ Linux IRQ */
    irq_flow_handler_t  handler;       /* handle_edge_irq أو handle_level_irq */
    unsigned int        *parents;      /* الـ IRQs من الـ parent interrupt controller */
    /* ... */
};
```

يعني الـ GPIO controller نفسه ممكن يكون **interrupt controller** بيديه interrupts للـ parent (زي GIC في ARM). الـ `gpio_chip->to_irq()` callback بيـ translate من GPIO line number لـ Linux IRQ number.

---

### إيه اللي الـ GPIO Subsystem بيعمله vs. اللي بيـ delegate للـ Drivers

| المهمة | المسؤول |
|---|---|
| تسجيل الـ GPIO lines وحمايتها من الـ double-request | gpiolib core |
| تطبيق الـ active-low logic (inversion) | gpiolib core تلقائياً |
| الـ sysfs/cdev userspace interface | gpiolib core |
| الـ device tree / ACPI GPIO lookup | gpiolib core |
| إدارة الـ GPIO numbers namespace | gpiolib core |
| توصيل GPIO بـ pinctrl | gpiolib core |
| فعلياً قراءة/كتابة الـ hardware register | gpio_chip driver |
| تحديد الـ GPIO count وأسماء الـ lines | gpio_chip driver |
| IRQ translation (GPIO offset → parent IRQ) | gpio_chip driver |
| الـ debounce hardware-level | gpio_chip driver (اختياري) |
| hardware timestamp support | gpio_chip driver (اختياري) |

---

### الـ GPIO UAPI — التطور من v1 لـ v2

```
قديم (deprecated):
/sys/class/gpio/export       ← كتابة رقم GPIO
/sys/class/gpio/gpio42/value ← قراءة/كتابة value
/sys/class/gpio/gpio42/direction

جديد (v1 — CONFIG_GPIO_CDEV_V1):
/dev/gpiochipN
ioctl(GPIOHANDLES_MAX lines request)    ← struct gpiohandle_request
ioctl(GPIOHANDLE_GET_LINE_VALUES_IOCTL)

أحدث (v2 — default):
/dev/gpiochipN
ioctl(GPIO_V2_GET_LINE_IOCTL)           ← struct gpio_v2_line_request
read() / poll()                         ← edge events مع timestamps
```

الـ v2 أضاف:
- دعم لـ **edge events** مع hardware timestamps (HTE subsystem)
- دعم لـ **multiple lines** في request واحدة بشكل أكثر مرونة
- **64-bit alignment** لكل الـ uAPI structs (للـ 32/64-bit compatibility)

---

### الخلاصة — الصورة الكاملة

```
Hardware Pins
     │
     ▼
[gpio_chip] ←── driver يـimplementه (pl061, mxc, pca953x, ...)
     │
     │ gpiochip_add_data()
     ▼
[gpio_device] ←── kernel يخلقه تلقائياً
  ├── chrdev ──────────────────────────────► /dev/gpiochipN
  ├── descs[] ─── gpio_desc[0..ngpio-1]
  ├── chip ──────────────────────────────── RCU pointer → gpio_chip
  ├── srcu / desc_srcu ─────────────────── concurrency protection
  ├── line_state_notifier ──────────────── kernel consumers notification
  └── device_notifier ──────────────────── userspace fd notification

Consumer (kernel driver):
  gpiod_get() → gpio_desc → gpio_chip callbacks → hardware

Consumer (userspace):
  open("/dev/gpiochip0") → ioctl() → gpiolib-cdev → gpio_desc → gpio_chip → hardware
```

الـ `gpiolib-cdev.h` هو **الباب الصغير** اللي بيفتح الـ userspace على كل ده — فانكشنتين بس، لكن وراهم architecture متكامل بيوفر safe، concurrent، multi-consumer GPIO access للـ Linux system.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Header File

الـ `gpiolib-cdev.h` بسيط جداً — بيحدد فقط function prototype لـ registration و unregistration:

```c
int  gpiolib_cdev_register(struct gpio_device *gdev, dev_t devt);
void gpiolib_cdev_unregister(struct gpio_device *gdev);
```

كل التعقيد الحقيقي موجود في `gpiolib-cdev.c`. الـ header ده بمثابة "واجهة عامة" للـ GPIO character device subsystem.

---

### 0. الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### `GPIOD_FLAG_*` — بتات الـ `gpio_desc.flags`

| Bit | اسم الـ Flag                      | المعنى                         |
| --- | --------------------------------- | ------------------------------ |
| 0   | `GPIOD_FLAG_REQUESTED`            | الـ line محجوز لـ consumer     |
| 1   | `GPIOD_FLAG_IS_OUT`               | الاتجاه output                 |
| 2   | `GPIOD_FLAG_EXPORT`               | مصدّر لـ userspace             |
| 3   | `GPIOD_FLAG_SYSFS`                | مصدّر عبر `/sys/class/gpio`    |
| 6   | `GPIOD_FLAG_ACTIVE_LOW`           | منطق معكوس (active-low)        |
| 7   | `GPIOD_FLAG_OPEN_DRAIN`           | open-drain output              |
| 8   | `GPIOD_FLAG_OPEN_SOURCE`          | open-source output             |
| 9   | `GPIOD_FLAG_USED_AS_IRQ`          | متصل بـ IRQ                    |
| 10  | `GPIOD_FLAG_IRQ_IS_ENABLED`       | الـ IRQ مفعّل                  |
| 11  | `GPIOD_FLAG_IS_HOGGED`            | محجوز من الـ kernel نفسه       |
| 12  | `GPIOD_FLAG_TRANSITORY`           | ممكن يفقد قيمته في sleep/reset |
| 13  | `GPIOD_FLAG_PULL_UP`              | pull-up مفعّل                  |
| 14  | `GPIOD_FLAG_PULL_DOWN`            | pull-down مفعّل                |
| 15  | `GPIOD_FLAG_BIAS_DISABLE`         | bias معطّل                     |
| 16  | `GPIOD_FLAG_EDGE_RISING`          | cdev يراقب rising edge         |
| 17  | `GPIOD_FLAG_EDGE_FALLING`         | cdev يراقب falling edge        |
| 18  | `GPIOD_FLAG_EVENT_CLOCK_REALTIME` | timestamps من REALTIME clock   |
| 19  | `GPIOD_FLAG_EVENT_CLOCK_HTE`      | timestamps من hardware HTE     |
| 20  | `GPIOD_FLAG_SHARED`               | الـ pin مشترك بين consumers    |
| 21  | `GPIOD_FLAG_SHARED_PROXY`         | proxy virtual لـ pin مشترك     |

#### `GPIO_V2_LINE_FLAG_*` — الـ uAPI flags (v2)

| Flag | الوظيفة |
|------|---------|
| `GPIO_V2_LINE_FLAG_USED` | الـ line مستخدم، مش متاح |
| `GPIO_V2_LINE_FLAG_ACTIVE_LOW` | منطق معكوس |
| `GPIO_V2_LINE_FLAG_INPUT` | اتجاه input |
| `GPIO_V2_LINE_FLAG_OUTPUT` | اتجاه output |
| `GPIO_V2_LINE_FLAG_EDGE_RISING` | كشف rising edge |
| `GPIO_V2_LINE_FLAG_EDGE_FALLING` | كشف falling edge |
| `GPIO_V2_LINE_FLAG_OPEN_DRAIN` | open-drain |
| `GPIO_V2_LINE_FLAG_OPEN_SOURCE` | open-source |
| `GPIO_V2_LINE_FLAG_BIAS_PULL_UP` | pull-up |
| `GPIO_V2_LINE_FLAG_BIAS_PULL_DOWN` | pull-down |
| `GPIO_V2_LINE_FLAG_BIAS_DISABLED` | بدون bias |
| `GPIO_V2_LINE_FLAG_EVENT_CLOCK_REALTIME` | REALTIME timestamp |
| `GPIO_V2_LINE_FLAG_EVENT_CLOCK_HTE` | hardware timestamp |

#### Macro Groups للـ Validation

| Macro | الـ flags اللي بيجمعها |
|-------|----------------------|
| `GPIO_V2_LINE_BIAS_FLAGS` | PULL_UP \| PULL_DOWN \| DISABLED |
| `GPIO_V2_LINE_DIRECTION_FLAGS` | INPUT \| OUTPUT |
| `GPIO_V2_LINE_DRIVE_FLAGS` | OPEN_DRAIN \| OPEN_SOURCE |
| `GPIO_V2_LINE_EDGE_FLAGS` | EDGE_RISING \| EDGE_FALLING |
| `GPIO_V2_LINE_VALID_FLAGS` | كل الـ flags الصحيحة مجمعة |
| `GPIO_V2_LINE_EDGE_DETECTOR_FLAGS` | ACTIVE_LOW \| HTE \| EDGE_* |

#### Config Options

| Option | الأثر |
|--------|-------|
| `CONFIG_GPIO_CDEV_V1` | يفعّل الـ ABI القديمة (v1): `linehandle_state`، `lineevent_state`، ioctls القديمة |
| `CONFIG_HTE` | يفعّل دعم Hardware Timestamp Engine في `struct line` |
| `CONFIG_PINCTRL` | يضيف `pin_ranges` لـ `gpio_device` |
| `CONFIG_COMPAT` | يفعّل 32-bit compat ioctls |
| `CONFIG_PROC_FS` | يفعّل `show_fdinfo` لـ `/proc/PID/fdinfo` |

---

### 1. قائمة الـ Structs المهمة

#### `struct gpio_device` — (في `gpiolib.h`)

الـ **container الرئيسي** للـ GPIO chip في الـ kernel. يعيش حتى بعد إزالة الـ chip إذا كان userspace لسه شايله.

| Field                 | النوع                           | الغرض                                   |
| --------------------- | ------------------------------- | --------------------------------------- |
| `dev`                 | `struct device`                 | الـ device object في sysfs              |
| `chrdev`              | `struct cdev`                   | الـ character device (`/dev/gpiochipN`) |
| `id`                  | `int`                           | الرقم N في اسم الـ device               |
| `owner`               | `struct module *`               | يمنع unload الـ module لو في GPIO نشط   |
| `chip`                | `struct gpio_chip __rcu *`      | pointer للـ chip الفعلي (محمي بـ SRCU)  |
| `descs`               | `struct gpio_desc *`            | array من descriptors (حجمه `ngpio`)     |
| `valid_mask`          | `unsigned long *`               | bitmask للـ lines الصالحة للاستخدام     |
| `desc_srcu`           | `struct srcu_struct`            | يحمي consistency الـ descriptors        |
| `ngpio`               | `u16`                           | عدد الـ GPIO lines                      |
| `can_sleep`           | `bool`                          | الـ chip callbacks ممكن تنام            |
| `label`               | `const char *`                  | اسم وصفي للـ chip                       |
| `line_state_notifier` | `struct raw_notifier_head`      | يُخطر المشتركين بتغيرات حالة الـ lines  |
| `line_state_lock`     | `rwlock_t`                      | يحمي الـ `line_state_notifier`          |
| `line_state_wq`       | `struct workqueue_struct *`     | workqueue للإشعارات في process context  |
| `device_notifier`     | `struct blocking_notifier_head` | يُخطر عند unregister الـ device         |
| `srcu`                | `struct srcu_struct`            | يحمي الـ pointer للـ chip               |

---

#### `struct gpio_desc` — (في `gpiolib.h`)

الـ **descriptor** لسطر GPIO واحد. الـ userspace بيتعامل معاه عبر offset، لكن kernel بيشتغل عليه مباشرة.

| Field | النوع | الغرض |
|-------|-------|-------|
| `gdev` | `struct gpio_device *` | pointer للـ parent device |
| `flags` | `unsigned long` | بتات الحالة (GPIOD_FLAG_*) |
| `label` | `struct gpio_desc_label __rcu *` | اسم الـ consumer (محمي بـ RCU) |
| `name` | `const char *` | اسم الـ line (من الـ DT أو firmware) |
| `hog` | `struct device_node *` | إذا كان الـ line محجوز من DT |
| `debounce_period_us` | `unsigned int` | فترة الـ debounce بالـ microseconds |

---

#### `struct gpio_chardev_data` — الـ open instance للـ `/dev/gpiochipN`

بيتنشأ عند كل `open()` على الـ `/dev/gpiochipN`. بيمثل جلسة userspace كاملة مع الـ chip.

| Field | النوع | الغرض |
|-------|-------|-------|
| `gdev` | `struct gpio_device *` | الـ chip اللي فُتحت عليه |
| `wait` | `wait_queue_head_t` | لـ blocking read على lineinfo events |
| `events` | `KFIFO(gpio_v2_line_info_changed, 32)` | طابور تغييرات حالة الـ lines |
| `lineinfo_changed_nb` | `struct notifier_block` | يستقبل تغييرات حالة الـ lines |
| `device_unregistered_nb` | `struct notifier_block` | يستقبل حدث unregister الـ device |
| `watched_lines` | `unsigned long *` | bitmap للـ lines اللي userspace بيراقبها |
| `watch_abi_version` | `atomic_t` | (v1 only) يمنع خلط v1 و v2 في نفس الـ fd |
| `fp` | `struct file *` | pointer للـ file الحالي (لـ get_file_active) |

---

#### `struct linereq` — طلب lines (v2 API)

بيتنشأ عند `GPIO_V2_GET_LINE_IOCTL`. بيمثل مجموعة GPIO lines محجوزة من userspace عبر الـ API الجديدة.

| Field | النوع | الغرض |
|-------|-------|-------|
| `gdev` | `struct gpio_device *` | الـ chip المالك |
| `label` | `const char *` | اسم الـ consumer |
| `num_lines` | `u32` | عدد الـ lines في الطلب |
| `wait` | `wait_queue_head_t` | لـ blocking read على edge events |
| `device_unregistered_nb` | `struct notifier_block` | يستقبل حدث unregister الـ device |
| `event_buffer_size` | `u32` | حجم الـ KFIFO المخصص |
| `events` | `KFIFO_PTR(gpio_v2_line_event)` | طابور edge events للـ userspace |
| `seqno` | `atomic_t` | رقم تسلسلي للـ events عند `num_lines > 1` |
| `config_mutex` | `struct mutex` | يحمي تعديلات الـ configuration |
| `lines[]` | `struct line` | array مرنة من line states |

---

#### `struct line` — حالة line واحدة ضمن `linereq`

كل عنصر في `linereq.lines[]` هو `struct line`. بيحوي كل info خاصة بسطر GPIO واحد.

| Field | النوع | الغرض |
|-------|-------|-------|
| `desc` | `struct gpio_desc *` | الـ descriptor |
| `req` | `struct linereq *` | back-pointer للـ request الأب |
| `irq` | `unsigned int` | رقم الـ IRQ المخصص لـ edge detection |
| `edflags` | `u64` | الـ edge flags الفعالة (ACTIVE_LOW \| EDGE_* \| HTE) |
| `timestamp_ns` | `u64` | timestamp محفوظ بين hardirq و IRQ thread |
| `req_seqno` | `u32` | الرقم التسلسلي على مستوى الـ request |
| `line_seqno` | `u32` | الرقم التسلسلي على مستوى الـ line |
| `work` | `struct delayed_work` | الـ debounce worker |
| `sw_debounced` | `unsigned int` | هل الـ SW debounce نشط؟ |
| `level` | `unsigned int` | آخر مستوى منطقي بعد debounce |
| `hdesc` | `struct hte_ts_desc` | (HTE) descriptor للـ hardware timestamping |
| `raw_level` | `int` | (HTE) مستوى الـ line وقت الحدث |
| `total_discard_seq` | `u32` | (HTE+debounce) عداد الـ events المهملة |
| `last_seqno` | `u32` | (HTE+debounce) آخر seqno قبل انتهاء الـ debounce |

---

#### `struct linehandle_state` — (v1 API، `CONFIG_GPIO_CDEV_V1`)

بيتنشأ عند `GPIO_GET_LINEHANDLE_IOCTL`. الـ v1 API القديمة للتحكم في GPIO lines.

| Field | النوع | الغرض |
|-------|-------|-------|
| `gdev` | `struct gpio_device *` | الـ chip المالك |
| `label` | `const char *` | اسم الـ consumer |
| `descs[]` | `struct gpio_desc *[GPIOHANDLES_MAX]` | array من الـ descriptors |
| `num_descs` | `u32` | عدد الـ lines |

---

#### `struct lineevent_state` — (v1 API، `CONFIG_GPIO_CDEV_V1`)

بيتنشأ عند `GPIO_GET_LINEEVENT_IOCTL`. بيراقب line واحد فقط لـ edge events في الـ v1 API.

| Field | النوع | الغرض |
|-------|-------|-------|
| `gdev` | `struct gpio_device *` | الـ chip المالك |
| `label` | `const char *` | اسم الـ consumer |
| `desc` | `struct gpio_desc *` | الـ line الواحد |
| `eflags` | `u32` | RISING \| FALLING |
| `irq` | `int` | رقم الـ IRQ |
| `wait` | `wait_queue_head_t` | لـ blocking read |
| `device_unregistered_nb` | `struct notifier_block` | يستقبل حدث unregister |
| `events` | `KFIFO(gpioevent_data, 16)` | طابور ثابت 16 event |
| `timestamp` | `u64` | timestamp بين hardirq و thread |

---

#### `struct lineinfo_changed_ctx` — Context للـ work item

بيتنشأ في `lineinfo_changed_notify()` ويتنفذ في الـ workqueue عشان الـ pinctrl call ممكن تنام.

| Field | النوع | الغرض |
|-------|-------|-------|
| `work` | `struct work_struct` | الـ work item |
| `chg` | `struct gpio_v2_line_info_changed` | بيانات التغيير |
| `gdev` | `struct gpio_device *` | reference count مرفوع حتى يتم emit الحدث |
| `cdev` | `struct gpio_chardev_data *` | الـ session المستهدفة |

---

### 2. مخطط علاقات الـ Structs (ASCII)

```
+------------------------------------------------------------------+
|                      gpio_device                                  |
|  dev (struct device)  <-- /dev/gpiochipN                         |
|  chrdev (struct cdev) <-- file_operations: gpio_fileops           |
|  chip __rcu --------------------------------------------> gpio_chip (HW driver)
|  descs[] ------------------------------------------> gpio_desc[0..ngpio-1]
|  line_state_notifier <-- raw_notifier_head                        |
|  line_state_lock (rwlock_t)                                       |
|  line_state_wq ------------------------------------------> workqueue
|  device_notifier <-- blocking_notifier_head                       |
|  srcu <-- protects chip pointer                                   |
+---------------------------+--------------------------------------+
                            | open()
                            v
+------------------------------------------------------------------+
|                   gpio_chardev_data                               |
|  gdev ------------------------------------------> gpio_device    |
|  wait (wait_queue_head_t)                                         |
|  events KFIFO[32] <-- gpio_v2_line_info_changed                  |
|  lineinfo_changed_nb --> registered on gdev->line_state_notifier |
|  device_unregistered_nb -> registered on gdev->device_notifier   |
|  watched_lines[] (bitmap, size=ngpio)                             |
|  fp --------------------------------------------> struct file    |
+---------------------------+--------------------------------------+
                            | GPIO_V2_GET_LINE_IOCTL
                            v
+------------------------------------------------------------------+
|                       linereq                                     |
|  gdev ------------------------------------------> gpio_device    |
|  label                                                            |
|  num_lines                                                        |
|  wait (wait_queue_head_t)                                         |
|  events KFIFO_PTR <-- gpio_v2_line_event                         |
|  seqno (atomic_t)                                                 |
|  config_mutex                                                     |
|  device_unregistered_nb -> gdev->device_notifier                 |
|  lines[0..num_lines-1] -----------------------------------+       |
+-----------------------------------------------------------+-------+
                                                            |
            +-----------------------------------------------v-------+
            |                    line                               |
            |  desc ----------------------------------> gpio_desc   |
            |  req  ----------------------------------> linereq     |
            |  irq  (IRQ number for edge detect)                   |
            |  edflags (u64: edge/clock/polarity flags)            |
            |  timestamp_ns (hardirq cache)                        |
            |  req_seqno / line_seqno                              |
            |  work (delayed_work: debounce_work_func)             |
            |  sw_debounced / level                                |
            |  hdesc (hte_ts_desc) [CONFIG_HTE]                   |
            +------------------------------------------------------+

gpio_desc:
+--------------------------------------+
|            gpio_desc                  |
|  gdev ----------------> gpio_device  |
|  flags (unsigned long: GPIOD_FLAG_*) |
|  label __rcu --> gpio_desc_label     |
|  name (const char *)                 |
|  debounce_period_us                  |
+--------------------------------------+
```

---

### 3. دورة حياة الـ Structs — Lifecycle Diagrams

#### دورة حياة `gpio_chardev_data` (الـ `/dev/gpiochipN` session)

```
userspace open("/dev/gpiochipN")
        |
        v
gpio_chrdev_open()
  +- kzalloc(gpio_chardev_data)
  +- bitmap_zalloc(watched_lines, ngpio)
  +- init_waitqueue_head(&cdev->wait)
  +- INIT_KFIFO(cdev->events)
  +- gpio_device_get(gdev)          <- refcount++
  +- raw_notifier_chain_register()  <- subscribe to line state changes
  +- blocking_notifier_chain_register() <- subscribe to device unregister
  +- file->private_data = cdev
        |
        |  [in use: ioctl / poll / read]
        |
userspace close(fd)
        |
        v
gpio_chrdev_release()
  +- blocking_notifier_chain_unregister()
  +- raw_notifier_chain_unregister()
  +- bitmap_free(watched_lines)
  +- gpio_device_put(gdev)          <- refcount--
  +- kfree(cdev)
```

---

#### دورة حياة `linereq` (v2 line request)

```
ioctl(GPIO_V2_GET_LINE_IOCTL, &gpio_v2_line_request)
        |
        v
linereq_create()
  +- copy_from_user(ulr)
  +- gpio_v2_line_config_validate()
  +- kvzalloc(linereq + lines[num_lines])
  +- gpio_device_get(gdev)
  +- for each line:
  |    +- INIT_DELAYED_WORK(&line->work, debounce_work_func)
  |    +- gpiod_request_user(desc, label)
  |    +- gpio_v2_line_config_flags_to_desc_flags()
  |    +- gpiod_direction_input/output_nonotify()
  |    +- edge_detector_setup()      <- setup IRQ or HTE
  |    +- gpiod_line_state_notify()  <- CHANGED_REQUESTED
  +- blocking_notifier_chain_register() <- subscribe device_notifier
  +- anon_inode_getfile("gpio-line", &line_fileops, lr)
  +- fd_install(fd, file)
        |
        |  [in use: read events / ioctl get/set values / ioctl set config]
        |
userspace close(line_fd)
        |
        v
linereq_release() -> linereq_free()
  +- blocking_notifier_chain_unregister()
  +- for each line:
  |    +- edge_detector_stop()
  |    |    +- free_irq() (or hte_ts_put)
  |    |    +- cancel_delayed_work_sync()
  |    |    +- clear edflags, sw_debounced
  |    +- gpiod_free(desc)
  +- kfifo_free(&lr->events)
  +- kfree(label)
  +- gpio_device_put(gdev)
  +- kvfree(lr)
```

---

#### دورة حياة `gpiolib_cdev_register` / `gpiolib_cdev_unregister`

```
GPIO driver probe
  +- gpiochip_add_data()
        +- gpiolib_cdev_register(gdev, devt)
             +- cdev_init(&gdev->chrdev, &gpio_fileops)
             +- alloc_ordered_workqueue(line_state_wq)
             +- cdev_device_add()   <- /dev/gpiochipN appears


GPIO driver remove
  +- gpiochip_remove()
        +- gpiolib_cdev_unregister(gdev)
             +- destroy_workqueue(line_state_wq)
             +- cdev_device_del()   <- /dev/gpiochipN disappears
             +- blocking_notifier_call_chain(device_notifier, 0)
                  <- wakes all waiting sessions with EPOLLHUP
```

---

### 4. Call Flow Diagrams

#### Flow: userspace يقرأ قيمة GPIO (v2)

```
userspace:
  ioctl(line_fd, GPIO_V2_LINE_GET_VALUES_IOCTL, &lv)
    |
    v
linereq_ioctl()
  +- guard(srcu)(&lr->gdev->srcu)   <- protects chip pointer
  +- rcu_access_pointer(gdev->chip) <- if chip gone -> -ENODEV
  +- linereq_get_values(lr, ip)
       +- copy_from_user(&lv)
       +- scan lv.mask -> build descs[] array
       +- gpiod_get_array_value_complex()
       |    +- gc->get_multiple() / gc->get()  <- hardware read
       +- [if sw_debounced] debounced_value()  <- cached value
       +- copy_to_user(&lv)
```

---

#### Flow: edge event — من الـ hardware للـ userspace

```
GPIO hardware edge occurs
    |
    v (hardirq context)
edge_irq_handler(irq, line)
  +- line->timestamp_ns = line_event_timestamp()  <- early timestamp
  +- [num_lines>1] line->req_seqno = atomic_inc_return(&lr->seqno)
  +- return IRQ_WAKE_THREAD
    |
    v (threaded IRQ context)
edge_irq_thread(irq, line)
  +- [no timestamp] le.timestamp_ns = line_event_timestamp()
  +- determine le.id: RISING or FALLING
  +- line->line_seqno++
  +- le.seqno = (num_lines==1) ? line_seqno : req_seqno
  +- le.offset = gpiod_hwgpio(line->desc)
  +- linereq_put_event(lr, &le)
       +- scoped_guard(spinlock, &lr->wait.lock)
       +- [kfifo full] kfifo_skip() -> drop oldest
       +- kfifo_in(&lr->events, le, 1)
       +- wake_up_poll(&lr->wait, EPOLLIN)
            |
            v
userspace:
  read(line_fd, buf, sizeof(gpio_v2_line_event))
    +- linereq_read()
         +- wait_event_interruptible_locked(lr->wait, !kfifo_empty)
         +- kfifo_out(&lr->events, &le, 1)
         +- copy_to_user(buf, &le)
```

---

#### Flow: SW debounce

```
GPIO pin bounces
    |
    v (hardirq)
debounce_irq_handler(irq, line)
  +- mod_delayed_work(system_percpu_wq, &line->work,
                      usecs_to_jiffies(debounce_period_us))
    |  [debounce timer expires]
    v (workqueue context)
debounce_work_func(work)
  +- level = gpiod_get_raw_value_cansleep(line->desc)
  +- [level == line->level] -> no change, return
  +- WRITE_ONCE(line->level, level)
  +- [active_low] level = !level
  +- [edge not matching] -> ignore
  +- fill gpio_v2_line_event
  +- linereq_put_event(lr, &le) -> wakes userspace read
```

---

#### Flow: lineinfo watch — userspace يراقب تغيير حالة line

```
ioctl(chip_fd, GPIO_V2_GET_LINEINFO_WATCH_IOCTL, &lineinfo)
  +- lineinfo_get(cdev, ip, watch=true)
       +- test_and_set_bit(offset, cdev->watched_lines)
       +- gpio_desc_to_lineinfo() + copy_to_user()

[later: line state changes - request/release/config]
    |
    v (atomic notifier, possibly inside spinlock)
gpiod_line_state_notify(desc, action)
  +- raw_notifier_call_chain(&gdev->line_state_notifier)
       +- lineinfo_changed_notify(nb, action, desc)
            +- test_bit(offset, cdev->watched_lines)  -> skip if not watched
            +- get_file_active(&cdev->fp)             -> protect fd
            +- kzalloc(lineinfo_changed_ctx, GFP_ATOMIC)
            +- ctx->chg.event_type = action
            +- gpio_desc_to_lineinfo(desc, &ctx->chg.info, atomic=true)
            +- gpio_device_get(gdev)
            +- queue_work(gdev->line_state_wq, &ctx->work)
                  |
                  v (ordered workqueue / process context)
            lineinfo_changed_func(work)
              +- [!USED] pinctrl_gpio_can_use_line() -> maybe set USED
              +- kfifo_in_spinlocked(&cdev->events, &ctx->chg)
              +- wake_up_poll(&cdev->wait, EPOLLIN)
              +- gpio_device_put()
              +- fput(cdev->fp)
              +- kfree(ctx)
                  |
                  v
userspace:
  read(chip_fd) -> lineinfo_watch_read()
    +- copy_to_user(gpio_v2_line_info_changed)
```

---

#### Flow: `gpiolib_cdev_register` و `gpiolib_cdev_unregister`

```
gpiolib_cdev_register(gdev, devt):
  +- cdev_init(&gdev->chrdev, &gpio_fileops)
  +- gdev->dev.devt = MKDEV(MAJOR(devt), gdev->id)
  +- alloc_ordered_workqueue("%s", WQ_HIGHPRI, dev_name) -> line_state_wq
  +- cdev_device_add(&gdev->chrdev, &gdev->dev)  -> /dev/gpiochipN appears
  +- [chip gone by now?] cdev_device_del() + destroy_workqueue -> -ENODEV

gpiolib_cdev_unregister(gdev):
  +- destroy_workqueue(gdev->line_state_wq)
  +- cdev_device_del()
  +- blocking_notifier_call_chain(&gdev->device_notifier, 0, NULL)
       +- -> gpio_device_unregistered_notify() in each open gpio_chardev_data
       |    +- wake_up_poll(&cdev->wait, EPOLLIN | EPOLLERR)
       +- -> linereq_unregistered_notify() in each active linereq
            +- wake_up_poll(&lr->wait, EPOLLIN | EPOLLERR)
```

---

### 5. استراتيجية الـ Locking

#### خريطة الـ Locks

| Lock | النوع | بيحمي إيه | من يمسكه |
|------|-------|-----------|---------|
| `gdev->srcu` | SRCU | `gdev->chip` pointer — يمنع الوصول بعد إزالة الـ chip | كل ioctl، كل read، كل poll |
| `gdev->desc_srcu` | SRCU | consistency الـ `gpio_desc->label` | `gpio_desc_to_lineinfo`، `gpiod_get_label` |
| `gdev->line_state_lock` | `rwlock_t` | الـ `line_state_notifier` chain | writer: register/unregister; reader: notify path |
| `lr->config_mutex` | mutex | تغييرات الـ config (`linereq_set_config`، `linereq_set_values`) | linereq ioctls |
| `lr->wait.lock` | spinlock (مدمج في wait_queue) | الـ `lr->events` KFIFO وقت القراءة والكتابة | `linereq_put_event`، `linereq_read`، `linereq_poll` |
| `cdev->wait.lock` | spinlock | الـ `cdev->events` KFIFO لـ lineinfo events | `lineinfo_changed_func`، `lineinfo_watch_read` |

---

#### قواعد مهمة في الـ Locking

**الـ SRCU بدل RCU العادي:**
بيُستخدم `srcu_struct` وليس `rcu_struct` العادي لأن الـ GPIO chip callbacks ممكن تنام (`can_sleep`)، والـ SRCU يسمح بـ read-side critical sections تنام.

```c
guard(srcu)(&lr->gdev->srcu);
/* access gdev->chip safely here */
```

**الـ Notifier chains وترتيب الـ Locks:**
- الـ `line_state_notifier` هو `raw_notifier_head` — بيتنادى من atomic context (spinlock مسكود)، لذلك `lineinfo_changed_notify` بيعمل `kzalloc(GFP_ATOMIC)` ويؤجل أي sleeping call (`pinctrl_gpio_can_use_line`) للـ workqueue.
- الـ `device_notifier` هو `blocking_notifier_head` — بيتنادى من process context فقط (في `gpiolib_cdev_unregister`).

**لا يجوز مسك `config_mutex` وبعدين `wait.lock`:**
الـ `linereq_set_values` بيمسك `config_mutex` أولاً، بعدين جوا `gpiod_set_array_value_complex` ممكن يوصل لـ chip callbacks. الـ `wait.lock` (spinlock) بيتمسك بشكل مستقل في `linereq_put_event` و`linereq_read` — ولا يتمسك داخل `config_mutex`.

**الـ `edflags` و`sw_debounced` و`level`:**
بيتُقرأ بـ `READ_ONCE` / بيتُكتب بـ `WRITE_ONCE` بدون lock، لأن:
- الـ setters (`linereq_set_config`، `edge_detector_stop`) متبادلة التأثير (mutually exclusive) عبر `config_mutex`
- الـ readers (`edge_irq_thread`، `debounce_work_func`) يقدروا يعيشوا مع قيمة قديمة قليلاً (slightly stale)

**ترتيب الـ Lock Ordering (من الأعلى للأدنى):**

```
gdev->srcu (SRCU)
  +- gdev->line_state_lock (rwlock) [only for register/unregister notifiers]
       +- lr->config_mutex (mutex)
            +- lr->wait.lock (spinlock) [held for very short duration]
```

لا يجوز عكس هذا الترتيب أبداً لتفادي deadlock.
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ Functions والـ APIs — Cheatsheet

#### Group 1: Registration / Cleanup (Public API)

| Function | Signature | الغرض |
|---|---|---|
| `gpiolib_cdev_register` | `int (struct gpio_device*, dev_t)` | تسجيل الـ chardev لشريحة GPIO |
| `gpiolib_cdev_unregister` | `void (struct gpio_device*)` | إزالة الـ chardev |

#### Group 2: Chip-level chardev Operations

| Function | الغرض |
|---|---|
| `gpio_chrdev_open` | فتح `/dev/gpiochipN` من الـ userspace |
| `gpio_chrdev_release` | إغلاق الـ fd وتنظيف الـ state |
| `gpio_ioctl` | dispatcher للـ ioctl على مستوى الشريحة |
| `chipinfo_get` | إرجاع `gpiochip_info` للـ userspace |
| `lineinfo_get` | قراءة معلومات line واحدة (v2 ABI) |
| `lineinfo_get_v1` | قراءة معلومات line واحدة (v1 ABI legacy) |
| `lineinfo_unwatch` | إيقاف مراقبة line معينة |
| `lineinfo_watch_poll` | `poll()` على أحداث تغيير حالة الـ lines |
| `lineinfo_watch_read` | `read()` لأحداث تغيير حالة الـ lines |

#### Group 3: Line Request (v2 API — `linereq`)

| Function | الغرض |
|---|---|
| `linereq_create` | إنشاء line request جديد (v2) |
| `linereq_free` | تحرير كل موارد الـ linereq |
| `linereq_release` | release callback للـ file |
| `linereq_ioctl` | dispatcher ioctl لملف الـ line |
| `linereq_get_values` | قراءة قيم الـ lines |
| `linereq_set_values` | كتابة قيم الـ lines |
| `linereq_set_config` | إعادة تهيئة الـ lines وقت التشغيل |
| `linereq_poll` | poll للانتظار على أحداث الـ edges |
| `linereq_read` | قراءة أحداث الـ edges من الـ KFIFO |
| `linereq_unregistered_notify` | إشعار عند حذف الشريحة |
| `linereq_show_fdinfo` | عرض معلومات الـ fd في `/proc/PID/fdinfo` |

#### Group 4: Line Handle (v1 Legacy — `linehandle`)

| Function | الغرض |
|---|---|
| `linehandle_create` | إنشاء handle لمجموعة lines (v1) |
| `linehandle_free` | تحرير موارد الـ handle |
| `linehandle_release` | release callback |
| `linehandle_ioctl` | GET/SET values + SET config |
| `linehandle_set_config` | إعادة تهيئة الـ lines |
| `linehandle_validate_flags` | التحقق من صحة الـ flags |
| `linehandle_flags_to_desc_flags` | تحويل uAPI flags لـ desc flags |

#### Group 5: Line Event (v1 Legacy — `lineevent`)

| Function | الغرض |
|---|---|
| `lineevent_create` | إنشاء event monitor على line واحدة (v1) |
| `lineevent_free` | تحرير الموارد |
| `lineevent_release` | release callback |
| `lineevent_ioctl` | GET_LINE_VALUES فقط |
| `lineevent_poll` | poll للانتظار على أحداث الـ IRQ |
| `lineevent_read` | قراءة `gpioevent_data` من الـ KFIFO |
| `lineevent_irq_handler` | hardirq handler — يحفظ الـ timestamp |
| `lineevent_irq_thread` | threaded IRQ — يبني الـ event |

#### Group 6: Edge Detection & Debounce

| Function | الغرض |
|---|---|
| `edge_detector_setup` | تهيئة الـ IRQ أو HTE لكشف الـ edge |
| `edge_detector_update` | تحديث إعدادات الـ edge detector |
| `edge_detector_stop` | إيقاف كل آليات الكشف |
| `edge_detector_fifo_init` | تهيئة الـ KFIFO للأحداث |
| `edge_irq_handler` | hardirq: يحفظ الـ timestamp |
| `edge_irq_thread` | threaded: يبني ويدفع الـ event |
| `debounce_setup` | تهيئة الـ SW/HW debounce |
| `debounce_work_func` | workqueue handler للـ SW debounce |
| `debounce_irq_handler` | hardirq handler لبدء الـ debounce timer |
| `debounced_value` | قراءة القيمة المنقحة للـ line |
| `hte_edge_setup` | تهيئة الـ Hardware Timestamp Engine |
| `process_hw_ts` | HTE callback في hardirq context |
| `process_hw_ts_thread` | HTE callback في thread context |

#### Group 7: Helpers / Converters

| Function | الغرض |
|---|---|
| `gpio_desc_to_lineinfo` | تحويل `gpio_desc` لـ `gpio_v2_line_info` |
| `gpio_v2_line_config_flags` | استخراج flags لـ line معينة من الـ config |
| `gpio_v2_line_config_output_value` | قراءة القيمة الافتراضية للـ output |
| `gpio_v2_line_config_debounced` | هل الـ line فيها debounce؟ |
| `gpio_v2_line_config_debounce_period` | استخراج فترة الـ debounce |
| `gpio_v2_line_flags_validate` | التحقق من صحة الـ flags (v2) |
| `gpio_v2_line_config_validate` | التحقق من صحة الـ config كاملة |
| `gpio_v2_line_config_flags_to_desc_flags` | تحويل v2 flags لـ desc flags |
| `gpio_v2_line_info_to_v1` | تحويل v2 line info لـ v1 |
| `gpio_v2_line_info_changed_to_v1` | تحويل v2 changed event لـ v1 |
| `line_event_timestamp` | اختيار مصدر الـ timestamp (MONO/RT/HTE) |
| `line_event_id` | تحديد نوع الـ edge event من مستوى الـ line |
| `linereq_put_event` | وضع event في الـ KFIFO مع overflow handling |
| `make_irq_label` | إنشاء string label للـ IRQ |
| `free_irq_label` | تحرير الـ label |
| `lineinfo_changed_notify` | notifier callback عند تغير حالة الـ line |
| `lineinfo_changed_func` | workqueue handler لإرسال الـ notification |
| `gpio_device_unregistered_notify` | wake_up عند حذف الشريحة |

---

### Group 1: Registration & Cleanup

هذا الـ group هو الـ public API الوحيد المُعرَّف في `gpiolib-cdev.h`. الهدف منه تسجيل وإزالة الـ character device الخاص بشريحة GPIO ضمن الـ VFS.

---

#### `gpiolib_cdev_register`

```c
int gpiolib_cdev_register(struct gpio_device *gdev, dev_t devt);
```

**ما تفعله:** تهيئ الـ `cdev` المرتبط بالـ `gpio_device` وتضيفه للـ kernel، ثم تُنشئ `ordered_workqueue` لمعالجة أحداث تغيير حالة الـ lines. تتحقق في النهاية أن الـ `gpio_chip` لا يزال موجوداً قبل الإعلان عن النجاح.

**الـ Parameters:**
- `gdev`: الـ `gpio_device` المُراد تسجيل الـ chardev له.
- `devt`: الـ `dev_t` الأساسي للـ GPIO subsystem (يُستخدم `MAJOR(devt)` مع `gdev->id` كـ minor number).

**الـ Return:** `0` عند النجاح، أو `errno` سالب (`-ENOMEM`, `-ENODEV`).

**Key Details:**
- يستدعي `cdev_device_add()` لتسجيل الـ cdev والـ device دفعة واحدة، مما يضمن ظهور `/dev/gpiochipN` فقط بعد اكتمال الإعداد.
- الـ `line_state_wq` هو `ordered_workqueue` بـ `WQ_HIGHPRI` لضمان الترتيب وعدم التأخير في إرسال إشعارات تغيير الـ line state.
- عند فشل `cdev_device_add()`، يُدمَّر الـ workqueue قبل الرجوع بالخطأ.
- يأخذ SRCU read lock للتحقق من وجود الـ `chip` — إذا اختفى بين `cdev_device_add` والتحقق، يُزال الـ cdev مباشرة.

**Caller:** `gpiolib_dev_init()` أو `gpiochip_add_data_with_key()` ضمن مسار تسجيل الشريحة.

```
gpiolib_cdev_register():
  cdev_init(&gdev->chrdev, &gpio_fileops)
  gdev->dev.devt = MKDEV(MAJOR(devt), gdev->id)
  alloc_ordered_workqueue() → gdev->line_state_wq
  cdev_device_add()         ← /dev/gpiochipN يظهر هنا
  srcu_lock → check gdev->chip still valid
    if !chip → cdev_device_del() + destroy_workqueue() → -ENODEV
  return 0
```

---

#### `gpiolib_cdev_unregister`

```c
void gpiolib_cdev_unregister(struct gpio_device *gdev);
```

**ما تفعله:** تزيل الـ chardev من الـ VFS، تدمر الـ workqueue، ثم تُشعر كل الـ fd المفتوحة أن الشريحة اختفت عبر الـ `device_notifier`.

**الـ Parameters:**
- `gdev`: الـ `gpio_device` المُراد إزالته.

**الـ Return:** لا يرجع قيمة.

**Key Details:**
- `blocking_notifier_call_chain(&gdev->device_notifier, 0, NULL)` يُوقظ كل المستخدمين الذين فتحوا line requests أو lineevent أو الـ chardev نفسه، ليحصلوا على `EPOLLERR` عند الـ `poll()`.
- الترتيب مهم: `destroy_workqueue` أولاً يضمن انتهاء كل الـ pending work قبل إزالة الـ cdev.

**Caller:** مسار حذف الشريحة `gpiochip_remove()`.

---

### Group 2: Chip-level chardev Operations

هذا الـ group يُنفذ الـ `file_operations` للـ `/dev/gpiochipN`. كل `open()` ينشئ `gpio_chardev_data` مستقل يحمل state الجلسة.

---

#### `gpio_chrdev_open`

```c
static int gpio_chrdev_open(struct inode *inode, struct file *file);
```

**ما تفعله:** تُنشئ `gpio_chardev_data` لكل `open()` جديد، وتخصص bitmap للـ watched lines، وتُسجِّل notifier لتغييرات حالة الـ lines ونوتيفايير آخر لحذف الشريحة.

**Key Details:**
- تستخدم `container_of(inode->i_cdev, struct gpio_device, chrdev)` للوصول للـ `gdev`.
- تأخذ SRCU read lock للتحقق من أن الـ chip لا يزال موجوداً — لو الشريحة تُزال أثناء الـ open يرجع `-ENODEV`.
- الـ `watched_lines` bitmap بحجم `gdev->ngpio` bits — كل bit تمثل line تحت المراقبة.
- يُسجَّل `lineinfo_changed_nb` في `line_state_notifier` (raw_notifier, يُستدعى من atomic context).
- يُسجَّل `device_unregistered_nb` في `device_notifier` (blocking_notifier).
- الـ cleanup path كاملة عند أي خطأ — لا resource leaks.

**Return:** `0` عند النجاح، `-ENOMEM` أو `-ENODEV`.

```
gpio_chrdev_open():
  gdev = container_of(inode->i_cdev, gpio_device, chrdev)
  srcu_read_lock → check chip exists
  kzalloc(gpio_chardev_data)
  bitmap_zalloc(gdev->ngpio)          ← watched_lines bitmap
  register lineinfo_changed_nb        ← raw_notifier (atomic-safe)
  register device_unregistered_nb     ← blocking_notifier
  file->private_data = cdev
  nonseekable_open()
```

---

#### `gpio_chrdev_release`

```c
static int gpio_chrdev_release(struct inode *inode, struct file *file);
```

**ما تفعله:** تُزيل كل الـ notifiers المسجلة، تحرر الـ bitmap، تُقلل الـ ref count على الـ `gdev`، وتحرر الـ `cdev` struct.

**Key Details:**
- يجب إزالة `lineinfo_changed_nb` بـ `write_lock_irqsave` لأنه registered في raw_notifier يُستدعى من IRQ context.
- `gpio_device_put(gdev)` يُقلل الـ refcount — قد يُطلق الـ gdev إذا كانت الشريحة مُزالة مسبقاً.

---

#### `gpio_ioctl`

```c
static long gpio_ioctl(struct file *file, unsigned int cmd, unsigned long arg);
```

**ما تفعله:** الـ dispatcher الرئيسي لكل `ioctl()` على `/dev/gpiochipN`. تتحقق من وجود الـ chip ثم توجه الـ command للـ handler المناسب.

**الـ Commands المدعومة:**

| Command | Handler | ABI |
|---|---|---|
| `GPIO_GET_CHIPINFO_IOCTL` | `chipinfo_get()` | v1+v2 |
| `GPIO_GET_LINEHANDLE_IOCTL` | `linehandle_create()` | v1 only |
| `GPIO_GET_LINEEVENT_IOCTL` | `lineevent_create()` | v1 only |
| `GPIO_GET_LINEINFO_IOCTL` | `lineinfo_get_v1(..., false)` | v1 only |
| `GPIO_GET_LINEINFO_WATCH_IOCTL` | `lineinfo_get_v1(..., true)` | v1 only |
| `GPIO_V2_GET_LINEINFO_IOCTL` | `lineinfo_get(..., false)` | v2 |
| `GPIO_V2_GET_LINEINFO_WATCH_IOCTL` | `lineinfo_get(..., true)` | v2 |
| `GPIO_V2_GET_LINE_IOCTL` | `linereq_create()` | v2 |
| `GPIO_GET_LINEINFO_UNWATCH_IOCTL` | `lineinfo_unwatch()` | v1+v2 |

**Key Details:**
- تأخذ SRCU read lock طوال فترة الـ ioctl — أي حذف للشريحة أثناء الـ ioctl سيُؤجَّل حتى انتهاء الـ SRCU grace period.
- `rcu_access_pointer(gdev->chip)` يتحقق من وجود الـ chip بدون أي overhead.

---

#### `chipinfo_get`

```c
static int chipinfo_get(struct gpio_chardev_data *cdev, void __user *ip);
```

**ما تفعله:** ترجع `struct gpiochip_info` للـ userspace تحتوي على اسم الشريحة، الـ label، وعدد الـ lines.

**Key Details:** بسيطة جداً — `strscpy` للأسماء ثم `copy_to_user`. لا locking إضافي مطلوب لأن هذه البيانات static بعد تسجيل الشريحة.

---

#### `lineinfo_get` / `lineinfo_get_v1`

```c
static int lineinfo_get(struct gpio_chardev_data *cdev, void __user *ip, bool watch);
static int lineinfo_get_v1(struct gpio_chardev_data *cdev, void __user *ip, bool watch);
```

**ما تفعلان:** تقرأ معلومات line واحدة بـ offset محدد. إذا كان `watch=true`، تُسجِّل الـ line في الـ `watched_lines` bitmap لتلقي إشعارات التغيير مستقبلاً.

**Key Details:**
- الـ `lineinfo_ensure_abi_version()` تمنع خلط v1 و v2 watch في نفس الجلسة — إذا بدأت بـ v1 watch لا يمكنك الانتقال لـ v2 وبالعكس.
- `test_and_set_bit()` atomic — تمنع مراقبة نفس الـ line مرتين، يرجع `-EBUSY` إذا كانت مُراقَبة مسبقاً.
- عند فشل `copy_to_user()` بعد تسجيل الـ watch، يتم `clear_bit()` لتنظيف الحالة.
- الـ v1 تستدعي `gpio_v2_line_info_to_v1()` للتحويل لصيغة قديمة متوافقة.

---

#### `lineinfo_unwatch`

```c
static int lineinfo_unwatch(struct gpio_chardev_data *cdev, void __user *ip);
```

**ما تفعله:** تُوقف مراقبة line محددة بـ `test_and_clear_bit()` على الـ `watched_lines` bitmap.

**Return:** `-EBUSY` إذا لم تكن الـ line تحت المراقبة أصلاً، `0` عند النجاح.

---

#### `lineinfo_watch_poll` / `lineinfo_watch_read`

```c
static __poll_t lineinfo_watch_poll(struct file *file, struct poll_table_struct *pollt);
static ssize_t lineinfo_watch_read(struct file *file, char __user *buf, size_t count, loff_t *off);
```

**ما تفعلان:** `poll()` و`read()` على أحداث تغيير حالة الـ lines المُراقَبة. الأحداث تُخزَّن في KFIFO بحجم 32 حدث من نوع `gpio_v2_line_info_changed`.

**Key Details:**
- `lineinfo_watch_read` تدعم v1 وv2 في نفس الكود — حجم الـ event يختلف بين النسختين ويُحدَّد حسب `watch_abi_version`.
- blocking read تستخدم `wait_event_interruptible_locked()` داخل الـ spinlock — آمن لأن الـ spinlock يُفرَج عنه أثناء النوم.
- `guard(srcu)` يضمن عدم تحرر الـ `gdev` أثناء الـ read.

---

### Group 3: Line Request — v2 API

الـ `linereq` هو القلب الرئيسي للـ GPIO character device API v2. يدير مجموعة من الـ lines بـ config موحدة مع دعم edge detection وdebounce.

---

#### `linereq_create`

```c
static int linereq_create(struct gpio_device *gdev, void __user *ip);
```

**ما تفعله:** المعالج الرئيسي لـ `GPIO_V2_GET_LINE_IOCTL`. تقرأ `gpio_v2_line_request` من الـ userspace، تتحقق منه، تُخصص `linereq` struct، تطلب كل الـ GPIO descriptors، تُهيئ الـ edge detectors، ثم تُنشئ anonymous inode وترجع الـ fd للـ userspace.

**الـ Parameters:**
- `gdev`: الشريحة المطلوبة.
- `ip`: مؤشر لـ `struct gpio_v2_line_request` في الـ userspace.

**Return:** `0` عند النجاح، أو `errno` سالب.

**Key Details:**
- الـ `linereq` يُخصَّص بـ `kvzalloc(struct_size(lr, lines, num_lines))` — flexible array member للـ `line` structs.
- كل line تحصل على `INIT_DELAYED_WORK(&line->work, debounce_work_func)` حتى لو لا يوجد debounce — جاهز للاستخدام لاحقاً.
- الـ `event_buffer_size` يُحسب: إذا لم يُحدد، يكون `num_lines * 16`؛ بحد أقصى `GPIO_V2_LINES_MAX * 16`.
- يُسجَّل `device_unregistered_nb` لاستقبال إشعار حذف الشريحة.
- الـ fd يُنشأ بـ `get_unused_fd_flags()` + `anon_inode_getfile("gpio-line", ...)` — الـ fd لا يظهر للـ userspace حتى `fd_install()`.
- عند فشل `copy_to_user()` للـ response، يُستدعى `fput()` وليس `linereq_free()` مباشرة — لأن `fput` يُطلق `release` callback الذي يستدعي `linereq_free`.

```
linereq_create():
  copy_from_user(gpio_v2_line_request)
  gpio_v2_line_config_validate()
  kvzalloc(struct_size(lr, lines, num_lines))
  for each line:
    gpio_device_get_desc()
    gpiod_request_user()
    gpio_v2_line_config_flags_to_desc_flags()
    gpiod_direction_output/input_nonotify()
    edge_detector_setup()              ← IRQ/HTE setup
    gpiod_line_state_notify(REQUESTED)
  register device_unregistered_nb
  get_unused_fd_flags()
  anon_inode_getfile("gpio-line", &line_fileops, lr)
  copy_to_user(fd back to user)
  fd_install()
```

---

#### `linereq_free`

```c
static void linereq_free(struct linereq *lr);
```

**ما تفعله:** تُحرر كل موارد الـ linereq: تُزيل الـ notifier، توقف كل الـ edge detectors، تحرر الـ GPIO descriptors، تحرر الـ KFIFO، وتحرر الـ struct.

**Key Details:**
- ترتيب التحرير مهم: `edge_detector_stop()` أولاً قبل `gpiod_free()` لضمان إيقاف الـ IRQs والـ work items.
- تستخدم `kvfree()` وليس `kfree()` لأن الـ `linereq` قد يكون كبيراً (flexible array) وخُصِّص بـ `kvzalloc`.

---

#### `linereq_ioctl`

```c
static long linereq_ioctl(struct file *file, unsigned int cmd, unsigned long arg);
```

**ما تفعله:** dispatcher الـ ioctl لـ fd الـ line request. يدعم 3 commands فقط.

| Command | Handler |
|---|---|
| `GPIO_V2_LINE_GET_VALUES_IOCTL` | `linereq_get_values()` |
| `GPIO_V2_LINE_SET_VALUES_IOCTL` | `linereq_set_values()` |
| `GPIO_V2_LINE_SET_CONFIG_IOCTL` | `linereq_set_config()` |

---

#### `linereq_get_values`

```c
static long linereq_get_values(struct linereq *lr, void __user *ip);
```

**ما تفعله:** تقرأ القيم المنطقية لمجموعة من الـ lines المحددة بـ bitmask. تدعم القراءة من الـ lines المُبرمجة كـ output أيضاً.

**Key Details:**
- الـ userspace يُرسل `gpio_v2_line_values { mask, bits }` — `mask` يحدد أي lines تُقرأ، `bits` يحمل القيم.
- `gpiod_get_array_value_complex()` تتطلب compacted descriptor array — يُبنى ديناميكياً لكل حالة `num_get > 1`.
- لحالة `num_get == 1`: لا allocation — مؤشر مباشر لـ `&lr->lines[i].desc`.
- Lines ذات SW debounce تُقرأ من `debounced_value()` بدلاً من الـ hardware مباشرة.
- لا mutex لأن القراءة تتحمل قيم stale بشكل طبيعي.

---

#### `linereq_set_values`

```c
static long linereq_set_values(struct linereq *lr, void __user *ip);
```

**ما تفعله:** تكتب قيماً على الـ output lines المحددة بـ bitmask.

**Key Details:**
- تأخذ `config_mutex` — لا يمكن تعديل القيم أثناء تغيير الـ config.
- تُبني compacted desc array بنفس منطق `linereq_get_values`.
- تتحقق ضمنياً من كون الـ line output عبر `gpiod_set_array_value_complex()` — إذا كانت input ستفشل على مستوى الـ driver.

---

#### `linereq_set_config`

```c
static long linereq_set_config(struct linereq *lr, void __user *ip);
```

**ما تفعله:** تُعيد تهيئة lines حية وقت التشغيل — تغيير الاتجاه، الـ bias، الـ drive mode، والـ edge detection بدون إعادة فتح الـ fd.

**Key Details:**
- تأخذ `config_mutex` — serializes مع `linereq_set_values` وتكرارات `linereq_set_config` نفسها.
- Lines غير محددة بـ direction flag في الـ config الجديدة تُترك كما هي (`continue`).
- عند التحويل لـ output: `edge_detector_stop()` أولاً ثم `gpiod_direction_output_nonotify()`.
- عند التحويل لـ input: `gpiod_direction_input_nonotify()` ثم `edge_detector_update()`.
- كل تغيير يُطلق `gpiod_line_state_notify(CHANGED_CONFIG)` لتحديث الـ watchers.

---

#### `linereq_poll`

```c
static __poll_t linereq_poll(struct file *file, struct poll_table_struct *wait);
```

**ما تفعله:** تُسجَّل الـ process في `lr->wait` لتُوقَظ عند وصول edge events. تعيد `EPOLLIN | EPOLLRDNORM` إذا كان الـ KFIFO غير فارغ.

**Key Details:**
- تستخدم `kfifo_is_empty_spinlocked_noirqsave()` للتحقق بأمان من حالة الـ KFIFO.
- إذا الشريحة اختفت: `EPOLLHUP | EPOLLERR`.

---

#### `linereq_read`

```c
static ssize_t linereq_read(struct file *file, char __user *buf,
                             size_t count, loff_t *f_ps);
```

**ما تفعله:** تنقل `gpio_v2_line_event` structs من الـ KFIFO للـ userspace. تدعم قراءة متعددة الأحداث في نفس الـ `read()` call.

**Key Details:**
- الـ loop `do { ... } while (count >= bytes_read + sizeof(le))` تقرأ أكبر عدد ممكن من الأحداث في call واحد.
- الـ `wait_event_interruptible_locked()` تنتظر داخل الـ spinlock — آمن لأن الـ spinlock يُفرَج عنه أثناء النوم.
- `O_NONBLOCK` يُرجع `-EAGAIN` فوراً إذا كان الـ KFIFO فارغاً.
- `count < sizeof(le)` يُرجع `-EINVAL` — يجب أن يكون الـ buffer كافياً لحدث واحد على الأقل.

---

#### `linereq_unregistered_notify`

```c
static int linereq_unregistered_notify(struct notifier_block *nb,
                                        unsigned long action, void *data);
```

**ما تفعله:** عند حذف الشريحة، تُوقظ كل العمليات المنتظرة على `lr->wait` بـ `EPOLLIN | EPOLLERR` لتتلقى الخطأ.

---

### Group 4: Line Handle — v1 Legacy API

الـ `linehandle` هو النظير القديم للـ `linereq` — يدير مجموعة lines بدون event support ولا debounce. مُدعوم فقط عند تفعيل `CONFIG_GPIO_CDEV_V1`.

---

#### `linehandle_create`

```c
static int linehandle_create(struct gpio_device *gdev, void __user *ip);
```

**ما تفعله:** معالج `GPIO_GET_LINEHANDLE_IOCTL`. تُنشئ handle لمجموعة lines وترجع anonymous fd. مشابهة لـ `linereq_create` لكن أبسط — لا edge detection ولا KFIFO.

**Key Details:**
- تستخدم `DEFINE_FREE(linehandle_free, ...)` و`__free()` attribute للـ cleanup التلقائي عند الخروج من الـ scope — نمط `cleanup.h` الحديث.
- `FD_PREPARE` macro يُنشئ الـ fd والـ file atomically.
- `retain_and_null_ptr(lh)` يمنع الـ cleanup من تحرير الـ `lh` بعد نقل الملكية للـ file.

---

#### `linehandle_ioctl`

```c
static long linehandle_ioctl(struct file *file, unsigned int cmd, unsigned long arg);
```

**ما تفعله:** يدعم GET_LINE_VALUES، SET_LINE_VALUES، وSET_CONFIG.

**Key Details:**
- `GPIOHANDLE_GET_LINE_VALUES_IOCTL`: يقرأ قيم كل الـ lines دفعة واحدة.
- `GPIOHANDLE_SET_LINE_VALUES_IOCTL`: يتحقق من أن الـ line الأولى output (`GPIOD_FLAG_IS_OUT`) — لأن كل الـ lines تشترك نفس الـ flags.
- `GPIOHANDLE_SET_CONFIG_IOCTL`: يُفوَّض لـ `linehandle_set_config()`.

---

#### `linehandle_set_config`

```c
static long linehandle_set_config(struct linehandle_state *lh, void __user *ip);
```

**ما تفعله:** تُعيد تهيئة كل الـ lines في الـ handle بنفس الـ config. يُطبَّق نفس الـ direction والـ flags على جميع الـ lines.

**Key Details:**
- على عكس الـ v2 API، v1 يُطبِّق نفس الـ config على كل الـ lines — لا per-line configuration.
- يجب تحديد direction صراحة (INPUT أو OUTPUT) — لا "keep as-is".
- يستدعي `gpiod_line_state_notify(CHANGED_CONFIG)` لكل line.

---

#### `linehandle_validate_flags`

```c
static int linehandle_validate_flags(u32 flags);
```

**ما تفعله:** تتحقق من صحة الـ v1 flags — لا تعارضات بين INPUT/OUTPUT، لا OPEN_DRAIN + OPEN_SOURCE معاً، الـ bias يتطلب direction، لا أكثر من bias flag واحد.

---

#### `linehandle_flags_to_desc_flags`

```c
static void linehandle_flags_to_desc_flags(u32 lflags, unsigned long *flagsp);
```

**ما تفعله:** تحوّل v1 uAPI flags (`GPIOHANDLE_REQUEST_*`) لـ descriptor flags (`GPIOD_FLAG_*`) بطريقة atomic.

**Key Details:**
- تستخدم `READ_ONCE` / `WRITE_ONCE` لتجنب data races — الـ flags تُقرأ وتُكتب بشكل atomic.
- `assign_bit()` macro يعين الـ bit بناءً على شرط boolean.

---

### Group 5: Line Event — v1 Legacy API

الـ `lineevent` يدير line واحدة للمراقبة فقط (input + IRQ) — الواجهة القديمة قبل أن يدعم `linereq` الأحداث.

---

#### `lineevent_create`

```c
static int lineevent_create(struct gpio_device *gdev, void __user *ip);
```

**ما تفعله:** معالج `GPIO_GET_LINEEVENT_IOCTL`. تطلب line واحدة كـ input، تُسجَّل threaded IRQ لكشف الـ edges، وترجع anonymous fd.

**Key Details:**
- لا يدعم output lines — يُرجع `-EINVAL` إذا طُلب OUTPUT أو OPEN_DRAIN أو OPEN_SOURCE.
- الـ edge flags تُحسب مع مراعاة ACTIVE_LOW — rising edge بـ active-low تُتحول لـ falling IRQ trigger وبالعكس.
- يُسجَّل notifier للـ device unregistered.
- KFIFO بحجم ثابت 16 حدث — `DECLARE_KFIFO(events, struct gpioevent_data, 16)`.

---

#### `lineevent_irq_handler`

```c
static irqreturn_t lineevent_irq_handler(int irq, void *p);
```

**ما تفعله:** hardirq handler — يحفظ الـ timestamp فقط ويُعيد `IRQ_WAKE_THREAD`.

**Key Details:** التصميم المتعمد — الـ timestamp يُلتقط في hardirq context ليكون أقرب ما يمكن للحدث الفعلي، قبل أي scheduling overhead.

---

#### `lineevent_irq_thread`

```c
static irqreturn_t lineevent_irq_thread(int irq, void *p);
```

**ما تفعله:** الـ threaded handler — يُكمل بناء الـ `gpioevent_data` (يقرأ مستوى الـ line لتحديد rising/falling إذا كان كلاهما مفعلاً) ثم يضعه في الـ KFIFO ويُوقظ الـ waiters.

**Key Details:**
- إذا لم يتوفر timestamp من الـ hardirq handler (حالة nested threaded interrupt)، يأخذ timestamp جديد.
- `kfifo_in_spinlocked_noirqsave()` — يستخدم `wait.lock` كـ spinlock للـ KFIFO.

---

### Group 6: Edge Detection & Debounce

هذا الـ group هو الأكثر تعقيداً — يُنفذ منظومة كاملة لكشف الـ edges مع دعم SW debounce وHTE.

---

#### `edge_detector_setup`

```c
static int edge_detector_setup(struct line *line,
                                struct gpio_v2_line_config *lc,
                                unsigned int line_idx, u64 edflags);
```

**ما تفعله:** تُهيئ آلية كشف الـ edge للـ line — تُقرر إذا كانت HTE أو SW debounce أو threaded IRQ العادي.

**الـ Parameters:**
- `line`: الـ line struct الخاص بهذه الـ line.
- `lc`: الـ line config الكاملة.
- `line_idx`: index الـ line في الـ request.
- `edflags`: edge flags المطلوبة (RISING/FALLING/BOTH + HTE + ACTIVE_LOW).

**Key Details:**
- إذا فيها debounce config تستدعي `debounce_setup()` أولاً — الـ debouncer نفسه يتولى الـ edge detection.
- إذا `sw_debounced` مفعّل بعد `debounce_setup()`: يخرج مبكراً — الـ debounce work func ستولد الأحداث.
- إذا `CONFIG_HTE` مفعّل والـ flag `EVENT_CLOCK_HTE` محدد: يستدعي `hte_edge_setup()`.
- وإلا: `request_threaded_irq()` مع `edge_irq_handler` كـ hardirq و`edge_irq_thread` كـ thread.
- الـ `IRQF_ONESHOT` ضروري للـ threaded IRQ — يمنع re-enabling الـ IRQ قبل انتهاء الـ thread.
- الـ ACTIVE_LOW يُعكس الـ IRQ trigger flags.

```
edge_detector_setup():
  if edge flags set → edge_detector_fifo_init()
  if debounce config → debounce_setup()
  if sw_debounced → return 0   (debouncer handles events)
  if HTE requested → hte_edge_setup()
  else:
    gpiod_to_irq()
    compute irqflags (RISING/FALLING, respect ACTIVE_LOW)
    request_threaded_irq(edge_irq_handler, edge_irq_thread)
    line->irq = irq
```

---

#### `edge_detector_update`

```c
static int edge_detector_update(struct line *line,
                                 struct gpio_v2_line_config *lc,
                                 unsigned int line_idx, u64 edflags);
```

**ما تفعله:** تُحدِّث إعدادات الـ edge detector عند `linereq_set_config()` — إذا لم يتغير شيء لا تفعل شيئاً، وإلا توقف الـ detector القديم وتُنشئ جديداً.

**Key Details:**
- مقارنة `active_edflags == edflags` و`debounce_period_us` — exit مبكر إذا لا تغيير.
- حالة SW debounce مستمرة: فقط تُحدث `debounce_period_us` في الـ desc — الـ work func ستقرأها تلقائياً.

---

#### `edge_detector_stop`

```c
static void edge_detector_stop(struct line *line);
```

**ما تفعله:** تُوقف كل آليات الكشف — تحرر الـ IRQ إذا وجد، تُلغي الـ HTE إذا وجد، تُلغي الـ delayed work، وتُصفّر الـ state.

**Key Details:**
- `free_irq(line->irq, line)` يُرجع الـ label كـ `void*` — يُحرَّر بـ `free_irq_label()` مباشرة.
- `cancel_delayed_work_sync()` يضمن انتهاء أي debounce work pending قبل الإلغاء.
- `line->level` لا يُصفَّر عمداً — يُحافظ على آخر قيمة ليقرأها `debounced_value()` بشكل صحيح حتى بعد التوقف.

---

#### `edge_irq_handler`

```c
static irqreturn_t edge_irq_handler(int irq, void *p);
```

**ما تفعله:** hardirq handler — يحفظ `timestamp_ns` ويُحدّث `req_seqno` للـ multi-line requests، ثم يُعيد `IRQ_WAKE_THREAD`.

**Key Details:**
- `line_event_timestamp()` يختار المصدر: MONOTONIC (default) أو REALTIME أو HTE.
- `atomic_inc_return(&lr->seqno)` لـ multi-line فقط — single line يستخدم `line_seqno` مباشرة لتوفير الـ atomic overhead.

---

#### `edge_irq_thread`

```c
static irqreturn_t edge_irq_thread(int irq, void *p);
```

**ما تفعله:** الـ threaded handler — يقرأ مستوى الـ line (لـ EDGE_BOTH)، يُكمل الـ `gpio_v2_line_event`، ويضعه في الـ KFIFO.

**Key Details:**
- `memset(&le, 0, ...)` إلزامي — يمنع تسريب kernel stack للـ userspace.
- الـ timestamp يأتي من `line->timestamp_ns` (من الـ hardirq) — إذا `0` يأخذ timestamp جديد (nested interrupt).
- `gpiod_get_value_cansleep()` مقبول في thread context.
- `le.seqno = (lr->num_lines == 1) ? le.line_seqno : line->req_seqno` — تحسين للحالة الشائعة.

---

#### `debounce_setup`

```c
static int debounce_setup(struct line *line, unsigned int debounce_period_us);
```

**ما تفعله:** تُحاول أولاً الـ HW debounce عبر `gpio_do_set_config(PIN_CONFIG_INPUT_DEBOUNCE)`. إذا فشل بـ `-ENOTSUPP`، تُهيئ SW debounce بـ IRQ أو HTE.

**Key Details:**
- لا تستدعي `gpiod_set_config()` العادية لتجنب إصدار حدث `CHANGED_CONFIG` مزدوج.
- الـ IRQ للـ SW debounce يستخدم `IRQF_TRIGGER_FALLING | IRQF_TRIGGER_RISING` على كلا الـ edges — الـ work func هي التي تقرر الـ edge الفعلي.
- `WRITE_ONCE(line->sw_debounced, 1)` يُعلم باقي الكود أن الـ debounce SW مفعّل.
- إذا `debounce_period_us == 0`: يُزيل الـ SW debounce فقط (يُرجع بعد الـ HW check المُخفق).

---

#### `debounce_work_func`

```c
static void debounce_work_func(struct work_struct *work);
```

**ما تفعله:** تُقرأ المستوى الفعلي للـ line، تُقارنه بـ `line->level` المحفوظ، وإذا تغير تُحدِّثه وترسل الـ edge event المناسب.

**Key Details:**
- تعمل في workqueue context (يمكن النوم).
- `READ_ONCE(line->level) == level` exit مبكر — bounce يُتجاهل.
- `WRITE_ONCE(line->level, level)` يحفظ المستوى الجديد — atomic لتجنب race مع `debounced_value()`.
- تتعامل مع HTE: تُحسب `diff_seqno` من `last_seqno` و`total_discard_seq` لحساب الأحداث المُهدَرة.
- الـ edge event يُرسَل فقط إذا كان الـ edge المُكتَشَف مطلوباً (RISING أو FALLING أو BOTH).

```
debounce_work_func():
  level = gpiod_get_raw_value_cansleep()  OR  raw_level (HTE)
  if error → return
  if level unchanged → return
  WRITE_ONCE(line->level, level)
  if no edge flags → return          (debounce only, no event)
  apply ACTIVE_LOW inversion
  if unwanted edge → return
  build gpio_v2_line_event
  compute seqno (HTE: use last_seqno - discarded; else: increment)
  linereq_put_event()
```

---

#### `debounce_irq_handler`

```c
static irqreturn_t debounce_irq_handler(int irq, void *p);
```

**ما تفعله:** بسيطة جداً — تُجدِّد الـ delayed work بالفترة المطلوبة (`mod_delayed_work`). هذا هو جوهر الـ SW debounce: كل interrupt يُمدد الانتظار.

**Key Details:**
- `mod_delayed_work(system_percpu_wq, ...)` — يُلغي الـ work الحالي (إذا وجد) ويُجدوله من جديد. فقط عند انتهاء فترة الـ debounce بدون انقطاع ستُنفَّذ `debounce_work_func`.

---

#### `debounced_value`

```c
static bool debounced_value(struct line *line);
```

**ما تفعله:** تُرجع القيمة المنطقية المُنقحة للـ line مع تطبيق ACTIVE_LOW.

**Key Details:**
- `READ_ONCE(line->level)` — يقرأ آخر قيمة سجلتها `debounce_work_func` بدون locking.
- حتى بعد `edge_detector_stop()` تُحافظ على `line->level` — هذا متعمد ليضمن أن `linereq_get_values()` يقرأ قيمة معقولة.

---

#### `linereq_put_event`

```c
static void linereq_put_event(struct linereq *lr, struct gpio_v2_line_event *le);
```

**ما تفعله:** تضع event في الـ KFIFO وتُوقظ الـ waiters. إذا كان الـ KFIFO ممتلئاً، تُسقط أقدم حدث (overflow policy: drop oldest).

**Key Details:**
- `scoped_guard(spinlock, &lr->wait.lock)` — نفس الـ lock المستخدم في الـ read side — يضمن consistency.
- `kfifo_skip()` + `kfifo_in()` عند overflow — FIFO يُبقي أحدث الأحداث.
- `wake_up_poll()` خارج الـ lock لتجنب priority inversion.
- `pr_debug_ratelimited()` عند الـ overflow — لا printk spam.

---

### Group 7: Line Info Helpers & Notification

---

#### `gpio_desc_to_lineinfo`

```c
static void gpio_desc_to_lineinfo(struct gpio_desc *desc,
                                   struct gpio_v2_line_info *info, bool atomic);
```

**ما تفعله:** تحوّل الـ `gpio_desc` الداخلي لـ `gpio_v2_line_info` القابلة للإرسال للـ userspace. تقرأ كل الـ flags والـ attributes.

**الـ Parameters:**
- `atomic`: إذا `true`، لا تستدعي `pinctrl_gpio_can_use_line()` (sleeping) — يُستدعى لاحقاً في workqueue.

**Key Details:**
- تستخدم `CLASS(gpio_chip_guard, guard)` لحماية الوصول للـ chip.
- SRCU read lock عند قراءة الـ label لتجنب race مع `gpiod_free()`.
- حساب `USED` flag ليس atomic بالكامل — ممكن race في حالة نادرة لكن المستخدم يجب على أي حال أن يطلب الـ line ليعرف يقيناً.
- `debounce_period_us` يُضاف كـ attribute إذا غير صفر.

---

#### `lineinfo_changed_notify`

```c
static int lineinfo_changed_notify(struct notifier_block *nb,
                                    unsigned long action, void *data);
```

**ما تفعله:** تُستدعى من الـ `line_state_notifier` (raw_notifier) عند أي تغيير في حالة line. تتحقق إذا كانت الـ line تحت المراقبة، وإذا نعم تُخصص `lineinfo_changed_ctx` وتُجدول work.

**Key Details:**
- تعمل في atomic context (raw_notifier يُستدعى من spinlock) — لذا كل العمل الثقيل مؤجَّل للـ workqueue.
- `GFP_ATOMIC` للـ kzalloc — لا يمكن النوم.
- `get_file_active()` يُزيد refcount على الـ file — يضمن عدم تحرر الـ cdev قبل انتهاء الـ work.
- `gpio_device_get()` على الـ gdev — لنفس السبب.
- `queue_work(gdev->line_state_wq, ...)` — الـ ordered workqueue يضمن الترتيب الصحيح للأحداث.

---

#### `lineinfo_changed_func`

```c
static void lineinfo_changed_func(struct work_struct *work);
```

**ما تفعله:** تُكمل معالجة الـ notification في process context — تستدعي `pinctrl_gpio_can_use_line()` إذا لزم، ثم تضع الحدث في KFIFO الـ `gpio_chardev_data`.

**Key Details:**
- تُحرر `gpio_device_put()` و`fput()` في النهاية — كل الـ references التي أُخذت في `lineinfo_changed_notify`.

---

#### `line_event_timestamp`

```c
static u64 line_event_timestamp(struct line *line);
```

**ما تفعله:** تختار مصدر الـ timestamp بناءً على الـ flags:
- `GPIOD_FLAG_EVENT_CLOCK_REALTIME` → `ktime_get_real_ns()` (CLOCK_REALTIME)
- `GPIOD_FLAG_EVENT_CLOCK_HTE` → `line->timestamp_ns` (من الـ HTE hardware)
- Default → `ktime_get_ns()` (CLOCK_MONOTONIC)

---

#### `gpio_v2_line_flags_validate`

```c
static int gpio_v2_line_flags_validate(u64 flags);
```

**ما تفعله:** تتحقق من كل قيود الـ v2 flags — لا INPUT + OUTPUT معاً، Edge يتطلب INPUT، Drive يتطلب OUTPUT، Bias يتطلب Direction، لا OPEN_DRAIN + OPEN_SOURCE، لا أكثر من clock source، لا HTE بدون `CONFIG_HTE`.

---

#### `gpio_v2_line_config_validate`

```c
static int gpio_v2_line_config_validate(struct gpio_v2_line_config *lc,
                                         unsigned int num_lines);
```

**ما تفعله:** تتحقق من الـ config كاملة — عدد الـ attrs ضمن الحد (`GPIO_V2_LINE_NUM_ATTRS_MAX`)، الـ padding يجب أن يكون صفر (لتوافق مستقبلي)، ثم تتحقق من flags كل line.

---

#### `gpio_v2_line_config_flags_to_desc_flags`

```c
static void gpio_v2_line_config_flags_to_desc_flags(u64 lflags,
                                                     unsigned long *flagsp);
```

**ما تفعله:** تحوّل v2 uAPI flags لـ internal descriptor flags. تستخدم `READ_ONCE`/`WRITE_ONCE` pattern لـ atomicity.

**Key Details:**
- Direction: `set_bit(GPIOD_FLAG_IS_OUT)` أو `clear_bit()` — لا `assign_bit()` لأن الاتجاه exclusive.
- Edge flags، bias، drive، clock source كلها تُحوَّل بـ `assign_bit()`.

---

#### `hte_edge_setup` (CONFIG_HTE)

```c
static int hte_edge_setup(struct line *line, u64 eflags);
```

**ما تفعله:** تُهيئ الـ Hardware Timestamp Engine لالتقاط أحداث الـ GPIO بدقة عالية. تُراعي الـ ACTIVE_LOW عند تحديد الـ HTE edge direction.

**Key Details:**
- `hte_init_line_attr()` يُعرِّف الـ GPIO للـ HTE subsystem.
- `hte_ts_get()` يُخصص الـ HTE descriptor.
- `hte_request_ts_ns(hdesc, process_hw_ts, process_hw_ts_thread, line)` يُسجَّل callback اثنان: primary في IRQ context، secondary في thread context.

---

#### `process_hw_ts` / `process_hw_ts_thread` (CONFIG_HTE)

```c
static enum hte_return process_hw_ts(struct hte_ts_data *ts, void *p);
static enum hte_return process_hw_ts_thread(void *p);
```

**ما تفعلان:** `process_hw_ts` (IRQ context): يحفظ timestamp وraw_level، يتحقق من SW debounce، إذا SW debounce يُجدوِّل work وإلا يُحدث الـ seqno ويُعيد `HTE_RUN_SECOND_CB`. `process_hw_ts_thread` (thread context): يُكمل بناء الـ event ويضعه في الـ KFIFO.

---

### ملاحظات معمارية عامة

```
/dev/gpiochipN
      |
      |-- open()    --> gpio_chardev_data (per-fd state)
      |               |-- watched_lines bitmap
      |               |-- events KFIFO (line_info_changed)
      |               `-- notifiers
      |
      `-- ioctl()
            |-- GPIO_GET_CHIPINFO_IOCTL      --> chipinfo_get
            |-- GPIO_V2_GET_LINE_IOCTL       --> linereq_create
            |       `-- /anon_inode:gpio-line   (linereq fd)
            |               |-- read()   --> edge events KFIFO
            |               |-- poll()   --> wait on events
            |               `-- ioctl()  --> get/set values, set config
            |
            |-- GPIO_GET_LINEHANDLE_IOCTL    --> linehandle_create [v1]
            |       `-- /anon_inode:gpio-linehandle
            |
            |-- GPIO_GET_LINEEVENT_IOCTL     --> lineevent_create [v1]
            |       `-- /anon_inode:gpio-event
            |
            `-- GPIO_V2_GET_LINEINFO_WATCH_IOCTL --> register in watched_lines
```

**الـ Locking Matrix:**

| Resource | Lock |
|---|---|
| `linereq` config/values | `config_mutex` |
| `linereq` KFIFO + waitqueue | `wait.lock` (spinlock) |
| `gpio_chardev_data` KFIFO | `wait.lock` (spinlock) |
| `line_state_notifier` | `line_state_lock` (rwlock) |
| `device_notifier` | blocking_notifier (semaphore) |
| `gdev->chip` pointer | SRCU |
| `desc->flags` | `READ_ONCE`/`WRITE_ONCE` |
## Phase 5: دليل الـ Debugging الشامل

الـ `gpiolib-cdev.h` هو الـ header اللي بيعرّف واجهة الـ character device للـ GPIO subsystem في الـ Linux kernel. بيصدّر function اتنين بس: `gpiolib_cdev_register()` و `gpiolib_cdev_unregister()`، اللي بيشتغلوا على `struct gpio_device`. الـ debugging هنا بيشمل كل الـ stack: من الـ cdev registration لحد الـ ioctl handlers وأحداث الـ GPIO lines.

---

### Software Level

#### 1. debugfs Entries

الـ GPIO subsystem بيكتب معلومات كتير في `/sys/kernel/debug/gpio`:

```bash
# اقرأ حالة كل الـ GPIO chips المسجلة
cat /sys/kernel/debug/gpio

# مثال على الـ output
# gpiochip0: GPIOs 0-31, parent: platform/fe200000.gpio, pinctrl-bcm2835:
#  gpio-2   (                    |sysfs               ) in  hi IRQ
#  gpio-17  (                    |led0                ) out lo
```

| الـ Entry | المعلومة | الأهمية |
|-----------|----------|---------|
| `/sys/kernel/debug/gpio` | حالة كل الـ descriptors | الأساسية لأي debugging |
| `/sys/kernel/debug/pinctrl/<chip>/pinconf-pins` | إعدادات الـ pin مع الـ GPIO | تأكيد الـ pull/drive |
| `/sys/kernel/debug/pinctrl/<chip>/gpio-ranges` | الـ mapping بين GPIO وpin | لو في تداخل مع pinctrl |

```bash
# شوف الـ pinctrl info المرتبطة بالـ gpiochip
ls /sys/kernel/debug/pinctrl/
cat /sys/kernel/debug/pinctrl/pinctrl-handles
```

#### 2. sysfs Entries

```bash
# اعرف كل الـ gpiochips الموجودة
ls /sys/bus/gpio/devices/
# أو
ls /sys/class/gpio/

# معلومات عن chip معين
cat /sys/class/gpio/gpiochip0/label    # اسم الـ chip (gpio_device->label)
cat /sys/class/gpio/gpiochip0/base     # الـ base number
cat /sys/class/gpio/gpiochip0/ngpio    # عدد الـ lines

# تأكيد إن الـ cdev اتسجل صح
ls -la /dev/gpiochip*
# output متوقع:
# crw------- 1 root root 254, 0 Feb 24 10:00 /dev/gpiochip0

# شوف الـ major/minor عشان تتأكد من الـ devt
cat /proc/devices | grep gpio
```

```bash
# لو استخدمت gpio-cdev v2: شوف الـ line info عبر gpioinfo
gpioinfo gpiochip0

# output مثال:
# gpiochip0 - 54 lines:
#   line   0:      unnamed       unused   input  active-high
#   line  17:      unnamed         "led"  output  active-high [used]
```

#### 3. ftrace — Tracepoints والـ Events

```bash
# شوف الـ GPIO events المتاحة
grep gpio /sys/kernel/tracing/available_events

# enable الـ events المهمة
echo 1 > /sys/kernel/tracing/tracing_on
echo 'gpio:*' > /sys/kernel/tracing/set_event

# أو events محددة
echo gpio_direction > /sys/kernel/tracing/set_event
echo gpio_value     >> /sys/kernel/tracing/set_event

# ابدأ الـ trace
echo 1 > /sys/kernel/tracing/tracing_on
# ... شغّل الـ userspace tool اللي بتـ debug
echo 0 > /sys/kernel/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/tracing/trace
```

لـ trace الـ cdev ioctls نفسها:

```bash
# trace الـ function calls داخل gpiolib-cdev
echo 'linereq_create'         > /sys/kernel/tracing/set_ftrace_filter
echo 'linehandle_create'     >> /sys/kernel/tracing/set_ftrace_filter
echo 'gpiolib_cdev_register' >> /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1        > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace_pipe
```

#### 4. printk / Dynamic Debug

الـ `gpiolib-cdev.c` بيستخدم `pr_err` و`dev_err` ومحتاج تفعّل الـ dynamic debug عشان تشوف الـ `dev_dbg` و`pr_debug`:

```bash
# فعّل كل الـ debug messages في gpiolib-cdev
echo 'file gpiolib-cdev.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل الـ gpio subsystem
echo 'module gpio +p'         > /sys/kernel/debug/dynamic_debug/control

# تأكيد إن الـ rules اتضافت
grep gpiolib-cdev /sys/kernel/debug/dynamic_debug/control

# اقرأ الـ messages من الـ kernel ring buffer
dmesg -w | grep -i gpio
journalctl -k -f | grep gpio
```

#### 5. Kernel Config Options للـ Debugging

| Config | الغرض |
|--------|--------|
| `CONFIG_GPIO_CDEV` | تفعيل الـ character device interface (لازم يكون `=y`) |
| `CONFIG_GPIO_CDEV_V1` | تفعيل الـ legacy v1 ioctl API — مهم لو بتـ debug تطبيقات قديمة |
| `CONFIG_DEBUG_GPIO` | يضيف extra validation وـ debug prints في الـ gpiolib |
| `CONFIG_GPIO_SYSFS` | يكشف الـ GPIOs عبر sysfs — مفيد للـ debugging اليدوي |
| `CONFIG_GPIOLIB_IRQCHIP` | لو الـ chip بيدعم interrupts — لازم للـ edge event debugging |
| `CONFIG_DEBUG_LOCK_ALLOC` | يكتشف lock ordering violations في الـ GPIO drivers |
| `CONFIG_PROVE_LOCKING` | يفعّل lockdep — يكتشف deadlocks في الـ GPIO code |
| `CONFIG_DYNAMIC_DEBUG` | يخلي `dev_dbg()` / `pr_debug()` قابلة للتفعيل runtime |
| `CONFIG_KASAN` | يكتشف memory corruption في الـ gpio_device / gpio_desc |
| `CONFIG_KCSAN` | يكتشف data races في الـ concurrent GPIO access |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'GPIO_CDEV|DEBUG_GPIO|GPIO_SYSFS'
# أو
grep -E 'GPIO_CDEV|DEBUG_GPIO' /boot/config-$(uname -r)
```

#### 6. أدوات الـ Subsystem (gpio-tools)

```bash
# عرض كل الـ chips والـ lines
gpiodetect
# output:
# gpiochip0 [pinctrl-bcm2835] (54 lines)

gpioinfo --chip gpiochip0

# قراءة قيمة line
gpioget gpiochip0 17

# كتابة قيمة
gpioset gpiochip0 17=1

# مراقبة الـ edge events (يستخدم الـ cdev v2 ioctls)
gpiomon --num-events=5 --rising-edge gpiochip0 17

# strace على gpiomon/gpioset لرؤية الـ ioctls
strace -e ioctl gpioget gpiochip0 17
# هتشوف GPIO_GET_CHIPINFO_IOCTL, GPIO_V2_GET_LINEINFO_IOCTL, etc.
```

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|-----------------|--------|------|
| `"gpiochip_add_data() failed"` | فشل تسجيل الـ gpiochip | تحقق من `ngpio` والـ base، وتعارض الـ IDs |
| `"GPIO_V2_GET_LINEINFO_IOCTL: line X out of range"` | رقم الـ line أكبر من `ngpio` | راجع `gpio_device->ngpio` |
| `"line X is not available"` | الـ line محجوزة أو في `valid_mask` | افحص `gpio_device->valid_mask`، أو الـ consumer Label في debugfs |
| `"request: invalid flags"` | flags غير مدعومة في الـ ioctl struct | تحقق إن الـ flags بتطابق `GPIOHANDLE_REQUEST_VALID_FLAGS` |
| `"register_chrdev_region() failed"` | تعارض في الـ device numbers | `cat /proc/devices` وتحقق من تكرار الـ major |
| `"cdev_add() failed"` | فشل إضافة الـ cdev | تحقق من `devt` اللي اتمرر لـ `gpiolib_cdev_register()` |
| `"GPIO line X is an interrupt"` | محاولة تغيير direction لـ IRQ line | لا تعدّل IRQ lines كـ output |
| `"nonexclusive access to line X"` | أكتر من process حاجزة نفس الـ line | `grep -r "" /sys/kernel/debug/gpio` لتحديد الـ consumer |
| `"ENOMEM allocating line state"` | نفد الـ memory عند إنشاء `linereq` | `cat /proc/meminfo`، راجع الـ kfifo buffer sizes |
| `"GPIO irq not assigned"` | طلب edge event على line بدون IRQ | تأكد إن الـ gpio_irq_chip صح مهيأ |

#### 8. مواضع `dump_stack()` / `WARN_ON()`

النقاط الاستراتيجية للـ instrumentation في `gpiolib-cdev.c`:

```c
/* في gpiolib_cdev_register() — تحقق إن الـ gdev صالح */
int gpiolib_cdev_register(struct gpio_device *gdev, dev_t devt)
{
    WARN_ON(!gdev);                          /* gdev ده NULL؟ */
    WARN_ON(MAJOR(devt) == 0);               /* devt اتحسب صح؟ */
    /* ... */
}

/* في linereq_create() — تحقق من الـ line count */
if (WARN_ON(ulr->num_lines == 0 || ulr->num_lines > GPIO_V2_LINES_MAX)) {
    dump_stack();   /* شوف مين طلب */
    return ERR_PTR(-EINVAL);
}

/* لو بتـ debug race condition على الـ gpio_device */
WARN_ON(!srcu_read_lock_held(&gdev->desc_srcu));
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

```bash
# قارن قيمة الـ line في debugfs مع القراءة الفعلية
cat /sys/kernel/debug/gpio | grep "gpio-17"
# gpio-17  (led0) out hi

# اقرأ بنفس الـ userspace tool
gpioget gpiochip0 17
# 1  ← لازم يطابق "hi" في debugfs

# لو بيختلفوا → مشكلة في الـ hardware أو الـ driver
```

#### 2. Register Dump Techniques

```bash
# افتح الـ GPIO controller base address من الـ Device Tree أو datasheet
# مثال: BCM2835 GPIO base = 0x3F200000

# باستخدام devmem2
devmem2 0x3F200000 w    # GPFSEL0 — Function Select
devmem2 0x3F200034 w    # GPLEV0  — Pin Level
devmem2 0x3F200004 w    # GPFSEL1

# باستخدام /dev/mem مباشرة (محتاج CONFIG_DEVMEM=y)
dd if=/dev/mem bs=4 count=1 skip=$((0x3F200034 / 4)) 2>/dev/null | xxd

# باستخدام io tool (من package i2c-tools أو standalone)
io -4 0x3F200034

# تحقق من register الـ direction
# GPFSEL: 000 = input, 001 = output لكل 3 bits
# الـ offset يحتاج حساب: pin N في register GPFSEL(N/10) bits [(N%10)*3 +2 : (N%10)*3]
```

```bash
# استخراج الـ base address من الـ DT
cat /proc/device-tree/gpio@7e200000/reg | xxd
# أو
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "gpio@"
```

#### 3. Logic Analyzer / Oscilloscope Tips

```
الـ GPIO character device بيمرّ بالـ layers دي:

userspace ioctl
     │
     ▼
gpiolib_cdev_register() → cdev_add()
     │
     ▼
linereq_create() / linehandle_create()
     │
     ▼
gpiod_direction_output() / gpiod_set_value()
     │
     ▼
gpio_chip->set() / ->direction_output()
     │
     ▼
Physical Pin
```

نصائح عملية:
- **Latency**: قيس الـ delay من `gpioset` call لحد تغيير الـ pin. أي delay > 100µs يدل على مشكلة في الـ scheduling أو الـ driver.
- **Glitches**: لو شايف glitches على الـ pin عند الـ direction change، الـ driver مش بيعمل read-modify-write صح.
- **Trigger**: استخدم edge trigger على الـ logic analyzer وـ timestamp مع الـ ftrace events لمقارنة الـ timing.
- **Pull resistors**: قيس الـ voltage على الـ pin بـ high-impedance multimeter قبل driver initialization — هيكشف الـ default pull state.

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log | التشخيص |
|---------------------|--------------------------|----------|
| Pin مش متوصل أو مقطوع | `gpio-17: value read-back failed` | تحقق من التوصيل الفيزيائي |
| تعارض مع pinctrl | `pin X already requested` | راجع DT: pin مش في gpio-hog وـ pinctrl في نفس الوقت |
| Power domain مش شغال | `gpiochip0: can't enable clock` | `clk_get_rate` على الـ GPIO clock |
| Voltage mismatch (3.3V vs 1.8V) | قيم عشوائية، timeout في الـ edge events | قيس Vio على الـ GPIO bank |
| Level shifter خربان | `gpio: value stuck at 0` | bypass الـ level shifter مؤقتاً للتشخيص |
| IRQ line مش متصلة | `no interrupt for gpio X` في dmesg | `cat /proc/interrupts | grep gpio` |

#### 5. Device Tree Debugging

```bash
# تحقق إن الـ DT node للـ gpiochip موجود وصحيح
ls /proc/device-tree/
# ابحث عن gpio nodes
find /proc/device-tree -name "gpio*" -type d 2>/dev/null

# اقرأ properties أساسية
hexdump -C /proc/device-tree/gpio@7e200000/reg        # base address
cat /proc/device-tree/gpio@7e200000/compatible         # driver binding
hexdump -C /proc/device-tree/gpio@7e200000/gpio-cells  # #gpio-cells

# تحقق من gpio-hog nodes اللي ممكن تحجز lines
find /proc/device-tree -name "gpio-hog" 2>/dev/null

# استخدم dtc لتحويل الـ DT لـ readable format
dtc -I fs -O dts /proc/device-tree 2>/dev/null > /tmp/current.dts
grep -A 20 "gpiochip0\|gpio@" /tmp/current.dts

# تأكيد إن الـ driver ربط نفسه بالـ DT node
ls -la /sys/bus/platform/drivers/gpio-bcm2835/
# أو
cat /sys/bus/platform/devices/*/driver 2>/dev/null | grep gpio
```

```bash
# تحقق من gpio-ranges (الـ mapping بين GPIO وpin numbers)
hexdump -C /proc/device-tree/gpio@7e200000/gpio-ranges
# format: phandle gpio-base pin-base count
# مثال: <&pinctrl 0 0 54> يعني GPIO 0-53 يطابق pins 0-53
```

---

### Practical Commands — Ready to Copy

#### تشخيص أولي سريع

```bash
#!/bin/bash
# gpio-debug-quick.sh — سكريبت تشخيص سريع للـ GPIO cdev

echo "=== GPIO Chips ==="
gpiodetect

echo -e "\n=== Character Devices ==="
ls -la /dev/gpiochip*

echo -e "\n=== Kernel Debug State ==="
cat /sys/kernel/debug/gpio

echo -e "\n=== Device Numbers ==="
grep gpio /proc/devices

echo -e "\n=== Loaded GPIO Modules ==="
lsmod | grep gpio

echo -e "\n=== Recent GPIO Kernel Messages ==="
dmesg | grep -i gpio | tail -30
```

#### تشخيص مشكلة cdev registration

```bash
# لو /dev/gpiochipN مش موجود بعد boot
dmesg | grep -E "gpiochip|cdev|chrdev"
# دور على:
# "gpiochip_add_data_with_key() failed"  ← مشكلة في الـ chip registration
# "cdev_add failed"                       ← مشكلة في الـ devt

# تحقق من الـ device creation
udevadm info --query=all --name=/dev/gpiochip0
udevadm test /sys/class/gpio/gpiochip0 2>&1 | grep -E "MAJOR|MINOR|creating"
```

#### trace ioctl calls كاملة

```bash
# ftrace على كل functions الـ gpiolib-cdev
cd /sys/kernel/tracing
echo 0             > tracing_on
echo function_graph > current_tracer
echo 'gpiolib_cdev_*:gpiolib_cdev.c' > set_ftrace_filter
# أضف الـ ioctls
echo 'linereq_*'  >> set_ftrace_filter
echo 'linehandle_*' >> set_ftrace_filter
echo 1             > tracing_on

# شغّل الـ test
gpioget gpiochip0 17

echo 0 > tracing_on
cat trace
# output مثال:
# 0) gpiolib_cdev_register() {
#      cdev_init();
#      cdev_add();   /* 0 = success */
#    }
```

#### تحقق من memory leaks في الـ gpio_device

```bash
# استخدم kmemleak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak | grep -A5 "gpio"

# أو راقب الـ slab
cat /proc/slabinfo | grep gpio
# غالباً هتلاقي "gpio_desc" slab
```

#### مراقبة line state changes في real-time

```bash
# شغّل الـ event monitoring في background
gpiomon --rising-edge --falling-edge gpiochip0 17 &
GPIO_MON_PID=$!

# فعّل الـ ftrace في نفس الوقت
echo gpio:* > /sys/kernel/tracing/set_event
echo 1      > /sys/kernel/tracing/tracing_on

# trigger حدث
gpioset gpiochip0 17=1
sleep 0.1
gpioset gpiochip0 17=0

# وقف
echo 0 > /sys/kernel/tracing/tracing_on
kill $GPIO_MON_PID

# حلل النتيجة
cat /sys/kernel/tracing/trace | grep -E "gpio_value|gpio_direction"
# output مثال:
# gpioset-1234  [001] .... gpio_direction: gpio=17 direction=out
# gpioset-1234  [001] .... gpio_value: gpio=17 set=1
```

#### تشخيص hardware register مباشرة

```bash
# BCM2835/BCM2711 (Raspberry Pi) — GPIO Level Register
# GPLEV0 (pins 0-31) offset 0x34 من الـ base
GPIO_BASE=0x3F200000   # Pi 3
# GPIO_BASE=0xFE200000  # Pi 4

# اقرأ GPLEV0
devmem2 $((GPIO_BASE + 0x34)) w
# output: Value at address 0x3F200034 (0xXXXXXXXX): 0x000A0000
# bit 17 = 1 يعني pin 17 = HIGH

# قارن بالـ kernel state
gpioget gpiochip0 17
# 1  ← لازم يطابق bit 17 في GPLEV0
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — `/dev/gpiochip0` مش بيظهر بعد البوت

#### العنوان
**الـ character device مش بيتسجل بسبب فشل الـ workqueue**

#### السياق
شركة صناعية بتبني industrial gateway على RK3562 لـ Modbus RTU. الـ gateway بيتحكم في 8 relays عبر GPIO. البورد بتتبوت بشكل طبيعي بس الـ userspace daemon بيفشل يفتح `/dev/gpiochip0` بـ `ENOENT`.

#### المشكلة
الـ daemon بيعمل `open("/dev/gpiochip0", O_RDWR)` وبيرجع `-1 ENOENT` — الـ device node مش موجود خالص.

#### التحليل
الكود في `gpiolib_cdev_register()`:

```c
int gpiolib_cdev_register(struct gpio_device *gdev, dev_t devt)
{
    /* allocate the ordered workqueue for line state events */
    gdev->line_state_wq = alloc_ordered_workqueue("%s", WQ_HIGHPRI,
                                                  dev_name(&gdev->dev));
    if (!gdev->line_state_wq)
        return -ENOMEM;  /* <-- early return, cdev_device_add never called */

    ret = cdev_device_add(&gdev->chrdev, &gdev->dev);
    ...
}
```

**الـ flow**:
1. `alloc_ordered_workqueue()` فشل بسبب memory pressure أثناء البوت.
2. الدالة رجعت `-ENOMEM` فوراً.
3. `cdev_device_add()` ما اتنادتش أبداً.
4. الـ device node ما اتعملش في `/dev`.

تأكيد من الـ dmesg:
```bash
dmesg | grep -i gpio
# لا يوجد "added GPIO chardev" للـ gpiochip0
dmesg | grep -i "out of memory"
# يظهر OOM killer أو memory pressure أثناء البوت
```

#### الحل
1. زيادة `CMA` أو `memreserve` في الـ DT لضمان وجود ذاكرة كافية أثناء البوت المبكر.
2. مراقبة الـ workqueue creation في الـ boot log:
```bash
dmesg | grep -E "gpiochip|workqueue"
```
3. في extreme cases، تأخير تشغيل الـ daemon حتى يكتمل الـ probing:
```bash
# في systemd unit
After=sys-subsystem-gpio.device
```

#### الدرس المستفاد
`gpiolib_cdev_register()` لها **prerequisite مخفي**: الـ `alloc_ordered_workqueue()` لازم ينجح قبل ما الـ cdev يتسجل. أي failure فيها = لا يوجد `/dev/gpiochipN` خالص، مش مجرد غياب الـ line state notifications.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — تطبيق بيتعلق عند قراءة GPIO events

#### العنوان
**تطبيق بيـblock على `read()` من `/dev/gpiochip1` بعد hot-unplug للـ USB GPIO expander**

#### السياق
Android TV box بيستخدم USB GPIO expander (مبني على Allwinner H616 base + USB hub) للتحكم في LED ring. الـ HAL layer بيفتح `/dev/gpiochip1` ويعمل `watch` على الـ lines. لما المستخدم بيشيل الـ USB expander، الـ HAL thread بيتعلق ومش بيرجع.

#### المشكلة
الـ `read()` على الـ chardev بيبقى blocked حتى بعد ما الـ device اتنزع. الـ HAL thread معلق وبيسبب ANR في بعض الأحيان.

#### التحليل
لما الـ GPIO chip بيتنزع، الكود التالي بيتنفذ:

```c
void gpiolib_cdev_unregister(struct gpio_device *gdev)
{
    destroy_workqueue(gdev->line_state_wq);
    cdev_device_del(&gdev->chrdev, &gdev->dev);
    /* هنا بيبعث إشارة للـ waitqueues المفتوحة */
    blocking_notifier_call_chain(&gdev->device_notifier, 0, NULL);
}
```

الـ `blocking_notifier_call_chain()` بينادي:

```c
static int gpio_device_unregistered_notify(struct notifier_block *nb,
                                           unsigned long action, void *data)
{
    struct gpio_chardev_data *cdev = container_of(nb, ...);
    /* wake up كل اللي نايمين على الـ wait queue */
    wake_up_poll(&cdev->wait, EPOLLIN | EPOLLERR);
    return NOTIFY_OK;
}
```

**المشكلة الفعلية**: الـ HAL بيستخدم `read()` بدون `O_NONBLOCK` وبدون `poll()` أولاً. بعد الـ `wake_up_poll()`، الـ `lineinfo_watch_read()` بتتحقق:

```c
guard(srcu)(&cdev->gdev->srcu);
if (!rcu_access_pointer(cdev->gdev->chip))
    return -ENODEV;  /* <-- هترجع هنا */
```

لكن الـ HAL ما بيتحققش من `-ENODEV` وبيعيد الـ `read()` في loop بلا نهاية.

#### الحل
في الـ HAL code:
```c
/* استخدام poll() مع timeout قبل read() */
struct pollfd pfd = {
    .fd = chip_fd,
    .events = EPOLLIN | EPOLLERR | EPOLLHUP,
};
int ret = poll(&pfd, 1, 1000 /* ms */);
if (ret > 0 && (pfd.revents & (EPOLLERR | EPOLLHUP))) {
    /* الـ device اتنزع، اخرج بنعومة */
    close(chip_fd);
    return;
}
```

#### الدرس المستفاد
الـ `gpiolib_cdev_unregister()` بترسل إشارة `EPOLLERR` عبر `gpio_device_unregistered_notify()`. أي userspace code لازم يراقب هذه الإشارة في `poll()` قبل `read()` عشان يتعامل مع hot-unplug بشكل صحيح.

---

### السيناريو الثالث: IoT Sensor Board على STM32MP1 — `EPERM` عند محاولة تغيير قيمة GPIO

#### العنوان
**`GPIOHANDLE_SET_LINE_VALUES_IOCTL` بيرجع `-EPERM` بشكل غير متوقع**

#### السياق
IoT sensor node على STM32MP1 بيستخدم GPIO لتشغيل external sensor power rail. الـ Python script على الـ userspace بيفتح الـ line ويحاول يكتب قيمة، بس بيرجع `EPERM`.

#### المشكلة
```python
import gpiod
chip = gpiod.Chip('/dev/gpiochip0')
line = chip.get_line(15)
line.request(consumer="sensor-ctrl", type=gpiod.LINE_REQ_DIR_IN)  # <-- INPUT!
line.set_value(1)  # --> PermissionError: [Errno 1] Operation not permitted
```

#### التحليل
في `linehandle_ioctl()`:

```c
case GPIOHANDLE_SET_LINE_VALUES_IOCTL:
    /*
     * All line descriptors were created at once with the same
     * flags so just check if the first one is really output.
     */
    if (!test_bit(GPIOD_FLAG_IS_OUT, &lh->descs[0]->flags))
        return -EPERM;  /* <-- هنا المشكلة */
```

الـ `GPIOD_FLAG_IS_OUT` بيتحط في `linehandle_flags_to_desc_flags()`:

```c
static void linehandle_flags_to_desc_flags(u32 lflags, unsigned long *flagsp)
{
    /* GPIOD_FLAG_IS_OUT يتحط لما GPIOHANDLE_REQUEST_OUTPUT موجود */
    assign_bit(GPIOD_FLAG_ACTIVE_LOW, &flags,
               lflags & GPIOHANDLE_REQUEST_ACTIVE_LOW);
    // ... لكن IS_OUT نفسه يتحدد في gpiod_direction_output_nonotify
}
```

السبب: الـ script طلب الـ line كـ **INPUT** (`LINE_REQ_DIR_IN`) ثم حاول يكتب عليها. الـ kernel صح تماماً — لا يمكن كتابة قيمة على line مُعرَّفة كـ input.

#### الحل
```python
line.request(
    consumer="sensor-ctrl",
    type=gpiod.LINE_REQ_DIR_OUT,  # OUTPUT
    default_vals=[0]
)
line.set_value(1)  # يعمل الآن
```

أو لو محتاج input وoutput في نفس الوقت، استخدم `linehandle_set_config()` عبر `GPIOHANDLE_SET_CONFIG_IOCTL` لتغيير الـ direction ديناميكياً، مع التأكد من وجود `GPIOHANDLE_REQUEST_DIRECTION_FLAGS` في الـ flags.

#### الدرس المستفاد
الـ `GPIOHANDLE_SET_LINE_VALUES_IOCTL` يفحص `GPIOD_FLAG_IS_OUT` على أول descriptor فقط. الـ `-EPERM` هنا مش permission error بالمعنى الأمني، ده error تصميمي — المبرمج طلب الـ line بـ direction غلط.

---

### السيناريو الرابع: Custom Board Bring-up على i.MX8MP — برنامج monitoring بيحصل على stale GPIO info

#### العنوان
**`GPIO_V2_GET_LINEINFO_WATCH_IOCTL` بيرجع `-EBUSY` عند محاولة مشاهدة نفس الـ line مرتين**

#### السياق
أثناء bring-up لبورد custom على i.MX8MP (مستخدمة في automotive ECU)، مهندس الـ BSP يكتب monitoring tool بيراقب تغييرات GPIO lines. الأداة بتكرر تسجيل watch على نفس الـ line بعد كل reconnect وبتحصل على `-EBUSY`.

#### المشكلة
```c
// في monitoring loop
ioctl(chip_fd, GPIO_V2_GET_LINEINFO_WATCH_IOCTL, &lineinfo);
// عند الـ reconnect: -EBUSY
```

#### التحليل
في `lineinfo_get()`:

```c
static int lineinfo_get(struct gpio_chardev_data *cdev, void __user *ip,
                        bool watch)
{
    ...
    if (watch) {
        /* test_and_set_bit: لو الـ bit موجود بالفعل، ارجع EBUSY */
        if (test_and_set_bit(lineinfo.offset, cdev->watched_lines))
            return -EBUSY;
    }
    ...
}
```

الـ `watched_lines` هو `bitmap` يُخزَّن في `gpio_chardev_data` التي تُنشأ في `gpio_chrdev_open()`:

```c
cdev->watched_lines = bitmap_zalloc(gdev->ngpio, GFP_KERNEL);
```

**سبب المشكلة**: المهندس يعيد استخدام نفس الـ `chip_fd` المفتوح ويحاول يسجل watch مرة تانية على نفس الـ offset. الـ bit بالفعل محجوز من المرة الأولى.

**الحل الصحيح**: يجب عمل `unwatch` أولاً:

```c
/* unwatch ثم watch مجدداً */
__u32 offset = 15;
ioctl(chip_fd, GPIO_GET_LINEINFO_UNWATCH_IOCTL, &offset);
ioctl(chip_fd, GPIO_V2_GET_LINEINFO_WATCH_IOCTL, &lineinfo);
```

كود `lineinfo_unwatch()` يمسح الـ bit:

```c
static int lineinfo_unwatch(struct gpio_chardev_data *cdev, void __user *ip)
{
    ...
    if (!test_and_clear_bit(offset, cdev->watched_lines))
        return -EBUSY;  /* لم يكن مراقباً أصلاً */
    return 0;
}
```

أو يغلق الـ `chip_fd` ويفتحه من جديد — الـ `gpio_chrdev_release()` بيعمل `bitmap_free()` ثم الـ `gpio_chrdev_open()` الجديد بيعمل `bitmap_zalloc()` نظيفة.

#### الدرس المستفاد
الـ `watched_lines` bitmap مرتبط بالـ **file descriptor**، مش بالـ GPIO line نفسها. كل `open()` جديد = bitmap جديد نظيف. لو عايز تعيد watch على line موجودة بالفعل، lazim تعمل `unwatch` أولاً أو تفتح الـ chip من جديد.

---

### السيناريو الخامس: AM62x Board — تعارض الـ V1 وV2 ABI في نفس العملية

#### العنوان
**`-EPERM` عند استخدام `GPIO_V2_GET_LINEINFO_WATCH_IOCTL` بعد استخدام `GPIO_GET_LINEINFO_WATCH_IOCTL`**

#### السياق
بورد industrial على AM62x بيشغّل legacy Python script يستخدم الـ V1 GPIO API وscript جديد يستخدم V2 API، وكلاهم يفتح نفس `/dev/gpiochip0` من نفس الـ process (عبر library قديمة وجديدة مدمجتين في نفس الـ daemon).

#### المشكلة
الـ script الأول يعمل `GPIO_GET_LINEINFO_WATCH_IOCTL` (V1) بنجاح. الـ script التاني يعمل `GPIO_V2_GET_LINEINFO_WATCH_IOCTL` ويحصل على `-EPERM`.

#### التحليل
الكود يمنع اختلاط الـ ABI versions في نفس الـ file descriptor.

في `lineinfo_get_v1()`:

```c
#ifdef CONFIG_GPIO_CDEV_V1
static int lineinfo_get_v1(struct gpio_chardev_data *cdev, void __user *ip,
                           bool watch)
{
    if (watch) {
        /* يسجل ABI version = 1 */
        if (lineinfo_ensure_abi_version(cdev, 1))
            return -EPERM;
        ...
    }
}
```

في `lineinfo_get()` (V2):

```c
static int lineinfo_get(struct gpio_chardev_data *cdev, void __user *ip,
                        bool watch)
{
    if (watch) {
#ifdef CONFIG_GPIO_CDEV_V1
        /* لو ABI version مضبوطة على 1 بالفعل، ارفض */
        if (lineinfo_ensure_abi_version(cdev, 2))
            return -EPERM;
#endif
        ...
    }
}
```

الدالة `lineinfo_ensure_abi_version()`:

```c
static int lineinfo_ensure_abi_version(struct gpio_chardev_data *cdata,
                                       unsigned int version)
{
    /* atomic compare-and-swap: 0 -> version */
    int abiv = atomic_cmpxchg(&cdata->watch_abi_version, 0, version);

    if (abiv == version)
        return 0;  /* نفس الـ version أو أول مرة تُسجَّل

    return abiv;   /* version مختلفة = -EPERM للـ caller */
}
```

**الـ watch_abi_version** هو `atomic_t` في `gpio_chardev_data`. أول `watch` ioctl يضبطه ويقفله. أي محاولة لاستخدام version مختلفة على نفس الـ fd ترجع `-EPERM`.

#### الحل
```bash
# افتح fd منفصل لكل ABI version
fd_v1 = open("/dev/gpiochip0", O_RDWR)  # للـ V1 watches
fd_v2 = open("/dev/gpiochip0", O_RDWR)  # للـ V2 watches
```

كل `open()` يُنشئ `gpio_chardev_data` منفصل بـ `watch_abi_version = 0` خاص به.

```c
/* gpio_chrdev_open() */
cdev = kzalloc(sizeof(*cdev), GFP_KERNEL);
// watch_abi_version يبدأ 0 لأن kzalloc بيصفر الذاكرة
```

#### الدرس المستفاد
كل `/dev/gpiochipN` يمكن فتحه عدة مرات. كل `open()` = state منفصل تماماً بما فيه الـ `watch_abi_version`. الـ V1 و V2 watch APIs **لا يمكن مزجهما** على نفس الـ file descriptor لأن الـ kernel يقفل الـ ABI version عند أول watch ioctl لضمان consistency في الـ `lineinfo_watch_read()` الذي يقرر حجم الـ event بناءً على الـ version.
## Phase 7: مصادر ومراجع

### مقدمة

الملف `gpiolib-cdev.h` هو header بسيط بيعرف الـ interface الخاص بـ GPIO character device layer في الـ kernel. بيحتوي على إعلان دالتين أساسيتين: `gpiolib_cdev_register` و`gpiolib_cdev_unregister`. المصادر دي هتساعدك تفهم الـ GPIO cdev subsystem بعمق.

---

### مقالات LWN.net

| المقال | الأهمية |
|--------|---------|
| [GPIO Character Device Documentation](https://lwn.net/Articles/958322/) | توثيق الـ uAPI الرسمي للـ chardev — مرجع أساسي |
| [gpio: cdev: add uAPI v2](https://lwn.net/Articles/832139/) | المقال اللي غطى إضافة الـ v2 API في kernel 5.10 |
| [gpiolib: add an ioctl() for monitoring line status changes](https://lwn.net/Articles/812198/) | إضافة `GPIO_GET_LINEINFO_WATCH_IOCTL` للـ cdev |
| [The cdev interface](https://lwn.net/Articles/195805/) | الـ character device interface في الـ kernel بشكل عام |
| [gpio: sysfs: send character device notifications for sysfs class events](https://lwn.net/Articles/995569/) | التكامل بين الـ sysfs والـ cdev notifications |
| [gpio: Add GPIO Aggregator Driver](https://lwn.net/Articles/798971/) | driver بيستخدم الـ cdev interface |

---

### التوثيق الرسمي في الـ kernel

**الـ Documentation paths** الأهم:

```
Documentation/driver-api/gpio/
├── index.rst          ← نقطة البداية للـ GPIO subsystem
├── consumer.rst       ← استخدام GPIO من داخل الـ kernel
├── driver.rst         ← كيفية كتابة GPIO controller driver
└── legacy.rst         ← الـ integer-based API القديم (deprecated)

Documentation/userspace-api/gpio/
├── chardev.rst        ← الـ cdev uAPI v1 و v2 كاملة
└── index.rst
```

**روابط مباشرة:**

- [GPIO Character Device Userspace API](https://docs.kernel.org/userspace-api/gpio/chardev.html) — الـ ioctl المتاحة، الـ structs، والـ flags
- [General Purpose Input/Output (GPIO) — Kernel Docs](https://www.kernel.org/doc/html/latest/driver-api/gpio/index.html) — الـ driver-api الكاملة

---

### الـ Source Code الأساسي

الملفات المرتبطة مباشرة بالـ header:

```
drivers/gpio/gpiolib-cdev.h   ← الملف المدروس — الـ interface
drivers/gpio/gpiolib-cdev.c   ← الـ implementation الكاملة
drivers/gpio/gpiolib.c        ← الـ core gpiolib اللي بيستدعي register/unregister
drivers/gpio/gpiolib.h        ← الـ internal structs زي gpio_device
include/uapi/linux/gpio.h     ← الـ uAPI structs والـ ioctls
```

- [gpiolib-cdev.c على GitHub (torvalds/linux)](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpiolib-cdev.c)
- [gpiolib.c على GitHub (torvalds/linux)](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpiolib.c)

---

### نقاشات الـ Mailing List والـ Patches

#### سلسلة uAPI v2 (الأهم)

الـ uAPI v2 اتطورت عبر 9 إصدارات من الـ patches قبل ما تتقبل في kernel 5.10:

- [PATCH v2 00/18 — gpio: cdev: add uAPI V2](https://lkml.kernel.org/lkml/20200725041955.9985-4-warthog618@gmail.com/T/) — البداية من Kent Gibson
- [PATCH v4 00/20 — gpio: cdev: add uAPI v2](https://lkml.kernel.org/lkml/20200814030257.135463-19-warthog618@gmail.com/T/)
- [PATCH v9 00/20 — gpio: cdev: add uAPI v2 (final)](https://yhbt.net/lore/all/20200924080921.GE17562@sol/t/)

#### نقاشات الـ Bug Fixes

- [Re: gpiolib: cdev: Fix use after free in lineinfo_changed_notify](https://lore.kernel.org/linux-kernel/20240502015122.GA15967@rigel/) — مناقشة use-after-free في الـ notifier
- [Re: gpiolib: cdev: release IRQs when the gpio chip device is removed](https://lore.kernel.org/lkml/20240227092627.23b883c5@bootlin.com/) — cleanup صح لما الـ device بتتشال

#### Patch الـ chardev الأصلي (v1)

- [gpio: add a userspace chardev ABI for GPIOs (v3)](https://patchwork.ozlabs.org/patch/580307/) — الـ patch الأول اللي عرّف الـ cdev ABI

---

### Kernel Commits المهمة

| الـ Commit | الوصف |
|-----------|-------|
| `d7c51b47ac11` | إضافة الـ GPIO character device الأصلية (kernel 4.8) |
| `aad14237a5fd` | gpio: introduce GPIO_V2 uAPI |
| `0486beaf88d2` | [diff على Google Git](https://kernel.googlesource.com/pub/scm/linux/kernel/git/brauner/linux/+/0486beaf88d2460e9dbcbba281dab683a838f0c6%5E1..0486beaf88d2460e9dbcbba281dab683a838f0c6/) |

للبحث في تاريخ الـ commits:

```bash
git log --oneline drivers/gpio/gpiolib-cdev.c | head -20
git log --oneline drivers/gpio/gpiolib-cdev.h
```

---

### libgpiod — المكتبة الرسمية للـ Userspace

الـ `gpiolib_cdev_register` بيسجل الـ `/dev/gpiochipN` اللي `libgpiod` بتتعامل معاه:

- [libgpiod على kernel.org](https://git.kernel.org/pub/scm/libs/libgpiod/libgpiod.git/)
- [libgpiod mirror على GitHub](https://github.com/brgl/libgpiod) — للـ issues والنقاشات
- [Linux kernel GPIO user space interface](https://sergioprado.blog/new-linux-kernel-gpio-user-space-interface/) — شرح عملي كويس
- [An Introduction to chardev GPIO and Libgpiod on the Raspberry PI](https://www.beyondlogic.org/an-introduction-to-chardev-gpio-and-libgpiod-on-the-raspberry-pi/) — tutorial عملي

---

### Kernelnewbies.org

- [Linux_6.2 — GPIO latch driver و software nodes](https://kernelnewbies.org/Linux_6.2) — بيذكر تغييرات الـ gpiolib في 6.2
- [Linux_6.6 — GPIO expander improvements](https://kernelnewbies.org/Linux_6.6)
- [Linux_6.13 — Aspeed G7 GPIO support](https://kernelnewbies.org/Linux_6.13)
- [LinuxChanges — تاريخ التغييرات](https://kernelnewbies.org/LinuxChanges)

---

### eLinux.org

- [New GPIO Interface for User Space (PDF — ELCE 2017)](https://elinux.org/images/7/74/Elce2017_new_GPIO_interface.pdf) — presentation بتشرح ليه الـ cdev أفضل من الـ sysfs
- [Introduction to pin muxing and GPIO control under Linux (PDF — ELC 2021)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) — شامل جداً
- [EBC GPIO via mmap](https://elinux.org/EBC_GPIO_via_mmap) — مقارنة طرق الوصول للـ GPIO
- [EBC gpio Polling and Interrupts](https://elinux.org/EBC_gpio_Polling_and_Interrupts) — polling والـ interrupts في الـ GPIO

---

### كتب مقترحة

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 3**: Char Drivers — أساسي لفهم `cdev_init`, `cdev_add`, `cdev_del`
- **الفصل 6**: Advanced Char Driver Operations — الـ `ioctl`, `poll`, `fasync`
- متاح مجاناً: [LDD3 Online](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 13**: The Virtual Filesystem — فهم الـ file operations
- **الفصل 17**: Devices and Modules — `dev_t`, `cdev`, character devices

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 8**: Device Drivers — GPIO في السياق الـ embedded
- بيغطي الفرق بين الـ sysfs interface القديم والـ cdev الجديد

#### Linux Driver Development for Embedded Processors — Alberto Liberal de los Ríos
- بيركز على GPIO descriptor API والـ cdev في الـ embedded systems

---

### مقالات إضافية مفيدة

- [Stop using /sys/class/gpio – it's deprecated](https://www.thegoodpenguin.co.uk/blog/stop-using-sys-class-gpio-its-deprecated/) — شرح ليه لازم تتحول للـ cdev
- [GPIOLib overview — STM32MPU Wiki](https://wiki.st.com/stm32mpu/wiki/GPIOLib_overview)
- [How To Use gpiolib — Timesys LinuxLink](https://linuxlink.timesys.com/docs/wiki/engineering/HOWTO_Use_gpiolib)

---

### Search Terms للبحث عن معلومات أكتر

```
# بحث عام
linux gpio character device uapi
gpiolib cdev register unregister
/dev/gpiochip linux kernel internals

# بحث في الـ source
git log --all --grep="gpio: cdev" -- drivers/gpio/
git log --all --grep="GPIO_V2" -- include/uapi/linux/gpio.h

# بحث في الـ mailing list
site:lore.kernel.org gpiolib cdev
site:lkml.kernel.org gpio chardev uapi

# أدوات الـ userspace
libgpiod gpiomon gpioget gpioset
gpio-utils linux userspace tools
```
## Phase 8: Writing simple module

### الهدف

هنعمل module بيـhook الـfunction `gpiolib_cdev_register` اللي موجودة في `drivers/gpio/gpiolib-cdev.c` عن طريق **kprobe**. الـfunction دي بتتاخد كل ما يتسجل GPIO chip كـcharacter device (`/dev/gpiochipN`)، فده بيخلينا نشوف كل chip بيتضاف للـsystem مع الـ`dev_t` بتاعه.

---

### الـModule الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * gpio_cdev_probe.c
 *
 * kprobe on gpiolib_cdev_register() — fires every time a GPIO chip
 * gets a /dev/gpiochipN character device created for it.
 *
 * Build with a Makefile that sets:
 *   obj-m += gpio_cdev_probe.o
 */

/* --- Includes --- */
#include <linux/module.h>       /* MODULE_*, module_init/exit            */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe …      */
#include <linux/device.h>       /* struct device, dev_name()             */
#include <linux/kdev_t.h>       /* MAJOR(), MINOR() macros               */

/*
 * Forward-declare the internal struct just enough to read its fields.
 * We only need what's visible at the hook point: the device embedded
 * inside gpio_device and its integer id.
 *
 * Full definition lives in drivers/gpio/gpiolib.h (internal header),
 * so we re-declare the minimum here rather than pulling a private header.
 */
struct gpio_device {
    struct device   dev;   /* must be first — matches the real layout */
    struct cdev     chrdev;
    int             id;
    /* ... rest of fields we don't touch ... */
};

/* ------------------------------------------------------------------ */
/* kprobe pre-handler                                                  */
/* ------------------------------------------------------------------ */

/*
 * gpiolib_cdev_register signature:
 *   int gpiolib_cdev_register(struct gpio_device *gdev, dev_t devt);
 *
 * On x86-64 the arguments arrive in:
 *   regs->di  -> gdev  (first arg)
 *   regs->si  -> devt  (second arg, dev_t is typedef'd unsigned int)
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve arguments from CPU registers (x86-64 calling convention) */
    struct gpio_device *gdev = (struct gpio_device *)regs->di;
    dev_t devt = (dev_t)regs->si;

    /*
     * dev_name() returns the kernel device name string, e.g. "gpiochip0".
     * MAJOR/MINOR decompose the allocated device number so we can see
     * which major was assigned and which minor corresponds to this chip's id.
     */
    pr_info("gpio_cdev_probe: registering cdev for [%s] "
            "id=%d devt=%u (%u:%u)\n",
            dev_name(&gdev->dev),
            gdev->id,
            devt,
            MAJOR(devt),
            MINOR(devt));

    /* Return 0 → let the original function continue normally */
    return 0;
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */

static struct kprobe gpio_cdev_kp = {
    /*
     * Symbol name must match exactly what's in /proc/kallsyms.
     * gpiolib_cdev_register is NOT exported via EXPORT_SYMBOL, but
     * kprobes can hook any symbol present in kallsyms as long as
     * CONFIG_KPROBES=y and the symbol isn't on the kprobe blacklist.
     */
    .symbol_name = "gpiolib_cdev_register",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* init / exit                                                         */
/* ------------------------------------------------------------------ */

static int __init gpio_cdev_probe_init(void)
{
    int ret;

    ret = register_kprobe(&gpio_cdev_kp);
    if (ret < 0) {
        pr_err("gpio_cdev_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("gpio_cdev_probe: kprobe planted at %s (%p)\n",
            gpio_cdev_kp.symbol_name, gpio_cdev_kp.addr);
    return 0;
}

static void __exit gpio_cdev_probe_exit(void)
{
    /*
     * Must unregister before the module is unloaded; otherwise the
     * kprobe's pre_handler pointer will point to freed text and the
     * next GPIO registration will panic the kernel.
     */
    unregister_kprobe(&gpio_cdev_kp);
    pr_info("gpio_cdev_probe: kprobe removed\n");
}

module_init(gpio_cdev_probe_init);
module_exit(gpio_cdev_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on gpiolib_cdev_register to trace GPIO cdev creation");
```

---

### شرح كل جزء

#### الـIncludes

| Header | ليه محتاجينه |
|---|---|
| `linux/module.h` | الـmacros الأساسية زي `MODULE_LICENSE` و`module_init/exit` |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل الـAPI بتاعه |
| `linux/device.h` | `struct device` و`dev_name()` عشان نطبع اسم الـdevice |
| `linux/kdev_t.h` | الـmacros `MAJOR()` و`MINOR()` عشان نفك الـ`dev_t` |

الـheader الداخلي `drivers/gpio/gpiolib.h` مش متاح خارج الـdriver، فعشان كده عملنا **forward declaration** بالـfields اللي محتاجينها بس.

---

#### الـForward Declaration

```c
struct gpio_device { struct device dev; struct cdev chrdev; int id; ... };
```

**الـ`struct gpio_device`** struct داخلي مش exported للـmodules. بنعرفه بالـfields الأولى بس (الترتيب مهم جداً عشان الـlayout في الـmemory لازم يتطابق مع الـoriginal). ده بيخلينا نقرأ `gdev->id` و`gdev->dev` بأمان.

---

#### الـ`handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

ده الـ**callback** اللي بيشتغل قبل ما `gpiolib_cdev_register` تنفذ أي instruction. الـargs بنجيبهم من الـregisters لأن الـkprobe بيوقف الـexecution على أول instruction من الـfunction، فالـcalling convention لسه ما اتبدلش.

- **`regs->di`**: الـfirst argument (`gdev`) على x86-64.
- **`regs->si`**: الـsecond argument (`devt`).
- الـreturn value `0` بيقول للـkprobe framework "كمّل تنفيذ الـoriginal function عادي".

---

#### الـ`struct kprobe`

```c
static struct kprobe gpio_cdev_kp = {
    .symbol_name = "gpiolib_cdev_register",
    .pre_handler = handler_pre,
};
```

**الـ`.symbol_name`** بدل ما نحط عنوان ثابت، بنسيب الـkernel يحل الـsymbol وقت الـregistration من الـkallsyms table. الـ`.pre_handler` بيشير للـcallback بتاعنا.

---

#### الـ`module_init`

```c
ret = register_kprobe(&gpio_cdev_kp);
```

**`register_kprobe`** بتحط breakpoint (INT3 على x86) في أول byte من `gpiolib_cdev_register`. لو فشلت — مثلاً لو الـsymbol مش موجود أو الـfunction على الـblacklist — بنرجع الـerror فوراً.

---

#### الـ`module_exit`

```c
unregister_kprobe(&gpio_cdev_kp);
```

لازم نعمل **unregister** قبل ما الـmodule يتشال من الـmemory. لو سبنا الـkprobe مسجلة والـmodule اتشال، الـkernel هيحاول ينفذ `handler_pre` من عنوان memory اتحررت فعلاً وهيحصل kernel panic فوري.

---

### طريقة التجربة

```bash
# بناء الـmodule
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميله
sudo insmod gpio_cdev_probe.ko

# شوف الـlog
sudo dmesg | grep gpio_cdev_probe

# لو عندك GPIO controller جرّب rebind
echo "0000:00:1f.0" | sudo tee /sys/bus/pci/drivers/some_gpio/unbind
echo "0000:00:1f.0" | sudo tee /sys/bus/pci/drivers/some_gpio/bind

# تفريغ الـmodule
sudo rmmod gpio_cdev_probe
```

**مثال output متوقع:**

```
gpio_cdev_probe: kprobe planted at gpiolib_cdev_register (0xffffffffc08a1230)
gpio_cdev_probe: registering cdev for [gpiochip0] id=0 devt=260 (260:0)
gpio_cdev_probe: registering cdev for [gpiochip1] id=1 devt=260 (260:1)
```

---

### ملاحظات مهمة

- الـkprobe بيشتغل في **atomic context** على x86 عند الـINT3، فـ`pr_info` في الـ`pre_handler` مقبولة لأنها non-sleeping. تجنب أي lock يعمل sleep جوا الـhandler.
- الـ`gpiolib_cdev_register` مش في الـkprobe blacklist (زي `__schedule` وأصحابها) فالـhook آمن.
- لو الـkernel اتبنى بـ`CONFIG_KPROBES=n` أو بـ`CONFIG_KALLSYMS=n`، الـmodule مش هيشتغل.
