## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: PIN CONTROL SUBSYSTEM

الـ file ده جزء من **PIN CONTROL SUBSYSTEM** في Linux kernel، المسؤول عنه Linus Walleij، وبيتواصل عن طريق `linux-gpio@vger.kernel.org`.

---

### المشكلة اللي بيحلها

تخيل معايا إنك بتشتري لوحة إلكترونية (مثلاً Raspberry Pi أو أي SoC). الـ SoC ده بيبقى فيه مئات الـ **pins** (أرجل)، وكل pin ممكن يشتغل بأكتر من طريقة:

- يكون **GPIO** (دخل أو خرج رقمي عادي)
- يكون **I2C** (بروتوكول تواصل)
- يكون **UART** (serial communication)
- يكون **SPI** (بروتوكول تاني)
- أو أي function تاني من عشرات الـ functions

المشكلة: مين يقرر أن الـ pin رقم 42 هيشتغل UART وليس GPIO؟ ومين يحدد إن مستوى الجهد بتاعه 3.3V وليس 1.8V؟ ومين يحدد إن فيه pull-up resistor؟

الإجابة: **PIN CONTROL SUBSYSTEM**.

---

### القصة الكاملة

تخيل إن عندك أوركسترا (الـ SoC) فيها 200 عازف (الـ pins). كل عازف يقدر يعزف أكتر من آلة موسيقية. الـ **pinctrl subsystem** هو المايسترو اللي بيقول: "أنت هتعزف كمان"، "وأنت هتعزف بيانو"، "وأنت مش هتعزف خالص دلوقتي".

دلوقتي، مين اللي بيقول للمايسترو إيه الترتيب المطلوب؟ هنا يجي دور `machine.h`.

---

### دور الـ `machine.h` تحديداً

الـ file ده هو **واجهة الـ board/machine** مع الـ pinctrl subsystem. يعني هو المكان اللي فيه code البورد (مثلاً board file أو device tree complement) بيقول للـ kernel:

> "أنا عارف إن الجهاز اللي اسمه `serial0` محتاج الـ pins دي تشتغل UART في الـ state الـ `default`، وتتحول لـ sleep state لما الجهاز ينام."

الـ file بيوفر **mapping table**: جدول بيربط بين:
1. **الجهاز** (device) محتاج إيه
2. **الـ state** اللي المفروض يتحوله
3. **الـ pin group** المطلوب
4. **الـ function** المطلوب (mux)
5. **الـ configurations** (جهد، pull-up، drive strength، إلخ)

---

### المفاهيم الأساسية في الـ File

#### 1. الـ `pinctrl_map_type` — أنواع الـ Mapping

```c
enum pinctrl_map_type {
    PIN_MAP_TYPE_INVALID,        /* غلط */
    PIN_MAP_TYPE_DUMMY_STATE,    /* state وهمي لو مفيش pinctrl حقيقي */
    PIN_MAP_TYPE_MUX_GROUP,      /* اختيار function للـ pin group */
    PIN_MAP_TYPE_CONFIGS_PIN,    /* إعدادات لـ pin واحد */
    PIN_MAP_TYPE_CONFIGS_GROUP,  /* إعدادات لـ group من الـ pins */
};
```

#### 2. الـ `pinctrl_map_mux` — Mux Mapping

```c
struct pinctrl_map_mux {
    const char *group;    /* اسم الـ pin group */
    const char *function; /* اسم الـ function (مثلاً "uart0", "i2c1") */
};
```

يعني: "خد الـ group دي وخليها تشتغل كـ UART".

#### 3. الـ `pinctrl_map_configs` — Configuration Mapping

```c
struct pinctrl_map_configs {
    const char *group_or_pin; /* اسم الـ pin أو الـ group */
    unsigned long *configs;   /* مصفوفة إعدادات (جهد، pull، إلخ) */
    unsigned int num_configs; /* عدد الإعدادات */
};
```

يعني: "خد الـ pin ده وبرمجه بالإعدادات دي (3.3V، pull-up مفعّل، إلخ)".

#### 4. الـ `pinctrl_map` — الجدول الكامل

```c
struct pinctrl_map {
    const char *dev_name;      /* اسم الجهاز المستخدم للـ pins */
    const char *name;          /* اسم الـ state ("default", "sleep", إلخ) */
    enum pinctrl_map_type type; /* نوع الـ mapping */
    const char *ctrl_dev_name; /* اسم الـ pin controller */
    union {
        struct pinctrl_map_mux mux;
        struct pinctrl_map_configs configs;
    } data;
};
```

ده الـ struct الرئيسي — كل سطر في الـ mapping table هو `pinctrl_map` واحدة.

---

### الـ States: إيه معنى "default" و"sleep"؟

الـ `pinctrl-state.h` بيعرّف 4 states أساسية:

| State | متى بيتستخدم |
|-------|-------------|
| `"default"` | الحالة العادية، الجهاز شغال |
| `"init"` | قبل ما driver يعمل probe (لو التحويل لـ default ممكن يسبب glitch) |
| `"idle"` | الجهاز مستنى، بعض الـ clocks متوقفة |
| `"sleep"` | أعمق حالة نوم |

مثلاً UART في الـ `default` بيكون: pins متوصلة بالـ UART hardware.
في الـ `sleep` بيكون: pins بتتحول لـ GPIO high-impedance لتوفير الطاقة.

---

### مثال عملي

تخيل بورد فيها UART متوصل على pin group اسمه `"uart0grp"`:

```c
static const struct pinctrl_map myboard_pinctrl_map[] = {
    /* UART0: default state - pins work as UART */
    PIN_MAP_MUX_GROUP_DEFAULT("serial0.0", "pinctrl.0", "uart0grp", "uart0"),

    /* UART0: sleep state - pins become high-impedance */
    PIN_MAP_MUX_GROUP("serial0.0", PINCTRL_STATE_SLEEP, "pinctrl.0", "uart0grp", "gpio"),

    /* Set drive strength for uart0grp */
    PIN_MAP_CONFIGS_GROUP_DEFAULT("serial0.0", "pinctrl.0", "uart0grp", uart0_configs),
};

/* Register with the subsystem */
pinctrl_register_mappings(myboard_pinctrl_map, ARRAY_SIZE(myboard_pinctrl_map));
```

الـ macros زي `PIN_MAP_MUX_GROUP_DEFAULT` بتسهّل كتابة الجدول من غير تكرار.

---

### مفهوم الـ "HOG"

لو الـ `dev_name` بتاع الـ mapping هو نفس اسم الـ pin controller نفسه، الـ mapping ده بيتسمى **"hog"**. يعني الـ pin controller بيحجز الـ pins دي لنفسه فور ما يتسجل، من غير ما أي جهاز تاني يطلبها. ده بيتستخدم للـ pins اللي البورد محتاجها دايماً مشغولة بطريقة معينة (مثلاً LED أو reset line).

---

### الفرق بين `pinctrl_register_mappings` و `devm_pinctrl_register_mappings`

```c
/* تسجيل يدوي - المبرمج مسؤول عن إلغاء التسجيل */
int pinctrl_register_mappings(const struct pinctrl_map *map,
                              unsigned int num_maps);

/* تسجيل مرتبط بعمر الـ device - بيتلغى تلقائياً لما الجهاز يتحذف */
int devm_pinctrl_register_mappings(struct device *dev,
                                   const struct pinctrl_map *map,
                                   unsigned int num_maps);
```

---

### الـ `pinctrl_provide_dummies()`

لو النظام مش معاه pin controller حقيقي (مثلاً في بيئة تطوير أو emulator)، الدالة دي بتخلي الـ kernel يتعامل مع أي `PIN_MAP_TYPE_DUMMY_STATE` من غير ما يطلع error. ده بيسهّل الـ portability.

---

### ملفات الـ Subsystem اللي المفروض تعرفها

#### الـ Headers الأساسية

| الملف | الدور |
|-------|-------|
| `include/linux/pinctrl/machine.h` | **الملف ده** — واجهة البورد، mapping table |
| `include/linux/pinctrl/pinctrl-state.h` | تعريف الـ states (default/sleep/idle/init) |
| `include/linux/pinctrl/consumer.h` | واجهة الـ drivers اللي بتستخدم الـ pins |
| `include/linux/pinctrl/pinctrl.h` | واجهة تسجيل الـ pin controller نفسه |
| `include/linux/pinctrl/pinmux.h` | واجهة الـ mux operations |
| `include/linux/pinctrl/pinconf.h` | واجهة الـ configuration operations |
| `include/linux/pinctrl/pinconf-generic.h` | generic configuration parameters |
| `include/linux/pinctrl/devinfo.h` | pinctrl info embedded في struct device |

#### الـ Core Files

| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/core.c` | قلب الـ subsystem — إدارة التسجيل والـ mapping |
| `drivers/pinctrl/core.h` | internal headers للـ core |
| `drivers/pinctrl/pinmux.c` | تنفيذ الـ mux logic |
| `drivers/pinctrl/pinconf.c` | تنفيذ الـ configuration logic |
| `drivers/pinctrl/devicetree.c` | تكامل مع الـ Device Tree |

#### الـ Hardware Drivers (أمثلة)

| الملف | الـ Hardware |
|-------|-------------|
| `drivers/pinctrl/pinctrl-amd.c` | AMD SoCs |
| `drivers/pinctrl/freescale/` | NXP/Freescale iMX |
| `drivers/pinctrl/intel/` | Intel platforms |
| `drivers/pinctrl/mediatek/` | MediaTek SoCs |
| `drivers/pinctrl/samsung/` | Samsung Exynos |
| `drivers/pinctrl/bcm/` | Broadcom platforms |

#### الـ Documentation

| الملف | المحتوى |
|-------|---------|
| `Documentation/driver-api/pin-control.rst` | الـ full API documentation |
| `Documentation/devicetree/bindings/pinctrl/` | Device Tree bindings |

---

### العلاقة بين الـ Files بالصورة

```
Board/Machine Code
        │
        │  يستخدم
        ▼
  machine.h  ◄── الملف ده
  (mapping table)
        │
        │  يسجّل فيه
        ▼
  drivers/pinctrl/core.c
  (قلب الـ subsystem)
        │
        ├──► pinmux.c  (يختار function)
        └──► pinconf.c (يضبط إعدادات)
                │
                │  يتصل بـ
                ▼
        HW Driver (pinctrl-amd.c, etc.)
                │
                │  يتحكم في
                ▼
        Physical SoC Pins
```

---

### خلاصة

الـ `machine.h` هو **لغة التواصل** بين البورد والـ pinctrl subsystem. البورد بتقول "الجهاز X محتاج الـ pins دي تشتغل بالطريقة دي في الـ state ده"، وبعدين الـ core بيترجم الكلام ده لأوامر حقيقية على الـ hardware. من غيره، كل جهاز كان هيضطر يتعامل مع الـ pin controller مباشرة، وده كان هيخلي الكود فوضى ومش portable.
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة اللي بيحلها الـ Pinctrl

أي SoC من نوع ARM — خد مثلاً Raspberry Pi CM4 أو STM32MP1 — فيه مئات الـ pins فيزيائية. كل pin ممكن يشتغل بأكتر من وظيفة: UART TX، أو SPI MOSI، أو GPIO عادي، أو PWM. ده اللي بيتسمى **pin multiplexing** أو pinmux.

المشكلة قبل وجود الـ pinctrl subsystem كانت:
- كل driver بيعمل write مباشر على registers الـ SoC عشان يضبط الـ pins
- لو اتنين drivers احتاجوا نفس الـ pin، في crash صامت في الـ hardware
- مافيش centralized ownership — مين بيتحكم في pin 42؟
- الـ board-specific configuration بتتبهدل جوه الـ drivers نفسها بدل ما تبقى منفصلة
- power management: لما الـ system بيخش sleep، لازم الـ pins تتغير configuration — مين بيعمل ده؟

**الملخص:** محتاجين نظام مركزي يمتلك كل الـ pins، يمنع conflicts، ويسمح لـ devices بطلب الـ configuration اللي محتاجاها حسب الـ state (default, sleep, idle).

---

### الحل: الـ Pinctrl Subsystem

الـ kernel عمل subsystem واحد مسؤول عن:

1. **تسجيل كل الـ pins** الموجودة في الـ SoC مع أسمائها وأرقامها
2. **تعريف الـ groups**: مجموعة pins بتشتغل مع بعض كـ function واحدة (مثلاً pins 10,11,12,13 هم UART1)
3. **تعريف الـ functions**: الوظيفة اللي ممكن تتحول ليها group (مثلاً "uart1", "spi0", "i2c1")
4. **الـ mapping table**: جدول بيربط كل device باسمه بالـ group والـ function اللي محتاجها
5. **الـ states**: كل device عنده states مختلفة (default, sleep, idle) وكل state ليه configuration مختلفة

---

### التشبيه من الواقع

تخيل مبنى حكومي كبير فيه **لوحة توصيلات مركزية** (patch panel). كل خط تليفون في المبنى مروّح على اللوحة دي.

| مكون الـ Pinctrl | التشبيه |
|---|---|
| `pinctrl_dev` | اللوحة المركزية نفسها |
| `pinctrl_pin_desc` | كل مقبس في اللوحة باسمه ورقمه |
| `pingroup` | مجموعة مقابس بتمثل خط واحد (مثلاً دور 3) |
| `pinfunction` | الخدمة اللي ممكن يتوصل بيها (تليفون، إنترنت، فاكس) |
| `pinctrl_map` | ورقة التعليمات: "مكتب 301 يحتاج خط إنترنت من المقابس 10-11" |
| `pinctrl_state` | وضع الشركة: عمل عادي vs. بعد ساعات العمل vs. إجازة |
| device driver | الموظف اللي بيطلب الخدمة من اللوحة |

لما موظف (driver) يجي يطلب يشتغل على خط، اللوحة:
- بتتأكد محدش تاني بيستخدمه
- بتوصله بالخدمة الصح
- لما يروح (sleep) بتعيد تعيين الخط لوضع محدد

---

### البيكتشر الكامل: Architecture Diagram

```
  ┌─────────────────────────────────────────────────────────┐
  │                    Linux Kernel                          │
  │                                                         │
  │  ┌──────────────┐    ┌──────────────┐   ┌───────────┐  │
  │  │  UART Driver │    │  SPI Driver  │   │I2C Driver │  │
  │  └──────┬───────┘    └──────┬───────┘   └─────┬─────┘  │
  │         │  devm_pinctrl_get │                 │         │
  │         │  pinctrl_select_state               │         │
  │         └──────────────┬────┘─────────────────┘         │
  │                        │  consumer API                   │
  │               ┌────────▼──────────┐                     │
  │               │   pinctrl core    │                     │
  │               │  (drivers/pinctrl)│                     │
  │               │                  │                     │
  │               │  • owns mapping  │                     │
  │               │    table         │                     │
  │               │  • resolves      │                     │
  │               │    conflicts     │                     │
  │               │  • manages states│                     │
  │               └────────┬─────────┘                     │
  │                        │ pinctrl_ops / pinmux_ops       │
  │                        │ pinconf_ops                    │
  │               ┌────────▼─────────┐                     │
  │               │  pin controller  │  ← SoC-specific     │
  │               │    driver        │    driver            │
  │               │  (e.g. pinctrl-  │                     │
  │               │   bcm2835.c)     │                     │
  │               └────────┬─────────┘                     │
  │                        │ MMIO register writes           │
  └────────────────────────┼────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │     SoC Hardware        │
              │  GPIO/MUX/CONF registers│
              │  Pin 0: UART0_TX        │
              │  Pin 1: UART0_RX        │
              │  Pin 2: SPI0_MOSI       │
              │  ...                    │
              └─────────────────────────┘

  Board-level mapping table (static, defined in board file or DT):
  ┌─────────────────────────────────────────────────────────┐
  │  struct pinctrl_map board_maps[] = {                    │
  │    PIN_MAP_MUX_GROUP_DEFAULT("uart0", "pinctrl0",       │
  │                               "uart0grp", "uart0"),     │
  │    PIN_MAP_CONFIGS_PIN_DEFAULT("uart0", "pinctrl0",     │
  │                                "PIN_A0", uart_cfg),     │
  │  };                                                     │
  └─────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: الـ Mapping Table

