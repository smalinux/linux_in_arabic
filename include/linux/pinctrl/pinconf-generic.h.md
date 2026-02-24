## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: PIN CONTROL SUBSYSTEM

**المشرف:** Linus Walleij `<linusw@kernel.org>`
**القائمة البريدية:** `linux-gpio@vger.kernel.org`
**المسار في الكود:** `include/linux/pinctrl/` و `drivers/pinctrl/`

---

### المشكلة اللي بيحلها الـ pinctrl بشكل عام

تخيّل إنك عندك لوحة إلكترونية فيها معالج (SoC) زي Raspberry Pi أو أي embedded board. الـ SoC ده فيه مئات الـ **pins** — الأرجل الفيزيائية اللي بتتوصل بيها بالعالم الخارجي.

المشكلة إن كل pin ممكن يشتغل بأكتر من طريقة:
- ممكن يكون **GPIO** عادي (input أو output)
- ممكن يكون جزء من **I2C** أو **SPI** أو **UART**
- ممكن يكون **PWM** أو **clock**

وده بيُسمى **pin multiplexing (pinmux)** — يعني نفس الرجل بتختار هو هيعمل إيه.

لكن حتى لو اخترت وظيفة الـ pin، لازم تحدد **كيف يتصرف كهربائيًا**:
- هل عليه **pull-up** (مقاومة بتشده لـ VDD)؟
- هل عليه **pull-down** (مقاومة بتشده للـ GND)؟
- هل يشتغل بـ **open-drain** ولا **push-pull**؟
- هل يقدر يوصّل **تيار كتير** ولا قليل (drive strength)؟
- هل فيه **debounce** على الإشارة؟

ده بالظبط اللي بيتكلم فيه الـ **pinconf** — الجزء اللي بيتحكم في **الخصائص الكهربائية** للـ pin.

---

### القصة الكاملة بالتفصيل

#### المشكلة قبل الـ pinctrl

في أيام Linux القديمة، كل driver كان بيتعامل مع الـ pins بطريقته الخاصة. Driver الـ I2C عنده كود لضبط الـ pull-up، driver الـ SPI عنده كود تاني، وكل hardware platform عنده كود مختلف تالت. النتيجة؟ **فوضى كاملة** وكود مكرر في كل مكان.

#### الحل: الـ pinctrl framework

جاء `pinctrl` كـ **framework موحّد** بيفصل بين:
1. **من يطلب؟** — الـ device driver (مثلاً driver الـ I2C)
2. **من ينفذ؟** — الـ pin controller driver اللي بيعرف تفاصيل الـ hardware

#### دور الـ pinconf-generic.h

لما الـ framework اتعمل، كل شركة كانت بتعمل **تعريفاتها الخاصة** لخصائص الـ pins. شركة بتسمي الـ pull-up بطريقة، وشركة تانية بطريقة مختلفة. فجاء `pinconf-generic.h` كـ **لغة مشتركة** — قاموس موحّد بيقول:

> "مهما كان الـ hardware اللي شغال عليه، استخدم الأسماء دي للـ pin configurations"

---

### ما بيعمله الملف ده تحديدًا

الملف `pinconf-generic.h` بيقدم **ثلاثة أشياء رئيسية**:

#### 1. الـ `enum pin_config_param` — قاموس المعاملات

ده enum بيعدد **كل الخصائص الممكنة** لأي pin في أي hardware:

```c
enum pin_config_param {
    PIN_CONFIG_BIAS_BUS_HOLD,       /* keep last value on tristate bus */
    PIN_CONFIG_BIAS_DISABLE,        /* disable all bias */
    PIN_CONFIG_BIAS_HIGH_IMPEDANCE, /* high-Z / tristate / floating */
    PIN_CONFIG_BIAS_PULL_DOWN,      /* pull to GND, arg = resistance in Ohms */
    PIN_CONFIG_BIAS_PULL_PIN_DEFAULT, /* hardware-defined default pull */
    PIN_CONFIG_BIAS_PULL_UP,        /* pull to VDD, arg = resistance in Ohms */
    PIN_CONFIG_DRIVE_OPEN_DRAIN,    /* open-drain output mode */
    PIN_CONFIG_DRIVE_OPEN_SOURCE,   /* open-source/emitter output mode */
    PIN_CONFIG_DRIVE_PUSH_PULL,     /* standard push-pull output */
    PIN_CONFIG_DRIVE_STRENGTH,      /* max current in mA */
    PIN_CONFIG_DRIVE_STRENGTH_UA,   /* max current in uA */
    PIN_CONFIG_INPUT_DEBOUNCE,      /* debounce time in usecs */
    PIN_CONFIG_INPUT_ENABLE,        /* enable/disable input buffer */
    PIN_CONFIG_INPUT_SCHMITT,       /* schmitt-trigger with custom threshold */
    PIN_CONFIG_INPUT_SCHMITT_ENABLE,/* enable/disable schmitt-trigger */
    PIN_CONFIG_INPUT_SCHMITT_UV,    /* schmitt-trigger threshold in uV */
    PIN_CONFIG_MODE_LOW_POWER,      /* low power mode */
    PIN_CONFIG_MODE_PWM,            /* PWM mode */
    PIN_CONFIG_LEVEL,               /* output level high(1) or low(0) */
    PIN_CONFIG_OUTPUT_ENABLE,       /* enable output buffer */
    PIN_CONFIG_OUTPUT_IMPEDANCE_OHMS, /* output impedance in ohms */
    PIN_CONFIG_PERSIST_STATE,       /* keep state across sleep/reset */
    PIN_CONFIG_POWER_SOURCE,        /* select power supply */
    PIN_CONFIG_SKEW_DELAY,          /* clock skew / latch delay */
    PIN_CONFIG_SKEW_DELAY_INPUT_PS, /* input skew in picoseconds */
    PIN_CONFIG_SKEW_DELAY_OUTPUT_PS,/* output latch delay in picoseconds */
    PIN_CONFIG_SLEEP_HARDWARE_STATE,/* sleep-related state marker */
    PIN_CONFIG_SLEW_RATE,           /* slew rate selection */
    PIN_CONFIG_END = 0x7F,          /* last standard config */
    PIN_CONFIG_MAX = 0xFF,          /* max value for packed format */
};
```

الـ `PIN_CONFIG_END` بيفضل مكان للـ hardware-specific configs من `0x80` لـ `0xFF`.

#### 2. الـ Packed Format — ضغط المعلومات في `unsigned long`

بدل ما تبعت struct كاملة لكل config، الـ framework بيضغط الـ **parameter + argument** في `unsigned long` واحد:

```
  bits 31..8        bits 7..0
┌──────────────────┬──────────┐
│   argument       │  param   │
│   (24 bits)      │ (8 bits) │
└──────────────────┴──────────┘
```

المـ macro اللي بيعمل ده:

```c
#define PIN_CONF_PACKED(p, a) ((a << 8) | ((unsigned long) p & 0xffUL))
```

وفيه inline functions للاستخراج:

```c
/* extract the parameter type from packed config */
static inline enum pin_config_param pinconf_to_config_param(unsigned long config)
{
    return (enum pin_config_param) (config & 0xffUL);
}

/* extract the argument value from packed config */
static inline u32 pinconf_to_config_argument(unsigned long config)
{
    return (u32) ((config >> 8) & 0xffffffUL);
}

/* pack param + argument into a single unsigned long */
static inline unsigned long pinconf_to_config_packed(enum pin_config_param param,
                                                      u32 argument)
{
    return PIN_CONF_PACKED(param, argument);
}
```

#### 3. الـ Device Tree Integration

الملف بيوفر functions بتحول الـ **Device Tree nodes** لـ `pinctrl_map` entries — يعني بتقرأ الـ DT وتترجمه لـ pin configurations:

```c
/* parse a DT subnode and build pinctrl map entries */
int pinconf_generic_dt_subnode_to_map(struct pinctrl_dev *pctldev,
        struct device_node *np, struct pinctrl_map **map,
        unsigned int *reserved_maps, unsigned int *num_maps,
        enum pinctrl_map_type type);

/* parse a full DT node to map (calls subnode version internally) */
int pinconf_generic_dt_node_to_map(struct pinctrl_dev *pctldev,
        struct device_node *np_config, struct pinctrl_map **map,
        unsigned int *num_maps, enum pinctrl_map_type type);

/* free the map allocated by the above */
void pinconf_generic_dt_free_map(struct pinctrl_dev *pctldev,
        struct pinctrl_map *map, unsigned int num_maps);
```

وفيه inline wrappers للاستخدام الشائع:

```c
/* for group-level configs */
pinconf_generic_dt_node_to_map_group(...)

/* for individual pin configs */
pinconf_generic_dt_node_to_map_pin(...)

/* auto-detect type from DT properties */
pinconf_generic_dt_node_to_map_all(...)
```

---

### مثال حياتي: ضبط I2C pins على Raspberry Pi

في الـ Device Tree بتكتب:

```dts
&i2c1 {
    pinctrl-names = "default";
    pinctrl-0 = <&i2c1_pins>;
};

i2c1_pins: i2c1 {
    brcm,pins = <2 3>;
    brcm,function = <4>;         /* alt0 = I2C */
    brcm,pull = <2>;             /* pull-up */
};
```

لما الكرنل بيشغّل I2C driver، الـ `pinconf_generic_dt_node_to_map` بتقرأ الـ DT node دي وبتبني list من الـ configs. بعدين الـ pinctrl core بتبعتها لـ pin controller driver اللي بيضبط الـ hardware فعلاً.

---

### مثال كمان: ضبط UART بـ pull-up في الكود مباشرة

```c
unsigned long configs[] = {
    PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 10000),  /* 10K pull-up */
    PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 8),    /* 8mA drive */
};
```

---

### الـ structs المساعدة

```c
/* used for debugfs display of a config parameter */
struct pin_config_item {
    const enum pin_config_param param;  /* which parameter */
    const char * const display;         /* human-readable name */
    const char * const format;          /* printf format for argument */
    bool has_arg;                       /* does it take an argument? */
    const char * const *values;         /* optional named values array */
    size_t num_values;
};

/* maps a DT property name to a pin_config_param */
struct pinconf_generic_params {
    const char * const property;        /* DT property name e.g. "bias-pull-up" */
    enum pin_config_param param;        /* corresponding enum value */
    u32 default_value;                  /* value if property has no argument */
    const char * const *values;         /* optional named values */
    size_t num_values;
};
```

---

### العلاقة بين ملفات الـ pinctrl subsystem

```
                    ┌─────────────────────────┐
                    │   Device Driver          │
                    │  (I2C, SPI, UART, etc.)  │
                    └────────────┬────────────┘
                                 │ uses
                    ┌────────────▼────────────┐
                    │   pinctrl consumer API   │
                    │  include/linux/pinctrl/  │
                    │     consumer.h           │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────▼──────────────────┐
              │          pinctrl CORE                │
              │       drivers/pinctrl/core.c         │
              │     drivers/pinctrl/pinconf.c        │
              │  drivers/pinctrl/pinconf-generic.c  ◄──── يستخدم pinconf-generic.h
              │     drivers/pinctrl/devicetree.c     │
              └──────────┬──────────────┬───────────┘
                         │              │
              ┌──────────▼──────┐  ┌────▼──────────────┐
              │  pinmux layer   │  │  pinconf layer     │
              │   pinmux.h      │  │  pinconf.h         │
              │                 │  │  pinconf-generic.h │◄── الملف ده
              └──────────┬──────┘  └────────────────────┘
                         │
              ┌──────────▼──────────────────────────────┐
              │     Hardware-specific drivers            │
              │  drivers/pinctrl/bcm/pinctrl-bcm2835.c  │
              │  drivers/pinctrl/intel/pinctrl-intel.c  │
              │  drivers/pinctrl/nomadik/               │
              │  ... (مئات الـ drivers لكل SoC)         │
              └─────────────────────────────────────────┘
```

---

### ملفات لازم تعرفها

| الملف | الدور |
|-------|-------|
| `include/linux/pinctrl/pinconf-generic.h` | **الملف ده** — قاموس الـ generic pin configs |
| `include/linux/pinctrl/pinconf.h` | تعريف `pinconf_ops` — الـ operations اللي lazem driver يـimplements |
| `include/linux/pinctrl/pinctrl.h` | تعريف `pinctrl_dev` و `pinctrl_ops` — الـ core structs |
| `include/linux/pinctrl/pinmux.h` | الـ pin multiplexing interface |
| `include/linux/pinctrl/machine.h` | الـ `pinctrl_map` و mapping macros |
| `include/linux/pinctrl/consumer.h` | الـ API اللي الـ device drivers بيستخدموها |
| `include/linux/pinctrl/devinfo.h` | ربط الـ pinctrl بالـ device struct |
| `drivers/pinctrl/core.c` | الـ pinctrl core implementation |
| `drivers/pinctrl/pinconf-generic.c` | الـ implementation للـ DT parsing functions |
| `drivers/pinctrl/devicetree.c` | الـ DT integration للـ pinmux |
| `Documentation/driver-api/pin-control.rst` | التوثيق الرسمي الكامل |
## Phase 2: شرح الـ Pinctrl / Pin Configuration Framework

---

### المشكلة — ليه الـ Subsystem ده موجود أصلاً؟

في أي SoC حديث زي Raspberry Pi أو STM32 أو i.MX8، في مئات الـ **pins** الجسمانية على الشريحة. كل pin ممكن يشتغل في أكتر من mode:

- I2C SDA
- SPI MOSI
- GPIO عادي
- UART TX

المشكلة مش بس في الـ **mux** (اختيار الـ function). المشكلة الأعمق إن كل pin ليه خصائص كهربية ممكن تتضبط:

| الخاصية | مثال عملي |
|---|---|
| **Pull-up / Pull-down** | I2C محتاج pull-up، UART مش محتاجه |
| **Drive strength** | SPI بـ 8mA، GPIO بـ 2mA |
| **Slew rate** | High-speed signals محتاجة slew rate عالي |
| **Open-drain** | I2C دايمًا open-drain |
| **Schmitt trigger** | Inputs بيضوضة كتير |
| **Debounce** | Buttons |
| **Input/Output impedance** | RF signals |

المشكلة الحقيقية: كل SoC vendor بيكتب الـ pin configuration بطريقة مختلفة تمامًا. من غير framework موحد، كل driver هيحتوي على كود خاص بالـ hardware، وده كسر مبدأ الـ **portability**.

---

### الحل — الـ Kernel بيعمل إيه؟

الـ **pinctrl subsystem** في الـ kernel بيقسم المسؤولية لطبقتين:

1. **Generic Layer** — واجهة موحدة بتعرف كل الـ parameters الممكنة بشكل abstract (ده اللي بنشوفه في `pinconf-generic.h`)
2. **Driver Layer** — كل SoC بيكتب driver خاص بيترجم الـ generic parameters لـ register writes حقيقية

الـ `pinconf-generic.h` بالذات بيحل مشكلة **توحيد الـ pin configuration vocabulary** — بدل ما كل driver يعرف "pull-up" بطريقته الخاصة، في `PIN_CONFIG_BIAS_PULL_UP` معياري يفهمه كل الكود.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    Consumer Drivers                              │
  │    (I2C driver, SPI driver, UART driver, GPIO driver, ...)       │
  │                                                                  │
  │    pinctrl_get() → pinctrl_lookup_state() → pinctrl_select()    │
  └───────────────────────────┬─────────────────────────────────────┘
                              │  Generic API
  ┌───────────────────────────▼─────────────────────────────────────┐
  │                  pinctrl core (kernel/pinctrl/)                  │
  │                                                                  │
  │  ┌─────────────────┐    ┌──────────────────────────────────┐    │
  │  │  pinmux core    │    │  pinconf core                    │    │
  │  │  (mux/function) │    │  (pin electrical configuration)  │    │
  │  └────────┬────────┘    └───────────┬──────────────────────┘    │
  │           │                         │                            │
  │           │   ┌─────────────────────┘                           │
  │           │   │  pinconf-generic layer                          │
  │           │   │  ┌────────────────────────────────────────┐    │
  │           │   │  │ enum pin_config_param                  │    │
  │           │   │  │ PIN_CONF_PACKED macro                  │    │
  │           │   │  │ pinconf_generic_dt_node_to_map()       │    │
  │           │   │  │ struct pinconf_generic_params          │    │
  │           │   │  └────────────────────────────────────────┘    │
  └───────────┼───┼───────────────────────────────────────────────-─┘
              │   │
  ┌───────────▼───▼────────────────────────────────────────────────┐
  │               SoC-specific pinctrl drivers                      │
  │                                                                  │
  │   ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
  │   │ pinctrl-bcm  │  │ pinctrl-imx  │  │ pinctrl-stm32        │ │
  │   │ (Broadcom)   │  │ (NXP i.MX)   │  │ (ST Microelectronics)│ │
  │   └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │
  └──────────┼─────────────────┼──────────────────────┼────────────┘
             │                 │                       │
  ┌──────────▼─────────────────▼──────────────────────▼────────────┐
  │                      Real Hardware                               │
  │         SoC Pin Controller Registers (IOMUXC, GPIOMUX, etc.)    │
  └─────────────────────────────────────────────────────────────────┘
