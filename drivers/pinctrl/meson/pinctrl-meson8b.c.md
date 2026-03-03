## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الـ Subsystem؟

الـ file ده جزء من **AMLOGIC MESON SoC** subsystem في Linux kernel، وبالتحديد من **Pinctrl (Pin Control) framework**. المسؤول عنه في MAINTAINERS هو Xianwei Zhao، والـ mailing list هو `linux-amlogic` و `linux-gpio`.

---

### الصورة الكبيرة — القصة الحقيقية

#### المشكلة اللي بيحلها

تخيل معايا إنك عندك لوحة إلكترونية — **SoC (System on Chip)** اسمه **Amlogic Meson8b**، ده شريحة بتتحكم في تليفزيون ذكي أو Android TV Box. الشريحة دي فيها مئات الـ **pins** — الأرجل المعدنية اللي بتتوصل بيها المكونات ببعض.

كل pin ممكن يشتغل بأكتر من طريقة واحدة:
- ممكن يبعت بيانات SD card
- أو يبعت بيانات UART (serial port)
- أو يبعت بيانات SPI
- أو يكون GPIO عادي (high/low)
- أو يشيل إشارة Ethernet
- أو HDMI، I2C، I2S، وغيرهم

**المشكلة**: مين بيقرر إيه وظيفة كل pin وقت التشغيل؟ الـ kernel محتاج driver بيعرف الخريطة الكاملة للشريحة دي ويقدر يغير إعداداتها.

#### الحل — الـ Pinctrl Framework

الـ Linux kernel عنده framework اسمه **pinctrl** — زي "مدير الموارد" اللي بيقسم الـ pins على الـ drivers التانية. كل driver محتاج pin بيطلبه من الـ pinctrl، والـ pinctrl بيعمل التعديل في الـ hardware registers.

الـ file `pinctrl-meson8b.c` هو **قاموس الشريحة Meson8b** — بيقول للـ framework:
- "ده عندنا X pins، كل واحد اسمه إيه"
- "ده لو عايز تشغل SD card على bank A، استخدم الـ pins دي"
- "ده لو عايز UART، اضغط على البت ده في الـ register ده"

---

### المفاهيم الأساسية في الـ File

#### 1. الـ Buses — جزئين للشريحة

الـ Meson8b مقسم لجزئين:

| Bus | الوظيفة | الـ Pins |
|-----|---------|---------|
| **CBUS** (Controller Bus) | الـ pins العادية للـ peripherals | GPIOX, GPIOY, GPIODV, GPIOH, CARD, BOOT, DIF |
| **AOBUS** (Always-On Bus) | الـ pins اللي فاقية حتى لو الجهاز في sleep | GPIOAO |

الـ AOBUS مهمة لأن أجهزة زي remote control محتاجة تصحّي الجهاز من النوم — لازم تفضل واقفة.

#### 2. الـ Pins

```c
/* كل pin ليه رقم وعنوان في النظام */
static const struct pinctrl_pin_desc meson8b_cbus_pins[] = {
    MESON_PIN(GPIOX_0),   /* pin رقم GPIOX_0 */
    MESON_PIN(GPIOX_1),
    /* ... */
};
```

#### 3. الـ Groups — مجموعات الـ Pins لكل Function

كل **function** (وظيفة) بتتطلب مجموعة pins معينة:

```c
/* الـ pins اللازمة لـ SD card على bank A */
static const unsigned int sd_d0_a_pins[] = { GPIOX_0 };
static const unsigned int sd_clk_a_pins[] = { GPIOX_8 };
static const unsigned int sd_cmd_a_pins[] = { GPIOX_9 };
```

#### 4. الـ Groups Table — الربط بين الـ Group والـ Register

```c
GROUP(sd_d0_a, 8, 5),
/* GROUP(اسم_الجروب, رقم_الريجستر, رقم_البت) */
```

يعني: عشان تفعّل الـ `sd_d0_a`، اضرب bit رقم 5 في register رقم 8.

#### 5. الـ Functions — الوظائف النهائية

```c
static const struct meson_pmx_func meson8b_cbus_functions[] = {
    FUNCTION(gpio_periphs),  /* GPIO عادي */
    FUNCTION(sd_a),          /* SD card على bank A */
    FUNCTION(uart_a),        /* UART A */
    FUNCTION(ethernet),      /* Ethernet */
    FUNCTION(hdmi),          /* HDMI */
    /* ... */
};
```

#### 6. الـ Banks — الـ Registers اللي بتتحكم في الـ Pins

```c
/*  name       first      last     irq      pullen  pull   dir    out    in  */
BANK("X0..11", GPIOX_0, GPIOX_11, 97, 108, 4, 0,  4, 0,  0, 0,  1, 0,  2, 0),
```

كل bank بيحدد: أول وآخر pin، وأرقام الـ registers المسؤولة عن:
- **pullen**: تفعيل الـ pull resistor
- **pull**: اتجاه الـ pull (up أو down)
- **dir**: الاتجاه (input أو output)
- **out**: قيمة الخروج
- **in**: قراءة الدخل

---

### قصة تخيلية — سيناريو حقيقي

تخيل إن عندك Android TV Box بشريحة Meson8b. وقت boot:

1. الـ kernel بيلاقي في **Device Tree** إن فيه `amlogic,meson8b-cbus-pinctrl`
2. بيشغّل الـ `meson8b_pinctrl_driver` → بيستدعي `meson_pinctrl_probe`
3. الـ SD card driver بيطلب: "أنا محتاج SD interface على bank B"
4. الـ pinctrl بيشوف في `meson8b_cbus_groups`: "sd_b → CARD_0, CARD_1, CARD_2, CARD_3, CARD_4, CARD_5"
5. بيشوف `GROUP(sd_d0_b, 2, 15)` → بيكتب bit 15 في register 2
6. الـ SD card دلوقتي شغّالة!

لو بعدين UART driver طلب نفس الـ pins — الـ pinctrl بيرفض لأنها محجوزة.

---

### الـ ASCII Diagram — معمارية الـ Pinctrl في Meson8b

```
  ┌─────────────────────────────────────────────────┐
  │              Linux Kernel                        │
  │                                                  │
  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
  │  │ SD Driver│  │ETH Driver│  │ UART Driver  │   │
  │  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
  │       │              │               │            │
  │       └──────────────┴───────────────┘            │
  │                      │                            │
  │              ┌────────▼────────┐                  │
  │              │ pinctrl core    │  (generic layer) │
  │              └────────┬────────┘                  │
  │                       │                           │
  │          ┌────────────▼────────────┐              │
  │          │  pinctrl-meson.c        │  (Meson ops) │
  │          └────────────┬────────────┘              │
  │                       │                           │
  │    ┌──────────────────▼──────────────────┐        │
  │    │       pinctrl-meson8b.c              │        │
  │    │  (pin table + group table +          │        │
  │    │   function table + bank table)       │        │
  │    └──────────────────┬──────────────────┘        │
  └───────────────────────┼──────────────────────────┘
                          │  regmap writes
              ┌───────────▼───────────┐
              │   Meson8b Hardware     │
              │  ┌─────┐   ┌───────┐  │
              │  │CBUS │   │ AOBUS │  │
              │  │Regs │   │ Regs  │  │
              │  └──┬──┘   └───┬───┘  │
              │     │          │      │
              │  GPIOX..DIF  GPIOAO   │
              └───────────────────────┘
```

---

### الـ Files المهمة اللي المبرمج لازم يعرفها

#### Core Framework (Generic)
| الـ File | الوظيفة |
|---------|---------|
| `drivers/pinctrl/meson/pinctrl-meson.c` | الكود العام لكل SoCs — الـ probe، الـ GPIO ops |
| `drivers/pinctrl/meson/pinctrl-meson.h` | الـ structs والـ macros المشتركة: `meson_bank`، `meson_pmx_group`، `meson_pinctrl_data` |
| `drivers/pinctrl/meson/pinctrl-meson8-pmx.c` | الكود المسؤول عن الـ pinmux (تغيير وظيفة الـ pin) لجيل Meson8 |
| `drivers/pinctrl/meson/pinctrl-meson8-pmx.h` | الـ macros: `GROUP`، `GPIO_GROUP`، `PMX_DATA` |

#### SoC-Specific Data Files (نفس المنهج)
| الـ File | الشريحة |
|---------|---------|
| `drivers/pinctrl/meson/pinctrl-meson8b.c` | **Meson8b** (ملفنا) |
| `drivers/pinctrl/meson/pinctrl-meson8.c` | Meson8 (شقيقة أقدم) |
| `drivers/pinctrl/meson/pinctrl-meson-gxbb.c` | GXBB SoC |
| `drivers/pinctrl/meson/pinctrl-meson-gxl.c` | GXL SoC |
| `drivers/pinctrl/meson/pinctrl-meson-g12a.c` | G12A SoC |
| `drivers/pinctrl/meson/pinctrl-meson-axg.c` | AXG SoC |

#### Device Tree Bindings
| الـ File | الوظيفة |
|---------|---------|
| `include/dt-bindings/gpio/meson8b-gpio.h` | تعريف أسماء وأرقام الـ pins للـ Device Tree |
| `arch/arm/boot/dts/amlogic/` | الـ Device Tree files للـ boards اللي بتستخدم Meson8b |

#### Generic Pinctrl Layer
| الـ File | الوظيفة |
|---------|---------|
| `include/linux/pinctrl/pinctrl.h` | الـ API العام للـ pinctrl framework |
| `include/linux/pinctrl/pinmux.h` | الـ API للـ pinmux operations |

---

### خلاصة — ليه الـ File ده موجود؟

الـ file `pinctrl-meson8b.c` هو **data-only file** — مفيش خوارزميات معقدة فيه. كل اللي فيه هو:

1. **قائمة الـ pins** — كل pin في الشريحة بالاسم والرقم
2. **جدول الـ groups** — كل function محتاجة إيه من pins وفين الـ register والبت
3. **جدول الـ functions** — الوظائف المتاحة (SD، UART، SPI، Ethernet، ...)
4. **جدول الـ banks** — الـ registers اللي بتتحكم في direction، pull، وغيرهم

الكود الحقيقي اللي بيعمل الشغل موجود في `pinctrl-meson.c` و`pinctrl-meson8-pmx.c`. الـ file ده بس بيديهم البيانات الخاصة بـ Meson8b.

الفكرة دي ذكية — لو Amlogic عملت شريحة جديدة، بيعملوا file جديد بنفس الشكل بالبيانات الجديدة فقط، من غير ما يعيدوا كتابة أي منطق.
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة اللي بيحلها الـ Subsystem ده

أي SoC حديث — زي الـ Amlogic Meson8b اللي احنا بنشوفه — فيه مئات الـ physical pins. كل pin ممكن يشتغل بأكتر من وظيفة واحدة في نفس الوقت (مش في نفس اللحظة، بس نفس الـ physical pad بيقدر يعمل UART أو SPI أو GPIO أو I2C).

**المشكلة الحقيقية** إنك لو سبت كل driver يـ configure الـ pins بنفسه، هيحصل chaos كامل:
- الـ UART driver ممكن يـ conflict مع الـ SPI driver على نفس الـ pin.
- محدش عارف مين اخد الـ pin ده.
- الـ bootloader بيـ configure الـ pins بطريقة، والـ kernel بيحاول يعمل حاجة تانية.
- كل SoC بيعمل الـ mux بطريقة مختلفة جداً في الـ registers.

بدون framework مركزي، كل board كانت بتحتاج patch في كل driver عشان تـ hardcode الـ pin configuration بتاعتها.

---

### الحل اللي الكرنل اختاره

الـ **pinctrl subsystem** هو الحل. الفكرة المركزية إنه بيعمل abstraction layer واحدة بين:
- الـ **hardware**: الـ physical pins والـ mux registers في كل SoC.
- الـ **consumers**: الـ drivers اللي محتاجة pins زي UART, SPI, I2C.
- الـ **Device Tree**: اللي بيوصف الـ board configuration.

الكرنل بيتحكم في كل عملية تحويل pin من function لـ function في مكان واحد، وبيمنع conflict، وبيوفر API موحد لكل الـ drivers.

---

### التشبيه من الواقع (Mapped بالكامل)

تخيل **لوحة توصيل كهربائية في مبنى** (patch panel في data center):

| المفهوم في الـ Patch Panel | المفهوم في الـ pinctrl |
|---|---|
| الـ physical ports في الـ panel | الـ `pinctrl_pin_desc` — كل pin له رقم واسم |
| الـ cable group اللي بيوصل room لـ room | الـ `meson_pmx_group` — مجموعة pins بتخدم function واحدة |
| نوع الخدمة (data, voice, video) | الـ `meson_pmx_func` — (uart_a, spi, ethernet...) |
| الـ physical switch في الـ panel | الـ **mux register bit** (مثلاً `GROUP(uart_tx_a, 4, 17)` = reg 4, bit 17) |
| مدير الـ data center | الـ **pinctrl core** في الكرنل |
| الـ electrician اللي يعرف الـ panel ده بالتحديد | الـ **pinctrl-meson8b.c** — الـ SoC-specific driver |
| طلب من قسم IT لتوصيل port معين | الـ consumer driver يطلب `pinctrl_select_state()` |
| الـ floor plan بتاع المبنى | الـ Device Tree |

كل لما الـ UART driver بيـ probe، بيطلب من "مدير الـ data center" (الـ pinctrl core) إنه يوصل الـ cables المناسبة (يـ set الـ mux bits). المدير بيرجع للـ electrician (meson8b driver) اللي يعرف أي switch يقلب.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                    Consumer Drivers                          │
  │  [uart-meson]  [spi-meson]  [mmc-meson]  [i2c-meson]  ...  │
  │       │              │             │            │            │
  │       └──────────────┴─────────────┴────────────┘            │
  │                          │                                   │
  │                 pinctrl_select_state()                       │
  └─────────────────────────────────────────────────────────────┘
                             │
  ┌─────────────────────────▼─────────────────────────────────┐
  │                  pinctrl CORE  (kernel/pinctrl/)            │
  │                                                             │
  │   ┌────────────┐   ┌────────────┐   ┌────────────────┐    │
  │   │  pinmux    │   │  pinconf   │   │  GPIO bridge   │    │
  │   │  (pmx ops) │   │  (conf ops)│   │ (gpio_chip)    │    │
  │   └─────┬──────┘   └─────┬──────┘   └───────┬────────┘    │
  └─────────┼────────────────┼────────────────────┼────────────┘
            │                │                    │
  ┌─────────▼────────────────▼────────────────────▼────────────┐
  │           pinctrl-meson.c  (Meson Generic Layer)            │
  │                                                             │
  │  meson_pinctrl_probe()   meson_pmx_ops   meson_pinconf_ops  │
  │  meson_gpio_chip_ops                                        │
  └─────────────────────────┬───────────────────────────────────┘
                             │ uses
  ┌──────────────────────────▼──────────────────────────────────┐
  │          pinctrl-meson8b.c  (THIS FILE — SoC Data)          │
  │                                                             │
  │  meson8b_cbus_pins[]       meson8b_aobus_pins[]             │
  │  meson8b_cbus_groups[]     meson8b_aobus_groups[]           │
  │  meson8b_cbus_functions[]  meson8b_aobus_functions[]        │
  │  meson8b_cbus_banks[]      meson8b_aobus_banks[]            │
  │  meson8b_cbus_pinctrl_data  meson8b_aobus_pinctrl_data      │
  └─────────────────────────────────────────────────────────────┘
                             │ talks to HW via
  ┌──────────────────────────▼──────────────────────────────────┐
  │                  regmap  (MMIO registers)                    │
  │                                                             │
  │   reg_mux    reg_pullen    reg_pull    reg_gpio    reg_ds   │
  └─────────────────────────────────────────────────────────────┘
                             │
  ┌──────────────────────────▼──────────────────────────────────┐
  │              Amlogic Meson8b SoC Hardware                    │
  │                                                             │
  │   CBUS domain (reg_mux regs 0-9)    AOBUS domain (reg 0)    │
  │   GPIOX, GPIOY, GPIODV,            GPIOAO_0..13            │
  │   GPIOH, CARD, BOOT, DIF           GPIO_BSD_EN, TEST_N     │
  └─────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction

الـ pinctrl framework بيبني abstraction على 3 مستويات متداخلة:

```
  Physical Level:   Pin  (pinctrl_pin_desc)
                     │
  Grouping Level:   Group (meson_pmx_group)  ← مجموعة pins لها معنى مشترك
                     │
  Function Level:   Function (meson_pmx_func) ← الخدمة اللي الـ group بتقدمها
```

#### المستوى الأول — الـ `pinctrl_pin_desc`

ده أبسط حاجة. كل pin في الـ SoC بيتعرف بـ:
- **رقم عالمي unique** (`number`) — الـ enum value زي `GPIOX_0`.
- **اسم** (`name`) — string للـ debugfs والـ DT.

```c
/* MESON_PIN(GPIOX_0) يتحول لـ: */
{ .number = GPIOX_0, .name = "GPIOX_0" }
```

الـ `meson8b_cbus_pins[]` فيه كل الـ 103 pin في الـ CBUS domain، والـ `meson8b_aobus_pins[]` فيه 16 pin في الـ AOBUS domain.

#### المستوى الثاني — الـ `meson_pmx_group`

الـ group هي المفهوم المحوري. هي مجموعة pins بتشتغل مع بعض عشان تأدي وظيفة معينة.

```c
/*
 * GROUP(uart_tx_a, 4, 17) يتحول لـ:
 * - الاسم: "uart_tx_a"
 * - الـ pins: uart_tx_a_pins[] = { GPIOX_4 }
 * - الـ mux data: reg=4, bit=17, is_gpio=false
 */
static const struct meson_pmx_group meson8b_cbus_groups[] = {
    /* ... */
    GROUP(uart_tx_a,  4, 17),  /* set bit 17 in mux reg 4 */
    GROUP(uart_rx_a,  4, 16),
    /* ... */
    GPIO_GROUP(GPIOX_4),       /* no mux bit needed for GPIO mode */
};
```

