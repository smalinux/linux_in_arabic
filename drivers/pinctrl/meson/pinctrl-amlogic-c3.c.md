## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ subsystem ده؟

الملف ده جزء من **subsystem** اسمه **Pin Control (pinctrl)** في Linux kernel، وتحديدًا الـ driver الخاص بـ **Amlogic Meson SoCs**. في الـ `MAINTAINERS` file، الـ subsystem ده موجود تحت:

- **ARM/Amlogic Meson SoC support** — اللي بيشمل `drivers/pinctrl/meson/` كلها.
- **AMLOGIC PINCTRL DRIVER** — entry منفصل مخصوص للـ pinctrl driver الجديد.

---

### المشكلة اللي بيحلها الملف ده — بالقصة

تخيل معايا إنك عندك شريحة (SoC) — زي الـ **Amlogic C3** — فيها عدد محدود من الـ **physical pins** على الـ chip. كل pin ده عبارة عن "طرف معدني" صغير بيخرج من الشريحة.

المشكلة إن الشريحة دي فيها كتير أوي من الـ peripherals الداخلية: UART، I2C، SPI، eMMC، LCD، PWM، وغيرها. وكل peripheral محتاج pins عشان يتواصل مع العالم الخارجي. لكن الـ pins على الشريحة محدودة!

**الحل هو الـ Pin Multiplexing (pinmux):**
زي ما بتعمل في أي مبنى فيه مدخل واحد بس، بتحط حارس يوجّه كل شخص للدور المناسب. هنا الـ kernel بيلعب دور "الحارس": كل pin ممكن يشتغل في أكتر من وظيفة (function)، لكن في وقت واحد بس وظيفة واحدة.

**مثال حقيقي:**
`GPIOB_0` في الـ C3 SoC ممكن يشتغل:
- كـ `emmc_nand_d0` (بيانات eMMC/NAND)
- كـ `pwm_g_b` (إشارة PWM)
- كـ `spi_a_mosi_b` (بيانات SPI)
- كـ `uart_a_tx_b` (إرسال UART)
- كـ `lcd_d0` (بيانات LCD)
- أو كـ **GPIO** عادي

الـ kernel محتاج يعرف "احنا دلوقتي عايزين الـ `GPIOB_0` ده يشتغل إيه؟" وفقًا للـ device tree اللي المبرمج كتبه.

---

### هدف الملف ده تحديدًا

**الـ `pinctrl-amlogic-c3.c`** هو الـ **hardware description file** للـ **Amlogic C3 SoC** بس. مش فيه logic تنفيذي — ده مجرد بيانات ثابتة (static data tables) بتوصف:

1. **ايه الـ pins الموجودة** على الشريحة (55 pin موزعين على banks: A, B, C, D, E, X, TEST_N).
2. **ايه الـ functions** اللي كل pin يقدر يعملها (من GPIO عادي لـ UART، I2C، SPI، eMMC، LCD، TDM، وغيرها).
3. **ايه الـ register offsets** اللي الـ hardware بيستخدمها عشان يضبط كل ده (pull-up، direction، drive-strength، mux).

---

### الصورة الكبيرة بالأرقام

| Bank | عدد الـ pins | أمثلة الاستخدامات |
|------|-------------|-------------------|
| GPIOB | 15 (B0–B14) | eMMC، NAND، SPI Flash، LCD، UART |
| GPIOX | 14 (X0–X13) | SDIO، SPI، TDM، LCD، PWM، UART |
| GPIOC | 7 (C0–C6) | SD Card، JTAG، TDM، SPI، PWM |
| GPIOD | 7 (D0–D6) | PWM، Ethernet LED، UART، I2C، IR |
| GPIOE | 5 (E0–E4) | PWM، I2C، CLK |
| GPIOA | 6 (A0–A5) | UART، PWM، I2C، JTAG، LCD CLK |
| GPIO_TEST_N | 1 | اختبار فقط |

**عدد الـ functions المتاحة:** 46 function مختلفة (uart_a → uart_e، i2c0 → i2c3، pwm_a → pwm_n، emmc، nand، sdcard، sdio، spi_a، spi_b، spif، lcd، tdm، pdm، eth، jtag، ir، إلخ).

---

### كيف بيشتغل النظام ككل — رحلة الـ pin

```
Device Tree (.dts)                  pinctrl subsystem
┌─────────────────────┐            ┌──────────────────────────────┐
│  &uart_a {          │  probe()   │  pinctrl-meson.c             │
│    pinctrl-0 =      │ ────────►  │  (generic logic)             │
│    <&uart_a_x_pins> │            │         │                    │
│  };                 │            │         ▼                    │
└─────────────────────┘            │  pinctrl-amlogic-c3.c        │
                                   │  (C3-specific tables)        │
                                   │         │                    │
                                   │         ▼                    │
                                   │  pinctrl-meson-axg-pmx.c     │
                                   │  (register write ops)        │
                                   └──────────────────────────────┘
                                            │
                                            ▼
                                   Hardware Registers (C3 SoC)
                                   ┌──────────────────────────────┐
                                   │  MUX reg 0x03 bits [3:0]     │
                                   │  → GPIOX_0 = func2 (SPI_A)  │
                                   └──────────────────────────────┘
```

لما الـ kernel يشوف في الـ device tree إن الـ UART_A محتاج الـ `uart_a_rx_x` و`uart_a_tx_x`، بيروح للـ tables الموجودة في الملف ده، يلاقي إن `uart_a_tx_x` هو `GPIOX_1` على الـ function 6، وبعدين يكتب القيمة `6` في الـ register المناسب في الـ hardware.

---

### ليه الملف ده مش فيه أي logic؟

لأن Amlogic عملت **abstraction ممتاز**: الـ logic الكلي موجود في `pinctrl-meson.c`، وكل SoC جديد (C3، S4، G12A، AXG، etc.) بيجيب ملف C صغير فيه الـ data tables بس. ده بيخلي إضافة SoC جديد سهلة جدًا — بس تكتب tables ومش محتاج تعيد كتابة الـ logic.

---

### الملفات المرتبطة — اللي لازم تعرفها

#### الـ Core Framework
| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/meson/pinctrl-meson.c` | الـ generic logic: probe، bank lookup، GPIO ops، regmap |
| `drivers/pinctrl/meson/pinctrl-meson.h` | الـ structs الأساسية: `meson_bank`، `meson_pinctrl_data`، `meson_pmx_group`، `meson_pmx_func` |
| `drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c` | logic كتابة الـ mux registers للـ AXG وما بعده (بما فيهم C3) |
| `drivers/pinctrl/meson/pinctrl-meson-axg-pmx.h` | macros الـ `GROUP`، `GPIO_GROUP`، `BANK_PMX`، struct `meson_pmx_bank` |

#### ملفات SoCs الأخرى (للمقارنة)
| الملف | الـ SoC |
|-------|---------|
| `drivers/pinctrl/meson/pinctrl-meson-s4.c` | Amlogic S4 |
| `drivers/pinctrl/meson/pinctrl-amlogic-a4.c` | Amlogic A4 (أحدث) |
| `drivers/pinctrl/meson/pinctrl-meson-g12a.c` | Amlogic G12A |
| `drivers/pinctrl/meson/pinctrl-meson-axg.c` | Amlogic AXG |
| `drivers/pinctrl/meson/pinctrl-meson8.c` | Meson8 (قديم) |

#### الـ Device Tree Bindings
| الملف | الدور |
|-------|-------|
| `include/dt-bindings/gpio/amlogic-c3-gpio.h` | تعريف الأسماء الرقمية للـ pins (GPIOB_0 = 5، إلخ) |
| `arch/arm64/boot/dts/amlogic/` | ملفات الـ device tree للـ boards اللي بتستخدم C3 |

---

### ملخص سريع

**الـ `pinctrl-amlogic-c3.c`** هو "خريطة الأسلاك" الخاصة بـ Amlogic C3 SoC. بيقول للـ kernel: "الشريحة دي فيها الـ pins دي، وكل pin يقدر يعمل الوظايف دي، والـ registers المسؤولة هي دي." الـ logic الحقيقي موجود في `pinctrl-meson.c` اللي بيقرأ الـ tables دي ويطبقها على الـ hardware.
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة اللي بيحلها الـ Pinctrl Subsystem

الـ SoC الحديث زي Amlogic C3 بيحتوي على مئات الـ physical pins. كل pin ممكن يشتغل بأكثر من وظيفة — نفس الـ pin الجسدي ممكن يكون:
- GPIO عادي (input/output)
- UART TX
- SPI MOSI
- PWM output
- I2C SDA

ده بيسموه **pin multiplexing** أو اختصاراً **pinmux**.

المشكلة المتعددة الأوجه:

| المشكلة | التفاصيل |
|---------|----------|
| **Hardware diversity** | كل SoC عنده register layout مختلف تماماً |
| **Shared pins** | نفس الـ pin بيتطلبه أكثر من driver في نفس الوقت |
| **Configuration** | pull-up/pull-down، drive-strength، direction — كلها بتتغير per-pin |
| **Boot state** | اللي بيحتاجه الـ bootloader غير اللي بيحتاجه الـ kernel |

لو كل driver حل المشكلة بنفسه، الكود هيتشتت، وهيحصل conflicts لما درايفرين يحاولوا يتحكموا في نفس الـ pin في نفس الوقت.

---

### الحل — الـ Pinctrl Subsystem

الـ kernel قدم subsystem مركزي اسمه **pinctrl** — موجود في `drivers/pinctrl/` — بيعمل الآتي:

1. **Abstraction layer**: بيخفي الـ hardware register details ورا واجهة موحدة.
2. **Ownership tracking**: بيعرف مين استخدم كل pin وبيمنع الـ conflicts.
3. **DT integration**: بيقرأ الـ pin configuration من الـ Device Tree بدل ما تكون hardcoded.
4. **GPIO bridging**: بيوصل الـ pinctrl بالـ `gpio_chip` عشان الـ GPIO subsystem يقدر يشتغل.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Consumer Drivers                          │
│  (MMC driver, SPI driver, UART driver, I2C driver, etc.)        │
│                                                                  │
│   pinctrl_select_state("default")  ← from Device Tree           │
└────────────────────────────┬────────────────────────────────────┘
                             │ pinctrl_get / pinctrl_select_state
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Pinctrl Core (kernel/generic)                  │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────┐  │
│  │  pinctrl_dev   │  │  pinctrl_map   │  │  pinctrl_state   │  │
│  │  (per device)  │  │  (pin→func)    │  │  (named state)   │  │
│  └────────────────┘  └────────────────┘  └──────────────────┘  │
│                                                                  │
│   Interfaces: pinctrl_ops | pinmux_ops | pinconf_ops            │
└──────────────────────┬──────────────────────────────────────────┘
                       │ calls vtable ops
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              Platform Driver — pinctrl-amlogic-c3.c              │
│              (+ pinctrl-meson.c for common logic)                │
│                                                                  │
│  c3_periphs_pins[]     ← كل الـ pins بالاسم والرقم             │
│  c3_periphs_groups[]   ← groups (كل function بتاعة set من pins)│
│  c3_periphs_functions[]← functions (uart_a, spi_a, emmc, ...)  │
│  c3_periphs_banks[]    ← GPIO banks + register offsets          │
│  meson_axg_pmx_ops     ← implementation للـ pinmux_ops          │
└──────────────────────┬──────────────────────────────────────────┘
                       │ regmap_write / regmap_read
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              Hardware Registers — Amlogic C3 SoC                 │
│                                                                  │
│  PMX regs  (0x00–0x12): function select per-pin (4 bits each)  │
│  PULLEN    (0x03,0x23…): pull resistor enable                   │
│  PULL      (0x04,0x24…): pull direction (up/down)               │
│  DIR       (0x02,0x22…): GPIO direction (in/out)                │
│  OUT       (0x01,0x21…): GPIO output value                      │
│  IN        (0x00,0x20…): GPIO input value                       │
│  DS        (0x07,0x27…): drive strength                         │
└─────────────────────────────────────────────────────────────────┘
```

---

### Analogy عميق — لوحة توزيع الكهرباء

تخيل فندق كبير فيه غرف كتير، وفي الـ basement في **لوحة توزيع كهربا** (distribution board) فيها مئات الـ circuit breakers.

| مفهوم Kernel | الـ Analogy |
|-------------|-------------|
| **Physical pin** | wire فيزيائي بيوصل من الـ board للـ SoC |
| **Pin bank** (GPIOB, GPIOX) | صف من الـ breakers في اللوحة — كلهم بيشاركوا نفس الـ control panel |
| **Pin group** (emmc_clk_pins) | الوايرات اللي بتوصل جهاز واحد (مثلاً الغسالة) لمصدر الكهرباء |
| **Function** (emmc) | الخدمة الكاملة (الغسالة بكل وايراتها) |
| **Pinmux register** | المفتاح اللي بيحدد إيه اللي موصل على الـ wire ده |
| **pinctrl_select_state** | اللي بيجي يقول "وصل الغسالة" — الـ maintenance man |
| **Pinctrl core** | مدير اللوحة اللي بيمنع حد يوصل جهازين على نفس الـ socket |
| **meson_axg_pmx_ops** | الـ manual بتاع اللوحة — بيقول ازاي تشغل كل مفتاح |
| **regmap** | الـ remote control للوحة — ما بتحتاجش تروح هناك فيزيائياً |
| **DT pin config node** | ورقة الطلب اللي الـ tenant بيكتب فيها إيه اللي محتاجه |

الجزء الأعمق في الـ analogy: زي ما في الفندق مش ممكن تشغل الغسالة والمكنسة على نفس الـ socket في نفس الوقت — الـ pinctrl core بيعمل exactly ده. لو `uart_a` طلب GPIOB_0 وجه، والـ SPI طلبه بعده، الـ core بيرفض أو بيـ override حسب الـ policy.

---

### الـ Core Abstraction — الفكرة المركزية

الفكرة الجوهرية في الـ pinctrl framework هي **الفصل بين ثلاث مستويات**:

```
Level 1: PIN        — الـ physical wire اللي على الـ chip
          ↓
Level 2: GROUP      — مجموعة pins بتشكل function واحدة (مثلاً emmc_clk + emmc_cmd + emmc_d0..7)
          ↓
Level 3: FUNCTION   — الاسم المنطقي للـ peripheral (emmc, uart_a, spi_b, ...)
```

الـ consumer driver (مثلاً MMC driver) بيطلب function بالاسم. الـ core بيترجم الاسم ده لـ groups، والـ groups لـ pins، والـ pins لـ register bits، وبيكتب في الـ registers.

---

### شرح الـ Data Structures في العمق

#### 1. `struct pinctrl_pin_desc` — وصف الـ pin الواحد

```c
/* من include/linux/pinctrl/pinctrl.h */
struct pinctrl_pin_desc {
    unsigned int number;  /* رقم فريد — global pin number space */
    const char *name;     /* اسم للـ debugfs والـ DT */
    void *drv_data;       /* driver-specific data — pinctrl core بيمسهاش */
};

/* الـ C3 driver بيبني array منها */
static const struct pinctrl_pin_desc c3_periphs_pins[] = {
    MESON_PIN(GPIOE_0),  /* expand to: { GPIOE_0, "GPIOE_0" } */
    MESON_PIN(GPIOB_0),
    /* ... 54 pins total */
};
```

#### 2. `struct meson_pmx_group` — الـ pin group

```c
/* من pinctrl-meson.h */
struct meson_pmx_group {
    const char *name;           /* "emmc_clk" أو "GPIOB_8" */
    const unsigned int *pins;   /* array of pin numbers */
    unsigned int num_pins;      /* غالباً 1 في الـ C3 */
    const void *data;           /* pointer لـ meson_pmx_axg_data */
};

/* الـ data بيحتوي على رقم الـ function في الـ mux register */
struct meson_pmx_axg_data {
    unsigned int func;  /* 0=GPIO, 1=func1, 2=func2, ... 6=func6 */
};
```

الـ GROUP macro بيملا الحقلين دول مع بعض:

