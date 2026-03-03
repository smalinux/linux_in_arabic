## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملفان ينتميان لـ **ARM/Amlogic Meson SoC** subsystem، وتحديداً لـ **Pinctrl/Pinmux** framework الخاص بـ SoCs من Amlogic. الـ maintainer entry في MAINTAINERS يغطي كل `drivers/pinctrl/meson/` تحت مظلة `ARM/Amlogic Meson SoC`.

---

### القصة من الأول — ليه أصلاً في حاجة اسمها Pinmux؟

تخيل إنك بتبني مدينة صغيرة، وعندك شوارع محدودة. كل شارع ممكن يوصّل لأكتر من حي بس مش في نفس الوقت. اللي بيحدد كل شارع يروح فين هو "مفتاح" في لوحة تحكم مركزية.

الـ **SoC** (System-on-Chip) زي Amlogic Meson-AXG هو chip واحدة فيها كل حاجة: CPU، GPU، USB، UART، I2C، SPI، وغيرهم. بس المشكلة إن الـ chip فيها pins (الأرجل اللي بتتوصل بيها بالعالم الخارجي) أقل بكتير من الـ functions اللي محتاجها. الحل؟ كل pin ممكن يشتغل بأكتر من طريقة — يعني نفس الـ pin الجسدي ممكن يكون:
- GPIO عادي (تشغّل LED مثلاً)
- خط UART TX
- خط I2C SDA
- إشارة PWM

ده اللي بيتسمى **Pinmux** — إنت بتختار "وظيفة" الـ pin من قايمة.

---

### الفرق بين الجيل الأول والجيل التاني — اللي الملفين دول بيحكوه

#### الجيل الأول — `pinctrl-meson8-pmx`

الـ **Meson8** (SoCs القديمة زي S905) كانت بتستخدم نظام بسيط جداً: كل function group عندها **bit واحد** في register محدد. عايز تفعّل I2C؟ اضرب الـ bit. عايز توقفه؟ امسح الـ bit. بس المشكلة إن كل pin ممكن يبقى في أكتر من group، فلازم لما تختار function تـ**disable** كل الـ groups التانية اللي بتستخدم نفس الـ pin أول.

```
Pin 5 → Group A (UART TX) → bit 3 في Register 2
Pin 5 → Group B (I2C SDA) → bit 7 في Register 1
لما تختار Group A، الكود يمسح bit 7 تلقائياً عشان ميتعارضوش
```

#### الجيل التاني — `pinctrl-meson-axg-pmx` (ده الملف اللي بندرسه)

الـ **AXG** وما بعده جابوا تصميم أذكى بكتير. بدل ما كل function تبقى لها bit منفصل، كل pin عنده **حقل 4 bits متتالية** في الـ register. القيمة في الـ 4 bits دي بتحدد الـ function:
- القيمة `0` = GPIO mode دايماً
- القيمة `1` = Function 1 (مثلاً UART)
- القيمة `2` = Function 2 (مثلاً I2C)
- وهكذا لـ 15 function مختلفة لنفس الـ pin

```
Pin 5 → bits [19:16] في Register X
القيمة 0000 → GPIO
القيمة 0001 → UART TX
القيمة 0010 → I2C SDA
القيمة 0011 → PWM
```

الجمال هنا إنك **مش محتاج تـdisable حاجة تانية** — بمجرد ما تكتب القيمة الجديدة في الـ 4 bits، الـ function القديمة اتلغت تلقائياً. أبسط، أسرع، وأقل أخطاء.

---

### الصورة الكبيرة — إزاي الكود بيشتغل

```
Linux Kernel
    │
    ▼
pinctrl core (drivers/pinctrl/core.c)
    │   يعرف الـ framework العام
    │
    ▼
pinctrl-meson.c  ←── الـ core driver للـ Meson family
    │   يعمل probe، يسجّل الـ controller، يتعامل مع GPIO و pull-up/pull-down
    │
    ├──► pinctrl-meson8-pmx.c    (الجيل الأول: 1-bit per function)
    │
    └──► pinctrl-meson-axg-pmx.c (الجيل التاني: 4-bit field per pin)  ← نحن هنا
              │
              ▼
         regmap → يكتب في hardware registers مباشرةً
```

الـ `pinctrl-meson-axg-pmx.c` بيقدم لـ pinctrl core الـ **`pinmux_ops`** — مجموعة الـ callbacks اللي بيناديها الـ framework لما أي driver في الـ kernel يقول "أنا محتاج الـ UART على الـ pins دي".

---

### تفاصيل الكود بالبساطة

#### الـ `meson_pmx_bank` — خريطة الـ pins

```c
struct meson_pmx_bank {
    const char *name;     // اسم الـ bank (مثلاً "GPIOX")
    unsigned int first;   // أول pin رقمه
    unsigned int last;    // آخر pin رقمه
    unsigned int reg;     // أول register للـ bank ده
    unsigned int offset;  // offset بالـ bits داخل الـ register
};
```

الـ pins مش كلها في مكان واحد — بتتقسم لـ **banks**. كل bank عبارة عن مجموعة pins متتالية ليها registers خاصة بيها.

#### الماكرو `BANK_PMX`

```c
#define BANK_PMX(n, f, l, r, o)  { .name=n, .first=f, .last=l, .reg=r, .offset=o }
```

بيتستخدم في ملفات الـ SoC المحددة (زي `pinctrl-meson-axg.c`) عشان يعرّف الـ banks بطريقة مختصرة.

#### حساب موقع الـ Pin في الـ Register

```c
// كل pin ليه 4 bits → shift = (pin - first_pin_in_bank) * 4
shift = pin - bank->first;
*reg    = bank->reg + (bank->offset + shift*4) / 32;  // رقم الـ register
*offset = (bank->offset + shift*4) % 32;               // موقعه جوّاه
```

يعني لو الـ pin التاسع في bank بيبدأ من bit 0 في reg 0:
- الـ pin التاسع → shift=8 → bit position = 32 → ده **register 1** bit 0

#### العملية الأساسية — `meson_axg_pmx_update_function`

```c
// اكتب قيمة الـ function (0-15) في الـ 4 bits الخاصة بالـ pin
regmap_update_bits(pc->reg_mux,
    reg << 2,           // عنوان الـ register (× 4 لأن byte addressing)
    0xf << offset,      // mask الـ 4 bits
    (func & 0xf) << offset);  // القيمة الجديدة
```

#### الـ `meson_axg_pmx_ops` — الواجهة للـ kernel

```c
const struct pinmux_ops meson_axg_pmx_ops = {
    .set_mux             = meson_axg_pmx_set_mux,      // اختار function لـ group
    .get_functions_count = meson_pmx_get_funcs_count,  // كام function عندنا؟
    .get_function_name   = meson_pmx_get_func_name,    // إيه اسم كل function؟
    .get_function_groups = meson_pmx_get_groups,       // الـ groups اللي فيها
    .gpio_request_enable = meson_axg_pmx_request_gpio, // حوّل الـ pin لـ GPIO
};
```

لما UART driver يقول "أنا محتاج pins"، الـ pinctrl core بينادي `set_mux` اللي بيكتب الـ function number في كل pin في الـ group.

---

### الـ `GROUP` و `GPIO_GROUP` Macros — إزاي الـ SoC drivers بتستخدمهم

```c
// في ملف pinctrl-meson-axg.c مثلاً:
GROUP(uart_a_tx, 1)   // uart_a_tx_pins[] + function value = 1
GPIO_GROUP(GPIOX_0)   // GPIOX_0 كـ pin واحد + function value = 0
```

الماكرو `GROUP` بيبني `meson_pmx_group` struct كاملة من اسم الـ group واسم array الـ pins والـ function number — كل ده في سطر واحد.

---

### الملفات المهمة اللي المطوّر المبتدئ لازم يعرفها

| الملف | الدور |
|---|---|
| `drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c` | **الملف الأساسي** — الجيل التاني من الـ pmx |
| `drivers/pinctrl/meson/pinctrl-meson-axg-pmx.h` | الـ structs والـ macros للجيل التاني |
| `drivers/pinctrl/meson/pinctrl-meson.h` | الـ core structs المشتركة بين كل الـ Meson drivers |
| `drivers/pinctrl/meson/pinctrl-meson.c` | الـ core driver: probe، GPIO، pull، direction |
| `drivers/pinctrl/meson/pinctrl-meson-axg.c` | تعريف كل pins وfunctions وbanks الخاصة بـ AXG SoC |
| `drivers/pinctrl/meson/pinctrl-meson-g12a.c` | نفس الفكرة لـ G12A SoC (يستخدم نفس الـ pmx) |
| `drivers/pinctrl/meson/pinctrl-meson8-pmx.c` | الجيل الأول من الـ pmx (مقارنة مهمة) |
| `drivers/pinctrl/meson/pinctrl-meson8-pmx.h` | الـ structs للجيل الأول |
| `include/linux/pinctrl/pinmux.h` | تعريف `struct pinmux_ops` في الـ kernel |
| `drivers/pinctrl/core.c` | الـ pinctrl framework نفسه |
| `arch/arm64/boot/dts/amlogic/` | الـ Device Tree files اللي بتحدد الـ pin configurations |
## Phase 2: شرح الـ Pinctrl / Pinmux Framework

### المشكلة اللي بيحلها الـ Subsystem ده

في أي SoC حديث زي Amlogic Meson-AXG، كل **pin** جسمانياً على الـ chip ممكن يشتغل بأكتر من وظيفة. مثلاً:

- **GPIOA_0** ممكن يبقى: GPIO عادي، أو TX لـ UART، أو SDA لـ I2C، أو CLK لـ SPI
- **GPIOA_7** ممكن يبقى: GPIO، أو PWM channel، أو شيء تاني

لو كل driver كتب code خاص بيه يعمل `readl/writel` على registers الـ mux بشكل direct، هيحصل:

1. **تعارض**: driver اتنين بيكتبوا على نفس الـ register في نفس الوقت
2. **تكرار**: كل driver بيعيد نفس logic إدارة الـ bits
3. **عدم قابلية للنقل**: code مش reusable بين SoCs مختلفة
4. **Device Tree مش مفهوم**: مفيش طريقة موحدة تعرف في الـ DT إن الـ UART يستخدم pins معينة

**الـ Pinctrl Framework** اتصنع عشان يحل كل ده من خلال abstraction layer مركزي بين الـ hardware والـ drivers.

---

### الحل اللي بيقدمه الـ Kernel

الـ kernel بيعمل **subsystem مركزي** اسمه **pinctrl** بيمسك كل العمليات دي:

| الوظيفة | المعنى |
|---------|--------|
| **Pin Enumeration** | تسجيل كل الـ pins بأرقامها وأسمائها |
| **Pin Grouping** | تجميع الـ pins اللي بتعمل مع بعض (مثلاً: uart0_tx + uart0_rx = group) |
| **Function Mapping** | ربط كل function (uart0, i2c1, spi2...) بالـ groups المتاحة لها |
| **Mux Control** | تغيير الـ register bits الفعلية لتفعيل function معينة |
| **Conflict Detection** | منع اتنين drivers يستخدموا نفس الـ pin في نفس الوقت |
| **GPIO Integration** | لما pin مفيش حاجة تاخده، يرجع GPIO تلقائي |

---

### الـ Analogy: سنترالة تليفونات

تخيل **سنترالة تليفونات** في شركة كبيرة:

```
الـ Analogy                          ←→   الـ Kernel Concept
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
الخط الجسماني (سلك نحاسي)           ←→   الـ physical pin على الـ chip
السنترالة نفسها (PBX)               ←→   الـ pinctrl driver (meson-axg-pmx)
لوح التوصيلات (patch panel)         ←→   الـ mux registers في الـ hardware
"ربط الخط ده بقسم المحاسبة"         ←→   set_mux(func=uart0, group=uart0_pins)
قاموس "مين بيستخدم أنهي خط"         ←→   الـ pin_desc + ownership في الـ core
طلب خط جديد                        ←→   gpio_request_enable()
تعارض خطين على نفس السلك           ←→   -EBUSY من الـ pinctrl core
```

**الفرق المهم**: السنترالة نفسها (الـ PBX) هي الـ `pinctrl_dev` المسجل، اللوح اللي فيه الـ patch wires هو الـ hardware registers، وإنت كـ admin بتقول للسنترالة "وصّل الخط ده للقسم ده" — مش بتمس الأسلاك بإيدك.

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Device Tree (DTS)                            │
│   &uart0 { pinctrl-names = "default"; pinctrl-0 = <&uart0_pins>; }  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ (parsed at probe time)
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Consumer Drivers Layer                           │
│          uart_driver    i2c_driver    spi_driver    pwm_driver      │
│               │              │             │              │          │
│               └──────────────┴─────────────┴──────────────┘         │
│                              │ devm_pinctrl_get() + select_state()  │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  Pinctrl Core (kernel/pinctrl/core.c)               │
│                                                                     │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  pinctrl_desc   │  │  pinctrl_dev     │  │  pin_desc[]      │   │
│  │  .name          │  │  .desc → above   │  │  per-pin state   │   │
│  │  .pins[]        │  │  .owner tracking │  │  (who owns me?)  │   │
│  │  .pctlops       │  │                  │  │                  │   │
│  │  .pmxops ──────────────────────────────────────────────►    │   │
│  │  .confops       │  │                  │  │                  │   │
│  └─────────────────┘  └──────────────────┘  └──────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              pinmux_ops (vtable)                            │    │
│  │   .set_mux()           → meson_axg_pmx_set_mux()           │    │
│  │   .get_functions_count → meson_pmx_get_funcs_count()        │    │
│  │   .get_function_name   → meson_pmx_get_func_name()          │    │
│  │   .get_function_groups → meson_pmx_get_groups()             │    │
│  │   .gpio_request_enable → meson_axg_pmx_request_gpio()       │    │
│  └────────────────────────────────┬────────────────────────────┘    │
└───────────────────────────────────┼─────────────────────────────────┘
                                    │ (callback implementation)
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│           Meson AXG PMX Driver (pinctrl-meson-axg-pmx.c)            │
│                                                                     │
│  meson_axg_pmx_set_mux()                                           │
│      │                                                              │
│      ├── loop over group->pins[]                                    │
│      │       │                                                      │
│      │       └── meson_axg_pmx_update_function(pin, func_val)      │
│      │               │                                              │
│      │               ├── meson_axg_pmx_get_bank(pin) → pmx_bank   │
│      │               ├── meson_pmx_calc_reg_and_offset()           │
│      │               └── regmap_update_bits(reg_mux, ...)          │
│                                                                     │
└───────────────────────────────────┬─────────────────────────────────┘
                                    │ regmap abstraction
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Hardware: Meson AXG SoC                          │
│                                                                     │
│   MUX Registers (reg_mux regmap)                                    │
│   ┌──────────────────────────────────────────────┐                 │
│   │  REG0: [pin3:0][pin2:0][pin1:0][pin0:0]      │  ← 4 bits/pin  │
│   │  REG1: [pin7:0][pin6:0][pin5:0][pin4:0]      │                 │
│   │  ...                                          │                 │
│   │  value 0x0 = GPIO mode                        │                 │
│   │  value 0x1..0xF = function 1..15              │                 │
│   └──────────────────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