**تفصيل مهم:** نفس الـ physical pin (مثلاً GPIOX_4) ممكن يظهر في أكتر من group:
- `uart_tx_a_pins[] = { GPIOX_4 }` — لما هو UART TX
- `sdxc_d0_0_a_pins[] = { GPIOX_4 }` — لما هو SDXC data pin
- `pcm_out_a_pins[] = { GPIOX_4 }` — لما هو PCM out

ده معناه إن الـ pinctrl core بيمنع تعيين اكتر من group على نفس الـ pin في نفس الوقت.

#### المستوى الثالث — الـ `meson_pmx_func`

الـ function بتجمع مجموعة من الـ groups اللي كلها بتخدم نفس الـ peripheral.

```c
/* FUNCTION(uart_a) يتحول لـ: */
{
    .name      = "uart_a",
    .groups    = uart_a_groups,       /* ["uart_tx_a", "uart_rx_a", ...] */
    .num_groups = ARRAY_SIZE(uart_a_groups),
}
```

الـ Device Tree بيشير للـ function باسمها، والكرنل بيبحث عن الـ groups المطلوبة جوا الـ function ده.

---

### الـ Bank — المفهوم اللي بيربط كل حاجة بالـ GPIO

الـ **`meson_bank`** هي إضافة Meson-specific فوق الـ pinctrl standard. هي بتصف مجموعة pins بتتحكم فيها بنفس الـ registers الـ GPIO (direction, pull, output, input).

```c
/*
 * BANK("X0..11", GPIOX_0, GPIOX_11, 97, 108,
 *       4, 0,   <- pullen_reg=4, pullen_bit_start=0
 *       4, 0,   <- pull_reg=4,   pull_bit_start=0
 *       0, 0,   <- dir_reg=0,    dir_bit_start=0
 *       1, 0,   <- out_reg=1,    out_bit_start=0
 *       2, 0)   <- in_reg=2,     in_bit_start=0
 */
BANK("X0..11", GPIOX_0, GPIOX_11, 97, 108, 4, 0, 4, 0, 0, 0, 1, 0, 2, 0),
```

كل pin جوا الـ bank بيتحكم فيه بـ bit offset = `pin_number - bank.first` داخل الـ register المذكور.

```
مثال: GPIOX_3 في bank "X0..11"
  - offset = GPIOX_3 - GPIOX_0 = 3
  - dir register = reg 0, bit (0 + 3) = bit 3
  - out register = reg 1, bit (0 + 3) = bit 3
  - in  register = reg 2, bit (0 + 3) = bit 3
```

---

### العلاقة بين الـ Structs (Graph)

```
meson_pinctrl_data
  ├── pins[]         → pinctrl_pin_desc[]    (كل pin بـ number + name)
  │
  ├── groups[]       → meson_pmx_group[]
  │     ├── name     → "uart_tx_a"
  │     ├── pins[]   → { GPIOX_4 }
  │     └── data     → meson8_pmx_data
  │           ├── reg   = 4         (mux register index)
  │           ├── bit   = 17        (bit to set for this function)
  │           └── is_gpio = false
  │
  ├── funcs[]        → meson_pmx_func[]
  │     ├── name     → "uart_a"
  │     └── groups[] → ["uart_tx_a", "uart_rx_a", "uart_cts_a", "uart_rts_a"]
  │
  ├── banks[]        → meson_bank[]
  │     ├── first / last  → range of pin numbers
  │     ├── irq_first / irq_last
  │     └── regs[]   → meson_reg_desc[MESON_NUM_REG]
  │           ├── [MESON_REG_PULLEN] = { reg, bit }
  │           ├── [MESON_REG_PULL]   = { reg, bit }
  │           ├── [MESON_REG_DIR]    = { reg, bit }
  │           ├── [MESON_REG_OUT]    = { reg, bit }
  │           └── [MESON_REG_IN]     = { reg, bit }
  │
  └── pmx_ops        → &meson8_pmx_ops   (الـ vtable لعمليات الـ mux)


meson_pinctrl  (runtime state — instance per probe)
  ├── data     → meson_pinctrl_data*  (الـ static data من فوق)
  ├── pcdev    → pinctrl_dev*         (الـ handle في الـ pinctrl core)
  ├── desc     → pinctrl_desc         (ما بيتـ register في الـ core)
  ├── reg_mux  → regmap*              (MMIO للـ mux registers)
  ├── reg_gpio → regmap*              (MMIO للـ GPIO registers)
  ├── reg_pull → regmap*
  ├── reg_pullen → regmap*
  └── chip     → gpio_chip            (الـ GPIO subsystem integration)
```

---

### الـ Two-Domain Design: CBUS و AOBUS

الـ Meson8b بيفصل الـ pins لـ domain تانيين مختلفين تماماً:

```
  ┌──────────────────────────────────┐   ┌──────────────────────────────┐
  │         CBUS Domain              │   │        AOBUS Domain           │
  │  (Clock/Bus — main power)        │   │  (Always-On — stays powered  │
  │                                  │   │   even in suspend)           │
  │  GPIOX, GPIOY, GPIODV            │   │                              │
  │  GPIOH, CARD, BOOT, DIF          │   │  GPIOAO_0..13                │
  │                                  │   │  GPIO_BSD_EN, GPIO_TEST_N    │
  │  103 pins                        │   │  16 pins                     │
  │  31 functions                    │   │  11 functions                │
  │  compatible:                     │   │  compatible:                 │
  │  "amlogic,meson8b-cbus-pinctrl"  │   │  "amlogic,meson8b-aobus-…"  │
  └──────────────────────────────────┘   └──────────────────────────────┘
              │                                       │
              └─────────────────┬─────────────────────┘
                                │
                  meson_pinctrl_probe() يتم استدعاؤه مرتين
                  (مرة لكل compatible string في الـ DT)
```

الـ AOBUS عنده `parse_dt = &meson8_aobus_parse_dt_extra` لأن الـ AOBUS registers بتيجي من DT بطريقة مختلفة عن الـ CBUS.

---

### مثال عملي: إزاي UART TX يشتغل على GPIOX_4

لما الـ UART driver بيطلب "uart_a" state من الـ DT:

```
1. pinctrl core → يبحث عن function "uart_a"
   → يلاقيها في meson8b_cbus_functions[]

2. pinctrl core → يأخذ groups من uart_a_groups[]
   → ["uart_tx_a", "uart_rx_a", "uart_cts_a", "uart_rts_a"]

3. لكل group → يجيب الـ meson8_pmx_data من .data
   uart_tx_a: reg=4, bit=17

4. meson8_pmx_ops.set_mux() → يستدعي
   regmap_update_bits(pc->reg_mux, 4 * 4, BIT(17), BIT(17))
   ← بيكتب 1 في bit 17 في register رقم 4

5. الـ hardware الـ Meson8b بيشوف الـ bit ده
   → بيوصل الـ GPIOX_4 pad لـ UART TX signal داخلياً
```

---

### الـ GPIO Integration

الـ pinctrl في Meson بيكون في نفس الوقت **GPIO controller**. الـ `gpio_chip` جوا `meson_pinctrl` بيوفر:
- `gpio_request` → بيسمح بـ GPIO mode لـ pin معين (بيزيل الـ mux bit).
- `gpio_direction_input/output` → بيكتب في `MESON_REG_DIR` بتاع الـ bank.
- `gpio_get/set` → بيقرأ/يكتب في `MESON_REG_IN`/`MESON_REG_OUT`.

ده بيعني إنك لازم تفهم الـ **GPIO subsystem** في الكرنل (`include/linux/gpio/driver.h`) عشان تفهم الـ integration. الـ GPIO subsystem هو اللي بيوفر `/sys/class/gpio` وـ `gpiod_*` APIs للـ user space والـ drivers.

---

### إيه اللي الـ Framework بيملكه وإيه اللي بيـ Delegate للـ Driver

| المسؤولية | مين بيعملها |
|---|---|
| منع الـ pin conflict | **pinctrl core** — بيرفض تعيين pin لأكتر من function |
| ترتيب الـ state transitions | **pinctrl core** — عبر `pinctrl_map` |
| تحليل الـ DT `pinctrl-0`, `pinctrl-names` | **pinctrl core** |
| تعريف كل pin بـ number واسم | **meson8b.c** — `meson8b_cbus_pins[]` |
| تعريف الـ groups وأي pins فيها | **meson8b.c** — `meson8b_cbus_groups[]` |
| تعريف الـ functions وأي groups فيها | **meson8b.c** — `meson8b_cbus_functions[]` |
| معرفة أي register/bit يتحكم في كل function | **meson8b.c** — `GROUP(name, reg, bit)` |
| معرفة أي register/bit للـ GPIO direction/pull | **meson8b.c** — `BANK(...)` |
| الكتابة الفعلية في الـ hardware registers | **pinctrl-meson.c** — `meson8_pmx_ops` via **regmap** |
| الـ GPIO chip operations | **pinctrl-meson.c** — `meson_gpio_chip_ops` |
| إنشاء الـ regmap instances | **pinctrl-meson.c** — `meson_pinctrl_probe()` |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Flags والـ Config Options — Cheatsheet

#### `enum meson_reg_type` — أنواع الـ Registers

| القيمة | الوصف |
|--------|-------|
| `MESON_REG_PULLEN` | register التحكم في تفعيل الـ pull resistor |
| `MESON_REG_PULL` | register اتجاه الـ pull (up/down) |
| `MESON_REG_DIR` | register اتجاه الـ pin (input/output) |
| `MESON_REG_OUT` | register كتابة القيمة على الـ pin |
| `MESON_REG_IN` | register قراءة القيمة من الـ pin |
| `MESON_REG_DS` | register قوة الإشارة (drive strength) |
| `MESON_NUM_REG` | عدد الأنواع (مستخدم لحجم الـ array) |

#### `enum meson_pinconf_drv` — قيم الـ Drive Strength

| القيمة | التيار |
|--------|--------|
| `MESON_PINCONF_DRV_500UA` | 500 µA |
| `MESON_PINCONF_DRV_2500UA` | 2500 µA |
| `MESON_PINCONF_DRV_3000UA` | 3000 µA |
| `MESON_PINCONF_DRV_4000UA` | 4000 µA |

#### الـ GPIO Banks في Meson8b

| Bank | أول Pin | آخر Pin | عدد الـ IRQs | الوصف |
|------|---------|---------|--------------|-------|
| `X0..11` | GPIOX_0 | GPIOX_11 | 12 | General purpose — وصل إشارات خارجية |
| `X16..21` | GPIOX_16 | GPIOX_21 | 6 | امتداد GPIOX |
| `Y0..1` | GPIOY_0 | GPIOY_1 | 2 | TSIN/SPDIF |
| `Y3` | GPIOY_3 | GPIOY_3 | 1 | SPDIF / XTAL |
| `Y6..14` | GPIOY_6 | GPIOY_14 | 9 | TSIN data bus |
| `DV9` | GPIODV_9 | GPIODV_9 | 1 | PWM |
| `DV24..29` | GPIODV_24 | GPIODV_29 | 6 | I2C / UART |
| `H` | GPIOH_0 | GPIOH_9 | 10 | HDMI / SPI / Ethernet |
| `CARD` | CARD_0 | CARD_6 | 7 | SD card B |
| `BOOT` | BOOT_0 | BOOT_18 | 19 | NAND / eMMC / NOR |
| `DIF` | DIF_0_P | DIF_4_N | لا يوجد | Ethernet differential |
| `AO` | GPIOAO_0 | GPIO_TEST_N | 14 | Always-On domain |

#### الـ Macros المهمة

| الـ Macro | الهدف |
|-----------|-------|
| `MESON_PIN(x)` | ينشئ `pinctrl_pin_desc` من اسم الـ pin |
| `GROUP(grp, r, b)` | ينشئ `meson_pmx_group` مع بيانات الـ register والـ bit |
| `GPIO_GROUP(gpio)` | ينشئ group خاص بـ GPIO mode (`is_gpio = true`) |
| `FUNCTION(fn)` | ينشئ `meson_pmx_func` يربط function بمجموعة groups |
| `BANK(n, f, l, ...)` | ينشئ `meson_bank` بدون drive strength |
| `BANK_DS(...)` | ينشئ `meson_bank` مع drive strength |
| `PMX_DATA(r, b, g)` | ينشئ `meson8_pmx_data` داخلي للـ group |

---

### 1. الـ Structs المهمة

#### `struct pinctrl_pin_desc`
**المصدر**: `<linux/pinctrl/pinctrl.h>` — معرّف في الـ kernel core

**الغرض**: وصف pin واحد بالنسبة لنظام الـ pinctrl.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `number` | `unsigned` | رقم الـ pin (global index) |
| `name` | `const char *` | اسم الـ pin مثل `"GPIOX_0"` |
| `drv_data` | `void *` | بيانات خاصة بالـ driver (غير مستخدمة هنا) |

```c
/* يُنشأ باستخدام الـ macro */
MESON_PIN(GPIOX_0)
/* يُوسَّع إلى: */
PINCTRL_PIN(GPIOX_0, "GPIOX_0")
```

---

#### `struct meson8_pmx_data`
**المصدر**: `pinctrl-meson8-pmx.h`

**الغرض**: البيانات الخاصة بـ Meson8 لكل group — تحدد الـ register والـ bit اللي يتحكم في تفعيل الـ function على الـ hardware.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `is_gpio` | `bool` | لو `true` الـ pin في GPIO mode (مش function) |
| `reg` | `unsigned int` | offset الـ register في الـ mux regmap |
| `bit` | `unsigned int` | رقم الـ bit داخل الـ register |

```c
/* مثال: GROUP(sd_d0_a, 8, 5) */
/* يعني: set bit 5 في register offset 8 لتفعيل sd_d0_a */
.data = (const struct meson8_pmx_data[]){
    PMX_DATA(8, 5, false),
}
```

---

#### `struct meson_pmx_group`
**المصدر**: `pinctrl-meson.h`

**الغرض**: تجميع مجموعة pins تحت اسم واحد مع بيانات الـ mux register. الـ group هو الوحدة الأساسية في الـ pinmux.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `name` | `const char *` | اسم الـ group مثل `"sd_d0_a"` |
| `pins` | `const unsigned int *` | مصفوفة أرقام الـ pins في الـ group |
| `num_pins` | `unsigned int` | عدد الـ pins |
| `data` | `const void *` | pointer لـ `meson8_pmx_data` |

**ملاحظة**: في Meson8b لكل group بيانات `meson8_pmx_data` واحدة — يعني كل الـ pins في الـ group تُفعَّل بـ bit واحدة.

---

#### `struct meson_pmx_func`
**المصدر**: `pinctrl-meson.h`

**الغرض**: الـ function هو اسم الوظيفة الكاملة (مثل UART، SPI) وبيجمع عدة groups تحته.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `name` | `const char *` | اسم الـ function مثل `"uart_a"` |
| `groups` | `const char * const *` | مصفوفة أسماء الـ groups |
| `num_groups` | `unsigned int` | عدد الـ groups |

```c
/* مثال: FUNCTION(uart_a) */
{
    .name = "uart_a",
    .groups = uart_a_groups,   /* {"uart_tx_a", "uart_rx_a", ...} */
    .num_groups = 4,
}
```

---

#### `struct meson_reg_desc`
**المصدر**: `pinctrl-meson.h`

**الغرض**: يصف موقع تحكم واحد (bit في register) لخاصية معينة للـ pin.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `reg` | `unsigned int` | offset الـ register في الـ regmap |
| `bit` | `unsigned int` | رقم الـ bit |

---

#### `struct meson_bank`
**المصدر**: `pinctrl-meson.h`

**الغرض**: يمثل مجموعة pins متجاورة تتشارك نفس الـ register layout. الـ bank هو الوحدة الأساسية للتحكم في GPIO (direction, pull, output, input).

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `name` | `const char *` | اسم الـ bank مثل `"X0..11"` |
| `first` | `unsigned int` | رقم أول pin في الـ bank |
| `last` | `unsigned int` | رقم آخر pin في الـ bank |
| `irq_first` | `int` | أول hwirq مرتبط بالـ bank (-1 لو مفيش) |
| `irq_last` | `int` | آخر hwirq مرتبط بالـ bank |
| `regs[MESON_NUM_REG]` | `meson_reg_desc[6]` | descriptors لكل نوع register |

```c
/* مثال: BANK("X0..11", GPIOX_0, GPIOX_11, 97, 108, 4, 0, 4, 0, 0, 0, 1, 0, 2, 0) */
/* يعني:
   pullen  → reg=4,  bit=0  (لكل pin offset بالـ index داخل الـ bank)
   pull    → reg=4,  bit=0
   dir     → reg=0,  bit=0
   out     → reg=1,  bit=0
   in      → reg=2,  bit=0
*/
```

---

#### `struct meson_pinctrl_data`
**المصدر**: `pinctrl-meson.h`

**الغرض**: البيانات الثابتة (compile-time) لكل controller — كل شيء الـ probe function محتاجه لإعداد الـ controller.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `name` | `const char *` | اسم الـ controller |
| `pins` | `pinctrl_pin_desc *` | مصفوفة كل الـ pins |
| `groups` | `meson_pmx_group *` | مصفوفة كل الـ groups |
| `funcs` | `meson_pmx_func *` | مصفوفة كل الـ functions |
| `num_pins` | `unsigned int` | عدد الـ pins |
| `num_groups` | `unsigned int` | عدد الـ groups |
| `num_funcs` | `unsigned int` | عدد الـ functions |
| `banks` | `meson_bank *` | مصفوفة الـ banks |
| `num_banks` | `unsigned int` | عدد الـ banks |
| `pmx_ops` | `pinmux_ops *` | عمليات الـ pinmux (تشير لـ `meson8_pmx_ops`) |
| `pmx_data` | `void *` | بيانات إضافية للـ pmx (غير مستخدمة في Meson8b) |
| `parse_dt` | `int (*)(meson_pinctrl *)` | دالة تحليل DT إضافية (للـ aobus فقط) |

