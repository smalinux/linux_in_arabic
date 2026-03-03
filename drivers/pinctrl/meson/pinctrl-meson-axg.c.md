## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من **ARM/Amlogic Meson SoC support** — الـ subsystem المسؤول عن دعم معالجات Amlogic Meson في Linux kernel. الـ maintainers هم Neil Armstrong و Kevin Hilman من Linaro/BayLibre. الكود كله موجود في `drivers/pinctrl/meson/`.

---

### المشكلة اللي بتتحل

تخيل عندك بورد إلكترونية فيها معالج (SoC). المعالج ده فيه مئات الـ **pins** — أرجل معدنية بتوصل المعالج بالعالم الخارجي. كل pin ممكن يشتغل بأكتر من طريقة:

- يكون **GPIO** عادي (دخل أو خرج رقمي)
- يكون **UART TX** بيبعت بيانات سيريال
- يكون **I2C SDA** بيتكلم مع حساسات
- يكون **SPI CLK** بيتحكم في فلاش
- يكون **Ethernet MDIO** أو **PWM** أو **JTAG** أو ... إلخ

المشكلة: pin واحد جسديًا بس وظائف كتير. مين بيقرر يشتغل بأي وظيفة؟ ده الـ **pinctrl subsystem** — وهو اللي بيعمل **pin multiplexing (pinmux)**.

---

### القصة كاملة — ELI5

تخيل عندك لوحة مفاتيح فيها 100 مفتاح. كل مفتاح ممكن يتكلم مع اتنين أو تلاتة أجهزة مختلفة في نفس الوقت بس يستخدمه جهاز واحد بس في الوقت ده. فيه **مفتاح تحكم** بيقول: "المفتاح ده دلوقتي للـ WiFi مش للـ Bluetooth".

ده بالظبط اللي بيحصل في الـ SoC. عندك **registers** في الـ hardware بتتحكم في وظيفة كل pin. تكتب قيمة معينة في الـ register، الـ pin بيشتغل كـ UART. تكتب قيمة تانية، نفس الـ pin بيشتغل كـ SPI.

الـ **AXG** هو اسم جيل محدد من معالجات Amlogic Meson — بيستخدم في أجهزة زي Android TV boxes و single-board computers. الملف ده بيقول للكرنل: "هذا هو الـ Meson AXG، وده جدول كل الـ pins بتاعته، وكل وظيفة ممكن يشتغلها كل pin."

---

### معمارية الـ Hardware في AXG

الـ AXG فيه **دومينين** مستقلين للـ pins:

| الدومين | الـ Pins | الغرض |
|---------|----------|--------|
| **periphs** (Peripherals) | GPIOZ, BOOT, GPIOA, GPIOX, GPIOY | الأجهزة الطرفية العادية (UART, SPI, I2C, ETH, ...) |
| **aobus** (Always-On Bus) | GPIOAO, GPIO_TEST_N | الـ pins اللي شغالة حتى لو المعالج نايم (power management) |

الدومين الـ **Always-On** مهم جدًا لأن الـ hardware بتاعته صحيان 24/7 حتى في deep sleep — بيستخدم للـ IR receiver (ريموت كنترول) والـ UART للـ debug وغيره.

---

### بنية الكود — كيف بيشتغل

```
pin رقمي (مثلاً GPIOX_8)
        ↓
  bank (مجموعة pins متجاورة)
        ↓
  group (pin أو أكتر بيعملوا وظيفة واحدة)
        ↓
  function (الاسم المنطقي للوظيفة: uart_a, spi0, ...)
        ↓
  register في الـ hardware (بيحدد الـ mux value)
```

**الـ `pinctrl_pin_desc`**: جدول بأسماء وأرقام كل الـ pins.

**الـ `meson_pmx_group`**: كل group بيمثل مجموعة pins بتشتغل مع بعض لوظيفة معينة، مع رقم الـ mux function المقابل.

**الـ `meson_pmx_func`**: الاسم المنطقي للوظيفة (مثلاً `uart_a`) مع قائمة الـ groups اللي بتحققها.

**الـ `meson_bank`**: بيحدد للكرنل عنوان الـ registers اللي بتتحكم في كل bank (pull-up/down، direction، output، input، drive strength).

**الـ `meson_pmx_bank`**: بيحدد عنوان الـ mux register لكل bank.

---

### مثال عملي

GPIOX_8 في الـ AXG ممكن يكون:
- **GPIO** عادي (function 0)
- **uart_tx_a** — إرسال UART A (function 1 في bank X)
- **eth_txd0_x** — إرسال Ethernet (function 4 في bank X)

```c
/* GPIOX_8 ممكن يشتغل كـ UART TX أو Ethernet TX أو GPIO */
static const unsigned int uart_tx_a_pins[] = {GPIOX_8};   /* UART A TX */
static const unsigned int eth_txd0_x_pins[] = {GPIOX_8};  /* Ethernet TX data 0 */

GROUP(uart_tx_a, 1),    /* function 1 في bank X */
GROUP(eth_txd0_x, 4),   /* function 4 في bank X */
```

الـ `GROUP(uart_tx_a, 1)` معناها: لما تيجي تفعّل UART A TX، اكتب القيمة 1 في الـ mux register للـ pin ده.

---

### الـ Always-On Domain (aobus)

الـ aobus مختلف شوية — بيحتاج initialization خاصة عبر `meson8_aobus_parse_dt_extra()`. ده لأن الـ always-on domain بيشارك نفس الـ register space مع الـ power management controller، فيحتاج إعداد إضافي من الـ Device Tree.

---

### الوظائف الكاملة في AXG

**Peripherals domain:**

| الفئة | الوظائف |
|-------|---------|
| Storage | emmc, nand, nor, sdio |
| Serial | uart_a, uart_b, uart_ao_b_z |
| SPI | spi0, spi1 |
| I2C | i2c0, i2c1, i2c2, i2c3 |
| Network | eth (RGMII + MII) |
| PWM | pwm_a, pwm_b, pwm_c, pwm_d, pwm_vs |
| Audio | spdif_in, spdif_out, pdm, tdma, tdmb, tdmc, mclk_b, mclk_c |
| Debug | jtag_ee |

**Always-On domain:**

| الفئة | الوظائف |
|-------|---------|
| Serial | uart_ao_a, uart_ao_b |
| I2C | i2c_ao, i2c_ao_slave |
| IR | remote_input_ao, remote_out_ao |
| PWM | pwm_ao_a, pwm_ao_b, pwm_ao_c, pwm_ao_d |
| Debug | jtag_ao |
| Clock | gen_clk_ee |

---

### الملفات المكوّنة للـ Subsystem

```
drivers/pinctrl/meson/
├── pinctrl-meson.c              ← الكود المشترك لكل Meson SoCs (probe, gpio ops, regmap)
├── pinctrl-meson.h              ← الـ structs والـ macros المشتركة
├── pinctrl-meson-axg.c          ← ← ملفنا: بيانات AXG SoC (pins, groups, functions, banks)
├── pinctrl-meson-axg-pmx.c      ← منطق الـ mux الخاص بجيل AXG (كيف تكتب في الـ register)
├── pinctrl-meson-axg-pmx.h      ← structs الـ mux الخاصة بـ AXG
├── pinctrl-meson-g12a.c         ← نفس الفكرة لـ G12A SoC
├── pinctrl-meson-gxl.c          ← نفس الفكرة لـ GXL SoC
├── pinctrl-meson-gxbb.c         ← نفس الفكرة لـ GXBB SoC
├── pinctrl-meson8.c             ← نفس الفكرة لـ Meson8 SoC
├── pinctrl-meson8b.c            ← نفس الفكرة لـ Meson8b SoC
├── pinctrl-meson8-pmx.c         ← منطق الـ mux الخاص بجيل Meson8
├── pinctrl-meson8-pmx.h
├── pinctrl-meson-a1.c           ← A1 SoC
├── pinctrl-meson-s4.c           ← S4 SoC
├── pinctrl-amlogic-c3.c         ← C3 SoC
├── pinctrl-amlogic-t7.c         ← T7 SoC
└── pinctrl-amlogic-a4.c         ← A4 SoC

include/dt-bindings/gpio/
└── meson-axg-gpio.h             ← أرقام الـ pins للـ Device Tree و kernel headers

arch/arm64/boot/dts/amlogic/
└── meson-axg.dtsi               ← الـ Device Tree اللي بيعرف الـ pinctrl nodes
```

---

### العلاقة بين الملفات

```
pinctrl-meson-axg.c          ← بيانات ثابتة (static data tables)
        ↓ يستخدم
pinctrl-meson.h               ← تعريف الـ structs المشتركة
        ↓
pinctrl-meson.c               ← الـ core logic: probe, gpio_chip, regmap ops
        ↓ يستخدم
pinctrl-meson-axg-pmx.c      ← كيف تكتب الـ mux value في الـ register
        ↓
Linux pinctrl framework       ← الـ API الموحدة في الكرنل
(drivers/pinctrl/core.c)
```

الملف `pinctrl-meson-axg.c` هو **بيانات بحتة** — مافيهوش أي logic. كل الـ logic موجودة في `pinctrl-meson.c` و`pinctrl-meson-axg-pmx.c`. الفكرة إن كل SoC جديد محتاج بس يضيف ملف بيانات زيه.
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة — ليش الـ Pinctrl موجود أصلاً؟

الـ SoC الحديث زي Amlogic Meson AXG عنده مئات الـ pins فيزيائية، لكن كل pin ممكن يشتغل بأكتر من وظيفة واحدة. مثلاً GPIOX_8 ممكن يكون:
- GPIO عادي
- UART_A TX
- Ethernet TX data

المشكلة إن ده بيتحكم فيه عن طريق **مux registers** في الـ hardware — bit fields بتحدد الـ function لكل pin. قبل الـ pinctrl framework، كل driver كان بيعمل اللي عايزه في الـ hardware مباشرة:
- الـ UART driver بيكتب في register معين
- الـ SPI driver بيكتب في register تاني
- ومفيش coordination خالص

النتيجة: تعارضات، pins اتطلب من اتنين drivers في نفس الوقت، وcode متكرر في كل driver.

### الحل — نهج الـ Kernel

الـ kernel عمل **subsystem مركزي** بيملك كل الـ pins ويعمل arbitration بينها. الفكرة:

1. **كل pin عنده رقم فريد** في namespace عالمي
2. الـ pins بتتجمع في **groups** — مجموعة pins بتعمل وظيفة واحدة سوا
3. الـ groups بتتجمع في **functions** — اسم الـ peripheral اللي هيشتغل
4. أي driver عايز يستخدم pins، بيطلبهم من الـ pinctrl core — اللي بيتأكد مفيش conflict

---

### Big Picture Architecture

```
  ┌──────────────────────────────────────────────────────────────┐
  │                      Device Tree (.dts)                       │
  │  amlogic,meson-axg-periphs-pinctrl                           │
  │  amlogic,meson-axg-aobus-pinctrl                             │
  └─────────────────────┬────────────────────────────────────────┘
                        │ of_match → meson_axg_pinctrl_dt_match
                        ▼
  ┌──────────────────────────────────────────────────────────────┐
  │              Platform Driver Layer                            │
  │         meson_axg_pinctrl_driver                             │
  │         .probe = meson_pinctrl_probe()  [pinctrl-meson.c]    │
  └─────────────────────┬────────────────────────────────────────┘
                        │
                        ▼
  ┌──────────────────────────────────────────────────────────────┐
  │            meson_pinctrl_data  (static const)                │
  │  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐   │
  │  │    pins[]   │  │   groups[]   │  │     funcs[]       │   │
  │  │ pinctrl_pin_│  │meson_pmx_    │  │ meson_pmx_func    │   │
  │  │    desc     │  │   group      │  │ (name + groups[]) │   │
  │  └─────────────┘  └──────────────┘  └───────────────────┘   │
  │  ┌─────────────┐  ┌──────────────┐                           │
  │  │   banks[]   │  │  pmx_ops     │                           │
  │  │ meson_bank  │  │ meson_axg_   │                           │
  │  │(reg offsets)│  │   pmx_ops    │                           │
  │  └─────────────┘  └──────────────┘                           │
  └─────────────────────┬────────────────────────────────────────┘
                        │ pinctrl_register_and_init()
                        ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                   Pinctrl Core (kernel)                       │
  │   - يحتفظ بـ global pin ownership table                      │
  │   - يعمل arbitration بين الـ consumers                       │
  │   - يعرض /sys/kernel/debug/pinctrl/                          │
  └────────────┬──────────────────────┬───────────────────────────┘
               │                      │
               ▼                      ▼
  ┌────────────────────┐   ┌─────────────────────────┐
  │   GPIO Subsystem   │   │   Peripheral Drivers     │
  │   gpio_chip        │   │  (UART, SPI, I2C, ETH)  │
  │   (embedded in     │   │  بيطلبوا pinctrl state   │
  │   meson_pinctrl)   │   │  من الـ device tree      │
  └────────────────────┘   └─────────────────────────┘
```

---

### التقسيم الفيزيائي: Periphs vs AO

الـ Meson AXG عنده **power domain منفصل** للـ Always-On (AO) — بيفضل شغال حتى لما باقي الـ SoC نايم. عشان كده في **اتنين instances** منفصلين للـ pinctrl:

```
┌────────────────────────────────┐    ┌─────────────────────────────┐
│   Peripherals Domain           │    │   Always-On Domain           │
│   (periphs-banks)              │    │   (aobus-banks)              │
│                                │    │                              │
│  GPIOZ[0..10]  → bank Z        │    │  GPIOAO[0..13]  → bank AO   │
│  BOOT[0..14]   → bank BOOT     │    │  GPIO_TEST_N                 │
│  GPIOA[0..20]  → bank A        │    │                              │
│  GPIOX[0..22]  → bank X        │    │  Functions:                  │
│  GPIOY[0..15]  → bank Y        │    │  uart_ao_a/b, i2c_ao         │
│                                │    │  pwm_ao_*, jtag_ao           │
│  Functions: emmc, nand, nor    │    │  remote_input, ir_out        │
│  sdio, spi0/1, i2c0-3, uart    │    │                              │
│  eth, pwm, spdif, tdm, pdm     │    │  regmap منفصل               │
└────────────────────────────────┘    └─────────────────────────────┘
       compatible:                           compatible:
  "amlogic,meson-axg-periphs-pinctrl"  "amlogic,meson-axg-aobus-pinctrl"
```

كل instance عنده `meson_pinctrl_data` منفصل بيشير لـ pins وgroups وfuncs وbanks خاصة بيه.

---

### التشبيه الواقعي — مكتب الإيجارات

تخيل مبنى فيه **شقق** (pins)، وكل شقة ممكن تتأجر كـ:
- **سكني** (GPIO)
- **تجاري** (UART)
- **طبي** (SPI)
- وهكذا

| المفهوم الحقيقي | التشبيه |
|---|---|
| `pinctrl_pin_desc` | عقد ملكية كل شقة باسمها ورقمها |
| `meson_pmx_group` | مجموعة شقق متجاورة (زي فيلا) — لازم تتأجر مع بعض |
| `meson_pmx_func` | نوع الاستخدام: "مستشفى" يحتاج مجموعات معينة |
| `meson_bank` | حي سكني — كل الشقق فيه بتتحكم بنفس مكتب الحي (register base) |
| `pinctrl core` | مكتب التسجيل المركزي — ما يأجرش نفس الشقة لاتنين |
| `set_mux()` | كتابة "مستشفى" في سجل الشقة رسمياً |
| `regmap` | الأرشيف الإلكتروني للسجلات — write-through للـ hardware |

التفاصيل المهمة في التشبيه:
- **الشقة نفسها** ممكن تكون في نفس الوقت "جزء من فيلا SDIO" و"شقة GPIO منفردة" — لكن ما تنفعش الاتنين مع بعض → ده الـ conflict اللي الـ pinctrl core بيمنعه
- **الحي (bank)** عنده مكتب (register) واحد بيتحكم في كل شققه — اتجاه، إخراج، إدخال، pull-up/down كلها في register offset محدد
- **مكتب التسجيل** (pinctrl core) ما بيشوفش الـ hardware مباشرة — بيكلم مكتب الحي (bank regs) عبر الأرشيف الإلكتروني (regmap)

---

### الـ Core Abstraction: المفهوم المحوري

المفهوم المحوري في الـ pinctrl framework هو **الفصل بين التعريف والتشغيل**:

```
  التعريف (static const data — compile time):
  ┌──────────────────────────────────────────────────────────┐
  │  pinctrl_pin_desc[]  ← "الـ pin ده اسمه GPIOX_8"       │
  │  meson_pmx_group[]   ← "uart_tx_a يحتاج GPIOX_8"       │
  │  meson_pmx_func[]    ← "uart_a وظيفة، تحتاج uart_tx_a" │
  │  meson_bank[]        ← "GPIOX في register 0x6 offset 0" │
  └──────────────────────────────────────────────────────────┘

  التشغيل (runtime — عند probe وعند طلب driver):
  ┌──────────────────────────────────────────────────────────┐
  │  pinctrl_register_and_init() ← يسجل كل ده في الـ core  │
  │  set_mux()  ← يكتب في الـ register الصح قيمة الـ func  │
  │  gpio_request() ← يحجز pin ويكتب function=0 (GPIO mode) │
  └──────────────────────────────────────────────────────────┘
```

