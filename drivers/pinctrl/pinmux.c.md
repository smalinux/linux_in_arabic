## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتكلم عنه الملف

الملفان `pinmux.c` و `pinmux.h` جزء من **PIN CONTROLLER subsystem** في Linux kernel، المسؤول عنه Linus Walleij وبيتتبعه على قائمة البريد `linux-gpio@vger.kernel.org`. الـ subsystem ده بيشمل كل حاجة في `drivers/pinctrl/` و `include/linux/pinctrl/`.

---

### المشكلة اللي بيحلها الملف ده — الـ Big Picture

#### تخيل معايا: الـ SoC زي لوحة مفاتيح مشتركة

تخيل عندك **لوحة مفاتيح فيها 100 مفتاح** (ده الـ SoC — System on Chip).

كل مفتاح (= **pin**) ممكن يشتغل بأكتر من طريقة:
- يشتغل كـ **GPIO** (وصلة رقمية عادية input/output)
- يشتغل كـ **UART TX** (إرسال serial)
- يشتغل كـ **I2C SDA** (data line للـ I2C bus)
- يشتغل كـ **SPI MOSI** (data line للـ SPI bus)
- يشتغل كـ **PWM** (إشارة نبضية)

المشكلة: **مفتاح واحد بس** — ما تقدرش تستخدمه لأكتر من وظيفة في نفس الوقت.

لو الـ UART driver أخد المفتاح ده وحوّله لـ TX، والـ GPIO driver جه يحاول يستخدمه كـ GPIO، هيحصل **تعارض** وكل حاجة هتتعطل.

هنا بيجي دور **pinmux** — ده هو **حارس المرور** اللي بيقرر مين يستخدم أنهي pin وبأي وظيفة.

---

### القصة الكاملة بالتفصيل

#### المشهد الأول: الـ SoC Hardware

في أي SoC (Raspberry Pi، Qualcomm Snapdragon، STM32، إلخ)، جوا الـ chip في **registers** بتحدد كل pin هيشتغل إزاي. مثلاً:

```
Register 0x4000_0010:
  bits [1:0] = 00 → GPIO
  bits [1:0] = 01 → UART_TX
  bits [1:0] = 10 → I2C_SDA
  bits [1:0] = 11 → SPI_MOSI
```

الـ pin controller driver (اللي بيكتبه مصنع الـ chip) بيعرف إزاي يكتب في الـ registers دي.

#### المشهد التاني: الـ Kernel بيبدأ

لما الـ kernel بيبدأ (boot)، بيقرأ الـ device tree أو ACPI tables وبيعرف:
- "الـ UART device محتاج pin 42 يشتغل كـ UART_TX"
- "الـ I2C device محتاج pin 15 يشتغل كـ I2C_SDA"
- "الـ GPIO driver محتاج يقدر يوصل لـ pins 0-31"

#### المشهد التالت: دور pinmux.c

لما أي driver بيطلب يستخدم pin، الكود في `pinmux.c` بيعمل الآتي:

```
1. تأكد إن الـ pin مش مستخدم حد تاني (pin_request)
2. سجّل مين صاحبه (mux_owner أو gpio_owner)
3. قول للـ hardware driver "وليها الـ function ده على الـ group ده" (set_mux)
4. لما خلص، حرّر الـ pin (pin_free)
```

---

### المفاهيم الأساسية

| المفهوم | المعنى | مثال حقيقي |
|---------|--------|-----------|
| **pin** | وصلة فيزيائية واحدة على الـ SoC | رجل واحد من الـ chip |
| **function** | وظيفة ممكن الـ pin يعملها | UART، I2C، GPIO، SPI |
| **group** | مجموعة pins بتشتغل مع بعض لتحقيق function | UART محتاج TX + RX = group من pinين |
| **mux** | اختصار multiplexer — المفتاح اللي بيحدد الـ function | زي الـ switch في الـ circuit |
| **pinctrl_map** | جدول بيقول "الـ device X محتاج الـ function Y على الـ group Z" | خريطة الـ wiring |
| **pinctrl_setting** | الـ map بعد ما اتترجمت لـ selector numbers | الأوامر الجاهزة للتنفيذ |

---

### الـ strict mode — قصة مهمة

في بعض الـ controllers، الـ pin ممكن يشتغل كـ GPIO وفي نفس الوقت مضبوط على function تاني (زي إنه configured كـ UART لكن مش متصل بـ UART driver لسه).

الـ `strict` mode بيقول "**لا** — لو الـ pin مستخدم كـ mux function، ما ينفعش يتاخد كـ GPIO، والعكس":

```c
/* من pinmux.c */
if (ops->strict && desc->mux_usecount && !func_is_gpio)
    return false;  /* مش هينفع يتستخدم كـ GPIO */
```

---

### المكونات الداخلية للكود

#### `pinmux.c` — الـ implementation

| الـ Function | الدور |
|-------------|-------|
| `pinmux_check_ops()` | يتأكد إن الـ driver نفّذ الـ operations المطلوبة (get_functions_count، set_mux، إلخ) |
| `pin_request()` | يحجز pin لمستخدم معين — يفحص التعارضات أولاً |
| `pin_free()` | يحرر الـ pin ويرجع اسم المالك القديم |
| `pinmux_request_gpio()` | يطلب pin عشان يشتغل كـ GPIO |
| `pinmux_free_gpio()` | يحرر الـ GPIO pin |
| `pinmux_gpio_direction()` | يحدد اتجاه الـ GPIO (input/output) |
| `pinmux_map_to_setting()` | يحوّل الـ map (function name كـ string) لـ setting (selector كـ رقم) |
| `pinmux_enable_setting()` | يفعّل الـ mux setting — يحجز كل الـ pins في الـ group ويبعت set_mux للـ hardware |
| `pinmux_disable_setting()` | يعطّل الـ mux setting ويحرر الـ pins |

#### الـ Generic Functions (للـ drivers اللي مش عايزة تكتب boilerplate)

```c
/* بدل ما كل driver يكتب نفس الكود، الـ generic layer بيوفّر: */
pinmux_generic_add_function()      /* أضف function جديدة */
pinmux_generic_get_function_name() /* اجلب اسم function بالـ selector */
pinmux_generic_get_function_groups() /* اجلب الـ groups المرتبطة بـ function */
pinmux_generic_remove_function()   /* امسح function */
pinmux_generic_free_functions()    /* امسح كل الـ functions */
```

الـ functions دي بتستخدم **radix tree** (`pin_function_tree`) لتخزين واسترجاع الـ functions بكفاءة.

---

### رحلة الـ pin من الطلب للتفعيل

```
Device Driver
    │
    ▼
pinctrl_select_state()          ← في core.c
    │
    ▼
pinmux_enable_setting()         ← في pinmux.c
    │
    ├── get_group_pins()         ← جلب الـ pins اللي في الـ group
    │
    ├── pin_request() × N       ← حجز كل pin واحد واحد
    │       │
    │       ├── فحص mux_owner   ← هل الـ pin محجوز؟
    │       ├── فحص gpio_owner  ← هل محجوز كـ GPIO؟
    │       └── gpio_request_enable() أو request()  ← إبلاغ الـ hardware driver
    │
    ├── تسجيل mux_setting على كل pin descriptor
    │
    └── ops->set_mux()          ← الأمر الفعلي للـ hardware: "وليها!"
            │
            ▼
        Hardware Register       ← اتغيرت القيمة في الـ SoC
```

---

### الـ debugfs interface

لو فعّلت `CONFIG_DEBUG_FS`، بتلاقي ملفات في `/sys/kernel/debug/pinctrl/<device>/`:

| الملف | المحتوى |
|-------|---------|
| `pinmux-functions` | كل الـ functions المتاحة والـ groups المرتبطة بيها |
| `pinmux-pins` | حالة كل pin (مين مالكه، أيه الـ function) |
| `pinmux-select` | تقدر تكتب فيه "group_name function_name" لتغيير الـ mux يدوياً |

---

### ملفات لازم تعرفها

#### الـ Core Files (القلب)

| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/core.c` | الـ core الرئيسي للـ subsystem — تسجيل الـ controllers، إدارة الـ states |
| `drivers/pinctrl/core.h` | الـ internal structures: `pin_desc`، `pinctrl_dev`، إلخ |
| `drivers/pinctrl/pinmux.c` | **(الملف ده)** — منطق الـ muxing والـ ownership |
| `drivers/pinctrl/pinmux.h` | الـ internal interface بين core وـ pinmux |
| `drivers/pinctrl/pinconf.c` | إدارة الـ pin configuration (pull-up، drive strength، إلخ) |
| `drivers/pinctrl/devicetree.c` | قراءة الـ pinctrl settings من الـ device tree |

#### الـ Public Headers (الـ API)

| الملف | الدور |
|-------|-------|
| `include/linux/pinctrl/pinmux.h` | الـ `pinmux_ops` struct — الـ interface اللي الـ drivers بتنفذه |
| `include/linux/pinctrl/pinctrl.h` | الـ `pinctrl_desc`، `pinfunction`، الـ pins registration |
| `include/linux/pinctrl/consumer.h` | الـ API اللي الـ device drivers بتستخدمه (pinctrl_get، pinctrl_select_state) |
| `include/linux/pinctrl/machine.h` | الـ `pinctrl_map` — ربط الـ devices بالـ functions |
| `include/linux/pinctrl/pinctrl-state.h` | أسماء الـ states المعيارية: "default"، "sleep"، "idle" |

#### مثال Hardware Driver

| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/pinctrl-bcm2835.c` | الـ driver الخاص بـ Raspberry Pi — بينفذ `pinmux_ops` |
| `drivers/pinctrl/intel/pinctrl-intel.c` | الـ driver الخاص بـ Intel platforms |
| `drivers/pinctrl/qcom/pinctrl-msm.c` | الـ driver الخاص بـ Qualcomm SoCs |

---

### الـ pinmux_ops — العقد بين pinmux.c والـ Hardware Driver

الـ hardware driver لازم ينفذ الـ struct ده:

```c
/* من include/linux/pinctrl/pinmux.h */
struct pinmux_ops {
    /* اختياري: هل الـ pin متاح للـ mux؟ */
    int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);
    int (*free)(struct pinctrl_dev *pctldev, unsigned int offset);

    /* إجباري: كام function عندك؟ وإيه اسم الـ function رقم N؟ */
    int (*get_functions_count)(struct pinctrl_dev *pctldev);
    const char *(*get_function_name)(struct pinctrl_dev *pctldev,
                                     unsigned int selector);

    /* إجباري: الـ function دي بتشتغل على أنهي groups؟ */
    int (*get_function_groups)(struct pinctrl_dev *pctldev,
                               unsigned int selector,
                               const char * const **groups,
                               unsigned int *num_groups);

    /* إجباري: فعّل الـ function دي على الـ group ده في الـ hardware */
    int (*set_mux)(struct pinctrl_dev *pctldev,
                   unsigned int func_selector,
                   unsigned int group_selector);

    /* اختياري: للـ GPIO المتسارع */
    int (*gpio_request_enable)(...);
    void (*gpio_disable_free)(...);
    int (*gpio_set_direction)(...);
    bool (*function_is_gpio)(...);

    bool strict; /* لا تسمح بالمشاركة بين GPIO وـ mux */
};
```

---

### الخلاصة بجملة واحدة

**الـ pinmux** هو "مأمور السجل" اللي بيسجّل مين مالك أنهي pin، بيمنع التعارضات، وبيأمر الـ hardware driver يغيّر الـ register المناسب عشان الـ pin يشتغل بالوظيفة المطلوبة.
## Phase 2: شرح الـ Pinmux Framework

### المشكلة — ليه الـ Pinmux موجود؟

أي SoC حديث — خد مثلاً Allwinner H3 أو STM32F4 — فيه مئات الـ pins جسدياً على الشريحة، لكن الـ peripherals المتاحة (UART, SPI, I2C, PWM, I2S, Ethernet…) أكتر بكتير من عدد الـ pins. الحل؟ كل pin ممكن يشتغل بأكتر من وظيفة — **مثلاً PA2 ممكن تكون UART2_TX أو TIM2_CH3 أو GPIO عادي**.

الـ hardware بيتحكم في ده عن طريق **مux register** — بتكتب فيه value محددة فيختار الـ pin يشتغل إزاي.

**المشكلة الحقيقية** إن من غير subsystem مركزي:
- كل driver بيكتب على نفس الـ registers من ورا ضهر بعضهم
- ما فيش حاجة تمنع UART driver من سرقة pin اللي SPI driver شغال عليه
- كل board يكتب الـ pin configuration بطريقة مختلفة

---

### الحل — الـ Pinmux Subsystem

الـ kernel بيحل الموضوع بـ **subsystem مركزي** يتحكم في كل عمليات الـ mux. الـ subsystem بيوفر:

1. **تسجيل وظائف الـ pins** — كل وظيفة (function) اسمها إيه وبتشتغل على أي مجموعة pins (group)
2. **نظام ownership** — كل pin بيتسجل باسم من طلبه، ما ينفعش اتنين يستخدموا نفس الـ pin في نفس الوقت
3. **فصل السياسة عن التنفيذ** — الـ core بيقرر "مين له حق"، الـ driver بيعمل فعلاً الـ hardware write

---

### التشبيه الواقعي — صالة الاجتماعات في شركة

تخيل شركة فيها **10 غرف اجتماعات** بس فيه **50 فريق** يحتاجوا يجتمعوا. كل غرفة ممكن تتحول لـ: studio تصوير، مكتب عمل، أو غرفة اجتماع.

| عنصر الشركة | المقابل في الـ Pinmux |
|---|---|
| الغرفة الجسدية | الـ **pin** الجسدي |
| وظيفة الغرفة (مكتب/studio) | الـ **function** (UART/SPI/GPIO) |
| الفريق اللي بيستخدم الغرفة | الـ **mux_owner** |
| مدير الجدول اللي بيحجز | الـ **pinmux core** |
| كل فريق بيحجز من خلال المدير | كل driver بيطلب عبر `pinmux_enable_setting()` |
| المدير بيرفض حجزين في نفس الوقت | الـ `pin_request()` بيرفض لو `mux_usecount != 0` |
| مجموعة غرف للحفلات | الـ **pin group** |
| نوع الحفلة (زفاف/مؤتمر) | الـ **function** اللي بتتطبق على الـ group |
| الموظف اللي بيعمل الـ setup الجسدي | الـ **pin controller driver** اللي بيكتب الـ registers |
| الموظف بيسأل المدير الأول | الـ `ops->request()` callback |

الدقة في التشبيه: المدير (الـ core) ما بيعرفش إزاي يضبط الأوضة جسدياً — ده شغل الموظف (الـ driver). المدير بس بيضمن ما فيش تعارض في الحجزات.

---

### Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                     Consumer Side                               │
  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
  │   │  UART    │  │  SPI     │  │  I2C     │  │  GPIO        │  │
  │   │  driver  │  │  driver  │  │  driver  │  │  subsystem   │  │
  └───┴────┬─────┴──┴────┬─────┴──┴────┬─────┴──┴──────┬───────┘──┘
           │              │              │               │
           │ pinctrl_get()│              │               │ pinctrl_gpio_request()
           │ + select_state│             │               │
           ▼              ▼              ▼               ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                   pinctrl core  (core.c)                        │
  │   ┌─────────────────────────────────────────────────────────┐   │
  │   │               pinmux core  (pinmux.c)                   │   │
  │   │  pinmux_enable_setting()  pinmux_request_gpio()         │   │
  │   │  pinmux_disable_setting() pinmux_free_gpio()            │   │
  │   │  pinmux_map_to_setting()  pinmux_gpio_direction()       │   │
  │   │  pin_request()  [ownership tracking]                    │   │
  │   └───────────────────────────┬─────────────────────────────┘   │
  └───────────────────────────────┼─────────────────────────────────┘
                                  │  calls pmxops callbacks
                                  ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │              Pin Controller Driver  (e.g. pinctrl-stm32.c)      │
  │                                                                  │
  │   struct pinmux_ops {                                            │
  │       .request            → check/reserve hardware resource      │
  │       .free               → release hardware resource            │
  │       .get_functions_count → how many mux functions exist?       │
  │       .get_function_name  → name of function[N]                  │
  │       .get_function_groups→ which pin groups support function[N] │
  │       .set_mux            → write to hardware mux registers      │
  │       .gpio_request_enable→ fast path: mux single pin to GPIO    │
  │       .gpio_disable_free  → fast path: release GPIO mux          │
  │       .gpio_set_direction → configure input/output in hardware   │
  │       .function_is_gpio   → is this function the GPIO function?  │
  │       .strict             → enforce exclusive ownership?         │
  │   }                                                              │
  └───────────────────────────┬─────────────────────────────────────┘
                               │
                               ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │                    Hardware Registers                            │
  │   GPIOA_MODER[1:0] = 0b10  → Alternate Function mode           │
  │   GPIOA_AFRL[7:4]  = 0x07  → AF7 = UART1                      │
  └─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — Functions, Groups, Settings

الـ pinmux subsystem بيبني نموذجه على **3 concepts أساسية**:

#### 1. الـ Pin Group
مجموعة pins جسدية بتشتغل مع بعضها كوحدة. مثلاً SPI1 بتتطلب 4 pins دايماً (CLK, MOSI, MISO, CS). الـ group بيقول "الـ pins دي بتتحرك مع بعض."

#### 2. الـ Function
الـ peripheral أو الوظيفة اللي ممكن تتحط على مجموعة pins. مثلاً `"spi1"` أو `"uart2"` أو `"gpio"`. function واحدة ممكن تتدعم على أكتر من group (لو الـ SoC بيسمح بـ SPI1 على أكتر من مجموعة pins بديلة).

