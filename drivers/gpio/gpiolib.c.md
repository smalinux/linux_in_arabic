## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

**الـ `gpiolib.c`** ينتمي لـ **GPIO Subsystem** في Linux kernel، وده من أكتر الـ subsystems انتشارًا في embedded Linux. المسؤولون عنه هما Linus Walleij و Bartosz Golaszewski، وبيتحكم في كل الـ GPIO lines في النظام من أول ما بيبدأ لحد ما بيوقف.

---

### الصورة الكبيرة — قبل أي كود

#### GPIO إيه ده؟

تخيل إن عندك كباس كهرباء بسيط، ممكن تشغله أو تطفيه، وممكن كمان تحس لو حاجة ضغطت عليه. ده بالظبط معنى **GPIO** — General Purpose Input/Output. في أي معالج أو SoC (System on Chip)، في عشرات لحد مئات من الـ pins الجسدية الصغيرة، كل واحدة منهم ممكن تكون:

- **Input**: بتسمع — تقرأ حالة زرار، أو sensor، أو إشارة من chip تاني.
- **Output**: بتكلم — تشعل LED، تحرك relay، توجه إشارة لـ chip تاني.

خد مثال حقيقي: في **Raspberry Pi**، الـ 40 pin header ده بالظبط، الـ GPIO pins. لما بتكتب Python وتقول `GPIO.output(18, HIGH)` فأنت بتتكلم مع الـ GPIO subsystem في الـ kernel بالنهاية.

---

### المشكلة اللي `gpiolib.c` بيحلها

#### القصة:

في أي SoC حديث، ممكن يكون عندك:
- **GPIO controller** داخل الـ CPU نفسه (مثلًا في Raspberry Pi الـ BCM2835 عنده GPIO controller خاص بيه)
- **GPIO expander** على الـ I2C bus (زي `PCA9555` اللي بيديك 16 pin إضافي عبر سلكين بس)
- **GPIO** مدمج في PMIC (power management chip)

كل واحد من دول شغّال بطريقة مختلفة تمامًا على مستوى الـ hardware. لو مفيش subsystem موحد، كل driver محتاج يعرف بالظبط هو بيتكلم مع أنهي hardware.

الـ **GPIO subsystem** بيحل المشكلة دي بطريقة أنيقة:

```
┌─────────────────────────────────────────────────────┐
│                   User Space                        │
│         (Python scripts, systemd, etc.)             │
└────────────────────┬────────────────────────────────┘
                     │ /dev/gpiochipN  or  sysfs
┌────────────────────▼────────────────────────────────┐
│              gpiolib.c  ← الملف ده                  │
│     "المكتبة المركزية" — قلب الـ GPIO subsystem     │
│                                                     │
│  - بيسجّل كل gpio_chip في النظام                    │
│  - بيدير الـ descriptors لكل line                   │
│  - بيوفر API موحد للـ drivers والـ user space        │
└──────┬─────────────┬────────────────┬───────────────┘
       │             │                │
┌──────▼──┐   ┌──────▼──┐    ┌───────▼──┐
│BCM2835  │   │PCA9555  │    │PMIC GPIO │
│ driver  │   │ driver  │    │ driver   │
│(SoC GPIO│   │(I2C     │    │(MFD)     │
│ lines)  │   │expander)│    │          │
└─────────┘   └─────────┘    └──────────┘
```

---

### هدف الملف تحديدًا

**الـ `gpiolib.c`** هو القلب — **core implementation** — للـ GPIO subsystem. مش driver لـ hardware معين، لكنه الـ framework اللي كل الـ drivers بتتسجل فيه وكل المستخدمين بيتعاملوا معاه.

وظائفه الرئيسية:

| الوظيفة | الشرح |
|---------|-------|
| **تسجيل الـ GPIO chips** | `gpiochip_add_data_with_key()` — لما driver بيسجّل chip عنده |
| **إزالة الـ GPIO chips** | `gpiochip_remove()` — لما driver بيتنزل (hotplug) |
| **إدارة الـ descriptors** | كل GPIO line عندها `gpio_desc` بيتتبع حالتها |
| **Request / Free** | `gpiod_request()` / `gpiod_free()` — حجز GPIO لـ driver معين |
| **قراءة / كتابة القيم** | `gpiod_get_value()` / `gpiod_set_value()` |
| **إدارة الـ IRQ** | تحويل GPIO line لـ interrupt |
| **Lookup** | `gpiod_find()` — إيجاد GPIO بناءً على DT أو ACPI أو lookup tables |
| **debugfs** | `/sys/kernel/debug/gpio` عشان تشوف حالة كل الـ GPIOs |

---

### الكيانات الرئيسية في الكود

#### 1. `struct gpio_device` (الـ container الرئيسي)
```c
struct gpio_device {
    struct device    dev;      /* Linux device object */
    struct cdev      chrdev;   /* /dev/gpiochipN */
    struct gpio_chip __rcu *chip;  /* pointer للـ hardware driver */
    struct gpio_desc *descs;   /* array of descriptors, واحد لكل line */
    u16              ngpio;    /* عدد الـ lines */
    unsigned int     base;     /* رقم أول GPIO في الـ global namespace */
    const char       *label;   /* اسم الـ chip */
    ...
};
```

#### 2. `struct gpio_chip` (واجهة الـ hardware driver)
الـ driver بيملأ الـ struct ده بالـ callback functions (زي `get`, `set`, `direction_input`, `direction_output`)، وبعدين يسجلها في `gpiolib.c`.

#### 3. `struct gpio_desc` (descriptor لكل line)
```c
// كل GPIO line عندها descriptor بيحتفظ بـ:
// - flags (requested, active_low, open_drain, ...)
// - label (مين طالبها)
// - pointer لـ gdev
```

---

### قصة حياة GPIO — من أول ما بيبدأ الـ kernel

**السيناريو**: مثلًا في Raspberry Pi 4 عندنا BCM2711 GPIO controller.

```
1. Kernel boot
   └─> bcm2835_gpio driver يشتغل
       └─> بيملأ struct gpio_chip بالـ callbacks
           └─> بيستدعي gpiochip_add_data()
               └─> gpiolib.c يعمل:
                   ├─ kzalloc(gpio_device)  ← allocate الـ container
                   ├─ ida_alloc()           ← يديه ID (مثلًا gpiochip0)
                   ├─ alloc descs[]         ← array of gpio_desc
                   ├─ يضيفه لـ gpio_devices list
                   ├─ of_gpiochip_add()     ← يربطه بـ Device Tree
                   └─ gcdev_register()      ← ينشئ /dev/gpiochip0

2. Driver تاني (مثلًا LED driver) يطلب GPIO:
   └─> gpiod_get(dev, "power", GPIOD_OUT_LOW)
       └─> gpiolib.c يبحث في DT عن GPIO المرتبط بـ "power"
           └─> gpiod_request() → يحجز الـ descriptor
               └─> gc->request() ← يستدعي الـ hardware driver

3. LED driver يشعل الـ LED:
   └─> gpiod_set_value(desc, 1)
       └─> gpiolib.c يتحقق من ACTIVE_LOW flag
           └─> gc->set() ← الـ actual hardware write

4. Driver بيتنزل (rmmod):
   └─> gpiochip_remove()
       └─> gpiolib.c:
           ├─ يمسح من gpio_devices list
           ├─ rcu_assign_pointer(gdev->chip, NULL)  ← "يصمّت" الـ chip
           ├─ ينظف الـ IRQs
           └─> gpio_device_put()  ← reference count = 0 → free
```

---

### ليه الـ Descriptor Model وليس الـ Number Model؟

القديم كان بيستخدم أرقام عالمية (`gpio_get_value(42)`). المشكلة إن الرقم 42 على بورد مختلف يعني GPIO مختلف تمامًا. الجديد (وده اللي `gpiolib.c` بيبنيه) هو الـ **descriptor model**:

```c
/* القديم — LEGACY، مش بيتكتب في كود جديد */
gpio_set_value(42, 1);  /* رقم 42 يعني إيه بالظبط؟ */

/* الجديد — descriptor-based */
struct gpio_desc *led = gpiod_get(&pdev->dev, "power", GPIOD_OUT_LOW);
gpiod_set_value(led, 1);  /* واضح ومرتبط بالـ device */
```

الـ descriptor بيحمل كل المعلومات عن الـ GPIO (هو إيه، عند أنهي chip، polarity، إلخ).

---

### الـ Global GPIO Number Space — ليغاسي

في الملف في حاجة مهمة جدًا:
```c
#define GPIO_DYNAMIC_BASE  512
```
الأرقام 0-511 محجوزة للـ legacy drivers اللي لسه بتستخدم static allocation. كل الـ drivers الجديدة بتاخد أرقام من 512 فما فوق **ديناميكيًا**. والهدف النهائي هو إلغاء الـ global numberspace كلها والاعتماد على الـ descriptors فقط.

---

### الـ Locking والـ Concurrency

الـ `gpiolib.c` بيستخدم عدة آليات لحماية البيانات:

| الـ Mechanism | الاستخدام |
|--------------|-----------|
| **SRCU** (`gpio_devices_srcu`) | حماية قائمة الـ GPIO devices أثناء القراءة (read-mostly) |
| **Mutex** (`gpio_devices_lock`) | حماية القائمة أثناء الكتابة (add/remove) |
| **SRCU** (`gdev->srcu`) | حماية الـ pointer للـ chip نفسه (hotplug safety) |
| **SRCU** (`gdev->desc_srcu`) | حماية labels الـ descriptors |

ده مهم جدًا لأن الـ GPIO chips ممكن تتنزل في أي وقت (USB GPIO expander مثلًا)، وفي نفس الوقت ممكن code تاني بيقرأ قيمة من نفس الـ chip.

---

### الملفات المرتبطة اللي لازم تعرفها

#### Core Framework:
| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpiolib.c` | **الملف ده** — core implementation |
| `drivers/gpio/gpiolib.h` | internal structs (`gpio_device`, `gpio_array`) |
| `include/linux/gpio/driver.h` | واجهة الـ hardware drivers (`gpio_chip`, `gpio_irq_chip`) |
| `include/linux/gpio/consumer.h` | واجهة الـ consumers (`gpiod_get`, `gpiod_set_value`, ...) |
| `include/linux/gpio/machine.h` | lookup tables للـ board files |

#### الملفات المكملة للـ Core:
| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpiolib-cdev.c` | character device interface (`/dev/gpiochipN`) |
| `drivers/gpio/gpiolib-sysfs.c` | sysfs interface (legacy) |
| `drivers/gpio/gpiolib-of.c` | Device Tree integration |
| `drivers/gpio/gpiolib-acpi-core.c` | ACPI integration (x86/UEFI) |
| `drivers/gpio/gpiolib-swnode.c` | software nodes integration |
| `drivers/gpio/gpiolib-devres.c` | `devm_gpiod_get()` — managed resources |
| `drivers/gpio/gpiolib-legacy.c` | الـ legacy integer-based API |
| `drivers/gpio/gpiolib-shared.c` | shared GPIO functionality |

#### Hardware Drivers (أمثلة):
| الملف | الـ Hardware |
|-------|-------------|
| `drivers/gpio/gpio-pl061.c` | ARM PrimeCell PL061 (QEMU) |
| `drivers/gpio/gpio-pca953x.c` | NXP PCA953x I2C expanders |
| `drivers/gpio/gpio-mmio.c` | memory-mapped GPIO controllers |
| `drivers/gpio/gpio-sim.c` | simulated GPIO (للتطوير والاختبار) |

#### UAPI:
| الملف | الدور |
|-------|-------|
| `include/uapi/linux/gpio.h` | ioctl structures للـ character device |
| `tools/gpio/` | user-space tools (`gpioinfo`, `gpioset`, `gpioget`) |

---

### ملاحظة على الـ debugfs

لما تكتب في Linux:
```bash
cat /sys/kernel/debug/gpio
```
بتشوف output زي:
```
gpiochip0: GPUs 0 to 57, parent: platform/fe200000.gpio, pinctrl-bcm2835:
 gpio-2  (SDA1                ) in  hi
 gpio-3  (SCL1                ) in  hi
 gpio-18 (pwm0                ) out lo
```

ده كله بيتولده `gpiolib_seq_show()` في `gpiolib.c` — بيلفّ على كل الـ GPIO devices والـ descriptors المسجلين.
## Phase 2: شرح الـ GPIO (gpiolib) Framework

---

### المشكلة اللي بيحلها الـ gpiolib

في أي SoC زي Raspberry Pi أو STM32 أو i.MX8، عندك مئات الـ GPIO pins. بس كل vendor بيعمل hardware مختلف:

- Broadcom BCM2835 بيتحكم في GPIO بـ memory-mapped registers في عنوان `0x20200000`.
- NXP i.MX8 بيستخدم IP block مختلف خالص.
- بعد كده عندك GPIO expanders زي MCP23017 فوق I2C.
- وعندك FPGAs بتعمل GPIOs soft.

**المشكلة:** لو كل driver بيتعامل مع الـ GPIO بطريقته، الكود اللي بيستخدم GPIO (زي driver للـ LED أو reset line) هيكون مرتبط بـ hardware محدد. ده كارثة في portability.

**الـ gpiolib** هو الـ abstraction layer اللي بيحل المشكلة دي، بيوفر:

1. **API موحدة** لأي consumer (driver أو userspace) يطلب GPIO بغض النظر عن الـ hardware.
2. **Registration framework** لأي GPIO controller driver يسجل نفسه.
3. **Lookup system** ربط الـ GPIO بالـ device اللي بيستخدمه عن طريق DT / ACPI / machine board files.
4. **IRQ integration** تحويل GPIO line لـ interrupt source.
5. **Character device interface** للـ userspace يتعامل مع GPIO بدون sysfs legacy.

---

### الـ Solution: Descriptor-Based GPIO

الكيرنل قديمًا كان بيستخدم integer-based GPIO (زي `gpio_request(123, "label")`). ده deprecated وبيتشال تدريجيًا.

الطريقة الحديثة هي **GPIO Descriptor API**:
- كل GPIO بتاخد `struct gpio_desc *` بدل رقم.
- الـ descriptor بيحتوي على كل حاجة: الـ device اللي بيملك الـ pin، الـ flags (active-low, open-drain, ...).
- الـ lookup بيتم عن طريق DT property اسمها `gpios` أو عن طريق ACPI أو machine lookup table.

---

### مثال Real-World

خد **Raspberry Pi 4** — الـ BCM2711 SoC عنده GPIO controller بـ 58 pin. الـ driver (`drivers/gpio/gpio-bcm-virt.c` أو ARM PL061) بيسجل نفسه كـ `gpio_chip` بـ 58 GPIO.

فلو عندك driver للـ LED متوصل بـ GPIO17، هيعمل:

```c
struct gpio_desc *led = gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
gpiod_set_value(led, 1); /* turn on */
```

الـ driver مش عارف ولا محتاج يعرف إنه بيشتغل على BCM2711. ممكن نفس الكود يشتغل على i.MX8 بدون تعديل.

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    USERSPACE                                 │
│   /dev/gpiochipN (char dev)    /sys/class/gpio (legacy)     │
└───────────────┬─────────────────────────┬───────────────────┘
                │                         │
                ▼                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   GPIOLIB CORE  (gpiolib.c)                  │
│                                                              │
│  ┌─────────────────┐   ┌──────────────────┐                 │
│  │ Consumer API     │   │ Registration API │                 │
│  │ gpiod_get()      │   │ gpiochip_add_    │                 │
│  │ gpiod_set_value()│   │ data_with_key()  │                 │
│  │ gpiod_to_irq()   │   │ gpiochip_remove()│                 │
│  └────────┬────────┘   └──────────────────┘                 │
│           │                                                  │
│  ┌────────▼──────────────────────────────────────────────┐  │
│  │              gpio_devices list (sorted by base)        │  │
│  │  gpio_device ──► gpio_device ──► gpio_device          │  │
│  │   [base=0]         [base=32]       [base=512]          │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Lookup Backends:                                     │    │
│  │   gpiolib-of.c    (Device Tree)                      │    │
│  │   gpiolib-acpi.c  (ACPI / UEFI)                      │    │
│  │   gpiolib-swnode.c (Software nodes)                  │    │
│  │   gpio_lookup_list (machine board files)             │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ IRQ Integration:                                     │    │
│  │   gpiochip_add_irqchip()                             │    │
│  │   irq_domain  (flat or hierarchical)                 │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────┬──────────────────────────────────┘
                           │  calls gpio_chip callbacks
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                GPIO CONTROLLER DRIVERS                       │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  gpio-pl061  │  │ gpio-mcp23s08│  │  gpio-pca953x    │  │
│  │ (ARM PrimeCell│  │  (SPI expdr) │  │  (I2C expander)  │  │
│  │  on-chip GPIO)│  │              │  │                  │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                 │                    │             │
│         ▼                 ▼                    ▼             │
│     MMIO regs         SPI transfer         I2C transfer     │
└─────────────────────────────────────────────────────────────┘
```

---

### الـ Real-World Analogy — شركة كهرباء

تخيل إنك بتبني مصنع وعندك آلات كتير محتاجة كهرباء، بس كل آلة بيجيها كهرباء من مصدر مختلف:
- الـ floor A من شبكة الكهرباء الرئيسية (220V)
- الـ floor B من solar panels
- الـ floor C من generator

**المشكلة:** مش معقول كل آلة تعرف إنها متوصلة بإيه وتتعامل معاه مباشرة.

**الحل:** بنحط **distribution board (لوحة توزيع)** في النص. كل آلة بتقول "أنا محتاجة كهرباء من القابس رقم 5"، واللوحة بتعرف إن القابس ده متوصل بالـ solar panel وبتوصلها.

**الـ mapping للـ kernel concepts:**

| الـ Analogy | الـ Kernel Concept |
|---|---|
| المصنع كله | الـ SoC / board |
| لوحة التوزيع | **gpiolib core** |
| مصادر الكهرباء المختلفة | GPIO controllers (on-chip, I2C expander, FPGA) |
| القوابس (outlets) | الـ GPIO lines |
| الـ outlet number | `struct gpio_desc *` (descriptor) |
| الآلة اللي بتطلب كهرباء | Consumer driver (LED driver, reset driver) |
| ورقة توصيل القابس بالمصدر | Device Tree `gpio-ranges` / ACPI tables |
| رقم القابس على اللوحة | `base` في الـ `gpio_device` |
| خاصية "كهرباء 3-فاز" (special) | GPIO flags: open-drain, active-low, pull-up |
| Electrician who installs outlets | `gpiochip_add_data_with_key()` — registers the chip |
| رفع القابس لو فيه عطل | `gpiochip_remove()` |

حتى لو غيرت solar panel بـ wind turbine (بدلت GPIO expander) — الآلة مش حاسسة بحاجة، بتشتغل normal.

---

### الـ Core Abstraction

الفكرة المحورية في gpiolib هي **الفصل بين ثلاثة entities**:

```
struct gpio_chip       ──► الـ hardware-specific operations (callbacks)
struct gpio_device     ──► الـ kernel object (reference counting, lifetime mgmt)
struct gpio_desc       ──► الـ per-line state (flags, label, owner)
```

#### 1. `struct gpio_chip` — واجهة الـ Driver

```c
struct gpio_chip {
    const char *label;         /* "bcm2835-gpio" مثلاً */
    struct gpio_device *gpiodev; /* back-pointer للـ device */
    struct device *parent;     /* physical parent device */

    /* Hardware callbacks — الـ driver لازم يملأ دول */
    int  (*request)(struct gpio_chip *gc, unsigned int offset);
    void (*free)(struct gpio_chip *gc, unsigned int offset);
    int  (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)(struct gpio_chip *gc, unsigned int offset);
    int  (*set)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*set_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int  (*set_config)(struct gpio_chip *gc, unsigned int offset, unsigned long config);

    int  base;   /* أول رقم GPIO في الـ global numberspace (deprecated) */
    u16  ngpio;  /* عدد الـ GPIO lines في الـ chip ده */
    bool can_sleep; /* لو true → الـ callbacks ممكن تنام (I2C/SPI expanders) */

    struct gpio_irq_chip irq; /* IRQ integration config */
};
```

الـ `gpio_chip` هو الجزء اللي بيكتبه الـ driver author. بيملأ الـ function pointers بالكود اللي بيتكلم مع الـ hardware مباشرة.

#### 2. `struct gpio_device` — الـ Kernel Object

```c
struct gpio_device {
    struct device      dev;          /* embedded kernel device */
    struct cdev        chrdev;       /* /dev/gpiochipN */
    int                id;           /* رقم الـ chip (N في gpiochipN) */
    struct module      *owner;
    struct gpio_chip __rcu *chip;    /* RCU pointer للـ gpio_chip */
    struct gpio_desc   *descs;       /* array[ngpio] of descriptors */
    unsigned long      *valid_mask;  /* bitmap of valid lines */
    unsigned int       base;         /* first GPIO number */
    u16                ngpio;
    const char         *label;
    struct list_head   list;         /* linked in gpio_devices list */
    struct srcu_struct srcu;         /* protects chip pointer */
    struct srcu_struct desc_srcu;    /* protects descriptor labels */
    /* notifiers for line state changes */
    struct raw_notifier_head   line_state_notifier;
    struct blocking_notifier_head device_notifier;
    /* pinctrl integration */
    struct list_head   pin_ranges;
};
```

الـ `gpio_device` هو object بيعيش في الـ kernel حتى لو الـ hardware اتشال (hotplug). بيحمي الـ userspace من الـ dangling pointers. الـ `chip` field محمية بـ SRCU (Sleepable RCU) عشان ممكن يبقى NULL وقت الـ removal.

#### 3. `struct gpio_desc` — الـ Per-Line Descriptor

```c
struct gpio_desc {
    struct gpio_device *gdev;  /* الـ device اللي بيملك الـ line دي */
    unsigned long       flags; /* bitmask of GPIOD_FLAG_* */

