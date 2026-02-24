## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبعه الملف

الملف `include/linux/pinctrl/pinmux.h` جزء من **PIN CONTROL SUBSYSTEM** — المسجّل في `MAINTAINERS` تحت اسم "PIN CONTROL SUBSYSTEM"، ومنتهي في:
- `drivers/pinctrl/`
- `include/linux/pinctrl/`
- `Documentation/driver-api/pin-control.rst`

المسؤول عن الـ subsystem ده هو **Linus Walleij** من Linaro، وكتبه أصلاً لـ ST-Ericsson سنة 2011.

---

### الصورة الكبيرة — قبل أي كود

#### تخيّل المشكلة كده

عندك شريحة SoC (نظام متكامل على رقاقة) — زي Raspberry Pi أو أي معالج ARM — فيها مثلاً **100 pin** جسدي على أطراف الشريحة.

كل pin ممكن يشتغل بأكتر من دور:
- يشتغل **GPIO** (رقمي عادي: عالي/واطي)
- يشتغل **UART TX/RX** (منفذ تسلسلي)
- يشتغل **SPI CLK/MISO/MOSI** (ناقل بيانات سريع)
- يشتغل **I2C SDA/SCL**
- وغيرهم

**لكن** — كل pin مش ممكن يشتغل بكل الأدوار دي في نفس الوقت. لازم تختار. ده اللي بيتسمى **Pinmuxing** (اختصار لـ Pin Multiplexing).

#### القياس من الواقع

تخيّل إنك عندك **مقبس كهربا متعدد** — نفس الفتحة ممكن تحط فيها شاحن موبايل، أو جهاز تلفزيون، أو مكنة قهوة — بس مش الثلاثة مع بعض. لازم تختار واحد. الـ **pinmux** هو الكود اللي بيحدد "مين هيستخدم الفتحة دي دلوقتي؟"

---

### الـ Pinmux Subsystem بيحل إيه بالظبط؟

#### المشكلة بدون subsystem

لو مفيش نظام مركزي، كل driver هيحاول يمسك الـ pins اللي هو عايزها من غير ما يعرف إن driver تاني ممكن يكون محجّزها. النتيجة:
- تعارض بين الـ drivers
- hardware corruption
- سلوك غير متوقع

#### الحل

الـ **pinctrl core** بيلعب دور الـ "traffic controller" — بيسمح لـ driver يطلب pin، بيتأكد إن محدش تاني بيستخدمه، وبعدين بيقوله "اتفضل، الطريق واضح."

---

### هدف ملف `pinmux.h` تحديداً

الملف ده يعرّف **`struct pinmux_ops`** — وهي الـ **vtable** (جدول الـ function pointers) اللي كل **pin controller driver** لازم يملّيها عشان يقدر يشارك في نظام الـ pinmux.

ببساطة: ده الـ **contract** أو الـ **interface** اللي الـ kernel بيقول فيه لكل driver hardware جديد:
> "لو عايز تشتغل معايا وتدعم pinmux، لازم تنفّذ الـ functions دي."

---

### القصة الكاملة خطوة بخطوة

```
+-----------+       طلب UART       +--------------+       set_mux()      +----------+
|  UART     | -------------------> | pinctrl core | -------------------> |  HW PIN  |
|  Driver   |                      |  (core.c)    |   عبر pinmux_ops     | Controller|
+-----------+                      +--------------+                      +----------+
                                          |
                                    بيسأل: مين شايل الـ pin ده؟
                                    لو حد تاني → يرفض
                                    لو مفيش → يعمل request() ثم set_mux()
```

**الخطوات:**

1. الـ UART driver بيحتاج يشتغل → بيطلب من الـ pinctrl core إنه "يفعّل" الـ function الخاصة بيه على مجموعة معينة من الـ pins.
2. الـ core بيبص على جدول الـ `pinmux_ops` الخاص بالـ pin controller.
3. بيستدعي `request()` عشان يعرف لو الـ pin متاح.
4. لو متاح → بيستدعي `set_mux()` عشان يضبط الـ hardware فعلاً.
5. لو الـ driver خلص → بيستدعي `free()` عشان يرجّع الـ pin للـ pool.

---

### شرح `struct pinmux_ops` — كل field ولماذا موجود

```c
struct pinmux_ops {
    /* هل الـ pin متاح للـ muxing؟ */
    int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);

    /* تحرير الـ pin بعد الاستخدام */
    int (*free)(struct pinctrl_dev *pctldev, unsigned int offset);

    /* كام function متاحة في الـ controller ده؟ (UART, SPI, I2C...) */
    int (*get_functions_count)(struct pinctrl_dev *pctldev);

    /* إيه اسم الـ function رقم N؟ */
    const char *(*get_function_name)(struct pinctrl_dev *pctldev,
                                     unsigned int selector);

    /* الـ function دي بتشتغل على أنهي groups من الـ pins؟ */
    int (*get_function_groups)(struct pinctrl_dev *pctldev,
                               unsigned int selector,
                               const char * const **groups,
                               unsigned int *num_groups);

    /* هل الـ function دي هي GPIO بالذات؟ (مهم للـ strict mode) */
    bool (*function_is_gpio)(struct pinctrl_dev *pctldev,
                             unsigned int selector);

    /* فعّل الـ mux فعلاً على الـ hardware */
    int (*set_mux)(struct pinctrl_dev *pctldev,
                   unsigned int func_selector,
                   unsigned int group_selector);

    /* خلّي الـ pin ده GPIO تحديداً */
    int (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                               struct pinctrl_gpio_range *range,
                               unsigned int offset);

    /* اعكس gpio_request_enable */
    void (*gpio_disable_free)(struct pinctrl_dev *pctldev,
                              struct pinctrl_gpio_range *range,
                              unsigned int offset);

    /* الـ GPIO ده input ولا output؟ */
    int (*gpio_set_direction)(struct pinctrl_dev *pctldev,
                              struct pinctrl_gpio_range *range,
                              unsigned int offset,
                              bool input);

    /* strict = true: لا تسمح للـ pin يشتغل GPIO وfunction في نفس الوقت */
    bool strict;
};
```

---

### العلاقة بين الـ Concepts الأساسية

```
Pin Controller
    │
    ├── Pins         ← الـ physical pins (مرقّمة)
    │
    ├── Groups       ← مجموعات من الـ pins اللي بتشتغل مع بعض
    │                   مثلاً: {pin5, pin6, pin7} = "spi0_group"
    │
    └── Functions    ← الأدوار المتاحة
                        مثلاً: "uart0", "spi0", "i2c1", "gpio"
                        كل function ممكن تشتغل على group واحدة أو أكتر
```

الـ `pinmux_ops` هو الجسر بين الـ **core** (اللي بيعرف الـ concepts دي) وبين الـ **hardware driver** (اللي بيعرف كيف يضبط الـ registers فعلاً).

---

### الـ strict Mode — ليه مهم؟

الـ `strict` flag ده قصة مثيرة. طبيعياً الـ kernel بيسمح إن الـ pin يشتغل GPIO وفي نفس الوقت driver تاني يكون "مطالب" بيه لـ function تانية — طالما محدش بيستخدمه فعلاً. لكن لو `strict = true`، الـ core بيرفض على طول لو فيه أي تعارض محتمل حتى لو نظري. ده بيفيد في الـ hardware اللي بيستغيش تشغيل الحالتين مع بعض أصلاً.

---

### الملفات المرتبطة اللي لازم تعرفها

| الملف | دوره |
|---|---|
| `include/linux/pinctrl/pinctrl.h` | يعرّف `pinctrl_desc`, `pinctrl_ops`, `pingroup`, `pinfunction` — الهيكل الأساسي للـ pin controller |
| `include/linux/pinctrl/pinmux.h` | **ملفنا** — يعرّف `pinmux_ops` (الـ vtable) |
| `include/linux/pinctrl/pinconf.h` | يعرّف `pinconf_ops` — لضبط خصائص الـ pin (pull-up, drive strength...) |
| `include/linux/pinctrl/consumer.h` | واجهة الـ drivers اللي بتستخدم الـ pinctrl (مش اللي بتنفّذه) |
| `include/linux/pinctrl/machine.h` | الـ mapping table بين الـ devices والـ functions |
| `include/linux/pinctrl/pinctrl-state.h` | الـ states المعروفة: default, init, sleep, idle |
| `drivers/pinctrl/core.c` | قلب الـ subsystem — بينفّذ الـ logic الفعلي |
| `drivers/pinctrl/pinmux.c` | بينفّذ الـ pinmux logic ويستدعي الـ `pinmux_ops` |
| `drivers/pinctrl/devicetree.c` | بيقرأ الـ pinctrl config من Device Tree |

---

### أمثلة على Hardware Drivers بتنفّذ `pinmux_ops`

- `drivers/pinctrl/bcm/pinctrl-bcm2835.c` — Raspberry Pi
- `drivers/pinctrl/freescale/pinctrl-imx.c` — NXP i.MX processors
- `drivers/pinctrl/intel/pinctrl-intel.c` — Intel platforms
- `drivers/pinctrl/nomadik/` — ST-Ericsson (الـ SoC الأصلي اللي اتكتب ليه الكود)

---

### الخلاصة

الـ `pinmux.h` هو **عقد صغير وواضح** — سطر واحد من الـ struct بيحدد كل اللي الـ kernel محتاجه من أي hardware driver عايز يشارك في إدارة الـ pin multiplexing. الـ core بيدير التعارضات، والـ driver بيعرف فقط كيف يكلّم الـ hardware. الفصل ده هو قوة الـ Linux kernel subsystem architecture.
## Phase 2: شرح الـ Pinmux Framework

### المشكلة اللي الـ Pinmux بيحلها

في أي SoC حديث (STM32، i.MX8، Allwinner A64، إلخ) عندك مئات الـ physical pins على الـ chip. لكن الـ silicon اللي جوا الـ chip فيها عشرات الـ peripherals: UART، SPI، I2C، PWM، USB، Ethernet، إلخ. المشكلة الأساسية: **عدد الـ pins أقل بكتير من عدد الـ peripheral signals**.

الحل الأزلي في الـ hardware: كل pin ممكن تشتغل في أكتر من وظيفة — اللي بنسميها **pin multiplexing** أو **pinmux**. مثلاً:

```
Pin PA5 يقدر يكون:
  - GPIO output عادي
  - SPI1_SCK (clock لـ SPI bus)
  - TIM2_CH1 (PWM channel)
  - EVENTOUT (internal signal)
```

السؤال: مين بيختار؟ ومنين؟ وإزاي تمنع تعارض؟ لو driver الـ SPI اخد الـ pin وبعدين driver الـ UART حاول ياخده، مين بيمنع؟

الـ kernel قبل الـ pinctrl subsystem: كل driver كان بيكتب كود خاص بيه يعمل `ioremap` على الـ hardware register ويغير الـ bits يدويًا. النتيجة: chaos — كود متكرر، ما فيش تنسيق، وما فيش حماية من التعارض.

---

### الحل: الـ Pinctrl/Pinmux Subsystem

الـ kernel حل المشكلة بطبقة abstraction اسمها **pinctrl subsystem** (اتضافت في الـ kernel 3.0 تقريباً، 2011). الـ pinmux هو جزء منها مسؤول تحديداً عن اختيار الـ function لكل pin أو group من الـ pins.

الفكرة الأساسية:
- **Pin controller driver** واحد بيعرف كل الـ pins والـ functions الممكنة ويسجل نفسه مع الـ subsystem.
- الـ subsystem بيتولى الـ arbitration (مين ممسك الـ pin، conflict detection).
- الـ consumer drivers (SPI، UART، إلخ) بيطلبوا الـ state اللي محتاجينه باسم symbolic (مش بـ register address).

---

### الـ Real-World Analogy — مكتب الهاتف الحكومي

تخيل مبنى حكومي فيه **100 خط تليفون** بس فيه **500 موظف**. كل خط ممكن يتوصل لـ قسم مختلف:
- نفس الخط رقم 42 يقدر يكون: فاكس، تليفون رئيس، أو خط طوارئ.

الحل: في **مكتب مركزي** (الـ pinctrl core) بيدير كل الطلبات.

| المكتب المركزي | pinctrl core |
|---|---|
| الخط التليفوني | physical pin |
| الوظيفة (فاكس/تليفون) | mux function |
| القسم اللي بيطلب الخط | consumer driver (SPI/UART) |
| سجل استخدام الخطوط | مصفوفة `pin_desc` + `mux_owner` |
| مدير المكتب اللي بيحدد الطلبات | `pinmux_ops` في الـ controller driver |
| الموظف اللي بيفضل نفس الخط بدون إذن | conflict → الـ `strict` mode يمنعه |
| نموذج الطلب الرسمي | `struct pinctrl_map` |
| التوصيل الفعلي للخط | `set_mux()` تكتب الـ hardware register |

لما قسم جديد يطلب خطًا، المكتب بيتحقق: هل في حد تاني شايل الخط ده؟ لو آه يرفض. لو لأ يوافق ويسجّل مين الشايل.

هذا بالضبط ما بيعمله `pinmux_ops::request()` و`pinmux_ops::set_mux()` تحت إشراف الـ pinctrl core.

---

### الـ Big Picture Architecture

