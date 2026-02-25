## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي ليه الملف

**الـ GPIO Subsystem** — واحد من أكبر subsystems في الـ Linux kernel، بيشيل كل كود إدارة الـ GPIO lines. المنتِجون الرئيسيون هما Linus Walleij وBartosz Golaszewski، والـ mailing list هي `linux-gpio@vger.kernel.org`.

الملف `gpiolib.h` هو **الـ internal header** الخاص بقلب الـ GPIO subsystem — مش للاستخدام العام، ده للكود الداخلي بس.

---

### المشكلة اللي بيحلها الـ GPIO Subsystem

تخيل إنك بتبني SoC (System on Chip) — زي اللي في الـ Raspberry Pi أو أي embedded board. الـ chip دي فيها عشرات أو مئات من الـ pins. بعض الـ pins دي بيجي منها أو بيروح فيها إشارة رقمية 0 أو 1 — دي هي الـ **GPIO lines** (General Purpose Input/Output).

مثال حياتي: تخيل مبنى فيه مئة نافذة. كل نافذة ممكن تكون:
- **مفتوحة أو مغلقة** (قراءة input — زي حساس باب أو زرار)
- **بتفتحها أو بتسكرها إنت** (كتابة output — زي تشغيل LED أو relay)

المشكلة إن كل vendor بيعمل chip بطريقة مختلفة في الـ hardware. فكيف يعمل الـ kernel كود واحد موحد يشتغل على كل الـ chips دي؟ الإجابة هي الـ **GPIO Subsystem** مع مكتبته المركزية **gpiolib**.

---

### الصورة الكبيرة: إزاي الـ gpiolib شغال

```
+-------------------+
|  User Space App   |  ← /dev/gpiochipN أو /sys/class/gpio
+-------------------+
         |
+-------------------+
|   GPIO Character  |  ← gpiolib-cdev.c
|   Device (CDEV)   |
+-------------------+
         |
+-------------------+
|   gpiolib Core    |  ← gpiolib.c + gpiolib.h  ← احنا هنا
|  (الكود الداخلي)  |
+-------------------+
         |
    +---------+----------+----------+
    |         |          |          |
  gpio_chip  pinctrl   ACPI    Device Tree
  driver     subsystem          (OF)
    |
+-------------------+
|  Hardware Driver  |  ← gpio-aspeed.c, gpio-zynq.c, ... إلخ
| (كود الـ hardware)|
+-------------------+
         |
+-------------------+
|  الـ GPIO pins    |  ← الحديد الفعلي
|  على الـ SoC      |
+-------------------+
```

الـ `gpiolib.h` هو **العمود الفقري الداخلي** — بيعرّف الـ structs والـ functions اللي بيستخدمها كود الـ gpiolib مع بعضه داخليًا، من غير ما يكشفها للـ drivers أو الـ user space.

---

### أهم ما في الملف: الـ Structs الثلاثة

#### 1. `struct gpio_device` — البطاقة الشخصية للـ GPIO Controller

ده الـ container الرئيسي اللي بيمثل جهاز GPIO كامل في الـ kernel. فيه:
- `dev` و`chrdev`: بيخلي الجهاز يظهر في `/dev/gpiochipN`
- `chip __rcu`: pointer للـ `gpio_chip` اللي فيه callbacks الـ hardware الفعلية — محمي بـ **SRCU** (Sleepable RCU) عشان لو الـ chip اتشالت من الـ system، الـ pointers ما تتكسرش
- `descs`: array من الـ descriptors — كل عنصر بيمثل line واحدة
- `line_state_notifier`: بيبعت events لو line اتغير حالتها
- `can_sleep`: مهم جدًا — بيحدد إذا كان الـ driver ممكن يـ sleep أو لازم يشتغل في atomic context

#### 2. `struct gpio_desc` — التمثيل الداخلي لـ GPIO Line واحدة

كل line عندها descriptor خاص بيها. فيه:
- `flags`: بيتمask فيه كل حالة الـ line — هل هي input أو output؟ active-low؟ connected to IRQ؟ مطلوبة؟ مـ exported؟
- `label`: مين اللي طلبها (اسم الـ consumer)
- `name`: اسم الـ line نفسها من الـ device tree أو ACPI

الـ descriptor ده بيفضل موجود حتى لو الـ hardware driver اتفرغ — لأن userspace ممكن لسه شايل reference ليه.

#### 3. `struct gpio_array` — Fast Path لـ Array Operations

لو عندك موتور بيتحكم فيه 8 GPIOs مع بعض — ما تجيش تعملهم set كل واحد لوحده. الـ `gpio_array` بيخزن معلومات مُحسوبة مسبقًا (masks) عشان تعمل set لكذا line بـ operation واحدة بدل كذا operations.

---

### الـ Separation بين `gpio_device` و`gpio_chip`

ده من أذكى التصاميم في الـ GPIO subsystem:

| `gpio_device` | `gpio_chip` |
|---|---|
| بيعيش طول ما في مستخدم للـ device | بيتشال مع الـ hardware driver |
| فيه الـ runtime state والـ descriptors | فيه الـ hardware callbacks |
| محمي بـ SRCU | ممكن يتشال في أي وقت |
| بيتعامل معه الـ user space | بيتعامل معه الـ kernel drivers فقط |

الحكمة: لو driver اتـ unload، الـ `gpio_chip` بيتشال، لكن الـ `gpio_device` بيفضل موجود لحد ما كل الـ user space يخلص من الـ file descriptors بتاعته.

---

### الـ `gpio_chip_guard` — الحماية من الـ Race Conditions

```c
/* مثال: لما بتاخد قراءة من GPIO */
scoped_guard(gpio_chip_guard, desc) {
    /* هنا مضمون إن الـ chip موجودة */
    gc->get(gc, offset);
}
/* هنا الـ guard اتحرر — الـ chip ممكن تتشال دلوقتي */
```

ده بيحمي من الـ scenario ده:
1. Thread A بيقرأ GPIO
2. Thread B بيعمل `rmmod` للـ driver
3. Thread A ما يتعملش crash

الـ SRCU lock بيضمن إن الـ `gpio_chip` pointer صالح طول مدة استخدامه.

---

### الـ Notifier System

الملف فيه نظام إشعارات على مستويين:
- `line_state_notifier` + `line_state_wq`: بيبعت events لما GPIO line تتـ request أو تتـ release أو يتغير config — ده اللي بيشغل الـ `/dev/gpiochipN` character device events
- `device_notifier`: بيبلغ الـ waitqueues لما الـ device نفسه بيتشال

---

### الـ Flags بالتفصيل

```c
#define GPIOD_FLAG_REQUESTED    0  /* شايلها consumer */
#define GPIOD_FLAG_IS_OUT       1  /* output mode */
#define GPIOD_FLAG_ACTIVE_LOW   6  /* القيمة المنطقية معكوسة */
#define GPIOD_FLAG_USED_AS_IRQ  9  /* متوصل بـ interrupt */
#define GPIOD_FLAG_IS_HOGGED    11 /* الـ kernel نفسه حاجزها */
#define GPIOD_FLAG_TRANSITORY   12 /* بتفقد قيمتها بعد sleep */
#define GPIOD_FLAG_SHARED       20 /* أكتر من consumer بيستخدموها */
```

الـ **active-low** مثلًا: في بعض الـ hardware، LED بتضيء لما الـ pin = 0 مش 1. الـ kernel بيعكس القيم تلقائيًا لما الـ flag ده يكون set.

---

### الـ Macros للـ Iteration

```c
/* اتمشى على كل GPIOs في chip */
for_each_gpio_desc(gc, desc) { ... }

/* اتمشى على اللي عندها flag معين بس */
for_each_gpio_desc_with_flag(gc, desc, GPIOD_FLAG_REQUESTED) { ... }

/* اتمشى على كل property names ممكنة من DT/ACPI */
for_each_gpio_property_name(propname, "reset") { ... }
/* بيجرب: "reset-gpios", "reset-gpio", ... */
```

---

### الملفات المكونة للـ Subsystem

#### الـ Core (قلب الـ subsystem)

| الملف | الدور |
|---|---|
| `drivers/gpio/gpiolib.c` | التنفيذ الرئيسي — registration، request، set/get |
| `drivers/gpio/gpiolib.h` | **ملفنا** — الـ internal structs والـ APIs |
| `drivers/gpio/gpiolib-cdev.c` | الـ character device — `/dev/gpiochipN` |
| `drivers/gpio/gpiolib-sysfs.c` | الـ sysfs interface القديم — `/sys/class/gpio` |
| `drivers/gpio/gpiolib-devres.c` | الـ `devm_gpiod_*` functions |
| `drivers/gpio/gpiolib-of.c` | قراءة GPIO lookups من الـ Device Tree |
| `drivers/gpio/gpiolib-acpi-core.c` | قراءة GPIO lookups من ACPI |
| `drivers/gpio/gpiolib-swnode.c` | الـ software node lookups |
| `drivers/gpio/gpiolib-legacy.c` | الـ integer-based API القديم |
| `drivers/gpio/gpiolib-shared.c` | shared GPIO support |

#### الـ Public Headers

| الملف | الدور |
|---|---|
| `include/linux/gpio/driver.h` | الـ API للـ GPIO chip drivers |
| `include/linux/gpio/consumer.h` | الـ API للـ device drivers اللي بتستخدم GPIOs |
| `include/linux/gpio.h` | legacy GPIO API |
| `include/linux/of_gpio.h` | Device Tree GPIO helpers |

#### مثال على Hardware Drivers

| الملف | الـ Hardware |
|---|---|
| `drivers/gpio/gpio-aspeed.c` | ASPEED BMC SoC |
| `drivers/gpio/gpio-pl061.c` | ARM PrimeCell PL061 |
| `drivers/gpio/gpio-zynq.c` | Xilinx Zynq |
| `drivers/gpio/gpio-aggregator.c` | virtual GPIO من عدة chips |

---

### ملفات مهمة للقارئ

- **`drivers/gpio/gpiolib.c`**: الـ implementation الفعلي لكل اللي في `gpiolib.h`
- **`include/linux/gpio/driver.h`**: الـ `struct gpio_chip` الكاملة — دي اللي بيملأها كل hardware driver
- **`include/linux/gpio/consumer.h`**: `gpiod_get()`, `gpiod_set_value()` وغيرها — دي الـ API اللي بتستخدمها الـ drivers الأخرى
- **`drivers/gpio/gpiolib-cdev.h`** و **`gpiolib-cdev.c`**: عشان تفهم إزاي الـ userspace بيتكلم مع الـ GPIO
- **`Documentation/driver-api/gpio/`**: الـ documentation الرسمية
## Phase 2: شرح الـ GPIO Framework

### المشكلة — ليه الـ GPIO subsystem موجود أصلاً؟

في أي SoC زي Raspberry Pi أو STM32 أو i.MX8، في الآف الـ GPIO pins. المشكلة إن مصادر الـ GPIO مختلفة جداً:

- **GPIO controller داخل الـ SoC نفسه** — زي `pl061` على ARM أو `mxc-gpio` على i.MX
- **GPIO expander على I2C/SPI** — زي `pcf8574` أو `mcp23017`
- **FPGA** بيوفر GPIO lines
- **PMIC** بيعرض GPIO pins

كل واحد من دول بيتحكم في الـ hardware بطريقة مختلفة خالص. لو كل driver كتب كود خاص بيه عشان يتعامل مع الـ GPIO، هيبقى عندنا:

1. كود متكرر بشكل هائل في كل driver
2. مستحيل تعمل driver portable بين boards مختلفة
3. مفيش طريقة موحدة للـ userspace يوصل للـ GPIO

---

### الحل — فكرة الـ gpiolib

الـ kernel حل المشكلة دي بالـ **gpiolib** — وهو الـ GPIO subsystem. الفكرة المحورية:

> كل قطعة hardware بتوفر GPIO تعمل **`gpio_chip`** وتملأ function pointers. الـ gpiolib يتولى الباقي كله.

الـ framework بيفصل بين **provider** (الـ driver اللي بيتحكم في الـ hardware) و **consumer** (الـ driver اللي عايز يستخدم GPIO — زي driver للـ LED أو button).

---

### التشبيه العملي — البريد الإلكتروني

تخيل إن عندك تطبيق بريد إلكتروني. التطبيق ده بيعمل `send()` من غير ما يعرف هل هتتبعت الرسالة via Gmail SMTP أو Exchange أو Postfix.

| تشبيه البريد | GPIO subsystem |
|---|---|
| تطبيق البريد اللي بيقول "ابعت رسالة" | الـ consumer driver (LED, button, regulator) |
| الـ `send()` API | `gpiod_get()`, `gpiod_set_value()` |
| Gmail / Exchange / Postfix | الـ GPIO provider driver (`gpio_chip`) |
| الـ mail server configuration | device tree / ACPI lookup |
| الـ mail framework نفسه (JavaMail مثلاً) | الـ gpiolib (gpiolib.c) |
| الـ email address | الـ `gpio_desc` — مش رقم، بيشير لـ pin معين بالسياق |

الـ consumer driver بيطلب GPIO بـ connection ID زي `"reset"` أو `"enable"` — مش برقم. الـ gpiolib بيبحث في device tree أو ACPI عن الـ mapping ده ويرجعله `gpio_desc*` جاهز.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        USERSPACE                                │
  │         /dev/gpiochipN  (character device - modern API)         │
  │         /sys/class/gpio (sysfs - legacy API)                    │
  └──────────────┬──────────────────────────────────────────────────┘
                 │
  ┌──────────────▼──────────────────────────────────────────────────┐
  │                      KERNEL - gpiolib                           │
  │                                                                 │
  │   ┌────────────────┐    ┌──────────────────────────────────┐    │
  │   │  Consumer API  │    │         Core Logic               │    │
  │   │                │    │                                  │    │
  │   │  gpiod_get()   │    │  - Descriptor lookup (DT/ACPI)   │    │
  │   │  gpiod_set_    │───▶│  - Active-low translation        │    │
  │   │    value()     │    │  - Flag management               │    │
  │   │  gpiod_to_irq()│    │  - Ownership/request tracking    │    │
  │   │  gpiod_put()   │    │  - Notifier chain                │    │
  │   └────────────────┘    └──────────────┬───────────────────┘    │
  │                                        │                        │
  │              ┌─────────────────────────▼──────────────┐         │
  │              │           gpio_device                   │         │
  │              │  (runtime state, lives beyond chip)     │         │
  │              │                                         │         │
  │              │  ┌──────────────────────────────────┐   │         │
  │              │  │   gpio_desc[]  (per-line state)   │   │         │
  │              │  │   flags, label, debounce, ...     │   │         │
  │              │  └──────────────────────────────────┘   │         │
  │              │                                         │         │
  │              │  ┌──────────────────────────────────┐   │         │
  │              │  │   gpio_chip* (RCU pointer)        │   │         │
  │              │  │   → direction_input()             │   │         │
  │              │  │   → direction_output()            │   │         │
  │              │  │   → get() / set()                 │   │         │
  │              │  │   → to_irq()                      │   │         │
  │              │  └──────────────────────────────────┘   │         │
  │              └─────────────────────────────────────────┘         │
  └──────────────────────────────────────────────────────────────────┘
                 │
  ┌──────────────▼──────────────────────────────────────────────────┐
  │                  GPIO Provider Drivers                          │
  │                                                                 │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
  │  │  pl061.c    │  │  pcf8574.c  │  │  mxc-gpio.c             │ │
  │  │ (ARM SoC    │  │  (I2C       │  │  (i.MX SoC              │ │
  │  │  GPIO ctrl) │  │  expander)  │  │  GPIO controller)       │ │
  │  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────────┘ │
  └─────────┼────────────────┼────────────────────┼────────────────┘
            │                │                    │
  ┌─────────▼────────────────▼────────────────────▼────────────────┐
  │                       HARDWARE                                  │
  │    ARM PrimeCell    PCF8574 chip          i.MX8 GPIO block      │
  └─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الـ Descriptor Model

الـ abstraction الأساسية في الـ gpiolib هي إنه بدل ما تتعامل مع **أرقام GPIO** (اللي كانت global وبتتعارض)، دلوقتي بتتعامل مع **`gpio_desc*`** — وهو pointer لـ struct بيمثل الـ pin بكل سياقه.

الـ `gpio_desc` مش بس رقم — هو بيحتوي على:
- مين اللي طلبه (`label`)
- هل هو active-low ولا active-high
- هل هو output ولا input
- هل بيتستخدم كـ IRQ
- الـ debounce settings

ده معناه إن الـ consumer driver مش محتاج يفهم الـ hardware على الإطلاق — بيتعامل مع منطق التطبيق بس.

---

### الـ Structs الأساسية وعلاقتها ببعض