    struct gpio_desc_label __rcu *label; /* "led-power" مثلاً */
    const char *name;                    /* "GPIO17" اسم ثابت من الـ DT */
    unsigned int debounce_period_us;     /* debounce config */
};
```

الـ flags هي قلب الـ descriptor:

```
Bit 0  GPIOD_FLAG_REQUESTED    → الـ line محجوزة (في استخدام)
Bit 1  GPIOD_FLAG_IS_OUT       → اتجاه: output
Bit 6  GPIOD_FLAG_ACTIVE_LOW   → الـ logic مقلوب (HIGH = inactive)
Bit 7  GPIOD_FLAG_OPEN_DRAIN   → open-drain mode
Bit 9  GPIOD_FLAG_USED_AS_IRQ  → الـ pin مستخدم كـ interrupt
Bit 11 GPIOD_FLAG_IS_HOGGED    → محجوز من الـ kernel نفسه (hog)
Bit 13 GPIOD_FLAG_PULL_UP      → pull-up enabled
Bit 14 GPIOD_FLAG_PULL_DOWN    → pull-down enabled
```

---

### الـ Struct Relationships

```
gpio_devices (global sorted list)
    │
    ▼
┌────────────────────────────────────────────┐
│ struct gpio_device                          │
│   dev → kernel device (refcounted)          │
│   id  = 0                                   │
│   base = 512 (dynamic)                      │
│   ngpio = 32                                │
│   label = "bcm2835-gpio"                    │
│   chip ──────────────────────────────────►  │
│   descs ──►[ desc0 | desc1 | ... | desc31 ] │
│              ▲                              │
│              │ each desc has:              │
│              │  gdev → back to gpio_device │
│              │  flags (bitmask)            │
│              │  label (RCU-protected)      │
└────────────────────────────────────────────┘
                    │
                    │ (RCU pointer)
                    ▼
┌────────────────────────────────────────────┐
│ struct gpio_chip                            │
│   label = "bcm2835-gpio"                   │
│   ngpio = 32                               │
│   base  = 512                              │
│   get_direction → bcm_gpio_get_direction() │
│   direction_input → bcm_gpio_dir_in()     │
│   direction_output → bcm_gpio_dir_out()   │
│   get → bcm_gpio_get()                    │
│   set → bcm_gpio_set()                    │
│   irq.chip → &bcm_irq_chip               │
│   irq.domain → irq_domain *              │
└────────────────────────────────────────────┘
```

**الـ RCU pointer من `gpio_device` لـ `gpio_chip`** ده critical design decision: لو الـ driver اتشال (module unload أو hotplug removal)، الكيرنل بيعمل `rcu_assign_pointer(gdev->chip, NULL)` ثم `synchronize_srcu()`. ده بيضمن إن أي thread بيقرأ الـ chip pointer هيلاقيه NULL وبيرجع `-ENODEV` بدل kernel panic.

---

### الـ gpio_chip_guard — الـ SRCU Lock Pattern

فيه pattern بيتكرر في كل operation:

```c
/* DEFINE_CLASS macro يعمل scoped SRCU guard */
CLASS(gpio_chip_guard, guard)(desc);
if (!guard.gc)
    return -ENODEV;

/* هنا آمن نستخدم guard.gc */
ret = guard.gc->get(guard.gc, gpiod_hwgpio(desc));
/* بعد الـ scope يخلص → srcu_read_unlock تتعمل automatically */
```

الـ `gpio_chip_guard` struct:

```c
struct gpio_chip_guard {
    struct gpio_device *gdev;
    struct gpio_chip   *gc;   /* NULL لو الـ chip اتشال */
    int idx;                  /* SRCU read lock index */
};
```

ده equivalent لـ `rcu_read_lock()` بس للـ sleepable SRCU، لأن بعض GPIO operations ممكن تنام (I2C/SPI expanders).

---

### Registration Flow — `gpiochip_add_data_with_key()`

لما driver بيعمل `gpiochip_add_data_with_key(gc, data, ...)`:

```
1. alloc gpio_device (kzalloc)
2. set dev.type = gpio_dev_type, dev.bus = gpio_bus_type
3. alloc IDA id  → name = "gpiochipN"
4. alloc descs[] = kcalloc(ngpio, sizeof(gpio_desc))
5. find base:
   └── gc->base < 0 → dynamic: gpiochip_find_base_unlocked()
   └── gc->base >= 0 → static (deprecated, warns)
6. gpiodev_add_to_list_unlocked() → sorted insert by base range
7. set desc[i].gdev = gdev for all i
8. gpiochip_set_names() → from DT "gpio-line-names" property
9. gpiochip_init_valid_mask() → from DT "gpio-reserved-ranges"
10. of_gpiochip_add() → DT integration
11. gpiochip_add_pin_ranges() → pinctrl integration
12. acpi_gpiochip_add() → ACPI integration
13. machine_gpiochip_add() → board file hogs
14. gpiochip_add_irqchip() → IRQ domain creation
15. gpiochip_setup_dev() → device_add() + sysfs + cdev
```

لو أي خطوة فشلت، في unwinding كامل بيتعمل (goto chain) عشان ميحصلش resource leak.

---

### IRQ Integration — الـ GPIO كـ Interrupt Source

الـ gpiolib بيدعم تحويل أي GPIO line لـ interrupt عن طريق **irqchip integration**.

#### Flat (Simple) IRQ Domain

الأكثر شيوعًا — GPIO controller بيكون له parent interrupt واحد (أو عدة):

```
         User
          │  request_irq(irq_N, handler, ...)
          ▼
   Linux IRQ subsystem
          │  maps irq_N → GPIO hwirq offset
          ▼
   gpio_irq_chip.domain  (irq_domain)
          │  gpiochip_irq_map() يربط كل hwirq بـ irq_chip
          ▼
   Parent IRQ (e.g., GIC SPI 45)
          │  gpiochip_irq_parent_handler()
          ▼
   Hardware: GPIO controller fires interrupt
```

#### Hierarchical IRQ Domain

للـ SoCs اللي فيها GIC → GPIO controller hierarchy:

```
   GPIO hwirq 5
       │
       │ child_to_parent_hwirq()
       ▼
   GIC SPI 45 (parent hwirq)
       │
       │ irq_domain_alloc_irqs_parent()
       ▼
   Linux IRQ N (virtual)
```

الـ `struct gpio_irq_chip` بيحتوي على:

```c
struct gpio_irq_chip {
    struct irq_chip    *chip;          /* الـ irq_chip callbacks */
    struct irq_domain  *domain;        /* IRQ translation domain */
    struct irq_domain  *parent_domain; /* للـ hierarchical */
    irq_flow_handler_t  handler;       /* handle_edge_irq أو handle_level_irq */
    unsigned int       *parents;       /* parent IRQ numbers */
    unsigned int        num_parents;
    bool                threaded;      /* للـ I2C/SPI expanders */
    /* callbacks للـ hierarchical */
    int (*child_to_parent_hwirq)(...);
    int (*populate_parent_alloc_arg)(...);
};
```

---

### Pinctrl Integration

الـ pinctrl subsystem (اشرحه بإيجاز: بيتحكم في مux الـ pins للـ alternate functions) بيتكامل مع gpiolib عن طريق `gpio-ranges`.

```
gpio_device.pin_ranges → list of gpio_pin_range
    │
    └── range.gc = gpio_chip *
        range.base = GPIO global number start
        range.pin_base = pinctrl pin number start
        range.npins = count
        range.pctldev = pinctrl_dev *
```

لما driver بيعمل `gpiod_request()` → الـ gpiolib بيعمل `gpiochip_generic_request()` → `pinctrl_gpio_request()` عشان يحجز الـ pin من الـ pinctrl ويحطه في GPIO mux.

---

### ما بيملكه الـ gpiolib مقابل ما بيفوضه للـ Drivers

#### الـ gpiolib بيملك (يتحكم فيه):

| Responsibility | الـ Code |
|---|---|
| Global GPIO number allocation | `gpiochip_find_base_unlocked()` |
| Descriptor lifecycle (request/free) | `gpiod_request_commit()`, `gpiod_free_commit()` |
| Flags management (active-low, open-drain, ...) | `gpiod_configure_flags()` |
| IRQ domain creation | `gpiochip_add_irqchip()` |
| Userspace interface (/dev/gpiochipN) | `gpiolib_cdev_register()` |
| Sysfs interface (legacy) | `gpiochip_sysfs_register()` |
| Thread safety (SRCU, mutexes) | `gpio_devices_lock`, `gdev->srcu` |
| Valid mask enforcement | `gpiochip_line_is_valid()` |
| Pinctrl coordination | `gpiochip_generic_request()` |
| Line state notifications | `gpiod_line_state_notify()` |

#### الـ gpiolib بيفوضه للـ Driver:

| Responsibility | الـ Callback |
|---|---|
| قراءة قيمة الـ pin من الـ register | `gc->get()` |
| كتابة قيمة على الـ pin | `gc->set()` |
| ضبط الاتجاه: input | `gc->direction_input()` |
| ضبط الاتجاه: output بقيمة | `gc->direction_output()` |
| قراءة الاتجاه الحالي | `gc->get_direction()` |
| تفعيل / تعطيل pin (power gating) | `gc->request()` / `gc->free()` |
| تطبيق pin config (pull, drive, debounce) | `gc->set_config()` |
| قراءة/كتابة أكثر من pin دفعة واحدة | `gc->get_multiple()` / `gc->set_multiple()` |
| تهيئة IRQ mask الخاص بالـ chip | `gc->irq.init_valid_mask()` |
| تهيئة pin ranges | `gc->add_pin_ranges()` |

---

### Active-Low و Open-Drain Emulation

الـ gpiolib بيعمل software emulation لـ electrical characteristics مهمة:

#### Active-Low:
```c
/* في gpiod_direction_output_nonotify() */
if (test_bit(GPIOD_FLAG_ACTIVE_LOW, &flags))
    value = !value;  /* قلب الـ value قبل ما تبعته للـ hardware */
```

يعني لو LED متوصل active-low وعملت `gpiod_set_value(desc, 1)` (يعني "enable")، الـ gpiolib هيكتب 0 على الـ pin.

#### Open-Drain Emulation:
```c
if (test_bit(GPIOD_FLAG_OPEN_DRAIN, &flags)) {
    /* أولاً: حاول تفعّلها hardware */
    ret = gpio_set_config(desc, PIN_CONFIG_DRIVE_OPEN_DRAIN);
    if (!ret) goto set_output_value;  /* hardware يدعم open-drain */

    /* Hardware مش بيدعم → emulate بـ software:
     * لو value = 1 (high): حط الـ pin في input mode (floating → pull-up يشدها high)
     * لو value = 0 (low):  حط الـ pin في output mode وكتب 0
     */
    if (value)
        goto set_output_flag;  /* switch to input mode */
}
```

---

### الـ Global GPIO Numberspace — Legacy وليه موجود

الـ kernel لسه بيحتفظ بـ global integer numberspace للـ backward compatibility:

```
GPIO 0 → 511:   reserved for legacy static allocation
GPIO 512 → INT_MAX: dynamic allocation (modern drivers)
```

الـ `gpio_devices` list بتتفرز بالـ base, وكل device بتاخد range مش بيتداخل مع غيره. الـ `gpio_to_desc()` بتلف على الـ list وتلاقي الـ device اللي فيها الـ GPIO number ده:

```c
struct gpio_desc *gpio_to_desc(unsigned gpio) {
    list_for_each_entry_srcu(gdev, &gpio_devices, list, ...) {
        if (gdev->base <= gpio && gdev->base + gdev->ngpio > gpio)
            return &gdev->descs[gpio - gdev->base];
    }
    return NULL;
}
```

الكيرنل نفسه بيقول في الكود (line 1122):
```c
/* TODO: this allocates a Linux GPIO number base in the global
 * GPIO numberspace for this chip. In the long run we want to
 * get *rid* of this numberspace and use only descriptors */