```

---

### الـ Device Tree والـ Mapping Flow

الـ **Device Tree** هو المصدر الأول للـ pin configuration في النظم المدمجة. لما الـ kernel يبدأ:

```
DT node (pinctrl properties)
        │
        │  pinconf_generic_dt_node_to_map()
        ▼
struct pinctrl_map[]  ←── array of mapping entries
        │
        │  pinctrl_register_mappings()
        ▼
pinctrl core database
        │
        │  pinctrl_select_state()
        ▼
SoC driver .pin_config_set()
        │
        ▼
Hardware register write
```

---

### التشبيه الواقعي — مع Mapping كامل

تخيل إنك مهندس في مبنى شركة كبيرة، وعندك **لوحة توصيل مركزية (patch panel)** بمئات المنافذ. كل منفذ ممكن يتوصل بـ:
- تليفون
- شبكة إنترنت
- كاميرا
- نظام إنذار

**الـ mapping:**

| تشبيه | مفهوم Kernel |
|---|---|
| المبنى بأكمله | الـ SoC |
| لوحة التوصيل المركزية | الـ Pin Controller hardware |
| المنفذ الواحد في اللوحة | الـ Pin |
| توصيل المنفذ بـ تليفون أو شبكة | الـ Mux (pin function selection) |
| ضبط مقاومة الخط أو التردد المسموح | الـ Pin Configuration (pinconf) |
| دليل الـ standards الكهربية القياسية | `enum pin_config_param` |
| مهندس التوصيل المتخصص في كل مبنى | الـ SoC-specific pinctrl driver |
| نموذج الطلب الموحد للإدارة | الـ generic pinconf layer |
| ملف التوصيلات الموجود في الدرج | الـ Device Tree pinctrl node |

الجزء الأعمق في التشبيه: الـ **دليل القياسي** (= `enum pin_config_param`) موجود عشان مهندس في مبنى Broadcom ومهندس في مبنى NXP يملوا **نفس النموذج**، حتى لو طريقة تنفيذ الطلب جسمانيًا مختلفة تمامًا.

---

### الـ Core Abstraction — الفكرة المحورية

الـ abstraction الأساسية في `pinconf-generic.h` هي **packed unsigned long**:

```c
/*
 * Lower 8 bits  = pin_config_param  (WHAT to configure)
 * Upper 24 bits = argument          (HOW MUCH / which value)
 */
#define PIN_CONF_PACKED(p, a) ((a << 8) | ((unsigned long) p & 0xffUL))
```

```
  63        31        15    8 7       0
  ┌─────────────────────────┬─────────┐
  │   argument (24 bits)    │  param  │
  │   (value / strength)    │ (8 bits)│
  └─────────────────────────┴─────────┘
         e.g. 4 (mA)           e.g. PIN_CONFIG_DRIVE_STRENGTH (0x09)
```

يعني قيمة واحدة من نوع `unsigned long` بتحمل **كل المعلومات** اللازمة لـ pin configuration entry واحدة. الـ array من القيم دي هو اللي بتحمله `struct pinctrl_map_configs`.

**الـ helper functions:**

```c
/* استخراج الـ param من الـ packed value */
static inline enum pin_config_param pinconf_to_config_param(unsigned long config)
{
    return (enum pin_config_param)(config & 0xffUL);
}

/* استخراج الـ argument (القيمة الفعلية) */
static inline u32 pinconf_to_config_argument(unsigned long config)
{
    return (u32)((config >> 8) & 0xffffffUL);  /* 24 bits */
}

/* تجميع param + argument في قيمة واحدة */
static inline unsigned long pinconf_to_config_packed(enum pin_config_param param,
                                                      u32 argument)
{
    return PIN_CONF_PACKED(param, argument);
}
```

---

### الـ `enum pin_config_param` — التصنيف الكامل

الـ enum ده هو **قاموس الـ pinconf**. بنقسمه لـ categories:

#### Bias (مقاومة الـ pin للـ floating)

```
PIN_CONFIG_BIAS_DISABLE          → بلا bias نهائيًا
PIN_CONFIG_BIAS_HIGH_IMPEDANCE   → Hi-Z / tristate (disconnected)
PIN_CONFIG_BIAS_BUS_HOLD         → bus keeper (يحتفظ بآخر قيمة)
PIN_CONFIG_BIAS_PULL_DOWN        → يسحب لـ GND (arg = resistance بـ Ω)
PIN_CONFIG_BIAS_PULL_UP          → يسحب لـ VDD (arg = resistance بـ Ω)
PIN_CONFIG_BIAS_PULL_PIN_DEFAULT → الـ hardware نفسه يقرر
```

**مثال عملي:** I2C محتاج `PIN_CONFIG_BIAS_PULL_UP` على الـ SDA وSCL. لو نسيت ده، الـ bus مش هيشتغل.

#### Drive Mode (طريقة drive الـ output)

```
PIN_CONFIG_DRIVE_PUSH_PULL    → active high + active low (الأكثر شيوعًا)
PIN_CONFIG_DRIVE_OPEN_DRAIN   → active low فقط، high = Hi-Z (I2C, 1-wire)
PIN_CONFIG_DRIVE_OPEN_SOURCE  → active high فقط، low = Hi-Z (نادر)
PIN_CONFIG_DRIVE_STRENGTH     → التيار الأقصى بـ mA
PIN_CONFIG_DRIVE_STRENGTH_UA  → التيار الأقصى بـ µA (دقة أعلى)
```

**Open-drain بالتفصيل:**

```
VDD
 │
 R (external pull-up)
 │
 ├──── SDA line ────── SDA (device B)
 │
[drain] ← NMOS transistor
 │
GND

لما الـ output = 1: transistor off → line pulled up by R → HIGH
لما الـ output = 0: transistor on  → line pulled to GND → LOW
```

ده بيخلي أكتر من device تتكلم على نفس الـ wire بدون short circuit.

#### Input Configuration

```
PIN_CONFIG_INPUT_ENABLE          → تفعيل الـ input buffer
PIN_CONFIG_INPUT_DEBOUNCE        → debounce time بـ µs (للـ buttons)
PIN_CONFIG_INPUT_SCHMITT         → Schmitt trigger (custom threshold)
PIN_CONFIG_INPUT_SCHMITT_ENABLE  → تفعيل/تعطيل Schmitt trigger
PIN_CONFIG_INPUT_SCHMITT_UV      → Schmitt hysteresis بـ µV
```

**Schmitt Trigger — ليه مهم؟**

```
بدون Schmitt trigger:
  signal ─────/\/\/\─────    ← noise يسبب multiple transitions
             ↑↑↑↑↑↑
          false edges!

مع Schmitt trigger:
  VH ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ (upper threshold)
  VL ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ (lower threshold)

  signal ─────────────────    ← يقطع بس لما يعدي الـ threshold
                              ← hysteresis = VH - VL
```

#### Output Configuration

```
PIN_CONFIG_OUTPUT_ENABLE          → تفعيل output buffer (بدون تحديد قيمة)
PIN_CONFIG_OUTPUT_IMPEDANCE_OHMS  → impedance بـ Ω
PIN_CONFIG_LEVEL                  → قيمة الـ pin (0 أو 1)
```

#### Timing & Power

```
PIN_CONFIG_SLEW_RATE              → معدل تغير الإشارة (custom format)
PIN_CONFIG_SKEW_DELAY             → تأخير الـ clock (custom)
PIN_CONFIG_SKEW_DELAY_INPUT_PS    → input skew بـ picoseconds
PIN_CONFIG_SKEW_DELAY_OUTPUT_PS   → output latch delay بـ picoseconds
PIN_CONFIG_MODE_LOW_POWER         → low power mode
PIN_CONFIG_MODE_PWM               → PWM mode
PIN_CONFIG_POWER_SOURCE           → اختيار مصدر الطاقة
PIN_CONFIG_PERSIST_STATE          → الحفاظ على الـ config عبر sleep/reset
PIN_CONFIG_SLEEP_HARDWARE_STATE   → تعليم الـ config كـ sleep state
```

**الـ boundaries الخاصة:**

```c
PIN_CONFIG_END = 0x7F,  /* آخر standard param, 7 bits */
PIN_CONFIG_MAX = 0xFF,  /* أقصى قيمة في الـ 8-bit field */
/* القيم من 0x80 → 0xFF محجوزة للـ vendor-specific params */
```

---

### الـ Structs الأساسية

#### `struct pin_config_item` — للـ Debugfs

```c
struct pin_config_item {
    const enum pin_config_param param;   /* أي parameter */
    const char * const display;          /* اسم للعرض */
    const char * const format;           /* format string للـ value */
    bool has_arg;                        /* هل ليه argument؟ */
    const char * const *values;          /* أسماء القيم (optional) */
    size_t num_values;                   /* عدد القيم */
};
```

ده بيُستخدم في `/sys/kernel/debug/pinctrl/` عشان تعرض الـ pin configuration بشكل human-readable.

**مثال:**

```c
/* من drivers/pinctrl/pinconf-generic.c */
static const struct pin_config_item conf_items[] = {
    PCONFDUMP(PIN_CONFIG_BIAS_PULL_UP,   "input bias pull up",   "ohms", true),
    PCONFDUMP(PIN_CONFIG_DRIVE_STRENGTH, "output drive strength","mA",   true),
    /* ... */
};
```

#### `struct pinconf_generic_params` — ترجمة DT Properties

```c
struct pinconf_generic_params {
    const char * const property;    /* اسم الـ DT property */
    enum pin_config_param param;    /* الـ enum المقابل */
    u32 default_value;              /* القيمة الافتراضية */
    const char * const *values;     /* optional: أسماء القيم */
    size_t num_values;
};
```

ده هو الـ lookup table اللي بيترجم من DT string لـ kernel enum:

```
DT: "bias-pull-up"           →  PIN_CONFIG_BIAS_PULL_UP,    default=1
DT: "drive-strength = <8>"   →  PIN_CONFIG_DRIVE_STRENGTH,  default=0
DT: "input-schmitt-enable"   →  PIN_CONFIG_INPUT_SCHMITT_ENABLE, default=1
```

---

### الـ DT Parsing Functions — الـ Generic Layer عملياً

```
                   Device Tree
                       │
            ┌──────────┴──────────┐
            │  pinctrl node       │
            │  ┌──────────────┐   │
            │  │ state "default"│  │
            │  │  ┌──────────┐ │  │
            │  │  │ pin group│ │  │
            │  │  │ pins = ..│ │  │
            │  │  │ bias-... │ │  │
            │  │  └──────────┘ │  │
            │  └──────────────┘   │
            └──────────┬──────────┘
                       │
      pinconf_generic_dt_node_to_map()
                       │
            ┌──────────▼──────────┐
            │  struct pinctrl_map │  × N entries
            │  ┌────────────────┐ │
            │  │ dev_name       │ │
            │  │ state_name     │ │
            │  │ type = CONFIGS │ │
            │  │ configs[]      │ │  ← packed unsigned long[]
            │  └────────────────┘ │
            └─────────────────────┘
```

**الثلاث دوال:**

```c
/* تحويل DT subnode واحد → map entries */
int pinconf_generic_dt_subnode_to_map(struct pinctrl_dev *pctldev,
    struct device_node *np,
    struct pinctrl_map **map,
    unsigned int *reserved_maps,
    unsigned int *num_maps,
    enum pinctrl_map_type type);  /* PIN أو GROUP */

/* تحويل كل الـ DT config node */
int pinconf_generic_dt_node_to_map(struct pinctrl_dev *pctldev,
    struct device_node *np_config,
    struct pinctrl_map **map,
    unsigned int *num_maps,
    enum pinctrl_map_type type);

/* تحرير الـ memory بعد الاستخدام */
void pinconf_generic_dt_free_map(struct pinctrl_dev *pctldev,
    struct pinctrl_map *map,
    unsigned int num_maps);
```

**الـ convenience inlines** بتختصر الـ type argument:

```c
/* لـ SoC drivers اللي بتدعم configs على group level فقط */
pinconf_generic_dt_node_to_map_group(pctldev, np, map, num_maps);
/* ↳ يستدعي pinconf_generic_dt_node_to_map(..., PIN_MAP_TYPE_CONFIGS_GROUP) */

/* لـ SoC drivers اللي بتدعم configs على pin level فقط */
pinconf_generic_dt_node_to_map_pin(pctldev, np, map, num_maps);
/* ↳ يستدعي pinconf_generic_dt_node_to_map(..., PIN_MAP_TYPE_CONFIGS_PIN) */

/* لـ SoC drivers اللي بتحدد النوع تلقائيًا من الـ DT */
pinconf_generic_dt_node_to_map_all(pctldev, np, map, num_maps);
/* ↳ يستدعي pinconf_generic_dt_node_to_map(..., PIN_MAP_TYPE_INVALID) */
/*   → الـ parser يستنتج النوع من الـ DT properties */
```

---

### الـ `struct pinctrl_map` — ربط كل حاجة ببعض

ده التعريف من `machine.h`:

```c
struct pinctrl_map {
    const char *dev_name;        /* اسم الـ consumer device */
    const char *name;            /* اسم الـ state (e.g. "default", "sleep") */
    enum pinctrl_map_type type;  /* MUX_GROUP أو CONFIGS_PIN أو CONFIGS_GROUP */
    const char *ctrl_dev_name;   /* اسم الـ pinctrl controller */
    union {
        struct pinctrl_map_mux mux;         /* لو type = MUX_GROUP */
        struct pinctrl_map_configs configs;  /* لو type = CONFIGS_* */
    } data;
};
```

```
struct pinctrl_map relationship:

┌──────────────────────────────────────────────────────┐
│ struct pinctrl_map                                   │
│                                                      │
│  dev_name ────────────► "spi0"  (consumer)           │
│  name     ────────────► "default" (state)            │
│  type     ────────────► PIN_MAP_TYPE_CONFIGS_PIN     │
│  ctrl_dev ────────────► "pinctrl.0" (controller)    │
│                                                      │
│  data.configs:                                       │
│  ┌─────────────────────────────────────────────┐    │
│  │ group_or_pin ──► "SPI0_MOSI"                │    │
│  │ configs[]    ──► [packed_config_1,           │    │
│  │                   packed_config_2, ...]      │    │
│  │ num_configs  ──► 2                           │    │
│  └─────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

---

### مثال واقعي كامل — I2C على i.MX8

**الـ Device Tree:**

```dts
/* في ملف الـ board DTS */
&iomuxc {
    pinctrl_i2c1: i2c1grp {
        fsl,pins = <
            MX8MM_IOMUXC_I2C1_SCL_I2C1_SCL    0x400001c3
            MX8MM_IOMUXC_I2C1_SDA_I2C1_SDA    0x400001c3
        >;
    };
};

&i2c1 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c1>;
    status = "okay";
};
```

**الـ Flow داخل الـ Kernel:**

```
1. OF (Open Firmware) parser يقرأ الـ DT

2. i.MX pinctrl driver يستدعي:
   pinconf_generic_dt_node_to_map_group(pctldev, np, &map, &num_maps)

3. الـ generic layer يحول:
   "fsl,pins" → array of struct pinctrl_map

4. لما i2c1 driver يتعمل probe:
   pinctrl_get(dev) → pinctrl_lookup_state(p, "default")
   → pinctrl_select_state(p, state)

5. الـ pinctrl core يستدعي:
   imx_pinconf_set(pctldev, pin_id, configs, num_configs)

6. الـ i.MX driver يكتب في:
   IOMUXC_SW_PAD_CTL_PAD_I2C1_SCL register
```

---

### ملكية المسؤوليات — ماذا يملك الـ Generic Layer وماذا يفوّض؟

| المسؤولية | Generic Layer | SoC Driver |
|---|---|---|
| تعريف **vocabulary** الـ config parameters | ✓ | - |
| الـ **packing format** (8-bit param + 24-bit arg) | ✓ | - |
| قراءة وتحويل **Device Tree** properties | ✓ | - |
| **Debugfs** display names وformats | ✓ | - |
| ترجمة params لـ **register values** | - | ✓ |
| معرفة **أي registers** للـ pin configuration | - | ✓ |
| دعم **vendor-specific params** (0x80–0xFF) | - | ✓ |
| **Validation** إن الـ param مدعوم على هذا الـ SoC | - | ✓ |
| إدارة الـ **clock gating** للـ pin controller | - | ✓ |

---

### الـ Subsystems الأخرى المرتبطة

- **GPIO subsystem** — بيستخدم الـ pinctrl لضبط الـ pin قبل استخدامه كـ GPIO. لازم تفهم إن pinctrl وGPIO منفصلين مفاهيميًا حتى لو نفس الـ hardware.
- **Device Tree / OF (Open Firmware)** — المصدر الأساسي للـ pin configuration data في الأنظمة المدمجة الحديثة.
- **Device core (driver model)** — الـ `pinctrl_select_state()` بيُستدعى تلقائيًا من `__device_driver_probe()` لما driver يعمل probe على device.
- **Regmap subsystem** — بعض الـ pin controllers بتستخدم regmap للوصول للـ registers (MMIO أو I2C أو SPI-controlled pin expanders).
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Config Options — Cheatsheet