الجمال هنا: الـ driver لـ Meson AXG **ما بيكتبش منطق** — بس بيعرّف data. كل المنطق في الـ generic `meson_pinctrl_probe()` و`meson_axg_pmx_ops`.

---

### البنى الأساسية وعلاقتها ببعض

```
pinctrl_desc
├── pins[]  →  pinctrl_pin_desc[]
│              { number=GPIOX_8, name="GPIOX_8" }
│
├── pctlops →  pinctrl_ops
│              get_groups_count / get_group_pins / dt_node_to_map
│
└── pmxops  →  pinmux_ops  (= meson_axg_pmx_ops)
               set_mux() / gpio_request_enable()


meson_pinctrl_data
├── pins[]    → pinctrl_pin_desc[]   (نفس الـ array)
│
├── groups[]  → meson_pmx_group[]
│   ├── name = "uart_tx_a"
│   ├── pins = { GPIOX_8 }
│   └── data → meson_pmx_axg_data { func = 1 }
│               ↑ ده رقم الـ function في الـ mux register
│
├── funcs[]   → meson_pmx_func[]
│   ├── name = "uart_a"
│   └── groups = { "uart_tx_a", "uart_rx_a", "uart_cts_a", "uart_rts_a" }
│
├── banks[]   → meson_bank[]
│   ├── name = "X"
│   ├── first = GPIOX_0,  last = GPIOX_22
│   └── regs[]
│       ├── [MESON_REG_PULLEN] = { reg=2, bit=0 }
│       ├── [MESON_REG_PULL]   = { reg=2, bit=0 }
│       ├── [MESON_REG_DIR]    = { reg=6, bit=0 }
│       ├── [MESON_REG_OUT]    = { reg=7, bit=0 }
│       └── [MESON_REG_IN]     = { reg=8, bit=0 }
│
└── pmx_data  → meson_axg_pmx_data
    └── pmx_banks[] → meson_pmx_bank[]
        ├── name = "X"
        ├── first = GPIOX_0,  last = GPIOX_22
        ├── reg = 0x4          ← base register للـ mux
        └── offset = 0         ← bit offset
```

---

### الـ Bank: وحدة التحكم في الـ GPIO

الـ **`meson_bank`** هو أهم بنية للـ GPIO control. كل bank هو مجموعة pins متتالية بتتحكم فيها عبر نفس مجموعة الـ registers. مثال من الكود:

```c
/* من pinctrl-meson-axg.c السطر 1011 */
BANK("X", GPIOX_0, GPIOX_22, 61, 83,
/*        pullen_reg  pullen_bit  pull_reg  pull_bit */
          2,          0,          2,        0,
/*        dir_reg     dir_bit     out_reg   out_bit */
          6,          0,          7,        0,
/*        in_reg      in_bit */
          8,          0),
```

ده معناه:
- GPIOX هو bank من pin 61 (GPIOX_0) لـ pin 83 (GPIOX_22)
- **PULLEN** (تفعيل الـ pull resistor): register 2، من bit 0
- **PULL** (up أو down): register 2، من bit 0 — bit مختلف لكن نفس الـ reg offset
- **DIR** (اتجاه input/output): register 6، من bit 0
- **OUT** (قيمة الإخراج): register 7، من bit 0
- **IN** (قراءة القيمة): register 8، من bit 0

لما الـ core يحتاج يحدد اتجاه GPIOX_5 مثلاً:
```
pin_index = GPIOX_5 - GPIOX_0 = 5
actual_bit = dir_bit + pin_index = 0 + 5 = 5
regmap_write(reg_gpio, reg=6, bit=5, value=OUTPUT)
```

---

### الـ PMX Bank: وحدة التحكم في الـ Mux

الـ **`meson_pmx_bank`** بيحدد الـ mux registers — مختلفة تماماً عن الـ GPIO registers:

```c
/* من pinctrl-meson-axg.c السطر 1025 */
BANK_PMX("X", GPIOX_0, GPIOX_22, 0x4, 0),
```

- كل pin في bank X بيتحكم فيه بـ **4 bits** في register 0x4
- الـ function number من `meson_pmx_axg_data.func` بيتكتب في الـ 4 bits دول
- مثلاً `GROUP(uart_tx_a, 1)` معناها: لما تفعّل uart_tx_a اكتب `1` في الـ 4 bits الخاصة بالـ pin ده

```
Register 0x4 layout (GPIOX mux):
 ┌────┬────┬────┬────┬────┬────┬─────
 │ X22│ X21│ X20│ ... │ X1 │ X0 │
 │4bit│4bit│4bit│    │4bit│4bit│
 └────┴────┴────┴────┴────┴────┴─────

لما GPIOX_8 = uart_tx_a (func=1):
bits [35:32] = 0001
```

---

### الـ Function كـ Pin Sharing

نقطة مهمة: نفس الـ physical function ممكن يتنفذ على **groups مختلفة**. مثال من الكود:

```c
/* spi1 ممكن يشتغل على GPIOX أو GPIOA */
static const char * const spi1_groups[] = {
    "spi1_clk_x", "spi1_mosi_x", "spi1_miso_x", "spi1_ss0_x",  /* على X */
    "spi1_clk_a", "spi1_mosi_a", "spi1_miso_a", "spi1_ss0_a",  /* على A */
    "spi1_ss1"
};
```

الـ device tree بيحدد أي group تستخدم:
```dts
spi1 {
    pinctrl-0 = <&spi1_x_pins>;  /* استخدم الـ group على GPIOX */
};
```

---

### ملكية الـ Subsystem مقابل التفويض للـ Driver

| الـ Pinctrl Core يملك | الـ Driver (meson) يتولى |
|---|---|
| Global pin ownership و arbitration | تعريف الـ pins وأسمائها (`meson_axg_periphs_pins[]`) |
| منع conflict بين consumers | تعريف الـ groups وأي pins فيها |
| Device tree parsing (pinctrl-state) | تعريف الـ functions وأي groups تحتاجها |
| Debugfs interface | حساب الـ register offset لكل pin في كل bank |
| suspend/resume لـ pin states | الكتابة الفعلية في hardware عبر `set_mux()` |
| ربط الـ GPIO subsystem بالـ pinctrl | إدارة الـ regmaps (mux, pull, dir, out, in) |
| نشر consumer API (`pinctrl_get/select_state`) | الـ `parse_dt` الخاص بـ AO domain |

---

### مسار تنفيذ `set_mux()` — من الـ Driver Request للـ Hardware

```
Consumer Driver (e.g., UART driver)
    │
    │ pinctrl_select_state(pinctrl, "default")
    ▼
Pinctrl Core
    │ يبحث عن الـ function + group من الـ DT mapping
    │ يتأكد إن الـ pins مش محجوزة
    │
    │ pmxops->set_mux(pctldev, func_selector=uart_a, group_selector=uart_tx_a)
    ▼
meson_axg_pmx_ops.set_mux()  [pinctrl-meson-axg-pmx.c]
    │ بيجيب meson_pmx_axg_data.func = 1  (من group data)
    │ بيحسب الـ pin رقمه في الـ pmx_bank
    │ offset = (pin - bank->first) * 4
    │
    │ regmap_update_bits(reg_mux, bank->reg, mask, func_value)
    ▼
Hardware MUX Register
    [GPIOX_8 bits] = 0001  →  UART_A TX mode
```

---

### الـ GPIO كـ Function خاصة

الـ GPIO ما هوش حاجة خارج الـ pinctrl — هو **function رقم 0** لكل pin. لما الـ gpio_chip يطلب pin:

```c
/* GPIO_GROUP macro */
#define GPIO_GROUP(gpio)
    .data = (const struct meson_pmx_axg_data[]){
        PMX_DATA(0),   /* func = 0 دايماً للـ GPIO */
    }
```

الـ `gpio_chip` نفسه **embedded** في `meson_pinctrl` struct:

```c
struct meson_pinctrl {
    struct pinctrl_dev *pcdev;   /* handle للـ pinctrl core */
    struct regmap *reg_mux;      /* mux registers */
    struct regmap *reg_gpio;     /* gpio dir/in/out registers */
    struct gpio_chip chip;       /* ← الـ GPIO chip embedded هنا */
    ...
};
```

ده معناه: الـ GPIO subsystem (اللي هو subsystem تاني — بيدير الـ `/dev/gpiochip*` وبيوفر `gpiod_get()` API للـ drivers) بيشتغل مع الـ pinctrl مباشرة عبر `gpio_chip` المدمج.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Flags والـ Config Options — Cheatsheet

#### enum meson_reg_type
بتحدد نوع الـ register اللي بيتحكم في pin معين:

| القيمة | الوظيفة |
|---|---|
| `MESON_REG_PULLEN` | تفعيل/تعطيل الـ pull resistor |
| `MESON_REG_PULL` | اتجاه الـ pull (up/down) |
| `MESON_REG_DIR` | اتجاه الـ pin (input/output) |
| `MESON_REG_OUT` | كتابة قيمة output |
| `MESON_REG_IN` | قراءة قيمة input |
| `MESON_REG_DS` | drive strength |
| `MESON_NUM_REG` | عدد الأنواع (sentinel) |

#### enum meson_pinconf_drv
قيم الـ drive strength المتاحة:

| القيمة | التيار |
|---|---|
| `MESON_PINCONF_DRV_500UA` | 500 µA |
| `MESON_PINCONF_DRV_2500UA` | 2500 µA |
| `MESON_PINCONF_DRV_3000UA` | 3000 µA |
| `MESON_PINCONF_DRV_4000UA` | 4000 µA |

#### مناطق الـ GPIO Banks — AXG SoC

| Bank | أول pin | آخر pin | عدد الـ Pins | الاستخدام الرئيسي |
|---|---|---|---|---|
| Z | GPIOZ_0 | GPIOZ_10 | 11 | SPI0, I2C0/1, UART-B, PWM, SPDIF |
| BOOT | BOOT_0 | BOOT_14 | 15 | eMMC, NAND, NOR flash |
| A | GPIOA_0 | GPIOA_20 | 21 | SPI1, I2C2/3, TDM, PDM, SPDIF, MCLK |
| X | GPIOX_0 | GPIOX_22 | 23 | SDIO, SPI1, UART-A/B, JTAG, ETH, PWM, TDM |
| Y | GPIOY_0 | GPIOY_15 | 16 | Ethernet RGMII |
| AO | GPIOAO_0 | GPIOAO_13 | 14 | UART-AO, I2C-AO, PWM-AO, IR, JTAG-AO |

#### قيم الـ Function Selector (PMX)
الرقم اللي بيتحدد في `GROUP(name, N)` هو رقم الـ mux function:

| الرقم | المعنى |
|---|---|
| 0 | GPIO (default) |
| 1 | Function 1 (primary alternate) |
| 2 | Function 2 (secondary alternate) |
| 3 | Function 3 |
| 4 | Function 4 |

---

### 1. كل الـ Structs المهمة

#### struct pinctrl_pin_desc
```c
// من include/linux/pinctrl/pinctrl.h — تمثل pin واحد بالاسم والرقم
struct pinctrl_pin_desc {
    unsigned number;       // رقم الـ pin globally unique
    const char *name;      // اسمه مثل "GPIOZ_0"
    void *drv_data;        // بيانات خاصة بالدرايفر (مش مستخدمة هنا)
};
```
**الاستخدام في AXG:** يتبنى عبر ماكرو `MESON_PIN(x)` → `PINCTRL_PIN(x, #x)`. في الملف في مصفوفتين: `meson_axg_periphs_pins[]` (100 pin) و`meson_axg_aobus_pins[]` (15 pin).

---

#### struct meson_pmx_group
```c
struct meson_pmx_group {
    const char *name;            // اسم الـ group مثل "spi0_clk"
    const unsigned int *pins;    // مصفوفة أرقام الـ pins في الـ group
    unsigned int num_pins;       // عدد الـ pins
    const void *data;            // pointer لـ meson_pmx_axg_data (function selector)
};
```
**الهدف:** تجميع الـ pins اللي بتشتغل مع بعض لتفعيل function معين. مثلاً `spi0_clk` هو group من pin واحد (GPIOZ_0) ومربوط بـ function رقم 1.

الـ `data` field بيشاور على:
```c
struct meson_pmx_axg_data {
    unsigned int func;  // رقم الـ mux function (1-4)
};
```

---

#### struct meson_pmx_func
```c
struct meson_pmx_func {
    const char *name;               // اسم الـ function مثل "spi0"
    const char * const *groups;     // أسماء الـ groups المرتبطة
    unsigned int num_groups;        // عددها
};
```
**الهدف:** تجميع كل الـ groups اللي بتكوّن function كاملة. مثلاً function `spi0` بتشمل groups: `spi0_clk`, `spi0_mosi`, `spi0_miso`, `spi0_ss0`, `spi0_ss1`, `spi0_ss2`. يتبنى عبر ماكرو `FUNCTION(fn)`.

---

#### struct meson_reg_desc
```c
struct meson_reg_desc {
    unsigned int reg;   // offset الـ register في الـ regmap
    unsigned int bit;   // رقم الـ bit داخل الـ register
};
```
**الهدف:** وصف موقع تحكم pin واحد في register معين. كل bank عنده مصفوفة من `MESON_NUM_REG` وصف.

---

#### struct meson_bank
```c
struct meson_bank {
    const char *name;                      // اسم الـ bank مثل "Z"
    unsigned int first;                    // أول pin في الـ bank (رقم)
    unsigned int last;                     // آخر pin
    int irq_first;                         // أول hwirq number
    int irq_last;                          // آخر hwirq number
    struct meson_reg_desc regs[MESON_NUM_REG]; // وصف registers لكل نوع تحكم
};
```
**الهدف:** يصف bank كامل من الـ GPIO. كل pin في الـ bank متحكم فيه بنفس الـ registers (بـ bit offset مختلف). مثلاً bank "Z":
```c
BANK("Z", GPIOZ_0, GPIOZ_10, 14, 24,
     3, 0,   // pullen: reg=3, bit=0
     3, 0,   // pull:   reg=3, bit=0
     9, 0,   // dir:    reg=9, bit=0
     10, 0,  // out:    reg=10, bit=0
     11, 0)  // in:     reg=11, bit=0
```

---

#### struct meson_pmx_bank
```c
struct meson_pmx_bank {
    const char *name;    // اسم الـ bank
    unsigned int first;  // أول pin
    unsigned int last;   // آخر pin
    unsigned int reg;    // register الـ mux لهذا الـ bank
    unsigned int offset; // bit offset للـ pin الأول
};
```
**الهدف:** يصف الـ register المسؤول عن مستوى الـ pinmux (اختيار الـ function) لكل pin في الـ bank. مثلاً bank "Z" → `reg=0x2, offset=0`.

---

#### struct meson_axg_pmx_data
```c
struct meson_axg_pmx_data {
    const struct meson_pmx_bank *pmx_banks;   // مصفوفة الـ pmx banks
    unsigned int num_pmx_banks;               // عددها
};
```
**الهدف:** تجميع كل الـ pmx banks لمنطقة واحدة (periphs أو aobus) في struct واحد يتمرر للـ `meson_pinctrl_data`.

---

#### struct meson_pinctrl_data
```c
struct meson_pinctrl_data {
    const char *name;                          // "periphs-banks" أو "aobus-banks"
    const struct pinctrl_pin_desc *pins;       // مصفوفة الـ pins
    const struct meson_pmx_group *groups;      // مصفوفة الـ groups
    const struct meson_pmx_func *funcs;        // مصفوفة الـ functions
    unsigned int num_pins;
    unsigned int num_groups;
    unsigned int num_funcs;
    const struct meson_bank *banks;            // مصفوفة الـ GPIO banks
    unsigned int num_banks;
    const struct pinmux_ops *pmx_ops;          // pointer لـ ops table الـ mux
    const void *pmx_data;                      // pointer لـ meson_axg_pmx_data
    int (*parse_dt)(struct meson_pinctrl *pc); // callback خاص بـ DT parsing
};
```
**الهدف:** الـ "manifest" الكامل للـ pinctrl controller. الـ `of_device_id` بيشاور عليه مباشرة — لما الـ kernel يعمل match مع الـ DT compatible string، بياخد هذا الـ struct وبيمرره للـ probe function.

---

#### struct meson_pinctrl
```c
struct meson_pinctrl {
    struct device *dev;              // الـ platform device
    struct pinctrl_dev *pcdev;       // الـ pinctrl subsystem handle
    struct pinctrl_desc desc;        // وصف الـ controller للـ subsystem
    struct meson_pinctrl_data *data; // الـ static data (pins/groups/funcs/banks)
    struct regmap *reg_mux;          // regmap للـ mux registers
    struct regmap *reg_pullen;       // regmap للـ pull-enable registers
    struct regmap *reg_pull;         // regmap للـ pull direction registers
    struct regmap *reg_gpio;         // regmap للـ GPIO (dir/in/out) registers
    struct regmap *reg_ds;           // regmap للـ drive-strength registers
    struct gpio_chip chip;           // الـ GPIO chip للـ gpiolib
    struct fwnode_handle *fwnode;    // firmware node للـ DT
};
```
**الهدف:** الـ runtime state الكامل لـ instance واحد من الـ controller. بيتبنى في الـ heap أثناء الـ probe.

---

### 2. مخطط علاقات الـ Structs

