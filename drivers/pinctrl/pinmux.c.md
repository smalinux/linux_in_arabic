## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Pin Muxing؟

تخيل عندك لوحة إلكترونية — مثلاً Raspberry Pi أو أي SoC (System on Chip) — فيها مئات الـ **pins** (أرجل المعالج). كل pin ده عبارة عن سلك صغير بيخرج من الشريحة. المشكلة إن الشريحة فيها وظائف كتير جداً: UART، SPI، I2C، PWM، GPIO، I2S وغيرها — لكن الأرجل (pins) محدودة العدد.

فإيه الحل؟ كل pin ممكن يشتغل بأكتر من وظيفة واحدة، بس في وقت واحد بس يقدر يشتغل بوظيفة واحدة. ده اللي بيتسمى **Pin Multiplexing** أو اختصاراً **Pinmux**.

### القصة: المنفذ المشترك

تخيل إن عندك باب واحد في الشقة — ممكن يدخل منه ناس الأكل (UART)، أو ناس الإنترنت (SPI)، أو صاحبك (GPIO) — بس في وقت واحد بس نوع واحد. لازم حد "يحجز" الباب ده لغرض معين، وما يجيش تاني حد يستخدمه في نفس الوقت. لو اتنين اتشاجروا على نفس الباب، الـ kernel بيقول "لأ" لواحد منهم.

ده بالظبط اللي بيعمله ملف `pinmux.c`: بيدير مين بيستخدم أنهي pin، وبيمنع التعارضات.

---

### الـ Subsystem اللي ينتمي ليه

الملفين دول جزء من **PIN CONTROL SUBSYSTEM** — المسجل في `MAINTAINERS` تحت:

```
PIN CONTROL SUBSYSTEM
M:  Linus Walleij <linusw@kernel.org>
L:  linux-gpio@vger.kernel.org
F:  drivers/pinctrl/
F:  include/linux/pinctrl/
F:  Documentation/driver-api/pin-control.rst
```

---

### الهدف من `pinmux.c` و `pinmux.h`

الـ **pinctrl subsystem** فيه ثلاث طبقات:

| الطبقة | الملف | المسؤولية |
|--------|-------|-----------|
| **Core** | `core.c` | تسجيل الـ pins وإدارة الـ states |
| **Pinmux** | `pinmux.c` | تحديد وظيفة كل pin (UART? GPIO? SPI?) |
| **Pinconf** | `pinconf.c` | إعداد خصائص الـ pin (pull-up، drive strength...) |

الـ `pinmux.c` تحديداً مسؤول عن:

1. **حجز الـ pins**: قبل ما أي وظيفة تشتغل، لازم "تطلب" الـ pins اللي هي محتاجاها
2. **ضبط الـ mux**: إخبار الـ hardware بالفعل "حول الـ pin ده لوظيفة UART"
3. **منع التعارض**: لو pin اتحجز لـ SPI، ما يتحجزش تاني لـ I2C
4. **دعم GPIO**: حالة خاصة لما الـ pin يشتغل كـ GPIO عادي

---

### الـ Functions والـ Groups: القاموس الأساسي

قبل ما تفهم الكود، لازم تعرف:

- **Function**: وظيفة يقدر يعملها الـ pin controller — مثلاً `"uart0"`, `"spi1"`, `"i2c2"`. كل function ليها اسم.
- **Group**: مجموعة pins بتعمل الوظيفة دي مع بعض — مثلاً وظيفة `"uart0"` محتاجة pin رقم 10 (TX) وpin رقم 11 (RX) مع بعض.
- **Selector**: رقم (index) بيمثل الـ function أو الـ group — لأن الـ kernel بيشتغل بالأرقام داخلياً أسرع من الـ strings.

```
Function "uart0"  →  Group "uart0-grp"  →  Pins [10, 11]
Function "spi1"   →  Group "spi1-grp"  →  Pins [20, 21, 22, 23]
Function "gpio"   →  Group "gpio10"    →  Pins [10]
```

---

### القصة الكاملة: رحلة الـ UART من البداية للنهاية

**السيناريو:** عندك UART driver عايز يشتغل على Pin 10 و Pin 11.

```
Device Tree
    │
    ▼
pinctrl_map  (machine.h)
  dev="uart0", state="default", function="uart0", group="uart0-grp"
    │
    ▼
pinmux_map_to_setting()          ← يحول الاسم "uart0" لـ selector رقم
    │   pinmux_func_name_to_selector()
    │
    ▼
pinmux_enable_setting()          ← يحجز الـ pins ويفعّل الـ mux
    │   pin_request(pin=10, owner="uart0")
    │   pin_request(pin=11, owner="uart0")
    │   ops->set_mux(func_selector, group_selector)
    │                    │
    │                    ▼
    │            Hardware Register
    │            GPIO_MUX[10] = UART_TX
    │            GPIO_MUX[11] = UART_RX
    ▼
UART driver يشتغل بشكل طبيعي
```

لما الـ UART بيتوقف:
```
pinmux_disable_setting()  →  pin_free(10)  →  pin_free(11)
```

---

### الـ strict mode: الحارس الصارم

الـ **strict mode** هو إعداد في `struct pinmux_ops` — لما يتفعّل، بيمنع أي pin من الاشتراك بين GPIO وأي function تاني في نفس الوقت. بدونه، بعض الـ hardware يسمح بالاشتراك (مثلاً لو function هي نفسها GPIO).

```c
/* strict=true: يرفض طلب GPIO لو الـ pin محجوز لـ mux function */
if (ops->strict && desc->mux_usecount && !func_is_gpio)
    return false;
```

---

### الـ Generic Functions: مكتبة الـ Lazy Drivers

الجزء التاني في الملف (`CONFIG_GENERIC_PINMUX_FUNCTIONS`) هو "مكتبة جاهزة" لصانعي الـ drivers — بدل ما كل driver يكتب نفس الكود لإدارة قائمة الـ functions، الـ kernel بيوفر:

```c
pinmux_generic_add_function()     // أضف function جديدة
pinmux_generic_get_function_name() // اسأل عن اسمها
pinmux_generic_remove_function()  // احذفها
```

البيانات اتخزن في `radix_tree` (شجرة بحث سريعة) داخل الـ `pinctrl_dev`.

---

### مين بيستدعي مين؟ خريطة التفاعلات

```
┌──────────────────────────────────────────────┐
│              Consumer (UART, SPI, ...)        │
│   pinctrl_get() / pinctrl_select_state()      │
└────────────────────┬─────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────┐
│              core.c (pinctrl core)            │
│   يفسر الـ states ويستدعي pinmux + pinconf   │
└──────┬─────────────────────────┬─────────────┘
       │                         │
       ▼                         ▼
┌─────────────┐         ┌──────────────────┐
│  pinmux.c   │         │    pinconf.c      │
│ (الوظيفة)   │         │ (الخصائص)         │
└──────┬──────┘         └──────────────────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│         Hardware Driver (مثلاً imx.c)        │
│   struct pinmux_ops { .set_mux = imx_mux }  │
└─────────────────────────────────────────────┘
```

---

### الـ GPIO كحالة خاصة

الـ GPIO ليست مجرد function عادية — الـ kernel بيتعامل معها بشكل مختلف:

```c
// لما GPIO driver يطلب pin
pinmux_request_gpio()
    └── pin_request(pctldev, pin, owner, gpio_range)
            // لاحظ: gpio_range != NULL = GPIO mode
            └── ops->gpio_request_enable()  // يفعّل GPIO mode في الـ hardware

// الـ mux_owner vs gpio_owner: تتبعان بشكل منفصل في pin_desc
desc->mux_owner   // "uart0" لو مضبوط لـ mux
desc->gpio_owner  // "gpiochip0:10" لو مضبوط لـ GPIO
```

---

### الـ debugfs: نافذة التشخيص

الـ `pinmux.c` يضيف ثلاث ملفات في `/sys/kernel/debug/pinctrl/<controller>/`:

| الملف | المحتوى |
|-------|---------|
| `pinmux-functions` | قائمة كل الـ functions والـ groups |
| `pinmux-pins` | حالة كل pin: مين حاجزه؟ |
| `pinmux-select` | (write-only) تفعيل mux يدوياً للاختبار |

مثال على output:
```
pin 10 (PA10): device uart0 function uart0 group uart0-grp
pin 20 (PA20): GPIO gpiochip0:20
pin 30 (PA30): UNCLAIMED
```

---

### الملفات الأساسية في الـ Subsystem

#### Core (اللب)
| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/core.c` | المحرك الرئيسي: تسجيل الـ controllers والـ states |
| `drivers/pinctrl/core.h` | التعريفات الداخلية (struct pin_desc, ...) |
| `drivers/pinctrl/pinmux.c` | **ملفنا**: إدارة الـ mux والـ function switching |
| `drivers/pinctrl/pinmux.h` | واجهة داخلية بين core.c وpinmux.c |
| `drivers/pinctrl/pinconf.c` | إدارة خصائص الـ pins (pull, drive strength) |
| `drivers/pinctrl/devicetree.c` | تحليل الـ Device Tree وبناء الـ pinctrl_map |

#### Headers (الواجهات العامة)
| الملف | الدور |
|-------|-------|
| `include/linux/pinctrl/pinmux.h` | تعريف `struct pinmux_ops` (الـ driver interface) |
| `include/linux/pinctrl/pinctrl.h` | تعريف `struct pinctrl_desc`, `pingroup`, `pinfunction` |
| `include/linux/pinctrl/machine.h` | تعريف `struct pinctrl_map` (ربط devices بالـ pins) |
| `include/linux/pinctrl/consumer.h` | واجهة المستخدمين (UART, SPI, ...) |
| `include/linux/pinctrl/pinconf.h` | واجهة الـ pin configuration |

#### Hardware Drivers (أمثلة)
| الملف | الـ Hardware |
|-------|-------------|
| `drivers/pinctrl/freescale/pinctrl-imx.c` | NXP i.MX SoCs |
| `drivers/pinctrl/intel/pinctrl-intel.c` | Intel SoCs |
| `drivers/pinctrl/bcm/pinctrl-bcm2835.c` | Raspberry Pi |
| `drivers/pinctrl/meson/` | Amlogic SoCs |

---

### ملخص سريع

**الـ `pinmux.c`** هو "شرطي المرور" للـ pins — بيحدد مين يستخدم أنهي pin، ويمنع التعارضات، ويوصل الأوامر للـ hardware الفعلي. الـ `pinmux.h` هو "التذكرة الداخلية" اللي بتربط الـ core بالـ pinmux بدون ما أي حد برا يشوف التفاصيل.
## Phase 2: شرح الـ Pinmux Framework

### المشكلة اللي بيحلها الـ Pinmux

أي SoC حديث — خد مثلاً الـ STM32H7 أو Raspberry Pi CM4 — فيه مئات الـ pins على الـ package، بس الـ IP cores الداخلية (UART، SPI، I2C، PWM، ADC) أكتر بكتير من عدد الـ pins. الحل الهاردوير الكلاسيكي هو الـ **pin multiplexing**: كل pin واحد ممكن يشتغل بأكتر من وظيفة، وبيتحدد ده بـ register في الـ SoC.

المشكلة من غير framework:
- كل driver ييجي يكتب مباشرة في الـ IOMUX registers → conflict بين drivers.
- UART driver بياخد pin بدون ما يسأل GPIO driver → short circuit منطقي.
- الـ board support code بتتكرر في كل driver.
- مفيش أي طريقة تعرف مين شايل الـ pin ده في runtime.

**الـ pinmux subsystem** بيحل ده بإنه يكون الـ single owner اللي بيخلّي أي pin لـ function واحدة بس في اللحظة الواحدة، وبيعمل locking صريح.

---

### الحل — المدخل اللي الكيرنل اتبعه

الكيرنل قسّم الموضوع لثلاث مفاهيم:

| المفهوم | المعنى |
|---------|--------|
| **Pin** | رقم فريد عالمي للـ pad الفيزيائي |
| **Group** | مجموعة pins بتشتغل مع بعض كوحدة واحدة (مثلاً 4 pins لـ SPI0) |
| **Function** | الوظيفة اللي ممكن تتفعّل على group (مثلاً "spi0"، "uart1"، "gpio") |

الـ framework بيقدم:
1. **Ownership tracking**: كل pin عنده `mux_owner` و `gpio_owner`.
2. **Ops table delegation**: الـ driver بيعمل implement لـ `struct pinmux_ops`، والـ core بيستدعيها.
3. **Map table**: الـ board/DT بيعرّف "الـ device ده محتاج الـ function دي على الـ group ده".
4. **State machine**: الـ consumer driver بياخد pinctrl state (default، sleep، ...) والـ core بيطبّقه.

---

### التشبيه الواقعي — مكتبة فيها أدوات مشتركة

تخيل مكتبة هندسية (= الـ SoC) فيها أدوات (= pins). كل أداة ممكن تتستخدم لأغراض مختلفة:
- مسطرة → قياس أو رسم.
- قلم → كتابة أو تحديد.

**الـ librarian** (= pinctrl core) هو اللي بيقرر مين بياخد الأداة.
- لو حد طلب الأداة للرسم (= UART) والأداة متاحة → بيديهاله ويكتب اسمه في السجل (`mux_owner`).
- لو حد تاني طلبها لنفس الوقت → الـ librarian بيرفض ويقول "متاخدة من X".
- لو الأداة اتاخدت كـ "pen" وحد حاول ياخدها كـ "pencil" في نفس الوقت في الـ strict mode → رفض صريح.

الماب (= `pinctrl_map`) هو قائمة الحجوزات المسبقة. كل device بيقول "أنا هحتاج الأداة دي في حالة التشغيل العادية".

كل جزء في التشبيه له مقابل حقيقي:

| التشبيه | الكيرنل |
|---------|---------|
| الأداة الفيزيائية | `pin` رقم في global space |
| نوع الاستخدام | `function` (uart، spi، gpio...) |
| مجموعة الأدوات | `group` (مثلاً uart0_grp = [PA9, PA10]) |
| سجل الحجوزات | `pin_desc->mux_owner` / `gpio_owner` |
| الـ librarian | pinmux core (`pinmux.c`) |
| قائمة الحجوزات | `pinctrl_map[]` في board code |
| مندوب الـ librarian في الـ shelf | `pinmux_ops` في driver |

---

### الصورة الكبيرة — Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │                      Consumer Drivers                        │
  │   UART driver    SPI driver    I2C driver    GPIO subsystem  │
  └──────┬───────────────┬────────────┬──────────────┬──────────┘
         │               │            │              │
         │  pinctrl_get() / pinctrl_select_state()   │
         ▼               ▼            ▼              ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    pinctrl core  (core.c)                    │
  │   - يحتفظ بـ pinctrl_map table                              │
  │   - يحوّل state names → settings                            │
  │   - يستدعي pinmux layer لتفعيل الـ mux                      │
  └──────────────────────────┬──────────────────────────────────┘
                             │
                    ┌────────┴────────┐
                    │  pinmux core    │  ← (pinmux.c — هذا الملف)
                    │                 │
                    │ pin_request()   │  ← يتحقق من الملكية
                    │ pin_free()      │  ← يحرر الـ pin
                    │ pinmux_enable_setting()  ← يفعّل الـ mux
                    └────────┬────────┘
                             │  استدعاء pinmux_ops callbacks
                             ▼
  ┌─────────────────────────────────────────────────────────────┐
  │              Pin Controller Driver (مثلاً pinctrl-stm32.c)   │
  │                                                             │
  │  .set_mux()              → يكتب في IOMUX registers          │
  │  .request() / .free()    → حجز/تحرير هاردوير               │
  │  .gpio_request_enable()  → يحوّل pin لـ GPIO mode           │
  │  .gpio_set_direction()   → input/output                     │
  └──────────────────────────┬──────────────────────────────────┘
                             │
                    ┌────────┴────────┐
                    │  SoC Hardware   │
                    │  IOMUX / Mux    │
                    │  Registers      │
                    └─────────────────┘

  Data flow لـ Map Table:
  ┌─────────────────────────────────────────────────────────────┐
  │  Board/DTS → pinctrl_map[] → pinctrl_setting → pin_desc     │
  │                                                             │
  │  مثال: "uart0" device في state "default":                   │
  │    function = "uart0"  →  selector = 3                      │
  │    group    = "uart0_grp" → pins = [42, 43]                 │
  │    يستدعي: set_mux(pctldev, func=3, group=5)                │
  └─────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ pinmux subsystem بيقدم abstraction واحدة أساسية:

> **"Pin ownership with function-group binding"**
>
> أي pin في أي لحظة مملوك لـ entity واحدة بس (أو مش مملوك خالص)،
> والملكية دي مربوطة بـ function محددة على group محدد.

ده بيتجسّد في `struct pin_desc` (معرّفة في `core.h`):

```c
struct pin_desc {
    struct pinctrl_dev *pctldev;   /* الـ controller المسؤول */
    const char         *name;      /* اسم الـ pin */

