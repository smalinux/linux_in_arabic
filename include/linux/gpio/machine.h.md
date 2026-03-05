## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبع له الملف

الملف `include/linux/gpio/machine.h` جزء من **GPIO Subsystem** في Linux kernel، المعروف بـ **gpiolib**. المسؤولون عنه هما Linus Walleij و Bartosz Golaszewski، والـ mailing list هي `linux-gpio@vger.kernel.org`.

---

### الصورة الكبيرة — قصة المشكلة

#### أولًا: إيه هو الـ GPIO أصلًا؟

**GPIO** — General Purpose Input/Output — ده pin على chip أو microcontroller ممكن تحوله لـ input أو output بشكل برمجي. يعني زي مفتاح كهربائي رقمي: تعمله HIGH أو LOW، أو تقرا منه.

مثال حقيقي: Raspberry Pi فيه 40 pin، كل واحد منهم ممكن يشغل LED، يقرا ضغطة زرار، يتحكم في relay، إلخ.

#### ثانيًا: المشكلة اللي الملف ده بيحلها

تخيل إنك بتبني **embedded board** — زي board خاصة بالشركة مبنية على SoC (System on Chip). على الـ board دي:

- فيه **reset button** متوصل على GPIO رقم 17 في chip اسمها `gpiochip0`
- فيه **LED** متوصلة على GPIO رقم 5 في chip اسمها `gpiochip1`
- فيه **enable pin** لـ Wi-Fi module متوصل على GPIO رقم 3 في `gpiochip0`

الـ driver بتاع الـ Wi-Fi مش المفروض يعرف "أنا على أنهي board؟ وأنهي GPIO رقم كام؟" — ده مش شغلته. شغلته إنه يطلب: "أنا عايز الـ GPIO اللي اسمه `wlan-enable` بتاعي".

**فمين بيربط الاسم `wlan-enable` بـ GPIO رقم 3 في `gpiochip0`؟**

هنا بالظبط بييجي دور `machine.h` — ده الملف اللي بيعرّف **جدول الربط** (lookup table) اللي الـ platform code أو الـ board file بيحطوه، عشان لما الـ driver يسأل عن GPIO بالاسم، الـ kernel يلاقيه.

---

### القصة كاملة: رحلة الـ GPIO من الـ Board للـ Driver

```
[ Board File / Platform Code ]
        |
        | يعرّف gpiod_lookup_table
        | يربط ("gpiochip0", pin 3) بـ ("wlan-enable")
        |
        v
[ gpiod_add_lookup_table() ]  <-- من machine.h
        |
        | يضيف الجدول في قايمة الـ kernel
        |
        v
[ Wi-Fi Driver يعمل gpiod_get(dev, "enable", ...) ]
        |
        | الـ kernel يبحث في الـ lookup tables
        | يلاقي "wlan-enable" → gpiochip0, pin 3
        |
        v
[ Driver يشغل/يوقف الـ GPIO بدون ما يعرف رقمه الحقيقي ]
```

---

### محتوى الملف بالتفصيل

#### 1. `enum gpio_lookup_flags` — خصائص الـ GPIO

بيحدد طبيعة الـ GPIO الكهربائية والمنطقية:

| Flag | معناه |
|------|--------|
| `GPIO_ACTIVE_HIGH` | الـ signal يكون 1 = فعّال (الافتراضي) |
| `GPIO_ACTIVE_LOW` | الـ signal يكون 0 = فعّال (مقلوب) |
| `GPIO_OPEN_DRAIN` | open-drain — مشترك مع أجهزة تانية على نفس الخط |
| `GPIO_OPEN_SOURCE` | open-source — عكس open-drain |
| `GPIO_PERSISTENT` | الـ GPIO يفضل على حاله بعد suspend (افتراضي) |
| `GPIO_TRANSITORY` | الـ GPIO يتريسيت بعد suspend/resume |
| `GPIO_PULL_UP` | تفعيل الـ internal pull-up resistor |
| `GPIO_PULL_DOWN` | تفعيل الـ internal pull-down resistor |
| `GPIO_PULL_DISABLE` | تعطيل الـ pull resistors |

#### 2. `struct gpiod_lookup` — صف واحد في جدول الربط

```c
struct gpiod_lookup {
    const char *key;       /* اسم الـ chip أو اسم الـ GPIO line */
    u16 chip_hwnum;        /* رقم الـ pin داخل الـ chip (أو U16_MAX لو بنبحث بالاسم) */
    const char *con_id;    /* الاسم اللي الـ driver بيعرف الـ GPIO بيه */
    unsigned int idx;      /* index لو في أكتر من GPIO بنفس الاسم */
    unsigned long flags;   /* flags من enum gpio_lookup_flags */
};
```

#### 3. `struct gpiod_lookup_table` — الجدول كله لجهاز معين

```c
struct gpiod_lookup_table {
    struct list_head list;   /* ربط الجداول مع بعض في linked list */
    const char *dev_id;      /* اسم الجهاز اللي الجدول ده بتاعه */
    struct gpiod_lookup table[]; /* flexible array من الـ lookups */
};
```

#### 4. `struct gpiod_hog` — الـ GPIO Hog

**GPIO Hog** ده مفهوم مختلف تمامًا: بدل ما driver يطلب الـ GPIO، الـ kernel نفسه يـ"يمسك" الـ GPIO فور ما الـ chip تتسجل، ويحطه في state معينة (input أو output بقيمة محددة) — من غير ما أي driver يطلبه.

مثال: pin مسؤول عن تفعيل ذاكرة خارجية — لازم يكون HIGH من أول ما الـ system يبدأ، من غير ما أي driver يتحمل.

```c
struct gpiod_hog {
    struct list_head list;
    const char *chip_label;  /* اسم الـ chip */
    u16 chip_hwnum;          /* رقم الـ pin */
    const char *line_name;   /* اسم وصفي للـ line */
    unsigned long lflags;    /* lookup flags */
    int dflags;              /* direction + value flags */
};
```

#### 5. الـ Macros — تسهيل التعريف

```c
/* تعريف entry واحدة في جدول الربط */
#define GPIO_LOOKUP(_key, _chip_hwnum, _con_id, _flags)

/* نفسه بس بـ index صريح لو في GPIOs بنفس الاسم */
#define GPIO_LOOKUP_IDX(_key, _chip_hwnum, _con_id, _idx, _flags)

/* جدول كامل بـ lookup واحدة بس لجهاز واحد */
#define GPIO_LOOKUP_SINGLE(_name, _dev_id, _key, _chip_hwnum, _con_id, _flags)

/* تعريف hog واحد */
#define GPIO_HOG(_chip_label, _chip_hwnum, _line_name, _lflags, _dflags)
```

#### 6. الـ API Functions

```c
/* يضيف جدول للـ kernel's global lookup list */
void gpiod_add_lookup_table(struct gpiod_lookup_table *table);

/* يضيف مصفوفة من الجداول دفعة واحدة */
void gpiod_add_lookup_tables(struct gpiod_lookup_table **tables, size_t n);

/* يشيل جدول (مثلًا لما module يتunload) */
void gpiod_remove_lookup_table(struct gpiod_lookup_table *table);

/* يضيف hog entries */
void gpiod_add_hogs(struct gpiod_hog *hogs);
void gpiod_remove_hogs(struct gpiod_hog *hogs);
```

لو `CONFIG_GPIOLIB` مش مفعّل، كل الدوال بتتحول لـ `static inline` فاضية — الكود يكمل من غير error.

---

### مثال حقيقي من حياة الـ Embedded Developer

```c
/* في board file خاص بـ custom board */
#include <linux/gpio/machine.h>

/* ربط GPIOs بالـ Wi-Fi driver */
static struct gpiod_lookup_table wlan_gpio_table = {
    .dev_id = "wlan-device",        /* اسم الجهاز في الـ kernel */
    .table = {
        /* chip اسمها "gpiochip0"، pin رقم 3، الـ driver يطلبه بـ "enable" */
        GPIO_LOOKUP("gpiochip0", 3, "enable", GPIO_ACTIVE_HIGH),
        /* terminator — لازم موجود */
        {}
    },
};

static int __init board_init(void)
{
    gpiod_add_lookup_table(&wlan_gpio_table);
    return 0;
}
```

والـ Wi-Fi driver يعمل:
```c
/* من غير ما يعرف أي رقم GPIO أو أي chip */
struct gpio_desc *en = devm_gpiod_get(dev, "enable", GPIOD_OUT_HIGH);
```

---

### ليه مش بنحط الأرقام مباشرة في الـ Driver؟

لأن نفس الـ Wi-Fi chip ممكن تتحط على 10 boards مختلفة، وكل board GPIO رقمه مختلف. الـ driver يفضل generic، والـ board-specific information تتحط في مكان واحد: الـ lookup table.

ده مبدأ **separation of concerns** — كل كود يعرف بس اللي محتاجه.

---

### الفرق بين Lookup Table و Device Tree و ACPI

| الطريقة | الاستخدام |
|---------|----------|
| **Lookup Table** (`machine.h`) | Boards بدون DT/ACPI — platform data pure C |
| **Device Tree** | Embedded systems حديثة (ARM, RISC-V) |
| **ACPI** | x86 / enterprise hardware |

الـ `machine.h` هو الحل الـ "C-only" القديم والموثوق للـ boards اللي مش بتستخدم DT.

---

### الملفات المرتبطة اللي لازم تعرفها

#### Core — قلب الـ GPIO Subsystem

| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpiolib.c` | الـ core library — بيخزن الـ lookup tables وبينفذ البحث فيها |
| `drivers/gpio/gpiolib-legacy.c` | الـ legacy integer-based GPIO API |
| `drivers/gpio/gpiolib-devres.c` | الـ devm_ wrappers لإدارة الـ resources تلقائيًا |
| `drivers/gpio/gpiolib-of.c` | دعم الـ Device Tree |
| `drivers/gpio/gpiolib-acpi-core.c` | دعم الـ ACPI |
| `drivers/gpio/gpiolib-cdev.c` | الـ userspace API (character device) |

#### Headers الأساسية

| الملف | الدور |
|-------|-------|
| `include/linux/gpio/machine.h` | **الملف ده** — lookup tables و hogs |
| `include/linux/gpio/consumer.h` | الـ API اللي الـ drivers بتستخدمه للـ get/set |
| `include/linux/gpio/driver.h` | الـ API اللي الـ GPIO chip drivers بتستخدمه |
| `include/linux/gpio/property.h` | دعم software nodes |
| `include/linux/gpio.h` | الـ legacy API القديم |

#### Hardware Drivers (أمثلة)

| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpio-mmio.c` | Generic memory-mapped GPIO |
| `drivers/gpio/gpio-dwapb.c` | Synopsys DesignWare APB GPIO |
| `drivers/gpio/gpio-ich.c` | Intel ICH/PCH GPIO |
| `drivers/gpio/gpio-mockup.c` | Virtual GPIO للـ testing |

#### Documentation

| المسار | المحتوى |
|--------|---------|
| `Documentation/driver-api/gpio/` | توثيق الـ API كاملًا |
| `Documentation/admin-guide/gpio/` | توثيق للـ sysfs/userspace |
| `Documentation/devicetree/bindings/gpio/` | DT bindings |
## Phase 2: شرح الـ GPIO Machine Lookup Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في الـ embedded Linux، الـ driver بتاع أي device محتاج يتعامل مع GPIO pins — مثلاً driver بتاع SD card محتاج GPIO للـ card-detect، driver بتاع LCD محتاج GPIO للـ reset، إلخ.

السؤال: **إزاي الـ driver يعرف انهي GPIO بالظبط يستخدمه؟**

#### الطريقة القديمة (legacy — integer numberspace)

```c
/* driver بيطلب GPIO رقم 42 hardcoded */
gpio_request(42, "sd-card-detect");
gpio_direction_input(42);
```

**المشكلة الجوهرية:** رقم 42 ده hardware-specific تماماً — على board تانية نفس الـ SD driver ممكن يحتاج GPIO رقم 17 أو 103. يعني:

- الـ driver بيبقى مربوط بـ board معينة
- مستحيل تعيد استخدام نفس الـ driver على hardware مختلف من غير ما تعدّل الـ source
- الـ GPIO numbers مش stable — بتتغير لو أضفت GPIO controller جديد

#### المشكلة الحقيقية

الـ driver محتاج يعبّر عن احتياجه بشكل **functional** مش **physical**:

> "أنا محتاج الـ GPIO الخاص بـ card-detect"

مش:

> "أنا محتاج GPIO رقم 42"

---

### الحل — الـ GPIO Descriptor + Machine Lookup Framework

الـ kernel حل المشكلة بفصل المسؤوليات في طبقتين:

| الطبقة | المسؤولية | المَن يكتبها |
|--------|-----------|--------------|
| **Consumer (driver)** | بيطلب GPIO باسم وظيفي `"card-detect"` | driver developer |
| **Machine/Board code** | بيربط الاسم الوظيفي بالـ GPIO الفعلي | board/platform developer |

الربط ده بيتم عن طريق **lookup tables** مُسجَّلة في الـ kernel — وده بالضبط اللي `machine.h` بيعرّفه.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    BOARD / PLATFORM CODE                    │
  │   (arch/arm/mach-xxx/board.c  أو  drivers/platform/xxx.c)  │
  │                                                             │
  │   gpiod_lookup_table: dev="mmc0", con="card-detect" → GPIO5│
  │   gpiod_lookup_table: dev="lcd0", con="reset"       → GPIO12│
  │   gpiod_hog:          chip="gpiochip0", hwnum=3 → "wifi-en" │
  └────────────────┬────────────────────────────────────────────┘
                   │  gpiod_add_lookup_table()
                   │  gpiod_add_hogs()
                   ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                   GPIOLIB CORE                              │
  │   ┌─────────────────┐     ┌──────────────────────────┐     │
  │   │  lookup_table   │     │   hog list               │     │
  │   │  linked list    │     │   (auto-claimed lines)   │     │
  │   └────────┬────────┘     └──────────────────────────┘     │
  │            │  lookup by (dev_id + con_id)                   │
  │            ▼                                                 │
  │   ┌─────────────────┐                                       │
  │   │  gpio_desc *    │  ← central abstraction                │
  │   └────────┬────────┘                                       │
  └────────────┼────────────────────────────────────────────────┘
               │  gpiod_get(dev, "card-detect", GPIOD_IN)
               ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                  CONSUMER DRIVER                            │
  │   (drivers/mmc/host/sdhci.c)                                │
  │                                                             │
  │   struct gpio_desc *cd = devm_gpiod_get(dev,               │
  │                              "card-detect", GPIOD_IN);      │
  │   int val = gpiod_get_value(cd);                            │
  └─────────────────────────────────────────────────────────────┘
               │
               ▼
  ┌─────────────────────────────────────────────────────────────┐
  │              GPIO CONTROLLER DRIVER                         │
  │   (drivers/gpio/gpio-pl061.c, gpio-mxc.c, etc.)            │
  │                                                             │
  │   struct gpio_chip → .get(), .set(), .direction_input()     │
  └─────────────────────────────────────────────────────────────┘
               │
               ▼
          [ HARDWARE: GPIO Controller registers ]
