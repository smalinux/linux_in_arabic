## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من **GPIO Subsystem** في Linux kernel، بالتحديد من طبقة **gpiolib** — وهي الطبقة الداخلية اللي بتدير كل GPIO chips في النظام. المشرفين عليه هما Linus Walleij و Bartosz Golaszewski، والـ mailing list هو `linux-gpio@vger.kernel.org`.

---

### الفكرة الكبيرة — GPIO إيه ده أصلاً؟

تخيل إن عندك Raspberry Pi أو أي embedded board. فيه pins صغيرة على الجنب — دي الـ **GPIO pins** (General Purpose Input/Output). ممكن تقول لـ pin: "بقى output وشغّل LED"، أو "بقى input واحكيلي لو حد ضغط button". الـ kernel بيشوف الـ GPIO chip اللي على الـ hardware ويعرضها للـ userspace بطرق مختلفة.

---

### الـ sysfs إيه؟

الـ **sysfs** هو نظام ملفات افتراضي في Linux بيتعمل mount على `/sys`. الـ kernel بيعرض فيه معلومات الأجهزة كـ files وـ directories. بدل ما تكتب C code وـ syscalls، بتعمل `cat` أو `echo` على files في `/sys` وتتحكم في الـ hardware.

مثال حقيقي:
```bash
# اعرف اتجاه GPIO رقم 17
cat /sys/class/gpio/gpio17/direction

# غير القيمة لـ high
echo "1" > /sys/class/gpio/gpio17/value
```

---

### المشكلة اللي بتحلها

لما الـ kernel بيلاقي GPIO chip جديدة (مثلاً عند boot أو لما driver بيتحمّل)، محتاج يعرضها للـ userspace. في الـ **legacy sysfs interface** القديمة، كانت الطريقة هي إنك تعمل `echo 17 > /sys/class/gpio/export` وبعدين تظهر directory `/sys/class/gpio/gpio17/` فيها attributes زي `direction`، `value`، `edge`.

الـ **modern interface** بدّلها بـ **character device** (`/dev/gpiochipN`) واللي بيستخدم `ioctl`، لكن الـ sysfs القديمة لسه موجودة لأسباب **backward compatibility**.

---

### دور الملف `gpiolib-sysfs.h`

الملف ده هو الـ **internal header** اللي بيعرض interface بسيطة جداً لجوا الـ gpiolib:

```c
/* لو CONFIG_GPIO_SYSFS متفعّلة، دي الـ real functions */
int gpiochip_sysfs_register(struct gpio_device *gdev);
void gpiochip_sysfs_unregister(struct gpio_device *gdev);
```

لو الـ `CONFIG_GPIO_SYSFS` مش متفعّلة في الـ kernel config، بيستبدلهم بـ **no-op stubs** (functions فاضية بترجع 0)، فالكود التاني في gpiolib ممكن يكلمهم من غير ما يحتاج `#ifdef` في كل مكان.

---

### القصة الكاملة بالترتيب

```
1. Driver بيجيب GPIO chip جديدة
        ↓
2. gpiolib.c بتعمل struct gpio_device
        ↓
3. بتكلم gpiochip_sysfs_register(gdev)
        ↓
4. gpiolib-sysfs.c بتعمل entries في /sys/class/gpio/
        ↓
5. Userspace يقدر يتعامل مع GPIOs عن طريق read/write على files
        ↓
6. لما الـ chip تتشال: gpiochip_sysfs_unregister(gdev)
        ↓
7. الـ sysfs entries بتتمسح
```

---

### ليه الملف صغير جداً؟

لأن الـ header ده بس بيعمل **decoupling** — بيفصل الـ implementation اللي في `gpiolib-sysfs.c` عن باقي الـ gpiolib. الـ `gpiolib.c` بيعرف بس إن في function اسمها `gpiochip_sysfs_register` بتاخد `gpio_device*` — مش محتاج يعرف أي تفصيلة داخلية.

الـ **compile-time switching** عن طريق `#ifdef CONFIG_GPIO_SYSFS` بيخلي الـ kernel يُبنى من غير sysfs support خالص لو محتاج تقليل الـ footprint (مثلاً في embedded systems صغيرة).

---

### الـ struct الأساسي: `gpio_device`

```c
struct gpio_device {
    struct device    dev;      /* Linux device model integration */
    struct cdev      chrdev;   /* character device لـ /dev/gpiochipN */
    int              id;       /* رقم الـ chip */
    struct gpio_chip *chip;    /* pointer للـ hardware driver */
    struct gpio_desc *descs;   /* array of GPIO line descriptors */
    u16              ngpio;    /* عدد الـ GPIO lines */
    const char       *label;   /* اسم وصفي مثل "pinctrl-bcm2835" */
    /* ... */
};
```

الـ `gpiolib-sysfs.h` بيعمل **forward declaration** فقط لـ `struct gpio_device` — مش محتاج يعرف كل تفاصيله، بس محتاج يعرف إنه موجود.

---

### الملفات المرتبطة

| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpiolib.c` | الـ core — بيدير كل الـ GPIO devices وبيكلم sysfs |
| `drivers/gpio/gpiolib-sysfs.c` | الـ implementation الفعلية لـ sysfs interface |
| `drivers/gpio/gpiolib-sysfs.h` | الملف ده — internal header للـ decoupling |
| `drivers/gpio/gpiolib-cdev.c` | الـ modern character device interface |
| `drivers/gpio/gpiolib-cdev.h` | header للـ cdev interface |
| `drivers/gpio/gpiolib.h` | main internal header — تعريف `gpio_device` و`gpio_desc` |
| `drivers/gpio/gpiolib-legacy.c` | كود compatibility قديم |
| `include/linux/gpio/driver.h` | public header لكتّاب الـ GPIO drivers |
| `include/linux/gpio/consumer.h` | public header لمستخدمي الـ GPIOs من kernel code |
| `include/linux/gpio.h` | legacy GPIO API |

---

### ملفات الـ Hardware Drivers (أمثلة)

| الملف | الـ Hardware |
|-------|-------------|
| `drivers/gpio/gpio-bcm2835.c` | Raspberry Pi GPIO |
| `drivers/gpio/gpio-pl061.c` | ARM PrimeCell GPIO |
| `drivers/gpio/gpio-intel-mid.c` | Intel MID platforms |
| `drivers/gpio/gpio-regmap.c` | regmap-based GPIO controllers |

---

### خلاصة

الـ `gpiolib-sysfs.h` ملف صغير جداً لكن دوره مهم: بيعمل **clean interface** بين الـ core GPIO library وبين الـ sysfs layer، ومع ذلك بيسمح بـ **conditional compilation** كاملة لو الـ sysfs مش مطلوبة. كل الـ GPIO subsystem مبني على مبدأ الفصل ده — كل طبقة بتعرف عن التاني بس اللي محتاجه بالظبط.
## Phase 2: شرح الـ GPIO sysfs Framework

### المشكلة اللي بتحلها الـ subsystem دي

في الـ embedded Linux، الـ userspace application محتاج يتحكم في GPIO — يشغل LED، يقرأ button، يبعث signal لـ external chip. المشكلة إن GPIO هو hardware resource بيتحكم فيه الـ kernel بشكل حصري. الـ userspace مش بيقدر يكلم الـ hardware مباشرة.

الحل الأول اللي اتعمل في الـ kernel كان الـ **GPIO sysfs interface** — بتعمل files في `/sys/class/gpio/` وكل file بيمثل GPIO line. الـ userspace بيكتب وبيقرأ من الـ files دي زي ما بيتعامل مع أي text file.

المشكلة إن الـ approach ده deprecated دلوقتي، اتبدل بـ **GPIO character device** (`/dev/gpiochipN`) اللي بيستخدم ioctls وأسرع وأأمن. بس الـ sysfs interface لسه موجود للـ legacy systems.

---

### الـ Problem بالتفصيل

**قبل** الـ GPIO subsystem كان كل driver بيعمل `/proc` أو `/sysfs` entry خاص بيه بشكل عشوائي — مفيش standard، مفيش consistent API. الـ userspace كان لازم يعرف implementation details لكل chip.

**الـ GPIO sysfs framework** جاء يعمل:
1. **توحيد الـ interface** — نفس الـ paths لكل GPIO على أي platform
2. **abstraction** — الـ userspace مش محتاج يعرف هو شغال على BCM2835 أو i.MX8 أو أي SoC تاني
3. **access control** — الـ kernel بيتحكم مين يقدر يوصل لأيه

---

### الـ Solution — الـ Kernel بيعمل إيه؟

الـ GPIO subsystem بيستخدم الـ **sysfs** (اللي هو جزء من الـ kobject/device model) عشان يعمل files بتمثل GPIO chips والـ lines الخاصة بيها.

**الـ sysfs** هو virtual filesystem بيـmount على `/sys/` — كل entry فيه بيمثل kernel object. الـ device model في الـ kernel (راجع `drivers/base/`) بيبني الـ hierarchy ده أوتوماتيك لما بتسجل devices.

الـ GPIO sysfs framework بيعمل التالي:
- لما `gpiochip_sysfs_register()` بتتكلم، بتعمل entry في `/sys/class/gpio/gpiochipN/`
- لما الـ userspace يعمل `export` لـ GPIO معين، بيتعمل entry `/sys/class/gpio/gpioN/` فيه attributes

---

### الـ Real-World Analogy — مديرية الكهرباء

تخيل إن الـ GPIO chip هو **لوحة توزيع كهربا** في عمارة. كل **قاطع** في اللوحة هو GPIO line.

| التشبيه | الحقيقة في الـ kernel |
|---|---|
| اللوحة كلها | `struct gpio_device` — الـ chip كله |
| كل قاطع | `struct gpio_desc` — GPIO line واحدة |
| رقم القاطع | hardware GPIO number (hwnum) |
| لافتة على القاطع | `label` field في `struct gpio_desc` |
| غرفة التحكم (`/sys/class/gpio/`) | الـ sysfs directory |
| ورقة الطلب اللي بتملأها عشان توصل للقاطع | `export` — بيخلي الـ GPIO accessible من الـ userspace |
| إذن الدخول | `gpiod_request()` — الـ kernel بيـclaim الـ GPIO |
| ورقة تشغيل/إطفاء | `/sys/class/gpio/gpioN/value` |
| تحديد الاتجاه (input/output) | `/sys/class/gpio/gpioN/direction` |
| إشعار لما الحالة تتغير | edge detection + `poll()` on `/value` |

لما الكهربائي (الـ userspace app) عايز يتحكم في قاطع معين، هو بيكتب رقمه في ورقة الطلب (يكتب الـ GPIO number في `/sys/class/gpio/export`). اللوحة (الـ kernel) بتديله غرفة خاصة بيه في `/sys/class/gpio/gpioN/`. من جوه الغرفة دي، هو يقدر يقرأ الحالة أو يغيرها.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                       Userspace                             │
│                                                             │
│  echo 17 > /sys/class/gpio/export                           │
│  echo out > /sys/class/gpio/gpio17/direction                │
│  echo 1   > /sys/class/gpio/gpio17/value                    │
└──────────────────────┬──────────────────────────────────────┘
                       │  VFS read/write
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    sysfs layer                              │
│            (/sys/class/gpio/...)                            │
│                                                             │
│  export_store()  direction_show/store()  value_show/store() │
└──────────────────────┬──────────────────────────────────────┘
                       │ calls
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              gpiolib-sysfs.c                                │
│                                                             │
│  gpiochip_sysfs_register()                                  │
│  gpiochip_sysfs_unregister()                                │
│  struct gpiod_data  (per exported line)                     │
│  struct gpiodev_data (per chip)                             │
└──────────────────────┬──────────────────────────────────────┘
                       │ uses
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    gpiolib core                             │
│              (gpiolib.c / gpiolib.h)                        │
│                                                             │
│  struct gpio_device   ─────────┐                           │
│  struct gpio_desc[]            │                           │
│  gpiod_request()               │                           │
│  gpiod_direction_input/output()│                           │
│  gpiod_get/set_value()         │                           │
└──────────────────────┬─────────┼───────────────────────────┘
                       │         │
          ┌────────────┘         └──────────────┐
          ▼                                     ▼
┌──────────────────────┐         ┌──────────────────────────┐
│   struct gpio_chip   │         │   struct gpio_chip       │
│  (gpio-pca953x.c)    │         │  (gpio-bcm-virt.c)       │
│                      │         │                          │
│  .get()              │         │  .get()                  │
│  .set()              │         │  .set()                  │
│  .direction_input()  │         │  .direction_input()      │
│  .direction_output() │         │  .direction_output()     │
└──────────────────────┘         └──────────────────────────┘
          │                                     │
          ▼                                     ▼
    I2C Register                         Memory-Mapped
    Read/Write                           Register I/O
```

---

### الـ Core Abstraction

الـ abstraction المحورية في الـ GPIO sysfs هي **تحويل file I/O إلى hardware operations**.

الـ kernel بيعمل ده عن طريق:

1. **`struct device_attribute`** — بتربط كل sysfs file بـ callback functions (show/store)
2. **`struct gpiod_data`** — الـ per-line state اللي بيربط الـ sysfs file بالـ `gpio_desc`
3. **`struct gpiodev_data`** — الـ per-chip state اللي بيتتبع كل الـ exported lines

```
sysfs file read/write
        │
        ▼
direction_show() / direction_store()
value_show()     / value_store()
        │
        │  container_of(attr, struct gpiod_data, dir_attr)
        ▼
struct gpiod_data {
    struct gpio_desc *desc;    ──────────────┐
    struct device    *dev;                   │
    struct mutex      mutex;                 │
    struct kobject   *parent;                │
    struct device_attribute dir_attr;        │
    struct device_attribute val_attr;        │
}                                            │
                                             ▼
                                    struct gpio_desc {
                                        struct gpio_device *gdev;
                                        unsigned long flags;
                                        /* GPIOD_FLAG_IS_OUT */
                                        /* GPIOD_FLAG_SYSFS  */
                                        /* GPIOD_FLAG_ACTIVE_LOW */
                                        ...
                                    }
                                             │
                                             ▼
                                    struct gpio_device {
                                        struct gpio_chip *chip; (RCU)
                                        struct gpio_desc *descs;
                                        u16 ngpio;
                                        ...
                                    }
                                             │
                                             ▼
                                    struct gpio_chip {
                                        int (*get)(gc, offset);
                                        void (*set)(gc, offset, val);
                                        int (*direction_input)(gc, offset);
                                        int (*direction_output)(gc, offset, val);
                                    }
```

---

### الـ CONFIG_GPIO_SYSFS Guard Pattern

الـ header file `gpiolib-sysfs.h` بيستخدم **conditional compilation pattern** مهم:

```c
#ifdef CONFIG_GPIO_SYSFS

int gpiochip_sysfs_register(struct gpio_device *gdev);
void gpiochip_sysfs_unregister(struct gpio_device *gdev);

#else

/* No-op stubs when sysfs support is disabled */
static inline int gpiochip_sysfs_register(struct gpio_device *gdev)
{
    return 0;  /* always success — nothing to do */
}

static inline void gpiochip_sysfs_unregister(struct gpio_device *gdev)
{
    /* nothing */
}

#endif
```