    /* Mux ownership */
    unsigned int        mux_usecount; /* كام function طالبة الـ pin ده */
    const char         *mux_owner;   /* اسم الـ device صاحب الـ mux */
    const struct pinctrl_setting_mux *mux_setting; /* الـ func+group المفعّلين */

    /* GPIO ownership (منفصل!) */
    const char         *gpio_owner;  /* اسم الـ GPIO consumer */

    struct mutex        mux_lock;    /* حماية concurrent access */
};
```

النقطة الدقيقة: فيه **مالكَين محتملَين** لكل pin:
- `mux_owner`: لما pin اتاخد لـ function معينة (UART، SPI، ...)
- `gpio_owner`: لما GPIO subsystem طلب الـ pin

الـ `strict` flag في `pinmux_ops` بيتحكم في إزاي الـ core يتعامل مع الحالة دي:
- `strict = false` (default): ممكن يكون فيه mux_owner وgpio_owner في نفس الوقت (بعض الـ controllers بتدعم ده).
- `strict = true`: ممنوع تماماً — إما ده أو ده.

```c
/* من pinmux.c — قلب الـ ownership check */
if ((!gpio_range || ops->strict) && !gpio_ok &&
    desc->mux_usecount && strcmp(desc->mux_owner, owner)) {
    dev_err(..., "pin %s already requested by %s; cannot claim for %s\n", ...);
    goto out;
}
```

---

### الـ Structs وعلاقتها ببعض

```
  pinctrl_desc
  ┌──────────────────────────────┐
  │ name                         │
  │ pins[]   → pinctrl_pin_desc  │──────┐
  │ npins                        │      │
  │ pctlops  → pinctrl_ops       │      │  pin فيزيائي: رقم + اسم
  │ pmxops   → pinmux_ops ◄──────┼──────┼──── هنا مكمن الـ pinmux
  │ confops  → pinconf_ops       │      │
  └──────────────────────────────┘      ▼
                                   pinctrl_pin_desc
                                   ┌──────────────┐
                                   │ number       │
                                   │ name         │
                                   │ drv_data     │
                                   └──────────────┘

  pinmux_ops (vtable — الـ driver بيملأه)
  ┌────────────────────────────────────────────┐
  │ request()              ← اطلب pin للـ mux  │
  │ free()                 ← حرره              │
  │ get_functions_count()  ← كام function؟     │
  │ get_function_name()    ← اسم function[i]  │
  │ get_function_groups()  ← groups بتاعتها   │
  │ function_is_gpio()     ← GPIO function?   │
  │ set_mux()              ← فعّل الـ mux ◄── MANDATORY │
  │ gpio_request_enable()  ← GPIO fast path   │
  │ gpio_disable_free()    ← GPIO fast path   │
  │ gpio_set_direction()   ← input/output     │
  │ strict                 ← bool flag        │
  └────────────────────────────────────────────┘

  pinctrl_map (board/DTS level)
  ┌─────────────────────────────┐
  │ dev_name    "spi0"          │  ← مين اللي بيطلب
  │ name        "default"       │  ← state اسمه
  │ type        MUX_GROUP       │
  │ ctrl_dev_name "pinctrl0"    │  ← مين الـ controller
  │ data.mux:                   │
  │   .group    "spi0_grp"      │  ← الـ group
  │   .function "spi0"          │  ← الـ function
  └─────────────────────────────┘
          │
          ▼  pinmux_map_to_setting()
  pinctrl_setting
  ┌─────────────────────────────┐
  │ pctldev                     │  ← pointer للـ controller
  │ data.mux:                   │
  │   .func   = 3  (selector)   │  ← رقم الـ function
  │   .group  = 5  (selector)   │  ← رقم الـ group
  └─────────────────────────────┘
          │
          ▼  pinmux_enable_setting()
  يستدعي:
    1. get_group_pins(group=5) → pins=[42,43]
    2. pin_request(42) + pin_request(43)
    3. desc[42].mux_setting = &setting.data.mux
    4. set_mux(pctldev, func=3, group=5)
```

---

### Generic Pinmux Functions — الـ Helper Layer

الـ Kconfig option `CONFIG_GENERIC_PINMUX_FUNCTIONS` بيوفر implementation جاهزة للـ functions الأساسية. الـ driver بدل ما يكتب `get_functions_count` و`get_function_name` من الصفر، بيستخدم:

```c
/* الـ driver بيضيف function بسطر واحد */
pinmux_generic_add_function(pctldev, "uart0", uart0_groups, ARRAY_SIZE(uart0_groups), NULL);

/* والـ core بيخزنها في radix tree */
// pin_function_tree: index → function_desc
```

الـ `struct function_desc` هي الـ wrapper:

```c
struct function_desc {
    const struct pinfunction *func;  /* الاسم والـ groups */
    void *data;                      /* driver-specific data */
};

struct pinfunction {
    const char        *name;
    const char *const *groups;   /* ["uart0_grp_a", "uart0_grp_b"] */
    size_t             ngroups;
    unsigned long      flags;    /* PINFUNCTION_FLAG_GPIO لو GPIO */
};
```

الـ radix tree بيخلّي الـ lookup بـ O(log n) بدل linear scan.

---

### Flow كامل — من Driver Probe لـ set_mux

```
  1. Board/DTS يعرّف:
     PIN_MAP_MUX_GROUP_DEFAULT("uart0", "pinctrl0", "uart0_grp", "uart0")
                │
                ▼
  2. pinctrl_register_mappings() يضيف الـ map للـ global list

  3. uart0 driver يعمل:
     devm_pinctrl_get(dev)           → pinctrl handle
     pinctrl_lookup_state(p, "default") → pinctrl_state
     pinctrl_select_state(p, state)  → triggers:

  4. pinctrl core → pinmux_map_to_setting()
     - pinmux_func_name_to_selector("uart0") → func_selector=3
     - get_function_groups(3) → ["uart0_grp"]
     - pinctrl_get_group_selector("uart0_grp") → group_selector=5

  5. pinmux_enable_setting()
     - get_group_pins(5) → pins=[42, 43]
     - pin_request(42, "uart0", NULL):
         desc[42].mux_usecount++
         desc[42].mux_owner = "uart0"
         ops->request(pctldev, 42)       ← driver hook
     - pin_request(43, ...) → نفس العملية
     - desc[42].mux_setting = &setting.data.mux
     - ops->set_mux(pctldev, 3, 5)      ← يكتب في IOMUX registers

  6. لو فيه error في أي خطوة:
     - rollback: pin_free() لكل pin اتاخد
     - desc->mux_setting = NULL
```

---

### ما بيملكه الـ Pinmux Core مقابل ما بيفوّضه للـ Driver

| المسؤولية | المالك |
|-----------|--------|
| تتبع `mux_owner` و `gpio_owner` | **pinmux core** |
| الـ locking على الـ `pin_desc` | **pinmux core** (mutex per pin) |
| التحقق من conflicts | **pinmux core** |
| ترجمة function name → selector | **pinmux core** |
| ترجمة group name → selector | **pinctrl core** |
| الكتابة في الـ IOMUX registers | **driver** عبر `set_mux()` |
| تحديد الـ groups المرتبطة بكل function | **driver** عبر `get_function_groups()` |
| التحقق من أن الـ pin متاح هاردوير | **driver** عبر `request()` |
| GPIO fast path (direction، enable) | **driver** عبر `gpio_*` callbacks |
| تعريف الـ pins والـ groups نفسها | **driver** + **board/DTS** |

---

### ملاحظة على الـ GPIO Path

الـ GPIO subsystem (راجع `gpiolib.c` — هو المسؤول عن إدارة GPIO lines وتوجيه الطلبات للـ pin controller) بيستدعي:

```c
pinmux_request_gpio(pctldev, range, pin, gpio)
```

ده بيعمل `pin_request()` بـ `gpio_range != NULL`، فبيسلك مسار مختلف:
- بيضع `gpio_owner` بدل `mux_owner`
- بيستدعي `gpio_request_enable()` بدل `request()` لو موجودة
- الـ `strict` flag بيقرر لو ممكن GPIO ومux في نفس الوقت أو لأ

ده بيخلّي GPIO subsystem وpinmux subsystem مترابطين بـ pinctrl layer كوسيط، مش مباشرةً مع الـ hardware registers.

---

### debugfs — رؤية الحالة في Runtime

الـ pinmux بيوفر ثلاث ملفات في `/sys/kernel/debug/pinctrl/<controller>/`:

```bash
# كل الـ functions المتاحة وlـ groups بتاعتها
cat /sys/kernel/debug/pinctrl/pinctrl0/pinmux-functions
# function 0: gpio, groups = [ gpio0 gpio1 ... ]
# function 1: uart0, groups = [ uart0_grp ]

# حالة كل pin
cat /sys/kernel/debug/pinctrl/pinctrl0/pinmux-pins
# pin 42 (PA10): uart0 (GPIO UNCLAIMED) function uart0 group uart0_grp
# pin 43 (PA11): uart0 (GPIO UNCLAIMED) function uart0 group uart0_grp
# pin 44 (PA12): (MUX UNCLAIMED) (GPIO UNCLAIMED)

# تفعيل mux يدوياً للـ debugging
echo "uart0_grp uart0" > /sys/kernel/debug/pinctrl/pinctrl0/pinmux-select
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### الـ `PINFUNCTION_FLAG_GPIO`

| Flag | القيمة | المعنى |
|------|--------|--------|
| `PINFUNCTION_FLAG_GPIO` | `BIT(0)` | الـ function دي بتاعة GPIO — بيستخدمها `function_is_gpio` callback عشان يعرف يفرق بين GPIO وأي function تانية |

#### الـ `pinctrl_map_type` (من `linux/pinctrl/machine.h`)

| النوع | المعنى |
|-------|--------|
| `PIN_MAP_TYPE_INVALID` | غير صالح |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمية، مفيش hardware |
| `PIN_MAP_TYPE_MUX_GROUP` | ضبط mux لـ group من الـ pins |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | ضبط config لـ group |
| `PIN_MAP_TYPE_CONFIGS_PIN` | ضبط config لـ pin واحد |

#### الـ `CONFIG_*` Options المؤثرة على الـ pinmux

| Option | الأثر |
|--------|-------|
| `CONFIG_PINMUX` | يفعّل كل كود الـ pinmux، بدونه كل الـ functions بتبقى stubs |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | يفعّل الـ generic API: `pinmux_generic_*` + الـ `function_desc` struct + الـ `pin_function_tree` في `pinctrl_dev` |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | يفعّل الـ generic group management: `pin_group_tree` في `pinctrl_dev` |
| `CONFIG_DEBUG_FS` | يضيف `/sys/kernel/debug/pinctrl/<dev>/pinmux-functions` و `pinmux-pins` و `pinmux-select` |

#### الـ `strict` flag في `pinmux_ops`

| الوضع | السلوك |
|-------|--------|
| `strict = false` | ممكن pin واحد يكون عنده `mux_owner` وفي نفس الوقت `gpio_owner` |
| `strict = true` | ممنوع تماماً — لو الـ pin عنده أي owner، أي طلب جديد هيتـreject |

---

### كل الـ Structs المهمة

---

#### `struct pinmux_ops`
**الغرض:** الـ vtable اللي الـ driver بيملّيها — كل الـ hardware-specific operations للـ muxing.

| Field | النوع | الوصف |
|-------|-------|-------|
| `request` | `int (*)(pctldev, offset)` | اطلب pin معين قبل أي mux — الـ driver يقرر هيقبل ولا لأ |
| `free` | `int (*)(pctldev, offset)` | حرر الـ pin بعد الاستخدام |
| `get_functions_count` | `int (*)(pctldev)` | كام function موجود في الـ controller |
| `get_function_name` | `const char *(*)(pctldev, selector)` | اسم الـ function رقم `selector` |
| `get_function_groups` | `int (*)(pctldev, selector, **groups, *ngroups)` | الـ groups اللي بتدعم الـ function ده |
| `function_is_gpio` | `bool (*)(pctldev, selector)` | هل الـ function دي GPIO؟ (اختياري) |
| `set_mux` | `int (*)(pctldev, func_selector, group_selector)` | **المهمة الأساسية** — اكتب على الـ hardware register |
| `gpio_request_enable` | `int (*)(pctldev, range, offset)` | اطلب وفعّل GPIO على pin معين |
| `gpio_disable_free` | `void (*)(pctldev, range, offset)` | عكس `gpio_request_enable` |
| `gpio_set_direction` | `int (*)(pctldev, range, offset, input)` | حدد input/output للـ GPIO |
| `strict` | `bool` | منع استخدام الـ pin لـ GPIO ومقصد آخر في نفس الوقت |

الـ callbacks **الإلزامية**: `get_functions_count` + `get_function_name` + `get_function_groups` + `set_mux` — كلهم بيتتحقق منهم في `pinmux_check_ops()`.

---

#### `struct pinfunction`
**الغرض:** وصف function واحدة — الاسم والـ groups اللي بتدعمها.

| Field | النوع | الوصف |
|-------|-------|-------|
| `name` | `const char *` | اسم الـ function مثلاً `"uart0"`, `"spi1"` |
| `groups` | `const char * const *` | array بأسماء الـ groups اللي الـ function دي تشتغل عليهم |
| `ngroups` | `size_t` | عدد الـ groups |
| `flags` | `unsigned long` | حالياً فيه بس `PINFUNCTION_FLAG_GPIO` |

**Macros للتعريف:**
```c
/* function عادية */
PINCTRL_PINFUNCTION("uart0", uart0_groups, ARRAY_SIZE(uart0_groups))

/* function من نوع GPIO */
PINCTRL_GPIO_PINFUNCTION("gpio", gpio_groups, ARRAY_SIZE(gpio_groups))
```

---

#### `struct function_desc`
**الغرض:** الـ wrapper اللي بيحط `pinfunction` جوا الـ generic infrastructure — بيتخزن في `pin_function_tree`.

| Field | النوع | الوصف |
|-------|-------|-------|
| `func` | `const struct pinfunction *` | بوينتر للـ function descriptor الحقيقي |
| `data` | `void *` | بيانات خاصة بالـ driver — الـ core مبيلمسهاش |

---

#### `struct pin_desc`
**الغرض:** descriptor لكل pin فيزيائي — بيتخزن في `pin_desc_tree` بالـ radix tree.

| Field | النوع | الوصف |
|-------|-------|-------|
| `pctldev` | `struct pinctrl_dev *` | الـ controller اللي بيملك الـ pin ده |
| `name` | `const char *` | اسم الـ pin مثلاً `"PA0"` |
| `dynamic_name` | `bool` | الاسم اتعمله `kstrdup` ولازم يتحرر |
| `drv_data` | `void *` | بيانات خاصة بالـ driver |
| `mux_usecount` | `unsigned int` | كام client بيستخدم الـ pin ده كـ mux — لما يبقى صفر، الـ pin free |
| `mux_owner` | `const char *` | اسم الـ device اللي عمل `pinctrl_get()` |
| `mux_setting` | `const struct pinctrl_setting_mux *` | آخر mux setting اتطبق على الـ pin ده |
| `gpio_owner` | `const char *` | لو `pinctrl_gpio_request()` اتعمل، اسم الـ GPIO owner |
| `mux_lock` | `struct mutex` | بيحمي الـ fields دي ضد الـ race conditions |

---

#### `struct pinctrl_setting_mux`
**الغرض:** بيانات الـ mux setting — اللي function مع أي group.

| Field | النوع | الوصف |
|-------|-------|-------|
| `group` | `unsigned int` | selector رقم الـ group |
| `func` | `unsigned int` | selector رقم الـ function |

---

#### `struct pinctrl_setting`
**الغرض:** setting واحد (mux أو config) مرتبط بـ state معين.

| Field | النوع | الوصف |
|-------|-------|-------|
| `node` | `struct list_head` | عقدة في قائمة الـ `pinctrl_state->settings` |
| `type` | `enum pinctrl_map_type` | نوع الـ setting |
| `pctldev` | `struct pinctrl_dev *` | الـ controller المسؤول |
| `dev_name` | `const char *` | اسم الـ device اللي طلب الـ setting ده |
| `data.mux` | `struct pinctrl_setting_mux` | لو النوع `MUX_GROUP` |
| `data.configs` | `struct pinctrl_setting_configs` | لو النوع `CONFIGS_*` |

---

#### `struct pinctrl_dev`
**الغرض:** الـ runtime instance للـ pin controller — بيتسجل لما الـ driver يعمل `pinctrl_register_and_init()`.

