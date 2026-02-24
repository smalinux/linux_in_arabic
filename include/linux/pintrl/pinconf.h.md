## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **PIN CONTROL SUBSYSTEM** في Linux kernel، المسؤول عنه Linus Walleij، والـ mailing list بتاعته هو `linux-gpio@vger.kernel.org`. الـ subsystem ده بيتحكم في كل اللي يخص الـ pins الفيزيائية على الـ SoC أو الـ microcontroller.

---

### الصورة الكبيرة — القصة من الأول

#### المشكلة

تخيل عندك لوحة إلكترونية (مثلاً Raspberry Pi أو هاتف Android). الـ chip الرئيسية (SoC) بيخرج منها مئات الـ **pins** — زي أرجل الحشرة. كل pin ممكن يشتغل في أكتر من role:

- يبعت/يستقبل إشارة GPIO عادية (high/low)
- يبقى جزء من bus زي UART، SPI، I2C
- يبقى signal خاص زي PWM أو clock

بس مش بس كده — كل pin ليه خصائص **كهربية** ممكن تتحكم فيها:

| الخاصية | المعنى العملي |
|---|---|
| **Pull-up / Pull-down** | مقاومة داخلية بتسحب الـ pin لـ VDD أو GND |
| **Drive strength** | قد إيه الـ pin يقدر يوصل تيار (mA) |
| **Open-drain** | الـ pin بيشيل فقط، مش بيدفع — شائع في I2C |
| **Schmitt trigger** | تنقية الإشارة من الضوضاء |
| **Slew rate** | سرعة تغيير الإشارة من low لـ high |
| **Debounce** | تجاهل الاهتزازات السريعة عند قراءة الـ input |

المشكلة القديمة: كل driver كان بيتحكم في الـ pins بنفسه بطريقة عشوائية — chaos كامل. مفيش standard.

#### الحل — الـ pinctrl subsystem

الـ Linux kernel عمل **subsystem موحد** يتوسط بين:

```
+------------------+        +-------------------+        +------------------+
|  Consumer Driver |  --->  |  pinctrl core     |  --->  |  pinctrl driver  |
|  (e.g., SPI drv) |        |  (الوسيط المحايد) |        |  (hardware-specific) |
+------------------+        +-------------------+        +------------------+
```

الـ subsystem اتقسم لـ 3 طبقات:

1. **pinctrl** — إدارة الـ pins نفسها (التعداد والتسمية)
2. **pinmux** — تحديد وظيفة الـ pin (GPIO vs UART vs SPI)
3. **pinconf** — ضبط الخصائص الكهربية للـ pin ← ده بالظبط ملف `pinconf.h`

---

### دور `pinconf.h` تحديداً

الـ file ده بيعرّف **العقد (contract)** اللي لازم أي hardware driver يوفّره لو عايز يدعم **pin configuration** — يعني ضبط الخصائص الكهربية للـ pins.

الـ file بيعرّف struct واحدة فقط: **`struct pinconf_ops`** — وهي جدول function pointers بيقول للـ core "أنا كـ hardware driver، ده اللي أقدر أعمله":

```c
struct pinconf_ops {
#ifdef CONFIG_GENERIC_PINCONF
    bool is_generic;  /* هل بيستخدم الـ generic interface؟ */
#endif

    /* قراءة config pin واحد */
    int (*pin_config_get)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *config);

    /* كتابة config على pin واحد أو أكتر */
    int (*pin_config_set)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *configs,
                          unsigned int num_configs);

    /* قراءة config على group من الـ pins */
    int (*pin_config_group_get)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *config);

    /* كتابة config على group كامل */
    int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *configs,
                                unsigned int num_configs);

    /* debugfs hooks اختيارية */
    void (*pin_config_dbg_show)(...);
    void (*pin_config_group_dbg_show)(...);
    void (*pin_config_config_dbg_show)(...);
};
```

#### ليه Function Pointers؟

لأن كل hardware مختلف. STM32 بيضبط الـ pull-up بطريقة، Qualcomm Snapdragon بطريقة تانية، وRaspberry Pi بطريقة تالتة. الـ `pinconf_ops` بيعمل **abstraction layer** — الـ core بيكلم نفس الـ interface بغض النظر عن الـ hardware.

---

### القصة بالكامل — مثال واقعي

تخيل Android phone فيه SoC من Qualcomm. الـ I2C bus محتاج pin 47 و pin 48 يكونوا:
- وظيفتهم: I2C (مش GPIO)
- كهربياً: open-drain، pull-up 4.7kΩ

**ما يحصل خطوة بخطوة:**

1. الـ Device Tree بيحدد إن الـ I2C driver يحتاج `pinctrl-0 = <&i2c_pins>`
2. لما الـ I2C driver بيعمل `probe`، الـ **pinctrl core** بيشتغل
3. الـ core بيقرأ الـ map، بيشوف إن الـ pins دول محتاجين config
4. بيبعت call لـ `pin_config_set()` في الـ Qualcomm pinctrl driver
5. الـ driver بيكتب في الـ registers الصح على الـ hardware
6. الـ I2C bus يشتغل صح

لو من غير `pinconf`: كل driver كان هيروح يكتب في الـ registers بنفسه → conflicts → kernel panics → chaos.

---

### الـ `unsigned long config` — التفصيلة المهمة

الـ config بيتحمل معلومتين في variable واحد:

```
bits [31:16] = PIN_CONFIG_xxx  (نوع الـ config، زي PIN_CONFIG_BIAS_PULL_UP)
bits [15:0]  = argument        (القيمة، زي 4700 أوم)
```

ده اللي بيتحدد في الـ sibling file: `pinconf-generic.h`.

---

### الفرق بين Generic وCustom

| النوع | الاستخدام |
|---|---|
| **Generic** (`is_generic = true`) | الـ driver يستخدم `pinconf-generic.c` اللي الـ core بيفسره عبر `PIN_CONFIG_*` enums |
| **Custom** | الـ driver بيعرّف format خاص بيه في الـ `config` value |

---

### الملفات المرتبطة — اللي المبرمج لازم يعرفها

#### الـ Core (الأساس)

| الملف | الدور |
|---|---|
| `include/linux/pinctrl/pinconf.h` | **الملف ده** — الـ ops interface |
| `include/linux/pinctrl/pinconf-generic.h` | الـ `PIN_CONFIG_*` enums والـ generic params |
| `include/linux/pinctrl/pinctrl.h` | تعريف `pinctrl_dev`، `pinctrl_pin_desc`، `pingroup` |
| `include/linux/pinctrl/pinmux.h` | الـ `pinmux_ops` — وظيفة الـ pin |
| `include/linux/pinctrl/machine.h` | الـ `pinctrl_map` — الـ mapping بين states والـ pins |
| `include/linux/pinctrl/consumer.h` | الـ API اللي الـ consumer drivers بيستخدموه |
| `include/linux/pinctrl/devinfo.h` | ربط الـ pinctrl بالـ device struct |

#### الـ Core Implementation

| الملف | الدور |
|---|---|
| `drivers/pinctrl/core.c` | الـ core logic — registration، lookup، mapping |
| `drivers/pinctrl/pinconf.c` | تنفيذ الـ pinconf API |
| `drivers/pinctrl/pinconf-generic.c` | الـ generic config parser |
| `drivers/pinctrl/pinmux.c` | تنفيذ الـ pinmux |
| `drivers/pinctrl/devicetree.c` | parse الـ device tree للـ pinctrl |

#### Hardware Drivers (أمثلة)

| الملف | الـ Hardware |
|---|---|
| `drivers/pinctrl/intel/` | Intel SoCs |
| `drivers/pinctrl/nomadik/` | ST-Ericsson Nomadik (المنشأ الأصلي للـ subsystem) |
| `drivers/pinctrl/bcm/` | Broadcom (Raspberry Pi) |
| `drivers/pinctrl/meson/` | Amlogic |
| `drivers/pinctrl/freescale/` | NXP/Freescale |

#### التوثيق

- `Documentation/driver-api/pin-control.rst` — الـ official documentation الكامل
## Phase 2: شرح الـ Pin Configuration (pinconf) Framework

---

### المشكلة اللي بيحلها الـ pinconf

في أي SoC حديث، كل pin جسدي على الـ chip مش بس "ياخد إشارة أو يبعتها" — ده pin كهربائي معقد بيتحكم فيه hardware registers بتضبط خصائصه الكهربائية:

- هل فيه **pull-up** أو **pull-down** resistor داخلي متفعّل؟
- إيه الـ **drive strength** بالـ mA؟
- هل الـ output بيشتغل بـ **push-pull** ولا **open-drain**؟
- هل في **schmitt trigger** على الـ input عشان يمنع noise؟
- إيه الـ **slew rate** (سرعة انتقال الإشارة من low لـ high)؟

المشكلة: كل vendor بيعمل الـ hardware register layout بتاعه بشكل مختلف تماماً.
بدون abstraction layer، كل driver محتاج يعرف التفاصيل الـ hardware-specific لكل SoC — وده كارثة في الـ maintainability.

---

### الحل اللي بيقدمه الـ pinconf

الـ **pinconf subsystem** هو الجزء من الـ pinctrl framework المسؤول عن abstract الـ electrical/hardware configuration للـ pins.

بيعمل ده عن طريق:

1. تعريف **vtable موحد** (`struct pinconf_ops`) — الـ driver بيملّيه بـ function pointers.
2. تعريف **encoding موحد** للـ configuration values — كل config بيتحط في `unsigned long` واحد.
3. دعم نوعين من التعامل: **custom** (كل vendor بيعمل encoding خاص به) وـ **generic** (`CONFIG_GENERIC_PINCONF`) اللي بيستخدم `enum pin_config_param` موحد.

---

### الـ Big Picture Architecture

```
         ┌─────────────────────────────────────────────────────┐
         │              Consumer Side (Device Drivers)         │
         │   SPI driver, I2C driver, UART driver, etc.         │
         │   يطلبوا pin states زي "default", "sleep"           │
         └────────────────────┬────────────────────────────────┘
                              │ pinctrl_select_state()
                              ▼
         ┌─────────────────────────────────────────────────────┐
         │              pinctrl core  (kernel/pinctrl/)        │
         │  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  │
         │  │ pinctrl_ops │  │  pinmux_ops  │  │pinconf_ops│  │
         │  │(grouping/DT)│  │(mux function)│  │(electrical│  │
         │  │             │  │              │  │  config)  │  │
         │  └─────────────┘  └──────────────┘  └─────┬─────┘  │
         └────────────────────────────────────────────┼────────┘
                                                      │
                              ┌───────────────────────┘
                              │ calls pin_config_set() / pin_config_get()
                              ▼
         ┌─────────────────────────────────────────────────────┐
         │         Pin Controller Driver (Provider)            │
         │  مثلاً: pinctrl-bcm2835.c, pinctrl-stm32.c          │
         │  بيطبّق pinconf_ops بتاعه حسب hardware الـ SoC      │
         │                                                     │
         │  pin_config_set() {                                 │
         │      // اكتب في register معين في hardware           │
         │      writel(val, base + PIN_CFG_REG(pin));          │
         │  }                                                  │
         └─────────────────────────────────────────────────────┘
                              │
                              ▼
         ┌─────────────────────────────────────────────────────┐
         │              Physical SoC Hardware                  │
         │   GPIO/Pin Controller registers (pull, drive, etc.) │
         └─────────────────────────────────────────────────────┘
```

---

### مثال حقيقي: Raspberry Pi (BCM2835)

لما بتكتب في Device Tree:

```dts
&uart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart0_pins>;
};

uart0_pins: uart0_pins {
    brcm,pins = <14 15>;
    brcm,function = <4>;        /* ALT0 = UART */
    brcm,pull = <0 2>;          /* pin14: no pull, pin15: pull-up */
};
```

الـ `brcm,pull = <2>` ده pinconf config — بيتحوّل لـ `PIN_CONFIG_BIAS_PULL_UP` عند parsing الـ DT، وبعدين بيتبعت لـ `pin_config_set()` في الـ BCM2835 driver اللي بيكتب في register الـ GPPUD.

---

### الـ Core Abstraction: الـ `unsigned long config`

ده أهم concept في الـ pinconf — **كيف بيتحط الـ configuration في unsigned long واحد؟**

#### الـ Generic Encoding (CONFIG_GENERIC_PINCONF)

```
 bits 31..8                    bits 7..0
┌──────────────────────────────┬────────────┐
│      argument (24 bits)      │  param (8) │
└──────────────────────────────┴────────────┘
```

الـ **param** (lower 8 bits): نوع الـ configuration — من `enum pin_config_param`.
الـ **argument** (upper 24 bits): القيمة — مثلاً مقاومة الـ pull بالـ ohm، أو drive strength بالـ mA.

```c
/* PIN_CONF_PACKED macro من pinconf-generic.h */
#define PIN_CONF_PACKED(p, a) ((a << 8) | ((unsigned long) p & 0xffUL))

/* مثال: pull-up بمقاومة 10K ohm */
unsigned long cfg = PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 10000);

/* استخراج الـ param والـ argument */
enum pin_config_param p = pinconf_to_config_param(cfg);  // PIN_CONFIG_BIAS_PULL_UP
u32 arg = pinconf_to_config_argument(cfg);               // 10000
```

#### الـ Custom Encoding (بدون GENERIC_PINCONF)

الـ driver بيعمل encoding خاص بيه — الـ framework مبيفهمش القيمة، بس بيمررها للـ driver.

---

### شرح تفصيلي لـ `struct pinconf_ops`

ده الـ vtable الأساسي اللي كل pin controller driver لازم يملّيه لو بيدعم pin configuration:

```c
struct pinconf_ops {
#ifdef CONFIG_GENERIC_PINCONF
    bool is_generic;  /* علّم لو الـ driver بيستخدم generic encoding */
#endif

    /* اقرأ config pin معين — config بيتحط في *config */
    int (*pin_config_get)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *config);

    /* اضبط config لـ pin معين — ممكن يكون في configs[] أكتر من config */
    int (*pin_config_set)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *configs,
                          unsigned int num_configs);

    /* نفس الكلام بس للـ group كلها بدل pin واحد */
    int (*pin_config_group_get)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *config);

    int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *configs,
                                unsigned int num_configs);

    /* debugfs hooks — اختيارية */
    void (*pin_config_dbg_show)(...);
    void (*pin_config_group_dbg_show)(...);
    void (*pin_config_config_dbg_show)(...);
};
```

**ليه `num_configs` وlist؟** لأن pin واحد ممكن يحتاج أكتر من config في وقت واحد — مثلاً pull-up + drive strength + schmitt trigger في نفس الوقت. الـ framework بيبعتهم كـ array دفعة واحدة وبيترك للـ driver يطبّقهم.

---

### الـ Generic Parameters (`enum pin_config_param`)

الـ `pinconf-generic.h` بيعرّف كل الـ configurations المحتملة:

| Category | Parameters |
|---|---|
| **Bias** | `PIN_CONFIG_BIAS_DISABLE`, `PIN_CONFIG_BIAS_PULL_UP`, `PIN_CONFIG_BIAS_PULL_DOWN`, `PIN_CONFIG_BIAS_HIGH_IMPEDANCE`, `PIN_CONFIG_BIAS_BUS_HOLD` |
| **Drive** | `PIN_CONFIG_DRIVE_PUSH_PULL`, `PIN_CONFIG_DRIVE_OPEN_DRAIN`, `PIN_CONFIG_DRIVE_OPEN_SOURCE`, `PIN_CONFIG_DRIVE_STRENGTH` (mA), `PIN_CONFIG_DRIVE_STRENGTH_UA` (µA) |
| **Input** | `PIN_CONFIG_INPUT_ENABLE`, `PIN_CONFIG_INPUT_DEBOUNCE` (µs), `PIN_CONFIG_INPUT_SCHMITT`, `PIN_CONFIG_INPUT_SCHMITT_ENABLE` |
| **Output** | `PIN_CONFIG_OUTPUT_ENABLE`, `PIN_CONFIG_OUTPUT_IMPEDANCE_OHMS`, `PIN_CONFIG_LEVEL` |
| **Timing** | `PIN_CONFIG_SLEW_RATE`, `PIN_CONFIG_SKEW_DELAY`, `PIN_CONFIG_SKEW_DELAY_INPUT_PS`, `PIN_CONFIG_SKEW_DELAY_OUTPUT_PS` |
| **Power** | `PIN_CONFIG_MODE_LOW_POWER`, `PIN_CONFIG_POWER_SOURCE`, `PIN_CONFIG_PERSIST_STATE` |
| **Misc** | `PIN_CONFIG_MODE_PWM`, `PIN_CONFIG_SLEEP_HARDWARE_STATE`, `PIN_CONFIG_BIAS_PULL_PIN_DEFAULT` |

الـ range من 0 لـ `PIN_CONFIG_END = 0x7F` محجوز للـ standard params.
أي driver محتاج params مخصوصة ليه يبدأ من `PIN_CONFIG_END + 1` لـ `PIN_CONFIG_MAX = 0xFF`.

---

### كيف تتربط الـ structs ببعض

```
struct pinctrl_desc
├── const struct pinconf_ops  *confops    ← pointer للـ vtable
├── const struct pinctrl_ops  *pctlops   ← grouping + DT parsing
├── const struct pinmux_ops   *pmxops    ← mux function selection
├── const struct pinctrl_pin_desc *pins  ← array of all pins
│       ├── .number  (unique pin ID)
│       ├── .name    ("GPIO0", "PA1", etc.)
│       └── .drv_data (driver-specific per-pin data)
│
│   [Generic PINCONF only]
├── const struct pinconf_generic_params *custom_params
│       ├── .property    (DT property name: "drive-strength")
│       ├── .param       (PIN_CONFIG_DRIVE_STRENGTH)
│       └── .default_value
└── const struct pin_config_item *custom_conf_items  ← debugfs display
        ├── .param
        ├── .display     ("drive strength")
        └── .format      ("mA")
```

عند registration:

```
pinctrl_register_and_init(desc, dev, drv_data, &pctldev)
         │
         ▼
    struct pinctrl_dev  (internal, managed by core)
         ├── desc  → pointer للـ pinctrl_desc
         ├── list of states (default, sleep, idle, ...)
         └── list of pinctrl_map entries
```

---

### الـ pinconf_ops في سياق الـ pinctrl_map

الـ **pinctrl_map** (من `machine.h`) هي الـ "وصفة" اللي بتربط device باسمها بـ pin configuration:

```c
/* مثال من board file أو DT parsed output */
static unsigned long uart_pin_cfgs[] = {
    PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 1),
    PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 8),  /* 8mA */
};

static struct pinctrl_map board_map[] = {
    PIN_MAP_CONFIGS_PIN("uart0", PINCTRL_STATE_DEFAULT,
                        "pinctrl-bcm2835", "GPIO14", uart_pin_cfgs),
};
```

الـ flow:

```
device probe
    └─► pinctrl_get() + pinctrl_lookup_state("default")
             └─► pinctrl_select_state()
                      └─► يمشي على كل entry في الـ state
                               └─► للـ PIN_MAP_TYPE_CONFIGS_PIN:
                                        └─► pinconf_apply_setting()
                                                 └─► confops->pin_config_set()
                                                          └─► hardware write
```

---

### تشابه حقيقي: نظام ضبط الميكروفون في استوديو

تخيّل استوديو تسجيل — فيه engineer بيتحكم في ضبط كل ميكروفون:

| مفهوم الاستوديو | مفهوم الـ kernel |
|---|---|
| كل ميكروفون على الـ mixing board | كل pin على الـ SoC |
| إعدادات gain, EQ, compression | إعدادات pull, drive, slew |
| الـ mixing board hardware | الـ pin controller hardware registers |
| صفحة الـ presets "صوت vocals" | الـ `pinctrl_map` لـ state معينة |
| الـ channel strip UI (slider + knobs) | الـ `pinconf_ops` vtable |
| مهندس الصوت اللي بيضبط القيم | الـ pinconf core |
| الـ fader الـ analog المختلف لكل ماركة console | الـ hardware-specific register encoding |
| `num_configs` array | تعديل أكتر من knob على نفس الـ channel في وقت واحد |
| debugfs `pin_config_dbg_show` | الـ VU meter اللي بيعرض القيمة الحالية |

النقطة المهمة: الـ engineer (pinconf core) مش بيعرف التفاصيل الـ hardware لكل mixing console — هو بيشتغل على abstraction layer واضح، والـ console نفسه (الـ driver) هو اللي بيتعامل مع الـ hardware الفعلي.

---

### الفرق بين Custom وGeneric pinconf

#### Custom pinconf
الـ driver بيعمل encoding خاص بيه في الـ `unsigned long` — مش مقيّد بـ `enum pin_config_param`. الـ core بيمرر القيمة blindly للـ `pin_config_set()`.

```c
/* مثال: driver بيستخدم top 16 bits للـ register offset
   والـ bottom 16 bits للـ register value */
unsigned long cfg = (REG_PULL_CTRL << 16) | PULL_UP_VALUE;
```

#### Generic pinconf (`is_generic = true`)
الـ driver بيستخدم `enum pin_config_param` + argument encoding الموحد. الـ framework عنده code جاهز لـ:
- **DT parsing**: بيحوّل `bias-pull-up` في الـ DTS لـ `PIN_CONFIG_BIAS_PULL_UP` أوتوماتيكياً عبر `pinconf_generic_dt_node_to_map()`.
- **debugfs display**: بيعرض human-readable names.
- **custom extensions**: الـ driver يقدر يضيف params خاصة فوق الـ standard params عبر `custom_params` في الـ `pinctrl_desc`.

---

### الـ pinconf — مالكه وما بيفوّضه

**الـ pinconf core يمتلك:**
- الـ API العام (`pin_config_get()`, `pin_config_set()` في `drivers/pinctrl/pinconf.c`).
- التحقق من صحة الـ calls وتمريرها للـ driver.
- الـ generic DT parsing للـ standard properties.
- الـ debugfs infrastructure.
- الـ state machine اللي بيطبّق الـ configs عند transition بين states.

**بيفوّض للـ driver:**
- الـ hardware register writes.
- التحقق من إن الـ config مدعوم على hardware معين (`-ENOTSUPP` لو لأ).
- التحقق من إن القيمة valid لهذا الـ hardware (`-EINVAL` لو الـ config مدعوم بس القيمة غلط).
- الـ encoding الـ custom للـ `unsigned long` لو مش بيستخدم generic.
- إدارة الـ per-pin hardware state.

---

### ملاحظة على الـ Subsystems المترابطة

الـ **pinconf** جزء من الـ **pinctrl subsystem** الأكبر اللي فيه 3 أوجه:

1. **pinctrl_ops** — grouping الـ pins وقراءة الـ DT (الـ "ما هي الـ pins دي؟").
2. **pinmux_ops** — تحديد الـ function لكل group (الـ "على أي peripheral بتتوصل؟").
3. **pinconf_ops** — الـ electrical configuration (الـ "بتشتغل إزاي كهربائياً؟").

الـ **GPIO subsystem** (`gpio_chip`) بيتداخل مع الـ pinctrl في موضوع الـ `pinctrl_gpio_range` — لكنه subsystem منفصل.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### جدول الـ Enum: `pin_config_param` — Cheatsheet

| القيمة | الوصف | الـ Argument |
|--------|--------|--------------|
| `PIN_CONFIG_BIAS_BUS_HOLD` | bus keeper — يحافظ على آخر قيمة على الـ bus | يُتجاهل |
| `PIN_CONFIG_BIAS_DISABLE` | يعطّل أي bias على الـ pin | يُتجاهل |
| `PIN_CONFIG_BIAS_HIGH_IMPEDANCE` | وضع high-Z / tristate | يُتجاهل |
| `PIN_CONFIG_BIAS_PULL_DOWN` | pull-down للـ GND | أوم أو custom (0 = تعطيل) |
| `PIN_CONFIG_BIAS_PULL_PIN_DEFAULT` | pull حسب الـ hardware default | 0 = ignore، != 0 = فعّل |
| `PIN_CONFIG_BIAS_PULL_UP` | pull-up للـ VDD | أوم أو custom |
| `PIN_CONFIG_DRIVE_OPEN_DRAIN` | open-drain output | يُتجاهل |
| `PIN_CONFIG_DRIVE_OPEN_SOURCE` | open-source (open-emitter) output | يُتجاهل |
| `PIN_CONFIG_DRIVE_PUSH_PULL` | push-pull output (الوضع الافتراضي) | يُتجاهل |
| `PIN_CONFIG_DRIVE_STRENGTH` | الحد الأقصى للتيار (mA) | mA |
| `PIN_CONFIG_DRIVE_STRENGTH_UA` | الحد الأقصى للتيار (µA) | µA |
| `PIN_CONFIG_INPUT_DEBOUNCE` | debounce للإشارة عند القراءة | µs (0 = تعطيل) |
| `PIN_CONFIG_INPUT_ENABLE` | تفعيل / تعطيل input buffer | 1=تفعيل، 0=تعطيل |
| `PIN_CONFIG_INPUT_SCHMITT` | schmitt-trigger مع threshold مخصص | custom threshold |
| `PIN_CONFIG_INPUT_SCHMITT_ENABLE` | تفعيل / تعطيل schmitt-trigger | 1=تفعيل، 0=تعطيل |
| `PIN_CONFIG_INPUT_SCHMITT_UV` | schmitt-trigger مع threshold بـ µV | µV |
| `PIN_CONFIG_MODE_LOW_POWER` | وضع low-power | 1=تفعيل، 0=تعطيل |
| `PIN_CONFIG_MODE_PWM` | تهيئة الـ pin لـ PWM | custom |
| `PIN_CONFIG_LEVEL` | قيمة output أو قراءة قيمة الخط | 1=high، 0=low |
| `PIN_CONFIG_OUTPUT_ENABLE` | تفعيل output buffer بدون قيمة محددة | 1=تفعيل، 0=تعطيل |
| `PIN_CONFIG_OUTPUT_IMPEDANCE_OHMS` | impedance عند الـ output | أوم |
| `PIN_CONFIG_PERSIST_STATE` | الحفاظ على حالة الـ pin بعد sleep أو reset | يُتجاهل |
| `PIN_CONFIG_POWER_SOURCE` | اختيار مصدر الطاقة | custom |
| `PIN_CONFIG_SKEW_DELAY` | clock skew أو latch delay (مشترك) | custom |
| `PIN_CONFIG_SKEW_DELAY_INPUT_PS` | clock skew فقط عند الـ input | ps |
| `PIN_CONFIG_SKEW_DELAY_OUTPUT_PS` | latch delay فقط عند الـ output | ps |
| `PIN_CONFIG_SLEEP_HARDWARE_STATE` | يشير أن هذا config خاص بحالة النوم | يُتجاهل |
| `PIN_CONFIG_SLEW_RATE` | slew rate للـ pin | custom |
| `PIN_CONFIG_END` | آخر قيمة للـ standard configs | `= 0x7F` |
| `PIN_CONFIG_MAX` | الحد الأقصى للقيم في الـ packed format | `= 0xFF` |