```c
/* GPIOB_8 على func1 يعني emmc_clk */
GROUP(emmc_clk, 1)
/* بيتحول لـ: */
{
    .name = "emmc_clk",
    .pins = emmc_clk_pins,         /* = { GPIOB_8 } */
    .num_pins = 1,
    .data = (struct meson_pmx_axg_data[]){ .func = 1 },
}
```

#### 3. `struct meson_pmx_func` — الـ function

```c
struct meson_pmx_func {
    const char *name;               /* "emmc" أو "uart_a" */
    const char * const *groups;     /* array of group names */
    unsigned int num_groups;
};

/* الـ FUNCTION macro: */
FUNCTION(emmc)
/* بيتحول لـ: */
{
    .name = "emmc",
    .groups = emmc_groups,   /* {"emmc_nand_d0", ..., "emmc_cmd", ...} */
    .num_groups = ARRAY_SIZE(emmc_groups),
}
```

**ملاحظة مهمة**: function واحدة ممكن عندها groups في أكتر من bank. مثلاً `uart_a` عندها groups في Bank B وBank C وBank X — ده بيدي الـ board designer flexibility يختار أي bank يستخدم.

#### 4. `struct meson_bank` — الـ GPIO bank

```c
struct meson_bank {
    const char *name;        /* "B", "X", "D", ... */
    unsigned int first;      /* GPIOB_0 */
    unsigned int last;       /* GPIOB_14 */
    int irq_first;           /* hwirq للـ GPIO interrupt */
    int irq_last;
    struct meson_reg_desc regs[MESON_NUM_REG]; /* register map */
};
```

الـ `meson_reg_desc` هو pair من (register_offset, bit_offset) لكل نوع تحكم:

```c
/* Bank B في الـ C3: */
BANK_DS("B", GPIOB_0, GPIOB_14, 0, 14,
    /* pullen_reg, bit */  0x53, 0,
    /* pull_reg,   bit */  0x54, 0,
    /* dir_reg,    bit */  0x52, 0,
    /* out_reg,    bit */  0x51, 0,
    /* in_reg,     bit */  0x50, 0,
    /* ds_reg,     bit */  0x57, 0)
```

يعني لو عايز تقرأ قيمة GPIOB_3: بتروح لـ `reg_gpio[0x50]`، بتاخد الـ bit رقم 3 (لأنه أول pin في الـ bank + offset 3).

#### 5. `struct meson_pmx_bank` — الـ PMX register bank

```c
struct meson_pmx_bank {
    const char *name;
    unsigned int first;   /* أول pin */
    unsigned int last;    /* آخر pin */
    unsigned int reg;     /* أول register للـ mux في الـ bank */
    unsigned int offset;  /* bit offset */
};

/* Bank B: */
BANK_PMX("B", GPIOB_0, GPIOB_14, 0x00, 0)
```

كل pin بياخد 4 bits في الـ mux register (عشان في 7 functions + GPIO = 8 قيم → بس في الـ meson بياخدوا 4 bits عشان في عندهم لحد func 6). يعني GPIOB_0 في register 0x00 bits [3:0]، و GPIOB_1 في bits [7:4]، إلخ.

---

### العلاقة بين الـ Structs — رسم توضيحي

```
meson_pinctrl_data (c3_periphs_pinctrl_data)
│
├── pins[]     → pinctrl_pin_desc[]   (54 pins, رقم + اسم)
│
├── groups[]   → meson_pmx_group[]   (GPIO groups + function groups)
│               │
│               └── data → meson_pmx_axg_data { func=N }
│
├── funcs[]    → meson_pmx_func[]    (uart_a, emmc, spi_b, ...)
│               │
│               └── groups[] → string names → يرجع يطابق groups[] بالاسم
│
├── banks[]    → meson_bank[]        (GPIOB, GPIOX, GPIOC, ...)
│               │
│               └── regs[] → meson_reg_desc { reg, bit } × 6 types
│
├── pmx_ops    → meson_axg_pmx_ops   (vtable للـ pinmux operations)
│
└── pmx_data   → meson_axg_pmx_data
                 └── pmx_banks[] → meson_pmx_bank[] (PMX register layout)
```

---

### الـ Pinctrl Subsystem بيملك إيه vs. بيـ delegate إيه للـ Driver

| المسؤولية | Pinctrl Core يملكها | Meson Driver يملكها |
|-----------|--------------------|--------------------|
| تتبع مين استخدم كل pin | ✅ | ❌ |
| منع الـ conflicts بين drivers | ✅ | ❌ |
| ترجمة DT nodes لـ pin maps | ✅ (generic) أو driver | ✅ (`meson_a1_parse_dt_extra`) |
| تسجيل الـ pin controller | ✅ (`pinctrl_register`) | ❌ |
| كتابة الـ mux register فعلياً | ❌ | ✅ (`meson_axg_pmx_ops`) |
| تعريف الـ register layout | ❌ | ✅ (`c3_periphs_banks`, `c3_periphs_pmx_banks`) |
| الـ GPIO chip operations | ✅ (gpio core) | ✅ (يملا `gpio_chip`) |
| قراءة/كتابة pin values | ❌ | ✅ (عبر `regmap`) |
| الـ pull/drive-strength config | ❌ | ✅ (`meson_pinconf_ops`) |
| الـ IRQ handling للـ GPIO | ❌ | ✅ (عبر `gpio_chip.irq`) |

---

### الـ regmap — مفهوم مهم لازم تفهمه

**الـ regmap** هو abstraction layer للـ register access في الـ kernel. بدل ما يكتب الـ driver مباشرة `writel(val, base + offset)` — اللي بيعتمد على الـ physical address — الـ regmap بيديه:

- Thread-safety
- Caching
- Debugging (عبر `/sys/kernel/debug/regmap/`)
- تجريد عن طريقة الـ access (MMIO, I2C, SPI)

الـ `meson_pinctrl` بياخد عدة `regmap` instances:

```c
struct meson_pinctrl {
    struct regmap *reg_mux;    /* PMX registers — function select */
    struct regmap *reg_pullen; /* pull enable registers */
    struct regmap *reg_pull;   /* pull direction registers */
    struct regmap *reg_gpio;   /* GPIO direction/value registers */
    struct regmap *reg_ds;     /* drive strength registers */
    /* ... */
};
```

في الـ C3 (عبر `meson_a1_parse_dt_extra`)، الـ regmaps دي بتيجي من الـ DT nodes — مش من base address ثابت في الـ code — عشان نفس الـ driver code يشتغل على SoCs مختلفة.

---

### الـ Platform Driver و الـ Probe Flow

```
module_platform_driver(c3_pinctrl_driver)
         │
         └── .probe = meson_pinctrl_probe   (في pinctrl-meson.c)
                      │
                      ├── 1. يقرأ الـ of_device_id → يجيب c3_periphs_pinctrl_data
                      ├── 2. يعمل regmap instances من الـ DT
                      ├── 3. يملا pinctrl_desc من الـ data
                      ├── 4. يسجل الـ pin controller: pinctrl_register_and_init()
                      ├── 5. يسجل الـ GPIO ranges
                      └── 6. يعمل gpio_chip ويسجله
```

الـ `c3_pinctrl_dt_match` بيربط الـ DT compatible string بالـ data struct:

```c
static const struct of_device_id c3_pinctrl_dt_match[] = {
    {
        .compatible = "amlogic,c3-periphs-pinctrl",
        .data = &c3_periphs_pinctrl_data,  /* كل الـ C3-specific data */
    },
    { }
};
```

لو الـ DT فيه node بالـ compatible ده، الـ kernel بيـ match الـ driver ده ويشغله. الـ `meson_pinctrl_probe` — اللي مش موجودة في الـ C3 file نفسه — هي الـ shared probe موجودة في `pinctrl-meson.c` وبتستخدم الـ data اللي الـ C3 file عرّفها.

---

### الـ Mux Register Mechanics — كيف بيتم الـ Function Select فعلياً

كل pin في الـ C3 بياخد **4 bits** في الـ PMX registers. الـ 4 bits دي بتحدد الـ function:

```
func=0 → GPIO mode
func=1 → Alternative function 1
func=2 → Alternative function 2
...
func=6 → Alternative function 6
```

مثال عملي — GPIOB_8:
```
Bank B → PMX reg starts at 0x00
GPIOB_8 → pin number 8 in bank → bits [35:32] في register 0x01
(كل register 32-bit → pins 0-7 في reg 0x00، pins 8-15 في reg 0x01)

لو كتبنا func=1 → emmc_clk شغال
لو كتبنا func=2 → nand_wen_clk شغال
لو كتبنا func=3 → pwm_j_b شغال
لو كتبنا func=4 → lcd_d3 شغال
لو كتبنا func=5 → spi_a_ss0_b شغال
لو كتبنا func=6 → uart_a_rts_b شغال
لو كتبنا func=0 → GPIO عادي
```

الـ `meson_axg_pmx_ops` بتحسب الـ register و bit offset من الـ `meson_pmx_bank` وبتعمل `regmap_update_bits()`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Flags والـ Config Options — Cheatsheet

#### enum meson_reg_type — أنواع الـ registers اللي بيتحكم فيها الـ driver

| القيمة | الرقم | الوظيفة |
|---|---|---|
| `MESON_REG_PULLEN` | 0 | تفعيل/تعطيل الـ pull resistor |
| `MESON_REG_PULL` | 1 | اتجاه الـ pull (up/down) |
| `MESON_REG_DIR` | 2 | اتجاه الـ pin (input/output) |
| `MESON_REG_OUT` | 3 | قيمة الـ output |
| `MESON_REG_IN` | 4 | قراءة الـ input |
| `MESON_REG_DS` | 5 | الـ drive strength |
| `MESON_NUM_REG` | 6 | عدد الـ registers (للـ array sizing) |

#### enum meson_pinconf_drv — قيم الـ drive strength المتاحة

| القيمة | التيار |
|---|---|
| `MESON_PINCONF_DRV_500UA` | 500 µA |
| `MESON_PINCONF_DRV_2500UA` | 2500 µA |
| `MESON_PINCONF_DRV_3000UA` | 3000 µA |
| `MESON_PINCONF_DRV_4000UA` | 4000 µA |

#### الـ GPIO Banks في C3 SoC

| Bank | أول pin | آخر pin | عدد الـ pins | أول IRQ | آخر IRQ | Mux Reg |
|---|---|---|---|---|---|---|
| X | GPIOX_0 | GPIOX_13 | 14 | 40 | 53 | 0x03 |
| D | GPIOD_0 | GPIOD_6 | 7 | 33 | 39 | 0x10 |
| E | GPIOE_0 | GPIOE_4 | 5 | 22 | 26 | 0x12 |
| C | GPIOC_0 | GPIOC_6 | 7 | 15 | 21 | 0x09 |
| B | GPIOB_0 | GPIOB_14 | 15 | 0 | 14 | 0x00 |
| A | GPIOA_0 | GPIOA_5 | 6 | 27 | 32 | 0x0b |
| TEST_N | GPIO_TEST_N | GPIO_TEST_N | 1 | 54 | 54 | 0x02 |

#### أرقام الـ Function في الـ Mux

| الرقم | المعنى |
|---|---|
| 0 | GPIO (no mux, default) |
| 1 | func1 (الوظيفة الأولى للـ bank) |
| 2 | func2 |
| 3 | func3 |
| 4 | func4 |
| 5 | func5 |
| 6 | func6 |

#### الوظائف المدعومة (Functions) في C3

| الوظيفة | عدد الـ Groups |
|---|---|
| gpio_periphs | 54 (كل الـ GPIOs) |
| uart_a..uart_e | متعددة |
| i2c0..i2c3, i2c_slave | متعددة |
| pwm_a..pwm_n | متعددة |
| emmc, nand, spif | block storage |
| sdcard, sdio | SD |
| spi_a, spi_b | SPI |
| tdm, pdm, mclk_0, mclk_1 | audio |
| lcd | display (18-bit) |
| eth | Ethernet LEDs |
| jtag_a, jtag_b | debug |
| ir_in, ir_out | infrared |
| gen_clk, clk12_24, clk_32k_in | clocks |

---

### 1. الـ Structs المهمة

#### struct pinctrl_pin_desc

**الغرض:** وصف pin واحد بالـ ID والاسم — ده الـ building block الأساسي.

```c
struct pinctrl_pin_desc {
    unsigned number;   /* رقم الـ pin globally unique */
    const char *name;  /* اسم نصي زي "GPIOB_0" */
    void *drv_data;    /* بيانات إضافية للـ driver */
};
```

في الملف بيتعرف كل pin عن طريق الـ macro:
```c
#define MESON_PIN(x) PINCTRL_PIN(x, #x)
/* مثال: MESON_PIN(GPIOB_0) → { GPIOB_0, "GPIOB_0", NULL } */
```

المصفوفة `c3_periphs_pins[]` فيها 54 pin للـ C3 SoC.

---

#### struct meson_pmx_group

**الغرض:** تجميع pin أو أكثر مع بعض تحت اسم واحد، ومعاه رقم الـ function اللي يشتغل بيه.

```c
struct meson_pmx_group {
    const char *name;          /* اسم الـ group زي "uart_a_tx_b" */
    const unsigned int *pins;  /* مصفوفة أرقام الـ pins */
    unsigned int num_pins;     /* عدد الـ pins */
    const void *data;          /* → meson_pmx_axg_data (func number) */
};
```

نوعان من الـ groups:
- **GPIO group:** pin واحد، func = 0 (الـ GPIO الافتراضي)، يتعرف بـ `GPIO_GROUP(gpio)`
- **Function group:** pin أو أكثر، func = 1..6، يتعرف بـ `GROUP(grp, f)`

**مثال:**
```c
/* GPIO group — func 0 */
GPIO_GROUP(GPIOB_0)
/* → { "GPIOB_0", {GPIOB_0}, 1, PMX_DATA(0) } */

/* Function group — func 1 */
GROUP(emmc_clk, 1)
/* → { "emmc_clk", emmc_clk_pins, 1, PMX_DATA(1) } */
```

الـ `data` بيشاور على `meson_pmx_axg_data`:
```c
struct meson_pmx_axg_data {
    unsigned int func;  /* رقم الـ function: 0..6 */
};
```

---

#### struct meson_pmx_func

**الغرض:** تجميع عدة groups تحت اسم وظيفة واحدة — الـ kernel بيعرضها لـ userspace.

```c
struct meson_pmx_func {
    const char *name;               /* اسم الوظيفة زي "uart_a" */
    const char * const *groups;     /* مصفوفة أسماء الـ groups */
    unsigned int num_groups;        /* عدد الـ groups */
};
```

يتعرف عن طريق:
```c
#define FUNCTION(fn) \
    { .name = #fn, .groups = fn##_groups, .num_groups = ARRAY_SIZE(fn##_groups) }
```

**مثال:**
```c
FUNCTION(uart_a)
/* → { "uart_a", uart_a_groups, 10 } */
/* uart_a_groups يحتوي: "uart_a_tx_b", "uart_a_rx_b", ..., "uart_a_rx_d" */
```

---

#### struct meson_reg_desc

**الغرض:** تحديد موقع bit معين في register معين — يُستخدم للتحكم في pull/dir/out/in/ds لكل pin.

```c
struct meson_reg_desc {
    unsigned int reg;  /* offset في الـ regmap */
    unsigned int bit;  /* رقم الـ bit داخل الـ register */
};
```

---

#### struct meson_bank

**الغرض:** وصف bank كامل من الـ GPIOs، بما في ذلك كل الـ register descriptors للتحكم في الـ pins.

```c
struct meson_bank {
    const char *name;       /* اسم الـ bank زي "B" */
    unsigned int first;     /* أول pin رقمه */
    unsigned int last;      /* آخر pin رقمه */
    int irq_first;          /* أول hwirq للـ bank */
    int irq_last;           /* آخر hwirq للـ bank */
    struct meson_reg_desc regs[MESON_NUM_REG]; /* 6 registers */
};
```

`regs[MESON_NUM_REG]` — مصفوفة بـ 6 عناصر:

```
regs[0] = PULLEN  → {reg=0x53, bit=0}  ← Bank B مثلاً
regs[1] = PULL    → {reg=0x54, bit=0}
regs[2] = DIR     → {reg=0x52, bit=0}
regs[3] = OUT     → {reg=0x51, bit=0}
regs[4] = IN      → {reg=0x50, bit=0}
regs[5] = DS      → {reg=0x57, bit=0}
```

