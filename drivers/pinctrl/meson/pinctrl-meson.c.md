## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

**الـ** `pinctrl-meson.c` هو قلب نظام التحكم في الـ pins لشرائح **Amlogic Meson** — وهي شرائح ARM تُستخدم في صناديق Android TV وأجهزة الـ single-board computers زي Raspberry Pi لكن من Amlogic (مثل Odroid-C2، Libre Computer، إلخ).

الملف ينتمي لـ subsystem اسمه **pinctrl** (Pin Control Subsystem) في الـ Linux kernel، وده من أهم الـ subsystems لأي SoC.

---

### قصة المشكلة — تخيل معايا

تخيل إنك بتبني شريحة SoC فيها 200 طرف (pin) جسدي. كل طرف ده ممكن يعمل أكتر من وظيفة:

- الـ **GPIOX_5** مثلاً ممكن يبقى:
  - GPIO عادي (input/output)
  - أو طرف UART لبروتوكول serial
  - أو طرف I2C لبروتوكول آخر
  - أو CLK لساعة خارجية

ده اللي بيسموه **pin multiplexing (pinmux)** — كل pin بيعمل أكتر من دور، واللي بيقرر الدور ده هو bit في register معين جوه الـ SoC.

**المشكلة:** مين بيحدد إن الـ GPIOX_5 ده هيشتغل UART مش GPIO؟ — الـ kernel محتاج framework موحد يتعامل مع كل ده.

**الحل:** الـ **pinctrl subsystem** — وملف `pinctrl-meson.c` ده هو الـ core driver اللي بيجيب الـ framework ده ويطبقه على شرائح Amlogic Meson تحديداً.

---

### فكرة الـ Banks

الـ pins في Meson SoCs مش بتتحكم فيها كل واحدة على حدة. هي متجمعة في **banks** — زي بنك X، بنك Y، بنك Z، بنك AO (Always-On)، إلخ.

```
SoC Meson8:
Bank X  → GPIOX_0  .. GPIOX_21
Bank Y  → GPIOY_0  .. GPIOY_16
Bank DV → GPIODV_0 .. GPIODV_29
Bank AO → GPIOAO_0 .. GPIOAO_13   ← دائماً شغال حتى لو الـ SoC نايم
```

الـ **AO bank** (Always-On) خاص لأنه في power domain منفصل ومش بيتقفل أبداً — ده بيخليه مناسب للـ wakeup pins والـ power buttons.

---

### أنواع الـ Registers اللي بيتحكم فيها الـ Driver

كل bank عنده مجموعة registers بتتحكم في 4 أو 5 خصائص لكل pin:

| Register | الوظيفة |
|----------|---------|
| `mux` | بيحدد وظيفة الـ pin (GPIO أو UART أو I2C أو غيره) |
| `gpio` | الاتجاه (input/output) + قراءة/كتابة القيمة |
| `pull-enable` | تفعيل/تعطيل الـ pull resistor |
| `pull` | تحديد اتجاه الـ pull (up أو down) |
| `ds` | **drive strength** — قوة التيار الخارج (500µA حتى 4000µA) |

كل bit في كل register بيتحكم في pin واحد. الـ driver بيحسب رقم الـ register والـ bit بدقة عن طريق `meson_calc_reg_and_bit()`.

---

### الـ Driver بيعمل إيه بالظبط؟

الملف بيعمل **ثلاثة أشياء رئيسية** في آن واحد:

#### 1. Pin Controller (pinctrl)
بيسجل نفسه مع kernel pinctrl framework — بيعرّف الـ groups والـ functions المتاحة على كل pin.

#### 2. Pin Mux (pinmux)
بيتيح للـ drivers التانية (UART, SPI, I2C) إنها "تطلب" الـ pins اللي محتاجاها وتحولها لوظيفتها الصح.

#### 3. GPIO Controller (gpiolib)
بيسجل نفسه كـ GPIO chip — فالكود اللي بيقول `gpio_get_value(5)` بيتنفذ في النهاية عن طريق `meson_gpio_get()` في الملف ده.

---

### رحلة الـ probe — إزاي الـ driver بيبدأ

```
platform_device بيتسجل (من DT)
         ↓
meson_pinctrl_probe()
         ↓
meson_pinctrl_parse_dt()     ← بيقرأ الـ Device Tree ويعمل map للـ registers
         ↓
devm_pinctrl_register()      ← بيسجل مع pinctrl subsystem
         ↓
meson_gpiolib_register()     ← بيسجل GPIO chip مع gpiolib
```

الـ Device Tree بيحدد أماكن الـ registers (mux, gpio, pull, pull-enable, ds) عن طريق `reg-names`.

---

### الـ Pin Configuration — إزاي بتضبط pin

لو عايز تضبط pin 5 يكون pull-up:

```
DT: pinctrl-0 = <&uart_pins>;
    uart_pins: uart-pins {
        pins = "GPIOX_5";
        bias-pull-up;
    };
```

الـ kernel بيترجم ده لنداء `meson_pinconf_set()` اللي بيختار بين:
- `meson_pinconf_enable_bias()` للـ pull-up/down
- `meson_pinconf_disable_bias()` لتعطيل الـ pull
- `meson_pinconf_set_drive_strength()` لضبط قوة التيار
- `meson_pinconf_set_output()` لجعل الـ pin output

---

### الـ Regmap — ليه مش بيكتب على الـ registers مباشرة؟

الـ driver بيستخدم **regmap** بدل الكتابة المباشرة لـ MMIO registers. ده بيدي:
- حماية من الـ concurrent access
- إمكانية الـ debugging عبر debugfs
- abstraction موحد بغض النظر عن نوع الـ bus

---

### أنواع الـ SoCs المدعومة

الـ driver ده هو core مشترك بين كل الـ SoCs الآتية — كل SoC عنده ملف خاص بيعرّف الـ pins والـ banks بتاعته بس بيستخدم نفس الـ core:

| السوك | الملف |
|-------|-------|
| Meson8 / 8m2 | `pinctrl-meson8.c` |
| Meson8b | `pinctrl-meson8b.c` |
| Meson GXBB | `pinctrl-meson-gxbb.c` |
| Meson GXL | `pinctrl-meson-gxl.c` |
| Meson AXG | `pinctrl-meson-axg.c` |
| Meson G12A | `pinctrl-meson-g12a.c` |
| Meson A1 | `pinctrl-meson-a1.c` |
| Meson S4 | `pinctrl-meson-s4.c` |
| Amlogic C3 | `pinctrl-amlogic-c3.c` |
| Amlogic T7 | `pinctrl-amlogic-t7.c` |
| Amlogic A4 | `pinctrl-amlogic-a4.c` |

---

### الملفات المهمة اللي لازم تعرفها

#### Core الـ Subsystem
| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/meson/pinctrl-meson.c` | **الملف الحالي** — core مشترك لكل Meson SoCs |
| `drivers/pinctrl/meson/pinctrl-meson.h` | الـ structs والـ macros المشتركة (meson_bank, meson_pinctrl_data, إلخ) |
| `drivers/pinctrl/meson/pinctrl-meson8-pmx.c/h` | منطق الـ pinmux لجيل Meson8 القديم |
| `drivers/pinctrl/meson/pinctrl-meson-axg-pmx.c/h` | منطق الـ pinmux لجيل AXG والأحدث |

#### الـ SoC-specific Drivers
| الملف | SoC |
|-------|-----|
| `pinctrl-meson8.c` | Meson8, Meson8m2 (ARM 32-bit) |
| `pinctrl-meson-g12a.c` | G12A/G12B/SM1 (أكثر شيوعاً في Android TV boxes) |
| `pinctrl-meson-a1.c` | A1 — خاص لأن pull/gpio/ds في نفس الـ register range |

#### Kernel Framework Headers
| الملف | الدور |
|-------|-------|
| `include/linux/pinctrl/pinctrl.h` | الـ API الأساسي للـ pinctrl framework |
| `include/linux/pinctrl/pinmux.h` | الـ API الخاص بالـ pin multiplexing |
| `include/linux/pinctrl/pinconf-generic.h` | الـ pin configuration parameters الجاهزة |
| `include/linux/gpio/driver.h` | الـ gpio_chip API |

#### Device Tree Bindings
| الملف | الدور |
|-------|-------|
| `Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl.yaml` | كيفية كتابة DT لهذا الـ driver |
| `include/dt-bindings/gpio/meson8-gpio.h` | أسماء الـ pins للـ DT |
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة — ليه الـ Framework ده موجود أصلاً؟

في أي SoC حديث زي Amlogic Meson، كل pin جسمانياً على الـ chip ممكن يشتغل بأكتر من وظيفة واحدة:

- نفس الـ pin ممكن يكون GPIO عادي، أو يكون TX لـ UART، أو CLK لـ SPI، أو SCL لـ I2C.
- الـ hardware بيتحكم في الاختيار ده عن طريق **register bits** في memory-mapped registers.
- كل peripheral driver (UART, SPI, I2C...) محتاج يقول "أنا عايز الـ pin ده يشتغل في الـ mode بتاعي".

من غير framework موحد، كل driver كان هيكتب code خاص بيه يعمل ioremap ويكتب في registers بنفسه. النتيجة؟

- تعارض بين drivers على نفس الـ pin.
- كود متكرر في كل driver.
- مفيش visibility على مين بيستخدم إيه.
- الـ DT (Device Tree) مش عارف يعبر عن الـ pin state بشكل موحد.

**الـ Pinctrl Framework** جاء عشان يكون الـ single point of authority لكل حاجة تخص الـ pins.

---

### الحل — Kernel's Approach

الـ kernel قسّم المشكلة لـ 3 مناطق مستقلة:

| المنطقة | المعنى | الـ ops struct |
|---------|--------|---------------|
| **Pin Control (pctl)** | إدارة الـ groups والـ pins نفسها | `pinctrl_ops` |
| **Pin Mux (pmx)** | اختيار الـ function لكل pin/group | `pinmux_ops` |
| **Pin Config (conf)** | ضبط خصائص الـ pin (pull, drive strength...) | `pinconf_ops` |

كل driver بيسجّل نفسه عن طريق `pinctrl_desc` فيه مؤشرات للـ 3 vtables دول. الـ framework بعد كده بيكون الوسيط اللي:

1. يحل النزاع لو اتنين عايزين نفس الـ pin.
2. يترجم الـ DT إلى pin states.
3. يوفر debugfs interface لمعرفة state كل pin.

---

### التشبيه — مكتب توزيع الغرف في فندق

تخيل فندق فيه غرف (pins)، وكل غرفة ممكن تتحول لـ:
- غرفة نوم عادية (GPIO)
- قاعة اجتماعات (UART)
- استوديو تسجيل (SPI)
- مكتب عمل (I2C)

| الفندق | الـ Kernel |
|--------|-----------|
| مكتب الاستقبال | **Pinctrl Core** — الجهة اللي بتوزع وبتتحكم |
| كتالوج الغرف والأدوار | **`pinctrl_pin_desc[]`** — قائمة بكل الـ pins واسم كل واحد |
| أنواع الغرف (suite, standard...) | **pin groups** — مجموعة pins بتخدم function واحدة |
| طلب نزيل لغرفة بنوع معين | **`pinmux_ops.set_mux()`** — device driver بيطلب function معينة |
| قفل الباب على النزيل | **pin ownership** — الـ core بيمنع تعارض الـ pins |
| ديكور الغرفة (تكييف، إضاءة) | **`pinconf_ops`** — pull-up/down, drive strength |
| مدير الفندق الفعلي | **Meson pinctrl driver** — اللي بيتعامل مع الـ hardware registers |
| صاحب الفندق | **Amlogic Meson SoC** — الـ hardware نفسه |

المكتب (core) ما بيعرفش الغرف على المستوى الفيزيائي — ده شغل المدير (driver). المكتب بس بيقول "غرفة 5 محجوزة لنزيل X، مش ممكن حد تاني ياخدها".

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Consumer Drivers                        │
│   UART driver    SPI driver    I2C driver    GPIO user   │
└────────┬───────────────┬───────────────┬────────┬────────┘
         │ pinctrl_get() │               │        │ gpiod_get()
         │ pinctrl_select_state()        │        │
         ▼               ▼               ▼        ▼
┌─────────────────────────────────────────────────────────┐
│              Linux Pinctrl Core (kernel/pinctrl/)        │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ pinctrl_ops  │  │ pinmux_ops   │  │ pinconf_ops   │  │
│  │ (groups)     │  │ (functions)  │  │ (pull/drive)  │  │
│  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘  │
│         │                 │                   │          │
│         └─────────────────┼───────────────────┘          │
│                           │                              │
│              pinctrl_desc (vtable container)             │
└───────────────────────────┬──────────────────────────────┘
                            │ devm_pinctrl_register()
                            ▼
┌─────────────────────────────────────────────────────────┐
│         Meson Pinctrl Driver (pinctrl-meson.c)           │
│                                                          │
│  meson_pctrl_ops ──► get_groups / get_group_pins         │
│  meson_pinconf_ops ► set/get pull, drive, direction      │
│  (pmx_ops) ─────── ► set_mux (في الـ SoC-specific file) │
│                                                          │
│  meson_pinctrl ──► meson_pinctrl_data (static tables)    │
│       │                  │                               │
│       │           ┌──────┴──────────────────┐            │
│       │           │  pins[], groups[],       │            │
│       │           │  funcs[], banks[]        │            │
│       │           └──────────────────────────┘            │
│       │                                                   │
│  regmap ──► reg_mux / reg_gpio / reg_pull / reg_pullen   │
│             reg_ds                                        │
└───────────────────────────┬──────────────────────────────┘
                            │ ioremap + regmap_read/write
                            ▼
┌─────────────────────────────────────────────────────────┐
│       Amlogic Meson SoC — Hardware Registers             │
│                                                          │
│  PERIPHS_PIN_MUX_*   ← mux selection registers           │
│  GPIO_*              ← direction, output, input          │
│  PAD_PULL_UP_*       ← pull direction                    │
│  PAD_PULL_UP_EN_*    ← pull enable/disable               │
│  PAD_DS_*            ← drive strength (2 bits/pin)       │
└─────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — إيه الفكرة المحورية؟

الـ pinctrl framework بيقوم على 3 مفاهيم متداخلة:

#### 1. Pin
الوحدة الأساسية — رقم global وحيد لكل pad في الـ SoC.
```c
struct pinctrl_pin_desc {
    unsigned int number;  /* رقم global الـ pin */
    const char  *name;    /* اسم بشري زي "GPIOX_0" */
    void        *drv_data;
};
/* في meson: */
#define MESON_PIN(x) PINCTRL_PIN(x, #x)
```

#### 2. Group
مجموعة من الـ pins اللي بتشتغل سوا لخدمة function واحدة. مثلاً الـ UART يحتاج group من RX+TX.
```c
struct meson_pmx_group {
    const char       *name;
    const unsigned int *pins;    /* array of pin numbers */
    unsigned int      num_pins;
    const void       *data;      /* SoC-specific mux data */
};
```

#### 3. Function
الوظيفة اللي ممكن تتفعّل على group معينة. Function واحدة ممكن تبقى available على groups متعددة (مثلاً UART0 ممكن يكون على GPIOX أو GPIOH).
```c
struct meson_pmx_func {
    const char        *name;          /* "uart_ao_a" مثلاً */
    const char *const *groups;        /* أسماء الـ groups المتاحة */
    unsigned int       num_groups;
};
```

**العلاقة بينهم:**
```
Function "uart_ao_a"
    └── Group "uart_ao_a_tx"  ──► pins: [GPIOAO_0]
    └── Group "uart_ao_a_rx"  ──► pins: [GPIOAO_1]
