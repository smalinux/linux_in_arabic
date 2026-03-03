## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من subsystem اسمه **ARM/Amlogic Meson SoC support** — موجود في MAINTAINERS تحت القسم ده، وبيغطي كل حاجة خاصة بـ SoCs الـ Amlogic Meson، من بينها المجلد `drivers/pinctrl/meson/` اللي فيه الملف ده.

---

### المشكلة اللي بيحلها الملف ده — القصة

تخيل عندك شريحة إلكترونية (SoC) — زي Amlogic Meson A1 اللي بتلاقيها في smart speakers وأجهزة IoT صغيرة. الشريحة دي فيها أرجل (pins) فيزيائية بتخرج منها. كل رِجل ممكن تشتغل بأكتر من طريقة:

- تبقى **GPIO** عادي (input/output رقمي)
- تبقى **UART TX** لبروتوكول serial
- تبقى **I2C SDA** لبروتوكول I2C
- تبقى **PWM** لتحكم في موتور أو LED
- وكذا أكتر...

المشكلة: الـ kernel محتاج يعرف — لكل رِجل فيزيائية — إيه الوظايف الممكنة، وإزاي يضبط الـ register الصح في الـ hardware عشان يحدد الوظيفة المطلوبة.

الحل: **pinctrl driver** — ده driver بيعمل mapping بين أسماء الوظايف (زي `uart_a_tx`) والـ pins الفيزيائية والـ registers اللي بتتحكم فيهم.

---

### الـ A1 SoC — مين هو؟

الـ **Meson A1** هو SoC من Amlogic مصمم للـ IoT وأجهزة الصوت الذكية. فيه 62 pin موزعين على 5 banks:

| Bank | الأول | الأخير | عدد الـ pins | الاستخدام الرئيسي |
|------|-------|--------|--------------|-------------------|
| **P** | GPIOP_0 | GPIOP_12 | 13 | PSRAM interface |
| **B** | GPIOB_0 | GPIOB_6 | 7 | SD card / SPI Flash |
| **X** | GPIOX_0 | GPIOX_16 | 17 | UART, I2C, SPI, TDM |
| **F** | GPIOF_0 | GPIOF_12 | 13 | UART, PWM, JTAG, CEC |
| **A** | GPIOA_0 | GPIOA_11 | 12 | Audio (TDM, PDM, MCLK) |

---

### الهدف من الملف

الملف `pinctrl-meson-a1.c` بيعمل حاجة واحدة بس: **يعرّف الجداول الثابتة** (static tables) اللي بتوصف الـ A1 SoC بالكامل للـ pinctrl framework:

1. **قائمة الـ pins** كلها (`meson_a1_periphs_pins`) — 62 pin باسمهم.
2. **قائمة الـ groups** (`meson_a1_periphs_groups`) — كل group هو مجموعة pins بتشكل وظيفة واحدة، زي `uart_a_tx` على GPIOX_11.
3. **قائمة الـ functions** (`meson_a1_periphs_functions`) — زي `uart_a`, `i2c0`, `spi_a`, `pwm_a`... إلخ. كل function عندها قائمة بالـ groups الممكنة.
4. **جداول الـ banks** (`meson_a1_periphs_banks`) — بتحدد لكل bank عناوين الـ registers المسؤولة عن: pull-up/down، اتجاه الـ pin، القراءة، الكتابة، وقوة الـ drive.
5. **جداول الـ PMX banks** (`meson_a1_periphs_pmx_banks`) — بتحدد الـ register offset الخاص بـ pinmux لكل bank.

الملف مفيش فيه logic — كله data tables. الـ logic الحقيقي في `pinctrl-meson.c` و`pinctrl-meson-axg-pmx.c`.

---

### الـ Pin Multiplexing — إزاي بيشتغل؟

كل pin عنده **function number** (0 إلى 7 عادةً):

- **0** = GPIO (الوضع الافتراضي)
- **1-7** = وظايف تانية حسب الـ SoC datasheet

مثال: GPIOX_11 ممكن يبقى:
- `func 1` → `uart_a_tx`
- `func 2` → `i2c3_sck_x`

الـ driver بيكتب رقم الـ function في الـ bits المناسبة في الـ register المقابل للـ bank.

```
GPIOX_11  ──►  [PMX Register 0x3, bits 21:19]  ──►  001 = uart_a_tx
                                                      010 = i2c3_sck_x
                                                      000 = GPIO
```

---

### الوظايف الموجودة في الـ A1

الـ A1 بيدعم الوظايف دي:

| الفئة | الوظايف |
|-------|---------|
| **Serial** | uart_a, uart_b, uart_c |
| **I2C** | i2c0, i2c1, i2c2, i2c3, i2c_slave |
| **SPI** | spi_a, spif (SPI Flash) |
| **Memory** | psram |
| **Storage** | sdcard |
| **Audio** | tdm_a, tdm_b, tdm_vad, pdm, mclk_0, mclk_vad, spdif_in |
| **PWM** | pwm_a..f, pwm_a/b/c_hiz |
| **Clock** | gen_clk, clk_32k_in, clk25, clk12_24 |
| **Debug** | jtag_a, sw (SWD), tst_out |
| **AV** | cec_a, cec_b, remote_input, remote_out |
| **Misc** | mute_key, mute_en |

---

### الـ Platform Driver Registration

في نهاية الملف، الـ driver بيسجل نفسه عن طريق:

```c
/* يربط الـ compatible string بـ data struct الخاصة بالـ A1 */
static const struct of_device_id meson_a1_pinctrl_dt_match[] = {
    {
        .compatible = "amlogic,meson-a1-periphs-pinctrl",
        .data = &meson_a1_periphs_pinctrl_data,
    },
    { },
};

/* الـ probe function جاية من pinctrl-meson.c — مش محتاج يعرفها هنا */
static struct platform_driver meson_a1_pinctrl_driver = {
    .probe  = meson_pinctrl_probe,
    .driver = {
        .name   = "meson-a1-pinctrl",
        .of_match_table = meson_a1_pinctrl_dt_match,
    },
};
module_platform_driver(meson_a1_pinctrl_driver);
```

لما الـ kernel يلاقي node في الـ Device Tree بـ compatible string `"amlogic,meson-a1-periphs-pinctrl"`، بيستدعي `meson_pinctrl_probe` اللي بتاخد الجداول دي وتسجلها في الـ pinctrl framework.

---

### الملفات المهمة في الـ Subsystem

```
drivers/pinctrl/meson/
├── pinctrl-meson.c              ← الـ core logic: probe, GPIO ops, pinconf
├── pinctrl-meson.h              ← الـ structs والـ macros المشتركة
├── pinctrl-meson-axg-pmx.c      ← منطق الـ pinmux للجيل الـ AXG وما بعده (شامل A1)
├── pinctrl-meson-axg-pmx.h      ← تعريف BANK_PMX, GROUP, GPIO_GROUP macros
├── pinctrl-meson-a1.c           ← *** الملف ده *** — data tables للـ A1
├── pinctrl-meson-g12a.c         ← نفس الفكرة لـ G12A SoC
├── pinctrl-meson-gxl.c          ← نفس الفكرة لـ GXL SoC
├── pinctrl-meson-axg.c          ← نفس الفكرة لـ AXG SoC
├── pinctrl-meson-s4.c           ← نفس الفكرة لـ S4 SoC
├── pinctrl-amlogic-c3.c         ← نفس الفكرة لـ C3 SoC
├── pinctrl-amlogic-a4.c         ← نفس الفكرة لـ A4 SoC
├── pinctrl-amlogic-t7.c         ← نفس الفكرة لـ T7 SoC
├── pinctrl-meson8.c             ← نفس الفكرة لـ Meson8 SoC
├── pinctrl-meson8b.c            ← نفس الفكرة لـ Meson8b SoC
└── pinctrl-meson8-pmx.c/h       ← منطق الـ pinmux للجيل القديم (Meson8)
```

**الـ headers المهمة:**
- `include/dt-bindings/gpio/meson-a1-gpio.h` — بيعرّف أرقام الـ GPIO للـ A1 (GPIOP_0=0, ...) وبيُستخدم في الـ Device Tree

**الـ Device Tree:**
- `arch/arm64/boot/dts/amlogic/` — بيحتوي على ملفات الـ DTS اللي بتستخدم `"amlogic,meson-a1-periphs-pinctrl"`
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة اللي الـ Subsystem بيحلها

في أي SoC حديث زي الـ Amlogic Meson A1، كل pin جسماني على الـ chip مش بيعمل حاجة واحدة بس — هو ممكن يبقى GPIO عادي، أو UART TX، أو I2C SDA، أو SPI MOSI، أو PWM output، أو حتى TDM audio signal، كل ده بيتحدد من خلال **mux register** في الـ hardware.

المشكلة إن من غير framework موحد:
- كل driver كان هيحاول يكتب على نفس الـ registers بشكل مباشر → race conditions وتعارضات
- الـ board configuration كانت مبعثرة جوه الـ drivers نفسهم → صعوبة في الـ porting لـ boards مختلفة
- مفيش طريقة لـ Device Tree أو ACPI يقول "الـ UART محتاج GPIOX_11 و GPIOX_12" بشكل standard

**الـ pinctrl subsystem** اتعمل عشان يكون **الوسيط المركزي** اللي بيتحكم في:
1. Multiplexing — أي function بيستخدم أي pin
2. Configuration — pull-up/down، drive strength، direction
3. GPIO — تحويل الـ pin لـ GPIO وتشغيله

---

### الحل اللي الـ Kernel بياخده

الـ kernel بيقسم المسألة لطبقات:

| الطبقة | المسؤولية |
|--------|-----------|
| **pinctrl core** | API موحد، state machine، DT parsing |
| **pinctrl driver** (مثلاً pinctrl-meson-a1.c) | وصف الـ hardware الخاص بالـ SoC |
| **consumer driver** (مثلاً uart-driver) | طلب الـ state اللي محتاجه |

الـ consumer مش بيعرف حاجة عن الـ registers — بيقول بس "أنا عايز state اسمها `default`" والـ framework هو اللي بيترجم ده.

---

### التشبيه الواقعي — لوحة توصيل الهاتف القديمة

تخيل **لوحة توصيل الهاتف** اللي كانت بتستخدمها شركات التليفون زمان:

```
  Caller A ──┐
  Caller B ──┤   [Switchboard Operator]   ├── Line 1 (GPIOX_11)
  Caller C ──┤                            ├── Line 2 (GPIOX_12)
  Caller D ──┘                            └── Line 3 (GPIOF_4)
```

**التمثيل الكامل:**

| عنصر في التشبيه | المقابل في الـ Kernel |
|-----------------|----------------------|
| الـ Lines (الأسلاك الجسمانية) | الـ **pins** — GPIOX_11، GPIOF_4 |
| الـ Callers (UART، I2C، SPI) | الـ **functions** — uart_a، i2c0، spi_a |
| عملية التوصيل | الـ **pinmux** — كتابة الـ mux register |
| الـ Operator | الـ **pinctrl core** |
| دفتر التعليمات عند الـ Operator | الـ **pinctrl_data** struct |
| الـ Wire gauge (سُمك السلك) | الـ **drive strength** config |
| الـ Plug direction (in/out) | الـ **direction** register |

الـ Caller مش بيوصل نفسه — بيطلب من الـ Operator، والـ Operator هو اللي بيعرف أي سلك يستخدم.

---

### البنية الكبيرة للـ Framework

```
  ┌─────────────────────────────────────────────────────────┐
  │                    Consumer Drivers                      │
  │   uart_driver   i2c_driver   spi_driver   gpio_driver   │
  │       │              │            │             │        │
  │       └──────────────┴────────────┴─────────────┘        │
  │                          │                               │
  │                  pinctrl_select_state()                  │
  └──────────────────────────┼──────────────────────────────┘
                             │
  ┌──────────────────────────▼──────────────────────────────┐
  │                   pinctrl CORE                          │
  │  (drivers/pinctrl/core.c + pinmux.c + pinconf.c)        │
  │                                                         │
  │  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │
  │  │  pin space  │  │  group/func  │  │  state machine│  │
  │  │  registry   │  │  database    │  │  per device   │  │
  │  └─────────────┘  └──────────────┘  └───────────────┘  │
  └──────────────────────────┬──────────────────────────────┘
                             │ calls ops vtable
  ┌──────────────────────────▼──────────────────────────────┐
  │          SoC-Specific pinctrl Driver                    │
  │         (pinctrl-meson-a1.c + pinctrl-meson.c)          │
  │                                                         │
  │  pinctrl_ops  ──► get_groups, dt_node_to_map            │
  │  pinmux_ops   ──► set_mux  (meson_axg_pmx_ops)          │
  │  pinconf_ops  ──► pin_config_set (pull, drive)          │
  │  gpio_chip    ──► direction_input/output, get, set      │
  └──────────────────────────┬──────────────────────────────┘
                             │ regmap read/write
  ┌──────────────────────────▼──────────────────────────────┐
  │                  Hardware Registers                      │
  │  ┌──────────┐ ┌─────────┐ ┌──────────┐ ┌────────────┐  │
  │  │ MUX regs │ │DIR regs │ │PULL regs │ │  DS regs   │  │
  │  │ (0x0-0x9)│ │(0x21,..)│ │(0x23,..) │ │ (0x25,..)  │  │
  │  └──────────┘ └─────────┘ └──────────┘ └────────────┘  │
  └─────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction: Pin → Group → Function → State

ده أهم مفهوم في الـ subsystem. الـ hierarchy بيتكون من 4 مستويات:

```
  STATE "default"
       │
       ▼
  FUNCTION "uart_a"
       │
       ├──► GROUP "uart_a_tx"  ──► PIN GPIOX_11
       ├──► GROUP "uart_a_rx"  ──► PIN GPIOX_12
       ├──► GROUP "uart_a_cts" ──► PIN GPIOX_13
       └──► GROUP "uart_a_rts" ──► PIN GPIOX_14