الـ abstraction الأساسية اللي بيقدمها الـ machine.h هي **`struct pinctrl_map`** — وهي الجسر بين الـ board configuration والـ pinctrl core.

#### تشريح الـ struct pinctrl_map

```c
struct pinctrl_map {
    const char *dev_name;       /* اسم الـ device اللي محتاج التهيئة دي */
    const char *name;           /* اسم الـ state: "default", "sleep", ... */
    enum pinctrl_map_type type; /* نوع الإدخال: MUX أو CONFIG */
    const char *ctrl_dev_name;  /* اسم الـ pin controller المسؤول */
    union {
        struct pinctrl_map_mux mux;         /* لو نوعه MUX_GROUP */
        struct pinctrl_map_configs configs;  /* لو نوعه CONFIGS_* */
    } data;
};
```

الـ `union` هنا ذكي: ممكن الـ entry بتحدد مux (توجيه الـ pin لـ function معينة)، أو configs (ضبط خصائص كالـ pull-up/down والـ drive strength).

#### أنواع الـ map entries

```
PIN_MAP_TYPE_INVALID       → مش مستخدم، للحماية من zero-init
PIN_MAP_TYPE_DUMMY_STATE   → state موجود بس مش محتاج أي pin changes
PIN_MAP_TYPE_MUX_GROUP     → حوّل group من الـ pins لـ function معينة
PIN_MAP_TYPE_CONFIGS_PIN   → اضبط config على pin واحد
PIN_MAP_TYPE_CONFIGS_GROUP → اضبط config على group كاملة
```

---

### العلاقة بين الـ Structs

```
struct pinctrl_map[]
    │
    ├── .dev_name ──────────────────────► "spi0.0"
    ├── .name ──────────────────────────► "default"
    ├── .type ──────────────────────────► PIN_MAP_TYPE_MUX_GROUP
    ├── .ctrl_dev_name ─────────────────► "pinctrl0"  (الـ controller)
    └── .data.mux
            ├── .group ─────────────────► "spi0_grp"
            └── .function ──────────────► "spi0"
                                              │
                              ┌───────────────▼────────────────┐
                              │   struct pinfunction           │
                              │   .name = "spi0"               │
                              │   .groups[] = {"spi0_grp"}     │
                              └───────────────┬────────────────┘
                                              │
                              ┌───────────────▼────────────────┐
                              │   struct pingroup              │
                              │   .name = "spi0_grp"           │
                              │   .pins[] = {10, 11, 12, 13}   │
                              └───────────────┬────────────────┘
                                              │
                              ┌───────────────▼────────────────┐
                              │ struct pinctrl_pin_desc[]      │
                              │ pin 10: "SPI0_MOSI"            │
                              │ pin 11: "SPI0_MISO"            │
                              │ pin 12: "SPI0_CLK"             │
                              │ pin 13: "SPI0_CS0"             │
                              └────────────────────────────────┘
```

---

### الـ States وإزاي بيشتغلوا

الـ `pinctrl-state.h` بيعرّف 4 states أساسية:

```c
#define PINCTRL_STATE_DEFAULT "default"  /* الـ normal operation */
#define PINCTRL_STATE_INIT    "init"     /* قبل probe() لو محتاج */
#define PINCTRL_STATE_IDLE    "idle"     /* runtime suspend */
#define PINCTRL_STATE_SLEEP   "sleep"    /* system suspend */
```

**الـ flow عملياً:**

```
device probe()
    │
    ├─► pinctrl core يدور على map entries بـ dev_name
    ├─► يلاقي "default" state
    ├─► يطلب من الـ pin controller يعمل mux للـ group
    ├─► يطلب منه يطبق الـ configs
    └─► الـ pins جاهزة للـ driver

pm_runtime_suspend()
    │
    └─► pinctrl_pm_select_idle_state()
            └─► نفس الـ process بس بـ "idle" state
                (ممكن الـ pins تتحول لـ GPIO input بدون pull)

system_suspend()
    │
    └─► pinctrl_pm_select_sleep_state()
            └─► "sleep" state
                (ممكن الـ pins تتعمل pull-down عشان توفر power)
```

---

### الـ HOG Mechanism

لما `ctrl_dev_name == dev_name` في الـ map entry، ده معناه إن الـ pin controller نفسه بيعمل "hog" لهذه الـ pins — بيطلبها لنفسه وقت التسجيل. ده بيُستخدم للـ pins اللي محتاجة تتهيأ من أول لحظة (مثلاً enable pin لـ power regulator).

```c
/* Pin controller يعمل hog للـ group لنفسه */
#define PIN_MAP_MUX_GROUP_HOG_DEFAULT(dev, grp, func) \
    PIN_MAP_MUX_GROUP(dev, PINCTRL_STATE_DEFAULT, dev, grp, func)
//                                               ^^^
//                                    ctrl_dev_name = dev_name → hog!
```

---

### الـ Macros وإزاي بتوفّر Boilerplate

بدل ما تكتب الـ struct initialization يدوياً في كل مرة:

```c
/* بدون macro — مزعج ومعرض للـ typos */
static const struct pinctrl_map uart_maps[] = {
    {
        .dev_name      = "uart0",
        .name          = "default",
        .type          = PIN_MAP_TYPE_MUX_GROUP,
        .ctrl_dev_name = "pinctrl0",
        .data.mux = {
            .group    = "uart0grp",
            .function = "uart0",
        },
    },
};

/* بالـ macro — نظيف وواضح */
static const struct pinctrl_map uart_maps[] = {
    PIN_MAP_MUX_GROUP_DEFAULT("uart0", "pinctrl0", "uart0grp", "uart0"),
};
```

الـ `PIN_MAP_CONFIGS_PIN` macro بيستخدم `ARRAY_SIZE(cfgs)` أوتوماتيك عشان يحسب عدد الـ configs ويحطها في `num_configs`.

---

### مثال كامل من الواقع: STM32 UART

```c
/* board file أو platform data */

/* config values: pull-up enabled, drive strength = 2mA */
static unsigned long uart1_pin_cfg[] = {
    PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 1),
    PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 2),
};

static const struct pinctrl_map stm32_board_maps[] = {
    /* default state: enable UART1 mux */
    PIN_MAP_MUX_GROUP_DEFAULT(
        "48000000.serial",   /* dev_name: UART1 device */
        "soc:pinctrl@50020000", /* ctrl_dev_name: STM32 pinctrl */
        "uart1_pins_a",      /* group name */
        "uart1"              /* function name */
    ),
    /* default state: configure TX pin */
    PIN_MAP_CONFIGS_PIN_DEFAULT(
        "48000000.serial",
        "soc:pinctrl@50020000",
        "PA9",               /* pin name */
        uart1_pin_cfg        /* array of configs */
    ),
    /* sleep state: disable the pins */
    PIN_MAP_MUX_GROUP(
        "48000000.serial",
        "sleep",
        "soc:pinctrl@50020000",
        "uart1_sleep_pins",
        "gpio"               /* revert to GPIO in sleep */
    ),
};

/* في board init أو driver */
pinctrl_register_mappings(stm32_board_maps, ARRAY_SIZE(stm32_board_maps));
```

---

### مين بيمتلك إيه؟

| المسؤولية | الـ Pinctrl Core | الـ Pin Controller Driver |
|---|---|---|
| تسجيل الـ pins وأسمائها | يحتفظ بالـ list | يوفر `pinctrl_pin_desc[]` |
| تعريف الـ groups | يحتفظ بالـ list | يوفر `pinctrl_ops.get_group_pins()` |
| تعريف الـ functions | يحتفظ بالـ list | يوفر `pinmux_ops.get_function_name()` |
| الـ mapping table | يملكها ويحلها | لا علاقة له بيها |
| conflict detection | نعم، هو يمنع | لا يعرف |
| كتابة الـ registers | لا | نعم، هو وبس |
| PM state transitions | يطلب الـ state | ينفذ الـ state فعلياً |
| DT parsing | يستدعي الـ driver | ينفذ `dt_node_to_map()` |

---

### الـ Consumer API Flow

```
Driver                   Pinctrl Core              Pin Controller HW
  │                           │                           │
  │── devm_pinctrl_get() ────►│                           │
  │                           │ ابحث في mapping table     │
  │◄── struct pinctrl* ───────│                           │
  │                           │                           │
  │── pinctrl_lookup_state() ►│                           │
  │   ("default")             │ ابحث في states list       │
  │◄── struct pinctrl_state* ─│                           │
  │                           │                           │
  │── pinctrl_select_state() ►│                           │
  │                           │── pmxops->set_mux() ─────►│
  │                           │── confops->pin_config() ──►│
  │                           │                           │ write registers
  │◄── 0 (success) ───────────│◄── 0 ─────────────────────│
```

---

### ملخص: الـ Pinctrl هو الـ Hardware Resource Manager للـ Pins

الـ pinctrl subsystem بيقدم:
- **Ownership model**: pin واحد = owner واحد في أي وقت
- **State machine**: كل device ليه states معروفة وانتقالات محددة
- **Separation of concerns**: الـ board config منفصلة عن الـ driver logic
- **الـ machine.h** تحديداً بيمثل الـ "عقد" بين الـ board designer والـ kernel: بيحدد بالاسم مين يحتاج إيه، في أي state، من أي controller
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Config Options والـ State Strings — Cheatsheet

#### `enum pinctrl_map_type` — أنواع الـ mapping entries

| القيمة | المعنى | متى تُستخدم |
|--------|--------|-------------|
| `PIN_MAP_TYPE_INVALID` | entry فاسد/غير مهيأ | قيمة افتراضية — لا تُستخدم مباشرة |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمي بدون أي إعداد فعلي | لما device بيطلب state بس مش محتاج pin config حقيقي |
| `PIN_MAP_TYPE_MUX_GROUP` | تغيير الـ mux function لمجموعة pins | إعداد الـ UART أو I2C على pins معينة |
| `PIN_MAP_TYPE_CONFIGS_PIN` | ضبط إعدادات (pull/drive) على pin واحد | تفعيل pull-up على pin بعينه |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | ضبط إعدادات على مجموعة pins | ضبط drive strength لكل pins الـ SD card |

#### Standard State Names — من `pinctrl-state.h`

| الـ Macro | القيمة النصية | الاستخدام |
|-----------|--------------|-----------|
| `PINCTRL_STATE_DEFAULT` | `"default"` | الحالة الطبيعية أثناء الشغل — يتطبق تلقائياً قبل `probe()` |
| `PINCTRL_STATE_INIT` | `"init"` | قبل `probe()` لما الـ default ممكن يعمل glitch |
| `PINCTRL_STATE_IDLE` | `"idle"` | نظام مرتاح بس صاحي — `pm_runtime_suspend()` |
| `PINCTRL_STATE_SLEEP` | `"sleep"` | أعمق حالة نوم — `suspend()` |

#### `CONFIG_PINCTRL` — الـ Kconfig option

| الحالة | السلوك |
|--------|--------|
| مفعّل | كل الـ functions حقيقية ومُسجَّلة في الـ core |
| معطَّل | كل الـ functions بتبقى `static inline` وترجع 0 أو void — zero overhead |

---

### 1. الـ Structs المهمة

---

#### `struct pinctrl_map_mux`

**الغرض:** بتحمل اسم الـ group واسم الـ function اللي هيتربط بيه — ده الـ payload لأي entry من نوع `PIN_MAP_TYPE_MUX_GROUP`.

```c
struct pinctrl_map_mux {
    const char *group;     // اسم مجموعة الـ pins (ممكن NULL → أول group مناسب)
    const char *function;  // اسم الـ mux function (مثلاً "uart0", "i2c1")
};
```

**العلاقات:**
- محتواه موجود جوه `union data` في `struct pinctrl_map`
- الـ pin controller بيستخدم `function` عشان يعمل lookup في جدول الـ functions الخاص بيه
- لو `group == NULL` الـ core بياخد أول group بيدعم الـ function دي

---

#### `struct pinctrl_map_configs`

**الغرض:** بتحمل مصفوفة الإعدادات (pull-up/down، drive strength، إلخ) اللي هتتطبق على pin أو group.

```c
struct pinctrl_map_configs {
    const char *group_or_pin;   // اسم الـ pin أو الـ group
    unsigned long *configs;     // array من config values (driver-specific encoding)
    unsigned int num_configs;   // عدد الـ entries في الـ array
};
```

**العلاقات:**
- موجود جوه `union data` في `struct pinctrl_map` لما النوع `CONFIGS_PIN` أو `CONFIGS_GROUP`
- الـ `configs` array بتكون static في الغالب في board file
- كل controller بيفسر `unsigned long` بالطريقة بتاعته — مفيش تنسيق موحد

---

#### `struct pinctrl_map`

**الغرض:** ده الـ struct الأساسي في الملف. كل entry في جدول الـ mapping هو `pinctrl_map` واحد. البورد بتبني array منهم وتسجلهم في الـ core.

