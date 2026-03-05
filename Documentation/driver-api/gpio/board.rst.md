## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **GPIO Subsystem** في Linux kernel، وبالتحديد من قسم **Driver API documentation**. المسؤولون عنه هم Linus Walleij و Bartosz Golaszewski، والـ mailing list هي `linux-gpio@vger.kernel.org`.

---

### إيه هو الـ GPIO أصلاً؟

تخيل إن عندك Arduino أو Raspberry Pi وعايز توصل بيه LED أو زرار أو sensor. بتستخدم **pin** من الـ pins الموجودة على الـ board — ده الـ **GPIO (General Purpose Input/Output)**: pin عام ممكن تحوله input أو output حسب الحاجة.

في الـ Linux kernel، كل جهاز ممكن يكون عنده GPIO controller، وده بيشرف على مجموعة من الـ GPIO lines. المشكلة إنك محتاج طريقة تقول فيها للـ driver: "الـ GPIO اللي بيتحكم في الـ LED اللي عندي هو رقم 15 من chip اسمها gpio.0" — وده بالظبط اللي بيشرحه الـ file ده.

---

### إيه المشكلة اللي بيحلها الـ file ده؟

**السيناريو:** عندك device اسمه `foo` فيه:
- 3 LEDs (أحمر، أخضر، أزرق)
- زرار power

الـ driver بتاع `foo` مش المفروض يعرف أي GPIO chip موجودة أو أنهي رقم hardware. ده من مسؤولية **Board** أو **Platform** — يعني اللي بنى الـ hardware.

فمحتاج **خريطة ربط (mapping)** تقول: "الـ LED الأحمر = GPIO 15 من chip gpio.0، والـ power = GPIO 1 من نفس الـ chip لكن active-low."

الـ file ده بيشرح **3 طرق مختلفة** للربط ده حسب نوع الـ platform:

---

### الطريقة الأولى: Device Tree

لو الـ hardware بيستخدم **Device Tree** (معظم ARM platforms)، بتعرّف الـ GPIOs جوه الـ device node مباشرةً:

```c
foo_device {
    compatible = "acme,foo";
    led-gpios = <&gpio 15 GPIO_ACTIVE_HIGH>,  /* red */
                <&gpio 16 GPIO_ACTIVE_HIGH>,  /* green */
                <&gpio 17 GPIO_ACTIVE_HIGH>;  /* blue */
    power-gpios = <&gpio 1 GPIO_ACTIVE_LOW>;
};
```

الـ driver بعدين بيطلبهم ببساطة:

```c
red   = gpiod_get_index(dev, "led", 0, GPIOD_OUT_HIGH);
green = gpiod_get_index(dev, "led", 1, GPIOD_OUT_HIGH);
blue  = gpiod_get_index(dev, "led", 2, GPIOD_OUT_HIGH);
power = gpiod_get(dev, "power", GPIOD_OUT_HIGH);
```

الـ driver مش شايف أرقام hardware — بس اسم الـ function ("led", "power"). الـ kernel هو اللي بيعمل الـ translation.

---

### الطريقة الثانية: ACPI

على أجهزة x86 بتستخدم **ACPI** (زي laptops وبعض embedded boards)، نفس الفكرة بس بـ ACPI syntax مع `_DSD` (Device Specific Data) اللي اتضافت في ACPI 5.1:

```
Name (_DSD, Package () {
    ToUUID("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
    Package () {
        Package () { "led-gpios", Package () { ^FOO, 0, 0, 1, ^FOO, 1, 0, 1, ^FOO, 2, 0, 1 } },
        Package () { "power-gpios", Package () { ^FOO, 3, 0, 0 } },
    }
})
```

الـ driver نفسه مش بيتغير — نفس `gpiod_get()` calls.

---

### الطريقة الثالثة: Software Nodes

**Software Nodes** هي طريقة حديثة بديلة لـ **Platform Data** القديم. بتبني structure في الـ memory شبيهة بالـ Device Tree لكن بـ C code:

```c
/* 1. عرّف node للـ GPIO controller */
static const struct software_node gpio_controller_node = {
    .name = "gpio-foo",
};

/* 2. عرّف properties للـ LED device */
static const struct property_entry led_device_props[] = {
    PROPERTY_ENTRY_STRING("label", "myboard:green:status"),
    PROPERTY_ENTRY_GPIO("gpios", &gpio_controller_node, 42, GPIO_ACTIVE_HIGH),
    { }
};

/* 3. عرّف الـ software node */
static const struct software_node led_device_swnode = {
    .name = "status-led",
    .properties = led_device_props,
};
```

الـ macro `PROPERTY_ENTRY_GPIO()` بيربط الـ consumer بالـ GPIO controller عبر الـ software node بتاعته.

---

### الطريقة الرابعة: Platform Data (الطريقة القديمة)

على boards قديمة مفيهاش Device Tree ولا ACPI، بتعمل **lookup table** في board file:

```c
struct gpiod_lookup_table gpios_table = {
    .dev_id = "foo.0",  /* اسم الـ device */
    .table = {
        GPIO_LOOKUP_IDX("gpio.0", 15, "led",   0, GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP_IDX("gpio.0", 16, "led",   1, GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP_IDX("gpio.0", 17, "led",   2, GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP    ("gpio.0",  1, "power",    GPIO_ACTIVE_LOW),
        { }, /* نهاية الجدول */
    },
};

/* في وقت الـ boot */
gpiod_add_lookup_table(&gpios_table);
```

---

### مفهوم الـ GPIO Hog

**GPIO Hog** = line بتتحجز أوتوماتيكي من الـ kernel نفسه من غير ما أي driver يطلبها. مثلاً power enable pin لازم يكون HIGH دايماً من أول ما الـ chip تتشغل:

```c
struct gpiod_hog gpio_hog_table[] = {
    GPIO_HOG("gpio.0", 10, "foo", GPIO_ACTIVE_LOW, GPIOD_OUT_HIGH),
    { }
};
gpiod_add_hogs(gpio_hog_table);
```

---

### مفهوم Fast Bitmap Processing

لما بتطلب **array of pins** مرة واحدة، الـ kernel ممكن يعمل get/set لكل الـ pins بـ single bitmap operation بدل ما يلف عليهم واحد واحد — لكن بشروط:
- الـ pin[0] لازم يكون hardware number 0
- الـ pins المتتالية في نفس الـ chip لازم hardware numbers بتاعتها تساوي indexes بتاعتها

لو الشروط دي متحققتش، بيستخدم الـ slow path — اللي بيلف على كل pin على حدة.

---

### الـ Key Concept: active-high vs active-low

ميزة كبيرة في الـ new descriptor-based API إن الـ `active-low` بتتعمل handle في وقت الـ mapping مش في الـ driver. يعني:

- الـ driver بيقول `GPIOD_OUT_HIGH` للـ power GPIO
- لكن لأن الـ mapping قال `GPIO_ACTIVE_LOW`، الـ kernel بيحوله لـ 0 فعلياً على الـ hardware
- الـ driver ميعرفش ولا يهمه — هو بس شغال بالـ logical value

---

### الـ Files المهمة في الـ Subsystem

| الملف | الدور |
|---|---|
| `Documentation/driver-api/gpio/board.rst` | الـ file ده — بيشرح GPIO mapping من ناحية الـ board |
| `Documentation/driver-api/gpio/consumer.rst` | كيف الـ drivers بتستخدم الـ GPIOs |
| `Documentation/driver-api/gpio/driver.rst` | كيف تكتب GPIO controller driver |
| `Documentation/driver-api/gpio/intro.rst` | مقدمة عامة للـ GPIO subsystem |
| `Documentation/driver-api/gpio/legacy-boards.rst` | دليل تحويل board files القديمة لـ software nodes |
| `include/linux/gpio/machine.h` | تعريف `gpiod_lookup`، `gpiod_lookup_table`، `gpiod_hog`، والـ macros |
| `include/linux/gpio/property.h` | تعريف `PROPERTY_ENTRY_GPIO()` macro للـ software nodes |
| `drivers/gpio/gpiolib.c` | الـ core implementation للـ GPIO subsystem |
| `drivers/gpio/gpiolib-of.c` | دعم Device Tree |
| `drivers/gpio/gpiolib-acpi-core.c` | دعم ACPI |
| `drivers/gpio/gpiolib-swnode.c` | دعم Software Nodes |
| `drivers/gpio/gpiolib-devres.c` | الـ devres (device resource management) helpers |
| `include/linux/gpio.h` | الـ public API header الرئيسي |

---

### ملخص سريع

الـ file ده بيجاوب على سؤال واحد: **"كيف يعرف الـ kernel إن الـ GPIO رقم كذا في chip كذا هو اللي بيتحكم في الـ LED في device كذا؟"**

الإجابة: عبر 4 طرق حسب نوع الـ platform — Device Tree، ACPI، Software Nodes، أو Platform Data. الـ driver نفسه بيستخدم نفس الـ API (`gpiod_get`) في كل الحالات، والـ mapping هي اللي بتختلف.
## Phase 2: شرح الـ GPIO Mappings Framework

### المشكلة — ليه الـ Framework ده موجود أصلاً؟

في أي embedded system، الـ driver بيحتاج يتحكم في GPIO معين على الـ hardware. الطريقة القديمة (Legacy Integer API) كانت بتخلي الـ driver يعمل كده:

```c
gpio_request(15, "led-red");   /* hardcoded GPIO number! */
gpio_direction_output(15, 1);
```

**المشكلة الجوهرية** إن الـ GPIO number 15 ده مش معناه حاجة مستقلة — هو بيعتمد على:
- إيه الـ GPIO controller الموجود على الـ SoC
- ترتيب تسجيل الـ controllers في الـ kernel
- كمية الـ GPIOs في كل controller

يعني نفس الكود على board تانية، الـ GPIO 15 ممكن يكون pin تاني خالص. وكمان الـ driver بقى فيه معلومة hardware محددة مش المفروض تكون فيه.

**المشكلة التانية**: الـ active-low logic. لو الـ GPIO بـ active-low (يعني 0 = on)، كل الـ drivers كانت لازم تعمل XOR يدوي لو عايزين يفعّلوا الـ signal — كل واحد بطريقة مختلفة.

---

### الحل — مبدأ الـ GPIO Descriptor API

الـ kernel حل المشكلة بفصل الـ concerns في 3 طبقات:

1. **الـ Hardware Description** (DT / ACPI / Platform Data): مين متصل بمين على الـ board
2. **الـ GPIO Subsystem Core** (gpiolib): بيعمل lookup ويرجع descriptor مجرد
3. **الـ Consumer Driver**: بيتكلم بالـ *function name* مش بالـ number

الـ driver بقى بيقول: "أنا عايز الـ GPIO اللي اسمه `led` من الـ device اللي أنا مرتبط بيه" — والـ subsystem هو اللي يحل الـ mapping.

```
Driver:  gpiod_get(dev, "led", GPIOD_OUT_HIGH)
                       ↓
         GPIO Core: اعمل lookup في الـ board description
                       ↓
         Board:     gpio.0 pin 15, active-high
                       ↓
         struct gpio_desc* ← مجرد opaque handle
```

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   Consumer Drivers                           │
│   (leds-gpio, spi-gpio, regulator-gpio, mmc-pwrseq, ...)   │
│                                                             │
│   gpiod_get(dev, "reset", GPIOD_OUT_LOW)                   │
│   gpiod_set_value(desc, 1)                                  │
└────────────────────────┬────────────────────────────────────┘
                         │  Consumer API (gpio/consumer.h)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  GPIO Core (gpiolib)                        │
│                                                             │
│  ┌─────────────────┐    ┌──────────────────────────────┐   │
│  │  Lookup Engine  │    │   Active-low Translation     │   │
│  │                 │    │   Open-drain Handling        │   │
│  │ DT lookup       │    │   Descriptor Management      │   │
│  │ ACPI lookup     │    │                              │   │
│  │ Platform lookup │    └──────────────────────────────┘   │
│  └────────┬────────┘                                       │
│           │                                                 │
│  ┌────────▼─────────────────────────────────────────────┐  │
│  │              struct gpio_desc (per-line state)       │  │
│  │  gdev → struct gpio_device (owns the chip)          │  │
│  │  flags: direction, active-low, open-drain, ...      │  │
│  └────────┬─────────────────────────────────────────────┘  │
└───────────┼─────────────────────────────────────────────────┘
            │  chip ops (gpio_chip callbacks)
            ▼
┌─────────────────────────────────────────────────────────────┐
│              GPIO Controllers (Providers)                    │
│                                                             │
│  ┌────────────┐  ┌────────────┐  ┌──────────────────────┐  │
│  │ gpio-omap  │  │ gpio-pl061 │  │  gpio-pca953x (I2C)  │  │
│  │ (SoC GPIO) │  │ (ARM PL061)│  │  (expander on bus)   │  │
│  └────────────┘  └────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│         Board Description (Hardware Mapping Layer)          │
│                                                             │
│   ┌─────────────────┐  ┌──────────┐  ┌────────────────┐   │
│   │   Device Tree   │  │   ACPI   │  │ Platform Data  │   │
│   │ led-gpios =     │  │  _DSD    │  │ gpiod_lookup   │   │
│   │ <&gpio 15 HIGH> │  │          │  │ _table         │   │
│   └─────────────────┘  └──────────┘  └────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

### الـ Real-World Analogy — المستودع والـ Barcode

تخيل نظام مستودع كبير:

| عنصر في الـ Analogy | المقابل في الـ Kernel |
|---|---|
| **المستودع** نفسه | الـ GPIO controller (gpio_chip) |
| **رقم الرف** في المستودع | الـ `chip_hwnum` (hardware number) |
| **اسم المنتج** (مثلاً "قلم أحمر") | الـ `con_id` (function name مثل "led") |
| **كتالوج المستودع** | الـ `gpiod_lookup_table` (platform data) |
| **فاتورة الشراء** من الكتالوج | الـ `struct gpio_desc*` اللي بيرجعه `gpiod_get()` |
| **مدير المستودع** | الـ GPIO Core (gpiolib) |
| **الـ barcode scanner** | الـ lookup engine (DT/ACPI/platform) |
| **خاصية المنتج**: كهرباء أو لا | الـ `GPIO_ACTIVE_LOW` flag |

الـ driver (الـ عامل اللي بيطلب) مش بيعرف رقم الرف — بيقول بس "عايز قلم أحمر". المدير (gpiolib) هو اللي بيبحث في الكتالوج ويجيبه من الرف الصح. وكمان لو المنتج جاي معبّأ معكوس (active-low)، المدير هو اللي بيعكسه قبل ما يدّيه للعامل — مش شغلة العامل إنه يفكر في ده.

---

### الـ Core Abstraction — الـ `struct gpio_desc`