الـ pattern ده بيعمل حاجتين مهمين:

**أولاً:** لما `CONFIG_GPIO_SYSFS=n` (زي في الـ embedded systems اللي مش محتاجة sysfs)، الـ compiler بيحذف الـ sysfs code كله — zero overhead. الـ functions بتبقى `static inline` والـ compiler بيـinline الـ no-ops ومفيش function call overhead خالص.

**ثانياً:** الـ caller code في `gpiolib.c` ممكن يكلم `gpiochip_sysfs_register()` بدون ما يعمل `#ifdef` في كل حتة — الـ header بيخفي التفاصيل دي.

ده مبدأ الـ **Kconfig-controlled stub** اللي بتلاقيه في طول الـ kernel.

---

### الـ Dual Interface: Legacy vs. New

الـ GPIO sysfs فيه طبقتين:

```
┌─────────────────────────────────────────────────┐
│  CONFIG_GPIO_SYSFS_LEGACY (deprecated)          │
│                                                 │
│  /sys/class/gpio/gpio17/                        │
│      direction   ← "in" / "out" / "high" / "low"│
│      value       ← "0" / "1"                    │
│      edge        ← "none"/"rising"/"falling"/"both"│
│      active_low  ← "0" / "1"                    │
│                                                 │
│  يستخدم struct gpiod_data + irq handling        │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│  New sysfs (under gpiochipN)                    │
│                                                 │
│  /sys/class/gpio/gpiochip0/                     │
│      gpio0/                                     │
│          direction                              │
│          value                                  │
│                                                 │
│  No IRQ/edge support — use char dev for that    │
└─────────────────────────────────────────────────┘
```

الـ `struct gpiod_data` بيعمل conditional compilation داخلياً:

```c
struct gpiod_data {
    struct list_head list;
    struct gpio_desc *desc;
    struct device    *dev;
    struct mutex      mutex;

#if IS_ENABLED(CONFIG_GPIO_SYSFS_LEGACY)
    struct kernfs_node *value_kn;  /* for poll() notification */
    int  irq;
    unsigned char irq_flags;
    struct device_attribute edge_attr;
    struct device_attribute active_low_attr;
    /* ... legacy attr groups ... */
#endif

    bool direction_can_change;
    struct kobject   *parent;
    struct device_attribute dir_attr;
    struct device_attribute val_attr;
    /* ... new-style attr groups ... */
};
```

---

### الـ Pinctrl Dependency

الـ GPIO subsystem بيتكامل مع **Pinctrl subsystem** (اللي بيتحكم في الـ pin muxing والـ electrical characteristics). لما `CONFIG_PINCTRL=y`، الـ `struct gpio_device` بيحتوي على `list_head pin_ranges` بيربطه بالـ pinctrl device.

**الـ Pinctrl** بـ جملة واحدة: هو الـ subsystem اللي بيتحكم في إزاي الـ physical pin بيشتغل — GPIO ولا UART TX ولا SPI MISO — وده بييجي **قبل** ما تقدر تستخدم الـ GPIO.

---

### الـ Ownership: مين بيعمل إيه؟

| المسؤولية | الـ GPIO sysfs layer | الـ gpiolib core | الـ gpio_chip driver |
|---|---|---|---|
| إنشاء الـ sysfs files | ✓ | | |
| الـ export/unexport logic | ✓ | | |
| الـ mutex للـ attribute access | ✓ | | |
| الـ IRQ request للـ edge detection | ✓ (legacy) | | |
| الـ descriptor lifecycle | | ✓ | |
| الـ request/free GPIO | | ✓ | |
| تحديد الـ direction | | ✓ (validates) | ✓ (executes) |
| قراءة/كتابة الـ value | | ✓ (validates) | ✓ (executes) |
| الـ hardware register access | | | ✓ |
| الـ RCU protection للـ chip pointer | | ✓ | |

الـ sysfs layer بتـown الـ presentation layer بس — كل operation حقيقية بتنزل للـ core وبعدين للـ driver.

---

### الـ RCU Protection Pattern

الـ `struct gpio_device` بيحتفظ بـ pointer للـ `gpio_chip` محمي بـ **SRCU** (Sleepable RCU):

```c
struct gpio_device {
    struct gpio_chip __rcu *chip;   /* RCU-protected pointer */
    struct srcu_struct srcu;        /* SRCU domain */
    ...
};
```

ده مهم لأن الـ GPIO chip ممكن يتـunregister في أي وقت (لو كان على USB مثلاً)، والـ sysfs attributes ممكن لسه بتتكلم. الـ `gpio_chip_guard` في `gpiolib.h` بيعمل SRCU read lock بشكل أوتوماتيك:

```c
DEFINE_CLASS(gpio_chip_guard,
    struct gpio_chip_guard,
    srcu_read_unlock(&_T.gdev->srcu, _T.idx),   /* cleanup */
    ({
        struct gpio_chip_guard _guard;
        _guard.gdev = desc->gdev;
        _guard.idx  = srcu_read_lock(&_guard.gdev->srcu);
        _guard.gc   = srcu_dereference(_guard.gdev->chip,
                                       &_guard.gdev->srcu);
        _guard;
    }),
    struct gpio_desc *desc)
```

ده بيضمن إن الـ chip مش هيتـfree وإحنا بنشتغل عليه — وبيستخدم الـ `cleanup.h` **scoped guard** pattern اللي الـ kernel بدأ يعتمد عليه بدل الـ goto error handling.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags, Enums, وخيارات الـ Config — Cheatsheet

#### الـ `GPIOD_FLAG_*` — بيتات الـ `gpio_desc.flags`

| البيت | الاسم | المعنى |
|-------|-------|---------|
| 0 | `GPIOD_FLAG_REQUESTED` | الـ GPIO محجوز حاليًا |
| 1 | `GPIOD_FLAG_IS_OUT` | اتجاه الـ GPIO = output |
| 2 | `GPIOD_FLAG_EXPORT` | الـ GPIO مُصدَّر لـ userspace |
| 3 | `GPIOD_FLAG_SYSFS` | التصدير جاء عن طريق `/sys/class/gpio` تحديدًا |
| 6 | `GPIOD_FLAG_ACTIVE_LOW` | المنطق معكوس (active-low) |
| 7 | `GPIOD_FLAG_OPEN_DRAIN` | open-drain |
| 8 | `GPIOD_FLAG_OPEN_SOURCE` | open-source |
| 9 | `GPIOD_FLAG_USED_AS_IRQ` | متصل بـ IRQ |
| 10 | `GPIOD_FLAG_IRQ_IS_ENABLED` | الـ IRQ مفعّل |
| 11 | `GPIOD_FLAG_IS_HOGGED` | محجوز كـ hog في الـ device tree |
| 12 | `GPIOD_FLAG_TRANSITORY` | قد يضيع قيمته عند sleep/reset |
| 16 | `GPIOD_FLAG_EDGE_RISING` | sysfs يراقب rising edge |
| 17 | `GPIOD_FLAG_EDGE_FALLING` | sysfs يراقب falling edge |

#### الـ `GPIO_IRQF_TRIGGER_*` — trigger flags خاصة بـ sysfs (legacy فقط)

| الثابت | القيمة | المعنى |
|--------|--------|---------|
| `GPIO_IRQF_TRIGGER_NONE` | `0` | لا يوجد edge detection |
| `GPIO_IRQF_TRIGGER_FALLING` | `BIT(0)` | راقب الحافة الهابطة |
| `GPIO_IRQF_TRIGGER_RISING` | `BIT(1)` | راقب الحافة الصاعدة |
| `GPIO_IRQF_TRIGGER_BOTH` | `BIT(0)\|BIT(1)` | راقب الاتجاهين |

#### الـ Enums لترتيب الـ sysfs attributes

```
GPIO_SYSFS_LINE_CLASS_ATTR_*  (legacy — /sys/class/gpio/gpioN/)
  DIRECTION=0, VALUE=1, EDGE=2, ACTIVE_LOW=3, SENTINEL=4, SIZE=5

GPIO_SYSFS_LINE_CHIP_ATTR_*   (chip interface — /sys/class/gpio/chipX/)
  DIRECTION=0, VALUE=1, SENTINEL=2, SIZE=3
```

#### خيارات الـ Kconfig المؤثرة

| الخيار | الأثر |
|--------|-------|
| `CONFIG_GPIO_SYSFS` | يُفعّل الـ sysfs interface كله — بدونه الدوال تصبح stubs فارغة |
| `CONFIG_GPIO_SYSFS_LEGACY` | يُفعّل الـ `/sys/class/gpio/gpioN/` القديم + global export/unexport + `edge` و `active_low` attributes |

---

### الـ Structs الأساسية

#### 1. `struct gpiod_data`

**الغرض:** حاوية الحالة الكاملة لـ GPIO line واحد مُصدَّر عبر sysfs. واحد لكل `gpio_desc` مصدَّر.

```c
struct gpiod_data {
    struct list_head list;           /* ربطه في قائمة exported_lines */

    struct gpio_desc *desc;          /* المؤشر للـ descriptor الفعلي */
    struct device    *dev;           /* الـ class device (legacy: /sys/class/gpio/gpioN) */

    struct mutex mutex;              /* يحمي كل عمليات read/write على هذا الـ GPIO */

    /* Legacy only */
    struct kernfs_node *value_kn;   /* kernfs node للـ "value" — لإرسال poll notifications */
    int    irq;                      /* رقم الـ IRQ المرتبط */
    unsigned char irq_flags;         /* GPIO_IRQF_TRIGGER_* الحالي */

    bool direction_can_change;       /* هل يسمح الـ chip بتغيير الاتجاه؟ */

    struct kobject *parent;          /* parent kobject في chipX — لإضافة attr groups */

    /* الـ sysfs attributes المدارة ديناميكيًا */
    struct device_attribute dir_attr;         /* "direction" */
    struct device_attribute val_attr;         /* "value" */
    struct device_attribute edge_attr;        /* "edge" (legacy) */
    struct device_attribute active_low_attr;  /* "active_low" (legacy) */

    /* Legacy: attr group للـ /sys/class/gpio/gpioN/ */
    struct attribute        *class_attrs[GPIO_SYSFS_LINE_CLASS_ATTR_SIZE];
    struct attribute_group   class_attr_group;
    const struct attribute_group *class_attr_groups[2];

    /* Chip interface: attr group تحت /sys/class/gpio/chipX/gpioN/ */
    struct attribute        *chip_attrs[GPIO_SYSFS_LINE_CHIP_ATTR_SIZE];
    struct attribute_group   chip_attr_group;
    const struct attribute_group *chip_attr_groups[2];
};
```

**الاتصالات:**
- يشير إلى `gpio_desc` (الـ descriptor الفعلي للـ GPIO)
- يُضاف في قائمة `gpiodev_data.exported_lines`
- الـ **`dev`** يشير لـ `struct device` في `gpio_class` (legacy فقط)

---

#### 2. `struct gpiodev_data`

**الغرض:** حاوية sysfs على مستوى chip كامل — تُنشأ مرة واحدة لكل `gpio_device` عند تسجيله.

```c
struct gpiodev_data {
    struct list_head  exported_lines;  /* قائمة كل gpiod_data المُصدَّرة من هذا الـ chip */
    struct gpio_device *gdev;          /* back-pointer للـ gpio_device الأصلي */
    struct device *cdev_id;            /* /sys/class/gpio/chipX (جديد، بالـ device ID) */
    struct device *cdev_base;          /* /sys/class/gpio/gpiochipN (legacy، بالـ base number) */
};
```

**الاتصالات:**
- يشير إلى `gpio_device` (الـ GPIO device في الـ kernel)
- الـ `cdev_id` و `cdev_base` هم `struct device` objects مسجلة في `gpio_class`
- يُخزَّن كـ driver data في الـ class devices عبر `dev_set_drvdata`

---

#### 3. `struct gpio_desc`

**الغرض:** الـ descriptor الأساسي لكل GPIO line — معرَّف في `gpiolib.h`.

```c
struct gpio_desc {
    struct gpio_device *gdev;   /* الـ GPIO device الذي ينتمي إليه */
    unsigned long       flags;  /* GPIOD_FLAG_* bitmask */
    struct gpio_desc_label __rcu *label;  /* اسم المستهلك */
    const char *name;           /* اسم الـ line */
};
```

---

#### 4. `struct gpio_device`

**الغرض:** الـ container الداخلي للـ GPIO controller — يعيش بعد رحيل الـ chip driver.

```c
struct gpio_device {
    struct device        dev;       /* الـ device في نظام الـ driver model */
    struct cdev          chrdev;    /* character device للـ /dev/gpiochipN */
    int                  id;        /* رقم الـ device (لـ chipX في sysfs) */
    struct gpio_chip __rcu *chip;   /* مؤشر للـ chip driver (محمي بـ SRCU) */
    struct gpio_desc    *descs;     /* مصفوفة الـ descriptors */
    unsigned int         base;      /* الرقم الأساسي (legacy numbering) */
    u16                  ngpio;     /* عدد الـ lines */
    const char          *label;     /* اسم الـ chip */
    struct srcu_struct   srcu;      /* يحمي مؤشر الـ chip */
};
```

---

#### 5. `struct gpio_class` (static)

```c
static const struct class gpio_class = {
    .name = "gpio",
    /* legacy: .class_groups = gpio_class_groups (export/unexport) */
};
```

**الغرض:** تسجيل الـ `/sys/class/gpio/` في kernel class system — كل devices الـ sysfs GPIO تنتمي لهذا الـ class.

---

### مخطط علاقات الـ Structs (ASCII)

```
                        ┌─────────────────────────────────────────────┐
                        │              gpio_class                      │
                        │         /sys/class/gpio/                     │
                        └──────────────────┬──────────────────────────┘
                                           │ يحتوي على devices
                    ┌──────────────────────┴───────────────────────────┐
                    │                                                   │
          ┌─────────▼──────────┐                           ┌───────────▼──────────┐
          │  device (cdev_id)  │                           │  device (cdev_base)  │
          │  /sys/.../chipX    │                           │ /sys/.../gpiochipN   │
          │  driver_data ──►───┼────────┐  [legacy only]  │  driver_data ──►─────┼───┐
          └────────────────────┘        │                  └──────────────────────┘   │
                                        │                                             │
                                        ▼ (same pointer) ◄───────────────────────────┘
                               ┌────────────────────┐
                               │   gpiodev_data     │
                               │ ┌────────────────┐ │
                               │ │ exported_lines │─┼──► list of gpiod_data
                               │ └────────────────┘ │
                               │  gdev ─────────────┼──────────────────────────┐
                               │  cdev_id           │                           │
                               │  cdev_base         │                           ▼
                               └────────────────────┘              ┌──────────────────────┐
                                                                   │    gpio_device       │
                                                                   │  dev, id, ngpio      │
                                                                   │  chip (SRCU ptr) ──► gpio_chip
                                                                   │  descs[] ──────────► gpio_desc[]
                                                                   └──────────────────────┘

         ┌────────────────────────────────────────────────────────────────┐
         │  gpiod_data  (واحد لكل GPIO line مُصدَّر)                      │
         │                                                                │
         │  list ─────────────────────────────────────────────────────── │ ─► (linked in exported_lines)
         │  desc ──────────────────────────────────────────────────────► │ gpio_desc
         │  dev  ──────────────────────────────────────────────────────► │ device (/sys/class/gpio/gpioN) [legacy]
         │  mutex                                                         │
         │  value_kn ─────────────────────────────────────────────────► │ kernfs_node ("value") [legacy]
         │  irq, irq_flags                                                │
         │  direction_can_change                                          │
         │  parent ────────────────────────────────────────────────────► │ kobject (chipX)
         │  dir_attr, val_attr, edge_attr, active_low_attr                │
         │  chip_attr_group (name="gpioN", attrs=[dir,val])               │
         └────────────────────────────────────────────────────────────────┘
```

