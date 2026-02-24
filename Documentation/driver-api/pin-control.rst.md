## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ PINCTRL Subsystem؟

تخيل إنك بتشتري شقة جديدة، وفيها لوحة كهربائية رئيسية فيها 64 كابل خارج منها. كل كابل ممكن تتصل بيه أكتر من جهاز — مثلاً الكابل رقم 5 ممكن يشتغل مع التكييف، أو مع النور، أو مع الإنترنت، بس مش الاتنين مع بعض في نفس الوقت. دي بالظبط مشكلة الـ **pin multiplexing (pinmux)**.

الـ **PINCTRL subsystem** (Pin Control Subsystem) هو الـ framework اللي بيدير الـ physical pins الموجودة على الـ SoC (System on Chip) أو أي chip بتكون متصلة بيه. الـ pins دي هي الـ "أرجل" الفعلية بتاعة الـ chip اللي بتتصل بيها الأسلاك والدوائر الإلكترونية.

---

### المشكلة اللي بتحلها

الـ SoC الحديث (زي اللي في الموبايل أو الـ Raspberry Pi) بيكون فيه:

- مئات الـ pins فيزيائية على الـ chip
- عشرات الـ peripherals جوه: SPI، I2C، UART، MMC، GPIO، إلخ
- بس الـ pins أقل بكتير من مجموع كل الـ peripherals

**الحل؟** كل pin ممكن يعمل أكتر من وظيفة بس وظيفة واحدة في وقت واحد.

مثال حقيقي: الـ Raspberry Pi عنده pins ممكن تشتغل كـ SPI أو كـ GPIO. لما تفعّل الـ SPI الـ pins دي بقت مشغولة وميقدرش تستخدمها كـ GPIO.

الـ PINCTRL هو اللي بيتحكم في كل ده بشكل منظم من غير ما الـ drivers تتقاتل على الـ pins.

---

### الـ Subsystem ده بيعمل إيه بالتحديد؟

الـ PINCTRL بيتكلم في ثلاث مستويات:

| المستوى | الاسم | الوظيفة |
|---|---|---|
| 1 | **Pin Enumeration** | ترقيم وتسمية كل pin على الـ chip |
| 2 | **Pin Multiplexing (pinmux)** | تحديد أي function هتستخدم الـ pin دي (SPI? I2C? GPIO?) |
| 3 | **Pin Configuration (pinconf)** | ضبط الخصائص الكهربية: pull-up, pull-down, drive strength, إلخ |

---

### القصة الكاملة: رحلة الـ Pin من الـ Hardware لـ Driver

**المشهد:** عندك SoC فيه 64 pin، وعايز تشغّل SPI و I2C في نفس الوقت.

**1. التسجيل (Registration):**
أول ما الـ pin controller driver بيشتغل، بيسجّل نفسه في الـ PINCTRL core بـ descriptor فيه أسماء وأرقام كل الـ pins:

```c
const struct pinctrl_pin_desc foo_pins[] = {
    PINCTRL_PIN(0, "A8"),  /* pin رقم 0 اسمه A8 */
    PINCTRL_PIN(1, "B8"),
    /* ... */
    PINCTRL_PIN(63, "H1"),
};

static struct pinctrl_desc foo_desc = {
    .name  = "foo",
    .pins  = foo_pins,
    .npins = ARRAY_SIZE(foo_pins),
    .owner = THIS_MODULE,
};
```

**2. تعريف الـ Groups:**
الـ pins المتشابهة بتتجمع في **pin groups** — مجموعة الـ pins اللي بتستخدمها الـ SPI مع بعض:

```c
static const unsigned int spi0_pins[] = { 0, 8, 16, 24 };
static const unsigned int i2c0_pins[] = { 24, 25 };
```

**3. تعريف الـ Functions:**
كل function (SPI, I2C, MMC) بترتبط بمجموعة واحدة أو أكتر من الـ groups:

```c
static const char * const spi0_groups[] = { "spi0_0_grp", "spi0_1_grp" };
static const char * const i2c0_groups[] = { "i2c0_grp" };

static const struct pinfunction foo_functions[] = {
    PINCTRL_PINFUNCTION("spi0", spi0_groups, ARRAY_SIZE(spi0_groups)),
    PINCTRL_PINFUNCTION("i2c0", i2c0_groups, ARRAY_SIZE(i2c0_groups)),
};
```

**4. الـ Mapping (ربط الجهاز بالـ Function):**
الـ board/machine code بيقول: "الجهاز `foo-spi.0` هيستخدم function `spi0`":

```c
static const struct pinctrl_map mapping[] __initconst = {
    {
        .dev_name      = "foo-spi.0",
        .name          = PINCTRL_STATE_DEFAULT, /* الحالة الافتراضية */
        .type          = PIN_MAP_TYPE_MUX_GROUP,
        .ctrl_dev_name = "pinctrl-foo",
        .data.mux.function = "spi0",
    },
};
```

**5. الاستخدام من الـ Driver:**
الـ driver بيطلب الـ pins بشكل بسيط:

```c
struct pinctrl       *p;
struct pinctrl_state *s;

p = devm_pinctrl_get(&device);           /* اجيب handle للـ pinctrl */
s = pinctrl_lookup_state(p, "default");  /* ابحث عن الـ state المطلوب */
pinctrl_select_state(p, s);              /* فعّل الـ state ده */
```

---

### الـ PINMUX: القلب النابض

الـ **PINMUX** (Pin Multiplexing) هو أهم جزء في الـ subsystem. الفكرة:

```
Physical Pin A8
    │
    ├──► SPI CLK    (لو اخترنا function spi0)
    ├──► I2C SCL    (لو اخترنا function i2c0)
    └──► GPIO       (لو مفيش حد تاني بيستخدمه)
```

الـ PINCTRL core بيمنع التعارض تلقائياً — لو اتنين drivers حاولوا يستخدموا نفس الـ pin، التاني هيتردّ عليه بـ error.

---

### العلاقة مع الـ GPIO Subsystem

الـ GPIO والـ PINCTRL بيتشاركوا نفس الـ physical pins، وده بيخلق علاقة تكاملية:

```
                pin config regs
                      │
Physical Pin ──── pad ──── pinmux ──┬── SPI
                                    ├── I2C
                                    ├── MMC
                                    └── GPIO ←── GPIO subsystem
```

- الـ **PINCTRL**: بيتحكم في الـ muxing والـ configuration (pull-up/down, drive strength)
- الـ **GPIO subsystem**: بيتحكم في قراءة وكتابة القيم (high/low) ودي شغلة مختلفة

الـ `pinctrl_gpio_request()` و `pinctrl_gpio_free()` هي الجسر الوحيد المسموح بيه بين الاتنين، وبس من جوه الـ gpiolib نفسه.

---

### الـ Pin States: قوة الـ Power Management

كل device ممكن يكون ليه أكتر من **state** للـ pins بتاعته:

| State | الاستخدام |
|---|---|
| `default` | التشغيل العادي |
| `init` | قبل الـ probe |
| `sleep` | في وضع السكون (توفير طاقة) |
| `idle` | خمول مؤقت |

مثال: الـ UART TX pin بتشتغل عادي كـ UART، بس لما الجهاز ينام بتتحول لـ GPIO output low لتوفير الطاقة:

```c
/* وضع التشغيل العادي */
pinctrl_pm_select_default_state(dev);

/* وضع النوم - الـ pin بيتحول لـ GPIO low */
pinctrl_pm_select_sleep_state(dev);
```

---

### مكونات الـ Subsystem

#### Core Files (اللب):

| الملف | الوظيفة |
|---|---|
| `drivers/pinctrl/core.c` | الـ core الرئيسي: تسجيل الـ controllers، إدارة الـ pins |
| `drivers/pinctrl/core.h` | internal structs للـ core |
| `drivers/pinctrl/pinconf.c` | إدارة الـ pin configuration |
| `drivers/pinctrl/pinconf-generic.c` | الـ generic pin config parameters |
| `drivers/pinctrl/devicetree.c` | قراءة الـ pinctrl config من الـ Device Tree |
| `drivers/pinctrl/pinctrl-generic.c` | generic pinmux/pinconf driver |

#### Headers (الواجهات):

| الملف | الوظيفة |
|---|---|
| `include/linux/pinctrl/pinctrl.h` | الـ API الرئيسي لكتابة pin controller driver |
| `include/linux/pinctrl/pinmux.h` | الـ ops الخاصة بالـ pinmux |
| `include/linux/pinctrl/pinconf.h` | الـ ops الخاصة بالـ pin configuration |
| `include/linux/pinctrl/pinconf-generic.h` | الـ generic config parameters (pull-up, drive strength، إلخ) |
| `include/linux/pinctrl/consumer.h` | الـ API اللي بيستخدمه الـ driver لطلب الـ pins |
| `include/linux/pinctrl/machine.h` | الـ mapping table structures |
| `include/linux/pinctrl/pinctrl-state.h` | الـ standard state names (default, sleep، إلخ) |
| `include/linux/pinctrl/devinfo.h` | الـ per-device pinctrl info |

#### Hardware Drivers (أمثلة):

| الملف | الـ Platform |
|---|---|
| `drivers/pinctrl/pinctrl-rockchip.c` | Rockchip SoCs (RK3288, RK3399) |
| `drivers/pinctrl/pinctrl-single.c` | single-register pinmux (OMAP, etc.) |
| `drivers/pinctrl/bcm/pinctrl-bcm2835.c` | Raspberry Pi (BCM2835) |
| `drivers/pinctrl/freescale/pinctrl-imx.c` | NXP i.MX SoCs |
| `drivers/pinctrl/pinctrl-at91-pio4.c` | Microchip AT91 (SAM) |
| `drivers/pinctrl/pinctrl-amd.c` | AMD x86 GPIO/pinctrl |
| `drivers/pinctrl/pinctrl-apple-gpio.c` | Apple Silicon |

#### الـ Debugfs:

```
/sys/kernel/debug/pinctrl/
├── pinctrl-devices      ← قائمة الـ controllers المسجّلة
├── pinctrl-handles      ← الـ handles المفعّلة حالياً
├── pinctrl-maps         ← كل الـ mappings
└── <controller-name>/
    ├── pins             ← كل الـ pins ومعلوماتها
    ├── pingroups        ← الـ groups
    ├── pinmux-functions ← الـ functions المتاحة
    ├── pinmux-pins      ← مين بيستخدم أي pin
    ├── gpio-ranges      ← الـ GPIO ranges
    ├── pinconf-pins     ← الـ config لكل pin
    └── pinmux-select    ← activate function يدوياً
```

---

### الملفات ذات الصلة اللي تعرفها

- **`Documentation/driver-api/pin-control.rst`** — الملف ده نفسه
- **`Documentation/devicetree/bindings/pinctrl/`** — كيف تكتب الـ DT bindings
- **`Documentation/devicetree/bindings/gpio/gpio.txt`** — ربط الـ pinctrl بالـ GPIO من الـ DT
- **`arch/arm/mach-ux500/Kconfig`** — مثال على تفعيل الـ pinctrl في machine-specific Kconfig
## Phase 2: شرح الـ Pin Control (pinctrl) Framework

---

### المشكلة اللي بيحلها الـ pinctrl

أي SoC حديث — خد مثلاً Allwinner H3 أو STM32MP1 — فيه عشرات أو مئات الـ physical pins على الـ package. المشكلة إنك مش قادر تعمل لكل function في الـ chip pin منفصل، ده هيخلي الـ chip ضخمة ومكلفة بشكل خيالي. الحل الهاردوير المنطقي: نخلي الـ pin الواحد يقدر يشتغل كـ UART TX، أو GPIO، أو PWM، أو I2C SDA — بس واحدة في وقت واحد.

ده بيخلق مشكلة في الـ kernel: مين بيتحكم في إن الـ pin ده UART دلوقتي؟ مين بيحطه في الوضع الصح من ناحية الـ pull-up/pull-down/drive strength؟ ومين بيمنع conflicts لو درايفرين حاولوا يستخدموا نفس الـ pin في نفس الوقت؟

قبل الـ pinctrl framework، الإجابة كانت فوضى: كل SoC بيعمل static configuration في الـ board file، مفيش centralized tracking، ومحدش بيحمي الـ pins من الـ conflict.

---

### الحل — نهج الـ kernel

الـ kernel بيحل المشكلة بـ **abstraction layer** موحد اسمه الـ **pinctrl subsystem**. الفكرة الأساسية:

1. كل pin controller (الـ hardware block المسؤول عن الـ pins) بيـregister نفسه في الـ framework ويقول "أنا بتحكم في pins رقم 0 لـ 127".
2. الـ framework بيحتفظ بـ centralized database للـ pins وإيه اللي بيستخدمهم.
3. الـ consumer (أي device driver محتاج pins) بيطلب state معينة بالاسم مش بـ registers مباشرة.
4. الـ framework بيـcheck الـ conflicts وبيفوض العملية الفعلية للـ driver.

---

### التشبيه الواقعي — لوحة توزيع الكهرباء في المستشفى

تخيل لوحة توزيع كهرباء مركزية في مستشفى. في الـ لوحة دي:

| عنصر في المستشفى | مقابله في pinctrl |
|---|---|
| **اللوحة المركزية** نفسها | الـ `pinctrl_dev` — الـ instance الواحدة للـ pin controller |
| **الكابلات الخارجة من اللوحة** | الـ physical pins على الـ SoC package |
| **مفتاح التوجيه** بتاع كل كابل | الـ pinmux hardware (الـ MUX registers) |
| **المعمل X بيطلب توصيل قاعة Y** | الـ `pinctrl_map` — device بيطلب function على pin group |
| **المسؤول اللي بيوافق أو يرفض** | الـ pinctrl core اللي بيتحقق من الـ conflicts |
| **جهاز الدراما بيشتغل فيها الكابل** | الـ consumer driver (SPI, I2C, UART ...) |
| **خصائص الكابل** (سمكه، عازله) | الـ pin configuration (pull-up, drive strength, slew rate) |
| **وضع الطوارئ للمستشفى** | الـ `PINCTRL_STATE_SLEEP` |
| **الوضع الاعتيادي** | الـ `PINCTRL_STATE_DEFAULT` |

والأهم: المسؤول (pinctrl core) هو الوحيد اللي يعرف مين شاغل إيه. لو معمل تاني حاول يأخذ كابل شاغله معمل تاني، المسؤول بيرفض على طول.

---

### Big Picture Architecture

```
╔══════════════════════════════════════════════════════════════════╗
║                      Consumer Drivers Layer                      ║
║  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  ║
║  │ UART drv │  │  SPI drv │  │  I2C drv │  │  GPIO subsys   │  ║
║  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬────────┘  ║
╚═══════╪═════════════╪═════════════╪═════════════════╪═══════════╝
        │             │             │                 │
        │  devm_pinctrl_get()       │    pinctrl_gpio_request()
        │  pinctrl_lookup_state()   │                 │
        │  pinctrl_select_state()   │                 │
        ▼             ▼             ▼                 ▼
╔══════════════════════════════════════════════════════════════════╗
║                    pinctrl CORE (kernel/drivers/pinctrl/)        ║
║                                                                  ║
║  ┌─────────────────────────────────────────────────────────┐    ║
║  │  Pin ownership DB  │  Mapping table  │  State machine   │    ║
║  │  (who owns pin N)  │  (device→func)  │  (default/sleep) │    ║
║  └────────────────────┴─────────────────┴──────────────────┘    ║
║                                                                  ║
║  pinctrl_ops vtable   pinmux_ops vtable   pinconf_ops vtable     ║
╚══════════════════════════════════════════════════════════════════╝
        │                    │                    │
        ▼                    ▼                    ▼
╔══════════════════════════════════════════════════════════════════╗
║              Pin Controller Driver (Provider Layer)              ║
║   e.g. drivers/pinctrl/pinctrl-stm32.c                         ║
║                                                                  ║
║  ┌────────────────┐  ┌───────────────┐  ┌──────────────────┐   ║
║  │  pinctrl_desc  │  │  pinmux_ops   │  │   pinconf_ops    │   ║
║  │  (pin list)    │  │  (set_mux())  │  │ (pin_config_set) │   ║
║  └────────────────┘  └───────────────┘  └──────────────────┘   ║
╚══════════════════════════════════════════════════════════════════╝
        │
        ▼
╔══════════════════════════════════════════════════════════════════╗
║                    Hardware Registers (MMIO)                     ║
║   MUX_REG, PULL_REG, DRIVE_REG, ...                             ║
╚══════════════════════════════════════════════════════════════════╝
```

---

### الـ Core Abstraction — المفاهيم الأساسية

الـ framework بيبني كل شيء على **5 مفاهيم أساسية** مترابطة:

#### 1. Pin
أصغر وحدة في الـ framework. عبارة عن `unsigned int` (رقم) + اسم string. الـ numbering محلي لكل pin controller.

```c
// من pinctrl.h
struct pinctrl_pin_desc {
    unsigned int number;  // رقم Pin محلي للـ controller
    const char *name;     // "PA3"، "GPIO_47"، إلخ
    void *drv_data;       // data خاصة بالـ driver
};

// الـ macro المريحة
#define PINCTRL_PIN(a, b) { .number = a, .name = b }

// مثال واقعي لـ STM32-style SoC
static const struct pinctrl_pin_desc myboard_pins[] = {
    PINCTRL_PIN(0,  "PA0"),
    PINCTRL_PIN(1,  "PA1"),
    PINCTRL_PIN(16, "PB0"),  // numbering ممكن تكون sparse
    PINCTRL_PIN(17, "PB1"),
};
```

#### 2. Pin Group
مجموعة pins بتشتغل مع بعض كوحدة واحدة لـ function معينة. SPI محتاج 4 pins (CLK, MOSI, MISO, CS)، لازم يتعاملوا كـ group.

```c
// من pinctrl.h
struct pingroup {
    const char *name;           // "spi0_grp"
    const unsigned int *pins;   // مصفوفة أرقام الـ pins
    size_t npins;               // عدد الـ pins
};

// مثال
static const unsigned int spi0_pins[] = { 0, 8, 16, 24 };
static const struct pingroup my_groups[] = {
    PINCTRL_PINGROUP("spi0_grp", spi0_pins, ARRAY_SIZE(spi0_pins)),
};
```

#### 3. Function
الـ function هو الـ peripheral اللي ممكن يتوصل على مجموعة من الـ pin groups. مثلاً "spi0" هي function ممكن تتوصل على "spi0_grp_A" أو "spi0_grp_B" (في مناطق مختلفة من الـ chip).

```c
// من pinctrl.h
struct pinfunction {
    const char *name;               // "spi0"، "i2c0"، "uart2"
    const char * const *groups;     // الـ groups الممكنة لهذه الـ function
    size_t ngroups;
    unsigned long flags;            // PINFUNCTION_FLAG_GPIO لو كانت GPIO function
};

// مثال: spi0 ممكن تشتغل على groupين مختلفين
static const char * const spi0_groups[] = { "spi0_grp_A", "spi0_grp_B" };

static const struct pinfunction my_functions[] = {
    PINCTRL_PINFUNCTION("spi0", spi0_groups, ARRAY_SIZE(spi0_groups)),
    PINCTRL_PINFUNCTION("i2c0", i2c0_groups, ARRAY_SIZE(i2c0_groups)),
};
```

#### 4. Mapping Table — قلب الـ Board Configuration

الـ `pinctrl_map` هي اللي بتربط بين الـ abstract world (device + state) والـ concrete world (pin controller + function + group). ده بيتعرف في الـ board file أو الـ device tree.

```c
// من machine.h
struct pinctrl_map {
    const char *dev_name;       // "spi0" — اسم الـ device المحتاج
    const char *name;           // "default" — اسم الـ state
    enum pinctrl_map_type type; // MUX_GROUP أو CONFIGS_PIN أو ...
    const char *ctrl_dev_name;  // "pinctrl-foo" — اسم الـ pin controller
    union {
        struct pinctrl_map_mux mux;         // function + group
        struct pinctrl_map_configs configs; // pin/group configs
    } data;
};
```