```

**الـ Pin** هو الوحدة الأصغر — رقم فريد + اسم.

**الـ Group** هو مجموعة من الـ pins بتشتغل سوا لتوصيل function معين. في الـ Meson A1، معظم الـ groups عبارة عن pin واحد بس (single-pin group). ده لأن كل pin بيتحكم فيه بـ 4 bits في الـ mux register بشكل مستقل.

**الـ Function** هو الـ peripheral اللي بيستخدم مجموعة من الـ groups. مثلاً `uart_a` بيستخدم `uart_a_tx` + `uart_a_rx` + اختياري `uart_a_cts` + `uart_a_rts`.

**الـ State** اسم symbolic بيجي من الـ Device Tree (مثلاً `default`، `sleep`) بيُترجم لمجموعة من الـ mappings.

---

### الـ Structs الأساسية وعلاقتها ببعض

```
  pinctrl_desc
  ┌──────────────────────────────────┐
  │ name: "periphs-banks"            │
  │ pins: → pinctrl_pin_desc[]       │◄── MESON_PIN(GPIOX_11) = {.number=..., .name="GPIOX_11"}
  │ npins: 63                        │
  │ pctlops: → pinctrl_ops           │◄── get_groups_count, get_group_name, dt_node_to_map
  │ pmxops:  → meson_axg_pmx_ops     │◄── set_mux (بيكتب في الـ MUX registers)
  │ confops: → pinconf_ops           │◄── pin_config_set (pull, drive strength)
  └──────────────────────────────────┘
           ▲
           │ يُبنى من
  meson_pinctrl_data
  ┌──────────────────────────────────┐
  │ pins:   → meson_a1_periphs_pins  │
  │ groups: → meson_a1_periphs_groups│
  │ funcs:  → meson_a1_periphs_funcs │
  │ banks:  → meson_a1_periphs_banks │
  │ pmx_ops: → meson_axg_pmx_ops     │
  │ pmx_data: → axg_pmx_banks_data   │
  └──────────────────────────────────┘
```

---

### الـ meson_pmx_group بالتفصيل

```c
struct meson_pmx_group {
    const char *name;          // "uart_a_tx"
    const unsigned int *pins;  // → { GPIOX_11 }
    unsigned int num_pins;     // 1
    const void *data;          // → meson_pmx_axg_data { .func = 1 }
};
```

الـ `data` field بيحتوي على **رقم الـ function** اللي لازم يتكتب في الـ mux register. مثلاً:

```c
GROUP(uart_a_tx, 1),   // .func = 1 → كتابة 0x1 في الـ 4 bits الخاصة بـ GPIOX_11
GROUP(i2c3_sck_x, 2), // .func = 2 → كتابة 0x2 في الـ 4 bits الخاصة بـ GPIOX_11
GROUP(gen_clk_x, 7),  // .func = 7 → كتابة 0x7 في الـ 4 bits الخاصة بـ GPIOX_7
```

الـ Meson A1 بيستخدم 4 bits لكل pin في الـ mux register، يعني كل pin ممكن يكون له حتى 15 function مختلفة (func 0 = GPIO، 1-7 = alt functions).

---

### الـ meson_bank والـ Register Layout

```c
struct meson_bank {
    const char *name;         // "X"
    unsigned int first;       // GPIOX_0
    unsigned int last;        // GPIOX_16
    int irq_first;            // 20  (hwirq base)
    int irq_last;             // 36
    struct meson_reg_desc regs[MESON_NUM_REG];
    // [MESON_REG_PULLEN] = { 0x23, 0 }  ← pullen register offset, bit offset
    // [MESON_REG_PULL]   = { 0x24, 0 }  ← pull register
    // [MESON_REG_DIR]    = { 0x22, 0 }  ← direction register
    // [MESON_REG_OUT]    = { 0x21, 0 }  ← output register
    // [MESON_REG_IN]     = { 0x20, 0 }  ← input register
    // [MESON_REG_DS]     = { 0x25, 0 }  ← drive strength register
};
```

كل bank بيتمثل **domain فيزيائي** على الـ SoC — الـ P bank للـ PSRAM، الـ B bank للـ boot/storage، الـ X bank للـ general purpose، الـ F bank للـ audio/clocks، والـ A bank للـ audio.

الـ `meson_reg_desc` بيحتوي على `reg` (offset في الـ regmap) و `bit` (أول bit لأول pin في الـ bank). لما الـ core عايز يتحكم في `GPIOX_5` مثلاً:

```
pin_offset = GPIOX_5 - GPIOX_0 = 5
actual_bit = bank.regs[MESON_REG_DIR].bit + pin_offset = 0 + 5 = 5
actual_reg = bank.regs[MESON_REG_DIR].reg = 0x22
→ regmap_update_bits(reg_gpio, 0x22, BIT(5), value)
```

---

### الـ meson_pmx_bank والـ MUX Register Layout

```c
struct meson_pmx_bank {
    const char *name;    // "X"
    unsigned int first;  // GPIOX_0
    unsigned int last;   // GPIOX_16
    unsigned int reg;    // 0x3  ← base register للـ mux
    unsigned int offset; // 0    ← bit offset للـ pin الأول
};
```

الـ X bank مثلاً بيبدأ من register `0x3`. كل pin بياخد 4 bits، يعني GPIOX_0 في bits [3:0] من reg 0x3، وGPIOX_1 في bits [7:4]، وهكذا. لما وصلنا GPIOX_8 (اللي بياخد bits [31:28] من reg 0x3) بينتقل للـ reg التالي تلقائياً.

```
Reg 0x3:  [GPIOX_7][GPIOX_6][GPIOX_5][GPIOX_4][GPIOX_3][GPIOX_2][GPIOX_1][GPIOX_0]
           31:28    27:24    23:20    19:16    15:12    11:8     7:4      3:0

Reg 0x4:  [GPIOX_15][GPIOX_14][GPIOX_13][GPIOX_12][GPIOX_11][GPIOX_10][GPIOX_9][GPIOX_8]
Reg 0x5:  [...........GPIOX_16 in bits 3:0............]
```

---

### علاقة الـ Structs الكاملة — big picture

```
  meson_a1_pinctrl_driver
         │
         │ .probe = meson_pinctrl_probe()
         ▼
  meson_pinctrl (runtime state per device)
  ┌────────────────────────────────────────────┐
  │ dev     → platform_device                  │
  │ pcdev   → pinctrl_dev (kernel core object) │
  │ desc    → pinctrl_desc ───────────────────►│── pins[], pctlops, pmxops, confops
  │ data    → meson_pinctrl_data ─────────────►│── groups[], funcs[], banks[], pmx_ops
  │ reg_mux → regmap (MUX registers)           │
  │ reg_gpio→ regmap (DIR/IN/OUT/PULL regs)    │
  │ reg_ds  → regmap (drive strength regs)     │
  │ chip    → gpio_chip ──────────────────────►│── direction_input/output, get, set
  └────────────────────────────────────────────┘
```

الـ `meson_pinctrl` struct ده الـ **runtime state** الكامل للـ driver. الـ `pinctrl_desc` هو الوصف الـ static للـ hardware، والـ `meson_pinctrl_data` هو الـ SoC-specific data اللي بتتغير من chip لـ chip.

---

### ما بيمتلكه الـ Framework vs ما بيفوضه للـ Driver

| الـ pinctrl Core بيمتلك | الـ Driver بيفوض إليه |
|--------------------------|----------------------|
| State machine لكل consumer device | وصف الـ pins الفيزيائية (الأسماء والأرقام) |
| DT parsing للـ `pinctrl-X` properties | وصف الـ groups وأي pins تنتمي لكل group |
| Conflict detection (pin مش ممكن يتاخد من اتنين سوا) | وصف الـ functions وأي groups تنتمي لكل function |
| GPIO-to-pin mapping | الكتابة الفعلية في الـ hardware registers |
| Locking وـ reference counting | حساب الـ register offset لكل pin |
| debugfs interface | drive strength values الخاصة بالـ SoC |
| IRQ domain integration | DT extra parsing (مثلاً `meson_a1_parse_dt_extra`) |

---

### الـ DT Integration — كيف بيتربط كل ده ببعض وقت الـ Boot

الـ Device Tree بيحتوي على:

```dts
/* في الـ SoC dtsi */
periphs_pinctrl: pinctrl@400 {
    compatible = "amlogic,meson-a1-periphs-pinctrl";
    reg = <0x0 0x400 0x0 0x200>;
    /* ... */
};

/* في الـ board dts أو driver node */
&uart_AO {
    pinctrl-names = "default";
    pinctrl-0 = <&uart_a_pins>;
    status = "okay";
};