```

---

### تمثيل واقعي — فندق بـ extension numbers

تخيل فندق كبير:

| عنصر الفندق | المقابل في الـ kernel |
|-------------|----------------------|
| **النزيل (guest)** | الـ consumer driver |
| **اسم الخدمة** "محتاج room service" | الـ `con_id` — الاسم الوظيفي للـ GPIO |
| **الـ receptionist** | الـ GPIOLIB core |
| **دفتر أرقام الغرف الداخلية** | الـ `gpiod_lookup_table` |
| **رقم الغرفة الداخلي** (extension 214) | الـ `chip_hwnum` — رقم الـ GPIO في الـ chip |
| **اسم الـ chip** (برج A) | الـ `key` في `gpiod_lookup` |
| **النزيل الدائم اللي بيحجز غرفة من أول ما الفندق يفتح** | الـ **GPIO hog** |

**الربط الكامل:**

- النزيل مش محتاج يعرف رقم الغرفة — هو بس يقول "أنا محتاج room service"، والـ receptionist بيبحث في الدفتر ويوصّله
- لو الفندق اتوسّع وتغيرت أرقام الغرف، النزيل مش محتاج يغيّر حاجة — بس الـ receptionist بيحدّث الدفتر
- النزيل الدائم ده مثل `GPIO_HOG` — الـ kernel بيحجزه أوتوماتيك من أول ما الـ GPIO controller يتسجّل، قبل ما أي driver يطلبه

---

### الـ Core Abstractions

#### 1. `struct gpiod_lookup`

```c
struct gpiod_lookup {
    const char *key;       /* اسم الـ chip أو اسم الـ GPIO line */
    u16 chip_hwnum;        /* رقم الـ pin داخل الـ chip (أو U16_MAX لو بتبحث باسم الـ line) */
    const char *con_id;    /* الاسم الوظيفي من وجهة نظر الـ device — "reset", "card-detect" */
    unsigned int idx;      /* index لو في أكتر من GPIO بنفس الـ con_id */
    unsigned long flags;   /* GPIO_ACTIVE_LOW, GPIO_OPEN_DRAIN, إلخ */
};
```

ده الـ **atomic unit** في الـ lookup system — بيربط اسم وظيفي بـ GPIO فيزيائي محدد.

#### 2. `struct gpiod_lookup_table`

```c
struct gpiod_lookup_table {
    struct list_head list;      /* عشان يتربط في الـ global linked list */
    const char *dev_id;         /* اسم الـ device اللي الجدول ده خاص بيه */
    struct gpiod_lookup table[]; /* flexible array — الـ entries بتنتهي بـ {} فاضية */
};
```

**الـ `dev_id`** ده المفتاح الرئيسي — لما driver بيعمل `gpiod_get(dev, "reset", ...)` الـ core بيبحث في كل الجداول المسجّلة عن الجدول اللي `dev_id` بتاعه بيطابق اسم الـ device.

```
  gpiod_lookup_tables (global linked list)
  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
  │ dev_id: "mmc0"   │───▶│ dev_id: "lcd0"   │───▶│ dev_id: "usb0"   │
  │ table[0]:        │    │ table[0]:        │    │ table[0]:        │
  │  con="card-det"  │    │  con="reset"     │    │  con="vbus-en"   │
  │  key="gpiochip0" │    │  key="gpiochip1" │    │  key="gpiochip0" │
  │  hwnum=5         │    │  hwnum=12        │    │  hwnum=22        │
  │ table[1]: {}     │    │ table[1]: {}     │    │ table[1]: {}     │
  └──────────────────┘    └──────────────────┘    └──────────────────┘
```

#### 3. `struct gpiod_hog`

```c
struct gpiod_hog {
    struct list_head list;
    const char *chip_label;   /* اسم الـ GPIO chip */
    u16 chip_hwnum;           /* رقم الـ pin في الـ chip */
    const char *line_name;    /* اسم الـ line لما يتعرض في /sys/kernel/debug/gpio */
    unsigned long lflags;     /* line flags: ACTIVE_LOW, OPEN_DRAIN, إلخ */
    int dflags;               /* direction flags: GPIOD_OUT_HIGH, GPIOD_IN, إلخ */
};
```

الـ **GPIO hog** ده مفهوم مختلف تماماً — مش بيربط GPIO بـ driver معين، لكن بيقول للـ kernel:

> "لما الـ GPIO controller ده يتسجّل، احجز الـ pin ده أوتوماتيك وحطه في state معين — حتى لو مفيش driver طلبه"

**use case حقيقي:** pin بيتحكم في power rail لازم يكون HIGH من أول ما الـ system يبوت، قبل أي driver يشتغل.

---

### الـ enum `gpio_lookup_flags` — بالتفصيل

```c
enum gpio_lookup_flags {
    GPIO_ACTIVE_HIGH   = (0 << 0),  /* default — logic 1 = voltage high */
    GPIO_ACTIVE_LOW    = (1 << 0),  /* inverted — logic 1 = voltage low */
    GPIO_OPEN_DRAIN    = (1 << 1),  /* open-drain output */
    GPIO_OPEN_SOURCE   = (1 << 2),  /* open-source (open-collector complement) */
    GPIO_PERSISTENT    = (0 << 3),  /* default — state يفضل لو الـ driver sleep */
    GPIO_TRANSITORY    = (1 << 3),  /* state ممكن يتعمل reset لو الـ driver sleep */
    GPIO_PULL_UP       = (1 << 4),  /* enable internal pull-up */
    GPIO_PULL_DOWN     = (1 << 5),  /* enable internal pull-down */
    GPIO_PULL_DISABLE  = (1 << 6),  /* disable any pull resistor */
};
```

**الـ `GPIO_ACTIVE_LOW`** مهم جداً — لما الـ board code بيقول إن الـ GPIO ده active-low، الـ GPIOLIB core بيعمل polarity inversion أوتوماتيك. يعني الـ driver لما بيعمل `gpiod_set_value(desc, 1)` الـ kernel بيكتب 0 على الـ hardware. الـ driver مش محتاج يعرف الـ polarity — ده abstraction حقيقي.

---

### الـ Macros — لماذا موجودة وإيه الفرق بينهم؟

#### `GPIO_LOOKUP_IDX` — الـ base macro

```c
#define GPIO_LOOKUP_IDX(_key, _chip_hwnum, _con_id, _idx, _flags) \
(struct gpiod_lookup) {                                            \
    .key        = _key,                                            \
    .chip_hwnum = _chip_hwnum,                                     \
    .con_id     = _con_id,                                         \
    .idx        = _idx,                                            \
    .flags      = _flags,                                          \
}
```

بيبني `struct gpiod_lookup` مباشرةً — compound literal في C99.

#### `GPIO_LOOKUP` — بيستخدم `_idx = 0`

```c
#define GPIO_LOOKUP(_key, _chip_hwnum, _con_id, _flags) \
    GPIO_LOOKUP_IDX(_key, _chip_hwnum, _con_id, 0, _flags)
```

للحالة الأكتر شيوعاً — GPIO واحد لكل function.

#### `GPIO_LOOKUP_SINGLE` — complete table في سطر واحد

```c
#define GPIO_LOOKUP_SINGLE(_name, _dev_id, _key, _chip_hwnum, _con_id, _flags) \
static struct gpiod_lookup_table _name = {                                      \
    .dev_id = _dev_id,                                                          \
    .table  = {                                                                 \
        GPIO_LOOKUP(_key, _chip_hwnum, _con_id, _flags),                        \
        {},  /* terminator */                                                   \
    },                                                                          \
}
```

---

### مثال عملي كامل — board code لـ SD card

```c
/* في board file: arch/arm/mach-imx/board-myboard.c */
#include <linux/gpio/machine.h>

/* تعريف الـ lookup table */
static struct gpiod_lookup_table mmc0_gpio_table = {
    .dev_id = "sdhci-esdhc-imx.0",  /* اسم الـ device instance */
    .table  = {
        /* card detect: chip "gpio4", pin 5, active-low */
        GPIO_LOOKUP("gpio4", 5, "cd", GPIO_ACTIVE_LOW),

        /* write protect: chip "gpio4", pin 6, active-high */
        GPIO_LOOKUP("gpio4", 6, "wp", GPIO_ACTIVE_HIGH),

        /* terminator — لازم يكون آخر entry */
        {},
    },
};

/* hog: wifi enable pin — يكون HIGH من أول ما الـ system يبوت */
static struct gpiod_hog board_hogs[] = {
    GPIO_HOG("gpio3", 10, "wifi-en", GPIO_ACTIVE_HIGH, GPIOD_OUT_HIGH),
    {},
};

static void __init myboard_init(void)
{
    /* تسجيل الجدول في الـ GPIOLIB */
    gpiod_add_lookup_table(&mmc0_gpio_table);

    /* تسجيل الـ hogs */
    gpiod_add_hogs(board_hogs);
}
```

```c
/* في الـ driver: drivers/mmc/host/sdhci-esdhc-imx.c */
#include <linux/gpio/consumer.h>

static int sdhci_esdhc_imx_probe(struct platform_device *pdev)
{
    struct gpio_desc *cd_gpio, *wp_gpio;

    /*
     * بيبحث في الـ lookup tables عن entry بـ:
     *   dev_id = "sdhci-esdhc-imx.0"
     *   con_id = "cd"
     * الـ GPIOLIB بيرجع gpio_desc* جاهز للاستخدام
     */
    cd_gpio = devm_gpiod_get(&pdev->dev, "cd", GPIOD_IN);
    if (IS_ERR(cd_gpio))
        return PTR_ERR(cd_gpio);

    wp_gpio = devm_gpiod_get_optional(&pdev->dev, "wp", GPIOD_IN);

    /* استخدام — الـ active-low inversion بيتعمل أوتوماتيك */
    int card_present = gpiod_get_value_cansleep(cd_gpio);
    /* card_present = 1 لو الكارت موجود، حتى لو الـ pin voltage = 0 */
}
```

---

### إيه اللي الـ Framework بيملكه vs إيه اللي بيديه للـ drivers

```
  ┌──────────────────────────────────────────────────────────┐
  │          GPIOLIB / Machine Lookup Framework              │
  │                    يملك وينفّذ:                          │
  │                                                          │
  │  ✓ الـ global linked list للـ lookup tables              │
  │  ✓ الـ global linked list للـ hogs                       │
  │  ✓ خوارزمية البحث (dev_id + con_id matching)            │
  │  ✓ polarity inversion (ACTIVE_LOW)                       │
  │  ✓ auto-hog عند تسجيل الـ GPIO chip                     │
  │  ✓ الـ gpio_desc allocation والـ ownership tracking      │
  └───────────────────┬──────────────────────────────────────┘
                      │ يفوّض للـ GPIO chip driver
                      ▼
  ┌──────────────────────────────────────────────────────────┐
  │              GPIO Chip Driver (provider)                 │
  │                    ينفّذ:                                 │
  │                                                          │
  │  • .get()              — قراءة قيمة الـ pin              │
  │  • .set()              — كتابة قيمة على الـ pin          │
  │  • .direction_input()  — ضبط الـ pin كـ input            │
  │  • .direction_output() — ضبط الـ pin كـ output           │
  │  • .set_config()       — pull-up/down, open-drain, إلخ  │
  └──────────────────────────────────────────────────────────┘