```
  ┌──────────────────────────────────────────────────────────────┐
  │                      gpio_device                             │
  │                                                              │
  │  dev ──────────────────────────────▶ struct device          │
  │  chrdev ───────────────────────────▶ struct cdev            │
  │  chip ──────(RCU ptr)──────────────▶ struct gpio_chip       │
  │  descs ─────────────────────────┐                           │
  │  ngpio (عدد الـ lines)          │                           │
  │  valid_mask (bitmask)           │                           │
  │  label ("gpiochip0")            │                           │
  │  list ──────────────────────────┼──▶ linked list of all     │
  │  line_state_notifier            │       gpio_devices         │
  │  device_notifier                │                           │
  │  srcu (protects chip ptr)       │                           │
  │  pin_ranges ────────────────────┼──▶ list of gpio_pin_range │
  └─────────────────────────────────┼───────────────────────────┘
                                    │
                                    ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                gpio_desc  [array of ngpio]                  │
  │                                                             │
  │  gdev ──────────────────────────────▶ parent gpio_device   │
  │  flags (bitmask)                                           │
  │    bit 0: REQUESTED      bit 9: USED_AS_IRQ                │
  │    bit 1: IS_OUT         bit 11: IS_HOGGED                 │
  │    bit 6: ACTIVE_LOW     bit 13: PULL_UP                   │
  │    bit 7: OPEN_DRAIN     bit 14: PULL_DOWN                 │
  │  label (RCU ptr) ──────────────────▶ gpio_desc_label       │
  │  name (const char*)                                        │
  │  debounce_period_us                                        │
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │                      gpio_chip                              │
  │           (hardware abstraction - driver fills this)        │
  │                                                             │
  │  label                                                      │
  │  gpiodev ───────────────────────────▶ parent gpio_device   │
  │  parent ────────────────────────────▶ struct device        │
  │  ngpio, base, offset                                        │
  │  can_sleep (I2C/SPI expanders set this)                    │
  │                                                             │
  │  /* Function pointers filled by driver */                  │
  │  request()           → optional, enable power/clock        │
  │  free()              → optional, disable power/clock       │
  │  get_direction()     → read current direction              │
  │  direction_input()   → configure as input                  │
  │  direction_output()  → configure as output + set value     │
  │  get()               → read single pin value               │
  │  get_multiple()      → read multiple pins atomically       │
  │  set()               → write single pin value              │
  │  set_multiple()      → write multiple pins atomically      │
  │  set_config()        → pull-up/down, drive strength, etc.  │
  │  to_irq()            → convert pin offset to IRQ number    │
  │                                                             │
  │  irq ───────────────────────────────▶ gpio_irq_chip        │
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │                    gpio_array                               │
  │          (optimization for multi-pin operations)            │
  │                                                             │
  │  desc[]     → array of gpio_desc pointers                  │
  │  size        → number of pins                              │
  │  gdev        → parent gpio_device                          │
  │  get_mask    → bitmask for fast read path                  │
  │  set_mask    → bitmask for fast write path                  │
  │  invert_mask → flexible array for active-low handling      │
  └─────────────────────────────────────────────────────────────┘
```

---

### الـ gpio_device مقابل الـ gpio_chip — الفرق الجوهري

ده من أهم الـ design decisions في الـ gpiolib الحديث:

**الـ `gpio_chip`** هو الـ hardware-specific part — الـ driver بيملأه ويسجله. لو الـ driver اتـ unload (مثلاً module removal)، الـ `gpio_chip` بيتشال.

**الـ `gpio_device`** هو الـ persistent runtime state. بيعيش حتى لو الـ `gpio_chip` اتشال، طالما في userspace process لسه فاتحة الـ `/dev/gpiochipN`. ده بيحمي الـ userspace من الـ crash.

العلاقة بينهم:
```c
/* gpio_device بيمسك pointer للـ gpio_chip عبر RCU */
struct gpio_chip __rcu *chip;  /* in gpio_device */

/* gpio_chip بيرجع للـ gpio_device */
struct gpio_device *gpiodev;   /* in gpio_chip */
```

الـ **SRCU (Sleepable RCU)** هنا مهم جداً — بيسمح للـ reader يـ sleep أثناء الـ critical section، وده ضروري لأن بعض الـ GPIO operations تحتاج تتعامل مع I2C/SPI اللي ممكن تنام.

---

### الـ gpio_chip_guard — حماية الـ chip pointer

```c
struct gpio_chip_guard {
    struct gpio_device *gdev;
    struct gpio_chip   *gc;
    int                 idx;
};

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

ده بيستخدم الـ **scoped resource management** — نوع من الـ RAII في C. لما تعمل `CLASS(gpio_chip_guard, guard)(desc)` في function، الـ SRCU read lock بيتأخد أوتوماتيك، وبينطلق لما تخرج من الـ scope. ده بيضمن إن الـ `gpio_chip` ما يتشالش من تحت إيدنا وإحنا بنشتغل بيه.

---

### الـ Active-Low Translation — بيتعمل فين؟

ده من الأشياء اللي الـ gpiolib بيعملها نيابة عن كل driver.

لو pin مكتوب في device tree إنه `active-low`:
```dts
reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
```

الـ gpiolib بيحط `GPIOD_FLAG_ACTIVE_LOW` في الـ `gpio_desc.flags`. لما الـ consumer يعمل:
```c
gpiod_set_value(desc, 1); /* logical HIGH = assert reset */
```

الـ gpiolib بيعكس القيمة أوتوماتيك قبل ما يبعتها للـ `gpio_chip->set()`. الـ provider driver ما بيشوفش الـ logic level — بيشوف الـ physical level بس.

لو الـ consumer عايز يتجاوز الـ translation ده، بيستخدم `gpiod_set_raw_value()`.

---

### الـ Notifier Chain — الـ GPIO Event System

الـ `gpio_device` بيحتوي على notifier chain مزدوج:

```c
struct raw_notifier_head  line_state_notifier;  /* for kernel subscribers */
struct blocking_notifier_head device_notifier;  /* for char dev waitqueues */
```

لما حالة أي GPIO line بتتغير (بتتطلب، بتتحرر، بتتضبط)، الـ `gpiod_line_state_notify()` بتبعث notification. الـ `line_state_wq` workqueue بيضمن إن الـ notifications دي بتتبعت من process context، مش من atomic context — ضروري لأن subscribers ممكن يحتاجوا يـ sleep.

---

### الـ gpio_array Fastpath — تحسين الـ Multi-Pin Operations

```c
struct gpio_array {
    struct gpio_desc  **desc;
    unsigned int        size;
    struct gpio_device *gdev;
    unsigned long      *get_mask;
    unsigned long      *set_mask;
    unsigned long       invert_mask[]; /* flexible array member */
};
```

لما بتطلب array من GPIOs عبر `gpiod_get_array()` وكلهم على نفس الـ `gpio_chip`، الـ gpiolib بيبني `gpio_array` بـ pre-computed bitmasks. الـ `gpiod_set_array_value_complex()` ممكن تعمل `set_multiple()` call واحدة بدل n calls. ده مهم جداً في embedded systems اللي بتتحكم في data buses أو LED matrices.

---

### الـ Pinctrl Integration

```c
#ifdef CONFIG_PINCTRL
    struct list_head pin_ranges; /* in gpio_device */
#endif
```

الـ **pinctrl subsystem** (subsystem منفصل) هو المسؤول عن ضبط الـ pin mux — بمعنى إن الـ pin ده هيشتغل كـ GPIO ولا كـ UART TX ولا كـ SPI CLK. لما GPIO driver بيسجل نفسه، ممكن يقول للـ pinctrl subsystem "الـ pin range من 0 لـ 31 بتاعتي" عبر `gpiochip_add_pin_range()`. الـ `gpio_pin_range` struct بيربط بين الـ GPIO offset والـ pinctrl pin number.

---

### ملخص — مين بيملك إيه

| المسؤولية | بيعملها |
|---|---|
| تسجيل وإلغاء تسجيل الـ GPIO chip | الـ provider driver عبر `gpiochip_add_data()` / `gpiochip_remove()` |
| تخصيص وإدارة الـ `gpio_desc` array | الـ gpiolib تلقائياً |
| Active-low / active-high translation | الـ gpiolib |
| البحث في device tree / ACPI عن GPIO | الـ gpiolib |
| Ownership tracking (مين بيستخدم الـ pin) | الـ gpiolib |
| توفير `/dev/gpiochipN` للـ userspace | الـ gpiolib عبر `cdev` |
| Fastpath bitmasks للـ arrays | الـ gpiolib |
| فعلياً قراءة/كتابة الـ hardware | الـ provider driver (gpio_chip callbacks) |
| Pin mux configuration | الـ pinctrl subsystem |
| IRQ domain للـ GPIO interrupts | الـ provider driver + gpiolib irqchip helper |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### `GPIOD_FLAG_*` — flags الـ `gpio_desc`

كل flag ده **bit number** جوه `desc->flags` (من النوع `unsigned long`).

| Bit | الاسم | المعنى |
|-----|-------|--------|
| 0 | `GPIOD_FLAG_REQUESTED` | الـ GPIO متطلوب (in use) |
| 1 | `GPIOD_FLAG_IS_OUT` | الـ GPIO في وضع output |
| 2 | `GPIOD_FLAG_EXPORT` | متصدّر لـ userspace |
| 3 | `GPIOD_FLAG_SYSFS` | متصدّر عبر `/sys/class/gpio` |
| 6 | `GPIOD_FLAG_ACTIVE_LOW` | منطق معكوس (active-low) |
| 7 | `GPIOD_FLAG_OPEN_DRAIN` | نوع open-drain |
| 8 | `GPIOD_FLAG_OPEN_SOURCE` | نوع open-source |
| 9 | `GPIOD_FLAG_USED_AS_IRQ` | متوصّل بـ IRQ |
| 10 | `GPIOD_FLAG_IRQ_IS_ENABLED` | الـ IRQ المتوصّل به enabled |
| 11 | `GPIOD_FLAG_IS_HOGGED` | الـ line محجوز (hogged) من kernel |
| 12 | `GPIOD_FLAG_TRANSITORY` | ممكن يفقد قيمته عند sleep/reset |
| 13 | `GPIOD_FLAG_PULL_UP` | pull-up مفعّل |
| 14 | `GPIOD_FLAG_PULL_DOWN` | pull-down مفعّل |
| 15 | `GPIOD_FLAG_BIAS_DISABLE` | bias معطّل |
| 16 | `GPIOD_FLAG_EDGE_RISING` | الـ CDEV بيكتشف rising edge |
| 17 | `GPIOD_FLAG_EDGE_FALLING` | الـ CDEV بيكتشف falling edge |
| 18 | `GPIOD_FLAG_EVENT_CLOCK_REALTIME` | timestamps من REALTIME clock |
| 19 | `GPIOD_FLAG_EVENT_CLOCK_HTE` | timestamps من hardware timestamp engine |
| 20 | `GPIOD_FLAG_SHARED` | الـ pin متشارك بين أكتر من consumer |
| 21 | `GPIOD_FLAG_SHARED_PROXY` | virtual proxy لـ shared pin |

#### `enum gpiod_flags` — flags التهيئة عند الطلب

**الـ** `enum gpiod_flags` بتتمرر لـ `gpiod_get()` عشان تحدد الاتجاه والقيمة الابتدائية.

| القيمة | المعنى |
|--------|--------|
| `GPIOD_ASIS` | مافيش تغيير في الاتجاه |
| `GPIOD_IN` | input |
| `GPIOD_OUT_LOW` | output، قيمة ابتدائية = 0 |
| `GPIOD_OUT_HIGH` | output، قيمة ابتدائية = 1 |
| `GPIOD_OUT_LOW_OPEN_DRAIN` | open-drain output = 0 |
| `GPIOD_OUT_HIGH_OPEN_DRAIN` | open-drain output = 1 |

#### Config Options مهمة

| الـ Kconfig | الأثر على `gpiolib.h` |
|-------------|----------------------|
| `CONFIG_PINCTRL` | يضيف `pin_ranges` جوه `gpio_device` |
| `CONFIG_OF_DYNAMIC` | يضيف `hog` (device_node) جوه `gpio_desc` |
| `CONFIG_GPIO_CDEV` | يضيف `debounce_period_us` جوه `gpio_desc` |
| `CONFIG_GPIOLIB_IRQCHIP` | يضيف `struct gpio_irq_chip irq` جوه `gpio_chip` |
| `CONFIG_OF_GPIO` | يضيف `of_gpio_n_cells` وغيره جوه `gpio_chip` |

---

### كل الـ Structs المهمة

#### 1. `struct gpio_device`

**الـ** `gpio_device` هو الـ **state container الرئيسي** للـ GPIO device. بيعيش طول ما في حد بيستخدم الـ device حتى لو الـ chip اتنزع.

| الـ Field | النوع | الشرح |
|-----------|-------|-------|
| `dev` | `struct device` | الـ kernel device object — أساس الـ device model |
| `chrdev` | `struct cdev` | الـ character device لتعامل userspace |
| `id` | `int` | رقم تعريف الـ chip (مثل: `gpiochip0`) |
| `owner` | `struct module *` | يمنع unload الـ module وفيه GPIO نشط |
| `chip` | `struct gpio_chip __rcu *` | pointer للـ chip الفعلي (محمي بـ SRCU) |
| `descs` | `struct gpio_desc *` | مصفوفة الـ descriptors (حجمها = `ngpio`) |
| `valid_mask` | `unsigned long *` | bitmask للـ GPIO lines الصالحة فعلاً |
| `desc_srcu` | `struct srcu_struct` | يحمي الـ descriptors المكشوفة لـ userspace |
| `ngpio` | `u16` | عدد الـ GPIO lines |
| `can_sleep` | `bool` | هل callbacks الـ driver ممكن تنام؟ |
| `base` | `unsigned int` | أول رقم GPIO في الـ global numberspace (deprecated) |
| `label` | `const char *` | اسم وصفي للـ chip (e.g. `"gpio-pca9539"`) |
| `data` | `void *` | بيانات خاصة بالـ driver instance |
| `list` | `struct list_head` | ربط الـ devices ببعض في قائمة عالمية |
| `line_state_notifier` | `struct raw_notifier_head` | إشعارات تغيير حالة الـ lines |
| `line_state_lock` | `rwlock_t` | يحمي الـ `line_state_notifier` |
| `line_state_wq` | `struct workqueue_struct *` | thread منفصل لإرسال أحداث الـ line state |
| `device_notifier` | `struct blocking_notifier_head` | إشعار الـ cdev wait queues عند unregister |
| `srcu` | `struct srcu_struct` | يحمي الـ pointer للـ `chip` |
| `pin_ranges` | `struct list_head` | (PINCTRL فقط) نطاقات الـ pins الخاصة بالـ controller |

**علاقته بالـ Structs التانية:**
- يحتوي على `struct gpio_chip __rcu *chip` ← الـ hardware abstraction
- يحتوي على مصفوفة `struct gpio_desc *descs`
- بيتوصله من `struct gpio_chip` عبر `gc->gpiodev`

---

#### 2. `struct gpio_chip`

**الـ** `gpio_chip` هو الـ **hardware abstraction layer** — بيعرّف كل الـ ops اللي الـ driver لازم يوفرها.

| الـ Field | الشرح |
|-----------|-------|
| `label` | اسم وظيفي للـ chip |
| `gpiodev` | pointer للـ `gpio_device` المقابل |
| `parent` | الـ parent device (مثلاً I2C device) |
| `fwnode` | firmware node (DT/ACPI) |
| `request` / `free` | تفعيل/تعطيل الـ GPIO pin عند الطلب/التحرير |
| `get_direction` | اقرا اتجاه الـ pin |
| `direction_input` / `direction_output` | اضبط الاتجاه |
| `get` / `get_multiple` | اقرا قيمة pin واحدة أو أكتر |
| `set` / `set_multiple` | اكتب قيمة pin واحدة أو أكتر |
| `set_config` | ضبط إعدادات متقدمة (pull-up، debounce، ...) |
| `to_irq` | حوّل GPIO offset لـ IRQ number |
| `init_valid_mask` | حدد الـ GPIOs الصالحة |
| `add_pin_ranges` | ربط الـ GPIO pins بالـ pinctrl |
| `en_hw_timestamp` / `dis_hw_timestamp` | تفعيل/تعطيل hardware timestamps |
| `base` | أول GPIO number (deprecated، مفضّل تبعت -1) |
| `ngpio` | عدد الـ GPIO lines |
| `can_sleep` | هل الـ get/set ممكن ينام (مثل I2C expanders)؟ |
| `irq` | (GPIOLIB_IRQCHIP) الـ irqchip المدمج |

---

#### 3. `struct gpio_desc`

**الـ** `gpio_desc` هو الـ **descriptor** لـ GPIO line واحدة — ده اللي الـ consumer بياخده من `gpiod_get()`.

| الـ Field | الشرح |
|-----------|-------|
| `gdev` | pointer للـ `gpio_device` الأب |
| `flags` | bitmask من `GPIOD_FLAG_*` |
| `label` | اسم الـ consumer (محمي بـ RCU) |
| `name` | اسم الـ line (من DT أو `gpio_chip.names`) |
| `hog` | (OF_DYNAMIC) الـ device_node اللي حاجز الـ line |
| `debounce_period_us` | (GPIO_CDEV) فترة الـ debounce بالـ microseconds |

---

#### 4. `struct gpio_array`

**الـ** `gpio_array` بتستخدمه `gpiod_get_array()` عشان تاخد مجموعة GPIO lines دفعة واحدة وتتعامل معاها كـ **fast-path batch**.

| الـ Field | الشرح |
|-----------|-------|
| `desc` | مصفوفة من pointers لـ `gpio_desc` |
| `size` | عدد الـ descriptors |
| `gdev` | الـ parent GPIO device |
| `get_mask` | bitmask للـ pins اللي هتتقرأ في fast path |
| `set_mask` | bitmask للـ pins اللي هتتكتب في fast path |
| `invert_mask[]` | bitmask للـ pins اللي قيمتها لازم تتعكس (active-low) — flexible array |

---

#### 5. `struct gpio_desc_label`

**الـ** `gpio_desc_label` wrapper صغير حول الـ consumer label عشان يتحرك بأمان مع الـ RCU.

```c
struct gpio_desc_label {
    struct rcu_head rh;  /* للـ RCU-safe freeing */
    char str[];          /* الـ label string نفسه -- flexible array */
};
```

---

#### 6. `struct gpio_chip_guard`

**الـ** `gpio_chip_guard` هو **RAII guard** بيضمن إن الـ `gpio_chip` pointer مش هيتشال تحت إيدك وانت شغال.

```c
struct gpio_chip_guard {
    struct gpio_device *gdev;  /* الـ device */
    struct gpio_chip   *gc;    /* الـ chip pointer (ممكن يبقى NULL) */
    int                 idx;   /* SRCU read-lock index */
};
```

بيتعمل بالـ `DEFINE_CLASS` macro — عند الخروج من الـ scope تلقائياً بيعمل `srcu_read_unlock`.

---

### علاقات الـ Structs — ASCII Diagram

```
                    +----------------------------------------------+
                    |          struct gpio_device                  |
                    |                                              |
                    |  dev (struct device)                         |
                    |  chrdev (struct cdev)                        |
                    |  id, label, base, ngpio                      |
                    |  can_sleep, owner                            |
                    |  valid_mask (unsigned long*)                 |
                    |  desc_srcu (srcu_struct) --------+           |
                    |  srcu (srcu_struct) ----------+  |           |
                    |  line_state_notifier          |  |           |
                    |  line_state_lock (rwlock_t)   |  |           |
                    |  line_state_wq                |  |           |
                    |  device_notifier              |  |           |
                    |  pin_ranges (list_head)       |  |           |
                    |  list (list_head) -----> global list         |
                    |                               |  |           |
                    |  chip (gpio_chip __rcu*) --+  |  |           |
                    |  descs (gpio_desc*) -----+ |  |  |           |
                    +-------------------------|-|-+--+--+----------+
                                             | |   |   |
                         protects chip ptr --+ |   |   +-- protects descs[]
                                               |   |
              +--------------------------------+   +---------------------+
              |                                                         |
              v                                                         v
  +---------------------------+                    +--------------------+
  |    struct gpio_chip       |                    | struct gpio_desc   |
  |                           |  [0..ngpio-1]      |  (one per line)    |
  |  label, owner             |                    |                    |
  |  gpiodev* ---------------------back-ptr------> |  gdev* --------+  |
  |  parent (device*)         |                    |  flags         |  |
  |  fwnode                   |                    |  label* -----> |  |
  |                           |                    |    rcu_head    |  |
  |  request()                |                    |    char str[]  |  |
  |  free()                   |                    |  name*         |  |
  |  get_direction()          |                    |  hog*          |  |
  |  direction_input()        |                    |  debounce_us   |  |
  |  direction_output()       |                    +----------------+  |
  |  get()                    |                         |              |
  |  get_multiple()           |                         +-back-ptr-----+
  |  set()                    |
  |  set_multiple()           |     +------------------------------+
  |  set_config()             |     |    struct gpio_array         |
  |  to_irq()                 |     |                              |
  |  init_valid_mask()        |     |  desc[] --> gpio_desc ptrs   |
  |  add_pin_ranges()         |     |  size                        |
  |  en/dis_hw_timestamp()    |     |  gdev* --> gpio_device       |
  |                           |     |  get_mask                    |
  |  base, ngpio, can_sleep   |     |  set_mask                    |
  |  irq (gpio_irq_chip)      |     |  invert_mask[]               |
  +---------------------------+     +------------------------------+

  +------------------------------+
  |   struct gpio_chip_guard     |
  |                              |
  |  gdev* --> gpio_device       |
  |  gc*   --> gpio_chip (SRCU)  |
  |  idx   = srcu lock token     |
  +------------------------------+