### Core Abstractions: الـ Structs وعلاقتها ببعض

#### 1. مستوى الـ Hardware Description

```
meson_pinctrl_data          ← الـ "blueprint" الثابت للـ SoC
    │
    ├── pins[]              ← كل الـ pins بأرقامها (pinctrl_pin_desc)
    ├── groups[]            ← meson_pmx_group: مجموعات الـ pins
    ├── funcs[]             ← meson_pmx_func: الـ functions المتاحة
    ├── banks[]             ← meson_bank: للـ GPIO/pull/direction control
    ├── pmx_ops             ← pointer لـ meson_axg_pmx_ops
    └── pmx_data            ← pointer لـ meson_axg_pmx_data
                                    │
                                    └── pmx_banks[]   ← meson_pmx_bank
                                            (تصف أين في الـ registers)
```

#### 2. تدفق الـ Mux Selection

```
مثلاً: تفعيل UART0 على AXG
                │
                ▼
   func = "uart0"  (meson_pmx_func)
   group = "uart0_a"  (meson_pmx_group)
                │
                ├── group->pins = [GPIOA_0, GPIOA_1]
                └── group->data = meson_pmx_axg_data { .func = 1 }
                                                              │
                                              القيمة اللي هتتكتب في الـ register
```

#### 3. حساب الـ Register والـ Offset

**الـ `meson_pmx_bank`** بتوصف إزاي pins range بتتحكم في الـ mux registers:

```c
struct meson_pmx_bank {
    const char   *name;    // اسم الـ bank (للـ debug)
    unsigned int  first;   // أول pin رقمه
    unsigned int  last;    // آخر pin رقمه
    unsigned int  reg;     // base register index
    unsigned int  offset;  // bit offset للـ first pin داخل الـ base register
};
```

**المعادلة الجوهرية** في `meson_pmx_calc_reg_and_offset()`:

```
shift      = pin - bank->first          // رقم الـ pin النسبي في الـ bank
bit_pos    = bank->offset + (shift × 4) // كل pin بياخد 4 bits
reg_index  = bank->reg + bit_pos / 32   // كل register 32 bit
bit_offset = bit_pos % 32               // الـ offset داخل الـ register
```

**مثال حسابي**:

```
bank: BANK_PMX("GPIOA", first=0, last=15, reg=0, offset=0)
pin = 5 (GPIOA_5)

shift      = 5 - 0 = 5
bit_pos    = 0 + (5 × 4) = 20
reg_index  = 0 + 20/32 = 0   → REG0
bit_offset = 20 % 32 = 20

الـ mask  = 0xF << 20
القيمة   = (func & 0xF) << 20
```

```
REG0 (32-bit register):
 31      28 27      24 23      20 19      16 ...  3       0
┌─────────┬──────────┬──────────┬──────────┬──── ┬────────┐
│ pin7[3:0]│ pin6[3:0]│ pin5[3:0]│ pin4[3:0]│ .. │pin0[3:0]│
└─────────┴──────────┴──────────┴──────────┴────┴────────┘
                     ↑
              GPIOA_5 bits [23:20]
```

---

### الـ Regmap Dependency

> **الـ regmap subsystem**: طبقة abstraction فوق الـ MMIO registers. بدل `readl/writel` direct، بيوفر thread-safe access، caching، وعمليات atomic زي `regmap_update_bits`. الـ pinctrl driver يستخدمه لأن تعديل الـ mux bits لازم يبقى atomic.

الـ `regmap_update_bits(pc->reg_mux, reg << 2, 0xf << offset, (func & 0xf) << offset)` بتعمل:
1. `reg << 2`: byte address (كل register = 4 bytes)
2. `0xf << offset`: الـ mask (4 bits بس)
3. `(func & 0xf) << offset`: القيمة الجديدة

---

### الـ vtable: `meson_axg_pmx_ops`

```c
const struct pinmux_ops meson_axg_pmx_ops = {
    // الـ core بيكلمها لتفعيل function على group
    .set_mux              = meson_axg_pmx_set_mux,

    // introspection: الـ core يسأل "كام function عندك؟ وأسمائها إيه؟"
    .get_functions_count  = meson_pmx_get_funcs_count,
    .get_function_name    = meson_pmx_get_func_name,
    .get_function_groups  = meson_pmx_get_groups,

    // لما GPIO subsystem يطلب pin → اكتب 0 في الـ mux register
    .gpio_request_enable  = meson_axg_pmx_request_gpio,
};
```

**الـ `gpio_request_enable`** ببساطة بتكتب `func=0`:

```c
static int meson_axg_pmx_request_gpio(struct pinctrl_dev *pcdev,
            struct pinctrl_gpio_range *range, unsigned int offset)
{
    struct meson_pinctrl *pc = pinctrl_dev_get_drvdata(pcdev);
    return meson_axg_pmx_update_function(pc, offset, 0); // 0 = GPIO mode
}
```

---

### علاقة الـ Structs الكاملة (Graph)

```
pinctrl_dev  (الـ instance الحي في الـ runtime)
    │
    └─► meson_pinctrl  (private data بتاعت الـ driver)
              │
              ├── data ──────────────────► meson_pinctrl_data  (const, SoC-specific)
              │                                    │
              │                                    ├── pins[]        pinctrl_pin_desc
              │                                    ├── groups[] ──► meson_pmx_group
              │                                    │                       │
              │                                    │               ┌───────┴────────┐
              │                                    │           pins[]      data ──► meson_pmx_axg_data
              │                                    │                                   (.func = N)
              │                                    │
              │                                    ├── funcs[] ───► meson_pmx_func
              │                                    │                   (.groups = ["uart0_a", ...])
              │                                    │
              │                                    ├── pmx_ops ──► meson_axg_pmx_ops (vtable)
              │                                    │
              │                                    └── pmx_data ─► meson_axg_pmx_data
              │                                                         │
              │                                                   pmx_banks[] ─► meson_pmx_bank
              │                                                                    (.first/.last/.reg/.offset)
              │
              ├── reg_mux ────────────────► regmap  (للـ mux registers)
              ├── reg_gpio ───────────────► regmap  (للـ GPIO direction/value)
              ├── reg_pull ───────────────► regmap  (للـ pull-up/down)
              └── reg_pullen ─────────────► regmap  (pull enable)
```

---

### الـ Macros: اختصار تعريف الـ Data

**الـ `BANK_PMX`** في الـ header بيبني `meson_pmx_bank`:

```c
#define BANK_PMX(n, f, l, r, o) \
    { .name=n, .first=f, .last=l, .reg=r, .offset=o }

// مثال من SoC data:
static const struct meson_pmx_bank meson_axg_pmx_banks[] = {
    BANK_PMX("GPIOA",  0, 15, 0, 0),   // pins 0-15 → REG0 from bit 0
    BANK_PMX("GPIOX", 16, 35, 2, 0),   // pins 16-35 → REG2 from bit 0
};
```

**الـ `GROUP`** بيبني `meson_pmx_group` مع ربط الـ pins array والـ func value:

```c
#define GROUP(grp, f) \
    { .name=#grp, .pins=grp##_pins, .num_pins=ARRAY_SIZE(grp##_pins), \
      .data=(const struct meson_pmx_axg_data[]){ PMX_DATA(f) } }

// مثال:
static const unsigned int uart_a_tx_pins[] = { GPIOX_11 };
static const unsigned int uart_a_rx_pins[] = { GPIOX_12 };
// GROUP(uart_a_tx, 1) → pins=[GPIOX_11], func=1
```

**الـ `GPIO_GROUP`** لما pin بيشتغل كـ GPIO فقط:

```c
#define GPIO_GROUP(gpio) \
    { .name=#gpio, .pins=(unsigned int[]){gpio}, .num_pins=1, \
      .data=(struct meson_pmx_axg_data[]){ PMX_DATA(0) } }
// func=0 دايماً = GPIO mode
```

---

### ملكية الـ Subsystem: إيه بيملكه وإيه بيفوّضه

| الـ Pinctrl Core يملك | الـ Meson AXG Driver يملك |
|-----------------------|--------------------------|
| قائمة الـ pins وأرقامها | تفاصيل الـ register layout |
| ownership tracking (مين بياخد أنهي pin) | حساب reg + offset لكل pin |
| conflict detection (-EBUSY) | الكتابة الفعلية على الـ hardware عبر regmap |
| integration مع Device Tree | تعريف الـ banks وتقسيم الـ pins فيها |
| integration مع GPIO subsystem | تحديد قيمة الـ func لكل function/group |
| الـ sysfs interface للـ debug | الـ 4-bit encoding الخاص بـ AXG generation |

---

### الفرق بين الـ "First Generation" والـ AXG (Second Generation)

الكود نفسه بيشرح في التعليق الأول:

```
الجيل الأول: bits غير متتالية، كل function bit منفصل (set = enable)
الجيل الثاني (AXG): 4 bits متتالية لكل pin، value encoding:
    0x0 = GPIO
    0x1 = Function 1
    0x2 = Function 2
    ...
    0xF = Function 15
```

الـ AXG approach أوضح وأقل error-prone، وبيسمح بـ 15 function لكل pin بدل عدد محدود من الـ bits.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Macros & Config Cheatsheet

| Macro | الغرض | المعاملات |
|-------|--------|-----------|
| `BANK_PMX(n, f, l, r, o)` | يعرّف `meson_pmx_bank` واحد | name, first pin, last pin, reg offset, bit offset |
| `PMX_DATA(f)` | يعرّف `meson_pmx_axg_data` | func value (0 = GPIO) |
| `GROUP(grp, f)` | يعرّف `meson_pmx_group` لـ function group | group name, function index |
| `GPIO_GROUP(gpio)` | يعرّف group مكوّن من pin واحد GPIO | pin name/number |
| `FUNCTION(fn)` | يعرّف `meson_pmx_func` | function name |
| `BANK(...)` | يعرّف `meson_bank` بدون drive-strength | اسم + pin range + reg descriptors |
| `BANK_DS(...)` | يعرّف `meson_bank` مع drive-strength | نفس BANK + dsr/dsb |
| `MESON_PIN(x)` | يعرّف `pinctrl_pin_desc` | pin index |

---

### Enums الأساسية

#### `enum meson_reg_type`

| القيمة | المعنى | ما تتحكم فيه |
|--------|--------|--------------|
| `MESON_REG_PULLEN` | 0 | تفعيل/تعطيل pull resistor |
| `MESON_REG_PULL` | 1 | pull-up أو pull-down |
| `MESON_REG_DIR` | 2 | input أو output |
| `MESON_REG_OUT` | 3 | قيمة الـ output |
| `MESON_REG_IN` | 4 | قراءة الـ input |
| `MESON_REG_DS` | 5 | drive strength |
| `MESON_NUM_REG` | 6 | عدد الأنواع (sentinel) |

#### `enum meson_pinconf_drv`

| القيمة | التيار |
|--------|--------|
| `MESON_PINCONF_DRV_500UA` | 500 µA |
| `MESON_PINCONF_DRV_2500UA` | 2500 µA |
| `MESON_PINCONF_DRV_3000UA` | 3000 µA |
| `MESON_PINCONF_DRV_4000UA` | 4000 µA |

---

### Function Value الخاص بـ AXG Pinmux

| القيمة | المعنى |
|--------|--------|
| `0` | GPIO mode دايمًا |
| `1..15` | function modes مختلفة (4-bit field) |

---

### كل الـ Structs المهمة

#### `struct meson_pmx_bank`

**الغرض:** يصف نطاق من الـ pins بيشاركوا نفس الـ MUX register region في الـ AXG IP.

```c
struct meson_pmx_bank {
    const char   *name;     // اسم الـ bank (للـ debug)
    unsigned int  first;    // أول pin number في الـ bank
    unsigned int  last;     // آخر pin number في الـ bank
    unsigned int  reg;      // base register offset (بالـ 32-bit words)
    unsigned int  offset;   // bit offset داخل الـ base reg لأول pin
};
```

**العلاقات:**
- مجموعة منهم بتتجمع في `meson_axg_pmx_data.pmx_banks[]`
- الـ `meson_axg_pmx_get_bank()` بتبحث فيهم بـ pin number
- الـ `meson_pmx_calc_reg_and_offset()` بتستخدم حقوله عشان تحسب العنوان الفعلي