الـ `pinctrl_map_type` عندها 4 أنواع:

| النوع | الوظيفة |
|---|---|
| `PIN_MAP_TYPE_MUX_GROUP` | اختيار function على group من الـ pins |
| `PIN_MAP_TYPE_CONFIGS_PIN` | ضبط خصائص pin واحد (pull, drive) |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | ضبط خصائص group كاملة |
| `PIN_MAP_TYPE_DUMMY_STATE` | state فاضي (للـ hardware اللي مش محتاج config) |

#### 5. State

كل device عنده مجموعة **named states** زي "default" و"sleep". الـ framework بيدير الانتقال بينهم:

```c
// من pinctrl-state.h
#define PINCTRL_STATE_DEFAULT "default"  // التشغيل الاعتيادي
#define PINCTRL_STATE_INIT    "init"     // قبل probe() لو بيعمل glitch
#define PINCTRL_STATE_IDLE    "idle"     // pm_runtime_suspend
#define PINCTRL_STATE_SLEEP   "sleep"    // suspend() الكامل
```

---

### العلاقة بين الـ Structs

```
pinctrl_desc
    │
    ├── pins[]  ──────────────► pinctrl_pin_desc[]
    │                               (number, name, drv_data)
    │
    ├── pctlops ──────────────► pinctrl_ops
    │                               get_groups_count()
    │                               get_group_name()
    │                               get_group_pins()
    │                               dt_node_to_map()
    │
    ├── pmxops  ──────────────► pinmux_ops
    │                               get_functions_count()
    │                               get_function_name()
    │                               get_function_groups()
    │                               set_mux()          ← الأهم
    │                               gpio_request_enable()
    │                               strict (bool)
    │
    └── confops ──────────────► pinconf_ops
                                    pin_config_get()
                                    pin_config_set()
                                    pin_config_group_set()


pinctrl_map (board/DT level)
    │
    ├── dev_name  ──► "spi0"
    ├── name      ──► "default"
    ├── type      ──► PIN_MAP_TYPE_MUX_GROUP
    ├── ctrl_dev_name ──► "pinctrl-stm32"
    └── data.mux
            ├── function ──► "spi0"    (يتحول لـ index في foo_functions[])
            └── group    ──► "spi0_grp" (يتحول لـ index في foo_groups[])
```

---

### الـ Three Vtables — إيه اللي بيعمله كل واحد

#### pinctrl_ops — الـ Catalog

ده مش بيعمل حاجة في الـ hardware. هو بس بيخبر الـ core "إيه الـ groups الموجودة وإيه اللي فيها من pins". الـ core بيقرأه عشان يبني الـ database الداخلية.

```c
static struct pinctrl_ops foo_pctrl_ops = {
    .get_groups_count = foo_get_groups_count,  // كام group عندك؟
    .get_group_name   = foo_get_group_name,    // اسم group رقم N؟
    .get_group_pins   = foo_get_group_pins,    // pins الـ group دي؟
    .dt_node_to_map   = foo_dt_node_to_map,   // parse DT nodes (optional)
};
```

#### pinmux_ops — الـ Switcher

ده هو اللي بيتلمس الـ hardware فعلاً. لما الـ core يقرر إن device X يستخدم function Y على group Z، بيكل `set_mux()` للـ driver.

```c
static int foo_set_mux(struct pinctrl_dev *pctldev,
                       unsigned int func_selector,   // index في foo_functions[]
                       unsigned int group_selector)  // index في foo_groups[]
{
    // الـ core ضمن إنه مفيش conflict قبل ما يوصلنا هنا
    u8 regbit = BIT(group_selector);
    writeb((readb(MUX_REG) | regbit), MUX_REG);
    return 0;
}
```

الـ flag المهم `strict`:
- `strict = true` → الـ core بيمنع إن نفس الـ pin يتستخدم كـ GPIO وفي نفس الوقت كـ mux function.
- `strict = false` → ممكن يشتغلوا مع بعض (بعض الـ hardware بتسمح بده).

#### pinconf_ops — الـ Electrician

ده بيتحكم في الخصائص الكهربية للـ pin: pull-up، pull-down، drive strength، open-drain، slew rate. مستقل عن الـ mux.

```c
static int foo_pin_config_set(struct pinctrl_dev *pctldev,
                              unsigned int pin,
                              unsigned long *configs,  // مصفوفة configs
                              unsigned int num_configs)
{
    int i;
    for (i = 0; i < num_configs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        u32 arg = pinconf_to_config_argument(configs[i]);

        switch (param) {
        case PIN_CONFIG_BIAS_PULL_UP:
            /* ضبط pull-up register */
            break;
        case PIN_CONFIG_DRIVE_STRENGTH:
            /* ضبط drive strength register بالـ arg */
            break;
        }
    }
    return 0;
}
```

---

### Registration Flow — ازاي بيشتغل الـ Driver

```
1. Pin Controller Driver Init
         │
         ▼
   pinctrl_register_and_init(&desc, dev, drv_data, &pctldev)
         │
         ▼
   [Core] builds internal pin database
   [Core] validates ops vtables
         │
         ▼
   pinctrl_enable(pctldev)
         │
         ▼
   [Core] processes "hogged" mappings
   (mappings where dev_name == ctrl_dev_name)
```

الـ **hogging** مفهوم مهم: لو في mapping table entry اسم الـ device فيها نفس اسم الـ pin controller نفسه، الـ core بيفعلها automatically لما الـ controller يتسجل. ده مفيد لـ pins اللي لازم تتضبط من أول boot مش بس لما device معين يـprobe.

---

### Consumer API — ازاي الـ Driver بيطلب Pins

```
devm_pinctrl_get(dev)
    │  قراية الـ mapping table، تخصيص struct pinctrl
    ▼
pinctrl_lookup_state(p, "default")
    │  ابحث عن entries في الـ mapping table بنفس الـ state name
    ▼
pinctrl_select_state(p, state)
    │  verify no conflicts
    │  call driver's set_mux() and pin_config_set()
    ▼
  Hardware configured ✓
```

```c
// مثال كامل في driver probe
static int my_spi_probe(struct platform_device *pdev)
{
    struct pinctrl *pctl;
    struct pinctrl_state *default_state, *sleep_state;

    /* get handle — بيـparse الـ mapping table */
    pctl = devm_pinctrl_get(&pdev->dev);
    if (IS_ERR(pctl))
        return PTR_ERR(pctl);

    /* lookup state — بيدور على "default" في الـ mappings */
    default_state = pinctrl_lookup_state(pctl, PINCTRL_STATE_DEFAULT);
    if (IS_ERR(default_state))
        return PTR_ERR(default_state);

    /* select state — بيكتب في الـ hardware registers */
    return pinctrl_select_state(pctl, default_state);
}
```

ملاحظة مهمة: الـ device core نفسه بيعمل ده automatically لو الـ DT فيه `pinctrl-0 = <...>` — الـ driver مش محتاج يعمل حاجة خالص في الحالة دي.

---

### التكامل مع الـ GPIO Subsystem

ده من أكثر الأجزاء اللي بتحصل فيها confusion. الـ subsystem المهم اللي لازم تعرفه هنا: **GPIO subsystem** — بيوفر `gpiod_get()` / `gpiod_set_value()` وإلخ للتعامل مع pins كـ 1-bit I/O.

العلاقة بين pinctrl وGPIO بتحصل عبر الـ `pinctrl_gpio_range`:

```c
// من pinctrl.h
struct pinctrl_gpio_range {
    struct list_head node;
    const char *name;
    unsigned int id;
    unsigned int base;      // بداية الـ GPIO numbers
    unsigned int pin_base;  // بداية الـ pin numbers المقابلة
    unsigned int npins;
    unsigned int const *pins; // أو مصفوفة غير خطية
    struct gpio_chip *gc;
};
```

الـ mapping بين GPIO number وpin number مهم لأن الـ pin controller بيتكلم بـ "pin numbers" والـ GPIO subsystem بيتكلم بـ "GPIO numbers" — وهما ممكن يكونوا مختلفين.

```
GPIO number 48  ──► pinctrl_gpio_range lookup ──► pin number 64
                                                        │
                                                        ▼
                                               pinmux_ops.gpio_request_enable(pin=64)
```

#### GPIO Mode Pitfall

المهندسين بيسموا في الـ datasheet بعض الـ pin configurations "GPIO mode" وده بيخلق confusion. الـ kernel approach الصح:

- لو الـ pin هيتستخدم كـ 1-bit I/O من kernel code → استخدم `gpiod_get()` من الـ GPIO subsystem.
- لو الـ pin بس محتاج حالة كهربية معينة (output low أثناء sleep مثلاً) → استخدم `PIN_CONFIG_LEVEL` في الـ pinctrl mapping table مباشرة، من غير ما تلمس الـ GPIO API.

```c
// الطريقة الصح لـ UART TX pin أثناء sleep
static unsigned long uart_sleep_mode[] = {
    PIN_CONF_PACKED(PIN_CONFIG_LEVEL, 0),  // output low — مش GPIO
};

static struct pinctrl_map uart_map[] __initdata = {
    PIN_MAP_MUX_GROUP("uart0", PINCTRL_STATE_DEFAULT,
                      "pinctrl-foo", "uart_grp", "uart0"),
    PIN_MAP_MUX_GROUP("uart0", PINCTRL_STATE_SLEEP,
                      "pinctrl-foo", "uart_grp", "gpio-mode"),  // function اسمها "gpio-mode" في الـ driver
    PIN_MAP_CONFIGS_PIN("uart0", PINCTRL_STATE_SLEEP,
                        "pinctrl-foo", "UART_TX", uart_sleep_mode),
};
```

---

### ما يملكه الـ pinctrl Core مقابل ما يفوضه للـ Driver

| المسؤولية | الـ pinctrl Core | الـ Pin Controller Driver |
|---|---|---|
| تتبع مين شاغل الـ pin | ✓ | ✗ |
| منع الـ conflicts | ✓ | ✗ |
| إدارة الـ states machine | ✓ | ✗ |
| Parse الـ mapping table | ✓ | ✗ |
| كتابة MUX registers | ✗ | ✓ (`set_mux()`) |
| كتابة PULL/DRIVE registers | ✗ | ✓ (`pin_config_set()`) |
| تعريف الـ pin groups | ✗ | ✓ (`pinctrl_ops`) |
| تعريف الـ functions | ✗ | ✓ (`pinmux_ops`) |
| معرفة register layout للـ SoC | ✗ | ✓ |
| DT parsing لـ vendor-specific nodes | ✗ | ✓ (`dt_node_to_map()`) |

---

### Debugfs — أدوات التشخيص

```bash
# شوف كل الـ pin controllers الموجودة
cat /sys/kernel/debug/pinctrl/pinctrl-devices

# شوف كل الـ pins وحالتهم
cat /sys/kernel/debug/pinctrl/pinctrl-foo/pins

# شوف مين شاغل إيه من الـ pins
cat /sys/kernel/debug/pinctrl/pinctrl-foo/pinmux-pins

# شوف الـ mappings المسجلة
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# فعّل function يدوياً (للـ testing)
echo "spi0_grp spi0" > /sys/kernel/debug/pinctrl/pinctrl-foo/pinmux-select
```

---

### Runtime Pinmuxing

في بعض الحالات، device بيحتاج يغير الـ mux أثناء الـ runtime. مثلاً SPI controller موجود على SoC لكن ممكن يشتغل على pin set A أو pin set B حسب الـ board layout:

```c
struct pinctrl *p;
struct pinctrl_state *s_pos_a, *s_pos_b;

/* في probe */
p = devm_pinctrl_get(&device);
s_pos_a = pinctrl_lookup_state(p, "pos-A");
s_pos_b = pinctrl_lookup_state(p, "pos-B");

/* Switch at runtime — لازم يتعمل من process context */
pinctrl_select_state(p, s_pos_a);  /* SPI على pins A8-A5 */
/* ... */
pinctrl_select_state(p, s_pos_b);  /* SPI على pins G4-G1 */
```

الـ core بيـdeactivate الـ state القديمة قبل ما يـactivate الجديدة — تحرير الـ pins القديمة وأخذ الجديدة مع التحقق من الـ conflicts.

---

### ملخص المفاهيم الأساسية

```
Physical Pin
    │
    └── موصوف في pinctrl_pin_desc (رقم + اسم)
    │
    └── بيتجمع في pingroup (مجموعة بتخدم function واحدة)
    │
    └── الـ group بيتربط بـ function في pinfunction
    │
    └── الـ function + group + device بيتربطوا في pinctrl_map
    │
    └── الـ map بيتقرأ لما consumer يعمل devm_pinctrl_get()
    │
    └── لما يعمل pinctrl_select_state() → core يتحقق من conflicts
                                          → بعدين يكل الـ driver
                                          → driver يكتب في الـ registers
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Flags والـ Config Options

#### `enum pinctrl_map_type` — نوع الـ mapping entry

| القيمة | المعنى |
|---|---|
| `PIN_MAP_TYPE_INVALID` | قيمة غير صالحة / خطأ |
| `PIN_MAP_TYPE_DUMMY_STATE` | state فاضي بدون أي تأثير على الـ hardware |
| `PIN_MAP_TYPE_MUX_GROUP` | تفعيل function معينة على group من الـ pins |
| `PIN_MAP_TYPE_CONFIGS_PIN` | ضبط إعدادات كهربية على pin واحد |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | ضبط إعدادات كهربية على group كاملة |

---

#### `enum pin_config_param` — معاملات الإعداد الكهربي للـ pin

| المعامل | الوحدة / القيمة | المعنى |
|---|---|---|
| `PIN_CONFIG_BIAS_DISABLE` | — | إيقاف كل الـ bias |
| `PIN_CONFIG_BIAS_PULL_UP` | Ohms | pull-up لـ VDD |
| `PIN_CONFIG_BIAS_PULL_DOWN` | Ohms | pull-down لـ GND |
| `PIN_CONFIG_BIAS_HIGH_IMPEDANCE` | — | tristate / Hi-Z |
| `PIN_CONFIG_BIAS_BUS_HOLD` | — | bus keeper / repeater |
| `PIN_CONFIG_BIAS_PULL_PIN_DEFAULT` | 0/1 | pull حسب الـ hardware default |
| `PIN_CONFIG_DRIVE_PUSH_PULL` | — | الـ output الاعتيادي (active high & low) |
| `PIN_CONFIG_DRIVE_OPEN_DRAIN` | — | open drain (open collector) |
| `PIN_CONFIG_DRIVE_OPEN_SOURCE` | — | open source (open emitter) |
| `PIN_CONFIG_DRIVE_STRENGTH` | mA | أقصى تيار يمكن للـ pin أن يدفعه أو يسحبه |
| `PIN_CONFIG_DRIVE_STRENGTH_UA` | µA | نفس السابق لكن بدقة أعلى |
| `PIN_CONFIG_INPUT_DEBOUNCE` | µs | وقت الـ debounce عند القراءة |
| `PIN_CONFIG_INPUT_ENABLE` | 0/1 | تفعيل/تعطيل الـ input buffer |
| `PIN_CONFIG_INPUT_SCHMITT_ENABLE` | 0/1 | تفعيل Schmitt trigger |
| `PIN_CONFIG_INPUT_SCHMITT_UV` | µV | عتبة الـ Schmitt trigger |
| `PIN_CONFIG_LEVEL` | 0/1 | drive الـ pin low أو high (بديل GPIO في sleep) |
| `PIN_CONFIG_OUTPUT_ENABLE` | 0/1 | تفعيل output buffer بدون drive قيمة |
| `PIN_CONFIG_OUTPUT_IMPEDANCE_OHMS` | Ohms | impedance للـ output |
| `PIN_CONFIG_MODE_LOW_POWER` | 0/1 | وضع توفير الطاقة |
| `PIN_CONFIG_MODE_PWM` | — | وضع PWM |
| `PIN_CONFIG_SLEW_RATE` | custom | سرعة تغيير إشارة الـ output |
| `PIN_CONFIG_SKEW_DELAY` | custom | تأخير الـ clock skew أو latch |
| `PIN_CONFIG_SKEW_DELAY_INPUT_PS` | ps | تأخير الـ input فقط |
| `PIN_CONFIG_SKEW_DELAY_OUTPUT_PS` | ps | تأخير الـ output فقط |
| `PIN_CONFIG_PERSIST_STATE` | — | حفظ الإعداد عبر الـ sleep أو reset |
| `PIN_CONFIG_POWER_SOURCE` | custom | اختيار مصدر الطاقة للـ pin |
| `PIN_CONFIG_SLEEP_HARDWARE_STATE` | — | علامة أن الـ state ده للـ sleep |
| `PIN_CONFIG_END` | 0x7F | آخر قيمة للـ standard params |
| `PIN_CONFIG_MAX` | 0xFF | أقصى قيمة في الـ packed format |

> **الـ packed format:** `PIN_CONF_PACKED(param, arg)` → الـ param في bits 0-7، الـ argument في bits 8-31.

---

#### Kconfig Options المهمة

| الـ Option | الأثر |
|---|---|
| `CONFIG_PINCTRL` | تفعيل الـ subsystem كله |
| `CONFIG_GENERIC_PINCONF` | استخدام الـ generic pinconf parser مع الـ device tree |
| `CONFIG_PINMUX` | تفعيل الـ pinmux (function/group switching) |
| `CONFIG_PINCONF` | تفعيل الـ pin configuration |
| `CONFIG_PM` | يضيف `sleep_state` و`idle_state` في الـ `dev_pin_info` |
| `CONFIG_OF` | يفعّل `of_pinctrl_get()` للقراءة من الـ device tree |
| `CONFIG_DEBUG_FS` | يفعّل الملفات تحت `/sys/kernel/debug/pinctrl` |

---

#### Standard State Names

| الـ State | متى يُستخدم |
|---|---|
| `PINCTRL_STATE_DEFAULT` = `"default"` | الحالة الطبيعية أثناء التشغيل، يُفعَّل قبل `probe()` |
| `PINCTRL_STATE_INIT` = `"init"` | قبل `probe()` في الحالات الحرجة (تجنب glitch) |
| `PINCTRL_STATE_IDLE` = `"idle"` | runtime suspend / pm_runtime_idle |
| `PINCTRL_STATE_SLEEP` = `"sleep"` | suspend عميق (ordinary `.suspend()`) |

---

### الـ Structs الأساسية

---

#### `struct pinctrl_pin_desc`

**الغرض:** وصف pin واحد فيزيائي داخل الـ controller.

```c
struct pinctrl_pin_desc {
    unsigned int number;   /* رقم الـ pin في الـ namespace المحلي للـ controller */
    const char *name;      /* اسم الـ pin مثل "A8" أو "UART_TX" */
    void *drv_data;        /* بيانات خاصة بالـ driver، الـ core لا يمسها */
};
```

- يُنشأ عادةً في array ثابت `const struct pinctrl_pin_desc foo_pins[]`
- **الـ macro:** `PINCTRL_PIN(number, name)` و`PINCTRL_PIN_ANON(number)`
- يُشار إليه من `pinctrl_desc.pins`

---

#### `struct pingroup`

**الغرض:** تجميع عدة pins في وحدة منطقية واحدة تُدار معاً (مثلاً كل pins الـ SPI).

```c
struct pingroup {
    const char *name;           /* اسم الـ group مثل "spi0_0_grp" */
    const unsigned int *pins;   /* array بأرقام الـ pins */
    size_t npins;               /* عدد الـ pins في الـ group */
};
```

- **الـ macro:** `PINCTRL_PINGROUP(name, pins, npins)`
- يُستخدم من `pinctrl_ops.get_group_pins()` لإرجاع الـ pins
- أسماء الـ groups يجب أن تكون **unique** داخل نفس الـ controller

---

#### `struct pinfunction`

**الغرض:** وصف function يمكن تفعيلها على مجموعة من الـ groups (مثلاً `spi0` تشتغل على `spi0_0_grp` أو `spi0_1_grp`).

```c
struct pinfunction {
    const char *name;               /* اسم الـ function مثل "spi0" أو "i2c0" */
    const char * const *groups;     /* array بأسماء الـ groups المتوافقة */
    size_t ngroups;                 /* عدد الـ groups */
    unsigned long flags;            /* PINFUNCTION_FLAG_GPIO لو كانت GPIO function */
};
```

- **الـ macro:** `PINCTRL_PINFUNCTION(name, groups, ngroups)`
- **الـ macro الخاص بـ GPIO:** `PINCTRL_GPIO_PINFUNCTION(...)` يضبط `flags = PINFUNCTION_FLAG_GPIO`
- تربط الـ `pinmux_ops.get_function_groups()` الـ function بالـ groups الخاصة بها

---

#### `struct pinctrl_gpio_range`

**الغرض:** ربط نطاق من GPIO numbers بنطاق من pins في الـ controller، لأن الـ GPIO subsystem وpin control subsystem عندهم namespaces مختلفة.

```c
struct pinctrl_gpio_range {
    struct list_head node;      /* عقدة في list الـ gpio_ranges للـ controller */
    const char *name;           /* اسم الـ GPIO chip */
    unsigned int id;            /* ID للـ chip */
    unsigned int base;          /* أول GPIO number في هذا النطاق */
    unsigned int pin_base;      /* أول pin number مقابل (لو linear mapping) */
    unsigned int npins;         /* عدد الـ pins */
    unsigned int const *pins;   /* array بأرقام الـ pins (لو sparse mapping) */
    struct gpio_chip *gc;       /* pointer للـ gpio_chip المرتبط */
};
```

- لو الـ mapping linear → يستخدم `pin_base`، والـ `pins` تبقى NULL
- لو الـ mapping sparse → يستخدم `pins` array، ويُهمل `pin_base`
- يُضاف للـ controller عبر `pinctrl_add_gpio_range(pctl, &range)`

---

#### `struct pinctrl_ops`

**الغرض:** vtable للعمليات العامة على الـ controller (enumeration الـ groups والـ device tree parsing).

```c
struct pinctrl_ops {
    /* إجباري: إرجاع عدد الـ groups */
    int (*get_groups_count)(struct pinctrl_dev *pctldev);