```

---

### الـ Bank — الوحدة التنظيمية لـ Meson

الـ Meson SoCs بتقسم الـ pins في **banks** (A, B, X, Y, Z, AO, BOOT, CARD, ...). الـ bank مش مجرد اسم — هو بيحدد **إزاي تحسب الـ register والـ bit** لأي عملية على الـ pin.

```c
struct meson_bank {
    const char      *name;       /* "GPIOX" */
    unsigned int     first;      /* أول pin number في البنك */
    unsigned int     last;       /* آخر pin number */
    int              irq_first;  /* أول hwirq */
    int              irq_last;
    struct meson_reg_desc regs[MESON_NUM_REG]; /* 6 أنواع registers */
};
```

كل bank عنده **6 أنواع من الـ registers** معرّفة في `meson_reg_desc`:

```c
enum meson_reg_type {
    MESON_REG_PULLEN,  /* enable/disable pull resistor */
    MESON_REG_PULL,    /* pull up أو pull down */
    MESON_REG_DIR,     /* اتجاه الـ GPIO: input أم output */
    MESON_REG_OUT,     /* قيمة الـ output */
    MESON_REG_IN,      /* قراءة الـ input */
    MESON_REG_DS,      /* drive strength (2 bits per pin) */
    MESON_NUM_REG,
};
```

كل `meson_reg_desc` فيه `{reg, bit}` بتاعت **أول pin في الـ bank**، وبعدين الـ driver يحسب offset الـ pin الفعلي:

```c
static void meson_calc_reg_and_bit(const struct meson_bank *bank,
                                   unsigned int pin,
                                   enum meson_reg_type reg_type,
                                   unsigned int *reg, unsigned int *bit)
{
    const struct meson_reg_desc *desc = &bank->regs[reg_type];

    /* bit = start_bit + (pin - first_pin) * stride */
    *bit = (desc->bit + pin - bank->first) * meson_bit_strides[reg_type];
    /* لو الـ bit تعدى 31، انتقل للـ register التالي */
    *reg = (desc->reg + (*bit / 32)) * 4;
    *bit &= 0x1f; /* خلي الـ bit في نطاق 0-31 */
}
```

لاحظ `meson_bit_strides[]`:
```c
static const unsigned int meson_bit_strides[] = {
    1, 1, 1, 1, 1, 2, 1
    /* DS بيستخدم 2 bits لكل pin، الباقي 1 bit */
};
```

الـ drive strength بيحتاج 2 bits لأن فيه 4 قيم (500uA, 2500uA, 3000uA, 4000uA).

---

### الـ Register Map في Meson

كل pin controller عنده regmaps منفصلة، بتتبنى من الـ DT:

```
gpio_np (device tree node)
├── reg-names = "mux"         → pc->reg_mux    (pin muxing)
├── reg-names = "gpio"        → pc->reg_gpio   (dir + out + in)
├── reg-names = "pull"        → pc->reg_pull   (pull up/down)
├── reg-names = "pull-enable" → pc->reg_pullen (pull enable)
└── reg-names = "ds"          → pc->reg_ds     (drive strength, optional)
```

في بعض الـ SoCs القديمة (meson8 AO bus)، `reg_pull` و`reg_pullen` نفس الـ register. في A1، كل حاجة في `reg_gpio`. الـ driver بيتعامل مع ده عن طريق الـ `parse_dt` callback في `meson_pinctrl_data`.

---

### الـ Struct Relationships — الصورة الكاملة

```
meson_pinctrl  (instance per controller)
│
├── dev          ──► struct device
├── pcdev        ──► struct pinctrl_dev (owned by framework)
├── desc         ──► struct pinctrl_desc
│       ├── pctlops  ──► meson_pctrl_ops    (groups query)
│       ├── pmxops   ──► (SoC-specific)     (mux set)
│       ├── confops  ──► meson_pinconf_ops  (pull/drive)
│       ├── pins     ──► pinctrl_pin_desc[] (flat pin list)
│       └── npins
│
├── data         ──► meson_pinctrl_data  (static, per-SoC)
│       ├── pins[]    ──► pinctrl_pin_desc[]
│       ├── groups[]  ──► meson_pmx_group[]
│       ├── funcs[]   ──► meson_pmx_func[]
│       ├── banks[]   ──► meson_bank[]
│       └── pmx_ops   ──► pinmux_ops (SoC-specific mux logic)
│
├── reg_mux      ──► regmap  (mux registers)
├── reg_gpio     ──► regmap  (GPIO dir/out/in)
├── reg_pull     ──► regmap  (pull direction)
├── reg_pullen   ──► regmap  (pull enable)
├── reg_ds       ──► regmap  (drive strength, nullable)
│
└── chip         ──► gpio_chip  (gpiolib integration)
        ├── get_direction  ──► meson_gpio_get_direction
        ├── direction_input ──► meson_gpio_direction_input
        ├── direction_output ──► meson_gpio_direction_output
        ├── get  ──► meson_gpio_get
        └── set  ──► meson_gpio_set
```

---

### الـ Vtables — الـ Ops الثلاث

#### pinctrl_ops — إدارة الـ Groups
```c
static const struct pinctrl_ops meson_pctrl_ops = {
    .get_groups_count = meson_get_groups_count, /* كام group؟ */
    .get_group_name   = meson_get_group_name,   /* اسم group رقم N */
    .get_group_pins   = meson_get_group_pins,   /* pins الـ group */
    .dt_node_to_map   = pinconf_generic_dt_node_to_map_all, /* parse DT */
    .dt_free_map      = pinctrl_utils_free_map,
    .pin_dbg_show     = meson_pin_dbg_show,
};
```

#### pinconf_ops — ضبط خصائص الـ Pin
```c
static const struct pinconf_ops meson_pinconf_ops = {
    .pin_config_get       = meson_pinconf_get,
    .pin_config_set       = meson_pinconf_set,
    .pin_config_group_get = meson_pinconf_group_get,
    .pin_config_group_set = meson_pinconf_group_set,
    .is_generic           = true, /* بيستخدم generic pinconf encoding */
};
```

الـ `pin_config_set` بتحوّل `PIN_CONFIG_*` enums لـ register writes:

```c
/* مثال: تفعيل pull-up على pin */
case PIN_CONFIG_BIAS_PULL_UP:
    ret = meson_pinconf_enable_bias(pc, pin, true);
    /* ده بيكتب في reg_pull ثم reg_pullen */
```

#### pinmux_ops — اختيار الـ Function
الـ `pmx_ops` مش معرّف مباشرة في هذا الملف، بيجي من `pc->data->pmx_ops` — يعني كل SoC بيحدد `set_mux` الخاص بيه (في ملفات زي `pinctrl-meson8.c` أو `pinctrl-meson-axg-pmx.c`). الـ common functions بس (count/name/groups) معرّفة هنا:

```c
int meson_pmx_get_funcs_count(struct pinctrl_dev *pcdev) { ... }
EXPORT_SYMBOL_GPL(meson_pmx_get_funcs_count);

const char *meson_pmx_get_func_name(struct pinctrl_dev *pcdev, unsigned selector) { ... }
EXPORT_SYMBOL_GPL(meson_pmx_get_func_name);

int meson_pmx_get_groups(struct pinctrl_dev *pcdev, unsigned selector, ...) { ... }
EXPORT_SYMBOL_GPL(meson_pmx_get_groups);
```

---

### إيه اللي الـ Framework بيملكه vs. ما بيفوّضه للـ Driver؟

| المهمة | الـ Framework يملكه | يفوّضه للـ Driver |
|--------|---------------------|-------------------|
| حل تعارض الـ pins بين drivers | نعم | لا |
| ترجمة DT `pinctrl-*` properties | نعم (generic) | DT-to-map callback |
| debugfs (`/sys/kernel/debug/pinctrl/`) | نعم | pin_dbg_show فقط |
| pin ownership tracking | نعم | لا |
| الكتابة الفعلية في register الـ mux | لا | `pmx_ops.set_mux()` |
| حساب أي register وأي bit لكل pin | لا | `meson_calc_reg_and_bit()` |
| إدارة pull-up/down فعلياً | لا | `meson_pinconf_enable_bias()` |
| معرفة الـ banks وحدودها | لا | `meson_get_bank()` |
| تسجيل الـ gpio_chip | لا | `meson_gpiolib_register()` |
| ربط الـ regmap بالـ DT address ranges | لا | `meson_pinctrl_parse_dt()` |

---

### الـ GPIO Integration — الـ Subsystem اللي لازم تعرفه

الـ **GPIO subsystem** (gpiolib) هو الـ framework المسؤول عن تصدير الـ pins كـ `/dev/gpiochipN` و`gpiod_get()`. هو منفصل عن الـ pinctrl لكن مرتبط بيه:

- لما driver يعمل `gpiod_get()` على pin، الـ gpiolib بيتصل بـ `pinmux_ops.gpio_request_enable()` عشان يعلم الـ pinctrl إن الـ pin اتحجز كـ GPIO.
- الـ meson driver بيسجّل `gpio_chip` بـ `gpiochip_add_data()` ويربطه بالـ pinctrl device عن طريق `gpiochip_generic_request` / `gpiochip_generic_free`.

```c
static int meson_gpiolib_register(struct meson_pinctrl *pc)
{
    pc->chip.request       = gpiochip_generic_request;  /* يتصل بـ pinctrl */
    pc->chip.free          = gpiochip_generic_free;
    pc->chip.get_direction = meson_gpio_get_direction;  /* يقرأ reg_gpio */
    pc->chip.direction_input  = meson_gpio_direction_input;
    pc->chip.direction_output = meson_gpio_direction_output;
    pc->chip.get           = meson_gpio_get;
    pc->chip.set           = meson_gpio_set;
    /* ... */
    return gpiochip_add_data(&pc->chip, pc);
}
```

الـ `regmap` subsystem — لازم تعرفه كمان: هو abstraction فوق الـ `ioremap`/`readl`/`writel`، بيوفر caching وdebugging وatomic read-modify-write عن طريق `regmap_update_bits()`.

---

### تسلسل الـ Probe — من مين لمين

```
platform_device probe
        │
        ▼
meson_pinctrl_probe()
        │
        ├─► devm_kzalloc(meson_pinctrl)
        │
        ├─► of_device_get_match_data()  ← يجيب meson_pinctrl_data من الـ DT match
        │
        ├─► meson_pinctrl_parse_dt()
        │       ├── gpiochip_node_get_first()  ← يلاقي الـ gpio child node
        │       ├── meson_map_resource("mux")  ← ioremap + regmap init
        │       ├── meson_map_resource("gpio")
        │       ├── meson_map_resource("pull")
        │       ├── meson_map_resource("pull-enable")
        │       ├── meson_map_resource("ds")   ← optional
        │       └── data->parse_dt(pc)         ← SoC-specific fixups
        │
        ├─► تهيئة pinctrl_desc
        │       ├── pctlops = &meson_pctrl_ops
        │       ├── pmxops  = pc->data->pmx_ops  (SoC-specific)
        │       ├── confops = &meson_pinconf_ops
        │       ├── pins    = pc->data->pins
        │       └── npins   = pc->data->num_pins
        │
        ├─► devm_pinctrl_register()  ← يسجّل في الـ pinctrl core
        │
        └─► meson_gpiolib_register() ← يسجّل gpio_chip في الـ gpiolib
```

---

### مثال حي — ضبط UART على Meson G12A

لما الـ UART driver بيقول "أنا عايز UART_AO_A":

1. الـ pinctrl core بيبحث عن function اسمها `uart_ao_a`.
2. بيلاقيها في `meson_pmx_func[]` وبيشوف groups بتاعتها.
3. بيستدعي `pmx_ops.set_mux(func_selector, group_selector)`.
4. الـ SoC-specific driver بيكتب الـ bit المناسب في `reg_mux`.
5. الـ pin بقى يشتغل كـ UART TX/RX بدل GPIO.

لو حد بعد كده حاول يعمل `gpiod_get()` على نفس الـ pin، الـ framework هيرجع `-EBUSY` تلقائياً.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Enums والـ Flags الأساسية

#### `enum meson_reg_type` — أنواع الـ Registers

| القيمة | Index | وظيفتها |
|---|---|---|
| `MESON_REG_PULLEN` | 0 | تفعيل/تعطيل الـ pull resistor |
| `MESON_REG_PULL` | 1 | اتجاه الـ pull: up أو down |
| `MESON_REG_DIR` | 2 | اتجاه الـ GPIO: input أو output |
| `MESON_REG_OUT` | 3 | قيمة الـ output: high أو low |
| `MESON_REG_IN` | 4 | قراءة قيمة الـ input |
| `MESON_REG_DS` | 5 | drive strength (2 bits/pin) |
| `MESON_NUM_REG` | 6 | عدد الأنواع — sentinel للـ array size |

**الـ `meson_bit_strides[]`** يحدد عدد الـ bits لكل pin لكل نوع:

```c
static const unsigned int meson_bit_strides[] = {
    1, 1, 1, 1, 1, 2, 1
    // PULLEN PULL DIR OUT IN  DS  (sentinel)
};
```

الـ `MESON_REG_DS` يأخذ **2 bits** لأنه يدعم 4 مستويات.

---

#### `enum meson_pinconf_drv` — مستويات الـ Drive Strength

| القيمة | Code | التيار |
|---|---|---|
| `MESON_PINCONF_DRV_500UA` | 0 | 0.5 mA |
| `MESON_PINCONF_DRV_2500UA` | 1 | 2.5 mA |
| `MESON_PINCONF_DRV_3000UA` | 2 | 3.0 mA |
| `MESON_PINCONF_DRV_4000UA` | 3 | 4.0 mA |

---

#### `pin_config_param` — المعاملات المدعومة (من kernel pinconf-generic)

| المعامل | الوظيفة | الـ arg |
|---|---|---|
| `PIN_CONFIG_BIAS_DISABLE` | إلغاء الـ pull | لا يوجد |
| `PIN_CONFIG_BIAS_PULL_UP` | pull-up | لا يوجد |
| `PIN_CONFIG_BIAS_PULL_DOWN` | pull-down | لا يوجد |
| `PIN_CONFIG_DRIVE_STRENGTH_UA` | قوة الـ output | تيار بالـ µA |
| `PIN_CONFIG_OUTPUT_ENABLE` | تفعيل output mode | 0 أو 1 |
| `PIN_CONFIG_LEVEL` | تعيين قيمة output | 0 أو 1 |

---

### الـ Structs الأساسية

---

#### `struct meson_reg_desc`

**الغرض:** يصف موقع أي خاصية لأول pin في الـ bank داخل الـ regmap.

```c
struct meson_reg_desc {
    unsigned int reg;  /* register offset (بالـ words) في الـ regmap */
    unsigned int bit;  /* رقم الـ bit لأول pin في الـ bank */
};
```

**كيف تُستخدم:** دايمًا جزء من `meson_bank.regs[]` — عمرها ما تُستخدم لوحدها.

---

#### `struct meson_bank`

**الغرض:** يصف bank كاملة — مجموعة pins متتالية تشترك في نفس register layout.

```c
struct meson_bank {
    const char *name;        /* اسم الـ bank: "X", "Y", "AO", إلخ */
    unsigned int first;      /* رقم أول pin (global pin number) */
    unsigned int last;       /* رقم آخر pin */
    int irq_first;           /* أول hwirq في الـ bank (-1 إذا مش مدعوم) */
    int irq_last;            /* آخر hwirq */
    struct meson_reg_desc regs[MESON_NUM_REG]; /* descriptor لكل نوع register */
};
```

**مثال تعريف bank بالـ macro:**

```c
BANK("X", GPIOX_0, GPIOX_19, 0, 19,
      /* PULLEN reg,bit */ 3, 0,
      /* PULL    reg,bit */ 3, 0,
      /* DIR     reg,bit */ 0, 0,
      /* OUT     reg,bit */ 0, 0,
      /* IN      reg,bit */ 1, 0);