**ملاحظة على الـ Custom params:** أي driver يريد parameters إضافية يبدأ من `PIN_CONFIG_END + 1` وصولاً إلى `PIN_CONFIG_MAX`.

---

### جدول الـ Config Options المهمة

| الـ Kconfig | الأثر |
|-------------|--------|
| `CONFIG_GENERIC_PINCONF` | يُفعّل الـ `is_generic` flag داخل `pinconf_ops`، ويتيح `num_custom_params`، `custom_params`، `custom_conf_items` في `pinctrl_desc` |

---

### الـ Packed Config Format — Cheatsheet

```
unsigned long config:
 ┌────────────────────────────┬───────────┐
 │  bits [31:8] — argument    │ bits[7:0] │
 │  (24 bits, قيمة الـ param) │   param   │
 └────────────────────────────┴───────────┘
```

| الـ Macro / Inline | الوظيفة |
|-------------------|---------|
| `PIN_CONF_PACKED(p, a)` | يجمع param + argument في `unsigned long` واحد |
| `pinconf_to_config_param(config)` | يستخرج الـ param من الـ config |
| `pinconf_to_config_argument(config)` | يستخرج الـ argument من الـ config |
| `pinconf_to_config_packed(param, arg)` | يبني الـ config من param + argument |

---

### الـ Structs المهمة

#### 1. `struct pinconf_ops`

**الغرض:** جدول عمليات الـ pin configuration — الـ driver يملأه، والـ pinctrl core يستدعيه.

```c
struct pinconf_ops {
#ifdef CONFIG_GENERIC_PINCONF
    bool is_generic;          // هل الـ driver يستخدم generic pinconf؟
#endif
    // قراءة config لـ pin محدد
    int (*pin_config_get)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *config);

    // كتابة configs لـ pin محدد (يقبل مصفوفة)
    int (*pin_config_set)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *configs,
                          unsigned int num_configs);

    // قراءة config لـ group كامل
    int (*pin_config_group_get)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *config);

    // كتابة configs لـ group كامل
    int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *configs,
                                unsigned int num_configs);

    // debugfs hooks — اختيارية
    void (*pin_config_dbg_show)(...);
    void (*pin_config_group_dbg_show)(...);
    void (*pin_config_config_dbg_show)(...);
};
```

**الارتباطات:**
- **الـ `pinctrl_dev`** هو الـ handle الذي يمثل الـ pin controller device، يُمرَّر لكل callback.
- **الـ `pinctrl_desc`** يحمل pointer لـ `pinconf_ops` في field اسمه `confops`.
- الـ core يصل لـ ops من خلال: `pctldev->desc->confops->pin_config_set(...)`.

---

#### 2. `struct pinctrl_desc`

**الغرض:** الـ descriptor الرئيسي للـ pin controller — يُسجَّل مع الـ pinctrl core عند الـ probe.

```c
struct pinctrl_desc {
    const char *name;                         // اسم الـ controller
    const struct pinctrl_pin_desc *pins;      // مصفوفة وصف الـ pins
    unsigned int npins;                       // عدد الـ pins
    const struct pinctrl_ops *pctlops;        // عمليات الـ grouping و DT mapping
    const struct pinmux_ops *pmxops;          // عمليات الـ muxing
    const struct pinconf_ops *confops;        // عمليات الـ configuration ← محور ملفنا
    struct module *owner;                     // الـ module المالك
#ifdef CONFIG_GENERIC_PINCONF
    unsigned int num_custom_params;           // عدد الـ custom DT params
    const struct pinconf_generic_params *custom_params; // تعريفات الـ custom params
    const struct pin_config_item *custom_conf_items;    // عرضها في debugfs
#endif
    bool link_consumers;                      // device link مع الـ consumers
};
```

---

#### 3. `struct pin_config_item`

**الغرض:** يصف كيفية عرض param معين في الـ debugfs.

```c
struct pin_config_item {
    const enum pin_config_param param;   // أي param يصفه
    const char * const display;          // اسم للعرض في debugfs
    const char * const format;           // format string للـ argument
    bool has_arg;                        // هل للـ param argument؟
    const char * const *values;          // مصفوفة قيم نصية اختيارية
    size_t num_values;
};
```

---

#### 4. `struct pinconf_generic_params`

**الغرض:** يربط خاصية DT نصية (مثل `"bias-pull-up"`) بـ `pin_config_param` وقيمة افتراضية.

```c
struct pinconf_generic_params {
    const char * const property;     // اسم الـ DT property
    enum pin_config_param param;     // الـ enum المقابل
    u32 default_value;               // القيمة الافتراضية إذا لم يُحدَّد argument
    const char * const *values;      // قيم نصية إضافية
    size_t num_values;
};
```

---

#### 5. `struct pinctrl_pin_desc`

**الغرض:** وصف pin واحد — الـ number والاسم وبيانات خاصة بالـ driver.

```c
struct pinctrl_pin_desc {
    unsigned int number;   // رقم فريد في الـ global pin space
    const char *name;      // اسم للـ debugging
    void *drv_data;        // بيانات خاصة بالـ driver، الـ core لا يلمسها
};
```

---

#### 6. `struct pingroup`

**الغرض:** يجمع مجموعة pins تحت اسم واحد، يُستخدم في الـ mux والـ config معاً.

```c
struct pingroup {
    const char *name;             // اسم الـ group
    const unsigned int *pins;     // مصفوفة أرقام الـ pins
    size_t npins;                 // عدد الـ pins
};
```

---

### مخطط علاقات الـ Structs

```
                    ┌─────────────────────┐
                    │   pinctrl_desc      │
                    │─────────────────────│
                    │ name                │
                    │ pins ──────────────►│ pinctrl_pin_desc[]
                    │ npins               │   { number, name, drv_data }
                    │ pctlops ───────────►│ pinctrl_ops
                    │ pmxops ────────────►│ pinmux_ops
                    │ confops ───────────►│ pinconf_ops  ◄── محور الملف
                    │ owner               │
                    │ custom_params ─────►│ pinconf_generic_params[]
                    │ custom_conf_items ─►│ pin_config_item[]
                    └──────────┬──────────┘
                               │ pinctrl_register_and_init()
                               ▼
                    ┌─────────────────────┐
                    │   pinctrl_dev       │ (داخلي في الـ core)
                    │─────────────────────│
                    │ desc ──────────────►│ pinctrl_desc (أعلاه)
                    │ driver_data         │ (بيانات الـ driver)
                    │ dev ───────────────►│ struct device
                    │ gpio_ranges (list)  │
                    └─────────────────────┘
                               │
                               │ يُمرَّر لكل callback في
                               ▼
                    ┌─────────────────────┐
                    │   pinconf_ops       │
                    │─────────────────────│
                    │ is_generic (bool)   │
                    │ pin_config_get()    │──► يقرأ unsigned long config
                    │ pin_config_set()    │──► يكتب unsigned long configs[]
                    │ pin_config_group_get│
                    │ pin_config_group_set│
                    │ *_dbg_show()        │──► seq_file (debugfs)
                    └─────────────────────┘

unsigned long config (packed format):
    ┌──────────────────────────┬──────────┐
    │ argument [bits 31:8]     │ param    │
    │ (24 bits)                │ [7:0]    │
    └──────────────────────────┴──────────┘
         ▲                          ▲
         │                          │
    pinconf_to_config_argument()  pinconf_to_config_param()
```

---

### مخطط دورة حياة الـ Pin Config

```
1. DRIVER PROBE
   │
   ▼
   يعرّف الـ driver:
   ┌─────────────────────────────────────┐
   │ static const struct pinconf_ops my_conf_ops = {   │
   │     .is_generic = true,                           │
   │     .pin_config_get  = my_pin_config_get,         │
   │     .pin_config_set  = my_pin_config_set,         │
   │     .pin_config_group_get = my_group_config_get,  │
   │     .pin_config_group_set = my_group_config_set,  │
   │ };                                                │
   └─────────────────────────────────────┘
   │
   ▼
   يبني pinctrl_desc مع confops = &my_conf_ops
   │
   ▼
   pinctrl_register_and_init(desc, dev, drv_data, &pctldev)
   │
   ▼
   الـ core يخزّن desc داخل pinctrl_dev
   pinctrl_enable(pctldev)  ←── يجعله متاحاً للـ consumers

2. CONSUMER يطلب CONFIG
   │
   ▼
   consumer driver:
   pinctrl_select_state(p, state)
   │
   ▼
   pinctrl core يقرأ الـ pinctrl_map للـ state
   │
   ▼
   يستدعي pinconf_apply_setting()
   │
   ├─► للـ pin منفرد:
   │       pctldev->desc->confops->pin_config_set(
   │           pctldev, pin_num, configs[], num_configs)
   │
   └─► للـ group:
           pctldev->desc->confops->pin_config_group_set(
               pctldev, group_selector, configs[], num_configs)

3. DEBUGFS
   │
   ▼
   pinconf_show_config() تستدعي:
       confops->pin_config_dbg_show(pctldev, seq_file, pin)
       confops->pin_config_group_dbg_show(pctldev, seq_file, group)
       confops->pin_config_config_dbg_show(pctldev, seq_file, config)

4. TEARDOWN
   │
   ▼
   pinctrl_unregister(pctldev)
   الـ core يُزيل الـ pctldev من الـ global list
   الـ pinconf_ops لا تحتاج cleanup خاص (static structs في الـ driver)
```

---

### مخطط Call Flow — تطبيق config من الـ DT

```
Device Tree:
    pinctrl-0 = <&uart0_pins>;
    uart0_pins: uart0-pins {
        bias-pull-up;
        drive-strength = <4>;
    };

Consumer driver calls:
    devm_pinctrl_get_select_default(dev)
        │
        ▼
    pinctrl_select_state(p, default_state)
        │
        ▼
    pinctrl_commit_state()
        │
        ├─► للـ PIN_MAP_TYPE_CONFIGS_PIN:
        │       pin_config_set(pctldev, pin, configs[], n)
        │           │
        │           ▼
        │       [generic path إذا is_generic=true]
        │       pinconf_generic_validate_config()
        │           │
        │           ▼
        │       confops->pin_config_set(pctldev, pin, configs, n)
        │           │
        │           ▼
        │       [driver impl] my_pin_config_set():
        │           pinconf_to_config_param(config)  → PIN_CONFIG_BIAS_PULL_UP
        │           pinconf_to_config_argument(config) → value
        │           ← يكتب الـ hardware registers
        │
        └─► للـ PIN_MAP_TYPE_CONFIGS_GROUP:
                pin_config_group_set(pctldev, selector, configs[], n)
                    │
                    ▼
                confops->pin_config_group_set(...)
                    ← يطبّق على كل pins في الـ group
```

---

### مخطط Call Flow — Generic DT Parsing

```
DT node بـ properties مثل "bias-pull-up", "drive-strength"
    │
    ▼
pinconf_generic_dt_node_to_map(pctldev, np, &map, &num_maps, type)
    │
    ├─► يبحث عن كل property مذكورة في pinconf_generic_params[]
    │   (أو custom_params في الـ pinctrl_desc)
    │
    ├─► لكل property:
    │       يبني config = PIN_CONF_PACKED(param, value)
    │       يضيفه للـ pinctrl_map
    │
    └─► يُعيد map[] للـ core

لاحقاً عند تطبيق الـ state:
    core يمشي على map[] ويستدعي pin_config_set / pin_config_group_set
```

---

### استراتيجية الـ Locking

الـ `pinconf.h` نفسه لا يُعرّف locks — الـ locking مسؤولية الـ pinctrl core (`pinctrl_dev`).

| الـ Lock | يوجد في | يحمي |
|---------|---------|------|
| `pctldev->mutex` | `struct pinctrl_dev` (داخلي) | كل العمليات على الـ pin controller (get/set config, select state) |
| `pinctrldev_list_mutex` | `pinctrl.c` (global) | الـ global list of all registered pin controllers |

**ترتيب الـ Locking:**
```
pinctrldev_list_mutex  (global)
    └─► pctldev->mutex (per-controller)
            └─► [hardware access داخل الـ driver]
```

**قواعد مهمة:**
- الـ callbacks في `pinconf_ops` تُستدعى دائماً وهو حامل `pctldev->mutex`.
- الـ driver لا يحتاج يأخذ lock إضافي ما لم يكن عنده state داخلي خاص به.
- الـ `pin_config_dbg_show` callbacks ممكن تتعامل مع seq_file بأمان لأن debugfs لها locking خاص بها.

---

### ملخص العلاقة بين الـ Structs (جدول)

| الـ Struct | يرتبط بـ | طريقة الارتباط |
|-----------|----------|---------------|
| `pinconf_ops` | `pinctrl_desc` | `confops` pointer |
| `pinconf_ops` | `pinctrl_dev` | عبر `pctldev->desc->confops` |
| `pinconf_ops` | `pinctrl_dev` (arg) | كل callback يستقبل `pctldev` |
| `pin_config_item` | `pinctrl_desc` | `custom_conf_items[]` |
| `pinconf_generic_params` | `pinctrl_desc` | `custom_params[]` |
| `pinconf_generic_params` | `pin_config_item` | نفس الـ `param` enum يربطهم |
| `pinctrl_pin_desc` | `pinctrl_desc` | `pins[]` |
| `pingroup` | `pinctrl_ops` | الـ `get_group_pins` يُعيد pins الـ group |
| `unsigned long config` | `pinconf_ops` callbacks | packed format (param + argument) |
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

الـ `pinconf.h` بيعرّف **vtable واحدة** بس هي `struct pinconf_ops` — مفيش functions حرة فيه، كل حاجة function pointers جوه الـ struct. الـ framework بيبقى هو اللي بيكال الـ callbacks دي.