---

#### `struct meson_pinctrl`
**المصدر**: `pinctrl-meson.h`

**الغرض**: الحالة الكاملة لـ controller واحد في الـ runtime — بيُخزَّن كـ driver private data.

| الحقل | النوع | الوصف |
|-------|-------|-------|
| `dev` | `struct device *` | الـ device المرتبط بالـ platform driver |
| `pcdev` | `struct pinctrl_dev *` | handle الـ pinctrl core |
| `desc` | `struct pinctrl_desc` | وصف الـ controller (ops, pins, إلخ) |
| `data` | `meson_pinctrl_data *` | البيانات الثابتة لهذا الـ controller |
| `reg_mux` | `struct regmap *` | regmap لـ function mux registers |
| `reg_pullen` | `struct regmap *` | regmap لـ pull-enable registers |
| `reg_pull` | `struct regmap *` | regmap لـ pull direction registers |
| `reg_gpio` | `struct regmap *` | regmap لـ GPIO dir/in/out registers |
| `reg_ds` | `struct regmap *` | regmap لـ drive strength registers |
| `chip` | `struct gpio_chip` | الـ GPIO chip المُدمج |
| `fwnode` | `struct fwnode_handle *` | الـ firmware node من DT |

---

### 2. مخططات العلاقات بين الـ Structs

```
                    ┌─────────────────────────────┐
                    │   meson_pinctrl_data         │
                    │  (compile-time static data)  │
                    │                              │
                    │  .pins ─────────────────────►│ pinctrl_pin_desc[]
                    │  .groups ───────────────────►│ meson_pmx_group[]
                    │  .funcs ────────────────────►│ meson_pmx_func[]
                    │  .banks ────────────────────►│ meson_bank[]
                    │  .pmx_ops ──────────────────►│ pinmux_ops (meson8_pmx_ops)
                    │  .parse_dt ─────────────────►│ meson8_aobus_parse_dt_extra()
                    └─────────────────────────────┘
                               ▲
                               │ .data points to
                    ┌──────────┴──────────────────┐
                    │      meson_pinctrl           │
                    │   (runtime per-controller)  │
                    │                              │
                    │  .dev ──────────────────────►│ struct device
                    │  .pcdev ────────────────────►│ struct pinctrl_dev (kernel)
                    │  .desc ─────────────────────►│ struct pinctrl_desc
                    │  .reg_mux ──────────────────►│ struct regmap
                    │  .reg_gpio ─────────────────►│ struct regmap
                    │  .reg_pull ─────────────────►│ struct regmap
                    │  .reg_pullen ───────────────►│ struct regmap
                    │  .reg_ds ───────────────────►│ struct regmap
                    │  .chip ─────────────────────►│ struct gpio_chip
                    └─────────────────────────────┘


   meson_pmx_group                    meson_pmx_func
  ┌──────────────────┐               ┌──────────────────┐
  │ .name = "sd_d0_a"│        ┌─────►│ .name = "sd_a"   │
  │ .pins → [GPIOX_0]│        │      │ .groups → [       │
  │ .num_pins = 1    │        │      │   "sd_d0_a",      │
  │ .data ──────────►│─┐      │      │   "sd_d1_a", ...] │
  └──────────────────┘ │      │      │ .num_groups = 6   │
                        │      │      └──────────────────┘
  ┌──────────────────┐ │      │
  │ meson8_pmx_data  │◄┘      │  الـ function يُشير للـ group
  │ .reg = 8         │        │  بالاسم (string lookup)
  │ .bit = 5         │        │
  │ .is_gpio = false │        │
  └──────────────────┘        │

   meson_bank                         meson_reg_desc
  ┌──────────────────────────┐       ┌─────────────┐
  │ .name = "X0..11"         │       │ .reg = 4    │
  │ .first = GPIOX_0         │       │ .bit = 0    │
  │ .last  = GPIOX_11        │       └─────────────┘
  │ .irq_first = 97          │
  │ .irq_last  = 108         │  regs[MESON_NUM_REG]:
  │ .regs[PULLEN] ──────────►│─────► {reg=4, bit=0}
  │ .regs[PULL]   ──────────►│─────► {reg=4, bit=0}
  │ .regs[DIR]    ──────────►│─────► {reg=0, bit=0}
  │ .regs[OUT]    ──────────►│─────► {reg=1, bit=0}
  │ .regs[IN]     ──────────►│─────► {reg=2, bit=0}
  │ .regs[DS]     ──────────►│─────► {reg=0, bit=0}
  └──────────────────────────┘
```

---

### 3. دورة الحياة الكاملة — Lifecycle Diagram

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    COMPILE TIME                                  │
  │                                                                  │
  │  1. تعريف الـ pins:                                             │
  │     meson8b_cbus_pins[]  ──► MESON_PIN(GPIOX_0), ...           │
  │     meson8b_aobus_pins[] ──► MESON_PIN(GPIOAO_0), ...          │
  │                                                                  │
  │  2. تعريف الـ pin arrays (لكل function group):                  │
  │     sd_d0_a_pins[] = { GPIOX_0 }                               │
  │     uart_tx_a_pins[] = { GPIOX_4 }                              │
  │                                                                  │
  │  3. تعريف الـ groups:                                           │
  │     meson8b_cbus_groups[] ──► GPIO_GROUP() + GROUP()            │
  │                                                                  │
  │  4. تعريف الـ function name arrays:                             │
  │     uart_a_groups[] = { "uart_tx_a", "uart_rx_a", ... }        │
  │                                                                  │
  │  5. تعريف الـ functions:                                        │
  │     meson8b_cbus_functions[] ──► FUNCTION(uart_a), ...         │
  │                                                                  │
  │  6. تعريف الـ banks:                                            │
  │     meson8b_cbus_banks[] ──► BANK("X0..11", ...)               │
  │                                                                  │
  │  7. تجميع كل شيء في meson_pinctrl_data:                        │
  │     meson8b_cbus_pinctrl_data = { .pins, .groups, ... }        │
  └─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    BOOT TIME — Driver Registration              │
  │                                                                  │
  │  builtin_platform_driver(meson8b_pinctrl_driver)               │
  │      ──► platform_driver_register()                             │
  │          ──► يُسجَّل الـ driver في الـ platform bus            │
  └─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    PROBE TIME                                    │
  │                                                                  │
  │  meson_pinctrl_probe(pdev):                                     │
  │    1. تخصيص meson_pinctrl (devm_kzalloc)                       │
  │    2. قراءة .data من of_device_id ──► meson8b_cbus_pinctrl_data│
  │    3. مطابقة regmaps من DT (reg_mux, reg_gpio, ...)             │
  │    4. استدعاء parse_dt() إن وُجدت (للـ aobus فقط)              │
  │    5. تسجيل pinctrl_desc ──► devm_pinctrl_register()           │
  │    6. تسجيل gpio_chip ──► devm_gpiochip_add_data()             │
  └─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    RUNTIME USAGE                                 │
  │                                                                  │
  │  Device driver يطلب pin configuration من DT:                   │
  │    pinctrl_select_state() ──► pmx_set() ──► meson8_pmx_ops      │
  │                                                                  │
  │  GPIO usage:                                                     │
  │    gpio_direction_output() ──► gpio_chip.direction_output()     │
  │    ──► regmap_write(reg_gpio, BANK.regs[DIR].reg, ...)          │
  └─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    TEARDOWN                                      │
  │                                                                  │
  │  driver unload / system shutdown:                               │
  │    devm_* resources تُحرَّر تلقائياً                           │
  │    gpiochip_remove() + pinctrl_unregister()                     │
  └─────────────────────────────────────────────────────────────────┘
```

---

### 4. مخططات تدفق الـ Calls

#### 4.1 تفعيل Function (Pinmux)

```
device driver (e.g. UART driver) requests pinctrl state
  └─► pinctrl_select_state(p, state)           [pinctrl core]
        └─► pin_request() for each pin in group
              └─► pmx_ops->set_mux(pcdev, func_selector, group_selector)
                    └─► meson8_pmx_ops.set_mux()      [pinctrl-meson8-pmx.c]
                          └─► meson_pmx_set_one_func(pc, group)
                                └─► data = group->data  (meson8_pmx_data)
                                      ├─► if data->is_gpio:
                                      │     regmap_write(reg_mux, ...)  clear bit
                                      └─► else:
                                            regmap_update_bits(reg_mux,
                                              data->reg, BIT(data->bit), BIT(data->bit))
                                                └─► Hardware MUX register updated
```

#### 4.2 قراءة/كتابة GPIO

```
gpio_get_value(gpio_num)
  └─► gpio_chip->get(chip, offset)            [gpio_chip callback]
        └─► meson_gpio_get(chip, offset)       [pinctrl-meson.c]
              └─► meson_calc_reg_and_bit(pc, pin, MESON_REG_IN, &reg, &bit)
                    ├─► bank = مطابقة bank اللي فيه الـ pin
                    │     (first <= pin <= last)
                    └─► reg = bank->regs[IN].reg
                        bit = bank->regs[IN].bit + (pin - bank->first)
              └─► regmap_read(pc->reg_gpio, reg, &val)
                    └─► return (val >> bit) & 1
```

#### 4.3 إعداد Pull Resistor

```
pin_config_set(pin, PIN_CONFIG_BIAS_PULL_UP)
  └─► pinconf_ops->pin_config_set()           [pinctrl core]
        └─► meson_pinconf_set(pcdev, pin, configs, ...)  [pinctrl-meson.c]
              └─► meson_calc_reg_and_bit(pc, pin, MESON_REG_PULLEN, &reg, &bit)
              └─► regmap_update_bits(pc->reg_pullen, reg, BIT(bit), BIT(bit))
              └─► meson_calc_reg_and_bit(pc, pin, MESON_REG_PULL, &reg, &bit)
              └─► regmap_update_bits(pc->reg_pull, reg, BIT(bit), BIT(bit))
                    └─► Hardware PULL registers updated
```

#### 4.4 تسجيل الـ Driver عند الـ Boot

```
builtin_platform_driver(meson8b_pinctrl_driver)
  └─► platform_driver_register(&meson8b_pinctrl_driver)
        └─► kernel matches DT "amlogic,meson8b-cbus-pinctrl"
              └─► meson_pinctrl_probe(pdev)
                    ├─► of_match_device() ──► gets meson8b_cbus_pinctrl_data
                    ├─► devm_kzalloc(sizeof(meson_pinctrl))
                    ├─► meson_pinctrl_parse_dt()
                    │     ├─► of_parse_phandle() ──► gets regmap nodes
                    │     ├─► device_node_to_regmap() × 5 regmaps
                    │     └─► data->parse_dt(pc)  [aobus only]
                    ├─► pinctrl_desc_init():
                    │     .pins  = data->pins
                    │     .npins = data->num_pins
                    │     .ops   = &meson_pctrl_ops
                    │     .pmxops = data->pmx_ops  ──► meson8_pmx_ops
                    ├─► devm_pinctrl_register(dev, &pc->desc, pc)
                    │     └─► pc->pcdev ← pinctrl_dev handle
                    └─► devm_gpiochip_add_data(dev, &pc->chip, pc)
                          └─► GPIO subsystem registered
```

---

### 5. استراتيجية الـ Locking

الـ driver في `pinctrl-meson8b.c` نفسه **لا يحتوي على locks صريحة** — الحماية تتم على مستويات أعلى:

#### 5.1 الـ Pinctrl Core Lock

| الـ Lock | النوع | يحمي |
|----------|-------|-------|
| `pinctrl_dev->mutex` | `struct mutex` | طلبات الـ pin ownership وتغيير الـ state |

**الـ pinctrl core** بيمسك الـ mutex قبل أي استدعاء لـ `pmx_ops->set_mux()` أو `pinconf_ops->pin_config_set()`.

#### 5.2 الـ Regmap Lock

| الـ Lock | النوع | يحمي |
|----------|-------|-------|
| `regmap internal lock` | `spinlock` أو `mutex` حسب الـ config | عمليات القراءة/كتابة على الـ registers |

كل `regmap_update_bits()` و`regmap_read()` protected داخلياً من الـ regmap framework.

#### 5.3 الـ GPIO Chip Lock

| الـ Lock | النوع | يحمي |
|----------|-------|-------|
| `gpio_chip->bgpio_lock` | `spinlock` | عمليات الـ GPIO الـ banked |

#### 5.4 ترتيب الـ Locks (Lock Ordering)

```
pinctrl_dev->mutex        (أعلى مستوى — pinctrl core)
       │
       ▼
regmap internal lock      (لكل عملية read/write على register)
       │
       ▼
gpio bgpio_lock           (لعمليات الـ GPIO السريعة في interrupt context)
```

**القاعدة**: مفيش حالة بتمسك أكتر من lock واحد في نفس الوقت في كود الـ Meson driver نفسه — الـ regmap lock بيُمسك ويُحرَّر داخل كل استدعاء بشكل atomic، فمفيش خطر deadlock على مستوى الـ driver.

#### 5.5 الـ Registers المحمية لكل Domain

| الـ Domain | الـ Regmap | محمي بـ |
|------------|-----------|---------|
| cbus mux | `reg_mux` | regmap lock + pinctrl mutex |
| cbus GPIO dir/in/out | `reg_gpio` | regmap lock + gpio bgpio_lock |
| cbus pull enable | `reg_pullen` | regmap lock + pinctrl mutex |
| cbus pull dir | `reg_pull` | regmap lock + pinctrl mutex |
| cbus drive strength | `reg_ds` | regmap lock + pinctrl mutex |
| aobus (كل العمليات) | `reg_mux` (shared) | regmap lock + pinctrl mutex |

**ملاحظة مهمة**: الـ aobus له `reg_mux` منفصل عن الـ cbus لأن الـ Always-On domain له address space مختلف — الـ `meson8_aobus_parse_dt_extra()` بتُعد هذا الـ regmap بشكل منفصل عند الـ probe.
## Phase 4: شرح الـ Functions

---

### ملخص الـ APIs والـ Functions المهمة

#### جدول الـ Macros — Data Declaration

| Macro | المكان | الغرض |
|---|---|---|
| `MESON_PIN(x)` | `pinctrl-meson.h` | يعرّف `pinctrl_pin_desc` باسم الـ pin تلقائياً |
| `BANK(n,f,l,fi,li,...)` | `pinctrl-meson.h` | يعرّف `meson_bank` بكل الـ register offsets و IRQ range |
| `BANK_DS(...)` | `pinctrl-meson.h` | نفس `BANK` لكن مع drive-strength register إضافي |
| `GROUP(grp, r, b)` | `pinctrl-meson8-pmx.h` | يعرّف `meson_pmx_group` لـ function group بـ mux register و bit |
| `GPIO_GROUP(gpio)` | `pinctrl-meson8-pmx.h` | يعرّف group بـ `is_gpio=true`، بدون mux bit حقيقي |
| `FUNCTION(fn)` | `pinctrl-meson.h` | يعرّف `meson_pmx_func` بربط الـ function باسمه وقائمة groups |
| `PMX_DATA(r, b, g)` | `pinctrl-meson8-pmx.h` | يملأ `meson8_pmx_data` بـ register، bit، وflag GPIO |

#### جدول الـ Runtime Functions

| Function | الملف | الفئة |
|---|---|---|
| `meson_pinctrl_probe` | `pinctrl-meson.c` | Registration |
| `meson_pinctrl_parse_dt` | `pinctrl-meson.c` | Registration / DT Parsing |
| `meson8_aobus_parse_dt_extra` | `pinctrl-meson.c` | Registration / DT Parsing |
| `meson_gpiolib_register` | `pinctrl-meson.c` | Registration / GPIO |
| `meson_map_resource` | `pinctrl-meson.c` | Helper / regmap |
| `meson_get_bank` | `pinctrl-meson.c` | Helper |
| `meson_calc_reg_and_bit` | `pinctrl-meson.c` | Helper |
| `meson8_pmx_set_mux` | `pinctrl-meson8-pmx.c` | Pinmux Runtime |
| `meson8_pmx_disable_other_groups` | `pinctrl-meson8-pmx.c` | Pinmux Helper |
| `meson8_pmx_request_gpio` | `pinctrl-meson8-pmx.c` | Pinmux / GPIO |
| `meson_pmx_get_funcs_count` | `pinctrl-meson.c` | Pinmux Query |
| `meson_pmx_get_func_name` | `pinctrl-meson.c` | Pinmux Query |
| `meson_pmx_get_groups` | `pinctrl-meson.c` | Pinmux Query |
| `meson_get_groups_count` | `pinctrl-meson.c` | Pinctrl Query |
| `meson_get_group_name` | `pinctrl-meson.c` | Pinctrl Query |
| `meson_get_group_pins` | `pinctrl-meson.c` | Pinctrl Query |
| `meson_pinconf_set` | `pinctrl-meson.c` | Pin Config |
| `meson_pinconf_get` | `pinctrl-meson.c` | Pin Config |
| `meson_pinconf_group_set` | `pinctrl-meson.c` | Pin Config |
| `meson_pinconf_enable_bias` | `pinctrl-meson.c` | Pin Config Helper |
| `meson_pinconf_disable_bias` | `pinctrl-meson.c` | Pin Config Helper |
| `meson_pinconf_set_drive_strength` | `pinctrl-meson.c` | Pin Config Helper |
| `meson_pinconf_set_output` | `pinctrl-meson.c` | Pin Config Helper |
| `meson_pinconf_set_drive` | `pinctrl-meson.c` | Pin Config Helper |
| `meson_pinconf_set_output_drive` | `pinctrl-meson.c` | Pin Config Helper |
| `meson_pinconf_get_pull` | `pinctrl-meson.c` | Pin Config Helper |
| `meson_pinconf_get_output` | `pinctrl-meson.c` | Pin Config Helper |
| `meson_pinconf_get_drive` | `pinctrl-meson.c` | Pin Config Helper |
| `meson_pinconf_get_drive_strength` | `pinctrl-meson.c` | Pin Config Helper |
| `meson_gpio_get_direction` | `pinctrl-meson.c` | GPIO |
| `meson_gpio_direction_input` | `pinctrl-meson.c` | GPIO |
| `meson_gpio_direction_output` | `pinctrl-meson.c` | GPIO |
| `meson_gpio_get` | `pinctrl-meson.c` | GPIO |
| `meson_gpio_set` | `pinctrl-meson.c` | GPIO |
| `builtin_platform_driver` | `pinctrl-meson8b.c` | Registration macro |

---

### فئة 1: Data Declaration Macros

هذه الـ macros هي جوهر الملف `pinctrl-meson8b.c` — الملف كله تقريباً عبارة عن static data يتم تعريفها عبر هذه الـ macros. ما في الملف **مش كود تنفيذي** بالمعنى الحرفي، لكنه **إعلان بيانات compile-time** يصف topology الـ pins كاملة للـ Meson8b SoC.

---

#### `MESON_PIN(x)`

```c
#define MESON_PIN(x) PINCTRL_PIN(x, #x)
// expands to: { .number = x, .name = #x }
```

**ما يعمله:** يحول enum value زي `GPIOX_0` لـ `pinctrl_pin_desc` struct فيها الرقم والاسم كـ string. بيُستخدم لبناء arrays زي `meson8b_cbus_pins[]` و`meson8b_aobus_pins[]`.

**Parameters:**
- `x` — الـ pin enum value المعرّف في `meson8b-gpio.h`

**Key details:** الـ `#x` هو stringification، يعني الاسم بيتحدد compile-time من اسم الـ enum نفسه — مفيش overhead runtime.