```

**العلاقة مع باقي الـ structs:**
- `meson_pinctrl_data` تحتوي على array من `meson_bank`
- `meson_get_bank()` يبحث فيها بالـ pin number
- `meson_calc_reg_and_bit()` تستخدم `regs[]` داخلها لحساب الـ register address

---

#### `struct meson_pmx_group`

**الغرض:** يمثل مجموعة pins تُفعَّل معًا لأداء وظيفة معينة (pinmux group).

```c
struct meson_pmx_group {
    const char *name;           /* اسم الـ group: "i2c0_sda_x_pins" مثلًا */
    const unsigned int *pins;   /* array of global pin numbers */
    unsigned int num_pins;      /* حجم الـ array */
    const void *data;           /* بيانات خاصة بالـ SoC (mux register/bit) */
};
```

**العلاقة:**
- `meson_pinctrl_data.groups[]` تحتوي عليها
- `meson_pmx_func` تشير إليها بالاسم (string references)

---

#### `struct meson_pmx_func`

**الغرض:** يمثل وظيفة كاملة (مثل `i2c0`, `spi0`, `uart_a`) تتكون من groups.

```c
struct meson_pmx_func {
    const char *name;              /* اسم الـ function */
    const char * const *groups;   /* array of group names */
    unsigned int num_groups;       /* عدد الـ groups */
};
```

**العلاقة:**
- `meson_pinctrl_data.funcs[]` تحتوي عليها
- الـ pinmux core يطلب الـ functions ويربطها بالـ groups

---

#### `struct meson_pinctrl_data`

**الغرض:** البيانات الثابتة (const) الخاصة بكل SoC — تعريف كامل للـ pin controller.

```c
struct meson_pinctrl_data {
    const char *name;                    /* اسم الـ controller */
    const struct pinctrl_pin_desc *pins; /* array of all pins */
    const struct meson_pmx_group *groups;/* array of all groups */
    const struct meson_pmx_func *funcs;  /* array of all functions */
    unsigned int num_pins;
    unsigned int num_groups;
    unsigned int num_funcs;
    const struct meson_bank *banks;      /* array of banks */
    unsigned int num_banks;
    const struct pinmux_ops *pmx_ops;   /* SoC-specific pinmux ops */
    const void *pmx_data;               /* SoC-specific mux data */
    int (*parse_dt)(struct meson_pinctrl *pc); /* optional extra DT parsing */
};
```

**ملاحظة:** هذا الـ struct هو **read-only** — كل SoC يعرّف instance منه كـ `static const` في ملفه الخاص (مثل `pinctrl-meson8.c`).

---

#### `struct meson_pinctrl`

**الغرض:** الـ runtime state الكامل للـ pin controller instance — المحور الرئيسي للـ driver.

```c
struct meson_pinctrl {
    struct device *dev;              /* الـ platform device */
    struct pinctrl_dev *pcdev;       /* handle من الـ pinctrl core */
    struct pinctrl_desc desc;        /* descriptor مُسجَّل في الـ core */
    struct meson_pinctrl_data *data; /* بيانات الـ SoC (const) */
    struct regmap *reg_mux;          /* regmap لـ pin muxing */
    struct regmap *reg_pullen;       /* regmap لـ pull enable */
    struct regmap *reg_pull;         /* regmap لـ pull direction */
    struct regmap *reg_gpio;         /* regmap لـ GPIO dir/in/out */
    struct regmap *reg_ds;           /* regmap لـ drive strength (اختياري) */
    struct gpio_chip chip;           /* الـ GPIO chip المُسجَّل في gpiolib */
    struct fwnode_handle *fwnode;    /* firmware node للـ GPIO child */
};
```

**هذا الـ struct هو قلب الـ driver** — كل function تمر من خلاله.

---

#### `struct pinctrl_desc` (من kernel)

**الغرض:** الـ descriptor اللي يُسجَّل في الـ pinctrl subsystem.

| الحقل | النوع | القيمة في Meson |
|---|---|---|
| `name` | `const char *` | `"pinctrl-meson"` |
| `pins` | `pinctrl_pin_desc[]` | من `pc->data->pins` |
| `npins` | `unsigned int` | من `pc->data->num_pins` |
| `pctlops` | `pinctrl_ops *` | `&meson_pctrl_ops` |
| `pmxops` | `pinmux_ops *` | من `pc->data->pmx_ops` |
| `confops` | `pinconf_ops *` | `&meson_pinconf_ops` |

---

### مخطط علاقات الـ Structs (ASCII)

```
  ┌─────────────────────────────────────────────────────┐
  │              meson_pinctrl  (runtime state)          │
  │                                                     │
  │  dev ──────────────────────────► struct device      │
  │  pcdev ─────────────────────────► pinctrl_dev       │
  │  desc ──────────────────────────► pinctrl_desc      │
  │         │                          │                │
  │         │                          ├─ pctlops       │
  │         │                          ├─ pmxops        │
  │         │                          └─ confops       │
  │  data ──┼──────────────────────────► meson_pinctrl_data (const)
  │         │                            │              │
  │  reg_mux ──► regmap (MMIO)           ├─ pins[]      │
  │  reg_pullen ► regmap (MMIO)          ├─ groups[] ──► meson_pmx_group[]
  │  reg_pull ──► regmap (MMIO)          ├─ funcs[]  ──► meson_pmx_func[]
  │  reg_gpio ──► regmap (MMIO)          ├─ banks[]  ──► meson_bank[]
  │  reg_ds ────► regmap (MMIO)          │                │
  │  chip ──────► gpio_chip              │                └─ regs[6] ──► meson_reg_desc[6]
  │  fwnode ────► fwnode_handle          └─ pmx_ops       │
  └─────────────────────────────────────────────────────┘
                                                          │
                              ┌───────────────────────────┘
                              ▼
                    meson_bank {
                      first, last        ← pin range
                      irq_first/last     ← IRQ range
                      regs[MESON_NUM_REG] = {
                        [PULLEN] = {reg, bit}
                        [PULL]   = {reg, bit}
                        [DIR]    = {reg, bit}
                        [OUT]    = {reg, bit}
                        [IN]     = {reg, bit}
                        [DS]     = {reg, bit}
                      }
                    }
```

---

### مخطط الـ Register Address Calculation

```
pin = 42,  bank.first = 32,  MESON_REG_DS

desc = bank->regs[MESON_REG_DS]   →  {reg=10, bit=0}

bit_pos = (desc.bit + pin - bank.first) * stride
        = (0 + 42 - 32) * 2
        = 10 * 2 = 20

reg_offset = (desc.reg + (bit_pos / 32)) * 4
           = (10 + 0) * 4 = 40   ← byte offset في الـ regmap

final_bit  = bit_pos & 0x1f = 20

regmap_update_bits(pc->reg_ds, 40, 0x3 << 20, ds_val << 20)
```

---

### دورة الحياة — Lifecycle Diagram

```
Boot / Driver Load
        │
        ▼
meson_pinctrl_probe(pdev)
        │
        ├─① devm_kzalloc → allocate meson_pinctrl (pc)
        │
        ├─② pc->data = of_device_get_match_data()
        │          └── يجيب الـ meson_pinctrl_data الخاص بالـ SoC
        │
        ├─③ meson_pinctrl_parse_dt(pc)
        │       ├── gpiochip_node_get_first() → pc->fwnode
        │       ├── meson_map_resource("mux")       → pc->reg_mux
        │       ├── meson_map_resource("gpio")      → pc->reg_gpio
        │       ├── meson_map_resource("pull")      → pc->reg_pull
        │       ├── meson_map_resource("pull-enable")→ pc->reg_pullen
        │       ├── meson_map_resource("ds")        → pc->reg_ds (optional)
        │       └── pc->data->parse_dt(pc)          → SoC-specific fixups
        │                  ├── meson8_aobus: reg_pullen = reg_pull
        │                  └── meson_a1:    reg_pull/pullen/ds = reg_gpio
        │
        ├─④ fill pc->desc (pinctrl_desc)
        │       ├── pctlops  = &meson_pctrl_ops
        │       ├── pmxops   = pc->data->pmx_ops
        │       └── confops  = &meson_pinconf_ops
        │
        ├─⑤ devm_pinctrl_register(dev, &pc->desc, pc)
        │       └── يرجع pc->pcdev (pinctrl_dev handle)
        │
        └─⑥ meson_gpiolib_register(pc)
                ├── fill pc->chip (gpio_chip)
                └── gpiochip_add_data(&pc->chip, pc)

                        ▼
              DRIVER FULLY OPERATIONAL
                        │
                 (device removed)
                        │
                        ▼
        devm cleanup:
          - pinctrl_unregister (auto via devm)
          - gpiochip_remove (auto via devm)
          - iounmap all regmaps (auto via devm)
```

---

### مخطط تدفق الـ Calls — Call Flow Diagrams

#### 1. تغيير Pinmux لـ function (مثلاً تفعيل UART)

```
device driver requests pinctrl state "default"
    │
    ▼
pinctrl_select_state()              ← pinctrl core
    │
    ▼
pinmux_enable_setting()             ← pinctrl core
    │
    ▼
pc->data->pmx_ops->set_mux()        ← SoC-specific (e.g. meson8_pmx_set)
    │
    ├── meson_pmx_get_groups()      ← يجيب groups الـ function
    │       └── pc->data->funcs[selector].groups
    │
    └── for each pin in group:
            write mux bits to pc->reg_mux via regmap_update_bits()
                └── hardware: pin now routes UART signal
```

#### 2. قراءة GPIO input

```
gpio_get_value(gpio_num)
    │
    ▼
meson_gpio_get(chip, gpio)
    │
    ├── meson_get_bank(pc, gpio, &bank)
    │       └── linear search: bank.first <= gpio <= bank.last
    │
    ├── meson_calc_reg_and_bit(bank, gpio, MESON_REG_IN, &reg, &bit)
    │       └── calculates byte offset + bit position
    │
    └── regmap_read(pc->reg_gpio, reg, &val)
            └── returns !!(val & BIT(bit))
```

#### 3. ضبط Pin Configuration (pull-up)

```
pinctrl core calls meson_pinconf_set(pcdev, pin, configs, num)
    │
    ├── param = PIN_CONFIG_BIAS_PULL_UP
    │
    └── meson_pinconf_enable_bias(pc, pin, pull_up=true)
            │
            ├── meson_get_bank(pc, pin, &bank)
            │
            ├── meson_calc_reg_and_bit(..., MESON_REG_PULL, &reg, &bit)
            │       └── regmap_update_bits(pc->reg_pull, reg, BIT(bit), BIT(bit))
            │               ← set pull direction = UP
            │
            └── meson_calc_reg_and_bit(..., MESON_REG_PULLEN, &reg, &bit)
                    └── regmap_update_bits(pc->reg_pullen, reg, BIT(bit), BIT(bit))
                            ← enable the pull resistor
```

#### 4. تعيين GPIO كـ output وكتابة قيمة

```
gpio_direction_output(gpio, value)
    │
    ▼
meson_gpio_direction_output(chip, gpio, value)
    │
    └── meson_pinconf_set_output_drive(pc, gpio, value)
            │
            ├── meson_pinconf_set_output(pc, pin, true)
            │       └── meson_pinconf_set_gpio_bit(pc, pin, MESON_REG_DIR, false)
            │               ← DIR=0 means output (inverted logic!)
            │               └── regmap_update_bits(pc->reg_gpio, reg, BIT(bit), 0)
            │
            └── meson_pinconf_set_drive(pc, pin, high=value)
                    └── meson_pinconf_set_gpio_bit(pc, pin, MESON_REG_OUT, value)
                            └── regmap_update_bits(pc->reg_gpio, reg, BIT(bit), ...)
```

---

### الـ Ops Tables — جداول العمليات

#### `meson_pctrl_ops` — عمليات الـ Pin Control

| الـ callback | الـ function | الوظيفة |
|---|---|---|
| `get_groups_count` | `meson_get_groups_count` | عدد الـ groups |
| `get_group_name` | `meson_get_group_name` | اسم group بالـ index |
| `get_group_pins` | `meson_get_group_pins` | pins الـ group |
| `dt_node_to_map` | `pinconf_generic_dt_node_to_map_all` | parse DT config |
| `dt_free_map` | `pinctrl_utils_free_map` | free DT map |
| `pin_dbg_show` | `meson_pin_dbg_show` | debugfs display |

#### `meson_pinconf_ops` — عمليات الـ Pin Configuration

| الـ callback | الـ function | الوظيفة |
|---|---|---|
| `pin_config_get` | `meson_pinconf_get` | اقرأ config لـ pin واحد |
| `pin_config_set` | `meson_pinconf_set` | اكتب config لـ pin واحد |
| `pin_config_group_get` | `meson_pinconf_group_get` | دايمًا `-ENOTSUPP` |
| `pin_config_group_set` | `meson_pinconf_group_set` | iterate على كل pins الـ group |
| `is_generic` | `true` | يستخدم generic pinconf framework |

---

### استراتيجية الـ Locking

الـ driver **لا يعرّف locks خاصة به** — يعتمد بالكامل على الـ locking المدمج في:

| الطبقة | الـ Lock | يحمي |
|---|---|---|
| **regmap** | `regmap->lock` (mutex أو spinlock) | كل قراءة/كتابة على الـ MMIO registers |
| **pinctrl core** | `pctldev->mutex` | قوائم الـ settings وتغيير الـ states |
| **gpiolib** | `gpio_device->srcu` + `gpio_chip` locks | عمليات الـ GPIO |

#### مسار الـ Lock عند عملية GPIO:

```
user calls gpio_set_value()
    │
    └── gpiolib acquires its internal lock
            │
            └── meson_gpio_set() called
                    │
                    └── regmap_update_bits()
                            │
                            └── regmap acquires regmap->lock
                                    │
                                    └── read-modify-write MMIO register
                                    └── regmap releases regmap->lock
                        gpiolib releases lock
```

**لا يوجد lock ordering مباشر** في الـ driver نفسه لأن:
- الـ regmap يُدار من داخل الـ subsystems
- الـ `can_sleep = true` في الـ gpio_chip يسمح باستخدام mutex (وليس spinlock) في الـ regmap

#### خاصية مهمة في الـ AO Bank:

الـ SoCs القديمة (meson8 وما قبلها) لديها الـ AO bank في **always-on power domain** — تستخدم register set مختلف، لكن **نفس locking strategy** لأن التمييز يحدث فقط من خلال `pc->reg_*` pointers المختلفة عند الـ probe.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — جميع الـ Functions

#### Category 1: Core Helpers

| Function | Signature Summary | الغرض |
|---|---|---|
| `meson_get_bank` | `(pc, pin, **bank) → int` | إيجاد الـ bank اللي بيحتوي على pin معين |
| `meson_calc_reg_and_bit` | `(bank, pin, reg_type, *reg, *bit)` | حساب offset الـ register والـ bit لأي pin |

#### Category 2: Pinctrl Ops (مربوطة بـ `meson_pctrl_ops`)

| Function | الغرض |
|---|---|
| `meson_get_groups_count` | بترجع عدد الـ groups |
| `meson_get_group_name` | بترجع اسم group بالـ selector |
| `meson_get_group_pins` | بترجع array الـ pins اللي في group |
| `meson_pin_dbg_show` | debugfs hook للـ pin |

#### Category 3: Pinmux Ops (exported — مربوطة بالـ SoC-specific drivers)

| Function | الغرض |
|---|---|
| `meson_pmx_get_funcs_count` | عدد الـ mux functions |
| `meson_pmx_get_func_name` | اسم function بالـ selector |
| `meson_pmx_get_groups` | groups بتاعة function معينة |

#### Category 4: Pinconf Internal Helpers

| Function | الغرض |
|---|---|
| `meson_pinconf_set_gpio_bit` | كتابة bit واحد في reg_gpio |
| `meson_pinconf_get_gpio_bit` | قراءة bit واحد من reg_gpio |
| `meson_pinconf_set_output` | تحديد الاتجاه output |
| `meson_pinconf_get_output` | قراءة اتجاه الـ pin |
| `meson_pinconf_set_drive` | كتابة قيمة output |
| `meson_pinconf_get_drive` | قراءة قيمة output |
| `meson_pinconf_set_output_drive` | set direction=output + set value |
| `meson_pinconf_disable_bias` | تعطيل الـ pull resistor |
| `meson_pinconf_enable_bias` | تفعيل pull-up أو pull-down |
| `meson_pinconf_set_drive_strength` | ضبط drive strength بالـ µA |
| `meson_pinconf_get_pull` | قراءة حالة الـ pull |
| `meson_pinconf_get_drive_strength` | قراءة drive strength الحالية |

#### Category 5: Pinconf Ops (مربوطة بـ `meson_pinconf_ops`)

| Function | الغرض |
|---|---|
| `meson_pinconf_set` | entry point لـ pinconf_set من الـ subsystem |
| `meson_pinconf_get` | entry point لـ pinconf_get من الـ subsystem |
| `meson_pinconf_group_set` | تطبيق config على كل pins الـ group |
| `meson_pinconf_group_get` | غير مدعوم — بترجع `-ENOTSUPP` |

#### Category 6: GPIO Chip Ops

| Function | الغرض |
|---|---|
| `meson_gpio_get_direction` | بترجع `GPIO_LINE_DIRECTION_IN/OUT` |
| `meson_gpio_direction_input` | ضبط الـ pin كـ input |
| `meson_gpio_direction_output` | ضبط الـ pin كـ output + كتابة قيمة |
| `meson_gpio_set` | كتابة قيمة على output pin |
| `meson_gpio_get` | قراءة قيمة input pin |
| `meson_gpiolib_register` | تسجيل الـ gpio_chip في gpiolib |

#### Category 7: Init / Probe

| Function | الغرض |
|---|---|
| `meson_map_resource` | map register range من DT إلى regmap |
| `meson_pinctrl_parse_dt` | parse الـ DT node وتهيئة الـ regmaps |
| `meson8_aobus_parse_dt_extra` | workaround لـ AO bus في Meson8 |
| `meson_a1_parse_dt_extra` | workaround لـ A1 SoC (unified regs) |
| `meson_pinctrl_probe` | entry point الرئيسي للـ driver |

---

### Category 1: Core Helpers

الوظيفة الأساسية لهاتين الـ functions هي ترجمة رقم الـ pin المجرد إلى عنوان فعلي في الـ regmap. كل العمليات على الـ pins تمر عليهم.

---

#### `meson_get_bank`

```c
static int meson_get_bank(struct meson_pinctrl *pc,
                          unsigned int pin,
                          const struct meson_bank **bank);
