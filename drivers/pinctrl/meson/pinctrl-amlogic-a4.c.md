## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Subsystem؟

الملف ده جزء من **AMLOGIC PINCTRL DRIVER** — وده subsystem موجود جوه **Linux Pinctrl Framework** اللي بيتحكم في الـ pins الجسمانية اللي على الـ SoC بتاع Amlogic.

المسؤول عنه في MAINTAINERS هو **Xianwei Zhao** `<xianwei.zhao@amlogic.com>` وبيشتغل تحت **linux-amlogic mailing list**.

---

### الصورة الكبيرة — بالمثال

تخيل عندك لوحة مفاتيح فيها 100 زرار. كل زرار ممكن يعمل أكتر من حاجة:
- الزرار "A" ممكن يكتب حرف A، أو لو ضغطت Fn يعمل وظيفة تانية (زي رفع الصوت).
- نفس الزرار ممكن يتحول لـ GPIO عادي يقرأ حالة مفتاح.

الـ **SoC** بتاع Amlogic (زي A4, S7, S6) فيه مئات الـ **pins** الجسمانية على الشريحة. كل pin ممكن يشتغل في وظايف مختلفة حسب الإعداد:
- **UART** (تواصل serial)
- **I2C** (تواصل مع sensors)
- **SPI** (تواصل مع flash)
- **GPIO** عادي (تشغيل LED، قراءة زرار)

مشكلة: ال hardware بيستخدم **registers** في الذاكرة لتحديد وظيفة كل pin. لو كل driver حاول يعدل الـ registers دي بنفسه — chaos. لازم يبقى في **مدير مركزي** يتحكم في كل ده.

ده هو دور الـ **Pinctrl Subsystem** في Linux، والملف ده هو **الـ driver** اللي بيكلم الـ hardware بتاع Amlogic جيل A4 وما بعده.

---

### ليه الملف ده موجود؟

الـ Amlogic عندها أجيال كتير من الـ SoCs وكل جيل له طريقة مختلفة شوية في الـ pin registers. الأجيال القديمة (Meson8, GXBB, G12A) كل واحدة فيها ملف C منفصل. المشكلة: كل ما جاء SoC جديد، لازم ملف C جديد كامل — **code duplication بشكل مزعج**.

الملف `pinctrl-amlogic-a4.c` جاء بنهج جديد: **مرن وقابل للتوسع**. يعني لدعم SoC جديد، بس تضيف **DTS node** جديد من غير ما تكتب C file جديدة. وده اللي كتبوه صراحةً في Kconfig:

```
This driver is simplify subsequent support for new amlogic SoCs,
to support new Amlogic SoCs, only need to add the corresponding dts file,
no additional binding header files or C file are added.
```

---

### الـ Driver بيعمل إيه بالظبط؟

الـ driver بيتحكم في **3 وظايف أساسية** لكل pin:

#### 1. الـ Pin Multiplexing (Pinmux)
بيحدد الـ pin شغال في أنهي **function** — الـ register بتاع الـ MUX بيتقسم لـ fields 4-bit، كل field بيحدد الوظيفة.

```
[  MUX REG  ]
| 4b | 4b | 4b | 4b |  ← كل 4 bits = function لـ pin واحد
  P0   P1   P2   P3
```

#### 2. الـ GPIO Control
لما الـ pin بيتحول لـ GPIO، الـ driver بيتحكم في 5 registers لكل bank:

```c
#define AML_REG_PULLEN  0   /* enable/disable pull resistor */
#define AML_REG_PULL    1   /* pull up or pull down */
#define AML_REG_DIR     2   /* input or output direction */
#define AML_REG_OUT     3   /* output value (high/low) */
#define AML_REG_IN      4   /* read input value */
#define AML_REG_DS      5   /* drive strength (current) */
```

#### 3. الـ Pin Configuration (Pinconf)
تحديد خصائص الـ pin: pull-up/pull-down، drive strength (500µA لـ 4000µA).

---

### الـ Multi-Mux Problem — القصة الغريبة

في بعض الـ SoCs زي S7 وS6، في pins معينة **بتستخدم register بنك تاني** للـ mux بدل بنكها الأصلي. ده بسبب قرارات hardware design غريبة.

مثلاً في S7: الـ pins من `GPIOX[16]` لـ `GPIOX[19]` بتستخدم register بتاع بنك `GPIOCC` من offset 24 بدل register بنك GPIOX.

```c
static const struct multi_mux multi_mux_s7[] = {
    {
        .m_bank_id  = AMLOGIC_GPIO_CC,  /* main bank (GPIOCC) */
        .m_bit_offs = 24,               /* bit offset in GPIOCC's mux reg */
        .sid = (AMLOGIC_GPIO_X << 8) + 16, /* subordinate start: GPIOX[16] */
        .eid = (AMLOGIC_GPIO_X << 8) + 19, /* subordinate end:   GPIOX[19] */
    },
};
```

الـ `struct multi_mux` هو الحل اللي الـ driver بيستخدمه للتعامل مع الشذوذ ده.

---

### رحلة الـ Driver من البداية لحد ما يشتغل

```
Kernel Boot
    │
    ▼
platform_driver_probe()
    │
    ├─► اقرأ DTS: كام GPIO bank؟ كام function؟
    │       └─► aml_pctl_dt_child_count()
    │
    ├─► سجّل كل GPIO bank
    │       ├─► اقرأ reg addresses (gpio, mux, ds)
    │       ├─► map الـ registers في الذاكرة عبر regmap
    │       └─► سجّل gpio_chip في kernel
    │
    ├─► سجّل الـ functions وgroups من DTS
    │       └─► aml_pctl_parse_functions()
    │
    └─► سجّل الـ pinctrl_dev في kernel
            └─► devm_pinctrl_register()
```

---

### البنى الأساسية (Key Structs)

| Struct | وظيفتها |
|--------|---------|
| `aml_pinctrl` | الـ main state للـ driver — بيحتوي على كل banks وfunctions وgroups |
| `aml_gpio_bank` | بيمثّل GPIO bank واحد (GPIOA, GPIOB...) مع الـ regmaps بتاعته |
| `aml_pio_control` | reg offsets وbit offsets لكل نوع register في الـ bank |
| `aml_pmx_func` | function واحدة (UART, SPI, ...) ومجموعة الـ pin groups بتاعتها |
| `aml_pctl_group` | group واحدة من pins بتاعت function معينة |
| `multi_mux` | حالات الـ pins اللي بتستخدم register بنك مختلف للـ mux |
| `aml_pctl_data` | الـ chip-specific data (بتتحدد من DTS compatible string) |

---

### الـ Pin Numbering Scheme

الـ driver بيستخدم scheme ذكي لترقيم الـ pins:

```
pin_id = bank_id << 8 + offset_in_bank

مثال:
  GPIOX[5] = (23 << 8) + 5 = 0x1705
  GPIOA[0] = (0  << 8) + 0 = 0x0000
```

ده بيسهّل معرفة الـ bank من الـ pin_id مباشرة بعملية shift بسيطة.

---

### الـ SoCs المدعومة

| Compatible String | الـ SoC | ملاحظات |
|------------------|---------|---------|
| `amlogic,pinctrl-a4` | A4 | base, بدون multi_mux |
| `amlogic,pinctrl-a5` | A5 | يستخدم a4 كـ fallback |
| `amlogic,pinctrl-s7` | S7 | فيه multi_mux لـ GPIOX[16:19] |
| `amlogic,pinctrl-s7d` | S7D | يستخدم s7 كـ fallback |
| `amlogic,pinctrl-s6` | S6 | فيه multi_mux لـ GPIOX وGPIOD |

---

### الملفات المرتبطة

#### الملف ده مباشرة:
- `/workspace/external/linux/drivers/pinctrl/meson/pinctrl-amlogic-a4.c` — الـ driver الرئيسي

#### الـ DT Bindings:
- `/workspace/external/linux/Documentation/devicetree/bindings/pinctrl/amlogic,pinctrl-a4.yaml` — مواصفات الـ DTS

#### الـ Headers المهمة:
- `/workspace/external/linux/include/dt-bindings/pinctrl/amlogic,pinctrl.h` — bank IDs وماكرو AML_PINMUX
- `/workspace/external/linux/include/linux/pinctrl/pinctrl.h` — Linux pinctrl core structs
- `/workspace/external/linux/include/linux/pinctrl/pinmux.h` — pinmux_ops interface
- `/workspace/external/linux/include/linux/pinctrl/pinconf.h` — pinconf_ops interface

#### الـ Pinctrl Core:
- `/workspace/external/linux/drivers/pinctrl/core.h` — internal core structs
- `/workspace/external/linux/drivers/pinctrl/pinctrl-utils.h` — utility functions (reserve_map, add_map_mux)
- `/workspace/external/linux/drivers/pinctrl/pinconf.h` — generic pinconf helpers

#### الـ Drivers الأقدم (للمقارنة):
- `/workspace/external/linux/drivers/pinctrl/meson/pinctrl-meson.c` — الجيل القديم
- `/workspace/external/linux/drivers/pinctrl/meson/pinctrl-meson-g12a.c` — G12A specific
- `/workspace/external/linux/drivers/pinctrl/meson/pinctrl-amlogic-c3.c` — C3 chip
- `/workspace/external/linux/drivers/pinctrl/meson/pinctrl-amlogic-t7.c` — T7 chip

#### الـ DTS Files (SoC specific):
- `/workspace/external/linux/arch/arm64/boot/dts/amlogic/` — DTS files للـ SoCs المدعومة
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة: ليه الـ Pinctrl موجود أصلاً؟

في أي SoC حديث زي Amlogic A4/S7/S6، كل pin جسمانيًا على الـ chip ممكن يعمل أكتر من وظيفة. الـ pin الواحد ممكن يكون:

- **GPIO** عادي (input/output)
- **UART TX** لـ serial port
- **I2C SDA** لـ I2C bus
- **PWM** output
- **SPI MOSI** لـ SPI bus

الـ hardware بيحل ده عن طريق **mux registers** — كل 4 bits في الـ register بتحدد الـ function للـ pin ده. المشكلة إن كل SoC عنده layout مختلف تمامًا لـ registers دي، وعدد الـ pins مختلف، والـ functions المتاحة مختلفة.

قبل الـ pinctrl framework، كل driver كان بيعمل `ioremap` على الـ registers المتعلقة بيه وبيعمل `read-modify-write` على الـ bits بتاعته مباشرةً. النتيجة؟

- **race conditions**: driver A وdriver B بيكتبوا على نفس الـ register في نفس الوقت
- **لا يوجد ownership**: مفيش حاجة بتمنع driver X من تغيير pin محجوز لـ driver Y
- **تكرار الكود**: كل SoC فيه copy/paste من الـ mux logic نفسه بس بأرقام مختلفة
- **Device Tree integration معدومة**: مفيش طريقة standard لـ DT node يقول "أنا عايز الـ UART على الـ pins دي"

### الحل: الـ Pinctrl Framework

الـ kernel حل المشكلة بـ **abstraction layer** اسمه `pinctrl`. الفكرة الجوهرية:

> **كل SoC بيسجّل pin controller واحد بيملك كل الـ pins، وأي driver عايز pin لازم يطلبه من الـ controller ده.**

الـ framework بيوفر:
1. **Central registry** للـ pins وأسمائها
2. **Ownership tracking**: مين حاجز الـ pin ده دلوقتي
3. **Conflict detection**: منع اتنين يحجزوا نفس الـ pin
4. **DT parsing**: تحويل `pinctrl-0 = <&uart_pins>` لـ register writes فعلية
5. **Generic pin config**: pull-up/pull-down/drive-strength بـ API موحدة

---

### الـ Big Picture Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │           Consumer Drivers (users)           │
                    │  UART driver  I2C driver  SPI driver  ...   │
                    │       │           │           │              │
                    │  pinctrl_get() + pinctrl_select_state()     │
                    └──────────────────┬──────────────────────────┘
                                       │
                    ┌──────────────────▼──────────────────────────┐
                    │          Pinctrl Core (kernel/pinctrl/)      │
                    │                                              │
                    │  ┌─────────────┐  ┌──────────────────────┐  │
                    │  │ pinctrl_dev │  │   State Machine       │  │
                    │  │  (registry) │  │ (default/idle/sleep)  │  │
                    │  └─────────────┘  └──────────────────────┘  │
                    │                                              │
                    │  Calls ops vtables:                          │
                    │  pctlops → pmxops → confops                 │
                    └──────────────────┬──────────────────────────┘
                                       │  calls vtable functions
                    ┌──────────────────▼──────────────────────────┐
                    │     Pin Controller Driver (this file)        │
                    │   drivers/pinctrl/meson/pinctrl-amlogic-a4  │
                    │                                              │
                    │  aml_pctrl_ops   aml_pmx_ops  aml_pinconf_ops│
                    │        │               │            │        │
                    │        └───────────────┴────────────┘        │
                    │                       │                      │
                    │         aml_pinctrl (main state struct)      │
                    │         aml_gpio_bank[] (per GPIO bank)      │
                    └──────────────────┬──────────────────────────┘
                                       │  regmap_read/write
                    ┌──────────────────▼──────────────────────────┐
                    │            Hardware Registers                 │
                    │                                              │
                    │  ┌──────────┐ ┌──────────┐ ┌────────────┐  │
                    │  │ MUX regs │ │GPIO regs  │ │  DS regs   │  │
                    │  │(4bits/pin│ │DIR/IN/OUT │ │drive-str.  │  │
                    │  │func sel) │ │PULL/PULLEN│ │(2bits/pin) │  │
                    │  └──────────┘ └──────────┘ └────────────┘  │
                    └─────────────────────────────────────────────┘
```

---

### التشبيه الحقيقي: مكتب الاتصالات الوطني

تخيل إن الـ SoC هو مبنى فيه **200 خط تليفون** (pins). الـ Pinctrl framework هو **مكتب الاتصالات** المسؤول عن توصيل كل خط لخدمة.

| عنصر في التشبيه | المقابل في الـ kernel |
|---|---|
| مبنى الـ 200 خط | الـ SoC بـ pins بتاعته |
| مكتب الاتصالات المركزي | الـ `pinctrl_dev` (core registry) |
| سجل توزيع الخطوط | قائمة `pinctrl_pin_desc[]` |
| لوحة الـ patch panel (الـ MUX) | `pinmux_ops` + MUX registers |
| قواعد تشغيل الخط (impedance, voltage) | `pinconf_ops` (pull, drive-strength) |
| شركة UART تطلب خط | UART driver يعمل `pinctrl_select_state()` |
| شركة I2C تطلب خط تاني | I2C driver يطلب pins تانية |
| ممنوع شركتين على نفس الخط | Ownership tracking في الـ core |
| موظف مكتب الاتصالات | الـ controller driver (الملف ده) |
| تعليمات تشغيل الـ patch panel | الـ vtable ops structures |
| كتالوج الخدمات المتاحة | Device Tree pinctrl nodes |

لما شركة UART (driver) تيجي تقول "عايز خط رقم 45 و46"، مكتب الاتصالات (core) بيسأل الموظف (driver) عن إزاي تعمل التوصيل. الموظف بيروح يعمل write على لوحة الـ patch panel (MUX registers). لو شركة تانية جت تطلب نفس الخط، مكتب الاتصالات يرفض طلبها.

---

### الـ Core Abstraction: Pin Groups و Functions

الفكرة المحورية في الـ pinctrl framework هي التفريق بين **pin group** و**function**:

- **Pin Group**: مجموعة pins مرتبطة ببعض. مثلًا `uart0_pins` = {pin 45, pin 46}.
- **Function**: الوظيفة المطلوبة. مثلًا `uart0` = "شغّل UART على الـ group ده".

الـ mapping بيبقى: `function` → يدعم → قائمة `groups` → كل group فيها → قائمة `pins` + قيمة الـ mux لكل pin.

في الملف ده، ده بيتخزن في:
```c
struct aml_pmx_func {
    const char   *name;     // اسم الـ function زي "uart0"
    const char  **groups;   // أسماء الـ groups اللي بتدعمها
    unsigned int  ngroups;
};

struct aml_pctl_group {
    const char    *name;    // اسم الـ group زي "uart0_pins"
    unsigned int   npins;   // عدد الـ pins في الـ group
    unsigned int  *pins;    // أرقام الـ pins (global pin numbers)
    unsigned int  *func;    // قيمة الـ mux المقابلة لكل pin (0xf max)
};
```

الـ groups والـ functions بتتبنى **ديناميكيًا من الـ Device Tree** في `aml_pctl_probe_dt()` — مفيش static tables في الكود، كل حاجة بتيجي من الـ DT nodes.

---

### تشريح الـ Struct Hierarchy

```
struct aml_pinctrl  (الـ state الرئيسي للـ driver)
├── struct pinctrl_dev *pctl          ← handle من الـ framework core
├── struct aml_gpio_bank *banks[]     ← array of GPIO banks
│   └── struct aml_gpio_bank
│       ├── struct gpio_chip          ← الـ GPIO subsystem interface
│       │   ├── .get()               ← اقرأ قيمة الـ pin
│       │   ├── .set()               ← اكتب قيمة الـ pin
│       │   ├── .direction_input()   ← اتجه input
│       │   └── .direction_output()  ← اتجه output
│       ├── struct aml_pio_control pc ← reg/bit offsets لكل register type
│       │   ├── reg_offset[6]        ← رقم الـ register لكل نوع (DIR,IN,OUT,...)
│       │   └── bit_offset[6]        ← الـ bit offset الابتدائي في الـ register
│       ├── u32 bank_id              ← GPIOA=0, GPIOB=1, ...
│       ├── u32 mux_bit_offs         ← offset الـ mux bits في الـ bank
│       ├── unsigned int pin_base    ← bank_id << 8 (global pin number base)
│       ├── struct regmap *reg_mux   ← الـ regmap للـ MUX registers
│       ├── struct regmap *reg_gpio  ← الـ regmap للـ GPIO registers
│       └── struct regmap *reg_ds    ← الـ regmap للـ drive-strength registers
├── struct aml_pmx_func *functions[] ← list of mux functions
├── struct aml_pctl_group *groups[]  ← list of pin groups
└── const struct aml_pctl_data *data ← SoC-specific quirks (multi_mux)
```

ملاحظة مهمة: الـ `pin_base = bank_id << 8` معناه إن كل bank ممكن يحتوي على 256 pin كحد أقصى، والـ global pin number بيتحسب بالـ bank ID في الـ high bits. ده بيسهّل reverse lookup: عرفت الـ global pin number، قسمته على 256 تعرف الـ bank.

---

### الـ Three Vtable Interfaces

الـ framework بيتكلم مع الـ driver عبر **3 vtables مختلفة**، كل واحدة مسؤولة عن جزء:

#### 1. `pinctrl_ops` — الـ Group Management

```c
static const struct pinctrl_ops aml_pctrl_ops = {
    .get_groups_count = aml_get_groups_count,  // كام group عندك؟
    .get_group_name   = aml_get_group_name,    // ايه اسم الـ group ده؟
    .get_group_pins   = aml_get_group_pins,    // ايه الـ pins في الـ group ده؟
    .dt_node_to_map   = aml_dt_node_to_map_pinmux, // حوّل DT node لـ map entries
    .dt_free_map      = pinconf_generic_dt_free_map,
    .pin_dbg_show     = aml_pin_dbg_show,
};
```

الـ `dt_node_to_map` هي الأهم — دي اللي بتحوّل الـ DT node اللي فيه `pinmux` property لـ `pinctrl_map` entries يفهمها الـ core.

#### 2. `pinmux_ops` — الـ Function Selection

```c
static const struct pinmux_ops aml_pmx_ops = {
    .set_mux              = aml_pmx_set_mux,       // فعّل function معين على group
    .get_functions_count  = aml_pmx_get_funcs_count,
    .get_function_name    = aml_pmx_get_fname,
    .get_function_groups  = aml_pmx_get_groups,
    .gpio_request_enable  = aml_pmx_request_gpio,  // حوّل pin لـ GPIO (func=0)
};
```

لما consumer driver يعمل `pinctrl_select_state()`, الـ core بيستدعي `set_mux()` اللي بتستدعي `aml_pctl_set_function()` اللي بتعمل `regmap_update_bits()` على الـ MUX registers.

#### 3. `pinconf_ops` — الـ Pin Configuration

```c
static const struct pinconf_ops aml_pinconf_ops = {
    .pin_config_get       = aml_pinconf_get,        // اقرأ config الـ pin
    .pin_config_set       = aml_pinconf_set,        // اكتب config الـ pin
    .pin_config_group_get = aml_pinconf_group_get,  // غير مدعوم
    .pin_config_group_set = aml_pinconf_group_set,  // اضبط config لكل pins الـ group
    .is_generic           = true,                   // يستخدم generic pinconf API
};
```

الـ `is_generic = true` معناها إن الـ driver بيستخدم الـ standard `PIN_CONFIG_*` macros (زي `PIN_CONFIG_BIAS_PULL_UP`) بدل ما يعرّف parameters خاصة بيه.

---

### الـ Register Layout على الـ Hardware

لكل GPIO bank، في Amlogic A4/S7/S6، الـ registers بتتنظم كالتالي:

```
reg_gpio base address:
  offset 0x00 (reg index 0): IN      — قراءة قيمة الـ pin
  offset 0x04 (reg index 1): OUT     — كتابة قيمة الـ pin (output)
  offset 0x08 (reg index 2): DIR     — direction (1=input, 0=output)
  offset 0x0C (reg index 3): PULLEN  — تفعيل/تعطيل pull resistor
  offset 0x10 (reg index 4): PULL    — اتجاه الـ pull (1=up, 0=down)