| Field | النوع | الوصف |
|-------|-------|-------|
| `node` | `struct list_head` | عقدة في القائمة العامة للـ controllers |
| `desc` | `const struct pinctrl_desc *` | الـ descriptor اللي الـ driver سجّله |
| `pin_desc_tree` | `struct radix_tree_root` | كل الـ `pin_desc` مفهرسة بالـ pin number |
| `pin_group_tree` | `struct radix_tree_root` | الـ groups (لو `GENERIC_PINCTRL_GROUPS`) |
| `num_groups` | `unsigned int` | عدد الـ groups |
| `pin_function_tree` | `struct radix_tree_root` | الـ functions كـ `function_desc` (لو `GENERIC_PINMUX_FUNCTIONS`) |
| `num_functions` | `unsigned int` | عدد الـ functions |
| `gpio_ranges` | `struct list_head` | قائمة الـ `pinctrl_gpio_range` المضافة |
| `dev` | `struct device *` | الـ device الحقيقي |
| `owner` | `struct module *` | للـ refcounting |
| `driver_data` | `void *` | بيانات الـ driver |
| `p` | `struct pinctrl *` | للـ "hogging" — الـ pins اللي الـ controller نفسه بيحجزها |
| `hog_default` | `struct pinctrl_state *` | الـ state الافتراضية للـ hogged pins |
| `hog_sleep` | `struct pinctrl_state *` | الـ sleep state للـ hogged pins |
| `mutex` | `struct mutex` | بيحمي كل operations على الـ controller ده |
| `device_root` | `struct dentry *` | root الـ debugfs لو `CONFIG_DEBUG_FS` |

---

#### `struct pinctrl_desc`
**الغرض:** الـ static descriptor اللي الـ driver بيملّيه وبيسجله — بيبقى `const`.

| Field | النوع | الوصف |
|-------|-------|-------|
| `name` | `const char *` | اسم الـ controller |
| `pins` | `const struct pinctrl_pin_desc *` | array بكل الـ pins |
| `npins` | `unsigned int` | عدد الـ pins |
| `pctlops` | `const struct pinctrl_ops *` | vtable الـ group management |
| `pmxops` | `const struct pinmux_ops *` | vtable الـ mux operations |
| `confops` | `const struct pinconf_ops *` | vtable الـ pin configuration |
| `owner` | `struct module *` | الـ module المالك |
| `link_consumers` | `bool` | عمل device link مع الـ consumers للـ suspend/resume ordering |

---

#### `struct pinctrl_gpio_range`
**الغرض:** بيربط range من الـ GPIO numbers بـ pins معينة في الـ controller.

| Field | النوع | الوصف |
|-------|-------|-------|
| `node` | `struct list_head` | في قائمة `pinctrl_dev->gpio_ranges` |
| `name` | `const char *` | اسم الـ GPIO chip |
| `id` | `unsigned int` | ID الـ chip |
| `base` | `unsigned int` | أول GPIO number في الـ range |
| `pin_base` | `unsigned int` | أول pin number مقابل الـ base GPIO |
| `npins` | `unsigned int` | عدد الـ pins في الـ range |
| `pins` | `const unsigned int *` | array اختياري بأرقام الـ pins (لو مش متتالية) |
| `gc` | `struct gpio_chip *` | الـ GPIO chip اللي بيملك الـ range |

---

### علاقات الـ Structs — ASCII Diagram

```
  ┌─────────────────────────────────────────────────────────────┐
  │                     pinctrl_desc (const)                    │
  │  name, pins[], npins                                        │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
  │  │ pinctrl_ops  │  │ pinmux_ops   │  │ pinconf_ops  │     │
  │  │ (pctlops)    │  │ (pmxops)     │  │ (confops)    │     │
  │  └──────────────┘  └──────────────┘  └──────────────┘     │
  └──────────────────────────┬──────────────────────────────────┘
                             │ desc →
                             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │                     pinctrl_dev (runtime)                    │
  │  mutex, dev, owner, driver_data                             │
  │                                                             │
  │  pin_desc_tree (radix)                                      │
  │    ├── pin#0 → pin_desc { mux_usecount, mux_owner,         │
  │    │                      mux_setting*, gpio_owner,         │
  │    │                      mux_lock }                        │
  │    ├── pin#1 → pin_desc                                     │
  │    └── ...                                                  │
  │                                                             │
  │  pin_function_tree (radix)  [GENERIC_PINMUX_FUNCTIONS]      │
  │    ├── 0 → function_desc { func → pinfunction, data }      │
  │    ├── 1 → function_desc                                    │
  │    └── ...                                                  │
  │                                                             │
  │  pin_group_tree (radix)     [GENERIC_PINCTRL_GROUPS]        │
  │    ├── 0 → group_desc { grp → pingroup { pins[] }, data }  │
  │    └── ...                                                  │
  │                                                             │
  │  gpio_ranges (list)                                         │
  │    └── pinctrl_gpio_range { base, pin_base, gc* }          │
  │                                                             │
  │  p → pinctrl (for hogging)                                  │
  │        └── states (list)                                    │
  │              └── pinctrl_state                              │
  │                    └── settings (list)                      │
  │                          └── pinctrl_setting                │
  │                                └── data.mux → setting_mux  │
  │                                      { func, group }        │
  └──────────────────────────────────────────────────────────────┘

  pin_desc->mux_setting ─────────────────────────────────┐
                                                         ▼
                                              pinctrl_setting_mux
                                              { func, group }
```

---

### Lifecycle Diagram — الـ Function Registration

```
Driver Probe
    │
    ▼
pinctrl_register_and_init(desc, dev, drv_data, &pctldev)
    │  يخلق pinctrl_dev، يملي pin_desc_tree
    │
    ▼
pinmux_check_ops(pctldev)
    │  يتحقق: get_functions_count, get_function_name,
    │          get_function_groups, set_mux كلهم موجودين
    │  لو ناقص حاجة → -EINVAL
    │
    ▼
[GENERIC_PINMUX_FUNCTIONS]
pinmux_generic_add_pinfunction(pctldev, func, data)
    │  يخلق function_desc بـ devm_kzalloc
    │  يعمل radix_tree_insert في pin_function_tree
    │  يزود num_functions++
    │
    ▼
pinctrl_enable(pctldev)
    │  يضيف الـ controller للقائمة العامة
    │  يعمل hog للـ pins اللي عليها default-state
    │
    ▼
[Controller Ready — ينتظر طلبات الـ consumers]


Driver Remove
    │
    ▼
pinmux_generic_free_functions(pctldev)   [أو pinmux_generic_remove_function]
    │  يمسح كل الـ entries من pin_function_tree
    │  devm_kfree تتولى تحرير الذاكرة
    │
    ▼
pinctrl_unregister(pctldev)
    │  يشيل الـ controller من القائمة العامة
    │
    ▼
[Done]
```

---

### Lifecycle Diagram — الـ Mux Setting (Consumer Device)

```
Consumer device (مثلاً UART driver) يطلب pin state
    │
    ▼
pinctrl_get(dev)   →  يخلق struct pinctrl
    │
    ▼
pinctrl_lookup_state(p, "default")  →  pinctrl_state
    │
    ▼
pinctrl_select_state(p, state)
    │
    │  for each setting in state->settings:
    ▼
pinmux_enable_setting(setting)
    │
    ├─ pctlops->get_group_pins(group) ──→ pins[]
    │
    ├─ for each pin:
    │    pin_request(pctldev, pin, dev_name, NULL)
    │       ├─ pin_desc_get() ──→ desc
    │       ├─ lock(desc->mux_lock)
    │       ├─ تحقق من mux_usecount / mux_owner / gpio_owner
    │       ├─ desc->mux_usecount++
    │       ├─ desc->mux_owner = dev_name
    │       ├─ unlock(desc->mux_lock)
    │       ├─ try_module_get(pctldev->owner)
    │       └─ ops->request(pctldev, pin)   [اختياري]
    │
    ├─ for each pin:
    │    lock(desc->mux_lock)
    │    desc->mux_setting = &setting->data.mux
    │    unlock(desc->mux_lock)
    │
    └─ ops->set_mux(pctldev, func, group)  ──→ يكتب على الـ HW register
           │
           ▼
       [Pins now muxed to the requested function]


Consumer device يتحرر
    │
    ▼
pinmux_disable_setting(setting)
    │
    ├─ pctlops->get_group_pins(group) ──→ pins[]
    │
    └─ for each pin:
         lock(desc->mux_lock)
         is_equal = (desc->mux_setting == &setting->data.mux)
         unlock(desc->mux_lock)
         if is_equal:
           pin_free(pctldev, pin, NULL)
             ├─ desc->mux_usecount--
             ├─ لو وصل لصفر: desc->mux_owner = NULL, mux_setting = NULL
             ├─ ops->free(pctldev, pin)   [اختياري]
             └─ module_put(pctldev->owner)
```

---

### Lifecycle Diagram — GPIO Pin Request

```
GPIO consumer يطلب gpio#42
    │
    ▼
gpiochip API  →  pinctrl_gpio_request(gpio=42)
    │
    ▼
pinmux_request_gpio(pctldev, range, pin, gpio)
    │
    ├─ kasprintf("%s:%d", range->name, gpio)  →  owner = "gpiochip0:42"
    │
    └─ pin_request(pctldev, pin, owner, range)
           ├─ pin_desc_get()
           ├─ lock(desc->mux_lock)
           ├─ لو strict وفيه mux_owner أو لو الـ pin مشغول → error
           ├─ desc->gpio_owner = owner
           ├─ unlock(desc->mux_lock)
           ├─ try_module_get(pctldev->owner)
           └─ ops->gpio_request_enable(pctldev, range, pin)


GPIO consumer بيحرر gpio#42
    │
    ▼
pinctrl_gpio_free(gpio=42)
    │
    ▼
pinmux_free_gpio(pctldev, pin, range)
    │
    └─ pin_free(pctldev, pin, range)
           ├─ lock(desc->mux_lock)
           ├─ owner = desc->gpio_owner
           ├─ desc->gpio_owner = NULL
           ├─ unlock(desc->mux_lock)
           ├─ ops->gpio_disable_free(pctldev, range, pin)
           ├─ module_put(pctldev->owner)
           └─ returns owner string
    │
    └─ kfree(owner)   [الـ string اتعملت kasprintf]
```

---

### Call Flow Diagrams

#### تفعيل الـ Mux

```
pinmux_enable_setting(setting)
  │
  ├─→ pctldev->desc->pctlops->get_group_pins(pctldev, group, &pins, &num_pins)
  │       └─ [HW driver returns pin numbers for this group]
  │
  ├─→ pin_request(pctldev, pins[i], dev_name, NULL)   [for each pin]
  │       ├─→ pin_desc_get(pctldev, pin)
  │       │       └─ radix_tree_lookup(&pctldev->pin_desc_tree, pin)
  │       │
  │       ├─→ [check ownership under mux_lock]
  │       │       if mux_usecount && mux_owner != requester → EBUSY
  │       │
  │       ├─→ try_module_get(pctldev->owner)
  │       │
  │       └─→ pctldev->desc->pmxops->request(pctldev, pin)   [optional]
  │               └─ [driver validates pin availability]
  │
  ├─→ [store mux_setting in each pin_desc under mux_lock]
  │
  └─→ pctldev->desc->pmxops->set_mux(pctldev, func, group)
          └─ [driver writes hardware MUX register]
                  └─ [Pin now routes to requested peripheral]
```

#### الـ Generic Function Lookup

```
pinmux_func_name_to_selector(pctldev, "uart0")
  │
  ├─→ pmxops->get_functions_count(pctldev)
  │       [GENERIC: return pctldev->num_functions]
  │
  └─→ loop: pmxops->get_function_name(pctldev, selector)
              [GENERIC: radix_tree_lookup(&pin_function_tree, selector)]
                          └─ function_desc->func->name
              if match → return selector
```

#### إضافة Function جديدة (GENERIC path)

```
pinmux_generic_add_pinfunction(pctldev, func, data)
  │
  ├─→ pinmux_func_name_to_selector()   [تحقق مكرر؟ → return existing]
  │
  ├─→ selector = pctldev->num_functions
  │
  ├─→ devm_kzalloc(pctldev->dev, sizeof(function_desc))
  │
  ├─→ devm_kmemdup_const(pctldev->dev, func, sizeof(*func))
  │       └─ copies pinfunction struct safely
  │
  ├─→ function->data = data
  │
  └─→ radix_tree_insert(&pctldev->pin_function_tree, selector, function)
          pctldev->num_functions++
          return selector
```

---

### استراتيجية الـ Locking

#### الـ Locks الموجودة وما بتحميه

| الـ Lock | النوع | الموجود في | بيحمي إيه |
|---------|-------|-----------|-----------|
| `pinctrl_dev->mutex` | `struct mutex` | `pinctrl_dev` | كل العمليات على الـ controller: قراءة/كتابة الـ functions/groups، debugfs، الـ gpio_ranges list |
| `pin_desc->mux_lock` | `struct mutex` | كل `pin_desc` | الـ fields الخاصة بالـ ownership: `mux_usecount`, `mux_owner`, `mux_setting`, `gpio_owner` |
| `pinctrl_maps_mutex` | `struct mutex` | global | قائمة `pinctrl_maps` العامة |

#### ترتيب الـ Locks (Lock Ordering)

```
pinctrl_dev->mutex     (خارجي — يتاخد أول)
    └── pin_desc->mux_lock   (داخلي — يتاخد جوا الـ pinctrl_dev->mutex)
```

**مهم:** كود الـ debugfs بياخد `pctldev->mutex` قبل ما يلوب على الـ pins ويعمل `scoped_guard(mutex, &desc->mux_lock)` — ده بيضمن الترتيب الصح ومفيش deadlock.

#### استخدام `scoped_guard` و `guard`

الكود بيستخدم الـ `guard(mutex)` و `scoped_guard(mutex, ...)` من `<linux/cleanup.h>` — ده automatic mutex unlock عند الخروج من الـ scope (حتى لو رجع بـ error). مثال:

```c
/* يعمل lock وبعدين unlock تلقائي عند نهاية الـ scope */
scoped_guard(mutex, &desc->mux_lock) {
    desc->mux_usecount++;
    desc->mux_owner = owner;
}
/* هنا الـ lock اتحرر تلقائي */
```

#### تفاصيل الـ `mux_usecount`

- الـ `mux_usecount` مش boolean — ممكن يكون أكبر من 1 لو multiple clients طلبوا نفس الـ group.
- أول طلب بيزود الـ count ويعمل `ops->request()`.
- الطلبات التانية بتزود الـ count وبترجع مباشرة بدون ما تعمل `request()` تاني.
- التحرير بينقص الـ count — بس لما يوصل صفر، بيتم `ops->free()` وتصفية الـ ownership.

```c
/* pin_request: من أول طلب بعد ده */
desc->mux_usecount++;
if (desc->mux_usecount > 1)
    return 0;  /* already owned, just increment */

desc->mux_owner = owner;
/* then call ops->request() */
```
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions — Cheatsheet

#### Group 1: Validation & Initialization

| Function | Signature (موجز) | الغرض |
|---|---|---|
| `pinmux_check_ops` | `int (pctldev)` | يتحقق إن الـ driver نفّذ الـ ops الإلزامية |
| `pinmux_validate_map` | `int (map, i)` | يتحقق إن الـ map entry فيها function name |

#### Group 2: GPIO Mux Interface

| Function | Signature (موجز) | الغرض |
|---|---|---|
| `pinmux_can_be_used_for_gpio` | `bool (pctldev, pin)` | يسأل: هل الـ pin ممكن يتحول لـ GPIO؟ |
| `pinmux_request_gpio` | `int (pctldev, range, pin, gpio)` | يحجز pin لـ GPIO |
| `pinmux_free_gpio` | `void (pctldev, pin, range)` | يحرر pin من الـ GPIO mode |
| `pinmux_gpio_direction` | `int (pctldev, range, pin, input)` | يضبط اتجاه الـ GPIO (input/output) |

#### Group 3: Setting Lifecycle (Core Internal)

| Function | Signature (موجز) | الغرض |
|---|---|---|
| `pinmux_map_to_setting` | `int (map, setting)` | يحوّل الـ map entry لـ setting قابل للتنفيذ |
| `pinmux_free_setting` | `void (setting)` | placeholder — لا تعمل حالياً |
| `pinmux_enable_setting` | `int (setting)` | يطبّق الـ mux setting على الـ hardware |
| `pinmux_disable_setting` | `void (setting)` | يلغي الـ mux setting ويحرر الـ pins |

#### Group 4: Low-level Pin Ownership (Static — Internal)

| Function | Signature (موجز) | الغرض |
|---|---|---|
| `pin_request` | `int (pctldev, pin, owner, gpio_range)` | يحجز pin واحد ويسجل owner |
| `pin_free` | `const char *(pctldev, pin, gpio_range)` | يحرر pin واحد ويرجع اسم الـ owner |
| `pinmux_func_name_to_selector` | `int (pctldev, function)` | يبحث عن selector بناءً على اسم الـ function |

#### Group 5: Generic Functions API (CONFIG_GENERIC_PINMUX_FUNCTIONS)