uart_a_pins: uart-a {
    mux {
        groups = "uart_a_tx", "uart_a_rx";
        function = "uart_a";
    };
};
```

لما الـ UART driver بيعمل `pinctrl_select_state(pctldev, "default")`، الـ core بيـparse الـ DT، بيلاقي إن `uart_a_tx` هو group جوه function `uart_a`، بيطلب من الـ `meson_axg_pmx_ops.set_mux()` إنه يكتب func=1 في الـ 4 bits الخاصة بـ GPIOX_11 في register 0x4 (X bank mux).

---

### نقطة مهمة: الـ Regmap Subsystem

الـ pinctrl-meson مش بيكتب في الـ registers مباشرة — بيستخدم الـ **regmap subsystem**، ده subsystem تاني بيوفر:
- Caching للـ register values
- Locking تلقائي
- Endianness handling
- Debugfs للـ register dump

الـ `meson_pinctrl` struct بيحتوي على خمس `regmap *`:
- `reg_mux` → للـ function mux registers
- `reg_pullen` → لـ pull-enable registers
- `reg_pull` → لـ pull up/down direction
- `reg_gpio` → للـ direction/in/out registers
- `reg_ds` → للـ drive strength registers

الفصل ده بيخلي كل نوع من الـ registers ممكن يتعامل معه بشكل مستقل، وعلى بعض الـ SoCs ممكن يكون عندهم regmaps مختلفة لأنواع مختلفة (مثلاً always-on domain بيبقى في address space منفصل).
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Flags والـ Config Options

#### `enum meson_reg_type` — أنواع الـ registers للـ GPIO

| القيمة | المعنى |
|--------|--------|
| `MESON_REG_PULLEN` | تفعيل الـ pull-up/pull-down |
| `MESON_REG_PULL` | اتجاه الـ pull (up أو down) |
| `MESON_REG_DIR` | اتجاه الـ pin (input/output) |
| `MESON_REG_OUT` | قيمة الخروج (output value) |
| `MESON_REG_IN` | قراءة قيمة الدخول (input value) |
| `MESON_REG_DS` | قوة الـ drive strength |
| `MESON_NUM_REG` | عدد الأنواع (6) — يُستخدم لتحديد حجم مصفوفة الـ regs |

#### `enum meson_pinconf_drv` — قيم الـ Drive Strength

| القيمة | التيار |
|--------|--------|
| `MESON_PINCONF_DRV_500UA` | 500 microampere |
| `MESON_PINCONF_DRV_2500UA` | 2500 microampere |
| `MESON_PINCONF_DRV_3000UA` | 3000 microampere |
| `MESON_PINCONF_DRV_4000UA` | 4000 microampere |

#### الـ GPIO Banks في Meson A1 — Cheatsheet

| Bank | أول pin | آخر pin | عدد الـ pins | IRQ base | reg_pullen | reg_pull | reg_dir | reg_out | reg_in | reg_ds |
|------|---------|---------|--------------|----------|-----------|---------|---------|---------|--------|--------|
| P | GPIOP_0 | GPIOP_12 | 13 | 0..12 | 0x3 | 0x4 | 0x2 | 0x1 | 0x0 | 0x5 |
| B | GPIOB_0 | GPIOB_6 | 7 | 13..19 | 0x13 | 0x14 | 0x12 | 0x11 | 0x10 | 0x15 |
| X | GPIOX_0 | GPIOX_16 | 17 | 20..36 | 0x23 | 0x24 | 0x22 | 0x21 | 0x20 | 0x25 |
| F | GPIOF_0 | GPIOF_12 | 13 | 37..49 | 0x33 | 0x34 | 0x32 | 0x31 | 0x30 | 0x35 |
| A | GPIOA_0 | GPIOA_11 | 12 | 50..61 | 0x43 | 0x44 | 0x42 | 0x41 | 0x40 | 0x45 |

#### الـ PMX Banks — Cheatsheet للـ Mux Registers

| Bank | أول pin | آخر pin | reg | offset |
|------|---------|---------|-----|--------|
| P | GPIOP_0 | GPIOP_12 | 0x0 | 0 |
| B | GPIOB_0 | GPIOB_6 | 0x2 | 0 |
| X | GPIOX_0 | GPIOX_16 | 0x3 | 0 |
| F | GPIOF_0 | GPIOF_12 | 0x6 | 0 |
| A | GPIOA_0 | GPIOA_11 | 0x8 | 0 |

#### الـ Functions والـ Function Numbers (Mux)

| Function | الـ Func# على الـ register | الـ Banks المستخدمة |
|----------|--------------------------|---------------------|
| GPIO (default) | 0 | كل البنوك |
| psram | 1 | P |
| spif | 1 | B |
| sdcard | 1 أو 2 | B, X |
| uart_a | 1 | X |
| uart_b | 1, 5 | F, X |
| uart_c | 3 | X |
| i2c0 | 1, 4 | F |
| i2c1 | 2, 5 | A, X |
| i2c2 | 2, 3, 5 | X, A |
| i2c3 | 2, 4 | X, F |
| spi_a | 2, 4 | X, A |
| pwm_a..f | 1..7 | متعددة |
| tdm_a | 1, 2 | X |
| tdm_b | 1, 2 | A |
| pdm | 3 | X, A |
| jtag_a | 1 | F |
| tst_out | 6 | A |

---

### 1. أهم الـ Structs

#### `struct pinctrl_pin_desc`

**الغرض:** وصف pin واحد — الاسم والـ ID الرقمي.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `number` | `unsigned int` | رقم الـ pin (GPIO number) |
| `name` | `const char *` | الاسم النصي مثل `"GPIOX_0"` |
| `drv_data` | `void *` | بيانات إضافية خاصة بالـ driver |

**الاستخدام في الملف:** المصفوفة `meson_a1_periphs_pins[]` تحتوي على 62 pin مُعرَّف بالـ macro `MESON_PIN(x)` الذي يُوسَّع إلى `PINCTRL_PIN(x, #x)`.

---

#### `struct meson_pmx_group`

**الغرض:** تمثيل **مجموعة pins** يمكن تفعيلها كوحدة واحدة لوظيفة معينة.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `name` | `const char *` | اسم المجموعة مثل `"uart_a_tx"` |
| `pins` | `const unsigned int *` | مصفوفة أرقام الـ pins في المجموعة |
| `num_pins` | `unsigned int` | عدد الـ pins |
| `data` | `const void *` | يُشير إلى `meson_pmx_axg_data` الذي يحمل رقم الـ function |

**الربط:** كل entry في `meson_a1_periphs_groups[]` هو `meson_pmx_group` — المجموعات GPIO لها func=0، والمجموعات الوظيفية لها func=1..7.

---

#### `struct meson_pmx_axg_data`

**الغرض:** البيانات الخاصة بنظام AXG/A1 الـ pinmux — تحمل رقم الـ function المطلوب تفعيله في الـ register.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `func` | `unsigned int` | رقم الـ function (0=GPIO, 1..7=وظيفة خاصة) |

**الربط:** مُخزَّن كـ `void *data` داخل `meson_pmx_group`.

---

#### `struct meson_pmx_func`

**الغرض:** تمثيل **وظيفة كاملة** — كـ `uart_a` أو `spi_a` — وتجميع كل مجموعاتها.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `name` | `const char *` | اسم الوظيفة مثل `"uart_a"` |
| `groups` | `const char * const *` | مصفوفة أسماء المجموعات التابعة لها |
| `num_groups` | `unsigned int` | عدد المجموعات |

**الاستخدام:** `meson_a1_periphs_functions[]` يحتوي على 40 function، كل واحد مُعرَّف بالـ macro `FUNCTION(fn)`.

---

#### `struct meson_reg_desc`

**الغرض:** وصف موقع bit واحد في register — يُستخدم لتحديد كيفية التحكم في خاصية معينة (pull/dir/out/in/ds) لكل bank.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `reg` | `unsigned int` | offset الـ register في الـ regmap |
| `bit` | `unsigned int` | رقم الـ bit داخل الـ register |

---

#### `struct meson_bank`

**الغرض:** تمثيل **bank** كامل من الـ GPIO — مجموعة pins متجاورة لها نفس الـ register base.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `name` | `const char *` | اسم البنك مثل `"P"` أو `"X"` |
| `first` | `unsigned int` | أول pin في البنك |
| `last` | `unsigned int` | آخر pin في البنك |
| `irq_first` | `int` | أول hwirq في البنك |
| `irq_last` | `int` | آخر hwirq في البنك |
| `regs[MESON_NUM_REG]` | `meson_reg_desc[]` | مصفوفة من 6 descriptors: pullen, pull, dir, out, in, ds |

**الربط:** `meson_a1_periphs_banks[]` يحتوي 5 banks (P, B, X, F, A). كل bank يُشير إلى registers مختلفة عبر الـ `regmap`.

---

#### `struct meson_pmx_bank`

**الغرض:** وصف **bank الـ pinmux** — يحدد أي register يتحكم في mux لكل pin في البنك.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `name` | `const char *` | اسم البنك |
| `first` | `unsigned int` | أول pin |
| `last` | `unsigned int` | آخر pin |
| `reg` | `unsigned int` | offset register الـ mux في الـ regmap |
| `offset` | `unsigned int` | bit offset للـ pin الأول |

**الربط:** `meson_a1_periphs_pmx_banks[]` هو مصفوفة من 5 banks — كل 4 bits تمثل function لـ pin واحد.

---

#### `struct meson_axg_pmx_data`

**الغرض:** تغليف مصفوفة الـ pmx_banks مع عددها — يُمرَّر كـ `pmx_data` للـ pinctrl data.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `pmx_banks` | `const struct meson_pmx_bank *` | مصفوفة الـ pmx banks |
| `num_pmx_banks` | `unsigned int` | عدد الـ banks |

---

#### `struct meson_pinctrl_data`

**الغرض:** **الـ static configuration** الكاملة للـ SoC — كل ما يحتاجه الـ driver لتشغيل الـ pinctrl.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `name` | `const char *` | اسم وصفي مثل `"periphs-banks"` |
| `pins` | `const struct pinctrl_pin_desc *` | مصفوفة وصف الـ pins |
| `groups` | `const struct meson_pmx_group *` | مصفوفة المجموعات |
| `funcs` | `const struct meson_pmx_func *` | مصفوفة الـ functions |
| `banks` | `const struct meson_bank *` | مصفوفة الـ GPIO banks |
| `num_pins/groups/funcs/banks` | `unsigned int` | أعداد العناصر |
| `pmx_ops` | `const struct pinmux_ops *` | يُشير إلى `meson_axg_pmx_ops` |
| `pmx_data` | `const void *` | يُشير إلى `meson_a1_periphs_pmx_banks_data` |
| `parse_dt` | `int (*)(struct meson_pinctrl *)` | callback لتحليل الـ device tree — `meson_a1_parse_dt_extra` |

---

#### `struct meson_pinctrl`

**الغرض:** **الـ runtime instance** — يحمل حالة الـ driver بعد الـ probe.

| الحقل | النوع | المعنى |
|-------|-------|--------|
| `dev` | `struct device *` | الـ device المرتبط |
| `pcdev` | `struct pinctrl_dev *` | handle للـ pinctrl core |
| `desc` | `struct pinctrl_desc` | وصف الـ pinctrl للـ kernel framework |
| `data` | `struct meson_pinctrl_data *` | الـ static config الخاص بالـ SoC |
| `reg_mux` | `struct regmap *` | regmap للـ mux registers |
| `reg_pullen` | `struct regmap *` | regmap لتفعيل الـ pull |
| `reg_pull` | `struct regmap *` | regmap لاتجاه الـ pull |
| `reg_gpio` | `struct regmap *` | regmap لـ GPIO control (dir/in/out) |
| `reg_ds` | `struct regmap *` | regmap لـ drive strength |
| `chip` | `struct gpio_chip` | واجهة الـ GPIO subsystem |
| `fwnode` | `struct fwnode_handle *` | نقطة الربط بالـ device tree |

---

### 2. مخطط علاقات الـ Structs

```
meson_a1_periphs_pinctrl_data  (static const, one instance per SoC variant)
│
├─── .pins ──────────────────► meson_a1_periphs_pins[]
│                                (pinctrl_pin_desc × 62)
│                                  [GPIOP_0..GPIOA_11]
│
├─── .groups ────────────────► meson_a1_periphs_groups[]
│                                (meson_pmx_group × ~200)
│                                  ┌─ .name = "uart_a_tx"
│                                  ├─ .pins → uart_a_tx_pins[]
│                                  └─ .data → meson_pmx_axg_data{.func=1}
│
├─── .funcs ─────────────────► meson_a1_periphs_functions[]
│                                (meson_pmx_func × 40)
│                                  ┌─ .name = "uart_a"
│                                  └─ .groups → uart_a_groups[]
│                                               ["uart_a_tx","uart_a_rx",...]
│
├─── .banks ─────────────────► meson_a1_periphs_banks[]
│                                (meson_bank × 5)
│                                  ┌─ .name = "X"
│                                  ├─ .first = GPIOX_0
│                                  ├─ .last  = GPIOX_16
│                                  ├─ .irq_first = 20
│                                  └─ .regs[6] → meson_reg_desc
│                                       [PULLEN={0x23,0}, PULL={0x24,0},
│                                        DIR={0x22,0}, OUT={0x21,0},
│                                        IN={0x20,0},  DS={0x25,0}]
│
├─── .pmx_ops ───────────────► meson_axg_pmx_ops  (extern, في axg-pmx.c)
│
└─── .pmx_data ──────────────► meson_a1_periphs_pmx_banks_data
                                 (meson_axg_pmx_data)
                                   └─ .pmx_banks → meson_a1_periphs_pmx_banks[]
                                                    (meson_pmx_bank × 5)
                                                      ┌─ .name = "X"
                                                      ├─ .first = GPIOX_0
                                                      ├─ .last  = GPIOX_16
                                                      ├─ .reg   = 0x3
                                                      └─ .offset = 0


meson_pinctrl  (runtime, allocated at probe)
│
├─── .data ──────────────────► meson_a1_periphs_pinctrl_data (أعلاه)
├─── .pcdev ─────────────────► pinctrl_dev  (kernel pinctrl core)
├─── .chip ──────────────────► gpio_chip   (kernel GPIO subsystem)
├─── .reg_mux ───────────────► regmap      (mux registers)
├─── .reg_pullen ────────────► regmap      (pull-enable registers)
├─── .reg_pull ──────────────► regmap      (pull direction registers)
├─── .reg_gpio ──────────────► regmap      (GPIO dir/in/out registers)
└─── .reg_ds ────────────────► regmap      (drive strength registers)
```

---

### 3. مخطط دورة حياة الـ Driver

```
Device Tree (DT)
  compatible = "amlogic,meson-a1-periphs-pinctrl"
         │
         ▼
meson_a1_pinctrl_dt_match[]
  .data = &meson_a1_periphs_pinctrl_data
         │
         ▼ (kernel matches DT node)
platform_driver.probe()
  = meson_pinctrl_probe()         ← مُعرَّف في pinctrl-meson.c
         │
         ├── allocate meson_pinctrl (kzalloc)
         ├── مطابقة of_device_id → جلب pinctrl_data
         ├── parse_dt() → meson_a1_parse_dt_extra()
         │     └── قراءة regmaps من DT (reg_mux, reg_gpio, etc.)
         ├── بناء pinctrl_desc من pins/groups/funcs
         ├── pinctrl_register() → تسجيل مع الـ pinctrl core
         │     └── يُعيد pcdev
         ├── gpiochip_add_data() → تسجيل مع الـ GPIO subsystem
         └── Driver جاهز
         │
         ▼ (استخدام من DT consumer)
pinctrl_select_state()
  └── يستدعي pmx_ops->set_mux()
        └── meson_axg_pmx_set()
              ├── البحث عن group في groups[]
              ├── قراءة func من meson_pmx_axg_data
              ├── البحث عن pmx_bank حسب pin
              └── كتابة الـ func في regmap[reg + pin_offset]
         │
         ▼ (عند إزالة الـ driver)
platform_driver.remove()
  ├── gpiochip_remove()
  └── pinctrl_unregister()
```

---

### 4. مخطط تدفق الـ Pinmux (Call Flow)

#### سيناريو: DT node يطلب تفعيل uart_a على GPIOX_11/12

```
DT consumer node:
  pinctrl-0 = <&uart_a_pins>;
       │
       ▼
pinctrl_select_state(dev, "default")
  │
  ▼
pinctrl_commit_state()
  │
  ▼
pinmux_apply_muxsetting()
  │
  ▼
pmx_ops->set_mux(pcdev, func_selector, group_selector)
  = meson_axg_pmx_set_mux()          [pinctrl-meson-axg-pmx.c]
  │
  ├── جلب group: meson_a1_periphs_groups["uart_a_tx"]
  │     .pins = {GPIOX_11}
  │     .data → meson_pmx_axg_data{.func = 1}
  │
  ├── البحث في meson_a1_periphs_pmx_banks[]
  │     عن bank يحتوي GPIOX_11:
  │     bank "X": first=GPIOX_0, last=GPIOX_16, reg=0x3, offset=0
  │
  ├── حساب الموقع:
  │     pin_index = GPIOX_11 - GPIOX_0 = 11
  │     bit_offset = offset + pin_index * 4 = 0 + 44 = 44
  │     reg = 0x3 + (44 / 32) = 0x4  [word offset]
  │     bit = 44 % 32 = 12
  │
  └── regmap_update_bits(pc->reg_mux, reg, mask, func<<bit)
        → كتابة 0x1 في bits [15:12] للـ register 0x4
          → الـ hardware يُوصِّل GPIOX_11 بالـ UART_A_TX
```

#### سيناريو: قراءة GPIO input

```
gpio_get_value(GPIOX_5)
  │
  ▼
gpio_chip.get(chip, offset)
  = meson_gpio_get()                  [pinctrl-meson.c]
  │
  ├── meson_calc_reg_and_bit(pc, gpio, MESON_REG_IN, &reg, &bit)
  │     ├── البحث في banks[] عن bank يحتوي GPIOX_5
  │     │     bank "X": regs[MESON_REG_IN] = {0x20, 0}
  │     └── reg=0x20, bit=0+(GPIOX_5-GPIOX_0)=5
  │
  └── regmap_read(pc->reg_gpio, 0x20, &val)
        return (val >> 5) & 1
```

#### سيناريو: ضبط drive strength

```
pin_config_set(pin, PIN_CONFIG_DRIVE_STRENGTH_UA, 2500)
  │
  ▼
pinconf_ops->pin_config_set()
  = meson_pinconf_set()               [pinctrl-meson.c]
  │
  ├── تحديد القيمة: 2500uA → MESON_PINCONF_DRV_2500UA = 1
  │
  ├── meson_calc_reg_and_bit(pc, pin, MESON_REG_DS, &reg, &bit)
  │     bank "X": regs[MESON_REG_DS] = {0x25, 0}
  │     bit = (GPIOX_5 - GPIOX_0) * 2 = 10  (2 bits per pin for DS)
  │
  └── regmap_update_bits(pc->reg_ds, 0x25, 0x3<<10, 1<<10)
```

---

### 5. استراتيجية الـ Locking

#### نظرة عامة

الملف `pinctrl-meson-a1.c` نفسه **لا يحتوي على أي lock** — كل البيانات فيه هي `static const`، أي read-only بعد التحميل. الـ locking يحدث في الطبقات الأدنى.

#### جدول الـ Locks

| الـ Lock | الموقع | ما يحميه |
|---------|--------|---------|
| `regmap` internal lock | داخل kernel regmap framework | قراءة/كتابة الـ registers — يمنع race conditions على bus access |
| `gpio_chip.bgpio_lock` | `gpio_chip` (spinlock) | عمليات GPIO المتزامنة (get/set/direction) |
| `pinctrl_dev` mutex | `pinctrl_dev` في الـ pinctrl core | تغيير الـ mux state — يمنع تغيير configuration من threads متعددة |
| `pinctrl_maps` mutex | pinctrl core | قراءة/تعديل خريطة الـ pin assignments |

#### ترتيب الـ Locks (Lock Ordering)

```
pinctrl_dev.mutex          (الأعلى — يُؤخذ أولاً)
      │
      ▼
gpio_chip.bgpio_lock       (spinlock — داخل GPIO ops)
      │
      ▼
regmap internal lock       (الأدنى — يُؤخذ أخيراً، قصير المدة)
```

**قاعدة:** لا يُؤخذ `bgpio_lock` وأنت ممسك بـ `pinctrl mutex` في نفس الوقت من السياق نفسه، لتجنب deadlock. الـ `regmap` lock هو spinlock أو mutex حسب نوع الـ bus (MMIO = spinlock).

#### الـ Static Data — لا تحتاج lock

كل هذه المصفوفات في الملف معرَّفة بـ `static const` — تُقرأ فقط بعد boot:
- `meson_a1_periphs_pins[]`
- `meson_a1_periphs_groups[]`
- `meson_a1_periphs_functions[]`
- `meson_a1_periphs_banks[]`
- `meson_a1_periphs_pmx_banks[]`

**لا يوجد write access** لهذه البيانات بعد تحميل الـ module، لذلك لا تحتاج أي حماية.
## Phase 4: شرح الـ Functions

---

### ملخص — Cheatsheet للـ APIs والـ Macros الأساسية

| الاسم | النوع | الغرض |
|---|---|---|
| `MESON_PIN(x)` | Macro | يعرّف `pinctrl_pin_desc` لكل pin باسمه |
| `GPIO_GROUP(gpio)` | Macro | يعرّف `meson_pmx_group` لـ GPIO في وضع plain GPIO |
| `GROUP(grp, f)` | Macro | يعرّف `meson_pmx_group` لـ alternate function بـ function number |
| `FUNCTION(fn)` | Macro | يعرّف `meson_pmx_func` بربط الاسم بـ array الـ groups |
| `BANK_DS(...)` | Macro | يعرّف `meson_bank` كامل بكل register offsets + drive strength |
| `BANK_PMX(n,f,l,r,o)` | Macro | يعرّف `meson_pmx_bank` لـ pinmux register mapping |
| `PMX_DATA(f)` | Macro | يعرّف `meson_pmx_axg_data` بـ function number |
| `meson_a1_parse_dt_extra()` | Function | يوحّد الـ regmaps للـ A1 SoC (pull/pullen/ds → reg_gpio) |
| `meson_pinctrl_probe()` | Function | الـ platform probe الرئيسي، يسجّل الـ pinctrl + gpiolib |
| `module_platform_driver()` | Macro | يسجّل الـ driver مع kernel عبر `platform_driver_register` |
| `MODULE_DEVICE_TABLE(of, ...)` | Macro | يُصدّر الـ `of_device_id` للـ module autoloading |
| `meson_axg_pmx_ops` | extern | الـ `pinmux_ops` implementation الموروثة من AXG |

---

### Group 1: Static Data Tables — تعريف الـ Pin Descriptors والـ Groups

هذه المجموعة مش functions بالمعنى الكلاسيكي، لكنها الجزء الأهم في الملف. كل اللي بيحصل في الـ driver هو إنك تملأ structures ثابتة (const static)، والـ core pinctrl framework بيقرأها في runtime. الـ driver نفسه لا يحتوي على أي function مكتوبة يدوياً غير اللي بتيجي من الـ core.

---

#### `MESON_PIN(x)`

```c
#define MESON_PIN(x) PINCTRL_PIN(x, #x)
// expands to:
// { .number = (x), .name = #x }
```

**ما بيعمله:** بيولّد `struct pinctrl_pin_desc` لكل pin في الـ SoC. الـ A1 عنده 62 pin موزعين على 5 banks: GPIOP (13 pins)، GPIOB (7 pins)، GPIOX (17 pins)، GPIOF (13 pins)، GPIOA (12 pins).

**Parameters:**
- `x` — رقم الـ pin كما هو معرّف في `meson-a1-gpio.h` (enum values زي `GPIOP_0`)

**القيمة المُرجَعة:** struct literal من نوع `pinctrl_pin_desc`.

**Key details:** الأرقام لازم تكون contiguous داخل الـ array لأن الـ core بيعمل index lookup مباشر. الـ name string بيُستخدم في `/sys/kernel/debug/pinctrl/`.

---

#### `GPIO_GROUP(gpio)`

```c
#define GPIO_GROUP(gpio)                        \
    {                                           \
        .name = #gpio,                          \
        .pins = (const unsigned int[]){ gpio }, \
        .num_pins = 1,                          \
        .data = (const struct meson_pmx_axg_data[]){ PMX_DATA(0) }, \
    }
```

**ما بيعمله:** بيعرّف `meson_pmx_group` لـ GPIO في وضع GPIO العادي (function = 0). كل pin في الـ A1 ليه group باسمه بالظبط زي اسم الـ pin، وده اللي بيخلي الـ DT consumer يقدر يـ request pin بالاسم.

**Parameters:**
- `gpio` — اسم الـ GPIO constant (مثلاً `GPIOP_0`)

**Key details:** الـ `data` بيحتوي على `func = 0` وده معناه GPIO function (no mux). الـ 62 GPIO groups الأولى في `meson_a1_periphs_groups[]` كلها من هذا النوع.

---

#### `GROUP(grp, f)`

```c
#define GROUP(grp, f)                                       \
    {                                                       \
        .name = #grp,                                       \
        .pins = grp ## _pins,                               \
        .num_pins = ARRAY_SIZE(grp ## _pins),               \
        .data = (const struct meson_pmx_axg_data[]){        \
            PMX_DATA(f),                                    \
        },                                                  \
    }
```

**ما بيعمله:** بيربط اسم الـ function group (زي `uart_a_tx`) بالـ pins array المقابلة (زي `uart_a_tx_pins[]`)، وبيحدد رقم الـ alternate function `f` (من 1 لـ 7) اللي لازم يتكتب في الـ mux register.

**Parameters:**
- `grp` — اسم الـ group بدون `_pins` suffix
- `f` — رقم الـ function في الـ hardware mux register (1–7)

**Key details:** الـ `f` بيتحدد حسب hardware datasheet. مثلاً `uart_a_tx` على bank X بـ func=1، بينما `spi_a_mosi_x7` على نفس البنك بـ func=4.

---

#### `FUNCTION(fn)`

```c
#define FUNCTION(fn)                            \
    {                                           \
        .name = #fn,                            \
        .groups = fn ## _groups,                \
        .num_groups = ARRAY_SIZE(fn ## _groups),\
    }
```

**ما بيعمله:** بيعرّف `meson_pmx_func` بربط اسم الـ function (زي `"uart_a"`) بـ array الأسماء `uart_a_groups[]` التي تحتوي على كل الـ groups التي يمكن استخدامها لتنشيط هذه الـ function.

**Parameters:**
- `fn` — اسم الـ function

**Key details:** الـ A1 عنده 40 function مسجّلة في `meson_a1_periphs_functions[]`. الـ `gpio_periphs` هي الـ function الأولى (index 0) وده الـ default.

---

### Group 2: Bank Registration Macros — تعريف الـ GPIO Banks

---

#### `BANK_DS(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib, dsr, dsb)`

```c
#define BANK_DS(n, f, l, fi, li, per, peb, pr, pb, dr, db, or, ob, ir, ib, dsr, dsb) \
    {                                                \
        .name      = n,                              \
        .first     = f,                              \
        .last      = l,                              \
        .irq_first = fi,                             \
        .irq_last  = li,                             \
        .regs = {                                    \
            [MESON_REG_PULLEN] = { per, peb },       \
            [MESON_REG_PULL]   = { pr,  pb  },       \
            [MESON_REG_DIR]    = { dr,  db  },       \
            [MESON_REG_OUT]    = { or,  ob  },       \
            [MESON_REG_IN]     = { ir,  ib  },       \
            [MESON_REG_DS]     = { dsr, dsb },       \
        },                                           \
    }
```

**ما بيعمله:** بيعرّف `meson_bank` كامل مع كل الـ register offsets اللازمة للتحكم في الـ GPIO (pull-enable, pull direction, data direction, output, input, drive strength).

**Parameters:**
- `n` — اسم الـ bank (مثلاً `"P"`)
- `f`, `l` — أول وآخر pin number في الـ bank
- `fi`, `li` — hwirq range (لربط الـ GPIO بالـ interrupt controller)
- `per, peb` — register offset + bit للـ pull-enable
- `pr, pb` — register offset + bit للـ pull direction
- `dr, db` — register offset + bit للـ direction (input/output)
- `or, ob` — register offset + bit للـ output value
- `ir, ib` — register offset + bit للـ input read
- `dsr, dsb` — register offset + bit للـ drive strength

**Key details لـ A1:** الـ A1 بيستخدم regmap واحد (`reg_gpio`) لكل العمليات لأن `meson_a1_parse_dt_extra` بيوحّد الـ regmaps. مثلاً bank "P":
```
BANK_DS("P", GPIOP_0, GPIOP_12, 0, 12,
        0x3,0  /* pullen */,  0x4,0  /* pull */,
        0x2,0  /* dir */,     0x1,0  /* out */,
        0x0,0  /* in */,      0x5,0  /* ds */)
```
الـ `0x3`, `0x4`, إلخ هي word offsets داخل الـ regmap.

---

#### `BANK_PMX(n, f, l, r, o)`

```c
#define BANK_PMX(n, f, l, r, o) \
    {                           \
        .name   = n,            \
        .first  = f,            \
        .last   = l,            \
        .reg    = r,            \
        .offset = o,            \
    }
```

**ما بيعمله:** بيعرّف `meson_pmx_bank` الذي يحدد في أي register وأي bit offset تبدأ الـ mux bits لهذا البنك.

**Parameters:**
- `n` — اسم البنك
- `f`, `l` — نطاق الـ pins
- `r` — register offset داخل `reg_mux` regmap (word-addressable)
- `o` — bit offset لأول pin في البنك داخل الـ register

**Key details:** كل pin بياخد 4 bits في الـ mux register على الـ AXG/A1. مثلاً bank "P" (13 pins) بيبدأ من reg=0x0، bit=0. لما تكمّل الـ 8 pins الأولى (32 bits = 8×4)، الـ bank الجاي بيبدأ من reg=0x2 وهكذا.

```
Bank P:  reg=0x0, offset=0  → pins  0..12 → regs 0x0 & 0x1
Bank B:  reg=0x2, offset=0  → pins 13..19 → reg  0x2
Bank X:  reg=0x3, offset=0  → pins 20..36 → regs 0x3..0x5
Bank F:  reg=0x6, offset=0  → pins 37..49 → regs 0x6..0x7
Bank A:  reg=0x8, offset=0  → pins 50..61 → regs 0x8..0x9
```

---

### Group 3: Driver Registration — الـ probe والـ module setup

---

#### `meson_a1_parse_dt_extra()`

```c
int meson_a1_parse_dt_extra(struct meson_pinctrl *pc)
```

**ما بيعمله:** بيحلّ خصوصية الـ A1 SoC: على عكس الأجيال القديمة من Meson، الـ A1 بيستخدم single register space للـ GPIO والـ pull والـ drive strength كلها في نفس الـ regmap. الـ function بتوحّد `reg_pull`، `reg_pullen`، و`reg_ds` ليشاروا كلهم لنفس الـ `reg_gpio`.

**Parameters:**
- `pc` — مؤشر لـ `struct meson_pinctrl` التي تحتوي على الـ regmaps

**القيمة المُرجَعة:** دايماً 0 (success).

**Key details:**
- بيُستدعى من `meson_pinctrl_parse_dt()` في الـ core عبر function pointer `pc->data->parse_dt`
- على SoCs القديمة كانت فيه regmaps منفصلة لـ pull وللـ GPIO
- بعد الـ call، أي access لـ pull/ds بيروح لنفس الـ `reg_gpio` base address

**Pseudocode:**
```
meson_a1_parse_dt_extra(pc):
    pc->reg_pull   = pc->reg_gpio   // same regmap for pull
    pc->reg_pullen = pc->reg_gpio   // same regmap for pull-enable
    pc->reg_ds     = pc->reg_gpio   // same regmap for drive-strength
    return 0
```

---

#### `meson_pinctrl_probe()` (من `pinctrl-meson.c`، مُستخدَمة في الـ A1 driver)

```c
int meson_pinctrl_probe(struct platform_device *pdev)
```

**ما بيعمله:** الـ probe function الرئيسية للـ platform driver. بتخصص `meson_pinctrl`، بتقرأ الـ `of_device_id` match data لتحديد الـ A1-specific data، بتـ parse الـ DT، وبتسجّل الـ pinctrl device مع الـ framework ثم الـ GPIO chip.

**Parameters:**
- `pdev` — الـ `platform_device` اللي الـ kernel بيمرره عند الـ match مع `"amlogic,meson-a1-periphs-pinctrl"`

**القيمة المُرجَعة:** 0 عند النجاح، أو error code سالب.

**Key details — call chain:**
```
meson_pinctrl_probe(pdev)
    ├── devm_kzalloc()              // alloc meson_pinctrl
    ├── of_device_get_match_data()  // gets &meson_a1_periphs_pinctrl_data
    ├── meson_pinctrl_parse_dt(pc)
    │   ├── meson_map_resource()    // map reg_mux, reg_gpio, etc.
    │   └── pc->data->parse_dt(pc)  // → meson_a1_parse_dt_extra()
    ├── devm_pinctrl_register()     // register with pinctrl core
    │   ├── .pctlops = meson_pctrl_ops
    │   ├── .pmxops  = meson_axg_pmx_ops
    │   └── .confops = meson_pinconf_ops
    └── meson_gpiolib_register(pc)  // register GPIO chip
```

**Locking:** الـ devm variants بتضمن cleanup تلقائي عند unload. الـ `devm_pinctrl_register` بتحجز الـ lock الداخلي للـ framework.

---

#### `module_platform_driver(meson_a1_pinctrl_driver)`

```c
static struct platform_driver meson_a1_pinctrl_driver = {
    .probe  = meson_pinctrl_probe,
    .driver = {
        .name           = "meson-a1-pinctrl",
        .of_match_table = meson_a1_pinctrl_dt_match,
    },
};
module_platform_driver(meson_a1_pinctrl_driver);
```

**ما بيعمله:** Macro بيوسّع لـ `module_init` + `module_exit` اللي بيسجّلوا/يلغوا تسجيل الـ `platform_driver`. بيطلب من الـ driver core إنه يتصل بـ `meson_pinctrl_probe` لما يلاقي DT node بـ compatible `"amlogic,meson-a1-periphs-pinctrl"`.

**Key details:** الـ `.of_match_table` ربط الـ compatible string بـ `meson_a1_periphs_pinctrl_data` عبر `.data` field، اللي بيتقرأها `of_device_get_match_data()` في الـ probe.

---

#### `MODULE_DEVICE_TABLE(of, meson_a1_pinctrl_dt_match)`

```c
static const struct of_device_id meson_a1_pinctrl_dt_match[] = {
    {
        .compatible = "amlogic,meson-a1-periphs-pinctrl",
        .data = &meson_a1_periphs_pinctrl_data,
    },
    { },
};
MODULE_DEVICE_TABLE(of, meson_a1_pinctrl_dt_match);
```

**ما بيعمله:** بيصدّر الـ `of_device_id` table في الـ module binary metadata، وده بيخلي `udev` و`modprobe` يعرفوا تلقائياً إنهم يـ load هذا الـ module لما يلاقوا الـ compatible string في الـ Device Tree.

---

### Group 4: الـ meson_axg_pmx_ops — الـ Pinmux Operations (AXG-inherited)

الـ A1 driver بيورث الـ pinmux ops من AXG implementation عبر:

```c
.pmx_ops = &meson_axg_pmx_ops,
```

الـ ops بتتضمن:

| Operation | الغرض |
|---|---|
| `get_functions_count` | → `meson_pmx_get_funcs_count()` |
| `get_function_name` | → `meson_pmx_get_func_name()` |
| `get_function_groups` | → `meson_pmx_get_groups()` |
| `set_mux` | كتابة الـ function number في الـ pmx register |

**عند `set_mux`:**
1. الـ core بيمرر group index
2. الـ op بتحدد الـ `meson_pmx_bank` المناسب للـ pin
3. بتحسب bit position: `(pin - bank->first) * 4 + bank->offset`
4. بتكتب الـ function number (0–7) في الـ 4 bits دول عبر `regmap_update_bits()` على `reg_mux`

---

### Group 5: الـ Data Structures الختامية — الـ pinctrl_data الموحّدة

#### `meson_a1_periphs_pinctrl_data`

```c
static const struct meson_pinctrl_data meson_a1_periphs_pinctrl_data = {
    .name      = "periphs-banks",
    .pins      = meson_a1_periphs_pins,        // 62 pinctrl_pin_desc
    .groups    = meson_a1_periphs_groups,       // 62 GPIO + ~175 func groups
    .funcs     = meson_a1_periphs_functions,    // 40 functions
    .banks     = meson_a1_periphs_banks,        // 5 GPIO banks (P/B/X/F/A)
    .num_pins  = ARRAY_SIZE(...),
    .num_groups= ARRAY_SIZE(...),
    .num_funcs = ARRAY_SIZE(...),
    .num_banks = ARRAY_SIZE(...),
    .pmx_ops   = &meson_axg_pmx_ops,
    .pmx_data  = &meson_a1_periphs_pmx_banks_data,
    .parse_dt  = &meson_a1_parse_dt_extra,
};
```

**ما بيعمله:** الـ "كتالوج" الكامل للـ A1 periphs domain. الـ probe بيقرأه من الـ `of_device_id.data`، وكل عملية pinctrl/GPIO بتُحال لهذا الـ struct.

**Key details:**
- `parse_dt` function pointer بيسمح لكل SoC إنه يعمل post-DT-parse initialization بدون inheritance عميقة
- `pmx_data` بيحتوي على الـ `meson_axg_pmx_data` اللي بتقول للـ pmx ops إزاي تمشي على الـ pmx banks

---

### الـ Flow الكامل — من DT إلى GPIO Request

```
DT compatible: "amlogic,meson-a1-periphs-pinctrl"
        |
        v
module_platform_driver → platform_driver_register
        |
        v (kernel finds DT match)
meson_pinctrl_probe(pdev)
    ├── alloc meson_pinctrl (pc)
    ├── pc->data = &meson_a1_periphs_pinctrl_data
    ├── meson_pinctrl_parse_dt(pc)
    │   ├── map reg_mux  (من DT "mux" resource)
    │   ├── map reg_gpio (من DT "gpio" resource)
    │   └── meson_a1_parse_dt_extra(pc)
    │       └── reg_pull = reg_pullen = reg_ds = reg_gpio
    ├── devm_pinctrl_register(pc->desc)
    │   ├── pctlops: group/pin queries → meson_a1_periphs_groups[]
    │   ├── pmxops:  set_mux → BANK_PMX table → regmap write
    │   └── confops: pull/drive → BANK_DS table → regmap write
    └── meson_gpiolib_register(pc)
        └── register gpio_chip → /sys/class/gpio/...

Consumer (e.g. UART driver):
    pinctrl_select_state("uart_a")
        → find function "uart_a" in meson_a1_periphs_functions[]
        → find group "uart_a_tx" → pin GPIOX_11, func=1
        → set_mux: reg=0x3, bit=(11*4)=44 → write 1 → 4 bits at offset 44
```
## Phase 5: دليل الـ Debugging الشامل

الـ driver المعني هو `pinctrl-meson-a1.c` — الـ pin controller والـ GPIO driver للـ Amlogic Meson A1 SoC. الـ driver يتحكم في 5 banks (P, B, X, F, A) بمجموع 63 pin، ويستخدم الـ `regmap` للوصول للـ registers عبر 5 regmaps مختلفة: `reg_mux`, `reg_pullen`, `reg_pull`, `reg_gpio`, `reg_ds`.

---

### Software Level

#### 1. debugfs — المسارات والقراءة

الـ pinctrl subsystem يُصدّر معلومات مفصّلة في `/sys/kernel/debug/pinctrl/`.

```bash
# اعرض كل الـ pinctrl devices المُسجَّلة
ls /sys/kernel/debug/pinctrl/

# المسار المتوقع للـ A1 periphs
PINCTRL_DEV=$(ls /sys/kernel/debug/pinctrl/ | grep periphs)
echo "Device: $PINCTRL_DEV"

# اعرض كل الـ pins وحالتها
cat /sys/kernel/debug/pinctrl/${PINCTRL_DEV}/pins

# اعرض كل الـ groups المُعرَّفة (psram, sdcard, uart_a, ...)
cat /sys/kernel/debug/pinctrl/${PINCTRL_DEV}/pingroups

# اعرض الـ pinmux — أي function مُفعَّل على أي pin
cat /sys/kernel/debug/pinctrl/${PINCTRL_DEV}/pinmux-pins

# اعرض الـ functions المتاحة (gpio_periphs, uart_a, i2c0, ...)
cat /sys/kernel/debug/pinctrl/${PINCTRL_DEV}/pinmux-functions

# اعرض الـ pin config (pull, drive-strength, direction)
cat /sys/kernel/debug/pinctrl/${PINCTRL_DEV}/pinconf-pins
cat /sys/kernel/debug/pinctrl/${PINCTRL_DEV}/pinconf-groups
```

**مثال على output من `pinmux-pins`:**
```
pin 20 (GPIOX_0): meson-a1-pinctrl.0 function uart_c group uart_c_tx_x0
pin 21 (GPIOX_1): meson-a1-pinctrl.0 function uart_c group uart_c_rx_x1
pin 37 (GPIOF_0): UNCLAIMED
```

---

#### 2. sysfs — المسارات المهمة

```bash
# تحقق من الـ GPIO chip المُسجَّلة
ls /sys/class/gpio/
cat /sys/class/gpio/gpiochip*/label
cat /sys/class/gpio/gpiochip*/ngpio
cat /sys/class/gpio/gpiochip*/base

# الـ GPIO sysfs (legacy interface) — مثال على GPIOX_0 = base + 20
BASE=$(cat /sys/class/gpio/gpiochip*/base | head -1)
echo $((BASE + 20)) > /sys/class/gpio/export
cat /sys/class/gpio/gpio$((BASE + 20))/direction
cat /sys/class/gpio/gpio$((BASE + 20))/value

# الـ pinctrl device في sysfs
ls /sys/bus/platform/devices/ | grep pinctrl
ls /sys/bus/platform/drivers/meson-a1-pinctrl/

# regmap debugfs (مهم جداً لمشاهدة قيم الـ registers)
ls /sys/kernel/debug/regmap/
# ابحث عن الـ entries المتعلقة بالـ pinctrl
ls /sys/kernel/debug/regmap/ | grep -i gpio
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# mount tracefs لو مش موجود
mount -t tracefs tracefs /sys/kernel/tracing

cd /sys/kernel/tracing

# فعّل الـ pinctrl events
echo 1 > events/pinctrl/enable
# أو بالتفصيل:
ls events/pinctrl/
echo 1 > events/pinctrl/pinctrl_pins/enable
echo 1 > events/pinctrl/pinctrl_groups/enable

# فعّل الـ GPIO events
ls events/gpio/
echo 1 > events/gpio/enable

# استخدم function tracer لتتبع دوال الـ driver
echo function > current_tracer
echo 'meson*' > set_ftrace_filter
echo 1 > tracing_on
# ... افعل العملية المشبوهة ...
cat trace | head -50
echo 0 > tracing_on

# تتبع meson_pinctrl_probe تحديداً
echo meson_pinctrl_probe > set_ftrace_filter
echo function_graph > current_tracer
echo 1 > tracing_on
# ... أعد تحميل الـ module ...
cat trace
```

**مثال على output من GPIO events:**
```
kworker/0:1-45    [000] .....  123.456: gpio_direction: gpio=20 dir=in
kworker/0:1-45    [000] .....  123.457: gpio_value: gpio=20 get=1
```

---

#### 4. printk / dynamic debug

```bash
# فعّل الـ dynamic debug لملف الـ driver
echo 'file pinctrl-meson-a1.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file pinctrl-meson.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file pinctrl-meson-axg-pmx.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل لكل الـ pinctrl subsystem
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع الـ stack trace لكل message
echo 'file pinctrl-meson-a1.c +ps' > /sys/kernel/debug/dynamic_debug/control

# اعرض الـ dynamic debug entries الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep meson

# تعقّب الـ kernel log
dmesg -w | grep -E 'pinctrl|meson|gpio'
dmesg --level=debug,info | grep -i meson
```

لو أردت تفعيل الـ debug منذ boot:
```bash
# في kernel cmdline
dyndbg="file pinctrl-meson-a1.c +p"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_DEBUG_GPIO` | تفعيل الـ GPIO debugging — يُضيف checks إضافية |
| `CONFIG_PINCTRL_MESON` | الـ driver الأساسي — لازم يكون مُفعَّل |
| `CONFIG_DEBUG_PINCTRL` | تفعيل الـ pinctrl debugging الداخلي |
| `CONFIG_REGMAP_DEBUG` | debug الـ regmap transactions |
| `CONFIG_DEBUG_FS` | لازم مُفعَّل لعمل debugfs |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل الـ dynamic debug |
| `CONFIG_GPIOLIB_IRQCHIP` | debugging للـ GPIO IRQ |
| `CONFIG_GPIO_SYSFS` | تصدير الـ GPIO في sysfs |
| `CONFIG_PINCTRL_SINGLE` | مفيد للمقارنة |
| `CONFIG_PROVE_LOCKING` | يكتشف الـ locking bugs في الـ pinctrl |
| `CONFIG_LOCKDEP` | مع الـ locking debugging |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_DEBUG_GPIO|CONFIG_PINCTRL|CONFIG_REGMAP_DEBUG'
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# gpioinfo (من libgpiod) — يعرض كل الـ GPIO lines
gpioinfo

# gpiodetect — يعرض الـ GPIO chips
gpiodetect

# gpioget / gpioset — قراءة وكتابة الـ GPIO
gpioget gpiochip0 20           # اقرأ GPIOX_0
gpioset gpiochip0 20=1         # اكتب 1 على GPIOX_0

# pinctrl tool (لو موجود في الـ distro)
# بديل: استخدم debugfs مباشرة

# iomem — تحقق من الـ memory regions المحجوزة
cat /proc/iomem | grep -i gpio
cat /proc/iomem | grep -i pinctrl

# تحقق من الـ interrupts المرتبطة
cat /proc/interrupts | grep -i gpio
```

---

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `probe with driver meson-a1-pinctrl failed with error -22` | `EINVAL` — خطأ في الـ DT أو الـ reg-names مش موجودة | تحقق من الـ DT: `gpio`, `mux`, `pull`, `pull-enable`, `ds` |
| `can't get GPIO chip` | فشل في تسجيل الـ GPIO chip | تحقق من `CONFIG_GPIOLIB` ومن عدم تعارض الـ base numbers |
| `failed to request GPIO` | الـ pin محجوز من driver تاني | استخدم `cat /sys/kernel/debug/pinctrl/*/pinmux-pins` لمعرفة المانع |
| `pin X already requested` | تعارض في طلب الـ pin | تحقق من الـ DT — نفس الـ pin متطلوب من أكتر من consumer |
| `invalid pin X` | رقم الـ pin خارج الـ range | تحقق من الـ DT pinctrl-0 — الـ numbers لازم تتطابق مع meson-a1-gpio.h |
| `could not find pinmux function` | الـ function name في الـ DT مش موجود في الـ driver | تحقق من إملاء الـ function name في الـ DT vs. `meson_a1_periphs_functions[]` |
| `regmap mmio failed` | فشل في الوصول لـ MMIO registers | تحقق من الـ `reg` property في الـ DT ومن الـ clock/power domain |
| `failed to add pinctrl device` | خطأ عام في تسجيل الـ pinctrl | راجع الـ full dmesg وتحقق من الـ resources |
| `pinctrl: pins are not configured` | الـ consumer طلب state غير مُعرَّفة | تأكد أن `pinctrl-names` و `pinctrl-0` صح في الـ consumer node |
| `GPIO line X is busy` | الـ line محجوزة من kernel code آخر | استخدم `gpioinfo` لمعرفة المالك |

---

#### 8. مواضع dump_stack() و WARN_ON() الاستراتيجية

الأماكن اللي تُضيف فيها `WARN_ON()` أو `dump_stack()` لو كنت بتحقق في مشكلة:

```c
/* في meson_pinctrl_probe — بعد فشل regmap init */
if (IS_ERR(pc->reg_gpio)) {
    dump_stack();   /* شوف من طلب الـ probe */
    return PTR_ERR(pc->reg_gpio);
}

/* في دالة meson_axg_pmx_set_mux — تحقق من الـ func value */
WARN_ON(func > 7);  /* الـ A1 يدعم func 0-7 كحد أقصى */

/* في meson_gpio_request — تحقق من الـ pin range */
WARN_ON(pin >= pc->data->num_pins);

/* في meson_calc_reg_and_bit — لو الـ bank مش موجود */
WARN_ON(!bank);

/* في parse_dt — لو regmap fwnode مش موجود */
WARN_ON(!fwnode_device_is_compatible(fwnode, "amlogic,meson-a1-periphs-pinctrl"));
```

**نقاط مهمة:**
- أي دالة `set_mux` → تحقق إن الـ `func` value يتطابق مع الـ GROUP macro في الملف
- الـ `meson_a1_parse_dt_extra` → نقطة دخول critical لأن الـ A1 يستخدمها بدل `meson8_aobus_parse_dt_extra`
- الـ `BANK_DS` macro → يُعرِّف الـ register offsets لكل bank، أي خطأ هنا يودي لكتابة على registers غلط

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يتطابق مع الـ Kernel State

الـ Meson A1 periphs registers layout (من `BANK_DS` macros في الكود):

| Bank | PULLEN reg | PULL reg | DIR reg | OUT reg | IN reg | DS reg |
|---|---|---|---|---|---|---|
| P (GPIOP) | 0x3 | 0x4 | 0x2 | 0x1 | 0x0 | 0x5 |
| B (GPIOB) | 0x13 | 0x14 | 0x12 | 0x11 | 0x10 | 0x15 |
| X (GPIOX) | 0x23 | 0x24 | 0x22 | 0x21 | 0x20 | 0x25 |
| F (GPIOF) | 0x33 | 0x34 | 0x32 | 0x31 | 0x30 | 0x35 |
| A (GPIOA) | 0x43 | 0x44 | 0x42 | 0x41 | 0x40 | 0x45 |

الـ PMX registers (من `BANK_PMX`):

| Bank | MUX reg base | Pins | Bits per pin |
|---|---|---|---|
| P | 0x0 | GPIOP_0..12 | 4 bits |
| B | 0x2 | GPIOB_0..6 | 4 bits |
| X | 0x3 | GPIOX_0..16 | 4 bits |
| F | 0x6 | GPIOF_0..12 | 4 bits |
| A | 0x8 | GPIOA_0..11 | 4 bits |

```bash
# تحقق من الـ pin direction عبر الـ sysfs
cat /sys/kernel/debug/pinctrl/*/pinconf-pins | grep GPIOX_0

# قارن مع القيمة المباشرة من الـ register
# GPIOX DIR reg = 0x22 * 4 = offset 0x88 من base address
```

---

#### 2. Register Dump عبر devmem2 / /dev/mem

أول خطوة هي معرفة الـ base address من الـ DT أو `/proc/iomem`:

```bash
# اعرف الـ base address للـ GPIO registers
cat /proc/iomem | grep -i gpio
# مثال output:
# ff800000-ff800fff : ff800000.periphs-gpio

# استخدم devmem2 لقراءة الـ registers (مثال على GPIOX bank)
# GPIOX IN reg = base + 0x20*4 = base + 0x80
devmem2 0xff800080 w   # اقرأ GPIOX input register
devmem2 0xff800084 w   # GPIOX output register
devmem2 0xff800088 w   # GPIOX direction register
devmem2 0xff80008C w   # GPIOX pull-enable register
devmem2 0xff800090 w   # GPIOX pull register

# لقراءة الـ MUX register لـ GPIOX (reg 0x3 = offset 0xC)
devmem2 0xff80000C w   # GPIOX MUX reg0 — pins 0-7
devmem2 0xff800010 w   # GPIOX MUX reg1 — pins 8-15
devmem2 0xff800014 w   # GPIOX MUX reg2 — pin 16

# مثال على تفسير MUX register لـ GPIOX_0 (bits 3:0):
# 0x0 = GPIO mode
# 0x1 = sdcard_d0_x
# 0x2 = i2c2_sck_x0
# 0x3 = uart_c_tx_x0
# 0x4 = pwm_e_x2 (هنا غلط لأن pwm_e_x2 على GPIOX_2)
```

```bash
# dump كامل لـ GPIO registers عبر /dev/mem (لو devmem2 مش موجود)
python3 -c "
import mmap, struct
base = 0xff800000
size = 0x1000
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), size, offset=base, access=mmap.ACCESS_READ)
    for i in range(0, 0x100, 4):
        val = struct.unpack('<I', m[i:i+4])[0]
        print(f'offset 0x{i:03x}: 0x{val:08x}')
"
```

---

#### 3. Logic Analyzer / Oscilloscope

للتحقق من سلوك الـ pins على الـ hardware:

```
Signal        | الـ Pin       | ما تراقبه
--------------|---------------|------------------------------------------
UART TX       | GPIOX_11      | Baud rate، idle high، start/stop bits
I2C SCL       | GPIOF_9/11    | 100kHz أو 400kHz، hold times
I2C SDA       | GPIOF_10/12   | ACK/NAK، repeated start
SPI CLK       | GPIOX_4/10    | Polarity، phase (CPOL/CPHA)
SPI MOSI      | GPIOX_2/7     | Data integrity
PSRAM CLK     | GPIOP_1       | تحقق من التردد والـ duty cycle
SD Card CLK   | GPIOB_4/X_4   | 400kHz (init) أو 25/50MHz (data)
PWM output    | حسب الـ group  | Frequency، duty cycle
JTAG TCK      | GPIOF_4       | تردد منتظم أثناء الـ debugging
```

**نصائح:**
- الـ pull resistors على الـ A1 يمكن تكون internal أو external — لو الـ signal مش clean، تحقق من الـ `PULL` و `PULLEN` registers
- الـ drive strength لها 4 قيم: 500µA, 2500µA, 3000µA, 4000µA — لو الـ signal slow، زود الـ DS
- الـ open-drain signals (I2C) لازم يكون الـ pin direction = input مع الـ pull-up مُفعَّل

---

#### 4. Hardware Issues الشائعة وأنماطها في الـ Kernel Log

| المشكلة الهاردوير | النمط في dmesg | الحل |
|---|---|---|
| الـ pin مش initialized (float) | الـ GPIO يقرأ random values | تحقق من الـ pinctrl state في الـ DT consumer |
| wrong pull config | `I2C timeout`, `SPI transfer failed` | تحقق من PULLEN/PULL registers |
| drive strength ضعيف | إشارة slow على الـ scope | اضبط `drive-strength-microamp` في الـ DT |
| تعارض في الـ mux | `pin X already requested by Y` | فحص الـ DT للـ overlapping pinctrl consumers |
| voltage domain mismatch | الـ pin يقرأ دايما 0 رغم الـ input | تحقق من الـ power domain للـ bank |
| الـ GPIO base address غلط في DT | `regmap mmio failed` أو kernel panic | قارن الـ DT `reg` مع الـ A1 datasheet |
| الـ clock للـ GPIO controller مش enabled | `probe failed -EPROBE_DEFER` | تحقق من الـ `clocks` property في الـ DT node |

---

#### 5. Device Tree Debugging

الـ compatible string المطلوبة: `"amlogic,meson-a1-periphs-pinctrl"`

```bash
# اعرض الـ DT node للـ pinctrl
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 30 'meson-a1-periphs-pinctrl'

# أو مباشرة
cat /sys/firmware/devicetree/base/soc/bus@*/pinctrl@*/compatible
ls /sys/firmware/devicetree/base/soc/bus@*/pinctrl@*/

# تحقق من الـ reg property (لازم يكون 5 regions: gpio, mux, pull, pull-enable, ds)
hexdump -C /sys/firmware/devicetree/base/soc/bus@*/pinctrl@*/reg

# تحقق من الـ reg-names
strings /sys/firmware/devicetree/base/soc/bus@*/pinctrl@*/reg-names
# المتوقع: gpio mux pull pull-enable ds

# تحقق من كل consumer node
grep -r 'pinctrl-0' /sys/firmware/devicetree/base/ 2>/dev/null | head -20

# فحص الـ DT source
find /boot -name '*.dtb' | xargs -I{} dtc -I dtb -O dts {} 2>/dev/null | grep -A 20 'meson-a1-periphs'
```

**الـ DT الصحيح للـ A1 periphs pinctrl:**
```dts
periphs_pinctrl: pinctrl@4000 {
    compatible = "amlogic,meson-a1-periphs-pinctrl";
    reg-names = "gpio", "mux", "pull", "pull-enable", "ds";
    reg = <0x4000 0x...>,   /* gpio */
          <0x0000 0x...>,   /* mux */
          <0x4000 0x...>,   /* pull */
          <0x4000 0x...>,   /* pull-enable */
          <0x4000 0x...>;   /* ds */
    gpio-controller;
    #gpio-cells = <2>;
    interrupt-controller;
    #interrupt-cells = <2>;
};
```

---

### Practical Commands

#### جاهز للـ Copy-Paste

```bash
#!/bin/bash
# =============================================================
# Meson A1 Pinctrl Full Debug Script
# =============================================================

PINCTRL_BASE="/sys/kernel/debug/pinctrl"
PINCTRL_DEV=$(ls $PINCTRL_BASE 2>/dev/null | grep -i periphs | head -1)

echo "=== [1] Pinctrl Device ==="
echo "Found: $PINCTRL_DEV"

echo ""
echo "=== [2] All Pins Status ==="
cat "$PINCTRL_BASE/$PINCTRL_DEV/pins" 2>/dev/null || echo "NOT AVAILABLE"

echo ""
echo "=== [3] Pinmux (which function on which pin) ==="
cat "$PINCTRL_BASE/$PINCTRL_DEV/pinmux-pins" 2>/dev/null | head -40

echo ""
echo "=== [4] Pin Config (pull, drive-strength) ==="
cat "$PINCTRL_BASE/$PINCTRL_DEV/pinconf-pins" 2>/dev/null | head -40

echo ""
echo "=== [5] GPIO Chips ==="
gpiodetect 2>/dev/null || ls /sys/class/gpio/gpiochip*/label 2>/dev/null

echo ""
echo "=== [6] GPIO Lines ==="
gpioinfo 2>/dev/null | head -40

echo ""
echo "=== [7] IRQ Map ==="
cat /proc/interrupts | grep -i gpio | head -20

echo ""
echo "=== [8] IOmem Regions ==="
cat /proc/iomem | grep -iE 'gpio|pinctrl'

echo ""
echo "=== [9] Kernel Messages ==="
dmesg | grep -iE 'meson.*pin|pinctrl.*meson|gpio.*meson' | tail -30

echo ""
echo "=== [10] Regmap Debug ==="
ls /sys/kernel/debug/regmap/ 2>/dev/null | grep -i gpio
```

```bash
# تفعيل dynamic debug لكل الـ meson pinctrl drivers
for f in pinctrl-meson-a1.c pinctrl-meson.c pinctrl-meson-axg-pmx.c; do
    echo "file $f +p" > /sys/kernel/debug/dynamic_debug/control && echo "Enabled: $f"
done

# مراقبة الـ kernel log في real-time
dmesg -w --level=debug,info,warn,err 2>/dev/null | grep -iE 'meson|pinctrl|gpio'
```

```bash
# فحص pin معين بالتفصيل — مثال: GPIOX_0 = pin index 20
PIN_NUM=20
PIN_NAME="GPIOX_0"

echo "=== Pin $PIN_NAME (index $PIN_NUM) ==="
cat /sys/kernel/debug/pinctrl/*/pinmux-pins 2>/dev/null | grep "pin $PIN_NUM"
cat /sys/kernel/debug/pinctrl/*/pinconf-pins 2>/dev/null | grep "pin $PIN_NUM"

# legacy sysfs export
BASE=$(cat /sys/class/gpio/gpiochip0/base 2>/dev/null)
GPIO_NUM=$((BASE + PIN_NUM))
echo $GPIO_NUM > /sys/class/gpio/export 2>/dev/null
echo "Direction: $(cat /sys/class/gpio/gpio$GPIO_NUM/direction 2>/dev/null)"
echo "Value:     $(cat /sys/class/gpio/gpio$GPIO_NUM/value 2>/dev/null)"
echo $GPIO_NUM > /sys/class/gpio/unexport 2>/dev/null
```

```bash
# تحقق من الـ MUX register لـ GPIOX bank
# الـ GPIOX PMX reg base = 0x3 (units of 32-bit words)
# كل 4 pins تاخد register واحد (4 bits per pin)
# GPIOX_0..7  → reg 0x3 → offset 0x3*4 = 0xC
# GPIOX_8..15 → reg 0x4 → offset 0x10
# GPIOX_16    → reg 0x5 → offset 0x14

# اقرأ من regmap debugfs (أسهل من devmem2)
REG_REGMAP=$(ls /sys/kernel/debug/regmap/ | grep -i gpio | head -1)
if [ -n "$REG_REGMAP" ]; then
    echo "=== Regmap Dump ==="
    cat /sys/kernel/debug/regmap/$REG_REGMAP/registers | head -50
fi
```

```bash
# فعّل ftrace لتتبع pinmux operations
cd /sys/kernel/tracing
echo 0 > tracing_on
echo function > current_tracer
echo 'meson_axg_pmx_set*' > set_ftrace_filter
echo 'meson_pinctrl*' >> set_ftrace_filter
echo 1 > tracing_on

# أعد تحميل consumer device لتفعيل الـ pinctrl
echo "Trigger your device operation now..."
sleep 5

echo 0 > tracing_on
echo "=== Trace Results ==="
cat trace | grep -v '^#' | head -50
echo nop > current_tracer
```

#### تفسير الـ Output

```
# output من pinmux-pins:
pin 20 (GPIOX_0): meson-a1-pinctrl.0 function uart_c group uart_c_tx_x0
#                 ^-- الـ pinctrl device   ^-- الـ function   ^-- الـ group
# يعني GPIOX_0 متوصّل على uart_c TX — func=3 في الـ MUX register

pin 37 (GPIOF_0): UNCLAIMED
# يعني الـ pin في الـ GPIO mode أو غير مُطلوب من أي consumer

# output من pinconf-pins:
pin 20 (GPIOX_0): input bias pull up, drive strength 500 uA
#                  ^-- pull-up مُفعَّل       ^-- drive strength الأقل

# output من regmap registers:
30: 00000001
# offset 0x30 = GPIOF IN register → bit 0 = GPIOF_0 = 1 (high)
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: IoT Gateway على Amlogic A1 — UART مش شغال بعد تغيير الـ DT

#### العنوان
**الـ UART console مش بيظهر على IoT gateway بعد bring-up**

#### السياق
الشغل: industrial IoT gateway بيشتغل على Amlogic Meson A1 SoC. المنتج محتاج debug console على UART-A وكمان UART-B لربطه بـ cellular modem. الـ board designer اختار GPIOX_11/GPIOX_12 لـ UART-A وGPIOX_7/GPIOX_8 لـ UART-B.

#### المشكلة
بعد ما الفريق كتب الـ device tree وبوت الكيرنل، مفيش أي output على console. الـ earlycon شغال بس بعد ما الـ pinctrl driver بيتحمل — silence تام.

#### التحليل
بص على الـ file: UART-A بيستخدم:
```c
/* uart_a */
static const unsigned int uart_a_tx_pins[]  = { GPIOX_11 };
static const unsigned int uart_a_rx_pins[]  = { GPIOX_12 };
```
**الـ GROUP** بتاعهم:
```c
GROUP(uart_a_tx, 1),  /* bank X, func1 */
GROUP(uart_a_rx, 1),
```
وفي الـ PMX bank:
```c
BANK_PMX("X", GPIOX_0, GPIOX_16, 0x3, 0),
```
يعني الـ mux register لـ bank X هو `0x3`، كل pin بياخد 4 bits. GPIOX_11 = offset `11*4 = 44` bits من بداية reg `0x3`.

الـ engineer كتب في الـ DT:
```dts
&uart_a {
    pinctrl-0 = <&uart_a_pins>;
    pinctrl-names = "default";
    status = "okay";
};
```
بس نسي يعرّف الـ `uart_a_pins` node في الـ pinctrl section، فالـ `meson_axg_pmx_ops` ما اتنفذتش، والـ pin فضل على `func0` = GPIO mode.

#### الحل
```dts
/* في الـ &periphs_pinctrl */
uart_a_pins: uart-a-pins {
    mux {
        groups = "uart_a_tx", "uart_a_rx";
        function = "uart_a";
        bias-disable;
    };
};
```
بعدين verify:
```bash
# بعد البوت
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep GPIOX_11
# المفروض يظهر: GPIOX_11: uart_a_tx (uart_a)
```
لو الـ pin لسه GPIO:
```bash
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep GPIOX_11
# GPIOX_11: GPIO GPIOX_11  <-- ده المشكلة
```

#### الدرس المستفاد
الـ pin definition في `pinctrl-meson-a1.c` صح — المشكلة دايما في الـ DT consumer. لازم تتأكد إن كل function معرّفة في `meson_a1_periphs_functions[]` موجودة بالاسم الصح في الـ DT `groups` property.

---

### السيناريو الثاني: Android TV Box — تعارض بين SPI Flash وSD Card على bank B

#### العنوان
**الـ SPI NOR flash بيتعارض مع SD card على Amlogic A1 في Android TV box**

#### السياق
Android TV box رخيص بيستخدم Amlogic A1. الـ board engineer عايز يبوت من SPI NOR flash (للـ bootloader) ويفضل يستخدم microSD card للـ Android data partition. البنية:
- SPI flash: GPIOB_0..GPIOB_5 (func1 = spif)
- SD card: GPIOB_0..GPIOB_5 (func2 = sdcard_b)

#### المشكلة
الـ kernel بيبوت من NAND بنجاح، بس لما بيحاول يـ mount الـ SD card، الـ MMC driver بيـ fail بـ timeout errors.

#### التحليل
بص على الـ source:
```c
/* spif — bank B func1 */
static const unsigned int spif_mo_pins[]     = { GPIOB_0 };
static const unsigned int spif_mi_pins[]     = { GPIOB_1 };
static const unsigned int spif_wp_n_pins[]   = { GPIOB_2 };
static const unsigned int spif_hold_n_pins[] = { GPIOB_3 };
static const unsigned int spif_clk_pins[]    = { GPIOB_4 };
static const unsigned int spif_cs_pins[]     = { GPIOB_5 };

/* sdcard — bank B func2 */
static const unsigned int sdcard_d0_b_pins[] = { GPIOB_0 };
static const unsigned int sdcard_d1_b_pins[] = { GPIOB_1 };
static const unsigned int sdcard_d2_b_pins[] = { GPIOB_2 };
static const unsigned int sdcard_d3_b_pins[] = { GPIOB_3 };
static const unsigned int sdcard_clk_b_pins[]= { GPIOB_4 };
static const unsigned int sdcard_cmd_b_pins[]= { GPIOB_5 };
```
نفس الـ pins! الـ PMX bank لـ B:
```c
BANK_PMX("B", GPIOB_0, GPIOB_6, 0x2, 0),
```
الـ `meson_axg_pmx_ops` بيكتب الـ function number (1 أو 2) في الـ 4-bit field الخاص بكل pin في register `0x2`. لو الـ bootloader سابهم على func1 (spif) والـ kernel DT بيطلب func2 (sdcard)، لازم يتعمل switch صريح.

المشكلة إن في الـ DT في نفس الوقت:
```dts
&spifc { status = "okay"; };  /* بيطلب spif function */
&sd_emmc_b { status = "okay"; };  /* بيطلب sdcard function */
```

الـ pinctrl core بيحاول يـ claim نفس الـ pins لاتنين — بيفشل الثاني.

#### الحل
لازم تختار واحد فقط في وقت واحد. لو الـ SPI flash للـ bootloader بس وما محتاجوش من الـ Linux kernel:
```dts
&spifc {
    status = "disabled";  /* الـ bootloader يشتغل عليه، الكيرنل لا */
};

&sd_emmc_b {
    pinctrl-0 = <&sdcard_b_pins>;
    pinctrl-names = "default";
    status = "okay";
};
```
أو لو محتاج الاتنين، استخدم الـ alternative sdcard على bank X:
```c
/* sdcard على bank X — func1 */
GROUP(sdcard_d0_x, 1),
GROUP(sdcard_clk_x, 1),
/* إلخ */
```

#### الدرس المستفاد
**الـ pin sharing بين spif و sdcard_b على bank B هو design intentional** — الـ A1 SoC ميتوقعش تشتغل الاتنين في نفس الوقت. دايما راجع الـ pin assignment table في `meson_a1_periphs_groups[]` قبل ما تـ enable أكتر من peripheral.

---

### السيناريو الثالث: Industrial ECU — PWM مش بيشتغل على المتوقع

#### العنوان
**الـ PWM لـ fan control بياخد output على pin غلط في automotive ECU**

#### السياق
Industrial ECU بيستخدم Amlogic A1 لـ low-power edge processing. الـ design محتاج PWM-D لـ temperature-controlled fan. الـ hardware team وصّل الـ fan driver IC على GPIOX_10.

#### المشكلة
الـ fan مش بتتحرك رغم إن الـ PWM driver بيـ report إنه شغال. الـ oscilloscope على GPIOX_10 مش بيشوف أي signal.

#### التحليل
الـ engineer بيبص في الـ file:
```c
/* pwm_d */
static const unsigned int pwm_d_x10_pins[] = { GPIOX_10 };
static const unsigned int pwm_d_x13_pins[] = { GPIOX_13 };
static const unsigned int pwm_d_x15_pins[] = { GPIOX_15 };
static const unsigned int pwm_d_f_pins[]   = { GPIOF_11 };
```
وفي الـ groups:
```c
GROUP(pwm_d_x15, 1),   /* bank X func1 */
GROUP(pwm_d_x13, 2),   /* bank X func2 */
GROUP(pwm_d_x10, 6),   /* bank X func6 !! */
GROUP(pwm_d_f,   2),   /* bank F func2 */
```
الـ `pwm_d_x10` موجود على **func6** مش func1. الـ engineer كتب في الـ DT:
```dts
pwm_d_pins: pwm-d-pins {
    mux {
        groups = "pwm_d_x10";
        function = "pwm_d";
    };
};
```
ده صح! بس المشكلة إن GPIOX_9 و GPIOX_10 متطلوبين كمان من الـ TDM-A driver:
```c
GROUP(tdm_a_fs,   1),   /* GPIOX_9,  func1 */
GROUP(tdm_a_sclk, 1),   /* GPIOX_10, func1 */
```
والـ audio subsystem enabled في الـ DT وبيـ claim GPIOX_10 على func1 قبل الـ PWM!

#### الحل
1. Disable الـ TDM-A لو مش محتاجه:
```dts
&tdm_a { status = "disabled"; };
```
2. أو use PWM-D على pin مش متعارض (GPIOX_13 أو GPIOX_15):
```dts
pwm_d_pins: pwm-d-pins {
    mux {
        groups = "pwm_d_x13";  /* func2 على X13، مش متعارض */
        function = "pwm_d";
    };
};
```
Debug عملي:
```bash
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep GPIOX_10
# لو بيظهر tdm_a_sclk — الـ TDM حاجز الـ pin
```

#### الدرس المستفاد
الـ function number في `GROUP(name, func)` مهم جداً — نفس الـ pin ممكن يظهر في أكتر من function بـ func numbers مختلفة. لازم تتأكد مفيش driver تاني بيـ claim نفس الـ pin بـ func number أقل (أولوية أعلى).

---

### السيناريو الرابع: Custom Board Bring-up — I2C مش بيـ detect devices

#### العنوان
**الـ I2C2 مش بيشوف أي device على custom Amlogic A1 board أثناء bring-up**

#### السياق
Custom board جديد بيستخدم Amlogic A1 لـ smart home hub. الـ board عنده:
- I2C2 على GPIOA_4/GPIOA_5 لربطه بـ temperature sensor (TMP102)
- I2C1 على GPIOA_10/GPIOA_11 لربطه بـ EEPROM

#### المشكلة
```bash
i2cdetect -y 2
# No devices found — blank table
```

#### التحليل
الـ engineer بيراجع الـ file:

**الـ I2C2 options على bank A:**
```c
static const unsigned int i2c2_sck_a4_pins[] = { GPIOA_4 };
static const unsigned int i2c2_sda_a5_pins[] = { GPIOA_5 };
static const unsigned int i2c2_sck_a8_pins[] = { GPIOA_8 };
static const unsigned int i2c2_sda_a9_pins[] = { GPIOA_9 };
```
وفي الـ groups:
```c
GROUP(i2c2_sck_a4, 3),   /* bank A func3 */
GROUP(i2c2_sda_a5, 3),   /* bank A func3 */
GROUP(i2c2_sck_a8, 5),   /* bank A func5 */
GROUP(i2c2_sda_a9, 5),   /* bank A func5 */
```
الـ engineer كتب الـ DT بالاسم الغلط:
```dts
/* غلط */
i2c2_pins: i2c2-pins {
    mux {
        groups = "i2c2_sck_a", "i2c2_sda_a";  /* اسم غلط! */
        function = "i2c2";
    };
};
```
الاسم الصح موجود في `meson_a1_periphs_groups[]` هو `"i2c2_sck_a4"` و`"i2c2_sda_a5"` — مش `"i2c2_sck_a"`.

الـ `meson_pinctrl_probe` بيـ fail بـ:
```
pinctrl: requested group 'i2c2_sck_a' not found
```
بس الـ message ممكن تتلف في الـ boot log.

#### الحل
```dts
i2c2_a4_pins: i2c2-a4-pins {
    mux {
        /* الاسم الصح من meson_a1_periphs_groups */
        groups = "i2c2_sck_a4", "i2c2_sda_a5";
        function = "i2c2";
        bias-pull-up;
    };
};

&i2c2 {
    pinctrl-0 = <&i2c2_a4_pins>;
    pinctrl-names = "default";
    clock-frequency = <400000>;
    status = "okay";

    tmp102@48 {
        compatible = "ti,tmp102";
        reg = <0x48>;
    };
};
```
Verify:
```bash
# الـ kernel log أثناء البوت
dmesg | grep pinctrl | grep i2c
# المفروض: meson-a1-pinctrl: pin GPIOA_4 already requested?
# أو: i2c2_sck_a4 (i2c2) مقبولة

i2cdetect -y 2
# المفروض يظهر 0x48
```

#### الدرس المستفاد
أسماء الـ groups في الـ DT لازم تتطابق **حرفياً** مع الأسماء المعرّفة في `meson_a1_periphs_groups[]` — الـ macro `GROUP(i2c2_sck_a4, 3)` يحول لاسم string هو `"i2c2_sck_a4"` بالضبط. أي اختلاف في الاسم = silent failure أو kernel warning.

---

### السيناريو الخامس: Smart Speaker — JTAG مش شغال أثناء الـ debug

#### العنوان
**الـ JTAG debugger مش بيـ connect على Amlogic A1 smart speaker بعد enable الـ audio**

#### السياق
فريق embedded بيطور smart speaker بيستخدم Amlogic A1. المنتج فيه:
- TDM-B لـ 6-channel audio output على GPIOA_1..GPIOA_8
- JTAG-A للـ debug على GPIOF_4..GPIOF_7

في مرحلة الـ bring-up كل شيء تمام، بس بعد ما الـ audio driver اتضاف، الـ JTAG بطّل يشتغل.

#### المشكلة
الـ OpenOCD بيـ report:
```
Error: JTAG scan chain interrogation failed: all ones
Error: Check JTAG interface, connections...
```

#### التحليل
**الـ JTAG-A pins:**
```c
static const unsigned int jtag_a_clk_pins[] = { GPIOF_4 };
static const unsigned int jtag_a_tms_pins[] = { GPIOF_5 };
static const unsigned int jtag_a_tdi_pins[] = { GPIOF_6 };
static const unsigned int jtag_a_tdo_pins[] = { GPIOF_7 };
```
Groups:
```c
GROUP(jtag_a_clk, 1),  /* bank F func1 */
GROUP(jtag_a_tms, 1),
GROUP(jtag_a_tdi, 1),
GROUP(jtag_a_tdo, 1),
```
دلوقتي بص على الـ **SWD** pins:
```c
static const unsigned int swclk_pins[] = { GPIOF_4 };
static const unsigned int swdio_pins[] = { GPIOF_5 };
```
وبص على الـ SPDIF:
```c
static const unsigned int spdif_in_f6_pins[] = { GPIOF_6 };
static const unsigned int spdif_in_f7_pins[] = { GPIOF_7 };
```

الـ audio driver لما اتضاف، enable الـ SPDIF input:
```dts
&spdif_in {
    pinctrl-0 = <&spdif_f6_pins>;
    status = "okay";
};
```
ده بيـ claim GPIOF_6 و GPIOF_7 — نفس pins الـ JTAG TDI/TDO.

الـ `meson_axg_pmx_ops` بيكتب func2 في register `0x6` (bank F PMX) للـ pins دي:
```c
BANK_PMX("F", GPIOF_0, GPIOF_12, 0x6, 0),
```
فالـ JTAG hardware بيشوف pins مش في JTAG mode.

#### الحل
**للـ production:** disable الـ JTAG node في الـ production DT.

**للـ debug build:** disable الـ SPDIF على F6/F7 وuse بديل:
```dts
/* debug DT overlay */
&spdif_in {
    status = "disabled";  /* تعطيل SPDIF على F6/F7 */
};

&jtag_a {
    pinctrl-0 = <&jtag_a_pins>;
    pinctrl-names = "default";
    status = "okay";
};
```
أو استخدم SWD بدل JTAG (أقل pins):
```dts
sw_pins: sw-pins {
    mux {
        groups = "swclk", "swdio";  /* GPIOF_4, GPIOF_5 فقط */
        function = "sw";
    };
};
```
هكذا GPIOF_6/GPIOF_7 متاحين للـ SPDIF.

Debug للـ conflict:
```bash
# شوف مين حاجز الـ pins
cat /sys/kernel/debug/pinctrl/periphs-banks/pinmux-pins | grep -E "GPIOF_[4-7]"

# المتوقع في حالة conflict:
# GPIOF_4: GPIO GPIOF_4     (jtag مش active)
# GPIOF_6: spdif_in_f6 (spdif_in)
# GPIOF_7: spdif_in_f7 (spdif_in)
```

#### الدرس المستفاد
الـ JTAG والـ SPDIF بيتشاركوا نفس الـ pins على bank F في Amlogic A1. في الـ production image دايما disable الـ JTAG. وفي الـ debug image، اختار: إما JTAG (GPIOF_4..7 كلها) أو SPDIF-in (F6 أو F7) + SWD (F4+F5 فقط). الـ pinctrl-meson-a1.c بيحدد الـ constraints دي بوضوح في الـ `GROUP` definitions — الـ hardware reality موثقة في الـ code.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ pinctrl subsystem وتطبيقاتها على Amlogic Meson SoCs:

| المقال | الوصف |
|--------|--------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | مقدمة شاملة للـ pinctrl subsystem وازاي اتضاف في kernel 3.1 |
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | أول patch series أنشأ الـ subsystem من الأساس |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | نقاش مراجعة النسخة السابعة من الـ subsystem |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | التوثيق الرسمي الأولي للـ subsystem |
| [pinctrl: meson-a1: add pinctrl driver](https://lwn.net/Articles/804174/) | الـ patch series اللي أضافت driver الـ Meson A1 تحديداً — ده المصدر المباشر لملف `pinctrl-meson-a1.c` |
| [Pinctrl driver for Amlogic Meson SoCs](https://lwn.net/Articles/616225/) | أول driver للـ Amlogic Meson SoCs بقلم Beniamino Galvani |
| [Amlogic Meson pinctrl driver](https://lwn.net/Articles/620822/) | مراجعة أعمق للـ driver الأصلي مع device tree bindings |
| [irqchip: meson: add support for the gpio interrupt controller](https://lwn.net/Articles/703946/) | إضافة GPIO interrupt controller لـ Meson SoCs |
| [pinctrl: meson-s4: add pinctrl driver](https://lwn.net/Articles/879912/) | مثال على SoC لاحق يتبع نفس نمط الـ A1 |
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532909/) | مقدمة للـ GPIO subsystem اللي بيتكامل مع pinctrl |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/) | إضافة واجهة pin config generic للـ subsystem |

---

### توثيق الـ Kernel الرسمي

**الـ paths دي موجودة جوه شجرة الـ kernel مباشرةً:**

```
Documentation/driver-api/pin-control.rst
    - الـ reference الرسمي الكامل للـ pinctrl subsystem
    - يشرح pinmux / pinconf / GPIO ranges
    - يوضح كيفية كتابة pinctrl driver جديد

Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl-a1.yaml
    - الـ DT binding الرسمي للـ Meson A1 periphs pinctrl
    - يحدد الـ compatible string: "amlogic,meson-a1-periphs-pinctrl"

Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl-common.yaml
    - الـ binding المشترك بين كل SoCs الـ Amlogic

Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl-g12a-periphs.yaml
    - مثال binding لجيل G12A — بيوضح التطور من G12A للـ A1

Documentation/gpio/
    - توثيق GPIO subsystem اللي بيتعامل مع pinctrl
```

---

### Commits المهمة في الـ Kernel

**الـ commits دي أضافت وطورت الـ driver:**

| الـ Patch/Commit | الوصف |
|-----------------|--------|
| [v3 patch: add compatible for Meson A1 pin controller](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1571050492-6598-2-git-send-email-qianggui.song@amlogic.com/) | إضافة الـ compatible string الأولية للـ A1 — المرسل: Qianggui Song من Amlogic |
| [v5: arm64: dts: meson: a1: add pinctrl controller support](https://patchwork.kernel.org/project/linux-amlogic/patch/1573203636-7436-4-git-send-email-qianggui.song@amlogic.com/) | إضافة node الـ pinctrl لـ device tree الـ A1 |
| [v6: arm64: dts: meson: a1: add pinctrl controller support](https://patchwork.kernel.org/project/linux-amlogic/patch/1573819429-6937-4-git-send-email-qianggui.song@amlogic.com/) | النسخة النهائية من الـ DT node بعد review |
| [pinctrl: meson: fix drive strength register and bit calculation](https://patchwork.kernel.org/project/linux-amlogic/patch/20200610041329.12948-1-hhk7734@gmail.com/) | إصلاح حساب رجستر الـ drive strength — مهم لفهم الـ `BANK_DS` macro |
| [v6: pinctrl: meson: add support for GPIO interrupts](https://patchwork.kernel.org/project/linux-amlogic/patch/a86d38af-9103-7557-9986-2dc187815569@gmail.com/) | إضافة دعم GPIO interrupts لـ Meson |

**للبحث عن commits مباشرةً في LKML:**

- [PATCH v2 1/3 - add compatible for Amlogic Meson A1 pin controller](https://lkml.iu.edu/hypermail/linux/kernel/1910.1/00326.html)
- [PATCH v3 0/3 - Amlogic Meson pinctrl driver (الأصلي)](https://lkml.iu.edu/hypermail/linux/kernel/1411.2/00173.html)

---

### نقاشات Mailing List

**linux-amlogic list** على patchwork هو المكان الأساسي لمتابعة تطوير الـ driver:

- الـ maintainer الحالي للـ Meson pinctrl: **Neil Armstrong** (BayLibre)
- المؤلف الأصلي للـ A1 driver: **Qianggui Song** (Amlogic)
- للمتابعة: `linux-amlogic@lists.infradead.org`

---

### eLinux.org

**الـ pages دي مفيدة لفهم pinctrl في السياق الأوسع:**

- [Introduction to pin muxing and GPIO control under Linux (PDF)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf)
  — شرح ELC 2021 كامل عن pinmux وGPIO control من elinux.org
- [Tests:i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl)
  — مثال عملي على استخدام pinctrl مع i2c-demux في device tree

---

### KernelNewbies.org

**الـ pages دي بتوثق التغييرات في كل kernel release للـ pinctrl subsystem:**

| الـ Kernel | الرابط |
|-----------|--------|
| Linux 6.8  | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) |
| Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| Linux 6.14 | [kernelnewbies.org/Linux_6.14](https://kernelnewbies.org/Linux_6.14) |
| Linux 6.17 | [kernelnewbies.org/Linux_6.17](https://kernelnewbies.org/Linux_6.17) |
| Linux Changes (عام) | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) |

ابحث في الـ pages دي عن كلمة `pinctrl` أو `Amlogic` عشان تلاقي التغييرات المتعلقة بالـ subsystem في كل release.

---

### كتب مقترحة

#### Linux Device Drivers (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول المهمة:**
  - Chapter 1: An Introduction to Device Drivers
  - Chapter 9: Communicating with Hardware — لفهم register I/O
  - Chapter 14: The Linux Device Model — لفهم `platform_driver` و `of_match_table`
- **ملاحظة:** الكتاب قديم (kernel 2.6) لكن المفاهيم الأساسية للـ platform driver لسه صالحة
- **تحميل مجاني:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development (Robert Love) — الطبعة الثالثة
- **الفصول المهمة:**
  - Chapter 17: Devices and Modules — فهم `MODULE_DEVICE_TABLE` و `platform_driver`
  - Chapter 18: Debugging the Linux Kernel
- **مفيد لـ:** فهم لماذا `module_platform_driver()` macro موجود وبيعمل إيه

#### Embedded Linux Primer (Christopher Hallinan) — الطبعة الثانية
- **الفصول المهمة:**
  - Chapter 15: Devices, Drivers, and the Kernel
  - فصل device tree bindings وكيفية ربط الـ driver بالـ DT
- **مفيد لـ:** فهم كيف يتم تكوين pinctrl في الـ device tree على embedded systems

#### Linux Device Driver Development (John Madieu) — الطبعة الثانية
- **الأحدث** من LDD3 ويغطي kernel 5.x+
- فصل كامل عن GPIO و pinctrl subsystem
- أمثلة عملية على كتابة pinctrl drivers

---

### مسارات بحث إضافية

**للبحث في الـ kernel source:**

```bash
# كل drivers الـ Meson pinctrl
ls drivers/pinctrl/meson/

# الـ headers المشتركة
cat drivers/pinctrl/meson/pinctrl-meson.h

# الـ AXG pmx ops المستخدمة في A1
cat drivers/pinctrl/meson/pinctrl-meson-axg-pmx.h

# DT bindings الـ Amlogic
ls Documentation/devicetree/bindings/pinctrl/amlogic*

# الـ GPIO DT headers
ls include/dt-bindings/gpio/meson*
```

**Search Terms للبحث عن معلومات أكثر:**

```
"amlogic meson a1 pinctrl"
"meson-a1-periphs-pinctrl"
"pinctrl-meson-axg-pmx"
"BANK_DS meson pinctrl"
"meson_pinctrl_probe"
"amlogic pinmux function groups"
"linux pinctrl driver tutorial"
"platform_driver pinctrl"
"of_device_id pinctrl meson"
```

**للبحث في git log:**

```bash
git log --oneline --all -- drivers/pinctrl/meson/pinctrl-meson-a1.c
git log --oneline --all -- drivers/pinctrl/meson/pinctrl-meson.h
git log --oneline --all -- drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c
```
## Phase 8: Writing simple module

### الفكرة

الـ `pinctrl-meson-a1.c` بيسجّل `platform_driver` اسمه `meson_a1_pinctrl_driver` اللي بيستخدم `meson_pinctrl_probe` كـ probe function. هنستخدم **kprobe** نعلّق على `meson_pinctrl_probe` عشان نطبع اسم الـ device اللي بيتعمله probe وعدد الـ pins المتاحة — ده معلومة مفيدة لأي حد بيـ debug تسجيل الـ pinctrl على SoC.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe module: hook meson_pinctrl_probe to log pinctrl registration
 * on Amlogic Meson A1 (and compatible) SoCs.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/platform_device.h>

/* meson_pinctrl_probe receives a platform_device as its only argument */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86_64  : first arg in rdi
     * On arm64   : first arg in x0
     * kprobes gives us regs — use the helper that maps to the ABI.
     * regs_get_kernel_argument(regs, 0) works on both arches.
     */
    struct platform_device *pdev =
        (struct platform_device *)regs_get_kernel_argument(regs, 0);

    if (!pdev)
        return 0;

    pr_info("[meson_pinctrl_kprobe] probe called for device: \"%s\"  id=%d\n",
            pdev->name, pdev->id);

    /* print the driver_override if set, useful when multiple compatible match */
    if (pdev->driver_override)
        pr_info("[meson_pinctrl_kprobe] driver_override = \"%s\"\n",
                pdev->driver_override);

    return 0; /* 0 = let the real function continue normally */
}

/* called after meson_pinctrl_probe returns — log the return value */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* regs_return_value() reads the return register (rax / x0) */
    long ret = regs_return_value(regs);

    pr_info("[meson_pinctrl_kprobe] probe returned: %ld (%s)\n",
            ret, ret == 0 ? "success" : "error");
}

/* define and initialise the kprobe struct */
static struct kprobe kp = {
    .symbol_name    = "meson_pinctrl_probe", /* symbol to hook            */
    .pre_handler    = handler_pre,           /* called before the function */
    .post_handler   = handler_post,          /* called after it returns    */
};

static int __init meson_pinctrl_probe_hook_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[meson_pinctrl_kprobe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[meson_pinctrl_kprobe] planted at %p (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

static void __exit meson_pinctrl_probe_hook_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[meson_pinctrl_kprobe] removed kprobe from %p\n", kp.addr);
}

module_init(meson_pinctrl_probe_hook_init);
module_exit(meson_pinctrl_probe_hook_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe hook on meson_pinctrl_probe to trace Meson A1 pinctrl registration");
```

---

### Makefile للـ out-of-tree build

```makefile
obj-m += meson_pinctrl_probe_hook.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

```c
#include <linux/kprobes.h>
#include <linux/platform_device.h>
```

الـ `kprobes.h` بيجيب تعريف `struct kprobe` و `register_kprobe` / `unregister_kprobe`.
الـ `platform_device.h` محتاجينه عشان نقدر نـ cast الـ argument الأول ونوصل لـ `pdev->name`.

---

#### الـ `handler_pre` — callback قبل التنفيذ

```c
struct platform_device *pdev =
    (struct platform_device *)regs_get_kernel_argument(regs, 0);
```

الـ `meson_pinctrl_probe` بياخد `struct platform_device *` كأول argument. الـ `regs_get_kernel_argument(regs, 0)` بيقرا الـ register المناسب حسب الـ ABI (rdi على x86_64, x0 على arm64) فالكود portable على الأرشيكتشرين.
بنطبع اسم الـ device وإيه اللي الـ driver override بتاعته عشان نفهم أي pinctrl bank بيتسجّل.

---

#### الـ `handler_post` — callback بعد الرجوع

```c
long ret = regs_return_value(regs);
```

الـ `regs_return_value()` بيقرا الـ return register بعد ما الدالة الأصلية خلصت، فنقدر نشوف هل الـ probe نجح (0) ولا فيه error. ده مفيد لأن الـ `meson_pinctrl_probe` بتعمل regmap setup وممكن تفشل.

---

#### الـ `struct kprobe kp`

```c
static struct kprobe kp = {
    .symbol_name = "meson_pinctrl_probe",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};
```

الـ kernel بيحوّل الـ `symbol_name` لعنوان فعلي وقت الـ `register_kprobe`. لازم الـ symbol يكون exported أو على الأقل موجود في الـ `System.map` — الـ `meson_pinctrl_probe` موجود في `pinctrl-meson.c` ومش `static` فممكن الـ kprobe يوصله.

---

#### الـ `module_init` / `module_exit`

الـ `register_kprobe` بيزرع software breakpoint (int3 على x86 أو BRK على arm64) على أول instruction في الدالة المستهدفة.
الـ `unregister_kprobe` في الـ exit ضروري عشان يشيل الـ breakpoint ويرجع الـ instruction الأصلية — لو ما اتعملش ده والـ module اتنزع، أي call لـ `meson_pinctrl_probe` هيـ crash الـ kernel.

---

### إزاي تجرّبه

```bash
# بناء الموديول
make KDIR=/lib/modules/$(uname -r)/build

# تحميله
sudo insmod meson_pinctrl_probe_hook.ko

# تشغيل probe يدوي (rebind الـ driver)
echo "meson-a1-pinctrl" > /sys/bus/platform/drivers/meson-a1-pinctrl/unbind
echo "meson-a1-pinctrl" > /sys/bus/platform/drivers/meson-a1-pinctrl/bind

# مشاهدة اللوج
dmesg | grep meson_pinctrl_kprobe

# تنظيف
sudo rmmod meson_pinctrl_probe_hook
```

**مثال مخرجات متوقعة:**

```
[meson_pinctrl_kprobe] planted at ffffffffc0123456 (meson_pinctrl_probe)
[meson_pinctrl_kprobe] probe called for device: "meson-a1-pinctrl"  id=-1
[meson_pinctrl_kprobe] probe returned: 0 (success)
[meson_pinctrl_kprobe] removed kprobe from ffffffffc0123456
```