#### 3. الـ Setting (pinctrl_setting_mux)
```c
struct pinctrl_setting_mux {
    unsigned int group;   /* selector للـ group المختار */
    unsigned int func;    /* selector للـ function المختارة */
};
```
ده **الاختيار الفعلي** — "استخدم function رقم 3 على group رقم 7". الـ setting بيتخزن في `pin_desc->mux_setting` لكل pin في الـ group.

---

### مخطط العلاقات بين الـ Structs

```
struct pinctrl_dev
├── desc → struct pinctrl_desc
│          ├── pmxops → struct pinmux_ops  (driver callbacks)
│          ├── pctlops → struct pinctrl_ops
│          └── pins[] / npins
│
├── pin_desc_tree (radix tree, indexed by pin number)
│   └── struct pin_desc
│       ├── name
│       ├── mux_usecount       ← عدد من طلبوا الـ pin
│       ├── mux_owner          ← "stm32-uart.1" مثلاً
│       ├── mux_setting ──────────────────────────────┐
│       ├── gpio_owner         ← "gpiochip0:5" مثلاً  │
│       └── mux_lock (mutex)                          │
│                                                     │
├── pin_function_tree (radix tree, indexed by selector)
│   └── struct function_desc  (CONFIG_GENERIC_PINMUX_FUNCTIONS)
│       ├── func → struct pinfunction
│       │          ├── name    ← "uart1"
│       │          ├── groups[] ← ["uart1-default", "uart1-alt"]
│       │          └── ngroups
│       └── data  ← driver-specific
│
└── pin_group_tree (radix tree, indexed by selector)
    └── struct group_desc
        ├── grp → struct pingroup
        │         ├── name   ← "uart1-default"
        │         ├── pins[] ← [10, 11]
        │         └── npins
        └── data

الـ pinctrl_setting_mux (referenced by mux_setting above):
struct pinctrl_setting_mux {
    unsigned int group;  ──→ index into pin_group_tree
    unsigned int func;   ──→ index into pin_function_tree
}
```

---

### الـ Ownership System — إزاي الـ pin_request() بيشتغل

ده القلب الحقيقي للـ pinmux core. كل pin فيه في `pin_desc` الخاصة بيه:

```c
unsigned int mux_usecount; /* reference count للـ mux users */
const char  *mux_owner;    /* اسم الـ device اللي claim الـ pin للـ function */
const char  *gpio_owner;   /* اسم الـ GPIO لو اتطلب كـ GPIO */
```

**السيناريوهات:**

```
السيناريو 1: pin طلبه function (مثلاً UART)
─────────────────────────────────────────
  pin_request(pctldev, pin=10, owner="stm32-uart.1", gpio_range=NULL)
  → mux_usecount++ (يبقى 1)
  → mux_owner = "stm32-uart.1"
  → بيستدعي ops->request(pctldev, 10) لو موجودة

السيناريو 2: نفس الـ device طلب نفس الـ pin تاني مرة
──────────────────────────────────────────────────
  pin_request(pctldev, pin=10, owner="stm32-uart.1", gpio_range=NULL)
  → مقارنة mux_owner == owner → true
  → mux_usecount++ (يبقى 2)
  → return 0  (بدون استدعاء ops->request)

السيناريو 3: device تاني يحاول يأخذ الـ pin
────────────────────────────────────────────
  pin_request(pctldev, pin=10, owner="stm32-spi.0", gpio_range=NULL)
  → mux_usecount != 0 && mux_owner != owner
  → return -EBUSY  + error log

السيناريو 4: طلب GPIO على pin مش له function
──────────────────────────────────────────────
  pinmux_request_gpio(pctldev, range, pin=10, gpio=5)
  → owner = kasprintf("gpiochip0:5")
  → pin_request(..., gpio_range=range)
  → gpio_owner = "gpiochip0:5"
  → بيستدعي ops->gpio_request_enable()
```

---

### الـ strict Mode

لما `ops->strict = true`، الـ controller بيفرض قاعدة صارمة:
**pin مش ممكن يكون له `mux_owner` و`gpio_owner` في نفس الوقت، حتى لو الـ function هو GPIO**.

بيُستخدم في controllers زي STM32 اللي عندها مفهوم واضح: pin إما في mode الـ alternate function أو في mode الـ GPIO — مش الاتنين.

لما `strict = false` (الـ default)، ممكن pin يكون له mux function وفي نفس الوقت يتطلب كـ GPIO — بعض الـ SoCs القديمة بتسمح بكده.

```c
/* من pinmux_can_be_used_for_gpio() */
if (ops->strict && desc->mux_usecount && !func_is_gpio)
    return false;  /* strict mode: pin مشغول بـ non-GPIO function */

return !(ops->strict && !!desc->gpio_owner);
/* strict mode: pin مشغول بـ GPIO → مش متاح للـ mux */
```

---

### الـ Map-to-Setting Pipeline

لما device بيطلب activate state معين (مثلاً "default")، الـ flow بيكون:

```
Device Tree / board file
      │
      ▼
struct pinctrl_map {
    .dev_name  = "stm32-uart.1",
    .name      = "default",
    .type      = PIN_MAP_TYPE_MUX_GROUP,
    .data.mux = {
        .function = "uart1",
        .group    = "uart1-default",   /* optional */
    }
}
      │
      ▼  pinmux_validate_map()
      │  → يتأكد إن .function موجودة

      ▼  pinmux_map_to_setting()
      │  → pinmux_func_name_to_selector() يحول "uart1" → func=3
      │  → pmxops->get_function_groups()  يجيب groups المتاحة
      │  → لو .group محدد: match_string() يتحقق إنها valid
      │  → pinctrl_get_group_selector() يحول "uart1-default" → group=7
      │  → setting->data.mux = { .func=3, .group=7 }

      ▼  pinmux_enable_setting()
      │  → pctlops->get_group_pins() يجيب pins[] = [10, 11]
      │  → pin_request() على كل pin
      │  → desc->mux_setting = &setting->data.mux  لكل pin
      │  → ops->set_mux(pctldev, func=3, group=7)
      │       ← هنا الـ driver بيكتب فعلاً على الـ hardware registers
      ▼
    Hardware configured!
```

---

### الـ Generic Pinmux Functions (CONFIG_GENERIC_PINMUX_FUNCTIONS)

بدل ما كل driver يكتب `get_functions_count` / `get_function_name` / `get_function_groups` من الصفر، الـ kernel بيوفر **generic implementation** بتستخدم **radix tree** (`pin_function_tree` في `pinctrl_dev`).

```c
/* الـ driver بس بيعمل كده وبخلص */
pinmux_generic_add_function(pctldev,
    "uart1",                          /* اسم الـ function */
    uart1_groups, ARRAY_SIZE(uart1_groups),  /* الـ groups */
    &uart1_data                       /* driver-specific data */
);

/* الـ core تلقائياً بيعمل: */
struct function_desc *function = devm_kzalloc(...);
function->func = { .name="uart1", .groups=uart1_groups, .ngroups=N };
radix_tree_insert(&pctldev->pin_function_tree, selector, function);
pctldev->num_functions++;
```

الـ radix tree بتعطي O(log n) lookup بـ selector number — مهم جداً لـ SoCs اللي عندها مئات الـ functions.

---

### الـ GPIO Fast Path

الـ GPIO subsystem بيتعامل مع الـ pins بشكل مختلف — مش عن طريق states وmaps، ولكن عن طريق `pinctrl_gpio_request()` مباشرة. الـ pinmux core بيوفر fast path مخصوص لده:

```
GPIO subsystem                   pinmux core              Driver
     │                               │                      │
     │  pinmux_request_gpio()        │                      │
     │ ─────────────────────────────►│                      │
     │                               │  pin_request()       │
     │                               │  (gpio_range != NULL)│
     │                               │                      │
     │                               │  ops->gpio_request_enable()
     │                               │ ────────────────────►│
     │                               │                      │ write HW
     │  pinmux_gpio_direction()      │                      │
     │ ─────────────────────────────►│                      │
     │                               │  ops->gpio_set_direction()
     │                               │ ────────────────────►│
     │                               │                      │ write HW
```

**ليه fast path؟** لأن الـ GPIO بتطلب تحريك pin واحد بس، مش group كامل. الـ `gpio_request_enable` callback بتأخذ `pinctrl_gpio_range` وـ offset — مش function selector وgroup selector.

---

### إيه اللي الـ Core بيمتلكه vs. إيه اللي بيفوّضه للـ Driver

| المسؤولية | الـ Core (pinmux.c) | الـ Driver (pmxops) |
|---|---|---|
| تتبع الـ ownership | نعم — `mux_owner`, `mux_usecount`, `gpio_owner` | لأ |
| منع التعارض بين drivers | نعم — `pin_request()` checks | لأ |
| ربط الـ map بـ selector numbers | نعم — `pinmux_map_to_setting()` | لأ |
| الكتابة على الـ hardware registers | لأ | نعم — `set_mux()` |
| تحديد قائمة الـ functions | لأ | نعم — `get_functions_count`, `get_function_name` |
| تحديد قائمة الـ groups لكل function | لأ | نعم — `get_function_groups` |
| التحقق من إمكانية استخدام pin | نعم — بيسأل الـ driver أولاً | نعم — `request()` callback |
| الـ reference counting للـ module | نعم — `try_module_get` / `module_put` | لأ |
| الـ debugfs | نعم — `pinmux_init_device_debugfs()` | جزئياً — بيقرأ من الـ ops |

---

### ملاحظة على الـ Subsystems المرتبطة

- **الـ pinctrl core** (عكس الـ pinmux): بيتكلم عن pin groups وpin configs (pull-up, drive strength) — مش الـ mux function. الـ pinmux بيكمل الـ pinctrl مش بيستبدله.
- **الـ GPIO subsystem**: بيستخدم `pinmux_request_gpio()` / `pinmux_free_gpio()` / `pinmux_gpio_direction()` لما يحتاج يعمل mux على pin. علاقة اتجاهية: GPIO يطلب من pinmux.
- **الـ device tree / machine mapping**: الـ `pinctrl_map` بيتيجي من DT parsing أو board file — ده خارج الـ pinmux نفسه، موجود في `pinctrl/devicetree.c` و`pinctrl/machine.c`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### `enum pinctrl_map_type` — نوع الـ mapping entry

| القيمة | المعنى |
|---|---|
| `PIN_MAP_TYPE_INVALID` | قيمة غير صالحة، sentinel |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمية بدون أي إعداد فعلي للـ hardware |
| `PIN_MAP_TYPE_MUX_GROUP` | إعداد mux لمجموعة pins |
| `PIN_MAP_TYPE_CONFIGS_PIN` | إعداد config (pull/drive) لـ pin واحد |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | إعداد config لمجموعة pins كاملة |

#### `PINFUNCTION_FLAG_GPIO` — flag على الـ `pinfunction`

| الـ Flag | القيمة | المعنى |
|---|---|---|
| `PINFUNCTION_FLAG_GPIO` | `BIT(0)` | الـ function دي هي GPIO mode — بتُستخدم مع `strict` mode عشان الـ core يعرف إن الـ pin مش occupied بـ function تانية |

#### الـ Config Options المهمة

| الـ Kconfig | الأثر |
|---|---|
| `CONFIG_PINMUX` | بيفعّل كل subsystem الـ pinmux — لو معطل، كل الفنكشنز بتتحول لـ stubs بترجع 0 |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | بيفعّل الـ generic radix-tree backend للـ functions — الـ driver مش محتاج يعمل إدارة functions يدوية |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | نفس الفكرة للـ groups |
| `CONFIG_DEBUG_FS` | بيضيف `/sys/kernel/debug/pinctrl/<dev>/pinmux-{functions,pins,select}` |

#### الـ Strict Mode

| الحالة | `strict = false` | `strict = true` |
|---|---|---|
| pin مأخوذ بـ mux | ممكن GPIO يطلبه بشرط نفس الـ owner | ممنوع تمامًا |
| pin مأخوذ بـ GPIO | ممكن mux يطلبه | ممنوع تمامًا |
| الاستخدام الشائع | controllers قديمة | controllers حديثة precise |

---

### الـ Structs المهمة

#### 1. `struct pinmux_ops` — vtable الـ driver

**الغرض:** الـ interface الرئيسي اللي كل driver لازم يطبّقه عشان يدعم الـ pinmux. الـ core بيتعامل مع الـ hardware من خلاله بس.

```c
struct pinmux_ops {
    /* اختياري: هل ممكن نطلب الـ pin ده؟ */
    int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);

    /* اختياري: عكس request */
    int (*free)(struct pinctrl_dev *pctldev, unsigned int offset);

    /* إجباري: كام function عندك؟ */
    int (*get_functions_count)(struct pinctrl_dev *pctldev);

    /* إجباري: اسم الـ function رقم selector */
    const char *(*get_function_name)(struct pinctrl_dev *pctldev,
                                      unsigned int selector);

    /* إجباري: الـ groups المرتبطة بالـ function */
    int (*get_function_groups)(struct pinctrl_dev *pctldev,
                               unsigned int selector,
                               const char * const **groups,
                               unsigned int *num_groups);

    /* اختياري: هل الـ function دي هي GPIO؟ */
    bool (*function_is_gpio)(struct pinctrl_dev *pctldev,
                              unsigned int selector);

    /* إجباري: فعّل الـ mux على الـ hardware */
    int (*set_mux)(struct pinctrl_dev *pctldev,
                   unsigned int func_selector,
                   unsigned int group_selector);

    /* اختياري: طلب GPIO على pin بعينه */
    int (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                                struct pinctrl_gpio_range *range,
                                unsigned int offset);

    /* اختياري: تحرير GPIO */
    void (*gpio_disable_free)(struct pinctrl_dev *pctldev,
                               struct pinctrl_gpio_range *range,
                               unsigned int offset);

    /* اختياري: تحديد اتجاه الـ GPIO */
    int (*gpio_set_direction)(struct pinctrl_dev *pctldev,
                               struct pinctrl_gpio_range *range,
                               unsigned int offset,
                               bool input);

    /* لو true: لا يسمح بمشاركة pin بين GPIO وmux */
    bool strict;
};
```

**الـ callbacks الإجبارية لـ `pinmux_check_ops()`:**
- `get_functions_count`
- `get_function_name`
- `get_function_groups`
- `set_mux`

---

#### 2. `struct pinctrl_dev` — الـ pin controller device

**الغرض:** التمثيل الداخلي لأي pin controller مسجّل في الـ kernel. بيجمع كل المعلومات: الـ descriptor، الـ pins، الـ functions، الـ groups، والـ GPIO ranges.

| الحقل | النوع | المعنى |
|---|---|---|
| `node` | `list_head` | ربطه في قائمة الـ controllers العالمية |
| `desc` | `const struct pinctrl_desc *` | الـ descriptor الي جه من الـ driver |
| `pin_desc_tree` | `radix_tree_root` | شجرة تحتوي كل الـ `pin_desc` مفهرسة برقم الـ pin |
| `pin_function_tree` | `radix_tree_root` | شجرة الـ functions (لو `CONFIG_GENERIC_PINMUX_FUNCTIONS`) |
| `pin_group_tree` | `radix_tree_root` | شجرة الـ groups (لو `CONFIG_GENERIC_PINCTRL_GROUPS`) |
| `num_functions` | `unsigned int` | عداد الـ functions المسجّلة |
| `gpio_ranges` | `list_head` | قائمة الـ GPIO ranges المرتبطة |
| `dev` | `struct device *` | الـ device المقابل |
| `owner` | `struct module *` | الـ module — بيُحسب `refcount` عليه لكل pin مطلوب |
| `p` | `struct pinctrl *` | الـ pinctrl handle الخاص بالـ controller نفسه (للـ hog) |
| `hog_default` / `hog_sleep` | `struct pinctrl_state *` | الـ states المحجوزة لنفس الـ controller |
| `mutex` | `struct mutex` | بيحمي عمليات الـ controller كلها |
| `device_root` | `struct dentry *` | جذر الـ debugfs لهذا الـ controller |

---

#### 3. `struct pinctrl_desc` — الـ descriptor الي بيقدمه الـ driver

**الغرض:** الـ static description اللي الـ driver بيسجّله عند الـ `pinctrl_register_and_init()`.

| الحقل | المعنى |
|---|---|
| `name` | اسم الـ controller |
| `pins` | مصفوفة `pinctrl_pin_desc` بكل الـ pins |
| `npins` | عدد الـ pins |
| `pctlops` | vtable عمليات الـ groups |
| `pmxops` | **vtable الـ pinmux** — القلب بتاع ملفنا |
| `confops` | vtable الـ pin config |
| `owner` | الـ module |
| `link_consumers` | لو `true`: بيعمل device link بين الـ controller والـ consumers |

---

#### 4. `struct pin_desc` — descriptor الـ pin الواحد

**الغرض:** الحالة الداخلية لكل pin فيزيائي — من يملكه وإيه الـ mux setting المفعّلة عليه.

```c
struct pin_desc {
    struct pinctrl_dev     *pctldev;     /* الـ controller المالك */
    const char             *name;        /* اسم الـ pin */
    bool                    dynamic_name;/* الاسم اتعمل بـ kasprintf */
    void                   *drv_data;   /* بيانات خاصة بالـ driver */

    /* الحقول دي بس لو CONFIG_PINMUX مفعّل */
    unsigned int            mux_usecount; /* عدد مرات الطلب بـ mux */
    const char             *mux_owner;   /* اسم الـ device الطالب */
    const struct pinctrl_setting_mux *mux_setting; /* الـ setting الفعلي */
    const char             *gpio_owner;  /* اسم الـ GPIO الطالب */
    struct mutex            mux_lock;    /* يحمي الحقول السابقة */
};
```