```

بتعمل linear scan على array الـ banks المعرفة في `pc->data->banks` وبتدور على الـ bank اللي الـ `pin` يقع في نطاقه بين `bank->first` و `bank->last`. لما تلاقيه، بتكتب pointer الـ bank في `*bank` وبترجع صفر.

**Parameters:**
- `pc` — الـ pinctrl instance الخاص بالـ SoC
- `pin` — رقم الـ pin العالمي (global pin number)
- `bank` — output pointer، بيتحط فيه pointer على الـ `meson_bank` اللي فيه الـ pin

**Return:** `0` لو نجح، `-EINVAL` لو الـ pin مش موجود في أي bank.

**Key details:** الـ scan بسيط O(n) — عدد الـ banks صغير جداً (7-8 بالأكثر للـ SoC الواحد). مفيش locking هنا لأن الـ `banks` array هي read-only بعد الـ probe.

**Caller context:** بتُستدعى من أي function تحتاج تشتغل على pin معين قبل ما تبدأ تكتب/تقرأ registers.

---

#### `meson_calc_reg_and_bit`

```c
static void meson_calc_reg_and_bit(const struct meson_bank *bank,
                                   unsigned int pin,
                                   enum meson_reg_type reg_type,
                                   unsigned int *reg,
                                   unsigned int *bit);
```

بتحسب الـ byte offset للـ register والـ bit index داخله لـ pin معين ونوع register معين (pull، dir، out، in، ds...). الحساب بيأخذ في الاعتبار الـ `meson_bit_strides` — الـ DS register بيستخدم 2 bits لكل pin بدل 1.

**Parameters:**
- `bank` — الـ bank اللي فيه الـ pin
- `pin` — رقم الـ pin العالمي
- `reg_type` — `enum meson_reg_type` (PULLEN, PULL, DIR, OUT, IN, DS)
- `reg` — output: byte offset في الـ regmap
- `bit` — output: bit index داخل الـ register (0-31)

**Return:** void — الـ output في `*reg` و `*bit`.

**Key details:**

```
/* Pseudocode */
desc = bank->regs[reg_type]          // base reg + base bit للـ bank
raw_bit = (desc.bit + (pin - bank->first)) * stride[reg_type]
*reg  = (desc.reg + raw_bit / 32) * 4   // byte offset
*bit  = raw_bit % 32                    // bit position في الـ word
```

الضرب في 4 لتحويل word offset لـ byte offset لأن الـ `regmap` config بيستخدم `reg_stride = 4` (32-bit registers).

**الـ `meson_bit_strides`:** كل entries بقيمة 1 ماعدا `MESON_REG_DS` (index 5) اللي قيمته 2، لأن الـ drive strength محتاج 2-bit field لكل pin.

---

### Category 2: Pinctrl Ops

هاتي الـ functions هي implementation الـ vtable `pinctrl_ops` اللي بيطلبها الـ pinctrl core من أي driver. بتدي الـ core access للـ metadata بتاعة الـ groups.

---

#### `meson_get_groups_count`

```c
static int meson_get_groups_count(struct pinctrl_dev *pcdev);
```

بترجع عدد الـ pinmux groups المعرفة في البيانات الثابتة للـ SoC.

**Parameters:** `pcdev` — الـ pinctrl device handle.

**Return:** `pc->data->num_groups` — عدد صحيح موجب.

**Caller:** الـ pinctrl core وقت الـ enumeration وال debugfs.

---

#### `meson_get_group_name`

```c
static const char *meson_get_group_name(struct pinctrl_dev *pcdev,
                                        unsigned selector);
```

بترجع اسم الـ group بالـ index `selector` من الـ `groups` array.

**Parameters:** `selector` — index الـ group.

**Return:** pointer على string ثابت في الـ `.rodata`.

---

#### `meson_get_group_pins`

```c
static int meson_get_group_pins(struct pinctrl_dev *pcdev,
                                unsigned selector,
                                const unsigned **pins,
                                unsigned *num_pins);
```

بتكشف الـ pins اللي بتكوّن group معين — بتكتب pointer على الـ array في `*pins` والعدد في `*num_pins`.

**Return:** دايماً `0` — ما فيش validation لأن الـ selector بيتم التحقق منه من الـ core قبل الاستدعاء.

---

#### `meson_pin_dbg_show`

```c
static void meson_pin_dbg_show(struct pinctrl_dev *pcdev,
                               struct seq_file *s,
                               unsigned offset);
```

بتكتب اسم الـ device في ملف الـ debugfs. Implementation بسيطة جداً — مجرد `seq_printf`.

---

### Category 3: Pinmux Ops (Exported)

هاتي الـ 3 functions بيتم تصديرها بـ `EXPORT_SYMBOL_GPL` لأن كل driver خاص بـ SoC (مثلاً `pinctrl-meson8.c`) بيحتاجها لتكوين الـ `pinmux_ops` vtable الخاصة بيه. هي بتوفر الجزء المشترك بين كل الـ SoCs.

---

#### `meson_pmx_get_funcs_count`

```c
int meson_pmx_get_funcs_count(struct pinctrl_dev *pcdev);
```

بترجع عدد الـ mux functions المعرفة للـ SoC الحالي.

**Return:** `pc->data->num_funcs`.

---

#### `meson_pmx_get_func_name`

```c
const char *meson_pmx_get_func_name(struct pinctrl_dev *pcdev,
                                    unsigned selector);
```

بترجع اسم الـ function بالـ index `selector` من الـ `funcs` array.

**Return:** string ثابت في الـ `.rodata`.

---

#### `meson_pmx_get_groups`

```c
int meson_pmx_get_groups(struct pinctrl_dev *pcdev,
                         unsigned selector,
                         const char * const **groups,
                         unsigned * const num_groups);
```

بتكشف الـ groups اللي بتشترك في function معينة. بتكتب pointer على array من strings في `*groups` والعدد في `*num_groups`.

**Return:** دايماً `0`.

---

### Category 4: Pinconf Internal Helpers

هاتي الـ functions هي اللي بتقرأ وبتكتب الـ hardware registers فعلياً. كلها تمر على `meson_get_bank` ثم `meson_calc_reg_and_bit` ثم `regmap_read/update_bits`.

---

#### `meson_pinconf_set_gpio_bit` / `meson_pinconf_get_gpio_bit`

```c
static int meson_pinconf_set_gpio_bit(struct meson_pinctrl *pc,
                                      unsigned int pin,
                                      unsigned int reg_type,
                                      bool arg);

static int meson_pinconf_get_gpio_bit(struct meson_pinctrl *pc,
                                      unsigned int pin,
                                      unsigned int reg_type);
```

الـ low-level primitives للكتابة والقراءة من `pc->reg_gpio`. الـ set بيستخدم `regmap_update_bits` لتعديل bit واحد فقط. الـ get بيقرأ الـ register ويعمل mask على الـ bit المحسوب.

**Parameters:**
- `pin` — رقم الـ pin
- `reg_type` — نوع الـ register (DIR, OUT, IN من `meson_reg_type`)
- `arg` — (للـ set فقط) القيمة المراد كتابتها

**Return (get):** `1` أو `0` لو نجح، قيمة سالبة لو في error.

**Key details:** الـ `reg_gpio` regmap بيستخدم لعمليات DIR و OUT و IN فقط. الـ `regmap_update_bits` thread-safe بسبب الـ regmap internal lock.

---

#### `meson_pinconf_set_output` / `meson_pinconf_get_output`

```c
static int meson_pinconf_set_output(struct meson_pinctrl *pc,
                                    unsigned int pin, bool out);

static int meson_pinconf_get_output(struct meson_pinctrl *pc,
                                    unsigned int pin);
```

**الـ `set_output`:** بيكتب في `MESON_REG_DIR`. لاحظ الـ inversion: في hardware المسون، الـ `DIR bit = 0` معناه output و `DIR bit = 1` معناه input، فبيتم عمل `!out` قبل الكتابة.

**الـ `get_output`:** بترجع `1` لو الـ pin configured كـ output، `0` لو input. الـ inversion بيتعمل تاني هنا (`return !ret`).

---

#### `meson_pinconf_set_drive` / `meson_pinconf_get_drive`

```c
static int meson_pinconf_set_drive(struct meson_pinctrl *pc,
                                   unsigned int pin, bool high);

static int meson_pinconf_get_drive(struct meson_pinctrl *pc,
                                   unsigned int pin);
```

بيكتب/يقرأ قيمة الـ output في `MESON_REG_OUT`. لا علاقة لهم باتجاه الـ pin — المفروض الـ pin يكون output قبل ما تستدعيهم.

---

#### `meson_pinconf_set_output_drive`

```c
static int meson_pinconf_set_output_drive(struct meson_pinctrl *pc,
                                          unsigned int pin, bool high);
```

بتعمل atomic sequence من عمليتين: أول بتضبط الـ pin كـ output عن طريق `meson_pinconf_set_output`، وبعدين بتكتب القيمة عن طريق `meson_pinconf_set_drive`. لو الخطوة الأولى فشلت، بيرجع الـ error فوراً.

**Caller context:** بتُستدعى من `meson_gpio_direction_output` ومن `meson_pinconf_set` لما الـ param هو `PIN_CONFIG_LEVEL`.

---

#### `meson_pinconf_disable_bias`

```c
static int meson_pinconf_disable_bias(struct meson_pinctrl *pc,
                                      unsigned int pin);
```

بتعطل الـ pull resistor بالكامل عن طريق clear bit الـ `MESON_REG_PULLEN` في الـ `pc->reg_pullen`. ما بتعملش أي تغيير على `MESON_REG_PULL` — الاتجاه (up/down) بيفضل محفوظ في الـ hardware لكن غير فعّال.

**Key details:** بعد الـ disable، الـ pin يكون في حالة high-impedance من ناحية الـ bias.

---

#### `meson_pinconf_enable_bias`

```c
static int meson_pinconf_enable_bias(struct meson_pinctrl *pc,
                                     unsigned int pin, bool pull_up);
```

بتفعّل الـ pull resistor. العملية في خطوتين مرتبتين:
1. بتكتب الاتجاه (up=1 أو down=0) في `MESON_REG_PULL` عبر `pc->reg_pull`
2. بتعمل set لـ enable bit في `MESON_REG_PULLEN` عبر `pc->reg_pullen`

**Parameters:**
- `pull_up` — `true` لـ pull-up، `false` لـ pull-down

**Key details:** الترتيب مهم: الاتجاه يتحدد أول قبل التفعيل لتجنب glitch. في بعض الـ SoCs (مثلاً Meson8 AO bus)، `reg_pull == reg_pullen`، لكن الـ logic نفسها شغالة.

---

#### `meson_pinconf_set_drive_strength`

```c
static int meson_pinconf_set_drive_strength(struct meson_pinctrl *pc,
                                            unsigned int pin,
                                            u16 drive_strength_ua);
```

بتضبط الـ drive strength بالـ microamps. الـ hardware بيدعم 4 levels: 500µA، 2500µA، 3000µA، 4000µA. كل pin بياخد 2-bit field في `pc->reg_ds`.

```c
/* Pseudocode */
if (!pc->reg_ds) return -ENOTSUPP;  // not all SoCs support DS
meson_get_bank(pc, pin, &bank);
meson_calc_reg_and_bit(bank, pin, MESON_REG_DS, &reg, &bit);
ds_val = quantize(drive_strength_ua); // snap to nearest supported level
regmap_update_bits(pc->reg_ds, reg, 0x3 << bit, ds_val << bit);
```

**Parameters:**
- `drive_strength_ua` — القيمة بالـ µA، بيتم tقريب للقيمة الأعلى المدعومة

**Return:** `-ENOTSUPP` لو الـ SoC ما بيدعمش الـ DS registers.

**Key details:** لو القيمة أكبر من 4000µA، بيطبع `dev_warn_once` وبيستخدم 4000µA. الـ 2-bit mask هو `0x3 << bit`.

---

#### `meson_pinconf_get_pull`

```c
static int meson_pinconf_get_pull(struct meson_pinctrl *pc,
                                  unsigned int pin);
```

بتقرأ حالة الـ pull وبترجع `PIN_CONFIG_BIAS_DISABLE` أو `PIN_CONFIG_BIAS_PULL_UP` أو `PIN_CONFIG_BIAS_PULL_DOWN`.

```c
/* Pseudocode */
read MESON_REG_PULLEN bit:
  if 0 → return PIN_CONFIG_BIAS_DISABLE
  if 1 → read MESON_REG_PULL bit:
            if 1 → return PIN_CONFIG_BIAS_PULL_UP
            if 0 → return PIN_CONFIG_BIAS_PULL_DOWN
```

**Return:** قيمة `pin_config_param` enum كـ int.

---

#### `meson_pinconf_get_drive_strength`

```c
static int meson_pinconf_get_drive_strength(struct meson_pinctrl *pc,
                                            unsigned int pin,
                                            u16 *drive_strength_ua);
```

بتقرأ الـ 2-bit DS field وبتترجمها لقيمة µA في `*drive_strength_ua`.

**Return:** `0` لو نجح، `-ENOTSUPP` لو ما فيش DS register، `-EINVAL` لو القيمة مش valid.

---

### Category 5: Pinconf Ops Entry Points

---

#### `meson_pinconf_set`

```c
static int meson_pinconf_set(struct pinctrl_dev *pcdev,
                             unsigned int pin,
                             unsigned long *configs,
                             unsigned num_configs);
```

الـ entry point الرسمي للـ pinconf_set من الـ pinctrl subsystem. بتعمل loop على array الـ configs وبتطبق كل واحدة على الـ pin المحدد.

```c
/* Pseudocode */
for each config in configs[]:
    param = pinconf_to_config_param(config)
    arg   = pinconf_to_config_argument(config)  // for some params only
    switch(param):
        BIAS_DISABLE     → meson_pinconf_disable_bias()
        BIAS_PULL_UP     → meson_pinconf_enable_bias(..., true)
        BIAS_PULL_DOWN   → meson_pinconf_enable_bias(..., false)
        DRIVE_STRENGTH_UA→ meson_pinconf_set_drive_strength(..., arg)
        OUTPUT_ENABLE    → meson_pinconf_set_output(..., arg)
        LEVEL            → meson_pinconf_set_output_drive(..., arg)
        default          → return -ENOTSUPP
```

**Parameters:**
- `pin` — رقم الـ pin
- `configs` — array من encoded config values (param + arg في 64-bit)
- `num_configs` — عدد العناصر في الـ array

**Return:** `0` لو كل الـ configs اتطبقت، أو أول error يواجهه.

**Key details:** عند أول خطأ، الـ function بتوقف ولا تكمل بقية الـ configs. الـ `configs` format هو الـ generic pinconf encoding (`pinconf_to_config_param` + `pinconf_to_config_argument`).

---

#### `meson_pinconf_get`

```c
static int meson_pinconf_get(struct pinctrl_dev *pcdev,
                             unsigned int pin,
                             unsigned long *config);
```

الـ entry point الرسمي للـ pinconf_get. بتقرأ `*config` كـ param، وبتحاول تجيب القيمة من الـ hardware، وبتكتب النتيجة المـ encoded في `*config`.

**Key details:**
- للـ bias params، بتقارن الـ param الحالي من الـ hardware بالـ param المطلوب — لو ما بيتطابقوش بترجع `-EINVAL`. القيمة المرجعة `arg` هي `60000` (convention ثابت في الـ kernel لـ resistance value).
- لـ `PIN_CONFIG_OUTPUT_ENABLE`: بتتحقق إن الـ pin هو output فعلاً، لو لا بترجع `-EINVAL`.
- لـ `PIN_CONFIG_LEVEL`: لازم الـ pin يكون output وبعدين بتقرأ القيمة.

**Return:** `0` ومعه `*config` مُحدّث، أو error code.

---

#### `meson_pinconf_group_set`

```c
static int meson_pinconf_group_set(struct pinctrl_dev *pcdev,
                                   unsigned int num_group,
                                   unsigned long *configs,
                                   unsigned num_configs);
```

بتطبق نفس الـ configs على جميع pins الـ group بعمل loop وبتستدعي `meson_pinconf_set` لكل pin. بتتجاهل errors من pins منفردة — الـ return دايماً `0`.

**Parameters:**
- `num_group` — index الـ group
- `configs` / `num_configs` — نفس format الـ `meson_pinconf_set`

---

#### `meson_pinconf_group_get`

```c
static int meson_pinconf_group_get(struct pinctrl_dev *pcdev,
                                   unsigned int group,
                                   unsigned long *config);