```

**ملاحظة مهمة على الـ `key` في `gpiod_lookup`:** ممكن يكون:
- اسم الـ `gpio_chip` (مثل `"gpio4"`) → الـ core بيبحث عن الـ chip بالاسم ده وبياخد منه `chip_hwnum`
- أو اسم الـ GPIO line نفسها لو `chip_hwnum == U16_MAX` → الـ core بيبحث في كل الـ chips عن line بالاسم ده (مع تحذير: line names مش guaranteed unique)

---

### علاقة الـ subsystem بالـ subsystems التانية

| الـ Subsystem | الصلة |
|---------------|-------|
| **Device Model (`struct device`)** | الـ `gpiod_get()` بياخد `struct device *` — الـ lookup بيتم بـ `dev_name(dev)` اللي بيطابق `dev_id` في الجدول |
| **OF / Device Tree** | لو الـ board بتستخدم DT، الـ GPIO assignment بيتم عبر DT properties مباشرةً (`gpios = <&gpio1 5 GPIO_ACTIVE_LOW>`) — الـ machine lookup بيتجاوز — بس `machine.h` بيبقى fallback للـ boards من غير DT |
| **devres (devm_*)** | الـ `devm_gpiod_get()` بيربط الـ gpio_desc بـ lifetime الـ device — لما الـ device يتشال الـ GPIO بيتحرر أوتوماتيك |
| **ACPI** | زي الـ DT، الـ ACPI بيعمل GPIO assignment مباشر — machine lookup هو الـ fallback للـ pure platform code |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums — Cheatsheet

#### `enum gpio_lookup_flags` — بتوصف خصايص الـ GPIO line

| القيمة | البت | المعنى |
|---|---|---|
| `GPIO_ACTIVE_HIGH` | bit0 = 0 | الـ line شغال بالـ logic-high (default) |
| `GPIO_ACTIVE_LOW` | bit0 = 1 | الـ logic معكوس، يعني 0V = active |
| `GPIO_OPEN_DRAIN` | bit1 = 1 | الـ output بيشد للـ GND بس، محتاج pull-up خارجي |
| `GPIO_OPEN_SOURCE` | bit2 = 1 | الـ output بيشد للـ VCC بس، محتاج pull-down خارجي |
| `GPIO_PERSISTENT` | bit3 = 0 | الـ line بيفضل على حاله لما الـ system يصحى من sleep |
| `GPIO_TRANSITORY` | bit3 = 1 | الـ line ممكن يتغير أثناء الـ sleep/suspend |
| `GPIO_PULL_UP` | bit4 = 1 | فعّل الـ internal pull-up resistor |
| `GPIO_PULL_DOWN` | bit5 = 1 | فعّل الـ internal pull-down resistor |
| `GPIO_PULL_DISABLE` | bit6 = 1 | عطّل أي pull resistor داخلي |
| `GPIO_LOOKUP_FLAGS_DEFAULT` | 0 | `ACTIVE_HIGH` + `PERSISTENT` — القيمة الافتراضية |

> **ملاحظة مهمة:** الـ flags دي بتتحط في `gpiod_lookup.flags` و`gpiod_hog.lflags`، وبيقراها الـ GPIO core لما بيعمل `gpiod_get()`.

#### `enum gpiod_flags` — بتحدد اتجاه الـ GPIO لما بتجيبه

| القيمة | المعنى |
|---|---|
| `GPIOD_ASIS` | متغيرش أي إعداد موجود |
| `GPIOD_IN` | اتجاهه input |
| `GPIOD_OUT_LOW` | output وابدأ بـ LOW |
| `GPIOD_OUT_HIGH` | output وابدأ بـ HIGH |
| `GPIOD_OUT_LOW_OPEN_DRAIN` | open-drain output وابدأ بـ LOW |
| `GPIOD_OUT_HIGH_OPEN_DRAIN` | open-drain output وابدأ بـ HIGH |

---

### الـ Config Options المهمة

| الـ Option | الأثر |
|---|---|
| `CONFIG_GPIOLIB` | لو مش مفعّل، كل الـ API بتبقى `static inline` فاضية — الكود بيكمبايل بدون error لكن ما بيعملش حاجة |

---

### الـ Structs المهمة

#### `struct gpiod_lookup`

**الغرض:** سطر واحد في جدول الـ lookup — بيربط GPIO فيزيائي موجود على chip معين بـ consumer منطقي (device + اسم وظيفة).

```c
struct gpiod_lookup {
    const char *key;       /* اسم الـ chip (gpiochip0) أو اسم الـ line */
    u16 chip_hwnum;        /* رقم الـ GPIO على الـ chip (0-based)،
                              أو U16_MAX لو key هو اسم الـ line */
    const char *con_id;    /* اسم الوظيفة من ناحية الـ device، مثلاً "reset" */
    unsigned int idx;      /* الـ index لو في أكتر من GPIO بنفس con_id */
    unsigned long flags;   /* bitmask من gpio_lookup_flags */
};
```

**الاتصالات:**
- موجود جوه `gpiod_lookup_table.table[]` كـ flexible array
- الـ GPIO core بيمشي عليه لما بيعمل `gpiod_get(dev, con_id, flags)`

---

#### `struct gpiod_lookup_table`

**الغرض:** جدول كامل بكل الـ GPIO mappings لـ device معين. بيتسجّل في الـ kernel وبيتدور عليه وقت الـ lookup.

```c
struct gpiod_lookup_table {
    struct list_head list;      /* للربط في الـ global linked list */
    const char *dev_id;         /* اسم الـ device، مثلاً "spi0.0" */
    struct gpiod_lookup table[]; /* flexible array — بيتنهي بـ entry فاضية */
};
```

**الاتصالات:**
- `list` بيربطه بـ global list جوه الـ GPIO core (`gpio_lookup_list`)
- بيتعبّى بـ macros زي `GPIO_LOOKUP_IDX` و`GPIO_LOOKUP`
- الـ entry الأخيرة في `table[]` بتكون `{}` كـ sentinel لإنهاء الـ array

---

#### `struct gpiod_hog`

**الغرض:** بيعرّف GPIO line اتمسك (hogged) من الـ kernel نفسه ومش متاح للـ drivers — مثلاً LED دايماً شغال، أو reset line محتاج يفضل في state معين.

```c
struct gpiod_hog {
    struct list_head list;      /* ربط في الـ global hog list */
    const char *chip_label;     /* اسم الـ GPIO chip */
    u16 chip_hwnum;             /* رقم الـ GPIO على الـ chip */
    const char *line_name;      /* اسم consumer للـ hog */
    unsigned long lflags;       /* gpio_lookup_flags — polarity/pull */
    int dflags;                 /* gpiod_flags — direction/value */
};
```

**الفرق الجوهري بين `lookup` و`hog`:**

| الجانب | `gpiod_lookup` | `gpiod_hog` |
|---|---|---|
| مين بيستخدمه | driver عبر `gpiod_get()` | الـ kernel core مباشرة |
| وقت التفعيل | لما الـ driver يطلبه | لحظة register الـ chip |
| قابل للتحرير | آه، بـ `gpiod_put()` | لأ، محجوز دايماً |
| `con_id` | موجود | مفيش — مجرد `line_name` |

---

### رسم العلاقات بين الـ Structs

```
Platform Init Code
       │
       ▼
gpiod_lookup_table
┌─────────────────────────────────┐
│ list_head ──────────────────────┼──► gpio_lookup_list (global in core)
│ dev_id = "spi0.0"               │
│ table[] ──────────────────────┐ │
└───────────────────────────────┼─┘
                                │
              ┌─────────────────▼──────────────────┐
              │  gpiod_lookup [0]                   │
              │  .key        = "gpiochip0"          │
              │  .chip_hwnum = 17                   │
              │  .con_id     = "cs"                 │
              │  .idx        = 0                    │
              │  .flags      = GPIO_ACTIVE_LOW      │
              ├────────────────────────────────────-┤
              │  gpiod_lookup [1]                   │
              │  .key        = "gpiochip0"          │
              │  .chip_hwnum = 18                   │
              │  .con_id     = "reset"              │
              │  .idx        = 0                    │
              │  .flags      = GPIO_ACTIVE_HIGH     │
              ├─────────────────────────────────────┤
              │  {}  ← sentinel (نهاية الـ array)   │
              └─────────────────────────────────────┘


gpiod_hog
┌─────────────────────────────────┐
│ list_head ──────────────────────┼──► gpio_hog_list (global in core)
│ chip_label = "gpiochip1"        │
│ chip_hwnum = 5                  │
│ line_name  = "power-enable"     │
│ lflags     = GPIO_ACTIVE_HIGH   │
│ dflags     = GPIOD_OUT_HIGH     │
└─────────────────────────────────┘
```

---

### الـ Macros وعلاقتها بالـ Structs

```
GPIO_LOOKUP_IDX(key, hwnum, con_id, idx, flags)
    │
    └──► يبني struct gpiod_lookup مباشرة بالـ compound literal

GPIO_LOOKUP(key, hwnum, con_id, flags)
    │
    └──► يستدعي GPIO_LOOKUP_IDX مع idx=0

GPIO_LOOKUP_SINGLE(name, dev_id, key, hwnum, con_id, flags)
    │
    └──► يبني struct gpiod_lookup_table كامل بـ entry واحدة + sentinel
         مناسب للـ board files البسيطة

GPIO_HOG(chip_label, hwnum, line_name, lflags, dflags)
    │
    └──► يبني struct gpiod_hog بالـ compound literal
         بيتحط في array تنتهي بـ {} وتتبعت لـ gpiod_add_hogs()
```

---

### دورة حياة الـ Lookup Table

```
1. DEFINITION (board/platform file)
   ─────────────────────────────────
   static struct gpiod_lookup_table my_table = {
       .dev_id = "leds-gpio",
       .table  = {
           GPIO_LOOKUP("gpiochip0", 10, "led0", GPIO_ACTIVE_HIGH),
           {}
       },
   };

2. REGISTRATION (عادةً في arch_initcall أو board_init)
   ─────────────────────────────────────────────────────
   gpiod_add_lookup_table(&my_table);
       │
       └──► mutex_lock(&gpio_lookup_lock)
            list_add_tail(&table->list, &gpio_lookup_list)
            mutex_unlock(&gpio_lookup_lock)

3. USAGE (من جوه الـ driver)
   ────────────────────────────
   desc = gpiod_get(dev, "led0", GPIOD_OUT_LOW);
       │
       └──► GPIO core بيدور في gpio_lookup_list
            بيتطابق dev->name مع dev_id
            بيتطابق "led0" مع con_id
            بيجيب الـ gpio_desc من الـ chip

4. TEARDOWN (لو الـ platform تتشال ديناميكياً)
   ──────────────────────────────────────────────
   gpiod_remove_lookup_table(&my_table);
       │
       └──► mutex_lock(&gpio_lookup_lock)
            list_del(&table->list)
            mutex_unlock(&gpio_lookup_lock)
```

---

### دورة حياة الـ Hog

```
1. DEFINITION
   ────────────
   static struct gpiod_hog my_hogs[] = {
       GPIO_HOG("gpiochip0", 5, "wifi-enable",
                GPIO_ACTIVE_HIGH, GPIOD_OUT_HIGH),
       {}
   };

2. REGISTRATION
   ─────────────
   gpiod_add_hogs(my_hogs);
       │
       └──► لكل entry حتى sentinel:
            list_add_tail(&hog->list, &gpio_hog_list)
       │
       └──► لو الـ chip اتسجّل قبل كده، يتطبق الـ hog فوراً
            gpiochip_hog_gpio(chip, hog)
               └──► gpiod_request() + gpiod_configure_flags()

3. CHIP REGISTRATION (لو جه بعدين)
   ──────────────────────────────────
   gpiochip_add_data()
       └──► gpiochip_apply_hogs(chip)
            └──► يدور في gpio_hog_list
                 لو chip_label == chip->label:
                     يحجز الـ line ويحط direction/value

4. REMOVAL
   ─────────
   gpiod_remove_hogs(my_hogs);
       └──► list_del() لكل entry
            الـ lines المحجوزة بتفضل، بس الـ hog entry اتشالت
```

---

### Call Flow — من الـ Driver لحد الـ Hardware

```
driver يطلب GPIO:
─────────────────
  gpiod_get(dev, "reset", GPIOD_OUT_LOW)
    │
    ├──► gpiod_find(dev, "reset", 0)
    │       │
    │       └──► يدور في gpio_lookup_list
    │            يتطابق dev_id + con_id + idx
    │            يرجع (chip_name, hwnum, flags)
    │
    ├──► gpio_find_chip_by_name(chip_name)
    │       └──► يرجع struct gpio_chip *
    │
    ├──► gpiochip_get_desc(chip, hwnum)
    │       └──► يرجع struct gpio_desc *
    │
    ├──► gpiod_request(desc, "reset")
    │       └──► chip->request(chip, hwnum)  [اختياري]
    │
    └──► gpiod_configure_flags(desc, GPIOD_OUT_LOW, lookup_flags)
            ├──► gpiod_set_consumer_name()
            ├──► gpio_set_config() → chip->set_config()
            └──► gpiod_direction_output(desc, 0)
                    └──► chip->direction_output(chip, hwnum, 0)
                              └──► يكتب في الـ hardware register
```

---

### Call Flow — تسجيل الـ Hog

```
gpiod_add_hogs(hogs[])
    │
    ├── يضيف كل hog في gpio_hog_list
    │
    └── لكل chip مسجّل:
        gpiochip_apply_hogs(chip)
            │
            └── لكل hog في القايمة:
                لو chip->label == hog->chip_label:
                    desc = gpiochip_get_desc(chip, hog->chip_hwnum)
                    gpiod_request(desc, hog->line_name)
                    gpiod_configure_flags(desc, hog->dflags, hog->lflags)
                    ──► الـ line اتحجز ومش هيلاقيه أي driver
```

---

### استراتيجية الـ Locking

الـ file نفسه ما بيعرّفش locks — اللوكات موجودة في الـ GPIO core (`drivers/gpio/gpiolib.c`). لكن الـ API المعرفة هنا بتتعامل مع:

| الـ Lock | بيحمي إيه | النوع |
|---|---|---|
| `gpio_lookup_lock` | الـ `gpio_lookup_list` global list | `mutex` |
| `gpio_hog_lock` | الـ `gpio_hog_list` global list | `mutex` |
| `gpio_desc->srcu` أو `chip->bgpio_lock` | وصول الـ descriptor نفسه | حسب الـ chip |

**ترتيب الـ Lock (لتجنب deadlock):**

```
gpio_lookup_lock
    └──► gpio_chip->lock (داخلي في الـ chip)
             └──► hardware register access
```

> **مهم:** الـ `gpiod_add_lookup_table()` و`gpiod_remove_lookup_table()` آمنين للاستخدام من أي context يقدر يمسك mutex (process context). ما ينفعوش من interrupt context.

---

### مثال واقعي — Raspberry Pi style board

```c
/* board file: arch/arm/mach-bcm2835/board.c */

#include <linux/gpio/machine.h>

/* جدول الـ GPIO لـ SPI flash */
static struct gpiod_lookup_table spi_flash_gpios = {
    .dev_id = "spi0.0",                         /* اسم الـ SPI device */
    .table = {
        GPIO_LOOKUP("pinctrl-bcm2835", 8,
                    "cs", GPIO_ACTIVE_LOW),      /* chip select — active low */
        GPIO_LOOKUP("pinctrl-bcm2835", 25,
                    "reset", GPIO_ACTIVE_LOW),   /* reset — active low */
        {}                                       /* sentinel إجباري */
    },
};

/* hog لـ power LED دايماً شغال */
static struct gpiod_hog board_hogs[] = {
    GPIO_HOG("pinctrl-bcm2835", 16,
             "pwr-led", GPIO_ACTIVE_LOW, GPIOD_OUT_LOW), /* LED يضوي */
    {}
};

static int __init board_init(void)
{
    gpiod_add_lookup_table(&spi_flash_gpios);
    gpiod_add_hogs(board_hogs);
    return 0;
}
arch_initcall(board_init);
```

```c
/* driver: drivers/mtd/spi/spi-flash.c */
static int spi_flash_probe(struct spi_device *spi)
{
    struct gpio_desc *cs, *rst;

    /* الـ core بيدور في gpio_lookup_list ويلاقي "spi0.0" → "cs" */
    cs = devm_gpiod_get(&spi->dev, "cs", GPIOD_OUT_HIGH);

    rst = devm_gpiod_get(&spi->dev, "reset", GPIOD_OUT_HIGH);

    /* استخدام عادي */
    gpiod_set_value(rst, 1); /* assert reset */
    /* ... */
    gpiod_set_value(rst, 0); /* release reset */
}
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Registration & Lifecycle APIs