```

---

### دورة حياة الـ GPIO Device — Lifecycle Diagram

```
Driver fills in:
  gc->label, gc->ngpio, gc->get(), gc->set(), ...
             |
             v
    gpiochip_add_data(gc, data)
             |
             +-- allocate gpio_device
             +-- allocate gpio_desc[ngpio]
             +-- register device in /sys
             +-- register cdev (character device)
             +-- link gc->gpiodev <-> gdev->chip
             +-- call init_valid_mask() if present
             +-- call add_pin_ranges()  (CONFIG_PINCTRL)
             +-- gpiochip_irqchip_add() (CONFIG_GPIOLIB_IRQCHIP)
             +-- add gdev to global list
                          |
                          v
              +---- REGISTERED & ACTIVE ----+
              |                             |
              |  consumer: gpiod_get()      |
              |    -> gpiod_request()       |
              |    -> set REQUESTED flag    |
              |    -> configure direction   |
              |    -> gpiod_configure_flags |
              |                             |
              |  consumer: gpiod_set_value()|
              |    -> gpio_chip_guard(SRCU) |
              |    -> gc->set()             |
              |                             |
              |  consumer: gpiod_put()      |
              |    -> gpiod_free()          |
              |    -> clear flags           |
              |    -> notify line_state     |
              +-----------------------------+
                          |
                          v  (driver unload or device removal)
             gpiochip_remove(gc)
             |
             +-- srcu_assign_pointer(gdev->chip, NULL)
             +-- synchronize_srcu(&gdev->srcu)
             +-- remove irqchip
             +-- remove pin ranges
             +-- unregister cdev
             +-- device_del() from /sys
             +-- put_device()
                  |
                  +-- no more refs?  --> kfree(gdev->descs), kfree(gdev)
                  +-- userspace refs?--> gdev stays alive, chip=NULL
                                         (guard returns gc=NULL -> ENODEV)
```

---

### Call Flow Diagrams

#### طلب GPIO من Consumer

```
consumer driver calls:
  gpiod_get(dev, "enable", GPIOD_OUT_LOW)
    |
    +-> gpiod_find_and_request(dev, fwnode, "enable", 0, GPIOD_OUT_LOW, label, true)
          |
          +-> lookup in DT/ACPI/platform data
          |     -> finds gpio_desc
          |
          +-> gpiod_request(desc, label)
                |
                +-> gpiod_request_commit(desc, label)
                      |
                      +-> test_and_set_bit(GPIOD_FLAG_REQUESTED, &desc->flags)
                      +-> rcu_assign_pointer(desc->label, new_label)
                      +-> notify line_state_notifier
                      |
                      +-> gpiod_configure_flags(desc, con_id, lflags, GPIOD_OUT_LOW)
                            |
                            +-> gpiod_direction_output_nonotify(desc, 0)
                            |     |
                            |     +-> CLASS(gpio_chip_guard, guard)(desc)
                            |           | srcu_read_lock(&gdev->srcu)
                            |           +-> gc->direction_output(gc, offset, 0)
                            |                 -> hardware register write
                            |
                            +-> set GPIOD_FLAG_IS_OUT in desc->flags
```

#### قراءة قيمة GPIO واحدة

```
gpiod_get_value(desc)
  |
  +-> CLASS(gpio_chip_guard, guard)(desc)
  |     srcu_read_lock(&gdev->srcu)
  |     gc = srcu_dereference(gdev->chip, &gdev->srcu)
  |
  +-> if gc == NULL -> return -ENODEV
  |
  +-> gc->get(gc, offset)
        -> read hardware register
        -> return 0 or 1
  |
  +-> SRCU read unlock (automatic at scope exit via guard destructor)
```

#### قراءة array من GPIOs — fast path vs slow path

```
gpiod_get_array_value_complex(raw, can_sleep, size, desc_array, array_info, value_bitmap)
  |
  +-> if array_info != NULL && all GPIOs on same chip:
  |     FAST PATH
  |     +-> gc->get_multiple(gc, array_info->get_mask, value_bitmap)
  |           -> read all bits in one hardware access
  |           -> XOR with invert_mask (active-low correction)
  |
  +-> if GPIOs span multiple chips:
        SLOW PATH (loop per desc)
        +-> for each desc: gc->get(gc, offset)
```

#### إصدار حدث line state

```
gpiod_direction_output(desc, value)    [or direction_input, request, free]
  |
  +-> gpiod_direction_output_nonotify(desc, value)   [actual hw work]
        -> gc->direction_output(gc, offset, value)
  |
  +-> gpiod_line_state_notify(desc, GPIOLINE_CHANGED_CONFIG)
        |
        +-> read_lock(&gdev->line_state_lock)
        +-> raw_notifier_call_chain(&gdev->line_state_notifier, action, desc)
        +-> read_unlock(...)
        |
        +-> queue_work(gdev->line_state_wq, ...)
              -> runs notifier in process context (safe for blocking ops)
```

---

### استراتيجية الـ Locking

#### الـ Locks الموجودة

| الـ Lock | النوع | بيحمي إيه؟ |
|----------|-------|------------|
| `gdev->srcu` | `struct srcu_struct` | الـ pointer لـ `gdev->chip` — يضمن الـ chip مش بيتشال أثناء الاستخدام |
| `gdev->desc_srcu` | `struct srcu_struct` | الـ descriptors المكشوفة لـ userspace |
| `gdev->line_state_lock` | `rwlock_t` | قائمة الـ `line_state_notifier` |
| `desc->label` (implicit) | `rcu_head` في `gpio_desc_label` | الـ label string — قراءة بـ RCU، تعديل بـ `rcu_assign_pointer` |

#### ترتيب الـ Locking وقواعده

```
Rule 1 — accessing the chip:
  1. srcu_read_lock(&gdev->srcu)          [via gpio_chip_guard]
  2. gc = srcu_dereference(gdev->chip, ...)
  3. if gc == NULL -> bail out (chip was removed)
  4. call gc->ops()
  5. srcu_read_unlock() at scope exit

Rule 2 — line state notifications:
  writers (register notifier): write_lock(&gdev->line_state_lock)
  readers (fire notification):  read_lock(&gdev->line_state_lock)
  NEVER hold srcu lock while calling notifiers (may sleep)

Rule 3 — label update:
  rcu_assign_pointer(desc->label, new_label)
  kfree_rcu(old_label, rh)
  readers: rcu_dereference(desc->label) inside srcu or rcu read section
```

#### ليه SRCU وليس spinlock او classic RCU؟

**الـ** `srcu` (Sleepable RCU) هنا لأن:
- الـ `gc->get()` و `gc->set()` ممكن **ينام** (مثلاً I2C expander over I2C bus) — وده ممنوع مع spinlock أو classic RCU.
- عند `gpiochip_remove()` بيعمل `synchronize_srcu()` وده بيضمن إن مافيش حد شايل reference على الـ chip لما تتشال.

```
Timeline during gpiochip_remove():

  Thread A (remove):                   Thread B (reader):
  srcu_assign_pointer(chip, NULL)
                                        srcu_read_lock(&gdev->srcu) [idx]
  synchronize_srcu(...)  <-- BLOCKS     gc = srcu_dereference(...) -> NULL
                                        return -ENODEV
                                        srcu_read_unlock([idx])
  synchronize_srcu returns
  kfree(chip_memory)
```

**الـ** `gpio_chip_guard` بيغلّف الـ SRCU lock في pattern RAII نظيف — الـ lock بياخد في البداية والـ unlock بييجي تلقائي لما الـ guard يخرج من الـ scope.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Functions & APIs جدول

| Function / Macro | النوع | الغرض الرئيسي |
|---|---|---|
| `to_gpio_device()` | static inline | تحويل `struct device *` → `struct gpio_device *` |
| `gpiod_request()` | function | طلب امتلاك GPIO descriptor |
| `gpiod_request_commit()` | function | تنفيذ الطلب الفعلي (تُستدعى من `gpiod_request`) |
| `gpiod_request_user()` | static inline | wrapper فوق `gpiod_request` للـ userspace path |
| `gpiod_free()` | function | تحرير GPIO descriptor مع notifier |
| `gpiod_free_commit()` | function | تنفيذ التحرير الفعلي |
| `gpiod_find_and_request()` | function | بحث عن GPIO في firmware ثم طلبه |
| `gpiod_get_array_value_complex()` | function | قراءة مصفوفة GPIOs (fast-path أو slow-path) |
| `gpiod_set_array_value_complex()` | function | كتابة مصفوفة GPIOs |
| `gpiod_set_transitory()` | function | ضبط خاصية transitory على descriptor |
| `gpiod_line_state_notify()` | function | إطلاق notifier عند تغيُّر حالة line |
| `gpiod_direction_output_nonotify()` | function | ضبط اتجاه output بدون إطلاق notifier |
| `gpiod_direction_input_nonotify()` | function | ضبط اتجاه input بدون إطلاق notifier |
| `gpio_do_set_config()` | function | إرسال pinconf config لـ chip |
| `gpiod_configure_flags()` | function | تطبيق `lflags` + `dflags` على descriptor |
| `gpio_set_debounce_timeout()` | function | ضبط debounce period بالـ microseconds |
| `gpiod_hog()` | function | hog (حجز دائم) لـ GPIO line من kernel |
| `gpiochip_get_ngpios()` | function | قراءة عدد GPIOs من chip أو device property |
| `gpiochip_get_desc()` | function | الحصول على `gpio_desc` من hwnum |
| `gpiod_get_label()` | function | قراءة consumer label بشكل RCU-safe |
| `gpiod_not_found()` | macro | فحص إذا كان descriptor يمثل -ENOENT |
| `for_each_gpio_desc()` | macro | iteration على كل descriptors لـ chip |
| `for_each_gpio_desc_with_flag()` | macro | iteration مع فلترة بـ flag معين |
| `for_each_gpio_property_name()` | macro | iteration على suffixes لـ DT/ACPI lookup |
| `gpio_chip_guard` CLASS | RAII class | SRCU read lock على `gpio_device->srcu` |
| `gpiod_err/warn/dbg()` | macros | logging مع prefix يحتوي رقم GPIO + label |
| `gpiochip_err/warn/info/dbg()` | macros | logging عبر `dev_*` API لـ chip device |

---

### المجموعة 1: Type Helpers & Conversion

هذه الـ helpers تربط بين الـ `struct device` generic وبين الـ `struct gpio_device` الداخلي.

---

#### `to_gpio_device()`

```c
static inline struct gpio_device *to_gpio_device(struct device *dev)
{
    return container_of(dev, struct gpio_device, dev);
}
```

**ما تفعله:** تستخدم `container_of` للوصول للـ `gpio_device` الذي يحتوي على هذا الـ `struct device`. الاستخدام الأساسي في الـ device model callbacks حين يصل الكود للـ `struct device *` فقط.

| Parameter | الوصف |
|---|---|
| `dev` | مؤشر لـ `struct device` المضمَّن داخل `gpio_device` |

**Return:** مؤشر للـ `gpio_device` الأب.

**ملاحظة:** لا يوجد locking — الـ caller مسؤول عن ضمان أن الـ `gpio_device` لا يزال حياً (مثلاً بـ reference count عبر `gpio_device_get()`).

---

#### `gpiod_not_found()` macro

```c
#define gpiod_not_found(desc)  (IS_ERR(desc) && PTR_ERR(desc) == -ENOENT)
```

**ما يفعله:** يفرق بين "GPIO غير موجود في firmware" (`-ENOENT`) وبين أي خطأ حقيقي. يُستخدم بشكل كثيف في كود البحث عن GPIOs من DT/ACPI لتمييز "optional GPIO غير موجود" عن "خطأ فعلي".

**مثال:**
```c
desc = gpiod_find_and_request(...);
if (gpiod_not_found(desc)) {
    desc = NULL; /* اختياري وغير موجود — مقبول */
} else if (IS_ERR(desc)) {
    return PTR_ERR(desc); /* خطأ فعلي */
}
```

---

### المجموعة 2: SRCU Guard — `gpio_chip_guard`

الـ `gpio_chip` مؤقت — قد يُزال أثناء التشغيل. الـ `gpio_device` بالمقابل قد يبقى حياً طالما هناك userspace references. هذا الـ guard يضمن الوصول الآمن للـ `gpio_chip` عبر SRCU.

```c
DEFINE_CLASS(gpio_chip_guard,
     struct gpio_chip_guard,
     srcu_read_unlock(&_T.gdev->srcu, _T.idx),   /* cleanup */
     ({
        struct gpio_chip_guard _guard;
        _guard.gdev = desc->gdev;
        _guard.idx = srcu_read_lock(&_guard.gdev->srcu);
        _guard.gc = srcu_dereference(_guard.gdev->chip,
                         &_guard.gdev->srcu);
        _guard;
     }),
     struct gpio_desc *desc)
```

**ما يفعله:** عند `CLASS(gpio_chip_guard, guard)(desc)` يقوم بـ:
1. أخذ SRCU read lock على `gdev->srcu`
2. قراءة `gdev->chip` بشكل atomic عبر `srcu_dereference`
3. عند خروج الـ scope: تحرير الـ SRCU lock تلقائياً (cleanup attribute)

**الـ struct المُعرَّف:**
```c
struct gpio_chip_guard {
    struct gpio_device *gdev;  /* الـ gpio_device الأب */
    struct gpio_chip *gc;      /* الـ chip pointer (قد يكون NULL لو أُزيل) */
    int idx;                   /* SRCU read index */
};
```

**لماذا SRCU وليس spinlock:** الـ SRCU يتيح read-side zero-cost في fast path، والـ write-side (إزالة الـ chip) ينتظر grace period بعد إزالة الـ pointer. هذا يضمن أن أي `gc` pointer مقروء بـ `srcu_dereference` لن يُحرَّر أثناء الاستخدام.

**ملاحظة حرجة:** بعد أخذ الـ guard يجب دائماً فحص `guard.gc != NULL`:
```c
CLASS(gpio_chip_guard, guard)(desc);
if (!guard.gc)
    return -ENODEV;  /* الـ chip أُزيل */