| Function | Export | الغرض |
|---|---|---|
| `pinmux_generic_get_function_count` | GPL | عدد الـ functions المسجلة |
| `pinmux_generic_get_function_name` | GPL | اسم function من الـ selector |
| `pinmux_generic_get_function_groups` | GPL | الـ groups المرتبطة بـ function |
| `pinmux_generic_get_function` | GPL | يرجع `function_desc` كاملة |
| `pinmux_generic_function_is_gpio` | GPL | هل الـ function هي GPIO؟ |
| `pinmux_generic_add_function` | GPL | يضيف function بالاسم والـ groups |
| `pinmux_generic_add_pinfunction` | GPL | يضيف function من `pinfunction` struct |
| `pinmux_generic_remove_function` | GPL | يحذف function بالـ selector |
| `pinmux_generic_free_functions` | internal | يحذف كل الـ functions |

#### Group 6: DebugFS

| Function | الغرض |
|---|---|
| `pinmux_init_device_debugfs` | ينشئ ملفات debugfs للـ device |
| `pinmux_functions_show` | يعرض قائمة الـ functions في debugfs |
| `pinmux_pins_show` | يعرض حالة كل pin في debugfs |
| `pinmux_show_map` | يطبع معلومات الـ map entry |
| `pinmux_show_setting` | يطبع معلومات الـ setting المفعّل |
| `pinmux_select_write` | يسمح بضبط الـ mux يدوياً من debugfs |

---

### Group 1: Validation & Initialization

الـ core يستدعي الاتنين دول أثناء تسجيل الـ pin controller، قبل ما أي driver يشتغل. هدفهم التحقق من صحة الـ driver descriptor والـ map table اللي جوّاه.

---

#### `pinmux_check_ops`

```c
int pinmux_check_ops(struct pinctrl_dev *pctldev);
```

بيتحقق إن الـ `pinmux_ops` اللي سجّلها الـ driver فيها الـ callbacks الإلزامية: `get_functions_count`، `get_function_name`، `get_function_groups`، و`set_mux`. لو أي واحدة ناقصة يرجع `-EINVAL`. كمان بيعدّي على كل الـ functions ويتأكد إن كل واحدة ليها اسم غير NULL.

**Parameters:**
- `pctldev` — الـ pin controller device المسجّل

**Return:** `0` لو تمام، `-EINVAL` لو أي op ناقصة أو function بدون اسم.

**Key Details:**
- بيُستدعى من `pinctrl_register()` / `devm_pinctrl_register()` مرة واحدة.
- لا يحتاج lock لأن الـ device لسه في طور التسجيل.
- الـ `ops->get_function_name()` بتتعمل loop على كل الـ functions — عملية O(n).

**Who calls it:** `pinctrl_register_and_init()` → `pinctrl_register()`.

---

#### `pinmux_validate_map`

```c
int pinmux_validate_map(const struct pinctrl_map *map, int i);
```

يتحقق إن الـ `pinctrl_map` entry من نوع MUX فيها `function` name غير NULL. بسيطة جداً — guard بدائي يمنع الـ null dereference لاحقاً.

**Parameters:**
- `map` — الـ map entry المراد التحقق منها
- `i` — رقمها في الـ table (للـ error message بس)

**Return:** `0` أو `-EINVAL`.

**Who calls it:** `pinctrl_register_mappings()` تعمل loop على كل الـ entries وتستدعيها.

---

### Group 2: GPIO Mux Interface

الـ GPIO subsystem لما بيطلب pin معين، بيمر بالـ pinmux layer عشان يتأكد إن الـ pin مش محجوز لحاجة تانية، وعشان يضبط الـ multiplexer في الـ hardware للـ GPIO mode. الـ functions دي هي الواجهة بين `gpiolib` والـ pinmux core.

---

#### `pinmux_can_be_used_for_gpio`

```c
bool pinmux_can_be_used_for_gpio(struct pinctrl_dev *pctldev, unsigned int pin);
```

بيسأل: هل الـ pin ده ممكن يتحول لـ GPIO دلوقتي؟ لو الـ controller مش `strict`، دايماً يرجع `true`. لو `strict`، بيبص على `mux_usecount` و`gpio_owner` — لو الـ pin محجوز لـ function مش GPIO يرجع `false`.

**Parameters:**
- `pctldev` — الـ pin controller
- `pin` — رقم الـ pin في الـ global namespace

**Return:** `true` لو الـ pin متاح لـ GPIO، `false` لو محجوز لـ function تانية.

**Key Details:**
- يأخذ `desc->mux_lock` بالـ `guard(mutex)` — scoped locking تلقائي.
- لو `desc == NULL` أو `ops == NULL` يرجع `true` افتراضياً (conservative).
- بيستخدم `ops->function_is_gpio()` لو موجودة عشان يميز GPIO function من غيرها.

**Who calls it:** `pinctrl_gpio_request()` في `core.c`.

---

#### `pinmux_request_gpio`

```c
int pinmux_request_gpio(struct pinctrl_dev *pctldev,
                        struct pinctrl_gpio_range *range,
                        unsigned int pin, unsigned int gpio);
```

يحجز pin واحد لاستخدامه كـ GPIO. بيبني اسم الـ owner تلقائياً على شكل `"range_name:gpio_number"` باستخدام `kasprintf`، ثم يستدعي `pin_request()`.

**Parameters:**
- `pctldev` — الـ controller
- `range` — الـ GPIO range اللي فيها الـ GPIO ده
- `pin` — رقم الـ pin في الـ controller
- `gpio` — رقم الـ GPIO العالمي (لبناء اسم الـ owner)

**Return:** `0` أو error code سالب.

**Key Details:**
- الـ owner string تتخصص بـ `kasprintf(GFP_KERNEL, ...)` — ممكن تفشل بـ `-ENOMEM`.
- لو `pin_request()` فشل، الـ owner string بتتحرر فوراً بـ `kfree`.
- لو نجح، الـ owner string بتتحفظ في `desc->gpio_owner` وبتتحرر لاحقاً في `pinmux_free_gpio`.

**Who calls it:** `pinctrl_gpio_request()` في `core.c`.

---

#### `pinmux_free_gpio`

```c
void pinmux_free_gpio(struct pinctrl_dev *pctldev, unsigned int pin,
                      struct pinctrl_gpio_range *range);
```

يحرر pin من الـ GPIO mode. بيستدعي `pin_free()` اللي بترجع pointer للـ owner string، ثم بيحررها بـ `kfree` — لأن الـ string اتخصصت ديناميكياً في `pinmux_request_gpio`.

**Parameters:**
- `pctldev`, `pin`, `range` — نفس `pinmux_request_gpio`

**Return:** void

**Who calls it:** `pinctrl_gpio_free()` في `core.c`.

---

#### `pinmux_gpio_direction`

```c
int pinmux_gpio_direction(struct pinctrl_dev *pctldev,
                          struct pinctrl_gpio_range *range,
                          unsigned int pin, bool input);
```

يضبط اتجاه الـ GPIO (input أو output) عن طريق استدعاء `ops->gpio_set_direction()`. لو الـ callback مش موجودة، يرجع `0` — الـ hardware ممكن يتعامل مع الاتجاه بطريقة تانية.

**Parameters:**
- `pctldev` — الـ controller
- `range` — الـ GPIO range
- `pin` — رقم الـ pin
- `input` — `true` للـ input، `false` للـ output

**Return:** قيمة `ops->gpio_set_direction()` أو `0`.

**Who calls it:** `pinctrl_gpio_direction_input/output()` في `core.c`.

---

### Group 3: Setting Lifecycle

الـ pinctrl core بتدير state machine — كل device بيقدر يكون في state معين (مثلاً `default`، `sleep`). كل state بيحتوي على قائمة `pinctrl_setting`. الـ functions دي هي المسؤولة عن تحويل الـ map table entries لـ settings، وتفعيلها، وإلغاءها.

---

#### `pinmux_map_to_setting`

```c
int pinmux_map_to_setting(const struct pinctrl_map *map,
                          struct pinctrl_setting *setting);
```

يحوّل `pinctrl_map` entry (اللي بيحدد function name وgroup name كـ strings) إلى `pinctrl_setting` (اللي بيحتوي على selectors أرقام جاهزة للاستخدام). بيعمل name lookup للـ function وللـ group.

**Parameters:**
- `map` — الـ map entry من الـ mapping table (strings)
- `setting` — الـ setting المراد ملؤه (يُعدَّل in-place)

**Return:** `0` أو error code سالب.

**Key Details — Pseudocode:**

```
1. تحقق إن pctldev فيه pmxops
2. استدعِ pinmux_func_name_to_selector(map->data.mux.function)
   → يرجع fsel أو -EINVAL
3. setting->data.mux.func = fsel
4. استدعِ pmxops->get_function_groups(fsel)
   → يرجع groups[] و num_groups
5. لو map->data.mux.group محدد:
     match_string(groups, group_name) → تحقق إن الـ group صح
   لو مش محدد:
     استخدم groups[0] (default)
6. استدعِ pinctrl_get_group_selector(group_name)
   → يرجع gsel
7. setting->data.mux.group = gsel
8. return 0
```

**Who calls it:** `pinctrl_map_add_setting()` في `core.c` عند بناء الـ state.

---

#### `pinmux_free_setting`

```c
void pinmux_free_setting(const struct pinctrl_setting *setting);
```

Stub فاضية حالياً — placeholder لمستقبل لو احتاج الـ setting تحرير resources. مفيش حاجة تتعمل جوّاها.

---

#### `pinmux_enable_setting`

```c
int pinmux_enable_setting(const struct pinctrl_setting *setting);
```

يطبّق mux setting على الـ hardware — ده القلب الحقيقي للـ pinmux core. بيعمل 3 خطوات: يجيب الـ pins من الـ group، يحجزهم واحداً واحداً، يضبط الـ mux في الـ hardware.

**Parameters:**
- `setting` — الـ setting اللي فيه `func` و`group` selectors

**Return:** `0` أو error code سالب.

**Key Details — Pseudocode:**

```
1. استدعِ pctlops->get_group_pins(group) → pins[], num_pins
   (لو فشل: warning فقط، num_pins = 0)

2. for i in 0..num_pins:
     pin_request(pctldev, pins[i], setting->dev_name, NULL)
     لو فشل: اطبع error → goto err_pin_request

3. for i in 0..num_pins:
     mutex_lock(desc->mux_lock)
     desc->mux_setting = &setting->data.mux
     mutex_unlock(desc->mux_lock)

4. ops->set_mux(pctldev, func, group)
   لو فشل: goto err_set_mux

5. return 0

err_set_mux:
   امسح mux_setting من كل الـ descs
err_pin_request:
   حرر كل الـ pins اللي اتحجزت (بالعكس)
   return ret
```

**Locking:**
- `desc->mux_lock` لكل pin على حدا (per-pin mutex) باستخدام `scoped_guard`.
- الـ `pctldev->mutex` مش مأخود هنا — المسؤولية على الـ caller.

**Side Effects:**
- يزوّد `mux_usecount` لكل pin.
- يضبط `desc->mux_setting` pointer.
- يستدعي `try_module_get()` عبر `pin_request()` — يزوّد refcount للـ module.

**Who calls it:** `pinctrl_commit_state()` في `core.c`.

---

#### `pinmux_disable_setting`

```c
void pinmux_disable_setting(const struct pinctrl_setting *setting);
```

يلغي setting مفعّل — عكس `pinmux_enable_setting`. بيمشي على الـ pins في الـ group ويتحقق إن كل pin لسه مرتبط بالـ setting ده (مش setting تاني اتطبق عليه)، لو نعم يحرره بـ `pin_free`.

**Parameters:**
- `setting` — الـ setting المراد إلغاؤه

**Return:** void

**Key Details:**
- بيتحقق `desc->mux_setting == &setting->data.mux` قبل الـ `pin_free` — ده important جداً عشان منحرّرش pin محجوز بـ setting تاني.
- لو الـ pin مش مرتبط بالـ setting ده، بيطبع warning بس ومبيحرروش.
- اللـ `set_mux` callback مش بتتعكس هنا — الـ hardware state مش بيترجعش تلقائياً.

**Who calls it:** `pinctrl_commit_state()` عند الانتقال من state لـ state تاني.

---

### Group 4: Low-level Pin Ownership (Static — Internal)

دول الـ functions الأساسية اللي بتدير ملكية كل pin على مستوى `pin_desc`. مش exported — internal للـ pinmux.c بس.

---

#### `pin_request` (static)

```c
static int pin_request(struct pinctrl_dev *pctldev,
                       int pin, const char *owner,
                       struct pinctrl_gpio_range *gpio_range);
```

يحجز pin واحد لـ owner معين. لو `gpio_range != NULL` فده طلب GPIO، لو `NULL` فده طلب mux. بيدير `mux_usecount` (reference counting) ويمنع الـ conflicts حسب `strict` mode.

**Parameters:**
- `pctldev` — الـ controller
- `pin` — رقم الـ pin
- `owner` — string اسم الـ owner (device name أو GPIO name)
- `gpio_range` — لو طلب GPIO، الـ range المناسبة؛ لو NULL فطلب mux

**Return:** `0` أو error code سالب.

**Key Details:**

```
Conflict Detection Logic:
┌─────────────────────────────────────────────────────────────┐
│ هل gpio_range == NULL (mux request)?                         │
│   - لو mux_usecount > 0 && owner != mux_owner → EBUSY      │
│   - (إلا لو function_is_gpio → مسموح بالشارك)              │
│                                                              │
│ هل gpio_range != NULL (GPIO request)?                        │
│   - لو gpio_owner != NULL → EBUSY                           │
│   - لو strict && mux_usecount > 0 → EBUSY                  │
└─────────────────────────────────────────────────────────────┘
```

**Reference Counting:**
- لو mux request: `mux_usecount++` — لو > 1 يرجع `0` مباشرة بدون استدعاء `ops->request()` مرة تانية.
- لو GPIO: `gpio_owner = owner` مباشرة (مفيش usecount).

**Module Refcount:**
- `try_module_get(pctldev->owner)` — يمنع unload الـ module لو فيه pins محجوزة.
- لو `ops->gpio_request_enable` أو `ops->request` فشلت → `module_put()` + rollback.

**Error Path:**
- يعمل rollback كامل: يشيل الـ owner ويرجع `mux_usecount`.
- يستخدم `dev_err_probe` للـ logging عشان يتعامل صح مع `-EPROBE_DEFER`.

---

#### `pin_free` (static)

```c
static const char *pin_free(struct pinctrl_dev *pctldev, int pin,
                            struct pinctrl_gpio_range *gpio_range);
```

يحرر pin واحد. بيرجع pointer للـ owner string عشان الـ caller يحررها لو اتخصصت ديناميكياً (زي حالة GPIO).

**Parameters:**
- `pctldev` — الـ controller
- `pin` — رقم الـ pin
- `gpio_range` — لو GPIO release، لو NULL فـ mux release

**Return:** الـ owner string السابقة (الـ caller مسؤول عن تحريرها لو محتاج) أو `NULL` لو error.

**Key Details:**
- لو mux: `mux_usecount--` — لو لسه > 0 يرجع `NULL` بدون تحرير (reference counting).
- لو GPIO: يشيل `gpio_owner` مباشرة.
- يستدعي `ops->gpio_disable_free` أو `ops->free` حسب النوع.
- `module_put(pctldev->owner)` دايماً في الآخر.
- لو `mux_usecount == 0` بعد الـ decrement: يشيل `mux_setting` pointer.

---

#### `pinmux_func_name_to_selector` (static)

```c
static int pinmux_func_name_to_selector(struct pinctrl_dev *pctldev,
                                        const char *function);
```

بيعمل linear search على كل الـ functions المسجلة عشان يلاقي selector بناءً على الاسم. O(n) في عدد الـ functions.

**Parameters:**
- `pctldev` — الـ controller
- `function` — اسم الـ function المطلوب

**Return:** الـ selector (رقم غير سالب) أو `-EINVAL` لو مش موجود.

**Who calls it:** `pinmux_map_to_setting`، `pinmux_generic_add_pinfunction`، `pinmux_select_write`.

---

### Group 5: Generic Functions API

لو الـ driver بيستخدم `CONFIG_GENERIC_PINMUX_FUNCTIONS`، مش محتاج يبني الـ function registry بتاعه من الصفر. الـ core بيوفر radix tree (`pin_function_tree`) جاهز ومجموعة functions كاملة لإدارته.

الـ `struct function_desc` هي الـ wrapper:

```c
struct function_desc {
    const struct pinfunction *func;  /* name + groups[] + flags */
    void *data;                      /* driver-private data */
};
```

---

#### `pinmux_generic_get_function_count`

```c
int pinmux_generic_get_function_count(struct pinctrl_dev *pctldev);
```

يرجع `pctldev->num_functions` مباشرة.

**Who calls it:** الـ driver بيحطها في `pmxops->get_functions_count`.

---

#### `pinmux_generic_get_function_name`