    /* إجباري: إرجاع اسم الـ group بالـ selector */
    const char *(*get_group_name)(struct pinctrl_dev *pctldev, unsigned int selector);

    /* إجباري: إرجاع array الـ pins وعددها للـ group */
    int (*get_group_pins)(struct pinctrl_dev *pctldev, unsigned int selector,
                          const unsigned int **pins, unsigned int *num_pins);

    /* اختياري: عرض info في debugfs */
    void (*pin_dbg_show)(struct pinctrl_dev *pctldev, struct seq_file *s,
                         unsigned int offset);

    /* اختياري: تحويل device tree node لـ mapping entries */
    int (*dt_node_to_map)(struct pinctrl_dev *pctldev, struct device_node *np_config,
                          struct pinctrl_map **map, unsigned int *num_maps);

    /* اختياري: تحرير الـ mapping entries اللي أنشأها dt_node_to_map */
    void (*dt_free_map)(struct pinctrl_dev *pctldev,
                        struct pinctrl_map *map, unsigned int num_maps);
};
```

---

#### `struct pinmux_ops`

**الغرض:** vtable لعمليات الـ multiplexing — تحديد الـ functions وتفعيلها على الـ pins.

```c
struct pinmux_ops {
    /* اختياري: التحقق من إمكانية استخدام pin للـ mux */
    int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);
    int (*free)(struct pinctrl_dev *pctldev, unsigned int offset);

    /* إجباري: enumeration الـ functions */
    int (*get_functions_count)(struct pinctrl_dev *pctldev);
    const char *(*get_function_name)(struct pinctrl_dev *pctldev, unsigned int selector);
    int (*get_function_groups)(struct pinctrl_dev *pctldev, unsigned int selector,
                               const char * const **groups, unsigned int *num_groups);

    /* اختياري: هل الـ function دي GPIO؟ */
    bool (*function_is_gpio)(struct pinctrl_dev *pctldev, unsigned int selector);

    /* إجباري: تفعيل الـ mux على الـ hardware */
    int (*set_mux)(struct pinctrl_dev *pctldev,
                   unsigned int func_selector, unsigned int group_selector);

    /* اختياري: تفعيل GPIO mode على pin واحد */
    int (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                               struct pinctrl_gpio_range *range, unsigned int offset);
    void (*gpio_disable_free)(struct pinctrl_dev *pctldev,
                              struct pinctrl_gpio_range *range, unsigned int offset);
    int (*gpio_set_direction)(struct pinctrl_dev *pctldev,
                              struct pinctrl_gpio_range *range,
                              unsigned int offset, bool input);

    /* منع استخدام نفس الـ pin من GPIO ومن mux في نفس الوقت */
    bool strict;
};
```

- **الـ `strict` flag:** لما يكون `true`، الـ core يرفض طلب أي pin يُستخدم فعلاً من GPIO أو من mux function أخرى، حتى لو الـ hardware يسمح بذلك عملياً (مثل مخطط B في الوثيقة)

---

#### `struct pinconf_ops`

**الغرض:** vtable لقراءة وكتابة الإعدادات الكهربية للـ pins والـ groups.

```c
struct pinconf_ops {
#ifdef CONFIG_GENERIC_PINCONF
    bool is_generic;  /* يستخدم الـ generic DT parser */
#endif

    /* قراءة/كتابة إعداد pin واحد */
    int (*pin_config_get)(struct pinctrl_dev *pctldev, unsigned int pin,
                          unsigned long *config);
    int (*pin_config_set)(struct pinctrl_dev *pctldev, unsigned int pin,
                          unsigned long *configs, unsigned int num_configs);

    /* قراءة/كتابة إعداد group كاملة */
    int (*pin_config_group_get)(struct pinctrl_dev *pctldev, unsigned int selector,
                                unsigned long *config);
    int (*pin_config_group_set)(struct pinctrl_dev *pctldev, unsigned int selector,
                                unsigned long *configs, unsigned int num_configs);

    /* debugfs hooks اختيارية */
    void (*pin_config_dbg_show)(...);
    void (*pin_config_group_dbg_show)(...);
    void (*pin_config_config_dbg_show)(...);
};
```

- كل `config` هو `unsigned long` بالـ packed format: `[arg:24][param:8]`
- الـ `pin_config_set` بتاخد array من الـ configs بدل واحد، عشان ممكن تضبط أكثر من إعداد في نفس الوقت

---

#### `struct pinctrl_desc`

**الغرض:** الـ descriptor الرئيسي اللي يسجّله الـ driver في الـ core — يضم كل مكونات الـ controller.

```c
struct pinctrl_desc {
    const char *name;                          /* اسم الـ controller مثل "pinctrl-foo" */
    const struct pinctrl_pin_desc *pins;       /* array الـ pins */
    unsigned int npins;                        /* عدد الـ pins */
    const struct pinctrl_ops *pctlops;         /* vtable العمليات العامة */
    const struct pinmux_ops *pmxops;           /* vtable الـ pinmux */
    const struct pinconf_ops *confops;         /* vtable الـ pinconf */
    struct module *owner;                      /* للـ refcounting */
#ifdef CONFIG_GENERIC_PINCONF
    unsigned int num_custom_params;            /* عدد الـ custom params */
    const struct pinconf_generic_params *custom_params; /* تعريف الـ custom params */
    const struct pin_config_item *custom_conf_items;    /* طريقة عرضها في debugfs */
#endif
    bool link_consumers; /* ينشئ device link بين الـ controller والـ consumers */
};
```

- يُمرَّر لـ `pinctrl_register_and_init()` أو `devm_pinctrl_register_and_init()`
- بعد الـ register، يُستدعى `pinctrl_enable()` لتفعيل الـ hogging وغيره

---

#### `struct pinctrl_map`

**الغرض:** إدخال في جدول الـ mapping — يصف كيف يُوصّل device معين بـ function ومجموعة pins في controller محدد.

```c
struct pinctrl_map {
    const char *dev_name;      /* اسم الـ device consumer مثل "foo-spi.0" */
    const char *name;          /* اسم الـ state مثل "default" أو "sleep" */
    enum pinctrl_map_type type; /* نوع الـ entry */
    const char *ctrl_dev_name; /* اسم الـ pin controller */
    union {
        struct pinctrl_map_mux mux;         /* لو type == MUX_GROUP */
        struct pinctrl_map_configs configs; /* لو type == CONFIGS_* */
    } data;
};
```

**الـ sub-structs:**

```c
struct pinctrl_map_mux {
    const char *group;    /* اسم الـ group (ممكن NULL لاختيار أول group) */
    const char *function; /* اسم الـ function */
};

struct pinctrl_map_configs {
    const char *group_or_pin; /* اسم الـ pin أو الـ group */
    unsigned long *configs;   /* array قيم الإعدادات (packed format) */
    unsigned int num_configs; /* عدد الإعدادات */
};
```

---

#### `struct pinctrl` (opaque)

**الغرض:** handle لـ consumer (device) يمثّل كل الـ pin control state لهذا الـ device. الـ core ينشئه ويديره داخلياً.

- يُحصل عليه من `devm_pinctrl_get(dev)` أو `pinctrl_get(dev)`
- يحتوي داخلياً على list من الـ `pinctrl_state` المرتبطة بالـ device
- **يُعامَل كـ cookie** — الـ driver لا يرى تفاصيله

---

#### `struct pinctrl_state` (opaque)

**الغرض:** يمثل state واحد (مثلاً "default") بما فيه من settings. يُحصل عليه من `pinctrl_lookup_state(p, name)`.

---

#### `struct dev_pin_info`

**الغرض:** يُخزَّن في `struct device` كـ `dev->pins`، ويحتفظ بالـ handles للـ states المعيارية الأربعة.

```c
struct dev_pin_info {
    struct pinctrl *p;                  /* الـ handle الرئيسي */
    struct pinctrl_state *default_state;
    struct pinctrl_state *init_state;
#ifdef CONFIG_PM
    struct pinctrl_state *sleep_state;
    struct pinctrl_state *idle_state;
#endif
};
```

- يُنشَأ تلقائياً من الـ device core عبر `pinctrl_bind_pins()` قبل الـ probe

---

#### `struct pin_config_item` و `struct pinconf_generic_params`

| الـ Struct | الغرض |
|---|---|
| `pin_config_item` | يربط `pin_config_param` بنص عرض لـ debugfs |
| `pinconf_generic_params` | يربط DT property name بـ `pin_config_param` وقيمة default |

---

### مخططات علاقات الـ Structs

```
                    ┌─────────────────────────────────────────────────┐
                    │              pinctrl_desc                       │
                    │  .name                                          │
                    │  .pins ──────────────► pinctrl_pin_desc[]       │
                    │  .npins                                         │
                    │  .pctlops ───────────► pinctrl_ops              │
                    │  .pmxops ────────────► pinmux_ops               │
                    │  .confops ───────────► pinconf_ops              │
                    │  .owner                                         │
                    └────────────┬────────────────────────────────────┘
                                 │ pinctrl_register_and_init()
                                 ▼
                    ┌────────────────────────┐
                    │     pinctrl_dev        │  ← الـ core ينشئه ويديره
                    │  (internal to core)    │
                    │  .desc ─────────────── │──► pinctrl_desc
                    │  .gpio_ranges (list) ──│──► pinctrl_gpio_range[]
                    │  .pin_desc_tree ────── │──► radix tree of pins
                    └────────────────────────┘
                             ▲
                             │ pctldev
                    ┌────────┴──────────┐
                    │   pinctrl_ops     │   ← driver implements
                    │   pinmux_ops      │
                    │   pinconf_ops     │
                    └───────────────────┘

    ═══ consumer side ═══════════════════════════════════════════════

    ┌──────────────┐    pinctrl_register_mappings()
    │ pinctrl_map[]│────────────────────────────────► mapping table
    │  .dev_name   │                                  (in core memory)
    │  .name       │
    │  .type       │
    │  .data.mux   │──► pinctrl_map_mux
    │  .data.cfg   │──► pinctrl_map_configs
    └──────────────┘

    struct device
    └── dev->pins ──► dev_pin_info
                       ├── .p ──────────────► pinctrl (handle)
                       │                        └── states list
                       ├── .default_state ──► pinctrl_state
                       ├── .init_state ─────► pinctrl_state
                       ├── .sleep_state ────► pinctrl_state
                       └── .idle_state ─────► pinctrl_state

    ═══ GPIO bridge ═════════════════════════════════════════════════

    gpio_chip
    └── (gc pointer) ◄── pinctrl_gpio_range
                           ├── .base      (GPIO number space)
                           ├── .pin_base  (pin number space, linear)
                           ├── .pins[]    (pin number space, sparse)
                           └── .node ◄──── list in pinctrl_dev
```

---

### مخطط دورة حياة الـ Pin Controller Driver

```
┌─────────────────────────────────────────────────────────────┐
│                  DRIVER INIT / PROBE                        │
│                                                             │
│  1. define pinctrl_pin_desc[]  (static array)               │
│  2. define pingroup[]          (static array)               │
│  3. define pinfunction[]       (static array)               │
│  4. implement pinctrl_ops, pinmux_ops, pinconf_ops          │
│  5. fill pinctrl_desc                                       │
│  6. pinctrl_register_and_init(&desc, dev, drv_data, &pctl)  │
│       → core allocates pinctrl_dev                          │
│       → core builds pin radix tree                          │
│       → core hogging: looks for dev_name == ctrl_dev_name   │
│  7. pinctrl_enable(pctl)                                     │
│       → applies hogged mappings immediately                  │
│  8. [optional] pinctrl_add_gpio_range(pctl, &range)         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  RUNTIME (consumer usage)                   │
│                                                             │
│  devm_pinctrl_get(dev)                                      │
│       → allocates struct pinctrl, parses mapping table      │
│  pinctrl_lookup_state(p, "default")                         │
│       → returns struct pinctrl_state handle                 │
│  pinctrl_select_state(p, s)                                 │
│       → calls pmxops->set_mux() per mux entry               │
│       → calls confops->pin_config_set() per config entry    │
│  [PM suspend]                                               │
│       → pinctrl_pm_select_sleep_state(dev)                  │
│  [PM resume]                                                │
│       → pinctrl_pm_select_default_state(dev)                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  TEARDOWN                                   │
│                                                             │
│  devm_pinctrl_put(p)  or  auto via devm on device remove    │
│       → releases all pin reservations                        │
│  pinctrl_unregister(pctl)  /  devm auto on driver remove    │
│       → removes pinctrl_dev from global list                 │
│       → frees gpio_ranges, pin tree, etc.                   │
└─────────────────────────────────────────────────────────────┘
```

---

### مخططات تدفق الاستدعاءات

#### تسجيل الـ Controller

```
driver probe()
  └─► pinctrl_register_and_init(&desc, dev, drv_data, &pctl)
        ├─► allocate struct pinctrl_dev
        ├─► copy desc → pctldev->desc
        ├─► build radix_tree for each pin in desc.pins[]
        ├─► validate pmxops, confops callbacks
        ├─► register pctldev in global list
        └─► return 0 (pctl set)

  └─► pinctrl_enable(pctl)
        └─► pinctrl_claim_hogs(pctldev)
              └─► for each map where map.dev_name == pctldev.dev_name:
                    └─► pinctrl_get() → pinctrl_lookup_state()
                          └─► pinctrl_select_state()
```

---

#### Consumer يطلب state

```
device driver foo_probe()
  └─► devm_pinctrl_get(&dev)
        └─► pinctrl_get(dev)
              ├─► allocate struct pinctrl
              ├─► scan mapping table for entries matching dev_name
              └─► return pinctrl handle p

  └─► pinctrl_lookup_state(p, "default")
        ├─► scan p->states list for name == "default"
        └─► return pinctrl_state handle s

  └─► pinctrl_select_state(p, s)
        ├─► for each MUX_GROUP setting in state s:
        │     ├─► core checks no pin collision (ownership tracking)
        │     ├─► pmxops->request(pctldev, pin)     [optional]
        │     └─► pmxops->set_mux(pctldev, func_sel, grp_sel)
        │             └─► driver writes MUX register
        └─► for each CONFIGS setting in state s:
              └─► confops->pin_config_set(pctldev, pin, configs, num)
                      └─► driver writes config registers
```

---

#### GPIO Subsystem يطلب pin

```
gpiod_get(dev, "foo")  [GPIO consumer]
  └─► gpio_chip.request(gc, offset)
        └─► pinctrl_gpio_request(gc, offset)
              ├─► find pinctrl_dev owning this gpio range
              ├─► find pin number from gpio offset
              ├─► check no mux_owner conflict (if strict=true)
              └─► pmxops->gpio_request_enable(pctldev, range, offset)
                      └─► driver enables GPIO mux on that pin

gpiod_direction_output(desc, value)
  └─► gpio_chip.direction_output(gc, offset, value)
        └─► pinctrl_gpio_direction_output(gc, offset)
              └─► pmxops->gpio_set_direction(pctldev, range, offset, false)
                      └─► driver sets direction register
```

---

#### تغيير الـ state أثناء الـ Runtime (Runtime Pinmuxing)

```
foo_switch()
  └─► pinctrl_select_state(p, s1)   /* position A */
        ├─► deactivate current mux settings
        │     └─► pmxops->free() for pins leaving
        └─► activate new mux settings
              └─► pmxops->set_mux(pctldev, func_spi0, grp_spi0_0)
                    └─► writeb(BIT(grp_spi0_0), MUX_REG)

  └─► pinctrl_select_state(p, s2)   /* position B */
        ├─► deactivate s1 settings
        └─► pmxops->set_mux(pctldev, func_spi0, grp_spi0_1)
              └─► writeb(BIT(grp_spi0_1), MUX_REG)
```

---

#### PM Suspend/Resume

```
foo_suspend()
  └─► pinctrl_pm_select_sleep_state(dev)
        └─► pinctrl_select_state(dev->pins->p, dev->pins->sleep_state)
              └─► [apply sleep mux + config settings]

foo_resume()
  └─► pinctrl_pm_select_init_state(dev)      /* اختياري */
        └─► pinctrl_select_state(..., init_state)

  └─► [driver resumes hardware]

  └─► pinctrl_pm_select_default_state(dev)
        └─► pinctrl_select_state(..., default_state)
```

---

### استراتيجية الـ Locking

الـ pinctrl core يستخدم **mutex** داخلي (`pinctrl_mutex` أو per-device mutex) لحماية:

| ما يُحمى | الـ Lock |
|---|---|
| Global mapping table | `pinctrl_maps_mutex` |
| الـ `pinctrl_dev` وkل قوائمه الداخلية (gpio_ranges, pin ownership) | `pinctrl_dev->mutex` |
| تغيير حالة الـ ownership عند `set_mux` و`pin_config_set` | نفس الـ mutex أعلاه |
| الـ `pinctrl` handle الخاص بالـ consumer وlist الـ states | يُستخدم في process context فقط |

**قواعد الـ locking:**

1. **`pinctrl_get()` و`pinctrl_lookup_state()`** → process context فقط، ممكن تستغرق وقت (parsing).
2. **`pinctrl_select_state()`** → تقدر تتسمى non-blocking في معظم الحالات، **لكن** لو الـ controller على bus بطيء (I2C/SPI) → لازم process context.
3. **`pinctrl_gpio_request()`** → يُستدعى من `gpio_chip.request()` اللي ممكن تيجي من أي context، لكن الـ pinctrl core نفسه يأخذ mutex فمش atomic-safe بطبيعته — الـ gpiolib بتضمن الاتساق.
4. **ترتيب الـ acquire:** لو الـ driver محتاج يأخذ lock خاص بيه وlock الـ pinctrl core، لازم يأخذ lock بتاعه الأول لتجنب deadlock.
5. **الـ `strict` flag** → بيضيف فحص مزدوج: الـ core يتحقق من `mux_owner` و`gpio_owner` قبل ما يسمح بأي طلب جديد على الـ pin.

```
ownership model per pin:
┌──────────┬──────────────┬───────────────┐
│ pin state│ mux_owner    │ gpio_owner    │
├──────────┼──────────────┼───────────────┤
│ free     │ NULL         │ NULL          │
│ muxed    │ "foo-spi.0"  │ NULL          │
│ gpio     │ NULL         │ gpio_chip ptr │
│ conflict │ (blocked if strict=true)     │
└──────────┴──────────────┴───────────────┘
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet للـ API الكامل

