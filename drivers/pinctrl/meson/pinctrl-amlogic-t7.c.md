## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

الملف ده جزء من **ARM/Amlogic Meson SoC support** subsystem — تحديداً الـ **pinctrl/meson** driver layer اللي بيتكلم عنه في MAINTAINERS تحت:

```
ARM/Amlogic Meson SoC support
F:  drivers/pinctrl/meson/
```

المسؤولون عنه: Neil Armstrong و Kevin Hilman من Baylibre، والـ mailing list هو `linux-amlogic@lists.infradead.org` و `linux-gpio@vger.kernel.org`.

---

### الـ SoC المستهدف: Amlogic T7

الـ **Amlogic T7** هو SoC من شركة Amlogic (شركة صينية متخصصة في media processors)، مستهدف بشكل أساسي في **set-top boxes** وأجهزة **smart TV** وأنظمة **multimedia embedded**. الـ T7 نسخة متقدمة من عائلة Meson، بيضم ARM64 cores مع IP blocks لـ HDMI RX/TX و Ethernet و PCIe و eMMC وغيرها.

---

### الصورة الكبيرة: ليه محتاجين pinctrl أصلاً؟

#### القصة بالبساطة

تخيّل عندك لوحة طباعة (PCB) فيها شريحة SoC كبيرة زي الـ T7. الشريحة دي عندها مئات الـ **physical pins** — كل pin ممكن يشتغل بأكتر من طريقة:

- نفس الـ pin ممكن يكون **GPIO** عادي (input/output رقمي).
- أو ممكن يكون **UART TX** لبروتوكول serial.
- أو ممكن يكون **SPI MOSI** لبروتوكول SPI.
- أو ممكن يكون **I2C SDA** لبروتوكول I2C.

ده اللي بيسموه **pin multiplexing** أو اختصاراً **pinmux** — نفس الـ wire الجسدي بيقدر يلعب أكتر من دور، بس دور واحد بس في الوقت الواحد.

**المشكلة:** مين اللي بيقرر كل pin هيعمل إيه؟ ومين اللي بيكتب في registers الـ hardware عشان يفعّل الوظيفة الصح؟

**الحل:** الـ **pinctrl subsystem** في Linux kernel — ده framework موحّد بيدي كل driver القدرة إنه يطلب "أنا محتاج UART على الـ pins دي" وتبقى من مسؤولية الـ pinctrl driver إنه يكتب في الـ registers الصح.

---

### دور الملف `pinctrl-amlogic-t7.c` تحديداً

الـ **pinctrl subsystem** في Linux عنده مكونين رئيسيين:

1. **Core framework** (`pinctrl-meson.c`) — الكود العام المشترك بين كل Meson SoCs.
2. **Per-SoC data file** (`pinctrl-amlogic-t7.c`) — ده اللي بنتكلم عنه — مجرد **tables of data** بتصف الـ hardware بالتفصيل للـ T7 تحديداً.

الملف ده **مش بيعمل logic** — هو بيوصف فقط:

| ما يصفه | مثال |
|---|---|
| كل الـ pins الموجودة | `GPIOB_0` إلى `GPIO_TEST_N` — إجمالي ~157 pin |
| الـ **banks** (مجموعات pins) | Bank B, C, X, W, D, E, Z, T, M, Y, H, TEST_N |
| الـ **register addresses** لكل bank | pull-enable, pull, dir, out, in, drive-strength |
| الـ **pin groups** | مثلاً `emmc_nand_d0` هو GPIOB_0 بيمثّل data line 0 لـ eMMC |
| الـ **functions** (الوظائف الممكنة) | `emmc`, `sdcard`, `uart_c`, `spi0`, `hdmirx_a`... إلخ |
| الـ **function number** لكل group | رقم من 1 إلى 4 بيتكتب في registers الـ mux |

---

### قصة عملية: كيف يشتغل الـ eMMC على T7؟

```
المرحلة الأولى: Device Tree
──────────────────────────────
المطوّر بيكتب في .dts:
    &mmc0 {
        pinctrl-names = "default";
        pinctrl-0 = <&emmc_pins>;
    };
    emmc_pins: emmc-pins {
        mux {
            groups = "emmc_nand_d0", "emmc_clk", "emmc_cmd", ...;
            function = "emmc";
        };
    };

المرحلة الثانية: Kernel Boot
──────────────────────────────
1. mmc driver بيطلب من pinctrl subsystem: "فعّل الـ emmc function"
2. pinctrl core بيسأل: إيه الـ groups اللي في الـ emmc function؟
3. بيلاقي في t7_periphs_functions[]:
       FUNCTION(emmc)  →  emmc_groups[]  →  ["emmc_nand_d0", ..., "emmc_cmd"]
4. كل group فيها الـ pin اللي هيتأثر + الـ func number (=1)
5. pinctrl-meson-axg-pmx.c بيكتب في الـ mux register:
       Bank B, reg 0x00, func = 1  (eMMC function)
6. الـ GPIOB_0..11 دلوقتي بيشتغلوا كـ eMMC data/clk/cmd
```

---

### البنية الداخلية للملف

```
pinctrl-amlogic-t7.c
│
├── t7_periphs_pins[]          ← قائمة كل الـ pins (pinctrl_pin_desc)
│     MESON_PIN(GPIOB_0) .. MESON_PIN(GPIO_TEST_N)
│
├── *_pins[] arrays            ← لكل function group: أرقام الـ pins
│     emmc_nand_d0_pins[] = {GPIOB_0}
│     sdcard_d0_pins[]    = {GPIOC_0}
│     ...
│
├── t7_periphs_groups[]        ← meson_pmx_group: اسم + pins + func number
│     GPIO_GROUP(GPIOB_0)      ← GPIO mode (func=0)
│     GROUP(emmc_nand_d0, 1)   ← eMMC mode (func=1)
│     GROUP(nor_hold, 2)       ← NOR flash mode (func=2)
│     ...
│
├── *_groups[] string arrays   ← لكل function: أسماء الـ groups اللي بتكوّنه
│     emmc_groups[] = {"emmc_nand_d0", ..., "emmc_cmd"}
│
├── t7_periphs_functions[]     ← meson_pmx_func: ربط function باسمه وgroups
│     FUNCTION(emmc), FUNCTION(sdcard), FUNCTION(uart_c), ...
│
├── t7_periphs_banks[]         ← meson_bank: عناوين registers لكل bank
│     BANK_DS("B", GPIOB_0, GPIOB_12, irq=0..12,
│             pullen=0x2b, pull=0x2c, dir=0x2a,
│             out=0x29, in=0x28, ds=0x2f)
│
├── t7_periphs_pmx_banks[]     ← meson_pmx_bank: عناوين mux registers
│     BANK_PMX("B", GPIOB_0, GPIOB_12, reg=0x00, offset=0)
│
└── t7_periphs_pinctrl_data    ← الـ meson_pinctrl_data الرئيسي
      ← بيجمع كل حاجة فوق في struct واحد
      ← بيستخدم meson_axg_pmx_ops للـ pinmux operations
      ← بيستخدم meson_a1_parse_dt_extra للـ device tree parsing
```

---

### الوظائف (Functions) المدعومة في T7

الـ T7 من أغنى الـ SoCs في عدد الـ functions نظراً لاستهدافه لـ multimedia applications:

| Category | Functions |
|---|---|
| Storage | `emmc`, `nor`, `sdcard`, `sdio` |
| Serial | `uart_c`, `uart_d`, `uart_e`, `uart_f`, `uart_ao_a`, `uart_ao_b` |
| SPI | `spi0`, `spi1`, `spi2`, `spi3`, `spi4`, `spi5` |
| I2C | `i2c0`, `i2c1`, `i2c2`, `i2c3`, `i2c4`, `i2c5`, `i2c0_ao`, `i2c1_ao` |
| PWM | `pwm_a..f`, `pwm_ao_a..h`, `pwm_vs` |
| Audio | `tdm`, `mclk`, `pdm`, `spdif_in`, `spdif_out` |
| Video/Display | `hdmirx_a/b/c`, `hdmitx`, `hsync`, `vsync`, `sync_3d`, `vx1_a/b`, `edp_a/b` |
| Network | `eth` (RGMII Gigabit Ethernet) |
| Debug | `jtag_a`, `jtag_b` |
| Transport Stream | `tsin_a`, `tsin_b`, `tsin_c`, `tsin_d` (DVB/TV tuner input) |
| Misc | `pcieck`, `rtc_clk`, `remote_in/out`, `iso7816`, `clk12_24`, `clk25m`, `mic_mute` |

---

### الـ Banks في T7

الـ T7 عنده **12 bank** لتنظيم الـ pins:

```
Bank B  → GPIOB_0  .. GPIOB_12   (13 pins) → eMMC / NOR Flash
Bank C  → GPIOC_0  .. GPIOC_6    (7 pins)  → SD Card / JTAG-B / SPI1
Bank X  → GPIOX_0  .. GPIOX_19   (20 pins) → SDIO / UART-C / I2C2 / PWM / TDM
Bank W  → GPIOW_0  .. GPIOW_16   (17 pins) → HDMI RX (3 ports) / HDMI TX / CEC
Bank D  → GPIOD_0  .. GPIOD_12   (13 pins) → UART-AO / I2C-AO / JTAG-A / Remote
Bank E  → GPIOE_0  .. GPIOE_6    (7 pins)  → PWM-AO / I2C-AO / RTC / CLK
Bank Z  → GPIOZ_0  .. GPIOZ_13   (14 pins) → Ethernet RGMII / TSin-B / SPI4/5
Bank T  → GPIOT_0  .. GPIOT_23   (24 pins) → TDM / MCLK / SPI0/3 / I2C0/2 / SPDIF
Bank M  → GPIOM_0  .. GPIOM_13   (14 pins) → TDM / SPI1 / I2C2/3 / UART-D / PDM
Bank Y  → GPIOY_0  .. GPIOY_18   (19 pins) → SPI2 / UART-D/E / I2C4/5 / TSin-C/D / Video
Bank H  → GPIOH_0  .. GPIOH_7    (8 pins)  → I2C3/4 / UART-F / Ethernet LEDs
Bank TEST_N → GPIO_TEST_N         (1 pin)   → Test pin
```

كل bank عنده **6 register sets** (كل set عبارة عن offset + bit في الـ regmap):
- `PULLEN`: تفعيل pull resistor
- `PULL`: اتجاه الـ pull (up أو down)
- `DIR`: input أو output
- `OUT`: قيمة الـ output
- `IN`: قراءة الـ input
- `DS`: drive strength (500µA / 2500µA / 3000µA / 4000µA)

---

### الفرق بين T7 وأجيال Meson القديمة

الـ T7 بيستخدم الـ **AXG-style pinmux** (`meson-axg-pmx`) مش الـ Meson8 القديم — الفرق الجوهري:

- **Meson8 style**: كل pin عنده multiple mux bits مبعثرة
- **AXG style**: كل bank عنده register واحد و4 bits لكل pin بيحددوا الـ function number (0=GPIO, 1=func1, 2=func2, 3=func3, 4=func4)

ده بيخلي الكود أبسط وأوضح.

---

### الملفات المرتبطة اللي لازم تعرفها

#### Core Layer (مشترك بين كل Meson SoCs)

| الملف | الدور |
|---|---|
| `drivers/pinctrl/meson/pinctrl-meson.c` | الكود الأساسي: probe, GPIO ops, pin config |
| `drivers/pinctrl/meson/pinctrl-meson.h` | التعريفات: `meson_bank`, `meson_pinctrl_data`, `meson_pmx_group` |
| `drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c` | تنفيذ `meson_axg_pmx_ops` — كتابة الـ mux registers |
| `drivers/pinctrl/meson/pinctrl-meson-axg-pmx.h` | `meson_pmx_bank`, `BANK_PMX`, `GROUP`, `GPIO_GROUP` macros |

#### Per-SoC Data Files (نفس النمط لكل SoC)

| الملف | الـ SoC |
|---|---|
| `pinctrl-amlogic-t7.c` | **T7** (ملفنا) — multimedia SoC |
| `pinctrl-amlogic-a4.c` | A4 — نسخة أحدث |
| `pinctrl-amlogic-c3.c` | C3 |
| `pinctrl-meson-s4.c` | S4 |
| `pinctrl-meson-g12a.c` | G12A — Amlogic S905X3 |
| `pinctrl-meson-axg.c` | AXG — أول جيل بـ AXG pmx |
| `pinctrl-meson-a1.c` | A1 — low-power variant |
| `pinctrl-meson8.c` / `meson8b.c` | أقدم الأجيال (32-bit) |

#### DT Bindings Header

| الملف | الدور |
|---|---|
| `include/dt-bindings/gpio/amlogic,t7-periphs-pinctrl.h` | تعريف أرقام الـ GPIO للاستخدام في Device Tree (`GPIOB_0 = 0`, ...) |

#### Device Tree Binding Documentation

| الملف | الدور |
|---|---|
| `Documentation/devicetree/bindings/pinctrl/amlogic,pinctrl-a4.yaml` | مثال على schema الـ DT binding لنفس النمط |
## Phase 2: شرح الـ pinctrl Framework

### المشكلة — ليه الـ pinctrl موجود أصلاً؟

أي SoC حديث عنده مئات الـ **pins** الجسدية. كل pin ممكن يشتغل في أكتر من دور في نفس الوقت — GPIO عادي، أو طرف I2C، أو خط SPI، أو إشارة UART، إلخ. الـ hardware بيحدد الأدوار دي عن طريق **mux registers** — كل مجموعة bits في register بتحدد "هذا الـ pin يعمل function رقم كذا".

المشكلة هي:
- كل driver (مثلاً UART driver) بيحتاج يعرف: "أي pin بيبعت بياناتي؟"
- لو كل driver قرأ/كتب في الـ mux registers بشكل مستقل → تعارض وفوضى.
- لو الـ bootloader سيّب الـ pins على حال → مش مضمون إن الـ kernel يلاقيها صح.
- بالإضافة إن كل pin ليه إعدادات تانية: pull-up/down، drive strength، direction.

**الـ pinctrl subsystem** هو الجهة الوحيدة اللي تتكلم مع الـ pin mux hardware — باقي الـ drivers بيطلبوا منه.

---

### الحل — المدخل اللي اختاره الـ kernel

الـ kernel بيجرد العملية في ثلاث طبقات:

| الطبقة | المسمى | المعنى |
|--------|--------|--------|
| طبقة واصف الـ pins | `pinctrl_pin_desc` | قائمة بكل pin وله رقم واسم |
| طبقة الـ mux | groups + functions | مجموعات pins ممكن تشتغل سوا لوظيفة معينة |
| طبقة الـ config | pin configurations | pull، drive strength، direction |

الـ subsystem بيعرض **API موحدة** للـ drivers والـ Device Tree، وبيفوّض تنفيذ العمليات الفعلية للـ SoC-specific driver اللي بيعرف الـ registers.

---

### تشبيه من الواقع — لوحة الكهرباء في المبنى

تخيل مبنى به 200 مقبس كهربائي. كل مقبس ممكن يتوصل بـ: شبكة كهرباء عادية، UPS، generator، أو تتركه مقطوع. في المبنى في **غرفة كهرباء مركزية** فيها لوحات توزيع.

- **الـ pins** = المقابس الجسدية
- **الـ mux register** = الكابلات اللي جوّا الحائط + switches اللي بتحدد المقبس ده موصول بإيه
- **الـ bank** = طابق كامل في المبنى — كل طابق عنده لوحة توزيع منفصلة (register base منفصل)
- **الـ function** = "شبكة الـ UART" مثلاً — زي شبكة كهرباء لها مقابس معينة بس
- **الـ group** = مجموعة مقابس محددة تخص شبكة واحدة (UART_TX, UART_RX)
- **الـ pinctrl driver** = الكهربائي المسؤول الوحيد عن لوحة التوزيع — مش مسموح لأي حد تاني يلمسها
- **الـ consumer driver** (UART driver مثلاً) = ساكن في شقة — بيطلب من الكهربائي "وصّل مقابسي بشبكة الـ generator"، مش بيعمل الشغل بنفسه