```c
struct pinctrl_map {
    const char *dev_name;       // اسم الـ device اللي بيستخدم الـ mapping دي
    const char *name;           // اسم الـ state (مثلاً "default", "sleep")
    enum pinctrl_map_type type; // نوع الـ entry
    const char *ctrl_dev_name;  // اسم الـ pin controller المسؤول (NULL في DUMMY)
    union {
        struct pinctrl_map_mux mux;        // لما type == MUX_GROUP
        struct pinctrl_map_configs configs; // لما type == CONFIGS_*
    } data;
};
```

**العلاقات:**
- البورد بتعمل static array من `pinctrl_map` وتبعتها لـ `pinctrl_register_mappings()`
- الـ core بيخزنهم ويعمل lookup بيهم لما device بيطلب state معين
- لو `dev_name == ctrl_dev_name` → الـ entry ده **hog**: الـ controller نفسه بياخده عند التسجيل

---

### 2. رسم العلاقات بين الـ Structs

```
Board/Machine Code
        │
        │  static const struct pinctrl_map my_maps[] = { ... }
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│                   struct pinctrl_map                    │
│                                                         │
│  dev_name  ──────────────────────► "spi0.0"            │
│  name      ──────────────────────► "default"           │
│  type      ──────────────────────► PIN_MAP_TYPE_MUX_GROUP│
│  ctrl_dev_name ──────────────────► "pinctrl.0"         │
│                                                         │
│  union data                                             │
│  ┌──────────────────────┐  ┌──────────────────────────┐│
│  │ struct pinctrl_map_mux│  │struct pinctrl_map_configs ││
│  │  group   → "spi-grp" │  │  group_or_pin → "PA0"    ││
│  │  function→ "spi0"    │  │  configs      → [array]  ││
│  │                      │  │  num_configs  → 3        ││
│  └──────────────────────┘  └──────────────────────────┘│
│   (يُستخدم لو MUX_GROUP)    (يُستخدم لو CONFIGS_*)      │
└─────────────────────────────────────────────────────────┘
        │
        │ pinctrl_register_mappings()
        ▼
┌─────────────────────────┐
│   pinctrl core (kernel) │
│  (pinctrl_maps list)    │
│                         │
│  lookup by dev_name     │
│       + state name      │
└─────────────────────────┘
        │
        ▼ عند طلب state
┌─────────────────────────┐
│   struct pinctrl_dev    │  ← الـ controller نفسه
│   (pin controller drv)  │
│                         │
│  .pinmux_ops            │  ← بيطبق الـ MUX
│  .pinconf_ops           │  ← بيطبق الـ CONFIGS
└─────────────────────────┘
```

---

### 3. دورة حياة الـ Mapping — Lifecycle Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      Board Init / Driver Init               │
│                                                             │
│  1. CREATION                                                │
│     ──────────────────────────────────────────────────────  │
│     Board file بتعرّف static array من pinctrl_map[]        │
│     باستخدام الـ macros (PIN_MAP_MUX_GROUP_DEFAULT, إلخ)   │
│                          │                                  │
│                          ▼                                  │
│  2. REGISTRATION                                            │
│     ──────────────────────────────────────────────────────  │
│     pinctrl_register_mappings(map, num_maps)                │
│          أو                                                 │
│     devm_pinctrl_register_mappings(dev, map, num_maps)      │
│                                                             │
│     الـ core بيضيف الـ maps لـ global list                 │
│     لو HOG entry → الـ controller يطبقه فوراً              │
│                          │                                  │
│                          ▼                                  │
│  3. USAGE                                                   │
│     ──────────────────────────────────────────────────────  │
│     Device driver:                                          │
│       pinctrl_get(dev)          → pinctrl handle            │
│       pinctrl_lookup_state()    → state handle              │
│       pinctrl_select_state()    → تطبيق الـ map entries     │
│                                                             │
│     Core بيبحث في الـ maps عن entries تطابق:               │
│       (dev_name == device, name == state_name)              │
│                          │                                  │
│                          ▼                                  │
│  4. TEARDOWN                                                │
│     ──────────────────────────────────────────────────────  │
│     pinctrl_unregister_mappings(map)                        │
│          أو تلقائياً مع devm عند remove الـ device         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 4. Call Flow Diagrams

#### 4.1 تسجيل الـ Maps وتطبيق HOG

```
board_init() / driver_probe()
        │
        ├─► pinctrl_register_mappings(maps, n)
        │         │
        │         ├─ loop على كل entry
        │         ├─ validate type
        │         ├─ أضف لـ global pinctrl_maps list
        │         └─ لو dev_name == ctrl_dev_name (HOG)
        │                   │
        │                   └─► create_pinctrl(ctrl_dev)
        │                             │
        │                             └─► pinctrl_select_state("default")
        │                                       │
        │                                       ├─► pinmux_enable_setting()
        │                                       └─► pinconf_apply_setting()
        │
        └─ return 0
```

#### 4.2 Device يطلب State

```
device_driver_probe(dev)
        │
        ├─► devm_pinctrl_get(dev)
        │         │
        │         └─► pinctrl_get()
        │                   │
        │                   └─ ابحث في maps عن dev_name
        │                      ابني قائمة settings
        │
        ├─► pinctrl_lookup_state(p, "default")
        │         │
        │         └─ ابحث في settings عن name=="default"
        │            رجّع struct pinctrl_state*
        │
        └─► pinctrl_select_state(p, state)
                  │
                  ├─ لكل setting في الـ state:
                  │
                  ├─► [MUX_GROUP]
                  │     └─► pinmux_ops->set_mux(group, function)
                  │
                  └─► [CONFIGS_PIN / CONFIGS_GROUP]
                        └─► pinconf_ops->pin_config_set(pin, configs, n)
```

#### 4.3 الـ `devm_` vs Manual Registration

```
devm_pinctrl_register_mappings(dev, map, n)
        │
        ├─► pinctrl_register_mappings(map, n)   ← نفس التسجيل
        │
        └─► devm_add_action(dev,                ← ربط cleanup بـ device lifetime
                pinctrl_unregister_mappings,
                map)
                    │
                    └─ عند device remove → pinctrl_unregister_mappings(map) تلقائياً
```

---

### 5. الـ Macros — جدول مرجعي سريع

| الـ Macro | النوع | ملاحظة |
|-----------|-------|--------|
| `PIN_MAP_DUMMY_STATE(dev, state)` | `DUMMY_STATE` | بدون ctrl_dev_name وبدون data |
| `PIN_MAP_MUX_GROUP(dev, state, pinctrl, grp, func)` | `MUX_GROUP` | كامل الإعداد |
| `PIN_MAP_MUX_GROUP_DEFAULT(dev, pinctrl, grp, func)` | `MUX_GROUP` | state = `"default"` تلقائياً |
| `PIN_MAP_MUX_GROUP_HOG(dev, state, grp, func)` | `MUX_GROUP` | `ctrl_dev_name = dev` → HOG |
| `PIN_MAP_MUX_GROUP_HOG_DEFAULT(dev, grp, func)` | `MUX_GROUP` | HOG + default state |
| `PIN_MAP_CONFIGS_PIN(dev, state, pinctrl, pin, cfgs)` | `CONFIGS_PIN` | `ARRAY_SIZE(cfgs)` تلقائياً |
| `PIN_MAP_CONFIGS_PIN_DEFAULT(...)` | `CONFIGS_PIN` | state = `"default"` |
| `PIN_MAP_CONFIGS_PIN_HOG(...)` | `CONFIGS_PIN` | HOG |
| `PIN_MAP_CONFIGS_PIN_HOG_DEFAULT(...)` | `CONFIGS_PIN` | HOG + default |
| `PIN_MAP_CONFIGS_GROUP(dev, state, pinctrl, grp, cfgs)` | `CONFIGS_GROUP` | على مجموعة |
| `PIN_MAP_CONFIGS_GROUP_DEFAULT(...)` | `CONFIGS_GROUP` | state = `"default"` |
| `PIN_MAP_CONFIGS_GROUP_HOG(...)` | `CONFIGS_GROUP` | HOG |
| `PIN_MAP_CONFIGS_GROUP_HOG_DEFAULT(...)` | `CONFIGS_GROUP` | HOG + default |

---

### 6. الـ Locking Strategy

الملف `machine.h` نفسه ما بيعرفش locks — ده header للـ data structures بس. الـ locking بيحصل في الـ core (`drivers/pinctrl/core.c`).

#### من بيحمي إيه؟

```
┌────────────────────────────────────────────────────────────┐
│  Global: pinctrl_maps list                                 │
│                                                            │
│  Lock: pinctrl_maps_mutex  (mutex)                         │
│  ──────────────────────────────────────────────────────    │
│  بيحمي: قائمة كل الـ map tables المسجلة                   │
│  يُقفل في:                                                 │
│    - pinctrl_register_mappings()     → write               │
│    - pinctrl_unregister_mappings()   → write               │
│    - pinctrl_get() / lookup          → read                │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  Per-device: struct pinctrl                                │
│                                                            │
│  Lock: pinctrl_dev->mutex  (mutex)                         │
│  ──────────────────────────────────────────────────────    │
│  بيحمي: قائمة states وإعدادات كل device                   │
│  يُقفل في:                                                 │
│    - pinctrl_select_state()                                │
│    - pinctrl_lookup_state()                                │
└────────────────────────────────────────────────────────────┘
```

#### ترتيب الـ Lock Acquisition (Lock Ordering)

```
pinctrl_maps_mutex
        │
        └─► (يُحرر قبل)
                │
                ▼
        pinctrl_dev->mutex
```

**القاعدة:** `pinctrl_maps_mutex` دايماً يتاخد قبل أي per-device mutex — عشان منعدمش deadlock.

الـ `struct pinctrl_map` نفسها **read-only بعد التسجيل** — الـ board بتعملها static const، مفيش حاجة بتكتب فيها بعد كده، فالـ locking على قراءتها minimal.
## Phase 4: شرح الـ Functions

### ملخص سريع — Cheatsheet

#### الـ Enums والـ Structs الأساسية

| الاسم | النوع | الغرض |
|---|---|---|
| `pinctrl_map_type` | enum | نوع الـ mapping entry |
| `pinctrl_map_mux` | struct | بيانات الـ MUX لـ group معينة |
| `pinctrl_map_configs` | struct | بيانات الـ configuration لـ pin أو group |
| `pinctrl_map` | struct | الـ mapping entry الكاملة للـ board/machine |

#### الـ Functions

| الـ Function | الغرض | الـ Context |
|---|---|---|
| `pinctrl_register_mappings()` | تسجيل static mapping table | init-time / board code |
| `devm_pinctrl_register_mappings()` | نفس الأول لكن managed | driver probe |
| `pinctrl_unregister_mappings()` | إلغاء تسجيل mapping table | cleanup |
| `pinctrl_provide_dummies()` | تفعيل dummy mode لو مفيش controller | board init |

#### الـ Macros — Cheatsheet

| الـ Macro | المعنى |
|---|---|
| `PIN_MAP_DUMMY_STATE` | entry وهمية بدون controller |
| `PIN_MAP_MUX_GROUP` | ربط group بـ function معين |
| `PIN_MAP_MUX_GROUP_DEFAULT` | نفس السابق بـ state = "default" |
| `PIN_MAP_MUX_GROUP_HOG` | الـ controller نفسه يعمل hog للـ mapping |
| `PIN_MAP_MUX_GROUP_HOG_DEFAULT` | hog + default state |
| `PIN_MAP_CONFIGS_PIN` | config لـ pin واحد |
| `PIN_MAP_CONFIGS_PIN_DEFAULT` | config لـ pin بـ default state |
| `PIN_MAP_CONFIGS_PIN_HOG` | config + hog لـ pin |
| `PIN_MAP_CONFIGS_PIN_HOG_DEFAULT` | config + hog + default لـ pin |
| `PIN_MAP_CONFIGS_GROUP` | config لـ group |
| `PIN_MAP_CONFIGS_GROUP_DEFAULT` | config لـ group بـ default state |
| `PIN_MAP_CONFIGS_GROUP_HOG` | config + hog لـ group |
| `PIN_MAP_CONFIGS_GROUP_HOG_DEFAULT` | config + hog + default لـ group |

---

### تصنيف الـ Functions

```
┌─────────────────────────────────────────┐
│           pinctrl machine API           │
├───────────────┬─────────────────────────┤
│ Registration  │ pinctrl_register_mappings│
│               │ devm_pinctrl_register_  │
│               │ mappings                │
├───────────────┼─────────────────────────┤
│ Cleanup       │ pinctrl_unregister_     │
│               │ mappings                │
├───────────────┼─────────────────────────┤
│ Helpers       │ pinctrl_provide_dummies │
├───────────────┼─────────────────────────┤
│ Macros        │ PIN_MAP_*               │
└───────────────┴─────────────────────────┘
```

---

### الـ Data Types

#### `enum pinctrl_map_type`

```c
enum pinctrl_map_type {
    PIN_MAP_TYPE_INVALID,       /* invalid / uninitialized */
    PIN_MAP_TYPE_DUMMY_STATE,   /* placeholder, no real controller */
    PIN_MAP_TYPE_MUX_GROUP,     /* mux a pin group to a function */
    PIN_MAP_TYPE_CONFIGS_PIN,   /* configure a single pin */
    PIN_MAP_TYPE_CONFIGS_GROUP, /* configure a pin group */
};
```

الـ `pinctrl_map_type` بيحدد إيه اللي الـ mapping entry دي عايزة تعمله — مجرد state وهمية، أو تحديد الـ mux، أو ضبط الـ config لـ pin أو group. الـ core بيستخدم النوع ده عشان يعرف يفسر الـ `data` union جوه `pinctrl_map`.

---

#### `struct pinctrl_map_mux`

```c
struct pinctrl_map_mux {
    const char *group;    /* pin group name, NULL = first applicable */
    const char *function; /* mux function to select */
};
```

الـ `group` ممكن يكون NULL وفي الحالة دي الـ pinctrl core بياخد أول group مناسبة للـ function. الـ `function` لازم يكون موجود في الـ pinmux_ops الخاصة بالـ controller.

---

#### `struct pinctrl_map_configs`

```c
struct pinctrl_map_configs {
    const char *group_or_pin; /* target pin or group name */
    unsigned long *configs;   /* array of config param/value pairs */
    unsigned int num_configs; /* number of entries in configs array */
};
```

الـ `configs` هي array من `unsigned long` والـ format بتاعها بتختلف من controller لـ controller — كل driver بيعرف الـ bits بنفسه. الـ `num_configs` بيتحسب أوتوماتيك لو استخدمت الـ macros باستخدام `ARRAY_SIZE()`.