```c
const char *pinmux_generic_get_function_name(struct pinctrl_dev *pctldev,
                                              unsigned int selector);
```

يعمل `radix_tree_lookup(&pctldev->pin_function_tree, selector)` ويرجع `function->func->name`. لو مش موجود يرجع `NULL`.

---

#### `pinmux_generic_get_function_groups`

```c
int pinmux_generic_get_function_groups(struct pinctrl_dev *pctldev,
                                       unsigned int selector,
                                       const char * const **groups,
                                       unsigned int * const ngroups);
```

يجيب الـ function من الـ radix tree ويملي `*groups` بـ `func->groups` و`*ngroups` بـ `func->ngroups`.

**Return:** `0` أو `-EINVAL` لو الـ selector مش موجود.

---

#### `pinmux_generic_get_function`

```c
const struct function_desc *
pinmux_generic_get_function(struct pinctrl_dev *pctldev, unsigned int selector);
```

يرجع الـ `function_desc` كاملة — للـ drivers اللي تحتاج الـ `data` pointer.

---

#### `pinmux_generic_function_is_gpio`

```c
bool pinmux_generic_function_is_gpio(struct pinctrl_dev *pctldev,
                                     unsigned int selector);
```

يبص على `function->func->flags & PINFUNCTION_FLAG_GPIO`. الـ driver بيحط الـ flag ده عند تعريف الـ GPIO function.

**Who calls it:** الـ driver بيحطها في `pmxops->function_is_gpio`.

---

#### `pinmux_generic_add_function`

```c
int pinmux_generic_add_function(struct pinctrl_dev *pctldev,
                                const char *name,
                                const char * const *groups,
                                const unsigned int ngroups,
                                void *data);
```

Convenience wrapper — بيبني `pinfunction` struct باستخدام `PINCTRL_PINFUNCTION(name, groups, ngroups)` ثم يستدعي `pinmux_generic_add_pinfunction`.

**Return:** الـ selector الجديد أو error code.

---

#### `pinmux_generic_add_pinfunction`

```c
int pinmux_generic_add_pinfunction(struct pinctrl_dev *pctldev,
                                   const struct pinfunction *func, void *data);
```

ده الـ function الأساسية اللي بتضيف function فعلاً في الـ radix tree.

**Key Details — Flow:**

```
1. pinmux_func_name_to_selector(func->name)
   لو موجود → يرجع الـ selector الموجود (idempotent)

2. selector = pctldev->num_functions

3. devm_kzalloc(pctldev->dev, sizeof(function_desc))
   → memoized على الـ device lifecycle

4. devm_kmemdup_const(func) → نسخة من الـ pinfunction

5. radix_tree_insert(&pin_function_tree, selector, function)

6. num_functions++

7. return selector
```

**Important:** بيستخدم `devm_*` — الـ memory بتتحرر تلقائياً عند `device_unregister`. الـ kernel نفسه بيعلّق على ده كـ "technical debt" في الكود (FIXME comment).

**Return:** الـ selector (>= 0) أو error code سالب.

---

#### `pinmux_generic_remove_function`

```c
int pinmux_generic_remove_function(struct pinctrl_dev *pctldev,
                                   unsigned int selector);
```

يحذف function من الـ radix tree بـ `radix_tree_delete` ويحرر الـ `function_desc` بـ `devm_kfree`. يقلل `num_functions`.

**Note:** الـ caller مسؤول عن الـ locking.

---

#### `pinmux_generic_free_functions`

```c
void pinmux_generic_free_functions(struct pinctrl_dev *pctldev);
```

يعمل `radix_tree_for_each_slot` ويحذف كل الـ entries. مبيحررش الـ `function_desc` struct نفسه لأن الـ `devm_*` بيعمل ده تلقائياً.

---

### Group 6: DebugFS

الـ debugfs interface بيوفر رؤية real-time لحالة الـ pinmux في runtime — مفيد جداً للـ debugging على الـ embedded targets.

---

#### `pinmux_init_device_debugfs`

```c
void pinmux_init_device_debugfs(struct dentry *devroot,
                                struct pinctrl_dev *pctldev);
```

ينشئ 3 ملفات تحت `devroot` في debugfs:

| الملف | الصلاحيات | الوظيفة |
|---|---|---|
| `pinmux-functions` | `0444` | قراءة كل الـ functions والـ groups |
| `pinmux-pins` | `0444` | حالة كل pin (mux_owner, gpio_owner) |
| `pinmux-select` | `0200` | كتابة fقط — ضبط mux يدوياً |

---

#### `pinmux_functions_show`

```c
static int pinmux_functions_show(struct seq_file *s, void *what);
```

يمشي على كل الـ functions بـ `get_function_name` و`get_function_groups` ويطبع:
```
function 0: uart0, groups = [ uart0_pins uart0_flow ]
function 1: spi0, groups = [ spi0_pins ]
```

يأخذ `pctldev->mutex` طول الـ iteration.

---

#### `pinmux_pins_show`

```c
static int pinmux_pins_show(struct seq_file *s, void *what);
```

يمشي على كل الـ pins في الـ controller ويطبع حالة كل pin. الـ output بيختلف حسب `strict` mode:

```
# strict mode:
pin 42 (GPIO_B10): device uart0 (HOG) function uart0 group uart0_pins

# non-strict mode:
pin 42 (GPIO_B10): uart0 (GPIO UNCLAIMED) function uart0 group uart0_pins
```

يأخذ `pctldev->mutex` (outer) و`desc->mux_lock` (per-pin، inner) — ترتيب الـ locking ثابت لتجنب deadlock.

---

#### `pinmux_select_write`

```c
static ssize_t pinmux_select_write(struct file *file,
                                   const char __user *user_buf,
                                   size_t len, loff_t *ppos);
```

بيسمح للمطور يكتب مباشرة في debugfs لضبط الـ mux. الـ format المتوقع:
```bash
echo "group_name function_name" > /sys/kernel/debug/pinctrl/DEVICE/pinmux-select
```

**Flow:**
```
1. memdup_user_nul → copy from userspace
2. strstrip → remove leading/trailing spaces
3. split on first whitespace → gname / fname
4. pinmux_func_name_to_selector(fname) → fsel
5. get_function_groups(fsel) → validate gname
6. pinctrl_get_group_selector(gname) → gsel
7. ops->set_mux(pctldev, fsel, gsel)
```

**Note:** مبيعملش `pin_request` — بيكتب الـ hardware مباشرة بدون tracking. للـ debugging فقط، مش للإنتاج.

---

#### `pinmux_show_map` / `pinmux_show_setting`

```c
void pinmux_show_map(struct seq_file *s, const struct pinctrl_map *map);
void pinmux_show_setting(struct seq_file *s, const struct pinctrl_setting *setting);
```

دول helpers بيستدعيهم الـ core لطباعة معلومات الـ map والـ setting في debugfs. مباشرة ومفيش complexity.

---

### ملاحظات عامة على الـ Locking Model

```
┌──────────────────────────────────────────────────────────────┐
│                    Locking Hierarchy                          │
│                                                              │
│  pctldev->mutex          (per-controller, coarse-grained)    │
│      └── desc->mux_lock  (per-pin, fine-grained)             │
│                                                              │
│  القاعدة: دايماً خد pctldev->mutex الأول لو محتاجهم معاً   │
│  pinmux_enable_setting: مش بياخد pctldev->mutex             │
│  (المسؤولية على pinctrl_commit_state في core.c)             │
└──────────────────────────────────────────────────────────────┘
```

الـ `scoped_guard(mutex, &desc->mux_lock)` اللي بتظهر كتير في الكود هي الـ cleanup-based locking من `<linux/cleanup.h>` — بتضمن إن الـ lock بيتحرر تلقائياً حتى لو اتعمل `goto` أو `return` جوّا الـ scope.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ pinmux subsystem بيسجّل ملفات debugfs تلقائيًا عن طريق `pinmux_init_device_debugfs()` لما يتفعّل `CONFIG_DEBUG_FS`.

| المسار | الصلاحية | المحتوى |
|--------|----------|---------|
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-functions` | 0444 | كل الـ functions المسجّلة + الـ groups بتاعتها |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-pins` | 0444 | كل pin وحالته: owner، gpio_owner، function، group |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-select` | 0200 | Write-only: force mux setting يدوي |
| `/sys/kernel/debug/pinctrl/<dev>/pinctrl-handles` | 0444 | الـ handles النشطة + الـ settings |
| `/sys/kernel/debug/pinctrl/<dev>/pinctrl-maps` | 0444 | جدول الـ mapping الكامل للـ machine |
| `/sys/kernel/debug/pinctrl/pinctrl-handles` | 0444 | global view لكل الـ handles |
| `/sys/kernel/debug/gpio` | 0444 | الـ GPIO lines + الـ requesters — مرتبط بالـ pinmux |

**قراءة الـ pinmux-pins:**

```bash
# اقرأ حالة كل pin مع مين طالبه
cat /sys/kernel/debug/pinctrl/$(ls /sys/kernel/debug/pinctrl/ | head -1)/pinmux-pins
```

**مثال على الـ output:**
```
Pinmux settings per pin
Format: pin (name): mux_owner gpio_owner hog?
pin 0 (GPIO0): (MUX UNCLAIMED) (GPIO UNCLAIMED)
pin 5 (UART0_TX): uart0 (GPIO UNCLAIMED) function uart0 group uart0_grp
pin 6 (UART0_RX): uart0 (GPIO UNCLAIMED) function uart0 group uart0_grp
pin 10 (SPI0_CLK): pinctrl-imx (HOG) function spi0 group spi0_grp
pin 20 (GPIO20): (MUX UNCLAIMED) gpio-keys:20
```

**التفسير:**
- `(MUX UNCLAIMED)` → الـ pin مش مأخود لأي function
- `(HOG)` → الـ pin controller نفسه طالبه من الـ DT (`pinctrl-0` في نود الـ controller)
- `gpio-keys:20` → طلبته الـ driver دي كـ GPIO

**قراءة الـ pinmux-functions:**

```bash
cat /sys/kernel/debug/pinctrl/$(ls /sys/kernel/debug/pinctrl/ | head -1)/pinmux-functions
```

**مثال:**
```
function 0: gpio, groups = [ gpio0 gpio1 gpio2 ... ]
function 1: uart0, groups = [ uart0_grp uart0_noflow_grp ]
function 2: spi0, groups = [ spi0_grp spi0_quad_grp ]
function 3: i2c1, groups = [ i2c1_grp ]
```

**Force mux يدوي للاختبار (write إلى pinmux-select):**

```bash
# تفعيل function "uart0" على group "uart0_grp"
echo "uart0_grp uart0" > /sys/kernel/debug/pinctrl/<dev>/pinmux-select
```

> تحذير: `pinmux-select` بيستدعي `set_mux()` مباشرة بدون أي resource tracking — للاختبار بس.

---

#### 2. مدخلات الـ sysfs

```bash
# قائمة بكل الـ pinctrl controllers المسجّلة
ls /sys/bus/platform/drivers/pinctrl/
ls /sys/class/gpio/

# معلومات الـ device
cat /sys/bus/platform/devices/<pinctrl-dev>/driver
ls /sys/bus/platform/devices/<pinctrl-dev>/

# الـ GPIO lines عبر libgpiod
gpiodetect          # كل الـ GPIO chips
gpioinfo gpiochip0  # كل lines + requesters
```

---

#### 3. الـ ftrace — Tracepoints والـ Events

الـ pinctrl subsystem عنده tracepoints في `include/trace/events/pinctrl.h`:

```bash
# عرض كل الـ trace events المتاحة للـ pinctrl
ls /sys/kernel/debug/tracing/events/pinctrl/

# تفعيل كل الـ pinctrl events
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# أو events محددة
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_map_add/enable
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_state_init/enable

# تتبع مع function tracer لحظة probe
echo function > /sys/kernel/debug/tracing/current_tracer
echo 'pinmux*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل العملية المشبوهة ...
cat /sys/kernel/debug/tracing/trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

**تتبع `pin_request` و `pin_free` بالـ kprobe:**

```bash
# أضف kprobe على pin_request
echo 'p:pin_req pin_request pctldev=%di pin=%si owner=%dx' \
    > /sys/kernel/debug/tracing/kprobe_events

echo 1 > /sys/kernel/debug/tracing/events/kprobes/pin_req/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

**مثال output من الـ trace:**
```
     init-1    [000] ....  5.123: pin_req: (pin_request+0x0/0x140) pctldev=0xffff... pin=5 owner=0xffff...
     init-1    [000] ....  5.124: pin_req: pctldev=0xffff... pin=6 owner=uart0
```

---

#### 4. الـ printk والـ Dynamic Debug

الكود بيستخدم `dev_dbg()` بشكل أساسي في `pin_request()`:

```c
dev_dbg(pctldev->dev, "request pin %d (%s) for %s\n",
        pin, desc->name, owner);
```

**تفعيل الـ dynamic debug للـ pinmux subsystem:**

```bash
# تفعيل كل messages الـ pinmux core
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control

# أو تفعيل ملف محدد
echo 'file drivers/pinctrl/pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع stack trace
echo 'file drivers/pinctrl/pinmux.c +ps' > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل الـ pinctrl subsystem
echo 'file drivers/pinctrl/* +p' > /sys/kernel/debug/dynamic_debug/control

# إيقاف
echo 'file drivers/pinctrl/pinmux.c -p' > /sys/kernel/debug/dynamic_debug/control
```

**أو عبر kernel command line (للمشاكل في الـ boot):**

```
dyndbg="file drivers/pinctrl/pinmux.c +p"
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| الـ Option | الغرض |
|-----------|--------|
| `CONFIG_PINMUX` | الـ pinmux core نفسه — لازم يكون enabled |
| `CONFIG_DEBUG_FS` | يفعّل `/sys/kernel/debug/pinctrl/*` |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | دعم الـ `function_desc` + الـ radix tree |
| `CONFIG_PINCTRL` | الـ pinctrl core — prerequisite |
| `CONFIG_DEBUG_KERNEL` | يفعّل assertions إضافية |
| `CONFIG_DEBUG_SPINLOCK` | يكشف locking errors في `mux_lock` |
| `CONFIG_PROVE_LOCKING` | `lockdep` — يكشف deadlocks في الـ mutex |
| `CONFIG_LOCK_STAT` | إحصائيات الـ lock contention |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `dev_dbg()` runtime |
| `CONFIG_KALLSYMS` | أسماء الـ symbols في stack traces |
| `CONFIG_STACKTRACE` | مطلوب للـ lockdep وغيره |
| `CONFIG_KPROBES` | للـ kprobe tracing على `pin_request` |
| `CONFIG_FTRACE` | للـ function tracing |

**في `.config` أو `make menuconfig`:**

```bash
# تحقق من الـ options الحالية
grep -E 'CONFIG_(PINMUX|PINCTRL|DEBUG_FS|DYNAMIC_DEBUG)' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**libgpiod (أفضل من الـ sysfs القديم):**

```bash
# اعرض كل الـ GPIO chips وحالة الـ lines
gpiodetect
gpioinfo gpiochip0

# اقرأ قيمة GPIO (يطلبه ويحرره)
gpioget gpiochip0 5

# راقب الـ events على GPIO
gpiomon gpiochip0 5
```

**pinctrl tool (في بعض الـ distros):**

```bash
# من package devmem2 أو util-linux
# مباشرة للـ debugfs
find /sys/kernel/debug/pinctrl/ -type f | xargs -I{} sh -c 'echo "=== {} ==="; cat {}'
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `pinmux ops lacks necessary functions` | الـ driver ما implement الـ ops المطلوبة (`get_functions_count`, `get_function_name`, `get_function_groups`, `set_mux`) | تحقق من `struct pinmux_ops` في الـ driver |
| `pinmux ops has no name for function%u` | `get_function_name()` رجّعت NULL لـ function معين | بق في الـ driver — تحقق من الـ function table |
| `pin %d is not registered so it cannot be requested` | pin number غير موجود في `pin_desc` | تحقق من `npins` وحدود الـ `pinctrl_desc.pins` |
| `pin %s already requested by %s; cannot claim for %s` | pin conflict — اتنين طالبين نفس الـ pin | تحقق من الـ DT: هل نفس الـ pin في أكتر من `pinctrl-0` node؟ |
| `could not request pin %d (%s) from group %s on device %s` | `pin_request()` فشل لـ pin في group | ممكن conflict أو `ops->request()` رجّع error |
| `invalid function %s in map table` | اسم الـ function في الـ DT/board مش موجود في الـ driver | طابق أسماء functions في الـ DT مع `get_function_name()` |
| `invalid group "%s" for function "%s"` | الـ group المطلوب مش ضمن الـ groups التابعة للـ function | راجع `get_function_groups()` في الـ driver |
| `function %s can't be selected on any group` | `get_function_groups()` رجّعت 0 groups | بق في الـ driver — الـ function ما ليهاش groups |
| `does not support mux function` | `pmxops` = NULL — الـ controller مش بيدعم pinmux | تحقق من `pinctrl_desc.pmxops` |
| `could not increase module refcount for pin %d` | الـ module بدأ يتـunload أثناء request | race condition أو الـ module مش محمّل صح |
| `could not get pins for group %s` | `get_group_pins()` فشل | الـ group selector غلط أو بق في الـ pctlops |
| `not freeing pin %d (%s) as part of deactivating group %s` | الـ `mux_setting` pointer اتغير — مش نفس الـ setting اللي طلبت الـ pin | state inconsistency — ممكن double free أو race |
| `set_mux() failed: %d` | الـ hardware رفض الـ mux setting | تحقق من الـ hardware datasheet + الـ register values |