reg_ds base address:
  offset 0x1C (reg index 7): DS      — drive strength (2 bits per pin)
                                        00=500µA, 01=2500µA, 10=3000µA, 11=4000µA

reg_mux base address:
  variable layout:            MUX     — function select (4 bits per pin)
                                        0000=GPIO, 0001..1111=alt functions
```

الـ mapping ده بيتخزن في `aml_def_regoffs[]`:
```c
static const unsigned int aml_def_regoffs[AML_NUM_REG] = {
    3, 4, 2, 1, 0, 7   // PULLEN, PULL, DIR, OUT, IN, DS
};
```

والـ bit stride لكل register:
```c
static const unsigned int aml_bit_strides[AML_NUM_REG] = {
    1, 1, 1, 1, 1, 2    // كل registers بـ 1 bit/pin ماعدا DS بـ 2 bits/pin
};
```

---

### الـ Multi-Mux Quirk: لما الـ Hardware بيخرج عن المعتاد

في بعض الـ SoCs زي S7 وS6، في pins معينة بتستخدم **mux register من bank تاني** بدل الـ bank بتاعها. مثلًا في S7:

```c
static const struct multi_mux multi_mux_s7[] = {
    {
        .m_bank_id  = AMLOGIC_GPIO_CC,  // الـ register موجود في bank CC
        .m_bit_offs = 24,               // بيبدأ من bit 24 في register الـ CC
        .sid = (AMLOGIC_GPIO_X << 8) + 16,  // من pin 16 في bank X
        .eid = (AMLOGIC_GPIO_X << 8) + 19,  // لـ pin 19 في bank X
    },
};
```

يعني pins 16-19 من GPIOX بيتتحكم فيها عبر register في GPIOCC مش GPIOX. الـ driver بيتعامل مع ده في `aml_pctl_set_function()`:

```c
/* لو الـ pin ضمن الـ range الخاص، اعمل lookup للـ bank التاني */
if (pin_id >= p_mux->sid && pin_id <= p_mux->eid) {
    /* ابحث عن الـ bank اللي فيه الـ register الفعلي */
    for (i = 0; i < info->nbanks; i++) {
        if (info->banks[i].bank_id == p_mux->m_bank_id) {
            bank = &info->banks[i];
            break;
        }
    }
    /* احسب الـ offset من أول pin الـ subordinate range */
    shift = (pin_id - p_mux->sid) << 2;
    ...
}
```

---

### الـ GPIO Subsystem Integration

الـ pinctrl framework محتاج يتكامل مع الـ **GPIO subsystem** — وده subsystem تاني في الـ kernel مسؤول عن التحكم في الـ GPIO lines.

> **الـ GPIO subsystem**: بيوفر `/sys/class/gpio` وـ `gpiod_get()` API للـ drivers. كل GPIO controller بيسجّل `gpio_chip` فيه.

كل `aml_gpio_bank` جواه `gpio_chip` كامل:

```c
static const struct gpio_chip aml_gpio_template = {
    .request          = gpiochip_generic_request,   // يطلب pin من pinctrl
    .free             = gpiochip_generic_free,       // يحرر pin
    .set_config       = gpiochip_generic_config,     // يضبط config عبر pinconf
    .set              = aml_gpio_set,                // يكتب output value
    .get              = aml_gpio_get,                // يقرأ input value
    .direction_input  = aml_gpio_direction_input,    // DIR bit = 1
    .direction_output = aml_gpio_direction_output,   // DIR bit = 0 + set value
    .get_direction    = aml_gpio_get_direction,      // يقرأ DIR bit
    .can_sleep        = true,                        // regmap ممكن يعمل sleep
};
```

لما `gpiochip_generic_request()` بتتستدعى، هي بتستدعي `pinctrl_gpio_request()` في الـ core اللي بيستدعي `gpio_request_enable` في الـ `pinmux_ops` اللي هي `aml_pmx_request_gpio()` اللي بتعمل `aml_pctl_set_function(..., 0)` — يعني بتحط الـ MUX value على 0 اللي يعني GPIO mode.

```
gpiod_get(dev, "reset", GPIOD_OUT_HIGH)
    └── gpio_chip.request()
        └── gpiochip_generic_request()
            └── pinctrl_gpio_request()
                └── pinmux_ops.gpio_request_enable()
                    └── aml_pmx_request_gpio()
                        └── aml_pctl_set_function(pin, func=0)
                            └── regmap_update_bits(reg_mux, ...)
                                └── MUX[pin] = 0x0 (GPIO mode)
```

---

### الـ Regmap Integration

> **الـ regmap**: kernel abstraction layer فوق الـ MMIO/I2C/SPI register access. بيوفر caching، byte-swapping، debugfs entries، وthread-safe access.

الملف ده بيستخدم `devm_regmap_init_mmio()` لإنشاء regmap لكل region من الـ hardware:

```c
struct regmap_config aml_regmap_config = {
    .reg_bits  = 32,   // عنوان الـ register 32-bit
    .val_bits  = 32,   // قيمة الـ register 32-bit
    .reg_stride = 4,   // كل register بـ 4 bytes
};
```

يعني كل read/write بيتعمل على الـ 32-bit registers. الـ `regmap_update_bits(map, reg, mask, val)` بتعمل:
1. قراءة الـ register الحالي
2. mask & val لتعديل الـ bits المطلوبة فقط
3. كتابة القيمة الجديدة

ده بيضمن إن تغيير bit واحد مش بيخرب باقي الـ bits في نفس الـ register.

---

### ما بيملكه الـ Framework مقابل ما بيفوّضه للـ Driver

| المسؤولية | مين بيعملها؟ |
|---|---|
| Ownership tracking (مين حاجز الـ pin) | Pinctrl **Core** |
| Conflict detection (منع اتنين على نفس الـ pin) | Pinctrl **Core** |
| DT state machine (default/idle/sleep states) | Pinctrl **Core** |
| تحويل DT properties لـ pin numbers وـ configs | الـ **Driver** (`dt_node_to_map`) |
| كتابة الـ MUX registers على الـ hardware | الـ **Driver** (`set_mux`) |
| حساب reg/bit address لكل pin | الـ **Driver** (`aml_calc_reg_and_bit`) |
| إدارة الـ pull-up/pull-down hardware | الـ **Driver** (`pinconf_ops`) |
| تسجيل الـ GPIO chips | الـ **Driver** (`gpiochip_add_data`) |
| Debugfs entries تلقائية | Pinctrl **Core** |
| الـ `sysfs` GPIO interface | GPIO **subsystem** |
| الـ regmap read/write atomicity | الـ **regmap** subsystem |

الـ driver في الملف ده مسؤول بالكامل عن: معرفة layout الـ registers، حساب أي register وأي bit يتعدّل لكل pin، وتنفيذ الـ read-modify-write. الـ core مسؤول عن كل حاجة تانية من coordination وstate management.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Enums والـ Defines — Cheatsheet

#### تعريفات الـ Register Indices (`AML_REG_*`)

| Macro | Value | الوظيفة |
|-------|-------|---------|
| `AML_REG_PULLEN` | 0 | تفعيل/تعطيل الـ pull resistor |
| `AML_REG_PULL` | 1 | اتجاه الـ pull (up أو down) |
| `AML_REG_DIR` | 2 | اتجاه الـ GPIO (input أو output) |
| `AML_REG_OUT` | 3 | قيمة الخرج (0 أو 1) |
| `AML_REG_IN` | 4 | قراءة قيمة الدخل |
| `AML_REG_DS` | 5 | Drive Strength |
| `AML_NUM_REG` | 6 | العدد الكلي لأنواع الـ registers |

#### الـ Default Register Offsets (`aml_def_regoffs[]`)

| Index | Register | Default Offset في الـ regmap |
|-------|----------|------------------------------|
| 0 (PULLEN) | Pull Enable | 3 |
| 1 (PULL) | Pull Direction | 4 |
| 2 (DIR) | Direction | 2 |
| 3 (OUT) | Output Value | 1 |
| 4 (IN) | Input Value | 0 |
| 5 (DS) | Drive Strength | 7 |

#### الـ Bit Strides لكل نوع register

| Register | Stride (bits per pin) | السبب |
|----------|-----------------------|-------|
| PULLEN | 1 | bit واحد فقط per pin |
| PULL | 1 | bit واحد فقط per pin |
| DIR | 1 | bit واحد فقط per pin |
| OUT | 1 | bit واحد فقط per pin |
| IN | 1 | bit واحد فقط per pin |
| DS | 2 | قيمتين bit لتمثيل 4 مستويات تيار |

#### الـ Enum: `aml_pinconf_drv` — Drive Strength Levels

| Value | Enum | التيار المقابل |
|-------|------|---------------|
| 0 | `PINCONF_DRV_500UA` | 500 µA |
| 1 | `PINCONF_DRV_2500UA` | 2500 µA |
| 2 | `PINCONF_DRV_3000UA` | 3000 µA |
| 3 | `PINCONF_DRV_4000UA` | 4000 µA (default عند تجاوز الحد) |

#### الـ Compatible Strings والـ Platform Data المرتبطة

| Compatible | Private Data | ملاحظة |
|------------|-------------|--------|
| `amlogic,pinctrl-a4` | NULL | لا يوجد multi_mux خاص |
| `amlogic,pinctrl-s7` | `s7_priv_data` | entry واحد multi_mux |
| `amlogic,pinctrl-s6` | `s6_priv_data` | مدخلان multi_mux |

#### Pin Config Parameters المدعومة

| Parameter | Get | Set | الوظيفة |
|-----------|-----|-----|---------|
| `PIN_CONFIG_BIAS_DISABLE` | نعم | نعم | تعطيل الـ pull |
| `PIN_CONFIG_BIAS_PULL_UP` | نعم | نعم | تفعيل pull-up |
| `PIN_CONFIG_BIAS_PULL_DOWN` | نعم | نعم | تفعيل pull-down |
| `PIN_CONFIG_DRIVE_STRENGTH_UA` | نعم | نعم | ضبط التيار بـ µA |
| `PIN_CONFIG_OUTPUT_ENABLE` | نعم | نعم | تحويل الـ pin لخرج |
| `PIN_CONFIG_LEVEL` | نعم | نعم | ضبط مستوى الخرج |

---

### الـ Structs الأساسية

#### 1. `struct aml_pio_control`

**الغرض:** يحتفظ بالـ offsets الخاصة بكل GPIO bank داخل الـ register space — كيف تحسب أين يقع الـ bit الخاص بـ pin معين لأي نوع register.

```c
struct aml_pio_control {
    u32 gpio_offset;           /* غير مستخدم فعلياً في الكود الحالي */
    u32 reg_offset[AML_NUM_REG]; /* offset الـ register داخل regmap لكل نوع */
    u32 bit_offset[AML_NUM_REG]; /* starting bit offset لأول pin في الـ bank */
};
```

| Field | الشرح |
|-------|-------|
| `reg_offset[i]` | رقم الـ register (بوحدة 32-bit word) اللي فيه أول bit لنوع register `i` |
| `bit_offset[i]` | رقم الـ bit داخل ذلك الـ register للـ pin الأول في الـ bank |

**الاستخدام:** يُستخدم في `aml_calc_reg_and_bit()` و`aml_gpio_calc_reg_and_bit()` لحساب عنوان الـ bit بالضبط:
```c
*bit = (pin - range->pin_base) * stride + bank->pc.bit_offset[reg_type];
*reg = (bank->pc.reg_offset[reg_type] + (*bit / 32)) * 4;
*bit &= 0x1f;
```

---

#### 2. `struct multi_mux`

**الغرض:** يصف حالة استثنائية حيث مجموعة من الـ pins في bank معين تستخدم **register الـ mux الخاص بـ bank آخر** (الـ main bank). هذا شائع في SoCs لأسباب تصميمية داخلية.

```c
struct multi_mux {
    unsigned int m_bank_id;  /* bank_id صاحب الـ mux register الفعلي */
    unsigned int m_bit_offs; /* bit offset داخل mux register الـ main bank */
    unsigned int sid;        /* (bank_id << 8) + start_pin_in_bank */
    unsigned int eid;        /* (bank_id << 8) + end_pin_in_bank */
};
```

**مثال حقيقي (s7):**
- الـ pins 16–19 من GPIOX تستخدم mux register الخاص بـ GPIOCC ابتداءً من bit 24.

**الترميز في `sid`/`eid`:**
```
sid = (AMLOGIC_GPIO_X << 8) + 16
    = bank_id في الـ byte العلوي + رقم الـ pin المحلي في الـ byte السفلي
```

---

#### 3. `struct aml_pctl_data`

**الغرض:** يُمثل البيانات الخاصة بكل SoC (private data) المرتبطة بالـ compatible string في device tree.

```c
struct aml_pctl_data {
    unsigned int number;          /* عدد entries في مصفوفة multi_mux */
    const struct multi_mux *p_mux; /* مؤشر لمصفوفة multi_mux الخاصة بهذا الـ SoC */
};
```

يُخزَّن في `of_device_id.data` ويُسترد عبر `of_device_get_match_data()`.

---

#### 4. `struct aml_pmx_func`

**الغرض:** يمثل **دالة pinmux** — أي وظيفة محيطية (peripheral function) ممكن تُعيَّن للـ pins (مثل UART، SPI، I2C...).

```c
struct aml_pmx_func {
    const char   *name;    /* اسم الدالة كما في device tree */
    const char  **groups;  /* أسماء pin groups التي تدعم هذه الدالة */
    unsigned int  ngroups; /* عدد الـ groups */
};
```

يُبنى من device tree: كل child node في pinctrl node (غير gpio-controller) يصبح `aml_pmx_func`.

---

#### 5. `struct aml_pctl_group`

**الغرض:** يمثل **مجموعة pins** تعمل معاً لتفعيل دالة معينة. كل group تحمل قائمة الـ pins وقيمة الـ mux لكل pin.

```c
struct aml_pctl_group {
    const char   *name;    /* اسم الـ group كما في device tree */
    unsigned int  npins;   /* عدد الـ pins في الـ group */
    unsigned int *pins;    /* مصفوفة أرقام الـ pins (global pin numbers) */
    unsigned int *func;    /* قيم الـ mux المقابلة لكل pin */
};
```

**هام:** الـ `pins` و`func` يُملآن من `pinconf_generic_parse_dt_pinmux()` اللي تقرأ خاصية `pinmux` من device tree.

---

#### 6. `struct aml_gpio_bank`

**الغرض:** الـ struct المحوري — يمثل **GPIO bank كامل** بكل ما يتعلق به من register maps وإعدادات.

```c
struct aml_gpio_bank {
    struct gpio_chip    gpio_chip;    /* embedded: واجهة GPIO للـ kernel */
    struct aml_pio_control pc;        /* offsets حساب bit/reg لكل register type */
    u32                 bank_id;      /* رقم الـ bank (AMLOGIC_GPIO_X, ...) */
    u32                 mux_bit_offs; /* offset إضافي في mux reg لهذا الـ bank */
    unsigned int        pin_base;     /* = bank_id << 8، أساس ترقيم الـ pins */
    struct regmap      *reg_mux;      /* regmap لـ registers الـ mux */
    struct regmap      *reg_gpio;     /* regmap لـ registers الـ GPIO (dir/in/out/pull) */
    struct regmap      *reg_ds;       /* regmap لـ drive strength (قد يساوي reg_gpio) */
    const struct multi_mux *p_mux;    /* إذا كان هذا bank له exception في mux، مؤشر له */
};
```

**ملاحظات مهمة:**
- **الـ `gpio_chip` embedded في أول الـ struct** → ماكرو `gpio_chip_to_bank()` يستخدم `container_of` للوصول للـ bank من أي مؤشر `gpio_chip`.
- **الـ `pin_base = bank_id << 8`** → يترك 256 pin لكل bank، مما يجعل ترقيم الـ pins globally unique.
- **الـ `reg_ds`** إذا لم يوجد في device tree يُضبط = `reg_gpio` (fallback).

---

#### 7. `struct aml_pinctrl`

**الغرض:** الـ struct الرئيسي (driver state) — يجمع كل شيء لكل pinctrl device كامل.

```c
struct aml_pinctrl {
    struct device        *dev;        /* الـ device المرتبط بالـ driver */
    struct pinctrl_dev   *pctl;       /* handle من الـ pinctrl subsystem */
    struct aml_gpio_bank *banks;      /* مصفوفة الـ GPIO banks */
    int                   nbanks;     /* عدد الـ banks */
    struct aml_pmx_func  *functions;  /* مصفوفة الـ pinmux functions */
    int                   nfunctions; /* عدد الـ functions */
    struct aml_pctl_group *groups;    /* مصفوفة الـ pin groups */
    int                   ngroups;    /* عدد الـ groups */
    const struct aml_pctl_data *data; /* بيانات SoC-specific (multi_mux) */
};
```

يُخزَّن كـ platform driver data عبر `platform_set_drvdata()` ويُسترد في كل callback عبر `pinctrl_dev_get_drvdata()`.

---

#### 8. `struct pinctrl_desc` (kernel struct، يُملأ في probe)

```c
struct pinctrl_desc {
    const char               *name;     /* اسم الـ pinctrl device */
    const struct pinctrl_pin_desc *pins; /* وصف كل pin (رقم + اسم) */
    unsigned int              npins;    /* إجمالي عدد الـ pins */
    const struct pinctrl_ops *pctlops;  /* → aml_pctrl_ops */
    const struct pinmux_ops  *pmxops;   /* → aml_pmx_ops */
    const struct pinconf_ops *confops;  /* → aml_pinconf_ops */
    struct module            *owner;
};
```

---

### مخططات العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────┐
│                    aml_pinctrl                          │
│  dev ──────────────────────────────────→ struct device  │
│  pctl ─────────────────────────────────→ pinctrl_dev    │
│  data ─────────────────────────────────→ aml_pctl_data  │
│                                              │           │
│  banks[] ──┐                           p_mux[]          │
│  nbanks    │                               │             │
│            ↓                               ↓             │
│  functions[] ──→ aml_pmx_func[]    multi_mux[]           │
│  nfunctions                                              │
│                                                         │
│  groups[] ──→ aml_pctl_group[]                          │
│  ngroups                                                │
└─────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                   aml_gpio_bank                          │
│  ┌─────────────────────────┐                             │
│  │      gpio_chip          │ ← embedded (أول field)      │
│  │  label, ngpio, fwnode   │                             │
│  │  .get / .set / .dir_*   │                             │
│  └─────────────────────────┘                             │
│                                                          │
│  pc ────────────→ aml_pio_control                        │
│                     reg_offset[6]                        │
│                     bit_offset[6]                        │
│                                                          │
│  reg_mux ───────→ regmap (mux registers)                 │
│  reg_gpio ──────→ regmap (gpio registers)                │
│  reg_ds ────────→ regmap (ds registers, قد = reg_gpio)   │
│  p_mux ─────────→ multi_mux (أو NULL)                    │
└──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│               pinctrl_gpio_range  (kernel)               │
│  pin_base, npins                                         │
│  gc ───────────→ gpio_chip (داخل aml_gpio_bank)          │
└─────────────────────────────────────────────────────────┘
         ↑
         │  pinctrl_find_gpio_range_from_pin()
         │
┌─────────────────┐
│  pinctrl_dev    │
│  (kernel owned) │
└─────────────────┘
```

