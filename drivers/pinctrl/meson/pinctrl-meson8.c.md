## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ subsystem ده؟

الملف `pinctrl-meson8.c` جزء من **ARM/Amlogic Meson SoC support** subsystem، اللي بيتولى كل حاجة خاصة بـ SoCs من شركة **Amlogic** (مصنّعة لرقائق Meson المستخدمة في صناديق TV وأجهزة Android المدمجة). الـ maintainers موجودين في `linux-amlogic@lists.infradead.org`.

---

### القصة من الأول: ليه محتاجين pinctrl أصلاً؟

تخيّل إنك بتصمم لوحة إلكترونية (PCB). عندك رقاقة **Meson8** في النص، وعايز توصّل بيها:
- شريحة Wi-Fi عن طريق **SDIO**
- شريحة صوت عن طريق **I2S**
- شاشة عن طريق **HDMI**
- ذاكرة flash عن طريق **NAND**
- انترنت عن طريق **Ethernet**

المشكلة: الرقاقة عندها مثلاً 200 pin فيزيائي، لكن عندها آلاف "وظائف" ممكنة (UART، SPI، I2C، SD، PWM...). **مش كل وظيفة ممكن تشتغل على كل pin في نفس الوقت.**

كل pin فيزيائي ممكن يشتغل في واحدة من عدة modes:
1. **GPIO عادي** — input أو output رقمي بسيط
2. **وظيفة خاصة** — زي TX لـ UART، أو MOSI لـ SPI

الـ **Pin Controller** هو الـ driver اللي:
- يعرف كل الـ pins الموجودة في الرقاقة
- يعرف إيه الـ "functions" اللي ممكن تتعمل على كل مجموعة pins
- لما الـ kernel يقول "عايز UART على GPIOX_4 و GPIOX_5"، يروح يكتب في الـ registers الصح عشان يعمل المطلوب

---

### الصورة الكبيرة للملف ده

**`pinctrl-meson8.c`** هو الـ **hardware description file** الخاص برقاقتين:
- **Amlogic Meson8** — رقاقة ARM Cortex-A9 quad-core لصناديق TV
- **Amlogic Meson8m2** — نسخة معدّلة بدعم Gigabit Ethernet كامل

الملف مش فيه كود تنفيذي بالمعنى الحقيقي — هو في الأساس **جدول بيانات ضخم** (data tables) يصف:

| البيانات | الوصف |
|---|---|
| `meson8_cbus_pins[]` | كل الـ pins في الـ CBUS domain (~140 pin) |
| `meson8_aobus_pins[]` | كل الـ pins في الـ AO (Always-On) domain (~16 pin) |
| `meson8_cbus_groups[]` | مجموعات الـ pins مع رقم الـ register والـ bit المقابل |
| `meson8_aobus_groups[]` | نفس الفكرة للـ AO domain |
| `meson8_cbus_functions[]` | الوظائف المتاحة (UART, SPI, I2C, NAND...) |
| `meson8_aobus_functions[]` | الوظائف في الـ AO domain |
| `meson8_cbus_banks[]` | الـ banks مع عناوين registers الـ GPIO |

---

### قصة الـ Two Domains

الـ Meson8 عندها نظام غريب شوية: فيه **domain أساسي (CBUS)** وـ **domain خاص (AO Bus)**:

```
┌─────────────────────────────────────────────┐
│              Meson8 SoC                      │
│                                             │
│  ┌──────────────┐     ┌──────────────────┐  │
│  │  CBUS Domain │     │  AO (Always-On)  │  │
│  │              │     │     Domain       │  │
│  │  ~140 pins   │     │   ~16 pins       │  │
│  │  GPIOX, Y,   │     │  GPIOAO_0..13    │  │
│  │  DV, H, Z,   │     │  GPIO_BSD_EN     │  │
│  │  CARD, BOOT  │     │  GPIO_TEST_N     │  │
│  │              │     │                  │  │
│  │ يتوقف لما    │     │ يشتغل حتى لو    │  │
│  │ الجهاز ينام  │     │ الجهاز نايم!    │  │
│  └──────────────┘     └──────────────────┘  │
└─────────────────────────────────────────────┘
```

الـ AO domain مهم جداً لأنه مسؤول عن أشياء زي:
- استقبال إشارة **remote control** (GPIOAO_7)
- الـ **HDMI CEC** للتحكم في التلفزيون وهو في standby
- الـ **UART** للـ debug حتى في الـ power-save mode

---

### كيف بيشتغل الـ Pin Multiplexing؟

كل group في الـ `meson8_cbus_groups[]` بيحدد:
- اسم الـ group
- الـ pins المنتمية ليه
- رقم الـ **MUX register** والـ **bit** اللي لازم يتكتب فيه `1` لتفعيل الوظيفة

```c
/* مثال: تفعيل UART TX على GPIOX_4 */
GROUP(uart_tx_a0, 4, 17),
/* يعني: register رقم 4، bit رقم 17 */
```

لما driver تاني (مثلاً UART driver) يطلب تفعيل pin معين، الـ `pinctrl-meson.c` (الـ core) بيستخدم الجدول ده ويكتب في الـ register الصح.

---

### الـ Banks وإزاي بتتحكم في GPIO

كل **bank** في `meson8_cbus_banks[]` بيصف مجموعة pins متجاورة وعناوين الـ registers الخاصة بيها:

```c
/*        name  first    last       irq       pullen  pull    dir     out     in   */
BANK("X", GPIOX_0, GPIOX_21, 112, 133, 4,0, 4,0, 0,0, 1,0, 2,0),
```

يعني لـ GPIOX bank:
- **pullen register**: رقم 4، offset 0 — لتفعيل pull-up/down
- **pull register**: رقم 4، offset 0 — لتحديد الاتجاه (up أو down)
- **dir register**: رقم 0، offset 0 — input أم output
- **out register**: رقم 1، offset 0 — لكتابة قيمة
- **in register**: رقم 2، offset 0 — لقراءة قيمة

---

### الوظائف المتاحة على الـ Meson8

الملف بيعرّف أكثر من 30 وظيفة للـ CBUS:

| الوظيفة | الـ Pins المستخدمة |
|---|---|
| `sd_a`, `sd_b`, `sd_c` | SDIO cards على GPIOX, CARD, BOOT |
| `nand` | NAND flash على BOOT pins |
| `nor` | SPI NOR flash على BOOT pins |
| `ethernet` | Ethernet على GPIOZ |
| `hdmi` | HDMI على GPIOH |
| `uart_a`, `uart_b`, `uart_c` | UART على GPIOX, GPIOY, GPIODV |
| `i2c_a`, `i2c_b`, `i2c_c`, `i2c_d` | I2C على أماكن متعددة |
| `spi` | SPI على GPIOH و GPIOZ |
| `i2s`, `spdif` | Audio على GPIOY |
| `dvin` | Digital Video Input على GPIODV |
| `enc` | Video Encoder على GPIODV |
| `pwm_a` .. `pwm_e` | PWM على pins متنوعة |

---

### لماذا نفس الـ GPIO يظهر في أكثر من function؟

مثال حقيقي: `GPIOX_4` ممكن يكون:
- `pcm_out_a` — صوت PCM
- `uart_tx_a0` — UART transmit
- أو plain GPIO

ده الـ **pin multiplexing** في أبسط صورة. الـ Device Tree بيقول للـ kernel إيه الـ function المطلوبة، والـ pinctrl driver بيطبّقها.

---

### ملفات الـ Subsystem

#### الـ Core Framework
| الملف | الدور |
|---|---|
| `drivers/pinctrl/meson/pinctrl-meson.c` | الـ core driver: probe، GPIO ops، pin config |
| `drivers/pinctrl/meson/pinctrl-meson.h` | structs وmacros مشتركة لكل Meson SoCs |
| `drivers/pinctrl/meson/pinctrl-meson8-pmx.c` | منطق الـ pinmux الخاص بالجيل الأول (Meson8) |
| `drivers/pinctrl/meson/pinctrl-meson8-pmx.h` | macros لتعريف GROUP وGPIO_GROUP |

#### ملفات الـ Hardware Description (كل SoC بملفه)
| الملف | الرقاقة |
|---|---|
| `pinctrl-meson8.c` | **Meson8 و Meson8m2** ← الملف الحالي |
| `pinctrl-meson8b.c` | Meson8b |
| `pinctrl-meson-gxbb.c` | Meson GXBB |
| `pinctrl-meson-gxl.c` | Meson GXL |
| `pinctrl-meson-axg.c` | Meson AXG |
| `pinctrl-meson-g12a.c` | Meson G12A |
| `pinctrl-meson-a1.c` | Meson A1 |
| `pinctrl-meson-s4.c` | Meson S4 |
| `pinctrl-amlogic-c3.c` | Amlogic C3 |
| `pinctrl-amlogic-a4.c` | Amlogic A4 |

#### الـ Device Tree Bindings
| الملف | الوصف |
|---|---|
| `Documentation/devicetree/bindings/pinctrl/amlogic,meson8-pinctrl-cbus.yaml` | DT binding للـ CBUS |
| `Documentation/devicetree/bindings/pinctrl/amlogic,meson8-pinctrl-aobus.yaml` | DT binding للـ AO Bus |
| `Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl-common.yaml` | البنية المشتركة |

#### الـ GPIO Headers
| الملف | الوصف |
|---|---|
| `include/dt-bindings/gpio/meson8-gpio.h` | أرقام الـ GPIO macros (GPIOX_0, GPIOAO_0...) |

---

### ملخص بجملة واحدة

**الـ `pinctrl-meson8.c`** هو "خريطة الأسلاك" الكاملة لرقاقة Meson8 — بيقول للـ Linux kernel إيه كل pin، وعلى أي register تتحكم فيه، وإيه الوظائف المتاحة عليه — والـ core في `pinctrl-meson.c` هو اللي بيستخدم الخريطة دي فعلياً.
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة — ليه الـ Pinctrl موجود؟

أي SoC حديث زي الـ Amlogic Meson8 فيه مئات الـ physical pins. الـ pin الواحد ممكن يشتغل بأكتر من وظيفة في نفس الوقت على مستوى الـ hardware — مثلاً `GPIOX_4` ممكن يكون:
- `uart_tx_a0` (UART transmit)
- `pcm_out_a` (PCM audio output)
- `sdxc_d47_a` (SDXC data line)

الـ hardware بيحدد ده عن طريق **mux registers** — bits في registers معينة بتختار الوظيفة. المشكلة إن:

1. كل SoC له registers مختلفة وبتخطيط (layout) مختلف تماماً.
2. الـ drivers (UART, SPI, I2C, SDMMC) محتاجين يطلبوا الـ pins دول من غير ما يعرفوا تفاصيل الـ hardware.
3. في نفس الوقت محتاج الـ GPIO subsystem يتعامل مع نفس الـ pins لما مفيش driver بيستخدمهم.
4. لازم حاجة تمنع درايفرين يتنازعوا على نفس الـ pin.

بدون framework مركزي، كل driver كان هيكتب كود hardware-specific بنفسه — فوضى كاملة.

---

### الحل — الـ Pinctrl Framework

الكيرنل حل المشكلة دي عن طريق **abstraction layer** كامل اسمه `pinctrl`. الفكرة:

- فيه **pin controller driver** واحد لكل SoC (أو لكل bus domain في الـ SoC).
- الـ driver ده بيسجّل نفسه مع الـ pinctrl core عن طريق `pinctrl_desc`.
- الـ consumer drivers (UART, SPI, etc.) بيطلبوا states بالاسم ("default", "sleep") من الـ DT، والـ core بيترجمها لـ register writes.

الـ framework بيدير ثلاث عمليات أساسية:
| العملية | المعنى |
|---------|--------|
| **pinmux** | اختيار وظيفة الـ pin (UART أو SPI أو GPIO) |
| **pinconf** | ضبط خصائص الـ pin (pull-up/down, drive strength, slew rate) |
| **GPIO** | تحكم مباشر في الـ pin كـ input/output |

---

### التشبيه الواقعي — لوحة المفاتيح متعددة الاستخدام

تخيل لوحة مفاتيح موسيقية (keyboard) فيها 128 مفتاح. كل مفتاح ممكن:
- يعزف نوت موسيقية (وظيفة GPIO)
- يكون جزء من chord محدد (وظيفة UART/SPI/I2C)
- يتحكم في volume/pitch (pin configuration)

| المفهوم الموسيقي | المقابل في الكيرنل |
|---|---|
| المفتاح الفيزيائي | الـ `pin` — `pinctrl_pin_desc` |
| الـ chord (مجموعة مفاتيح معاً) | الـ `group` — `meson_pmx_group` |
| نوع الموسيقى (جاز، كلاسيك) | الـ `function` — `meson_pmx_func` |
| زر التحديد على اللوحة | الـ **mux register bit** |
| من بيقرر أيه اللي هيتعزف | الـ **pinctrl core** |
| المؤلف الموسيقي | الـ **consumer driver** (بيقول محتاج chord X) |
| المهندس الصوتي | الـ **pin controller driver** (بيعمل الـ wiring الفعلي) |
| قسم الآلات الوترية | الـ **CBUS domain** |
| قسم الآلات الإيقاعية | الـ **AOBUS domain** |

الـ consumer driver مش محتاج يعرف إيه الـ register اللي هيكتب فيه — بس بيقول "محتاج UART function" والـ framework بيترتب الباقي.

---

### الـ Big Picture Architecture

```
 ┌─────────────────────────────────────────────────────────────┐
 │                    Consumer Drivers                         │
 │   [UART]  [SPI]  [I2C]  [SDMMC]  [ETH]  [I2S]  [HDMI]    │
 └──────────────────────┬──────────────────────────────────────┘
                        │  pinctrl_get() / pinctrl_select_state()
                        ▼
 ┌─────────────────────────────────────────────────────────────┐
 │                   PINCTRL CORE                              │
 │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
 │  │  pinmux core │  │ pinconf core │  │  GPIO framework  │  │
 │  │  (pin mux    │  │  (pull/drive │  │  integration     │  │
 │  │   selection) │  │   strength)  │  │  gpiolib         │  │
 │  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
 └─────────┼────────────────┼───────────────────┼─────────────┘
           │                │                   │
           ▼                ▼                   ▼
 ┌─────────────────────────────────────────────────────────────┐
 │          Pin Controller Driver (HW-specific)                │
 │                                                             │
 │   ┌──────────────────────┐  ┌──────────────────────────┐   │
 │   │  meson8_cbus_pinctrl │  │  meson8_aobus_pinctrl    │   │
 │   │  (pinctrl-meson8.c)  │  │  (pinctrl-meson8.c)      │   │
 │   │                      │  │                          │   │
 │   │  pins: X,Y,DV,H,Z,   │  │  pins: AO_0..AO_13,     │   │
 │   │        CARD,BOOT      │  │        BSD_EN, TEST_N    │   │
 │   │  banks: 7 banks      │  │  banks: 1 bank (AO)      │   │
 │   │  funcs: 33 functions │  │  funcs: 9 functions      │   │
 │   └──────────┬───────────┘  └──────────────┬───────────┘   │
 └──────────────┼────────────────────────────┼────────────────┘
                │                            │
                ▼                            ▼
 ┌──────────────────────────┐  ┌─────────────────────────────┐
 │  CBUS Registers (MMIO)   │  │  AOBUS Registers (MMIO)     │
 │  reg_mux[0..9]           │  │  reg_mux[0]                 │
 │  reg_pullen, reg_pull    │  │  reg_pullen, reg_pull       │
 │  reg_gpio (dir/in/out)   │  │  reg_gpio                   │
 └──────────────────────────┘  └─────────────────────────────┘
                │                            │
                ▼                            ▼
 ┌─────────────────────────────────────────────────────────────┐
 │           Amlogic Meson8 SoC Hardware                       │
 │  Physical Pins: GPIOX, GPIOY, GPIODV, GPIOH, GPIOZ,        │
 │                 CARD, BOOT, GPIOAO                          │
 └─────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — التجريد الأساسي

الـ framework بيقلب التفكير من "أنا driver وأنا بكتب في registers مباشرة" إلى **declarative model**:

> "أنا driver محتاج الـ `uart_a function`. الـ pinctrl core يرتبلي ده."

الـ core abstraction بيتكون من ثلاث طبقات:

#### 1. الـ Pin (الوحدة الأساسية)

```c
/* pinctrl.h — الـ pin الفيزيائي */
struct pinctrl_pin_desc {
    unsigned int number;  /* رقم unique في الـ global space */
    const char   *name;   /* اسم نصي زي "GPIOX_4" */
    void         *drv_data; /* driver-specific data */
};

/* pinctrl-meson8.c — كيف بيتعرف الـ pin */
#define MESON_PIN(x) PINCTRL_PIN(x, #x)