---

#### `struct meson_axg_pmx_data`

**الغرض:** الـ top-level container للـ pinmux banks الخاص بجيل AXG، بيتحط في `meson_pinctrl_data.pmx_data`.

```c
struct meson_axg_pmx_data {
    const struct meson_pmx_bank *pmx_banks;    // array of banks
    unsigned int                 num_pmx_banks; // عدد الـ banks
};
```

**العلاقات:**
- بيتخزن في `meson_pinctrl_data.pmx_data` كـ `void *`
- الـ driver بيعمله cast في `meson_axg_pmx_get_bank()`

---

#### `struct meson_pmx_axg_data`

**الغرض:** الـ per-group data، بيحمل رقم الـ function اللي المفروض يتكتب في الـ hardware register.

```c
struct meson_pmx_axg_data {
    unsigned int func;  // 4-bit function selector (0 = GPIO)
};
```

**العلاقات:**
- بيتخزن في `meson_pmx_group.data` كـ `void *`
- الـ `meson_axg_pmx_set_mux()` بيعمله cast ويقرا `func` منه

---

#### `struct meson_pmx_group`

**الغرض:** يمثل مجموعة pins بتنتمي لنفس الـ function (مثلاً كل pins الـ UART معًا).

```c
struct meson_pmx_group {
    const char         *name;      // اسم الـ group
    const unsigned int *pins;      // array of pin numbers
    unsigned int        num_pins;  // عدد الـ pins
    const void         *data;      // pointer لـ meson_pmx_axg_data
};
```

**العلاقات:**
- بيتخزن في `meson_pinctrl_data.groups[]`
- الـ `meson_axg_pmx_set_mux()` بيمشي على `pins[]` ويطبق `data->func` على كل pin

---

#### `struct meson_pmx_func`

**الغرض:** يجمع الـ groups المرتبطة بـ function واحدة (مثلاً `uart_a` function تجمع TX group و RX group).

```c
struct meson_pmx_func {
    const char        *name;       // اسم الـ function
    const char *const *groups;     // array of group names
    unsigned int       num_groups; // عدد الـ groups
};
```

**العلاقات:**
- بيتخزن في `meson_pinctrl_data.funcs[]`
- الـ pinctrl core بيسأل عنه عبر `get_function_name()` و `get_function_groups()`

---

#### `struct meson_reg_desc`

**الغرض:** يصف موقع بت واحد في الـ regmap بيتحكم في خاصية معينة لأول pin في الـ bank.

```c
struct meson_reg_desc {
    unsigned int reg; // register offset في الـ regmap
    unsigned int bit; // bit index داخل الـ register
};
```

**العلاقات:**
- `meson_bank.regs[MESON_NUM_REG]` — array من 6 descriptors لكل bank

---

#### `struct meson_bank`

**الغرض:** يصف bank من الـ GPIO pins مع كل الـ register descriptors للتحكم في pull/dir/out/in/ds.

```c
struct meson_bank {
    const char          *name;
    unsigned int         first;
    unsigned int         last;
    int                  irq_first;  // hwirq base
    int                  irq_last;
    struct meson_reg_desc regs[MESON_NUM_REG]; // 6 descriptors
};
```

**العلاقات:**
- بيتخزن في `meson_pinctrl_data.banks[]`
- الـ GPIO/pinconf subsystem بيستخدمه لعمليات pull/dir/drive-strength

---

#### `struct meson_pinctrl_data`

**الغرض:** الـ static per-SoC data، بيوصف كل الـ pins والـ groups والـ functions والـ banks للـ SoC بعينه.

```c
struct meson_pinctrl_data {
    const char                  *name;
    const struct pinctrl_pin_desc *pins;      // كل الـ pins
    const struct meson_pmx_group  *groups;    // كل الـ groups
    const struct meson_pmx_func   *funcs;     // كل الـ functions
    unsigned int                   num_pins;
    unsigned int                   num_groups;
    unsigned int                   num_funcs;
    const struct meson_bank       *banks;     // GPIO banks
    unsigned int                   num_banks;
    const struct pinmux_ops       *pmx_ops;   // مؤشر للـ ops (meson_axg_pmx_ops)
    const void                    *pmx_data;  // مؤشر لـ meson_axg_pmx_data
    int (*parse_dt)(struct meson_pinctrl *pc); // optional DT parse hook
};
```

**العلاقات:**
- بيتوصل بـ `meson_pinctrl.data`
- الـ `pmx_ops` بيشاور على `meson_axg_pmx_ops`
- الـ `pmx_data` بيتعمل cast لـ `meson_axg_pmx_data` وقت الـ mux

---

#### `struct meson_pinctrl`

**الغرض:** الـ runtime instance للـ driver، بيجمع كل الـ state للـ pinctrl controller.

```c
struct meson_pinctrl {
    struct device              *dev;        // platform device
    struct pinctrl_dev         *pcdev;      // pinctrl core handle
    struct pinctrl_desc         desc;       // وصف الـ controller
    struct meson_pinctrl_data  *data;       // static SoC data
    struct regmap              *reg_mux;    // regmap للـ MUX registers
    struct regmap              *reg_pullen; // regmap للـ pull-enable
    struct regmap              *reg_pull;   // regmap للـ pull direction
    struct regmap              *reg_gpio;   // regmap للـ GPIO
    struct regmap              *reg_ds;     // regmap للـ drive-strength
    struct gpio_chip            chip;       // GPIO chip
    struct fwnode_handle       *fwnode;     // device tree node
};
```

**العلاقات:**
- الـ `pinctrl_dev_get_drvdata(pcdev)` بيرجعه في كل callback
- الـ `reg_mux` هو الـ regmap اللي بتكتب فيه الـ `meson_axg_pmx_update_function()`

---

### Struct Relationship Diagram

```
meson_pinctrl_data (static, per-SoC)
├── pins[]         → pinctrl_pin_desc[]
├── groups[]       → meson_pmx_group[]
│                      └── data → meson_pmx_axg_data { func }
├── funcs[]        → meson_pmx_func[]
│                      └── groups[] → (names pointing back to groups[])
├── banks[]        → meson_bank[]
│                      └── regs[] → meson_reg_desc[6]
├── pmx_ops        → meson_axg_pmx_ops (pinmux_ops)
└── pmx_data       → meson_axg_pmx_data
                         └── pmx_banks[] → meson_pmx_bank[]

meson_pinctrl (runtime, per-instance)
├── data           ──────────────────→ meson_pinctrl_data (above)
├── pcdev          → pinctrl_dev (kernel core)
├── desc           → pinctrl_desc
├── reg_mux        → regmap (MUX hw registers)
├── reg_pullen     → regmap
├── reg_pull       → regmap
├── reg_gpio       → regmap
├── reg_ds         → regmap
├── chip           → gpio_chip
└── fwnode         → fwnode_handle (DT)
```

---

### Lifecycle Diagram

```
Boot / probe
─────────────
platform_device
    │
    ▼
meson_pinctrl_probe()
    │  allocates meson_pinctrl
    │  maps regmaps (reg_mux, reg_gpio, ...)
    │  fills pinctrl_desc from data->pins/groups/funcs
    │
    ▼
pinctrl_register() / devm_pinctrl_register()
    │  kernel core registers the controller
    │
    ▼
meson_pinctrl.pcdev  ←── kernel holds this handle
    │
    ▼
[Ready — waiting for consumers]

Pin mux request (consumer driver)
──────────────────────────────────
consumer calls pinctrl_select_state()
    │
    ▼
pinctrl core → meson_axg_pmx_ops.set_mux(pcdev, func_num, group_num)
    │
    ▼
meson_axg_pmx_set_mux()
    │  gets meson_pinctrl via drvdata
    │  iterates group->pins[]
    │
    ▼
meson_axg_pmx_update_function(pc, pin, func)
    │
    ├── meson_axg_pmx_get_bank()   → finds meson_pmx_bank for pin
    ├── meson_pmx_calc_reg_and_offset() → computes reg & bit offset
    └── regmap_update_bits(reg_mux, ...) → writes 4-bit func to HW

GPIO request
────────────
gpio_request() / pinctrl GPIO claim
    │
    ▼
meson_axg_pmx_ops.gpio_request_enable()
    │
    ▼
meson_axg_pmx_request_gpio(pcdev, range, offset)
    │
    └── meson_axg_pmx_update_function(pc, offset, 0)
            └── writes func=0 (GPIO mode) to HW

Teardown
────────
driver unload / device remove
    │
    ▼
devm cleanup → pinctrl_unregister()
    └── all regmaps released
```

---

### Call Flow Diagrams

#### set_mux — تغيير الـ function لـ group

```
pinctrl_select_state()          [kernel pinctrl core]
  └─► meson_axg_pmx_set_mux(pcdev, func_num, group_num)
          │
          │  pc = pinctrl_dev_get_drvdata(pcdev)
          │  func = &pc->data->funcs[func_num]
          │  group = &pc->data->groups[group_num]
          │  pmx_data = (meson_pmx_axg_data *) group->data
          │
          └─► for each pin in group->pins[]:
                meson_axg_pmx_update_function(pc, pin, pmx_data->func)
                  │
                  ├─► meson_axg_pmx_get_bank(pc, pin, &bank)
                  │     loops pmx->pmx_banks[]
                  │     returns bank where bank.first <= pin <= bank.last
                  │
                  ├─► meson_pmx_calc_reg_and_offset(bank, pin, &reg, &offset)
                  │     shift = pin - bank->first
                  │     reg    = bank->reg + (bank->offset + shift*4) / 32
                  │     offset = (bank->offset + shift*4) % 32
                  │
                  └─► regmap_update_bits(pc->reg_mux,
                                         reg << 2,        // byte address
                                         0xf << offset,   // 4-bit mask
                                         func << offset)  // 4-bit value
                              └─► hardware MUX register updated
```

#### gpio_request_enable — إعادة الـ pin لـ GPIO mode

```
gpio_request() or pinctrl claim
  └─► meson_axg_pmx_request_gpio(pcdev, range, offset)
          │
          └─► meson_axg_pmx_update_function(pc, offset, 0)
                  │  (same path as above but func=0)
                  └─► regmap_update_bits(..., 0x0 << offset)
                              └─► pin returns to GPIO mode
```

#### get_functions_count / get_function_name / get_function_groups

```
pinctrl sysfs / consumer query
  ├─► meson_pmx_get_funcs_count(pcdev)
  │       └─► returns pc->data->num_funcs
  │
  ├─► meson_pmx_get_func_name(pcdev, selector)
  │       └─► returns pc->data->funcs[selector].name
  │
  └─► meson_pmx_get_groups(pcdev, selector, **groups, *num_groups)
          └─► *groups    = pc->data->funcs[selector].groups
              *num_groups = pc->data->funcs[selector].num_groups
```

---

### Register Address Calculation

الـ AXG pinmux بيستخدم **4 bits لكل pin** (continuous)، والحساب هو:

```
shift = pin_number - bank.first

absolute_bit_offset = bank.offset + (shift × 4)

reg    = bank.reg + (absolute_bit_offset / 32)   // 32-bit word index
offset = absolute_bit_offset % 32                 // bit position inside word

byte_address = reg × 4    // regmap_update_bits يستخدم byte addressing
mask         = 0xf << offset
value        = (func & 0xf) << offset
```

مثال: لو `bank.first=10`, `bank.reg=2`, `bank.offset=8`, وأردنا pin=12:
```
shift = 12 - 10 = 2
abs_bit = 8 + (2 × 4) = 16
reg    = 2 + 16/32 = 2
offset = 16 % 32  = 16
byte_addr = 2 × 4 = 0x8
mask  = 0xf0000
```

---

### Locking Strategy

| العملية | الـ Lock | المكان |
|---------|---------|--------|
| `regmap_update_bits()` | **internal regmap lock** (mutex أو spinlock حسب الـ regmap config) | داخل الـ regmap subsystem تلقائيًا |
| `pinctrl_select_state()` | **pinctrl mutex** في `pinctrl_dev` | الـ pinctrl core قبل ما يستدعي الـ ops |
| GPIO operations | **gpio_chip lock** | الـ GPIO subsystem |

**ملاحظات:**
- الـ driver نفسه **مش بيمسك أي lock صريح** — بيعتمد على ضمانات الـ pinctrl core والـ regmap.
- الـ `regmap_update_bits()` على الـ `reg_mux` هي الـ critical section الوحيدة، ومحمية بـ internal regmap lock.
- مفيش lock ordering خاص بالـ driver لأن الـ data الـ static (banks, groups, funcs) هي `const` ومش بتتغير بعد الـ probe.
- الـ `meson_pinctrl` instance نفسه بيتعدل بس وقت الـ probe قبل ما أي consumer يشتغل، فمفيش race على الـ struct fields.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs

#### جدول الـ Functions الأساسية

| Function | النوع | الملف | الغرض |
|---|---|---|---|
| `meson_axg_pmx_get_bank` | static helper | `.c` | بتجيب الـ `pmx_bank` اللي بيتحكم في الـ pin |
| `meson_pmx_calc_reg_and_offset` | static helper | `.c` | بتحسب رقم الـ register والـ bit offset للـ pin |
| `meson_axg_pmx_update_function` | static core | `.c` | بتكتب الـ function value في الـ mux register |
| `meson_axg_pmx_set_mux` | pmx op | `.c` | الـ entry point لتفعيل function على group كامل |
| `meson_axg_pmx_request_gpio` | pmx op | `.c` | بترجّع الـ pin لـ GPIO mode (value = 0) |

#### جدول الـ Macros