الـ central idea في الـ subsystem ده هي الـ **GPIO Descriptor**. بدل ما تتكلم بـ integer numbers، كل GPIO line بيتمثل كـ opaque pointer:

```c
struct gpio_desc; /* defined internally in gpiolib.c — consumer ما يشوفش internals */
```

الـ descriptor بيحمل:
- مين الـ chip اللي بيملكه (`gpio_device`)
- إيه الـ hardware offset جوا الـ chip
- الـ flags الخاصة بالـ line (active-low, open-drain, etc.)
- الـ label (لـ debugging)

**الفكرة الجوهرية**: الـ `gpio_desc` هو عقد بين الـ board description والـ driver — الـ board بتقول "الـ led هو GPIO 15 من الـ controller ده بـ active-high"، والـ driver بيمسك الـ descriptor ويتعامل معاه بـ logical values مش physical.

---

### الـ Structs الأساسية وعلاقتها ببعض

#### 1. الـ `gpiod_lookup` — وحدة الـ mapping

```c
struct gpiod_lookup {
    const char *key;       /* label بتاع الـ gpio_chip أو اسم الـ line */
    u16 chip_hwnum;        /* رقم الـ pin جوا الـ chip (أو U16_MAX لو key = line name) */
    const char *con_id;    /* اسم الـ function زي "led" أو "reset" */
    unsigned int idx;      /* index لو في أكتر من GPIO بنفس الـ function */
    unsigned long flags;   /* GPIO_ACTIVE_HIGH / GPIO_ACTIVE_LOW / ... */
};
```

#### 2. الـ `gpiod_lookup_table` — جدول الـ board

```c
struct gpiod_lookup_table {
    struct list_head list;       /* linked في global list جوا الـ kernel */
    const char *dev_id;          /* اسم الـ device: "foo.0" */
    struct gpiod_lookup table[]; /* flexible array — النهاية بـ {} فاضي */
};
```

#### 3. الـ `gpiod_hog` — حجز مبكر للـ line

```c
struct gpiod_hog {
    struct list_head list;
    const char *chip_label;  /* اسم الـ chip */
    u16 chip_hwnum;          /* رقم الـ pin */
    const char *line_name;   /* اسم الـ consumer */
    unsigned long lflags;    /* line flags (active-low, etc.) */
    int dflags;              /* direction flags (GPIOD_OUT_HIGH, etc.) */
};
```

#### علاقة الـ Structs ببعض

```
gpiod_add_lookup_table()
        │
        ▼
  gpio_lookup_list (global linked list)
        │
        ├── gpiod_lookup_table { dev_id="foo.0" }
        │       └── table[]
        │             ├── gpiod_lookup { key="gpio.0", hwnum=15, con_id="led", idx=0 }
        │             ├── gpiod_lookup { key="gpio.0", hwnum=16, con_id="led", idx=1 }
        │             └── gpiod_lookup { key="gpio.0", hwnum=17, con_id="led", idx=2 }
        │
        └── gpiod_lookup_table { dev_id="bar.1" }
                └── table[]
                      └── gpiod_lookup { key="gpio.1", hwnum=3, con_id="reset", idx=0 }

عند استدعاء gpiod_get(dev_foo, "led", GPIOD_OUT_HIGH):
        │
        ▼
 بيبحث في gpio_lookup_list عن dev_id="foo.0" + con_id="led" + idx=0
        │
        ▼
 بيلاقي: chip="gpio.0", hwnum=15, flags=GPIO_ACTIVE_HIGH
        │
        ▼
 بيجيب الـ gpio_desc* المقابل للـ chip "gpio.0" pin 15
        │
        ▼
 بيرجع struct gpio_desc* جاهز للاستخدام
```

---

### طرق الـ Board Description الثلاث مقارنةً

| الطريقة | الـ Use Case | الـ Mechanism | الـ Lookup |
|---|---|---|---|
| **Device Tree** | Embedded Linux (ARM, RISC-V) | `led-gpios = <&gpio 15 HIGH>` في DTS | DT property parser |
| **ACPI** | x86 / UEFI platforms | `_DSD` + `GpioIo` resources | ACPI GPIO property API |
| **Software Nodes** | No DT/ACPI, بديل modern لـ platform_data | `PROPERTY_ENTRY_GPIO()` + `software_node` | Property API (نفس DT) |
| **Platform Data** | Legacy board files (غير مفضل) | `gpiod_lookup_table` + `gpiod_add_lookup_table()` | Manual list search |

---

### الـ Macros — اختصارات الـ Lookup Table

```c
/* أبسط صورة: GPIO واحد بـ function واحد */
GPIO_LOOKUP("gpio.0", 15, "led", GPIO_ACTIVE_HIGH)
/* equivalent to: */
GPIO_LOOKUP_IDX("gpio.0", 15, "led", 0, GPIO_ACTIVE_HIGH)

/* لو محتاج أكتر من GPIO بنفس الـ function */
GPIO_LOOKUP_IDX("gpio.0", 15, "led", 0, GPIO_ACTIVE_HIGH),  /* red   */
GPIO_LOOKUP_IDX("gpio.0", 16, "led", 1, GPIO_ACTIVE_HIGH),  /* green */
GPIO_LOOKUP_IDX("gpio.0", 17, "led", 2, GPIO_ACTIVE_HIGH),  /* blue  */

/* hog: الـ kernel يمسك الـ GPIO من غير driver */
GPIO_HOG("gpio.0", 10, "status-led", GPIO_ACTIVE_LOW, GPIOD_OUT_HIGH)
```

---

### الـ GPIO Hog — مفهوم مهم

الـ **GPIO hog** هو طلب إن الـ kernel يمسك GPIO معين لنفسه من غير ما driver يطلبه صراحة. بيتعمل عادةً لـ:
- GPIOs بتتحكم في الـ power rail من أول boot
- lines محتاجة تكون في state معينة قبل ما أي driver يشتغل

الـ hog بيتفعّل أوتوماتيك **لحظة تسجيل الـ gpiochip** (أو لحظة `gpiod_add_hogs()` لو الـ chip اتسجلت قبليه).

```
gpiochip_add()
      │
      ▼
gpiochip_setup_dev()
      │
      ▼
gpiochip_add_pin_ranges() → gpiochip_irqchip_add()
      │
      ▼
gpiod_hog_table_walk()  ← بيشوف لو في hogs مسجلة على الـ chip دي
      │
      ▼
gpiod_hog() على كل line ← بيحجز الـ descriptor ويضبط direction
```

---

### الـ Active-Low Transparency — الـ magic الحقيقي

ده من أهم مزايا الـ descriptor API. لو GPIO محدد active-low في الـ mapping:

```c
/* الـ board بتقول: power GPIO → active-low */
GPIO_LOOKUP("gpio.0", 1, "power", GPIO_ACTIVE_LOW)

/* الـ driver بيقول: فعّل الـ power */
gpiod_set_value(power_desc, 1);  /* logical 1 = ON */
/* الـ core بيترجمه أوتوماتيك لـ physical 0 */
```

الـ driver ما بيعرفش ولا بيهمه إن الـ signal معكوس. الـ GPIO core بيعمل XOR في الـ `gpiod_set_value()` لو الـ `GPIO_ACTIVE_LOW` flag موجود.

مقارنة بالـ legacy:
```c
/* legacy: الـ driver لازم يعرف هو active-low */
gpio_set_value(1, !enable);  /* ugly! المعرفة الـ hardware موجودة في الـ driver */

/* descriptor: شفاف تماماً */
gpiod_set_value(desc, enable);  /* الـ core بيتعامل مع الـ polarity */
```

---

### Fast Bitmap Path — الـ Array Optimization

لما تطلب array من GPIOs بـ `gpiod_get_array()`، الـ core بيحاول يعمل **fast path** لو الشروط اتوفرت:

```
شروط الـ Fast Path:
┌─────────────────────────────────────────────────────┐
│ 1. pin[0].hwnum == 0                                │
│ 2. pin[i].hwnum == i  (consecutive hardware order) │
│ 3. كلهم في نفس الـ chip                           │
│ 4. مفيش open-drain في الـ output path              │
└─────────────────────────────────────────────────────┘
        ↓ لو الشروط اتوفرت
┌─────────────────────────────────────────────────────┐
│  bitmap يتبعت مباشرة لـ .get_multiple()/.set_multiple() │
│  في الـ gpio_chip — operation واحدة للكل            │
└─────────────────────────────────────────────────────┘

        ↓ لو الشروط ما اتوفرتش
┌─────────────────────────────────────────────────────┐
│  slow path: كل pin بـ operation منفردة             │
└─────────────────────────────────────────────────────┘
```

الـ pins اللي مش في نفس الـ chip أو مش في الترتيب الصح بتتعمل لها slow path بالـ individual calls — لكن الـ pins اللي في الـ chip التاني بتتعمل لها fast path منفصلة بين بعض.

---

### الـ Subsystem بيملك إيه ومش بيملك إيه

| الـ GPIO Mappings Framework بيملك | بيفوّض لـ Driver / Hardware |
|---|---|
| الـ lookup logic (DT/ACPI/platform) | تنفيذ `set()`/`get()` الفعلي (في `gpio_chip`) |
| الـ active-low / polarity translation | اختيار الـ hardware pin numbering scheme |
| الـ descriptor lifecycle (get/put) | الـ interrupt handling (عشان `irq_chip` — subsystem تاني) |
| الـ hog mechanism | إعداد الـ pin multiplexing (عشان pinctrl — subsystem تاني) |
| الـ array fast-path detection | تعريف إيه الـ GPIO functions الخاصة بالـ device |
| الـ devres integration (`devm_gpiod_get`) | تسجيل الـ board description نفسها (شغلة الـ BSP) |

> **ملاحظة**: الـ **pinctrl subsystem** مرتبط جداً بالـ GPIO — بيتحكم في الـ pin muxing (إيه الـ function اللي الـ pin بيعملها). الـ GPIO core بيتكلم معاه ضمنياً لما يضبط direction الـ GPIO line.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums — Cheatsheet

#### `enum gpio_lookup_flags` — (من `linux/gpio/machine.h`)

| Flag | Value | المعنى |
|------|-------|--------|
| `GPIO_ACTIVE_HIGH` | `0 << 0` | الـ line active عند logic high (default) |
| `GPIO_ACTIVE_LOW` | `1 << 0` | الـ line active عند logic low — kernel بيعكس القيمة |
| `GPIO_OPEN_DRAIN` | `1 << 1` | الـ pin open-drain — ما بيدفعش high بنفسه |
| `GPIO_OPEN_SOURCE` | `1 << 2` | الـ pin open-source — ما بيدفعش low بنفسه |
| `GPIO_PERSISTENT` | `0 << 3` | الـ state بيتحافظ عليه عند suspend/resume (default) |
| `GPIO_TRANSITORY` | `1 << 3` | الـ state ممكن يتغير عند suspend/resume |
| `GPIO_PULL_UP` | `1 << 4` | يفعّل pull-up resistor |
| `GPIO_PULL_DOWN` | `1 << 5` | يفعّل pull-down resistor |
| `GPIO_PULL_DISABLE` | `1 << 6` | يعطّل أي pull resistor |
| `GPIO_LOOKUP_FLAGS_DEFAULT` | `ACTIVE_HIGH \| PERSISTENT` | الـ default لما مش محدد |

---

#### `enum gpiod_flags` — (من `linux/gpio/consumer.h`)

الـ flags دي بتتبعت لـ `gpiod_get()` عشان تحدد الاتجاه والقيمة الابتدائية:

| Flag | Value | المعنى |
|------|-------|--------|
| `GPIOD_ASIS` | `0` | ما تغيرش أي حاجة — استخدمه لو الـ firmware سيطر على الـ pin |
| `GPIOD_IN` | `BIT(0)` | input mode |
| `GPIOD_OUT_LOW` | `BIT(0)\|BIT(1)` | output، قيمة ابتدائية low |
| `GPIOD_OUT_HIGH` | `BIT(0)\|BIT(1)\|BIT(2)` | output، قيمة ابتدائية high |
| `GPIOD_OUT_LOW_OPEN_DRAIN` | `OUT_LOW\|BIT(3)` | output open-drain، ابتدائي low |
| `GPIOD_OUT_HIGH_OPEN_DRAIN` | `OUT_HIGH\|BIT(3)` | output open-drain، ابتدائي high |

> **مهم:** الـ `gpiod_flags` مش بتتعمل OR مع بعض — كل قيمة مستقلة.

---

#### Config Options

| Option | المعنى |
|--------|--------|
| `CONFIG_GPIOLIB` | يفعّل subsystem الـ GPIO في الـ kernel |
| `CONFIG_OF_GPIO` | يفعّل دعم Device Tree للـ GPIO |
| `CONFIG_ACPI` | يفعّل دعم ACPI للـ GPIO عبر `_DSD` |

---

### الـ Structs المهمة

#### 1. `struct gpiod_lookup`

**الغرض:** وحدة بيانات واحدة تمثل mapping بين GPIO line معينة وـ function محددة في device.

```c
struct gpiod_lookup {
    const char *key;        /* label الـ chip أو اسم الـ GPIO line */
    u16 chip_hwnum;         /* رقم الـ GPIO داخل الـ chip (أو U16_MAX لو key هو اسم line) */
    const char *con_id;     /* اسم الـ function من وجهة نظر الـ driver ("led", "power") */
    unsigned int idx;       /* الـ index لو في أكتر من GPIO تحت نفس con_id */
    unsigned long flags;    /* bitmask من gpio_lookup_flags */
};
```

**مثال واقعي:**
```c
/* GPIO رقم 15 في chip "gpio.0" → function "led" index 0، active high */
GPIO_LOOKUP_IDX("gpio.0", 15, "led", 0, GPIO_ACTIVE_HIGH)
```

**العلاقات:** بتتجمع جوه `gpiod_lookup_table` كـ flexible array.

---

#### 2. `struct gpiod_lookup_table`

**الغرض:** جدول lookup كامل خاص بـ device معين، بيربط الـ device بمجموعة من الـ GPIO mappings.

```c
struct gpiod_lookup_table {
    struct list_head list;      /* ربطه في global linked list */
    const char *dev_id;         /* identifier الـ device اللي هيستخدم الـ GPIOs دي */
    struct gpiod_lookup table[];/* flexible array من الـ lookups، ينتهي بـ entry فاضية */
};
```

**مثال واقعي:**
```c
struct gpiod_lookup_table gpios_table = {
    .dev_id = "foo.0",          /* اسم الـ platform device */
    .table = {
        GPIO_LOOKUP_IDX("gpio.0", 15, "led", 0, GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP_IDX("gpio.0", 16, "led", 1, GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP("gpio.0", 1, "power", GPIO_ACTIVE_LOW),
        { },                    /* sentinel — نهاية الجدول */
    },
};
```