static const struct pinctrl_pin_desc meson8_cbus_pins[] = {
    MESON_PIN(GPIOX_0),   /* number=0, name="GPIOX_0" */
    MESON_PIN(GPIOX_1),
    /* ... 133 pin في الـ cbus domain */
    MESON_PIN(BOOT_18),
};
```

الـ pin number هو index عالمي داخل الـ controller — مش رقم GPIO.

#### 2. الـ Group (مجموعة pins بتشتغل مع بعض)

```c
/* pinctrl-meson.h */
struct meson_pmx_group {
    const char        *name;      /* "sd_d0_a" */
    const unsigned int *pins;     /* array of pin numbers */
    unsigned int       num_pins;  /* عدد الـ pins */
    const void        *data;      /* mux register info */
};
```

الـ group بيقول "الـ function دي محتاجة الـ pins دول مع بعض". مثلاً:

```c
/* pinctrl-meson8.c */
static const unsigned int sdxc_d47_a_pins[] = {
    GPIOX_4, GPIOX_5, GPIOX_6, GPIOX_7  /* 4 pins مع بعض */
};

/* الـ GROUP macro بيربط الـ pins بالـ mux register */
GROUP(sdxc_d47_a, 5, 12),
/*               ^  ^
                 |  bit 12 في الـ register
                 register #5 في الـ mux regmap */
```

#### 3. الـ Function (الوظيفة اللي بتجمع groups)

```c
/* pinctrl-meson.h */
struct meson_pmx_func {
    const char       *name;       /* "sdxc_a" */
    const char * const *groups;   /* ["sdxc_d0_a", "sdxc_d13_a", ...] */
    unsigned int      num_groups;
};

/* pinctrl-meson8.c */
static const char * const sdxc_a_groups[] = {
    "sdxc_d0_a", "sdxc_d13_a", "sdxc_d47_a",
    "sdxc_clk_a", "sdxc_cmd_a"
};

FUNCTION(sdxc_a),
/* يتوسع لـ: { .name="sdxc_a",
               .groups=sdxc_a_groups,
               .num_groups=5 } */
```

الـ function بتقول "لو حد طلب SDXC-A، ممكن يستخدم أي من الـ groups دول".

---

### الـ Bank — طبقة التجريد للـ GPIO

الـ `meson_bank` هي الـ abstraction الخاصة بـ Meson لمجموعة pins بتتحكم فيها بنفس الـ registers:

```c
/* pinctrl-meson.h */
struct meson_reg_desc {
    unsigned int reg;  /* offset في الـ regmap */
    unsigned int bit;  /* بداية الـ bits للـ pin الأول في الـ bank */
};

struct meson_bank {
    const char *name;       /* "X", "Y", "BOOT", "AO", ... */
    unsigned int first;     /* أول pin number */
    unsigned int last;      /* آخر pin number */
    int irq_first;          /* أول hwirq رقم */
    int irq_last;
    struct meson_reg_desc regs[MESON_NUM_REG];
    /* regs[PULLEN] = {reg, bit} لتفعيل الـ pull resistor
       regs[PULL]   = {reg, bit} لاتجاه الـ pull (up/down)
       regs[DIR]    = {reg, bit} لاتجاه الـ pin (input/output)
       regs[OUT]    = {reg, bit} للكتابة على الـ pin
       regs[IN]     = {reg, bit} للقراءة من الـ pin
       regs[DS]     = {reg, bit} للـ drive strength */
};
```

مثال فعلي من `pinctrl-meson8.c`:

```c
BANK("X",    GPIOX_0,  GPIOX_21,  112, 133,
/*   name    first     last       irq_first irq_last
     pullen_reg  pullen_bit  pull_reg  pull_bit
     dir_reg     dir_bit     out_reg   out_bit   in_reg  in_bit  */
     4,  0,   4,  0,   0,  0,   1,  0,   2,  0),
/*   ^   ^
     |   bit offset داخل الـ register (bit 0 = أول pin في الـ bank)
     register number في الـ regmap
*/
```

الـ pin `GPIOX_4` مثلاً هو رقم 4 من بداية الـ bank X، فبيتحكم فيه بـ bit 4 في كل register.

---

### العلاقة بين الـ Structs — رسم تفصيلي

```
meson_pinctrl_data
├── pins[]          ─── pinctrl_pin_desc[]
│                       ├── .number = GPIOX_0
│                       └── .name   = "GPIOX_0"
│
├── groups[]        ─── meson_pmx_group[]
│                       ├── .name     = "sd_d0_a"
│                       ├── .pins     = {GPIOX_0}
│                       └── .data     ─── meson8_pmx_data
│                                         ├── .reg = 8
│                                         ├── .bit = 5
│                                         └── .is_gpio = false
│
├── funcs[]         ─── meson_pmx_func[]
│                       ├── .name      = "sd_a"
│                       └── .groups    = ["sd_d0_a", "sd_d1_a", ...]
│
├── banks[]         ─── meson_bank[]
│                       ├── .name   = "X"
│                       ├── .first  = GPIOX_0
│                       ├── .last   = GPIOX_21
│                       └── .regs[] = {PULLEN, PULL, DIR, OUT, IN, DS}
│
└── pmx_ops         ─── pinmux_ops (meson8_pmx_ops)
                        ├── .get_functions_count
                        ├── .get_function_name
                        ├── .get_function_groups
                        └── .set_mux  ← بيكتب في الـ mux register
```

---

### الـ Two Domains — CBUS و AOBUS

الـ Meson8 فيه **domain انفصال فيزيائي** مهم جداً:

```
┌──────────────────────────────────────────────────────┐
│                 Meson8 SoC                           │
│                                                      │
│  ┌─────────────────────┐   ┌───────────────────────┐ │
│  │    CBUS Domain      │   │    AOBUS Domain       │ │
│  │  (always-on bus)    │   │  (always-on bus)      │ │
│  │                     │   │                       │ │
│  │  GPIOX  (22 pins)   │   │  GPIOAO_0..13 (14p)  │ │
│  │  GPIOY  (17 pins)   │   │  GPIO_BSD_EN          │ │
│  │  GPIODV (30 pins)   │   │  GPIO_TEST_N          │ │
│  │  GPIOH  (10 pins)   │   │                       │ │
│  │  GPIOZ  (15 pins)   │   │  يشتغل حتى لو        │ │
│  │  CARD   (7 pins)    │   │  الـ main power off   │ │
│  │  BOOT   (19 pins)   │   │                       │ │
│  │                     │   │  Functions:           │ │
│  │  = 120 pins total   │   │  UART-AO, IR remote,  │ │
│  │                     │   │  I2C-AO, HDMI-CEC,    │ │
│  │  Compatible:        │   │  I2S-AO, PWM-AO       │ │
│  │  amlogic,meson8-    │   │                       │ │
│  │  cbus-pinctrl       │   │  Compatible:          │ │
│  │                     │   │  amlogic,meson8-      │ │
│  │                     │   │  aobus-pinctrl        │ │
│  └─────────────────────┘   └───────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

كل domain بيتسجّل كـ `platform_driver` مستقل — ليه؟ لأن كل domain له **regmap منفصل**. الـ `meson_pinctrl` struct بيحتفظ بـ 5 regmaps مختلفة:

```c
/* pinctrl-meson.h */
struct meson_pinctrl {
    struct device          *dev;
    struct pinctrl_dev     *pcdev;    /* handle من الـ pinctrl core */
    struct pinctrl_desc     desc;     /* الـ descriptor اللي بيتسجّل */
    struct meson_pinctrl_data *data;  /* الـ SoC-specific data */
    struct regmap  *reg_mux;          /* mux function registers */
    struct regmap  *reg_pullen;       /* pull enable registers */
    struct regmap  *reg_pull;         /* pull direction registers */
    struct regmap  *reg_gpio;         /* GPIO dir/in/out registers */
    struct regmap  *reg_ds;           /* drive strength registers */
    struct gpio_chip chip;            /* GPIO chip للـ gpiolib */
    struct fwnode_handle *fwnode;
};
```

**الـ regmap subsystem** (مهم يعرفه القارئ): abstraction layer فوق MMIO registers — بيوفر caching، locking، وـ debug access موحد.

---

### الـ Mux Mechanism — كيف بيتم اختيار الوظيفة؟

الـ `GROUP` macro بيضم **رقم register** و**رقم bit** لكل function group:

```c
/* pinctrl-meson8-pmx.h */
#define GROUP(grp, r, b)
    .data = PMX_DATA(r, b, false)
    /* r = register number في الـ mux regmap
       b = bit number داخل الـ register
       false = مش GPIO mode */

/* مثال:
   GROUP(sd_d0_a, 8, 5)
   → لتفعيل SD data-0 على bank-A:
     اكتب 1 في bit[5] من register[8] في الـ mux regmap */

/* GPIO_GROUP:
   GPIO_GROUP(GPIOX_0)
   → PMX_DATA(0, 0, true)
   → is_gpio=true يعني مش محتاج تكتب في mux registers
     بس تخلي الـ OEN (Output Enable Not) في الـ DIR register */
```

لما الـ consumer driver بيطلب `sd_a` function على الـ group `sd_d0_a`:
1. الـ pinctrl core بيستدعي `meson8_pmx_ops.set_mux()`
2. اللي بيقرأ الـ `meson8_pmx_data` من الـ group
3. ويكتب `1` في `reg[8]` bit `5` في الـ `reg_mux` regmap
4. الـ hardware دلوقتي بيروت الـ pin لـ SD controller

---

### الـ BANK Macro — تفكيك الأرقام

```c
BANK("Z", GPIOZ_0, GPIOZ_14, 14, 28,
     1,  0,   /* pullen: reg=1, bit_offset=0  */
     1,  0,   /* pull:   reg=1, bit_offset=0  */
     3, 17,   /* dir:    reg=3, bit_offset=17 */
     4, 17,   /* out:    reg=4, bit_offset=17 */
     5, 17),  /* in:     reg=5, bit_offset=17 */
```

لو عايز تقرأ قيمة `GPIOZ_3` (هو رقم 3 من بداية الـ bank):
```
pin_in_bank = GPIOZ_3 - GPIOZ_0 = 3
actual_bit  = in.bit + pin_in_bank = 17 + 3 = bit 20
register    = reg_gpio[in.reg] = reg_gpio[5]
→ اقرأ bit[20] من register[5]
```

---

### الـ of_device_id — ربط الـ DT بالـ Driver

```c
/* pinctrl-meson8.c */
static const struct of_device_id meson8_pinctrl_dt_match[] = {
    {
        .compatible = "amlogic,meson8-cbus-pinctrl",
        .data = &meson8_cbus_pinctrl_data,  /* ← static data للـ cbus */
    },
    {
        .compatible = "amlogic,meson8-aobus-pinctrl",
        .data = &meson8_aobus_pinctrl_data, /* ← static data للـ aobus */
    },
    {
        .compatible = "amlogic,meson8m2-cbus-pinctrl",
        .data = &meson8_cbus_pinctrl_data,  /* Meson8m2 نفس الـ data */
    },
    { },
};
```

لما الـ kernel يلاقي `amlogic,meson8-cbus-pinctrl` في الـ Device Tree، بيبعت الـ `meson8_cbus_pinctrl_data` struct للـ `probe` function، اللي هي `meson_pinctrl_probe` المشتركة بين كل أجيال الـ Meson SoCs.

---

### ملكية الـ Framework مقابل الـ Driver

| المهمة | الـ pinctrl core يملكها | الـ meson8 driver يملكها |
|--------|------------------------|--------------------------|
| تتبع مين بيستخدم أيه pin | نعم | لا |
| منع تعارض الـ pins | نعم | لا |
| parse الـ DT `pinctrl-0` في consumer | نعم | لا |
| تعريف الـ pins والـ groups والـ functions | لا | نعم |
| كتابة في الـ mux registers | لا | نعم (`meson8_pmx_ops`) |
| ضبط الـ pull/drive strength | لا | نعم (عبر `pinconf_ops`) |
| تسجيل الـ GPIO chip | لا | نعم (عبر `gpio_chip`) |
| تفسير DT nodes خاصة بالـ SoC | لا | نعم (`parse_dt` callback) |
| الـ `pinctrl_desc` registration | تستقبله | تبنيه وتبعته |

---

### خلاصة — الفكرة المركزية

الـ pinctrl framework بيقدم **فصل كامل** بين:
- **من بيطلب** (consumer driver: UART يطلب pins)
- **ماذا يطلب** (function + group بالاسم، مش registers)
- **كيف يتنفذ** (pin controller driver: يكتب في الـ mux register الصح)

الـ `pinctrl-meson8.c` هو **pure data file** — بيعرّف الـ topology الكاملة للـ Meson8 SoC كـ static tables، والـ `pinctrl-meson.c` (الـ core Meson driver) هو اللي فيه الـ logic الفعلي. ده نموذج تصميم ممتاز: **data-driven driver** بيفصل الـ hardware description عن الـ behavior.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Macros — Cheatsheet

#### enum meson_reg_type — أنواع السجلات

| القيمة | المعنى | الوظيفة |
|--------|--------|---------|
| `MESON_REG_PULLEN` | Pull-Enable | تفعيل/تعطيل الـ pull resistor |
| `MESON_REG_PULL` | Pull Direction | up أو down |
| `MESON_REG_DIR` | Direction | input أو output |
| `MESON_REG_OUT` | Output Value | كتابة قيمة GPIO |
| `MESON_REG_IN` | Input Value | قراءة قيمة GPIO |
| `MESON_REG_DS` | Drive Strength | قوة التيار (drive current) |
| `MESON_NUM_REG` | Count | عدد الأنواع = 6 |

#### enum meson_pinconf_drv — قيم الـ Drive Strength

| القيمة | التيار |
|--------|--------|
| `MESON_PINCONF_DRV_500UA` | 500 µA |
| `MESON_PINCONF_DRV_2500UA` | 2500 µA |
| `MESON_PINCONF_DRV_3000UA` | 3000 µA |
| `MESON_PINCONF_DRV_4000UA` | 4000 µA |

#### الـ Macros المهمة

| الـ Macro | الاستخدام | الموقع |
|-----------|-----------|--------|
| `MESON_PIN(x)` | ينشئ `pinctrl_pin_desc` من اسم GPIO | `pinctrl-meson.h` |
| `BANK(n,f,l,fi,li,...)` | ينشئ `meson_bank` بدون DS | `pinctrl-meson.h` |
| `BANK_DS(...)` | ينشئ `meson_bank` مع DS | `pinctrl-meson.h` |
| `GROUP(grp,r,b)` | ينشئ `meson_pmx_group` لـ function pin group | `pinctrl-meson8-pmx.h` |
| `GPIO_GROUP(gpio)` | ينشئ group لـ GPIO خالص (`is_gpio=true`) | `pinctrl-meson8-pmx.h` |
| `FUNCTION(fn)` | ينشئ `meson_pmx_func` من اسم function | `pinctrl-meson.h` |
| `PMX_DATA(r,b,g)` | ينشئ `meson8_pmx_data` (reg, bit, is_gpio) | `pinctrl-meson8-pmx.h` |

#### GPIO Banks في Meson8 — Cheatsheet

| Bank | أول Pin | آخر Pin | عدد Pins | IRQ Base | الوظيفة الرئيسية |
|------|---------|---------|----------|----------|-----------------|
| X | GPIOX_0 | GPIOX_21 | 22 | 112 | SD/SDXC/UART/PCM/I2C |
| Y | GPIOY_0 | GPIOY_16 | 17 | 95 | UART/PCM/I2S/SPDIF |
| DV | GPIODV_0 | GPIODV_29 | 30 | 65 | DVIN/ENC/VGA/UART |
| H | GPIOH_0 | GPIOH_9 | 10 | 29 | HDMI/SPI/I2C |
| Z | GPIOZ_0 | GPIOZ_14 | 15 | 14 | ETH/SPI/I2C/PWM |
| CARD | CARD_0 | CARD_6 | 7 | 58 | SD/SDXC card slot |
| BOOT | BOOT_0 | BOOT_18 | 19 | 39 | NAND/NOR/SD/SDXC |
| AO | GPIOAO_0 | GPIO_TEST_N | 16 | 0 | Always-On: UART/I2C/I2S/Remote |

---

### 1. الـ Structs المهمة

#### 1.1 `struct pinctrl_pin_desc`

**الغرض:** يصف pin واحد — اسمه ورقمه الفريد في النظام.

| Field | النوع | المعنى |
|-------|-------|--------|
| `number` | `unsigned int` | رقم الـ pin (enum من dt-bindings) |
| `name` | `const char *` | اسم الـ pin نصياً مثل `"GPIOX_0"` |
| `drv_data` | `void *` | بيانات إضافية (غير مستخدمة هنا) |

**الـ Macro المستخدم:**
```c
MESON_PIN(GPIOX_0)
/* يتوسع إلى: PINCTRL_PIN(GPIOX_0, "GPIOX_0") */
```

**الاستخدام في الملف:** المصفوفتان `meson8_cbus_pins[]` و `meson8_aobus_pins[]` — إجمالي ~135 pin.

---

#### 1.2 `struct meson8_pmx_data`

**الغرض:** البيانات الخاصة بـ Meson8 المرتبطة بكل group — تحدد مكان بت الـ mux في الـ register.