| Macro | الغرض |
|---|---|
| `BANK_PMX(n,f,l,r,o)` | بيعرّف `meson_pmx_bank` بمعطياته |
| `PMX_DATA(f)` | بيعرّف `meson_pmx_axg_data` بالـ function value |
| `GROUP(grp, f)` | بيعرّف `meson_pmx_group` لـ multi-pin function group |
| `GPIO_GROUP(gpio)` | بيعرّف group خاص بـ single GPIO pin |

#### جدول الـ Ops المُصدَّرة

| Symbol | النوع | الوصف |
|---|---|---|
| `meson_axg_pmx_ops` | `struct pinmux_ops` | الـ vtable اللي بيسجّلها الـ driver في الـ pinctrl core |

---

### التصنيف المنطقي للـ Functions

```
┌─────────────────────────────────────────────────────────┐
│                  Pinmux Driver Stack                    │
│                                                         │
│  pinctrl core                                           │
│       │                                                 │
│       ▼  pinmux_ops vtable                              │
│  meson_axg_pmx_set_mux / meson_axg_pmx_request_gpio    │
│       │                                                 │
│       ▼  core logic                                     │
│  meson_axg_pmx_update_function                          │
│       │                                                 │
│       ├──► meson_axg_pmx_get_bank      (lookup)         │
│       └──► meson_pmx_calc_reg_and_offset (math)         │
│                 │                                       │
│                 ▼  regmap write                         │
│           regmap_update_bits(pc->reg_mux, ...)          │
└─────────────────────────────────────────────────────────┘
```

---

### Category 1: Lookup & Calculation Helpers

الـ functions دي بتعمل الحسابات الرياضية اللازمة عشان نعرف أي register وأي bits بتتحكم في الـ pin معين. منفصلة عن أي write operation.

---

#### `meson_axg_pmx_get_bank`

```c
static int meson_axg_pmx_get_bank(struct meson_pinctrl *pc,
                                  unsigned int pin,
                                  const struct meson_pmx_bank **bank)
```

**بتعمل إيه:**
بتلف على كل الـ `pmx_banks` المعرّفة في الـ SoC data وبتدور على الـ bank اللي الـ `pin` واقع في نطاقها (بين `first` و `last`). لو لقت الـ bank بترجع pointer ليها عبر الـ `bank` argument.

**البارامترات:**
| Parameter | النوع | الوصف |
|---|---|---|
| `pc` | `struct meson_pinctrl *` | الـ pinctrl instance اللي فيه الـ SoC data والـ regmaps |
| `pin` | `unsigned int` | رقم الـ pin الـ global في الـ SoC |
| `bank` | `const struct meson_pmx_bank **` | output: pointer لأقرب bank match |

**الـ Return Value:**
- `0` — تم الإيجاد بنجاح، الـ `*bank` صالح
- `-EINVAL` — الـ pin مش موجود في أي bank (invalid pin number)

**Key Details:**
- الـ search بالـ brute-force linear scan على `pmx->num_pmx_banks`، مفيش hashing.
- الـ `pmx_data` بتيجي من `pc->data->pmx_data` اللي بيكون `const struct meson_axg_pmx_data *`، بيتكاست implicitly.
- مفيش locking هنا — الـ caller هو المسئول عن أي locking لازم (الـ pinctrl core بيمسك الـ `pctldev->mutex` قبل ما يكال أي pmx op).
- الـ `meson_pmx_bank` بتوصف نطاق من الـ pins وموقعهم في الـ mux register space.

**Pseudocode:**
```
for i in 0..num_pmx_banks:
    if pin >= banks[i].first AND pin <= banks[i].last:
        *bank = &banks[i]
        return 0
return -EINVAL
```

---

#### `meson_pmx_calc_reg_and_offset`

```c
static int meson_pmx_calc_reg_and_offset(const struct meson_pmx_bank *bank,
                                         unsigned int pin,
                                         unsigned int *reg,
                                         unsigned int *offset)
```

**بتعمل إيه:**
بتحسب رقم الـ register (`*reg`) والـ bit offset (`*offset`) داخل الـ register ده اللي بتحكم في الـ pin المطلوب. الـ AXG IP بيستخدم **4 bits لكل pin** (continuous) لاختيار الـ function.

**البارامترات:**
| Parameter | النوع | الوصف |
|---|---|---|
| `bank` | `const struct meson_pmx_bank *` | الـ bank اللي الـ pin ينتمي ليها |
| `pin` | `unsigned int` | رقم الـ pin الـ global |
| `reg` | `unsigned int *` | output: رقم الـ register (word index) |
| `offset` | `unsigned int *` | output: bit position داخل الـ register |

**الحسابات الجوهرية:**
```c
shift = pin - bank->first;           // position داخل الـ bank (0-based)

// كل pin بياخد 4 bits → shift * 4 = shift << 2
// بنضيف bank->offset عشان نعدّل على أي initial bit offset
// بعدين بنقسم على 32 عشان نعرف رقم الـ 32-bit register

*reg    = bank->reg + (bank->offset + (shift << 2)) / 32;
*offset = (bank->offset + (shift << 2)) % 32;
```

**مثال عملي:**
لو `bank->first = 0`, `bank->offset = 0`, `bank->reg = 0`, والـ `pin = 5`:
- `shift = 5`
- `bit_pos = 0 + (5 * 4) = 20`
- `*reg = 0 + 20/32 = 0` → Register 0
- `*offset = 20 % 32 = 20` → bits [23:20] في Register 0

**الـ Return Value:** دايماً `0` — مفيش error path هنا.

**Key Details:**
- الـ `bank->reg` هو word index مش byte offset، لذلك في `meson_axg_pmx_update_function` بيتعمله `<< 2` عشان يتحول لـ byte offset للـ `regmap_update_bits`.
- الـ `bank->offset` بيسمح بـ banks متعددة تشارك نفس الـ register space بـ start offset مختلف.

---

### Category 2: Core Mux Write

الـ function دي هي اللي بتعمل الـ actual hardware write.

---

#### `meson_axg_pmx_update_function`

```c
static int meson_axg_pmx_update_function(struct meson_pinctrl *pc,
                                         unsigned int pin,
                                         unsigned int func)
```

**بتعمل إيه:**
بتكتب الـ function selector (4-bit value) في الـ mux register الصح للـ pin المحدد. الـ value 0 = GPIO mode، أي value تانية = function mode. بتستخدم `regmap_update_bits` عشان تضمن atomic read-modify-write.

**البارامترات:**
| Parameter | النوع | الوصف |
|---|---|---|
| `pc` | `struct meson_pinctrl *` | الـ pinctrl instance |
| `pin` | `unsigned int` | رقم الـ pin |
| `func` | `unsigned int` | الـ function value (0 = GPIO, 1..15 = functions) |

**الـ Return Value:**
- `0` — نجاح
- `-EINVAL` — لو الـ pin مش موجود في أي bank (من `get_bank`)
- قيمة سالبة — لو الـ `regmap_update_bits` فشلت (I/O error مثلاً)

**Key Details:**
- بتستخدم الـ mask `0xf << offset` عشان تمسك بس الـ 4 bits اللازمة وما تلمسش الـ pins التانية.
- الـ `reg << 2` هو تحويل من word index لـ byte offset لأن `regmap_update_bits` بتاخد byte offset.
- الـ `pc->reg_mux` هو الـ regmap الخاص بالـ mux registers، منفصل عن regmaps التانية (pull, GPIO, etc.).
- الـ `regmap_update_bits` نفسها thread-safe لو الـ regmap اتعمله `use_single_write = true` أو عنده internal locking.

**Pseudocode:**
```
bank = get_bank(pc, pin)         // lookup
calc_reg_and_offset(bank, pin, &reg, &offset)
regmap_update_bits(
    pc->reg_mux,
    reg << 2,                    // byte address
    0xf << offset,               // mask: 4 bits
    (func & 0xf) << offset       // value
)
```

**مثال على الـ register write:**
```
Register (32-bit): [pin7_func][pin6_func][pin5_func][pin4_func][pin3_func]...
                     bits31-28   bits27-24  bits23-20  bits19-16  bits15-12
                    (4 bits each)

لو pin=5, func=3:
  offset = 20
  mask   = 0xf00000
  value  = 0x300000  → bits [23:20] = 0b0011
```

---

### Category 3: Pinmux Ops (Entry Points من الـ pinctrl Core)

الـ functions دي هي الـ callbacks اللي الـ pinctrl core بيكالها. بتشكّل الـ `pinmux_ops` vtable.

---

#### `meson_axg_pmx_set_mux`

```c
static int meson_axg_pmx_set_mux(struct pinctrl_dev *pcdev,
                                  unsigned int func_num,
                                  unsigned int group_num)
```

**بتعمل إيه:**
الـ main entry point لتفعيل function على group كامل. بتلف على كل الـ pins في الـ group وبتكال `meson_axg_pmx_update_function` لكل pin بنفس الـ function value المعرّفة في الـ `meson_pmx_axg_data`.

**البارامترات:**
| Parameter | النوع | الوصف |
|---|---|---|
| `pcdev` | `struct pinctrl_dev *` | الـ pinctrl device handle من الـ core |
| `func_num` | `unsigned int` | index للـ function في `pc->data->funcs[]` |
| `group_num` | `unsigned int` | index للـ group في `pc->data->groups[]` |

**الـ Return Value:**
- `0` — كل الـ pins اتعمل ليهم mux بنجاح
- قيمة سالبة — أول error بيوقف الـ loop ويرجع error (no rollback!)

**Key Details:**
- الـ `func_num` بيُستخدم بس للـ debug logging (`dev_dbg`)، الـ actual function value بييجي من `pmx_data->func` الموجودة في الـ group data نفسها.
- الـ design ده بيعني إن الـ function value مخزنة per-group مش per-function، لأن نفس الـ function ممكن تستخدم values مختلفة على groups مختلفة.
- **مفيش rollback:** لو الـ pin الثالث في group من 5 pins فشل، الـ pins الأولين هيفضلوا متغيرين. الـ caller (pinctrl core) مش بيضمن rollback في الحالة دي.
- بيتكال من `pin_request` path في الـ pinctrl core بعد ما الـ core يتحقق إن الـ function والـ group موجودين.
- الـ pinctrl core بيمسك `pctldev->mutex` قبل ما يكال الـ callback.

**Pseudocode:**
```
pc = get_drvdata(pcdev)
func = &pc->data->funcs[func_num]
group = &pc->data->groups[group_num]
pmx_data = (meson_pmx_axg_data *) group->data

dev_dbg("enable function %s, group %s", func->name, group->name)

for i in 0..group->num_pins:
    ret = update_function(pc, group->pins[i], pmx_data->func)
    if ret != 0:
        return ret   // early exit, no rollback

return 0
```

---

#### `meson_axg_pmx_request_gpio`

```c
static int meson_axg_pmx_request_gpio(struct pinctrl_dev *pcdev,
                                       struct pinctrl_gpio_range *range,
                                       unsigned int offset)
```

**بتعمل إيه:**
بترجّع الـ pin لـ GPIO mode بكتابة الـ function value = 0 في الـ mux register. بيتكال لما الـ GPIO subsystem يطلب السيطرة على الـ pin عبر `gpio_request` أو `gpiod_get`.

**البارامترات:**
| Parameter | النوع | الوصف |
|---|---|---|
| `pcdev` | `struct pinctrl_dev *` | الـ pinctrl device |
| `range` | `struct pinctrl_gpio_range *` | الـ GPIO range المسجّلة (مش بتُستخدم هنا) |
| `offset` | `unsigned int` | رقم الـ pin (global pin number) |

**الـ Return Value:**
- `0` — نجاح
- قيمة سالبة — error من `update_function`

**Key Details:**
- الـ implementation بسيطة جداً: `meson_axg_pmx_update_function(pc, offset, 0)` — الـ 0 هو GPIO mode selector.
- الـ `range` parameter متجاهلة تماماً لأن الـ offset كافي.
- الـ function دي بتضمن إن أي pin بيتطلب من الـ GPIO subsystem هيتحوّل automatically من أي function mode كان فيه.
- بتُستدعى من `pinmux_request_gpio` في `kernel/pinctrl/pinmux.c`.

---

### Category 4: الـ Ops Table والـ Export

---

#### `meson_axg_pmx_ops`

```c
const struct pinmux_ops meson_axg_pmx_ops = {
    .set_mux              = meson_axg_pmx_set_mux,
    .get_functions_count  = meson_pmx_get_funcs_count,
    .get_function_name    = meson_pmx_get_func_name,
    .get_function_groups  = meson_pmx_get_groups,
    .gpio_request_enable  = meson_axg_pmx_request_gpio,
};
EXPORT_SYMBOL_GPL(meson_axg_pmx_ops);
```

**بتعمل إيه:**
الـ vtable الرئيسية اللي بتربط الـ pinctrl core بالـ AXG mux hardware. بتتسجل في `meson_pinctrl_data.pmx_ops` لكل SoC بيستخدم الـ AXG generation.

| Callback | الـ Implementation | الغرض |
|---|---|---|
| `get_functions_count` | `meson_pmx_get_funcs_count` (common) | عدد الـ functions المتاحة |
| `get_function_name` | `meson_pmx_get_func_name` (common) | اسم الـ function بالـ index |
| `get_function_groups` | `meson_pmx_get_groups` (common) | الـ groups التابعة لـ function |
| `set_mux` | `meson_axg_pmx_set_mux` (local) | تفعيل الـ mux على الـ hardware |
| `gpio_request_enable` | `meson_axg_pmx_request_gpio` (local) | تحويل pin لـ GPIO mode |