```

غير مدعوم — بترجع `-ENOTSUPP` مباشرة. الـ pinctrl subsystem متوقع ده من بعض الـ drivers.

---

### Category 6: GPIO Chip Ops

هاتي الـ functions بتكوّن الـ vtable الـ `gpio_chip` اللي بيستخدمها gpiolib. كلها thin wrappers فوق الـ pinconf helpers.

---

#### `meson_gpio_get_direction`

```c
static int meson_gpio_get_direction(struct gpio_chip *chip, unsigned gpio);
```

بترجع `GPIO_LINE_DIRECTION_OUT` (0) أو `GPIO_LINE_DIRECTION_IN` (1) حسب حالة الـ DIR register.

**Caller:** gpiolib وقت الـ sysfs أو kernel API queries.

---

#### `meson_gpio_direction_input`

```c
static int meson_gpio_direction_input(struct gpio_chip *chip, unsigned gpio);
```

Wrapper مباشر لـ `meson_pinconf_set_output(pc, gpio, false)`.

---

#### `meson_gpio_direction_output`

```c
static int meson_gpio_direction_output(struct gpio_chip *chip,
                                       unsigned gpio, int value);
```

Wrapper لـ `meson_pinconf_set_output_drive` — بيضبط الاتجاه والقيمة في نفس الوقت.

---

#### `meson_gpio_set`

```c
static int meson_gpio_set(struct gpio_chip *chip,
                          unsigned int gpio, int value);
```

Wrapper لـ `meson_pinconf_set_drive` — بيكتب قيمة output فقط بدون تغيير الاتجاه.

**Return:** `int` — `0` لو نجح، error code لو فشل.

---

#### `meson_gpio_get`

```c
static int meson_gpio_get(struct gpio_chip *chip, unsigned gpio);
```

بتقرأ القيمة من `MESON_REG_IN` — الـ input latch للـ pin. بتستخدم `meson_calc_reg_and_bit` ثم `regmap_read` ثم bit masking.

**Return:** `1` أو `0` — قيمة الـ pin المقروءة من الـ hardware.

**Key details:** بتقرأ من `pc->reg_gpio` بالـ `MESON_REG_IN` type — مش الـ output register. ده مهم لأن الـ input والـ output registers مختلفة حتى لو الـ pin configured كـ output.

---

#### `meson_gpiolib_register`

```c
static int meson_gpiolib_register(struct meson_pinctrl *pc);
```

بتهيّئ وتسجل الـ `gpio_chip` struct في الـ gpiolib. بتملأ كل fields الـ chip:

| Field | القيمة |
|---|---|
| `label` | `pc->data->name` |
| `parent` | `pc->dev` |
| `fwnode` | `pc->fwnode` (من DT) |
| `request` / `free` | `gpiochip_generic_request/free` |
| `set_config` | `gpiochip_generic_config` |
| `base` | `-1` (dynamic allocation) |
| `ngpio` | `pc->data->num_pins` |
| `can_sleep` | `true` (regmap I/O قد ينام) |

**Return:** `0` لو نجح، error من `gpiochip_add_data` لو فشل.

**Key details:** `base = -1` بيخلي الـ kernel يختار الـ GPIO base number تلقائياً، ده الـ best practice للـ DT-based drivers. `can_sleep = true` لأن الـ regmap access على بعض الـ buses (مثلاً SPMI, I2C) قد يكون blocking — حتى لو الـ MMIO هنا سريع، السلامة مهمة.

---

### Category 7: Init / Probe

---

#### `meson_map_resource`

```c
static struct regmap *meson_map_resource(struct meson_pinctrl *pc,
                                         struct device_node *node,
                                         char *name);
```

بتحوّل resource مسمّى من الـ DT إلى `regmap`. الخطوات:

```c
/* Pseudocode */
i    = of_property_match_string(node, "reg-names", name) // find index
res  = of_address_to_resource(node, i)                   // get phys addr
base = devm_ioremap_resource(pc->dev, &res)              // ioremap
config.max_register = resource_size(&res) - 4            // set bounds
config.name = devm_kasprintf(...)                        // unique name
return devm_regmap_init_mmio(pc->dev, base, &config)     // create regmap
```

**Parameters:**
- `node` — الـ DT node اللي فيه `reg-names`
- `name` — الاسم المراد البحث عنه (مثلاً `"mux"`, `"gpio"`, `"pull"`)

**Return:** `struct regmap *` لو نجح، `NULL` لو ما فيش الـ resource، أو `ERR_PTR` لو في error.

**Key details:** بيستخدم `meson_regmap_config` global struct — ده يعني ما ينفعش يتستدعى بشكل parallel. الـ `max_register` بيتحدد dynamically من حجم الـ resource لمنع out-of-bounds writes. كل الـ allocations هي `devm_` فبتتحرر تلقائياً عند الـ unbind.

---

#### `meson_pinctrl_parse_dt`

```c
static int meson_pinctrl_parse_dt(struct meson_pinctrl *pc);
```

بتحلل الـ DT وتملأ الـ `pc->reg_*` pointers. الترتيب والأهمية:

```c
/* Pseudocode */
chips = gpiochip_node_count(pc->dev)  // must be exactly 1
pc->fwnode = gpiochip_node_get_first(pc->dev)
gpio_np    = to_of_node(pc->fwnode)

// Mandatory:
pc->reg_mux  = meson_map_resource(..., "mux")   // error if missing
pc->reg_gpio = meson_map_resource(..., "gpio")  // error if missing

// Optional (NULL if not present):
pc->reg_pull   = meson_map_resource(..., "pull")
pc->reg_pullen = meson_map_resource(..., "pull-enable")
pc->reg_ds     = meson_map_resource(..., "ds")

// SoC-specific override:
if (pc->data->parse_dt)
    return pc->data->parse_dt(pc);
```

**Return:** `0` لو نجح، `-EINVAL` لو مفيش gpio node أو أكتر من واحد، error من الـ regmap creation لو الـ mandatory registers مش موجودين.

**Key details:** وجود `reg_pull` و `reg_pullen` بيكونوا `NULL` في بعض الـ SoCs اللي بتوحّد الـ registers — الـ SoC-specific `parse_dt` hook هو اللي بيصلح ده. الـ `reg_ds` اختياري تماماً.

---

#### `meson8_aobus_parse_dt_extra`

```c
int meson8_aobus_parse_dt_extra(struct meson_pinctrl *pc);
```

خاصة بـ Meson8/Meson8b AO bus. في الـ AO domain من هاتي الـ SoCs، الـ `pull-enable` register هو نفسه `pull` register، فبتعمل:

```c
pc->reg_pullen = pc->reg_pull;
```

**Return:** `-EINVAL` لو `pc->reg_pull == NULL`، وإلا `0`.

**Exported:** `EXPORT_SYMBOL_GPL` — بيتستدعى من `pinctrl-meson8-ao.c` و `pinctrl-meson8b-ao.c`.

---

#### `meson_a1_parse_dt_extra`

```c
int meson_a1_parse_dt_extra(struct meson_pinctrl *pc);
```

خاصة بـ A1 SoC (ومن بعده G12A AO). في هاتي الـ SoCs، registers الـ GPIO والـ pull والـ DS هي كلها نفس الـ register range:

```c
pc->reg_pull   = pc->reg_gpio;
pc->reg_pullen = pc->reg_gpio;
pc->reg_ds     = pc->reg_gpio;
```

**Return:** دايماً `0`.

**Exported:** `EXPORT_SYMBOL_GPL`.

---

#### `meson_pinctrl_probe`

```c
int meson_pinctrl_probe(struct platform_device *pdev);
```

الـ entry point الرئيسي للـ driver — بيُستدعى من الـ kernel عند تطابق الـ device مع الـ `platform_driver`. بيمر على المراحل التالية:

```c
/* Pseudocode */
// 1. Allocate
pc = devm_kzalloc(dev, sizeof(*pc))
pc->dev  = dev
pc->data = of_device_get_match_data(dev)  // SoC-specific data from DT match

// 2. Parse DT + map registers
meson_pinctrl_parse_dt(pc)

// 3. Fill pinctrl_desc
pc->desc = {
    .name    = "pinctrl-meson",
    .pctlops = &meson_pctrl_ops,
    .pmxops  = pc->data->pmx_ops,    // SoC-specific mux ops
    .confops = &meson_pinconf_ops,
    .pins    = pc->data->pins,
    .npins   = pc->data->num_pins,
}

// 4. Register with pinctrl core
pc->pcdev = devm_pinctrl_register(pc->dev, &pc->desc, pc)

// 5. Register GPIO chip
meson_gpiolib_register(pc)
```

**Parameters:** `pdev` — الـ platform device من الـ kernel.

**Return:** `0` لو نجح، error code لو أي خطوة فشلت.

**Key details:**
- `of_device_get_match_data` بترجع الـ `meson_pinctrl_data *` المخزن في `of_device_id.data` — ده هو الفرق الوحيد بين driver لـ SoC واحد وآخر.
- `devm_pinctrl_register` بتسجل الـ pinctrl device وبتربطه بالـ `devm` lifecycle فبيتحرر تلقائياً.
- الـ `pc->data->pmx_ops` بيجي من الـ SoC-specific driver (مثلاً `pinctrl-meson8.c`) لأن الـ mux bit manipulation مختلف لكل SoC.
- الـ `meson_pinconf_ops` و `meson_pctrl_ops` مشتركة بين كل الـ SoCs ومعرّفة static في هذا الملف.

**Exported:** `EXPORT_SYMBOL_GPL` — كل SoC-specific driver بيستدعيه من الـ `platform_driver.probe`.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المدخلات المهمة

**الـ pinctrl subsystem** بيعمل expose لمعلومات تفصيلية جوا `/sys/kernel/debug/pinctrl/`.

```bash
# اعرف كل الـ pinctrl controllers المسجلين
ls /sys/kernel/debug/pinctrl/

# الـ output بيطلع حاجة زي:
# pinctrl-meson  (أو اسم الـ SoC مثلاً: pinctrl-meson-g12a)
# pinctrl-handles
# pinctrl-devices
# pinctrl-maps

# اقرأ كل الـ pins وحالتها
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pins

# اقرأ الـ pingroups المتاحة
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pingroups

# اقرأ الـ pinmux functions
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pinmux-functions

# اقرأ الـ pinmux pins — بيقولك كل pin اتعمله map لأي function
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pinmux-pins

# اقرأ الـ pinconf لكل pin (pull, drive strength, direction)
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pinconf-pins

# اقرأ الـ pinconf على مستوى الـ groups
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pinconf-groups
```

**مثال عملي على قراءة pinmux-pins:**
```
pin 0 (GPIOX_0): meson-g12a-periphs-pinctrl group: emmc_d0 function: emmc
pin 7 (GPIOX_7): meson-g12a-periphs-pinctrl group: emmc_clk function: emmc
pin 60 (GPIOA_0): UNCLAIMED
```
السطر ده بيقولك إن `GPIOX_0` متعمله mux لـ `emmc` وإن `GPIOA_0` مش مستخدم.

---

#### 2. sysfs — المدخلات المهمة

**الـ GPIO subsystem** بيعمل expose عبر `/sys/class/gpio/` والأحدث `/sys/bus/platform/devices/`.

```bash
# اعرف رقم الـ gpiochip الخاص بـ Meson
ls /sys/class/gpio/
# gpiochip0  gpiochip85  ...

# اقرأ معلومات الـ chip
cat /sys/class/gpio/gpiochip0/label    # اسم الـ bank مثلاً: meson-gpio
cat /sys/class/gpio/gpiochip0/base     # أول رقم GPIO
cat /sys/class/gpio/gpiochip0/ngpio    # عدد الـ pins

# export pin معين للتجربة (مثلاً pin 10)
echo 10 > /sys/class/gpio/export

# اقرأ الاتجاه
cat /sys/class/gpio/gpio10/direction   # in أو out

# اقرأ القيمة
cat /sys/class/gpio/gpio10/value       # 0 أو 1

# ارجعه للـ kernel
echo 10 > /sys/class/gpio/unexport

# اقرأ معلومات الـ platform device مباشرة
ls /sys/bus/platform/devices/ | grep pinctrl
cat /sys/bus/platform/devices/ff634400.bus:pinctrl/uevent
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# شوف الـ events المتاحة للـ gpio
ls /sys/kernel/debug/tracing/events/gpio/

# فعّل تتبع كل أحداث الـ gpio
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو فعّل event معين
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل العملية اللي عايز تتتبعها هنا...

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace

# مثال على output:
# <...>-123 [000] .... gpio_direction: gpio=10 direction=out
# <...>-123 [000] .... gpio_value: gpio=10 get=0

# تتبع الـ function calls جوا pinctrl driver بالاسم
echo 'meson_pinconf_set' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# تتبع stack عند كل استدعاء
echo 1 > /sys/kernel/debug/tracing/options/func_stack_trace
```

---

#### 4. printk و dynamic debug

**تفعيل الـ dynamic debug** للـ subsystem ده بدون إعادة compile:

```bash
# فعّل كل رسائل dev_dbg في ملف pinctrl-meson.c
echo 'file pinctrl-meson.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل رسائل الـ meson pinctrl directory
echo 'file drivers/pinctrl/meson/* +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع الـ stack trace
echo 'file pinctrl-meson.c +ps' > /sys/kernel/debug/dynamic_debug/control

# تابع الـ kernel log
dmesg -w | grep -i pinctrl
dmesg -w | grep -i meson

# أو عبر journalctl
journalctl -k -f | grep -i pinctrl
```

**رسائل dev_dbg الموجودة في الـ driver:**
- `"pinconf for pin %u is %lu\n"` — تظهر عند `meson_pinconf_get` لكل pin يتقرأ config بتاعته
- `"set pinconf for group %s\n"` — تظهر عند `meson_pinconf_group_set`
- `"ds registers not found - skipping\n"` — تظهر لو SoC مش فيه drive strength

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_DEBUG_PINCTRL` | يفعّل logging إضافي في الـ pinctrl core |
| `CONFIG_PINCTRL` | أساسي لتشغيل الـ subsystem |
| `CONFIG_GPIOLIB` | أساسي لتسجيل الـ GPIO chip |
| `CONFIG_GPIO_SYSFS` | يفتح `/sys/class/gpio/` |
| `CONFIG_DEBUG_GPIO` | يضيف checks إضافية في الـ GPIO calls |
| `CONFIG_REGMAP_DEBUGFS` | يعمل expose للـ regmap registers في debugfs |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل dev_dbg messages وقت الـ runtime |
| `CONFIG_FTRACE` | أساسي لاستخدام function tracing |
| `CONFIG_TRACING` | يفعّل الـ tracepoints |
| `CONFIG_OF` | لازم لقراءة الـ Device Tree |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(DEBUG_PINCTRL|DEBUG_GPIO|REGMAP_DEBUGFS|GPIO_SYSFS)'
```

---

#### 6. devlink وأدوات خاصة بالـ Subsystem

**الأداة الرئيسية: `pinctrl-tool` أو `gpiod-tools`:**

```bash
# تثبيت أدوات الـ GPIO (libgpiod)
apt-get install gpiod   # أو عبر buildroot/yocto

# اعرض كل الـ gpiochips
gpiodetect

# Output:
# gpiochip0 [meson-gpio] (85 lines)

# اعرض حالة كل الـ lines في chip
gpioinfo gpiochip0

# اقرأ قيمة pin معين (line 10)
gpioget gpiochip0 10

# اكتب قيمة
gpioset gpiochip0 10=1

# راقب التغييرات على pin
gpiomon gpiochip0 10

# regmap debugfs — اقرأ قيم الـ registers مباشرة
ls /sys/kernel/debug/regmap/
# ff634400.bus:pinctrl-mux
# ff634400.bus:pinctrl-gpio
# ...

cat /sys/kernel/debug/regmap/ff634400.bus:pinctrl-gpio/registers
# Output:
# 00: 00000000
# 04: 00ff0000
# ...
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `mux registers not found` | مفيش reg-names = "mux" في الـ DT | تحقق من `reg-names` في الـ gpio node |
| `gpio registers not found` | مفيش reg-names = "gpio" في الـ DT | تحقق من `reg-names` وعدد الـ `reg` entries |
| `no gpio node found` | الـ pinctrl node مفيش فيه child gpio node | أضف الـ gpio child node في الـ DTS |
| `multiple gpio nodes` | أكتر من gpio child node | الـ driver بيدعم node واحد بس |
| `can't add gpio chip %s` | `gpiochip_add_data` فشل | تعارض في الـ gpio base أو overlapping ranges |
| `can't register pinctrl device` | `devm_pinctrl_register` فشل | تحقق من الـ pinctrl_desc: pins, npins, pmxops |
| `drive-strength not supported` | SoC مش فيه `reg_ds` | الـ SoC ده مش بيدعم drive strength أو ناقص من الـ DT |
| `pin %u: invalid drive-strength: %d` | قيمة drive strength أكبر من 4000 uA | استخدم قيمة ≤ 4000 uA |
| `pin X is already claimed` | pin اتعمله request من driver تاني | تحقق من التعارضات في الـ pinctrl map |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في meson_get_bank() — لو pin خارج كل البنوك */
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
    /* نقطة مهمة: pin مش موجود في أي bank */
    WARN(1, "pin %u not found in any bank (num_banks=%d)\n",
         pin, pc->data->num_banks);
    return -EINVAL;
}