```c
struct meson8_pmx_data {
    bool         is_gpio;  /* true = هذا GPIO خالص، مش function */
    unsigned int reg;      /* رقم الـ register في الـ mux regmap */
    unsigned int bit;      /* رقم البت داخل الـ register */
};
```

**مثال حقيقي:**
```c
/* GROUP(sd_d0_a, 8, 5) يتوسع إلى: */
{
    .name = "sd_d0_a",
    .pins = sd_d0_a_pins,          /* = { GPIOX_0 } */
    .num_pins = 1,
    .data = (const struct meson8_pmx_data[]){
        PMX_DATA(8, 5, false),     /* reg=8, bit=5, ليس GPIO */
    },
}
```
**الـ GPIO_GROUP:**
```c
/* GPIO_GROUP(GPIOX_0) يتوسع إلى: */
{
    .name = "GPIOX_0",
    .pins = (const unsigned int[]){ GPIOX_0 },
    .num_pins = 1,
    .data = (const struct meson8_pmx_data[]){
        PMX_DATA(0, 0, true),      /* is_gpio=true */
    },
}
```

---

#### 1.3 `struct meson_pmx_group`

**الغرض:** يمثل "مجموعة pins" ترتبط بـ function واحدة. كل group عندها اسم وقائمة pins وبيانات mux.

```c
struct meson_pmx_group {
    const char         *name;      /* مثل "sd_d0_a" أو "GPIOX_0" */
    const unsigned int *pins;      /* مصفوفة أرقام الـ pins */
    unsigned int        num_pins;  /* عدد الـ pins في المجموعة */
    const void         *data;      /* يشير لـ meson8_pmx_data */
};
```

**الاستخدام:** مصفوفتا `meson8_cbus_groups[]` (حوالي 220 group) و `meson8_aobus_groups[]` (32 group).

كل group نوعان:
- **GPIO_GROUP:** لـ pin واحد — `is_gpio=true` — يعني الـ pin شغال GPIO خالص
- **GROUP:** لـ function مثل `sd_d0_a` — `is_gpio=false` — مع reg+bit للـ mux

---

#### 1.4 `struct meson_pmx_func`

**الغرض:** يمثل "وظيفة" كاملة (مثل UART، SPI، Ethernet) — تجمع عدة groups تحت اسم واحد.

```c
struct meson_pmx_func {
    const char         *name;       /* مثل "ethernet" */
    const char * const *groups;     /* مصفوفة أسماء الـ groups */
    unsigned int        num_groups; /* عددها */
};
```

**مثال:**
```c
/* FUNCTION(ethernet) يتوسع إلى: */
{
    .name = "ethernet",
    .groups = ethernet_groups,   /* = { "eth_tx_clk_50m", "eth_tx_en", ... } */
    .num_groups = ARRAY_SIZE(ethernet_groups),
}
```

الـ cbus عندها 33 function، الـ aobus عندها 9 functions.

---

#### 1.5 `struct meson_reg_desc`

**الغرض:** يصف موقع بت واحد في الـ regmap — يستخدم لوصف مكان كل عملية (pull, dir, out, in).

```c
struct meson_reg_desc {
    unsigned int reg;  /* offset في الـ regmap */
    unsigned int bit;  /* رقم البت */
};
```

---

#### 1.6 `struct meson_bank`

**الغرض:** يصف "bank" كاملة من الـ GPIO — تحدد نطاق الـ pins وكل الـ registers المسؤولة عن control.

```c
struct meson_bank {
    const char          *name;                    /* "X", "Y", "AO" ... */
    unsigned int         first;                   /* رقم أول pin */
    unsigned int         last;                    /* رقم آخر pin */
    int                  irq_first;               /* أول hwirq */
    int                  irq_last;                /* آخر hwirq */
    struct meson_reg_desc regs[MESON_NUM_REG];    /* 6 entries: PULLEN/PULL/DIR/OUT/IN/DS */
};
```

**مثال حقيقي (Bank X):**
```c
BANK("X", GPIOX_0, GPIOX_21, 112, 133,
     4,  0,   /* PULLEN: reg=4, bit=0  */
     4,  0,   /* PULL:   reg=4, bit=0  */
     0,  0,   /* DIR:    reg=0, bit=0  */
     1,  0,   /* OUT:    reg=1, bit=0  */
     2,  0)   /* IN:     reg=2, bit=0  */
```
كل رقم (reg, bit) يشير إلى **أول pin في الـ bank** — للـ pin رقم N في الـ bank نضيف (N - first) للـ bit.

---

#### 1.7 `struct meson_pinctrl_data`

**الغرض:** الـ "descriptor" الجامع لكل بيانات الـ pinctrl الثابتة الخاصة بـ SoC معينة.

```c
struct meson_pinctrl_data {
    const char                   *name;        /* "cbus-banks" أو "ao-bank" */
    const struct pinctrl_pin_desc *pins;        /* قائمة كل الـ pins */
    const struct meson_pmx_group  *groups;      /* قائمة كل الـ groups */
    const struct meson_pmx_func   *funcs;       /* قائمة كل الـ functions */
    unsigned int                   num_pins;
    unsigned int                   num_groups;
    unsigned int                   num_funcs;
    const struct meson_bank       *banks;       /* قائمة الـ banks */
    unsigned int                   num_banks;
    const struct pinmux_ops       *pmx_ops;     /* pointer لـ ops table */
    const void                    *pmx_data;    /* extra data */
    int (*parse_dt)(struct meson_pinctrl *pc);  /* callback لـ DT parsing */
};
```

في الملف عندنا نسختان:
- `meson8_cbus_pinctrl_data` — للـ cbus (البنوك X/Y/DV/H/Z/CARD/BOOT)
- `meson8_aobus_pinctrl_data` — للـ aobus (البنك AO) مع `parse_dt = meson8_aobus_parse_dt_extra`

---

#### 1.8 `struct meson_pinctrl`

**الغرض:** الـ runtime state للـ driver — ينشأ عند الـ probe ويحتفظ بكل الـ regmaps والـ GPIO chip.

```c
struct meson_pinctrl {
    struct device              *dev;        /* الـ platform device */
    struct pinctrl_dev         *pcdev;      /* handle من الـ pinctrl core */
    struct pinctrl_desc         desc;       /* descriptor للـ pinctrl core */
    struct meson_pinctrl_data  *data;       /* pointer للـ static data أعلاه */
    struct regmap              *reg_mux;    /* regmap لـ mux registers */
    struct regmap              *reg_pullen; /* regmap لـ pull-enable */
    struct regmap              *reg_pull;   /* regmap لـ pull direction */
    struct regmap              *reg_gpio;   /* regmap لـ GPIO (dir/out/in) */
    struct regmap              *reg_ds;     /* regmap لـ drive strength */
    struct gpio_chip            chip;       /* الـ GPIO chip للـ gpiolib */
    struct fwnode_handle       *fwnode;     /* الـ DT node */
};
```

---

### 2. مخططات العلاقات بين الـ Structs

```
مستوى الـ Static Data (compile-time):
═══════════════════════════════════════

meson8_cbus_pinctrl_data
┌─────────────────────────────┐
│ name = "cbus-banks"         │
│ *pins ──────────────────────┼──→ meson8_cbus_pins[]
│                             │      [pinctrl_pin_desc × 133]
│ *groups ────────────────────┼──→ meson8_cbus_groups[]
│                             │      [meson_pmx_group × ~220]
│                             │        ↓ each group has:
│                             │        └─ *data → meson8_pmx_data
│                             │                  {reg, bit, is_gpio}
│ *funcs ─────────────────────┼──→ meson8_cbus_functions[]
│                             │      [meson_pmx_func × 33]
│                             │        ↓ each func has:
│                             │        └─ **groups → string names
│ *banks ─────────────────────┼──→ meson8_cbus_banks[]
│                             │      [meson_bank × 7]
│                             │        ↓ each bank has:
│                             │        └─ regs[6]: {PULLEN,PULL,DIR,OUT,IN,DS}
│ *pmx_ops ───────────────────┼──→ meson8_pmx_ops (from pinctrl-meson8-pmx.c)
│ parse_dt = NULL             │
└─────────────────────────────┘

meson8_aobus_pinctrl_data
┌─────────────────────────────┐
│ name = "ao-bank"            │
│ *pins ──────────────────────┼──→ meson8_aobus_pins[] [16 pins]
│ *groups ────────────────────┼──→ meson8_aobus_groups[] [~32 groups]
│ *funcs ─────────────────────┼──→ meson8_aobus_functions[] [9 funcs]
│ *banks ─────────────────────┼──→ meson8_aobus_banks[] [1 bank: "AO"]
│ *pmx_ops ───────────────────┼──→ meson8_pmx_ops
│ parse_dt ───────────────────┼──→ meson8_aobus_parse_dt_extra()
└─────────────────────────────┘


مستوى الـ Runtime State (probe-time):
═══════════════════════════════════════

platform_device
     │
     │ probe()
     ▼
meson_pinctrl  (heap-allocated)
┌──────────────────────────────────┐
│ *dev ────────────────────────────┼──→ platform_device.dev
│ *data ───────────────────────────┼──→ meson8_cbus_pinctrl_data (أو aobus)
│                                  │      (من of_device_get_match_data)
│ desc ────────────────────────────┼──→ pinctrl_desc
│   .name                          │      (يُبنى من data->name)
│   .pins = data->pins             │
│   .npins = data->num_pins        │
│   .pctlops                       │
│   .pmxops = data->pmx_ops        │
│   .confops                       │
│                                  │
│ *pcdev ──────────────────────────┼──→ pinctrl_dev  (من pinctrl_register)
│                                  │
│ *reg_mux ────────────────────────┼──→ regmap (MMIO لـ mux bits)
│ *reg_pullen ─────────────────────┼──→ regmap (MMIO لـ pull-enable)
│ *reg_pull ───────────────────────┼──→ regmap (MMIO لـ pull direction)
│ *reg_gpio ───────────────────────┼──→ regmap (MMIO لـ dir/out/in)
│ *reg_ds ─────────────────────────┼──→ regmap (MMIO لـ drive strength)
│                                  │
│ chip ────────────────────────────┼──→ gpio_chip (embedded)
│   .label                         │
│   .ngpio = num pins in bank      │
│   .base                          │
│   .get/set/direction ops         │
│                                  │
│ *fwnode ─────────────────────────┼──→ DT node handle
└──────────────────────────────────┘


علاقة meson_pmx_group ↔ meson_bank:
═════════════════════════════════════

meson_pmx_group "sd_d0_a"          meson_bank "X"
┌──────────────────┐               ┌────────────────────────────┐
│ pins = {GPIOX_0} │               │ first = GPIOX_0            │
│ data:            │               │ last  = GPIOX_21           │
│  reg = 8         │               │ regs[MESON_REG_DIR].reg=0  │
│  bit = 5         │               │ regs[MESON_REG_OUT].reg=1  │
│  is_gpio = false │               │ regs[MESON_REG_IN ].reg=2  │
└──────────────────┘               └────────────────────────────┘
         │                                    │
         │ mux: write 1 to reg_mux[8] bit 5  │ gpio: read/write via reg_gpio
         ▼                                    ▼
    Hardware MUX Register              Hardware GPIO Register
```

---

### 3. مخططات الـ Lifecycle

#### 3.1 تسجيل الـ Driver

```
kernel boot
    │
    ▼
builtin_platform_driver(meson8_pinctrl_driver)
    │  يسجل meson8_pinctrl_driver في الـ platform bus
    │
    ▼
DT matching: of_match_table = meson8_pinctrl_dt_match[]
    │
    │  compatible = "amlogic,meson8-cbus-pinctrl"
    │      → .data = &meson8_cbus_pinctrl_data
    │
    │  compatible = "amlogic,meson8-aobus-pinctrl"
    │      → .data = &meson8_aobus_pinctrl_data
    │
    │  compatible = "amlogic,meson8m2-cbus-pinctrl"
    │      → .data = &meson8_cbus_pinctrl_data  (نفس البيانات!)
    │
    │  compatible = "amlogic,meson8m2-aobus-pinctrl"
    │      → .data = &meson8_aobus_pinctrl_data (نفس البيانات!)
    │
    ▼
meson8_pinctrl_driver.probe = meson_pinctrl_probe()
```

#### 3.2 الـ Probe Flow

```
meson_pinctrl_probe(pdev)
    │
    ├─ devm_kzalloc → ينشئ struct meson_pinctrl *pc
    │
    ├─ of_device_get_match_data → pc->data = &meson8_cbus_pinctrl_data
    │
    ├─ pc->data->parse_dt(pc)   [فقط للـ aobus: meson8_aobus_parse_dt_extra]
    │      └─ يجلب الـ AO regmaps من الـ DT
    │
    ├─ يبني regmaps من الـ DT resources:
    │      pc->reg_mux, pc->reg_pullen, pc->reg_pull
    │      pc->reg_gpio, pc->reg_ds
    │
    ├─ يبني pc->desc من pc->data:
    │      desc.name = data->name
    │      desc.pins = data->pins
    │      desc.npins = data->num_pins
    │      desc.pmxops = data->pmx_ops
    │
    ├─ devm_pinctrl_register(dev, &pc->desc, pc)
    │      └─ يسجل في الـ pinctrl subsystem → يعطي pc->pcdev
    │
    └─ meson_gpiolib_register(pc)
           └─ يسجل pc->chip في الـ gpiolib
```

#### 3.3 دورة حياة الـ Pin Function

```
[CREATE]                    [REGISTER]                 [USE]
pin arrays defined    →   pinctrl_register()     →  consumer driver calls:
(compile-time static)      builds internal DB        pinctrl_get()
                                                      pinctrl_lookup_state()
                                                      pinctrl_select_state()
                                                           │
                                                           ▼
                                                      pmx_ops->set_mux()
                                                           │
                                                           ▼
                                                      regmap_update_bits(
                                                        reg_mux,
                                                        group->data->reg,
                                                        BIT(group->data->bit),
                                                        BIT(group->data->bit))

[TEARDOWN]
platform_driver_unregister()
  → pinctrl_unregister()
  → gpiochip_remove()
```

---

### 4. مخططات الـ Call Flow

#### 4.1 تفعيل Function على Pin

```
Consumer driver (e.g. MMC driver)
    │
    │ pinctrl_select_state(state)
    ▼
pinctrl core
    │ iterates pmx settings
    ▼
pmx_ops->set_mux(pcdev, func_selector, group_selector)
    │   [meson8_pmx_ops.set_mux]
    │
    ├─ meson_pmx_group *group = &data->groups[group_selector]
    ├─ meson8_pmx_data *pmx_data = group->data
    │
    ├─ if pmx_data->is_gpio:
    │      return 0  (nothing to do for plain GPIO mode)
    │
    └─ regmap_update_bits(
           pc->reg_mux,
           pmx_data->reg * 4,        /* register offset */
           BIT(pmx_data->bit),       /* mask */
           BIT(pmx_data->bit))       /* value: set the bit */
               │
               ▼
         Hardware MUX register bit set
         → pin now routed to peripheral function


مثال حقيقي: تفعيل sd_d0_a على GPIOX_0
    GROUP(sd_d0_a, 8, 5)
    → reg_mux[8], bit 5 = 1
    → GPIOX_0 يبدأ يشتغل كـ SD Data0
```

#### 4.2 GPIO Read/Write Flow

```
GPIO consumer
    │
    │ gpiod_get_value(desc)
    ▼
gpiolib core
    │
    ▼
gpio_chip->get(chip, offset)
    │   [meson_gpio_get]
    │
    ├─ meson_get_bank(pc, offset, &bank)
    │      └─ loops meson_bank[] checking first <= offset <= last
    │
    ├─ bit = offset - bank->first + bank->regs[MESON_REG_IN].bit
    ├─ reg = bank->regs[MESON_REG_IN].reg
    │
    └─ regmap_read(pc->reg_gpio, reg * 4, &val)
           └─ return (val >> bit) & 1


GPIO Write (gpiod_set_value):
    └─ gpio_chip->set(chip, offset, value)
           └─ regmap_update_bits(pc->reg_gpio,
                  bank->regs[MESON_REG_OUT].reg * 4,
                  BIT(bit), value ? BIT(bit) : 0)


GPIO Direction (gpiod_direction_output):
    └─ gpio_chip->direction_output(chip, offset, value)
           ├─ regmap_update_bits(reg_gpio, DIR_reg, BIT(bit), 0) /* 0=output */
           └─ set value via OUT register
```

#### 4.3 Pin Config (Pull / Drive Strength)

```
pinctrl_select_state()
    │
    ▼
confops->pin_config_set(pcdev, pin, configs, num_configs)
    │
    ├─ PIN_CONFIG_BIAS_PULL_UP:
    │      regmap_update_bits(reg_pullen, bank->regs[PULLEN].reg, bit, 1)
    │      regmap_update_bits(reg_pull,   bank->regs[PULL].reg,   bit, 1)
    │
    ├─ PIN_CONFIG_BIAS_PULL_DOWN:
    │      regmap_update_bits(reg_pullen, ..., 1)  /* enable */
    │      regmap_update_bits(reg_pull,   ..., 0)  /* direction=down */
    │
    ├─ PIN_CONFIG_BIAS_DISABLE:
    │      regmap_update_bits(reg_pullen, ..., 0)  /* disable */
    │
    └─ PIN_CONFIG_DRIVE_STRENGTH_UA:
           regmap_update_bits(reg_ds, bank->regs[DS].reg, 2-bit field, value)
```