#### `enum pin_config_param` — كل القيم الممكنة

| القيمة | الـ Value | الوحدة / الـ Argument | الاستخدام |
|---|---|---|---|
| `PIN_CONFIG_BIAS_BUS_HOLD` | 0 | ignored | يحافظ على آخر قيمة على الـ tristate bus |
| `PIN_CONFIG_BIAS_DISABLE` | 1 | ignored | يوقف أي bias |
| `PIN_CONFIG_BIAS_HIGH_IMPEDANCE` | 2 | ignored | يحول الـ pin لـ high-Z / floating |
| `PIN_CONFIG_BIAS_PULL_DOWN` | 3 | Ohms (أو custom) | pull-down — يفعّله لو القيمة != 0 |
| `PIN_CONFIG_BIAS_PULL_PIN_DEFAULT` | 4 | 0/1 | pull حسب الـ hardware default |
| `PIN_CONFIG_BIAS_PULL_UP` | 5 | Ohms (أو custom) | pull-up — يفعّله لو القيمة != 0 |
| `PIN_CONFIG_DRIVE_OPEN_DRAIN` | 6 | ignored | open drain / open collector |
| `PIN_CONFIG_DRIVE_OPEN_SOURCE` | 7 | ignored | open emitter |
| `PIN_CONFIG_DRIVE_PUSH_PULL` | 8 | ignored | push-pull — الوضع الافتراضي |
| `PIN_CONFIG_DRIVE_STRENGTH` | 9 | mA | قدرة الـ drive بالـ milliampere |
| `PIN_CONFIG_DRIVE_STRENGTH_UA` | 10 | µA | قدرة الـ drive بالـ microampere |
| `PIN_CONFIG_INPUT_DEBOUNCE` | 11 | µsec | وقت الـ debounce — 0 يوقفه |
| `PIN_CONFIG_INPUT_ENABLE` | 12 | 0/1 | يفعّل/يوقف الـ input buffer |
| `PIN_CONFIG_INPUT_SCHMITT` | 13 | custom threshold | schmitt-trigger mode بـ threshold |
| `PIN_CONFIG_INPUT_SCHMITT_ENABLE` | 14 | 0/1 | يفعّل/يوقف schmitt-trigger |
| `PIN_CONFIG_INPUT_SCHMITT_UV` | 15 | µV | schmitt-trigger threshold بالـ microvolt |
| `PIN_CONFIG_MODE_LOW_POWER` | 16 | 0/custom | وضع توفير الطاقة |
| `PIN_CONFIG_MODE_PWM` | 17 | custom | يضبط الـ pin لـ PWM |
| `PIN_CONFIG_LEVEL` | 18 | 0/1 | يقرأ أو يكتب مستوى الـ output (low/high) |
| `PIN_CONFIG_OUTPUT_ENABLE` | 19 | 0/1 | يفعّل الـ output buffer بدون drive |
| `PIN_CONFIG_OUTPUT_IMPEDANCE_OHMS` | 20 | Ohms | impedance الـ output |
| `PIN_CONFIG_PERSIST_STATE` | 21 | ignored | يحافظ على الحالة بعد الـ sleep أو reset |
| `PIN_CONFIG_POWER_SOURCE` | 22 | custom | يختار مصدر الطاقة |
| `PIN_CONFIG_SKEW_DELAY` | 23 | custom | skew rate أو latch delay |
| `PIN_CONFIG_SKEW_DELAY_INPUT_PS` | 24 | ps | skew delay للـ input بالـ picosecond |
| `PIN_CONFIG_SKEW_DELAY_OUTPUT_PS` | 25 | ps | latch delay للـ output بالـ picosecond |
| `PIN_CONFIG_SLEEP_HARDWARE_STATE` | 26 | ignored | إشارة إن الحالة دي sleep-related |
| `PIN_CONFIG_SLEW_RATE` | 27 | custom | slew rate بين الـ 0 و 1 |
| `PIN_CONFIG_END` | 0x7F | — | آخر قيمة standard |
| `PIN_CONFIG_MAX` | 0xFF | — | أقصى قيمة في الـ packed format |

> **ملحوظة:** القيم من 0x80 لـ 0xFE متاحة للـ vendor-specific configs — الـ driver يعرّفها من `PIN_CONFIG_END + 1`.

---

#### `enum pinctrl_map_type` — أنواع الـ mapping

| القيمة | المعنى |
|---|---|
| `PIN_MAP_TYPE_INVALID` | قيمة غير صالحة — يُستخدم كـ sentinel أو لتفعيل الـ auto-detect من DT |
| `PIN_MAP_TYPE_DUMMY_STATE` | حالة وهمية — الـ driver مش موجود |
| `PIN_MAP_TYPE_MUX_GROUP` | تغيير الـ mux function لمجموعة pins |
| `PIN_MAP_TYPE_CONFIGS_PIN` | ضبط configuration لـ pin واحد |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | ضبط configuration لمجموعة pins |

---

### الـ Packed Config Format — Cheatsheet

```
unsigned long config:
 ┌─────────────────────────────┬────────────────┐
 │  bits [31:8] — argument     │ bits [7:0] — param │
 │  (24 bits, max 0xFFFFFF)    │ (8 bits, max 0xFF) │
 └─────────────────────────────┴────────────────┘
```

| الـ Macro / Function | الوظيفة |
|---|---|
| `PIN_CONF_PACKED(p, a)` | يبني قيمة packed من param و argument |
| `pinconf_to_config_param(config)` | يستخرج الـ param من الـ config |
| `pinconf_to_config_argument(config)` | يستخرج الـ argument من الـ config |
| `pinconf_to_config_packed(param, arg)` | نفس الـ macro بس كـ inline function |

---

### الـ Structs الأساسية

---

#### `struct pin_config_item`

**الغرض:** وصف metadata لكل `pin_config_param` لأغراض الـ debugfs وطباعة الـ configuration — يُستخدم في جداول الـ `pinconf_generic_dump_*`.

```c
struct pin_config_item {
    const enum pin_config_param param;   /* الـ param اللي بيصفه */
    const char * const display;          /* اسم بشري للطباعة */
    const char * const format;           /* صيغة الـ argument مثلاً "mA" أو "Ohm" */
    bool has_arg;                        /* هل الـ param ليه argument ذو معنى؟ */
    const char * const *values;          /* مصفوفة من الأسماء النصية للقيم */
    size_t num_values;                   /* عدد العناصر في values */
};
```

| الـ Field | النوع | الشرح |
|---|---|---|
| `param` | `enum pin_config_param` | الـ enum value اللي بيمثله الـ item ده |
| `display` | `const char *` | اسم مقروء زي `"bias-pull-up"` |
| `format` | `const char *` | وحدة الـ argument مثلاً `"mA"` — `NULL` لو مفيش |
| `has_arg` | `bool` | لو `false` الـ argument يُتجاهل في الطباعة |
| `values` | `const char **` | مصفوفة أسماء نصية للقيم المتقدرة — `NULL` لو مفيش |
| `num_values` | `size_t` | حجم مصفوفة `values` |

**الـ Macros المساعدة:**

```c
/* بدون values — الحالة العادية */
PCONFDUMP(param, display, format, has_arg)

/* مع values — زي enum أو named states */
PCONFDUMP_WITH_VALUES(param, display, format, has_arg, values, num_values)
```

**مثال عملي:**
```c
/* في pinconf-generic.c */
static const struct pin_config_item conf_items[] = {
    PCONFDUMP(PIN_CONFIG_BIAS_PULL_UP, "input bias pull up", "ohms", true),
    PCONFDUMP(PIN_CONFIG_DRIVE_STRENGTH, "output drive strength", "mA", true),
    PCONFDUMP(PIN_CONFIG_BIAS_DISABLE, "input bias disabled", NULL, false),
};
```

---

#### `struct pinconf_generic_params`

**الغرض:** ربط اسم الـ DT property بالـ `pin_config_param` المقابل — يُستخدم في دالة `pinconf_generic_dt_subnode_to_map` لتحويل الـ Device Tree nodes إلى `pinctrl_map` entries.

```c
struct pinconf_generic_params {
    const char * const property;    /* اسم الـ DT property زي "bias-pull-up" */
    enum pin_config_param param;    /* الـ param المقابل */
    u32 default_value;              /* القيمة الافتراضية لو ما في argument في الـ DT */
    const char * const *values;     /* أسماء القيم المتقدرة لو الـ property enum */
    size_t num_values;              /* حجم مصفوفة values */
};
```

| الـ Field | النوع | الشرح |
|---|---|---|
| `property` | `const char *` | اسم الـ DT property المطابق |
| `param` | `enum pin_config_param` | الـ param الناتج |
| `default_value` | `u32` | القيمة لو الـ property موجود بدون argument |
| `values` | `const char **` | أسماء نصية للقيم المتقدرة — لتحليل الـ DT strings |
| `num_values` | `size_t` | حجم مصفوفة `values` |

**مثال — الجدول الافتراضي في `pinconf-generic.c`:**
```c
static const struct pinconf_generic_params dt_params[] = {
    { "bias-bus-hold",       PIN_CONFIG_BIAS_BUS_HOLD,      0 },
    { "bias-disable",        PIN_CONFIG_BIAS_DISABLE,       0 },
    { "bias-pull-down",      PIN_CONFIG_BIAS_PULL_DOWN,     1 },
    { "bias-pull-up",        PIN_CONFIG_BIAS_PULL_UP,       1 },
    { "drive-strength",      PIN_CONFIG_DRIVE_STRENGTH,     0 },
    { "input-debounce",      PIN_CONFIG_INPUT_DEBOUNCE,     0 },
    /* ... */
};
```

---

#### `struct pinctrl_map` (من `machine.h`)

**الغرض:** الـ mapping entry الأساسي — يربط device بـ pinctrl state بـ configuration. البورد أو الـ DT بيولد مصفوفة من هذه الـ structs وبيسجلها في الـ pinctrl core.

```c
struct pinctrl_map {
    const char *dev_name;       /* اسم الـ device اللي بيستخدم الـ map */
    const char *name;           /* اسم الـ state مثلاً "default" أو "sleep" */
    enum pinctrl_map_type type; /* نوع الـ mapping */
    const char *ctrl_dev_name;  /* اسم الـ pin controller */
    union {
        struct pinctrl_map_mux mux;         /* لو type == MUX_GROUP */
        struct pinctrl_map_configs configs; /* لو type == CONFIGS_* */
    } data;
};
```

| الـ Field | النوع | الشرح |
|---|---|---|
| `dev_name` | `const char *` | اسم الـ device صاحب الـ request |
| `name` | `const char *` | اسم الـ pinctrl state |
| `type` | `enum pinctrl_map_type` | نوع الـ map entry |
| `ctrl_dev_name` | `const char *` | اسم الـ pinctrl controller |
| `data.mux` | `pinctrl_map_mux` | اسم الـ group والـ function |
| `data.configs` | `pinctrl_map_configs` | اسم الـ pin/group ومصفوفة الـ configs |

---

#### `struct pinctrl_map_mux` (من `machine.h`)

**الغرض:** تحديد الـ mux function لمجموعة pins.

```c
struct pinctrl_map_mux {
    const char *group;    /* اسم الـ pin group — NULL يعني أول group متاح */
    const char *function; /* اسم الـ function المطلوب */
};
```

---

#### `struct pinctrl_map_configs` (من `machine.h`)

**الغرض:** تحديد الـ configuration parameters لـ pin أو group.

```c
struct pinctrl_map_configs {
    const char *group_or_pin;   /* اسم الـ pin أو الـ group */
    unsigned long *configs;     /* مصفوفة packed config values */
    unsigned int num_configs;   /* عدد الـ configs */
};
```

كل عنصر في مصفوفة `configs` هو `unsigned long` فيه param في الـ bits السفلية وargument في الـ bits العلوية — اللي أنتجه `PIN_CONF_PACKED`.

---

### رسم العلاقات بين الـ Structs

```
pinconf_generic_params[]          pin_config_item[]
(DT property → param table)      (debugfs display table)
        │                                 │
        │ يستخدم                           │ يستخدم
        ▼                                 ▼
enum pin_config_param ◄───────────────────┘
        │
        │ يُعبّأ في
        ▼
unsigned long (packed config)
  [bits 31:8 = argument | bits 7:0 = param]
        │
        │ يُخزّن في
        ▼
pinctrl_map_configs.configs[]
        │
        │ جزء من
        ▼
pinctrl_map.data.configs
        │
        │ أو
        ▼
pinctrl_map.data.mux  ◄─── pinctrl_map_mux
        │
        │ نوعه
        ▼
enum pinctrl_map_type
        │
        │ كلها تُسجّل في
        ▼
pinctrl_dev (core — مش معرّف هنا)
```

---

### رسم دورة حياة الـ Pin Configuration من الـ DT

```
[Device Tree]
  └── pinctrl node
        └── subnode (مثلاً: uart0-pins)
              ├── pins = "GPIO_0", "GPIO_1"
              ├── bias-pull-up
              └── drive-strength = <4>

          ▼  pinconf_generic_dt_node_to_map()

[Kernel — map allocation]
  └── pinctrl_map[] array (مخصّص dynamically)
        ├── entry[0]: type=CONFIGS_PIN, pin="GPIO_0"
        │              configs=[pull_up_packed, drive_packed]
        └── entry[1]: type=CONFIGS_PIN, pin="GPIO_1"
                       configs=[pull_up_packed, drive_packed]

          ▼  pinctrl_register_mappings()

[pinctrl core — internal map list]
  └── يخزّن الـ maps مربوطة بالـ device

          ▼  pinctrl_get() + pinctrl_lookup_state()

[Driver applies state]
  └── pinctrl_select_state("default")
        └── pinconf_apply_setting()
              └── ops->pin_config_set(pctldev, pin, configs, num)
                    └── [Hardware Register Write]

          ▼  device_remove() أو pm_runtime_suspend()

[Teardown]
  └── pinconf_generic_dt_free_map()
        └── kfree(map entries)
```

---

### رسم تدفق الـ DT Parsing — `pinconf_generic_dt_subnode_to_map`

```
pinconf_generic_dt_subnode_to_map(pctldev, np, map, reserved, num, type)
  │
  ├─► قراءة "pins" أو "groups" من الـ DT node
  │     └── يحدد قائمة الـ pins أو الـ groups المستهدفة
  │
  ├─► للـ MUX: قراءة "function" property
  │     └── ينشئ CONFIGS_PIN أو MUX_GROUP entry
  │
  ├─► للـ CONFIGS: يمشي على dt_params[]
  │     └── لكل property موجود في الـ DT:
  │           ├── يقرأ القيمة (أو يستخدم default_value)
  │           └── ينشئ packed config: PIN_CONF_PACKED(param, value)
  │
  ├─► يحجز pinctrl_map entries جديدة (أو يوسّع الـ array)
  │     └── pinctrl_utils_reserve_map()
  │
  └─► يملأ الـ map entries ويزوّد num_maps
```

---

### رسم تدفق الاستدعاء — الـ Helper Functions

```
Driver (مثلاً: pinctrl-bcm2835.c)
  │
  └── .dt_node_to_map = pinconf_generic_dt_node_to_map_all
                            │
                            ▼
              pinconf_generic_dt_node_to_map(type=INVALID)
                            │
                            ├── للـ child node واحد:
                            │     pinconf_generic_dt_subnode_to_map()
                            │           │
                            │           ├── pinctrl_utils_reserve_map()
                            │           ├── pinctrl_utils_add_map_mux()
                            │           └── pinctrl_utils_add_map_configs()
                            │
                            └── يكرر على كل child nodes

──────────────────────────────────────────────────────

  Driver (pinctrl-single.c — configs only)
  │
  └── .dt_node_to_map = pinconf_generic_dt_node_to_map_pin
                            │
                            ▼
              pinconf_generic_dt_node_to_map(type=CONFIGS_PIN)
                            │ يجبر النوع على PIN
                            └── pinconf_generic_dt_subnode_to_map()

──────────────────────────────────────────────────────

  Driver (pinctrl-tegra.c — group configs)
  │
  └── .dt_node_to_map = pinconf_generic_dt_node_to_map_group
                            │
                            ▼
              pinconf_generic_dt_node_to_map(type=CONFIGS_GROUP)
                            │ يجبر النوع على GROUP
                            └── pinconf_generic_dt_subnode_to_map()
```

---

### رسم الـ Packed Config Encoding / Decoding