بالتفصيل:
- لما الـ UART driver يعمل `probe`، بيطلب الـ **pinctrl state** اللي اسمه "default" — زي الساكن اللي بيطلب تشغيل الكهرباء عند دخوله الشقة
- الـ kernel بيبحث في الـ DT عن الـ pin configuration المرتبطة بالـ node ده
- الـ pinctrl subsystem بيترجم الطلب ده لـ: "اكتب قيمة كذا في bit كذا من register كذا"
- لو حد تاني طلب نفس الـ pin لـ function تانية — الـ pinctrl بيرفض ويرجع error (تعارض)

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                         User Space / DT                             │
  │   pinctrl-0 = <&i2c2_x_pins>;  /* Device Tree node */              │
  └───────────────────────────┬─────────────────────────────────────────┘
                              │  DT parsing at probe time
  ┌───────────────────────────▼─────────────────────────────────────────┐
  │                  Linux pinctrl Core (drivers/pinctrl/core.c)         │
  │                                                                      │
  │  pinctrl_get() → pinctrl_lookup_state() → pinctrl_select_state()    │
  │                                                                      │
  │  Owns: pin ownership tracking, conflict detection, state machines    │
  └──────────┬─────────────────────────┬───────────────────────────────┘
             │ pinmux_ops              │ pinconf_ops
  ┌──────────▼──────────┐   ┌─────────▼──────────────────────────────┐
  │   pinmux layer      │   │         pinconf layer                   │
  │  (mux selection)    │   │  (pull, drive-strength, direction)      │
  └──────────┬──────────┘   └─────────┬──────────────────────────────┘
             │                        │
  ┌──────────▼────────────────────────▼──────────────────────────────┐
  │         Meson/Amlogic T7 pinctrl driver                           │
  │         pinctrl-amlogic-t7.c  +  pinctrl-meson.c                 │
  │                                                                    │
  │  meson_axg_pmx_ops  (pinmux_ops implementation)                   │
  │  meson_pinconf_ops  (pinconf_ops implementation)                  │
  │                                                                    │
  │  Data tables:                                                      │
  │    t7_periphs_pins[]      → pinctrl_pin_desc array                │
  │    t7_periphs_groups[]    → meson_pmx_group array                 │
  │    t7_periphs_functions[] → meson_pmx_func array                  │
  │    t7_periphs_banks[]     → meson_bank array (gpio config regs)   │
  │    t7_periphs_pmx_banks[] → meson_pmx_bank array (mux regs)      │
  └──────────┬────────────────────────────────────────────────────────┘
             │ regmap read/write
  ┌──────────▼────────────────────────────────────────────────────────┐
  │              Hardware Registers (MMIO via regmap)                  │
  │                                                                    │
  │  MUX registers:  GPIOB_MUX @ 0x00, GPIOX_MUX @ 0x02, ...        │
  │  Pull-enable:    GPIOB_PULLEN @ 0x2b, ...                         │
  │  Pull:           GPIOB_PULL @ 0x2c, ...                           │
  │  Direction:      GPIOB_DIR @ 0x2a, ...                            │
  │  Output:         GPIOB_OUT @ 0x29, ...                            │
  │  Input:          GPIOB_IN @ 0x28, ...                             │
  │  Drive Strength: GPIOB_DS @ 0x2f, ...                             │
  └───────────────────────────────────────────────────────────────────┘

  ┌───────────────────────────────────────────────────────────────────┐
  │   Consumer drivers (يطلبوا pins عبر DT + pinctrl API)            │
  │                                                                    │
  │   sdhci (EMMC)  →  GPIOB_0..7, GPIOB_8, GPIOB_10, GPIOB_11      │
  │   spi0          →  GPIOT_18..21                                    │
  │   i2c2          →  GPIOX_17..18  OR  GPIOT_22..23  OR GPIOM_12..13│
  │   eth (RGMII)   →  GPIOZ_0..13                                    │
  │   hdmirx        →  GPIOW_0..11                                    │
  └───────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الفكرة المحورية هي **فصل الـ "ماذا تريد" عن "كيف تفعله"**:

- الـ consumer driver بيقول: "أنا عايز function اسمها `i2c2` على الـ group `i2c2_sda_x` و `i2c2_sck_x`"
- الـ core بيتحقق من التعارضات ويتأكد إن مفيش driver تاني ماسك نفس الـ pins
- الـ T7 driver بيعرف: الـ `i2c2_sda_x` معناها GPIOX_17 → روح register 0x02 offset معين، اكتب function number 1

الـ **مفتاح** هو إن كل pin مش بيتعامل معاه مباشرة — الـ core بيتعامل مع **groups** (مجموعات pins) و**functions** (وظائف). الـ T7 SoC مثلاً عنده pin واحد (GPIOX_14) ممكن يكون:
- `uart_c_cts` (function 1 على bank X)
- `clk12_24_x` (function 2 على bank X)

الـ driver بيخزّن ده في جدول static وبيرجعه للـ core لما يطلبه.

---

### الـ Structs وعلاقتها ببعض

```
meson_pinctrl_data           ← النقطة المحورية للـ driver، بتجمع كل الجداول
│
├─► pinctrl_pin_desc[]       ← قائمة كل الـ pins (رقم + اسم)
│       MESON_PIN(GPIOB_0) → { .number=GPIOB_0, .name="GPIOB_0" }
│
├─► meson_pmx_group[]        ← كل group فيها: اسم، pins[] ، function number
│       GROUP(emmc_clk, 1)  ← GPIOB_8 شغّل function 1 في bank B
│       GPIO_GROUP(GPIOB_0) ← function 0 = GPIO mode
│
├─► meson_pmx_func[]         ← كل function: اسم + قائمة أسماء groups
│       FUNCTION(emmc)  →  emmc_groups[] = {"emmc_nand_d0", ..., "emmc_clk", ...}
│
├─► meson_bank[]             ← per-bank register descriptors للـ GPIO config
│   ┌─────────────────────────────────────────────────────────────┐
│   │  BANK_DS("B", GPIOB_0, GPIOB_12, ...)                      │
│   │    .regs[MESON_REG_PULLEN] = { 0x2b, 0 }  ← reg + bit_offset│
│   │    .regs[MESON_REG_PULL]   = { 0x2c, 0 }                   │
│   │    .regs[MESON_REG_DIR]    = { 0x2a, 0 }                   │
│   │    .regs[MESON_REG_OUT]    = { 0x29, 0 }                   │
│   │    .regs[MESON_REG_IN]     = { 0x28, 0 }                   │
│   │    .regs[MESON_REG_DS]     = { 0x2f, 0 }                   │
│   └─────────────────────────────────────────────────────────────┘
│
└─► meson_axg_pmx_data       ← بيخزّن meson_pmx_bank[] للـ mux registers
        meson_pmx_bank{"B", GPIOB_0, GPIOB_12, reg=0x00, offset=0}
        مفهوم: لكل pin في bank B، الـ mux bits موجودة في reg 0x00
```

```
meson_pinctrl              ← الـ runtime state object (per-device)
│
├─ pinctrl_dev *pcdev      ← مسجّل مع الـ pinctrl core
├─ pinctrl_desc desc       ← بتحتوي pinctrl_ops + pinmux_ops + pinconf_ops
├─ regmap *reg_mux         ← للـ mux registers
├─ regmap *reg_pullen      ← للـ pull-enable registers
├─ regmap *reg_pull        ← للـ pull direction registers
├─ regmap *reg_gpio        ← للـ direction/in/out registers
├─ regmap *reg_ds          ← للـ drive strength registers
└─ gpio_chip chip          ← بيسجّل نفسه كـ GPIO controller برضو
```

**ملاحظة مهمة:** الـ `meson_pinctrl` بيسجّل نفسه كـ GPIO controller (`gpio_chip`) في نفس الوقت. ده يعني الـ GPIO subsystem والـ pinctrl subsystem بيتشاركوا نفس الـ hardware.

---

### الـ Pin Multiplexing — آلية التنفيذ

لما الـ `meson_axg_pmx_ops.set_mux` بتتنادى:

```c
/* لكل pin في الـ group المطلوبة: */
/* 1. ابحث عن meson_pmx_bank اللي بتغطي هذا الـ pin */
/* 2. احسب bit offset: (pin - bank.first) * 4 + bank.offset */
/* 3. اكتب الـ func number في الـ 4 bits دي */
regmap_update_bits(pc->reg_mux, bank->reg, mask, func << bit);
```

كل pin بيأخد **4 bits** في الـ mux register. الـ func=0 معناها GPIO mode. الـ func=1..7 معناها alternate functions.

مثال ملموس:
```
GPIOC_0 → Bank C → BANK_PMX("C", GPIOC_0, GPIOC_6, reg=0x07, offset=0)
pin index في الـ bank = GPIOC_0 - GPIOC_0 = 0
bit position = 0 * 4 + 0 = bit 0..3 في register 0x07

GROUP(jtag_b_tdo, 2) → GPIOC_0 مع func=2
→ اكتب 0x2 في bits [3:0] من register 0x07
```

---

### الـ Banks — الوحدة الأساسية للتنظيم

الـ T7 SoC عنده **12 bank**:

| Bank | الـ Pins | الـ IRQ Range | الاستخدام الأساسي |
|------|---------|--------------|-----------------|
| B | GPIOB_0..12 | 0..12 | eMMC / NAND / NOR |
| C | GPIOC_0..6 | 13..19 | SD Card |
| X | GPIOX_0..19 | 20..39 | SDIO, UART, I2C, PWM |
| W | GPIOW_0..16 | 40..56 | HDMI RX/TX |
| D | GPIOD_0..12 | 57..69 | AO domain: UART, I2C, JTAG |
| E | GPIOE_0..6 | 70..76 | PWM AO |
| Z | GPIOZ_0..13 | 77..90 | Ethernet RGMII |
| T | GPIOT_0..23 | 91..114 | TDM, SPI, I2C |
| M | GPIOM_0..13 | 115..128 | TDM, PDM, SPI |
| Y | GPIOY_0..18 | 129..147 | SPI, UART, Display |
| H | GPIOH_0..7 | 148..155 | I2C, Ethernet LEDs |
| TEST_N | 1 pin | 156 | Test mode |

كل bank عنده **مجموعة registers منفصلة** — مش register مشترك. ده يخلي الكود أبسط: لما عايز تعرف الـ pull-enable bit لـ GPIOZ_5 مثلاً:
1. ابحث عن الـ bank اللي بتحتوي GPIOZ_5 → bank "Z" (first=GPIOZ_0)
2. الـ `regs[MESON_REG_PULLEN]` = { 0x13, 0 }
3. bit index = GPIOZ_5 - GPIOZ_0 = 5
4. عمل `regmap_update_bits(pc->reg_pullen, 0x13, BIT(5), value)`

---

### الـ Drive Strength — قيم التيار

الـ T7 بيدعم **4 مستويات** لكل pin:

```c
enum meson_pinconf_drv {
    MESON_PINCONF_DRV_500UA,   /* 0.5 mA  - للإشارات الحساسة */
    MESON_PINCONF_DRV_2500UA,  /* 2.5 mA  - الاستخدام العام */
    MESON_PINCONF_DRV_3000UA,  /* 3.0 mA  - للـ high-speed */
    MESON_PINCONF_DRV_4000UA,  /* 4.0 mA  - للأحمال الكبيرة */
};
```

الـ DS registers مشابهة في التنظيم للـ pull registers — كل bank عنده DS register منفصل (`MESON_REG_DS`).

---

### الـ GPIO Integration — subsystem داخل subsystem

**الـ GPIO subsystem:** يحتاج تعريف مسبق — هو الـ layer اللي بيخلي الـ userspace وبقية الـ drivers يتحكموا في individual pins كـ input/output. الـ kernel بيمثّله بـ `gpio_chip` struct.

الـ `meson_pinctrl` بيعمل `gpiochip_add_data` اللي بيسجّل الـ pins كـ GPIO lines. الـ driver بيوفر:
- `.get()` → قراءة `MESON_REG_IN`
- `.set()` → كتابة `MESON_REG_OUT`
- `.direction_input()` → كتابة 1 في `MESON_REG_DIR`
- `.direction_output()` → كتابة 0 في `MESON_REG_DIR` + ضبط القيمة

وبيوفر برضو GPIO interrupts عبر الـ `irq_first`/`irq_last` fields في `meson_bank`.

---

### ما يملكه الـ subsystem وما يفوّضه للـ driver

```
┌────────────────────────────────────────────────────────────────────┐
│              pinctrl CORE يملك                                     │
├────────────────────────────────────────────────────────────────────┤
│  • Pin ownership tracking (مين ماسك أي pin)                       │
│  • Conflict detection (منع تعارض drivers)                          │
│  • State machine (default, sleep, idle states)                     │
│  • DT/ACPI parsing لـ pin configurations                           │
│  • sysfs/debugfs interface                                         │
│  • التنسيق مع الـ GPIO subsystem                                   │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│              T7 Driver يملك (تفوّض من الـ core)                   │
├────────────────────────────────────────────────────────────────────┤
│  • قراءة/كتابة الـ mux registers الفعلية                          │
│  • حساب bit offsets لكل pin في كل bank                             │
│  • تطبيق pull-up/pull-down/drive-strength                          │
│  • إدارة الـ regmaps (reg_mux, reg_gpio, reg_pull, ...)            │
│  • تعريف الجداول الثابتة (pins, groups, functions, banks)          │
│  • DT extra parsing (meson_a1_parse_dt_extra)                      │
└────────────────────────────────────────────────────────────────────┘
```

---

### الـ Device Tree Integration

الـ driver بيتعرّف في الـ DT عبر:
```
compatible = "amlogic,t7-periphs-pinctrl";
```

الـ probe function الوحيدة هي `meson_pinctrl_probe` من `pinctrl-meson.c` — الـ T7 driver ما عندوش probe خاص. الـ `t7_periphs_pinctrl_data` بتربط كل الجداول مع بعض وبتُمرَّر عبر `of_device_id.data`.

```c
/* ربط الـ compatible string بالـ data tables */
static const struct of_device_id t7_pinctrl_dt_match[] = {
    {
        .compatible = "amlogic,t7-periphs-pinctrl",
        .data = &t7_periphs_pinctrl_data,  /* كل الجداول هنا */
    },
    { }
};
```

الـ `parse_dt = &meson_a1_parse_dt_extra` بتعالج حالة خاصة موجودة في T7 وA1 — وهي إن الـ regmaps (mux, pull, gpio, ds) ممكن تيجي من أكتر من `reg` property في الـ DT node.

---

### الـ Function Multiplicity — نفس الوظيفة على banks مختلفة

نقطة مهمة في تصميم T7: بعض الـ functions ممكن تتفعّل على pins في banks مختلفة. مثلاً:

```c
/* الـ i2c2 ممكن تكون على X أو T أو M */
static const char * const i2c2_groups[] = {
    "i2c2_sda_x", "i2c2_sck_x",   /* Bank X */
    "i2c2_sda_t", "i2c2_sck_t",   /* Bank T */
    "i2c2_sda_m", "i2c2_sck_m",   /* Bank M */
};
```

الـ DT node بيحدد أي group بالظبط بيستخدم. الـ pinctrl core بيمرر الـ group name للـ driver اللي بيحدد منها الـ pins والـ register.

ده بيدي المصمم مرونة في الـ PCB layout — يقدر يوصّل الـ I2C2 لأي من الـ 3 مجموعات حسب ما يريح التوجيه على الـ board.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Flags المهمة

#### `enum meson_reg_type` — أنواع الـ registers

| القيمة | الاسم | الوظيفة |
|--------|-------|---------|
| 0 | `MESON_REG_PULLEN` | تفعيل أو تعطيل pull resistor |
| 1 | `MESON_REG_PULL` | اتجاه الـ pull (up/down) |
| 2 | `MESON_REG_DIR` | اتجاه الـ pin (input/output) |
| 3 | `MESON_REG_OUT` | قيمة الـ output (0/1) |
| 4 | `MESON_REG_IN` | قراءة قيمة الـ input |
| 5 | `MESON_REG_DS` | Drive Strength |
| 6 | `MESON_NUM_REG` | عدد الـ registers (sentinel) |

**الـ stride** لكل نوع: معظمها 1 بت لكل pin — `MESON_REG_DS` بيستخدم 2 بت.

```c
static const unsigned int meson_bit_strides[] = {
    1, 1, 1, 1, 1, 2, 1
    /* PULLEN PULL DIR OUT IN DS NUM */
};
```

---

#### `enum meson_pinconf_drv` — قيم Drive Strength

| القيمة | الاسم | التيار |
|--------|-------|--------|
| 0 | `MESON_PINCONF_DRV_500UA` | 500 µA |
| 1 | `MESON_PINCONF_DRV_2500UA` | 2500 µA |
| 2 | `MESON_PINCONF_DRV_3000UA` | 3000 µA |
| 3 | `MESON_PINCONF_DRV_4000UA` | 4000 µA |

---

#### Config Options — GPIO Banks في T7

| اسم الـ Bank | أول Pin | آخر Pin | عدد الـ Pins | مثال وظيفة |
|-------------|---------|---------|-------------|-----------|
| B | GPIOB_0 | GPIOB_12 | 13 | eMMC, NAND, NOR |
| C | GPIOC_0 | GPIOC_6 | 7 | SD Card, JTAG_B, SPI1 |
| X | GPIOX_0 | GPIOX_19 | 20 | SDIO, UART_C, TDM, I2C2 |
| W | GPIOW_0 | GPIOW_16 | 17 | HDMI RX/TX A/B/C, CEC |
| D | GPIOD_0 | GPIOD_12 | 13 | UART_AO_A, JTAG_A, PWM_AO |
| E | GPIOE_0 | GPIOE_6 | 7 | PWM_AO A–G, I2C_AO, RTC |
| Z | GPIOZ_0 | GPIOZ_13 | 14 | Ethernet RGMII, TSIN_B, SPI4/5 |
| T | GPIOT_0 | GPIOT_23 | 24 | TDM, SPI0, I2C1, MCLK |
| M | GPIOM_0 | GPIOM_13 | 14 | TDM, SPI1, PDM, I2C2/3 |
| Y | GPIOY_0 | GPIOY_18 | 19 | SPI2, UART_D/E, TSIN_C/D, EDP |
| H | GPIOH_0 | GPIOH_7 | 8 | Ethernet LED, I2C3/4, UART_F |
| TEST_N | GPIO_TEST_N | GPIO_TEST_N | 1 | Test pin |

**مجموع الـ pins**: 169 pin في البنك الواحد (periphs).

---

### الـ Structs المهمة

---

#### `struct meson_reg_desc`