**نقطة مهمة:** `mux_usecount` مش boolean — نفس الـ device ممكن يطلب نفس الـ group أكتر من مرة لو فيه entries متعددة في mapping table بتشاور على نفس الـ group.

---

#### 5. `struct pinctrl_setting` — الـ setting المُفعَّل

**الغرض:** تمثيل وحدة الإعداد الواحدة — إما mux أو config — داخل الـ state.

```c
struct pinctrl_setting {
    struct list_head          node;      /* رابط في قائمة الـ state */
    enum pinctrl_map_type     type;      /* MUX_GROUP أو CONFIGS_* */
    struct pinctrl_dev       *pctldev;  /* الـ controller المسؤول */
    const char               *dev_name; /* اسم الـ device المالك */
    union {
        struct pinctrl_setting_mux     mux;     /* func + group selectors */
        struct pinctrl_setting_configs configs; /* config values */
    } data;
};
```

---

#### 6. `struct pinctrl_setting_mux` — بيانات الـ mux setting

**الغرض:** بيخزّن الـ selectors الرقمية للـ function والـ group اللي اتحوّلت من أسماء نصية عبر `pinmux_map_to_setting()`.

```c
struct pinctrl_setting_mux {
    unsigned int group; /* رقم الـ group selector */
    unsigned int func;  /* رقم الـ function selector */
};
```

---

#### 7. `struct pinctrl_map` — جدول الـ mapping الخام

**الغرض:** الـ static table اللي البورد أو الـ DT بيوفره — بيربط device باسم state بـ function وgroup.

```c
struct pinctrl_map {
    const char *dev_name;      /* اسم الـ device المستخدم */
    const char *name;          /* اسم الـ state ("default", "sleep") */
    enum pinctrl_map_type type;
    const char *ctrl_dev_name; /* اسم الـ pin controller */
    union {
        struct pinctrl_map_mux     mux;     /* group + function (نصي) */
        struct pinctrl_map_configs configs;
    } data;
};
```

---

#### 8. `struct function_desc` — الـ generic function descriptor

**الغرض:** wrapper بيستخدمه `CONFIG_GENERIC_PINMUX_FUNCTIONS` لتخزين الـ functions في الـ radix tree.

```c
struct function_desc {
    const struct pinfunction *func; /* الـ function الفعلية (name + groups) */
    void *data;                     /* بيانات خاصة بالـ driver */
};
```

---

#### 9. `struct pinfunction` — وصف الـ function

```c
struct pinfunction {
    const char *name;              /* اسم الـ function مثلاً "uart0" */
    const char * const *groups;   /* الـ groups اللي تقدر تشتغل عليها */
    size_t ngroups;
    unsigned long flags;           /* PINFUNCTION_FLAG_GPIO أو 0 */
};
```

---

#### 10. `struct pinctrl_gpio_range` — نطاق الـ GPIO

**الغرض:** بيحدد نطاق من أرقام الـ GPIO المقابلة لأرقام الـ pins في الـ controller.

| الحقل | المعنى |
|---|---|
| `name` | اسم الـ GPIO chip |
| `base` | أول رقم GPIO في النطاق |
| `pin_base` | أول رقم pin مقابل |
| `npins` | عدد الـ pins في النطاق |
| `pins` | مصفوفة صريحة لأرقام الـ pins (أو NULL) |
| `gc` | مؤشر اختياري للـ `gpio_chip` |

---

### مخطط علاقات الـ Structs (ASCII)

```
   ┌─────────────────────────────────────────────────────────┐
   │                    Board / DT                           │
   │  pinctrl_map[]                                          │
   │  { dev_name, state_name, type, ctrl_dev_name,          │
   │    data.mux = { group="grp0", function="uart0" } }      │
   └────────────────────┬────────────────────────────────────┘
                        │ pinctrl_register_mappings()
                        ▼
   ┌──────────────────────────────┐
   │      pinctrl_maps (list)     │◄── global list
   │  pinctrl_map[] + num_maps    │
   └──────────────────────────────┘
                        │ pinmux_map_to_setting()
                        ▼
   ┌──────────────────────────────────────────────────────────┐
   │                  pinctrl_dev                             │
   │  ┌────────────────────────────────────────────────────┐  │
   │  │ desc ──► pinctrl_desc                             │  │
   │  │          ├── pmxops ──► pinmux_ops (vtable)       │  │
   │  │          ├── pctlops ──► pinctrl_ops               │  │
   │  │          └── pins[]  ──► pinctrl_pin_desc[]        │  │
   │  ├── pin_desc_tree (radix) ──► pin_desc × N           │  │
   │  ├── pin_function_tree (radix) ──► function_desc × M  │  │
   │  ├── pin_group_tree (radix) ──► group_desc × K        │  │
   │  ├── gpio_ranges (list) ──► pinctrl_gpio_range × R    │  │
   │  ├── p ──► pinctrl (self-hog handle)                  │  │
   │  └── mutex                                             │  │
   │  └────────────────────────────────────────────────────┘  │
   └──────────────────────────────────────────────────────────┘
          │                              │
          │ pin_desc_get()               │ radix_tree_lookup()
          ▼                              ▼
   ┌──────────────┐           ┌─────────────────────┐
   │   pin_desc   │           │   function_desc      │
   │  ├─ name     │           │  ├─ func ──► pinfunction
   │  ├─ mux_usecount         │  └─ data (driver)    │
   │  ├─ mux_owner            └─────────────────────┘
   │  ├─ mux_setting ──► pinctrl_setting_mux
   │  ├─ gpio_owner            ┌─────────────────────┐
   │  └─ mux_lock              │  pinfunction         │
   └──────────────┘            │  ├─ name "uart0"    │
                               │  ├─ groups[]        │
                               │  ├─ ngroups         │
                               │  └─ flags (GPIO?)   │
                               └─────────────────────┘

   ┌──────────────────────────────────────────────────────────┐
   │                    pinctrl (per consumer device)         │
   │  ├─ dev ──► struct device                               │
   │  ├─ states (list) ──┐                                   │
   │  ├─ state ──────────┤► pinctrl_state                    │
   │  └─ users (kref)    │   ├─ name "default"               │
   │                     │   └─ settings (list)              │
   │                     │        ▼                          │
   │                     │   pinctrl_setting                 │
   │                     │   ├─ type = MUX_GROUP             │
   │                     │   ├─ pctldev ──► pinctrl_dev      │
   │                     │   ├─ dev_name                     │
   │                     │   └─ data.mux                     │
   │                     │       ├─ func = 3 (selector)      │
   │                     │       └─ group = 1 (selector)     │
   └──────────────────────────────────────────────────────────┘
```

---

### دورة حياة الـ Pin Controller (Lifecycle)

```
  [Driver Probe]
       │
       ▼
  pinctrl_register_and_init(desc, dev, data, &pctldev)
       │
       ├─► alloc pinctrl_dev
       ├─► copy desc pointer
       ├─► init radix trees (pin_desc, function, group)
       ├─► pinmux_check_ops()          ← يتحقق من الـ vtable
       ├─► register all pins ──► pin_desc per pin in radix tree
       │
       ▼
  pinctrl_enable(pctldev)
       │
       ├─► hog pins (self-mapping entries)
       └─► add to global pinctrl_dev list

  [Runtime: Consumer Driver]
       │
       ▼
  devm_pinctrl_get(dev)  ──► struct pinctrl *p
       │
       ▼
  pinctrl_lookup_state(p, "default")  ──► struct pinctrl_state *s
       │
       ▼
  pinctrl_select_state(p, s)
       │
       ├─ for each pinctrl_setting in state->settings:
       │     pinmux_enable_setting(setting)
       │         ├─ get_group_pins() ──► pins[]
       │         ├─ pin_request() × num_pins
       │         │     ├─ mux_usecount++
       │         │     ├─ try_module_get()
       │         │     └─ ops->request() or gpio_request_enable()
       │         ├─ desc->mux_setting = &setting->data.mux
       │         └─ ops->set_mux(func, group) ──► HARDWARE REGISTER

  [Suspend / Driver Remove]
       │
       ▼
  pinctrl_select_state(p, sleep_state)  أو  pinctrl_put(p)
       │
       ▼
  pinmux_disable_setting(setting)
       │
       ├─ for each pin: check mux_setting matches
       └─ pin_free()
             ├─ mux_usecount--
             ├─ ops->free() or gpio_disable_free()
             └─ module_put()

  [Controller Remove]
       │
       ▼
  pinctrl_unregister(pctldev)
       │
       ├─ pinmux_generic_free_functions()
       ├─ free all pin_descs from radix tree
       └─ remove from global list
```

---

### مخطط تدفق الـ GPIO Request

```
gpio_request(gpio_num)                    [GPIO subsystem]
    │
    ▼
pinctrl_gpio_request(pin)                 [pinctrl core]
    │
    ▼
pinmux_request_gpio(pctldev, range, pin, gpio)
    │
    ├─ kasprintf("%s:%d", range->name, gpio)  ← owner string
    │
    ▼
pin_request(pctldev, pin, owner, range)
    │
    ├─ pin_desc_get() ──► desc
    ├─ [lock desc->mux_lock]
    │   ├─ check mux_usecount (strict? GPIO conflicts mux?)
    │   ├─ check gpio_owner (already GPIO-owned?)
    │   └─ desc->gpio_owner = owner
    ├─ [unlock]
    ├─ try_module_get(pctldev->owner)
    └─ ops->gpio_request_enable(pctldev, range, pin)
           │
           ▼
       HARDWARE: switch pin to GPIO mode

gpio_direction_input(gpio_num)
    │
    ▼
pinmux_gpio_direction(pctldev, range, pin, input=true)
    │
    └─ ops->gpio_set_direction(pctldev, range, pin, true)
           │
           ▼
       HARDWARE: set direction register

gpio_free(gpio_num)
    │
    ▼
pinmux_free_gpio(pctldev, pin, range)
    │
    ▼
pin_free(pctldev, pin, range)
    │
    ├─ [lock mux_lock]
    │   ├─ owner = desc->gpio_owner
    │   └─ desc->gpio_owner = NULL
    ├─ [unlock]
    ├─ ops->gpio_disable_free(pctldev, range, pin)
    ├─ module_put()
    └─ kfree(owner)
```

---

### مخطط تدفق الـ Mux Setting (pinmux_enable_setting)

```
pinctrl_select_state(p, state)
    │
    └─► for each setting WHERE type == PIN_MAP_TYPE_MUX_GROUP:
            │
            ▼
        pinmux_enable_setting(setting)
            │
            ├─ pctlops->get_group_pins(group_selector)
            │       ──► pins[], num_pins
            │
            ├─ [Loop i = 0..num_pins-1]
            │   pin_request(pctldev, pins[i], dev_name, NULL)
            │       ├─ desc = pin_desc_get(pins[i])
            │       ├─ [lock mux_lock]
            │       │   ├─ check conflicts
            │       │   ├─ mux_usecount++
            │       │   └─ mux_owner = dev_name
            │       ├─ [unlock]
            │       ├─ try_module_get()
            │       └─ ops->request(pctldev, pin)   [optional]
            │
            ├─ [Loop i = 0..num_pins-1]
            │   [lock mux_lock]
            │   desc->mux_setting = &setting->data.mux
            │   [unlock]
            │
            └─ ops->set_mux(pctldev, func_selector, group_selector)
                    │
                    ▼
                HARDWARE REGISTERS
                (بيكتب قيمة في register الـ mux على الـ SoC)

    [On error at set_mux:]
        for each pin already requested:
            [lock mux_lock]  desc->mux_setting = NULL  [unlock]
        while --i >= 0:
            pin_free(pctldev, pins[i], NULL)
```

---

### مخطط تدفق `pinmux_map_to_setting` — التحويل من نص لـ selectors

```
pinmux_map_to_setting(map, setting)
    │
    ├─ map->data.mux.function = "uart0"   (نص)
    │
    ├─ pinmux_func_name_to_selector(pctldev, "uart0")
    │       └─ loop: ops->get_function_name(pctldev, selector)
    │               حتى يلاقي "uart0" ──► returns selector N
    │
    ├─ setting->data.mux.func = N
    │
    ├─ pmxops->get_function_groups(pctldev, N, &groups, &num_groups)
    │       ──► groups = ["grp0", "grp1"], num_groups = 2
    │
    ├─ لو map->data.mux.group محدد:
    │       match_string(groups, num_groups, group) ──► index
    │   لو مش محدد:
    │       group = groups[0]  (الأول افتراضيًا)
    │
    ├─ pinctrl_get_group_selector(pctldev, group)
    │       ──► returns group_selector M
    │
    └─ setting->data.mux.group = M
```

---

### مخطط تدفق `pinmux_generic_add_pinfunction` — تسجيل function

```
driver calls pinmux_generic_add_function(pctldev, "uart0", groups, ngroups, data)
    │
    ├─ PINCTRL_PINFUNCTION("uart0", groups, ngroups) ──► struct pinfunction
    │
    └─ pinmux_generic_add_pinfunction(pctldev, &func, data)
            │
            ├─ pinmux_func_name_to_selector() ──► لو موجود return selector مباشرة
            │
            ├─ selector = pctldev->num_functions  (index جديد)
            │
            ├─ devm_kzalloc() ──► function_desc
            │
            ├─ devm_kmemdup_const() ──► copy of pinfunction
            │
            ├─ function->data = data
            │
            ├─ radix_tree_insert(&pctldev->pin_function_tree, selector, function)
            │
            ├─ pctldev->num_functions++
            │
            └─ return selector
```

---

### استراتيجية الـ Locking

الـ pinmux بيستخدم **مستويين من الـ locks**:

#### المستوى الأول: `pctldev->mutex`

| يحمي | من أي context |
|---|---|
| عمليات الـ controller ككل | process context فقط |
| قراءة/كتابة الـ debugfs (`pinmux_functions_show`, `pinmux_pins_show`) | يُمسك طول العملية |
| الـ `pinmux_select_write` (debugfs write) | لا يُمسك صراحة — بيستخدم functions داخلية |

```c
// مثال: debugfs
mutex_lock(&pctldev->mutex);
nfuncs = pmxops->get_functions_count(pctldev);
// ... loop ...
mutex_unlock(&pctldev->mutex);
```

#### المستوى الثاني: `pin_desc->mux_lock`

| يحمي | من أي context |
|---|---|
| `mux_usecount` | process context |
| `mux_owner` | process context |
| `mux_setting` | process context |
| `gpio_owner` | process context |

الـ kernel بيستخدم `scoped_guard(mutex, &desc->mux_lock)` أو `guard(mutex)` من cleanup.h للـ automatic unlock.

```c
scoped_guard(mutex, &desc->mux_lock) {
    desc->mux_usecount++;
    desc->mux_owner = owner;
}
// automatically unlocked here
```

#### ترتيب الـ Locking (Lock Ordering)

```
pctldev->mutex          (الأعلى — outer lock)
    └─► pin_desc->mux_lock   (الأدنى — inner lock)
```

**مهم جدًا:** أي كود يمسك الاتنين لازم يمسك `pctldev->mutex` الأول عشان يتجنب الـ deadlock.

#### الـ Radix Tree والـ Locking

الـ `pin_function_tree` و`pin_desc_tree` بيتحموا بـ `pctldev->mutex` — الـ generic functions زي `pinmux_generic_remove_function()` بتشير صراحةً إن الـ caller هو المسؤول عن أخد الـ lock:

```c
/*
 * Note that the caller must take care of locking.
 */
int pinmux_generic_remove_function(struct pinctrl_dev *pctldev, ...)
```

#### جدول ملخص الـ Locks

| الـ Lock | يحمي | من يمسكه |
|---|---|---|
| `pctldev->mutex` | الـ function/group trees، الـ gpio_ranges list، الـ debugfs reads | الـ core وبعض الـ callers |
| `pin_desc->mux_lock` | `mux_usecount`, `mux_owner`, `mux_setting`, `gpio_owner` | `pin_request()`, `pin_free()`, `pinmux_enable_setting()`, `pinmux_can_be_used_for_gpio()` |
| `pinctrl_maps_mutex` | قائمة الـ `pinctrl_maps` العالمية | `pinctrl_register_mappings()` والـ core |
## Phase 4: شرح الـ Functions

---

### ملخص — Cheatsheet للـ Functions

#### مجموعة: Validation & Initialization

| Function | Location | الغرض |
|---|---|---|
| `pinmux_check_ops()` | pinmux.c | التحقق من اكتمال الـ `pinmux_ops` callbacks |
| `pinmux_validate_map()` | pinmux.c | التحقق من صحة مدخل `pinctrl_map` للـ mux |

#### مجموعة: GPIO ↔ Pinmux Bridge

| Function | Location | الغرض |
|---|---|---|
| `pinmux_can_be_used_for_gpio()` | pinmux.c | فحص هل الـ pin متاح للـ GPIO |
| `pinmux_request_gpio()` | pinmux.c | حجز pin لاستخدامه كـ GPIO |
| `pinmux_free_gpio()` | pinmux.c | تحرير pin من الـ GPIO |
| `pinmux_gpio_direction()` | pinmux.c | ضبط اتجاه الـ GPIO (input/output) |

#### مجموعة: Setting Lifecycle