---

### 5. استراتيجية الـ Locking

#### 5.1 ما الذي يحتاج lock؟

**الـ Meson8 pinctrl لا يعرّف locks خاصة بيه** — يعتمد كلياً على الـ locking المدمج في طبقتين:

| الطبقة | آلية الـ Lock | ما تحميه |
|--------|--------------|---------|
| **pinctrl core** | `mutex` داخلي في `pinctrl_dev` | تسجيل/تغيير الـ states، الـ `set_mux` calls |
| **regmap** | `spinlock` أو `mutex` حسب الـ regmap config | الكتابة/القراءة من الـ MMIO registers |
| **gpiolib** | `spinlock` في `gpio_chip` | عمليات GPIO المتزامنة |

#### 5.2 ترتيب الـ Locking (Lock Ordering)

```
pinctrl_state_select()
    │
    │ يحمل: pinctrl_dev->mutex
    ▼
pmx_ops->set_mux()
    │
    │ يحمل: regmap internal lock
    ▼
regmap_update_bits()
    │
    ▼
hardware register write
    (لا يوجد interrupt context هنا — كلها sleepable context)
```

#### 5.3 ملاحظات مهمة عن الـ Locking

- **الـ static data** (`meson8_cbus_pinctrl_data`, arrays) هي `const` ومعرّفة compile-time — لا تحتاج lock لأنها read-only.
- **الـ regmaps** تحتوي على الـ locking الداخلي — الـ driver لا يحتاج يعمل lock صريح قبل `regmap_update_bits`.
- **الـ aobus** وعندها DT parsing إضافي في `meson8_aobus_parse_dt_extra` — يشتغل في probe context (single-threaded) فمفيش race condition.
- **الـ GPIO interrupts**: الـ `irq_first` و `irq_last` في `meson_bank` يحددوا نطاق الـ hardware IRQs — الـ irq chip المرتبط يتعامل مع الـ locking بتاعه بشكل منفصل.
- **مفيش spinlock في الـ hot path** — كل العمليات تعمل في sleepable context وده مناسب لأن الـ regmap يعتمد على `regmap_mmio` اللي ممكن يستخدم `spinlock` داخلياً للـ atomic access.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ Macros — Cheatsheet

#### Data Description Macros

| الـ Macro | الـ Signature المبسط | الغرض |
|-----------|----------------------|--------|
| `MESON_PIN` | `MESON_PIN(x)` | يُنشئ `pinctrl_pin_desc` من اسم الـ pin |
| `BANK` | `BANK(name, first, last, irq_first, irq_last, ...)` | يُعرّف bank كامل بمعامل registers |
| `BANK_DS` | `BANK_DS(..., dsr, dsb)` | نفس `BANK` مع إضافة drive-strength register |
| `GROUP` | `GROUP(grp, reg, bit)` | يُنشئ `meson_pmx_group` لـ function group |
| `GPIO_GROUP` | `GPIO_GROUP(gpio)` | يُنشئ group خاص بـ GPIO بدون function bit |
| `FUNCTION` | `FUNCTION(fn)` | يُنشئ `meson_pmx_func` يربط اسم الـ function بمصفوفة groups |
| `PMX_DATA` | `PMX_DATA(r, b, g)` | يُنشئ `meson8_pmx_data` بـ register/bit/is_gpio |

#### Runtime PMX Functions

| الـ Function | الـ Context | الغرض |
|-------------|-------------|--------|
| `meson8_pmx_disable_other_groups` | internal, sleepable | تعطيل كل groups تستخدم نفس الـ pin |
| `meson8_pmx_set_mux` | pinmux core callback | تفعيل function group على pin |
| `meson8_pmx_request_gpio` | gpio core callback | تحويل pin لـ GPIO mode |

#### Common Functions (من pinctrl-meson.c، تُستخدم هنا)

| الـ Function | الغرض |
|-------------|--------|
| `meson_pmx_get_funcs_count` | إرجاع عدد الـ functions |
| `meson_pmx_get_func_name` | إرجاع اسم function بـ index |
| `meson_pmx_get_groups` | إرجاع groups الـ function |
| `meson_pinctrl_probe` | الـ probe المشترك للـ driver |
| `meson8_aobus_parse_dt_extra` | parse إضافي خاص بـ AO bus |

---

### Category 1: Static Data Tables — تعريف الـ Pins والـ Banks

هذه المجموعة مش functions بالمعنى الحرفي — هي **static const data structures** يُعرِّفها الـ driver لتصف topology الـ SoC. الـ pinctrl core يقرأها عند الـ probe. الـ Meson8 driver مقسّم لـ domain اتنين: **cbus** (معظم pins الـ SoC) و**aobus** (الـ Always-On domain).

---

#### `MESON_PIN(x)` — Pin Descriptor Macro

```c
/* expands to: */
PINCTRL_PIN(x, #x)
/* which creates: */
struct pinctrl_pin_desc { .number = x, .name = #x }
```

بيُنشئ entry واحد في مصفوفة `pinctrl_pin_desc`. الـ `x` هو الـ enum value للـ pin (مُعرَّف في `meson8-gpio.h`)، والـ name بيتحوّل لـ string literal تلقائيًا. الـ pinctrl core بيستخدم الـ descriptor list دي لما يُسجّل الـ controller.

**المصفوفات المُعرَّفة:**

| المصفوفة | عدد الـ Pins | الـ Domain |
|----------|-------------|-----------|
| `meson8_cbus_pins[]` | 133 pin | CBUS (X, Y, DV, H, Z, CARD, BOOT) |
| `meson8_aobus_pins[]` | 16 pin | AO (GPIOAO_0..13 + BSD_EN + TEST_N) |

---

#### `BANK(...)` / `BANK_DS(...)` — Bank Definition Macro

```c
#define BANK(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib) \
    BANK_DS(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib, 0, 0)

#define BANK_DS(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib, dsr, dsb) \
    { .name = n, .first = f, .last = l,                    \
      .irq_first = fi, .irq_last = li,                     \
      .regs = {                                             \
          [MESON_REG_PULLEN] = { per, peb },                \
          [MESON_REG_PULL]   = { pr,  pb  },                \
          [MESON_REG_DIR]    = { dr,  db  },                \
          [MESON_REG_OUT]    = { or,  ob  },                \
          [MESON_REG_IN]     = { ir,  ib  },                \
          [MESON_REG_DS]     = { dsr, dsb },                \
      } }
```

كل **bank** هو مجموعة pins بيتحكم فيها بمنطقة متصلة من registers. الـ macro بيملا `struct meson_bank` بـ register/bit descriptors لكل عملية (pull-enable، pull-direction، GPIO direction، output، input، drive-strength).

**مثال — Bank X من meson8_cbus_banks:**
```c
BANK("X", GPIOX_0, GPIOX_21, 112, 133, 4, 0, 4, 0, 0, 0, 1, 0, 2, 0)
/*   name  first    last      irq_f irq_l
     pullen_reg=4, pullen_bit=0
     pull_reg=4,   pull_bit=0
     dir_reg=0,    dir_bit=0
     out_reg=1,    out_bit=0
     in_reg=2,     in_bit=0  */
```

**الـ Banks المُعرَّفة:**

| Bank | First Pin | Last Pin | HW IRQ Range | الـ Domain |
|------|-----------|----------|-------------|-----------|
| "X"  | GPIOX_0   | GPIOX_21 | 112–133 | CBUS |
| "Y"  | GPIOY_0   | GPIOY_16 | 95–111  | CBUS |
| "DV" | GPIODV_0  | GPIODV_29| 65–94   | CBUS |
| "H"  | GPIOH_0   | GPIOH_9  | 29–38   | CBUS |
| "Z"  | GPIOZ_0   | GPIOZ_14 | 14–28   | CBUS |
| "CARD"| CARD_0   | CARD_6   | 58–64   | CBUS |
| "BOOT"| BOOT_0   | BOOT_18  | 39–57   | CBUS |
| "AO" | GPIOAO_0  | GPIO_TEST_N | 0–13 | AO  |

---

#### `GROUP(grp, reg, bit)` — Peripheral Function Group Macro

```c
#define GROUP(grp, r, b) \
    { .name = #grp,                          \
      .pins = grp ## _pins,                  \
      .num_pins = ARRAY_SIZE(grp ## _pins),  \
      .data = (const struct meson8_pmx_data[]){ PMX_DATA(r, b, false) } }
```

بيُنشئ entry في `meson_pmx_group`. الـ `data` بيشير لـ `meson8_pmx_data` بيحدد أي register وأي bit لازم يتفعّل عشان تشتغل الـ function دي. الـ `false` معناها إنها مش GPIO group — فيها mux bit فعلي.

**مثال:**
```c
GROUP(uart_tx_a0, 4, 17)
/* name="uart_tx_a0", pins={GPIOX_4},
   reg_mux[4*4] bit17 = 1 to enable */
```

---

#### `GPIO_GROUP(gpio)` — GPIO-Only Group Macro

```c
#define GPIO_GROUP(gpio) \
    { .name = #gpio,                                    \
      .pins = (const unsigned int[]){ gpio },           \
      .num_pins = 1,                                    \
      .data = (const struct meson8_pmx_data[]){         \
          PMX_DATA(0, 0, true) } }
```

كل pin عنده GPIO group بـ `is_gpio = true`. ده بيقول للـ pmx layer إن في حالة الـ GPIO مش لازم تكتب أي bit — بس تمسح كل الـ function bits التانية.

---

#### `FUNCTION(fn)` — Pinmux Function Macro

```c
#define FUNCTION(fn) \
    { .name = #fn,                             \
      .groups = fn ## _groups,                 \
      .num_groups = ARRAY_SIZE(fn ## _groups) }
```

بيُنشئ `meson_pmx_func` بيربط اسم function (زي `"uart_a"`) بمصفوفة الـ strings اللي بتحدد الـ groups المتاحة. الـ pinctrl core بيعرضهم لـ userspace عبر `pinctrl-handles` في الـ DT.

**الـ Functions المُعرَّفة في CBUS:**

| الـ Function | عدد الـ Groups | الـ Interface |
|-------------|--------------|-------------|
| `gpio_periphs` | 133 | GPIO mode لكل CBUS pins |
| `sd_a/b/c` | 6 لكل | SD card controller |
| `sdxc_a/b/c` | 5 لكل | SDXC (eMMC-compatible) |
| `uart_a/b/c` | 4/8/4 | UART controllers |
| `i2c_a/b/c/d` | 2–6 | I2C buses |
| `spi` | 10 | SPI controllers (0 و1) |
| `ethernet` | 14 | RMII/GMII Ethernet |
| `nand` | 11 | Raw NAND flash |
| `nor` | 4 | SPI NOR flash |
| `hdmi` | 4 | HDMI HPD/DDC/CEC |
| `pwm_a..e` | 1–3 لكل | PWM channels |
| `enc` | 18 | Video encoder bus |
| `dvin` | 5 | Digital video input |
| `i2s` | 8 | I2S audio |
| `spdif` | 2 | S/PDIF audio |
| `iso7816` | 4 | Smart card |
| `xtal` | 2 | Crystal clock output |
| `vga` | 2 | VGA sync |

**الـ Functions المُعرَّفة في AO Bus:**

| الـ Function | الـ Interface |
|-------------|-------------|
| `gpio_aobus` | GPIO لـ AO pins |
| `uart_ao` | UART_AO A |
| `uart_ao_b` | UART_AO B |
| `remote` | IR remote TX/RX |
| `i2c_slave_ao` | I2C slave في AO |
| `i2c_mst_ao` | I2C master في AO |
| `pwm_f_ao` | PWM_F channel |
| `i2s_ao` | I2S في AO domain |
| `hdmi_cec_ao` | HDMI CEC في AO |

---

### Category 2: PMX Runtime Functions — تغيير الـ Pinmux

دي الـ functions الـ runtime اللي الـ pinctrl core بيستدعيها لما يحتاج يغير وظيفة pin.

---

#### `meson8_pmx_disable_other_groups()`

```c
static void meson8_pmx_disable_other_groups(struct meson_pinctrl *pc,
                                            unsigned int pin,
                                            int sel_group)
```

**ما بتعمله:** بتعمل iterate على كل `num_groups` groups في الـ controller، وبتمسح الـ mux bit لأي group (غير GPIO groups وغير `sel_group`) إنها تستخدم نفس الـ `pin`. ده بيضمن إن pin ما يكونش مفعّل في أكتر من function في نفس الوقت.

**الـ Parameters:**

| الـ Parameter | النوع | الوصف |
|-------------|-------|--------|
| `pc` | `struct meson_pinctrl *` | الـ controller state، بيحتوي على `reg_mux` regmap |
| `pin` | `unsigned int` | الـ global pin number اللي محتاج نمسح groups بتاعته |
| `sel_group` | `int` | index الـ group المختار (يتجاهله)، أو `-1` لمسح الكل |

**الـ Return:** `void`

**Key Details:**
- بيعمل `regmap_update_bits(pc->reg_mux, pmx_data->reg * 4, BIT(pmx_data->bit), 0)` — الـ `* 4` لأن الـ register offset بالـ word units لكن `regmap` بيتعامل بالـ bytes
- بيتجاهل groups اللي `pmx_data->is_gpio == true` (ما فيهاش mux bit حقيقي)
- بيتجاهل الـ `sel_group` المختار عشان يتجنب glitches

**Pseudocode:**
```
for each group i in pc->data->groups:
    pmx_data = group[i].data
    if pmx_data->is_gpio OR i == sel_group:
        continue
    for each pin_j in group[i].pins:
        if pin_j == pin:
            regmap_update_bits(reg_mux, pmx_data->reg*4,
                               BIT(pmx_data->bit), 0)  // clear
```

**Who Calls It:**
- `meson8_pmx_set_mux()` — لكل pin في الـ group المختار
- `meson8_pmx_request_gpio()` — بـ `sel_group = -1` لمسح الكل

---

#### `meson8_pmx_set_mux()`

```c
static int meson8_pmx_set_mux(struct pinctrl_dev *pcdev,
                              unsigned func_num,
                              unsigned group_num)
```

**ما بتعمله:** ده الـ callback الأساسي للـ pinmux. بيستدعيه الـ pinctrl core لما يطبّق `pinctrl-handle` من الـ DT أو لما يُستدعى `pinctrl_select_state()`. بيعمل الخطوات التالية:
1. لكل pin في الـ group المختار، يعطّل كل الـ groups التانية اللي بتستخدم نفس الـ pin
2. لو الـ function مش GPIO (func_num != 0)، يُفعّل الـ mux bit الخاص بالـ group

**الـ Parameters:**

| الـ Parameter | النوع | الوصف |
|-------------|-------|--------|
| `pcdev` | `struct pinctrl_dev *` | الـ pinctrl device handle |
| `func_num` | `unsigned` | index الـ function في `pc->data->funcs[]` |
| `group_num` | `unsigned` | index الـ group في `pc->data->groups[]` |

**الـ Return:** `0` نجاح، أو error code من `regmap_update_bits()`

**Key Details:**
- `pinctrl_dev_get_drvdata(pcdev)` بيجيب `meson_pinctrl *pc`
- لـ `func_num == 0` (GPIO)، بس بيمسح الـ conflicting groups بدون ما يكتب أي bit
- الـ bit بيتكتب بـ `regmap_update_bits(pc->reg_mux, pmx_data->reg * 4, BIT(pmx_data->bit), BIT(pmx_data->bit))` — set bit
- الـ locking بيتم في الـ pinctrl core قبل استدعاء الـ callback

**Pseudocode:**
```
pc = pinctrl_dev_get_drvdata(pcdev)
func = pc->data->funcs[func_num]
group = pc->data->groups[group_num]
pmx_data = group->data

for each pin in group->pins:
    meson8_pmx_disable_other_groups(pc, pin, group_num)

if func_num != 0:  // not GPIO
    regmap_update_bits(pc->reg_mux,
                       pmx_data->reg * 4,
                       BIT(pmx_data->bit),
                       BIT(pmx_data->bit))  // set
return ret
```

**Who Calls It:**
- الـ pinctrl core عبر `pinmux_ops.set_mux` عند تطبيق state

---

#### `meson8_pmx_request_gpio()`

```c
static int meson8_pmx_request_gpio(struct pinctrl_dev *pcdev,
                                   struct pinctrl_gpio_range *range,
                                   unsigned offset)
```

**ما بتعمله:** لما GPIO subsystem يطلب pin عبر `gpio_request()` أو `gpiod_get()`، الـ pinctrl core بيستدعي الـ callback ده. بيمسح كل الـ function groups اللي بتستخدم الـ pin ده (بـ `sel_group = -1`)، فيتحول الـ pin لـ GPIO mode بشكل نظيف.