```

---

### الـ Hog Mechanism

الـ **GPIO hog** هو لما الـ kernel نفسه بيحجز GPIO line من غير أي consumer driver. بيتحدد في DT:

```dts
gpio-controller@0 {
    gpio-hog;
    gpios = <4 GPIO_ACTIVE_HIGH>;
    output-high;
    line-name = "enable-wifi";
};
```

الـ `gpiochip_machine_hog()` بيعمل `gpiod_hog()` اللي بيعمل `gpiod_request()` ويفضل requested طول ما الـ system شغال. ده مفيد لـ hardware lines زي reset pins اللي لازم تكون في state معين من البداية.

---

### الـ Concurrency Model

الـ gpiolib بيستخدم عدة locking mechanisms:

```
┌─────────────────────────────────────────────────────────┐
│ Lock                 │ يحمي إيه                          │
├─────────────────────────────────────────────────────────┤
│ gpio_devices_lock    │ insertion/deletion في gpio_devices │
│ (mutex)              │ list                               │
├─────────────────────────────────────────────────────────┤
│ gpio_devices_srcu    │ read traversal للـ gpio_devices    │
│ (SRCU)               │ list                               │
├─────────────────────────────────────────────────────────┤
│ gdev->srcu           │ الـ gpio_chip pointer (chip removal │
│ (SRCU)               │ safety)                            │
├─────────────────────────────────────────────────────────┤
│ gdev->desc_srcu      │ الـ gpio_desc label pointers       │
│ (SRCU)               │                                    │
├─────────────────────────────────────────────────────────┤
│ gpio_lookup_lock     │ gpio_lookup_list (board file tables │
│ (mutex)              │                                    │
├─────────────────────────────────────────────────────────┤
│ desc->flags          │ atomic READ_ONCE/WRITE_ONCE        │
│ (atomic ops)         │                                    │
└─────────────────────────────────────────────────────────┘
```

الـ **SRCU (Sleepable RCU)** بيُستخدم هنا بدل العادي RCU لأن GPIO operations ممكن تنام (I2C expanders بتاخد وقت). الـ SRCU بيسمح لـ read-side critical sections تنام، بعكس الـ RCU العادي.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options

#### الـ `GPIOD_FLAG_*` — بتات الـ `gpio_desc.flags`

| Bit | الـ Macro | المعنى |
|-----|-----------|---------|
| 0 | `GPIOD_FLAG_REQUESTED` | الـ GPIO مستخدم حالياً (مطلوب من consumer) |
| 1 | `GPIOD_FLAG_IS_OUT` | اتجاه output |
| 2 | `GPIOD_FLAG_EXPORT` | exported للـ user-space |
| 3 | `GPIOD_FLAG_SYSFS` | exported عبر `/sys/class/gpio` |
| 6 | `GPIOD_FLAG_ACTIVE_LOW` | منطق مقلوب — الـ "1" logic = الـ "0" physical |
| 7 | `GPIOD_FLAG_OPEN_DRAIN` | open-drain: لما output=1 بتحول input |
| 8 | `GPIOD_FLAG_OPEN_SOURCE` | open-source: لما output=0 بتحول input |
| 9 | `GPIOD_FLAG_USED_AS_IRQ` | الـ line مستخدم كـ interrupt |
| 10 | `GPIOD_FLAG_IRQ_IS_ENABLED` | الـ IRQ فعّال حالياً |
| 11 | `GPIOD_FLAG_IS_HOGGED` | الـ driver حجز الـ line لنفسه (hog) |
| 12 | `GPIOD_FLAG_TRANSITORY` | ممكن يفقد قيمته في sleep/reset |
| 13 | `GPIOD_FLAG_PULL_UP` | pull-up مفعّل |
| 14 | `GPIOD_FLAG_PULL_DOWN` | pull-down مفعّل |
| 15 | `GPIOD_FLAG_BIAS_DISABLE` | bias معطّل صراحةً |
| 16 | `GPIOD_FLAG_EDGE_RISING` | CDEV بيراقب rising edge |
| 17 | `GPIOD_FLAG_EDGE_FALLING` | CDEV بيراقب falling edge |
| 18 | `GPIOD_FLAG_EVENT_CLOCK_REALTIME` | timestamps من REALTIME clock |
| 19 | `GPIOD_FLAG_EVENT_CLOCK_HTE` | timestamps من hardware timestamp engine |
| 20 | `GPIOD_FLAG_SHARED` | الـ GPIO مشترك بين أكتر من consumer |
| 21 | `GPIOD_FLAG_SHARED_PROXY` | virtual proxy لـ physically shared pin |

#### الـ `enum gpio_lookup_flags` — بتاع lookup

| القيمة | المعنى |
|--------|---------|
| `GPIO_ACTIVE_HIGH` (0) | الـ "1" logic = high physical |
| `GPIO_ACTIVE_LOW` (1) | الـ "1" logic = low physical |
| `GPIO_OPEN_DRAIN` (2) | open-drain configuration |
| `GPIO_OPEN_SOURCE` (4) | open-source configuration |
| `GPIO_TRANSITORY` (8) | يفقد القيمة في sleep |
| `GPIO_PULL_UP` (16) | pull-up |
| `GPIO_PULL_DOWN` (32) | pull-down |
| `GPIO_PULL_DISABLE` (64) | بدون bias |

#### الـ Config Options الأساسية

| الـ Kconfig | التأثير |
|-------------|---------|
| `CONFIG_GPIOLIB_IRQCHIP` | يفعّل كود الـ `gpio_irq_chip` داخل gpiolib |
| `CONFIG_IRQ_DOMAIN_HIERARCHY` | يدعم الـ hierarchical interrupt domains |
| `CONFIG_PINCTRL` | يفعّل ربط الـ GPIO بالـ pinctrl (الـ `pin_ranges`) |
| `CONFIG_GPIO_CDEV` | يفعّل الـ character device interface |
| `CONFIG_OF_GPIO` | دعم Device Tree |
| `CONFIG_OF_DYNAMIC` | دعم DT hotplug |
| `CONFIG_HTE` | Hardware Timestamp Engine |
| `CONFIG_GPIOLIB_FASTPATH_LIMIT` | عدد الـ GPIOs في fast path لـ set/get array |

#### الـ Constants المهمة

| الـ Macro | القيمة | الغرض |
|-----------|--------|--------|
| `GPIO_DYNAMIC_BASE` | 512 | بداية الـ dynamic allocation |
| `GPIO_DYNAMIC_MAX` | `INT_MAX` | نهاية الـ global numberspace |
| `GPIO_DEV_MAX` | 256 | أقصى عدد GPIO chip devices |
| `FASTPATH_NGPIO` | `CONFIG_GPIOLIB_FASTPATH_LIMIT` | حد الـ fast path للـ array ops |

---

### 1. الـ Structs المهمة

#### 1.1 `struct gpio_device` — الحاوية الرئيسية

**الغرض:** الـ state container الداخلي لكل GPIO controller. بتعيش حتى بعد ما الـ chip يتشال، طالما في user-space بيستخدمها. دي هي الـ reference-counted object.

**الفيلدات:**

| الفيلد | النوع | الشرح |
|--------|-------|--------|
| `dev` | `struct device` | الـ kernel device object — basis of reference counting |
| `chrdev` | `struct cdev` | الـ character device لـ `/dev/gpiochipN` |
| `id` | `int` | رقم unique من الـ `gpio_ida` |
| `owner` | `struct module *` | الـ module المالك — يمنع unload |
| `chip` | `struct gpio_chip __rcu *` | مؤشر RCU للـ chip الفعلي — يبقى NULL بعد removal |
| `descs` | `struct gpio_desc *` | array بعدد `ngpio` من الـ descriptors |
| `valid_mask` | `unsigned long *` | bitmap: أي pins صالحة للاستخدام |
| `desc_srcu` | `struct srcu_struct` | يحمي labels الـ descriptors من concurrent access |
| `base` | `unsigned int` | رقم أول GPIO في الـ global numberspace (deprecated) |
| `ngpio` | `u16` | عدد GPIO lines |
| `can_sleep` | `bool` | هل الـ chip callbacks ممكن تنام (I2C/SPI expanders) |
| `label` | `const char *` | اسم تعريفي للـ chip |
| `data` | `void *` | driver-private data |
| `list` | `struct list_head` | بيربطه في الـ `gpio_devices` global list |
| `line_state_notifier` | `struct raw_notifier_head` | يُبلّغ المشتركين بتغيرات الـ lines |
| `line_state_lock` | `rwlock_t` | يحمي الـ notifier من IRQ context |
| `line_state_wq` | `struct workqueue_struct *` | لإرسال events في process context |
| `device_notifier` | `struct blocking_notifier_head` | يُبلّغ CDEV waitqueues بإزالة الـ device |
| `srcu` | `struct srcu_struct` | يحمي المؤشر للـ `chip` من concurrent access |
| `pin_ranges` | `struct list_head` | قائمة GPIO↔pinctrl ranges (بـ `CONFIG_PINCTRL`) |

**علاقاتها:** تحتوي على `gpio_chip` عبر RCU pointer. تملك array من `gpio_desc`.

---

#### 1.2 `struct gpio_chip` — الواجهة البرمجية للـ Driver

**الغرض:** الـ driver بيملأ الـ struct ده ويسجّله. فيه الـ callbacks التي تنفّذ العمليات على الـ hardware الفعلي.

**الفيلدات:**

| الفيلد | النوع | الشرح |
|--------|-------|--------|
| `label` | `const char *` | اسم الـ chip (مثلاً "gpio-ich") |
| `gpiodev` | `struct gpio_device *` | back-pointer للـ state container |
| `parent` | `struct device *` | الـ parent device |
| `fwnode` | `struct fwnode_handle *` | firmware node اختياري |
| `owner` | `struct module *` | deprecated، استخدم `gc->parent->driver->owner` |
| `request` | callback | يُفعّل الـ pin (clock/power) |
| `free` | callback | يُعطّل الـ pin |
| `get_direction` | callback | يقرأ اتجاه الـ pin: 0=out, 1=in |
| `direction_input` | callback | يضبط الـ pin كـ input |
| `direction_output` | callback | يضبط الـ pin كـ output مع initial value |
| `get` | callback | يقرأ القيمة: 0=low, 1=high |
| `get_multiple` | callback | يقرأ قيم متعددة بـ bitmask |
| `set` | callback | يكتب قيمة |
| `set_multiple` | callback | يكتب قيم متعددة بـ bitmask |
| `set_config` | callback | ضبط pinconf (bias, drive mode, debounce) |
| `to_irq` | callback | يحوّل GPIO offset لرقم IRQ |
| `init_valid_mask` | callback | يحدد أي pins صالحة |
| `add_pin_ranges` | callback | يضيف GPIO↔pin ranges |
| `en_hw_timestamp` | callback | يفعّل hardware timestamps |
| `dis_hw_timestamp` | callback | يعطّل hardware timestamps |
| `base` | `int` | أول GPIO number — سالب = dynamic |
| `ngpio` | `u16` | عدد الـ lines |
| `offset` | `u16` | offset لما أكتر من chip per device |
| `names` | `const char * const *` | أسماء الـ lines |
| `can_sleep` | `bool` | هل الـ callbacks ممكن تنام |
| `irq` | `struct gpio_irq_chip` | embedded IRQ chip (بـ `CONFIG_GPIOLIB_IRQCHIP`) |
| `of_gpio_n_cells` | `unsigned int` | عدد cells في DT specifier |
| `of_xlate` | callback | ترجمة DT specifier لـ hwnum + flags |

---

#### 1.3 `struct gpio_desc` — الـ Descriptor لكل Line

**الغرض:** الـ opaque handle اللي الـ consumers بيشتغلوا بيه. واحدة per GPIO line. بتعيش في `gdev->descs[hwnum]`.

**الفيلدات:**

| الفيلد | النوع | الشرح |
|--------|-------|--------|
| `gdev` | `struct gpio_device *` | back-pointer للـ parent device |
| `flags` | `unsigned long` | bitmask من `GPIOD_FLAG_*` — atomic ops بـ `test_bit`/`set_bit` |
| `label` | `struct gpio_desc_label __rcu *` | اسم الـ consumer — RCU-protected |
| `name` | `const char *` | اسم الـ line (من DT أو driver) |
| `hog` | `struct device_node *` | DT node اللي hog الـ line (بـ `CONFIG_OF_DYNAMIC`) |
| `debounce_period_us` | `unsigned int` | فترة debounce بالـ microseconds (بـ `CONFIG_GPIO_CDEV`) |

---

#### 1.4 `struct gpio_irq_chip` — الـ IRQ Controller المدمج

**الغرض:** embedded في `gpio_chip`، بيحوّل الـ GPIO chip لـ interrupt controller.

**الفيلدات المهمة:**

| الفيلد | النوع | الشرح |
|--------|-------|--------|
| `chip` | `struct irq_chip *` | الـ irqchip implementation |
| `domain` | `struct irq_domain *` | maps gpio hwirq → linux IRQ number |
| `fwnode` | `struct fwnode_handle *` | لدعم hierarchical domains |
| `parent_domain` | `struct irq_domain *` | إذا موجود = hierarchical mode |
| `child_to_parent_hwirq` | callback | يترجم GPIO hwirq لـ parent hwirq |
| `populate_parent_alloc_arg` | callback | يملأ parent alloc info |
| `child_offset_to_irq` | callback | يحوّل GPIO offset لـ child IRQ number |
| `child_irq_domain_ops` | `struct irq_domain_ops` | ops لـ hierarchical domain |
| `handler` | `irq_flow_handler_t` | IRQ handler (مثلاً `handle_simple_irq`) |
| `default_type` | `unsigned int` | النوع الافتراضي (`IRQ_TYPE_*`) |
| `parent_handler` | `irq_flow_handler_t` | chained handler للـ parent IRQ |
| `num_parents` | `unsigned int` | عدد parent IRQs |
| `parents` | `unsigned int *` | قائمة parent IRQ numbers |
| `map` | `unsigned int *` | per-GPIO parent IRQ mapping |
| `valid_mask` | `unsigned long *` | bitmap: أي GPIO lines تقدر تكون IRQs |
| `threaded` | `bool` | هل الـ handlers nested threads |
| `initialized` | `bool` | barrier flag — لازم `true` قبل ما `to_irq` يشتغل |

---

#### 1.5 `struct gpio_array` — الـ Fast Path للـ Array Operations

**الغرض:** يُخزّن metadata يسمح بـ bulk get/set على نفس الـ chip في عملية واحدة بدل loop.

| الفيلد | الشرح |
|--------|--------|
| `desc` | pointer لـ array الـ descriptors |
| `size` | عدد العناصر |
| `gdev` | الـ parent device للـ fast path |
| `get_mask` | bitmap: أي GPIOs تتقرأ بالـ fast path |
| `set_mask` | bitmap: أي GPIOs تتكتب بالـ fast path |
| `invert_mask[]` | bitmap: أي GPIOs active-low (محتاجة XOR) |

---

#### 1.6 `struct gpio_desc_label` — الـ RCU-Safe Label

```c
struct gpio_desc_label {
    struct rcu_head rh;  /* for call_srcu() free */
    char str[];          /* flexible array: the label string */
};
```
**بيتحرر** عبر `call_srcu()` لضمان عدم وجود readers نشطين.

---

#### 1.7 `struct gpio_chip_guard` — الـ SRCU Lock Guard

```c
struct gpio_chip_guard {
    struct gpio_device *gdev;
    struct gpio_chip *gc;   /* NULL لو الـ chip اترفع */
    int idx;                /* SRCU read lock index */
};
```
**بيشتغل** كـ scoped guard بـ `CLASS(gpio_chip_guard, guard)(desc)`. بيعمل `srcu_read_lock` على `gdev->srcu` ويجيب `gc` بشكل آمن.

---

#### 1.8 `struct gpiod_lookup_table` + `struct gpiod_lookup`

```c
struct gpiod_lookup {
    const char *key;        /* chip label or GPIO line name */
    u16 chip_hwnum;         /* hardware offset, U16_MAX = use line name */
    const char *con_id;     /* "reset", "enable", etc. */
    unsigned int idx;       /* index within same con_id */
    unsigned long flags;    /* GPIO_ACTIVE_LOW, GPIO_OPEN_DRAIN... */
};

struct gpiod_lookup_table {
    struct list_head list;  /* في gpio_lookup_list */
    const char *dev_id;     /* اسم الـ consumer device */
    struct gpiod_lookup table[]; /* zero-terminated */
};
```

---

#### 1.9 `struct gpiod_hog`

```c
struct gpiod_hog {
    struct list_head list;
    const char *chip_label;  /* اسم الـ chip */
    u16 chip_hwnum;          /* الـ GPIO offset */
    const char *line_name;   /* consumer label */
    unsigned long lflags;    /* lookup flags */
    int dflags;              /* gpiod_flags */
};
```

---

#### 1.10 `struct gpio_pin_range` (بـ `CONFIG_PINCTRL`)

```c
struct gpio_pin_range {
    struct list_head node;
    struct pinctrl_dev *pctldev;
    struct pinctrl_gpio_range range;
};
```
بيربط GPIO offset بـ pinctrl pin number.

---

### 2. رسم علاقات الـ Structs

```
                     gpio_devices (global list, SRCU-protected)
                          │
              ┌───────────┴───────────┐
              │                       │
    ┌─────────▼──────────┐  ┌────────▼──────────┐
    │   gpio_device      │  │   gpio_device      │  ...
    │ ─────────────────  │  │ ─────────────────  │
    │ dev (struct device)│  │ dev (struct device)│
    │ chrdev (cdev)      │  │ chrdev (cdev)      │
    │ id, base, ngpio    │  │                    │
    │ label, data        │  └────────────────────┘
    │ srcu (chip guard)  │
    │ desc_srcu          │
    │ valid_mask*        │
    │ line_state_notifier│
    │ ──────────────     │
    │ chip ──RCU──────────────────────────────────────────┐
    │ descs ──────────────────────────────────────────┐   │
    │ pin_ranges──►[gpio_pin_range]─►[pinctrl_dev]    │   │
    └─────────────────────────────────────────────────│───┘
                                                      │   │
              ┌───────────────────────────────────────┘   │
              │  descs[0..ngpio-1]                         │
    ┌─────────▼──────────┐                      ┌──────────▼──────────┐
    │    gpio_desc        │ ×ngpio               │    gpio_chip         │
    │ ─────────────────  │                      │ ─────────────────── │
    │ gdev ◄─────────────┼──(back-ptr)          │ gpiodev ◄───────────┼─(back)
    │ flags (atomic bits)│                      │ parent, fwnode       │
    │ label (RCU ptr)────┼──►[gpio_desc_label]  │ callbacks:           │
    │ name               │                      │  .request/.free      │
    │ debounce_period_us │                      │  .get/.set           │
    └────────────────────┘                      │  .direction_*        │
                                                │  .set_config         │
                                                │  .to_irq             │
                                                │ base, ngpio, names   │
                                                │ irq ──────────────── │
                                                └──────────────────────┘
                                                         │
                                               ┌─────────▼──────────────┐
                                               │   gpio_irq_chip         │
                                               │ ─────────────────────  │
                                               │ chip ──►[irq_chip]      │
                                               │ domain ──►[irq_domain]  │
                                               │ parent_domain (hier.)   │
                                               │ parents[], map[]        │
                                               │ valid_mask bitmap       │
                                               │ handler, default_type   │
                                               └─────────────────────────┘

gpio_lookup_list (global, mutex-protected)
    │
    └──►[gpiod_lookup_table]──►[gpiod_lookup entries]
              │ dev_id matches consumer

gpio_machine_hogs (global, mutex-protected)
    └──►[gpiod_hog]──►[gpiod_hog]──►...
```

---

### 3. دورة حياة الـ GPIO Device

```
Driver calls gpiochip_add_data()
    │
    ├─[1] kzalloc gpio_device (gdev)
    ├─[2] ida_alloc → gdev->id
    ├─[3] dev_set_name → "gpiochipN"
    ├─[4] gpiochip_get_ngpios() → gdev->ngpio
    ├─[5] kcalloc(ngpio) → gdev->descs
    ├─[6] kstrdup_const → gdev->label
    ├─[7] init_srcu_struct (gdev->srcu + gdev->desc_srcu)
    ├─[8] mutex_lock(gpio_devices_lock)
    │       └─ gpiochip_find_base_unlocked() → base (dynamic)
    │       └─ gpiodev_add_to_list_unlocked() → gpio_devices list
    │     mutex_unlock
    ├─[9] gpiochip_set_desc_names() / gpiochip_set_names()
    ├─[10] gpiochip_init_valid_mask()
    ├─[11] for each desc: desc->gdev = gdev, init GPIOD_FLAG_IS_OUT
    ├─[12] of_gpiochip_add() — DT integration
    ├─[13] gpiochip_add_pin_ranges() — pinctrl integration
    ├─[14] acpi_gpiochip_add()
    ├─[15] machine_gpiochip_add() — process hogs
    ├─[16] gpiochip_irqchip_init_valid_mask()
    ├─[17] gpiochip_irqchip_init_hw()
    ├─[18] gpiochip_add_irqchip()
    │       ├─ gpiochip_hierarchy_create_domain() OR
    │       └─ gpiochip_simple_create_domain()
    │       └─ gpiochip_set_irq_hooks()
    ├─[19] gpio_device_setup_shared()
    └─[20] gpiochip_setup_dev()
            ├─ gcdev_register() → device_add() + cdev
            └─ gpiochip_sysfs_register()

          [GPIO device LIVE — consumers can request GPIOs]

Driver calls gpiochip_remove()
    │
    ├─[1] gpiochip_sysfs_unregister()
    ├─[2] gpiochip_free_hogs()
    ├─[3] gpiochip_free_remaining_irqs()
    ├─[4] list_del_rcu(gdev->list) + synchronize_srcu(gpio_devices_srcu)
    ├─[5] rcu_assign_pointer(gdev->chip, NULL) ← chip "numbed"
    ├─[6] synchronize_srcu(gdev->srcu) ← wait for chip users
    ├─[7] gpio_device_teardown_shared()
    ├─[8] gpiochip_irqchip_remove()
    ├─[9] acpi_gpiochip_remove() / of_gpiochip_remove()
    ├─[10] gpiochip_remove_pin_ranges()
    ├─[11] gpiochip_free_valid_mask()
    ├─[12] gpiochip_set_data(NULL)
    ├─[13] gcdev_unregister() ← cdev + sysfs removed
    └─[14] gpio_device_put() ← decrement ref count
              │
              └─ if refcount=0: gpiodev_release()
                    ├─ synchronize_srcu(desc_srcu)
                    ├─ cleanup_srcu_struct(desc_srcu + srcu)
                    ├─ ida_free(gdev->id)
                    ├─ kfree_const(label)
                    ├─ kfree(descs)
                    └─ kfree(gdev)
```

---

### 4. دورة حياة طلب الـ GPIO (Request/Free)

```
Consumer: gpiod_get(dev, "reset", GPIOD_OUT_HIGH)
    │
    └─► gpiod_get_index()
            └─► gpiod_find_and_request()
                    ├─ scoped_guard(srcu, gpio_devices_srcu)
                    ├─ gpiod_fwnode_lookup()
                    │       ├─ of_find_gpio() [DT]
                    │       ├─ acpi_find_gpio() [ACPI]
                    │       └─ swnode_find_gpio() [swnode]
                    ├─ gpiod_find() [platform lookup table fallback]
                    │       └─ gpio_lookup_lock (mutex)
                    ├─ gpiod_request(desc, label)
                    │       ├─ try_module_get(gdev->owner)
                    │       ├─ gpiod_request_commit()
                    │       │       ├─ CLASS(gpio_chip_guard, guard)(desc)
                    │       │       ├─ test_and_set_bit(GPIOD_FLAG_REQUESTED)
                    │       │       ├─ gc->request(gc, offset) [optional]
                    │       │       ├─ gc->get_direction() [cache direction]
                    │       │       └─ desc_set_label(desc, label)
                    │       └─ gpio_device_get(gdev) ← refcount++
                    └─ gpiod_configure_flags()
                            ├─ set ACTIVE_LOW/OPEN_DRAIN/PULL_* bits
                            └─ gpiod_direction_output_nonotify() OR
                               gpiod_direction_input_nonotify()

          [descriptor valid for use]

Consumer: gpiod_put(desc)
    └─► gpiod_free(desc)
            ├─ gpiod_free_commit()
            │       ├─ CLASS(gpio_chip_guard, guard)(desc)
            │       ├─ gc->free(gc, offset) [optional]
            │       ├─ clear all GPIOD_FLAG_* bits
            │       └─ desc_set_label(desc, NULL)
            ├─ module_put(gdev->owner)
            └─ gpio_device_put(gdev) ← refcount--
```

---

### 5. Call Flow Diagrams

#### 5.1 قراءة قيمة GPIO (`gpiod_get_value`)

```
consumer: gpiod_get_value(desc)
    │
    ├─ VALIDATE_DESC(desc) ← NULL or ERR_PTR check
    ├─ WARN_ON(desc->gdev->can_sleep) ← no sleeping in atomic
    └─► gpiod_get_raw_value_commit(desc)
            ├─ gdev = desc->gdev
            ├─ guard(srcu)(&gdev->srcu) ← lock chip pointer
            ├─ gc = srcu_dereference(gdev->chip, &gdev->srcu)
            ├─ if !gc → return -ENODEV
            └─► gpio_chip_get_value(gc, desc)
                    └─► gpiochip_get(gc, hwgpio)
                            ├─ lockdep_assert_held(&gc->gpiodev->srcu)
                            └─► gc->get(gc, offset) ← hardware read
                                    └─► [driver reads GPIO register]
    │
    └─ if GPIOD_FLAG_ACTIVE_LOW: value = !value
    └─ return logical value
```

#### 5.2 كتابة قيمة GPIO (`gpiod_set_value`)

```
consumer: gpiod_set_value(desc, 1)
    │
    ├─ VALIDATE_DESC(desc)
    ├─ WARN_ON(can_sleep)
    └─► gpiod_set_value_nocheck(desc, 1)
            ├─ if ACTIVE_LOW: value = !value (→ 0)
            ├─ if OPEN_DRAIN:
            │       └─► gpio_set_open_drain_value_commit(desc, 0)
            │               ├─ value=0: gpiochip_direction_output(gc, offset, 0)
            │               └─ value=1: gpiochip_direction_input(gc, offset)
            ├─ elif OPEN_SOURCE:
            │       └─► gpio_set_open_source_value_commit()
            └─ else:
                    └─► gpiod_set_raw_value_commit(desc, value)
                            ├─ check GPIOD_FLAG_IS_OUT → else -EPERM
                            ├─ CLASS(gpio_chip_guard, guard)(desc)
                            ├─ trace_gpio_value(...)
                            └─► gpiochip_set(guard.gc, hwgpio, value)
                                    ├─ lockdep_assert_held(&gc->gpiodev->srcu)
                                    └─► gc->set(gc, offset, value)
                                            └─► [driver writes GPIO register]
```

#### 5.3 ضبط الاتجاه (`gpiod_direction_input`)

```
consumer: gpiod_direction_input(desc)
    │
    ├─ VALIDATE_DESC(desc)
    └─► gpiod_direction_input_nonotify(desc)
            ├─ CLASS(gpio_chip_guard, guard)(desc)
            ├─ if gc->direction_input:
            │       └─► gpiochip_direction_input(guard.gc, hwgpio)
            │               ├─ lockdep_assert_held(&gc->gpiodev->srcu)
            │               └─► gc->direction_input(gc, offset)
            │                       └─► [driver configures pin as input]
            ├─ clear_bit(GPIOD_FLAG_IS_OUT, &desc->flags)
            └─► gpio_set_bias(desc) ← apply pull-up/down/disable
    │
    └─► gpiod_line_state_notify(desc, GPIO_V2_LINE_CHANGED_CONFIG)
            └─► raw_notifier_call_chain(&gdev->line_state_notifier, ...)
```

#### 5.4 تسجيل الـ IRQ Chip

```
gpiochip_add_irqchip(gc, lock_key, request_key)
    │
    ├─ if !gc->irq.chip → return 0 (no irqchip)
    ├─ if gc->irq.parent_handler && gc->can_sleep → error
    ├─ type = gc->irq.default_type
    │
    ├─ if gpiochip_hierarchy_is_hierarchical(gc):  [parent_domain != NULL]
    │       └─► gpiochip_hierarchy_create_domain(gc)
    │               ├─ gpiochip_hierarchy_setup_domain_ops()
    │               │       ├─ ops.alloc = gpiochip_hierarchy_irq_domain_alloc
    │               │       ├─ ops.activate = gpiochip_irq_domain_activate
    │               │       └─ ops.translate = gpiochip_hierarchy_irq_domain_translate
    │               └─► irq_domain_create_hierarchy(parent_domain, ...)
    │
    └─ else:
            └─► gpiochip_simple_create_domain(gc)
                    └─► irq_domain_create_simple(fwnode, ngpio, first,
                                                 &gpiochip_domain_ops, gc)
    │
    ├─ if gc->irq.parent_handler (chained IRQ):
    │       └─ for each parent: irq_set_chained_handler_and_data()
    │
    ├─► gpiochip_set_irq_hooks(gc) ← wrap irq_enable/disable for tracking
    └─► gpiochip_irqchip_add_allocated_domain(gc, domain, false)
            ├─ gc->to_irq = gpiochip_to_irq
            ├─ gc->irq.domain = domain
            └─ barrier(); gc->irq.initialized = true ← safe to use
    │
    └─► acpi_gpiochip_request_interrupts(gc)
```

#### 5.5 الـ GPIO كـ IRQ (`gpiod_to_irq` + `gpiochip_lock_as_irq`)

```
consumer: gpiod_to_irq(desc)
    │
    ├─ guard(srcu)(&gdev->srcu)
    ├─ gc = srcu_dereference(gdev->chip, ...)
    └─► gc->to_irq(gc, offset)   [= gpiochip_to_irq]
            ├─ check gc->irq.initialized (else -EPROBE_DEFER)
            ├─ check gpiochip_irqchip_irq_valid()
            ├─ if hierarchical:
            │       └─► irq_create_fwspec_mapping()
            └─ else:
                    └─► irq_create_mapping(domain, offset)

irq framework calls: gpiochip_irq_domain_activate(domain, data, reserve)
    └─► gpiochip_lock_as_irq(gc, hwirq)
            ├─ check GPIOD_FLAG_IS_OUT && !OPEN_DRAIN → error
            ├─ set_bit(GPIOD_FLAG_USED_AS_IRQ)
            └─ set_bit(GPIOD_FLAG_IRQ_IS_ENABLED)

On IRQ: gpiochip_irq_map(d, irq, hwirq)
    ├─ irq_set_chip_data(irq, gc)
    ├─ irq_set_lockdep_class(irq, lock_key, request_key)
    ├─ irq_set_chip_and_handler(irq, gc->irq.chip, gc->irq.handler)
    └─ if gc->irq.num_parents==1: irq_set_parent(irq, parents[0])
```

#### 5.6 الـ GPIO Array Fast Path

```
consumer: gpiod_get_array_value(N, desc_array, array_info, bitmap)
    │
    └─► gpiod_get_array_value_complex(raw=false, can_sleep=false, ...)
            │
            ├─ if array_info valid (all GPIOs from same chip, sequential):
            │       ├─ guard(srcu)(&array_info->gdev->srcu)
            │       ├─ gc = srcu_dereference(...)
            │       └─► gpio_chip_get_multiple(gc, get_mask, value_bitmap)
            │               └─► gc->get_multiple(gc, mask, bits)  [ONE call!]
            │       └─ if !raw: bitmap_xor(bitmap, bitmap, invert_mask, N)
            │
            └─ else (slow path — mixed chips or non-sequential):
                    └─ while i < N:
                            ├─ CLASS(gpio_chip_guard, guard)(desc_array[i])
                            ├─ collect all GPIOs from same chip in mask
                            └─► gpio_chip_get_multiple(guard.gc, mask, bits)
                            └─ extract individual values + apply ACTIVE_LOW
```

---

### 6. استراتيجية الـ Locking

#### 6.1 جدول الـ Locks

| الـ Lock | النوع | ما يحميه | ترتيبه |
|----------|--------|----------|---------|
| `gpio_devices_lock` | `mutex` | قائمة `gpio_devices` من تعديل متزامن | 1 (أعلى) |
| `gpio_devices_srcu` | SRCU | قراءة قائمة `gpio_devices` (read-side) | 1 (read) |
| `gpio_lookup_lock` | `mutex` | قائمة `gpio_lookup_list` | مستقل |
| `gpio_machine_hogs_mutex` | `mutex` | قائمة `gpio_machine_hogs` | مستقل |
| `gdev->srcu` | SRCU | المؤشر `gdev->chip` من dangling pointer | 2 |
| `gdev->desc_srcu` | SRCU | حقل `desc->label` من dangling pointer | 3 |
| `gdev->line_state_lock` | `rwlock_t` | الـ `line_state_notifier` (يُستدعى من IRQ) | 4 (leaf) |
| `irqd->request_mutex` | `mutex` | عند free_irq في `gpiod_free_irqs` | مستقل |

#### 6.2 شرح الاستراتيجية

**SRCU بدل RCU العادي:**
الـ `gdev->srcu` بيحمي `gdev->chip` pointer. اختاروا SRCU بدل RCU لأن الـ read-side ممكن ينام (في حالة sleepable GPIO chips). الـ writer (في `gpiochip_remove`) بيعمل:
```c
rcu_assign_pointer(gdev->chip, NULL);  /* atomic set to NULL */
synchronize_srcu(&gdev->srcu);          /* wait for all readers */
```

**الـ `gpio_chip_guard`:**
الـ DEFINE_CLASS الـ scoped guard بيعمل `srcu_read_lock` في الـ constructor وبيعمل `srcu_read_unlock` في الـ destructor تلقائياً:
```c
CLASS(gpio_chip_guard, guard)(desc);
if (!guard.gc)  /* chip was removed */
    return -ENODEV;
/* gc is valid here — chip can't disappear */
```
الـ `lockdep_assert_held(&gc->gpiodev->srcu)` داخل `gpiochip_get/set/direction_*` بتتحقق إن الـ SRCU lock مسكود.

**الـ `desc->flags` — Lock-free atomic:**
كل bit operations على `desc->flags` بتتم بـ `test_bit`, `set_bit`, `clear_bit`, `assign_bit`, `READ_ONCE`, `WRITE_ONCE` — لا يحتاج lock. ده safe لأن الـ bits مستقلة ومش في ناس بتكتب على نفس الـ bit من أماكن مختلفة في نفس الوقت.

**الـ `gpio_devices_lock` ⊃ `gdev->srcu`:**
لما بتضيف chip للقائمة لازم تمسك `gpio_devices_lock` (mutex) وبعدين تعمل `list_add_rcu`. القارئين بيمسكوا `gpio_devices_srcu` فقط. الترتيب دايماً: `gpio_devices_lock` → `gdev->srcu` → `gdev->desc_srcu`.

**الـ `line_state_lock` (rwlock):**
مستخدم في `gpiod_line_state_notify` لأنه ممكن يتستدعى من IRQ context (spinlock-safe). الـ writers (subscribe/unsubscribe) بيمسكوا write lock.

#### 6.3 مخطط lock ordering

```
gpio_devices_lock (mutex)  ← لتعديل القائمة
    │
    └─► [list manipulation: list_add_rcu / list_del_rcu]

gpio_devices_srcu (SRCU read)  ← للقراءة من القائمة
    │
    └─► gdev->srcu (SRCU read)  ← للوصول لـ gdev->chip
            │
            └─► gdev->desc_srcu (SRCU read)  ← للوصول لـ desc->label
                    │
                    └─► gdev->line_state_lock (rwlock)  ← leaf lock
                                                          (IRQ-safe)

gpio_lookup_lock (mutex)  ← مستقل (لا يُمسك مع غيره)
gpio_machine_hogs_mutex (mutex)  ← مستقل
irqd->request_mutex  ← مستقل (في free_irq path)
```
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ Functions المهمة (Cheatsheet)

#### Category 1: Descriptor & Device Accessors

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `gpio_to_desc` | `struct gpio_desc *gpio_to_desc(unsigned gpio)` | تحويل رقم GPIO عالمي لـ descriptor |
| `gpio_device_get_desc` | `struct gpio_desc *gpio_device_get_desc(struct gpio_device *gdev, unsigned int hwnum)` | جيب الـ descriptor من الـ hwnum |
| `desc_to_gpio` | `int desc_to_gpio(const struct gpio_desc *desc)` | عكس: descriptor → رقم عالمي |
| `gpiod_hwgpio` | `int gpiod_hwgpio(const struct gpio_desc *desc)` | الـ hardware offset للـ desc |
| `gpiod_to_chip` | `struct gpio_chip *gpiod_to_chip(const struct gpio_desc *desc)` | *deprecated* — جيب الـ chip من الـ desc |
| `gpiod_to_gpio_device` | `struct gpio_device *gpiod_to_gpio_device(struct gpio_desc *desc)` | جيب الـ gpio_device من الـ desc |
| `gpio_device_get_base` | `int gpio_device_get_base(struct gpio_device *gdev)` | أول رقم GPIO في الـ numberspace |
| `gpio_device_get_label` | `const char *gpio_device_get_label(struct gpio_device *gdev)` | الـ label بتاعت الـ device |
| `gpio_device_get_chip` | `struct gpio_chip *gpio_device_get_chip(struct gpio_device *gdev)` | *deprecated* — جيب الـ chip من الـ gdev |

#### Category 2: Chip Registration & Removal

| Function | الغرض |
|---|---|
| `gpiochip_add_data_with_key` | الـ main registration path — بتسجّل الـ gpio_chip |
| `gpiochip_remove` | إلغاء تسجيل الـ chip وتحرير الـ resources |
| `gpiochip_setup_dev` | تسجّل الـ device في الـ bus + sysfs |
| `gpiochip_get_ngpios` | بتحدد عدد الـ GPIOs من الـ firmware أو الـ chip |
| `gpio_device_find` | بتدور على device بـ callback مخصص |
| `gpio_device_find_by_label` | بتدور بالـ label |
| `gpio_device_find_by_fwnode` | بتدور بالـ firmware node |
| `gpio_device_get` | زيادة الـ refcount |
| `gpio_device_put` | تقليل الـ refcount |
| `gpio_device_to_device` | جيب الـ `struct device *` من `gpio_device` |

#### Category 3: GPIO Request & Free

| Function | الغرض |
|---|---|
| `gpiod_request` | طلب GPIO بشكل آمن مع رفع الـ refcount |
| `gpiod_request_commit` | الـ low-level actual request |
| `gpiod_free` | تحرير GPIO + تنزيل الـ refcount |
| `gpiod_free_commit` | الـ low-level actual free |
| `gpiochip_request_own_desc` | الـ chip driver يطلب GPIO لنفسه |
| `gpiochip_free_own_desc` | تحرير GPIO طلبه الـ chip driver لنفسه |

#### Category 4: Direction Control

| Function | الغرض |
|---|---|
| `gpiod_get_direction` | اقرأ الاتجاه الحالي (0=out, 1=in) |
| `gpiod_direction_input` | اضبط الـ GPIO كـ input + notify |
| `gpiod_direction_input_nonotify` | نفسه بدون إرسال notification |
| `gpiod_direction_output` | اضبط الـ GPIO كـ output (logical) + notify |
| `gpiod_direction_output_raw` | اضبط كـ output (raw) + notify |
| `gpiod_direction_output_nonotify` | low-level output بدون notify |

#### Category 5: Value Get/Set

| Function | Raw/Logical | Sleep-safe? |
|---|---|---|
| `gpiod_get_raw_value` | raw | لا |
| `gpiod_get_value` | logical | لا |
| `gpiod_get_raw_value_cansleep` | raw | نعم |
| `gpiod_get_value_cansleep` | logical | نعم |
| `gpiod_set_raw_value` | raw | لا |
| `gpiod_set_value` | logical | لا |
| `gpiod_set_raw_value_cansleep` | raw | نعم |
| `gpiod_set_value_cansleep` | logical | نعم |
| `gpiod_get_raw_array_value` | raw array | لا |
| `gpiod_get_array_value` | logical array | لا |
| `gpiod_set_raw_array_value` | raw array | لا |
| `gpiod_set_array_value` | logical array | لا |
| `gpiod_get_array_value_complex` | core array engine | — |
| `gpiod_set_array_value_complex` | core array engine | — |

#### Category 6: Configuration

| Function | الغرض |
|---|---|
| `gpiod_set_config` | ضبط config pack (pinconf format) |
| `gpiod_set_debounce` | ضبط وقت الـ debounce |
| `gpiod_set_transitory` | تحديد هل الـ GPIO بيحتفظ بحالته أم لا |
| `gpiod_is_active_low` | اعرف هل الـ GPIO active-low |
| `gpiod_toggle_active_low` | قلب الـ active-low flag |
| `gpiod_set_consumer_name` | غيّر الاسم الخاص بالـ consumer |
| `gpio_set_debounce_timeout` | ضبط debounce + إرسال notification |
| `gpio_do_set_config` | low-level تمرير config للـ chip |

#### Category 7: IRQ Integration

| Function | الغرض |
|---|---|
| `gpiod_to_irq` | جيب رقم الـ Linux IRQ من الـ GPIO |
| `gpiochip_lock_as_irq` | اقفل الـ GPIO للاستخدام كـ IRQ |
| `gpiochip_unlock_as_irq` | افتح الـ GPIO من وضع الـ IRQ |
| `gpiochip_enable_irq` | فعّل الـ IRQ |
| `gpiochip_disable_irq` | عطّل الـ IRQ |
| `gpiochip_line_is_irq` | اعرف هل الـ line مستخدمة كـ IRQ |
| `gpiochip_reqres_irq` | اطلب الـ GPIO كـ IRQ مع الـ module ref |
| `gpiochip_relres_irq` | حرر الـ GPIO من وضع الـ IRQ |
| `gpiochip_irq_reqres` | irq_chip callback — يستدعي `gpiochip_reqres_irq` |
| `gpiochip_irq_relres` | irq_chip callback — يستدعي `gpiochip_relres_irq` |
| `gpiochip_add_irqchip` | ربط الـ irqchip بالـ gpiochip أثناء التسجيل |
| `gpiochip_irqchip_remove` | فك الارتباط أثناء الـ remove |
| `gpiochip_irqchip_add_domain` | ربط irqdomain خارجي |
| `gpiochip_irq_map` | callback للـ irqdomain لتعيين IRQ |
| `gpiochip_populate_parent_fwspec_twocell` | تعبئة fwspec للـ parent بـ 2 cells |
| `gpiochip_populate_parent_fwspec_fourcell` | تعبئة fwspec للـ parent بـ 4 cells |

#### Category 8: Pinctrl Integration

| Function | الغرض |
|---|---|
| `gpiochip_generic_request` | طلب GPIO pin من الـ pinctrl |
| `gpiochip_generic_free` | تحرير pin من الـ pinctrl |
| `gpiochip_generic_config` | تمرير config للـ pinctrl |
| `gpiochip_add_pingroup_range` | ربط GPIO range بـ pingroup |
| `gpiochip_add_pin_range_with_pins` | ربط GPIO range بقائمة pins |
| `gpiochip_remove_pin_ranges` | إزالة كل الـ pin ranges |

#### Category 9: Consumer API (Lookup & Get)

| Function | الغرض |
|---|---|
| `gpiod_get` | جيب GPIO بالاسم الوظيفي (index=0) |
| `gpiod_get_optional` | نفسه لكن يرجع NULL لو مش موجود |
| `gpiod_get_index` | جيب GPIO بالاسم الوظيفي وindex محدد |
| `gpiod_get_index_optional` | نفسه لكن يرجع NULL |
| `gpiod_get_array` | جيب كل الـ GPIOs لوظيفة معينة |
| `gpiod_get_array_optional` | نفسه لكن يرجع NULL |
| `fwnode_gpiod_get_index` | جيب GPIO من fwnode مباشرة |
| `gpiod_put` | حرر descriptor واحد |
| `gpiod_put_array` | حرر كل الـ array |
| `gpiod_count` | عدّ GPIOs لوظيفة معينة |
| `gpiod_find_and_request` | الـ core engine للبحث والطلب |
| `gpiod_configure_flags` | تطبيق flags على descriptor بعد الطلب |

#### Category 10: Hogs & Lookup Tables

| Function | الغرض |
|---|---|
| `gpiod_add_lookup_table` | سجّل جدول lookup للـ platform |
| `gpiod_add_lookup_tables` | سجّل مصفوفة جداول |
| `gpiod_remove_lookup_table` | إلغاء تسجيل جدول |
| `gpiod_add_hogs` | سجّل GPIO hogs من الـ machine code |
| `gpiod_remove_hogs` | إلغاء تسجيل hogs |
| `gpiod_hog` | احجز GPIO كـ hog بشكل دائم |

#### Category 11: Label & Validity Helpers

| Function | الغرض |
|---|---|
| `gpiod_get_label` | جيب الـ label الحالي للـ descriptor |
| `gpiod_is_equal` | قارن descriptors |
| `gpiochip_line_is_valid` | اعرف هل الـ GPIO line صالحة |
| `gpiochip_query_valid_mask` | جيب الـ valid_mask بتاع الـ chip |
| `gpiochip_line_is_irq` | هل الـ line مستخدمة كـ IRQ |
| `gpiochip_line_is_open_drain` | هل الـ line open-drain |
| `gpiochip_line_is_open_source` | هل الـ line open-source |
| `gpiochip_line_is_persistent` | هل الـ line بتحتفظ بحالتها |
| `gpiochip_dup_line_label` | نسخة من الـ label للقراءة |
| `gpiod_cansleep` | هل الـ GPIO chip ممكن ينام |
| `gpiod_is_shared` | هل الـ GPIO مشترك بين consumers |

#### Category 12: Subsystem Init & DebugFS

| Function | الغرض |
|---|---|
| `gpiolib_dev_init` | تهيئة الـ subsystem كاملاً (core_initcall) |
| `gpiolib_debugfs_init` | إنشاء `/sys/kernel/debug/gpio` |
| `gpiolib_seq_show` | عرض معلومات chip في debugfs |
| `gpiod_line_state_notify` | إرسال notification لكل المشتركين |
| `gpiochip_get_data` | جيب الـ private driver data |

---

### Group 1: Descriptor & Device Accessors

**الغرض:** ربط بين التمثيلات المختلفة للـ GPIO — رقم عالمي (legacy)، hardware offset، `gpio_desc`، `gpio_device`، `gpio_chip`.

---

#### `gpio_to_desc`

```c
struct gpio_desc *gpio_to_desc(unsigned gpio);
```

بتمشي على قائمة `gpio_devices` محمية بـ SRCU وبتدور على أي device تكون `base <= gpio < base + ngpio` وترجع الـ descriptor المقابل. دي الطريقة الوحيدة للتحويل من الـ legacy GPIO number space (رقم Linux عالمي) لـ descriptor.

- **`gpio`**: الرقم العالمي في الـ global GPIO numberspace
- **Return**: `struct gpio_desc *` أو `NULL` لو مش موجود
- **Locking**: بتاخد SRCU read lock على `gpio_devices_srcu`
- **Context**: any; SRCU لا يبلوك

---

#### `gpio_device_get_desc`

```c
struct gpio_desc *gpio_device_get_desc(struct gpio_device *gdev,
                                       unsigned int hwnum);
```

بترجع الـ descriptor المرتبط بـ hardware pin number داخل device بعينها. بتستخدم `array_index_nospec` للحماية من Spectre v1.

- **`gdev`**: الـ GPIO device
- **`hwnum`**: الـ hardware offset (0-based داخل الـ chip)
- **Return**: `struct gpio_desc *` أو `ERR_PTR(-EINVAL)` لو الـ offset أكبر من `ngpio`
- **Note**: لا ترفع الـ refcount — المتصل مسؤول إن الـ device تفضل حية

---

#### `desc_to_gpio`

```c
int desc_to_gpio(const struct gpio_desc *desc);
```

بتحسب رقم GPIO العالمي عن طريق `desc->gdev->base + (desc - &desc->gdev->descs[0])`. **deprecated** ومتوقع تتشال في المستقبل.

- **Return**: رقم عالمي (int)

---

#### `gpiod_hwgpio`

```c
int gpiod_hwgpio(const struct gpio_desc *desc);
```

بتحسب الـ hardware offset بأخذ الفرق بين موقع الـ descriptor في الـ descs array وبداية الـ array. بتُستدعى في كل مسار hot (get/set value).

- **Return**: hardware offset

---

#### `gpiod_to_chip` *(deprecated)*

```c
struct gpio_chip *gpiod_to_chip(const struct gpio_desc *desc);
```

**تحذير**: غير آمنة — الـ chip ممكن تتحرر بعد ما ترجعها من غير SRCU lock. استخدمها بس لو متأكد إن الـ device حية.

---

#### `gpio_device_get_chip` *(deprecated)*

```c
struct gpio_chip *gpio_device_get_chip(struct gpio_device *gdev);
```

بتسحب الـ chip عبر `rcu_dereference_check`. نفس التحذير.

---

### Group 2: Chip Registration & Removal

**الغرض:** دورة حياة الـ GPIO controller — من التسجيل لحد الإزالة.

---

#### `gpiochip_add_data_with_key` *(الأهم في الملف)*

```c
int gpiochip_add_data_with_key(struct gpio_chip *gc, void *data,
                               struct lock_class_key *lock_key,
                               struct lock_class_key *request_key);
```

ده الـ registration path الرئيسي. الـ macro `gpiochip_add_data` بتاخد لديه static lock class keys من الـ compiler.

**تسلسل العمليات:**

```
alloc gpio_device
  ├── set device type / bus / parent
  ├── ida_alloc → gdev->id
  ├── dev_set_name → "gpiochipN"
  ├── gpiochip_get_ngpios → تحديد عدد الـ GPIOs
  ├── alloc descs array
  ├── copy label
  ├── init notifiers (line_state, device)
  ├── init_srcu_struct (gdev->srcu, gdev->desc_srcu)
  ├── [mutex: gpio_devices_lock]
  │   ├── gpiochip_find_base_unlocked (لو base < 0)
  │   └── gpiodev_add_to_list_unlocked
  ├── gpiochip_set_desc_names (لو gc->names)
  ├── gpiochip_set_names (من device properties)
  ├── gpiochip_init_valid_mask
  ├── init desc->gdev و direction flags للكل
  ├── of_gpiochip_add
  ├── gpiochip_add_pin_ranges
  ├── acpi_gpiochip_add
  ├── machine_gpiochip_add (machine hogs)
  ├── gpiochip_irqchip_init_valid_mask
  ├── gpiochip_irqchip_init_hw
  ├── gpiochip_add_irqchip
  ├── gpio_device_setup_shared
  └── [لو gpiolib_initialized] gpiochip_setup_dev
```

- **`gc`**: الـ gpio_chip الجاهز مع الـ callbacks والـ metadata
- **`data`**: driver-private data — متاحة بعدين عبر `gpiochip_get_data()`
- **`lock_key`, `request_key`**: lockdep classes للـ IRQ locking
- **Return**: 0 نجح، negative error لو فشل. فشل هنا خطير — ممكن الـ system ما يبوتش
- **Locking**: بياخد `gpio_devices_lock` mutex لإضافة الـ gdev للقائمة
- **Caller**: driver probe function

**Error unwind path:** بتمشي عكس خطوات الـ setup — كل خطوة نجحت بتتلغى في الـ err labels.

---

#### `gpiochip_remove`

```c
void gpiochip_remove(struct gpio_chip *gc);
```

بتعكس كل عمليات `gpiochip_add_data_with_key`. التسلسل مهم جداً لأن الـ userspace ممكن يكون شايل references.

```
gpiochip_sysfs_unregister
gpiochip_free_hogs
gpiochip_free_remaining_irqs
[mutex: gpio_devices_lock] list_del_rcu
synchronize_srcu(gpio_devices_srcu)   ← انتظر أي SRCU readers ينهوا
rcu_assign_pointer(gdev->chip, NULL)  ← "numbify" الـ chip
synchronize_srcu(gdev->srcu)          ← انتظر أي readers للـ chip
gpio_device_teardown_shared
gpiochip_irqchip_remove
acpi_gpiochip_remove
of_gpiochip_remove
gpiochip_remove_pin_ranges
gpiochip_free_valid_mask
gpiochip_set_data(gc, NULL)           ← invalidate driver data
gcdev_unregister(gdev)                ← chardev unregister
gpio_device_put(gdev)                 ← آخر reference
```

- **Locking**: بياخد `gpio_devices_lock` لإزالة الـ gdev من القائمة، بعدين `synchronize_srcu` مزدوجة
- **Side effect**: لو في userspace فاتح الـ chardev، الـ gdev بتفضل حية لحد ما يقفل

---

#### `gpiochip_setup_dev`

```c
static int gpiochip_setup_dev(struct gpio_device *gdev);
```

بتسجل الـ device في الـ bus (أو تعمل chardev لو `CONFIG_GPIO_CDEV` enabled) وبعدين `gpiochip_sysfs_register`. بتُستدعى من `gpiolib_dev_init` للـ chips اللي اتسجلت قبل الـ subsystem، أو مباشرة من `gpiochip_add_data_with_key` لو الـ subsystem بالفعل initialized.

---

#### `gpio_device_find`

```c
struct gpio_device *gpio_device_find(const void *data,
                                     int (*match)(struct gpio_chip *gc,
                                                  const void *data));
```

بتمشي على قائمة `gpio_devices` بـ SRCU وبتفحص كل device بالـ `match` callback. أول ما الـ callback يرجع non-zero، بترجع reference مرفوعة للـ `gpio_device`.

- **Return**: `gpio_device *` مع refcount مرفوع، أو `NULL`
- **Note**: المتصل **لازم** يحرر الـ reference بـ `gpio_device_put()`
- **Context**: may sleep (`might_sleep()`)

---

### Group 3: GPIO Request & Free

**الغرض:** استئثار consumer بـ GPIO line واحدة، مع تتبع الـ ownership.

---

#### `gpiod_request`

```c
int gpiod_request(struct gpio_desc *desc, const char *label);
```

بتحاول تاخد `try_module_get` للـ owner module، لو نجح بتستدعي `gpiod_request_commit`، ولو نجح بترفع الـ `gpio_device` refcount.

```c
try_module_get(desc->gdev->owner)
  └── gpiod_request_commit(desc, label)
        ├── gpio_chip_guard(desc)         ← SRCU read lock + chip pointer
        ├── test_and_set_bit(GPIOD_FLAG_REQUESTED)  ← atomic claim
        ├── gpiochip_line_is_valid()
        ├── gc->request(gc, offset)       ← optional chip callback
        ├── gc->get_direction → gpiod_get_direction()
        └── desc_set_label(desc, label)   ← SRCU-protected label alloc
```

- **Return**: 0 نجح، `-EBUSY` لو الـ GPIO مطلوب بالفعل، `-EPROBE_DEFER` لو الـ module مش موجود
- **`label`**: اسم الـ consumer — بيظهر في `/sys/kernel/debug/gpio`
- **Locking**: `GPIOD_FLAG_REQUESTED` flag بيتضبط atomic — هو الـ exclusive ownership mechanism
- **Caller**: consumer drivers, usually in probe

---

#### `gpiod_free`

```c
void gpiod_free(struct gpio_desc *desc);
```

بتستدعي `gpiod_free_commit` بعدين `module_put` وبعدين `gpio_device_put`. اللي بيعمله `gpiod_free_commit`:

```c
gc->free(gc, hwgpio)           ← optional chip callback
clear_bit(GPIOD_FLAG_ACTIVE_LOW)
clear_bit(GPIOD_FLAG_REQUESTED)
clear_bit(GPIOD_FLAG_OPEN_DRAIN)
/* ... clear all flags ... */
desc_set_label(desc, NULL)     ← SRCU-protected label free (deferred)
WRITE_ONCE(desc->flags, flags) ← atomic flags update
gpiod_line_state_notify(RELEASED)
```

- **Context**: بتستدعي `might_sleep()` — ممكن تنام

---

#### `gpiochip_request_own_desc`

```c
struct gpio_desc *gpiochip_request_own_desc(struct gpio_chip *gc,
                                            unsigned int hwnum,
                                            const char *label,
                                            enum gpio_lookup_flags lflags,
                                            enum gpiod_flags dflags);
```

بتسمح للـ chip driver نفسه يطلب GPIO منه. الفرق عن `gpiod_request`: ما بترفعش reference count الـ module. بتستخدم لـ GPIO hogs والـ chip-internal control lines.

- **Return**: `struct gpio_desc *` أو `ERR_PTR()`

---

### Group 4: Direction Control

**الغرض:** تحديد اتجاه الـ GPIO (input/output) مع مراعاة الـ flags.

---

#### `gpiod_direction_input`

```c
int gpiod_direction_input(struct gpio_desc *desc);
```

بتستدعي `gpiod_direction_input_nonotify` وبعدها لو نجحت بترسل `GPIO_V2_LINE_CHANGED_CONFIG` notification.

#### `gpiod_direction_input_nonotify`

```c
int gpiod_direction_input_nonotify(struct gpio_desc *desc);
```

```
gpio_chip_guard(desc)              ← SRCU lock + chip
لو gc->direction_input موجود:
    gpiochip_direction_input(gc, offset)  ← gc->direction_input callback
لو مش موجود لكن gc->get_direction موجود:
    تحقق إن الاتجاه فعلاً input وإلا return -EIO
clear_bit(GPIOD_FLAG_IS_OUT)
gpio_set_bias(desc)               ← تطبيق pull-up/pull-down/disable
trace_gpio_direction(gpio, 1, ret)
```

- **Return**: 0 أو negative error
- **Locking**: `gpio_chip_guard` بيمسك SRCU read lock طول مدة الـ call

---

#### `gpiod_direction_output`

```c
int gpiod_direction_output(struct gpio_desc *desc, int value);
```

بتستدعي `gpiod_direction_output_nonotify` ثم notification.

#### `gpiod_direction_output_nonotify`

```c
int gpiod_direction_output_nonotify(struct gpio_desc *desc, int value);
```

الـ flow الداخلي معقد لأنه بيتعامل مع كل modes:

```
لو ACTIVE_LOW: value = !value
لو IRQ enabled على الـ line: return -EIO  ← ما ينفعش تضبط IRQ line كـ output

لو OPEN_DRAIN:
    جرب gpio_set_config(PIN_CONFIG_DRIVE_OPEN_DRAIN) بـ hardware
    لو نجح: اكمل للـ set_output_value
    لو value=1: روح set_output_flag (high-Z عبر input mode)

لو OPEN_SOURCE:
    جرب gpio_set_config(PIN_CONFIG_DRIVE_OPEN_SOURCE) بـ hardware
    لو value=0: روح set_output_flag

وإلا:
    gpio_set_config(PIN_CONFIG_DRIVE_PUSH_PULL)

set_output_value:
    gpio_set_bias(desc)
    gpiod_direction_output_raw_commit(desc, value)

set_output_flag:           ← emulation mode (high-Z)
    gpiod_direction_input_nonotify(desc)
    set_bit(GPIOD_FLAG_IS_OUT)   ← لازم عشان نقدر نغير الـ value بعدين
```

---

#### `gpiod_direction_output_raw_commit`

```c
static int gpiod_direction_output_raw_commit(struct gpio_desc *desc, int value);
```

الـ lowest-level output direction setter. لو `direction_output` موجود بيستدعيه، لو مش موجود بيعمل sanity check ثم `gpiochip_set`.

---

### Group 5: Value Get/Set

**الغرض:** قراءة وكتابة قيم الـ GPIO lines.

---

#### `gpiod_get_raw_value_commit`

```c
static int gpiod_get_raw_value_commit(const struct gpio_desc *desc);
```

الـ actual implementation للقراءة. بتاخد SRCU lock على `gdev->srcu` وبتستدعي `gc->get()` عبر `gpio_chip_get_value`.

- **Return**: 0 أو 1 للقيمة الفيزيائية، أو negative error
- **Tracing**: بتطلق `trace_gpio_value(gpio, 1, value)` للـ ftrace

---

#### `gpiod_get_value`

```c
int gpiod_get_value(const struct gpio_desc *desc);
```

بتقرأ القيمة logical (بتعكسها لو `GPIOD_FLAG_ACTIVE_LOW` set). بـ `WARN_ON` لو الـ chip ممكن ينام — استخدم `gpiod_get_value_cansleep` بدلها.

---

#### `gpiod_set_raw_value_commit`

```c
static int gpiod_set_raw_value_commit(struct gpio_desc *desc, bool value);
```

بتتحقق إن الـ flag `GPIOD_FLAG_IS_OUT` set (ما ينفعش تكتب على input) ثم بتستدعي `gpiochip_set`.

---

#### `gpiod_set_value_nocheck`

```c
static int gpiod_set_value_nocheck(struct gpio_desc *desc, int value);
```

بتطبق الـ logical semantics:
- لو `ACTIVE_LOW`: بتعكس الـ value
- لو `OPEN_DRAIN`: تستدعي `gpio_set_open_drain_value_commit`
- لو `OPEN_SOURCE`: تستدعي `gpio_set_open_source_value_commit`
- وإلا: `gpiod_set_raw_value_commit`

---

#### `gpio_set_open_drain_value_commit`

```c
static int gpio_set_open_drain_value_commit(struct gpio_desc *desc, bool value);
```

Emulation للـ open-drain في software:
- `value=1` → اضبط الـ line كـ **input** (high-Z) ← الـ pull-up يشيل الـ line لـ HIGH
- `value=0` → اضبط الـ line كـ **output=0** ← pull low actively

---

#### `gpiod_get_array_value_complex` / `gpiod_set_array_value_complex`

دول الـ core engines للـ array operations. مهمين جداً للـ performance:

```c
int gpiod_get_array_value_complex(bool raw, bool can_sleep,
                                  unsigned int array_size,
                                  struct gpio_desc **desc_array,
                                  struct gpio_array *array_info,
                                  unsigned long *value_bitmap);
```

**Fast path** (لو `array_info` valid — كل الـ GPIOs على نفس الـ chip بترتيب hardware):
```
guard(srcu)(&array_info->gdev->srcu)
gc = srcu_dereference(chip)
gpio_chip_get_multiple(gc, get_mask, value_bitmap)   ← استدعاء واحد للـ chip
[optional] bitmap_xor بالـ invert_mask للـ active-low
```

**Slow path** (GPIOs على chips مختلفة أو ترتيب مختلف):
```
for each chip group:
    DECLARE_BITMAP(fastpath_mask, FASTPATH_NGPIO)  ← stack allocation لو صغير
    جمّع كل GPIOs نفس الـ chip
    gpio_chip_get_multiple()
    apply active-low inversions
```

- **`raw`**: `true` = قيم فيزيائية، `false` = logical (يطبق ACTIVE_LOW)
- **`can_sleep`**: `true` = تقدر تستدعي من sleeping context
- **`array_info`**: الـ fast-path cache المُبني في `gpiod_get_array()`
- **`FASTPATH_NGPIO`**: `CONFIG_GPIOLIB_FASTPATH_LIMIT` — عدد الـ GPIOs اللي بنستخدم stack allocation ليها بدل heap

---

### Group 6: Configuration

**الغرض:** ضبط خصائص الـ GPIO pin عبر الـ pinconf interface.

---

#### `gpio_do_set_config`

```c
int gpio_do_set_config(struct gpio_desc *desc, unsigned long config);
```

بتمرر الـ packed pinconf config للـ `gc->set_config()`. لو الـ config كان `PIN_CONFIG_INPUT_DEBOUNCE`، بتحفظ الـ debounce value في `desc->debounce_period_us` للـ userspace.

- **Return**: 0 أو `-ENOTSUPP` لو الـ chip ما بيدعمش الـ config

---

#### `gpiod_set_config`

```c
int gpiod_set_config(struct gpio_desc *desc, unsigned long config);
```

Public wrapper فوق `gpio_do_set_config`. بعد النجاح، بترسل `GPIO_V2_LINE_CHANGED_CONFIG` notification للـ configs اللي بتهم الـ userspace (bias, drive mode, debounce).

---

#### `gpiod_configure_flags`

```c
int gpiod_configure_flags(struct gpio_desc *desc, const char *con_id,
                          unsigned long lflags, enum gpiod_flags dflags);
```

بتُستدعى من `gpiod_find_and_request` بعد الطلب. بتترجم `gpio_lookup_flags` و`gpiod_flags` لـ descriptor flags وتضبط الاتجاه:

```
lflags & GPIO_ACTIVE_LOW    → set GPIOD_FLAG_ACTIVE_LOW
lflags & GPIO_OPEN_DRAIN    → set GPIOD_FLAG_OPEN_DRAIN
lflags & GPIO_PULL_UP       → set GPIOD_FLAG_PULL_UP
... إلخ

dflags & GPIOD_FLAGS_BIT_DIR_OUT:
    → gpiod_direction_output_nonotify(desc, value)
else:
    → gpiod_direction_input_nonotify(desc)
```

---

#### `gpio_set_bias`

```c
static int gpio_set_bias(struct gpio_desc *desc);
```

بتقرأ الـ pull flags من `desc->flags` وبتختار الـ `pin_config_param` المناسب (BIAS_DISABLE / BIAS_PULL_UP / BIAS_PULL_DOWN) وبتبعته للـ `gpio_set_config_with_argument_optional`.

---

### Group 7: IRQ Integration

**الغرض:** ربط الـ GPIO lines بالـ Linux interrupt subsystem.

---

#### `gpiochip_add_irqchip`

```c
static int gpiochip_add_irqchip(struct gpio_chip *gc,
                                struct lock_class_key *lock_key,
                                struct lock_class_key *request_key);
```

بتُستدعى من `gpiochip_add_data_with_key`. الـ flow:

```
لو gc->irq.chip == NULL: return 0 (مش فيه irqchip)

لو gc->irq.parent_domain موجود:
    domain = gpiochip_hierarchy_create_domain(gc)   ← hierarchical
وإلا:
    domain = gpiochip_simple_create_domain(gc)       ← flat

لو parent_handler موجود:
    for each parent:
        irq_set_chained_handler_and_data()

gpiochip_set_irq_hooks(gc)   ← wrap hooks لو مش IMMUTABLE
gpiochip_irqchip_add_allocated_domain()
acpi_gpiochip_request_interrupts()
```

---

#### `gpiochip_set_irq_hooks`

```c
static void gpiochip_set_irq_hooks(struct gpio_chip *gc);
```

لو الـ irqchip مش `IRQCHIP_IMMUTABLE` (legacy case)، بتـ"wrap" الـ irq_enable/disable/mask/unmask callbacks بـ wrappers خاصة بـ gpiolib تستدعي `gpiochip_enable_irq`/`gpiochip_disable_irq` قبل/بعد الـ original callbacks. ده مهم لتتبع الـ `GPIOD_FLAG_IRQ_IS_ENABLED`.

---

#### `gpiochip_irq_map`

```c
static int gpiochip_irq_map(struct irq_domain *d, unsigned int irq,
                            irq_hw_number_t hwirq);
```

الـ `.map` callback للـ `irq_domain_ops`. بتُستدعى لما الـ kernel يحتاج يعمل mapping لـ hwirq جديد:

```
gpiochip_irqchip_irq_valid()           ← تحقق إن الـ GPIO line صالحة للـ IRQ
irq_set_chip_data(irq, gc)
irq_set_lockdep_class()
irq_set_chip_and_handler(irq, chip, handler)
irq_set_nested_thread() لو gc->irq.threaded
irq_set_noprobe()
irq_set_parent() لو فيه parent
irq_set_irq_type() للـ default type
```

---

#### `gpiochip_lock_as_irq`

```c
int gpiochip_lock_as_irq(struct gpio_chip *gc, unsigned int offset);
```

بتقفل الـ GPIO line للاستخدام كـ IRQ. بتتحقق إن الـ line مش configured كـ output (إلا لو open-drain). بتضبط `GPIOD_FLAG_USED_AS_IRQ` و`GPIOD_FLAG_IRQ_IS_ENABLED`.

- **Return**: 0 نجح أو `-EIO` لو configured كـ output

---

#### `gpiod_to_irq`

```c
int gpiod_to_irq(const struct gpio_desc *desc);
```

بتحول GPIO descriptor لـ Linux IRQ number. لو الـ chip عنده `gc->to_irq` callback بتستدعيه. لو الـ irqchip موجود لكن مش initialized بعد، بترجع `-EPROBE_DEFER`.

---

#### `gpiochip_hierarchy_irq_domain_alloc`

```c
static int gpiochip_hierarchy_irq_domain_alloc(struct irq_domain *d,
                                               unsigned int irq,
                                               unsigned int nr_irqs,
                                               void *data);
```

الـ `.alloc` callback للـ hierarchical irqdomain. ده مهم جداً في modern SoCs اللي فيها interrupt hierarchy (مثلاً GPIO → GIC):

```
translate(fwspec) → hwirq, type
child_to_parent_hwirq(gc, hwirq, type) → parent_hwirq, parent_type
irq_domain_set_info(d, irq, hwirq, chip, gc, handler, ...)
populate_parent_alloc_arg(gc, &gpio_parent_fwspec, parent_hwirq, parent_type)
irq_domain_alloc_irqs_parent(d, irq, 1, &gpio_parent_fwspec)
```

---

#### `gpiochip_populate_parent_fwspec_twocell` / `_fourcell`

```c
int gpiochip_populate_parent_fwspec_twocell(struct gpio_chip *gc,
                                            union gpio_irq_fwspec *gfwspec,
                                            unsigned int parent_hwirq,
                                            unsigned int parent_type);
```

بتملا `irq_fwspec` للـ parent irqdomain:
- **twocell**: `{parent_hwirq, parent_type}` — للـ GIC standard
- **fourcell**: `{0, parent_hwirq, 0, parent_type}` — للـ interrupt controllers اللي محتاجة 4 cells (مثلاً بعض SoCs)

---

### Group 8: Pinctrl Integration

**الغرض:** ربط الـ GPIO subsystem بالـ pin controller subsystem.

---

#### `gpiochip_generic_request` / `gpiochip_generic_free`

```c
int gpiochip_generic_request(struct gpio_chip *gc, unsigned int offset);
void gpiochip_generic_free(struct gpio_chip *gc, unsigned int offset);
```

الـ default `gc->request` / `gc->free` callbacks اللي ممكن الـ driver يحطهم مباشرة. بيتحققوا إن فيه pin ranges مسجلة وبعدين بيستدعوا `pinctrl_gpio_request`/`free`.

---

#### `gpiochip_add_pingroup_range`

```c
int gpiochip_add_pingroup_range(struct gpio_chip *gc,
                                struct pinctrl_dev *pctldev,
                                unsigned int gpio_offset,
                                const char *pin_group);
```

بتربط مجموعة GPIOs من الـ chip بـ named pin group في الـ pin controller. بتُستخدم بدل DT `gpio-ranges` property. **deprecated** للـ DT-based drivers.

---

#### `gpiochip_add_pin_range_with_pins`

```c
int gpiochip_add_pin_range_with_pins(struct gpio_chip *gc,
                                     const char *pinctl_name,
                                     unsigned int gpio_offset,
                                     unsigned int pin_offset,
                                     unsigned int const *pins,
                                     unsigned int npins);
```

بتربط GPIO range بـ pin range في الـ pinctrl. لو `pins != NULL`، بتسمح بـ non-consecutive mapping.

---

### Group 9: Consumer API (Lookup & Get)

**الغرض:** الـ API اللي الـ consumer drivers بتستخدمه لطلب الـ GPIOs.

---

#### `gpiod_find_and_request` *(Core Engine)*

```c
struct gpio_desc *gpiod_find_and_request(struct device *consumer,
                                         struct fwnode_handle *fwnode,
                                         const char *con_id,
                                         unsigned int idx,
                                         enum gpiod_flags flags,
                                         const char *label,
                                         bool platform_lookup_allowed);
```

الـ engine الأساسي لكل عمليات الـ GPIO lookup. الـ flow:

```
[SRCU lock على gpio_devices_srcu]
gpiod_fwnode_lookup(fwnode, ...) → يجرب DT/ACPI/swnode

لو الـ GPIO shared:
    gpio_shared_add_proxy_lookup()
    force platform lookup

لو مش لقيناه وplatform_lookup_allowed:
    gpiod_find() → يبحث في gpio_lookup_list

gpiod_request(desc, label)
gpiod_configure_flags(desc, con_id, lookupflags, flags)
gpiod_line_state_notify(REQUESTED)
```

- **`platform_lookup_allowed`**: `false` لـ `fwnode_gpiod_get_index` (no machine table), `true` لـ `gpiod_get_index`

---

#### `gpiod_get` / `gpiod_get_optional`

```c
struct gpio_desc *__must_check gpiod_get(struct device *dev,
                                         const char *con_id,
                                         enum gpiod_flags flags);
```

أبسط consumer API. بتستدعي `gpiod_get_index(dev, con_id, 0, flags)`.

- **`con_id`**: اسم الوظيفة — مثلاً `"reset"`, `"enable"`, `"cs"`
- **`flags`**: `GPIOD_IN`, `GPIOD_OUT_LOW`, `GPIOD_OUT_HIGH`, إلخ
- **Return**: `struct gpio_desc *` أو `ERR_PTR(-ENOENT)` أو `ERR_PTR(-EPROBE_DEFER)`

---

#### `gpiod_get_array`

```c
struct gpio_descs *__must_check gpiod_get_array(struct device *dev,
                                                const char *con_id,
                                                enum gpiod_flags flags);
```

بتطلب كل الـ GPIOs لوظيفة معينة. بتبني أيضاً `gpio_array` structure للـ fast-path processing:

```
count = gpiod_count(dev, con_id)
alloc gpio_descs + gpio_array (لو index 0 يساوي hwnum 0)

for each GPIO:
    gpiod_get_index(dev, con_id, i, flags)
    لو على نفس الـ fast chip وبنفس ترتيب الـ hardware:
        mark في get_mask و set_mask
    لو active-low:
        mark في invert_mask
    لو open-drain/source:
        clear من set_mask (ما ينفعش fast write)
```

**الـ fast-path conditions:**
- كل الـ GPIOs على نفس الـ chip
- الـ hardware pin numbers بنفس ترتيب الـ array indices
- مش open-drain أو open-source

---

#### `gpiod_configure_flags`

بتترجم الـ flags وتضبطها على الـ descriptor. تفاصيل في Group 6.

---

### Group 10: Hogs & Lookup Tables

**الغرض:** حجز GPIOs بشكل دائم (hogs) أو تسجيل جداول lookup للـ board-level GPIO assignments.

---

#### `gpiod_add_lookup_table` / `gpiod_remove_lookup_table`

```c
void gpiod_add_lookup_table(struct gpiod_lookup_table *table);
void gpiod_remove_lookup_table(struct gpiod_lookup_table *table);
```

بتضيف/تشيل جدول من `gpio_lookup_list` (محمية بـ `gpio_lookup_lock` mutex). الجداول دي بتُستخدم كـ fallback لو الـ DT/ACPI مش موجود.

**مثال من board file:**
```c
static struct gpiod_lookup_table my_board_leds = {
    .dev_id = "leds-gpio",
    .table = {
        GPIO_LOOKUP("gpiochip0", 10, "led-red", GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP("gpiochip0", 11, "led-green", GPIO_ACTIVE_HIGH),
        { }
    },
};
```

---

#### `gpiod_add_hogs` / `gpiod_remove_hogs`

```c
void gpiod_add_hogs(struct gpiod_hog *hogs);
void gpiod_remove_hogs(struct gpiod_hog *hogs);
```

بتسجّل GPIO hogs (GPIOs يطلبها الـ kernel لنفسه بشكل دائم). الـ hog بيتطبق على الـ chip لما بتتسجل. لو الـ chip بالفعل مسجلة، بيطبق فيها على طول.

---

#### `gpiod_hog`

```c
int gpiod_hog(struct gpio_desc *desc, const char *name,
              unsigned long lflags, enum gpiod_flags dflags);
```

بتحجز GPIO descriptor كـ hog:
```
test_and_set_bit(GPIOD_FLAG_IS_HOGGED)  ← atomic check للـ double-hog
gpiochip_request_own_desc(gc, hwnum, name, lflags, dflags)
```

---

### Group 11: Subsystem Init

---

#### `gpiolib_dev_init`

```c
static int __init gpiolib_dev_init(void);
core_initcall(gpiolib_dev_init);
```

بتُستدعى مبكراً في الـ boot sequence (`core_initcall`):

```
bus_register(&gpio_bus_type)         ← تسجيل الـ GPIO bus
driver_register(&gpio_stub_drv)      ← الـ stub driver للـ DT-only chips
alloc_chrdev_region(&gpio_devt, 0, GPIO_DEV_MAX, GPIOCHIP_NAME)
gpiolib_initialized = true
gpiochip_setup_devs()               ← setup كل الـ chips المسجلة قبل كده
[optional] of_reconfig_notifier_register  ← للـ dynamic DT
```

قبل `core_initcall`، الـ chips ممكن تتسجل (في early boot) لكن الـ device nodes ما بتتعملش. بعد `gpiolib_initialized = true`، كل chip جديدة هيتعملها device node على طول.

---

#### `gpiolib_debugfs_init`

```c
static int __init gpiolib_debugfs_init(void);
subsys_initcall(gpiolib_debugfs_init);
```

بتعمل `/sys/kernel/debug/gpio` باستخدام `seq_file` API. كل GPIO device بتظهر كـ entry مع كل GPIOs المطلوبة وحالتها.

---

#### `gpiod_line_state_notify`

```c
void gpiod_line_state_notify(struct gpio_desc *desc, unsigned long action);
```

بترسل notification لكل المشتركين في `gdev->line_state_notifier`:

```c
guard(read_lock_irqsave)(&desc->gdev->line_state_lock);
raw_notifier_call_chain(&desc->gdev->line_state_notifier, action, desc);
```

- **`action`**: `GPIO_V2_LINE_CHANGED_REQUESTED`, `GPIO_V2_LINE_CHANGED_RELEASED`, أو `GPIO_V2_LINE_CHANGED_CONFIG`
- **Context**: irq-safe — بتاخد `read_lock_irqsave`
- **Use**: الـ character device (cdev) بيشترك فيها لعمل event notifications للـ userspace

---

### ملاحظات على الـ Locking في الملف

```
gpio_devices_lock (mutex)
    ← يحمي التعديل على قائمة gpio_devices

gpio_devices_srcu
    ← يحمي الـ read-only traversal للقائمة (lock-free reads)

gdev->srcu
    ← يحمي الـ pointer لـ gdev->chip (هو اللي بيتعمله NULL في remove)

gdev->desc_srcu
    ← يحمي الـ pointer لـ desc->label

gpio_lookup_lock (mutex)
    ← يحمي gpio_lookup_list

gpio_machine_hogs_mutex (mutex)
    ← يحمي gpio_machine_hogs

gdev->line_state_lock (rwlock)
    ← يحمي line_state_notifier (write=very rare, read=on every notify)

GPIOD_FLAG_REQUESTED (atomic bit)
    ← الـ exclusive ownership mechanism للـ GPIO lines
```

**الـ gpio_chip_guard CLASS:**

```c
CLASS(gpio_chip_guard, guard)(desc);
if (!guard.gc) return -ENODEV;
```

هو scoped guard بيمسك `srcu_read_lock` على `gdev->srcu` وبيجيب `gc` منه. لو الـ chip اتشالت (NULL)، `guard.gc` هيكون NULL والـ caller يرجع `-ENODEV`. ده بيضمن إن الـ chip مش بتتحرر أثناء استخدامها.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — القراءة والتفسير

**الـ** `gpiolib.c` بتسجّل entry واحدة في debugfs جوّا `gpiolib_debugfs_init()`:

```
/sys/kernel/debug/gpio
```

القراءة:

```bash
cat /sys/kernel/debug/gpio
```

مثال على الـ output:

```
gpiochip0: 32 GPIOs, parent: platform/fe200000.gpio, pinctrl-bcm2835, can sleep:
 gpio-5   (                    |reset               ) out hi
 gpio-17  (                    |led0                ) out lo ACTIVE LOW
 gpio-22  (                    |button              ) in  hi IRQ
 gpio-27  (spi0-cs1            )

gpiochip1: 8 GPIOs, parent: i2c/1-0020, mcp23008:
 gpio-32  (                    |mux-sel             ) out lo
```

**تفسير الحقول:**

| الحقل | المعنى |
|---|---|
| `gpio-N` | رقم الـ GPIO في الـ global numberspace |
| `(name\|label)` | اسم الـ line الـ hardware \| اسم الـ consumer |
| `out` / `in` | اتجاه الـ GPIO |
| `hi` / `lo` | القيمة الحالية |
| `IRQ` | الـ GPIO مُستخدم كـ interrupt |
| `ACTIVE LOW` | الـ polarity معكوسة |
| `can sleep` | الـ chip callbacks ممكن تنام (I2C/SPI-based) |

**الـ** `gpiolib_dbg_show()` بتعرض بس الـ GPIOs اللي:
- `GPIOD_FLAG_REQUESTED` مضبوط (GPIO مطلوب من consumer)، أو
- `GPIOD_FLAG_USED_AS_IRQ` مضبوط

الـ GPIOs الـ unrequested اللي عندها اسم بس بتظهر من غير قيمة.

---

#### 2. sysfs — المسارات المهمة

```
/sys/bus/gpio/devices/           # كل GPIO devices المسجّلة
/sys/bus/gpio/devices/gpiochipN/ # device واحد
/sys/bus/gpio/devices/gpiochipN/dev    # major:minor للـ char device
/sys/class/gpio/                 # الـ legacy sysfs interface (CONFIG_GPIO_SYSFS_LEGACY)
/sys/class/gpio/export           # اكتب رقم GPIO لتصدير line
/sys/class/gpio/unexport         # اكتب رقم GPIO لإلغاء التصدير
/sys/class/gpio/gpioN/direction  # "in" أو "out"
/sys/class/gpio/gpioN/value      # 0 أو 1
/sys/class/gpio/gpioN/edge       # "none", "rising", "falling", "both"
/sys/class/gpio/gpioN/active_low # 0 أو 1
```

الـ newer interface (char device):
```
/dev/gpiochipN                   # character device للـ ioctl-based API
```

أوامر جاهزة:

```bash
# اعرض كل gpiochips
ls /sys/bus/gpio/devices/

# اعرف parent device لـ gpiochip0
cat /sys/bus/gpio/devices/gpiochip0/uevent

# صدّر GPIO 17 وشوف قيمته (legacy)
echo 17 > /sys/class/gpio/export
cat /sys/class/gpio/gpio17/direction
cat /sys/class/gpio/gpio17/value

# إلغاء التصدير
echo 17 > /sys/class/gpio/unexport
```

---

#### 3. ftrace — Tracepoints الـ GPIO

**الـ** `gpiolib.c` بيعرّف `CREATE_TRACE_POINTS` ويضم `<trace/events/gpio.h>` اللي فيها tracepoint اتنين بس:

| Tracepoint | بيتفعّل فين |
|---|---|
| `gpio_direction` | `gpiod_direction_input_nonotify()` و `gpiod_direction_output_raw_commit()` |
| `gpio_value` | `gpiod_get_raw_value_commit()` و `gpiod_set_raw_value_commit()` و array functions |

تفعيل:

```bash
# تفعيل كل gpio events
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو event بعينه
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# قرا الـ trace
cat /sys/kernel/debug/tracing/trace
```

مثال على الـ output:

```
     kworker/0:1-12    [000]  1234.567890: gpio_direction: 22 in (0)
     bash-1023         [001]  1234.568012: gpio_value: 22 get 1
     bash-1023         [001]  1234.568100: gpio_value: 17 set 0
```

**تفسير الـ format:**
- `gpio_direction`: `GPIO_NUMBER  in|out  (ERROR_CODE)` — الـ error_code = 0 يعني success
- `gpio_value`: `GPIO_NUMBER  get|set  VALUE` — get = قراءة، set = كتابة

---

#### 4. printk / dynamic debug

**الـ** macros المستخدمة في الكود:

```c
gpiochip_dbg(gc, ...)   /* pr_debug + chip label */
gpiochip_warn(gc, ...)  /* dev_warn */
gpiochip_err(gc, ...)   /* dev_err */
gpiod_dbg(desc, ...)    /* pr_debug + desc info */
gpiod_warn(desc, ...)   /* dev_warn */
gpiod_err(desc, ...)    /* dev_err */
```

تفعيل الـ debug messages الـ dynamic:

```bash
# فعّل كل debug messages في gpiolib
echo "file gpiolib.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل function بعينها
echo "func gpiochip_add_data_with_key +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل كل gpio subsystem
echo "module gpiolib +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق إيه اللي مفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep gpio
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|---|---|
| `CONFIG_DEBUG_FS` | يفعّل `/sys/kernel/debug/gpio` |
| `CONFIG_GPIO_SYSFS` | يفعّل `/sys/class/gpio` interface |
| `CONFIG_GPIOLIB_IRQCHIP` | دعم الـ irqchip داخل gpiolib |
| `CONFIG_GPIO_CDEV` | الـ character device `/dev/gpiochipN` |
| `CONFIG_GPIO_CDEV_V1` | الـ legacy ioctl API |
| `CONFIG_DEBUG_GPIO` | extra GPIO validation checks |
| `CONFIG_LOCKDEP` | يكتشف lock ordering bugs في GPIO/IRQ |
| `CONFIG_PROVE_LOCKING` | WARN_ON عند lockdep violations |
| `CONFIG_GPIOLIB_FASTPATH_LIMIT` | تحديد عدد الـ GPIOs للـ fast path |
| `CONFIG_OF_GPIO` | دعم Device Tree GPIO |
| `CONFIG_ACPI_GPIO` | دعم ACPI GPIO |

تفعيل في `.config`:

```bash
scripts/config --enable CONFIG_DEBUG_FS
scripts/config --enable CONFIG_DEBUG_GPIO
scripts/config --enable CONFIG_GPIO_CDEV
scripts/config --set-val CONFIG_GPIOLIB_FASTPATH_LIMIT 512
```

---

#### 6. أدوات subsystem-specific

**الـ** `gpiotools` (من `tools/gpio/` في kernel source):

```bash
# بناء الأدوات
make -C tools/gpio

# عرض كل gpiochips
./lsgpio

# قراءة قيم GPIO lines
./gpio-watch /dev/gpiochip0 22

# الـ libgpiod user-space library (modern API)
apt-get install gpiod

gpiodetect              # اعرض كل chips
gpioinfo gpiochip0      # اعرض كل lines في chip
gpioget gpiochip0 22    # اقرأ قيمة line 22
gpioset gpiochip0 17=1  # اضبط line 17 على 1
gpiomon gpiochip0 22    # راقب events على line 22
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `GPIOs %d..%d (%s) failed to register, %d` | فشل `gpiochip_add_data_with_key()` | شوف الـ error code المُرجع |
| `GPIO integer space overlap, cannot add chip` | تداخل في الـ GPIO number range | استخدم dynamic allocation (base = -1) |
| `tried to insert a GPIO chip with zero lines` | `gc->ngpio == 0` | اضبط `ngpio` صح أو استخدم `ngpios` property |
| `Static allocation of GPIO base is deprecated` | الـ driver بيحدد base ثابت | حوّل لـ dynamic allocation |
| `Detected name collision for GPIO name '%s'` | اسم GPIO متكرر في نظام | راجع `gpio-line-names` في DT |
| `gpio-line-names too short` | عدد الأسماء في DT أقل من الـ offset | اضبط `gpio-line-names` ليغطي كل الـ pins |
| `tried to flag a GPIO set as output for IRQ` | محاولة استخدام output GPIO كـ IRQ | حوّل الـ GPIO لـ input أولاً |
| `you cannot have chained interrupts on a chip that may sleep` | `can_sleep=true` مع `parent_handler` | استخدم nested threads بدل chained |
| `not an immutable chip, please consider fixing it!` | الـ irqchip مش معلّم بـ `IRQCHIP_IMMUTABLE` | أضف الـ flag للـ irqchip |
| `missing direction_input() operation and line is output` | محاولة إعداد input بس الـ line مضبوطة output | شوف الـ hardware state |
| `cannot find GPIO chip %s, deferring` | الـ chip المطلوب ما اتسجّلش لسّه | طبيعي عند boot، هيتحل بعدين |
| `requesting hog GPIO %s failed` | فشل في حجز GPIO hog | شوف الـ GPIO مش متحجز من حاجة تانية |
| `nonexclusive access to GPIO for %s` | أكتر من consumer بيستخدم نفس الـ GPIO | استخدم `gpio-shared-proxy` mechanism |
| `multiple pull-up, pull-down or pull-disable enabled` | تعارض في الـ lookup flags | راجع الـ DT أو lookup table |
| `can not allocate irq for GPIO line` | فشل تخصيص IRQ في hierarchy domain | شوف parent irqdomain |
| `missing irqdomain vital data` | `child_to_parent_hwirq` أو `fwnode` مش موجود | اكمّل تهيئة الـ `gpio_irq_chip` |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

أماكن مفيدة لإضافة instrumentation مؤقتة:

```c
/* في gpiod_request_commit() — لو GPIO بيتطلب من حتتين */
if (test_and_set_bit(GPIOD_FLAG_REQUESTED, &desc->flags)) {
    dump_stack(); /* من مين بيجي الطلب التاني؟ */
    return -EBUSY;
}

/* في gpiochip_add_data_with_key() — عند فشل التسجيل */
ret = gpiodev_add_to_list_unlocked(gdev);
if (ret) {
    WARN(1, "GPIO overlap: base=%d, ngpio=%d\n", base, gc->ngpio);
    goto err_cleanup_desc_srcu;
}

/* في gpiochip_irq_map() — لو IRQ map بيفشل */
if (!gpiochip_irqchip_irq_valid(gc, hwirq)) {
    WARN_ON(1); /* hwirq خارج الـ valid_mask */
    return -ENXIO;
}

/* في gpiod_direction_output_nonotify() — لو بيحاول يضبط output على GPIO مربوط بـ IRQ مفعّل */
if (test_bit(GPIOD_FLAG_USED_AS_IRQ, &flags) &&
    test_bit(GPIOD_FLAG_IRQ_IS_ENABLED, &flags)) {
    dump_stack(); /* مين بيعمل ده؟ */
    return -EIO;
}

/* في validate_desc() — لو pointer غلط بيوصل */
if (IS_ERR(desc)) {
    WARN(1, "%s: invalid GPIO errorpointer: %pe\n", func, desc);
    return PTR_ERR(desc);
}
```

**نقاط مراقبة SRCU:**

```c
/* تحقق من SRCU lock محتفظ بيه صح */
lockdep_assert_held(&gc->gpiodev->srcu); /* موجود في gpiochip_get_direction() مثلاً */
```

---

### Hardware Level

#### 1. التحقق من تطابق الـ Hardware State مع الـ Kernel State

```bash
# اقرأ state الـ kernel
cat /sys/kernel/debug/gpio | grep "gpio-22"
# مثال: gpio-22  (button) in  hi IRQ

# قارنه بقيمة hardware الحقيقية عبر sysfs
echo 22 > /sys/class/gpio/export
cat /sys/class/gpio/gpio22/value    # لازم تطابق "hi" فوق
cat /sys/class/gpio/gpio22/direction

# أو عبر libgpiod
gpioget gpiochip0 22
```

لو في اختلاف بين kernel state وقراءة hardware:
- الـ `GPIOD_FLAG_IS_OUT` في `desc->flags` ممكن يكون stale
- الـ `gpiod_get_direction()` بيعمل refresh من hardware

---

#### 2. Register Dump — قراءة GPIO Controller Registers مباشرة

للـ memory-mapped GPIO controllers (زي BCM2835 على Raspberry Pi):

```bash
# تركيب devmem2
apt-get install devmem2

# BCM2835 GPIO base address = 0x3F200000
# GPLEV0 register (GPIO Pin Level 0) عند offset 0x34
devmem2 0x3F200034 w    # اقرأ مستوى pins 0-31

# GPFSEL0 register (Function Select) عند offset 0x00
devmem2 0x3F200000 w    # bits 2:0 للـ GPIO0: 000=input, 001=output

# GPPUD (Pull-Up/Down) عند offset 0x94
devmem2 0x3F200094 w
```

عبر `/dev/mem` (يحتاج `CONFIG_DEVMEM=y`):

```bash
# قراءة 4 bytes من register
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 4096, offset=0x3F200000)
    val = struct.unpack('<I', mm[0x34:0x38])[0]
    print(f'GPLEV0 = 0x{val:08X}')
    # bit N = state of GPIO N
    for i in range(32):
        print(f'  GPIO{i} = {(val >> i) & 1}')
"
```

لـ i2c/SPI-based GPIO expanders (زي MCP23017):

```bash
# قراءة GPIOA register (0x12) عبر i2c-tools
i2cget -y 1 0x20 0x12    # GPIOA current value
i2cget -y 1 0x20 0x00    # IODIRA: 0=output, 1=input
i2cget -y 1 0x20 0x0C    # GPPUA: pull-up configuration
```

---

#### 3. Logic Analyzer / Oscilloscope

نقاط مهمة للاستخدام:

**للـ edge detection:**
- ضع probe على الـ GPIO line المشكلة
- فعّل trigger على الـ event المتوقع (rising/falling)
- قارن timing الـ interrupt في kernel مع الـ edge الحقيقية

**للـ open-drain lines:**
- الـ `gpiolib.c` بيحاكي open-drain بواسطة `gpiochip_direction_input()` للـ HIGH و `gpiochip_direction_output(gc, offset, 0)` للـ LOW
- على الـ scope: لو الـ line مابترجعش HIGH بسرعة → pull-up مش موجود أو قيمته عالي جداً

**للـ IRQ lines:**
- تأكد إن الـ edge على الـ scope متوافق مع `IRQ_TYPE_EDGE_RISING/FALLING`
- لو الـ interrupt مابيحصلش: ابحث عن glitch أو bounce بيعمل re-trigger قبل ما الـ handler يخلّص

**للـ SPI/I2C GPIO expanders:**
- راقب الـ communication على الـ bus
- الـ `can_sleep=true` في kernel يعني الـ GPIO access بتحصل في process context، مش irq context

---

#### 4. Hardware Issues الشائعة → Kernel Log Patterns

| المشكلة | Pattern في الـ Kernel Log |
|---|---|
| Pull-up/down مش مضبوط صح | لا يوجد warning مباشر، بس GPIO value بيتغيّر عشوائي |
| GPIO Controller مش detected | `gpiolib: failed to register` أو `probe failed` |
| IRQ لا يتولّد | صمت تام بعد `gpio_direction: N in (0)` |
| IRQ متعلق (stuck) | `irq N: nobody cared` أو `irq N: nobody cared, disabling` |
| Open-drain شغّال غلط | GPIO بيقرأ HIGH بس المفروض LOW → المحاكاة غلطت direction |
| GPIO expander يرجع خطأ I2C | `gpiochip_err` messages متعلقة بـ `request()`/`get()` callbacks |
| Base overlap | `GPIO integer space overlap, cannot add chip` |
| Hotplug race | `gpiod_to_irq: -EPROBE_DEFER` أو `ENODEV` |

---

#### 5. Device Tree Debugging

**تحقق إن الـ DT صح:**

```bash
# شوف الـ DT compiled
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "gpio"

# شوف gpio-ranges
cat /sys/firmware/devicetree/base/soc/gpio@7e200000/gpio-ranges 2>/dev/null | xxd

# تحقق من gpio-line-names
cat /sys/firmware/devicetree/base/soc/gpio@7e200000/gpio-line-names 2>/dev/null

# شوف الـ fwnode المرتبط بـ gpiochip
ls -la /sys/bus/gpio/devices/gpiochip0/of_node
```

**الـ gpiolib.c warnings المتعلقة بالـ DT:**

```c
/* لو gpio-line-names أقصر من المطلوب */
dev_warn(dev, "gpio-line-names too short (length %d), cannot map names for the gpiochip at offset %u\n", count, chip->offset);

/* لو اسم متكرر */
dev_warn(&gdev->dev, "Detected name collision for GPIO name '%s'\n", gc->names[i]);

/* لو DT مع default_type != IRQ_TYPE_NONE */
WARN(fwnode && type != IRQ_TYPE_NONE, "%pfw: Ignoring %u default trigger\n", fwnode, type);
```

**مطابقة الـ DT مع الـ hardware:**

```bash
# طباعة كامل DT node لـ GPIO controller
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
  awk '/gpio@7e200000/{p=1} p{print} /^};/{if(p)exit}'

# تحقق gpio-reserved-ranges
# الـ gpiolib بتطبّقها في gpiochip_apply_reserved_ranges()
# شوف valid_mask عبر:
cat /sys/kernel/debug/gpio | grep "gpiochip0"
# الـ GPIOs المحجوزة مش هتظهر في debug output
```

---

### Practical Commands

#### أوامر جاهزة للـ Copy

**فحص كامل لحالة GPIO subsystem:**

```bash
#!/bin/bash
echo "=== GPIO Debug Report ==="
echo ""
echo "--- debugfs ---"
cat /sys/kernel/debug/gpio
echo ""
echo "--- gpiochips ---"
ls /sys/bus/gpio/devices/
echo ""
echo "--- char devices ---"
ls -la /dev/gpiochip*
echo ""
echo "--- sysfs exported ---"
ls /sys/class/gpio/ 2>/dev/null
```

**تفعيل كامل لـ ftrace GPIO:**

```bash
# إعادة ضبط الـ trace buffer
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace

# فعّل gpio events
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# فعّل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل العملية اللي عايز تتعب عليها...
sleep 5

# أوقف وشوف النتيجة
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep "gpio_"
```

**مراقبة GPIO line events بـ libgpiod:**

```bash
# راقب كل events على GPIO 22 في gpiochip0
gpiomon --num-events=10 --rising-edge --falling-edge gpiochip0 22

# مثال output:
# RISING EDGE event on line 22 at 1708734123.456789
# FALLING EDGE event on line 22 at 1708734124.123456
```

**تفعيل dynamic debug لـ gpiolib:**

```bash
# فعّل كل debug في gpiolib
echo "file drivers/gpio/gpiolib.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# فعّل بس function معينة
echo "func gpiod_find_and_request +p" > /sys/kernel/debug/dynamic_debug/control

# شوف في dmesg
dmesg -w | grep -i gpio

# تعطيل
echo "file drivers/gpio/gpiolib.c -p" > /sys/kernel/debug/dynamic_debug/control
```

**فحص GPIO request conflicts:**

```bash
# شوف مين حاجز GPIO ده
cat /sys/kernel/debug/gpio | grep "gpio-17"
# gpio-17  (                    |button-irq          ) in  hi IRQ

# أو عبر libgpiod
gpioinfo gpiochip0 | grep "line 17"
# line 17: "button-irq" input active-high [used]
```

**قراءة register hardware مباشرة (BCM2835 example):**

```bash
# تأكد من تحميل /dev/mem
ls -la /dev/mem

# قراءة GPLEV0 (GPIO 0-31 levels) للـ BCM2835
devmem2 0x3F200034 w

# قراءة GPFSEL1 (GPIO 10-19 function select)
devmem2 0x3F200004 w

# تفسير GPFSEL: كل 3 bits لـ GPIO واحد:
# 000 = Input
# 001 = Output
# 100 = Alternate function 0
# الخ
```

**فحص IRQ mapping لـ GPIO:**

```bash
# اعرف IRQ رقم GPIO N (بعد طلبه)
GPIO_NUM=22
GPIO_IRQ=$(cat /sys/kernel/debug/gpio | awk "/gpio-$GPIO_NUM.*IRQ/{print NR}")

# أو عبر proc
cat /proc/interrupts | grep "gpio"

# فحص irqdomain
ls /sys/kernel/debug/irq/domains/ | grep gpio
cat /sys/kernel/debug/irq/domains/gpio*/   # لو موجود
```

**اختبار GPIO set/get عبر sysfs (legacy):**

```bash
GPIO=17
# تصدير
echo $GPIO > /sys/class/gpio/export

# اضبط كـ output وارسل high
echo out > /sys/class/gpio/gpio${GPIO}/direction
echo 1 > /sys/class/gpio/gpio${GPIO}/value

# تحقق
cat /sys/kernel/debug/gpio | grep "gpio-${GPIO}"
# gpio-17  (|gpio_sysfs) out hi

# إلغاء التصدير
echo $GPIO > /sys/class/gpio/unexport
```

**فحص pinctrl-GPIO ranges:**

```bash
# اعرف الـ pin ranges المرتبطة بـ gpiochip
cat /sys/kernel/debug/pinctrl/pinctrl-handles

# أو
ls /sys/kernel/debug/pinctrl/
cat /sys/kernel/debug/pinctrl/*/gpio-ranges 2>/dev/null
```

**مثال output كامل لـ `/sys/kernel/debug/gpio` على Raspberry Pi 4:**

```
gpiochip0: 58 GPIOs, parent: platform/fe200000.gpio, pinctrl-bcm2711, can sleep:
 gpio-5   (                    |reset               ) out lo
 gpio-17  (                    |led0                ) out hi ACTIVE LOW
 gpio-22  (spi0-cs0            |spi0 CS0            ) out hi
 gpio-27  (                    |button              ) in  hi IRQ

gpiochip1: 8 GPIOs, parent: i2c/1-0020, mcp23008:
 gpio-512 (                    |relay0              ) out lo
 gpio-513 (                    |relay1              ) out lo
```

**تفسير الـ output:**
- `gpiochip0: 58 GPIOs` → الـ chip عنده 58 line
- `parent: platform/fe200000.gpio` → مرتبط بـ platform device عند الـ address ده
- `can sleep` → الـ callbacks ممكن تنام (I2C/SPI based أو slow)
- `gpio-512` → dynamic allocation بدأ من 512 (راجع `GPIO_DYNAMIC_BASE`)
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: GPIO interrupt مش بيشتغل على RK3562 Industrial Gateway

#### العنوان
GPIO-based interrupt على I2C touch controller في industrial gateway بـ RK3562 مش بيـfire

#### السياق
شركة بتعمل industrial HMI gateway بـ RK3562، الـ touchscreen controller (GT911) متوصل بـ I2C ومحتاج GPIO interrupt. الـ board بيبوت بس اللمس مش شغال خالص، الـ driver بيلاقي الـ GPIO بس الـ IRQ مش بيـtrigger.

#### المشكلة
الـ driver بيعمل `gpiod_to_irq()` على الـ GPIO line وبيرجع `-EPROBE_DEFER` بدل IRQ number صح. الـ touch driver بيتـdefer للأبد.

#### التحليل

في `gpiolib.c` السطر 4024:
```c
int gpiod_to_irq(const struct gpio_desc *desc)
{
    ...
    guard(srcu)(&gdev->srcu);
    gc = srcu_dereference(gdev->chip, &gdev->srcu);
    ...
    if (gc->to_irq) {
        ret = gc->to_irq(gc, offset);
        if (ret)
            return ret;
        return -ENXIO;  /* Zero means NO_IRQ */
    }