**الغرض**: بتوصف موقع بت واحد في الـ regmap — register offset + bit index.

```c
struct meson_reg_desc {
    unsigned int reg;  /* offset في الـ regmap (وحدة: كلمة 32-bit) */
    unsigned int bit;  /* رقم البت الأول لأول pin في الـ bank */
};
```

**بتُستخدم**: مصفوفة `regs[MESON_NUM_REG]` جوا `meson_bank`.

---

#### `struct meson_bank`

**الغرض**: بيوصف مجموعة pins متجاورة — بنك واحد من GPIO. بيحدد أي bits في أي registers بتتحكم في الـ pull / direction / output / input / drive-strength.

```c
struct meson_bank {
    const char *name;         /* اسم البنك: "B", "X", "Z", ... */
    unsigned int first;       /* رقم أول pin في البنك */
    unsigned int last;        /* رقم آخر pin في البنك */
    int irq_first;            /* أول hwirq في البنك */
    int irq_last;             /* آخر hwirq */
    struct meson_reg_desc regs[MESON_NUM_REG]; /* 6 descriptors */
};
```

**مثال عملي** — Bank B:
```c
BANK_DS("B", GPIOB_0, GPIOB_12, 0, 12,
    0x2b, 0,   /* PULLEN: reg=0x2b, bit=0 */
    0x2c, 0,   /* PULL:   reg=0x2c, bit=0 */
    0x2a, 0,   /* DIR:    reg=0x2a, bit=0 */
    0x29, 0,   /* OUT:    reg=0x29, bit=0 */
    0x28, 0,   /* IN:     reg=0x28, bit=0 */
    0x2f, 0);  /* DS:     reg=0x2f, bit=0 */
```

---

#### `struct meson_pmx_group`

**الغرض**: بيوصف "مجموعة pins" ممكن يتفعل فيها function معين. كل مجموعة ممكن تكون pin واحد (GPIO) أو أكتر (مثلاً eMMC data bus).

```c
struct meson_pmx_group {
    const char *name;          /* اسم الـ group: "emmc_clk", "GPIOB_0" */
    const unsigned int *pins;  /* مصفوفة أرقام الـ pins */
    unsigned int num_pins;     /* عدد الـ pins */
    const void *data;          /* → meson_pmx_axg_data (يحتوي func number) */
};
```

نوعين من الـ groups:
- **GPIO_GROUP**: pin واحد، `func=0` دايمًا (GPIO mode).
- **GROUP(name, func_num)**: pin واحد + رقم الـ function (1–4).

---

#### `struct meson_pmx_axg_data`

**الغرض**: بيحمل رقم الـ function اللي لازم يتكتب في الـ mux register — القيمة اللي بتتحدد 4-bits لكل pin.

```c
struct meson_pmx_axg_data {
    unsigned int func;  /* 0 = GPIO, 1..4 = alternate function */
};
```

---

#### `struct meson_pmx_func`

**الغرض**: بيوصف "وظيفة" كاملة زي `emmc` أو `uart_c` — بيجمع كل الـ groups اللي تخص الوظيفة دي.

```c
struct meson_pmx_func {
    const char *name;              /* اسم الوظيفة: "emmc", "spi0", ... */
    const char * const *groups;    /* مصفوفة أسماء الـ groups */
    unsigned int num_groups;       /* عدد الـ groups */
};
```

**مثال**:
```c
/* emmc_groups يحتوي: "emmc_nand_d0"..."emmc_nand_d7","emmc_clk","emmc_cmd","emmc_nand_ds" */
FUNCTION(emmc);
/* ينتج: { .name="emmc", .groups=emmc_groups, .num_groups=11 } */
```

---

#### `struct meson_pmx_bank`

**الغرض**: بيوصف أي register (في regmap الـ mux) بيتحكم في الـ pinmux لكل pin في البنك — كل pin بياخد 4 bits.

```c
struct meson_pmx_bank {
    const char *name;      /* اسم البنك */
    unsigned int first;    /* أول pin */
    unsigned int last;     /* آخر pin */
    unsigned int reg;      /* أول register في الـ mux regmap */
    unsigned int offset;   /* bit offset لأول pin */
};
```

**مثال** — Bank B:
```c
BANK_PMX("B", GPIOB_0, GPIOB_12, 0x00, 0);
/* pins GPIOB_0..12 → mux reg 0x00, 4 bits/pin من bit 0 */
```

---

#### `struct meson_axg_pmx_data`

**الغرض**: container بيجمع كل `meson_pmx_bank` الخاصة بالـ SoC — بيتمرر كـ `pmx_data` في `meson_pinctrl_data`.

```c
struct meson_axg_pmx_data {
    const struct meson_pmx_bank *pmx_banks;  /* مصفوفة الـ PMX banks */
    unsigned int num_pmx_banks;              /* عددها */
};
```

---

#### `struct meson_pinctrl_data`

**الغرض**: static descriptor للـ SoC — بيحتوي كل المعلومات الثابتة اللي بيحتاجها الـ driver. بيتشارك بين كل الـ instances من نفس الـ SoC.

```c
struct meson_pinctrl_data {
    const char *name;                      /* "periphs-banks" */
    const struct pinctrl_pin_desc *pins;   /* → t7_periphs_pins[] */
    const struct meson_pmx_group *groups;  /* → t7_periphs_groups[] */
    const struct meson_pmx_func *funcs;    /* → t7_periphs_functions[] */
    unsigned int num_pins;
    unsigned int num_groups;
    unsigned int num_funcs;
    const struct meson_bank *banks;        /* → t7_periphs_banks[] */
    unsigned int num_banks;
    const struct pinmux_ops *pmx_ops;      /* → &meson_axg_pmx_ops */
    const void *pmx_data;                  /* → &t7_periphs_pmx_banks_data */
    int (*parse_dt)(struct meson_pinctrl *pc); /* → meson_a1_parse_dt_extra */
};
```

---

#### `struct meson_pinctrl`

**الغرض**: الـ runtime instance — بيتعمل لكل device. بيحمل كل الـ regmaps والـ gpio_chip والـ pinctrl_dev.

```c
struct meson_pinctrl {
    struct device *dev;            /* الـ platform device */
    struct pinctrl_dev *pcdev;     /* معرف الـ pinctrl core */
    struct pinctrl_desc desc;      /* الـ descriptor المتمرر للـ core */
    struct meson_pinctrl_data *data; /* → البيانات الثابتة للـ SoC */
    struct regmap *reg_mux;        /* registers الـ pinmux */
    struct regmap *reg_pullen;     /* registers تفعيل الـ pull */
    struct regmap *reg_pull;       /* registers اتجاه الـ pull */
    struct regmap *reg_gpio;       /* registers الـ GPIO (dir/in/out) */
    struct regmap *reg_ds;         /* registers الـ Drive Strength */
    struct gpio_chip chip;         /* الـ GPIO chip للـ gpiolib */
    struct fwnode_handle *fwnode;  /* FW node من الـ DT */
};
```

**ملاحظة مهمة على T7**: في `meson_a1_parse_dt_extra`:
```c
pc->reg_pull    = pc->reg_gpio;  /* نفس الـ reg_gpio */
pc->reg_pullen  = pc->reg_gpio;
pc->reg_ds      = pc->reg_gpio;
```
يعني T7 بيستخدم register واحد للـ GPIO + pull + DS — تصميم مختلف عن الـ SoCs القديمة.

---

### رسم علاقات الـ Structs

```
                    ┌─────────────────────────────────────┐
                    │       meson_pinctrl_data (static)   │
                    │  .pins  → pinctrl_pin_desc[]        │
                    │  .groups→ meson_pmx_group[]         │
                    │  .funcs → meson_pmx_func[]          │
                    │  .banks → meson_bank[]              │
                    │  .pmx_ops→ pinmux_ops               │
                    │  .pmx_data→ meson_axg_pmx_data ─┐  │
                    │  .parse_dt()                    │  │
                    └────────────────┬────────────────┘  │
                                     │ (متمرر من of_match)│
                    ┌────────────────▼────────────────┐  │
                    │       meson_pinctrl (runtime)   │  │
                    │  .data  ──────────────────────► │  │
                    │  .pcdev ──► pinctrl_dev         │  │
                    │  .desc  ──► pinctrl_desc        │  │
                    │  .chip  ──► gpio_chip           │  │
                    │  .reg_mux  ──► regmap           │  │
                    │  .reg_gpio ──► regmap           │  │
                    │  .reg_pull ──► regmap           │  │
                    │  .reg_pullen──► regmap          │  │
                    │  .reg_ds   ──► regmap           │  │
                    └─────────────────────────────────┘  │
                                                          │
              ┌───────────────────────────────────────────┘
              ▼
   meson_axg_pmx_data
   .pmx_banks ──► meson_pmx_bank[]
                  { name, first, last, reg, offset }
                  (كل bank بيحدد أي mux registers)


   meson_pmx_group
   .pins ──► unsigned int[]
   .data ──► meson_pmx_axg_data { .func = 0..4 }

   meson_pmx_func
   .groups ──► const char*[] (أسماء groups)

   meson_bank
   .regs[6] ──► meson_reg_desc[]
                { reg, bit } × 6 أنواع
```

---

### رسم بنية الـ pinmux register لكل pin

```
Mux Register (32-bit word):
Bit:  31 30 29 28 | 27 26 25 24 | ... | 7 6 5 4 | 3 2 1 0
      [ pin+7 fn ]  [ pin+6 fn ]        [pin+1fn]  [pin0fn]
       4 bits/pin    4 bits/pin           4 bits     4 bits

fn=0 → GPIO mode
fn=1 → func1 (e.g. emmc_nand_d0 on GPIOB_0)
fn=2 → func2 (e.g. nor_hold    on GPIOB_3)
fn=3 → func3
fn=4 → func4
```

---

### رسم دورة الحياة — من الـ probe للاستخدام

```
kernel boot
    │
    ▼
platform_driver_register(&t7_pinctrl_driver)
    │ .probe = meson_pinctrl_probe
    │
    ▼
meson_pinctrl_probe(pdev)
    │
    ├─► devm_kzalloc → struct meson_pinctrl *pc
    │
    ├─► of_device_get_match_data()
    │       └── pc->data = &t7_periphs_pinctrl_data
    │
    ├─► meson_pinctrl_parse_dt(pc)
    │       ├── gpiochip_node_count / gpiochip_node_get_first
    │       ├── meson_map_resource(pc, node, "mux")   → pc->reg_mux
    │       ├── meson_map_resource(pc, node, "gpio")  → pc->reg_gpio
    │       ├── meson_map_resource(pc, node, "pull")  → pc->reg_pull
    │       ├── meson_map_resource(pc, node, "pull-enable") → pc->reg_pullen
    │       ├── meson_map_resource(pc, node, "ds")   → pc->reg_ds
    │       └── pc->data->parse_dt(pc)
    │               └── meson_a1_parse_dt_extra(pc)
    │                     (reg_pull = reg_pullen = reg_ds = reg_gpio)
    │
    ├─► pinctrl_desc تجهيز:
    │       pc->desc.pctlops  = &meson_pctrl_ops
    │       pc->desc.pmxops   = &meson_axg_pmx_ops
    │       pc->desc.confops  = &meson_pinconf_ops
    │       pc->desc.pins     = t7_periphs_pins
    │       pc->desc.npins    = 169
    │
    ├─► devm_pinctrl_register(dev, &pc->desc, pc)
    │       └── pc->pcdev (pinctrl core يسجّل الـ controller)
    │
    └─► meson_gpiolib_register(pc)
            ├── gpio_chip تجهيز (get/set/direction...)
            └── gpiochip_add_data(&pc->chip, pc)

═══════════════════════════════════════════════════════════
runtime: طلب تفعيل function
═══════════════════════════════════════════════════════════

device driver (مثلاً MMC)
    │
    ▼
pinctrl_select_state(pinctrl, state)  [pinctrl core]
    │
    ▼
meson_axg_pmx_set_mux(pcdev, func_num, group_num)
    │
    ├── func = &pc->data->funcs[func_num]   /* e.g. emmc */
    ├── group = &pc->data->groups[group_num] /* e.g. emmc_clk */
    ├── pmx_data = group->data              /* func=1 */
    │
    └── for each pin in group.pins:
            meson_axg_pmx_update_function(pc, pin, func=1)
                ├── meson_axg_pmx_get_bank(pc, pin, &bank)
                │       (يبحث في pmx_banks عن البنك الصح)
                ├── meson_pmx_calc_reg_and_offset(bank, pin, &reg, &offset)
                │       reg    = bank->reg + (bank->offset + shift*4) / 32
                │       offset = (bank->offset + shift*4) % 32
                └── regmap_update_bits(pc->reg_mux,
                        reg<<2,
                        0xf << offset,
                        func << offset)
                    (يكتب 4 bits في الـ mux register)

═══════════════════════════════════════════════════════════
runtime: GPIO request (تحويل pin لـ GPIO)
═══════════════════════════════════════════════════════════

gpiochip_generic_request → pinctrl core
    │
    ▼
meson_axg_pmx_request_gpio(pcdev, range, offset)
    │
    └── meson_axg_pmx_update_function(pc, offset, func=0)
            (يكتب 0 في الـ 4 bits → GPIO mode)

═══════════════════════════════════════════════════════════
runtime: GPIO direction/value
═══════════════════════════════════════════════════════════

gpio_direction_output(gpio, value)
    │
    ▼
meson_gpio_direction_output(chip, gpio, value)
    │
    └── meson_pinconf_set_output_drive(pc, gpio, value)
            ├── meson_get_bank(pc, gpio, &bank)
            ├── meson_calc_reg_and_bit(bank, gpio, MESON_REG_DIR, &reg, &bit)
            │       bit = desc->bit + (gpio - bank->first) * stride
            │       reg = (desc->reg + bit/32) * 4
            ├── regmap_update_bits(pc->reg_gpio, reg, BIT(bit), 0)  /* output */
            └── regmap_update_bits(pc->reg_gpio, reg, BIT(bit), value)
```

---

### رسم Call Flow — pinconf (pull / drive-strength)

```
pinconf_generic_dt_node_to_map_all()   [DT parsing]
    │
    ▼ (kernel applies pin config)
meson_pinconf_set(pcdev, pin, configs[], num_configs)
    │
    ├── PIN_CONFIG_BIAS_PULL_UP:
    │       meson_pinconf_set_pullup(pc, pin, true)
    │           ├── meson_get_bank → bank
    │           ├── meson_calc_reg_and_bit(MESON_REG_PULLEN) → write 1
    │           └── meson_calc_reg_and_bit(MESON_REG_PULL)   → write 1
    │
    ├── PIN_CONFIG_BIAS_PULL_DOWN:
    │       مثل pull-up لكن MESON_REG_PULL → write 0
    │
    ├── PIN_CONFIG_BIAS_DISABLE:
    │       meson_calc_reg_and_bit(MESON_REG_PULLEN) → write 0
    │
    └── PIN_CONFIG_DRIVE_STRENGTH_UA:
            meson_pinconf_set_ds_raw(pc, pin, ds_val)
                ├── meson_get_bank → bank
                ├── meson_calc_reg_and_bit(MESON_REG_DS, &reg, &bit)
                │       (stride=2 → bit = desc->bit + (pin-first)*2)
                └── regmap_update_bits(pc->reg_ds, reg, 0x3<<bit, ds<<bit)
```

---

### استراتيجية الـ Locking

الـ driver ما بيعمل explicit locking على مستوى الـ structs — بيعتمد على:

| الطبقة | آلية الـ Locking | اللي بتحميه |
|--------|----------------|------------|
| **pinctrl core** | `mutex` داخلي في `pinctrl_dev` | تسلسل استدعاءات `set_mux` و`pin_config_set` |
| **regmap** | `regmap->lock` (spinlock أو mutex حسب الـ config) | الوصول الـ atomic للـ MMIO registers عبر `regmap_update_bits` |
| **gpio_chip** | `gpio_chip->bgpio_lock` (إذا استخدم `bgpio_init`) | هنا بيستخدم `gpiochip_add_data` بدون bgpio — الـ locking للـ gpio direction/value عبر regmap |
| **devm** | lifecycle مربوط بالـ device | الـ memory والـ regmaps |

**ترتيب الـ locks** (lock ordering):
```
pinctrl_dev->mutex
    └── regmap->lock  (دايمًا من الداخل — مش العكس)
```

**`regmap_update_bits`** هي atomic read-modify-write — بتضمن إن تغيير بت واحد في الـ mux register ما يأثرش على الـ pins التانية في نفس الـ register.

---

### ملخص العلاقة بين كل الـ Data Structures

```
of_device_id
  └── .data → meson_pinctrl_data
                   │
          ┌────────┼──────────────┬─────────────────┐
          ▼        ▼              ▼                 ▼
   pinctrl_    meson_pmx_    meson_pmx_      meson_bank[]
   pin_desc[]  group[]       func[]          (regs[6]=meson_reg_desc)
   (169 pins)  (groups+GPIO) (functions)     (12 banks)
                   │
                   └── .data → meson_pmx_axg_data { func=1..4 }

                                   meson_pinctrl_data
                                        │
                                   .pmx_data
                                        │
                                   meson_axg_pmx_data
                                        │
                                   meson_pmx_bank[]
                                   (12 banks, reg+offset للـ mux)
```