**العلاقة الكاملة:**

```
platform_device
      │
      ▼
aml_pinctrl ──────────────────────────────────────────────────────┐
  │                                                               │
  ├── banks[0..nbanks-1]                                         │
  │     └── aml_gpio_bank                                        │
  │           ├── gpio_chip  ◄── container_of ── gpio_chip_to_bank
  │           ├── aml_pio_control (reg/bit offsets)              │
  │           ├── regmap *reg_mux                                │
  │           ├── regmap *reg_gpio                               │
  │           ├── regmap *reg_ds                                 │
  │           └── multi_mux *p_mux ─────────────────────────────┤
  │                                                              │
  ├── functions[0..nfunctions-1]                                 │
  │     └── aml_pmx_func                                         │
  │           ├── name (from DT)                                 │
  │           └── groups[] → names of aml_pctl_group             │
  │                                                              │
  ├── groups[0..ngroups-1]                                       │
  │     └── aml_pctl_group                                       │
  │           ├── pins[] (global pin numbers)                    │
  │           └── func[] (mux values per pin)                    │
  │                                                              │
  └── data ──→ aml_pctl_data                                     │
                 └── p_mux[] → multi_mux[] ─────────────────────┘
```

---

### مخطط الـ Lifecycle

#### مرحلة الـ Probe (الإنشاء والتسجيل)

```
platform_driver.probe (aml_pctl_probe)
│
├─ 1. devm_kzalloc: تخصيص aml_pinctrl و pinctrl_desc
│
├─ 2. aml_pctl_probe_dt()
│     ├─ aml_pctl_dt_child_count()       ← حساب nbanks, nfunctions, ngroups
│     ├─ devm_kcalloc() ×3               ← تخصيص banks[], functions[], groups[]
│     ├─ of_device_get_match_data()      ← جلب aml_pctl_data (multi_mux)
│     ├─ aml_count_pins()               ← حساب إجمالي عدد الـ pins
│     ├─ devm_kcalloc(pinctrl_pin_desc) ← تخصيص pin descriptors
│     │
│     └─ for each DT child:
│           ├─ if gpio-controller:
│           │    ├─ aml_gpiolib_register_bank()
│           │    │    ├─ aml_bank_number()    ← قراءة bank_id من gpio-ranges
│           │    │    ├─ aml_map_resource() ×3 ← mux, gpio, ds → regmap
│           │    │    ├─ copy aml_gpio_template → bank->gpio_chip
│           │    │    ├─ init_bank_register_bit() ← ضبط pc.reg_offset[] + p_mux
│           │    │    └─ bank->pin_base = bank_id << 8
│           │    └─ fill pinctrl_pin_desc entries
│           │
│           └─ if function node:
│                └─ aml_pctl_parse_functions()
│                     ├─ func->name = np->name
│                     └─ for each child group:
│                          └─ pinconf_generic_parse_dt_pinmux()
│                               → grp->pins[], grp->func[], grp->npins
│
├─ 3. ضبط pinctrl_desc (ops + name)
│
├─ 4. devm_pinctrl_register()           ← تسجيل الـ pinctrl مع الـ kernel
│      → info->pctl = pinctrl_dev
│
└─ 5. for each bank:
       └─ gpiochip_add_data(&bank->gpio_chip, bank)
            ← تسجيل الـ GPIO chip مع gpiolib
```

#### مرحلة الـ Teardown

```
devm_* functions handle cleanup automatically:
  devm_pinctrl_register  → pinctrl_unregister (auto)
  gpiochip_add_data      → gpiochip_remove (يجب explicit أو devm variant)
  devm_kcalloc           → kfree (auto)
  devm_ioremap_resource  → iounmap (auto)
  devm_regmap_init_mmio  → regmap_exit (auto)
```

---

### مخططات تدفق الاستدعاءات (Call Flow)

#### 1. ضبط الـ Pinmux (تفعيل peripheral function)

```
consumer driver calls pinctrl_select_state()
  │
  └─► pinctrl core calls aml_pmx_set_mux(pctldev, fselector, group_id)
        │
        ├─ info->groups[group_id]  ← الـ group المطلوب
        │
        └─ for each pin in group:
              │
              ├─ pinctrl_find_gpio_range_from_pin()
              │    └─► returns pinctrl_gpio_range (contains gpio_chip*)
              │
              └─ aml_pctl_set_function(info, range, pin_id, func_val)
                    │
                    ├─ [if bank has p_mux AND pin in sid..eid range]
                    │    ├─ find main bank by m_bank_id
                    │    ├─ calc shift from pin - p_mux->sid
                    │    └─► regmap_update_bits(main_bank->reg_mux, reg, mask, val)
                    │                           (4-bit field per pin)
                    │
                    └─ [normal case]
                         ├─ aml_pmx_calc_reg_and_offset()
                         │    └─ shift = ((pin - pin_base) << 2) + mux_bit_offs
                         └─► regmap_update_bits(bank->reg_mux, reg, 0xf<<off, func<<off)
```

#### 2. طلب GPIO (تحويل pin لـ GPIO mode)

```
gpio_request() أو gpiod_get()
  │
  └─► pinctrl core calls aml_pmx_request_gpio(pctldev, range, pin)
        │
        └─ aml_pctl_set_function(info, range, pin, 0)
               └─► regmap_update_bits(reg_mux, ..., 0xf<<offset, 0)
                   (function = 0 = GPIO mode)
```

#### 3. ضبط Pin Configuration

```
pinctrl_select_state() أو consumer calls pin_config_set()
  │
  └─► aml_pinconf_set(pcdev, pin, configs[], num_configs)
        │
        └─ for each config:
              ├─ PIN_CONFIG_BIAS_DISABLE
              │    └─► aml_pinconf_disable_bias()
              │          ├─ aml_calc_reg_and_bit(..., AML_REG_PULLEN, ...)
              │          └─► regmap_update_bits(reg_gpio, reg, BIT(bit), 0)
              │
              ├─ PIN_CONFIG_BIAS_PULL_UP / PULL_DOWN
              │    └─► aml_pinconf_enable_bias(info, pin, pull_up)
              │          ├─ calc PULL reg → set direction bit
              │          └─ calc PULLEN reg → set enable bit
              │
              ├─ PIN_CONFIG_DRIVE_STRENGTH_UA
              │    └─► aml_pinconf_set_drive_strength()
              │          ├─ map µA value → PINCONF_DRV_* enum
              │          └─► regmap_update_bits(reg_ds, reg, 0x3<<bit, ds_val<<bit)
              │
              ├─ PIN_CONFIG_OUTPUT_ENABLE
              │    └─► aml_pinconf_set_output()
              │          └─► regmap_update_bits(reg_gpio, ..., AML_REG_DIR, ...)
              │              (DIR=0 → output)
              │
              └─ PIN_CONFIG_LEVEL
                   └─► aml_pinconf_set_output_drive()
                         ├─ set DIR=0 (output)
                         └─► set OUT bit to high/low
```

#### 4. عمليات GPIO الأساسية

```
gpio_direction_input(gpio)
  └─► aml_gpio_direction_input(chip, gpio)
        ├─ aml_gpio_calc_reg_and_bit(bank, AML_REG_DIR, gpio, &reg, &bit)
        └─► regmap_update_bits(reg_gpio, reg, BIT(bit), BIT(bit))
            (DIR bit = 1 → input)

gpio_direction_output(gpio, value)
  └─► aml_gpio_direction_output(chip, gpio, value)
        ├─ set DIR bit = 0  (output)
        └─ set OUT bit = value

gpio_get_value(gpio)
  └─► aml_gpio_get(chip, gpio)
        ├─ aml_gpio_calc_reg_and_bit(bank, AML_REG_IN, gpio, &reg, &bit)
        └─► regmap_read(reg_gpio, reg, &val) → return !!(val & BIT(bit))

gpio_set_value(gpio, value)
  └─► aml_gpio_set(chip, gpio, value)
        └─► regmap_update_bits(reg_gpio, OUT_reg, BIT(bit), value ? BIT(bit) : 0)
```

#### 5. قراءة Pin Configuration

```
pin_config_get(pin, param)
  └─► aml_pinconf_get(pcdev, pin, &config)
        │
        ├─ PIN_CONFIG_BIAS_*
        │    └─► aml_pinconf_get_pull()
        │          ├─ read PULLEN reg → if 0: BIAS_DISABLE
        │          └─ read PULL reg   → if 1: PULL_UP else PULL_DOWN
        │
        ├─ PIN_CONFIG_DRIVE_STRENGTH_UA
        │    └─► aml_pinconf_get_drive_strength()
        │          ├─ check bank->reg_ds (or -EOPNOTSUPP)
        │          ├─ read DS reg, extract 2-bit field
        │          └─ map to µA value
        │
        └─ PIN_CONFIG_LEVEL
             ├─ aml_pinconf_get_output() → read DIR bit (must be output)
             └─ aml_pinconf_get_drive()  → read OUT bit
```

---

### مخطط حساب عنوان الـ Register والـ Bit

#### لعمليات الـ GPIO المباشرة (`aml_gpio_calc_reg_and_bit`)

```
Input: bank, reg_type (مثلاً AML_REG_DIR=2), gpio (رقم محلي داخل الـ bank)

stride = aml_bit_strides[reg_type]     // مثلاً 1 لـ DIR، 2 لـ DS

bit = gpio * stride + bank->pc.bit_offset[reg_type]
      │                      │
      │                      └── offset إضافي إذا كان الـ bank لا يبدأ من bit 0
      └── الـ pin المحلي ضروب stride

reg = (bank->pc.reg_offset[reg_type] + (bit / 32)) * 4
      │                                     │
      │                                     └── هل تجاوزنا 32 bit؟ ننتقل لـ register التالي
      └── الـ register الأساسي لهذا النوع

bit &= 0x1f  // bit position داخل الـ 32-bit register
```

**مثال:** pin رقم 5 في bank، REG_DS (stride=2), bit_offset=0, reg_offset[DS]=7
```
bit = 5 * 2 + 0 = 10
reg = (7 + 10/32) * 4 = (7 + 0) * 4 = 28
bit = 10 & 0x1f = 10
→ اقرأ 2 bits من الـ register offset 28، بدءاً من bit 10
```

#### لعمليات الـ Mux (`aml_pmx_calc_reg_and_offset`)

```
Input: pin_id (global), range->pin_base, initial offset (= bank->mux_bit_offs)

shift = ((pin_id - range->pin_base) << 2) + offset
        │                                    │
        │                                    └── offset إضافي للـ bank في الـ mux reg
        └── 4 bits per pin (mux value 0-15)

reg    = (shift / 32) * 4
offset = shift % 32
```

---

### استراتيجية الـ Locking

الـ driver **لا يستخدم locks صريحة** في الكود بنفسه — يعتمد كلياً على:

| المصدر | الـ Lock المستخدم | يحمي |
|--------|-------------------|------|
| `regmap_update_bits()` | internal regmap mutex أو spinlock | الكتابة الذرية على الـ registers |
| `pinctrl core` | `pinctrl_dev->mutex` | تغيير الـ state والـ function assignment |
| `gpiolib core` | `gpio_device->srcu` + internal locks | طلبات الـ GPIO وتغيير الاتجاه |

**ترتيب الـ Locks (Lock Ordering):**

```
pinctrl_dev->mutex
    └── (يستدعي callbacks)
         └── regmap internal lock
              └── hardware register access (MMIO, atomic)
```

**نقطة مهمة:** `aml_gpio_chip.can_sleep = true` — هذا يعني:
- عمليات الـ GPIO تستخدم **mutex** (وليس spinlock)
- مناسب لـ regmap-mmio الذي قد يكون بطيئاً
- لا يجوز استدعاء GPIO ops من **interrupt context**

**الـ regmap config:**
```c
struct regmap_config aml_regmap_config = {
    .reg_bits  = 32,  // عرض عنوان الـ register
    .val_bits  = 32,  // عرض قيمة الـ register
    .reg_stride = 4,  // المسافة بين الـ registers بالبايت
};
```
الـ stride = 4 يعني كل register address يتقدم بـ 4 bytes — متوافق مع الأجهزة 32-bit memory-mapped.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs — Cheatsheet

#### Registration & Probe

| Function | Category | Summary |
|---|---|---|
| `aml_pctl_probe` | Registration | Entry point — يبني الـ pinctrl device كامل |
| `aml_pctl_probe_dt` | Registration | يقرأ الـ device tree ويملأ الـ `aml_pinctrl` struct |
| `aml_gpiolib_register_bank` | Registration | يسجّل bank واحد (regmap + gpio_chip) |
| `aml_map_resource` | Registration | يعمل ioremap لـ named resource ويرجع regmap |
| `init_bank_register_bit` | Registration | يحسب الـ bit/reg offsets لكل bank |
| `aml_pctl_parse_functions` | Registration | يقرأ function node من الـ DT |
| `aml_pctl_dt_child_count` | Registration | يعدّ الـ banks + functions + groups في الـ DT |
| `aml_count_pins` | Registration | يعدّ إجمالي الـ pins في كل الـ GPIO banks |
| `aml_bank_pins` | Registration | يرجع عدد الـ pins في bank واحد من `gpio-ranges` |
| `aml_bank_number` | Registration | يرجع الـ bank index من `gpio-ranges` |

#### Pinmux (الـ `pinmux_ops` vtable)

| Function | Summary |
|---|---|
| `aml_pmx_set_mux` | يطبق الـ mux function على كل pins في group |
| `aml_pmx_request_gpio` | يضع الـ pin في GPIO mode (func=0) |
| `aml_pmx_get_funcs_count` | يرجع عدد الـ functions |
| `aml_pmx_get_fname` | يرجع اسم function بـ index |
| `aml_pmx_get_groups` | يرجع الـ groups المرتبطة بـ function |
| `aml_pctl_set_function` | الـ core helper — يكتب الـ mux value في الـ register |
| `aml_pmx_calc_reg_and_offset` | يحسب الـ register address والـ bit offset للـ mux |

#### Pinconf (الـ `pinconf_ops` vtable)

| Function | Summary |
|---|---|
| `aml_pinconf_get` | يقرأ config parameter لـ pin واحد |
| `aml_pinconf_set` | يكتب config parameters على pin واحد |
| `aml_pinconf_group_set` | يطبق config على كل pins في group |
| `aml_pinconf_group_get` | غير مدعوم — يرجع `-EOPNOTSUPP` |
| `aml_pinconf_get_pull` | يقرأ pull state من الـ PULLEN + PULL registers |
| `aml_pinconf_get_drive_strength` | يقرأ drive strength من الـ DS register |
| `aml_pinconf_get_output` | يتحقق إذا الـ pin configured كـ output |
| `aml_pinconf_get_drive` | يقرأ الـ output level المكتوب في OUT register |
| `aml_pinconf_get_gpio_bit` | Generic — يقرأ bit واحد من أي GPIO register |
| `aml_pinconf_disable_bias` | يعطل الـ pull resistor |
| `aml_pinconf_enable_bias` | يفعّل pull-up أو pull-down |
| `aml_pinconf_set_drive_strength` | يكتب الـ drive strength enum في DS register |
| `aml_pinconf_set_output` | يحدد اتجاه الـ pin (DIR register) |
| `aml_pinconf_set_drive` | يكتب الـ output value (OUT register) |
| `aml_pinconf_set_output_drive` | يضع الـ pin output ويحدد الـ level |
| `aml_pinconf_set_gpio_bit` | Generic — يكتب bit واحد في أي GPIO register |

#### Pinctrl Ops (الـ `pinctrl_ops` vtable)

| Function | Summary |
|---|---|
| `aml_get_groups_count` | يرجع عدد الـ groups |
| `aml_get_group_name` | يرجع اسم group بـ index |
| `aml_get_group_pins` | يرجع مصفوفة الـ pins في group |
| `aml_pin_dbg_show` | debugfs hook — يطبع اسم الـ device |
| `aml_dt_node_to_map_pinmux` | يحوّل DT node لـ pinctrl_map entries |

#### GPIO Chip Ops