| Function | Location | الغرض |
|---|---|---|
| `pinmux_map_to_setting()` | pinmux.c | تحويل `pinctrl_map` لـ `pinctrl_setting` |
| `pinmux_free_setting()` | pinmux.c | تحرير الـ setting (stub حالياً) |
| `pinmux_enable_setting()` | pinmux.c | تفعيل mux setting على الـ hardware |
| `pinmux_disable_setting()` | pinmux.c | تعطيل mux setting وتحرير الـ pins |

#### مجموعة: Internal Helpers

| Function | Location | الغرض |
|---|---|---|
| `pin_request()` | pinmux.c (static) | حجز pin منفرد — العملية الأساسية |
| `pin_free()` | pinmux.c (static) | تحرير pin منفرد |
| `pinmux_func_name_to_selector()` | pinmux.c (static) | بحث عن selector بالاسم |

#### مجموعة: Generic Functions (CONFIG_GENERIC_PINMUX_FUNCTIONS)

| Function | Export | الغرض |
|---|---|---|
| `pinmux_generic_get_function_count()` | `EXPORT_SYMBOL_GPL` | عدد الـ functions المسجلة |
| `pinmux_generic_get_function_name()` | `EXPORT_SYMBOL_GPL` | اسم function بالـ selector |
| `pinmux_generic_get_function_groups()` | `EXPORT_SYMBOL_GPL` | مجموعات الـ function |
| `pinmux_generic_get_function()` | `EXPORT_SYMBOL_GPL` | الـ `function_desc` بالكامل |
| `pinmux_generic_function_is_gpio()` | `EXPORT_SYMBOL_GPL` | هل الـ function هي GPIO؟ |
| `pinmux_generic_add_function()` | `EXPORT_SYMBOL_GPL` | إضافة function بالاسم والـ groups |
| `pinmux_generic_add_pinfunction()` | `EXPORT_SYMBOL_GPL` | إضافة function بـ `pinfunction` struct |
| `pinmux_generic_remove_function()` | `EXPORT_SYMBOL_GPL` | حذف function بالـ selector |
| `pinmux_generic_free_functions()` | internal | حذف جميع الـ functions |

#### مجموعة: DebugFS

| Function | الغرض |
|---|---|
| `pinmux_init_device_debugfs()` | إنشاء ملفات debugfs للـ device |
| `pinmux_show_map()` | طباعة `pinctrl_map` في debugfs |
| `pinmux_show_setting()` | طباعة `pinctrl_setting` في debugfs |
| `pinmux_functions_show()` | `/sys/kernel/debug/.../pinmux-functions` |
| `pinmux_pins_show()` | `/sys/kernel/debug/.../pinmux-pins` |
| `pinmux_select_write()` | كتابة في `pinmux-select` لتفعيل mux يدوياً |

---

### مجموعة 1: Validation & Initialization

الـ core يستدعي هذه الـ functions أثناء تسجيل الـ pin controller للتأكد من صحة البيانات قبل أي عملية.

---

#### `pinmux_check_ops()`

```c
int pinmux_check_ops(struct pinctrl_dev *pctldev);
```

الـ core يستدعيها في `pinctrl_register()` للتحقق من أن الـ driver يوفر الـ callbacks الإلزامية في `pinmux_ops`. تتحقق من وجود `get_functions_count`، `get_function_name`، `get_function_groups`، و`set_mux`. بعدها تمشي على كل الـ functions وتتأكد من أن كل واحدة عندها اسم غير NULL.

**Parameters:**
- `pctldev` — الـ pin controller المسجَّل، يحتوي على `desc->pmxops`

**Return:**
- `0` عند النجاح
- `-EINVAL` إذا أي callback إلزامي ناقص أو أي function بدون اسم

**Key Details:**
- يُشغَّل مرة واحدة فقط وقت التسجيل — لا يوجد locking ولا race condition هنا
- إذا رجع error فالـ `pinctrl_register()` بتفشل ومايتسجلش الـ controller

**Caller:** `pinctrl_register()` / `devm_pinctrl_register()` في `core.c`

---

#### `pinmux_validate_map()`

```c
int pinmux_validate_map(const struct pinctrl_map *map, int i);
```

تتحقق من أن الـ `pinctrl_map` من النوع `PIN_MAP_TYPE_MUX_GROUP` عنده `function` غير NULL. الـ `group` ممكن يكون NULL (الـ core هيختار الأول تلقائياً).

**Parameters:**
- `map` — مدخل الـ machine map
- `i` — رقم المدخل في الـ array (للـ error message فقط)

**Return:**
- `0` عند النجاح
- `-EINVAL` إذا `map->data.mux.function == NULL`

**Caller:** `pinctrl_register_map()` في `core.c`

---

### مجموعة 2: Internal Helpers — pin_request / pin_free

دي أهم functions في الملف — كل العمليات الخارجية بتتكل عليها.

---

#### `pin_request()` (static)

```c
static int pin_request(struct pinctrl_dev *pctldev,
                       int pin, const char *owner,
                       struct pinctrl_gpio_range *gpio_range);
```

الـ function المحورية اللي بتحجز pin منفرد. بتفرّق بين نوعين من الحجز:
- **GPIO request** (لما `gpio_range != NULL`): يكتب في `desc->gpio_owner`
- **Mux request** (لما `gpio_range == NULL`): يزيّد `desc->mux_usecount` ويكتب في `desc->mux_owner`

الـ `mux_usecount` بيسمح بالـ sharing — لو نفس الـ owner طلب نفس الـ pin تاني مرة يرجع `0` فوراً بدون ما يكلّم الـ hardware.

**Parameters:**
- `pctldev` — الـ pin controller
- `pin` — رقم الـ pin في الـ global space
- `owner` — string يمثّل المالك (اسم الـ device أو `"rangename:gpio_num"`)
- `gpio_range` — إذا طلب GPIO يبعت الـ range، إذا mux يبعت NULL

**Return:**
- `0` عند النجاح
- `-EINVAL` إذا الـ pin مش مسجّل أو في conflict
- error من `ops->request` أو `ops->gpio_request_enable`

**Key Details — Locking:**
- يمسك `desc->mux_lock` (per-pin mutex) في كل قراءة/كتابة لحقول الـ descriptor
- يستخدم `scoped_guard(mutex, ...)` — الـ lock بيتفكّ تلقائياً عند الخروج من الـ scope
- بعد تسجيل الـ ownership يستدعي `try_module_get()` لحماية الـ driver module

**Error Path:**
```
pin_request()
  ├── pin_desc_get()           → get descriptor
  ├── [mux_lock] check conflicts
  ├── set gpio_owner or mux_usecount/mux_owner
  ├── try_module_get()         → pin module ref
  ├── ops->gpio_request_enable() OR ops->request()
  └── on failure:
       ├── module_put()
       └── [mux_lock] rollback ownership
```

**الـ strict mode:**
لما `ops->strict == true`، المنع بيكون bilateral — pin مأخود كـ GPIO ممكن ميتطلبش كـ mux والعكس، حتى لو مختلف owners.

---

#### `pin_free()` (static)

```c
static const char *pin_free(struct pinctrl_dev *pctldev, int pin,
                            struct pinctrl_gpio_range *gpio_range);
```

عكس `pin_request()`. بتنقص `mux_usecount` وإذا وصلت لـ صفر بتمسح `mux_owner` و`mux_setting`. تعيد pointer للـ owner string — مهم لأن الـ caller هو اللي مسؤول عن `kfree()` اللي عملها `kasprintf()` في حالة الـ GPIO.

**Return:**
- الـ `owner` string pointer
- `NULL` لو الـ pin مش مسجّل أو `mux_usecount` كان zero (مع `WARN_ON`)

**Key Details:**
- `mux_usecount > 0` بعد التنقيص: بترجع NULL بدون ما تكلّم الـ hardware — shared ownership
- عند free الـ GPIO: يستدعي `ops->gpio_disable_free()`
- عند free الـ mux: يستدعي `ops->free()`
- دايماً بتنهي بـ `module_put()` مقابل `try_module_get()` اللي اتعمل في `pin_request()`

---

### مجموعة 3: GPIO ↔ Pinmux Bridge

---

#### `pinmux_can_be_used_for_gpio()`

```c
bool pinmux_can_be_used_for_gpio(struct pinctrl_dev *pctldev, unsigned int pin);
```

بتسأل: هل ممكن أستخدم الـ pin ده كـ GPIO دلوقتي؟ الجواب بيعتمد على حالة الـ pin والـ `strict` mode في الـ controller.

بتفحص:
1. لو الـ pin مفيش له descriptor أو مفيش `pmxops` → آمن (return true)
2. لو في `mux_setting` والـ driver يعرف يحدد هل الـ function دي GPIO أصلاً (`function_is_gpio`)
3. في الـ strict mode: لو الـ pin متأخد كـ mux بـ non-GPIO function → false
4. في الـ strict mode: لو الـ pin عنده `gpio_owner` → false

**Key Details:**
- يمسك `desc->mux_lock` للقراءة الآمنة
- المنطق ده مهم قبل ما `gpiochip_request_own_desc()` تستدعي `pinmux_request_gpio()`

**Caller:** `gpiochip_generic_request()` في `gpiolib.c`

---

#### `pinmux_request_gpio()`

```c
int pinmux_request_gpio(struct pinctrl_dev *pctldev,
                        struct pinctrl_gpio_range *range,
                        unsigned int pin, unsigned int gpio);
```

الـ public API اللي بيطلبه الـ GPIO subsystem لحجز pin. بتبني owner string بـ `kasprintf()` بالشكل `"rangename:gpio_num"` وتبعته لـ `pin_request()`.

**Parameters:**
- `range` — الـ GPIO range المطابق للـ GPIO controller
- `pin` — رقم الـ pin في الـ pin controller space
- `gpio` — رقم الـ GPIO في الـ global GPIO space

**Return:**
- `0` عند النجاح
- `-ENOMEM` لو `kasprintf()` فشلت
- أي error من `pin_request()`

**Key Details:**
- الـ owner string بتتخزن في `desc->gpio_owner` وبتتحرر في `pinmux_free_gpio()`
- مسؤولية الـ `kfree()` للـ owner string على `pinmux_free_gpio()` عن طريق القيمة المرجعة من `pin_free()`

---

#### `pinmux_free_gpio()`

```c
void pinmux_free_gpio(struct pinctrl_dev *pctldev, unsigned int pin,
                      struct pinctrl_gpio_range *range);
```

عكس `pinmux_request_gpio()`. بتستدعي `pin_free()` وتاخد الـ owner string المرجعة وتعمل لها `kfree()`.

**Caller:** `gpiochip_generic_free()` في `gpiolib.c`

---

#### `pinmux_gpio_direction()`

```c
int pinmux_gpio_direction(struct pinctrl_dev *pctldev,
                          struct pinctrl_gpio_range *range,
                          unsigned int pin, bool input);
```

بتضبط اتجاه الـ GPIO pin على مستوى الـ pinmux. لو الـ driver عنده `gpio_set_direction` callback بتستدعيه، غير كده بترجع `0` مباشرة (لو الـ hardware مش محتاج إعداد خاص للاتجاه).

**Parameters:**
- `input` — `true` = input، `false` = output

**Caller:** `pinctrl_gpio_direction_input/output()` في `core.c`

---

### مجموعة 4: Setting Lifecycle

الـ `pinctrl_setting` هو التمثيل الداخلي للـ active mux configuration. دي الـ functions اللي بتدير حياته.

---

#### `pinmux_map_to_setting()`

```c
int pinmux_map_to_setting(const struct pinctrl_map *map,
                          struct pinctrl_setting *setting);
```

بتحوّل الـ `pinctrl_map` (اللي بتيجي من machine code أو DT) لـ `pinctrl_setting` اللي فيه selectors رقمية جاهزة للـ hardware. دي أساساً مرحلة "resolve" — بتترجم أسماء لـ indices.

**Pseudocode Flow:**
```
pinmux_map_to_setting(map, setting):
  1. func_name → pinmux_func_name_to_selector() → setting->data.mux.func
  2. pmxops->get_function_groups(func) → groups[], num_groups
  3. if map->data.mux.group specified:
       match_string(groups, map->group) → validate
     else:
       use groups[0]   <- default: first group
  4. pinctrl_get_group_selector(group_name) → setting->data.mux.group
  5. return 0
```

**Return:**
- `0` عند النجاح
- `-EINVAL` لو الـ function أو الـ group مش موجود

**Key Details:**
- لا توجد locking هنا — الـ `pctldev->mutex` بيتمسك من الـ caller في `core.c`
- `setting->data.mux.func` و `setting->data.mux.group` هم selectors رقمية مش أسماء

---

#### `pinmux_free_setting()`

```c
void pinmux_free_setting(const struct pinctrl_setting *setting);
```

**الـ function دي stub فارغة حالياً** — التعليق في الكود يقول "currently unused". مكانها محجوز في الـ API للـ cleanup المستقبلي لو الـ settings احتاجت موارد ديناميكية.

---

#### `pinmux_enable_setting()`

```c
int pinmux_enable_setting(const struct pinctrl_setting *setting);
```

أعقد function في الملف. بتطبّق فعلياً الـ mux setting على الـ hardware. المراحل:

**Pseudocode Flow:**
```
pinmux_enable_setting(setting):
  1. pctlops->get_group_pins(group) -> pins[], num_pins
     on error: warn, set num_pins=0 (non-fatal)

  2. for each pin in pins[]:
       pin_request(pin, setting->dev_name, NULL)  <- mux request
       on error: goto err_pin_request

  3. for each pin in pins[]:
       [mux_lock] desc->mux_setting = &setting->data.mux

  4. ops->set_mux(pctldev, func, group)   <- actual HW config
     on error: goto err_set_mux

  5. return 0

err_set_mux:
  for each pin: [mux_lock] desc->mux_setting = NULL
err_pin_request:
  for pins already requested: pin_free()  <- rollback
  return ret
```

**Key Details:**
- الـ rollback كامل — لو أي step فشل الـ function بتتراجع عن كل اللي عملته
- `desc->mux_setting` بيتضبط قبل `set_mux()` للـ debugfs وفحوصات `pinmux_can_be_used_for_gpio()`
- كل write على `desc->mux_setting` بيتعمل تحت `desc->mux_lock`
- الـ `dev_name` بيبقى الـ `mux_owner` للـ pins دي

**Caller:** `pinctrl_select_state()` عن طريق `pinctrl_commit_state()` في `core.c`

---

#### `pinmux_disable_setting()`

```c
void pinmux_disable_setting(const struct pinctrl_setting *setting);
```

عكس `pinmux_enable_setting()`. بتمشي على pins الـ group وبتحرر كل pin عنده `mux_setting` تساوي `&setting->data.mux`. لو الـ pin اتأخد من setting تاني في الأثناء بتطبع warning وميحرروش.

**Key Details:**
- الـ comparison بتتعمل بـ pointer equality مش بـ value — ضمان أن ده نفس الـ setting object
- مفيش explicit call لـ `ops->set_mux` هنا — الـ hardware بيتعدّل لما الـ state الجديد يتفعّل

---

#### `pinmux_func_name_to_selector()` (static)

```c
static int pinmux_func_name_to_selector(struct pinctrl_dev *pctldev,
                                        const char *function);
```

Linear scan على كل الـ functions اللي الـ driver يدعمها للبحث عن الـ selector بالاسم.

**Return:**
- الـ selector (unsigned int مُرجَع كـ int) عند النجاح
- `-EINVAL` لو الاسم مش موجود

**Key Details:**
- `O(n)` في عدد الـ functions — مقبول لأن عدد الـ functions عادة صغير
- مُستخدَمة في `pinmux_map_to_setting()` وفي `pinmux_select_write()` (debugfs) وفي `pinmux_generic_add_pinfunction()` للـ idempotency check

---

### مجموعة 5: Generic Functions (CONFIG_GENERIC_PINMUX_FUNCTIONS)

الـ generic layer بتوفر implementation جاهزة لـ `pinmux_ops` callbacks للـ drivers اللي مش عايزة تكتب إدارة الـ functions من الصفر. البيانات مخزنة في `radix_tree` في `pctldev->pin_function_tree`.

---

#### `pinmux_generic_get_function_count()`

```c
int pinmux_generic_get_function_count(struct pinctrl_dev *pctldev);
```

بترجع `pctldev->num_functions` مباشرة. بسيطة جداً لكن مطلوبة كـ callback في `pinmux_ops`.

---

#### `pinmux_generic_get_function_name()`

```c
const char *pinmux_generic_get_function_name(struct pinctrl_dev *pctldev,
                                              unsigned int selector);
```

بتعمل `radix_tree_lookup()` بالـ selector وبترجع `function->func->name`.

**Return:**
- الاسم عند النجاح
- `NULL` لو الـ selector مش موجود في الـ radix tree

---

#### `pinmux_generic_get_function_groups()`

```c
int pinmux_generic_get_function_groups(struct pinctrl_dev *pctldev,
                                       unsigned int selector,
                                       const char * const **groups,
                                       unsigned int * const ngroups);
```

بتعمل lookup بالـ selector وبتملي `*groups` و`*ngroups` من `function->func->groups` و`function->func->ngroups`.

**Return:**
- `0` عند النجاح
- `-EINVAL` لو الـ selector مش موجود

---

#### `pinmux_generic_get_function()`

```c
const struct function_desc *pinmux_generic_get_function(
    struct pinctrl_dev *pctldev, unsigned int selector);
```

بترجع الـ `function_desc` بالكامل بدلاً من جزء منه. مفيدة لـ drivers اللي عايزة تصل لـ `function->data` (الـ driver-specific data).

---

#### `pinmux_generic_function_is_gpio()`

```c
bool pinmux_generic_function_is_gpio(struct pinctrl_dev *pctldev,
                                     unsigned int selector);
```