---

### مخطط دورة حياة الـ GPIO Chip في sysfs

```
Boot / module_init
       │
       ▼
gpiolib_sysfs_init()   [postcore_initcall]
       │
       ├─► class_register(&gpio_class)
       │       └── ينشئ /sys/class/gpio/
       │
       └─► gpio_device_find(NULL, gpiofind_sysfs_register)
               └── لكل chip مسجّل مبكرًا:
                       └── gpiochip_sysfs_register(gdev)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
gpiochip_sysfs_register(gdev)
       │
       ├─► kmalloc(gpiodev_data)
       ├─► INIT_LIST_HEAD(&data->exported_lines)
       ├─► [legacy] device_create_with_groups → cdev_base
       │       /sys/class/gpio/gpiochipN/
       │       attrs: base, label, ngpio
       │
       └─► device_create_with_groups → cdev_id
               /sys/class/gpio/chipX/
               attrs: label, ngpio, export(wo), unexport(wo)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
gpiochip_sysfs_unregister(gdev)
       │
       ├─► scoped_guard(sysfs_lock):
       │       ├─► [legacy] device_unregister(cdev_base)
       │       ├─► device_unregister(cdev_id)
       │       └─► kfree(gpiodev_data)
       │
       └─► for_each_gpio_desc_with_flag(GPIOD_FLAG_SYSFS):
               ├─► gpiod_unexport(desc)
               └─► gpiod_free(desc)
```

---

### مخطط دورة حياة تصدير GPIO Line واحد

```
userspace يكتب رقم في export
         │
         ▼
[legacy] export_store()           أو     chip_export_store()
         │                                      │
         └──────────────────┬───────────────────┘
                            ▼
                  export_gpio_desc(desc)
                            │
                            ├─► gpiod_request_user(desc, "sysfs")
                            ├─► gpiod_set_transitory(desc, false)
                            ├─► gpiod_export(desc, true)
                            │       └─► set GPIOD_FLAG_SYSFS
                            └─► gpiod_line_state_notify(REQUESTED)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
gpiod_export(desc, direction_may_change)
         │
         ├─► test_and_set_bit(GPIOD_FLAG_EXPORT)    [atomic, no double export]
         ├─► guard(sysfs_lock)
         ├─► kzalloc(gpiod_data)
         ├─► mutex_init(&desc_data->mutex)
         ├─► gpiod_attr_init(dir_attr, val_attr, ...)
         │
         ├─► [legacy] device_create_with_groups → /sys/class/gpio/gpioN/
         │       attrs: direction, value, edge(*), active_low(*)
         │       (*) مرئية فقط إذا gpio_is_visible() وافق
         │
         ├─► [legacy] sysfs_get_dirent("value") → value_kn   [لـ poll notify]
         │
         ├─► gdev_get_data(gdev)   [الحصول على gpiodev_data]
         │
         ├─► kasprintf("gpioN") → chip_attr_group.name
         ├─► sysfs_create_groups(parent_kobject, chip_attr_groups)
         │       ينشئ: /sys/class/gpio/chipX/gpioN/{direction,value}
         │
         └─► list_add(&desc_data->list, &gdev_data->exported_lines)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
gpiod_unexport(desc)
         │
         ├─► scoped_guard(sysfs_lock)
         │       ├─► list_for_each_entry → إيجاد gpiod_data
         │       ├─► list_del(&desc_data->list)
         │       ├─► clear_bit(GPIOD_FLAG_EXPORT)
         │       ├─► [legacy] sysfs_put(value_kn)
         │       ├─► [legacy] device_unregister(dev)
         │       ├─► [legacy] gpio_sysfs_free_irq() إذا كان هناك IRQ
         │       └─► sysfs_remove_groups(parent, chip_attr_groups)
         │
         ├─► mutex_destroy(&desc_data->mutex)
         └─► kfree(desc_data)
```

---

### مخططات تدفق الـ Call Flows

#### قراءة direction

```
userspace: cat /sys/class/gpio/chipX/gpioN/direction
    │
    ▼
direction_show(dev, attr, buf)
    │
    ├─► container_of(attr, gpiod_data, dir_attr)  → يحصل على gpiod_data
    ├─► scoped_guard(mutex, &data->mutex)
    │       ├─► gpiod_get_direction(desc)
    │       │       └─► gpio_chip.ops->get_direction(gc, offset)  → hardware
    │       └─► test_bit(GPIOD_FLAG_IS_OUT, &desc->flags)
    │
    └─► sysfs_emit(buf, "in" أو "out")
```

#### كتابة value

```
userspace: echo 1 > /sys/class/gpio/chipX/gpioN/value
    │
    ▼
value_store(dev, attr, buf, size)
    │
    ├─► container_of(attr, gpiod_data, val_attr)
    ├─► kstrtol(buf, 0, &value)
    ├─► guard(mutex)(&data->mutex)
    └─► gpiod_set_value_cansleep(desc, value)
            │
            └─► gpio_chip.ops->set(gc, offset, value)   → hardware register
```

#### تغيير edge (legacy) وعلاقته بالـ IRQ

```
userspace: echo falling > /sys/class/gpio/gpioN/edge
    │
    ▼
edge_store()
    │
    ├─► sysfs_match_string(trigger_names, buf) → flags
    ├─► guard(mutex)(&data->mutex)
    ├─► إذا كان هناك irq قديم: gpio_sysfs_free_irq(data)
    │       ├─► free_irq(data->irq, data)
    │       ├─► gpiochip_unlock_as_irq(gc, offset)
    │       └─► clear_bit(GPIOD_FLAG_EDGE_RISING/FALLING)
    │
    └─► gpio_sysfs_request_irq(data, flags)
            │
            ├─► CLASS(gpio_chip_guard, guard)(desc)  [SRCU lock]
            ├─► gpiod_to_irq(desc)              → رقم الـ IRQ من الـ chip
            ├─► gpiochip_lock_as_irq(gc, offset)
            ├─► request_any_context_irq(irq, gpio_sysfs_irq, irq_flags, ...)
            │       └─► عند وقوع الـ edge:
            │               gpio_sysfs_irq(irq, data)
            │                   └─► sysfs_notify_dirent(value_kn)
            │                           └─► يوقظ poll() في userspace
            └─► data->irq_flags = flags

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
active_low effect على IRQ polarity:
    active_low=1 + edge=falling  →  IRQ يُطلق على RISING hardware edge
    active_low=1 + edge=rising   →  IRQ يُطلق على FALLING hardware edge
    (المنطق مقلوب في gpio_sysfs_request_irq عند test_bit GPIOD_FLAG_ACTIVE_LOW)
```

---

### استراتيجية الـ Locking

#### الـ Locks المستخدمة وما تحميه

| الـ Lock | النوع | يحمي ماذا؟ |
|----------|-------|------------|
| `sysfs_lock` | `DEFINE_MUTEX` (global, static) | عمليات export/unexport بالكامل — يمنع race بين التصدير ورحيل الـ chip، ويحمي قائمة `exported_lines` |
| `gpiod_data.mutex` | `struct mutex` (per GPIO line) | كل عمليات القراءة والكتابة على GPIO line واحد: direction, value, edge, active_low, irq_flags |
| `gpio_device.srcu` | SRCU (per gpio_device) | يحمي مؤشر `gdev->chip` من الاختفاء أثناء الاستخدام |

#### ترتيب الـ Locks (Lock Ordering)

```
sysfs_lock (global mutex)
    └─► gpiod_data.mutex (per-line mutex)
            └─► gpio_chip_guard SRCU (read lock)
                    └─► hardware register access
```

**قاعدة صارمة:** لا يجوز أخذ `sysfs_lock` بعد `gpiod_data.mutex` — يجب دائمًا الترتيب من الأعلى للأسفل.

#### ملاحظات مهمة على الـ Locking

- **`GPIOD_FLAG_EXPORT`:** يُضبط atomically بـ `test_and_set_bit` قبل أخذ `sysfs_lock` — هذا يمنع double-export بدون lock كامل.
- **`GPIOD_FLAG_SYSFS`:** لا يحتاج lock إضافي — التعليق في الكود يوضح أنه مجرد علامة لتتبع من طلب التصدير.
- **IRQ free عند unexport:** يتم بعد `device_unregister()` داخل `sysfs_lock` لتجنب race مع `edge_store()` الذي يمكن أن يُستدعى من thread آخر أثناء unregister.
- **`gpio_chip_guard`:** يستخدم SRCU لأن عمليات الـ GPIO قد تنام — `gpiochip_lock_as_irq` يحتاج هذا الـ guard ليضمن أن الـ chip لم يُزال في منتصف العملية.
- **`scoped_guard` vs `guard`:** كلاهما RAII — `scoped_guard` يحرر الـ lock في نهاية الـ block، `guard` يحرره بنهاية الدالة.
- **`gdev_get_data()`:** مُعلَّمة بـ `__must_hold(&sysfs_lock)` — يجب أن `sysfs_lock` محجوز قبل استدعائها.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ Public API (exported via EXPORT_SYMBOL_GPL)

| Function | Signature | الغرض |
|---|---|---|
| `gpiod_export` | `int gpiod_export(struct gpio_desc *desc, bool direction_may_change)` | تعمل export لـ GPIO عبر sysfs |
| `gpiod_export_link` | `int gpiod_export_link(struct device *dev, const char *name, struct gpio_desc *desc)` | تعمل symlink في sysfs لـ GPIO مُعمَّل export |
| `gpiod_unexport` | `void gpiod_unexport(struct gpio_desc *desc)` | تزيل الـ GPIO من sysfs |
| `gpiochip_sysfs_register` | `int gpiochip_sysfs_register(struct gpio_device *gdev)` | تسجّل الـ gpiochip في الـ gpio_class |
| `gpiochip_sysfs_unregister` | `void gpiochip_sysfs_unregister(struct gpio_device *gdev)` | تلغي تسجيل الـ gpiochip وتحرر موارده |

#### الـ sysfs Attribute Handlers

| Function | الـ Attribute | Mode | الغرض |
|---|---|---|---|
| `direction_show` | `direction` | r | تقرأ الاتجاه (in/out) |
| `direction_store` | `direction` | w | تكتب الاتجاه (in/out/high/low) |
| `value_show` | `value` | r | تقرأ قيمة الـ GPIO |
| `value_store` | `value` | w | تكتب قيمة الـ GPIO |
| `edge_show` | `edge` (legacy) | r | تقرأ trigger edge |
| `edge_store` | `edge` (legacy) | w | تكتب trigger edge |
| `active_low_show` | `active_low` (legacy) | r | تقرأ polarity |
| `active_low_store` | `active_low` (legacy) | w | تكتب polarity |
| `base_show` | `base` (legacy) | r | تقرأ gpio_chip base |
| `label_show` | `label` | r | تقرأ label الـ chip |
| `ngpio_show` | `ngpio` | r | تقرأ عدد الـ GPIOs |
| `export_store` | `export` (legacy class) | w | تعمل export بـ GPIO number |
| `unexport_store` | `unexport` (legacy class) | w | تعمل unexport بـ GPIO number |
| `chip_export_store` | `export` (chip device) | w | تعمل export بـ offset |
| `chip_unexport_store` | `unexport` (chip device) | w | تعمل unexport بـ offset |

#### الـ Internal Helpers

| Function | الغرض |
|---|---|
| `gpio_sysfs_irq` | IRQ handler — يُبلّغ عن edge event |
| `gpio_sysfs_request_irq` | يطلب IRQ ويضبط edge triggering |
| `gpio_sysfs_free_irq` | يحرر IRQ ويُلغي edge bits |
| `gpio_sysfs_set_active_low` | يضبط active_low ويُعيد تهيئة IRQ |
| `gpio_is_visible` | يتحكم في ظهور الـ attributes بالـ kobject |
| `export_gpio_desc` | منطق الـ export الداخلي |
| `unexport_gpio_desc` | منطق الـ unexport الداخلي |
| `do_chip_export_store` | helper مشترك لـ chip export/unexport |
| `match_gdev` | callback لـ class_find_device بالـ gdev |
| `gdev_get_data` | يجيب الـ gpiodev_data من الـ gpio_class |
| `gpiod_attr_init` | يهيئ device_attribute dynamically |
| `match_export` | callback لإيجاد الـ class device بـ desc |
| `gpiofind_sysfs_register` | callback لـ gpio_device_find عند الـ init |
| `gpiolib_sysfs_init` | `postcore_initcall` — يسجّل الـ gpio_class |

---

### Group 1: التسجيل وإلغاء التسجيل (Registration / Cleanup)

هذه المجموعة مسؤولة عن ربط وفك ربط الـ `gpio_device` بالـ sysfs class، وتعمل على مستوى الـ chip كله وليس على مستوى line منفردة.

---

#### `gpiochip_sysfs_register`

```c
int gpiochip_sysfs_register(struct gpio_device *gdev)
```

**ما تعمله:** تُنشئ class devices في `/sys/class/gpio/` للـ gpio chip. في الوضع legacy تنشئ `gpiochipN` و`chipX`، وفي الوضع الحديث تنشئ `chipX` فقط. إذا لم تكن الـ `gpio_class` مسجّلة بعد (early boot)، ترجع `0` بدون عمل شيء — سيتم التسجيل لاحقاً من `gpiolib_sysfs_init`.

**Parameters:**
- `gdev` — الـ `gpio_device` المراد تسجيله، يحتوي على `id`, `chip`, `srcu`.

**Return value:** `0` عند النجاح، أو كود خطأ سلبي (ENOMEM, ENODEV).

**Key details:**
- يأخذ `guard(srcu)` على `gdev->srcu` لضمان أن الـ `chip` pointer صالح خلال العملية.
- يأخذ `guard(mutex)(&sysfs_lock)` لمنع التسابق مع العمليات المتزامنة.
- يُخصص `gpiodev_data` ويُهيئ `exported_lines` list.
- في legacy mode: ينشئ device باسم `gpiochipN` (N = chip->base) مع `gpiochip_groups`.
- في الوضع الحديث: ينشئ device باسم `chipX` (X = gdev->id) مع `gpiochip_ext_groups` التي تحتوي على `export`/`unexport` attributes.
- الـ parent يُفضَّل `chip->parent` إن وُجد، وإلا `&gdev->dev`.

**Caller:** يُستدعى من `gpiolib_sysfs_init` عبر `gpiofind_sysfs_register`، ومن gpiolib core عند تسجيل chip جديد.