```
Encoding:
  param  = PIN_CONFIG_DRIVE_STRENGTH  (= 9 = 0x09)
  argument = 8  (mA)

  PIN_CONF_PACKED(9, 8):
    = (8 << 8) | (9 & 0xFF)
    = 0x00000800 | 0x00000009
    = 0x00000809

  ┌──────────┬──────────┬──────────┬──────────┐
  │  byte 3  │  byte 2  │  byte 1  │  byte 0  │
  │   0x00   │   0x00   │   0x08   │   0x09   │
  └──────────┴──────────┴──────────┴──────────┘
                                   └── param (8 bits)
                         └─────────── argument bits 8-15
               └─────────────────── argument bits 16-23
  └─────────────────────────────── argument bits 24-31 (unused)

Decoding:
  pinconf_to_config_param(0x809)   → 0x09 = PIN_CONFIG_DRIVE_STRENGTH
  pinconf_to_config_argument(0x809) → (0x809 >> 8) & 0xFFFFFF = 8 mA
```

---

### استراتيجية الـ Locking

الـ `pinconf-generic.h` نفسه **ما يعرّفش أي locks** — إنما بيوفر infrastructure stateless. الـ locking بيحصل في طبقة أعلى:

| الـ Lock | الموقع | بيحمي إيه |
|---|---|---|
| `pinctrl_dev->mutex` | `pinctrl-core.c` | الـ map list وكل عمليات register/unregister |
| `pinctrl_maps_mutex` | `pinctrl-core.c` | الـ global map list (`pinctrl_maps`) |
| `pinctrl->mutex` | `pinctrl-core.c` | الـ state selection لكل device |
| لا يوجد | `pinconf-generic.h` | الـ config items و params tables هي read-only بعد الـ init |

**ترتيب الـ locking (lock ordering):**
```
pinctrl_maps_mutex
    └── pinctrl_dev->mutex
            └── pinctrl->mutex
```

الـ `pinconf_generic_dt_*` functions بتُستدعى من سياق الـ `.dt_node_to_map` ops اللي بتكون محمية بـ `pinctrl_dev->mutex` في الـ core — الـ driver نفسه ما محتاجش يعمل lock إضافي.

الـ config tables (`pinconf_generic_params[]` و `pin_config_item[]`) هي **static read-only** بعد الـ init فـ thread-safe بطبيعتها.
## Phase 4: شرح الـ Functions

### ملخص — Cheatsheet

#### الـ Inline Helpers (Packing/Unpacking)

| Function / Macro | الغرض | الـ Return |
|---|---|---|
| `PIN_CONF_PACKED(p, a)` | يعمل pack لـ param + argument في `unsigned long` واحد | `unsigned long` |
| `pinconf_to_config_param(config)` | يستخرج الـ param (lower 8 bits) من الـ packed config | `enum pin_config_param` |
| `pinconf_to_config_argument(config)` | يستخرج الـ argument (upper 24 bits) من الـ packed config | `u32` |
| `pinconf_to_config_packed(param, argument)` | wrapper فوق `PIN_CONF_PACKED` | `unsigned long` |

#### الـ DT Parsing Functions

| Function | الغرض | الـ Return |
|---|---|---|
| `pinconf_generic_dt_subnode_to_map()` | يحول subnode واحد من DT لـ `pinctrl_map` entries | `int` (0 or -errno) |
| `pinconf_generic_dt_node_to_map()` | يحول node كامل (يتضمن subnodes) لـ map entries | `int` (0 or -errno) |
| `pinconf_generic_dt_free_map()` | يحرر الـ map entries المخصصة | `void` |
| `pinconf_generic_dt_node_to_map_group()` | wrapper — يفرض `PIN_MAP_TYPE_CONFIGS_GROUP` | `int` (0 or -errno) |
| `pinconf_generic_dt_node_to_map_pin()` | wrapper — يفرض `PIN_MAP_TYPE_CONFIGS_PIN` | `int` (0 or -errno) |
| `pinconf_generic_dt_node_to_map_all()` | wrapper — يخلي الـ parser يقرر النوع تلقائياً | `int` (0 or -errno) |

---

### Group 1: الـ Config Encoding / Decoding Helpers

الـ pinctrl subsystem بيعبّر عن كل pin configuration value كـ `unsigned long` واحد بس. الـ format: الـ 8 bits الأدنى هي `pin_config_param` (الـ enum)، والـ 24 bits الأعلى هي الـ argument (القيمة العددية زي mA أو Ohms). الـ helpers دي بتعمل الـ packing والـ unpacking بشكل type-safe.

---

#### `PIN_CONF_PACKED`

```c
#define PIN_CONF_PACKED(p, a) ((a << 8) | ((unsigned long) p & 0xffUL))
```

الـ macro الأساسي اللي بيعمل encode لـ `pin_config_param` مع argument عددي في `unsigned long` واحد. الـ param بياخد 8 bits الأدنى، والـ argument بياخد الـ 24 bits الفاضلة. بيستخدمه كل الـ inline wrappers الأخرى.

**Parameters:**
- `p` — الـ `enum pin_config_param` (زي `PIN_CONFIG_BIAS_PULL_UP`)
- `a` — القيمة العددية (مثلاً `100000` للـ ohms)

**Key details:** الـ argument بيتـ shift يسار 8 bits، يعني أقصى قيمة ممكنة للـ argument هي `0xFFFFFF` (16 million). لو الـ argument أكبر من كده، البيانات بتتقطع بدون أي warning — مسؤولية الـ driver.

---

#### `pinconf_to_config_param`

```c
static inline enum pin_config_param pinconf_to_config_param(unsigned long config)
{
    return (enum pin_config_param) (config & 0xffUL);
}
```

بيعمل decode للـ `pin_config_param` من الـ packed `unsigned long`. بيعمل AND مع `0xFF` عشان يجيب الـ 8 bits الأدنى بس.

**Parameters:**
- `config` — الـ packed unsigned long اللي اتعمله تخزين مسبقاً بـ `PIN_CONF_PACKED`

**Return:** الـ `enum pin_config_param` المقابل — زي `PIN_CONFIG_DRIVE_STRENGTH`

**Caller context:** بيتستدعى من الـ pin controller driver في الـ `.pin_config_set()` callback عشان يعرف إيه الـ config اللي المطلوب منه يطبقه.

---

#### `pinconf_to_config_argument`

```c
static inline u32 pinconf_to_config_argument(unsigned long config)
{
    return (u32) ((config >> 8) & 0xffffffUL);
}
```

بيستخرج الـ argument من الـ packed config. بيعمل right shift بـ 8 bits وبعدين AND مع `0xFFFFFF`.

**Parameters:**
- `config` — الـ packed unsigned long

**Return:** `u32` — القيمة العددية للـ argument. مثلاً لو الـ param هو `PIN_CONFIG_DRIVE_STRENGTH`، الـ return هيكون قيمة الـ mA.

**Caller context:** نفس سياق `pinconf_to_config_param` — الـ driver بيفك الـ encoding في الـ config callback. مثال حقيقي من `drivers/pinctrl/`:

```c
/* typical usage in a pin controller driver */
for (i = 0; i < num_configs; i++) {
    param = pinconf_to_config_param(configs[i]);
    arg   = pinconf_to_config_argument(configs[i]);

    switch (param) {
    case PIN_CONFIG_DRIVE_STRENGTH:
        set_drive_strength(pin, arg); /* arg is in mA */
        break;
    case PIN_CONFIG_BIAS_PULL_UP:
        enable_pull_up(pin, arg);
        break;
    /* ... */
    }
}
```

---

#### `pinconf_to_config_packed`

```c
static inline unsigned long pinconf_to_config_packed(enum pin_config_param param,
                                                     u32 argument)
{
    return PIN_CONF_PACKED(param, argument);
}
```

الـ inline function wrapper فوق الـ macro `PIN_CONF_PACKED`. بيوفر type checking أقوى لأنه function مش macro.

**Parameters:**
- `param` — الـ `enum pin_config_param`
- `argument` — القيمة العددية للـ config

**Return:** `unsigned long` — الـ packed config جاهز للتخزين في `pinctrl_map_configs.configs[]`

---

### Group 2: الـ DT-to-Map Parsing Functions

الـ functions دي هي قلب الـ generic pinconf DT parsing. هي اللي بتأخذ الـ Device Tree nodes وبتحوّلها لـ `struct pinctrl_map` entries اللي الـ pinctrl core بيفهمها. كل pin controller driver ممكن يستخدمها مباشرة بدل ما يكتب parser خاص بيه.

---

#### `pinconf_generic_dt_subnode_to_map`

```c
int pinconf_generic_dt_subnode_to_map(struct pinctrl_dev *pctldev,
        struct device_node *np, struct pinctrl_map **map,
        unsigned int *reserved_maps, unsigned int *num_maps,
        enum pinctrl_map_type type);
```

بيحول **subnode واحد** من الـ DT لـ map entries. الـ subnode ده بيحتوي على properties زي `pins`, `groups`, `bias-pull-up`, `drive-strength` إلخ. الـ function بتـ parse الـ pin/group names والـ config properties وبتبني الـ `pinctrl_map` entries المقابلة.

**Parameters:**
- `pctldev` — الـ `struct pinctrl_dev` للـ controller المسؤول. بيستخدمه للوصول للـ pin/group descriptors عشان يتحقق من الأسماء.
- `np` — الـ `device_node` للـ subnode اللي بيتـ parse. مثلاً `uart0_pins` node داخل الـ pinctrl node.
- `map` — pointer to pointer للـ array الجاري بناءه. الـ function ممكن تعمل realloc له.
- `reserved_maps` — عدد الـ entries المحجوزة في الـ map array حالياً (capacity).
- `num_maps` — عدد الـ entries المستخدمة فعلاً. بيتزود بعد كل entry جديدة.
- `type` — النوع المطلوب: `PIN_MAP_TYPE_CONFIGS_PIN`, `PIN_MAP_TYPE_CONFIGS_GROUP`, أو `PIN_MAP_TYPE_INVALID` للـ auto-detect.

**Return:** `0` للنجاح، أو error code سلبي زي `-ENOMEM` أو `-EINVAL`.

**Key details:**
- لو `type == PIN_MAP_TYPE_INVALID`، الـ function بتحدد النوع تلقائياً بناءً على وجود `pins` أو `groups` property.
- الـ function بتعمل allocate لـ `configs` arrays داخلياً، والـ cleanup بيتم عبر `pinconf_generic_dt_free_map`.
- بتمشي على كل الـ `pinconf_generic_params` المعرّفة وبتشيك لو موجودة في الـ DT node.

**Pseudocode flow:**
```
pinconf_generic_dt_subnode_to_map(pctldev, np, map, reserved_maps, num_maps, type):
    if type == INVALID:
        if np has "pins"  → type = CONFIGS_PIN
        if np has "groups" → type = CONFIGS_GROUP

    allocate configs[] array

    for each param in pinconf_generic_params[]:
        if of_property_read_u32(np, param.property, &value) == OK:
            configs[n++] = PIN_CONF_PACKED(param.param, value)

    if n == 0:
        free configs
        return 0  /* nothing to do */

    get pin/group names from np ("pins" or "groups" property)
    for each name:
        reserve_map(map, reserved_maps, num_maps, +1)
        map[*num_maps].type = type
        map[*num_maps].data.configs.group_or_pin = name
        map[*num_maps].data.configs.configs = configs
        map[*num_maps].data.configs.num_configs = n
        (*num_maps)++

    return 0
```

---

#### `pinconf_generic_dt_node_to_map`

```c
int pinconf_generic_dt_node_to_map(struct pinctrl_dev *pctldev,
        struct device_node *np_config, struct pinctrl_map **map,
        unsigned int *num_maps, enum pinctrl_map_type type);
```

هي الـ entry point الرئيسية للـ DT parsing. بتحول **node كامل** (اللي ممكن يحتوي على subnodes) لـ map entries. بتستدعي `pinconf_generic_dt_subnode_to_map` على كل subnode، وكمان بتعالج الـ node نفسها لو كانت تحتوي على config properties مباشرة.

**Parameters:**
- `pctldev` — نفس الـ pin controller device.
- `np_config` — الـ parent configuration node (المعروف بـ "pinctrl node" في الـ DT). مثلاً:
  ```dts
  &pinctrl {
      uart0_default: uart0-default {     /* ← np_config */
          pins1 {                         /* ← subnode */
              pins = "GPIO0", "GPIO1";
              bias-pull-up;
          };
      };
  };
  ```
- `map` — pointer to pointer للـ map array المخصص.
- `num_maps` — عدد الـ entries المبنية.
- `type` — نوع الـ mapping المطلوب.

**Return:** `0` أو error code سلبي.

**Key details:**
- الـ function بتبدأ بـ `*map = NULL` و `*num_maps = 0`.
- بتمشي على الـ subnodes بـ `for_each_available_child_of_node()`.
- لو الـ node نفسه (مش subnodes) عنده config properties، بتعالجهم كـ subnode أيضاً.
- لو فشلت، بتستدعي `pinconf_generic_dt_free_map` تلقائياً عشان تنظف.

**Pseudocode flow:**
```
pinconf_generic_dt_node_to_map(pctldev, np_config, map, num_maps, type):
    *map = NULL
    *num_maps = 0
    reserved_maps = 0

    /* handle the node itself if it has direct config properties */
    ret = pinconf_generic_dt_subnode_to_map(pctldev, np_config, map,
                                            &reserved_maps, num_maps, type)
    if ret < 0: goto err

    /* walk all subnodes */
    for_each_available_child_of_node(np_config, np):
        ret = pinconf_generic_dt_subnode_to_map(pctldev, np, map,
                                                &reserved_maps, num_maps, type)
        if ret < 0: goto err

    return 0
err:
    pinconf_generic_dt_free_map(pctldev, *map, *num_maps)
    return ret
```

---

#### `pinconf_generic_dt_free_map`

```c
void pinconf_generic_dt_free_map(struct pinctrl_dev *pctldev,
        struct pinctrl_map *map, unsigned int num_maps);
```

بتحرر كل الـ memory المخصصة من `pinconf_generic_dt_node_to_map`. بتمشي على كل الـ map entries وبتحرر الـ `configs` arrays الداخلية، وبعدين بتحرر الـ map array نفسه.

**Parameters:**
- `pctldev` — الـ pin controller device (مش بيتستخدم حالياً في الـ implementation لكن جزء من الـ API).
- `map` — الـ map array اللي اترجعت من `pinconf_generic_dt_node_to_map`. لو `NULL`، الـ function بترجع بدون أي عمل.
- `num_maps` — عدد الـ entries الصالحة في الـ array.

**Return:** `void`

**Key details:**
- بتيتـ assign لـ `.dt_free_map` callback في `struct pinctrl_ops` للـ driver.
- مهم إنها بتحرر الـ inner `configs` array الداخلي لكل entry من نوع `CONFIGS_PIN` أو `CONFIGS_GROUP` قبل ما تحرر الـ map array الخارجي.
- لو الـ map كان `NULL`، الـ function safe تتستدعى.

---

### Group 3: الـ Convenience Wrapper Inlines

الـ wrappers دي بتبسط استخدام `pinconf_generic_dt_node_to_map` عن طريق تثبيت الـ `type` parameter. كل driver بيختار الـ wrapper المناسب بناءً على كيفية تعريفه للـ pins في الـ DT.

---

#### `pinconf_generic_dt_node_to_map_group`

```c
static inline int pinconf_generic_dt_node_to_map_group(struct pinctrl_dev *pctldev,
        struct device_node *np_config, struct pinctrl_map **map,
        unsigned int *num_maps)
{
    return pinconf_generic_dt_node_to_map(pctldev, np_config, map, num_maps,
            PIN_MAP_TYPE_CONFIGS_GROUP);
}
```

يُستدعى من الـ `.dt_node_to_map` callback لو الـ driver بيتعامل مع configs على مستوى الـ groups فقط. مثال: SoC بيسمح بضبط الـ pull-up على group كامل من الـ pins مش على pin منفرد.

**Return:** مباشرة من `pinconf_generic_dt_node_to_map`.

---

#### `pinconf_generic_dt_node_to_map_pin`

```c
static inline int pinconf_generic_dt_node_to_map_pin(struct pinctrl_dev *pctldev,
        struct device_node *np_config, struct pinctrl_map **map,
        unsigned int *num_maps)
{
    return pinconf_generic_dt_node_to_map(pctldev, np_config, map, num_maps,
            PIN_MAP_TYPE_CONFIGS_PIN);
}
```

نفس الفكرة لكن كل config entry هتكون مرتبطة بـ pin فردي (مش group). مناسب للـ controllers اللي بتدعم per-pin configuration.

**Return:** مباشرة من `pinconf_generic_dt_node_to_map`.

---

#### `pinconf_generic_dt_node_to_map_all`

```c
static inline int pinconf_generic_dt_node_to_map_all(struct pinctrl_dev *pctldev,
        struct device_node *np_config, struct pinctrl_map **map,
        unsigned *num_maps)
{
    return pinconf_generic_dt_node_to_map(pctldev, np_config, map, num_maps,
            PIN_MAP_TYPE_INVALID);
}
```

الأذكى من بين الـ wrappers — بيمرر `PIN_MAP_TYPE_INVALID` عشان يخلي الـ parser يقرر النوع تلقائياً بناءً على محتوى الـ DT (وجود `pins` مقابل `groups`). الأنسب للـ controllers اللي بتدعم الاتنين.