| الـ Callback | الـ Direction | الـ Scope |
|---|---|---|
| `pin_config_get` | read | single pin |
| `pin_config_set` | write | single pin |
| `pin_config_group_get` | read | pin group |
| `pin_config_group_set` | write | pin group |
| `pin_config_dbg_show` | debug output | single pin |
| `pin_config_group_dbg_show` | debug output | pin group |
| `pin_config_config_dbg_show` | debug output | raw config value |

---

### الـ struct pinconf_ops — نظرة عامة

```c
struct pinconf_ops {
#ifdef CONFIG_GENERIC_PINCONF
    bool is_generic;
#endif
    int (*pin_config_get)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *config);

    int (*pin_config_set)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *configs,
                          unsigned int num_configs);

    int (*pin_config_group_get)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *config);

    int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *configs,
                                unsigned int num_configs);

    void (*pin_config_dbg_show)(struct pinctrl_dev *pctldev,
                                struct seq_file *s,
                                unsigned int offset);

    void (*pin_config_group_dbg_show)(struct pinctrl_dev *pctldev,
                                      struct seq_file *s,
                                      unsigned int selector);

    void (*pin_config_config_dbg_show)(struct pinctrl_dev *pctldev,
                                       struct seq_file *s,
                                       unsigned long config);
};
```

الـ `struct pinconf_ops` هي الـ **vtable** اللي بيحددها كل pin controller driver عشان يدعم pin configuration. الـ pinctrl core بيشوف الـ `confops` pointer في `struct pinctrl_desc` — لو `NULL` يبقى الـ driver مش بيدعم pin config خالص. الـ driver بيخزن pointer لـ instance من الـ struct دي في الـ `pinctrl_desc.confops`.

---

### تصنيف الـ Callbacks

#### Category 1: Runtime Read/Write Operations

دي الـ callbacks الأساسية اللي بتشتغل في وقت التشغيل — الـ framework والـ consumer drivers بيكالوها.

---

#### `pin_config_get`

```c
int (*pin_config_get)(struct pinctrl_dev *pctldev,
                      unsigned int pin,
                      unsigned long *config);
```

بتجيب الـ configuration الحالية لـ pin معين. الـ `config` بيتبعت كـ in/out — الـ caller بيحط فيه الـ parameter ID في الـ upper bits (لو generic pinconf)، والـ driver بيرجع القيمة في الـ lower bits.

**الـ Parameters:**
- `pctldev` — الـ pin controller device handle اللي رجع من `pinctrl_register_and_init()`
- `pin` — الـ global pin number من الـ pin number space بتاع الـ controller
- `config` — pointer لـ `unsigned long` بيحمل الـ parameter type في الـ upper bits والـ driver بيكتب القيمة فيه

**الـ Return:**
- `0` — نجح وجاب الـ config
- `-ENOTSUPP` — الـ parameter ده مش مدعوم على الـ controller ده خالص
- `-EINVAL` — الـ parameter مدعوم بس مش enabled/configured حالياً

**Key Details:**
- الفرق بين `-ENOTSUPP` و`-EINVAL` مهم جداً: الأول يعني hardware capability غير موجودة، التاني يعني موجودة بس off.
- في حالة `CONFIG_GENERIC_PINCONF`، الـ framework بيستخدم `FIELD_GET`/`FIELD_PREP` على الـ `config` value عشان يفصل الـ parameter ID عن الـ value.
- الـ locking: الـ pinctrl core عادةً بيمسك الـ `pctldev->mutex` قبل ما يكال الـ callback.

**Who calls it:**
- `pinconf_get_config()` في `drivers/pinctrl/pinconf.c`
- الـ debugfs infrastructure لما تعمل `cat` على `/sys/kernel/debug/pinctrl/*/pins`

---

#### `pin_config_set`

```c
int (*pin_config_set)(struct pinctrl_dev *pctldev,
                      unsigned int pin,
                      unsigned long *configs,
                      unsigned int num_configs);
```

بتطبق array من الـ configurations على pin واحد. الـ driver بيلف على الـ `configs[]` ويطبق كل واحدة على حدة في sequence. لو أي config fail، المفروض يرجع error فوراً.

**الـ Parameters:**
- `pctldev` — الـ pin controller device handle
- `pin` — الـ global pin number
- `configs` — array من `unsigned long` values، كل element بتحمل (parameter_id | value) مع بعض
- `num_configs` — عدد العناصر في الـ `configs` array

**الـ Return:**
- `0` — كل الـ configs اتطبقت تمام
- negative error code — فشل في تطبيق واحدة أو أكتر

**Key Details:**
- الـ driver لازم يعمل loop على `num_configs` — الـ framework ممكن يبعت أكتر من config في call واحدة (e.g., bias-pull-up + drive-strength مع بعض).
- لو الـ driver يدعم generic pinconf (`is_generic = true`)، الـ framework بيكال `pinconf_generic_parse_dt_config()` الأول عشان يحول الـ DT properties لـ `configs[]` array، وبعدين يكال الـ callback ده.
- لا يجب عمل sleep جوا الـ callback لو الـ caller في atomic context — بعض الـ platforms بتطلبها من IRQ context.

**Pseudocode Flow (driver side):**

```c
int my_pin_config_set(struct pinctrl_dev *pctldev, unsigned int pin,
                      unsigned long *configs, unsigned int num_configs)
{
    struct my_pctrl *pctrl = pinctrl_dev_get_drvdata(pctldev);
    int i, ret;

    for (i = 0; i < num_configs; i++) {
        u32 param = pinconf_to_config_param(configs[i]);
        u32 arg   = pinconf_to_config_argument(configs[i]);

        switch (param) {
        case PIN_CONFIG_BIAS_PULL_UP:
            ret = my_set_pull(pctrl, pin, PULL_UP);
            break;
        case PIN_CONFIG_DRIVE_STRENGTH:
            ret = my_set_drive(pctrl, pin, arg);
            break;
        default:
            ret = -ENOTSUPP;
        }

        if (ret)
            return ret;
    }
    return 0;
}
```

**Who calls it:**
- `pinconf_apply_config()` في الـ pinctrl core
- `pinctrl_commit_state()` لما يتطبق pinctrl state جديد

---

#### `pin_config_group_get`

```c
int (*pin_config_group_get)(struct pinctrl_dev *pctldev,
                            unsigned int selector,
                            unsigned long *config);
```

نفس `pin_config_get` بالظبط لكن على مستوى الـ **pin group** مش pin فردي. الـ `selector` هنا هو الـ group index مش pin number. المفيد لما كل pins الـ group بتشارك نفس الـ config (e.g., shared bias).

**الـ Parameters:**
- `pctldev` — الـ pin controller handle
- `selector` — الـ group selector index (نفسه اللي بيرجع من `get_groups_count`/`get_group_name`)
- `config` — in/out: parameter type + returned value

**الـ Return:**
- نفس قواعد `pin_config_get`: `0`, `-ENOTSUPP`, `-EINVAL`

**Key Details:**
- الـ framework مش بيكال الـ callback ده لو الـ driver `NULL`ها — بيfall back لـ `pin_config_get` على كل pin منفرد في الـ group.
- التطبيق الشائع: بيجيب الـ config من أول pin في الـ group وبيرجعها كـ representative للـ group كلها.

**Who calls it:**
- `pinconf_get_config()` لما الـ caller بيطلب config على group مش pin

---

#### `pin_config_group_set`

```c
int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                            unsigned int selector,
                            unsigned long *configs,
                            unsigned int num_configs);
```

بتطبق array من الـ configurations على **كل pins** في group واحدة. أكفأ من استدعاء `pin_config_set` لكل pin على حدة لأن الـ hardware ممكن يعمل atomic write لكل الـ group.

**الـ Parameters:**
- `pctldev` — الـ pin controller handle
- `selector` — الـ group selector index
- `configs` — array من configs زي `pin_config_set`
- `num_configs` — عدد الـ configs

**الـ Return:**
- `0` أو negative error

**Key Details:**
- لو الـ driver `NULL`ها، الـ core بيكال `pin_config_set` على كل pin في الـ group individually.
- الـ driver المتقن بيعمل bulk write للـ hardware registers لكل الـ pins في loop واحدة.
- الـ group selector بييجي من الـ pinmux mapping — نفسه اللي بيتحدد في الـ DT `pinctrl-0` property.

**Who calls it:**
- `pinconf_apply_config()` لما الـ map entry بتبص على group مش individual pin

---

### Category 2: Debugfs Hooks

الـ callbacks دي **optional** — لو الـ driver مش محتاج debugfs output مخصوص، ممكن يسيبها `NULL` والـ core بيتعامل معاها. بتتكال بس لما الـ kernel compiled مع `CONFIG_DEBUG_FS`.

---

#### `pin_config_dbg_show`

```c
void (*pin_config_dbg_show)(struct pinctrl_dev *pctldev,
                            struct seq_file *s,
                            unsigned int offset);
```

بتطبع الـ configuration الحالية لـ pin معين على الـ `seq_file` بتاع الـ debugfs. الـ driver بيكتب فيها أي معلومات مفيدة: pull state، drive strength، slew rate، إلخ.