#### Group 1: تسجيل الـ Pin Controller (Driver-side)

| Function | Header | الغرض |
|---|---|---|
| `pinctrl_register_and_init()` | `pinctrl.h` | تسجيل الـ controller + تهيئته بدون تفعيل |
| `pinctrl_enable()` | `pinctrl.h` | تفعيل الـ controller بعد التسجيل |
| `pinctrl_register()` | `pinctrl.h` | **deprecated** — تسجيل + تفعيل فوري |
| `pinctrl_unregister()` | `pinctrl.h` | إلغاء تسجيل الـ controller |
| `devm_pinctrl_register_and_init()` | `pinctrl.h` | نسخة devm من `pinctrl_register_and_init()` |
| `devm_pinctrl_register()` | `pinctrl.h` | **deprecated** — نسخة devm من `pinctrl_register()` |

#### Group 2: إدارة الـ GPIO Ranges

| Function | الغرض |
|---|---|
| `pinctrl_add_gpio_range()` | إضافة GPIO range واحد للـ controller |
| `pinctrl_add_gpio_ranges()` | إضافة مصفوفة من GPIO ranges |
| `pinctrl_remove_gpio_range()` | إزالة GPIO range |
| `pinctrl_find_and_add_gpio_range()` | بحث عن controller بالاسم وإضافة range |
| `pinctrl_find_gpio_range_from_pin()` | إيجاد الـ range الذي يحتوي pin معين |
| `pinctrl_get_group_pins()` | استرجاع الـ pins من group باسمه |

#### Group 3: Consumer API (Device Driver-side)

| Function | Header | الغرض |
|---|---|---|
| `pinctrl_get()` | `consumer.h` | الحصول على handle للـ pinctrl |
| `devm_pinctrl_get()` | `consumer.h` | نسخة devm — cleanup تلقائي |
| `pinctrl_put()` | `consumer.h` | تحرير الـ handle |
| `devm_pinctrl_put()` | `consumer.h` | explicit release للـ devm handle |
| `pinctrl_lookup_state()` | `consumer.h` | البحث عن state بالاسم |
| `pinctrl_select_state()` | `consumer.h` | تفعيل state معين |
| `pinctrl_select_default_state()` | `consumer.h` | تفعيل الـ default state مباشرة |
| `pinctrl_get_select()` | `consumer.h` | inline: get + lookup + select |
| `pinctrl_get_select_default()` | `consumer.h` | inline: get + select default |
| `devm_pinctrl_get_select()` | `consumer.h` | devm version من `pinctrl_get_select()` |
| `devm_pinctrl_get_select_default()` | `consumer.h` | devm version من `pinctrl_get_select_default()` |

#### Group 4: GPIO-Pinctrl Bridge (gpiolib فقط)

| Function | الغرض |
|---|---|
| `pinctrl_gpio_can_use_line()` | هل يمكن استخدام الـ GPIO line؟ |
| `pinctrl_gpio_request()` | طلب pin للاستخدام كـ GPIO |
| `pinctrl_gpio_free()` | تحرير الـ pin من GPIO mode |
| `pinctrl_gpio_direction_input()` | ضبط الاتجاه: input |
| `pinctrl_gpio_direction_output()` | ضبط الاتجاه: output |
| `pinctrl_gpio_set_config()` | تطبيق pin config على GPIO |

#### Group 5: PM State Helpers

| Function | الغرض |
|---|---|
| `pinctrl_pm_select_default_state()` | تفعيل "default" state في PM context |
| `pinctrl_pm_select_init_state()` | تفعيل "init" state |
| `pinctrl_pm_select_sleep_state()` | تفعيل "sleep" state |
| `pinctrl_pm_select_idle_state()` | تفعيل "idle" state |

#### Group 6: Mapping Table Registration

| Function | الغرض |
|---|---|
| `pinctrl_register_mappings()` | تسجيل mapping table في الـ core |
| `devm_pinctrl_register_mappings()` | نسخة devm |
| `pinctrl_unregister_mappings()` | إلغاء تسجيل mapping table |
| `pinctrl_provide_dummies()` | السماح بـ dummy states عند غياب الـ hardware |

#### Group 7: Generic Pinconf DT Helpers

| Function | الغرض |
|---|---|
| `pinconf_generic_dt_node_to_map()` | تحويل DT node لـ mapping entries |
| `pinconf_generic_dt_subnode_to_map()` | تحويل DT subnode |
| `pinconf_generic_dt_free_map()` | تحرير الـ map المُنشأ من DT |
| `pinconf_generic_dt_node_to_map_group()` | inline wrapper: type = GROUP |
| `pinconf_generic_dt_node_to_map_pin()` | inline wrapper: type = PIN |
| `pinconf_generic_dt_node_to_map_all()` | inline wrapper: type = INVALID (auto-detect) |

#### Group 8: Helper Inlines

| Function | الغرض |
|---|---|
| `pinconf_to_config_param()` | استخراج param من packed config |
| `pinconf_to_config_argument()` | استخراج argument من packed config |
| `pinconf_to_config_packed()` | تجميع param + argument في unsigned long |
| `dev_pinctrl()` | استرجاع الـ `struct pinctrl *` من `struct device` |

---

### الـ Callback Tables (vtables)

| Vtable Struct | يُسجَّل في | الغرض |
|---|---|---|
| `struct pinctrl_ops` | `pinctrl_desc.pctlops` | عمليات groups وDT parsing |
| `struct pinmux_ops` | `pinctrl_desc.pmxops` | عمليات الـ mux |
| `struct pinconf_ops` | `pinctrl_desc.confops` | عمليات الـ pin configuration |

---

### Group 1: تسجيل الـ Pin Controller

هذه المجموعة هي نقطة دخول الـ driver — تُسجِّل الـ pin controller في الـ framework وتُنشئ الـ `struct pinctrl_dev` الذي يمثّل الـ controller داخل الـ kernel.

---

#### `pinctrl_register_and_init()`

```c
int pinctrl_register_and_init(const struct pinctrl_desc *pctldesc,
                              struct device *dev,
                              void *driver_data,
                              struct pinctrl_dev **pctldev);
```

بتسجّل الـ pin controller في الـ subsystem بس من غير ما تفعّله فوراً — لازم تستدعي `pinctrl_enable()` بعدين. ده بيسمح للـ driver يعمل أي initialization لازمة قبل ما الـ core يبدأ يستخدم الـ controller.

**Parameters:**
- `pctldesc` — الـ descriptor الكامل للـ controller (pins, ops, name, إلخ)
- `dev` — الـ `struct device *` الخاص بالـ controller
- `driver_data` — بيانات خاصة بالـ driver، الـ core ما بيلمسهاش، يُسترجَع لاحقاً بـ `pinctrl_dev_get_drvdata()`
- `pctldev` — **output**: pointer للـ `struct pinctrl_dev *` المُنشأ

**Return:** `0` عند النجاح، كود خطأ سالب عند الفشل.

**Key details:**
- الـ preferred API منذ kernel 4.x — استخدمه دائماً عوضاً عن `pinctrl_register()`.
- لا يُشغِّل أي hogging أو pin claiming حتى تستدعي `pinctrl_enable()`.
- يحجز memory داخلياً لـ `pinctrl_dev` — لا تُحرِّرها يدوياً.
- يُسجِّل الـ debugfs entries تلقائياً.

**Caller context:** process context فقط، عادةً من `probe()`.

---

#### `pinctrl_enable()`

```c
int pinctrl_enable(struct pinctrl_dev *pctldev);
```

يُفعِّل الـ pin controller بعد تسجيله بـ `pinctrl_register_and_init()`. عند هذه النقطة يبدأ الـ core في معالجة الـ mapping table entries المرتبطة بهذا الـ controller، ويُنفِّذ أي **hogging** entries.

**Parameters:**
- `pctldev` — الـ handle المُسترجَع من `pinctrl_register_and_init()`

**Return:** `0` عند النجاح.

**Key details:**
- الـ hogging يحدث هنا: إذا كان `dev_name == ctrl_dev_name` في الـ mapping table، الـ core يستدعي `pinctrl_get()` و`pinctrl_select_state()` تلقائياً على هذه الـ entries.
- يجب استدعاؤه بعد اكتمال أي initialization يحتاجها الـ driver قبل قبول الـ requests.

**Caller context:** process context، مباشرةً بعد `pinctrl_register_and_init()`.

---

#### `pinctrl_unregister()`

```c
void pinctrl_unregister(struct pinctrl_dev *pctldev);
```

إلغاء تسجيل الـ pin controller وتحرير كل resources مرتبطة به. يُعكِس عملية `pinctrl_register_and_init()` كاملاً.

**Parameters:**
- `pctldev` — الـ handle المراد إلغاء تسجيله

**Key details:**
- يُحرِّر الـ debugfs entries.
- يُحرِّر الـ GPIO ranges المُضافة.
- إذا كان هناك hogged states، يُحرِّرها.
- للـ `devm_` variants، الـ framework يستدعيها تلقائياً عند `device_del()`.

**Caller context:** process context، من `remove()` أو error path في `probe()`.

---

#### `devm_pinctrl_register_and_init()`

```c
int devm_pinctrl_register_and_init(struct device *dev,
                                   const struct pinctrl_desc *pctldesc,
                                   void *driver_data,
                                   struct pinctrl_dev **pctldev);
```

نفس `pinctrl_register_and_init()` لكن مع ربط lifetime الـ controller بـ `struct device *`. عند إزالة الـ device يُستدعَى `pinctrl_unregister()` تلقائياً.

**Key details:** الـ preferred API للـ drivers الحديثة — يُلغي الحاجة لـ cleanup manual.

---

### Group 2: إدارة الـ GPIO Ranges

هذه المجموعة تُعرِّف العلاقة بين الـ pin number space الخاص بالـ controller والـ GPIO number space العالمي. ضرورية حتى يعرف الـ GPIO subsystem أي pin controller يتحكم في GPIO معين.

---

#### `pinctrl_add_gpio_range()`

```c
void pinctrl_add_gpio_range(struct pinctrl_dev *pctldev,
                            struct pinctrl_gpio_range *range);
```

يُسجِّل mapping بين range من GPIO numbers وrange من pin numbers في controller معين. بعد هذه الاستدعاء، عندما يطلب الـ GPIO subsystem عمليات على GPIO رقم معين، الـ pinctrl core يبحث في هذا الـ range ويجد الـ controller المناسب.

**Parameters:**
- `pctldev` — الـ controller الذي سيتولى هذا الـ range
- `range` — struct تصف الـ mapping (اسم، base GPIO، base pin، عدد الـ pins)

**Key details:**
- استدعاء هذه الدالة من الـ pinctrl driver نفسه **deprecated** — استخدم DT binding بدلاً منه (راجع `Documentation/devicetree/bindings/gpio/gpio.txt`).
- تُضيف الـ range لـ linked list داخلي في `pctldev`.
- إذا كان `range->pins != NULL` يُهمَل `range->pin_base` ويُستخدم الـ array المُخصَّص.

**Caller context:** process context.

---

#### `pinctrl_add_gpio_ranges()`

```c
void pinctrl_add_gpio_ranges(struct pinctrl_dev *pctldev,
                             struct pinctrl_gpio_range *ranges,
                             unsigned int nranges);
```

نسخة batch من `pinctrl_add_gpio_range()` — تُضيف `nranges` ranges دفعة واحدة.

---

#### `pinctrl_remove_gpio_range()`

```c
void pinctrl_remove_gpio_range(struct pinctrl_dev *pctldev,
                               struct pinctrl_gpio_range *range);
```

إزالة GPIO range مُسجَّل مسبقاً. يُستخدَم عند unbinding أو hot-unplug للـ GPIO chip.

---

#### `pinctrl_find_and_add_gpio_range()`

```c
struct pinctrl_dev *pinctrl_find_and_add_gpio_range(const char *devname,
                                                     struct pinctrl_gpio_range *range);
```

يبحث عن pin controller باسمه (`devname`) ويُضيف الـ GPIO range إليه. مفيد عندما يحتاج الـ GPIO driver إيجاد الـ pinctrl controller المقابل له.

**Return:** `struct pinctrl_dev *` للـ controller المُوجَد، أو ERR_PTR عند الخطأ.

---

#### `pinctrl_get_group_pins()`

```c
int pinctrl_get_group_pins(struct pinctrl_dev *pctldev,
                           const char *pin_group,
                           const unsigned int **pins,
                           unsigned int *num_pins);
```

يسترجع مصفوفة الـ pins المنتمية لـ group باسمه. مفيد لتهيئة `pinctrl_gpio_range.pins` و`pinctrl_gpio_range.npins` مباشرةً من اسم group.

**Parameters:**
- `pin_group` — اسم الـ group
- `pins` — output: pointer لمصفوفة الـ pin numbers
- `num_pins` — output: عدد الـ pins

**Return:** `0` عند النجاح، `-EINVAL` إذا لم يُوجَد الـ group.

---

### Group 3: Consumer API

هذه المجموعة هي واجهة الـ device drivers التي **تستهلك** خدمات الـ pinctrl — أي تطلب pin states وتُفعِّلها. الـ flow الأساسي: `get → lookup_state → select_state → put`.

---

#### `pinctrl_get()` / `devm_pinctrl_get()`

```c
struct pinctrl * __must_check pinctrl_get(struct device *dev);
struct pinctrl * __must_check devm_pinctrl_get(struct device *dev);
```

يُنشئ handle يمثّل علاقة الـ device بنظام الـ pinctrl. يقرأ الـ mapping table ويُخصِّص `struct pinctrl` داخلي يحمل قائمة بالـ states المتاحة لهذا الـ device.

**Parameters:**
- `dev` — الـ device الذي يطلب الـ pinctrl handle

**Return:** pointer لـ `struct pinctrl` أو `ERR_PTR()` عند الخطأ.

**Key details:**
- عملية **بطيئة** — تُحلِّل الـ mapping table بالكامل. لا تستدعيها من interrupt context.
- إذا لم يُوجَد الـ pinctrl driver بعد (مثلاً لم يُسجَّل بعد)، ترجع `-EPROBE_DEFER` — الـ driver core يُعيد المحاولة لاحقاً.
- الـ `devm_pinctrl_get()` يُسجِّل destructor تلقائياً يستدعي `pinctrl_put()` عند `device_del()`.
- `__must_check` — يجب دائماً فحص قيمة الإرجاع.

**Pairing rules:**
```
pinctrl_get()       ↔  pinctrl_put()       (فقط)
devm_pinctrl_get()  ↔  devm_pinctrl_put()  (أو تلقائي)
لا تخلط بينهم!
```

**Pseudocode:**
```
pinctrl_get(dev):
    scan global mapping table for entries where dev_name == dev->name
    allocate struct pinctrl
    for each matching map entry:
        parse function/group/config into pinctrl->states list
    return pinctrl handle
```

---

#### `pinctrl_lookup_state()`

```c
struct pinctrl_state * __must_check pinctrl_lookup_state(struct pinctrl *p,
                                                          const char *name);
```

يبحث عن state بالاسم داخل الـ handle المُنشأ بـ `pinctrl_get()`. الـ state هو مجموعة من الـ mux + config settings مُجمَّعة تحت اسم واحد مثل `"default"` أو `"sleep"`.

**Parameters:**
- `p` — الـ pinctrl handle
- `name` — اسم الـ state (مثل `PINCTRL_STATE_DEFAULT`, `"sleep"`, `"pos-A"`)

**Return:** `struct pinctrl_state *` أو `ERR_PTR(-ENODEV)` إذا لم يُوجَد.

**Key details:**
- عملية بطيئة نسبياً — اعمل lookup مرة واحدة في `probe()` واحفظ النتيجة.
- الـ state name يجب أن يطابق ما هو معرَّف في الـ mapping table أو DT.

---

#### `pinctrl_select_state()`

```c
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *s);
```

تُفعِّل state معين: تُطبِّق كل الـ mux settings وكل الـ pin configurations المرتبطة بهذا الـ state. هذه هي الدالة التي **تلمس الـ hardware فعلياً**.

**Parameters:**
- `p` — الـ pinctrl handle
- `s` — الـ state المُراد تفعيله (مُسترجَع من `pinctrl_lookup_state()`)

**Return:** `0` عند النجاح، كود خطأ سالب عند الفشل.

**Key details:**
- نظرياً fast path — بس لو الـ pin controller registers على slow bus (I2C/SPI)، ممكن تكون بطيئة أو تنام.
- **لا تستدعيها من non-blocking context** إلا إذا تأكدت أن الـ controller على fast bus مباشر.
- تُعطِّل الـ state السابق وتُفعِّل الجديد atomically من منظور الـ subsystem (لكن ليس hardware-atomically).
- تمر على كل الـ mux groups في الـ state وتستدعي `.set_mux()` لكل واحد، ثم تُطبِّق الـ pin configs.

**Pseudocode:**
```
pinctrl_select_state(p, s):
    deactivate currently active state (if any):
        for each mux setting in old state:
            call pmxops->free() for each pin
    activate new state:
        for each mux setting in s:
            call pmxops->request() for each pin
            call pmxops->set_mux(func_selector, group_selector)
        for each config setting in s:
            call confops->pin_config_set() or pin_config_group_set()
    p->state = s
```

---

#### `pinctrl_put()` / `devm_pinctrl_put()`

```c
void pinctrl_put(struct pinctrl *p);
void devm_pinctrl_put(struct pinctrl *p);
```

يُحرِّر الـ `struct pinctrl` handle وكل resources مرتبطة به — يُلغي تفعيل أي active state ويُحرِّر الـ pins المحجوزة.

**Key details:**
- `devm_pinctrl_put()` نادر الاستخدام لأن الـ cleanup يحدث تلقائياً، لكن مفيد في error paths داخل `probe()`.
- بعد `pinctrl_put()` الـ handle غير صالح — لا تستخدمه.

---

#### `pinctrl_get_select()` (inline helper)

```c
static inline struct pinctrl * __must_check
pinctrl_get_select(struct device *dev, const char *name);
```

يدمج `pinctrl_get()` + `pinctrl_lookup_state()` + `pinctrl_select_state()` في استدعاء واحد. إذا فشلت أي خطوة، تُنظِّف ما سبق وترجع ERR_PTR.

**Key details:**
- مناسب للـ drivers البسيطة التي لا تحتاج تبديل states في runtime.
- الـ handle المُرجَع لازم تحفظه لاستخدامه لاحقاً في `pinctrl_put()`.

---

#### `pinctrl_select_default_state()`

```c
int pinctrl_select_default_state(struct device *dev);
```

تفعيل `PINCTRL_STATE_DEFAULT` مباشرة من الـ `struct device *` بدون الحاجة لـ pinctrl handle. تستخدم `dev->pins` التي أعدّها الـ device core مسبقاً.

---

### Group 4: PM State Helpers

هذه الدوال مصمَّمة لاستخدامها في الـ `suspend`/`resume` callbacks مباشرةً — تتعامل مع الـ `dev->pins` structure الذي يُعدُّه الـ device core مسبقاً في `pinctrl_bind_pins()`.