لكل pin في الـ bank، الـ driver بيحسب الـ bit الصح بإضافة `(pin - bank->first)` لـ `regs[type].bit`.

الـ Macro:
```c
BANK_DS("B", GPIOB_0, GPIOB_14, 0, 14,
    0x53,0, 0x54,0, 0x52,0, 0x51,0, 0x50,0, 0x57,0)
/*   pullen   pull    dir     out     in      ds    */
```

---

#### struct meson_pmx_bank

**الغرض:** ربط كل GPIO bank بالـ mux register الخاص بيه — ده للـ AXG-style pinmux.

```c
struct meson_pmx_bank {
    const char *name;      /* اسم الـ bank */
    unsigned int first;    /* أول pin */
    unsigned int last;     /* آخر pin */
    unsigned int reg;      /* base register للـ mux في هذا الـ bank */
    unsigned int offset;   /* bit offset ابتداء منه */
};
```

**مثال:**
```c
BANK_PMX("B", GPIOB_0, GPIOB_14, 0x00, 0)
/* 15 pins × 4 bits/pin = 60 bits → registers 0x00, 0x01, 0x02 */
```

---

#### struct meson_axg_pmx_data

**الغرض:** تجميع كل الـ pmx banks معاً في structure واحد يُمرر للـ core.

```c
struct meson_axg_pmx_data {
    const struct meson_pmx_bank *pmx_banks;  /* مصفوفة الـ banks */
    unsigned int num_pmx_banks;              /* عددهم */
};
```

في C3:
```c
static const struct meson_axg_pmx_data c3_periphs_pmx_banks_data = {
    .pmx_banks    = c3_periphs_pmx_banks,   /* 7 banks */
    .num_pmx_banks = 7,
};
```

---

#### struct meson_pinctrl_data

**الغرض:** الـ static data الخاصة بكل SoC — المرجعية الكاملة للـ pinctrl core.

```c
struct meson_pinctrl_data {
    const char *name;                          /* "periphs-banks" */
    const struct pinctrl_pin_desc *pins;       /* مصفوفة الـ pins (54 pin) */
    const struct meson_pmx_group *groups;      /* مصفوفة الـ groups */
    const struct meson_pmx_func *funcs;        /* مصفوفة الـ functions */
    unsigned int num_pins;
    unsigned int num_groups;
    unsigned int num_funcs;
    const struct meson_bank *banks;            /* GPIO bank descriptors */
    unsigned int num_banks;
    const struct pinmux_ops *pmx_ops;          /* → meson_axg_pmx_ops */
    const void *pmx_data;                      /* → meson_axg_pmx_data */
    int (*parse_dt)(struct meson_pinctrl *pc); /* → meson_a1_parse_dt_extra */
};
```

هذا الـ struct هو الـ "شخصية" الكاملة للـ C3 SoC في نظر الـ pinctrl subsystem.

---

#### struct meson_pinctrl

**الغرض:** الـ instance الحي للـ driver أثناء الـ runtime — بيحتوي كل الـ state.

```c
struct meson_pinctrl {
    struct device *dev;            /* الـ platform device */
    struct pinctrl_dev *pcdev;     /* مؤشر على الـ pinctrl core device */
    struct pinctrl_desc desc;      /* وصف الـ pinctrl للـ core */
    struct meson_pinctrl_data *data; /* الـ SoC-specific data */
    struct regmap *reg_mux;        /* regmap للـ mux registers */
    struct regmap *reg_pullen;     /* regmap لتفعيل الـ pull */
    struct regmap *reg_pull;       /* regmap لاتجاه الـ pull */
    struct regmap *reg_gpio;       /* regmap لـ dir/in/out */
    struct regmap *reg_ds;         /* regmap للـ drive strength */
    struct gpio_chip chip;         /* الـ GPIO chip للـ kernel */
    struct fwnode_handle *fwnode;  /* الـ device tree node */
};
```

---

### 2. مخططات العلاقات بين الـ Structs

```
                    ┌─────────────────────────────────┐
                    │     meson_pinctrl_data           │
                    │  (c3_periphs_pinctrl_data)       │
                    │                                  │
                    │  .pins  ─────────────────────────┼──► pinctrl_pin_desc[]
                    │  .groups ────────────────────────┼──► meson_pmx_group[]
                    │  .funcs  ────────────────────────┼──► meson_pmx_func[]
                    │  .banks  ────────────────────────┼──► meson_bank[]
                    │  .pmx_ops ───────────────────────┼──► meson_axg_pmx_ops
                    │  .pmx_data ──────────────────────┼──► meson_axg_pmx_data
                    │  .parse_dt ──────────────────────┼──► meson_a1_parse_dt_extra()
                    └───────────────┬─────────────────┘
                                    │ .data
                                    ▼
                    ┌─────────────────────────────────┐
                    │       meson_pinctrl              │
                    │   (runtime instance)             │
                    │                                  │
                    │  .dev ──────► platform_device    │
                    │  .pcdev ────► pinctrl_dev        │
                    │  .desc ─────► pinctrl_desc       │
                    │  .reg_mux ──► regmap             │
                    │  .reg_gpio ─► regmap             │
                    │  .reg_ds ───► regmap             │
                    │  .chip ─────► gpio_chip          │
                    └─────────────────────────────────┘

                    ┌─────────────────────────────────┐
                    │      meson_axg_pmx_data          │
                    │  .pmx_banks ────────────────────►│ meson_pmx_bank[]
                    │  .num_pmx_banks = 7              │  .reg = mux base offset
                    └─────────────────────────────────┘

                    ┌─────────────────────────────────┐
                    │       meson_pmx_group            │
                    │  .name = "emmc_clk"              │
                    │  .pins ─────────────────────────►│ unsigned int[] {GPIOB_8}
                    │  .data ─────────────────────────►│ meson_pmx_axg_data
                    │                                  │   .func = 1
                    └─────────────────────────────────┘

                    ┌─────────────────────────────────┐
                    │       meson_pmx_func             │
                    │  .name = "emmc"                  │
                    │  .groups ───────────────────────►│ char*[] {"emmc_clk", ...}
                    │  .num_groups = 12                │  (أسماء → lookup في groups[])
                    └─────────────────────────────────┘

                    ┌─────────────────────────────────┐
                    │        meson_bank                │
                    │  .name = "B"                     │
                    │  .first = GPIOB_0                │
                    │  .last  = GPIOB_14               │
                    │  .regs[PULLEN] = {0x53, 0}       │
                    │  .regs[PULL]   = {0x54, 0}       │
                    │  .regs[DIR]    = {0x52, 0}       │
                    │  .regs[OUT]    = {0x51, 0}       │
                    │  .regs[IN]     = {0x50, 0}       │
                    │  .regs[DS]     = {0x57, 0}       │
                    └─────────────────────────────────┘
```

---

### 3. دورة حياة الـ Driver — Lifecycle Diagram

```
  ┌─────────────────────────────────────────────────────────────┐
  │  CREATION / REGISTRATION                                    │
  │                                                             │
  │  1. Kernel يقرأ DT: compatible = "amlogic,c3-periphs-pinctrl"
  │         │                                                   │
  │         ▼                                                   │
  │  2. platform_driver_probe() يطلع منه مؤشر على               │
  │     c3_periphs_pinctrl_data (عن طريق of_device_id.data)     │
  │         │                                                   │
  │         ▼                                                   │
  │  3. meson_pinctrl_probe() يُستدعى:                          │
  │     a. يخصص meson_pinctrl                                  │
  │     b. يستدعي parse_dt → meson_a1_parse_dt_extra()         │
  │        → يجمع الـ regmaps (reg_mux, reg_gpio, reg_ds, ...)  │
  │     c. يسجل pinctrl_desc مع الـ kernel                     │
  │        → devm_pinctrl_register()                           │
  │     d. يسجل gpio_chip                                       │
  │        → devm_gpiochip_add_data()                           │
  │         │                                                   │
  │         ▼                                                   │
  │  4. الـ driver جاهز — الـ kernel بيعرف كل pins وgroups وfuncs
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │  USAGE                                                      │
  │                                                             │
  │  consumer driver (e.g. UART driver) يطلب pinmux:           │
  │  pinctrl_select_state() → pmx_ops->set_mux()               │
  │  → يبحث في groups عن الـ group المطلوب                      │
  │  → يقرأ meson_pmx_axg_data.func                            │
  │  → يكتب في reg_mux عن طريق regmap_update_bits()            │
  │                                                             │
  │  GPIO consumer يطلب input/output:                          │
  │  gpio_direction_input() → .get() → reg_gpio read           │
  │  gpio_direction_output() → .set() → reg_gpio write         │
  └─────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │  TEARDOWN                                                   │
  │                                                             │
  │  module_exit / device removal:                              │
  │  platform_driver_unregister()                               │
  │  → devm cleanup يشيل gpio_chip وpinctrl_dev تلقائياً        │
  └─────────────────────────────────────────────────────────────┘
```

---

### 4. Call Flow Diagrams

#### 4.1 تفعيل Pinmux (مثال: تشغيل UART A على Bank B)

```
consumer driver (uart-meson.c)
  → pinctrl_select_state(pctldev, state)
      → pinmux_apply_muxsetting()
          → pmx_ops->set_mux(pcdev, func_selector, group_selector)
              [func_selector = index of "uart_a" in c3_periphs_functions]
              [group_selector = index of "uart_a_tx_b" in c3_periphs_groups]
              → meson_axg_pmx_ops.set_mux()
                  → يبحث عن meson_pmx_group بالـ group_selector
                  → يقرأ group->data → meson_pmx_axg_data.func = 6
                  → يبحث عن meson_pmx_bank اللي فيه GPIOB_0
                      → c3_periphs_pmx_banks: Bank "B", reg=0x00
                  → يحسب الـ bit: (GPIOB_0 - bank->first) * 4 = 0
                  → regmap_update_bits(reg_mux, 0x00, mask, func<<bit)
                      → Hardware register يُكتب 0x6 في bits[3:0]
```

#### 4.2 قراءة GPIO Input

```
userspace / kernel consumer
  → gpiod_get_value(desc)
      → gpio_chip->get(chip, offset)
          → meson_gpio_get() [في pinctrl-meson.c]
              → يبحث عن meson_bank اللي فيه الـ pin
              → bank->regs[MESON_REG_IN] = {reg, bit}
              → bit_offset = offset - (bank->first) + regs[IN].bit
              → regmap_read(reg_gpio, regs[IN].reg, &val)
              → return (val >> bit_offset) & 1
```

#### 4.3 ضبط Drive Strength

```
pinctrl_configure_pin(pctldev, pin, PIN_CONFIG_DRIVE_STRENGTH)
  → pctlops->pin_config_set()
      → meson_pinconf_set() [في pinctrl-meson.c]
          → يبحث عن meson_bank
          → bank->regs[MESON_REG_DS] = {reg=0x57, bit=0}
          → يحوّل قيمة uA → meson_pinconf_drv enum
          → regmap_update_bits(reg_ds, 0x57, mask, drv_val)
```

#### 4.4 Probe Flow المبسط

```
module_platform_driver(c3_pinctrl_driver)
  → platform_driver_register()
      → عند match مع DT:
          → c3_pinctrl_driver.probe = meson_pinctrl_probe(pdev)
              → of_device_get_match_data() → c3_periphs_pinctrl_data
              → devm_kzalloc(meson_pinctrl)
              → data->parse_dt(pc) = meson_a1_parse_dt_extra(pc)
                  → يجمع regmaps من DT:
                      "mux"    → pc->reg_mux
                      "gpio"   → pc->reg_gpio
                      "ds"     → pc->reg_ds
                      "pull"   → pc->reg_pull
                      "pull-en"→ pc->reg_pullen
              → meson_pinctrl_build_state()
              → devm_pinctrl_register(dev, &desc, pc) → pc->pcdev
              → devm_gpiochip_add_data(dev, &pc->chip, pc)
```

---

### 5. استراتيجية الـ Locking

الملف `pinctrl-amlogic-c3.c` نفسه ما فيهوش أي locks لأنه **بيانات static فقط** — كلها `const` arrays لا تتغير بعد الـ boot.

الـ locking الحقيقي بيحصل في طبقتين أعلى:

#### 5.1 طبقة الـ Pinctrl Core (kernel/drivers/pinctrl/core.c)

| الـ Lock | نوعه | بيحمي إيه |
|---|---|---|
| `pctldev->mutex` | `mutex` | كل العمليات على الـ pinctrl device |
| per-pin state lock | داخل الـ core | تغيير الـ mux state |

#### 5.2 طبقة الـ GPIO (gpio_chip)

| الـ Lock | نوعه | بيحمي إيه |
|---|---|---|
| `gpio_chip->bgpio_lock` | `spinlock` | قراءة/كتابة الـ GPIO registers |

الـ regmap نفسه عنده `regmap->lock` داخلي بيحمي الـ register access.

#### 5.3 ترتيب الـ Locks (Lock Ordering)

```
pinctrl_dev->mutex          (الأعلى — يُمسك أولاً)
    └─► gpio_chip->bgpio_lock  (spinlock — يُمسك جوا الـ mutex)
            └─► regmap->lock    (داخلي في regmap_update_bits)
```

**مهم:** ما في deadlock ممكن لأن:
- الـ `mutex` ما بيتمسكش من interrupt context
- الـ `spinlock` يتمسك من أي context
- الـ regmap lock داخلي ومش exposed

#### 5.4 الـ const Protection

كل البيانات في الملف معلنة `const static`:
```c
static const struct meson_bank c3_periphs_banks[]  = { ... };
static const struct meson_pmx_group c3_periphs_groups[] = { ... };
static const struct meson_pinctrl_data c3_periphs_pinctrl_data = { ... };
```

ده معناه إن الـ SoC-specific data محمية بشكل implicit من الكتابة على مستوى الـ compiler — ما في أي thread يقدر يعدل فيها أثناء الـ runtime، فمحتاجتش locks.
## Phase 4: شرح الـ Functions

---

### ملخص الـ APIs والـ Macros المهمة

#### جدول الـ Macros (Data Descriptors)

| الـ Macro | الغرض | الملف |
|-----------|--------|-------|
| `MESON_PIN(x)` | تعريف `pinctrl_pin_desc` من اسم الـ pin | `pinctrl-meson.h` |
| `GROUP(grp, f)` | تعريف `meson_pmx_group` لـ function group بـ func رقم `f` | `pinctrl-meson-axg-pmx.h` |
| `GPIO_GROUP(gpio)` | تعريف `meson_pmx_group` لـ GPIO group (func=0) | `pinctrl-meson-axg-pmx.h` |
| `FUNCTION(fn)` | تعريف `meson_pmx_func` من اسم الـ function | `pinctrl-meson.h` |
| `BANK_DS(...)` | تعريف `meson_bank` كامل مع drive-strength | `pinctrl-meson.h` |
| `BANK_PMX(n,f,l,r,o)` | تعريف `meson_pmx_bank` لجزء الـ pinmux | `pinctrl-meson-axg-pmx.h` |
| `PMX_DATA(f)` | تعريف `meson_pmx_axg_data` بـ function index | `pinctrl-meson-axg-pmx.h` |

#### جدول الـ Functions الخارجية المستخدمة

| الـ Function | المصدر | الغرض |
|-------------|--------|-------|
| `meson_pinctrl_probe` | `pinctrl-meson.c` | الـ probe callback المشترك |
| `meson_a1_parse_dt_extra` | `pinctrl-meson-a1.c` | parse الـ DT للـ regmap setup |
| `meson_axg_pmx_ops` | `pinctrl-meson-axg-pmx.c` | الـ pinmux ops المشتركة |
| `meson_pmx_get_funcs_count` | `pinctrl-meson.c` | عدد الـ functions |
| `meson_pmx_get_func_name` | `pinctrl-meson.c` | اسم الـ function |
| `meson_pmx_get_groups` | `pinctrl-meson.c` | groups الـ function |
| `module_platform_driver` | kernel macro | تسجيل وإلغاء تسجيل الـ driver |
| `MODULE_DEVICE_TABLE` | kernel macro | توليد الـ device ID table |