| Function / Macro | النوع | الغرض |
|---|---|---|
| `gpiod_add_lookup_table()` | function | تسجيل lookup table واحدة في الـ global list |
| `gpiod_add_lookup_tables()` | function | تسجيل مصفوفة من الـ lookup tables دفعةً واحدة |
| `gpiod_remove_lookup_table()` | function | إزالة lookup table مسجَّلة من الـ global list |
| `gpiod_add_hogs()` | function | تسجيل مجموعة GPIO hogs وتطبيقها فوراً لو الـ chip موجود |
| `gpiod_remove_hogs()` | function | إزالة GPIO hogs من الـ global hog list |

#### Compile-time Helper Macros

| Macro | الغرض |
|---|---|
| `GPIO_LOOKUP_IDX()` | بناء `struct gpiod_lookup` بـ index صريح |
| `GPIO_LOOKUP()` | اختصار لـ `GPIO_LOOKUP_IDX` بـ idx=0 |
| `GPIO_LOOKUP_SINGLE()` | تعريف `gpiod_lookup_table` كاملة لـ device واحد بـ entry واحدة |
| `GPIO_HOG()` | بناء `struct gpiod_hog` entry |

#### Flags Enum

| Flag | القيمة | المعنى |
|---|---|---|
| `GPIO_ACTIVE_HIGH` | 0 | الـ line active عند high |
| `GPIO_ACTIVE_LOW` | bit 0 | الـ line active عند low (polarity inversion) |
| `GPIO_OPEN_DRAIN` | bit 1 | open-drain output |
| `GPIO_OPEN_SOURCE` | bit 2 | open-source output |
| `GPIO_PERSISTENT` | 0 | الـ line تحتفظ بقيمتها عند sleep/reset |
| `GPIO_TRANSITORY` | bit 3 | الـ line ممكن تخسر قيمتها عند sleep/reset |
| `GPIO_PULL_UP` | bit 4 | تفعيل pull-up |
| `GPIO_PULL_DOWN` | bit 5 | تفعيل pull-down |
| `GPIO_PULL_DISABLE` | bit 6 | إلغاء الـ bias تماماً |

---

### تصنيف الـ Functions

```
┌─────────────────────────────────────────────────┐
│             gpio/machine.h API                  │
├─────────────────┬───────────────────────────────┤
│  Lookup Tables  │  Hog Management               │
│  (platform data │  (exclusive ownership of      │
│   GPIO mapping) │   GPIO lines at boot)         │
├─────────────────┴───────────────────────────────┤
│           Build-time Macros                     │
│  (GPIO_LOOKUP, GPIO_HOG, GPIO_LOOKUP_SINGLE)    │
└─────────────────────────────────────────────────┘
```

الـ header ده بيمثل **platform data layer** في الـ GPIO descriptor framework. وظيفته الأساسية إنه يربط الـ GPIO lines الـ physical بالـ devices والـ functions بتاعتهم من غير ما تستخدم Device Tree أو ACPI — خصوصاً في الـ embedded platforms اللي بتستخدم board files.

---

### Group 1: Lookup Table Registration

الـ lookup tables بتخلي الـ platform code يقول: "الـ device اسمه `spi0.0` محتاج GPIO من الـ chip اسمه `gpiochip0`، الـ line رقم 5، وهو اسمه `cs`". لما driver يطلب `gpiod_get(dev, "cs", ...)` الـ framework يبحث في الـ registered tables ويلاقي الـ mapping الصح.

---

#### `gpiod_add_lookup_table()`

```c
void gpiod_add_lookup_table(struct gpiod_lookup_table *table);
```

**الـ function بتعمل إيه:**
بتضيف `gpiod_lookup_table` واحدة لـ global linked list اسمها `gpio_lookup_list`. هي wrapper بسيطة على `gpiod_add_lookup_tables()` بتبعتلها pointer للـ table في array من عنصر واحد.

**Parameters:**
- `table` — pointer لـ `gpiod_lookup_table` اللي هتتسجل. لازم تبقى statically allocated أو تعيش طول مدة التسجيل.

**Return value:** `void` — مفيش error return.

**Key details:**
- بتاخد `gpio_lookup_lock` mutex قبل ما تعدل الـ list.
- الـ table بتُضاف في **tail** الـ list عشان الـ order يبقى deterministic.
- مفيش validation على الـ table content — الـ lookup بيحصل lazy وقت الـ `gpiod_get()`.
- exported بـ `EXPORT_SYMBOL_GPL`.

**Who calls it:** Platform init code في board files أو platform drivers وقت الـ `__init` أو `probe`.

---

#### `gpiod_add_lookup_tables()`

```c
void gpiod_add_lookup_tables(struct gpiod_lookup_table **tables, size_t n);
```

**الـ function بتعمل إيه:**
بتسجل مصفوفة كاملة من الـ lookup tables دفعةً واحدة تحت mutex واحد، أكفأ من استدعاء `gpiod_add_lookup_table()` على كل table لوحدها.

**Parameters:**
- `tables` — array من الـ pointers لـ `gpiod_lookup_table`.
- `n` — عدد الـ tables في الـ array.

**Return value:** `void`.

**Key details:**
- بتاخد `gpio_lookup_lock` mutex مرة واحدة بس وبتعمل كل الـ `list_add_tail()` calls جوا الـ critical section — ده بيقلل الـ lock contention.
- هي الـ function الأساسية، و`gpiod_add_lookup_table()` بتفوض إليها.

**Pseudocode:**
```c
void gpiod_add_lookup_tables(tables, n):
    lock(gpio_lookup_lock)           // guard(mutex)
    for i = 0 to n-1:
        list_add_tail(tables[i]->list, gpio_lookup_list)
    unlock(gpio_lookup_lock)         // auto via guard
```

**Who calls it:** Platform init code، أو `gpiod_add_lookup_table()` نفسها.

---

#### `gpiod_remove_lookup_table()`

```c
void gpiod_remove_lookup_table(struct gpiod_lookup_table *table);
```

**الـ function بتعمل إيه:**
بتشيل الـ `gpiod_lookup_table` من الـ global list. لازم تتاخد قبل ما الـ platform code يحرر الـ memory بتاعة الـ table.

**Parameters:**
- `table` — pointer لـ الـ table اللي هتتشال. لو `NULL` بتـ return فوراً من غير عمل حاجة.

**Return value:** `void`.

**Key details:**
- بتـ guard بـ `gpio_lookup_lock` mutex.
- بتـ `list_del()` فقط — مفيش memory free لأن الـ table ممكن تكون statically allocated.
- **مهم:** لو حد شال الـ table وهو فيه device لسه ما استدعاش `gpiod_get()` بعد، الـ subsequent `gpiod_get()` هيفشل أو يرجع `-ENOENT`.
- الـ **caller** هو المسؤول عن إن مفيش أي device لسه محتاج الـ table دي.

**Who calls it:** Platform teardown code أو وقت module unload.

---

### Group 2: GPIO Hog Management

**الـ GPIO hogging** معناه إن الـ kernel بيستولي على GPIO line ويحتجزه لنفسه من غير ما يكون فيه driver صريح بيطلبه. الـ hog بيتطبق تلقائياً لما الـ GPIO chip يتسجل. الفرق بين الـ hog والـ lookup إن الـ lookup بيربط GPIO بـ device، أما الـ hog فهو exclusive ownership من غير consumer device.

---

#### `gpiod_add_hogs()`

```c
void gpiod_add_hogs(struct gpiod_hog *hogs);
```

**الـ function بتعمل إيه:**
بتسجل array من الـ `gpiod_hog` entries في الـ global `gpio_machine_hogs` list. لو الـ chip المقصود متسجل فعلاً، بتحاول تـ hog الـ lines فوراً في نفس الـ call.

**Parameters:**
- `hogs` — pointer لأول عنصر في array من `gpiod_hog`. الـ array لازم تنتهي بـ sentinel entry فيها `chip_label = NULL`.

**Return value:** `void`.

**Key details:**
- بتاخد `gpio_machine_hogs_mutex`.
- لكل entry، بتدور على الـ chip بـ `gpio_device_find_by_label(hog->chip_label)`.
- لو لقت الـ chip، بتستدعي `gpiochip_machine_hog()` اللي بتطلب الـ GPIO descriptor وبتضبط الـ direction/value.
- لو الـ chip مش موجود وقتها، الـ hog هيتطبق later لما الـ chip يتسجل عن طريق `gpiochip_machine_hog()` من `gpiochip_add_data()`.
- الـ returned `gpio_device` pointer بيتحرر تلقائياً بـ `__free(gpio_device_put)` (scoped cleanup).

**Pseudocode:**
```c
void gpiod_add_hogs(hogs):
    lock(gpio_machine_hogs_mutex)
    for hog in hogs (until chip_label == NULL):
        list_add_tail(hog->list, gpio_machine_hogs)
        gdev = gpio_device_find_by_label(hog->chip_label)
        if gdev:
            gc = gpio_device_get_chip(gdev)
            gpiochip_machine_hog(gc, hog)   // apply now
            gpio_device_put(gdev)            // auto-cleanup
    unlock(gpio_machine_hogs_mutex)
```

**Who calls it:** Board init code أو platform drivers في الـ `__init` phase.

---

#### `gpiod_remove_hogs()`

```c
void gpiod_remove_hogs(struct gpiod_hog *hogs);
```

**الـ function بتعمل إيه:**
بتشيل array من الـ hog entries من الـ global list. لكن **لا** بترجع الـ GPIO lines للحالة غير-مـ hogged — يعني الـ descriptors اللي اتأخدت بتفضل محجوزة.

**Parameters:**
- `hogs` — pointer لأول entry في الـ array، منتهية بـ sentinel `{.chip_label = NULL}`.

**Return value:** `void`.

**Key details:**
- بتاخد `gpio_machine_hogs_mutex`.
- بتعمل `list_del()` بس من الـ list، مفيش `gpiod_put()` أو release للـ GPIO.
- ده ممكن يكون مقصوداً (الـ line تفضل محجوزة) أو محتاج الـ caller يعمل cleanup يدوي لو أراد.
- نفس convention الـ sentinel-terminated array زي `gpiod_add_hogs()`.

**Who calls it:** Platform teardown أو module unload code.

---

### Group 3: Build-time Helper Macros

الـ macros دي بتوفر compile-time syntax sugar لبناء الـ data structures. مفيش runtime overhead.

---

#### `GPIO_LOOKUP_IDX()`

```c
#define GPIO_LOOKUP_IDX(_key, _chip_hwnum, _con_id, _idx, _flags) \
(struct gpiod_lookup) {                                            \
    .key        = _key,                                            \
    .chip_hwnum = _chip_hwnum,                                     \
    .con_id     = _con_id,                                         \
    .idx        = _idx,                                            \
    .flags      = _flags,                                          \
}
```

**الـ macro بيعمل إيه:**
بيبني `struct gpiod_lookup` بـ compound literal. ده الـ primitive macro اللي بيستخدمه كل اللي تحته.

**Parameters:**
- `_key` — اسم الـ chip (مثلاً `"gpiochip0"`) أو اسم الـ GPIO line. لو `_chip_hwnum == U16_MAX` بيتعامل معاه كـ line name.
- `_chip_hwnum` — الـ hardware index للـ GPIO جوا الـ chip (يبدأ من 0). أو `U16_MAX` لـ lookup by name.
- `_con_id` — الاسم من وجهة نظر الـ consumer device (مثلاً `"reset"`, `"cs"`, `"enable"`).
- `_idx` — الـ index لو الـ consumer بيستخدم أكتر من GPIO بنفس `con_id`.
- `_flags` — bitmask من `gpio_lookup_flags`.

**Usage example:**
```c
// SPI chip select — GPIO 12 من gpiochip0، active low
GPIO_LOOKUP_IDX("gpiochip0", 12, "cs", 0, GPIO_ACTIVE_LOW)

// مصفوفة GPIOs بنفس الاسم "led"
GPIO_LOOKUP_IDX("gpiochip1", 5,  "led", 0, GPIO_ACTIVE_HIGH)
GPIO_LOOKUP_IDX("gpiochip1", 6,  "led", 1, GPIO_ACTIVE_HIGH)
GPIO_LOOKUP_IDX("gpiochip1", 7,  "led", 2, GPIO_ACTIVE_HIGH)
// consumer بيستخدم gpiod_get_index(dev, "led", 0/1/2)
```

---

#### `GPIO_LOOKUP()`

```c
#define GPIO_LOOKUP(_key, _chip_hwnum, _con_id, _flags) \
    GPIO_LOOKUP_IDX(_key, _chip_hwnum, _con_id, 0, _flags)
```

**الـ macro بيعمل إيه:**
اختصار لـ `GPIO_LOOKUP_IDX` لما بيكون فيه GPIO واحد بس تحت الـ `con_id` (الـ idx بيبقى 0 تلقائياً).

**Parameters:** نفس `GPIO_LOOKUP_IDX` من غير الـ `_idx`.

---

#### `GPIO_LOOKUP_SINGLE()`

```c
#define GPIO_LOOKUP_SINGLE(_name, _dev_id, _key, _chip_hwnum, _con_id, _flags) \
static struct gpiod_lookup_table _name = {                       \
    .dev_id = _dev_id,                                           \
    .table = {                                                   \
        GPIO_LOOKUP(_key, _chip_hwnum, _con_id, _flags),         \
        {},   /* sentinel */                                     \
    },                                                           \
}
```

**الـ macro بيعمل إيه:**
بيعرف `static struct gpiod_lookup_table` كاملة باسم `_name` بـ entry واحدة بس. ده الأشيع في الـ simple board files.

**Parameters:**
- `_name` — اسم الـ C variable اللي هيتعرف.
- `_dev_id` — الـ device name (زي `dev_name(dev)`) اللي الـ table دي مخصصة ليه.
- باقي الـ parameters بتتعدى لـ `GPIO_LOOKUP()`.