```
                     ┌─────────────────────────────────────────────────┐
                     │              User Space / Device Tree            │
                     │  pinctrl-0 = <&uart1_pins_default>;              │
                     └────────────────────┬────────────────────────────┘
                                          │ DT parse
                                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        pinctrl CORE  (drivers/pinctrl/core.c)           │
│                                                                         │
│  ┌──────────────┐   ┌────────────────┐   ┌─────────────────────────┐   │
│  │ pinctrl_map  │   │ pinctrl_state  │   │  conflict/owner tracking │   │
│  │ (mapping DB) │   │ (named states) │   │  pin_desc[].mux_owner   │   │
│  └──────┬───────┘   └───────┬────────┘   └─────────────────────────┘   │
│         │                   │                                           │
│         └──────────┬────────┘                                           │
│                    ▼                                                     │
│           pinmux_enable_setting()                                        │
│                    │                                                     │
└────────────────────┼────────────────────────────────────────────────────┘
                     │ calls ops
                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              Pin Controller Driver  (e.g., pinctrl-stm32.c)             │
│                                                                         │
│   struct pinctrl_desc {                                                 │
│       .pins    = stm32_pins[],    ← كل الـ pins موصوفة                 │
│       .pctlops = &stm32_pctl_ops, ← group/pin queries                  │
│       .pmxops  = &stm32_pmx_ops,  ← pinmux_ops ← الموضوع بتاعنا       │
│       .confops = &stm32_conf_ops, ← pull-up/drive-strength             │
│   }                                                                     │
│                                                                         │
│   ┌──────────────────────────────────────┐                              │
│   │         struct pinmux_ops            │                              │
│   │  .request()         → check HW      │                              │
│   │  .free()            → release HW    │                              │
│   │  .get_functions_count()             │                              │
│   │  .get_function_name()               │                              │
│   │  .get_function_groups()             │                              │
│   │  .set_mux()         → write regs    │                              │
│   │  .gpio_request_enable()             │                              │
│   │  .gpio_disable_free()               │                              │
│   │  .gpio_set_direction()              │                              │
│   │  .strict = true/false               │                              │
│   └──────────────────────────────────────┘                              │
│                    │                                                     │
└────────────────────┼────────────────────────────────────────────────────┘
                     │ writes
                     ▼
        ┌────────────────────────┐
        │   SoC Hardware Regs    │
        │  MODER / AFRL / AFSEL  │
        │  (STM32 example)       │
        └────────────────────────┘

    CONSUMERS (من فوق بيطلبوا الـ states):
    ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
    │  UART    │  │  SPI     │  │  I2C     │  │  GPIO    │
    │  driver  │  │  driver  │  │  driver  │  │  subsys  │
    └──────────┘  └──────────┘  └──────────┘  └──────────┘
```

---

### الـ Core Abstractions — المفاهيم الأساسية بالتفصيل

#### 1. الـ Pin

أصغر وحدة. كل pin عنده `number` فريد في الـ system وعنده `name`.

```c
struct pinctrl_pin_desc {
    unsigned int  number;  /* unique ID عددي */
    const char   *name;    /* "PA5", "GPIO42", إلخ */
    void         *drv_data;/* بيانات خاصة بالـ driver */
};
```

#### 2. الـ Pin Group

مجموعة pins بتشتغل مع بعض كـ وحدة واحدة. مثلاً SPI محتاج 4 pins (MOSI، MISO، SCK، CS) — دول **group واحد**.

```c
struct pingroup {
    const char         *name;   /* "spi1-grp" */
    const unsigned int *pins;   /* [5, 6, 7, 8] */
    size_t              npins;  /* 4 */
};
```

#### 3. الـ Function

الوظيفة اللي ممكن تتعين لـ group. مثلاً function اسمها `"spi1"` ممكن تتطبق على group `"spi1-grp"`.

```c
struct pinfunction {
    const char        *name;    /* "spi1" */
    const char *const *groups;  /* ["spi1-grp", "spi1-alt-grp"] */
    size_t             ngroups; /* عدد الـ groups المتاحة */
    unsigned long      flags;   /* PINFUNCTION_FLAG_GPIO لو GPIO */
};
```

#### 4. الـ Mapping Table

الـ `struct pinctrl_map` هي "الخريطة" اللي بتربط **device** بـ **state** بـ **function+group**. الـ board code أو الـ Device Tree بيعرفها.

```c
struct pinctrl_map {
    const char            *dev_name;      /* "spi1" — الـ device اللي بتخدمه */
    const char            *name;          /* "default" — اسم الـ state */
    enum pinctrl_map_type  type;          /* MUX_GROUP أو CONFIGS_PIN، إلخ */
    const char            *ctrl_dev_name; /* "pinctrl0" — مين الـ controller */
    union {
        struct pinctrl_map_mux     mux;   /* group + function */
        struct pinctrl_map_configs configs;
    } data;
};
```

مثال عملي (board file):

```c
static const struct pinctrl_map myboard_pin_map[] = {
    /* لما الـ SPI1 يكون active، استخدم الـ spi1 function على spi1-grp */
    PIN_MAP_MUX_GROUP_DEFAULT("spi1.0", "pinctrl0", "spi1-grp", "spi1"),
    /* لما الـ UART2 يكون active */
    PIN_MAP_MUX_GROUP_DEFAULT("uart2.0", "pinctrl0", "uart2-grp", "uart2"),
};
```

---

### العلاقة بين الـ Structs — مخطط كامل

```
pinctrl_desc
├── .pins[]          → pinctrl_pin_desc[]    (كل الـ pins في الـ SoC)
├── .pctlops         → pinctrl_ops           (group queries)
│     ├── get_groups_count()
│     ├── get_group_name()
│     └── get_group_pins()
├── .pmxops          → pinmux_ops            ← ملفنا هنا
│     ├── request()
│     ├── free()
│     ├── get_functions_count()
│     ├── get_function_name()
│     ├── get_function_groups()
│     ├── function_is_gpio()
│     ├── set_mux()                          ← الـ critical path
│     ├── gpio_request_enable()
│     ├── gpio_disable_free()
│     ├── gpio_set_direction()
│     └── .strict
└── .confops         → pinconf_ops           (pull/drive/slew config)

pinctrl_map[]        → mapping table (board/DT)
  └── pinctrl_map_mux
        ├── .group    → pingroup name
        └── .function → pinfunction name

pinctrl_gpio_range   → ربط GPIO numbering بالـ pin numbering
  ├── .base     → أول رقم GPIO
  ├── .pin_base → أول رقم pin مقابل
  ├── .npins    → عدد الـ pins في الـ range
  └── .gc       → gpio_chip*
```

---

### تفصيل `struct pinmux_ops` — كل callback ومتى بيتعمل call

#### `request(pctldev, offset)`

```c
int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);
```

الـ core بيعمله call **قبل** أي `set_mux()`. الـ driver بيتحقق: هل الـ pin ده متاح للـ mux أصلاً؟ بعض الـ pins ممكن تكون reserved للـ debug (JTAG) أو power — الـ driver يقدر يقول "لأ" بـ return سالب.

#### `free(pctldev, offset)`

عكس الـ `request()`. الـ core بيعمله call لما الـ consumer بيعمل deselect للـ state أو بيعمل release للـ pinctrl handle.

#### `get_functions_count(pctldev)`

```c
int (*get_functions_count)(struct pinctrl_dev *pctldev);
```

بيرجع عدد الـ functions المتاحة في الـ controller (مثلاً 50 function). الـ core بيستخدمه للـ enumeration والـ debugfs.

#### `get_function_name(pctldev, selector)`

```c
const char *(*get_function_name)(struct pinctrl_dev *pctldev,
                                  unsigned int selector);
```

الـ `selector` هو index من 0 لـ `count-1`. بيرجع الاسم: `"spi1"`, `"uart2"`, إلخ. الـ core بيعمل lookup بالاسم لما بيقرأ الـ mapping table.

#### `get_function_groups(pctldev, selector, groups, num_groups)`

```c
int (*get_function_groups)(struct pinctrl_dev *pctldev,
                           unsigned int selector,
                           const char *const **groups,
                           unsigned int *num_groups);
```

بعض الـ functions ممكن تشتغل على أكتر من group (routing alternatives). مثلاً UART2 ممكن يكون على `uart2-grp-a` أو `uart2-grp-b` — الـ board بيختار من الـ mapping table أي واحد.

#### `function_is_gpio(pctldev, selector)`

```c
bool (*function_is_gpio)(struct pinctrl_dev *pctldev,
                          unsigned int selector);
```

مهمة جداً للـ `strict` mode. بيعرف الـ core إن function معينة هي GPIO function. الـ STM32 مثلاً عنده function اسمها `"gpio"` لكل pin — الـ core لازم يعرفها عشان يطبق قواعد مختلفة في الـ conflict check.

#### `set_mux(pctldev, func_selector, group_selector)`

```c
int (*set_mux)(struct pinctrl_dev *pctldev,
               unsigned int func_selector,
               unsigned int group_selector);
```

**ده الـ critical path** — ده اللي بيكتب فعلاً في الـ hardware registers. الـ core بيديه `func_selector` و`group_selector` بعد ما يتحقق من الـ conflicts. الـ driver بيفضل يعمل:

```c
static int stm32_pmx_set_mux(struct pinctrl_dev *pctldev,
                              unsigned func_sel, unsigned grp_sel)
{
    /* جيب الـ group */
    const struct pingroup *grp = &pdata->groups[grp_sel];
    /* للـ function المحددة، ايه الـ alternate function number؟ */
    unsigned af = pdata->functions[func_sel].af_num;

    for (i = 0; i < grp->npins; i++) {
        pin = grp->pins[i];
        /* اكتب في الـ AFR register للـ STM32 */
        stm32_pmx_set_af(pctldev, pin, af);
    }
    return 0;
}
```

#### GPIO-specific callbacks

```c
int  (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                             struct pinctrl_gpio_range *range,
                             unsigned int offset);
void (*gpio_disable_free)  (struct pinctrl_dev *pctldev,
                             struct pinctrl_gpio_range *range,
                             unsigned int offset);
int  (*gpio_set_direction) (struct pinctrl_dev *pctldev,
                             struct pinctrl_gpio_range *range,
                             unsigned int offset,
                             bool input);
```

الـ GPIO subsystem (مش الـ pinmux consumers العاديين) بيستخدم دول مباشرة عن طريق `consumer.h`:

```c
pinctrl_gpio_request()        → gpio_request_enable()
pinctrl_gpio_free()           → gpio_disable_free()
pinctrl_gpio_direction_input()→ gpio_set_direction(..., true)
```

الـ `pinctrl_gpio_range` بيعرف الـ mapping بين الـ GPIO namespace والـ pin namespace:

```c
struct pinctrl_gpio_range {
    struct list_head  node;
    const char       *name;
    unsigned int      id;
    unsigned int      base;      /* أول GPIO number */
    unsigned int      pin_base;  /* أول pin number مقابل */
    unsigned int      npins;     /* عدد الـ pins */
    unsigned int     *pins;      /* أو array صريح */
    struct gpio_chip *gc;        /* الـ gpio_chip المقابلة */
};
```

#### `.strict`

```c
bool strict;
```

لو `true`: الـ core يرفض أي طلب يجمع بين GPIO use ومux use على نفس الـ pin في نفس الوقت — حتى لو الـ hardware نفسه يقدر يعمل كده. الـ `function_is_gpio()` لازم تكون موجودة عشان الـ strict mode يشتغل صح.

---

### الـ Pinmux بيملك إيه vs. بيفوّض إيه

| الـ pinmux core يملك (يتحكم فيه) | الـ controller driver يملك (بيفوّضه الـ core) |
|---|---|
| Conflict detection: مين ممسك الـ pin | تحديد الـ functions والـ groups الفعلية |
| Arbitration: تسجيل الـ mux_owner لكل pin | كتابة الـ hardware registers |
| State machine: `default`/`sleep`/`idle` | الـ hardware-level lock (request/free) |
| Mapping table lookup (اسم → selector) | إيه معنى كل selector على الـ HW |
| GPIO/mux conflict rules (strict mode) | هل الـ pin فعلاً يقدر يشتغل كـ GPIO |
| debugfs entries للـ pin states | per-pin debug info (`pin_dbg_show`) |

---

### تدفق طلب كامل — من الـ consumer للـ hardware

```
Consumer driver probe():
    devm_pinctrl_get(dev)
        → core يبحث في mapping table عن entries لـ dev
        → يبني struct pinctrl مع كل الـ states

    pinctrl_lookup_state(p, "default")
        → يرجع struct pinctrl_state*

    pinctrl_select_state(p, state)
        ↓
        pinmux_enable_setting()
            ↓
            لكل pin في الـ group:
                pinmux_ops.request(pctldev, pin)   ← driver: هل الـ pin متاح؟
                تحديث pin_desc[pin].mux_owner
            ↓
            pinmux_ops.set_mux(pctldev,
                               func_selector,
                               group_selector)      ← driver: اكتب الـ registers
```

---

### مثال عملي: UART على STM32

على STM32MP1، الـ UART4 بيحتاج PA11 (TX) و PA12 (RX).

الـ Device Tree:

```dts
&uart4 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart4_pins_a>;   /* default state */
    pinctrl-1 = <&uart4_sleep_pins_a>;
};

uart4_pins_a: uart4-0 {
    pins1 { /* TX */
        pinmux = <STM32_PINMUX('A', 11, AF6)>;
        bias-disable;
        drive-push-pull;
    };
    pins2 { /* RX */
        pinmux = <STM32_PINMUX('A', 12, AF6)>;
        bias-disable;
    };
};
```

الـ `STM32_PINMUX('A', 11, AF6)` بيبني `group_selector` و`func_selector`. لما الـ UART driver بيعمل `pinctrl_select_state(..., "default")`:

1. الـ core بيلاقي الـ group اللي فيه PA11 وPA12.
2. بيعمل call `request()` للاتنين.
3. بيسجلهم `mux_owner = "uart4.0"`.
4. بيعمل call `set_mux()` اللي بيكتب `AF6` في الـ `AFRL`/`AFRH` registers للـ STM32.

لو حاجة تانية حاولت تاخد PA11 بعد كده، الـ core بيشوف `mux_owner != NULL` ويرجع `-EBUSY`.

---

### علاقة الـ Pinmux بالـ subsystems التانية

**الـ GPIO subsystem** (اللي لازم تعرفه): بيوفر `struct gpio_chip` اللي بتمثل الـ GPIO controller. الـ pinmux بيتكلم معاه عن طريق `pinctrl_gpio_range` و`gpio_request_enable()`. الـ GPIO subsystem مش بيكتب الـ mux registers هو — بيعمل call للـ pinmux.

**الـ Pinconf subsystem**: مسؤول عن الـ electrical properties (pull-up، drive strength، slew rate). الـ pinmux مش بيتدخل فيها — الـ `pinctrl_desc` بيضم الاتنين `pmxops` و`confops` جنب بعض.

**الـ Device Tree subsystem**: الـ `pinctrl_ops.dt_node_to_map()` بتحوّل الـ DT nodes لـ `pinctrl_map[]` entries. من غيره كل المعلومات كانت هتتكتب في الـ board C files يدويًا.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags, Enums, Config Options — Cheatsheet