| Function | Summary |
|---|---|
| `aml_gpio_get_direction` | يقرأ اتجاه الـ pin من DIR register |
| `aml_gpio_direction_input` | يضع DIR bit = 1 (input) |
| `aml_gpio_direction_output` | يضع DIR bit = 0 + يكتب الـ value في OUT |
| `aml_gpio_set` | يكتب الـ output value في OUT register |
| `aml_gpio_get` | يقرأ الـ input value من IN register |
| `aml_gpio_calc_reg_and_bit` | يحسب register + bit للـ GPIO ops |

#### Register Calculation Helpers

| Function | Summary |
|---|---|
| `aml_calc_reg_and_bit` | حساب register + bit للـ pinconf (يستخدم `aml_pio_control`) |
| `aml_gpio_calc_reg_and_bit` | نفس الهدف لكن مباشر من الـ `aml_gpio_bank` |
| `aml_pmx_calc_reg_and_offset` | حساب register + bit offset للـ mux registers |

---

### Group 1: Registration & Initialization

الـ functions دي بتبني هيكل الـ driver كامل من الـ device tree وبتسجله في الـ pinctrl و GPIO subsystems.

---

#### `aml_pctl_probe`

```c
static int aml_pctl_probe(struct platform_device *pdev)
```

ده الـ platform driver probe function — الـ entry point الوحيد. بيعمل allocate للـ `aml_pinctrl` و `pinctrl_desc`، بيستدعي `aml_pctl_probe_dt` لقراءة الـ DT، بعدين بيملأ الـ `pinctrl_desc` بالـ ops tables وبيسجله بـ `devm_pinctrl_register`. في الآخر بيعمل loop على كل الـ banks وبيسجل كل `gpio_chip` بـ `gpiochip_add_data`.

**Parameters:**
- `pdev` — الـ `platform_device` اللي الـ kernel بيمرره لأي platform driver probe

**Return value:** `0` نجاح، أو error code سلبي

**Key details:**
- كل الـ allocations هي `devm_*` — مفيش manual cleanup
- الـ `gpiochip_add_data` بيمرر `&info->banks[i]` كـ driver data — الـ GPIO callbacks بتستخدمه بـ `gpiochip_get_data`
- لو فيه bank فشل في الـ gpiochip registration، الـ probe بيرجع فورًا بدون cleanup للـ banks السابقة — معمول على افتراض الـ devm سيتولى الأمر

**Who calls it:** الـ kernel platform bus عند match مع `aml_pctl_of_match`

**Pseudocode:**
```
aml_pctl_probe(pdev):
    alloc pctl_desc, info (devm)
    info->dev = dev
    platform_set_drvdata(pdev, info)

    aml_pctl_probe_dt(pdev, pctl_desc, info)  // parse DT

    fill pctl_desc->pctlops, pmxops, confops
    devm_pinctrl_register(dev, pctl_desc, info)  // register pinctrl

    for each bank:
        gpiochip_add_data(&bank->gpio_chip, bank)
    return 0
```

---

#### `aml_pctl_probe_dt`

```c
static int aml_pctl_probe_dt(struct platform_device *pdev,
                             struct pinctrl_desc *pctl_desc,
                             struct aml_pinctrl *info)
```

بيقرأ الـ device tree node كامل ويبني كل الـ data structures. أول حاجة بيعدّ الـ banks/functions/groups بـ `aml_pctl_dt_child_count`، بعدين بيعمل allocate للـ arrays. بيمرّ على كل child node — لو فيه `gpio-controller` property يعامله كـ GPIO bank وبيستدعي `aml_gpiolib_register_bank`، غير كده يعامله كـ function node ويستدعي `aml_pctl_parse_functions`.

**Parameters:**
- `pdev` — الـ platform device
- `pctl_desc` — الـ descriptor اللي هيتملأ بالـ pins array
- `info` — الـ driver state struct

**Return value:** `0` أو error code

**Key details:**
- الـ `pin_base` لكل bank = `bank_id << 8` — الـ global pin number encoding
- بيستخدم `devm_kasprintf_strarray` لتوليد pin names تلقائيًا (مثلاً "GPIOA0", "GPIOA1", ...)
- الـ `pdesc->number` بيتساوى بـ `k` اللي بيبدأ من `pin_base` — ده الـ global pin number في الـ pinctrl subsystem
- الـ `of_device_get_match_data` بيجيب الـ `aml_pctl_data` الـ platform-specific (للـ S7 و S6 فيها multi_mux data)

---

#### `aml_gpiolib_register_bank`

```c
static int aml_gpiolib_register_bank(struct aml_pinctrl *info,
                                     int bank_nr, struct device_node *np)
```

بيسجّل bank GPIO واحد. بيجيب الـ `bank_id` من `gpio-ranges` property في الـ DT، بعدين بيعمل `ioremap` للـ registers الثلاثة (mux, gpio, ds) بـ `aml_map_resource`. بيكوبي الـ `aml_gpio_template` للـ `gpio_chip` وبيملأ الـ ngpio و fwnode و parent. بيستدعي `init_bank_register_bit` لحساب الـ offsets.

**Parameters:**
- `info` — الـ driver state
- `bank_nr` — index الـ bank في مصفوفة `info->banks`
- `np` — الـ device node الخاص بالـ bank

**Return value:** `0` أو error

**Key details:**
- الـ TEST_N و ANALOG banks لا يملكان mux register — بيتقبل `NULL` فيها
- لو مفيش `ds` register، `reg_ds` بيتساوى `reg_gpio` (الـ DS data موجود في نفس الـ GPIO register space)
- `gpio_chip.base = -1` يعني الـ kernel يختار الـ base تلقائيًا من الـ `fwnode`
- الـ `pin_base = bank_id << 8` — ده الـ encoding الأساسي لكل الـ global pin numbers

---

#### `aml_map_resource`

```c
static struct regmap *aml_map_resource(struct device *dev, unsigned int id,
                                       struct device_node *node, char *name)
```

بيعمل lookup لـ named register region في الـ DT بـ `reg-names` property، بيعمل `devm_ioremap_resource`، وبيرجع `regmap` configured بـ 32-bit registers و 4-byte stride.

**Parameters:**
- `dev` — الـ parent device (للـ devm)
- `id` — الـ bank ID (لاسم الـ regmap في debugging)
- `node` — الـ device node اللي فيه الـ `reg` و `reg-names`
- `name` — الاسم المطلوب مثل `"mux"`, `"gpio"`, `"ds"`

**Return value:** pointer لـ `regmap` أو `NULL` لو مش موجود أو `ERR_PTR` لو فيه error

**Key details:**
- الـ `regmap_config.max_register` بيتحسب من `resource_size` — بيحمي من الـ out-of-bounds access
- الـ `regmap` config: `reg_bits=32`, `val_bits=32`, `reg_stride=4` — يعني كل access هو 32-bit word aligned
- الاسم بيتبنى كـ `"GPIOA-mux"` مثلًا لسهولة الـ debugging في debugfs

---

#### `init_bank_register_bit`

```c
static void init_bank_register_bit(struct aml_pinctrl *info,
                                   struct aml_gpio_bank *bank)
```

بيهيّئ الـ `aml_pio_control` struct للـ bank. بيضع الـ default register offsets من `aml_def_regoffs[]` (IN=0, OUT=1, DIR=2, PULLEN=3, PULL=4, DS=7) والـ bit offsets كلها تبدأ من 0. بعدين بيفحص الـ `multi_mux` data — لو الـ bank هو الـ "main" bank في multi_mux يضع الـ `mux_bit_offs`، لو هو الـ "subordinate" bank يحفظ pointer للـ `p_mux` struct.

**Parameters:**
- `info` — الـ pinctrl state (للوصول لـ `data`)
- `bank` — الـ bank المراد تهيئته

**Return value:** void

**Key details:**
- الـ `aml_def_regoffs` يعكس ترتيب الـ registers الفعلي في الـ hardware:
  - Index 0 (IN) → register word 0
  - Index 1 (OUT) → register word 1
  - Index 2 (DIR) → register word 2
  - Index 3 (PULLEN) → register word 3
  - Index 4 (PULL) → register word 4
  - Index 5 (DS) → register word 7
- الـ `multi_mux` mechanism موجود لـ S7 و S6 — بعض الـ pins في bank X مثلًا بتستخدم mux register من bank CC

---

#### `aml_pctl_parse_functions`

```c
static int aml_pctl_parse_functions(struct device_node *np,
                                    struct aml_pinctrl *info, u32 index,
                                    int *grp_index)
```

بيقرأ function node من الـ DT وبيملأ `aml_pmx_func` struct. كل child node هو group — بيستدعي `pinconf_generic_parse_dt_pinmux` لكل child لقراءة الـ `pinmux` property اللي بتحتوي على pairs من (global_pin_id, mux_value).

**Parameters:**
- `np` — الـ function device node
- `info` — الـ driver state
- `index` — index الـ function في `info->functions[]`
- `grp_index` — counter مشترك للـ groups (بيتزود مع كل group)

**Return value:** `0` أو error

**Key details:**
- بيستخدم `for_each_child_of_node_scoped` — الـ reference counting للـ child node بيتعمل automatically
- `pinconf_generic_parse_dt_pinmux` بيرجع arrays من `pins[]` و `func[]` — ده الـ global pin IDs وقيم الـ mux المقابلة
- لو function مفيهاش groups (child count = 0) بيرجع `-EINVAL`

---

#### `aml_pctl_dt_child_count`

```c
static void aml_pctl_dt_child_count(struct aml_pinctrl *info,
                                    struct device_node *np)
```

pass أولي على الـ DT node بيعدّ الـ banks، functions، وإجمالي الـ groups. ده ضروري قبل الـ allocation لأن الـ `devm_kcalloc` محتاجة الـ counts.

**Parameters:**
- `info` — بيملأ `info->nbanks`, `info->nfunctions`, `info->ngroups`
- `np` — الـ pinctrl device node

**Key details:** بيفرّق بين الـ node types بوجود `gpio-controller` property

---

#### `aml_count_pins` / `aml_bank_pins` / `aml_bank_number`

```c
static unsigned int aml_count_pins(struct device_node *np)
static u32 aml_bank_pins(struct device_node *np)
static int aml_bank_number(struct device_node *np)
```

الثلاثة بيقروا الـ `gpio-ranges` property. الـ property format هو: `<&pinctrl_phandle pin_base npins>` لكن هنا الـ `args[1]` بيكون `bank_id << 8`.

- **`aml_bank_pins`** بيرجع `args[2]` = عدد الـ pins
- **`aml_bank_number`** بيرجع `args[1] >> 8` = الـ bank ID
- **`aml_count_pins`** بيجمع pins لكل الـ banks

**Key details:** الـ `of_node_put` مطلوب بعد `of_parse_phandle_with_fixed_args` لأن الـ function بيزود reference count للـ phandle node

---

### Group 2: Register Address Calculation Helpers

هدف المجموعة دي هو تحويل (pin, reg_type) لـ (register_offset, bit_position) — وده الـ core logic اللي كل الـ GPIO و pinconf operations بتعتمد عليه.

---

#### `aml_calc_reg_and_bit`

```c
static int aml_calc_reg_and_bit(struct pinctrl_gpio_range *range,
                                unsigned int pin,
                                unsigned int reg_type,
                                unsigned int *reg, unsigned int *bit)
```

بيحسب الـ register address والـ bit position لأي GPIO operation. بيستخدم `aml_pio_control` struct المخزّن في الـ `aml_gpio_bank`.

**Parameters:**
- `range` — الـ GPIO range اللي فيها الـ pin (من الـ pinctrl subsystem)
- `pin` — الـ global pin number
- `reg_type` — واحد من `AML_REG_PULLEN`, `AML_REG_PULL`, `AML_REG_DIR`, `AML_REG_OUT`, `AML_REG_IN`, `AML_REG_DS`
- `*reg` — output: الـ byte offset للـ register
- `*bit` — output: الـ bit position داخل الـ register (0-31)

**الحساب:**
```c
*bit = (pin - range->pin_base) * stride + bank->pc.bit_offset[reg_type];
*reg = (bank->pc.reg_offset[reg_type] + (*bit / 32)) * 4;
*bit &= 0x1f;  // modulo 32
```

- الـ `stride` هو 1 لكل الـ registers ماعدا DS اللي هو 2 (كل pin بيحتاج 2 bits للـ drive strength)
- الضرب في 4 لأن كل register هو 4 bytes (32-bit word addressing)

**Who calls it:** كل الـ `aml_pinconf_*` functions

---

#### `aml_gpio_calc_reg_and_bit`

```c
static inline int aml_gpio_calc_reg_and_bit(struct aml_gpio_bank *bank,
                                            unsigned int reg_type,
                                            unsigned int gpio,
                                            unsigned int *reg,
                                            unsigned int *bit)
```

نفس منطق `aml_calc_reg_and_bit` بالضبط لكن بيستقبل الـ `aml_gpio_bank` مباشرة بدل الـ `pinctrl_gpio_range`. ده بيستخدم في الـ GPIO chip callbacks اللي مش بتعرف عن الـ pinctrl range.

**Parameters:**
- `bank` — الـ GPIO bank
- `reg_type` — نوع الـ register
- `gpio` — offset الـ pin داخل الـ bank (relative، مش global)
- `*reg`, `*bit` — outputs

**Key details:** `inline` — no function call overhead في الـ hot path للـ GPIO operations

---

#### `aml_pmx_calc_reg_and_offset`

```c
static int aml_pmx_calc_reg_and_offset(struct pinctrl_gpio_range *range,
                                       unsigned int pin, unsigned int *reg,
                                       unsigned int *offset)
```

بيحسب register و bit offset للـ mux registers. الـ mux registers مختلفة عن الـ GPIO registers — كل pin بياخد 4 bits (للـ mux function value 0-15).

**الحساب:**
```c
shift = ((pin - range->pin_base) << 2) + *offset;  // x4 لأن كل pin = 4 bits
*reg   = (shift / 32) * 4;
*offset = shift % 32;
```

**Parameters:**
- `range` — GPIO range
- `pin` — global pin number
- `*reg` — input: الـ initial bit offset (الـ `mux_bit_offs`)، output: register byte offset
- `*offset` — output: bit position في الـ register

---

### Group 3: Pinmux Operations

الـ `pinmux_ops` vtable — الـ core بيستدعيها لتطبيق الـ mux settings على الـ hardware.

---

#### `aml_pctl_set_function` — Core Mux Writer

```c
static int aml_pctl_set_function(struct aml_pinctrl *info,
                                 struct pinctrl_gpio_range *range,
                                 int pin_id, int func)
```

ده أهم function في الـ mux path. بيكتب الـ mux value (4 bits) في الـ مكان الصحيح في الـ mux register.

**Logic:**
- أول ما بيفحص الـ `multi_mux` case: لو الـ pin في نطاق الـ `sid..eid` الخاص بـ subordinate bank، بيروح يبحث عن الـ main bank بـ `m_bank_id` ويستخدم register بتاعه بـ offset محسوب من `(pin_id - sid) << 2`
- في الحالة العادية، بيستخدم `aml_pmx_calc_reg_and_offset` ويكتب في `bank->reg_mux`

**Parameters:**
- `info` — الـ driver state (محتاجه للبحث عن الـ main bank في الـ multi_mux case)
- `range` — الـ GPIO range للـ pin
- `pin_id` — الـ global pin ID
- `func` — قيمة الـ mux function (0-15)، القيمة 0 = GPIO mode

**Return value:** نتيجة `regmap_update_bits` أو `-EINVAL`

**Key details:**
- الـ mask دائمًا `0xf << offset` — 4 bits لكل pin
- لو `bank->reg_mux == NULL` (TEST_N أو ANALOG) بيرجع 0 بدون error
- الـ multi_mux case: الـ S7 chip مثلاً الـ pins X16-X19 مكتوب mux-هم في الـ CC bank register بدءًا من bit 24

**Pseudocode:**
```
aml_pctl_set_function(info, range, pin_id, func):
    bank = gpio_chip_to_bank(range->gc)
    offset = bank->mux_bit_offs

    if bank->p_mux:
        if pin_id in [p_mux->sid .. p_mux->eid]:
            find main bank by p_mux->m_bank_id
            shift = (pin_id - sid) << 2
            reg = (shift/32)*4, offset = shift%32
            regmap_update_bits(main_bank->reg_mux, reg, 0xf<<offset, func<<offset)
            return

    // normal path
    aml_pmx_calc_reg_and_offset(range, pin_id, &reg, &offset)
    regmap_update_bits(bank->reg_mux, reg, 0xf<<offset, func<<offset)
```

---

#### `aml_pmx_set_mux`

```c
static int aml_pmx_set_mux(struct pinctrl_dev *pctldev, unsigned int fselector,
                           unsigned int group_id)
```

الـ callback الرئيسي لتطبيق function mux. بيمرّ على كل pins في الـ group وبيستدعي `aml_pctl_set_function` لكل pin بالـ mux value المخصصة له.

**Parameters:**
- `pctldev` — الـ pinctrl device
- `fselector` — index الـ function (مش مستخدم هنا — الـ mux values جوّا الـ group)
- `group_id` — index الـ group في `info->groups[]`

**Key details:**
- كل pin في الـ group ممكن يكون له mux value مختلفة — الـ `group->func[i]` بيحدد قيمة الـ mux لكل pin على حدة
- `pinctrl_find_gpio_range_from_pin` بتجيب الـ range بتاع كل pin (لأن كل bank عنده range منفصلة)

---

#### `aml_pmx_request_gpio`

```c
static int aml_pmx_request_gpio(struct pinctrl_dev *pctldev,
                                struct pinctrl_gpio_range *range,
                                unsigned int pin)
```

لما الـ GPIO subsystem يطلب pin عن طريق `gpio_request`، الـ pinctrl core بيستدعي الـ function دي. بتحط الـ mux value = 0 على الـ pin (GPIO mode).

**Who calls it:** الـ pinctrl core عند `pinctrl_gpio_request()` → `pinmux_request_gpio()`

---

#### `aml_pmx_get_funcs_count` / `aml_pmx_get_fname` / `aml_pmx_get_groups`

الـ three query callbacks اللي الـ pinctrl core و sysfs/debugfs بيستخدموها.

```c
static int aml_pmx_get_funcs_count(struct pinctrl_dev *pctldev)
// يرجع info->nfunctions

static const char *aml_pmx_get_fname(struct pinctrl_dev *pctldev,
                                     unsigned int selector)
// يرجع info->functions[selector].name

static int aml_pmx_get_groups(struct pinctrl_dev *pctldev,
                              unsigned int selector,
                              const char * const **grps,
                              unsigned * const ngrps)
// يملأ *grps و *ngrps من info->functions[selector]
```

كلهم read-only, no locking needed — بيقروا فقط data structures ثابتة بُنيت في probe time.

---

### Group 4: Pin Configuration Operations

---

#### `aml_pinconf_get`

```c
static int aml_pinconf_get(struct pinctrl_dev *pcdev, unsigned int pin,
                           unsigned long *config)
```