بتفحص الـ flag `PINFUNCTION_FLAG_GPIO` في `function->func->flags`. دي اللي بتخلي الـ strict mode يشتغل صح مع الـ GPIO functions.

**Return:**
- `true` لو `PINFUNCTION_FLAG_GPIO` مضبوط
- `false` في أي حالة تانية بما فيها لو الـ selector مش موجود

---

#### `pinmux_generic_add_function()`

```c
int pinmux_generic_add_function(struct pinctrl_dev *pctldev,
                                const char *name,
                                const char * const *groups,
                                const unsigned int ngroups,
                                void *data);
```

Convenience wrapper بتبني `pinfunction` struct باستخدام ماكرو `PINCTRL_PINFUNCTION()` وبتبعته لـ `pinmux_generic_add_pinfunction()`.

---

#### `pinmux_generic_add_pinfunction()`

```c
int pinmux_generic_add_pinfunction(struct pinctrl_dev *pctldev,
                                   const struct pinfunction *func, void *data);
```

الـ core function للإضافة. المراحل:
1. بتتحقق لو الـ function موجودة بالفعل بالاسم (idempotent) → بترجع الـ selector الموجود
2. بتخصص `function_desc` بـ `devm_kzalloc()` — مرتبطة بـ lifetime الـ device
3. بتعمل `devm_kmemdup_const()` للـ `pinfunction` struct
4. بتدرجها في الـ `pin_function_tree` بالـ selector التالي
5. بتزوّد `num_functions`

**Key Details:**
- استخدام `devm_*` بدل `kzalloc` عادي — تعمل cleanup تلقائياً عند `device_unregister()`
- الكود نفسه فيه تعليق يعترف إن استخدام `devres` في subsystem core مش أفضل ممارسة لكنه موجود كـ technical debt
- الـ `func` pointer بيتنسخ بـ `devm_kmemdup_const()` — الـ caller ممكن يمرر pointer لـ stack variable

**Return:**
- الـ selector الجديد (أو الموجود لو الاسم مكرر)
- `-ENOMEM` أو error من `radix_tree_insert()`

---

#### `pinmux_generic_remove_function()`

```c
int pinmux_generic_remove_function(struct pinctrl_dev *pctldev,
                                   unsigned int selector);
```

بتحذف function بالـ selector من الـ radix tree وبتعمل `devm_kfree()` للـ descriptor. الـ `pinfunction` struct نفسه مش محتاج explicit free لأنه اتخصص بـ `devm_kmemdup_const()`.

**Key Details:**
- الـ comment في الكود يقول "the caller must take care of locking" — مش thread-safe بدون لوك خارجي

---

#### `pinmux_generic_free_functions()`

```c
void pinmux_generic_free_functions(struct pinctrl_dev *pctldev);
```

بتمسح كل الـ radix tree بـ `radix_tree_for_each_slot()` وتضبط `num_functions = 0`. مش بتعمل `devm_kfree()` للعناصر لأن الـ `devres` هيتولى ده عند device teardown.

**Caller:** `pinctrl_unregister()` في `core.c`

---

### مجموعة 6: DebugFS

---

#### `pinmux_init_device_debugfs()`

```c
void pinmux_init_device_debugfs(struct dentry *devroot,
                                struct pinctrl_dev *pctldev);
```

بتنشئ تلاتة ملفات تحت `/sys/kernel/debug/pinctrl/<device>/`:

| الملف | الصلاحيات | المحتوى |
|---|---|---|
| `pinmux-functions` | `0444` | قائمة الـ functions والـ groups |
| `pinmux-pins` | `0444` | حالة كل pin (owner, mux, gpio) |
| `pinmux-select` | `0200` | write-only لتفعيل mux يدوياً |

---

#### `pinmux_show_map()`

```c
void pinmux_show_map(struct seq_file *s, const struct pinctrl_map *map);
```

بتطبع `group` و`function` من الـ map entry. لو الـ group مش محدد بتطبع "(default)".

---

#### `pinmux_show_setting()`

```c
void pinmux_show_setting(struct seq_file *s,
                         const struct pinctrl_setting *setting);
```

بتطبع اسم الـ group والـ function مع selectors الرقمية للـ active setting.

---

#### `pinmux_select_write()` (static)

```c
static ssize_t pinmux_select_write(struct file *file,
                                   const char __user *user_buf,
                                   size_t len, loff_t *ppos);
```

تسمح بـ manual mux selection من userspace عن طريق كتابة `"group_name function_name"` في ملف `pinmux-select`. مفيدة جداً لـ debugging في الـ development.

**Flow:**
```
write("uart0_grp uart") ->
  memdup_user_nul() -> parse gname, fname
  pinmux_func_name_to_selector(fname) -> fsel
  pmxops->get_function_groups(fsel) -> validate gname
  pinctrl_get_group_selector(gname) -> gsel
  pmxops->set_mux(pctldev, fsel, gsel)
```

**Key Details:**
- الـ `pinmux_select_show()` ترجع `-EPERM` دايماً — الـ read مش مدعوم
- بتكسر الـ normal ownership model — لا يوجد `pin_request()` هنا
- تستخدمها للـ debugging فقط مش للـ production

---

### الـ `struct pinmux_ops` — الـ Driver Contract

```c
struct pinmux_ops {
    /* Optional: can driver allow/deny a pin being muxed? */
    int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);
    int (*free)(struct pinctrl_dev *pctldev, unsigned int offset);

    /* Mandatory: enumerate functions */
    int (*get_functions_count)(struct pinctrl_dev *pctldev);
    const char *(*get_function_name)(struct pinctrl_dev *pctldev,
                                     unsigned int selector);
    int (*get_function_groups)(struct pinctrl_dev *pctldev,
                               unsigned int selector,
                               const char * const **groups,
                               unsigned int *num_groups);

    /* Optional but recommended for strict mode */
    bool (*function_is_gpio)(struct pinctrl_dev *pctldev,
                             unsigned int selector);

    /* Mandatory: apply mux to hardware */
    int (*set_mux)(struct pinctrl_dev *pctldev,
                   unsigned int func_selector,
                   unsigned int group_selector);

    /* Optional: GPIO-specific mux acceleration */
    int (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                               struct pinctrl_gpio_range *range,
                               unsigned int offset);
    void (*gpio_disable_free)(struct pinctrl_dev *pctldev,
                              struct pinctrl_gpio_range *range,
                              unsigned int offset);
    int (*gpio_set_direction)(struct pinctrl_dev *pctldev,
                              struct pinctrl_gpio_range *range,
                              unsigned int offset,
                              bool input);

    /* strict: prevent simultaneous GPIO + mux use of same pin */
    bool strict;
};
```

| Callback | إلزامي؟ | الاستخدام |
|---|---|---|
| `request` | اختياري | الـ driver يرفض أو يقبل حجز الـ pin |
| `free` | اختياري | تحرير ما عمله `request` |
| `get_functions_count` | **إلزامي** | عدد الـ functions |
| `get_function_name` | **إلزامي** | اسم function بالـ selector |
| `get_function_groups` | **إلزامي** | groups بتاعة function |
| `function_is_gpio` | اختياري | للـ strict mode |
| `set_mux` | **إلزامي** | كتابة الـ HW registers |
| `gpio_request_enable` | اختياري | GPIO mux acceleration |
| `gpio_disable_free` | اختياري | عكس `gpio_request_enable` |
| `gpio_set_direction` | اختياري | ضبط direction على الـ pinmux level |
| `strict` | flag | منع dual ownership |

---

### الـ `struct function_desc` (CONFIG_GENERIC_PINMUX_FUNCTIONS)

```c
struct function_desc {
    const struct pinfunction *func;  /* name, groups[], ngroups */
    void *data;                      /* driver-specific opaque data */
};
```

الـ generic layer بتخزن هذا الـ struct في `pctldev->pin_function_tree` (radix tree indexed by selector). الـ `data` field ده بيتوصله لـ driver وممكن يبقى pointer لـ register addresses أو أي بيانات خاصة بالـ SoC.

---

### تفاعل الـ Subsystems — Big Picture

```
GPIO Subsystem              pinctrl core              pin controller driver
     |                           |                           |
     |-- gpio_request()          |                           |
     |    |                      |                           |
     |    +-- pinmux_request_gpio()                         |
     |         |                 |                           |
     |         +-- pin_request() --> ops->gpio_request_enable() --> [HW]
     |                           |                           |
Device Driver                    |                           |
     |                           |                           |
     |-- pinctrl_select_state()  |                           |
     |    |                      |                           |
     |    +-- pinmux_enable_setting()                        |
     |         |-- pin_request() x N pins                   |
     |         +-- ops->set_mux() --------------------------> [HW regs]
     |                           |                           |
     |-- devm cleanup            |                           |
          |                      |                           |
          +-- pinmux_disable_setting()                       |
               +-- pin_free() x N --> ops->free() --------> [HW]
```
## Phase 5: دليل الـ Debugging الشامل

### Software Level

#### 1. مدخلات الـ debugfs

الـ pinmux بيسجل مدخلاته في `/sys/kernel/debug/pinctrl/<controller-name>/` أوتوماتيك لما بتفعّل `CONFIG_DEBUG_FS`. الدالة `pinmux_init_device_debugfs()` بتعمل ٣ ملفات:

| المدخل | الـ Permission | الوظيفة |
|--------|----------------|---------|
| `pinmux-functions` | 0444 | بيعرض كل الـ functions والـ groups المرتبطة بيها |
| `pinmux-pins` | 0444 | بيعرض حالة كل pin — مين الـ owner وهل هو HOG |
| `pinmux-select` | 0200 | بيخليك تعمل `set_mux()` يدوي وقت الـ debug |

**قراءة الـ functions:**

```bash
# اعرف اسم الـ controller الأول
CTRL=$(ls /sys/kernel/debug/pinctrl/ | head -1)
cat /sys/kernel/debug/pinctrl/$CTRL/pinmux-functions
```

مثال output:

```
function 0: gpio,  groups = [ gpio0 gpio1 gpio2 ]
function 1: uart0, groups = [ uart0_grp ]
function 2: spi0,  groups = [ spi0_grp spi0_cs_grp ]
```

**قراءة حالة الـ pins:**

```bash
cat /sys/kernel/debug/pinctrl/$CTRL/pinmux-pins
```

مثال output لـ strict controller:

```
Pinmux settings per pin
Format: pin (name): mux_owner|gpio_owner (strict) hog?
pin 12 (GPIO12): device serial0 function uart0 group uart0_grp
pin 13 (GPIO13): device serial0 function uart0 group uart0_grp
pin 20 (GPIO20): GPIO gpiochip0:20
pin 25 (GPIO25): UNCLAIMED
```

مثال output لـ non-strict controller:

```
Format: pin (name): mux_owner gpio_owner hog?
pin 10 (GPIO10): serial0 (GPIO UNCLAIMED) function uart0 group uart0_grp
pin 11 (GPIO11): (MUX UNCLAIMED) gpiochip0:11
```

**تفعيل mux يدويًا عبر pinmux-select (الصيغة: `<group> <function>`):**

```bash
echo "uart0_grp uart0" > /sys/kernel/debug/pinctrl/$CTRL/pinmux-select
# لو نجح: مفيش output
# لو فشل: راجع dmesg
dmesg | tail -5
```

**عرض الـ maps وباقي المعلومات:**

```bash
cat /sys/kernel/debug/pinctrl/$CTRL/maps        # كل الـ pinctrl maps المسجلة
cat /sys/kernel/debug/pinctrl/$CTRL/pins        # كل الـ pins المسجلة مع أرقامها
cat /sys/kernel/debug/pinctrl/$CTRL/pingroups   # الـ pin groups وأعضاؤها
```

---

#### 2. مدخلات الـ sysfs

الـ pinmux نفسه ما بيعرضش entries مباشرة في sysfs، لكن المعلومات متاحة من خلال:

| المسار | المعلومة |
|--------|---------|
| `/sys/class/gpio/export` | طلب GPIO pin من user space — بيستدعي `pinmux_request_gpio()` |
| `/sys/class/gpio/gpioN/direction` | تحقق إن الـ direction اتسطّ صح بعد `pinmux_gpio_direction()` |
| `/sys/bus/platform/devices/<dev>/driver` | تأكد إن الـ pinctrl driver اتحمل |

```bash
# تصدير GPIO 20 ومراقبة حالتها في pinmux-pins
echo 20 > /sys/class/gpio/export
cat /sys/class/gpio/gpio20/direction
cat /sys/kernel/debug/pinctrl/$CTRL/pinmux-pins | grep "pin 20"
# المفروض تلاقي: pin 20 (GPIO20): GPIO gpiochip0:20
```

```bash
# استخدام libgpiod (الطريقة الحديثة)
gpiodetect                 # يعرض كل GPIO controllers
gpioinfo gpiochip0         # يعرض حالة كل pin في الـ chip
gpioget gpiochip0 14       # يقرأ قيمة pin
gpiomon gpiochip0 14       # يراقب edge events
```

---

#### 3. الـ ftrace — function tracing على دوال الـ pinmux

الـ pinmux ما بيوفرش tracepoints مخصصة، لكن **function tracing** على دواله الأساسية بيكشف كل حاجة:

```bash
T=/sys/kernel/tracing
echo 0 > $T/tracing_on
echo function > $T/current_tracer

# فلترة على دوال الـ pinmux الأساسية
echo 'pin_request pin_free pinmux_enable_setting pinmux_disable_setting \
      pinmux_request_gpio pinmux_free_gpio pinmux_gpio_direction' \
  > $T/set_ftrace_filter

echo 1 > $T/tracing_on
# شغّل العملية اللي بتسبب المشكلة هنا ...
echo 0 > $T/tracing_on
cat $T/trace
echo nop > $T/current_tracer   # cleanup
```

**تتبع call chain كامل عبر function_graph:**

```bash
echo function_graph > $T/current_tracer
echo 'pinmux_enable_setting' > $T/set_graph_function
echo 1 > $T/tracing_on
```

مثال output:

```
 1)               |  pinmux_enable_setting() {
 1)               |    pin_request() {
 1)   0.312 us    |      pin_desc_get();
 1)   1.204 us    |      ops->gpio_request_enable();
 1)               |    }
 1)               |    ops->set_mux();
 1)   3.891 us    |  }
```

**تتبع الـ driver-specific set_mux مباشرة:**

```bash
# مثال لـ sunxi pinctrl driver
echo 'sunxi_pmx_set_mux' > $T/set_ftrace_filter
echo 1 > $T/tracing_on
```

---

#### 4. الـ printk والـ Dynamic Debug

الـ `pinmux.c` بيستخدم `pr_fmt` بـ prefix `"pinmux core: "` وبيستخدم `dev_dbg` للرسائل العادية.

```bash
# تفعيل كل dev_dbg في ملف pinmux.c
echo 'file drivers/pinctrl/pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع line numbers وأسماء الدوال والـ module
echo 'file drivers/pinctrl/pinmux.c +pflm' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل الـ pinctrl subsystem
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق مما تم تفعيله
grep pinmux /sys/kernel/debug/dynamic_debug/control
```

أهم رسالة `dev_dbg` في الكود:

```c
/* في pin_request() — بتظهر عند كل طلب pin */
dev_dbg(pctldev->dev, "request pin %d (%s) for %s\n",
    pin, desc->name, owner);
```

```bash
# مراقبة مستمرة
dmesg -w | grep "pinmux core:"
dmesg -w | grep "pin.*requested"
```

**تفعيل من الـ boot:**

```
# في kernel command line
dyndbg="file drivers/pinctrl/pinmux.c +p" loglevel=8
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| الـ Config | الوظيفة |
|-----------|---------|
| `CONFIG_PINMUX` | الـ pinmux subsystem الأساسي — لازم يكون مفعّل |
| `CONFIG_DEBUG_FS` | لازم عشان تتفعّل ملفات debugfs الثلاثة |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | الـ generic implementation باستخدام radix tree |
| `CONFIG_PROVE_LOCKING` | بيكتشف deadlocks في `desc->mux_lock` mutex |
| `CONFIG_DEBUG_MUTEXES` | تتبع mutex ownership في `mux_lock` |
| `CONFIG_LOCKDEP` | تتبع تسلسل الـ lock acquisition |
| `CONFIG_DYNAMIC_DEBUG` | لازم مفعّل عشان `dev_dbg` يشتغل runtime |
| `CONFIG_KALLSYMS` | مهم لقراءة stack traces بأسماء دوال مفهومة |
| `CONFIG_FRAME_POINTER` | بيحسّن جودة الـ stack traces |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'PINMUX|PINCTRL|DEBUG_FS|DYNAMIC_DEBUG|PROVE_LOCKING'
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# عرض كل الـ debugfs entries للـ pinctrl subsystem
ls /sys/kernel/debug/pinctrl/
ls /sys/kernel/debug/pinctrl/$CTRL/

# script شامل لتفريغ حالة كل الـ controllers
#!/bin/bash
PINCTRL_DIR="/sys/kernel/debug/pinctrl"
for ctrl in "$PINCTRL_DIR"/*/; do
    name=$(basename "$ctrl")
    echo "=== Controller: $name ==="
    echo "--- Functions ---"
    cat "$ctrl/pinmux-functions" 2>/dev/null
    echo "--- Pin States ---"
    cat "$ctrl/pinmux-pins" 2>/dev/null
    echo ""
done
```