#### `enum pinctrl_map_type` (من `machine.h`)

| القيمة | المعنى |
|---|---|
| `PIN_MAP_TYPE_INVALID` | entry فاسد / غير معرَّف |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمية — لا تعمل على hardware حقيقي |
| `PIN_MAP_TYPE_MUX_GROUP` | تحديد function لمجموعة pins عبر الـ mux |
| `PIN_MAP_TYPE_CONFIGS_PIN` | إعداد configuration لـ pin واحد |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | إعداد configuration لمجموعة pins |

#### `PINFUNCTION_FLAG_GPIO` (من `pinctrl.h`)

| الـ Flag | القيمة | الاستخدام |
|---|---|---|
| `PINFUNCTION_FLAG_GPIO` | `BIT(0)` | يحدد إن الـ function دي هي GPIO — بتُستخدم مع `function_is_gpio` callback |

#### Kconfig Options المؤثرة على الملف

| الـ Option | الأثر |
|---|---|
| `CONFIG_PINCTRL` | لو مش مفعَّل، كل الـ mapping APIs بتبقى stubs فاضية |
| `CONFIG_GENERIC_PINCONF` | بيضيف `custom_params` و `custom_conf_items` جوه `pinctrl_desc` |
| `CONFIG_OF` | بيفعّل `of_pinctrl_get()` للـ Device Tree |

#### Macros المهمة لتعريف Mapping Table

| الـ Macro | الغرض |
|---|---|
| `PIN_MAP_DUMMY_STATE(dev, state)` | يعرّف state وهمية |
| `PIN_MAP_MUX_GROUP(dev, state, pinctrl, grp, func)` | يربط group بـ function |
| `PIN_MAP_MUX_GROUP_DEFAULT(...)` | نفسه بس state = `default` |
| `PIN_MAP_MUX_GROUP_HOG(...)` | الـ controller نفسه يحجز الـ mapping لنفسه |
| `PIN_MAP_CONFIGS_PIN(...)` | configuration لـ pin واحد |
| `PIN_MAP_CONFIGS_GROUP(...)` | configuration لـ group من pins |
| `PINCTRL_PIN(a, b)` | يعرّف pin descriptor بـ number واسم |
| `PINCTRL_PINGROUP(name, pins, npins)` | يعرّف pingroup |
| `PINCTRL_PINFUNCTION(name, groups, ngroups)` | يعرّف function |
| `PINCTRL_GPIO_PINFUNCTION(name, groups, ngroups)` | نفسه بس بـ `PINFUNCTION_FLAG_GPIO` |

---

### الـ Structs المهمة

#### 1. `struct pinmux_ops` — قلب الملف

**الغرض**: الـ vtable (جدول الـ virtual functions) اللي كل driver لازم يملأه عشان يدعم الـ pinmuxing. الـ core بيستخدمه للتفاوض مع الـ hardware عن ملكية الـ pins والـ mux settings.

| الـ Field | النوع | الشرح |
|---|---|---|
| `request` | `int (*)(pctldev, offset)` | الـ core بيسأل: "ممكن أستخدم الـ pin ده؟" — الـ driver يرد بـ 0 أو error |
| `free` | `int (*)(pctldev, offset)` | عكس `request` — يحرر الـ pin بعد ما خلصنا منه |
| `get_functions_count` | `int (*)(pctldev)` | كام function متاح في الـ controller ده؟ |
| `get_function_name` | `const char *(*)(pctldev, selector)` | اسم الـ function رقم N |
| `get_function_groups` | `int (*)(pctldev, selector, **groups, *num_groups)` | الـ groups اللي بتدعم الـ function ده |
| `function_is_gpio` | `bool (*)(pctldev, selector)` | هل الـ function ده هو GPIO؟ (مهم للـ strict mode) |
| `set_mux` | `int (*)(pctldev, func_selector, group_selector)` | الأمر الحقيقي — اكتب في الـ register عشان تفعّل الـ mux |
| `gpio_request_enable` | `int (*)(pctldev, range, offset)` | حوّل الـ pin لـ GPIO mode |
| `gpio_disable_free` | `void (*)(pctldev, range, offset)` | عكس `gpio_request_enable` |
| `gpio_set_direction` | `int (*)(pctldev, range, offset, input)` | اضبط اتجاه الـ GPIO: input أو output |
| `strict` | `bool` | لو `true`، مش مسموح لـ pin يكون GPIO وfunction في نفس الوقت |

**الربط بالـ Structs التانية**: بيتحط جوه `pinctrl_desc.pmxops`.

---

#### 2. `struct pinctrl_desc` — بطاقة تعريف الـ Controller

**الغرض**: الـ descriptor الكامل للـ pin controller. الـ driver بيملأه وبيمرره لـ `pinctrl_register_and_init()`.

| الـ Field | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | اسم الـ controller (بيظهر في sysfs/debugfs) |
| `pins` | `const struct pinctrl_pin_desc *` | array بكل الـ pins اللي الـ controller بيتحكم فيها |
| `npins` | `unsigned int` | عدد الـ pins |
| `pctlops` | `const struct pinctrl_ops *` | vtable الـ grouping والـ DT parsing |
| `pmxops` | `const struct pinmux_ops *` | vtable الـ pinmux — ده اللي في ملفنا |
| `confops` | `const struct pinconf_ops *` | vtable الـ pin configuration (pull-up, drive strength...) |
| `owner` | `struct module *` | للـ refcounting |
| `num_custom_params` | `unsigned int` | (لو `CONFIG_GENERIC_PINCONF`) عدد الـ params الخاصة بالـ driver |
| `custom_params` | `const struct pinconf_generic_params *` | تعريف الـ params الخاصة |
| `custom_conf_items` | `const struct pin_config_item *` | طريقة عرضها في debugfs |
| `link_consumers` | `bool` | يعمل device link مع الـ consumers عشان يضمن ترتيب suspend/resume |

---

#### 3. `struct pinctrl_pin_desc` — وصف الـ Pin الواحد

**الغرض**: يعرّف كل pin في الـ controller بـ ID واسم وبيانات خاصة بالـ driver.

| الـ Field | النوع | الشرح |
|---|---|---|
| `number` | `unsigned int` | رقم الـ pin الفريد في الـ global pin space |
| `name` | `const char *` | اسم الـ pin (مثلاً `"PA0"`, `"GPIO_0"`) |
| `drv_data` | `void *` | بيانات خاصة بالـ driver — الـ core مش بيلمسها |

---

#### 4. `struct pingroup` — مجموعة الـ Pins

**الغرض**: بيجمع مجموعة pins تحت اسم واحد. الـ functions بتشتغل على groups مش على pins منفردة.

| الـ Field | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | اسم الـ group (مثلاً `"spi0_grp"`) |
| `pins` | `const unsigned int *` | array بأرقام الـ pins في الـ group |
| `npins` | `size_t` | عدد الـ pins |

---

#### 5. `struct pinfunction` — تعريف الـ Function

**الغرض**: يعرّف الـ mux function (مثلاً SPI, UART, I2C) والـ groups اللي ممكن تستخدمه.

| الـ Field | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | اسم الـ function (مثلاً `"spi0"`, `"uart1"`, `"gpio"`) |
| `groups` | `const char * const *` | أسماء الـ groups المتوافقة مع الـ function ده |
| `ngroups` | `size_t` | عدد الـ groups |
| `flags` | `unsigned long` | `PINFUNCTION_FLAG_GPIO` لو ده GPIO function |

---

#### 6. `struct pinctrl_gpio_range` — نطاق الـ GPIO

**الغرض**: يعرّف مجال من أرقام الـ GPIO اللي الـ pin controller بيتحكم فيها، ويربطها بـ `gpio_chip`.

| الـ Field | النوع | الشرح |
|---|---|---|
| `node` | `struct list_head` | للربط في قائمة الـ ranges في الـ controller |
| `name` | `const char *` | اسم المجال |
| `id` | `unsigned int` | ID للـ range |
| `base` | `unsigned int` | أول رقم GPIO في الـ range |
| `pin_base` | `unsigned int` | أول رقم pin مقابل للـ range (لو `pins == NULL`) |
| `npins` | `unsigned int` | عدد الـ pins/GPIOs في الـ range |
| `pins` | `unsigned int const *` | array صريح بأرقام الـ pins (أو NULL) |
| `gc` | `struct gpio_chip *` | الـ gpio_chip المقابل لـ range ده |

---

#### 7. `struct pinctrl_map` — جدول الـ Mapping

**الغرض**: يربط device بـ function أو configuration في حالة (state) معينة. الـ board code بيعرّف array منهم.

| الـ Field | النوع | الشرح |
|---|---|---|
| `dev_name` | `const char *` | اسم الـ device اللي محتاج الـ mapping ده |
| `name` | `const char *` | اسم الـ state (مثلاً `"default"`, `"sleep"`) |
| `type` | `enum pinctrl_map_type` | نوع الـ mapping |
| `ctrl_dev_name` | `const char *` | اسم الـ pin controller |
| `data.mux` | `struct pinctrl_map_mux` | (لو النوع MUX_GROUP) الـ group والـ function |
| `data.configs` | `struct pinctrl_map_configs` | (لو النوع CONFIGS_*) الـ pin/group والـ configs |

---

### رسم العلاقات بين الـ Structs

```
+---------------------------+
|      pinctrl_desc         |  <-- الـ driver بيملأه ويسجّله
|---------------------------|
| name                      |
| pins --> pinctrl_pin_desc[]|
| npins                     |
| pctlops --> pinctrl_ops   |  (grouping, DT)
| pmxops  --> pinmux_ops    |  (muxing — ملفنا)
| confops --> pinconf_ops   |  (configuration)
| owner                     |
+---------------------------+
            |
            | pinctrl_register_and_init()
            v
+---------------------------+
|      pinctrl_dev          |  <-- الـ core يعمله ويحتفظ بيه (opaque للـ drivers)
|---------------------------|
| desc --> pinctrl_desc     |
| gpio_ranges --> list_head |---> pinctrl_gpio_range --> pinctrl_gpio_range --> ...
| ...                       |
+---------------------------+

+---------------------------+        +---------------------------+
|      pinfunction          |        |        pingroup           |
|---------------------------|        |---------------------------|
| name: "spi0"              |        | name: "spi0_grp"          |
| groups: ["spi0_grp", ...] |------> | pins: [10, 11, 12, 13]   |
| ngroups: 1                |        | npins: 4                  |
| flags: 0                  |        +---------------------------+
+---------------------------+

+---------------------------+
|      pinctrl_map          |  <-- board code
|---------------------------|
| dev_name: "spi0"          |
| name: "default"           |
| type: MUX_GROUP           |
| ctrl_dev_name: "pinctrl0" |
| data.mux:                 |
|   group: "spi0_grp"       |
|   function: "spi0"        |
+---------------------------+

+---------------------------+
|    pinctrl_gpio_range     |
|---------------------------|
| base: 32  (GPIO start)    |
| pin_base: 64 (pin start)  |
| npins: 32                 |
| gc --> gpio_chip          |
| node --> list_head        |
+---------------------------+
```

---

### رسم دورة الحياة — Lifecycle Diagram

```
[Board/Driver Code]
       |
       | 1. يعرّف pinctrl_desc بـ pmxops مملوء
       v
pinctrl_register_and_init(desc, dev, drv_data, &pctldev)
       |
       | 2. الـ core يخلق pinctrl_dev داخلياً
       | 3. يسجّل الـ pins، يبني الـ radix tree
       v
pinctrl_enable(pctldev)
       |
       | 4. يبدأ يطبق الـ hog mappings (اللي الـ controller بيحجزها لنفسه)
       v
[Controller جاهز — devices ممكن تطلب states]
       |
       | 5. device يطلب state عبر pinctrl_get() + pinctrl_lookup_state() + pinctrl_select_state()
       |      --> الـ core يبحث في الـ mapping table
       |      --> يلاقي MUX_GROUP entry
       |      --> يسمي pinmux_ops->request() لكل pin في الـ group
       |      --> يسمي pinmux_ops->set_mux()
       v
[Pins شغّالة بالـ function المطلوب]
       |
       | 6. device يطلب state تانية (مثلاً "sleep")
       |      --> الـ core يـ release الـ old mux
       |      --> يطبق الـ new mux
       v
[Device يتوقف / Module Unload]
       |
       | 7. pinctrl_unregister(pctldev)
       |      --> يحرر كل الـ pins المحجوزة
       |      --> يمسح الـ gpio_ranges
       |      --> يحذف pinctrl_dev
       v
[Done]
```

**GPIO Path بديل**:
```
gpio_request(gpio_num)
       |
       v
pinctrl_gpio_request(pin)
       |
       | --> pinmux_ops->request(pctldev, pin)
       | --> pinmux_ops->gpio_request_enable(pctldev, range, offset)
       v
[Pin في GPIO mode]
       |
gpio_direction_input() / gpio_direction_output()
       |
       | --> pinmux_ops->gpio_set_direction(pctldev, range, offset, input)
       v
[GPIO جاهز للاستخدام]
```

---

### رسم تدفق الـ Calls — Call Flow Diagrams

#### تطبيق Mux Function

```
device driver calls pinctrl_select_state(p, state)
  └─> pinctrl core: pinmux_enable_setting()
        └─> for each pin in group:
              └─> pin_request()
                    └─> ops->request(pctldev, pin_offset)
                          └─> [driver validates pin is free]
                                └─> returns 0 or -EBUSY
        └─> ops->set_mux(pctldev, func_selector, group_selector)
              └─> [driver writes to HW mux register]
                    └─> e.g.: writel(FUNC_SPI, base + MUX_REG(pin))
```

#### طلب GPIO

```
gpiod_get(dev, "reset", GPIOD_OUT_LOW)
  └─> gpio subsystem: gpiochip_request_own_desc()
        └─> pinctrl_gpio_request(gpio_num)
              └─> pinmux core: pin_request(pctldev, pin)
                    └─> ops->request(pctldev, offset)
              └─> ops->gpio_request_enable(pctldev, range, offset)
                    └─> [driver sets pin to GPIO mux]

gpio_direction_output(gpio, 0)
  └─> gpiochip->direction_output()
        └─> pinctrl_gpio_direction_output()
              └─> ops->gpio_set_direction(pctldev, range, offset, false)
                    └─> [driver sets direction register]
```