---

#### 8. نقاط استراتيجية لـ `dump_stack()` و `WARN_ON()`

**النقاط المهمة في الكود:**

```c
/* في pin_free() — موجودة أصلًا في الكود */
if (WARN_ON(!desc->mux_usecount))
    return NULL;
/* تكشف: محاولة free pin أكتر من ما اتطلب */

/* نقطة مقترحة في pin_request() لتتبع الـ ownership conflicts */
WARN_ON(desc->mux_usecount > 1 && strcmp(desc->mux_owner, owner));

/* في pinmux_enable_setting() بعد set_mux() */
if (ret) {
    dump_stack(); /* يوضح call chain كامل لمين طلب الـ setting */
}
```

**أضف `pr_debug` مؤقت في `pinmux_disable_setting()` لتتبع الـ is_equal check:**

```c
/* في pinmux_disable_setting() */
scoped_guard(mutex, &desc->mux_lock)
    is_equal = (desc->mux_setting == &(setting->data.mux));

/* نقطة مقترحة للـ debugging */
if (!is_equal) {
    dev_warn(pctldev->dev,
             "mux_setting mismatch on pin %d: expected %p got %p\n",
             pins[i], &setting->data.mux, desc->mux_setting);
    dump_stack(); /* يكشف من غيّر الـ mux_setting */
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

```bash
# 1. شوف الـ kernel يقول إيه عن الـ pin
cat /sys/kernel/debug/pinctrl/<dev>/pinmux-pins | grep "pin 5"
# Output: pin 5 (UART0_TX): uart0 (GPIO UNCLAIMED) function uart0 group uart0_grp

# 2. تحقق من الـ hardware register يطابق الـ expected mux value
# مثلًا على i.MX: IOMUXC_SW_MUX_CTL_PAD_UART0_TX في 0x020E0084
devmem2 0x020E0084 w
# Expected: 0x0 (ALT0 = UART) / 0x5 (ALT5 = GPIO)

# 3. تحقق من الـ GPIO direction يطابق
cat /sys/kernel/debug/gpio | grep "gpio-5"
```

**مخطط التحقق:**

```
Kernel (pinmux-pins)          Hardware Register
  function = uart0    <---->   MUX_CTL = 0x0 (ALT0)
  function = gpio     <---->   MUX_CTL = 0x5 (ALT5)
  UNCLAIMED           <---->   MUX_CTL = reset default
```

---

#### 2. تقنيات الـ Register Dump

**باستخدام `devmem2`:**

```bash
# اقرأ register واحد (32-bit)
devmem2 0x020E0084 w

# اكتب قيمة جديدة (للاختبار فقط — بدون الـ kernel!)
devmem2 0x020E0084 w 0x5

# loop لقراءة range من الـ registers
for addr in $(seq 0x020E0000 4 0x020E00FF); do
    printf "0x%08x: " $addr
    devmem2 $addr w 2>/dev/null | grep -oP '0x[0-9A-Fa-f]+'
done
```

**باستخدام `/dev/mem` مع Python:**

```python
import mmap, struct

with open('/dev/mem', 'r+b') as f:
    base = 0x020E0000  # IOMUXC base على i.MX6
    size = 0x1000
    m = mmap.mmap(f.fileno(), size, offset=base)
    for i in range(0, 64, 4):
        val = struct.unpack('<I', m[i:i+4])[0]
        print(f"0x{base+i:08x}: 0x{val:08x}")
    m.close()
```

**باستخدام `io` utility:**

```bash
# من package iotools أو busybox
io -4 -r 0x020E0084      # قراءة 32-bit
io -4 -w 0x020E0084 0x0  # كتابة
```

**على ARM64 مع MMIO:**

```bash
# تحقق إن /dev/mem مفعّل: CONFIG_DEVMEM=y + CONFIG_STRICT_DEVMEM=n
grep DEVMEM /boot/config-$(uname -r)
```

---

#### 3. نصائح الـ Logic Analyzer / Oscilloscope

**للتحقق من الـ pinmux setting على مستوى الـ hardware:**

```
السيناريو: UART0_TX (pin 5) مفروض يكون مُعرَّف كـ UART

الخطوات:
1. Connect probe to physical pin
2. في الـ kernel: echo "uart0_grp uart0" > .../pinmux-select
3. راقب الـ signal على الـ oscilloscope

Expected:
  ┌─┐ ┌──┐ ┌─┐
──┘ └─┘  └─┘ └──   (UART serial data)

إذا شفت:
  ─────────────    (static high/low) → الـ mux ما اتغيرش = hardware issue
  ─/─/─/─/─/─     (clock-like) → اتمسّك كـ GPIO output clock
```

**نقاط المراقبة:**
- راقب الـ pin أثناء `probe` للـ device (بعد `pinmux_enable_setting` مباشرة)
- راقب الـ glitch عند switching: بعض الـ controllers بيعمل momentary float
- استخدم `gpioset` مع GPIO function للتحقق من الـ direction

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة الـ Hardware | النمط في الـ Kernel Log | التشخيص |
|---------------------|------------------------|---------|
| الـ pin مش موصول أو مقطوع | `pin_request()` بينجح لكن الـ device مش شغّال | اقرأ الـ register يدوي بـ `devmem2` |
| الـ MUX register read-only (lock) | `set_mux()` فشل صامت أو رجّع success لكن ما اتغيرش | تحقق من الـ Security/Trustzone settings |
| Pull-up/Pull-down غلط | الـ GPIO input بيقرأ قيمة عشوائية | راجع `pinconf` settings + `pinctrl-imx` conf registers |
| الـ Power domain مش enabled | `pin_request()` ينجح لكن `set_mux()` يفشل بـ timeout | تفعيل الـ clock/power للـ IOMUXC أو الـ GPIO controller |
| الـ pin مستخدم من الـ bootloader | الـ kernel يلاقي الـ pin مأخود أو بـ مفاجأة | اعمل dump للـ registers قبل وبعد الـ kernel boot |
| Reset value غلط بعد الـ suspend/resume | الـ device بيشتغل صح بعد boot لكن يفشل بعد suspend | أضف `pinctrl_pm_select_default_state()` في الـ driver |

---

#### 5. الـ Device Tree Debugging

**تحقق إن الـ DT يطابق الـ hardware:**

```bash
# اعرض الـ compiled DT
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "pinctrl"

# أو الـ DT source
find /sys/firmware/devicetree/base -name "compatible" | xargs -I{} sh -c \
    'echo "{}:"; strings {}'

# تحقق من الـ pinctrl phandles
cat /sys/firmware/devicetree/base/soc/iomuxc/compatible 2>/dev/null
strings /sys/firmware/devicetree/base/soc/iomuxc/status 2>/dev/null
```

**تحقق من الـ pinctrl-0 mapping في الـ DT:**

```bash
# هل الـ device اتربط بالـ pinctrl state صح؟
cat /sys/kernel/debug/pinctrl/<dev>/pinctrl-maps | grep -A5 "device"
```

**مثال DT صح مقابل خاطئ:**

```dts
/* صح - uart0 عنده pinctrl-0 يشاور على group صح */
&uart0 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart0_pins>;
    status = "okay";
};

&iomuxc {
    uart0_pins: uart0-pins {
        fsl,pins = <
            MX6Q_PAD_UART0_TX__UART0_TX  0x1b0b1   /* الـ mux + الـ config */
            MX6Q_PAD_UART0_RX__UART0_RX  0x1b0b1
        >;
    };
};

/* خاطئ - pinctrl-0 بيشاور على group تانية */
&uart0 {
    pinctrl-0 = <&spi0_pins>;  /* Wrong group! */
};
```

**للتحقق إن الـ DT node اتحمل:**

```bash
# هل الـ pinctrl node enabled؟
cat /sys/bus/platform/devices/*/of_node/status 2>/dev/null

# هل الـ maps اتحملت؟
cat /sys/kernel/debug/pinctrl/*/pinctrl-maps 2>/dev/null | head -40
```

---

### Practical Commands

#### كل الأوامر جاهزة للنسخ

```bash
#!/bin/bash
# === pinmux-debug.sh ===
# شغّله كـ root

PCTLDEV=$(ls /sys/kernel/debug/pinctrl/ | grep -v "^pinctrl-" | head -1)
DBGBASE="/sys/kernel/debug/pinctrl/$PCTLDEV"

echo "=== [1] Pin Controller Device ==="
echo "Using: $PCTLDEV"

echo ""
echo "=== [2] All Registered Functions ==="
cat "$DBGBASE/pinmux-functions"

echo ""
echo "=== [3] All Pins Status ==="
cat "$DBGBASE/pinmux-pins"

echo ""
echo "=== [4] Active Pinctrl Handles ==="
cat "$DBGBASE/pinctrl-handles" 2>/dev/null

echo ""
echo "=== [5] Pinctrl Map Table ==="
cat "$DBGBASE/pinctrl-maps" 2>/dev/null

echo ""
echo "=== [6] GPIO Status ==="
cat /sys/kernel/debug/gpio 2>/dev/null

echo ""
echo "=== [7] Claimed Pins (non-UNCLAIMED) ==="
cat "$DBGBASE/pinmux-pins" | grep -v UNCLAIMED
```

```bash
# تفعيل الـ dynamic debug للـ pinmux core
echo 'file drivers/pinctrl/pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/core.c +p'   > /sys/kernel/debug/dynamic_debug/control

# مراقبة الـ kernel messages في real time
dmesg -w | grep -E 'pinmux|pinctrl|pin '
```

```bash
# Force mux setting (للاختبار)
PCTLDEV=$(ls /sys/kernel/debug/pinctrl/ | grep -v "^pinctrl-" | head -1)
echo "uart0_grp uart0" > /sys/kernel/debug/pinctrl/$PCTLDEV/pinmux-select
echo $?  # 0 = success
```

```bash
# تتبع pin_request باستخدام ftrace kprobe
echo 'p:my_pin_req pin_request pin=%si' > /sys/kernel/debug/tracing/kprobe_events
echo 1 > /sys/kernel/debug/tracing/events/kprobes/my_pin_req/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... افعل الـ operation المشبوهة ...
cat /sys/kernel/debug/tracing/trace | grep my_pin_req
# cleanup
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo '-:my_pin_req' > /sys/kernel/debug/tracing/kprobe_events
```

```bash
# تحقق من الـ mux register يدوي (مثال i.MX6)
IOMUXC_BASE=0x020E0000

# MUX control لـ UART1_TX_DATA (offset 0x84)
devmem2 $((IOMUXC_BASE + 0x84)) w
# 0x0 = ALT0 (UART)  |  0x5 = ALT5 (GPIO)

# اقرأ 16 register متتالي
for i in $(seq 0 4 60); do
    addr=$(printf "0x%08x" $((IOMUXC_BASE + i)))
    val=$(devmem2 $addr w 2>/dev/null | grep -oP '(?<=Read at address  0x[0-9a-f]+: )\S+')
    echo "$addr: $val"
done
```

```bash
# تحقق من الـ module refcount
cat /proc/modules | grep pinctrl
# الـ column الثالث هو عدد الـ users

# تحقق من الـ devices المرتبطة بالـ pinctrl
ls -la /sys/bus/platform/devices/*/driver | grep pinctrl
```

```bash
# مقارنة الـ DT مع الـ kernel state
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    awk '/pinctrl/{found=1} found{print; if(/};/){found=0}}' | head -60

# مقابل
cat /sys/kernel/debug/pinctrl/*/pinctrl-maps 2>/dev/null
```

#### تفسير الـ Output الشائع

```
# pinmux-pins output مع strict mode:
pin 5 (UART0_TX): device uart0 function uart0 group uart0_grp
#   ↑ pin number  ↑ pin name    ↑ mux_owner    ↑ function    ↑ group

# pinmux-pins output بدون strict mode:
pin 5 (UART0_TX): uart0 (GPIO UNCLAIMED) function uart0 group uart0_grp
#                  ↑mux_owner ↑gpio_owner=none

# pin conflict في dmesg:
# pinmux core: pin GPIO5 already requested by uart0; cannot claim for spi0
#   ↑ اتنين drivers بيطلبوا نفس الـ pin — راجع الـ DT
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — UART3 مش شغال بعد bring-up

#### العنوان
**الـ UART3 مش بيظهر في `/dev/` على industrial gateway بـ RK3562**

#### السياق
شركة بتبني industrial gateway بيربط sensors بـ Modbus RTU. الـ SoC هو **RK3562**، والـ UART3 المفروض يتوصل بـ RS-485 transceiver. الـ board جديدة، أول مرة بيتعمل لها bring-up.

#### المشكلة
الـ driver `serial8250` بيطلع في `dmesg` لكن `/dev/ttyS3` مش بيظهر. الـ application بتفشل لما بتحاول تفتح الـ port.

```bash
# dmesg بيدي:
[    2.341] pinctrl core: pin uart3-tx already requested by fe640000.uart; cannot claim for fe650000.uart
```

#### التحليل
الـ error جاي من `pin_request()` في `pinmux.c`:

```c
/* في pin_request() — السطر 152-158 */
if ((!gpio_range || ops->strict) && !gpio_ok &&
    desc->mux_usecount && strcmp(desc->mux_owner, owner)) {
    dev_err(pctldev->dev,
        "pin %s already requested by %s; cannot claim for %s\n",
        desc->name, desc->mux_owner, owner);
    goto out;
}
```

الـ `mux_usecount` للـ pin مش صفر — يعني pin اتطلب من device تانية. السبب: في الـ DT، فيه node تاني بيستخدم نفس الـ pin group. الـ `pinmux_enable_setting()` بتلف على كل الـ pins في الـ group وبتعمل `pin_request()` لكل واحدة — لو أي حاجة request نفس الـ pin قبل كده وأصحابها ما فرّغوش، الـ call بتفشل.

```c
/* في pinmux_enable_setting() — السطر 461-476 */
for (i = 0; i < num_pins; i++) {
    ret = pin_request(pctldev, pins[i], setting->dev_name, NULL);
    if (ret) {
        /* ... error handling ... */
        goto err_pin_request;
    }
}
```

المشكلة الفعلية: في الـ DT، نودين بيستخدموا نفس الـ pinmux group بالغلط:

```dts
/* غلط — نفس الـ pins في اتنين nodes */
&uart2 {
    pinctrl-0 = <&uart3m0_xfer>;  /* غلط! uart2 بياخد pins بتاعة uart3 */
    status = "okay";
};

&uart3 {
    pinctrl-0 = <&uart3m0_xfer>;
    status = "okay";
};
```

#### الحل
```bash
# الخطوة الأولى: اشوف مين بيمسك الـ pins
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinmux-pins | grep uart3
# output: pin 32 (uart3-tx): device fe640000.uart function uart2 group uart2m0-xfer

# الخطوة التانية: اشوف الـ DT المحمّل
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinmux-functions | grep uart
```

تصحيح الـ DT:
```dts
/* صح — كل uart بـ pins خاصة بيه */
&uart2 {
    pinctrl-0 = <&uart2m0_xfer>;  /* pins بتاعة uart2 صح */
    status = "okay";
};

&uart3 {
    pinctrl-0 = <&uart3m0_xfer>;  /* pins بتاعة uart3 صح */
    status = "okay";
};
```

#### الدرس المستفاد
الـ `pin_request()` بتحفظ `mux_owner` في `desc->mux_owner` وبتزود `mux_usecount`. لما يجي owner تاني يطلب نفس الـ pin، الكود بيعمل `strcmp(desc->mux_owner, owner)` — لو مختلفين، error فوري. دايما افحص `/sys/kernel/debug/pinctrl/` الأول قبل ما تبدأ تـ debug الـ driver نفسه.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — GPIO وـ SPI بيتعاركوا

#### العنوان
**الـ SPI flash مش بيشتغل وـ GPIO بيعمل conflict على Allwinner H616**

#### السياق
Android TV box بـ **Allwinner H616**. الـ SPI0 متوصل بـ NOR flash بيحمل bootloader backup. الـ engineer بيحاول يستخدم نفس الـ pins كـ GPIO للـ factory test mode — بيحط jumper ويشوف لو الـ pin low → يدخل test mode.