/* في meson_calc_reg_and_bit() — تحقق من الـ bit overflow */
static void meson_calc_reg_and_bit(...)
{
    /* WARN_ON لو الـ bit خرج عن الـ 32-bit boundary بشكل غير متوقع */
    WARN_ON(*bit >= 32);
}

/* في meson_pinctrl_probe() — لو الـ data pointer فاضي */
    WARN_ON(!pc->data);
    WARN_ON(!pc->data->pmx_ops);
```

---

### Hardware Level

#### 1. التحقق من أن حالة الـ Hardware تطابق الـ Kernel State

```bash
# الخطوة 1: اقرأ الـ register value من الـ kernel عبر regmap debugfs
cat /sys/kernel/debug/regmap/ff800000.bus:pinctrl-gpio/registers

# الخطوة 2: قارن بالـ physical register عبر devmem2
# (عنوان الـ GPIO register للـ AO domain في Meson G12A مثلاً: 0xff800000)
devmem2 0xff800028 w   # اقرأ 32-bit word

# الخطوة 3: قارن القيمتين — لازم يكونوا متطابقين
# لو مختلفين: في cache coherency issue أو الـ regmap config غلط

# مثال عملي: تحقق من direction الـ GPIOX_0
# bank X: REG_DIR offset في الـ bank descriptor
# استخدم meson_calc_reg_and_bit logic يدوياً:
# bit = (desc->bit + pin - bank->first) * stride
# reg = (desc->reg + (bit / 32)) * 4
devmem2 0xffd10004 w   # REG_DIR للـ GPIOX bank
```

---

#### 2. Register Dump Techniques

```bash
# devmem2 — اقرأ register بشكل مباشر
# تثبيت: apt-get install devmem2
devmem2 <physical_address> w    # 32-bit read
devmem2 <physical_address> w <value>  # 32-bit write

# مثال: Meson G12A periphs GPIO base = 0xffd10000
# MUX registers
devmem2 0xffd10000 w   # PERIPHS_PIN_MUX_0
devmem2 0xffd10004 w   # PERIPHS_PIN_MUX_1
# GPIO registers
devmem2 0xffd10120 w   # GPIO_0_EN_N (direction)
devmem2 0xffd10124 w   # GPIO_0_OUT
devmem2 0xffd10128 w   # GPIO_0_IN
# Pull enable
devmem2 0xffd10148 w   # PULL_UP_EN_0
# Pull direction
devmem2 0xffd10158 w   # PULL_UP_0
# Drive strength
devmem2 0xffd100d0 w   # DS_0

# /dev/mem — بديل أبطأ
busybox devmem 0xffd10000

# io tool (من package io أو busybox)
io -4 -r 0xffd10000
```

**Meson G12A Register Map Summary:**

```
AO domain  (Always-On):   0xff800000
Periphs:                  0xffd10000

Offset layout (Periphs):
+0x000 → MUX registers    (reg_mux)
+0x100 → DS registers     (reg_ds)
+0x120 → GPIO direction   (reg_gpio)
+0x148 → Pull-enable      (reg_pullen)
+0x158 → Pull direction   (reg_pull)
```

---

#### 3. Logic Analyzer و Oscilloscope Tips

```
المشكلة                   الأداة المناسبة         ما تبحث عنه
─────────────────────────────────────────────────────────────
Pin مش بيتغير           Logic Analyzer          راقب الـ signal على الـ pin نفسه
Pull-up/down غلط         Oscilloscope            ارسم الـ voltage level في حالة Hi-Z
Mux غلط (SPI vs UART)   Logic Analyzer (2ch)    قارن بروتوكول الـ signal
Drive strength ضعيف      Oscilloscope            ارسم rise/fall time
Glitch عند الـ boot      Logic Analyzer + Trig   trigger على أول edge بعد الـ power-on
```

```bash
# قبل توصيل الـ analyzer:
# 1. اتحقق إن الـ pin اتعمله mux لـ GPIO مش لـ function
gpioinfo gpiochip0 | grep "line 10"

# 2. اعمل export وset direction
gpioset gpiochip0 10=0
# 3. وصّل الـ probe على الـ pin
# 4. غير القيمة وشوف الـ signal
gpioset gpiochip0 10=1
```

---

#### 4. Hardware Issues الشائعة ← Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log | التفسير |
|---|---|---|
| خط VCC مقطوع للـ AO domain | `mux registers not found` عند الـ boot | الـ ioremap فشل لأن الـ memory مش accessible |
| تلف في الـ pull resistor الداخلي | `pin X: unexpected value` في الـ driver | الـ input value مش ثابت |
| تعارض في الـ signal مع component تاني | `pin is already claimed` | driver تاني عمل request لنفس الـ pin |
| regulator مش شغال قبل الـ pinctrl | `gpio registers not found` أو `-EPROBE_DEFER` | تسلسل الـ probe غلط |
| PCB trace مقطوعة | الـ GPIO value ثابت على 0 أو 1 بغض النظر عن الـ output | مش في الـ log، يتحقق بالـ oscilloscope |

---

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT المحمّل فعلاً في الـ runtime
ls /proc/device-tree/
ls /proc/device-tree/soc/bus@ffd00000/

# اقرأ الـ properties مباشرة
cat /proc/device-tree/soc/bus@ffd00000/pinctrl@400/compatible
# amlogic,meson-g12a-periphs-pinctrl

# تحقق من reg-names في الـ gpio child node
cat /proc/device-tree/soc/bus@ffd00000/pinctrl@400/gpio@10/reg-names
# يجب أن يحتوي على: mux gpio pull pull-enable ds

# dtc — فكك الـ DTB المحمّل
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 30 "pinctrl@"

# أو استخدم fdtdump على الـ DTB file مباشرة
fdtdump /boot/meson-g12a-sei510.dtb | grep -A 50 "periphs-pinctrl"

# تحقق من عدد الـ reg entries يطابق عدد reg-names
# لكل اسم في reg-names لازم يكون فيه entry واحد في reg
```

**DT Node المطلوب لـ Meson Pinctrl (مثال صحيح):**
```dts
&periphs_pinctrl {
    /* parent node */
    compatible = "amlogic,meson-g12a-periphs-pinctrl";
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    gpio_periphs: bank@40 {
        /* gpio child node */
        compatible = "amlogic,meson-g12a-periphs-gpio";
        reg = <0x0 0x40  0x0 0x1c>,   /* mux */
              <0x0 0xe8  0x0 0x18>,   /* ds */
              <0x0 0x120 0x0 0x18>,   /* gpio */
              <0x0 0x148 0x0 0x18>,   /* pull-enable */
              <0x0 0x158 0x0 0x18>;   /* pull */
        reg-names = "mux", "ds", "gpio", "pull-enable", "pull";
        gpio-controller;
        #gpio-cells = <2>;
        gpio-ranges = <&periphs_pinctrl 0 0 85>;
    };
};
```

---

### Practical Commands

#### 1. Shell Commands جاهزة للنسخ

```bash
# ====== فحص شامل للـ pinctrl state ======
PINCTRL_DEV=$(ls /sys/kernel/debug/pinctrl/ | grep -v 'handles\|devices\|maps' | head -1)
echo "=== Pinctrl Device: $PINCTRL_DEV ==="
echo "--- Pinmux Pins ---"
cat /sys/kernel/debug/pinctrl/${PINCTRL_DEV}/pinmux-pins
echo "--- Pinconf Pins ---"
cat /sys/kernel/debug/pinctrl/${PINCTRL_DEV}/pinconf-pins

# ====== فحص GPIO chip ======
gpiodetect
CHIP=$(gpiodetect | grep meson | awk '{print $1}')
gpioinfo $CHIP

# ====== فحص pin معين (مثلاً pin 15) ======
PIN=15
echo "GPIO $PIN direction:"
cat /sys/kernel/debug/pinctrl/${PINCTRL_DEV}/pinmux-pins | grep "pin $PIN "

# ====== تفعيل dynamic debug للـ meson pinctrl ======
echo 'file pinctrl-meson.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/meson/ +p' > /sys/kernel/debug/dynamic_debug/control
dmesg -c; echo "Dynamic debug enabled. Perform operation then check dmesg."

# ====== ftrace لتتبع pinconf_set ======
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo '' > /sys/kernel/debug/tracing/trace
echo 'meson_pinconf_set meson_pinconf_get meson_get_bank' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... اعمل العملية اللي عايز تتتبعها ...
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace

# ====== قراءة regmap registers ======
for RMAP in /sys/kernel/debug/regmap/*pinctrl*; do
    echo "=== $(basename $RMAP) ==="
    cat $RMAP/registers 2>/dev/null | head -20
done

# ====== تحقق من DT properties ======
for node in /proc/device-tree/soc/*/pinctrl*/*/; do
    echo "Node: $node"
    [ -f "$node/reg-names" ] && echo "  reg-names: $(strings $node/reg-names | tr '\n' ' ')"
    [ -f "$node/compatible" ] && echo "  compatible: $(cat $node/compatible)"
done
```

---

#### 2. مثال Output وكيفية تفسيره

**مثال: output من `pinmux-pins`:**
```
pin 0 (GPIOX_0): meson-g12a-periphs-pinctrl group: emmc_d0 function: emmc
pin 1 (GPIOX_1): meson-g12a-periphs-pinctrl group: emmc_d1 function: emmc
pin 8 (GPIOX_8): UNCLAIMED
pin 9 (GPIOX_9): (MUX UNCLAIMED) (GPIO UNCLAIMED)
```

- `group: emmc_d0 function: emmc` → الـ pin اتعمله mux لـ eMMC، مش GPIO
- `UNCLAIMED` → الـ pin مش محجوز من أي driver، ممكن تستخدمه
- `(MUX UNCLAIMED) (GPIO UNCLAIMED)` → مش اتعمله request نهائياً

**مثال: output من `pinconf-pins`:**
```
pin 0 (GPIOX_0): bias-pull-up
pin 1 (GPIOX_1): bias-disable
pin 60 (GPIOA_0): output-enable drive-strength-microamp=2500
```

- `bias-pull-up` → الـ PULLEN bit = 1، الـ PULL bit = 1
- `bias-disable` → الـ PULLEN bit = 0 (pull مش مفعّل)
- `output-enable drive-strength-microamp=2500` → اتجاه output، والـ DS bits = 01 (MESON_PINCONF_DRV_2500UA)

**مثال: output من regmap registers:**
```
/sys/kernel/debug/regmap/ffd10000.pinctrl-gpio/registers:
00000000: 00000000   ← GPIO_0_EN_N: كل الـ pins input (bit=1 = input)
00000004: 00000000
00000120: 0000ffff   ← الـ bits 0-15 = output direction لأول 16 pin
```

التفسير: الـ register `0x120` (GPIO direction) فيه `0x0000ffff` يعني الـ pins 0-15 direction = output (`0`) والباقي input (`1`) — بالعكس لأن `MESON_REG_DIR` بيتم invert في `meson_pinconf_set_output`: القيمة `1` في الـ register = input.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Android TV Box على Amlogic S905X4 — UART ما بيشتغلش

#### العنوان
**الـ UART console بيظهر garbage بعد boot على S905X4 TV box**

#### السياق
بتشتغل على Android TV box بيستخدم Amlogic S905X4 (مبني على Meson G12A architecture). العميل بيشتكي إن الـ serial console بتطلع أحرف عشوائية بعد ما يبدأ الـ kernel يشتغل، رغم إن الـ u-boot console تمام.

#### المشكلة
الـ UART0 (على Bank AO) بيطلع garbage بدل النص الصحيح. السبب الأغلب: الـ pin mux مش اتضبط صح، أو الـ pull configuration خاطئة على pins الـ TX/RX.

#### التحليل
الـ flow بيبدأ من `meson_pinctrl_probe()`:

```c
// probe بيقرأ match data من DT
pc->data = (struct meson_pinctrl_data *) of_device_get_match_data(dev);

// بعدين بيعمل parse للـ DT
ret = meson_pinctrl_parse_dt(pc);
```

جوا `meson_pinctrl_parse_dt()` بيعمل map للـ registers:

```c
pc->reg_mux = meson_map_resource(pc, gpio_np, "mux");   // mux registers
pc->reg_gpio = meson_map_resource(pc, gpio_np, "gpio"); // direction/in/out
pc->reg_pull = meson_map_resource(pc, gpio_np, "pull");
pc->reg_pullen = meson_map_resource(pc, gpio_np, "pull-enable");
```

لو الـ DT فيه اسم `reg-names` غلط (مثلاً كاتب `"pull"` بدل `"pull-enable"` أو العكس)، الـ `meson_map_resource()` بترجع `NULL` أو `ERR_PTR`، وبعدين:

```c
if (IS_ERR(pc->reg_pull))
    pc->reg_pull = NULL;   // بيتسبت NULL بدل ما يفشل
```

لما الـ UART driver بعدين يطلب `PIN_CONFIG_BIAS_PULL_UP` على pin الـ RX، الكود بيوصل لـ:

```c
static int meson_pinconf_enable_bias(struct meson_pinctrl *pc,
                                     unsigned int pin, bool pull_up)
{
    // ...
    ret = regmap_update_bits(pc->reg_pull, reg, BIT(bit), val);
    // لو reg_pull == NULL → crash أو silent fail
```

الـ `meson_calc_reg_and_bit()` بتحسب الـ offset بناءً على `bank->regs[MESON_REG_PULL].reg` و `.bit`، لو القيم غلطة في macro الـ `BANK()` في الـ SoC-specific file هتكتب على register غلط.

#### الحل

**1. تحقق من DT:**

```bash
# على board متشغل
cat /proc/device-tree/periphs/pinctrl@4b0/gpio@14/reg-names
# المتوقع: mux gpio pull pull-enable ds
```

**2. تحقق من الـ pinmux state:**

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pinmux-pins | grep uart
# المتوقع: pin 145 (GPIOAO_0): uart_ao_a
```

**3. تحقق من الـ pull configuration:**

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pins | grep -A2 GPIOAO
```

**4. إصلاح DT:**

```dts
/* في gpio node الخاص بـ AO domain */
&gpio_ao {
    reg = <0x0 0x14 0x0 0x8    /* gpio */
           0x0 0x14 0x0 0x8>;  /* pull (same range on G12A) */
    reg-names = "gpio", "pull";
    /* pull-enable مش محتاجه على G12A — meson_a1_parse_dt_extra نموذجها */
};
```

على G12A، `meson_a1_parse_dt_extra` هو الصح:

```c
int meson_a1_parse_dt_extra(struct meson_pinctrl *pc)
{
    pc->reg_pull   = pc->reg_gpio;   // نفس الـ register range
    pc->reg_pullen = pc->reg_gpio;
    pc->reg_ds     = pc->reg_gpio;
    return 0;
}
```

تأكد إن الـ `parse_dt` pointer في `meson_pinctrl_data` بيشاور على الـ function الصح.

#### الدرس المستفاد
على Meson SoCs مش كل الـ banks بتستخدم register ranges منفصلة. الـ AO bank خصوصاً بيختلف بين أجيال الـ SoCs. فشل `meson_map_resource()` silent (بيحوّل `ERR_PTR` لـ `NULL`) يخلي الـ pin configuration تصمت بدل ما تفشل، وده بيصعّب التشخيص.

---

### السيناريو 2: Industrial Gateway على Amlogic A113D — SPI لا يستجيب

#### العنوان
**الـ SPI flash مش بيتعرف على industrial gateway بعد kernel update**

#### السياق
Gateway صناعي بيستخدم Amlogic A113D (مبني على Meson AXG). بعد تحديث الـ kernel من 5.15 لـ 6.1، الـ SPI NOR flash بطّل يشتغل وأول رسالة في dmesg: `spi-nor: unrecognized JEDEC id bytes`.

#### المشكلة
الـ SPI pins (MOSI, MISO, CLK, CS) اتنقلوا من function مظبوط لـ GPIO mode بعد التحديث. السبب: تغيير في ترتيب `groups` array في SoC data file أدى لاختلاف `selector` index.

#### التحليل

الـ pinmux selection بيمشي كده:

```c
const char *meson_pmx_get_func_name(struct pinctrl_dev *pcdev,
                                    unsigned selector)
{
    struct meson_pinctrl *pc = pinctrl_dev_get_drvdata(pcdev);
    return pc->data->funcs[selector].name;  // selector = index في array
}

int meson_pmx_get_groups(struct pinctrl_dev *pcdev, unsigned selector,
                         const char * const **groups,
                         unsigned * const num_groups)
{
    *groups    = pc->data->funcs[selector].groups;
    *num_groups = pc->data->funcs[selector].num_groups;
    return 0;
}
```

الـ pinctrl core بيطابق بالاسم مش بالـ index، لكن لو الـ group name اتغير في الـ SoC-specific driver بين الإصدارين، وفي نفس الوقت الـ DT لسه بيستخدم الاسم القديم، هيبقى في mismatch.

`meson_get_group_pins()` بيرجع الـ pins بناءً على الـ selector:

```c
static int meson_get_group_pins(struct pinctrl_dev *pcdev, unsigned selector,
                                const unsigned **pins, unsigned *num_pins)
{
    *pins     = pc->data->groups[selector].pins;
    *num_pins = pc->data->groups[selector].num_pins;
    return 0;
}
```

لو `groups[selector].pins` بيشاور على pins غلط (بسبب إعادة ترتيب الـ array)، الـ mux هيتكتب على bits غلط في `reg_mux`.

الكتابة بتتم عبر `meson_calc_reg_and_bit()`:

```c
static void meson_calc_reg_and_bit(const struct meson_bank *bank,
                                   unsigned int pin,
                                   enum meson_reg_type reg_type,
                                   unsigned int *reg, unsigned int *bit)
{
    const struct meson_reg_desc *desc = &bank->regs[reg_type];

    *bit = (desc->bit + pin - bank->first) * meson_bit_strides[reg_type];
    *reg = (desc->reg + (*bit / 32)) * 4;  // byte offset
    *bit &= 0x1f;
}
```

لو `pin` القادم من group data غلط، `*reg` و `*bit` هيبقوا غلط كمان.

#### الحل

**1. تحقق من الـ active pinmux:**

```bash
# شوف إيه الـ function المضبوطة على SPI pins
cat /sys/kernel/debug/pinctrl/pinctrl-handles

# أو أكثر تفصيلاً
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pinmux-pins | grep -E "GPIOX_(0|1|2|3)"
```

**2. تحقق من الـ group names في kernel:**

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pingroups | grep spi
```

**3. قارن بالـ DT:**

```bash
grep -r "spi" /proc/device-tree/ 2>/dev/null | grep pinctrl
```

**4. إصلاح DT لو الاسم اتغير:**

```dts
/* قبل الإصلاح — الاسم القديم */
&spi0 {
    pinctrl-0 = <&spi0_pins>;
    pinctrl-names = "default";
};