#### الاستعلام عن الـ Functions والـ Groups

```
pinctrl core (init / DT parse):
  └─> ops->get_functions_count(pctldev)
        └─> returns N
  └─> for i in 0..N:
        └─> ops->get_function_name(pctldev, i)
              └─> returns "spi0", "uart1", "gpio", ...
        └─> ops->get_function_groups(pctldev, i, &groups, &num)
              └─> returns ["spi0_grp"], 1
        └─> ops->function_is_gpio(pctldev, i)   [optional]
              └─> returns true/false
```

---

### استراتيجية الـ Locking

الـ pinmux.h نفسه ما بيعرّفش locks — دي مسؤولية الـ core (`drivers/pinctrl/core.c`). لكن الـ design بيفترض الآتي:

#### الـ Locks اللي بتحمي الـ Pinmux

| الـ Lock | المكان | اللي بيحميه |
|---|---|---|
| `pctldev->mutex` | داخل `pinctrl_dev` | كل عمليات الـ request/free/set_mux — يمنع race بين threads |
| `pinctrl_list_mutex` (global) | الـ core | قائمة الـ controllers المسجّلين (`pinctrldev_list`) |
| `gpio_lock` (في gpio_chip) | gpio subsystem | الـ gpio_ranges المربوطة بالـ controller |

#### قواعد الـ Lock Ordering (تسلسل الـ Locks)

```
[أعلى] pinctrl_list_mutex      (global — يُقفل أول)
           |
           v
       pctldev->mutex           (per-controller — يُقفل تاني)
           |
           v
       gpio_chip->bgpio_lock    (per-gpio-chip — يُقفل أخيراً)
```

**القاعدة الذهبية**: أي callback في `pinmux_ops` بيُستدعى وهو شايل `pctldev->mutex`. يعني الـ driver callbacks **ممنوع** يحاولوا يقفلوا الـ mutex تاني (deadlock).

#### الـ `strict` flag وأثره على الـ Locking

```c
bool strict;  // في pinmux_ops
```

لو `strict = true`، الـ core بيفحص **قبل** ما يديك الـ pin:
- هل في `gpio_owner` بيستخدم الـ pin ده؟
- هل في `mux_owner` تاني بيستخدمه؟

لو الإجابة بـ "أيوه" في أي منهم → يرفض الـ request بـ `-EBUSY` حتى لو الـ function المطلوب هو GPIO نفسه.

---

### ملخص العلاقات الكاملة

```
Board/DT
  |
  | pinctrl_map[] array
  v
pinctrl core (machine.c)
  |
  | يبحث بالاسم
  v
pinctrl_dev  <------------ pinctrl_register_and_init()
  |                                    ^
  |                                    |
  +---> pinctrl_desc ──────────────────+
  |       |
  |       +---> pinctrl_pin_desc[]   (كل الـ pins)
  |       +---> pinctrl_ops          (grouping)
  |       +---> pinmux_ops           (muxing)  ← ملفنا
  |       +---> pinconf_ops          (config)
  |
  +---> gpio_ranges list
          |
          +---> pinctrl_gpio_range ──> gpio_chip

pinmux_ops callbacks:
  request / free          → التحقق من توافر الـ pin
  get_functions_count     → عدد الـ functions
  get_function_name       → اسم الـ function
  get_function_groups     → الـ groups المدعومة
  function_is_gpio        → هل ده GPIO function؟
  set_mux                 → الكتابة الفعلية في الـ HW
  gpio_request_enable     → تحويل الـ pin لـ GPIO
  gpio_disable_free       → تحرير GPIO mode
  gpio_set_direction      → input / output
  strict (flag)           → منع الاستخدام المزدوج
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

الـ `pinmux.h` بيعرّف struct واحدة بس هي `struct pinmux_ops`، وهي الـ **vtable** اللي كل pin controller driver لازم يملّيها عشان يدعم الـ pinmuxing. مفيش functions تانية معرّفة في الفايل ده — كل حاجة هنا callbacks يطلبها الـ pinmux core من الـ driver.

| الـ Callback | النوع | الإلزامية | الغرض |
|---|---|---|---|
| `request` | `int (*)` | اختيارية | يتحقق إن الـ pin متاح للـ mux |
| `free` | `int (*)` | اختيارية | يرجع الـ pin بعد ما خلص |
| `get_functions_count` | `int (*)` | إلزامية | عدد الـ functions المتاحة |
| `get_function_name` | `const char *(*)` | إلزامية | اسم الـ function بالـ index |
| `get_function_groups` | `int (*)` | إلزامية | الـ groups المرتبطة بالـ function |
| `function_is_gpio` | `bool (*)` | اختيارية | هل الـ function دي GPIO؟ |
| `set_mux` | `int (*)` | إلزامية | يفعّل الـ mux على الـ hardware |
| `gpio_request_enable` | `int (*)` | اختيارية | يحوّل الـ pin لـ GPIO mode |
| `gpio_disable_free` | `void (*)` | اختيارية | يلغي الـ GPIO mode |
| `gpio_set_direction` | `int (*)` | اختيارية | يضبط الاتجاه (input/output) للـ GPIO |
| `strict` (field) | `bool` | اختيارية | يمنع التشارك بين GPIO و mux |

---

### تصنيف الـ Callbacks

الـ callbacks اتجمعت في تلات مجموعات حسب وظيفتها:

1. **Pin Ownership** — `request` / `free`: إدارة ملكية الـ pin.
2. **Function Enumeration** — `get_functions_count` / `get_function_name` / `get_function_groups` / `function_is_gpio`: الـ core بيستخدمهم عشان يعرف إيه اللي الـ controller بيعرضه.
3. **Mux Activation** — `set_mux`: التفعيل الفعلي على الـ hardware.
4. **GPIO Acceleration** — `gpio_request_enable` / `gpio_disable_free` / `gpio_set_direction`: bypass لمسار الـ mux العادي عشان الـ GPIO أسرع.

---

### المجموعة الأولى: Pin Ownership Callbacks

الـ core بيطلب الـ pins قبل ما يغيّر أي mux setting. الـ driver بيتحقق إن الـ pin مش محجوز لأغراض تانية على مستوى الـ hardware (مثلاً pin مربوط بـ always-on clock).

---

#### `request`

```c
int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);
```

الـ core بيستدعيه عشان يعرف لو ممكن يستخدم الـ pin ده في الـ mux. الـ driver بيتحقق من حاجات hardware-specific زي إن الـ pin مش locked أو مش في وضع protected. لو الـ driver مش عايز يدي الـ pin يرجّع error سالب.

**Parameters:**
- `pctldev`: الـ pin controller device handle، جاي من `pinctrl_register_and_init()`.
- `offset`: رقم الـ pin في الـ global pin number space بتاع الـ controller ده.

**Return:**
- `0` لو الـ pin متاح.
- سالب (مثلاً `-EBUSY` أو `-EINVAL`) لو مش متاح.

**Key details:**
- بيتستدعى قبل `set_mux` — الـ core بيعمل acquire للـ pins الأول.
- لو الـ controller معندوش hardware ownership concept، ممكن ما يحتاجش يعمل implement ليه.
- الـ locking: الـ pinctrl core بياخد `pinctrl_dev`'s lock قبل ما يكال الـ callback.

**Caller context:**
الـ pinmux core عند تطبيق state جديدة — `pinmux_request_gpio()` أو `pinmux_enable_setting()`.

---

#### `free`

```c
int (*free)(struct pinctrl_dev *pctldev, unsigned int offset);
```

عكس `request`، بيتستدعى لما الـ core خلص من الـ pin وعايز يحرره. الـ driver بيعمل أي cleanup hardware-level محتاجه زي تسجيل إن الـ pin مش محجوز.

**Parameters:**
- `pctldev`: نفس الـ controller handle.
- `offset`: رقم الـ pin.

**Return:**
- `0` عادةً (نادراً بيرجع error).

**Key details:**
- symmetric مع `request`.
- بعض الـ drivers بيتركوه `NULL` لو `request` نفسه `NULL`.

---

### المجموعة الثانية: Function Enumeration Callbacks

الـ pinmux core محتاج يعرف إيه الـ functions اللي الـ controller بيدعمها وإيه الـ pin groups المرتبطة بكل function. ده بيحصل عند الـ initialization أو عند parsing الـ device tree.

---

#### `get_functions_count`

```c
int (*get_functions_count)(struct pinctrl_dev *pctldev);
```

بيرجع عدد الـ mux functions الكلي اللي الـ controller بيعرفها. الـ core بيستخدمه كـ upper bound عند الـ iteration بالـ selector.

**Parameters:**
- `pctldev`: الـ controller handle.

**Return:**
- عدد صحيح موجب يمثل عدد الـ functions.

**Key details:**
- بيتستدعى مرة واحدة عند الـ registration أو عند debugfs.
- الـ selector في باقي الـ callbacks بيكون في النطاق `[0, count-1]`.

---

#### `get_function_name`

```c
const char *(*get_function_name)(struct pinctrl_dev *pctldev,
                                  unsigned int selector);
```

بيرجع اسم الـ function بالـ index (selector). الـ core بيستخدم الاسم ده عشان يعمل match مع الـ device tree أو الـ board file mappings.

**Parameters:**
- `pctldev`: الـ controller handle.
- `selector`: الـ index من `0` لـ `get_functions_count() - 1`.

**Return:**
- pointer لـ string ثابتة (عادةً في الـ driver's static array) — مش بيتعمل لها `kfree`.

**Key details:**
- الـ string لازم تفضل valid طول عمر الـ driver.
- بيتستدعى من `pinmux_func_name_to_selector()` عشان يعمل reverse lookup.

**Caller context:**
الـ pinmux core عند parsing الـ `pinctrl-*` properties في الـ device tree.

---

#### `get_function_groups`

```c
int (*get_function_groups)(struct pinctrl_dev *pctldev,
                            unsigned int selector,
                            const char * const **groups,
                            unsigned int *num_groups);
```

بيرجع مصفوفة بأسماء الـ pin groups اللي ممكن تستخدمها مع الـ function دي. كل group اسمها بيتحول لأرقام pins عن طريق الـ `pinctrl_ops->get_group_pins()`.

**Parameters:**
- `pctldev`: الـ controller handle.
- `selector`: الـ function index.
- `groups`: output pointer — الـ driver بيكتب فيه pointer لمصفوفة strings.
- `num_groups`: output — الـ driver بيكتب فيه عدد الـ groups.

**Return:**
- `0` عند النجاح.
- سالب عند الخطأ.

**Key details:**
- الـ `groups` array لازم تكون static في الـ driver — الـ core ما بيعمل copy.
- الـ core بيستخدم الـ group name عشان يجيب الـ pins الفعلية.

**Pseudocode flow:**
```
driver->get_function_groups(pctldev, selector, &groups, &ngroups)
  └─→ *groups = driver_function_table[selector].groups
  └─→ *ngroups = driver_function_table[selector].ngroups
  └─→ return 0
```

---

#### `function_is_gpio`

```c
bool (*function_is_gpio)(struct pinctrl_dev *pctldev,
                          unsigned int selector);
```

بيحدد إذا كانت الـ function دي هي الـ GPIO function. ده بيسمح للـ core إنه يستخدم الـ GPIO acceleration path (`gpio_request_enable` وأخواتها) بدل المسار العادي.

**Parameters:**
- `pctldev`: الـ controller handle.
- `selector`: الـ function index.

**Return:**
- `true` لو الـ function دي GPIO.
- `false` في الحالات التانية.

**Key details:**
- مفيد لما الـ controller بيستخدم `strict = true`، لأن الـ core محتاج يعرف إيه الـ GPIO function تحديداً.
- لو `NULL`، الـ core بيستخدم alternative logic (زي `PINFUNCTION_FLAG_GPIO`).
- الـ controllers اللي بتستخدم one-group-per-pin هي الأكثر استفادة منه.

---

### المجموعة الثالثة: Mux Activation

---

#### `set_mux`

```c
int (*set_mux)(struct pinctrl_dev *pctldev,
               unsigned int func_selector,
               unsigned int group_selector);
```

ده الـ callback الأهم — بيعمل الـ mux الفعلي على الـ hardware. الـ driver بيكتب في الـ registers اللي بتحدد الـ function الـ pins هتشتغل بيها.

**Parameters:**
- `pctldev`: الـ controller handle.
- `func_selector`: index الـ function المطلوب.
- `group_selector`: index الـ pin group المطلوب.

**Return:**
- `0` لو الـ mux اتضبط.
- سالب (مثلاً `-EIO`, `-EINVAL`) لو حصل error.

**Key details:**
- الـ core بيكون عمل `request()` على كل pin في الـ group قبل الاستدعاء ده.
- conflict detection بيحصل في الـ core قبل الاستدعاء — الـ driver مش محتاج يعمل ده.
- بعض الـ controllers بتتجاهل `group_selector` لو عندها function واحدة بـ group واحدة بس.
- على الـ driver يضبط الـ hardware registers atomically لو ممكن.

**Caller context:**
`pinmux_enable_setting()` في `drivers/pinctrl/core.c`، اللي بيتكال من `pinctrl_select_state()`.

**Pseudocode flow:**
```
pinctrl_select_state(p, state)
  └─→ pinmux_enable_setting(setting)
        ├─→ for each pin in group: ops->request(pin)
        └─→ ops->set_mux(pctldev, func_sel, group_sel)
              └─→ writel(mux_val, base + pin_reg[group_sel])
```

---

### المجموعة الرابعة: GPIO Acceleration Callbacks

الـ GPIO subsystem عادةً بيتعامل مع الـ pinmux عن طريق الـ `gpio_request()` / `gpio_free()`. عشان ما تبقاش overhead كبيرة من مسار الـ pinmux الكامل، الـ pinmux core بيوفر callbacks مخصوصة للـ GPIO.

---

#### `gpio_request_enable`

```c
int (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                            struct pinctrl_gpio_range *range,
                            unsigned int offset);