**Return:** مباشرة من `pinconf_generic_dt_node_to_map`.

---

### Group 4: الـ Macros الـ Debug/Display

#### `PCONFDUMP` و `PCONFDUMP_WITH_VALUES`

```c
#define PCONFDUMP_WITH_VALUES(a, b, c, d, e, f) {       \
    .param = a, .display = b, .format = c, .has_arg = d,\
    .values = e, .num_values = f                        \
    }

#define PCONFDUMP(a, b, c, d)   PCONFDUMP_WITH_VALUES(a, b, c, d, NULL, 0)
```

الـ macros دي بتسهل إنشاء `struct pin_config_item` entries في الـ tables. الـ `pin_config_item` بيستخدمه الـ debugfs interface للـ pinctrl subsystem عشان يعرض الـ config values بشكل human-readable.

**Parameters لـ PCONFDUMP:**
- `a` — الـ `enum pin_config_param`
- `b` — الـ display string (الاسم اللي بيظهر في debugfs)
- `c` — الـ format string (زي `"%u"` أو `"mA"`)
- `d` — `bool` — هل الـ param عنده argument يتعرض؟

**مثال استخدام حقيقي من `drivers/pinctrl/pinconf-generic.c`:**
```c
static const struct pin_config_item conf_items[] = {
    PCONFDUMP(PIN_CONFIG_BIAS_BUS_HOLD,    "input bias bus hold", NULL, false),
    PCONFDUMP(PIN_CONFIG_BIAS_DISABLE,     "input bias disabled", NULL, false),
    PCONFDUMP(PIN_CONFIG_DRIVE_STRENGTH,   "output drive strength", "mA", true),
    PCONFDUMP(PIN_CONFIG_SLEW_RATE,        "slew rate", NULL, true),
    /* ... */
};
```

---

### الـ Data Structures المرتبطة

#### `struct pin_config_item`

```c
struct pin_config_item {
    const enum pin_config_param param;   /* which config param */
    const char * const display;          /* human-readable name for debugfs */
    const char * const format;           /* unit string, e.g. "mA" */
    bool has_arg;                        /* does this param take a value? */
    const char * const *values;          /* optional: named values array */
    size_t num_values;                   /* size of values array */
};
```

بتوصف كيفية عرض config param معين في الـ debugfs. الـ `values` array بتسمح بتسمية القيم العددية — مثلاً `{0: "disabled", 1: "enabled"}`.

---

#### `struct pinconf_generic_params`

```c
struct pinconf_generic_params {
    const char * const property;   /* DT property name, e.g. "bias-pull-up" */
    enum pin_config_param param;   /* corresponding PIN_CONFIG_* enum */
    u32 default_value;             /* value to use if property has no argument */
    const char * const *values;    /* optional named values */
    size_t num_values;             /* size of values array */
};
```

الجسر بين الـ DT properties والـ `pin_config_param` enum. الـ parser بيمشي على array من الـ `pinconf_generic_params` (معرّفة في `pinconf-generic.c`) وبيشيك لو كل property موجودة في الـ DT node. لو موجودة، بيعمل encode لها مع `default_value` (أو القيمة المقروءة من الـ DT) باستخدام `PIN_CONF_PACKED`.

**مثال:** الـ property `"bias-pull-up"` في الـ DT بتتحول لـ `PIN_CONFIG_BIAS_PULL_UP` مع `default_value = 1`.

---

### تدفق الـ DT Parsing الكامل — ASCII Diagram

```
Device Tree Node (e.g. uart0-pins)
        │
        ▼
pinconf_generic_dt_node_to_map_all()
        │
        ▼
pinconf_generic_dt_node_to_map(type=INVALID)
        │
        ├── pinconf_generic_dt_subnode_to_map(np_config itself)
        │        │
        │        ├── detect type (CONFIGS_PIN or CONFIGS_GROUP)
        │        ├── walk pinconf_generic_params[]
        │        │       └── of_property_read_u32() for each
        │        ├── allocate configs[]
        │        └── build pinctrl_map entries
        │
        └── for_each_available_child_of_node()
                 └── pinconf_generic_dt_subnode_to_map(child)
                          └── (same flow as above)

Final Result: struct pinctrl_map[] array
        │
        ▼
pinctrl core consumes it via pinctrl_register_mappings()
        │
        ▼
On cleanup: pinconf_generic_dt_free_map()
```

---

### ملاحظات مهمة للـ Driver Developer

1. **الـ `configs[]` array lifetime** — الـ configs المخصصة في `pinconf_generic_dt_subnode_to_map` بتعيش طول فترة الـ mapping، ولازم تتحرر بـ `pinconf_generic_dt_free_map` بس.

2. **الـ `PIN_CONFIG_END` و `PIN_CONFIG_MAX`** — الـ `END = 0x7F` هو آخر param معرّف في الـ generic API. الـ drivers ممكن تعرف custom params ابتداءً من `PIN_CONFIG_END + 1` لحد `PIN_CONFIG_MAX = 0xFF`. ده بيسمح بـ 128 custom param لكل driver.

3. **الـ Argument Encoding** — الـ 24-bit argument field معناه إن قيم الـ impedance وغيرها بتتضغط، ولازم تتعامل معاها بحذر لو بتتجاوز `0xFFFFFF`.

4. **الـ `PIN_MAP_TYPE_INVALID`** في `pinconf_generic_dt_node_to_map_all` — ده مش bug، ده feature: بيسمح لنفس الـ DT node تخلط بين per-pin وper-group configs.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المسارات والقراءة

**الـ pinctrl subsystem** بيعرض معلومات تفصيلية جداً عبر debugfs. mount بتاعها بيبقى عادةً على `/sys/kernel/debug`.

```bash
# mount debugfs لو مش موجودة
mount -t debugfs none /sys/kernel/debug

# الـ root directory للـ pinctrl
ls /sys/kernel/debug/pinctrl/
```

```
/sys/kernel/debug/pinctrl/
├── <controller-name>/          # مثلاً: pinctrl-bcm2835, gpio-0
│   ├── pinconf-pins            # pin configs للـ individual pins
│   ├── pinconf-groups          # pin configs للـ groups
│   ├── pinmux-functions        # المـ functions المتاحة
│   ├── pinmux-pins             # مين بيستخدم كل pin
│   ├── pins                    # كل الـ pins المسجلة
│   └── gpio-ranges             # الـ GPIO ranges المرتبطة
└── pinctrl-handles             # الـ handles النشطة لكل device
```

```bash
# اقرأ الـ pin configurations الحالية
cat /sys/kernel/debug/pinctrl/<ctrl>/pinconf-pins

# مثال output:
# Pin 23 (GPIO23): bias-pull-up, drive-strength:8
# Pin 24 (GPIO24): bias-disable, input-enable

# اقرأ الـ group configs
cat /sys/kernel/debug/pinctrl/<ctrl>/pinconf-groups

# شوف مين بيستخدم كل pin (مهم لتشخيص conflict)
cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-pins

# output نموذجي:
# pin 14 (GPIO14): uart0 (GPIO UNCLAIMED)
# pin 15 (GPIO15): uart0 (GPIO UNCLAIMED)
# pin 2  (GPIO2):  i2c1 GPIO i2c1:11

# شوف كل الـ handles النشطة
cat /sys/kernel/debug/pinctrl/pinctrl-handles
```

**تفسير الـ pinconf-pins output:**

| القيمة | المعنى |
|--------|--------|
| `bias-pull-up` | `PIN_CONFIG_BIAS_PULL_UP` مفعّل |
| `bias-pull-down` | `PIN_CONFIG_BIAS_PULL_DOWN` مفعّل |
| `bias-disable` | كل الـ bias معطّل |
| `drive-strength:N` | `PIN_CONFIG_DRIVE_STRENGTH` = N mA |
| `input-enable` | `PIN_CONFIG_INPUT_ENABLE` مفعّل |
| `output-enable` | `PIN_CONFIG_OUTPUT_ENABLE` مفعّل |
| `slew-rate:N` | `PIN_CONFIG_SLEW_RATE` = N |

---

#### 2. sysfs — المسارات المهمة

```bash
# الـ GPIO class — كل pin ممكن يتعرض كـ GPIO
ls /sys/class/gpio/

# export pin معين للـ userspace (مثلاً GPIO 23)
echo 23 > /sys/class/gpio/export

# بعد الـ export
ls /sys/class/gpio/gpio23/
# direction  value  edge  active_low  device  power  subsystem  uevent

# اقرأ الاتجاه الحالي
cat /sys/class/gpio/gpio23/direction   # in أو out

# اقرأ القيمة الحالية
cat /sys/class/gpio/gpio23/value       # 0 أو 1

# الـ pinctrl device في sysfs
ls /sys/bus/platform/drivers/pinctrl*/
find /sys/devices -name "pinctrl" -type d

# الـ states المتاحة لـ device معين
ls /sys/devices/.../pinctrl/
# default  init  idle  sleep  pinctrl-maps  pinctrl-devices

# الـ state الحالية
cat /sys/devices/.../pinctrl/pinctrl-state   # active state name
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

**الـ pinctrl subsystem** عنده tracepoints جاهزة:

```bash
# شوف الـ events المتاحة
ls /sys/kernel/debug/tracing/events/pinctrl/

# الـ events الرئيسية:
# pinctrl_single_config_pin
# pin_config_set
# pin_config_get

# فعّل كل events الـ pinctrl
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# أو event معين
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pin_config_set/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل الـ operation اللي بتحاول تـ debug
# مثلاً: بدّل state الـ device
echo default > /sys/devices/.../pinctrl/pinctrl-state   # مش الطريقة الصح لكن للتجربة

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# أو اتابع الـ trace live
cat /sys/kernel/debug/tracing/trace_pipe
```

**نموذج trace output:**

```
# tracer: nop
           <...>-123 [001] .... pin_config_set: pin=14 config=0x00000101
           <...>-123 [001] .... pin_config_set: pin=14 config=0x00010002
```

**تفسير الـ config value:** استخدم `PIN_CONF_PACKED` — الـ lower 8 bits هي `pin_config_param`، الـ upper 24 bits هي الـ argument.

```bash
# مثال: 0x00010002
# param = 0x02 = PIN_CONFIG_BIAS_PULL_DOWN
# arg   = 0x0001 = 1 (enabled)
```

**استخدام function_graph tracer لتتبع الـ call chain:**

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo pinconf_apply_setting > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

---

#### 4. printk / Dynamic Debug

**تفعيل dynamic debug للـ pinctrl subsystem:**

```bash
# فعّل كل debug messages في ملفات الـ pinctrl
echo 'file drivers/pinctrl/* +p' > /sys/kernel/debug/dynamic_debug/control

# أو لـ module معين
echo 'module pinctrl_bcm2835 +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع الـ stack trace لكل message
echo 'file drivers/pinctrl/pinconf.c +ps' > /sys/kernel/debug/dynamic_debug/control

# flags: p=printk, f=func, l=line, m=module, t=thread, s=stack

# اقرأ الـ rules الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep pinctrl
```

**من command line أثناء boot:**

```bash
# في kernel cmdline
dyndbg="file drivers/pinctrl/* +p"
# أو
pinctrl.dyndbg=+p
```

**تفعيل الـ loglevel لمشاهدة الرسائل:**

```bash
# ارفع الـ loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk

# أو
dmesg -n 8

# اتابع الـ kernel log
dmesg -w | grep -i pinctrl
journalctl -k -f | grep -i pinctrl
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_DEBUG_PINCTRL` | يفعّل debug messages أساسية في الـ pinctrl core |
| `CONFIG_PINCTRL` | يجب يكون مفعّل (أساسي) |
| `CONFIG_GENERIC_PINCONF` | يفعّل الـ generic pinconf (اللي بيستخدم `pinconf-generic.h`) |
| `CONFIG_DEBUG_FS` | يلزم لـ debugfs entries |
| `CONFIG_TRACING` | يلزم لـ ftrace |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل dynamic debug |
| `CONFIG_DEBUG_GPIO` | يفعّل GPIO debugging الإضافي |
| `CONFIG_PROVE_LOCKING` | يكشف lockdep violations في الـ pinctrl locks |
| `CONFIG_DEBUG_OBJECTS` | يكشف object lifecycle bugs |
| `CONFIG_KASAN` | يكشف memory bugs في الـ drivers |
| `CONFIG_KCSAN` | يكشف data races |

```bash
# فحص الـ config الحالي
zcat /proc/config.gz | grep -E 'PINCTRL|PINCONF|GENERIC_PIN'

# أو
grep -E 'PINCTRL|PINCONF' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**pinctrl-utils (من kernel tools):**

```bash
# مش موجودة كـ standalone tool عادةً، لكن ممكن تستخدم
# الـ sysfs interface مباشرة

# أداة gpio-event-mon من tools/gpio/
cd tools/gpio && make
./gpio-event-mon -n gpiochip0 -o 23   # راقب events على GPIO 23

# أداة lsgpio
./lsgpio   # قائمة بكل GPIO chips والـ lines

# output نموذجي:
# GPIO chip: gpiochip0, "pinctrl-bcm2835", 54 GPIO lines
#   line   0: unnamed           unused   input  active-high
#   line  14: "txd0"            "uart0"  output active-high [used]
#   line  15: "rxd0"            "uart0"  input  active-high [used]
```

**استخدام libgpiod:**

```bash
# gpioinfo — يشبه lsgpio لكن أشمل
gpioinfo gpiochip0

# gpioget — اقرأ قيمة pin
gpioget gpiochip0 23

# gpioset — اكتب قيمة pin
gpioset gpiochip0 23=1
```

---

#### 7. رسائل الـ Error الشائعة

| رسالة الخطأ | المعنى | الحل |
|-------------|--------|------|
| `pin X is already requested` | الـ pin محجوز بـ driver تاني | افحص `pinmux-pins` وحدد المالك، اتأكد مفيش تعارض في DT |
| `could not request pin X` | فشل طلب الـ pin | ممكن يكون محجوز أو الـ controller مش مسجّل |
| `pin config not supported` | الـ controller مش بيدعم الـ param ده | افحص `pin_config_item` table في الـ driver |
| `invalid pin configuration` | `PIN_CONF_PACKED` argument غلط | افحص الـ DT property values |
| `pinconfig ops missing` | الـ controller مش عامل implement الـ `pin_config_set` | الـ driver ناقص |
| `pinctrl state default not found` | الـ device طلب state مش موجودة في mapping | افحص الـ DT pinctrl-0 و pinctrl-names |
| `pinmux: cannot mux pin X` | تعارض بين اتنين devices على نفس الـ pin | افحص الـ DT وتأكد كل device بيستخدم pins منفصلة |
| `Property bias-pull-up takes value ranges 0-1` | argument خاطئ لـ DT property | راجع قيمة الـ property في DT |
| `-ENODEV: pinctrl not found` | الـ pinctrl controller مش probe اتعمل له | افحص dmesg عن errors في الـ controller probe |
| `pinctrl_get() failed for state 'default'` | الـ device مش لاقي الـ state المطلوبة | افحص `pinctrl-names` في DT |
| `pin X already in use` | الـ GPIO request conflict مع pinmux | تأكد الـ pin مش مستخدم كـ GPIO وكـ function في نفس الوقت |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

**الأماكن المثالية لإضافة debugging assertions في الـ pinconf-generic code:**

```c
/* في pinconf_generic_dt_subnode_to_map — بعد parsing الـ config */
int pinconf_generic_dt_subnode_to_map(struct pinctrl_dev *pctldev,
                                       struct device_node *np, ...)
{
    unsigned long config;
    /* تحقق إن الـ param في النطاق الصح */
    WARN_ON(pinconf_to_config_param(config) > PIN_CONFIG_END);

    /* تحقق إن الـ map pointer مش NULL قبل الكتابة */
    WARN_ON(!map);
}
```

```c
/* في الـ driver عند تطبيق الـ config */
static int mydrv_pin_config_set(struct pinctrl_dev *pctldev,
                                 unsigned pin, unsigned long *configs,
                                 unsigned num_configs)
{
    int i;
    for (i = 0; i < num_configs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        u32 arg = pinconf_to_config_argument(configs[i]);

        /* dump_stack لو شوفنا config غريب */
        if (param == PIN_CONFIG_MAX) {
            dev_err(pctldev->dev, "unexpected PIN_CONFIG_MAX\n");
            dump_stack();
            return -EINVAL;
        }

        WARN_ON(param > PIN_CONFIG_END && param < PIN_CONFIG_MAX);
    }
}
```

**نقاط مهمة:**

- **قبل** `pinconf_apply_setting()`: تحقق إن الـ `configs` array مش فاضية
- **بعد** DT parsing: تحقق إن `num_maps > 0` لو الـ node غير فاضي
- **عند** `PIN_CONFIG_PERSIST_STATE`: تحقق إن الـ hardware فعلاً بيدعم state retention
- **في** الـ `pin_config_get`: استخدم `WARN_ON_ONCE` لو الـ driver رجّع config مختلف عن اللي اتسجّل

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