**Pseudocode:**
```
gpiochip_sysfs_register(gdev):
    if gpio_class not registered → return 0  // too early
    srcu_read_lock(gdev->srcu)
    chip = srcu_dereference(gdev->chip)
    if !chip → return -ENODEV
    allocate gpiodev_data
    mutex_lock(sysfs_lock)
    [legacy] create device "gpiochipN" with gpiochip_groups
    create device "chipX" with gpiochip_ext_groups
    return 0
```

---

#### `gpiochip_sysfs_unregister`

```c
void gpiochip_sysfs_unregister(struct gpio_device *gdev)
```

**ما تعمله:** تُزيل الـ class devices من sysfs وتحرر كل الـ GPIOs المُعمَّل عليها export من userspace (`GPIOD_FLAG_SYSFS`). تُعالج أيضاً الحالة التي تُزال فيها الـ chip قبل أن يعمل userspace unexport.

**Parameters:**
- `gdev` — الـ `gpio_device` المراد فك تسجيله.

**Return value:** لا يرجع شيء.

**Key details:**
- يأخذ `scoped_guard(mutex, &sysfs_lock)` لإيجاد وإزالة الـ `gpiodev_data`.
- يستخدم `gdev_get_data` داخلياً (يتطلب `sysfs_lock` محجوزاً).
- بعد تحرير الـ sysfs_lock يأخذ SRCU lock ويمشي على كل الـ descriptors بـ `GPIOD_FLAG_SYSFS` فيعمل `gpiod_unexport` ثم `gpiod_free`.
- الترتيب مهم: `device_unregister` أولاً ثم cleanup الـ GPIOs لمنع race مع edge_store.

**Caller:** يُستدعى من gpiolib core عند unregister الـ chip (مثلاً عند hot-unplug).

---

#### `gpiolib_sysfs_init`

```c
static int __init gpiolib_sysfs_init(void)
```

**ما تعمله:** تُسجّل `gpio_class` في نظام الـ device model، ثم تمشي على كل الـ chips المسجّلة مسبقاً (early boot) وتعمل sysfs registration لها.

**Key details:**
- مُعلَّمة بـ `postcore_initcall` — تعمل قبل `arch_initcall` حتى يكون sysfs جاهزاً عند init الـ architecture.
- تستخدم `gpio_device_find(NULL, gpiofind_sysfs_register)` للتمشية على الكل.
- الـ `gpio_class` تُعرَّف كـ `static const struct class` مع اسم `"gpio"`.

---

### Group 2: الـ Export/Unexport (GPIO Line Visibility)

هذه المجموعة تتعامل مع إظهار وإخفاء GPIOs منفردة في sysfs.

---

#### `gpiod_export`

```c
int gpiod_export(struct gpio_desc *desc, bool direction_may_change)
```

**ما تعمله:** تُنشئ sysfs entries لـ GPIO line واحدة. تُخصص `gpiod_data`، تُهيئ الـ attributes، تُنشئ class device (legacy) و/أو attribute group تحت `chipX` kobject. تضع `GPIOD_FLAG_EXPORT` في flags الـ descriptor.

**Parameters:**
- `desc` — الـ `gpio_desc` المطلوب تصديره، يجب أن يكون مطلوباً مسبقاً (`GPIOD_FLAG_REQUESTED`).
- `direction_may_change` — هل يُسمح لـ userspace بتغيير الاتجاه؟ يُضبط فقط إذا كان الـ chip يدعم كلاً من `direction_input` و`direction_output`.

**Return value:** `0` عند النجاح، أو:
- `-ENOENT` إذا لم تُسجَّل `gpio_class` بعد.
- `-EINVAL` إذا كان `desc == NULL`.
- `-ENODEV` إذا لم يكن الـ chip موجوداً.
- `-EPERM` إذا كان الـ GPIO مُعمَّل عليه export مسبقاً أو غير مطلوب.
- `-ENOMEM` في حالة فشل allocation.

**Key details:**
- يستخدم `test_and_set_bit(GPIOD_FLAG_EXPORT, ...)` للحماية من export مزدوج — atomic operation.
- الـ `sysfs_lock` mutex يُحجز طوال العملية لمنع التسابق مع gpiochip_sysfs_unregister.
- في legacy mode: ينشئ `/sys/class/gpio/gpioN/` مع attributes: `direction`, `value`, `edge`, `active_low`.
- دائماً: ينشئ attribute group `gpio<offset>` تحت `/sys/class/gpio/chipX/`.
- يُضيف `desc_data` لـ `gdev_data->exported_lines` list.
- خطأ exit path مُرتَّب بعناية (labeled gotos) لتنظيف الموارد بشكل صحيح.

**Caller:** يُستدعى من `export_gpio_desc` (sysfs write)، ومن drivers مباشرة بعد `gpiod_request`.

**Pseudocode:**
```
gpiod_export(desc, direction_may_change):
    check gpio_class registered
    check desc != NULL
    guard gpio_chip
    atomic set GPIOD_FLAG_EXPORT (fail if already set)
    mutex_lock(sysfs_lock)
    check GPIOD_FLAG_REQUESTED
    allocate gpiod_data
    init mutex, direction_can_change
    init dir_attr, val_attr (+ edge_attr, active_low_attr in legacy)
    [legacy] device_create_with_groups → /sys/class/gpio/gpioN/
    [legacy] get value_kn kernfs_node
    get gdev_data from class
    kasprintf chip_attr_group.name = "gpio<offset>"
    sysfs_create_groups under chipX kobject
    list_add to exported_lines
    return 0
on error:
    cleanup in reverse order → clear GPIOD_FLAG_EXPORT
```

---

#### `gpiod_export_link`

```c
int gpiod_export_link(struct device *dev, const char *name,
                      struct gpio_desc *desc)
```

**ما تعمله:** تُنشئ symlink في `/sys/.../dev/name` يُشير لـ `/sys/class/gpio/gpioN` الخاص بالـ GPIO المُعمَّل export. متاحة في legacy mode فقط.

**Parameters:**
- `dev` — الـ device الذي سيحتوي الـ symlink تحت kobject الخاص به.
- `name` — اسم الـ symlink.
- `desc` — الـ GPIO descriptor المُعمَّل export مسبقاً.

**Return value:** `0` أو كود خطأ. في حالة عدم تفعيل legacy mode يُرجع `-EOPNOTSUPP`.

**Key details:**
- يستخدم `class_find_device` مع `match_export` للبحث عن الـ class device المقابل للـ desc.
- `sysfs_create_link` تُنشئ الـ symlink الفعلي.
- المستدعي مسؤول عن حذف الـ symlink لاحقاً بـ `sysfs_remove_link`.

---

#### `gpiod_unexport`

```c
void gpiod_unexport(struct gpio_desc *desc)
```

**ما تعمله:** عكس `gpiod_export` — تُزيل كل sysfs entries الخاصة بالـ GPIO line وتُحرر `gpiod_data`. تُلغي `GPIOD_FLAG_EXPORT` وتحرر IRQ إن كان مضبوطاً (legacy).

**Parameters:**
- `desc` — الـ `gpio_desc` المراد إخفاؤه.

**Return value:** لا يرجع شيء.

**Key details:**
- يبحث عن `desc_data` في `exported_lines` list عبر `gpiod_is_equal`.
- في legacy: يفعل `sysfs_put(value_kn)` ثم `device_unregister` ثم `gpio_sysfs_free_irq` — الترتيب منطقي لمنع race مع poll(2).
- `mutex_destroy` و`kfree` يحدثان خارج `sysfs_lock` scope لتجنب deadlock.
- `scoped_guard` يضمن تحرير الـ lock حتى في حالات الـ early return.

**Caller:** يُستدعى من `unexport_gpio_desc`، ومن `gpiochip_sysfs_unregister`، وضمنياً من `gpiod_free`.

---

#### `export_gpio_desc`

```c
static int export_gpio_desc(struct gpio_desc *desc)
```

**ما تعمله:** wrapper داخلي يتحقق من صحة الـ line (valid + not masked)، يطلب الـ GPIO باسم `"sysfs"`، يمنع الـ transient flag، ثم يستدعي `gpiod_export`.

**Key details:**
- يستخدم `gpiochip_line_is_valid` للتأكد من أن الـ GPIO غير masked في الـ chip.
- يضبط `GPIOD_FLAG_SYSFS` بعد نجاح الـ export، ويُرسل `GPIO_V2_LINE_CHANGED_REQUESTED` notification.
- `gpiod_set_transitory(desc, false)` يمنع تحرير الـ GPIO عند power management transitions.

---

#### `unexport_gpio_desc`

```c
static int unexport_gpio_desc(struct gpio_desc *desc)
```

**ما تعمله:** يتحقق من `GPIOD_FLAG_SYSFS` بـ `test_and_clear_bit` atomically، ثم يستدعي `gpiod_unexport` و`gpiod_free`.

**Return value:** `-EINVAL` إذا لم يكن الـ GPIO مُعمَّل export من sysfs.

---

#### `do_chip_export_store`

```c
static ssize_t do_chip_export_store(struct device *dev,
                                    struct device_attribute *attr,
                                    const char *buf, ssize_t size,
                                    int (*handler)(struct gpio_desc *desc))
```

**ما تعمله:** helper مشترك لـ `chip_export_store` و`chip_unexport_store`. يقرأ رقم الـ offset من الـ user، يجيب الـ `gpio_desc` من الـ `gpio_device`، ثم يستدعي الـ `handler` المُمرَّر.

**Parameters:**
- `handler` — function pointer يُشير إلى `export_gpio_desc` أو `unexport_gpio_desc`.

---

### Group 3: الـ sysfs Attribute Handlers (Line-Level)

كل هذه الـ functions هي `show`/`store` callbacks تُستدعى من الـ VFS عبر الـ sysfs layer.

---

#### `direction_show` / `direction_store`

```c
static ssize_t direction_show(struct device *dev,
                              struct device_attribute *attr, char *buf)

static ssize_t direction_store(struct device *dev,
                               struct device_attribute *attr,
                               const char *buf, size_t size)
```

**الـ show:** يقرأ اتجاه الـ GPIO الحالي بـ `gpiod_get_direction` ثم يفحص `GPIOD_FLAG_IS_OUT`. يكتب `"in\n"` أو `"out\n"` في الـ buffer.

**الـ store:** يقبل أربع قيم:
- `"high"` → `gpiod_direction_output_raw(desc, 1)` — output مستوى عالي
- `"out"` أو `"low"` → `gpiod_direction_output_raw(desc, 0)` — output مستوى منخفض
- `"in"` → `gpiod_direction_input(desc)`

**Locking:** كلاهما يأخذ `data->mutex` لمنع التسابق مع قراءات/كتابات متزامنة.

**Container:** يستخدم `container_of(attr, struct gpiod_data, dir_attr)` للوصول لـ `gpiod_data` من الـ `device_attribute`.

---

#### `value_show` / `value_store`

```c
static ssize_t value_show(struct device *dev, struct device_attribute *attr,
                          char *buf)

static ssize_t value_store(struct device *dev, struct device_attribute *attr,
                           const char *buf, size_t size)
```

**الـ show:** يستدعي `gpiod_get_value_cansleep` ضمن `scoped_guard(mutex)`. يكتب `0` أو `1` في الـ buffer. النتيجة تأخذ `active_low` في الاعتبار (عبر gpiolib layer).

**الـ store:** يحوّل الـ string لـ `long` بـ `kstrtol`، ثم يستدعي `gpiod_set_value_cansleep`. يعمل في sleeping context (مناسب لـ I2C GPIO expanders).

**Key details:** استخدام `cansleep` variants مهم لأن الـ sysfs callbacks تعمل في process context ويُسمح بالـ sleep.

---

#### `edge_show` / `edge_store` (Legacy only)

```c
static ssize_t edge_show(struct device *dev, struct device_attribute *attr,
                         char *buf)

static ssize_t edge_store(struct device *dev, struct device_attribute *attr,
                          const char *buf, size_t size)
```

**الـ show:** يقرأ `data->irq_flags` ويُرجعه كـ string من `trigger_names[]`: `"none"`, `"falling"`, `"rising"`, `"both"`.

**الـ store:** يُطابق الـ input بـ `sysfs_match_string`. إذا تغيّر الـ flags:
1. يحرر الـ IRQ القديم بـ `gpio_sysfs_free_irq` إن كان موجوداً.
2. يطلب IRQ جديد بـ `gpio_sysfs_request_irq`.
3. يُرسل `GPIO_V2_LINE_CHANGED_CONFIG` notification.

**Locking:** `guard(mutex)(&data->mutex)` طوال العملية.

---

#### `active_low_show` / `active_low_store` (Legacy only)

```c
static ssize_t active_low_show(struct device *dev,
                               struct device_attribute *attr, char *buf)

static ssize_t active_low_store(struct device *dev,
                                struct device_attribute *attr,
                                const char *buf, size_t size)
```

**الـ show:** يفحص `GPIOD_FLAG_ACTIVE_LOW` في `desc->flags`.

**الـ store:** يستدعي `gpio_sysfs_set_active_low` بعد تحويل الـ input لـ `long`.

---

#### `base_show` / `label_show` / `ngpio_show` (Chip-level)

```c
static ssize_t base_show(struct device *dev, struct device_attribute *attr, char *buf)  // legacy
static ssize_t label_show(struct device *dev, struct device_attribute *attr, char *buf)
static ssize_t ngpio_show(struct device *dev, struct device_attribute *attr, char *buf)
```

**ما تعمل:** تقرأ الـ `gpiodev_data` من `dev_get_drvdata(dev)` وتُرجع `gdev->base` (legacy فقط)، `gdev->label`، أو `gdev->ngpio`.

---

#### `export_store` / `unexport_store` (Legacy Class-level)

```c
static ssize_t export_store(const struct class *class,
                            const struct class_attribute *attr,
                            const char *buf, size_t len)

static ssize_t unexport_store(const struct class *class,
                              const struct class_attribute *attr,
                              const char *buf, size_t len)
```

**ما تعمل:** class-level attributes في `/sys/class/gpio/export` و`/sys/class/gpio/unexport`. تقبل GPIO number الكلي (base + offset)، تجيب الـ `gpio_desc` بـ `gpio_to_desc(gpio)` وتستدعي `export_gpio_desc` أو `unexport_gpio_desc`.

**ملاحظة:** هذه الـ interface قديمة (legacy) وتعتمد على الـ GPIO number العالمي الذي يعتمد على `chip->base` الذي قد يتغيّر بين reboots.

---

### Group 4: الـ IRQ Management (Legacy only)

هذه المجموعة تُتيح لـ userspace استخدام `poll(2)` على `/sys/class/gpio/gpioN/value` للـ edge detection.

---

#### `gpio_sysfs_irq`

```c
static irqreturn_t gpio_sysfs_irq(int irq, void *priv)
```

**ما تعمله:** الـ IRQ handler. عند وقوع edge، يستدعي `sysfs_notify_dirent(data->value_kn)` لإيقاظ أي `poll(2)` منتظر على الـ `value` file.