**Key Details:**
- الـ `EXPORT_SYMBOL_GPL` بيخلي الـ symbol متاح للـ modules التانية اللي بتـ build على AXG pinctrl (زي كل الـ SoC-specific pinctrl modules).
- الـ common functions (get_funcs_count, get_func_name, get_groups) مش في الملف ده — موجودة في `pinctrl-meson.c` وبيستخدمها أجيال متعددة من الـ driver.
- الـ `strict` field مش set — الـ pinctrl core بيُقدر يقبل multiple functions على نفس الـ pin لو الـ SoC data سمحت بكده.

---

### الـ Data Structures المُستخدمة

#### `struct meson_pmx_bank`

```c
struct meson_pmx_bank {
    const char   *name;    // اسم الـ bank (debug only)
    unsigned int  first;   // أول pin رقم في الـ bank
    unsigned int  last;    // آخر pin رقم في الـ bank
    unsigned int  reg;     // word index لأول register في الـ bank
    unsigned int  offset;  // bit offset للـ first pin في الـ register
};
```

الـ struct دي بتحدد الـ mapping بين الـ pin numbers والـ hardware registers. كل bank عبارة عن نطاق متصل من الـ pins.

#### `struct meson_axg_pmx_data`

```c
struct meson_axg_pmx_data {
    const struct meson_pmx_bank *pmx_banks;   // array الـ banks
    unsigned int                 num_pmx_banks; // حجم الـ array
};
```

بتتخزن في `meson_pinctrl_data.pmx_data` وبتتكاست في `meson_axg_pmx_get_bank`.

#### `struct meson_pmx_axg_data`

```c
struct meson_pmx_axg_data {
    unsigned int func;  // الـ 4-bit function selector value للـ hardware
};
```

بتتخزن في `meson_pmx_group.data` وبتتكاست في `meson_axg_pmx_set_mux`. بتفصل الـ "ما هو الـ group" عن "ما هو الـ hardware value".

---

### تدفق العملية الكاملة: من `pinctrl_select_state` لـ Register Write

```
User/Driver calls: pinctrl_select_state(state)
         │
         ▼
pinctrl core: pinmux_apply_muxsetting()
         │
         ▼
.set_mux callback → meson_axg_pmx_set_mux(pcdev, func_num, group_num)
         │
         │  [loop over group->pins[i]]
         ▼
meson_axg_pmx_update_function(pc, pin, pmx_data->func)
         │
         ├──► meson_axg_pmx_get_bank(pc, pin, &bank)
         │         └── linear search → found bank
         │
         ├──► meson_pmx_calc_reg_and_offset(bank, pin, &reg, &offset)
         │         └── reg = bank->reg + (offset + shift*4) / 32
         │             offset = (bank->offset + shift*4) % 32
         │
         └──► regmap_update_bits(pc->reg_mux,
                                  reg << 2,          ← byte address
                                  0xf << offset,     ← 4-bit mask
                                  func << offset)    ← 4-bit value
                    │
                    ▼
              Hardware MUX Register Updated
```

---

### ملاحظات على الـ Design

**لماذا الـ function value في الـ group مش في الـ function؟**
الـ `meson_pmx_axg_data.func` موجودة في الـ group data مش في الـ function struct، لأن نفس الـ logical function (مثلاً UART TX) ممكن تتحول لـ hardware value مختلفة على pins مختلفة. الـ design ده بيدي flexibility أكتر.

**لماذا 4 bits لكل pin؟**
الـ AXG generation من Amlogic عدّلت عن الـ generations الأقدم اللي كانت بتستخدم single bits مع priority logic. الـ 4 bits بتسمح بـ 16 function selector per pin (0 = GPIO, 1-15 = functions)، وبيكون الـ layout continuous في الـ register space مما يبسّط الـ hardware decoding.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs entries

**الـ pinctrl subsystem** بيعرض entries تحت `/sys/kernel/debug/pinctrl/`:

```bash
# اعرض كل الـ pinctrl devices الموجودة
ls /sys/kernel/debug/pinctrl/

# مثال على output في Meson-AXG:
# pinctrl-periphs   pinctrl-aobus

# اقرأ الـ pinmux state لكل pin
cat /sys/kernel/debug/pinctrl/pinctrl-periphs/pinmux-pins

# اقرأ الـ groups المتاحة
cat /sys/kernel/debug/pinctrl/pinctrl-periphs/pingroups

# اقرأ الـ functions المتاحة
cat /sys/kernel/debug/pinctrl/pinctrl-periphs/pinmux-functions

# اعرف أي group/function مستخدم دلوقتي
cat /sys/kernel/debug/pinctrl/pinctrl-periphs/pinconf-pins
```

**تفسير output الـ `pinmux-pins`:**
```
pin 0 (GPIOAO_0): device pinctrl-aobus function uart_ao_a group uart_ao_a_tx
pin 1 (GPIOAO_1): UNCLAIMED
```
- `UNCLAIMED` → الـ pin في GPIO mode (func=0)، مش مستخدم من أي driver
- `device ... function ... group` → الـ pin معمله mux لـ function معين

---

#### 2. sysfs entries

```bash
# اعرض الـ GPIO chips المتاحة
ls /sys/class/gpio/

# export gpio معين وشوف state
echo 410 > /sys/class/gpio/export
cat /sys/class/gpio/gpio410/direction
cat /sys/class/gpio/gpio410/value

# اعرف base number للـ gpiochip
cat /sys/class/gpio/gpiochip410/base
cat /sys/class/gpio/gpiochip410/ngpio
cat /sys/class/gpio/gpiochip410/label

# regmap debug (لو CONFIG_REGMAP_DEBUGFS=y)
ls /sys/kernel/debug/regmap/
# هتلاقي entries زي:  c8834000.bus:periphs-pinctrl
cat /sys/kernel/debug/regmap/c8834000.bus\:periphs-pinctrl/registers
```

**الـ regmap registers output بيكون:**
```
00: 00000000
04: 00003000
08: 00000000
```
- كل register = 32-bit، كل 4-bits بتحكم في function لـ pin واحد
- القيمة 0 = GPIO mode، القيم 1-15 = alternative functions

---

#### 3. ftrace — Tracepoints والـ Events

```bash
# شوف الـ pinctrl events المتاحة
ls /sys/kernel/debug/tracing/events/pinctrl/

# فعّل كل الـ pinctrl events
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# أو events محددة
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_single_set/enable

# تتبع الـ meson_axg_pmx_set_mux باستخدام function tracer
echo function > /sys/kernel/debug/tracing/current_tracer
echo meson_axg_pmx_set_mux > /sys/kernel/debug/tracing/set_ftrace_filter
echo meson_axg_pmx_update_function >> /sys/kernel/debug/tracing/set_ftrace_filter
echo meson_pmx_calc_reg_and_offset >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شوف النتيجة
cat /sys/kernel/debug/tracing/trace

# وقف الـ tracing
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

**مثال على trace output:**
```
kworker/0:1-45    [000] ....  123.456789: meson_axg_pmx_set_mux <-pinmux_enable_setting
kworker/0:1-45    [000] ....  123.456790: meson_axg_pmx_update_function <-meson_axg_pmx_set_mux
kworker/0:1-45    [000] ....  123.456791: meson_pmx_calc_reg_and_offset <-meson_axg_pmx_update_function
```

---

#### 4. printk / Dynamic Debug

**الـ driver بيستخدم `dev_dbg`** في `meson_axg_pmx_set_mux`:
```c
dev_dbg(pc->dev, "enable function %s, group %s\n", func->name, group->name);
```

لتفعيل هذا الـ message:

```bash
# طريقة 1: dynamic debug
echo "file pinctrl-meson-axg-pmx.c +p" > /sys/kernel/debug/dynamic_debug/control

# طريقة 2: تفعيل كل الـ pinctrl debug messages
echo "module pinctrl_meson_axg +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق إن التفعيل اشتغل
grep pinctrl-meson-axg /sys/kernel/debug/dynamic_debug/control

# شوف الـ output
dmesg -w | grep -E "pinctrl|meson.*pmx"

# مثال على output بعد التفعيل:
# [  45.123] pinctrl-meson-axg-pmx c8834000.bus:periphs-pinctrl: enable function uart_a, group uart_a_tx
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوصف |
|--------|--------|
| `CONFIG_PINCTRL_MESON_AXG` | تفعيل الـ driver الأساسي |
| `CONFIG_DEBUG_PINCTRL` | تفعيل debug logs في الـ pinctrl core |
| `CONFIG_REGMAP_DEBUGFS` | expose الـ regmap registers في debugfs |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل dynamic debug للـ `dev_dbg` |
| `CONFIG_FTRACE` | تفعيل function tracing |
| `CONFIG_GPIO_SYSFS` | export GPIO عبر sysfs |
| `CONFIG_PROVE_LOCKING` | كشف لـ locking bugs |
| `CONFIG_DEBUG_GPIO` | فعّل extra checks في GPIO subsystem |
| `CONFIG_GPIOLIB_IRQCHIP` | debugging لـ GPIO IRQ |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "PINCTRL|REGMAP_DEBUG|DEBUG_GPIO"
```

---

#### 6. devlink وأدوات الـ Subsystem

```bash
# استخدم pinctrl CLI tool (من pinctrl-utils أو busybox)
pinctrl list          # اعرض كل الـ controllers
pinctrl info          # معلومات تفصيلية
pinctrl get GPIOAO_0  # اعرف state لـ pin معين

# gpiodetect + gpioinfo (من libgpiod)
gpiodetect            # اعرض كل الـ GPIO chips
gpioinfo gpiochip0    # اعرض كل الـ pins وحالتها

# مثال على gpioinfo output:
# gpiochip0 - 86 lines:
#   line   0:  "GPIOAO_0"  unused   input  active-high
#   line   1:  "GPIOAO_1"  "uart0"  output active-high [used]

# gpioget/gpioset للـ testing
gpioget gpiochip0 5         # اقرأ قيمة pin 5
gpioset gpiochip0 5=1       # اكتب 1 على pin 5

# regmap register dump باستخدام devmem2
# أولاً اعرف الـ base address من DT أو /proc/iomem
cat /proc/iomem | grep pinctrl
```

---

#### 7. جدول Error Messages الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|-----------------|--------|-------|
| `pinctrl request already granted for pin X` | الـ pin مستخدم من driver تاني | تحقق من DT conflicts، شوف `pinmux-pins` |
| `pin X is not valid` | رقم الـ pin خارج النطاق | تحقق من `first`/`last` في `BANK_PMX` |
| `could not get pin X` | فشل في الـ pin lookup | تحقق إن الـ pin معرّف في `meson_pins[]` |
| `-EINVAL` من `meson_axg_pmx_get_bank` | الـ pin مش موجود في أي bank | الـ `pmx_banks` ناقصة أو `first`/`last` غلط |
| `regmap_update_bits failed` | فشل كتابة الـ register | مشكلة في الـ regmap/bus، تحقق من الـ clock |
| `pinctrl: no pins found for group` | الـ group فاضل من pins | مشكلة في تعريف الـ `GROUP()` macro |
| `function X has no group` | الـ function مش مربوط بـ group | تحقق من `FUNCTION()` macro في SoC data |
| `Failed to lookup pin` | الـ pin descriptor مش موجود | الـ `num_pins` أو `pins[]` array غلط |

---

#### 8. Strategic Points لـ `dump_stack()` / `WARN_ON()`

**النقاط الاستراتيجية في الكود:**

```c
/* في meson_axg_pmx_get_bank — لو الـ pin مش في أي bank */
static int meson_axg_pmx_get_bank(struct meson_pinctrl *pc,
            unsigned int pin,
            const struct meson_pmx_bank **bank)
{
    /* WARN_ON هنا لو رجعنا -EINVAL يكشف من اللي بعت pin غلط */
    int ret = -EINVAL;
    ...
    WARN_ON(ret == -EINVAL); /* pin %u not in any pmx bank */
    return ret;
}

/* في meson_axg_pmx_update_function — قبل regmap_update_bits */
/* WARN_ON(func > 0xf) — الـ func value 4-bit بس */

/* في meson_axg_pmx_set_mux — لو pmx_data == NULL */
WARN_ON(!pmx_data);

/* dump_stack() مفيد لو عندك double-mux لنفس الـ pin */
```

---

### Hardware Level

---

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

```bash
# خطوة 1: اعرف أي function المفروض يكون على الـ pin
cat /sys/kernel/debug/pinctrl/pinctrl-periphs/pinmux-pins | grep "GPIOX_0"
# Output: pin 50 (GPIOX_0): device ... function uart_a group uart_a_tx

# خطوة 2: احسب الـ register والـ offset يدوياً
# مثال: GPIOX_0 = pin 50، bank GPIOX: first=50, reg=2, offset=0
# shift = 50 - 50 = 0
# reg = 2 + (0 + 0*4)/32 = 2
# offset = (0 + 0) % 32 = 0
# القيمة المتوقعة في bits [3:0] = function number (مثلاً 1 لـ UART)

# خطوة 3: اقرأ الـ register الفعلي
devmem2 0xc8834008 w    # reg=2 → offset = 2*4 = 8 → base + 8
# Output: 0xC8834008 = 0x00000001  ← bits[3:0] = 1 (UART_A)
```

---

#### 2. Register Dump Techniques

```bash
# اعرف الـ base address للـ pinmux registers من /proc/iomem
cat /proc/iomem | grep -i "periphs\|pinctrl\|c883"

# مثال output:
# c8834400-c8834fff : c8834400.bus:periphs-pinctrl

# dump كل الـ mux registers (عادةً 8-16 register)
for i in $(seq 0 7); do
    addr=$((0xC8834400 + i * 4))
    printf "REG[%d] @ 0x%08X = " $i $addr
    devmem2 $(printf "0x%08X" $addr) w 2>/dev/null | grep "Read at"