الفرق الجوهري:
- **`meson_bank`**: بيتحكم في pull / dir / in / out / DS (عبر `reg_gpio`)
- **`meson_pmx_bank`**: بيتحكم في الـ mux function selection (عبر `reg_mux`)
- الاتنين منفصلين لأن الـ registers الخاصة بيهم في memory space مختلف.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet لكل الـ APIs والـ Macros المهمة

#### Data Definition Macros

| Macro / API | الغرض | الملف المصدر |
|---|---|---|
| `MESON_PIN(x)` | يعرّف `pinctrl_pin_desc` لـ pin واحد | `pinctrl-meson.h` |
| `GPIO_GROUP(gpio)` | يعرّف `meson_pmx_group` لـ GPIO mode (func=0) | `pinctrl-meson-axg-pmx.h` |
| `GROUP(grp, f)` | يعرّف `meson_pmx_group` لـ alternate function | `pinctrl-meson-axg-pmx.h` |
| `FUNCTION(fn)` | يعرّف `meson_pmx_func` بربطها بـ groups array | `pinctrl-meson.h` |
| `BANK_DS(...)` | يعرّف `meson_bank` مع كل registers (pullen, pull, dir, out, in, ds) | `pinctrl-meson.h` |
| `BANK_PMX(n,f,l,r,o)` | يعرّف `meson_pmx_bank` — pinmux register map لكل bank | `pinctrl-meson-axg-pmx.h` |
| `PMX_DATA(f)` | يعرّف `meson_pmx_axg_data` — func number داخل GROUP | `pinctrl-meson-axg-pmx.h` |

#### Runtime APIs (من الـ framework — مش defined في الملف ده)

| Function | الغرض |
|---|---|
| `meson_pinctrl_probe()` | الـ probe الرئيسي — يسجل الـ pinctrl + GPIO chip |
| `meson_a1_parse_dt_extra()` | يـ parse الـ DT extra nodes (regmaps) لنمط A1/T7 |
| `meson_axg_pmx_ops` | الـ `pinmux_ops` vtable — enable/disable/set_mux |
| `module_platform_driver()` | يسجل الـ `platform_driver` بشكل static |
| `MODULE_DEVICE_TABLE()` | يربط الـ `of_device_id` بالـ module للـ auto-load |

---

### Category 1: Pin Descriptor Table — `t7_periphs_pins[]`

هذه الـ category مسؤولة عن تعريف كل الـ physical pins المتاحة في الـ SoC وتسجيلها في الـ pinctrl framework.

#### الـ Array: `t7_periphs_pins[]`

```c
static const struct pinctrl_pin_desc t7_periphs_pins[] = {
    MESON_PIN(GPIOB_0),
    MESON_PIN(GPIOB_1),
    /* ... */
    MESON_PIN(GPIO_TEST_N),
};
```

**ما تعمله:**
- الـ array ده هو الـ "census" الكامل لكل pin موجود في الـ T7 periphs domain.
- كل entry عبارة عن `struct pinctrl_pin_desc { .number = X, .name = "X" }` يتولد عن طريق `MESON_PIN(x)` اللي يـ expand لـ `PINCTRL_PIN(x, #x)`.
- الـ T7 فيه 12 bank: B(13), C(7), X(20), W(17), D(13), E(7), Z(14), T(24), M(14), Y(19), H(8), TEST_N(1) — إجمالي **157 pin**.

**الـ Macro `MESON_PIN(x)`:**

```c
#define MESON_PIN(x) PINCTRL_PIN(x, #x)
// expands to:
// { .number = x, .name = #x }
```

- **`x`**: الـ enum value للـ pin (مُعرّف في `dt-bindings/gpio/amlogic,t7-periphs-pinctrl.h`)
- الـ name بيبقى نفس الـ symbol كـ string (مثلاً `"GPIOB_0"`)
- الـ framework بيستخدم الـ number كـ index في الـ groups/banks lookup

---

### Category 2: Pin Arrays لكل Function Group — `xxx_pins[]`

```c
static const unsigned int emmc_nand_d0_pins[] = { GPIOB_0 };
static const unsigned int sdcard_d0_pins[]    = { GPIOC_0 };
/* ... إلخ */
```

**ما تعمله:**
كل array بتحدد الـ pins اللي تنتمي لـ mux group واحد. معظم الـ groups في T7 هي single-pin groups (pin واحد بيشغل function واحد). الـ arrays دي بتُستخدم مباشرة من الـ `GROUP()` macro.

**نمط التسمية:**
- **اسم الـ signal** + **اسم الـ bank أو رقم الـ pin** إذا كان الـ signal موجود على أكثر من bank.
- مثلاً: `uart_ao_a_tx_d0_pins` → UART AO A TX على Bank D Pin 0، وكذلك `uart_ao_a_tx_w2_pins` → نفس الـ function على Bank W Pin 2.

**أهمية التمييز:**
الـ T7 يدعم nonfixed routing — نفس الـ peripheral ممكن يُوصل على أكثر من bank، والـ driver بيديك flexibility إنك تختار أي bank.

---

### Category 3: PMX Groups Table — `t7_periphs_groups[]`

#### الـ Array: `t7_periphs_groups[]`

```c
static const struct meson_pmx_group t7_periphs_groups[] = {
    /* GPIO groups (func=0) */
    GPIO_GROUP(GPIOB_0),
    /* ... */
    /* Function groups */
    GROUP(emmc_nand_d0, 1),
    GROUP(nor_hold,     2),
    /* ... */
};
```

**ما تعمله:**
الـ array ده هو الـ complete pinmux group registry. كل entry فيه هو `struct meson_pmx_group` بيحتوي على:
- **الاسم** كـ string (للـ DT lookup)
- **الـ pins array** و عددها
- **الـ data pointer** → `meson_pmx_axg_data { .func = N }` حيث N هو رقم الـ alternate function (0=GPIO, 1..4=AFs)

#### الـ Macro `GPIO_GROUP(gpio)`:

```c
#define GPIO_GROUP(gpio) \
    { \
        .name = #gpio, \
        .pins = (const unsigned int[]){ gpio }, \
        .num_pins = 1, \
        .data = (const struct meson_pmx_axg_data[]){ PMX_DATA(0) }, \
    }
```

- بيعرّف group باسم الـ GPIO نفسه وـ func=0 (GPIO mode).
- الـ framework بيستخدمه لما تختار إن الـ pin يشتغل كـ plain GPIO.

#### الـ Macro `GROUP(grp, f)`:

```c
#define GROUP(grp, f) \
    { \
        .name = #grp, \
        .pins = grp ## _pins, \
        .num_pins = ARRAY_SIZE(grp ## _pins), \
        .data = (const struct meson_pmx_axg_data[]){ PMX_DATA(f) }, \
    }
```

- **`grp`**: اسم الـ group — بيحدد الـ `grp_pins[]` array تلقائياً بالـ token pasting.
- **`f`**: رقم الـ function (1-4) — بيتحفظ في الـ `meson_pmx_axg_data.func` ويُستخدم من `meson_axg_pmx_ops` وقت الـ `set_mux` لكتابة الـ value الصح في الـ mux register.

**الـ Function Numbers في T7:**
```
func=0  → GPIO (default)
func=1  → Primary alternate function
func=2  → Secondary alternate function
func=3  → Tertiary alternate function
func=4  → Quaternary alternate function
```

---

### Category 4: Function Groups Strings — `xxx_groups[]`

```c
static const char * const emmc_groups[] = {
    "emmc_nand_d0", "emmc_nand_d1", /* ... */
};

static const char * const spi0_groups[] = {
    "spi0_mosi", "spi0_miso", "spi0_sclk",
    "spi0_ss0",  "spi0_ss1",  "spi0_ss2",
};
```

**ما تعمله:**
كل array بتجمع أسماء الـ PMX groups اللي تخص peripheral واحد. الـ pinctrl framework بيستخدمها للـ DT binding لما device-driver بيطلب function معين — بيبحث عن الـ group المناسب بالاسم.

**ملاحظات مهمة:**
- الـ `gpio_periphs_groups[]` هي أكبر array وبتحتوي على أسماء كل الـ GPIO pins — بتُستخدم للـ GPIO fallback function.
- الـ functions المعقدة زي `i2c2` بيكون ليها groups في أكثر من bank:
  ```c
  static const char * const i2c2_groups[] = {
      "i2c2_sda_x", "i2c2_sck_x",  /* Bank X */
      "i2c2_sda_t", "i2c2_sck_t",  /* Bank T */
      "i2c2_sda_m", "i2c2_sck_m",  /* Bank M */
  };
  ```
  الـ DT author بيختار أي bank يستخدم بـ `pinctrl-0` و `pinctrl-names`.

**قائمة الـ Peripherals المدعومة (T7):**
| Category | الـ Peripherals |
|---|---|
| Storage | emmc, nor, sdcard, sdio |
| UART | uart_c, uart_d, uart_e, uart_f, uart_ao_a, uart_ao_b |
| SPI | spi0, spi1, spi2, spi3, spi4, spi5 |
| I2C | i2c0, i2c1, i2c2, i2c3, i2c4, i2c5, i2c0_ao, i2c1_ao, i2c0_slave_ao |
| Audio | tdm, mclk, pdm, spdif_in, spdif_out |
| Video | hdmirx_a/b/c, hdmitx, hsync, vsync, sync_3d, vx1_a/b, edp_a/b |
| PWM | pwm_a..f, pwm_ao_a..h, pwm_vs, pwm_ao_c_hiz, pwm_ao_g_hiz |
| Transport Stream | tsin_a, tsin_b, tsin_c, tsin_d |
| Ethernet | eth |
| Debug/Misc | jtag_a, jtag_b, gen_clk, clk12_24, clk25m, remote_in/out, rtc_clk, iso7816, pcieck, wd_rsto, mic_mute, cec_a/b |

---

### Category 5: Functions Table — `t7_periphs_functions[]`

#### الـ Array: `t7_periphs_functions[]`

```c
static const struct meson_pmx_func t7_periphs_functions[] = {
    FUNCTION(gpio_periphs),
    FUNCTION(emmc),
    FUNCTION(nor),
    /* ... 80+ functions ... */
    FUNCTION(mic_mute),
};
```

**ما تعمله:**
الـ array ده هو الـ function registry الكامل اللي الـ pinctrl framework بيـ expose للـ userspace والـ device drivers. كل entry هو `struct meson_pmx_func`:

```c
struct meson_pmx_func {
    const char *name;          /* "emmc", "spi0", ... */
    const char * const *groups; /* pointer to xxx_groups[] */
    unsigned int num_groups;
};
```

#### الـ Macro `FUNCTION(fn)`:

```c
#define FUNCTION(fn) \
    { \
        .name = #fn, \
        .groups = fn ## _groups, \
        .num_groups = ARRAY_SIZE(fn ## _groups), \
    }
```

- بيعمل token-paste للـ `fn` مع `_groups` عشان يوصّل كل function بـ groups array المناسبة.
- الاسم بيبقى نفس الاسم في الـ DT property `function = "emmc"`.

**Pseudocode — كيف الـ framework يستخدمها:**
```
DT: pinctrl-0 = &emmc_pins_default
pinctrl-node: groups = "emmc_clk"; function = "emmc"

1. pinctrl_select_state() يُطلب
2. framework يبحث عن function "emmc" في t7_periphs_functions[]
3. يتحقق إن group "emmc_clk" موجود في emmc_groups[]
4. يجيب الـ meson_pmx_group للـ group ده
5. يستدعي meson_axg_pmx_ops.set_mux() بالـ func number من الـ data
6. meson_axg_pmx_ops يكتب func value في الـ mux register المناسب
```

---

### Category 6: GPIO Banks — `t7_periphs_banks[]`

#### الـ Array: `t7_periphs_banks[]`

```c
static const struct meson_bank t7_periphs_banks[] = {
    /* name  first      last        irq_first  irq_last */
    BANK_DS("D", GPIOD_0, GPIOD_12, 57, 69,
    /*  pullen_reg  pullen_bit  pull_reg  pull_bit
        dir_reg     dir_bit     out_reg   out_bit
        in_reg      in_bit      ds_reg    ds_bit  */
        0x03, 0,  0x04, 0,  0x02, 0,  0x01, 0,  0x00, 0,  0x07, 0),
    /* ... */
};
```

**ما تعمله:**
الـ array ده بيحدد لكل GPIO bank:
1. **أول وآخر pin** في الـ bank.
2. **range الـ hardware IRQ numbers** المرتبطة بالـ bank.
3. **Register offsets** لكل عملية GPIO (pull-enable, pull-direction, direction, output, input, drive-strength).

#### الـ Macro `BANK_DS`:

```c
#define BANK_DS(n, f, l, fi, li,
                per, peb,   /* pullen: reg, bit */
                pr,  pb,    /* pull: reg, bit */
                dr,  db,    /* dir: reg, bit */
                or,  ob,    /* out: reg, bit */
                ir,  ib,    /* in: reg, bit */
                dsr, dsb)   /* ds: reg, bit */
```

- الـ `bit` في كل زوج هو الـ **base bit** للـ pin الأول في الـ bank — الـ runtime code بيضيف `pin - bank->first` للوصول لأي pin داخل الـ bank.
- الـ `reg` هو offset في الـ regmap (وحدة word).
- الـ T7 بيستخدم `BANK_DS` (مع drive-strength) — مش `BANK` البسيط.

**مثال عملي — Bank D:**
```
BANK_DS("D", GPIOD_0, GPIOD_12, 57, 69,
    pullen=0x03, pull=0x04, dir=0x02,
    out=0x01,    in=0x00,   ds=0x07)

لتشغيل GPIOD_5 كـ output:
  pin_offset = GPIOD_5 - GPIOD_0 = 5
  dir_reg = 0x02, dir_bit = 0 + 5 = 5
  → regmap_update_bits(reg_gpio, 0x02*4, BIT(5), 0)  // 0=output
  out_reg = 0x01, out_bit = 5
  → regmap_update_bits(reg_gpio, 0x01*4, BIT(5), BIT(5))  // set high
```

**جدول الـ Banks في T7:**

| Bank | First | Last | IRQ Range | pullen | pull | dir | out | in | ds |
|---|---|---|---|---|---|---|---|---|---|
| D | GPIOD_0 | GPIOD_12 | 57-69 | 0x03 | 0x04 | 0x02 | 0x01 | 0x00 | 0x07 |
| E | GPIOE_0 | GPIOE_6 | 70-76 | 0x0b | 0x0c | 0x0a | 0x09 | 0x08 | 0x0f |
| Z | GPIOZ_0 | GPIOZ_13 | 77-90 | 0x13 | 0x14 | 0x12 | 0x11 | 0x10 | 0x17 |
| H | GPIOH_0 | GPIOH_7 | 148-155 | 0x1b | 0x1c | 0x1a | 0x19 | 0x18 | 0x1f |
| C | GPIOC_0 | GPIOC_6 | 13-19 | 0x23 | 0x24 | 0x22 | 0x21 | 0x20 | 0x27 |
| B | GPIOB_0 | GPIOB_12 | 0-12 | 0x2b | 0x2c | 0x2a | 0x29 | 0x28 | 0x2f |
| X | GPIOX_0 | GPIOX_19 | 20-39 | 0x33 | 0x34 | 0x32 | 0x31 | 0x30 | 0x37 |
| T | GPIOT_0 | GPIOT_23 | 91-114 | 0x43 | 0x44 | 0x42 | 0x41 | 0x40 | 0x47 |
| Y | GPIOY_0 | GPIOY_18 | 129-147 | 0x53 | 0x54 | 0x52 | 0x51 | 0x50 | 0x57 |
| W | GPIOW_0 | GPIOW_16 | 40-56 | 0x63 | 0x64 | 0x62 | 0x61 | 0x60 | 0x67 |
| M | GPIOM_0 | GPIOM_13 | 115-128 | 0x73 | 0x74 | 0x72 | 0x71 | 0x70 | 0x77 |
| TEST_N | GPIO_TEST_N | GPIO_TEST_N | 156-156 | 0x83 | 0x84 | 0x82 | 0x81 | 0x80 | 0x87 |

**نمط الـ Register Layout:**
الـ T7 بيستخدم نمط ثابت لكل bank: الـ base address بتزيد بمقدار 0x08 بين كل bank، وداخل كل bank الـ registers متسلسلة (in=base, out=base+1, dir=base+2, pullen=base+3, pull=base+4, ds=base+7).

---

### Category 7: Pinmux Banks — `t7_periphs_pmx_banks[]`

#### الـ Array: `t7_periphs_pmx_banks[]`

```c
static const struct meson_pmx_bank t7_periphs_pmx_banks[] = {
    /*      name    first       last        reg    offset */
    BANK_PMX("D",  GPIOD_0,  GPIOD_12,  0x0a,  0),
    BANK_PMX("E",  GPIOE_0,  GPIOE_6,   0x0c,  0),
    /* ... */
};
```

**ما تعمله:**
كل entry بيحدد الـ mux register base لكل GPIO bank. الـ `meson_axg_pmx_ops` بيستخدم الـ array ده لمعرفة فين بيكتب الـ function selector.

#### الـ Macro `BANK_PMX(n, f, l, r, o)`:

```c
#define BANK_PMX(n, f, l, r, o) \
    { .name=n, .first=f, .last=l, .reg=r, .offset=o }
```