---

### تصنيف الـ Functions والـ Data

الملف `pinctrl-amlogic-c3.c` هو **data-only driver** — يعني مفيش runtime logic مكتوب فيه صراحةً. كل الـ logic موروثة من الـ Meson pinctrl framework. الملف بيعمل حاجة واحدة بس: **بيعرّف الـ static data tables** اللي الـ framework بيستخدمها، وبيربطها بـ `platform_driver`.

التصنيف:

1. **Pin Descriptor Tables** — تعريف الـ pins الفيزيائية
2. **Pin Arrays** — ربط كل signal بالـ GPIO pin بتاعه
3. **Pin Group Tables** — الـ GPIO groups والـ function groups
4. **Function String Arrays** — مين الـ groups اللي كل function بيدعمها
5. **Function Table** — الـ functions المدعومة كلها
6. **Bank Tables** — GPIO register mapping (pull, dir, in, out, ds)
7. **PMX Bank Tables** — pinmux register mapping
8. **Driver Registration** — الـ `platform_driver` والـ `of_device_id`

---

### الفئة 1: Pin Descriptor Tables

#### الغرض

الـ `pinctrl_pin_desc` هو الوحدة الأساسية في الـ pinctrl subsystem — بيعرّف كل pin بـ `number` وـ `name`. الـ `c3_periphs_pins[]` بيحتوي على **كل الـ pins** اللي الـ SoC C3 بيوفرها في الـ peripheral domain.

#### المستخدَم

```c
static const struct pinctrl_pin_desc c3_periphs_pins[] = {
    MESON_PIN(GPIOE_0),
    MESON_PIN(GPIOE_1),
    /* ... 55 pin total ... */
    MESON_PIN(GPIO_TEST_N),
};
```

#### `MESON_PIN(x)`

```c
#define MESON_PIN(x) PINCTRL_PIN(x, #x)
// expands to:
// { .number = x, .name = #x }
```

**بيعمل إيه:** بيحول الـ enum value للـ GPIO (زي `GPIOB_0 = 5`) لـ `pinctrl_pin_desc` struct فيها الـ number والـ name كـ string تلقائياً بـ stringification.

**الـ pins المعرّفة:** 55 pin موزّعة على banks:
- **GPIOE**: 5 pins (E_0..E_4) — bank صغير للـ I2C/PWM
- **GPIOB**: 15 pins (B_0..B_14) — bank الـ eMMC/NAND الرئيسي
- **GPIOC**: 7 pins (C_0..C_6) — SD card وـ JTAG
- **GPIOX**: 14 pins (X_0..X_13) — bank كبير متعدد الاستخدامات
- **GPIOD**: 7 pins (D_0..D_6) — Ethernet LEDs وـ PWM وغيره
- **GPIOA**: 6 pins (A_0..A_5) — UART/I2C/JTAG
- **GPIO_TEST_N**: 1 pin — test/debug

هذه الـ array بتُسجَّل في الـ `meson_pinctrl_data` وبتروح للـ `pinctrl_desc` اللي الـ `pinctrl_register()` بيستخدمها.

---

### الفئة 2: Pin Arrays (Signal-to-Pin Mapping)

#### الغرض

لكل **signal** ممكن يتعمله mux على الـ hardware، في array بتحدد **أي GPIO pins** بتنتمي لهذا الـ signal. معظم الـ signals في C3 بتاخد pin واحد بس.

#### مثال

```c
/* Bank E func1 */
static const unsigned int pwm_a_pins[]     = { GPIOE_0 };
static const unsigned int pwm_b_pins[]     = { GPIOE_1 };
static const unsigned int i2c2_sda_pins[]  = { GPIOE_2 };
static const unsigned int i2c2_scl_pins[]  = { GPIOE_3 };

/* Bank B func3 - SPI Flash */
static const unsigned int spif_hold_pins[] = { GPIOB_3 };
static const unsigned int spif_mo_pins[]   = { GPIOB_4 };
static const unsigned int spif_mi_pins[]   = { GPIOB_5 };
static const unsigned int spif_clk_pins[]  = { GPIOB_6 };
```

**تفصيل التسمية:** الـ suffix في الاسم بيوضح:
- `_b`, `_c`, `_x`, `_d`, `_a` → الـ bank اللي فيه الـ pin
- `_x0`, `_x8` → أكتر تحديداً للـ pin number في نفس الـ bank
- مثال: `pwm_g_x0_pins[]` يعني PWM-G على GPIOX_0، و`pwm_g_x8_pins[]` يعني PWM-G على GPIOX_8

**ليه arrays وأنا Pin واحد؟** لأن الـ framework بيدعم groups بأكتر من pin (زي الـ eMMC اللي محتاج 8 data lines + clk + cmd + ds + rst). الـ emmc_nand_d0 لـ emmc_nand_ds كلهم arrays من pin واحد بس، لكن ممكن في الـ SoCs التانية يكونوا أكتر.

---

### الفئة 3: Pin Group Tables

#### الغرض

الـ `c3_periphs_groups[]` هو جدول يجمع كل الـ **GPIO groups** (GPIO mode = func 0) والـ **function groups** (alternate functions). ده اللي الـ pinctrl core بيشوفه لما يسأل "إيه الـ groups المتاحة؟"

#### `GPIO_GROUP(gpio)`

```c
#define GPIO_GROUP(gpio)                                        \
    {                                                           \
        .name = #gpio,                                          \
        .pins = (const unsigned int[]){ gpio },                 \
        .num_pins = 1,                                          \
        .data = (const struct meson_pmx_axg_data[]){            \
            PMX_DATA(0),   /* func = 0 → pure GPIO */          \
        },                                                      \
    }
```

**بيعمل إيه:** بينشئ `meson_pmx_group` للـ GPIO mode. الـ `func = 0` معناه إن الـ pin هيشتغل كـ GPIO عادي (no alternate function). في C3 في 55 GPIO group مقابل 55 pin.

**Parameters:**
- `gpio`: اسم الـ GPIO enum (مثلاً `GPIOX_0`)

**لا يرجع قيمة** — macro ينشئ struct literal.

#### `GROUP(grp, f)`

```c
#define GROUP(grp, f)                                           \
    {                                                           \
        .name = #grp,                                           \
        .pins = grp ## _pins,   /* يربطه بالـ array المعرّفة */\
        .num_pins = ARRAY_SIZE(grp ## _pins),                   \
        .data = (const struct meson_pmx_axg_data[]){            \
            PMX_DATA(f),    /* func number 1-6 */               \
        },                                                      \
    }
```

**بيعمل إيه:** بينشئ `meson_pmx_group` لـ alternate function. الـ `f` بيحدد رقم الـ function في الـ hardware mux register (1 لـ 6).

**Parameters:**
- `grp`: اسم الـ group (بيتحول لـ `grp_pins` و `#grp`)
- `f`: function number (1-6) — الـ bits اللي هتتكتب في الـ pinmux register

**مثال:**
```c
GROUP(emmc_clk, 1),    /* GPIOB_8 → func1 = eMMC clock */
GROUP(spif_clk, 3),    /* GPIOB_6 → func3 = SPI Flash clock */
GROUP(uart_a_tx_b, 6), /* GPIOB_0 → func6 = UART-A TX */
```

الـ `c3_periphs_groups[]` في النهاية بيحتوي على 55 GPIO group + حوالي 250+ function group.

---

### الفئة 4: Function String Arrays

#### الغرض

لكل **function** (زي `uart_a`، `spi_a`، `emmc`)، في array من strings بتحدد **كل الـ groups** اللي ممكن تعمل هذه الـ function. ده بيخلي نفس الـ function ممكن يتعمل على أكتر من bank.

#### مثال تفصيلي

```c
static const char * const uart_a_groups[] = {
    /* Bank B func6 */
    "uart_a_tx_b", "uart_a_rx_b", "uart_a_cts_b", "uart_a_rts_b",
    /* Bank C func6 */
    "uart_a_rx_c", "uart_a_tx_c",
    /* Bank X func6 */
    "uart_a_rx_x", "uart_a_tx_x",
    /* Bank D func2 */
    "uart_a_tx_d", "uart_a_rx_d",
};
```

**ده معناه:** UART-A ممكن يتوصل على 4 banks مختلفة (B, C, X, D) — الـ board designer يختار حسب الـ PCB.

```c
static const char * const pwm_g_groups[] = {
    "pwm_g_b",  /* GPIOB_0 func3 */
    "pwm_g_c",  /* GPIOC_0 func5 */
    "pwm_g_d",  /* GPIOD_0 func1 */
    "pwm_g_x0", /* GPIOX_0 func5 */
    "pwm_g_x8", /* GPIOX_8 func5 */
};
```

**PWM-G** ممكن يخرج على 5 pins مختلفة في 4 banks — مرونة عالية.

الـ functions المعرّفة في C3 تشمل:
- **UART**: uart_a, uart_b, uart_c, uart_d, uart_e (وـ uart_f في bank A)
- **I2C**: i2c0..i2c3, i2c_slave
- **PWM**: pwm_a..pwm_n (14 channel)
- **SPI**: spi_a, spi_b, spif (SPI Flash dedicated)
- **Storage**: emmc, nand, sdcard, sdio
- **Audio**: tdm, pdm, mclk_0, mclk_1
- **Display**: lcd (18-bit parallel RGB)
- **Debug**: jtag_a, jtag_b
- **Misc**: eth, ir_in, ir_out, gen_clk, clk12_24, clk_32k_in, gpio_periphs

---

### الفئة 5: Function Table

#### `FUNCTION(fn)`

```c
#define FUNCTION(fn)                                            \
    {                                                           \
        .name    = #fn,                                         \
        .groups  = fn ## _groups,                               \
        .num_groups = ARRAY_SIZE(fn ## _groups),                \
    }
```

**بيعمل إيه:** بينشئ `meson_pmx_func` struct يربط اسم الـ function بالـ array بتاعته من strings.

```c
static const struct meson_pmx_func c3_periphs_functions[] = {
    FUNCTION(gpio_periphs),  /* .name="gpio_periphs", .groups=gpio_periphs_groups */
    FUNCTION(uart_a),
    FUNCTION(uart_b),
    /* ... 46 functions total ... */
    FUNCTION(lcd),
};
```

الـ `c3_periphs_functions[]` بتتسجل في `meson_pinctrl_data` وبيستخدمها الـ core في:
- `pinmux_get_func_count()` → يرجع `num_funcs`
- `pinmux_get_func_name(selector)` → يرجع `funcs[selector].name`
- `pinmux_get_groups(selector)` → يرجع `funcs[selector].groups`

---

### الفئة 6: Bank Tables (GPIO Register Mapping)

#### الغرض

الـ `c3_periphs_banks[]` بيعمل mapping بين كل **GPIO bank** والـ **hardware registers** اللي بتتحكم فيه (pull enable، pull direction، data direction، output value، input value، drive strength).

#### `BANK_DS(...)`

```c
#define BANK_DS(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib, dsr, dsb) \
    {                                                           \
        .name      = n,                                         \
        .first     = f,                                         \
        .last      = l,                                         \
        .irq_first = fi,                                        \
        .irq_last  = li,                                        \
        .regs = {                                               \
            [MESON_REG_PULLEN] = { per, peb },  /* pull enable */ \
            [MESON_REG_PULL]   = { pr,  pb  },  /* pull up/down */ \
            [MESON_REG_DIR]    = { dr,  db  },  /* direction */  \
            [MESON_REG_OUT]    = { or,  ob  },  /* output value */ \
            [MESON_REG_IN]     = { ir,  ib  },  /* input value */ \
            [MESON_REG_DS]     = { dsr, dsb },  /* drive strength */ \
        },                                                      \
    }
```

**Parameters:**
- `n`: اسم الـ bank ("X", "D", "E", "C", "B", "A", "TEST_N")
- `f`, `l`: أول وآخر pin في الـ bank
- `fi`, `li`: أول وآخر hwirq number للـ bank (للـ GPIO interrupts)
- `per/peb`: register offset / bit start للـ pull enable
- `pr/pb`: register / bit للـ pull direction (up/down)
- `dr/db`: register / bit للـ data direction (input/output)
- `or/ob`: register / bit لكتابة الـ output
- `ir/ib`: register / bit لقراءة الـ input
- `dsr/dsb`: register / bit لـ drive strength

**مثال حقيقي من C3:**

```c
BANK_DS("X",  GPIOX_0,  GPIOX_13,  40, 53,
    0x03, 0,   /* pullen: reg=0x03, bit_start=0 */
    0x04, 0,   /* pull:   reg=0x04, bit_start=0 */
    0x02, 0,   /* dir:    reg=0x02, bit_start=0 */
    0x01, 0,   /* out:    reg=0x01, bit_start=0 */
    0x00, 0,   /* in:     reg=0x00, bit_start=0 */
    0x07, 0),  /* ds:     reg=0x07, bit_start=0 */
```

الـ framework بيحسب رقم الـ bit الفعلي كالتالي:
```
bit = bit_start + (pin_number - bank_first)
```

يعني `GPIOX_5` → `bit = 0 + (5 - 0) = 5` في register `0x07` للـ drive strength.

**Pattern الـ C3:** كل bank عنده register offset `0xN0` (مثلاً Bank B = `0x50`, Bank A = `0x60`). الـ registers داخل كل bank:
- `+0` = input
- `+1` = output
- `+2` = direction
- `+3` = pull enable
- `+4` = pull
- `+7` = drive strength

الـ IRQ range:
- GPIOB: 0–14
- GPIOC: 15–21
- GPIOE: 22–26
- GPIOA: 27–32
- GPIOD: 33–39
- GPIOX: 40–53
- TEST_N: 54

---

### الفئة 7: PMX Bank Tables

#### الغرض

الـ `c3_periphs_pmx_banks[]` بيحدد أين في الـ regmap موجودة الـ **pinmux function selection registers** لكل bank.

#### `BANK_PMX(n, f, l, r, o)`

```c
#define BANK_PMX(n, f, l, r, o)    \
    {                               \
        .name   = n,                \
        .first  = f,                \
        .last   = l,                \
        .reg    = r,   /* register offset في الـ mux regmap */ \
        .offset = o,   /* bit offset للـ pin الأول */          \
    }
```

**Parameters:**
- `n`: اسم الـ bank
- `f`, `l`: أول وآخر pin
- `r`: register offset في الـ mux regmap (بالـ word offset)
- `o`: bit position للـ pin الأول في الـ register

**مثال:**

```c
static const struct meson_pmx_bank c3_periphs_pmx_banks[] = {
    /* name     first       last         reg   offset */
    BANK_PMX("B", GPIOB_0,  GPIOB_14,   0x00, 0),
    BANK_PMX("X", GPIOX_0,  GPIOX_13,   0x03, 0),
    BANK_PMX("C", GPIOC_0,  GPIOC_6,    0x09, 0),
    BANK_PMX("A", GPIOA_0,  GPIOA_5,    0x0b, 0),
    BANK_PMX("D", GPIOD_0,  GPIOD_6,    0x10, 0),
    BANK_PMX("E", GPIOE_0,  GPIOE_4,    0x12, 0),
    BANK_PMX("TEST_N", GPIO_TEST_N, GPIO_TEST_N, 0x02, 0),
};
```

الـ `meson_axg_pmx_ops` بيستخدم الـ `meson_pmx_axg_data.func` الموجودة في كل group مع الـ bank info ده لحساب:
```
mux_reg = bank->reg + (pin - bank->first) * 4 / 32
mux_bit = (pin - bank->first) * 4 % 32
```

كل pin بياخد **4 bits** في الـ mux register → 8 pins لكل 32-bit word → هذا هو لوجو الـ layout.

الـ `meson_axg_pmx_data` بيجمع الـ PMX banks:

```c
static const struct meson_axg_pmx_data c3_periphs_pmx_banks_data = {
    .pmx_banks     = c3_periphs_pmx_banks,
    .num_pmx_banks = ARRAY_SIZE(c3_periphs_pmx_banks),
};
```

---

### الفئة 8: Driver Registration

#### `meson_pinctrl_data` — الـ Master Configuration