#ifdef CONFIG_GPIOLIB_IRQCHIP
    if (gc->irq.chip) {
        return -EPROBE_DEFER;  /* <-- هنا المشكلة */
    }
#endif
    return -ENXIO;
}
```

المسار اللي بيحصل:
1. الـ gpiochip (RK3562 GPIO controller) عنده `gc->irq.chip` مضبوطة.
2. بس الـ `gc->irq.initialized` لسه `false` — الـ irqchip لسه بيتسجل.
3. في `gpiochip_to_irq()` السطر 1919:
```c
static int gpiochip_to_irq(struct gpio_chip *gc, unsigned int offset)
{
    ...
    if (!gc->irq.initialized)
        return -EPROBE_DEFER;  /* الـ chip لسه ما اتسجلش */
    ...
}
```
4. السبب الحقيقي: الـ touch driver بيـprobe قبل ما `gpiochip_add_irqchip()` يخلص، والـ `gc->irq.initialized` بيتضبط في `gpiochip_irqchip_add_allocated_domain()` السطر 2072 بس بعد ما `barrier()` يتعمل.

**Timeline المشكلة:**
```
[T0] rk3562-gpio driver: gpiochip_add_data_with_key() يبدأ
[T1] gt911-touch driver يبدأ probe (race condition)
[T2] gt911: gpiod_to_irq() → gpiochip_to_irq() → -EPROBE_DEFER
[T3] gpiochip_irqchip_add_allocated_domain(): gc->irq.initialized = true
[T4] gt911: لا حد بيـretrigger probe تاني
```

#### الحل

**1. تأكد إن الـ DT binding صح:**
```dts
/* في DT الـ RK3562 */
&i2c4 {
    gt911: touchscreen@5d {
        compatible = "goodix,gt911";
        reg = <0x5d>;
        interrupt-parent = <&gpio3>;
        interrupts = <5 IRQ_TYPE_EDGE_FALLING>;
        irq-gpios = <&gpio3 5 GPIO_ACTIVE_LOW>;
        reset-gpios = <&gpio2 14 GPIO_ACTIVE_LOW>;
    };
};
```

**2. تحقق من الـ kernel log:**
```bash
dmesg | grep -E "gpio|gt911|i2c4|defer"
# لازم تشوف:
# "gt911 3-005d: supply ... not found, using dummy regulator"
# وبعدين retry بعد GPIO init
```

**3. لو المشكلة race condition حقيقية:**
```bash
# راجع إن CONFIG_GPIOLIB_IRQCHIP=y
grep CONFIG_GPIOLIB_IRQCHIP /boot/config-$(uname -r)