```
of_device_id (DT compatible)
    │
    └──► meson_pinctrl_data  (static const, defined per SoC variant)
              │
              ├── pins[]        ──► pinctrl_pin_desc[]
              │                         └── { number, name }
              │
              ├── groups[]      ──► meson_pmx_group[]
              │                         ├── pins[]  ──► unsigned int[]
              │                         └── data    ──► meson_pmx_axg_data
              │                                              └── func (1-4)
              │
              ├── funcs[]       ──► meson_pmx_func[]
              │                         └── groups[]  ──► char*[] (group names)
              │
              ├── banks[]       ──► meson_bank[]
              │                         └── regs[MESON_NUM_REG]
              │                                  └── meson_reg_desc { reg, bit }
              │
              ├── pmx_ops       ──► pinmux_ops (meson_axg_pmx_ops)
              │
              └── pmx_data      ──► meson_axg_pmx_data
                                        └── pmx_banks[]  ──► meson_pmx_bank[]
                                                 └── { name, first, last, reg, offset }


meson_pinctrl  (runtime instance, allocated in probe)
    ├── dev          ──► platform_device
    ├── pcdev        ──► pinctrl_dev  (subsystem handle)
    ├── desc         ──► pinctrl_desc (registered with subsystem)
    ├── data         ──► meson_pinctrl_data  (points to static data above)
    ├── reg_mux      ──► regmap  (mmio: mux function select)
    ├── reg_pullen   ──► regmap  (mmio: pull-enable bits)
    ├── reg_pull     ──► regmap  (mmio: pull direction bits)
    ├── reg_gpio     ──► regmap  (mmio: dir/in/out bits)
    ├── reg_ds       ──► regmap  (mmio: drive-strength bits)
    ├── chip         ──► gpio_chip  (registered with gpiolib)
    └── fwnode       ──► fwnode_handle  (device tree node)
```

---

### 3. مخطط الـ Lifecycle — من الـ Boot للـ Teardown

```
Kernel Boot
    │
    ▼
platform_driver_register(&meson_axg_pinctrl_driver)
    │   ← module_platform_driver() macro
    │
    ▼
DT Matching
    │   kernel يبحث عن compatible string في DT:
    │   "amlogic,meson-axg-periphs-pinctrl"  →  meson_axg_periphs_pinctrl_data
    │   "amlogic,meson-axg-aobus-pinctrl"    →  meson_axg_aobus_pinctrl_data
    │
    ▼
meson_pinctrl_probe(pdev)   [في pinctrl-meson.c]
    │
    ├── devm_kzalloc()          → allocate struct meson_pinctrl
    ├── pc->data = match->data  → ربط الـ static data
    ├── meson_map_resources()   → map الـ MMIO regions → regmaps
    ├── parse_dt() callback     → (aobus فقط) يفحص DT nodes إضافية
    ├── pinctrl_register()      → تسجيل مع pinctrl subsystem
    │       └── يحفظ pins[], groups[], funcs[] في الـ subsystem
    └── gpiochip_add()          → تسجيل مع gpiolib
    │
    ▼
REGISTERED — جاهز لاستقبال requests

    ┌────────────────────────────────────┐
    │         Runtime Operation          │
    │                                    │
    │  Consumer driver calls:            │
    │  pinctrl_select_state()            │
    │      → pmx_set_mux()              │
    │      → pinconf_set()              │
    └────────────────────────────────────┘
    │
    ▼
Driver Unload / System Shutdown
    │
    ├── gpiochip_remove()
    ├── pinctrl_unregister()
    └── devm_* → automatic resource release
```

---

### 4. مخطط Call Flow — تفعيل Pinmux Function

**سيناريو:** درايفر SPI بده يفعّل الـ pins على GPIOZ لـ SPI0.

```
SPI driver probe
    │
    └── devm_pinctrl_get_select_default(dev)
            │
            ▼
        pinctrl_select_state(pctldev, state)
            │
            ▼
        pinmux_enable_setting()
            │
            ├── pmx_ops->set_mux()
            │       │   [meson_axg_pmx_ops.set_mux]
            │       │
            │       ▼
            │   meson_axg_pmx_set_mux(pctldev, func_selector, group_selector)
            │       │
            │       ├── lookup group in pc->data->groups[group_selector]
            │       │       → gets meson_pmx_group { pins, data->func }
            │       │
            │       ├── find meson_pmx_bank for the pin range
            │       │       → gets { reg, offset }
            │       │
            │       └── regmap_update_bits(pc->reg_mux, reg, mask, val)
            │               │
            │               ▼
            │           MMIO write → hardware MUX register
            │           (e.g., reg=0x2, bits[1:0] = 0b01 for func=1)
            │
            └── pinconf_ops->pin_config_set()  (if pull/drive configured)
                    │
                    ▼
                meson_pinconf_set()
                    │
                    ├── find meson_bank for pin
                    ├── regmap_update_bits(pc->reg_pullen, ...)
                    ├── regmap_update_bits(pc->reg_pull, ...)
                    └── regmap_update_bits(pc->reg_ds, ...)
```

---

### 5. مخطط Call Flow — قراءة/كتابة GPIO

```
Consumer calls gpio_set_value(gpio_num, val)
    │
    ▼
gpiolib dispatch → gpio_chip ops
    │
    ▼
meson_gpio_set(chip, offset, value)
    │
    ├── meson_calc_reg_and_bit(pc, offset, MESON_REG_OUT, &reg, &bit)
    │       │
    │       ├── find bank: pc->data->banks[i] where first<=offset<=last
    │       ├── pin_index = offset - bank->first
    │       ├── reg = bank->regs[MESON_REG_OUT].reg + pin_index/32
    │       └── bit = bank->regs[MESON_REG_OUT].bit + pin_index%32
    │
    └── regmap_update_bits(pc->reg_gpio, reg, BIT(bit), value ? BIT(bit) : 0)
            │
            ▼
        MMIO write → GPIO OUT register


Consumer calls gpio_get_value(gpio_num)
    │
    ▼
meson_gpio_get(chip, offset)
    │
    ├── meson_calc_reg_and_bit(pc, offset, MESON_REG_IN, &reg, &bit)
    └── regmap_read(pc->reg_gpio, reg, &val)
            └── return (val >> bit) & 1
```

---

### 6. مخطط Call Flow — الـ IRQ (GPIO Interrupt)

```
Request IRQ on GPIO pin
    │
    ▼
irq_domain_xlate()
    │   maps DT interrupt spec → hwirq number
    │   hwirq = irq_first + (pin - bank->first)
    │
    ▼
meson_gpio_to_irq() / irq_chip ops
    │
    ├── تفعيل الـ interrupt في الـ parent interrupt controller
    │   (GIC أو AO interrupt controller)
    │
    └── GPIOINT hardware monitors the pin state
```

---

### 7. استراتيجية الـ Locking

الـ file ده نفسه ما بيعمل locking صريح — كل الـ locking بيتعمل في طبقات تانية:

| الطبقة | آلية الـ Lock | بتحمي إيه |
|---|---|---|
| **pinctrl subsystem** | `mutex` في `pinctrl_dev` | تعديل الـ pin states (select/deselect) |
| **regmap** | internal `spinlock` أو `mutex` (حسب الـ config) | read/write على الـ MMIO registers |
| **gpiolib** | `gpio_chip->bgpio_lock` (spinlock) | عمليات الـ GPIO (get/set/direction) |
| **irq_chip** | `irq_desc` lock في الـ IRQ subsystem | تفعيل/تعطيل الـ interrupts |

**ترتيب الـ locks (Lock Ordering):**
```
pinctrl mutex
    └── regmap mutex/spinlock
            └── (لا يوجد nested locks بعدها)

gpiolib spinlock  (مستقل عن pinctrl mutex — لا تستخدمهم مع بعض)
```

**ملاحظة مهمة:** الـ `meson_pinctrl_data` structs كلها `const static` — بيانات read-only مش محتاجة locking. الـ runtime state الوحيد في `struct meson_pinctrl` اللي بيتبنى مرة واحدة في الـ probe وبعدين بيتحمى بـ regmap.

---

### 8. العلاقة بين الـ Domain الفيزيائي والـ Data Structures

```
Hardware AXG SoC
┌─────────────────────────────────────────────────────┐
│  Peripherals Domain (EE - Everything Else)          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│  │ Bank Z   │ │ Bank BOOT│ │ Bank A   │  ...        │
│  │ 11 pins  │ │ 15 pins  │ │ 21 pins  │            │
│  └──────────┘ └──────────┘ └──────────┘            │
│  reg_mux    → /pinmux registers/                    │
│  reg_gpio   → /GPIO dir/in/out registers/           │
│  reg_pull   → /pull registers/                      │
└─────────────────────────────────────────────────────┘
         ↕ mapped by DT: "amlogic,meson-axg-periphs-pinctrl"
┌─────────────────────────────────────────────────────┐
│  meson_axg_periphs_pinctrl_data                     │
│    pins[]   → 100 pinctrl_pin_desc                  │
│    groups[] → ~200 meson_pmx_group                  │
│    funcs[]  → 29 meson_pmx_func                     │
│    banks[]  → 5 meson_bank (Z,BOOT,A,X,Y)           │
│    pmx_data → 5 meson_pmx_bank                      │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  Always-On Domain (AO)                              │
│  ┌──────────┐                                       │
│  │ Bank AO  │  14 pins  (+ GPIO_TEST_N)             │
│  └──────────┘                                       │
│  power domain يظل شغال حتى في الـ suspend          │
└─────────────────────────────────────────────────────┘
         ↕ mapped by DT: "amlogic,meson-axg-aobus-pinctrl"
┌─────────────────────────────────────────────────────┐
│  meson_axg_aobus_pinctrl_data                       │
│    pins[]   → 15 pinctrl_pin_desc                   │
│    groups[] → ~40 meson_pmx_group                   │
│    funcs[]  → 13 meson_pmx_func                     │
│    banks[]  → 1 meson_bank (AO)                     │
│    parse_dt → meson8_aobus_parse_dt_extra()         │
└─────────────────────────────────────────────────────┘
```
## Phase 4: شرح الـ Functions

### ملخص سريع — Cheatsheet

#### الـ Macros والـ Static Data Constructors

| الـ Macro / Initializer | الغرض | الملف |
|---|---|---|
| `MESON_PIN(x)` | بيعمل `pinctrl_pin_desc` من اسم الـ pin | `pinctrl-meson.h` |
| `BANK(n,f,l,fi,li,...)` | بيبني `meson_bank` بكل الـ register descriptors | `pinctrl-meson.h` |
| `BANK_PMX(n,f,l,r,o)` | بيبني `meson_pmx_bank` لتحديد موقع الـ mux register | `pinctrl-meson-axg-pmx.h` |
| `GROUP(grp,f)` | بيبني `meson_pmx_group` لـ function group مع function index | `pinctrl-meson-axg-pmx.h` |
| `GPIO_GROUP(gpio)` | بيبني `meson_pmx_group` لـ GPIO mode (func=0) | `pinctrl-meson-axg-pmx.h` |
| `FUNCTION(fn)` | بيبني `meson_pmx_func` ويربطها بالـ groups array | `pinctrl-meson.h` |
| `PMX_DATA(f)` | بيبني `meson_pmx_axg_data` بالـ function index | `pinctrl-meson-axg-pmx.h` |

#### الـ Runtime Functions (من pinctrl-meson-axg-pmx.c)

| الـ Function | النوع | الغرض |
|---|---|---|
| `meson_axg_pmx_get_bank()` | static | بيلاقي الـ `meson_pmx_bank` اللي بيحتويه الـ pin |
| `meson_pmx_calc_reg_and_offset()` | static | بيحسب الـ register address والـ bit offset للـ pin |
| `meson_axg_pmx_update_function()` | static | بيكتب الـ function value في الـ mux register |
| `meson_axg_pmx_set_mux()` | pinmux_ops callback | بيفعّل function معينة على group من الـ pins |
| `meson_axg_pmx_request_gpio()` | pinmux_ops callback | بيرجّع الـ pin لـ GPIO mode |

#### الـ Static Data Tables (من pinctrl-meson-axg.c)

| الـ Table | النوع | الغرض |
|---|---|---|
| `meson_axg_periphs_pins[]` | `pinctrl_pin_desc[]` | كل الـ pins في الـ periphs domain |
| `meson_axg_aobus_pins[]` | `pinctrl_pin_desc[]` | كل الـ pins في الـ AO domain |
| `meson_axg_periphs_groups[]` | `meson_pmx_group[]` | GPIO groups + function groups للـ periphs |
| `meson_axg_aobus_groups[]` | `meson_pmx_group[]` | GPIO groups + function groups للـ AObus |
| `meson_axg_periphs_functions[]` | `meson_pmx_func[]` | كل الـ functions المتاحة في الـ periphs domain |
| `meson_axg_aobus_functions[]` | `meson_pmx_func[]` | كل الـ functions المتاحة في الـ AO domain |
| `meson_axg_periphs_banks[]` | `meson_bank[]` | bank descriptors للـ Z,BOOT,A,X,Y banks |
| `meson_axg_aobus_banks[]` | `meson_bank[]` | bank descriptor للـ AO bank |
| `meson_axg_periphs_pmx_banks[]` | `meson_pmx_bank[]` | pmx register layout للـ periphs |
| `meson_axg_aobus_pmx_banks[]` | `meson_pmx_bank[]` | pmx register layout للـ AObus |
| `meson_axg_periphs_pinctrl_data` | `meson_pinctrl_data` | descriptor كامل للـ periphs controller |
| `meson_axg_aobus_pinctrl_data` | `meson_pinctrl_data` | descriptor كامل للـ AObus controller |
| `meson_axg_pinctrl_dt_match[]` | `of_device_id[]` | الـ DT compatible strings |

---

### Category 1: الـ Static Pin Descriptor Arrays

#### الغرض العام

دي أول طبقة في الـ pinctrl subsystem — بتعرّف كل pin له اسم ورقم. الـ `MESON_PIN` macro بتبني `pinctrl_pin_desc` struct بالـ pin index كـ ID والاسم كـ string. الفايل بيعمل array منفصل لكل domain: الـ **periphs domain** (العادي) والـ **AObus domain** (الـ Always-On).

---

#### `MESON_PIN(x)` — الـ Macro

```c
#define MESON_PIN(x) PINCTRL_PIN(x, #x)
// PINCTRL_PIN(id, name) بيبني pinctrl_pin_desc { .number = id, .name = name }
```

الـ macro ده بيأخد الـ enum value (زي `GPIOZ_0`) ويبني منها `pinctrl_pin_desc` بالرقم كـ ID والاسم كـ string literal.

**المكان في الكود:**

```c
// الـ periphs pins — 86 pin في 5 banks: Z, BOOT, A, X, Y
static const struct pinctrl_pin_desc meson_axg_periphs_pins[] = {
    MESON_PIN(GPIOZ_0),   // → { .number = GPIOZ_0, .name = "GPIOZ_0" }
    MESON_PIN(GPIOZ_1),
    // ...
    MESON_PIN(GPIOY_15),
};

// الـ AObus pins — 15 pin في bank واحد: AO
static const struct pinctrl_pin_desc meson_axg_aobus_pins[] = {
    MESON_PIN(GPIOAO_0),
    // ...
    MESON_PIN(GPIO_TEST_N), // test pin خاص
};
```

**النقاط المهمة:**
- الـ `x` في `MESON_PIN(x)` هو enum value معرّف في `dt-bindings/gpio/meson-axg-gpio.h`
- الـ number بيتحدد بالـ enum، مش بالترتيب في الـ array — لازم يكونوا متتاليين بدون فجوات عشان الـ bank lookups تشتغل صح
- الـ `GPIO_TEST_N` pin خاص بالـ AO domain وبيُستخدم للـ testing

---

### Category 2: الـ Pin Group Definitions (pin arrays + group structs)

#### الغرض العام

كل function في الـ hardware بتحتاج مجموعة محددة من الـ pins. الكود بيعرّف أولاً array من الـ pin numbers لكل group، وبعدين بيستخدم الـ `GROUP` و `GPIO_GROUP` macros عشان يبني الـ `meson_pmx_group` structs.

---

#### `GPIO_GROUP(gpio)` — الـ Macro

```c
#define GPIO_GROUP(gpio)                        \
    {                                           \
        .name = #gpio,                          \
        .pins = (const unsigned int[]){ gpio }, \
        .num_pins = 1,                          \
        .data = (const struct meson_pmx_axg_data[]){ PMX_DATA(0), }, \
    }
```

بيبني group لـ pin واحد بـ `func = 0` (ده معناه GPIO mode). لما الـ pinctrl subsystem يطلب GPIO mode لأي pin، بيبعت function value 0 للـ mux register.

**الاستخدام:**
```c
// في meson_axg_periphs_groups[]
GPIO_GROUP(GPIOZ_0),  // → group اسمها "GPIOZ_0" بـ pin واحد وfunc=0
GPIO_GROUP(BOOT_0),
// ...
```

---

#### `GROUP(grp, f)` — الـ Macro

```c
#define GROUP(grp, f)                           \
    {                                           \
        .name = #grp,                           \
        .pins = grp ## _pins,                   \
        .num_pins = ARRAY_SIZE(grp ## _pins),   \
        .data = (const struct meson_pmx_axg_data[]){ PMX_DATA(f), }, \
    }
```