**الـ Parameters:**

| الـ Parameter | النوع | الوصف |
|-------------|-------|--------|
| `pcdev` | `struct pinctrl_dev *` | الـ pinctrl device handle |
| `range` | `struct pinctrl_gpio_range *` | الـ GPIO range (مش بيُستخدم مباشرة هنا) |
| `offset` | `unsigned` | الـ GPIO offset = global pin number |

**الـ Return:** دايمًا `0` (ما فيش error path)

**Key Details:**
- بيمرر `sel_group = -1` لـ `meson8_pmx_disable_other_groups()` — يعني بيمسح **كل** الـ function groups
- ما بيكتبش أي GPIO-specific register — ده بيتم بعدين في `gpio_chip.direction_input/output`
- لو الـ pin اتستخدم قبل كده في function، الـ function بتتوقف فورًا

**Who Calls It:**
- `pinctrl_gpio_request()` من الـ GPIO subsystem

---

### Category 3: Exported PMX Operations Table

#### `meson8_pmx_ops` — `struct pinmux_ops`

```c
const struct pinmux_ops meson8_pmx_ops = {
    .set_mux             = meson8_pmx_set_mux,
    .get_functions_count = meson_pmx_get_funcs_count,
    .get_function_name   = meson_pmx_get_func_name,
    .get_function_groups = meson_pmx_get_groups,
    .gpio_request_enable = meson8_pmx_request_gpio,
};
EXPORT_SYMBOL_GPL(meson8_pmx_ops);
```

الـ `meson8_pmx_ops` هو الـ vtable اللي بتسجّله `meson_pinctrl_data` عبر حقل `pmx_ops`. الـ pinctrl core بيستخدمه لاستدعاء الـ callbacks الصح وقت الـ runtime.

| الـ Op | الـ Implementation | الغرض |
|--------|-------------------|--------|
| `set_mux` | `meson8_pmx_set_mux` | تطبيق function على group |
| `get_functions_count` | `meson_pmx_get_funcs_count` | عدد الـ functions (من pinctrl-meson.c) |
| `get_function_name` | `meson_pmx_get_func_name` | اسم function بـ index |
| `get_function_groups` | `meson_pmx_get_groups` | groups الخاصة بـ function |
| `gpio_request_enable` | `meson8_pmx_request_gpio` | تفعيل GPIO mode |

الـ `meson8_pmx_ops` بيُضاف في `meson8_cbus_pinctrl_data.pmx_ops` و`meson8_aobus_pinctrl_data.pmx_ops`، وبينتقل للـ `pinctrl_desc` في `meson_pinctrl_probe()`.

---

### Category 4: Platform Driver Registration

#### `meson8_pinctrl_dt_match[]` — OF Device ID Table

```c
static const struct of_device_id meson8_pinctrl_dt_match[] = {
    { .compatible = "amlogic,meson8-cbus-pinctrl",
      .data = &meson8_cbus_pinctrl_data },
    { .compatible = "amlogic,meson8-aobus-pinctrl",
      .data = &meson8_aobus_pinctrl_data },
    { .compatible = "amlogic,meson8m2-cbus-pinctrl",
      .data = &meson8_cbus_pinctrl_data },
    { .compatible = "amlogic,meson8m2-aobus-pinctrl",
      .data = &meson8_aobus_pinctrl_data },
    { },
};
```

**ما بتعمله:** بتربط الـ DT compatible strings بـ pinctrl_data الصح. الـ Meson8 والـ Meson8m2 بيشتركوا نفس الـ data (الفرق الوحيد إن Meson8m2 بيدعم `eth_rxd2/3/txd2/3` اللي موجودة في نفس الجدول بـ comment "only available on Meson8m2").

**الـ compatible Strings:**

| الـ Compatible | الـ SoC | الـ Domain |
|---------------|---------|-----------|
| `amlogic,meson8-cbus-pinctrl` | Meson8 | Main pinctrl |
| `amlogic,meson8-aobus-pinctrl` | Meson8 | Always-On |
| `amlogic,meson8m2-cbus-pinctrl` | Meson8m2 | Main pinctrl |
| `amlogic,meson8m2-aobus-pinctrl` | Meson8m2 | Always-On |

---

#### `meson8_pinctrl_driver` + `builtin_platform_driver()`

```c
static struct platform_driver meson8_pinctrl_driver = {
    .probe  = meson_pinctrl_probe,
    .driver = {
        .name           = "meson8-pinctrl",
        .of_match_table = meson8_pinctrl_dt_match,
    },
};
builtin_platform_driver(meson8_pinctrl_driver);
```

**ما بتعمله:** بيسجّل الـ `platform_driver` مع الـ kernel. الـ `builtin_platform_driver()` macro بيعمل `platform_driver_register()` كـ `device_initcall` — يعني الـ driver بيتسجّل في مرحلة early boot قبل `module_init`. ده مهم لأن الـ pinctrl لازم يكون جاهز قبل ما أي device تاني يطلب pins.

**الـ Probe Function:** `meson_pinctrl_probe` (في pinctrl-meson.c) — بياخد الـ `of_device_id.data` (اللي هو `meson_pinctrl_data *`) وبيسجّل الـ pinctrl controller مع الـ core.

---

### Category 5: الـ Static Pin Group Arrays — Data Topology

#### Pin Arrays (مثال)

```c
static const unsigned int uart_tx_a0_pins[] = { GPIOX_4 };
static const unsigned int sdxc_d13_a_pins[] = { GPIOX_1, GPIOX_2, GPIOX_3 };
```

كل array بتُعرِّف الـ physical pins اللي بتنتمي لـ pinmux group واحد. الـ `GROUP()` macro بيشير ليهم بـ `## _pins` pattern. معظم functions بيكون ليها pin واحد بس، لكن بعض interfaces زي SDXC و NAND بيكون ليها مجموعة.

#### مصفوفات الـ Function Groups (مثال)

```c
static const char * const uart_a_groups[] = {
    "uart_tx_a0", "uart_rx_a0", "uart_cts_a0", "uart_rts_a0",
    "uart_tx_a1", "uart_rx_a1", "uart_cts_a1", "uart_rts_a1"
};
```

دي مصفوفة strings بتُعرِّف كل الـ pin groups اللي ممكن تتستخدم لـ function معينة. الـ consumer في الـ DT بيختار subsets منها عبر `pinctrl-0 = <&uart_a_handle>`.

---

### ملاحظات معمارية مهمة

```
DT Node (compatible: "amlogic,meson8-cbus-pinctrl")
       │
       ▼
platform_driver.probe → meson_pinctrl_probe()
       │
       ├── reads .data → meson8_cbus_pinctrl_data
       │       ├── .pins   → meson8_cbus_pins[]     (133 pins)
       │       ├── .groups → meson8_cbus_groups[]   (GPIO + function groups)
       │       ├── .funcs  → meson8_cbus_functions[] (33 functions)
       │       ├── .banks  → meson8_cbus_banks[]    (7 banks)
       │       └── .pmx_ops → &meson8_pmx_ops
       │
       └── registers pinctrl_dev with pinctrl core
               │
               ├── pinmux request → meson8_pmx_set_mux()
               │       └── meson8_pmx_disable_other_groups()
               │               └── regmap_update_bits(reg_mux)
               │
               └── gpio_request → meson8_pmx_request_gpio()
                       └── meson8_pmx_disable_other_groups(sel=-1)
```

**الـ Register Addressing:** الـ `pmx_data->reg` هو word index — بيُضرب في 4 للوصول للـ byte address في الـ regmap. ده consistent مع Meson hardware اللي registers بتاعته 32-bit aligned.

**الـ AO Domain الخصوصية:** الـ `meson8_aobus_pinctrl_data` عنده `parse_dt = &meson8_aobus_parse_dt_extra` — ده callback إضافي بيُستدعى من `meson_pinctrl_probe()` عشان يعمل map لـ registers الخاصة بـ AO domain اللي بتيجي من source تانية في الـ DT (مش نفس regmap الـ CBUS).
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

**الـ pinctrl subsystem** بيعرض معلومات تفصيلية في `/sys/kernel/debug/pinctrl/`:

```bash
# اعرف الـ pinctrl devices الموجودة
ls /sys/kernel/debug/pinctrl/

# مثال واقعي على Meson8:
# /sys/kernel/debug/pinctrl/cbus-banks/
# /sys/kernel/debug/pinctrl/ao-bank/
```

| المدخل | المحتوى | كيفية القراءة |
|--------|---------|---------------|
| `pinctrl-maps` | كل الـ mappings من DT nodes لـ pins | يبيّن أي function اتعين لأي group |
| `pins` | كل الـ pins مع أسمائها ورقمها | مفيد للتأكد إن الـ pin موجود صح |
| `pinmux-functions` | الـ functions المتاحة (uart_a, i2c_a, ...) | يأكد تسجيل الـ function صح |
| `pinmux-pins` | أي pin مضبوط على أي function | هنا تشوف conflict بين devices |
| `pinconf-pins` | الـ configuration الحالية (pull, drive) | لو pull-up/pull-down غلط |
| `pinconf-groups` | نفس الـ pinconf لكن على مستوى group | |

```bash
# أهم أمر عملي — شوف الـ muxing الحالي لكل pin
cat /sys/kernel/debug/pinctrl/cbus-banks/pinmux-pins

# شوف إذا في conflict
cat /sys/kernel/debug/pinctrl/cbus-banks/pinmux-pins | grep -v "GPIO"

# شوف الـ pin configs (pull-up/pull-down/drive)
cat /sys/kernel/debug/pinctrl/cbus-banks/pinconf-pins
```

**مثال على output طبيعي:**
```
pin 0 (GPIOX_0): meson8-pinctrl gpio chip
pin 9 (GPIOX_9): device platform:mmc0 function sd_a group sd_cmd_a
pin 8 (GPIOX_8): device platform:mmc0 function sd_a group sd_clk_a
```

لو شايف `pin X: UNCLAIMED` — يعني الـ pin مش متخصصلوش وممكن يكون GPIO فعلياً أو الـ driver ما probe-اش صح.

---

#### 2. مدخلات الـ sysfs

```bash
# GPIO chip info
ls /sys/class/gpio/
cat /sys/class/gpio/gpiochip0/label    # "meson8-gpio" أو ما شابه
cat /sys/class/gpio/gpiochip0/base     # أول رقم GPIO في الـ chip
cat /sys/class/gpio/gpiochip0/ngpio    # عدد الـ GPIOs

# لو الـ GPIO متصدَّر (exported)
echo 10 > /sys/class/gpio/export       # صدّر GPIOX_10 (رقم = base + offset)
cat /sys/class/gpio/gpio10/direction   # in/out
cat /sys/class/gpio/gpio10/value       # 0/1
```

**الـ regulator/power domain** المرتبط بالـ AO bank:
```bash
# AO bank دايماً powered — بس تأكد
cat /sys/kernel/debug/regulator/regulator_summary | grep -i ao
```

---

#### 3. استخدام الـ ftrace

**الـ tracepoints المهمة:**

```bash
# فعّل الـ tracing على pinctrl events
echo 1 > /sys/kernel/debug/tracing/tracing_on

# مشاهدة أي function من pinctrl بتتنادى
echo "meson_*" > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace

# أو استخدم events
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

**tracepoints مفيدة:**
```bash
ls /sys/kernel/debug/tracing/events/gpio/
# gpio_value      — كل مرة بتتغير قيمة GPIO
# gpio_direction  — كل مرة بيتغير اتجاه GPIO
```

```bash
# trace الـ regmap operations على مسجلات الـ pinctrl
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/enable
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_read/enable

# فلتر على الـ meson8 pinctrl regmap بس
echo 'name == "meson8-cbus"' > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/filter
cat /sys/kernel/debug/tracing/trace_pipe
```

**مثال على output مفيد:**
```
mmc0-worker-1234 [0] regmap_reg_write: meson8-cbus reg=20 val=0x00080000
# reg=20 hex = register 8 (×4) → هو الـ MUX register 8 → bank X
# val=0x00080000 → bit 19 set → sd_cmd_a enabled
```

---

#### 4. الـ printk والـ dynamic debug

```bash
# فعّل dynamic debug على كل كود pinctrl-meson
echo "file pinctrl-meson.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file pinctrl-meson8.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file pinctrl-meson8-pmx.c +p" > /sys/kernel/debug/dynamic_debug/control

# شوف الرسائل
dmesg -w | grep -i "meson\|pinctrl\|mux"
```

**رفع مستوى الـ loglevel مؤقتاً:**
```bash
echo 8 > /proc/sys/kernel/printk
# أو
dmesg -n 7
```

**لو عايز تشوف الـ probe sequence كاملة:**
```bash
# أعد تحميل الـ module (لو مش builtin)
modprobe -r pinctrl-meson8
dmesg -C
modprobe pinctrl-meson8
dmesg | grep -i "meson8\|pinctrl\|bank\|probe"
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| الـ Config | الوظيفة | متى تفعّله |
|-----------|---------|-----------|
| `CONFIG_DEBUG_PINCTRL` | يفعّل الـ verbose logging في pinctrl core | أول خطوة في الـ debug |
| `CONFIG_GPIO_SYSFS` | يكشف GPIOs في sysfs للـ manual testing | لازم للتحقق اليدوي |
| `CONFIG_DEBUG_GPIO` | validates GPIO operations وبيعمل WARN_ON | لكشف استخدام GPIO غلط |
| `CONFIG_REGMAP_DEBUGFS` | يكشف الـ regmap registers في debugfs | لمشاهدة قيم مسجلات الـ pinmux مباشرة |
| `CONFIG_DEBUG_FS` | prerequisite لكل الـ debugfs entries | |
| `CONFIG_PINCTRL_MESON` | الـ driver الأساسي نفسه | |
| `CONFIG_DYNAMIC_DEBUG` | يتيح `pr_debug` runtime control | |
| `CONFIG_KALLSYMS` | أسماء الـ functions في stack traces | |
| `CONFIG_STACKTRACE` | stack trace في WARN_ON | |

```bash
# تأكد إن الـ config شغال
grep -E "CONFIG_DEBUG_PINCTRL|CONFIG_DEBUG_GPIO|CONFIG_REGMAP_DEBUGFS" /proc/config.gz 2>/dev/null | zcat | head
# أو
zcat /proc/config.gz | grep "PINCTRL\|GPIO_SYSFS\|REGMAP_DEBUG"
```

---

#### 6. أدوات الـ subsystem

```bash
# pinctrl-utils من الـ kernel tools (لو متاحة في distro)
# بديلها القراءة المباشرة من debugfs

# أداة gpio للـ userspace
apt install gpiod   # أو pacman -S libgpiod
gpiodetect          # بيشوف كل الـ GPIO chips
gpioinfo gpiochip0  # بيعرض كل pins في الـ chip مع اتجاهها وقيمتها

# مثال واقعي:
# gpiochip0 - 133 lines:
#         line   0:      "GPIOX_0"       unused   input  active-high
#         line   8:      "GPIOX_8"    "sd_clk_a"  output active-high [used]

# قراءة/ضبط GPIO
gpioget gpiochip0 0        # اقرأ GPIOX_0
gpioset gpiochip0 21=1     # اضبط GPIOX_21 = 1
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `meson8-pinctrl: cannot get gpio regmap` | ما لقاش الـ reg-gpio في DT | تأكد من `reg-names = "gpio"` في DT node |
| `meson8-pinctrl: probe failed: -ENODEV` | الـ compatible string غلط أو ما فيش تطابق | تأكد من `compatible = "amlogic,meson8-cbus-pinctrl"` |
| `pin X is already requested` | conflict — اتنين devices طلبوا نفس الـ pin | شوف `pinmux-pins` وحدّد مين طلبه أول |
| `could not request pin X` | الـ pin busy أو ما عرفش يعمل mux | تأكد إن ما فيش overlap في DT pinctrl configs |
| `meson_get_bank: pin X not found` | رقم الـ pin خارج كل الـ banks | الـ pin_id أكبر من `meson8_cbus_banks` range |
| `Failed to enable pull for pin X` | مشكلة في regmap write | تأكد من الـ clock الخاص بالـ cbus enabled |
| `pinctrl_select_state() failed` | ما قدرش يطبق الـ pinmux state | الـ function أو group مش معرّف في الـ DT |
| `gpio-X: failed to set direction` | الـ direction register مش قابل للكتابة | تأكد من صلاحيات الـ clock أو الـ power domain |
| `no regmap for mux` | `reg_mux` == NULL | الـ DT ما عندوش `reg-names = "mux"` |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

في `pinctrl-meson.c` الـ `meson_get_bank()` هو نقطة الإخفاق الأولى — أي pin مش موجود في أي bank هيرجع `-EINVAL`. ضع `WARN_ON` هنا:

```c
static int meson_get_bank(struct meson_pinctrl *pc, unsigned int pin,
                          const struct meson_bank **bank)
{
    int i;
    for (i = 0; i < pc->data->num_banks; i++) {
        if (pin >= pc->data->banks[i].first &&
            pin <= pc->data->banks[i].last) {
            *bank = &pc->data->banks[i];
            return 0;
        }
    }
    /* Strategic debug point — pin not found in any bank */
    WARN(1, "meson-pinctrl: pin %u not found in any bank\n", pin);
    return -EINVAL;
}
```

**نقاط مهمة أخرى:**

```c
/* في meson8_pmx_ops set_mux — بعد regmap_update_bits */
ret = regmap_update_bits(pc->reg_mux, reg, BIT(bit), arg);
WARN_ON_ONCE(ret != 0); /* regmap write failed — clock off? */