```

بيتستدعى لما الـ GPIO subsystem يطلب pin معين يبقى GPIO. الـ driver بيحوّل الـ pin لـ GPIO mode مباشرةً بدون المرور بمسار الـ function/group العادي.

**Parameters:**
- `pctldev`: الـ controller handle.
- `range`: الـ `pinctrl_gpio_range` اللي الـ pin ده واقع فيها — بتحتوي على الـ `base`, `pin_base`, `npins`, وoptional `gc`.
- `offset`: الـ pin number في الـ global pin space.

**Return:**
- `0` عند النجاح.
- سالب عند الفشل.

**Key details:**
- بيتستدعى من `pinmux_request_gpio()` في الـ core.
- الـ core بيعمل `request()` على الـ pin الأول.
- لازم implement ده لو الـ controller عايز يدعم الـ accelerated GPIO path.
- الـ `range` بتساعد الـ driver يعرف أي GPIO controller محتاج الـ pin ده.

---

#### `gpio_disable_free`

```c
void (*gpio_disable_free)(struct pinctrl_dev *pctldev,
                           struct pinctrl_gpio_range *range,
                           unsigned int offset);
```

عكس `gpio_request_enable`، بيتستدعى لما الـ GPIO بيتحرر. الـ driver بيرجع الـ pin لحالته الافتراضية أو يلغي الـ GPIO mode.

**Parameters:**
- `pctldev`: الـ controller handle.
- `range`: نفس الـ GPIO range.
- `offset`: رقم الـ pin.

**Return:**
- `void` — مش بيرجع حاجة، الـ errors بتتلوغ بس.

**Key details:**
- بيتستدعى من `pinmux_free_gpio()`.
- بعد الاستدعاء ده، الـ core بيكال `free()` على الـ pin.
- لازم يكون symmetric مع `gpio_request_enable`.

---

#### `gpio_set_direction`

```c
int (*gpio_set_direction)(struct pinctrl_dev *pctldev,
                           struct pinctrl_gpio_range *range,
                           unsigned int offset,
                           bool input);
```

بعض الـ controllers بتحتاج تغيّر الـ mux configuration لما الـ GPIO بيتحوّل من input لـ output والعكس. ده الـ callback اللي بيعمل ده.

**Parameters:**
- `pctldev`: الـ controller handle.
- `range`: الـ GPIO range.
- `offset`: رقم الـ pin.
- `input`: `true` لو محتاج input، `false` لو output.

**Return:**
- `0` عند النجاح.
- سالب عند الفشل.

**Key details:**
- بيتستدعى من `pinmux_gpio_direction()`.
- الـ GPIO core بيكاله من `gpiochip_direction_input()` و `gpiochip_direction_output()`.
- مش كل الـ controllers محتاجة ده — لو مش محتاج اتركه `NULL`.
- الـ controllers اللي عندها separate mux bits للـ direction (input enable / output enable) بتحتاجه.

**Caller context:**
```
gpio_direction_input(gpio)
  └─→ gpiochip_direction_input(chip, offset)
        └─→ pinmux_gpio_direction(pctldev, range, pin, input=true)
              └─→ ops->gpio_set_direction(pctldev, range, pin, true)
```

---

### الـ `strict` Field

```c
bool strict;
```

ده مش callback — ده flag بيخلي الـ core يرفض أي طلب يحاول يستخدم نفس الـ pin في نفس الوقت كـ GPIO وكـ mux function تانية.

**التأثير:**
- لو `true`: الـ core بيتحقق من `gpio_owner` و `mux_owner` قبل ما يقبل أي request جديد — لو الـ pin محجوز لأي منهم يرفض.
- لو `false` (default): ممكن نفس الـ pin يكون عند الاتنين في نفس الوقت لو الـ hardware يدعم ده.

**متى تستخدمه:**
الـ controllers اللي بتستخدم one-group-per-pin وعندها dedicated GPIO function — في الحالة دي `strict = true` بيكون منطقي ومفيد لتجنب الـ conflicts.

---

### الصورة الكاملة — كيف بيشتغل الـ pinmux core مع الـ vtable

```
Driver                          Pinmux Core
------                          -----------
struct pinmux_ops {             pinmux_enable_setting():
  .request        ←──────────────  ops->request(pin)  [for each pin]
  .get_func_count ←──────────────  ops->get_functions_count()
  .get_func_name  ←──────────────  ops->get_function_name(sel)
  .get_func_groups←──────────────  ops->get_function_groups(sel, ...)
  .set_mux        ←──────────────  ops->set_mux(func_sel, group_sel)
}

                                pinmux_request_gpio():
  .gpio_request_enable←────────── ops->gpio_request_enable(range, pin)
  .gpio_set_direction ←────────── ops->gpio_set_direction(range, pin, in)
  .gpio_disable_free  ←────────── ops->gpio_disable_free(range, pin)
```

---

### مثال عملي — Driver يملّي الـ vtable

```c
static const struct pinmux_ops foo_pmxops = {
    .request            = foo_pmx_request,
    .free               = foo_pmx_free,
    .get_functions_count = foo_get_functions_count,
    .get_function_name  = foo_get_function_name,
    .get_function_groups = foo_get_function_groups,
    .function_is_gpio   = foo_function_is_gpio,
    .set_mux            = foo_set_mux,
    .gpio_request_enable = foo_gpio_request_enable,
    .gpio_disable_free  = foo_gpio_disable_free,
    .gpio_set_direction = foo_gpio_set_direction,
    .strict             = true,   /* one-group-per-pin design */
};

static const struct pinctrl_desc foo_desc = {
    .name    = "foo-pinctrl",
    .pins    = foo_pins,
    .npins   = ARRAY_SIZE(foo_pins),
    .pctlops = &foo_pctlops,
    .pmxops  = &foo_pmxops,   /* هنا بتربط الـ vtable */
    .owner   = THIS_MODULE,
};

static int foo_probe(struct platform_device *pdev)
{
    struct pinctrl_dev *pctldev;
    return pinctrl_register_and_init(&foo_desc, &pdev->dev,
                                     NULL, &pctldev);
}
```

في `set_mux` مثلاً:

```c
static int foo_set_mux(struct pinctrl_dev *pctldev,
                        unsigned int func_sel,
                        unsigned int group_sel)
{
    struct foo_pinctrl *priv = pinctrl_dev_get_drvdata(pctldev);
    const struct foo_group *grp = &priv->groups[group_sel];
    unsigned int mux_val = priv->functions[func_sel].mux_val;

    /* write mux register for each pin in group */
    for (int i = 0; i < grp->npins; i++) {
        unsigned int pin = grp->pins[i];
        writel(mux_val, priv->base + FOO_MUX_REG(pin));
    }
    return 0;
}
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs

الـ pinmux subsystem بيعرض معلومات تفصيلية جوا `/sys/kernel/debug/pinctrl/`.

**المسارات الأساسية:**

```
/sys/kernel/debug/pinctrl/
├── <controller-name>/
│   ├── pins          ← كل الـ pins المسجلة مع أرقامها
│   ├── pingroups     ← الـ groups وأي pins فيها
│   ├── pinmux-pins   ← كل pin: owner (مين بيستخدمه) + function
│   ├── pinmux-functions ← كل function: اسمها + الـ groups المرتبطة بيها
│   └── pinconf-pins  ← إعدادات الـ pin config (pull, drive, etc.)
```

**طريقة القراءة:**

```bash
# اعرف كل الـ controllers المتاحة
ls /sys/kernel/debug/pinctrl/

# اعرف حالة كل pin: مين مستخدمه وإيه الـ function
cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-pins

# اعرف الـ functions وتوزيعها على الـ groups
cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-functions

# اعرف الـ groups وأرقام الـ pins الجوايها
cat /sys/kernel/debug/pinctrl/<ctrl>/pingroups
```

**مثال على output الـ `pinmux-pins`:**

```
pin 12 (GPIO12): device spi0.0 function spi group spi0-grp
pin 13 (GPIO13): UNCLAIMED
pin 14 (GPIO14): device uart0 function uart group uart0-grp
pin 15 (GPIO15): GPIO UNCLAIMED
```

- `UNCLAIMED` → الـ pin مش محجوز من حد.
- الـ `device` → الـ driver اللي طالب الـ pin.
- الـ `function` → الـ mux function المفعّلة على الـ pin.

---

#### 2. مدخلات الـ sysfs

```
/sys/bus/platform/drivers/pinctrl/
/sys/class/gpio/
```

```bash
# اعرف الـ GPIO chips المتاحة ومداها
cat /sys/kernel/debug/gpio

# تحقق من حالة GPIO معين (مستخدم أو لا)
cat /sys/class/gpio/gpio<N>/direction
cat /sys/class/gpio/gpio<N>/value
```

**ملاحظة:** الـ `gpio_request_enable` و `gpio_disable_free` في `pinmux_ops` بيتفاعلوا مع الـ GPIO subsystem، فمهم تشوف حالة الـ GPIO في sysfs لما تعمل debug على الـ GPIO mux.

---

#### 3. الـ ftrace — Tracepoints/Events

الـ pinctrl subsystem عنده tracepoints جوا `kernel/drivers/pinctrl/`:

```bash
# فعّل كل events الـ pinctrl
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# أو بشكل انتقائي
ls /sys/kernel/debug/tracing/events/pinctrl/

# ابدأ التتبع
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

**Events مهمة للـ pinmux تحديداً:**

| Event | ما يكشفه |
|---|---|
| `pinctrl_map_add` | لما بيتضاف mapping جديد |
| `pinctrl_state_select` | لما بيتغير state (default/sleep/idle) |
| `pinmux_enable_setting` | لما بيتفعّل mux setting |
| `pinmux_disable_setting` | لما بيتعطّل mux setting |

```bash
# فعّل function graph tracing على set_mux
echo 'pinmux_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

---

#### 4. الـ printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لكل الـ pinctrl subsystem
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control

# أو لملف محدد
echo 'file drivers/pinctrl/pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع line numbers وأسماء functions
echo 'file drivers/pinctrl/pinmux.c +pflm' > /sys/kernel/debug/dynamic_debug/control
```

**معنى الـ flags:**
- `p` → print
- `f` → اتضيف اسم الـ function في الرسالة
- `l` → اتضاف رقم السطر
- `m` → اتضاف اسم الـ module

**لتفعيل كل الـ printk من boot:**

```
# في kernel command line
dyndbg="file drivers/pinctrl/* +p"
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| Config | الغرض |
|---|---|
| `CONFIG_PINCTRL` | التفعيل الأساسي للـ subsystem |
| `CONFIG_DEBUG_PINCTRL` | يفعّل checks إضافية وـ warnings جوا الـ core |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | دعم الـ generic group management |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | دعم الـ generic function management |
| `CONFIG_PINMUX` | يفعّل الـ pinmux support أصلاً |
| `CONFIG_DEBUG_FS` | لازم يكون مفعّل عشان تشتغل مع `/sys/kernel/debug/pinctrl/` |
| `CONFIG_DYNAMIC_DEBUG` | عشان تستخدم الـ dynamic debug |
| `CONFIG_KALLSYMS` | يساعد في الـ stack traces |
| `CONFIG_PROVE_LOCKING` | يكشف لو في deadlock في الـ locks اللي بيستخدمها الـ pinctrl |
| `CONFIG_LOCKDEP` | تتبع الـ locking order |

---

#### 6. أدوات خاصة بالـ Subsystem

**`pinctrl` tool (من `libpinctrl` أو sysfs مباشرة):**

```bash
# مافيش userspace tool رسمي، بس ممكن تستخدم:

# اقرأ كل الـ pins وحالتها
for f in /sys/kernel/debug/pinctrl/*/pinmux-pins; do
    echo "=== $f ==="; cat "$f"; done

# ابحث عن pin معين (مثلاً pin 42)
grep -r "pin 42" /sys/kernel/debug/pinctrl/

# افحص لو في conflict على pin
grep -v UNCLAIMED /sys/kernel/debug/pinctrl/*/pinmux-pins | grep "pin <N>"
```

**`gpioinfo` و `gpioget` (من `libgpiod`):**

```bash
# اعرض كل الـ GPIO chips والـ lines
gpioinfo

# اقرأ حالة GPIO line معينة
gpioget <chip> <line>

# راقب الـ GPIO events
gpiomon <chip> <line>
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `pin X already requested` | الـ pin محجوز من driver تاني | افحص `pinmux-pins` في debugfs، تأكد من الـ DT mapping |
| `could not request pin X for device Y` | `request()` callback رجع error | افحص `dmesg` للتفاصيل، تحقق من الـ pin range في الـ driver |
| `pin-X (Y) status -16` | `-EBUSY`: الـ pin مستخدم | في conflict، شوف مين حاجز الـ pin |
| `could not get function 'X' for device 'Y'` | الـ function مش موجودة في الـ driver | تحقق من `pinmux-functions`، راجع الـ DT `function` property |
| `pin group X not registered` | الـ group الـ DT بيشاور عليها مش موجودة | راجع `pingroups` في debugfs، تأكد اسم الـ group صح في الـ DT |
| `could not set pins for group` | `set_mux()` فشل | افحص الـ hardware registers، راجع الـ driver implementation |
| `GPIO pin X is already used as GPIO` | في strict mode: الـ pin اتطلب كـ GPIO وكـ function | راجع لو `strict = true` في `pinmux_ops`، افحص الـ consumers |
| `pin X is not valid` | رقم الـ pin خارج النطاق | تحقق من `npins` في `pinctrl_desc` |
| `pinmux ops lacking` | الـ controller مش عنده `pmxops` | لازم تضيف `struct pinmux_ops` للـ `pinctrl_desc` |
| `pin X already used by Y` | محاولة استخدام pin محجوز في mode تاني | افحص الـ `mux_owner` و `gpio_owner` |

---

#### 8. Strategic Points للـ `dump_stack()` و `WARN_ON()`

**أهم النقاط اللي ممكن تحط فيها `WARN_ON()`:**

```c
/* في request() callback — تحقق إن الـ pin رقمه valid */
int my_pmx_request(struct pinctrl_dev *pctldev, unsigned int offset)
{
    struct my_pinctrl *priv = pinctrl_dev_get_drvdata(pctldev);
    WARN_ON(offset >= priv->npins);  /* pin out of range */
    /* ... */
}