```bash
# libgpiod tools
gpiodetect                         # يعرض كل GPIO chips
gpioinfo gpiochip0                 # يعرض pin states مع الـ names
gpioset --mode=time --sec=2 gpiochip0 14=1   # يضبط pin لمدة معينة

# python-gpiod
python3 -c "
import gpiod
chip = gpiod.Chip('gpiochip0')
info = chip.get_info()
print(f'Chip: {info.name}, {info.num_lines} lines')
for i in range(min(20, info.num_lines)):
    linfo = chip.get_line_info(i)
    print(f'  Line {i}: {linfo.name}, consumer={linfo.consumer}, used={linfo.used}')
"
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| رسالة الخطأ | المعنى | الحل |
|-------------|--------|------|
| `pinmux ops lacks necessary functions` | الـ `pinmux_ops` ناقص فيه واحدة أو أكتر من: `get_functions_count`، `get_function_name`، `get_function_groups`، `set_mux` | افحص `struct pinmux_ops` في الـ driver وتأكد إن الـ 4 callbacks موجودين |
| `pinmux ops has no name for function%u` | الـ `get_function_name()` بيرجع NULL لـ selector معين | الـ driver لازم يرجع اسم valid لكل function من 0 لـ nfuncs-1 |
| `failed to register map %s (%d): no function given` | الـ `pinctrl_map` entry ليهاش `data.mux.function` | في الـ DT أو board file، تأكد إن `function` property موجودة في كل mux entry |
| `invalid function %s in map table` | الـ function name في الـ map مش موجود في الـ controller | قارن الأسماء في الـ DT مع اللي بيرجعه `get_function_name()` في الـ driver |
| `can't query groups for function %s` | `get_function_groups()` فشل | تحقق من الـ driver implementation |
| `function %s can't be selected on any group` | `get_function_groups()` رجع `ngroups = 0` | الـ function محتاجة group واحدة على الأقل |
| `invalid group "%s" for function "%s"` | الـ group المحددة في الـ map مش ضمن groups الـ function | تحقق من الـ DT — `pinctrl-0` بيشاور على group موجودة؟ |
| `pin %s already requested by %s; cannot claim for %s` | الـ pin محجوز من جهة تانية — conflict | افحص `pinmux-pins` وشوف مين حاجز الـ pin، وراجع الـ DT |
| `pin %d is not registered so it cannot be requested` | الـ pin number مش موجود في الـ `pin_desc` | الـ controller لازم يسجل الـ pin ده في `pinctrl_register_pins()` |
| `could not increase module refcount for pin %d` | الـ pinctrl module بيتم unload وقت الـ request | race condition في module lifecycle |
| `could not request pin %d (%s) from group %s on device %s` | `pin_request()` فشل لـ pin داخل group | الرسالة قبليها بتوضح السبب — غالبًا pin conflict |
| `not freeing pin %d (%s) as part of deactivating group %s` | الـ `mux_setting` pointer اتغير من تحت الـ function | race condition أو double-free في الـ setting lifecycle |
| `does not support mux function` | الـ `pctldev->desc->pmxops` = NULL | الـ controller driver مش بيدعم pinmux أصلاً |
| `pin is not registered so it cannot be freed` | محاولة تحرير pin مش مسجلة | bug في الـ driver — `pin_free()` اتحلت بـ pin number غلط |

---

#### 8. نقاط استراتيجية لـ dump_stack() وWARN_ON()

الـ kernel بيستخدم `WARN_ON` في `pin_free()` أصلاً:

```c
/* في pin_free() — موجودة في الكود الأصلي */
if (WARN_ON(!desc->mux_usecount))
    return NULL;
/* بتطلع warning لو حاولت تحرر pin أكتر من ما طلبتها */
```

**نقاط إضافية مفيدة للـ debugging:**

```c
/* في pin_request() — لمعرفة مين طلب pin معينة */
if (pin == TARGET_PIN) {
    dev_warn(pctldev->dev, "pin %d requested by: %s\n", pin, owner);
    dump_stack(); /* يطبع الـ call stack كامل — مفيد لاكتشاف من أين جاء الطلب */
}

/* في pinmux_enable_setting() — تتبع فشل set_mux */
ret = ops->set_mux(pctldev, setting->data.mux.func,
                   setting->data.mux.group);
if (WARN_ON(ret)) {
    dump_stack();
    goto err_set_mux;
}

/* تحقق من consistency الـ mux_usecount */
WARN_ON(desc->mux_usecount < 0);

/* تحقق إن الـ owner مش NULL لما يكون usecount > 0 */
WARN_ON(desc->mux_usecount > 0 && !desc->mux_owner);

/* في pinmux_disable_setting() — الـ dev_warn موجودة في الكود */
/* ممكن تضيف dump_stack() هنا لمعرفة الـ setting التاني */
dev_warn(pctldev->dev,
         "not freeing pin %d (%s) as part of deactivating group %s"
         " - it is already used for some other setting",
         pins[i], desc->name, gname);
/* أضف: dump_stack(); */
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware تطابق حالة الـ Kernel

الهدف: التأكد إن الـ MUX register في الـ SoC فعلاً اتغير بعد ما `set_mux()` اتنفّذ.

```bash
# الخطوة 1: اعرف الـ kernel state
cat /sys/kernel/debug/pinctrl/$CTRL/pinmux-pins | grep "pin 14"
# output: pin 14 (GPIO14): device uart0 function uart0 group uart0_grp

# الخطوة 2: اقرأ الـ MUX register من الـ hardware مباشرة
# مثال Raspberry Pi: GPFSEL1 يتحكم في GPIO10-19
devmem2 0x3F200004 w
# Output: 0x00024000 ← bits[14:12] = 100 (ALT0 = UART) ← GPIO14 ✓

# الخطوة 3: قارن
# الـ kernel قال UART + الـ register بيقول ALT0 → متطابقين
# الـ kernel قال UART + الـ register بيقول 000 (GPIO Input) → set_mux() فشل أو مش اتنفّذ
```

---

#### 2. تقنيات الـ Register Dump

```bash
# devmem2 — أسهل طريقة لقراءة registers
devmem2 0x3F200004 w    # قراءة word (32-bit)

# busybox devmem (بديل)
busybox devmem 0x3F200004 32

# io utility
io -4 -r 0x3F200004

# قراءة block من الـ registers دفعة واحدة
BASE=0x3F200000   # BCM2835 GPIO base
for offset in 0 4 8 12 16 20; do
    ADDR=$(printf "0x%08x" $((BASE + offset)))
    VAL=$(devmem2 $ADDR w 2>/dev/null | awk '/Value at/ {print $NF}')
    echo "REG[$ADDR] = $VAL"
done
```

**عبر /dev/mem وPython:**

```bash
python3 << 'EOF'
import mmap, struct

BASE = 0x3F200000  # BCM2835 GPIO base
with open('/dev/mem', 'r+b') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=BASE)
    gpfsel1 = struct.unpack('<I', m[4:8])[0]  # GPFSEL1 offset=4
    print(f"GPFSEL1 = 0x{gpfsel1:08x}")
    # GPIO14: bits [14:12]
    gpio14_func = (gpfsel1 >> 12) & 0x7
    funcs = {0:'INPUT', 1:'OUTPUT', 4:'ALT0(UART)', 5:'ALT1', 6:'ALT2', 7:'ALT3', 3:'ALT4', 2:'ALT5'}
    print(f"GPIO14 = {funcs.get(gpio14_func, hex(gpio14_func))}")
    m.close()
EOF
```

**جدول تفسير bits (generic 3-bit mux field — مثال BCM2835):**

| قيمة الـ bits | الوظيفة |
|--------------|---------|
| `000` | GPIO Input |
| `001` | GPIO Output |
| `100` | ALT0 — الـ function الأساسية للـ pin |
| `101` | ALT1 |
| `110` | ALT2 |
| `111` | ALT3 |
| `011` | ALT4 |
| `010` | ALT5 |

---

#### 3. Logic Analyzer وOscilloscope

**متى تستخدم Logic Analyzer:**

- التحقق إن الـ function signal ظهر فعلاً على الـ pin بعد الـ mux activation.
- لو الـ kernel بيقول الـ pin = UART لكن مفيش data بتطلع.
- الكشف عن glitches أثناء الـ mux switching.

```
Setup أساسي:
Pin Under Test ──────────── Channel 0 (Logic Analyzer)
GND ─────────────────────── GND (Logic Analyzer)

Sample Rate: >= 10x أعلى frequency متوقعة
مثال: UART 115200 baud → sample rate >= 1.15MHz
```

```bash
# sigrok/pulseview — تحليل UART signal
sigrok-cli -d fx2lafw --config samplerate=2m \
  --channels D0=TX,D1=RX \
  --time 100ms \
  -P uart:baudrate=115200:tx=TX:rx=RX \
  -A uart=rx-data,tx-data
```

**قراءة نتيجة Oscilloscope:**

- الـ signal موجود بـ voltage levels صح (3.3V) → الـ mux اشتغل
- الـ signal موجود لكن levels غلط → مشكلة في الـ pull-up/pull-down — مش pinmux
- الـ pin flat HIGH طول الوقت → الـ MUX register مش اتكتب صح أو الـ UART TX idle state

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | الـ Kernel Log Pattern | التفسير والحل |
|---------|----------------------|--------------|
| Pin conflict بين driverين | `pin GPIO14 already requested by uart0; cannot claim for gpiochip0:14` | دriverين بيطلبوا نفس الـ pin — راجع الـ DT |
| `set_mux()` بيرجع 0 لكن الـ hardware مش بيستجيب | لا error | الـ driver بيكتب في address غلط، أو الـ peripheral clock متوقف |
| HOG pin بيتعارض مع consumer driver | `pin GPIO5 already requested by pinctrl-device; cannot claim for spi0` | الـ hog في الـ DT محتجز الـ pin — عدّل الـ DT |
| الـ mux صح لكن مفيش signal | لا error في pinmux | افحص الـ clock gating للـ peripheral والـ power domain |
| الـ pin بيرجع لـ GPIO بعد suspend/resume | `not freeing pin... already used for some other setting` | race بين suspend/resume وـ pinmux disable/enable |
| strict controller وGPIO request على pin مشغولة | `pin GPIO10 already requested by i2c0; cannot claim for gpiochip0:10` | الـ controller `strict=true` — الـ pin مش ممكن تتشارك |

---

#### 5. الـ Device Tree Debugging

```bash
# عرض الـ DT المُحمَّل فعلاً
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 "pinctrl"

# أو باستخدام fdtdump على الـ DTB
fdtdump /boot/bcm2835-rpi-b.dtb | grep -A 10 "uart0_pins"

# تحقق من الـ pinctrl properties على device معينة
cat /proc/device-tree/soc/uart@7e201000/pinctrl-names

# عرض الـ maps المسجلة — يتحقق من إن الـ DT parsing نجح
cat /sys/kernel/debug/pinctrl/$CTRL/maps | grep uart0
```

**مثال DT صح لـ UART pinmux:**

```dts
/* في الـ SoC .dtsi */
uart0_pins: uart0-pins {
    pins = "GPIO14", "GPIO15";
    function = "uart0";   /* لازم يطابق get_function_name() في الـ driver */
};

/* في الـ board .dts */
&uart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart0_pins>;
    status = "okay";
};
```

**أخطاء DT شائعة ونتائجها:**

```
# خطأ 1: function name مختلف
DT:     function = "uart";           /* "uart" */
driver: get_function_name() → "uart0"  /* "uart0" */
نتيجة في dmesg: invalid function uart in map table

# خطأ 2: group name غلط
DT:     groups = "uart0_grp";        /* underscore */
driver: returns "uart0-grp"          /* hyphen */
نتيجة: invalid group "uart0_grp" for function "uart0"

# خطأ 3: ناسي pinctrl-names
# → pinctrl_bind_pins() مش هتتعمل
# → الـ device بيشتغل بالـ default hardware state من الـ bootloader
```

```bash
# تحقق إن الـ pinctrl bind اشتغل للـ device
cat /sys/kernel/debug/pinctrl/$CTRL/pinmux-pins | grep -i "uart\|spi\|i2c"
# لو الـ device موجود → الـ DT mapping صح
# لو مش موجود → راجع pinctrl-0 / pinctrl-names في الـ DT
```

---

### Practical Commands

#### 1. الفحص الشامل السريع

```bash
#!/bin/bash
# quick-pinmux-debug.sh
CTRL=$(ls /sys/kernel/debug/pinctrl/ | head -1)
BASE="/sys/kernel/debug/pinctrl/$CTRL"
echo "=== Controller: $CTRL ==="

echo ""
echo "=== Available Functions ==="
cat "$BASE/pinmux-functions"

echo ""
echo "=== Pin States ==="
cat "$BASE/pinmux-pins"

echo ""
echo "=== Active Pins (not UNCLAIMED) ==="
cat "$BASE/pinmux-pins" | grep -v UNCLAIMED

echo ""
echo "=== HOG Pins ==="
cat "$BASE/pinmux-pins" | grep HOG

echo ""
echo "=== Map Table ==="
cat "$BASE/maps" 2>/dev/null || echo "(maps not available)"
```

---

#### 2. البحث عن pin conflict

```bash
# ابحث عن pins مشغولة في كل الـ controllers
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -v UNCLAIMED

# ابحث عن pins محجوزة من device وGPIO في نفس الوقت
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep "GPIO"

# تشخيص conflict خطوة بخطوة
# الخطأ: "pin GPIO5 already requested by i2c0; cannot claim for spi0"

# الخطوة 1: مين حاجز الـ pin؟
cat /sys/kernel/debug/pinctrl/$CTRL/pinmux-pins | grep "pin 5"

# الخطوة 2: مين يفترض يستخدمها؟
cat /sys/kernel/debug/pinctrl/$CTRL/maps | grep "pin 5\|GPIO5"

# الخطوة 3: افحص الـ DT
dtc -I fs /proc/device-tree 2>/dev/null | grep -B5 -A5 "GPIO5\|pin 5"

# الخطوة 4: هل i2c0 المفروض يكون disabled؟
cat /proc/device-tree/soc/i2c*/status 2>/dev/null
```

---

#### 3. تفعيل dynamic debug أثناء الـ probe

```bash
# قبل ربط الـ driver
echo 'file drivers/pinctrl/pinmux.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شغّل الـ driver
echo "soc:pinctrl" > /sys/bus/platform/drivers/my-pinctrl/bind

# تابع الـ log
dmesg -w | grep -E "pinmux|pinctrl"
```

---

#### 4. اختبار mux يدوي عبر pinmux-select

```bash
CTRL=$(ls /sys/kernel/debug/pinctrl/ | head -1)
BASE="/sys/kernel/debug/pinctrl/$CTRL"

# اعرف الـ functions والـ groups المتاحة
cat "$BASE/pinmux-functions"

# اعمل set_mux يدوي — الصيغة: "<group> <function>"
echo "uart0_grp uart0" > "$BASE/pinmux-select"

# تحقق إن الـ setting اتطبق
cat "$BASE/pinmux-pins" | grep -i uart

# لو فشل: راجع dmesg
dmesg | tail -10
```

---

#### 5. تفريغ الـ registers عبر devmem2

```bash
#!/bin/bash
# dump-pinctrl-regs.sh — يفرغ block من الـ pin control registers
BASE_ADDR=0x3F200000   # غيّر لعنوان الـ SoC بتاعك (BCM2835 GPIO base هنا)
COUNT=8                # عدد الـ GPFSEL registers (GPFSEL0-GPFSEL5 + padding)

echo "=== GPIO Function Select Registers ==="
for i in $(seq 0 $((COUNT-1))); do
    ADDR=$(printf "0x%08X" $((BASE_ADDR + i * 4)))
    VAL=$(devmem2 $ADDR w 2>/dev/null | awk '/Value at/ {print $NF}')
    echo "GPFSEL$i [$ADDR] = $VAL"
done
```

مثال output:

```
=== GPIO Function Select Registers ===
GPFSEL0 [0x3F200000] = 0x00000000   ← GPIO0-9 كلهم inputs
GPFSEL1 [0x3F200004] = 0x00024000   ← GPIO14,15 = ALT0 (UART)
GPFSEL2 [0x3F200008] = 0x00000924   ← GPIO20-23 = ALT0 (PCM)
```

---

#### 6. ftrace كامل على دوال الـ pinmux

```bash
T=/sys/kernel/tracing
echo 0 > $T/tracing_on
echo function > $T/current_tracer
echo 'pin_request pin_free pinmux_enable_setting pinmux_disable_setting \
      pinmux_request_gpio pinmux_free_gpio' > $T/set_ftrace_filter
echo 1 > $T/tracing_on

# أجرِ العملية المطلوبة هنا
echo 1 > /sys/class/gpio/export  # مثلاً

echo 0 > $T/tracing_on
cat $T/trace | grep -E "pin_request|pinmux"
echo nop > $T/current_tracer   # cleanup دايمًا
```

---

#### 7. مراقبة الـ kernel log في real-time

```bash
# تفعيل dynamic debug
echo 'file drivers/pinctrl/pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control

# مراقبة مستمرة
dmesg -w | grep -E "pinmux core:|pin.*requested|cannot claim"

# أو باستخدام journalctl
journalctl -k -f | grep -i "pinmux\|pinctrl"
```

---

#### 8. تفسير pinmux-pins بالتفصيل

```
pin 12 (GPIO12): device serial0 (HOG) function uart0 group uart0_grp
│    │   │        │       │       │           │              │
│    │   │        │       │       │           │              └─ group name
│    │   │        │       │       │           └─ function name
│    │   │        │       │       └─ controller هو اللي طلب الـ pin (HOG)
│    │   │        │       └─ mux_owner = الـ device اللي طالب الـ mux
│    │   │        └─ strict controller mode
│    │   └─ pin name
│    └─ pin number in global pin space
└─ keyword "pin"
```

```
pin 20 (GPIO20): (MUX UNCLAIMED) gpiochip0:20
                  └─ مفيش function طلب الـ pin
                                   └─ الـ GPIO subsystem طالب الـ pin