/* في meson_pinconf_set — قبل ما تكتب pull register */
WARN_ON(!bank); /* bank should never be NULL here */

/* في probe — تأكد إن الـ banks غير فاضية */
WARN_ON(pc->data->num_banks == 0);
```

**dump_stack() مفيدة في:**
- `meson_pinctrl_probe()` لو رجعت `-EPROBE_DEFER` أكتر من مرة
- أي مسار بيرجع `-ENODEV` بعد ما الـ DT اتفك

---

### Hardware Level

#### 1. التحقق من state الـ Hardware مقابل الـ Kernel

**الـ Meson8 pinmux** يعمل عبر registers في نطاق الـ **CBUS** (عنوان أساسي `0xC1100000`) وعنوان **AOBUS** (عنوان أساسي `0xC8100000`).

```bash
# احسب عنوان الـ MUX register
# من مصدر pinctrl-meson8.c: GROUP(sd_cmd_a, 8, 0) → reg=8, bit=0
# reg offset = 8 * 4 = 0x20
# CBUS MUX base = 0xC1108000 (نمط شائع على Meson8)

# قارن بين ما يعتقده الـ kernel وما هو موجود فعلاً:
cat /sys/kernel/debug/regmap/*/registers | grep -A2 "20:"
```

**خطوات التحقق:**
1. اقرأ `pinmux-pins` من debugfs → هذا ما يعتقده الـ kernel.
2. اقرأ الـ register الفعلي من الـ hardware (انظر القسم التالي).
3. قارن: لو الـ kernel قال bit=1 بس الـ hardware يقول 0 → مشكلة في regmap أو الـ clock.

---

#### 2. Register Dump

**استخدام `devmem2`** — الأسهل على Embedded Linux:

```bash
# تثبيت devmem2
apt install devmem2
# أو ابنيه من source

# اقرأ MUX register #8 (sd_a group) على CBUS
# CBUS pinmux base على Meson8 = 0xC1108000
# offset لـ reg 8 = 8 * 4 = 0x20
devmem2 0xC1108020 w
# مثال output: Value at address 0xC1108020 (0xc1108020): 0x00000001
# bit 0 = 1 → sd_cmd_a enabled ✓

# اقرأ PULLEN register لـ bank X (reg 4, bit 0, offset = 4*4 = 0x10)
# CBUS gpio/pull base = 0xC1109880 (نمط Meson8)
devmem2 0xC1109880 w    # PULLEN register 0 للـ bank X
devmem2 0xC1109884 w    # PULL direction register 0
devmem2 0xC1109808 w    # DIR register للـ bank X

# AO bus registers
devmem2 0xC8100014 w    # AO MUX register 0
devmem2 0xC8100024 w    # AO PULLEN register
```

**استخدام `/dev/mem` مع Python للـ dump سريع:**

```bash
python3 -c "
import mmap, struct
fd = open('/dev/mem', 'rb')
# CBUS pinmux block — 10 registers
base = 0xC1108000
size = 0x100
mem = mmap.mmap(fd.fileno(), size, offset=base)
for i in range(10):
    val = struct.unpack('<I', mem[i*4:(i+1)*4])[0]
    print(f'MUX_REG[{i}] @ {base+i*4:#010x} = {val:#010x}')
fd.close()
"
```

**استخدام `io` tool من busybox:**
```bash
io -4 -r 0xC1108020   # اقرأ word من عنوان MUX_REG_8
```

---

#### 3. Logic Analyzer / Oscilloscope

| السيناريو | ماذا تراقب | ما تبحث عنه |
|-----------|-----------|------------|
| UART لا يشتغل | TX pin (مثلاً GPIOX_4) | يجب أن يتأرجح — لو ثابت → mux غلط |
| SD card لا يُقرأ | CLK pin (GPIOX_8) | clock يجب يكون 400kHz عند init |
| I2C hung | SDA + SCL معاً | SDA عالق low → slave لم يُـrelease الـ bus |
| SPI no data | MOSI + SCLK | MOSI تتغير عند كل حافة SCLK |
| Ethernet لا يـlink | MDC + MDIO | MDC يجب أن يـclock بـ ~2.5MHz |

**نصائح عملية:**
- شيّك الـ **voltage level** أول — Meson8 بعض الـ banks تعمل على 3.3V وبعضها 1.8V.
- لو الـ pin مش بيتحرك خالص → الـ mux مش اتضبط أو الـ GPIO mode.
- لو بيتحرك بس بشكل غريب → اشك في الـ pull resistor أو drive strength.
- اضبط الـ trigger على الـ CLK line واراقب الـ data lines.

---

#### 4. مشاكل Hardware شائعة وأنماطها في الـ Kernel Log

| المشكلة الـ Hardware | النمط في dmesg | التفسير |
|--------------------|----------------|---------|
| خطأ في توصيل SD card | `mmc0: error -110 whilst initialising SD card` | CLK أو CMD مش موصلين — راجع mux |
| pull-up ناقص على I2C | `i2c i2c-0: timeout waiting for bus ready` | SDA/SCL محتاجين pull-up خارجي |
| Ethernet PHY مش شايفها | `stmmac: MDIO bus not found` | GPIOZ_12/13 (mdio/mdc) مش مضبوطين |
| UART noise | `serial: too much work for irq` | مشكلة في الـ baud rate أو الـ signal integrity |
| HDMI HPD لا يُكتشف | `drm: connector HDMI-A-1: no hotplug` | GPIOH_0 (hdmi_hpd) الـ mux أو الـ pull غلط |
| SPI transfer fails | `spi_master spi0: transfer timeout` | CS أو CLK أو MOSI mux غلط |

---

#### 5. Device Tree Debugging

**تأكد من تطابق الـ DT مع الـ Hardware:**

```bash
# اقرأ الـ DT الـ compiled المحمّل فعلاً
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "pinctrl@"

# تأكد من وجود الـ compatible strings الصح
grep -r "amlogic,meson8" /proc/device-tree/ 2>/dev/null | strings

# شوف الـ pinctrl references من الـ devices
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -B2 -A5 "pinctrl-0"
```

**أخطاء DT شائعة:**

```dts
/* خطأ شائع — نسي reg-names */
pinctrl_cbus: pinctrl@c1108000 {
    compatible = "amlogic,meson8-cbus-pinctrl";
    reg = <0xc1108000 0x4000>;
    /* مفقود: reg-names = "mux"; */
    /* النتيجة: cannot get mux regmap → probe fail */
};

/* الصحيح */
pinctrl_cbus: pinctrl@c1108000 {
    compatible = "amlogic,meson8-cbus-pinctrl";
    reg = <0xc1108000 0x4000>,
          <0xc1109880 0x200>,
          <0xc1109800 0x40>;
    reg-names = "mux", "pull", "gpio";
    #gpio-cells = <2>;
    gpio-controller;
};
```

```bash
# لو بتشك في الـ DT structure، فك ضغطه وافحصه:
dtc -I dtb -O dts /boot/dtb/amlogic/meson8-something.dtb 2>/dev/null | \
    grep -A 30 "cbus-pinctrl\|aobus-pinctrl"

# تأكد إن الـ mmc node بيستخدم الـ pinctrl صح
dtc -I dtb -O dts /boot/dtb/amlogic/meson8-something.dtb 2>/dev/null | \
    grep -B2 -A 10 "meson8-sd-a\|pinctrl-0"
```

---

### Practical Commands

#### أوامر جاهزة للنسخ والتشغيل

**1. كشف حالة الـ pinmux بالكامل:**
```bash
#!/bin/bash
PCTRL_BASE=/sys/kernel/debug/pinctrl
for dev in "$PCTRL_BASE"/*/; do
    echo "=== $(basename $dev) ==="
    echo "--- pinmux-pins ---"
    cat "$dev/pinmux-pins" | grep -v "GPIO\|GPIOAO"  # أظهر الـ muxed فقط
    echo "--- pinconf-pins ---"
    cat "$dev/pinconf-pins" | head -30
done
```

**2. اقرأ MUX register معين:**
```bash
# reg number من GROUP macro في pinctrl-meson8.c
# مثال: GROUP(sd_cmd_a, 8, 0) → reg=8
REG_NUM=8
CBUS_MUX_BASE=0xC1108000
OFFSET=$(( REG_NUM * 4 ))
ADDR=$(printf "0x%X" $(( CBUS_MUX_BASE + OFFSET )) )
echo "Reading MUX register $REG_NUM at $ADDR"
devmem2 $ADDR w
```

**3. تتبع كل كتابة على مسجلات الـ pinctrl:**
```bash
#!/bin/bash
# Enable regmap tracing for meson pinctrl
TRACE=/sys/kernel/debug/tracing
echo 0 > $TRACE/tracing_on
echo "" > $TRACE/trace
echo 1 > $TRACE/events/regmap/regmap_reg_write/enable
echo 1 > $TRACE/tracing_on
echo "Tracing pinctrl regmap writes... press Ctrl+C to stop"
cat $TRACE/trace_pipe | grep -i "meson\|pinctrl"
```

**4. اختبار GPIO يدوياً:**
```bash
#!/bin/bash
# اختبر GPIOX_10 (xtal_32k_out or pwm_e)
# أول: شوف الـ GPIO base number
CHIP=gpiochip0
BASE=$(cat /sys/class/gpio/$CHIP/base)
PIN_OFFSET=10   # GPIOX_10
GPIO_NUM=$((BASE + PIN_OFFSET))

echo "Testing GPIO $GPIO_NUM (GPIOX_$PIN_OFFSET)"
echo $GPIO_NUM > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio$GPIO_NUM/direction
echo 1 > /sys/class/gpio/gpio$GPIO_NUM/value
sleep 1
echo 0 > /sys/class/gpio/gpio$GPIO_NUM/value
echo $GPIO_NUM > /sys/class/gpio/unexport
echo "Done — check pin with multimeter or oscilloscope"
```

**5. مقارنة الـ expected مع الـ actual mux state:**
```bash
#!/bin/bash
# Expected: uart_tx_ao_a on GPIOAO_0 → AO MUX register 0, bit 12
AO_MUX_BASE=0xC8100014   # AO MUX reg 0 address (Meson8 typical)
echo "Expected: uart_tx_ao_a → AO_MUX[0] bit 12 should be 1"
VAL=$(devmem2 $AO_MUX_BASE w | grep "Value" | awk '{print $NF}')
echo "Actual AO_MUX[0] = $VAL"
BIT12=$(( (16#${VAL##0x} >> 12) & 1 ))
echo "bit 12 = $BIT12 (1=uart enabled, 0=GPIO mode)"
```

**6. فحص driver probe:**
```bash
# هل الـ driver probe نجح؟
ls /sys/bus/platform/drivers/meson8-pinctrl/
# يجب أن تجد symlinks للـ devices

# تفاصيل الـ probe:
dmesg | grep -i "meson8-pinctrl\|pinctrl.*meson\|amlogic.*pinctrl"

# هل الـ GPIO chip اتسجل؟
cat /sys/class/gpio/gpiochip*/label | grep -i meson
```

**7. Dynamic debug شامل لحظة الـ boot:**
```bash
# في kernel command line (في /boot/extlinux/extlinux.conf أو UBOOT env)
# أضف:
# dyndbg="file pinctrl-meson.c +p; file pinctrl-meson8.c +p"
# بعدين:
dmesg | grep -i "meson.*pinctrl\|pinctrl.*meson" | head -50
```

**8. تأكيد تطابق DT مع الـ board:**
```bash
# شوف الـ compatible strings للـ pinctrl nodes الموجودة
find /proc/device-tree -name "compatible" -exec sh -c \
  'val=$(strings "$1"); echo "$1: $val"' _ {} \; 2>/dev/null | \
  grep -i "amlogic\|meson8"

# تأكد من وجود الـ GPIO cells
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
  grep -A5 "#gpio-cells"
```

**9. فحص IRQ mapping (bank X: irq 112–133، bank Y: 95–111، إلخ):**
```bash
# من تعريف BANK("X", GPIOX_0, GPIOX_21, 112, 133, ...)
cat /proc/interrupts | grep -E "11[2-9]|12[0-9]|13[0-3]"
# لو IRQ معمول له request → GPIO interrupts شغالة
```

**10. فحص pull-up/pull-down registers:**
```bash
# PULLEN register للـ bank X: reg offset 4 في regmap
# من BANK("X", ..., 4, 0, 4, 0, ...) → pullen reg=4, bit=0
# في regmap debugfs:
cat /sys/kernel/debug/regmap/*/registers | awk '
  /^[0-9a-f]+:/ {
    addr = strtonum("0x" $1)
    # PULLEN registers are at offsets 0x10-0x40 typically
    if (addr >= 0x10 && addr <= 0x50)
      printf "PULLEN/PULL offset 0x%02x: %s\n", addr, $2
  }
'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Android TV Box بـ Amlogic S812 — HDMI-CEC مش شغال

#### العنوان
**الـ HDMI-CEC مش بيشتغل على TV Box بـ Meson8m2**

#### السياق
شركة بتعمل Android TV box بـ **Amlogic S812** (وهو Meson8m2). الجهاز بيشتغل تمام، بس الـ **HDMI-CEC** مش بيتجاوب — يعني المستخدم مش قادر يتحكم في التليفزيون من الريموت عبر CEC، ولا السيستم بيصحى من الـ standby لما التلفاز بيتشغل.

#### المشكلة
الـ engineer اكتشف إن مفيش إشارة على خط CEC. الـ oscilloscope مبيشوفش أي نشاط على الـ pin.

#### التحليل
بنبص على الـ DT وبنشوف:

```c
/* bank H في الملف */
static const unsigned int hdmi_cec_pins[]   = { GPIOH_3 };
static const unsigned int spi_ss0_0_pins[]  = { GPIOH_3 };

GROUP(hdmi_cec,  1, 23),   /* register 1, bit 23 */
GROUP(spi_ss0_0, 9, 13),   /* register 9, bit 13 */
```

**الـ GPIOH_3 shared بين اتنين functions**: `hdmi_cec` و`spi_ss0_0`.

بعدين بنبص على الـ DT:

```dts
/* الـ DT كان فيه SPI node بيستخدم GPIOH_3 */
&spi {
    pinctrl-0 = <&spi_pins>;   /* spi_pins بتحدد GPIOH_3 */
    status = "okay";
};
```

الـ pinctrl driver لما `meson8_pmx_ops` set the mux لـ SPI، بيكتب في register 9 bit 13، وده بيغير الـ mux لـ GPIOH_3 من `hdmi_cec` لـ SPI. الـ HDMI CEC node موجود في DT بس مفيش SPI device على هذا الـ pin في الـ board الحقيقية!

#### الحل

**1. في الـ DT، نشيل الـ SPI أو نغير الـ pins بتاعته:**

```dts
/* نعطل الـ SPI لو مش محتاجينه */
&spi {
    status = "disabled";
};

/* أو نستخدم SPI على bank Z بدل H */
spi_z_pins: spi-z-pins {
    mux {
        groups = "spi_ss0_1", "spi_sclk_1",
                 "spi_mosi_1", "spi_miso_1";
        function = "spi";
    };
};
```

**2. نتأكد إن hdmi_cec_ao أو hdmi_cec (bank H) هو اللي active:**

```dts
hdmi_cec_pins: hdmi-cec-pins {
    mux {
        groups = "hdmi_cec";  /* GPIOH_3 */
        function = "hdmi";
    };
};
```

**3. Debug commands:**

```bash
# شوف الـ pinmux state الحالي
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep GPIOH_3

# شوف الـ functions المتاحة
cat /sys/kernel/debug/pinctrl/*/pinmux-functions | grep hdmi

# اتأكد إن الـ mux register صح
devmem2 0xC1109880 w   # CBUS MUX reg 1 — bit 23 لازم يكون set
```

#### الدرس المستفاد
الـ **GPIOH_3 shared** بين `hdmi_cec` و`spi_ss0_0`. لازم دايمًا تتأكد من الـ pin conflict في `meson8_cbus_groups[]` قبل تفعيل أي function. الـ kernel مش بيدي warning تلقائي لو اتنين functions بيتنافسوا على نفس الـ bit في نفس الـ register.

---

### السيناريو الثاني: Industrial Gateway بـ Meson8 — UART Debug Console مش شغالة

#### العنوان
**الـ Serial Console مش بتطلع على Industrial Gateway**

#### السياق
شركة بتبني **industrial gateway** بـ **Amlogic Meson8** لجمع بيانات من sensors. الـ board وصلت من المصنع ومش فيه إشارة على الـ UART debug console. الـ bootloader بيظهر على screen خارجي بس الـ serial port ساكت.

#### المشكلة
الـ engineer وصّل logic analyzer على الـ UART TX pin بس مفيش إشارة. الـ board notes بتقول الـ console على UART-AO (Always-On domain).

#### التحليل
بنبص على الـ aobus في الملف:

```c
/* bank AO */
static const unsigned int uart_tx_ao_a_pins[] = { GPIOAO_0 };
static const unsigned int uart_rx_ao_a_pins[] = { GPIOAO_1 };