بيبني function group مع function index `f` (1-4). الـ `f` ده هو القيمة اللي بتتكتب في الـ 4-bit mux field في الـ register.

**مثال:**
```c
// BOOT_8 ممكن يكون emmc_clk (func=1) أو nand_ce0 (func=2)
GROUP(emmc_clk, 1),  // → { .name="emmc_clk", .pins=emmc_clk_pins, func=1 }
GROUP(nand_ce0, 2),  // → { .name="nand_ce0", .pins=nand_ce0_pins, func=2 }
```

**النقاط المهمة:**
- الـ `f` بيتراوح بين 1 و 4 في AXG — 4 bits لكل pin = 16 function ممكنة، بس عملياً 4 كافيين
- القيمة 0 محجوزة دايماً لـ GPIO mode
- الـ `grp ## _pins` بيشير لـ static array معرّفة قبل الـ groups array مباشرة
- الـ function index مش globally unique — نفس الرقم في bank مختلف معناه function مختلفة

---

#### `PMX_DATA(f)` — الـ Macro

```c
#define PMX_DATA(f) { .func = f, }
// بيبني meson_pmx_axg_data { unsigned int func; }
```

**الـ struct المرتبط:**
```c
struct meson_pmx_axg_data {
    unsigned int func; // القيمة اللي بتتكتب في الـ 4-bit mux field
};
```

ده الـ opaque `data` pointer في `meson_pmx_group`. الـ runtime code بيعمل cast له عشان يجيب الـ function number.

---

### Category 3: الـ Function Lists (function → groups mapping)

#### الغرض العام

الـ pinctrl subsystem بيشتغل بمفهوم الـ **function** اللي بتحتوي على مجموعة من الـ **groups**. كل function string array بتعدد الـ groups اللي ممكن تحقق الـ function دي.

---

#### `FUNCTION(fn)` — الـ Macro

```c
#define FUNCTION(fn)                        \
    {                                       \
        .name = #fn,                        \
        .groups = fn ## _groups,            \
        .num_groups = ARRAY_SIZE(fn ## _groups), \
    }
```

**البيانات المرتبطة:**
```c
// مثال على function groups definition
static const char * const spi1_groups[] = {
    "spi1_clk_x", "spi1_mosi_x", "spi1_miso_x", "spi1_ss0_x", // via GPIOX
    "spi1_clk_a", "spi1_mosi_a", "spi1_miso_a", "spi1_ss0_a", // via GPIOA
    "spi1_ss1"
};

// الاستخدام في meson_axg_periphs_functions[]
FUNCTION(spi1), // → { .name="spi1", .groups=spi1_groups, .num_groups=9 }
```

**الـ periphs functions (29 function):**

| الـ Function | عدد الـ Groups | الوصف |
|---|---|---|
| `gpio_periphs` | 86 | كل الـ pins كـ GPIO |
| `emmc` | 11 | eMMC boot interface |
| `nand` | 14 | NAND flash (بيشارك data pins مع eMMC) |
| `nor` | 6 | SPI NOR flash |
| `sdio` | 6 | SDIO/SD card |
| `spi0` | 6 | SPI0 (via GPIOZ) |
| `spi1` | 9 | SPI1 (via GPIOX أو GPIOA) |
| `uart_a` | 4 | UART-A (via GPIOX) |
| `uart_b` | 8 | UART-B (via GPIOZ أو GPIOX) |
| `uart_ao_b_z` | 4 | UART-AO-B (via GPIOZ — crossover خاص) |
| `i2c0..i2c3` | 2-6 لكل | I2C buses متعددة |
| `eth` | 23 | Ethernet RMII+RGMII |
| `pwm_a..pwm_vs` | 1-4 لكل | PWM channels |
| `spdif_in/out` | 4-5 لكل | S/PDIF audio |
| `jtag_ee` | 4 | JTAG للـ EE domain |
| `pdm` | 6 | PDM microphone |
| `mclk_b/c` | 1 لكل | Audio master clock |
| `tdma/b/c` | 9-12 لكل | TDM audio interface |

**الـ AObus functions (13 function):**

| الـ Function | الوصف |
|---|---|
| `gpio_aobus` | كل الـ AO pins كـ GPIO |
| `uart_ao_a/b` | UART في الـ Always-On domain |
| `i2c_ao` | I2C في الـ AO domain (3 pin pairs) |
| `i2c_ao_slave` | I2C slave mode في AO |
| `remote_input/out_ao` | IR remote receiver/transmitter |
| `pwm_ao_a..d` | PWM channels في AO domain |
| `jtag_ao` | JTAG للـ AO domain |
| `gen_clk_ee` | General purpose clock output |

---

### Category 4: الـ Bank Descriptors

#### الغرض العام

الـ **bank** هو وحدة التنظيم الأساسية في الـ Meson GPIO/pinctrl. كل bank بيمثل مجموعة pins بيتحكم فيها مجموعة واحدة ومتتالية من الـ register bits. الـ `BANK` macro بيبني الـ `meson_bank` struct.

---

#### `BANK(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib)` — الـ Macro

```c
#define BANK(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib) \
    BANK_DS(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib, 0, 0)
// BANK_DS بيبني meson_bank مع كل الـ register descriptors
```

**البارامترات:**

| البارامتر | المعنى |
|---|---|
| `n` | اسم الـ bank (string) |
| `f` | أول pin في الـ bank |
| `l` | آخر pin في الـ bank |
| `fi`, `li` | أول وآخر hwirq number |
| `per`, `peb` | pull-enable register, bit offset |
| `pr`, `pb` | pull direction register, bit offset |
| `dr`, `db` | direction register, bit offset |
| `or`, `ob` | output register, bit offset |
| `ir`, `ib` | input register, bit offset |

**مثال من الكود:**
```c
static const struct meson_bank meson_axg_periphs_banks[] = {
    // "Z"  first=GPIOZ_0  last=GPIOZ_10  irq=14..24
    //       pullen=reg3@bit0  pull=reg3@bit0  dir=reg9@bit0  out=reg10@bit0  in=reg11@bit0
    BANK("Z",    GPIOZ_0,  GPIOZ_10, 14, 24, 3, 0, 3, 0, 9,  0, 10, 0, 11, 0),
    BANK("BOOT", BOOT_0,   BOOT_14,  25, 39, 4, 0, 4, 0, 12, 0, 13, 0, 14, 0),
    BANK("A",    GPIOA_0,  GPIOA_20, 40, 60, 0, 0, 0, 0, 0,  0, 1,  0, 2,  0),
    BANK("X",    GPIOX_0,  GPIOX_22, 61, 83, 2, 0, 2, 0, 6,  0, 7,  0, 8,  0),
    BANK("Y",    GPIOY_0,  GPIOY_15, 84, 99, 1, 0, 1, 0, 3,  0, 4,  0, 5,  0),
};

static const struct meson_bank meson_axg_aobus_banks[] = {
    // AO bank خاص — الـ pull-enable register offset = 16 (في نفس الـ reg_gpio)
    BANK("AO", GPIOAO_0, GPIOAO_13, 0, 13, 0, 16, 0, 0, 0, 0, 0, 16, 1, 0),
};
```

**النقاط المهمة:**
- الـ register indices دي relative لـ regmap base الخاص بكل domain
- الـ AO bank بيستخدم `reg_gpio` لكل العمليات (pull + dir + in + out) — على عكس الـ periphs اللي بيستخدم regmaps منفصلة
- الـ `irq_first/irq_last` بيحددوا range الـ hardware IRQ numbers اللي ممكن تترتبط بالـ GPIO pins دي

---

#### `BANK_PMX(n, f, l, r, o)` — الـ Macro

```c
#define BANK_PMX(n, f, l, r, o)  \
    { .name=n, .first=f, .last=l, .reg=r, .offset=o }
```

الـ struct `meson_pmx_bank` بتعرّف موقع الـ mux registers بشكل منفصل عن الـ `meson_bank`، عشان الـ AXG generation غيّر شكل الـ mux registers.

**البيانات:**
```c
static const struct meson_pmx_bank meson_axg_periphs_pmx_banks[] = {
    // "Z"  GPIOZ_0..GPIOZ_10  mux reg base=0x2, bit offset=0
    BANK_PMX("Z",    GPIOZ_0,  GPIOZ_10, 0x2, 0),
    BANK_PMX("BOOT", BOOT_0,   BOOT_14,  0x0, 0),
    BANK_PMX("A",    GPIOA_0,  GPIOA_20, 0xb, 0),
    BANK_PMX("X",    GPIOX_0,  GPIOX_22, 0x4, 0),
    BANK_PMX("Y",    GPIOY_0,  GPIOY_15, 0x8, 0),
};
```

- الـ `reg` هو index الـ 32-bit register في الـ mux regmap (مش byte address)
- الـ `offset` هو bit offset الأولي — بيُحسب بعدين بواسطة `meson_pmx_calc_reg_and_offset()`

---

### Category 5: الـ PMX Data Aggregators

#### الغرض العام

الـ `meson_axg_pmx_data` بتجمع كل الـ pmx banks في struct واحدة بتتحط في الـ `meson_pinctrl_data`.

```c
// الـ struct التعريفية
struct meson_axg_pmx_data {
    const struct meson_pmx_bank *pmx_banks;
    unsigned int num_pmx_banks;
};
```

```c
// الـ instances في pinctrl-meson-axg.c
static const struct meson_axg_pmx_data meson_axg_periphs_pmx_banks_data = {
    .pmx_banks     = meson_axg_periphs_pmx_banks,
    .num_pmx_banks = ARRAY_SIZE(meson_axg_periphs_pmx_banks), // = 5
};

static const struct meson_axg_pmx_data meson_axg_aobus_pmx_banks_data = {
    .pmx_banks     = meson_axg_aobus_pmx_banks,
    .num_pmx_banks = ARRAY_SIZE(meson_axg_aobus_pmx_banks),   // = 1
};
```

---

### Category 6: الـ Main Controller Descriptors

#### الغرض العام

الـ `meson_pinctrl_data` هو الـ descriptor الكامل لكل controller — بيجمع كل الـ static data في struct واحدة بتتديها للـ probe function عن طريق الـ DT match.

---

#### `meson_axg_periphs_pinctrl_data` و `meson_axg_aobus_pinctrl_data`

```c
static const struct meson_pinctrl_data meson_axg_periphs_pinctrl_data = {
    .name      = "periphs-banks",
    .pins      = meson_axg_periphs_pins,       // pinctrl_pin_desc array
    .groups    = meson_axg_periphs_groups,     // meson_pmx_group array
    .funcs     = meson_axg_periphs_functions,  // meson_pmx_func array
    .banks     = meson_axg_periphs_banks,      // meson_bank array
    .num_pins  = ARRAY_SIZE(meson_axg_periphs_pins),
    .num_groups = ARRAY_SIZE(meson_axg_periphs_groups),
    .num_funcs = ARRAY_SIZE(meson_axg_periphs_functions),
    .num_banks = ARRAY_SIZE(meson_axg_periphs_banks),
    .pmx_ops   = &meson_axg_pmx_ops,              // pinmux_ops vtable
    .pmx_data  = &meson_axg_periphs_pmx_banks_data,
    // .parse_dt = NULL (بيستخدم الـ default)
};

static const struct meson_pinctrl_data meson_axg_aobus_pinctrl_data = {
    .name      = "aobus-banks",
    .pins      = meson_axg_aobus_pins,
    .groups    = meson_axg_aobus_groups,
    .funcs     = meson_axg_aobus_functions,
    .banks     = meson_axg_aobus_banks,
    .num_pins  = ARRAY_SIZE(meson_axg_aobus_pins),
    .num_groups = ARRAY_SIZE(meson_axg_aobus_groups),
    .num_funcs = ARRAY_SIZE(meson_axg_aobus_functions),
    .num_banks = ARRAY_SIZE(meson_axg_aobus_banks),
    .pmx_ops   = &meson_axg_pmx_ops,
    .pmx_data  = &meson_axg_aobus_pmx_banks_data,
    .parse_dt  = meson8_aobus_parse_dt_extra,  // AO domain بيحتاج extra DT parsing
};
```

**النقاط المهمة:**
- الـ `parse_dt` callback موجود بس في الـ AObus data — بيتعامل مع الـ AO domain اللي بيستخدم regmap مختلف
- الـ `pmx_ops` هي الـ vtable اللي بيستخدمها الـ pinctrl core للـ mux operations
- الـ struct ده بيتحط كـ `.data` في الـ `of_device_id` entry

---

### Category 7: الـ Runtime PMX Functions

دول هم الـ functions الفعلية اللي بتشغّل الـ hardware — موجودة في `pinctrl-meson-axg-pmx.c`.

---

#### `meson_axg_pmx_get_bank()`

```c
static int meson_axg_pmx_get_bank(struct meson_pinctrl *pc,
                                   unsigned int pin,
                                   const struct meson_pmx_bank **bank)
```

بيدور على الـ `meson_pmx_bank` اللي بيقع فيها الـ pin المطلوب عن طريق linear search في مصفوفة الـ pmx banks.

**البارامترات:**
- `pc` — الـ pinctrl instance الرئيسي
- `pin` — رقم الـ pin المطلوب
- `bank` — output pointer بيتملى بالـ bank المناسبة لو اتلقت

**القيمة الراجعة:**
- `0` لو اتلقت الـ bank
- `-EINVAL` لو مفيش bank بتحتوي الـ pin ده

**الـ Pseudocode:**
```
for each bank in pc->data->pmx_data->pmx_banks:
    if pin >= bank.first AND pin <= bank.last:
        *bank = &pmx_banks[i]
        return 0
return -EINVAL
```

**Who calls it:** بيتنادى من `meson_axg_pmx_update_function()` فقط. مفيش locking — الـ caller مسؤول.

---

#### `meson_pmx_calc_reg_and_offset()`

```c
static int meson_pmx_calc_reg_and_offset(const struct meson_pmx_bank *bank,
                                          unsigned int pin,
                                          unsigned int *reg,
                                          unsigned int *offset)
```

بيحسب الـ register index والـ bit offset اللي بيتحكم في الـ mux function للـ pin. الـ AXG generation بيستخدم **4 bits لكل pin** (على عكس الجيل السابق اللي كان بيستخدم single bits).

**البارامترات:**
- `bank` — الـ pmx bank اللي بيقع فيها الـ pin
- `pin` — رقم الـ pin
- `reg` — output: الـ register index (relative to mux regmap base)
- `offset` — output: bit offset داخل الـ register

**الحساب:**
```c
shift = pin - bank->first;          // index الـ pin داخل الـ bank
// كل pin بياخد 4 bits، يعني shift بالـ bits = shift << 2
total_bit_offset = bank->offset + (shift << 2);
*reg    = bank->reg + total_bit_offset / 32;  // كل 8 pins في register واحد
*offset = total_bit_offset % 32;              // bit position داخل الـ register
```

**مثال عملي:**
```
GPIOZ_0: bank.reg=0x2, bank.offset=0, pin-first=0
  → total_bits = 0 + 0*4 = 0
  → reg = 0x2 + 0/32 = 0x2, offset = 0 (bits 0..3)

GPIOZ_8: pin-first=8
  → total_bits = 0 + 8*4 = 32
  → reg = 0x2 + 32/32 = 0x3, offset = 0 (bits 0..3 في الـ register التالي)
```

**القيمة الراجعة:** دايماً 0 (مفيش error cases).

**Who calls it:** بيتنادى من `meson_axg_pmx_update_function()` بعد `meson_axg_pmx_get_bank()`.

---

#### `meson_axg_pmx_update_function()`

```c
static int meson_axg_pmx_update_function(struct meson_pinctrl *pc,
                                          unsigned int pin,
                                          unsigned int func)
```

ده القلب الفعلي للـ pinmux — بيكتب الـ function value (0-15) في الـ 4-bit field الخاص بالـ pin في الـ mux register.

**البارامترات:**
- `pc` — الـ pinctrl instance
- `pin` — رقم الـ pin
- `func` — القيمة المطلوبة (0 = GPIO, 1-4 = alternate functions)

**القيمة الراجعة:**
- `0` لو نجح
- قيمة سالبة (من `meson_axg_pmx_get_bank` أو `regmap_update_bits`) لو فشل

**الـ Pseudocode:**
```
ret = meson_axg_pmx_get_bank(pc, pin, &bank)
if ret: return ret

meson_pmx_calc_reg_and_offset(bank, pin, &reg, &offset)

// بيكتب 4 bits في الـ register
// mask = 0xF << offset (4 bits في المكان الصح)
// val  = (func & 0xF) << offset
regmap_update_bits(pc->reg_mux, reg << 2, 0xf << offset, (func & 0xf) << offset)
```

**النقاط المهمة:**
- الـ `reg << 2` تحويل من word index لـ byte address
- الـ `regmap_update_bits` بتعمل read-modify-write atomically
- الـ func بيتحكم فيه الـ `meson_pmx_axg_data.func` الموجود في كل group

**Who calls it:**
- `meson_axg_pmx_set_mux()` — لكل pin في الـ group
- `meson_axg_pmx_request_gpio()` — بـ `func=0` عشان يرجّع الـ pin لـ GPIO

---

#### `meson_axg_pmx_set_mux()` — الـ pinmux_ops callback

```c
static int meson_axg_pmx_set_mux(struct pinctrl_dev *pcdev,
                                  unsigned int func_num,
                                  unsigned int group_num)
```