/* guard.gc آمن للاستخدام هنا */
/* srcu_read_unlock تلقائي عند نهاية scope */
```

---

### المجموعة 3: Descriptor Acquisition — طلب الـ GPIO

هذه المجموعة تغطي مسار طلب امتلاك GPIO line بالكامل.

---

#### `gpiod_request()`

```c
int gpiod_request(struct gpio_desc *desc, const char *label);
```

**ما تفعله:** الـ entry point الرئيسي لطلب GPIO. تتحقق من صحة الـ descriptor، تأخذ `gpio_chip_guard` (SRCU)، ثم تستدعي `gpiod_request_commit()` للتنفيذ الفعلي. تُطلق `gpiod_line_state_notify()` بعد النجاح.

| Parameter | الوصف |
|---|---|
| `desc` | الـ GPIO descriptor المراد طلبه |
| `label` | اسم المستهلك (consumer name)، يُحفظ في `desc->label` |

**Return:** `0` عند النجاح، أو error code سالب (`-EBUSY` لو محجوز، `-EPROBE_DEFER` لو الـ chip غير جاهز).

**Key details:**
- لا يجب استدعاؤها من atomic context إذا كان `chip->can_sleep = true`
- الـ `GPIOD_FLAG_REQUESTED` bit يُضبط atomically في `gpiod_request_commit()`
- تُطلق notification بعد النجاح من process context

---

#### `gpiod_request_commit()`

```c
int gpiod_request_commit(struct gpio_desc *desc, const char *label);
```

**ما تفعله:** التنفيذ الداخلي لطلب الـ GPIO. تضبط `GPIOD_FLAG_REQUESTED` في `desc->flags` بشكل atomic، تستدعي `gc->request()` callback (إن وُجد)، وتحفظ الـ label عبر RCU-safe allocation.

**Pseudocode:**
```
gpiod_request_commit(desc, label):
    if test_and_set_bit(GPIOD_FLAG_REQUESTED, &desc->flags):
        return -EBUSY           /* محجوز بالفعل */

    if gc->request:
        ret = gc->request(gc, gpio_chip_hwgpio(desc))
        if ret:
            clear_bit(GPIOD_FLAG_REQUESTED, &desc->flags)
            return ret          /* rollback */

    new_label = kzalloc(sizeof(*label) + strlen(label) + 1)
    rcu_assign_pointer(desc->label, new_label)
    return 0
```

**Who calls it:** `gpiod_request()` فقط، داخل SRCU read-side critical section.

---

#### `gpiod_request_user()`

```c
static inline int gpiod_request_user(struct gpio_desc *desc, const char *label)
{
    int ret;
    ret = gpiod_request(desc, label);
    if (ret == -EPROBE_DEFER)
        ret = -ENODEV;
    return ret;
}
```

**ما تفعله:** wrapper يُحوِّل `-EPROBE_DEFER` إلى `-ENODEV` لأن الـ userspace (character device) لا يفهم semantics الـ deferred probe. منطقي — الـ userspace يستدعي `ioctl()` مباشرة بدون probe mechanism.

---

#### `gpiod_find_and_request()`

```c
struct gpio_desc *gpiod_find_and_request(struct device *consumer,
                     struct fwnode_handle *fwnode,
                     const char *con_id,
                     unsigned int idx,
                     enum gpiod_flags flags,
                     const char *label,
                     bool platform_lookup_allowed);
```

**ما تفعله:** تجمع عمليتين: البحث عن GPIO في firmware node أو platform lookup، ثم طلبه وتكوينه مباشرة. هي الـ function الأساسية التي تستدعيها `gpiod_get()` وأخواتها.

| Parameter | الوصف |
|---|---|
| `consumer` | الـ device المستهلك للـ GPIO |
| `fwnode` | firmware node للبحث فيه (DT node أو ACPI node) |
| `con_id` | اسم الـ connection (مثلاً `"reset"`, `"enable"`) |
| `idx` | الـ index داخل الـ property لو في مصفوفة |
| `flags` | `enum gpiod_flags` لضبط الاتجاه والقيمة الابتدائية |
| `label` | consumer label للـ descriptor |
| `platform_lookup_allowed` | يسمح بالبحث في platform GPIO lookup tables |

**Return:** مؤشر صالح لـ `gpio_desc`، أو `ERR_PTR()` مع error code.

**Pseudocode:**
```
gpiod_find_and_request(consumer, fwnode, con_id, idx, flags, label, platform):
    desc = gpiod_find(consumer, fwnode, con_id, idx, &lflags, &dflags)
    if not found and platform:
        desc = platform_gpio_lookup(consumer, con_id, idx)
    if IS_ERR(desc):
        return desc

    ret = gpiod_request(desc, label)
    if ret:
        return ERR_PTR(ret)

    ret = gpiod_configure_flags(desc, con_id, lflags, flags | dflags)
    if ret:
        gpiod_free(desc)        /* rollback */
        return ERR_PTR(ret)

    return desc
```

---

### المجموعة 4: Descriptor Release — تحرير الـ GPIO

---

#### `gpiod_free()`

```c
void gpiod_free(struct gpio_desc *desc);
```

**ما تفعله:** تحرر GPIO descriptor مع إطلاق notifier لإبلاغ المشتركين (مثل GPIO character device). تستدعي `gpiod_free_commit()` للتنفيذ الفعلي، ثم `gpiod_line_state_notify()`.

| Parameter | الوصف |
|---|---|
| `desc` | الـ descriptor المراد تحريره |

**Key details:**
- يجب أن تكون الـ GPIO قد طُلبت مسبقاً (`GPIOD_FLAG_REQUESTED` مضبوط)
- تُصدر `WARN_ON` إذا كانت الـ GPIO لا تزال used as IRQ
- تُرسل `GPIOLINE_CHANGED_RELEASED` event للـ character device

---

#### `gpiod_free_commit()`

```c
void gpiod_free_commit(struct gpio_desc *desc);
```

**ما تفعله:** التنفيذ الداخلي. تستدعي `gc->free()` callback (إن وُجد)، تمسح `GPIOD_FLAG_REQUESTED` وكل الـ flags المرتبطة، وتُحرر الـ label memory بـ `kfree_rcu()`.

**Key details:**
- الـ label تُحرَّر بعد RCU grace period (`kfree_rcu`) لضمان سلامة الـ readers الحاليين
- **Who calls it:** `gpiod_free()` فقط، داخل SRCU read-side critical section

---

### المجموعة 5: Array Value Operations — fast path

من أكثر الـ functions تعقيداً. تدعم fast-path عبر `struct gpio_array` وسlow-path generic.

---

#### `gpiod_get_array_value_complex()`

```c
int gpiod_get_array_value_complex(bool raw, bool can_sleep,
                  unsigned int array_size,
                  struct gpio_desc **desc_array,
                  struct gpio_array *array_info,
                  unsigned long *value_bitmap);
```

**ما تفعله:** تقرأ قيم مصفوفة GPIO lines. إذا كانت جميع الـ GPIOs من نفس الـ chip وتدعم `get_multiple`، تستدعي `gc->get_multiple()` مرة واحدة كـ fast-path. وإلا تقرأ كل line منفردة كـ slow-path.

| Parameter | الوصف |
|---|---|
| `raw` | إذا `true`: تتجاهل `ACTIVE_LOW` polarity inversion |
| `can_sleep` | إذا `true`: يُسمح بـ sleeping (لـ I2C/SPI expanders) |
| `array_size` | عدد الـ GPIOs في المصفوفة |
| `desc_array` | مصفوفة مؤشرات `gpio_desc` |
| `array_info` | معلومات fast-path (من `gpiod_get_array()`)، قد يكون NULL |
| `value_bitmap` | bitmap الإخراج — bit N يمثل قيمة `desc_array[N]` |

**Return:** `0` عند النجاح، error code سالب عند الفشل.

**Fast-path vs Slow-path:**
```
إذا array_info->get_mask يغطي كل lines من نفس chip وchip يدعم get_multiple:
    [FAST] gc->get_multiple(gc, get_mask, bits)
           طبِّق invert_mask للـ ACTIVE_LOW
           انسخ النتيجة للـ value_bitmap
وإلا:
    [SLOW] for كل desc في array:
               value_bitmap[i] = gc->get(gc, hwnum)
               إذا ACTIVE_LOW: اعكس القيمة
```

**Locking:** تستخدم `gpio_chip_guard` (SRCU) لكل chip. في حال `can_sleep=false` تتحقق بـ `WARN_ON(chip->can_sleep)`.

---

#### `gpiod_set_array_value_complex()`

```c
int gpiod_set_array_value_complex(bool raw, bool can_sleep,
                  unsigned int array_size,
                  struct gpio_desc **desc_array,
                  struct gpio_array *array_info,
                  unsigned long *value_bitmap);
```

**ما تفعله:** تكتب قيم مصفوفة GPIO. منطقها مماثل لـ `gpiod_get_array_value_complex()` لكن باتجاه الكتابة. تطبق ACTIVE_LOW inversion على الـ value_bitmap قبل الإرسال للـ chip، وتدعم `gc->set_multiple()` كـ fast-path.

**ملاحظة خاصة بـ open-drain:** للـ lines من نوع open-drain، إذا أردت ضبطها high لا تستخدم `set()` مباشرة بل تبدّل الاتجاه لـ input (هذا السلوك مُضمَّن في كود التنفيذ).

---

### المجموعة 6: Direction & State Control

---

#### `gpiod_direction_output_nonotify()`

```c
int gpiod_direction_output_nonotify(struct gpio_desc *desc, int value);
```

**ما تفعله:** تضبط GPIO كـ output بقيمة ابتدائية، بدون إطلاق `line_state_notifier`. الـ "nonotify" variant تُستخدم داخلياً حين لا نريد إشعار الـ character device (مثلاً أثناء initialization أو hog setup).

| Parameter | الوصف |
|---|---|
| `desc` | الـ GPIO descriptor |
| `value` | القيمة الابتدائية (0 أو 1)، logical (تأخذ ACTIVE_LOW بعين الاعتبار) |

**Return:** `0` نجاح، أو error code.

**Key details:**
- تضبط `GPIOD_FLAG_IS_OUT` في `desc->flags` عند النجاح
- تستدعي `gc->direction_output(gc, hwnum, value)` أو تُحاكي السلوك لـ open-drain
- الـ caller يُطلق الـ notifier بنفسه بعدها لو احتاج

---

#### `gpiod_direction_input_nonotify()`

```c
int gpiod_direction_input_nonotify(struct gpio_desc *desc);
```

**ما تفعله:** تضبط GPIO كـ input بدون notifier. تمسح `GPIOD_FLAG_IS_OUT` وتستدعي `gc->direction_input()`.

**من يستدعيها:** الـ public API `gpiod_direction_input()` تستدعيها ثم تُطلق الـ notifier بعدها.

---

#### `gpiod_set_transitory()`

```c
int gpiod_set_transitory(struct gpio_desc *desc, bool transitory);
```

**ما تفعله:** تضبط أو تمسح الـ `GPIOD_FLAG_TRANSITORY` على الـ descriptor، ثم تُرسل `PIN_CONFIG_PERSIST_STATE` config للـ chip عبر `gpio_do_set_config()`. الـ transitory GPIOs قد تفقد حالتها في sleep/reset.

| Parameter | الوصف |
|---|---|
| `desc` | الـ GPIO descriptor |
| `transitory` | `true` = قد يفقد الحالة، `false` = persistent |

**Return:** `0` نجاح، أو `-ENOTSUPP` لو الـ chip لا يدعم هذا الـ config (وهو acceptable).

---

### المجموعة 7: Notification System

---

#### `gpiod_line_state_notify()`

```c
void gpiod_line_state_notify(struct gpio_desc *desc, unsigned long action);
```

**ما تفعله:** تُطلق `raw_notifier_call_chain()` على `gdev->line_state_notifier` لإبلاغ المشتركين (أساساً الـ GPIO character device) بأن حالة line تغيرت. تُستخدم عند request/free/reconfigure.

| Parameter | الوصف |
|---|---|
| `desc` | الـ GPIO descriptor الذي تغير |
| `action` | نوع التغيير (`GPIOLINE_CHANGED_REQUESTED`, `GPIOLINE_CHANGED_RELEASED`, `GPIOLINE_CHANGED_CONFIG`) |

**Locking:** تأخذ `gdev->line_state_lock` (rwlock) قبل استدعاء الـ notifier chain.

**Context:** يجب أن تُستدعى من process context لأن الـ notifier handlers قد ينامون. في السياقات التي لا تسمح بالنوم تُجدوَل عبر `gdev->line_state_wq` workqueue.

---

### المجموعة 8: Configuration Helpers

---

#### `gpio_do_set_config()`

```c
int gpio_do_set_config(struct gpio_desc *desc, unsigned long config);
```

**ما تفعله:** ترسل `pinconf` config مُعبَّأة بصيغة `PIN_CONF_PACKED` للـ chip عبر `gc->set_config()` callback. هي الـ lower-level helper التي تستخدمها `gpiod_set_debounce()` و`gpiod_set_transitory()` وغيرها.

| Parameter | الوصف |
|---|---|
| `desc` | الـ GPIO descriptor |
| `config` | قيمة config مُعبَّأة بـ `PIN_CONF_PACKED(param, arg)` macro |

**Return:** `0` نجاح، `-ENOTSUPP` لو الـ chip لا يدعم `set_config`، أو error آخر.

**Key details:** تستخدم `CLASS(gpio_chip_guard, guard)(desc)` داخلياً للوصول الآمن للـ chip.

---

#### `gpiod_configure_flags()`

```c
int gpiod_configure_flags(struct gpio_desc *desc, const char *con_id,
        unsigned long lflags, enum gpiod_flags dflags);
```

**ما تفعله:** تُطبق مجموعة كاملة من الـ flags على descriptor تم طلبه حديثاً. تُحوِّل `enum gpiod_flags` (الـ public API) و`gpio_lookup_flags` (من DT/ACPI) إلى hardware configuration.

| Parameter | الوصف |
|---|---|
| `desc` | الـ descriptor بعد `gpiod_request()` |
| `con_id` | اسم الـ connection (للـ logging) |
| `lflags` | `gpio_lookup_flags`: ACTIVE_LOW, OPEN_DRAIN, PULL_UP/DOWN |
| `dflags` | `gpiod_flags`: اتجاه output/input والقيمة الابتدائية |

**Pseudocode:**
```
gpiod_configure_flags(desc, con_id, lflags, dflags):
    if lflags & GPIO_ACTIVE_LOW:   set_bit(GPIOD_FLAG_ACTIVE_LOW, &desc->flags)
    if lflags & GPIO_OPEN_DRAIN:   set_bit(GPIOD_FLAG_OPEN_DRAIN, &desc->flags)
    if lflags & GPIO_OPEN_SOURCE:  set_bit(GPIOD_FLAG_OPEN_SOURCE, &desc->flags)
    if lflags & GPIO_PULL_UP:      gpio_do_set_config(desc, BIAS_PULL_UP)
    if lflags & GPIO_PULL_DOWN:    gpio_do_set_config(desc, BIAS_PULL_DOWN)
    if lflags & GPIO_TRANSITORY:   gpiod_set_transitory(desc, true)

    if dflags & GPIOD_FLAGS_BIT_DIR_SET:
        if dflags & GPIOD_FLAGS_BIT_DIR_OUT:
            value = !!(dflags & GPIOD_FLAGS_BIT_DIR_VAL)
            return gpiod_direction_output_nonotify(desc, value)
        else:
            return gpiod_direction_input_nonotify(desc)
    return 0
```

---

#### `gpio_set_debounce_timeout()`

```c
int gpio_set_debounce_timeout(struct gpio_desc *desc, unsigned int debounce);
```

**ما تفعله:** تضبط debounce period بالـ microseconds. تحفظ القيمة في `desc->debounce_period_us` (متاحة فقط بـ `CONFIG_GPIO_CDEV` للـ software debounce)، ثم تستدعي `gpio_do_set_config()` مع `PIN_CONFIG_INPUT_DEBOUNCE`.

| Parameter | الوصف |
|---|---|
| `desc` | الـ GPIO descriptor (يجب أن يكون input) |
| `debounce` | المدة بالـ microseconds |

**Return:** `0` نجاح أو إذا الـ chip لا يدعم hardware debounce (مقبول — software debounce يعمل)، أو error سالب عند فشل حقيقي.

---

### المجموعة 9: GPIO Hogging

---

#### `gpiod_hog()`

```c
int gpiod_hog(struct gpio_desc *desc, const char *name,
        unsigned long lflags, enum gpiod_flags dflags);