**Parameters:**
- `irq` — رقم الـ IRQ.
- `priv` — مؤشر لـ `gpiod_data`.

**Return value:** `IRQ_HANDLED` دائماً.

**Key details:** يعمل في interrupt context — لا يجوز sleep. `value_kn` هو `kernfs_node` للـ `value` attribute، يُحفظ في `gpiod_data` عند الـ export.

---

#### `gpio_sysfs_request_irq`

```c
static int gpio_sysfs_request_irq(struct gpiod_data *data, unsigned char flags)
```

**ما تعمله:** تطلب IRQ للـ GPIO وتضبط edge triggering. تأخذ في الاعتبار `active_low` عند تحديد الـ trigger direction.

**Parameters:**
- `data` — الـ `gpiod_data` للـ GPIO (يحتوي على الـ mutex المحجوز).
- `flags` — مزيج من `GPIO_IRQF_TRIGGER_FALLING` و`GPIO_IRQF_TRIGGER_RISING`.

**Return value:** `0` أو كود خطأ سلبي.

**Key details:**
- `gpiod_to_irq(desc)` يحوّل الـ GPIO لـ Linux IRQ number.
- إذا كان `GPIOD_FLAG_ACTIVE_LOW` مضبوطاً، تُعكس الـ trigger direction (falling↔rising).
- `gpiochip_lock_as_irq` يمنع الـ GPIO من التحرك إلى output mode أثناء استخدامه كـ IRQ.
- `request_any_context_irq` يسمح للـ handler بالعمل في threaded context إذا لزم الأمر.
- يضبط `GPIOD_FLAG_EDGE_FALLING` و/أو `GPIOD_FLAG_EDGE_RISING` في `desc->flags`.

**Caller:** يُستدعى من `edge_store` ومن `gpio_sysfs_set_active_low`.

**Locking requirement:** المستدعي يجب أن يحجز `data->mutex`.

**Error path:**
```
err_unlock: gpiochip_unlock_as_irq → clear edge bits
err_clr_bits: clear edge bits
```

---

#### `gpio_sysfs_free_irq`

```c
static void gpio_sysfs_free_irq(struct gpiod_data *data)
```

**ما تعمله:** تحرر الـ IRQ المطلوب وتُلغي الـ irq lock على الـ GPIO. تُصفّر `irq_flags` وتمسح edge bits.

**Key details:**
- يستخدم `gpio_chip_guard` CLASS للحصول على `gc` pointer بشكل آمن.
- إذا لم يكن الـ `gc` موجوداً (chip detached)، ترجع بدون عمل شيء — هذا intentional لأن الـ chip يُنظّف بنفسه.
- `free_irq(data->irq, data)` — الـ `data` هو cookie الذي مُرِّر لـ `request_any_context_irq`.

**Locking:** المستدعي يجب أن يحجز `data->mutex` إلا في حالة post-deregistration.

---

#### `gpio_sysfs_set_active_low`

```c
static int gpio_sysfs_set_active_low(struct gpiod_data *data, int value)
```

**ما تعمله:** تضبط أو تمسح `GPIOD_FLAG_ACTIVE_LOW` في الـ descriptor. إذا كان الـ IRQ مضبوطاً على edge واحد فقط (falling أو rising)، تحرر الـ IRQ وتُعيد طلبه بـ trigger المعكوس ليتوافق مع الـ polarity الجديدة.

**Parameters:**
- `data` — `gpiod_data` (mutex محجوز من المستدعي).
- `value` — nonzero = active low.

**Return value:** `0` أو كود خطأ من `gpio_sysfs_request_irq`.

**Key details:**
- إذا كان `BOTH` edges مضبوطاً، لا حاجة لإعادة تهيئة الـ IRQ (كلا الـ edges مطلوب على أي حال).
- يُرسل `GPIO_V2_LINE_CHANGED_CONFIG` بعد التغيير لإبلاغ uAPI consumers.

---

### Group 5: الـ Visibility و Helper Functions

---

#### `gpio_is_visible`

```c
static umode_t gpio_is_visible(struct kobject *kobj, struct attribute *attr,
                               int n)
```

**ما تعمله:** callback في `attribute_group.is_visible`. تتحكم في أذونات ظهور الـ attributes ديناميكياً بناءً على قدرات الـ GPIO.

**Parameters:**
- `kobj` — الـ kobject المالك للـ attribute group.
- `attr` — الـ attribute المراد التحقق منه.
- `n` — index الـ attribute في المصفوفة.

**Return value:** `mode` المناسب للـ attribute، أو `0` لإخفائه كلياً.

**منطق الإخفاء:**
- `direction`: يُخفى إذا كان `direction_can_change == false`.
- `edge` (legacy): يُخفى إذا كان الـ GPIO لا يدعم IRQ (`gpiod_to_irq < 0`) أو كان output مع `direction_can_change == false`.

---

#### `gdev_get_data`

```c
static struct gpiodev_data *
gdev_get_data(struct gpio_device *gdev) __must_hold(&sysfs_lock)
```

**ما تعمله:** يبحث في الـ `gpio_class` عن الـ device المرتبط بالـ `gdev` المُعطى، ويُرجع الـ `gpiodev_data` المرتبط به.

**Key details:**
- مُعلَّمة بـ `__must_hold(&sysfs_lock)` — sparse annotation تُنبّه إذا استُدعيت بدون الـ lock.
- تستخدم `class_find_device` مع `match_gdev` callback.
- الـ `cdev` المُرجَّع يُحرر تلقائياً بـ `__free(put_device)` (cleanup.h macro).

---

#### `gpiod_attr_init`

```c
static void gpiod_attr_init(struct device_attribute *dev_attr, const char *name,
                            ssize_t (*show)(...), ssize_t (*store)(...))
```

**ما تعمله:** تُهيئ `device_attribute` ديناميكياً بـ `sysfs_attr_init` (لتجنب lockdep warnings)، اسم، mode `0644`، وcallbacks.

**Key details:** الـ `sysfs_attr_init` ضروري لأن هذه الـ attributes تُخصَّص ديناميكياً (`kzalloc`) وليست static — بدونه lockdep سيُبلّغ عن warning.

---

#### `match_gdev` / `match_export`

```c
static int match_gdev(struct device *dev, const void *desc)
static int match_export(struct device *dev, const void *desc)  // legacy
```

**ما تعمل:** callbacks لـ `class_find_device`. ترجع nonzero عند التطابق:
- `match_gdev`: تتحقق من `data->gdev == gdev`.
- `match_export`: تستخدم `gpiod_is_equal` للمقارنة بين الـ descriptors.

---

#### `gpiofind_sysfs_register`

```c
static int gpiofind_sysfs_register(struct gpio_chip *gc, const void *data)
```

**ما تعمله:** callback لـ `gpio_device_find` — تستدعي `gpiochip_sysfs_register` لكل chip موجود. ترجع `0` دائماً حتى لا تتوقف المشي على الـ chips.

---

### ملاحظات التصميم العامة

```
Architecture Overview:
══════════════════════════════════════════════════════════════
  userspace write to /sys/class/gpio/export
        │
        ▼
  export_store (legacy class attr)
  OR chip_export_store (chip device attr)
        │
        ▼
  do_chip_export_store → export_gpio_desc
        │                       │
        │               gpiod_request_user
        │               gpiod_set_transitory(false)
        │                       │
        ▼                       ▼
  gpiod_export(desc, true)
        │
        ├── [legacy] device_create → /sys/class/gpio/gpioN/
        │              attrs: direction, value, edge, active_low
        │
        └── sysfs_create_groups under /sys/class/gpio/chipX/gpio<offset>/
                       attrs: direction, value

  Edge detection flow (legacy):
  ════════════════════════════
  write "rising" to /sys/class/gpio/gpioN/edge
        │
  edge_store → gpio_sysfs_request_irq
        │       └── request_any_context_irq → gpio_sysfs_irq
        │
  poll(2) on /sys/class/gpio/gpioN/value blocks...
        │
  GPIO edge occurs → gpio_sysfs_irq fires
        └── sysfs_notify_dirent(value_kn) → poll(2) returns
```

**الـ Locking hierarchy:**
1. `sysfs_lock` mutex — يحمي export/unexport operations وقائمة الـ exported_lines.
2. `gdev->srcu` — يحمي `gdev->chip` pointer من الاختفاء.
3. `data->mutex` (per-line) — يحمي العمليات على line واحدة (direction, value, edge).

لا يجوز حجز `data->mutex` ثم `sysfs_lock` — الترتيب المعكوس يسبب deadlock.
## Phase 5: دليل الـ Debugging الشامل

الـ `gpiolib-sysfs.h` بيعرّف الـ interface الخاص بـ `CONFIG_GPIO_SYSFS` — اللي بيكشف الـ GPIO lines للـ userspace عبر `/sys/class/gpio`. الـ debugging هنا بيشمل مستويين: الـ sysfs interface نفسه، والـ gpiolib اللي تحته.

---

### Software Level

#### 1. debugfs Entries

**الـ main entry** للـ GPIO debugging:

```bash
# اقرأ state كل الـ GPIO chips المسجّلة
cat /sys/kernel/debug/gpio
```

**مثال على الـ output:**

```
gpiochip0: GPIOs 0-31, parent: platform/fe200000.gpio, pinctrl-bcm2835:
 gpio-2   (                    |sda1                ) in  hi
 gpio-3   (                    |scl1                ) in  hi
 gpio-17  (                    |led0                ) out lo
 gpio-27  (                    |button              ) in  lo    ACTIVE LOW
```

**تفسير الـ output:**
| الحقل | المعنى |
|---|---|
| `gpiochip0` | اسم الـ chip + range |
| `parent` | الـ platform device |
| `in/out` | الـ direction الحالي |
| `hi/lo` | القيمة الحالية |
| `ACTIVE LOW` | الـ polarity inversion مفعّل |
| الاسم بين `[]` | الـ consumer اللي request الـ line |

```bash
# شوف كل الـ chips المسجّلة
ls /sys/kernel/debug/gpio

# على بعض الـ kernels أحدث
cat /sys/kernel/debug/gpio | grep -A 50 "gpiochip0"
```

#### 2. sysfs Entries

**الـ entries الأساسية لـ GPIO sysfs (`CONFIG_GPIO_SYSFS`):**

```bash
# اتأكد إن الـ GPIO sysfs مفعّل
ls /sys/class/gpio/
# النتيجة: export  gpiochip0  gpiochip32  unexport

# شوف معلومات الـ chip
cat /sys/class/gpio/gpiochip0/base       # الـ base number
cat /sys/class/gpio/gpiochip0/ngpio      # عدد الـ lines
cat /sys/class/gpio/gpiochip0/label      # اسم الـ chip

# export GPIO للـ userspace
echo 17 > /sys/class/gpio/export

# بعد الـ export، هتلاقي directory جديد
ls /sys/class/gpio/gpio17/
# النتيجة: active_low  direction  edge  power  subsystem  uevent  value

# اقرأ الـ attributes
cat /sys/class/gpio/gpio17/direction     # in أو out
cat /sys/class/gpio/gpio17/value         # 0 أو 1
cat /sys/class/gpio/gpio17/active_low    # 0 أو 1
cat /sys/class/gpio/gpio17/edge          # none, rising, falling, both

# حرر الـ GPIO
echo 17 > /sys/class/gpio/unexport
```

**جدول الـ sysfs attributes:**
| الـ Attribute | الـ Permissions | الـ Values |
|---|---|---|
| `direction` | rw | `in`, `out`, `high`, `low` |
| `value` | rw | `0`, `1` |
| `active_low` | rw | `0`, `1` |
| `edge` | rw | `none`, `rising`, `falling`, `both` |

#### 3. ftrace — Tracepoints وـ Events

```bash
# شوف الـ GPIO events المتاحة
ls /sys/kernel/debug/tracing/events/gpio/

# الـ events الموجودة عادةً:
# gpio_value  gpio_direction

# فعّل كل الـ GPIO events
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو event واحد بس
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# افعل العملية اللي عايز تتبعها
echo 1 > /sys/class/gpio/gpio17/value

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace
```

**مثال على الـ trace output:**

```
           <...>-1234  [001] ....  1234.567890: gpio_value: 17 set 1
           <...>-1234  [001] ....  1234.567891: gpio_direction: 17 set output
```

**لـ trace الـ sysfs registration نفسها:**

```bash
# trace الـ function calls في الـ gpiolib
echo 'gpiochip_sysfs_register' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'gpiochip_sysfs_unregister' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

#### 4. printk وـ Dynamic Debug

```bash
# فعّل الـ dynamic debug لـ gpiolib-sysfs
echo 'file gpiolib-sysfs.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل الـ gpio subsystem
echo 'module gpio +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع الـ function name وـ line number
echo 'file gpiolib-sysfs.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ active debug points
cat /sys/kernel/debug/dynamic_debug/control | grep gpio
```

**الـ flags المتاحة في `+pflmt`:**
| الـ Flag | المعنى |
|---|---|
| `p` | فعّل الـ printk |
| `f` | اطبع اسم الـ function |
| `l` | اطبع رقم الـ line |
| `m` | اطبع اسم الـ module |
| `t` | اطبع الـ thread ID |

**على مستوى الـ boot، ممكن تضيف في `cmdline`:**

```bash
# في /boot/cmdline.txt أو GRUB_CMDLINE_LINUX
dyndbg="file gpiolib-sysfs.c +p"
```

#### 5. Kernel Config Options للـ Debugging

```kconfig
# أساسية لازم تكون مفعّلة
CONFIG_GPIO_SYSFS=y          # الـ sysfs interface نفسه
CONFIG_GPIOLIB=y             # الـ core library
CONFIG_DEBUG_GPIO=y          # extra validation وـ warnings

# لـ debugging عام
CONFIG_DEBUG_FS=y            # يفعّل /sys/kernel/debug
CONFIG_DYNAMIC_DEBUG=y       # يفعّل dynamic debug
CONFIG_TRACING=y             # يفعّل ftrace
CONFIG_GPIO_CDEV=y           # الـ character device interface (أحدث من sysfs)

# لـ pinctrl integration
CONFIG_PINCTRL=y
CONFIG_DEBUG_PINCTRL=y

# لو بتشتبه في race conditions
CONFIG_PROVE_LOCKING=y       # lockdep
CONFIG_DEBUG_SPINLOCK=y
CONFIG_DEBUG_MUTEXES=y

# لـ memory issues
CONFIG_KASAN=y               # kernel address sanitizer
CONFIG_DEBUG_OBJECTS=y       # object lifecycle tracking
```

#### 6. أدوات خاصة بالـ Subsystem

**`libgpiod` — الـ replacement الرسمي لـ sysfs GPIO:**

```bash
# تثبيت الأدوات
apt install gpiod   # Debian/Ubuntu
# أو
dnf install libgpiod-utils   # Fedora

# اعرض كل الـ chips
gpiodetect
# النتيجة:
# gpiochip0 [pinctrl-bcm2835] (54 lines)

# اعرض تفاصيل chip معينة
gpioinfo gpiochip0
# النتيجة:
#   line   0:      unnamed       unused   input  active-high
#   line   2:      "SDA1"        unused   input  active-high
#   line  17:      "LED0"    "led-gpio"  output  active-high [used]