#### المشكلة
لما الـ SPI driver بيشتغل، محاولة تعمل `gpio_request()` على نفس الـ pin بتفشل بـ `-EBUSY`. لما بيعمل العكس ويطلب الـ GPIO الأول، الـ SPI بيفشل.

```bash
[    3.102] spi-sun6i 4025000.spi: could not request pin 68 (PC4) from group spi0-cs0 on device 300b000.pinctrl
[    3.103] pinctrl core: pin PC4 already requested by spi0; cannot claim for gpiochip0:68
```

#### التحليل
الـ H616 pinctrl driver بيستخدم `ops->strict = true`. لما `strict` شغال، الـ `pinmux_can_be_used_for_gpio()` بتتصرف مختلف:

```c
/* في pinmux_can_be_used_for_gpio() — السطر 105-108 */
if (ops->strict && desc->mux_usecount && !func_is_gpio)
    return false;   /* الـ pin متحجز لـ function مش GPIO */

return !(ops->strict && !!desc->gpio_owner);
```

وفي `pin_request()` لما الـ GPIO بيحاول يحجز:

```c
/* السطر 160-165 */
if ((gpio_range || ops->strict) && !gpio_ok && desc->gpio_owner) {
    dev_err(pctldev->dev,
        "pin %s already requested by %s; cannot claim for %s\n",
        desc->name, desc->gpio_owner, owner);
    goto out;
}
```

في الـ strict mode، مفيش sharing خالص. الـ SPI حجز الـ pin أول → الـ GPIO request بتفشل قطعي.

الـ `gpio_ok` بتتحدد من `ops->function_is_gpio`:
```c
/* السطر 144-150 */
if (mux_setting) {
    if (ops->function_is_gpio)
        gpio_ok = ops->function_is_gpio(pctldev, mux_setting->func);
} else {
    gpio_ok = true;  /* مفيش mux_setting = GPIO مقبول */
}
```

#### الحل
الـ solution الصح: استخدم SPI device نفسه للـ CS pin بدل GPIO مباشر، أو استخدم pin تاني للـ factory test:

```dts
/* استخدم pin منفصل للـ factory test */
factory_test: factory-test {
    compatible = "gpio-keys";
    test-button {
        gpios = <&pio 2 8 GPIO_ACTIVE_LOW>;  /* PC8 — pin منفصل */
        label = "factory-test";
    };
};
```

لو لازم تستخدم نفس الـ pin، استخدم pinctrl states:

```dts
&spi0 {
    pinctrl-names = "default", "gpio-mode";
    pinctrl-0 = <&spi0_pins>;       /* SPI function */
    pinctrl-1 = <&spi0_gpio_pins>;  /* نفس الـ pins كـ GPIO */
};
```

```bash
# للـ debug: شوف الـ strict mode وحالة الـ pins
cat /sys/kernel/debug/pinctrl/300b000.pinctrl/pinmux-pins
# Format: pin (name): mux_owner|gpio_owner (strict)
```

#### الدرس المستفاد
الـ `strict` flag في `pinmux_ops` بتغير كل الـ conflict resolution logic. في الـ strict controllers، الـ pin عنده owner واحد بس في أي وقت — إما mux function أو GPIO، مش الاتنين. لازم تعرف لو الـ SoC controller بتاعك strict ولا لأ قبل ما تصمم الـ hardware.

---

### السيناريو الثالث: IoT Sensor Board على STM32MP1 — Pinmux Driver مش بيـ probe

#### العنوان
**الـ STM32 pinctrl driver بيفشل في probe بـ `-EINVAL` والسبب مش واضح**

#### السياق
IoT sensor board بـ **STM32MP1**. الـ engineer بيكتب custom pinctrl driver لـ FPGA-based pin controller مربوط بالـ SoC. الـ driver بيـ probe بالـ DT لكن بيرجع `-EINVAL` فوراً.

#### المشكلة
```bash
[    1.823] pinctrl core: pinmux ops lacks necessary functions
[    1.824] stm32mp1-custom-pinctrl: probe failed: -22
```

#### التحليل
الـ error جاي من `pinmux_check_ops()` في أول ما الـ pinctrl device بيتسجل:

```c
/* في pinmux_check_ops() — السطر 43-50 */
if (!ops ||
    !ops->get_functions_count ||
    !ops->get_function_name ||
    !ops->get_function_groups ||
    !ops->set_mux) {
    dev_err(pctldev->dev, "pinmux ops lacks necessary functions\n");
    return -EINVAL;
}
```

الأربعة functions دول **إجبارية** — لو أي واحدة `NULL`، الـ registration بتفشل. الـ engineer نسي يـ implement `get_function_groups`:

```c
/* كود الـ driver الغلط */
static const struct pinmux_ops custom_pmxops = {
    .get_functions_count = custom_get_functions_count,
    .get_function_name   = custom_get_function_name,
    /* get_function_groups ناسي! */
    .set_mux             = custom_set_mux,
};
```

بعد كده، حتى لو الـ ops كاملة، في check تاني:

```c
/* السطر 52-62 */
nfuncs = ops->get_functions_count(pctldev);
while (selector < nfuncs) {
    const char *fname = ops->get_function_name(pctldev, selector);
    if (!fname) {
        dev_err(pctldev->dev,
            "pinmux ops has no name for function%u\n", selector);
        return -EINVAL;
    }
    selector++;
}
```

يعني حتى لو `get_function_name` موجودة، لو رجعت `NULL` لأي selector، الـ registration بتفشل برضو.

#### الحل
```c
/* كود صح */
static const char * const custom_uart_groups[] = { "uart0_grp" };
static const char * const custom_spi_groups[]  = { "spi0_grp", "spi1_grp" };

static const struct pinfunction custom_functions[] = {
    PINCTRL_PINFUNCTION("uart0", custom_uart_groups, ARRAY_SIZE(custom_uart_groups)),
    PINCTRL_PINFUNCTION("spi0",  custom_spi_groups,  ARRAY_SIZE(custom_spi_groups)),
};

static int custom_get_function_groups(struct pinctrl_dev *pctldev,
                                      unsigned int selector,
                                      const char * const **groups,
                                      unsigned int * const ngroups)
{
    /* لازم يـ return groups لكل function */
    *groups  = custom_functions[selector].groups;
    *ngroups = custom_functions[selector].ngroups;
    return 0;
}

static const struct pinmux_ops custom_pmxops = {
    .get_functions_count = custom_get_functions_count,
    .get_function_name   = custom_get_function_name,
    .get_function_groups = custom_get_function_groups,  /* الـ missing function */
    .set_mux             = custom_set_mux,
};
```

```bash
# للـ verification بعد الـ fix
cat /sys/kernel/debug/pinctrl/custom-pinctrl/pinmux-functions
# function 0: uart0, groups = [ uart0_grp ]
# function 1: spi0, groups = [ spi0_grp spi1_grp ]
```

#### الدرس المستفاد
الـ `pinmux_check_ops()` هي الـ gatekeeper الأولى — بتتأكد إن الأربعة ops الأساسية موجودة وإن كل function عنده اسم. لما بتكتب custom pinctrl driver، الأربعة دول **required**، مفيش optional فيهم. استخدم `CONFIG_GENERIC_PINMUX_FUNCTIONS` لو ممكن عشان `pinmux_generic_*` functions تعمل الشغل ده أوتوماتيك.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — Pinmux State Switching بيعمل Kernel Warning

#### العنوان
**`WARN_ON` في `pin_free()` أثناء runtime pinmux switching على i.MX8**

#### السياق
Automotive ECU بـ **i.MX8MP**. الـ system بيعمل dynamic switching لـ pinmux states — الـ CAN bus pins بيتحولوا لـ GPIO في الـ sleep mode عشان يقلل الـ current consumption. الـ switching بيحصل بـ `pinctrl_select_state()`.

#### المشكلة
```bash
[  142.331] WARNING: CPU: 2 PID: 847 at drivers/pinctrl/pinmux.c:252
[  142.331] pinmux: pin is not freed more times than allocated
[  142.331] Call trace:
[  142.331]  pin_free+0x7c/0x120
[  142.331]  pinmux_disable_setting+0x94/0x110
```

#### التحليل
الـ `WARN_ON` ده في `pin_free()`:

```c
/* في pin_free() — السطر 248-256 */
scoped_guard(mutex, &desc->mux_lock) {
    if (!gpio_range) {
        /*
         * A pin should not be freed more times than allocated.
         */
        if (WARN_ON(!desc->mux_usecount))
            return NULL;
        desc->mux_usecount--;
        if (desc->mux_usecount)
            return NULL;
    }
    /* ... */
}
```

الـ `mux_usecount` وصل صفر قبل الـ free. السبب: الـ `pinmux_disable_setting()` بتتحقق من `mux_setting`:

```c
/* في pinmux_disable_setting() — السطر 551-564 */
scoped_guard(mutex, &desc->mux_lock)
    is_equal = (desc->mux_setting == &(setting->data.mux));

if (is_equal) {
    pin_free(pctldev, pins[i], NULL);  /* بتعمل free بس لو الـ setting هي اللي حاطة الـ mux */
} else {
    dev_warn(pctldev->dev,
             "not freeing pin %d (%s) as part of deactivating group %s"
             " - it is already used for some other setting",
             pins[i], desc->name, gname);
}
```

المشكلة: الـ engineer عمل double-disable لنفس الـ setting بالغلط — call `pinctrl_select_state()` لـ sleep state مرتين متتاليتين بدون state tracking. الـ مرة الأولى عملت free صح، الـ مرة التانية لقت `mux_usecount = 0` → WARN_ON.

الـ flow المشكلة:
```
pinctrl_select_state(sleep) → pinmux_disable_setting() → pin_free() → mux_usecount: 1→0
pinctrl_select_state(sleep) → pinmux_disable_setting() → pin_free() → mux_usecount: 0 → WARN!
```

#### الحل
```c
/* في الـ ECU power management driver */
static int ecu_can_suspend(struct device *dev)
{
    struct ecu_priv *priv = dev_get_drvdata(dev);

    /* تأكد مش بتعمل switch لنفس الـ state */
    if (priv->current_pinctrl_state == PINCTRL_STATE_SLEEP)
        return 0;  /* already in sleep, skip */

    int ret = pinctrl_select_state(priv->pctl, priv->sleep_state);
    if (!ret)
        priv->current_pinctrl_state = PINCTRL_STATE_SLEEP;

    return ret;
}

static int ecu_can_resume(struct device *dev)
{
    struct ecu_priv *priv = dev_get_drvdata(dev);

    if (priv->current_pinctrl_state == PINCTRL_STATE_DEFAULT)
        return 0;

    int ret = pinctrl_select_state(priv->pctl, priv->default_state);
    if (!ret)
        priv->current_pinctrl_state = PINCTRL_STATE_DEFAULT;

    return ret;
}
```

```bash
# للـ monitoring أثناء الـ switching
watch -n 0.5 'cat /sys/kernel/debug/pinctrl/30330000.pinctrl/pinmux-pins | grep can'
```

#### الدرس المستفاد
الـ `mux_usecount` في `pin_desc` هو reference counter دقيق. `pinmux_enable_setting()` بتزوده، `pin_free()` بتنقصه. الـ WARN_ON عند `mux_usecount == 0` هو protection من double-free. في الـ runtime PM code، دايما track الـ current state عشان تمنع double-disable. الـ kernel مش بيحفظ "current state" بالنيابة عنك في الـ pinctrl layer.

---

### السيناريو الخامس: Custom Board على AM62x — debugfs لـ Manual Pinmux Override أثناء Bringup

#### العنوان
**استخدام `pinmux-select` debugfs لاختبار HDMI pins على AM62x بدون DT recompile**

#### السياق
Engineer بيعمل bring-up لـ custom display board بـ **TI AM62x**. الـ HDMI controller بيتشكى إن الـ I2C DDC pins مش شغالة. الـ DT معمول بشكل صح نظرياً، لكن الـ engineer عايز يـ verify إن الـ pinmux hardware بيستجاوب صح قبل ما يضيع وقت في الـ DT.

#### المشكلة
```bash
[    5.441] cdns-mhdp8546 imx8mq-hdmi: failed to read EDID
[    5.441] cdns-mhdp8546 imx8mq-hdmi: HPD lost
# الـ HDMI مش بيشوف الـ monitor
```

#### التحليل
الـ `pinmux-select` debugfs file بيسمح بـ manual override. الـ implementation في `pinmux_select_write()`:

```c
/* في pinmux_select_write() — السطر 715-788 */
static ssize_t pinmux_select_write(struct file *file, const char __user *user_buf,
                                   size_t len, loff_t *ppos)
{
    /* format: "group_name function_name" */
    buf = memdup_user_nul(user_buf, len);

    gname = strstrip(buf);          /* strip whitespace */
    /* find separator (space) between group and function */
    for (fname = gname; !isspace(*fname); fname++) { ... }
    *fname = '\0';
    fname = skip_spaces(fname + 1); /* skip extra spaces */

    /* lookup function selector */
    ret = pinmux_func_name_to_selector(pctldev, fname);
    fsel = ret;

    /* verify group belongs to this function */
    ret = pmxops->get_function_groups(pctldev, fsel, &groups, &num_groups);
    ret = match_string(groups, num_groups, gname);

    /* get group selector */
    gsel = pinctrl_get_group_selector(pctldev, gname);

    /* apply mux directly! */
    ret = pmxops->set_mux(pctldev, fsel, gsel);
}
```

الـ file موجود على path:
```
/sys/kernel/debug/pinctrl/<controller>/pinmux-select
```

#### الحل — الـ Debug Session الكاملة

```bash
# الخطوة 1: شوف الـ controller الموجود
ls /sys/kernel/debug/pinctrl/
# 901000.pinctrl  pinctrl-handles  pinctrl-maps

# الخطوة 2: شوف الـ functions المتاحة
cat /sys/kernel/debug/pinctrl/901000.pinctrl/pinmux-functions
# function 0: gpio0, groups = [ gpio0-0 gpio0-1 ... ]
# function 15: i2c5, groups = [ i2c5-pins ]
# function 23: hdmi-ddc, groups = [ hdmi-ddc-pins ]

# الخطوة 3: شوف الحالة الحالية للـ pins
cat /sys/kernel/debug/pinctrl/901000.pinctrl/pinmux-pins | grep -E "PIN_H|hdmi|i2c5"
# pin 42 (PIN_H3): UNCLAIMED
# pin 43 (PIN_H4): UNCLAIMED

# الخطوة 4: الـ pins UNCLAIMED يعني الـ DT مش بيوصّل صح
# جرب manual override
echo "hdmi-ddc-pins hdmi-ddc" > /sys/kernel/debug/pinctrl/901000.pinctrl/pinmux-select

# الخطوة 5: verify
cat /sys/kernel/debug/pinctrl/901000.pinctrl/pinmux-pins | grep "PIN_H"
# pin 42 (PIN_H3): device hdmi function hdmi-ddc group hdmi-ddc-pins
# pin 43 (PIN_H4): device hdmi function hdmi-ddc group hdmi-ddc-pins

# الخطوة 6: جرب الـ HDMI تاني
i2cdetect -y 5  # لو ظهر device على 0x50 → الـ DDC شغال → المشكلة في الـ DT
```

لما الـ manual override نجح، الـ engineer اكتشف إن الـ DT كان فيه typo في اسم الـ pinmux group:

```dts
/* غلط */
&hdmi0 {
    pinctrl-0 = <&hdmi_ddc_pin>;    /* underscore */
};

/* والـ pinctrl node اسمه */
hdmi-ddc-pins: hdmi-ddc-pins {     /* dash */
    pinmux = <AM62X_IOPAD(...)>;
};
```

```c
/* pinmux_map_to_setting() السطر 404-413 — لما الاسم مش مطابق */
if (map->data.mux.group) {
    group = map->data.mux.group;
    ret = match_string(groups, num_groups, group);
    if (ret < 0) {
        dev_err(pctldev->dev,
            "invalid group \"%s\" for function \"%s\"\n",
            group, map->data.mux.function);
        return ret;
    }
}
```

الـ `match_string()` بتعمل exact match — underscore vs dash = failure.

```dts
/* الحل: توحيد الأسماء */
hdmi_ddc_pins: hdmi-ddc-pins {
    pinmux = <AM62X_IOPAD(0x01d0, PIN_INPUT, 0)>;  /* SCL */
    pinmux = <AM62X_IOPAD(0x01d4, PIN_INPUT, 0)>;  /* SDA */
};

&hdmi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&hdmi_ddc_pins>;  /* label reference — مش string matching */
};
```

#### الدرس المستفاد
الـ `pinmux-select` debugfs file هي أداة bring-up قوية جداً — بتسمحلك تـ apply أي pinmux function على أي group مباشرة بدون recompile. الـ `pinmux_select_write()` بتعمل الـ exact same path زي الـ normal kernel path بس بدون الـ DT validation. لو الـ manual override نجح وإنما الـ DT فشل، المشكلة دايماً في الـ string matching بين الـ map table وأسماء الـ groups/functions. ابدأ بـ debugfs دايماً في الـ bring-up قبل ما تتعمق في الـ DT.
## Phase 7: مصادر ومراجع

