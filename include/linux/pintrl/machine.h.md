## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

الملف `include/linux/pinctrl/machine.h` ينتمي لـ **PIN CONTROL SUBSYSTEM** — المسؤول عنه Linus Walleij، والـ mailing list هو `linux-gpio@vger.kernel.org`، والـ tree الرسمي على kernel.org هو `linusw/linux-pinctrl.git`.

---

### المشكلة اللي بيحلها الـ pinctrl subsystem

تخيل إن عندك لوحة إلكترونية (مثلاً Raspberry Pi أو أي SoC). الـ **SoC** (System on Chip) بيكون فيه مئات الـ **pins** — طرفيات مادية صغيرة على الشريحة. كل pin ممكن يشتغل بأكتر من وظيفة:

```
Pin رقم 5 → ممكن يكون:
  - GPIO عادي (input/output)
  - TX لـ UART
  - SDA لـ I2C
  - MOSI لـ SPI
```

ده اللي بيتسمى **pin muxing** — الـ hardware عنده multiplexer داخلي بيخليك تختار وظيفة الـ pin.

مش بس كده، كل pin ممكن تضبطه كمان:
- **pull-up / pull-down** (مقاومة داخلية)
- **drive strength** (قوة التيار)
- **slew rate** (سرعة التحويل)
- **open drain / push-pull**

ده اللي بيتسمى **pin configuration**.

---

### القصة الكاملة — مين بيتكلم مع مين

```
┌─────────────────────────────────────────────────────────┐
│                    BOARD / MACHINE                       │
│  (الـ board-level code أو Device Tree)                  │
│                                                          │
│   "عايز UART0 يشتغل على Pin5 و Pin6"                    │
│   "عايز Pin5 يكون pull-up"                               │
└───────────────────┬─────────────────────────────────────┘
                    │  pinctrl_map table
                    ▼
┌─────────────────────────────────────────────────────────┐
│              PIN CONTROL CORE                            │
│         drivers/pinctrl/core.c                          │
│                                                          │
│  بياخد الـ mapping table ويحولها لطلبات للـ driver      │
└───────────────────┬─────────────────────────────────────┘
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
┌─────────────────┐   ┌─────────────────┐
│  PIN MUX OPS    │   │  PIN CONF OPS   │
│  (pinmux.h)     │   │  (pinconf.h)    │
│                 │   │                 │
│ اختيار الوظيفة │   │  ضبط الخصائص   │
└────────┬────────┘   └────────┬────────┘
         │                     │
         └──────────┬──────────┘
                    ▼
┌─────────────────────────────────────────────────────────┐
│           HARDWARE PIN CONTROLLER DRIVER                 │
│    (مثلاً: drivers/pinctrl/qcom/pinctrl-msm.c)         │
│                                                          │
│  بيكتب في registers الـ SoC الفعلية                    │
└─────────────────────────────────────────────────────────┘
```

---

### دور `machine.h` تحديداً — "لغة التواصل بين الـ board والـ core"

الملف ده هو الـ **machine interface** — يعني الواجهة اللي بيتكلم بيها كود الـ board (أو الـ platform code) مع الـ pinctrl core.

فكر فيه زي **قائمة الطلبات** اللي بتكتبها قبل ما توصي. كل سطر في القائمة بيقول:

> "الجهاز X، لما يكون في حالة Y، يستخدم الـ pin group Z بالوظيفة W، وبالـ config ده."

ده بالظبط اللي بتعبر عنه **`struct pinctrl_map`**.

---

### الـ Structures الأساسية في الملف

#### 1. `enum pinctrl_map_type` — نوع الطلب

```c
enum pinctrl_map_type {
    PIN_MAP_TYPE_INVALID,        /* غلط */
    PIN_MAP_TYPE_DUMMY_STATE,    /* state وهمي، مفيش hardware */
    PIN_MAP_TYPE_MUX_GROUP,      /* اختيار وظيفة لـ group من الـ pins */
    PIN_MAP_TYPE_CONFIGS_PIN,    /* ضبط config لـ pin واحد */
    PIN_MAP_TYPE_CONFIGS_GROUP,  /* ضبط config لـ group كامل */
};
```

#### 2. `struct pinctrl_map_mux` — بيان الـ mux

```c
struct pinctrl_map_mux {
    const char *group;    /* اسم الـ pin group، مثلاً "uart0_grp" */
    const char *function; /* الوظيفة المطلوبة، مثلاً "uart0" */
};
```

#### 3. `struct pinctrl_map_configs` — بيان الـ configuration

```c
struct pinctrl_map_configs {
    const char *group_or_pin; /* اسم الـ pin أو الـ group */
    unsigned long *configs;   /* مصفوفة قيم الـ config */
    unsigned int num_configs; /* عدد الـ configs */
};
```

#### 4. `struct pinctrl_map` — الطلب الكامل

```c
struct pinctrl_map {
    const char *dev_name;      /* اسم الجهاز اللي عايز الـ pin ده */
    const char *name;          /* اسم الـ state: "default", "sleep", ... */
    enum pinctrl_map_type type;/* نوع الطلب */
    const char *ctrl_dev_name; /* اسم الـ pin controller نفسه */
    union {
        struct pinctrl_map_mux mux;
        struct pinctrl_map_configs configs;
    } data;
};
```

---

### الـ States — "أحوال الجهاز"

الـ pinctrl subsystem بيفهم إن الجهاز بيمر بـ **states** مختلفة، وكل state محتاج الـ pins تتضبط بطريقة مختلفة:

| State | المعنى |
|-------|--------|
| `"default"` | الحالة العادية — الجهاز شغال |
| `"init"` | قبل الـ probe — لو الـ default بيعمل glitch |
| `"idle"` | الجهاز هادي — ممكن نوفر شوية power |
| `"sleep"` | الجهاز نايم — أقل استهلاك |

الـ strings دي متعرفة في `pinctrl-state.h` اللي بيتضمنه `machine.h`.

---

### الـ Macros — اختصار الكتابة

الملف بيوفر macros كتيرة عشان تبني الـ `pinctrl_map` table من غير ما تكتب كل field بإيدك:

```c
/* مثال حقيقي لـ board code */
static const struct pinctrl_map myboard_pinctrl_maps[] = {
    /* UART0: default state */
    PIN_MAP_MUX_GROUP_DEFAULT("serial0", "pinctrl@0", "uart0_grp", "uart0"),

    /* I2C1: default state مع pull-up */
    PIN_MAP_MUX_GROUP_DEFAULT("i2c1", "pinctrl@0", "i2c1_grp", "i2c1"),
    PIN_MAP_CONFIGS_GROUP_DEFAULT("i2c1", "pinctrl@0", "i2c1_grp", i2c1_configs),

    /* UART0: sleep state */
    PIN_MAP_MUX_GROUP("serial0", "sleep", "pinctrl@0", "uart0_sleep_grp", "uart0"),
};
```

#### الـ HOG macros

لما `dev_name == ctrl_dev_name` — يعني الـ pin controller نفسه بيحجز الـ pins دي لنفسه — ده اللي بيتسمى **pin hogging**. الـ macros اللي بتنتهي بـ `_HOG` بتسهل ده:

```c
PIN_MAP_MUX_GROUP_HOG_DEFAULT(dev, grp, func)
/* يعني: dev_name = ctrl_dev_name = dev */
```

---

### الـ API الرئيسي

```c
/* تسجيل الـ mapping table مع الـ core */
int pinctrl_register_mappings(const struct pinctrl_map *map,
                              unsigned int num_maps);

/* نسخة managed — بتتمسح لوحدها لما الـ device يتمسح */
int devm_pinctrl_register_mappings(struct device *dev,
                                   const struct pinctrl_map *map,
                                   unsigned int num_maps);

/* إلغاء التسجيل */
void pinctrl_unregister_mappings(const struct pinctrl_map *map);

/* لو مفيش pinctrl hardware — اعمل dummy states */
void pinctrl_provide_dummies(void);
```

لو `CONFIG_PINCTRL` مش معمل compile — كل الـ functions دي بتتحول لـ stubs فاضية، وده بيضمن إن الـ board code يشتغل حتى من غير pinctrl support.

---

### قصة عملية — SoC جديد على Board جديدة

**السيناريو:** عندنا SoC اسمه "MySoC"، وبنعمل board اسمها "MyBoard". الـ board فيها UART وI2C وSPI.

1. **مطور الـ SoC** بيكتب الـ pin controller driver في `drivers/pinctrl/pinctrl-mysoc.c` — بيعرف كل الـ pins والـ groups والـ functions اللي الـ hardware بيدعمها.

2. **مطور الـ Board** بيكتب في `arch/arm/mach-mysoc/myboard.c` (أو Device Tree):

```c
static unsigned long uart_pin_configs[] = {
    PIN_CONFIG_DRIVE_STRENGTH, 8,  /* 8mA */
};

static const struct pinctrl_map myboard_maps[] = {
    PIN_MAP_MUX_GROUP_DEFAULT("mysoc-uart.0", "mysoc-pinctrl",
                              "uart0_pins", "uart0"),
    PIN_MAP_CONFIGS_PIN_DEFAULT("mysoc-uart.0", "mysoc-pinctrl",
                                "PA1", uart_pin_configs),
};

/* في الـ board init */
pinctrl_register_mappings(myboard_maps, ARRAY_SIZE(myboard_maps));
```

3. لما الـ UART driver يعمل `probe`، الـ pinctrl core بيبحث تلقائياً عن الـ mapping اللي اسم device بتاعه "mysoc-uart.0"، وبيطبق الـ "default" state.

4. لما الـ UART يدخل sleep، بيقول للـ pinctrl core: "روح حالة sleep"، فبيطبق الـ sleep mappings.

---

### الفرق بين `machine.h` والملفات التانية

| الملف | الجمهور المستهدف | الدور |
|-------|-----------------|-------|
| `machine.h` | Board code / platform code | تعريف الـ mapping table |
| `consumer.h` | Device drivers | استخدام الـ pinctrl (get/select state) |
| `pinctrl.h` | Pin controller drivers | تسجيل الـ controller نفسه |
| `pinmux.h` | Pin controller drivers | تعريف الـ mux operations |
| `pinconf.h` | Pin controller drivers | تعريف الـ config operations |
| `pinctrl-state.h` | الكل | أسماء الـ states القياسية |

---

### الملفات المكوّنة للـ Subsystem

#### Core

| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/core.c` | قلب الـ subsystem — إدارة الـ devices والـ mappings |
| `drivers/pinctrl/pinmux.c` | تنفيذ الـ mux logic |
| `drivers/pinctrl/pinconf.c` | تنفيذ الـ config logic |
| `drivers/pinctrl/devicetree.c` | دعم الـ Device Tree |

#### Headers

| الملف | الدور |
|-------|-------|
| `include/linux/pinctrl/machine.h` | **الملف ده** — machine interface |
| `include/linux/pinctrl/consumer.h` | واجهة الـ device drivers |
| `include/linux/pinctrl/pinctrl.h` | واجهة الـ controller drivers |
| `include/linux/pinctrl/pinmux.h` | mux operations interface |
| `include/linux/pinctrl/pinconf.h` | config operations interface |
| `include/linux/pinctrl/pinconf-generic.h` | generic config parameters |
| `include/linux/pinctrl/pinctrl-state.h` | أسماء الـ states القياسية |
| `include/linux/pinctrl/devinfo.h` | معلومات الـ device في الـ core |

#### Hardware Drivers (أمثلة)

| الملف | الـ Hardware |
|-------|------------|
| `drivers/pinctrl/qcom/` | Qualcomm Snapdragon SoCs |
| `drivers/pinctrl/intel/` | Intel SoCs |
| `drivers/pinctrl/samsung/` | Samsung Exynos SoCs |
| `drivers/pinctrl/renesas/` | Renesas SoCs |
| `drivers/pinctrl/mediatek/` | MediaTek SoCs |
| `drivers/pinctrl/freescale/` | NXP/Freescale SoCs |
| `drivers/pinctrl/bcm/` | Broadcom SoCs |
| `drivers/pinctrl/pinctrl-single.c` | Generic single-register driver |

#### Documentation

| الملف | المحتوى |
|-------|---------|
| `Documentation/driver-api/pin-control.rst` | التوثيق الرسمي الكامل للـ subsystem |
## Phase 2: شرح الـ Pinctrl (Pin Control) Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

أي SoC فيه عشرات أو مئات من الـ **physical pins**. كل pin ممكن يشتغل بأكتر من طريقة:
- GPIO عادي
- UART TX/RX
- SPI CLK/MISO/MOSI
- I2C SDA/SCL
- PWM output
- وغيرها

المشكلة إن نفس الـ pin المادي لازم يتحدد قبل ما الـ driver يشتغل: هيعمل إيه بالظبط؟ وبأي إعدادات كهربية (pull-up؟ drive strength؟ slew rate؟)

قبل الـ pinctrl subsystem، كل driver كان بيكتب كود خاص بيه يكلم الـ hardware registers على طول — **سپاگيتي** محض، ومفيش تنسيق بين الـ drivers. لو driver A و driver B اتنازعوا على نفس الـ pin، الكيرنل مش هيعرف.

---

### الحل — الـ Kernel بيعمل إيه؟

الـ **pinctrl subsystem** بيقدم طبقة abstraction مركزية بين الـ hardware pins والـ drivers اللي بتستخدمها.

الفكرة الأساسية في **ثلاث محاور**:

| المحور | المعنى |
|--------|--------|
| **Pin Multiplexing (pinmux)** | تحديد وظيفة الـ pin (UART؟ SPI؟ GPIO؟) |
| **Pin Configuration (pinconf)** | الإعدادات الكهربية (pull-up/down، drive strength، إلخ) |
| **Pin Mapping (machine)** | ربط كل device باحتياجاته من الـ pins وإيمتى |

الـ `machine.h` بالذات هو **واجهة الـ board/machine** — ده اللي بيقول للـ subsystem: "الـ UART device عايز الـ pins دي، في الـ state دي".

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    Board / Machine Code                      │
  │         (arch/arm/mach-xxx  أو  DTS Device Tree)            │
  │                                                             │
  │   static const struct pinctrl_map board_pinmap[] = {        │
  │       PIN_MAP_MUX_GROUP_DEFAULT("uart0", "pinctrl0",        │
  │                                 "uart0_grp", "uart"),        │
  │       PIN_MAP_CONFIGS_PIN_DEFAULT(...),                      │
  │   };                                                         │
  └──────────────────────┬──────────────────────────────────────┘
                         │  pinctrl_register_mappings()
                         ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                  pinctrl CORE (kernel/pinctrl)               │
  │                                                             │
  │  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐  │
  │  │  Mapping DB  │   │  State FSM   │   │  Pin Ownership │  │
  │  │  (pinctrl_   │   │  (default,   │   │  (conflict     │  │
  │  │   maps list) │   │  sleep, idle)│   │   detection)   │  │
  │  └──────────────┘   └──────────────┘   └────────────────┘  │
  └───────────────┬─────────────────────────────────────────────┘
                  │  calls pinctrl_ops / pinmux_ops / pinconf_ops
                  ▼
  ┌─────────────────────────────────────────────────────────────┐
  │              Pin Controller Driver (e.g., pinctrl-bcm2835)  │
  │                                                             │
  │   struct pinctrl_desc {                                     │
  │       .pins     = bcm2835_pins[],                           │
  │       .pctlops  = &bcm2835_pctl_ops,                        │
  │       .pmxops   = &bcm2835_pmx_ops,                         │
  │       .confops  = &bcm2835_conf_ops,                        │
  │   };                                                         │
  └───────────────┬─────────────────────────────────────────────┘
                  │  direct register writes
                  ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                  Physical SoC Hardware                       │
  │            GPIO/Pinmux Registers (MMIO)                     │
  └─────────────────────────────────────────────────────────────┘

  الـ Consumers (اللي بتستخدم الـ pinctrl):
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ uart drv │  │ spi drv  │  │ i2c drv  │  │ gpio drv │
  └──────────┘  └──────────┘  └──────────┘  └──────────┘
       │              │              │              │
       └──────────────┴──────────────┴──────────────┘
                      │  pinctrl_get() + pinctrl_select_state()
                      ▼
                  pinctrl CORE
```