**العلاقات:**
- بيتسجّل في الـ kernel عبر `gpiod_add_lookup_table()`
- الـ `list` field بيربطه في global list بيقرأها الـ GPIO subsystem عند كل `gpiod_get()`

---

#### 3. `struct gpiod_hog`

**الغرض:** يعمل "hog" لـ GPIO line — يعني الـ kernel نفسه بياخد الـ line ويحتفظ بيها من غير ما driver يطلبها صراحةً. مفيد لـ lines زي power enables اللازم تبقى active طول الوقت.

```c
struct gpiod_hog {
    struct list_head list;      /* ربطه في global hog list */
    const char *chip_label;     /* label الـ chip اللي فيها الـ line */
    u16 chip_hwnum;             /* رقم الـ GPIO داخل الـ chip */
    const char *line_name;      /* اسم consumer للـ hogged line */
    unsigned long lflags;       /* gpio_lookup_flags (active high/low, etc.) */
    int dflags;                 /* gpiod_flags (direction + initial value) */
};
```

**مثال واقعي:**
```c
/* احجز GPIO رقم 10 في "gpio.0" كـ output high من أول ما الـ chip تتسجل */
GPIO_HOG("gpio.0", 10, "foo", GPIO_ACTIVE_LOW, GPIOD_OUT_HIGH)
```

---

#### 4. `struct gpio_desc`

**الغرض:** الـ descriptor اللي بيمثل GPIO line واحدة من وجهة نظر الـ consumer. هو الـ handle اللي الـ driver بيشتغل بيه بعد `gpiod_get()`. التعريف الكامل internal في الـ kernel (opaque للـ driver).

```c
/* من وجهة نظر الـ consumer بس: */
struct gpio_desc *red;
red = gpiod_get_index(dev, "led", 0, GPIOD_OUT_HIGH);
gpiod_set_value(red, 1);
gpiod_put(red);
```

---

#### 5. `struct gpio_descs`

**الغرض:** يمثل array من الـ GPIO descriptors بتُرجع من `gpiod_get_array()` — بتسمح بـ batch operations على مجموعة GPIOs.

```c
struct gpio_descs {
    struct gpio_array *info;    /* معلومات داخلية عن الـ array (opaque) */
    unsigned int ndescs;        /* عدد الـ descriptors */
    struct gpio_desc *desc[];   /* flexible array من الـ descriptors */
};
```

**الـ fast path:** لو الـ array بتحقق الشروط (hwnum[0] == 0، والـ consecutive pins بـ indexes متطابقة مع hw numbers) → الـ kernel بيعدّي الـ bitmap مباشرة لـ `.get/set_multiple()` callback بدون ما يلف على كل pin.

---

#### 6. `struct software_node` و `struct property_entry`

**الغرض:** بيسمحوا بـ board code يبني structure زي الـ device tree في الـ memory بدون firmware خارجي. الـ `PROPERTY_ENTRY_GPIO()` macro بتربط software node بـ GPIO controller.

```c
/* node تمثل الـ GPIO controller */
static const struct software_node gpio_controller_node = {
    .name = "gpio-foo",   /* لازم يطابق label الـ gpiochip */
};

/* properties للـ LED device */
static const struct property_entry led_device_props[] = {
    PROPERTY_ENTRY_STRING("label", "myboard:green:status"),
    PROPERTY_ENTRY_GPIO("gpios", &gpio_controller_node, 42, GPIO_ACTIVE_HIGH),
    { }
};
```

---

### مخطط العلاقات بين الـ Structs

```
Board Code (platform data path)
────────────────────────────────

  gpiod_lookup_table
  ┌──────────────────────────────────────┐
  │ list       ──────────────────────────┼──→ global gpio_lookup_list
  │ dev_id = "foo.0"                     │
  │ table[]                              │
  │  ┌──────────────────────────────┐    │
  │  │ gpiod_lookup                 │    │
  │  │  key       = "gpio.0"        │    │
  │  │  chip_hwnum= 15              │    │
  │  │  con_id    = "led"           │    │
  │  │  idx       = 0               │    │
  │  │  flags     = ACTIVE_HIGH     │    │
  │  ├──────────────────────────────┤    │
  │  │ gpiod_lookup (more entries)  │    │
  │  ├──────────────────────────────┤    │
  │  │ { } ← sentinel               │    │
  │  └──────────────────────────────┘    │
  └──────────────────────────────────────┘

  gpiod_hog
  ┌──────────────────────────────────────┐
  │ list       ──────────────────────────┼──→ global gpio_hog_list
  │ chip_label = "gpio.0"                │
  │ chip_hwnum = 10                      │
  │ line_name  = "foo"                   │
  │ lflags     = ACTIVE_LOW              │
  │ dflags     = GPIOD_OUT_HIGH          │
  └──────────────────────────────────────┘

Software Node path
────────────────────────────────

  software_node (controller)          software_node (consumer)
  ┌───────────────────┐               ┌──────────────────────────┐
  │ name = "gpio-foo" │◄──────────────┤ properties[]             │
  └───────────────────┘  PROPERTY_    │  PROPERTY_ENTRY_GPIO()   │
         │               ENTRY_GPIO   │  .name = "gpios"         │
         │                            │  .ref  → gpio_controller │
         ▼                            └──────────────────────────┘
   gpiochip (label="gpio-foo")
   يتطابق بالاسم فقط

Consumer (driver)
────────────────────────────────

  gpio_descs
  ┌─────────────────────────────┐
  │ info    → gpio_array (ops)  │
  │ ndescs  = 3                 │
  │ desc[]                      │
  │  ┌──────────┐               │
  │  │gpio_desc │ → red LED     │
  │  ├──────────┤               │
  │  │gpio_desc │ → green LED   │
  │  ├──────────┤               │
  │  │gpio_desc │ → blue LED    │
  │  └──────────┘               │
  └─────────────────────────────┘
```

---

### مخطط الـ Lifecycle

#### Platform Data GPIO Mapping Lifecycle

```
Board Init
    │
    ▼
[1] تعريف gpiod_lookup_table في board file
    │
    ▼
[2] gpiod_add_lookup_table(&gpios_table)
    │  يضيف الجدول في global linked list
    │
    ▼
[3] platform_device_register() للـ consumer device
    │
    ▼
[4] driver probe() يستدعي gpiod_get() / gpiod_get_index()
    │  الـ GPIO subsystem يبحث في global lookup list
    │  يطابق dev_id + con_id + idx
    │
    ▼
[5] يُرجع gpio_desc* (حجز exclusive للـ line)
    │
    ▼
[6] driver يستخدم gpiod_set_value() / gpiod_get_value()
    │
    ▼
[7] driver remove() يستدعي gpiod_put(desc)
    │  يُفرج عن الـ line
    │
    ▼
[8] (اختياري) gpiod_remove_lookup_table() لو الـ board بيُفكّك
```

#### GPIO Hog Lifecycle

```
gpiod_add_hogs(hog_table)
    │
    ▼
هل gpiochip موجود بالفعل؟
    │
    ├──[نعم]──→ hog line فوراً
    │            gpiod_get() + gpiod_direction_output()
    │
    └──[لأ]───→ يُسجّل في global hog list
                    │
                    ▼
              عند gpiochip_add() لاحقاً
                    │
                    ▼
              الـ kernel يمسح الـ hog list
              ويعمل hog لأي line مطابقة
```

#### Software Node GPIO Lifecycle

```
[1] software_node_register_node_group(swnodes[])
    │  يسجّل nodes الـ controller والـ consumer
    │
    ▼
[2] platform_device_register()
    │  يربط device بـ software node عبر .fwnode
    │
    ▼
[3] driver probe() → gpiod_get()
    │  الـ GPIO subsystem يقرأ properties من software node
    │  يبحث عن PROPERTY_ENTRY_GPIO بالاسم المطلوب
    │  يطابق GPIO controller بـ node name == chip label
    │
    ▼
[4] يُرجع gpio_desc* ويشتغل زي أي GPIO عادي
```

---

### مخطط Call Flow

#### `gpiod_get()` — Platform Data Path

```
driver probe():
  gpiod_get(dev, "led", GPIOD_OUT_HIGH)
    │
    ▼
  gpiod_find()                         ← بيبحث في lookup tables
    │
    ├── يمشي على global gpio_lookup_list
    │     لكل gpiod_lookup_table:
    │       └── لو table->dev_id يطابق dev_name(dev):
    │             يمشي على table->table[]
    │             يطابق entry->con_id == "led" && entry->idx == 0
    │               → يُرجع chip label + hwnum + flags
    │
    ▼
  gpio_find_and_request()
    │
    ├── gpiochip_find(key)             ← يجيب الـ chip بالـ label
    ├── gpio_to_desc(chip, hwnum)      ← يحول رقم لـ descriptor
    ├── gpiod_request(desc)            ← يحجز الـ line (exclusive)
    └── gpio_set_config(flags)         ← يطبق ACTIVE_LOW/PULL_UP etc.
    │
    ▼
  gpiod_configure_flags(desc, GPIOD_OUT_HIGH)
    │
    ├── gpiod_direction_output(desc, 1)
    │     └── chip->ops->direction_output(chip, hwnum, value)
    │               ↓
    │         hardware register write
    │
    ▼
  يُرجع gpio_desc* للـ driver
```

#### `gpiod_get_array()` — Fast Bitmap Path

```
driver probe():
  gpiod_get_array(dev, "led", GPIOD_OUT_HIGH)
    │
    ▼
  يجمع كل GPIOs بالـ con_id "led" في gpio_descs
    │
    ▼
  هل الشروط متحققة؟
  (hwnum[0]==0, consecutive hwnum == consecutive idx)
    │
    ├──[نعم]── fast path:
    │           gpiod_set_array_value()
    │             → chip->ops->set_multiple(chip, mask, bits)
    │                   ↓
    │             write بيتم دفعة واحدة على الـ hardware
    │
    └──[لأ]─── slow path:
                for each desc:
                  gpiod_set_value(desc, val)
                    → chip->ops->set(chip, hwnum, val)
```

---

### لocking Strategy

الـ `board.rst` نفسه ما بيتكلمش عن الـ locking بشكل صريح، لكن الـ GPIO subsystem بيستخدم:

| Lock | يحمي إيه | أين |
|------|----------|-----|
| `gpio_lock` (spinlock) | `gpio_lookup_list` و `gpio_hog_list` | في `gpiolib.c` عند add/remove lookup tables |
| `gpio_desc->lock` (per-descriptor) | حالة الـ descriptor (direction، value، flags) | عند كل `gpiod_request()` و `gpiod_put()` |
| `gpiochip->bgpio_lock` | bank-level GPIO registers في generic implementations | عند `set_multiple()` في fast path |

**ترتيب الـ Locking:**
```
gpio_lock (global list lock)
  └── gpiochip lock (chip-level)
        └── bgpio_lock (register-level)
```

**ملاحظة الـ hog:** الـ hogging بيتم في سياق `gpiochip_add()` اللي بيشتغل في thread context، مش interrupt context، عشان كده مش بيحتاج spinlock خاص بيه.
## Phase 4: شرح الـ Functions

---

### ملخص عام — Cheatsheet

#### Registration & Mapping APIs

| Function / Macro | الغرض |
|---|---|
| `gpiod_add_lookup_table()` | تسجيل جدول mapping واحد في النظام |
| `gpiod_add_lookup_tables()` | تسجيل مصفوفة من جداول الـ mapping دفعة واحدة |
| `gpiod_remove_lookup_table()` | إزالة جدول mapping مسجَّل |
| `gpiod_add_hogs()` | تسجيل جدول GPIO hogs |
| `gpiod_remove_hogs()` | إزالة جدول GPIO hogs |
| `software_node_register_node_group()` | تسجيل مجموعة software nodes في الـ firmware node subsystem |

#### Consumer Acquisition APIs

| Function | الغرض |
|---|---|
| `gpiod_get()` | الحصول على GPIO descriptor بالاسم الوظيفي (index=0) |
| `gpiod_get_index()` | الحصول على GPIO descriptor بالاسم والـ index |
| `gpiod_get_optional()` | نفس `gpiod_get()` لكن بيرجع NULL لو مش موجود |
| `gpiod_get_index_optional()` | نفس `gpiod_get_index()` لكن بيرجع NULL لو مش موجود |
| `gpiod_get_array()` | الحصول على array كاملة من الـ GPIOs |
| `gpiod_get_array_optional()` | نفس السابق لكن optional |
| `gpiod_put()` | تحرير GPIO descriptor |
| `gpiod_put_array()` | تحرير array من الـ descriptors |
| `devm_gpiod_get()` | نفس `gpiod_get()` بإدارة devres |
| `devm_gpiod_get_index()` | نفس `gpiod_get_index()` بإدارة devres |
| `gpiod_count()` | عدد الـ GPIOs المتاحة لـ function معين |

#### Helper Macros

| Macro | الغرض |
|---|---|
| `GPIO_LOOKUP(key, hwnum, con_id, flags)` | تعريف lookup entry واحد (idx=0) |
| `GPIO_LOOKUP_IDX(key, hwnum, con_id, idx, flags)` | تعريف lookup entry مع index صريح |
| `GPIO_LOOKUP_SINGLE(name, dev_id, key, hwnum, con_id, flags)` | تعريف جدول lookup كامل بـ entry واحد |
| `GPIO_HOG(chip_label, hwnum, line_name, lflags, dflags)` | تعريف hog entry واحد |
| `PROPERTY_ENTRY_GPIO(name, chip_node, idx, flags)` | تعريف GPIO property في software node |

---

### Group 1: Registration & Lookup Table Management

الـ platform board code محتاج يربط كل GPIO بـ device ووظيفة محددة قبل ما أي driver يطلبه. الآليات دي بتتم عبر **lookup tables** بتتسجّل في kernel global list، وبعدين الـ `gpiod_get()` family بتبحث فيها.

---

#### `gpiod_add_lookup_table()`

```c
void gpiod_add_lookup_table(struct gpiod_lookup_table *table);
```

بتضيف جدول lookup واحد لـ global linked list اللي الـ GPIO subsystem بيبحث فيها. الـ table بتحتوي على اسم الـ device (`dev_id`) ومصفوفة من entries كل واحد بيحدد GPIO بعينه.

**Parameters:**
- `table` — pointer لـ `struct gpiod_lookup_table` المعرَّفة في board code، لازم تفضل valid طول فترة الاستخدام (عادةً static).

**Return value:** void — مفيش error path.

**Key details:**
- بتضيف الـ `table->list` لـ `gpio_lookup_list` باستخدام spinlock داخلي.
- الـ table لازم تنتهي بـ empty entry `{}` عشان الـ subsystem يعرف نهاية المصفوفة.
- لو الـ gpiochip موجود بالفعل وفيه hog entries مرتبطة، الـ hogging بيحصل فورًا.