الـ dispatcher الرئيسي لقراءة pin configuration. بيفكّك الـ `config` packed value عشان يعرف الـ parameter المطلوب، بعدين بيستدعي الـ helper المناسب.

**Parameters:**
- `pcdev` — الـ pinctrl device
- `pin` — الـ global pin number
- `*config` — input: الـ parameter type encoded بـ `PIN_CONFIG_*`، output: الـ packed (param, value)

**الـ Parameters المدعومة:**

| Parameter | Action |
|---|---|
| `PIN_CONFIG_BIAS_DISABLE` | بيتحقق إن الـ pull معطّل |
| `PIN_CONFIG_BIAS_PULL_UP` | بيتحقق إن الـ pull-up مفعّل |
| `PIN_CONFIG_BIAS_PULL_DOWN` | بيتحقق إن الـ pull-down مفعّل |
| `PIN_CONFIG_DRIVE_STRENGTH_UA` | بيرجع الـ drive strength بالـ microamps |
| `PIN_CONFIG_OUTPUT_ENABLE` | بيتحقق إن الـ pin configured كـ output |
| `PIN_CONFIG_LEVEL` | بيرجع الـ output level (0 أو 1) |

**Key details:**
- الـ bias check منطقه: لو `aml_pinconf_get_pull(info, pin) == param` يرجع `arg=1`، غير كده `-EINVAL`
- الـ `PIN_CONFIG_LEVEL` بيتطلب إن الـ pin يكون output أولًا، غير كده `-EINVAL`
- في النهاية بيعمل `*config = pinconf_to_config_packed(param, arg)`

---

#### `aml_pinconf_get_pull`

```c
static int aml_pinconf_get_pull(struct aml_pinctrl *info, unsigned int pin)
```

بيقرأ state الـ pull resistor في خطوتين:
1. بيقرأ `PULLEN` register — لو الـ bit = 0 يرجع `PIN_CONFIG_BIAS_DISABLE`
2. لو مفعّل، بيقرأ `PULL` register — لو الـ bit = 1 يرجع `PIN_CONFIG_BIAS_PULL_UP`، غير كده `PIN_CONFIG_BIAS_PULL_DOWN`

**Return value:** `PIN_CONFIG_BIAS_*` enum value أو error code

**Key details:** بيستخدم `aml_calc_reg_and_bit` مرتين بـ reg types مختلفة — الـ PULLEN register هو enable/disable والـ PULL register هو الاتجاه

---

#### `aml_pinconf_get_drive_strength`

```c
static int aml_pinconf_get_drive_strength(struct aml_pinctrl *info,
                                          unsigned int pin,
                                          u16 *drive_strength_ua)
```

بيقرأ 2 bits من الـ DS register ويحوّلهم لقيمة بالـ microamps.

**الـ Mapping:**

| Bits | Value |
|------|-------|
| 00 (`PINCONF_DRV_500UA`) | 500 µA |
| 01 (`PINCONF_DRV_2500UA`) | 2500 µA |
| 10 (`PINCONF_DRV_3000UA`) | 3000 µA |
| 11 (`PINCONF_DRV_4000UA`) | 4000 µA |

**Key details:** لو `bank->reg_ds == NULL` يرجع `-EOPNOTSUPP` — لكن في الـ probe code الـ `reg_ds` بيتساوى `reg_gpio` لو مش موجود كـ separate resource، مش NULL

---

#### `aml_pinconf_get_gpio_bit`

```c
static int aml_pinconf_get_gpio_bit(struct aml_pinctrl *info,
                                    unsigned int pin,
                                    unsigned int reg_type)
```

Generic helper بيرجع قيمة bit واحد (0 أو 1) من أي GPIO register.

**Return value:** `1` لو الـ bit set، `0` لو clear، أو error code سلبي

**Who calls it:** `aml_pinconf_get_output` (بـ `AML_REG_DIR`) و `aml_pinconf_get_drive` (بـ `AML_REG_OUT`)

---

#### `aml_pinconf_get_output` / `aml_pinconf_get_drive`

```c
static int aml_pinconf_get_output(struct aml_pinctrl *info, unsigned int pin)
// يرجع 1 لو الـ pin output (DIR bit = 0 = output في Amlogic)

static int aml_pinconf_get_drive(struct aml_pinctrl *info, unsigned int pin)
// يرجع قيمة OUT register bit
```

**Key details:** الـ DIR bit logic معكوس — `DIR=1` يعني input، `DIR=0` يعني output. عشان كده `aml_pinconf_get_output` بيعمل `return !ret`.

---

#### `aml_pinconf_set`

```c
static int aml_pinconf_set(struct pinctrl_dev *pcdev, unsigned int pin,
                           unsigned long *configs, unsigned int num_configs)
```

الـ dispatcher لكتابة pin configuration. بيعمل loop على كل الـ configs وبيستدعي الـ helper المناسب لكل parameter.

**Parameters:**
- `configs` — مصفوفة من packed (param, arg) values
- `num_configs` — عدد الـ configs

**الـ Parameters المدعومة:**

| Parameter | Helper المستدعي |
|---|---|
| `PIN_CONFIG_BIAS_DISABLE` | `aml_pinconf_disable_bias` |
| `PIN_CONFIG_BIAS_PULL_UP` | `aml_pinconf_enable_bias(pin, true)` |
| `PIN_CONFIG_BIAS_PULL_DOWN` | `aml_pinconf_enable_bias(pin, false)` |
| `PIN_CONFIG_DRIVE_STRENGTH_UA` | `aml_pinconf_set_drive_strength` |
| `PIN_CONFIG_OUTPUT_ENABLE` | `aml_pinconf_set_output` |
| `PIN_CONFIG_LEVEL` | `aml_pinconf_set_output_drive` |

**Key details:** fail-fast — لو أي config فشلت بيرجع فورًا بدون تطبيق الباقي

---

#### `aml_pinconf_disable_bias`

```c
static int aml_pinconf_disable_bias(struct aml_pinctrl *info, unsigned int pin)
```

بيعطّل الـ pull resistor بتصفير bit واحد في الـ `PULLEN` register.

```c
regmap_update_bits(bank->reg_gpio, reg, BIT(bit), 0);  // clear PULLEN bit
```

---

#### `aml_pinconf_enable_bias`

```c
static int aml_pinconf_enable_bias(struct aml_pinctrl *info, unsigned int pin,
                                   bool pull_up)
```

بيفعّل الـ pull resistor في خطوتين:
1. بيكتب الاتجاه في `PULL` register (1=up, 0=down)
2. بيفعّل الـ pull بتشغيل bit في `PULLEN` register

**Key details:** الترتيب مهم — لازم تحدد الاتجاه قبل ما تفعّل الـ pull، عشان متحصلش glitch على الـ line

---

#### `aml_pinconf_set_drive_strength`

```c
static int aml_pinconf_set_drive_strength(struct aml_pinctrl *info,
                                          unsigned int pin,
                                          u16 drive_strength_ua)
```

بيحوّل قيمة الـ microamps لـ 2-bit enum ويكتبها في الـ DS register.

**Key details:**
- بيستخدم threshold comparison مش exact match — أي قيمة ≤ 500 تروح للـ 500µA bucket
- قيم أكبر من 4000 بتطلع warning وبتتحول تلقائيًا لـ 4000µA (max)
- الـ mask هو `0x3 << bit` لأن كل pin بياخد 2 bits في الـ DS register

---

#### `aml_pinconf_set_gpio_bit`

```c
static int aml_pinconf_set_gpio_bit(struct aml_pinctrl *info,
                                    unsigned int pin,
                                    unsigned int reg_type,
                                    bool arg)
```

Generic helper بيكتب bit واحد في أي GPIO register. `arg=true` → set، `arg=false` → clear.

---

#### `aml_pinconf_set_output` / `aml_pinconf_set_drive` / `aml_pinconf_set_output_drive`

```c
static int aml_pinconf_set_output(struct aml_pinctrl *info, unsigned int pin, bool out)
// يكتب !out في DIR register (DIR=0 = output)

static int aml_pinconf_set_drive(struct aml_pinctrl *info, unsigned int pin, bool high)
// يكتب high في OUT register

static int aml_pinconf_set_output_drive(struct aml_pinctrl *info, unsigned int pin, bool high)
// يضع output ثم يكتب الـ level — atomic sequence من set_output + set_drive
```

**Key details لـ `set_output`:** الـ DIR bit logic معكوس — `aml_pinconf_set_output(info, pin, true)` بيستدعي `aml_pinconf_set_gpio_bit(info, pin, AML_REG_DIR, false)` عشان يضع DIR=0

---

#### `aml_pinconf_group_set`

```c
static int aml_pinconf_group_set(struct pinctrl_dev *pcdev,
                                 unsigned int num_group,
                                 unsigned long *configs,
                                 unsigned int num_configs)
```

بيطبق نفس الـ configs على كل pins في group واحدة بالـ loop.

**Key details:** بيتجاهل الـ return value من `aml_pinconf_set` في الـ loop — ده يمكن يكون bug لأنه مش fail-fast هنا زي الـ single-pin version

---

#### `aml_pinconf_group_get`

```c
static int aml_pinconf_group_get(struct pinctrl_dev *pcdev,
                                 unsigned int group, unsigned long *config)
// يرجع -EOPNOTSUPP دائمًا
```

مش مدعوم — الـ generic pinctrl core بتتعامل مع ده.

---

### Group 5: Pinctrl Group Operations

---

#### `aml_get_groups_count` / `aml_get_group_name` / `aml_get_group_pins`

الـ three mandatory callbacks في `pinctrl_ops`.

```c
static int aml_get_groups_count(struct pinctrl_dev *pctldev)
// يرجع info->ngroups

static const char *aml_get_group_name(struct pinctrl_dev *pctldev,
                                      unsigned int selector)
// يرجع info->groups[selector].name

static int aml_get_group_pins(struct pinctrl_dev *pctldev,
                              unsigned int selector,
                              const unsigned int **pins,
                              unsigned int *npins)
// يملأ *pins و *npins من info->groups[selector]
// يرجع -EINVAL لو selector >= ngroups
```

---

#### `aml_pin_dbg_show`

```c
static void aml_pin_dbg_show(struct pinctrl_dev *pcdev, struct seq_file *s,
                             unsigned int offset)
```

بيطبع اسم الـ device في debugfs لكل pin. Optional callback — مفيد لـ `cat /sys/kernel/debug/pinctrl/*/pins`.

---

#### `aml_dt_node_to_map_pinmux`

```c
static int aml_dt_node_to_map_pinmux(struct pinctrl_dev *pctldev,
                                     struct device_node *np,
                                     struct pinctrl_map **map,
                                     unsigned int *num_maps)
```

بيحوّل DT node لـ `pinctrl_map` entries اللي الـ pinctrl core بيستخدمها لتطبيق settings. بيقرأ `pinmux` property وأي pin config properties، ثم:
1. بيحجز maps بـ `pinctrl_utils_reserve_map`
2. بيضيف mux map بـ `pinctrl_utils_add_map_mux` (group name → function name)
3. لو فيه configs، بيضيفهم بـ `pinctrl_utils_add_map_configs` كـ `PIN_MAP_TYPE_CONFIGS_GROUP`

**Parameters:**
- `np` — الـ group node (child of function node)
- `*map` — output: مصفوفة الـ maps المبنية
- `*num_maps` — output: عدد الـ maps

**Key details:**
- `np->name` هو اسم الـ group، `pnode->name` (parent) هو اسم الـ function
- الـ cleanup: لو فيه error بيستدعي `pinctrl_utils_free_map` وبيعمل `kfree(configs)`
- بيستخدم `pinconf_generic_dt_free_map` كـ `dt_free_map` callback في الـ vtable

**Pseudocode:**
```
aml_dt_node_to_map_pinmux(pctldev, np, map, num_maps):
    get pinmux property — fail if missing
    get parent node (function name)
    parse dt configs (pull, drive-strength, etc.)

    reserve_map(1 + (num_configs ? 1 : 0))
    add_map_mux(group=np->name, function=parent->name)
    if num_configs:
        add_map_configs(group=np->name, configs)

    kfree(configs)
    return 0
```

---

### Group 6: GPIO Chip Callbacks

الـ callbacks دي بتنفّذ الـ `gpio_chip` interface — الـ kernel GPIO subsystem بيستدعيها مباشرة.

---

#### `aml_gpio_get_direction`

```c
static int aml_gpio_get_direction(struct gpio_chip *chip, unsigned int gpio)
```

بيقرأ DIR bit من الـ register. لو الـ bit = 1 يرجع `GPIO_LINE_DIRECTION_IN`، لو = 0 يرجع `GPIO_LINE_DIRECTION_OUT`.

**Parameters:**
- `chip` — الـ gpio_chip struct
- `gpio` — الـ pin offset داخل الـ bank (relative index)

---

#### `aml_gpio_direction_input`

```c
static int aml_gpio_direction_input(struct gpio_chip *chip, unsigned int gpio)
```

بيضع DIR bit = 1 (input mode) بـ `regmap_update_bits`.

---

#### `aml_gpio_direction_output`

```c
static int aml_gpio_direction_output(struct gpio_chip *chip, unsigned int gpio,
                                     int value)
```

خطوتين:
1. بيضع DIR bit = 0 (output mode)
2. بيكتب الـ initial value في OUT register

**Key details:** اتجاه أول ثم قيمة — عكس ده ممكن يسبب glitch لو الـ output buffer يشتغل قبل ما يُحدد الاتجاه

---

#### `aml_gpio_set`

```c
static int aml_gpio_set(struct gpio_chip *chip, unsigned int gpio, int value)
```

بيكتب قيمة الـ output في OUT register. بيستخدم `regmap_update_bits` مع mask = `BIT(bit)`.

---

#### `aml_gpio_get`

```c
static int aml_gpio_get(struct gpio_chip *chip, unsigned int gpio)
```

بيقرأ IN register وبيرجع `!!(val & BIT(bit))` — دائمًا 0 أو 1.

**Key details:** بيقرأ من `AML_REG_IN` مش `AML_REG_OUT` — ده بيخليه يشتغل صح حتى لو الـ pin input. الـ `regmap_read` هنا بدون فحص return value — potential silent error.

---

### الـ Operations Tables (vtables)

```c
/* pinctrl_ops — group management + DT parsing */
static const struct pinctrl_ops aml_pctrl_ops = {
    .get_groups_count = aml_get_groups_count,
    .get_group_name   = aml_get_group_name,
    .get_group_pins   = aml_get_group_pins,
    .dt_node_to_map   = aml_dt_node_to_map_pinmux,
    .dt_free_map      = pinconf_generic_dt_free_map,  // generic helper
    .pin_dbg_show     = aml_pin_dbg_show,
};

/* pinmux_ops — function/mux management */
static const struct pinmux_ops aml_pmx_ops = {
    .set_mux              = aml_pmx_set_mux,
    .get_functions_count  = aml_pmx_get_funcs_count,
    .get_function_name    = aml_pmx_get_fname,
    .get_function_groups  = aml_pmx_get_groups,
    .gpio_request_enable  = aml_pmx_request_gpio,
};

/* pinconf_ops — pin electrical configuration */
static const struct pinconf_ops aml_pinconf_ops = {
    .pin_config_get       = aml_pinconf_get,
    .pin_config_set       = aml_pinconf_set,
    .pin_config_group_get = aml_pinconf_group_get,  // returns -EOPNOTSUPP
    .pin_config_group_set = aml_pinconf_group_set,
    .is_generic           = true,  // enables generic DT parsing
};
```

---

### مخطط تدفق Call Graph

```
platform_driver probe
└── aml_pctl_probe()
    ├── aml_pctl_probe_dt()
    │   ├── aml_pctl_dt_child_count()      // count banks/funcs/groups
    │   ├── aml_gpiolib_register_bank()    // per GPIO bank
    │   │   ├── aml_bank_number()          // read gpio-ranges
    │   │   ├── aml_map_resource()         // ioremap + regmap (x3)
    │   │   └── init_bank_register_bit()   // compute reg/bit offsets
    │   └── aml_pctl_parse_functions()     // per function node
    │       └── pinconf_generic_parse_dt_pinmux()
    ├── devm_pinctrl_register()            // register with pinctrl core
    └── gpiochip_add_data()               // register GPIO chips

GPIO request (user/driver)
└── pinctrl_gpio_request()
    └── aml_pmx_request_gpio()
        └── aml_pctl_set_function(pin, func=0)  // GPIO mode

pinctrl_select_state() [e.g. "default"]
└── aml_pmx_set_mux()
    └── aml_pctl_set_function()
        └── aml_pmx_calc_reg_and_offset()
        └── regmap_update_bits(reg_mux, ...)

pin_config_set()
└── aml_pinconf_set()
    ├── aml_pinconf_disable_bias()      // BIAS_DISABLE
    ├── aml_pinconf_enable_bias()       // BIAS_PULL_UP / PULL_DOWN
    ├── aml_pinconf_set_drive_strength()// DRIVE_STRENGTH_UA
    ├── aml_pinconf_set_output()        // OUTPUT_ENABLE
    └── aml_pinconf_set_output_drive()  // LEVEL
        └── [all use aml_calc_reg_and_bit() + regmap_update_bits()]
```
## Phase 5: دليل الـ Debugging الشامل

الـ driver اللي بنتكلم عنه هو `pinctrl-amlogic-a4.c` — الـ pin controller الخاص بـ Amlogic SoCs (A4, S7, S6). بيتحكم في الـ GPIO banks، الـ mux registers، الـ pull/drive-strength configuration، وبيشتغل فوق الـ `regmap` بـ MMIO.

---

### Software Level

#### 1. debugfs Entries

الـ pinctrl subsystem بيعمل entries تلقائي تحت `/sys/kernel/debug/pinctrl/`:

```
/sys/kernel/debug/pinctrl/
└── <dev-name>/          ← اسمه زي "fd000000.periphs-pinctrl"
    ├── pins             ← كل الـ pins المسجلة واسمها ورقمها
    ├── pingroups        ← كل الـ groups والـ pins اللي جوّاها
    ├── pinmux-pins      ← كل pin → function/owner حالياً
    ├── pinmux-functions ← كل function → groups
    └── pinconf-pins     ← config لكل pin (pull, drive, ...)
```

**قراءة فعلية:**

```bash
# اعرف اسم الـ device
ls /sys/kernel/debug/pinctrl/

# اقرأ حالة كل pin
cat /sys/kernel/debug/pinctrl/fd000000.periphs-pinctrl/pins

# شوف مين مسيطر على أي pin
cat /sys/kernel/debug/pinctrl/fd000000.periphs-pinctrl/pinmux-pins

# شوف config الـ pull والـ drive لكل pin
cat /sys/kernel/debug/pinctrl/fd000000.periphs-pinctrl/pinconf-pins
```

**مثال output من `pinmux-pins`:**

```
pin 600 (GPIOX_16): amlogic-pinctrl (GPIO UNCLAIMED)
pin 601 (GPIOX_17): spi0 (GPIO UNCLAIMED)
pin 602 (GPIOX_18): (MUX UNCLAIMED) (GPIO UNCLAIMED)
```