---

### الـ Real-World Analogy — المقسم الكهربائي في العمارة

تخيل عمارة سكنية فيها **لوحة كهربائية مركزية** (distribution board):

| العنصر في الـ Analogy | المقابل في الـ pinctrl |
|-----------------------|------------------------|
| العمارة نفسها | الـ SoC |
| اللوحة الكهربائية | الـ Pin Controller Driver |
| كل قاطع (circuit breaker) | كل physical pin |
| الشقة اللي بتستخدم الكهرباء | الـ device driver (UART, SPI...) |
| الفني اللي بيوزع الأسلاك | الـ pinctrl core |
| مخطط التوزيع (wiring diagram) | الـ `pinctrl_map` table |
| "تشغيل المكيف في الصيف فقط" | الـ pinctrl state (default/sleep/idle) |
| اللوحة مش هتوصل نفس الخط لشقتين | conflict detection في الـ pinctrl |

لما الشقة 3A (UART driver) محتاجة كهرباء (pins)، بتطلب من المقسم (pinctrl core). المقسم بيرجع للمخطط (pinctrl_map) ويشوف إيه الأسلاك المخصصة ليها، ويوجّه الفني (pin controller driver) يضبط الـ hardware بالظبط.

لو الشقة 5B طلبت نفس الخط اللي 3A شاغله، المقسم يرفض ويقول "مش متاح".

---

### الـ Core Abstraction — فكرة الـ State Machine

الـ abstraction الجوهرية في الـ pinctrl هي إن كل device عنده **states** مش مجرد configuration واحدة ثابتة.

الـ `pinctrl-state.h` بيعرّف الـ standard states:

```c
#define PINCTRL_STATE_DEFAULT "default"   // شغّال عادي
#define PINCTRL_STATE_INIT    "init"      // قبل probe (لتجنب glitches)
#define PINCTRL_STATE_IDLE    "idle"      // runtime suspend
#define PINCTRL_STATE_SLEEP   "sleep"     // system suspend
```

الـ device لما بيصحى من الـ sleep بيطلب `default`، ولما بيدخل الـ PM idle بيطلب `idle`. الـ core بيتعامل مع الانتقالات دي.

---

### شرح الـ Structs في `machine.h`

#### 1. `enum pinctrl_map_type` — نوع الـ mapping

```c
enum pinctrl_map_type {
    PIN_MAP_TYPE_INVALID,        // خطأ / uninitialized
    PIN_MAP_TYPE_DUMMY_STATE,    // state وهمي (للـ drivers اللي مش عايزة pinctrl)
    PIN_MAP_TYPE_MUX_GROUP,      // تحديد وظيفة (UART/SPI/I2C)
    PIN_MAP_TYPE_CONFIGS_PIN,    // إعدادات كهربية على pin واحد
    PIN_MAP_TYPE_CONFIGS_GROUP,  // إعدادات كهربية على group من الـ pins
};
```

كل entry في الـ mapping table بتاعة الـ board هي واحدة من الأنواع دي.

---

#### 2. `struct pinctrl_map_mux` — بيانات الـ multiplexing

```c
struct pinctrl_map_mux {
    const char *group;      // اسم الـ pin group (e.g., "uart0_grp")
    const char *function;   // الوظيفة المطلوبة (e.g., "uart")
};
```

الـ **group** هو مجموعة الـ pins اللي بتشتغل مع بعض لخدمة function واحدة — مثلاً UART0 محتاج TX + RX + RTS + CTS مع بعض، دول "group".

الـ **function** هي الـ hardware function اللي هتتحط على الـ group دي (الـ pin controller driver بيعرّفها في `struct pinfunction`).

---

#### 3. `struct pinctrl_map_configs` — بيانات الإعدادات الكهربية

```c
struct pinctrl_map_configs {
    const char *group_or_pin;   // اسم الـ pin أو الـ group
    unsigned long *configs;     // array من الإعدادات
    unsigned int num_configs;   // عدد الإعدادات في الـ array
};
```

الـ `configs` هي array من الـ `unsigned long` — كل value فيها بتحمل الإعداد encoded. الـ encoding ده hardware-specific: الـ pin controller driver هو اللي بيفهمه. مثلاً:

```c
/* مثال: pull-up + drive strength 8mA */
static unsigned long uart_pin_configs[] = {
    PIN_CONFIG_BIAS_PULL_UP,
    PIN_CONFIG_DRIVE_STRENGTH | (8 << 8),
};
```

---

#### 4. `struct pinctrl_map` — الـ mapping entry الكاملة

```c
struct pinctrl_map {
    const char *dev_name;        // اسم الـ device اللي عايز الـ pins دي
    const char *name;            // اسم الـ state ("default", "sleep", ...)
    enum pinctrl_map_type type;  // نوع الـ entry
    const char *ctrl_dev_name;   // اسم الـ pin controller اللي بيتحكم فيها
    union {
        struct pinctrl_map_mux mux;
        struct pinctrl_map_configs configs;
    } data;                      // البيانات حسب الـ type
};
```

ده الـ **central struct** في الـ machine interface. كل entry بتقول:
- `dev_name`: مين اللي محتاج؟ (مثلاً `"soc:uart@12000000"`)
- `name`: في أنهي state؟ (`"default"`)
- `type`: إيه نوع الطلب؟ (mux أو config؟)
- `ctrl_dev_name`: مين الـ pin controller المسؤول؟
- `data`: التفاصيل

---

#### الـ HOG Mechanism — الـ Pin Controller بيحجز الـ Pins بتاعته

الـ **hog** معناها إن الـ pin controller نفسه بيحجز بعض الـ pins لحسابه عند الـ registration مباشرة. يحصل ده لما `dev_name == ctrl_dev_name`.

```c
/* المثال: الـ pin controller بيحجز LED pin لنفسه */
PIN_MAP_MUX_GROUP_HOG_DEFAULT("pinctrl0", "led_grp", "gpio")
// يتوسع إلى:
PIN_MAP_MUX_GROUP("pinctrl0", PINCTRL_STATE_DEFAULT, "pinctrl0", "led_grp", "gpio")
//                 ^^^dev^^^                           ^^^ctrl^^^ — نفسه!
```

ده بيحصل automatically في `pinctrl_register_and_init()`.

---

### علاقة الـ Structs ببعض

```
struct pinctrl_map[]              (الـ board mapping table)
    │
    ├── dev_name ──────────────────► struct device (consumer driver)
    │
    ├── ctrl_dev_name ─────────────► struct pinctrl_dev
    │                                    │
    │                                    └── struct pinctrl_desc
    │                                            │
    │                                            ├── pins[]  ──► struct pinctrl_pin_desc[]
    │                                            ├── pctlops ──► struct pinctrl_ops
    │                                            ├── pmxops  ──► struct pinmux_ops
    │                                            └── confops ──► struct pinconf_ops
    │
    ├── type == MUX_GROUP
    │       └── data.mux ──────────► struct pinctrl_map_mux
    │                                    ├── group    ──► struct pingroup (في الـ driver)
    │                                    └── function ──► struct pinfunction (في الـ driver)
    │
    └── type == CONFIGS_*
            └── data.configs ──────► struct pinctrl_map_configs
                                         ├── group_or_pin
                                         └── configs[] (hardware-specific encoding)
```

---

### الـ Convenience Macros — الـ Board Code بيكتبه إزاي؟

الـ macros بتخلي الـ board file واضح وقصير. مثال حقيقي:

```c
/* board file: arch/arm/mach-omap2/board-xxx.c */
static const struct pinctrl_map board_maps[] = {
    /* UART0: default state — MUX pins لـ UART */
    PIN_MAP_MUX_GROUP_DEFAULT(
        "omap-serial.0",      // dev_name
        "pinctrl",            // ctrl_dev_name
        "uart0_pins",         // group
        "uart0"               // function
    ),

    /* UART0: default state — config (pull-up على RX) */
    PIN_MAP_CONFIGS_PIN_DEFAULT(
        "omap-serial.0",
        "pinctrl",
        "uart0_rxd",          // pin واحد
        uart0_rxd_configs     // unsigned long[]
    ),

    /* UART0: sleep state — غيّر الـ function لـ GPIO لتقليل الـ leakage */
    PIN_MAP_MUX_GROUP(
        "omap-serial.0",
        PINCTRL_STATE_SLEEP,
        "pinctrl",
        "uart0_pins",
        "gpio"
    ),
};

/* التسجيل في الـ board init */
pinctrl_register_mappings(board_maps, ARRAY_SIZE(board_maps));
```

---

### `devm_pinctrl_register_mappings` — الـ Resource Management

الـ `devm_` variant بتربط الـ unregistration بـ device lifetime. لما الـ device اتشال (أو فشلت الـ probe)، الـ kernel بيعمل `pinctrl_unregister_mappings()` تلقائياً. ده جزء من نظام الـ **devres** — اللي هو subsystem مستقل بيدير lifecycle الـ resources المربوطة بـ device.

---

### `pinctrl_provide_dummies()` — التعامل مع Board بدون Pinctrl

لو الـ board مش محتاجة pinctrl (أو الـ hardware مش بيدعمها)، الـ drivers اللي بتطلب pinctrl states هتفشل. الـ `pinctrl_provide_dummies()` بتقول للـ core: "أي state مش موجودة، تعاملها كـ DUMMY_STATE ومتفشلش". ده بيخلي الـ driver code uniform من غير `#ifdef CONFIG_PINCTRL` في كل حتة.

---

### الـ Pinctrl Subsystem — بيمتلك إيه؟ وبيفوّض إيه؟

| الـ Core (بيمتلكه) | الـ Driver (بيفوّض إليه) |
|--------------------|--------------------------|
| قاعدة بيانات الـ mappings | تعريف الـ pins والـ groups |
| conflict detection (مين شاغل الـ pin) | تنفيذ الـ MUX على الـ registers |
| state machine (default/sleep/idle) | تنفيذ الـ config على الـ registers |
| واجهة الـ consumers (`pinctrl_get`, `pinctrl_select_state`) | DT parsing (`dt_node_to_map`) |
| الـ debugfs interface | driver-specific config encoding |
| الـ HOG processing | GPIO range management |

---

### مفاهيم تانية لازم تعرفها قبل تتعمق أكتر

- **GPIO subsystem**: بيتكامل مع الـ pinctrl عبر `pinctrl_gpio_range`. لما الـ GPIO driver بيطلب pin، الـ pinctrl core بيتحقق من الـ ownership.
- **Device Tree (DT)**: في الأنظمة الحديثة، الـ `pinctrl_map` مش بتتكتب يدوياً في الـ board file — بتتولد من الـ DT عبر `dt_node_to_map()` في الـ driver. الـ machine.h API هو الـ C-based legacy interface.
- **devres (devm_*)**: نظام الـ resource management الخاص بالـ kernel، بيضمن إن الـ resources تتحرر لما الـ device scope ينتهي.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### الـ Context العام

الـ file ده (`machine.h`) هو الـ **machine-level interface** للـ pinctrl subsystem في الـ Linux kernel. بيوفر الـ types والـ macros اللي بتسمح للـ board/platform code إنها تعرّف جدول mapping بين الـ devices والـ pin configurations. الـ "machine" هنا يعني الـ SoC board أو الـ platform، مش الـ driver نفسه.

---

### Step 0: الـ Enums، الـ Config Options، والـ Standard States

#### الـ `pinctrl_map_type` Enum

| القيمة | المعنى |
|---|---|
| `PIN_MAP_TYPE_INVALID` | entry غير صالح، للـ error detection |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمي بدون أي تغيير حقيقي في الـ hardware |
| `PIN_MAP_TYPE_MUX_GROUP` | تحديد الـ mux function لمجموعة pins |
| `PIN_MAP_TYPE_CONFIGS_PIN` | تطبيق config parameters على pin واحد بالاسم |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | تطبيق config parameters على group كاملة |