الفكرة: الـ kernel بيحفظ الـ desired state، الـ hardware registers بتحفظ الـ actual state — لازم يتطابقوا.

```bash
# خطوة 1: اقرأ الـ kernel state
cat /sys/kernel/debug/pinctrl/<ctrl>/pinconf-pins
# Pin 23: bias-pull-up, drive-strength:8

# خطوة 2: اقرأ الـ hardware register المقابل
# (العنوان بيتوقف على الـ SoC — مثال: BCM2835)
# GPIO Pull-up/down register: base + 0x94
devmem2 0x3F200094 w   # GPPUD — كنترول الـ pull up/down
devmem2 0x3F20009C w   # GPPUDCLK0 — clock للـ GPIO 0-31

# قارن النتيجة بالـ datasheet
# لو الـ bit مش متوافق مع الـ kernel state → bug في الـ driver
```

**script للمقارنة الآلية:**

```bash
#!/bin/bash
# compare_pinconf.sh — قارن kernel state مع hardware registers

PIN=23
KERNEL_STATE=$(cat /sys/kernel/debug/pinctrl/pinctrl-bcm2835/pinconf-pins | grep "Pin $PIN")
echo "Kernel: $KERNEL_STATE"

# اقرأ الـ FSEL register (function select)
FSEL_REG=$((0x3F200000 + (PIN / 10) * 4))
FSEL_VAL=$(devmem2 $FSEL_REG w 2>/dev/null | grep "Value" | awk '{print $NF}')
FSEL_SHIFT=$(( (PIN % 10) * 3 ))
FSEL_BITS=$(( (FSEL_VAL >> FSEL_SHIFT) & 7 ))
echo "Hardware FSEL: $FSEL_BITS (0=input, 1=output, 4-7=alt)"
```

---

#### 2. Register Dump — الأدوات والتقنيات

```bash
# devmem2 — أكثر أداة شائعة
# تثبيت
apt install devmem2
# أو من المصدر: https://github.com/radimkarnis/devmem2

# قراءة register واحد (32-bit)
devmem2 0x3F200000 w

# كتابة register
devmem2 0x3F200094 w 0x00000002   # set pull-down

# io — بديل خفيف
apt install busybox
busybox devmem 0x3F200000 32

# /dev/mem — للقراءة اليدوية
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mem = mmap.mmap(f.fileno(), 0x100, offset=0x3F200000)
    val = struct.unpack('<I', mem.read(4))[0]
    print(f'GPFSEL0: 0x{val:08x}')
"

# iomem — شوف الـ memory map
cat /proc/iomem | grep -i gpio
# 3f200000-3f2000b3 : /soc/gpio@7e200000
```

**dump سريع لمنطقة registers:**

```bash
# dump 256 bytes من الـ GPIO base
for i in $(seq 0 4 252); do
    addr=$(printf "0x%x" $((0x3F200000 + i)))
    val=$(devmem2 $addr w 2>/dev/null | grep "Value" | awk '{print $NF}')
    echo "$addr: $val"
done
```

---

#### 3. Logic Analyzer / Oscilloscope — نصائح عملية

**للـ pull-up/pull-down verification:**

```
         VDD (3.3V)
           │
          [R]  ← pull-up resistor (داخلي أو خارجي)
           │
    Pin ───┤────── Probe CH1
           │
          [R]  ← pull-down resistor
           │
          GND
```

- **Oscilloscope**: قيس الـ voltage level في حالة floating — لو pull-up شغال هيكون قريب من VDD
- **Logic Analyzer**: استخدم trigger على rising/falling edge عشان تتحقق من الـ `INPUT_DEBOUNCE` timing
- **Drive Strength**: شغّل square wave بـ GPIO وقيس الـ rise time — كلما زادت الـ strength، أسرع الـ rise time

```
# جدول rise time تقريبي لـ BCM2835
| Drive Strength | Rise Time (typ) |
|----------------|-----------------|
| 2 mA           | ~50 ns          |
| 4 mA           | ~25 ns          |
| 8 mA           | ~12 ns          |
| 16 mA          | ~6 ns           |
```

**لـ Schmitt Trigger (PIN_CONFIG_INPUT_SCHMITT):**
- قيس الـ hysteresis: الـ threshold من low→high يختلف عن high→low
- نموذجي: V_IH ≈ 1.6V، V_IL ≈ 0.9V (لـ 3.3V GPIO)

---

#### 4. مشاكل Hardware شائعة وأنماطها في الـ Kernel Log

| المشكلة | Pattern في الـ dmesg | التشخيص |
|---------|---------------------|---------|
| Pin مش موجود فعلياً في الـ HW | `pin X: no function found` | الـ DT بيحدد pin رقمه غلط |
| Power domain مش شغال | `pinctrl: failed to enable power` | الـ voltage regulator مش جاهز |
| I2C pull-up ضعيف | `i2c: timeout, scl stuck low` | `PIN_CONFIG_DRIVE_STRENGTH` صغير أو pull-up مش شغال |
| SPI MISO floating | بيانات خاطئة أو random | `PIN_CONFIG_BIAS_PULL_DOWN` ناقص على MISO |
| UART noise | `uart: parity error` | `PIN_CONFIG_INPUT_SCHMITT` مش مفعّل أو مفيش |
| GPIO interrupt مش بيشتغل | `gpio-keys: no irq` | `PIN_CONFIG_BIAS_PULL_UP` ناقص على الـ interrupt pin |
| Pin state مش بيتحفظ بعد suspend | device مش بيشتغل بعد resume | `PIN_CONFIG_PERSIST_STATE` مش متسجّل |
| Open-drain مش شغال | `i2c: arbitration lost` | `PIN_CONFIG_DRIVE_OPEN_DRAIN` مش متسجّل |

---

#### 5. Device Tree Debugging

**التحقق إن الـ DT بيطابق الـ Hardware:**

```bash
# اقرأ الـ DT المستخدم فعلياً (المُدمج في الـ kernel)
ls /sys/firmware/devicetree/base/

# استخرج الـ DT الحالي كـ dts قابل للقراءة
dtc -I fs -O dts /sys/firmware/devicetree/base/ > /tmp/current.dts

# أو استخدم fdtdump
fdtdump /boot/dtb-$(uname -r) | grep -A 20 "pinctrl"

# تحقق من الـ pinctrl nodes
grep -A 30 "pinctrl@" /tmp/current.dts
```

**مثال DT صحيح لـ pin config يستخدم `pinconf-generic`:**

```dts
&i2c1 {
    pinctrl-names = "default";
    pinctrl-0 = <&i2c1_pins>;
};

i2c1_pins: i2c1 {
    brcm,pins = <2 3>;
    brcm,function = <4>;            /* ALT0 = I2C */
    brcm,pull = <2>;                /* pull-up */
    /* الـ generic pinconf properties المقابلة: */
    /* bias-pull-up; */
    /* drive-strength = <8>; */
};
```

**أخطاء DT شائعة:**

```bash
# لو الـ property اسمه غلط — مش هيطبّق وملوش error واضح
# افحص الـ pinconf_generic_params table في الـ driver
cat /sys/kernel/debug/pinctrl/<ctrl>/pinconf-pins
# لو الـ config مش ظاهر → الـ DT property مش اتعرف

# تحقق من الـ DT bindings
cat /sys/firmware/devicetree/base/<device>/pinctrl-0
# يرجع phandle للـ pin config node

# فك الـ phandle
grep -r "phandle" /sys/firmware/devicetree/base/ 2>/dev/null | head -20
```

**script للتحقق من الـ DT pinconf properties:**

```bash
#!/bin/bash
# check_dt_pinconf.sh
NODE="/sys/firmware/devicetree/base/soc/gpio@7e200000"

echo "=== Pin Config Properties ==="
for prop in bias-pull-up bias-pull-down bias-disable drive-strength \
            input-enable output-enable slew-rate input-schmitt-enable; do
    if [ -f "$NODE/$prop" ]; then
        val=$(xxd "$NODE/$prop" 2>/dev/null | head -1)
        echo "  $prop: $val"
    fi
done
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

**1. فحص سريع لحالة الـ pinctrl:**

```bash
# شوف كل الـ controllers المسجّلة
ls /sys/kernel/debug/pinctrl/

# شوف كل الـ configs دفعة واحدة
for ctrl in /sys/kernel/debug/pinctrl/*/; do
    echo "=== $(basename $ctrl) ==="
    cat "$ctrl/pinconf-pins" 2>/dev/null | head -20
    echo
done
```

**2. تشخيص conflict في الـ pins:**

```bash
# ابحث عن pins محجوزة من أكثر من device
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -v "UNCLAIMED" | sort

# ابحث عن errors في dmesg المتعلقة بالـ pinctrl
dmesg | grep -iE "pinctrl|pinconf|pinmux|pin.*request|pin.*conflict"

# مثال output:
# [    2.145] pinctrl: pin GPIO14 already requested by uart0; cannot claim for spi0
```

**3. قراءة config لـ pin معين:**

```bash
# استخراج config لـ pin رقم 14 فقط
cat /sys/kernel/debug/pinctrl/pinctrl-bcm2835/pinconf-pins | grep "pin 14"

# output نموذجي:
# pin 14 (GPIO14): bias-disable, drive-strength: 8 mA, output-enable
```

**4. فك تشفير قيمة PIN_CONF_PACKED:**

```bash
# مثال: قيمة config = 0x00000206
# param = 0x06 = PIN_CONFIG_DRIVE_STRENGTH
# arg   = 0x000002 = 2 mA
python3 -c "
config = 0x00000206
param = config & 0xFF
arg   = (config >> 8) & 0xFFFFFF
params = {
    0: 'BIAS_BUS_HOLD', 1: 'BIAS_DISABLE', 2: 'BIAS_HIGH_IMPEDANCE',
    3: 'BIAS_PULL_DOWN', 4: 'BIAS_PULL_PIN_DEFAULT', 5: 'BIAS_PULL_UP',
    6: 'DRIVE_OPEN_DRAIN', 7: 'DRIVE_OPEN_SOURCE', 8: 'DRIVE_PUSH_PULL',
    9: 'DRIVE_STRENGTH', 10: 'DRIVE_STRENGTH_UA', 11: 'INPUT_DEBOUNCE',
    12: 'INPUT_ENABLE', 13: 'INPUT_SCHMITT', 14: 'INPUT_SCHMITT_ENABLE',
    15: 'INPUT_SCHMITT_UV', 16: 'MODE_LOW_POWER', 17: 'MODE_PWM',
    18: 'LEVEL', 19: 'OUTPUT_ENABLE', 20: 'OUTPUT_IMPEDANCE_OHMS',
    21: 'PERSIST_STATE', 22: 'POWER_SOURCE', 23: 'SKEW_DELAY',
}
print(f'param = {params.get(param, hex(param))}')
print(f'arg   = {arg}')
"
```

**5. تفعيل debug logging شامل وحفظ الـ output:**

```bash
# فعّل كل debug levels
echo 8 > /proc/sys/kernel/printk
echo 'file drivers/pinctrl/* +pmfl' > /sys/kernel/debug/dynamic_debug/control
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# احفظ الـ output
dmesg -w > /tmp/pinctrl_debug.log &
DMESG_PID=$!

# اعمل الـ operation المشكلة هنا
# مثلاً: probe driver معين
echo "my_device" > /sys/bus/platform/drivers/my_driver/bind

# وقف الـ logging
kill $DMESG_PID
echo 0 > /sys/kernel/debug/tracing/tracing_on
cp /sys/kernel/debug/tracing/trace /tmp/ftrace_pinctrl.log

echo "Logs saved to /tmp/pinctrl_debug.log and /tmp/ftrace_pinctrl.log"
```

**6. تحقق من الـ DT parsing:**

```bash
# شوف إيه الـ maps اللي اتعملت من الـ DT
cat /sys/kernel/debug/pinctrl/pinctrl-handles

# output نموذجي:
# device: e6060000.i2c
# state: default
#   type: MUX_GROUP controller pinctrl@e6060000 group i2c2 function i2c
#   type: CONFIGS_PIN controller pinctrl@e6060000 pin SCL2 config 00000165

# فك config 0x00000165
# param = 0x65 = 101... هنا بيكون custom vendor param
# arg = 0x0001 = 1
```

**7. اختبار pin config من userspace (للـ testing فقط):**

```bash
# export GPIO وجرب الـ configs المختلفة
echo 23 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio23/direction
echo 1 > /sys/class/gpio/gpio23/value

# قيس الـ output voltage بالـ multimeter
# غيّر الـ drive strength في الـ DT وأعد التجربة

# unexport بعد الخلاص
echo 23 > /sys/class/gpio/unexport
```

**8. مقارنة state قبل وبعد suspend:**

```bash
# احفظ الـ state قبل الـ suspend
cat /sys/kernel/debug/pinctrl/*/pinconf-pins > /tmp/before_suspend.txt

# اعمل suspend وrwake
echo mem > /sys/power/state

# بعد الـ resume — قارن
cat /sys/kernel/debug/pinctrl/*/pinconf-pins > /tmp/after_suspend.txt
diff /tmp/before_suspend.txt /tmp/after_suspend.txt

# لو فيه فرق → مشكلة في PIN_CONFIG_PERSIST_STATE أو suspend/resume hooks
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: I2C بيشتغل بس بيتعلق على RK3562 Industrial Gateway

#### العنوان
**الـ I2C bus بيتعلق بعد أول transaction على gateway صناعي بـ RK3562**

#### السياق
شركة بتعمل industrial gateway بـ RK3562 لجمع بيانات من حساسات Modbus عبر I2C. الـ board بتشتغل تمام على bench، لكن في البيئة الصناعية بعد ساعة بتتعلق الـ I2C ومحتاج restart كامل.

#### المشكلة
الـ I2C controller بيدخل في حالة hang. الـ `dmesg` بيظهر:

```
i2c i2c-1: timeout waiting for bus ready
i2c i2c-1: i2c transfer failed: -110
```

الـ oscilloscope بيُريّ إن الـ SDA line واقفة على low — classic I2C bus stuck.

#### التحليل
المهندس فتح الـ DTS وشاف إن الـ I2C pins معملالهاش pull-up صح:

```dts
/* DTS قبل الإصلاح — غلط */
&i2c1 {
    pinctrl-names = "default";
    pinctrl-0 = <&i2c1_pins>;
    status = "okay";
};

&pinctrl {
    i2c1_pins: i2c1-pins {
        rockchip,pins =
            <0 RK_PB3 1 &pcfg_pull_none>,  /* SDA */
            <0 RK_PB4 1 &pcfg_pull_none>;  /* SCL */
    };
};
```

**الـ `pcfg_pull_none`** يعني `PIN_CONFIG_BIAS_DISABLE` — مفيش pull-up خالص. الـ I2C بيحتاج `PIN_CONFIG_DRIVE_OPEN_DRAIN` مع pull-up خارجي أو internal pull-up.

**الكود المسؤول في `pinconf-generic.h`:**

```c
/* الماكرو ده بيـpack الـ param في lower 8 bits والـ argument في upper 24 */
#define PIN_CONF_PACKED(p, a) ((a << 8) | ((unsigned long) p & 0xffUL))
```

لما الـ DTS بيقول `bias-pull-none`، الـ `pinconf_generic_dt_subnode_to_map()` بتحول ده لـ:

```c
PIN_CONF_PACKED(PIN_CONFIG_BIAS_DISABLE, 0)
// = (0 << 8) | (1 & 0xFF) = 0x00000001
```

يعني الـ hardware بيـdisable أي bias تماماً. لما noise أو glitch بيحصل في البيئة الصناعية، الـ SDA بتـfloat لـ low وما فيش حاجة ترفعها.

الـ `pinconf_generic_params` struct الـ mapping من DT property للـ enum:

```c
struct pinconf_generic_params {
    const char * const property;     /* "bias-pull-up" مثلاً */
    enum pin_config_param param;     /* PIN_CONFIG_BIAS_PULL_UP */
    u32 default_value;               /* القيمة الافتراضية */
    ...
};
```

الـ lookup بيتم في `pinconf_generic_dt_subnode_to_map()` اللي بتمشي على الـ params table وتدور على matching DT property.

#### الحل

```dts
/* DTS بعد الإصلاح */
&pinctrl {
    i2c1_pins: i2c1-pins {
        rockchip,pins =
            /* open-drain + internal pull-up ~47kΩ على كل line */
            <0 RK_PB3 1 &pcfg_pull_up>,
            <0 RK_PB4 1 &pcfg_pull_up>;
    };
};
```

ده بيـgenerate في الـ map:

```c
PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 1)
// lower 8 bits = PIN_CONFIG_BIAS_PULL_UP (5)
// upper 24 bits = 1 (enabled)
```

للـ verification:

```bash
# اتأكد إن الـ config اتطبق
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pins | grep -A5 "pb3"