```c
static const struct meson_pinctrl_data c3_periphs_pinctrl_data = {
    .name      = "periphs-banks",
    .pins      = c3_periphs_pins,
    .groups    = c3_periphs_groups,
    .funcs     = c3_periphs_functions,
    .banks     = c3_periphs_banks,
    .num_pins  = ARRAY_SIZE(c3_periphs_pins),
    .num_groups = ARRAY_SIZE(c3_periphs_groups),
    .num_funcs = ARRAY_SIZE(c3_periphs_functions),
    .num_banks = ARRAY_SIZE(c3_periphs_banks),
    .pmx_ops   = &meson_axg_pmx_ops,
    .pmx_data  = &c3_periphs_pmx_banks_data,
    .parse_dt  = &meson_a1_parse_dt_extra,
};
```

**بيعمل إيه:** ده الـ "glue struct" اللي بيجمع كل الـ static data في وحدة واحدة. الـ `meson_pinctrl_probe()` بياخد pointer ليه من `of_device_id.data`.

**Fields:**
- `.pins / .num_pins` → الـ `pinctrl_desc` اللي بيتسجل في الـ pinctrl core
- `.groups / .num_groups` → الـ groups المتاحة للـ DT `pinctrl-*` properties
- `.funcs / .num_funcs` → الـ functions المتاحة للـ DT `function` property
- `.banks / .num_banks` → لعمليات الـ GPIO (direction، value، pull، DS)
- `.pmx_ops` → الـ `pinmux_ops` — هنا `meson_axg_pmx_ops` اللي بيتعامل مع الـ 4-bit mux
- `.pmx_data` → passed للـ ops كـ private data
- `.parse_dt` → callback لـ parse الـ DT node وإعداد الـ regmaps

**الـ parse_dt = meson_a1_parse_dt_extra:**
الـ C3 بيستخدم نفس الـ parse function بتاعة الـ A1 لأن كلهم بيستخدموا نفس الـ regmap layout (single-regmap design للـ peripherals). بتعمل:
1. تقرأ الـ `reg-names` من الـ DT
2. تعمل regmap للـ GPIO registers
3. تعمل regmap للـ pinmux registers
4. ممكن تعمل regmap للـ drive-strength لو موجود

#### `c3_pinctrl_dt_match` — الـ OF Match Table

```c
static const struct of_device_id c3_pinctrl_dt_match[] = {
    {
        .compatible = "amlogic,c3-periphs-pinctrl",
        .data       = &c3_periphs_pinctrl_data,
    },
    { }  /* sentinel */
};
MODULE_DEVICE_TABLE(of, c3_pinctrl_dt_match);
```

**بيعمل إيه:** بيحدد الـ compatible string اللي الـ kernel بيستخدمها للـ match مع الـ DT node. لما الـ kernel يلاقي `compatible = "amlogic,c3-periphs-pinctrl"` في الـ DT، بيشغّل الـ probe ويمرر `c3_periphs_pinctrl_data` كـ `platform_data`.

**`MODULE_DEVICE_TABLE`:** بيولّد الـ `__mod_of__table` section في الـ kernel module، اللي بيستخدمها `modprobe` وـ `udev` للـ autoloading.

#### `c3_pinctrl_driver` — الـ Platform Driver

```c
static struct platform_driver c3_pinctrl_driver = {
    .probe  = meson_pinctrl_probe,
    .driver = {
        .name           = "amlogic-c3-pinctrl",
        .of_match_table = c3_pinctrl_dt_match,
    },
};
module_platform_driver(c3_pinctrl_driver);
```

**بيعمل إيه:** بيسجّل الـ `platform_driver` في الـ kernel. الـ `module_platform_driver()` macro بيعمل `module_init()` و`module_exit()` تلقائياً يستخدموا `platform_driver_register()` و`platform_driver_unregister()`.

**الـ `.probe = meson_pinctrl_probe`:** ده الـ shared probe function. تدفق تنفيذه:

```
meson_pinctrl_probe()
├── of_match_device()          → يجيب c3_periphs_pinctrl_data
├── data->parse_dt(pc)         → meson_a1_parse_dt_extra()
│   ├── يعمل reg_mux regmap
│   ├── يعمل reg_gpio regmap
│   └── يعمل reg_ds regmap (لو في DS support)
├── meson_gpiolib_register()   → يسجّل gpio_chip
├── pinctrl_register()         → يسجّل pinctrl_desc
│   ├── pins = c3_periphs_pins
│   ├── pctlops = &meson_pctrl_ops
│   └── pmxops = &meson_axg_pmx_ops
└── return 0
```

---

### Flow Chart: كيف يتم تطبيق الـ Pinmux

```
DT: pinctrl-0 = <&uart_a_pins>;
       ↓
pinctrl_select_state()
       ↓
meson_pmx_set()  [in meson_axg_pmx_ops]
       ↓
يبحث في c3_periphs_groups[] عن group "uart_a_tx_b"
       ↓
يجيب group->data->func = 6
       ↓
يبحث في c3_periphs_pmx_banks[] عن bank يحتوي GPIOB_0
       ↓
يجيب bank "B": reg=0x00, offset=0
       ↓
bit_pos = (GPIOB_0 - GPIOB_0) * 4 = 0
       ↓
regmap_update_bits(reg_mux, 0x00, 0xF << 0, 6 << 0)
       ↓
الـ hardware يوجّه GPIOB_0 لـ UART-A TX
```

---

### مقارنة بين الـ Data Structures

| الـ Struct | الغرض | من يستخدمها |
|-----------|--------|-------------|
| `pinctrl_pin_desc` | تعريف الـ pin (number + name) | pinctrl core |
| `meson_pmx_group` | group من pins + function number | meson_axg_pmx_ops |
| `meson_pmx_func` | function name + group names | pinctrl core (pmx) |
| `meson_bank` | GPIO register descriptors | meson GPIO ops |
| `meson_pmx_bank` | mux register location | meson_axg_pmx_ops |
| `meson_axg_pmx_data` | مجموعة الـ pmx_banks | meson_axg_pmx_ops |
| `meson_pinctrl_data` | كل الـ data مجمّعة | meson_pinctrl_probe |
| `meson_pinctrl` | runtime state | كل الـ ops |

---

### ملاحظات مهمة على الـ Implementation

1. **الـ Driver ده Data-Only:** مفيش ولا function واحدة بتُكتب runtime code في الملف ده. كل الـ intelligence موجودة في:
   - `pinctrl-meson.c` (GPIO ops, probe, pmx common)
   - `pinctrl-meson-axg-pmx.c` (mux set/get ops)
   - `pinctrl-meson-a1.c` (parse_dt)

2. **الـ 4-bit Mux Fields:** الـ C3 بيستخدم 4 bits لكل pin في الـ mux register (functions 0-6، مع 0 = GPIO). ده بيتطابق مع الـ AXG/A1 design.

3. **الـ Regmap Separation:** في الـ C3 في regmaps منفصلة للـ GPIO control والـ pinmux control والـ drive strength — مش زي الـ Meson8 القديم اللي كان كل حاجة في regmap واحد.

4. **الـ IRQ Numbers:** الـ irq_first/irq_last في `BANK_DS` بتمثل الـ **hardware IRQ offsets** داخل الـ GPIO interrupt controller — مش kernel IRQ numbers.

5. **الـ Drive Strength Values:** الـ C3 بيدعم 4 قيم: 500µA، 2500µA، 3000µA، 4000µA (معرّفة في `meson_pinconf_drv` enum).
## Phase 5: دليل الـ Debugging الشامل

الـ driver ده هو `pinctrl-amlogic-c3.c` — بيعمل register وبيدير الـ pin controller والـ GPIO للـ Amlogic C3 SoC. فيه 7 banks (X, D, E, C, B, A, TEST_N)، كل bank بيتحكم في pull-enable، pull، direction، output، input، وdrive-strength عن طريق regmap. الـ mux registers منفصلة (reg_mux) عن الـ GPIO registers (reg_gpio).

---

### Software Level

#### 1. debugfs entries

الـ pinctrl subsystem بيسجل entries تحت `/sys/kernel/debug/pinctrl/`:

```bash
# اعرف كل الـ pinctrl devices المتاحة
ls /sys/kernel/debug/pinctrl/

# الناتج المتوقع على C3:
# pinctrl-maps  periphs-banks  ...

# اقرأ الـ pin groups المعرّفة في الـ driver
cat /sys/kernel/debug/pinctrl/periphs-banks/pins
# يطلع شبه:
# registered pins: 55
# pin 0 (GPIOE_0) ...
# pin 5 (GPIOB_0) ...

# اعرف الـ pingroups
cat /sys/kernel/debug/pinctrl/periphs-banks/pingroups
# يطلع كل group باسمها والـ pins جوّاها

# اعرف الـ pinmux functions
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-functions

# اعرف إيه اللي مُفعَّل دلوقتي
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins
# مثال:
# pin 0 (GPIOE_0): (MUX UNCLAIMED) (GPIO UNCLAIMED)
# pin 5 (GPIOB_0): mmc0 (GPIO UNCLAIMED)

# اقرأ الـ pin configs (pull, drive-strength)
cat /sys/kernel/debug/pinctrl/periphs-banks/pinconf-pins
cat /sys/kernel/debug/pinctrl/periphs-banks/pinconf-groups
```

#### 2. sysfs entries

```bash
# قائمة الـ GPIO chips
ls /sys/class/gpio/
# gpiochip<N> — رقم N بيتغير حسب الترتيب في النظام

# اعرف base و ngpio للـ chip
cat /sys/class/gpio/gpiochip<N>/base
cat /sys/class/gpio/gpiochip<N>/ngpio
cat /sys/class/gpio/gpiochip<N>/label
# label بيطلع "periphs-banks" أو اسم الـ bank

# GPIO sysfs الكلاسيكي (deprecated بس مفيد للـ debug)
echo <gpio_num> > /sys/class/gpio/export
cat /sys/class/gpio/gpio<N>/direction
cat /sys/class/gpio/gpio<N>/value

# الـ gpiod character device (الطريقة الحديثة)
ls /dev/gpiochip*
gpiodetect        # يطلع كل الـ chips
gpioinfo gpiochip0  # يطلع حالة كل pin في الـ chip

# الـ regmap debugfs — مهم جداً لقراءة الـ registers
ls /sys/kernel/debug/regmap/
# كل regmap بيظهر بـ address بتاعه

cat /sys/kernel/debug/regmap/<addr>-<name>/registers
# يطلع dump لكل الـ registers بقيمها
```

#### 3. ftrace — tracepoints وevents

```bash
# mount tracefs لو مش موجود
mount -t tracefs nodev /sys/kernel/tracing

# اعرف الـ events المتاحة لـ pinctrl وgpio
grep -r "pinctrl\|gpio" /sys/kernel/tracing/available_events

# فعّل pinctrl events
echo 1 > /sys/kernel/tracing/events/pinctrl/enable

# فعّل gpio events
echo 1 > /sys/kernel/tracing/events/gpio/enable

# فعّل specific events
echo 1 > /sys/kernel/tracing/events/pinctrl/pinctrl_setting_prop_add/enable
echo 1 > /sys/kernel/tracing/events/pinctrl/pinctrl_setting_mux_enable/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/tracing/tracing_on
# ... اعمل الـ operation اللي عايز تتبعها ...
cat /sys/kernel/tracing/trace

# فعّل function tracing لـ meson pinctrl functions
echo "meson_pinctrl*" > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace

# تتبع meson_axg_pmx_ops functions
echo "meson_axg*" >> /sys/kernel/tracing/set_ftrace_filter
```

#### 4. printk / dynamic debug

```bash
# فعّل dynamic debug لكل ملفات الـ meson pinctrl
echo "file pinctrl-amlogic-c3.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file pinctrl-meson.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file pinctrl-meson-axg-pmx.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل لكل الـ module
echo "module pinctrl_amlogic_c3 +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل للـ pinctrl subsystem كله
echo "module pinctrl +p" > /sys/kernel/debug/dynamic_debug/control

# اعمل verbose أكتر (+pflmt = print, func, line, module, thread)
echo "file pinctrl-meson.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ dynamic debug المفعّلة
cat /sys/kernel/debug/dynamic_debug/control | grep pinctrl

# شوف الـ kernel log
dmesg -w | grep -i "pinctrl\|gpio\|meson\|amlogic"
```

#### 5. Kernel config options للـ debugging

| Config | الوصف |
|--------|-------|
| `CONFIG_DEBUG_PINCTRL` | يفعّل verbose logging في الـ pinctrl core |
| `CONFIG_GPIOLIB_IRQCHIP` | يدعم الـ GPIO IRQ chip — لازم للـ interrupt debug |
| `CONFIG_GPIO_SYSFS` | يفعّل `/sys/class/gpio` |
| `CONFIG_DEBUG_GPIO` | يضيف checks وwarnings في الـ gpiolib |
| `CONFIG_GENERIC_PINCONF` | الـ pin config generic layer — لازم موجود |
| `CONFIG_PINMUX` | الـ pinmux core — بدونه مفيش mux |
| `CONFIG_DEBUG_FS` | يفعّل debugfs — لازم للـ pinctrl debug entries |
| `CONFIG_REGMAP_DEBUGFS` | يفعّل register dump عبر debugfs |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug` / `dev_dbg` بشكل dynamic |
| `CONFIG_KALLSYMS` | يفيد في قراءة stack traces |
| `CONFIG_LOCKDEP` | يكشف deadlocks في الـ lock hierarchy |

```bash
# تحقق من الـ configs الحالية
zcat /proc/config.gz | grep -E "CONFIG_(DEBUG_PINCTRL|DEBUG_GPIO|REGMAP_DEBUGFS|GPIO_SYSFS)"
```

#### 6. أدوات subsystem-specific

```bash
# gpio-tools (من libgpiod)
gpiodetect                    # كل الـ GPIO chips
gpioinfo gpiochip0            # حالة كل الـ lines في chip
gpioget gpiochip0 5           # اقرأ GPIOB_0 (مثال)
gpioset gpiochip0 5=1         # اكتب على الـ GPIO
gpiomon gpiochip0 5           # راقب التغييرات

# pinctrl-utils (لو موجودة في الـ userspace)
# بدلها استخدم debugfs مباشرة

# regmap dump عبر devmem2 أو /dev/mem
# الـ base address للـ periphs GPIO على C3 من الـ DT
# مثال: base = 0xfe004000 (راجع DT)
devmem2 0xfe004000 w          # قرأ reg_gpio base
devmem2 0xfe004400 w          # bank B registers (offset 0x50*4)

# فحص الـ clock (لأن بعض الـ pin functions محتاجة clock)
cat /sys/kernel/debug/clk/clk_summary | grep gpio
```

#### 7. جدول رسائل الـ error الشائعة

| رسالة في dmesg | المعنى | الحل |
|----------------|--------|-------|
| `pinctrl-amlogic-c3: probe failed` | فشل الـ probe بالكامل | افحص الـ DT compatible وirq وregmap |
| `could not get regmap for periphs` | الـ syscon/regmap مش متاح | تأكد أن الـ parent node `amlogic,meson-c3-periphs` موجود في الـ DT |
| `pin X already requested` | pin مطلوب من أكتر من driver | افحص `cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins` |
| `invalid pin number` | pin number خارج الـ range | راجع الـ DT node والـ pin numbers في `amlogic-c3-gpio.h` |
| `GPIO chip registration failed` | تعارض في الـ gpio_chip base | `gpiodetect` وتأكد من عدم تكرار الـ label |
| `pinmux: pin X already used by Y` | تعارض في الـ mux | راجع الـ DT pinctrl nodes — اتنين بيطلبوا نفس الـ pin |
| `failed to parse 'pinctrl-0'` | خطأ في الـ DT pinctrl property | تأكد من وجود `pinctrl-names` و`pinctrl-0` صح |
| `regmap_write failed` | كتابة على register فشلت | افحص الـ clock وpower domain للـ GPIO block |
| `irq_domain_add failed` | فشل في إنشاء الـ IRQ domain | افحص الـ `interrupts` property في الـ DT |
| `request_irq failed for GPIO X` | فشل request الـ IRQ | افحص الـ irq_first/irq_last في `BANK_DS` والـ GIC config |

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في meson_pinctrl_probe — لو reg_gpio مش اتعمل صح */
if (IS_ERR(pc->reg_gpio)) {
    WARN_ON(1); /* إيه اللي رجع من regmap_init؟ */
    dump_stack();
    return PTR_ERR(pc->reg_gpio);
}

/* في meson_axg_pmx_set_mux — لو الـ func value مش متوقع */
WARN_ON(data->func > 7); /* الـ C3 بيدعم func 0-6 بس */

/* لو بنشك إن pin اتطلب من thread تاني */
WARN_ON(!mutex_is_locked(&pc->pcdev->mutex));

/* في meson_gpio_direction_output/input — لو bank مش لقيه */
WARN_ON_ONCE(bank == NULL);
dump_stack(); /* عشان نشوف مين طلب الـ GPIO ده */
```