done

# استخدام /dev/mem مع python لـ dump كامل
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 64, offset=0xC8834400)
    for i in range(16):
        val = struct.unpack('<I', m[i*4:i*4+4])[0]
        print(f'REG[{i:2d}] = 0x{val:08X}  pins: ', end='')
        for p in range(8):
            print(f'P{i*8+p}={( val>>(p*4))&0xF}', end=' ')
        print()
"
```

**تفسير الـ register dump:**
```
REG[ 0] = 0x00011100  pins:  P0=0 P1=1 P2=1 P3=1 P4=0 P5=0 P6=0 P7=0
                              ^GPIO ^UART ^UART ^UART ^GPIO
```
- كل 4 bits = function لـ pin واحد
- 0 = GPIO mode
- 1-15 = alternative functions حسب الـ SoC datasheet

---

#### 3. Logic Analyzer / Oscilloscope Tips

```
استخدام Logic Analyzer:
┌─────────────────────────────────────────────────────┐
│  CH0: TX pin → تحقق إن الـ signal بيظهر بعد الـ mux  │
│  CH1: CLK    → تحقق من الـ frequency                 │
│  CH2: GPIO   → لازم يكون stable لو في GPIO mode      │
└─────────────────────────────────────────────────────┘

نقاط مهمة:
- الـ pin في GPIO mode (func=0) → لازم output يكون DC ثابت
- بعد set_mux لـ UART → لازم تشوف idle line (HIGH)
- بعد set_mux لـ I2C → لازم تشوف pull-up على SDA/SCL
- لو الـ pin مش بيتغير بعد kernel set_mux → الـ register مش اتكتب
```

**Setup للـ Logic Analyzer:**
```bash
# قبل التوصيل، تحقق من الـ voltage level
cat /sys/class/gpio/gpio410/active_low
# الـ Meson-AXG بيشتغل على 3.3V في الغالب

# اعمل toggle سريع لـ GPIO لتأكيد الاتصال
gpioset gpiochip0 10=1; sleep 0.1; gpioset gpiochip0 10=0
```

---

#### 4. Common Hardware Issues وكيف تظهر في الـ Kernel Log

| المشكلة | Pattern في الـ dmesg | السبب المحتمل |
|---------|---------------------|--------------|
| الـ pin مش بيتغير | لا يوجد output/signal | الـ clock للـ bus الـ pinctrl معمله gate |
| قيمة الـ function غلط | سلوك غريب للـ peripheral | `func` value في `PMX_DATA()` غلط |
| الـ bus error على regmap | `regmap: error in read` | الـ base address في DT غلط |
| الـ pin شايل function قديم | peripheral مش بيشتغل | مش اتعمله reset بعد boot |
| الـ drive strength ضعيف | signal distorted في oscilloscope | `MESON_REG_DS` مش اتضبط |

```bash
# تحقق من الـ clock
cat /sys/kernel/debug/clk/clk_summary | grep -i "periphs\|pinctrl"

# لو الـ clock disabled → الـ regmap writes مش هتشتغل
```

---

#### 5. Device Tree Debugging

```bash
# اعرض الـ DT node للـ pinctrl
cat /proc/device-tree/soc/bus@c8834400/pinctrl@400/compatible
# Expected: amlogic,meson-axg-periphs-pinctrl

# اعرض كل الـ pinctrl nodes في الـ DT
find /proc/device-tree -name "pinctrl*" -exec echo {} \;

# تحقق من الـ reg property (base address + size)
hexdump -C /proc/device-tree/soc/bus@c8834400/pinctrl@400/reg

# اعرض الـ pin configurations المستخدمة
find /proc/device-tree -name "pinctrl-0" 2>/dev/null | head -5

# تحقق من الـ gpio-ranges
cat /proc/device-tree/soc/bus@c8834400/pinctrl@400/gpio-ranges 2>/dev/null | hexdump -C

# dump الـ DT compiled (dtc)
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "periphs-pinctrl"
```

**مثال على DT صح لـ Meson-AXG:**
```dts
periphs_pinctrl: pinctrl@400 {
    compatible = "amlogic,meson-axg-periphs-pinctrl";
    #address-cells = <1>;
    #size-cells = <1>;
    ranges;

    gpio: bank@40 {
        reg = <0x40  0x4c>,   /* GPIO control */
              <0x0   0x40>;   /* PMX registers */
        reg-names = "gpio", "mux";
        gpio-controller;
        #gpio-cells = <2>;
    };
};
```

**علامات إن الـ DT غلط:**
```bash
dmesg | grep -E "meson.*pinctrl|pinctrl.*probe|of_property"
# لو شفت: "failed to get reg" أو "missing reg property"
# → الـ reg addresses في DT غلطانة
```

---

### Practical Commands

---

#### جميع الأوامر جاهزة للـ Copy

```bash
#!/bin/bash
# === Meson-AXG PMX Debug Script ===

PINCTRL_DIR="/sys/kernel/debug/pinctrl"
PINCTRL_NAME="pinctrl-periphs"   # غيّره حسب SoC

# 1. اعرض كل الـ pins وحالتها
echo "=== Pinmux State ==="
cat ${PINCTRL_DIR}/${PINCTRL_NAME}/pinmux-pins

# 2. اعرض الـ functions المتاحة
echo "=== Available Functions ==="
cat ${PINCTRL_DIR}/${PINCTRL_NAME}/pinmux-functions

# 3. اعرض الـ groups المتاحة
echo "=== Pin Groups ==="
cat ${PINCTRL_DIR}/${PINCTRL_NAME}/pingroups

# 4. فعّل dynamic debug
echo "file pinctrl-meson-axg-pmx.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "Dynamic debug enabled for pinctrl-meson-axg-pmx.c"

# 5. فعّل ftrace لتتبع set_mux
echo function > /sys/kernel/debug/tracing/current_tracer
echo meson_axg_pmx_set_mux      > /sys/kernel/debug/tracing/set_ftrace_filter
echo meson_axg_pmx_update_function >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "ftrace enabled — trigger your driver then run:"
echo "  cat /sys/kernel/debug/tracing/trace"

# 6. اعرض الـ GPIO info
gpiodetect 2>/dev/null || echo "install libgpiod for gpiodetect"
gpioinfo   2>/dev/null | head -20

# 7. dump الـ regmap registers لو متاح
REGMAP_BASE=$(ls /sys/kernel/debug/regmap/ 2>/dev/null | grep pinctrl | head -1)
if [ -n "$REGMAP_BASE" ]; then
    echo "=== Regmap Registers ==="
    cat /sys/kernel/debug/regmap/${REGMAP_BASE}/registers
fi
```

---

#### تفسير نتيجة `pinmux-pins`

```
pin 50 (GPIOX_0): device c8834400.bus:periphs-pinctrl function uart_a group uart_a_tx

التفسير:
  pin 50        → رقم الـ pin في النظام
  GPIOX_0       → اسم الـ pin في الـ SoC
  device ...    → الـ pinctrl controller اللي بيتحكم فيه
  function uart_a → الـ function المفعّل دلوقتي
  group uart_a_tx → الـ group اللي الـ pin ده جزء منه
```

---

#### التحقق اليدوي من حساب الـ Register

```bash
# مثال: pin 55، bank GPIOX (first=50, reg=2, offset=0)
PIN=55
BANK_FIRST=50
BANK_REG=2
BANK_OFFSET=0
BASE_ADDR=0xC8834400

shift=$((PIN - BANK_FIRST))                        # = 5
bit_offset=$((BANK_OFFSET + shift * 4))            # = 20
reg=$((BANK_REG + bit_offset / 32))               # = 2
offset=$((bit_offset % 32))                        # = 20

reg_addr=$((BASE_ADDR + reg * 4))
printf "Register address: 0x%08X\n" $reg_addr     # 0xC8834408
printf "Bit offset in register: %d\n" $offset      # 20
printf "Mask: 0x%08X\n" $((0xF << offset))        # 0x00F00000

# اقرأ الـ register
devmem2 $(printf "0x%08X" $reg_addr) w
# مثال output: 0xC8834408: 0x00100000
# bits[23:20] = 1 → function 1 (UART_A)
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Android TV Box بـ Amlogic S905X3 — UART مش شغال بعد boot

#### العنوان
**الـ debug UART بيطلع garbage بعد kernel يبدأ على S905X3 TV box**

#### السياق
بتعمل bring-up لـ Android TV box بـ **Amlogic S905X3** (نفس جيل AXG pinmux). الـ bootloader بيطبع على UART كويس، بس بعد ما الـ kernel يبدأ الـ console بيطلع garbage أو مفيش output خالص.

#### المشكلة
الـ UART_TX و UART_RX pins مش بتاخد الـ function الصح. الـ console متوقف وانت عميا تماماً على الـ board.

#### التحليل
الـ flow بيبدأ من `meson_axg_pmx_set_mux`:

```c
static int meson_axg_pmx_set_mux(struct pinctrl_dev *pcdev,
            unsigned int func_num, unsigned int group_num)
{
    /* بياخد الـ group اللي فيه pins الـ UART */
    const struct meson_pmx_group *group = &pc->data->groups[group_num];
    struct meson_pmx_axg_data *pmx_data =
        (struct meson_pmx_axg_data *)group->data;

    for (i = 0; i < group->num_pins; i++) {
        ret = meson_axg_pmx_update_function(pc, group->pins[i],
            pmx_data->func);  /* func = رقم الـ mux value */
    }
}
```

الـ `meson_axg_pmx_update_function` بتكتب 4-bit field في الـ register:

```c
ret = regmap_update_bits(pc->reg_mux, reg << 2,
    0xf << offset, (func & 0xf) << offset);
```

لو الـ `func` value غلط (مثلاً 2 بدل 1)، الـ pin بيروح لـ function تانية مش UART.

الـ `meson_pmx_calc_reg_and_offset` بتحسب الـ register والـ bit:

```c
shift = pin - bank->first;
*reg = bank->reg + (bank->offset + (shift << 2)) / 32;
*offset = (bank->offset + (shift << 2)) % 32;
```

`shift << 2` يعني كل pin بياخد 4 bits. لو `bank->first` غلط في تعريف `BANK_PMX`، الـ shift بيتحسب غلط والـ write بيروح على pin تاني.

#### الحل

**1. تحقق من الـ DT:**
```bash
# شوف الـ pinctrl node في DT
dtc -I dtb -O dts /proc/device-tree > /tmp/full.dts
grep -A 10 "uart_a" /tmp/full.dts
```

**2. اقرأ الـ mux register مباشرة:**
```bash
# على S905X3 الـ periphs pinmux base عادةً 0xFF634400
devmem2 0xFF634400 w   # PIN_MUX_REG0
devmem2 0xFF634404 w   # PIN_MUX_REG1
```

**3. شوف الـ `pmx_data->func` الصح في الـ SoC header:**
```c
/* في pinctrl-meson-g12a.c */
static const struct meson_pmx_axg_data uart_a_tx_mux = PMX_DATA(1);
/* تأكد إن الـ func value = 1 مش 0 أو 2 */
```

**4. فعّل الـ dynamic debug:**
```bash
echo "file pinctrl-meson-axg-pmx.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg | grep "enable function"
```

#### الدرس المستفاد
الـ `func` value في `PMX_DATA(f)` لازم يتطابق بالظبط مع الـ SoC datasheet. قيمة 0 دايماً = GPIO mode. أي قيمة تانية = alternate function مختلفة. غلطة في الـ header بتعمي الـ console كله.

---

### السيناريو 2: Industrial Gateway بـ Amlogic A311D — SPI flash مش بيتعرف عليه

#### العنوان
**الـ SPI NOR flash مش بيظهر في `/dev/mtd*` على gateway صناعي بـ A311D**

#### السياق
بتبني industrial gateway بـ **Amlogic A311D** (من عيلة AXG). الـ SPI NOR flash المفروض يكون على SPI0. الـ kernel بيطلع بدون أي `/dev/mtd` وسجل kernel فيه `spi-nor: unrecognized JEDEC id`.

#### المشكلة
الـ SPI CLK, MOSI, MISO, CS pins لسه شغالين كـ GPIO (func=0) بدل ما يتحولوا لـ SPI.

#### التحليل
لما `meson_axg_pmx_set_mux` بيتنادى للـ SPI group:

```c
/* الـ group بيحتوي على 4 pins: CLK, MOSI, MISO, CS */
for (i = 0; i < group->num_pins; i++) {
    ret = meson_axg_pmx_update_function(pc, group->pins[i],
        pmx_data->func);
```

الـ `meson_axg_pmx_get_bank` بتدور على الـ bank الصح:

```c
for (i = 0; i < pmx->num_pmx_banks; i++)
    if (pin >= pmx->pmx_banks[i].first &&
            pin <= pmx->pmx_banks[i].last) {
        *bank = &pmx->pmx_banks[i];
        return 0;
    }

return -EINVAL;  /* لو الـ pin مش في أي bank => error */
```

لو الـ pin number في الـ group definition مش موجود في أي `meson_pmx_bank`، الـ function بترجع `-EINVAL` وبقية الـ pins مبتتعملش mux، فبيفضلوا GPIO.

#### الحل

**1. شوف الـ error في kernel log:**
```bash
dmesg | grep -i "pinctrl\|spi\|mux"
# ابحث عن: "meson_axg_pmx_get_bank: pin XX not found"
```

**2. افحص الـ bank definitions في الـ SoC pinctrl file:**
```c
/* في pinctrl-meson-a1.c أو المكافئ لـ A311D */
static const struct meson_pmx_bank meson_a1_pmx_banks[] = {
    BANK_PMX("P",    0,  62,  0,  0),
    /* تأكد إن كل الـ SPI pins داخلة في النطاق */
};
```