**Who calls it:** Board init code، عادةً في `board_init()` أو `postcore_initcall`.

---

#### `gpiod_add_lookup_tables()`

```c
void gpiod_add_lookup_tables(struct gpiod_lookup_table **tables, size_t n);
```

بتسجّل مصفوفة من الـ lookup tables دفعة واحدة بدل ما تستدعي `gpiod_add_lookup_table()` لكل table على حدة. مناسبة لـ boards معقدة بيها كتير من الـ devices.

**Parameters:**
- `tables` — مصفوفة pointers على جداول الـ lookup.
- `n` — عدد الجداول في المصفوفة.

**Return value:** void.

**Key details:** Loop بسيط بيستدعي `gpiod_add_lookup_table()` لكل entry.

**Who calls it:** Board init code بدلًا من استدعاءات متعددة.

---

#### `gpiod_remove_lookup_table()`

```c
void gpiod_remove_lookup_table(struct gpiod_lookup_table *table);
```

بتشيل جدول lookup من الـ global list. مهمة لـ modules اللي بتسجّل lookup tables وعايزة تشيلها وقت unload.

**Parameters:**
- `table` — نفس الـ pointer اللي اتعمل `gpiod_add_lookup_table()` عليه.

**Return value:** void.

**Key details:**
- بتستخدم نفس الـ spinlock.
- لو الـ table فيها hogs، الـ hog lines مش بتتحرر تلقائيًا — الـ caller مسؤول.

**Who calls it:** Module cleanup code أو `board_exit()`.

---

### Group 2: GPIO Hog Management

الـ **GPIO hog** هو line بتاخده الـ kernel لنفسها مباشرةً من غير ما driver consumer يطلبه صراحةً. بيُستخدم لـ GPIO lines اللي بتحتاج تتسيطر عليها عند boot (زي reset lines أو power enables).

---

#### `gpiod_add_hogs()`

```c
void gpiod_add_hogs(struct gpiod_hog *hogs);
```

بتسجّل مصفوفة من الـ hog entries. لو الـ gpiochip المقابل موجود بالفعل وقت التسجيل، الـ hogging بيحصل فورًا. لو الـ chip جاي لاحقًا، الـ hogging بيحصل وقت ما الـ chip بيتسجّل.

**Parameters:**
- `hogs` — مصفوفة `struct gpiod_hog`، لازم تنتهي بـ empty entry `{}`.

**Return value:** void.

**Key details:**
- بتضيف الـ entries لـ `gpio_hog_list` المحمية بـ spinlock.
- بعد الإضافة بتعمل `gpiochip_walk_gpios()` أو ما يعادله عشان تطبّق الـ hogs على chips موجودة.
- الـ `dflags` بيحدد الاتجاه والقيمة (نفس `enum gpiod_flags`).

**Who calls it:** Board init code بعد ما تعرّف الـ `gpiod_hog` table.

```c
/* مثال عملي */
struct gpiod_hog gpio_hog_table[] = {
    GPIO_HOG("gpio.0", 10, "foo", GPIO_ACTIVE_LOW, GPIOD_OUT_HIGH),
    { }
};

gpiod_add_hogs(gpio_hog_table);
```

---

#### `gpiod_remove_hogs()`

```c
void gpiod_remove_hogs(struct gpiod_hog *hogs);
```

بتشيل hog entries من الـ global list. الـ GPIO lines المُشغّلة حاليًا بتتحرر.

**Parameters:**
- `hogs` — نفس الـ pointer اللي اتعمل `gpiod_add_hogs()` عليه.

**Return value:** void.

**Who calls it:** Module cleanup أو board teardown code.

---

### Group 3: Helper Macros للـ Lookup Table Definitions

الـ macros دي مش functions بالمعنى الحرفي، بس هي الأسلوب الوحيد المعتمد لتعريف الـ lookup entries. الـ kernel enforces عليهم عشان يضمن consistency.

---

#### `GPIO_LOOKUP_IDX`

```c
#define GPIO_LOOKUP_IDX(_key, _chip_hwnum, _con_id, _idx, _flags)  \
(struct gpiod_lookup) {                                             \
    .key = _key,                                                    \
    .chip_hwnum = _chip_hwnum,                                      \
    .con_id = _con_id,                                              \
    .idx = _idx,                                                    \
    .flags = _flags,                                                \
}
```

بيُنشئ `struct gpiod_lookup` literal بشكل inline. ده الـ macro الأساسي اللي كل الـ macros التانية بتعتمد عليه.

**Parameters:**
- `_key` — اسم الـ gpiochip label (مثلًا `"gpio.0"`) أو اسم الـ GPIO line. لو `chip_hwnum = U16_MAX` يبقى ده line name lookup.
- `_chip_hwnum` — رقم الـ GPIO داخل الـ chip (hardware number)، أو `U16_MAX` لو الـ key هو line name.
- `_con_id` — اسم الوظيفة من منظور الـ consumer driver (مثلًا `"led"`, `"power"`). ممكن يكون `NULL` يطابق أي consumer.
- `_idx` — الـ index داخل الـ function (زي index 0, 1, 2 للـ RGB LEDs).
- `_flags` — bitmask من `enum gpio_lookup_flags`.

---

#### `GPIO_LOOKUP`

```c
#define GPIO_LOOKUP(_key, _chip_hwnum, _con_id, _flags) \
    GPIO_LOOKUP_IDX(_key, _chip_hwnum, _con_id, 0, _flags)
```

Shortcut لـ `GPIO_LOOKUP_IDX` بـ `idx = 0`. مناسب للـ single GPIO per function.

---

#### `GPIO_LOOKUP_SINGLE`

```c
#define GPIO_LOOKUP_SINGLE(_name, _dev_id, _key, _chip_hwnum, _con_id, _flags) \
static struct gpiod_lookup_table _name = {              \
    .dev_id = _dev_id,                                  \
    .table = {                                          \
        GPIO_LOOKUP(_key, _chip_hwnum, _con_id, _flags),\
        {},                                             \
    },                                                  \
}
```

بيعرّف جدول lookup كامل بـ entry واحد بس كـ static variable. مناسب جدًا لـ simple devices بـ GPIO واحد.

**Key detail:** الناتج هو **definition** مش declaration، فمش بتحتاج تعمل `gpiod_add_lookup_table()` بشكل منفصل في بعض macros الـ helper.

---

#### `GPIO_HOG`

```c
#define GPIO_HOG(_chip_label, _chip_hwnum, _line_name, _lflags, _dflags) \
(struct gpiod_hog) {                                                      \
    .chip_label = _chip_label,                                            \
    .chip_hwnum = _chip_hwnum,                                            \
    .line_name = _line_name,                                              \
    .lflags = _lflags,                                                    \
    .dflags = _dflags,                                                    \
}
```

بيُنشئ `struct gpiod_hog` literal. الـ `lflags` للـ line properties (active-low, open-drain إلخ) والـ `dflags` للاتجاه والقيمة الابتدائية.

---

#### `PROPERTY_ENTRY_GPIO`

```c
#define PROPERTY_ENTRY_GPIO(_name_, _chip_node_, _idx_, _flags_) \
    PROPERTY_ENTRY_REF(_name_, _chip_node_, _idx_, _flags_)
```

بيضيف GPIO reference كـ firmware property في software node. الـ `_chip_node_` هو pointer لـ `struct software_node` بتاع الـ GPIO controller، والـ `_name_` هو اسم الـ property (زي `"gpios"`).

**Key detail:** الـ software node بتاع الـ controller لازم يكون مسجّل وله `name` يطابق الـ controller label.

---

### Group 4: Consumer Acquisition APIs

الـ consumer driver بيطلب GPIO عن طريق `gpiod_get()` family. الـ functions دي بتبحث في الـ DT / ACPI / software nodes / platform lookup tables وبترجع `struct gpio_desc *`.

---

#### `gpiod_get()`

```c
struct gpio_desc *__must_check gpiod_get(struct device *dev,
                                         const char *con_id,
                                         enum gpiod_flags flags);
```

بتطلب أول GPIO مربوط بـ `con_id` لهذا الـ device. الـ `__must_check` بيجبر الـ caller يتحقق من الـ return value.

**Parameters:**
- `dev` — الـ device المالك للـ GPIO.
- `con_id` — اسم الوظيفة (مثلًا `"led"`, `"power"`). الـ subsystem بيحوّله داخليًا لـ `"<con_id>-gpios"` أو `"<con_id>-gpio"` في البحث.
- `flags` — `enum gpiod_flags`: `GPIOD_IN`, `GPIOD_OUT_LOW`, `GPIOD_OUT_HIGH`, إلخ.

**Return value:**
- Pointer صحيح لـ `gpio_desc` في حالة النجاح.
- `ERR_PTR(-ENOENT)` لو مش موجود.
- `ERR_PTR(-EPROBE_DEFER)` لو الـ chip لسه متسجلش.
- أخطاء تانية حسب الـ chip driver.

**Key details:**
- Equivalent لـ `gpiod_get_index(dev, con_id, 0, flags)`.
- الـ active-low handling بيتحكم فيه الـ mapping وتلقائيًا شفاف للـ consumer.
- بيحجز الـ GPIO (request)، فلازم تتبعها بـ `gpiod_put()`.

**Who calls it:** Driver `probe()` function.

```
gpiod_get()
    → lookup in DT / ACPI / swnode / platform table
    → gpiochip_find() + gpio_request()
    → apply direction/value from flags
    → return gpio_desc *
```

---

#### `gpiod_get_index()`

```c
struct gpio_desc *__must_check gpiod_get_index(struct device *dev,
                                               const char *con_id,
                                               unsigned int idx,
                                               enum gpiod_flags flags);
```

بتطلب GPIO محدد بـ index داخل function. بتستخدم لما في أكتر من GPIO بنفس الوظيفة (مثلًا RGB LEDs).

**Parameters:**
- `dev` — الـ device المالك.
- `con_id` — اسم الوظيفة.
- `idx` — الـ index (0-based) داخل الـ function.
- `flags` — نفس `enum gpiod_flags`.

**Return value:** نفس `gpiod_get()`.

**Who calls it:** Driver `probe()` لما يحتاج GPIOs متعددة بنفس الاسم الوظيفي.

```c
/* مثال: RGB LED */
red   = gpiod_get_index(dev, "led", 0, GPIOD_OUT_HIGH);
green = gpiod_get_index(dev, "led", 1, GPIOD_OUT_HIGH);
blue  = gpiod_get_index(dev, "led", 2, GPIOD_OUT_HIGH);
```

---

#### `gpiod_get_optional()` و `gpiod_get_index_optional()`

```c
struct gpio_desc *__must_check gpiod_get_optional(struct device *dev,
                                                  const char *con_id,
                                                  enum gpiod_flags flags);

struct gpio_desc *__must_check gpiod_get_index_optional(struct device *dev,
                                                        const char *con_id,
                                                        unsigned int index,
                                                        enum gpiod_flags flags);
```

نفس `gpiod_get()` و `gpiod_get_index()` لكن بيرجعوا `NULL` بدل `ERR_PTR(-ENOENT)` لو الـ GPIO مش موجود في الـ board description. مناسبة للـ optional hardware features.

**Return value:**
- Pointer صحيح لو موجود.
- `NULL` لو مش موجود (لا خطأ).
- `ERR_PTR` لأي خطأ تاني (زي `-EPROBE_DEFER`).

**Key detail:** الـ driver بيتحقق بـ `IS_ERR_OR_NULL()` مش `IS_ERR()` فقط.

---

#### `gpiod_get_array()` و `gpiod_get_array_optional()`

```c
struct gpio_descs *__must_check gpiod_get_array(struct device *dev,
                                                const char *con_id,
                                                enum gpiod_flags flags);

struct gpio_descs *__must_check gpiod_get_array_optional(struct device *dev,
                                                         const char *con_id,
                                                         enum gpiod_flags flags);
```

بتطلب كل الـ GPIOs المربوطة بـ `con_id` في array واحدة من نوع `struct gpio_descs`. بتسمح بـ bulk operations وممكن تستفيد من الـ fast bitmap path لو الـ array اتحققت شروطه.

**Parameters:**
- `dev`, `con_id`, `flags` — نفس السابق.

**Return value:**
- `struct gpio_descs *` بيحتوي على `ndescs` و array من الـ descriptors.
- `ERR_PTR` في حالة الخطأ.
- `NULL` للـ optional version لو مفيش GPIOs.

**Key details:**
- الـ `info` field بيحتوي على opaque `gpio_array` structure بتوفّر fast path metadata.
- شروط الـ fast bitmap path: الـ hwnum بتاع index 0 = 0، والـ hwnum لكل member = الـ array index بتاعه، وكلهم على نفس الـ chip.
- الـ open-drain وopen-source pins بتتستبعد من الـ fast output path.

**Pseudocode:**
```
gpiod_get_array()
    → count GPIOs matching con_id
    → allocate gpio_descs + gpio_array
    → for each GPIO: gpiod_get_index()
    → analyze array layout → set fast_path flag
    → return gpio_descs
```

---

#### `gpiod_put()` و `gpiod_put_array()`

```c
void gpiod_put(struct gpio_desc *desc);
void gpiod_put_array(struct gpio_descs *descs);
```

بيحرروا الـ GPIO descriptor(s) المُطلوبة. لازم يتم استدعاؤهم في `remove()` أو error path بتاع `probe()` لأي GPIO أتطلب بـ non-devm functions.

**Parameters:**
- `desc` / `descs` — الـ descriptor(s) المراد تحريره.

**Key details:**
- `might_sleep()` — مش آمن من interrupt context.
- `gpiod_put_array()` بتحرر كل descriptor في المصفوفة ثم الـ `gpio_descs` struct نفسه.

---

### Group 5: Managed (devres) Variants

كل الـ devm_gpiod_* functions بتشتغل زي counterparts غير الـ devm، بس بتسجّل release action في الـ device's devres list عشان الـ GPIO يتحرر تلقائيًا لما الـ device يتشال.

| Managed Function | Counterpart |
|---|---|
| `devm_gpiod_get()` | `gpiod_get()` |
| `devm_gpiod_get_index()` | `gpiod_get_index()` |
| `devm_gpiod_get_optional()` | `gpiod_get_optional()` |
| `devm_gpiod_get_index_optional()` | `gpiod_get_index_optional()` |
| `devm_gpiod_get_array()` | `gpiod_get_array()` |
| `devm_gpiod_get_array_optional()` | `gpiod_get_array_optional()` |
| `devm_gpiod_put()` | `gpiod_put()` (early release) |
| `devm_gpiod_put_array()` | `gpiod_put_array()` (early release) |

**Key detail:** `devm_gpiod_unhinge()` بتفصل الـ GPIO عن الـ devres binding من غير ما تحرره — مفيدة لو الـ driver عايز يتحكم يدويًا في الـ lifetime بعد ما طلبه بـ devm.