---

#### `BANK(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib)`

```c
#define BANK(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib) \
    BANK_DS(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib, 0, 0)
```

**ما يعمله:** يعرّف `meson_bank` struct بالكامل — اسم الـ bank، أول pin وآخر pin، أول وآخر IRQ، وخمس `meson_reg_desc` entries لـ: PULLEN، PULL، DIR، OUT، IN. الـ `BANK_DS` نسخة موسعة بـ drive-strength register.

**Parameters (بالترتيب):**
| Parameter | المعنى |
|---|---|
| `n` | اسم الـ bank كـ string |
| `f` | أول pin enum في الـ bank |
| `l` | آخر pin enum في الـ bank |
| `fi` | أول IRQ hardware number (أو `-1` لو مفيش) |
| `li` | آخر IRQ hardware number |
| `per, peb` | register offset و bit للـ pull-enable |
| `pr, pb` | register offset و bit للـ pull direction |
| `dr, db` | register offset و bit للـ GPIO direction |
| `or, ob` | register offset و bit للـ GPIO output value |
| `ir, ib` | register offset و bit للـ GPIO input read |

**مثال حقيقي من الملف:**
```c
BANK("X0..11", GPIOX_0, GPIOX_11, 97, 108,
     4, 0,   /* pullen: reg=4, bit=0 */
     4, 0,   /* pull:   reg=4, bit=0 */
     0, 0,   /* dir:    reg=0, bit=0 */
     1, 0,   /* out:    reg=1, bit=0 */
     2, 0)   /* in:     reg=2, bit=0 */
```

**Key details:** الـ DIF bank ليها `irq_first = -1`، يعني مش متوثق إنها بتدعم GPIO interrupts. الـ register offsets هنا relative للـ regmap اللي بيتبناه `meson_map_resource`.

---

#### `GROUP(grp, r, b)` و`GPIO_GROUP(gpio)`

```c
#define GROUP(grp, r, b) {                          \
    .name = #grp,                                    \
    .pins = grp##_pins,                              \
    .num_pins = ARRAY_SIZE(grp##_pins),              \
    .data = (const struct meson8_pmx_data[]){        \
        PMX_DATA(r, b, false),                       \
    },                                               \
}

#define GPIO_GROUP(gpio) {                           \
    .name = #gpio,                                   \
    .pins = (const unsigned int[]){ gpio },          \
    .num_pins = 1,                                   \
    .data = (const struct meson8_pmx_data[]){        \
        PMX_DATA(0, 0, true),                        \
    },                                               \
}
```

**ما يعملوه:** `GROUP` يربط اسم الـ function group بـ array الـ pins المتعلقة بيه، وبالـ mux register و bit اللي بيُفعّل هذا الـ group. `GPIO_GROUP` يعرف group لـ pin واحد بـ `is_gpio=true` — ده معناه إن تفعيل الـ GPIO mode بيتم عن طريق تعطيل كل الـ mux bits، مش عن طريق set bit محدد.

**مثال:**
```c
GROUP(uart_tx_a, 4, 17)
// => pin GPIOX_4، mux register 4، bit 17
// لما يتفعّل UART_A TX، بيتعمل set للـ bit 17 في register 4*4=0x10
```

---

#### `FUNCTION(fn)`

```c
#define FUNCTION(fn) {                              \
    .name = #fn,                                    \
    .groups = fn##_groups,                          \
    .num_groups = ARRAY_SIZE(fn##_groups),          \
}
```

**ما يعمله:** يبني `meson_pmx_func` من `const char * const *` array. كل function زي `uart_a` بيكون ليه `uart_a_groups[]` array بأسماء الـ groups المنتمية ليه.

**Key details:** الـ `func_num=0` في كل من الـ cbus و aobus محجوز لـ GPIO mode (الـ `gpio_periphs` و`gpio_aobus`). ده critical لـ `meson8_pmx_set_mux` اللي بيتعامل مع `func_num=0` كـ special case.

---

### فئة 2: Registration — probe و setup

---

#### `meson_pinctrl_probe`

```c
int meson_pinctrl_probe(struct platform_device *pdev);
```

**ما يعمله:** الـ probe الوحيد للـ driver — يُخصّص `meson_pinctrl` struct، يقرأ الـ DT، يسجل الـ pinctrl device، ويسجل الـ GPIO chip. ده الـ entry point الذي يستدعيه platform bus بعد match الـ device.

**Parameters:**
- `pdev` — الـ `platform_device` اللي بيجي من الـ platform bus بعد match الـ `of_device_id`

**Return value:** `0` عند النجاح، قيمة سالبة عند الفشل (propagated من الـ sub-calls)

**Pseudocode flow:**
```
meson_pinctrl_probe(pdev):
    pc = devm_kzalloc(sizeof meson_pinctrl)
    pc->data = of_device_get_match_data(dev)   // يجيب الـ meson8b_cbus/aobus_pinctrl_data
    meson_pinctrl_parse_dt(pc)                 // يبني الـ regmaps
    fill pc->desc (pinctrl_desc)
    pc->pcdev = devm_pinctrl_register(pc->desc)
    meson_gpiolib_register(pc)                 // يسجل الـ gpio_chip
```

**Key details:**
- `of_device_get_match_data` بيرجع الـ `meson8b_cbus_pinctrl_data` أو `meson8b_aobus_pinctrl_data` بناءً على الـ `compatible` في الـ DT.
- `devm_pinctrl_register` بيربط الـ `pinctrl_desc` (اللي بيضم الـ `pctlops`، `pmxops`، `confops`) بالـ `pinctrl` subsystem.
- الـ `pmx_ops` بييجي من `pc->data->pmx_ops` وده للـ meson8b هو `meson8_pmx_ops`.

---

#### `meson_pinctrl_parse_dt`

```c
static int meson_pinctrl_parse_dt(struct meson_pinctrl *pc);
```

**ما يعمله:** يقرأ الـ DT node، يبحث عن child GPIO node، ثم يعمل `ioremap` وينشئ `regmap` لكل register range (mux، gpio، pull، pull-enable، ds). لو الـ SoC فيه `parse_dt` callback في الـ data struct، بيستدعيه في النهاية.

**Parameters:**
- `pc` — الـ pinctrl instance

**Return value:** `0` عند النجاح

**Key details:**
- `reg_mux` و`reg_gpio` إلزاميان — لو مش موجودين يرجع error.
- `reg_pull`، `reg_pullen`، `reg_ds` اختياريون — NULL لو مش موجودين.
- للـ meson8b aobus بيُستدعى `meson8_aobus_parse_dt_extra` كـ `parse_dt` callback.

---

#### `meson8_aobus_parse_dt_extra`

```c
int meson8_aobus_parse_dt_extra(struct meson_pinctrl *pc);
```

**ما يعمله:** خاص بالـ AO bus في Meson8/8b. الـ AO domain بيستخدم نفس الـ register لـ pull و pull-enable — يعني `reg_pullen = reg_pull`. بعد الـ `meson_pinctrl_parse_dt` العادي، الـ `reg_pullen` بيكون NULL لأنه مش موجود كـ DT resource منفصل، فالـ function ده بيصلح الموضوع.

**Parameters:**
- `pc` — الـ pinctrl instance

**Return value:** `0` عند النجاح، `-EINVAL` لو `reg_pull` نفسه NULL

**Key details:** مُعرَّف في `pinctrl-meson.c` ومُصدَّر بـ `EXPORT_SYMBOL_GPL`. بيُستدعى كـ `pc->data->parse_dt(pc)` من داخل `meson_pinctrl_parse_dt`.

---

#### `meson_gpiolib_register`

```c
static int meson_gpiolib_register(struct meson_pinctrl *pc);
```

**ما يعمله:** يُهيئ ويسجل الـ `gpio_chip` المدمج في الـ `meson_pinctrl` struct. بيعين كل الـ callbacks (direction_input، direction_output، get، set، get_direction) ثم يستدعي `gpiochip_add_data`.

**Parameters:**
- `pc` — الـ pinctrl instance

**Return value:** `0` عند النجاح، قيمة سالبة من `gpiochip_add_data` عند الفشل

**Key details:**
- `pc->chip.base = -1` يعني الـ kernel بيختار الـ base number ديناميكياً.
- `pc->chip.ngpio = pc->data->num_pins` — عدد الـ GPIO lines مساوي لعدد الـ pins.
- `pc->chip.can_sleep = true` لأن الوصول للـ registers عبر `regmap` قد يستخدم I2C أو SPI في SoCs أخرى (وإن كان هنا MMIO).
- `pc->chip.fwnode` بييجي من الـ GPIO child node في الـ DT.

---

#### `meson_map_resource`

```c
static struct regmap *meson_map_resource(struct meson_pinctrl *pc,
                                          struct device_node *node,
                                          char *name);
```

**ما يعمله:** يبحث عن register range باسمه في `reg-names` property في الـ DT، يعمل `ioremap` ليه، وينشئ `regmap` بـ 32-bit register و32-bit value width.

**Parameters:**
- `pc` — الـ pinctrl instance (للـ dev pointer)
- `node` — الـ DT node للـ GPIO child
- `name` — اسم الـ resource زي `"mux"`، `"gpio"`، `"pull"`، `"pull-enable"`، `"ds"`

**Return value:** pointer لـ `regmap` عند النجاح، `NULL` أو `ERR_PTR` عند الفشل

**Key details:**
- `meson_regmap_config.max_register` بيتحدد dynamically بناءً على حجم الـ resource، لتجنب out-of-bounds access.
- الـ config name بيتبنى بـ `devm_kasprintf` يعني managed memory.

---

### فئة 3: Pinmux Runtime Operations

هذه الـ functions تُنفّذ الـ `pinmux_ops` interface — قلب عملية تحديد وظيفة الـ pin.

---

#### `meson8_pmx_set_mux`

```c
static int meson8_pmx_set_mux(struct pinctrl_dev *pcdev,
                               unsigned func_num,
                               unsigned group_num);
```

**ما يعمله:** ينشّط function محدد على group محدد من الـ pins. أولاً يعطّل كل الـ groups الأخرى اللي بتستخدم نفس الـ pins (لتجنب conflict)، ثم يعمل set للـ mux bit الخاص بالـ group المطلوب (إلا لو الـ function هو GPIO = func_num 0).

**Parameters:**
- `pcdev` — الـ pinctrl device
- `func_num` — index الـ function في `meson8b_cbus_functions[]` أو `meson8b_aobus_functions[]`
- `group_num` — index الـ group في `meson8b_cbus_groups[]` أو `meson8b_aobus_groups[]`

**Return value:** `0` عند النجاح، قيمة سالبة من `regmap_update_bits` عند الفشل

**Pseudocode flow:**
```
meson8_pmx_set_mux(pcdev, func_num, group_num):
    func = data->funcs[func_num]
    group = data->groups[group_num]
    pmx_data = group->data

    for each pin in group->pins:
        meson8_pmx_disable_other_groups(pc, pin, group_num)
        // يعطّل كل group تاني يشارك هذا الـ pin

    if func_num != 0:   // مش GPIO mode
        regmap_update_bits(reg_mux, pmx_data->reg * 4,
                           BIT(pmx_data->bit), BIT(pmx_data->bit))
        // يفعّل الـ mux bit
```

**Key details:**
- `func_num=0` هو GPIO — مش بيعمل set لأي bit، بس بيعطّل الباقي.
- الـ `pmx_data->reg * 4` لأن الـ register offsets في الـ GROUP macro مضروبة في 4 bytes.
- الـ caller هو الـ pinctrl core عند تطبيق DT pinmux configuration.

---

#### `meson8_pmx_disable_other_groups`

```c
static void meson8_pmx_disable_other_groups(struct meson_pinctrl *pc,
                                             unsigned int pin,
                                             int sel_group);
```

**ما يعمله:** يمشي على كل الـ groups المعرّفة في الـ data struct، وأي group بيحتوي على `pin` المحدد (غير الـ `sel_group` المختار وغير GPIO groups) يعمل له clear للـ mux bit. ده بيضمن مبدأ الـ "one function per pin" — لا يمكن تفعيل function اتنين على نفس الـ pin.

**Parameters:**
- `pc` — الـ pinctrl instance
- `pin` — رقم الـ pin
- `sel_group` — index الـ group المحدد اللي محتاج إبقاءه (أو `-1` لتعطيل الكل)

**Return value:** void

**Key details:**
- لو `sel_group = -1` (زي في حالة GPIO request) بيعطّل الكل بدون استثناء.
- Skip الـ GPIO groups (`pmx_data->is_gpio == true`) لأنها مش بتحتاج clear.
- بيعمل `regmap_update_bits` بـ mask=`BIT(bit)` و value=`0`.

---

#### `meson8_pmx_request_gpio`

```c
static int meson8_pmx_request_gpio(struct pinctrl_dev *pcdev,
                                    struct pinctrl_gpio_range *range,
                                    unsigned offset);
```

**ما يعمله:** يُعيّن الـ pin لـ GPIO mode بتعطيل كل الـ function mux bits عليه. بيُستدعى من الـ pinctrl core لما يُطلب الـ GPIO عبر `gpio_request` أو `gpiod_get`.

**Parameters:**
- `pcdev` — الـ pinctrl device
- `range` — الـ GPIO range (غير مُستخدم هنا)
- `offset` — رقم الـ pin/GPIO

**Return value:** دايماً `0`

**Key details:** يستدعي `meson8_pmx_disable_other_groups(pc, offset, -1)` بـ `sel_group=-1` يعني يعطّل الكل.

---

### فئة 4: Pinmux Query Operations

---

#### `meson_pmx_get_funcs_count`

```c
int meson_pmx_get_funcs_count(struct pinctrl_dev *pcdev);
```

**ما يعمله:** يرجع عدد الـ functions المتاحة — الـ `num_funcs` من الـ pinctrl data. يُستدعى من الـ pinctrl core وأيضاً من الـ sysfs/debugfs.

---

#### `meson_pmx_get_func_name`

```c
const char *meson_pmx_get_func_name(struct pinctrl_dev *pcdev,
                                     unsigned selector);
```

**ما يعمله:** يرجع اسم الـ function بالـ index — من `funcs[selector].name`.

---

#### `meson_pmx_get_groups`

```c
int meson_pmx_get_groups(struct pinctrl_dev *pcdev,
                          unsigned selector,
                          const char * const **groups,
                          unsigned * const num_groups);
```

**ما يعمله:** يرجع قائمة أسماء الـ groups المنتمية لـ function معين. الـ pinctrl core بيستخدم ده لبناء الـ mapping.

**Parameters:**
- `selector` — function index
- `groups` — out pointer لـ array أسماء الـ groups
- `num_groups` — out pointer لعدد الـ groups

---

### فئة 5: Pin Configuration (pinconf) Operations

هذه الـ functions تتحكم في الـ electrical properties للـ pins: pull-up/down، drive strength، وdirection.

---

#### `meson_get_bank`

```c
static int meson_get_bank(struct meson_pinctrl *pc,
                           unsigned int pin,
                           const struct meson_bank **bank);
```

**ما يعمله:** يبحث عن الـ `meson_bank` اللي بيحتوي على الـ pin المحدد بمقارنة `pin >= bank.first && pin <= bank.last`. Linear search على الـ banks array.

**Parameters:**
- `pin` — رقم الـ pin العالمي
- `bank` — out pointer للـ bank المُوجَد

**Return value:** `0` عند النجاح، `-EINVAL` لو الـ pin مش موجود في أي bank

---

#### `meson_calc_reg_and_bit`

```c
static void meson_calc_reg_and_bit(const struct meson_bank *bank,
                                    unsigned int pin,
                                    enum meson_reg_type reg_type,
                                    unsigned int *reg,
                                    unsigned int *bit);
```

**ما يعمله:** يحسب الـ register offset والـ bit index الفعليَّين للـ pin لنوع معين من الـ registers. المعادلة تأخذ في الاعتبار أن بعض الـ register types (زي DS) تستخدم 2-bit per pin بدل 1-bit.