# شوف الـ config المطبق فعلاً
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinconf-pins | grep "pb3"
```

#### الدرس المستفاد
الـ `PIN_CONFIG_BIAS_DISABLE` مش "default safe" — في البيئات الصناعية، الـ floating lines بتسبب lockups. أي open-drain bus زي I2C أو 1-Wire لازم يكون عنده pull-up صريح في الـ DT، سواء internal عبر `bias-pull-up` أو external resistor مع `drive-open-drain`.

---

### السيناريو 2: SPI Flash بيتكلم بـ garbage على STM32MP1 IoT Sensor

#### العنوان
**الـ SPI NOR flash بيرجع corrupt data على STM32MP157 IoT node**

#### السياق
منتج IoT sensor بيستخدم STM32MP157 مع W25Q64 SPI NOR flash لتخزين configuration. الـ flash بيُقرأ تمام على bench بـ 10 MHz، لكن لما الـ production firmware شغّل بـ 40 MHz بدأ الـ data يطلع corrupt.

#### المشكلة
```
m25p80 spi0.0: unrecognized JEDEC id bytes: ff ff ff
spi_master spi0: failed to transfer one message from queue
```

الـ logic analyzer بيُريّ الـ MISO line بتتأخر — الـ signal edges مش clean عند 40 MHz.

#### التحليل
المهندس فحص الـ DTS للـ SPI pins:

```dts
/* STM32MP1 DTS — قبل */
spi0_pins_a: spi0-0 {
    pins1 {
        pinmux = <STM32_PINMUX('A', 5, AF5)>,  /* SCK  */
                 <STM32_PINMUX('A', 7, AF5)>;  /* MOSI */
        bias-disable;
        /* مفيش drive-strength مذكور! */
    };
    pins2 {
        pinmux = <STM32_PINMUX('A', 6, AF5)>;  /* MISO */
        bias-disable;
        input-enable;
    };
};
```

الـ `pinconf_generic_dt_subnode_to_map()` بتـparse الـ DT node وتبني `pinctrl_map` entries. لما مفيش `drive-strength` مذكور، الـ STM32 pinctrl driver بيستخدم default value من الـ `pinconf_generic_params` table.

الـ STM32 driver عنده:

```c
/* من drivers/pinctrl/stm32/pinctrl-stm32.c */
static const struct pinconf_generic_params stm32_dt_params[] = {
    { "st,slew-rate", PIN_CONFIG_SLEW_RATE, 0 },  /* default = low speed */
    ...
};
```

الـ `default_value = 0` يعني **low slew rate** — slow edges، مناسب لـ 10 MHz بس مش لـ 40 MHz.

الـ packed value اللي اتبعت للـ hardware:

```c
pinconf_to_config_packed(PIN_CONFIG_SLEW_RATE, 0)
// = PIN_CONF_PACKED(PIN_CONFIG_SLEW_RATE, 0)
// = (0 << 8) | (PIN_CONFIG_SLEW_RATE & 0xFF)
// = PIN_CONFIG_SLEW_RATE = 0x1A (26)
```

الـ driver بعدين بيعمل:

```c
unsigned long config = pinconf_to_config_packed(PIN_CONFIG_SLEW_RATE, 0);
enum pin_config_param param = pinconf_to_config_param(config);
// param = config & 0xFF = 26 = PIN_CONFIG_SLEW_RATE ✓
u32 arg = pinconf_to_config_argument(config);
// arg = (config >> 8) & 0xFFFFFF = 0 → low speed
```

#### الحل

```dts
/* STM32MP1 DTS — بعد الإصلاح */
spi0_pins_a: spi0-0 {
    pins1 {
        pinmux = <STM32_PINMUX('A', 5, AF5)>,
                 <STM32_PINMUX('A', 7, AF5)>;
        bias-disable;
        drive-push-pull;
        slew-rate = <3>;    /* very high speed على STM32 */
        output-impedance-ohms = <50>;
    };
    pins2 {
        pinmux = <STM32_PINMUX('A', 6, AF5)>;
        bias-pull-up;
        input-enable;
        input-schmitt-enable;   /* PIN_CONFIG_INPUT_SCHMITT_ENABLE */
    };
};
```

الـ `slew-rate = <3>` بيـgenerate:
```c
PIN_CONF_PACKED(PIN_CONFIG_SLEW_RATE, 3)
// = (3 << 8) | 0x1A = 0x0000031A
```

للـ debug:
```bash
# على الـ target
cat /sys/kernel/debug/pinctrl/soc:pin-controller/pinconf-groups | grep -A10 "spi0"

# شوف الـ slew rate المطبق
grep -r "slew" /sys/kernel/debug/pinctrl/
```

#### الدرس المستفاد
الـ `pinconf_generic_params.default_value` بيـsilently override غياب الـ DT property. عند رفع الـ SPI clock frequency، لازم تكون explicit في الـ DT مع `slew-rate` و`drive-push-pull` — ماتعتمدش على الـ defaults عشان ممكن تكون conservative جداً.

---

### السيناريو 3: UART بيفضل يـwake-up الـ system من Sleep على i.MX8M

#### العنوان
**الـ system بيصحى من suspend بدون سبب على i.MX8M automotive ECU**

#### السياق
automotive ECU بيستخدم i.MX8M Plus. المتطلب إن الـ system يـsuspend لـ deep sleep ويصحى بس لما يجي data على UART4 من الـ CAN transceiver. المشكلة: الـ system بيصحى كل شوية بدون data.

#### المشكلة
```
PM: suspend entry (deep)
PM: Wakeup source: serial4-uart
...
PM: suspend exit
# بيتكرر كل 30 ثانية تقريباً
```

الـ UART4 RX line فاضية — مفيش data.

#### التحليل
فحص الـ pinctrl sleep state في الـ DTS:

```dts
/* i.MX8M DTS — المشكلة */
&uart4 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&pinctrl_uart4>;
    pinctrl-1 = <&pinctrl_uart4_sleep>;
    fsl,uart-has-rtscts;
    status = "okay";
};

pinctrl_uart4_sleep: uart4grp-sleep {
    fsl,pins = <
        MX8MP_IOMUXC_UART4_RXD__UART4_DCE_RX   0x140  /* المشكلة هنا */
        MX8MP_IOMUXC_UART4_TXD__UART4_DCE_TX   0x140
    >;
};
```

القيمة `0x140` في i.MX8 format = `PIN_CONFIG_BIAS_PULL_UP` enabled. الـ RX line عندها internal pull-up لـ 3.3V، لكن الـ CAN transceiver في الـ sleep mode بيسحب الـ line لـ level متغيّر بسبب الـ leakage current، وده بيخلي الـ UART controller يشوف edges وهمية.

لما `pinconf_generic_dt_subnode_to_map()` بتـparse الـ sleep state، بتبني:

```c
/* الـ map entry للـ sleep state */
struct pinctrl_map sleep_map = {
    .dev_name = "uart4",
    .name = "sleep",                           /* PINCTRL_STATE_SLEEP */
    .type = PIN_MAP_TYPE_CONFIGS_PIN,
    .data.configs = {
        .group_or_pin = "UART4_RXD",
        .configs = &packed_config,             /* bias-pull-up */
        .num_configs = 1,
    },
};
```

الـ `PIN_CONFIG_SLEEP_HARDWARE_STATE` موجود في الـ enum بس مش مستخدم هنا. الحل الصح هو استخدام `PIN_CONFIG_BIAS_HIGH_IMPEDANCE` أو `PIN_CONFIG_BIAS_PULL_DOWN` على الـ RX في sleep state.

```c
/* الـ packed values المختلفة */
PIN_CONF_PACKED(PIN_CONFIG_BIAS_HIGH_IMPEDANCE, 0)  /* hi-Z — أفضل للـ sleep */
PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_DOWN, 1)       /* pull-down — يمنع الـ spurious wakeup */
PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 1)         /* ده اللي عنده — بيسبب المشكلة */
```

الـ `PIN_CONFIG_BIAS_PULL_UP` = 5 في الـ enum:
```c
pinconf_to_config_param(0x140)
// 0x140 & 0xFF = 0x40 = 64... مش بالضبط PIN_CONFIG_BIAS_PULL_UP
// i.MX8 بيستخدم format مختلف — الـ 0x140 ده iomux raw value مش generic packed
```

هنا التعقيد: الـ i.MX8 بيستخدم raw IOMUX values مش الـ generic `PIN_CONF_PACKED()` format. لكن لما تستخدم `pinconf_generic_dt_node_to_map_all()` مع DT properties زي `bias-pull-up`، الكود بيستخدم الـ generic format صح.

#### الحل

```dts
pinctrl_uart4_sleep: uart4grp-sleep {
    fsl,pins = <
        /* pull-down على RX في sleep — يمنع الـ float */
        MX8MP_IOMUXC_UART4_RXD__GPIO5_IO28   0x104  /* GPIO mode + pull-down */
        MX8MP_IOMUXC_UART4_TXD__UART4_DCE_TX 0x104
    >;
};
```

أو باستخدام الـ generic properties:

```dts
pinctrl_uart4_sleep: uart4grp-sleep {
    pins {
        pinmux = "UART4_RXD";
        bias-pull-down;
        input-enable;
        sleep-hardware-state;   /* PIN_CONFIG_SLEEP_HARDWARE_STATE */
    };
};
```

للـ debug:
```bash
# شوف الـ current pinctrl state
cat /sys/kernel/debug/pinctrl/pinctrl-imx/pinconf-pins | grep -i "uart4"

# افحص الـ wakeup sources
cat /sys/power/pm_wakeup_sources

# تحقق من الـ sleep state transitions
echo 1 > /sys/kernel/debug/pinctrl/print_mutex_stats
```

#### الدرس المستفاد
الـ `PIN_CONFIG_SLEEP_HARDWARE_STATE` موجود في `pinconf-generic.h` عشان الـ driver يعرف إن الـ config ده خاص بحالة النوم. الـ sleep state في الـ DT مش بس للـ power saving — غلط في الـ bias config بيـruin الـ suspend/resume cycle كله. الـ RX lines اللي ممكن تـfloat في sleep لازم تاخد `bias-pull-down` مش `bias-pull-up`.

---

### السيناريو 4: HDMI مش بيظهر على Android TV Box بـ Allwinner H616

#### العنوان
**الـ HDMI output مش بيظهر بعد ما الـ DDC pins اتغير configuration على H616 TV box**

#### السياق
منتج Android TV box بيستخدم Allwinner H616. بعد kernel upgrade من 5.15 لـ 6.1، الـ HDMI بطل يشتغل — الشاشة مش بتتعرف على الـ device. الـ kernel log:

```
sun8i-hdmi-phy: DDC I2C read failed: -6
drm: failed to get edid
```

#### المشكلة
الـ HDMI DDC (Display Data Channel) بيستخدم I2C للـ EDID. الـ DDC pins فيها pull-up 4.7kΩ خارجية على الـ board، بس الـ kernel driver بيـenable internal pull-up كمان — وده بيعمل conflict.

#### التحليل
في الـ kernel 6.1، الـ Allwinner H616 pinctrl driver اتحدّث وأضاف default `bias-pull-up` للـ DDC pins. الـ DTS:

```dts
/* sun50i-h616 DTS — kernel 6.1 */
hdmi_ddc_pins: hdmi-ddc-pins {
    pins = "PH8", "PH9";
    function = "ddc";
    bias-pull-up;   /* أضيف في الـ 6.1 */
};
```

لما `pinconf_generic_dt_subnode_to_map()` بتـprocess ده:

```c
/* بتدور في الـ params table على "bias-pull-up" property */
static const struct pinconf_generic_params dt_params[] = {
    { "bias-disable",       PIN_CONFIG_BIAS_DISABLE,       0 },
    { "bias-pull-up",       PIN_CONFIG_BIAS_PULL_UP,       1 },  /* match! */
    { "bias-pull-down",     PIN_CONFIG_BIAS_PULL_DOWN,     0 },
    ...
};

/* الـ packed config اللي اتحط في الـ map */
unsigned long config = PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 1);
// = (1 << 8) | 5 = 0x105
```

الـ `pinconf_to_config_param(0x105)`:
```c
return (enum pin_config_param)(0x105 & 0xFF);
// = 5 = PIN_CONFIG_BIAS_PULL_UP ✓
```

الـ `pinconf_to_config_argument(0x105)`:
```c
return (u32)((0x105 >> 8) & 0xFFFFFF);
// = 1 → pull-up enabled
```

الـ H616 internal pull-up قيمته حوالي 100kΩ. عنده مع الـ external 4.7kΩ:
```
R_parallel = (100k × 4.7k) / (100k + 4.7k) ≈ 4.5kΩ
```

ده مش مشكلة كبيرة من ناحية الـ resistance، بس المشكلة إن الـ H616 internal pull-up بيتأخر في enable والـ DDC controller بيبدأ الـ transaction قبل ما الـ line تـsettle، فبيشوف glitch في البداية.

#### الحل

```dts
/* board-level DTS override */
&hdmi_ddc_pins {
    bias-disable;   /* نعطل الـ internal pull-up، الخارجي كافي */
};
```

ده بيبدّل الـ packed config:
```c
PIN_CONF_PACKED(PIN_CONFIG_BIAS_DISABLE, 0)
// = (0 << 8) | 1 = 0x001
// param = 0x001 & 0xFF = 1 = PIN_CONFIG_BIAS_DISABLE ✓
```

للـ verification:
```bash
# على Android (adb shell أو serial console)
cat /sys/kernel/debug/pinctrl/pio/pinconf-pins | grep -E "PH8|PH9"

# تست الـ HDMI DDC مباشرة
i2cdetect -y 3  # إذا كان DDC bus هو i2c-3
```

```bash
# debug أعمق — شوف الـ map entries المبنية
echo 1 > /sys/kernel/debug/pinctrl/pio/pins
```

#### الدرس المستفاد
الـ `PIN_CONF_PACKED` macro بيـpack الـ config في 32-bit value — الـ lower 8 bits للـ param، والـ upper 24 للـ argument. لما kernel يتحدث وbehavior الـ default يتغير لـ pins حساسة زي DDC أو USB D+/D-، لازم الـ board DTS يكون explicit ومش يعتمد على الـ SoC defaults. `bias-disable` صريحة أحسن من غياب property.

---

### السيناريو 5: GPIO بيـread قيمة غلط على AM62x Custom Board Bring-up

#### العنوان
**الـ GPIO interrupt من external sensor مش بيشتغل صح على AM62x custom board**

#### السياق
فريق bring-up بيشغل custom board بيستخدم Texas Instruments AM62x. الـ board عندها IMU sensor بيتكلم على SPI وبيعمل interrupt على GPIO pin. الـ interrupt بيـfire بس القيمة المقروءة من الـ GPIO دايماً 0 حتى لو الـ IMU بيقول interrupt active.

#### المشكلة
```
imu_irq: unexpected interrupt, GPIO value = 0
imu: failed to read interrupt status, retrying...
```

الـ logic analyzer بيأكد إن الـ INT pin بالفعل بيعمل rising edge.

#### التحليل
المهندس راجع الـ DTS وشاف:

```dts
/* AM62x custom board DTS — المشكلة */
imu_node: imu@0 {
    compatible = "invensense,icm42688";
    reg = <0>;
    spi-max-frequency = <10000000>;
    interrupt-parent = <&gpio0>;
    interrupts = <15 IRQ_TYPE_EDGE_RISING>;
    int-gpios = <&gpio0 15 GPIO_ACTIVE_HIGH>;
};

&main_pmx0 {
    imu_irq_pins: imu-irq-pins {
        pinctrl-single,pins = <
            AM62X_IOPAD(0x003C, PIN_INPUT, 7)  /* GPIO0_15 */
        >;
    };
};
```

الـ `PIN_INPUT` في AM62x = `0x00050000` في raw format. بس الـ issue إن الـ pin مش عندها `input-schmitt-enable`.

لما `pinconf_generic_dt_subnode_to_map()` بتـparse الـ node دي، بتدور على properties زي:
- `bias-pull-up` / `bias-pull-down` → مش موجودين
- `input-schmitt-enable` → مش موجود
- `input-enable` → مفهوم ضمنياً من `PIN_INPUT`

الـ pin بيستخدم standard CMOS input buffer — بدون Schmitt trigger. الـ INT signal من الـ IMU عندها rise time بطيء نسبياً (حوالي 50ns بسبب الـ parasitic capacitance على الـ long trace). الـ CMOS buffer بيشوف الـ slow edge كـ multiple transitions.

الـ config الصح كان لازم يحتوي على:

```c
PIN_CONF_PACKED(PIN_CONFIG_INPUT_SCHMITT_ENABLE, 1)
// lower 8 bits = PIN_CONFIG_INPUT_SCHMITT_ENABLE (14)
// upper 24 bits = 1 (enabled)
// = (1 << 8) | 14 = 0x10E
```

يعني الـ packed value `0x10E` بيتحول:

```c
pinconf_to_config_param(0x10E)   // = 14 = PIN_CONFIG_INPUT_SCHMITT_ENABLE
pinconf_to_config_argument(0x10E) // = 1 → enabled
```

#### الحل

```dts
/* AM62x custom board DTS — بعد الإصلاح */
&main_pmx0 {
    imu_irq_pins: imu-irq-pins {
        pinctrl-single,pins = <
            AM62X_IOPAD(0x003C, PIN_INPUT, 7)
        >;
        /* نضيف generic properties */
        input-schmitt-enable;       /* PIN_CONFIG_INPUT_SCHMITT_ENABLE = 1 */
        bias-pull-down;             /* نمنع الـ float، الـ IMU active-high INT */
    };
};
```

أو باستخدام `pinctrl-single,pins` مع AM62x-specific bits اللي تـinclude الـ Schmitt trigger flag.

للـ debug والـ verification:

```bash
# اتأكد من الـ config المطبق
cat /sys/kernel/debug/pinctrl/main_pmx0/pinconf-pins | grep "0x003c\|GPIO0_15"