- **`reg`**: الـ register الأول في الـ mux register block للـ bank ده.
- **`offset`**: الـ bit offset للـ pin الأول في الـ bank.
- الـ T7 بيستخدم **4 bits per pin** في الـ mux registers — يعني كل register يحتوي على 8 pins.
- لـ pin رقم N في bank بـ first=F وـ reg=R وـ offset=O:
  ```
  abs_bit = (N - F) * 4 + O
  reg_offset = R + (abs_bit / 32)
  bit_in_reg = abs_bit % 32
  → write func_value (0..4) at bits [bit_in_reg+3 : bit_in_reg]
  ```

**مثال:**
```
BANK_PMX("B", GPIOB_0, GPIOB_12, 0x00, 0)
لـ GPIOB_8 (emmc_clk, func=1):
  abs_bit = (8-0)*4 + 0 = 32
  reg = 0x00 + (32/32) = 0x01
  bit = 32%32 = 0
  → write 1 at bits [3:0] of reg 0x01
```

---

### Category 8: Driver Registration Data

#### الـ Struct: `t7_periphs_pmx_banks_data`

```c
static const struct meson_axg_pmx_data t7_periphs_pmx_banks_data = {
    .pmx_banks     = t7_periphs_pmx_banks,
    .num_pmx_banks = ARRAY_SIZE(t7_periphs_pmx_banks),
};
```

**ما تعمله:**
هذا الـ wrapper struct بيجمع الـ pmx banks array مع عددها في structure واحدة. بيتمرر لـ `meson_axg_pmx_ops` وقت الـ probe عبر `meson_pinctrl_data.pmx_data`.

---

#### الـ Struct: `t7_periphs_pinctrl_data`

```c
static const struct meson_pinctrl_data t7_periphs_pinctrl_data = {
    .name      = "periphs-banks",
    .pins      = t7_periphs_pins,
    .groups    = t7_periphs_groups,
    .funcs     = t7_periphs_functions,
    .banks     = t7_periphs_banks,
    .num_pins  = ARRAY_SIZE(t7_periphs_pins),
    .num_groups= ARRAY_SIZE(t7_periphs_groups),
    .num_funcs = ARRAY_SIZE(t7_periphs_functions),
    .num_banks = ARRAY_SIZE(t7_periphs_banks),
    .pmx_ops   = &meson_axg_pmx_ops,
    .pmx_data  = &t7_periphs_pmx_banks_data,
    .parse_dt  = &meson_a1_parse_dt_extra,
};
```

**ما تعمله:**
الـ struct ده هو الـ "personality" الكاملة للـ T7 pinctrl driver. الـ `meson_pinctrl_probe()` المشترك بياخد الـ pointer ده من الـ `of_device_id.data` ويبني كل الـ pinctrl infrastructure عليه.

**الـ Fields المهمة:**
| Field | القيمة | الأهمية |
|---|---|---|
| `pins` | `t7_periphs_pins` | 157 pin descriptor |
| `groups` | `t7_periphs_groups` | 157 GPIO + 300+ function groups |
| `funcs` | `t7_periphs_functions` | 80+ functions |
| `banks` | `t7_periphs_banks` | 12 GPIO banks |
| `pmx_ops` | `&meson_axg_pmx_ops` | AXG-style 4-bit mux |
| `pmx_data` | `&t7_periphs_pmx_banks_data` | mux register map |
| `parse_dt` | `&meson_a1_parse_dt_extra` | A1-style DT parsing (unified regmap) |

**لماذا `meson_a1_parse_dt_extra`؟**
الـ T7 يتبع نفس نمط الـ A1 في الـ DT حيث الـ GPIO والـ mux registers تجي من نفس الـ regmap node (مش من أكثر من node زي SoCs القديمة).

---

#### الـ Array: `t7_pinctrl_dt_match[]`

```c
static const struct of_device_id t7_pinctrl_dt_match[] = {
    {
        .compatible = "amlogic,t7-periphs-pinctrl",
        .data = &t7_periphs_pinctrl_data,
    },
    { }  /* sentinel */
};
MODULE_DEVICE_TABLE(of, t7_pinctrl_dt_match);
```

**ما تعمله:**
- بيربط الـ compatible string في الـ DTS بالـ driver data.
- لما الـ kernel يلاقي node بـ `compatible = "amlogic,t7-periphs-pinctrl"` في الـ DT، بيـ call `meson_pinctrl_probe()` مع الـ data pointer من الـ `t7_periphs_pinctrl_data`.
- `MODULE_DEVICE_TABLE` بيضيف الـ alias في الـ module metadata عشان الـ `udev/modprobe` يـ load الـ driver تلقائياً.

---

#### الـ Driver: `t7_pinctrl_driver` و `module_platform_driver()`

```c
static struct platform_driver t7_pinctrl_driver = {
    .probe  = meson_pinctrl_probe,
    .driver = {
        .name           = "amlogic-t7-pinctrl",
        .of_match_table = t7_pinctrl_dt_match,
    },
};
module_platform_driver(t7_pinctrl_driver);
```

**ما تعمله:**
- **`t7_pinctrl_driver`**: يعرّف الـ `platform_driver` struct — الـ probe هو `meson_pinctrl_probe` المشترك بين كل Meson SoCs.
- **`module_platform_driver()`**: macro بيـ expand إلى `module_init()` و `module_exit()` اللي بيسجلوا ويـ deregister الـ platform driver مع الـ kernel.

**Pseudocode — تسلسل الـ Probe:**
```
module_init()
  → platform_driver_register(&t7_pinctrl_driver)

kernel matches DT node "amlogic,t7-periphs-pinctrl"
  → t7_pinctrl_driver.probe(pdev)
    = meson_pinctrl_probe(pdev)
      → of_match_device() → gets t7_periphs_pinctrl_data
      → allocate struct meson_pinctrl *pc
      → pc->data = t7_periphs_pinctrl_data
      → t7_periphs_pinctrl_data.parse_dt(pc)
         = meson_a1_parse_dt_extra(pc)
           → of_parse_phandle() → get regmap handles
           → pc->reg_gpio = regmap for GPIO
           → pc->reg_mux  = regmap for mux (same or separate)
      → pinctrl_register(&pc->desc, ...)
         → registers t7_periphs_pins[]
         → registers t7_periphs_groups[]
         → registers t7_periphs_functions[]
      → gpiochip_add(&pc->chip)
         → registers 157 GPIO lines
         → sets up irqchip from t7_periphs_banks[]
```

---

### Category 9: الـ Inherited Runtime Functions (من `pinctrl-meson.c`)

هذه الـ functions غير معرّفة في الملف الحالي لكنها جزء لا يتجزأ من سلوك الـ driver:

#### `meson_pinctrl_probe()`

```c
int meson_pinctrl_probe(struct platform_device *pdev);
```

- الـ probe المشترك لكل Meson pinctrl drivers.
- يُستدعى من الـ platform bus عند match الـ DT compatible.
- يقرأ الـ `of_device_id.data` → `meson_pinctrl_data`.
- يستدعي `parse_dt` callback (في T7: `meson_a1_parse_dt_extra`).
- يسجل الـ pinctrl device وـ GPIO chip.
- **Caller context**: `__init` / module load time.

#### `meson_a1_parse_dt_extra()`

```c
int meson_a1_parse_dt_extra(struct meson_pinctrl *pc);
```

- يـ parse الـ DT node لجيب الـ regmap handles.
- الـ T7 (زي الـ A1) بيستخدم `syscon` regmap مشترك للـ GPIO والـ mux registers.
- يملأ `pc->reg_gpio`, `pc->reg_mux`, `pc->reg_pullen`, `pc->reg_pull`, `pc->reg_ds`.
- **Key detail**: مش محتاج multiple `reg` properties زي SoCs قديمة — كل شيء في regmap واحد.

#### `meson_axg_pmx_ops` (extern)

```c
extern const struct pinmux_ops meson_axg_pmx_ops;
```

الـ vtable اللي بتحتوي على:
- **`set_mux()`**: بيكتب الـ func number (0-4) في الـ 4-bit field المناسب في الـ mux register، مستعيناً بـ `t7_periphs_pmx_banks[]` لتحديد الـ register والـ bit.
- **`gpio_set_direction()`**: بيـ disable الـ mux (func=0) لما الـ pin يتحول لـ GPIO.
- **Key detail**: الـ AXG PMX يستخدم 4 bits per pin (يدعم حتى 15 function)، بينما الـ T7 يحتاج بس 3 bits (max func=4) لكن الـ hardware layout هو 4-bit.

---

### Architecture Overview — كيف الـ Data Structures تتكامل

```
t7_pinctrl_dt_match[]
        │
        │ .data pointer
        ▼
t7_periphs_pinctrl_data
   ├── .pins  ──────────► t7_periphs_pins[]       (157 pinctrl_pin_desc)
   ├── .groups ─────────► t7_periphs_groups[]      (GPIO + function groups)
   │                            │
   │                            └─ GROUP(emmc_clk, 1)
   │                                  ├── .pins = emmc_clk_pins[]
   │                                  └── .data = { .func = 1 }
   ├── .funcs ──────────► t7_periphs_functions[]   (80+ meson_pmx_func)
   │                            │
   │                            └─ FUNCTION(emmc)
   │                                  └── .groups = emmc_groups[]
   ├── .banks ──────────► t7_periphs_banks[]        (12 meson_bank)
   │                            └─ BANK_DS("B", ...) GPIO regs
   ├── .pmx_ops ────────► meson_axg_pmx_ops         (set_mux vtable)
   ├── .pmx_data ───────► t7_periphs_pmx_banks_data
   │                            └─ t7_periphs_pmx_banks[]  (mux reg map)
   └── .parse_dt ───────► meson_a1_parse_dt_extra()

platform_driver.probe = meson_pinctrl_probe()
  → reads all above → registers pinctrl_dev + gpio_chip
```
## Phase 5: دليل الـ Debugging الشامل

الـ driver اللي بندرسه هو `pinctrl-amlogic-t7.c` — بيتحكم في الـ **pin controller** والـ **GPIO** الخاص بـ Amlogic T7 SoC. الـ T7 عنده 12 bank (B, C, X, W, D, E, Z, T, Y, H, M, TEST_N) وكل pin ممكن يشتغل في mode مختلف (GPIO أو function مزود زي emmc, uart, i2c, spi, etc.). الـ debugging هنا بيدور حول 3 محاور: mux register صح؟، GPIO direction/value صح؟، DT binding صح؟

---

### Software Level

#### 1. debugfs Entries

الـ pinctrl subsystem بيعمل entries تلقائياً تحت `/sys/kernel/debug/pinctrl/`.

| المسار | الوصف |
|--------|-------|
| `/sys/kernel/debug/pinctrl/` | قائمة كل الـ pinctrl devices المسجلة |
| `/sys/kernel/debug/pinctrl/<dev>/pins` | كل الـ pins مع أرقامها |
| `/sys/kernel/debug/pinctrl/<dev>/pingroups` | كل الـ groups وأي pins فيها |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-functions` | كل الـ functions وأي groups بتدعمها |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-pins` | أي function assigned لكل pin حالياً |
| `/sys/kernel/debug/pinctrl/<dev>/pinconf-pins` | الـ config الحالي (pull, drive-strength) لكل pin |
| `/sys/kernel/debug/gpio` | حالة كل GPIO line (direction, value, owner) |

```bash
# اعرف كل الـ pinctrl controllers الموجودة
ls /sys/kernel/debug/pinctrl/

# مثلاً output هيبقى:
# pinctrl-handles  pinctrl-maps  ff634480.bus:periphs-banks@0

# اقرأ الـ mux الحالي لكل pin
cat /sys/kernel/debug/pinctrl/ff634480.bus\:periphs-banks@0/pinmux-pins

# output sample:
# pin 0 (GPIOB_0): (MUX UNCLAIMED) (GPIO UNCLAIMED)
# pin 8 (GPIOB_8): device ff634480.mmc function emmc group emmc_clk

# اشوف الـ pull/drive-strength
cat /sys/kernel/debug/pinctrl/ff634480.bus\:periphs-banks@0/pinconf-pins

# اشوف كل الـ groups
cat /sys/kernel/debug/pinctrl/ff634480.bus\:periphs-banks@0/pingroups
```

#### 2. sysfs Entries

```bash
# GPIO sysfs interface (legacy, كمان مفيد للـ debug السريع)
# اعرف أي GPIO number يقابل GPIOX_0 (bank X أول pin)
# Bank X بتبدأ من pin 20 في الـ T7 (حسب الـ irq_first في BANK_DS)

# لو الـ gpiochip مسجل
ls /sys/class/gpio/
cat /sys/class/gpio/gpiochip0/label
cat /sys/class/gpio/gpiochip0/base
cat /sys/class/gpio/gpiochip0/ngpio

# export pin عشان تتحكم فيه
echo 20 > /sys/class/gpio/export
cat /sys/class/gpio/gpio20/direction   # in أو out
cat /sys/class/gpio/gpio20/value       # 0 أو 1

# libgpiod أحسن بكتير
gpiodetect                              # اعرف كل الـ gpiochip devices
gpioinfo gpiochip0                      # اعرف كل الـ lines ومين بيستخدمها
gpioget gpiochip0 20                    # اقرأ GPIOX_0
gpioset gpiochip0 20=1                  # اكتب GPIOX_0
```

#### 3. ftrace — Tracepoints والـ Events

الـ pinctrl subsystem عنده tracepoints مدمجة.

```bash
# شوف الـ events المتاحة
ls /sys/kernel/debug/tracing/events/gpio/
ls /sys/kernel/debug/tracing/events/regmap/   # الـ T7 بيستخدم regmap

# فعّل GPIO tracing
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل الكود اللي بتشوف فيه المشكلة
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# فعّل regmap tracing عشان تشوف الـ register reads/writes
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/enable
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_read/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# output sample:
# regmap_reg_write: amlogic-t7-pinctrl reg=0x02 val=0x00000001
# gpio_value: gpio=20 set=1

# فعّل function tracer على meson_pinctrl_probe
echo function > /sys/kernel/debug/tracing/current_tracer
echo meson_pinctrl_probe > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

#### 4. printk / dynamic debug

```bash
# فعّل dynamic debug لكل الـ pinctrl subsystem
echo "module pinctrl_meson +p" > /sys/kernel/debug/dynamic_debug/control

# أو خليه أكتر تحديداً على الـ T7 driver
echo "file pinctrl-amlogic-t7.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل dynamic debug لـ regmap (لأن الـ driver بيستخدم regmap بشكل أساسي)
echo "module regmap +p" > /sys/kernel/debug/dynamic_debug/control

# لو عايز verbose أكتر (مع function name وline number)
echo "file pinctrl-amlogic-t7.c +pflmt" > /sys/kernel/debug/dynamic_debug/control

# شوف الـ logs
dmesg -w | grep -i pinctrl
dmesg -w | grep -i "amlogic-t7"
dmesg -w | grep -i regmap
```

#### 5. Kernel Config للـ Debugging

| الـ Option | الوصف |
|-----------|-------|
| `CONFIG_DEBUG_PINCTRL` | يفعّل verbose logging في الـ pinctrl core |
| `CONFIG_GPIO_SYSFS` | يكشف GPIO عبر sysfs للـ debug |
| `CONFIG_DEBUG_GPIO` | يفعّل extra validation في الـ GPIO layer |
| `CONFIG_REGMAP_DEBUGFS` | يعمل debugfs entries للـ regmap registers |
| `CONFIG_DYNAMIC_DEBUG` | يسمح بـ runtime enable/disable للـ pr_debug |
| `CONFIG_PROVE_LOCKING` | يكشف lock violations في الـ pinctrl |
| `CONFIG_DEBUG_KERNEL` | Base option لتفعيل معظم debugging features |
| `CONFIG_OF_DYNAMIC` | مفيد لو بتعمل DT overlay debugging |
| `CONFIG_PINCTRL_SINGLE` | reference implementation ممكن تقارن بيها |
| `CONFIG_COMPILE_TEST` | يسمح بـ build testing على x86 |

```bash
# تأكد من الـ config الموجود في kernel الحالي
zcat /proc/config.gz | grep -E "CONFIG_DEBUG_PINCTRL|CONFIG_GPIO_SYSFS|CONFIG_REGMAP_DEBUGFS"
```

#### 6. Subsystem-Specific Tools

الـ T7 driver مش بيستخدم devlink، بس فيه tools تانية مفيدة جداً:

```bash
# pinctrl utility (من package libpinctrl-utils أو kernel tools)
# لو مش موجود — استخدم مباشرة debugfs

# اعرف أي function اتعمله set لـ group معين
cat /sys/kernel/debug/pinctrl/*/pinmux-functions | grep -A5 "function: uart_c"

# اعرف الـ consumer (الـ driver اللي claim الـ pin)
cat /sys/kernel/debug/pinctrl/*/pinctrl-maps

# regmap debugfs — بيعرض كل الـ registers الحالية
# الـ T7 عنده 5 regmaps: reg_mux, reg_pullen, reg_pull, reg_gpio, reg_ds
ls /sys/kernel/debug/regmap/
# كل device هيبان باسمه
cat /sys/kernel/debug/regmap/ff634480.bus:periphs-banks@0/registers