---

### Hardware Level

#### 1. التحقق إن الـ hardware state بيطابق الـ kernel state

```bash
# خطوة 1: اعرف الـ expected register value من الـ kernel
cat /sys/kernel/debug/regmap/*/registers
# دوّر على الـ regmap اللي اسمه زي "fe004000.pinctrl" أو ما شابه

# خطوة 2: قارن بالـ actual hardware value عبر devmem2
# مثال: bank B pullen register — reg=0x53, base افترضنا 0xfe004000
# كل reg هو word offset: 0x53 * 4 = 0x14C
devmem2 0xfe00414C w
# لو القيمة مختلفة عن الـ regmap → في مشكلة cache أو concurrent access

# خطوة 3: قارن direction الـ pin مع الـ GPIO value
# bank B direction: reg=0x52 → offset 0x148
devmem2 0xfe004148 w  # bit 0 = GPIOB_0 direction (0=output, 1=input)

# خطوة 4: تحقق من الـ mux register
# bank B mux: reg=0x00 → offset 0x000
# كل pin محتاج 4 bits → GPIOB_0 في bits [3:0]
devmem2 0xfe000000 w
```

#### 2. تقنيات الـ Register Dump

```bash
# devmem2 (أسهل طريقة)
# تثبيت: apt install devmem2 أو build من المصدر

# اقرأ bank X registers كلها (offset 0x00 → 0x07 * 4 = 0x1C)
# base address من الـ DT — مثال 0xfe004000

# GPIO X registers:
devmem2 0xfe004000 w  # in  reg (0x00)
devmem2 0xfe004004 w  # out reg (0x01)
devmem2 0xfe004008 w  # dir reg (0x02)
devmem2 0xfe00400C w  # pullen (0x03)
devmem2 0xfe004010 w  # pull   (0x04)
devmem2 0xfe00401C w  # ds     (0x07)

# loop لـ dump سريع للـ bank
BASE=0xfe004000
for i in 0 4 8 12 16 20 24 28; do
    printf "0x%08X: " $((BASE + i))
    devmem2 $((BASE + i)) w 2>/dev/null | grep -oP '0x[0-9A-Fa-f]+'
done

# /dev/mem مباشرة (لو devmem2 مش موجود)
python3 -c "
import mmap, struct
f = open('/dev/mem', 'rb')
m = mmap.mmap(f.fileno(), 0x100, offset=0xfe004000)
for i in range(0, 0x20, 4):
    val = struct.unpack('<I', m[i:i+4])[0]
    print(f'  +0x{i:02X}: 0x{val:08X}')
m.close(); f.close()
"

# io utility (من package ioport أو busybox)
io -4 -r 0xfe004000
```

#### 3. Logic Analyzer / Oscilloscope

```
نصائح عملية:
─────────────────────────────────────────────────────────────

1. SDIO/SDCARD debugging (GPIOX_0-5 / GPIOC_0-6):
   - اربط الـ probe على CLK (GPIOX_4 أو GPIOC_4)
   - trigger على rising edge
   - تحقق من timing بين CMD وDATA
   - الـ drive-strength المنخفض بيسبب slow edges → زود الـ DS

2. UART debugging (أي UART group):
   - اربط على TX line
   - set trigger level = 1.8V (الـ C3 IO voltage)
   - قيس الـ bit period: 1/baudrate
   - لو في glitches → افحص الـ pull resistor الـ setting

3. I2C debugging (GPIOE_2/3 وغيرها):
   - راقب SDA + SCL مع الـ protocol decode
   - لو مفيش ACK → pin مش configured صح أو مفيش pull-up
   - تحقق من الـ drive-strength: MESON_PINCONF_DRV_2500UA للـ I2C عادةً

4. SPI debugging (GPIOB أو GPIOX):
   - 4-channel: MOSI, MISO, CLK, SS
   - تحقق من polarity وphase مع الـ device datasheet
   - الـ spif (SPI flash) على GPIOB_3-7 + GPIOB_13

5. نصيحة عامة للـ C3:
   - الـ IO voltage هو 1.8V أو 3.3V حسب الـ bank
   - bank B غالباً 1.8V (eMMC/NAND)
   - bank C (sdcard) ممكن 3.3V
   - اتأكد من الـ oscilloscope probe ground وlevels
```

#### 4. مشاكل الـ hardware الشائعة وأنماطها في الـ kernel log

| مشكلة hardware | نمط في dmesg | التشخيص |
|----------------|-------------|---------|
| الـ GPIO لا يستجيب | لا يوجد log — الـ driver يعمل بس الـ hardware صامت | `devmem2` على input reg — لو دايماً 0 → الـ IO power domain مش شغال |
| تعارض mux (اتنين peripherals على نفس الـ pin) | `pin X already used` أو output مش متوقع | `cat pinmux-pins` — شوف مين hold الـ pin |
| drive-strength منخفض جداً | SDIO أو SPI errors عند speed عالية | زود `drive-strength` في الـ DT pinctrl node |
| pull غلط على UART RX | framing errors، junk data | تحقق من `bias-pull-up` في الـ DT |
| الـ interrupt مش بييجي | لا يوجد IRQ في `cat /proc/interrupts` | افحص `irq_first`/`irq_last` في `BANK_DS` وDT interrupts property |
| الـ eMMC مش بيـboot | `mmc0: error -110 whilst initialising MMC card` | افحص GPIOB_0-11 mux = func1، وpull-up على CMD |

#### 5. Device Tree Debugging

```bash
# اعرف الـ DT المُحمَّل فعلياً
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | grep -A 30 "periphs-pinctrl"
# أو
cat /proc/device-tree/soc/bus@fe000000/pinctrl@4000/compatible

# تحقق من الـ compatible string
cat /proc/device-tree/soc/bus@fe000000/pinctrl@4000/compatible | xxd
# لازم يكون: amlogic,c3-periphs-pinctrl

# تحقق من الـ reg property (base address)
xxd /proc/device-tree/soc/bus@fe000000/pinctrl@4000/reg

# شوف الـ pinctrl nodes للـ peripherals
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null | \
    grep -B2 -A10 "pinctrl-0\|pinctrl-names"

# تحقق من الـ gpio-controller property
ls /proc/device-tree/soc/bus@fe000000/pinctrl@4000/
# لازم يكون فيه: gpio-controller, #gpio-cells

# تحقق من الـ interrupts property
xxd /proc/device-tree/soc/bus@fe000000/pinctrl@4000/interrupts

# تحقق من الـ bank register offsets (من الـ driver)
# BANK_DS("B", GPIOB_0, GPIOB_14, 0, 14,
#   pullen=0x53, pull=0x54, dir=0x52, out=0x51, in=0x50, ds=0x57)
# كل register offset ده * 4 = byte offset من الـ base

# تحقق DT syntax
dtc -I dts -O dtb your-c3-board.dts -o /dev/null
# لو فيه errors → هتظهر هنا

# تحقق من إن الـ pinctrl node بيـmatch الـ driver
grep "amlogic,c3-periphs-pinctrl" /sys/bus/platform/drivers/amlogic-c3-pinctrl/*/of_node/compatible 2>/dev/null
```

---

### Practical Commands

#### جاهز للـ copy — كل سيناريو

```bash
# ══════════════════════════════════════════════
# 1. اعرف حالة كل الـ pins دلوقتي
# ══════════════════════════════════════════════
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | head -60

# مثال الناتج:
# pin 0 (GPIOE_0): pwm (GPIO UNCLAIMED)
# pin 5 (GPIOB_0): mmc0 (GPIO UNCLAIMED)
# pin 14 (GPIOB_9): (MUX UNCLAIMED) (GPIO UNCLAIMED)
# التفسير: pin 0 محجوز لـ pwm، pin 5 محجوز لـ mmc0، pin 14 حر

# ══════════════════════════════════════════════
# 2. اعرف الـ groups المتاحة لـ function معينة
# ══════════════════════════════════════════════
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-functions | grep -A5 "uart_a"

# مثال الناتج:
# function: uart_a, groups = [ uart_a_tx_b uart_a_rx_b uart_a_cts_b ... ]

# ══════════════════════════════════════════════
# 3. تحقق من الـ pin config (pull, drive-strength)
# ══════════════════════════════════════════════
cat /sys/kernel/debug/pinctrl/periphs-banks/pinconf-pins | grep -A3 "GPIOB_10"

# ══════════════════════════════════════════════
# 4. فعّل كل الـ debug للـ pinctrl subsystem
# ══════════════════════════════════════════════
echo "module pinctrl_amlogic_c3 +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "file pinctrl-meson.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "file pinctrl-meson-axg-pmx.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
dmesg -C  # امسح الـ log القديم
# ... اعمل الـ operation ...
dmesg | grep -E "pinctrl|meson|amlogic"

# ══════════════════════════════════════════════
# 5. dump الـ regmap registers (GPIO bank B كمثال)
# ══════════════════════════════════════════════
# افتح الـ regmap المناسب
REGMAP_DIR=$(ls /sys/kernel/debug/regmap/ | grep -i "gpio\|pinctrl" | head -1)
cat /sys/kernel/debug/regmap/$REGMAP_DIR/registers

# مثال الناتج:
# 50: 00000000   ← bank B in register
# 51: 00000000   ← bank B out register
# 52: 00007fff   ← bank B dir register (bits 0-14 = inputs)
# 53: 00007fff   ← bank B pullen (كل الـ pull-enable فعّال)
# 54: 00007fff   ← bank B pull (كل الـ pull-up)
# 57: aaaaaaaa   ← bank B drive-strength (2500uA لكل pin)

# ══════════════════════════════════════════════
# 6. ftrace لتتبع mux operations
# ══════════════════════════════════════════════
echo 0 > /sys/kernel/tracing/tracing_on
echo "" > /sys/kernel/tracing/trace
echo "meson_axg_pmx_set_mux" > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
# ... trigger the mux operation (مثلاً modprobe driver بيطلب pin) ...
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace

# مثال الناتج:
#           <...>-1234  [001]  123.456: meson_axg_pmx_set_mux <-pinmux_enable_setting
# التفسير: meson_axg_pmx_set_mux اتنادت من pinmux_enable_setting

# ══════════════════════════════════════════════
# 7. تحقق GPIO interrupt
# ══════════════════════════════════════════════
cat /proc/interrupts | grep -i "gpio\|meson"
# مثال:
#  54:    0    0  GIC-0  54 Edge  periphs-banks:54
# التفسير: GPIO_TEST_N (irq_first=54, irq_last=54) — لو 0 الـ interrupt مجاش

# ══════════════════════════════════════════════
# 8. gpioinfo لقراءة شاملة
# ══════════════════════════════════════════════
gpioinfo
# مثال الناتج:
# gpiochip0 - 55 lines:
#         line   0:  "GPIOE_0"       unused  input  active-high
#         line   5:  "GPIOB_0"       "mmc0"  output active-high [used]
#         line  20:  "GPIOC_0"   "sdcard_d0" output active-high [used]
# التفسير:
#   - unused = مش محجوز من أي driver
#   - [used] = محجوز، اسم الـ consumer بين ""
#   - output/input = الـ direction الحالي
```

```bash
# ══════════════════════════════════════════════
# 9. تشخيص سريع لمشكلة "الـ peripheral مش شغال"
# ══════════════════════════════════════════════

# خطوة 1: هل الـ pin محجوز؟
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep "GPIOB_0"
# لو "(MUX UNCLAIMED)" → الـ driver مش طالب الـ pin → مشكلة DT أو driver probe

# خطوة 2: هل الـ function صح؟
# مثلاً eMMC لازم GPIOB_0 يكون func1 (emmc_nand_d0)
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep "GPIOB_0"
# لو شايف function تانية → في conflict في الـ DT

# خطوة 3: هل الـ pull correct؟
cat /sys/kernel/debug/pinctrl/periphs-banks/pinconf-pins | grep "GPIOB_10"

# خطوة 4: اقرأ الـ register مباشرة وقارن
# bank B mux reg base = 0x00, GPIOB_0 = bits[3:0]
# (الـ base address من الـ DT — اقرأها من /proc/device-tree)
BASE=$(cat /proc/device-tree/soc/bus@fe000000/pinctrl@4000/reg | \
       python3 -c "import sys,struct; d=sys.stdin.buffer.read(); print(hex(struct.unpack('>II',d[:8])[0]))")
echo "Pinctrl base: $BASE"
devmem2 $BASE w  # mux register bank B
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART مش شغال على Android TV Box بـ Amlogic C3

#### العنوان
**الـ debug console مش بتظهر output على TV box بعد تغيير الـ DTS**

#### السياق
شركة بتعمل Android TV box بـ **Amlogic C3 SoC**. الـ engineering team غيّرت الـ PCB layout في الـ revision الجديدة وحرّكت الـ UART console من bank B لـ bank X عشان يبقى قريب من الـ USB-to-UART chip على الـ board الجديدة.

#### المشكلة
بعد تعديل الـ DTS، الـ serial console بطّلت تشتغل خالص. مفيش `printk` بيظهر، والـ U-Boot بيبوت بدون output. الـ tester حاسس إن الـ UART نفسه شغّال لأن الـ driver بيتحمّل بدون error.

#### التحليل
الـ engineer فتح الـ DTS وشاف الـ node بيستخدم:

```dts
/* DTS قبل التعديل — bank B */
&uart_a {
    pinctrl-names = "default";
    pinctrl-0 = <&uart_a_b_pins>;
    status = "okay";
};
```

بعد التعديل كتبوا:
```dts
/* DTS بعد التعديل — محاولة استخدام bank X */
&uart_a {
    pinctrl-names = "default";
    pinctrl-0 = <&uart_a_x_pins>;
    status = "okay";
};
```

المشكلة إنهم نسوا يعرّفوا الـ pinctrl node الصح. الـ driver لما بيعمل `probe` بيكلّم `meson_pinctrl_probe()` اللي بتجيب الـ `compatible = "amlogic,c3-periphs-pinctrl"` وتبني كل الـ groups من `c3_periphs_groups[]`.

بص على تعريفات الـ groups في الملف:

```c
/* Bank B func6 — UART_A على bank B */
static const unsigned int uart_a_tx_b_pins[] = { GPIOB_0 };
static const unsigned int uart_a_rx_b_pins[] = { GPIOB_1 };

/* Bank X func6 — UART_A على bank X */
static const unsigned int uart_a_rx_x_pins[] = { GPIOX_0 };
static const unsigned int uart_a_tx_x_pins[] = { GPIOX_1 };
```

وفي `uart_a_groups[]`:
```c
static const char * const uart_a_groups[] = {
    "uart_a_tx_b", "uart_a_rx_b", "uart_a_cts_b", "uart_a_rts_b",
    "uart_a_rx_c", "uart_a_tx_c", "uart_a_rx_x", "uart_a_tx_x",
    "uart_a_tx_d", "uart_a_rx_d",
};
```

الـ group names موجودة، لكن الـ DTS node اللي كتبوه مش هو الـ pinctrl handle الصح. لازم كل group تتعرّف كـ subnode في الـ DTS.

الـ PMX bank بتاعت X:
```c
BANK_PMX("X", GPIOX_0, GPIOX_13, 0x03, 0),
```
يعني الـ mux register للـ bank X هو offset `0x03` وأول pin هو `GPIOX_0`.

الـ `GROUP(uart_a_rx_x, 6)` يعني الـ func number = 6، يعني الـ pinmux bits للـ GPIOX_0 لازم تتكتب بـ value 6.

#### الحل

```dts
/* في ملف الـ DTS الخاص بالـ board */
&periphs_pinctrl {
    uart_a_x_pins: uart-a-x {
        mux {
            groups = "uart_a_rx_x", "uart_a_tx_x";
            function = "uart_a";
            bias-disable;
        };
    };
};