/* بعد الإصلاح — الاسم الجديد في kernel 6.1 */
&spi0 {
    pinctrl-0 = <&spifc_pins>;  /* اسم الـ group اتغير */
    pinctrl-names = "default";
};
```

#### الدرس المستفاد
الـ pinctrl group names جزء من ABI الـ DT. أي تغيير في اسم `meson_pmx_group.name` في الـ SoC-specific driver يكسر backward compatibility مع DTBs القديمة. لازم تراجع `git log --follow` على الـ SoC pinctrl file لما تعمل kernel upgrade.

---

### السيناريو 3: IoT Sensor Board على Amlogic S905W — I2C يشتغل أحياناً بس أحياناً لأ

#### العنوان
**الـ I2C sensor على S905W بيعمل intermittent failures في humidity وtemperature readings**

#### السياق
board صغير لقياس البيئة بيستخدم S905W مع sensor HDC1080 على I2C bus 0. الـ firmware بيشتغل لساعات وبعدين يبدأ يطلع `-EIO` على قراءات الـ I2C، ولازم reset عشان يعود يشتغل.

#### المشكلة
الـ I2C pins (SDA, SCL) على Bank X مش عندهم pull-up مضبوط صح. الـ drive-strength اتضبط بـ `drive-strength-microamp = <500>` في DT، وده تحت الـ minimum اللازم لـ I2C fast-mode على PCB بيها capacitance عالية.

#### التحليل

الـ `meson_pinconf_set_drive_strength()` بتحوّل الـ microamp value لـ enum:

```c
static int meson_pinconf_set_drive_strength(struct meson_pinctrl *pc,
                                            unsigned int pin,
                                            u16 drive_strength_ua)
{
    // ...
    if (drive_strength_ua <= 500) {
        ds_val = MESON_PINCONF_DRV_500UA;   // أضعف drive
    } else if (drive_strength_ua <= 2500) {
        ds_val = MESON_PINCONF_DRV_2500UA;
    } else if (drive_strength_ua <= 3000) {
        ds_val = MESON_PINCONF_DRV_3000UA;
    } else if (drive_strength_ua <= 4000) {
        ds_val = MESON_PINCONF_DRV_4000UA;
    } else {
        dev_warn_once(pc->dev,
                      "pin %u: invalid drive-strength : %d , default to 4mA\n",
                      pin, drive_strength_ua);
        ds_val = MESON_PINCONF_DRV_4000UA;
    }

    // بيكتب 2-bit value في reg_ds
    ret = regmap_update_bits(pc->reg_ds, reg, 0x3 << bit, ds_val << bit);
```

الـ `meson_bit_strides[MESON_REG_DS] = 2` يعني كل pin بياخد 2 bits في الـ DS register:

```c
static const unsigned int meson_bit_strides[] = {
    1, 1, 1, 1, 1, 2, 1
    //              ^ MESON_REG_DS = index 5 → stride = 2
};
```

الحساب في `meson_calc_reg_and_bit()`:

```c
*bit = (desc->bit + pin - bank->first) * meson_bit_strides[reg_type];
// مثال: pin 20, bank->first=14, desc->bit=0, stride=2
// bit = (0 + 20 - 14) * 2 = 12
*reg = (desc->reg + (12 / 32)) * 4 = desc->reg * 4
*bit = 12 & 0x1f = 12
```

بعدين الكتابة: `0x3 << 12` mask و `ds_val << 12` value.

لو `pc->reg_ds == NULL` (S905W لا يدعم DS):

```c
if (!pc->reg_ds) {
    dev_err(pc->dev, "drive-strength not supported\n");
    return -ENOTSUPP;
}
```

الـ `meson_pinconf_set()` بترجع error لكن الـ pinctrl core أحياناً بيتجاهل `-ENOTSUPP` بصمت.

#### الحل

**1. تحقق إن الـ reg_ds موجود على الـ SoC ده:**

```bash
dmesg | grep "ds registers"
# لو شفت: "ds registers not found - skipping" → ds مش مدعوم
```

**2. تحقق من الـ pull config الحالية:**

```bash
# اقرأ pull state على I2C pins
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pinconf-pins | grep -A5 "GPIOX_10\|GPIOX_11"
```

**3. صلح DT — شيل الـ drive-strength وركّز على الـ pull-up:**

```dts
/* قبل — غلط */
&i2c0_sda_x {
    pins = "GPIOX_10";
    drive-strength-microamp = <500>;  /* مش مدعوم أو ضعيف جداً */
};

/* بعد — صح */
i2c0_default: i2c0-default-pins {
    mux {
        groups = "i2c0_sda_x", "i2c0_sck_x";
        function = "i2c0";
        bias-pull-up;   /* enable internal pull-up */
    };
};
```

**4. لو محتاج drive strength عال — استخدم external pull-up resistors 4.7kΩ وعطّل internal pull:**

```dts
mux {
    bias-disable;  /* external resistors موجودة */
};
```

#### الدرس المستفاد
`meson_pinconf_set_drive_strength()` بتعمل `dev_warn_once` لو القيمة أكبر من 4000 uA، لكن ما بتعملش warning لو الـ SoC مش بيدعم DS أصلاً. الـ intermittent failures على I2C غالباً بيبدأ بـ signal integrity — افحص pull-up وdrive-strength قبل ما تشك في الـ I2C driver نفسه.

---

### السيناريو 4: Custom Board Bring-up على Amlogic S922X — GPIO Output مش بيتغير

#### العنوان
**LED control GPIO على S922X بيجري set لكن LED مش بيضيء**

#### السياق
board مخصص للـ industrial automation بيستخدم Amlogic S922X. الـ engineer بيحاول يتحكم في status LED على `GPIOA_14` عبر sysfs، بيكتب `1` لـ `/sys/class/gpio/gpioXXX/value` لكن LED ما بيضيءش.

#### المشكلة
الـ pin اتضبط كـ output لكن ما اتعملش mux من function mode لـ GPIO mode أول. الـ `reg_mux` لسه بيحول الـ pin لـ peripheral function بدل GPIO.

#### التحليل

لما الـ userspace يكتب `1` على sysfs، الـ flow:

```
sysfs write → gpio_chip.set() → meson_gpio_set()
```

```c
static int meson_gpio_set(struct gpio_chip *chip, unsigned int gpio, int value)
{
    return meson_pinconf_set_drive(gpiochip_get_data(chip), gpio, value);
}

static int meson_pinconf_set_drive(struct meson_pinctrl *pc,
                                   unsigned int pin, bool high)
{
    return meson_pinconf_set_gpio_bit(pc, pin, MESON_REG_OUT, high);
}

static int meson_pinconf_set_gpio_bit(struct meson_pinctrl *pc,
                                      unsigned int pin,
                                      unsigned int reg_type,
                                      bool arg)
{
    const struct meson_bank *bank;
    unsigned int reg, bit;
    int ret;

    ret = meson_get_bank(pc, pin, &bank);  // إيجاد الـ bank
    if (ret)
        return ret;

    meson_calc_reg_and_bit(bank, pin, reg_type, &reg, &bit);
    return regmap_update_bits(pc->reg_gpio, reg,   // يكتب في gpio regmap
                              BIT(bit), arg ? BIT(bit) : 0);
}
```

الكود بيكتب في `MESON_REG_OUT` صح، لكن المشكلة إن الـ pin لسه على function `pwm_f` مثلاً، وده بيبقى configured في `reg_mux`. الـ mux hardware بيطغى على قيمة الـ GPIO output register.

الـ `meson_gpio_direction_output()` بيستدعي:

```c
static int meson_gpio_direction_output(struct gpio_chip *chip,
                                       unsigned gpio, int value)
{
    return meson_pinconf_set_output_drive(gpiochip_get_data(chip),
                                          gpio, value);
}

static int meson_pinconf_set_output_drive(struct meson_pinctrl *pc,
                                          unsigned int pin, bool high)
{
    int ret;
    ret = meson_pinconf_set_output(pc, pin, true);  // يضبط DIR
    if (ret)
        return ret;
    return meson_pinconf_set_drive(pc, pin, high);  // يضبط OUT
}
```

الكود بيضبط DIR وOUT بس ما بيلمسش `reg_mux`. ده عمداً — الـ mux switch بيتم عبر الـ pinmux ops مش hنا.

المشكلة إن `gpiochip_generic_request()` المسجلة في:

```c
pc->chip.request = gpiochip_generic_request;
```

بتستدعي `pinctrl_gpio_request()` اللي المفروض تعمل mux لـ GPIO function، لكن لو الـ pin مش معرّف في `pinctrl_gpio_range` صح هيفشل صامت.

#### الحل

**1. تحقق من الـ mux state قبل وبعد:**

```bash
# قبل export
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pinmux-pins | grep GPIOA_14

# export الـ pin
echo 414 > /sys/class/gpio/export  # رقم افتراضي، احسبه من base

# بعد export
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pinmux-pins | grep GPIOA_14
# المتوقع: "GPIO GPIOA_14"
```

**2. تحقق من الـ DIR register:**

```bash
# اقرأ direction
cat /sys/class/gpio/gpio414/direction
# لازم يكون: out
```

**3. تحقق من الـ OUT register مباشرة عبر devmem:**

```bash
# احسب register address من BANK definition
# مثلاً GPIOA bank: gpio base = 0xFF634400
# MESON_REG_OUT reg=1, bit=14 → reg offset = 4
devmem 0xFF634404 32  # قرأ OUT register
# Bit 14 لازم يكون 1
```

**4. لو المشكلة في الـ mux، صلح DT:**

```dts
/* تأكد إن GPIOA_14 مش allocated لـ function تانية */
/* أضف gpio-hog لو عايز تجبر GPIO mode من boot */
&gpio {
    led-hog {
        gpio-hog;
        gpios = <GPIOA_14 GPIO_ACTIVE_HIGH>;
        output-high;
        line-name = "status-led";
    };
};
```

#### الدرس المستفاد
`meson_gpio_set()` بتكتب في `reg_gpio` بس ما بتعملش mux. مشاكل GPIO "لا يستجيب" على Meson SoCs الأول تحقق من الـ mux register مش من الـ GPIO output register. استخدم `pinmux-pins` debugfs للتأكد.

---

### السيناريو 5: Automotive ECU على Amlogic A311D — Drive Strength يسبب EMI فوق الحد

#### العنوان
**الـ CAN bus signals على A311D بتعمل EMI تعدي حدود CISPR 25**

#### السياق
ECU للسيارات بيستخدم Amlogic A311D للمعالجة. الـ EMC testing بتظهر emissions عالية على ترددات مرتبطة بـ CAN bus. الـ board layout صح ومعندوش مشكلة، لكن الـ slew rate على الـ GPIO pins المتصلة بـ CAN transceiver عالي أكتر من اللازم.

#### المشكلة
الـ GPIO pins المتصلة بـ CAN transceiver enable/standby lines اتضبطت بـ `drive-strength-microamp = <4000>` افتراضياً. ده بيعمل edges حادة جداً وبتسبب EMI.

#### التحليل

لما الـ DT بيحدد `drive-strength-microamp`:

```dts
can0_ctrl: can0-ctrl-pins {
    pins = "GPIOX_5";
    drive-strength-microamp = <4000>;  /* عالي جداً */
};
```

الـ `meson_pinconf_set()` بتستقبل `PIN_CONFIG_DRIVE_STRENGTH_UA`:

```c
static int meson_pinconf_set(struct pinctrl_dev *pcdev, unsigned int pin,
                             unsigned long *configs, unsigned num_configs)
{
    // ...
    case PIN_CONFIG_DRIVE_STRENGTH_UA:
        arg = pinconf_to_config_argument(configs[i]);  // arg = 4000
        // ...
        ret = meson_pinconf_set_drive_strength(pc, pin, arg);
        break;
```

في `meson_pinconf_set_drive_strength()`:

```c
} else if (drive_strength_ua <= 4000) {
    ds_val = MESON_PINCONF_DRV_4000UA;  // 2-bit value = 0x3
}

// بيكتب 0x3 في 2 bits
ret = regmap_update_bits(pc->reg_ds, reg, 0x3 << bit, ds_val << bit);
```

الـ 4 values المتاحة (`500uA`, `2500uA`, `3000uA`, `4000uA`) بتتحكم في output impedance، وبالتالي في الـ slew rate. مفيش حاجة اسمها "slew rate control" منفصلة على Meson — الـ drive strength هو الـ control الوحيد.

لو عايز تتحقق من القيمة الحالية:

```c
static int meson_pinconf_get_drive_strength(struct meson_pinctrl *pc,
                                            unsigned int pin,
                                            u16 *drive_strength_ua)
{
    // يقرأ 2 bits ويحوّلهم لـ uA value
    switch ((val >> bit) & 0x3) {
    case MESON_PINCONF_DRV_500UA:   *drive_strength_ua = 500;   break;
    case MESON_PINCONF_DRV_2500UA:  *drive_strength_ua = 2500;  break;
    case MESON_PINCONF_DRV_3000UA:  *drive_strength_ua = 3000;  break;
    case MESON_PINCONF_DRV_4000UA:  *drive_strength_ua = 4000;  break;
    }
}
```

لو `MESON_PINCONF_DRV_500UA = 0` (أول قيمة في enum)، الـ register بيبقى `0x0` وده معناه أضعف drive.

#### الحل

**1. قرأ الـ DS register الحالي:**

```bash
# اقرأ قيمة DS على GPIOX_5
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pinconf-pins | grep -A3 GPIOX_5
# المتوقع: drive-strength-microamp:4000
```

**2. صلح DT — قلل الـ drive strength:**

```dts
can0_ctrl: can0-ctrl-pins {
    pins = "GPIOX_5";
    drive-strength-microamp = <500>;   /* أضعف drive → edges أبطأ → أقل EMI */
    bias-disable;                       /* مش محتاج pull على output line */
};
```

**3. تحقق من runtime:**

```bash
# بعد تطبيق DT الجديد
cat /sys/kernel/debug/pinctrl/pinctrl-meson/pinconf-pins | grep -A3 GPIOX_5
# المتوقع: drive-strength-microamp:500
```

**4. تحقق برمجياً إن الـ DS register اتغير:**

```bash
# احسب DS register address من BANK_DS macro
# GPIOX bank DS: reg base + offset
devmem 0xFF634820 32   # DS register للـ bank X
# Bits 10:11 (للـ pin 5, stride=2 → bit=10) لازم يكونوا 0x0 (500uA)
```

**5. لو مش كفاية — أضف series resistor على الـ PCB:**
الـ drive strength options محدودة (4 فقط). لو 500uA لسه بتسبب EMI، ضيف 33Ω series resistor مع 100pF pull-down على الـ signal line.

#### الدرس المستفاد
الـ `meson_bit_strides[MESON_REG_DS] = 2` يعني الـ DS register بياخد ضعف المساحة مقارنة بباقي الـ registers. غلطة شائعة هي حساب DS register offset غلط يدوياً في tests. استخدم دايماً `pinconf-pins` debugfs للتحقق من القيم بدل manual register reading. في automotive، الـ drive strength مش بس performance — هو EMC parameter.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

الـ LWN.net هو المرجع الأهم لمتابعة تطور الـ pinctrl subsystem في الـ Linux kernel — كل patch set مهم بيتناقش هناك قبل ما يتدمج.

| المقال | الأهمية |
|--------|---------|
| [Pinctrl driver for Amlogic Meson SoCs](https://lwn.net/Articles/616225/) | أول إرسال للـ driver الأساسي لـ Meson |
| [Amlogic Meson pinctrl driver (v3)](https://lwn.net/Articles/620822/) | النسخة المستقرة اللي اتدمجت — بتوضح البنية المشتركة لكل SoCs |
| [pinctrl: meson-a1: add pinctrl driver](https://lwn.net/Articles/804174/) | إضافة دعم Meson-A1 مع تغييرات في الـ register layout |
| [pinctrl: meson-s4: add pinctrl driver](https://lwn.net/Articles/879912/) | أحدث جيل من الـ Amlogic SoCs |
| [Add pinctrl driver support for Amlogic T7 SoCs](https://lwn.net/Articles/945285/) | دعم عائلة T7 الجديدة |
| [Add support for Amlogic S7/S7D/S6 pinctrl](https://lwn.net/Articles/1022709/) | أحدث إضافات 2024 |
| [irqchip: meson: add support for the gpio interrupt controller](https://lwn.net/Articles/703946/) | دعم الـ GPIO IRQ المرتبط بالـ pinctrl |
| [The pin control subsystem](https://lwn.net/Articles/468759/) | شرح شامل للـ pinctrl subsystem من المطور الأصلي Linus Walleij |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | النسخة المستقرة من الـ subsystem قبل الدمج |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/) | إضافة الـ pinconf interface اللي بيستخدمه `pinctrl-meson.c` |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | تطوير الـ `PIN_CONFIG_*` parameters |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | الـ documentation الرسمي على LWN |

---

### التوثيق الرسمي في الـ Kernel

```
Documentation/driver-api/pinctl.rst          ← الـ pinctrl subsystem API كاملة
Documentation/devicetree/bindings/pinctrl/   ← DT bindings لكل SoCs
Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl.yaml
```

الـ kernel documentation الرسمي على الويب:
- [PINCTRL subsystem — kernel.org](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html)
- [Documentation/pinctrl.txt — kernel.org](https://www.kernel.org/doc/Documentation/pinctrl.txt)

---

### Patchwork — مناقشات الـ Mailing List

الـ patches الأساسية اللي شكّلت الـ driver:

| الـ Patch | الموضوع |
|-----------|---------|
| [pinctrl: meson: add support for GPIO interrupts (v6)](https://patchwork.kernel.org/project/linux-amlogic/patch/a86d38af-9103-7557-9986-2dc187815569@gmail.com/) | إضافة الـ GPIO IRQ support |
| [pinctrl: meson: fix drive strength register calculation](https://patchwork.kernel.org/project/linux-amlogic/patch/20200610041329.12948-1-hhk7734@gmail.com/) | إصلاح حساب الـ `MESON_REG_DS` لـ banks أكبر من 16 pin |
| [pinctrl: add compatible for Amlogic Meson A1 (v3)](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1571050492-6598-2-git-send-email-qianggui.song@amlogic.com/) | دعم A1 مع الـ `meson_a1_parse_dt_extra` |
| [pinctrl: meson: add a new callback for SoCs fixup](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1572004167-24153-3-git-send-email-qianggui.song@amlogic.com/) | إضافة الـ `parse_dt` callback في `meson_pinctrl_data` |
| [pinctrl: meson: fix gxbb ao pull register bits](https://patchwork.kernel.org/project/linux-amlogic/patch/20181029151340.9087-2-jbrunet@baylibre.com/) | إصلاح الـ AO bank pull registers |
| [Re: pinctrl: meson: Fix typo in device table macro](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2596226.html) | مناقشة على LKML |
| [LKML: pinctrl: add compatible for Meson A1 (v2)](https://lkml.iu.edu/hypermail/linux/kernel/1910.1/00326.html) | أرشيف LKML |

---

### eLinux.org

- [Introduction to pin muxing and GPIO control under Linux (PDF)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) — محاضرة من ELC-2021 بتشرح الـ pinmux وGPIO تحت Linux من الأساسيات للتطبيق
- [Tests: i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) — مثال عملي على استخدام الـ pinctrl مع I2C multiplexing
- [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) — روابط لمصادر الـ kernel بشكل عام

---

### kernelnewbies.org — متابعة التغييرات

صفحات الـ kernel releases على kernelnewbies بتوضح التغييرات اللي اتضافت على الـ pinctrl subsystem في كل إصدار:

| الإصدار | الرابط |
|---------|--------|
| Linux 6.8 | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) |
| Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| Linux 6.14 | [kernelnewbies.org/Linux_6.14](https://kernelnewbies.org/Linux_6.14) |
| Linux 6.17 | [kernelnewbies.org/Linux_6.17](https://kernelnewbies.org/Linux_6.17) |
| Linux Changes (كل الإصدارات) | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) |

---

### الكود المصدري المباشر على Elixir

```
https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl/meson/pinctrl-meson.c
https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl/meson/pinctrl-meson.h
https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/pinctrl.h
https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/pinmux.h
https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/pinconf-generic.h
```

---

### الكتب الموصى بها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الفصل 9**: I/O Memory Access — يشرح `ioremap` و`regmap` اللي بيستخدمهم `meson_map_resource()`
- **الفصل 14**: The Linux Device Model — يشرح `platform_device` و`devm_*` APIs
- **متاح مجاناً**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 17**: Devices and Modules — الـ `platform_driver` registration وDrivers lifecycle
- **الفصل 14**: The Block I/O Layer — فهم الـ regmap abstraction

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 15**: Debugging Embedded Linux Applications — `debugfs` وsysfs للـ pinctrl
- **الفصل 16**: Kernel Debugging Techniques — استخدام `/sys/kernel/debug/pinctrl/`

#### Professional Embedded ARM Development — James Langbridge
- يغطي الـ pinmux/pinconf في سياق ARM SoCs بما فيها Amlogic

---

### مسار الـ Git History للـ Driver

```bash
# تتبع تاريخ الملف الأساسي
git log --oneline -- drivers/pinctrl/meson/pinctrl-meson.c

# أول commit أضاف الـ driver (Beniamino Galvani, 2014)
git log --follow --oneline -- drivers/pinctrl/meson/

# البحث عن commits خاصة بـ drive strength
git log --oneline --all --grep="meson.*drive.strength"

# البحث عن commits خاصة بـ A1 SoC
git log --oneline --all --grep="meson.*a1\|meson_a1"
```

---

### مسارات الـ sysfs وdebugfs للفحص العملي

```bash
# عرض كل الـ pin controllers المسجلة
ls /sys/kernel/debug/pinctrl/

# فحص الـ muxing الحالي
cat /sys/kernel/debug/pinctrl/<controller>/pinmux-functions
cat /sys/kernel/debug/pinctrl/<controller>/pinmux-pins

# فحص الـ pin configuration
cat /sys/kernel/debug/pinctrl/<controller>/pinconf-pins

# على بورد Amlogic فعلي
cat /sys/kernel/debug/pinctrl/*/pins | grep -i meson
```

---

### كلمات البحث للمزيد من المعلومات

للبحث في Google أو Elixir أو patchwork:

```
pinctrl meson amlogic regmap
pinctrl_ops pinmux_ops pinconf_ops linux kernel
meson_bank meson_pinctrl_data
devm_pinctrl_register linux
gpiochip_add_data pinctrl
amlogic gpio pinmux device tree binding
pinconf_generic_dt_node_to_map_all
PIN_CONFIG_BIAS_PULL_UP kernel
MESON_REG_PULLEN MESON_REG_DS
meson8_aobus_parse_dt_extra
meson_a1_parse_dt_extra
regmap_update_bits gpio driver
```

---

### مصادر Device Tree Bindings

```bash
# الـ DT binding documentation
Documentation/devicetree/bindings/pinctrl/amlogic,meson-pinctrl.yaml

# أمثلة فعلية في الـ DTS files
arch/arm64/boot/dts/amlogic/
arch/arm/boot/dts/amlogic/

# البحث عن استخدام الـ pinctrl في DTS
grep -r "amlogic,meson.*-pinctrl" arch/arm*/boot/dts/amlogic/
```

---

### ملخص أهم المصادر

| الأولوية | المصدر | لماذا؟ |
|----------|--------|--------|
| **أولاً** | [LWN: Amlogic Meson pinctrl driver](https://lwn.net/Articles/620822/) | يشرح البنية المعمارية للـ driver |
| **ثانياً** | [LWN: The pin control subsystem](https://lwn.net/Articles/468759/) | فهم الـ subsystem من الأساس |
| **ثالثاً** | [kernel.org pinctrl docs](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html) | الـ API الرسمي |
| **رابعاً** | [ELC-2021 PDF on elinux](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) | شرح بصري عملي |
| **خامساً** | [Patchwork: GPIO IRQ support](https://patchwork.kernel.org/project/linux-amlogic/patch/a86d38af-9103-7557-9986-2dc187815569@gmail.com/) | أهم تطوير على الـ driver |
## Phase 8: Writing simple module

### الـ Function المختارة: `meson_pinctrl_probe`

الـ function دي exported بـ `EXPORT_SYMBOL_GPL` وبتتعمل call لما أي Amlogic Meson SoC بيعمل register لـ pin controller بتاعه — ده معناه إننا نقدر نـ hook عليها بـ kprobe ونشوف بالضبط إيه platform device اللي بيتسجل وامتى.

---

### الـ Complete Module

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe module: trace meson_pinctrl_probe() calls
 * Hooks into the Amlogic Meson pinctrl probe function and logs
 * the platform device name whenever a pin controller registers.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/platform_device.h>

/* kprobe struct — one per hooked symbol */
static struct kprobe kp_probe;

/*
 * pre_handler — runs just before meson_pinctrl_probe() executes.
 * regs: CPU register snapshot at the probe point.
 * On arm64/x86_64 the first argument (pdev) lives in regs->regs[0] / regs->di.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
#ifdef CONFIG_X86_64
    /* On x86-64: first arg is in RDI */
    struct platform_device *pdev = (struct platform_device *)regs->di;
#elif defined(CONFIG_ARM64)
    /* On arm64: first arg is in x0 */
    struct platform_device *pdev = (struct platform_device *)regs->regs[0];
#else
    /* Generic fallback — may not work on all arches */
    struct platform_device *pdev = NULL;
#endif

    if (pdev)
        pr_info("[meson-kprobe] meson_pinctrl_probe() called: pdev->name=\"%s\" id=%d\n",
                pdev->name ? pdev->name : "(null)",
                pdev->id);
    else
        pr_info("[meson-kprobe] meson_pinctrl_probe() called (arch not supported for arg extraction)\n");

    return 0; /* 0 = continue normal execution */
}

/*
 * post_handler — runs right after meson_pinctrl_probe() returns.
 * flags: architecture-specific flags (not used here).
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * On x86-64 the return value sits in RAX.
     * On arm64 it sits in x0.
     */
#ifdef CONFIG_X86_64
    long ret = (long)regs->ax;
#elif defined(CONFIG_ARM64)
    long ret = (long)regs->regs[0];
#else
    long ret = 0;
#endif

    pr_info("[meson-kprobe] meson_pinctrl_probe() returned: %ld (%s)\n",
            ret, ret == 0 ? "OK" : "ERROR");
}

static int __init meson_kprobe_init(void)
{
    int ret;

    /* تحديد الـ symbol اللي هنـ hook عليه */
    kp_probe.symbol_name = "meson_pinctrl_probe";
    kp_probe.pre_handler  = handler_pre;
    kp_probe.post_handler = handler_post;

    ret = register_kprobe(&kp_probe);
    if (ret < 0) {
        pr_err("[meson-kprobe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[meson-kprobe] kprobe planted at %s (%px)\n",
            kp_probe.symbol_name, kp_probe.addr);
    return 0;
}

static void __exit meson_kprobe_exit(void)
{
    /* لازم نشيل الـ kprobe قبل ما الـ module يتـ unload عشان منـ crash الـ kernel */
    unregister_kprobe(&kp_probe);
    pr_info("[meson-kprobe] kprobe removed from %s\n", kp_probe.symbol_name);
}

module_init(meson_kprobe_init);
module_exit(meson_kprobe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe tracer for meson_pinctrl_probe() — Amlogic Meson pinctrl");
```

---

### Makefile للـ Module

```makefile
obj-m := meson_kprobe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# تحميل الـ module
sudo insmod meson_kprobe.ko

# مشاهدة الـ output في real-time
sudo dmesg -w | grep meson-kprobe

# تفريغ الـ module
sudo rmmod meson_kprobe
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
#include <linux/platform_device.h>
```

الـ `kprobes.h` بيجيب تعريف `struct kprobe` وكل الـ API المتعلقة بيه. الـ `platform_device.h` محتاجينه عشان نـ cast الـ argument الأول لـ `meson_pinctrl_probe` اللي هو من النوع `struct platform_device *`.

---

#### الـ `struct kprobe`

```c
static struct kprobe kp_probe;
```

ده الـ descriptor اللي بيقول للـ kernel "اعمل breakpoint على الـ symbol ده وادي الـ control للـ handlers دول". الـ kernel بيحط `INT3` (x86) أو `BRK` (arm64) في أول byte من الـ function.

---

#### الـ `handler_pre` — الـ Callback قبل التنفيذ

الـ handler ده بيتعمل call قبل ما `meson_pinctrl_probe` تشتغل. بناخد `regs` اللي هو snapshot من الـ CPU registers، ومنه بنجيب أول argument (الـ `pdev`). محتاجين `#ifdef` لأن كل architecture بتحط الـ arguments في registers مختلفة — `rdi` على x86-64 و `x0` على arm64.

---

#### الـ `handler_post` — الـ Callback بعد التنفيذ

بعد ما الـ function ترجع، بناخد الـ return value من `regs->ax` (x86-64) أو `regs->regs[0]` (arm64) ونطبعه. ده مفيد عشان نعرف لو الـ probe نجح ولا ولا.

---

#### الـ `module_init` — التسجيل

```c
kp_probe.symbol_name = "meson_pinctrl_probe";
ret = register_kprobe(&kp_probe);
```

الـ kernel بيـ resolve الـ symbol name لعنوان حقيقي في memory، وبيحط الـ breakpoint فيه. لو الـ symbol مش موجود أو مش exported للـ kprobe subsystem، الـ `register_kprobe` بترجع خطأ.

---

#### الـ `module_exit` — إلغاء التسجيل

```c
unregister_kprobe(&kp_probe);
```

ضروري جداً إننا نشيل الـ kprobe قبل ما الـ module يتـ unload. لو الـ breakpoint فضل في memory بعد ما الـ handler code اتشال، أي call لـ `meson_pinctrl_probe` هتعمل kernel panic.

---

### مثال على الـ Output

```
[  12.345678] [meson-kprobe] kprobe planted at meson_pinctrl_probe (ffff800010ab1234)
[  12.401234] [meson-kprobe] meson_pinctrl_probe() called: pdev->name="ff634400.bus" id=-1
[  12.401890] [meson-kprobe] meson_pinctrl_probe() returned: 0 (OK)
[  12.450000] [meson-kprobe] meson_pinctrl_probe() called: pdev->name="ff800000.bus" id=-1
[  12.450512] [meson-kprobe] meson_pinctrl_probe() returned: 0 (OK)
```

لاحظ إن في Amlogic Meson SoCs عادةً بيتعمل probe لـ controller أكتر من مرة — واحد للـ AO domain (Always-On) وواحد للـ EE domain (Everything Else). الـ kprobe بيـ catch الاتنين.