ده الـ callback الرئيسي للـ pinmux — بيتنادى من الـ pinctrl core لما userspace أو driver يطلب تفعيل function معينة. بيمشي على كل الـ pins في الـ group ويسيت الـ mux function.

**البارامترات:**
- `pcdev` — الـ pinctrl device (بيتحول لـ `meson_pinctrl` عن طريق `pinctrl_dev_get_drvdata`)
- `func_num` — index الـ function المطلوبة في `pc->data->funcs[]`
- `group_num` — index الـ group المطلوب في `pc->data->groups[]`

**القيمة الراجعة:**
- `0` لو نجح
- error code لو فيه pin فشل

**الـ Pseudocode:**
```
pc = pinctrl_dev_get_drvdata(pcdev)
func  = &pc->data->funcs[func_num]
group = &pc->data->groups[group_num]
pmx_data = (meson_pmx_axg_data *) group->data

dev_dbg("enable function %s, group %s", func->name, group->name)

for i in 0..group->num_pins-1:
    ret = meson_axg_pmx_update_function(pc, group->pins[i], pmx_data->func)
    if ret: return ret  // fail-fast

return 0
```

**النقاط المهمة:**
- الـ function name في الـ log (`func->name`) بيجي من `meson_pmx_func` لكن الـ function number الفعلي اللي بيتكتب في الـ hardware بيجي من `pmx_data->func` في الـ group — مهم الفرق ده
- مفيش explicit locking هنا — الـ pinctrl core بيمسك الـ mutex قبل ما ينادي الـ callback
- الـ fail-fast approach معناه لو pin واحد فشل، الباقيين مش بيتحدثوا

**Who calls it:** الـ pinctrl core من `pin_request()` / `pinmux_enable_setting()`.

---

#### `meson_axg_pmx_request_gpio()` — الـ pinmux_ops callback

```c
static int meson_axg_pmx_request_gpio(struct pinctrl_dev *pcdev,
                                       struct pinctrl_gpio_range *range,
                                       unsigned int offset)
```

بيتنادى لما GPIO driver يطلب pin معين. ببساطة بيحط القيمة 0 في الـ mux field عشان يحول الـ pin لـ GPIO mode.

**البارامترات:**
- `pcdev` — الـ pinctrl device
- `range` — الـ GPIO range (مش مستخدم — بس موجود عشان الـ interface)
- `offset` — رقم الـ pin

**القيمة الراجعة:** نفس `meson_axg_pmx_update_function()` — 0 أو error.

```c
// التنفيذ بسيط جداً
return meson_axg_pmx_update_function(pc, offset, 0);
//                                              ↑ func=0 = GPIO mode
```

**Who calls it:** الـ GPIO subsystem عن طريق `gpiochip_request()` → `pinctrl_gpio_request()`.

---

### Category 8: الـ PMX Ops Vtable

```c
const struct pinmux_ops meson_axg_pmx_ops = {
    .set_mux              = meson_axg_pmx_set_mux,
    .get_functions_count  = meson_pmx_get_funcs_count,  // من pinctrl-meson.c
    .get_function_name    = meson_pmx_get_func_name,    // من pinctrl-meson.c
    .get_function_groups  = meson_pmx_get_groups,       // من pinctrl-meson.c
    .gpio_request_enable  = meson_axg_pmx_request_gpio,
};
EXPORT_SYMBOL_GPL(meson_axg_pmx_ops);
```

الـ vtable دي بتتربط بـ `meson_pinctrl_data.pmx_ops` وبيستخدمها الـ pinctrl core.

**الـ Shared Functions من `pinctrl-meson.c`:**

| الـ Function | الغرض |
|---|---|
| `meson_pmx_get_funcs_count()` | بيرجع `pc->data->num_funcs` |
| `meson_pmx_get_func_name()` | بيرجع `pc->data->funcs[selector].name` |
| `meson_pmx_get_groups()` | بيرجع الـ groups array والعدد لـ function معينة |

---

### Category 9: الـ Device Registration

#### `meson_axg_pinctrl_dt_match[]`

```c
static const struct of_device_id meson_axg_pinctrl_dt_match[] = {
    {
        .compatible = "amlogic,meson-axg-periphs-pinctrl",
        .data = &meson_axg_periphs_pinctrl_data,
    },
    {
        .compatible = "amlogic,meson-axg-aobus-pinctrl",
        .data = &meson_axg_aobus_pinctrl_data,
    },
    { }, // sentinel
};
MODULE_DEVICE_TABLE(of, meson_axg_pinctrl_dt_match);
```

الـ DT match table بيربط كل compatible string بالـ descriptor المناسب. الـ AXG SoC بيظهر كـ **جهازين منفصلين** في الـ device tree (periphs + aobus).

---

#### `meson_axg_pinctrl_driver` و `module_platform_driver()`

```c
static struct platform_driver meson_axg_pinctrl_driver = {
    .probe  = meson_pinctrl_probe,  // الـ probe من pinctrl-meson.c (shared)
    .driver = {
        .name           = "meson-axg-pinctrl",
        .of_match_table = meson_axg_pinctrl_dt_match,
    },
};

module_platform_driver(meson_axg_pinctrl_driver);
```

الـ `module_platform_driver()` macro بيعمل `module_init` و `module_exit` تلقائياً. الـ `probe` function هي `meson_pinctrl_probe` المشتركة من `pinctrl-meson.c` — مش محتاجين probe خاصة للـ AXG.

**الـ Probe Flow:**
```
meson_pinctrl_probe()
    ← of_device_get_match_data() → meson_axg_periphs_pinctrl_data أو aobus
    ← بيبني الـ regmaps من DT resources
    ← لو .parse_dt موجود → meson8_aobus_parse_dt_extra() للـ AO domain
    ← pinctrl_register() بيسجل الـ controller
    ← gpiochip_add_data() بيسجل الـ GPIO chip
```

---

### البنية الكاملة — ASCII Diagram

```
meson_axg_pinctrl_dt_match
        │
        ├── "amlogic,meson-axg-periphs-pinctrl"
        │       └── meson_axg_periphs_pinctrl_data
        │               ├── pins[]    ← MESON_PIN() × 86
        │               ├── groups[]  ← GPIO_GROUP() + GROUP(fn, mux_val)
        │               ├── funcs[]   ← FUNCTION() × 29
        │               ├── banks[]   ← BANK(Z,BOOT,A,X,Y)
        │               ├── pmx_ops   ← meson_axg_pmx_ops vtable
        │               └── pmx_data  ← meson_axg_periphs_pmx_banks_data
        │                               └── BANK_PMX(Z,BOOT,A,X,Y)
        │
        └── "amlogic,meson-axg-aobus-pinctrl"
                └── meson_axg_aobus_pinctrl_data
                        ├── pins[]    ← MESON_PIN() × 15
                        ├── groups[]  ← GPIO_GROUP() + GROUP()
                        ├── funcs[]   ← FUNCTION() × 13
                        ├── banks[]   ← BANK(AO)
                        ├── pmx_ops   ← meson_axg_pmx_ops vtable
                        ├── pmx_data  ← meson_axg_aobus_pmx_banks_data
                        └── parse_dt  ← meson8_aobus_parse_dt_extra()

مسار تفعيل function (مثال: spi0 على GPIOZ):
pinctrl core
    → meson_axg_pmx_set_mux(pcdev, func_num=spi0, group_num=spi0_clk)
        → meson_axg_pmx_update_function(pc, GPIOZ_0, func=1)
            → meson_axg_pmx_get_bank(pc, GPIOZ_0) → bank "Z" (reg=0x2)
            → meson_pmx_calc_reg_and_offset(bank, GPIOZ_0)
                → shift=0, reg=0x2, offset=0
            → regmap_update_bits(reg_mux, 0x8, 0xF, 0x1)
                → bits[3:0] of reg 0x8 = 0b0001 (function 1 = SPI0)
```
## Phase 5: دليل الـ Debugging الشامل

الـ driver اللي بنتكلم عنه هو `pinctrl-meson-axg.c` — الـ pin controller الخاص بـ Amlogic Meson AXG SoC. بيتحكم في مجموعتين من الـ banks: **periphs** (GPIOZ, BOOT, GPIOA, GPIOX, GPIOY) و**aobus** (GPIOAO + GPIO_TEST_N). كل pin ممكن يشتغل كـ GPIO عادي أو يتحول لـ alternate function (SPI, I2C, UART, EMMC, NAND, …) عن طريق الـ pinmux registers.

---

### Software Level

#### 1. مدخلات الـ debugfs المتعلقة بالـ Pinctrl

الـ pinctrl subsystem بيعرض معلومات تفصيلية في `/sys/kernel/debug/pinctrl/`.

```bash
# اعرف كل الـ pinctrl devices الموجودة
ls /sys/kernel/debug/pinctrl/

# النتيجة المتوقعة على AXG:
# fd4b0000.periphs-pinctrl  (أو اسم مشابه)
# ff800014.aobus-pinctrl
```

| المسار | ما يعرضه |
|--------|----------|
| `/sys/kernel/debug/pinctrl/<dev>/pins` | كل الـ pins المسجلة (GPIOZ_0 … GPIO_TEST_N) مع أرقامها |
| `/sys/kernel/debug/pinctrl/<dev>/pingroups` | كل الـ groups (emmc, spi0, i2c1_sck_z …) والـ pins التابعة لكل group |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-pins` | الـ pin المحجوز حالياً لأي function |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-functions` | الـ functions المتاحة وكل groups تبعها |
| `/sys/kernel/debug/pinctrl/<dev>/gpio-ranges` | الـ mapping بين GPIO number وأرقام الـ pins |
| `/sys/kernel/debug/gpio` | حالة كل GPIO: direction, value, requested أو لأ |

```bash
# اقرأ حالة الـ pinmux
cat /sys/kernel/debug/pinctrl/fd4b0000.periphs-pinctrl/pinmux-pins

# مثال على الناتج:
# pin 0 (GPIOZ_0): (MUX UNCLAIMED) (GPIO UNCLAIMED)
# pin 8 (GPIOX_0): device fd401000.mmc function sdio group sdio_d0
# pin 61 (GPIOX_0): device fd401000.mmc function sdio group sdio_d0

# اقرأ حالة كل الـ GPIOs
cat /sys/kernel/debug/gpio
# مثال:
# gpiochip0: GPIOs 410-494, parent: platform/fd4b0000.periphs-pinctrl, periphs-banks:
#  gpio-410 (GPIOZ_0            ) in  hi IRQ
#  gpio-418 (GPIOX_0            ) out hi [sdio-wifi-power]
```

---

#### 2. مدخلات الـ sysfs

```bash
# اعرف اسم الـ gpiochip المرتبط بـ periphs أو aobus
ls /sys/bus/platform/devices/ | grep pinctrl

# شوف الـ GPIO chip المقابل
ls /sys/class/gpio/
# gpiochip410  gpiochip495  (الأرقام بتتغير حسب compile)

# export GPIO معين للتجربة (مثلاً GPIOX_0 = offset 0 في bank X)
echo 418 > /sys/class/gpio/export
cat /sys/class/gpio/gpio418/direction   # in أو out
cat /sys/class/gpio/gpio418/value       # 0 أو 1

# اعرف الـ label والـ consumer
cat /sys/class/gpio/gpiochip410/label
cat /sys/class/gpio/gpiochip410/ngpio

# regmap-based registers عن طريق sysfs (لو مفعّل)
ls /sys/kernel/debug/regmap/
```

---

#### 3. الـ ftrace — tracepoints وأحداث الـ pinctrl

```bash
# فعّل الـ tracing
mount -t tracefs tracefs /sys/kernel/tracing

# اعرف الـ events المتاحة للـ pinctrl
ls /sys/kernel/tracing/events/gpio/
ls /sys/kernel/tracing/events/regmap/

# تتبع كل عمليات قراءة/كتابة الـ regmap (هنا بيحصل كل تغيير في الـ pinmux registers)
echo 1 > /sys/kernel/tracing/events/regmap/regmap_reg_write/enable
echo 1 > /sys/kernel/tracing/events/regmap/regmap_reg_read/enable

# تتبع كل تغييرات الـ GPIO
echo 1 > /sys/kernel/tracing/events/gpio/enable

# شغّل الـ trace وافحص
echo 1 > /sys/kernel/tracing/tracing_on
# ... افعل العملية اللي بتحاول تعملها debug ...
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace

# مثال على ناتج regmap write:
# mmc0-0:0000:  regmap_reg_write: meson-axg-periphs-pinctrl reg=6 val=0x00000001

# تتبع دالة معينة بـ function_graph
echo meson_pinctrl_probe > /sys/kernel/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
# أعد تشغيل الـ module أو trigger الـ probe
cat /sys/kernel/tracing/trace
```

---

#### 4. الـ printk والـ dynamic debug

```bash
# فعّل dynamic debug للـ pinctrl subsystem كله
echo 'module pinctrl_meson_axg +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/meson/pinctrl-meson-axg.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/meson/pinctrl-meson.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل الـ pinctrl messages
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ kernel log مباشرة
dmesg -w | grep -i pinctrl
dmesg -w | grep -i meson
dmesg -w | grep -i gpio

# رفع loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk

# في cmdline (لو بتـ debug الـ boot early):
# loglevel=8 dyndbg="file drivers/pinctrl/meson/* +p"
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| الـ Config | الوظيفة |
|-----------|---------|
| `CONFIG_DEBUG_GPIO` | يفعّل فحوصات إضافية وـ debug messages في GPIO subsystem |
| `CONFIG_PINCTRL_MESON` | أساسي لتشغيل الـ driver |
| `CONFIG_PINCTRL` | الـ pinctrl core |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | تفعيل الـ group abstraction |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | تفعيل الـ function abstraction |
| `CONFIG_DEBUG_FS` | لازم لـ `/sys/kernel/debug/pinctrl/` |
| `CONFIG_REGMAP_DEBUGFS` | يظهر الـ regmap registers في debugfs |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug()` و `dev_dbg()` dynamically |
| `CONFIG_LOCK_STAT` | يكشف spinlock contention في interrupt context |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في الـ pin request/release |
| `CONFIG_GPIO_SYSFS` | يكشف GPIOs عبر `/sys/class/gpio/` |
| `CONFIG_GPIOLIB_IRQCHIP` | لو بتـ debug الـ GPIO interrupts |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(DEBUG_GPIO|PINCTRL|REGMAP_DEBUGFS|DYNAMIC_DEBUG)'
```

---

#### 6. أدوات الـ devlink وأدوات خاصة بالـ Subsystem

الـ `pinctrl-meson-axg` بيستخدم `regmap` للوصول للـ registers. مفيش devlink support مباشر، لكن في أدوات مفيدة:

```bash
# pinctrl utility من meson-tools أو busybox (لو متاح)
# استخدم /sys/kernel/debug/pinctrl مباشرة بدل أدوات خاصة

# اقرأ كل الـ regmap registers (periphs)
cat /sys/kernel/debug/regmap/fd4b0000.periphs-pinctrl/registers
# النتيجة:
# 00: 00000000   (mux bank BOOT reg 0x0)
# 04: 00000001   (mux bank Z   reg 0x2 mapped)
# ...

# أداة gpio-utils (من libgpiod) — أحدث وأفضل من sysfs
gpiodetect          # اعرف كل GPIO chips
gpioinfo gpiochip0  # اعرف pins مع labels
gpioget gpiochip0 8 # اقرأ قيمة pin رقم 8
gpioset gpiochip0 8=1 # اكتب قيمة

# ioctl-based — اقرأ status الـ gpio عبر /dev/gpiochipN
cat /sys/class/gpio/gpiochip410/label  # periphs-banks
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة في dmesg | المعنى | الحل |
|---------------|--------|------|
| `meson-axg-pinctrl: failed to get clock` | الـ driver لم يجد الـ clock المطلوب | تحقق من DT node وـ `clocks` property |
| `meson-axg-pinctrl: can't request region for resource` | تعارض في عنوان الـ MMIO registers | تحقق من DT `reg` property وعدم تكرار الـ range |
| `pin X is not valid` | رقم الـ pin خارج نطاق الـ bank | تحقق من صحة pin number في DT |
| `could not get regulator` | الـ aobus domain يحتاج regulator غير موجود | أضف `vcc-ao-supply` في DT |
| `pin already requested` | pin محجوز من driver آخر قبل ما يحجزه driver جديد | افحص `pinmux-pins` في debugfs لمعرفة المعتدي |
| `pin does not belong to requested mux group` | محاولة تفعيل function على pin لا يدعمه | راجع `meson_axg_periphs_groups[]` في الـ source |
| `pinctrl_select_state: pinctrl state not found` | الـ DT node بيطلب `pinctrl-0` لكن الـ state مش موجود | أضف `pinctrl-0` و`pinctrl-names = "default"` في DT node |
| `failed to find pinctrl handle` | الـ device node مش linked صح للـ pinctrl node | تحقق من `pinctrl-0 = <&pcfg_xxx>` في DT |
| `regmap_write failed` | فشل الكتابة في الـ MMIO | تحقق من الـ clock والـ power domain |
| `GPIO irq not mapped` | طلب interrupt على GPIO مش عنده hwirq | راجع `irq_first/irq_last` في `BANK()` definition |

---