---

### Group 6: Software Node APIs

الـ **software node** بيسمح لـ board code يوصف الـ hardware في C بدون DT أو ACPI. الـ GPIO binding فيه بيشتغل عن طريق `PROPERTY_ENTRY_GPIO()` اللي بيربط الـ consumer بالـ GPIO controller node.

#### `software_node_register_node_group()`

```c
int software_node_register_node_group(const struct software_node **node_group);
```

بتسجّل مجموعة software nodes بالترتيب. لازم node الـ GPIO controller يتسجّل قبل أي consumer بيرفر إليه.

**Parameters:**
- `node_group` — NULL-terminated مصفوفة pointers لـ `struct software_node`.

**Return value:** `0` للنجاح، أو error code.

**Key details:**
- الـ node بتاع الـ GPIO controller لازم `name` بتاعه يطابق الـ `label` بتاع الـ gpiochip بالضبط.
- الـ node ممكن يكون standalone (مش محتاج يكون attached للـ controller device مباشرةً).

**Pseudocode:**
```c
/* Setup flow كامل */

// 1. Controller node
static const struct software_node gpio_ctrl_node = {
    .name = "gpio-foo",  /* يطابق gpiochip label */
};

// 2. Consumer properties
static const struct property_entry led_props[] = {
    PROPERTY_ENTRY_GPIO("gpios", &gpio_ctrl_node, 42, GPIO_ACTIVE_HIGH),
    { }
};

// 3. Consumer node
static const struct software_node led_node = {
    .name = "status-led",
    .properties = led_props,
};

// 4. Register
const struct software_node *nodes[] = {
    &gpio_ctrl_node,
    &led_node,
    NULL
};
software_node_register_node_group(nodes);

// 5. Driver بعدين بيعمل:
gpiod_get(dev, "gpios", GPIOD_OUT_HIGH);
```

---

### Group 7: Lookup Mechanism — كيف بتشتغل الـ APIs داخليًا

فهم الـ lookup flow مهم لـ debugging.

```
gpiod_get(dev, "led", GPIOD_OUT_HIGH)
         │
         ▼
    gpiod_find()  ← بتبحث بالترتيب:
         │
         ├─ 1. Device Tree / ACPI fwnode
         │       → ابحث عن "led-gpios" أو "led-gpio" في device node
         │
         ├─ 2. Software Node (swnode)
         │       → نفس الـ property lookup على software_node properties
         │
         └─ 3. Platform Lookup Table (gpio_lookup_list)
                 → ابحث عن entry بـ:
                   table->dev_id  == dev_name(dev)   (أو NULL)
                   entry->con_id  == "led"            (أو NULL)
                   entry->idx     == 0
                 → لو key هو chip label: gpiochip_find(key)
                 → لو key هو line name: gpiochip_find_by_line_name(key)
```

**النقطة المهمة:** الـ `con_id` في `gpiod_get()` بيتحوّل داخليًا لـ `"<con_id>-gpios"` أو `"<con_id>-gpio"` في الـ DT/ACPI lookup باستخدام `snprintf(... "%s-%s", con_id, gpio_suffixes[])`. لكن في الـ platform lookup table، الـ `con_id` بيتقارن مباشرةً بدون suffix.

---

### Group 8: Arrays of Pins — Fast Bitmap Path

الـ fast path بيسمح بـ bulk get/set operations بدون overhead لكل pin على حدة، عن طريق تمرير bitmap مباشرةً لـ `gpiochip->get_multiple()` / `gpiochip->set_multiple()`.

#### شروط التأهل للـ Fast Path

```
Array[0].hwnum == 0
    AND
Array[i].hwnum == i  (for all i on same chip as [0])
    AND
Array[i] on same chip as Array[0]
```

```
Fast Path Layout Example:
─────────────────────────────────
Index:   0    1    2    3
HW num:  0    1    2    3    ✓ Fast path eligible
Chip:    A    A    A    A
─────────────────────────────────

Non-eligible:
Index:   0    1    2    3
HW num:  0    1    5    3    ✗ hwnum[2] ≠ index[2]
─────────────────────────────────

Mixed chips:
Index:   0    1    2    3
HW num:  0    1    0    1    ✗ chips[2,3] ≠ chip[0]
Chip:    A    A    B    B
         ↑fast↑   ↑slow↑
─────────────────────────────────
```

**الـ pins المُستبعدة دائمًا من fast output:** open-drain وopen-source pins حتى لو الـ array متأهلة للـ fast path بشكل عام.

---

### الـ Flags — مرجع سريع

#### `enum gpio_lookup_flags` (في mapping time)

| Flag | القيمة | المعنى |
|---|---|---|
| `GPIO_ACTIVE_HIGH` | 0 | Active high (default) |
| `GPIO_ACTIVE_LOW` | bit 0 | Active low — transparent للـ consumer |
| `GPIO_OPEN_DRAIN` | bit 1 | Open drain configuration |
| `GPIO_OPEN_SOURCE` | bit 2 | Open source configuration |
| `GPIO_PERSISTENT` | 0 | يحتفظ بقيمته عند suspend/resume |
| `GPIO_TRANSITORY` | bit 3 | قد يفقد قيمته عند suspend/resume |
| `GPIO_PULL_UP` | bit 4 | Enable internal pull-up |
| `GPIO_PULL_DOWN` | bit 5 | Enable internal pull-down |
| `GPIO_PULL_DISABLE` | bit 6 | Disable pull resistor |

#### `enum gpiod_flags` (في acquisition time)

| Flag | المعنى |
|---|---|
| `GPIOD_ASIS` | لا تغيّر الاتجاه أو القيمة |
| `GPIOD_IN` | اضبط كـ input |
| `GPIOD_OUT_LOW` | اضبط كـ output، قيمة منطقية = 0 |
| `GPIOD_OUT_HIGH` | اضبط كـ output، قيمة منطقية = 1 |
| `GPIOD_OUT_LOW_OPEN_DRAIN` | open-drain output منخفض |
| `GPIOD_OUT_HIGH_OPEN_DRAIN` | open-drain output مرتفع |
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بالـ GPIO Mappings

الـ **debugfs** بيوفر معلومات مفصلة عن الـ gpiochips والـ line mappings.

```bash
# Mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# اعرض كل الـ GPIO chips والـ lines المسجّلة
cat /sys/kernel/debug/gpio

# مثال على الـ output
# gpiochip0: GPIOs 0-31, parent: platform/gpio.0, gpio-foo:
#  gpio-1   (power               ) out lo  ACTIVE LOW
#  gpio-15  (led                 ) out hi
#  gpio-16  (led                 ) out hi
#  gpio-17  (led                 ) out hi
```

**تفسير الـ output:**
- `out lo ACTIVE LOW` → الـ line mapped كـ `GPIO_ACTIVE_LOW`، القيمة الفيزيائية 0
- `out hi` → الـ line active high، القيمة الفيزيائية 1
- لو شفت الـ line بدون اسم → مش مـ mapped لأي consumer

```bash
# اعرض lookup tables المسجّلة (kernel >= 5.15 مع CONFIG_GPIO_CDEV)
cat /sys/kernel/debug/gpio-lookups 2>/dev/null || echo "not available"

# اعرض الـ GPIO hogs
grep -r "" /sys/kernel/debug/gpio 2>/dev/null | grep -i hog
```

---

#### 2. مدخلات الـ sysfs

```bash
# اعرض كل الـ gpiochips المتاحة
ls /sys/bus/gpio/devices/

# اعرض معلومات gpiochip معين
cat /sys/class/gpio/gpiochip0/label    # اسم الـ chip (المهم في GPIO_LOOKUP key)
cat /sys/class/gpio/gpiochip0/base     # أول رقم GPIO في الـ chip
cat /sys/class/gpio/gpiochip0/ngpio   # عدد الـ lines

# لو عارف رقم GPIO الفيزيائي، export عشان تقرأه
echo 15 > /sys/class/gpio/export
cat /sys/class/gpio/gpio15/direction   # in أو out
cat /sys/class/gpio/gpio15/value       # 0 أو 1
cat /sys/class/gpio/gpio15/active_low  # 1 = active low

# GPIO character device (الطريقة الحديثة)
gpioinfo gpiochip0   # يحتاج libgpiod
gpiodetect           # يعرض كل الـ chips
```

**ملاحظة مهمة:** الـ `sysfs GPIO interface` deprecated في kernels حديثة، استخدم `libgpiod` بدلها.

---

#### 3. الـ ftrace — Tracepoints والـ Events

```bash
# Enable GPIO tracepoints
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو enable events محددة
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الكود اللي بيستخدم GPIO، بعدين اقرأ
cat /sys/kernel/debug/tracing/trace

# مثال على output
# <...>-123 [000] .... gpiod_direction_output: gpio=15 value=1
# <...>-123 [000] .... gpio_value: gpio=15 get=0 value=1

# Filter على chip معين
echo 'gpio >= 15 && gpio <= 17' > /sys/kernel/debug/tracing/events/gpio/gpio_value/filter
```

للـ **function tracing** على دوال gpiolib:

```bash
# Trace دوال gpiod_get و gpiod_set_value
echo 'gpiod_get gpiod_get_index gpiod_set_value gpiod_direction_output' \
    > /sys/kernel/debug/tracing/set_ftrace_filter

echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

---

#### 4. الـ printk والـ Dynamic Debug

تفعيل الـ **dynamic debug** لـ subsystem الـ GPIO:

```bash
# Enable كل debug messages في gpio subsystem
echo 'file drivers/gpio/* +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/gpiolib.c +p' > /sys/kernel/debug/dynamic_debug/control

# Enable الـ lookup table messages تحديداً
echo 'file drivers/gpio/gpiolib-devprop.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/gpiolib-acpi.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/gpiolib-of.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages
dmesg -w | grep -i gpio

# Enable في kernel cmdline (للـ early boot debugging)
# dyndbg="file drivers/gpio/* +p"
```

لو محتاج **verbose lookup** أثناء الـ boot:

```bash
# في /etc/default/grub
GRUB_CMDLINE_LINUX="gpio.debug=1 dyndbg='file drivers/gpio/* +pflmt'"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_GPIOLIB` | الـ GPIO library الأساسية — لازم تكون enabled |
| `CONFIG_DEBUG_GPIO` | يفعّل debug messages في gpiolib |
| `CONFIG_GPIO_CDEV` | الـ character device interface (الحديث) |
| `CONFIG_GPIO_CDEV_V1` | backward compat مع V1 API |
| `CONFIG_GPIO_SYSFS` | الـ sysfs interface (legacy) |
| `CONFIG_GPIO_VIRTUSER` | virtual GPIO user للـ testing |
| `CONFIG_GPIO_SIM` | simulated GPIO chip للـ unit tests |
| `CONFIG_GPIOLIB_FASTPATH_LIMIT` | عدد الـ GPIOs في fast path (default 512) |
| `CONFIG_GENERIC_PINCONF` | debugging للـ pin configuration |
| `CONFIG_PINCTRL_SINGLE` | single pinctrl للـ simple boards |
| `CONFIG_SOFTIRQ_STAT` | لو الـ GPIO interrupts بتتأخر |
| `CONFIG_DEBUG_SHIRQ` | debug للـ shared IRQs |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'GPIO|PINCTRL' | grep -v '^#'
# أو
grep -E 'CONFIG_(GPIO|PINCTRL)' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**libgpiod tools** (الأدوات الحديثة البديلة عن sysfs):

```bash
# اعرض كل الـ GPIO chips
gpiodetect

# اعرض كل الـ lines في chip معين مع أسمائها
gpioinfo gpiochip0

# اقرأ قيمة GPIO
gpioget gpiochip0 15

# اكتب قيمة GPIO
gpioset gpiochip0 15=1

# Monitor تغيير قيمة GPIO
gpiomon gpiochip0 15

# مثال على output من gpioinfo
# gpiochip0 - 32 lines:
#         line   0:      unnamed       unused   input  active-high
#         line   1:      "power"    "foo.0"  output  active-low [used]
#         line  15:        "led"    "foo.0"  output  active-high [used]
```

**للـ DT debugging:**

```bash
# اعرض الـ DT node للـ device
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "foo_device"

# أو
cat /sys/firmware/devicetree/base/foo_device/led-gpios | xxd
```

**للـ ACPI debugging:**

```bash
# اعرض ACPI tables
acpidump -b
iasl -d DSDT.dat

# اعرض GPIO properties من ACPI
cat /sys/bus/acpi/devices/*/path
ls /sys/bus/acpi/devices/ACPI0000:00/
```

**للـ Software Nodes:**

```bash
# اعرض registered software nodes
ls /sys/kernel/debug/software_nodes/ 2>/dev/null

# أو من sysfs
find /sys -name "fwnode" 2>/dev/null
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `gpiod_get: no GPIO consumer led found` | مفيش mapping للـ function "led" | تحقق من DT property `led-gpios` أو lookup table |
| `gpiod_get: GPIO chip not found` | الـ key في `GPIO_LOOKUP` مش مطابق لـ label الـ chip | تحقق من `cat /sys/class/gpio/gpiochipX/label` |
| `GPIO line 15 already in use` | الـ GPIO محجوز من driver تاني أو hog | تحقق من `gpioinfo` — شوف الـ consumer |
| `gpio-15 (led): gpiochip_request failed` | الـ chip driver رفض الـ request | تحقق من الـ pinmux / pinctrl conflicts |
| `could not find GPIO chip 'gpio.0'` | الـ gpiochip لسه مش registered | تأكد من الـ probe order، استخدم deferred probe |
| `invalid GPIO flags 0x...` | flags غلط في `GPIO_LOOKUP` | راجع `enum gpio_lookup_flags` |
| `chip_hwnum 65535 (U16_MAX): no matching GPIO line name` | استخدمت `U16_MAX` كـ chip_hwnum بس مفيش line باسم الـ key | تأكد إن الـ GPIO line name موجود في الـ chip |
| `GPIO array: fast bitmap path disabled` | الـ array مش في hardware order | رتّب الـ GPIOs بحيث hwnum = index |
| `gpio hog: gpiochip 'gpio.0' not found, deferring` | الـ chip لسه مش loaded وقت الـ hog registration | عادي — الـ kernel بيعمل retry تلقائياً |
| `ACPI: Failed to translate GPIO resource` | ACPI _CRS فيها GPIO لكن مفيش controller مناسب | تحقق من الـ `_DSD` properties وإن الـ GPIO controller registered |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

ضيف الـ checks دي في drivers أثناء التطوير:

```c
#include <linux/gpio/consumer.h>

/* في gpiod_get() error path */
struct gpio_desc *desc = gpiod_get(dev, "led", GPIOD_OUT_HIGH);
if (IS_ERR(desc)) {
    /* بدلاً من dev_err فقط، أضف stack trace */
    dev_err(dev, "Failed to get LED GPIO: %ld\n", PTR_ERR(desc));
    dump_stack();  /* مفيد لمعرفة مين بيطلب الـ GPIO */
    return PTR_ERR(desc);
}