---

#### `struct pinctrl_map`

```c
struct pinctrl_map {
    const char *dev_name;      /* consumer device name */
    const char *name;          /* state name e.g. "default", "sleep" */
    enum pinctrl_map_type type;
    const char *ctrl_dev_name; /* pinctrl controller device name */
    union {
        struct pinctrl_map_mux mux;
        struct pinctrl_map_configs configs;
    } data;
};
```

ده الـ building block بتاع الـ machine-level pin configuration. كل entry بتربط device معين بـ state معينة بـ controller معين. لو `dev_name == ctrl_dev_name` يبقى ده **hog** — يعني الـ controller نفسه بيحجز الـ mapping لنفسه أثناء الـ registration.

---

### فئة أولى — Registration Functions

#### `pinctrl_register_mappings()`

```c
int pinctrl_register_mappings(const struct pinctrl_map *map,
                               unsigned int num_maps);
```

بتسجل array من الـ `pinctrl_map` entries في الـ global mapping list بتاع الـ pinctrl core. الـ core بيعمل copy من الـ pointer مش من البيانات نفسها، فالـ array لازم تفضل valid طول عمر الـ kernel أو لحد ما تعمل unregister.

**الـ Parameters:**

| الـ Parameter | الوصف |
|---|---|
| `map` | pointer لـ array من `pinctrl_map` (static عادةً) |
| `num_maps` | عدد الـ entries في الـ array |

**الـ Return:**
- `0` نجاح
- سالب لو في error (مثلاً invalid entries أو memory allocation failure)

**تفاصيل مهمة:**
- بتعمل iterate على كل الـ entries وبتتحقق من صحتها
- الـ function مش thread-safe أثناء الـ boot لكن في الـ production يستخدم lock داخلي
- لو الـ controller بتاع أي entry متسجل بالفعل، الـ core بيعمل **hogging** للـ entries المناسبة فوراً
- المكان المناسب لاستدعاؤها: `arch_initcall()` أو `board_init()` أو `postcore_initcall()`

**مثال:**

```c
static const struct pinctrl_map my_board_map[] = {
    PIN_MAP_MUX_GROUP_DEFAULT("serial0", "pinctrl0", "uart0grp", "uart0"),
    PIN_MAP_CONFIGS_PIN_DEFAULT("serial0", "pinctrl0", "PA0", uart_pin_cfg),
};

static int __init my_board_init(void)
{
    return pinctrl_register_mappings(my_board_map,
                                     ARRAY_SIZE(my_board_map));
}
arch_initcall(my_board_init);
```

---

#### `devm_pinctrl_register_mappings()`

```c
int devm_pinctrl_register_mappings(struct device *dev,
                                    const struct pinctrl_map *map,
                                    unsigned int num_maps);
```

نفس `pinctrl_register_mappings()` بالظبط لكن مع **devres** — يعني لما الـ device يتنزل (unbind/remove) الـ mappings بتتشال أوتوماتيك. مناسبة جداً للـ drivers اللي بتسجل mappings من `probe()`.

**الـ Parameters:**

| الـ Parameter | الوصف |
|---|---|
| `dev` | الـ device المسؤول عن الـ mappings (للـ devres) |
| `map` | نفس بتاع `pinctrl_register_mappings` |
| `num_maps` | نفس بتاع `pinctrl_register_mappings` |

**الـ Return:**
- `0` نجاح
- سالب لو في error

**تفاصيل مهمة:**
- بتعمل `devm_add_action()` داخلياً مربوطة بالـ `dev`
- لما `dev` يتنزل، الـ cleanup بتستدعي `pinctrl_unregister_mappings()` أوتوماتيك
- مناسبة أكثر من `pinctrl_register_mappings()` في الـ platform drivers الحديثة

**مثال:**

```c
static int my_pinctrl_probe(struct platform_device *pdev)
{
    return devm_pinctrl_register_mappings(&pdev->dev,
                                          board_maps,
                                          ARRAY_SIZE(board_maps));
}
```

---

### فئة تانية — Cleanup Functions

#### `pinctrl_unregister_mappings()`

```c
void pinctrl_unregister_mappings(const struct pinctrl_map *map);
```

بتشيل الـ mapping table من الـ global list. بتستخدم الـ pointer نفسه كـ key عشان تلاقي الـ entry وتشيلها.

**الـ Parameters:**

| الـ Parameter | الوصف |
|---|---|
| `map` | نفس الـ pointer اللي اتبعت لـ register (مش copy) |

**الـ Return:** void

**تفاصيل مهمة:**
- لازم تبعت نفس الـ `map` pointer اللي استخدمته في الـ register
- مش بتعمل free للـ array نفسها — ده مسؤولية الـ caller
- لو في device بيستخدم الـ mappings دي وقت الـ unregister، السلوك غير محدد — لازم تتأكد إن مفيش devices لسه بتستخدمها
- بتستخدم list lock داخلي

---

### فئة تالتة — Helper Functions

#### `pinctrl_provide_dummies()`

```c
void pinctrl_provide_dummies(void);
```

بتخلي الـ pinctrl core يقبل الـ `PIN_MAP_TYPE_DUMMY_STATE` entries كـ valid states بدون ما يحتاج controller حقيقي. لازم تتستدعى من الـ board code على الـ platforms اللي مش عندها pinctrl controller فعلي بس محتاجة الـ device drivers تشتغل بدون errors.

**الـ Parameters:** لا يوجد

**الـ Return:** void

**تفاصيل مهمة:**
- بتضرب flag global (`pinctrl_dummy_state`) في الـ core
- بدونها، لو device طلب state موجودة بس نوعها `DUMMY_STATE`، الـ core بيرجع error
- مفيدة في بيئات الـ simulation أو الـ platforms اللي بدأت تنقل من hardcoded pin setup لـ pinctrl proper

**مثال:**

```c
static void __init my_platform_init(void)
{
    /* no real pinctrl controller on this board variant */
    pinctrl_provide_dummies();
    pinctrl_register_mappings(maps, ARRAY_SIZE(maps));
}
```

---

### فئة رابعة — Convenience Macros

الـ macros دي بتختصر إنشاء الـ `pinctrl_map` entries وبتمنع الأخطاء اليدوية. كلها بتنتج static initializer مناسب لـ array initialization.

---

#### `PIN_MAP_DUMMY_STATE`

```c
#define PIN_MAP_DUMMY_STATE(dev, state)
```

بتنشئ entry من نوع `PIN_MAP_TYPE_DUMMY_STATE` — بدون controller وبدون data. الـ pinctrl core بيعترف بيها كـ state صالحة لو اتستدعت `pinctrl_provide_dummies()`.

| الـ Parameter | الوصف |
|---|---|
| `dev` | اسم الـ consumer device |
| `state` | اسم الـ state |

**مثال:**

```c
PIN_MAP_DUMMY_STATE("spi0", "default")
```

---

#### `PIN_MAP_MUX_GROUP`

```c
#define PIN_MAP_MUX_GROUP(dev, state, pinctrl, grp, func)
```

بتنشئ entry لربط pin group بـ mux function. دي الـ macro الأم للـ mux entries — باقي الـ variants بتبني عليها.

| الـ Parameter | الوصف |
|---|---|
| `dev` | consumer device name |
| `state` | pin state name |
| `pinctrl` | pinctrl controller device name |
| `grp` | pin group name (ممكن NULL) |
| `func` | mux function name |

---

#### `PIN_MAP_MUX_GROUP_DEFAULT`

```c
#define PIN_MAP_MUX_GROUP_DEFAULT(dev, pinctrl, grp, func)
    PIN_MAP_MUX_GROUP(dev, PINCTRL_STATE_DEFAULT, pinctrl, grp, func)
```

نفس `PIN_MAP_MUX_GROUP` لكن بيحط `PINCTRL_STATE_DEFAULT` = `"default"` كـ state أوتوماتيك. ده الأكثر استخداماً في board files.

---

#### `PIN_MAP_MUX_GROUP_HOG` و `PIN_MAP_MUX_GROUP_HOG_DEFAULT`

```c
#define PIN_MAP_MUX_GROUP_HOG(dev, state, grp, func)
    PIN_MAP_MUX_GROUP(dev, state, dev, grp, func)

#define PIN_MAP_MUX_GROUP_HOG_DEFAULT(dev, grp, func)
    PIN_MAP_MUX_GROUP(dev, PINCTRL_STATE_DEFAULT, dev, grp, func)
```

الـ **hog** معناه إن الـ `ctrl_dev_name == dev_name` — يعني الـ pinctrl controller بيحجز الـ mapping لنفسه. الـ core لما بيشوف ده أثناء الـ registration بيطبق الـ mapping فوراً بدون انتظار أي consumer.

**متى تستخدمه:** لما الـ pins مش بتاعة device تاني — مثلاً pins الـ controller نفسه (debug LEDs، oscillator pins، إلخ).

---

#### `PIN_MAP_CONFIGS_PIN`

```c
#define PIN_MAP_CONFIGS_PIN(dev, state, pinctrl, pin, cfgs)
```

بتنشئ config entry لـ pin واحد. الـ `cfgs` هي array من `unsigned long`، والـ macro بتستخدم `ARRAY_SIZE(cfgs)` أوتوماتيك فلازم تبعت array مش pointer.

| الـ Parameter | الوصف |
|---|---|
| `dev` | consumer device name |
| `state` | state name |
| `pinctrl` | controller name |
| `pin` | pin name |
| `cfgs` | array of config values (لازم تكون local array) |

**مثال:**

```c
static unsigned long uart_pin_configs[] = {
    PIN_CONFIG_DRIVE_STRENGTH | (4 << 8),
    PIN_CONFIG_BIAS_PULL_UP,
};

PIN_MAP_CONFIGS_PIN_DEFAULT("serial0", "pinctrl0", "PA0", uart_pin_configs)
```

---

#### `PIN_MAP_CONFIGS_PIN_DEFAULT` / `PIN_MAP_CONFIGS_PIN_HOG` / `PIN_MAP_CONFIGS_PIN_HOG_DEFAULT`

Convenience wrappers فوق `PIN_MAP_CONFIGS_PIN` — بتحط default state أو hog أو الاتنين.

| الـ Macro | الـ state | الـ ctrl |
|---|---|---|
| `PIN_MAP_CONFIGS_PIN_DEFAULT` | `"default"` | منفصل |
| `PIN_MAP_CONFIGS_PIN_HOG` | محدد | = `dev` |
| `PIN_MAP_CONFIGS_PIN_HOG_DEFAULT` | `"default"` | = `dev` |

---

#### `PIN_MAP_CONFIGS_GROUP` وعائلتها

```c
#define PIN_MAP_CONFIGS_GROUP(dev, state, pinctrl, grp, cfgs)
#define PIN_MAP_CONFIGS_GROUP_DEFAULT(dev, pinctrl, grp, cfgs)
#define PIN_MAP_CONFIGS_GROUP_HOG(dev, state, grp, cfgs)
#define PIN_MAP_CONFIGS_GROUP_HOG_DEFAULT(dev, grp, cfgs)
```

نفس منطق `PIN_MAP_CONFIGS_PIN` لكن بتطبق الـ config على **group** من الـ pins مرة واحدة. مفيد لما يكون عندك bus كامل (I2C, SPI, إلخ) محتاج نفس الـ config.

---

### الـ Pseudocode Flow — Registration

```
pinctrl_register_mappings(map, num_maps)
│
├─ validate each entry in map[]
│   ├─ type must not be INVALID
│   ├─ dev_name must not be NULL
│   └─ ctrl_dev_name required unless DUMMY_STATE
│
├─ allocate mapping_list node
│
├─ add to global maps list (with lock)
│
└─ for each registered pinctrl controller:
    └─ try_hog_maps()
        └─ if dev_name == ctrl_dev_name → apply mapping immediately
```

---

### علاقة الـ Machine Layer بباقي الـ Subsystem

```
Board/Platform Code
      │
      │  pinctrl_register_mappings()
      ▼
  Global Map List  ◄──── devm_pinctrl_register_mappings()
      │
      │  (lookup by dev_name + state_name)
      ▼
  pinctrl_get() / pinctrl_lookup_state() / pinctrl_select_state()
      │
      ▼
  pinctrl_ops → pinmux_ops → pinconf_ops
  (implemented by the controller driver)
```

الـ machine.h بتعرف الـ "what" — أي device، أي state، أي function. الـ controller driver بتعرف الـ "how" — إزاي تضبط الـ hardware registers.

---

### ملاحظات على الـ `CONFIG_PINCTRL` Guards

لو الـ kernel اتبنى بدون `CONFIG_PINCTRL`، كل الـ functions بتتحول لـ `static inline` stubs بترجع 0 أو void. ده بيخلي الـ board code والـ drivers يشتغلوا بدون `#ifdef` كتير — الـ linker بيشيل الـ calls الفاضية أوتوماتيك.
## Phase 5: دليل الـ Debugging الشامل

### Software Level

---

#### 1. debugfs — الـ entries المهمة

الـ **pinctrl subsystem** بيعرض معلومات تفصيلية جداً عبر `debugfs`. المسار الأساسي هو `/sys/kernel/debug/pinctrl/`.

```bash
# اعرف كل الـ pin controllers المسجلة
ls /sys/kernel/debug/pinctrl/

# مثال على output:
# pinctrl-maps   ← الـ mapping table كلها
# 1c20800.pinctrl/
# 1f02c00.pinctrl/
```

| الـ Entry | المحتوى | الاستخدام |
|---|---|---|
| `pinctrl/pinctrl-maps` | كل الـ `pinctrl_map` entries المسجلة في النظام | تأكد إن الـ machine mappings اتسجلت صح |
| `pinctrl/<dev>/pins` | قائمة كل الـ pins مع أرقامها | تحقق من أسماء الـ pins |
| `pinctrl/<dev>/pingroups` | كل الـ pin groups المتاحة | تأكد إن الـ group name في الـ map صح |
| `pinctrl/<dev>/pinmux-functions` | الـ mux functions المدعومة | تحقق من أسماء الـ functions |
| `pinctrl/<dev>/pinmux-pins` | أي device بيستخدم أي pin دلوقتي | اكتشف conflicts |
| `pinctrl/<dev>/pinconf-pins` | الـ config الحالي لكل pin | تحقق من pull/drive settings |
| `pinctrl/<dev>/pinconf-groups` | الـ config على مستوى الـ group | |