الـ `aml_pin_dbg_show` في الكود بتطبع اسم الـ device فقط — معلومة بسيطة، لكن الـ pinmux-pins هو الأهم.

---

#### 2. sysfs Entries

الـ GPIO chips المسجلة عبر `gpiochip_add_data` بتظهر في:

```
/sys/class/gpio/
├── gpiochipN/
│   ├── base        ← أول رقم GPIO في الـ chip
│   ├── ngpio       ← عدد الـ GPIOs
│   ├── label       ← "GPIOX", "GPIOA", ...
│   └── device/     ← رابط للـ platform device
```

```bash
# اعرف كل الـ GPIO banks
for d in /sys/class/gpio/gpiochip*/; do
    echo "$(cat $d/label): base=$(cat $d/base) ngpio=$(cat $d/ngpio)"
done

# export pin رقم 600 وشوف اتجاهه
echo 600 > /sys/class/gpio/export
cat /sys/class/gpio/gpio600/direction
cat /sys/class/gpio/gpio600/value
```

الـ `pin_base` في الكود = `bank_id << 8`، يعني GPIOX (bank 23) = base 0x1700 = 5888. لكن الـ GPIO number الفعلي بيحدده الـ kernel عند التسجيل.

---

#### 3. ftrace — Tracepoints والـ Events

```bash
# فعّل الـ tracing على الـ pinctrl subsystem
cd /sys/kernel/debug/tracing

# اعرف الـ events المتاحة
grep -r "pin" available_events

# فعّل events مهمة
echo 1 > events/regmap/regmap_reg_write/enable
echo 1 > events/regmap/regmap_reg_read/enable

# filter على الـ regmap الخاص بالـ pinctrl device
echo 'name == "GPIOX-mux"' > events/regmap/regmap_reg_write/filter

# ابدأ الـ tracing
echo 1 > tracing_on
# ... نفّذ العملية اللي بتـ debug عليها ...
echo 0 > tracing_on
cat trace
```

**مثال output:**

```
kworker/0:1-42    [000] .... 123.456: regmap_reg_write: GPIOX-mux reg=0 val=0x00000002
```

ده بيقول إن الـ mux register offset 0 اتكتب بيه القيمة 2 → function 2 لأول 8 pins في GPIOX.

---

#### 4. printk / Dynamic Debug

الكود بيستخدم `dev_dbg` في أماكن مهمة:

```c
dev_dbg(info->dev, "pinconf for pin %u is %lu\n", pin, *config);
dev_dbg(dev, "nbanks = %d\n", info->nbanks);
dev_dbg(dev, "nfunctions = %d\n", info->nfunctions);
dev_dbg(dev, "ngroups = %d\n", info->ngroups);
dev_dbg(dev, "Function[%d\t name:%s,\tgroups:%d]\n", ...);
dev_dbg(info->dev, "ds registers not found - skipping\n");
```

**تفعيل الـ dynamic debug:**

```bash
# فعّل كل الـ dev_dbg في الـ driver
echo "file pinctrl-amlogic-a4.c +p" > /sys/kernel/debug/dynamic_debug/control

# أو فعّل module كامل
echo "module amlogic-pinctrl +p" > /sys/kernel/debug/dynamic_debug/control

# تحقق إن التفعيل نجح
grep "pinctrl-amlogic-a4" /sys/kernel/debug/dynamic_debug/control

# شوف الـ output
dmesg -w | grep pinctrl
```

**أو عبر kernel cmdline عند الـ boot:**

```
dyndbg="file pinctrl-amlogic-a4.c +p"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_DEBUG_PINCTRL` | يفعّل الـ verbose logging في الـ pinctrl core |
| `CONFIG_PINCTRL_AMLOGIC_A4` | الـ driver نفسه — لازم يكون `=y` أو `=m` |
| `CONFIG_REGMAP_DEBUGFS` | يظهر regmap registers في debugfs |
| `CONFIG_DEBUG_GPIO` | يضيف extra checks للـ GPIO operations |
| `CONFIG_GPIOLIB_FASTPATH_LIMIT` | يتحكم في متى يستخدم الـ kernel fastpath |
| `CONFIG_DYNAMIC_DEBUG` | لازم يكون مفعّل عشان `dev_dbg` يشتغل |
| `CONFIG_KALLSYMS` | مهم لـ stack traces مفهومة |
| `CONFIG_OF` | لازم للـ DT parsing |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "PINCTRL|DEBUG_GPIO|REGMAP_DEBUGFS"
```

---

#### 6. regmap debugfs

الـ driver بيسجل كل bank كـ regmap منفصل باسم واضح (`GPIOX-mux`، `GPIOX-gpio`، `GPIOX-ds`):

```bash
ls /sys/kernel/debug/regmap/
# هتشوف entries زي:
# fd000000.periphs-pinctrl-GPIOX-mux
# fd000000.periphs-pinctrl-GPIOX-gpio
# fd000000.periphs-pinctrl-GPIOX-ds

# اقرأ كل registers للـ GPIO bank
cat /sys/kernel/debug/regmap/fd000000.periphs-pinctrl-GPIOX-gpio/registers
```

**مثال output:**

```
00: 00000000   ← reg_IN  (offset 0 = aml_def_regoffs[AML_REG_IN]=0 × 4)
04: 00000000   ← reg_OUT (offset 1 = 1×4)
08: ffffffff   ← reg_DIR (offset 2 = 2×4) — كل الـ pins input
0c: 00000000   ← reg_PULLEN (offset 3 = 3×4) — pull disabled
10: 00000000   ← reg_PULL  (offset 4 = 4×4)
1c: 00000000   ← reg_DS    (offset 7 = 7×4)
```

تفسير الـ offsets من الكود:
```c
static const unsigned int aml_def_regoffs[AML_NUM_REG] = {
    3, 4, 2, 1, 0, 7  // PULLEN, PULL, DIR, OUT, IN, DS
};
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ kernel log | السبب | الحل |
|---|---|---|
| `mux registers not found` | الـ `reg-names = "mux"` مش موجود في DT لـ bank مش TEST_N/ANALOG | تحقق من الـ DT node — أضف `reg-names` و`reg` المناسبين |
| `gpio registers not found` | الـ `reg-names = "gpio"` مش موجود في DT | لازم يكون موجود لكل bank |
| `drive-strength not supported` | الـ bank مش عنده `reg-names = "ds"` | طبيعي لبعض الـ banks — الـ driver بيـ fallback لـ reg_gpio |
| `Missing pinmux property` | الـ DT node ما فيهوش `pinmux` property | تأكد إن كل pin group عنده `pinmux` |
| `Missing function node` | الـ group node ما عندهوش parent function node | راجع هيكل الـ DT |
| `No groups defined` | الـ function node فاضي — مفيش child nodes | أضف groups كـ child nodes للـ function |
| `get num=N bank identity fail` | `gpio-ranges` مش صح أو مش موجود | تحقق من `gpio-ranges = <&pctl base 0 npins>` في الـ bank node |
| `Failed pinctrl registration` | مشكلة في تسجيل الـ pinctrl device | شوف الـ error code — غالباً تعارض في الـ pins أو مشكلة memory |
| `Failed to add gpiochip(N)!` | مشكلة في تسجيل الـ GPIO chip | غالباً تعارض في الـ GPIO base number |
| `pin N: invalid drive-strength : D , default to 4mA` | قيمة الـ drive-strength أكبر من 4000uA | استخدم قيمة ≤ 4000uA: 500, 2500, 3000, 4000 |
| `function :F, groups:G fail` | الـ `pinmux` property في الـ group node فيها مشكلة | تحقق من صيغة `AML_PINMUX(bank, offset, mode)` |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في aml_pctl_set_function — لو الـ bank اللي المفروض يعمل mux مش موجود */
if (!bank || !bank->reg_mux) {
    WARN_ON(!bank);  /* ← هنا لو الـ bank_id في multi_mux غلط */
    return -EINVAL;
}

/* في aml_gpiolib_register_bank — لو الـ bank_id خارج نطاق aml_bank_name */
WARN_ON(bank->bank_id >= ARRAY_SIZE(aml_bank_name));

/* في aml_gpio_get_direction — لو regmap_read فشل بشكل غير متوقع */
ret = regmap_read(bank->reg_gpio, reg, &val);
if (WARN_ON(ret))  /* ← يطبع stack trace + يكمّل */
    return GPIO_LINE_DIRECTION_IN;

/* في init_bank_register_bit — لو الـ multi_mux data غلطانة */
WARN_ON(p_mux->m_bank_id >= ARRAY_SIZE(aml_bank_name));
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

الـ registers بتكون في 3 مجموعات لكل bank: `mux`، `gpio`، `ds`.

**خطوات التحقق:**

```bash
# 1. اعرف العنوان الفيزيائي من DT
dtc -I fs /sys/firmware/devicetree/base | grep -A 20 "periphs-pinctrl"

# 2. اقرأ الـ register مباشرة من الـ hardware (مثلاً GPIOX gpio regs على عنوان 0xfe004000)
devmem2 0xfe004000 w   # reg_IN  (offset 0)
devmem2 0xfe004004 w   # reg_OUT (offset 1×4)
devmem2 0xfe004008 w   # reg_DIR (offset 2×4)
devmem2 0xfe00400c w   # reg_PULLEN (offset 3×4)
devmem2 0xfe004010 w   # reg_PULL (offset 4×4)
devmem2 0xfe00401c w   # reg_DS  (offset 7×4)

# 3. قارن مع ما يقوله الـ kernel
cat /sys/kernel/debug/regmap/*/registers
```

**ترتيب الـ registers من الكود:**
```
aml_def_regoffs: [IN=0, OUT=1, DIR=2, PULLEN=3, PULL=4, DS=7]
→ عناوين: base+0x00, base+0x04, base+0x08, base+0x0c, base+0x10, base+0x1c
```

---

#### 2. تقنيات الـ Register Dump

```bash
# devmem2 — لقراءة/كتابة register واحد
devmem2 <phys_addr> w         # قراءة word (32-bit)
devmem2 <phys_addr> w <val>   # كتابة

# io utility (من package: iotools)
io -4 -r <phys_addr>          # قراءة 32-bit

# /dev/mem مع python
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 0x100, offset=0xfe004000)
    for i in range(0, 0x20, 4):
        val = struct.unpack('<I', mm[i:i+4])[0]
        print(f'reg+{i:02x}: 0x{val:08x}')
    mm.close()
"

# dump كامل لنطاق الـ GPIO registers
dd if=/dev/mem bs=4 count=8 skip=$((0xfe004000/4)) 2>/dev/null | xxd
```

**مثال تفسير register DIR لـ GPIOX:**
```
reg+0x08: 0x0000FFFF
```
- الـ bits 0-15 = 1 → pins 0-15 هي inputs
- الـ bits 16-31 = 0 → pins 16-31 هي outputs

---

#### 3. نصائح Logic Analyzer / Oscilloscope

الـ driver بيستخدم `can_sleep = true` → الـ GPIO operations مش atomic → مقبول إنك تشوف latency.

**نقاط القياس المهمة:**

| السيناريو | ما تقيسه |
|---|---|
| مشكلة في الـ mux switching | قيس الـ pin قبل وبعد `pinctrl_select_state()` |
| مشكلة في الـ pull-up | قيس الـ voltage level: 3.3V = pull-up شغال، 0V = pull-down أو floating |
| مشكلة في drive strength | قيس الـ rise time للـ signal — 500uA بيعطي rise time أبطأ من 4000uA |
| multi_mux debugging | قيس pins GPIOX_16-19 وpin GPIOCC_8+ في نفس الوقت |

**للـ logic analyzer:**
- فعّل الـ trigger على أول edge على الـ pin المشكوك فيه
- قارن الـ timing مع الـ kernel trace من `ftrace`

---

#### 4. مشاكل الـ Hardware الشائعة وأنماط الـ Kernel Log

| المشكلة | Pattern في الـ dmesg | التشخيص |
|---|---|---|
| الـ pin شايل wrong function | لا يوجد error، لكن الـ device المتصل مش شغال | `cat pinmux-pins` — شوف لو func=0 (GPIO) بدل الـ func المطلوب |
| pull-up مش شغال | الـ device بيقرأ 0 رغم إن المفروض floating = 1 | `devmem2 PULLEN_reg` — تحقق إن الـ bit = 1 |
| drive-strength مش مضبوط | `pin N: invalid drive-strength` | راجع الـ DT — الـ DS register ممكن يكون على عنوان مختلف |
| الـ bank مش بيشتغل خالص | `gpio registers not found` | راجع `reg-names` في الـ DT node |
| multi_mux غلط | Pin بيعمل wrong function رغم إن الـ DT صح | راجع `multi_mux_s7/s6` — تحقق إن الـ `m_bit_offs` صح |
| regmap write فاشل | `regmap_reg_write` مش ظاهر في ftrace | تحقق من الـ clock للـ bus — ممكن الـ clock مش enabled |

---

#### 5. Device Tree Debugging

**تحقق من الـ DT المحمّل فعلاً:**

```bash
# اعرض الـ DT المحمّل بصيغة مقروءة
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | less

# ابحث عن الـ pinctrl node
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | \
    grep -A 50 "amlogic,pinctrl-a4\|amlogic,pinctrl-s7\|amlogic,pinctrl-s6"

# تحقق من gpio-ranges لبنك معين
cat /sys/firmware/devicetree/base/soc/periphs-pinctrl/GPIOX/gpio-ranges | xxd
```

**الـ DT الصح لازم يحقق:**

```dts
/* كل bank node لازم يكون فيه: */
gpio-controller;
#gpio-cells = <2>;
gpio-ranges = <&pctl (bank_id << 8) 0 npins>;
/* bank_id من AMLOGIC_GPIO_* في dt-bindings/pinctrl/amlogic,pinctrl.h */

/* الـ reg-names المطلوبة */
reg-names = "mux", "gpio";          /* الأساسي */
reg-names = "mux", "gpio", "ds";    /* لو drive-strength مدعوم */
```

**تحقق إن الـ `AML_PINMUX` macro صح:**
```c
/* AML_PINMUX(bank, offset, mode) = (((bank << 8) + offset) << 8) | mode */
/* مثلاً: GPIOX pin 5 بـ function 2: */
AML_PINMUX(AMLOGIC_GPIO_X, 5, 2)
= (((23 << 8) + 5) << 8) | 2
= ((5893) << 8) | 2
= 0x17050002
```

```bash
# تحقق من الـ pinmux values في الـ DT
cat /sys/firmware/devicetree/base/soc/periphs-pinctrl/uart0/uart0_default/pinmux | xxd
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

```bash
#!/bin/bash
# === Amlogic Pinctrl Debug Script ===

PCTL_DIR=$(ls /sys/kernel/debug/pinctrl/ | grep pinctrl | head -1)
PCTL_PATH="/sys/kernel/debug/pinctrl/$PCTL_DIR"

echo "=== Pinctrl Device: $PCTL_DIR ==="

echo -e "\n--- All Registered Pins ---"
cat "$PCTL_PATH/pins"

echo -e "\n--- Pin Mux Status ---"
cat "$PCTL_PATH/pinmux-pins" | grep -v "UNCLAIMED" | head -30

echo -e "\n--- Pin Configs (pull/drive) ---"
cat "$PCTL_PATH/pinconf-pins" | head -30

echo -e "\n--- GPIO Banks ---"
for d in /sys/class/gpio/gpiochip*/; do
    printf "%-12s base=%-5s ngpio=%s\n" \
        "$(cat $d/label)" "$(cat $d/base)" "$(cat $d/ngpio)"
done

echo -e "\n--- Regmap Registers (all banks) ---"
for d in /sys/kernel/debug/regmap/*/; do
    name=$(basename $d)
    echo "### $name ###"
    cat "$d/registers" 2>/dev/null | head -10
done
```

---

#### تفعيل الـ Dynamic Debug وقراءة النتائج

```bash
# فعّل
echo "file pinctrl-amlogic-a4.c +pflm" > /sys/kernel/debug/dynamic_debug/control

# تحقق
grep "pinctrl-amlogic-a4" /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages أثناء bind/probe
dmesg --follow | grep -E "pinctrl|amlogic|GPIO"

# إيقاف
echo "file pinctrl-amlogic-a4.c -p" > /sys/kernel/debug/dynamic_debug/control
```

**مثال output بعد التفعيل:**

```
[    2.345] amlogic-pinctrl fd000000.periphs-pinctrl: nbanks = 6
[    2.346] amlogic-pinctrl fd000000.periphs-pinctrl: nfunctions = 45
[    2.347] amlogic-pinctrl fd000000.periphs-pinctrl: ngroups = 180
[    2.348] amlogic-pinctrl fd000000.periphs-pinctrl: Function[0  name:uart0, groups:2]
[    2.350] amlogic-pinctrl fd000000.periphs-pinctrl: ds registers not found - skipping
```

---

#### فحص مشكلة pin معين (مثلاً GPIOX_5)

```bash
PIN=5
BANK="GPIOX"
BANK_ID=23  # AMLOGIC_GPIO_X = 23

# 1. اعرف الـ global pin number
grep "GPIOX_${PIN}" /sys/kernel/debug/pinctrl/*/pins

# 2. شوف الـ function الحالي
grep "pin.*GPIOX_${PIN}" /sys/kernel/debug/pinctrl/*/pinmux-pins

# 3. اقرأ الـ config
grep "pin.*GPIOX_${PIN}" /sys/kernel/debug/pinctrl/*/pinconf-pins