&uart_a {
    pinctrl-names = "default";
    pinctrl-0 = <&uart_a_x_pins>;
    status = "okay";
};
```

للتحقق من إن الـ pinmux اتطبّق صح:
```bash
# اتأكد إن الـ group اتحط على func 6
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep GPIOX_0
# المفروض تشوف: pin 27 (GPIOX_0): uart_a
```

#### الدرس المستفاد
**الـ group name في DTS لازم يطابق بالظبط الـ string في `c3_periphs_groups[]`**. وجود الـ group في `uart_a_groups[]` مش كافي — لازم الـ DTS subnode يعرّف الـ group name الصح زي ما هو معرّف في `GROUP(uart_a_rx_x, 6)`.

---

### السيناريو 2: SPI Flash مش بيتعرف على Industrial Gateway

#### العنوان
**الـ SPI NOR flash مش بيظهر في `/dev/` على gateway صناعي**

#### السياق
شركة بتبني **industrial IoT gateway** بـ Amlogic C3 للـ edge computing. الـ design بيستخدم SPI NOR flash خارجي على bank B عشان يخزّن الـ firmware updates. الـ chip هو W25Q128 متوصّل بـ GPIOB_3 (HOLD)، GPIOB_4 (MOSI)، GPIOB_5 (MISO)، GPIOB_6 (CLK)، GPIOB_7 (WP)، GPIOB_13 (CS).

#### المشكلة
بعد الـ boot، الـ kernel مش بيتعرف على الـ SPI flash. الـ `dmesg` بيقول:
```
spi-nor: probe of spi0.0 failed with error -110
```
والـ `/dev/mtd0` مش موجود.

#### التحليل
الـ engineer شال الـ board وقاس الـ signals بالـ oscilloscope، الـ CLK مفيش عليه أي activity. يعني الـ pinmux مش شغّال.

راجع الملف وشاف:

```c
/* Bank B func3 — SPIF (SPI Flash) */
static const unsigned int spif_hold_pins[]     = { GPIOB_3 };
static const unsigned int spif_mo_pins[]       = { GPIOB_4 };
static const unsigned int spif_mi_pins[]       = { GPIOB_5 };
static const unsigned int spif_clk_pins[]      = { GPIOB_6 };
static const unsigned int spif_wp_pins[]       = { GPIOB_7 };
static const unsigned int spif_cs_pins[]       = { GPIOB_13 };
```

وفي الـ groups:
```c
static const char * const spif_groups[] = {
    "spif_mo", "spif_mi", "spif_wp", "spif_cs",
    "spif_clk", "spif_hold", "spif_clk_loop",
};
```

هنا الـ function هو `spif` مش `spi_a`. الـ engineer كان كاتب في الـ DTS:
```dts
/* غلط — استخدم spi_a بدل spif */
mux {
    groups = "spi_a_mosi_b", "spi_a_miso_b", ...;
    function = "spi_a";
};
```

لكن الـ `spif` هو الـ dedicated SPI flash controller، مش الـ general-purpose `spi_a`. الـ PMX bank:
```c
BANK_PMX("B", GPIOB_0, GPIOB_14, 0x00, 0),
```

الـ `GROUP(spif_clk, 3)` يعني func = 3، بينما `GROUP(spi_a_clk_b, 5)` يعني func = 5. استخدام `spi_a` على نفس الـ pins بيحط الـ func bits بـ value 5 بدل 3، والـ SPIF controller مش بيشتغل.

#### الحل

```dts
&periphs_pinctrl {
    spif_pins: spif-pins {
        mux {
            groups = "spif_mo", "spif_mi", "spif_wp",
                     "spif_cs", "spif_clk", "spif_hold";
            function = "spif";
            bias-disable;
            drive-strength-microamp = <4000>;
        };
    };
};

&spifc {  /* SPIF controller node */
    pinctrl-names = "default";
    pinctrl-0 = <&spif_pins>;
    status = "okay";

    w25q128: flash@0 {
        compatible = "winbond,w25q128", "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <80000000>;
    };
};
```

للتحقق:
```bash
# اتأكد إن الـ pins اتحطوا على func 3
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep GPIOB_6
# المفروض: pin 6 (GPIOB_6): spif

# اتأكد الـ flash اتعرف
cat /proc/mtd
```

#### الدرس المستفاد
**C3 عنده controller منفصل للـ SPI flash اسمه `spif`**، مختلف عن الـ `spi_a` العام. لازم تفرّق بين الاتنين في الـ DTS. الـ `spif_clk_loop` pin هو optional feedback clock للـ high-speed operation، مش لازم تحطّه إلا لو الـ frequency فوق 50MHz.

---

### السيناريو 3: eMMC Boot Fails بعد Enable الـ NAND

#### العنوان
**الـ board بطّلت تـ boot من eMMC بعد محاولة تفعيل NAND على C3**

#### السياق
فريق هندسي بيجرّب يضيف دعم NAND flash لـ **C3-based media player** كـ alternative storage. الـ eMMC موجود على bank B ومشغّال. حاولوا يضيفوا NAND بجانبه في نفس الـ DTS.

#### المشكلة
بعد تعديل الـ DTS وعمل reboot، الـ kernel panic في الـ boot:
```
mmc0: error -110 whilst initialising MMC card
```
الـ eMMC اتعطّل تماماً.

#### التحليل
بص على الـ pin definitions في الملف:

```c
/* Bank B func1 — eMMC */
static const unsigned int emmc_nand_d0_pins[] = { GPIOB_0 };
/* ... */
static const unsigned int emmc_clk_pins[]     = { GPIOB_8 };
static const unsigned int emmc_cmd_pins[]      = { GPIOB_10 };
static const unsigned int emmc_nand_ds_pins[]  = { GPIOB_11 };

/* Bank B func2 — NAND (يشارك نفس الـ pins!) */
static const unsigned int nand_wen_clk_pins[]  = { GPIOB_8 };
static const unsigned int nand_ale_pins[]      = { GPIOB_9 };
static const unsigned int nand_ren_wr_pins[]   = { GPIOB_10 };
static const unsigned int nand_cle_pins[]      = { GPIOB_11 };
static const unsigned int nand_ce0_pins[]      = { GPIOB_12 };
```

المشكلة واضحة: `emmc_groups[]` و `nand_groups[]` **بيشاركوا نفس الـ pins الفيزيائية** على bank B:

```c
static const char * const emmc_groups[] = {
    "emmc_nand_d0", ..., "emmc_clk", "emmc_rst", "emmc_cmd", "emmc_nand_ds",
};

static const char * const nand_groups[] = {
    "emmc_nand_d0", ...,                    /* نفس الـ data pins */
    "nand_wen_clk", "nand_ale", "nand_ren_wr", "nand_cle", "nand_ce0",
};
```

الـ prefix `emmc_nand_` في اسم الـ pin هو hint صريح إن الـ pins مشتركة. لما الـ DTS حاول يفعّل الاتنين، الـ pinmux لـ GPIOB_8 (اللي هو `emmc_clk` بـ func=1 و `nand_wen_clk` بـ func=2) اتكتبت آخر قيمة وغلّطت الـ eMMC.

الـ PMX bank:
```c
BANK_PMX("B", GPIOB_0, GPIOB_14, 0x00, 0),
```
كل pin فيه 4 bits للـ mux function. لما الـ pinctrl subsystem بيطبّق الـ NAND config، بيكتب `func=2` على GPIOB_8 فوق الـ `func=1` بتاعت eMMC.

#### الحل

**C3 بيدعم eMMC أو NAND، مش الاتنين مع بعض على bank B.** الـ hardware design نفسه غلط. لو عايز الاتنين:
- eMMC على bank B
- NAND على controller تاني لو موجود

للتشخيص:
```bash
# شوف الـ pin conflicts
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep GPIOB_8

# شوف الـ pinmux register الحالي
devmem2 0xFE004000 w  # base + 0x00 * 4 — bank B mux reg
```

في الـ DTS، الصح إنك تختار واحد بس:
```dts
/* استخدم eMMC فقط، comment out الـ NAND */
&sd_emmc_b {
    pinctrl-0 = <&emmc_pins>;
    status = "okay";
};

/* &nand { status = "disabled"; }; */
```

#### الدرس المستفاد
**الـ `emmc_nand_` prefix في اسم الـ pin هو مؤشر مهم** إن الـ data bus مشترك. C3 بيدعم eMMC/NAND على نفس الـ bank B كـ mutually exclusive options. لازم تقرر في الـ hardware design مش في الـ software.

---

### السيناريو 4: I2C Sensor مش بيرد على Custom IoT Board

#### العنوان
**حساس temperature على I2C مش بيرد رغم إن الـ wiring صح**

#### السياق
فريق بيبني **IoT sensor board** بـ Amlogic C3 لقياس بيانات بيئية. بيستخدموا I2C لـ BME280 sensor. الـ hardware engineer وصّل الـ SDA/SCL على GPIOX_4 و GPIOX_5.

#### المشكلة
```bash
i2cdetect -y 3
# No devices found — كل الـ addresses بتجاوب بـ --
```
الـ pull-up resistors موجودة، الـ power صح، لكن مفيش response.

#### التحليل
الـ engineer فتح الملف وبصّ على الـ I2C options لـ bank X:

```c
/* Bank X func6 — I2C options */
static const unsigned int i2c3_sda_x_pins[] = { GPIOX_4 };
static const unsigned int i2c3_scl_x_pins[] = { GPIOX_5 };
static const unsigned int i2c1_sda_x_pins[] = { GPIOX_7 };
static const unsigned int i2c1_scl_x_pins[] = { GPIOX_8 };
```

وفي الـ groups:
```c
static const char * const i2c3_groups[] = {
    "i2c3_sda_c", "i2c3_scl_c",
    "i2c3_sda_x", "i2c3_scl_x",   /* GPIOX_4, GPIOX_5 */
    "i2c3_sda_d", "i2c3_scl_d",
};
```

GPIOX_4 وGPIOX_5 هما **I2C3**، مش I2C1 أو I2C0. لكن الـ DTS كان كاتب:

```dts
/* غلط — I2C1 controller مع pins بتاعت I2C3 */
&i2c1 {
    pinctrl-0 = <&i2c1_x_pins>;
    status = "okay";
    bme280@76 { ... };
};
```

لكن `i2c1_x_pins` كان معرّف بـ:
```dts
i2c1_x_pins: i2c1-x {
    mux {
        groups = "i2c3_sda_x", "i2c3_scl_x";  /* pins لـ I2C3 مش I2C1 */
        function = "i2c3";                      /* function غلط كمان */
    };
};
```

في `c3_periphs_groups[]`:
```c
GROUP(i2c3_sda_x, 6),   /* GPIOX_4, func=6 */
GROUP(i2c3_scl_x, 6),   /* GPIOX_5, func=6 */
```

الـ `BANK_PMX("X", GPIOX_0, GPIOX_13, 0x03, 0)` يعني لما الـ pinmux يكتب func=6 على GPIOX_4، ده بيوصّل الـ pin للـ I2C3 controller، مش I2C1. وجود الـ sensor على `&i2c1` node بيعني إن الـ kernel بيكلّم I2C1 hardware block بينما الـ pins متوصلة بـ I2C3.

#### الحل

```dts
&periphs_pinctrl {
    i2c3_x_pins: i2c3-x {
        mux {
            groups = "i2c3_sda_x", "i2c3_scl_x";
            function = "i2c3";
            bias-pull-up;
        };
    };
};

/* استخدم i2c3 مش i2c1 */
&i2c3 {
    pinctrl-names = "default";
    pinctrl-0 = <&i2c3_x_pins>;
    clock-frequency = <400000>;
    status = "okay";

    bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;
    };
};
```

للتشخيص:
```bash
# اتأكد إن الـ pins على الـ function الصح
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep -E "GPIOX_4|GPIOX_5"

# اتأكد إن الـ I2C3 controller موجود
ls /sys/bus/i2c/devices/

# جرّب الـ scan على الـ bus الصح
i2cdetect -y 3
```

#### الدرس المستفاد
**الـ pin location على bank X مش بتحدد الـ I2C controller number.** الـ `i2c3_sda_x` اسمها بيقول إنها I2C3 على bank X. لازم تطابق بين الـ function name في الـ DTS (`function = "i2c3"`) والـ controller node (`&i2c3`).

---

### السيناريو 5: Ethernet LEDs مش بتشتغل على Industrial Gateway

#### العنوان
**مصابيح الـ Ethernet Link/Activity مش بتومض على C3 gateway**

#### السياق
شركة بتبني **industrial Ethernet gateway** بـ Amlogic C3. الـ PHY chip متوصّل والـ network شغّالة، بس الـ LEDs على الـ RJ45 connector مش بتشتغل. المشكلة بدأت بعد إن الـ BSP team عدّلت الـ DTS لـ revision جديدة من الـ board.

#### المشكلة
```bash
# الـ network شغّال
ping 8.8.8.8  # يرد
ip link show eth0  # UP

# لكن الـ LEDs مش بتومض
# customer بيشتكي إن الـ device يبان مش متوصّل
```

#### التحليل
بصّ على الـ pin definitions:

```c
/* Bank D func1 — Ethernet LEDs */
static const unsigned int eth_led_act_pins[]  = { GPIOD_2 };
static const unsigned int eth_led_link_pins[] = { GPIOD_3 };
```

وفي الـ groups:
```c
static const char * const eth_groups[] = {
    "eth_led_act", "eth_led_link",
};
```

الـ bank D descriptor:
```c
BANK_DS("D", GPIOD_0, GPIOD_6, 33, 39,
    0x23, 0,  /* pullen */
    0x24, 0,  /* pull */
    0x22, 0,  /* dir */
    0x21, 0,  /* out */
    0x20, 0,  /* in */
    0x27, 0), /* drive strength */
```

والـ PMX bank:
```c
BANK_PMX("D", GPIOD_0, GPIOD_6, 0x10, 0),
```

الـ engineer شاف إن الـ DTS الجديد بيحط GPIOD_2 وGPIOD_3 كـ GPIO output للـ revision الجديدة اللي كانت بتستخدمهم لـ external LEDs:

```dts
/* revision السابقة — صح */
&periphs_pinctrl {
    eth_led_pins: eth-led {
        mux {
            groups = "eth_led_act", "eth_led_link";
            function = "eth";
            bias-disable;
        };
    };
};
```

لكن revision الجديدة كانت عندها:
```dts
/* revision الجديدة — غلط، حطّت الـ pins كـ GPIO بدل eth function */
eth_led_gpio: eth-led-gpio {
    mux {
        groups = "GPIOD_2", "GPIOD_3";  /* GPIO mode */
        function = "gpio_periphs";
    };
    conf {
        pins = "GPIOD_2", "GPIOD_3";
        output-high;  /* LED دايما ON */
    };
};
```

لما الـ pin بيكون في `gpio_periphs` mode (func=0)، الـ Ethernet MAC controller مش بيقدر يتحكّم في الـ LED signal. الـ `GROUP(eth_led_act, 1)` يعني لازم func=1 في الـ PMX registers عشان الـ MAC يتحكّم في الـ LED.

الـ bank D PMX register offset هو `0x10`. GPIOD_2 هو الـ pin الثالث في الـ bank (index 2)، كل pin بياخد 4 bits. يعني الـ bits اللي المفروض تكون `0001` (func=1) كانت `0000` (GPIO).

#### الحل

```dts
&periphs_pinctrl {
    eth_led_pins: eth-led {
        mux {
            groups = "eth_led_act", "eth_led_link";
            function = "eth";
            bias-disable;
            drive-strength-microamp = <2500>;
        };
    };
};