# شوف الـ IRQ domain اتعمل
cat /proc/interrupts | grep rk3562-gpio
ls /sys/kernel/irq/*/chip_name
```

**4. الحل الأضمن: تأكد من `probe ordering` في DT:**
```dts
/* أضف deferred-probe dependency صريح */
gt911 {
    ...
    status = "okay";
    /* fw_devlink هيتعامل مع الـ ordering تلقائي */
};
```

#### الدرس المستفاد
**الـ `gpiod_to_irq()` تقدر ترجع `-EPROBE_DEFER` حتى لو الـ GPIO chip موجود**، لأن الـ IRQ domain بيتضبط في مرحلة منفصلة داخل `gpiochip_add_irqchip()`. الـ consumer driver لازم يتعامل مع الـ `-EPROBE_DEFER` صح ويرجع بيه للأعلى، ومش يعمل fallback لـ polling خطأ.

---

### السيناريو 2: GPIO hog بيـcause kernel panic على STM32MP1 Industrial Board

#### العنوان
GPIO hog في DT على STM32MP1 بيعمل panic عند boot لأن chip offset غلط

#### السياق
Engineer بيعمل custom carrier board للـ STM32MP1 (مثلاً Avenger96). عنده power-enable GPIO محتاج يتضبط output-high من أول ما الـ GPIO chip يتسجل (قبل أي driver). استخدم `gpio-hog` في DT، بس الـ board بيـhang على boot.

#### المشكلة
الـ kernel panic بيحصل في `gpiod_hog()` مع null pointer dereference. الـ DT بيحدد GPIO offset أكبر من `gc->ngpio`.

#### التحليل

الـ flow من `gpiochip_add_data_with_key()` السطر 1200:
```c
machine_gpiochip_add(gc);      /* machine hogs */
/* ... */
ret = of_gpiochip_add(gc);     /* DT hogs */
```

**في `of_gpiochip_add()` الـ DT hogs بيتعملوا، بس الكود في `gpiochip_machine_hog()` السطر 928:**
```c
static void gpiochip_machine_hog(struct gpio_chip *gc, struct gpiod_hog *hog)
{
    struct gpio_desc *desc;

    desc = gpiochip_get_desc(gc, hog->chip_hwnum);
    if (IS_ERR(desc)) {
        gpiochip_err(gc, "%s: unable to get GPIO desc: %ld\n",
                     __func__, PTR_ERR(desc));
        return;  /* يرجع بدون panic — هنا الكود سليم */
    }

    rv = gpiod_hog(desc, hog->line_name, hog->lflags, hog->dflags);
    ...
}
```

المشكلة في `gpio_device_get_desc()` السطر 213:
```c
struct gpio_desc *
gpio_device_get_desc(struct gpio_device *gdev, unsigned int hwnum)
{
    if (hwnum >= gdev->ngpio)
        return ERR_PTR(-EINVAL);
    return &gdev->descs[array_index_nospec(hwnum, gdev->ngpio)];
}
```

**السبب الحقيقي:** الـ STM32MP1 بيستخدم multiple GPIO banks (GPIOA=0-15, GPIOB=16-31, ...). الـ DT node كان:
```dts
/* خطأ: اعتقد إن GPIOA pin 10 هو hwnum=10 */
/* بس الـ chip اللي بيمثل GPIOA بس ngpio=16 */
gpioa: gpio@50002000 {
    gpio-controller;
    #gpio-cells = <2>;
    ngpios = <16>;
    gpio-ranges = <&pinctrl 0 0 16>;

    /* hog غلط: chip_hwnum=42 > ngpio=16 */
    power-en-hog {
        gpio-hog;
        gpios = <42 0>;  /* خطأ */
        output-high;
        line-name = "power-enable";
    };
};
```

#### الحل

**1. تصحيح DT:**
```dts
gpiob: gpio@50003000 {
    gpio-controller;
    #gpio-cells = <2>;
    ngpios = <16>;

    power-en-hog {
        gpio-hog;
        gpios = <10 0>;   /* GPIO B pin 10 = hwnum صح داخل الـ GPIOB chip */
        output-high;
        line-name = "power-enable";
    };
};
```

**2. تحقق من الـ ngpio:**
```bash
# شوف ngpio لكل chip
for chip in /sys/bus/gpio/devices/gpiochip*; do
    echo "$chip: $(cat $chip/ngpio)"