**Usage example:**
```c
// في board file
GPIO_LOOKUP_SINGLE(myboard_spi_gpios,
                   "spi0.0",       // dev_id
                   "gpiochip0",    // chip key
                   8,              // hwnum
                   "cs",           // con_id
                   GPIO_ACTIVE_LOW);

// في __init
gpiod_add_lookup_table(&myboard_spi_gpios);
```

**Key detail:** الـ table دي `static` ومفيش لها memory management — الـ pointer بيفضل valid طول حياة الـ module.

---

#### `GPIO_HOG()`

```c
#define GPIO_HOG(_chip_label, _chip_hwnum, _line_name, _lflags, _dflags) \
(struct gpiod_hog) {                                                      \
    .chip_label = _chip_label,                                            \
    .chip_hwnum = _chip_hwnum,                                            \
    .line_name  = _line_name,                                             \
    .lflags     = _lflags,                                                \
    .dflags     = _dflags,                                                \
}
```

**الـ macro بيعمل إيه:**
بيبني `struct gpiod_hog` بـ compound literal. بيتستخدم عادةً جوا array منتهية بـ sentinel.

**Parameters:**
- `_chip_label` — label الـ GPIO chip (مطابق لـ `gpio_chip.label`).
- `_chip_hwnum` — hardware number للـ GPIO line جوا الـ chip.
- `_line_name` — الاسم اللي هيظهر في `/sys/kernel/debug/gpio` وفي الـ tools.
- `_lflags` — lookup flags (`gpio_lookup_flags`) — polarity، pull، إلخ.
- `_dflags` — direction flags من `gpiod_flags` enum (مثلاً `GPIOD_OUT_HIGH`, `GPIOD_IN`).

**Usage example:**
```c
// Board لازم تحجز GPIO 3 من gpiochip0 كـ output high من البداية
static struct gpiod_hog myboard_hogs[] = {
    GPIO_HOG("gpiochip0", 3, "wifi-enable",
             GPIO_ACTIVE_HIGH, GPIOD_OUT_HIGH),
    GPIO_HOG("gpiochip0", 7, "usb-hub-reset",
             GPIO_ACTIVE_LOW, GPIOD_OUT_LOW),
    {}  // sentinel
};

// في __init
gpiod_add_hogs(myboard_hogs);
```

---

### تدفق البيانات الكامل (End-to-End Flow)

```
Board File (__init)
    │
    ├─ GPIO_LOOKUP_IDX(...)        ← بناء struct gpiod_lookup
    ├─ gpiod_add_lookup_table()    ← تسجيل في gpio_lookup_list
    │
    └─ GPIO_HOG(...)               ← بناء struct gpiod_hog
       gpiod_add_hogs()            ← تسجيل + تطبيق فوري لو chip موجود

Driver probe() / consumer
    │
    └─ gpiod_get(dev, "cs", ...)
           │
           └─ gpiod_find_and_request()
                  │
                  └─ يبحث في gpio_lookup_list بـ dev_id
                         │
                         └─ يلاقي gpiod_lookup entry
                                │
                                └─ يطلب الـ GPIO descriptor
                                   ويرجعه للـ driver
```

---

### ملاحظات مهمة على الـ Locking

| Operation | Lock المستخدم |
|---|---|
| `gpiod_add_lookup_table()` | `gpio_lookup_lock` (mutex) |
| `gpiod_add_lookup_tables()` | `gpio_lookup_lock` (mutex) |
| `gpiod_remove_lookup_table()` | `gpio_lookup_lock` (mutex) |
| `gpiod_add_hogs()` | `gpio_machine_hogs_mutex` |
| `gpiod_remove_hogs()` | `gpio_machine_hogs_mutex` |

- الـ **lookup lock** و**hog lock** هما mutexes منفصلين — ممكن يتأخدوا بشكل مستقل.
- الـ functions دي **ممكن تنام** (mutex، مش spinlock) — لا تستدعيها من atomic context.
- الـ macros (GPIO_LOOKUP، GPIO_HOG، إلخ) **compile-time only** — zero runtime locking.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. Debugfs Entries

**الـ debugfs** بيوفر معلومات تفصيلية عن كل GPIO chip وكل line مسجلة في الـ kernel.

```bash
# Mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# اقرأ حالة كل GPIO chips والـ lines
cat /sys/kernel/debug/gpio
```

**مثال على الـ output:**

```
gpiochip0: GPIOs 0-31, parent: platform/fe200000.gpio, pinctrl-bcm2835:
 gpio-5   (                    |reset               ) out lo
 gpio-17  (                    |enable              ) out hi
 gpio-22  (                    |irq                 ) in  lo IRQ

gpiochip1: GPIOs 32-53, parent: platform/fe200000.gpio, pinctrl-bcm2835:
 gpio-40  (                    |) in  hi
```

**تفسير الـ output:**
- **اسم الـ chip** + base GPIO number
- الـ `con_id` (consumer name) اللي اتحدد في `gpiod_lookup.con_id`
- الاتجاه: `in` / `out`
- القيمة الحالية: `hi` / `lo`
- لو فيه IRQ مربوط

**التحقق من الـ lookup tables المسجلة:**

```bash
# مفيش debugfs entry مباشر للـ lookup tables،
# لكن ممكن تشوف الـ consumers اللي طلبوا GPIOs
cat /sys/kernel/debug/gpio | grep -A2 "gpio-"
```

**التحقق من الـ hogs:**

```bash
# الـ hogged lines بتظهر في debugfs بدون consumer device
# بيبان consumer name = line_name اللي اتحدد في gpiod_hog
cat /sys/kernel/debug/gpio | grep "hog\|HOG"
```

---

#### 2. Sysfs Entries

**الـ sysfs** بيكشف كل GPIO chip كـ class device.

```bash
# قائمة كل GPIO chips
ls /sys/class/gpio/

# معلومات عن chip معين
cat /sys/class/gpio/gpiochip0/label
cat /sys/class/gpio/gpiochip0/base
cat /sys/class/gpio/gpiochip0/ngpio

# export GPIO معين للـ userspace (legacy interface)
echo 17 > /sys/class/gpio/export
cat /sys/class/gpio/gpio17/direction
cat /sys/class/gpio/gpio17/value
cat /sys/class/gpio/gpio17/active_low

# الـ GPIO character device (الـ modern interface)
ls /dev/gpiochip*
```

**الـ GPIO character device (libgpiod):**

```bash
# عرض كل lines في chip
gpioinfo gpiochip0

# مثال output:
# gpiochip0 - 54 lines:
#   line   5:   "reset"    "mydevice"  output  active-high [used]
#   line  17:   "enable"   "mydevice"  output  active-high [used]
#   line  22:   "irq"      "mydevice"  input   active-high [used]
```

**ملاحظة:** الـ `key` في `gpiod_lookup` بيتطابق مع `label` في `/sys/class/gpio/gpiochipX/label`.

---

#### 3. Ftrace — Tracepoints والـ Events

**الـ GPIO subsystem** عنده tracepoints جاهزة.

```bash
# اعرض كل GPIO tracepoints المتاحة
ls /sys/kernel/debug/tracing/events/gpio/

# Events المتاحة:
# gpio_direction  - كل مرة بيتغير direction
# gpio_value      - كل مرة بتتقرأ/تتكتب قيمة
```

**تفعيل الـ tracing:**

```bash
# تفعيل GPIO events
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو event معين
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# تشغيل الـ tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شوف الـ output
cat /sys/kernel/debug/tracing/trace

# مثال output:
# mydriver-123  [001] ....  123.456789: gpio_direction: gpio=17 in=0
# mydriver-123  [001] ....  123.456800: gpio_value: gpio=17 get=0 value=1
```

**تتبع `gpiod_add_lookup_table` و `gpiod_add_hogs`:**

```bash
# ftrace function tracer لتتبع دوال الـ GPIO machine
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'gpiod_add_lookup_table
gpiod_add_hogs
gpiod_find
gpiod_get' > /sys/kernel/debug/tracing/set_ftrace_filter

echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل الـ driver
cat /sys/kernel/debug/tracing/trace
```

---

#### 4. Printk والـ Dynamic Debug

**تفعيل dynamic debug لـ GPIO subsystem:**

```bash
# تفعيل كل debug messages في gpio core
echo 'file drivers/gpio/gpiolib.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/gpiolib-of.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/gpiolib-acpi.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لملف machine.h المرتبط
echo 'file drivers/gpio/gpiolib.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# +p = print, +f = function name, +l = line number, +m = module, +t = thread ID

# أو تفعيل كل GPIO messages دفعة واحدة
echo 'module gpiolib +p' > /sys/kernel/debug/dynamic_debug/control
```

**تعديل loglevel أثناء الـ runtime:**

```bash
# رفع الـ loglevel لتشوف KERN_DEBUG messages
echo 8 > /proc/sys/kernel/printk

# متابعة الـ kernel log
dmesg -w | grep -i gpio
```

**رسائل مفيدة في الـ kernel log:**

```bash
# لما الـ driver بيسجل lookup table
dmesg | grep "GPIO lookup"

# لما GPIO بيتطلب
dmesg | grep "gpiod_get\|gpio request"
```

---

#### 5. Kernel Config Options للـ Debugging

| Option | الوصف |
|--------|-------|
| `CONFIG_GPIOLIB` | تفعيل الـ GPIO library — **أساسي** |
| `CONFIG_GPIO_SYSFS` | كشف GPIOs في `/sys/class/gpio` |
| `CONFIG_DEBUG_GPIO` | تفعيل runtime checks إضافية في الـ GPIO core |
| `CONFIG_GPIOLIB_IRQCHIP` | دعم IRQ عبر GPIO، مفيد لـ debugging interrupts |
| `CONFIG_GPIO_CDEV` | الـ character device interface (gpiochip) |
| `CONFIG_GPIO_CDEV_V1` | الـ legacy ioctl interface للـ compatibility |
| `CONFIG_GENERIC_PINCONF` | دعم pin configuration debugging |
| `CONFIG_DEBUG_FS` | لازم يكون مفعّل عشان debugfs يشتغل |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug messages |
| `CONFIG_TRACING` | تفعيل ftrace |
| `CONFIG_GPIO_MOCKUP` | GPIO chip وهمي للـ testing بدون hardware |

**تفعيل `CONFIG_DEBUG_GPIO` بيضيف:**
- تحقق من إن الـ GPIO مش بيتطلب مرتين
- تحقق من الـ direction قبل القراءة/الكتابة
- رسائل تفصيلية لكل `gpiod_get` فاشل

```bash
# تحقق من الـ config في الـ kernel الحالي
zcat /proc/config.gz | grep -E 'CONFIG_GPIO|CONFIG_DEBUG_GPIO'
# أو
cat /boot/config-$(uname -r) | grep -E 'CONFIG_GPIO|CONFIG_DEBUG_GPIO'
```

---

#### 6. أدوات خاصة بالـ Subsystem

**libgpiod — الأداة الرئيسية للـ debugging:**

```bash
# تثبيت
apt install gpiod   # Debian/Ubuntu
dnf install libgpiod-utils  # Fedora

# عرض كل chips وlines
gpiodetect          # يعرض كل gpiochip devices
gpioinfo            # يعرض كل lines في كل chips
gpioinfo gpiochip0  # chip معين

# قراءة قيمة line
gpioget gpiochip0 17

# كتابة قيمة
gpioset gpiochip0 17=1

# مراقبة events على line
gpiomon gpiochip0 22
```

**التحقق من الـ lookup tables المسجلة (kernel module):**

```bash
# مفيش tool مباشرة، لكن ممكن تستخدم /proc/kallsyms
# لو الـ lookup table اتعرّفت كـ symbol
cat /proc/kallsyms | grep "lookup_table\|gpiod_lookup"

# أو تستخدم crash/drgn لقراءة الـ gpio_lookup_list في الـ kernel
```

**pinctrl debugging (مرتبط بالـ GPIO):**

```bash
# حالة كل pins وعلاقتها بـ GPIO
cat /sys/kernel/debug/pinctrl/pinctrl-handles
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# chip معين
ls /sys/kernel/debug/pinctrl/
cat /sys/kernel/debug/pinctrl/fe200000.gpio/pinconf-pins
```

---

#### 7. رسائل الـ Error الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|-----------------|--------|------|
| `gpio-XX (NAME): gpiod_request: status -EBUSY` | الـ GPIO مطلوب من driver تاني | تحقق من `cat /sys/kernel/debug/gpio`، ابحث عن مين بيستخدمه |
| `gpiod_get: no GPIO consumer reset found` | الـ `con_id` مش موجود في أي lookup table أو DT | تأكد إن `gpiod_add_lookup_table` اتنادى قبل `probe`، تحقق من اسم الـ `con_id` |
| `gpio-XX: is already requested` | تكرار طلب نفس الـ GPIO | تأكد من إن `gpiod_put` بيتنادى في `remove` |
| `GPIO lookup for consumer reset failed` | فشل الـ lookup كله (مش DT ومش platform data) | تحقق من إن `key` في `gpiod_lookup` بيطابق اسم الـ chip |
| `gpiochip_add_data: GPIOs X..Y (NAME) failed to add` | تعارض في الـ GPIO number range | تحقق من الـ base address وعدد الـ GPIOs في الـ chip |
| `could not get gpio for CHIPNAME line N` | رقم الـ `chip_hwnum` في `gpiod_lookup` أكبر من عدد الـ lines | تحقق من `ngpio` في `/sys/class/gpio/gpiochipX/ngpio` |
| `GPIO line name "X" not found` | الـ `key` بيشير لـ line name (لما `chip_hwnum == U16_MAX`) وملقاش | تحقق من `gpioinfo` إن الـ line name موجود بالظبط |
| `gpiod_hog: unable to get GPIO chip N` | الـ chip المذكور في `gpiod_hog.chip_label` مش موجود وقت الـ hog | الـ hog بيتنادى قبل ما الـ chip يتسجل |
| `-EPROBE_DEFER` في الـ log | الـ GPIO chip مش ready لسه | طبيعي ومش error، الـ kernel هيعيد الـ probe |
| `WARNING: at drivers/gpio/gpiolib.c` مع `WARN_ON` | استخدام GPIO من غير ما يتطلب | تحقق من الـ driver code |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

**في `gpiod_add_lookup_table`** — تحقق إن الـ table مش NULL:

```c
void gpiod_add_lookup_table(struct gpiod_lookup_table *table)
{
    /* تحقق من صحة الـ table قبل الإضافة */
    if (WARN_ON(!table || !table->dev_id))
        return;
    /* ... */
}
```

**في `gpiod_get` / `gpiod_find`** — لما الـ lookup يفشل:

```c
/* في drivers/gpio/gpiolib.c - دالة gpiod_find */
if (!desc) {
    /* أضف dump_stack هنا لتتبع من طلب الـ GPIO */
    dev_warn(dev, "no GPIO found for %s\n", con_id);
    dump_stack();
}
```

**في `gpiod_add_hogs`** — تحقق من الـ chip:

```c
static void gpiochip_add_hog(struct gpio_chip *chip, struct gpiod_hog *hog)
{
    /* WARN_ON لو chip_hwnum أكبر من ngpio */
    if (WARN_ON(hog->chip_hwnum >= chip->ngpio))
        return;
}
```

**نقاط استراتيجية مقترحة:**

```c
/* 1. في platform driver probe() قبل gpiod_get */
dev_dbg(dev, "requesting GPIO con_id=%s idx=%d\n", con_id, idx);

/* 2. في gpiod_add_lookup_table لتأكيد التسجيل */
pr_debug("GPIO: registered lookup table for device '%s'\n", table->dev_id);

/* 3. لو desc رجع IS_ERR */
if (IS_ERR(desc)) {
    dev_err(dev, "gpiod_get(%s) failed: %ld\n", con_id, PTR_ERR(desc));
    dump_stack();  /* يساعد تعرف الـ call chain */
}
```

---

### Hardware Level

---

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

```bash
# الطريقة الأولى: قارن debugfs بالـ actual hardware
# اقرأ قيمة GPIO من الـ kernel
cat /sys/kernel/debug/gpio

# اقرأ نفس الـ GPIO من userspace
gpioget gpiochip0 17

# قارن بالـ oscilloscope أو multimeter على الـ pin الفعلي

# الطريقة الثانية: force output ثم قيس
gpioset gpiochip0 17=1   # اضبط HIGH
# قيس الجهد على الـ pin — المفروض +3.3V أو +1.8V حسب الـ VCC

gpioset gpiochip0 17=0   # اضبط LOW
# قيس الجهد — المفروض 0V
```

**تحقق من الـ direction:**

```bash
# الـ kernel يقول input أو output؟
cat /sys/kernel/debug/gpio | grep "gpio-17"
# gpio-17  (enable) out hi  ← يعني OUTPUT وقيمته HIGH

# هل الـ hardware فعلاً بيدرايف؟
# استخدم multimeter في وضع current للتحقق إن الـ pin بيدرايف فعلاً
```

---

#### 2. Register Dump Techniques

**باستخدام `devmem2`:**

```bash
# تثبيت
apt install devmem2

# قراءة GPIO register — مثال BCM2835 (Raspberry Pi)
# GPFSEL1 - GPIO Function Select 1 (GPIOs 10-19)
devmem2 0xFE200004 w   # قراءة 32-bit word

# GPLEV0 - GPIO Pin Level 0 (GPIOs 0-31)
devmem2 0xFE200034 w   # قيمة كل GPIOs الـ 32 دفعة واحدة

# مثال output:
# /dev/mem opened.
# Memory mapped at address 0x7f8b4000.
# Value at address 0xFE200034 (0x7f8b4034): 0x00020000
# Bit 17 = 1 → GPIO17 = HIGH
```

**باستخدام `/dev/mem` مباشرة مع Python:**

```python
import mmap, struct

# مثال: BCM2835 GPIO base
GPIO_BASE = 0xFE200000
PAGE_SIZE = 4096

with open('/dev/mem', 'r+b') as f:
    gpio_map = mmap.mmap(f.fileno(), PAGE_SIZE,
                         offset=GPIO_BASE)
    # قراءة GPLEV0 (offset 0x34)
    val = struct.unpack('I', gpio_map[0x34:0x38])[0]
    print(f"GPLEV0 = 0x{val:08X}")
    for i in range(32):
        print(f"GPIO{i:2d} = {(val >> i) & 1}")
```

**باستخدام `io` utility (busybox):**

```bash
# قراءة register
io -4 -r 0xFE200034

# كتابة register (GPSET0 - set GPIO17)
io -4 -w 0xFE20001C 0x00020000
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**الـ Logic Analyzer:**

```
GPIO Machine Debugging:
┌─────────────────────────────────────────────────┐
│  Channel 0: GPIO_RESET (active low)             │
│  ____          ________                         │
│      |________|                                 │
│        ↑                                        │
│   gpiod_set_value(reset_gpio, 1)                │
│                                                 │
│  Channel 1: GPIO_ENABLE                         │
│       ___________________________               │
│  ____|                                          │
│      ↑                                          │
│   gpiod_set_value(enable_gpio, 1)               │
└─────────────────────────────────────────────────┘
```

**نصائح عملية:**

1. **Trigger على الـ reset line** لما الـ driver بيتـ probe — بتشوف exact timing
2. **قيس الـ rise/fall time** — لو بطيء جداً يبقى فيه مشكلة pull resistor أو capacitance
3. **ابحث عن glitches** في الـ active-low signals — ممكن يكون مشكلة في الـ `GPIO_ACTIVE_LOW` flag
4. **قارن timing مع الـ kernel log timestamps** عن طريق sync signal

**الـ Oscilloscope:**

```bash
# قبل ما تشيل الـ probe، افعّل هذا الـ GPIO event tracing
# عشان تعرف exact kernel timestamp
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ driver
modprobe mydriver

# اقرأ الـ trace مع timestamps
cat /sys/kernel/debug/tracing/trace | grep "gpio=17"
# mydriver-123 [001] 1234.567890: gpio_value: gpio=17 get=0 value=1
# قارن 1234.567890 بالـ oscilloscope capture
```

---

#### 4. Common Hardware Issues وpatterns الـ Kernel Log

| المشكلة | Pattern في الـ Kernel Log | السبب المحتمل |
|---------|--------------------------|---------------|
| GPIO مش بيستجاب | `gpiod_get_value` بيرجع دايماً 0 أو 1 | Pin مش connected، أو الـ pull resistor غلط |
| تعارض مع pin multiplexing | `pinctrl: pin already requested` | Pin متحدد كـ GPIO وكـ function تانية في نفس الوقت |
| Signal مقلوب | الـ device مش بيشتغل رغم إن kernel بيقول HIGH | الـ `GPIO_ACTIVE_LOW` flag مش موجود في الـ `gpiod_lookup` |
| الـ IRQ مش بيشتغل | `irq: type mismatch` أو IRQ ما بييجيش | الـ edge trigger غلط أو pin مش configured كـ input |
| Hog مش بيشتغل | `gpiod_hog: unable to get GPIO` | `gpiod_add_hogs` بيتنادى قبل `gpiochip_add` |
| Floating pin | قيم random في `gpio_value` events | لازم تعرّف `GPIO_PULL_UP` أو `GPIO_PULL_DOWN` في الـ flags |
| Drive strength ضعيف | جهد منخفض عن المتوقع على الـ pin | الـ hardware بيحتاج تعديل في pinconf |

---

#### 5. Device Tree Debugging

**التحقق إن الـ DT بيطابق الـ Hardware:**

```bash
# اقرأ الـ compiled DT
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -A5 "gpio"

# أو اقرأ الـ raw DTS
cat /sys/firmware/devicetree/base/model
ls /sys/firmware/devicetree/base/

# تحقق من GPIO specifiers في الـ DT
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | \
    grep -B2 -A5 "gpios\|reset-gpios\|enable-gpios"
```

**مثال DT صح:**

```dts
mydevice@0 {
    compatible = "vendor,mydevice";

    /* reset-gpios → con_id = "reset" في الـ gpiod_get */
    reset-gpios = <&gpio 17 GPIO_ACTIVE_LOW>;
    enable-gpios = <&gpio 22 GPIO_ACTIVE_HIGH>;
};
```

**ملاحظة:** لو الـ device بيستخدم `gpiod_lookup_table` من platform data (machine.h)، الـ DT GPIO specifiers ممكن يتجاهلوا لو فيه overlap — الـ priority هي: **DT أولاً** ثم **ACPI** ثم **lookup table**.

```bash
# تحقق من overlaps
# لو الـ driver بيستخدم gpiod_lookup_table لكن الـ DT موجود،
# الـ DT هيكسب وهتلاقي في الـ log:
dmesg | grep "GPIO lookup"

# تحقق من الـ GPIO handles في الـ DT
cat /sys/firmware/devicetree/base/soc/mydevice@0/reset-gpios | xxd
```

**فحص الـ pinmux:**

```bash
# تأكيد إن الـ pin مضبوط كـ GPIO مش كـ alternate function
cat /sys/kernel/debug/pinctrl/fe200000.gpio/pinmux-pins | grep "pin 17"
# pin 17 (GPIO17): mydevice GPIO UNCLAIMED

# لو بيقول "UNCLAIMED" يعني مفيش driver طلبه
# لو بيقول اسم function غير GPIO، يبقى فيه مشكلة في الـ pinmux config
```

---

### Practical Commands

---

#### أوامر جاهزة للنسخ والتشغيل

**1. فحص سريع لكل GPIO state:**

```bash
#!/bin/bash
# gpio_debug_snapshot.sh - quick GPIO state snapshot

echo "=== GPIO Chips ==="
gpiodetect

echo ""
echo "=== All GPIO Lines ==="
gpioinfo

echo ""
echo "=== Kernel GPIO State ==="
cat /sys/kernel/debug/gpio

echo ""
echo "=== Pinctrl Handles ==="
cat /sys/kernel/debug/pinctrl/pinctrl-handles 2>/dev/null || echo "pinctrl-handles not available"

echo ""
echo "=== Pinctrl Maps ==="
cat /sys/kernel/debug/pinctrl/pinctrl-maps 2>/dev/null | head -50
```

**2. Enable GPIO tracing كامل:**

```bash
#!/bin/bash
# gpio_trace_enable.sh

# أوقف الـ tracing الأول
echo 0 > /sys/kernel/debug/tracing/tracing_on

# امسح الـ buffer
echo > /sys/kernel/debug/tracing/trace

# فعّل GPIO events
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# فعّل function tracing للدوال الأساسية
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'gpiod_get
gpiod_put
gpiod_set_value
gpiod_get_value
gpiod_add_lookup_table
gpiod_add_hogs
gpiochip_add_data' > /sys/kernel/debug/tracing/set_ftrace_filter

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

echo "Tracing enabled. Run: cat /sys/kernel/debug/tracing/trace_pipe"
echo "Stop with: echo 0 > /sys/kernel/debug/tracing/tracing_on"
```

**3. Dynamic debug للـ GPIO subsystem:**

```bash
#!/bin/bash
# gpio_dyndbg.sh

# فعّل كل debug messages في GPIO
for file in \
    "drivers/gpio/gpiolib.c" \
    "drivers/gpio/gpiolib-of.c" \
    "drivers/gpio/gpiolib-acpi.c" \
    "drivers/gpio/gpiolib-swnode.c"; do
    echo "file $file +pflmt" > /sys/kernel/debug/dynamic_debug/control 2>/dev/null
done

# ارفع الـ log level
echo 8 > /proc/sys/kernel/printk

echo "Dynamic debug enabled. Watch with: dmesg -w | grep -i gpio"
```

**4. تحقق من lookup table registration:**

```bash
# تحقق إن الـ driver مسجّل lookup table
# عن طريق مقارنة /sys/kernel/debug/gpio قبل وبعد تحميل الـ module

# قبل
cat /sys/kernel/debug/gpio > /tmp/gpio_before.txt

# حمّل الـ module
modprobe mydriver

# بعد
cat /sys/kernel/debug/gpio > /tmp/gpio_after.txt

# قارن
diff /tmp/gpio_before.txt /tmp/gpio_after.txt
```

**5. اختبار GPIO بدون driver (userspace):**

```bash
# اطلب GPIO مؤقتاً من userspace لتجربته
gpioget --bias=pull-up gpiochip0 17

# اضبطه output
gpioset --mode=wait gpiochip0 17=1
# في terminal تاني، قيس الجهد، ثم Ctrl+C

# راقب events
gpiomon --num-events=10 --rising-edge --falling-edge gpiochip0 22
# مثال output:
# 1234.567890 rising  gpiochip0 22
# 1234.789012 falling gpiochip0 22
```

**6. فحص الـ hogs:**

```bash
# الـ hogged GPIOs بتبان في debugfs بـ consumer name = line_name
cat /sys/kernel/debug/gpio | grep -v "^$"

# مثال output لـ hogged GPIO:
# gpio-5   (pwr-enable          |pwr-enable          ) out hi [kernel]
#                                  ↑ هنا الـ line_name من gpiod_hog
```

**7. تشخيص `-EPROBE_DEFER`:**

```bash
# لو الـ driver عنده probe_defer، هيبان في
cat /sys/kernel/debug/devices_deferred

# مثال output:
# mydevice: waiting for GPIO consumer
```

**8. فحص `chip_hwnum` صح:**

```bash
# تأكد إن chip_hwnum اللي حطيته في gpiod_lookup صح
# ngpio بتقولك عدد الـ lines (0 إلى ngpio-1)
cat /sys/class/gpio/gpiochip0/ngpio
# 54 → يعني valid range هو 0-53

# اعرض الـ line names المتاحة
gpioinfo gpiochip0 | awk '{print $1, $2, $3}'
# line   0: unnamed        unused
# line  17: "enable"       used
```

**تفسير نتيجة `gpioinfo`:**

```
gpiochip0 - 54 lines:
 line   5: "reset"    "mydevice"  output  active-low  [used]
 │          │           │          │        │           └─ مطلوب من driver
 │          │           │          │        └─────────── GPIO_ACTIVE_LOW في الـ flags
 │          │           │          └──────────────────── الاتجاه
 │          │           └─────────────────────────────── الـ consumer (con_id)
 │          └─────────────────────────────────────────── الـ line name في DT/hardware
 └────────────────────────────────────────────────────── chip_hwnum
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — الـ SPI CS مش شغال

#### العنوان
GPIO lookup بـ wrong chip label بيخلي الـ SPI chip select مش بيتحكم فيه