```

**ما تفعله:** تحجز GPIO line بشكل دائم من الـ kernel (ناتج عن DT `gpio-hog` property). تطلب الـ descriptor باسم `name`، تضبط `GPIOD_FLAG_IS_HOGGED`، وتستدعي `gpiod_configure_flags()`.

| Parameter | الوصف |
|---|---|
| `desc` | الـ descriptor المراد حجزه |
| `name` | اسم وصفي للـ hog (يظهر في `/sys` وـ debugfs) |
| `lflags` | lookup flags من DT |
| `dflags` | direction + initial value flags |

**Return:** `0` نجاح، أو error.

**Key details:**
- الـ hogged GPIO لا يمكن تحريره بـ `gpiod_put()` من أي driver آخر
- `GPIOD_FLAG_IS_HOGGED` يُضبط بعد نجاح `gpiod_request()`
- يُستدعى من `gpiochip_add_data()` path أثناء تسجيل الـ chip

---

### المجموعة 10: Chip Introspection Helpers

---

#### `gpiochip_get_ngpios()`

```c
int gpiochip_get_ngpios(struct gpio_chip *gc, struct device *dev);
```

**ما تفعله:** تحدد عدد الـ GPIO lines للـ chip. تقرأ أولاً من `gc->ngpio`، فإن كان صفراً تقرأ `ngpios` device property من firmware node. تتحقق أيضاً من أن القيمة لا تتجاوز `U16_MAX`.

| Parameter | الوصف |
|---|---|
| `gc` | الـ GPIO chip |
| `dev` | الـ device لقراءة firmware properties منه |

**Return:** عدد الـ GPIOs عند النجاح، أو error سالب.

---

#### `gpiochip_get_desc()`

```c
struct gpio_desc *gpiochip_get_desc(struct gpio_chip *gc, unsigned int hwnum);
```

**ما تفعله:** تُعيد الـ `gpio_desc` المقابل لـ hardware GPIO offset داخل الـ chip. Direct array access: `&gc->gpiodev->descs[hwnum]`.

| Parameter | الوصف |
|---|---|
| `gc` | الـ GPIO chip |
| `hwnum` | رقم الـ GPIO داخل الـ chip (0 حتى ngpio-1) |

**Return:** `struct gpio_desc *` صالح، أو `ERR_PTR(-EINVAL)` لو خارج النطاق.

**من يستدعيها:** `for_each_gpio_desc()` macro، وأي كود يحتاج تحويل hwnum → desc.

---

#### `gpiod_get_label()`

```c
const char *gpiod_get_label(struct gpio_desc *desc);
```

**ما تفعله:** تقرأ consumer label من `desc->label` بطريقة RCU-safe. يجب استدعاؤها من داخل `srcu_read_lock()` أو `scoped_guard(srcu, &desc->gdev->desc_srcu)`.

**Return:** مؤشر للـ string أو `NULL` لو لم يكن هناك label.

**Locking:** الـ label تُخزَّن كـ `struct gpio_desc_label __rcu *` مع `rcu_head` للـ deferred freeing، لذا يجب حماية القراءة بـ SRCU read lock لضمان عدم تحرير الـ label أثناء القراءة.

---

### المجموعة 11: Iteration Macros

---

#### `for_each_gpio_desc()`

```c
#define for_each_gpio_desc(gc, desc)                         \
    for (unsigned int __i = 0;                               \
         __i < gc->ngpio && (desc = gpiochip_get_desc(gc, __i)); \
         __i++)
```

**ما يفعله:** يُكرر على جميع الـ descriptors في chip واحد. يُستخدم في gpiolib internals لتطبيق عمليات batch على كل الـ lines.

**مثال استخدام:**
```c
struct gpio_desc *desc;
for_each_gpio_desc(gc, desc) {
    if (test_bit(GPIOD_FLAG_REQUESTED, &desc->flags))
        pr_info("gpio-%d in use\n", desc_to_gpio(desc));
}
```

---

#### `for_each_gpio_desc_with_flag()`

```c
#define for_each_gpio_desc_with_flag(gc, desc, flag)    \
    for_each_gpio_desc(gc, desc)                        \
        if (!test_bit(flag, &desc->flags)) {} else
```

**ما يفعله:** نفس `for_each_gpio_desc()` لكن يفلتر فقط الـ descriptors التي لديها `flag` معين مضبوط.

```c
/* مثال: التكرار على GPIOs المستخدمة كـ IRQ فقط */
for_each_gpio_desc_with_flag(gc, desc, GPIOD_FLAG_USED_AS_IRQ) {
    /* ... */
}
```

---

#### `for_each_gpio_property_name()`

```c
#define for_each_gpio_property_name(propname, con_id)
```

**ما يفعله:** يُكرر على جميع الـ GPIO suffixes المعيارية (مثلاً `"gpio"`, `"gpios"`) لبناء property names. لو `con_id = "reset"` ينتج `"reset-gpio"` ثم `"reset-gpios"`.

```c
char propname[32];
for_each_gpio_property_name(propname, "enable") {
    /* propname = "enable-gpio", "enable-gpios", ... */
    desc = fwnode_get_named_gpiod(fwnode, propname, 0, flags, label);
    if (!IS_ERR(desc))
        break;
}
```

---

### المجموعة 12: Logging Macros

#### مع GPIO descriptor prefix

```c
/* تُنشئ: pr_err("gpio-N (label): ...") */
#define gpiod_err(desc, fmt, ...)   __gpiod_pr(err,   desc, fmt, ##__VA_ARGS__)
#define gpiod_warn(desc, fmt, ...)  __gpiod_pr(warn,  desc, fmt, ##__VA_ARGS__)
#define gpiod_dbg(desc, fmt, ...)   __gpiod_pr(debug, desc, fmt, ##__VA_ARGS__)
```

**ما تفعله:** تُطبع رسائل log تتضمن رقم الـ GPIO (`desc_to_gpio(desc)`) والـ label كـ prefix. الـ label يُقرأ داخل `scoped_guard(srcu, &desc->gdev->desc_srcu)` لضمان عدم القراءة بعد التحرير.

**مثال:**
```
[  12.345] gpio-42 (reset): direction output failed: -5
```

#### مع chip prefix

```c
/* تُنشئ: dev_err(&gc->gpiodev->dev, "(chip_label): ...") */
#define gpiochip_err(gc, fmt, ...)  __gpiochip_pr(err,  gc, fmt, ##__VA_ARGS__)
#define gpiochip_warn(gc, fmt, ...) __gpiochip_pr(warn, gc, fmt, ##__VA_ARGS__)
#define gpiochip_info(gc, fmt, ...) __gpiochip_pr(info, gc, fmt, ##__VA_ARGS__)
#define gpiochip_dbg(gc, fmt, ...)  __gpiochip_pr(dbg,  gc, fmt, ##__VA_ARGS__)
```

**ما تفعله:** تُرسل رسائل عبر `dev_*` API مرتبطة بالـ `gpio_device->dev`، مما يُضيف الـ device path تلقائياً في الـ kernel log.

---

### مخطط التدفق الكامل — GPIO Request Flow

```
gpiod_get(dev, "reset", GPIOD_OUT_LOW)
    │
    ▼
gpiod_find_and_request(dev, fwnode, "reset", 0, GPIOD_OUT_LOW, "reset", true)
    │
    ├─[lookup]─▶ for_each_gpio_property_name(propname, "reset"):
    │               try "reset-gpio" → "reset-gpios" → ...
    │               desc = fwnode_get_named_gpiod(fwnode, propname, ...)
    │
    ├─[request]─▶ gpiod_request(desc, "reset")
    │                 │
    │                 ▼
    │             CLASS(gpio_chip_guard, guard)(desc)   ← SRCU lock
    │             gpiod_request_commit(desc, "reset")
    │                 ├─ test_and_set_bit(GPIOD_FLAG_REQUESTED)
    │                 ├─ gc->request(gc, hwnum)          ← chip callback
    │                 └─ rcu_assign_pointer(desc->label, ...)
    │             srcu_read_unlock(...)                  ← auto via guard
    │             gpiod_line_state_notify(desc, REQUESTED)
    │
    └─[configure]─▶ gpiod_configure_flags(desc, "reset", lflags, GPIOD_OUT_LOW)
                        ├─ set ACTIVE_LOW / OPEN_DRAIN flags if needed
                        └─ gpiod_direction_output_nonotify(desc, 0)
                               ├─ CLASS(gpio_chip_guard, guard)(desc)
                               ├─ gc->direction_output(gc, hwnum, 0)
                               └─ set_bit(GPIOD_FLAG_IS_OUT, &desc->flags)
```

---

### جدول الـ Flags الأساسية (`gpio_desc->flags`)

| Flag | Bit | المعنى |
|---|---|---|
| `GPIOD_FLAG_REQUESTED` | 0 | Line محجوزة بـ consumer |
| `GPIOD_FLAG_IS_OUT` | 1 | Line في وضع output |
| `GPIOD_FLAG_EXPORT` | 2 | مُصدَّرة لـ userspace |
| `GPIOD_FLAG_SYSFS` | 3 | مُصدَّرة عبر `/sys/class/gpio` |
| `GPIOD_FLAG_ACTIVE_LOW` | 6 | Polarity معكوسة |
| `GPIOD_FLAG_OPEN_DRAIN` | 7 | Open-drain output |
| `GPIOD_FLAG_OPEN_SOURCE` | 8 | Open-source output |
| `GPIOD_FLAG_USED_AS_IRQ` | 9 | Line مرتبطة بـ IRQ |
| `GPIOD_FLAG_IRQ_IS_ENABLED` | 10 | الـ IRQ مفعَّل |
| `GPIOD_FLAG_IS_HOGGED` | 11 | محجوزة دائماً من kernel |
| `GPIOD_FLAG_TRANSITORY` | 12 | قد تفقد حالتها في sleep |
| `GPIOD_FLAG_PULL_UP` | 13 | Pull-up مفعَّل |
| `GPIOD_FLAG_PULL_DOWN` | 14 | Pull-down مفعَّل |
| `GPIOD_FLAG_BIAS_DISABLE` | 15 | Pull مُعطَّل |
| `GPIOD_FLAG_EDGE_RISING` | 16 | CDEV يرصد rising edge |
| `GPIOD_FLAG_EDGE_FALLING` | 17 | CDEV يرصد falling edge |
| `GPIOD_FLAG_EVENT_CLOCK_REALTIME` | 18 | CDEV يُبلغ بـ REALTIME timestamps |
| `GPIOD_FLAG_EVENT_CLOCK_HTE` | 19 | CDEV يُبلغ بـ hardware timestamps |
| `GPIOD_FLAG_SHARED` | 20 | Line مشتركة بين consumers |
| `GPIOD_FLAG_SHARED_PROXY` | 21 | Virtual proxy لـ shared pin |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs entries

**الـ** `gpiolib` بيعرض معلومات تفصيلية جوه `/sys/kernel/debug/gpio`:

```bash
# اقرا حالة كل GPIO chips والـ descriptors
cat /sys/kernel/debug/gpio
```

**مثال على الـ output:**

```
gpiochip0: GPIOs 0-31, parent: platform/fe200000.gpio, pinctrl-bcm2835:
 gpio-2   (                    |sda1                ) in  hi    IRQ ACTIVE LOW
 gpio-3   (                    |scl1                ) in  hi    IRQ ACTIVE LOW
 gpio-17  (                    |reset               ) out lo
 gpio-27  (                    |led0                ) out hi    ACTIVE LOW
```

**تفسير الـ output:**

| عمود | معناه |
|------|--------|
| `gpio-N` | رقم الـ GPIO في numberspace العالمي (deprecated) |
| `(label)` | اسم الـ consumer أو الـ function |
| `in/out` | الاتجاه الحالي |
| `hi/lo` | القيمة الحالية (بعد تطبيق active-low) |
| `IRQ` | الـ line مربوطة بـ IRQ |
| `ACTIVE LOW` | الـ flag `GPIOD_FLAG_ACTIVE_LOW` مضبوط |

```bash
# عرض محتوى debugfs بالكامل مع GPIO chip details
ls /sys/kernel/debug/gpio*

# في بعض kernels في entry مخصص لكل chip
cat /sys/kernel/debug/gpiochip0
```

---

#### 2. sysfs entries

**الـ** sysfs GPIO interface (legacy — موجود لو `CONFIG_GPIO_SYSFS=y`):

```bash
# قائمة كل الـ GPIO devices المسجلة
ls /sys/bus/gpio/devices/

# معلومات gpiochip معين
ls /sys/class/gpio/
# مثال:
# gpiochip0  gpiochip32  gpiochip64

# عدد الـ lines في gpiochip0
cat /sys/class/gpio/gpiochip0/ngpio

# base address في numberspace (deprecated)
cat /sys/class/gpio/gpiochip0/base

# label الـ chip
cat /sys/class/gpio/gpiochip0/label
```

**الـ** character device interface (الطريقة الحديثة):

```bash
# كل gpiochip بيبقى له /dev/gpiochipN
ls -la /dev/gpiochip*

# استخدام gpioinfo من libgpiod
gpioinfo gpiochip0

# مثال output
# gpiochip0 - 54 lines:
#         line   0:      unnamed       unused   input  active-high
#         line   2:      "SDA1"        "i2c"    input  active-high [used]
```

**الـ** sysfs entries للـ `gpio_device`:

```bash
# device attributes
ls /sys/bus/gpio/devices/gpiochip0/
# of_node  power  subsystem  uevent  ...

# ufwnode / device tree link
cat /sys/bus/gpio/devices/gpiochip0/of_node
```

---

#### 3. ftrace — Tracepoints/Events

**تفعيل tracing لـ GPIO subsystem:**

```bash
# اعرف الـ events المتاحة
ls /sys/kernel/debug/tracing/events/gpio/

# الـ events الموجودة عادةً:
# gpio_direction  gpio_value

# فعّل كل gpio events
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو event معين
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل العملية اللي تحتاج تتتبعها
# مثال: تشغيل LED

# اقرا الـ trace
cat /sys/kernel/debug/tracing/trace
```

**مثال على trace output:**

```
# TASK-PID   CPU#  IRQS-OFF  TIMESTAMP  FUNCTION
#               |     |         |            |
   gpiotest-1234 [001] ....  1234.567890: gpio_direction: gpio=17 in=0
   gpiotest-1234 [001] ....  1234.567910: gpio_value: gpio=17 set=1
```

**فلترة على chip معين:**

```bash
# filter بـ gpio number
echo 'gpio >= 0 && gpio <= 31' > /sys/kernel/debug/tracing/events/gpio/gpio_value/filter
```

**استخدام function tracer لـ gpiolib functions:**

```bash
# تتبع gpiod_set_value و gpiod_get_value
echo 'gpiod_set_value*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'gpiod_get_value*' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'gpiod_direction*' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'gpiod_request*'   >> /sys/kernel/debug/tracing/set_ftrace_filter

echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

---

#### 4. printk / dynamic debug

**الـ** gpiolib بيستخدم `gpiod_dbg()` و `gpiochip_dbg()` اللي بيطلعوا بالـ `pr_debug` / `dev_dbg` level.

**تفعيل dynamic debug للـ gpiolib:**

```bash
# فعّل كل debug messages في gpiolib.c
echo 'file drivers/gpio/gpiolib.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل gpio drivers
echo 'file drivers/gpio/* +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل بـ function name
echo 'func gpiod_request +p' > /sys/kernel/debug/dynamic_debug/control
echo 'func gpiochip_add_data_with_key +p' > /sys/kernel/debug/dynamic_debug/control

# اعرض الـ dynamic debug settings الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep gpio

# تفعيل من kernel command line (في /etc/default/grub أو bootloader)
# GRUB_CMDLINE_LINUX="... dyndbg='file drivers/gpio/* +p'"
```

**رفع loglevel عشان تشوف الـ debug messages:**

```bash
# الطريقة السريعة
echo 8 > /proc/sys/kernel/printk

# أو
dmesg -n 8

# اتابع الـ kernel log
dmesg -w | grep -i gpio
journalctl -k -f | grep -i gpio
```

---

#### 5. Kernel Config Options للـ Debugging

| CONFIG | الوصف | الاستخدام |
|--------|--------|-----------|
| `CONFIG_DEBUG_GPIO` | يفعّل extra validation داخل gpiolib — يتحقق من الـ flags والـ direction قبل كل عملية | مهم جداً أثناء development |
| `CONFIG_GPIO_SYSFS` | يعرض legacy sysfs interface — ضروري للـ tools القديمة | debugging بالـ shell |
| `CONFIG_GPIOLIB_IRQCHIP` | يفعّل IRQ chip support جوه gpiolib | لو بتستخدم GPIO-to-IRQ |
| `CONFIG_GPIO_CDEV` | يفعّل character device interface (الحديث) | `gpioinfo`, `gpiomon`, `gpioset` |
| `CONFIG_GPIO_CDEV_V1` | يفعّل ABI v1 القديم في character device | backward compat |
| `CONFIG_OF_GPIO` | device tree GPIO support | لو بتشتغل على embedded |
| `CONFIG_PINCTRL` | pinctrl integration — لازم للـ `pin_ranges` | SoC debugging |
| `CONFIG_LOCKDEP` | يفعّل lockdep لـ GPIO IRQ locks | deadlock detection |
| `CONFIG_PROVE_LOCKING` | يثبت correctness الـ locking | race condition detection |
| `CONFIG_GPIO_MOCKUP` | GPIO chip وهمي للـ testing | unit tests بدون hardware |
| `CONFIG_GPIO_SIM` | GPIO simulator (بديل أحدث لـ mockup) | virtual GPIO testing |

```bash
# تحقق إيه الـ configs المفعلة
grep -E 'CONFIG_(DEBUG_GPIO|GPIO_SYSFS|GPIOLIB_IRQCHIP|GPIO_CDEV|GPIO_MOCKUP|GPIO_SIM)' /boot/config-$(uname -r)

# أو من داخل kernel source
zcat /proc/config.gz | grep -E 'CONFIG_(DEBUG_GPIO|GPIO_SYSFS)'
```

---

#### 6. أدوات خاصة بالـ GPIO Subsystem