done

# شوف الـ hog اتطبق
gpioinfo | grep "power-enable"
```

**3. لو محتاج debug قبل panic:**
```bash
# أضف early_printk في kernel command line
earlycon=stm32,0x40010000 loglevel=8

# الـ kernel log هيظهر:
# "gpio-N: unable to get GPIO desc: -22" (EINVAL)
```

**4. validate الـ hog بعد boot:**
```bash
gpioinfo gpiochip1 | grep -A2 "power-enable"
# المتوقع:
# line 10: "power-enable" unused output active-high [kernel]
```

#### الدرس المستفاد
الـ `gpio-hog` في DT لازم يكون `hwnum` **نسبي للـ chip اللي جواه**، مش absolute GPIO number. الـ `gpio_device_get_desc()` بيـvalidate ده وبيرجع `-EINVAL` بس الـ caller لازم يتعامل معاه صح. دايماً تحقق من `ngpio` لكل chip قبل ما تحدد offsets.

---

### السيناريو 3: GPIO valid_mask بيمنع استخدام pin على Allwinner H616 Android TV Box

#### العنوان
GPIO pin على Allwinner H616 بيرجع `-EINVAL` عند request رغم إنه موجود في DT

#### السياق
Engineer بيعمل Android TV box custom firmware على Allwinner H616 (مثلاً Orange Pi Zero 2). بيحاول يتحكم في IR LED عبر GPIO driver بس بيلاقي `-EINVAL` عند كل محاولة request.

#### المشكلة
الـ GPIO موجود في `/sys/kernel/debug/gpio` بس أي محاولة request بيرجع `-EINVAL`. الـ `gpioinfo` بيظهر الـ line بس بـ status غريب.

#### التحليل

في `gpiod_request_commit()` السطر 2456:
```c
int gpiod_request_commit(struct gpio_desc *desc, const char *label)
{
    ...
    offset = gpiod_hwgpio(desc);
    if (!gpiochip_line_is_valid(guard.gc, offset))
        return -EINVAL;  /* <-- هنا بيتوقف */
    ...
}
```

الـ `gpiochip_line_is_valid()` السطر 809:
```c
bool gpiochip_line_is_valid(const struct gpio_chip *gc, unsigned int offset)
{
    if (!gc->gpiodev)
        return true;
    if (likely(!gc->gpiodev->valid_mask))
        return true;
    return test_bit(offset, gc->gpiodev->valid_mask);  /* الـ bit مش مضبوط */
}
```

الـ `valid_mask` اتعمل في `gpiochip_init_valid_mask()` السطر 749:
```c
static int gpiochip_init_valid_mask(struct gpio_chip *gc)
{
    int ret;

    if (!(gpiochip_count_reserved_ranges(gc) || gc->init_valid_mask))
        return 0;

    gc->gpiodev->valid_mask = gpiochip_allocate_mask(gc);
    /* Assume by default all GPIOs are valid */
    bitmap_fill(p, gc->ngpio);  /* كل الـ bits = 1 */

    ret = gpiochip_apply_reserved_ranges(gc);  /* بيـclear الـ reserved */
    if (gc->init_valid_mask)
        return gc->init_valid_mask(gc, gc->gpiodev->valid_mask, gc->ngpio);
    ...
}
```

**الـ root cause على H616:**
الـ H616 GPIO driver بيعمل `init_valid_mask` callback بيـclear الـ pins اللي بتتحكم فيها الـ analog/USB circuitry. الـ IR LED pin (مثلاً `PL10`) كان جوا reserved range أو الـ `init_valid_mask` بيتعامل معاه كـ dedicated pin.

```bash
# تحقق من الـ reserved-ranges في DTS
dtc -I fs /sys/firmware/devicetree/base | grep -A5 "gpio-reserved-ranges"
# أو
hexdump -C /sys/firmware/fdt | grep -A2 "reserved"
```

#### الحل

**1. تحقق من الـ valid_mask عبر debugfs:**
```bash
cat /sys/kernel/debug/gpio | grep -A30 "gpiochip.*h616\|gpioPL"
# لو الـ line مش ظاهرة = masked out
```

**2. قرأ الـ DTS الأصلي:**
```bash
dtc -I dtb -O dts /boot/sun50i-h616.dtb 2>/dev/null | grep -A10 "gpio-reserved-ranges"
```

**3. لو الـ pin فعلاً في reserved range، أزله من DTS:**
```dts
/* في sun50i-h616.dtsi أو الـ board DTS */
&pio {
    /* إزالة الـ range اللي بتشمل PL10 */
    /* gpio-reserved-ranges = <96 2>; */  /* قبل */
    gpio-reserved-ranges = <96 1>;        /* بعد: بس PL8 و PL9 */
};
```

**4. أو لو الـ driver نفسه مش صح في `init_valid_mask`:**
```bash
# شوف الـ driver source
find /usr/src -name "sun50i*.c" | xargs grep -l "init_valid_mask"
# وراجع إيه الـ pins اللي بيتعمل mask عليها
```

**5. validation بعد الإصلاح:**
```bash
gpioinfo | grep "PL10\|ir-led"
# المتوقع: إن الـ line تظهر وتكون available
gpio-hammer -n gpiochip3 -o 10  # test بسيط
```

#### الدرس المستفاد
الـ `valid_mask` في `gpio_device` هو **silent guard** — الـ pin موجود في الـ descriptor array بس مش valid للاستخدام. الـ `-EINVAL` من `gpiod_request_commit()` مش معناه إن الـ pin رقمه غلط، ممكن يكون معناه إنه محجوز. دايماً فحص `gpio-reserved-ranges` في DTS وفحص `init_valid_mask` callback في الـ driver.

---

### السيناريو 4: GPIO name collision بيسبب wrong device على AM62x IoT Sensor Hub

#### العنوان
GPIO lookup بيرجع descriptor غلط على AM62x بسبب اسم مكرر بين GPIO controllers

#### السياق
IoT sensor hub بـ AM62x (Texas Instruments) فيه multiple GPIO controllers. الـ board designer سمى GPIO line باسم `"sensor-power"` في اتنين GPIO chips مختلفين (GPIODEV0 و GPIODEV2). الـ sensor driver بيعمل lookup بالاسم وبييجي مع الـ chip الغلط.

#### المشكلة
الـ sensor بيشتغل أحياناً وأحياناً لأ حسب الـ boot order. الـ power GPIO بييجي من chip غلطان.

#### التحليل

الـ `gpio_name_to_desc()` السطر 546:
```c
static struct gpio_desc *gpio_name_to_desc(const char * const name)
{
    struct gpio_device *gdev;
    struct gpio_desc *desc;
    struct gpio_chip *gc;