```

```
pin 25 (GPIO25): UNCLAIMED
                  └─ الـ pin حر تماماً — مش محجوز لأي function أو GPIO
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — UART مش شغال بعد bring-up

#### العنوان
**الـ UART2 مش بيظهر على gateway صناعي بعد تحويل الـ pinmux**

#### السياق
بنبني industrial gateway على RK3562 للتحكم في معدات المصنع. الـ gateway محتاج UART2 لاتصال RS-485 مع PLCs. الـ board جديدة وعملنا bring-up من zero.

#### المشكلة
بعد ما حطينا الـ device tree وبوتنا الكيرنل، `ttyS2` مش بيظهر خالص، وفي الـ dmesg:
```
pinmux core: pin-45 (uart2-tx): invalid function uart2 in map table
```

#### التحليل
الخطأ جاي من `pinmux_map_to_setting()`:

```c
ret = pinmux_func_name_to_selector(pctldev, map->data.mux.function);
if (ret < 0) {
    dev_err(pctldev->dev, "invalid function %s in map table\n",
        map->data.mux.function);
    return ret;
}
```

**الـ `pinmux_func_name_to_selector()`** بتلف على كل الـ functions المسجلة في الـ controller وتقارن الأسماء:

```c
while (selector < nfuncs) {
    const char *fname = ops->get_function_name(pctldev, selector);
    if (fname && !strcmp(function, fname))
        return selector;
    selector++;
}
return -EINVAL; /* ما لاقتش */
```

يعني الـ function name اللي في الـ DT مش متطابق مع اللي سجله driver الـ RK3562 pinctrl.

نشوف debugfs:
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rk3562/pinmux-functions
```
Output:
```
function 0: uart2m0, groups = [ uart2m0-xfer ]
function 1: uart2m1, groups = [ uart2m1-xfer ]
```

الـ function الصح اسمه `uart2m0` مش `uart2`.

#### الحل
نصلح الـ device tree:

```dts
/* خطأ */
pinctrl-0 = <&uart2_pins>;
&uart2_pins {
    function = "uart2";   /* ده غلط */
    groups = "uart2m0-xfer";
};

/* صح */
&uart2 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart2m0_xfer>;
    status = "okay";
};

&pinctrl {
    uart2 {
        uart2m0_xfer: uart2m0-xfer {
            rockchip,pins =
                <1 RK_PD0 4 &pcfg_pull_up>,   /* TX */
                <1 RK_PD1 4 &pcfg_pull_none>;  /* RX */
        };
    };
};
```

نتحقق بعد إصلاح:
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rk3562/pinmux-pins | grep -i uart
# pin 45 (gpio1-d0): device uart2: uart2m0 (uart2m0-xfer)
```

#### الدرس المستفاد
**الـ `pinmux_func_name_to_selector()`** بتعمل string comparison حرفية. أي فرق في الاسم بين الـ DT والـ driver بيرجع `-EINVAL`. دايمًا استخدم:
```bash
cat /sys/kernel/debug/pinctrl/<controller>/pinmux-functions
```
الأول عشان تعرف الأسماء الصح قبل ما تكتب الـ DT.

---

### السيناريو 2: Android TV Box على Allwinner H616 — conflict بين SPI وGPIO

#### العنوان
**الـ SPI flash مش بيشتغل وبيرجع "pin already requested"**

#### السياق
Android TV box على Allwinner H616. البورد بتستخدم SPI0 لـ NOR flash لتخزين configuration، ونفس الوقت عندنا GPIO expander بيستخدم بعض الـ pins دي عشان يتحكم في LED status.

#### المشكلة
الـ SPI driver بيفشل في probe وفي dmesg:
```
pinmux core: pin-34 (PC2): device spi0 already requested by gpio-leds; cannot claim for spi0
```

#### التحليل
الخطأ بيجي من `pin_request()`. الـ pin بيتأكد إن مفيش owner تاني بيمسكه:

```c
if ((!gpio_range || ops->strict) && !gpio_ok &&
    desc->mux_usecount && strcmp(desc->mux_owner, owner)) {
    dev_err(pctldev->dev,
        "pin %s already requested by %s; cannot claim for %s\n",
        desc->name, desc->mux_owner, owner);
    goto out;
}
```

الـ `gpio-leds` driver طلب PC2 كـ GPIO عشان يتحكم في LED. بعدين لما SPI driver جا يطلب نفس الـ pin جزء من group، `desc->mux_owner` مش NULL وهو مش نفس الـ owner فبيفشل.

نتحقق من الـ state:
```bash
cat /sys/kernel/debug/pinctrl/pio/pinmux-pins | grep PC2
# pin 34 (PC2): device gpio-leds (GPIO UNCLAIMED)
```

بعدين لما SPI حاول:
```bash
cat /sys/kernel/debug/pinctrl/pio/pinmux-pins | grep PC2
# pin 34 (PC2): gpio-leds spi0 CONFLICT
```

الـ H616 pinctrl driver بيعمل `ops->strict = true`، يعني مفيش sharing خالص.

#### الحل
نراجع الـ schematic، PC2 هو SPI0-MISO. الـ LED محتاج pin مختلف. نغير الـ LED لـ PC8 اللي مش مستخدمة:

```dts
/* قبل */
leds {
    compatible = "gpio-leds";
    status-led {
        gpios = <&pio 2 2 GPIO_ACTIVE_HIGH>; /* PC2 - conflict */
    };
};

/* بعد */
leds {
    compatible = "gpio-leds";
    status-led {
        gpios = <&pio 2 8 GPIO_ACTIVE_HIGH>; /* PC8 - free */
    };
};
```

لو مش عارفين أي pins حرة:
```bash
cat /sys/kernel/debug/pinctrl/pio/pinmux-pins | grep UNCLAIMED | head -20
```

#### الدرس المستفاد
**`pin_request()`** بتسجل الـ owner في `desc->mux_owner` أو `desc->gpio_owner`. مع `strict = true`، حتى لو الـ pin معمول mux لـ GPIO، مش ممكن حد تاني يطلبه. راجع الـ schematic دايمًا قبل تحديد الـ GPIO للـ LEDs والـ peripherals، وما تعتمدش على "بيشتغل في الـ reference board".

---

### السيناريو 3: IoT Sensor على STM32MP1 — الـ I2C بيتعطل بشكل عشوائي في production

#### العنوان
**الـ I2C3 بيتعطل بعد runtime PM cycle على STM32MP157**

#### السياق
IoT environmental sensor بيقيس temperature وhumidity كل 30 ثانية. الـ MCU بيدخل low-power sleep بين القراءات. المشكلة اتظهرت بعد 3 أيام من testing المتواصل.

#### المشكلة
بعد كذا دورة sleep/wake، الـ I2C3 بيتعطل ومش بيصحى. الـ log:
```
i2c i2c-3: timeout waiting for bus ready
pinmux core: not freeing pin 89 (PD13) as part of deactivating group i2c3-grp - it is already used for some other setting
```

#### التحليل
الرسالة دي بتيجي من `pinmux_disable_setting()`:

```c
scoped_guard(mutex, &desc->mux_lock)
    is_equal = (desc->mux_setting == &(setting->data.mux));

if (is_equal) {
    pin_free(pctldev, pins[i], NULL);
} else {
    dev_warn(pctldev->dev,
         "not freeing pin %d (%s) as part of deactivating group %s"
         " - it is already used for some other setting",
         pins[i], desc->name, gname);
}
```

الـ check بيقارن pointer: `desc->mux_setting == &(setting->data.mux)`. لو الـ setting object اتحركت في الـ memory أو اتعمل لها reallocate (مثلاً بسبب runtime PM قام بـ devm reallocation)، الـ pointer comparison هتفشل حتى لو الـ content نفسه.

لما الـ PM subsystem بيعمل `pinctrl_select_state("sleep")` فبيطلب enable لـ I2C sleep state، وبعدين لما بيصحى بيحاول `pinctrl_select_state("default")` — لكن الـ disable للـ sleep state فشل فالـ pin فضل محجوز للـ sleep config، وبعدين الـ default state ما اتطبقتش صح.

نشوف الحالة لحظة الـ bug:
```bash
cat /sys/kernel/debug/pinctrl/soc:pin-controller/pinmux-pins | grep PD13
# pin 89 (PD13): device i2c3-sleep (HOG) function i2c3 group i2c3-grp
```

المفروض يكون default مش sleep بعد الـ wake.

#### الحل
نضيف `pinctrl-1` لحالة الـ sleep بـ pins محايدة (input بدون function):

```dts
&i2c3 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&i2c3_pins_default>;
    pinctrl-1 = <&i2c3_pins_sleep>;
    status = "okay";
};

&pinctrl {
    i2c3_pins_default: i2c3-default {
        pins = "PD12", "PD13";
        function = "i2c3";
        bias-disable;
    };

    i2c3_pins_sleep: i2c3-sleep {
        /* في الـ sleep: خلي الـ pins input بدون function */
        pins = "PD12", "PD13";
        function = "gpio";
        bias-pull-up;
        input-enable;
    };
};
```

ونتأكد إن الـ PM callbacks بتستخدم `pinctrl_pm_select_sleep_state()` و `pinctrl_pm_select_default_state()` بالترتيب الصح.

#### الدرس المستفاد
**`pinmux_disable_setting()`** بتعمل pointer comparison مش value comparison. لو الـ setting object بيتغير عنوانه في الـ memory، الـ disable بيفشل صامت وبيسيب الـ pin محجوز. دايمًا حدد `pinctrl-names = "default", "sleep"` لأي peripheral بيدخل power management cycles.

---

### السيناريو 4: Automotive ECU على i.MX8MP — الـ CAN bus مش بيشتغل بعد driver load

#### العنوان
**الـ `set_mux` بيرجع error بعد ما `pinmux_check_ops` نجح على i.MX8MP**

#### السياق
Automotive ECU بيتحكم في نظام الفرامل. البورد على i.MX8MP. محتاجين CAN FD على `flexcan1`. الـ kernel بيبوت لكن `can0` مش بيظهر.

#### المشكلة
```
imx8mp-pinctrl: does not support mux function
flexcan 30bf0000.can: pinctrl: Error applying setting 1: -22
```

أو في حالة تانية:
```
pinmux core: pinmux ops lacks necessary functions
```

#### التحليل
الـ error الأول بيجي من `pinmux_map_to_setting()`:
```c
if (!pmxops) {
    dev_err(pctldev->dev, "does not support mux function\n");
    return -EINVAL;
}
```

الـ error التاني من `pinmux_check_ops()`:
```c
if (!ops ||
    !ops->get_functions_count ||
    !ops->get_function_name ||
    !ops->get_function_groups ||
    !ops->set_mux) {
    dev_err(pctldev->dev, "pinmux ops lacks necessary functions\n");
    return -EINVAL;
}
```

نشوف الـ driver الـ i.MX8MP pinctrl driver بيسجل الـ ops:

```bash
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pinmux-functions | grep can
# function 12: flexcan1, groups = [ flexcan1grp ]
# function 13: flexcan2, groups = [ flexcan2grp ]
```

الـ functions موجودة! إذن المشكلة مش في الـ check. نشوف الـ DT أكتر:

```bash
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pinmux-pins | grep can
# (nothing)
```

يعني الـ pinmux_enable_setting اتنفذت لكن `set_mux` فشل. نشوف الـ i.MX8MP driver:

```c
/* الـ set_mux function في driver الـ imx */
static int imx_pmx_set(struct pinctrl_dev *pctldev, unsigned selector,
                       unsigned group)
{
    /* بيكتب في IOMUXC registers */
    /* لو الـ clock للـ IOMUXC مش enabled بيفشل */
}
```

الـ IOMUXC clock مش معمله enable قبل الـ pinctrl registration.

#### الحل
نتأكد إن الـ IOMUXC clock enabled في الـ DT:

```dts
iomuxc: pinctrl@30330000 {
    compatible = "fsl,imx8mp-iomuxc";
    reg = <0x30330000 0x10000>;
    /* لازم الـ clock يكون enabled */
    clocks = <&clk IMX8MP_CLK_IOMUXC>;
    clock-names = "iomuxc";
};
```

وفي بعض الـ boards نحتاج نضيف `assigned-clocks` لو الـ default مش كافي.

نتحقق:
```bash
# نشوف لو الـ clock شغال
cat /sys/kernel/debug/clk/clk_summary | grep iomuxc

# نشوف الـ pinmux بعد الإصلاح
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pinmux-pins | grep can
# pin 120 (GPIO5_IO9): device flexcan1 function flexcan1 group flexcan1grp
```

#### الدرس المستفاد
**`pinmux_check_ops()`** بتتحقق من وجود الـ function pointers فقط. لو الـ `set_mux` موجودة لكن بتفشل internally (مثلاً clock مش enabled، أو register مش accessible)، الـ check بتنجح لكن الـ `pinmux_enable_setting()` بتفشل وقت `ops->set_mux()`. في الـ automotive context، اهتم بـ power domains وclocks قبل الـ pinctrl.

---

### السيناريو 5: Custom Board على AM62x — GPIO request يفشل بعد HDMI enable

#### العنوان
**GPIO لـ reset line بيفشل بعد ما HDMI يشتغل على AM62x**

#### السياق
Custom board للـ digital signage على TI AM62x. البورد بتعمل display على HDMI وبيتحكم في display controller خارجي عن طريق GPIO reset line. المشكلة بتظهر بعد enable الـ HDMI.

#### المشكلة
```
gpio gpiochip0: (pinctrl-am62x):
pinmux core: pin GPIO0_15 already requested by cdns_hdmi;
cannot claim for gpiochip0:15
```

الـ display controller reset مش بيشتغل وال HDMI screen مش بيبان.

#### التحليل
الـ AM62x pinctrl driver بيستخدم `ops->strict = true`. نفهم الـ flow:

أول حاجة، `pinmux_request_gpio()` بتتنفذ عشان الـ GPIO reset line:

```c
int pinmux_request_gpio(struct pinctrl_dev *pctldev,
                        struct pinctrl_gpio_range *range,
                        unsigned int pin, unsigned int gpio)
{
    const char *owner;
    int ret;

    owner = kasprintf(GFP_KERNEL, "%s:%d", range->name, gpio);
    if (!owner)
        return -ENOMEM;

    ret = pin_request(pctldev, pin, owner, range);
    ...
}
```

جوا `pin_request()`، مع `ops->strict`:

```c
if ((gpio_range || ops->strict) && !gpio_ok && desc->gpio_owner) {
    dev_err(pctldev->dev,
        "pin %s already requested by %s; cannot claim for %s\n",
        desc->name, desc->gpio_owner, owner);
    goto out;
}
```

الـ HDMI driver طلب GPIO0_15 كـ `gpio_owner` عشان HDMI hot-plug detection. بعدين الـ display controller driver حاول يطلب نفس الـ pin كـ reset GPIO وفشل.

نشوف الـ state:
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-am62x/pinmux-pins | grep GPIO0_15
# pin 15 (GPIO0_15): GPIO gpiochip0:15 (strict)
```

نشوف الـ HDMI DT:
```bash
grep -r "gpio0_15\|gpio0 15" /proc/device-tree/
```

لاقينا إن الـ HDMI driver بيستخدم GPIO0_15 كـ HPD (hot-plug detect) pin.

#### الحل

خيارين:

**الخيار الأول**: نغير الـ reset GPIO لـ pin مختلف (أسهل لو ممكن hardware):

```dts
/* قبل */
&display_controller {
    reset-gpios = <&gpio0 15 GPIO_ACTIVE_LOW>; /* conflict مع HDMI HPD */
};