**الـ** `libgpiod` tools — الطريقة الحديثة والمفضلة:

```bash
# تثبيت
apt install gpiod   # Debian/Ubuntu
dnf install libgpiod-utils  # Fedora

# عرض كل chips والـ lines
gpioinfo

# عرض chip معين
gpioinfo gpiochip0

# قراءة قيمة GPIO line
gpioget --chip gpiochip0 17

# كتابة قيمة
gpioset --chip gpiochip0 17=1

# مراقبة GPIO events (edge detection)
gpiomon --chip gpiochip0 --num-events 10 --rising-edge --falling-edge 17

# detect chips
gpiodetect
```

**الـ** `gpio-hammer` — stress testing:

```bash
# موجود في tools/gpio/ في kernel source
# بيعمل rapid toggling لاختبار الـ driver
./gpio-hammer -n gpiochip0 -o 17
```

**الـ** `devlink` — مش مباشر للـ GPIO لكن مفيد لو الـ GPIO جزء من network/platform device:

```bash
devlink dev show
devlink port show
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|-------|
| `gpio-N: gpiochip_add: tried to insert a GPIO chip with zero lines` | الـ `ngpio` = 0 في الـ `gpio_chip` | تحقق من initialization الـ driver |
| `gpio-N (label): gpiochip_add: GPIO lines not contiguous` | تعارض في الـ GPIO numbering | مرر `base = -1` لـ dynamic allocation |
| `gpiod_request: invalid GPIO` | `desc` مش valid أو خارج النطاق | تحقق من الـ DT/ACPI lookup |
| `gpiod_request: GPIO busy` | الـ `GPIOD_FLAG_REQUESTED` مضبوط بالفعل | في consumer تاني بيستخدم نفس الـ line |
| `gpio-N (label): gpiochip_irqchip_add: GPIO chip already has an irqchip` | ترتيب تسجيل غلط | سجل `gpiochip` قبل `irqchip` |
| `GPIO chip driver's request function is not allowed to sleep` | `can_sleep=true` لكن بيُستدعى من atomic context | استخدم `gpiod_set_value_cansleep` |
| `WARN_ON: GPIO line not hogged` | الـ `gpiod_hog` فشل | تحقق من DT: `gpio-hog` property |
| `-EPROBE_DEFER` مرتجع من `gpiod_get` | الـ GPIO controller ما اتسجلش لسه | طبيعي — الـ kernel بيعيد probe تاني |
| `-ENOENT` مرتجع من `gpiod_get` | الـ GPIO مش موجود في DT/ACPI/board file | تحقق من الـ con_id واسم الـ property |
| `gpiod_set_value: invalid GPIO` | استخدام `desc` بعد `gpiod_put` (use-after-free) | راجع lifecycle الـ descriptor |
| `irq N: nobody cared` | GPIO IRQ بيحصل ولا حد بيتعامل معاه | تحقق من الـ IRQ handler registration |
| `Disabling IRQ #N` | الـ IRQ محصل كتير أوي | افحص الـ hardware bounce أو الـ trigger type |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

**نقاط مهمة في `gpiolib.c` لإضافة `dump_stack()` أو `WARN_ON()`:**

```c
/* في gpiod_request_commit() — تحقق إن الـ desc مش مستخدم */
int gpiod_request_commit(struct gpio_desc *desc, const char *label)
{
    /* يكشف double-request bugs */
    WARN_ON(test_bit(GPIOD_FLAG_REQUESTED, &desc->flags));
    /* ... */
}

/* في gpiod_free_commit() — تحقق إن الـ desc فعلاً مستخدم */
void gpiod_free_commit(struct gpio_desc *desc)
{
    /* يكشف double-free bugs */
    WARN_ON(!test_bit(GPIOD_FLAG_REQUESTED, &desc->flags));
    /* ... */
}

/* في gpiod_direction_output_nonotify() — تحقق من atomic context */
int gpiod_direction_output_nonotify(struct gpio_desc *desc, int value)
{
    /* لو الـ chip بيحتاج sleep وإحنا في atomic context */
    WARN_ON(desc->gdev->can_sleep && in_atomic());
    /* ... */
}

/* في driver's .request callback — لمعرفة call chain */
static int mydriver_gpio_request(struct gpio_chip *gc, unsigned int offset)
{
    dev_dbg(&gc->gpiodev->dev, "GPIO %u requested\n", offset);
    if (unlikely(debug_callchain))
        dump_stack();   /* يكشف مين طلب الـ GPIO */
    return 0;
}
```

**نقاط مناسبة حول `gpiod_line_state_notify()`:**

```c
/* قبل وبعد الـ notify — لتتبع line state events */
void gpiod_line_state_notify(struct gpio_desc *desc, unsigned long action)
{
    /* WARN لو desc مش requested ولا محتاج notify */
    WARN_ON(!test_bit(GPIOD_FLAG_REQUESTED, &desc->flags) &&
            action != GPIOLINE_CHANGED_RELEASED);
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware تطابق حالة الـ Kernel

```bash
# اقرا القيمة الحالية من kernel perspective
gpioget --chip gpiochip0 17

# قارنها بالقراءة من debugfs
cat /sys/kernel/debug/gpio | grep 'gpio-17'

# لو الـ GPIO على SoC مع memory-mapped registers
# احسب عنوان الـ register من الـ datasheet
# مثال: BCM2835 GPIO registers base = 0x3F200000
# GPIO FSEL register for GPIO 17 = base + 0x04 (GPFSEL1)
# GPIO level register = base + 0x34 (GPLEV0)

# اقرا الـ register مباشرة (بيحتاج root)
devmem2 0x3F200034 w   # اقرا GPLEV0
# Bit 17 = حالة GPIO 17
```

**مقارنة نتيجة الـ kernel مع الـ hardware:**

```bash
#!/bin/bash
# compare-gpio.sh
GPIO=17
CHIP=gpiochip0

KERNEL_VAL=$(gpioget $CHIP $GPIO)
echo "Kernel sees GPIO $GPIO = $KERNEL_VAL"

# اقرا من hardware مباشرة (BCM2835 مثال)
HW_REG=$(devmem2 0x3F200034 w 2>/dev/null | awk '/Value/ {print $NF}')
HW_BIT=$(( (HW_REG >> GPIO) & 1 ))
echo "Hardware GPIO $GPIO = $HW_BIT"

if [ "$KERNEL_VAL" != "$HW_BIT" ]; then
    echo "MISMATCH! Check active-low flag or driver bug"
fi
```

---

#### 2. Register Dump Techniques

**باستخدام `devmem2`:**

```bash
# تثبيت
apt install devmem2

# قراءة 4 bytes من عنوان معين
devmem2 0x3F200000 w    # GPFSEL0 (GPIO 0-9 function select)
devmem2 0x3F200004 w    # GPFSEL1 (GPIO 10-19)
devmem2 0x3F200034 w    # GPLEV0  (GPIO 0-31 levels)
devmem2 0x3F200038 w    # GPLEV1  (GPIO 32-53 levels)

# كتابة قيمة
devmem2 0x3F20001C w 0x00020000  # SET GPIO 17 (GPSET0)
devmem2 0x3F200028 w 0x00020000  # CLR GPIO 17 (GPCLR0)
```

**باستخدام `/dev/mem` مباشرة مع Python:**

```python
import mmap, struct

GPIO_BASE = 0x3F200000  # BCM2835
BLOCK_SIZE = 4096

with open('/dev/mem', 'rb') as f:
    mem = mmap.mmap(f.fileno(), BLOCK_SIZE,
                    mmap.MAP_SHARED, mmap.PROT_READ,
                    offset=GPIO_BASE)
    mem.seek(0x34)  # GPLEV0 offset
    gplev0 = struct.unpack('<I', mem.read(4))[0]
    print(f"GPLEV0 = 0x{gplev0:08X}")
    print(f"GPIO 17 = {(gplev0 >> 17) & 1}")
```

**لو الـ GPIO على I2C expander (مثل MCP23017):**

```bash
# اكتشاف الـ device
i2cdetect -y 1

# قراءة direction و value registers
i2cget -y 1 0x20 0x00   # IODIRA — direction port A (1=input, 0=output)
i2cget -y 1 0x20 0x01   # IODIRB — direction port B
i2cget -y 1 0x20 0x12   # GPIOA  — value port A
i2cget -y 1 0x20 0x13   # GPIOB  — value port B
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**نقاط القياس الأساسية:**

```
GPIO Pin ──────────── Test Point ──── Logic Analyzer Ch1
GND     ──────────── GND         ──── Logic Analyzer GND
(optional) VCC ────────────────── Logic Analyzer Vref
```

**إعدادات الـ Logic Analyzer:**

```
Sample Rate:    >= 10x أسرع من أسرع toggle متوقع
                مثال: GPIO بيتقلب بـ 1MHz → sample بـ 10MHz على الأقل

Voltage Level:  3.3V أو 5V حسب الـ SoC
Threshold:      1.65V للـ 3.3V signals
                2.5V  للـ 5V signals

Trigger Mode:   Edge trigger (Rising/Falling/Both)
Pre-trigger:    20% من الـ capture buffer
```

**ربط الـ kernel trace بالـ hardware — scope trigger technique:**

```c
/* في الـ driver، استخدم GPIO تاني كـ scope trigger */
static void mydriver_critical_operation(void)
{
    /* Toggle debug GPIO HIGH — يظهر على الـ scope */
    gpiod_set_value(priv->debug_gpio, 1);

    /* العملية الحقيقية */
    do_something_important();

    /* Toggle LOW — يحدد نهاية العملية على الـ scope */
    gpiod_set_value(priv->debug_gpio, 0);
}
```

**أماكن القياس على الـ PCB:**

```
┌─────────────────────────────────────────┐
│  SoC / MCU                              │
│  GPIO17 ──── R33 (series) ──── TP17 ────│──── Load
│  GPIO27 ──── R34 (series) ──── TP27 ────│──── LED
│  GND    ─────────────────── TPGND ──────│──── GND
└─────────────────────────────────────────┘
                                  ↑
                         وصّل Logic Analyzer هنا
```

---

#### 4. Hardware Issues الشائعة + Kernel Log Patterns

| المشكلة | ما بيظهر في الـ Kernel Log | التحقيق |
|---------|--------------------------|---------|
| **Floating input** — الـ pin معلق بدون pull | قراءات عشوائية، لا error صريح | أضف `bias-pull-up` أو `bias-pull-down` في DT |
| **Voltage mismatch** — SoC 3.3V متصل بـ 5V | لا error لكن قراءات خاطئة دايماً | افحص بـ oscilloscope، أضف level shifter |
| **GPIO كـ IRQ مع bounce** | `irq N: nobody cared` أو `Disabling IRQ #N` | تحقق من `debounce_period_us`، أضف hardware filter |
| **I2C GPIO expander مش بيرد** | `i2c i2c-1: ... timeout` أو `-ETIMEDOUT` | افحص SDA/SCL بـ scope، تحقق من pull-up resistors |
| **GPIO مش بيتحرر** | `Unintended GPIO deferred free` | driver بيعمل `gpiod_free` قبل consumer خلص |
| **Active-low logic معكوسة** | LED بيشتغل عكس المتوقع | تحقق من `GPIOD_FLAG_ACTIVE_LOW` في DT |
| **SPI GPIO expander بطيء** | `spi_transfer timeout` + GPIO ops slow | افحص SPI clock، زود timeout |
| **Power sequencing خاطئ** | GPIO valid قبل power rail يستقر | راجع power sequencing في schematic |
| **GPIO shared بين consumers** | `gpiod_request: GPIO busy` | استخدم `GPIOD_FLAG_SHARED` أو pinmux |

---

#### 5. Device Tree Debugging

**التحقق إن الـ DT يطابق الـ Hardware:**

```bash
# اعرض الـ DT المحمل فعلياً (بيحتاج dtc)
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | less

# ابحث عن gpio-controller nodes
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 'gpio-controller'

# تحقق من gpio-cells
hexdump -C /sys/firmware/devicetree/base/soc/gpio@3f200000/\#gpio-cells

# اعرض كل properties لـ GPIO node
ls /sys/firmware/devicetree/base/soc/gpio@3f200000/
```

**مثال DT صح:**

```dts
/* GPIO controller node */
gpio0: gpio@3f200000 {
    compatible = "brcm,bcm2835-gpio";
    reg = <0x3f200000 0xb4>;
    gpio-controller;
    #gpio-cells = <2>;          /* <offset flags> */
    interrupts = <2 17>, <2 19>;
    interrupt-controller;
    #interrupt-cells = <2>;
};

/* Consumer node */
my_device {
    /* con_id="reset", offset=17, active-low */
    reset-gpios = <&gpio0 17 GPIO_ACTIVE_LOW>;

    /* GPIO hog example */
    en-hog {
        gpio-hog;
        gpios = <22 GPIO_ACTIVE_HIGH>;
        output-high;
        line-name = "enable";
    };
};
```

```bash
# تحقق إن الـ GPIO controller اتسجل صح
dmesg | grep -iE 'gpio.*registered|gpiochip.*added|gpiolib'

# تحقق من الـ pin ranges (لو CONFIG_PINCTRL=y)
cat /sys/kernel/debug/pinctrl/*/gpio-ranges

# تحقق من gpio-hog تم تطبيقه
cat /sys/kernel/debug/gpio | grep 'hogged\|hog'
```

---

### Practical Commands

#### Shell Commands جاهزة للنسخ

**1. نظرة عامة سريعة على حالة الـ GPIO system:**

```bash
# كل الـ chips
gpiodetect

# تفاصيل كاملة
gpioinfo --all

# من debugfs
cat /sys/kernel/debug/gpio

# الـ kernel messages المتعلقة بـ GPIO
dmesg | grep -iE 'gpio|gpiochip|gpiolib' | tail -50
```

**2. تتبع GPIO request/free:**

```bash
# فعّل dynamic debug
echo 'file drivers/gpio/gpiolib.c +p' > /sys/kernel/debug/dynamic_debug/control

# في terminal تاني، تابع الـ log
dmesg -w | grep -i gpio

# اعمل الـ operation (مثلاً load/unload driver)
modprobe my_gpio_driver
dmesg | tail -30
```

**3. ftrace لرصد كل GPIO calls:**

```bash
#!/bin/bash
# gpio-trace.sh
cd /sys/kernel/debug/tracing

echo 0 > tracing_on
echo > trace
echo function > current_tracer

# تحديد الـ functions المراد تتبعها
cat > set_ftrace_filter << 'EOF'
gpiod_get_value*
gpiod_set_value*
gpiod_direction*
gpiod_request*
gpiod_free*
gpiochip_get_desc
gpio_do_set_config
gpiod_line_state_notify
EOF

echo 1 > tracing_on
sleep 5     # اعمل العملية المطلوب تتبعها خلال الـ 5 ثواني
echo 0 > tracing_on

cat trace
```

**4. تشخيص GPIO IRQ:**

```bash
# اعرض كل IRQs المتعلقة بـ GPIO
cat /proc/interrupts | grep -i gpio

# مثال output:
#  46:      1234     0  GPIO  17  Edge  "my-device"
# col1: IRQ number | col2-N: count per CPU | آخر عمود: consumer

# تحقق من spurious IRQs
cat /proc/irq/46/spurious

# اعرض irqdomain mapping
cat /sys/kernel/debug/irq/irqdomain_mapping | grep gpio
```

**5. اختبار GPIO expander على I2C:**

```bash
# اكتشاف الـ device
i2cdetect -y 1

# قراءة كل registers (MCP23017 example: 23 registers)
for reg in $(seq 0 22); do
    printf "Reg 0x%02X: " $reg
    i2cget -y 1 0x20 $reg
done

# كتابة direction register (كل GPIOs = output)
i2cset -y 1 0x20 0x00 0x00   # IODIRA = all output
i2cset -y 1 0x20 0x01 0x00   # IODIRB = all output

# اضبط قيمة
i2cset -y 1 0x20 0x12 0xFF   # GPIOA = all high
```

**6. GPIO simulator للـ testing بدون hardware:**

```bash
# تحميل GPIO simulator module
modprobe gpio-sim

# إنشاء simulated GPIO chip عبر configfs
mount -t configfs none /sys/kernel/config

mkdir /sys/kernel/config/gpio-sim/my-chip
mkdir /sys/kernel/config/gpio-sim/my-chip/gpio-bank-a
echo 16 > /sys/kernel/config/gpio-sim/my-chip/gpio-bank-a/num_lines
echo 1 > /sys/kernel/config/gpio-sim/my-chip/live

# الآن بيبقى chip جديد
gpiodetect
# gpio-sim-0    [my-chip gpio-bank-a] (16 lines)

# اختبر
gpioget --chip gpio-sim-0 0
gpioset --chip gpio-sim-0 0=1

# cleanup
echo 0 > /sys/kernel/config/gpio-sim/my-chip/live
rmdir /sys/kernel/config/gpio-sim/my-chip/gpio-bank-a
rmdir /sys/kernel/config/gpio-sim/my-chip
```

**7. مثال output كامل لـ `cat /sys/kernel/debug/gpio`:**