**Parameters:**
- `bank` — الـ bank الحاوي للـ pin
- `pin` — رقم الـ pin
- `reg_type` — نوع الـ register (`MESON_REG_PULLEN`، `MESON_REG_PULL`، `MESON_REG_DIR`، `MESON_REG_OUT`، `MESON_REG_IN`، `MESON_REG_DS`)
- `reg` — out: الـ byte offset في الـ regmap
- `bit` — out: رقم الـ bit داخل الـ register

**المعادلة الأساسية:**
```c
stride = meson_bit_strides[reg_type]; // 1 لمعظمهم، 2 للـ DS
bit_pos = (bank->regs[reg_type].bit + pin - bank->first) * stride;
*reg = (bank->regs[reg_type].reg + bit_pos / 32) * 4;  // byte offset
*bit = bit_pos & 0x1f;  // bit within 32-bit register
```

**Key details:** ده الـ function الأكثر استدعاءً في الـ driver — كل عملية GPIO أو pinconf بتمر بيه.

---

#### `meson_pinconf_set`

```c
static int meson_pinconf_set(struct pinctrl_dev *pcdev,
                              unsigned int pin,
                              unsigned long *configs,
                              unsigned num_configs);
```

**ما يعمله:** يطبّق array من الـ pin configurations على pin واحد. يتكرر على الـ configs ويستدعي الـ helper المناسب لكل `pin_config_param`.

**Parameters:**
- `pin` — رقم الـ pin
- `configs` — array من packed config values (parameter + argument)
- `num_configs` — عدد الـ configs

**الـ configs المدعومة:**
| Parameter | الـ Helper |
|---|---|
| `PIN_CONFIG_BIAS_DISABLE` | `meson_pinconf_disable_bias` |
| `PIN_CONFIG_BIAS_PULL_UP` | `meson_pinconf_enable_bias(true)` |
| `PIN_CONFIG_BIAS_PULL_DOWN` | `meson_pinconf_enable_bias(false)` |
| `PIN_CONFIG_DRIVE_STRENGTH_UA` | `meson_pinconf_set_drive_strength` |
| `PIN_CONFIG_OUTPUT_ENABLE` | `meson_pinconf_set_output` |
| `PIN_CONFIG_LEVEL` | `meson_pinconf_set_output_drive` |

---

#### `meson_pinconf_enable_bias`

```c
static int meson_pinconf_enable_bias(struct meson_pinctrl *pc,
                                      unsigned int pin,
                                      bool pull_up);
```

**ما يعمله:** يُفعّل الـ pull resistor على الـ pin. أولاً يضبط اتجاه الـ pull (up أو down) في `reg_pull`، ثم يُفعّل الـ pull resistor نفسه في `reg_pullen`.

**Parameters:**
- `pull_up` — `true` لـ pull-up، `false` لـ pull-down

**Key details:**
- Meson8b AO domain: `reg_pullen == reg_pull` (بسبب `meson8_aobus_parse_dt_extra`) — الـ bit نفسه بيتكتب مرتين بنفس القيمة.
- الترتيب مهم: direction قبل enable لتجنب glitch.

---

#### `meson_pinconf_disable_bias`

```c
static int meson_pinconf_disable_bias(struct meson_pinctrl *pc,
                                       unsigned int pin);
```

**ما يعمله:** يعمل clear لـ bit الـ pull-enable فقط في `reg_pullen`، يعني يوقف الـ pull resistor بغض النظر عن اتجاهه.

---

#### `meson_pinconf_set_drive_strength`

```c
static int meson_pinconf_set_drive_strength(struct meson_pinctrl *pc,
                                             unsigned int pin,
                                             u16 drive_strength_ua);
```

**ما يعمله:** يضبط قيمة الـ drive strength بالـ microamps. يحول القيمة لـ 2-bit enum (`MESON_PINCONF_DRV_500UA` إلى `MESON_PINCONF_DRV_4000UA`) ويكتبها في `reg_ds`.

**Parameters:**
- `drive_strength_ua` — الـ drive strength بالـ microamps

**الـ mapping:**
| Range (uA) | Enum value |
|---|---|
| <= 500 | `MESON_PINCONF_DRV_500UA` (0) |
| <= 2500 | `MESON_PINCONF_DRV_2500UA` (1) |
| <= 3000 | `MESON_PINCONF_DRV_3000UA` (2) |
| <= 4000 | `MESON_PINCONF_DRV_4000UA` (3) |

**Key details:**
- لو `pc->reg_ds == NULL` بيرجع `-ENOTSUPP` — الـ Meson8b مش عندها drive-strength support.
- كل pin بياخد 2 bits لذا `regmap_update_bits(reg, 0x3 << bit, val << bit)`.

---

#### `meson_pinconf_set_output` و`meson_pinconf_set_drive`

```c
static int meson_pinconf_set_output(struct meson_pinctrl *pc,
                                     unsigned int pin, bool out);
static int meson_pinconf_set_drive(struct meson_pinctrl *pc,
                                    unsigned int pin, bool high);
```

**ما يعملوا:** `set_output` يضبط الـ DIR bit (0=output, 1=input — مقلوب!). `set_drive` يضبط الـ OUT bit بالقيمة المطلوبة.

**Key details:** الـ DIR register في Meson منعكس: `0` = output، `1` = input. لذا `set_output(pc, pin, true)` بيكتب `0` في الـ DIR bit.

---

#### `meson_pinconf_set_output_drive`

```c
static int meson_pinconf_set_output_drive(struct meson_pinctrl *pc,
                                           unsigned int pin, bool high);
```

**ما يعمله:** يجمع عمليتين في واحدة: يضبط الـ pin كـ output ثم يضبط قيمة الـ output. Atomic من ناحية الـ logic لكن ليس atomic على مستوى الـ registers.

---

#### `meson_pinconf_get`

```c
static int meson_pinconf_get(struct pinctrl_dev *pcdev,
                              unsigned int pin,
                              unsigned long *config);
```

**ما يعمله:** يقرأ الـ configuration الحالية للـ pin ويرجعها packed. يستدعي الـ helpers المناسبة بناءً على الـ `param` المطلوب.

**Return value:** `0` عند النجاح، `-EINVAL` لو الـ config المطلوبة غير متطابقة مع الحالة الفعلية، `-ENOTSUPP` للـ params غير المدعومة

**Key details:** للـ bias، بيرجع `60000` كـ argument — ده standard placeholder value في pinconf framework.

---

#### `meson_pinconf_get_pull`

```c
static int meson_pinconf_get_pull(struct meson_pinctrl *pc, unsigned int pin);
```

**ما يعمله:** يقرأ حالة الـ pull resistor: أولاً يشوف لو `PULLEN` bit مفعّل، لو لا يرجع `PIN_CONFIG_BIAS_DISABLE`. لو مفعّل يقرأ `PULL` bit ويرجع `PULL_UP` أو `PULL_DOWN`.

---

### فئة 6: GPIO Chip Operations

---

#### `meson_gpio_get_direction`

```c
static int meson_gpio_get_direction(struct gpio_chip *chip, unsigned gpio);
```

**ما يعمله:** يقرأ الـ DIR bit للـ pin ويرجع `GPIO_LINE_DIRECTION_OUT` أو `GPIO_LINE_DIRECTION_IN`.

**Key details:** بيستدعي `meson_pinconf_get_output` اللي بترجع `0` أو `1`، ثم يحوّلها للـ enum المناسب.

---

#### `meson_gpio_direction_input`

```c
static int meson_gpio_direction_input(struct gpio_chip *chip, unsigned gpio);
```

**ما يعمله:** يضبط الـ pin كـ input بـ `meson_pinconf_set_output(pc, gpio, false)`.

---

#### `meson_gpio_direction_output`

```c
static int meson_gpio_direction_output(struct gpio_chip *chip,
                                        unsigned gpio, int value);
```

**ما يعمله:** يضبط الـ pin كـ output ويكتب القيمة في نفس الوقت عبر `meson_pinconf_set_output_drive`.

---

#### `meson_gpio_get`

```c
static int meson_gpio_get(struct gpio_chip *chip, unsigned gpio);
```

**ما يعمله:** يقرأ قيمة الـ IN register للـ pin ويرجع `0` أو `1`. يعمل `meson_get_bank`، `meson_calc_reg_and_bit` بـ `MESON_REG_IN`، ثم `regmap_read`.

---

#### `meson_gpio_set`

```c
static int meson_gpio_set(struct gpio_chip *chip,
                           unsigned int gpio, int value);
```

**ما يعمله:** يكتب قيمة على الـ OUT register للـ pin لو كان output. يستدعي `meson_pinconf_set_drive`.

---

### فئة 7: Static Data Tables (Meson8b-specific)

ده مش "functions" بالمعنى التقليدي، لكنه الـ **core contribution** للملف `pinctrl-meson8b.c`. كل البيانات الـ static دي هي الـ board-specific knowledge للـ Meson8b SoC.

#### تسلسل البيانات وعلاقتها

```
pinctrl_pin_desc[]          ← MESON_PIN() macro
       ↓
meson_pmx_group[]           ← GPIO_GROUP() + GROUP() macros
       ↓
const char * const *_groups[] ← قوائم أسماء الـ groups لكل function
       ↓
meson_pmx_func[]            ← FUNCTION() macro
       ↓
meson_bank[]                ← BANK() macro
       ↓
meson_pinctrl_data          ← يجمع كل شيء
       ↓
of_device_id[]              ← يربط الـ data بالـ compatible string
       ↓
platform_driver             ← builtin_platform_driver()
```

#### الـ pin banks في Meson8b

| Bank | الـ pins | عدد الـ pins | ملاحظة |
|---|---|---|---|
| `X0..11` | GPIOX_0 → GPIOX_11 | 12 | General purpose |
| `X16..21` | GPIOX_16 → GPIOX_21 | 6 | General purpose |
| `Y0..1` | GPIOY_0 → GPIOY_1 | 2 | |
| `Y3` | GPIOY_3 | 1 | |
| `Y6..14` | GPIOY_6 → GPIOY_14 | 9 | |
| `DV9` | GPIODV_9 | 1 | |
| `DV24..29` | GPIODV_24 → GPIODV_29 | 6 | |
| `H` | GPIOH_0 → GPIOH_9 | 10 | |
| `CARD` | CARD_0 → CARD_6 | 7 | |
| `BOOT` | BOOT_0 → BOOT_18 | 19 | |
| `DIF` | DIF_0_P → DIF_4_N | 10 | بدون IRQ support |
| `AO` | GPIOAO_0 → GPIO_TEST_N | 16 | Always-On domain |

#### الفرق بين cbus و aobus

| | **cbus** | **aobus** |
|---|---|---|
| Power domain | Regular power domain | Always-On (لا ينقطع حتى في sleep) |
| الـ regmaps | reg_mux، reg_gpio، reg_pull، reg_pullen | reg_mux، reg_gpio، reg_pull (= reg_pullen) |
| `parse_dt` callback | `NULL` | `meson8_aobus_parse_dt_extra` |
| عدد الـ functions | 31 function | 11 function |
| الـ compatible | `amlogic,meson8b-cbus-pinctrl` | `amlogic,meson8b-aobus-pinctrl` |

#### `builtin_platform_driver`

```c
builtin_platform_driver(meson8b_pinctrl_driver);
```

**ما يعمله:** macro يُسجّل الـ `platform_driver` كـ built-in (مش module)، يعني بيُستدعى `platform_driver_register` مبكراً في الـ boot sequence — قبل الـ `module_init`. ده ضروري لأن الـ pinctrl يجب أن يكون جاهز قبل أي device يحتاج GPIO أو pin configuration.
## Phase 5: دليل الـ Debugging الشامل

الـ driver ده هو `pinctrl-meson8b` — بيتحكم في الـ pin multiplexing والـ GPIO لشريحة Amlogic Meson8b عبر busين منفصلين: **cbus** (الـ peripheral pins) و**aobus** (الـ always-on pins). الـ debugging بيتطلب فهم إن كل bank ليها registers منفصلة للـ direction، pull، output، و input.

---

### Software Level

#### 1. Debugfs Entries

الـ pinctrl subsystem بيعرض معلومات تفصيلية تحت `/sys/kernel/debug/pinctrl/`:

```bash
# اعرف كل الـ pinctrl devices المتاحة
ls /sys/kernel/debug/pinctrl/

# هتلاقي مثلاً:
# meson8b-pinctrl.cbus-banks
# meson8b-pinctrl.aobus-banks

# اعرف state الـ pins كلها (function + group المختار)
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.cbus-banks/pinmux-pins

# اعرف الـ groups المعرّفة
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.cbus-banks/pinmux-functions

# اعرف state كل pin (direction, pull, value)
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.cbus-banks/pinconf-pins

# لو عايز aobus (GPIOAO_*)
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.aobus-banks/pinmux-pins
```

**تفسير الـ output:**

```
pin 0 (GPIOX_0): meson8b-pinctrl.cbus-banks group sd_d0_a function sd_a
pin 4 (GPIOX_4): UNCLAIMED
```

- `UNCLAIMED` → الـ pin مش محجوز لأي function — ممكن يكون GPIO حر أو مش متعرّف في الـ DT.
- لو الـ pin محجوز بـ function خاطئ → فيه conflict في الـ DT أو الـ driver.

#### 2. Sysfs Entries

```bash
# اعرف الـ GPIO chip المقابل لـ cbus
ls /sys/class/gpio/

# export pin معين (رقم global)
echo 410 > /sys/class/gpio/export   # مثال: GPIOX_0 لو base=410

# اقرأ direction والقيمة
cat /sys/class/gpio/gpio410/direction
cat /sys/class/gpio/gpio410/value

# اعرف الـ gpiochip المرتبط بالـ driver
cat /sys/class/gpio/gpiochip*/label | grep meson
cat /sys/class/gpio/gpiochip410/ngpio
cat /sys/class/gpio/gpiochip410/base

# تحقق من الـ pinctrl device
ls /sys/bus/platform/drivers/meson8b-pinctrl/
```

**ملاحظة مهمة:** الـ Meson8b عنده بنكين منفصلين (cbus + aobus) كل واحد فيهم `gpio_chip` مستقل — يعني فيه gpiochip لكل bus.

#### 3. Ftrace — Tracepoints والـ Events

```bash
# فعّل الـ tracing للـ pinctrl subsystem
cd /sys/kernel/debug/tracing

# اعرف الـ events المتاحة
grep -r pinctrl available_events

# فعّل كل events الـ gpio
echo 1 > events/gpio/enable

# أو فعّل events محددة
echo 1 > events/pinctrl/pin_config_set/enable 2>/dev/null
echo 1 > events/pinctrl/pinmux_enable/enable 2>/dev/null

# ابدأ الـ trace
echo 1 > tracing_on
# ... شغّل الـ operation اللي بتفحصها ...
cat trace | grep -E "gpio|pinctrl|meson"
echo 0 > tracing_on

# تتبع function calls لـ meson_pinctrl_probe
echo meson_pinctrl_probe > set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on
cat trace
```

#### 4. Printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لكل الـ pinctrl subsystem
echo 'module pinctrl_meson8b +p' > /sys/kernel/debug/dynamic_debug/control

# أو لملف محدد
echo 'file drivers/pinctrl/meson/pinctrl-meson8b.c +p' \
     > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ pinctrl-meson.c (الـ core)
echo 'file drivers/pinctrl/meson/pinctrl-meson.c +pflmt' \
     > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ messages في dmesg
dmesg | grep -i "meson\|pinctrl\|gpio"

# فعّل verbose kernel log أثناء الـ boot
# أضف في kernel cmdline:
# dyndbg="module pinctrl_meson8b +p" loglevel=8
```

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_PINCTRL_MESON` | الـ core driver — لازم يكون enabled |
| `CONFIG_PINCTRL_MESON8B` | الـ driver نفسه للـ Meson8b |
| `CONFIG_DEBUG_GPIO` | بيضيف verbose logging لكل GPIO operations |
| `CONFIG_GPIO_SYSFS` | بيعرض GPIO عبر sysfs للـ testing اليدوي |
| `CONFIG_PINCTRL_MESON_PMX` | الـ pinmux helper |
| `CONFIG_REGMAP_DEBUGFS` | بيعرض الـ regmap registers في debugfs |
| `CONFIG_DEBUG_FS` | لازم يكون enabled عشان كل debugfs يشتغل |
| `CONFIG_DYNAMIC_DEBUG` | لتفعيل dynamic debug messages |
| `CONFIG_PROVE_LOCKING` | يكشف locking bugs في الـ pinctrl callbacks |
| `CONFIG_DEBUG_SPINLOCK` | يكشف spinlock misuse |

#### 6. Devlink وأدوات الـ Subsystem

الـ pinctrl مش بيستخدم devlink مباشرةً، لكن فيه أدوات userspace مفيدة:

```bash
# pinctrl-utils (لو مثبتة)
pinctrl-list          # يعرض كل الـ pincontrollers
pinctrl-info          # معلومات تفصيلية

# regmap debugfs — اقرأ الـ mux registers مباشرة
# الـ regmap الخاص بـ cbus يظهر تحت:
ls /sys/kernel/debug/regmap/

# مثال: اقرأ register رقم 2 (nand/sdcard mux) بالـ hex
cat /sys/kernel/debug/regmap/*/registers | grep "^00000008"
# الـ offset 8 = reg*4 في بعض الـ platforms

# استخدم gpio-utils لو متاحة
gpiodetect           # يعرض كل الـ gpiochips
gpioinfo gpiochip0   # يعرض كل الـ lines والـ state
gpioget gpiochip0 0  # يقرأ قيمة pin
gpioset gpiochip0 0=1 # يضبط قيمة pin
```

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `meson8b-pinctrl: can't get device resource` | مش لاقي الـ regmap resource في الـ DT | تأكد من `reg` properties في الـ DT node |
| `pinctrl request already granted` | conflict — اتنين drivers بيطلبوا نفس الـ pin | راجع الـ DT، شيل أي تكرار في `pinctrl-0` |
| `pin X already requested` | نفس الـ pin اتطلب مرتين | راجع الـ device nodes في الـ DT |
| `could not get pinctrl` | الـ probe فشل يربط الـ pinctrl handle | تأكد إن `pinctrl-0` و `pinctrl-names` موجودين في الـ DT |
| `could not set pins` | فشل في تفعيل الـ pinmux state | تأكد إن الـ group name في الـ DT صح |
| `Error applying setting, reverse things back` | فشل في apply الـ pin config | تحقق من الـ pull/drive-strength values |
| `GPIO_X: no irq` | الـ pin مش بيدعم interrupts (مثل DIF bank) | استخدم pin من bank تاني أو polling |
| `regmap update failed` | فشل في write على الـ MUX register | تحقق من الـ clock للـ cbus |
| `meson8b-pinctrl probe failed` | فشل كلي في الـ probe | اقرأ كل الـ dmesg بعد الرسالة |

#### 8. Strategic Points لـ dump_stack() و WARN_ON()

الأماكن الاستراتيجية في `pinctrl-meson.c` (الـ core) للـ debugging:

```c
/* في meson_pmx_set() — بعد ما يحسب الـ reg والـ bit */
static int meson_pmx_set(struct pinctrl_dev *pcdev,
                         unsigned func_selector,
                         unsigned group_selector)
{
    /* ضع WARN_ON هنا للتحقق من صحة الـ selector */
    WARN_ON(group_selector >= pc->data->num_groups);

    /* بعد الـ regmap_update_bits — تحقق من الـ return value */
    ret = regmap_update_bits(pc->reg_mux, ...);
    if (ret)
        dump_stack(); /* يساعد في تحديد من طلب الـ pinmux */
}

/* في meson_gpio_direction_input/output */
WARN_ON(!meson_calc_reg_and_bit(bank, pin, MESON_REG_DIR, &reg, &bit));

/* في meson_pinctrl_probe — بعد regmap_init */
WARN_ON(IS_ERR(pc->reg_mux));
```

---

### Hardware Level

#### 1. التحقق إن Hardware State يطابق Kernel State

الـ Meson8b بيتحكم في الـ pins عبر registers في الـ CBUS (عنوان base `0xC1100000`) وAOBUS (`0xC8100000`):

```bash
# تحقق إن الـ kernel شايف الـ register values الصح
# افتح debugfs regmap واقرأ الـ mux registers

# مثال: reg 2 في cbus (NAND/SD mux bits)
# الـ GROUP macro: GROUP(nand_io, 2, 26) → reg=2, bit=26
# يعني المفروض bit 26 في reg 2 تكون 1 لو nand مفعّل

cat /sys/kernel/debug/regmap/*/registers | awk '$1=="00000008" {print $2}'
# offset = reg_index * 4 = 2 * 4 = 8 → 0x00000008

# قارن الـ output بالـ expected value من الـ DT pinctrl state
```

#### 2. Register Dump Techniques

الـ CBUS MUX registers على الـ Meson8b:

```
CBUS base: 0xC1100000
MUX regs: offset 0x2000 + reg*4 (approx — verify datasheet)

AOBUS base: 0xC8100000
AO MUX reg: offset 0x0000
```

```bash
# استخدم devmem2 لقراءة registers مباشرة
# (تأكد إن /dev/mem مفتوح أو CONFIG_STRICT_DEVMEM=n)

# اقرأ AO MUX register (reg 0, بيتحكم في GPIOAO)
devmem2 0xC8100000 w    # word read

# اقرأ cbus mux register 1 (HDMI, SPDIF, etc.)
devmem2 0xC1108000 w    # مثال — verify exact offset

# أو عبر /dev/mem مباشرة
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 0x1000, offset=0xC8100000)
    val = struct.unpack('<I', m.read(4))[0]
    print(f'AO MUX reg: 0x{val:08X}')
    m.close()
"

# io command (من package ioport أو devmem)
io -4 -r 0xC8100000
```

**تفسير الـ bits:** كل bit في الـ MUX register يتحكم في تفعيل function معين. الـ `GROUP(uart_tx_ao_a, 0, 12)` معناها: reg=0، bit=12 في الـ AO MUX.

```
AO MUX Register (0xC8100000):
Bit 31: ir_blaster
Bit 30: i2s_am_clk_out
Bit 29: i2s_ao_clk_out
Bit 28: i2s_lr_clk_out
Bit 27: i2s_out_01
Bit 26: uart_tx_ao_b0
Bit 25: uart_rx_ao_b0
Bit 24: uart_tx_ao_b1
Bit 23: uart_rx_ao_b1
Bit 22: pwm_c2
Bit 21: ir_remote_out
Bit 18: clk_32k_in_out
Bit 17: hdmi_cec_1
Bit 16: spdif_out_1
Bit 15: i2s_ao_clk_in
Bit 14: i2s_lr_clk_in
Bit 13: i2s_in_ch01
Bit 12: uart_tx_ao_a
Bit 11: uart_rx_ao_a
Bit 10: uart_cts_ao_a
Bit  9: uart_rts_ao_a
Bit  8: uart_cts_ao_b
Bit  7: uart_rts_ao_b
Bit  6: i2c_mst_sck_ao
Bit  5: i2c_mst_sda_ao
Bit  2: i2c_sck_ao
Bit  1: i2c_sda_ao
Bit  0: remote_input
```

#### 3. Logic Analyzer / Oscilloscope Tips

```
للـ UART debugging (مثلاً uart_tx_ao_a على GPIOAO_0):
- بروتوكول: UART، Baud rate: حسب الـ config
- نقطة القياس: TP على GPIOAO_0
- المتوقع: idle HIGH، start bit LOW

للـ I2C debugging (مثلاً i2c_mst_sck_ao على GPIOAO_4):
- بروتوكول: I2C
- قيس SDA (GPIOAO_5) و SCL (GPIOAO_4) مع Trigger على START condition
- تأكد من وجود pull-up مناسب (1.8V أو 3.3V حسب الـ domain)

للـ SD/SDXC debugging:
- بروتوكول: SD
- قيس CLK، CMD، D0..D3
- لو فيه glitches → مشكلة في الـ pull-up أو الـ drive strength

للـ Ethernet (DIF bank):
- الـ DIF pins هي differential pairs (P/N)
- استخدم differential probe
- تأكد إن common mode voltage في النطاق الصح
```

#### 4. Hardware Issues → Kernel Log Patterns

| المشكلة الهاردوير | Pattern في الـ Kernel Log |
|-------------------|--------------------------|
| الـ pull-up مش موجود على I2C | `i2c i2c-X: timeout waiting for bus ready` |
| الـ SD card مش connected | `mmc0: error -110 whilst initialising SD card` |
| الـ UART TX stuck LOW | no output، أو `serial8250: too much work for irq` |
| الـ Ethernet DIF pins مش متوصلة | `eth0: Link is Down` بعد كل محاولات |
| الـ HDMI HPD مش اشتغل | `hdmi: no display detected` |
| conflict في الـ power domain | `regulator: failed to enable` قبل الـ pinctrl error |
| NAND WP pin مش configured | `nand: device is write protected` |
| GPIOAO_X تاخد interrupt وهي مش بتدعمه (GPIO_BSD_EN, GPIO_TEST_N) | `irq: no irq domain found` |

#### 5. Device Tree Debugging

الـ driver بيدعم compatible strings اتنين:
- `amlogic,meson8b-cbus-pinctrl`
- `amlogic,meson8b-aobus-pinctrl`

```bash
# تحقق إن الـ DT nodes موجودة وصح
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "meson8b.*pinctrl"

# أو
cat /proc/device-tree/soc/bus@c1100000/pinctrl@2000/compatible
# المتوقع: amlogic,meson8b-cbus-pinctrl

# تحقق من الـ reg property
hexdump -C /proc/device-tree/soc/bus@c1100000/pinctrl@2000/reg

# تحقق إن الـ pinctrl-0 في device معين صح
# مثال: UART device
cat /proc/device-tree/soc/serial@c11084c0/pinctrl-names
# المتوقع: "default"

ls /proc/device-tree/soc/serial@c11084c0/
# لازم يكون فيه: pinctrl-0 → phandle بيشاور على pinctrl node

# تأكد من الـ mux reg المستخدم
# GROUP(uart_tx_ao_a, 0, 12) → reg=0, bit=12 في aobus
# لازم يتطابق مع الـ datasheet
```

**أخطاء DT شائعة في الـ Meson8b:**

```
# خطأ: استخدام group من cbus في aobus device
&uart_AO {
    pinctrl-0 = <&uart_ao_a_pins>;  /* صح */
    /* لو كتبت uart_a_pins هنا → هيفشل */
};

# خطأ: نسيان إن GPIO_BSD_EN وGPIO_TEST_N
# مش بيدعموا GPIO interrupts (مذكور في الـ code comment)
```

---

### Practical Commands

#### سيناريو 1: الـ UART على الـ AO Bus مش شغال

```bash
# خطوة 1: تحقق إن الـ pinctrl اشتغل
dmesg | grep -E "meson8b|pinctrl|uart_AO"

# خطوة 2: شوف الـ pin state
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.aobus-banks/pinmux-pins | \
    grep -E "GPIOAO_0|GPIOAO_1"

# Output المتوقع لو اشتغل:
# pin 0 (GPIOAO_0): ... group uart_tx_ao_a function uart_ao

# خطوة 3: لو مش محجوز، اقرأ الـ AO MUX register
devmem2 0xC8100000 w
# bit 12 لازم تكون 1 لـ uart_tx_ao_a

# خطوة 4: تحقق من الـ direction
gpioinfo | grep -A 2 "GPIOAO_0"
```

#### سيناريو 2: SD Card على bank C (BOOT pins) مش بيشتغل

```bash
# تحقق إن الـ pins محجوزة للـ SD
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.cbus-banks/pinmux-pins | \
    grep -E "BOOT_[0-3]|BOOT_8|BOOT_10"

# Output المتوقع:
# pin XX (BOOT_0): ... group sd_d0_c function sd_c
# pin XX (BOOT_8): ... group sd_cmd_c function sd_c

# تحقق من pull-up (SD بيحتاج pull-up على D0..D3 وCMD)
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.cbus-banks/pinconf-pins | \
    grep BOOT

# تحقق من conflict مع NAND
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.cbus-banks/pinmux-pins | \
    grep "BOOT" | grep -v "sd_"
# لو ظهر فيه nand → conflict!
```

#### سيناريو 3: فحص كامل لـ Ethernet (DIF bank)

```bash
# تحقق من الـ DIF pins
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.cbus-banks/pinmux-pins | \
    grep "DIF"

# Output المتوقع:
# pin XX (DIF_0_P): ... group eth_rxd1 function ethernet
# pin XX (DIF_0_N): ... group eth_rxd0 function ethernet

# لاحظ إن DIF bank مش بتدعم interrupts (irq_first=-1, irq_last=-1)
# لو حاولت تستخدمها كـ GPIO interrupt → هيفشل

# اقرأ الـ ethernet mux registers
# GROUP(eth_rxd1, 6, 0) → reg=6, bit=0 في cbus
# offset = 6 * 4 = 24 = 0x18
devmem2 $((0xC1108000 + 0x18)) w   # مثال — verify exact base
```

#### سيناريو 4: فحص شامل بعد الـ Boot

```bash
#!/bin/bash
# pinctrl-meson8b-debug.sh

echo "=== Meson8b Pinctrl Debug Report ==="
echo ""

echo "--- Driver Probe Status ---"
dmesg | grep -i "meson8b\|pinctrl" | tail -20

echo ""
echo "--- CBUS Pinmux State ---"
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.cbus-banks/pinmux-pins 2>/dev/null \
    | grep -v "UNCLAIMED" | head -30

echo ""
echo "--- AOBUS Pinmux State ---"
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.aobus-banks/pinmux-pins 2>/dev/null \
    | grep -v "UNCLAIMED"

echo ""
echo "--- GPIO Chips ---"
gpiodetect 2>/dev/null || ls /sys/class/gpio/gpiochip*/label

echo ""
echo "--- Regmap Registers (AOBUS) ---"
cat /sys/kernel/debug/regmap/*/registers 2>/dev/null | head -20

echo ""
echo "--- Conflicts Check ---"
# لو نفس الـ pin ظهر اكتر من مرة → conflict
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.cbus-banks/pinmux-pins 2>/dev/null \
    | grep -v "UNCLAIMED" | awk '{print $1}' | sort | uniq -d
```

#### سيناريو 5: تفعيل Verbose Logging أثناء الـ Runtime

```bash
# فعّل dynamic debug لكل الـ meson pinctrl files
for f in pinctrl-meson8b pinctrl-meson pinctrl-meson8-pmx; do
    echo "module $f +pflmt" > /sys/kernel/debug/dynamic_debug/control 2>/dev/null
    echo "file drivers/pinctrl/meson/$f.c +pflmt" \
         > /sys/kernel/debug/dynamic_debug/control 2>/dev/null
done

# شوف الـ messages
dmesg -w | grep -E "meson|pinctrl|gpio" &
DMESG_PID=$!

# اعمل operation معينة (مثلاً: probe device)
echo "meson8b-pinctrl" > /sys/bus/platform/drivers/meson8b-pinctrl/bind 2>/dev/null

# وقف الـ monitoring
kill $DMESG_PID

# وقف الـ debug logging
echo "module pinctrl_meson8b -p" > /sys/kernel/debug/dynamic_debug/control
```

#### سيناريو 6: التحقق من الـ Pinconf (Pull/Drive Strength)

```bash
# اقرأ الـ pinconf لـ bank معينة
cat /sys/kernel/debug/pinctrl/meson8b-pinctrl.cbus-banks/pinconf-groups

# الـ BANK macro في الكود يحدد:
# BANK("X0..11", GPIOX_0, GPIOX_11, 97, 108,
#       pullen_reg=4, pullen_bit=0,    ← PULLEN register
#       pull_reg=4,   pull_bit=0,      ← PULL register
#       dir_reg=0,    dir_bit=0,       ← DIR register
#       out_reg=1,    out_bit=0,       ← OUT register
#       in_reg=2,     in_bit=0)        ← IN register

# تحقق من الـ pull للـ GPIOX_0:
# pullen_reg=4, pullen_bit=0 → bit 0 في reg 4 يفعّل الـ pull
# pull_reg=4,   pull_bit=0   → bit 0 في reg 4 يحدد up/down

# اقرأ الـ registers مباشرة عبر regmap debugfs
cat /sys/kernel/debug/regmap/*/registers | \
    awk 'NR>1 && $1~/^0000001[0-9]/ {printf "reg 0x%s = 0x%s\n", $1, $2}'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Android TV Box بيحتاج Ethernet بس ما بيشتغل

#### العنوان
**الـ Ethernet مش شغال على TV box مبني على Amlogic Meson8b**

#### السياق
شركة بتصنع media player رخيص مبني على Amlogic S905 (Meson8b-compatible). البورد اتصمم بـ integrated Ethernet controller. في يوم الـ bring-up الأول، الـ kernel بيبوت بدون مشكلة، لكن الـ `eth0` مش ظاهر خالص.

#### المشكلة
الـ Ethernet driver بيتحمل، لكن الـ MII/RMII signals مش بتتحرك. الـ `ip link show` بيرجع `eth0` كـ `NO-CARRIER`.

```bash
$ dmesg | grep eth
[    2.345] stmmac-pci: probe failed
$ cat /sys/kernel/debug/pinctrl/pinctrl-handles
# no ethernet function applied
```

الـ `eth_rxd0`, `eth_rxd1`, `eth_rx_dv`, `eth_tx_en` مش متعرّضين للـ MAC.

#### التحليل
في الملف `pinctrl-meson8b.c`، الـ Ethernet pins موزعة على **بنكين مختلفين**:

```c
/* bank DIF — الـ differential pins للـ PHY */
static const unsigned int eth_rxd1_pins[]    = { DIF_0_P };
static const unsigned int eth_rxd0_pins[]    = { DIF_0_N };
static const unsigned int eth_rx_dv_pins[]   = { DIF_1_P };
static const unsigned int eth_rx_clk_pins[]  = { DIF_1_N };
static const unsigned int eth_txd0_1_pins[]  = { DIF_2_P };
static const unsigned int eth_txd1_1_pins[]  = { DIF_2_N };
static const unsigned int eth_tx_en_pins[]   = { DIF_3_P };
static const unsigned int eth_ref_clk_pins[] = { DIF_3_N };
static const unsigned int eth_mdc_pins[]     = { DIF_4_P };
static const unsigned int eth_mdio_en_pins[] = { DIF_4_N };

/* bank H — بعض TX pins بديلة */
static const unsigned int eth_txd1_0_pins[]  = { GPIOH_5 };
static const unsigned int eth_txd0_0_pins[]  = { GPIOH_6 };
static const unsigned int eth_txd3_pins[]    = { GPIOH_7 };
static const unsigned int eth_txd2_pins[]    = { GPIOH_8 };
static const unsigned int eth_tx_clk_pins[]  = { GPIOH_9 };
```

الـ `ethernet_groups` بتجمعهم كلهم في function واحدة:

```c
static const char * const ethernet_groups[] = {
    "eth_tx_clk", "eth_tx_en", "eth_txd1_0", "eth_txd1_1",
    "eth_txd0_0", "eth_txd0_1", "eth_rx_clk", "eth_rx_dv",
    "eth_rxd1", "eth_rxd0", "eth_mdio_en", "eth_mdc", "eth_ref_clk",
    "eth_txd2", "eth_txd3", "eth_rxd3", "eth_rxd2",
    "eth_rxd3_h", "eth_rxd2_h"
};
```

الـ DIF bank بيتعرّف في `meson8b_cbus_banks`:
```c
BANK("DIF", DIF_0_P, DIF_4_N, -1, -1, 5, 8, 5, 8, 12, 12, 13, 12, 14, 12),
```

لاحظ `irq_first = -1` و `irq_last = -1` — ده معناه إن الـ DIF bank **مش موثق رسمياً** في datasheet:

```c
/*
 * The following bank is not mentionned in the public datasheet
 * There is no information whether it can be used with the gpio
 * interrupt controller
 */