### توثيق Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| [`Documentation/driver-api/pin-control.rst`](https://docs.kernel.org/driver-api/pin-control.html) | التوثيق الرسمي والشامل لـ pinctrl subsystem — يغطي pinmux و pinconf و GPIO integration |
| [`Documentation/pinctrl.txt`](https://www.kernel.org/doc/Documentation/pinctrl.txt) | النسخة الأقدم من التوثيق — مفيدة لفهم التصميم الأصلي |
| [`Documentation/devicetree/bindings/pinctrl/pinmux-node.yaml`](https://mjmwired.net/kernel/Documentation/devicetree/bindings/pinctrl/pinmux-node.yaml) | Device Tree bindings الخاصة بـ pinmux nodes |

**الـ** `Documentation/driver-api/pin-control.html` هو المرجع الأساسي — بيشرح كل concept من `pinmux_ops` لـ `set_mux()` لـ GPIO integration بتفصيل كامل.

---

### مقالات LWN.net

دي أهم المقالات اللي تتابعها بالترتيب:

#### 1. The pin control subsystem
**الرابط:** [https://lwn.net/Articles/468759/](https://lwn.net/Articles/468759/)

أهم مقال — كتبه Linus Walleij نفسه. بيشرح:
- ليه احتاجوا pinctrl بدل الـ GPIO-only approach
- العلاقة بين functions و groups و pins
- إزاي `pinmux_ops` اتصمم

#### 2. Documentation/pinctrl.txt — Early Patch Discussion
**الرابط:** [https://lwn.net/Articles/465077/](https://lwn.net/Articles/465077/)

نقاش الـ mailing list حول التوثيق الأول للـ subsystem — مفيد لفهم القرارات التصميمية الأولى.

#### 3. pinctrl: add a generic pin config interface
**الرابط:** [https://lwn.net/Articles/468770/](https://lwn.net/Articles/468770/)

بيشرح إضافة pinconf — مكمل لـ pinmux في الـ subsystem.

#### 4. pinctrl: add a pin config interface
**الرابط:** [https://lwn.net/Articles/471826/](https://lwn.net/Articles/471826/)

تطور الـ pinconf API — مهم لفهم الفرق بين mux و config.

#### 5. pinctrl: introduce the concept of a GPIO pin function category
**الرابط:** [https://lwn.net/Articles/1031226/](https://lwn.net/Articles/1031226/)

أحدث مقال — بيشرح `PINFUNCTION_FLAG_GPIO` وإزاي `pinmux_generic_function_is_gpio()` اتضافت للـ subsystem. مهم لفهم الكود الحديث في `pinmux.c`.

#### 6. pin controller subsystem v7
**الرابط:** [https://lwn.net/Articles/459190/](https://lwn.net/Articles/459190/)

المراجعة السابعة للـ subsystem قبل merge — بيكشف التطور التصميمي.

---

### نقاشات Mailing List

#### GPIO pin function category — Patch Series
**الرابط:** [https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10546.html](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10546.html)

نقاش patch series اللي أضافت مفهوم `PINFUNCTION_FLAG_GPIO` — مهم لفهم `pinmux_generic_function_is_gpio()`.

#### keembay: use a dedicated structure for the pinfunction description
**الرابط:** [https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10492.html](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10492.html)

بيشرح migration لـ `struct pinfunction` الجديد — الـ struct اللي `pinmux_generic_add_pinfunction()` بتستخدمه.

#### imx: use generic pinmux helpers
**الرابط:** [https://lore.kernel.org/patchwork/patch/746887/](https://lore.kernel.org/patchwork/patch/746887/)

مثال عملي على migration لـ `CONFIG_GENERIC_PINMUX_FUNCTIONS` — الـ pattern اللي معظم الـ drivers بتتبعه دلوقتي.

#### pinctrl-single: use generic pinmux helpers
**الرابط:** [https://patchwork.kernel.org/patch/9395509/](https://patchwork.kernel.org/patch/9395509/)

migration تاني مهم — `pinctrl-single` هو من أكتر الـ drivers استخداماً.

#### Linus Walleij on rockchip pinctrl fix
**الرابط:** [https://lkml.org/lkml/2026/2/24/668](https://lkml.org/lkml/2026/2/24/668)

نقاش حديث من Walleij نفسه — maintainer الـ subsystem.

---

### Source Code على GitHub

| الملف | الرابط |
|-------|--------|
| `drivers/pinctrl/pinmux.c` | [torvalds/linux — pinmux.c](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/pinmux.c) |
| `drivers/pinctrl/pinmux.h` | [torvalds/linux — pinmux.h](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/pinmux.h) |
| `drivers/pinctrl/Kconfig` | [torvalds/linux — Kconfig](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/Kconfig) |
| `linux-pinctrl tree (Walleij)` | [kernel.googlesource.com](https://kernel.googlesource.com/pub/scm/linux/kernel/git/linusw/linux-pinctrl/) |

---

### eLinux.org

| الصفحة | الوصف |
|--------|-------|
| [ELC-2021: Introduction to pin muxing and GPIO control under Linux](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) | Presentation من ELC 2021 — شرح عملي ممتاز لـ pinmux و GPIO تحت Linux |
| [EBC Device Trees](https://elinux.org/EBC_Device_Trees) | أمثلة عملية على pinctrl configuration في device tree overlays |
| [Jetson AGX Xavier: Check Pin Setting](https://www.elinux.org/Jetson/AGX_Xavier_Check_Pin_Setting) | يشرح كيفية استخدام `tegra_pinctrl_reg` debugfs interface للتحقق من mux registers |
| [Tests: i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) | اختبار dynamic pinmux switching مع i2c — use case متقدم |

---

### kernelnewbies.org

بيغطي التغييرات في الـ subsystem per-release:

| الإصدار | الرابط | أبرز التغييرات |
|---------|--------|----------------|
| Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) | freescale i.MX91, nuvoton ma35d1 pinctrl |
| Linux 6.13 | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) | pinctrl-zynqmp Versal, Canaan K230 |
| Linux 6.17 | [kernelnewbies.org/Linux_6.17](https://kernelnewbies.org/Linux_6.17) | Eswin eic7700, sm7635 pinctrl |
| Linux 6.8 | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) | Qualcomm و Samsung SoCs |
| Linux 4.8 | [kernelnewbies.org/Linux_4.8](https://kernelnewbies.org/Linux_4.8) | تغييرات مبكرة في generic pinmux helpers |

---

### كتب مرجعية

#### Linux Device Drivers (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم:** Chapter 14 — The Linux Device Model
- **ملاحظة:** الكتاب قديم (kernel 2.6) لكن الـ concepts الأساسية لا تزال صالحة. الـ pinctrl subsystem نفسه جاء بعد إصدار الكتاب.
- **رابط مجاني:** [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love
- **الطبعة:** الثالثة (تغطي kernel 2.6.34)
- **الفصول المهمة:**
  - Chapter 13: The Virtual Filesystem — لفهم debugfs اللي `pinmux_init_device_debugfs()` بتستخدمه
  - Chapter 14: The Block I/O Layer — لفهم نموذج الـ device model
- **ملاحظة:** الـ pinctrl جاء بعد الكتاب لكن فهم الـ kernel internals ضروري

#### Embedded Linux Primer — Christopher Hallinan
- **الطبعة:** الثانية
- **الفصل المهم:** Chapter 15: Embedded Linux Projects — Platform Devices and Device Tree
- **ملاحظة:** بيغطي device tree integration وهو الـ mechanism الأساسي لتكوين pinctrl على embedded platforms

#### Linux Device Driver Development — John Madieu
- **الناشر:** Packt Publishing (2022)
- **الفصل:** Chapter 12: Leveraging the Pinctrl Subsystem
- **ده أحسن مرجع حديث** يغطي pinctrl/pinmux بشكل مباشر مع أمثلة كود

---

### مواقع إضافية

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| embedded.com | [Linux device driver development: The pin control subsystem](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) | شرح عملي من منظور driver developer |
| STM32 Wiki | [Pinctrl overview](https://wiki.st.com/stm32mpu/wiki/Pinctrl_overview) | مثال حقيقي على STM32 — من أفضل التوثيقات الخاصة بـ vendor |
| krybot.com | [Introduction to the Pinctrl subsystem](https://blog.krybot.com/a?ID=01600-cb8bc57c-f987-4202-9b8d-0635739c052d) | ملخص سريع ومركز |
| infradead.org | [PINCTRL documentation (5.10-rc1+)](https://www.infradead.org/~mchehab/kernel_docs/driver-api/pinctl.html) | نسخة من التوثيق الرسمي |

---

### Search Terms للبحث عن معلومات أكتر

```
# للبحث في LKML و lore.kernel.org
pinctrl pinmux subsystem linux kernel
pinmux_ops set_mux linux kernel
pinctrl generic functions radix tree
pinmux_enable_setting linux
pinmux_request_gpio kernel
CONFIG_GENERIC_PINMUX_FUNCTIONS
PINFUNCTION_FLAG_GPIO
pinmux strict mode linux kernel
pin_request pin_free pinctrl core

# للبحث عن drivers مثال
pinctrl-single driver example
pinctrl imx generic pinmux
qualcomm pinctrl pinmux driver
rockchip pinctrl mux implementation

# للبحث عن debugging
debugfs pinmux-functions linux
debugfs pinmux-pins
pinmux-select debugfs linux kernel

# للبحث في Device Tree
devicetree pinctrl pinmux-names
pinctrl-0 pinctrl-names device tree
pinmux-node yaml binding
```

---

### مسار التعلم المقترح

```
1. اقرأ LWN Articles/468759 (The pin control subsystem) أولاً
   ↓
2. اقرأ Documentation/driver-api/pin-control.html كاملاً
   ↓
3. اتفرج على ELC-2021 PDF من elinux.org
   ↓
4. اتعلم من مثال حقيقي: STM32 Pinctrl overview
   ↓
5. اقرأ كود driver بسيط: drivers/pinctrl/pinctrl-single.c
   ↓
6. ارجع للـ source files: pinmux.c و pinmux.h
```
## Phase 8: Writing simple module

### الفكرة العامة

هنعمل module بيـhook الـ`pinmux_enable_setting` عن طريق **kprobe**، ده من أكتر الـentry points إثارةً في الـpinmux subsystem — بيتنادى كل مرة بيتفعل فيها mux setting على أي pin controller في النظام، يعني هنشوف كل مرة الـkernel بيحول function لـpin group.

---

### الـModule الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe on pinmux_enable_setting()
 *
 * Fires every time a pinmux setting is activated on any
 * pin controller device — prints the device name + function/group indices.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/pinctrl/pinctrl.h>   /* struct pinctrl_dev, pinctrl_dev_get_name() */
#include <linux/pinctrl/machine.h>   /* struct pinctrl_setting */

/*
 * Forward-declare only what we need from the internal setting struct.
 * pinctrl_setting is defined in <linux/pinctrl/machine.h> but the
 * mux sub-struct fields are accessible via the public header.
 */

/* ------------------------------------------------------------------ */
/*  kprobe pre-handler — runs just BEFORE pinmux_enable_setting()      */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument arrives in RDI (regs->di).
     * On ARM64 it is in x0 (regs->regs[0]).
     * kprobes gives us pt_regs — we cast the first arg to the
     * pointer type we expect.
     */
#ifdef CONFIG_X86_64
    const struct pinctrl_setting *setting =
        (const struct pinctrl_setting *)regs->di;
#elif defined(CONFIG_ARM64)
    const struct pinctrl_setting *setting =
        (const struct pinctrl_setting *)regs->regs[0];
#else
    /* Generic fallback — may not work on every arch */
    const struct pinctrl_setting *setting = NULL;
#endif

    if (!setting || !setting->pctldev)
        return 0;   /* skip if pointer looks bad */

    /*
     * pinctrl_dev_get_name() is safe to call here — it just returns
     * pctldev->desc->name without taking any lock.
     */
    pr_info("pinmux_enable_setting called: dev=\"%s\" func_selector=%u group_selector=%u dev_name=\"%s\"\n",
            pinctrl_dev_get_name(setting->pctldev),
            setting->data.mux.func,
            setting->data.mux.group,
            setting->dev_name ? setting->dev_name : "(none)");

    return 0;   /* 0 = continue normal execution */
}

/* ------------------------------------------------------------------ */
/*  kprobe post-handler — runs just AFTER the probed instruction       */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /* We only need pre — leave this empty but defined so kprobes is happy */
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "pinmux_enable_setting",  /* symbol to hook */
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ------------------------------------------------------------------ */
/*  module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init pinmux_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe registered on pinmux_enable_setting @ %p\n",
            kp.addr);   /* kp.addr filled by kernel after registration */
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit pinmux_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe unregistered from pinmux_enable_setting\n");
}

module_init(pinmux_probe_init);
module_exit(pinmux_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe demo: trace pinmux_enable_setting() calls");
```

---

### شرح كل جزء

#### الـ`#include`s

| Header | السبب |
|--------|-------|
| `<linux/kprobes.h>` | تعريف `struct kprobe` وكل الـAPI بتاعه — `register_kprobe`, `unregister_kprobe`. |
| `<linux/pinctrl/pinctrl.h>` | محتاجينه عشان `pinctrl_dev_get_name()` و`struct pinctrl_dev`. |
| `<linux/pinctrl/machine.h>` | فيه تعريف `struct pinctrl_setting` وداخله `data.mux.func` و`data.mux.group`. |

---

#### الـ`handler_pre`

ده القلب — بيشتغل **قبل** تنفيذ أول instruction في `pinmux_enable_setting`. الـkprobe بيديك `pt_regs` اللي فيها قيم الـregisters وقت الـcall، فبنسحب الـargument الأول (الـ`setting` pointer) من الـregister المناسب حسب الـarchitecture.

بعد كده بنطبع:
- اسم الـpin controller device (`dev`).
- رقم الـfunction selector المطلوب تفعيله.
- رقم الـgroup selector.
- اسم الـdevice اللي طلب الـsetting ده (`dev_name`، ممكن يكون اسم driver زي `"spi0"`).

ده مفيد جداً في الـdebugging — تقدر تشوف إيه الـfunctions اللي بتتفعل وبترتيب إيه أثناء الـboot أو لما device بيـprobe.

---

#### الـ`struct kprobe kp`

الـ**`symbol_name`** بيخلي الـkernel يحل الـaddress تلقائياً من الـkallsyms، من غير ما تحتاج تعرف الـaddress يدوياً. الـ`pre_handler` و`post_handler` بيتسجلوا هنا كـpointers.

---

#### الـ`module_init` — `register_kprobe`

**الـ`register_kprobe`** بيعمل كذا حاجة في خطوة واحدة: بيحل الـsymbol لـaddress، بيعمل **breakpoint** (INT3 على x86، BRK على ARM64) عند أول instruction في الـfunction، وبيسجل الـhandlers. لو رجع قيمة سالبة يبقى فيه مشكلة — الـsymbol مش موجود أو الـkprobes مش enabled في الـkernel.

---

#### الـ`module_exit` — `unregister_kprobe`

**لازم** تعمل unregister قبل ما الـmodule يـunload — لو متعملش، الـbreakpoint هيفضل موجود في الـcode لكن الـhandler pointers هتبقى invalid (الـmodule اتفك من الـmemory)، وده هيعمل **kernel panic** أول ما `pinmux_enable_setting` تتنادى. الـ`unregister_kprobe` بيشيل الـbreakpoint ويستنى لحد ما أي handler شغال حالياً يخلص قبل ما يرجع.

---

### طريقة البناء والتشغيل

```bash
# Makefile بسيط
# obj-m += pinmux_kprobe.o

make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

sudo insmod pinmux_kprobe.ko

# شوف الـoutput
sudo dmesg | grep "pinmux_enable_setting"

sudo rmmod pinmux_kprobe
```

**مثال عملي للـoutput** أثناء الـboot أو لما device بيـprobe:

```
[   2.341]  pinmux_enable_setting called: dev="pinctrl-bcm2835" func_selector=3 group_selector=7 dev_name="fe204000.spi"
[   2.342]  pinmux_enable_setting called: dev="pinctrl-bcm2835" func_selector=1 group_selector=2 dev_name="fe201000.uart"
```

يعني الـRaspberry Pi مثلاً بيحول الـpins بتاعة SPI وUART لـfunctions معينة — ودي بالظبط المعلومة اللي `pinmux_enable_setting` بتطبقها.

---

### ملاحظة على الـarchitecture

الـ`#ifdef CONFIG_X86_64 / CONFIG_ARM64` ده ضروري لأن قواعد الـcalling convention مختلفة — على x86-64 الـargument الأول في `rdi`، على ARM64 في `x0`. الـkprobes نفسها portable لكن سحب الـarguments من `pt_regs` معتمد على الـarch.