```
gpiochip0: GPIOs 0-53, parent: platform/3f200000.gpio, pinctrl-bcm2835:
 gpio-2   (                    |sda1                ) in  hi    IRQ ACTIVE LOW
 gpio-3   (                    |scl1                ) in  hi    IRQ ACTIVE LOW
 gpio-4   (                    |                    ) in  hi
 gpio-17  (                    |reset               ) out lo    ACTIVE LOW
 gpio-27  (                    |led0                ) out lo    ACTIVE LOW

# تفسير:
# gpio-17 out lo ACTIVE LOW:
#   الـ kernel طلب output قيمته logic-low
#   لكن لأن ACTIVE LOW → الـ physical pin = HIGH
#   يعني الـ reset line مش محصل (active-low reset)
#
# gpio-2 in hi IRQ ACTIVE LOW:
#   I2C SDA line مرفوعة (pull-up)
#   IRQ enabled على الـ line
#   ACTIVE LOW يعني الـ assertion بتحصل لما يهبط
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — GPIO مش بيتحرر صح وبيعمل kernel oops

#### العنوان
GPIO descriptor leak بيعمل crash في industrial gateway بيشتغل على RK3562

#### السياق
شركة بتعمل industrial IoT gateway بيشتغل على RK3562. الـ gateway ده بيتحكم في 8 relays عبر GPIO عشان يوصّل/يفصّل load في factory automation. الـ driver بتاع الـ relay مكتوب بـ C custom، وبيعمل `gpiod_get()` لكل relay في الـ `probe()`.

#### المشكلة
عند hot-unplug للـ USB-to-GPIO bridge module (بيعمل `rmmod`)، الـ kernel بيطلع:

```
BUG: kernel NULL pointer dereference at 0000000000000008
RIP: gpiolib.c:gpiod_free_commit
```

#### التحليل

الـ `gpiod_free_commit()` — اللي بتتنادى من `gpiod_free()` — بتحاول تعمل dereference لـ `desc->gdev` بعد ما الـ `gpio_device` اتحذف.

في `gpiolib.h`:

```c
struct gpio_desc {
    struct gpio_device  *gdev;   /* pointer لازم يفضل valid طول العمر */
    unsigned long        flags;
    ...
};
```

الـ `gpio_device` بيفضل موجود حتى بعد ما الـ chip يتشال (بسبب reference counting)، لكن لو الـ driver مش عامل `gpiod_put()` قبل ما الـ device يتحذف، بيحصل race. المشكلة الأساسية إن الـ driver مش بيفرّق بين:

- `gpiod_free()` — اللي بتعمل cleanup للـ `GPIOD_FLAG_REQUESTED` وبتبعت `line_state_notifier`
- `gpiod_free_commit()` — اللي بتعمل الشغل الفعلي

الـ `gpio_chip_guard` في الـ header بيوفّر SRCU locking:

```c
struct gpio_chip_guard {
    struct gpio_device *gdev;
    struct gpio_chip   *gc;
    int                 idx;
};

DEFINE_CLASS(gpio_chip_guard,
     struct gpio_chip_guard,
     srcu_read_unlock(&_T.gdev->srcu, _T.idx),  /* cleanup تلقائي */
     ({ ... srcu_read_lock(...); ... }),
     struct gpio_desc *desc)
```

الـ driver مبعملش lock صح، وبيحاول يعمل `gpiod_free()` بعد ما `srcu` بتاع الـ `gdev` اتحذف.

#### الحل

الـ driver المتضرر كان عنده:

```c
static int relay_remove(struct platform_device *pdev)
{
    /* WRONG: بيعمل free من غير ما يتأكد إن الـ gpio_device لسه valid */
    for (int i = 0; i < NUM_RELAYS; i++)
        gpiod_set_value(priv->relay_gpio[i], 0);
    return 0;
}
```

الصح:

```c
static void relay_remove(struct platform_device *pdev)
{
    struct relay_priv *priv = platform_get_drvdata(pdev);

    for (int i = 0; i < NUM_RELAYS; i++) {
        if (!IS_ERR_OR_NULL(priv->relay_gpio[i])) {
            gpiod_set_value_cansleep(priv->relay_gpio[i], 0);
            /* devm_gpiod_get تعمل cleanup تلقائي — استخدمها بدل gpiod_get */
        }
    }
}
```

والأفضل من الأساس يستخدم `devm_gpiod_get()` في الـ `probe()` عشان الـ kernel يعمل cleanup تلقائي بالترتيب الصح.

#### الدرس المستفاد
الـ `gpio_device` بيفضل موجود بعد ما الـ chip يتشال بسبب reference counting، لكن الـ `gpio_desc` الـ pointers ممكن تبقى dangling لو الـ driver مش بيستخدم `devm_*` variants. دايمًا استخدم `devm_gpiod_get()` في driver probe عشان تضمن cleanup ordering صح.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — HDMI 5V Power GPIO بيعمل حاجة غلط بسبب active-low confusion

#### العنوان
**الـ** `GPIOD_FLAG_ACTIVE_LOW` بيعمل polarity inversion غير متوقع في HDMI power control

#### السياق
Android TV box مبنية على Allwinner H616. الـ HDMI connector محتاج GPIO يتحكم في الـ 5V power rail. الـ hardware engineer وصّل transistor inverting stage، فالـ GPIO HIGH يطفي الـ 5V والـ LOW يشغّلها. الـ DT مكتوب فيه `output-low` ولا في حاجة active-low.

#### المشكلة
الـ HDMI شاشة مش بتتشغل أبدًا رغم إن الـ driver بيعمل `gpiod_set_value(hdmi_pwr, 1)` — اللي المفروض يعني "enable".

#### التحليل

في `gpiolib.h`، الـ `gpio_desc` بيحمل الـ flag ده:

```c
#define GPIOD_FLAG_ACTIVE_LOW   6  /* GPIO is active-low */
```

الـ `gpiod_set_value()` بتحترم الـ `GPIOD_FLAG_ACTIVE_LOW` وبتعمل inversion تلقائي. يعني لو `ACTIVE_LOW` مش set، والـ hardware طالب inversion، الـ logical "1" بيبعت physical HIGH — وده بيطفّي الـ HDMI power.

الـ DT كانت:

```dts
/* WRONG: مفيش active-low flag */
hdmi_pwr: hdmi-power {
    gpio = <&pio 7 6 GPIO_ACTIVE_HIGH>;
    output-low;
};
```

لما الـ driver بيعمل:
```c
/* gpiod_configure_flags بتشوف GPIOD_FLAG_ACTIVE_LOW في desc->flags */
gpiod_set_value(hdmi_pwr_gpio, 1);  /* logical ON */
```

بما إن `GPIOD_FLAG_ACTIVE_LOW` مش set في `desc->flags`، الـ value بيتبعت as-is: GPIO HIGH → transistor يطفّي الـ 5V.

#### الحل

تعديل الـ DT عشان يعكس الـ hardware polarity الصح:

```dts
/* CORRECT: إضافة GPIO_ACTIVE_LOW */
hdmi_pwr: hdmi-power {
    gpio = <&pio 7 6 GPIO_ACTIVE_LOW>;
    output-high;  /* logical high = physical low = transistor يشغّل */
};
```

بعد كده الـ kernel بيحط `GPIOD_FLAG_ACTIVE_LOW` في `desc->flags` تلقائيًا، والـ `gpiod_set_value(gpio, 1)` بيبقى معناه "enable HDMI" ويبعت physically LOW للـ transistor.

للتحقق:

```bash
# شوف الـ flags بتاعة الـ descriptor
cat /sys/kernel/debug/gpio | grep hdmi

# المخرج المتوقع بعد الإصلاح:
# gpio-XXX (hdmi-power       ) out lo ACTIVE LOW
```

#### الدرس المستفاد
الـ `GPIOD_FLAG_ACTIVE_LOW` في `gpio_desc.flags` بيتحط من الـ DT `GPIO_ACTIVE_LOW` flag. خلّي الـ polarity logic في الـ DT مش في الـ driver code — الـ driver يشتغل على logical values دايمًا (`1` = active).

---

### السيناريو الثالث: IoT Sensor Board على STM32MP1 — GPIO hog بيمنع driver من يطلب الـ pin

#### العنوان
**الـ** `GPIOD_FLAG_IS_HOGGED` بيسبب `-EBUSY` في SPI CS GPIO على STM32MP1

#### السياق
Custom IoT sensor board على STM32MP1. الـ board عندها SPI flash متوصّل بـ manual chip-select GPIO. الـ BSP engineer حط GPIO hog في الـ DT عشان يضمن إن الـ CS يفضل HIGH (deasserted) من بدري في الـ boot. الـ SPI driver بيحاول يطلب نفس الـ GPIO كـ CS.

#### المشكلة
الـ SPI flash driver بيرجع `-EBUSY` في الـ probe ومش بيشتغل. الـ dmesg بيقول:

```
spi-nor spi0.0: Failed to get chip select GPIO: -16
```

#### التحليل

في `gpiolib.h`:

```c
#define GPIOD_FLAG_IS_HOGGED    11  /* GPIO is hogged */
```

لما الـ DT عنده:

```dts
&gpiob {
    spi_cs_hog: spi-cs-hog {
        gpio-hog;
        gpios = <3 GPIO_ACTIVE_LOW>;
        output-high;
        line-name = "spi-cs";
    };
};
```

الـ `gpiod_hog()` بيتنادى في الـ `gpiochip_add_data()` وبيعمل `gpiod_request_commit()` للـ pin ده، وبيحط `GPIOD_FLAG_IS_HOGGED` و `GPIOD_FLAG_REQUESTED` في `desc->flags`.

لما الـ SPI driver بيجي يعمل `gpiod_request()`:

```c
int gpiod_request(struct gpio_desc *desc, const char *label)
{
    /* بيشوف GPIOD_FLAG_REQUESTED موجود → يرجع -EBUSY */
    if (test_and_set_bit(GPIOD_FLAG_REQUESTED, &desc->flags))
        return -EBUSY;
    ...
}
```

الـ `GPIOD_FLAG_SHARED` مش set (ده للـ shared pins بس)، فالـ request بيفشل.

#### الحل

**الخيار الأول:** شيل الـ GPIO hog من الـ DT وخلّي الـ SPI driver يتحكم فيه:

```dts
/* حذف الـ hog بالكامل */
```

**الخيار الثاني:** لو محتاج الـ hog في early boot، استخدم `gpiod_direction_output_nonotify()` في الـ driver بعد ما يعمل acquire للـ GPIO من الـ hog:

```c
/* في الـ driver probe، حرّر الـ hog يدوياً قبل ما تطلب */
/* لكن الأصح: استخدم GPIO هانم في DT عشان SPI subsystem يتحكم */
```

**الأصح:** في الـ DT، خلّي الـ SPI controller هو اللي يتحكم في الـ CS:

```dts
&spi0 {
    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        cs-gpios = <&gpiob 3 GPIO_ACTIVE_LOW>;
    };
};
```

```bash
# تشخيص الـ hogged GPIOs
cat /sys/kernel/debug/gpio | grep hogged
# gpio-35 (spi-cs           ) out hi ACTIVE LOW [hogged]
```

#### الدرس المستفاد
الـ GPIO hog بيعمل `gpiod_request_commit()` باكر جداً وبيحط `GPIOD_FLAG_IS_HOGGED | GPIOD_FLAG_REQUESTED`. أي driver تاني بيحاول يطلب نفس الـ pin بيلاقي `-EBUSY`. الـ hog مش بديل لـ proper driver ownership.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — gpio_array fastpath بيكسب performance لكن بيعمل data corruption

#### العنوان
**الـ** `gpio_array` fastpath و `invert_mask` بيعمل bit corruption في parallel LED control على i.MX8

#### السياق
Automotive ECU على i.MX8QM بيتحكم في 16 diagnostic LEDs بشكل parallel عبر `gpiod_set_array_value()`. بعض الـ LEDs موصّلة active-high والتانية active-low على نفس الـ GPIO controller عشان PCB routing constraints. الـ team بتستخدم `gpiod_get_array()` عشان تاخد كل الـ 16 مع بعض.

#### المشكلة
بعض الـ LEDs بتولع بعكس المتوقع. المشكلة مش consistent — بعض الـ patterns صح وبعضها غلط.

#### التحليل

في `gpiolib.h`:

```c
struct gpio_array {
    struct gpio_desc    **desc;
    unsigned int          size;
    struct gpio_device   *gdev;
    unsigned long        *get_mask;
    unsigned long        *set_mask;
    unsigned long         invert_mask[];  /* flexible array — حجمه dynamic */
};
```

الـ `gpiod_set_array_value_complex()` بتستخدم الـ fastpath لما كل الـ descriptors من نفس الـ `gdev`:

```c
int gpiod_set_array_value_complex(bool raw, bool can_sleep,
                                  unsigned int array_size,
                                  struct gpio_desc **desc_array,
                                  struct gpio_array *array_info,
                                  unsigned long *value_bitmap)
{
    /* fastpath: لو array_info موجودة وكل الـ GPIOs من نفس الـ chip */
    if (array_info && array_info->gdev == desc_array[0]->gdev) {
        /* بتطبق invert_mask على value_bitmap */
        /* invert_mask بيتحسب من GPIOD_FLAG_ACTIVE_LOW في كل desc */
    }
}
```

المشكلة: الـ `invert_mask[]` بيتحسب مرة واحدة في `gpiod_get_array()` بناءً على `GPIOD_FLAG_ACTIVE_LOW` في كل descriptor وقت الـ array creation. لو الـ developer بعدين غيّر الـ `raw` parameter بين calls بشكل غلط، الـ inversion ماتطبقتش صح.

الـ driver كان بيعمل:

```c
/* WRONG: raw=true بيتخطى الـ active-low inversion */
gpiod_set_array_value_complex(true, false, 16,
                               descs->desc, descs->info, bitmap);
```

بدل:

```c
/* CORRECT: raw=false عشان يحترم GPIOD_FLAG_ACTIVE_LOW */
gpiod_set_array_value_complex(false, false, 16,
                               descs->desc, descs->info, bitmap);
```

#### الحل

```c
/* استخدم الـ wrapper العادي اللي بيحترم polarity */
ret = gpiod_set_array_value(descs->ndescs, descs->desc,
                            descs->info, value_bitmap);
```

للـ debugging:

```bash
# شوف الـ active-low flags لكل line
gpioinfo gpiochip0 | grep -E "active-low|led"

# أو من debugfs
cat /sys/kernel/debug/gpio | grep -A2 "gpio-chip"
```

وللـ verify الـ `invert_mask` بيتحسب صح، أضف temporary debug في driver:

```c
/* طبع الـ invert_mask بعد gpiod_get_array */
for (int i = 0; i < descs->ndescs; i++) {
    pr_debug("LED[%d] active_low=%d\n", i,
             gpiod_is_active_low(descs->desc[i]));
}
```

#### الدرس المستفاد
الـ `gpio_array` fastpath و `invert_mask` بيتحسبوا من `GPIOD_FLAG_ACTIVE_LOW` في كل `gpio_desc`. استخدام `raw=true` بيتخطى كل الـ polarity logic. دايمًا استخدم الـ non-raw wrappers إلا لو عارف إنك بتتعامل مع physical values صريحة.

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — GPIO line_state_notifier بيسبب deadlock في early boot

#### العنوان
**الـ** `line_state_wq` و `line_state_lock` بيعملوا deadlock في custom AM62x board أثناء الـ bring-up

#### السياق
Engineer بيعمل bring-up لـ custom industrial board على AM62x (TI). الـ board عندها I2C GPIO expander (TCA9555) موصّل على I2C3. الـ engineer بيحاول يطلب GPIO من الـ expander في اتجاه early `initcall` عشان يتحكم في power sequencing.

#### المشكلة
الـ board بتتجمد في الـ boot بعد:

```
gpio-expander: registered TCA9555 on I2C3
```

الـ kernel مش بيكمل، ومفيش panic، مجرد hang. الـ watchdog بيعمل reset بعد 30 ثانية.

#### التحليل

في `gpiolib.h`:

```c
struct gpio_device {
    ...
    struct raw_notifier_head  line_state_notifier;
    rwlock_t                  line_state_lock;       /* RW-spinlock */
    struct workqueue_struct  *line_state_wq;
    ...
};
```

الـ `gpiod_line_state_notify()` بتتنادى من `gpiod_request()` وبتعمل:

```c
void gpiod_line_state_notify(struct gpio_desc *desc, unsigned long action)
{
    /* بتاخد read_lock على line_state_lock */
    read_lock(&desc->gdev->line_state_lock);
    raw_notifier_call_chain(&desc->gdev->line_state_notifier, action, desc);
    read_unlock(&desc->gdev->line_state_lock);
}
```

المشكلة: الـ early `initcall` كان بيعمل الآتي في sequence خطأ:

1. بيطلب GPIO من الـ TCA9555 expander
2. الـ `gpiod_request()` بتبعت notification عبر `line_state_notifier`
3. الـ notifier handler كان بيحاول يعمل I2C transaction (عشان يقرأ state)
4. الـ I2C controller على AM62x بيستخدم GPIO لـ bus recovery (GPIO-based I2C bus recovery feature)
5. الـ I2C GPIO recovery بيطلب GPIO تاني من نفس الـ expander
6. `gpiod_request()` تاني بيحاول ياخد `line_state_lock` تاني → deadlock

الـ `line_state_lock` هو `rwlock_t` (RW spinlock) — مش recursive. الـ second lock attempt من نفس الـ context بيعمل spin forever.

#### الحل

**الخيار الأول:** استخدم `gpiod_direction_output_nonotify()` في الـ early init path عشان تتجنب الـ notification:

```c
/* في الـ early power sequencing code */
/* استخدم _nonotify variant عشان تتجنب الـ notifier chain */
ret = gpiod_direction_output_nonotify(pwr_gpio, 1);
```

**الخيار الثاني:** افصل الـ I2C GPIO expander requests عن الـ early init — خلّيها تحصل بعد ما كل الـ subsystems تتهيأ:

```c
/* بدل early_initcall، استخدم device_initcall أو اعمل deferred work */
static int __init board_power_seq_init(void)
{
    /* schedule_work بتخلّي الـ request يحصل بعد ما الـ workqueues تتجهز */
    schedule_work(&power_seq_work);
    return 0;
}
device_initcall(board_power_seq_init);
```

**الخيار الثالث:** عزل الـ I2C bus recovery GPIO على GPIO controller منفصل مش على الـ expander:

```dts
/* في الـ DT، وصّل الـ I2C bus recovery GPIOs على SoC GPIO مباشرة */
&i2c3 {
    scl-gpios = <&gpio0 14 GPIO_ACTIVE_HIGH>;  /* SoC GPIO مش expander */
    sda-gpios = <&gpio0 15 GPIO_ACTIVE_HIGH>;
};
```

```bash
# للـ debugging: شوف الـ lockdep output
dmesg | grep -i "deadlock\|circular lock"