```

المهندس نسي يضيف الـ `pinctrl-0` في الـ Device Tree node الخاص بالـ Ethernet controller.

#### الحل
في الـ DTS:

```dts
&ethmac {
    status = "okay";
    pinctrl-0 = <&eth_rmii_pins>;
    pinctrl-names = "default";
};

eth_rmii_pins: eth-rmii {
    mux {
        groups = "eth_rxd0", "eth_rxd1", "eth_rx_dv",
                 "eth_rx_clk", "eth_txd0_1", "eth_txd1_1",
                 "eth_tx_en", "eth_ref_clk",
                 "eth_mdc", "eth_mdio_en";
        function = "ethernet";
    };
};
```

```bash
# تحقق بعد التغيير
$ cat /sys/kernel/debug/pinctrl/pinctrl-handles | grep eth
state default: pin DIF_0_P [ethernet]
state default: pin DIF_0_N [ethernet]
...
$ ip link show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
```

#### الدرس المستفاد
الـ DIF bank في Meson8b هو بنك خاص للـ differential signals، مش موثق في الـ public datasheet، ومش بيدعم GPIO interrupts (`irq = -1`). لازم تعرف إن الـ Ethernet على ده الـ SoC بيستخدم DIF pins مش GPIOH pins عادية.

---

### السيناريو 2: Industrial Gateway — الـ UART Console بيطلع garbage

#### العنوان
**الـ serial console بيطلع garbage على industrial gateway مبني على Meson8b**

#### السياق
شركة بتطور industrial gateway لـ SCADA systems. البورد بيستخدم Amlogic Meson8b. الـ bootloader شغال ومبيعملش مشكلة، لكن بعد ما الـ kernel بيبدأ الـ handoff، الـ serial console بيطلع characters غلط وأحياناً بيتوقف خالص.

#### المشكلة
الـ kernel command line فيها `console=ttyAML0,115200`. الـ UART شغال مع U-Boot، لكن في الـ kernel يبدأ الـ output مخبط.

```
U-Boot> [boot ok]
Linux kernel 5.x starting...
▒▒▒▒�???init: starting...
```

#### التحليل
الـ Meson8b عنده UARTs متعددة. في الملف، الـ UART pins موزعين على أماكن مختلفة:

```c
/* UART_A على GPIOX */
static const unsigned int uart_tx_a_pins[]  = { GPIOX_4 };
static const unsigned int uart_rx_a_pins[]  = { GPIOX_5 };

/* UART_AO — الـ Always-On UART على GPIOAO */
static const unsigned int uart_tx_ao_a_pins[] = { GPIOAO_0 };
static const unsigned int uart_rx_ao_a_pins[] = { GPIOAO_1 };

/* UART_B — نسختين! */
static const unsigned int uart_tx_b0_pins[] = { GPIOX_16 };  /* mux reg 4, bit 9 */
static const unsigned int uart_rx_b0_pins[] = { GPIOX_17 };

static const unsigned int uart_tx_b1_pins[] = { GPIOX_8 };   /* mux reg 6, bit 19 */
static const unsigned int uart_rx_b1_pins[] = { GPIOX_9 };
```

المشكلة هنا: الـ `uart_b_groups` بتجمع **كل الـ UART_B alternatives**:

```c
static const char * const uart_b_groups[] = {
    "uart_tx_b0", "uart_rx_b0", "uart_cts_b0", "uart_rts_b0",
    "uart_tx_b1", "uart_rx_b1", "uart_cts_b1", "uart_rts_b1"
};
```

المهندس في الـ DTS حدد `uart_b` كـ function، لكن ما حددش الصح groups. الـ pinctrl driver بيطبق أول group في الـ function — وده `uart_tx_b0` على `GPIOX_16`. لكن الـ hardware connections على `GPIOX_8` (uart_tx_b1).

الفرق في الـ mux register:
```c
GROUP(uart_tx_b0, 4, 9),   /* REG4, bit 9 */
GROUP(uart_tx_b1, 6, 19),  /* REG6, bit 19 */
```

لما اتنين bits بيتفعلوا في نفس الوقت، الـ UART output بيكون undefined behavior.

#### الحل
في الـ DTS لازم تحدد الـ groups بالاسم:

```dts
/* غلط — بيعمل ambiguity */
&uart_B {
    pinctrl-0 = <&uart_b_pins>;
};

uart_b_pins: uart-b {
    mux {
        groups = "uart_tx_b", "uart_rx_b";  /* غامض! */
        function = "uart_b";
    };
};

/* صح — بتحدد الـ exact pins */
uart_b_pins: uart-b {
    mux {
        groups = "uart_tx_b1", "uart_rx_b1"; /* GPIOX_8/9 */
        function = "uart_b";
    };
};
```

```bash
# تحقق من الـ mux state
$ cat /sys/kernel/debug/pinctrl/pinctrl-maps | grep uart_tx_b
map: state default type MUX_GROUP controller pinctrl@c1109880
    group uart_tx_b1 function uart_b
```

#### الدرس المستفاد
لما function عندها نفس الـ peripheral على multiple pin alternatives (زي uart_b0 و uart_b1)، لازم دايماً تحدد الـ exact group name في الـ DTS، ومش تعتمد على الـ function name وحده. الـ pinctrl driver بيختار بشكل arbitrary لو في تعارض.

---

### السيناريو 3: IoT Sensor Hub — الـ I2C مش بيكتشف devices

#### العنوان
**الـ I2C bus بيرجع `-ENODEV` لكل الـ sensors على IoT hub**

#### السياق
فريق بيبني IoT sensor hub بيستخدم Amlogic Meson8b board. البورد عندها 3 I2C sensors: temperature sensor على `I2C_A`، accelerometer على `I2C_B`، وـ pressure sensor على `I2C_C`. بعد الـ bring-up، الـ `i2cdetect` ما بيلاقيش أي devices.

#### المشكلة
```bash
$ i2cdetect -y 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
...
# كله "--" — مفيش devices
$ dmesg | grep i2c
[    1.234] i2c-meson i2c-meson.0: i2c controller registered
# الـ controller اتسجل بس ما فيش device responses
```

#### التحليل
في `pinctrl-meson8b.c`، الـ I2C على Meson8b عنده تعقيد تاني: كل I2C bus عنده **multiple physical locations**:

```c
/* I2C_A — على GPIODV */
static const unsigned int i2c_sda_a_pins[] = { GPIODV_24 };
static const unsigned int i2c_sck_a_pins[] = { GPIODV_25 };

/* I2C_B — نسختين */
static const unsigned int i2c_sda_b0_pins[] = { GPIODV_26 };
static const unsigned int i2c_sck_b0_pins[] = { GPIODV_27 };
static const unsigned int i2c_sda_b1_pins[] = { GPIOH_3 };
static const unsigned int i2c_sck_b1_pins[] = { GPIOH_4 };

/* I2C_C — نسختين */
static const unsigned int i2c_sda_c0_pins[] = { GPIODV_28 };
static const unsigned int i2c_sck_c0_pins[] = { GPIODV_29 };
static const unsigned int i2c_sda_c1_pins[] = { GPIOH_5 };
static const unsigned int i2c_sck_c1_pins[] = { GPIOH_6 };

/* I2C_D — نسختين */
static const unsigned int i2c_sda_d0_pins[] = { GPIOX_16 };
static const unsigned int i2c_sck_d0_pins[] = { GPIOX_17 };
static const unsigned int i2c_sda_d1_pins[] = { GPIOH_7 };
static const unsigned int i2c_sck_d1_pins[] = { GPIOH_8 };
```

الـ mux registers المرتبطة:
```c
GROUP(i2c_sda_a,  9, 31),  /* REG9, bit 31 */
GROUP(i2c_sck_a,  9, 30),  /* REG9, bit 30 */
GROUP(i2c_sda_b0, 9, 29),  /* REG9, bit 29 */
GROUP(i2c_sck_b0, 9, 28),  /* REG9, bit 28 */
GROUP(i2c_sda_c0, 9, 27),  /* REG9, bit 27 */
GROUP(i2c_sck_c0, 9, 26),  /* REG9, bit 26 */
```

المهندس عمل الـ DTS صح، لكن المشكلة كانت إن الـ `GPIODV` bank فيها **pull-up resistors معطلة**. في الـ `meson8b_cbus_banks`:

```c
BANK("DV24..29", GPIODV_24, GPIODV_29, 74, 79, 0, 24, 0, 24, 7, 24, 8, 24, 9, 24),
```

الـ BANK macro بيعمل `MESON_REG_PULLEN` بـ `{0, 24}` — الـ register 0، bit 24. لو الـ DTS ما بيفعلش الـ internal pull-up، الـ I2C open-drain lines بتفضل floating.

#### الحل
في الـ DTS لازم تضيف `bias-pull-up`:

```dts
i2c_a_pins: i2c-a {
    mux {
        groups = "i2c_sda_a", "i2c_sck_a";
        function = "i2c_a";
    };
    /* مهم جداً للـ I2C open-drain */
    conf {
        groups = "i2c_sda_a", "i2c_sck_a";
        bias-pull-up;
        drive-strength-microamp = <3000>;
    };
};
```

```bash
# تحقق من الـ pull configuration
$ cat /sys/kernel/debug/pinctrl/pinctrl-devices
# ابحث عن GPIODV_24/25 وشوف الـ pull state

$ i2cdetect -y 0
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: 40 -- -- -- -- -- -- --  # temperature sensor!
```

#### الدرس المستفاد
الـ `meson_pinctrl_data` بيعرّف `MESON_REG_PULLEN` و `MESON_REG_PULL` لكل bank. الـ I2C على Meson8b بيحتاج explicit `bias-pull-up` في الـ DTS لأن الـ internal pull-up معطّل بـ default. ده مش bug في الـ driver — ده design صح لكنه بيحتاج explicit configuration.

---

### السيناريو 4: Media Player — الـ HDMI مش بيشتغل والـ CEC مش بيستجاوب

#### العنوان
**الـ HDMI بيطلع صورة بس الـ CEC remote control مش شغال**

#### السياق
شركة بتصنع media player للسوق الأفريقي بيستخدم Amlogic Meson8b. الـ HDMI video و audio شغالين تمام. لكن الـ CEC (Consumer Electronics Control) — اللي بيخلي الريموت كونترول يتحكم في الـ TV — مش شغال خالص.

#### المشكلة
```bash
$ dmesg | grep cec
[    3.456] meson-cec: probe ok
[   45.123] meson-cec: no pin found for cec
$ cec-ctl --list-devices
# nothing
```

#### التحليل
في `pinctrl-meson8b.c`، الـ CEC موجود في **مكانين مختلفين**:

```c
/* CEC على GPIOH_3 — ضمن cbus */
static const unsigned int hdmi_cec_0_pins[] = { GPIOH_3 };

/* CEC على GPIOAO_12 — ضمن aobus */
static const unsigned int hdmi_cec_1_pins[] = { GPIOAO_12 };
```

وفي الـ functions:
```c
/* في meson8b_cbus_functions */
FUNCTION(hdmi),       /* بيشمل hdmi_cec_0 على GPIOH_3 */

/* في meson8b_aobus_functions */
FUNCTION(hdmi_cec),   /* بيشمل hdmi_cec_1 على GPIOAO_12 */
```

الـ `hdmi_groups`:
```c
static const char * const hdmi_groups[] = {
    "hdmi_hpd", "hdmi_sda", "hdmi_scl", "hdmi_cec_0"
};
```

الـ `hdmi_cec_groups`:
```c
static const char * const hdmi_cec_groups[] = {
    "hdmi_cec_1"
};
```

الفرق الجوهري:
- **`hdmi_cec_0`** على `GPIOH_3`: ده جزء من الـ `cbus` domain، بيتحكم فيه `meson8b_cbus_pinctrl_data`
- **`hdmi_cec_1`** على `GPIOAO_12`: ده جزء من الـ `aobus` domain، بيتحكم فيه `meson8b_aobus_pinctrl_data`

الـ hardware على البورد ده وصّل الـ CEC line لـ `GPIOAO_12` (لأنه في الـ Always-On domain — مناسب للـ standby). المهندس حدد في الـ DTS:

```dts
/* غلط */
&hdmi_tx {
    pinctrl-0 = <&hdmi_pins>;
};
hdmi_pins: hdmi {
    mux {
        groups = "hdmi_hpd", "hdmi_sda", "hdmi_scl", "hdmi_cec_0";
        function = "hdmi";  /* ده cbus، بس الـ CEC على aobus! */
    };
};
```

#### الحل
لازم تفصل الـ HDMI main pins عن الـ CEC، وتستخدم الـ aobus pinctrl:

```dts
/* في الـ cbus node */
hdmi_pins: hdmi {
    mux {
        groups = "hdmi_hpd", "hdmi_sda", "hdmi_scl";
        function = "hdmi";
    };
};

/* في الـ aobus node — لازم تكون referenced من الـ AO pinctrl */
hdmi_cec_ao_pins: hdmi-cec-ao {
    mux {
        groups = "hdmi_cec_1";
        function = "hdmi_cec";
    };
};

&hdmi_tx {
    pinctrl-0 = <&hdmi_pins>, <&hdmi_cec_ao_pins>;
    pinctrl-names = "default";
};
```

```bash
$ dmesg | grep cec
[    3.456] meson-cec: probe ok
[    3.470] meson-cec: CEC pin GPIOAO_12 configured
$ cec-ctl --list-devices
Found device: meson-cec
```

#### الدرس المستفاد
الـ Meson8b عنده **two separate pinctrl controllers**: `cbus` و `aobus`. الـ aobus يعيش في الـ Always-On power domain، ومهم جداً للـ standby features زي CEC. أي pin على `GPIOAO_*` لازم يتعامل معاه من خلال الـ `amlogic,meson8b-aobus-pinctrl` node في الـ DTS مش الـ cbus node.

---

### السيناريو 5: Custom Board Bring-up — الـ SPI Flash مش بيبوت

#### العنوان
**الـ NOR Flash على SPI مش بيُكتشف أثناء bring-up بورد custom على Meson8b**

#### السياق
فريق hardware صمم custom board للـ industrial automation بيستخدم Amlogic Meson8b. البورد بيستخدم NOR Flash كـ boot device بدل NAND. الـ U-Boot بيبوت من الـ eMMC، لكن الـ kernel مش بيشوف الـ NOR Flash.

#### المشكلة
```bash
$ dmesg | grep spi
[    2.123] spi-nor spi0.0: unrecognized JEDEC id bytes: ff ff ff
[    2.124] spi-nor: probe with driver spi-nor failed with error -19
```

الـ `ff ff ff` معناها إن الـ SPI lines مش بتتحرك أصلاً — كأنها floating.

#### التحليل
في `pinctrl-meson8b.c`، الـ SPI عنده alternative locations:

```c
/* SPI_0 — على GPIOX */
static const unsigned int spi_sclk_0_pins[] = { GPIOX_8 };
static const unsigned int spi_miso_0_pins[] = { GPIOX_9 };
static const unsigned int spi_mosi_0_pins[] = { GPIOX_10 };
static const unsigned int spi_ss0_0_pins[]  = { GPIOX_20 };

/* SPI_1 — على GPIOH */
static const unsigned int spi_ss0_1_pins[]  = { GPIOH_3 };
static const unsigned int spi_miso_1_pins[] = { GPIOH_4 };
static const unsigned int spi_mosi_1_pins[] = { GPIOH_5 };
static const unsigned int spi_sclk_1_pins[] = { GPIOH_6 };

/* SPI extra chip selects على GPIOH */
static const unsigned int spi_ss1_pins[]    = { GPIOH_0 };
static const unsigned int spi_ss2_pins[]    = { GPIOH_1 };
```

الـ GROUP macros بتكشف الـ mux register bits:

```c
/* SPI_0 */
GROUP(spi_sclk_0, 4, 22),   /* REG4, bit 22 */
GROUP(spi_miso_0, 4, 24),   /* REG4, bit 24 */
GROUP(spi_mosi_0, 4, 23),   /* REG4, bit 23 */
GROUP(spi_ss0_0,  4, 25),   /* REG4, bit 25 */

/* SPI_1 */
GROUP(spi_ss0_1,  9, 13),   /* REG9, bit 13 */
GROUP(spi_miso_1, 9, 12),   /* REG9, bit 12 */
GROUP(spi_mosi_1, 9, 11),   /* REG9, bit 11 */
GROUP(spi_sclk_1, 9, 10),   /* REG9, bit 10 */
```

لكن، المشكلة الأعمق: الـ hardware بيستخدم `BOOT_11`, `BOOT_12`, `BOOT_13`, `BOOT_18` للـ NOR Flash — وده في الـ **nor_groups**:

```c
static const unsigned int nor_d_pins[]  = { BOOT_11 };
static const unsigned int nor_q_pins[]  = { BOOT_12 };
static const unsigned int nor_c_pins[]  = { BOOT_13 };
static const unsigned int nor_cs_pins[] = { BOOT_18 };