/* تحقق إن الـ GPIO مش NULL قبل الاستخدام */
WARN_ON(!desc);
WARN_ON(IS_ERR(desc));

/* تحقق من active_low flag */
WARN_ON(gpiod_is_active_low(desc) != expected_polarity);

/* في lookup table registration */
WARN_ON(gpios_table.dev_id == NULL && con_id == NULL);
```

```c
/* في gpio chip driver — تحقق من hwnum */
WARN_ON(chip_hwnum >= chip->ngpio);

/* في array fast-path — تحقق من الـ order */
#ifdef CONFIG_DEBUG_GPIO
for (int i = 1; i < array_size; i++) {
    WARN_ON(descs[i]->gdev != descs[0]->gdev);
}
#endif
```

---

### Hardware Level

---

#### 1. مقارنة الـ Hardware State بالـ Kernel State

```bash
# اقرأ قيمة GPIO من kernel
gpioget gpiochip0 15
# القيمة دي logical (بتأخذ active_low في الاعتبار)

# اقرأ من debugfs — القيمة الفيزيائية
cat /sys/kernel/debug/gpio | grep "gpio-15"
# gpio-15  (led) out hi  ← هنا "hi" = الـ physical level

# لو الـ GPIO active_low والـ kernel بيقول "out hi"
# → الـ physical pin = 0V (low)
# → الـ logical value = 1 (active)
```

**جدول العلاقة بين الـ logical والـ physical:**

| Mapping | Logical Value | Physical Voltage |
|---|---|---|
| `GPIO_ACTIVE_HIGH` | 1 | High (3.3V / 5V) |
| `GPIO_ACTIVE_HIGH` | 0 | Low (0V) |
| `GPIO_ACTIVE_LOW` | 1 | Low (0V) |
| `GPIO_ACTIVE_LOW` | 0 | High (3.3V / 5V) |

---

#### 2. Register Dump Techniques

```bash
# devmem2 — اقرأ GPIO controller registers مباشرة
# مثال: BCM2835 GPIO base = 0x3F200000
devmem2 0x3F200000 w   # GPFSEL0 — function select
devmem2 0x3F200034 w   # GPLEV0 — current level للـ GPIOs 0-31
devmem2 0x3F200038 w   # GPLEV1 — current level للـ GPIOs 32-53

# io tool (من package iotools)
io -4 0x3F200034        # قرأ 4 bytes من العنوان

# /dev/mem (يحتاج root + CONFIG_STRICT_DEVMEM=n)
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 0x1000, offset=0x3F200000)
    val = struct.unpack('<I', m[0x34:0x38])[0]
    print(f'GPLEV0: 0x{val:08X}')
    # bit 15 = GPIO 15 current physical level
    print(f'GPIO15 physical level: {(val >> 15) & 1}')
"

# iomem — اعرض الـ memory mapped regions
cat /proc/iomem | grep -i gpio
```

**ملاحظة:** لازم تعرف base address الـ GPIO controller من الـ datasheet أو من:

```bash
cat /proc/iomem | grep -i gpio
# أو من DT
grep -r "reg = " /sys/firmware/devicetree/base/ 2>/dev/null | grep gpio
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**للـ Logic Analyzer:**

- اضبط الـ trigger على rising/falling edge على الـ GPIO pin
- فعّل الـ kernel tracing في نفس الوقت وقارن timestamps
- لو الـ GPIO active-low: الـ "active state" هو الـ low pulse — اضبط الـ trigger على falling edge

```
GPIO_ACTIVE_HIGH (led):
    Kernel: gpiod_set_value(led, 1)
    Pin:    _____|‾‾‾‾‾‾‾‾‾|_____
                 ↑ يجب أن ترى rising edge هنا

GPIO_ACTIVE_LOW (power):
    Kernel: gpiod_set_value(power, 1)  [يعني "enable" منطقياً]
    Pin:    ‾‾‾‾‾|_________|‾‾‾‾‾
                 ↑ falling edge! (active = low physically)
```

**للـ Oscilloscope:**
- افحص الـ voltage level: 3.3V أو 5V حسب الـ VIO الخاص بالـ controller
- افحص الـ rise/fall time — لو slow جداً: مشكلة في الـ pull resistor
- افحص الـ glitches عند الـ direction change (input → output)

---

#### 4. المشاكل الشائعة وأنماطها في الـ Kernel Log

| المشكلة الـ Hardware | النمط في dmesg | التفسير |
|---|---|---|
| GPIO controller مش initialized | `gpiod_get: no GPIO consumer found` في early boot | الـ chip probe جاي بعد الـ consumer — استخدم deferred probe |
| خطأ في الـ active-low wiring | الـ LED شتغل بـ `gpiod_set_value(0)` بدل `1` | غلط في `GPIO_ACTIVE_HIGH/LOW` في الـ lookup |
| Pin conflict مع pinctrl | `pin X is already requested` | الـ pinmux بيستخدم نفس الـ pin لـ function تانية |
| Pull resistor خاطئ | GPIO يقرأ random values وهو input | مفيش internal pull، الـ pin floating |
| Open drain خاطئ | `GPIO_OPEN_DRAIN` set بس الـ hardware مش open drain | الـ pin مش بيوصل لـ high صح |
| Voltage level mismatch | GPIO يقرأ دايماً 0 أو دايماً 1 | 5V signal على 3.3V GPIO — hardware damage محتمل |

---

#### 5. Device Tree Debugging

```bash
# تحقق إن الـ DT node موجود وصح
ls /sys/firmware/devicetree/base/
find /sys/firmware/devicetree/base -name "compatible" -exec \
    sh -c 'echo "=== $1 ==="; cat $1' _ {} \; 2>/dev/null | grep -B1 "acme,foo"

# اقرأ الـ led-gpios property raw
hexdump -C /sys/firmware/devicetree/base/foo_device/led-gpios
# Output يكون: phandle (4 bytes) + gpio_num (4 bytes) + flags (4 bytes)
# 00000000  00 00 00 01  00 00 00 0f  00 00 00 00  ← gpio 0x0f = 15, flags=0 (ACTIVE_HIGH)

# تحقق من الـ GPIO controller node نفسه
cat /sys/firmware/devicetree/base/gpio@3f200000/gpio-controller
cat /sys/firmware/devicetree/base/gpio@3f200000/\#gpio-cells

# compile الـ DT من /sys وافحصه
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null > /tmp/current.dts
grep -A 30 "foo_device" /tmp/current.dts

# تحقق إن الـ GPIO suffix صح ("gpios" مش "gpio")
# الـ kernel بيبحث عن: <con_id>-gpios و <con_id>-gpio
grep "gpios\|gpio" /tmp/current.dts | grep -i led
```

**تحقق من الـ con_id matching:**

```bash
# الـ kernel code بيعمل:
# snprintf(propname, "%s-%s", con_id, "gpios")
# يعني con_id="led" → يبحث عن "led-gpios"
# لو الـ DT عنده "led-gpio" (بدون s) → بيشتغل كـ fallback بس deprecated

# تحقق من الـ property names في DT
find /sys/firmware/devicetree/base -name "*gpio*" 2>/dev/null
```

---

### Practical Commands

---

#### مرجع سريع — Copy-Paste Commands

**1. فحص الـ GPIO Mappings الكاملة:**

```bash
# الفحص الأساسي الأول دايماً
cat /sys/kernel/debug/gpio && echo "---" && gpiodetect && gpioinfo
```

**2. تتبع gpiod_get failures:**

```bash
# Enable dynamic debug وراقب
echo 'file drivers/gpio/gpiolib.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/gpiolib-of.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/gpiolib-acpi.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
dmesg -w | grep -E 'gpio|GPIO'
```

**3. فحص lookup table registration:**

```bash
# Enable tracing ثم reload الـ module
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
modprobe -r foo_driver && modprobe foo_driver
cat /sys/kernel/debug/tracing/trace | grep -i "lookup\|gpio"
echo 0 > /sys/kernel/debug/tracing/events/gpio/enable
```

**4. تحقق من active-low صح:**

```bash
# اقرأ من debugfs واعرض physical state
python3 -c "
import subprocess
out = subprocess.check_output(['cat', '/sys/kernel/debug/gpio']).decode()
for line in out.split('\n'):
    if 'gpio-' in line:
        parts = line.strip().split()
        print(line.strip())
"

# أو أسرع
grep -E 'gpio-[0-9]+' /sys/kernel/debug/gpio | awk '{
    pin=$1; name=$2; dir=$4; level=$5; flags=$6
    print pin, name, "direction:", dir, "physical:", level, flags
}'
```

**5. اختبار GPIO بدون driver:**

```bash
# Export GPIO واختبره يدوياً (legacy sysfs)
GPIO=15
echo $GPIO > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio${GPIO}/direction
echo 1 > /sys/class/gpio/gpio${GPIO}/value
cat /sys/class/gpio/gpio${GPIO}/value      # يجب أن يرجع 1
echo $GPIO > /sys/class/gpio/unexport

# الطريقة الحديثة بـ libgpiod
gpioset --mode=signal gpiochip0 15=1
# اضغط Ctrl+C لتحرير الـ line
```

**6. فحص Software Nodes:**

```bash
# تحقق من registration
dmesg | grep -i "software_node\|swnode"

# اعرض الـ nodes المسجّلة
find /sys -name "*.properties" 2>/dev/null
ls /sys/firmware/swnode/ 2>/dev/null
```

**7. تحقق من GPIO Hogs:**

```bash
# الـ hogs تظهر في debugfs كـ "kernel" consumer
cat /sys/kernel/debug/gpio | grep kernel
# gpio-10  (foo                 ) out lo  ACTIVE LOW  [kernel]

# لو الـ hog مش ظاهر — enable debug
echo 'file drivers/gpio/gpiolib.c +p' > /sys/kernel/debug/dynamic_debug/control
dmesg | grep -i hog
```

**8. Platform Data Verification Script:**

```bash
#!/bin/bash
# verify-gpio-mapping.sh — تحقق من GPIO mapping لـ device معين
DEVICE="foo.0"
FUNCTION="led"

echo "=== Checking GPIO mappings for $DEVICE/$FUNCTION ==="
echo ""
echo "--- sysfs gpio chips ---"
for chip in /sys/class/gpio/gpiochip*; do
    label=$(cat $chip/label 2>/dev/null)
    base=$(cat $chip/base 2>/dev/null)
    ngpio=$(cat $chip/ngpio 2>/dev/null)
    echo "  $chip: label='$label' base=$base ngpio=$ngpio"
done

echo ""
echo "--- debugfs gpio state ---"
cat /sys/kernel/debug/gpio 2>/dev/null | grep -E "$FUNCTION|$DEVICE" || \
    echo "  [no entries found for $FUNCTION or $DEVICE]"

echo ""
echo "--- dmesg gpio errors ---"
dmesg | grep -i "gpio" | grep -iE "error|fail|not found|invalid" | tail -20
```

**9. مثال على output صح ومفسَّر:**

```
$ cat /sys/kernel/debug/gpio
gpiochip0: GPIOs 0-31, parent: platform/gpio.0, gpio-foo:            ← label يطابق key في GPIO_LOOKUP
 gpio-1   (power               ) out lo  ACTIVE LOW  foo.0           ← active-low، physical=0، logical=1 (enabled)
 gpio-15  (led                 ) out hi              foo.0           ← active-high، physical=1، logical=1 (on)
 gpio-16  (led                 ) out hi              foo.0
 gpio-17  (led                 ) out hi              foo.0
```

```
$ gpioinfo gpiochip0
gpiochip0 - 32 lines:
        line   0:      unnamed       unused   input  active-high
        line   1:       "power"     "foo.0"  output   active-low [used]  ← هنا "used" = مأخوذ من driver
        line  15:         "led"     "foo.0"  output  active-high [used]
        line  16:         "led"     "foo.0"  output  active-high [used]
        line  17:         "led"     "foo.0"  output  active-high [used]
```

**لو "unused" بدل "used":** الـ driver مش بيعمل `gpiod_get()` أو في خطأ في الـ lookup table.
**لو "unnamed":** الـ GPIO مش mapped لأي function.
**لو الـ line مش ظاهرة خالص:** الـ hwnum أكبر من `ngpio` للـ chip أو الـ chip label غلط.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ RS485 Enable لا يشتغل

#### العنوان
GPIO_ACTIVE_LOW مفهوم غلط في lookup table بيخلي الـ RS485 transceiver مايتفعلش

#### السياق
شركة بتعمل industrial gateway بيستخدم RK3562 SoC. الـ gateway عنده RS485 interface بيتحكم فيه driver عن طريق GPIO بيتصل بـ DE/RE pin في الـ transceiver (مثلاً MAX485). الـ board لا بيستخدم device tree ولا ACPI — بيستخدم legacy platform data لأن الـ BSP قديم ومعتمد على board file.

#### المشكلة
الـ RS485 مبيرسلش أي data. الـ UART بيشتغل ويبعت bytes لكن مفيش حاجة بتوصل على الـ bus. الـ transceiver في mode "receive" طول الوقت. المهندس يقيس على الـ DE/RE pin يلاقيه دايماً LOW.

#### التحليل
الـ driver بيعمل `gpiod_set_value(de_gpio, 1)` عشان يفعّل الـ transmit mode. بس لو الـ lookup table عرّف الـ GPIO بـ `GPIO_ACTIVE_LOW` بالغلط:

```c
/* board file — خطأ هنا */
static struct gpiod_lookup_table rs485_gpios = {
    .dev_id = "fe650000.serial",
    .table = {
        /* خطأ: DE/RE active-high circuit بس اتعرّف كـ active-low */
        GPIO_LOOKUP("gpio0", 23, "rs485-de", GPIO_ACTIVE_LOW),
        { },
    },
};
```

لما الـ driver يعمل:
```c
gpiod_set_value(de_gpio, 1); /* يقصد: "activate" */
```

الـ GPIO subsystem بيشوف إن الـ polarity هي `GPIO_ACTIVE_LOW`، فيترجم القيمة `1` (logical active) إلى `0` (physical low). النتيجة: الـ pin بيفضل LOW والـ transceiver في receive mode.

الـ `gpiod_lookup_flags` في `include/linux/gpio/machine.h`:
```c
enum gpio_lookup_flags {
    GPIO_ACTIVE_HIGH = (0 << 0), /* default */
    GPIO_ACTIVE_LOW  = (1 << 0), /* يقلب الـ polarity */
    ...
};
```

الـ `GPIO_ACTIVE_LOW` flag بيأثر على كل `gpiod_set_value()` و`gpiod_get_value()` — كل القيم بتتقلب transparently. ده بالظبط اللي الـ doc بيقوله:
> *"the active-low property is handled during mapping and is thus transparent to GPIO consumers"*