**3. قارن الـ pin numbers في الـ GROUP:**
```c
/* لو SPI_CLK هو pin 70 بس الـ bank بيوصل لـ 62 بس => miss */
static const unsigned int spi0_clk_pins[] = { 70 };
/* هنا المشكلة: pin 70 خارج نطاق الـ bank */
```

**4. الحل: صحح الـ BANK_PMX أو الـ pin number:**
```c
/* زود النطاق أو صحح الـ pin number حسب الـ datasheet */
BANK_PMX("P",    0,  75,  0,  0),
```

#### الدرس المستفاد
`meson_axg_pmx_get_bank` بترجع `-EINVAL` بصمت نسبي. الـ SPI driver ممكن يكمل initializition وبعدين يفشل بـ JEDEC error لأن الـ lines مش مضبوطة. دايماً تأكد إن نطاق الـ `BANK_PMX` بيغطي كل الـ pins المذكورة في الـ groups.

---

### السيناريو 3: IoT Sensor Board بـ Amlogic S905X3 — I2C sensor بيطلع garbage readings

#### العنوان
**الـ I2C temperature sensor بيطلع قراءات عشوائية على IoT board بـ S905X3**

#### السياق
بتبني IoT sensor node بـ **Amlogic S905X3** متصل بـ I2C temperature/humidity sensor. الـ readings بتجي عشوائية، وأحياناً الـ bus بيـ hang خالص. الـ `i2cdetect` بيـ timeout.

#### المشكلة
pins الـ I2C SDA و SCL اتحسب لهم wrong register offset بسبب غلطة في `BANK_PMX` offset parameter، فـ mux value بيتكتب على pins غلط.

#### التحليل
`meson_pmx_calc_reg_and_offset` بتحسب:

```c
shift = pin - bank->first;

/* كل pin بياخد 4 bits (shift << 2) */
*reg    = bank->reg + (bank->offset + (shift << 2)) / 32;
*offset = (bank->offset + (shift << 2)) % 32;
```

مثال: لو `bank->first = 0`, `bank->reg = 0`, `bank->offset = 0`:
- Pin 5 (SDA): `shift=5`, `reg = (0 + 20)/32 = 0`, `offset = 20`
- Pin 6 (SCL): `shift=6`, `reg = (0 + 24)/32 = 0`, `offset = 24`

لو حد غلط الـ `offset` في `BANK_PMX` وخلاه `8` بدل `0`:
- Pin 5: `reg = (8 + 20)/32 = 0`, `offset = 28` — غلط!
- Pin 6: `reg = (8 + 24)/32 = 1`, `offset = 0` — بيكتب في register تاني!

الـ `regmap_update_bits` بيكتب على register address غلط:
```c
ret = regmap_update_bits(pc->reg_mux, reg << 2,
    0xf << offset, (func & 0xf) << offset);
/* reg << 2: address بيتحول لـ byte offset */
```

#### الحل

**1. احسب يدوياً الـ expected register:**
```
I2C_SDA = pin 5, func = 2 (I2C mode)
Expected: reg=0, offset=20
Expected register value at 0x0: bits [23:20] = 0x2
```

**2. اقرأ الـ actual register:**
```bash
devmem2 0xFF634400 w  # MUX_REG0 على S905X3
# لو bits [23:20] = 0 => pin لسه GPIO
# لو bit [31:28] = 2 => الكتابة راحت على pin 7 بالغلط
```

**3. صحح الـ BANK_PMX:**
```c
/* قبل الإصلاح */
BANK_PMX("X",  0, 19,  0,  8),   /* offset=8 غلط */

/* بعد الإصلاح — رجعله 0 حسب الـ datasheet */
BANK_PMX("X",  0, 19,  0,  0),
```

**4. تحقق بعد الإصلاح:**
```bash
i2cdetect -y 0
# المفروض يظهر الـ sensor على address صح
```

#### الدرس المستفاد
الـ `bank->offset` في `BANK_PMX` بيحسب bit position داخل الـ register block. غلطة فيه بتعمل cascade error في كل حسابات `meson_pmx_calc_reg_and_offset`، وبتأثر على كل الـ pins في الـ bank، مش pin واحد بس.

---

### السيناريو 4: Custom Board بـ Amlogic A311D — GPIO request بيكسر SPI accidentally

#### العنوان
**طلب GPIO واحد بيعطل الـ SPI bus كله على custom board أثناء الـ bring-up**

#### السياق
فريق software بيعمل bring-up على **custom board بـ Amlogic A311D** لـ automotive application. الـ SPI display شغال كويس. الـ developer عمل `gpio_request` لـ pin واحد عشان يعمل debug LED، وفجأة الـ SPI display وقف.

#### المشكلة
الـ `gpio_request` بيستدعي `meson_axg_pmx_request_gpio` اللي بتفرس func=0 (GPIO mode) على الـ pin، وبالصدفة الـ pin ده هو SPI_CLK.

#### التحليل
`meson_axg_pmx_request_gpio` صريحة جداً:

```c
static int meson_axg_pmx_request_gpio(struct pinctrl_dev *pcdev,
            struct pinctrl_gpio_range *range, unsigned int offset)
{
    struct meson_pinctrl *pc = pinctrl_dev_get_drvdata(pcdev);

    /* بتكتب 0 على الـ 4-bit field => GPIO mode */
    return meson_axg_pmx_update_function(pc, offset, 0);
}
```

الـ `offset` هنا هو الـ global pin number. لو الـ developer غلط في رقم الـ pin وطلب مثلاً pin 15 وهو SPI_CLK:

```c
/* بتحسب الـ register والـ offset لـ pin 15 */
meson_pmx_calc_reg_and_offset(bank, 15, &reg, &offset);

/* بتكتب 0x0 في الـ 4-bit field => pin اتحول لـ GPIO */
regmap_update_bits(pc->reg_mux, reg << 2,
    0xf << offset, 0);
```

الـ SPI controller لسه بيحاول يرسل data بس الـ CLK line مش بتشتغل كـ SPI function تاني.

#### الحل

**1. شوف مين طلب الـ GPIO:**
```bash
cat /sys/kernel/debug/gpio
# شوف الـ "gpio-15" مين اللي بيمسكه
```

**2. شوف الـ pinctrl state الحالية:**
```bash
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep "pin 15"
```

**3. صحح الـ code:**
```c
/* كان غلط — طلب pin الـ SPI */
gpio_request(15, "debug-led");

/* الصح — استخدم الـ pin الصح للـ LED */
gpio_request(20, "debug-led");  /* pin 20 free GPIO */
```

**4. في الـ DTS: اعمل pinctrl exclusivity:**
```dts
&spi0 {
    pinctrl-0 = <&spi0_pins>;
    pinctrl-names = "default";
    /* الـ pinctrl framework هيمنع أي حد تاني من استخدام نفس الـ pins */
};
```

#### الدرس المستفاد
`meson_axg_pmx_request_gpio` بتعمل override فوري على الـ mux register من غير ما تسأل إيه الـ function الحالية. الـ pinctrl framework المفروض يمنع ده لو الـ pins متسجلين بشكل صح في الـ DT، لكن لو حد bypass الـ framework وطلب GPIO مباشرة، الـ SPI بيتكسر على طول.

---

### السيناريو 5: Automotive ECU بـ Amlogic A311D — HDMI مش بيظهر بعد suspend/resume

#### العنوان
**الـ HDMI output مش بترجع بعد system suspend/resume على automotive display unit بـ A311D**

#### السياق
بتبني automotive infotainment unit بـ **Amlogic A311D**. الـ HDMI شغال تمام في الـ normal boot. بعد إن الـ system يعمل suspend وبعدين resume، الـ HDMI display بيفضل أسود. الـ `/sys/class/drm` بيقول الـ connector connected بس مفيش image.

#### المشكلة
بعد الـ resume، الـ pinctrl state مش بترجع لـ default. الـ HDMI DDC pins (I2C للـ EDID) فاضلين على GPIO mode بدل ما يرجعوا لـ I2C/HDMI function.

#### التحليل
أثناء الـ suspend، الـ pinctrl states ممكن تترجع لـ sleep state. أثناء الـ resume، المفروض `meson_axg_pmx_set_mux` تتنادى تاني لـ restore الـ default state.

المشكلة إن الـ `meson_axg_pmx_ops` مش عنده `strict` flag، ولو الـ driver بيعمل resume بدون ما ينادي pinctrl_select_state صح، الـ pins بيفضلوا على الـ state الأخيرة قبل الـ suspend (ممكن تكون sleep/GPIO).

لما الـ HDMI driver بيحاول يقرأ الـ EDID عبر DDC:
1. DDC = I2C على pins معينة
2. الـ pins لسه GPIO mode (func=0) من الـ suspend
3. الـ `meson_axg_pmx_request_gpio` سبق واتنادت عليهم

ممكن كمان تكون الـ `meson_axg_pmx_update_function` اتنادت بـ func=0 من الـ GPIO subsystem:

```c
/* أثناء الـ suspend: GPIO subsystem ممكن يعمل ده */
meson_axg_pmx_update_function(pc, hdmi_ddc_sda_pin, 0);
meson_axg_pmx_update_function(pc, hdmi_ddc_scl_pin, 0);
```

#### الحل

**1. تحقق من الـ pin state بعد الـ resume:**
```bash
# بعد resume مباشرةً
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -i "hdmi\|ddc"
```

**2. اقرأ الـ mux register للـ DDC pins يدوياً:**
```bash
# لو DDC_SDA على pin 10, func المفروض = 1
# احسب الـ register يدوياً:
# shift = 10 - bank->first
# reg_addr = base + (offset + shift*4)/32 * 4
devmem2 0xFF634400 w
```

**3. في الـ DTS: ضيف sleep state للـ HDMI:**
```dts
&hdmi_tx {
    pinctrl-0 = <&hdmi_hpd_pins>, <&hdmi_ddc_pins>;
    pinctrl-1 = <&hdmi_hpd_pins_sleep>, <&hdmi_ddc_pins_sleep>;
    pinctrl-names = "default", "sleep";
};
```

**4. في الـ pinctrl data: عرّف sleep state بـ func=1 (مش 0):**
```c
/* خلي الـ DDC pins في HDMI mode حتى في الـ sleep */
static const struct meson_pmx_axg_data hdmi_ddc_sleep_mux = PMX_DATA(1);
```

**5. تأكد إن الـ HDMI driver بيعمل pinctrl_select_state("default") في resume:**
```c
static int hdmi_resume(struct device *dev)
{
    /* لازم يتنادى صراحةً */
    pinctrl_select_state(hdmi_priv->pctrl,
                         hdmi_priv->pins_default);
    /* ... */
}
```