    if (!name)
        return NULL;

    guard(srcu)(&gpio_devices_srcu);

    list_for_each_entry_srcu(gdev, &gpio_devices, list, ...) {
        guard(srcu)(&gdev->srcu);
        gc = srcu_dereference(gdev->chip, &gdev->srcu);
        if (!gc)
            continue;

        for_each_gpio_desc(gc, desc) {
            if (desc->name && !strcmp(desc->name, name))
                return desc;  /* أول match يرجع — حتى لو في اتنين */
        }
    }
    return NULL;
}
```

وفي `gpiochip_set_desc_names()` السطر 582:
```c
static void gpiochip_set_desc_names(struct gpio_chip *gc)
{
    /* First check all names if they are unique */
    for (i = 0; i != gc->ngpio; ++i) {
        gpio = gpio_name_to_desc(gc->names[i]);
        if (gpio)
            dev_warn(&gdev->dev,
                     "Detected name collision for GPIO name '%s'\n",
                     gc->names[i]);  /* warning بس مش error */
    }
    /* ثم بيضيف الأسماء */
    for (i = 0; i != gc->ngpio; ++i)
        gdev->descs[i].name = gc->names[i];
}
```

**المشكلة:** الـ kernel يطبع warning بس ما يمنعش التسجيل. الـ `gpio_name_to_desc` بترجع أول واحد في الـ list حسب الـ boot order.

**الـ GPIO devices list** بيتضبط في `gpiodev_add_to_list_unlocked()` السطر 496 مرتب بـ base number. الـ AM62x:
- GPIODEV0: base=0, ngpio=92
- GPIODEV2: base=200, ngpio=52

لو الـ "sensor-power" في GPIODEV0 pin 15 وكمان في GPIODEV2 pin 3، الـ lookup دايماً هيرجع GPIODEV0 pin 15 لأنه أول في الـ list. لو غلط هو اللي محتاج التحكم، الـ sensor هيتضرر.

#### الحل

**1. اكتشف الـ collision من الـ dmesg:**
```bash
dmesg | grep "name collision"
# [    2.451] gpio gpiochip2: Detected name collision for GPIO name 'sensor-power'
```

**2. أصلح الأسماء في DTS لتكون unique:**
```dts
/* GPIODEV0 في am62x-starter-kit.dts */
&main_gpio0 {
    gpio-line-names =
        "PM_INT", "", "", "", "", "", "", "",   /* 0-7 */
        "", "", "", "", "", "", "", "",          /* 8-15 */
        "sensor-power-primary", ...;            /* كان "sensor-power" */
};

/* GPIODEV2 */
&main_gpio2 {
    gpio-line-names =
        "", "", "", "sensor-power-secondary", ...; /* كان "sensor-power" */
};
```

**3. الـ consumer driver يستخدم DT lookup بدل name lookup:**
```c
/* بدل: */
desc = gpio_name_to_desc("sensor-power");  /* deprecated & unsafe */