#### الـ Standard State Strings (من `pinctrl-state.h`)

| الـ Macro | القيمة النصية | الاستخدام |
|---|---|---|
| `PINCTRL_STATE_DEFAULT` | `"default"` | الحالة الطبيعية عند تشغيل الـ device أو الرجوع من sleep |
| `PINCTRL_STATE_INIT` | `"init"` | قبل استدعاء `probe()` للـ driver لتجنب glitch على الـ pins |
| `PINCTRL_STATE_IDLE` | `"idle"` | عند الـ idle (pm_runtime_suspend/idle) |
| `PINCTRL_STATE_SLEEP` | `"sleep"` | أعمق حالة نوم عند الـ suspend |

#### الـ `CONFIG_PINCTRL` Kconfig Option

| الحالة | السلوك |
|---|---|
| مفعّل (`CONFIG_PINCTRL=y`) | الدوال الحقيقية متاحة: `pinctrl_register_mappings`، إلخ |
| مش مفعّل | كل الدوال بتبقى `static inline` ترجع `0` أو فاضية — بدون overhead |

---

### Step 1: الـ Structs المهمة

#### 1. `struct pinctrl_map_mux`

**الغرض:** بيحدد الـ mux function المطلوب تفعيله لـ group معينة من الـ pins.

```c
struct pinctrl_map_mux {
    const char *group;    /* اسم الـ pin group — لو NULL بياخد أول group مناسبة */
    const char *function; /* اسم الـ mux function المطلوب */
};
```

| الحقل | النوع | الشرح |
|---|---|---|
| `group` | `const char *` | اسم الـ group المُعرَّفة في الـ pin controller driver. ممكن تبقى `NULL` |
| `function` | `const char *` | اسم الـ function زي `"uart0"`، `"spi1"`، إلخ |

**الاتصال بـ structs تانية:** بتتضمن جوا الـ `union data` في `pinctrl_map`.

---

#### 2. `struct pinctrl_map_configs`

**الغرض:** بيحمل مصفوفة من الـ configuration parameters اللي هتتكتب على pin أو group (pull-up، drive strength، إلخ).

```c
struct pinctrl_map_configs {
    const char *group_or_pin; /* اسم الـ pin أو الـ group */
    unsigned long *configs;   /* pointer لمصفوفة الـ config values */
    unsigned int num_configs; /* عدد العناصر في المصفوفة */
};
```

| الحقل | النوع | الشرح |
|---|---|---|
| `group_or_pin` | `const char *` | اسم الـ pin (في حالة `CONFIGS_PIN`) أو الـ group (في حالة `CONFIGS_GROUP`) |
| `configs` | `unsigned long *` | مصفوفة من الـ packed config values — كل value بتحمل parameter ID وقيمته |
| `num_configs` | `unsigned int` | حجم المصفوفة، عادةً بيتحسب بـ `ARRAY_SIZE()` |

**ملاحظة:** صيغة الـ `configs` array بتعتمد على الـ pin controller نفسه أو الـ generic pinconf encoding.

---

#### 3. `struct pinctrl_map` — الـ struct الرئيسي

**الغرض:** الـ entry الواحدة في جدول الـ mapping اللي الـ board/machine code بيعرّفه. بيربط device معين بـ pin controller معين في state معين.

```c
struct pinctrl_map {
    const char *dev_name;      /* اسم الـ consumer device */
    const char *name;          /* اسم الـ state: "default"، "sleep"، إلخ */
    enum pinctrl_map_type type;/* نوع الـ entry */
    const char *ctrl_dev_name; /* اسم الـ pin controller device */
    union {
        struct pinctrl_map_mux     mux;
        struct pinctrl_map_configs configs;
    } data;                    /* البيانات الخاصة بالنوع */
};
```

| الحقل | النوع | الشرح |
|---|---|---|
| `dev_name` | `const char *` | اسم الـ consumer device زي `"soc:uart0"` — لازم يطابق `dev_name(device)` |
| `name` | `const char *` | اسم الـ state زي `"default"` أو `"sleep"` |
| `type` | `enum pinctrl_map_type` | نوع الـ entry بيحدد أنهي حقل في الـ `union` يُستخدم |
| `ctrl_dev_name` | `const char *` | اسم الـ pin controller نفسه. مش مستخدم مع `DUMMY_STATE` |
| `data.mux` | `struct pinctrl_map_mux` | بيتستخدم لما النوع `MUX_GROUP` |
| `data.configs` | `struct pinctrl_map_configs` | بيتستخدم لما النوع `CONFIGS_PIN` أو `CONFIGS_GROUP` |

**الـ Hog مفهوم:** لو `dev_name == ctrl_dev_name`، الـ pin controller driver بيـ"hog" الـ mapping ده لنفسه عند التسجيل.

---

### Step 2: الـ Macros — Cheatsheet

#### Convenience Macros للـ MUX

| الـ Macro | الـ State | الـ ctrl_dev_name |
|---|---|---|
| `PIN_MAP_MUX_GROUP(dev, state, pinctrl, grp, func)` | مخصص | مخصص |
| `PIN_MAP_MUX_GROUP_DEFAULT(dev, pinctrl, grp, func)` | `"default"` | مخصص |
| `PIN_MAP_MUX_GROUP_HOG(dev, state, grp, func)` | مخصص | = `dev` (hog) |
| `PIN_MAP_MUX_GROUP_HOG_DEFAULT(dev, grp, func)` | `"default"` | = `dev` (hog) |

#### Convenience Macros للـ PIN Configs

| الـ Macro | الـ State | الـ ctrl_dev_name |
|---|---|---|
| `PIN_MAP_CONFIGS_PIN(dev, state, pinctrl, pin, cfgs)` | مخصص | مخصص |
| `PIN_MAP_CONFIGS_PIN_DEFAULT(dev, pinctrl, pin, cfgs)` | `"default"` | مخصص |
| `PIN_MAP_CONFIGS_PIN_HOG(dev, state, pin, cfgs)` | مخصص | = `dev` |
| `PIN_MAP_CONFIGS_PIN_HOG_DEFAULT(dev, pin, cfgs)` | `"default"` | = `dev` |

#### Convenience Macros للـ GROUP Configs

| الـ Macro | الـ State | الـ ctrl_dev_name |
|---|---|---|
| `PIN_MAP_CONFIGS_GROUP(dev, state, pinctrl, grp, cfgs)` | مخصص | مخصص |
| `PIN_MAP_CONFIGS_GROUP_DEFAULT(dev, pinctrl, grp, cfgs)` | `"default"` | مخصص |
| `PIN_MAP_CONFIGS_GROUP_HOG(dev, state, grp, cfgs)` | مخصص | = `dev` |
| `PIN_MAP_CONFIGS_GROUP_HOG_DEFAULT(dev, grp, cfgs)` | `"default"` | = `dev` |

#### الـ Dummy State Macro

```c
PIN_MAP_DUMMY_STATE(dev, state)
// يعمل entry من نوع DUMMY_STATE بدون ctrl_dev_name أو data
```

---

### Step 3: رسم العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────────┐
│                     struct pinctrl_map                          │
│                                                                 │
│  dev_name ──────────────────────────────► "soc:uart0"          │
│  name ──────────────────────────────────► "default"            │
│  type ──────────────────────────────────► PIN_MAP_TYPE_MUX_GROUP│
│  ctrl_dev_name ─────────────────────────► "pinctrl-bcm2835"    │
│                                                                 │
│  data (union)                                                   │
│  ├─── [if MUX_GROUP] ─────────────────────────────────────────►│
│  │         struct pinctrl_map_mux                               │
│  │         ├── group    → "uart0_grp"                          │
│  │         └── function → "uart0"                              │
│  │                                                              │
│  └─── [if CONFIGS_PIN / CONFIGS_GROUP] ──────────────────────►│
│            struct pinctrl_map_configs                           │
│            ├── group_or_pin → "GPIO14"                         │
│            ├── configs      → [0x00050001, 0x00020003, ...]    │
│            └── num_configs  → 2                                 │
└─────────────────────────────────────────────────────────────────┘

                     ▼  تُسجَّل في الـ core عبر
         pinctrl_register_mappings(map_table, num_maps)

┌─────────────────────────────────────────────────────────────────┐
│               pinctrl core (internal list)                      │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│   │ pinctrl_map  │───►│ pinctrl_map  │───►│ pinctrl_map  │···  │
│   │ [entry 0]    │    │ [entry 1]    │    │ [entry 2]    │     │
│   └──────────────┘    └──────────────┘    └──────────────┘     │
└─────────────────────────────────────────────────────────────────┘
          ▼
   لما device يطلب state معينة:
   pinctrl_get() + pinctrl_lookup_state() + pinctrl_select_state()
          ▼
   الـ core بيدور على الـ entries المناسبة ويطبّقها
```

#### العلاقة مع `pinctrl_desc` (من `pinctrl.h`)

```
┌────────────────────┐         ┌─────────────────────────────────┐
│  struct pinctrl_   │         │  struct pinctrl_desc            │
│  map (machine.h)  │         │  (pinctrl.h)                    │
│                    │         │                                 │
│  ctrl_dev_name ───┼────────►│  name (must match)              │
│  data.mux.function┼────────►│  → pmxops->get_function_name()  │
│  data.mux.group   ┼────────►│  → pctlops->get_group_name()    │
└────────────────────┘         └─────────────────────────────────┘
```

---

### Step 4: الـ Lifecycle Diagram

#### lifecycle الـ mapping table من تعريفها للتطبيق

```
BOARD/MACHINE CODE (arch/ or drivers/platform/)
─────────────────────────────────────────────────
[1] DEFINE (static const)
    static const struct pinctrl_map board_maps[] = {
        PIN_MAP_MUX_GROUP_DEFAULT("soc:uart0", "pinctrl0", "uart0_grp", "uart0"),
        PIN_MAP_CONFIGS_PIN_DEFAULT("soc:uart0", "pinctrl0", "GPIO14", uart_cfgs),
    };

[2] REGISTER (early boot / board init)
    pinctrl_register_mappings(board_maps, ARRAY_SIZE(board_maps));
    ─── أو ───
    devm_pinctrl_register_mappings(dev, board_maps, ARRAY_SIZE(board_maps));

         ▼
[3] CORE STORES (في قائمة داخلية)
    الـ core بيعمل copy من الـ map entries ويحطها في linked list داخلي

         ▼
[4] CONSUMER DEVICE PROBES
    struct pinctrl *p = pinctrl_get(dev);
    struct pinctrl_state *s = pinctrl_lookup_state(p, PINCTRL_STATE_DEFAULT);
    pinctrl_select_state(p, s);
         │
         ├── الـ core بيلاقي الـ entries الخاصة بـ dev
         ├── لكل MUX_GROUP entry: بيستدعي ops->set_mux()
         └── لكل CONFIGS_* entry: بيستدعي ops->pin_config_set()

         ▼
[5] DEVICE RUNTIME (state switching)
    pm_runtime_suspend → pinctrl_select_state(p, sleep_state)
    pm_runtime_resume  → pinctrl_select_state(p, default_state)

         ▼
[6] DEVICE REMOVED / DRIVER UNLOADED
    pinctrl_put(p)         ← يحرر الـ pinctrl handle
    pinctrl_unregister_mappings(board_maps)  ← يشيل الـ entries
```

#### lifecycle الـ HOG entries (خاصة بالـ pin controller نفسه)

```
[1] Pin controller driver يسجّل نفسه:
    pinctrl_register_and_init(&pctldesc, dev, drv_data, &pctldev);
         ▼
[2] الـ core بيدور على الـ registered maps
         ▼
[3] لو وجد entry بـ dev_name == ctrl_dev_name
    الـ core بـ"يحجز" (hog) الـ entry ويطبّقها فوراً
    بدون انتظار consumer device
         ▼
[4] الـ pins تبقى في الـ state المطلوب طول فترة تشغيل الـ controller
```

---

### Step 5: الـ Call Flow Diagrams

#### تسجيل الـ mappings

```
board_init() / platform_driver probe()
  │
  └─► pinctrl_register_mappings(map, num_maps)
        │
        ├── validates map != NULL, num_maps > 0
        ├── allocates pinctrl_maps node
        ├── copies pointer to map array (لا بيعمل deep copy)
        └── adds node to global maps_node list
              (mutex: pinctrl_maps_mutex protects list)
```

#### اختيار state (pinctrl_select_state)

```
device driver
  │
  └─► pinctrl_select_state(pinctrl_handle, state)
        │
        ├─► for each map entry matching (dev_name, state->name):
        │     │
        │     ├─[MUX_GROUP]──► pinmux_enable_setting()
        │     │                   └─► ops->set_mux(pctldev, func_sel, grp_sel)
        │     │                         └─► writes hardware mux register
        │     │
        │     └─[CONFIGS_*]──► pinconf_apply_setting()
        │                         └─► ops->pin_config_set(pctldev, pin, cfgs, n)
        │                               └─► writes hardware config register
        │
        └─► returns 0 on success, negative errno on failure
```

#### الـ Device Tree path (بديل عن الـ machine.h)

```
Device Tree .dts
  │
  └─► pinctrl node في DT
        │
        └─► pin controller driver's dt_node_to_map()  [from pinctrl_ops]
              │
              ├── allocates struct pinctrl_map[] dynamically
              └── fills them exactly like board_maps[]
                  ── same pinctrl_map structure ──
```

---

### Step 6: الـ Locking Strategy

الـ `machine.h` بنفسه ما بيعرّفش locks، لكن الـ locking بيحصل في الـ pinctrl core اللي بيستخدم الـ structs دي:

| الـ Lock | المكان (في الـ core) | اللي بيحميه |
|---|---|---|
| `pinctrl_maps_mutex` (mutex) | `pinctrl/core.c` | الـ global list من الـ `pinctrl_map` entries المسجّلة |
| `pctldev->mutex` (mutex) | `pinctrl/core.c` | الـ state الداخلي للـ pin controller device نفسه |
| `pinctrl_list_mutex` (mutex) | `pinctrl/core.c` | قائمة الـ consumer handles |

#### ترتيب الـ Locking (Lock Ordering)

```
pinctrl_maps_mutex
    └── pctldev->mutex
            └── (hardware register access — no further locks needed)