#### الدرس المستفاد
الـ `meson_axg_pmx_set_mux` و `meson_axg_pmx_request_gpio` بيشتغلوا على مستوى الـ hardware registers مباشرة. لما الـ system بيعمل suspend/resume، الـ pinctrl framework مش بيعمل auto-restore للـ states. كل driver مسؤول إنه ينادي `pinctrl_select_state` في الـ resume path. والـ sleep pinctrl state المفروض تحافظ على الـ func المناسبة مش دايماً تحولها لـ GPIO.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ pinctrl subsystem والـ Meson drivers بشكل مباشر:

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| **The pin control subsystem** — المقال الأساسي اللي شرح فلسفة الـ pinctrl لما اتضافت للـ kernel | [lwn.net/Articles/468759](https://lwn.net/Articles/468759/) | ضروري جداً |
| **Documentation/pinctrl.txt** — أول نسخة من الـ documentation الرسمي على LWN | [lwn.net/Articles/465077](https://lwn.net/Articles/465077/) | ضروري |
| **pin controller subsystem v7** — الـ patchset اللي أدخل الـ subsystem للـ kernel | [lwn.net/Articles/459190](https://lwn.net/Articles/459190/) | تاريخي مهم |
| **Pinctrl driver for Amlogic Meson SoCs** — أول driver لـ Meson على الـ mainline | [lwn.net/Articles/616225](https://lwn.net/Articles/616225/) | مرجع مباشر |
| **Amlogic Meson pinctrl driver** — الـ patch series اللي بني عليه الكود الحالي | [lwn.net/Articles/620822](https://lwn.net/Articles/620822/) | مرجع مباشر |
| **pinctrl: meson-a1: add pinctrl driver** — مثال على جيل تالي من الـ Meson pinctrl | [lwn.net/Articles/804174](https://lwn.net/Articles/804174/) | للفهم المقارن |
| **pinctrl: meson-s4: add pinctrl driver** | [lwn.net/Articles/879912](https://lwn.net/Articles/879912/) | للفهم المقارن |
| **pinctrl: add a generic pin config interface** | [lwn.net/Articles/468770](https://lwn.net/Articles/468770/) | فهم الـ pinconf |
| **pinctrl: add a pin config interface** | [lwn.net/Articles/471826](https://lwn.net/Articles/471826/) | فهم الـ pinconf |

---

### توثيق الـ Kernel الرسمي

الملفات دي موجودة جوه الـ kernel source tree:

```
Documentation/driver-api/pin-control.rst
```

**الرابط الأونلاين:**
- [docs.kernel.org/driver-api/pin-control.html](https://docs.kernel.org/driver-api/pin-control.html) — الـ official reference الكامل للـ pinctrl subsystem

**ملفات الكود المباشرة في الـ kernel:**

```
drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c   ← الملف الأساسي في الدرس
drivers/pinctrl/meson/pinctrl-meson-axg-pmx.h   ← الـ header مع الـ macros
drivers/pinctrl/meson/pinctrl-meson.c            ← الـ core المشترك
drivers/pinctrl/meson/pinctrl-meson.h            ← الـ shared structs
drivers/pinctrl/meson/pinctrl-meson-axg.c        ← الـ AXG pin definitions
include/linux/pinctrl/pinmux.h                   ← الـ pinmux_ops interface
include/linux/regmap.h                           ← الـ regmap API
```

**الـ Kconfig entry للـ AXG driver:**
- [github.com/torvalds/linux — drivers/pinctrl/meson/Kconfig](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/meson/Kconfig)

---

### الـ Kernel Commits المهمة

#### الـ AXG Second-Generation PMX
الـ driver اتكتب سنة 2017 من Baylibre و Amlogic. للوصول للـ commits المتعلقة:

```bash
# ابحث في git log بالـ keyword
git log --oneline --all -- drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c

# أو ابحث بالاسم
git log --oneline --author="jbrunet@baylibre.com" -- drivers/pinctrl/meson/
git log --oneline --author="xingyu.chen@amlogic.com" -- drivers/pinctrl/meson/
```

#### Patchwork discussions مهمة:
- [Rework GX pmx function — Patchwork](https://patchwork.kernel.org/project/linux-amlogic/patch/1541777218-472-9-git-send-email-narmstrong@baylibre.com/) — الـ patch اللي فصل GX عن AXG pinmux
- [Add callback for SoCs fixup — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1572004167-24150-3-git-send-email-qianggui.song@amlogic.com/) — تطور الـ architecture
- [AXG SoC cleanup series — Patchwork](https://patchwork.kernel.org/project/linux-amlogic/cover/20181122090740.29739-1-narmstrong@baylibre.com/) — الـ full cleanup series

---

### نقاشات الـ Mailing List

- **لـ LKML أرشيف** — أول patch للـ Meson pinctrl:
  [lkml.iu.edu — PATCH 1/3 pinctrl: add driver for Amlogic Meson SoCs](https://lkml.iu.edu/hypermail/linux/kernel/1410.0/04163.html)

- **نقاش الـ Meson A1 compatible** — يوضح تطور الـ architecture:
  [lkml.kernel.org — PATCH v2 0/3 pinctrl: meson-a1](https://lkml.kernel.org/lkml/1570532999-23302-4-git-send-email-qianggui.song@amlogic.com/T/)

- **lore.kernel.org** — للبحث في النقاشات الحديثة:
  ```
  https://lore.kernel.org/linux-gpio/
  ```
  ابحث بـ: `pinctrl meson axg`

---

### المشروع الرسمي للـ Amlogic Linux

**Linux for Amlogic Meson** — مشروع community لـ mainline support:
- [linux-meson.com/mainlining.html](https://linux-meson.com/mainlining.html) — تقدم الـ mainlining لكل SoC
- [linux-meson.com/hardware.html](https://linux-meson.com/hardware.html) — الـ hardware support matrix

---

### صفحات Kernel Newbies

الـ pinctrl changes بتتوثق في كل release page:

| الإصدار | الرابط | ما يهمنا |
|---------|--------|---------|
| Linux 6.15 | [kernelnewbies.org/Linux_6.15](https://kernelnewbies.org/Linux_6.15) | Amlogic pinctrl driver updates |
| Linux 4.9 | [kernelnewbies.org/Linux_4.9](https://kernelnewbies.org/Linux_4.9) | وقت دخول AXG support |
| كل الـ Changes | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) | للبحث عن Meson |

---

### صفحات eLinux.org

- [elinux.org/Amlogic](https://elinux.org/Amlogic) — الصفحة الرئيسية لكل documentation خاص بـ Amlogic SoCs على eLinux
- [elinux.org/AML_Products](https://elinux.org/AML_Products) — قائمة المنتجات وربطها بالـ SoC families

---

### كتب موصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
المتاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

الفصول المتعلقة مباشرة بالدرس:

| الفصل | الموضوع | الصلة بالكود |
|-------|---------|-------------|
| **Chapter 9** | Communicating with Hardware | فهم الـ register access اللي بتستخدمه `regmap_update_bits` |
| **Chapter 14** | The Linux Device Model | فهم `pinctrl_dev` و `platform_device` |
| **Chapter 15** | Memory Mapping and DMA | فهم الـ `reg_mux` regmap |

> ملاحظة: LDD3 قديم (kernel 2.6) وما بيغطيش الـ pinctrl subsystem مباشرة، بس فهم الـ hardware access layers فيه مهم جداً.

---

#### Linux Kernel Development — Robert Love (3rd Edition)

| الفصل | الموضوع |
|-------|---------|
| **Chapter 17** | Devices and Modules — فهم `EXPORT_SYMBOL_GPL` و module licensing |
| **Chapter 18** | Debugging — لما تحتاج تـ debug الـ pinmux |

---

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

| الفصل | الموضوع |
|-------|---------|
| **Chapter 16** | Kernel Debugging Techniques | لفهم الـ `dev_dbg` calls في الكود |
| **Chapter 15** | Customizing the Root File System | Device Tree و pinctrl في السياق العملي |

---

#### Linux Device Drivers Development — John Madieu

الأحدث والأكثر صلة بالكود الحالي:
- [oreilly.com — regmap_update_bits chapter](https://www.oreilly.com/library/view/linux-device-drivers/9781785280009/9a629439-1b11-4c67-bdb7-b6da2522e689.xhtml)
- بيغطي الـ `regmap` API بشكل عملي مع أمثلة من كود حقيقي

---

### مصادر إضافية للـ regmap API

الـ `regmap_update_bits` اللي بيستخدمها الكود بالتفصيل:

- [pandysong.github.io/blog/post/regmap](https://pandysong.github.io/blog/post/regmap/) — شرح عملي بسيط للـ regmap API
- [opensourceforu.com — regmap: Reducing the Redundancy](https://www.opensourceforu.com/2017/01/regmap-reducing-redundancy-linux-code/) — مقال ممتاز عن فلسفة الـ regmap
- [hackerbikepacker.com — Linux Drivers 3: Regmap in Detail](https://hackerbikepacker.com/linux-drivers-regmap) — شرح تفصيلي

---

### Linux Kernel Driver Database

**الـ CONFIG entry للـ AXG driver:**
- [cateee.net — CONFIG_PINCTRL_MESON_AXG](https://cateee.net/lkddb/web-lkddb/PINCTRL_MESON_AXG.html) — تاريخ ظهور الـ Kconfig option واعتماديات الـ driver

---

### كلمات البحث المفيدة

للبحث عن معلومات إضافية استخدم الـ keywords دي:

```
# للـ pinctrl subsystem عموماً
"linux pinctrl subsystem internals"
"pinmux_ops linux kernel"
"pinctrl_dev drvdata"

# للـ Meson/Amlogic تحديداً
"amlogic meson axg pinctrl"
"pinctrl-meson-axg-pmx"
"meson axg pinmux 4-bit"
"CONFIG_PINCTRL_MESON_AXG"

# للـ regmap في الـ pinctrl
"regmap_update_bits pinctrl"
"regmap mux register"

# للـ mailing list
site:lore.kernel.org "pinctrl meson axg"
site:patchwork.kernel.org "meson axg pinctrl"

# للـ Device Tree bindings
"amlogic,meson-axg-periphs-pinctrl"
"amlogic,meson-axg-aobus-pinctrl"
```

---

### ملخص المراجع الأساسية

```
┌─────────────────────────────────────────────────────────┐
│  للمبتدئ في الـ pinctrl:                                │
│    → lwn.net/Articles/468759  (The pin control subsystem)│
│    → docs.kernel.org/driver-api/pin-control.html        │
│                                                          │
│  للفهم العميق لـ Meson:                                 │
│    → lwn.net/Articles/616225  (أول Meson driver)        │
│    → lwn.net/Articles/620822  (Meson pinctrl driver)    │
│    → linux-meson.com/mainlining.html                    │
│                                                          │
│  للكود العملي:                                           │
│    → drivers/pinctrl/meson/  (كل الـ meson drivers)    │
│    → include/linux/pinctrl/pinmux.h                     │
│    → include/linux/regmap.h                             │
│                                                          │
│  للـ patches والتاريخ:                                  │
│    → patchwork.kernel.org/project/linux-amlogic/        │
│    → lore.kernel.org/linux-gpio/                        │
└─────────────────────────────────────────────────────────┘
```
## Phase 8: Writing simple module

### الهدف

هنعمل module بيستخدم **kprobe** عشان نـhook الدالة `meson_axg_pmx_set_mux` — دي أهم دالة في الـdriver لأنها بتتنادى كل ما الـkernel يحتاج يغيّر الـmux function لأي pin group على SoC الـAmlogic AXG.

بكده هنشوف في الـdmesg كل مرة بيتعمل فيها pinmux switch: اسم الـfunction، اسم الـgroup، وأرقام الـselectors.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe module: hook meson_axg_pmx_set_mux
 * Logs every pinmux switch on Amlogic AXG SoCs.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/pinctrl/pinctrl.h>   /* struct pinctrl_dev */
#include <linux/pinctrl/pinmux.h>    /* pinmux_ops */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe hook on meson_axg_pmx_set_mux to log pinmux changes");

/* ------------------------------------------------------------------ */
/* pre_handler: called BEFORE meson_axg_pmx_set_mux executes           */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * ABI (arm64 / x86-64):
     *   arg0 = pcdev      -> regs->regs[0]  (arm64) / regs->di (x86-64)
     *   arg1 = func_num   -> regs->regs[1]  (arm64) / regs->si (x86-64)
     *   arg2 = group_num  -> regs->regs[2]  (arm64) / regs->dx (x86-64)
     *
     * We use the generic pt_regs helpers provided by kprobes.
     * regs_get_kernel_argument() is architecture-independent.
     */
    unsigned long func_num  = regs_get_kernel_argument(regs, 1);
    unsigned long group_num = regs_get_kernel_argument(regs, 2);

    pr_info("[axg_pmx_probe] set_mux called: func_selector=%lu  group_selector=%lu\n",
            func_num, group_num);

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                    */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "meson_axg_pmx_set_mux", /* symbol to hook */
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
static int __init axg_pmx_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[axg_pmx_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[axg_pmx_probe] kprobe planted at %s (%p)\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit axg_pmx_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[axg_pmx_probe] kprobe removed from %s\n", kp.symbol_name);
}

module_init(axg_pmx_probe_init);
module_exit(axg_pmx_probe_exit);
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/kprobes.h` | بيجيب `struct kprobe`، `register_kprobe`، `regs_get_kernel_argument` |
| `linux/pinctrl/pinctrl.h` | تعريف `struct pinctrl_dev` اللي بتاخده الدالة المـhookd كأول argument |
| `linux/pinctrl/pinmux.h` | تعريف `struct pinmux_ops` للفهم — مش مطلوب وقت الـruntime بس بيوضح السياق |

**الـ`linux/kernel.h`** و **`linux/module.h`** أساسيين في أي module عشان `pr_info` و `MODULE_LICENSE`.

---

#### `handler_pre`

ده الـcallback اللي بيتنادى قبل ما `meson_axg_pmx_set_mux` تشتغل.

**الـ`regs_get_kernel_argument(regs, N)`** بتجيب الـargument رقم N من الـABI بطريقة portable بين x86-64 وarm64 — بنتخطى argument 0 (الـ`pcdev`) لأننا بنطبع الـselectors بس اللي هما الأهم للـmonitoring.

الـreturn value لازم يكون `0` عشان الـkprobe framework يكمّل تنفيذ الدالة الأصلية بشكل طبيعي — لو رجّعنا قيمة تانية ممكن نوقف التنفيذ وده خطر على الـpinctrl subsystem.

---

#### `struct kprobe kp`

- **`symbol_name`**: بنحدد اسم الـsymbol بالـstring بدل عنوان ثابت عشان الـkernel بيحل الـaddress تلقائياً وقت الـregistration — أكثر أمانًا مع الـKASLR.
- **`pre_handler`**: بيتشتغل قبل الدالة — كافي لطباعة الـarguments لأنهم لسه موجودين في الـregisters.

---

#### `module_init` / `module_exit`

**الـ`register_kprobe`** بيـpatch الـinstruction الأولى في `meson_axg_pmx_set_mux` بـbreakpoint، وبيحفظ الـoriginal instruction عشان يرجّعها وقت الـunregister.

**الـ`unregister_kprobe`** في الـexit ضروري جداً — لو المـmodule اتـunload من غير ما نشيل الـkprobe، الـkernel هيفضل يقفز لكود اتشال من الـmemory وده بيسبب kernel panic فوري.

---

### Makefile

```makefile
obj-m += axg_pmx_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تجربة المـmodule

```bash
# بناء الـmodule
make

# تحميله
sudo insmod axg_pmx_probe.ko

# مراقبة الـlogs (على جهاز AXG حقيقي)
sudo dmesg -w | grep axg_pmx_probe

# إخراج متوقع عند تغيير pinmux:
# [axg_pmx_probe] kprobe planted at meson_axg_pmx_set_mux (0xffffffc008xxxxxx)
# [axg_pmx_probe] set_mux called: func_selector=3  group_selector=7

# إزالة المـmodule
sudo rmmod axg_pmx_probe
```

---

### ملاحظة على الـsymbol

الدالة `meson_axg_pmx_set_mux` هي `static` في الـsource — يعني مش exported.

**الـkprobes بتقدر تـhook الـstatic functions** طالما الـsymbol موجود في `/proc/kallsyms` (وده بيحصل في الـdebug builds أو لو `CONFIG_KALLSYMS_ALL=y`).

لو الـsymbol مش موجود، ممكن نـhook `meson_axg_pmx_ops` indirectly عن طريق كتابة wrapper أو استخدام الـaddress من `/proc/kallsyms` مباشرةً مع `kp.addr`.