&eth {
    pinctrl-names = "default";
    pinctrl-0 = <&eth_led_pins>;
    status = "okay";
};
```

للتشخيص:
```bash
# اتأكد إن الـ pins على func eth مش GPIO
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep -E "GPIOD_2|GPIOD_3"
# المفروض: pin X (GPIOD_2): eth

# اتأكد الـ PMX register value بـ devmem
# Bank D PMX reg = base + 0x10 * 4
devmem2 0xFE004040 w  # اقرأ الـ mux register لـ bank D

# اتحقق إن الـ GPIO direction مش بيـ override الـ function
cat /sys/kernel/debug/gpio | grep GPIOD_2
```

#### الدرس المستفاد
**حتى لو الـ network شغّالة، الـ LEDs محتاجة الـ pinmux يكون على الـ `eth` function صراحةً.** لما الـ pin في `gpio_periphs` mode، الـ MAC hardware مش بيقدر يتحكم في الـ output. الـ `eth_led_act` و `eth_led_link` هما pins خاصة بالـ PHY/MAC LED controller، مش GPIO عادية.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي تشرح الـ pinctrl subsystem من الأساس لحد التفاصيل:

| المقالة | الأهمية |
|---------|---------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | مقال تأسيسي بيشرح فكرة الـ pinctrl كـ superset للـ pinmux والـ pinconf |
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | الـ patch الأولى اللي أدخلت الـ subsystem للـ kernel |
| [drivers: create a pin control subsystem v8](https://lwn.net/Articles/460768/) | نسخة متطورة من الـ patch الأصلية |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | الـ documentation الرسمية جوا الـ kernel |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/) | إضافة الـ pin configuration (bias، drive strength) |
| [drivers: pinctrl sleep and idle states in the core](https://lwn.net/Articles/552972/) | إضافة الـ sleep وـ idle states للـ pinctrl core |
| [drivers/pinctrl: Add the concept of an "init" state](https://lwn.net/Articles/615322/) | إضافة الـ init state اللي بتتفعّل قبل الـ probe |
| [Pinctrl driver for Amlogic Meson SoCs](https://lwn.net/Articles/616225/) | مقال مخصص للـ driver الأصلي للـ Meson SoCs |
| [Amlogic Meson pinctrl driver](https://lwn.net/Articles/620822/) | نقاشات الـ mailing list حول الـ Meson pinctrl |
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532909/) | علاقة الـ GPIO بالـ pinctrl |

---

### التوثيق الرسمي في الـ kernel

الـ `Documentation/` جوا الـ kernel بتحتوي على:

```
Documentation/driver-api/pin-control.rst
```

ده الملف الرئيسي اللي بيشرح:
- تعريف الـ **pin controller** وازاي يتسجل
- الـ **pinmux** وازاي تعمل function selection
- الـ **pinconf** وازاي تضبط pull-up/pull-down وـ drive strength
- الـ **device tree bindings** الخاصة بالـ pinctrl
- ازاي الـ GPIO subsystem بيتكامل مع الـ pinctrl

كمان في:

```
Documentation/devicetree/bindings/pinctrl/
```

اللي فيها bindings خاصة بكل vendor، ومنها:

```
Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl.yaml
```

---

### الـ source files المباشرة في الـ kernel

الملفات المركزية اللي الـ `pinctrl-amlogic-c3.c` بيعتمد عليها:

```
drivers/pinctrl/meson/pinctrl-meson.c        # الـ core المشترك لكل Meson SoCs
drivers/pinctrl/meson/pinctrl-meson.h        # الـ structs الأساسية
drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c  # الـ pmx ops للـ AXG/C3
drivers/pinctrl/meson/pinctrl-meson-axg-pmx.h
drivers/pinctrl/core.c                       # الـ pinctrl core نفسه
drivers/pinctrl/pinmux.c                     # الـ pinmux logic
drivers/pinctrl/pinconf.c                    # الـ pin configuration logic
include/linux/pinctrl/pinctrl.h              # الـ public API
include/linux/pinctrl/pinmux.h
include/linux/pinctrl/pinconf.h
include/linux/pinctrl/machine.h
```

---

### نقاشات الـ mailing list

| الرابط | الموضوع |
|--------|---------|
| [PATCH 1/3: pinctrl: add driver for Amlogic Meson SoCs](https://lkml.iu.edu/hypermail/linux/kernel/1410.0/04163.html) | الـ patch الأولى للـ Meson pinctrl driver |
| [pinctrl: add compatible for Amlogic Meson A1](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1571050492-6598-2-git-send-email-qianggui.song@amlogic.com/) | إضافة دعم الـ A1 (نفس الـ infrastructure اللي بيستخدمها الـ C3) |
| [pinctrl: meson: add a new callback for SoCs fixup](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1572004167-24150-3-git-send-email-qianggui.song@amlogic.com/) | إضافة الـ `parse_dt` callback اللي بيستخدمها الـ C3 |
| [pinctrl: meson: add support for GPIO interrupts](https://patchwork.kernel.org/project/linux-amlogic/patch/a86d38af-9103-7557-9986-2dc187815569@gmail.com/) | إضافة الـ GPIO IRQ support |
| [pinctrl: meson: fix drive strength register and bit calculation](https://patchwork.kernel.org/project/linux-amlogic/patch/20200610041329.12948-1-hhk7734@gmail.com/) | fix مهم في حساب الـ DS registers |
| [PATCH v4 3/5: pinctrl: Add driver support for Amlogic SoCs](https://lore.kernel.org/lkml/20250122-amlogic-pinctrl-v4-3-4677b2e18ff1@amlogic.com/) | الـ driver الجديد الموحّد للـ Amlogic |

---

### كتب مقترحة

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 9**: "Communicating with Hardware" — بيغطي الـ I/O port mapping وازاي تقرأ/تكتب في الـ registers، وده أساسي لفهم الـ `BANK_DS` و`BANK_PMX` macros
- **الفصل 14**: "The Linux Device Model" — بيشرح الـ `platform_driver` وـ `of_device_id` اللي الـ C3 driver بيستخدمهم
- متاح مجاناً على: https://lwn.net/Kernel/LDD3/

#### Linux Kernel Development, 3rd Edition — Robert Love
- **الفصل 17**: "Devices and Modules" — بيشرح الـ device model وـ `module_platform_driver`
- مفيد لفهم الـ `MODULE_DEVICE_TABLE` و`of_match_table`

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **الفصل 15**: "Embedded Linux BSP" — بيشرح ازاي تكتب BSP وـ pin configuration لـ SoC جديدة
- مثالي لفهم سياق الـ `pinctrl-amlogic-c3.c` كـ board support code

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds
- الجزء الخاص بالـ device tree وـ pinctrl bindings مهم جداً لفهم الـ `compatible = "amlogic,c3-periphs-pinctrl"`

---

### kernelnewbies.org

صفحات تتبع التغييرات في الـ pinctrl subsystem عبر الإصدارات:

| الرابط | المحتوى |
|--------|---------|
| [Linux_6.17](https://kernelnewbies.org/Linux_6.17) | آخر تحديثات الـ pinctrl في الإصدار الأحدث |
| [Linux_6.14](https://kernelnewbies.org/Linux_6.14) | تغييرات الـ pinctrl في 6.14 |
| [Linux_6.11](https://kernelnewbies.org/Linux_6.11) | تغييرات الـ pinctrl في 6.11 |
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | الصفحة الرئيسية لتتبع كل التغييرات |

---

### elinux.org

| الرابط | المحتوى |
|--------|---------|
| [Pin Control & GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) | عرض تقديمي بيشرح العلاقة بين الـ pinctrl وـ GPIO |
| [EBC Device Trees](https://elinux.org/EBC_Device_Trees) | أمثلة عملية على الـ device tree لـ pinctrl-single |
| [Tests:i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) | مثال على استخدام الـ pinctrl مع الـ I2C، زي الـ `i2c0_sda_e` في الـ C3 |

---

### الـ kernel source على git

```bash
# تاريخ الـ commits في مجلد pinctrl/meson
git log --oneline drivers/pinctrl/meson/

# كل الـ commits الخاصة بالـ C3
git log --oneline drivers/pinctrl/meson/pinctrl-amlogic-c3.c

# مين كتب الـ driver ومتى
git log --follow -p drivers/pinctrl/meson/pinctrl-amlogic-c3.c

# البحث عن الـ compatible string في الـ kernel
git grep "amlogic,c3-periphs-pinctrl"
```

الـ kernel على GitHub (mirror):
- https://github.com/torvalds/linux/tree/master/drivers/pinctrl/meson
- https://github.com/torvalds/linux/blob/master/drivers/pinctrl/meson/pinctrl-amlogic-c3.c

---

### search terms للبحث عن معلومات أكتر

```
# للـ subsystem نفسه
linux kernel pinctrl subsystem internals
pinmux pinconf linux kernel driver development
struct pinctrl_desc linux kernel
pinctrl_register kernel API

# للـ Amlogic/Meson
amlogic meson pinctrl driver linux
pinctrl-meson axg pmx
meson_pinctrl_data struct
meson_bank linux kernel
BANK_DS macro amlogic

# للـ device tree
amlogic c3 device tree pinctrl bindings
"amlogic,c3-periphs-pinctrl" dts example
pinctrl-names pinctrl-0 device tree

# للـ debugging
pinctrl debugfs /sys/kernel/debug/pinctrl
gpio sysfs linux
pinmux status kernel debug
```
## Phase 8: Writing simple module

### الفكرة

**`meson_pinctrl_probe`** هي الـ function اللي بتتشغل لما الـ kernel يلاقي device بيتطابق مع `"amlogic,c3-periphs-pinctrl"` في الـ device tree. بدل ما نـ hook function جوه الـ driver نفسه (اللي static)، هنستخدم **kprobe** على `meson_pinctrl_probe` اللي هي exported ومكتوبة في `pinctrl-meson.c` — ودي أبسط طريقة آمنة نشوف بيها امتى الـ Amlogic C3 pin controller اتسجّل في الـ system.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_c3_pinctrl.c
 *
 * Hook meson_pinctrl_probe() to log when the Amlogic C3
 * pin-controller platform device is being probed.
 */

#include <linux/module.h>       /* MODULE_*, module_init/exit */
#include <linux/kprobes.h>      /* kprobe API                */
#include <linux/platform_device.h> /* struct platform_device */
#include <linux/printk.h>       /* pr_info                   */

/* ------------------------------------------------------------------ */
/*  Pre-handler: runs just before meson_pinctrl_probe() executes       */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * meson_pinctrl_probe(struct platform_device *pdev)
     *
     * On x86_64 : first arg is in RDI
     * On ARM64  : first arg is in X0
     * kprobes gives us pt_regs; we read the first argument register.
     */
#if defined(CONFIG_X86_64)
    struct platform_device *pdev =
        (struct platform_device *)regs->di;
#elif defined(CONFIG_ARM64)
    struct platform_device *pdev =
        (struct platform_device *)regs->regs[0];
#else
    struct platform_device *pdev = NULL;
#endif

    if (pdev)
        pr_info("[c3_pinctrl_probe] probing device: name=\"%s\" id=%d driver_override=%s\n",
                pdev->name,
                pdev->id,
                pdev->driver_override ? pdev->driver_override : "(none)");
    else
        pr_info("[c3_pinctrl_probe] probe triggered (pdev unreadable on this arch)\n");

    return 0; /* 0 = let the original function continue normally */
}

/* ------------------------------------------------------------------ */
/*  Post-handler: runs right after meson_pinctrl_probe() returns       */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * On x86_64 the return value sits in RAX after the call.
     * On ARM64  it sits in X0.
     */
#if defined(CONFIG_X86_64)
    long ret = (long)regs->ax;
#elif defined(CONFIG_ARM64)
    long ret = (long)regs->regs[0];
#else
    long ret = 0;
#endif

    pr_info("[c3_pinctrl_probe] probe returned: %ld (%s)\n",
            ret, ret == 0 ? "SUCCESS" : "FAILURE");
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe c3_kp = {
    .symbol_name = "meson_pinctrl_probe", /* target function by name  */
    .pre_handler = handler_pre,           /* called before the func   */
    .post_handler = handler_post,         /* called after the func    */
};

/* ------------------------------------------------------------------ */
/*  module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init c3_pinctrl_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&c3_kp);
    if (ret < 0) {
        pr_err("[c3_pinctrl_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[c3_pinctrl_probe] kprobe planted at %pS\n",
            c3_kp.addr); /* %pS prints symbol + offset */
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit c3_pinctrl_kprobe_exit(void)
{
    unregister_kprobe(&c3_kp);
    pr_info("[c3_pinctrl_probe] kprobe removed\n");
}

module_init(c3_pinctrl_kprobe_init);
module_exit(c3_pinctrl_kprobe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc example");
MODULE_DESCRIPTION("kprobe on meson_pinctrl_probe for Amlogic C3 pin-controller");
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `<linux/module.h>` | الـ macros الأساسية لأي module: `MODULE_LICENSE`، `module_init`، `module_exit` |
| `<linux/kprobes.h>` | يعرّف `struct kprobe` وكل الـ API بتاعتها: `register_kprobe`، `unregister_kprobe` |
| `<linux/platform_device.h>` | يعرّف `struct platform_device` عشان نقرأ اسم الـ device من أول argument |
| `<linux/printk.h>` | `pr_info` / `pr_err` للـ logging |

---

#### الـ `handler_pre`

ده الـ callback اللي بيتشغل **قبل** ما `meson_pinctrl_probe` تبدأ تنفيذ. بيأخد `struct pt_regs *regs` اللي فيه حالة الـ registers في لحظة الـ hook. بنقرأ منه الـ argument الأول (`pdev`) حسب الـ architecture (RDI على x86_64، X0 على ARM64)، وبنطبع اسم الـ platform device واليـ `id` بتاعه.

---

#### الـ `handler_post`

بيتشغل **بعد** ما الـ function ترجع مباشرة. بيقرأ الـ return value من الـ register المناسب (RAX أو X0) ويطبعه — لو رجعت 0 معناها الـ probe نجح، غير كده فيه error.

---

#### الـ `struct kprobe c3_kp`

| حقل | القيمة | الهدف |
|---|---|---|
| `.symbol_name` | `"meson_pinctrl_probe"` | الـ kernel بيحوّله لعنوان بالـ kallsyms |
| `.pre_handler` | `handler_pre` | callback قبل التنفيذ |
| `.post_handler` | `handler_post` | callback بعد التنفيذ |

الـ kprobe بيستبدل أول instruction من الـ function بـ breakpoint (INT3 على x86، BRK على ARM64) ولما يتشغل بيستدعي الـ handlers دول.

---

#### الـ `module_init`

بيسجّل الـ kprobe بـ `register_kprobe`. لو فشل (مثلاً الـ function مش موجودة في الـ kernel أو محمية بـ `notrace`) بيرجع error. لو نجح بيطبع العنوان الفعلي بالـ `%pS` format.

---

#### الـ `module_exit`

بيستدعي `unregister_kprobe` عشان يشيل الـ breakpoint ويرجع الـ function لحالتها الطبيعية — لازم يتعمل ده قبل الـ module يتـ unload وإلا يحصل kernel crash لو الـ breakpoint فضل موجود من غير handler.

---

### الـ Makefile والتشغيل

```bash
# Makefile
obj-m += kprobe_c3_pinctrl.o

# Build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# Load
sudo insmod kprobe_c3_pinctrl.ko

# Watch logs (probe fires on boot or when driver binds)
sudo dmesg | grep c3_pinctrl_probe

# Unload
sudo rmmod kprobe_c3_pinctrl
```

**مثال على الـ output المتوقع** لو الـ C3 hardware موجودة:

```
[c3_pinctrl_probe] kprobe planted at meson_pinctrl_probe+0x0/0x...
[c3_pinctrl_probe] probing device: name="amlogic-c3-pinctrl" id=-1 driver_override=(none)
[c3_pinctrl_probe] probe returned: 0 (SUCCESS)
```

> **ملحوظة:** لو الـ kernel مش على Amlogic C3 hardware، الـ kprobe بيتسجّل تمام لكن الـ handlers مش هتتشغل إلا لو الـ driver اتـ bind لـ device. ممكن تختبره manually بـ `platform_device_register` أو بـ device tree overlay.