# gpio-hammer (kernel tools) لاختبار GPIO بسرعة
# gpio-event-mon لمراقبة GPIO interrupts
```

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `pinctrl-amlogic-t7: could not request region for resource` | الـ I/O memory region مش متاحة — غالباً conflict مع driver تاني | تحقق من DT node الـ `reg` property وأنه صح وما فيش overlap |
| `failed to register pinctrl driver` | `meson_pinctrl_probe` فشل | راجع dmesg لرسايل قبليها — غالباً regmap init فشل |
| `pin X is not valid` | رقم الـ pin أكبر من `num_pins` في `meson_pinctrl_data` | تأكد إن الـ DT بيستخدم pin names موجودة في `t7_periphs_pins[]` |
| `could not get pinctrl for device` | الـ consumer driver ما لاقاش الـ pinctrl handle | تأكد إن `pinctrl-0` موجود في DT node الـ consumer |
| `pin X already requested` | pin اتطلب بالفعل من driver تاني | ابحث عن تعارض في الـ DT nodes |
| `invalid function X for pin X` | الـ function number المكتوب في GROUP() مش متاح للـ pin da | راجع `t7_periphs_groups[]` — الـ function number لازم يطابق الـ PMX bank |
| `regmap_read/write failed` | الـ regmap ما قدرش يوصل للـ register | تحقق من الـ clock أو power domain اللي بيتحكم في الـ PERIPHS block |
| `GPIO chip registration failed` | `gpiochip_add_data` فشل | غالباً numbering conflict مع gpiochip تانية في النظام |
| `request mux function failed` | `meson_axg_pmx_ops.set_mux` فشل | الـ PMX bank range مش بيغطي الـ pin da — راجع `t7_periphs_pmx_banks[]` |
| `of_get_child_by_name failed` | الـ DT subnode مش موجود | `meson_a1_parse_dt_extra` مش لاقي الـ regmap child nodes |

#### 8. Strategic Points لـ dump_stack() / WARN_ON()

```c
/* في meson_pinctrl_probe — بعد regmap_init مباشرة */
if (IS_ERR(pc->reg_mux)) {
    WARN_ON(1); /* هنا نعرف إن الـ mux regmap فشل */
    dump_stack();
}

/* في meson_axg_pmx_set_mux — لو الـ func number غير متوقع */
WARN_ON(func > 7); /* الـ T7 بيدعم max func = 4 في معظم banks */

/* في meson_pinctrl_gpio_direction — لو pin خارج النطاق */
WARN_ON(offset >= chip->ngpio);

/* لو الـ bank ما اتلاقيش لـ pin معين */
WARN_ON(!bank); /* بعد loop البحث في t7_periphs_banks */

/* في gpio_request — لو pin اتطلب مرتين */
WARN_ON(test_and_set_bit(offset, chip->valid_mask));
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

الـ T7 SoC بيستخدم regmap-based access للـ PERIPHS registers. كل bank عنده set من registers:
- `PULLEN` reg: تحكم enable الـ pull-up/pull-down
- `PULL` reg: تحدد up أو down
- `DIR` reg: input أو output
- `OUT` reg: قيمة الـ output
- `IN` reg: قيمة الـ input (read-only)
- `DS` reg: drive strength

```
Register Map لـ Bank D (GPIOD_0 .. GPIOD_12):
  PULLEN = base + 0x03*4
  PULL   = base + 0x04*4
  DIR    = base + 0x02*4
  OUT    = base + 0x01*4
  IN     = base + 0x00*4
  DS     = base + 0x07*4

Register Map لـ Bank B (GPIOB_0 .. GPIOB_12):
  PULLEN = base + 0x2b*4
  PULL   = base + 0x2c*4
  DIR    = base + 0x2a*4
  OUT    = base + 0x29*4
  IN     = base + 0x28*4
  DS     = base + 0x2f*4
```

خطوات التحقق:
1. اقرأ الـ register من الـ hardware مباشرة (devmem2)
2. قارن بالـ kernel state من debugfs
3. لو فيه فرق — يبقى إما driver bug أو hardware reset غير متوقع

#### 2. Register Dump Techniques

```bash
# افتكر: الـ PERIPHS base address للـ T7 هو في الـ DT
# اقرأه أول
cat /proc/device-tree/bus@ff600000/pinctrl@0/reg | xxd

# استخدم devmem2 لقراءة register مباشرة
# لو الـ base = 0xFF634480 مثلاً (من DT)
# وعايز تقرأ DIR register لـ Bank D (offset 0x02 × 4 = 0x08)
devmem2 0xFF634488 w   # DIR reg لـ Bank D

# أو استخدم /dev/mem
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    base = 0xFF634480
    m = mmap.mmap(f.fileno(), 0x1000, offset=base)
    val = struct.unpack('<I', m[0x08:0x0C])[0]
    print(f'GPIOD DIR = 0x{val:08X}')
    m.close()
"

# قرأ كل الـ PMX registers لـ Bank B (reg=0x00, يعني offset 0 في mux regmap)
devmem2 $((PMX_BASE + 0x00*4)) w   # mux reg لـ Bank B

# بديل باستخدام io utility (من package io أو busybox)
io -4 -r 0xFF634488
```

#### 3. Logic Analyzer / Oscilloscope Tips

**للـ I2C (مثلاً i2c2 على GPIOX_17/18):**
- قس الـ SDA و SCL بـ logic analyzer
- تحقق إن الـ pull-up resistors موجودة (4.7kΩ typical)
- ابحث عن missing ACK أو clock stretching غريب
- تحقق إن الـ drive strength مضبوط (MESON_PINCONF_DRV_500UA لـ low speed)

**للـ EMMC (Bank B):**
- استخدم oscilloscope بـ sample rate ≥ 400MHz عشان تمسك الـ HS400 signals
- قس الـ CLK (GPIOB_8) وتأكد من الـ duty cycle (50%)
- قس الـ DS (GPIOB_11) وتأكد من الـ strobe timing
- لو فيه bit errors — ابدأ بتقليل drive strength

**للـ UART debug:**
- استخدم UART-to-USB adapter
- تأكد من الـ baud rate وpolarity
- راقب الـ RX line وتأكد مش بيبقى floating (لازم pull-up أو يبقى driven)

**للـ SPI:**
- راقب الـ CS polarity (active low)
- تأكد من CPOL/CPHA settings مطابقة للـ device

```
EMMC Timing Diagram (HS200):
         _____   _____   _____
CLK  ___/     \_/     \_/     \___
         ___________
CMD  ___/           \___________
         _______________________
D0-7 ___X_______________________X___  (data valid at CLK edge)
          ^                     ^
          setup time            hold time
```

#### 4. Common Hardware Issues وـ Kernel Log Patterns

| المشكلة الـ Hardware | الـ Pattern في dmesg | التشخيص |
|--------------------|---------------------|---------|
| مش فيه pull-up على I2C | `i2c i2c-X: timeout` | قس الـ SDA/SCL بـ oscilloscope — هتلاقيها مش بترجع لـ HIGH |
| conflict في الـ pin mux (pin اتعمله assign لـ function غلط) | `request mux function failed` | قارن الـ DT pinmux config مع `t7_periphs_groups[]` |
| power domain مش enabled | `regmap_read failed: -EIO` | تأكد إن الـ PERIPHS power domain enabled قبل probe |
| clock مش enabled | `clk_enable failed` | تحقق من DT `clocks` property لـ pinctrl node |
| EMMC signal integrity | `mmc0: error -84 whilst initialising SD card` | قلل الـ bus speed وشوف هل المشكلة بتختفي |
| GPIO stuck | `gpio-X: debounce failed` | اقرأ الـ IN register مباشرة — لو الـ HW والـ SW مختلفين يبقى hardware problem |
| wrong drive strength | `eth: too many retransmissions` | زود الـ DS لـ high-speed signals زي RGMII |

#### 5. Device Tree Debugging

الـ compatible string للـ T7 هو `"amlogic,t7-periphs-pinctrl"` — لو ما اتلاقيش في DT هيفشل الـ probe.

```bash
# تحقق إن الـ DT node موجود وصح
cat /proc/device-tree/bus@ff600000/pinctrl@0/compatible | tr -d '\0'
# المتوقع: amlogic,t7-periphs-pinctrl

# شوف الـ reg property (base address وsize)
cat /proc/device-tree/bus@ff600000/pinctrl@0/reg | xxd

# شوف الـ gpio-controller property موجودة
ls /proc/device-tree/bus@ff600000/pinctrl@0/ | grep gpio

# تحقق من الـ pinmux config لـ UART node مثلاً
ls /proc/device-tree/bus@ff600000/uart@ff803000/
cat /proc/device-tree/bus@ff600000/uart@ff803000/pinctrl-names | tr -d '\0'

# تحقق من الـ overlaps في pin assignments
# الـ T7 بيستخدم meson_a1_parse_dt_extra — فيه regmap child nodes
ls /proc/device-tree/bus@ff600000/pinctrl@0/
# لازم تلاقي: gpio  mux  ds  pull  pull-enable

# شوف الـ irq config
cat /proc/device-tree/bus@ff600000/pinctrl@0/interrupt-controller | xxd
```

**مثال DT node صح للـ T7:**
```dts
pinctrl: pinctrl@0 {
    compatible = "amlogic,t7-periphs-pinctrl";
    #address-cells = <1>;
    #size-cells = <1>;
    ranges;

    gpio: gpio@ff634480 {
        reg = <0x0 0xff634480 0x0 0x800>;  /* gpio/pull/dir base */
        reg-names = "gpio";
        gpio-controller;
        #gpio-cells = <2>;
        interrupt-controller;
        #interrupt-cells = <2>;
    };

    mux@ff634000 {
        reg = <0x0 0xff634000 0x0 0x480>;  /* mux base */
        reg-names = "mux";
    };
};
```

---

### Practical Commands

#### مجموعة شاملة جاهزة للـ Copy

```bash
#!/bin/bash
# ===== T7 Pinctrl Debug Script =====

PINCTRL_DEV=$(ls /sys/kernel/debug/pinctrl/ | grep periphs | head -1)
echo "[*] Found pinctrl device: $PINCTRL_DEV"

# 1. اعرف الـ mux الحالي لكل pin
echo "=== Current Pin Mux ==="
cat "/sys/kernel/debug/pinctrl/$PINCTRL_DEV/pinmux-pins" 2>/dev/null | head -40

# 2. اعرف الـ pin config (pull, drive-strength)
echo "=== Pin Config ==="
cat "/sys/kernel/debug/pinctrl/$PINCTRL_DEV/pinconf-pins" 2>/dev/null | head -40

# 3. اعرف الـ GPIO state
echo "=== GPIO State ==="
cat /sys/kernel/debug/gpio 2>/dev/null

# 4. اعرف الـ pinctrl maps (consumer drivers)
echo "=== Pinctrl Maps ==="
cat /sys/kernel/debug/pinctrl/pinctrl-maps 2>/dev/null

# 5. شوف الـ regmap registers
echo "=== Regmap Registers ==="
ls /sys/kernel/debug/regmap/ | grep -i "periphs\|t7\|amlogic" | while read dev; do
    echo "--- $dev ---"
    cat "/sys/kernel/debug/regmap/$dev/registers" | head -30
done

# 6. فعّل dynamic debug
echo "file pinctrl-amlogic-t7.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
echo "module pinctrl_meson +p" > /sys/kernel/debug/dynamic_debug/control

# 7. فعّل GPIO tracing لمدة 5 ثواني
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
sleep 5
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo "=== GPIO Trace ==="
cat /sys/kernel/debug/tracing/trace | grep -v "^#" | head -50
echo 0 > /sys/kernel/debug/tracing/events/gpio/enable
echo > /sys/kernel/debug/tracing/trace  # clear
```

#### تشخيص مشكلة UART على GPIOX_12/13 (uart_c)

```bash
# تحقق إن uart_c_tx و uart_c_rx اتعمل لهم mux صح
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -E "GPIOX_12|GPIOX_13"

# المتوقع:
# pin 32 (GPIOX_12): device ff803000.uart function uart_c group uart_c_tx
# pin 33 (GPIOX_13): device ff803000.uart function uart_c group uart_c_rx

# لو مش كده — اعرف الـ consumer
cat /sys/kernel/debug/pinctrl/pinctrl-maps | grep -A5 "uart"

# تحقق من الـ pull config لـ RX pin
cat /sys/kernel/debug/pinctrl/*/pinconf-pins | grep "GPIOX_13"
# المتوقع: pin 33 (GPIOX_13): input bias pull up
```

#### تشخيص مشكلة EMMC على Bank B

```bash
# تحقق من كل pins الـ EMMC
for pin in GPIOB_0 GPIOB_1 GPIOB_2 GPIOB_3 GPIOB_4 GPIOB_5 GPIOB_6 GPIOB_7 GPIOB_8 GPIOB_10 GPIOB_11; do
    echo -n "$pin: "
    cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep "$pin"
done

# تحقق من drive strength (لازم يبقى 4000UA لـ HS400)
cat /sys/kernel/debug/pinctrl/*/pinconf-pins | grep -E "GPIOB_(0|1|2|3|4|5|6|7|8|10|11)"

# قرأ الـ DIR register لـ Bank B مباشرة (offset 0x2a × 4 = 0xa8)
# افتكر تحط الـ base address الصح من DT
BASE=$(cat /proc/device-tree/bus@ff600000/pinctrl@0/gpio*/reg | xxd -p | head -c16)
echo "GPIO base (hex): $BASE"
```

#### تشخيص مشكلة I2C على GPIOT_20/21

```bash
# i2c0 function على Bank T
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -E "GPIOT_20|GPIOT_21"

# اعرف الـ pull config
cat /sys/kernel/debug/pinctrl/*/pinconf-pins | grep -E "GPIOT_20|GPIOT_21"

# لو GPIOT_20 قبلاً كان spi0_sclk (func1) وعايز تشيله لـ i2c0_sck_t (func2)
# تأكد إن DT node الـ SPI مش بيطلب نفس الـ pin
grep -r "gpiot_20\|GPIOT_20" /proc/device-tree/
```

#### تفسير الـ Output

```
# output من pinmux-pins:
pin 0 (GPIOB_0): (MUX UNCLAIMED) (GPIO UNCLAIMED)
  └── الـ pin مش مستخدم من أي driver، وكمان مش مطلوب كـ GPIO

pin 8 (GPIOB_8): device ff634480.mmc function emmc group emmc_clk
  └── GPIOB_8 مستخدم من الـ mmc driver كـ emmc_clk function

# output من pinconf-pins:
pin 33 (GPIOX_13): input bias pull up, drive strength: 500 uA
  └── الـ RX pin عنده pull-up وdrive strength منخفض (صح للـ input)

# output من regmap registers:
0x00: 00000000   # Bank B OUT register — كل الـ outputs = 0
0x28: 0000ffff   # Bank B IN register — الـ bits 0-15 = 1 (floating مع pull-up)
0x2a: 00001fff   # Bank B DIR register — كل الـ pins direction = input (bit=1)
0x2b: 00001fff   # Bank B PULLEN — كل الـ pull-enable bits = 1
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Android TV Box — HDMI-TX مش شغال بعد bring-up

#### العنوان
**الـ HDMI output مش بيظهر على T7-based Android TV box بعد أول boot**

#### السياق
شركة بتعمل Android TV box بالـ Amlogic T7 SoC — نفس الـ SoC اللي بيغطيه الملف ده. المنتج في مرحلة bring-up، والـ board بيبوت، والـ kernel بيرفع، بس الشاشة فاضية تماماً. الـ HDMI cable متوصل صح وجرب بشاشات تانية.

#### المشكلة
الـ HDMI transmitter مش بيبعت أي signal. الـ `dmesg` بيظهر الـ HDMI driver اتلود، بس مفيش output.

#### التحليل
الـ HDMI-TX على T7 محتاج:
1. **الـ HPD line** (`hdmitx_hpd_in`) — عشان يحس بالشاشة.
2. **الـ DDC I2C** (`hdmitx_sda_w13` / `hdmitx_sck_w14` أو `hdmitx_sda_w2` / `hdmitx_sck_w3`) — عشان يقرأ EDID من الشاشة.

في الملف بنشوف:
```c
/* Bank W func1 */
static const unsigned int hdmitx_sda_w13_pins[] = { GPIOW_13 };
static const unsigned int hdmitx_sck_w14_pins[] = { GPIOW_14 };
static const unsigned int hdmitx_hpd_in_pins[]  = { GPIOW_15 };

/* Bank W func3 */
static const unsigned int hdmitx_sda_w2_pins[]  = { GPIOW_2 };
static const unsigned int hdmitx_sck_w3_pins[]  = { GPIOW_3 };
```

وفي الـ groups:
```c
static const char * const hdmitx_groups[] = {
    "hdmitx_sda_w13", "hdmitx_sck_w14", "hdmitx_hpd_in",
    "hdmitx_sda_w2",  "hdmitx_sck_w3",
};
```

الـ DT node الخاص بالـ pinctrl مكتوب فيه `hdmitx_sda_w13` و `hdmitx_sck_w14`، بس الـ board فعلياً GPIOW_2 و GPIOW_3 هم الواصلين على الـ DDC lines حسب الـ schematic. يعني الـ DT بيطلب func1 (W13/W14) بس الـ hardware routing بيمشي على W2/W3 اللي هو func3.

لما الـ kernel بيعمل `pinctrl_select_state` هو بيكتب function number 1 في الـ mux register الخاص بـ GPIOW_13 و GPIOW_14، اللي هم مش متوصلين. الـ GPIOW_2 و GPIOW_3 فاضيين أو في mode تاني.

الـ `meson_axg_pmx_ops` بتكتب الـ func field من `meson_pmx_axg_data` في الـ `reg` المحدد في `t7_periphs_pmx_banks`:
```c
BANK_PMX("W", GPIOW_0, GPIOW_16, 0x16, 0),
```

فالكتابة بتحصل على offset صح بس للـ pin الغلط.

#### الحل
تعديل الـ DT لاستخدام الـ pingroup الصح:

```dts
/* قبل */
&hdmitx {
    pinctrl-0 = <&hdmitx_pins>;
    pinctrl-names = "default";
};