/* استخدم: */
desc = gpiod_get(&pdev->dev, "sensor-power", GPIOD_OUT_LOW);
/* مع DT binding صح */
```

**4. تحقق من uniqueness:**
```bash
gpioinfo | grep "sensor-power"
# المتوقع: اسم واحد بس
```

**5. تحقق من الـ lookup يرجع الصح:**
```bash
gpiofind "sensor-power-primary"
# يطبع: gpiochip0 15
```

#### الدرس المستفاد
الـ `gpio_name_to_desc()` بترجع **أول match** بدون guarantee على أي chip — كما هو documented في الكود ("Note that there is no guarantee that GPIO names are globally unique!"). الـ production code لازم يعتمد على **DT/ACPI descriptor-based lookup** عبر `gpiod_get()` مش على name-based lookup. الـ name collision warning في dmesg لازم يتعامل معاه بجدية.

---

### السيناريو 5: Open-drain GPIO بيتصرف غلط على i.MX8 Automotive ECU بسبب ACTIVE_LOW + OPEN_DRAIN تعارض

#### العنوان
I2C bus reset GPIO على i.MX8MP في automotive ECU بيعمل reverse logic بسبب تعارض active-low مع open-drain emulation

#### السياق
Automotive ECU بـ i.MX8MP بيستخدم GPIO لـ reset I2C devices (CAN transceiver + EEPROM). الـ reset line بالـ hardware مصمم active-low open-drain. الـ engineer حدد في DT `GPIO_ACTIVE_LOW | GPIO_OPEN_DRAIN`. بس الـ bus reset بيحصل في الوقت الغلط — الـ devices بتتـreset لما المفروض يكونوا شغالين والعكس.

#### المشكلة
الـ I2C devices بتتـreset عشوائياً أو بالـ reverse timing من المتوقع.

#### التحليل

الـ flow في `gpiod_direction_output_nonotify()` السطر 3030:
```c
int gpiod_direction_output_nonotify(struct gpio_desc *desc, int value)
{
    unsigned long flags;
    int ret;

    flags = READ_ONCE(desc->flags);

    if (test_bit(GPIOD_FLAG_ACTIVE_LOW, &flags))
        value = !value;  /* step 1: invert بسبب ACTIVE_LOW */
    else
        value = !!value;

    /* ... IRQ check ... */

    if (test_bit(GPIOD_FLAG_OPEN_DRAIN, &flags)) {
        /* First see if we can enable open drain in hardware */
        ret = gpio_set_config(desc, PIN_CONFIG_DRIVE_OPEN_DRAIN);
        if (!ret)
            goto set_output_value;  /* لو hardware open-drain: يكمل بالقيمة المعكوسة */
        /* Emulate open drain by not actively driving the line high */
        if (value)
            goto set_output_flag;  /* step 2: لو value=1 بعد inversion: input mode */
    }
    ...
set_output_value:
    ret = gpio_set_bias(desc);
    if (ret)
        return ret;
    return gpiod_direction_output_raw_commit(desc, value);
```

**التفاصيل الحرجة:**

على i.MX8MP، الـ GPIO controller بيدعم hardware open-drain. فـ `gpio_set_config(desc, PIN_CONFIG_DRIVE_OPEN_DRAIN)` بينجح وبيـgoto `set_output_value`.

المشكلة في **التسلسل:**
1. Driver بيطلب `gpiod_direction_output(desc, 0)` = "assert reset" (logical low)
2. بسبب `ACTIVE_LOW`: value تتحول لـ `!0 = 1`
3. Hardware open-drain بينجح → بيكتب raw value = 1 → الـ pin يروح HIGH
4. Open-drain مع physical pull-down = الـ line بتطلع HIGH = reset مش بيحصل

وهكذا:
- Driver يقول "assert reset" → logical 0 → raw 1 → line HIGH → NO RESET
- Driver يقول "deassert reset" → logical 1 → raw 0 → line LOW → RESET يحصل

كل حاجة معكوسة.

**في `gpiod_set_value_nocheck()` السطر 3868:**
```c
static int gpiod_set_value_nocheck(struct gpio_desc *desc, int value)
{
    if (test_bit(GPIOD_FLAG_ACTIVE_LOW, &desc->flags))
        value = !value;  /* invert */

    if (test_bit(GPIOD_FLAG_OPEN_DRAIN, &desc->flags))
        return gpio_set_open_drain_value_commit(desc, value);
    ...
}
```

وفي `gpio_set_open_drain_value_commit()` السطر 3596:
```c
static int gpio_set_open_drain_value_commit(struct gpio_desc *desc, bool value)
{
    ...
    if (value) {
        ret = gpiochip_direction_input(guard.gc, offset);  /* HIGH = float = input */
    } else {
        ret = gpiochip_direction_output(guard.gc, offset, 0); /* LOW = drive low */
        ...
    }
    ...
}
```

هنا ده الـ software open-drain emulation. بس لو الـ hardware دعم open-drain كما في i.MX8MP، الـ path بيختلف ومفيش inversion للـ direction logic.

#### الحل

**الحل 1: تعديل DT ليكون semantically صح:**

لو الـ hardware pull هو pull-up (الغالب في open-drain reset circuits):
```dts
/* i.MX8MP custom ECU board */
&gpio4 {
    /* reset line: active-low, open-drain مع external pull-up */
};

i2c-bus-reset-gpio {
    compatible = "gpio-reset";
    reset-gpios = <&gpio4 20 (GPIO_ACTIVE_LOW | GPIO_OPEN_DRAIN)>;
    /* driver يستخدم gpiod_set_value(desc, 1) = deassert */
    /* gpiod_set_value(desc, 0) = assert reset */
};
```

**الحل 2: تحقق من hardware behavior:**
```bash
# بعد boot، راقب الـ line بـ oscilloscope أو:
gpioget gpiochip3 20
# لو الـ line HIGH وانت تتوقع reset = مفيش pull-down أو الـ polarity غلط

# جرب بيديك:
gpioset --mode=time --sec=1 gpiochip3 20=0
# شوف إيه اللي بيحصل على الـ I2C bus
```

**الحل 3: لو الـ iMX8 GPIO hardware open-drain بيسبب confusion:**
```dts
/* أزل GPIO_OPEN_DRAIN وخلي الـ software emulation يشتغل */
reset-gpios = <&gpio4 20 GPIO_ACTIVE_LOW>;
/* الـ driver يـconfigure open-drain بنفسه */
```

**الحل 4: Debug عبر debugfs:**
```bash
cat /sys/kernel/debug/gpio | grep -A2 "gpiochip3"
# شوف الـ direction والـ value الحالية
# وتابع الـ trace_gpio_direction trace
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
cat /sys/kernel/debug/tracing/trace
```

#### الدرس المستفاد
**`ACTIVE_LOW` و `OPEN_DRAIN` بيتشابكوا** بشكل غير بديهي في `gpiolib.c`. الـ `ACTIVE_LOW` inversion بيحصل **قبل** اتخاذ قرار الـ open-drain direction. لما الـ hardware بيدعم open-drain نيتيف (مثل i.MX8MP)، الـ path بيمر بـ `gpio_set_config` والـ raw value المعكوسة بتروح للـ hardware مباشرة. دايماً اختبر بـ oscilloscope أو logic analyzer بعد ضبط الـ GPIO flags، ومتفترضش إن "logical behavior = physical behavior" خصوصاً مع combination من الـ flags.
## Phase 7: مصادر ومراجع

### مصادر رسمية — توثيق الـ Kernel

| المصدر | الرابط |
|--------|--------|
| GPIO index — kernel docs | [docs.kernel.org/driver-api/gpio/index.html](https://docs.kernel.org/driver-api/gpio/index.html) |
| GPIO Driver Interface | [docs.kernel.org/driver-api/gpio/driver.html](https://docs.kernel.org/driver-api/gpio/driver.html) |
| GPIO Consumer Interface (descriptor-based) | [docs.kernel.org/driver-api/gpio/consumer.html](https://docs.kernel.org/driver-api/gpio/consumer.html) |
| GPIO Board Mappings | [docs.kernel.org/driver-api/gpio/board.html](https://docs.kernel.org/driver-api/gpio/board.html) |
| Legacy GPIO Interfaces | [static.lwn.net/kerneldoc/driver-api/gpio/legacy.html](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html) |

**المسارات داخل الـ kernel source tree:**

```
Documentation/driver-api/gpio/
├── index.rst          # نقطة الدخول الرئيسية
├── driver.rst         # واجهة كتابة الـ GPIO driver
├── consumer.rst       # واجهة استخدام الـ GPIO من الـ driver
├── board.rst          # ربط الـ GPIO بالأجهزة (DT, ACPI, platform)
├── legacy.rst         # الـ API القديم (gpio_get_value, etc.)
└── using-gpio.rst     # دليل استخدام عام
```

---

### مقالات LWN.net

دي أهم المقالات اللي غطّت تطور الـ gpiolib من أول ما اتضاف للـ kernel لحد الـ descriptor-based API:

| العنوان | الرابط | الأهمية |
|---------|--------|---------|
| GPIO in the kernel: an introduction | [lwn.net/Articles/532714](https://lwn.net/Articles/532714/) | مقدمة شاملة للـ GPIO API في الـ kernel |
| GPIO in the kernel: future directions | [lwn.net/Articles/533632](https://lwn.net/Articles/533632/) | مناقشة التحول من الأرقام للـ descriptors |
| GPIO implementation framework | [lwn.net/Articles/256461](https://lwn.net/Articles/256461/) | أول توثيق للـ gpiolib framework |
| Documentation: gpiolib: document new interface | [lwn.net/Articles/574055](https://lwn.net/Articles/574055/) | توثيق الـ descriptor-based interface الجديد |
| gpiolib: Add GPIO name support | [lwn.net/Articles/651500](https://lwn.net/Articles/651500/) | إضافة دعم الأسماء للـ GPIO lines |
| gpiolib: in-kernel documentation updates | [lwn.net/Articles/966723](https://lwn.net/Articles/966723/) | تحديثات توثيق حديثة تعكس الحالة الراهنة |

**ملاحظة:** مقالَي Jonathan Corbet ([532714](https://lwn.net/Articles/532714/) و[533632](https://lwn.net/Articles/533632/)) هما الأساس — اقراهم الأول.

---

### نقاشات الـ Mailing List

- **[RFC] gpiolib: introduce descriptor-based GPIO interface**
  نقاش الـ ARM kernel list اللي بدأت فيه فكرة استبدال الأرقام بالـ opaque descriptors:
  [linux-arm-kernel.infradead.narkive.com/...](https://linux-arm-kernel.infradead.narkive.com/iCoktLXa/rfc-gpiolib-introduce-descriptor-based-gpio-interface)

- **[GIT PULL] gpio: updates for v6.9-rc1** — تحديثات GPIO لـ v6.9 بما فيها إعادة هيكلة الـ locking باستخدام SRCU:
  [lists.openwall.net/linux-kernel/2024/03/11/314](https://lists.openwall.net/linux-kernel/2024/03/11/314)

- **Re: gpiolib: cdev: Fix use after free in lineinfo_changed_notify** — نقاش bug حقيقي في الـ cdev layer:
  [lore.kernel.org/linux-kernel/20240502015122.GA15967@rigel](https://lore.kernel.org/linux-kernel/20240502015122.GA15967@rigel/)

- **البحث في lore.kernel.org:**
  ```
  https://lore.kernel.org/linux-gpio/
  ```
  ده الـ mailing list الرسمي لكل patches وnقاشات الـ GPIO subsystem.

---

### كورد الـ Source Code على GitHub

- **gpiolib.c الحالية على master:**
  [github.com/torvalds/linux/blob/master/drivers/gpio/gpiolib.c](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpiolib.c)

- **تاريخ commits الـ GPIO subsystem:**
  ```bash
  git log --oneline drivers/gpio/gpiolib.c
  ```

- **أهم commits تاريخية:**

  | التغيير | الوصف |
  |---------|-------|
  | إدخال `gpio_desc` | استبدال الأرقام العددية بـ opaque descriptors |
  | إضافة `SRCU` locking | حماية قائمة الـ GPIO devices من الـ concurrent access |
  | إضافة character device interface | `/dev/gpiochipN` بديل الـ sysfs القديم |
  | deprecation الـ sysfs interface | `/sys/class/gpio` صار deprecated رسمياً |

---

### مصادر elinux.org

- **Pin Control and GPIO Update (PDF):**
  [elinux.org/images/c/cd/Pincontrol-gpio-update.pdf](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf)
  عرض تقديمي عن العلاقة بين الـ pinctrl subsystem والـ gpiolib.

- **Leapster Explorer: GPIO subsystem:**
  [elinux.org/Leapster_Explorer:_GPIO_subsystem](https://elinux.org/Leapster_Explorer:_GPIO_subsystem)
  مثال عملي على تطبيق GPIO subsystem في hardware حقيقي.

---

### مصادر kernelnewbies.org

- **Linux 6.2 — GPIO latch driver:**
  [kernelnewbies.org/Linux_6.2](https://kernelnewbies.org/Linux_6.2)

- **Linux 6.6 — GPIO changes:**
  [kernelnewbies.org/Linux_6.6](https://kernelnewbies.org/Linux_6.6)

- **Linux 6.11 — GPIO PWM driver:**
  [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11)

- **Linux 6.13 — Aspeed G7 GPIO:**
  [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13)

الصفحات دي بتوضح إزاي الـ GPIO subsystem بيتطور مع كل kernel release.

---

### مقالات خارجية مفيدة

- **Stop using /sys/class/gpio — it's deprecated** (The Good Penguin):
  [thegoodpenguin.co.uk/blog/stop-using-sys-class-gpio-its-deprecated](https://www.thegoodpenguin.co.uk/blog/stop-using-sys-class-gpio-its-deprecated/)
  شرح عملي ليه لازم تتحول للـ character device interface.

- **The descriptor-based GPIO interface** (Embedded.com):
  [embedded.com/linux-device-driver-development-the-descriptor-based-gpio-interface](https://www.embedded.com/linux-device-driver-development-the-descriptor-based-gpio-interface/)
  دليل مفصّل لكتابة drivers بالـ `gpiod_*` API.

---

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل:** Chapter 1 (An Introduction to Device Drivers) + Chapter 14 (The Linux Device Model)
- **الرابط المجاني:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- **ملاحظة:** الكتاب قديم (kernel 2.6) لكن مبادئ الـ bus/device/driver model اللي شرحها لسه صاحية.

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل المهم:** Chapter 17 (Devices and Modules)
- الكتاب بيشرح الـ device model والـ bus infrastructure اللي بيتبني عليها الـ gpiolib.

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل المهم:** Chapter 15 (Embedded Linux Quick Start)
- بيغطي الـ GPIO من منظور embedded systems بأمثلة hardware حقيقية.

#### The Linux Programming Interface — Michael Kerrisk
- مش عن الـ kernel internals بالظبط، لكن مفيد لفهم الـ user-space side للـ `/dev/gpiochipN`.

---

### ملفات الـ Headers المهمة في الـ Source Tree

```
include/linux/gpio/
├── driver.h       # struct gpio_chip وكل الـ driver API
├── consumer.h     # gpiod_get/set/free للـ consumer drivers
└── machine.h      # gpiod_lookup_table للـ platform data mappings

include/linux/gpio.h          # legacy API (deprecated)
include/uapi/linux/gpio.h     # ABI مع الـ userspace (ioctl structs)
drivers/gpio/
├── gpiolib.c      # الكود الأساسي — core implementation
├── gpiolib.h      # internal structs (gpio_device, gpio_desc)
├── gpiolib-cdev.c # character device interface
├── gpiolib-sysfs.c# sysfs interface (deprecated)
├── gpiolib-of.c   # Device Tree integration
└── gpiolib-acpi.c # ACPI integration
```

---

### كلمات البحث المقترحة

لو حابب تعمق أكتر، استخدم الكلمات دي:

```
gpiolib linux kernel internals
gpio_chip driver implementation
gpio descriptor gpiod_get kernel
linux gpio character device ioctl
gpio irq domain kernel
pinctrl gpio multiplexing linux
gpiod_lookup_table platform data
linux gpio SRCU locking
gpio_device kobject lifetime
linux gpio userspace ABI v2
```

للبحث في الـ mailing list:
```
https://lore.kernel.org/linux-gpio/?q=gpiolib
```

للبحث في الـ kernel source مباشرة:
```bash
# كل الـ GPIO drivers
ls drivers/gpio/

# دالة معينة
git log -S "gpiod_get" --oneline drivers/gpio/gpiolib.c

# من كتب سطر معين
git blame drivers/gpio/gpiolib.c
```
## Phase 8: Writing simple module

### الـ Target Function

الـ function اللي هنـhook عليها هي **`gpiod_direction_input`** — معرفة في `drivers/gpio/gpiolib.c` وـ exported بـ `EXPORT_SYMBOL_GPL`. بتتحكم في تحويل أي GPIO line لـ input mode، وبتتحكال من أي driver أو subsystem بيستخدم الـ GPIO API.

اختيارها مناسب لأن:
- بتتحكال كتير في الـ system (كل ما device بييجي up بيضبط GPIO directions).
- الـ argument بتاعها هو `struct gpio_desc *` — فيه معلومات مفيدة زي رقم الـ GPIO والـ label.
- الـ kprobe بيقدر يمسكها safely قبل ما تتنفذ.

---

### الـ Module Code

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * gpio_direction_probe.c
 *
 * A kprobe module that intercepts gpiod_direction_input() calls
 * and logs the GPIO number + device label to the kernel ring buffer.
 *
 * Useful for tracing which GPIOs are being set to input at runtime.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* kprobe, register_kprobe */
#include <linux/gpio/driver.h>  /* struct gpio_chip */
#include <linux/gpio.h>         /* struct gpio_desc (opaque pointer) */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on gpiod_direction_input — logs GPIO direction changes to input");

/* ------------------------------------------------------------------ */
/* Forward declaration of internal gpiolib struct we need to inspect.  */
/* We only access public fields via desc->gdev which is a gpio_device. */
/* Since gpio_desc is opaque in public headers, we replicate the        */
/* minimal layout we need — matching the kernel source exactly.         */
/* ------------------------------------------------------------------ */

/*
 * Minimal replica of struct gpio_device (from drivers/gpio/gpiolib.h).
 * We only need `base` (first GPIO number) and `label` (chip name).
 * Using offsetof tricks is fragile; instead we rely on the stable
 * exported helpers: desc_to_gpio() and gpiod_get_label() — both GPL.
 */

/* desc_to_gpio() and gpiod_get_label() are exported by gpiolib.c */
extern int desc_to_gpio(const struct gpio_desc *desc);
extern const char *gpiod_get_label(struct gpio_desc *desc);

/* ------------------------------------------------------------------ */
/* kprobe pre-handler: runs just before gpiod_direction_input()        */
/* ------------------------------------------------------------------ */

static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument is in RDI (regs->di).
     * On ARM64 it is in x0 (regs->regs[0]).
     * kprobes give us pt_regs, so we cast the first arg register
     * to get the gpio_desc pointer passed by the caller.
     */
#ifdef CONFIG_X86_64
    struct gpio_desc *desc = (struct gpio_desc *)regs->di;
#elif defined(CONFIG_ARM64)
    struct gpio_desc *desc = (struct gpio_desc *)regs->regs[0];
#else
    /* Generic fallback — may not work on all architectures */
    struct gpio_desc *desc = (struct gpio_desc *)regs->ARG1;
#endif

    /* Validate pointer before dereferencing */
    if (!desc || IS_ERR(desc))
        return 0;

    /* Use stable exported helpers — no internal struct access needed */
    pr_info("gpio_direction_probe: gpiod_direction_input() called | "
            "gpio=%d label=%s\n",
            desc_to_gpio(desc),
            gpiod_get_label(desc) ?: "(unrequested)");

    return 0; /* 0 = let the real function continue executing */
}

/* ------------------------------------------------------------------ */
/* kprobe post-handler: runs after gpiod_direction_input() returns     */
/* ------------------------------------------------------------------ */

static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * Return value is in RAX (x86-64) or x0 (ARM64).
     * We just log it so the user can see if the call succeeded.
     */
#ifdef CONFIG_X86_64
    long retval = (long)regs->ax;
#elif defined(CONFIG_ARM64)
    long retval = (long)regs->regs[0];
#else
    long retval = 0;
#endif

    if (retval)
        pr_info("gpio_direction_probe: gpiod_direction_input() returned %ld (error)\n",
                retval);
}

/* ------------------------------------------------------------------ */
/* kprobe struct                                                        */
/* ------------------------------------------------------------------ */

static struct kprobe kp = {
    .symbol_name = "gpiod_direction_input", /* الـ function اللي هنـhook عليها */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* module_init / module_exit                                            */
/* ------------------------------------------------------------------ */

static int __init gpio_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("gpio_direction_probe: register_kprobe failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("gpio_direction_probe: kprobe planted at %s (%p)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit gpio_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("gpio_direction_probe: kprobe removed from %s\n",
            kp.symbol_name);
}

module_init(gpio_probe_init);
module_exit(gpio_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | يحتوي على `struct kprobe`، `register_kprobe`، `unregister_kprobe` وتعريف `pt_regs`. |
| `linux/gpio/driver.h` | يعرف `struct gpio_chip` — محتاجينه لأن `gpio_desc` بترجع إليه. |
| `linux/gpio.h` | يجيب الـ public GPIO API زي `gpio_desc` كـ opaque type. |

الـ includes دي بتديك كل اللي محتاجه من غير ما تمس headers داخلية.

#### الـ `handler_pre`

بيتشغل **قبل** تنفيذ `gpiod_direction_input` بالظبط. بنقرأ الـ argument الأول من الـ registers (RDI على x86-64، x0 على ARM64) عشان نجيب الـ `gpio_desc *`. بعدين بنستخدم `desc_to_gpio()` و`gpiod_get_label()` — اللي هما exported بـ GPL — عشان نطبع رقم الـ GPIO واسم الـ label من غير ما نعتمد على internal structs.

#### الـ `handler_post`

بيتشغل **بعد** ما الـ function ترجع. بنطبع الـ return value لو كان فيه error — ده مفيد لو محتاج تعرف مين بيفشل في ضبط الـ GPIO direction.

#### الـ `struct kprobe`

الـ `.symbol_name` بيقول للـ kernel تحط الـ breakpoint فين. الـ kernel بيحل الاسم لـ address تلقائياً من الـ symbol table، فمش محتاج تـhardcode عنوان.

#### الـ `module_init`

`register_kprobe()` بتزرع الـ hook في الـ kernel. لو فشلت (مثلاً لو الـ function `inline` أو `__kprobes` protected) بترجع error ومش بيتحمل الـ module.

#### الـ `module_exit`

`unregister_kprobe()` **لازم** تتعمل في الـ exit عشان تشيل الـ hook قبل ما الـ module يتـunload. لو متعملتيش، الـ kernel هيحاول يتصل بـ handler code اتشالت من الـ memory وده kernel panic.

---

### الـ Makefile

```makefile
obj-m += gpio_direction_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة الـ Module

```bash
# بناء الـ module
make

# تحميل الـ module
sudo insmod gpio_direction_probe.ko

# شوف الـ output في الـ ring buffer
sudo dmesg | grep gpio_direction_probe

# لو عندك GPIO activity (مثلاً بعد plug/unplug device)
# هتلاقي output زي:
# gpio_direction_probe: gpiod_direction_input() called | gpio=496 label=button@0

# إزالة الـ module
sudo rmmod gpio_direction_probe

# تأكيد إن الـ kprobe اتشال
sudo dmesg | tail -5
```

---

### مثال على الـ Output المتوقع

```
[  123.456789] gpio_direction_probe: kprobe planted at gpiod_direction_input (ffffffffc0123456)
[  124.001234] gpio_direction_probe: gpiod_direction_input() called | gpio=496 label=button@0
[  124.001300] gpio_direction_probe: gpiod_direction_input() called | gpio=497 label=(unrequested)
[  130.999999] gpio_direction_probe: kprobe removed from gpiod_direction_input
```

الـ label بييجي `(unrequested)` لو الـ GPIO مش requested بعد — ده سلوك طبيعي في بعض init paths.