# 4. export كـ GPIO وقيس
GLOBAL_PIN=$(grep "GPIOX_${PIN})" /sys/kernel/debug/pinctrl/*/pins | grep -o 'pin [0-9]*' | awk '{print $2}')
echo $GLOBAL_PIN > /sys/class/gpio/export
echo "Direction: $(cat /sys/class/gpio/gpio${GLOBAL_PIN}/direction)"
echo "Value:     $(cat /sys/class/gpio/gpio${GLOBAL_PIN}/value)"
echo $GLOBAL_PIN > /sys/class/gpio/unexport
```

---

#### فحص الـ multi_mux (S7/S6 specific)

على الـ S7، الـ pins GPIOX_16 → GPIOX_19 بيستخدموا الـ mux register من GPIOCC مع bit offset 24:

```bash
# تحقق إن الـ GPIOCC mux register متسجل
ls /sys/kernel/debug/regmap/ | grep CC

# اقرأ الـ register اللي بيتحكم في GPIOX_16-19
# offset في GPIOCC-mux: bit 24-31
cat /sys/kernel/debug/regmap/*GPIOCC-mux*/registers

# تحقق من الـ ftrace لما تعمل pinctrl_select_state
echo "file pinctrl-amlogic-a4.c +p" > /sys/kernel/debug/dynamic_debug/control
echo 1 > /sys/kernel/debug/tracing/events/regmap/regmap_reg_write/enable
# ... نفّذ العملية ...
grep "GPIOCC-mux" /sys/kernel/debug/tracing/trace
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Android TV Box بـ Amlogic S7 — UART مش شغال بعد Bring-Up

#### العنوان
الـ UART للـ debug console مش بيظهر على Android TV box بعد تحميل الـ pinctrl driver

#### السياق
شركة بتعمل Android TV box باستخدام Amlogic S7 SoC. المهندس عامل bring-up للـ kernel وبيلاقي إن الـ serial console مش بيطلع أي output بعد ما الـ pinctrl driver يتحمل، رغم إن الـ UART hardware موجود وصح على الـ schematic.

#### المشكلة
الـ UART TX/RX pins مش بتتـ mux صح. الـ device tree بيحدد `pinmux` property صح، بس الـ data اللي بتتكتب في الـ mux register غلط، يعني الـ pin فاضل على GPIO function (0) مش على UART function.

#### التحليل
الـ S7 عنده `multi_mux` خاصة — الـ GPIOX[16:19] بيستخدموا mux registers من الـ GPIOCC bank بدل الـ GPIOX bank نفسه:

```c
static const struct multi_mux multi_mux_s7[] = {
    {
        .m_bank_id = AMLOGIC_GPIO_CC,   // الـ mux register جاي من CC bank
        .m_bit_offs = 24,               // بيبدأ من bit 24
        .sid = (AMLOGIC_GPIO_X << 8) + 16, // pin X16
        .eid = (AMLOGIC_GPIO_X << 8) + 19, // pin X19
    },
};
```

لما `aml_pctl_set_function` بتتشغل على pin X16-X19:

```c
if (bank->p_mux) {
    p_mux = bank->p_mux;
    if (pin_id >= p_mux->sid && pin_id <= p_mux->eid) {
        // بندور على CC bank
        bank = NULL;
        for (i = 0; i < info->nbanks; i++) {
            if (info->banks[i].bank_id == p_mux->m_bank_id) {
                bank = &info->banks[i];
                break;
            }
        }
        // بنكتب في CC bank's mux register مش X bank
        shift = (pin_id - p_mux->sid) << 2;
        ...
        return regmap_update_bits(bank->reg_mux, reg, ...);
    }
}
```

لو الـ GPIOCC bank اتحمل قبل GPIOX في الـ device tree، الـ `info->banks` array ممكن يكون الـ CC bank مش موجود فيه وقت الـ lookup، فالـ `bank` بيبقى `NULL` وبيرجع `-EINVAL` بصمت.

#### الحل
**1. تأكد من ترتيب الـ banks في DT:**

```dts
/* pinctrl node */
&pinctrl_aml {
    /* GPIOCC لازم يجي قبل GPIOX في الـ DT */
    gpiocc: bank@gpiocc {
        gpio-controller;
        gpio-ranges = <&pinctrl_aml (AMLOGIC_GPIO_CC << 8) 0 8>;
        reg-names = "mux", "gpio", "ds";
        reg = <0xFF700100 0x40>, <0xFF700200 0x40>, <0xFF700300 0x40>;
    };

    gpiox: bank@gpiox {
        gpio-controller;
        gpio-ranges = <&pinctrl_aml (AMLOGIC_GPIO_X << 8) 0 20>;
        ...
    };

    uart_a_pins: uart-a {
        uart-a {   /* function node */
            uart_a_group: uart-a-group {
                pinmux = <AML_PINMUX(AMLOGIC_GPIO_X, 16, 1)
                          AML_PINMUX(AMLOGIC_GPIO_X, 17, 1)>;
            };
        };
    };
};
```

**2. Debug بـ regmap debugfs:**

```bash
# شوف الـ CC bank mux register
cat /sys/kernel/debug/regmap/GPIOCC-mux/registers | grep -A2 "0c:"
# لو bits [27:24] هي 0 → الـ mux مش اتكتب
```

#### الدرس المستفاد
الـ `multi_mux` mechanism في S7/S6 بتكسر الافتراض الطبيعي "كل bank بيتحكم في mux registers بتاعته". أي pin في GPIOX[16:19] على S7 بيكتب mux value في GPIOCC bank. لازم تتأكد إن الـ CC bank موجود في الـ `info->banks` قبل ما تيجي تعمل mux على X bank pins دول.

---

### السيناريو 2: Industrial Gateway بـ Amlogic A4 — Drive Strength مش بتتغير

#### العنوان
الـ SPI lines بتفشل عند 50MHz على industrial gateway وتشتغل تمام عند 10MHz

#### السياق
منتج industrial gateway بيستخدم Amlogic A4 SoC. الـ SPI master بيتكلم مع flash chip خارجي. عند رفع الـ clock لـ 50MHz، الـ signal integrity بتتدهور والـ transactions بتفشل. الـ schematic مصمم صح، والـ trace lengths مقبولة، بس المهندس محتاج يرفع الـ drive strength للـ SPI pins.

#### المشكلة
المهندس بيضيف `drive-strength-microamp = <4000>` في الـ DT بس مش بيشوف فرق. بيعمل read-back ولاقى الـ drive strength فاضلة على 500µA.

#### التحليل
الـ `aml_pinconf_set_drive_strength` بتحسب الـ `reg` و `bit` بـ `AML_REG_DS`:

```c
static int aml_pinconf_set_drive_strength(struct aml_pinctrl *info,
                                          unsigned int pin,
                                          u16 drive_strength_ua)
{
    ...
    if (!bank->reg_ds) {
        dev_err(info->dev, "drive-strength not supported\n");
        return -EOPNOTSUPP;
    }

    aml_calc_reg_and_bit(range, pin, AML_REG_DS, &reg, &bit);
    // AML_REG_DS له stride = 2 (2 bits per pin)
    // reg_offset[5] = 7 (default من aml_def_regoffs)
    ...
    return regmap_update_bits(bank->reg_ds, reg, 0x3 << bit, ds_val << bit);
}
```

لو `bank->reg_ds` اتعينله نفس قيمة `bank->reg_gpio` (الـ fallback في `aml_gpiolib_register_bank` لما مفيش "ds" في `reg-names`):

```c
bank->reg_ds = aml_map_resource(dev, bank->bank_id, np, "ds");
if (IS_ERR_OR_NULL(bank->reg_ds)) {
    dev_dbg(info->dev, "ds registers not found - skipping\n");
    bank->reg_ds = bank->reg_gpio;  // fallback غلط!
}
```

لو الـ A4 SoC عنده DS register في address منفصل وإنت ناسي تضيف `"ds"` في `reg-names` بالـ DT، الـ driver بيكتب في offset غلط داخل الـ gpio register space — بيبوظ direction/output bits بدل DS bits.

#### الحل
**تصحيح الـ DT للـ bank المعني:**

```dts
gpiox: bank@gpiox {
    gpio-controller;
    gpio-ranges = <&pinctrl_aml (AMLOGIC_GPIO_X << 8) 0 20>;
    /* إضافة الـ ds register الصح */
    reg-names = "mux", "gpio", "ds";
    reg = <0xFF634000 0x40>,   /* mux */
          <0xFF634100 0x40>,   /* gpio */
          <0xFF634200 0x40>;   /* ds — كان ناقص! */
};
```

**تحقق بعد التصحيح:**

```bash
# اقرأ DS register قبل وبعد التغيير
devmem2 0xFF634238 w   # bit [1:0] للـ pin المعين

# أو بـ pinctrl debugfs
cat /sys/kernel/debug/pinctrl/*/pinconf-pins | grep GPIOX16
```

**تحقق من الـ drive strength class:**

```c
// aml_pinconf_get_drive_strength
switch ((val >> bit) & 0x3) {
    case PINCONF_DRV_500UA:  *drive_strength_ua = 500;  break;
    case PINCONF_DRV_4000UA: *drive_strength_ua = 4000; break;
    ...
}
```

#### الدرس المستفاد
لما `reg-names = "ds"` ناقص من الـ DT، الـ driver بيعمل fallback لـ `reg_gpio` بصمت (`dev_dbg` مش `dev_err`). الـ writes اللي المفروض تروح لـ DS register بتروح لـ offset غاط جوه gpio space. دايماً confirm إن كل الـ `reg-names` موجودة، وفعّل `CONFIG_DYNAMIC_DEBUG` عشان تشوف الـ `dev_dbg` messages.

---

### السيناريو 3: Custom Board Bring-Up بـ Amlogic S6 — I2C مش بيشتغل على GPIOD[6]

#### العنوان
الـ I2C على GPIOD[6] رافض يتـ mux بعد أي تغيير في الـ DT على custom board بـ S6

#### السياق
مهندس embedded بيعمل custom board bring-up باستخدام Amlogic S6. الـ I2C2 controller المفروض يتوصل بـ sensor على GPIOD[6] (SDA). الـ I2C master بيطلع timeout في كل transaction ومفيش ACK.

#### المشكلة
بعد تفتيش الـ oscilloscope، الـ SDA line مش بتتحرك — الـ pin فاضل configured كـ GPIO input مش I2C.

#### التحليل
الـ S6 SoC عنده حالة `multi_mux` خاصة لـ GPIOD[6]:

```c
static const struct multi_mux multi_mux_s6[] = {
    {
        /* GPIOX[16:19] → CC bank (زي S7) */
        .m_bank_id = AMLOGIC_GPIO_CC,
        .m_bit_offs = 24,
        .sid = (AMLOGIC_GPIO_X << 8) + 16,
        .eid = (AMLOGIC_GPIO_X << 8) + 19,
    }, {
        /* GPIOD[6] → GPIOF bank بدل GPIOD! */
        .m_bank_id = AMLOGIC_GPIO_F,
        .m_bit_offs = 4,
        .sid = (AMLOGIC_GPIO_D << 8) + 6,
        .eid = (AMLOGIC_GPIO_D << 8) + 6,
    },
};
```

يعني GPIOD[6] mux register موجود في GPIOF bank، بـ bit offset = 4. لما `aml_pctl_set_function` بتشتغل:

```c
if (pin_id >= p_mux->sid && pin_id <= p_mux->eid) {
    // pin_id = (AMLOGIC_GPIO_D << 8) + 6 = 0x0306
    // بندور على GPIOF bank
    for (i = 0; i < info->nbanks; i++) {
        if (info->banks[i].bank_id == p_mux->m_bank_id) { // AMLOGIC_GPIO_F
            bank = &info->banks[i];
            break;
        }
    }
    // بنكتب في GPIOF mux reg من bit 4
    shift = (pin_id - p_mux->sid) << 2;  // = 0 لأن D6 هو أول pin
    reg = 0;
    offset = 4 + 0 = 4;   // m_bit_offs = 4
    return regmap_update_bits(bank->reg_mux, 0, 0xf << 4, func << 4);
}
```

لو المهندس اعتقد إن الـ mux بيكتب في GPIOD registers وراح يقرأ منها، هيشوف صفر دايماً ويفتكر الـ driver عنده bug.

#### الحل
**تأكد إن GPIOF bank موجود في الـ DT ومتحمل قبل D:**

```dts
/* في الـ pinctrl node لـ S6 */
gpiof: bank@ff800100 {
    gpio-controller;
    gpio-ranges = <&pinctrl_aml (AMLOGIC_GPIO_F << 8) 0 14>;
    reg-names = "mux", "gpio", "ds";
    reg = <0xff800100 0x40>, <0xff800200 0x40>, <0xff800300 0x40>;
};

gpiod: bank@ff800080 {
    gpio-controller;
    gpio-ranges = <&pinctrl_aml (AMLOGIC_GPIO_D << 8) 0 12>;
    reg-names = "mux", "gpio", "ds";
    reg = <0xff800080 0x40>, <0xff800180 0x40>, <0xff800280 0x40>;
};

i2c2_pins: i2c2 {
    i2c2 {
        i2c2_group: i2c2-group {
            pinmux = <AML_PINMUX(AMLOGIC_GPIO_D, 6, 2)   /* SDA */
                      AML_PINMUX(AMLOGIC_GPIO_D, 7, 2)>; /* SCL */
        };
    };
};
```

**Debug: اقرأ GPIOF mux register مباشرة:**

```bash
# الـ bit [7:4] في GPIOF mux reg[0] المفروض = I2C function value
devmem2 0xff800100 w
# لو قيمة bits [7:4] = 0 → مش اتكتب
# لو قيمة bits [7:4] = 2 → I2C function صح
```

#### الدرس المستفاد
على S6، الـ `multi_mux_s6` بيعرّف حالتين مختلفتين من الـ cross-bank mux. المهندس لازم يراجع `multi_mux_s6[]` في الـ driver عشان يعرف أي bank بيتحكم فعلاً في الـ mux لكل pin غريب. مينفعش تفترض "D bank = D mux registers".

---

### السيناريو 4: IoT Sensor Node بـ Amlogic A4 — GPIO Interrupt مش بيشتغل بعد Pull-Up

#### العنوان
الـ interrupt من PIR sensor على GPIOC[3] مش بيوقّع رغم تفعيل الـ pull-up في الـ DT

#### السياق
IoT sensor node بيستخدم Amlogic A4. الـ PIR motion sensor بيوصل خرجه لـ GPIOC[3] active-low. الـ DT بيطلب `bias-pull-up` لكن الـ GPIO فاضل floating وبيعمل false triggers.

#### المشكلة
الـ pull-up مش بتتفعل رغم إنها موجودة في الـ DT.

#### التحليل
لما الـ framework بيطلب `PIN_CONFIG_BIAS_PULL_UP`، الـ driver بيدخل `aml_pinconf_enable_bias`:

```c
static int aml_pinconf_enable_bias(struct aml_pinctrl *info, unsigned int pin,
                                   bool pull_up)
{
    ...
    aml_calc_reg_and_bit(range, pin, AML_REG_PULL, &reg, &bit);
    if (pull_up)
        val = BIT(bit);

    /* الخطوة 1: ضبط اتجاه الـ pull (up أو down) */
    ret = regmap_update_bits(bank->reg_gpio, reg, BIT(bit), val);
    if (ret)
        return ret;

    aml_calc_reg_and_bit(range, pin, AML_REG_PULLEN, &reg, &bit);
    /* الخطوة 2: تفعيل الـ pull */
    return regmap_update_bits(bank->reg_gpio, reg, BIT(bit), BIT(bit));
}
```

`aml_calc_reg_and_bit` بتحسب الـ reg و bit:

```c
static int aml_calc_reg_and_bit(..., unsigned int reg_type, ...)
{
    // AML_REG_PULLEN = 0, stride = 1
    // AML_REG_PULL   = 1, stride = 1
    // reg_offset[0] = 3 (PULLEN default)
    // reg_offset[1] = 4 (PULL default)
    *bit = (pin - range->pin_base) * aml_bit_strides[reg_type]
           + bank->pc.bit_offset[reg_type];
    *reg = (bank->pc.reg_offset[reg_type] + (*bit / 32)) * 4;
    *bit &= 0x1f;
}
```

للـ GPIOC bank، لو `pin_base` غلط في الـ DT (الـ `gpio-ranges` second arg غلط)، الـ `pin - range->pin_base` بيطلع قيمة غلط، وبالتالي الـ bit بيتحسب في register تاني.

المشكلة الشائعة: `gpio-ranges = <&pctl (AMLOGIC_GPIO_C << 8) 0 8>` الـ pin_base = `0x0200` = 512 (الـ bank_id << 8). لو المهندس كتب بالغلط `gpio-ranges = <&pctl 0 0 8>` (بدون الـ bank_id)، الـ `pin - range->pin_base` بيبقى كبير جداً وبيخبط في register غلط.

#### الحل
**تصحيح الـ gpio-ranges:**

```dts
gpioc: bank@gpioc {
    gpio-controller;
    #gpio-cells = <2>;
    /* لازم (AMLOGIC_GPIO_C << 8) = (2 << 8) = 0x200 = 512 */
    gpio-ranges = <&pinctrl_aml 512 0 8>;
    /* أو بالـ macro */
    /* gpio-ranges = <&pinctrl_aml (AMLOGIC_GPIO_C << 8) 0 8>; */
    reg-names = "mux", "gpio", "ds";
    reg = <...>;
};
```

**تحقق من pull-up activation:**

```bash
# اقرأ PULLEN register (reg_offset[0]=3, بـ word 4-bytes = offset 0x0C)
devmem2 0xGPIOC_BASE+0x0C w
# الـ bit 3 (pin 3) لازم = 1

# اقرأ PULL direction register (reg_offset[1]=4, offset 0x10)
devmem2 0xGPIOC_BASE+0x10 w
# الـ bit 3 لازم = 1 (pull-up)

# أو بـ kernel sysfs
cat /sys/kernel/debug/pinctrl/*/pinconf-pins | grep "GPIOC3"
```

#### الدرس المستفاد
`aml_bank_number` بتقرأ الـ `gpio-ranges` وبتستخرج `args[1] >> 8` كـ bank_id. كل حسابات الـ bit/reg بتعتمد على `pin - range->pin_base` حيث `pin_base = bank_id << 8`. خطأ في `gpio-ranges` في الـ DT بيأثر على كل عمليات الـ GPIO والـ pull بشكل صامت.

---

### السيناريو 5: Automotive ECU بـ Amlogic S7 — HDMI HPD Pin عامل كـ GPIO بالغلط

#### العنوان
الـ HDMI Hot-Plug Detection مش شغال على infotainment ECU — الـ display مش بيتعرف تلقائياً

#### السياق
automotive infotainment ECU مبني على Amlogic S7. الـ HDMI HPD signal المفروض يتراقب بالـ kernel لإعلام الـ display subsystem. المهندس عمل رفع الـ pin كـ GPIO input للـ HPD لكن الـ HDMI driver مش بيتلقى الـ connect event.

#### المشكلة
الـ pin موضوع على GPIO function بدل HDMI HPD function — الـ `aml_pmx_request_gpio` بيعمل mux value = 0 (GPIO) على الـ pin.

#### التحليل
لما الـ HDMI driver بيطلب الـ GPIO للـ HPD:

```c
/* HDMI driver يستدعي gpiod_get → gpiochip_generic_request → pinmux_request_gpio */
static int aml_pmx_request_gpio(struct pinctrl_dev *pctldev,
                                struct pinctrl_gpio_range *range,
                                unsigned int pin)
{
    struct aml_pinctrl *info = pinctrl_dev_get_drvdata(pctldev);

    return aml_pctl_set_function(info, range, pin, 0);  // func=0 = GPIO mode!
}
```

هنا الـ `func=0` بيعمل override لأي mux value سابق وبيرجع الـ pin لـ GPIO mode. المشكلة إن الـ HDMI HPD على Amlogic S7 له dedicated hardware path يمر بـ mux function معينة (مش GPIO).

الحل الصح إن الـ HDMI driver يستخدم `pinctrl_select_state` مع named pinctrl state (`"default"`) اللي بتحدد الـ HDMI HPD function، مش `gpiod_get`.

لو المهندس حدد في الـ DT:

```dts
/* غلط: بيجبر الـ pin يبقى GPIO */
hdmi_hpd_gpio: hdmi-hpd-gpio {
    gpio-hog;
    gpios = <&gpiox 20 GPIO_ACTIVE_HIGH>;
    input;
};
```

بدل:

```dts
/* صح: بيحدد HDMI HPD function */
hdmi_hpd_pins: hdmi-hpd {
    hdmi_hpd {
        hdmi_hpd_grp: hdmi-hpd-grp {
            pinmux = <AML_PINMUX(AMLOGIC_GPIO_X, 20, 3)>;  /* func 3 = HDMI HPD */
        };
    };
};
```

الفرق: `gpio-hog` بيستدعي `gpiochip_generic_request` اللي بيستدعي `aml_pmx_request_gpio` اللي بيكتب `func=0`. بينما `pinctrl_select_state` بيستدعي `aml_pmx_set_mux` اللي بيكتب الـ function value الصح.

#### الحل
**تعديل الـ DT للـ infotainment:**

```dts
/* في pinctrl node */
hdmi_hpd_pins: hdmi-hpd-pins {
    hdmi-hpd-fn {
        hdmi_hpd_grp {
            /* GPIOX20 على func=3 = HDMI HPD على S7 */
            pinmux = <AML_PINMUX(AMLOGIC_GPIO_X, 20, 3)>;
            bias-pull-down;  /* HPD active-high, pull-down as default */
        };
    };
};

/* في HDMI device node */
&hdmi_tx {
    pinctrl-names = "default";
    pinctrl-0 = <&hdmi_hpd_pins>;
    status = "okay";
};
```

**تحقق من الـ mux value:**

```bash
# اقرأ الـ mux register لـ GPIOX20 على S7
# GPIOX20 = pin offset 20, كل pin 4 bits
# reg = (20 * 4) / 32 * 4 = offset 0x0A → word 2
# bit = (20 * 4) % 32 = 16

cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep "GPIOX20"
# المفروض: "function hdmi_hpd group hdmi_hpd_grp"
# لو: "GPIO GPIOX" → الـ mux غلط
```

**تتبع `aml_pmx_set_mux`:**

```c
static int aml_pmx_set_mux(struct pinctrl_dev *pctldev,
                            unsigned int fselector, unsigned int group_id)
{
    struct aml_pctl_group *group = &info->groups[group_id];
    for (i = 0; i < group->npins; i++) {
        range = pinctrl_find_gpio_range_from_pin(pctldev, group->pins[i]);
        /* group->func[i] هنا = 3 (HDMI HPD) مش 0 */
        aml_pctl_set_function(info, range, group->pins[i], group->func[i]);
    }
}
```

الـ `group->func[i]` بيجي من الـ DT عبر `pinconf_generic_parse_dt_pinmux` اللي بتحلل `AML_PINMUX(bank, offset, mode)` وتستخرج الـ `mode` (bits [7:0]).

#### الدرس المستفاد
`aml_pmx_request_gpio` دايماً بيكتب `func=0` — ده by design عشان يحرر الـ pin لـ GPIO use. أي peripheral بيحتاج dedicated mux function (HDMI, UART, I2C) لازم يستخدم `pinctrl_select_state` مع named state في الـ DT، مش `gpiod_get`/`gpio-hog`. الـ `gpio-hog` مفيد بس للـ pins اللي فعلاً محتاجة GPIO mode من البداية.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

الـ LWN.net هو المرجع الأول لمتابعة تطور الـ pinctrl subsystem في الـ Linux kernel.

| المقال | الوصف |
|--------|-------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | مقال أساسي يشرح فكرة الـ pinctrl subsystem وسبب إنشاؤه كبديل لكود الـ board-specific القديم |
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | أول patch series أضاف الـ pinctrl subsystem للـ kernel (v3.1) |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | النسخة السابعة من الـ patch series الأصلي — يوضح مراحل تطور الـ API |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | الـ documentation الأولى المرفقة مع الـ subsystem |
| [Pinctrl driver for Amlogic Meson SoCs](https://lwn.net/Articles/616225/) | أول driver لـ Amlogic Meson — الأساس الذي بُني عليه `pinctrl-amlogic-a4.c` |
| [Amlogic Meson pinctrl driver](https://lwn.net/Articles/620822/) | مناقشة الـ patch series الأولى بالتفصيل |
| [Amlogic Meson8b pinctrl driver support](https://lwn.net/Articles/638594/) | توسيع الـ driver لدعم Meson8b |
| [pinctrl: meson-a1: add pinctrl driver](https://lwn.net/Articles/804174/) | إضافة دعم A1 — أقرب سلف مباشر لمعمارية A4 |
| [Add pinctrl driver support for Amlogic T7 SoCs](https://lwn.net/Articles/945285/) | دعم T7 — يُظهر تطور نمط الـ multi-bank design |
| [Add support for Amlogic S7/S7D/S6 pinctrl](https://lwn.net/Articles/1022709/) | الـ patch series الأحدث الذي أضاف S7/S7D/S6 على قمة `pinctrl-amlogic-a4.c` |

---

### الـ Kernel Documentation الرسمية

```
Documentation/driver-api/pin-control.rst    ← الوثيقة الرئيسية للـ subsystem
Documentation/devicetree/bindings/pinctrl/  ← binding specs لكل vendor
Documentation/driver-api/gpio/             ← GPIO subsystem وعلاقته بالـ pinctrl
```

الـ online versions:

- [PINCTRL subsystem — kernel.org](https://docs.kernel.org/driver-api/pin-control.html)
- [GPIO Driver Interface](https://static.lwn.net/kerneldoc/driver-api/gpio/driver.html)
- [Subsystem drivers using GPIO](https://static.lwn.net/kerneldoc/driver-api/gpio/drivers-on-gpio.html)

الـ DT binding الخاص بالـ driver:
```
Documentation/devicetree/bindings/pinctrl/amlogic,pinctrl.yaml
include/dt-bindings/pinctrl/amlogic,pinctrl.h
```

---

### Kernel Commits المهمة

الـ commits التالية تتبع تطور الـ driver مباشرة:

| الحدث | المرجع |
|-------|-------|
| الـ patch series الأصلي لـ A4 driver (v4) | [lore.kernel.org — PATCH v4 3/5](https://lore.kernel.org/lkml/20250122-amlogic-pinctrl-v4-3-4677b2e18ff1@amlogic.com/) |
| إضافة S7/S7D/S6 على قمة A4 driver | [lwn.net/Articles/1022709](https://lwn.net/Articles/1022709/) |
| fix: device node reference leak في `aml_dt_node_to_map_pinmux` | [lkml.org/2026/2/19/5](https://lkml.org/lkml/2026/2/19/5) |
| fix: mark GPIO controller as sleeping | [lkml.org/2026/2/28/1118](https://lkml.org/lkml/2026/2/28/1118) |
| fix: pinctrl node في DTS للـ A4 | [lkml.org/2026/1/28/600](https://lkml.org/lkml/2026/1/28/600) |
| الـ original Meson pinctrl driver (2014) | [lkml.iu.edu — PATCH 1/3](https://lkml.iu.edu/hypermail/linux/kernel/1410.0/04163.html) |

---

### نقاشات الـ Mailing List

**linux-amlogic** هو القائمة البريدية الرئيسية لمتابعة تطور الـ driver:

- [Patchwork — linux-amlogic project](https://patchwork.kernel.org/project/linux-amlogic/list/)
- [GPIO interrupt support for Meson pinctrl (RFC v4)](https://patchwork.kernel.org/project/linux-amlogic/patch/3265fbc9-467d-67fa-8752-f750f234dedd@gmail.com/)
- [fix drive strength register calculation](https://patchwork.kernel.org/project/linux-amlogic/patch/20200610041329.12948-1-hhk7734@gmail.com/) — مهم لفهم `aml_pinconf_set_drive_strength()`
- [fix gxbb ao pull register bits](https://patchwork.kernel.org/project/linux-amlogic/patch/20181029151340.9087-2-jbrunet@baylibre.com/)
- [dt-bindings: convert Amlogic Meson pinctrl binding](https://lists.infradead.org/pipermail/linux-arm-kernel/2023-March/821938.html)
- [RFC: pinctrl driver for Amlogic SoCs — spinics](https://www.spinics.net/lists/linux-gpio/msg107463.html)

---

### مراجع eLinux.org

- [Amlogic — elinux.org](https://elinux.org/Amlogic) — فهرس عام لـ Amlogic SoCs، يشمل links لـ mainline kernel status وروابط الـ vendor BSP
- [Pin Control GPIO Update — PDF](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) — presentation قديمة لكنها تشرح العلاقة بين الـ pinctrl وGPIO بصورة بصرية
- [i2c-demux-pinctrl test](https://www.elinux.org/Tests:i2c-demux-pinctrl) — مثال على استخدام الـ pinctrl مع I2C

---

### kernelnewbies.org — متابعة التغييرات عبر الإصدارات

الصفحات التالية تُدرج كل تغيير في الـ pinctrl subsystem لكل إصدار kernel:

- [Linux_6.8 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.8)
- [Linux_6.11 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.11)
- [Linux_6.14 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.14)
- [Linux_6.15 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.15)
- [Linux_6.17 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.17)
- [LinuxChanges — kernelnewbies.org](https://kernelnewbies.org/LinuxChanges) — فهرس شامل لكل الإصدارات

ابحث في كل صفحة عن كلمة **pinctrl** أو **amlogic** لتتبع الإضافات الجديدة.

---

### كتب مُوصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 9**: Data Types in the Kernel — أنواع البيانات المستخدمة في الـ driver مثل `u32`, `u16`
- **الفصل 12**: PCI Drivers — patterns للـ `platform_driver` و`probe`
- **الفصل 14**: The Linux Device Model — فهم `struct device`, `devm_*` functions
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 14**: The Block I/O Layer — نمط الـ operations structs مثل `pinmux_ops`, `pinconf_ops`
- **الفصل 17**: Devices and Modules — `module_platform_driver`, `of_device_id`, `MODULE_DEVICE_TABLE`
- **الفصل 20**: Patches, Hacking, and the Community — كيفية تتبع الـ patches على LKML

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 15**: Debugging Embedded Linux — أدوات debug مفيدة لـ GPIO/pinctrl
- **الفصل 16**: Kernel Initialization — فهم ترتيب الـ probe وتهيئة الـ platform devices
- يُعطي context ممتاز لفهم كيف يتفاعل الـ pinctrl driver مع الـ bootloader وDTB

---

### Source Code ذو الصلة في الـ Kernel

```
drivers/pinctrl/meson/pinctrl-amlogic-a4.c   ← الملف الرئيسي
drivers/pinctrl/meson/                        ← drivers قديمة للمقارنة (meson8, axg, g12a...)
drivers/pinctrl/core.c                        ← الـ core subsystem
drivers/pinctrl/pinconf.c                     ← generic pinconf helpers
drivers/pinctrl/pinctrl-utils.c               ← utility functions
include/linux/pinctrl/pinctrl.h               ← struct pinctrl_dev, struct pinctrl_desc
include/linux/pinctrl/pinmux.h                ← struct pinmux_ops
include/linux/pinctrl/pinconf.h               ← struct pinconf_ops
include/linux/pinctrl/consumer.h              ← consumer API
include/dt-bindings/pinctrl/amlogic,pinctrl.h ← AMLOGIC_GPIO_* constants
```

---

### Search Terms للبحث عن معلومات إضافية

للبحث في الـ LKML وpatchwork وكود الـ kernel:

```
pinctrl amlogic a4 meson
pinctrl-amlogic-a4.c driver
amlogic,pinctrl-a4 device tree binding
pinmux_ops pinconf_ops implementation example
devm_pinctrl_register gpiochip_add_data
regmap_update_bits pinctrl GPIO
pinctrl_find_gpio_range_from_pin
gpio-ranges pinctrl kernel documentation
for_each_child_of_node_scoped pinctrl probe
pinconf_generic_parse_dt_pinmux
```
## Phase 8: Writing simple module

### الفكرة

الـ driver `pinctrl-amlogic-a4.c` بيسجّل GPIO banks عن طريق `gpiochip_add_data()`. الـ function دي معمولة للـ export في kernel ومتاحة للـ kprobe. هنعمل module بيعمل **kprobe** على `aml_gpio_direction_output` — الـ function اللي بتتقدم كل ما حاجة في النظام حاولت تحوّل GPIO pin لـ output وتكتب قيمة عليه. ده بيظهر: رقم الـ GPIO، القيمة المطلوبة (high/low)، واسم الـ gpio_chip (اللي هو اسم الـ bank زي "GPIOX").

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on aml_gpio_direction_output()
 * Logs every GPIO-as-output request on Amlogic A4/S7/S6 pinctrl driver.
 */
#include <linux/kernel.h>   /* pr_info, pr_err              */
#include <linux/module.h>   /* MODULE_* macros               */
#include <linux/kprobes.h>  /* struct kprobe, register_kprobe */
#include <linux/gpio/driver.h> /* struct gpio_chip, gpiochip_get_data */

/* ------------------------------------------------------------------ */
/* اسم الـ symbol اللي هنعمله probe — static function جوا الـ driver  */
#define PROBE_SYMBOL "aml_gpio_direction_output"

/*
 * pre_handler يتنفّذ قبل ما الـ kernel ينفّذ الـ instruction الأولى
 * في aml_gpio_direction_output.
 *
 * توقيع الـ function الأصلية:
 *   static int aml_gpio_direction_output(struct gpio_chip *chip,
 *                                        unsigned int gpio,
 *                                        int value);
 *
 * على x86_64:
 *   regs->di  = chip   (arg0)
 *   regs->si  = gpio   (arg1)
 *   regs->dx  = value  (arg2)
 *
 * على arm64:
 *   regs->regs[0] = chip
 *   regs->regs[1] = gpio
 *   regs->regs[2] = value
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct gpio_chip *chip;
    unsigned int gpio_offset;
    int value;

#if defined(CONFIG_X86_64)
    /* على x86_64 الـ arguments بتيجي في registers معينة */
    chip        = (struct gpio_chip *)regs->di;
    gpio_offset = (unsigned int)regs->si;
    value       = (int)regs->dx;
#elif defined(CONFIG_ARM64)
    /* على arm64 الـ arguments بتيجي في regs[0..2] */
    chip        = (struct gpio_chip *)regs->regs[0];
    gpio_offset = (unsigned int)regs->regs[1];
    value       = (int)regs->regs[2];
#else
    /* architecture مش مدعومة — نطبع تحذير بس نكمّل */
    pr_info_ratelimited("aml_gpio_probe: unsupported arch\n");
    return 0;
#endif

    /*
     * chip->label = اسم الـ bank (GPIOA, GPIOX, ...) اللي بيتعيّن
     * في aml_gpiolib_register_bank من aml_bank_name[].
     * gpio_offset = رقم الـ pin جوا الـ bank (من 0).
     * value = 1 يعني high, 0 يعني low.
     */
    pr_info("aml_gpio_probe: [%s] pin=%u -> %s\n",
            chip->label ? chip->label : "?",
            gpio_offset,
            value ? "HIGH" : "LOW");

    return 0; /* 0 = متابعة التنفيذ الطبيعي */
}

/* ------------------------------------------------------------------ */
/* الـ kprobe struct — بنحدد فيه اسم الـ symbol والـ handler          */
static struct kprobe aml_gpio_kp = {
    .symbol_name = PROBE_SYMBOL,
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
static int __init aml_gpio_probe_init(void)
{
    int ret;

    /*
     * register_kprobe بيبحث عن الـ symbol في kernel symbol table
     * وبيزرع breakpoint في أول instruction بتاعته.
     * لو الـ symbol مش موجود (driver مش loaded) بيرجع -ENOENT.
     */
    ret = register_kprobe(&aml_gpio_kp);
    if (ret < 0) {
        pr_err("aml_gpio_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("aml_gpio_probe: hooked %s at %px\n",
            PROBE_SYMBOL, aml_gpio_kp.addr);
    return 0;
}

static void __exit aml_gpio_probe_exit(void)
{
    /*
     * unregister_kprobe ضروري في exit عشان يشيل الـ breakpoint
     * من الذاكرة ويمنع أي crash لو الـ module اتفكّ وجه call
     * للـ handler اللي اتحذف.
     */
    unregister_kprobe(&aml_gpio_kp);
    pr_info("aml_gpio_probe: unhooked %s\n", PROBE_SYMBOL);
}

module_init(aml_gpio_probe_init);
module_exit(aml_gpio_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc example");
MODULE_DESCRIPTION("kprobe on aml_gpio_direction_output for Amlogic pinctrl");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/kprobes.h` | بيعرّف `struct kprobe` وفنكشنز `register_kprobe` / `unregister_kprobe` |
| `linux/gpio/driver.h` | بيعرّف `struct gpio_chip` عشان نقدر نقرأ `chip->label` من جوا الـ handler |
| `linux/module.h` | الـ macros الأساسية لأي kernel module |

---

#### الـ `handler_pre`

الـ `pre_handler` بيتنفّذ قبل أي instruction في `aml_gpio_direction_output`. بنجيب الـ arguments مباشرة من الـ CPU registers عشان الـ compiler لسه محطتهاش في stack frame — ده الطريقة الوحيدة الموثوقة في الـ kprobe context.

الـ `chip->label` بتكون اسم الـ bank اللي اتعيّن في `aml_gpiolib_register_bank`:
```c
bank->gpio_chip.label = aml_bank_name[bank->bank_id];
```
فهنشوف في الـ log حاجة زي:
```
aml_gpio_probe: [GPIOX] pin=5 -> HIGH
```

---

#### الـ `#if defined(CONFIG_X86_64) / CONFIG_ARM64`

الـ Amlogic A4/S7/S6 بتشتغل على **ARM64**، لكن الـ module محتاج يشتغل صح لو اتبنى على x86 للتجربة. الـ `#else` بيعمل `return 0` بدون crash.

---

#### الـ `module_init` / `module_exit`

- **init**: بيسجّل الـ kprobe — من اللحظة دي كل call لـ `aml_gpio_direction_output` بتعدي على الـ handler.
- **exit**: بيلغي التسجيل — لازم يحصل قبل ما الـ module يتحذف من memory عشان لو جه call بعد الـ unload ملاقيش handler بيعمل kernel panic.

---

### Makefile بسيط للتجربة

```makefile
obj-m := aml_gpio_probe.o

KDIR  ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
# تحميل الـ module
sudo insmod aml_gpio_probe.ko

# مشاهدة الـ log
sudo dmesg -w | grep aml_gpio_probe

# تفريغ الـ module
sudo rmmod aml_gpio_probe
```

---

### مثال على الـ output المتوقع

```
[  142.331201] aml_gpio_probe: hooked aml_gpio_direction_output at ffffffc0082a1c40
[  143.005892] aml_gpio_probe: [GPIOX] pin=5 -> HIGH
[  143.112034] aml_gpio_probe: [GPIOA] pin=2 -> LOW
[  143.998100] aml_gpio_probe: [GPIODV] pin=0 -> HIGH
```

كل سطر بيقولك: أنهي **bank**، أنهي **pin** جواه، والـ **قيمة** اللي الـ kernel طلبها.