# اقرأ قيمة GPIO
gpioget gpiochip0 17

# اكتب قيمة
gpioset gpiochip0 17=1

# راقب التغييرات (يستخدم character device، أفضل من sysfs poll)
gpiomon gpiochip0 17
```

**مقارنة بين الـ interfaces:**

```bash
# الطريقة القديمة (sysfs - deprecated)
echo 17 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio17/direction
echo 1 > /sys/class/gpio/gpio17/value

# الطريقة الجديدة (character device - مفضّلة)
gpioset gpiochip0 17=1
```

#### 7. جدول Error Messages الشائعة

| الـ Error Message في `dmesg` | المعنى | الحل |
|---|---|---|
| `GPIO_SYSFS is deprecated` | استخدام الـ sysfs interface | انتقل لـ `libgpiod` |
| `gpiochip_sysfs_register: sysfs registration failed` | فشل إنشاء الـ sysfs entries | تحقق من `CONFIG_GPIO_SYSFS=y` وـ dmesg للـ sysfs error |
| `gpio-NNN (label): is not an exported GPIO` | محاولة استخدام GPIO غير مـ export | `echo NNN > /sys/class/gpio/export` أول |
| `gpio-NNN: invalid GPIO` | الـ GPIO number خارج الـ range | تحقق من `ngpio` في `/sys/class/gpio/gpiochipX/ngpio` |
| `gpio-NNN: can't export to sysfs` | الـ GPIO مستخدم أو في conflict | تحقق من `cat /sys/kernel/debug/gpio` |
| `gpio-NNN (label): direction_store: invalid value` | قيمة غلط في `direction` | استخدم `in`، `out`، `high`، `low` بس |
| `gpio-NNN: gpiochip_lock_as_irq: tried to flag a GPIO that is not an input as IRQ` | محاولة IRQ على output GPIO | اعمل `echo in > direction` قبل استخدام `edge` |
| `sysfs: cannot create duplicate filename '/class/gpio/gpioNNN'` | double export | `echo NNN > /sys/class/gpio/unexport` أول |
| `gpio gpiochip0: (gpiolib) registering 54 GPIOs` | رسالة عادية عند الـ registration | مش error — يأكد إن الـ chip اتسجّل |

#### 8. استراتيجية `dump_stack()` و `WARN_ON()`

**النقاط الاستراتيجية في الـ gpiolib-sysfs:**

```c
/* في gpiochip_sysfs_register() — بعد فشل الـ device_create */
if (IS_ERR(dev)) {
    WARN_ON(1);  /* يطبع backtrace بدون kernel panic */
    dump_stack(); /* شوف مين استدعى الـ register */
    return PTR_ERR(dev);
}

/* في export_store() — لو الـ GPIO مش valid */
WARN_ON(!gpiochip_is_requested(chip, offset));

/* في direction_store() — تحقق من الـ state قبل التغيير */
WARN_ON(test_bit(FLAG_IS_OUT, &desc->flags) && !writable);
```

**في الـ code الجديد، استخدم:**

```c
/* بدل printk العادي */
dev_err(&gdev->dev, "sysfs registration failed: %d\n", ret);
dev_warn(&gdev->dev, "GPIO %d: unexpected state\n", offset);

/* لـ runtime checks */
if (WARN_ON_ONCE(gdev->ngpio == 0))
    return -EINVAL;
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

```bash
# اقرأ الـ current state من الـ kernel
cat /sys/kernel/debug/gpio

# قارن مع الـ hardware مباشرةً
# مثال لـ BCM2835 (Raspberry Pi) — الـ GPLEV0 register
devmem2 0x3F200034 w   # GPLEV0 — قرأية 32 GPIO
devmem2 0x3F200038 w   # GPLEV1 — الـ GPIOs 32-53

# اتأكد إن الـ bit المقابل للـ GPIO 17 = 1 لو الـ kernel قال "hi"
# GPIO 17 → bit 17 في GPLEV0
# قيمة متوقعة: 0x00020000
```

**جدول مقارنة الـ register vs kernel:**

```
Kernel:  gpio-17 (led0) out hi
Hardware: GPLEV0[17] = 1  ✓  متطابق

Kernel:  gpio-17 (led0) out lo
Hardware: GPLEV0[17] = 1  ✗  مشكلة hardware أو GPSET/GPCLR لم يُنفَّذ
```

#### 2. Register Dump Techniques

```bash
# devmem2 (لازم تثبّته)
apt install devmem2

# قراءة register واحد (BCM2835 GPIO base = 0x3F200000)
devmem2 0x3F200000 w   # GPFSEL0 — Function Select
devmem2 0x3F200034 w   # GPLEV0  — Level (Read)
devmem2 0x3F200004 w   # GPFSEL1

# مثال عملي — dump كامل لـ GPIO registers
for offset in 0x00 0x04 0x08 0x0C 0x10 0x14 0x1C 0x20 0x28 0x2C 0x34 0x38; do
    printf "Offset 0x%02x: " $offset
    devmem2 $((0x3F200000 + offset)) w 2>/dev/null | grep "Read at"
done
```

**باستخدام `/dev/mem` مع Python:**

```python
import mmap, struct

GPIO_BASE = 0x3F200000  # BCM2835
BLOCK_SIZE = 4096

with open('/dev/mem', 'rb') as f:
    mem = mmap.mmap(f.fileno(), BLOCK_SIZE,
                    offset=GPIO_BASE,
                    access=mmap.ACCESS_READ)
    # GPLEV0
    mem.seek(0x34)
    level = struct.unpack('<I', mem.read(4))[0]
    print(f"GPLEV0 = 0x{level:08X}")
    # GPIO 17 state
    print(f"GPIO17 = {(level >> 17) & 1}")
```

**لو `/dev/mem` متقفّل (`CONFIG_STRICT_DEVMEM=y`):**

```bash
# استخدم io tool من package i2c-tools
io -4 0x3F200034   # قراءة 32-bit
```

#### 3. Logic Analyzer وـ Oscilloscope Tips

**إعداد الـ Logic Analyzer:**

```
Channel 0 → GPIO pin المراد مراقبته
Channel 1 → Clock reference (لو موجود)
Channel 2 → Interrupt line (لو بتستخدم edge detection)

Sample Rate:  10x أكبر من أعلى frequency متوقعة
              مثال: 1MHz GPIO toggle → 10 MSPS على الأقل

Trigger:      Rising edge على الـ channel
Pre-trigger:  20% لشوف الـ state قبل الـ event
```

**سيناريوهات شائعة:**

```
المشكلة: GPIO مش بيتغير رغم إن الـ kernel بيكتب

الـ Check بالـ Scope:
1. قيس الـ voltage على الـ pin مباشرةً
2. لو الـ voltage 0V طول الوقت → pull-down قوي أو short
3. لو الـ voltage ثابت على VCC → pull-up قوي أو الـ driver output disabled

المشكلة: glitches على الـ line

الـ Check بالـ LA:
1. فعّل glitch filter (عادةً < 10ns بنشوزيها noise)
2. لو الـ glitch حقيقي → تحقق من الـ debounce في الـ DT
   debounce-interval = <10>; /* 10ms */
```

#### 4. Hardware Issues الشائعة وـ Kernel Log Patterns

| المشكلة | الـ dmesg Pattern | الـ Fix |
|---|---|---|
| الـ GPIO مش مُعرَّف في الـ DT | `gpio-controller: no GPIO chip registered` | أضف `gpio-controller;` في الـ DT node |
| الـ pin مستخدم من الـ pinctrl | `pin X is already requested` | تحقق من الـ pinmux conflicts |
| الـ voltage غلط (3.3V vs 1.8V) | لا يوجد error — hardware damage أو no signal | تحقق من الـ IO voltage banks |
| الـ interrupt مش بيشتغل | `irq: type mismatch, failed to map hwirq` | تأكد من `interrupts` property في الـ DT |
| الـ GPIO مش responsive بعد suspend | `gpio resume failed` | تحقق من الـ driver's `suspend/resume` hooks |
| overcurrent | kernel panic أو GPIO stuck | قيس الـ current على الـ pin (max عادةً 4-16mA) |

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT الـ compiled (الـ live DT)
ls /sys/firmware/devicetree/base/

# شوف كل الـ gpio controllers
find /sys/firmware/devicetree/base -name "gpio-controller" 2>/dev/null

# اقرأ properties الـ GPIO node
cat /sys/firmware/devicetree/base/soc/gpio@7e200000/compatible
cat /sys/firmware/devicetree/base/soc/gpio@7e200000/\#gpio-cells

# باستخدام fdtget (من device-tree-compiler)
apt install device-tree-compiler

# dump الـ full DT
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 10 "gpio@"

# تحقق من الـ GPIO consumer properties
fdtget /boot/dtbs/$(uname -r)/bcm2837-rpi-3-b.dtb /leds/led0 gpios
# النتيجة: <0x07 0x11 0x00>  → phandle=7, pin=17, flags=0 (ACTIVE_HIGH)
```

**مثال على DT node صح:**

```dts
/* في الـ .dts file */
gpio_leds {
    compatible = "gpio-leds";

    led0 {
        label = "led0";
        gpios = <&gpio 17 GPIO_ACTIVE_HIGH>;
        /* GPIO_ACTIVE_HIGH = 0, GPIO_ACTIVE_LOW = 1 */
    };
};

&gpio {
    gpio-controller;        /* لازم يكون موجود */
    #gpio-cells = <2>;      /* pin number + flags */
    ngpios = <54>;          /* عدد الـ lines */
};
```

**التحقق من الـ DT vs الـ actual hardware:**

```bash
# شوف الـ pinmux للـ GPIO 17
cat /sys/kernel/debug/pinctrl/pinctrl-bcm2835/pinmux-pins | grep "pin 17"
# النتيجة: pin 17 (GPIO17): gpio_generic device gpio-leds.0

# لو الـ pin مش في الـ list → مش مُعرَّف في الـ DT أو مش مـ register
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**1. تشخيص سريع للـ GPIO subsystem:**

```bash
#!/bin/bash
echo "=== GPIO Chips ==="
gpiodetect 2>/dev/null || ls /sys/class/gpio/

echo "=== GPIO State (debugfs) ==="
cat /sys/kernel/debug/gpio

echo "=== Exported GPIOs ==="
ls /sys/class/gpio/ | grep "^gpio[0-9]"

echo "=== Kernel Config ==="
zcat /proc/config.gz 2>/dev/null | grep -E "CONFIG_GPIO|CONFIG_DEBUG_GPIO"
```

**2. Export واختبار GPIO:**

```bash
GPIO=17
# Export
echo $GPIO > /sys/class/gpio/export

# اقرأ الـ state
echo "Direction: $(cat /sys/class/gpio/gpio${GPIO}/direction)"
echo "Value:     $(cat /sys/class/gpio/gpio${GPIO}/value)"
echo "Active Low:$(cat /sys/class/gpio/gpio${GPIO}/active_low)"

# اعمل output وـ toggle
echo out > /sys/class/gpio/gpio${GPIO}/direction
echo 1   > /sys/class/gpio/gpio${GPIO}/value
sleep 1
echo 0   > /sys/class/gpio/gpio${GPIO}/value

# Unexport
echo $GPIO > /sys/class/gpio/unexport
```

**3. مراقبة الـ GPIO sysfs registration في الـ dmesg:**

```bash
# فعّل dynamic debug قبل الـ driver load
echo 'file gpiolib-sysfs.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file gpiolib.c +p'       >> /sys/kernel/debug/dynamic_debug/control

# راقب الـ messages في real-time
dmesg -w | grep -i gpio &

# حمّل الـ driver
modprobe gpio-mockup gpio_mockup_ranges="-1,8"

# أو trigger registration بـ device unbind/bind
echo "fe200000.gpio" > /sys/bus/platform/drivers/pinctrl-bcm2835/unbind
echo "fe200000.gpio" > /sys/bus/platform/drivers/pinctrl-bcm2835/bind
```

**4. ftrace كامل لـ GPIO operations:**

```bash
#!/bin/bash
TRACE=/sys/kernel/debug/tracing

# صفّر الـ trace
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace

# فعّل GPIO events
echo 1 > $TRACE/events/gpio/enable 2>/dev/null

# فعّل function tracing لـ gpiolib
echo 'gpiochip_sysfs_*' > $TRACE/set_ftrace_filter
echo 'gpiod_*'          >> $TRACE/set_ftrace_filter
echo function_graph     > $TRACE/current_tracer

# ابدأ
echo 1 > $TRACE/tracing_on

# افعل الـ operation
echo 17 > /sys/class/gpio/export

# وقّف واقرأ
echo 0 > $TRACE/tracing_on
cat $TRACE/trace | head -50
```

**مثال على الـ output:**

```
 0)               |  gpiochip_sysfs_register() {
 0)   0.543 us    |    device_create_with_groups();
 0)   2.103 us    |    sysfs_create_group();
 0) + 15.234 us   |  }
```

**5. فحص الـ interrupt على GPIO:**

```bash
GPIO=17

# فعّل edge detection
echo $GPIO > /sys/class/gpio/export
echo in    > /sys/class/gpio/gpio${GPIO}/direction
echo both  > /sys/class/gpio/gpio${GPIO}/edge

# انتظر interrupt باستخدام select (Python)
python3 - <<'EOF'
import select, time

f = open('/sys/class/gpio/gpio17/value', 'r')
# initial read
f.read()

print("Waiting for GPIO edge...")
while True:
    r, w, e = select.select([], [], [f], 5.0)
    if e:
        f.seek(0)
        val = f.read().strip()
        print(f"Edge detected! value={val} at {time.time():.6f}")
    else:
        print("Timeout — no edge in 5 seconds")
        break
EOF
```

**6. تحقق من الـ sysfs registration errors:**

```bash
# شوف errors في الـ kernel log المتعلقة بالـ sysfs registration
dmesg | grep -E "gpio|sysfs" | grep -i "err\|fail\|warn"

# شوف الـ uevent للـ chip
udevadm info --query=all --path=/sys/class/gpio/gpiochip0

# مثال على الـ output الصح:
# P: /devices/platform/soc/3f200000.gpio/gpio/gpiochip0
# N: gpiochip0
# E: DEVNAME=gpiochip0
# E: SUBSYSTEM=gpio
# E: DEVTYPE=chip

# تحقق من الـ udev rules
udevadm test /sys/class/gpio/gpiochip0 2>&1 | grep -i gpio
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: GPIO sysfs مش شغال على بورد STM32MP1 — Industrial Gateway

#### العنوان
**`gpiochip_sysfs_register` مش بيتاخد call** — الـ sysfs entries مش بتظهر خالص

#### السياق
شركة بتعمل **industrial gateway** على بورد custom مبنية على **STM32MP1**. الـ production team محتاجة تتحكم في GPIO pins من userspace عشان تشغّل relays وتقرأ digital inputs. الـ engineer بيتوقع يلاقي `/sys/class/gpio/gpiochipX` بس مش بيلاقيه.