#### السياق
فريق بيعمل industrial gateway بـ RK3562 على custom board. الـ gateway بيتكلم مع RS485 transceiver عبر SPI، والـ chip select مربوط على `GPIO3_A5`. المشروع بيشتغل بـ platform data (مفيش Device Tree كامل جاهز لأن الـ BSP لسه في مرحلة bring-up).

#### المشكلة
الـ SPI driver بيطلب الـ CS GPIO عبر `gpiod_get(dev, "cs", GPIOD_OUT_HIGH)` — بيرجع `-ENOENT`. الـ SPI transaction مش بتكمل، والـ RS485 chip مش بيستجاوش.

#### التحليل
المهندس عمل lookup table زي ده:

```c
static struct gpiod_lookup_table spi_cs_table = {
    .dev_id = "spi0.0",
    .table = {
        GPIO_LOOKUP("gpio3", 5, "cs", GPIO_ACTIVE_LOW),
        {},
    },
};
```

**الـ key** هنا هو `"gpio3"` — المهندس افترض إن ده اسم الـ chip. بس على RK3562 اسم الـ gpiochip الفعلي اللي الكيرنل بيشوفه هو `"rk806-gpio"` أو `"gpio@fd8a0000"` حسب الـ pinctrl driver.

الكود في `gpio/gpiolib.c` بيعمل match على `gpiod_lookup.key` مع `gpiochip->label`. لو الـ label مش identical، الـ lookup بيفشل وبيرجع `NULL`، وبعدين `gpiod_get` بيرجع `-ENOENT`.

```c
/* struct gpiod_lookup */
struct gpiod_lookup {
    const char *key;       /* must match gpiochip->label exactly */
    u16 chip_hwnum;        /* hardware pin number on that chip */
    const char *con_id;    /* "cs" — what the driver asks for */
    unsigned int idx;
    unsigned long flags;
};
```

#### الحل

**الخطوة الأولى:** اعرف الاسم الصح للـ chip:

```bash
gpioinfo | head -20
# أو
cat /sys/kernel/debug/gpio
```

Output مثلاً:
```
gpiochip0 - 32 lines:  [gpio@fd8a0000]
gpiochip1 - 32 lines:  [gpio@fd8b0000]
gpiochip2 - 32 lines:  [gpio@fd8c0000]
gpiochip3 - 32 lines:  [gpio@fd8d0000]   ← ده اللي عايزه
```

**الخطوة الثانية:** صحح الـ key في الـ lookup table:

```c
static struct gpiod_lookup_table spi_cs_table = {
    .dev_id = "spi0.0",
    .table = {
        /* key must match gpiochip label from gpioinfo */
        GPIO_LOOKUP("gpio@fd8d0000", 5, "cs", GPIO_ACTIVE_LOW),
        {},
    },
};
```

**الخطوة الثالثة:** تأكد إن الـ table اتسجلت قبل ما الـ SPI driver يـ probe:

```c
static int __init gateway_init(void)
{
    gpiod_add_lookup_table(&spi_cs_table);
    return 0;
}
arch_initcall(gateway_init); /* قبل device_initcall */
```

#### الدرس المستفاد
**الـ `key` في `gpiod_lookup` لازم يكون identical لـ `gpiochip->label`** — مش اسم اختصار. دايما استخدم `gpioinfo` أو `/sys/kernel/debug/gpio` تعرف الاسم الصح قبل ما تكتب الـ lookup table.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — HDMI power stuck high

#### العنوان
`GPIO_ACTIVE_LOW` معكوس بيخلي الـ HDMI power rail مش بيتطفى

#### السياق
منتج Android TV box بـ Allwinner H616. الـ HDMI 5V power rail بيتتحكم فيه بـ GPIO عبر P-channel MOSFET — يعني لما الـ GPIO = LOW، الـ HDMI بيشتغل؛ لما الـ GPIO = HIGH، بيتوقف. المهندس عمل hog على الـ GPIO ده عشان يتحكم فيه مباشرة من الـ bootup.

#### المشكلة
بعد boot، الـ HDMI power موجود حتى لما المستخدم يطفى التلفزيون من التطبيق. الـ GPIO stuck high ومش بيتغير.

#### التحليل
الـ hog اتعمل كالآتي:

```c
static struct gpiod_hog hdmi_hogs[] = {
    GPIO_HOG("pio", 36, "hdmi-power", GPIO_ACTIVE_HIGH, GPIOD_OUT_HIGH),
    {},
};
```

**المشكلة:** الـ `lflags = GPIO_ACTIVE_HIGH` مع الـ `dflags = GPIOD_OUT_HIGH` بيخلي الـ physical pin = HIGH. بس الـ MOSFET P-channel محتاج LOW عشان يشغل الـ HDMI.

الكيرنل بيفسر الـ `dflags = GPIOD_OUT_HIGH` كـ "logical high" — وبيطبق الـ polarity من `lflags`:
- لو `GPIO_ACTIVE_HIGH`: logical high → physical HIGH
- لو `GPIO_ACTIVE_LOW`: logical high → physical LOW

```c
/* struct gpiod_hog */
struct gpiod_hog {
    const char *chip_label;
    u16 chip_hwnum;
    const char *line_name;
    unsigned long lflags;  /* polarity */
    int dflags;            /* direction + logical value */
};
```

الـ `GPIO_HOG` macro بيملي الـ struct:

```c
#define GPIO_HOG(_chip_label, _chip_hwnum, _line_name, _lflags, _dflags) \
(struct gpiod_hog) {                                                      \
    .chip_label = _chip_label,                                            \
    .chip_hwnum = _chip_hwnum,                                            \
    .line_name  = _line_name,                                             \
    .lflags     = _lflags,                                                \
    .dflags     = _dflags,                                                \
}
```

#### الحل

صحح الـ `lflags` عشان تعكس الـ polarity:

```c
static struct gpiod_hog hdmi_hogs[] = {
    /*
     * P-channel MOSFET: GPIO LOW = HDMI ON.
     * Use ACTIVE_LOW so logical "high" maps to physical LOW → HDMI ON.
     */
    GPIO_HOG("pio", 36, "hdmi-power", GPIO_ACTIVE_LOW, GPIOD_OUT_HIGH),
    {},
};

static int __init h616_board_init(void)
{
    gpiod_add_hogs(hdmi_hogs);
    return 0;
}
postcore_initcall(h616_board_init);
```

تحقق من الحالة:

```bash
gpioget gpiochip0 36   # لازم تقرأ 0 (physical LOW) لما الـ HDMI شغال
```

#### الدرس المستفاد
**الـ `lflags` في `gpiod_hog` بتحدد الـ polarity**، مش الـ physical state. لما عندك P-channel MOSFET أو أي دائرة inverting، استخدم `GPIO_ACTIVE_LOW` وعبّر عن الحالة المنطقية المطلوبة في `dflags`.

---

### السيناريو الثالث: IoT Sensor Board على STM32MP1 — I2C reset مش بيشتغل عند module unload

#### العنوان
الـ lookup table مش بتتشال عند `rmmod` وبيحصل kernel panic

#### السياق
شركة بتعمل IoT sensor node بـ STM32MP1. الـ I2C sensor (BME280) عنده reset GPIO. الـ platform code اتكتب كـ loadable module عشان يسهل الـ development. المهندس سجل الـ lookup table في `module_init`.

#### المشكلة
عند `rmmod` وبعدين `insmod` تاني، الكيرنل بيـ crash بـ `list_del corruption` في الـ GPIO lookup code.

#### التحليل
الـ module code:

```c
static struct gpiod_lookup_table bme280_gpio_table = {
    .dev_id = "i2c-BME280",
    .table = {
        GPIO_LOOKUP("GPIOE", 12, "reset", GPIO_ACTIVE_LOW | GPIO_OPEN_DRAIN),
        {},
    },
};

static int __init sensor_init(void)
{
    gpiod_add_lookup_table(&bme280_gpio_table);
    return platform_driver_register(&sensor_driver);
}

static void __exit sensor_exit(void)
{
    platform_driver_unregister(&sensor_driver);
    /* ← المهندس نسي gpiod_remove_lookup_table */
}
```

الـ `gpiod_add_lookup_table` بتضيف `bme280_gpio_table.list` لـ global linked list في الكيرنل. لما الـ module بيتـ unload، الـ memory بتتحرر، بس الـ list node لسه موجود في الـ global list. الـ `insmod` التاني بيحاول يضيف نفس الـ address تاني → corruption.

```c
/* الـ API اللي المهندس نسي يستخدمه */
void gpiod_remove_lookup_table(struct gpiod_lookup_table *table);
```

#### الحل

```c
static void __exit sensor_exit(void)
{
    platform_driver_unregister(&sensor_driver);
    /* remove the table before module memory is freed */
    gpiod_remove_lookup_table(&bme280_gpio_table);
}

module_init(sensor_init);
module_exit(sensor_exit);
```

تحقق من غياب الـ table بعد الـ unload:

```bash
rmmod sensor_module
cat /sys/kernel/debug/gpio_lookup_tables  # لو موجود
# أو
dmesg | grep bme280
```

#### الدرس المستفاد
**كل `gpiod_add_lookup_table` لازم يقابله `gpiod_remove_lookup_table`** في الـ cleanup path. الـ struct بيتضاف كـ list node في global list — لو الـ memory اتحررت والـ node لسه في الـ list، الـ corruption حتمي.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — عندنا عدة LEDs بنفس الـ con_id

#### العنوان
استخدام `GPIO_LOOKUP` بدل `GPIO_LOOKUP_IDX` بيخلي بس أول LED بيشتغل

#### السياق
ECU للسيارة بـ i.MX8MQ. الـ board عندها 4 status LEDs (أحمر، أصفر، أخضر، أزرق) كلها مربوطة على نفس الـ LED driver. الـ driver بيطلبهم بـ `gpiod_get_index(dev, "led", 0..3, ...)`.

#### المشكلة
بس الـ LED الأول (index 0) بيشتغل. الـ `gpiod_get_index(dev, "led", 1, ...)` بيرجع `-ENOENT`.

#### التحليل
المهندس عمل الـ table كالآتي:

```c
static struct gpiod_lookup_table ecu_leds_table = {
    .dev_id = "ecu-leds",
    .table = {
        /* wrong: GPIO_LOOKUP always sets idx=0 */
        GPIO_LOOKUP("GPIO1", 10, "led", GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP("GPIO1", 11, "led", GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP("GPIO1", 12, "led", GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP("GPIO1", 13, "led", GPIO_ACTIVE_HIGH),
        {},
    },
};
```

الـ `GPIO_LOOKUP` macro بيوسع لـ `GPIO_LOOKUP_IDX` مع `_idx = 0` دايما:

```c
#define GPIO_LOOKUP(_key, _chip_hwnum, _con_id, _flags) \
    GPIO_LOOKUP_IDX(_key, _chip_hwnum, _con_id, 0, _flags)
/*                                              ↑ always zero */
```

يعني الـ table فيها 4 entries بنفس `con_id = "led"` وكلهم `idx = 0`. الكيرنل بيلاقي أول match وبيقف — الـ entries التانية ملهاش index مختلف يميزهم.

#### الحل

استخدم `GPIO_LOOKUP_IDX` صح:

```c
static struct gpiod_lookup_table ecu_leds_table = {
    .dev_id = "ecu-leds",
    .table = {
        /* correct: each LED gets a unique idx */
        GPIO_LOOKUP_IDX("GPIO1", 10, "led", 0, GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP_IDX("GPIO1", 11, "led", 1, GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP_IDX("GPIO1", 12, "led", 2, GPIO_ACTIVE_HIGH),
        GPIO_LOOKUP_IDX("GPIO1", 13, "led", 3, GPIO_ACTIVE_HIGH),
        {},
    },
};
```

التحقق في الـ driver:

```c
/* driver code */
for (int i = 0; i < 4; i++) {
    leds[i] = gpiod_get_index(dev, "led", i, GPIOD_OUT_LOW);
    if (IS_ERR(leds[i]))
        dev_err(dev, "failed to get led %d\n", i);
}
```

#### الدرس المستفاد
**`GPIO_LOOKUP` دايما بيضع `idx = 0`**. لو عندك أكتر من GPIO بنفس الـ `con_id`، لازم تستخدم `GPIO_LOOKUP_IDX` وتعطي كل واحد index فريد. ده اللي بيخلي `gpiod_get_index()` يقدر يميز بينهم.

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — GPIO line name lookup مع duplicate names

#### العنوان
استخدام line name كـ key بدل chip label بيجيب GPIO غلط لأن الاسم مش unique

#### السياق
مهندس بيعمل bring-up لـ custom AM62x board. الـ board عندها reset GPIO لـ Ethernet PHY. مفيش chip label واضح في الـ early bring-up، فالمهندس قرر يستخدم GPIO line name (`"eth-reset"`) كـ key بدل chip label، وده ممكن عبر وضع `chip_hwnum = U16_MAX`.

#### المشكلة
الـ Ethernet PHY مش بتتـ reset صح. المهندس لاحظ إن فيه GPIO تاني في الـ system اسمه كمان `"eth-reset"` — موجود في GPIO expander مربوط على I2C.

#### التحليل
الـ lookup table:

```c
static struct gpiod_lookup_table eth_reset_table = {
    .dev_id = "dwc3-eth",
    .table = {
        /*
         * U16_MAX means: use @key as GPIO line name, not chip label.
         * Danger: line names are NOT guaranteed unique across the system!
         */
        GPIO_LOOKUP("eth-reset", U16_MAX, "reset", GPIO_ACTIVE_LOW),
        {},
    },
};
```

من docstring الـ `gpiod_lookup` struct في الملف:

```c
/**
 * @key: either the name of the chip the GPIO belongs to, or the GPIO line name
 *       Note that GPIO line names are not guaranteed to be globally unique,
 *       so this will use the first match found!
 */
```

الكيرنل بيعمل `strcmp` على كل line name في كل chip لحد ما يلاقي أول match — اللي كانت الـ GPIO expander مش SoC GPIO.

تحقق من الـ duplicate:

```bash
gpioinfo | grep "eth-reset"
# Output:
# gpiochip2 line 3:  "eth-reset" unused input active-high   ← expander
# gpiochip0 line 47: "eth-reset" unused input active-high   ← SoC
```

#### الحل