---

#### `pinctrl_pm_select_sleep_state()`

```c
int pinctrl_pm_select_sleep_state(struct device *dev);
```

تُفعِّل الـ "sleep" state الذي تم lookup له مسبقاً في `dev->pins->sleep_state`. تُستدعى من `suspend()` callbacks.

**Parameters:**
- `dev` — الـ device المراد وضعه في sleep state

**Return:** `0` عند النجاح، أو إذا لم يُعرَّف sleep state (لا تُعتبر خطأً).

**Key details:**
- فقط متاحة إذا كان `CONFIG_PM` مُفعَّلاً.
- الـ state يجب أن يكون معرَّفاً في DT أو mapping table باسم `"sleep"`.
- لو `dev->pins == NULL` أو `sleep_state == NULL` ترجع `0` بدون عمل شيء.

---

#### `pinctrl_pm_select_default_state()`

```c
int pinctrl_pm_select_default_state(struct device *dev);
```

ترجع الـ pins للـ "default" state — تُستدعى من `resume()` لاستعادة الحالة الطبيعية.

---

#### `pinctrl_pm_select_init_state()`

```c
int pinctrl_pm_select_init_state(struct device *dev);
```

تُفعِّل "init" state — مفيدة في `resume()` إذا احتاج الـ hardware تسلسل معين قبل الوصول لـ "default".

---

#### `pinctrl_pm_select_idle_state()`

```c
int pinctrl_pm_select_idle_state(struct device *dev);
```

تُفعِّل "idle" state — للـ runtime PM، عندما يكون الـ device idle لكن ليس في deep sleep.

**مثال كامل:**
```c
static int foo_suspend(struct device *dev)
{
    /* suspend the hardware first */
    foo_hw_suspend(dev);

    /* then put pins to sleep (e.g. Hi-Z or low-power bias) */
    return pinctrl_pm_select_sleep_state(dev);
}

static int foo_resume(struct device *dev)
{
    int ret;

    /* init state first if needed (e.g. clock enable sequence) */
    ret = pinctrl_pm_select_init_state(dev);
    if (ret)
        return ret;

    /* resume hardware */
    foo_hw_resume(dev);

    /* return to normal operation */
    return pinctrl_pm_select_default_state(dev);
}
```

---

### Group 5: GPIO-Pinctrl Bridge

هذه الدوال هي الجسر بين الـ GPIO subsystem والـ pinctrl subsystem. **يجب** استدعاؤها فقط من داخل gpiolib drivers — أي من داخل الـ `gpio_chip` callbacks، وليس من أي driver آخر مباشرةً.

---

#### `pinctrl_gpio_request()`

```c
int pinctrl_gpio_request(struct gpio_chip *gc, unsigned int offset);
```

يُطلَق من داخل `gpio_chip.request()`. يُخبر الـ pinctrl core أن GPIO معين سيُستخدَم كـ GPIO — الـ core يبحث عن الـ pin المقابل في الـ GPIO ranges ويتأكد أنه غير مُستخدَم من function أخرى، ثم يستدعي `pmxops->gpio_request_enable()` إذا كانت موجودة.

**Parameters:**
- `gc` — الـ gpio_chip صاحب الطلب
- `offset` — offset داخل الـ gpio_chip

**Return:** `0` عند النجاح، `-EBUSY` إذا كان الـ pin محجوزاً، `-ENODEV` إذا لم يُوجَد mapping.

**Key details:**
- إذا لم يُسجَّل الـ pin controller أي `gpio_request_enable()` callback، يحاول الـ core تفعيل function اسمها `"gpioN"` حيث N هو الـ GPIO number العالمي.
- الـ `strict` flag في `pinmux_ops` يتحكم فيما إذا كان يُسمَح بتشارك الـ pin مع function أخرى.

---

#### `pinctrl_gpio_free()`

```c
void pinctrl_gpio_free(struct gpio_chip *gc, unsigned int offset);
```

عكس `pinctrl_gpio_request()` — يُطلَق من `gpio_chip.free()`. يُخبر الـ pinctrl core أن الـ GPIO لم يعد مستخدماً ويستدعي `pmxops->gpio_disable_free()`.

---

#### `pinctrl_gpio_direction_input()` / `pinctrl_gpio_direction_output()`

```c
int pinctrl_gpio_direction_input(struct gpio_chip *gc, unsigned int offset);
int pinctrl_gpio_direction_output(struct gpio_chip *gc, unsigned int offset);
```

يُطلَقان من `gpio_chip.direction_input()` و`gpio_chip.direction_output()`. يستدعيان `pmxops->gpio_set_direction()` في الـ pin controller driver لتعيين الاتجاه — ضروري في بعض الـ SoCs حيث يتحكم الـ pin controller في اتجاه الـ GPIO وليس الـ GPIO block نفسه.

---

#### `pinctrl_gpio_can_use_line()`

```c
bool pinctrl_gpio_can_use_line(struct gpio_chip *gc, unsigned int offset);
```

يفحص ما إذا كان GPIO line معين متاحاً للاستخدام (غير محجوز من function أخرى في الـ pinctrl). مفيد لتقرير ما إذا كان يمكن request الـ line قبل محاولة ذلك.

**Return:** `true` إذا كان ممكناً الاستخدام.

---

#### `pinctrl_gpio_set_config()`

```c
int pinctrl_gpio_set_config(struct gpio_chip *gc, unsigned int offset,
                            unsigned long config);
```

يُطبِّق pin configuration (مثل pull-up/down) على GPIO pin عبر الـ pinctrl subsystem. يُستدعى من `gpio_chip.set_config()`.

---

### Group 6: Mapping Table Registration

الـ mapping table هي ما يربط الـ device بالـ function والـ group على controller معين. بدونها لا يعرف الـ pinctrl core أي pins يُفعِّل لأي device.

---

#### `pinctrl_register_mappings()`

```c
int pinctrl_register_mappings(const struct pinctrl_map *map,
                              unsigned int num_maps);
```

يُسجِّل مصفوفة من `struct pinctrl_map` في الـ global mapping table. يُستدعى عادةً من `__init` code في الـ board file أو من الـ pinctrl driver نفسه.

**Parameters:**
- `map` — مصفوفة الـ mapping entries
- `num_maps` — عدد الـ entries

**Return:** `0` عند النجاح.

**Key details:**
- الـ map يجب أن تبقى صالحة طوال عمر النظام (لا تستخدم stack-allocated maps بدون `__initdata`).
- الـ core لا يُنشئ نسخة — يُخزِّن pointer للـ map الأصلي.
- في الغالب اليوم يُستخدَم DT بدلاً من الـ static mapping tables — `dt_node_to_map()` callback في `pinctrl_ops` يُنشئ الـ map تلقائياً من DT.

---

#### `devm_pinctrl_register_mappings()`

```c
int devm_pinctrl_register_mappings(struct device *dev,
                                   const struct pinctrl_map *map,
                                   unsigned int num_maps);
```

نسخة devm — يُلغي تسجيل الـ map تلقائياً عند إزالة الـ device.

---

#### `pinctrl_unregister_mappings()`

```c
void pinctrl_unregister_mappings(const struct pinctrl_map *map);
```

يُزيل الـ mapping table من الـ global list. يُستخدَم لو أُضيف الـ map ديناميكياً وأُريد إزالته.

---

#### `pinctrl_provide_dummies()`

```c
void pinctrl_provide_dummies(void);
```

عند استدعائها، الـ core يتجاهل طلبات الـ pinctrl الفاشلة بدلاً من إرجاع خطأ. مفيد في بيئات مثل early boot أو platforms بدون pin controller hardware.

---

### Group 7: Pinmux Ops Callbacks (vtable للـ Driver)

هذه هي الـ callbacks التي يجب على الـ pin controller driver تنفيذها وتسجيلها في `struct pinmux_ops`.

---

#### `.get_functions_count()`

```c
int (*get_functions_count)(struct pinctrl_dev *pctldev);
```

يُرجِع العدد الإجمالي للـ functions المتاحة في هذا الـ controller. الـ core يستخدمه لتحديد نطاق الـ valid function selectors.

**Return:** عدد صحيح موجب يمثّل عدد الـ functions.

---

#### `.get_function_name()`

```c
const char *(*get_function_name)(struct pinctrl_dev *pctldev,
                                  unsigned int selector);
```

يُرجِع اسم الـ function رقم `selector`. الـ core يستخدمه للمطابقة مع الـ function المطلوب في الـ mapping table.

**Parameters:**
- `selector` — index الـ function (من 0 إلى `get_functions_count()-1`)

**Return:** string ثابت، لا تُخصِّص memory ديناميكياً.

---

#### `.get_function_groups()`

```c
int (*get_function_groups)(struct pinctrl_dev *pctldev,
                            unsigned int selector,
                            const char * const **groups,
                            unsigned int *num_groups);
```

يُرجِع قائمة أسماء الـ groups المتاحة لـ function معين. الـ core يستخدمها لإيجاد أي group سيُفعَّل مع هذه الـ function.

**Parameters:**
- `selector` — index الـ function
- `groups` — **output**: مصفوفة أسماء الـ groups
- `num_groups` — **output**: عدد الـ groups

**Return:** `0` دائماً في التنفيذ الطبيعي.

---

#### `.set_mux()`

```c
int (*set_mux)(struct pinctrl_dev *pctldev,
               unsigned int func_selector,
               unsigned int group_selector);
```

**أهم callback في الـ pinmux** — يُفعِّل function معينة على group معين بكتابة الـ registers. الـ core يضمن مسبقاً أن الـ pins غير محجوزة، فالـ driver لا يحتاج التحقق من التعارضات.

**Parameters:**
- `func_selector` — index الـ function المُختارة
- `group_selector` — index الـ group المُختار

**Return:** `0` عند النجاح.

**Key details:**
- يجب أن تكون atomic بقدر الإمكان.
- الـ core يحمي من تعارض الـ pins قبل الاستدعاء.
- يتحمل الـ driver مسؤولية كتابة الـ MUX registers الصحيحة.

---

#### `.request()` / `.free()`

```c
int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);
int (*free)(struct pinctrl_dev *pctldev, unsigned int offset);
```

**optional** — تُستدعى قبل وبعد استخدام الـ pin. مفيدة إذا احتاج الـ driver التحقق من إمكانية استخدام الـ pin (مثلاً فحص power domain أو clock).

---

#### `.gpio_request_enable()` / `.gpio_disable_free()`

```c
int (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                            struct pinctrl_gpio_range *range,
                            unsigned int offset);
void (*gpio_disable_free)(struct pinctrl_dev *pctldev,
                           struct pinctrl_gpio_range *range,
                           unsigned int offset);
```

**optional** — تُستدعى عند طلب أو تحرير GPIO pin. تسمح للـ driver بتخصيص عملية التحويل لـ GPIO mode لكل pin على حدة بدلاً من استخدام function عامة.

---

#### `.gpio_set_direction()`

```c
int (*gpio_set_direction)(struct pinctrl_dev *pctldev,
                           struct pinctrl_gpio_range *range,
                           unsigned int offset,
                           bool input);
```

**optional** — يضبط اتجاه GPIO pin في حالة احتياج الـ pin controller للتدخل في عملية ضبط الاتجاه.

**Parameters:**
- `input` — `true` للـ input، `false` للـ output

---

### Group 8: Pinconf Ops Callbacks (vtable للـ Driver)

---

#### `.pin_config_get()` / `.pin_config_set()`

```c
int (*pin_config_get)(struct pinctrl_dev *pctldev,
                       unsigned int pin,
                       unsigned long *config);

int (*pin_config_set)(struct pinctrl_dev *pctldev,
                       unsigned int pin,
                       unsigned long *configs,
                       unsigned int num_configs);
```

يقرأ أو يكتب configuration لـ pin فردي. الـ `config` هو `unsigned long` يحمل packed value: الـ 8 bits السفلية هي الـ `pin_config_param`، والـ 24 bits العليا هي القيمة.

**Key details:**
- استخدم `pinconf_to_config_param()` و`pinconf_to_config_argument()` لاستخراج القيم.
- إذا كان الـ param غير مدعوم، ارجع `-ENOTSUPP`.
- إذا كان مدعوماً لكن معطلاً، ارجع `-EINVAL`.
- `.pin_config_set()` يستقبل مصفوفة configs ويطبّقها كلها.

**مثال:**
```c
static int foo_pin_config_set(struct pinctrl_dev *pctldev,
                               unsigned int pin,
                               unsigned long *configs,
                               unsigned int num_configs)
{
    unsigned int i;
    for (i = 0; i < num_configs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        u32 arg = pinconf_to_config_argument(configs[i]);

        switch (param) {
        case PIN_CONFIG_BIAS_PULL_UP:
            foo_enable_pullup(pin, arg /* resistance in ohms */);
            break;
        case PIN_CONFIG_DRIVE_STRENGTH:
            foo_set_drive(pin, arg /* mA */);
            break;
        default:
            return -ENOTSUPP;
        }
    }
    return 0;
}
```

---

#### `.pin_config_group_get()` / `.pin_config_group_set()`

```c
int (*pin_config_group_get)(struct pinctrl_dev *pctldev,
                             unsigned int selector,
                             unsigned long *config);

int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                             unsigned int selector,
                             unsigned long *configs,
                             unsigned int num_configs);
```

نفس فكرة الـ per-pin callbacks لكن تُطبَّق على group كامل دفعة واحدة. `selector` هو الـ group index.

---

### Group 9: Pinctrl Ops Callbacks (vtable للـ Driver)

---

#### `.get_groups_count()` / `.get_group_name()` / `.get_group_pins()`

```c
int (*get_groups_count)(struct pinctrl_dev *pctldev);
const char *(*get_group_name)(struct pinctrl_dev *pctldev, unsigned int selector);
int (*get_group_pins)(struct pinctrl_dev *pctldev,
                       unsigned int selector,
                       const unsigned int **pins,
                       unsigned int *num_pins);
```

الثلاثي الإلزامي لتعريف الـ pin groups. الـ core يستخدمهم لاكتشاف كل الـ groups المتاحة وما تحتويه من pins.

**Key details:**
- `get_groups_count()` يُحدِّد نطاق الـ valid selectors.
- `get_group_name()` يُعيد string ثابت — لا memory allocation.
- `get_group_pins()` يُعيد pointer لمصفوفة ثابتة — لا يُنشئ copy.
- الثلاثة **إلزامية** إذا أردت دعم pinmux أو pinconf.

---

#### `.dt_node_to_map()` / `.dt_free_map()`

```c
int (*dt_node_to_map)(struct pinctrl_dev *pctldev,
                       struct device_node *np_config,
                       struct pinctrl_map **map,
                       unsigned int *num_maps);

void (*dt_free_map)(struct pinctrl_dev *pctldev,
                    struct pinctrl_map *map,
                    unsigned int num_maps);
```

يُحوِّل DT node لـ mapping table entries — هذا هو الـ DT parsing interface.

**Key details:**
- **optional** — لكن ضروري لأي driver يدعم DT.
- الـ map المُنشأ يُحرَّر لاحقاً بـ `dt_free_map()` من الـ core.
- معظم الـ drivers تستخدم `pinconf_generic_dt_node_to_map()` helpers بدلاً من كتابة هذا من الصفر.

**Pseudocode:**
```
dt_node_to_map(pctldev, np_config, map, num_maps):
    count entries needed from np_config subnodes
    allocate map array
    for each subnode in np_config:
        parse "pins" or "groups" property → group/pin names
        parse "function" property → function name
        parse pin config properties (bias, drive-strength, etc.)
        fill map entries (MUX_GROUP and/or CONFIGS_PIN/GROUP)
    *map = allocated_map
    *num_maps = count
    return 0
```

---

### Group 10: Generic Pinconf DT Helpers

---

#### `pinconf_generic_dt_node_to_map()`

```c
int pinconf_generic_dt_node_to_map(struct pinctrl_dev *pctldev,
                                    struct device_node *np_config,
                                    struct pinctrl_map **map,
                                    unsigned int *num_maps,
                                    enum pinctrl_map_type type);
```

يُوفِّر parsing جاهز لـ DT nodes التي تستخدم الـ generic pinconf properties (مثل `bias-pull-up`, `drive-strength`, `output-low`, إلخ). معظم الـ drivers الحديثة تستخدمه كـ `dt_node_to_map` implementation.

**Parameters:**
- `type` — نوع الـ map المُنشأ: `PIN_MAP_TYPE_CONFIGS_PIN`, `PIN_MAP_TYPE_CONFIGS_GROUP`, أو `PIN_MAP_TYPE_INVALID` للـ auto-detect.

**Key details:**
- يقرأ الـ standard DT properties المُعرَّفة في `pinconf-generic.h`.
- يُنشئ map entries ديناميكياً — يجب تحريرها بـ `pinconf_generic_dt_free_map()`.

---

### Group 11: Helper Macros والـ Packed Config

---

#### `PIN_CONF_PACKED()` و helper inlines

```c
#define PIN_CONF_PACKED(p, a) ((a << 8) | ((unsigned long) p & 0xffUL))

static inline enum pin_config_param pinconf_to_config_param(unsigned long config);
static inline u32 pinconf_to_config_argument(unsigned long config);
static inline unsigned long pinconf_to_config_packed(enum pin_config_param param, u32 argument);
```

الـ pinconf subsystem يُعبِّر عن الـ configuration كـ `unsigned long` واحد يحمل:
- bits [7:0]: الـ `pin_config_param` enum value
- bits [31:8]: القيمة/الـ argument

**مثال:**
```c
/* تفعيل pull-up بمقاومة 100kΩ */
unsigned long cfg = PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 100000);

/* استخراج لاحقاً */
enum pin_config_param p = pinconf_to_config_param(cfg); /* PIN_CONFIG_BIAS_PULL_UP */
u32 ohms = pinconf_to_config_argument(cfg);             /* 100000 */
```

---

#### `PINCTRL_PIN()` و `PINCTRL_PINGROUP()` و `PINCTRL_PINFUNCTION()`

```c
#define PINCTRL_PIN(a, b)                    /* { .number = a, .name = b } */
#define PINCTRL_PINGROUP(_name, _pins, _npins)  /* struct pingroup */
#define PINCTRL_PINFUNCTION(_name, _groups, _ngroups)  /* struct pinfunction */
```

**convenience macros** لتعريف الـ static tables:

```c
/* تعريف الـ pins */
static const struct pinctrl_pin_desc myctrl_pins[] = {
    PINCTRL_PIN(0, "PA0"),
    PINCTRL_PIN(1, "PA1"),
    /* ... */
};

/* تعريف الـ groups */
static const unsigned int spi_pins[] = { 0, 1, 2, 3 };
static const struct pingroup myctrl_groups[] = {
    PINCTRL_PINGROUP("spi0_grp", spi_pins, ARRAY_SIZE(spi_pins)),
};

/* تعريف الـ functions */
static const char * const spi_groups[] = { "spi0_grp" };
static const struct pinfunction myctrl_functions[] = {
    PINCTRL_PINFUNCTION("spi0", spi_groups, ARRAY_SIZE(spi_groups)),
};
```

---

### Group 12: Mapping Table Convenience Macros

هذه الـ macros تُبسِّط بناء الـ `struct pinctrl_map` arrays.

| Macro | الغرض |
|---|---|
| `PIN_MAP_MUX_GROUP(dev, state, pinctrl, grp, func)` | إضافة mux entry |
| `PIN_MAP_MUX_GROUP_DEFAULT(dev, pinctrl, grp, func)` | mux entry للـ default state |
| `PIN_MAP_MUX_GROUP_HOG_DEFAULT(dev, grp, func)` | hogged mux entry على الـ default state |
| `PIN_MAP_CONFIGS_PIN(dev, state, pinctrl, pin, cfgs)` | config لـ pin فردي |
| `PIN_MAP_CONFIGS_GROUP(dev, state, pinctrl, grp, cfgs)` | config لـ group |
| `PIN_MAP_DUMMY_STATE(dev, state)` | state فارغ (explicit empty) |