#### 8. أماكن استراتيجية لـ `dump_stack()` وـ `WARN_ON()`

```c
/* في pinctrl-meson.c — دالة meson_pinmux_set_mux() */
int meson_pinmux_set_mux(struct pinctrl_dev *pcdev,
                          unsigned func_selector,
                          unsigned group_selector)
{
    struct meson_pinctrl *pc = pinctrl_dev_get_drvdata(pcdev);
    const struct meson_pmx_group *group = &pc->data->groups[group_selector];

    /* DEBUG: تحقق إن الـ group data مش NULL */
    if (WARN_ON(!group->data))
        return -EINVAL;

    /* DEBUG: dump لو func_selector خارج النطاق */
    if (WARN_ON(func_selector >= pc->data->num_funcs)) {
        dump_stack();
        return -EINVAL;
    }
    ...
}

/* في meson_gpio_request() — لو pin اتطلب مرتين */
WARN_ON(test_and_set_bit(pin, pc->chip.used));
```

أماكن مهمة توضع فيها `WARN_ON()`:
- قبل كل `regmap_write()` — تحقق إن `pc->reg_mux` مش NULL
- بعد `meson_calc_reg_and_bit()` — تحقق من صحة الـ offset
- في `meson_pinctrl_probe()` — بعد كل `devm_regmap_init_mmio()`

---

### Hardware Level

#### 1. التحقق من تطابق حالة الـ Hardware مع حالة الـ Kernel

الـ AXG pinctrl بيكتب الـ function number مباشرة في الـ mux register (4 bits لكل pin):

```bash
# اقرأ الـ mux register من الـ kernel (عبر regmap debugfs)
cat /sys/kernel/debug/regmap/fd4b0000.periphs-pinctrl/registers

# Bank BOOT (EMMC): reg=0x0, bits[3:0]=function
# Bank Z:           reg=0x2 * 4 = offset 8
# Bank A:           reg=0xb * 4 = offset 44
# Bank X:           reg=0x4 * 4 = offset 16
# Bank Y:           reg=0x8 * 4 = offset 32

# قارن مع القيم في meson_axg_periphs_pmx_banks[]:
# BANK_PMX("Z",    GPIOZ_0,  GPIOZ_10, 0x2, 0)  -> reg 0x2
# BANK_PMX("BOOT", BOOT_0,   BOOT_14,  0x0, 0)  -> reg 0x0
# BANK_PMX("A",    GPIOA_0,  GPIOA_20, 0xb, 0)  -> reg 0xb
# BANK_PMX("X",    GPIOX_0,  GPIOX_22, 0x4, 0)  -> reg 0x4
# BANK_PMX("Y",    GPIOY_0,  GPIOY_15, 0x8, 0)  -> reg 0x8

# Bank AO (aobus): BANK_PMX("AO", GPIOAO_0, GPIOAO_13, 0x0, 0)
cat /sys/kernel/debug/regmap/ff800014.aobus-pinctrl/registers
```

طريقة تفسير قيمة الـ register:
- كل pin أخد 4 bits
- `0x0` = GPIO mode
- `0x1` = function 1 (مثلاً EMMC لبنك BOOT)
- `0x2` = function 2
- `0x3` = function 3
- `0x4` = function 4

```bash
# مثال: BOOT_8 (emmc_clk) لازم يكون function 1
# في مصفوفة الـ source: GROUP(emmc_clk, 1) -> function 1
# في الـ register: bits[35:32] لـ BOOT_8 = 0x1

# تحقق من GPIO direction register (MESON_REG_DIR)
# Bank X: dir reg = 6 (من BANK definition: dr=6, db=0)
# اقرأ من regmap:
cat /sys/kernel/debug/regmap/fd4b0000.periphs-pinctrl/registers | grep " 18:"
# offset 0x18 = reg 6 = DIR register for GPIOX
```

---

#### 2. تقنيات قراءة الـ Registers مباشرة من الـ Hardware

الـ AXG SoC بيشتغل على 32-bit ARM، الـ periphs base address عادةً `0xffd00000` والـ aobus `0xff800000`.

```bash
# باستخدام devmem2 (لو متاح)
# اقرأ BOOT mux register (base + reg*4)
# base address من DT: مثلاً 0xffd00000
devmem2 0xffd00000 w   # BOOT mux reg 0
devmem2 0xffd00008 w   # Z mux reg 0x2

# باستخدام /dev/mem (kernel لازم يدعم CONFIG_STRICT_DEVMEM=n)
dd if=/dev/mem bs=4 count=1 skip=$((0xffd00000/4)) 2>/dev/null | xxd

# باستخدام io (من busybox أو standalone)
io -4 0xffd00000   # قرأ 4 bytes من BOOT mux
io -4 0xffd00030   # قرأ pull-enable register for bank A

# AO bus registers
devmem2 0xff800000 w   # AO mux reg 0
devmem2 0xff800040 w   # AO pull-enable

# GPIO output register لبنك X (reg 7 = OUT, base offset = 7*4 = 0x1c)
devmem2 $((0xffd00000 + 7*4)) w
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**لـ I2C debugging (GPIOZ_6=SCL, GPIOZ_7=SDA لـ i2c0):**
- ربط الـ probe على test point قريب من الـ SoC
- تأكد من إن الـ pull-up موجود (عادةً 4.7kΩ لـ 400kHz)
- ابحث عن ACK missing = slave غير موجود أو الـ pin مش مضبوط صح

**لـ SPI debugging (GPIOZ_0=CLK, GPIOZ_1=MOSI, GPIOZ_2=MISO, GPIOZ_3=CS لـ spi0):**
- تحقق من polarity وـ phase (CPOL/CPHA) تتطابق مع الـ slave
- لو CLK مش بيظهر → الـ pinmux مش مضبوط، تحقق من function number

**لـ EMMC debugging (BOOT_0..7 = D0..D7, BOOT_8=CLK, BOOT_10=CMD):**
- استخدم oscilloscope بـ bandwidth ≥ 200MHz لـ HS200 mode
- ابحث عن eye diagram collapse = drive strength ضعيف
- الـ MESON_PINCONF_DRV_4000UA هو أقوى mode متاح للـ AXG

**نصائح عامة:**
```
+------------------+------------------+------------------------+
| الـ Signal       | الـ GPIO          | الـ Probe Point        |
+------------------+------------------+------------------------+
| EMMC CLK         | BOOT_8           | مقاومة series 22Ω     |
| I2C0 SCL         | GPIOZ_6          | نهاية pull-up         |
| SDIO CLK         | GPIOX_4          | قبل الـ connector      |
| ETH RGMII TX_CLK | GPIOY_8          | بعد الـ PHY            |
+------------------+------------------+------------------------+
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | النمط في dmesg | السبب المحتمل |
|---------|---------------|---------------|
| EMMC لا يعمل | `mmc0: error -110 whilst initialising MMC card` | الـ BOOT pins مش على function 1، أو drive strength ضعيف |
| SPI timeout | `spi-meson: transfer timed out` | GPIOZ_0..3 مش configured كـ spi0، أو الـ CS مش active low |
| I2C no ACK | `i2c i2c-0: i2c_meson: wait timeout` | GPIOZ_6/7 مش configured، أو مفيش pull-up |
| SDIO card detect fail | `mmc1: no card inserted` | GPIOX_0..5 مش على sdio function |
| ETH link down | `stmmac: no link` | GPIOY pins مش على eth function 1 |
| UART garbage | random characters أو silence | الـ TX/RX pins مقلوبين أو على function خاطئ |
| GPIO IRQ لا يطلق | لا log عند trigger | `irq_first/irq_last` في `BANK()` مش صح في DT |

---

#### 5. الـ Device Tree Debugging — تحقق من التطابق مع الـ Hardware

الـ driver بيدعم `compatible = "amlogic,meson-axg-periphs-pinctrl"` وـ `"amlogic,meson-axg-aobus-pinctrl"`.

```bash
# تحقق من الـ DT nodes الحالية
cat /proc/device-tree/soc/periphs-pinctrl/compatible
# amlogic,meson-axg-periphs-pinctrl

# اقرأ الـ reg property (base address)
hexdump -C /proc/device-tree/soc/periphs-pinctrl/reg

# افحص كل الـ pinctrl subnodes
ls /proc/device-tree/soc/periphs-pinctrl/

# تحقق من DT compiled source (بعد boot)
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "meson-axg-periphs"

# مثال DT صح لـ EMMC على AXG:
# &sd_emmc_b {
#     pinctrl-0 = <&emmc_pins>;
#     pinctrl-names = "default";
# };
#
# emmc_pins: emmc-pins {
#     mux {
#         groups = "emmc_nand_d0","emmc_nand_d1",...,"emmc_clk","emmc_cmd","emmc_ds";
#         function = "emmc";
#         drive-strength-microamp = <4000>;
#         bias-pull-up;
#     };
# };
```

**تحقق من تطابق الـ AO bus:**
```bash
# الـ aobus يحتاج parse_dt_extra = meson8_aobus_parse_dt_extra
# بيجيب الـ reg_pull من node ثاني في DT
cat /proc/device-tree/soc/aobus-pinctrl/compatible
# amlogic,meson-axg-aobus-pinctrl

# تحقق من وجود الـ reg ranges الصح (لازم يكون فيه reg entry ثاني للـ pull)
hexdump -C /proc/device-tree/soc/aobus-pinctrl/reg
```

---

### Practical Commands

#### مجموعة كاملة من الأوامر الجاهزة للنسخ

```bash
#!/bin/bash
# === Meson AXG Pinctrl Full Debug Script ===

PERIPHS_DEV=$(ls /sys/kernel/debug/pinctrl/ | grep periphs | head -1)
AOBUS_DEV=$(ls /sys/kernel/debug/pinctrl/ | grep aobus | head -1)

echo "=== [1] Pinctrl Devices ==="
ls /sys/kernel/debug/pinctrl/

echo ""
echo "=== [2] Periphs Pinmux State ==="
cat /sys/kernel/debug/pinctrl/${PERIPHS_DEV}/pinmux-pins 2>/dev/null | grep -v UNCLAIMED

echo ""
echo "=== [3] AOBUS Pinmux State ==="
cat /sys/kernel/debug/pinctrl/${AOBUS_DEV}/pinmux-pins 2>/dev/null | grep -v UNCLAIMED

echo ""
echo "=== [4] GPIO Status (requested only) ==="
cat /sys/kernel/debug/gpio | grep -v "^$"

echo ""
echo "=== [5] Periphs Register Map ==="
cat /sys/kernel/debug/regmap/${PERIPHS_DEV}/registers 2>/dev/null | head -40

echo ""
echo "=== [6] AOBUS Register Map ==="
cat /sys/kernel/debug/regmap/${AOBUS_DEV}/registers 2>/dev/null

echo ""
echo "=== [7] Pinctrl Errors in dmesg ==="
dmesg | grep -iE '(pinctrl|meson.*gpio|meson.*pin)' | tail -30

echo ""
echo "=== [8] GPIO Chip Info ==="
for chip in /sys/class/gpio/gpiochip*; do
    echo "Chip: $chip"
    echo "  Label: $(cat $chip/label)"
    echo "  Base:  $(cat $chip/base)"
    echo "  NGpio: $(cat $chip/ngpio)"
done
```

```bash
# --- تفعيل ftrace لمراقبة كل كتابة في الـ pinmux registers ---
cd /sys/kernel/tracing
echo 0 > tracing_on
echo > trace
echo 1 > events/regmap/regmap_reg_write/enable
echo 1 > tracing_on
sleep 5   # أو trigger الـ device اللي بتـ debug عليه
echo 0 > tracing_on
grep 'meson' trace | grep 'reg_write' | head -20
```

```bash
# --- تفعيل dynamic debug لكل الـ meson pinctrl files ---
echo 'file drivers/pinctrl/meson/pinctrl-meson-axg.c +pmfl' \
    > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/meson/pinctrl-meson.c +pmfl' \
    > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c +pmfl' \
    > /sys/kernel/debug/dynamic_debug/control
# أعد تشغيل الـ driver أو trigger probe
modprobe -r pinctrl-meson-axg && modprobe pinctrl-meson-axg
dmesg | tail -50
```

```bash
# --- فحص pin معين: مثلاً GPIOX_8 (uart_tx_a) ---
# GPIOX_8 = pin 61+8 = 69 في الـ global numbering (تقريبي)
# ابحث أولاً:
grep -r "GPIOX_8" /sys/kernel/debug/pinctrl/*/pins 2>/dev/null

# اقرأ حالته من الـ gpiolib
cat /sys/kernel/debug/gpio | grep -i "GPIOX_8"

# تحقق من الـ mux:
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -i "GPIOX_8"
# المتوقع لو uart_a مفعّل:
# pin 69 (GPIOX_8): device ffd23800.serial function uart_a group uart_tx_a
```

```bash
# --- اختبار GPIO output بالـ libgpiod ---
# ابحث عن رقم الـ chip الصح
gpiodetect
# gpiochip0 [periphs-banks] (85 lines)
# gpiochip1 [aobus-banks]   (15 lines)

# قرأ GPIOX_0 (line 61 في periphs-banks لو 0-indexed من أول pin في chip)
# الـ offset = pin_number - first_pin_of_chip
gpioget gpiochip0 61

# اكتب على GPIOZ_5
gpioset gpiochip0 5=1

# راقب IRQ على GPIOAO_6 (remote input)
gpiomon --num-events=10 gpiochip1 6
```

#### تفسير مثال على ناتج `pinmux-pins`

```
pin 0 (GPIOZ_0): (MUX UNCLAIMED) (GPIO UNCLAIMED)
  ^ pin free، مش محجوز لأي function أو GPIO

pin 8 (BOOT_8): device fd400000.mmc function emmc group emmc_clk
  ^ BOOT_8 محجوزة لـ emmc_clk، الـ function = emmc، الـ device = fd400000.mmc

pin 61 (GPIOX_0): device fd401000.mmc function sdio group sdio_d0
  ^ GPIOX_0 شغالة كـ sdio_d0

pin 100 (GPIOAO_0): (MUX UNCLAIMED) gpio-103 (uart-console/tx)
  ^ GPIOAO_0 مش على alternate function، لكنها محجوزة كـ GPIO عادية من uart-console
```

الأرقام في `GROUP(emmc_clk, 1)` في الـ source تعني إن الـ function value المكتوبة في الـ register هي `1` — وده يتطابق مع ناتج `pinmux-pins` لو الـ state صح.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART صامت على Amlogic A113X في Industrial Gateway

#### العنوان
**الـ debug console مش شغال على custom board بيستخدم Meson AXG SoC**

#### السياق
شركة بتطور industrial IoT gateway بتستخدم **Amlogic A113X** (وهو AXG-family SoC). البورد عندها UART للـ debug console بتتوصل على header خارجي. المنتج في مرحلة bring-up والمهندس مش شايف أي output على الـ serial terminal.

#### المشكلة
الـ bootloader شغال وبيطلع output تمام، لكن بعد ما الـ kernel يبدأ يبوت مافيش أي output. الـ UART_A هو المفروض يكون الـ console.

#### التحليل
الـ `uart_a` function في الملف معرّفه كده:

```c
static const char * const uart_a_groups[] = {
    "uart_tx_a", "uart_rx_a", "uart_cts_a", "uart_rts_a",
};
```

**الـ pins** المرتبطة:
```c
static const unsigned int uart_tx_a_pins[] = {GPIOX_8};
static const unsigned int uart_rx_a_pins[] = {GPIOX_9};
```

**الـ mux function number** للـ uart_a على bank X هو `1`:
```c
GROUP(uart_tx_a, 1),
GROUP(uart_rx_a, 1),
```

**الـ PMX bank** لـ GPIOX:
```c
BANK_PMX("X", GPIOX_0, GPIOX_22, 0x4, 0),
```

يعني الـ mux register للـ GPIOX بيبدأ من offset `0x4` في الـ regmap. الـ pin GPIOX_8 يقع في bit field داخل هذا الـ register.

المهندس اتشك على الـ DT وقلق من إن في node مش بيستخدم الـ group الصح:

```dts
/* خطأ - بيحاول يعمل uart_b بدل uart_a */
&uart_A {
    pinctrl-0 = <&uart_b_z_pins>;
    pinctrl-names = "default";
};
```

#### الحل
**أولاً**: اتحقق من الـ pinmux state الفعلي:

```bash
# شوف الـ pinctrl state اللي active
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep -i "GPIOX_8\|GPIOX_9"

# لو مفيش output، الـ mux مش اتعمل خالص
cat /sys/kernel/debug/pinctrl/periphs-banks/pins | grep "GPIOX_8"
```

**ثانياً**: صلح الـ DT ليستخدم `uart_a_pins` الصح:

```dts
/* في ملف الـ DTS للبورد */
&uart_A {
    status = "okay";
    pinctrl-0 = <&uart_a_pins>;
    pinctrl-names = "default";
};

/* التعريف الصح في amlogic-axg.dtsi */
uart_a_pins: uart-a {
    mux {
        groups = "uart_tx_a", "uart_rx_a";
        function = "uart_a";
        bias-disable;
    };
};
```

**ثالثاً**: تحقق من الـ kernel log:

```bash
dmesg | grep -i "pinctrl\|meson.*pin\|uart"
# المفروض تشوف: "meson-axg-pinctrl: periphs-banks initialized"
```