/* في set_mux() — تحقق إن الـ selectors في النطاق */
int my_pmx_set_mux(struct pinctrl_dev *pctldev,
                   unsigned int func_selector,
                   unsigned int group_selector)
{
    WARN_ON(func_selector >= MY_NUM_FUNCTIONS);
    WARN_ON(group_selector >= MY_NUM_GROUPS);
    /* ... */
}

/* في gpio_request_enable() — تحقق إن الـ range مش NULL */
int my_gpio_request_enable(struct pinctrl_dev *pctldev,
                           struct pinctrl_gpio_range *range,
                           unsigned int offset)
{
    if (WARN_ON(!range))
        return -EINVAL;
    /* ... */
}
```

**نقاط مناسبة لـ `dump_stack()`:**

```c
/* لو set_mux() اتنادى وانت مش متوقع — خصوصاً في conflict cases */
if (unlikely(already_configured)) {
    dev_err(pctldev->dev, "unexpected set_mux call on pin %u\n", offset);
    dump_stack();  /* يساعدك تعرف مين استدعى الـ mux تاني مرة */
}
```

---

### Hardware Level

---

#### 1. تحقق إن حالة الـ Hardware بتطابق الـ Kernel State

**المبدأ:** اللي بيسجله الـ kernel في debugfs لازم يتطابق مع قيم الـ registers الفعلية.

```bash
# خطوة 1: اقرأ الـ mux state من debugfs
cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-pins

# خطوة 2: اقرأ الـ hardware register المقابل (مثلاً IOMUXC_SW_MUX_CTL_PAD_GPIO1)
devmem2 <register_address> w

# خطوة 3: قارن القيمة بـ expected value من الـ datasheet
# لو مختلفين → في مشكلة في set_mux() أو في reset لم يتم properly
```

---

#### 2. Register Dump Techniques

```bash
# devmem2 — اقرأ register واحد
devmem2 0x30330000 w    # مثال: IOMUXC base على i.MX8

# اقرأ range كاملة من الـ registers (32 register بداية من base)
BASE=0x30330000
for i in $(seq 0 4 124); do
    ADDR=$(printf "0x%x" $((BASE + i)))
    VAL=$(devmem2 $ADDR w 2>/dev/null | grep "Read" | awk '{print $NF}')
    echo "[$ADDR] = $VAL"
done

# /dev/mem مع dd (لو devmem2 مش متاح)
dd if=/dev/mem bs=4 count=1 skip=$((0x30330000/4)) 2>/dev/null | xxd
```

**ملاحظة:** لازم `CONFIG_DEVMEM=y` وصلاحيات root.

```bash
# io tool (من package io أو buildroot)
io -4 -r 0x30330000    # قراءة 32-bit
io -4 -w 0x30330000 0x5  # كتابة (احذر!)
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**للـ pinmux debugging على مستوى الـ signal:**

- **Logic Analyzer:** اربط على الـ pin اللي بتعمله mux وتحقق:
  - هل الـ signal يتطابق مع الـ function المختارة (مثلاً SPI clock pattern بدل GPIO static)؟
  - هل في glitch وقت الـ mux switching (لازم يكون smooth)?
  - راجع الـ timing مقارنةً بـ `set_mux()` استدعاء في الـ kernel log.

- **Oscilloscope:**
  - افحص الـ voltage levels: بعض الـ mux configurations بتغير الـ drive strength.
  - افحص لو في floating pins (مش محددين pull-up/down) بيسببوا noise.

- **عملياً:**
  - استخدم `gpiomon` لتتبع GPIO events قبل ما تربط الـ analyzer.
  - فعّل `ftrace` على `set_mux` وقارن الـ timestamp بالـ signal على الـ analyzer.

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة | Pattern في الـ Kernel Log |
|---|---|
| الـ pin مش بيتحول لـ alternate function | `set_mux called but register not updated` (من driver custom printk) |
| الـ GPIO مش بيشتغل بعد mux | `gpio_request_enable: pin X in use by function Y` |
| الـ SPI/I2C مش بيشتغل بعد pinmux | `failed to request pin X for spi` + timeout errors |
| تعارض في الـ ownership | `pin X already requested by Y, cannot be requested by Z` |
| Boot hang بسبب pinmux | آخر رسالة في dmesg: `Applying default state for device` |
| الـ pin بيرجع لحالته القديمة بعد suspend | مفيش `pinctrl_pm_select_sleep_state` في الـ driver |

---

#### 5. Device Tree Debugging

**تحقق إن الـ DT بيتطابق مع الـ Hardware:**

```bash
# اعرض الـ DT المحمّل فعلياً (الـ live DT)
ls /proc/device-tree/
cat /proc/device-tree/soc/pinctrl@30330000/compatible

# اعرض الـ DT بصيغة readable
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "pinctrl"

# افحص الـ pinctrl node للـ device
cat /proc/device-tree/<device-path>/pinctrl-0
cat /proc/device-tree/<device-path>/pinctrl-names
```

**أكثر أسباب مشاكل الـ DT شيوعاً:**

```dts
/* خطأ شائع: اسم الـ function غلط */
&pinctrl {
    uart0_pins: uart0-pins {
        pinmux = <MX8MP_IOMUXC_UART1_RXD__UART1_DCE_RX>;
        /* لو الـ function name مش متطابق مع get_function_name() → فشل */
    };
};

/* تحقق: */
/* cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-functions */
/* اسم الـ function في DT لازم يتطابق بالضبط */
```

```bash
# افحص لو الـ pinctrl state اتطبق على الـ device
grep -r "pinctrl" /sys/bus/platform/devices/<device>/

# تحقق من الـ mapping اللي اتسجل
cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-pins | grep <device-name>
```

---

### Practical Commands

---

#### مجموعة الأوامر الجاهزة

**1. Overview سريع للـ pinmux state:**

```bash
#!/bin/bash
# pinmux-overview.sh — يعرض كل الـ controllers وحالة الـ pins

PINCTRL_BASE=/sys/kernel/debug/pinctrl
for CTRL in "$PINCTRL_BASE"/*/; do
    echo "=========================================="
    echo "Controller: $(basename "$CTRL")"
    echo "=========================================="
    echo "--- Functions ---"
    cat "${CTRL}pinmux-functions" 2>/dev/null
    echo ""
    echo "--- Pin Mux Status ---"
    cat "${CTRL}pinmux-pins" 2>/dev/null
    echo ""
done
```

**2. البحث عن pin معين:**

```bash
PIN=42
grep -r "pin $PIN" /sys/kernel/debug/pinctrl/*/pinmux-pins
```

**Example output:**
```
/sys/kernel/debug/pinctrl/30330000.pinctrl/pinmux-pins:
pin 42 (MX8MP_PAD_GPIO1_IO06): device uart0 function uart group uart0-grp
```
التفسير: الـ pin 42 محجوز من `uart0`، مفعّل عليه function `uart`، جزء من group `uart0-grp`.

---

**3. إيجاد الـ pins الـ unclaimed:**

```bash
grep "UNCLAIMED" /sys/kernel/debug/pinctrl/*/pinmux-pins
```

---

**4. تفعيل dynamic debug للـ pinmux وقت الـ runtime:**

```bash
# فعّل debug messages لكل ملفات pinctrl
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control

# راقب الـ output في real-time
dmesg -w | grep -i pinctrl
```

---

**5. تتبع set_mux calls بالـ ftrace:**

```bash
# نظّف الـ trace السابق
echo > /sys/kernel/debug/tracing/trace

# فعّل تتبع الـ function
echo 'pinmux_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل العملية اللي بتشتبه فيها (مثلاً: probe device)
echo "my_device" > /sys/bus/platform/drivers/my_driver/bind

# اقرأ النتيجة
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

**Example output:**
```
kworker/0:1-12  [000] ....  123.456789: pinmux_enable_setting <-pinctrl_commit_state
kworker/0:1-12  [000] ....  123.456800: my_pmx_set_mux <-pinmux_enable_setting
```
التفسير: شوف الـ call chain — من `pinctrl_commit_state` لـ `pinmux_enable_setting` لـ driver callback.

---

**6. مقارنة الـ hardware registers مع الـ expected state:**

```bash
# مثال على i.MX8MP: IOMUXC_SW_MUX_CTL_PAD_GPIO1_IO06
# Base: 0x30330000, Offset من الـ datasheet
REG_ADDR=0x303300CC  # GPIO1_IO06 MUX register
devmem2 $REG_ADDR w

# Expected output (UART function = ALT0 = 0x0):
# Read at address 0x303300CC (0xfffffffc303300cc): 0x00000000

# لو القيمة غير متوقعة → set_mux() مش بيكتب صح في الـ register
```

---

**7. افحص الـ gpio conflicts مع الـ pinmux:**

```bash
# اعرض كل الـ GPIO المستخدمة وأصحابها
cat /sys/kernel/debug/gpio

# قارن مع الـ pinmux-pins
diff <(grep -v UNCLAIMED /sys/kernel/debug/pinctrl/*/pinmux-pins | grep GPIO) \
     <(cat /sys/kernel/debug/gpio | grep "gpio-")
```

---

**8. تحقق من الـ pinctrl states للـ device:**

```bash
DEVICE="30a30000.spi"  # مثال

# اعرف الـ states المتاحة
find /proc/device-tree -path "*${DEVICE}*" -name "pinctrl-names" \
     -exec strings {} \;

# اعرف الـ state الحالية
grep "$DEVICE" /sys/kernel/debug/pinctrl/*/pinmux-pins
```

---

**9. Boot-time debugging — kernel cmdline:**

```bash
# أضف في /boot/cmdline.txt أو GRUB:
# dyndbg="file drivers/pinctrl/pinmux.c +p;file drivers/pinctrl/core.c +p"
# loglevel=8

# أو عن طريق sysctl بعد البوت:
sysctl -w kernel.printk="8 4 1 7"
```

---

**10. تحقق من strict mode conflicts:**

```bash
# لو الـ driver عنده strict=true في pinmux_ops
# هيظهر error لو نفس الـ pin اتطلب كـ GPIO وكـ mux function

# ابحث عن الـ error:
dmesg | grep -i "strict\|gpio.*owner\|mux.*owner\|already.*gpio\|already.*mux"
```

**Example output:**
```
pinctrl-bcm2835 fe200000.gpio: pin gpio4 already used as GPIO by pinctrl-bcm2835:gpio4
```
التفسير: في تعارض بين الـ GPIO consumer والـ mux consumer على نفس الـ pin — راجع الـ DT أو الـ driver اللي بيطلب الـ GPIO.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART مش شغال على RK3562 — Industrial Gateway

#### العنوان
**الـ UART2 مش بيظهر على `/dev/ttyS2` بعد bring-up الـ RK3562**

#### السياق
فريق بيشتغل على industrial gateway بيستخدم **RK3562** كـ main SoC. الـ gateway بيتكلم مع Modbus RTU devices عبر UART2. المنتج وصل مرحلة الـ DVT وكل حاجة تانية شغالة، بس الـ UART2 مش بيظهر في `/dev/` خالص.

#### المشكلة
الـ `dmesg` مفيش فيه error واضح. الـ device node موجود في DT، الـ `uart2` enabled، بس الـ driver مش بيـ probe. الـ engineer يشوف:

```bash
$ ls /dev/ttyS*
/dev/ttyS0  /dev/ttyS1
# ttyS2 غايب
```

#### التحليل
الـ engineer يبدأ يفحص الـ pinmux subsystem عبر debugfs:

```bash
$ cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinmux-pins
pin 24 (GPIO0_C0): uart2 GPIO UNCLAIMED
pin 25 (GPIO0_C1): uart2 GPIO UNCLAIMED
```

الـ pins هنا `UNCLAIMED` — معناها إن الـ `request()` callback في `struct pinmux_ops` اترفض أو ما اتكلمش أصلاً.

الكود ده جوه `pinmux.h` بيوضح إن الـ core بيكلم `request()` الأول:

```c
/* core calls this before set_mux() to acquire the pin */
int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);
```

الـ engineer يفحص الـ DT:

```dts
/* غلط — الـ pinctrl-0 بيشاور على group مش موجود */
&uart2 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&uart2m1_xfer>;  /* ← هنا المشكلة */
};
```

الـ RK3562 عنده multiplexing modes: `uart2m0` و`uart2m1`. الـ `uart2m1_xfer` group موجود في الـ DT source الخاص بـ Rockchip، بس الـ board DT فيها `uart2m0_xfer` على pins اتعملهالها conflict مع I2C3.

الـ pinmux core لما بيـ call الـ `get_function_groups()`:

```c
int (*get_function_groups)(struct pinctrl_dev *pctldev,
                           unsigned int selector,
                           const char * const **groups,
                           unsigned int *num_groups);
```

بيرجع groups متاحة، والـ `set_mux()` بيتفشل لأن الـ pins متعملهالهم `request()` من driver تاني (I2C3).

#### الحل

```dts
/* الحل: استخدام uart2m1 اللي على pins مفيش عليها conflict */
&uart2 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&uart2m1_xfer>;
};

/* وتأكد إن I2C3 مش بيستخدم نفس الـ pins */
&i2c3 {
    status = "disabled";
};
```

التحقق بعد الحل:

```bash
$ cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinmux-pins | grep uart2
pin 32 (GPIO0_D0): UNCLAIMED (GPIO UNCLAIMED)
pin 33 (GPIO0_D1): device uart2 function uart2 group uart2m1-xfer
```

#### الدرس المستفاد
الـ `request()` callback في `pinmux_ops` هو أول خط دفاع ضد pin conflicts. لو pin اتطلب من driver تاني، الـ core برفض الـ request التاني تلقائياً. دايماً افحص `pinmux-pins` في debugfs قبل ما تدور على bugs في الـ UART driver نفسه.

---

### السيناريو 2: SPI Flash مش بيتعرف على AM62x — IoT Sensor Hub

#### العنوان
**الـ SPI NOR Flash مش بيظهر على AM62x بعد إضافة GPIO expander**

#### السياق
فريق بيطور IoT sensor hub على **AM62x** (Texas Instruments). البورد بتستخدم SPI0 مع W25Q128 NOR flash للـ firmware storage. كل حاجة كانت شغالة، وبعد ما اتضافت GPIO expander متصلة بـ I2C1، الـ SPI flash اختفت من `/dev/mtd*`.

#### المشكلة
```bash
$ dmesg | grep spi
spi-nor spi0.0: unrecognized JEDEC id bytes: ff ff ff
# الـ flash مش بترد — CS line مش شغالة
```

#### التحليل
الـ engineer يفحص الـ pinmux state:

```bash
$ cat /sys/kernel/debug/pinctrl/pinctrl-am62x-pinctrl/pinmux-pins | grep spi0
pin 118 (SPI0_CS0): UNCLAIMED
```

الـ CS0 pin `UNCLAIMED`! الـ engineer يفحص الـ `strict` field في `pinmux_ops`:

```c
/* من pinmux.h */
bool strict;
/* لو true: الـ core يرفض أي pin اتطلب كـ GPIO ويتطلب كـ mux function في نفس الوقت */
```

الـ GPIO expander driver لما اتضاف، طلب GPIO expander interrupt line على نفس الـ pin اللي الـ SPI CS0 بيستخدمه، لأن الـ board designer ربطهم بغلطة على الـ PCB ورخّص بالـ DT.

الـ core يمر بالـ flow ده:

```
gpio_request_enable() →
    pinmux_ops.request(pctldev, CS0_pin) →
        returns 0 (GPIO driver يكسب أول)