**مثال:**
```c
static unsigned long uart_sleep_cfg[] = {
    PIN_CONF_PACKED(PIN_CONFIG_LEVEL, 0),         /* drive low */
};

static struct pinctrl_map uart_map[] __initdata = {
    /* normal operation: UART function on u0_group */
    PIN_MAP_MUX_GROUP("uart0", PINCTRL_STATE_DEFAULT,
                      "pinctrl-foo", "u0_group", "uart0"),

    /* sleep: gpio-mode (drive TX low) */
    PIN_MAP_MUX_GROUP("uart0", PINCTRL_STATE_SLEEP,
                      "pinctrl-foo", "u0_group", "gpio-mode"),
    PIN_MAP_CONFIGS_PIN("uart0", PINCTRL_STATE_SLEEP,
                        "pinctrl-foo", "UART_TX", uart_sleep_cfg),
};
```

---

### الـ Standard Pin States

```c
#define PINCTRL_STATE_DEFAULT  "default"   /* normal operation */
#define PINCTRL_STATE_INIT     "init"      /* before probe() */
#define PINCTRL_STATE_IDLE     "idle"      /* runtime suspend */
#define PINCTRL_STATE_SLEEP    "sleep"     /* system suspend */
```

| State | متى يُفعَّل تلقائياً | من يُفعِّله يدوياً |
|---|---|---|
| `default` | قبل `probe()` من الـ device core | `pinctrl_pm_select_default_state()` |
| `init` | قبل `probe()` (إذا موجود) | `pinctrl_pm_select_init_state()` |
| `sleep` | لا | `pinctrl_pm_select_sleep_state()` |
| `idle` | لا | `pinctrl_pm_select_idle_state()` |

---

### الـ `struct dev_pin_info` والـ Device Core Integration

```c
struct dev_pin_info {
    struct pinctrl       *p;             /* the pinctrl handle */
    struct pinctrl_state *default_state; /* "default" */
    struct pinctrl_state *init_state;    /* "init" */
    struct pinctrl_state *sleep_state;   /* "sleep" (CONFIG_PM only) */
    struct pinctrl_state *idle_state;    /* "idle" (CONFIG_PM only) */
};
```

الـ device core ينشئ هذه الـ struct في `pinctrl_bind_pins()` لكل device عند التسجيل. كل driver يحصل على handle جاهز في `dev->pins`. الـ `dev_pinctrl(dev)` helper يُرجِع `dev->pins->p` مباشرةً.

---

### `pinctrl_init_done()`

```c
int pinctrl_init_done(struct device *dev);
```

يُستدعى من الـ device core بعد انتهاء `probe()` ليُعلم الـ pinctrl core بالانتهاء من الـ init phase. إذا كان الـ device لا يزال في "init" state، يُحوِّله تلقائياً لـ "default" state.

**Caller:** `device_add()` / device core فقط — لا تستدعيها من الـ driver.

---

### الـ `pinctrl_dev_get_*()` Accessors

```c
const char *pinctrl_dev_get_name(struct pinctrl_dev *pctldev);
const char *pinctrl_dev_get_devname(struct pinctrl_dev *pctldev);
void       *pinctrl_dev_get_drvdata(struct pinctrl_dev *pctldev);
```

يُتيحان للـ driver الوصول للمعلومات المخزّنة في `pinctrl_dev` الـ opaque:
- `get_name()` — اسم الـ controller نفسه (من `pinctrl_desc.name`)
- `get_devname()` — اسم الـ `struct device` الأب
- `get_drvdata()` — الـ `driver_data` المُمرَّر في `pinctrl_register_and_init()`

**مثال نموذجي في driver:**
```c
static int foo_set_mux(struct pinctrl_dev *pctldev,
                        unsigned int func_sel, unsigned int grp_sel)
{
    /* استرجاع private data */
    struct foo_pinctrl *priv = pinctrl_dev_get_drvdata(pctldev);

    /* كتابة الـ register */
    writel(BIT(grp_sel), priv->base + MUX_REG);
    return 0;
}
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. الـ debugfs entries

الـ pinctrl subsystem بيعمل directory كامل جوه `/sys/kernel/debug/pinctrl` فيه كل حاجة محتاجها.

**الـ global files:**

| File | المحتوى |
|------|---------|
| `/sys/kernel/debug/pinctrl/pinctrl-devices` | كل الـ pin controller devices + support للـ pinmux/pinconf |
| `/sys/kernel/debug/pinctrl/pinctrl-handles` | الـ handles المتخصصة + الـ pinmux maps المقابلة |
| `/sys/kernel/debug/pinctrl/pinctrl-maps` | كل الـ pinctrl maps المسجلة في النظام |

**الـ per-controller files** (جوه `/sys/kernel/debug/pinctrl/<controller-name>/`):

| File | المحتوى |
|------|---------|
| `pins` | كل pin مسجل + معلومات إضافية زي register contents |
| `gpio-ranges` | الـ ranges اللي بتربط GPIO lines بالـ pins |
| `pingroups` | كل pin groups مسجلة |
| `pinconf-pins` | إعدادات الـ pin config لكل pin |
| `pinconf-groups` | إعدادات الـ pin config لكل group |
| `pinmux-functions` | كل pin functions + الـ groups المرتبطة بيها |
| `pinmux-pins` | مين هو owner لكل pin (mux owner / gpio owner / hog) |
| `pinmux-select` | **اكتب فيه** لتفعيل function لـ group معينة manually |

```bash
# اقرأ كل الـ controllers الموجودة
cat /sys/kernel/debug/pinctrl/pinctrl-devices

# مثال على output:
# name [pinmux] [pinconf]
# pinctrl-foo yes yes

# اعرض كل الـ pins على controller اسمه "pinctrl-foo"
cat /sys/kernel/debug/pinctrl/pinctrl-foo/pins

# مثال على output:
# pin 0 (A8)
# pin 1 (B8)
# pin 24 (A5) gpio-owner: spi0

# اعرف مين يستخدم كل pin
cat /sys/kernel/debug/pinctrl/pinctrl-foo/pinmux-pins

# مثال:
# pin 0 (A8): UNCLAIMED
# pin 24 (A5): device foo-spi.0 function spi0 group spi0_0_grp

# اعرض الـ pin config لكل pin
cat /sys/kernel/debug/pinctrl/pinctrl-foo/pinconf-pins

# فعّل function معينة يدوياً للاختبار
echo "spi0_0_grp spi0" > /sys/kernel/debug/pinctrl/pinctrl-foo/pinmux-select
```

---

#### 2. الـ sysfs entries

الـ pinctrl مش بيعمل حاجة كتير في sysfs مباشرة، لكن الـ GPIO subsystem (المرتبط بيه) بيعمل:

```bash
# اعرض كل الـ GPIO chips المعروفة للـ kernel
ls /sys/class/gpio/

# مثال:
# gpiochip0  gpiochip32  gpiochip48

# اعرض info عن chip معينة
cat /sys/class/gpio/gpiochip32/label
cat /sys/class/gpio/gpiochip32/base
cat /sys/class/gpio/gpiochip32/ngpio

# الـ GPIO ranges المربوطة بالـ pin controller
cat /sys/kernel/debug/pinctrl/pinctrl-foo/gpio-ranges
# مثال:
# GPIO ranges handled:
# 32: 32..47 => pins 32..47
# 48: 48..55 => pins 64..71
```

---

#### 3. الـ ftrace — tracepoints وأحداث مهمة

```bash
# اعرض كل الـ tracepoints المتاحة للـ pinctrl
ls /sys/kernel/debug/tracing/events/pinctrl/

# الـ tracepoints المهمة:
# pinctrl_devinfo   — عند تسجيل pin controller جديد
# pinctrl_select_state  — عند تفعيل state
# pinctrl_pins      — عند عمل get للـ pins

# فعّل tracing للـ pinctrl events كلها
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# أو فعّل event واحدة بس
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_select_state/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الكود اللي عايز تتبعه هنا...

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# مثال على output:
#   foo-driver-123 [000] .... pinctrl_select_state: device=foo-spi.0 state=default

# استخدم function_graph tracer لتتبع مسار pinctrl_select_state
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo pinctrl_select_state > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

---

#### 4. الـ printk والـ dynamic debug

```bash
# فعّل dynamic debug لكل ملفات الـ pinctrl subsystem
echo "file drivers/pinctrl/* +p" > /sys/kernel/debug/dynamic_debug/control

# أو فعّله لـ module معين (مثلاً pinctrl-bcm2835)
echo "module pinctrl_bcm2835 +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل مع line numbers وأسماء functions
echo "file drivers/pinctrl/core.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
# p = print, f = function name, l = line number, m = module name, t = thread id

# شوف الـ dynamic debug الـ active حالياً
cat /sys/kernel/debug/dynamic_debug/control | grep pinctrl

# من الـ kernel command line أثناء boot:
# dyndbg="file drivers/pinctrl/* +p"

# أو في /etc/modprobe.d/pinctrl.conf:
# options pinctrl_foo dyndbg=+p
```

---

#### 5. الـ kernel config options للـ debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_DEBUG_PINCTRL` | يفعّل extra checks وـ validation في الـ pinctrl core |
| `CONFIG_PINCTRL` | الـ pinctrl subsystem الأساسي (لازم يكون enabled) |
| `CONFIG_PINMUX` | دعم الـ pin multiplexing |
| `CONFIG_PINCONF` | دعم الـ pin configuration |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | generic groups support |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | generic functions support |
| `CONFIG_GENERIC_PINCONF` | generic pin config |
| `CONFIG_DEBUG_FS` | لازم enabled عشان `/sys/kernel/debug/pinctrl` يشتغل |
| `CONFIG_DYNAMIC_DEBUG` | لتفعيل dynamic debug messages |
| `CONFIG_GPIO_SYSFS` | يظهر GPIO في sysfs للـ debugging |
| `CONFIG_PROVE_LOCKING` | يكتشف deadlocks في الـ pinctrl locks |
| `CONFIG_LOCKDEP` | تتبع الـ lock dependencies |

```bash
# تحقق من الـ config الحالية
grep -E "CONFIG_(DEBUG_PINCTRL|PINMUX|PINCONF|DEBUG_FS)" /boot/config-$(uname -r)

# أو من داخل kernel source
grep -E "CONFIG_DEBUG_PINCTRL" .config
```

---

#### 6. أدوات خاصة بالـ subsystem

```bash
# استخدام gpioinfo (من حزمة gpiod) لرؤية GPIO المرتبطة بالـ pinctrl
gpioinfo

# مثال output:
# gpiochip0 - 54 lines:
#     line  0:   "GPIO0"  unused  input  active-high
#     line  1:   "GPIO1"  "spi0"  output active-high [used]

# gpiodetect لرؤية كل الـ GPIO controllers
gpiodetect

# pinctrl tool (إن وُجد في userspace)
# بعض الـ SoCs زي Raspberry Pi بيجي معاها raspi-gpio
raspi-gpio get        # اعرض حالة كل الـ pins
raspi-gpio get 17     # اعرض حالة pin 17 بس

# مثال output:
# GPIO 17: level=0 fsel=0 func=INPUT

# لمعرفة الـ pin states على Allwinner/Rockchip/etc عبر sysfs
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -v UNCLAIMED
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|--------------------|---------|------|
| `pinctrl: pin X already requested` | pin محجوز من device تانية | اعرف مين حاجز الـ pin من `pinmux-pins`، تأكد من الـ DT mapping |
| `pinctrl: request() failed` | الـ pinctrl driver رفض الطلب | تحقق من الـ `.request()` callback في الـ driver والـ hardware constraints |
| `could not get pinctrl` | `devm_pinctrl_get()` فشلت | الـ pinctrl driver مش registered بعد، محتاج `-EPROBE_DEFER` handling |
| `pinctrl: -EPROBE_DEFER` | الـ pinctrl driver مش جاهز وقت الـ probe | طبيعي أثناء boot، الـ driver هيحاول تاني |
| `could not find function X` | الـ function المطلوبة مش موجودة في الـ driver | تحقق من أسماء الـ functions في الـ DT وفي الـ driver |
| `pin X is not valid` | رقم الـ pin خارج النطاق | تحقق من `npins` في الـ `pinctrl_desc` |
| `could not find group X` | اسم الـ group مش موجود | تحقق من أسماء الـ groups في الـ DT وفي الـ driver |
| `pin X already in use` | تعارض بين function وـ GPIO | تحقق من الـ `strict` flag في `pinmux_ops` |
| `pinctrl state default not found` | الـ device محتاج pinctrl state لكن مش موجود في الـ DT | أضف الـ pinctrl state في الـ DT node |
| `pinmux: select() failed` | `.set_mux()` فشل | مشكلة hardware أو register write، فعّل dynamic debug للـ driver |

---

#### 8. أماكن استراتيجية لـ `dump_stack()` و`WARN_ON()`

```c
/* في pinctrl_select_state() — لما الـ state switch بيفشل بشكل غريب */
ret = ops->set_mux(pctldev, func_selector, group_selector);
if (ret) {
    WARN(1, "set_mux failed for function %s group %s: %d\n",
         func_name, group_name, ret);
    dump_stack();
}

/* في pin_request() — لكشف double-request */
WARN_ON(pin_is_requested(pctldev, pin));

/* في pinctrl_register() — للتحقق من صحة الـ descriptor */
WARN_ON(!pctldesc->name);
WARN_ON(!pctldesc->pins || pctldesc->npins == 0);

/* في gpio_request_enable — للتحقق من الـ range */
WARN_ON(offset >= range->npins);

/* في .pin_config_set() — للتحقق من القيم */
WARN_ON(config_value > MAX_DRIVE_STRENGTH);
```

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ hardware مع الـ kernel state

```bash
# خطوة 1: اعرف إيه اللي الـ kernel يعتقده
cat /sys/kernel/debug/pinctrl/pinctrl-foo/pinmux-pins
# pin 24 (A5): device foo-spi.0 function spi0 group spi0_0_grp

# خطوة 2: اعرف القيمة الفعلية في الـ register
# (محتاج تعرف base address من datasheet)
devmem2 0xFE200000 w   # اقرأ MUX register على عنوان معين

# خطوة 3: قارن بين القيمتين
# لو الـ kernel بيقول spi0 active لكن الـ register قيمته 0، يبقى في مشكلة

# خطوة 4: تحقق من الـ pinconf
cat /sys/kernel/debug/pinctrl/pinctrl-foo/pinconf-pins
# pin 24 (A5): drive-strength:8 bias-pull-up
```

---

#### 2. تقنيات قراءة الـ registers

```bash
# باستخدام devmem2 (أشهر tool)
# اقرأ register بحجم 32-bit على عنوان 0xFE200000
devmem2 0xFE200000 w

# اكتب قيمة في register
devmem2 0xFE200000 w 0x00000024

# باستخدام /dev/mem مباشرة (محتاج CONFIG_STRICT_DEVMEM=n أو /proc/iomem للعناوين)
# (الأفضل استخدام devmem2 بدل ده)

# باستخدام io tool (من الـ busybox أو package منفصل)
io -4 -r 0xFE200000    # اقرأ 32-bit
io -4 -w 0xFE200000 0x24  # اكتب

# اعرف الـ physical addresses المتاحة
cat /proc/iomem | grep -i "pin\|gpio\|mux"
# مثال:
# fe200000-fe2000ff : pinctrl@fe200000

# باستخدام Python مع /dev/mem
python3 -c "
import mmap, struct
with open('/dev/mem', 'r+b') as f:
    mm = mmap.mmap(f.fileno(), 4096, offset=0xFE200000)
    val = struct.unpack('I', mm[0:4])[0]
    print(hex(val))
    mm.close()
"
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**الـ Logic Analyzer:**

```
الإعداد المناسب لكشف مشاكل الـ pinmux:

1. ربط الـ probes:
   - Pin المفروض يكون SPI CLK → CH1
   - Pin المفروض يكون SPI MOSI → CH2
   - Pin المفروض يكون I2C SDA → CH3 (للتأكد مش بيتشارك)

2. إعداد الـ trigger:
   - ابدأ الـ capture عند أول edge على أي channel
   - معدل الـ sampling: 10x أعلى من الـ bus frequency

3. علامات المشكلة:
   - مفيش signal على pin المفروض يكون active → الـ mux مش اتعمل
   - signal على pin تاني غير المتوقع → الـ mux اتعمل على الـ group الغلط
   - signal ضعيف أو noisy → مشكلة في الـ drive strength config
   - ارتفاع بطيء في الـ signal → الـ pull-up/pull-down config غلط
```

**الـ Oscilloscope:**

```
لاكتشاف مشاكل الـ electrical config:

1. Pull-up/Pull-down verification:
   - افصل الـ external component
   - قيس الـ voltage على الـ pin
   - المفروض يكون VDD لو pull-up أو GND لو pull-down
   - لو floating → الـ config مش اشتغلت

2. Drive strength:
   - قيس rise time و fall time للـ signal
   - Rise time بطيء → زود الـ drive strength
   - Overshoot كبير → قلل الـ drive strength

3. Open drain verification:
   - Pin مفروض يكون open-drain
   - لو بيدفع VDD لوحده من غير external pull-up → مشكلة في الـ config
```

---

#### 4. مشاكل الـ hardware الشائعة وـ patterns الـ kernel log

| المشكلة الـ hardware | الـ pattern في الـ kernel log | كيفية التحقق |
|--------------------|--------------------------|-------------|
| Pin مش بيشتغل رغم الـ mux صح | لا يوجد error لكن الـ device مش شغال | قيس الـ voltage على الـ pin بالـ oscilloscope |
| تعارض بين اتنين functions على نفس الـ pin | `pin X already requested` | `cat pinmux-pins` وشوف مين حاجز الـ pin |
| مشكلة في الـ pull resistor | signal unstable / noise في الـ bus | افحص الـ `pinconf-pins` وقيس الـ voltage |
| Drive strength ضعيف | CRC errors في الـ SPI/I2C | قيس الـ rise time بالـ scope، زود الـ drive strength |
| الـ clock للـ pin controller مش enabled | `pinctrl: probe failed: -ENODEV` أو hang | تحقق من الـ clock gating registers |
| الـ voltage domain غلط | احتراق الـ pin أو عدم استجابة | تحقق من الـ VDD للـ IO bank من الـ datasheet |
| Boot order مشكلة | متكرر `EPROBE_DEFER` في الـ dmesg | الـ pinctrl driver لازم يتسجل قبل الـ consumer |

```bash
# لمتابعة EPROBE_DEFER
dmesg | grep -i "eprobe_defer\|pinctrl"

# لرؤية الـ probe order
dmesg | grep -E "probe|pinctrl" | head -50
```

---

#### 5. الـ Device Tree Debugging

```bash
# خطوة 1: تحقق من الـ DT الـ compiled الفعلي المستخدم
ls /proc/device-tree/
# أو
dtc -I fs /proc/device-tree/ 2>/dev/null | grep -A5 "pinctrl"

# خطوة 2: تحقق من الـ pinctrl node
cat /proc/device-tree/soc/pinctrl@fe200000/compatible
# المفروض يطابق الـ driver compatible string

# خطوة 3: تحقق من الـ pinctrl reference في device node
dtc -I fs /proc/device-tree/ 2>/dev/null | grep -B2 -A10 "pinctrl-0"
# مثال:
# spi0 {
#     pinctrl-names = "default", "sleep";
#     pinctrl-0 = <&spi0_pins_default>;
#     pinctrl-1 = <&spi0_pins_sleep>;
# }

# خطوة 4: تأكد من وجود الـ pinctrl states المطلوبة
dtc -I fs /proc/device-tree/ 2>/dev/null | grep "pinctrl-names"

# خطوة 5: قارن الـ pin numbers في الـ DT مع الـ hardware datasheet
# اعرض الـ pinctrl map من الـ DTB المصدر
dtc -I dtb -O dts /boot/dtb/your-board.dtb | grep -A20 "spi0_pins"

# خطوة 6: تحقق من تطابق الـ function name في الـ DT مع الـ driver
dtc -I fs /proc/device-tree/ 2>/dev/null | grep "function"
# يجب أن يطابق ما يظهر في:
cat /sys/kernel/debug/pinctrl/pinctrl-foo/pinmux-functions
```