# enable lockdep في kernel config
CONFIG_LOCKDEP=y
CONFIG_LOCK_STAT=y
```

#### الدرس المستفاد
الـ `line_state_wq` موجود بالظبط عشان يعمل الـ notifications في process context منفصل. لكن الـ `line_state_lock` هو spinlock وبيتاخد synchronously في `gpiod_line_state_notify()`. أي notifier handler يعمل operation تتطلب نفس الـ GPIO subsystem resources هيعمل deadlock. الـ `_nonotify` variants موجودة كـ escape hatch لحالات الـ internal/early use.
## Phase 7: مصادر ومراجع

### مقالات LWN.net الأساسية

دي أهم المقالات اللي بتغطي تطور الـ gpiolib والـ descriptor-based interface:

| المقال | الأهمية |
|--------|---------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة لـ GPIO subsystem في الكيرنل — نقطة البداية المثالية |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | بيشرح المشاكل في الـ integer-based API وليه اتحولنا لـ descriptor-based |
| [gpiolib: introduce descriptor-based GPIO interface](https://lwn.net/Articles/528226/) | أول patch series قدّم الـ `gpiod_*` functions وفكرة الـ `gpio_desc` |
| [gpio: introduce descriptor-based interface](https://lwn.net/Articles/531848/) | النسخة المتطورة من الـ proposal اللي وضّحت الـ opaque descriptor model |
| [New descriptor-based GPIO interface](https://lwn.net/Articles/565662/) | بيغطي التحديثات اللي اتعملت على الـ descriptor API قبل الـ merge |
| [Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/) | التوثيق الرسمي للـ interface الجديدة اللي اتضاف للكيرنل |
| [gpio: add pin biasing and drive mode to gpiolib](https://lwn.net/Articles/439095/) | إضافة pull-up/pull-down وـ open-drain support للـ gpiolib |
| [gpiolib: Add GPIO name support](https://lwn.net/Articles/651500/) | إضافة الـ `name` field للـ `gpio_desc` وتسمية الـ lines |

---

### توثيق الكيرنل الرسمي

**الـ** `Documentation/` **paths** في شجرة الكيرنل:

```
Documentation/driver-api/gpio/index.rst       ← نقطة الدخول الرئيسية
Documentation/driver-api/gpio/consumer.rst    ← استخدام GPIO من driver (gpiod_get, gpiod_set_value, ...)
Documentation/driver-api/gpio/driver.rst      ← كتابة GPIO chip driver (gpio_chip struct)
Documentation/driver-api/gpio/board.rst       ← GPIO mappings (DT, ACPI, platform data)
Documentation/driver-api/gpio/legacy.rst      ← الـ integer-based API القديمة (deprecated)
Documentation/driver-api/gpio/intro.rst       ← مقدمة للـ GPIO subsystem
```

**الروابط الإلكترونية للتوثيق:**

- [General Purpose I/O — الفهرس الكامل](https://docs.kernel.org/driver-api/gpio/index.html)
- [GPIO Descriptor Consumer Interface](https://docs.kernel.org/driver-api/gpio/consumer.html)
- [GPIO Descriptor Driver Interface](https://docs.kernel.org/driver-api/gpio/driver.html)
- [GPIO Mappings (Board/DT/ACPI)](https://docs.kernel.org/driver-api/gpio/board.html)
- [Legacy GPIO Interfaces (deprecated)](https://docs.kernel.org/driver-api/gpio/legacy.html)

---

### Commits مهمة في تاريخ gpiolib

الـ commits دي غيّرت شكل الـ subsystem بشكل جذري — ابحث عنها في `git log drivers/gpio/`:

| الـ commit / الوصف | الأثر |
|--------------------|-------|
| `d2876d08` — `gpio: introduce `struct gpio_desc`` | أول ظهور لـ `gpio_desc` كـ opaque descriptor بدل الـ integer |
| `ff2b1359` — `gpiolib: introduce `gpio_device`` | فصل الـ `gpio_device` عن الـ `gpio_chip` — جوهر الـ refactoring |
| `14e85c0e` — `gpio: make the gpiochip a real device` | تحويل الـ GPIO chip لـ proper Linux device بـ `struct device` |
| إضافة `SRCU` للـ `gpio_device` | حماية الـ `chip` pointer من الـ concurrent access بـ `srcu_struct` |
| إضافة `GPIOD_FLAG_SHARED` و`GPIOD_FLAG_SHARED_PROXY` | دعم الـ shared GPIO lines بين أكتر من consumer |

للتحقق من أي commit:
```bash
git log --oneline --follow drivers/gpio/gpiolib.h
git log --oneline --follow drivers/gpio/gpiolib.c | head -40
```

---

### Mailing List

**الـ** `linux-gpio` **mailing list** هو المكان الرئيسي لمناقشات الـ GPIO subsystem:

- [lore.kernel.org/linux-gpio](https://lore.kernel.org/linux-gpio/) — أرشيف كامل للـ patches والنقاشات
- المشرف الحالي: **Bartosz Golaszewski** (gpio subsystem maintainer)
- للبحث عن نقاش معين: `https://lore.kernel.org/linux-gpio/?q=gpio_device`

---

### كتب موصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 13** — *USB Drivers* (مفيد للفهم العام للـ device model)
- **الفصل 14** — *The Linux Device Model* — أساسي لفهم `struct device` اللي بيستخدمها `gpio_device`
- **متاح مجاناً:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- **ملاحظة:** LDD3 قديمة (kernel 2.6) وبتغطي الـ integer-based GPIO API — استخدمها للخلفية النظرية بس

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17** — *Devices and Modules* — يشرح الـ `struct device`, `cdev`, `module` كلها موجودة في `gpio_device`
- **الفصل 10** — *Kernel Synchronization Methods* — لفهم الـ `spinlock`, `rwlock`, `srcu` المستخدمة في الـ struct
- الكتاب ده أحسن مصدر لفهم الـ kernel internals الأساسية

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 15** — *Device Drivers* — بيغطي GPIO من منظور الـ embedded systems
- بيشرح ازاي الـ driver writer بيتعامل مع GPIO في السياق العملي
- مفيد لفهم الـ use cases الحقيقية للـ `gpiod_get()` و `gpiod_set_value()`

---

### eLinux.org

- [New GPIO Interface for User Space (ELCE 2017 — PDF)](https://elinux.org/images/7/74/Elce2017_new_GPIO_interface.pdf) — عرض ممتاز عن الـ character device interface (`/dev/gpiochip*`) اللي بيستخدم الـ `gpio_desc` internally
- [Introduction to pin muxing and GPIO control under Linux (ELC 2021 — PDF)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) — يشرح العلاقة بين الـ pinctrl subsystem والـ gpiolib وكيف `pin_ranges` في `gpio_device` بتربطهم

---

### kernelnewbies.org

- [LinuxChanges — تتبع تغييرات الكيرنل عبر الإصدارات](https://kernelnewbies.org/LinuxChanges) — ابحث عن "gpio" في أي صفحة إصدار لتتابع التغييرات
- [Linux 6.2 Changes](https://kernelnewbies.org/Linux_6.2) — يحتوي على تحديثات مهمة للـ GPIO subsystem
- [Linux 6.6 Changes](https://kernelnewbies.org/Linux_6.6) — تحديثات إضافية على الـ gpiolib
- [Linux 6.13 Changes](https://kernelnewbies.org/Linux_6.13) — أحدث التغييرات المتاحة

---

### Source Code المرجعي

```
drivers/gpio/gpiolib.h      ← الـ internal structs (gpio_device, gpio_desc, gpio_array)
drivers/gpio/gpiolib.c      ← التنفيذ الكامل للـ gpiolib core
drivers/gpio/gpiolib-cdev.c ← الـ character device interface (/dev/gpiochipN)
drivers/gpio/gpiolib-sysfs.c← الـ legacy sysfs interface (deprecated)
drivers/gpio/gpiolib-acpi.c ← ACPI GPIO support
drivers/gpio/gpiolib-of.c   ← Device Tree GPIO support
include/linux/gpio/driver.h ← الـ public API لكتابة GPIO chip driver (struct gpio_chip)
include/linux/gpio/consumer.h← الـ public API للـ consumer (gpiod_get, gpiod_flags)
```

---

### Search Terms للبحث عن معلومات أكتر

```
linux kernel gpiolib gpio_device gpio_desc
linux kernel "descriptor based gpio"
linux gpio gpiod_get gpiod_set_value driver tutorial
linux kernel gpio_chip vs gpio_device
linux kernel gpio SRCU RCU protection
linux kernel gpiochip character device /dev/gpiochip
linux gpio pinctrl pin_ranges integration
linux kernel gpio line state notifier
linux gpiod_flags consumer interface
linux kernel gpio hogging gpiod_hog
```
## Phase 8: Writing simple module

### الفكرة

**`gpiod_line_state_notify`** هي الدالة اللي بتتبعت كل مرة GPIO line بتتغير حالتها (requested / released / reconfigured). بدل ما نعمل kprobe عليها مباشرة، هنستخدم الـ **`blocking_notifier_head device_notifier`** اللي موجود في `struct gpio_device` — ده notifier رسمي بيتفعل لما GPIO device بتتضاف أو بتتشال. ده الأنسب من ناحية الأمان والـ API الصح.

الـ `device_notifier` هو `blocking_notifier_head` ومتاح من خلال `gpiochip_find()` + `gpio_device_get_by_label()` لكن للتعليم العملي هنعمل **kprobe على `gpiod_request`** لأنها:
- exported وموثقة
- بتتحط في كل مرة consumer بيطلب GPIO line
- بتحمل `struct gpio_desc *` و label — معلومات مفيدة للطباعة

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * gpio_req_probe.c
 *
 * Intercepts every call to gpiod_request() using a kprobe and prints
 * the GPIO chip label, line number, and consumer label to the kernel log.
 */

#include <linux/module.h>       /* MODULE_* macros, module_init/exit        */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, etc.     */
#include <linux/gpio/driver.h>  /* struct gpio_chip (accessed via gpio_desc)*/
#include <linux/printk.h>       /* pr_info()                                */

/* --------------------------------------------------------------------------
 * الـ struct gpio_desc معرفة في gpiolib.h اللي هو internal header,
 * مش متاح لـ out-of-tree modules. بنعرّف forward declaration بس
 * ونوصل للبيانات عن طريق الـ kprobe args بشكل opaque pointer.
 * الحل الصح هو استخدام gpiod_get_direction() وgpiochip_get_data()
 * اللي هم exported — لكن هنا بنطبع faddr بس كـ demonstration.
 * --------------------------------------------------------------------------*/

/* اسم الدالة اللي هنحط فيها الـ kprobe */
#define TARGET_FUNC  "gpiod_request"

/*
 * pre_handler: بيتنادى قبل ما الـ CPU ينفذ أول instruction في gpiod_request.
 *
 * @p   : pointer للـ kprobe نفسه (ممكن نوصل منه للـ symbol name)
 * @regs: pointer لـ struct pt_regs — بتحتوي على حالة الـ registers وقت الـ hit
 *
 * على x86-64:
 *   regs->di = الـ argument الأول  (struct gpio_desc *desc)
 *   regs->si = الـ argument الثاني (const char *label)
 */
static int gpio_req_pre_handler(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * نجيب الـ arguments من registers.
     * regs->di هو RDI — أول argument في calling convention على x86-64.
     * regs->si هو RSI — تاني argument.
     *
     * مش بنعمل dereference للـ desc pointer هنا علشان:
     * 1. الـ struct internal وmight change
     * 2. ممكن يكون invalid لو اتنادى من context غريب
     * بدل كده بنطبع الـ address بس + الـ label string.
     */
    const char *consumer_label = (const char *)regs->si;

    pr_info("[gpio_probe] gpiod_request() called: desc=%px, consumer=\"%s\"\n",
            (void *)regs->di,
            consumer_label ? consumer_label : "<NULL>");

    return 0; /* return 0 = let execution continue normally */
}

/* --------------------------------------------------------------------------
 * struct kprobe: الـ object الأساسي اللي بيعرّف الـ probe.
 * .symbol_name بتقول للـ kernel انه يحل الـ address من الـ symbol table.
 * .pre_handler بتتنادى قبل الـ instruction.
 * --------------------------------------------------------------------------*/
static struct kprobe gpio_kp = {
    .symbol_name = TARGET_FUNC,
    .pre_handler = gpio_req_pre_handler,
};

/* --------------------------------------------------------------------------
 * module_init: بيتنادى لما تعمل insmod.
 * register_kprobe() بتحط breakpoint في ذاكرة الـ kernel عند الـ symbol.
 * لو فشلت (مثلاً الـ symbol مش موجود أو محمي) بترجع error code.
 * --------------------------------------------------------------------------*/
static int __init gpio_probe_init(void)
{
    int ret;

    ret = register_kprobe(&gpio_kp);
    if (ret < 0) {
        pr_err("[gpio_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[gpio_probe] kprobe planted on %s at %px\n",
            TARGET_FUNC, gpio_kp.addr);
    return 0;
}

/* --------------------------------------------------------------------------
 * module_exit: بيتنادى لما تعمل rmmod.
 * unregister_kprobe() ضروري علشان:
 * - يشيل الـ breakpoint من الـ kernel text
 * - يمنع استدعاء handler بعد ما الـ module اتشال من الذاكرة
 *   (لو مشيلناهوش هيحصل kernel panic لأن الـ handler address بقى invalid)
 * --------------------------------------------------------------------------*/
static void __exit gpio_probe_exit(void)
{
    unregister_kprobe(&gpio_kp);
    pr_info("[gpio_probe] kprobe removed from %s\n", TARGET_FUNC);
}

module_init(gpio_probe_init);
module_exit(gpio_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("GPIO Learner");
MODULE_DESCRIPTION("kprobe on gpiod_request() to trace GPIO consumer requests");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية لكل kernel module: `MODULE_LICENSE`, `module_init`, `module_exit` |
| `linux/kprobes.h` | تعريف `struct kprobe`, `register_kprobe`, `unregister_kprobe`, و`struct pt_regs` |
| `linux/gpio/driver.h` | تعريف `struct gpio_chip` — محتاجينه كـ context لو حبينا نوسع الكود |
| `linux/printk.h` | `pr_info()` / `pr_err()` لطباعة رسايل في kernel log |

#### الـ `pre_handler`

الـ handler بياخد `struct pt_regs *regs` اللي فيها حالة كل registers وقت الـ hit. على x86-64 الـ calling convention بيحط arguments في `RDI, RSI, RDX, RCX, R8, R9` بالترتيب. **الأول (`regs->di`) هو الـ `gpio_desc*`**، والتاني (`regs->si`) هو الـ `label` string. بنطبع الـ pointer address والـ label علشان نشوف مين بيطلب ايه GPIO.

#### الـ `struct kprobe`

الـ `.symbol_name` بيخلي الـ kernel يحل الـ virtual address من الـ kallsyms في runtime — مش لازم نعرف الـ address manually. الـ `.pre_handler` بيتنادى قبل الـ instruction الأولى في الدالة المستهدفة.

#### الـ `module_exit` والـ unregister

**لازم** نعمل `unregister_kprobe()` في الـ exit. لو مشيلناش الـ breakpoint، الـ kernel هيحاول ينفذ الـ handler بعد ما الـ module اتشال من الذاكرة — الـ address بقى garbage — وده kernel panic مباشر. الـ unregister بيشيل الـ breakpoint ويستنى أي execution جارية تخلص قبل ما يرجع.

---

### طريقة التجربة

```bash
# بناء الـ module (محتاج Makefile بسيط)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod gpio_req_probe.ko

# تشغيل أي عملية بتطلب GPIO (مثلاً toggle LED أو قراءة pushbutton)
# أو simulate request عن طريق:
# echo "1" > /sys/class/gpio/export  (deprecated path - just for test)

# مشاهدة الـ log
sudo dmesg | grep gpio_probe

# إزالة الـ module
sudo rmmod gpio_req_probe
sudo dmesg | grep gpio_probe
```

**مثال output متوقع:**
```
[gpio_probe] kprobe planted on gpiod_request at ffffffffc0a12340
[gpio_probe] gpiod_request() called: desc=ffff888003c8a400, consumer="leds-gpio"
[gpio_probe] gpiod_request() called: desc=ffff888003c8a480, consumer="sysfs"
[gpio_probe] kprobe removed from gpiod_request
```