```bash
# اقرأ الـ mapping table كلها — ده أهم ملف للـ machine.h debugging
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# مثال على output:
# Mappings in the system, ordered by pin controller:
#
# device spi0
# state default
# type MUX_GROUP (2)
# controlling device pinctrl0
# group spi0grp
# function spi0
#
# device i2c1
# state default
# type CONFIGS_PIN (3)
# controlling device pinctrl0
# pin SDA
# config 00000001

# شوف مين بيستخدم الـ pins دلوقتي
cat /sys/kernel/debug/pinctrl/1c20800.pinctrl/pinmux-pins

# Output مثال:
# pin 0 (PA0): device 1c28000.uart function uart0 group uart0grp
# pin 1 (PA1): device 1c28000.uart function uart0 group uart0grp
# pin 4 (PA4): UNCLAIMED
```

---

#### 2. sysfs — الـ entries المهمة

الـ **sysfs** بتوفر معلومات عن الـ pin states على مستوى الـ device.

```bash
# شوف الـ pinctrl state الحالية للـ device
cat /sys/bus/platform/devices/<dev-name>/pinctrl-names
# مثال output: default sleep

# الـ state الحالية
cat /sys/bus/platform/devices/<dev-name>/pinctrl-0   # symlink لـ pinctrl handle

# شوف الـ pinctrl handle الـ device بيمسكه
ls -la /sys/bus/platform/devices/1c28000.uart/

# GPIO sysfs — لو الـ pin متعين كـ GPIO
ls /sys/class/gpio/
echo 32 > /sys/class/gpio/export
cat /sys/class/gpio/gpio32/direction
cat /sys/class/gpio/gpio32/value
```

---

#### 3. ftrace — الـ tracepoints والـ events

الـ **ftrace** عنده tracepoints مدمجة في الـ pinctrl subsystem.

```bash
# شوف الـ pinctrl events المتاحة
ls /sys/kernel/debug/tracing/events/pinctrl/

# الـ events المهمة:
# pinctrl_single_read_op
# pinctrl_map_add
# pinctrl_state_create
# pin_config_set
# pin_config_get
# pin_mux_set

# فعّل كل الـ pinctrl events
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# أو فعّل event معين
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pin_mux_set/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ device أو الـ driver اللي عايز تتابعه
# مثلاً: insmod my_driver.ko  أو  echo driver > /sys/bus/platform/drivers/mydrv/bind

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace

# مثال output:
#   kworker/0:1-12   [000]  123.456: pin_mux_set: pin=4 function=spi0
#   kworker/0:1-12   [000]  123.457: pin_config_set: pin=4 config=0x00000001

# function tracer لتتبع استدعاء pinctrl_register_mappings
echo function > /sys/kernel/debug/tracing/current_tracer
echo pinctrl_register_mappings > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

---

#### 4. printk و dynamic debug

الـ **dynamic debug** بيخلي تفعّل رسائل `dev_dbg` و `pr_debug` في الـ pinctrl بدون إعادة compile.

```bash
# فعّل كل الـ debug messages في الـ pinctrl subsystem
echo 'file drivers/pinctrl/* +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل لـ file معينة
echo 'file drivers/pinctrl/core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinconf.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/devicetree.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ rules الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep pinctrl

# من خلال kernel cmdline (في bootloader/grub)
# أضف: dyndbg="file drivers/pinctrl/* +p"

# أو في /etc/modprobe.d/ لـ module معين
# options pinctrl-sunxi dyndbg=+p

# printk level — تأكد إن الـ kernel بيطبع debug messages
dmesg -n 8
# أو
echo 8 > /proc/sys/kernel/printk

# تابع الـ kernel log
dmesg -w | grep -i pinctrl
```

---

#### 5. Kernel Config Options للـ Debugging

| الـ CONFIG | الوظيفة |
|---|---|
| `CONFIG_PINCTRL` | الـ pinctrl subsystem نفسه — لازم يكون مفعّل |
| `CONFIG_DEBUG_PINCTRL` | يفعّل رسائل debug إضافية في الـ pinctrl core |
| `CONFIG_PINMUX` | دعم الـ pin multiplexing |
| `CONFIG_PINCONF` | دعم الـ pin configuration |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | الـ generic group management |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | الـ generic function management |
| `CONFIG_GENERIC_PINCONF` | الـ generic pin config |
| `CONFIG_DEBUG_FS` | لازم مفعّل عشان debugfs يشتغل |
| `CONFIG_DYNAMIC_DEBUG` | لازم مفعّل عشان dynamic debug يشتغل |
| `CONFIG_FTRACE` | الـ tracing infrastructure |
| `CONFIG_TRACING` | الـ tracing base |
| `CONFIG_KALLSYMS` | بيساعد في قراءة الـ stack traces |
| `CONFIG_LOCKDEP` | يكتشف deadlocks في الـ pinctrl locks |
| `CONFIG_PROVE_LOCKING` | تحقق متقدم من الـ locking |
| `CONFIG_GPIO_SYSFS` | يعرض الـ GPIO pins في sysfs |

```bash
# تحقق من الـ config في الـ running kernel
zcat /proc/config.gz | grep -E 'PINCTRL|PINMUX|PINCONF|DEBUG_PINCTRL'

# أو
grep -E 'PINCTRL|PINMUX|PINCONF' /boot/config-$(uname -r)
```

---

#### 6. subsystem-specific tools

```bash
# pinctrl-utils من libgpiod/pinctrl tools (لو متاح)
# على بعض الـ distros:
apt install pinctrl-utils   # أو المكافئ

# على Raspberry Pi OS في الـ latest versions
pinctrl get          # اقرأ الـ state الحالي لكل pin
pinctrl set 4 op    # اضبط pin 4 كـ output

# raspi-gpio (على RPi)
raspi-gpio get       # اقرأ كل الـ GPIO pins

# gpioinfo من libgpiod
gpioinfo             # اعرض كل الـ GPIO chips والـ lines
gpiodetect           # اكتشف الـ GPIO controllers

# i2cdetect / i2cget — لتحقق من الـ I2C pins بعد الـ mux
i2cdetect -l
i2cdetect -y 1

# لو بتستخدم device مع regmap
cat /sys/kernel/debug/regmap/<dev>/registers

# devlink — للـ network devices مع pinctrl
devlink dev info
```

---

#### 7. رسائل الـ Error الشائعة

| رسالة الـ Error | المعنى | الحل |
|---|---|---|
| `could not get pinctrl` | الـ driver طلب pinctrl handle بس مش لاقي mapping للـ device | تأكد إن `dev_name` في الـ `pinctrl_map` يطابق اسم الـ device بالظبط |
| `could not find function <func>` | الـ function المطلوبة في الـ map مش موجودة عند الـ pin controller | تحقق من اسم الـ function في `pinctrl-maps` debugfs |
| `could not find group <grp>` | الـ group name في الـ map مش موجودة | اقرأ `pingroups` من debugfs وتأكد من الاسم |
| `pin <N> is already requested` | pin بيتطلبه أكتر من device — conflict | شوف `pinmux-pins` في debugfs، حدد مين بياخده |
| `could not get state <state>` | الـ state المطلوبة (default/sleep) مش معرفة في الـ map | أضف entry للـ state في الـ mapping table |
| `too many configs for pin` | عدد الـ configs تجاوز الحد المسموح | قلل عدد الـ config entries لهذا الـ pin |
| `invalid pin controller` | `ctrl_dev_name` في الـ map لا يطابق أي pin controller مسجل | تحقق من اسم الـ pin controller في `ls /sys/kernel/debug/pinctrl/` |
| `pinctrl_register_mappings: map name not set` | الـ `name` field في `pinctrl_map` فاضية أو NULL | تأكد إن الـ state name متعين في كل entry |
| `maps table is null` | تمرير `NULL` لـ `pinctrl_register_mappings` | تحقق من الـ initialization code |
| `devm_pinctrl_get() failed` | فشل في الحصول على pinctrl handle | شوف dmesg لرسالة أكثر تفصيلاً قبلها مباشرة |
| `pin already hog` | الـ pin controller بيحاول يـ hog pin محجوز بالفعل | مشكلة في الـ HOG macros — راجع الـ map entries |
| `could not select state` | `pinctrl_select_state()` فشلت | تحقق من الـ hardware و الـ mux settings |

---

#### 8. أماكن وضع dump_stack() و WARN_ON()

```c
/* في drivers/pinctrl/core.c — عند تسجيل الـ mappings */
int pinctrl_register_mappings(const struct pinctrl_map *map,
                               unsigned int num_maps)
{
    /* نقطة مهمة: تحقق إن الـ map مش NULL وإن num_maps > 0 */
    if (WARN_ON(!map || !num_maps))
        return -EINVAL;

    for (i = 0; i < num_maps; i++) {
        /* تحقق من كل entry إن dev_name موجود */
        if (WARN_ON(!map[i].dev_name)) {
            pr_err("map entry %d: dev_name is NULL\n", i);
            dump_stack();  /* ← هنا تعرف مين اللي بعت الـ map الفاسدة */
            return -EINVAL;
        }
    }
}

/* في drivers/pinctrl/pinmux.c — عند الـ mux conflict */
static int pin_request(struct pinctrl_dev *pctldev,
                       int pin, const char *owner,
                       struct pinctrl_gpio_range *gpio_range)
{
    desc = pin_desc_get(pctldev, pin);
    if (desc->mux_usecount) {
        /* ← هنا لو عايز تعرف مين كان بياخد الـ pin قبلك */
        WARN(1, "pin %d already in use by %s, requested by %s\n",
             pin, desc->mux_owner, owner);
        dump_stack();
    }
}

/* في machine driver — عند تسجيل الـ map */
static int __init my_board_init(void)
{
    int ret = pinctrl_register_mappings(my_pin_map, ARRAY_SIZE(my_pin_map));
    if (WARN_ON(ret)) {
        pr_err("Failed to register pinctrl mappings: %d\n", ret);
        dump_stack();
        return ret;
    }
    return 0;
}
```

**النقاط الاستراتيجية:**
- في `pinctrl_register_mappings()` — عند بداية الـ registration
- في `pinctrl_get()` — عند فشل إيجاد الـ device في الـ maps
- في `pinmux_request_gpio()` — عند conflict بين GPIO وـ pinmux
- في `pinctrl_select_state()` — عند فشل تغيير الـ state

---

### Hardware Level

---

#### 1. التحقق إن الـ Hardware state يطابق الـ Kernel state

```bash
# الخطوة الأولى: اقرأ الـ pinmux state من debugfs
cat /sys/kernel/debug/pinctrl/1c20800.pinctrl/pinmux-pins
# Output: pin 4 (PA4): device spi0 function spi0 group spi0grp

# الخطوة الثانية: اقرأ الـ pinconf للـ pin ده
cat /sys/kernel/debug/pinctrl/1c20800.pinctrl/pinconf-pins
# Output: pin 4 (PA4): pull up, drive strength 2

# الخطوة الثالثة: قارن مع الـ hardware registers (راجع datasheet)
# مثلاً على Allwinner A20:
# PA4 config register: 0x01c20804
# بت 16:18 = function select (3 bits per pin)
# بت 0:2 = function select for PA4

# تحقق من GPIO value لو متعين كـ GPIO
gpioget gpiochip0 4

# تحقق من الـ interrupt إن كان مفعّل
cat /proc/interrupts | grep -i gpio
```

---

#### 2. Register Dump

```bash
# devmem2 (لازم تثبته: apt install devmem2)
# اقرأ register معين — مثال Allwinner A20 PA config
devmem2 0x01c20800 w   # PA Config Register 0
devmem2 0x01c20804 w   # PA Config Register 1
devmem2 0x01c20808 w   # PA Data Register
devmem2 0x01c2080c w   # PA Drive Strength 0
devmem2 0x01c20810 w   # PA Pull Register 0

# /dev/mem — لو devmem2 مش متاح
# لازم CONFIG_STRICT_DEVMEM=n أو استخدم /dev/kmem
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0x01c20800 & ~0xfff, access=mmap.ACCESS_READ)
    offset = 0x01c20800 & 0xfff
    val = struct.unpack('<I', m[offset:offset+4])[0]
    print(f'PA_CFG0 = 0x{val:08x}')
    m.close()
"

# io utility (من busybox أو iotools)
io -4 -r 0x01c20800

# /sys/kernel/debug/regmap — لو الـ driver بيستخدم regmap
ls /sys/kernel/debug/regmap/
cat /sys/kernel/debug/regmap/1c20800.pinctrl/registers

# مثال output:
# 00000000: 11177776
# 00000004: 00000001
# تفسيره: كل 3 bits بتمثل function لـ pin واحد
```

**تحليل Register Values — مثال Allwinner:**

```
PA_CFG0 Register (0x01c20800):
Bit [2:0]  = PA0 function: 0=input, 1=output, 2=uart0_TX, ...
Bit [6:4]  = PA1 function
Bit [10:8] = PA2 function
...
Value 0x11177776 means:
PA0 = 6 (custom), PA1 = 7 (IO disable), PA2 = 7, PA3 = 7,
PA4 = 1 (output), PA5 = 1 (output), PA6 = 1 (output), PA7 = 1 (output)
```

---

#### 3. Logic Analyzer و Oscilloscope

**نصائح عملية:**

| الأداة | الاستخدام | النقطة |
|---|---|---|
| Logic Analyzer | تحقق من الـ mux function الفعلي | قِس على الـ pin مباشرة |
| Oscilloscope | قِس الـ drive strength و slew rate | قارن مع الـ config |
| Multimeter | تحقق من الـ pull-up/pull-down | قِس مقاومة للـ VCC/GND |

```
أماكن القياس:
                    +3.3V
                      │
                    [R] ← pull-up resistor (داخلي أو خارجي)
                      │
SoC Pin ────────────────── Logic Analyzer Channel 0
                      │
                    [R] ← pull-down (لو enabled)
                      │
                     GND

- لو الـ function = SPI: قِس SCLK (clock يجي regular)
- لو الـ function = UART: قِس TX (UART frames تظهر)
- لو الـ pull-up مفعّل بس السيجنال بيوقع: الـ drive strength ضعيفة أو الـ config غلط
```

**خطوات التحقق:**
1. قِس الـ DC voltage على الـ pin — هل يتطابق مع الـ expected state؟
2. فعّل الـ device وراقب الـ signal على الـ logic analyzer
3. قارن الـ timing مع الـ datasheet
4. لو مفيش activity: الـ mux مش مضبوط — راجع debugfs

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | الـ Pattern في dmesg | التفسير |
|---|---|---|
| Pin مش متعين للـ function الصح | `could not find function uart0` | الـ hardware بيستقبل بس الـ software مش بيشوفه |
| Conflict بين GPIO وـ pinmux | `pin X is already requested` | Pin متعين لـ GPIO ثم طُلب لـ peripheral |
| الـ pull resistor مش مضبوط | بيانات خاطئة أو glitches في I2C | `i2c i2c-1: arbitration lost` |
| Drive strength منخفضة | إشارة ضعيفة على high-speed bus | `spi transfer timeout` أو CRC errors |
| الـ pinctrl driver مش loaded | `devm_pinctrl_get() failed` | الـ hardware موجود بس الـ mapping مش متسجلة |
| الـ VCC للـ pin group خاطئ | الـ pin بيشتغل بس بـ voltage خاطئ | قد يتلف الـ device المتصل |

```bash
# مراقبة الـ kernel log أثناء الـ boot
dmesg | grep -E 'pinctrl|pinmux|pinconf|gpio' | head -50