الأفضل: استخدم chip label مع hardware number بدل line name:

```c
static struct gpiod_lookup_table eth_reset_table = {
    .dev_id = "dwc3-eth",
    .table = {
        /*
         * Use chip label + hwnum for unambiguous lookup.
         * AM62x SoC GPIO0 chip label from gpioinfo.
         */
        GPIO_LOOKUP("600000.gpio", 47, "reset", GPIO_ACTIVE_LOW),
        {},
    },
};
```

لو line name ضروري (مثلاً الـ label بيتغير بين boards)، تأكد إن الاسم unique:

```bash
gpioinfo | grep "eth-reset" | wc -l
# يجب يرجع 1 فقط
```

وبالإضافة، ممكن تعمل audit سريع عند الـ boot:

```c
/* في board init code */
static int __init am62x_board_init(void)
{
    pr_info("registering eth-reset GPIO lookup\n");
    gpiod_add_lookup_table(&eth_reset_table);
    return 0;
}
arch_initcall(am62x_board_init);
```

#### الدرس المستفاد
**الـ GPIO line names مش guaranteed تكون globally unique** — ده مذكور صراحة في الـ `gpiod_lookup` struct comment. لو عندك choice، استخدم chip label + hardware number عشان الـ lookup يكون deterministic. الـ line name lookup مفيد بس لما بتضمن uniqueness أو بتعمله explicit verification.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ GPIO descriptor API والـ machine lookup tables:

| المقال | الأهمية |
|--------|---------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة للـ GPIO subsystem في الكيرنل |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | بيشرح التوجه نحو الـ descriptor-based API وإزاي هيخلي الكود أكتر أمان |
| [gpio: introduce descriptor-based interface](https://lwn.net/Articles/531848/) | أول patch قدّم الـ `gpiod_*` API اللي فيه `gpiod_lookup` |
| [New descriptor-based GPIO interface](https://lwn.net/Articles/565662/) | شرح الـ API الجديد بعد ما اتقبل في الكيرنل |
| [Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/) | أول توثيق رسمي للـ interface الجديد |
| [gpio: Add GPIO Aggregator Driver](https://lwn.net/Articles/798971/) | driver بيجمع GPIOs متعددة في chip واحدة — بيستخدم نفس مفهوم الـ lookup |
| [gpio: consumer: new virtual driver](https://lwn.net/Articles/942479/) | driver افتراضي بيسمح بإنشاء lookup tables ديناميكياً عن طريق configfs |

---

### توثيق الكيرنل الرسمي

**الـ `Documentation/` paths** الأساسية:

```
Documentation/driver-api/gpio/board.rst      ← شرح gpiod_lookup_table و GPIO_LOOKUP macros
Documentation/driver-api/gpio/consumer.rst   ← إزاي الـ drivers بتستخدم gpiod_get()
Documentation/driver-api/gpio/driver.rst     ← إزاي تكتب GPIO chip driver
Documentation/driver-api/gpio/index.rst      ← فهرس كامل للـ GPIO subsystem
Documentation/driver-api/gpio/legacy.rst     ← الـ integer-based API القديم
```

**الروابط المباشرة:**

- [GPIO Mappings — kernel.org docs](https://docs.kernel.org/driver-api/gpio/board.html) ← الأهم لـ `machine.h`
- [General Purpose Input/Output (GPIO)](https://www.kernel.org/doc/html/latest/driver-api/gpio/index.html)
- [Legacy GPIO Interfaces](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html)
- [GPIO board.txt (older format)](https://www.kernel.org/doc/Documentation/gpio/board.txt)

---

### Commits وـ Patches مهمة

| الـ patch / commit | الوصف |
|--------------------|-------|
| [gpiolib: introduce descriptor-based GPIO interface](https://lwn.net/Articles/528226/) | الـ commit الأصلي اللي حول الـ GPIO من integers لـ descriptors |
| [gpio: export add/remove lookup table functions](https://lore.kernel.org/patchwork/patch/782055/) | export الـ `gpiod_add_lookup_table` و`gpiod_remove_lookup_table` للـ modules |
| [gpiolib: Add stubs for gpiod lookup table interface](https://lore.kernel.org/lkml/1494527071-6412-1-git-send-email-agust@denx.de/) | إضافة stubs لـ `!CONFIG_GPIOLIB` عشان الـ build ميفشلش |
| [ARM: OMAP1: ams-delta: add GPIO lookup tables](https://lore.kernel.org/linux-kernel//20180518210954.29044-4-jmkrzyszt@gmail.com/T/) | مثال حقيقي على تحويل board قديم لاستخدام `gpiod_lookup_table` |

---

### نقاشات الـ Mailing List

- [linux-gpio mailing list على lore.kernel.org](https://lore.kernel.org/linux-gpio/) — الأرشيف الرسمي لكل النقاشات على الـ GPIO subsystem
- [gpiolib stubs patch discussion على mail-archive](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1386243.html) — نقاش إضافة الـ stubs للـ `!GPIOLIB` case

للبحث في نقاشات تانية:
```
https://lore.kernel.org/linux-gpio/?q=gpiod_lookup_table
https://lore.kernel.org/linux-gpio/?q=GPIO_LOOKUP
https://lore.kernel.org/linux-gpio/?q=gpiod_hog
```

---

### كتب مقترحة

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 1**: تعريف الـ kernel modules والـ platform devices
- **الفصل 14**: The Linux Device Model — بيشرح `struct device` اللي بتستخدمه الـ lookup tables
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Ed.)
- **الفصل 17**: Devices and Modules — بيشرح إزاي الـ platform data بتتمرر للـ drivers
- مهم لفهم العلاقة بين `struct platform_device` والـ `gpiod_lookup_table`

#### Embedded Linux Primer — Christopher Hallinan (2nd Ed.)
- **الفصل 15**: Debugging Embedded Linux — بيتكلم عن GPIO debugging
- **الفصل 16**: Porting Linux to a Custom Platform — بيشرح إزاي تعمل board file وتضيف GPIO lookups

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds (3rd Ed.)
- **الفصل 11**: Interfacing with Device Drivers — بيغطي الـ `libgpiod` وعلاقتها بالـ kernel descriptor API

---

### Kernelnewbies.org

- [Linux_6.2 — GPIO changes](https://kernelnewbies.org/Linux_6.2) — بيوثق تغييرات الـ GPIO في كل release
- [Linux_6.6](https://kernelnewbies.org/Linux_6.6) — تغييرات إضافية في الـ GPIO subsystem
- [Linux_6.13](https://kernelnewbies.org/Linux_6.13) — أحدث تغييرات
- [LinuxChanges](https://kernelnewbies.org/LinuxChanges) — تتبع كل تغييرات الـ kernel عبر الإصدارات

**نقاشات الـ mailing list على kernelnewbies:**
- [Using gpio from device tree on platform devices](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html)
- [devm_gpiod_get usage to get the gpio num](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html)

---

### eLinux.org

- [New GPIO Interface for User Space (PDF — ELCE 2017)](https://elinux.org/images/7/74/Elce2017_new_GPIO_interface.pdf) — عرض شامل للـ character device interface الجديد وعلاقته بالـ descriptor API
- [Introduction to pin muxing and GPIO control under Linux (PDF — ELC 2021)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) — بيشرح pinctrl وGPIO معاً من الأساس
- [EBC gpio Polling and Interrupts](https://elinux.org/EBC_gpio_Polling_and_Interrupts) — أمثلة عملية للتعامل مع GPIO

---

### Search Terms للبحث عن معلومات أكتر

للبحث في مصادر تانية استخدم الـ keywords دي:

```
gpiod_lookup_table platform data kernel
GPIO_LOOKUP GPIO_LOOKUP_IDX macro kernel
gpiod_add_lookup_table gpiod_remove_lookup_table
gpio machine descriptor board file
gpiod_hog kernel driver
gpio_lookup_flags ACTIVE_LOW OPEN_DRAIN kernel
devm_gpiod_get platform device lookup
gpiolib descriptor interface linus walleij
linux gpio consumer driver lookup
```

**للبحث في git log:**
```bash
git log --oneline --all -- include/linux/gpio/machine.h
git log --oneline --all -- drivers/gpio/gpiolib.c | grep -i "lookup\|hog\|machine"
```
## Phase 8: Writing simple module

### الفكرة

**`gpiod_add_lookup_table`** هي أنسب function نعمل عليها kprobe — بيتم استدعاؤها وقت تسجيل أي lookup table جديدة للـ GPIO من أي driver أو platform code، فبنقدر نشوف مين بيضيف جدول وإيه هو الـ `dev_id` بتاعه.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_gpio_lookup.c
 * Hooks gpiod_add_lookup_table() to log every GPIO lookup table
 * registration that happens at runtime.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* kprobe API */
#include <linux/gpio/machine.h> /* struct gpiod_lookup_table */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("GPIO Watcher");
MODULE_DESCRIPTION("kprobe on gpiod_add_lookup_table to log GPIO lookup table registrations");

/* ---------------------------------------------------------------------------
 * kprobe handler — fires just BEFORE gpiod_add_lookup_table() executes
 * ---------------------------------------------------------------------------
 * pt_regs carries the CPU registers at the moment of the probe hit.
 * On x86-64: RDI = first argument = pointer to gpiod_lookup_table.
 * On arm64:  X0  = first argument.
 * regs_get_kernel_argument(regs, 0) is arch-agnostic.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve the first argument: pointer to gpiod_lookup_table */
    struct gpiod_lookup_table *tbl =
        (struct gpiod_lookup_table *)regs_get_kernel_argument(regs, 0);

    if (!tbl)
        return 0;

    /* Log the device ID this table belongs to (may be NULL for global tables) */
    pr_info("kprobe_gpio_lookup: gpiod_add_lookup_table() called | dev_id=\"%s\"\n",
            tbl->dev_id ? tbl->dev_id : "<NULL>");

    /* Walk the flexible array and print every entry until sentinel (key==NULL) */
    {
        int i = 0;
        const struct gpiod_lookup *entry = tbl->table;

        while (entry[i].key != NULL) {
            pr_info("  entry[%d]: key=\"%s\" hwnum=%u con_id=\"%s\" idx=%u flags=0x%lx\n",
                    i,
                    entry[i].key,
                    entry[i].chip_hwnum,
                    entry[i].con_id ? entry[i].con_id : "<NULL>",
                    entry[i].idx,
                    entry[i].flags);
            i++;
        }
    }

    return 0; /* 0 = continue normal execution of the probed function */
}

/* ---------------------------------------------------------------------------
 * kprobe struct — names the symbol we want to intercept
 * ---------------------------------------------------------------------------
 */
static struct kprobe kp = {
    .symbol_name = "gpiod_add_lookup_table",
    .pre_handler = handler_pre,
};

/* ---------------------------------------------------------------------------
 * module_init — register the kprobe
 * ---------------------------------------------------------------------------
 */
static int __init gpio_lookup_probe_init(void)
{
    int ret = register_kprobe(&kp);

    if (ret < 0) {
        pr_err("kprobe_gpio_lookup: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe_gpio_lookup: planted on %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ---------------------------------------------------------------------------
 * module_exit — MUST unregister before the module is unloaded
 * ---------------------------------------------------------------------------
 */
static void __exit gpio_lookup_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe_gpio_lookup: removed from %s\n", kp.symbol_name);
}

module_init(gpio_lookup_probe_init);
module_exit(gpio_lookup_probe_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `<linux/module.h>` | ماكروهات `module_init` / `module_exit` وـ `MODULE_LICENSE` |
| `<linux/kernel.h>` | `pr_info` / `pr_err` |
| `<linux/kprobes.h>` | الـ `struct kprobe` وـ `register_kprobe` / `unregister_kprobe` |
| `<linux/gpio/machine.h>` | الـ `struct gpiod_lookup_table` و `struct gpiod_lookup` اللي بنفسّر بيهم البيانات |

---

#### الـ `handler_pre`

الـ `pre_handler` بيتشغّل مباشرةً قبل ما الـ kernel ينفّذ `gpiod_add_lookup_table` فعلياً، يعني بنقدر نقرأ الـ argument قبل ما يتعمل فيه أي حاجة.

الـ `regs_get_kernel_argument(regs, 0)` بتجيب أول argument بطريقة portable على x86-64 وـ arm64 بدون ما نحتاج نكتب كود خاص لكل architecture.

بعدين بنعمل loop على الـ flexible array `tbl->table[]` لحد ما نلاقي sentinel entry (الـ `key == NULL`)، ونطبع كل entry بالتفصيل عشان نعرف أي GPIO line بيتسجّل لأي device.

---

#### الـ `kprobe` struct

الـ `symbol_name` بيخلّي الـ kernel يحلّ العنوان من الـ kallsyms وقت الـ `register_kprobe`، فمش محتاج نحط عنوان hardcoded.

---

#### الـ `module_init`

بيسجّل الـ kprobe ويطبع العنوان الفعلي اللي اتزرع فيه الـ probe — مفيد للـ debugging.

لو فشل التسجيل (مثلاً الـ CONFIG_KPROBES مش مفعّل أو الـ symbol مش exported) بيرجع الـ error code ومش بيكمل.

---

#### الـ `module_exit`

**لازم** نعمل `unregister_kprobe` قبل ما المودول يتشال من الذاكرة، لأنه لو الـ probe فضل مزروع وبعدين الـ handler_pre راح من الذاكرة، أي استدعاء لـ `gpiod_add_lookup_table` بعد كده هيعمل kernel panic فوري.

---

### Makefile للبناء

```makefile
obj-m += kprobe_gpio_lookup.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وتجربة

```bash
# بناء المودول
make

# تحميله
sudo insmod kprobe_gpio_lookup.ko

# تحميل أي GPIO driver (مثلاً على Raspberry Pi)
sudo modprobe gpio-pca953x

# مشاهدة اللوج
sudo dmesg | grep kprobe_gpio_lookup

# إزالة المودول
sudo rmmod kprobe_gpio_lookup
```

مثال على الـ output المتوقع:

```
kprobe_gpio_lookup: planted on gpiod_add_lookup_table at ffffffffc0123456
kprobe_gpio_lookup: gpiod_add_lookup_table() called | dev_id="spi0.0"
  entry[0]: key="gpiochip0" hwnum=17 con_id="reset" idx=0 flags=0x1
```