set_mux() for SPI →
    pinmux_ops.request(pctldev, CS0_pin) →
        returns -EBUSY (pin already owned by GPIO)
```

لأن الـ `strict = true` في AM62x pinctrl driver، الـ core يرفض تماماً أي overlap بين GPIO ownership وـ mux ownership.

#### الحل

أولاً، تأكيد المشكلة:

```bash
$ cat /sys/kernel/debug/pinctrl/pinctrl-am62x-pinctrl/gpio-ranges
GPIO chip: gpio0
  range 0: [0..31] ==> [0..31] on pinctrl-am62x
  ...
$ cat /sys/kernel/debug/gpio | grep "gpio 118"
gpio-118 (SPI0_CS0        |gpio-expander-irq ) in  lo
```

الحل في DT — نقل interrupt line للـ GPIO expander على pin تاني:

```dts
/* غلط */
&gpio_expander {
    interrupt-parent = <&main_gpio0>;
    interrupts = <118 IRQ_TYPE_EDGE_FALLING>;  /* نفس SPI CS0! */
};

/* صح — استخدام pin بديل */
&gpio_expander {
    interrupt-parent = <&main_gpio0>;
    interrupts = <45 IRQ_TYPE_EDGE_FALLING>;   /* pin منفصل */
    pinctrl-names = "default";
    pinctrl-0 = <&expander_irq_pins>;
};
```

#### الدرس المستفاد
الـ `strict` flag في `pinmux_ops` هو حارس صارم: pin واحد مش ممكن يبقى GPIO وـ mux function في نفس الوقت. لما بتضيف hardware جديدة على board موجودة، افحص الـ GPIO ranges وـ pin assignments في debugfs قبل ما تـ boot.

---

### السيناريو 3: HDMI مش شغال على Allwinner H616 — Android TV Box

#### العنوان
**شاشة HDMI سودا على Allwinner H616 TV Box بعد تحديث الـ kernel**

#### السياق
شركة بتصنع Android TV box بالـ **Allwinner H616**. بعد ترقية الـ kernel من 5.15 لـ 6.1، الـ HDMI output اتوقف خالص والشاشة سودا. الـ USB والـ WiFi لسه شغالين.

#### المشكلة
```bash
$ dmesg | grep hdmi
sunxi-hdmi display-engine: failed to enable pinctrl state default: -22
```

الـ `-22` ده `EINVAL` — معناه إن الـ pinmux state اتطلبت بس اترفضت.

#### التحليل
الـ engineer يفحص الـ `get_function_groups()` callback:

```c
int (*get_function_groups)(struct pinctrl_dev *pctldev,
                           unsigned int selector,
                           const char * const **groups,
                           unsigned int *num_groups);
```

في kernel 6.1، الـ Allwinner pinctrl driver اتعمله refactor وبعض الـ function names اتغيرت. الـ group اللي الـ HDMI driver بيطلبه بـ اسمه القديم مش موجود في الـ `get_function_name()` output:

```bash
$ cat /sys/kernel/debug/pinctrl/pinctrl-sunxi/pinmux-functions
...
function 42: hdmi_old      # القديم — اتشال
function 43: hdmi          # الجديد في 6.1
...
```

الـ `get_functions_count()` بيرجع العدد الصح، بس الـ DT binding بتشاور على `hdmi_old` اللي ما بقاش موجود. الـ core لما بيـ call الـ `set_mux()`:

```c
int (*set_mux)(struct pinctrl_dev *pctldev,
               unsigned int func_selector,
               unsigned int group_selector);
```

الـ `func_selector` بيبقى invalid (خارج حدود array) → `EINVAL`.

#### الحل

تحديث الـ DT لاستخدام الـ function name الجديد:

```dts
/* pinctrl node في sun50i-h616.dtsi — قبل */
hdmi_pins: hdmi-pins {
    pins = "PH0", "PH1";
    function = "hdmi_old";    /* ← القديم */
};

/* بعد */
hdmi_pins: hdmi-pins {
    pins = "PH0", "PH1";
    function = "hdmi";        /* ← الجديد في kernel 6.1 */
};
```

للتحقق إن الـ function موجود:

```bash
$ cat /sys/kernel/debug/pinctrl/pinctrl-sunxi/pinmux-functions | grep hdmi
function 43: hdmi
  group: PH0-PH1
```

ثم:
```bash
$ cat /sys/kernel/debug/pinctrl/pinctrl-sunxi/pinmux-pins | grep hdmi
pin 224 (PH0): device hdmi function hdmi group PH0-PH1
pin 225 (PH1): device hdmi function hdmi group PH0-PH1
```

#### الدرس المستفاد
الـ `get_function_name()` callback هو الـ source of truth لأسماء الـ functions. لو الـ kernel driver اتعمله refactor، أسماء الـ functions ممكن تتغير وتـ break الـ DT bindings القديمة. دايماً راجع `pinmux-functions` في debugfs بعد أي kernel upgrade.

---

### السيناريو 4: I2C بياخد -EBUSY على STM32MP1 — Automotive ECU

#### العنوان
**الـ I2C3 على STM32MP1 بيفشل في الـ probe بـ `-EBUSY` على Automotive ECU**

#### السياق
فريق بيشتغل على automotive ECU بيستخدم **STM32MP157** للـ body control module. الـ ECU بيستخدم I2C3 للتواصل مع temperature sensors. على بورد development بتشتغل، لكن على الـ production PCB بتفشل.

#### المشكلة
```bash
$ dmesg | grep i2c3
i2c i2c-3: Failed to request pin 67: -16
# EBUSY = -16
```

Pin 67 مطلوب من حاجة تانية.

#### التحليل
الـ `request()` callback في `pinmux_ops` بيمنع double-use:

```c
int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);
/* بيرجع -EBUSY لو pin اتطلب قبل كده */
```

الـ engineer يفحص:

```bash
$ cat /sys/kernel/debug/pinctrl/soc:pin-controller/pinmux-pins | grep "pin 67"
pin 67 (PD3): device can1 function can1 group can1-grp
```

**CAN1**! على الـ production PCB، الـ layout engineer ربط PD3 في الـ schematic كـ CAN1_TX بس هو كمان I2C3_SDA في الـ MCU pinout. الـ dev board كانت بتستخدم I2C3 على pins تانية (I2C3_ALT_PINS group).

الـ flow في الـ kernel:

```
CAN1 probe → pinmux_ops.request(pctldev, pin=67) → OK, owned by CAN1
I2C3 probe → pinmux_ops.request(pctldev, pin=67) → -EBUSY
```

الـ engineer يفحص الـ DT board file:

```dts
/* production board DT — مفيش override للـ I2C3 pins */
&i2c3 {
    status = "okay";
    /* بيستخدم default pins = PD3/PD4 اللي على CAN1 */
};

&can1 {
    status = "okay";
    pinctrl-0 = <&can1_pins_a>;  /* PD3 = CAN1_TX */
};
```

#### الحل

استخدام الـ alternate pin group للـ I2C3:

```dts
/* الـ STM32MP1 عنده I2C3 alt pins على PZ4/PZ5 */
i2c3_pins_alt: i2c3-alt-pins {
    pins1 {
        pinmux = <STM32_PINMUX('Z', 4, AF6)>;  /* I2C3_SCL */
        bias-disable;
        drive-open-drain;
        slew-rate = <0>;
    };
    pins2 {
        pinmux = <STM32_PINMUX('Z', 5, AF6)>;  /* I2C3_SDA */
        bias-disable;
        drive-open-drain;
        slew-rate = <0>;
    };
};

&i2c3 {
    status = "okay";
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&i2c3_pins_alt>;    /* ← alternate pins */
    pinctrl-1 = <&i2c3_sleep_alt>;
};
```

#### الدرس المستفاد
الـ `request()` callback في `pinmux_ops` بيكشف conflict مش ممكن يتكتشف غير بالـ runtime. في الـ automotive context، الـ production PCB ممكن تختلف عن الـ dev board في pin assignments. دايماً ضيف `pinmux-pins` dump في الـ bring-up checklist.

---

### السيناريو 5: GPIO مش بيشتغل كـ output على i.MX8MP — Custom Board Bring-up

#### العنوان
**GPIO مش بيقدر يتحول لـ output على i.MX8MP بسبب `strict` mode**

#### السياق
engineer بيعمل bring-up لـ custom board بيستخدم **i.MX8MP** (NXP). البورد عندها LED متصل بـ GPIO4_IO25. الـ engineer كاتب driver بسيط يضيء الـ LED، بس الـ `gpio_direction_output()` بتفشل.

#### المشكلة
```bash
$ echo 1 > /sys/class/gpio/gpio121/direction
bash: echo: write error: Device or resource busy
```

#### التحليل
الـ engineer يبدأ بالـ debugfs:

```bash
$ cat /sys/kernel/debug/pinctrl/pinctrl-imx8mp/pinmux-pins | grep "pin 121"
pin 121 (GPIO4_IO25): device uart4 function uart4 group uart4grp
```

الـ pin ده متخصص لـ UART4! بس الـ engineer مش عارف — الـ UART4 في DT `disabled` في الـ board file.

الـ flow الفعلي:

```
UART4 disabled → pinctrl state مش اتطلب → pin مش owned by UART4
GPIO request لـ pin 121 →
    gpio_request_enable() في pinmux_ops يتكلم →
    لكن iomux-v3 driver بيلاقي إن function selector للـ pin ده
    اتعمله set_mux() من bootloader لـ UART4 function!
```

الـ bootloader (U-Boot) عمل `set_mux()` لكل الـ UART pins في early init، حتى اللي disabled في الـ kernel DT. الـ kernel pinctrl core بيشوف الـ mux register قايل "UART4" وبيرفض الـ GPIO request لأن `strict = true` في الـ iMX8MP pinctrl driver:

```c
/* من pinmux.h */
bool strict;
/* Check both gpio_owner and mux_owner strictly before approving the pin request */
```

#### الحل

**الحل 1**: تعطيل الـ UART4 function في U-Boot أو تصفير الـ mux register قبل الـ kernel boot.

**الحل 2** (الأفضل): إضافة الـ GPIO4_IO25 كـ GPIO-only pin في DT وتأكيد عدم overlap مع أي function:

```dts
/* إضافة pinctrl group للـ LED */
pinctrl_led: ledgrp {
    fsl,pins = <
        /* GPIO4_IO25: GPIO mode = 0x5 في IMX8MP */
        MX8MP_IOMUXC_SAI2_TXFS__GPIO4_IO25  0x19
    >;
};

/* اللي اتغير: مفيش UART4 في DT + إضافة pinctrl للـ GPIO */
&gpio4 {
    led-gpio {
        gpio-hog;
        gpios = <25 GPIO_ACTIVE_HIGH>;
        output-high;
        line-name = "status-led";
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_led>;
    };
};
```

**الحل 3**: لو مش قادر تعدل U-Boot، force clear الـ mux register في early kernel:

```bash
# تحقق من قيمة الـ IOMUXC register
$ devmem2 0x30330150 w
# لو مش 0x5 (GPIO mode)، الـ bootloader حدده غلط
```

للتحقق النهائي بعد الحل:

```bash
$ cat /sys/kernel/debug/pinctrl/pinctrl-imx8mp/pinmux-pins | grep "pin 121"
pin 121 (GPIO4_IO25): GPIO CLAIMED by gpio4:25