# تابع في الـ real time
dmesg -w | grep -iE 'pin|gpio|mux'

# شوف الـ probe sequence
dmesg | grep -E 'probe|pinctrl' | grep -v 'succeeded'
```

---

#### 5. Device Tree Debugging

الـ **Device Tree** هو المصدر الرئيسي للـ pinctrl mappings في الأنظمة الحديثة. الـ `pinctrl_map` entries بتُبنى من الـ DT تلقائياً.

```bash
# اقرأ الـ compiled DTB الفعلي اللي الـ kernel بيشغله
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 'pinctrl'

# أو استخدم fdtdump لو عندك الـ DTB file
fdtdump /boot/dtb/myboard.dtb | grep -A 10 'pinctrl'

# شوف الـ DT properties للـ pinctrl node
ls /sys/firmware/devicetree/base/soc/pinctrl*/
cat /sys/firmware/devicetree/base/soc/pinctrl@1c20800/uart0-pins/function
cat /sys/firmware/devicetree/base/soc/pinctrl@1c20800/uart0-pins/pins

# تحقق إن الـ phandle صح
# في DTS: pinctrl-0 = <&uart0_pins_a>;
# هنا بنتحقق إن الـ reference صح
cat /sys/firmware/devicetree/base/uart@1c28000/pinctrl-names
# Expected output: default

# تحقق من compatible string
cat /sys/firmware/devicetree/base/soc/pinctrl@1c20800/compatible
# Expected: allwinner,sun7i-a20-pinctrl (مثلاً)

# لو في مشكلة في الـ DT overlay
ls /sys/kernel/config/device-tree/overlays/
cat /sys/kernel/config/device-tree/overlays/myoverlay/status
```

**أشهر أخطاء الـ DT:**

```dts
/* خطأ شائع: اسم الـ state مختلف */
/* في الـ DTS: */
&uart0 {
    pinctrl-names = "default", "sleep";  /* لازم يطابق PINCTRL_STATE_DEFAULT */
    pinctrl-0 = <&uart0_pins>;
    pinctrl-1 = <&uart0_pins_sleep>;
};

/* لو كتبت "Default" بدل "default" → kernel مش هيلاقيه */

/* خطأ ثاني: نسيت تعرّف الـ sleep state */
&uart0 {
    pinctrl-names = "default";  /* ← لو في suspend/resume هتحتاج sleep state */
    pinctrl-0 = <&uart0_pins>;
};
```

```bash
# تحقق إن الـ pinctrl mappings اتبنت من الـ DT
cat /sys/kernel/debug/pinctrl/pinctrl-maps | grep uart0
# لو مفيش output: الـ DT مش صح أو الـ driver مش probe

# شوف الـ OF (OpenFirmware) match للـ driver
dmesg | grep 'OF: graph'
dmesg | grep 'pinctrl.*probe'
```

---

### Practical Commands

---

#### سيناريو 1: الـ device مش بيشتغل بعد الـ boot

```bash
#!/bin/bash
# troubleshoot-pinctrl.sh — سكريبت متكامل للـ debugging

DEV="1c28000.uart"  # غيّره لاسم الـ device بتاعك
PINCTRL="1c20800.pinctrl"  # غيّره لاسم الـ pin controller

echo "=== 1. Check if device probe succeeded ==="
dmesg | grep -E "$DEV|probe" | tail -20

echo "=== 2. Check pinctrl mappings for device ==="
grep -A 8 "device $DEV" /sys/kernel/debug/pinctrl/pinctrl-maps

echo "=== 3. Check which pins are claimed ==="
grep "$DEV" /sys/kernel/debug/pinctrl/$PINCTRL/pinmux-pins

echo "=== 4. Check for conflicts ==="
grep "already requested" /var/log/kern.log 2>/dev/null || dmesg | grep "already"

echo "=== 5. Current pin config ==="
cat /sys/kernel/debug/pinctrl/$PINCTRL/pinconf-pins
```

---

#### سيناريو 2: تتبع تسجيل الـ mappings

```bash
# فعّل dynamic debug قبل الـ module load
echo 'file drivers/pinctrl/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/devicetree.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# سجّل الـ driver من جديد
echo 1c28000.uart > /sys/bus/platform/drivers/serial8250/unbind 2>/dev/null
echo 1c28000.uart > /sys/bus/platform/drivers/serial8250/bind

# شوف الـ log
dmesg | tail -50

# مثال output مفيد:
# [   12.345] drivers/pinctrl/core.c:752 pinctrl_get():
#              requesting pin control handle for device 1c28000.uart
# [   12.346] drivers/pinctrl/core.c:801 create_pinctrl():
#              found state "default" in map for device 1c28000.uart
# [   12.347] drivers/pinctrl/pinmux.c:386 pin_request():
#              pin 0 (PA0) claimed by device 1c28000.uart
```

---

#### سيناريو 3: تحقق من الـ machine mappings مباشرة

```bash
# اطبع الـ mapping table كاملة مع تفسير
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# فلتر حسب الـ device
awk '/^device/{dev=$2} dev=="spi0"{print}' \
    /sys/kernel/debug/pinctrl/pinctrl-maps

# شوف كل الـ MUX_GROUP entries
grep -A 6 "type MUX_GROUP" /sys/kernel/debug/pinctrl/pinctrl-maps

# شوف كل الـ CONFIGS entries
grep -A 8 "type CONFIGS" /sys/kernel/debug/pinctrl/pinctrl-maps
```

---

#### سيناريو 4: ftrace لتتبع state transitions

```bash
# تتبع كل استدعاءات pinctrl_select_state
cd /sys/kernel/debug/tracing

echo 0 > tracing_on
echo > trace
echo function > current_tracer
echo 'pinctrl_select_state' > set_ftrace_filter
echo 1 > tracing_on

# شغّل عملية suspend/resume
echo mem > /sys/power/state &
sleep 3
# أو: systemctl suspend

echo 0 > tracing_on
cat trace

# Output مثال:
# kworker/0:0-8   [000] 1234.56: pinctrl_select_state <-pinctrl_pm_select_sleep_state
# kworker/0:0-8   [000] 1234.57: pinctrl_select_state <-pinctrl_pm_select_default_state
```

---

#### سيناريو 5: تحقق من register values مقابل الـ DT config

```bash
# مقارنة الـ expected config من DT مع الـ actual hardware registers
# مثال لـ pin PA4 على Allwinner A20

echo "=== Expected from DT/debugfs ==="
cat /sys/kernel/debug/pinctrl/1c20800.pinctrl/pinconf-pins | grep "PA4"

echo "=== Actual hardware register ==="
# PA4 في الـ PA_CFG0 register، bits [18:16]
PA_CFG0=$(devmem2 0x01c20800 w 2>/dev/null | grep "Read" | awk '{print $NF}')
echo "PA_CFG0 = $PA_CFG0"
# استخرج bits [18:16] manually من الـ hex value

echo "=== GPIO value ==="
gpioget gpiochip0 4 2>/dev/null || echo "gpioget not available"
```

---

#### تفسير Output نموذجي لـ pinctrl-maps

```
Mappings in the system, ordered by pin controller:

device spi0                    ← اسم الـ device (dev_name في pinctrl_map)
state default                  ← اسم الـ state (name في pinctrl_map)
type MUX_GROUP (2)             ← PIN_MAP_TYPE_MUX_GROUP
controlling device pinctrl0   ← ctrl_dev_name
group spi0grp                  ← data.mux.group
function spi0                  ← data.mux.function

device spi0
state default
type CONFIGS_GROUP (4)         ← PIN_MAP_TYPE_CONFIGS_GROUP
controlling device pinctrl0
group spi0grp
config 00000001                ← data.configs.configs[0] (pull-up enabled مثلاً)
config 00000002                ← data.configs.configs[1]

device i2c1
state sleep                    ← state للـ power management
type MUX_GROUP (2)
controlling device pinctrl0
group i2c1grp-sleep
function gpio                  ← في الـ sleep: الـ pins بترجع GPIO
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART مش شغال على gateway صناعي بـ RK3562

#### العنوان
**الـ UART2 صامت تماماً على industrial gateway بـ RK3562 بعد migrate من DT إلى board file**

#### السياق
شركة بتعمل industrial IoT gateway بـ Rockchip RK3562. الفريق قرر يعمل custom BSP وكتب board file C بدل ما يعتمد على DT بالكامل، عشان يتحكم في pinctrl statically في early boot قبل ما يتحمل device tree بشكل كامل. الـ UART2 مربوط بـ RS485 transceiver بيتكلم مع PLCs.

#### المشكلة
بعد boot، الـ `/dev/ttyS2` موجود لكن أي write عليه مش بيوصل للـ hardware. الـ oscilloscope على TX pin بيبين إن الـ pin مش بيتحرك خالص — stuck على high.

#### التحليل
الفريق عمل board file وكتب `pinctrl_map` زي كده:

```c
/* board-rk3562-gateway.c */
static const struct pinctrl_map uart2_map[] = {
    PIN_MAP_MUX_GROUP("fe660000.serial",   /* dev_name: UART2 */
                      PINCTRL_STATE_DEFAULT,
                      "pinctrl",           /* ctrl_dev_name */
                      "uart2-xfer",        /* group */
                      "uart2"),            /* function */
};
```

المشكلة إن `dev_name` اللي كتبوه `"fe660000.serial"` غلط. الـ RK3562 datasheet بيقول base address هو `0xFE660000` لكن الـ kernel naming convention بتستخدم lowercase hex. لما راحوا يشوفوا:

```bash
ls /sys/bus/platform/devices/ | grep serial
# النتيجة: fe660000.serial  ← الاسم صح فعلاً
```

المشكلة مش في الاسم، المشكلة إنهم نسوا يسجلوا الـ mapping! كتبوا الـ array لكن ما استدعوش `pinctrl_register_mappings()`:

```c
/* في board init function */
static int __init gateway_init(void)
{
    /* نسوا السطر ده! */
    /* pinctrl_register_mappings(uart2_map, ARRAY_SIZE(uart2_map)); */

    platform_add_devices(gateway_devices, ARRAY_SIZE(gateway_devices));
    return 0;
}
```

لما الـ UART driver بيتحمل ويطلب `pinctrl_get()` بعدين `pinctrl_lookup_state(p, "default")` — مفيش mapping مسجل، فالـ subsystem بيرجع error أو بيستخدم fallback اللي مش بيعمل mux.

#### الحل

```c
static int __init gateway_init(void)
{
    int ret;

    /* سجّل الـ mappings الأول */
    ret = pinctrl_register_mappings(uart2_map, ARRAY_SIZE(uart2_map));
    if (ret) {
        pr_err("gateway: failed to register pinctrl mappings: %d\n", ret);
        return ret;
    }

    platform_add_devices(gateway_devices, ARRAY_SIZE(gateway_devices));
    return 0;
}
arch_initcall(gateway_init);
```

أو الأفضل في newer kernels استخدام `devm_pinctrl_register_mappings()` مع device pointer عشان cleanup تبقى automatic.

للتحقق:

```bash
# شوف الـ mappings المسجلة
cat /sys/kernel/debug/pinctrl/pinctrl-maps
# لازم تلاقي fe660000.serial بـ state "default"
```

#### الدرس المستفاد
الـ `struct pinctrl_map` array مجردها بيانات في الـ BSS/rodata — محتاج `pinctrl_register_mappings()` صراحةً عشان الـ subsystem يعرف بيها. الـ compile بيمشي بدون أي warning لو نسيت تسجيلها.

---

### السيناريو الثاني: SPI flash مش بيتعرف على Android TV box بـ Allwinner H616

#### العنوان
**الـ SPI NOR flash على Allwinner H616 TV box بيظهر في dmesg كـ "unknown" ومش بيتعرف**

#### السياق
شركة صينية بتنتج Android TV box بـ Allwinner H616. المنتج بيستخدم SPI NOR flash (W25Q128) لـ bootloader backup. الـ schematic بيوضح إن الـ flash متوصل بـ SPI0 على pins PC0-PC3.

#### المشكلة
الـ `spi-nor` driver بيظهر في dmesg:

```
spidev spi0.0: SPI-NOR not detected
```

والـ flash مش بيتعرف عليه رغم إن الـ hardware connections صح والـ flash نفسه سليم (اتعمله test بـ external programmer).

#### التحليل
فحصوا الـ DT وطلعت المشكلة إن الـ pinctrl state `"default"` موجود في DT لكن بدون config للـ drive strength:

```dts
/* h616.dtsi */
spi0_pins: spi0-pins {
    pins = "PC0", "PC1", "PC2", "PC3";
    function = "spi0";
    /* مفيش drive-strength */
};
```

الـ board file اللي الفريق أضافه بيحاول يعمل override بالـ `pinctrl_map` machine interface:

```c
static unsigned long spi0_cfg[] = {
    PIN_CONFIG_DRIVE_STRENGTH, 20,  /* mA */
};

static const struct pinctrl_map spi0_map[] = {
    PIN_MAP_MUX_GROUP_DEFAULT("spi0", "pio", "PC0-3", "spi0"),
    PIN_MAP_CONFIGS_GROUP_DEFAULT("spi0", "pio", "PC0-3", spi0_cfg),
};
```

المشكلة هنا في اسم الـ `group`: كتبوا `"PC0-3"` لكن الـ H616 pinctrl driver بيعرّف الـ group باسم `"spi0-default"` مش `"PC0-3"`. لما الـ core بيدور على الـ group في `pinctrl_map_mux`:

```c
struct pinctrl_map_mux {
    const char *group;    /* ← ده اللي بيتطابق مع اسم الـ group في الـ driver */
    const char *function;
};
```

الـ lookup بيفشل صامت، والـ pins بتفضل على الـ default GPIO function بدل SPI.

للتأكيد:

```bash
cat /sys/kernel/debug/pinctrl/pio/pinmux-functions
# بيبين كل function واسمها الصح

cat /sys/kernel/debug/pinctrl/pio/pinmux-pins
# بيبين state كل pin
# PC0..PC3 هيبانوا "GPIO" مش "spi0"
```

#### الحل

```c
static const struct pinctrl_map spi0_map[] = {
    /* استخدم الاسم الصح من driver */
    PIN_MAP_MUX_GROUP_DEFAULT("spi0", "pio", "spi0-default", "spi0"),
    PIN_MAP_CONFIGS_GROUP_DEFAULT("spi0", "pio", "spi0-default", spi0_cfg),
};
```

أو الأبسط: حل المشكلة في DT مباشرة بإضافة `drive-strength = <20>` لـ `spi0_pins` بدل board file.

#### الدرس المستفاد
الـ `group` field في `struct pinctrl_map_mux` لازم يتطابق حرفياً مع أسماء الـ groups اللي الـ pinctrl driver سجلها. مفيش validation وقت التسجيل — الـ error بييجي late في runtime لما الـ driver بيعمل probe.

---

### السيناريو الثالث: I2C glitch على sensor IoT بـ STM32MP1 في suspend/resume

#### العنوان
**الـ I2C bus بيتعلق بعد resume من suspend على STM32MP157 IoT sensor board**

#### السياق
board بتستخدم STM32MP157C لـ industrial IoT sensor. الـ board بيدخل في `suspend-to-RAM` كل دقيقة عشان توفير طاقة. بعد resume، الـ I2C1 اللي بيقرا من temperature sensors بيبقى stuck ومش بيرد.

#### المشكلة
بعد `echo mem > /sys/power/state` وصحيان:

```
i2c i2c-0: controller timed out
i2c i2c-0: bus stuck (SDA low), trying bus recovery
i2c i2c-0: recovery failed
```

#### التحليل
الفريق راح على الـ pinctrl sleep state. الـ STM32MP1 DT عنده:

```dts
i2c1_sleep_pins: i2c1-sleep {
    pins = "PA9", "PA10";
    function = "gpio";
    bias-disable;
};
```

لكن الـ board file عنده mapping إضافي بيتعارض:

```c
static unsigned long i2c1_sleep_cfg[] = {
    PIN_CONFIG_BIAS_PULL_DOWN,   /* ← المشكلة هنا */
    PIN_CONFIG_OUTPUT_ENABLE, 0,
};

static const struct pinctrl_map i2c1_map[] = {
    PIN_MAP_MUX_GROUP("stm32mp1-i2c1",
                      PINCTRL_STATE_SLEEP,
                      "soc:pinctrl@50002000",
                      "i2c1_sleep",
                      "gpio"),
    PIN_MAP_CONFIGS_GROUP("stm32mp1-i2c1",
                          PINCTRL_STATE_SLEEP,
                          "soc:pinctrl@50002000",
                          "i2c1_sleep",
                          i2c1_sleep_cfg),
};
```

الـ `PIN_CONFIG_BIAS_PULL_DOWN` على SDA وSCL وقت الـ sleep بيشد الـ lines لـ low. لما slave device عنده transaction ناقصة وقت الـ suspend، الـ bus بيفضل stuck بعد resume لأن SDA لسه low.

الـ `struct pinctrl_map_configs` بيبعت الـ `configs` array للـ hardware:

```c
struct pinctrl_map_configs {
    const char *group_or_pin;
    unsigned long *configs;    /* ← ده اللي بيتطبق على الـ HW */
    unsigned int num_configs;
};
```

#### الحل
غيّر الـ sleep config يستخدم `bias-disable` مش pull-down:

```c
static unsigned long i2c1_sleep_cfg[] = {
    PIN_CONFIG_BIAS_DISABLE,    /* مش pull-down */
    PIN_CONFIG_INPUT_ENABLE,    /* خلي الـ pins inputs */
};
```

وأضاف bus recovery في الـ I2C driver's resume:

```c
/* في i2c resume callback */
static int board_i2c_resume(struct device *dev)
{
    /* أول حاجة: select default state */
    pinctrl_pm_select_default_state(dev);
    /* بعدين recover الـ bus لو محتاج */
    i2c_recover_bus(adapter);
    return 0;
}
```

#### الدرس المستفاد
الـ `PINCTRL_STATE_SLEEP` config بيتطبق على الـ pins أثناء suspend. أي pull-down على I2C lines ممكن يعمل glitch أو يخلي الـ bus stuck. الـ sleep state المناسب لـ I2C هو input بدون pull.

---

### السيناريو الرابع: HDMI مش بيظهر على i.MX8MQ media player بعد board respin

#### العنوان
**الـ HDMI output مات تماماً على i.MX8MQ media player بعد PCB respin للـ rev2**

#### السياق
شركة بتعمل digital signage media player بـ NXP i.MX8MQ. عملوا PCB respin (rev2) غيّروا فيه routing بعض الـ pins وإضافوا مقاومات pull-up مختلفة على DDC lines (I2C للـ HDMI EDID). بعد respin، الـ HDMI شاشة مش بتظهر عليها صورة وdmesg بيقول:

```
imx-drm: hdmi: failed to read EDID
```

#### المشكلة
الـ EDID read فاشل يعني الـ DDC I2C (SCL/SDA على HDMI connector) مش شغال. الـ rev1 كان شغال تمام.

#### التحليل
الـ rev2 board file عنده mapping للـ HDMI DDC pins:

```c
/* board-imx8mq-player-rev2.c */
static unsigned long hdmi_ddc_cfg_rev2[] = {
    PIN_CONFIG_BIAS_PULL_UP,
    PIN_CONFIG_DRIVE_OPEN_DRAIN,
};

static const struct pinctrl_map hdmi_map_rev2[] = {
    PIN_MAP_CONFIGS_PIN_DEFAULT(
        "hdmi",
        "30330000.iomuxc",   /* ctrl_dev_name */
        "HDMI_DDC_SCL",
        hdmi_ddc_cfg_rev2
    ),
    PIN_MAP_CONFIGS_PIN_DEFAULT(
        "hdmi",
        "30330000.iomuxc",
        "HDMI_DDC_SDA",
        hdmi_ddc_cfg_rev2
    ),
};
```

المشكلة إن الـ rev2 PCB أضاف external 2.2kΩ pull-ups على DDC lines. مع `PIN_CONFIG_BIAS_PULL_UP` إضافي من الـ SoC، الـ effective pull-up قوي جداً وبيرفع rise time فوق الـ I2C spec. الـ EDID read بيفشل بسبب signal integrity مش بسبب function مشكلة.

الحل مش هو تغيير الـ `pinctrl_map` type — هو تغيير الـ config values. استخدموا `PIN_MAP_TYPE_CONFIGS_PIN` بشكل صح لكن بقيم غلط للـ rev2 hardware.

```bash
# للتشخيص: جرب DDC manually
i2cdetect -y 2   # I2C bus رقم 2 = DDC

# لو مش شايف 0x50 (EDID address) → signal problem
# استخدم oscilloscope على SCL/SDA
```

#### الحل

```c
/* للـ rev2: disable internal pull-up لأن عنده external */
static unsigned long hdmi_ddc_cfg_rev2[] = {
    PIN_CONFIG_BIAS_DISABLE,      /* مفيش internal pull */
    PIN_CONFIG_DRIVE_OPEN_DRAIN,
};
```

وعشان يفرق بين rev1 وrev2 في نفس الكود:

```c
static int __init player_pinctrl_init(void)
{
    const struct pinctrl_map *map;
    int num;

    if (board_is_rev2()) {
        map = hdmi_map_rev2;  /* no internal pull */
        num = ARRAY_SIZE(hdmi_map_rev2);
    } else {
        map = hdmi_map_rev1;  /* with internal pull */
        num = ARRAY_SIZE(hdmi_map_rev1);
    }

    return pinctrl_register_mappings(map, num);
}
```

#### الدرس المستفاد
الـ `struct pinctrl_map_configs` بتتحكم في electrical characteristics مش بس في الـ mux. أي board respin بيغير الـ external circuitry لازم يصحب review للـ pin configs حتى لو الـ function نفسه ما اتغيرش.

---

### السيناريو الخامس: automotive ECU بـ AM62x بيعمل kernel panic في early boot بسبب hog mapping غلط

#### العنوان
**الـ kernel بيعمل panic في early boot على TI AM62x automotive ECU بسبب pinctrl hog مكسور**

#### السياق
شركة bيعمل automotive ECU بـ TI AM625 (AM62x family). الـ board بيستخدم CAN FD للاتصال مع باقي الـ ECUs، وUART للـ debug console. الفريق استخدم `PIN_MAP_MUX_GROUP_HOG_DEFAULT` في board file عشان يعمل CAN pins مباشرة من الـ pinctrl driver نفسه عند تسجيله.

#### المشكلة
الـ kernel بيعمل panic في early boot:

```
[    0.312000] pinctrl core: add range [0, 127] for controller pinctrl@f4000
[    0.318000] BUG: kernel NULL pointer dereference, address: 0000000000000010
[    0.318000] PC is at pinctrl_commit_state+0x...
```

#### التحليل
الفريق كتب:

```c
static const struct pinctrl_map can0_hog_map[] = {
    PIN_MAP_MUX_GROUP_HOG_DEFAULT(
        "pinctrl@f4000",   /* dev = ctrl_dev_name للـ hog */
        "mcan0_pins",      /* group */
        "mcan0"            /* function */
    ),
};
```

الـ `PIN_MAP_MUX_GROUP_HOG_DEFAULT` macro بيبسّط إلى:

```c
PIN_MAP_MUX_GROUP("pinctrl@f4000",      /* dev_name */
                  PINCTRL_STATE_DEFAULT,
                  "pinctrl@f4000",      /* ctrl_dev_name = dev_name */
                  "mcan0_pins",
                  "mcan0")
```

لما `dev_name == ctrl_dev_name` ده معناه إن الـ pinctrl controller نفسه هو اللي بيعمل hog على الـ pins — ده صح. المشكلة إن الـ `group` name `"mcan0_pins"` مش موجود في الـ AM62x pinctrl driver. الـ driver بيستخدم naming convention `"mcan0-default"`.

لما الـ pinctrl driver بيتسجل، الـ core بيحاول يطبق الـ hog state فوراً (ده الـ hog behavior). بيدور على الـ group `"mcan0_pins"` ومش بيلاقيه، بيرجع pointer فاضي أو invalid state، وبعدين `pinctrl_commit_state` بيعمل NULL deref.

```bash
# لو وصلت لـ shell (عبر initramfs debug)
cat /sys/kernel/debug/pinctrl/pinctrl@f4000/pingroups
# هيبين الأسماء الصح زي "mcan0-default"
```

#### الحل

```c
static const struct pinctrl_map can0_hog_map[] = {
    PIN_MAP_MUX_GROUP_HOG_DEFAULT(
        "pinctrl@f4000",
        "mcan0-default",    /* الاسم الصح من الـ driver */
        "mcan0"
    ),
};
```

لو الـ group name مش واضح، الأسلم تستخدم `NULL` للـ group:

```c
/* لو group = NULL، الـ core بياخد أول group تناسب الـ function */
PIN_MAP_MUX_GROUP_HOG_DEFAULT(
    "pinctrl@f4000",
    NULL,       /* auto-select first matching group */
    "mcan0"
),
```

الـ `struct pinctrl_map_mux` بيوضح ده:

```c
struct pinctrl_map_mux {
    const char *group;    /* may be left NULL */
    const char *function;
};
```

#### الدرس المستفاد
الـ hog mappings بيتطبقوا فوراً وقت تسجيل الـ pinctrl driver — قبل ما أي device تاني يتحمل. أي خطأ فيها بيعمل crash في early boot. لما تكتب hog، إما تتأكد من الـ group name الصح، أو استخدم `NULL` وخلي الـ core يختار. راجع دايماً `pingroups` debugfs قبل ما تكتب الـ mapping.
## Phase 7: مصادر ومراجع

### توثيق Kernel الرسمي

أهم مرجع للبدء هو التوثيق الرسمي الموجود في شجرة الـ kernel مباشرةً:

| المسار | المحتوى |
|--------|---------|
| `Documentation/driver-api/pin-control.rst` | التوثيق الكامل للـ pinctrl subsystem — architecture, API, device tree, board files |
| `Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt` | Device tree bindings للـ pin controllers |
| `include/linux/pinctrl/machine.h` | الـ machine interface — `pinctrl_map`, macros |
| `include/linux/pinctrl/pinctrl.h` | الـ core pinctrl API |
| `include/linux/pinctrl/consumer.h` | الـ consumer API — `pinctrl_get`, `pinctrl_select_state` |
| `include/linux/pinctrl/pinconf.h` | الـ pin configuration interface |
| `include/linux/pinctrl/pinconf-generic.h` | الـ generic pin config parameters |
| `drivers/pinctrl/` | الـ core implementation + drivers |

الرابط الأسرع للتوثيق الـ rendered:

- [PINCTRL (PIN CONTROL) subsystem — docs.kernel.org](https://docs.kernel.org/driver-api/pin-control.html)
- [PINCTRL — kernel.org v4.14](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html)
- [pinctrl.txt (النسخة القديمة)](https://www.kernel.org/doc/Documentation/pinctrl.txt)

---

### مقالات LWN.net

**الـ LWN** هو المرجع التاريخي الأهم لفهم كيف تطور الـ subsystem من الصفر:

| المقال | الأهمية |
|--------|---------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | المقال الرئيسي — شرح كامل للـ architecture عند الإضافة لـ kernel 3.1 |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | نص التوثيق الأصلي كما وصل للـ mailing list |
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | الـ patch الأصلي — v9 من Linus Walleij |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | مرحلة تطور مبكرة للـ subsystem |
| [drivers: create a pin control subsystem v8](https://lwn.net/Articles/460768/) | الـ v8 patch series |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/467269/) | إضافة الـ generic pin configuration |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | الـ interface النهائي للـ pin config |
| [drivers: create a pinmux subsystem v3](https://lwn.net/Articles/447394/) | النسخة المبكرة قبل تغيير الاسم إلى pinctrl |
| [i2c: Add generic I2C multiplexer using pinctrl API](https://lwn.net/Articles/498071/) | مثال عملي على استخدام الـ pinctrl API |
| [pinctrl: introduce the concept of a GPIO pin function category](https://lwn.net/Articles/1031226/) | تطوير حديث (2025) — إضافة GPIO function category |

---

### Commits مهمة في الـ Kernel

الـ commits دي بتروي قصة تطور الـ `machine.h` من أول يوم:

```
2744e8afb3b7  drivers: create a pin control subsystem
              (Linus Walleij, May 2011 — الـ commit الأصلي)

1e2082b52072  pinctrl: enhance mapping table to support pin config operations
              (أضاف PIN_MAP_TYPE_CONFIGS_PIN و PIN_MAP_TYPE_CONFIGS_GROUP)

6e5e959dde0d  pinctrl: API changes to support multiple states per device
              (دعم multiple states للـ device الواحد)

46919ae63d48  pinctrl: introduce PINCTRL_STATE_DEFAULT, define hogs as that state
              (أضاف PINCTRL_STATE_DEFAULT)

5b3aa5f7c628  pinctrl: add pinctrl_provide_dummies interface for platforms to use
              (أضاف pinctrl_provide_dummies)

c72bed23b9e4  pinctrl: Allow modules to use pinctrl_[un]register_mappings
              (السماح للـ modules باستخدام register_mappings)

2e9ba1d9a31f  pinctrl: core: add devm_pinctrl_register_mappings()
              (Thomas Richard, Bootlin, May 2025 — أحدث إضافة)
```

للبحث في الـ git history مباشرةً:

```bash
git log --oneline --follow include/linux/pinctrl/machine.h
git log --oneline drivers/pinctrl/core.c | head -20
```

الـ maintainer tree:
- [pub/scm/linux/kernel/git/linusw/linux-pinctrl — Google Kernel Git](https://kernel.googlesource.com/pub/scm/linux/kernel/git/linusw/linux-pinctrl/)
- [Linaro Git — linux-pinctrl](https://git.linaro.org/people/linus.walleij/linux-pinctrl.git)

---

### kernelnewbies.org

صفحات الـ kernel releases بتوضح كل الـ pinctrl drivers الجديدة في كل إصدار:

| الصفحة | المحتوى |
|--------|---------|
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | تاريخ كل التغييرات في الـ kernel |
| [Linux 6.17](https://kernelnewbies.org/Linux_6.17) | pinctrl drivers جديدة: Eswin eic7700، Qualcomm sm7635، RPi RP1 |
| [Linux 6.15](https://kernelnewbies.org/Linux_6.15) | SG2042، Amlogic، Allwinner A523، Rockchip rk3528 |
| [Linux 6.11](https://kernelnewbies.org/Linux_6.11) | تحديثات على الـ pinctrl subsystem |
| [Linux 6.8](https://kernelnewbies.org/Linux_6.8) | Qualcomm X1E80100، SM8650 TLMM، Samsung pinctrl |
| [Linux 6.1](https://kernelnewbies.org/Linux_6.1) | Qualcomm LPASS LPI لـ sc8280xp و sm8450، Rockchip RV1126 |

---

### elinux.org

| الصفحة | المحتوى |
|--------|---------|
| [Pin Control & GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) | عرض تقديمي عن تطور الـ pinctrl/GPIO integration |
| [EBC Device Trees](https://elinux.org/EBC_Device_Trees) | أمثلة device tree لـ BeagleBone Black مع pinctrl |
| [Tests: i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) | اختبار الـ i2c-demux-pinctrl driver |
| [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) | موارد عامة للـ kernel development |

---

### مصادر خارجية إضافية

**مقالات تقنية:**

- [Linux device driver development: The pin control subsystem — embedded.com](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/)
  شرح عملي للـ subsystem مع code examples
- [Linux Device Tree Pinctrl Tutorial — blog.modest-destiny.com](https://blog.modest-destiny.com/posts/linux-device-tree-pinctrl-tutorial/)
  tutorial عملي لكتابة pinctrl في الـ device tree
- [Pinctrl overview — STM32MPU Wiki](https://wiki.st.com/stm32mpu/wiki/Pinctrl_overview)
  مثال واقعي على STM32 processors
- [Pinctrl device tree configuration — STM32MPU](https://wiki.st.com/stm32mpu/wiki/Pinctrl_device_tree_configuration)
  كيفية كتابة الـ DT bindings لـ ST platforms
- [Pin Multiplexing — Toradex Developer Center](https://developer.toradex.com/torizon/os-customization/use-cases/pin-multiplexing-changing-pin-functionalities-in-the-linux-device-tree/)
  تغيير pin functions في الـ device tree على الـ Toradex modules
- [Documentation/pinctrl.txt — Android Kernel (Google)](https://android.googlesource.com/kernel/common/+/bcmdhd-3.10/Documentation/pinctrl.txt)
  نسخة من التوثيق في Android kernel tree
- [include/linux/pinctrl/machine.h — Android kernel/msm](https://android.googlesource.com/kernel/msm/+/android-msm-mako-3.4-kitkat-mr1/include/linux/pinctrl/machine.h)
  نسخة تاريخية من الـ machine.h في Qualcomm MSM kernel

**Device Tree Bindings:**
- [pinctrl-bindings.txt — kernel.org](https://www.kernel.org/doc/Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt)

---

### كتب موصى بها

| الكتاب | الفصل ذو الصلة |
|--------|---------------|
| **Linux Device Drivers, 3rd Edition (LDD3)** — Corbet, Rubini, Kroah-Hartman | Chapter 1 (device model), Chapter 14 (الـ Linux Device Model) — الكتاب قديم وما بيغطيش الـ pinctrl مباشرةً لكنه أساسي لفهم الـ driver model |
| **Linux Kernel Development, 3rd Edition** — Robert Love | Chapter 17 (Devices and Modules) — فهم الـ device registration و bus/driver model |
| **Embedded Linux Primer, 2nd Edition** — Christopher Hallinan | Chapter 15 (Kernel Initialization) و Chapter 16 (اﻟـ device tree) — بيغطي الـ board setup والـ machine config |
| **Mastering Embedded Linux Programming, 3rd Edition** — Frank Vasquez, Chris Simmonds | Chapter 12 (Prototyping with Breakout Boards) — GPIO و pinctrl في الـ embedded context |
| **Linux Driver Development for Embedded Processors, 2nd Edition** — Alberto Liberal de los Ríos | Chapter 6 — شرح تفصيلي للـ pinctrl مع device tree على i.MX و Raspberry Pi |

---

### مصطلحات البحث

للبحث عن معلومات إضافية، استخدم الـ search terms دي:

```
linux pinctrl machine interface board file
linux pinctrl_map pinctrl_register_mappings
linux pinmux group function mapping table
linux pinctrl hog self-hog
linux pinctrl state default sleep idle
linux pinctrl device tree vs board file
linux pinctrl devm_pinctrl_register_mappings
linux pin controller driver development
pinctrl_provide_dummies stub
linux PIN_MAP_MUX_GROUP_HOG example
lkml pinctrl Linus Walleij
```

للبحث في الـ mailing list archives مباشرةً:
- [lore.kernel.org — linux-gpio](https://lore.kernel.org/linux-gpio/) (الـ pinctrl patches بتمر من هنا)
- [lkml.org](https://lkml.org/) — ابحث عن `pinctrl_map` أو `pinctrl_register_mappings`
- [spinics.net/lists/kernel](https://www.spinics.net/lists/kernel/) — أرشيف LKML

---

### ملخص أولويات القراءة

```
Priority 1 (ابدأ هنا):
  └── docs.kernel.org/driver-api/pin-control.html

Priority 2 (السياق التاريخي):
  └── lwn.net/Articles/468759/  (The pin control subsystem)
  └── lwn.net/Articles/465077/  (Documentation/pinctrl.txt original)

Priority 3 (تطبيق عملي):
  └── blog.modest-destiny.com   (Device Tree tutorial)
  └── embedded.com              (driver development article)
  └── wiki.st.com/stm32mpu      (real hardware example)

Priority 4 (عمق في الـ code):
  └── git log include/linux/pinctrl/machine.h
  └── drivers/pinctrl/core.c
```
## Phase 8: Writing simple module

### الفكرة

**`pinctrl_register_mappings`** هي الدالة اللي بتسجّل mapping table في subsystem الـ pinctrl — كل مرة أي board أو driver بيسجّل pin mappings جديدة، الدالة دي بتتنفّذ. هنعمل kprobe عليها عشان نشوف مين بيسجّل mappings، كام entry، وإيه اسم أول device.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_pinctrl.c
 * Hooks pinctrl_register_mappings() to log every mapping table
 * registration that happens on the system.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/pinctrl/machine.h>   /* struct pinctrl_map, pinctrl_map_type */

/* ------------------------------------------------------------------ */
/*  kprobe handler — runs just before pinctrl_register_mappings()      */
/* ------------------------------------------------------------------ */

static const char * const map_type_names[] = {
	[PIN_MAP_TYPE_INVALID]       = "INVALID",
	[PIN_MAP_TYPE_DUMMY_STATE]   = "DUMMY_STATE",
	[PIN_MAP_TYPE_MUX_GROUP]     = "MUX_GROUP",
	[PIN_MAP_TYPE_CONFIGS_PIN]   = "CONFIGS_PIN",
	[PIN_MAP_TYPE_CONFIGS_GROUP] = "CONFIGS_GROUP",
};

/*
 * pre_handler is called with pt_regs holding the function arguments.
 * On x86-64: rdi = map, rsi = num_maps
 * On arm64:  x0  = map, x1  = num_maps
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
#if defined(CONFIG_X86_64)
	const struct pinctrl_map *map      = (const struct pinctrl_map *)regs->di;
	unsigned int             num_maps  = (unsigned int)regs->si;
#elif defined(CONFIG_ARM64)
	const struct pinctrl_map *map      = (const struct pinctrl_map *)regs->regs[0];
	unsigned int             num_maps  = (unsigned int)regs->regs[1];
#else
	/* architecture not handled — just log the probe fired */
	pr_info("kprobe_pinctrl: register_mappings called (arch unsupported)\n");
	return 0;
#endif

	/* Basic sanity — pointer might be bogus on unusual configs */
	if (!map || num_maps == 0) {
		pr_info("kprobe_pinctrl: register_mappings called with empty map\n");
		return 0;
	}

	pr_info("kprobe_pinctrl: pinctrl_register_mappings() called\n");
	pr_info("  num_maps  = %u\n", num_maps);
	pr_info("  dev_name  = %s\n", map[0].dev_name  ? map[0].dev_name  : "(null)");
	pr_info("  map_name  = %s\n", map[0].name      ? map[0].name      : "(null)");
	pr_info("  ctrl_dev  = %s\n", map[0].ctrl_dev_name
	                               ? map[0].ctrl_dev_name : "(null)");

	/* Print type string if index is in range */
	if (map[0].type < ARRAY_SIZE(map_type_names))
		pr_info("  type      = %s\n", map_type_names[map[0].type]);

	/* If MUX_GROUP, also print function/group for the first entry */
	if (map[0].type == PIN_MAP_TYPE_MUX_GROUP) {
		pr_info("  mux.group    = %s\n",
		        map[0].data.mux.group    ? map[0].data.mux.group    : "(null)");
		pr_info("  mux.function = %s\n",
		        map[0].data.mux.function ? map[0].data.mux.function : "(null)");
	}

	return 0; /* 0 = continue normal execution of the probed function */
}

/* ------------------------------------------------------------------ */
/*  kprobe struct                                                       */
/* ------------------------------------------------------------------ */

static struct kprobe kp = {
	.symbol_name = "pinctrl_register_mappings",
	.pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/*  init / exit                                                         */
/* ------------------------------------------------------------------ */

static int __init kprobe_pinctrl_init(void)
{
	int ret;

	ret = register_kprobe(&kp);
	if (ret < 0) {
		pr_err("kprobe_pinctrl: register_kprobe failed: %d\n", ret);
		return ret;
	}

	pr_info("kprobe_pinctrl: hooked pinctrl_register_mappings @ %px\n",
	        kp.addr);
	return 0;
}

static void __exit kprobe_pinctrl_exit(void)
{
	unregister_kprobe(&kp);
	pr_info("kprobe_pinctrl: unhooked pinctrl_register_mappings\n");
}

module_init(kprobe_pinctrl_init);
module_exit(kprobe_pinctrl_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Example");
MODULE_DESCRIPTION("kprobe on pinctrl_register_mappings — log pin mapping registrations");
```

---

### شرح كل جزء

#### `map_type_names[]`
مصفوفة بتحوّل الـ enum `pinctrl_map_type` لـ string مقروء، عشان الـ log يكون واضح بدل ما يطلع رقم.

#### `handler_pre`
ده الـ callback اللي بيتشغّل قبل ما `pinctrl_register_mappings` تبدأ تنفّذ. بنجيب الـ arguments مباشرة من `pt_regs` لأن الـ kprobe بيشتغل قبل ما الـ stack frame يتبني، فمفيش طريقة تانية نوصّل للـ arguments غير السجلات.

#### architecture guard
الـ registers بتتغير على كل architecture — على x86-64 أول argument في `rdi` والتاني في `rsi`، بينما ARM64 بيستخدم `regs[0]` و`regs[1]`. الـ `#if` بيضمن إن الكود صح على الاتنين.

#### sanity check
لو الـ pointer `NULL` أو `num_maps == 0` بنطلع بدري عشان نتجنب kernel NULL dereference.

#### `pr_info` block
بنطبع: اسم الـ device المستخدم، اسم الـ state، اسم الـ pinctrl controller، نوع الـ mapping، وفي حالة `MUX_GROUP` بنطبع كمان الـ group والـ function — ده بيديك صورة كاملة عن إيه اللي بيتسجّل.

#### `kp` struct
بيحدد الـ `symbol_name` بالاسم عشان الـ kernel يعمل lookup في `kallsyms` ويجيب العنوان وقت الـ register — مش محتاج نحدد العنوان يدوي.

#### `module_init` / `module_exit`
`register_kprobe` بتحط breakpoint على الـ symbol وبتربط الـ handler. `unregister_kprobe` بترجع الـ instruction الأصلية وبتشيل الـ breakpoint نظيف من غير ما تأثّر على باقي الـ system.

---

### طريقة البناء والتشغيل

```bash
# Makefile بسيط
# obj-m += kprobe_pinctrl.o

make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
sudo insmod kprobe_pinctrl.ko
# حمّل أي driver بيستخدم pinctrl عشان تشوف الـ log
sudo dmesg | grep kprobe_pinctrl
sudo rmmod kprobe_pinctrl
```

### ملاحظة مهمة

الـ kprobe بيشتغل بس لو `CONFIG_KPROBES=y` موجود في الـ kernel config، ولو الـ symbol مش inlined أو مش `__init`. `pinctrl_register_mappings` exported وموجود في `drivers/pinctrl/core.c` فهو candidate مثالي.