#### الدرس المستفاد
في الـ AXG pinctrl، نفس الـ peripheral (زي UART_B) ممكن يتعين على أكتر من bank (GPIOZ أو GPIOX). لازم تتأكد إن الـ DT node بيرفر لـ group الصح اللي يطابق الـ physical connection على البورد.

---

### السيناريو الثاني: تعارض SPI و I2C على GPIOZ في IoT Sensor Node

#### العنوان
**الـ SPI flash مش بيتعرف عليه والـ I2C sensor بيطلع errors متقطعة**

#### السياق
device بيستخدم **Amlogic S400 (AXG)** في IoT application. البورد عندها:
- **SPI NOR flash** على spi0 (GPIOZ_0 إلى GPIOZ_5)
- **I2C temperature sensor** على i2c0 (GPIOZ_6, GPIOZ_7)

المهندس لاحظ إن الـ SPI flash بيظهر أحياناً وأحياناً لأ، والـ I2C بيديه `-EIO` عشوائية.

#### المشكلة
في الـ DT في conflict — مرتين الـ GPIOZ_6 اتخصص لـ function مختلفة.

#### التحليل
الـ file بيعرّف الـ pins دي بشكل واضح:

```c
/* spi0 على bank Z */
static const unsigned int spi0_clk_pins[]  = {GPIOZ_0};
static const unsigned int spi0_mosi_pins[] = {GPIOZ_1};
static const unsigned int spi0_miso_pins[] = {GPIOZ_2};
static const unsigned int spi0_ss0_pins[]  = {GPIOZ_3};
static const unsigned int spi0_ss1_pins[]  = {GPIOZ_4};
static const unsigned int spi0_ss2_pins[]  = {GPIOZ_5};

/* i2c0 على bank Z */
static const unsigned int i2c0_sck_pins[]  = {GPIOZ_6};
static const unsigned int i2c0_sda_pins[]  = {GPIOZ_7};
```

لكن كمان في تعريف مشكلة في الـ DT:

```c
/* uart_ao_b_z بيستخدم نفس pins الـ i2c0! */
static const unsigned int uart_ao_cts_b_z_pins[] = {GPIOZ_6};
static const unsigned int uart_ao_rts_b_z_pins[] = {GPIOZ_7};
```

لما المهندس اتشك على الـ DT لقى:

```dts
/* خطأ: enable اتنين على نفس الـ pins */
&i2c0 {
    status = "okay";
    pinctrl-0 = <&i2c0_pins>;
};

/* ده المشكلة - بيعمل mux لـ GPIOZ_6/7 لـ UART */
&uart_AO_B_z {
    status = "okay";
    pinctrl-0 = <&uart_ao_b_z_pins>;
};
```

الـ `meson_axg_periphs_groups` array بتحتوي الاتنين:
```c
GROUP(i2c0_sck, 1),      /* GPIOZ_6, func=1 */
GROUP(i2c0_sda, 1),      /* GPIOZ_7, func=1 */
...
GROUP(uart_ao_cts_b_z, 2), /* GPIOZ_6, func=2 */
GROUP(uart_ao_rts_b_z, 2), /* GPIOZ_7, func=2 */
```

الـ pinctrl core مش بيمنع الـ conflict دا تلقائياً لو الـ driver الاتنين اتـ probe بنجاح.

#### الحل
```bash
# شوف كل الـ pin conflicts
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep "GPIOZ_6\|GPIOZ_7"
# هتشوف آخر driver عمل mux هو اللي أخد الـ pin
```

صلح الـ DT بإنك disable الـ UART اللي مش محتاجه:

```dts
/* disable الـ uart_ao_b_z لأن الـ pins محتاجاها لـ i2c0 */
&uart_AO_B_z {
    status = "disabled";
};

&i2c0 {
    status = "okay";
    pinctrl-0 = <&i2c0_pins>;
    pinctrl-names = "default";

    temp_sensor: lm75@48 {
        compatible = "national,lm75";
        reg = <0x48>;
    };
};
```

#### الدرس المستفاد
الـ AXG pinctrl بيسمح لنفس الـ physical pin يتعرّف في أكتر من group بـ functions مختلفة. الـ kernel مش بيعمل hard conflict detection تلقائياً. لازم تراجع الـ `meson_axg_periphs_groups` وتتأكد إن كل pin بيتعين لـ function واحدة بس في الـ DT.

---

### السيناريو الثالث: eMMC لمش بيبوت بعد تغيير الـ DT على Android TV Box

#### العنوان
**Android TV Box بـ Amlogic A113X مش بيبوت من eMMC بعد kernel upgrade**

#### السياق
شركة بتنتج **Android TV Box** بتستخدم **Amlogic A113X** مع eMMC storage. بعد ما عملوا kernel upgrade وغيروا في الـ DT، الجهاز وقف يبوت ومحتاجين يـ recover من USB.

#### المشكلة
الـ eMMC مش بيتعرف عليه في الـ kernel الجديد. الـ dmesg بيظهر:

```
mmc0: error -110 whilst initialising MMC card
```

#### التحليل
الـ eMMC على AXG بيستخدم **BOOT pins** حصراً:

```c
/* eMMC data lines */
static const unsigned int emmc_nand_d0_pins[] = {BOOT_0};
static const unsigned int emmc_nand_d1_pins[] = {BOOT_1};
/* ... حتى BOOT_7 */
static const unsigned int emmc_clk_pins[]     = {BOOT_8};
static const unsigned int emmc_cmd_pins[]     = {BOOT_10};
static const unsigned int emmc_ds_pins[]      = {BOOT_13};  /* Data Strobe للـ HS400 */
```

الـ PMX bank للـ BOOT:
```c
BANK_PMX("BOOT", BOOT_0, BOOT_14, 0x0, 0),
```

الـ mux function number للـ emmc هو `1`:
```c
GROUP(emmc_nand_d0, 1),
GROUP(emmc_nand_d1, 1),
/* ... */
GROUP(emmc_clk, 1),
GROUP(emmc_cmd, 1),
GROUP(emmc_ds,  1),
```

المهندس لقى إن الـ DT الجديد عنده مشكلة في الـ `emmc_ds` pin:

```dts
/* DT قديم - صح */
emmc_pins: emmc {
    mux {
        groups = "emmc_nand_d0", "emmc_nand_d1",
                 "emmc_nand_d2", "emmc_nand_d3",
                 "emmc_nand_d4", "emmc_nand_d5",
                 "emmc_nand_d6", "emmc_nand_d7",
                 "emmc_clk", "emmc_cmd", "emmc_ds";
        function = "emmc";
        bias-pull-up;
        drive-strength-microamp = <4000>;
    };
};

/* DT جديد - غلط: حذف "emmc_ds" */
emmc_pins: emmc {
    mux {
        groups = "emmc_nand_d0", ..., "emmc_clk", "emmc_cmd";
        /* emmc_ds اتنسي! */
        function = "emmc";
    };
};
```

بدون الـ `emmc_ds` (BOOT_13)، الـ eMMC controller مش قادر يشتغل بـ HS400 mode لأن الـ Data Strobe line مش متعينة صح.

الـ BANK descriptor للـ BOOT:
```c
BANK("BOOT", BOOT_0, BOOT_14, 25, 39, 4, 0, 4, 0, 12, 0, 13, 0, 14, 0),
```

يعني IRQs بتبدأ من 25 لحد 39 للـ BOOT bank.

#### الحل
```bash
# اتحقق من الـ eMMC pinmux state
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep "BOOT_"

# اتحقق من الـ mmc speed mode
cat /sys/kernel/debug/mmc0/ios
```

صلح الـ DT:
```dts
emmc_pins: emmc {
    mux {
        groups = "emmc_nand_d0", "emmc_nand_d1",
                 "emmc_nand_d2", "emmc_nand_d3",
                 "emmc_nand_d4", "emmc_nand_d5",
                 "emmc_nand_d6", "emmc_nand_d7",
                 "emmc_clk", "emmc_cmd",
                 "emmc_ds";          /* لازم يكون موجود لـ HS400 */
        function = "emmc";
        bias-pull-up;
        drive-strength-microamp = <4000>;
    };
};
```

#### الدرس المستفاد
الـ `emmc_ds` pin (BOOT_13) ضروري لـ eMMC HS400 mode. غيابه بيخلي الـ controller يـ fallback لـ mode أبطأ أو يفشل خالص حسب الـ timing. دايماً include الـ data strobe في الـ eMMC pinmux group على AXG boards.

---

### السيناريو الرابع: JTAG Debug مش شغال على Automotive ECU

#### العنوان
**الـ JTAG debugger مش بيتعرف على الـ target في مرحلة production debugging**

#### السياق
فريق بيطور **automotive ECU** بيستخدم **Amlogic AXG** كـ application processor. في مرحلة الـ validation، محتاجين يستخدموا **JTAG** لـ debugging. الـ OpenOCD مش شايف الـ target.

#### المشكلة
الـ JTAG interface مش شغال رغم إن الـ hardware connections صح. الـ OpenOCD بيطلع:

```
Error: JTAG scan chain interrogation failed
```

#### التحليل
الـ AXG عنده **اتنين JTAG interfaces** معرّفين في الـ file:

**JTAG_EE** (للـ periphs/EE domain) على GPIOX:
```c
static const unsigned int jtag_tdo_x_pins[] = {GPIOX_0};
static const unsigned int jtag_tdi_x_pins[] = {GPIOX_1};
static const unsigned int jtag_clk_x_pins[] = {GPIOX_4};
static const unsigned int jtag_tms_x_pins[] = {GPIOX_5};
```

**JTAG_AO** (للـ always-on domain) على GPIOAO:
```c
static const unsigned int jtag_ao_tdi_pins[] = {GPIOAO_3};
static const unsigned int jtag_ao_tdo_pins[] = {GPIOAO_4};
static const unsigned int jtag_ao_clk_pins[] = {GPIOAO_5};
static const unsigned int jtag_ao_tms_pins[] = {GPIOAO_7};
```

المشكلة إن الـ GPIOX_0 إلى GPIOX_5 اتخصصوا لـ **SDIO** في الـ DT:

```c
/* sdio بيستخدم GPIOX_0 لحد GPIOX_5 */
static const unsigned int sdio_d0_pins[]  = {GPIOX_0};
static const unsigned int sdio_d1_pins[]  = {GPIOX_1};
static const unsigned int sdio_d2_pins[]  = {GPIOX_2};
static const unsigned int sdio_d3_pins[]  = {GPIOX_3};
static const unsigned int sdio_clk_pins[] = {GPIOX_4};
static const unsigned int sdio_cmd_pins[] = {GPIOX_5};
```

الـ JTAG_EE بيتعارض مع الـ SDIO لأن:
- `jtag_tdo_x` = GPIOX_0 = نفس `sdio_d0`
- `jtag_clk_x` = GPIOX_4 = نفس `sdio_clk`

كمان الـ `jtag_ee_groups` بيستخدم function رقم `2`:
```c
GROUP(jtag_tdo_x, 2),
GROUP(jtag_tdi_x, 2),
GROUP(jtag_clk_x, 2),
GROUP(jtag_tms_x, 2),
```

بينما الـ SDIO function رقم `1` — يعني الـ last mux operation هو اللي بيكسب.

#### الحل
الحل الصح هو استخدام **JTAG_AO** بدل JTAG_EE، لأن الـ GPIOAO pins مش بتتعارض مع الـ SDIO:

```dts
/* صلح الـ DT */
jtag_ao_pins: jtag-ao {
    mux {
        groups = "jtag_ao_tdi", "jtag_ao_tdo",
                 "jtag_ao_clk", "jtag_ao_tms";
        function = "jtag_ao";
        bias-disable;
    };
};

/* في الـ board node */
&pinctrl_aobus {
    jtag_ao_pins_default: jtag-ao-default {
        mux {
            groups = "jtag_ao_tdi", "jtag_ao_tdo",
                     "jtag_ao_clk", "jtag_ao_tms";
            function = "jtag_ao";
        };
    };
};
```

أو لو لازم تستخدم JTAG_EE، عطل الـ SDIO أول:

```dts
/* production debug mode */
&sdio {
    status = "disabled";  /* عطل SDIO عشان تحرر GPIOX_0-5 */
};
```

```bash
# تحقق إن الـ JTAG pins فاضية
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep "GPIOX_[0-5]"
```

#### الدرس المستفاد
الـ AXG عنده JTAG_EE وJTAG_AO. الـ JTAG_EE على GPIOX بيتعارض مع SDIO لأن نفس الـ pins. في الـ production boards، استخدم JTAG_AO على GPIOAO لأنه مستقل تماماً. الـ AO domain كمان بيفضل شغال حتى لو الـ EE domain في reset.

---

### السيناريو الخامس: PDM Microphone Array مش بيشتغل على Smart Speaker

#### العنوان
**الـ PDM microphone array صامت على custom Amlogic AXG smart speaker board**

#### السياق
شركة بتطور **smart speaker** بـ 4-microphone array بتستخدم **PDM interface** على **Amlogic A113X**. الـ ALSA device بيظهر في النظام لكن الـ recording صامت تماماً.

#### المشكلة
الـ `arecord` بيرجع silence رغم إن الـ microphones موصولة صح على الـ schematic.

#### التحليل
الـ PDM interface على AXG معرّف على **GPIOA**:

```c
/* pdm pins على bank A */
static const unsigned int pdm_dclk_a14_pins[] = {GPIOA_14};
static const unsigned int pdm_dclk_a19_pins[] = {GPIOA_19};  /* alternative */
static const unsigned int pdm_din0_pins[]      = {GPIOA_15};
static const unsigned int pdm_din1_pins[]      = {GPIOA_16};
static const unsigned int pdm_din2_pins[]      = {GPIOA_17};
static const unsigned int pdm_din3_pins[]      = {GPIOA_18};
```

الـ `pdm_groups`:
```c
static const char * const pdm_groups[] = {
    "pdm_din0", "pdm_din1", "pdm_din2", "pdm_din3",
    "pdm_dclk_a14", "pdm_dclk_a19",
};
```

المشكلة إن المهندس استخدم الـ mclk pin بدل pdm_dclk في الـ DT:

```dts
/* غلط: بيستخدم mclk_c بدل pdm_dclk */
pdm_pins: pdm {
    mux {
        groups = "pdm_din0", "pdm_din1", "pdm_din2", "pdm_din3",
                 "mclk_c";       /* ده مش pdm_dclk! */
        function = "pdm";        /* مش هيشتغل - mclk_c مش جزء من pdm function */
    };
};
```

الـ `mclk_c` و `pdm_dclk_a14` بيشتركوا في نفس الـ pin (GPIOA_14) لكن كل واحد function مختلفة:

```c
/* mclk_c على GPIOA_0 - مش GPIOA_14! */
static const unsigned int mclk_c_pins[] = {GPIOA_0};
static const unsigned int mclk_b_pins[] = {GPIOA_1};

/* pdm_dclk على GPIOA_14 */
static const unsigned int pdm_dclk_a14_pins[] = {GPIOA_14};
```

الـ function number للـ pdm:
```c
GROUP(pdm_dclk_a14, 1),   /* func=1 */
GROUP(pdm_din0, 1),
GROUP(pdm_din1, 1),
GROUP(pdm_din2, 1),
GROUP(pdm_din3, 1),
```

كمان في conflict محتمل مع الـ PWM:
```c
/* pwm_a_a بيستخدم نفس GPIOA_14 */
static const unsigned int pwm_a_a_pins[] = {GPIOA_14};
GROUP(pwm_a_a, 3),  /* func=3 */
```

#### الحل
```bash
# تحقق من الـ pinmux state للـ GPIOA pins
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep "GPIOA_1[4-8]"

# اتحقق من الـ PDM clock
cat /sys/kernel/debug/clk/pdm_dclk/clk_rate 2>/dev/null || \
    cat /sys/kernel/debug/clk/pdm/clk_rate

# اتحقق من الـ ALSA controls
amixer -c 0 contents | grep -i pdm
```

صلح الـ DT:

```dts
pdm_pins: pdm-pins {
    mux {
        /* dclk الصح هو pdm_dclk_a14 على GPIOA_14 */
        groups = "pdm_dclk_a14",
                 "pdm_din0",    /* GPIOA_15 */
                 "pdm_din1",    /* GPIOA_16 */
                 "pdm_din2",    /* GPIOA_17 */
                 "pdm_din3";    /* GPIOA_18 */
        function = "pdm";
        bias-disable;
    };
};

&pdm {
    status = "okay";
    pinctrl-0 = <&pdm_pins>;
    pinctrl-names = "default";
    #sound-dai-cells = <0>;
};
```

لو محتاج الـ mclk (GPIOA_0) لتغذية الـ microphones بـ master clock:

```dts
mclk_c_pins: mclk-c {
    mux {
        groups = "mclk_c";   /* GPIOA_0 */
        function = "mclk_c";
        drive-strength-microamp = <3000>;
    };
};
```

لاحظ إن `mclk_c` (GPIOA_0) و `pdm_dclk_a14` (GPIOA_14) هم **pins مختلفة تماماً** ومش فيه conflict بينهم.

#### الدرس المستفاد
الـ GPIOA bank على AXG كثيف جداً بالـ audio functions: PDM, TDM (A/B/C), SPDIF, MCLK. لازم تفرق بين:
- `mclk_c` = GPIOA_0 (master clock output)
- `pdm_dclk_a14` = GPIOA_14 (PDM bit clock)
- `pwm_a_a` = GPIOA_14 (نفس الـ pin، function مختلفة — conflict محتمل)