&pinctrl_periphs {
    hdmitx_pins: hdmitx-pins {
        mux {
            groups = "hdmitx_sda_w13", "hdmitx_sck_w14", "hdmitx_hpd_in";
            function = "hdmitx";
        };
    };
};

/* بعد — استخدام W2/W3 اللي متوصلين فعلاً */
&pinctrl_periphs {
    hdmitx_pins: hdmitx-pins {
        mux {
            groups = "hdmitx_sda_w2", "hdmitx_sck_w3", "hdmitx_hpd_in";
            function = "hdmitx";
        };
    };
};
```

للتحقق من الـ mux state بعد التعديل:
```bash
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -i "GPIOW_2\|GPIOW_3\|GPIOW_15"
```

#### الدرس المستفاد
الـ T7 بيدي خيارات متعددة للـ HDMI DDC على نفس الـ `hdmitx` function — W2/W3 (func3) وW13/W14 (func1). لازم تتأكد من الـ schematic أي pins متوصلين فعلاً على الـ board قبل ما تكتب الـ DT، وإلا الـ pinctrl بيشتغل صح من ناحية الـ software بس على pins غلط.

---

### السيناريو الثاني: Industrial Gateway — UART Debug مش رادّ بعد حذف JTAG

#### العنوان
**فقدان الـ UART console على industrial gateway بعد تعطيل JTAG في production DT**

#### السياق
شركة بتعمل industrial IoT gateway بالـ T7. في مرحلة development، الـ JTAG مفعّل. في production، المهندس عطّل الـ JTAG nodes وشال الـ pins من الـ DT. فجأة الـ serial console اختفى.

#### المشكلة
الـ UART console على `/dev/ttyAML0` (أو `ttyS0`) بقى مفيش output. الـ board بيبوت لكن ولا حاجة على الـ serial port.

#### التحليل
في الملف بنشوف إن Bank C بيدي pins لأكتر من function:

```c
/* Bank C func1 — SD Card */
static const unsigned int sdcard_d0_pins[] = { GPIOC_0 };
static const unsigned int sdcard_d1_pins[] = { GPIOC_1 };
static const unsigned int sdcard_d2_pins[] = { GPIOC_2 };
static const unsigned int sdcard_d3_pins[] = { GPIOC_3 };

/* Bank C func2 — JTAG B */
static const unsigned int jtag_b_tdo_pins[] = { GPIOC_0 };
static const unsigned int jtag_b_tdi_pins[] = { GPIOC_1 };
static const unsigned int uart_ao_a_rx_c_pins[] = { GPIOC_2 };  /* <-- هنا */
static const unsigned int uart_ao_a_tx_c_pins[] = { GPIOC_3 };  /* <-- وهنا */
static const unsigned int jtag_b_clk_pins[]  = { GPIOC_4 };
static const unsigned int jtag_b_tms_pins[]  = { GPIOC_5 };
```

وفي الـ `uart_ao_a_groups`:
```c
static const char * const uart_ao_a_groups[] = {
    "uart_ao_a_rx_c", "uart_ao_a_tx_c",  /* على GPIOC_2/3 */
    "uart_ao_a_tx_w2", "uart_ao_a_rx_w3",
    ...
    "uart_ao_a_tx_d0", "uart_ao_a_rx_d1",
};
```

الـ UART AO A بيقدر يشتغل على أكتر من bank. المهندس كان مستخدم الـ `uart_ao_a_rx_c` و `uart_ao_a_tx_c` على GPIOC_2/3 اللي هو نفس الـ Bank C اللي فيه JTAG_B. لما شال الـ JTAG node، الـ pinmux state للـ GPIOC_2/3 اتغير أو ماتبقاش configured، فالـ UART مش بيشتغل.

الأسوأ من كده، لو الـ default state للـ pin هو GPIO (function 0) بدل UART، الـ register بيتكتب بـ 0 بدل 2 للـ func، وده معناه مفيش UART output خالص.

#### الحل
استخدم `uart_ao_a_tx_d0` / `uart_ao_a_rx_d1` اللي على Bank D:

```c
/* من الملف — Bank D func1 */
static const unsigned int uart_ao_a_tx_d0_pins[] = { GPIOD_0 };
static const unsigned int uart_ao_a_rx_d1_pins[] = { GPIOD_1 };
```

الـ DT التعديل:

```dts
/* بدل استخدام GPIOC */
&pinctrl_periphs {
    uart_ao_a_default: uart-ao-a-default-pins {
        mux {
            /* استخدام Bank D بدل Bank C اللي فيه JTAG */
            groups = "uart_ao_a_tx_d0", "uart_ao_a_rx_d1";
            function = "uart_ao_a";
        };
    };
};
```

للتأكد من الـ mux register:
```bash
# BANK_PMX لـ D: reg=0x0a
# GPIOD_0 و GPIOD_1 هما أول pins في Bank D
# bits [3:0] للـ GPIOD_0، bits [7:4] للـ GPIOD_1
devmem2 0xFF634028 w   # عنوان تقريبي لـ periphs mux reg 0x0a
```

#### الدرس المستفاد
الـ T7 بيدي multiple routing options لنفس الـ UART — ده ميزة قوية بس ممكن تسبب مشاكل لو مش عارف الـ dependency. دايماً حدد الـ UART console على bank مستقل وبعيد عن الـ JTAG pins، وافتح الـ `pinmux-pins` debugfs قبل ما تشيل أي node من الـ DT.

---

### السيناريو الثالث: Automotive ECU — SPI الـ Flash مش بيتقرأ على الـ Boot

#### العنوان
**الـ NOR Flash على SPI4 مش بيتعرف عليه على T7-based automotive ECU**

#### السياق
شركة automotive بتستخدم T7 SoC في ECU. الـ system بيستخدم NOR flash خارجي متوصل على SPI4. الـ U-Boot بيشوف الـ flash، بس الـ Linux kernel بيفشل في mount الـ root filesystem منه.

#### المشكلة
`dmesg` بيظهر:
```
spi-nor spi4.0: unrecognized JEDEC id bytes: ff ff ff
```
الـ SPI controller موجود، بس الـ data مش بيتنقل صح.

#### التحليل
الـ SPI4 على T7 بيشتغل على Bank Z وده الـ func4:

```c
/* Bank Z func4 */
static const unsigned int spi4_mosi_pins[] = { GPIOZ_0 };
static const unsigned int spi4_miso_pins[] = { GPIOZ_1 };
static const unsigned int spi4_sclk_pins[] = { GPIOZ_2 };
static const unsigned int spi4_ss0_pins[]  = { GPIOZ_3 };
```

وفي الـ groups table:
```c
GROUP(spi4_mosi, 4),
GROUP(spi4_miso, 4),
GROUP(spi4_sclk, 4),
GROUP(spi4_ss0,  4),
```

الـ func=4 يعني الـ `meson_axg_pmx_ops` هيكتب القيمة 4 في الـ mux register.

بس المشكلة الحقيقية: الـ Bank Z نفس الـ pins (GPIOZ_0 إلى GPIOZ_3) بتستخدمها func1 لـ Ethernet (eth_mdio, eth_mdc, eth_rgmii_rx_clk, eth_rx_dv):

```c
/* Bank Z func1 — Ethernet */
static const unsigned int eth_mdio_pins[]        = { GPIOZ_0 };
static const unsigned int eth_mdc_pins[]         = { GPIOZ_1 };
static const unsigned int eth_rgmii_rx_clk_pins[] = { GPIOZ_2 };
static const unsigned int eth_rx_dv_pins[]        = { GPIOZ_3 };
```

الـ Ethernet driver بيـconfigure الـ pins على func1 أولاً لأن الـ eth node بييجي قبل الـ SPI node في الـ DT parsing. الـ SPI4 بعدين بيـrequire نفس الـ pins، بس الـ pinctrl framework بيكتشف الـ conflict ويـfail أو يـignore الـ request.

الدليل: في الـ `t7_periphs_pmx_banks`:
```c
BANK_PMX("Z", GPIOZ_0, GPIOZ_13, 0x05, 0),
```

الـ GPIOZ_0 بيحتاج nybble واحد (4 bits) في reg 0x05. الـ eth بيكتب 1، والـ SPI4 محتاج يكتب 4 — طالما الـ eth node مش disabled في الـ DT، الـ conflict هيفضل.

#### الحل
لازم تـdisable الـ eth node في الـ DT لو الـ board مش فيها Ethernet (الـ ECU ده isolated):

```dts
/* في الـ board DT */
&eth {
    status = "disabled";
};

&spi4 {
    status = "okay";
    pinctrl-0 = <&spi4_pins>;
    pinctrl-names = "default";

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <50000000>;
    };
};

&pinctrl_periphs {
    spi4_pins: spi4-pins {
        mux {
            groups = "spi4_mosi", "spi4_miso",
                     "spi4_sclk", "spi4_ss0";
            function = "spi4";
        };
    };
};
```

للتحقق من pin ownership:
```bash
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -E "GPIOZ_[0-3]"
```

#### الدرس المستفاد
الـ T7 بيستخدم Bank Z لـ Ethernet RGMII full 14 pins. أول 4 pins (GPIOZ_0-3) بيتشاركوا مع SPI4 و ISO7816 و TSIN. دايماً افحص الـ overlap في `t7_periphs_groups[]` قبل ما تحدد الـ SoC للمشروع، وحدد كل الـ peripherals غير المستخدمة كـ `disabled` في الـ DT.

---

### السيناريو الرابع: IoT Sensor Board — I2C بيديك Bus Errors متقطعة

#### العنوان
**الـ I2C2 بيدي `EREMOTEIO` errors متقطعة على T7 IoT sensor board**

#### السياق
شركة بتعمل sensor hub بالـ T7. الـ board فيها 3 حساسات على I2C2 bus. الـ driver بيشتغل ساعات وبعدين يقع فجأة. الـ errors مش reproducible بانتظام.

#### المشكلة
الـ `dmesg` بيظهر:
```
i2c i2c-2: i2c_transfer failed: -121
i2c i2c-2: ...adapter timeout
```
الـ logic analyzer بيظهر الـ SCL بيتعلق low في بعض الـ transactions.

#### التحليل
الـ I2C2 على T7 عنده 3 routing خيارات في الملف:

```c
/* على Bank X — func1 */
static const unsigned int i2c2_sda_x_pins[] = { GPIOX_17 };
static const unsigned int i2c2_sck_x_pins[] = { GPIOX_18 };

/* على Bank T — func2 */
static const unsigned int i2c2_sck_t_pins[] = { GPIOT_22 };
static const unsigned int i2c2_sda_t_pins[] = { GPIOT_23 };

/* على Bank M — func2 */
static const unsigned int i2c2_sda_m_pins[] = { GPIOM_12 };
static const unsigned int i2c2_sck_m_pins[] = { GPIOM_13 };
```

وفي الـ i2c2_groups:
```c
static const char * const i2c2_groups[] = {
    "i2c2_sda_x", "i2c2_sck_x",
    "i2c2_sda_t", "i2c2_sck_t",
    "i2c2_sda_m", "i2c2_sck_m",
};
```

الـ board بتستخدم GPIOT_22/23 (Bank T). المشكلة إن `spi0_ss1` و `spi0_ss2` على نفس الـ pins:

```c
/* Bank T func1 — SPI0 */
static const unsigned int spi0_ss1_pins[] = { GPIOT_22 };
static const unsigned int spi0_ss2_pins[] = { GPIOT_23 };
```

الـ SPI0 controller كان مـconfigure الـ pins على func1 (SPI) وبعدين تتـrelease لما الـ SPI transaction تخلص. بس الـ I2C2 كان مـconfigure على func2 (I2C). في حالات race، الـ SPI0 driver كان بيرجع يطلب الـ pins في وسط I2C transaction — وده بيـcause الـ SCL يتعلق.

ده ممكن يحصل لو الـ SPI0 و I2C2 في نفس الـ DT state بدون pin conflict detection صريح، خصوصاً لو كل واحد فيهم عنده `default` و `sleep` states مختلفة.

#### الحل
استخدم I2C2 على Bank X أو Bank M بعيداً عن Bank T اللي فيه SPI0:

```dts
&pinctrl_periphs {
    /* استخدام Bank X بدل Bank T لتجنب conflict مع SPI0 */
    i2c2_pins: i2c2-pins {
        mux {
            groups = "i2c2_sda_x", "i2c2_sck_x";
            function = "i2c2";
            bias-pull-up;
        };
    };
};

&i2c2 {
    pinctrl-0 = <&i2c2_pins>;
    pinctrl-names = "default";
    status = "okay";
    clock-frequency = <400000>;
};
```

وللتحقق من مفيش pin shared:
```bash
# اطبع كل الـ pins اللي multifunction
cat /sys/kernel/debug/pinctrl/*/pins | grep -v "GPIO only"

# اتأكد إن GPIOT_22/23 مش assigned لأكتر من driver
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -E "GPIOT_2[23]"
```

#### الدرس المستفاد
وجود أكتر من routing للـ I2C مش بس flexibility — هو أيضاً فرصة لـ silent conflicts مع peripherals تانية على نفس الـ bank. الـ T7 بيدي 3 خيارات لـ I2C2، اختار الـ bank اللي فيه أقل overlap مع باقي الـ active peripherals. استخدم دايماً `bias-pull-up` لأن الـ I2C بيحتاجه، والـ pinctrl driver لـ T7 بيدعمه عبر الـ `MESON_REG_PULLEN` registers.

---

### السيناريو الخامس: Custom Board Bring-up — eMMC Timeout عند أول Boot

#### العنوان
**الـ eMMC مش بيـprobe على T7 custom board — kernel panic عند mount root**

#### السياق
مهندس bring-up بيعمل custom board بالـ T7. الـ board الجديدة هي revision ثانية من board قديمة. الـ DT منقولة من الـ reference design مع تعديلات بسيطة. الـ kernel بيقع في:
```
mmc0: error -110 whilst initialising MMC card
Kernel panic - not syncing: VFS: Unable to mount root fs
```

#### المشكلة
الـ eMMC controller موجود ومش بيتقرأ على الـ bus.

#### التحليل
الـ eMMC على T7 على Bank B:

```c
/* Bank B func1 — eMMC */
static const unsigned int emmc_nand_d0_pins[]  = { GPIOB_0 };
static const unsigned int emmc_nand_d1_pins[]  = { GPIOB_1 };
static const unsigned int emmc_nand_d2_pins[]  = { GPIOB_2 };
static const unsigned int emmc_nand_d3_pins[]  = { GPIOB_3 };
static const unsigned int emmc_nand_d4_pins[]  = { GPIOB_4 };
static const unsigned int emmc_nand_d5_pins[]  = { GPIOB_5 };
static const unsigned int emmc_nand_d6_pins[]  = { GPIOB_6 };
static const unsigned int emmc_nand_d7_pins[]  = { GPIOB_7 };
static const unsigned int emmc_clk_pins[]      = { GPIOB_8 };
static const unsigned int emmc_cmd_pins[]      = { GPIOB_10 };
static const unsigned int emmc_nand_ds_pins[]  = { GPIOB_11 };
```

لاحظ إن GPIOB_9 مش في أي func — مش مستخدم للـ eMMC. الـ emmc groups:

```c
static const char * const emmc_groups[] = {
    "emmc_nand_d0", "emmc_nand_d1", "emmc_nand_d2", "emmc_nand_d3",
    "emmc_nand_d4", "emmc_nand_d5", "emmc_nand_d6", "emmc_nand_d7",
    "emmc_clk", "emmc_cmd", "emmc_nand_ds",
};
```

الـ revision ثانية من الـ board غيّرت الـ drive strength لأن الـ board layout اتغير وطول الـ traces زاد. الـ BANK_DS للـ Bank B:

```c
BANK_DS("B", GPIOB_0, GPIOB_12, 0, 12,
    0x2b, 0, 0x2c, 0, 0x2a, 0, 0x29, 0, 0x28, 0, 0x2f, 0),