**الـ Parameters:**
- `pctldev` — الـ pin controller handle
- `s` — الـ `seq_file` pointer للكتابة عليه بـ `seq_printf()` أو `seq_puts()`
- `offset` — الـ pin number (offset في الـ controller's pin space)

**الـ Return:** `void` — مفيش return

**Key Details:**
- مفيش locking مضمون من الـ framework على الـ `seq_file` — الـ `seq_file` نفسه thread-safe.
- ممنوع sleep طويل أو blocking operations جوا الـ callback.
- الناتج بيظهر في `/sys/kernel/debug/pinctrl/<device>/pins` لما تعمل `cat`.

**Who calls it:**
- `pinconf_show_pin()` في `drivers/pinctrl/pinconf.c` اللي بيتكال من الـ debugfs file operations

---

#### `pin_config_group_dbg_show`

```c
void (*pin_config_group_dbg_show)(struct pinctrl_dev *pctldev,
                                  struct seq_file *s,
                                  unsigned int selector);
```

نفس `pin_config_dbg_show` لكن على مستوى الـ group. بتطبع معلومات الـ configuration للـ group كلها.

**الـ Parameters:**
- `pctldev` — الـ pin controller handle
- `s` — الـ seq_file للكتابة
- `selector` — الـ group selector index

**الـ Return:** `void`

**Key Details:**
- الناتج بيظهر في `/sys/kernel/debug/pinctrl/<device>/pingroups`.
- الـ driver عادةً بيلف على pins الـ group ويطبع config كل واحدة.

**Who calls it:**
- الـ debugfs group display code في pinctrl core

---

#### `pin_config_config_dbg_show`

```c
void (*pin_config_config_dbg_show)(struct pinctrl_dev *pctldev,
                                   struct seq_file *s,
                                   unsigned long config);
```

بتطبع تفسير بشري لـ raw `unsigned long` config value. الـ framework بيكالها لما محتاج يعرض config value مش مفهومة للإنسان.

**الـ Parameters:**
- `pctldev` — الـ pin controller handle
- `s` — الـ seq_file للكتابة
- `config` — الـ raw config value اللي محتاج تفسير (نفس format الـ configs[] array)

**الـ Return:** `void`

**Key Details:**
- الـ callback ده بيفيد خصوصاً في الـ generic pinconf path عشان يطبع الـ parameter name والـ value بشكل مقروء.
- الـ implementation النموذجية: `seq_printf(s, "param=%d val=%d", PARAM(config), VAL(config))`.
- لو `NULL`، الـ framework بيطبع الـ raw hex value بدل الـ human-readable form.

**Who calls it:**
- الـ pinconf debugfs infrastructure لما بتعرض mappings

---

### الـ `is_generic` Flag

```c
#ifdef CONFIG_GENERIC_PINCONF
    bool is_generic;
#endif
```

الـ flag ده مش function، بس مهم جداً. لو `true`، بيقول للـ framework إن الـ driver بيستخدم الـ **generic pinconf** interface من `include/linux/pinctrl/pinconf-generic.h`. ده معناه:

- الـ framework بيفهم الـ standard `PIN_CONFIG_*` parameters (bias, drive-strength, إلخ) تلقائياً.
- الـ DT parsing بيتعمل تلقائياً بدون كود خاص في الـ driver.
- الـ `configs[]` array بتتبني من الـ DT properties تلقائياً من خلال `pinconf_generic_parse_dt_config()`.

بدون الـ flag ده (أو لو `false`)، الـ driver مسؤول عن parse الـ DT بنفسه وتعريف format الـ config values الـ custom بتاعته.

---

### Architecture Overview

```
Consumer Driver / DT
        |
        v
  pinctrl_select_state()
        |
        v
  pinctrl core (pinconf.c)
        |
        +---> pin_config_group_set()  <-- group-level config
        |
        +---> pin_config_set()        <-- per-pin config
        |
        v
  pin controller driver
  (implements pinconf_ops)
        |
        v
  Hardware Registers (SoC)


debugfs reads:
  pin_config_dbg_show()
  pin_config_group_dbg_show()
  pin_config_config_dbg_show()
```

---

### العلاقة بـ `struct pinctrl_desc`

الـ `pinconf_ops` بتتسجل كجزء من `pinctrl_desc.confops`:

```c
static const struct pinconf_ops my_pinconf_ops = {
    .is_generic           = true,
    .pin_config_get       = my_pin_config_get,
    .pin_config_set       = my_pin_config_set,
    .pin_config_group_get = my_pin_config_group_get,
    .pin_config_group_set = my_pin_config_group_set,
    .pin_config_dbg_show  = pinconf_generic_dump_pin,    /* from generic */
};

static const struct pinctrl_desc my_pctrl_desc = {
    .name     = "my-pinctrl",
    .pins     = my_pins,
    .npins    = ARRAY_SIZE(my_pins),
    .pctlops  = &my_pctrl_ops,
    .pmxops   = &my_pmx_ops,
    .confops  = &my_pinconf_ops,   /* <-- هنا */
    .owner    = THIS_MODULE,
};
```

لو `confops == NULL` في الـ `pinctrl_desc`، الـ core بيعتبر إن الـ controller مش بيدعم pin configuration خالص، وأي طلب config بيرجع error.
## Phase 5: دليل الـ Debugging الشامل

الـ `pinconf.h` بيعرّف الـ `struct pinconf_ops` — الـ vtable اللي بيسجّل فيها الـ pin controller driver كل عمليات الـ pin configuration. الـ debugging هنا بيشمل مستويين: الـ software (kernel subsystem) والـ hardware (pins فعليين على الـ PCB).

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ pinctrl subsystem بيعمل automatically hierarchy تحت `/sys/kernel/debug/pinctrl/`.

```
/sys/kernel/debug/pinctrl/
├── <controller-name>/          # مثال: pinctrl-bcm2835, 1c20800.pinctrl
│   ├── pins                    # كل الـ pins المسجّلة مع أسمائها وأرقامها
│   ├── pingroups               # الـ pin groups المتاحة
│   ├── pinconf-pins            # config الـ pins الفردية
│   ├── pinconf-groups          # config الـ groups
│   ├── pinmux-functions        # الـ mux functions المتاحة
│   ├── pinmux-pins             # الـ mux state لكل pin
│   └── gpio-ranges             # الـ GPIO ranges المربوطة بالـ controller
```

**كيفية القراءة:**

```bash
# اقرأ config كل الـ pins (بيستدعي pin_config_dbg_show لكل pin)
cat /sys/kernel/debug/pinctrl/<ctrl>/pinconf-pins

# اقرأ config الـ groups (بيستدعي pin_config_group_dbg_show)
cat /sys/kernel/debug/pinctrl/<ctrl>/pinconf-groups

# شوف كل الـ controllers المسجّلة
ls /sys/kernel/debug/pinctrl/

# شوف الـ pins بالتفصيل
cat /sys/kernel/debug/pinctrl/<ctrl>/pins
# مثال output:
# registered pins: 54
# pin 0 (GPIO0) drv_data=(nil)
# pin 1 (GPIO1) drv_data=(nil)
```

الـ hooks المسؤولة مباشرةً عن هذا الـ output هي الثلاث functions في `pinconf_ops`:
- `pin_config_dbg_show` ← بيطبع info كل pin فردي
- `pin_config_group_dbg_show` ← بيطبع info كل group
- `pin_config_config_dbg_show` ← بيفك تشفير قيمة الـ config وبيطبعها human-readable

---

#### 2. مدخلات الـ sysfs

الـ pinctrl مش بيكتب كتير في الـ sysfs مباشرةً، لكن:

```
/sys/bus/platform/drivers/<driver-name>/
/sys/devices/platform/<ctrl-device>/
    ├── driver/                 # اللي بيدير الـ controller
    ├── of_node -> ...          # لينك للـ DT node
    └── pinctrl-handles         # (بعض الـ drivers بيضيفوها)

# الـ GPIO sysfs (لو الـ pin متعمل export)
/sys/class/gpio/gpioN/
    ├── direction
    ├── value
    └── active_low
```

```bash
# شوف الـ device المرتبط بالـ pin controller
ls -la /sys/devices/platform/ | grep pinctrl

# شوف لو الـ consumer device اتربط بالـ pinctrl صح
cat /sys/devices/platform/<consumer-dev>/pinctrl-names 2>/dev/null
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

الـ pinctrl subsystem عنده tracepoints مدمجة:

```bash
# شوف الـ pinctrl events المتاحة
ls /sys/kernel/debug/tracing/events/pinctrl/

# الـ events الأساسية:
# pinctrl_gpio_request
# pinctrl_gpio_free
# pinctrl_gpio_direction
# pinctrl_select_state_begin
# pinctrl_select_state_end
```

**تفعيل الـ tracing:**

```bash
# تفعيل كل أحداث الـ pinctrl
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# تتبع pin config set/get فقط (لو متاحين)
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pin_config_set/enable
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pin_config_get/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# نفّذ الـ operation اللي عايز تتبعها
# مثال: اعمل state change لـ device
echo default > /sys/devices/platform/<dev>/pinctrl-state

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace

# وقف الـ trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo 0 > /sys/kernel/debug/tracing/events/pinctrl/enable
```

**تتبع الـ function calls مباشرةً:**

```bash
# تتبع كل functions الـ pinconf
echo 'pinconf_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'pin_config_*' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe  # live output
```

---

#### 4. الـ printk والـ Dynamic Debug

```bash
# تفعيل dynamic debug لـ pinctrl subsystem بالكامل
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لـ driver معيّن (مثال: pinctrl-bcm2835)
echo 'module pinctrl_bcm2835 +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل لـ file معيّن داخل الـ kernel
echo 'file drivers/pinctrl/core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinconf.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع line numbers وأسماء الـ functions في الـ output
echo 'module pinctrl +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug rules الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep pinctrl

# لو بتعمل kernel boot وعايز تفعّل من البداية
# أضف في kernel cmdline:
# dyndbg="module pinctrl +p"
```

**في وقت الـ boot عبر kernel command line:**

```
dyndbg="file drivers/pinctrl/pinconf.c +p; file drivers/pinctrl/core.c +p"
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_PINCTRL` | تفعيل الـ pinctrl subsystem الأساسي |
| `CONFIG_GENERIC_PINCONF` | تفعيل الـ generic pin config (يفعّل `is_generic` flag) |
| `CONFIG_DEBUG_PINCTRL` | تفعيل debug messages إضافية في الـ core |
| `CONFIG_PINCTRL_SINGLE` | الـ single pinctrl driver (مفيد للـ testing) |
| `CONFIG_DEBUG_FS` | **لازم يكون مفعّل** عشان `/sys/kernel/debug/pinctrl/` يشتغل |
| `CONFIG_TRACING` | تفعيل الـ ftrace infrastructure |
| `CONFIG_FUNCTION_TRACER` | تفعيل function tracing |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل `pr_debug()` dynamically |
| `CONFIG_PROVE_LOCKING` | اكتشاف الـ deadlocks في الـ pinctrl locks |
| `CONFIG_LOCKDEP` | تتبع الـ lock dependencies |
| `CONFIG_KASAN` | اكتشاف الـ memory corruption في الـ driver |

```bash
# تحقق من الـ config الحالية
grep -E 'CONFIG_(PINCTRL|GENERIC_PINCONF|DEBUG_PINCTRL|DEBUG_FS)' /boot/config-$(uname -r)
# أو
zcat /proc/config.gz | grep PINCTRL
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# pinctrl tool من الـ userspace (جزء من util-linux أو مستقل)
# بعض الـ distros بيجيب pinctrl command

# على Raspberry Pi (bcm2835):
raspi-gpio get <pin-number>    # قراءة state الـ pin
raspi-gpio set <pin-number> ip # ضبط كـ input

# قراءة الـ pinctrl state لكل الـ devices
for dev in /sys/devices/platform/*/; do
    state_file="$dev/pinctrl-state"
    [ -f "$state_file" ] && echo "$dev: $(cat $state_file)"
done

# شوف كل الـ pin controllers المسجّلة
ls /sys/kernel/debug/pinctrl/

# مقارنة الـ requested configs مع الـ applied
cat /sys/kernel/debug/pinctrl/<ctrl>/pinconf-pins > /tmp/before.txt
# افعل شيء
cat /sys/kernel/debug/pinctrl/<ctrl>/pinconf-pins > /tmp/after.txt
diff /tmp/before.txt /tmp/after.txt
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `pin_config_get op not implemented` | الـ driver ما عمّلش `pin_config_get` في الـ `pinconf_ops` | تحقق إن الـ driver بيملأ الـ vtable صح |
| `pin X is not valid` | رقم الـ pin خارج نطاق الـ `npins` في الـ `pinctrl_desc` | تحقق من حجم الـ pins array والـ pin numbers |
| `could not get pinctrl handle` | الـ device مش لاقي الـ pinctrl controller المطلوب | تحقق من `pinctrl-0` في الـ DT وإن الـ controller اتسجّل |
| `pinctrl: add range failed` | تعارض في الـ GPIO ranges | تحقق إن الـ ranges مش متداخلة |
| `pin already requested` | حاول driver تاني يطلب نفس الـ pin | تحقق من الـ pin ownership |
| `-ENOTSUPP` من `pin_config_get` | الـ config parameter غير مدعوم في هذا الـ controller | استخدم parameter مدعوم أو implement الـ support |
| `-EINVAL` من `pin_config_get` | الـ parameter مدعوم لكن مش مفعّل على الـ pin | فعّل الـ config أو تحقق من الـ hardware state |
| `pinctrl: neither ops nor DT node` | الـ controller اتسجّل من غير `pinctrl_ops` | أضف الـ `pctlops` للـ `pinctrl_desc` |
| `failed to apply state X` | فشل الـ `pin_config_set` أو الـ `pin_config_group_set` | افحص return code الـ driver وراجع الـ hardware constraints |
| `pinctrl handle locked` | الـ pinctrl handle اتقفل من thread تاني | مشكلة concurrency، راجع الـ locking |

---

#### 8. نقاط استراتيجية لـ dump_stack() وـ WARN_ON()

ضع الـ checks دي في الـ driver اللي بيـ implement الـ `pinconf_ops`:

```c
/* في pin_config_set — تحقق من الـ input */
int my_pin_config_set(struct pinctrl_dev *pctldev, unsigned int pin,
                      unsigned long *configs, unsigned int num_configs)
{
    /* تحقق من أن الـ pin صالح */
    if (WARN_ON(pin >= MY_NUM_PINS))
        return -EINVAL;

    /* تحقق من أن الـ configs pointer مش NULL */
    if (WARN_ON(!configs || num_configs == 0))
        return -EINVAL;

    for (int i = 0; i < num_configs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        u32 arg = pinconf_to_config_argument(configs[i]);

        /* تحقق من أن الـ param في النطاق الصحيح */
        if (WARN_ON(param > PIN_CONFIG_END)) {
            dev_err(pctldev->dev, "invalid config param %u\n", param);
            return -EINVAL;
        }

        /* نقطة debug استراتيجية: سجّل كل config تنطبق */
        dev_dbg(pctldev->dev, "pin %u: param=%u arg=%u\n", pin, param, arg);
    }
    /* ... */
}

/* في pin_config_get — dump الـ state لو unexpected */
int my_pin_config_get(struct pinctrl_dev *pctldev, unsigned int pin,
                      unsigned long *config)
{
    u32 reg_val = read_pin_register(pin); /* قراءة من الـ hardware */

    /* لو الـ register بيرجع قيمة غريبة */
    if (WARN_ON(reg_val == 0xDEADBEEF)) {
        dump_stack(); /* طبع الـ call stack كاملاً */
        return -EIO;
    }
    /* ... */
}
```

**نقاط WARN_ON الأهم:**
- بعد `pin_config_group_set` — تحقق إن كل الـ pins في الـ group اتضبطت
- عند تغيير config pin بيستخدمه driver تاني في نفس الوقت
- لو الـ `pin_config_dbg_show` بيقرأ state مختلف عن اللي اتكتب

---

### Hardware Level

#### 1. التحقق من تطابق الـ Hardware State مع الـ Kernel State

```
Kernel State                    Hardware State
─────────────────────           ────────────────────────
pinconf-pins debugfs      ←→    Actual register values
pin_config_get() return   ←→    Oscilloscope/multimeter
DT pin-config properties  ←→    Schematic pull-up/down R
```

```bash
# الخطوة 1: اقرأ الـ config من الـ kernel
cat /sys/kernel/debug/pinctrl/<ctrl>/pinconf-pins

# الخطوة 2: قارنها مع قيمة الـ register مباشرةً (الخطوة التالية)
```

---

#### 2. Register Dump Techniques

```bash
# استخدام devmem2 (لازم تكون موجود أو compile يدوي)
# اقرأ register بحجم 32-bit من عنوان معيّن
devmem2 0x01C20800 w    # مثال: Allwinner A20 PIO base

# استخدام /dev/mem (لو CONFIG_DEVMEM=y)
dd if=/dev/mem bs=4 count=1 skip=$((0x01C20800/4)) 2>/dev/null | xxd

# استخدام io tool (من package 'moreutils' أو مشابه)
io -4 0x01C20800        # قراءة 32-bit register

# استخدام Python لقراءة أكثر مرونة
python3 -c "
import mmap, os, struct
f = open('/dev/mem', 'rb')
m = mmap.mmap(f.fileno(), 4096, offset=0x01C20800 & ~0xFFF)
m.seek(0x01C20800 & 0xFFF)
val = struct.unpack('<I', m.read(4))[0]
print(f'Register: 0x{val:08X}')
m.close(); f.close()
"
```

**لتحديد عنوان الـ register الصح:**

```bash
# ابحث في الـ DTS عن الـ reg property للـ controller
cat /proc/device-tree/<controller-node>/reg | xxd

# أو من الـ DT compiled
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "pinctrl"
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

```
Scenario: التحقق من pull-up/pull-down
─────────────────────────────────────
1. اضبط الـ pin كـ input عبر pinctrl
2. اتركه floating (مش موصول بحاجة)
3. قس الـ voltage:
   - ~VDD (3.3V أو 1.8V) → pull-up مفعّل ✓
   - ~GND (0V)           → pull-down مفعّل ✓
   - floating/noise      → bias مش مضبوط ✗

Scenario: التحقق من drive strength
────────────────────────────────────
1. وصّل load معروف (مثال: 47Ω إلى GND)
2. اضبط الـ pin كـ output high
3. قس الـ voltage:
   - High voltage → drive strength كافية
   - Voltage sag  → drive strength قليلة، زوّد الـ config

Scenario: التحقق من slew rate
───────────────────────────────
1. استخدم oscilloscope بـ bandwidth عالية (> 100MHz)
2. اضبط output يتبدّل بين 0 و 1 بسرعة
3. قس rise time/fall time بالـ scope
4. قارن مع datasheet slew rate specifications
```

**معادلة التحويل:**

```
Drive Strength (mA) = ΔV / R_load
مثال: لو الـ voltage انخفض من 3.3V لـ 2.8V على 100Ω:
I = (3.3 - 2.8) / 100 = 5mA → الـ drive strength = 4mA غير كافية
```

---

#### 4. المشاكل الشائعة وأنماط الـ Kernel Log

| المشكلة الـ Hardware | نمط الـ Kernel Log | السبب |
|---|---|---|
| Pin مش pulling | `pinctrl: failed to set config` + جهاز مش شغّال | DT ما فيهاش `bias-pull-up` / `bias-pull-down` |
| I2C signal degradation | `i2c i2c-X: timeout` مع `scl` stuck | Drive strength منخفضة أو pull-up مش موجود |
| SPI data corruption | `spi: transfer failed` | Slew rate عالية جداً أو noise على الـ bus |
| GPIO interrupt مش بيجي | `irq: no irqchip found` أو لا response | الـ pin مش configured كـ interrupt input |
| Voltage clash | `pinctrl: pin voltage conflict` | الـ pin domain voltage مش متوافق مع الـ signal |
| Open-drain مش شغّال | الـ pin مش بيرجع high | ناسي الـ external pull-up resistor على الـ PCB |

---

#### 5. Device Tree Debugging — التحقق من تطابق الـ DT مع الـ Hardware

```bash
# Dump الـ DT المُحمَّل فعلياً في الـ kernel
dtc -I fs /proc/device-tree -O dts 2>/dev/null > /tmp/live_dt.dts

# ابحث عن كل الـ pin config nodes
grep -A 20 "pinctrl" /tmp/live_dt.dts

# شوف الـ pinctrl properties للـ device المشكلة
cat /proc/device-tree/<device-path>/pinctrl-names
cat /proc/device-tree/<device-path>/pinctrl-0 | xxd  # phandle
```

**مثال DT node للتحقق:**

```dts
/* DT المفروض يكون بالشكل ده */
&uart0 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart0_default>;   /* state "default" */
    pinctrl-1 = <&uart0_sleep>;     /* state "sleep"   */
    status = "okay";
};

uart0_default: uart0-default {
    pins = "PA4", "PA5";
    function = "uart0";
    bias-pull-up;                   /* PIN_CONFIG_BIAS_PULL_UP */
    drive-strength = <10>;          /* PIN_CONFIG_DRIVE_STRENGTH: 10mA */
};
```

```bash
# تحقق إن الـ phandle صح
# اقرأ الـ phandle value من الـ consumer device
hexdump -C /proc/device-tree/<consumer>/pinctrl-0

# ابحث عن الـ node اللي عنده نفس الـ phandle
grep -r "phandle" /proc/device-tree/ 2>/dev/null | grep <value>

# تحقق من الـ pin config properties اللي اتقرأت فعلاً
# (لازم تضيف dev_dbg في الـ dt_node_to_map function)
echo 'file drivers/pinctrl/pinconf-generic.c +p' > /sys/kernel/debug/dynamic_debug/control
dmesg | grep pinconf
```

**أكثر أخطاء الـ DT شيوعاً:**

```bash
# الخطأ 1: property اسمها غلط
# خطأ:  bias-pullup   ← اسم غلط
# صح:   bias-pull-up  ← الاسم الصحيح

# الخطأ 2: function غير مدعومة
# لو الـ kernel log يقول:
# "pinctrl: invalid function"
# اقرأ الـ functions المتاحة:
cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-functions

# الخطأ 3: pin اسمه غلط
# اقرأ الأسماء الصحيحة من:
cat /sys/kernel/debug/pinctrl/<ctrl>/pins
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

```bash
#!/bin/bash
# === pinconf Debug Script ===

CTRL=$(ls /sys/kernel/debug/pinctrl/ | head -1)
echo "=== Pin Controller: $CTRL ==="

# 1. كل الـ pins وأرقامها
echo "--- Registered Pins ---"
cat /sys/kernel/debug/pinctrl/$CTRL/pins

# 2. الـ pin config الحالية
echo "--- Pin Configurations ---"
cat /sys/kernel/debug/pinctrl/$CTRL/pinconf-pins 2>/dev/null || \
    echo "pinconf-pins not available (driver may not implement pin_config_dbg_show)"

# 3. الـ group configurations
echo "--- Group Configurations ---"
cat /sys/kernel/debug/pinctrl/$CTRL/pinconf-groups 2>/dev/null

# 4. الـ mux state
echo "--- Pinmux State ---"
cat /sys/kernel/debug/pinctrl/$CTRL/pinmux-pins 2>/dev/null

# 5. تفعيل dynamic debug
echo "--- Enabling Dynamic Debug ---"
echo "module pinctrl +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/pinctrl/pinconf.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "Dynamic debug enabled"

# 6. تفعيل ftrace
echo "--- Enabling ftrace for pinctrl ---"
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "ftrace enabled - reading trace_pipe (Ctrl+C to stop)"
timeout 10 cat /sys/kernel/debug/tracing/trace_pipe

# cleanup
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo 0 > /sys/kernel/debug/tracing/events/pinctrl/enable
```

---

#### مثال Output وكيفية تفسيره

**output من `pinconf-pins`:**

```
pin 0 (SDA): input bias-pull-up [100k ohms]
pin 1 (SCL): input bias-pull-up [100k ohms]
pin 4 (GPIO4): output drive-strength [8mA] slew-rate [0]
pin 17 (GPIO17): input bias-disable
pin 27 (GPIO27): output open-drain
```

**تفسير الـ output:**
- `bias-pull-up [100k ohms]` ← `PIN_CONFIG_BIAS_PULL_UP` مفعّل بقيمة 100kΩ
- `drive-strength [8mA]` ← `PIN_CONFIG_DRIVE_STRENGTH` = 8
- `bias-disable` ← الـ pin floating بدون أي bias
- `open-drain` ← `PIN_CONFIG_DRIVE_OPEN_DRAIN` مفعّل

**output من `dmesg` بعد تفعيل dynamic debug:**

```
[   12.345678] pinctrl core: registered pin controller bcm2835_pinctrl
[   12.345712] pinconf: pin_config_set: pin=4, param=PIN_CONFIG_DRIVE_STRENGTH, arg=8
[   12.345715] pinconf: pin_config_set: pin=4 OK
[   12.346001] pinctrl core: apply state 'default' for device spi0
```

**لو شفت الـ output ده:**

```
[   12.500000] pinconf: pin_config_get: pin=17 returned -ENOTSUPP
```

يعني الـ driver ما supportش الـ query لهذا الـ parameter — راجع الـ `pin_config_get` implementation وتحقق من الـ supported params.

```bash
# مثال: استخدام devmem2 وتفسير النتيجة
devmem2 0x3F200000 w
# Value at address 0x3F200000 (0x3f200000): 0x00240900
#
# 0x00240900 بالـ binary:
# 0000 0000 0010 0100 0000 1001 0000 0000
#
# على BCM2835: كل 3 bits بتحدد function لـ pin
# bits [2:0]   = pin 0 function = 000 = INPUT
# bits [5:3]   = pin 1 function = 000 = INPUT
# bits [8:6]   = pin 2 function = 010 = ALT5
# bits [11:9]  = pin 3 function = 100 = ALT0 (I2C)
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART مش شغال على STM32MP1 — Industrial Gateway

#### العنوان
**الـ pull-up مش متفعّل على UART RX — بيانات corrupted على gateway صناعي**

#### السياق
شركة بتعمل industrial gateway بناءً على STM32MP1 (Cortex-A7 + Cortex-M4). الـ gateway بيتكلم مع sensors عن طريق RS-485 via UART. في الـ bring-up، الـ UART بيشتغل بس البيانات corrupted بنسبة 30% خصوصاً لما الـ cable طويل.

#### المشكلة
الـ driver بيـ call `pin_config_set` عشان يـ configure الـ UART RX pin، بس الـ `configs` array بيبعت config parameter واحد بس — الـ `PIN_CONFIG_DRIVE_STRENGTH` — من غير ما يحدد `PIN_CONFIG_BIAS_PULL_UP`. الـ pin فاضل floating لما مفيش signal.

#### التحليل
في `pinconf.h`، الـ `pin_config_set` signature هو:

```c
int (*pin_config_set) (struct pinctrl_dev *pctldev,
                       unsigned int pin,
                       unsigned long *configs,      /* array of packed configs */
                       unsigned int num_configs);   /* how many configs in array */
```

الـ driver بيعمل كده:

```c
unsigned long cfg[] = {
    PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 4), /* 4mA only */
};
/* num_configs = 1, no pull-up included! */
ret = pinconf_ops->pin_config_set(pctldev, uart_rx_pin, cfg, 1);
```

الـ framework بيمرر الـ `configs` array كـ `unsigned long *` — يعني كل element هو packed value فيه الـ parameter ID في الـ high bits والـ argument في الـ low bits. المشكلة إن `num_configs = 1` وملقتش config للـ bias.

الـ `pin_config_get` لو اتسألت على الـ bias هترجع `-EINVAL` (available but disabled) — وده indicator واضح إن الـ pull-up موجود في الـ controller بس معطلش.

#### الحل
إضافة `PIN_CONFIG_BIAS_PULL_UP` للـ configs array:

```c
unsigned long cfg[] = {
    PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 4),
    PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 1),   /* enable pull-up */
};
ret = pinconf_ops->pin_config_set(pctldev, uart_rx_pin, cfg,
                                  ARRAY_SIZE(cfg)); /* num_configs = 2 */
```

أو في Device Tree:

```dts
&uart4 {
    pinctrl-0 = <&uart4_pins>;
    pinctrl-names = "default";
};

&pinctrl {
    uart4_pins: uart4-pins {
        pins = "PD6", "PD7";
        function = "uart4";
        bias-pull-up;           /* maps to PIN_CONFIG_BIAS_PULL_UP */
        drive-strength = <4>;
    };
};
```

#### الدرس المستفاد
الـ `pin_config_set` بتاخد **array** من configs (`unsigned long *configs, unsigned int num_configs`) — مش config واحدة. لازم تبعت كل الـ configs المطلوبة في نفس الـ call. الـ `-EINVAL` من `pin_config_get` معناها "available but disabled" — وده مش error، ده hint إنك محتاج تـ enable الـ config ده.

---

### السيناريو 2: SPI فلاش مش بيـ boot على Rockchip RK3562 — Android TV Box

#### العنوان
**الـ `pin_config_group_set` بترجع `-ENOTSUPP` — SPI NOR flash مش بيتعرف عليه**

#### السياق
منتج Android TV box بناءً على RK3562. الـ SPI NOR flash (W25Q128) هو الـ boot device. بعد kernel upgrade، الـ board بيفضل يـ reboot في loop. الـ bootloader شغال تمام، المشكلة في الـ kernel driver.

#### المشكلة
الـ pinctrl driver للـ RK3562 بيـ implement `pin_config_group_set` بس مش بيـ handle الـ `PIN_CONFIG_SLEW_RATE` parameter لأن الـ hardware مش بيـ support سرعة تحكم في الـ slew. بيرجع `-ENOTSUPP` بدل ما يتجاهل الـ unsupported parameter. الـ SPI driver الـ new version بيحاول يـ set slew rate فبيـ fail كله.

#### التحليل
الـ `pinconf_ops` struct في `pinconf.h`:

```c
int (*pin_config_group_set) (struct pinctrl_dev *pctldev,
                             unsigned int selector,   /* group selector */
                             unsigned long *configs,
                             unsigned int num_configs);
```

الـ comment في الـ header واضح:

```c
 * @pin_config_group_set: configure all pins in a group
```

الـ kernel doc بتقول إن `-ENOTSUPP` معناها "this config type is not available on this controller" — الـ SPI subsystem لما بيشوف الـ `-ENOTSUPP` من `pin_config_group_set` بيـ treat it كـ fatal error في بعض الـ versions الجديدة.

الـ RK3562 driver كان عنده:

```c
static int rk3562_pin_config_group_set(struct pinctrl_dev *pctldev,
                                       unsigned int selector,
                                       unsigned long *configs,
                                       unsigned int num_configs)
{
    for (int i = 0; i < num_configs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        switch (param) {
        case PIN_CONFIG_SLEW_RATE:
            return -ENOTSUPP; /* BUG: should skip, not fail */
        /* ... other cases ... */
        }
    }
    return 0;
}
```

#### الحل
الـ driver لازم يـ skip الـ unsupported params بدل ما يـ return error:

```c
static int rk3562_pin_config_group_set(struct pinctrl_dev *pctldev,
                                       unsigned int selector,
                                       unsigned long *configs,
                                       unsigned int num_configs)
{
    for (int i = 0; i < num_configs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        switch (param) {
        case PIN_CONFIG_SLEW_RATE:
            /* hardware doesn't support slew rate control, skip */
            break;
        /* ... other cases ... */
        default:
            return -ENOTSUPP;
        }
    }
    return 0;
}
```

#### الدرس المستفاد
الـ `pinconf_ops` contract واضح في `pinconf.h`: الـ `-ENOTSUPP` معناها "this config type not available at all" — مش "skipped". لو الـ hardware مش بيـ support parameter معين، الـ driver لازم يـ silently skip عشان caller تانية ميتأثروش. الـ comment في `pinconf.h` على `pin_config_group_get` و`pin_config_get` بيوضح الفرق بين `-ENOTSUPP` و `-EINVAL`.

---

### السيناريو 3: I2C جهاز مش بيتعرف على i.MX8MP — IoT Sensor Hub

#### العنوان
**الـ `pin_config_dbg_show` كاشفة إن الـ I2C pins في wrong mode — sensor مش بيـ respond**

#### السياق
IoT sensor hub بناءً على i.MX8MP. بيحمل BME680 (temperature/humidity/pressure) عن طريق I2C3. الـ sensor driver بيـ load بس `i2c_transfer` بترجع `-EREMOTEIO` دايماً. الـ hardware team أكد إن الـ hardware صح.

#### المشكلة
في الـ DT، الـ I2C3 pins اتعملها `pinctrl-0` لكن بـ wrong `function` — اتـ configure كـ GPIO بدل I2C. الـ `pin_config_dbg_show` في الـ `pinconf_ops` لو اتـ implemented صح كانت تكشف ده فوراً.

#### التحليل
الـ `pinconf_ops` في `pinconf.h` بيوفر:

```c
void (*pin_config_dbg_show) (struct pinctrl_dev *pctldev,
                             struct seq_file *s,
                             unsigned int offset);  /* pin offset/number */
```

الـ debugfs path هو `/sys/kernel/debug/pinctrl/<controller>/pins`. لو الـ i.MX8MP driver عنده `pin_config_dbg_show` مـ implemented، ممكن تقرأ حالة كل pin بما فيها الـ mux function والـ config.

```bash
# Check pin state via debugfs
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pins | grep -A2 "I2C3"
# Expected output if correct:
# pin 120 (I2C3_SCL): function=i2c3, drive=6mA, pull=up, open-drain
# Actual output (buggy DT):
# pin 120 (I2C3_SCL): function=gpio, drive=4mA, pull=none
```

الـ `pin_config_group_dbg_show` كمان مفيد:

```c
void (*pin_config_group_dbg_show) (struct pinctrl_dev *pctldev,
                                   struct seq_file *s,
                                   unsigned int selector); /* group selector */
```

```bash
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pingroups
# Shows which group the I2C3 pins belong to and their config
```

#### الحل
تصحيح الـ DT:

```dts
/* WRONG - was configured as GPIO */
&iomuxc {
    i2c3_pins: i2c3-pins {
        fsl,pins = <
            MX8MP_IOMUXC_I2C3_SCL__GPIO5_IO18  0x400001c3  /* BUG: GPIO mode */
            MX8MP_IOMUXC_I2C3_SDA__GPIO5_IO19  0x400001c3
        >;
    };
};

/* CORRECT - I2C function with proper config */
&iomuxc {
    i2c3_pins: i2c3-pins {
        fsl,pins = <
            MX8MP_IOMUXC_I2C3_SCL__I2C3_SCL    0x400001c3  /* open-drain, pull-up */
            MX8MP_IOMUXC_I2C3_SDA__I2C3_SDA    0x400001c3
        >;
    };
};
```

الـ `0x400001c3` value بيـ encode الـ pull-up + open-drain + drive strength كـ packed config يعرفه الـ i.MX8 pinctrl driver لما الـ `pin_config_set` يـ parse الـ DT.

#### الدرس المستفاد
الـ `pin_config_dbg_show` و `pin_config_group_dbg_show` في `pinconf_ops` هم أول tool للـ debug. لازم كل pinctrl driver يـ implement هم. الـ debugfs بيـ call الـ callbacks دول automatically لما تعمل `cat` على الـ pins/pingroups files.

---

### السيناريو 4: USB مش بيتعرف على الـ controller على AM62x — Automotive ECU

#### العنوان
**الـ `is_generic` flag مضبوطة غلط — الـ generic pinconf parser مش بيقرأ الـ DT صح**

#### السياق
Automotive ECU بناءً على TI AM62x (Sitara). بيستخدم USB للـ firmware update. الـ USB3.0 controller موجود في الـ SoC بس الـ kernel مش بيـ enumerate أي USB devices. الـ dmesg بيقول "failed to request pinctrl state".

#### المشكلة
الـ AM62x pinctrl driver عنده `is_generic = false` بس المفروض يكون `true` لأنه بيستخدم الـ generic pinconf interface. نتيجة لذلك، الـ framework بيحاول يـ call الـ driver-specific `pin_config_set` بدل الـ generic parser، وده بيـ fail لأن الـ driver مش معاه driver-specific parser للـ DT properties.

#### التحليل
في `pinconf.h`:

```c
struct pinconf_ops {
#ifdef CONFIG_GENERIC_PINCONF
    bool is_generic;    /* TRUE = use generic pinconf DT parser */
#endif
    int (*pin_config_get) ...
    int (*pin_config_set) ...
    ...
};
```

الـ `is_generic` flag بتقول للـ framework إن الـ driver بيـ support الـ standard DT properties زي `bias-pull-up`, `drive-strength`, `input-enable` إلخ بدل ما يستخدم vendor-specific properties.

لما الـ `is_generic = false`:
- الـ framework بيـ expect الـ driver يـ parse DT properties بنفسه
- الـ standard properties زي `bias-pull-up` بتتجاهل
- الـ USB pins بتفضل بـ default state (no pull, no drive config)

لما الـ `is_generic = true`:
- الـ `pinconf-generic.c` بيـ parse الـ standard DT properties
- كل property بتتحول لـ packed `unsigned long config`
- الـ `pin_config_set` بتاخد الـ configs دي

الـ AM62x driver كان فيه:

```c
static const struct pinconf_ops am62x_pinconf_ops = {
    .is_generic      = false,    /* BUG: should be true */
    .pin_config_get  = am62x_pinconf_get,
    .pin_config_set  = am62x_pinconf_set,
    /* group ops not implemented */
};
```

#### الحل

```c
static const struct pinconf_ops am62x_pinconf_ops = {
    .is_generic             = true,   /* enable generic DT parser */
    .pin_config_get         = am62x_pinconf_get,
    .pin_config_set         = am62x_pinconf_set,
    .pin_config_group_get   = am62x_pinconf_group_get,
    .pin_config_group_set   = am62x_pinconf_group_set,
};
```

والـ DT:

```dts
&usbss0 {
    pinctrl-names = "default";
    pinctrl-0 = <&usb0_pins>;
};

&main_pmx0 {
    usb0_pins: usb0-pins {
        pinctrl-single,pins = <
            AM62X_IOPAD(0x0254, PIN_OUTPUT, 0)   /* USB0_DRVVBUS */
        >;
        /* Generic pinconf properties - parsed because is_generic=true */
        bias-disable;
        drive-strength = <6>;
    };
};
```

#### الدرس المستفاد
الـ `is_generic` flag في `pinconf_ops` هو switch حاسم. لو الـ driver بيـ expect standard DT properties (`bias-pull-up`, `drive-strength`, etc) لازم `is_generic = true`. مش لازم تكتب DT parser من الصفر — الـ `pinconf-generic.c` موجود عشان كده. الـ `CONFIG_GENERIC_PINCONF` لازم يكون enabled في الـ kernel config كمان.

---

### السيناريو 5: HDMI مش بيشتغل على Allwinner H616 — Custom Board Bring-up

#### العنوان
**الـ `pin_config_config_dbg_show` غير مـ implemented — debugging الـ HDMI DDC pins صعب جداً**

#### السياق
Custom Android board بناءً على Allwinner H616 (Orange Pi Zero 2 compatible). الـ HDMI بيطلع صورة بس بدون صوت والـ EDID بيجيب garbage أحياناً. الـ DDC (I2C over HDMI) مش stable.

#### المشكلة
الـ H616 pinctrl driver مش عنده `pin_config_config_dbg_show` مـ implemented. لما الـ engineer يحاول يـ debug الـ HDMI DDC pin configs عن طريق debugfs، مش بيشوف أي معلومات مفيدة — بس hex values من غير تفسير.

#### التحليل
الـ `pinconf_ops` في `pinconf.h`:

```c
void (*pin_config_config_dbg_show) (struct pinctrl_dev *pctldev,
                                    struct seq_file *s,
                                    unsigned long config); /* single packed config */
```

الـ callback ده بيتـ call لما الـ debugfs يعرض config value واحدة — بيديك فرصة تـ decode الـ packed `unsigned long` وتعرضه بصورة human-readable.

بدون تنفيذه، الـ debugfs output هيبقى كده:

```
pin 215 (PH14/DDC_SCL): 0x00050001 0x00010001
```

مع التنفيذ، المفروض يبقى كده:

```
pin 215 (PH14/DDC_SCL):
  config[0]: BIAS_PULL_UP (1) — enabled
  config[1]: DRIVE_STRENGTH (5) — 20mA
```

الـ H616 driver كان عنده:

```c
static const struct pinconf_ops sun50i_h616_pinconf_ops = {
    .is_generic                  = true,
    .pin_config_get              = sunxi_pconf_get,
    .pin_config_set              = sunxi_pconf_set,
    .pin_config_group_get        = sunxi_pconf_group_get,
    .pin_config_group_set        = sunxi_pconf_group_set,
    .pin_config_dbg_show         = sunxi_pconf_dbg_show,
    .pin_config_group_dbg_show   = sunxi_pconf_group_dbg_show,
    /* pin_config_config_dbg_show = NULL — missing! */
};
```

#### الحل
إضافة `pin_config_config_dbg_show`:

```c
static void sunxi_pconf_config_dbg_show(struct pinctrl_dev *pctldev,
                                         struct seq_file *s,
                                         unsigned long config)
{
    /* Decode the packed config value into human-readable form */
    enum pin_config_param param = pinconf_to_config_param(config);
    u32 arg = pinconf_to_config_argument(config);

    switch (param) {
    case PIN_CONFIG_BIAS_PULL_UP:
        seq_printf(s, "pull-up(%s)", arg ? "enabled" : "disabled");
        break;
    case PIN_CONFIG_BIAS_PULL_DOWN:
        seq_printf(s, "pull-down(%s)", arg ? "enabled" : "disabled");
        break;
    case PIN_CONFIG_DRIVE_STRENGTH:
        seq_printf(s, "drive-strength(%umA)", arg);
        break;
    case PIN_CONFIG_INPUT_ENABLE:
        seq_printf(s, "input(%s)", arg ? "enabled" : "disabled");
        break;
    default:
        seq_printf(s, "unknown(param=%u, arg=%u)", param, arg);
        break;
    }
}

static const struct pinconf_ops sun50i_h616_pinconf_ops = {
    .is_generic                   = true,
    .pin_config_get               = sunxi_pconf_get,
    .pin_config_set               = sunxi_pconf_set,
    .pin_config_group_get         = sunxi_pconf_group_get,
    .pin_config_group_set         = sunxi_pconf_group_set,
    .pin_config_dbg_show          = sunxi_pconf_dbg_show,
    .pin_config_group_dbg_show    = sunxi_pconf_group_dbg_show,
    .pin_config_config_dbg_show   = sunxi_pconf_config_dbg_show, /* added */
};
```

بعد التطبيق، debugging الـ HDMI DDC pins:

```bash
# Check DDC pins config
cat /sys/kernel/debug/pinctrl/pio/pins | grep -A5 "PH14\|PH15"
# Now shows:
# pin 215 (PH14/DDC_SCL):
#   pull-up(enabled), drive-strength(20mA)
# pin 216 (PH15/DDC_SDA):
#   pull-up(enabled), drive-strength(20mA)
```

اكتشف إن الـ pull-up مفعّل بس الـ drive strength عالي جداً (20mA) — الـ HDMI spec بتقول max 6mA للـ DDC. ضبط الـ DT:

```dts
&pio {
    hdmi_ddc_pins: hdmi-ddc-pins {
        pins = "PH14", "PH15";
        function = "hdmi";
        bias-pull-up;
        drive-strength = <6>;   /* was 20, HDMI DDC spec max is 6mA */
    };
};
```

#### الدرس المستفاد
الـ `pin_config_config_dbg_show` في `pinconf_ops` هو optional بس قيمته كبيرة جداً وقت الـ debug. بيفضل الـ engineers يكتبوه من أول bring-up. الـ debugfs بيـ call الـ three debug callbacks (`pin_config_dbg_show`, `pin_config_group_dbg_show`, `pin_config_config_dbg_show`) تدريجياً — كل واحدة بتعطي level مختلف من الـ detail. الـ HDMI DDC bugs غالباً بتيجي من wrong drive strength مش من wrong pull config.
## Phase 7: مصادر ومراجع

### مقدمة

الـ `pinconf.h` جزء من **pinctrl subsystem** اللي اتضاف للـ Linux kernel سنة 2011. المصادر دي بتغطي الـ subsystem من الأساسيات للتفاصيل العملية.

---

### مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | مقال تأسيسي بيشرح فكرة الـ pinctrl subsystem كاملة — نقطة البداية المثالية |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | الـ patch الأصلية اللي أضافت الـ `pinconf_ops` وواجهة الـ pin configuration |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/) | مناقشة إضافة الـ generic pinconf interface (`pinconf-generic.h`) |
| [pinctrl/pinconfig: add debug interface](https://lwn.net/Articles/545790/) | إضافة الـ debugfs hooks اللي بتظهر في `pin_config_dbg_show` و `pin_config_group_dbg_show` |
| [pinctrl: common handling of generic pinconfig props in dt](https://lwn.net/Articles/553754/) | معالجة الـ device tree pinconf properties بشكل موحد |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | الـ documentation الرسمية الأولى للـ subsystem |
| [pinctrl: Add generic pinctrl-simple driver](https://lwn.net/Articles/496075/) | مثال عملي على driver بيستخدم الـ pinconf مع omap2+ |

---

### التوثيق الرسمي للـ Kernel

#### الـ Documentation الأساسية

```
Documentation/driver-api/pin-control.rst
```

**الـ URL المباشر:**
- [PINCTRL (PIN CONTROL) subsystem — kernel.org](https://docs.kernel.org/driver-api/pin-control.html)
- [النسخة القديمة v4.14](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html)
- [pinctrl.txt النص الخام](https://www.kernel.org/doc/Documentation/pinctrl.txt)

#### ملفات المصدر المرتبطة مباشرة

| الملف | الأهمية |
|-------|---------|
| `include/linux/pinctrl/pinconf.h` | الملف ده نفسه — تعريف `struct pinconf_ops` |
| `include/linux/pinctrl/pinconf-generic.h` | الـ generic interface فوق الـ pinconf_ops |
| `drivers/pinctrl/pinconf.c` | الـ core implementation للـ pinconf |
| `drivers/pinctrl/pinconf-generic.c` | الـ generic pinconf parsing والـ DT support |

#### روابط GitHub للمصدر

- [pinconf.h على GitHub](https://github.com/torvalds/linux/blob/master/include/linux/pinctrl/pinconf.h)
- [pinconf.c على GitHub](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/pinconf.c)
- [pinconf-generic.c على GitHub](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/pinconf-generic.c)
- [pinconf-generic.h على GitHub](https://github.com/torvalds/linux/blob/master/include/linux/pinctrl/pinconf-generic.h)

---

### Commits المهمة في تاريخ الـ Subsystem

الـ pinctrl subsystem اتبدأ من Linus Walleij في Linaro لحساب ST-Ericsson. الـ commits الأساسية اتقدرت تلاقيها في:

```bash
# شوف تاريخ الملف نفسه
git log --follow include/linux/pinctrl/pinconf.h

# شوف أول commit للـ subsystem
git log --oneline drivers/pinctrl/ | tail -20
```

**الـ Linaro Git Repository الأصلي للـ pinctrl:**
- [linux-pinctrl.git — Linaro Git Browser](https://git.linaro.org/people/linus.walleij/linux-pinctrl.git)

---

### نقاشات Mailing List

| الرابط | الموضوع |
|--------|---------|
| [LKML — pinconf DT dynamic alloc patch](https://lkml.iu.edu/hypermail/linux/kernel/1306.1/03349.html) | مناقشة تخصيص array ديناميكي في `pinconf_generic_parse_dt_config` |
| [LKML — pinctrl-single pinconf offsets](https://lkml.org/lkml/2026/2/9/44) | نقاش حديث عن الـ pinconf offsets في الـ pinctrl-single driver |
| [mail-archive — pinctrl debugfs](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg581364.html) | نقاش الـ debugfs entries للـ functionalities غير المنفذة |

**للبحث في الـ mailing list:**
```bash
# ابحث في lore.kernel.org
https://lore.kernel.org/linux-gpio/
```

الـ `linux-gpio` مailing list هو المكان الرئيسي لنقاشات الـ pinctrl/pinconf.

---

### كتب مقترحة

#### Linux Device Drivers 3rd Edition (LDD3)

- **الفصل 12**: I/O Memory — أساسيات التعامل مع hardware registers
- **الفصل 14**: The Linux Device Model — فهم الـ driver model اللي بيبني عليه الـ pinctrl

```
المصدر المجاني: https://lwn.net/Kernel/LDD3/
```

#### Linux Kernel Development — Robert Love (3rd Edition)

- **الفصل 17**: Devices and Modules — معمارية الـ device drivers في الـ kernel
- الكتاب ده مفيش فيه فصل مخصص للـ pinctrl (السبسيستم اتضاف بعد الإصدار)، لكن الفهم العام للـ driver model ضروري

#### Embedded Linux Primer — Christopher Hallinan

- **الفصل 15**: Debugging Embedded Linux — بيشمل debugging الـ hardware interfaces
- **الفصل 9**: File Systems — فهم sysfs وdebugfs اللي بيستخدمهم الـ pinctrl

#### Linux Device Driver Development — John Madieu (Packt, 2nd Edition)

- **الفصل 12**: Leveraging the Pinctrl and GPIO Subsystems
- ده أفضل كتاب فيه فصل كامل مخصص للـ pinctrl وبيشرح `pinconf_ops` بالتفصيل

---

### موارد eLinux.org

| الرابط | المحتوى |
|--------|---------|
| [Pin Control GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) | presentation شاملة عن الـ pinctrl وعلاقته بالـ GPIO |
| [EBC Device Trees](https://elinux.org/EBC_Device_Trees) | أمثلة عملية على `pinctrl-single` configuration في الـ device tree |
| [Tests:i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) | اختبارات الـ i2c-demux-pinctrl driver |
| [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) | مجموعة مراجع عامة للـ kernel development |

---

### موارد KernelNewbies.org

الـ KernelNewbies بيتتبع التغييرات في كل إصدار kernel. الصفحات دي بتوضح الـ pinctrl additions في كل release:

| الرابط | الإصدار |
|--------|---------|
| [Linux 6.17](https://kernelnewbies.org/Linux_6.17) | Eswin eic7700 pinctrl، RP1 pinmux/pinconf support |
| [Linux 6.15](https://kernelnewbies.org/Linux_6.15) | SG2042 pinctrl، Allwinner A523، x1600 SoC |
| [Linux 6.13](https://kernelnewbies.org/Linux_6.13) | zynqmp Versal، Canaan K230، Exynos pinctrl drivers |
| [Linux 6.11](https://kernelnewbies.org/Linux_6.11) | تغييرات الـ pinctrl في النسخة دي |
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | كل التغييرات عبر الإصدارات |

---

### مقالات إضافية

| الرابط | الوصف |
|--------|-------|
| [Linux device driver development: The pin control subsystem — Embedded.com](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) | شرح عملي للـ pinctrl subsystem من زاوية الـ embedded development |

---

### Search Terms للبحث عن معلومات أكتر

```
# للـ kernel source
pinconf_ops linux kernel
pinctrl_dev pinconf
pin_config_get pin_config_set implementation
generic pinconf DT parsing

# للـ mailing list (lore.kernel.org)
pinconf_ops site:lore.kernel.org
linux-gpio pinconf

# للـ documentation
linux kernel pin control driver api
pinctrl subsystem architecture

# لأمثلة عملية
pinconf_ops implementation example driver
pinctrl driver tutorial embedded linux
```

**الـ mailing list الرسمي:** `linux-gpio@vger.kernel.org`

**الـ maintainer الحالي:** Linus Walleij — بيراجع كل الـ patches المتعلقة بالـ pinctrl/pinconf
## Phase 8: Writing simple module

### الفكرة

**`pin_config_set`** هي أكتر function مثيرة للاهتمام في `pinconf_ops` — كل ما أي driver بيحاول يغير config لـ pin (pull-up، drive strength، إلخ) بيعدي منها. هنستخدم **kprobe** نعلق على `pin_config_set` في kernel core (`pinconf_set`) عشان نشوف مين بيغير إيه وإمتى.

الـ function المستهدفة في kernel core هي:

```
int pin_config_set(const char *dev_name, const char *name,
                   unsigned long config);
```

دي الـ exported API اللي بتتكلم مع `pinconf_ops->pin_config_set` من جوا.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on pin_config_set — trace every pin configuration change
 * Works with CONFIG_KPROBES=y and CONFIG_PINCTRL=y
 */

/* ---- includes ---- */
#include <linux/module.h>       /* MODULE_*, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/ptrace.h>       /* struct pt_regs */
#include <linux/pinctrl/consumer.h> /* pin_config_set prototype (for context) */

/* ---- kprobe definition ---- */
static struct kprobe kp_pin_cfg = {
    .symbol_name = "pin_config_set",  /* name of kernel symbol to probe */
};

/*
 * pre_handler — called just before pin_config_set executes
 *
 * On x86-64 the first three args land in:
 *   regs->di  → const char *dev_name
 *   regs->si  → const char *name  (pin name)
 *   regs->dx  → unsigned long config
 *
 * config is packed as: bits[7:0]  = enum pin_config_param
 *                       bits[31:8] = argument value
 * (see PIN_CONF_PACKED macro in pinconf-generic.h)
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Extract the three arguments from registers (x86-64 ABI) */
    const char  *dev_name = (const char *)regs->di;
    const char  *pin_name = (const char *)regs->si;
    unsigned long config  = (unsigned long)regs->dx;

    /* Decode the packed config value */
    unsigned int param = (unsigned int)(config & 0xffUL);
    unsigned int arg   = (unsigned int)((config >> 8) & 0xffffffUL);

    pr_info("[pinconf_probe] dev=%s pin=%s param=0x%02x arg=%u\n",
            dev_name  ? dev_name  : "(null)",
            pin_name  ? pin_name  : "(null)",
            param, arg);

    return 0; /* 0 = let the original function continue normally */
}

/* ---- module init ---- */
static int __init pinconf_probe_init(void)
{
    int ret;

    /* Assign the pre-handler then register the kprobe */
    kp_pin_cfg.pre_handler = handler_pre;

    ret = register_kprobe(&kp_pin_cfg);
    if (ret < 0) {
        pr_err("[pinconf_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[pinconf_probe] planted on %s at %p\n",
            kp_pin_cfg.symbol_name, kp_pin_cfg.addr);
    return 0;
}

/* ---- module exit ---- */
static void __exit pinconf_probe_exit(void)
{
    /* Must unregister before the module is unloaded to avoid a dangling hook */
    unregister_kprobe(&kp_pin_cfg);
    pr_info("[pinconf_probe] removed\n");
}

module_init(pinconf_probe_init);
module_exit(pinconf_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe tracer for pin_config_set — pinconf subsystem");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي module: `module_init`، `module_exit`، `MODULE_*` |
| `linux/kernel.h` | `pr_info` / `pr_err` للـ logging |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل API التسجيل |
| `linux/ptrace.h` | `struct pt_regs` اللي بيحمل قيم الـ registers وقت الـ probe |
| `linux/pinctrl/consumer.h` | للـ context البرمجي فقط، مش ضروري وقت الـ compile هنا |

الـ headers دي بتجمع كل اللي المودول محتاجه من غير زيادة.

---

#### الـ `struct kprobe`

```c
static struct kprobe kp_pin_cfg = {
    .symbol_name = "pin_config_set",
};
```

الـ kernel بيبحث عن الـ symbol ده في **kallsyms** وقت `register_kprobe`. لو الـ symbol موجود ومش inlined ومش protected، الـ kprobe بيتزرع بـ **breakpoint instruction** في أول bytة من الـ function.

---

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

**الـ `pre_handler`** بيتنفذ قبل ما الـ function الأصلية تشتغل، فممكن تشوف الـ arguments كاملة قبل ما حاجة تتغير. بنقرأ من الـ registers مباشرةً بناءً على **x86-64 System V ABI** اللي بيحط الـ args الأولانية في `rdi، rsi، rdx`.

الـ config value بتتفكك بنفس منطق macro الـ `PIN_CONF_PACKED`:
- `bits[7:0]` → رقم الـ `pin_config_param` (زي `PIN_CONFIG_BIAS_PULL_UP = 5`)
- `bits[31:8]` → الـ argument (مثلاً قيمة resistance بالـ Ohm)

---

#### الـ `module_init`

```c
kp_pin_cfg.pre_handler = handler_pre;
ret = register_kprobe(&kp_pin_cfg);
```

بنعيّن الـ handler الأول ثم بنسجّل الـ probe. لو `register_kprobe` رجعت قيمة سالبة (مثلاً `-ENOENT` لو الـ symbol مش موجود) المودول بيرجع الـ error مباشرةً ومبيتحملش.

---

#### الـ `module_exit`

```c
unregister_kprobe(&kp_pin_cfg);
```

**لازم** نشيل الـ kprobe قبل ما نفرغ صفحات الـ module من الـ memory. لو ما اتشالش والـ kernel حاول يتنفذ الـ breakpoint، هيلاقي الـ handler على عنوان memory اتمسحت → kernel panic. الـ `unregister_kprobe` بترجّع الـ original bytes وبتضمن إن مفيش execution جارية على الـ handler قبل ما ترجع.

---

### Makefile

```makefile
obj-m += pinconf_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة عملية

```bash
# Build
make

# Load
sudo insmod pinconf_probe.ko

# Trigger a pin config change (e.g. by loading a platform driver or via sysfs)
# Then watch the log:
sudo dmesg | grep pinconf_probe

# Expected output:
# [pinconf_probe] planted on pin_config_set at ffffffff819ab3c0
# [pinconf_probe] dev=soc:pinctrl pin=GPIO5 param=0x05 arg=10000

# Unload
sudo rmmod pinconf_probe
```

> لو `pin_config_set` مش ظاهر في `/proc/kallsyms` يبقى محتاج `CONFIG_KALLSYMS_ALL=y` أو تجرب على `pinconf_set` اللي هي الـ internal wrapper.