#### الحل

```c
/* board file — تصحيح */
static struct gpiod_lookup_table rs485_gpios = {
    .dev_id = "fe650000.serial",
    .table = {
        /* DE/RE pin: HIGH = transmit, LOW = receive → active-high */
        GPIO_LOOKUP("gpio0", 23, "rs485-de", GPIO_ACTIVE_HIGH),
        { },
    },
};
```

**أو** لو الـ hardware فعلاً active-low (نادر في RS485)، يبقى الـ driver هو اللي محتاج تعديل، مش الـ lookup.

```bash
# debug: تحقق من القيمة الفعلية على الـ pin
gpioget gpiochip0 23

# تحقق من الـ lookup flags المسجلة
cat /sys/kernel/debug/gpio
```

#### الدرس المستفاد
**الـ `GPIO_ACTIVE_LOW` flag مش مجرد documentation** — بيقلب كل القيم اللي بتتبعت عبر `gpiod_set_value()`. لازم تتطابق مع الـ hardware circuit الفعلي. دايماً اقرأ الـ schematic واتأكد إيه اللي بيعمل logic-high على الـ pin الفيزيائي.

---

### السيناريو 2: Android TV Box على Allwinner H616 — LED مش بتستجابش لأوامر الـ driver

#### العنوان
اسم الـ GPIO chip في الـ lookup table بيختلف عن الـ label الفعلي للـ gpiochip

#### السياق
منتج Android TV box بيستخدم Allwinner H616. الـ board عنده RGB status LED متوصل بـ 3 GPIOs من bank PL (اللي بيكون على RTC domain في H616). الـ driver للـ LEDs بيستخدم platform data مع `gpiod_lookup_table`.

#### المشكلة
الـ `leds-gpio` driver بيرجع `-ENOENT` من `gpiod_get_index()`. الـ LEDs مش بتشتغل خالص. في الـ dmesg:
```
leds-gpio foo.0: can't get LED gpio: -2
```

#### التحليل
الـ `gpiod_get_index()` بيعمل lookup في `gpiod_lookup_table` بيدور على entry بـ `key` يساوي label الـ gpiochip. المشكلة إن الـ `key` في الـ table غلط:

```c
/* board file — خطأ في الـ key */
static struct gpiod_lookup_table led_gpios = {
    .dev_id = "leds-gpio",
    .table = {
        GPIO_LOOKUP_IDX("pio",    15, "led", 0, GPIO_ACTIVE_HIGH), /* PA15 */
        GPIO_LOOKUP_IDX("pio",    16, "led", 1, GPIO_ACTIVE_HIGH), /* PA16 */
        GPIO_LOOKUP_IDX("r_pio",  10, "led", 2, GPIO_ACTIVE_HIGH), /* PL10 */
        { },
    },
};
```

في H616، الـ GPIO bank PL بيتحكم فيه controller تاني (`r_pio` أو حاجة تانية حسب الـ kernel version). لو اسم الـ chip في `/sys/kernel/debug/gpio` بيظهر كـ `300b000.pinctrl` مش `r_pio`، الـ lookup بيفشل.

من الـ doc:
> *"key is either the label of the gpiod_chip instance providing the GPIO, or the GPIO line name"*

```bash
# اعرف الأسماء الصح
cat /sys/kernel/debug/gpio | grep -E "gpiochip|label"
# أو
gpiodetect
```

الـ output ممكن يكون:
```
gpiochip0 - 288 lines  [300b000.pinctrl] (gpio-controller)
gpiochip1 - 32  lines  [7022000.pinctrl] (r_gpio-controller)
```

يعني الـ label الصح للـ PL bank هو `7022000.pinctrl`، مش `r_pio`.

#### الحل

```c
/* تصحيح بعد معرفة الأسماء الفعلية */
static struct gpiod_lookup_table led_gpios = {
    .dev_id = "leds-gpio",
    .table = {
        GPIO_LOOKUP_IDX("300b000.pinctrl", 15, "led", 0, GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP_IDX("300b000.pinctrl", 16, "led", 1, GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP_IDX("7022000.pinctrl", 10, "led", 2, GPIO_ACTIVE_HIGH),
        { },
    },
};
```

**أو بديل أفضل**: استخدم GPIO line name كـ key بدل label الـ chip:
```c
/* استخدم U16_MAX كـ chip_hwnum عشان تدل إن الـ key هو line name */
GPIO_LOOKUP_IDX("PL10", U16_MAX, "led", 2, GPIO_ACTIVE_HIGH),
```

#### الدرس المستفاد
الـ `key` في `GPIO_LOOKUP` لازم يتطابق بالظبط مع الـ `label` اللي الـ gpiochip اتسجل بيه. استخدم `gpiodetect` أو `/sys/kernel/debug/gpio` تعرف الأسماء الصح قبل ما تكتب الـ board file. البديل الأكثر stability هو استخدام GPIO line names مع `U16_MAX`.

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — GPIO Hog بيأخذ الـ SPI CS بالغلط

#### العنوان
`GPIO_HOG` على STM32MP1 بيـhog الـ SPI chip select قبل ما الـ SPI driver يتحمل

#### السياق
IoT sensor board بيستخدم STM32MP1 (Cortex-A7 + M4). الـ board عنده SPI flash (W25Q128) متوصل على SPI1 مع GPIO-controlled CS. الـ board file قديم وبيستخدم platform data. المهندس حاول يستخدم `gpiod_add_hogs()` لـ GPIO hog عشان يضمن إن الـ CS بيبقى HIGH (inactive) من أول ما الـ system يبدأ.

#### المشكلة
الـ SPI flash مش بيتعرفش. الـ `spi-nor` driver بيرجع `-EBUSY` لما بيحاول يطلب الـ CS GPIO. في الـ dmesg:
```
spi1.0: Failed to get CS GPIO: -16
```
(`-16` = `-EBUSY`)

#### التحليل
الـ `gpiod_add_hogs()` بيعمل hog على الـ GPIO line — يعني بيحجزها لنفسه ومحدش تاني يقدر يطلبها. لما الـ SPI driver بعدين بيحاول يعمل `gpiod_get()` على نفس الـ GPIO، بيلاقيها محجوزة.

```c
/* board file — مشكلة: hogging SPI CS */
static struct gpiod_hog spi_hogs[] = {
    /* خطأ: الـ CS محتاج يتحكم فيه الـ SPI driver مش الـ hog */
    GPIO_HOG("spi0-gpio", 4, "spi1-cs0", GPIO_ACTIVE_LOW, GPIOD_OUT_HIGH),
    { }
};

/* في init */
gpiod_add_hogs(spi_hogs);
```

من الـ doc:
> *"The line will be hogged as soon as the gpiochip is created"*

الـ GPIO hog مصمم للـ GPIOs اللي محتاجة تبقى في state معين ومحدش المفروض يتحكم فيها من code. أما الـ SPI CS، فالـ SPI subsystem نفسه محتاج يتحكم فيها أثناء الـ transfers.

#### الحل

```c
/* امسح الـ hog وخلي الـ SPI driver يتحكم في الـ CS */
/* استخدم lookup table عادية بدل hog */
static struct gpiod_lookup_table spi1_cs_gpios = {
    .dev_id = "spi1",
    .table = {
        /* الـ SPI framework بيطلب "cs" function */
        GPIO_LOOKUP("spi0-gpio", 4, "cs", GPIO_ACTIVE_LOW),
        { },
    },
};

/* في init */
gpiod_add_lookup_table(&spi1_cs_gpios);
/* مش gpiod_add_hogs() */
```

**متى تستخدم GPIO HOG؟**
- Power enable pin لـ regulator لازم يبقى HIGH من أول ما الـ system يبدأ
- Reset pin لـ external chip لازم يبقى deasserted
- مش للـ peripherals اللي بتتحكم فيها drivers

#### الدرس المستفاد
الـ `GPIO_HOG` مش بديل للـ `GPIO_LOOKUP`. الـ Hog معناه "أنا بملكها وحدي". لو GPIO محتاج driver تاني يتحكم فيه، استخدم lookup table مش hog.

---

### السيناريو 4: Custom Board Bring-up على i.MX8MM — Device Tree GPIO Mapping فاشل بسبب اسم function غلط

#### العنوان
`gpiod_get()` بيرجع `-ENOENT` على i.MX8MM لأن الـ property name في DT مش متطابق مع الـ con_id

#### السياق
bring-up لـ custom board مبني على i.MX8MM (NXP). الـ board عنده HDMI power enable GPIO متوصل على GPIO5_IO05. الـ DRM driver (لـ HDMI bridge) بيطلب الـ GPIO بـ `gpiod_get(dev, "hdmi-pwr", GPIOD_OUT_LOW)`.

#### المشكلة
الـ HDMI bridge driver بيفشل في probe:
```
imx8mm-hdmi-bridge 32c00000.hdmi: Failed to get hdmi-pwr GPIO: -2
```

المهندس أضاف الـ GPIO في الـ device tree لكن المشكلة لسه موجودة.

#### التحليل
الـ `gpiod_get(dev, "hdmi-pwr", flags)` داخلياً بيعمل:
```c
/* internally in gpiolib */
snprintf(propname, sizeof(propname), "%s-%s", con_id, gpio_suffixes[i]);
/* يعمل: "hdmi-pwr-gpios" أو "hdmi-pwr-gpio" */
```

من الـ doc:
> *"Internally, the GPIO subsystem prefixes the GPIO suffix ("gpios" or "gpio") with the string passed in con_id"*

المهندس كتب في الـ DT:
```dts
/* خطأ: اسم الـ property غلط */
&hdmi_bridge {
    hdmi_pwr-gpios = <&gpio5 5 GPIO_ACTIVE_HIGH>; /* underscore بدل dash */
};
```

الـ kernel بيدور على `hdmi-pwr-gpios` (بـ dash)، بس الـ DT عنده `hdmi_pwr-gpios` (بـ underscore). الـ OF layer في بعض الـ kernel versions بيـconvert الـ underscores لـ dashes، لكن مش دايماً — وده بيسبب confusion.

**الفرق:**
```dts
hdmi-pwr-gpios  = <&gpio5 5 GPIO_ACTIVE_HIGH>; /* صح */
hdmi_pwr-gpios  = <&gpio5 5 GPIO_ACTIVE_HIGH>; /* خطأ */
```

#### الحل

```dts
/* device tree — تصحيح */
&hdmi_bridge {
    compatible = "vendor,hdmi-bridge";
    /* اسم الـ property = con_id + "-gpios" */
    hdmi-pwr-gpios = <&gpio5 5 GPIO_ACTIVE_HIGH>;
};
```

```bash
# تحقق إن الـ DT property موجودة صح
dtc -I fs /sys/firmware/devicetree/base | grep -A2 hdmi-pwr

# أو من الـ debugfs
cat /sys/firmware/devicetree/base/hdmi_bridge/hdmi-pwr-gpios | xxd
```

#### الدرس المستفاد
الـ property name في الـ DT لازم يكون `<con_id>-gpios` بالظبط بـ dashes، وهيطابق الـ `con_id` اللي بتبعته لـ `gpiod_get()`. أي underscore أو حرف غلط بيخلي الـ lookup يفشل بـ `-ENOENT` صامت.

---

### السيناريو 5: Automotive ECU على AM62x — Fast Bitmap Array مش شغال لأن الـ Pin Ordering غلط

#### العنوان
GPIO array على AM62x مش بيستخدم fast bitmap path بسبب ترتيب الـ pins

#### السياق
automotive ECU بيستخدم TI AM62x SoC. الـ ECU بيتحكم في 8 output channels (relay drivers) عبر 8 GPIOs. الـ driver بيستخدم `gpiod_get_array()` ثم `gpiod_set_array_value()` عشان يتحكم في كل الـ channels بكفاءة في real-time. المتطلب: latency أقل من 100µs لتحديث كل الـ 8 outputs في نفس الوقت.

#### المشكلة
المهندس لاحظ إن الـ GPIO update بياخد وقت أكبر من المتوقع (حوالي 400µs). بـ oscilloscope، لاحظ إن الـ GPIOs مش بتتحدث في نفس اللحظة — في delay واضح بينهم.

#### التحليل
من الـ doc، الـ fast bitmap processing path بيشترط:

> *"pin hardware number of array member 0 must also be 0"*
> *"pin hardware numbers of consecutive array members which belong to the same chip as member 0 does must also match their array indexes"*

المهندس عرّف الـ GPIOs في الـ DT كده:
```dts
/* device tree للـ ECU relay controller */
&relay_ctrl {
    relay-gpios = <&gpio0 5  GPIO_ACTIVE_HIGH>,  /* index 0, hwnum 5  — مش صح */
                  <&gpio0 6  GPIO_ACTIVE_HIGH>,  /* index 1, hwnum 6 */
                  <&gpio0 7  GPIO_ACTIVE_HIGH>,  /* index 2, hwnum 7 */
                  <&gpio0 8  GPIO_ACTIVE_HIGH>,  /* index 3, hwnum 8 */
                  <&gpio0 9  GPIO_ACTIVE_HIGH>,  /* index 4, hwnum 9 */
                  <&gpio0 10 GPIO_ACTIVE_HIGH>,  /* index 5, hwnum 10 */
                  <&gpio0 11 GPIO_ACTIVE_HIGH>,  /* index 6, hwnum 11 */
                  <&gpio0 12 GPIO_ACTIVE_HIGH>;  /* index 7, hwnum 12 */
};
```

المشكلة: array member 0 بيستخدم `hwnum = 5`، بس الـ doc يقول لازم يكون `hwnum = 0`. كمان `hwnum` مش بيتطابق مع الـ index (index 0 → hwnum 5، index 1 → hwnum 6، إلخ بدل 0, 1, 2...).

نتيجة: الـ kernel مش بيستخدم `.set_multiple()` callback الـ efficient — بيعمل loop ويـset كل GPIO بشكل individual.

#### الحل

**الحل الأمثل**: اختار GPIOs من نفس الـ chip وابدأ من hwnum 0:

```dts
/* تصحيح: pins 0-7 من نفس الـ GPIO bank */
&relay_ctrl {
    relay-gpios = <&gpio1 0  GPIO_ACTIVE_HIGH>,  /* index 0, hwnum 0 */
                  <&gpio1 1  GPIO_ACTIVE_HIGH>,  /* index 1, hwnum 1 */
                  <&gpio1 2  GPIO_ACTIVE_HIGH>,  /* index 2, hwnum 2 */
                  <&gpio1 3  GPIO_ACTIVE_HIGH>,  /* index 3, hwnum 3 */
                  <&gpio1 4  GPIO_ACTIVE_HIGH>,  /* index 4, hwnum 4 */
                  <&gpio1 5  GPIO_ACTIVE_HIGH>,  /* index 5, hwnum 5 */
                  <&gpio1 6  GPIO_ACTIVE_HIGH>,  /* index 6, hwnum 6 */
                  <&gpio1 7  GPIO_ACTIVE_HIGH>;  /* index 7, hwnum 7 */
};
```