GROUP(uart_tx_ao_a,  0, 12),  /* AO register 0, bit 12 */
GROUP(uart_rx_ao_a,  0, 11),  /* AO register 0, bit 11 */

static const struct meson_bank meson8_aobus_banks[] = {
    BANK("AO", GPIOAO_0, GPIO_TEST_N, 0, 13,
         0, 16,   /* pullen: reg 0, bit 16 */
         0,  0,   /* pull:   reg 0, bit 0  */
         0,  0,   /* dir:    reg 0, bit 0  */
         0, 16,   /* out:    reg 0, bit 16 */
         1,  0),  /* in:     reg 1, bit 0  */
};
```

الـ AO domain عنده `parse_dt` خاص:

```c
static const struct meson_pinctrl_data meson8_aobus_pinctrl_data = {
    ...
    .parse_dt = &meson8_aobus_parse_dt_extra,  /* AO-specific init */
};
```

الـ `meson8_aobus_parse_dt_extra` بتقرأ الـ AO registers من الـ DT. لو الـ DT مش مُعرّف صح (compatible string غلط)، الـ AO pinctrl مش هيت probe ومفيش mux هيتعمل.

الـ DT الغلط:
```dts
/* غلط: compatible string مش متطابق */
pinctrl_aobus: pinctrl@c8100000 {
    compatible = "amlogic,meson8-cbus-pinctrl";  /* خطأ! */
};
```

المفروض يكون:
```dts
compatible = "amlogic,meson8-aobus-pinctrl";
```

#### الحل

**صحح الـ DT:**

```dts
pinctrl_aobus: pinctrl@c8100000 {
    compatible = "amlogic,meson8-aobus-pinctrl";
    #address-cells = <1>;
    #size-cells = <1>;

    uart_ao_a_pins: uart-ao-a-pins {
        mux {
            groups = "uart_tx_ao_a", "uart_rx_ao_a";
            function = "uart_ao";
        };
    };
};

&uart_AO {
    pinctrl-names = "default";
    pinctrl-0 = <&uart_ao_a_pins>;
    status = "okay";
};
```

**تأكد إن الـ probe نجح:**

```bash
dmesg | grep meson8-pinctrl
# المتوقع:
# meson8-pinctrl c8100000.pinctrl: probed

# شوف الـ pinctrl devices
ls /sys/bus/platform/drivers/meson8-pinctrl/
```

#### الدرس المستفاد
الـ **AO bus** عنده `compatible` string منفصل (`amlogic,meson8-aobus-pinctrl`) وله `parse_dt` مختلف عن الـ cbus. لو الـ `of_device_id` match فشل، الـ `meson8_aobus_pinctrl_data` مش هيتستخدم خالص والـ pins هتفضل في الـ default state.

---

### السيناريو الثالث: Meson8m2 Set-Top Box — Ethernet مش شغال بشكل صح (Gigabit مش متاح)

#### العنوان
**الـ Ethernet على S812 بيشتغل بـ 100Mbps بس مع إن الـ PHY يدعم Gigabit**

#### السياق
Set-top box بـ **Amlogic S812 (Meson8m2)**. الـ board عندها Gigabit Ethernet PHY. بس الشبكة دايمًا بتتفاوض على 100Mbps. الـ engineer بيشك في الـ pinmux.

#### المشكلة
الـ Gigabit Ethernet محتاج 8 data pins (RGMII: TXD0–TXD3 و RXD0–RXD3). الـ 100Mbps بيستخدم 4 بس (MII: TXD0-1, RXD0-1).

#### التحليل
بنبص على الملف:

```c
/* bank Z — Ethernet pins */
static const unsigned int eth_txd3_pins[]  = { GPIOZ_0 };
static const unsigned int eth_txd2_pins[]  = { GPIOZ_1 };
static const unsigned int eth_rxd3_pins[]  = { GPIOZ_2 };
static const unsigned int eth_rxd2_pins[]  = { GPIOZ_3 };
static const unsigned int eth_tx_clk_50m_pins[] = { GPIOZ_4 };
static const unsigned int eth_tx_en_pins[] = { GPIOZ_5 };
static const unsigned int eth_txd1_pins[]  = { GPIOZ_6 };
static const unsigned int eth_txd0_pins[]  = { GPIOZ_7 };
static const unsigned int eth_rx_clk_in_pins[] = { GPIOZ_8 };
static const unsigned int eth_rx_dv_pins[] = { GPIOZ_9 };
static const unsigned int eth_rxd1_pins[]  = { GPIOZ_10 };
static const unsigned int eth_rxd0_pins[]  = { GPIOZ_11 };
static const unsigned int eth_mdio_pins[]  = { GPIOZ_12 };
static const unsigned int eth_mdc_pins[]   = { GPIOZ_13 };

/* NOTE: the following four groups are only available on Meson8m2: */
GROUP(eth_rxd2, 6, 3),
GROUP(eth_rxd3, 6, 2),
GROUP(eth_txd2, 6, 1),
GROUP(eth_txd3, 6, 0),
```

التعليق في الكود واضح: `eth_txd2`, `eth_txd3`, `eth_rxd2`, `eth_rxd3` **خاصة بـ Meson8m2 بس**. يعني دي الـ pins اللي بتفرق بين 100Mbps و Gigabit.

لو الـ DT بيستخدم compatible الأصلي `meson8` وبيشيل الـ 4 groups دول من الـ `ethernet_groups`:

```dts
/* DT الغلط: بيستخدم meson8 compatible على Meson8m2 board */
ethernet_pins: ethernet-pins {
    mux {
        groups = "eth_tx_clk_50m", "eth_tx_en",
                 "eth_txd0", "eth_txd1",   /* فقط — مش فيه txd2/txd3 */
                 "eth_rx_clk_in", "eth_rx_dv",
                 "eth_rxd0", "eth_rxd1",   /* فقط — مش فيه rxd2/rxd3 */
                 "eth_mdio", "eth_mdc";
        function = "ethernet";
    };
};
```

بما إن `eth_txd2/3` و`eth_rxd2/3` مش configured، الـ PHY شايف بس 4 data lines فبيفاوض على 100Mbps.

#### الحل

```dts
/* للـ Meson8m2 — RGMII كامل */
ethernet_pins: ethernet-pins {
    mux {
        groups = "eth_tx_clk_50m", "eth_tx_en",
                 "eth_txd0", "eth_txd1", "eth_txd2", "eth_txd3",
                 "eth_rx_clk_in", "eth_rx_dv",
                 "eth_rxd0", "eth_rxd1", "eth_rxd2", "eth_rxd3",
                 "eth_mdio", "eth_mdc";
        function = "ethernet";
    };
};
```

**وتأكد إن الـ compatible في الـ pinctrl node صح:**

```dts
pinctrl_cbus: pinctrl@c1109880 {
    compatible = "amlogic,meson8m2-cbus-pinctrl";  /* مش meson8 */
    ...
};
```

```bash
# تأكد من الـ groups المفعلة
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep GPIOZ_[0-3]$

# تأكد من speed التفاوض
ethtool eth0 | grep Speed
```

#### الدرس المستفاد
الـ comment في الكود (`/* NOTE: only available on Meson8m2 */`) مش مجرد تعليق — ده تحذير عملي. الـ `meson8_cbus_pinctrl_data` واحدة لـ Meson8 و Meson8m2 (**نفس struct**)، بس في الـ DT لازم تضيف الـ groups الإضافية للـ Gigabit Ethernet لو الـ board S812.

---

### السيناريو الرابع: IoT Sensor Board — I2C Conflict مع SPI

#### العنوان
**الـ I2C بيتعطل بشكل متقطع على Meson8 IoT Board**

#### السياق
شركة بتصمم **IoT sensor hub** بـ **Amlogic Meson8**. الـ board عندها:
- I2C على GPIOZ_0/GPIOZ_1 لـ sensors
- SPI على GPIOH_4/5/6 لـ flash memory

بعد تشغيل المنتج بأسبوعين، الـ I2C بيوقف بشكل عشوائي.

#### المشكلة
الـ I2C transactions بتفشل أحيانًا بـ `EREMOTEIO` أو `ENXIO`.

#### التحليل
بنبص على الملف — GPIOZ_0 وGPIOZ_1:

```c
/* bank Z — multiple I2C options on same pins! */
static const unsigned int i2c_sda_a0_pins[] = { GPIOZ_0 };
static const unsigned int i2c_sck_a0_pins[] = { GPIOZ_1 };

static const unsigned int i2c_sda_a1_pins[] = { GPIOZ_0 };  /* نفس الـ pin */
static const unsigned int i2c_sck_a1_pins[] = { GPIOZ_1 };  /* نفس الـ pin */

static const unsigned int i2c_sda_a2_pins[] = { GPIOZ_0 };  /* نفس الـ pin! */
static const unsigned int i2c_sck_a2_pins[] = { GPIOZ_1 };  /* نفس الـ pin! */

GROUP(i2c_sda_a0, 5, 31),
GROUP(i2c_sck_a0, 5, 30),

GROUP(i2c_sda_a1, 5, 9),
GROUP(i2c_sck_a1, 5, 8),

GROUP(i2c_sda_a2, 5, 7),
GROUP(i2c_sck_a2, 5, 6),
```

الـ GPIOZ_0 و GPIOZ_1 بيتحكم فيهم **3 groups مختلفة** (a0, a1, a2) في **registers مختلفة** (bits 31/30, 9/8, 7/6 في register 5). لو الـ DT بيعرّف أكتر من واحدة:

```dts
/* الغلط: تعريف i2c_a0 وi2c_a1 معًا */
i2c_a_pins: i2c-a-pins {
    mux {
        groups = "i2c_sda_a0", "i2c_sck_a0",
                 "i2c_sda_a1", "i2c_sck_a1";  /* conflict! */
        function = "i2c_a";
    };
};
```

لما الـ driver بيعمل `pinmux_select` لـ `i2c_a1`، بيكتب في reg 5 bits 9/8، بس لو في code path تاني بيحاول `i2c_a0` (reg 5 bits 31/30)، ممكن الـ mux يتغير unexpectedly. السبب الحقيقي: `i2c_a_groups[]` بيحتوي على الـ 3 variants معًا:

```c
static const char * const i2c_a_groups[] = {
    "i2c_sda_a0", "i2c_sck_a0",  /* GPIOZ_0/1, reg5 bit 31/30 */
    "i2c_sda_a1", "i2c_sck_a1",  /* GPIOZ_0/1, reg5 bit 9/8  */
    "i2c_sda_a2", "i2c_sck_a2",  /* GPIOZ_0/1, reg5 bit 7/6  */
};
```

#### الحل

استخدم variant واحدة بس وبشكل explicit في الـ DT:

```dts
i2c_sensors_pins: i2c-sensors-pins {
    mux {
        groups = "i2c_sda_a0", "i2c_sck_a0";  /* فقط a0 */
        function = "i2c_a";
    };
};

&i2c_A {
    pinctrl-names = "default";
    pinctrl-0 = <&i2c_sensors_pins>;
    clock-frequency = <100000>;
    status = "okay";
};
```

**Debug:**

```bash
# شوف إيه اللي بيتحكم في GPIOZ_0
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep GPIOZ_0

# تابع الـ regmap مباشرة (register 5 في CBUS)
devmem2 0xC1109880 w   # اقرأ MUX register 5
# bit 31/30 = i2c_a0, bit 9/8 = i2c_a1, bit 7/6 = i2c_a2
```

#### الدرس المستفاد
وجود **multiple groups للـ same pins** (a0/a1/a2 كلها على GPIOZ_0/1) ده تصميم flexibility — بس ممكن يبقى مصدر conflict لو الـ DT عرّف أكتر من group. اختار group واحدة بس لكل pin وحط comment ليه.

---

### السيناريو الخامس: Custom Board Bring-Up — NAND Flash مش بيتعرف عليه

#### العنوان
**الـ NAND Flash مش بيتعرف عليه في Early Boot على Meson8 Custom Board**

#### السياق
فريق bring-up بيشغّل لأول مرة custom board بـ **Amlogic Meson8** مصممة لـ industrial data logger. الـ storage هو **NAND Flash** موصول على الـ BOOT bus. الـ u-boot مش بيعرف الـ NAND.

#### المشكلة
```
NAND: 0 MiB
```
يعني الـ NAND controller مش بيشوف أي device.

#### التحليل
بنبص على الملف:

```c
/* bank BOOT */
static const unsigned int nand_io_pins[] = {
    BOOT_0, BOOT_1, BOOT_2, BOOT_3,
    BOOT_4, BOOT_5, BOOT_6, BOOT_7  /* 8-bit data bus */
};
static const unsigned int nand_io_ce0_pins[]  = { BOOT_8 };
static const unsigned int nand_io_ce1_pins[]  = { BOOT_9 };
static const unsigned int nand_io_rb0_pins[]  = { BOOT_10 };
static const unsigned int nand_ale_pins[]     = { BOOT_11 };
static const unsigned int nand_cle_pins[]     = { BOOT_12 };
static const unsigned int nand_wen_clk_pins[] = { BOOT_13 };
static const unsigned int nand_ren_clk_pins[] = { BOOT_14 };
static const unsigned int nand_dqs_pins[]     = { BOOT_15 };
static const unsigned int nand_ce2_pins[]     = { BOOT_16 };
static const unsigned int nand_ce3_pins[]     = { BOOT_17 };

GROUP(nand_io,      2, 26),
GROUP(nand_io_ce0,  2, 25),
GROUP(nand_io_ce1,  2, 24),
GROUP(nand_io_rb0,  2, 17),
GROUP(nand_ale,     2, 21),
GROUP(nand_cle,     2, 20),
GROUP(nand_wen_clk, 2, 19),
GROUP(nand_ren_clk, 2, 18),
GROUP(nand_dqs,     2, 27),
```

ولاحظ:
```c
/* bank BOOT — SD على نفس الـ pins! */
static const unsigned int sd_d0_c_pins[] = { BOOT_0 };
static const unsigned int sd_d1_c_pins[] = { BOOT_1 };
...

GROUP(sd_d0_c, 6, 29),
GROUP(sd_d1_c, 6, 28),
...

/* وكمان NOR flash على BOOT_11-18 */
static const unsigned int nor_d_pins[]  = { BOOT_11 };
static const unsigned int nor_q_pins[]  = { BOOT_12 };
static const unsigned int nor_c_pins[]  = { BOOT_13 };
static const unsigned int nor_cs_pins[] = { BOOT_18 };
```

الـ BOOT bank بيشارك pins بين NAND، SD_C، وNOR flash. الـ bank عندها:

```c
BANK("BOOT", BOOT_0, BOOT_18, 39, 57,
     2,  0,   /* pullen */
     2,  0,   /* pull   */
     9,  0,   /* dir    */
    10,  0,   /* out    */
    11,  0),  /* in     */
```

الـ engineer لقى إن الـ DT عنده SD node مفعّل على BOOT pins:

```dts
/* الغلط: SD_C مفعّل على BOOT pins مع وجود NAND */
&sd_emmc_c {
    pinctrl-0 = <&sdxc_c_pins>;  /* BOOT_0..BOOT_3, BOOT_16, BOOT_17 */
    status = "okay";
};
```

الـ SD pinmux (register 6, bits 29-24) بيخلي BOOT_0-3 و BOOT_16-17 تشتغل كـ SD lines. ده بيمنع الـ NAND controller من الوصول لخطوط البيانات.

**في نفس الوقت، الـ nor_cs على BOOT_18** بييجي conflict مع الـ NAND CE2/CE3 اللي على BOOT_16/BOOT_17.

#### الحل

**1. عطّل SD_C وNOR في الـ DT:**

```dts
/* عطّل أي interface تاني على BOOT bank */
&sd_emmc_c {
    status = "disabled";
};

/* لو NOR مش مستخدم */
/* تأكد مفيش nor pinctrl active */
```

**2. فعّل NAND بشكل صح:**

```dts
nand_pins: nand-pins {
    mux {
        groups = "nand_io",
                 "nand_io_ce0",
                 "nand_io_rb0",
                 "nand_ale",
                 "nand_cle",
                 "nand_wen_clk",
                 "nand_ren_clk",
                 "nand_dqs";
        function = "nand";
    };
};