**أخطاء شائعة في الـ DT:**

```
1. اسم الـ function في DT مختلف عن اللي في الـ driver
   → "could not find function spi" في dmesg

2. اسم الـ group غلط أو مكتوب بشكل مختلف
   → "could not find group spi0_grp" في dmesg

3. الـ phandle بيشير لـ pin controller غلط
   → pins اتعملوا لكن على controller تاني

4. ناسي تضيف "pinctrl-names" أو الأسماء مش بتتطابق مع الـ code
   → "pinctrl state default not found"
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

```bash
# ===== تشخيص شامل في ثوانٍ =====

# 1. اعرف كل الـ pin controllers في النظام
cat /sys/kernel/debug/pinctrl/pinctrl-devices

# 2. اعرف كل الـ pins وحالتها (مين يستخدمها)
for ctrl in /sys/kernel/debug/pinctrl/*/; do
    echo "=== Controller: $ctrl ==="
    cat "${ctrl}pinmux-pins" 2>/dev/null | grep -v UNCLAIMED
done

# 3. اعرف كل الـ active mappings
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# 4. اعرف الـ pin config لكل pin
for ctrl in /sys/kernel/debug/pinctrl/*/; do
    echo "=== PinConf: $ctrl ==="
    cat "${ctrl}pinconf-pins" 2>/dev/null
done

# 5. تحقق من تعارضات الـ pins
cat /sys/kernel/debug/pinctrl/pinctrl-foo/pinmux-pins | grep -v "UNCLAIMED"

# 6. فعّل dynamic debug للـ pinctrl core
echo "file drivers/pinctrl/core.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/pinctrl/pinmux.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/pinctrl/pinconf.c +p" > /sys/kernel/debug/dynamic_debug/control

# 7. اعمل trace لكل state selections
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل الكود ...
cat /sys/kernel/debug/tracing/trace | grep pinctrl

# 8. جرب تفعيل mux يدوياً للاختبار
echo "spi0_0_grp spi0" > /sys/kernel/debug/pinctrl/pinctrl-foo/pinmux-select

# 9. اعرض GPIO ranges المربوطة بالـ pin controller
cat /sys/kernel/debug/pinctrl/pinctrl-foo/gpio-ranges

# 10. سكريبت تشخيص متكامل
#!/bin/bash
CTRL="pinctrl-foo"  # غيّر ده لاسم الـ controller بتاعك
BASE="/sys/kernel/debug/pinctrl/${CTRL}"
echo "======= Pinctrl Diagnostic Report ======="
echo "--- Devices ---"
cat /sys/kernel/debug/pinctrl/pinctrl-devices
echo ""
echo "--- Active Pins ---"
cat "${BASE}/pinmux-pins" | grep -v UNCLAIMED
echo ""
echo "--- Pin Groups ---"
cat "${BASE}/pingroups"
echo ""
echo "--- Functions ---"
cat "${BASE}/pinmux-functions"
echo ""
echo "--- Pin Config ---"
cat "${BASE}/pinconf-pins"
echo "=========================================="
```

**تفسير الـ output:**

```
# مثال على output من pinmux-pins:
pin 0 (A8):   UNCLAIMED          → الـ pin مش محجوز من حد
pin 8 (B5):   UNCLAIMED          → نفس
pin 16 (C5):  device foo-spi.0 function spi0 group spi0_0_grp  → SPI يستخدمه
pin 24 (A5):  device foo-spi.0 function spi0 group spi0_0_grp  → نفس SPI
pin 32 (D4):  GPIO 32 foo-gpio.0 → GPIO يستخدمه
pin 40 (E4):  GPIO 40 (hog)      → الـ system هاجه من البداية

# لو شايف pin محتاجه للـ device بتاعتك وهو محجوز لحاجة تانية:
# → ابحث عن الـ device اللي حاجزاه
# → تحقق من الـ DT إن مفيش تعارض في الـ pin assignments

# مثال على output من pinconf-pins:
pin 24 (A5): drive-strength:4 bias-pull-up
# يعني:
# drive-strength = 4mA
# pull-up resistor مفعّل
```

```bash
# أوامر إضافية للمواقف الصعبة

# لو الـ device بيرجع -EPROBE_DEFER باستمرار
dmesg -T | grep -i "eprobe_defer\|pinctrl\|probe" | tail -30

# لو مش عارف اسم الـ pin controller
ls /sys/kernel/debug/pinctrl/

# لو محتاج تعرف الـ base address للـ pin controller من الـ DT
cat /proc/device-tree/soc/*/compatible | strings | grep pinctrl

# تحقق من الـ clocks المطلوبة للـ pin controller
cat /proc/device-tree/soc/pinctrl@*/clocks 2>/dev/null | hexdump -C

# راقب الـ pinctrl errors في real-time
dmesg -w | grep -i pinctrl
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART بيشتغلش بعد Sleep على STM32MP1 في Industrial Gateway

#### العنوان
**الـ UART TX بيتحول لـ "GPIO mode" غلط أثناء suspend على STM32MP1**

#### السياق
شركة بتبني industrial gateway بـ STM32MP1 (Cortex-A7 + M4). الـ gateway بتتكلم مع external sensors عن طريق UART4. المنتج المفروض يدخل في sleep بعد فترة خمول عشان يوفر طاقة — وده متطلب صريح في الـ spec.

#### المشكلة
بعد أول suspend/resume cycle، الـ UART4 بيوقف التواصل مع الـ sensors خالص. الـ TX line بتفضل high impedance. اللوج بيظهر:

```
uart4: TX stuck HIGH after resume, sensors not responding
```

المهندس بيفحص الـ hardware بالـ oscilloscope ويلاقي الـ TX pin مش بيتحرك خالص حتى لو الـ UART driver بيكتب data.

#### التحليل
المشكلة في إزاي الـ pinctrl states اتعملت في الـ device tree. الـ file الأصلي بيوضح إن الـ standard states هي: `default`, `init`, `sleep`, `idle`.

الـ driver بيعمل:
```c
/* في foo_suspend() */
pinctrl_pm_select_sleep_state(dev);

/* في foo_resume() */
pinctrl_pm_select_init_state(dev);
/* ... resume logic ... */
pinctrl_pm_select_default_state(dev);
```

المهندس راح فحص الـ DT:

```dts
/* الـ DT اللي فيه المشكلة */
&uart4 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart4_pins_default>;
    pinctrl-1 = <&uart4_pins_sleep>;
};
```

وفي الـ pinctrl node:
```dts
uart4_pins_sleep: uart4-sleep {
    pins = "PA11", "PA12";
    function = "gpio";          /* غلط — بيحول الـ pin لـ GPIO مش low-power */
    drive-push-pull;
    bias-disable;
    /* مفيش output-low ! */
};
```

الـ documentation بيوضح صح إن "GPIO mode" في الـ datasheet مش لازم يتعامل معاه كـ GPIO Linux API. المفروض يُعامَل كـ **pin config setting** باستخدام `PIN_CONFIG_LEVEL`:

> "The solution is to not think that what the datasheet calls 'GPIO mode' has to be handled by the `<linux/gpio/consumer.h>` interface."

الـ `pinctrl_select_state()` بتبعت الـ call لـ `pinmux_ops->set_mux()` اللي بيكتب في الـ MUX register — لما اتكتب الـ gpio function من غير output value، الـ pin اتسبت floating.

#### الحل
تعديل الـ DT عشان الـ sleep state تحط الـ pin في output-low:

```dts
uart4_pins_sleep: uart4-sleep {
    pins = "PA11";               /* TX only needs grounding */
    function = "gpio";
    output-low;                  /* PIN_CONFIG_LEVEL = 0 */
    bias-disable;
};
```

أو باستخدام `PIN_CONF_PACKED` في الـ board file:

```c
static unsigned long uart4_sleep_mode[] = {
    PIN_CONF_PACKED(PIN_CONFIG_OUTPUT, 0),  /* drive low during sleep */
};

static struct pinctrl_map stm32_uart4_map[] __initdata = {
    PIN_MAP_MUX_GROUP("uart4", PINCTRL_STATE_DEFAULT,
                      "pinctrl-stm32mp1", "uart4_grp", "uart4"),
    PIN_MAP_CONFIGS_PIN("uart4", PINCTRL_STATE_SLEEP,
                        "pinctrl-stm32mp1", "UART4_TX",
                        uart4_sleep_mode),
};
```

#### الدرس المستفاد
"GPIO mode" في الـ datasheet = electrical configuration، مش Linux GPIO API. لما تشوف الـ datasheet بيقول "set pin to GPIO mode during sleep"، الترجمة الصح هي `PIN_CONFIG_OUTPUT` أو `PIN_CONFIG_LEVEL` في الـ pinctrl map — مش `gpiod_get()`. الـ pin لازم تتحكم فيه من خلال `pinctrl_select_state()` بـ sleep config صح.

---

### السيناريو 2: SPI Flash مش بتـ detect على RK3562 Android TV Box

#### العنوان
**الـ SPI0 مش شغال على RK3562 بسبب pin conflict مع SDMMC**

#### السياق
TV box بـ RK3562 SoC. الـ board فيها SPI NOR flash (25 series) متوصل على SPI0، وفي نفس الوقت الـ designer حط SD card slot. الـ software team بيعمل bring-up وبيلاقي إن الـ SPI flash مش بتـ detect خالص.

#### المشكلة
الـ kernel log بيظهر:

```
spi-rockchip ff1d0000.spi: Failed to request mux
pinctrl: pin gpio2-8 already used by mmc0
spi-rockchip ff1d0000.spi: probe failed: -EBUSY
```

#### التحليل
الـ pinmux subsystem بيمنع الـ conflict تلقائيًا. الـ documentation بيوضح:

> "PINS for a certain FUNCTION using a certain PIN GROUP on a certain PIN CONTROLLER are provided on a first-come first-serve basis, so if some other device mux setting or GPIO pin request has already taken your physical pin, you will be denied the use of it."

الـ flow اللي حصل:
1. الـ `mmc0` probe أول وطلب pins بتاعته → نجح، الـ pinmux core سجّل الـ pins دي مأخوذة
2. الـ `spi0` probe تاني وطلب نفس الـ pins → الـ `pinmux_request_gpio()` أو `set_mux()` لقي conflict

المهندس فحص الـ debugfs:

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinmux-pins | grep gpio2-8
```

النتيجة:
```
pin 72 (gpio2-8): mux:mmc0 gpio:(none) hog:no
```

وفحص الـ pingroups:
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pingroups
```

اكتشف إن الـ `spi0_grp` و `sdmmc0_grp` فيهم overlap في pins 70-73.

الـ DT كان فيه:
```dts
&spi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&spi0m1_pins>;  /* المهندس اختار المجموعة الغلط */
    ...
};

&sdmmc0 {
    pinctrl-names = "default";
    pinctrl-0 = <&sdmmc0_pins>;
    ...
};
```

الـ RK3562 عنده SPI0 بـ multiplexed groups: `spi0m0_pins` (conflict-free) و `spi0m1_pins` (بتعارض مع SDMMC).

#### الحل
الـ documentation بيشرح إن function واحدة ممكن تتعمل map لـ pin groups مختلفة. الحل هو اختيار الـ group التانية:

```dts
/* تعديل الـ DT */
&spi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&spi0m0_pins>;  /* المجموعة البديلة — conflict-free */
    cs-gpios = <&gpio3 RK_PC2 GPIO_ACTIVE_LOW>;
    status = "okay";
};
```

للتأكيد بعد الإصلاح:
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinmux-pins | grep -E "spi|mmc"
# المفروض كل واحد فيهم pins مختلفة
```

#### الدرس المستفاد
لما تيجي مشكلة `-EBUSY` في الـ probe، أول حاجة تعملها هي فحص `/sys/kernel/debug/pinctrl/<controller>/pinmux-pins` و `pingroups`. الـ pinmux subsystem بيمنع الـ conflicts أوتوماتيك، بس المهندس هو المسؤول إنه يختار الـ pin group الصح اللي متعارضش مع باقي الـ peripherals الموجودة على الـ board.

---

### السيناريو 3: I2C بيديني noise على Allwinner H616 في IoT Sensor Node

#### العنوان
**الـ I2C SCL بيعمل glitches بسبب pull-up مش متظبط على H616**

#### السياق
IoT sensor node بـ Allwinner H616. الـ board بتقيس temperature وhumidity وbaro pressure عن طريق 3 sensors على I2C2. الـ bus بيشتغل بس بـ high error rate — كل شوية packet loss وretry.

#### المشكلة
بالـ logic analyzer، الـ SCL line بتعمل ringing بعد كل rising edge. الـ pull-up resistors على الـ board قيمتهم 4.7kΩ — طبيعية. المهندس بيشك في الـ pin configuration.

#### التحليل
الـ H616 pinctrl driver بيدعم `PIN_CONFIG_DRIVE_STRENGTH` وكمان الـ slew rate control. الـ documentation بيوضح إن الـ pin configuration بتتعمل عن طريق `pinconf_ops`:

```c
static int foo_pin_config_set(struct pinctrl_dev *pctldev,
                              unsigned int offset,
                              unsigned long config)
{
    struct my_conftype *conf = (struct my_conftype *) config;

    switch (conf) {
        case PLATFORM_X_PULL_UP:
        /* set register bits */
        break;
    }
}
```

الـ configuration بتتعدى عن طريق الـ mapping table. المهندس فحص الـ DT:

```dts
i2c2_pins: i2c2-pins {
    pins = "PI0", "PI1";
    function = "i2c2";
    drive-strength = <40>;    /* 40mA — كتير جداً على I2C */
    bias-pull-up;
    /* مفيش slew rate control */
};
```

الـ drive strength العالية بتخلي الـ output transistor بيشحن الـ capacitance السريعة جداً → ringing.

فحص الـ debugfs للـ pin config الحالي:
```bash
cat /sys/kernel/debug/pinctrl/pio/pinconf-pins | grep -A3 "pin 224\|pin 225"
```

النتيجة:
```
pin 224 (PI0): drive-strength:40mA, bias-pull-up
pin 225 (PI1): drive-strength:40mA, bias-pull-up
```

#### الحل
تعديل الـ `pinconf_ops` أو الـ DT لتخفيض الـ drive strength:

```dts
i2c2_pins: i2c2-pins {
    pins = "PI0", "PI1";
    function = "i2c2";
    drive-strength = <10>;     /* تخفيض لـ 10mA — مناسب لـ I2C */
    bias-pull-up;
};
```

أو في الـ board file باستخدام `PIN_MAP_CONFIGS_GROUP`:

```c
static unsigned long i2c2_grp_configs[] = {
    PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 10),
    PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 0),
};

static struct pinctrl_map h616_map[] __initdata = {
    PIN_MAP_MUX_GROUP("i2c2", PINCTRL_STATE_DEFAULT,
                      "pio", "i2c2_grp", "i2c2"),
    PIN_MAP_CONFIGS_GROUP("i2c2", PINCTRL_STATE_DEFAULT,
                          "pio", "i2c2_grp", i2c2_grp_configs),
};
```

للتحقق بعد التعديل:
```bash
cat /sys/kernel/debug/pinctrl/pio/pinconf-pins | grep -A3 "pin 224\|pin 225"
# المفروض تظهر drive-strength:10mA
```

#### الدرس المستفاد
الـ pinconf مش بس `pull-up/pull-down`. الـ `drive-strength` والـ slew rate بيأثروا على الـ signal integrity بشكل مباشر، خصوصًا على الـ open-drain buses زي I2C. دايمًا ابدأ بـ minimum drive strength وازيدها لو محتاج.

---

### السيناريو 4: USB OTG مش بيتعرف على الـ Device على i.MX8M في Automotive ECU

#### العنوان
**الـ USB OTG ID pin مش متظبط كـ GPIO بسبب missing `gpio_request_enable` على i.MX8M**

#### السياق
automotive ECU بـ NXP i.MX8M Mini. الـ ECU بيحتاج يشتغل USB OTG: يشتغل host لما فيه device متوصل، ويشتغل device لما اتوصل بـ diagnostic PC. الـ OTG ID detection بيحتاج pin GPIO.

#### المشكلة
الـ USB stack مش بيشوف التغيير في الـ ID pin. الـ `usb_phy_vbus_on_used` بيفضل مش متغير. الـ driver بيقرأ الـ GPIO دايماً بنفس القيمة.

#### التحليل
الـ flow الصح هو:
1. الـ driver بيعمل `devm_gpiod_get()` على الـ ID pin
2. الـ gpiolib بتعمل `.request()` اللي بتستدعي `pinctrl_gpio_request()`
3. الـ pinctrl core بتدور على الـ pin controller المناسب عن طريق الـ **GPIO range** اللي اتسجل بـ `pinctrl_add_gpio_range()`
4. لو الـ pinctrl driver معملش `gpio_request_enable()` أو معملش alternative "gpioN" function — الـ pin بيفضل في وضعه القديم

الـ documentation بيوضح:

> "For this reason there are two functions a pin control driver can implement to enable only GPIO on an individual pin: `.gpio_request_enable()` and `.gpio_disable_free()`."

فحص الـ i.MX8M pinctrl driver:
```bash
grep -r "gpio_request_enable" drivers/pinctrl/freescale/
```

الـ driver عنده الـ implementation — المشكلة في الـ DT. الـ GPIO range مش اتسجل صح:

```bash
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/gpio-ranges
```

النتيجة كانت فاضية — الـ `pinctrl_add_gpio_range()` مش اتستدعت.

المهندس فحص الـ DT وقاله إن الـ `gpio-ranges` property ناقصة في الـ GPIO node:

```dts
/* الـ DT المكسور */
&gpio1 {
    /* مفيش gpio-ranges ! */
    status = "okay";
};
```

#### الحل
الـ documentation بيوضح إن الـ `pinctrl_add_gpio_range()` من الـ pinctrl driver **deprecated** وإن الحل الصح هو الـ DT binding:

```dts
/* الـ DT الصح */
&gpio1 {
    gpio-ranges = <&iomuxc 0 0 32>;  /* map GPIO1[0..31] → iomuxc pins [0..31] */
    status = "okay";
};
```

وبعد التعديل:
```bash
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/gpio-ranges
# يظهر: GPIO_RANGE[0] id=0 base=0 pin_base=0 npins=32 group="gpio1" name="gpio1"
```

#### الدرس المستفاد
الـ `gpio-ranges` في الـ DT هي الطريقة الحديثة لربط الـ GPIO controller بالـ pinctrl controller. بدونها، الـ `pinctrl_gpio_request()` مش بتعرف هي المفروض تكلم أنهي pin controller. لما GPIO pin مش بيتصرف كـ GPIO، فحص `gpio-ranges` أول حاجة.

---

### السيناريو 5: HDMI مش بيظهر صورة على AM62x في Custom Display Board

#### العنوان
**الـ HDMI DDC (I2C) فاشل على AM62x بسبب pin hogging غلط**

#### السياق
custom display board بـ Texas Instruments AM62x SoC. البورد مصممة لـ digital signage. الـ HDMI output مش بيشتغل — الشاشة بتقول "no signal" وكمان الـ EDID reading بيفشل.

#### المشكلة
الـ `drm` subsystem بيفشل في قراءة الـ EDID من الشاشة عن طريق الـ DDC channel (اللي هو I2C على الـ HDMI cable):

```
tilcdc: Failed to get EDID from HDMI monitor
i2c i2c-3: timeout waiting for bus ready
```

#### التحليل
الـ DDC channel بيستخدم I2C3 على الـ AM62x. المهندس فحص الـ pinmux:

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-am62/pinmux-pins | grep -E "I2C3|hog"
```