GROUP(nor_d,  5, 1),   /* REG5, bit 1 */
GROUP(nor_q,  5, 3),   /* REG5, bit 3 */
GROUP(nor_c,  5, 2),   /* REG5, bit 2 */
GROUP(nor_cs, 5, 0),   /* REG5, bit 0 */
```

ده مش SPI controller عادي — ده الـ NOR interface المخصص. المهندس كان بيحاول يستخدم `spi` function بدل `nor` function.

الـ `BOOT` bank:
```c
BANK("BOOT", BOOT_0, BOOT_18, 24, 42, 2, 0, 2, 0, 9, 0, 10, 0, 11, 0),
```

لكن `BOOT_11..13` و `BOOT_18` بيتشاركوا مع NAND pins:
```c
static const unsigned int nand_ale_pins[]     = { BOOT_11 };
static const unsigned int nand_cle_pins[]     = { BOOT_12 };
static const unsigned int nand_wen_clk_pins[] = { BOOT_13 };
static const unsigned int nand_dqs_18_pins[]  = { BOOT_18 };
```

لو الـ DTS بيفعّل الـ NAND function، الـ NOR Flash مش هيشتغل بسبب الـ pin conflict.

#### الحل
أولاً، تأكد إن الـ NAND معطّل في الـ DTS:

```dts
/* عطّل الـ NAND */
&nand {
    status = "disabled";
};

/* فعّل الـ NOR */
&spifc {
    status = "okay";
    pinctrl-0 = <&nor_pins>;
    pinctrl-names = "default";

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <50000000>;
    };
};

nor_pins: nor {
    mux {
        groups = "nor_d", "nor_q", "nor_c", "nor_cs";
        function = "nor";
    };
};
```

```bash
# تحقق من conflict resolution
$ cat /sys/kernel/debug/pinctrl/pinctrl-maps | grep BOOT_11
map: state default type MUX_GROUP controller pinctrl@c1109880
    group nor_d function nor

$ dmesg | grep spi-nor
[    2.100] spi-nor spi0.0: w25q128 (16384 Kbytes)
[    2.101] 4 fixed-partitions partitions found on MTD device spi0.0
```

إذا الـ conflict لسه موجود، استخدم:

```bash
# شوف مين بيستخدم BOOT_11
$ cat /sys/kernel/debug/gpio | grep BOOT_11
gpio-xx  (BOOT_11            ) out lo
# لو ظاهر كـ GPIO، معناها الـ mux مش اتطبق
```

#### الدرس المستفاد
الـ `BOOT` bank في Meson8b بيخدم أغراض متعددة: NAND، NOR، وـ SD Card (via `sd_c` و `sdxc_c` groups). كل الـ groups دي بتتشارك نفس الـ physical pins. لازم تعطّل صراحةً أي function مش محتاجه في الـ DTS، وإلا الـ pin conflict بيحصل. الـ `nor` function مختلف عن `spi` — هو interface مخصص للـ NOR Flash وبيتحكم في REG5 مش REG4 أو REG9.
## Phase 7: مصادر ومراجع

### مصادر LWN.net

دي أهم المقالات اللي اتنشرت على LWN.net وبتغطي تطور الـ pinctrl driver للـ Amlogic Meson SoCs:

| المقال | الرابط |
|--------|--------|
| Pinctrl driver for Amlogic Meson SoCs — أول patch للـ driver الأصلي | [LWN.net/Articles/616225](https://lwn.net/Articles/616225/) |
| Amlogic Meson pinctrl driver — نسخة محسّنة من الـ patchset | [LWN.net/Articles/620822](https://lwn.net/Articles/620822/) |
| pinctrl: meson-a1: add pinctrl driver — دعم Meson-A1 | [LWN.net/Articles/804174](https://lwn.net/Articles/804174/) |
| pinctrl: meson-s4: add pinctrl driver — دعم Meson-S4 | [LWN.net/Articles/879912](https://lwn.net/Articles/879912/) |
| irqchip: meson: add support for GPIO interrupt controller | [LWN.net/Articles/703946](https://lwn.net/Articles/703946/) |
| Add pinctrl driver support for Amlogic T7 SoCs | [LWN.net/Articles/945285](https://lwn.net/Articles/945285/) |
| pinctrl: add a generic pin config interface — أساس الـ subsystem | [LWN.net/Articles/468770](https://lwn.net/Articles/468770/) |
| Documentation/pinctrl.txt — أول توثيق رسمي للـ subsystem | [LWN.net/Articles/465077](https://lwn.net/Articles/465077/) |

---

### التوثيق الرسمي في الـ Kernel

#### داخل شجرة `Documentation/`

```
Documentation/driver-api/pinctl.rst          # الدليل الرئيسي للـ subsystem
Documentation/devicetree/bindings/pinctrl/   # DT bindings لكل الـ vendors
```

**الـ binding الخاص بـ Amlogic:**

```
Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl.yaml
```

**الـ dt-binding header للـ Meson8b:**

```c
include/dt-bindings/gpio/meson8b-gpio.h   // أسماء الـ pins اللي بيستخدمها الـ DTS
```

**الرابط الأونلاين للتوثيق:**

- [PINCTRL subsystem — kernel.org](https://www.kernel.org/doc/html/v4.13/driver-api/pinctl.html)
- [Documentation/pinctrl.txt — kernel.org](https://www.kernel.org/doc/Documentation/pinctrl.txt)

---

### الـ Source Files الأساسية في الـ Kernel

```
drivers/pinctrl/meson/pinctrl-meson.c        # الـ core المشترك لكل Meson SoCs
drivers/pinctrl/meson/pinctrl-meson.h        # الـ structs والـ macros المشتركة
drivers/pinctrl/meson/pinctrl-meson8-pmx.c   # عمليات الـ pinmux لـ Meson8/Meson8b
drivers/pinctrl/meson/pinctrl-meson8-pmx.h   # header للـ pmx ops
drivers/pinctrl/meson/pinctrl-meson8b.c      # تعريفات الـ pins والـ banks لـ Meson8b
```

---

### مناقشات الـ Mailing List

| الموضوع | الرابط |
|---------|--------|
| [PATCH 1/3] pinctrl: add driver for Amlogic Meson SoCs — أول إرسال | [lkml.iu.edu](https://lkml.iu.edu/hypermail/linux/kernel/1410.0/04163.html) |
| [RFC] meson8b: fix requesting GPIOs > GPIOZ_3 — bug fix مهم | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-amlogic/patch/20180124002738.32338-1-martin.blumenstingl@googlemail.com/) |
| pinctrl: meson: fix drive strength register calculation | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-amlogic/patch/20200610041329.12948-1-hhk7734@gmail.com/) |
| pinctrl: meson: add support for GPIO interrupts — RfC v4 | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-amlogic/patch/3265fbc9-467d-67fa-8752-f750f234dedd@gmail.com/) |
| pinctrl: meson: add support for GPIO interrupts — v6 | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-amlogic/patch/a86d38af-9103-7557-9986-2dc187815569@gmail.com/) |
| ARM: dts: amlogic: Split pinctrl device for Meson8/Meson8b | [lore.kernel.org](https://lore.kernel.org/all/1456869876-19320-4-git-send-email-carlo@caione.org/) |
| pinctrl: meson: add a new callback for SoCs fixup | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1572004167-24150-3-git-send-email-qianggui.song@amlogic.com/) |

---

### Commits مهمة في الـ Git History

**الـ commit الأول للـ Meson pinctrl driver** — أضافه **Beniamino Galvani** في أكتوبر 2014، بعدين جه **Carlo Caione** وأضاف الـ Meson8b-specific data في 2015 (هو صاحب الـ `pinctrl-meson8b.c` الأصلي كما يظهر في الـ copyright):

```
Author: Carlo Caione <carlo@endlessm.com>
File:   drivers/pinctrl/meson/pinctrl-meson8b.c
Commit: ضمن kernel v4.x series
```

**لمشاهدة تاريخ الـ commits:**

```bash
git log --oneline -- drivers/pinctrl/meson/pinctrl-meson8b.c
git log --oneline -- drivers/pinctrl/meson/pinctrl-meson.c
```

**موقع تتبع تطور الـ Meson mainlining:**

- [linux-meson.com/mainlining](https://linux-meson.com/mainlining.html)

**مقال BayLibre عن تحسين دعم Amlogic في mainline Linux:**

- [baylibre.com — Improved AmLogic Support in Mainline Linux](https://baylibre.com/improved-amlogic-support-mainline-linux/)

---

### Kernelnewbies.org

**الـ pinctrl** بيتذكر في صفحات changelog للكيرنل في kernelnewbies — الصفحات دي بتوثّق التغييرات لكل إصدار:

| الإصدار | الرابط |
|---------|--------|
| Linux 4.9 — تضمن تحسينات pinctrl | [kernelnewbies.org/Linux_4.9](https://kernelnewbies.org/Linux_4.9) |
| Linux 6.8 — إضافات كثيرة للـ pinctrl | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) |
| Linux 6.11 — pinctrl updates | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| Linux 6.15 — Amlogic pinctrl support | [kernelnewbies.org/Linux_6.15](https://kernelnewbies.org/Linux_6.15) |
| LinuxChanges — صفحة التغييرات العامة | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) |

---

### eLinux.org

مفيش صفحة مخصصة للـ Meson pinctrl على elinux.org، بس في مواد عامة مفيدة:

| المرجع | الرابط |
|--------|--------|
| Introduction to pin muxing and GPIO control under Linux (PDF — ELC 2021) | [elinux.org PDF](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) |
| Tests: i2c-demux-pinctrl | [elinux.org/Tests:i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) |
| Linux Kernel Resources | [elinux.org/Linux_Kernel_Resources](https://elinux.org/Linux_Kernel_Resources) |
| EBC Device Trees — أمثلة Device Tree مع pinctrl | [elinux.org/EBC_Device_Trees](https://elinux.org/EBC_Device_Trees) |

---

### الكتب المرجعية

#### Linux Device Drivers (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول ذات الصلة:**
  - Chapter 9: Communicating with Hardware — فهم الـ memory-mapped registers
  - Chapter 14: The Linux Device Model — فهم `platform_driver` و `of_device_id`
- **الرابط (مجاني):** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصول ذات الصلة:**
  - Chapter 17: Devices and Modules — الـ device model
  - Chapter 18: Debugging — لفهم الـ kernel debugging
- بيشرح كيف بتشتغل الـ subsystems مع بعض في الكيرنل

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل ذو الصلة:**
  - Chapter 14: Kernel Initialization — فهم الـ device tree و platform devices
  - بيغطي pin multiplexing على الـ embedded SoCs بشكل عملي

#### Linux Device Driver Development — John Madieu
- الكتاب الأحدث (2022) ومخصص للـ embedded Linux
- بيغطي الـ pinctrl subsystem بشكل مباشر مع أمثلة عملية
- مناسب جداً للي بيشتغل على Amlogic أو أي SoC مشابه

---

### Search Terms للبحث عن معلومات إضافية

للبحث في الـ kernel mailing list والـ patchwork:

```
site:lore.kernel.org pinctrl meson8b
site:patchwork.kernel.org amlogic pinctrl
site:lwn.net amlogic pinctrl
```

في الـ kernel source نفسه:

```bash
# البحث عن كل الـ drivers المشابهة
ls drivers/pinctrl/meson/

# قراءة الـ binding documentation
cat Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl.yaml

# تتبع تغييرات الـ driver
git log --oneline drivers/pinctrl/meson/

# البحث عن الـ compatible strings
grep -r "amlogic,meson8b" arch/arm/boot/dts/
```

**الكلمات المفتاحية للبحث:**

```
linux pinctrl subsystem meson amlogic
amlogic meson8b gpio pinmux driver
pinctrl-meson aobus cbus banks
meson8b GPIOX GPIOY GPIOAO pinctrl
linux kernel pin controller amlogic SoC
```
## Phase 8: Writing simple module

### الفكرة

**`meson_pinctrl_probe`** هي الـ entry point اللي بتتسمى لما الـ platform driver بتعمل match مع أي device من الـ `meson8b_pinctrl_dt_match` table — سواء كان `cbus` أو `aobus`. هنعمل **kprobe** عليها عشان نعرف إيمتى الـ pinctrl subsystem بيتهيأ على hardware الـ Amlogic Meson8b ونطبع اسم الـ platform device.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on meson_pinctrl_probe — Meson8b pinctrl probe tracer
 *
 * Hooks into meson_pinctrl_probe() to log each time the Meson8b
 * pin controller is probed (cbus or aobus bank).
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit, pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ...        */
#include <linux/platform_device.h> /* struct platform_device, dev_name()     */

/*
 * الـ kprobe struct — بيربط اسم الدالة المستهدفة
 * بالـ pre_handler اللي هيتنفذ قبل ما الدالة الأصلية تبدأ.
 */
static struct kprobe kp_meson_probe;

/*
 * pre_handler — بيتنفذ مباشرة قبل meson_pinctrl_probe()
 *
 * @p:    pointer للـ kprobe نفسه (مش محتاجينه هنا)
 * @regs: نسخة من registers اللحظة دي —
 *        الـ argument الأول (pdev) موجود في regs->di على x86-64
 *        أو regs->regs[0] على arm64؛ لكن المعيار الأسهل
 *        هو استخدام container_of / indirect — هنا بنكتفي
 *        بطباعة اسم رمزي من الـ kprobe symbol.
 *
 * بنلجأ للـ pt_regs عشان نقرأ الـ argument الأول (struct platform_device *)
 * بشكل portable عبر معمارية ARM64 اللي بتشتغل عليها Meson8b.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * على ARM64: الـ argument الأول بيتمرر في register x0
     * regs->regs[0] بيحتوي على قيمة الـ pointer لـ platform_device.
     * بنعمل cast لـ platform_device * بشكل آمن.
     */
    struct platform_device *pdev =
        (struct platform_device *)(unsigned long)regs->regs[0];

    if (pdev)
        pr_info("[meson8b_probe_tracer] meson_pinctrl_probe called "
                "for device: \"%s\"\n", dev_name(&pdev->dev));
    else
        pr_info("[meson8b_probe_tracer] meson_pinctrl_probe called "
                "(pdev = NULL?)\n");

    return 0; /* 0 = لازم نرجع 0 عشان الـ kprobe ميوقفش التنفيذ */
}

/*
 * post_handler — بيتنفذ بعد ما meson_pinctrl_probe() ترجع.
 * هنا بنستخدمه بس لتأكيد إن الـ probe خلص.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    pr_info("[meson8b_probe_tracer] meson_pinctrl_probe returned.\n");
}

static int __init meson8b_probe_tracer_init(void)
{
    int ret;

    /*
     * بنحدد اسم الدالة المستهدفة كـ string —
     * الـ kprobe subsystem بيحولها لعنوان بنفسه
     * عن طريق kallsyms.
     */
    kp_meson_probe.symbol_name = "meson_pinctrl_probe";
    kp_meson_probe.pre_handler  = handler_pre;
    kp_meson_probe.post_handler = handler_post;

    ret = register_kprobe(&kp_meson_probe);
    if (ret < 0) {
        pr_err("[meson8b_probe_tracer] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[meson8b_probe_tracer] kprobe planted on meson_pinctrl_probe "
            "at %p\n", kp_meson_probe.addr);
    return 0;
}

static void __exit meson8b_probe_tracer_exit(void)
{
    /*
     * لازم نشيل الـ kprobe قبل ما الـ module يتفك من الـ kernel
     * عشان لو الدالة اتنفذت بعد kein تفك الـ module
     * هيحصل kernel panic من الـ dangling handler pointer.
     */
    unregister_kprobe(&kp_meson_probe);
    pr_info("[meson8b_probe_tracer] kprobe removed.\n");
}

module_init(meson8b_probe_tracer_init);
module_exit(meson8b_probe_tracer_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example Author <author@example.com>");
MODULE_DESCRIPTION("kprobe tracer for meson_pinctrl_probe (Meson8b pinctrl)");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | كل الـ macros الأساسية للـ module: `pr_info`, `module_init`, `MODULE_LICENSE` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe`, `pt_regs` |
| `linux/platform_device.h` | `struct platform_device` و `dev_name()` عشان نطبع اسم الـ device |

**الـ** `linux/platform_device.h` ضروري لأن `meson_pinctrl_probe` بتاخد `struct platform_device *` كـ argument وحيد، واللي بنقراه من الـ registers.

---

#### الـ `handler_pre`

الـ kprobe بيوقف التنفيذ لحظة دخول الدالة المستهدفة قبل ما أي instruction فيها ينفذ. **الـ** `pt_regs` بتحتوي على حالة الـ registers — على ARM64 الـ argument الأول دايماً في `regs->regs[0]`، وده بيوفر وصول مباشر لـ `pdev` من غير ما نعدّل الـ source الأصلي.

---

#### الـ `handler_post`

بيتنفذ بعد إن `meson_pinctrl_probe` ترجع — مفيد لو محتاجين نحسب الوقت اللي أخدته أو نتحقق من نتيجتها لاحقاً. دلوقتي بيطبع رسالة بسيطة بس هيكله موجود للتوسع.

---

#### `module_init` / `module_exit`

- **`module_init`**: بيستدعي `register_kprobe` اللي بيدوّر على `meson_pinctrl_probe` في الـ kallsyms ويزرع الـ breakpoint. لو فشل (مثلاً الدالة `inline` أو الاسم مش موجود) بيرجع error ومبيحصلش load.
- **`module_exit`**: بيستدعي `unregister_kprobe` لإزالة الـ hook قبل تفريغ الـ module من الـ memory — ده حماية إلزامية من الـ use-after-free في kernel context.

---

### طريقة الاختبار

```bash
# بناء الـ module (Makefile بسيط)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod meson8b_probe_tracer.ko

# مراقبة الـ output (لو في Meson8b hardware أو QEMU)
sudo dmesg -w | grep meson8b_probe_tracer

# إزالة الـ module
sudo rmmod meson8b_probe_tracer
```

**ملحوظة**: لو الـ kernel مش على ARM64 (مثلاً x86-64 للاختبار) يبقى `regs->regs[0]` هيبقى `regs->di` — لازم تعدّل السطر ده بناءً على الـ architecture.