&nand {
    pinctrl-names = "default";
    pinctrl-0 = <&nand_pins>;
    status = "okay";
};
```

**3. Debug steps:**

```bash
# شوف الـ BOOT pins state
cat /sys/kernel/debug/pinctrl/*/pins | grep BOOT

# اقرأ MUX register 2 (NAND mux)
devmem2 0xC1109880 w   # اقرأ reg 2
# bit 26 لازم يكون 1 = nand_io enabled

# اقرأ MUX register 6 (SD_C mux)
# bits 29-24 لازم يكونوا 0 = SD_C disabled

# شوف الـ pinmux conflicts
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep BOOT
```

**4. تأكد إن الـ NAND controller شايل الـ pins بشكل صح:**

```bash
dmesg | grep -i nand
# المتوقع: "NAND: device found, Manufacturer ID: 0xXX"
```

#### الدرس المستفاد
الـ **BOOT bank** هو الأكثر تعقيدًا في Meson8 — بيحتضن NAND، SD_C، وNOR flash على **نفس الـ pins** بـ mux registers مختلفة. أثناء bring-up، لازم تعطّل كل interface غير الـ primary storage في الـ DT قبل ما تبدأ تـ debug الـ storage controller. جدول الـ groups ووجود `nand_ce2/ce3` على `BOOT_16/17` اللي هي نفس pins الـ `sd_cmd_c/sd_clk_c` يخلّي الـ conflict غير واضح لو مقراتش الملف كويس.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ** LWN.net هو المرجع الأساسي لمتابعة تطور الـ kernel — الروابط دي بتغطي كل مراحل تطور الـ pinctrl subsystem والـ Meson driver تحديداً.

| المقال | الوصف |
|--------|-------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | مقال أساسي بيشرح تصميم الـ pinctrl subsystem من الأول |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | نقاش الـ mailing list لأول patch series للـ pinctrl |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/) | إضافة الـ pin configuration interface |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | تطوير الـ pinconf API |
| [Pinctrl driver for Amlogic Meson SoCs](https://lwn.net/Articles/616225/) | أول driver للـ Meson pinctrl — البداية الفعلية للـ codebase |
| [Amlogic Meson pinctrl driver](https://lwn.net/Articles/620822/) | iteration تانية للـ Meson pinctrl driver |
| [Amlogic Meson8b pinctrl driver support](https://lwn.net/Articles/638594/) | إضافة دعم Meson8b للـ driver |
| [pinctrl: meson-a1: add pinctrl driver](https://lwn.net/Articles/804174/) | تطوير الـ driver للـ A1 SoC الحديث |
| [pinctrl: meson-s4: add pinctrl driver](https://lwn.net/Articles/879912/) | دعم الـ S4 SoC |
| [irqchip: meson: add support for the gpio interrupt controller](https://lwn.net/Articles/703946/) | إضافة دعم GPIO IRQ للـ Meson |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة في الـ kernel source tree وهي المرجع الرسمي للـ subsystem:

```
Documentation/driver-api/pinctl.rst          ← الوثيقة الرئيسية للـ pinctrl subsystem
Documentation/devicetree/bindings/pinctrl/   ← bindings الـ DT لكل pinctrl driver
drivers/pinctrl/meson/                       ← الـ source code للـ Meson pinctrl
include/linux/pinctrl/                       ← الـ headers الرسمية للـ subsystem
```

الـ online version:
- [PINCTRL subsystem — kernel.org docs](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html)
- [pinctrl.txt — kernel.org](https://www.kernel.org/doc/Documentation/pinctrl.txt)

الـ DT binding الخاص بالـ Meson8:
```
Documentation/devicetree/bindings/pinctrl/amlogic,meson8-pinctrl.yaml
```

---

### Commits مهمة في تاريخ الـ Driver

**الـ** commits دي بتمثل نقاط التحول الرئيسية في تطور الـ `pinctrl-meson8.c`:

| الـ commit | الوصف |
|------------|-------|
| الـ initial commit (Beniamino Galvani, 2014) | أول نسخة من الـ Meson8 pinctrl driver — أنشأ الـ `pinctrl-meson.c` والـ `pinctrl-meson8.c` |
| إضافة `meson8_aobus_parse_dt_extra` | فصل الـ AO bus parsing كـ callback منفصلة |
| إضافة دعم Meson8m2 | إضافة الـ `amlogic,meson8m2-*-pinctrl` compatible strings |
| إضافة GPIO IRQ support | ربط الـ GPIO interrupt controller بالـ pinctrl |
| تحويل الـ bindings لـ YAML | migration الـ DT bindings من `.txt` لـ `.yaml` schema |

للبحث في الـ git log:
```bash
git log --oneline -- drivers/pinctrl/meson/pinctrl-meson8.c
git log --oneline -- drivers/pinctrl/meson/pinctrl-meson.c
```

---

### نقاشات الـ Mailing List

- [RFC: irqchip meson gpio interrupt controller](https://linux-arm-kernel.infradead.narkive.com/k2chTD8j/rfc-00-10-irqchip-meson-add-support-for-the-gpio-interrupt-controller) — نقاش تفصيلي عن إضافة دعم الـ GPIO IRQ
- [ARM mailing list: pinctrl meson GPIO IRQs](https://www.spinics.net/lists/arm-kernel/msg423078.html) — نقاش حول تفعيل GPIO IRQs في الـ Meson pinctrl
- [PATCH: ARM meson8 machine definition](https://lists.gt.net/linux/kernel/2019456) — من أوائل الـ patches التي عرّفت الـ Meson8 machine

للبحث في الـ patchwork:
- [patchwork.kernel.org — linux-amlogic](https://patchwork.kernel.org/project/linux-amlogic/list/)

---

### موارد BayLibre والـ linux-meson.com

BayLibre هي الشركة اللي ساهمت بجزء كبير من الـ upstream work للـ Amlogic Meson:

- [How We Improved AmLogic Support in Mainline Linux](https://baylibre.com/improved-amlogic-support-mainline-linux/) — شرح BayLibre لمسيرة الـ mainlining
- [linux-meson.com — Mainlining Progress](https://linux-meson.com/mainlining.html) — متابعة شاملة لحالة الـ upstream support
- [linux-meson.com — Hardware Support](https://linux-meson.com/) — جدول الـ SoCs المدعومة وحالة كل driver

---

### kernelnewbies.org

**الـ** kernelnewbies بيوفر changelogs لكل kernel version — مفيد لمعرفة إيمتى اتضاف أو اتغير الـ driver:

| الصفحة | ما يخص الـ pinctrl |
|--------|-------------------|
| [Linux_6.15 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.15) | تحديثات الـ Amlogic pinctrl |
| [Linux_6.11 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.11) | تحديثات pinctrl عامة |
| [LinuxChanges — kernelnewbies.org](https://kernelnewbies.org/LinuxChanges) | أرشيف شامل للتغييرات |

---

### eLinux.org

- [Introduction to pin muxing and GPIO control under Linux (PDF)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) — presentation من ELC 2021 بيشرح الـ pinctrl و GPIO بشكل عملي
- [EBC Device Trees — elinux.org](https://elinux.org/EBC_Device_Trees) — أمثلة عملية لـ DT pinctrl configuration
- [i2c-demux-pinctrl — elinux.org](https://www.elinux.org/Tests:i2c-demux-pinctrl) — اختبار ربط الـ i2c مع الـ pinctrl

---

### الكتب الموصى بها

#### Linux Device Drivers (LDD3)
**الـ** LDD3 مجاني أونلاين وبيغطي الأساسيات:
- **Chapter 9** — *Communicating with Hardware*: فهم الـ memory-mapped I/O والـ register access
- **Chapter 14** — *The Linux Device Model*: فهم الـ `platform_driver` وكيف الـ pinctrl driver بيتسجل

> Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman — [تنزيل مجاني](https://lwn.net/Kernel/LDD3/)

---

#### Linux Kernel Development — Robert Love (الطبعة الثالثة)
- **Chapter 17** — *Devices and Modules*: فهم الـ device model وكيف الـ `platform_driver` بيشتغل
- **Chapter 5** — *System Calls*: فهم الـ kernel/userspace boundary اللي بيستخدمه الـ GPIO subsystem
- **Chapter 11** — *Timers and Time Management*: مفيد لفهم timing في الـ peripheral initialization

---

#### Embedded Linux Primer — Christopher Hallinan (الطبعة الثانية)
- **Chapter 14** — *Kernel Initialization*: فهم إزاي الـ pinctrl driver بيتبدأ في الـ boot
- **Chapter 15** — *Kernel Debugging Techniques*: debugging الـ GPIO والـ pinctrl

---

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds
- الطبعة الثالثة (2021) بتغطي الـ Device Tree والـ pinctrl بشكل حديث
- فصل خاص بالـ GPIO وكيفية التعامل مع الـ pinctrl من الـ DTS

---

### مصادر إضافية للبحث

#### Patchwork — تتبع الـ patches
```
https://patchwork.kernel.org/project/linux-amlogic/list/
https://patchwork.kernel.org/project/linux-arm-kernel/list/?q=meson+pinctrl
```

#### cateee.net — معلومات الـ Kconfig
- [CONFIG_PINCTRL_MESON8](https://cateee.net/lkddb/web-lkddb/PINCTRL_MESON8.html) — معلومات عن الـ config option وتاريخه في الـ kernels

---

### كلمات البحث الموصى بها

للبحث عن معلومات إضافية، استخدم الـ search terms دي:

```
# للبحث في الـ kernel source
pinctrl meson8 site:elixir.bootlin.com
amlogic meson8 gpio pinctrl driver
meson_pinctrl_probe implementation
meson_bank GPIO register layout

# للبحث في الـ mailing lists
linux-amlogic mailing list pinctrl
pinctrl meson GPIO interrupt hierarchy domain
GPIOAO aobus amlogic always-on domain

# للبحث في الـ git history
git log --all --oneline --grep="pinctrl.*meson8"
git log --all --oneline --grep="amlogic.*pinctrl"

# لفهم الـ DT bindings
amlogic,meson8-cbus-pinctrl devicetree binding
amlogic,meson8-aobus-pinctrl compatible
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `meson_pinctrl_probe` هي الدالة المسؤولة عن تسجيل أي pinctrl driver لـ Meson8 في الـ platform bus. بدلاً من تعديل الـ driver نفسه، هنحط **kprobe** على `meson_pinctrl_probe` عشان نعرف إمتى بيتسجّل pinctrl controller جديد، وإيه اسمه، وكام pin فيه — من غير ما نلمس الـ driver الأصلي.

ده مفيد جداً في debugging لأن pinctrl probe بيحصل مرة واحدة وقت boot، فالـ kprobe بيديك snapshot دقيق من بيانات الـ `meson_pinctrl_data` اللي بتتحط كـ platform data.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe module: hook meson_pinctrl_probe to log pinctrl registration info
 * Works on any kernel with Amlogic Meson8 / Meson8m2 pinctrl driver built-in.
 */

#include <linux/module.h>       /* MODULE_* macros, module_init/exit        */
#include <linux/kernel.h>       /* pr_info                                   */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, etc.      */
#include <linux/platform_device.h> /* struct platform_device               */

/* We need the meson pinctrl data struct layout to read fields */
#include <linux/pinctrl/pinctrl.h> /* struct pinctrl_pin_desc               */

/*
 * Forward-declare the meson_pinctrl_data struct as we know it from the driver
 * headers (pinctrl-meson.h). We only need the fields we'll print, so a
 * partial shadow struct is enough — we never write through it.
 */
struct meson_pinctrl_data_shadow {
    const char        *name;
    const void        *pins;       /* pinctrl_pin_desc array (opaque here)  */
    const void        *groups;
    const void        *funcs;
    unsigned int       num_pins;
    unsigned int       num_groups;
    unsigned int       num_funcs;
    const void        *banks;
    unsigned int       num_banks;
};

/* ------------------------------------------------------------------ */
/*  kprobe pre-handler: called just before meson_pinctrl_probe runs    */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * First argument to meson_pinctrl_probe is `struct platform_device *pdev`.
     * On arm/arm64 the first argument is in register x0 / r0.
     * regs_get_kernel_argument() is the portable helper (since 5.8).
     */
    struct platform_device *pdev =
        (struct platform_device *)regs_get_kernel_argument(regs, 0);

    if (!pdev)
        return 0;

    /*
     * The per-SoC pinctrl_data is stored in pdev->dev.platform_data when
     * the platform driver matches via of_device_id .data field and the core
     * copies it there via device_get_match_data(). In practice for meson
     * pinctrl it is accessed through of_device_get_match_data at probe time,
     * so platform_data may be NULL — we print what we safely can.
     */
    pr_info("meson8_kprobe: meson_pinctrl_probe called\n");
    pr_info("meson8_kprobe:   device name  : %s\n", dev_name(&pdev->dev));
    pr_info("meson8_kprobe:   driver name  : %s\n",
            pdev->dev.driver ? pdev->dev.driver->name : "<no driver yet>");
    pr_info("meson8_kprobe:   platform_data: %s\n",
            pdev->dev.platform_data ? "present" : "NULL (uses of_match_data)");

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/*  kprobe post-handler: called right after meson_pinctrl_probe runs   */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* ax/x0 on return holds the return value (int) */
    long retval = regs_return_value(regs);

    pr_info("meson8_kprobe: meson_pinctrl_probe returned %ld (%s)\n",
            retval, retval == 0 ? "OK" : "ERROR");
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "meson_pinctrl_probe", /* exported symbol to hook      */
    .pre_handler = handler_pre,           /* fires before the function    */
    .post_handler = handler_post,         /* fires after the function     */
};

/* ------------------------------------------------------------------ */
/*  module_init: register the kprobe                                    */
/* ------------------------------------------------------------------ */
static int __init meson8_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("meson8_kprobe: register_kprobe failed, ret=%d\n", ret);
        pr_err("meson8_kprobe: is meson_pinctrl_probe exported/built-in?\n");
        return ret;
    }

    pr_info("meson8_kprobe: planted kprobe at %p (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit: unregister the kprobe before the module is removed    */
/* ------------------------------------------------------------------ */
static void __exit meson8_kprobe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("meson8_kprobe: kprobe removed from %p\n", kp.addr);
}

module_init(meson8_kprobe_init);
module_exit(meson8_kprobe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe hook on meson_pinctrl_probe for Meson8 pinctrl debugging");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ `MODULE_*` macros وكل اللي يخص تسجيل الـ module |
| `linux/kernel.h` | الـ `pr_info` / `pr_err` logging macros |
| `linux/kprobes.h` | الـ `struct kprobe` ودوال الـ `register_kprobe` |
| `linux/platform_device.h` | الـ `struct platform_device` اللي بنقرا منها `dev_name` |
| `linux/pinctrl/pinctrl.h` | الـ `struct pinctrl_pin_desc` — للتوثيق وإثبات إننا فاهمين الـ type |

**الـ** `meson_pinctrl_data_shadow` shadow struct بنعرّفها محلياً لأن الـ header `pinctrl-meson.h` موجود في `drivers/` مش في `include/linux/` — يعني مش accessible لأي module خارجي. بناخد بس الـ fields اللي محتاجينها بالترتيب الصح.

---

#### الـ `handler_pre`

ده بيشتغل **قبل** ما `meson_pinctrl_probe` تتنفذ. بياخد أول argument (الـ `pdev`) عن طريق `regs_get_kernel_argument(regs, 0)` اللي هو portable عبر كل architectures. بنطبع اسم الـ device واسم الـ driver عشان نتأكد أي bank (cbus أو aobus) اتسجّل.

---

#### الـ `handler_post`

بيشتغل **بعد** ما الدالة ترجع. `regs_return_value(regs)` بتجيب الـ return value من الـ register المناسب (x0/eax/rax). لو الـ probe اتكلّمت بعد boot مش هنشوف حاجة — لكن في حالة **hot-plug** أو **device tree overlay** ممكن يتسجّل device جديد وقتها.

---

#### الـ `kp` struct

```c
.symbol_name = "meson_pinctrl_probe"
```

الـ kernel بيحوّل الاسم لعنوان وقت `register_kprobe` عن طريق `kallsyms`. لازم الـ function تكون **non-inline** وموجودة في `kallsyms` — وده الحال هنا لأن `meson_pinctrl_probe` معرّفة في `pinctrl-meson.c` كـ `int meson_pinctrl_probe(...)` ومش `static`.

---

#### الـ `module_init` و `module_exit`

- **الـ init**: بيسجّل الـ kprobe. لو فشل (مثلاً الـ symbol مش موجود أو الـ kernel مبني بدون `CONFIG_KPROBES`) بيرجّع error.
- **الـ exit**: `unregister_kprobe` **إجباري** قبل إزالة الـ module. لو مش عملناه والـ module اتفك، الـ kernel هيحاول ينفّذ `handler_pre` في عنوان بقى invalid → **kernel panic**.

---

### Makefile للتجربة

```makefile
obj-m += meson8_kprobe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
sudo insmod meson8_kprobe.ko
sudo dmesg | grep meson8_kprobe
sudo rmmod meson8_kprobe
```

> **ملاحظة**: الـ kprobe هتشتغل فعلياً لو الـ `meson_pinctrl_probe` اتكلّمت بعد `insmod` (مثلاً عبر device tree overlay على Meson8 board). على x86 بيظهر فقط "planted kprobe" ثم عند `rmmod` رسالة الإزالة.