# test the pin directly
gpioget gpiochip0 15   # should read 1 when IMU interrupts

# شوف الـ interrupt counter
cat /proc/interrupts | grep imu

# monitor IRQ activity
watch -n 1 "cat /proc/interrupts | grep imu"
```

للـ driver-level debug، الـ `pin_config_item` struct في `pinconf-generic.h` بيُستخدم في الـ debugfs output:

```c
/* من kernel/drivers/pinctrl/pinconf.c */
struct pin_config_item conf_items[] = {
    PCONFDUMP(PIN_CONFIG_INPUT_SCHMITT_ENABLE, "input schmitt enabled",
              NULL, true),
    /* ... */
};
```

الـ `PCONFDUMP` macro:
```c
#define PCONFDUMP(a, b, c, d) PCONFDUMP_WITH_VALUES(a, b, c, d, NULL, 0)
// = { .param=a, .display=b, .format=c, .has_arg=d, .values=NULL, .num_values=0 }
```

ده اللي بيظهر في `/sys/kernel/debug/pinctrl/.../pinconf-pins` — بيساعد في الـ verification.

#### الدرس المستفاد
الـ `PIN_CONFIG_INPUT_SCHMITT_ENABLE` مش luxury — على أي interrupt pin بيتلقى signal من خارج الـ SoC، لازم تـenable الـ Schmitt trigger عشان تتجنب الـ multiple edge detection على الـ slow signals. الـ `pinconf-generic.h` بيوفر الـ enum values دي، والـ DT properties المقابلة (`input-schmitt-enable`) سهلة الاستخدام. الـ bring-up checklist لازم تشمل مراجعة الـ interrupt pins للـ Schmitt trigger config.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأساسي لتاريخ تطوير الـ kernel — الروابط دي كلها حقيقية من نتايج البحث:

| المقال | الأهمية |
|--------|---------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | أول مقال شرح الـ pinctrl subsystem كاملًا — المرجع التأسيسي |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/) | الـ patch الأصلي اللي أضاف الـ `pin_config_param` enum |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | نسخة تانية من الـ interface مع تعديلات |
| [pinctrl/pinconfig: add debug interface](https://lwn.net/Articles/545790/) | إضافة الـ debugfs للـ pinconf — مهم لفهم الـ `PCONFDUMP` macro |
| [pinctrl: common handling of generic pinconfig props in dt](https://lwn.net/Articles/553754/) | إضافة `pinconf_generic_dt_node_to_map` — الوظيفة المركزية في الملف |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | الـ documentation الأصلية للـ subsystem |
| [gpio: add pinctrl based generic gpio driver](https://lwn.net/Articles/946636/) | تكامل الـ pinconf مع الـ GPIO framework |

---

### التوثيق الرسمي في الـ kernel

```bash
# أهم ملف توثيق للـ subsystem
Documentation/driver-api/pin-control.rst

# Device Tree bindings للـ pinctrl-generic
Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt

# مثال على driver يستخدم pinconf-generic
Documentation/devicetree/bindings/pinctrl/qcom,msm8916-pinctrl.yaml
```

**الـ online version:**
- [PINCTRL (PIN CONTROL) subsystem — kernel.org](https://docs.kernel.org/driver-api/pin-control.html)
- [PINCTRL v4.14 — kernel.org](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html)
- [pinctrl.txt — kernel.org](https://www.kernel.org/doc/Documentation/pinctrl.txt)

**قسم مهم جدًا في الـ documentation** مذكور صراحةً في الكود:
```c
/* من pinconf-generic.h سطر 97-98 */
* (Please see Documentation/driver-api/pin-control.rst,
*  section "GPIO mode pitfalls" for a discussion around this parameter.)
```

---

### الـ Source Files الأساسية في الـ kernel

الملفات دي هي اللي بتكمّل صورة الـ `pinconf-generic.h`:

```
include/linux/pinctrl/pinconf-generic.h   ← الملف الحالي
include/linux/pinctrl/pinconf.h           ← الـ ops interface
include/linux/pinctrl/pinctrl.h           ← الـ core registration
include/linux/pinctrl/machine.h           ← الـ pinctrl_map types
drivers/pinctrl/pinconf.c                 ← التنفيذ
drivers/pinctrl/pinconf-generic.c         ← DT parsing implementation
drivers/pinctrl/devicetree.c              ← DT map creation
```

**على GitHub:**
- [include/linux/pinctrl/ — torvalds/linux](https://github.com/torvalds/linux/blob/master/include/linux/pinctrl/pinconf-generic.h)
- [drivers/pinctrl/pinconf.c](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/pinconf.c)
- [Linaro git — linus.walleij/linux-pinctrl](https://git.linaro.org/people/linus.walleij/linux-pinctrl.git)

---

### Commits مهمة

لمتابعة تاريخ الكود:

```bash
# تاريخ الملف كله
git log --follow -- include/linux/pinctrl/pinconf-generic.h

# أول ما اتضاف pinconf-generic
git log --all --oneline --grep="pinconf generic"

# تاريخ الـ pinctrl subsystem من الأول
git log --oneline v3.1..v3.3 -- drivers/pinctrl/
```

**commits بارزة يستحق البحث عنها:**
- `b78d7b4` — إضافة الـ generic pinconfig DT parsing (Heiko Stuebner, 2013)
- الـ initial pinctrl merge في kernel 3.2 بواسطة Linus Walleij
- إضافة `PIN_CONFIG_DRIVE_STRENGTH_UA` و `PIN_CONFIG_INPUT_SCHMITT_UV` في kernels حديثة

**بحث في kernel commit history:**
- [linux-commits-search.typesense.org](https://linux-commits-search.typesense.org/) — ابحث عن `pinconf-generic`

---

### نقاشات الـ Mailing List

**الـ LKML** هو المكان اللي اتطورت فيه الـ API:

- [LKML: dynamically alloc temp array when parsing dt pinconf options](https://lkml.iu.edu/hypermail/linux/kernel/1306.1/03349.html) — نقاش تقني على `pinconf_generic_parse_dt_config`
- [LKML: pinctrl-single bit-per-mux DT flexibility](https://lkml.org/lkml/2026/2/9/44) — نقاش حديث 2026 على الـ pinconf offsets
- [mail-archive: pinctrl debugfs entries](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg581364.html)

**للبحث في الـ archives:**
```
https://lore.kernel.org/linux-gpio/
```
الـ `linux-gpio` mailing list هو الـ list الرسمية للـ pinctrl subsystem حاليًا.

---

### Kernelnewbies.org

مش فيه page مخصصة للـ pinctrl، لكن كل release notes بتوثّق التغييرات:

- [Linux_6.11 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.11) — تحديثات pinctrl drivers
- [Linux_6.8 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.8) — إضافات pinconf جديدة
- [LinuxChanges — kernelnewbies.org](https://kernelnewbies.org/LinuxChanges) — تاريخ التغييرات الكاملة

**ابحث في أي صفحة release** عن كلمة `pinctrl` — هتلاقي drivers جديدة وخصائص pinconf اتضافت.

---

### Elinux.org

- [Pin Control & GPIO update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) — عرض تقديمي من مؤتمر Embedded Linux — ممتاز لفهم العلاقة بين pinctrl وGPIO
- [EBC Device Trees — elinux.org](https://elinux.org/EBC_Device_Trees) — أمثلة عملية على pinctrl في device trees لـ BeagleBone
- [Tests: i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) — مثال على استخدام pinctrl مع I2C multiplexing

---

### كتب مقترحة

#### Linux Device Drivers (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المناسب:** Chapter 9 — Communicating with Hardware (GPIO و pin control concepts)
- **ملاحظة:** الكتاب قديم (2005) وما فيهوش pinctrl بالاسم، لكن المفاهيم الأساسية موجودة
- **متاح مجانًا:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل المناسب:** Chapter 17 — Devices and Modules
- **الأهمية:** بيشرح كيف الـ subsystems بتتسجل وبتتواصل — نفس النمط اللي الـ pinctrl بيتبعه
- **ISBN:** 978-0672329463

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المناسب:** Chapter 15 — Kernel Initialization — وChapter 16 — MTD Subsystem (نفس نمط الـ subsystem registration)
- **الأهمية المباشرة:** الكتاب بيغطي device trees والـ board configuration — اللي الـ pinconf-generic بيخدمه مباشرة
- **ISBN:** 978-0137017836

#### Linux Device Driver Development — John Madieu (2nd Edition, 2022)
- **أحدث كتاب** يغطي الـ pinctrl subsystem بشكل مباشر
- **الفصل:** Chapter 11 — Writing PCI Device Drivers / GPIO and Pin Control
- مقال منه: [linux-device-driver-development: the pin control subsystem — embedded.com](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/)

---

### مصادر إضافية على الويب

- **embedded.com:** [Linux device driver development: The pin control subsystem](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) — شرح عملي ممتاز
- **kernel.org GitHub:** [torvalds/linux — pinctrl/](https://github.com/torvalds/linux/tree/master/drivers/pinctrl)

---

### Search Terms للبحث عن معلومات أكثر

```
# للبحث في Google
"pinconf_generic_dt_node_to_map" site:github.com
"PIN_CONF_PACKED" linux kernel driver example
"pin_config_param" linux kernel pinctrl driver
pinctrl generic pinconf device tree bindings

# للبحث في lore.kernel.org
https://lore.kernel.org/linux-gpio/?q=pinconf-generic

# للبحث في Elixir Cross Referencer
https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/pinconf-generic.h

# للبحث عن drivers بتستخدم الـ API دي
grep -r "pinconf_generic_dt_node_to_map" drivers/pinctrl/
grep -r "PIN_CONF_PACKED" drivers/pinctrl/
```

**الـ Elixir Cross Referencer** هو أفضل أداة للتنقل في الـ kernel source:
- [pinconf-generic.h على Elixir](https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/pinconf-generic.h)
- بيظهر كل الملفات اللي بتستخدم الـ `pin_config_param` enum
## Phase 8: Writing simple module

### الـ Function المختارة: `pinconf_generic_dt_node_to_map`

الـ function دي هي نقطة دخول مهمة في الـ pinctrl subsystem — بتُحوّل DT node لـ mapping table. هنعمل **kprobe** عليها عشان نشوف أي device بيطلبها ومن أي DT node.

---

### الـ Module كامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on pinconf_generic_dt_node_to_map:
 * Traces every call to parse DT pin-config nodes into pinctrl maps.
 *
 * Useful to understand which devices trigger pinconf DT parsing at boot
 * or when a driver is hot-plugged.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* kprobe API */
#include <linux/printk.h>       /* pr_info */
#include <linux/pinctrl/pinctrl.h>   /* struct pinctrl_dev, pinctrl_dev_get_name */
#include <linux/of.h>           /* struct device_node, of_node_full_name */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Docs");
MODULE_DESCRIPTION("Trace pinconf_generic_dt_node_to_map via kprobe");

/* ------------------------------------------------------------------ *
 * kprobe pre-handler — fires just BEFORE the probed function runs    *
 * ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * pinconf_generic_dt_node_to_map signature:
     *   int pinconf_generic_dt_node_to_map(
     *       struct pinctrl_dev   *pctldev,   // arg0
     *       struct device_node   *np_config, // arg1
     *       struct pinctrl_map  **map,        // arg2
     *       unsigned int         *num_maps,   // arg3
     *       enum pinctrl_map_type type);      // arg4
     *
     * On x86-64: args in rdi, rsi, rdx, rcx, r8
     * On ARM64 : args in x0 , x1 , x2 , x3 , x4
     */
#if defined(CONFIG_X86_64)
    struct pinctrl_dev  *pctldev  = (struct pinctrl_dev *)regs->di;
    struct device_node  *np       = (struct device_node *)regs->si;
    int                  map_type = (int)regs->r8;
#elif defined(CONFIG_ARM64)
    struct pinctrl_dev  *pctldev  = (struct pinctrl_dev *)regs->regs[0];
    struct device_node  *np       = (struct device_node *)regs->regs[1];
    int                  map_type = (int)regs->regs[4];
#else
    struct pinctrl_dev  *pctldev  = NULL;
    struct device_node  *np       = NULL;
    int                  map_type = -1;
#endif

    /*
     * pinctrl_dev_get_name() returns the controller's device name string.
     * of_node_full_name() returns the full path of the DT node being parsed.
     * map_type tells us if we're building PIN or GROUP config maps.
     */
    pr_info("pinconf_map: controller=\"%s\" dt_node=\"%s\" map_type=%d\n",
            pctldev  ? pinctrl_dev_get_name(pctldev) : "(null)",
            np       ? of_node_full_name(np)         : "(null)",
            map_type);

    return 0; /* 0 = let the original function execute normally */
}

/* ------------------------------------------------------------------ *
 * kprobe struct — links handler to the target symbol by name          *
 * ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "pinconf_generic_dt_node_to_map",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ *
 * module_init — register the kprobe                                   *
 * ------------------------------------------------------------------ */
static int __init pinconf_trace_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("pinconf_trace: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("pinconf_trace: planted kprobe at %pS\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ *
 * module_exit — unregister the kprobe before module is removed        *
 * ------------------------------------------------------------------ */
static void __exit pinconf_trace_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("pinconf_trace: kprobe removed\n");
}

module_init(pinconf_trace_init);
module_exit(pinconf_trace_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيجيب الـ `kprobe` struct وكل الـ API بتاعتها |
| `linux/pinctrl/pinctrl.h` | بيعرّف `struct pinctrl_dev` و`pinctrl_dev_get_name()` |
| `linux/of.h` | بيعرّف `struct device_node` و`of_node_full_name()` |

الـ includes دي مش اختيارية — من غيرها الـ compiler مش هيعرف الـ types اللي بنستخدمها في الـ handler.

---

#### الـ `handler_pre` — الـ Callback

الـ handler بياخد `struct pt_regs *regs` اللي فيه قيم الـ registers لحظة الاستدعاء؛ منها بنستخرج الـ arguments بالـ ABI conventions بتاعة الـ architecture (x86-64 vs ARM64). ده ضروري لأن الـ kprobe مش بيديك الـ arguments مباشرةً — لازم تجيبهم من الـ registers بنفسك.

الـ `pinctrl_dev_get_name()` بترجع اسم الـ controller زي `"soc:pinctrl@ff634000"` — مهم جداً لمعرفة أي hardware block بيتعامل مع الـ DT node دي. الـ `of_node_full_name()` بترجع المسار الكامل في شجرة الـ device tree زي `"/soc/pinctrl@ff634000/i2c0-pins"`.

---

#### الـ `kprobe` struct

```c
static struct kprobe kp = {
    .symbol_name = "pinconf_generic_dt_node_to_map",
    .pre_handler = handler_pre,
};
```

بنربط الـ handler بالـ symbol بالاسم — الـ kernel بيحوّل الاسم لعنوان وقت الـ register. لو الـ symbol مش exported أو مش موجودة `register_kprobe` هيرجع error.

---

#### الـ `module_init` و`module_exit`

`register_kprobe` بيزرع **breakpoint** في الذاكرة عند بداية الـ function — أي استدعاء ليها هيوقّع الـ handler. `unregister_kprobe` في الـ exit ضروري **قبل** ما الـ module يُشال من الذاكرة؛ من غيره الـ breakpoint هيبقى شايل pointer لكود اتمسح، وده هيعمل kernel panic في أي استدعاء تالٍ.

---

### كيفية التجربة

```bash
# بناء الـ module (Makefile بسيط)
# obj-m += pinconf_trace.o
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod pinconf_trace.ko

# شوف الـ output (بيظهر وقت boot أو لما driver يتسجّل)
sudo dmesg | grep pinconf_map

# إزالة الـ module
sudo rmmod pinconf_trace
```

مثال على output متوقع على جهاز Rockchip:

```
pinconf_trace: planted kprobe at pinconf_generic_dt_node_to_map+0x0
pinconf_map: controller="ff634000.pinctrl" dt_node="/soc/pinctrl@ff634000/i2c0-pins" map_type=3
pinconf_map: controller="ff634000.pinctrl" dt_node="/soc/pinctrl@ff634000/uart2-pins" map_type=3
```

**الـ `map_type=3`** يعني `PIN_MAP_TYPE_CONFIGS_PIN` — الـ DT node بتحدد config لـ pins منفردة مش groups.