#### المشكلة
الـ kernel config على السيستم:
```bash
$ grep GPIO_SYSFS /boot/config-$(uname -r)
# CONFIG_GPIO_SYSFS is not set
```
الـ `/sys/class/gpio/` موجود بس فاضي تماماً. مفيش `export`، مفيش `gpiochipX`.

#### التحليل
في `gpiolib-sysfs.h` الكود صريح:

```c
#ifdef CONFIG_GPIO_SYSFS

int gpiochip_sysfs_register(struct gpio_device *gdev);
void gpiochip_sysfs_unregister(struct gpio_device *gdev);

#else

/* stub — بترجع 0 وتعمل إيه يعني */
static inline int gpiochip_sysfs_register(struct gpio_device *gdev)
{
    return 0;
}

static inline void gpiochip_sysfs_unregister(struct gpio_device *gdev)
{
}

#endif /* CONFIG_GPIO_SYSFS */
```

لما `CONFIG_GPIO_SYSFS` مش موجود، الـ `gpiochip_sysfs_register()` بتبقى **stub function** بترجع 0 من غير ما تعمل أي حاجة. الـ chip بيتسجل في gpiolib بشكل طبيعي، بس مفيش sysfs nodes بتتعمل خالص.

#### الحل
في `menuconfig`:
```bash
Device Drivers →
  GPIO support →
    [*] /sys/class/gpio/... (sysfs interface)
```

أو في `.config` أو `defconfig`:
```bash
CONFIG_GPIO_SYSFS=y
```

ثم rebuild الـ kernel وبعدين:
```bash
$ ls /sys/class/gpio/
export  gpiochip0  gpiochip48  unexport

$ echo 48 > /sys/class/gpio/export
$ echo out > /sys/class/gpio/gpio48/direction
$ echo 1 > /sys/class/gpio/gpio48/value
```

#### الدرس المستفاد
الـ `gpiolib-sysfs.h` بيستخدم **compile-time guarding** بـ `#ifdef CONFIG_GPIO_SYSFS`. الـ stub functions بتخلي الـ code يـ compile من غير errors حتى لو الـ feature مش enabled، بس وظيفتها **صفر**. لازم تفرق بين "الـ driver شغال" و"الـ sysfs interface enabled" — هما حاجتين منفصلتين تماماً.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — `unexport` بيعمل kernel panic

#### العنوان
**`gpiochip_sysfs_unregister` أثناء hot-unplug** يسبب use-after-free

#### السياق
منتج **Android TV Box** على **Allwinner H616**. الـ team بتعمل OTA kernel update وبتـ test hot-unplug لـ USB GPIO expander (MCP23017 على I2C). لما بيشيلوا الـ expander أثناء التشغيل، الـ system بيـ crash.

#### المشكلة
الـ crash trace:
```
BUG: KASAN: use-after-free in sysfs_remove_group+0x...
Call Trace:
  gpiochip_sysfs_unregister
  gpiochip_remove
  mcp23017_remove
  i2c_device_remove
```

الـ userspace process لسه فاتح `fd` على `/sys/class/gpio/gpio200/value` وبيعمل `poll()` عليه. في نفس الوقت الـ `gpiochip_sysfs_unregister` بيشيل الـ sysfs nodes.

#### التحليل
الـ `gpiochip_sysfs_unregister` signature في `gpiolib-sysfs.h`:

```c
void gpiochip_sysfs_unregister(struct gpio_device *gdev);
```

الـ `gdev` هو `struct gpio_device` اللي بيحتوي على:
- `struct device dev` — الـ kobject اللي عليه الـ sysfs tree
- `struct srcu_struct srcu` — عشان الـ RCU protection
- `struct blocking_notifier_head device_notifier` — عشان إعلام الـ waiters

الـ implementation في `gpiolib-sysfs.c` بتشيل الـ sysfs nodes بس مش بتنتظر الـ existing file descriptors تتأثر. الـ `gpio_device` ممكن يتعمله free قبل ما الـ userspace يخلص.

#### الحل
الـ fix الصح: الـ userspace لازم يـ handle `ENODEV`:
```c
/* في userspace application */
ret = write(fd, "1", 1);
if (ret < 0 && errno == ENODEV) {
    /* GPIO chip اتشال، اعمل cleanup */
    close(fd);
    reopen_gpio_if_needed();
}
```

وفي الـ kernel driver لازم تضمن:
```c
/* في mcp23017_remove */
gpiochip_remove(&mcp->chip); /* ده بيـ call gpiochip_sysfs_unregister */
/* انتظر SRCU grace period قبل ما تـ free الـ memory */
synchronize_srcu(&gdev->srcu);
```

#### الدرس المستفاد
الـ `gpiochip_sysfs_unregister` بيشيل الـ interface بس مش بيـ guarantee إن مفيش references عليه. الـ `struct gpio_device` بيعيش أطول من الـ chip بسبب الـ SRCU mechanism — الـ header بيكشف إن الـ `gdev` هو الـ unit اللي بيتم الـ unregister عليه، مش الـ `gpio_chip` مباشرة. ده مهم جداً في سيناريوهات الـ hot-unplug.

---

### السيناريو الثالث: IoT Sensor على AM62x — الـ sysfs GPIO مش بيظهر بعد `devm_`

#### العنوان
**الـ `gpiochip_sysfs_register` بيـ return error** مش بيتـ handle صح

#### السياق
**IoT sensor node** على **TI AM62x** بيقيس temperature وpressure. الـ custom kernel module بيعمل virtual GPIO chip (لـ software-controlled LEDs). الـ module بيـ load بس الـ sysfs entry مش بتظهر، ومفيش أي error في dmesg.

#### المشكلة
```bash
$ dmesg | grep gpio
[    2.345] gpio gpiochip4: registered GPIOs 128 to 159
# مفيش error، بس:
$ ls /sys/class/gpio/gpiochip128
ls: cannot access '/sys/class/gpio/gpiochip128': No such file or directory
```

#### التحليل
الـ function signature من `gpiolib-sysfs.h`:

```c
int gpiochip_sysfs_register(struct gpio_device *gdev);
```

الـ function بتـ return `int` — ممكن تفشل! الـ gpiolib داخلياً في `gpiochip_add_data_with_key()` بتعمل:

```c
/* من gpiolib.c */
status = gpiochip_sysfs_register(gdev);
if (status)
    goto err_remove_from_list; /* بس الـ error ممكن يتـ swallow */
```

في بعض الإصدارات القديمة أو الـ custom kernels، الـ error من `gpiochip_sysfs_register` ممكن يتعامل معاه بـ `dev_warn` بس مش بـ fail الـ registration كلها. النتيجة: الـ chip متسجل في gpiolib، بيشتغل من character device `/dev/gpiochipX`، بس مفيش sysfs nodes.

السبب الفعلي في الـ case ده: **sysfs** بيرفض create node بسبب اسم متكرر (stale entry من crash سابق):
```bash
$ ls /sys/class/gpio/
gpiochip128  # ده stale entry من kernel panic قبل كده!
```

#### الحل
```bash
# Clean up stale sysfs entries بعد crash
$ rmmod your_gpio_module
$ modprobe your_gpio_module
```

وفي الـ production code لازم تـ check الـ return value:
```c
ret = gpiochip_add_data(&chip->gc, chip);
if (ret) {
    dev_err(dev, "Failed to add GPIO chip: %d\n", ret);
    return ret;
}
/* لو gpiochip_sysfs_register فشلت، راجع dmesg بتدقيق */
```

#### الدرس المستفاد
الـ `gpiochip_sysfs_register` بتـ return `int` مش `void` — ده مش زينة. لازم دايماً تـ check الـ return value وتفهم إن فشل الـ sysfs registration ممكن ميـ fail الـ chip registration كلها، وده بيخلي الـ chip يشتغل بس من غير sysfs interface.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — `CONFIG_GPIO_SYSFS` ممنوع في production

#### العنوان
**تعطيل `CONFIG_GPIO_SYSFS` عن قصد** في الـ production image للـ security

#### السياق
شركة automotive بتبني **ECU (Electronic Control Unit)** على **NXP i.MX8QM** لسيارة كهربائية. الـ security team بتشترط إن الـ userspace ميقدرش يتحكم في GPIO pins مباشرة من الـ sysfs عشان يمنع الـ privilege escalation وأي attack surface زيادة.

#### المشكلة
الـ security audit لاقت إن:
```bash
$ ls /sys/class/gpio/
export  gpiochip0  gpiochip32  unexport

$ echo 32 > /sys/class/gpio/export  # أي process بيقدر يعمل ده!
$ echo out > /sys/class/gpio/gpio32/direction
$ echo 1 > /sys/class/gpio/gpio32/value  # يتحكم في brake relay مثلاً!
```

#### التحليل
الـ `gpiolib-sysfs.h` بيكشف الـ design بوضوح:

```c
#ifdef CONFIG_GPIO_SYSFS
/* الـ real functions — بتعمل sysfs nodes */
int gpiochip_sysfs_register(struct gpio_device *gdev);
void gpiochip_sysfs_unregister(struct gpio_device *gdev);

#else
/* الـ stubs — بتعمل إيه؟ لا إيه. صفر. */
static inline int gpiochip_sysfs_register(struct gpio_device *gdev)
{
    return 0;  /* success بدون أي عمل */
}
static inline void gpiochip_sysfs_unregister(struct gpio_device *gdev)
{
    /* empty — no sysfs, no problem */
}
#endif
```

الـ design مقصود: لو `CONFIG_GPIO_SYSFS=n`، الـ functions بتبقى no-ops، والـ GPIO subsystem شغال normal بس بدون أي sysfs exposure. الـ GPIO character device `/dev/gpiochipX` لسه موجود وبيقدر يتحكم فيه بـ proper permissions.

#### الحل
في الـ secure automotive `defconfig`:
```bash
# GPIO sysfs interface — disabled for security
# CONFIG_GPIO_SYSFS is not set

# بدلها نستخدم character device مع proper permissions
CONFIG_GPIO_CDEV=y
CONFIG_GPIO_CDEV_V1=n  # disable legacy API
```

وفي الـ `udev` rules:
```bash
# /etc/udev/rules.d/99-gpio-secure.rules
SUBSYSTEM=="gpio", KERNEL=="gpiochip*", GROUP="gpio-users", MODE="0660"
```

فقط الـ processes اللي في group `gpio-users` بتقدر تـ access الـ GPIO.

#### الدرس المستفاد
الـ `#ifdef CONFIG_GPIO_SYSFS` في `gpiolib-sysfs.h` هو **security boundary** مش بس feature toggle. في الـ embedded وautomotive systems، تعطيل الـ sysfs GPIO interface بـ compile-time option هو أقوى بكتير من أي runtime access control لأنه بيمنع الـ attack surface من الأساس. الـ character device API بيدي control أدق على الـ permissions.

---

### السيناريو الخامس: Custom Board Bring-up على RK3562 — الـ gpiochip بيتسجل بس الـ sysfs ordering غلط

#### العنوان
**الـ `gpiochip_sysfs_register` بيحصل قبل اكتمال الـ sysfs class** — race condition أثناء boot

#### السياق
مهندس بيعمل **board bring-up** لبورد custom على **Rockchip RK3562** فيها 5 GPIO banks. أثناء الـ boot الأول لاحظ إن بعض الـ gpiochip entries بتظهر في sysfs وبعضها لأ — والنتيجة بتتغير من boot للتاني.

#### المشكلة
```bash
# Boot 1:
$ ls /sys/class/gpio/
gpiochip0  gpiochip32  gpiochip64  export  unexport
# gpiochip96 و gpiochip128 مش موجودين!

# Boot 2:
$ ls /sys/class/gpio/
gpiochip0  gpiochip32  gpiochip64  gpiochip96  gpiochip128  export  unexport
# كلهم موجودين!
```

#### التحليل
الـ `gpiochip_sysfs_register(struct gpio_device *gdev)` بيتاخد call من `gpiochip_add_data_with_key()` في gpiolib. لو الـ GPIO driver اتـ probe قبل ما الـ `/sys/class/gpio` class اتـ initialize كامل، الـ `device_create()` داخل `gpiochip_sysfs_register` بترجع error.

الـ sequence بالمشكلة:
```
[0.123] gpio-class: /sys/class/gpio registered
[0.145] rk3562-gpio gpio0: gpiochip_sysfs_register → OK (gpiochip0)
[0.146] rk3562-gpio gpio1: gpiochip_sysfs_register → OK (gpiochip32)
[0.147] rk3562-gpio gpio2: gpiochip_sysfs_register → OK (gpiochip64)
[0.148] ... kernel busy with other init ...
[0.201] rk3562-gpio gpio3: gpiochip_sysfs_register → FAIL (-ENODEV)
         # الـ class لسه بتـ setup في context تاني
[0.202] rk3562-gpio gpio4: gpiochip_sysfs_register → OK (gpiochip128)
```

الـ `gpio3` اتسجل في gpiolib بنجاح بس الـ sysfs registration فشلت بـ `-ENODEV` أو `-EEXIST`.

الـ `gpiochip_sysfs_register` signature في الـ header:
```c
int gpiochip_sysfs_register(struct gpio_device *gdev);
```

بترجع `int` — الـ error ممكن يتـ log بـ `dev_warn` بس ما بيـ fail الـ chip registration كلها. النتيجة: الـ chip شغال عبر character device بس غايب من sysfs.

#### الحل
الـ debug الأول:
```bash
$ dmesg | grep -i "gpio.*sysfs\|gpiochip.*err"
[0.201] gpio gpiochip3: Failed to register gpio sysfs: -19
# -19 = ENODEV
```

الـ fix في ترتيب الـ `initcall` — نضمن إن الـ GPIO subsystem جاهز قبل الـ drivers:
```c
/* في gpiolib-sysfs.c */
postcore_initcall(gpiolib_sysfs_init);
/* الـ GPIO drivers بيتـ probe بعد كده في device_initcall */
```

وكـ workaround مؤقت أثناء الـ bring-up:
```bash
# re-trigger udev للـ chips اللي miss الـ sysfs registration
$ udevadm trigger --subsystem-match=gpio
$ ls /sys/class/gpio/  # دلوقتي المفروض يظهروا كلهم
```

وللـ production، تأكد إن الـ GPIO banks الـ 5 بيتـ probe في order صح عبر DT:
```dts
/* في rk3562.dtsi — ترتيب التعريف بيأثر على ترتيب probe */
gpio0: gpio@fdd60000 { status = "okay"; };
gpio1: gpio@fe740000 { status = "okay"; };
gpio2: gpio@fe750000 { status = "okay"; };
gpio3: gpio@fe760000 { status = "okay"; };
gpio4: gpio@fe770000 { status = "okay"; };
```

#### الدرس المستفاد
الـ `gpiochip_sysfs_register` و`gpiochip_sysfs_unregister` في `gpiolib-sysfs.h` بيعملوا guarding على مستوى الـ compile-time بـ `#ifdef`، بس الـ runtime failures بتاعتهم مش دايماً propagated بشكل ظاهر. أثناء الـ board bring-up على SoCs زي RK3562 اللي فيها GPIO banks كتير، لازم تـ trace كل chip على حدة وتـ verify إن الـ sysfs registration نجحت فعلاً مش بس إن الـ chip اتسجل في gpiolib.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي اتنشرت على LWN.net وبتغطي الـ GPIO sysfs interface بالتفصيل:

| المقالة | الوصف |
|---------|-------|
| [gpio: sysfs interface (updated)](https://lwn.net/Articles/286435/) | النسخة المحدّثة من تصميم الـ sysfs interface للـ GPIO — بيشرح الـ `gpio_export()` والـ `gpio_unexport()` |
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة للـ GPIO API الداخلية في الكيرنل، بتشمل الـ `gpio_request()` وعلاقته بالـ sysfs |
| [gpio: sysfs interface (original proposal)](https://lwn.net/Articles/280187/) | البروبوزال الأصلي لإضافة الـ sysfs interface للـ GPIO |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | بيناقش التحولات نحو الـ chardev API وإيه اللي كان غلط في الـ sysfs |
| [Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/) | توثيق الـ gpiod descriptor-based API الجديدة |
| [RFC: add a gpio-sysfs interface](https://lwn.net/Articles/194634/) | النقاش الأولاني على الميلينج ليست لإضافة الـ GPIO sysfs interface |

---

### توثيق الكيرنل الرسمي

الـ `Documentation/` paths الأساسية:

```
Documentation/admin-guide/gpio/         ← واجهات الـ userspace
Documentation/driver-api/gpio/          ← API الداخلية للـ drivers
Documentation/userspace-api/gpio/       ← chardev v1 و v2 ABI
```

- **[GPIO Sysfs Interface for Userspace](https://www.kernel.org/doc/Documentation/gpio/sysfs.txt)** — الملف الرسمي اللي بيوضح الـ `/sys/class/gpio/` interface وبيقول صراحة إنه deprecated

- **[Legacy GPIO Interfaces](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html)** — شرح الـ integer-based GPIO API القديمة اللي بيعتمد عليها الـ sysfs

- **[General Purpose Input/Output (GPIO)](https://static.lwn.net/kerneldoc/driver-api/gpio/index.html)** — الصفحة الرئيسية للـ GPIO documentation في الكيرنل

- **[GPIO Character Device Userspace API (v1)](https://www.kernel.org/doc/html/latest//userspace-api/gpio/chardev_v1.html)** — توثيق الـ chardev ABI اللي جه بديل الـ sysfs

- **[GPIO Character Device Userspace API (v2)](https://docs.kernel.org/userspace-api/gpio/chardev.html)** — الـ API الأحدث (Linux 5.10+)

---

### Commits مهمة في الكيرنل

| الـ Commit | الوصف |
|-----------|-------|
| [3c702e9](https://github.com/torvalds/linux/commit/3c702e9987e261042a07e43460a8148be254412e) | `gpio: add a userspace chardev ABI for GPIOs` — أول commit لـ Linus Walleij بيضيف الـ chardev كبديل للـ sysfs |
| [`drivers/gpio/gpiolib.c`](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpiolib.c) | الكود الحالي للـ gpiolib في الكيرنل |

لو عايز تتابع تاريخ ملف `gpiolib-sysfs.c` كامل:

```bash
git log --follow -p drivers/gpio/gpiolib-sysfs.c
git log --oneline drivers/gpio/gpiolib-sysfs.h
```

---

### نقاشات الميلينج ليست

- **[GPIO character device skeleton (patch series)](https://lore.kernel.org/all/1445502750-22672-1-git-send-email-linus.walleij@linaro.org/T/)** — الـ patch series الأصلية من Linus Walleij بتقدّم الـ chardev ABI وبتشرح ليه الـ sysfs محتاج يتستبدل

- **[Suggestion on GPIO sysfs interface (gpio_export)](https://lists.gt.net/linux/kernel/1064345)** — نقاش قديم على LKML حول الـ `gpio_export()` function

- **[gpio: add a userspace chardev ABI — Patchwork](https://patchwork.ozlabs.org/patch/580307/)** — الـ patch الرسمي مع التعليقات والـ review

---

### كتب مقترحة

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 14**: The Linux Device Model — بيشرح الـ `kobject` والـ `sysfs` اللي هما الأساس اللي بنت عليه الـ GPIO sysfs interface
- **الفصل 1**: An Introduction to Device Drivers — overview للـ driver types
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل 17**: Devices and Modules — بيتكلم عن الـ device model وتكامله مع الـ sysfs
- **الفصل 19**: Portability — بيذكر الـ GPIO في سياق الـ embedded systems

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 16**: Kernel Debugging Techniques — بيشمل استخدام الـ sysfs في debugging
- بيغطي الـ GPIO sysfs interface كأداة أساسية في الـ embedded development

---

### مصادر elinux.org

- **[EBC Flashing an LED](https://elinux.org/EBC_Flashing_an_LED)** — مثال عملي لاستخدام الـ `/sys/class/gpio/` لتشغيل LED على BeagleBone

- **[CI20 GPIO LED Blink Tutorial](https://elinux.org/CI20_GPIO_LED_Blink_Tutorial)** — tutorial كامل للـ GPIO sysfs على hardware حقيقي

- **[EBC GPIO via mmap](https://elinux.org/EBC_GPIO_via_mmap)** — مقارنة أداء الـ sysfs مع الـ mmap-based access

---

### مصادر kernelnewbies.org

- **[Linux_2_6_21](https://kernelnewbies.org/Linux_2_6_21)** — الكيرنل اللي بدأت فيه أولى خطوات الـ GPIO infrastructure

- **[Linux_2_6_25](https://kernelnewbies.org/Linux_2_6_25)** — يتضمن تغييرات على الـ gpiolib

- **[Linux_6.2](https://kernelnewbies.org/Linux_6.2)** — آخر التحديثات على الـ GPIO subsystem

- **[Using GPIO from device tree on platform devices](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html)** — نقاش عملي على ميلينج ليست kernelnewbies

- **[devm_gpiod_get usage](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html)** — سؤال عملي عن استخدام الـ descriptor-based API

---

### مصادر إضافية مهمة

- **[Stop using /sys/class/gpio — it's deprecated](https://www.thegoodpenguin.co.uk/blog/stop-using-sys-class-gpio-its-deprecated/)** — مقالة عملية بتشرح الهجرة من الـ sysfs للـ chardev

- **[New Linux kernel GPIO user space interface](https://sergioprado.blog/new-linux-kernel-gpio-user-space-interface/)** — شرح تفصيلي للـ chardev API الجديدة مقارنةً بالـ sysfs

- **[An Introduction to chardev GPIO and Libgpiod](https://www.beyondlogic.org/an-introduction-to-chardev-gpio-and-libgpiod-on-the-raspberry-pi/)** — بيشرح الـ `libgpiod` library اللي بتتعامل مع الـ chardev

- **[GPIO Programming: Using the sysfs Interface](https://www.ics.com/blog/gpio-programming-using-sysfs-interface)** — مقالة ICS بتشرح الـ sysfs interface بالكود

---

### Search Terms للبحث عن معلومات أكتر

```
# بحث عن الكود في الكيرنل
gpiolib-sysfs.c kernel source
gpiochip_sysfs_register implementation
CONFIG_GPIO_SYSFS kernel config

# بحث عن التاريخ والنقاشات
"gpio sysfs deprecated" lkml
"gpio chardev" "sysfs replacement" linux kernel
linus.walleij gpio sysfs

# بحث عن الاستخدام العملي
/sys/class/gpio export userspace
libgpiod vs sysfs gpio
gpio-cdev linux userspace

# بحث عن الكتب والتوثيق
"Linux Device Drivers" sysfs kobject
"embedded linux" gpio sysfs interface
```
## Phase 8: Writing simple module

### الفكرة

**`gpiochip_sysfs_register`** هي الدالة اللي بتربط الـ GPIO chip بالـ sysfs — بتتسجل عبر `EXPORT_SYMBOL_GPL` وبتعمل entries في `/sys/class/gpio/`. هنعمل **kprobe** عليها عشان نرصد كل مرة chip بتتسجل، ونطبع اسمها وعدد الـ lines بتاعتها.

الـ kprobe هو الأنسب هنا لأن الدالة مش tracepoint وملهاش notifier chain خاصة بيها — الـ kprobe بيخلينا نحط hook على أي رمز exported في الـ kernel من غير ما نعدل الـ source.

---

### الـ Module Code

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * gpio_sysfs_probe.c
 *
 * Hooks gpiochip_sysfs_register() via kprobe to log
 * each GPIO chip being registered into sysfs.
 */

#include <linux/module.h>      /* MODULE_* macros, module_init/exit      */
#include <linux/kernel.h>      /* pr_info, pr_err                         */
#include <linux/kprobes.h>     /* kprobe struct, register_kprobe, etc.    */
#include <linux/gpio/driver.h> /* struct gpio_device, gpio_chip           */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("GPIO Sysfs Watcher");
MODULE_DESCRIPTION("kprobe على gpiochip_sysfs_register لرصد تسجيل GPIO chips في sysfs");

/* -----------------------------------------------------------------------
 * Pre-handler: يُستدعى قبل تنفيذ gpiochip_sysfs_register
 *
 * الـ regs بتحتوي على registers الـ CPU لحظة الـ hook —
 * على x86-64 الـ argument الأول (gdev) موجود في rdi.
 * ----------------------------------------------------------------------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64: first argument is in rdi.
     * On ARM64:  first argument is in x0.
     * kernel's regs_get_kernel_argument() abstracts this.
     */
    struct gpio_device *gdev =
        (struct gpio_device *)regs_get_kernel_argument(regs, 0);

    if (!gdev)
        return 0; /* guard against NULL — shouldn't happen in normal flow */

    /*
     * gpio_device لديها حقلي label و ngpio.
     * label: اسم الـ chip (مثلاً "gpio-mockup-A" أو "amba-pl061").
     * ngpio: عدد الـ GPIO lines في الـ chip دي.
     */
    pr_info("[gpio_sysfs_probe] gpiochip_sysfs_register called: "
            "label=\"%s\" ngpio=%u id=%d\n",
            gdev->label  ? gdev->label : "(null)",
            gdev->ngpio,
            gdev->id);

    return 0; /* return 0 = continue normal execution */
}

/* -----------------------------------------------------------------------
 * Post-handler: يُستدعى بعد تنفيذ gpiochip_sysfs_register
 *
 * بنستخدمه عشان نعرف هل التسجيل نجح أم لا — لو الـ return value
 * غير صفر معناه في مشكلة في الـ sysfs registration.
 * ----------------------------------------------------------------------- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * ax/x0 contains the return value after the function executes.
     * regs_return_value() is arch-independent helper for this.
     */
    long ret = regs_return_value(regs);

    if (ret < 0)
        pr_warn("[gpio_sysfs_probe] gpiochip_sysfs_register returned error: %ld\n",
                ret);
    else
        pr_info("[gpio_sysfs_probe] gpiochip_sysfs_register succeeded (ret=%ld)\n",
                ret);
}

/* -----------------------------------------------------------------------
 * تعريف الـ kprobe نفسه
 *
 * symbol_name: اسم الدالة اللي هنحط عليها الـ probe — الـ kernel
 * بيحوّله لعنوان تلقائياً باستخدام kallsyms.
 * ----------------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "gpiochip_sysfs_register",
    .pre_handler = handler_pre,
    .post_handler = handler_post,
};

/* -----------------------------------------------------------------------
 * module_init: بيسجل الـ kprobe عند تحميل الـ module
 *
 * لو التسجيل فشل (مثلاً الـ symbol مش موجود أو CONFIG_KPROBES=n)
 * بنرجع error عشان الـ modprobe يعرف.
 * ----------------------------------------------------------------------- */
static int __init gpio_sysfs_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[gpio_sysfs_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[gpio_sysfs_probe] kprobe planted on gpiochip_sysfs_register @ %p\n",
            kp.addr);
    return 0;
}

/* -----------------------------------------------------------------------
 * module_exit: بيلغي تسجيل الـ kprobe عند إزالة الـ module
 *
 * الـ unregister ضروري عشان لو الـ module اتنزل والـ probe لسه
 * شغال، أي call للدالة هيـcrash الـ kernel لأن الـ handler
 * هيتنفذ في memory اتحررت.
 * ----------------------------------------------------------------------- */
static void __exit gpio_sysfs_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[gpio_sysfs_probe] kprobe removed from gpiochip_sysfs_register\n");
}

module_init(gpio_sysfs_probe_init);
module_exit(gpio_sysfs_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ `MODULE_*` macros و `module_init/exit` |
| `linux/kernel.h` | `pr_info`, `pr_err`, `pr_warn` |
| `linux/kprobes.h` | الـ `kprobe` struct وكل الـ API بتاعه |
| `linux/gpio/driver.h` | `struct gpio_device` عشان نقدر نقرأ `label` و `ngpio` |

---

#### الـ `handler_pre`

**الـ `pt_regs`** بيحتوي على حالة الـ CPU registers لحظة وقوع الـ probe — بنستخدم `regs_get_kernel_argument(regs, 0)` اللي هو wrapper مستقل عن الـ architecture (x86-64 / ARM64 / RISC-V) عشان نجيب الـ argument الأول للدالة وهو `struct gpio_device *gdev`.

**الـ `gpio_device`** بتحتوي على `label` (اسم الـ chip زي `"gpio-ich"` أو `"amba-pl061"`) و `ngpio` (عدد الـ lines) و `id` (رقم تعريفي فريد)، وده اللي بنطبعه في `pr_info`.

---

#### الـ `handler_post`

**`regs_return_value(regs)`** بترجع القيمة اللي رجعتها الدالة بعد تنفيذها — لو سالبة معناه في error في التسجيل، ودي معلومة مفيدة في debugging خصوصاً على systems فيها مشكلة في الـ sysfs init.

---

#### الـ `struct kprobe`

**`symbol_name`** بدل العنوان المباشر — الـ kernel بيستخدم `kallsyms` يحوّل الاسم لعنوان وقت التسجيل، وده أأمن وأسهل من hardcode العنوان اللي بيتغير بين builds.

---

#### الـ `module_exit`

**`unregister_kprobe`** ضروري جداً — لو نزلنا الـ module من غير ما نشيل الـ kprobe، أي GPIO chip بتتسجل بعد كده هتحاول تنادي `handler_pre` اللي اتحررت من الـ memory، وده بيـcrash الـ kernel فوراً.

---

### Makefile للتجميع

```makefile
obj-m += gpio_sysfs_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### طريقة الاختبار

```bash
# تحميل الـ module
sudo insmod gpio_sysfs_probe.ko

# تحميل GPIO chip وهمي لرصد الـ probe
sudo modprobe gpio-mockup gpio_mockup_ranges="0,8"

# مشاهدة الـ log
dmesg | grep gpio_sysfs_probe

# مثال للـ output المتوقع:
# [gpio_sysfs_probe] kprobe planted on gpiochip_sysfs_register @ ffffffffc01a3bc0
# [gpio_sysfs_probe] gpiochip_sysfs_register called: label="gpio-mockup-A" ngpio=8 id=0
# [gpio_sysfs_probe] gpiochip_sysfs_register succeeded (ret=0)

# إزالة الـ module بشكل نظيف
sudo rmmod gpio_sysfs_probe
```