```

**قاعدة:** `pinctrl_maps_mutex` دايماً يُحجز الأول. ممنوع تعكس الترتيب تجنباً لـ deadlock.

#### الـ `devm_` variant والـ Resource Management

```
devm_pinctrl_register_mappings(dev, map, num_maps)
  │
  ├── internally calls pinctrl_register_mappings()
  └── registers devres cleanup:
        on device removal → pinctrl_unregister_mappings(map)
            └── takes pinctrl_maps_mutex
            └── removes node from global list
            └── frees node (NOT the map array — it's static)
```

**ملاحظة مهمة:** الـ `pinctrl_map` array نفسها بتكون `static const` في الـ board code، فالـ core مش بيـfree الـ array — بس بيشيل الـ reference منها. الـ `devm_` version بتتأكد إن الـ reference اتشالت لو الـ device اتحذف.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ API Functions

| Function | الغرض | Context |
|---|---|---|
| `pinctrl_register_mappings()` | تسجيل جدول الـ pin mappings للـ machine | init / board code |
| `devm_pinctrl_register_mappings()` | نفس الأعلى بس مع devres auto-cleanup | driver probe |
| `pinctrl_unregister_mappings()` | إلغاء تسجيل الـ mappings | cleanup / exit |
| `pinctrl_provide_dummies()` | تفعيل dummy states للـ platforms اللي مش محتاجة pinctrl | early boot |

#### الـ Macros (Mapping Table Helpers)

| Macro | النوع | الاستخدام |
|---|---|---|
| `PIN_MAP_DUMMY_STATE` | `DUMMY_STATE` | state وهمية للـ devices اللي مش محتاجة pinctrl فعلي |
| `PIN_MAP_MUX_GROUP` | `MUX_GROUP` | ربط group بـ mux function |
| `PIN_MAP_MUX_GROUP_DEFAULT` | `MUX_GROUP` | زي الأعلى، state = "default" |
| `PIN_MAP_MUX_GROUP_HOG` | `MUX_GROUP` | الـ pinctrl driver يحجز الـ group لنفسه |
| `PIN_MAP_MUX_GROUP_HOG_DEFAULT` | `MUX_GROUP` | hog + default state |
| `PIN_MAP_CONFIGS_PIN` | `CONFIGS_PIN` | ضبط configs على pin واحدة |
| `PIN_MAP_CONFIGS_PIN_DEFAULT` | `CONFIGS_PIN` | configs على pin، state = "default" |
| `PIN_MAP_CONFIGS_PIN_HOG` | `CONFIGS_PIN` | hog config على pin |
| `PIN_MAP_CONFIGS_PIN_HOG_DEFAULT` | `CONFIGS_PIN` | hog + default state على pin |
| `PIN_MAP_CONFIGS_GROUP` | `CONFIGS_GROUP` | configs على group كاملة |
| `PIN_MAP_CONFIGS_GROUP_DEFAULT` | `CONFIGS_GROUP` | configs على group، state = "default" |
| `PIN_MAP_CONFIGS_GROUP_HOG` | `CONFIGS_GROUP` | hog config على group |
| `PIN_MAP_CONFIGS_GROUP_HOG_DEFAULT` | `CONFIGS_GROUP` | hog + default state على group |

---

### تصنيف الـ Functions

```
┌─────────────────────────────────────────────────────────┐
│              pinctrl machine.h API Groups                │
├─────────────────────┬───────────────────────────────────┤
│  Registration       │  pinctrl_register_mappings()       │
│                     │  devm_pinctrl_register_mappings()  │
├─────────────────────┼───────────────────────────────────┤
│  Cleanup            │  pinctrl_unregister_mappings()     │
├─────────────────────┼───────────────────────────────────┤
│  Platform Helpers   │  pinctrl_provide_dummies()         │
├─────────────────────┼───────────────────────────────────┤
│  Macro Helpers      │  PIN_MAP_* family                  │
└─────────────────────┴───────────────────────────────────┘
```

---

### Group 1: Registration

الهدف من الـ group ده إن الـ board code أو الـ platform driver يسجّل جدول الـ `pinctrl_map` مع الـ pinctrl core. الـ core بيحتفظ بكل الـ mappings في قائمة داخلية، وأي device بيطلب pinctrl state بيلاقي mapping الخاص بيه من خلال الـ `dev_name` و `name` اللي اتسجلوا.

---

#### `pinctrl_register_mappings()`

```c
int pinctrl_register_mappings(const struct pinctrl_map *map,
                               unsigned int num_maps);
```

**الـ function دي بتسجّل مصفوفة من الـ `pinctrl_map` entries مع الـ pinctrl core.** بتعمل copy للـ array كلها وتضيفها للـ global mappings list جوه الـ core. أي device بعد كده لما يطلب `pinctrl_get()` هيلاقي الـ mapping الخاص بيه من خلال الـ `dev_name`.

**Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `map` | `const struct pinctrl_map *` | الـ array من الـ mapping entries — لازم تبقى static أو valid طول عمر الـ system |
| `num_maps` | `unsigned int` | عدد الـ entries في الـ array |

**Return:**
- **`0`** عند النجاح
- **`-EINVAL`** لو الـ map أو `num_maps` غلط
- **`-ENOMEM`** لو في مشكلة في الـ memory allocation

**Key Details:**
- بتعمل `kmemdup()` للـ array كلها، فالـ caller مش محتاج يخلي الـ array valid بعد الـ call — بس في الـ static board code هي عادةً static anyway.
- بتشيل الـ `pinctrl_maps_mutex` أثناء إضافة الـ entries للـ list.
- كل entry بتتضاف كـ `pinctrl_maps` object يتعلق بيها sub-list من الـ map entries.
- لازم تتاخد الـ data structures اللي جوا الـ map (زي `configs` arrays) صح — الـ function مش بتعمل deep copy.

**Caller Context:**
- بتتاخد من `__init` board code أو من `platform_driver.probe()` قبل أي device يعمل `pinctrl_get()`.
- مثال واقعي: `arch/arm/mach-*/` board files.

**Pseudocode:**

```c
int pinctrl_register_mappings(map, num_maps):
    if (!map || !num_maps)
        return -EINVAL

    // validate each entry: dev_name, name, type must be set
    for each entry in map[0..num_maps-1]:
        validate_entry(entry)  // -EINVAL on bad entry

    // allocate wrapper struct + kmemdup of the map array
    maps_node = kzalloc(sizeof(*maps_node))
    maps_node->maps = kmemdup(map, num_maps * sizeof(*map))
    maps_node->num_maps = num_maps

    mutex_lock(&pinctrl_maps_mutex)
    list_add_tail(&maps_node->node, &pinctrl_maps)
    mutex_unlock(&pinctrl_maps_mutex)

    return 0
```

---

#### `devm_pinctrl_register_mappings()`

```c
int devm_pinctrl_register_mappings(struct device *dev,
                                    const struct pinctrl_map *map,
                                    unsigned int num_maps);
```

**الـ devres version من `pinctrl_register_mappings()`.** بتسجّل الـ mappings وبتربطها بعمر الـ `device`، فلما الـ device يتـ unbind أو يتـ remove، الـ mappings بتتـ unregister تلقائياً من غير ما الـ driver يحتاج يعمل cleanup يدوي.

**Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ device اللي هيتـ own الـ mappings — لما يتـ destroy، الـ mappings بتتمسح |
| `map` | `const struct pinctrl_map *` | الـ mapping array |
| `num_maps` | `unsigned int` | عدد الـ entries |

**Return:**
- **`0`** عند النجاح
- **`-ENOMEM`** لو فشل الـ devres allocation
- نفس errors الـ `pinctrl_register_mappings()`

**Key Details:**
- بتستخدم `devm_add_action_or_reset()` أو `devres` resource لتسجيل `pinctrl_unregister_mappings()` كـ cleanup action.
- الـ `dev` مش لازم يكون هو الـ pinctrl controller — ممكن يكون أي device عنده الـ mappings.
- مناسبة جداً للـ drivers اللي بيعملوا dynamic map registration في `probe()`.

**Caller Context:**
- `driver.probe()` مع توقع auto-cleanup على `driver.remove()`.

---

### Group 2: Cleanup

---

#### `pinctrl_unregister_mappings()`

```c
void pinctrl_unregister_mappings(const struct pinctrl_map *map);
```

**بتشيل الـ mappings اللي اتسجلت بـ `pinctrl_register_mappings()` من الـ global list.** بتدور على الـ `pinctrl_maps` node اللي بيتطابق مع الـ `map` pointer وبتشيله من الـ list وبتحرر الـ memory.

**Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `map` | `const struct pinctrl_map *` | نفس الـ pointer اللي اتبعت لـ `pinctrl_register_mappings()` — بيتستخدم كـ key للبحث |

**Return:** `void`

**Key Details:**
- بتشيل `pinctrl_maps_mutex` أثناء الـ list traversal والـ removal.
- لو أي device لسه شايل reference على هذه الـ mappings (يعني عمل `pinctrl_get()` ولسه معملش `pinctrl_put()`)، الـ behavior ممكن يبقى undefined — الـ core مش بيعمل reference counting على مستوى الـ map.
- الـ `devm_pinctrl_register_mappings()` بتستدعيها تلقائياً على الـ device destroy.

**Caller Context:**
- `__exit` functions أو `driver.remove()` للـ board code اليدوي.
- تلقائياً من الـ devres infrastructure.

---

### Group 3: Platform Helpers

---

#### `pinctrl_provide_dummies()`

```c
void pinctrl_provide_dummies(void);
```

**بتفعّل الـ "dummy state" support على مستوى الـ pinctrl core.** لما تتاخد، الـ core بيقبل الـ `PIN_MAP_TYPE_DUMMY_STATE` entries ومش بيرجع error لو device طلب state وملقاش mapping حقيقي ليها — بيعمل no-op بدل ما يفشل.

**Parameters:** لا يوجد

**Return:** `void`

**Key Details:**
- الـ flag اللي بتعمله set اسمه `pinctrl_dummy_state` (global bool في الـ core).
- الغرض منها: platforms زي ARM64 SoCs الجديدة ممكن يكون بعض devices معندهاش pinctrl requirements حقيقية بس الـ framework بيتوقع mapping. بدل ما تعمل لكل device mapping فاضية، بتاخد `pinctrl_provide_dummies()` مرة واحدة في الـ board/platform init.
- لو مش متفعلة: طلب state مش موجودة في الـ map بيرجع error للـ device.
- لو متفعلة: طلب state مش موجودة بيتعامل معاه كـ dummy (no-op) بنجاح.

**Caller Context:**
- `arch/*/mach-*/board*.c` أو `arch/*/kernel/setup.c` قبل أي device probe.

---

### Group 4: Macro Helpers — Mapping Table Builders

الـ macros دي بتسهّل بناء الـ `struct pinctrl_map` arrays في الـ board/platform code. كلها C compound literal initializers. تنقسم لـ 3 أنواع رئيسية:

---

#### النوع الأول: Dummy State Macros

##### `PIN_MAP_DUMMY_STATE(dev, state)`

```c
#define PIN_MAP_DUMMY_STATE(dev, state) \
    {                                    \
        .dev_name = dev,                 \
        .name = state,                   \
        .type = PIN_MAP_TYPE_DUMMY_STATE,\
    }
```

**بينشئ entry من نوع `DUMMY_STATE` في جدول الـ mappings.** الـ entry دي مش بتعمل أي mux أو config حقيقي — بس بتقول للـ core "الـ device ده عارف بالـ state دي، مش محتاج action". مناسبة للـ devices اللي موجودة على بعض boards بس مش محتاجة pinctrl فعلي.

| Parameter | الوصف |
|---|---|
| `dev` | اسم الـ device (string literal) — لازم يتطابق مع `dev_name(device)` |
| `state` | اسم الـ state (e.g., `"default"`, `"sleep"`) |

---

#### النوع الثاني: Mux Group Macros

##### `PIN_MAP_MUX_GROUP(dev, state, pinctrl, grp, func)`

```c
#define PIN_MAP_MUX_GROUP(dev, state, pinctrl, grp, func) \
    {                                                       \
        .dev_name = dev,                                    \
        .name = state,                                      \
        .type = PIN_MAP_TYPE_MUX_GROUP,                     \
        .ctrl_dev_name = pinctrl,                           \
        .data.mux = {                                       \
            .group = grp,                                   \
            .function = func,                               \
        },                                                  \
    }
```

**بينشئ entry بتعمل mux لـ pin group على function معينة.** الـ core لما يطبّق الـ state ده بيروح للـ pinctrl driver المسؤول ويطلب منه يعمل `set_mux()` على الـ group بالـ function المحددة.

| Parameter | الوصف |
|---|---|
| `dev` | اسم الـ device اللي محتاج الـ mux ده |
| `state` | اسم الـ pinctrl state |
| `pinctrl` | اسم الـ pinctrl controller device |
| `grp` | اسم الـ pin group (أو `NULL` لاختيار أول group مناسبة) |
| `func` | اسم الـ mux function |

**مثال واقعي:**
```c
// UART2 على i.MX6 يحتاج pins تتعمل mux على UART function
PIN_MAP_MUX_GROUP("2020000.uart", "default",
                  "20e0000.iomuxc",
                  "uart2grp", "uart2")
```

---

##### `PIN_MAP_MUX_GROUP_DEFAULT(dev, pinctrl, grp, func)`

```c
#define PIN_MAP_MUX_GROUP_DEFAULT(dev, pinctrl, grp, func) \
    PIN_MAP_MUX_GROUP(dev, PINCTRL_STATE_DEFAULT, pinctrl, grp, func)
```

**اختصار لـ `PIN_MAP_MUX_GROUP` مع `state = "default"`.** الـ `PINCTRL_STATE_DEFAULT` هو `"default"` — الـ state اللي بيتطلبها الـ framework تلقائياً قبل `driver.probe()`.

---

##### `PIN_MAP_MUX_GROUP_HOG(dev, state, grp, func)`

```c
#define PIN_MAP_MUX_GROUP_HOG(dev, state, grp, func) \
    PIN_MAP_MUX_GROUP(dev, state, dev, grp, func)
```

**الـ "hog" variant يعمل الـ `ctrl_dev_name = dev`** — يعني الـ pinctrl controller نفسه هو اللي بيحجز الـ pin group لنفسه. ده بيحصل تلقائياً على الـ registration لو الـ `dev_name == ctrl_dev_name`.

الفرق المهم: الـ hog بيتطبّق أثناء `pinctrl_register()` للـ controller نفسه مش انتظار device تاني يطلبه.

---

##### `PIN_MAP_MUX_GROUP_HOG_DEFAULT(dev, grp, func)`

اختصار: hog + `state = "default"`.

---

#### النوع الثالث: Configuration Macros

##### `PIN_MAP_CONFIGS_PIN(dev, state, pinctrl, pin, cfgs)`

```c
#define PIN_MAP_CONFIGS_PIN(dev, state, pinctrl, pin, cfgs) \
    {                                                         \
        .dev_name = dev,                                      \
        .name = state,                                        \
        .type = PIN_MAP_TYPE_CONFIGS_PIN,                     \
        .ctrl_dev_name = pinctrl,                             \
        .data.configs = {                                     \
            .group_or_pin = pin,                              \
            .configs = cfgs,                                  \
            .num_configs = ARRAY_SIZE(cfgs),                  \
        },                                                    \
    }
```

**بيضبط array من الـ config values على pin واحدة بعينها.** الـ `cfgs` لازم يكون `unsigned long[]` — الـ encoding بيفرق من controller لتاني (بعضها بيعمل `(param << 16 | value)`، بعضها بيستخدم custom encoding). الـ `ARRAY_SIZE(cfgs)` بيتحسب compile-time.

| Parameter | الوصف |
|---|---|
| `dev` | اسم الـ device |
| `state` | اسم الـ state |
| `pinctrl` | اسم الـ pinctrl controller |
| `pin` | اسم الـ pin (string — زي `"PA0"` أو `"gpio0"`) |
| `cfgs` | `unsigned long[]` array من الـ config values — لازم يكون array مش pointer عشان `ARRAY_SIZE` يشتغل صح |

**مثال واقعي:**
```c
static unsigned long uart_pin_configs[] = {
    PIN_CONFIG_BIAS_PULL_UP,
    PIN_CONFIG_DRIVE_STRENGTH | (8 << 8),
};

PIN_MAP_CONFIGS_PIN("serial0", "default",
                    "pinctrl@ff030000",
                    "uart0-tx", uart_pin_configs)
```

---

##### `PIN_MAP_CONFIGS_PIN_DEFAULT(dev, pinctrl, pin, cfgs)`

اختصار: `CONFIGS_PIN` + `state = "default"`.

---

##### `PIN_MAP_CONFIGS_PIN_HOG(dev, state, pin, cfgs)`

هنا `ctrl_dev_name = dev` — الـ controller يضبط config الـ pin لنفسه.

---

##### `PIN_MAP_CONFIGS_PIN_HOG_DEFAULT(dev, pin, cfgs)`

اختصار: hog config على pin + `state = "default"`.

---

##### `PIN_MAP_CONFIGS_GROUP(dev, state, pinctrl, grp, cfgs)`

```c
#define PIN_MAP_CONFIGS_GROUP(dev, state, pinctrl, grp, cfgs) \
    {                                                           \
        .dev_name = dev,                                        \
        .name = state,                                          \
        .type = PIN_MAP_TYPE_CONFIGS_GROUP,                     \
        .ctrl_dev_name = pinctrl,                               \
        .data.configs = {                                       \
            .group_or_pin = grp,                                \
            .configs = cfgs,                                    \
            .num_configs = ARRAY_SIZE(cfgs),                    \
        },                                                      \
    }
```

**نفس `PIN_MAP_CONFIGS_PIN` بس بيطبّق الـ configs على group كاملة من الـ pins.** الـ pinctrl driver بيعمل loop على كل pin في الـ group ويطبّق نفس الـ config array عليهم.

---

##### Variants:

| Macro | الفرق |
|---|---|
| `PIN_MAP_CONFIGS_GROUP_DEFAULT` | state = "default" |
| `PIN_MAP_CONFIGS_GROUP_HOG` | `ctrl_dev_name = dev` |
| `PIN_MAP_CONFIGS_GROUP_HOG_DEFAULT` | hog + default |

---

### الصورة الكاملة — Data Flow

```
Board Code / Platform Driver
         │
         │  static struct pinctrl_map board_maps[] = {
         │      PIN_MAP_MUX_GROUP_DEFAULT(...),
         │      PIN_MAP_CONFIGS_PIN_DEFAULT(...),
         │  };
         │
         ▼
pinctrl_register_mappings(board_maps, ARRAY_SIZE(board_maps))
  OR
devm_pinctrl_register_mappings(dev, board_maps, ARRAY_SIZE(board_maps))
         │
         ▼
   pinctrl core global list (pinctrl_maps)
         │
         │  (on device probe)
         ▼
   pinctrl_get(dev)  ──►  lookup by dev_name
         │
         ▼
   pinctrl_lookup_state(p, "default")  ──►  lookup by name
         │
         ▼
   pinctrl_select_state(p, state)
         │
         ├──► MUX_GROUP  ──► pinmux_ops.set_mux()
         └──► CONFIGS_*  ──► pinconf_ops.pin_config_set()
```

---

### ملاحظات مهمة للـ Embedded Developer

- **الـ `cfgs` array في الـ CONFIGS macros** لازم يكون `unsigned long[]` array كاملة مش pointer — لأن الـ macro بتستخدم `ARRAY_SIZE()` اللي بتعمل `sizeof(arr)/sizeof(arr[0])`، وده بيفشل على الـ pointers compile-time.
- **الـ HOG entries** بتتطبق فوراً على `pinctrl_register_controller()` مش على `pinctrl_get()`. مفيدة للـ pins اللي لازم تبقى configured من أول ما الـ controller يتسجّل (زي الـ I2C pins للـ PMIC).
- **الـ `CONFIG_PINCTRL=n`** stub implementations كلها ترجع 0 أو no-op — الـ code بيـ compile بدون pinctrl بدون مشاكل.
- **الـ `grp = NULL`** في `PIN_MAP_MUX_GROUP` مسموح — الـ core بياخد أول group تدعم الـ function المطلوبة.
- **الـ state names** (`PINCTRL_STATE_DEFAULT`, `PINCTRL_STATE_INIT`, `PINCTRL_STATE_IDLE`, `PINCTRL_STATE_SLEEP`) هم strings، مش enums — الـ lookup بيتم بـ `strcmp`.
## Phase 5: دليل الـ Debugging الشامل

> **السياق:** الـ `pinctrl/machine.h` بيعرّف الـ machine-level mapping table للـ pinctrl subsystem — يعني الـ `struct pinctrl_map` اللي بتربط كل device بالـ pin groups والـ functions والـ configs بتاعتها. الـ debugging هنا بيتمحور حول: هل الـ mappings اتسجلت صح؟ هل الـ mux/config اتطبق؟ هل الـ state transitions شغّالة؟

---

### Software Level

#### 1. debugfs Entries

الـ pinctrl subsystem بيعمل expose كامل في `/sys/kernel/debug/pinctrl/`:

```
/sys/kernel/debug/pinctrl/
├── <controller-name>/
│   ├── pins          ← كل الـ pins المسجلين مع أرقامهم
│   ├── pingroups     ← الـ groups المتاحة
│   ├── pinmux-functions  ← كل الـ mux functions المسجلة
│   ├── pinmux-pins   ← mapping: كل pin → function حالية
│   ├── pinconf-pins  ← configs حالية لكل pin
│   └── gpio-ranges   ← GPIO ranges المربوطة
└── pinctrl-maps      ← كل الـ struct pinctrl_map المسجلة (المهم هنا)
```

**الأهم للـ machine.h debugging:**

```bash
# اقرأ كل الـ mappings المسجلة من pinctrl_register_mappings()
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# اقرأ الـ mux state الحالية لكل pin
cat /sys/kernel/debug/pinctrl/<ctrl-name>/pinmux-pins

# اقرأ الـ pin configs الحالية
cat /sys/kernel/debug/pinctrl/<ctrl-name>/pinconf-pins

# شوف الـ groups المتاحة وعدد pins كل group
cat /sys/kernel/debug/pinctrl/<ctrl-name>/pingroups
```

**تفسير output الـ pinctrl-maps:**

```
Pinctrl maps:
device serial0-0    state default   type MUX_GROUP (2)
device serial0-0    state default   type CONFIGS_PIN (3)
  group uart0grp    function uart0
```

- **device**: الـ `dev_name` في الـ `struct pinctrl_map`
- **state**: الـ `name` (default/init/sleep/idle)
- **type**: نوع الـ map entry (MUX_GROUP=2, CONFIGS_PIN=3, CONFIGS_GROUP=4)
- **group / function**: القيم من الـ `struct pinctrl_map_mux`

---

#### 2. sysfs Entries

```bash
# تحقق إن الـ pinctrl device موجود
ls /sys/bus/platform/drivers/pinctrl/

# شوف الـ consumers (الـ devices اللي بتستخدم pinctrl)
ls /sys/class/pinctrl/

# لو link_consumers=true في pinctrl_desc، شوف الـ device links
ls /sys/bus/platform/devices/<device>/consumer:*/
cat /sys/bus/platform/devices/<device>/consumer:*/status
```

---

#### 3. ftrace — Tracepoints/Events

```bash
# فعّل كل events الخاصة بالـ pinctrl
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# أو فعّل events محددة
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_map_add/enable
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_state_init/enable
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_get_err/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ device أو اعمل probe
modprobe <your_driver>

# اقرأ النتائج
cat /sys/kernel/debug/tracing/trace

# وقف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**function tracer لتتبع الـ pinctrl_register_mappings:**

```bash
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'pinctrl_register_mappings' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'pinctrl_select_state' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'pin_request' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. printk / Dynamic Debug

```bash
# فعّل dynamic debug لكل الـ pinctrl subsystem
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control

# أو لملفات محددة في الـ core
echo 'file drivers/pinctrl/core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinconf.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ driver اللي بتستخدمه (مثال: pinctrl-bcm2835)
echo 'module pinctrl_bcm2835 +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل من kernel cmdline
dyndbg="module pinctrl +p"
```

**من الـ boot cmdline لتتبع مشاكل early init:**

```
pinctrl.debug=1
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_PINCTRL` | تفعيل الـ subsystem الأساسي |
| `CONFIG_DEBUG_PINCTRL` | تفعيل debug messages في الـ pinctrl core |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | generic group management مع debugging |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | generic mux debugging |
| `CONFIG_GENERIC_PINCONF` | generic pin config مع verbose output |
| `CONFIG_DEBUG_FS` | لازم يكون مفعّل عشان `/sys/kernel/debug/pinctrl/` |
| `CONFIG_PROVE_LOCKING` | يكتشف locking issues في الـ pinctrl core |
| `CONFIG_LOCKDEP` | يتحقق من صحة الـ lock ordering |
| `CONFIG_KASAN` | يكتشف memory issues في الـ map registration |
| `CONFIG_DYNAMIC_DEBUG` | لتفعيل الـ `pr_debug()` calls |

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# أداة pinctrl في busybox / buildroot (لو متاحة)
pinctrl list          # كل الـ pins
pinctrl info <pin>    # معلومات pin محدد
pinctrl get <pin>     # الـ mux function الحالية
pinctrl set <pin> <func>  # تغيير الـ function

# عبر sysfs مباشرة (لو مفيش أداة)
# تحقق إن الـ device طلب pinctrl state
grep -r "pinctrl" /sys/kernel/debug/pinctrl/pinctrl-maps | grep "your-device"
```

---

#### 7. جدول أشهر رسائل الخطأ

| رسالة الخطأ | السبب | الحل |
|---|---|---|
| `could not get pinctrl, -EPROBE_DEFER` | الـ pinctrl controller لسه ما تسجلش | انتظر probe order يتصلح، أو تحقق من `of_pinctrl_get()` |
| `pin already requested` | pin اتطلب من device تاني | شوف `/sys/kernel/debug/pinctrl/<ctrl>/pinmux-pins` |
| `could not find function <func>` | الـ function اسمها غلط في الـ map | قارن الاسم في `PIN_MAP_MUX_GROUP` مع الـ driver الـ `pmxops->get_function_name()` |
| `could not find group <grp>` | الـ group اسمها غلط | تحقق من `pinctrl-maps` و`pingroups` في debugfs |
| `could not find state <state>` | الـ state name مش موجود في الـ map | تأكد إن `PIN_MAP_MUX_GROUP` بيستخدم نفس الـ state string |
| `No maps for state default` | مفيش map entry لهذا الـ device/state | أضف `PIN_MAP_DUMMY_STATE` أو map حقيقي |
| `mapping table not correctly sized` | `num_maps` أصغر من عدد الـ entries الفعلي | استخدم `ARRAY_SIZE()` بدل رقم ثابت |
| `pinctrl_register_mappings: -ENOMEM` | نفاد ذاكرة أثناء التسجيل | زود `CONFIG_CMA_SIZE_MBYTES` أو تحقق من memory pressure |
| `pin does not belong to requested group` | config لـ pin مش في الـ group | تحقق من `struct pinctrl_map_configs.group_or_pin` |
| `devm_pinctrl_register_mappings: device is NULL` | الـ dev pointer فاضي | تأكد إن الـ device اتسجل قبل الـ mappings |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في دالة pinctrl_register_mappings() أو الـ driver اللي بيسجل الـ maps */

/* تحقق من صحة الـ map array قبل التسجيل */
for (i = 0; i < num_maps; i++) {
    WARN_ON(!map[i].dev_name);
    WARN_ON(!map[i].name);
    WARN_ON(map[i].type == PIN_MAP_TYPE_INVALID);

    if (map[i].type == PIN_MAP_TYPE_MUX_GROUP) {
        WARN_ON(!map[i].ctrl_dev_name);
        /* group يمكن يكون NULL (يستخدم أول group)، function لازم يكون موجود */
        WARN_ON(!map[i].data.mux.function);
    }
}

/* في الـ driver عند طلب state */
state = pinctrl_lookup_state(pctrl, PINCTRL_STATE_DEFAULT);
if (IS_ERR(state)) {
    WARN_ON(PTR_ERR(state) != -ENODEV); /* ENODEV معناه مفيش map — مش bug */
    dump_stack(); /* لو error غير متوقع */
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

```bash
# الخطوة 1: اعرف الـ pin رقم من debugfs
cat /sys/kernel/debug/pinctrl/<ctrl>/pins
# مثال output:
# pin 42 (GPIO_42) reg 0x100a8 offset 42

# الخطوة 2: شوف الـ mux المطبق حالياً في kernel
cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-pins | grep "pin 42"
# output: pin 42 (GPIO_42): UART0 [REQUESTED] (GPIO_42)

# الخطوة 3: تحقق من الـ register الفعلي في الـ hardware
# (راجع datasheet للـ SoC لمعرفة base address)
devmem2 0x100a8 w   # اقرأ الـ register
```

#### 2. Register Dump Techniques

```bash
# devmem2 (الأشيع)
devmem2 <phys_addr> [b|h|w]   # b=byte, h=halfword, w=word

# مثال: SoC بـ pinmux base = 0x02000000
devmem2 0x02000000 w   # اقرأ أول register

# لو devmem2 مش متاح، استخدم /dev/mem
dd if=/dev/mem bs=4 count=1 skip=$((0x02000000/4)) 2>/dev/null | hexdump -e '1/4 "0x%08x\n"'

# io utility (من package io أو ioport)
io -4 0x02000000

# للـ memory-mapped registers في kernel space (من module)
# استخدم ioremap() + ioread32()
```

**مثال script لعمل dump لـ pinmux registers:**

```bash
#!/bin/bash
# dump pinmux registers for i.MX8 (example)
BASE=0x30330000
echo "Pinmux register dump:"
for offset in $(seq 0 4 64); do
    addr=$((BASE + offset))
    printf "0x%08x: " $addr
    devmem2 $addr w 2>/dev/null | grep "Value" | awk '{print $NF}'
done
```

#### 3. Logic Analyzer / Oscilloscope Tips

**لتتبع مشاكل الـ pin configuration:**

- **قبل اي probe:** اتصل بالـ logic analyzer على الـ pin المشكوك فيه
- **تحقق من الـ drive strength:** الـ `configs` في `struct pinctrl_map_configs` بتتحكم في الـ drive strength — قس الـ slew rate بالـ oscilloscope
- **تحقق من الـ pull-up/pull-down:** قس الـ voltage level لما الـ pin مش driven — لو 0V فالـ pull-down شغّال، لو VDD فالـ pull-up شغّال
- **الـ glitches عند state transition:** سجّل الـ pin أثناء `pinctrl_select_state()` — لو في glitch يعني الـ `PINCTRL_STATE_INIT` محتاج يتضاف

```
Oscilloscope setup للـ UART debugging:
- Channel 1: TX pin
- Channel 2: RX pin
- Trigger: rising edge على CH1
- Time division: 1μs/div لـ 115200 baud
```

#### 4. Hardware Issues → Kernel Log Patterns

| المشكلة الهاردوير | الـ Pattern في الـ Kernel Log |
|---|---|
| Pin مش connected للـ SoC pinmux | `could not get pinctrl` + الـ device ما بيشتغلش |
| Pinmux register base address غلط في DT | `could not find controller` أو silent failure |
| الـ SoC مش بيدعم الـ function المطلوبة | `could not find function <name>` |
| Drive strength config غلط | الـ device بيشتغل لكن unstable أو slow |
| Power domain للـ pin مش enabled | `pin already requested` أو `pin not owned` |
| Boot ROM غيّر الـ mux ومش رجعه | Default state بيفشل أو بيحتاج `PINCTRL_STATE_INIT` |

#### 5. Device Tree Debugging

**تحقق إن الـ DT يطابق الـ hardware:**

```bash
# اقرأ الـ DT المحمّل فعلاً (compiled DT في الذاكرة)
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "pinctrl"

# تحقق من الـ compatible string
cat /sys/firmware/devicetree/base/soc/pinctrl@30330000/compatible

# تحقق من الـ pin states في node الـ device
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 10 "pinctrl-"
```

**أشيع مشاكل الـ DT مع الـ pinctrl mapping:**

```dts
/* صح: device بيشير لـ pin state */
&uart0 {
    pinctrl-names = "default", "sleep";   /* لازم تطابق الـ state names في الـ map */
    pinctrl-0 = <&uart0_default_pins>;    /* reference للـ pin configuration node */
    pinctrl-1 = <&uart0_sleep_pins>;
    status = "okay";
};

/* غلط شائع: mismatch في الاسم */
pinctrl-names = "defaults";   /* "defaults" بدل "default" → state مش هيتلاقي */
```

```bash
# تحقق من الـ pinctrl-names في الـ DT
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep "pinctrl-names"

# لو الـ DT من ملف .dtb على disk
dtc -I dtb -O dts /boot/dtbs/*.dtb 2>/dev/null | grep -A 5 "pinctrl-names"
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

**1. نظرة عامة سريعة على الـ pinctrl system:**

```bash
# كل الـ controllers المسجلة
ls /sys/kernel/debug/pinctrl/

# كل الـ mappings (from machine.h structs)
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# الـ mux الحالي لكل pin
CTRL=$(ls /sys/kernel/debug/pinctrl/ | grep -v pinctrl-maps | head -1)
cat /sys/kernel/debug/pinctrl/$CTRL/pinmux-pins
```

**مثال output للـ pinctrl-maps:**

```
Pinctrl maps:
device 2000000.uart  state default  type MUX_GROUP (2)
  group uart1grp  function uart1  controller 2290000.iomuxc
device 2000000.uart  state default  type CONFIGS_PIN (3)
  pin MX6Q_PAD_SD3_DAT7  configs 0x1b0b1
  pin MX6Q_PAD_SD3_DAT6  configs 0x1b0b1
```

**التفسير:**
- `device 2000000.uart` = الـ `dev_name` في `struct pinctrl_map`
- `state default` = الـ `name` = `PINCTRL_STATE_DEFAULT`
- `type MUX_GROUP (2)` = `PIN_MAP_TYPE_MUX_GROUP`
- `group uart1grp` = `data.mux.group`
- `function uart1` = `data.mux.function`
- `configs 0x1b0b1` = القيمة من `data.configs.configs[]`

**2. تتبع مشكلة device ما بيشتغلش:**

```bash
# هل الـ device اتعمله probe؟
dmesg | grep -i "your-device"

# هل فشل في طلب الـ pinctrl؟
dmesg | grep -i "pinctrl\|pinmux\|could not get"

# هل الـ mapping اتسجل؟
grep "your-device" /sys/kernel/debug/pinctrl/pinctrl-maps

# هل الـ pin اتطلب من حد تاني؟
CTRL=$(ls /sys/kernel/debug/pinctrl/ | grep -v pinctrl-maps | head -1)
grep "REQUESTED" /sys/kernel/debug/pinctrl/$CTRL/pinmux-pins
```

**3. فعّل كامل الـ pinctrl debug logging:**

```bash
# dynamic debug
echo 'module pinctrl +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinmux.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# بعدها trigger الـ probe
echo "your-device-name" > /sys/bus/platform/drivers/your-driver/bind

# شوف النتيجة
dmesg | tail -50
```

**4. التحقق من configs مطبقة صح:**

```bash
# اقرأ الـ pin configs الحالية
CTRL=$(ls /sys/kernel/debug/pinctrl/ | grep -v pinctrl-maps | head -1)
cat /sys/kernel/debug/pinctrl/$CTRL/pinconf-pins

# مثال output:
# pin 42 (GPIO_42) config 0x0001b0b1
# → 0x1b0b1 = drive strength 6 (DSE6), pull-up enabled, speed 100MHz
```

**5. Script شامل لتشخيص مشاكل الـ pinctrl:**

```bash
#!/bin/bash
# pinctrl_debug.sh - شامل لتشخيص مشاكل الـ pinctrl/machine mapping

DBGFS="/sys/kernel/debug/pinctrl"
DEV=${1:-""}

echo "=== Registered Controllers ==="
ls $DBGFS/

echo ""
echo "=== All Pin Mappings ==="
cat $DBGFS/pinctrl-maps

if [ -n "$DEV" ]; then
    echo ""
    echo "=== Mappings for device: $DEV ==="
    grep "$DEV" $DBGFS/pinctrl-maps
fi

echo ""
echo "=== Requested Pins (mux) ==="
for ctrl in $(ls $DBGFS/ | grep -v pinctrl-maps); do
    echo "--- Controller: $ctrl ---"
    grep "REQUESTED" $DBGFS/$ctrl/pinmux-pins 2>/dev/null || echo "(none)"
done

echo ""
echo "=== Kernel Messages ==="
dmesg | grep -iE "pinctrl|pinmux|pin_request|EPROBE_DEFER" | tail -20
```

**الاستخدام:**

```bash
chmod +x pinctrl_debug.sh
./pinctrl_debug.sh "2000000.uart"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART مش شغال على بورد RK3562 — Industrial Gateway

#### العنوان
**الـ UART2 على RK3562 بيتجاهله الـ kernel تماماً أثناء bring-up**

#### السياق
بتشتغل على industrial gateway بيستخدم **RK3562** — البورد مخصصة لجمع بيانات من sensors عبر RS-485 (UART2). البورد custom، مفيش device tree جاهز، والـ BSP مبني على kernel 6.1.

#### المشكلة
الـ UART driver بيـprobe بنجاح، بس مفيش بيانات بتوصل. الـ `dmesg` مفيش فيه errors. لما بتفتح `/dev/ttyS2` وبتحاول ترسل بيانات — صمت تام.

#### التحليل
أول خطوة: تتأكد إن الـ pinctrl mapping موجود ومظبوط.

```c
// في machine.h — ده هو اللي بيربط الـ device باسم الـ state
struct pinctrl_map {
    const char *dev_name;   // لازم يطابق اسم الـ UART device تماماً
    const char *name;       // "default", "idle", ...
    enum pinctrl_map_type type;
    const char *ctrl_dev_name; // اسم الـ pinctrl controller
    union { ... } data;
};
```

لما الـ driver بيعمل `pinctrl_get()` بيدور على الـ `pinctrl_map` بالـ `dev_name`. لو الاسم غلط — بيعدي في صمت لو `pinctrl_provide_dummies()` اتعملت.

```bash
# تشوف الـ mappings الـ registered
cat /sys/kernel/debug/pinctrl/pinctrl-maps
```

الناتج أظهر:
```
device: fe650000.serial
state: default
type: MUX_GROUP (2)
controlling device: pinctrl-rk3562
group: uart2-xfer
function: uart2
```

بس اسم الـ device في الـ map كان `"fe650000.uart"` وراح يتـregister الـ driver نفسه بـ`"fe650000.serial"` — mismatch في الـ `dev_name`.

#### الحل
في الـ board file أو الـ DT:

```c
// قبل الإصلاح — اسم غلط
PIN_MAP_MUX_GROUP_DEFAULT("fe650000.uart", "pinctrl-rk3562",
                           "uart2-xfer", "uart2"),

// بعد الإصلاح — الاسم يطابق ما بيطبعه kernel
PIN_MAP_MUX_GROUP_DEFAULT("fe650000.serial", "pinctrl-rk3562",
                           "uart2-xfer", "uart2"),
```

أو لو بتستخدم DT، متعتمدش على board-level mapping خالص — خلي الـ DTS يتكفل.

```bash
# تتأكد من اسم الـ device
ls /sys/bus/platform/devices/ | grep fe650000
```

#### الدرس المستفاد
الـ `dev_name` في `struct pinctrl_map` لازم يطابق `dev_name()` بالظبط — وده بيكون عادةً `"<reg-address>.<compatible-suffix>"`. أي فرق حرف واحد والـ pinctrl mapping بياخد طريق الـ dummy state في صمت تام لو `pinctrl_provide_dummies()` اتنادت.

---

### السيناريو الثاني: SPI Flash مش بيتعرف على STM32MP1 — IoT Sensor Node

#### العنوان
**الـ SPI1 على STM32MP1 بيرجع `-EPROBE_DEFER` باستمرار**

#### السياق
بتعمل IoT sensor node بـ **STM32MP157C-DK2**. الـ node محتاج SPI1 يتكلم مع W25Q128 flash chip. الـ driver شغال على بورد تانية بنفس الـ SoC، بس على البورد الجديدة مش بيـprobe.

#### المشكلة
الـ `dmesg` بيكرر:
```
spi-nor spi0.0: error -517 (EPROBE_DEFER)
```
وده بيستمر إلى ما لا نهاية — مش بيـrecover.

#### التحليل
الـ `-EPROBE_DEFER` من الـ SPI controller نفسه لأنه لسه مش جاهز — والسبب إن الـ pinctrl state مش اتـapply. الـ kernel بيطلب الـ `"default"` state عند الـ probe.

في `machine.h`:
```c
// الـ PIN_MAP_TYPE_DUMMY_STATE بيخلي الـ kernel يعدي من غير ما يـapply أي config
#define PIN_MAP_DUMMY_STATE(dev, state) \
    { \
        .dev_name = dev, \
        .name = state, \
        .type = PIN_MAP_TYPE_DUMMY_STATE, \
    }
```

لما بعمل:
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-maps | grep spi
```

لقيت:
```
device: spi1
state: default
type: DUMMY_STATE (1)
```

يعني `pinctrl_provide_dummies()` اتنادت وخلت كل device مش عندها mapping تاخد dummy state — بس المشكلة إن الـ SPI controller على هذه البورد **محتاج فعلاً** الـ mux يتـapply عشان يشتغل.

المشكلة الأصلية: الـ board initialization بتعمل `pinctrl_provide_dummies()` قبل ما تـregister الـ mappings الصح.

```c
// ترتيب خاطئ في board init
pinctrl_provide_dummies();   // كل device هتاخد dummy
pinctrl_register_mappings(board_maps, ARRAY_SIZE(board_maps)); // متأخر!
```

#### الحل
تتأكد إن الـ `pinctrl_register_mappings()` بتتنادى **قبل** `pinctrl_provide_dummies()`:

```c
// ترتيب صح
int ret = pinctrl_register_mappings(stm32mp1_board_maps,
                                    ARRAY_SIZE(stm32mp1_board_maps));
if (ret)
    pr_err("Failed to register pin mappings: %d\n", ret);

/* دلوقتي safe نعمل provide_dummies للـ devices التانية */
pinctrl_provide_dummies();
```

أو الأسهل: استخدم DT بدل board-level mappings خالص.

#### الدرس المستفاد
الـ `pinctrl_provide_dummies()` سلاح ذو حدين — بتسمح لـ devices من غير pinctrl mapping تشتغل، بس ممكن تـmask مشاكل حقيقية لو اتنادت قبل ما الـ mappings الصح تتـregister. الترتيب في الـ board init بالغ الأهمية.

---

### السيناريو الثالث: I2C Touchscreen على i.MX8MQ — Android TV Box

#### العنوان
**الـ touchscreen I2C بيشتغل بس بيـglitch لما الشاشة تتفتح**

#### السياق
Android TV box بـ **i.MX8MQ**، بيستخدم GT911 touchscreen على I2C3. المنتج في مرحلة mass production والـ QA بيرصد glitches في أول 2 ثانية بعد ما الشاشة تصحى من sleep.

#### المشكلة
لما الـ display بيطلع من `PINCTRL_STATE_SLEEP`، الـ touch أحياناً بيسجل touches وهمية لمدة 100-200ms. مش بيحصل دايماً — نسبة 30%.

#### التحليل
في `pinctrl-state.h`:
```c
#define PINCTRL_STATE_DEFAULT "default"
#define PINCTRL_STATE_SLEEP   "sleep"
```

الـ I2C3 pins لما بيرجعوا من sleep بيمروا بـ default state مباشرة. المشكلة إن الـ GT911 محتاج الـ pins يكونوا stable قبل ما الـ interrupt line يتـenable.

الـ mapping table في الـ board file كانت:
```c
static const struct pinctrl_map imx8mq_touch_maps[] = {
    /* default — SDA/SCL مع pull-up */
    PIN_MAP_MUX_GROUP_DEFAULT("3-005d", "pinctrl-imx8mq",
                               "i2c3-grp", "i2c3"),
    PIN_MAP_CONFIGS_PIN_DEFAULT("3-005d", "pinctrl-imx8mq",
                                 "i2c3-sda", touch_sda_cfg),

    /* sleep — pins كـGPIO input بدون pull */
    PIN_MAP_MUX_GROUP("3-005d", PINCTRL_STATE_SLEEP, "pinctrl-imx8mq",
                       "i2c3-sleep-grp", "gpio"),
};
```

المشكلة: مفيش `PINCTRL_STATE_INIT` mapping! لما الـ driver بيـprobe، الـ kernel بيحاول يطبق `"init"` state الأول، لو مش لاقيه بيروح لـ`"default"` مباشرة. من غير `"init"` state، الانتقال من sleep لـ default بيحصل بدون grace period.

#### الحل
إضافة `init` state بـ timing مختلف:

```c
/* configs للـ init state — بدون pull-up في البداية */
static unsigned long touch_init_sda_cfg[] = {
    PIN_CONFIG_BIAS_DISABLE,
    PIN_CONFIG_DRIVE_OPEN_DRAIN,
};

static const struct pinctrl_map imx8mq_touch_maps[] = {
    /* init — بدون pull-up عشان نمنع glitch */
    PIN_MAP_MUX_GROUP("3-005d", PINCTRL_STATE_INIT, "pinctrl-imx8mq",
                       "i2c3-grp", "i2c3"),
    PIN_MAP_CONFIGS_PIN("3-005d", PINCTRL_STATE_INIT, "pinctrl-imx8mq",
                         "i2c3-sda", touch_init_sda_cfg),

    /* default — pull-up كاملة بعد ما الـ chip جاهز */
    PIN_MAP_MUX_GROUP_DEFAULT("3-005d", "pinctrl-imx8mq",
                               "i2c3-grp", "i2c3"),
    PIN_MAP_CONFIGS_PIN_DEFAULT("3-005d", "pinctrl-imx8mq",
                                 "i2c3-sda", touch_sda_cfg),

    /* sleep */
    PIN_MAP_MUX_GROUP("3-005d", PINCTRL_STATE_SLEEP, "pinctrl-imx8mq",
                       "i2c3-sleep-grp", "gpio"),
};
```

#### الدرس المستفاد
الـ `PINCTRL_STATE_INIT` موجود لسبب وجيه — هو الـ state اللي بيـapply قبل الـ `probe()` عشان يمنع glitches. الـ kernel بعد الـ probe لو لقى الـ pins لسه في `"init"` بيحولهم لـ `"default"` أوتوماتيك. تجاهل الـ init state في devices حساسة بيسبب race conditions في الـ hardware.

---

### السيناريو الرابع: USB Host بيتعطل على AM62x — Automotive ECU

#### العنوان
**الـ USB2 host port على AM62x بيـdisconnect كل 30 دقيقة في بيئة automotive**

#### السياق
ECU في نظام infotainment بـ **TI AM62x SoC**. الـ USB host port بيتكلم مع USB-to-CAN adapter. في بيئة الاختبار (مكيف) كل حاجة تمام. في السيارة (درجة حرارة متغيرة، vibration) — الـ USB بيـdisconnect.

#### المشكلة
الـ dmesg بيظهر:
```
usb 1-1: USB disconnect, device number 2
```
بعد 25-35 دقيقة من التشغيل. مفيش hardware fault واضح.

#### التحليل
بعد تحليل عميق، المشكلة في الـ `idle` pinctrl state. الـ power management كان بيحط الـ USB controller في `PINCTRL_STATE_IDLE` أثناء الـ `pm_runtime_idle()`:

```c
// من machine.h — الـ idle state
#define PINCTRL_STATE_IDLE "idle"
```

الـ board mapping كان:
```c
static const struct pinctrl_map am62x_usb_maps[] = {
    PIN_MAP_MUX_GROUP_DEFAULT("dwc3.0", "pinctrl-am62x",
                               "usb0-grp", "usb0"),

    /* idle state — خفف الـ drive strength */
    PIN_MAP_MUX_GROUP("dwc3.0", PINCTRL_STATE_IDLE, "pinctrl-am62x",
                       "usb0-grp", "usb0"),
    PIN_MAP_CONFIGS_GROUP("dwc3.0", PINCTRL_STATE_IDLE, "pinctrl-am62x",
                           "usb0-grp", usb_idle_cfg),
};

/* الـ idle config */
static unsigned long usb_idle_cfg[] = {
    PIN_CONFIG_DRIVE_STRENGTH, 2,  // خففنا الـ drive strength لـ 2mA
};
```

الـ 2mA drive strength كان كافي في بيئة الـ lab، بس في السيارة الـ signal integrity بتتأثر بالـ EMI والـ temperature → الـ USB VBUS detection بتفشل.

```bash
# تتحقق من الـ current state
cat /sys/kernel/debug/pinctrl/pinctrl-am62x/pinconf-pins | grep usb0
```

#### الحل
تعديل الـ `idle` state config لتبقى بنفس قوة الـ default:

```c
/* رفعنا الـ drive strength في idle state */
static unsigned long usb_idle_cfg[] = {
    PIN_CONFIG_DRIVE_STRENGTH, 8,  // نفس الـ default
    PIN_CONFIG_BIAS_PULL_UP, 1,    // keep pull-up active
};
```

أو الأسلم: إزالة الـ `idle` state mapping خالص للـ USB — يفضل في `default` دايماً:

```c
// بنستخدم devm_pinctrl_register_mappings عشان يتنظف لو الـ driver اتـunload
ret = devm_pinctrl_register_mappings(&pdev->dev,
                                      am62x_usb_maps,
                                      ARRAY_SIZE(am62x_usb_maps));
```

#### الدرس المستفاد
الـ `PINCTRL_STATE_IDLE` ممكن يوفر power لكن في بيئات automotive ذات EMI عالي، تخفيض drive strength على USB أو Ethernet lines خطر. لازم تختبر على نطاق حرارة كامل (-40°C إلى +85°C) قبل production.

---

### السيناريو الخامس: HDMI Audio مش شغال على Allwinner H616 — Android TV Box

#### العنوان
**الـ HDMI audio output صامت على Allwinner H616 رغم إن الـ video شغال**

#### السياق
Android TV box رخيص بـ **Allwinner H616**. الـ video على HDMI تمام. الـ audio على HDMI صمت تام. اشتكت منه دفعة من 500 وحدة في السوق. الـ BSP مبني على kernel 5.15 معدّل.

#### المشكلة
الـ `aplay -l` بيظهر الـ HDMI audio device. الـ `aplay` بيشتغل من غير errors. بس الـ TV مش بتسمع صوت.

#### التحليل
الأول تتأكد إن الـ I2S pins اللي بتنقل audio data لـ HDMI encoder مظبوطة:

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-maps | grep -A5 "hdmi\|i2s"
```

الناتج:
```
device: hdmi-audio-codec
state: default
type: DUMMY_STATE (1)
```

الـ HDMI audio device بياخد **DUMMY_STATE**! يعني مفيش pin configuration بيتطبق عليه. الـ BSP developer عمل `pinctrl_provide_dummies()` وما عملش mapping للـ HDMI audio codec.

في `machine.h`:
```c
enum pinctrl_map_type {
    PIN_MAP_TYPE_INVALID,
    PIN_MAP_TYPE_DUMMY_STATE,  // ده اللي بيتطبق — لا mux، لا configs
    PIN_MAP_TYPE_MUX_GROUP,
    PIN_MAP_TYPE_CONFIGS_PIN,
    PIN_MAP_TYPE_CONFIGS_GROUP,
};
```

الـ I2S0 pins اللي المفروض تنقل audio للـ HDMI encoder كانت لازم تتـconfigure كـI2S function، بس بسبب الـ dummy state اتفضلوا في GPIO mode.

```bash
# تتأكد من الـ pin function الحالي
cat /sys/kernel/debug/pinctrl/pinctrl-h616/pinmux-pins | grep PH
# PH4: gpio — المشكلة! المفروض I2S0
```

#### الحل
إضافة mapping حقيقي للـ HDMI audio في الـ board file:

```c
/* H616 I2S0 pins للـ HDMI audio */
static const struct pinctrl_map h616_hdmi_audio_maps[] = {
    /* MUX: I2S0 function على الـ PH group */
    PIN_MAP_MUX_GROUP_DEFAULT("hdmi-audio-codec",
                               "pinctrl-h616",
                               "i2s0-grp",
                               "i2s0"),

    /* Config: drive strength مناسب للـ audio */
    PIN_MAP_CONFIGS_GROUP_DEFAULT("hdmi-audio-codec",
                                   "pinctrl-h616",
                                   "i2s0-grp",
                                   h616_i2s_cfg),
};

static int __init h616_board_init(void)
{
    return pinctrl_register_mappings(h616_hdmi_audio_maps,
                                     ARRAY_SIZE(h616_hdmi_audio_maps));
}
arch_initcall(h616_board_init);
```

أو في DTS:

```dts
&i2s0 {
    pinctrl-names = "default";
    pinctrl-0 = <&i2s0_pins>;
    status = "okay";
};
```

#### الدرس المستفاد
الـ `PIN_MAP_TYPE_DUMMY_STATE` بيمنع الـ kernel panic لكنه ممكن يخبي features مش شغالة بالكامل. في كل bring-up جديد، افحص `pinctrl-maps` في debugfs لكل device وتأكد إن النوع `MUX_GROUP` وليس `DUMMY_STATE` للـ peripherals الفعلية. الـ audio خصوصاً بياخد dummy بسهولة لأن الـ driver بيـprobe حتى من غير pins صح.
## Phase 7: مصادر ومراجع

---

### مصادر LWN.net

دي أهم المقالات اللي بتغطي الـ pinctrl subsystem وتحديداً الـ machine interface:

| المقالة | الوصف |
|---------|--------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | أهم مقالة — بتشرح الفكرة الكاملة للـ subsystem، والـ mapping بين الـ devices والـ pin controllers |
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | الـ patch الأولاني اللي أنشأ الـ subsystem |
| [drivers: create a pin control subsystem v8](https://lwn.net/Articles/460768/) | الـ patch النهائي قبل الـ merge — بيوضح التطور |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | مرحلة تطوير متقدمة — بتفيد في فهم كيف اتشكلت الـ API |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | الـ documentation الرسمي كما ظهر في الـ mailing list |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/) | إضافة الـ `PIN_MAP_TYPE_CONFIGS_*` للـ mapping table |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | تفاصيل الـ config parameters اللي بتتخزن في `pinctrl_map_configs` |
| [drivers/pinctrl: Add the concept of an "init" state](https://lwn.net/Articles/615322/) | إضافة الـ `PINCTRL_STATE_INIT` — مهم لفهم الـ state machine |
| [drivers: create a pinmux subsystem v2](https://lwn.net/Articles/442315/) | النسخة الأولى قبل ما تتحول لـ pinctrl — تاريخ الـ subsystem |

---

### التوثيق الرسمي في الـ Kernel

**الـ `Documentation/` الرسمية:**

- **`Documentation/driver-api/pin-control.rst`** — التوثيق الكامل للـ subsystem
  - الرابط الرسمي: [https://docs.kernel.org/driver-api/pin-control.html](https://docs.kernel.org/driver-api/pin-control.html)
  - بيشرح الـ `struct pinctrl_map`، الـ macros، وطريقة التسجيل

- **`Documentation/devicetree/bindings/pinctrl/`** — الـ Device Tree bindings للـ pin controllers المختلفة

- **الـ `include/linux/pinctrl/`** — الـ headers الأساسية:
  - `machine.h` — الـ machine interface (الملف ده نفسه)
  - `pinctrl-state.h` — تعريفات الـ states (DEFAULT, INIT, IDLE, SLEEP)
  - `consumer.h` — الـ API للـ consumer drivers
  - `pinctrl.h` — الـ core API للـ controllers

---

### كومِتات مهمة في الـ Kernel

| الـ Commit | الوصف |
|-----------|--------|
| [بحث في git](https://github.com/torvalds/linux/commits/master/include/linux/pinctrl/machine.h) | تاريخ التعديلات على `machine.h` كله |

**أهم commits تاريخياً:**
- الـ commit الأولاني للـ subsystem — **Linus Walleij** في 2011 لـ kernel 3.2، كتبه على behalf of Linaro لـ ST-Ericsson
- إضافة `devm_pinctrl_register_mappings` — بعدين لضمان الـ automatic cleanup
- إضافة `pinctrl_provide_dummies` — للـ platforms اللي مش محتاجة فعلياً pinctrl

للبحث اليدوي:
```bash
git log --oneline -- include/linux/pinctrl/machine.h
git log --oneline -- drivers/pinctrl/core.c | head -20
```

---

### نقاشات الـ Mailing List

- **[LKML Archive — linux-gpio list](https://www.spinics.net/lists/kernel/)** — الـ pinctrl patches بتمشي على `linux-gpio@vger.kernel.org`
- **[PATCH v8 00/10 — Add pinctrl support for AAEON UP board FPGA](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg09700.html)** — مثال حديث على استخدام الـ machine interface
- **Linaro Git — Linus Walleij's tree**: [https://git.linaro.org/people/linus.walleij/linux-pinctrl.git](https://git.linaro.org/people/linus.walleij/linux-pinctrl.git)

للبحث في الـ mailing list مباشرة:
```
https://lore.kernel.org/linux-gpio/?q=pinctrl+machine
```

---

### مصادر eLinux.org

| الصفحة | الفائدة |
|--------|---------|
| [Pin Control GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) | presentation بيشرح العلاقة بين الـ pinctrl والـ GPIO subsystems |
| [EBC Device Trees](https://elinux.org/EBC_Device_Trees) | أمثلة عملية على الـ pinctrl في الـ Device Tree على BeagleBone |
| [Tests: i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) | مثال على الـ runtime switching بين الـ states |

---

### kernelnewbies.org

الموقع بيسجّل التغييرات في كل إصدار kernel. ابحث فيه عن التغييرات الخاصة بالـ pinctrl:

| الصفحة | الفائدة |
|--------|---------|
| [Linux 6.11 Changes](https://kernelnewbies.org/Linux_6.11) | آخر تحديثات الـ pinctrl drivers |
| [Linux 6.8 Changes](https://kernelnewbies.org/Linux_6.8) | إضافات pinctrl drivers جديدة |
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | فهرس كل الإصدارات — ابحث بكلمة "pinctrl" |

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 14** — The Linux Device Model
- بيشرح الـ `struct device` وكيف الـ `dev_name()` بيتطابق مع الـ `dev_name` في `struct pinctrl_map`
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول 13–14** — The Virtual Filesystem, The Block I/O Layer
- للمفاهيم الأساسية للـ subsystem design في الـ kernel
- مفيد لفهم فلسفة تصميم الـ `devm_*` functions

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 16** — Porting Linux to a Custom Platform
- بيتكلم عن الـ board files والـ machine setup — نفس السياق اللي بتشتغل فيه `pinctrl_map`
- بيشرح كيف الـ machine-level code بيربط الـ hardware بالـ drivers

#### Linux Device Driver Development — John Madieu (2nd Edition, Packt)
- **الفصل 13** — Pin Control and GPIO Subsystem
- أحدث مرجع متخصص في الـ pinctrl
- بيغطي الـ `pinctrl_map`, الـ macros, والـ DT integration بالتفصيل

---

### مقالات خارجية متخصصة

| المقالة | الرابط |
|---------|--------|
| Linux device driver development: The pin control subsystem | [embedded.com](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) |
| PINCTRL subsystem — Kernel docs v4.14 | [kernel.org](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html) |
| Pinctrl overview — STM32MPU Wiki | [wiki.st.com](https://wiki.st.com/stm32mpu/wiki/Pinctrl_overview) |
| pinctrl.txt — Android kernel (Google) | [android.googlesource.com](https://android.googlesource.com/kernel/common/+/bcmdhd-3.10/Documentation/pinctrl.txt) |

---

### مصطلحات البحث

للقاء معلومات أكتر، استخدم الـ search terms دي:

```
linux kernel pinctrl_map machine interface
pinctrl_register_mappings board file
PIN_MAP_MUX_GROUP device tree vs board file
pinctrl state machine DEFAULT SLEEP IDLE
linux pinmux group function mapping
devm_pinctrl_register_mappings resource management
pinctrl hog driver self-registration
linux pinctrl machine.h Linus Walleij ST-Ericsson
```

**للبحث في الـ kernel source مباشرة:**
```bash
# كل الـ drivers اللي بتستخدم pinctrl_register_mappings
grep -r "pinctrl_register_mappings" drivers/ arch/ --include="*.c" -l

# أمثلة على الـ PIN_MAP_MUX_GROUP macro
grep -r "PIN_MAP_MUX_GROUP" arch/ --include="*.c" | head -20

# كل الـ board files اللي بتعمل pinctrl mapping
grep -r "struct pinctrl_map" arch/ --include="*.c" -l
```
## Phase 8: Writing simple module

### الفكرة

**`pinctrl_register_mappings()`** هي الدالة الأنسب للـ hook — بيتم استدعاؤها لما أي driver أو board يسجل mapping table جديدة في الـ pinctrl subsystem. باستخدام **kprobe**، هنقدر نشوف كل مرة بيتسجل فيها mappings وعدد الـ entries.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe module: hooks pinctrl_register_mappings()
 * Prints dev_name, map type, and count on every call.
 */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/pinctrl/machine.h> /* struct pinctrl_map, pinctrl_map_type */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs");
MODULE_DESCRIPTION("kprobe on pinctrl_register_mappings — logs every mapping registration");

/* ------------------------------------------------------------------ */
/* Helper: اسم نصي لكل نوع mapping عشان الـ log يبقى مفهوم           */
/* ------------------------------------------------------------------ */
static const char *map_type_str(enum pinctrl_map_type t)
{
    switch (t) {
    case PIN_MAP_TYPE_DUMMY_STATE:   return "DUMMY_STATE";
    case PIN_MAP_TYPE_MUX_GROUP:     return "MUX_GROUP";
    case PIN_MAP_TYPE_CONFIGS_PIN:   return "CONFIGS_PIN";
    case PIN_MAP_TYPE_CONFIGS_GROUP: return "CONFIGS_GROUP";
    default:                         return "INVALID";
    }
}

/* ------------------------------------------------------------------ */
/* pre-handler: بيتشغل قبل ما pinctrl_register_mappings تنفذ فعلياً  */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * ABI (x86-64): arg0 → RDI (map pointer), arg1 → RSI (num_maps)
     * On ARM64   : arg0 → x0,                  arg1 → x1
     * regs_get_kernel_argument() abstracts both.
     */
    const struct pinctrl_map *map =
        (const struct pinctrl_map *)regs_get_kernel_argument(regs, 0);
    unsigned int num_maps =
        (unsigned int)regs_get_kernel_argument(regs, 1);
    unsigned int i;

    if (!map || num_maps == 0) {
        pr_info("pinctrl_probe: called with NULL/empty map\n");
        return 0;
    }

    pr_info("pinctrl_probe: registering %u map(s)\n", num_maps);

    /* طبّع أول 8 entries بحد أقصى عشان ما نملاش الـ dmesg */
    for (i = 0; i < num_maps && i < 8; i++) {
        const struct pinctrl_map *m = &map[i];

        switch (m->type) {
        case PIN_MAP_TYPE_MUX_GROUP:
            pr_info("  [%u] dev=%-24s state=%-12s type=%-14s grp=%s func=%s\n",
                    i,
                    m->dev_name      ? m->dev_name      : "(null)",
                    m->name          ? m->name          : "(null)",
                    map_type_str(m->type),
                    m->data.mux.group    ? m->data.mux.group    : "*",
                    m->data.mux.function ? m->data.mux.function : "(null)");
            break;

        case PIN_MAP_TYPE_CONFIGS_PIN:
        case PIN_MAP_TYPE_CONFIGS_GROUP:
            pr_info("  [%u] dev=%-24s state=%-12s type=%-14s pin/grp=%s num_cfgs=%u\n",
                    i,
                    m->dev_name ? m->dev_name : "(null)",
                    m->name     ? m->name     : "(null)",
                    map_type_str(m->type),
                    m->data.configs.group_or_pin ? m->data.configs.group_or_pin : "(null)",
                    m->data.configs.num_configs);
            break;

        default:
            pr_info("  [%u] dev=%-24s state=%-12s type=%s\n",
                    i,
                    m->dev_name ? m->dev_name : "(null)",
                    m->name     ? m->name     : "(null)",
                    map_type_str(m->type));
            break;
        }
    }

    if (num_maps > 8)
        pr_info("  ... (%u more entries not shown)\n", num_maps - 8);

    return 0; /* 0 = let the real function execute normally */
}

/* ------------------------------------------------------------------ */
/* kprobe struct — بنحدد فيها اسم الدالة والـ handler                 */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "pinctrl_register_mappings",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init pinctrl_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("pinctrl_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("pinctrl_probe: kprobe planted on %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit pinctrl_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("pinctrl_probe: kprobe removed\n");
}

module_init(pinctrl_probe_init);
module_exit(pinctrl_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية للـ module — `module_init`, `MODULE_LICENSE`, إلخ |
| `linux/kernel.h` | `pr_info` / `pr_err` للـ logging |
| `linux/kprobes.h` | تعريف `struct kprobe` ودوال `register_kprobe` / `unregister_kprobe` |
| `linux/pinctrl/machine.h` | تعريف `struct pinctrl_map` و`enum pinctrl_map_type` اللي بنستخدمها في الـ cast |

---

#### `map_type_str()`

دالة مساعدة بترجع اسم نصي للـ `enum pinctrl_map_type`. هي موجودة عشان الـ `pr_info` يطبع نص واضح بدل رقم خام.

---

#### `handler_pre()`

ده الـ **pre-handler** — الكرنل بيستدعيه مباشرةً قبل تنفيذ `pinctrl_register_mappings`. بيستخدم `regs_get_kernel_argument()` عشان يجيب الـ arguments بطريقة portable على x86-64 وARM64 من غير ما يعتمد على اسم register معين.

الـ handler بيطبع:
- عدد الـ maps المطلوب تسجيلها.
- لكل entry: اسم الـ device، اسم الـ state، نوع الـ mapping، واسم الـ group/function أو الـ pin/configs حسب النوع.

الـ return value `0` معناه "اتابع تنفيذ الدالة الأصلية بشكل طبيعي".

---

#### `struct kprobe kp`

الـ `symbol_name` بيقول للـ kprobe framework على أنهي رمز يحط الـ breakpoint. الـ `pre_handler` هو الـ callback اللي بيتشغل. مفيش `post_handler` هنا لأننا مش محتاجين نشوف الـ return value.

---

#### `module_init` — `pinctrl_probe_init()`

بيستدعي `register_kprobe()` واللي:
1. بتحول اسم الرمز لـ address.
2. بتحط breakpoint (int3 على x86، BRK على ARM64).
3. بتربط الـ callback.

لو فشل (مثلاً الدالة مش exported أو الـ kprobes مش مفعّلة) بيرجع error code فوراً.

---

#### `module_exit` — `pinctrl_probe_exit()`

**لازم** نعمل `unregister_kprobe()` في الـ exit عشان الكرنل يشيل الـ breakpoint ويحرر الـ memory المرتبطة. لو ما عملناش ده وحُطّ الـ module، الـ breakpoint هيفضل موجود وهيودي لـ kernel panic لما الكود بتاعنا يُستدعى بعد ما الـ module اتشال من الذاكرة.

---

### Makefile للتجربة

```makefile
obj-m += pinctrl_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### تشغيل وملاحظة النتائج

```bash
# تحميل الـ module
sudo insmod pinctrl_probe.ko

# شيل أي driver بيستخدم pinctrl عشان تشوف الـ log
# أو انتظر أي hotplug event — الـ pinctrl غالباً بيتسجل وقت boot

# تابع الـ log
sudo dmesg | grep pinctrl_probe

# إزالة الـ module بنظافة
sudo rmmod pinctrl_probe
```

**مثال متوقع من الـ dmesg:**

```
[ 12.345] pinctrl_probe: kprobe planted on pinctrl_register_mappings at ffffffffc0123456
[ 12.891] pinctrl_probe: registering 4 map(s)
[ 12.891]   [0] dev=soc:pinctrl              state=default      type=MUX_GROUP     grp=uart0_grp func=uart0
[ 12.891]   [1] dev=soc:pinctrl              state=default      type=MUX_GROUP     grp=i2c1_grp  func=i2c1
[ 12.891]   [2] dev=soc:pinctrl              state=default      type=CONFIGS_PIN   pin/grp=GPIO0 num_cfgs=2
[ 12.891]   [3] dev=soc:pinctrl              state=sleep        type=CONFIGS_GROUP pin/grp=uart0_grp num_cfgs=1
```