$ echo 1 > /sys/class/gpio/gpio121/direction && echo 1 > /sys/class/gpio/gpio121/value
# LED يضيء
```

#### الدرس المستفاد
الـ `gpio_request_enable()` و`gpio_set_direction()` callbacks في `pinmux_ops` بيشتغلوا تحت `strict` mode — يعني الـ bootloader state بيأثر على الـ kernel pin ownership. دايماً افحص الـ mux registers الفعلية في hardware مش بس الـ DT، وابدأ الـ bring-up بـ `pinmux-pins` dump كأول خطوة.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي غطّت نشأة وتطور subsystem الـ pinctrl/pinmux في Linux kernel:

| المقال | الأهمية |
|--------|---------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | شرح شامل للـ pinctrl كـ superset فوق الـ pinmux والـ pinconf — أهم مقال للبداية |
| [drivers: create a pinmux subsystem v3](https://lwn.net/Articles/447394/) | الـ patch series الأصلي اللي أضاف الـ pinmux subsystem من Linus Walleij |
| [drivers: create a pinmux subsystem v2](https://lwn.net/Articles/442315/) | نسخة سابقة من الـ patch توضّح تطور التصميم |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | الـ documentation الأصلية المرفقة مع الـ subsystem |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | إضافة الـ `pinconf_ops` — التوسعة الطبيعية بعد الـ pinmux |
| [pinctrl: introduce the concept of a GPIO pin function category](https://lwn.net/Articles/1029298/) | مقترح إدخال مفهوم "GPIO function category" للـ strict pinmuxers |
| [pinctrl: add a driver for NVIDIA Tegra](https://lwn.net/Articles/478801/) | مثال real-world لأول driver كامل على الـ subsystem |

---

### التوثيق الرسمي في الـ kernel

```
Documentation/driver-api/pinctl.rst          # الوثيقة الرئيسية للـ subsystem كامل
Documentation/devicetree/bindings/pinctrl/   # bindings الـ device tree للـ pinctrl
```

**الـ online version:**
- [PINCTRL subsystem — kernel.org docs (v4.14)](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html)
- [Documentation/pinctrl.txt — kernel.org](https://www.kernel.org/doc/Documentation/pinctrl.txt)

---

### الكود المصدري المهم

الملفات الجوهرية اللي لازم تقراها جنب الـ header:

```
include/linux/pinctrl/pinmux.h     # الـ header موضوع الدراسة — struct pinmux_ops
drivers/pinctrl/pinmux.c           # الـ core implementation للـ pinmux
drivers/pinctrl/core.c             # الـ pinctrl core العام
drivers/pinctrl/core.h             # internal structs الـ core
```

**على GitHub (torvalds/linux):**
- [drivers/pinctrl/pinmux.c](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/pinmux.c)
- [drivers/pinctrl/pinmux.h](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/pinmux.h)

---

### Commits مهمة

| الحدث | المصدر |
|-------|--------|
| إنشاء الـ pinmux subsystem (2011) — Linus Walleij عن ST-Ericsson/Linaro | [LWN patch v3](https://lwn.net/Articles/447394/) |
| إضافة `strict` mode وحماية التعارض بين GPIO والـ mux functions | `git log --oneline drivers/pinctrl/pinmux.c` |
| إضافة `function_is_gpio` callback | مرتبطة بالـ strict mode |
| [GIT PULL] pin control changes for v6.19 — Linus Walleij](https://lkml.org/lkml/2025/12/8/736) | آخر دورة تطوير |

للبحث عن commits تاريخية:
```bash
git log --oneline --follow drivers/pinctrl/pinmux.c
git log --oneline --follow include/linux/pinctrl/pinmux.h
```

---

### نقاشات Mailing List

- [Re: PATCH v5 — pinctrl: introduce GPIO pin function category](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10536.html) — نقاش تقني على الـ `function_is_gpio` callback والـ strict mode
- [Linus Walleij على LKML — pinctrl generic board-level mux chips](https://lkml.org/lkml/2026/2/20/43) — مراجعة patches جديدة
- [lore.kernel.org — pinctrl](https://lore.kernel.org/linux-gpio/) — الـ mailing list الرسمي للـ GPIO/pinctrl (linux-gpio@vger.kernel.org)

للمتابعة المباشرة:
```
Mailing list: linux-gpio@vger.kernel.org
Maintainer: Linus Walleij <linus.walleij@linaro.org>
Archive: https://lore.kernel.org/linux-gpio/
```

---

### eLinux.org

- [Introduction to pin muxing and GPIO control under Linux (ELC 2021 PDF)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) — عرض تقديمي عملي ممتاز
- [EBC Device Trees — pinctrl examples](https://elinux.org/EBC_Device_Trees) — أمثلة device tree overlay للـ pinctrl
- [Jetson/AGX Xavier Check Pin Setting](https://www.elinux.org/Jetson/AGX_Xavier_Check_Pin_Setting) — debugfs + pinmux registers على hardware حقيقي
- [Tests: i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) — اختبار عملي للـ i2c mux فوق الـ pinctrl API

---

### KernelNewbies.org

صفحات الـ release notes بتحتوي على تغييرات الـ pinctrl في كل kernel version:

- [Linux 6.11 — pinctrl changes](https://kernelnewbies.org/Linux_6.11)
- [Linux 6.13 — pinctrl changes](https://kernelnewbies.org/Linux_6.13)
- [Linux 6.14 — pinctrl changes](https://kernelnewbies.org/Linux_6.14)
- [Linux 6.15 — pinctrl changes](https://kernelnewbies.org/Linux_6.15)
- [Linux 6.17 — pinctrl changes](https://kernelnewbies.org/Linux_6.17)
- [LinuxChanges — تاريخي شامل](https://kernelnewbies.org/LinuxChanges)

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 1**: نظرة عامة على الـ kernel subsystems
- **الفصل 14**: The Linux Device Model — الـ bus/device/driver model اللي الـ pinctrl بيبني عليه
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Ed.)
- **الفصل 1**: Introduction to the Linux Kernel
- **الفصل 17**: Devices and Modules — فهم الـ driver registration اللي الـ `pinctrl_register()` بيتبعها
- مفيد لفهم الـ kobject والـ sysfs اللي الـ pinctrl بيظهر فيهم

#### Embedded Linux Primer — Christopher Hallinan (2nd Ed.)
- **الفصل 15**: Debugging Embedded Linux Applications — بيغطي الـ GPIO والـ pin configuration على الـ embedded boards
- **الفصل 7**: Bootloader Fundamentals — علاقة الـ pin muxing بالـ bootloader وتهيئة الـ hardware

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- بيشرح الـ device model بعمق — أساسي لفهم كيف الـ `pinctrl_dev` بيتسجّل

---

### مصطلحات البحث

للبحث عن معلومات إضافية استخدم الكلمات دي:

```
"pinmux_ops" site:elixir.bootlin.com
"struct pinmux_ops" linux kernel
pinctrl "set_mux" driver example
pinctrl "gpio_request_enable" implementation
pinctrl strict mode conflict resolution
linux pinmux function group selector
"pinctrl_dev" "pinmux_ops" callback registration
linux pinctrl device tree bindings example
pinctrl debugfs /sys/kernel/debug/pinctrl
```

**أدوات مفيدة للاستكشاف:**

| الأداة | الرابط | الاستخدام |
|--------|--------|-----------|
| Elixir Cross Referencer | [elixir.bootlin.com](https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/pinmux.h) | تتبع الـ struct/callback عبر الكود كله |
| Bootlin Elixir | [elixir.bootlin.com/linux/latest](https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl) | قراءة كل drivers الـ pinctrl |
| lore.kernel.org | [lore.kernel.org/linux-gpio](https://lore.kernel.org/linux-gpio/) | نقاشات الـ mailing list |

---

### مثال driver حقيقي للمراجعة

أبسط driver يوضح `pinmux_ops` كامل:

```
drivers/pinctrl/pinctrl-single.c    # generic single-register pinmux driver
drivers/pinctrl/tegra/pinctrl-tegra.c  # Tegra — من أوائل الـ drivers
drivers/pinctrl/qcom/pinctrl-msm.c  # Qualcomm — أكثر شيوعًا في الـ production
```

```bash
# شوف كل implementors لـ struct pinmux_ops في الكود:
grep -r "pinmux_ops" drivers/pinctrl/ --include="*.c" -l
```
## Phase 8: Writing simple module

### الفكرة

**الـ** `set_mux` هي أكتر function مثيرة للاهتمام في `pinmux_ops` — هي اللي بتتنفذ فعلاً لما الـ kernel بيغير mux setting على pin معين. هنعمل **kprobe** على `pinmux_set_mux` (الـ internal core function اللي بتكال `pmxops->set_mux`) عشان نشوف كل مرة بيتعمل فيها mux switch على أي pin controller في النظام.

الـ function الداخلية في kernel core هي `pinmux_set_mux` في `drivers/pinctrl/core.c` — دي اللي بتنادي `pmxops->set_mux` وبتبعت `func_selector` و `group_selector`. هي exported بشكل indirect عبر الـ pinctrl core ومتاحة للـ kprobe.

---

### الـ Module

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * pinmux_spy.c - kprobe on pinmux_set_mux() to trace mux switches
 *
 * Every time the pinctrl core activates a mux function on a pin group,
 * we intercept the call and print the func_selector + group_selector.
 */

#include <linux/kernel.h>      /* pr_info, pr_err */
#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>     /* kprobe, register_kprobe */
#include <linux/pinctrl/pinctrl.h>  /* pinctrl_dev_get_name() */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("Trace pinmux_set_mux() calls via kprobe");

/*
 * pinmux_set_mux prototype (from drivers/pinctrl/pinmux.c):
 *
 *   int pinmux_set_mux(struct pinctrl_dev *pctldev,
 *                      unsigned int func_selector,
 *                      unsigned int group_selector);
 *
 * الـ kprobe بيعترض الـ call قبل ما تتنفذ،
 * والـ regs بتخلينا نقرأ الـ arguments من الـ calling convention.
 */

/* ---- pre-handler: بيتشغل فور ما الـ CPU تصل لأول instruction في الـ function ---- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * x86-64 calling convention:
     *   rdi = arg0 → pctldev
     *   rsi = arg1 → func_selector
     *   rdx = arg2 → group_selector
     *
     * ARM64:
     *   regs->regs[0] = pctldev
     *   regs->regs[1] = func_selector
     *   regs->regs[2] = group_selector
     *
     * بنستخدم macro عشان يشتغل على كلهم.
     */
#ifdef CONFIG_X86_64
    struct pinctrl_dev *pctldev = (struct pinctrl_dev *)regs->di;
    unsigned int func_sel      = (unsigned int)regs->si;
    unsigned int group_sel     = (unsigned int)regs->dx;
#elif defined(CONFIG_ARM64)
    struct pinctrl_dev *pctldev = (struct pinctrl_dev *)regs->regs[0];
    unsigned int func_sel      = (unsigned int)regs->regs[1];
    unsigned int group_sel     = (unsigned int)regs->regs[2];
#else
    /* على architectures تانية: بس نطبع إن الـ call حصل */
    struct pinctrl_dev *pctldev = NULL;
    unsigned int func_sel = 0, group_sel = 0;
#endif

    /*
     * pinctrl_dev_get_name() بترجع اسم الـ pin controller
     * زي "soc:pinctrl@fe600000" — بتساعدنا نعرف إيه الـ hardware.
     */
    if (pctldev)
        pr_info("[pinmux_spy] controller=%s func_sel=%u group_sel=%u\n",
                pinctrl_dev_get_name(pctldev), func_sel, group_sel);
    else
        pr_info("[pinmux_spy] set_mux called (arch unsupported for arg decode)\n");

    return 0; /* 0 = استمر في تنفيذ الـ original function */
}

/* ---- post-handler: بيتشغل بعد ما الـ function تخلص ---- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * بنطبع الـ return value من ax/regs[0] عشان نشوف لو الـ mux set نجح ولا لأ.
     * قيمة سالبة = error، صفر = success.
     */
#ifdef CONFIG_X86_64
    long ret = (long)regs->ax;
#elif defined(CONFIG_ARM64)
    long ret = (long)regs->regs[0];
#else
    long ret = 0;
#endif
    if (ret < 0)
        pr_info("[pinmux_spy] set_mux FAILED ret=%ld\n", ret);
}

/* ---- تعريف الـ kprobe نفسه ---- */
static struct kprobe kp = {
    /*
     * اسم الـ symbol اللي هنضع عليه الـ probe —
     * الـ kernel بيحوله لـ address وقت الـ register.
     */
    .symbol_name = "pinmux_set_mux",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ---- module_init: بيسجل الـ kprobe ---- */
static int __init pinmux_spy_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[pinmux_spy] register_kprobe failed: %d\n", ret);
        return ret;
    }

    /*
     * kp.addr بيتملى من الـ kernel بعد الـ register —
     * بنطبعه عشان نتأكد إن الـ probe اتوضع على الـ address الصح.
     */
    pr_info("[pinmux_spy] planted at %p (%s)\n", kp.addr, kp.symbol_name);
    return 0;
}

/* ---- module_exit: بيشيل الـ kprobe قبل ما الـ module يتـunload ---- */
static void __exit pinmux_spy_exit(void)
{
    /*
     * لازم نعمل unregister قبل الـ unload —
     * لو سبناه، الـ kernel هيحاول ينفذ handler مش موجود ويـcrash.
     */
    unregister_kprobe(&kp);
    pr_info("[pinmux_spy] removed\n");
}

module_init(pinmux_spy_init);
module_exit(pinmux_spy_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/kernel.h` | `pr_info()`, `pr_err()` |
| `linux/module.h` | `module_init/exit`, `MODULE_LICENSE` |
| `linux/kprobes.h` | `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/pinctrl/pinctrl.h` | `pinctrl_dev_get_name()` عشان نطبع اسم الـ controller |

**الـ** `pinctrl/pinctrl.h` بيجيب `pinctrl_dev_get_name()` اللي بتاخد `pctldev *` وبترجع اسمه كـ string — من غيرها هنطبع بس أرقام مش واضحة.

---

#### الـ `handler_pre`

بيتشغل مباشرة قبل تنفيذ `pinmux_set_mux`. بنقرأ الـ arguments من الـ `pt_regs` حسب الـ calling convention للـ architecture. الـ `func_selector` بيحدد أيه الـ function (مثلاً UART أو SPI)، والـ `group_selector` بيحدد أيه مجموعة الـ pins اللي هتتعمل mux عليها.

---

#### الـ `handler_post`

بيتشغل بعد ما الـ original function تخلص وترجع. بنشيك على الـ return value — لو سالب يبقى في error في الـ mux switching (مثلاً pin مشغول أو conflict). ده بيساعد في debugging الـ pinmux failures.

---

#### الـ `struct kprobe`

**الـ** `symbol_name` بيخلي الـ kernel يبحث عن عنوان `pinmux_set_mux` في الـ kallsyms وقت الـ `register_kprobe()`. ملوش تحكم في الـ address مباشرة — الـ kernel هو اللي بيملا `kp.addr` أوتوماتيك.

---

#### الـ `module_exit`

**الـ** `unregister_kprobe` ضروري قبل أي unload — لو الـ module اتشال والـ probe لسه موجود، أول call لـ `pinmux_set_mux` بعدها هتجيب kernel panic لأن الـ handler address بقى invalid.

---

### تجربة الـ Module

```bash
# بناء الـ module (Makefile بسيط)
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod pinmux_spy.ko

# شوف الـ output في الـ kernel log
sudo dmesg -w | grep pinmux_spy

# لو عندك device بيعمل pinmux (مثلاً USB أو UART بيتـenable):
# هتشوف output زي:
# [pinmux_spy] controller=soc:pinctrl func_sel=3 group_sel=7
# [pinmux_spy] controller=soc:pinctrl func_sel=3 group_sel=7

# إزالة الـ module
sudo rmmod pinmux_spy
```

### Makefile

```makefile
obj-m += pinmux_spy.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