دايماً راجع الـ `meson_axg_periphs_groups` array مباشرة في الـ source code عشان تتأكد من الـ pin assignment الفعلي لكل function.
## Phase 7: مصادر ومراجع

### مقدمة

الـ `pinctrl-meson-axg.c` هو driver خاص بـ Amlogic Meson AXG SoC، كتبه Xingyu Chen من Amlogic سنة 2017. المراجع التالية بتغطي كل المستويات: من الـ pinctrl subsystem نفسه، للـ Meson-specific code، للـ DT bindings.

---

### مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | المقال الأساسي اللي بيشرح فكرة الـ pinctrl subsystem وسبب إنشاؤه |
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | أول patch series لإنشاء الـ subsystem من الصفر |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | النسخة v7 من الـ subsystem قبل ما يتدمج في kernel |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | التوثيق الرسمي الأولي للـ pinmux conventions |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | إضافة الـ per-pin و per-group config interface للـ bias والـ drive |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/) | الـ generic pin config API |
| [Pinctrl driver for Amlogic Meson SoCs](https://lwn.net/Articles/616225/) | أول patch series لـ Meson pinctrl driver — البنية الأساسية اللي بيبني عليها AXG |
| [Amlogic Meson pinctrl driver](https://lwn.net/Articles/620822/) | النسخة المحدّثة من الـ Meson driver بعد المراجعة |
| [pinctrl: meson-a1: add pinctrl driver](https://lwn.net/Articles/804174/) | driver للـ A1 SoC — بيستخدم نفس الـ AXG PMX infrastructure |
| [pinctrl: meson-s4: add pinctrl driver](https://lwn.net/Articles/879912/) | driver للـ S4 SoC — يوضح كيف اتوسعت الـ AXG PMX |
| [irqchip: meson: add support for the gpio interrupt controller](https://lwn.net/Articles/703946/) | إضافة GPIO interrupt controller لـ Meson — مرتبط بالـ pinctrl data |
| [pinctrl: introduce the concept of a GPIO pin function category](https://lwn.net/Articles/1031226/) | مفهوم جديد لتصنيف الـ GPIO functions |

---

### توثيق Kernel الرسمي

#### داخل مصدر الـ Kernel

```
Documentation/driver-api/pin-control.rst   ← التوثيق الرئيسي للـ pinctrl subsystem
Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl.yaml
```

الـ `pin-control.rst` بيغطي:
- تعريف الـ `pinctrl_pin_desc` و `pinctrl_ops`
- الـ pin groups والـ functions
- الـ `pinmux_ops` و `pinconf_ops`
- الـ GPIO ranges والـ cross-referencing
- الـ device tree binding conventions

#### DT Binding للـ Meson AXG

الـ compatible strings المعرّفة في الملف:

```c
"amlogic,meson-axg-periphs-pinctrl"   /* periphs domain */
"amlogic,meson-axg-aobus-pinctrl"     /* always-on domain */
```

---

### Kernel Commits المهمة

#### إضافة الـ AXG Driver

- **[PATCH v4 2/2] ARM64: dts: meson-axg: add pinctrl DT info for Meson-AXG SoC**
  - الـ patch على mail-archive: https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1557813.html
  - بيضيف الـ DT nodes للـ AXG pinctrl

#### إعادة هيكلة الـ PMX

- **[u-boot] pinctrl: meson: rework gx pmx function**
  - Patchwork: https://patchwork.kernel.org/project/linux-amlogic/patch/1541777218-472-9-git-send-email-narmstrong@baylibre.com/
  - فصل الـ GX pmx function عن الـ common code عشان يدعم الـ AXG اللي عنده نظام PMX مختلف

#### إضافة SoC fixup callback

- **[v4,2/4] pinctrl: meson: add a new callback for SoCs fixup**
  - Patchwork: https://patchwork.kernel.org/project/linux-arm-kernel/patch/1572004167-24150-3-git-send-email-qianggui.song@amlogic.com/
  - الـ callback ده بيسمح لكل SoC يعمل fixup خاص بيه — بيستخدمه الـ AXG عبر `meson8_aobus_parse_dt_extra`

#### دعم GPIO Interrupts

- **[v6,6/9] pinctrl: meson: add support for GPIO interrupts**
  - Patchwork: https://patchwork.kernel.org/project/linux-amlogic/patch/a86d38af-9103-7557-9986-2dc187815569@gmail.com/

---

### نقاشات Mailing List

| النقاش | الرابط |
|--------|-------|
| [PATCH v2] pinctrl: meson-a1: add pinctrl driver | https://lkml.kernel.org/lkml/1570532999-23302-4-git-send-email-qianggui.song@amlogic.com/T/ |
| [v3] pinctrl: add compatible for Amlogic Meson A1 | https://patchwork.kernel.org/project/linux-arm-kernel/patch/1571050492-6598-2-git-send-email-qianggui.song@amlogic.com/ |
| arm64: dts: meson-axg: sort nodes consistently | http://www.mail-archive.com/linux-kernel@vger.kernel.org/msg1756128.html |
| [2/3] dt-bindings: pinctrl: meson-g12a: document pin name | https://patchwork.kernel.org/project/linux-amlogic/patch/20180704224511.29350-3-yixun.lan@amlogic.com/ |

---

### متابعة تطوير الـ Meson Platform

- **Linux for Amlogic Meson — Mainlining Progress:**
  https://linux-meson.com/mainlining.html
  - بيتابع حالة كل feature في kernel mainline للـ Meson SoCs

- **Kernel source على GitHub (للبحث):**
  https://github.com/torvalds/linux/blob/master/drivers/pinctrl/meson/Kconfig

---

### kernelnewbies.org — تغييرات الـ Kernel

الـ pinctrl updates بتتوثق في صفحات kernel versions على kernelnewbies:

| الإصدار | الرابط |
|---------|-------|
| Linux 6.15 | https://kernelnewbies.org/Linux_6.15 — بيذكر amlogic pinctrl driver updates |
| Linux 6.11 | https://kernelnewbies.org/Linux_6.11 — pinctrl updates متعددة |
| Linux 6.8  | https://kernelnewbies.org/Linux_6.8  — pinctrl driver additions |
| Linux 4.9  | https://kernelnewbies.org/Linux_4.9  — من أوائل الإصدارات اللي دعمت Meson pinctrl |

---

### elinux.org — مراجع عامة للـ Pinctrl

- **Introduction to pin muxing and GPIO control under Linux (ELC 2021):**
  https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf
  — شرح شامل لنظام الـ pinmux في Linux مع أمثلة عملية

- **Tests: i2c-demux-pinctrl:**
  https://www.elinux.org/Tests:i2c-demux-pinctrl
  — مثال على استخدام الـ pinctrl مع الـ i2c demux

---

### كتب مقترحة

#### Linux Device Drivers (LDD3)
- **الكتاب:** *Linux Device Drivers, 3rd Edition* — Corbet, Rubini, Kroah-Hartman
- **الفصل المهم:** Chapter 1 (Device Driver Introduction) + Chapter 6 (Advanced Char Driver Operations)
- الكتاب مجاني: https://lwn.net/Kernel/LDD3/
- ملاحظة: الكتاب قديم ومش بيغطي pinctrl subsystem مباشرة (أتضيف بعده)، بس بيغطي GPIO وبنية الـ drivers

#### Linux Kernel Development — Robert Love
- **الإصدار:** الثالث (3rd Edition)
- **الفصل المهم:** Chapter 17 (Devices and Modules)، Chapter 7 (Interrupts and Interrupt Handlers)
- بيفهّم كيف الـ platform drivers بتتسجل وبتتشغل — أساسي لفهم `module_platform_driver`

#### Embedded Linux Primer — Christopher Hallinan
- **الإصدار:** الثاني (2nd Edition)
- **الفصل المهم:** Chapter 16 (Porting Linux to a New Board)، Chapter 15 (Embedded Linux Kernel Configuration)
- بيغطي الـ pin configuration في context الـ embedded systems والـ Device Tree

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds
- **الإصدار:** الثالث (2021)
- **الفصل المهم:** Chapter 11 (Interfacing with Device Drivers)
- أحدث كتاب يغطي الـ pinctrl مع Device Tree بشكل عملي

---

### ملفات الـ Kernel المرتبطة مباشرة

```
drivers/pinctrl/meson/
├── pinctrl-meson-axg.c          ← الملف الرئيسي
├── pinctrl-meson-axg-pmx.c      ← PMX engine خاص بالـ AXG
├── pinctrl-meson-axg-pmx.h
├── pinctrl-meson.c              ← الـ common Meson pinctrl core
├── pinctrl-meson.h
├── pinctrl-meson8.c             ← مثال على SoC قديم يستخدم نفس البنية
├── pinctrl-meson-gxbb.c         ← جيل GX — سلف الـ AXG
├── pinctrl-meson-g12a.c         ← جيل G12A — يرث من AXG PMX

include/dt-bindings/gpio/
└── meson-axg-gpio.h             ← تعريفات الـ GPIO pins للـ DT
```

---

### مصطلحات البحث

لو عايز تبحث عن معلومات أكتر، استخدم:

```
"pinctrl meson axg" site:lore.kernel.org
"amlogic,meson-axg-periphs-pinctrl" site:github.com
"pinctrl-meson-axg" linux kernel
"meson_axg_pmx_ops" OR "meson_axg_pmx_banks"
"PINCTRL_MESON_AXG" Kconfig
pinctrl subsystem "pin groups" "pin functions" kernel documentation
"meson8_aobus_parse_dt_extra" amlogic
amlogic AXG S400 linux kernel bring-up
```

---

### ملخص سريع للمراجع حسب الأولوية

| الأولوية | المرجع | لماذا؟ |
|----------|--------|--------|
| ⭐⭐⭐ | [The pin control subsystem — LWN](https://lwn.net/Articles/468759/) | الأساس النظري للـ subsystem |
| ⭐⭐⭐ | [kernel.org: pin-control.rst](https://docs.kernel.org/driver-api/pin-control.html) | التوثيق الرسمي الكامل |
| ⭐⭐⭐ | [Amlogic Meson pinctrl driver — LWN](https://lwn.net/Articles/620822/) | أصل الـ Meson driver |
| ⭐⭐ | [linux-meson.com mainlining](https://linux-meson.com/mainlining.html) | حالة الـ AXG في mainline |
| ⭐⭐ | [PMX rework patch](https://patchwork.kernel.org/project/linux-amlogic/patch/1541777218-472-9-git-send-email-narmstrong@baylibre.com/) | سبب فصل الـ AXG PMX |
| ⭐⭐ | [SoC fixup callback](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1572004167-24150-3-git-send-email-qianggui.song@amlogic.com/) | فهم `parse_dt` callback |
| ⭐ | LDD3 Chapter 6 | بنية الـ drivers بشكل عام |
| ⭐ | [ELC 2021 PDF — elinux.org](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) | شرح عملي للـ pinmux |
## Phase 8: Writing simple module

### الفكرة

**الـ** `meson_pinctrl_probe` هي الدالة المُصدَّرة اللي بتعمل probe للـ platform driver الخاص بـ pinctrl في الـ Meson AXG SoC. بنستخدم **kprobe** عشان نعمل hook عليها ونطبع اسم الـ device اللي بيتم تهيئته وعدد الـ pins والـ groups — ده بيخلينا نشوف كل مرة بيتسجل فيها pinctrl controller جديد على النظام.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe module: hook meson_pinctrl_probe to log AXG pinctrl init events
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/platform_device.h> /* struct platform_device */

/*
 * Forward declaration of the internal pinctrl data struct we need.
 * We only use what's publicly known: pdev->name and pdev->id.
 */

/* ------------------------------------------------------------------ */
/* pre-handler: fires just before meson_pinctrl_probe() executes      */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86_64 the first argument (struct platform_device *pdev)
     * sits in register rdi. On ARM64 it sits in x0.
     * The kernel kprobe API gives us pt_regs; we read the first
     * argument register portably via regs_get_kernel_argument().
     */
    struct platform_device *pdev =
        (struct platform_device *)regs_get_kernel_argument(regs, 0);

    if (!pdev)
        return 0;

    /* Log the platform device name and id so we know which
     * pinctrl domain (periphs vs aobus) is being probed.      */
    pr_info("[axg-pinctrl-probe] probing device: %s (id=%d)\n",
            pdev->name ? pdev->name : "<null>",
            pdev->id);

    /* Log the device pointer itself for cross-referencing with
     * /sys/bus/platform/devices/                              */
    pr_info("[axg-pinctrl-probe] device ptr: %p, parent: %s\n",
            &pdev->dev,
            pdev->dev.parent ? dev_name(pdev->dev.parent) : "none");

    return 0; /* 0 = continue executing the real function */
}

/* ------------------------------------------------------------------ */
/* post-handler: fires after meson_pinctrl_probe() returns            */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * At this point the function has already returned.
     * On x86_64 the return value is in rax; regs->ax holds it.
     * On ARM64 it's in x0.
     * We log it to know if the probe succeeded (0) or failed (<0).
     */
    long retval = regs_return_value(regs);

    pr_info("[axg-pinctrl-probe] meson_pinctrl_probe returned: %ld (%s)\n",
            retval, retval == 0 ? "OK" : "ERROR");
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    /* Symbol name of the function we want to intercept.
     * The kernel resolves this to the real address at registration.  */
    .symbol_name = "meson_pinctrl_probe",
    .pre_handler = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/* module_init: register the kprobe                                   */
/* ------------------------------------------------------------------ */
static int __init axg_probe_hook_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[axg-pinctrl-probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[axg-pinctrl-probe] kprobe planted at %p (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit: unregister the kprobe                                 */
/* ------------------------------------------------------------------ */
static void __exit axg_probe_hook_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[axg-pinctrl-probe] kprobe removed\n");
}

module_init(axg_probe_hook_init);
module_exit(axg_probe_hook_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("AXG Pinctrl Observer");
MODULE_DESCRIPTION("kprobe hook on meson_pinctrl_probe for Meson AXG pinctrl");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية للـ kernel module زي `MODULE_LICENSE` وdeinit |
| `linux/kernel.h` | `pr_info` / `pr_err` للـ logging |
| `linux/kprobes.h` | `struct kprobe`، `register_kprobe`، `regs_get_kernel_argument` |
| `linux/platform_device.h` | `struct platform_device` اللي هو argument الدالة المستهدفة |

**الـ** `kprobes.h` هو القلب هنا — بيدي المودل القدرة يحط breakpoint ديناميكي في أي kernel symbol من غير ما يحتاج يعدّل في الكود الأصلي.

---

#### الـ `handler_pre`

ده بيتشغّل **قبل** دخول `meson_pinctrl_probe`. بنقرأ الـ argument الأول (الـ `struct platform_device *pdev`) من الـ registers باستخدام `regs_get_kernel_argument(regs, 0)` اللي هي portable عبر x86_64 وARM64 — بدل ما نقرأ `rdi` أو `x0` مباشرةً. بنطبع اسم الـ device وparent عشان نعرف أيه الـ domain (periphs أو aobus) اللي بيتم probe-ه.

---

#### الـ `handler_post`

ده بيتشغّل **بعد** ما `meson_pinctrl_probe` ترجع. بنقرأ الـ return value عبر `regs_return_value(regs)` ونطبعه — لو `0` يبقى الـ pinctrl اتسجّل صح، لو negative يبقى حصل error. ده مفيد لو الـ driver فشل يعمل `regmap` أو يحلل الـ DT.

---

#### الـ `struct kprobe kp`

```
.symbol_name = "meson_pinctrl_probe"
```

**الـ** kernel بيحوّل الاسم ده لعنوان حقيقي في الـ `.text` عند `register_kprobe`. النتيجة في `.addr` بنطبعها في الـ init عشان نتأكد إن الـ symbol اتقرأ صح.

---

#### الـ `module_init` / `module_exit`

- **الـ** `register_kprobe` بيحط الـ trap instruction في الكود ويربط الـ handlers — لازم يتعمل في الـ init قبل أي استخدام.
- **الـ** `unregister_kprobe` في الـ exit ضروري جداً: لو المودل اتنزّل من غير ما يشيل الـ kprobe، أي callback جديد هيحاول ينفّذ handler في address مش موجود ويعمل kernel panic.

---

### كيفية التجربة

```bash
# 1. ابني المودل
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# 2. حمّل المودل
insmod axg_pinctrl_hook.ko

# 3. شوف اللوج (لو الـ driver اتحمّل بعدين أو موجود في initrd)
dmesg | grep axg-pinctrl-probe

# 4. شيل المودل
rmmod axg_pinctrl_hook
```

**مثال مخرجات متوقعة:**

```
[axg-pinctrl-probe] kprobe planted at ffffffffc0123456 (meson_pinctrl_probe)
[axg-pinctrl-probe] probing device: meson-axg-pinctrl (id=-1)
[axg-pinctrl-probe] device ptr: 00000000abcd1234, parent: platform
[axg-pinctrl-probe] meson_pinctrl_probe returned: 0 (OK)
```

---

### Makefile

```makefile
obj-m += axg_pinctrl_hook.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(shell pwd) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(shell pwd) clean
```