/* بعد */
&display_controller {
    reset-gpios = <&gpio0 22 GPIO_ACTIVE_LOW>; /* pin حر */
};
```

**الخيار الثاني**: لو مش ممكن تغيير الـ hardware، نتحقق من `pinmux_can_be_used_for_gpio()`:

```c
bool pinmux_can_be_used_for_gpio(struct pinctrl_dev *pctldev, unsigned int pin)
{
    ...
    /* لو ops->strict والـ gpio_owner موجود: return false */
    return !(ops->strict && !!desc->gpio_owner);
}
```

مع `strict = true`، مفيش حل غير إن الـ HDMI يحرر الـ pin أول. نضيف ordering في الـ DT:

```dts
/* نخلي الـ display controller يـ probe بعد الـ HDMI */
&display_controller {
    /* نضيف dependency */
    power-domains = <&hdmi_domain>;
    /* أو نستخدم defer probe mechanism */
};
```

أو الحل الأنظف: نستخدم dedicated HPD pin للـ HDMI مختلف عن الـ reset.

نتحقق من النتيجة:
```bash
# بعد الإصلاح
cat /sys/kernel/debug/pinctrl/pinctrl-am62x/pinmux-pins | grep -E "GPIO0_15|GPIO0_22"
# pin 15 (GPIO0_15): GPIO gpiochip0:15 (strict) [HDMI HPD]
# pin 22 (GPIO0_22): GPIO gpiochip0:22 (strict) [display reset]
```

#### الدرس المستفاد
**`pinmux_can_be_used_for_gpio()`** و **`pin_request()`** مع `strict = true` بيمنعوا sharing خالص. كل GPIO لازم يكون له pin مخصص. في الـ digital signage systems، ارسم جدول كامل لتوزيع الـ GPIO قبل الـ layout وتأكد إن كل peripheral له pins حصرية خاصة بيه — خصوصًا لو الـ SoC بيستخدم strict pinmux model زي AM62x.
## Phase 7: مصادر ومراجع

### مصادر رسمية — kernel.org

| المصدر | الوصف |
|--------|-------|
| [docs.kernel.org — Pin Control subsystem](https://docs.kernel.org/driver-api/pin-control.html) | التوثيق الرسمي الكامل لـ pinctrl/pinmux داخل kernel.org — المرجع الأول والأهم |
| [kernel.org — Documentation/pinctrl.txt (raw)](https://www.kernel.org/doc/Documentation/pinctrl.txt) | النسخة النصية الأصلية من التوثيق التي كتبها Linus Walleij |
| [kernel.org/doc/html/v4.14 — PINCTRL subsystem](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html) | نسخة HTML من توثيق الـ API للـ pinctrl subsystem |

**الملفات ذات الصلة داخل شجرة kernel:**

```
Documentation/driver-api/pin-control.rst   ← التوثيق الرئيسي
include/linux/pinctrl/pinmux.h             ← public API للـ pinmux ops
include/linux/pinctrl/pinctrl.h            ← public API للـ pinctrl core
drivers/pinctrl/pinmux.c                   ← core implementation (المصدر المدروس)
drivers/pinctrl/pinmux.h                   ← internal interface (المصدر المدروس)
drivers/pinctrl/core.c                     ← pinctrl core
drivers/pinctrl/core.h                     ← internal core structures
```

---

### مقالات LWN.net

**الـ** LWN.net هو المصدر الأهم لمتابعة تطور الـ pinctrl subsystem منذ نشأته:

| المقال | الأهمية |
|--------|---------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | المقال التأسيسي الذي يشرح فكرة الـ pinctrl كـ superset لـ pinmux + pinconf — **ابدأ منه** |
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | النسخة السابعة من patch series الأولى — يكشف كيف تطور التصميم قبل الـ merge |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | أول إضافة للتوثيق الرسمي إلى kernel tree — يشمل شرح الـ pinmux functions والـ groups |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/) | إضافة الـ pinconf بجانب الـ pinmux — يوضح التفريق بين الوظيفتين |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | تفاصيل تطوير الـ pin configuration interface |
| [pinctrl: introduce the concept of a GPIO pin function category](https://lwn.net/Articles/1031226/) | مقال حديث (2024) يناقش `PINFUNCTION_FLAG_GPIO` وإشكالية الـ strict controllers على Qualcomm SoCs — مرتبط مباشرة بـ `pinmux_generic_function_is_gpio()` |
| [pinctrl: Add new pinctrl/GPIO driver](https://lwn.net/Articles/803863/) | مثال عملي على كتابة pinctrl driver جديد من الصفر |

---

### نقاشات Mailing List

| الرابط | الموضوع |
|--------|---------|
| [lkml.org — Linus Walleij on rockchip pinctrl](https://lkml.org/lkml/2026/2/24/668) | نقاش حديث (فبراير 2026) على LKML بين Walleij ومطوري الـ rockchip pinctrl |
| [mail-archive — GPIO pin function category v5](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10546.html) | النقاش الكامل حول patch series إدخال `PINFUNCTION_FLAG_GPIO` |
| [lkml — pinctrl: sirf/atlas7 GPIO internals](https://lkml.kernel.org/lkml/1455265873-21534-1-git-send-email-linus.walleij@linaro.org/) | مثال على فصل GPIO internals عن pinctrl — يشرح فلسفة التصميم |
| [lore.kernel.org — pinctrl: devm_kasprintf leak fix](https://lore.kernel.org/all/CACRpkdYX7_rVTp5o8diBSx0JB4iFGjqyzxsqb7etW67SWD=ZRQ@mail.gmail.com/) | مثال على review من Walleij لـ memory management في pinctrl drivers |

**للبحث المباشر في المراسلات:**

```
lore.kernel.org/linux-gpio/          ← القائمة الرئيسية للـ pinctrl/GPIO
lore.kernel.org/linux-kernel/        ← LKML العام
```

---

### مستودع الـ Git — Commits مهمة

**مستودع Linus Walleij الرسمي للـ pinctrl:**
```
https://kernel.googlesource.com/pub/scm/linux/kernel/git/linusw/linux-pinctrl/
```

**Commits تاريخية مهمة يمكن البحث عنها:**

```bash
# الـ commit الأول الذي أدخل الـ pinctrl subsystem (kernel 3.2 — 2011)
git log --oneline --follow -- drivers/pinctrl/pinmux.c | tail -5

# تتبع تطور pinmux_generic_add_function
git log --oneline -- drivers/pinctrl/pinmux.c

# البحث عن إضافة CONFIG_GENERIC_PINMUX_FUNCTIONS
git log --all --oneline --grep="GENERIC_PINMUX_FUNCTIONS"
```

**السياق التاريخي:**
- الـ `pinctrl` subsystem دخل في **Linux 3.2** (ديسمبر 2011)
- الـ `CONFIG_GENERIC_PINMUX_FUNCTIONS` أضيف لاحقاً لتوفير generic helpers
- الـ `strict` mode أضيف للتعامل مع controllers التي لا تسمح بمشاركة pin بين GPIO وfunction

---

### kernelnewbies.org — تغييرات pinctrl عبر الإصدارات

| الإصدار | الرابط | ما الجديد في pinctrl |
|---------|--------|---------------------|
| Linux 6.1 | [kernelnewbies.org/Linux_6.1](https://kernelnewbies.org/Linux_6.1) | تحسينات pinctrl drivers |
| Linux 6.8 | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) | إضافة drivers جديدة للـ pinctrl |
| Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) | تحديثات pinctrl لـ i.MX91 وغيرها |
| Linux 6.13 | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) | pinctrl لـ Qualcomm SM8650 وSamsung gs101 |
| Linux 6.14 | [kernelnewbies.org/Linux_6.14](https://kernelnewbies.org/Linux_6.14) | pinctrl لـ Eswin eic7700 وRaspberryPi RP1 |
| LinuxChanges | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) | صفحة الـ changelog الرئيسية — تحتوي قسم "Pin Controllers" |

---

### elinux.org — موارد عملية

| الرابط | المحتوى |
|--------|---------|
| [elinux.org — Pin Control GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) | عرض تقديمي مهم يشرح العلاقة بين pinctrl وGPIO subsystem |
| [elinux.org — EBC Device Trees](https://elinux.org/EBC_Device_Trees) | أمثلة عملية على إعداد الـ pinctrl-single في Device Tree للـ BeagleBone |
| [elinux.org — Tests:i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) | اختبار pinctrl مع الـ i2c-demux driver — مثال تطبيقي على تبديل الـ mux |

---

### مواقع خارجية مفيدة

| الرابط | المحتوى |
|--------|---------|
| [embedded.com — Linux pin control subsystem](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) | مقال تعليمي مفصل للمطورين الجدد على pinctrl |
| [STMicroelectronics — Pinctrl overview](https://wiki.st.com/stm32mpu/wiki/Pinctrl_overview) | شرح تطبيقي لـ pinctrl على STM32MP processors — مثال واقعي ممتاز |
| [GitHub — torvalds/linux pinmux.c](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/pinmux.c) | الكود الحالي على GitHub مع تاريخ التعديلات |

---

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الرابط:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/) — متاح مجاناً
- **الملاحظة:** الكتاب قديم (2005) ولا يغطي الـ pinctrl لأنه أُضيف في 2011 — لكنه المرجع الأساسي لفهم بنية الـ kernel subsystems والـ driver model

#### Linux Kernel Development, 3rd Edition
- **المؤلف:** Robert Love
- **الفصول ذات الصلة:**
  - Chapter 14: The Block I/O Layer — لفهم الـ request/free patterns المشابهة
  - Chapter 17: Devices and Modules — لفهم الـ `module_get/put` المستخدمة في `pin_request()`
  - Chapter 5: System Calls — لفهم الـ locking patterns (`mutex_lock/unlock`)

#### Embedded Linux Primer, 2nd Edition
- **المؤلف:** Christopher Hallinan
- **الفصول ذات الصلة:**
  - Chapter 16: Kernel Debugging Techniques — يشمل استخدام الـ debugfs مثل `pinmux-functions` و`pinmux-pins`
  - الفصول عن الـ Device Tree وكيفية ربطها بالـ pinctrl settings

#### Mastering Embedded Linux Programming
- **المؤلف:** Chris Simmonds
- يغطي الإصدارات الحديثة ويشمل شرحاً عملياً لـ Device Tree bindings مع pinctrl

---

### مصطلحات البحث

للعثور على المزيد من المعلومات استخدم هذه المصطلحات:

```
# بحث عام
"linux pinctrl pinmux"
"linux pin multiplexing kernel"
"pinctrl_ops set_mux kernel"
"pinmux_enable_setting linux"

# بحث متخصص
"CONFIG_GENERIC_PINMUX_FUNCTIONS"
"pinmux strict mode linux"
"pin_request pin_free kernel"
"PINFUNCTION_FLAG_GPIO"
"function_desc pinctrl kernel"

# بحث في الكود
# على codesearch.debian.net أو elixir.bootlin.com
site:elixir.bootlin.com pinmux_enable_setting
site:elixir.bootlin.com pinmux_ops

# بحث في Mailing List
site:lore.kernel.org "pinmux" "set_mux"
site:lkml.org "pinctrl" "strict"
```

**الـ** [Elixir Cross Referencer](https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl/pinmux.c) هو الأداة الأفضل لتصفح الكود والـ call graphs للـ pinmux subsystem بشكل تفاعلي.
## Phase 8: Writing simple module

### الهدف

**`pinmux_generic_add_function()`** هي واحدة من أكثر الـ exported functions إثارة في الـ pinmux subsystem — كل مرة driver بيسجّل function جديدة (زي "uart0", "spi1", "gpio") بتتعمل call ليها. هنعمل **kprobe** عليها عشان نشوف إيه الـ function اللي بتتسجل، وعلى أنهي controller، وكام group بتدعمها.

---

### الـ Module كاملاً

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on pinmux_generic_add_function()
 *
 * Intercepts every pinmux function registration and logs:
 *   - pin controller name
 *   - function name being registered
 *   - number of groups it supports
 */

/* --- Includes --- */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit         */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe ...       */
#include <linux/ptrace.h>       /* struct pt_regs — CPU register snapshot   */
#include <linux/pinctrl/pinctrl.h> /* struct pinctrl_dev, pinctrl_dev_get_name() */
#include <linux/pinctrl/pinmux.h>  /* pinmux_generic_add_function signature */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe: trace pinmux_generic_add_function() calls");

/* ------------------------------------------------------------------ */
/*  kprobe pre-handler — runs just BEFORE the probed function executes */
/* ------------------------------------------------------------------ */

/*
 * دي الـ callback اللي بتتنفذ قبل ما pinmux_generic_add_function تشتغل.
 * بنقرأ الـ arguments من الـ CPU registers عشان نعرف مين بيسجّل إيه.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * x86-64 calling convention (System V AMD64 ABI):
     *   arg0 → RDI  → struct pinctrl_dev *pctldev
     *   arg1 → RSI  → const char *name          (function name)
     *   arg2 → RDX  → const char * const *groups (array of group names)
     *   arg3 → RCX  → unsigned int ngroups       (number of groups)
     *   arg4 → R8   → void *data                 (driver private data)
     *
     * على ARM64:
     *   arg0 → x0, arg1 → x1, arg2 → x2, arg3 → x3, arg4 → x4
     *
     * بنستخدم regs_get_kernel_argument() اللي portable عبر architectures.
     */
    struct pinctrl_dev *pctldev;
    const char         *fname;
    unsigned int        ngroups;

    /* arg0: pointer to the pin controller device */
    pctldev = (struct pinctrl_dev *)regs_get_kernel_argument(regs, 0);

    /* arg1: name of the mux function being registered (e.g. "uart0") */
    fname   = (const char *)regs_get_kernel_argument(regs, 1);

    /* arg3: how many pin groups this function can be applied to */
    ngroups = (unsigned int)regs_get_kernel_argument(regs, 3);

    /*
     * بنتأكد إن الـ pointers صحيحة قبل ما نعمل dereference
     * عشان منوقعش kernel panic لو جاءتنا قيمة garbage.
     */
    if (!pctldev || !fname)
        return 0;

    pr_info("[pinmux_probe] controller=\"%s\"  function=\"%s\"  ngroups=%u\n",
            pinctrl_dev_get_name(pctldev),
            fname,
            ngroups);

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */

/*
 * الـ struct kprobe ده هو "الخطّاف" — بيحدد على أنهي symbol هنضع الـ breakpoint.
 * بنستخدم الاسم بدل العنوان عشان يكون portable بين kernel versions.
 */
static struct kprobe kp = {
    .symbol_name = "pinmux_generic_add_function",
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/*  module_init                                                         */
/* ------------------------------------------------------------------ */

/*
 * بنسجّل الـ kprobe هنا — لو فشل التسجيل (مثلاً الـ symbol مش موجود
 * أو CONFIG_KPROBES=n) بنرجع error عشان الـ module معيش.
 */
static int __init pinmux_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[pinmux_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[pinmux_probe] planted on %s at %px\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                         */
/* ------------------------------------------------------------------ */

/*
 * لازم نشيل الـ kprobe في الـ exit عشان الـ kernel يرجع يشتغل
 * بدون breakpoint — لو مشيناه من غير ما نعمل unregister هيحصل
 * kernel panic أو memory corruption.
 */
static void __exit pinmux_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[pinmux_probe] removed from %s\n", kp.symbol_name);
}

module_init(pinmux_probe_init);
module_exit(pinmux_probe_exit);
```

---

### Makefile

```makefile
# ضعه في نفس الـ directory مع الـ .c file
obj-m += pinmux_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### Build & Test

```bash
# Build
make

# Load
sudo insmod pinmux_probe.ko

# شوف الـ output في dmesg
sudo dmesg -w | grep pinmux_probe

# مثال لو فيه device بيـprobe دلوقتي
sudo modprobe pinctrl-bcm2835   # Raspberry Pi example

# Unload
sudo rmmod pinmux_probe
```

**مثال مخرجات متوقعة:**

```
[pinmux_probe] planted on pinmux_generic_add_function at ffffffffc08a1240
[pinmux_probe] controller="pinctrl-bcm2835"  function="gpio"    ngroups=54
[pinmux_probe] controller="pinctrl-bcm2835"  function="alt0"    ngroups=54
[pinmux_probe] controller="pinctrl-bcm2835"  function="alt1"    ngroups=54
[pinmux_probe] controller="pinctrl-bcm2835"  function="uart0"   ngroups=2
[pinmux_probe] controller="pinctrl-bcm2835"  function="spi0"    ngroups=1
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | يجيب الـ `struct kprobe` وكل الـ API الخاص بالـ kprobes |
| `linux/ptrace.h` | يجيب `struct pt_regs` اللي فيها snapshot من الـ CPU registers وقت الـ trap |
| `linux/pinctrl/pinctrl.h` | يجيب `struct pinctrl_dev` و `pinctrl_dev_get_name()` عشان نطبع اسم الـ controller |
| `linux/pinctrl/pinmux.h` | يجيب تعريف الـ `struct pinmux_ops` وهو مفيد للفهم لو احتجنا نوسّع الـ probe |

**الـ `linux/module.h`** ضروري لأي kernel module — من غيره مش هيشتغل `MODULE_LICENSE` ولا `module_init`.

#### الـ `handler_pre` callback

**الـ `struct pt_regs *regs`** هو الحجة الأهم — ده snapshot كامل من الـ CPU registers في اللحظة اللي الـ kprobe شافها وقبل ما الـ function الأصلية تشتغل. بنستخدم `regs_get_kernel_argument(regs, N)` بدل ما ناخد `regs->rdi` مباشرةً عشان الكود يشتغل على x86 وARM64 وRISC-V من غير تعديل.

#### ليه `return 0` من الـ handler؟

لو الـ `pre_handler` رجع قيمة غير صفر، الـ kprobe بيمنع تنفيذ الـ function الأصلية وده سلوك خطير في الـ production. **الـ `return 0`** معناها "اتفرجت بس، دلوقتي اشتغل زي ما أنت عارف".

#### الـ `register_kprobe` في `module_init`

**الـ `register_kprobe()`** بيدور على الـ symbol في الـ kernel symbol table، بيحسب العنوان، وبيضع breakpoint instruction (مثلاً `int3` على x86). لو الـ symbol مش exported أو الـ kernel اتبنى بـ `CONFIG_KPROBES=n`، بترجع error سالب ومش بنكمّل.

#### الـ `unregister_kprobe` في `module_exit`

لو مشيت الـ module من غير ما تشيل الـ kprobe، الـ kernel هيظل بيـexecute الـ breakpoint لكن الـ handler راح — ده بيأدي لـ **kernel panic** مؤكد. الـ `unregister_kprobe()` بيرجع الـ original instruction في مكان الـ breakpoint ويحرر الموارد.

---

### ملاحظات مهمة

- **`CONFIG_KPROBES=y`** لازم يكون enabled في الـ kernel config (موجود في كل distro حديثة).
- **`pinmux_generic_add_function`** بيتنادى بس وقت الـ driver probe، مش في الـ hot path، فمفيش overhead مهم.
- لو الـ kernel اتبنى بـ `CONFIG_GENERIC_PINMUX_FUNCTIONS=n` مش هتلاقي الـ symbol وهيفشل الـ register.
- الـ `pinctrl_dev_get_name()` آمنة تتنادى من الـ probe context لأنها بس بتقرأ string من الـ struct من غير locking.