النتيجة:
```
pin 83 (I2C3_SCL): mux:pinctrl-am62 gpio:(none) hog:yes
pin 84 (I2C3_SDA): mux:pinctrl-am62 gpio:(none) hog:yes
```

الـ `hog:yes` معناه إن الـ pin اتأخد من الـ pin controller نفسه وقت الـ registration. الـ documentation بيشرح الـ hogging:

> "Pin control map entries can be hogged by the core when the pin controller is registered. This means that the core will attempt to call `pinctrl_get()`, `pinctrl_lookup_state()` and `pinctrl_select_state()` on it immediately after the pin control device has been registered."

> "This occurs for mapping table entries where the client device name is equal to the pin controller device name, and the state name is `PINCTRL_STATE_DEFAULT`."

المهندس لقى في الـ board file:

```c
/* الـ mapping الغلط — بيعمل hog على I2C3 بـ wrong function */
{
    .dev_name = "pinctrl-am62",          /* نفس اسم الـ pin controller = hog */
    .name = PINCTRL_STATE_DEFAULT,
    .type = PIN_MAP_TYPE_MUX_GROUP,
    .ctrl_dev_name = "pinctrl-am62",
    .data.mux.function = "i2c3",         /* صح */
    /* المشكلة: الـ pin config مش ضايف pull-up للـ I2C open-drain */
},
```

وبعدين الـ `drm` driver بيعمل:
```c
p = devm_pinctrl_get(&hdmi_dev);
s = pinctrl_lookup_state(p, PINCTRL_STATE_DEFAULT);
ret = pinctrl_select_state(p, s);
```

لما الـ `drm` driver طلب الـ pins، الـ pinmux core رفض لأنهم مأخودين بالـ hog. الـ `pinctrl_select_state()` رجعت `-EBUSY`.

#### الحل
خطوتين:

**أولًا**: إزالة الـ hog الغلط من الـ board file وتغيير الـ `dev_name` لاسم الـ HDMI device:

```c
/* الـ mapping الصح */
{
    .dev_name = "hdmi-display",           /* اسم الـ device الفعلي مش الـ controller */
    .name = PINCTRL_STATE_DEFAULT,
    .type = PIN_MAP_TYPE_MUX_GROUP,
    .ctrl_dev_name = "pinctrl-am62",
    .data.mux.function = "i2c3",
},
```

**ثانيًا**: إضافة الـ pull-up config للـ I2C DDC pins:

```c
static unsigned long ddc_pin_configs[] = {
    PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 0),
    PIN_CONF_PACKED(PIN_CONFIG_DRIVE_OPEN_DRAIN, 0),
};

static struct pinctrl_map am62_hdmi_map[] __initdata = {
    PIN_MAP_MUX_GROUP("hdmi-display", PINCTRL_STATE_DEFAULT,
                      "pinctrl-am62", "i2c3_grp", "i2c3"),
    PIN_MAP_CONFIGS_PIN("hdmi-display", PINCTRL_STATE_DEFAULT,
                        "pinctrl-am62", "I2C3_SCL", ddc_pin_configs),
    PIN_MAP_CONFIGS_PIN("hdmi-display", PINCTRL_STATE_DEFAULT,
                        "pinctrl-am62", "I2C3_SDA", ddc_pin_configs),
};
```

للتحقق:
```bash
# التأكد إن الـ pins مش hog بعد الإصلاح
cat /sys/kernel/debug/pinctrl/pinctrl-am62/pinmux-pins | grep "I2C3"
# المفروض: hog:no

# التأكد إن الـ hdmi device أخد الـ pins
cat /sys/kernel/debug/pinctrl/pinctrl-am62/pinmux-pins | grep "i2c3"
# المفروض: مالك الـ pin هو hdmi-display device
```

#### الدرس المستفاد
الـ **system pin control hogging** بيحصل لما `dev_name == ctrl_dev_name` في الـ mapping. ده مفيد للـ pins اللي لازم تتأخد قبل أي device يـ probe (زي power rails)، بس لو حصل بالغلط على pins بتاعة device تاني، النتيجة هي `-EBUSY` عند الـ probe. دايمًا تحقق من الـ `pinmux-pins` debugfs file وابحث عن `hog:yes` لو device رفض يـ probe بـ pin conflict error.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المسار | المحتوى |
|--------|---------|
| `Documentation/driver-api/pin-control.rst` | التوثيق الرئيسي للـ pinctrl subsystem — المرجع الأول دايمًا |
| `Documentation/devicetree/bindings/pinctrl/` | الـ DT bindings لكل controller |
| `Documentation/devicetree/bindings/gpio/gpio.txt` | ربط الـ pinctrl بالـ GPIO عبر الـ device tree |
| `include/linux/pinctrl/pinctrl.h` | الـ API الأساسية للـ consumer و driver |
| `include/linux/pinctrl/pinmux.h` | الـ pinmux ops |
| `include/linux/pinctrl/pinconf.h` | الـ pinconf ops |
| `include/linux/pinctrl/pinconf-generic.h` | الـ generic pin config parameters |
| `include/linux/pinctrl/machine.h` | الـ mapping table macros |
| `include/linux/pinctrl/consumer.h` | الـ consumer API |
| `drivers/pinctrl/core.c` | قلب الـ subsystem — core implementation |
| `drivers/pinctrl/pinmux.c` | منطق الـ mux |
| `drivers/pinctrl/pinconf.c` | منطق الـ config |
| `drivers/pinctrl/devicetree.c` | دعم الـ device tree |

---

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأهم لفهم تطور الـ kernel — كل مقال فيه بيشرح "ليه" مش بس "إيه".

| المقال | الأهمية |
|--------|---------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | المقال الرئيسي اللي شرح الـ subsystem وقت إضافته في 3.1 |
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | أول patch series لإنشاء الـ subsystem |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | إضافة التوثيق الرسمي الأول |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/467269/) | إضافة الـ generic pinconf layer |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | توسيع واجهة الـ pinconf |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | إصدار v7 من الـ patch series قبل الدمج |
| [drivers: create a pin control subsystem v8](https://lwn.net/Articles/460768/) | إصدار v8 قبل الـ merge النهائي |
| [Pin page control subsystem](https://lwn.net/Articles/563239/) | تحديثات لاحقة على الـ subsystem |
| [pinctrl: introduce the concept of a GPIO pin function category](https://lwn.net/Articles/1031226/) | تطوير حديث — إدخال مفهوم الـ GPIO function category |
| [drivers/pinctrl: Add the concept of an "init" state](https://lwn.net/Articles/615322/) | إضافة الـ init state للـ lifecycle management |

---

### مقالات eLinux.org

**الـ eLinux** مركز موارد الـ embedded Linux — فيه presentations ووثائق عملية.

| المصدر | المحتوى |
|--------|---------|
| [Introduction to pin muxing and GPIO control under Linux (ELC 2021)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) | presentation شامل من مؤتمر ELC عن الـ pinmux والـ GPIO |
| [Pin Control and GPIO Update](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) | ملخص تحديثات الـ pin control والـ GPIO |
| [Tests:i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) | مثال عملي: استخدام الـ pinctrl مع الـ i2c-demux |
| [EBC Device Trees](https://elinux.org/EBC_Device_Trees) | شرح الـ device tree للـ BeagleBone مع أمثلة pinctrl |

---

### مقالات KernelNewbies.org

الصفحات دي بتوثق التغييرات في كل إصدار kernel — مهمة لمتابعة تطور الـ subsystem:

| الإصدار | رابط |
|---------|------|
| Linux 6.13 | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) |
| Linux 6.15 | [kernelnewbies.org/Linux_6.15](https://kernelnewbies.org/Linux_6.15) |
| Linux 6.17 | [kernelnewbies.org/Linux_6.17](https://kernelnewbies.org/Linux_6.17) |
| Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| جميع الإصدارات | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) |

---

### الـ Mailing List

الـ pinctrl بيتطور عبر **linux-gpio@vger.kernel.org** — الـ subsystem بيُسمى "pin control" لكن الـ mailing list مشترك مع الـ GPIO.

| المصدر | الرابط |
|--------|--------|
| LKML Archive — pinctrl patches | [lkml.org](https://lkml.org/) — ابحث عن `[PATCH] pinctrl` |
| Patchwork — pinctrl series تاريخية | [patches.linaro.org](https://patches.linaro.org/project/lkml/patch/1370439873-30053-1-git-send-email-linus.walleij@stericsson.com/) |
| GIT PULL pinctrl v6.19 | [lkml.org/lkml/2025/12/8/736](https://lkml.org/lkml/2025/12/8/736) |
| Linaro pinctrl git tree | [git.linaro.org/people/linus.walleij/linux-pinctrl.git](https://git.linaro.org/people/linus.walleij/linux-pinctrl.git) |

> الـ maintainer الأساسي هو **Linus Walleij** من Linaro — الـ patches بتيجي على `linux-gpio@vger.kernel.org` وبتتراجع عبر `drivers/pinctrl` في شجرة الـ kernel.

---

### Kernel Commits المهمة

| الحدث | الوصف |
|-------|-------|
| Linux 3.1 (2011) | أول دمج للـ pinctrl subsystem في الـ mainline |
| Linux 3.2 | إضافة الـ pinconf generic layer |
| Linux 3.14 | إضافة الـ `init` و `sleep` و `idle` states |
| Linux 4.x+ | دعم الـ device tree binding بشكل كامل |
| Linux 6.x | إضافة GPIO pin function categories |

```bash
# عشان تشوف تاريخ commits الـ pinctrl core:
git log --oneline drivers/pinctrl/core.c

# أول commit في الـ subsystem:
git log --oneline --follow -- drivers/pinctrl/ | tail -20

# تغييرات إصدار معين (مثلاً v6.1):
git log --oneline v6.0..v6.1 -- drivers/pinctrl/
```

---

### الكتب الموصى بها

#### Linux Device Drivers (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول ذات الصلة:** Chapter 9 (Communicating with Hardware) — أسلوب كتابة الـ drivers
- **ملاحظة:** الكتاب قديم (2005) ومش بيغطي الـ pinctrl مباشرةً، لكن الأساسيات اللي بيشرحها لازمة
- **متاح مجانًا:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (الطبعة الثالثة)
- **الفصول ذات الصلة:**
  - Chapter 17: Devices and Modules — فهم device model
  - Chapter 13: The Virtual Filesystem — فهم sysfs وdebugfs
- **الأهمية:** شرح الـ device model اللي بيعتمد عليه الـ pinctrl

#### Embedded Linux Primer — Christopher Hallinan
- **الفصول ذات الصلة:**
  - Chapter 15: Debugging Embedded Linux Applications
  - فصول الـ BSP وboard bring-up
- **الأهمية:** سياق عملي لاستخدام الـ pinctrl في الـ embedded systems

#### Linux Kernel Programming — Kaiwan N. Billimoria
- غطى الـ device tree وthe platform driver model بشكل حديث
- مناسب أكتر من LDD3 للـ topics الحديثة زي pinctrl

---

### مصادر إضافية

| المصدر | الرابط |
|--------|--------|
| توثيق kernel.org الرسمي | [kernel.org/doc/html/latest/driver-api/pinctl.html](https://www.kernel.org/doc/html/latest/driver-api/pinctl.html) |
| توثيق v4.14 (مستقر) | [kernel.org/doc/html/v4.14/driver-api/pinctl.html](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html) |
| Linux device driver development: pinctrl (embedded.com) | [embedded.com/linux-device-driver-development-the-pin-control-subsystem](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) |
| STM32 Pinctrl Overview | [wiki.st.com/stm32mpu/wiki/Pinctrl_overview](https://wiki.st.com/stm32mpu/wiki/Pinctrl_overview) |
| Android Kernel pinctrl.txt | [android.googlesource.com](https://android.googlesource.com/kernel/common/+/bcmdhd-3.10/Documentation/pinctrl.txt) |

---

### Search Terms للبحث عن معلومات أكتر

```
# بحث عام
linux kernel pinctrl subsystem
linux pinmux driver tutorial
linux pin control device tree binding
pinctrl_register_and_init example
devm_pinctrl_get_select example

# بحث في LWN
site:lwn.net pinctrl
site:lwn.net pinmux linux

# بحث في الكود
site:elixir.bootlin.com pinctrl_ops
site:elixir.bootlin.com pinmux_ops
site:elixir.bootlin.com pinconf_ops

# بحث في الـ mailing list
site:lkml.org "[PATCH] pinctrl"
site:lore.kernel.org pinctrl

# drivers حقيقية للاستلهام
drivers/pinctrl/pinctrl-bcm2835.c   # Raspberry Pi
drivers/pinctrl/qcom/               # Qualcomm SoCs
drivers/pinctrl/samsung/            # Samsung Exynos
drivers/pinctrl/stm32/              # STMicroelectronics
drivers/pinctrl/sunxi/              # Allwinner
```

---

### Elixir Cross Referencer

الأداة دي بتخليك تتصفح الـ kernel source code online مع links بين الـ symbols:

```
# مثال: شوف كل الـ implementations لـ pinmux_ops
https://elixir.bootlin.com/linux/latest/ident/pinmux_ops

# أو الـ pinctrl_desc struct
https://elixir.bootlin.com/linux/latest/ident/pinctrl_desc
```

**الرابط:** [elixir.bootlin.com](https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl)
## Phase 8: Writing simple module

### الفكرة

**`pinctrl_select_state()`** هي الدالة اللي بتتعمل call كل ما device بتغير حالة الـ pin control بتاعتها (مثلاً من `default` لـ `sleep`). دي نقطة مثالية للـ **kprobe** عشان نشوف مين بيغير إيه وإمتى.

---

### الـ Module كامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * pinctrl_select_state kprobe monitor
 *
 * Hooks pinctrl_select_state() to log every pin-state switch in the system.
 * Useful for debugging which device is changing its pin configuration and when.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/device.h>

/* Internal pinctrl structs — we only need their names, not full definitions.
 * The kernel exposes the dev pointer inside struct pinctrl, but the struct
 * is opaque (private to core). We access it via the pinctrl_get_device()
 * helper or by reading the dev field directly using the public header layout.
 *
 * Because struct pinctrl is opaque, we define a minimal shadow that mirrors
 * just the first field we need (struct device *dev), which has been stable
 * across kernels for years. */
struct pinctrl_shadow {
    struct device *dev;  /* first field in struct pinctrl — see core.c */
};

/* struct pinctrl_state is also opaque; the 'name' field is the first member */
struct pinctrl_state_shadow {
    struct list_head node;   /* linked into pinctrl->states list */
    const char *name;        /* state name, e.g. "default", "sleep" */
    /* ... rest we don't need */
};

/*
 * pre-handler: called just before pinctrl_select_state() executes.
 *
 * Prototype of the hooked function:
 *   int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state)
 *
 * The kprobe pre-handler receives the CPU register state at the call site.
 * On x86-64: rdi = first arg (p), rsi = second arg (state).
 * On ARM64:  x0  = first arg (p), x1  = second arg (state).
 * regs_get_kernel_argument() is architecture-agnostic and handles both.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* بنجيب الـ arguments من الـ registers بطريقة portable */
    struct pinctrl_shadow      *pinctrl = (struct pinctrl_shadow *)
                                          regs_get_kernel_argument(regs, 0);
    struct pinctrl_state_shadow *state  = (struct pinctrl_state_shadow *)
                                          regs_get_kernel_argument(regs, 1);

    /* null-check عشان منكسرش الـ kernel لو حصل حاجة غريبة */
    if (!pinctrl || !state)
        return 0;

    /*
     * dev_name() بترجع اسم الـ device من struct device بطريقة آمنة.
     * state->name بيديك اسم الحالة الجديدة (default / sleep / idle ...).
     */
    pr_info("pinctrl_probe: device [%s] switching to state [%s]\n",
            pinctrl->dev ? dev_name(pinctrl->dev) : "<unknown>",
            state->name  ? state->name             : "<unnamed>");

    return 0; /* 0 = تابع تنفيذ الدالة الأصلية بشكل طبيعي */
}

/* post-handler: بيتعمل call بعد ما pinctrl_select_state خلصت بنجاح */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* ممكن نقرأ الـ return value من ax/x0 لو محتاجين نعرف نجحت ولا لأ */
    long retval = regs_return_value(regs);

    if (retval != 0)
        pr_info("pinctrl_probe: state switch FAILED (ret=%ld)\n", retval);
}

/* تعريف الـ kprobe struct — بنحدد اسم الدالة بالـ symbol name */
static struct kprobe kp = {
    .symbol_name = "pinctrl_select_state",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

static int __init pinctrl_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("pinctrl_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("pinctrl_probe: kprobe planted at %s (%px)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit pinctrl_probe_exit(void)
{
    /*
     * لازم نعمل unregister قبل ما الـ module يتـremove —
     * لو سبنا الـ kprobe active والـ module اترفع، الكود
     * اللي الـ kprobe بيشاور عليه هيبقى invalid وهيحصل kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("pinctrl_probe: kprobe removed from %s\n", kp.symbol_name);
}

module_init(pinctrl_probe_init);
module_exit(pinctrl_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc-lab");
MODULE_DESCRIPTION("kprobe on pinctrl_select_state to trace pin-state switches");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|--------|
| `linux/kprobes.h` | بيوفر `struct kprobe`، `register_kprobe()`، `regs_get_kernel_argument()` |
| `linux/device.h` | عشان `dev_name()` اللي بترجع اسم الـ device من `struct device *` |
| `linux/module.h` | الـ macros الأساسية للـ kernel module |

#### الـ Shadow Structs

**`pinctrl_shadow`** و **`pinctrl_state_shadow`** هما تعريفات مبسطة بتطابق أول fields في الـ structs الحقيقية اللي في `core.c`. اتعملوا بالشكل ده عشان الـ structs الأصلية `opaque` ومش exported للـ headers العامة — بس الـ memory layout ثابت وبنستغله بشكل آمن.

#### الـ pre_handler

`regs_get_kernel_argument(regs, 0)` بيجيب الـ argument الأولاني (الـ `pinctrl *`) من الـ registers بصرف النظر عن الـ architecture (x86-64 ولا ARM64). بنطبع اسم الـ device واسم الـ state الجديدة — ده بيديك visibility كاملة على أي device في النظام بتغير pin configuration بتاعتها.

#### الـ post_handler

بنتحقق من الـ return value بعد ما الدالة خلصت. لو الـ state switch فشل (مثلاً الـ pinctrl driver رجّع error)، بنسجل الـ failure — مفيد في الـ debugging لو hardware مش responsive.

#### الـ module_exit وأهمية الـ unregister

لو منعملناش `unregister_kprobe()` في الـ exit، الـ kprobe handler هيظل موجود في الـ kernel وبيشاور على كود اترفع — أي call لـ `pinctrl_select_state` بعد كده هيعدي على عنوان invalid ويطلع **kernel panic**. عشان كده الـ unregister شرط أساسي مش اختياري.

---

### Makefile

```makefile
obj-m += pinctrl_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

### تجربة الـ module

```bash
# تحميل الـ module
sudo insmod pinctrl_probe.ko

# مشاهدة الـ logs في real-time
sudo dmesg -w | grep pinctrl_probe

# لو عندك device بيعمل suspend/resume، هتشوف حاجة زي:
# pinctrl_probe: device [soc:pinctrl@11000000] switching to state [sleep]
# pinctrl_probe: device [soc:pinctrl@11000000] switching to state [default]

# إزالة الـ module
sudo rmmod pinctrl_probe
```

> على أي SoC حقيقي (Raspberry Pi، BeagleBone، إلخ) هتشوف عشرات الـ devices بتعمل state switch أثناء الـ boot وعند الـ suspend.