**لو مش ممكن تغيير الـ HW routing**، الـ driver ممكن يعمل manual bitmask:
```c
/* driver code: manual atomic set بدون fast path */
struct gpio_descs *relays = gpiod_get_array(dev, "relay", GPIOD_OUT_LOW);

/* set كل الـ outputs atomically بـ bitmask */
DECLARE_BITMAP(values, 8);
bitmap_set(values, 0, 8); /* set all 8 */
gpiod_set_array_value(8, relays->desc, relays->info, values);
```

```bash
# تحقق إن الـ GPIO chip دعم .set_multiple
cat /sys/kernel/debug/gpio | grep "gpio1"
```

#### الدرس المستفاد
**الـ fast bitmap path مش automatic** — بيتطلب إن array member 0 يكون hwnum 0 وكل الـ members يكونوا sequential. في الـ automotive و real-time applications، الـ pin assignment على الـ PCB لازم يتخطط من البداية مع ده في الاعتبار. لو الـ HW routing مش optimal، الـ latency بيتضاعف لأن الـ kernel بيـset كل pin لوحده.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

دي أهم المقالات اللي اتكتبت على LWN.net عن الـ GPIO descriptor interface والـ board mappings:

| المقال | الوصف |
|--------|-------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة للـ GPIO subsystem في الكيرنل — نقطة بداية ممتازة |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | الاتجاهات المستقبلية للـ GPIO API وإزاي هيتغير |
| [gpio: introduce descriptor-based interface](https://lwn.net/Articles/531848/) | أول patch قدّم الـ `gpiod_*` API الجديدة |
| [gpiolib: introduce descriptor-based GPIO interface](https://lwn.net/Articles/528226/) | التفاصيل التقنية لإعادة هيكلة الـ gpiolib على أساس descriptors |
| [New descriptor-based GPIO interface](https://lwn.net/Articles/565662/) | تغطية الـ merged API الجديدة في الكيرنل |
| [Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/) | التوثيق الأولي للـ interface الجديدة |
| [GPIO implementation framework](https://lwn.net/Articles/256461/) | مقال تاريخي عن أول إطار عمل للـ gpiolib |

---

### التوثيق الرسمي في الكيرنل

**الـ** `Documentation/` paths الأساسية اللي لازم تعرفها:

```
Documentation/driver-api/gpio/board.rst        ← الملف ده نفسه (GPIO Mappings)
Documentation/driver-api/gpio/consumer.rst     ← واجهة المستهلك (gpiod_get/set)
Documentation/driver-api/gpio/driver.rst       ← كتابة GPIO controller driver
Documentation/driver-api/gpio/intro.rst        ← مقدمة الـ GPIO subsystem
Documentation/driver-api/gpio/legacy.rst       ← الـ API القديمة (integer-based)
Documentation/driver-api/gpio/legacy-boards.rst← تحويل board files للـ software nodes
Documentation/firmware-guide/acpi/gpio-properties.rst ← GPIO في ACPI
Documentation/devicetree/bindings/gpio/        ← DT bindings للـ GPIO controllers
```

**الـ** online documentation الحالية على kernel.org:

- [GPIO Mappings (board.rst)](https://www.kernel.org/doc/html/latest/driver-api/gpio/board.html)
- [GPIO Descriptor Consumer Interface](https://www.kernel.org/doc/html/latest/driver-api/gpio/consumer.html)
- [General Purpose Input/Output (GPIO) — Index](https://www.kernel.org/doc/html/latest/driver-api/gpio/index.html)

---

### Commits مهمة في تاريخ الـ GPIO

الـ commits دي غيّرت شكل الـ subsystem:

| الحدث | المصدر |
|-------|--------|
| إدخال الـ `gpiod_*` descriptor API | [LWN patch thread](https://lwn.net/Articles/531848/) |
| إضافة `gpiod_add_lookup_table()` للـ platform data | [LKML patch](https://lore.kernel.org/lkml/1494527071-6412-1-git-send-email-agust@denx.de/) |
| نقل GPIO board code من integer API للـ lookup tables | [Patchwork](https://patchwork.kernel.org/project/linux-omap/patch/20180518210954.29044-2-jmkrzyszt@gmail.com/) |

للبحث عن commits معينة:

```bash
# تاريخ كل التغييرات على ملف board.rst
git log --oneline -- Documentation/driver-api/gpio/board.rst

# commits الخاصة بالـ gpio machine interface
git log --oneline -- include/linux/gpio/machine.h

# commits الخاصة بالـ gpiolib core
git log --oneline -- drivers/gpio/gpiolib.c | head -30
```

---

### نقاشات Mailing List

- [Using GPIO from device tree on platform devices (kernelnewbies)](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html) — نقاش عملي عن ربط الـ GPIO بالـ device tree
- [Finding GPIO names under Linux (kernelnewbies)](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-May/016304.html) — إزاي تلاقي أسماء GPIO lines في النظام
- [gpio-mcp23s08 driver with multiple chips (kernelnewbies)](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-June/016408.html) — نقاش عن board-level GPIO setup مع multiple controllers
- [Add gpio_add_lookup_tables() — LKML](https://lkml.iu.edu/hypermail/linux/kernel/1708.0/00809.html) — نقاش إضافة helper لتسجيل أكتر من lookup table دفعة واحدة
- [gpiolib: Add stubs for gpiod lookup table — LKML](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1386243.html) — نقاش الـ !GPIOLIB stubs

---

### صفحات kernelnewbies.org

- [Linux_2_6_21 — kernelnewbies](https://kernelnewbies.org/Linux_2_6_21) — أول إصدار كيرنل أدخل الـ GPIO API
- [Linux_6.2 — kernelnewbies](https://kernelnewbies.org/Linux_6.2) — تغييرات الـ GPIO في كيرنل 6.2
- [Linux_6.6 — kernelnewbies](https://kernelnewbies.org/Linux_6.6) — تغييرات الـ GPIO في كيرنل 6.6
- [Linux_6.13 — kernelnewbies](https://kernelnewbies.org/Linux_6.13) — أحدث تغييرات الـ GPIO

---

### صفحات elinux.org

- [Pin Control and GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) — شرح العلاقة بين الـ pinctrl subsystem والـ GPIO
- [EBC Device Trees](https://elinux.org/EBC_Device_Trees) — تمارين عملية على BeagleBone لفهم الـ GPIO في device tree
- [Leapster Explorer: GPIO subsystem](https://elinux.org/Leapster_Explorer:_GPIO_subsystem) — مثال real-world على board-level GPIO setup
- [DHT-Walnut GPIO](https://elinux.org/DHT-Walnut_GPIO) — GPIO register management على معالج PPC405

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 1**: مقدمة لكتابة الـ kernel modules
- **الفصل 6**: Advanced Char Driver Operations — relevant لفهم device APIs
- الكتاب متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Devices and Modules — يشرح device model والـ platform devices
- **الفصل 14**: The Block I/O Layer — مفيد لفهم الـ descriptor pattern

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 14**: Kernel Initialization — يشرح board setup وتسجيل الـ platform devices
- **الفصل 15**: Debugging Embedded Linux — يشمل debugging الـ GPIO

#### Linux Device Driver Development — John Madieu (2nd Edition, 2022)
- **الفصل 15**: GPIO Controller Drivers — أحدث كتاب يغطي الـ gpiod API بالكامل
- **الفصل 11**: Kernel Memory Management — relevant للـ software nodes

---

### مصادر إضافية

- [Linux device driver development: The GPIO interface and device tree — embedded.com](https://www.embedded.com/linux-device-driver-development-the-gpio-interface-and-device-tree/)
- [Linux device driver development: The descriptor-based GPIO interface — embedded.com](https://www.embedded.com/linux-device-driver-development-the-descriptor-based-gpio-interface/)
- [GPIO device tree configuration — ST Microelectronics Wiki](https://wiki.st.com/stm32mpu/wiki/GPIO_device_tree_configuration)

---

### Search Terms للبحث عن معلومات أكتر

لو عايز تتعمق أكتر، استخدم الـ search terms دي:

```
# للبحث في LWN.net
site:lwn.net "gpio descriptor" kernel
site:lwn.net "gpiod_get" OR "gpiod_lookup"
site:lwn.net "gpio machine" platform data

# للبحث في LKML
site:lore.kernel.org gpio lookup table board
site:patchwork.kernel.org gpio machine.h

# للبحث العام
linux kernel "GPIO_LOOKUP" platform data board file
linux kernel "software_node" gpio property_entry
linux kernel gpio "PROPERTY_ENTRY_GPIO" example
linux kernel gpiod_add_hogs gpio hogging
linux kernel gpio "active low" descriptor transparent
linux kernel gpio "fast bitmap" array pins
```

---

### ملخص المصادر حسب الموضوع

| الموضوع | المصدر الأفضل |
|---------|--------------|
| مقدمة الـ GPIO API | [LWN: GPIO in the kernel](https://lwn.net/Articles/532714/) |
| الـ descriptor interface الجديدة | [LWN: New descriptor-based GPIO](https://lwn.net/Articles/565662/) |
| الـ Device Tree bindings | `Documentation/devicetree/bindings/gpio/` |
| الـ ACPI GPIO properties | `Documentation/firmware-guide/acpi/gpio-properties.rst` |
| الـ platform data lookup tables | [kernel.org: GPIO Mappings](https://www.kernel.org/doc/html/latest/driver-api/gpio/board.html) |
| كتابة GPIO controller | `Documentation/driver-api/gpio/driver.rst` |
| الـ consumer API كاملة | `Documentation/driver-api/gpio/consumer.rst` |
| تحويل board files | `Documentation/driver-api/gpio/legacy-boards.rst` |
## Phase 8: Writing simple module

### الفكرة

**`gpiod_add_lookup_table()`** هي الدالة اللي بتضيف جدول GPIO lookups للنظام — وده بيحصل في وقت التشغيل من أي كود board أو module. هنعمل kprobe على الدالة دي علشان نراقب كل مرة حد بيضيف جدول GPIOs، ونطبع اسم الـ device اللي بيطلب ده.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * gpio_lookup_probe.c
 *
 * Kprobe on gpiod_add_lookup_table() — prints the dev_id of every
 * GPIO lookup table that gets registered at runtime.
 */

/* kernel module basics */
#include <linux/module.h>
/* kprobe API */
#include <linux/kprobes.h>
/* gpio/machine.h defines struct gpiod_lookup_table */
#include <linux/gpio/machine.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("GPIO Probe Demo");
MODULE_DESCRIPTION("Kprobe on gpiod_add_lookup_table to log GPIO mappings");

/* ------------------------------------------------------------------ */
/* kprobe handler — called right before gpiod_add_lookup_table runs    */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument is in RDI.
     * On ARM64 it's in X0.
     * regs_get_kernel_argument() abstracts this for us.
     */
    struct gpiod_lookup_table *table =
        (struct gpiod_lookup_table *)regs_get_kernel_argument(regs, 0);

    if (!table)
        return 0;

    /* dev_id can be NULL — handle it gracefully */
    pr_info("gpio_lookup_probe: new lookup table registered, dev_id=\"%s\"\n",
            table->dev_id ? table->dev_id : "(null/global)");

    /* print the first entry's key so we know which chip is involved */
    if (table->table[0].key)
        pr_info("gpio_lookup_probe:   first entry key=\"%s\" hwnum=%u con_id=\"%s\"\n",
                table->table[0].key,
                table->table[0].chip_hwnum,
                table->table[0].con_id ? table->table[0].con_id : "(any)");

    return 0; /* 0 = continue normal execution */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "gpiod_add_lookup_table", /* الدالة اللي هنحط فيها الـ probe */
    .pre_handler = handler_pre,              /* callback قبل تنفيذ الدالة */
};

/* ------------------------------------------------------------------ */
/* module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init gpio_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("gpio_lookup_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("gpio_lookup_probe: planted kprobe at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit gpio_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("gpio_lookup_probe: kprobe removed\n");
}

module_init(gpio_probe_init);
module_exit(gpio_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الماكروهات الأساسية لأي kernel module |
| `linux/kprobes.h` | تعريف `struct kprobe` وفنكشنز `register/unregister_kprobe` |
| `linux/gpio/machine.h` | تعريف `struct gpiod_lookup_table` و `struct gpiod_lookup` اللي هنستخدمهم في الـ callback |

---

#### الـ `handler_pre` callback

ده الـ pre-handler بتاعنا — بيتشغل **قبل** ما `gpiod_add_lookup_table()` تبدأ فعلاً. بنستخدم `regs_get_kernel_argument(regs, 0)` علشان نجيب البوينتر الأول (الـ `table`) بطريقة portable على x86-64 وARM64 من غير ما نكتب كود معمارية مختلف. لو الـ `dev_id` كانت NULL معناها إن الجدول ده عام ومش مربوط بـ device بعينه.

---

#### الـ `struct kprobe kp`

**`symbol_name`** بتقول للـ kernel فين تزرع الـ breakpoint — في حالتنا على `gpiod_add_lookup_table`. الـ kernel هيحل الاسم لـ address وقت `register_kprobe()`. الـ `pre_handler` هو الفنكشن اللي هيتشغل كل مرة الـ probe بتتفعل.

---

#### الـ `module_init`

بتسجل الـ kprobe. لو فشلت (مثلاً الدالة مش موجودة أو KPROBES مش enabled)، بترجع الـ error code مباشرةً. الـ `%px` بيطبع الـ address بعد ما الـ kernel يحله علشان نتأكد إن الـ probe اتزرعت صح.

---

#### الـ `module_exit`

بتشيل الـ kprobe قبل ما الـ module يتـ unload. ده **إلزامي** — لو مشلتش الـ probe والـ module اتشال من الذاكرة، أي call لـ `gpiod_add_lookup_table()` هيـ jump لكود اتمسح وهيعمل kernel panic.

---

### Makefile

```makefile
obj-m += gpio_lookup_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميله
sudo insmod gpio_lookup_probe.ko

# شوف الـ log
sudo dmesg | grep gpio_lookup_probe

# لو عايز تجرب إن الـ probe بتشتغل، حمّل أي GPIO driver بعدها
# مثلاً:
sudo modprobe gpio-mockup gpio_mockup_ranges="0,8"

# فك تحميل الـ probe module
sudo rmmod gpio_lookup_probe
```

> **ملحوظة:** محتاج الـ kernel يكون compiled بـ `CONFIG_KPROBES=y` و `CONFIG_KALLSYMS=y`. على أغلب الـ distros ده بيكون موجود by default.