```

الـ `MESON_REG_DS` هو `0x2f` بـ offset 0. يعني drive strength register موجود. الـ default drive strength (500µA) مش كافي لـ eMMC data lines على traces أطول.

المشكلة التانية المحتملة: الـ DT كان بيحدد فقط 8 data pins بدون الـ `emmc_nand_ds` (Data Strobe):

```dts
/* DT ناقص */
mux {
    groups = "emmc_nand_d0", "emmc_nand_d1", "emmc_nand_d2",
             "emmc_nand_d3", "emmc_nand_d4", "emmc_nand_d5",
             "emmc_nand_d6", "emmc_nand_d7",
             "emmc_clk", "emmc_cmd";
    /* ناسي emmc_nand_ds ! */
    function = "emmc";
};
```

الـ Data Strobe مهم جداً في eMMC HS400 mode. بدونه الـ timing بيتعبط.

#### الحل

**أولاً:** أضف الـ `emmc_nand_ds` للـ DT:

```dts
&pinctrl_periphs {
    emmc_pins: emmc-pins {
        mux {
            groups = "emmc_nand_d0", "emmc_nand_d1", "emmc_nand_d2",
                     "emmc_nand_d3", "emmc_nand_d4", "emmc_nand_d5",
                     "emmc_nand_d6", "emmc_nand_d7",
                     "emmc_clk", "emmc_cmd", "emmc_nand_ds";
            function = "emmc";
            /* رفع الـ drive strength لـ traces أطول */
            drive-strength-microamp = <4000>;
        };
    };

    emmc_clk_gate_pins: emmc-clk-gate-pins {
        mux {
            groups = "emmc_clk";
            function = "gpio_periphs";
            bias-pull-down;
        };
    };
};
```

**ثانياً:** للتحقق من الـ drive strength register:
```bash
# DS register لـ Bank B هو 0x2f (offset 0)
# كل pin بياخد 2 bits
# القيم: 00=500µA, 01=2500µA, 10=3000µA, 11=4000µA
cat /sys/kernel/debug/pinctrl/*/pinconf-pins | grep GPIOB
```

**ثالثاً:** قلل الـ eMMC speed مؤقتاً للتشخيص:
```dts
&mmc0 {
    max-frequency = <52000000>;  /* بدل 200MHz */
    mmc-ddr-1_8v;
    /* شيل HS400 مؤقتاً */
};
```

#### الدرس المستفاد
الـ T7 بيدعم `BANK_DS` (Drive Strength) لكل pin عبر `MESON_REG_DS` في الـ `meson_bank` struct. الـ default drive strength ممكن يكون كافي على reference board بس مش كافي على custom board بـ traces مختلفة. دايماً افحص الـ eMMC timing على الـ oscilloscope بعد board spin، وتأكد إن `emmc_nand_ds` موجود في الـ DT خصوصاً لو بتستخدم HS200/HS400 mode.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأساسي لمتابعة تطور الـ kernel وكل patch series مهمة.

| المقالة | الوصف |
|---|---|
| [Pinctrl driver for Amlogic Meson SoCs](https://lwn.net/Articles/616225/) | أول patch series لـ pinctrl driver على Meson8 — الأساس اللي بنت عليه كل الـ drivers اللاحقة |
| [Amlogic Meson pinctrl driver](https://lwn.net/Articles/620822/) | مراجعة ونقاش الـ patch series الخاصة بـ Meson pinctrl driver |
| [Amlogic Meson8b pinctrl driver support](https://lwn.net/Articles/638594/) | إضافة دعم Meson8b فوق البنية التحتية المشتركة |
| [pinctrl: meson-a1: add pinctrl driver](https://lwn.net/Articles/804174/) | driver خاص بـ A1 SoC مع تعديلات على الـ pinctrl framework نفسه |
| [Add pinctrl driver support for Amlogic T7 SoCs](https://lwn.net/Articles/945285/) | الـ patch series بالضبط اللي أضافت دعم T7 (A311D2) — هو الـ driver اللي بندرسه |
| [Add support for Amlogic S7/S7D/S6 pinctrl](https://lwn.net/Articles/1022709/) | آخر إضافة للعائلة، تُظهر تطور الـ driver بعد T7 |
| [The pin control subsystem](https://lwn.net/Articles/468759/) | مقالة أساسية تشرح فلسفة الـ pinctrl subsystem من البداية |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | النسخة السابعة من الـ patch series اللي أنشأت الـ subsystem كله |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | أول نسخة من الـ documentation الرسمية على LWN |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/467269/) | إضافة الـ generic pin config interface للـ subsystem |
| [drivers/pinctrl: Add the concept of an "init" state](https://lwn.net/Articles/615322/) | إضافة مفهوم الـ "init" state في الـ pinctrl subsystem |

---

### التوثيق الرسمي في kernel

**الـ Documentation/** داخل الـ kernel source هو المرجع الأكثر دقة وتحديثاً.

```
Documentation/driver-api/pin-control.rst     ← التوثيق الرئيسي للـ pinctrl subsystem
Documentation/devicetree/bindings/pinctrl/   ← كل bindings الخاصة بـ pinctrl بصيغة YAML
Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl.yaml ← binding الخاص بـ Amlogic Meson
```

**الـ kernel.org documentation:**
- [PINCTRL subsystem — kernel.org](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html)
- [kernel.org: Documentation/pinctrl.txt](https://www.kernel.org/doc/Documentation/pinctrl.txt)

---

### ملفات الـ source المرتبطة مباشرة

الملفات دي في داخل الـ kernel source tree وأساسية لفهم الـ driver:

```
drivers/pinctrl/meson/pinctrl-amlogic-t7.c      ← الملف اللي بندرسه
drivers/pinctrl/meson/pinctrl-meson.c            ← الـ core logic المشترك
drivers/pinctrl/meson/pinctrl-meson.h            ← الـ structs والـ macros الأساسية
drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c   ← الـ PMX logic اللي بيستخدمه T7
drivers/pinctrl/meson/pinctrl-meson-axg-pmx.h   ← header الخاص بـ AXG PMX
include/dt-bindings/gpio/amlogic,t7-periphs-pinctrl.h ← تعريفات الـ GPIO pins
```

---

### Patchwork ونقاشات الـ mailing list

**الـ Patchwork** هو مكان تتبع كل الـ patches اللي اتبعتت على الـ mailing lists.

| الـ Patch / النقاش | الوصف |
|---|---|
| [PATCH 1/3: dt-bindings: pinctrl: Add compatibles for Amlogic T7 SoCs](https://lkml.rescloud.iu.edu/2309.1/07306.html) | الـ patch الأول من الـ series الخاص بـ T7 — الـ DT bindings |
| [Re: PATCH 2/3 pinctrl: Add driver support for Amlogic T7 SoCs](https://lore.kernel.org/linux-kernel/3f330773-ca80-4d0a-970c-806b0099b87e@linaro.org/) | review من neil.armstrong على الـ T7 driver |
| [v3: pinctrl: add compatible for Amlogic Meson A1 pin controller — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1571050492-6598-2-git-send-email-qianggui.song@amlogic.com/) | patch A1 — نموذج لكيفية إضافة SoC جديد |
| [v4: pinctrl: meson: add a new callback for SoCs fixup — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1572004167-24150-3-git-send-email-qianggui.song@amlogic.com/) | إضافة الـ SoC fixup callback — التصميم اللي اعتمد عليه T7 |
| [v6: pinctrl: meson: add support for GPIO interrupts — Patchwork](https://patchwork.kernel.org/project/linux-amlogic/patch/a86d38af-9103-7557-9986-2dc187815569@gmail.com/) | إضافة GPIO interrupt support |
| [PATCH v7: dt-bindings: pinctrl: Convert Amlogic Meson pinctrl binding](https://lists.infradead.org/pipermail/linux-arm-kernel/2023-March/821938.html) | تحويل الـ binding لـ YAML format — مارس 2023 |
| [PATCH 1/3: pinctrl: add driver for Amlogic Meson SoCs — LKML](https://lkml.iu.edu/hypermail/linux/kernel/1410.0/04163.html) | أقدم patch للـ Meson pinctrl — أكتوبر 2014 |

---

### موارد linux-meson.com

**الـ [linux-meson.com](https://linux-meson.com/)** هو الموقع غير الرسمي المخصص لمتابعة الـ mainlining الخاص بـ Amlogic Meson:

| الصفحة | الوصف |
|---|---|
| [Linux for Amlogic — الصفحة الرئيسية](https://linux-meson.com/) | نقطة البداية لكل معلومات عن دعم Amlogic في الـ mainline kernel |
| [Kernel mainlining progress](https://linux-meson.com/mainlining.html) | تتبع تفصيلي لكل feature اتضافت أو في طريقها — بما فيها pinctrl |
| [Hardware Support](https://linux-meson.com/hardware.html) | قائمة الـ SoCs والـ boards المدعومة وحالة كل feature |

---

### kernelnewbies.org

الصفحات دي بتوثق الـ pinctrl changes اللي اتضافت في كل kernel release:

| الصفحة | الوصف |
|---|---|
| [Linux_6.15 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.15) | يتضمن إضافة Amlogic pinctrl driver جديد ودعم SG2042 |
| [Linux_6.17 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.17) | Eswin eic7700 pinctrl ودعم Milos (sm7635) |
| [Linux_6.18 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.18) | إضافة bcm2712 pin control driver |
| [LinuxChanges — kernelnewbies.org](https://kernelnewbies.org/LinuxChanges) | صفحة التغييرات الكاملة عبر كل الإصدارات |

---

### elinux.org

| الصفحة | الوصف |
|---|---|
| [Pin Control/GPIO Update — elinux.org (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) | عرض تقديمي يشرح تحديثات الـ pinctrl وعلاقته بـ GPIO |
| [EBC Device Trees — elinux.org](https://elinux.org/EBC_Device_Trees) | أمثلة عملية على كيفية كتابة الـ device tree مع pinctrl |
| [Tests: i2c-demux-pinctrl — elinux.org](https://www.elinux.org/Tests:i2c-demux-pinctrl) | مثال على استخدام pinctrl مع الـ i2c-demux |

---

### كتب مقترحة

#### Linux Device Drivers (LDD3)
**المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل 9:** Communicating with Hardware — يشرح كيفية الوصول للـ registers والـ memory-mapped I/O
- **الفصل 14:** The Linux Device Model — يشرح الـ platform devices و device tree
- متاح مجاناً على: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development (Robert Love)
- **الفصل 17:** Devices and Modules — يغطي driver model والـ platform bus
- الأساس النظري للـ kernel internals قبل ما تغوص في pinctrl

#### Embedded Linux Primer (Christopher Hallinan)
- **الفصل 15:** Device Drivers — يتناول GPIO و pin muxing في سياق embedded systems
- أمثلة عملية على الـ DTS configuration

#### Professional Linux Kernel Architecture (Wolfgang Mauerer)
- تغطية عميقة للـ bus subsystems والـ device model اللي يعتمد عليه pinctrl

---

### Embedded.com

| المقالة | الوصف |
|---|---|
| [Linux device driver development: The pin control subsystem](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) | شرح عملي للـ pinctrl subsystem من منظور embedded development |

---

### مصطلحات البحث

لو عايز تلاقي معلومات إضافية، استخدم الـ search terms دي:

```
# بحث عام في الـ pinctrl subsystem
"linux kernel pinctrl subsystem" site:lwn.net
"pinctrl meson amlogic" site:patchwork.kernel.org
"pinctrl-meson" site:lore.kernel.org

# بحث خاص بـ T7
"pinctrl amlogic t7" OR "pinctrl-amlogic-t7"
"A311D2 pinctrl" OR "amlogic t7 pin controller"
"huqiang.qin amlogic pinctrl"

# بحث في الـ git history
git log --oneline --all -- drivers/pinctrl/meson/
git log --grep="amlogic.*t7.*pinctrl" --oneline

# بحث في الـ mailing list
lore.kernel.org/linux-gpio/?q=amlogic+t7+pinctrl
lore.kernel.org/linux-amlogic/?q=pinctrl+t7

# بحث في الـ device tree bindings
"amlogic,meson-pinctrl" site:kernel.org
"amlogic,t7-periphs-pinctrl" site:github.com
```

---

### روابط الـ kernel source على GitHub/Bootlin

| الرابط | الوصف |
|---|---|
| [pinctrl/meson على Elixir (bootlin.com)](https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl/meson) | Cross-reference لكل ملفات الـ meson pinctrl مع links بين الـ functions |
| [pinctrl-amlogic-t7.c على Elixir](https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl/meson/pinctrl-amlogic-t7.c) | قراءة الـ driver مباشرة مع الـ cross-references |
| [amlogic-t7-a311d2-khadas-vim4.dts على GitHub](https://github.com/torvalds/linux/blob/master/arch/arm64/boot/dts/amlogic/amlogic-t7-a311d2-khadas-vim4.dts) | مثال حقيقي لاستخدام T7 pinctrl في الـ device tree |
## Phase 8: Writing simple module

### الفكرة العامة

**الـ** `meson_pinctrl_probe` هي الـ function المسؤولة عن تهيئة الـ pin controller لأي SoC من Amlogic بما فيهم T7. بنحط عليها kprobe عشان نشوف كل مرة بيتعمل فيها probe لـ pinctrl device — مفيد جداً لتشخيص مشاكل الـ boot أو تتبع ترتيب تهيئة الـ peripherals.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe on meson_pinctrl_probe — traces every Amlogic pinctrl device
 * as it gets initialised during boot or hotplug.
 */

/* ① headers */
#include <linux/module.h>       /* MODULE_* macros, module_init/exit        */
#include <linux/kernel.h>       /* pr_info, KERN_INFO                        */
#include <linux/kprobes.h>      /* struct kprobe, register/unregister_kprobe */
#include <linux/platform_device.h> /* struct platform_device, .name field   */

/* ② the probe handler — called just before meson_pinctrl_probe executes */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * First argument of meson_pinctrl_probe(struct platform_device *pdev)
     * lives in the first integer/pointer argument register.
     * On arm64: regs->regs[0]; on x86-64: regs->di
     */
#if defined(CONFIG_ARM64)
    struct platform_device *pdev =
        (struct platform_device *)regs->regs[0];
#elif defined(CONFIG_X86_64)
    struct platform_device *pdev =
        (struct platform_device *)regs->di;
#else
    /* generic fallback — may not work on all arches */
    struct platform_device *pdev = NULL;
#endif

    if (pdev)
        pr_info("[t7_probe_trace] meson_pinctrl_probe called: "
                "device='%s' id=%d\n",
                pdev->name ? pdev->name : "<null>",
                pdev->id);
    else
        pr_info("[t7_probe_trace] meson_pinctrl_probe called "
                "(pdev unavailable on this arch)\n");

    return 0; /* 0 = let the original function run normally */
}

/* ③ kprobe struct — points to the symbol we want to intercept */
static struct kprobe kp = {
    .symbol_name = "meson_pinctrl_probe", /* exported in pinctrl-meson.c */
    .pre_handler = handler_pre,
};

/* ④ module init */
static int __init t7_probe_trace_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[t7_probe_trace] register_kprobe failed: %d\n", ret);
        return ret;
    }
    pr_info("[t7_probe_trace] kprobe planted on meson_pinctrl_probe "
            "at %p\n", kp.addr);
    return 0;
}

/* ⑤ module exit */
static void __exit t7_probe_trace_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[t7_probe_trace] kprobe removed from meson_pinctrl_probe\n");
}

module_init(t7_probe_trace_init);
module_exit(t7_probe_trace_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe tracer for Amlogic T7 pinctrl probe function");
```

---

### شرح كل جزء

#### ① الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | ضروري لأي kernel module — بيحتوي على الـ macros الأساسية |
| `linux/kernel.h` | عشان `pr_info` / `pr_err` |
| `linux/kprobes.h` | بيعرّف `struct kprobe` ووظائف الـ register/unregister |
| `linux/platform_device.h` | عشان نقدر نعمل cast صح لـ argument الأول ونوصل لـ `pdev->name` |

#### ② الـ `handler_pre`

**الـ** `pre_handler` بيتشغّل قبل ما الـ function الأصلية تأخذ الـ CPU — ده بيخلينا نشوف الـ arguments قبل ما تتغيّر. بنقرأ الـ `platform_device` من registers الـ calling convention حسب الـ architecture (arm64 أو x86-64) وبنطبع اسم الـ device وكمان الـ id بتاعه عشان نعرف إيه اللي بيتشغّل.

#### ③ `struct kprobe`

**الـ** `symbol_name` بيقول للـ kernel يدور على الـ address بتاع `meson_pinctrl_probe` في الـ symbol table — مش محتاجين hardcode address. الـ `pre_handler` هو اللي هيتنفّذ عند الـ hit.

#### ④ `module_init`

**الـ** `register_kprobe` بتزرع breakpoint في الـ kernel code بشكل آمن — لو فشلت (مثلاً الـ symbol مش موجود أو الـ kprobes مش مفعّل في الـ config) بنرجع الـ error فوراً.

#### ⑤ `module_exit`

**الـ** `unregister_kprobe` ضروري جداً عشان لو نزّلنا الـ module والـ kprobe لسّه موجودة، أي استدعاء لـ `meson_pinctrl_probe` بعدها هيحاول ينفّذ `handler_pre` في memory مش موجودة — ده kernel panic مضمون.

---

### Makefile للتجربة

```makefile
obj-m += t7_probe_trace.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

### تشغيل وتتبع النتيجة

```bash
# تحميل الـ module
sudo insmod t7_probe_trace.ko

# تابع الـ log
sudo dmesg | grep t7_probe_trace

# مثال للـ output عند وجود Amlogic T7 board:
# [t7_probe_trace] kprobe planted on meson_pinctrl_probe at ffffffffc0123456
# [t7_probe_trace] meson_pinctrl_probe called: device='amlogic-t7-pinctrl' id=-1

# إزالة الـ module
sudo rmmod t7_probe_trace
```

> **ملاحظة:** على x86 (مثلاً QEMU) مش هيظهر الـ device لأن الـ T7 hardware مش موجود، لكن الـ kprobe هيتسجّل بنجاح وهيطبع عند أي استدعاء لـ `meson_pinctrl_probe` لو كانت مكمّلة في الـ kernel.
