## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بيتبعه الملف

الملف `clk-gpio.c` جزء من **Common Clock Framework (CCF)** — الـ subsystem المسؤول عن إدارة كل الـ clocks جوه الـ Linux kernel. المسؤولون عنه: Michael Turquette و Stephen Boyd، والكود كله تحت `drivers/clk/`.

---

### الفكرة ببساطة — قصة قبل الكود

تخيل عندك جهاز embedded — مثلاً Raspberry Pi أو بورد TI — وفيه chip بتولد clock signal (إشارة ساعة) لجزء من الـ hardware، يعني بتقوله "اشتغل" أو "وقف". المشكلة: أحياناً الـ chip دي مش بتتحكم فيها بـ register جوه الـ SoC زي العادة، لكن بتتحكم فيها بـ **GPIO pin** — سلك رقمي بسيط يطلع 0 أو 1.

تخيل الموقف كده:
- عندك crystal oscillator (مصدر clock ثابت) على البورد.
- الـ oscillator ده عنده pin اسمه `OE` (Output Enable).
- لما تحط على الـ `OE` pin قيمة `HIGH` → الـ clock بيشتغل.
- لما تحط `LOW` → الـ clock بيوقف.

ده بالظبط اللي بيعمله `clk-gpio.c`: بيربط الـ **clock framework** بالـ **GPIO subsystem** عشان يقدر يتحكم في الـ clock بمجرد رفع أو خفض GPIO pin.

---

### ليه الملف ده مهم؟

الـ Linux kernel عنده نظام موحد لإدارة الـ clocks اسمه **CCF**. كل جهاز بيطلب clock من الـ framework وبيقوله "enable" أو "disable" من غير ما يعرف إزاي الـ hardware بتتحكم في الـ clock دي. الـ CCF بيستخدم **struct clk_ops** — مجموعة function pointers — عشان كل نوع clock يقدر يوفر implementation خاصة بيه.

**clk-gpio.c** بيوفر implementation لثلاث أنواع:

| النوع | الاسم في الكود | الوصف |
|---|---|---|
| **GPIO Gate** | `clk_gpio_gate_ops` | clock يتفتح ويتقفل بـ GPIO |
| **GPIO Sleeping Gate** | `clk_sleeping_gpio_gate_ops` | نفسه لكن الـ GPIO ممكن يـ sleep (مثلاً عبر I2C) |
| **GPIO Mux** | `clk_gpio_mux_ops` | اختيار بين مصدرين للـ clock عن طريق GPIO |
| **Gated Fixed Clock** | `clk_gated_fixed_ops` | clock ثابت التردد مع GPIO للـ enable وـ regulator للطاقة |

---

### القصة الكاملة: prepare vs enable

الـ CCF عنده مفهوم مهم: الـ clock له مرحلتين قبل ما تشتغل:

1. **prepare** — التحضير: ممكن يـ sleep (مثلاً تشغيل regulator أو التحدث مع I2C GPIO expander). بيتحكم فيه `prepare_mutex`.
2. **enable** — التشغيل الفعلي: لازم يكون atomic وما يـ sleep-ش. بيتحكم فيه `spinlock`.

لذلك الملف فيه نسختين:
- **Non-sleeping GPIO**: يتحكم فيها في الـ `enable` step (سريع، atomic).
- **Sleeping GPIO** (مثلاً GPIO على I2C expander): يتحكم فيها في الـ `prepare` step (مسموح ينام).

---

### الـ gated-fixed-clock: القصة الأكمل

في السيناريو الأكثر تعقيداً (`struct clk_gated_fixed`)، الـ clock ده:
1. مصدره خارجي وثابت التردد (مثلاً crystal بيشتغل على 24 MHz).
2. محتاج **regulator** (وحدة طاقة) يتشغل الأول قبل الـ clock نفسه.
3. وبعدين **GPIO** يتفعّل عشان يبدأ الـ clock.

الترتيب يكون:
```
prepare → enable regulator → enable GPIO (if sleeping)
enable  → toggle GPIO (if non-sleeping)
```

ده مثال واقعي على clock في embedded board بيحتاج power rail قبل ما يبدأ يشتغل.

---

### الـ Device Tree: كيف بيتعرف الـ driver على الـ clock؟

الـ driver بيستخدم `of_device_id` عشان يتعرف على نوعه من الـ Device Tree:

```c
static const struct of_device_id gpio_clk_match_table[] = {
    { .compatible = "gpio-mux-clock" },   // clock مع اختيار بين مصدرين
    { .compatible = "gpio-gate-clock" },  // clock يتفتح ويتقفل
    { }
};
```

مثال من Device Tree:

```dts
// gpio-gate-clock: clock بيتحكم فيه GPIO
clock {
    compatible = "gpio-gate-clock";
    clocks = <&parentclk>;
    #clock-cells = <0>;
    enable-gpios = <&gpio 1 GPIO_ACTIVE_HIGH>;
};

// gpio-mux-clock: اختيار بين مصدرين
clock {
    compatible = "gpio-mux-clock";
    clocks = <&parentclk1>, <&parentclk2>;
    #clock-cells = <0>;
    select-gpios = <&gpio 1 GPIO_ACTIVE_HIGH>;
};
```

---

### ASCII Diagram: مكانة الملف في النظام

```
 ┌─────────────────────────────────────┐
 │        Consumer Driver              │
 │  (e.g., USB, I2C, display driver)  │
 └────────────────┬────────────────────┘
                  │  clk_enable() / clk_disable()
                  ▼
 ┌─────────────────────────────────────┐
 │     Common Clock Framework (CCF)    │
 │          drivers/clk/clk.c          │
 └────────────────┬────────────────────┘
                  │  clk_ops->enable / disable / ...
                  ▼
 ┌─────────────────────────────────────┐
 │         clk-gpio.c                  │
 │  (GPIO-controlled clock driver)     │
 │  clk_gpio_gate_ops                  │
 │  clk_gpio_mux_ops                   │
 │  clk_gated_fixed_ops                │
 └────────────────┬────────────────────┘
                  │  gpiod_set_value()
                  ▼
 ┌─────────────────────────────────────┐
 │       GPIO Subsystem                │
 │    (gpiolib / GPIO chip driver)     │
 └─────────────────────────────────────┘
                  │
                  ▼
          [Physical GPIO Pin]
                  │
                  ▼
    [Clock chip OE/Enable pin]
```

---

### الملفات المهمة اللي المبرمج يعرفها

#### الـ Core Files (قلب الـ CCF):
| الملف | الدور |
|---|---|
| `drivers/clk/clk.c` | الـ core الرئيسي للـ CCF — يدير كل الـ clocks |
| `drivers/clk/clk-gpio.c` | **الملف نفسه** — GPIO-controlled clocks |
| `drivers/clk/clk-fixed-rate.c` | clock بتردد ثابت بدون أي تحكم |
| `drivers/clk/clk-gate.c` | gate clock عام بـ register bit |
| `drivers/clk/clk-mux.c` | mux clock عام بـ register |
| `drivers/clk/clk-divider.c` | clock divider بـ register |

#### الـ Headers المهمة:
| الملف | الدور |
|---|---|
| `include/linux/clk-provider.h` | تعريف `struct clk_ops`، `struct clk_hw`، `clk_init_data` |
| `include/linux/gpio/consumer.h` | API للتعامل مع الـ GPIO descriptors |
| `include/linux/clk.h` | الـ consumer API (clk_enable, clk_disable) |

#### الـ Device Tree Bindings:
| الملف | الدور |
|---|---|
| `Documentation/devicetree/bindings/clock/gpio-gate-clock.yaml` | schema لـ `gpio-gate-clock` |
| `Documentation/devicetree/bindings/clock/gpio-mux-clock.yaml` | schema لـ `gpio-mux-clock` |

#### ملفات مرتبطة في الـ GPIO Subsystem:
| الملف | الدور |
|---|---|
| `drivers/gpio/gpiolib.c` | الـ core للـ GPIO subsystem |
| `include/linux/gpio/consumer.h` | descriptor-based GPIO API |
## Phase 2: شرح الـ Common Clock Framework (CCF)

### المشكلة اللي الـ CCF بيحلها

قبل الـ CCF (قبل kernel 3.4)، كل platform بيعمل clock management بطريقته الخاصة. كل SoC vendor بيكتب كود من الصفر لإدارة الـ clocks — enabling، disabling، rate-setting — من غير أي abstraction مشترك. النتيجة:

- **كود متكرر** في كل platform (OMAP, i.MX, Exynos, Rockchip...).
- **مفيش sharing** للـ logic المشتركة زي reference counting أو clock tree traversal.
- **الـ consumers** (drivers اللي بتستخدم clocks) بيتكلموا مع كل platform بطريقة مختلفة.
- **مفيش standard** لـ clock tree representation أو dependency tracking.

**الـ CCF** جه يعمل abstraction layer موحد بين الـ hardware-specific clock implementations وبين الـ consumers.

---

### الحل — الـ Common Clock Framework

الـ CCF بيقسم الموضوع لجزئين واضحين:

| الجزء | المسؤولية |
|-------|-----------|
| **Consumer side** (`linux/clk.h`) | الـ API اللي بيستخدمه drivers: `clk_get`, `clk_enable`, `clk_set_rate`, إلخ |
| **Provider side** (`linux/clk-provider.h`) | الـ API اللي بيستخدمه clock drivers لتسجيل الـ clocks: `clk_hw_register`, `struct clk_ops`, إلخ |

الـ CCF بيبني **clock tree** في الـ kernel — كل clock عنده parent (أو أكتر في حالة mux)، والـ rate بتتحسب بالـ traversal من الـ root للـ leaf.

---

### الـ Real-World Analogy — شبكة توزيع الكهرباء

تخيل شبكة كهرباء في مدينة:

```
محطة توليد (Crystal Oscillator / PLL)
        |
    محول رئيسي (Clock Divider)
        |
   ┌────┴────┐
 حي A     حي B
 (Clock   (Clock
  Gate)    Mux)
   |         |────────────┐
 شقة 1    مصنع        مستشفى
(consumer) (consumer)  (consumer)
```

| عنصر الكهرباء | مقابله في الـ CCF |
|--------------|-----------------|
| محطة التوليد | crystal oscillator / PLL (root clock) |
| المحول الرئيسي | clock divider (`clk_divider`) |
| مفتاح الحي | clock gate (`clk_gate`) |
| اختيار مصدر (محطة A أو B) | clock mux (`clk_mux`) |
| العداد في كل شقة | reference count في الـ `clk_core` |
| شركة الكهرباء (مكتب مركزي) | الـ CCF core في `drivers/clk/clk.c` |
| كود كل محطة توليد | الـ hardware-specific clock driver |
| الجهاز الكهربائي | الـ consumer driver |

**المهم في الـ analogy**: لما شقة تطلب كهرباء (`clk_enable`)، شركة الكهرباء (CCF) بتتأكد إن المفتاح (gate) مفتوح، والمحول (divider) ضابط على الجهد الصح — الشقة مش مضطرة تعرف أي حاجة عن البنية التحتية. وكمان الشركة بتعمل reference counting — لو شقتين بيستخدموا نفس الحي، مش هتقفل المفتاح غير لما الاتنين يطلبوا التوقف.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Consumer Drivers                         │
│     (USB driver, UART driver, GPU driver, I2C driver, ...)      │
│                                                                 │
│   clk_get()  clk_enable()  clk_set_rate()  clk_set_parent()    │
└──────────────────────────┬──────────────────────────────────────┘
                           │  linux/clk.h  (Consumer API)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   CCF Core  (drivers/clk/clk.c)                 │
│                                                                 │
│  ┌──────────────┐  ┌────────────────┐  ┌─────────────────────┐ │
│  │  Clock Tree  │  │ Ref Counting   │  │  Rate Propagation   │ │
│  │  (radix tree │  │ prepare_count  │  │  (recalc_rate up    │ │
│  │  of clk_core)│  │ enable_count   │  │   the tree)         │ │
│  └──────────────┘  └────────────────┘  └─────────────────────┘ │
│                                                                 │
│  struct clk_core  ←→  struct clk_hw  ←→  struct clk_ops        │
└──────────────────────────┬──────────────────────────────────────┘
                           │  linux/clk-provider.h  (Provider API)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│               Hardware-Specific Clock Drivers                   │
│                                                                 │
│  ┌───────────────┐  ┌──────────────┐  ┌────────────────────┐   │
│  │ clk-gpio.c    │  │ clk-pll.c    │  │ clk-divider.c      │   │
│  │ (GPIO gate /  │  │ (PLL clock)  │  │ (clock divider)    │   │
│  │  GPIO mux /   │  │              │  │                    │   │
│  │  gated-fixed) │  │              │  │                    │   │
│  └───────┬───────┘  └──────┬───────┘  └────────┬───────────┘   │
└──────────┼─────────────────┼──────────────────-┼───────────────┘
           │                 │                   │
           ▼                 ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Real Hardware                            │
│                                                                 │
│    GPIO pin            PLL registers         Divider registers  │
│  (enable/select)       in SoC                in SoC             │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ CCF بيبني كل حاجة حوالين **ثلاث structs** مترابطين:

```
struct clk_hw  ─────► struct clk_init_data ─────► struct clk_ops
     │                      │                           │
  (الـ handle              (metadata:               (vtable:
  اللي بيربط               name, parents,          enable/disable/
  الـ provider             flags, num_parents)      set_rate/etc.)
  بالـ CCF core)
```

**الـ `struct clk_hw`** هو "نقطة التلاقي" — بيقعد جوه الـ hardware-specific struct وبيكون الـ bridge للـ CCF core.

```
struct clk_gpio {
    struct clk_hw  hw;      ◄── embedded clk_hw (أول field)
    struct gpio_desc *gpiod;◄── hardware-specific data
};
```

**الـ `struct clk_core`** (internal في CCF) هو الـ node في الـ clock tree — بيحتفظ بالـ reference counts وبيدير الـ parent/child relationships.

---

### العلاقة بين الـ Structs — رسم تفصيلي

```
                 ┌─────────────────────────────────────┐
                 │         struct clk_gpio              │
                 │  ┌──────────────────────────────┐   │
                 │  │       struct clk_hw           │   │
                 │  │   ┌───────────────────────┐  │   │
                 │  │   │  *core ──────────────────────────► struct clk_core
                 │  │   │  *clk  ──────────────────────────► struct clk (per-user)
                 │  │   │  *init ──────────────────────────► struct clk_init_data
                 │  │   └───────────────────────┘  │   │         │
                 │  └──────────────────────────────┘   │         │
                 │  *gpiod ──────────────────────────────────►  GPIO hardware
                 └─────────────────────────────────────┘    ┌───────────────┐
                                                             │ struct clk_   │
                                                             │ init_data     │
                                                             │  *name        │
                                                             │  *ops ────────────► struct clk_ops
                                                             │  *parent_data │         .enable
                                                             │  num_parents  │         .disable
                                                             │  flags        │         .is_enabled
                                                             └───────────────┘         .prepare
                                                                                       .unprepare
                                                                                       .recalc_rate
                                                                                       ...
```

**الـ `container_of` macro** هو السحر اللي بيخلي الكل يشتغل:

```c
#define to_clk_gpio(_hw) container_of(_hw, struct clk_gpio, hw)
```

الـ CCF بيبعت `struct clk_hw *hw` للـ callbacks — والـ driver بيستخدم `container_of` يرجع للـ `clk_gpio` الأصلي اللي فيه الـ `gpiod`.

---

### الـ prepare/enable Split — مفهوم مهم جداً

الـ CCF بيفصل بين مرحلتين لتشغيل الـ clock:

| المرحلة | الـ function | السياق | ليه؟ |
|---------|-------------|--------|-----|
| **prepare** | `clk_prepare()` → `.prepare` | يقدر ينام (sleepable) | للعمليات اللي محتاجة وقت زي enable regulator أو GPIO على I2C bus |
| **enable** | `clk_enable()` → `.enable` | atomic (no sleep) | للـ toggle السريع زي كتابة register أو GPIO مباشرة |

ده بيظهر صريح في الـ `clk-gpio.c`:

```c
/* GPIO عادي (fast, non-sleeping): يستخدم enable/disable */
static const struct clk_ops clk_gpio_gate_ops = {
    .enable     = clk_gpio_gate_enable,   /* gpiod_set_value() - atomic */
    .disable    = clk_gpio_gate_disable,
    .is_enabled = clk_gpio_gate_is_enabled,
};

/* GPIO بطيء (sleeping, e.g. on I2C GPIO expander): يستخدم prepare/unprepare */
static const struct clk_ops clk_sleeping_gpio_gate_ops = {
    .prepare    = clk_sleeping_gpio_gate_prepare,   /* gpiod_set_value_cansleep() */
    .unprepare  = clk_sleeping_gpio_gate_unprepare,
    .is_prepared = clk_sleeping_gpio_gate_is_prepared,
};
```

الـ driver بيختار الـ ops في runtime حسب `gpiod_cansleep()`:

```c
if (gpiod_cansleep(gpiod))
    ops = &clk_sleeping_gpio_gate_ops;
else
    ops = &clk_gpio_gate_ops;
```

---

### الـ Clock Types في الملف

الملف بيقدم **ثلاث أنواع** مختلفة من الـ clocks:

#### 1. GPIO Gate Clock (`gpio-gate-clock`)

```
parent_clock ──[GPIO=1: pass / GPIO=0: block]──► output
```

- يوروث الـ rate من الـ parent.
- GPIO high = clock enabled, GPIO low = clock disabled.
- يُستخدم في external oscillators أو clock buffers اللي بتتحكم فيها بـ GPIO.

**مثال حقيقي**: TI AM335x له external HDMI clock chip (SiI9022) بيتحكم في enable بـ GPIO.

#### 2. GPIO Mux Clock (`gpio-mux-clock`)

```
parent_0 ──┐
            ├──[GPIO=0: select parent_0 / GPIO=1: select parent_1]──► output
parent_1 ──┘
```

- بيختار بين اتنين sources.
- يُستخدم لما عندك مثلاً crystal oscillator وPLL وعايز تختار بينهم بـ GPIO.

```c
static u8 clk_gpio_mux_get_parent(struct clk_hw *hw)
{
    return gpiod_get_value_cansleep(clk->gpiod); /* 0 or 1 */
}

static int clk_gpio_mux_set_parent(struct clk_hw *hw, u8 index)
{
    gpiod_set_value_cansleep(clk->gpiod, index); /* set GPIO to index */
    return 0;
}
```

#### 3. Gated Fixed Clock (`gated-fixed-clock`)

أعقدهم — root clock (مفيش parent) بـ:
- **rate ثابت** محدد في device tree.
- **regulator** لإدارة الـ power supply (prepare phase).
- **GPIO** لتشغيل/إيقاف الـ clock output (enable phase أو prepare phase حسب نوع الـ GPIO).

```
Regulator ──[prepare: enable power]──► Clock Oscillator ──[GPIO enable]──► output
```

**struct hierarchy**:
```c
struct clk_gated_fixed {
    struct clk_gpio clk_gpio;   /* يحتوي على clk_hw + gpio_desc */
    struct regulator *supply;   /* power supply */
    unsigned long rate;         /* fixed frequency */
};
```

يعني `clk_gated_fixed` يحتوي على `clk_gpio` اللي بدوره يحتوي على `clk_hw`:

```
struct clk_gated_fixed
    └── struct clk_gpio
            ├── struct clk_hw  ◄── CCF entry point
            └── *gpiod
    ├── *supply (regulator)
    └── rate (e.g. 26000000 Hz)
```

---

### الـ Device Tree Integration

الـ driver بيتسجل كـ platform_driver مع `of_match_table`:

```c
static const struct of_device_id gpio_clk_match_table[] = {
    { .compatible = "gpio-mux-clock" },
    { .compatible = "gpio-gate-clock" },
    { }
};
```

مثال device tree node لـ GPIO gate clock:

```dts
clock_enable: clock-enable {
    compatible = "gpio-gate-clock";
    clocks = <&parent_clk>;      /* parent clock */
    enable-gpios = <&gpio1 5 0>; /* GPIO to control */
    #clock-cells = <0>;
};
```

ومثال لـ gated-fixed-clock:

```dts
wifi_clk: wifi-clock {
    compatible = "gated-fixed-clock";
    clock-frequency = <26000000>;    /* 26 MHz */
    clock-output-names = "wifi_ref";
    enable-gpios = <&gpio3 10 0>;
    vdd-supply = <&wifi_regulator>;
    #clock-cells = <0>;
};
```

---

### الـ CCF Registration Flow

```
gpio_clk_driver_probe()
        │
        ├── of_device_is_compatible() ──► is_mux?
        │
        ├── of_clk_get_parent_count() ──► num_parents
        │
        ├── devm_gpiod_get() ──► gpiod
        │
        ├── clk_hw_register_gpio_gate() or clk_hw_register_gpio_mux()
        │        │
        │        └── clk_register_gpio()
        │                 │
        │                 ├── devm_kzalloc(clk_gpio)
        │                 ├── init.name = node->name
        │                 ├── init.ops = clk_gpio_ops
        │                 ├── init.parent_data = gpio_parent_data
        │                 └── devm_clk_hw_register() ──► CCF يضيف الـ clock للـ tree
        │
        └── devm_of_clk_add_hw_provider() ──► يسمح للـ DT consumers يعملوا lookup
```

الـ `devm_clk_hw_register()` بيسجل الـ `clk_hw` في الـ CCF core اللي بيعمل:
1. ينشئ `struct clk_core` ويربطه بالـ `clk_hw`.
2. يضيف الـ clock للـ global clock tree.
3. يحسب الـ initial rate عن طريق `recalc_rate` (أو يورثها من الـ parent).

---

### ملكية الـ CCF مقابل مسؤولية الـ Driver

| الموضوع | مين يتولاه؟ |
|--------|------------|
| Reference counting (prepare_count, enable_count) | **CCF core** |
| Clock tree traversal وإيجاد الـ parents | **CCF core** |
| Rate caching والـ propagation لأعلى الشجرة | **CCF core** |
| debugfs entries للـ clock tree | **CCF core** |
| تشغيل/إيقاف الـ GPIO الفعلي | **Driver** (`clk_gpio_gate_enable`) |
| اختيار sleeping vs non-sleeping path | **Driver** (`gpiod_cansleep`) |
| enable الـ regulator | **Driver** (`clk_gated_fixed_prepare`) |
| حساب الـ fixed rate | **Driver** (`clk_gated_fixed_recalc_rate`) |
| تفسير الـ parent index لـ mux | **Driver** (`clk_gpio_mux_set_parent`) |

---

### الـ CLK_SET_RATE_PARENT Flag

```c
init.flags = CLK_SET_RATE_PARENT;
```

ده بيقول للـ CCF: "لما حد يطلب يغير الـ rate بتاعي، مرر الطلب لأبويا" — لأن الـ gpio gate/mux مش بيعمل rate conversion، الـ rate بتيجي من الـ parent مباشرة.

---

### الـ Subsystems التانية المرتبطة

- **GPIO Subsystem** (`gpiolib`): بيوفر `struct gpio_desc` والـ API اللي بتستخدمه الـ clk-gpio.c (`gpiod_set_value`, `gpiod_cansleep`). الـ gpio_desc هو abstraction فوق الـ hardware GPIO controller.
- **Regulator Framework**: بيوفر `struct regulator` وبيتحكم في power supply. في `clk_gated_fixed`، الـ regulator بيتفعّل في الـ prepare phase قبل ما الـ GPIO يتشغّل.
- **Device Tree / OF Framework**: بيوفر `of_clk_get_parent_count()`, `devm_of_clk_add_hw_provider()` — بيربط الـ DT nodes بالـ clock objects.
- **Platform Driver Model**: الـ driver بيتسجل كـ `platform_driver` وبيتطلع على `of_match_table` لمطابقة الـ compatible strings.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### CLK Flags (من `clk-provider.h`) — المستخدم في الملف

| Flag | القيمة | الوظيفة |
|------|--------|---------|
| `CLK_SET_RATE_PARENT` | `BIT(2)` | لما يتغير الـ rate، يتنقل التغيير للـ parent. مستخدم في `clk_register_gpio()` |
| `CLK_SET_RATE_GATE` | `BIT(0)` | لازم الـ clock يتوقف وقت تغيير الـ rate (مش مستخدم هنا لكن مهم معرفته) |
| `CLK_IS_CRITICAL` | `BIT(11)` | مينفعش الـ clock يتوقف أبداً |
| `CLK_IGNORE_UNUSED` | `BIT(3)` | متوقفش الـ clock حتى لو محدش بيستخدمه |

#### enum `gpiod_flags` (من `gpio/consumer.h`)

| القيمة | المعنى | الاستخدام في الملف |
|--------|--------|-------------------|
| `GPIOD_ASIS` | متغيّرش حالة الـ GPIO | — |
| `GPIOD_IN` | input mode | — |
| `GPIOD_OUT_LOW` | output, قيمة ابتدائية 0 | `devm_gpiod_get(..., GPIOD_OUT_LOW)` — كل الـ clocks هنا |
| `GPIOD_OUT_HIGH` | output, قيمة ابتدائية 1 | — |
| `GPIOD_OUT_LOW_OPEN_DRAIN` | open-drain, قيمة ابتدائية 0 | — |

#### الـ Compatible Strings (الـ DT matching)

| Compatible | نوع الـ Driver | الـ probe function |
|------------|---------------|-------------------|
| `"gpio-gate-clock"` | gate (enable/disable بـ GPIO) | `gpio_clk_driver_probe` |
| `"gpio-mux-clock"` | mux (اختيار parent بـ GPIO) | `gpio_clk_driver_probe` |
| `"gated-fixed-clock"` | fixed rate + gate (GPIO + regulator) | `clk_gated_fixed_probe` |

---

### الـ Structs المهمة

#### 1. `struct clk_gpio`

**الغرض:** الـ struct الأساسي اللي بيمثل أي clock بيتحكم فيه GPIO — سواء gate أو mux.

```c
struct clk_gpio {
    struct clk_hw   hw;      // entry point للـ CCF — لازم يكون أول field
    struct gpio_desc *gpiod; // الـ GPIO اللي بيتحكم في الـ clock
};
```

| Field | النوع | الدور |
|-------|-------|-------|
| `hw` | `struct clk_hw` | الـ handle اللي بيربط الـ CCF بالـ hardware implementation — يحتوي على `core`, `clk`, `init` |
| `gpiod` | `struct gpio_desc *` | descriptor للـ GPIO line — بيتجيب من `devm_gpiod_get()` |

**الماكرو المساعد:**
```c
#define to_clk_gpio(_hw) container_of(_hw, struct clk_gpio, hw)
// يحوّل pointer على clk_hw لـ pointer على clk_gpio
```

**الاتصال بالـ structs التانية:**
- `clk_hw.init` بيشاور على `struct clk_init_data` وقت التسجيل (بيتعمل null بعد `devm_clk_hw_register`)
- `clk_hw.core` بيشاور على `struct clk_core` (الـ internal CCF representation)
- `gpiod` بيشاور على `struct gpio_desc` (managed by GPIO subsystem)

---

#### 2. `struct clk_gated_fixed`

**الغرض:** امتداد لـ `clk_gpio` — clock بـ rate ثابت + gate بـ GPIO + power control بـ regulator.

```c
struct clk_gated_fixed {
    struct clk_gpio  clk_gpio; // يحتوي على hw + gpiod — لازم أول field
    struct regulator *supply;  // الـ power supply (vdd) — ممكن يبقى NULL
    unsigned long    rate;     // الـ rate الثابت (من "clock-frequency" في DT)
};
```

| Field | النوع | الدور |
|-------|-------|-------|
| `clk_gpio` | `struct clk_gpio` | يورّث الـ hw + gpiod — التضمين بيخلي `container_of` يشتغل |
| `supply` | `struct regulator *` | الـ power regulator — `NULL` لو مفيش |
| `rate` | `unsigned long` | الـ fixed frequency |

**الماكرو المساعد:**
```c
#define to_clk_gated_fixed(_clk_gpio) \
    container_of(_clk_gpio, struct clk_gated_fixed, clk_gpio)
// استخدام: to_clk_gated_fixed(to_clk_gpio(hw))
// سلسلة: clk_hw → clk_gpio → clk_gated_fixed
```

**الاتصال بالـ structs التانية:**
- يحتوي على `clk_gpio` كـ embedded struct (مش pointer)
- `supply` بيشاور على `struct regulator` (managed by regulator framework)

---

#### 3. `struct clk_hw` (من `clk-provider.h`)

**الغرض:** الـ bridge بين الـ hardware-specific struct والـ Common Clock Framework.

```c
struct clk_hw {
    struct clk_core         *core; // الـ internal CCF representation (opaque)
    struct clk              *clk;  // الـ per-user handle للـ consumers
    const struct clk_init_data *init; // init data — بيتعمل NULL بعد التسجيل
};
```

---

#### 4. `struct clk_init_data` (من `clk-provider.h`)

**الغرض:** البيانات الابتدائية اللي بتتحدد مرة واحدة وقت التسجيل.

```c
struct clk_init_data {
    const char              *name;        // اسم الـ clock
    const struct clk_ops    *ops;         // الـ callbacks
    const struct clk_parent_data *parent_data; // بيانات الـ parents
    u8                       num_parents; // عدد الـ parents
    unsigned long            flags;       // CLK_SET_RATE_PARENT وغيره
};
```

في `clk_register_gpio()`:
```c
init.flags = CLK_SET_RATE_PARENT; // يمرر تغيير الـ rate للـ parent
init.num_parents = num_parents;   // 0 للـ gate، 2 للـ mux
```

---

#### 5. `struct clk_ops` (من `clk-provider.h`)

**الغرض:** الـ vtable — مجموعة الـ callbacks اللي الـ CCF بيستدعيها.

الملف بيعرّف **4 نسخ** من الـ ops:

| الـ ops struct | الـ callbacks المعرّفة | متى يُستخدم |
|----------------|----------------------|-------------|
| `clk_gpio_gate_ops` | `enable`, `disable`, `is_enabled` | GPIO مش sleeping (fast) |
| `clk_sleeping_gpio_gate_ops` | `prepare`, `unprepare`, `is_prepared` | GPIO ممكن ينام (slow) |
| `clk_gpio_mux_ops` | `get_parent`, `set_parent`, `determine_rate` | GPIO mux دايماً |
| `clk_gated_fixed_ops` | `prepare`, `unprepare`, `is_prepared`, `enable`, `disable`, `is_enabled`, `recalc_rate` | fixed clock + non-sleeping GPIO |
| `clk_sleeping_gated_fixed_ops` | `prepare`, `unprepare`, `is_prepared`, `recalc_rate` | fixed clock + sleeping GPIO |

---

#### 6. `struct clk_parent_data` (من `clk-provider.h`)

```c
struct clk_parent_data {
    const struct clk_hw *hw;      // pointer مباشر للـ parent (internal)
    const char          *fw_name; // اسم في الـ firmware
    const char          *name;    // اسم global
    int                  index;   // index في الـ DT clocks property
};
```

في `clk_register_gpio()`:
```c
const struct clk_parent_data gpio_parent_data[] = {
    { .index = 0 }, // parent الأول من الـ DT
    { .index = 1 }, // parent التاني من الـ DT
};
```

---

### رسومات علاقات الـ Structs (ASCII)

#### البنية العامة للـ Inheritance

```
                    ┌─────────────────────────────┐
                    │       struct clk_hw          │
                    │  ─────────────────────────   │
                    │  core  → struct clk_core *   │
                    │  clk   → struct clk *        │
                    │  init  → clk_init_data *     │
                    └─────────────┬───────────────┘
                                  │ embedded (أول field)
                    ┌─────────────▼───────────────┐
                    │      struct clk_gpio         │
                    │  ─────────────────────────   │
                    │  hw    : struct clk_hw       │◄── to_clk_gpio(_hw)
                    │  gpiod → struct gpio_desc *  │
                    └─────────────┬───────────────┘
                                  │ embedded (أول field)
                    ┌─────────────▼───────────────┐
                    │   struct clk_gated_fixed     │
                    │  ─────────────────────────   │
                    │  clk_gpio : struct clk_gpio  │◄── to_clk_gated_fixed(_cg)
                    │  supply   → struct regulator │
                    │  rate     : unsigned long    │
                    └─────────────────────────────┘
```

#### علاقة الـ clk_hw بالـ CCF

```
  [driver code]
       │
       │  &clk_gpio->hw
       ▼
  struct clk_hw ◄────────────────────────────────────────┐
       │                                                  │
       │ .core                                            │ .hw
       ▼                                                  │
  struct clk_core ────────────────────────► struct clk_ops
       │                   .ops                    (vtable)
       │ .hw
       └──────────────────────────────────────────────────►
                            [back-reference]

  struct clk ────────────► struct clk_core
  (per-consumer handle)         .core
```

#### علاقة الـ clk_ops مع الـ GPIO states

```
clk_gpio_gate_ops:
  .enable      →  gpiod_set_value(gpiod, 1)   [non-sleeping context, IRQ safe]
  .disable     →  gpiod_set_value(gpiod, 0)   [non-sleeping context, IRQ safe]
  .is_enabled  →  gpiod_get_value(gpiod)       [non-sleeping context]

clk_sleeping_gpio_gate_ops:
  .prepare     →  gpiod_set_value_cansleep(gpiod, 1)  [sleeping context OK]
  .unprepare   →  gpiod_set_value_cansleep(gpiod, 0)
  .is_prepared →  gpiod_get_value_cansleep(gpiod)

clk_gpio_mux_ops:
  .get_parent  →  gpiod_get_value_cansleep(gpiod)     [0 أو 1]
  .set_parent  →  gpiod_set_value_cansleep(gpiod, idx)
  .determine_rate → __clk_mux_determine_rate()         [CCF helper]
```

---

### دورة الحياة (Lifecycle Diagrams)

#### دورة حياة `gpio-gate-clock` و `gpio-mux-clock`

```
[kernel boot / device probe]
          │
          ▼
  gpio_clk_driver_probe(pdev)
          │
          ├─► of_device_is_compatible() ──► is_mux = true/false
          │
          ├─► of_clk_get_parent_count()  ──► num_parents
          │
          ├─► devm_gpiod_get()           ──► struct gpio_desc *gpiod
          │         GPIOD_OUT_LOW (initial state = clock disabled)
          │
          ├─► clk_hw_register_gpio_gate()  أو  clk_hw_register_gpio_mux()
          │         │
          │         ├─► gpiod_cansleep() ──► اختيار sleeping/non-sleeping ops
          │         │
          │         └─► clk_register_gpio()
          │                   │
          │                   ├─► devm_kzalloc(clk_gpio)
          │                   ├─► init.name    = node->name
          │                   ├─► init.ops     = clk_gpio_ops
          │                   ├─► init.flags   = CLK_SET_RATE_PARENT
          │                   ├─► clk_gpio->hw.init = &init
          │                   └─► devm_clk_hw_register()
          │                             │
          │                             └─► CCF يسجّل الـ clock
          │                                 init pointer يتعمل NULL
          │
          └─► devm_of_clk_add_hw_provider()
                    │
                    └─► DT consumers يقدروا يطلبوا الـ clock بالاسم

[device remove — devm cleanup]
          │
          ├─► devm_of_clk_add_hw_provider cleanup ──► يُلغى الـ provider
          ├─► devm_clk_hw_register cleanup          ──► يُلغى التسجيل
          └─► devm_gpiod_get cleanup                ──► يُحرر الـ GPIO
```

#### دورة حياة `gated-fixed-clock`

```
[kernel boot / device probe]
          │
          ▼
  clk_gated_fixed_probe(pdev)
          │
          ├─► devm_kzalloc(clk_gated_fixed)
          │
          ├─► device_property_read_u32("clock-frequency") ──► clk->rate
          │
          ├─► device_property_read_string("clock-output-names") ──► clk_name
          │         (fallback: fwnode_get_name)
          │
          ├─► devm_regulator_get_optional("vdd") ──► clk->supply
          │         (NULL لو -ENODEV — الـ regulator اختياري)
          │
          ├─► devm_gpiod_get_optional("enable", GPIOD_OUT_LOW) ──► gpiod
          │
          ├─► gpiod_cansleep() ──► اختيار ops
          │         sleeping  → clk_sleeping_gated_fixed_ops
          │         non-sleep → clk_gated_fixed_ops
          │
          ├─► CLK_HW_INIT_NO_PARENT(clk_name, ops, 0)
          │         ──► no parent, no flags
          │
          ├─► devm_clk_hw_register(&clk->clk_gpio.hw)
          │
          └─► devm_of_clk_add_hw_provider()
```

#### دورة حياة enable/disable للـ consumer

```
Consumer calls clk_prepare_enable(clk):
          │
          ├─► clk_prepare()
          │       └─► ops->prepare(hw)
          │               [sleeping context — prepare_lock (mutex)]
          │               مثال: regulator_enable() + gpiod_set_value_cansleep(1)
          │
          └─► clk_enable()
                  └─► ops->enable(hw)
                          [atomic context — enable_lock (spinlock)]
                          مثال: gpiod_set_value(1)

Consumer calls clk_disable_unprepare(clk):
          │
          ├─► clk_disable()
          │       └─► ops->disable(hw)  [spinlock held]
          │               gpiod_set_value(0)
          │
          └─► clk_unprepare()
                  └─► ops->unprepare(hw)  [mutex held]
                          regulator_disable() + gpiod_set_value_cansleep(0)
```

---

### Call Flow Diagrams

#### تدفق `clk_enable` للـ gpio-gate-clock (non-sleeping)

```
consumer:  clk_enable(clk)
              │
              ▼
   CCF core:  clk_core_enable()
              │  [enable_lock spinlock acquired]
              │  checks enable_count, propagates to parent first
              ▼
   clk_gpio_gate_ops.enable(hw)
              │
              ▼
   clk_gpio_gate_enable(hw):
              │  to_clk_gpio(hw) → struct clk_gpio *
              ▼
   gpiod_set_value(clk->gpiod, 1)
              │
              ▼
   GPIO subsystem → hardware register write → GPIO pin HIGH
              │
              ▼
   Clock signal passes through gate ✓
```

#### تدفق `clk_set_parent` للـ gpio-mux-clock

```
consumer:  clk_set_parent(clk, new_parent)
              │
              ▼
   CCF core:  clk_core_set_parent_nolock()
              │  [prepare_lock mutex acquired]
              │  validates new parent is valid
              ▼
   clk_gpio_mux_ops.set_parent(hw, index)
              │
              ▼
   clk_gpio_mux_set_parent(hw, index):
              │  index = 0 → parent[0] selected
              │  index = 1 → parent[1] selected
              ▼
   gpiod_set_value_cansleep(clk->gpiod, index)
              │
              ▼
   GPIO pin = 0 or 1 → MUX hardware switches parent clock
```

#### تدفق الـ probe للـ gated-fixed-clock

```
platform bus detects "gated-fixed-clock" compatible:
              │
              ▼
   clk_gated_fixed_probe(pdev)
              │
              ├─[1]─► DT read: clock-frequency → rate
              │
              ├─[2]─► DT read: clock-output-names → clk_name
              │
              ├─[3]─► regulator framework: devm_regulator_get_optional("vdd")
              │           ─► struct regulator * (or NULL)
              │
              ├─[4]─► GPIO framework: devm_gpiod_get_optional("enable")
              │           ─► struct gpio_desc *
              │           ─► gpiod_cansleep() decides ops variant
              │
              ├─[5]─► CCF: devm_clk_hw_register()
              │           ─► clk_hw.init consumed → set NULL
              │           ─► clock added to CCF tree
              │
              └─[6]─► DT provider: devm_of_clk_add_hw_provider()
                          ─► consumers can now get this clock via DT
```

#### تدفق `clk_prepare` للـ sleeping gated fixed clock

```
consumer:  clk_prepare(clk)
              │
              ▼
   clk_sleeping_gated_fixed_ops.prepare(hw)
              │
              ▼
   clk_sleeping_gated_fixed_prepare(hw):
              │
              ├─[step 1]─► clk_gated_fixed_prepare(hw)
              │                │
              │                └─► regulator_enable(clk->supply)
              │                         [may sleep — mutex context OK]
              │
              └─[step 2]─► clk_sleeping_gpio_gate_prepare(hw)
                               │
                               └─► gpiod_set_value_cansleep(gpiod, 1)
                                        [may sleep — GPIO asserted]
                                        [if step 2 fails → step 1 rolled back]
```

---

### استراتيجية الـ Locking

الـ CCF بيستخدم نظام قفلين:

#### جدول الـ Locks

| القفل | النوع | يحمي إيه | السياق |
|-------|-------|-----------|--------|
| `prepare_lock` | `mutex` (يقدر ينام) | `prepare` / `unprepare` / `set_parent` / `set_rate` | process context فقط |
| `enable_lock` | `spinlock` | `enable` / `disable` / `is_enabled` | atomic context (IRQ safe) |

#### قاعدة الـ locking في الملف ده

- **`clk_gpio_gate_enable/disable`** → بيستخدموا `gpiod_set_value()` (non-sleeping) — آمن وقت الـ `enable_lock` spinlock.
- **`clk_sleeping_gpio_gate_prepare/unprepare`** → بيستخدموا `gpiod_set_value_cansleep()` (ممكن ينام) — آمن بس في `prepare_lock` mutex context.
- **`clk_gpio_mux_*`** → كلها `cansleep` لأن الـ mux switching ممكن يحتاج وقت.
- **`clk_gated_fixed_*`** → `regulator_enable/disable` ممكن تنام → لازم في `prepare` مش `enable`.

#### ترتيب الـ Locking (Lock Ordering)

```
prepare_lock (mutex)  ──بيتأخذ أول──►  enable_lock (spinlock)

clk_prepare_enable():
   acquire prepare_lock
      ops->prepare()        [GPIO/regulator — may sleep]
      acquire enable_lock
         ops->enable()      [GPIO only — no sleep]
      release enable_lock
   release prepare_lock
```

**مهم:** مينفعش تاخد `enable_lock` وبعدين تحاول تاخد `prepare_lock` — ده deadlock.

#### الـ sleeping vs non-sleeping decision

```
[at probe time]
gpiod_cansleep(gpiod) ?
      │
      ├── YES → ops with _cansleep variants → used in prepare context
      │             clk_sleeping_gpio_gate_ops
      │             clk_sleeping_gated_fixed_ops
      │
      └── NO  → ops with regular variants → safe in enable context
                    clk_gpio_gate_ops
                    clk_gated_fixed_ops
```

الـ `gpiod_cansleep()` بيرجع `true` لو الـ GPIO controller على bus بطيء (I2C, SPI) — في الحالة دي مينفعش تستخدمه في IRQ context أو atomic context.
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ Functions والـ APIs (Cheatsheet)

#### GPIO Gate Clock Ops

| Function | النوع | الـ ops table |
|---|---|---|
| `clk_gpio_gate_enable` | `.enable` | `clk_gpio_gate_ops` |
| `clk_gpio_gate_disable` | `.disable` | `clk_gpio_gate_ops` |
| `clk_gpio_gate_is_enabled` | `.is_enabled` | `clk_gpio_gate_ops` |
| `clk_sleeping_gpio_gate_prepare` | `.prepare` | `clk_sleeping_gpio_gate_ops` |
| `clk_sleeping_gpio_gate_unprepare` | `.unprepare` | `clk_sleeping_gpio_gate_ops` |
| `clk_sleeping_gpio_gate_is_prepared` | `.is_prepared` | `clk_sleeping_gpio_gate_ops` |

#### GPIO Mux Clock Ops

| Function | النوع | الـ ops table |
|---|---|---|
| `clk_gpio_mux_get_parent` | `.get_parent` | `clk_gpio_mux_ops` |
| `clk_gpio_mux_set_parent` | `.set_parent` | `clk_gpio_mux_ops` |
| `__clk_mux_determine_rate` | `.determine_rate` | `clk_gpio_mux_ops` (CCF helper) |

#### Gated Fixed Clock Ops

| Function | النوع | الـ ops table |
|---|---|---|
| `clk_gated_fixed_recalc_rate` | `.recalc_rate` | كلا الـ ops |
| `clk_gated_fixed_prepare` | `.prepare` | `clk_gated_fixed_ops` |
| `clk_gated_fixed_unprepare` | `.unprepare` | `clk_gated_fixed_ops` |
| `clk_gated_fixed_is_prepared` | `.is_prepared` | `clk_gated_fixed_ops` |
| `clk_sleeping_gated_fixed_prepare` | `.prepare` | `clk_sleeping_gated_fixed_ops` |
| `clk_sleeping_gated_fixed_unprepare` | `.unprepare` | `clk_sleeping_gated_fixed_ops` |

#### Registration & Probe

| Function | الغرض |
|---|---|
| `clk_register_gpio` | core internal builder لأي GPIO clock |
| `clk_hw_register_gpio_gate` | يسجل gate clock (sleeping أو لأ) |
| `clk_hw_register_gpio_mux` | يسجل mux clock |
| `gpio_clk_driver_probe` | platform probe لـ `gpio-gate-clock` / `gpio-mux-clock` |
| `clk_gated_fixed_probe` | platform probe لـ `gated-fixed-clock` |

---

### المجموعة 1: GPIO Gate Clock — Non-Sleeping Ops

**الغرض:** الـ ops دي بتتحكم في enable/disable الـ clock عن طريق GPIO في السياق الـ atomic (interrupt-safe). بتُستخدم لما الـ GPIO مش sleeping (يعني مش I2C/SPI expander).

---

#### `clk_gpio_gate_enable`

```c
static int clk_gpio_gate_enable(struct clk_hw *hw)
```

**الـ clk framework** بيستدعيها جوه `clk_enable()` اللي بتتشغل مع الـ `enable_lock` spinlock ممسوك. بتشيل الـ `clk_gpio` من الـ `clk_hw` باستخدام `to_clk_gpio` ثم بتعمل `gpiod_set_value(gpiod, 1)` لتشغيل الـ GPIO.

| Parameter | الوصف |
|---|---|
| `hw` | pointer للـ `clk_hw` المضمّن جوه `clk_gpio` |

**Return:** دايمًا `0` — مفيش error path هنا.

**Key details:**
- لازم تتنادى من context مش بيتقدر ينام (atomic). الـ `gpiod_set_value` نفسه مش بيعمل sleep.
- لو الـ GPIO كان sleeping، المفروض الكود اختار `clk_sleeping_gpio_gate_ops` بدلها.
- بتُنادى من `clk_core_enable()` في `drivers/clk/clk.c`.

---

#### `clk_gpio_gate_disable`

```c
static void clk_gpio_gate_disable(struct clk_hw *hw)
```

عكس الـ enable — بتعمل `gpiod_set_value(gpiod, 0)` لإيقاف الـ clock. بتُنادى من `clk_core_disable()`.

| Parameter | الوصف |
|---|---|
| `hw` | pointer للـ `clk_hw` |

**Return:** `void`.

**Key details:** نفس القيود الـ atomic. مش بتنام.

---

#### `clk_gpio_gate_is_enabled`

```c
static int clk_gpio_gate_is_enabled(struct clk_hw *hw)
```

بتقرأ الحالة الحالية للـ GPIO بـ `gpiod_get_value(gpiod)`. بتُنادى من الـ CCF لما يحتاج يعرف هل الـ clock شغال فعلاً على الـ hardware ولا لأ، خصوصًا أثناء الـ `clk_disable_unused`.

| Parameter | الوصف |
|---|---|
| `hw` | pointer للـ `clk_hw` |

**Return:** `1` لو الـ GPIO high (enabled)، `0` لو low.

**Key details:** بتقرأ من الـ hardware مباشرة — مش cache. لازم تكون atomic-safe.

---

### المجموعة 2: GPIO Gate Clock — Sleeping Ops

**الغرض:** لما يكون الـ GPIO على expander يشتغل فوق I2C أو SPI — يعني الـ set/get بتاخد وقت وممكن ينام. الـ CCF بيستدعي الـ ops دي من `clk_prepare`/`clk_unprepare` اللي بيتشغلوا مع `prepare_mutex` (mutex مش spinlock)، فبيُسمح بالنوم.

---

#### `clk_sleeping_gpio_gate_prepare`

```c
static int clk_sleeping_gpio_gate_prepare(struct clk_hw *hw)
```

بتعمل `gpiod_set_value_cansleep(gpiod, 1)` — نفس الـ enable لكن في الـ prepare context اللي بيقدر ينام. في الـ sleeping model، الـ prepare هو اللي بيشغل الـ GPIO فعلاً.

| Parameter | الوصف |
|---|---|
| `hw` | pointer للـ `clk_hw` |

**Return:** دايمًا `0`.

**Key details:** بتُنادى مع `prepare_mutex` ممسوك. `gpiod_set_value_cansleep` ممكن تعمل sleep — ده الفرق الجوهري عن الـ non-sleeping variant.

---

#### `clk_sleeping_gpio_gate_unprepare`

```c
static void clk_sleeping_gpio_gate_unprepare(struct clk_hw *hw)
```

بتعمل `gpiod_set_value_cansleep(gpiod, 0)` لإيقاف الـ clock في الـ unprepare context.

**Return:** `void`.

---

#### `clk_sleeping_gpio_gate_is_prepared`

```c
static int clk_sleeping_gpio_gate_is_prepared(struct clk_hw *hw)
```

بتقرأ حالة الـ GPIO بـ `gpiod_get_value_cansleep`. بتُنادى من الـ CCF للتحقق من حالة الـ prepare.

**Return:** `1` أو `0` حسب حالة الـ GPIO.

**Key details:** ممكن تنام — مسموح لأنها جوه `.is_prepared` اللي بتُنادى من sleeping context.

---

### المجموعة 3: GPIO Mux Clock Ops

**الغرض:** الـ mux بيستخدم GPIO واحد لاختيار بين parent اتنين. الـ GPIO high = parent index 1، GPIO low = parent index 0. كل الـ ops sleeping لأن الـ parent switching مش لازم يكون atomic.

---

#### `clk_gpio_mux_get_parent`

```c
static u8 clk_gpio_mux_get_parent(struct clk_hw *hw)
```

بتقرأ الـ GPIO بـ `gpiod_get_value_cansleep` وبترجع القيمة كـ parent index. بتُنادى من الـ CCF أثناء الـ initialization وبعد `clk_set_parent`.

| Parameter | الوصف |
|---|---|
| `hw` | pointer للـ `clk_hw` |

**Return:** `0` أو `1` — الـ index بتاع الـ parent الحالي.

**Key details:** الـ GPIO value بتترجم مباشرة لـ parent index — الـ polarity بتتحكم فيها الـ GPIO descriptor نفسه (active-low/high).

---

#### `clk_gpio_mux_set_parent`

```c
static int clk_gpio_mux_set_parent(struct clk_hw *hw, u8 index)
```

بتكتب الـ index (0 أو 1) على الـ GPIO بـ `gpiod_set_value_cansleep`. الـ CCF بيستدعيها لما حد يعمل `clk_set_parent`.

| Parameter | الوصف |
|---|---|
| `hw` | pointer للـ `clk_hw` |
| `index` | الـ parent index المطلوب (0 أو 1) |

**Return:** دايمًا `0`.

**Key details:** مفيش validation إن الـ index بين 0 و 1 — الـ CCF هو المسؤول عن ده قبل ما ينادي الـ callback.

---

### المجموعة 4: Gated Fixed Clock Ops

**الغرض:** الـ `clk_gated_fixed` بيجمع rate ثابت + regulator supply + GPIO enable في clock واحد. الـ prepare بيتحكم في الـ regulator (وممكن الـ GPIO لو sleeping)، والـ enable بيتحكم في الـ GPIO (لو non-sleeping).

---

#### `clk_gated_fixed_recalc_rate`

```c
static unsigned long clk_gated_fixed_recalc_rate(struct clk_hw *hw,
                                                  unsigned long parent_rate)
```

بترجع الـ rate الثابت المخزن في `clk_gated_fixed->rate`. بتتجاهل `parent_rate` خالص لأن الـ clock ده root clock ومالوش parent.

| Parameter | الوصف |
|---|---|
| `hw` | pointer للـ `clk_hw` |
| `parent_rate` | بيتجاهل — الـ clock مالوش parent |

**Return:** الـ rate الثابت بالـ Hz كـ `unsigned long`.

---

#### `clk_gated_fixed_prepare`

```c
static int clk_gated_fixed_prepare(struct clk_hw *hw)
```

بتشغّل الـ regulator supply بـ `regulator_enable`. لو مفيش supply (الـ pointer `NULL`)، بتعود على طول بـ `0`. بتُنادى في الـ prepare context (مع `prepare_mutex`).

| Parameter | الوصف |
|---|---|
| `hw` | pointer للـ `clk_hw` |

**Return:** `0` بالنجاح، أو كود خطأ من `regulator_enable`.

**Key details:**
- الـ `supply` ممكن يكون `NULL` لو مفيش regulator في الـ DT — الدالة بتتعامل مع الحالة دي بـ early return.
- `regulator_enable` بيزود reference count على الـ regulator، مش بس بيشغله.

---

#### `clk_gated_fixed_unprepare`

```c
static void clk_gated_fixed_unprepare(struct clk_hw *hw)
```

بتعمل `regulator_disable` لتقليل الـ reference count على الـ regulator. لو `supply` هو `NULL`، بترجع فورًا.

**Return:** `void`.

**Key details:** لازم يكون متوازن مع كل `clk_gated_fixed_prepare` ناجح.

---

#### `clk_gated_fixed_is_prepared`

```c
static int clk_gated_fixed_is_prepared(struct clk_hw *hw)
```

بتسأل الـ regulator subsystem هل الـ regulator مفعّل بـ `regulator_is_enabled`. لو `supply` هو `NULL`، بترجع `true` دايمًا.

**Return:** `1` لو prepared، `0` لو لأ.

**Key details:** الـ CCF بيستخدم الدالة دي للـ bookkeeping — لو الـ clock اتـ prepare من خارج kernel (مثلاً bootloader)، الـ CCF بيقدر يكتشف الحالة الصحيحة.

---

#### `clk_sleeping_gated_fixed_prepare`

```c
static int clk_sleeping_gated_fixed_prepare(struct clk_hw *hw)
```

دي الـ prepare المركّبة للـ sleeping variant. بتشغّل الـ regulator أولاً عن طريق `clk_gated_fixed_prepare`، ثم بتشغّل الـ GPIO بـ `clk_sleeping_gpio_gate_prepare`. لو الـ GPIO فشل، بتـ rollback الـ regulator.

**Pseudocode:**
```
ret = clk_gated_fixed_prepare(hw)      // enable regulator
if ret: return ret

ret = clk_sleeping_gpio_gate_prepare(hw)  // set GPIO high
if ret:
    clk_gated_fixed_unprepare(hw)      // rollback regulator
return ret
```

**Return:** `0` بالنجاح، أو كود خطأ.

**Key details:**
- الـ rollback pattern هنا مهم — لو GPIO فشل، الـ regulator بيتـ disable للحفاظ على consistency.
- في الـ sleeping variant، الـ GPIO بيتحكم فيه في الـ prepare (مش enable) لأنه ممكن ينام.

---

#### `clk_sleeping_gated_fixed_unprepare`

```c
static void clk_sleeping_gated_fixed_unprepare(struct clk_hw *hw)
```

بتعمل disable للـ regulator وبعدين بتنزّل الـ GPIO. الترتيب معكوس الـ prepare عشان تضمن shutdown آمن.

```c
clk_gated_fixed_unprepare(hw);         // disable regulator
clk_sleeping_gpio_gate_unprepare(hw);  // set GPIO low
```

**Return:** `void`.

---

### المجموعة 5: Internal Registration

**الغرض:** الدوال دي هي الـ internal builders اللي بتبني الـ `clk_gpio` struct وبتسجله في الـ CCF. مش exposed للخارج.

---

#### `clk_register_gpio`

```c
static struct clk_hw *clk_register_gpio(struct device *dev, u8 num_parents,
                                         struct gpio_desc *gpiod,
                                         const struct clk_ops *clk_gpio_ops)
```

دي الـ core function اللي بتعمل كل الشغل الفعلي للتسجيل. بتعمل `devm_kzalloc` للـ `clk_gpio` struct، بتملا الـ `clk_init_data`، وبتنادي `devm_clk_hw_register`.

**Pseudocode:**
```
clk_gpio = devm_kzalloc(dev, sizeof(*clk_gpio))
if !clk_gpio: return ERR_PTR(-ENOMEM)

init.name = dev->of_node->name
init.ops  = clk_gpio_ops
init.parent_data = [{index=0}, {index=1}]  // max 2 parents
init.num_parents = num_parents
init.flags = CLK_SET_RATE_PARENT

clk_gpio->gpiod = gpiod
clk_gpio->hw.init = &init

err = devm_clk_hw_register(dev, &clk_gpio->hw)
if err: return ERR_PTR(err)

return &clk_gpio->hw
```

| Parameter | الوصف |
|---|---|
| `dev` | الـ platform device — مهم للـ devm allocations |
| `num_parents` | عدد الـ parents (0 أو 1 للـ gate، 2 للـ mux) |
| `gpiod` | الـ GPIO descriptor المكتسب مسبقًا |
| `clk_gpio_ops` | الـ ops table المناسبة |

**Return:** pointer للـ `clk_hw` بالنجاح، أو `ERR_PTR` بالفشل.

**Key details:**
- الـ `parent_data` array معرّف دايمًا بـ 2 entries لكن `num_parents` هو اللي بيحدد قد إيه منهم الـ CCF بيستخدم.
- الـ `CLK_SET_RATE_PARENT` flag بيخلي الـ rate propagation تعدي للـ parent.
- الـ `devm_*` بتضمن cleanup أوتوماتيك لما الـ device يتـ unbind.

---

#### `clk_hw_register_gpio_gate`

```c
static struct clk_hw *clk_hw_register_gpio_gate(struct device *dev,
                                                  int num_parents,
                                                  struct gpio_desc *gpiod)
```

بتحدد الـ ops المناسبة (sleeping ولا لأ) بناءً على `gpiod_cansleep(gpiod)`، ثم بتنادي `clk_register_gpio`.

| Parameter | الوصف |
|---|---|
| `dev` | الـ platform device |
| `num_parents` | عدد الـ parents من الـ DT |
| `gpiod` | الـ GPIO descriptor |

**Return:** `clk_hw *` أو `ERR_PTR`.

**Key details:** الـ dispatch بين `clk_gpio_gate_ops` و `clk_sleeping_gpio_gate_ops` بيحصل هنا بناءً على قدرة الـ GPIO hardware.

---

#### `clk_hw_register_gpio_mux`

```c
static struct clk_hw *clk_hw_register_gpio_mux(struct device *dev,
                                                struct gpio_desc *gpiod)
```

wrapper بسيطة — بتنادي `clk_register_gpio` بـ `num_parents=2` و `clk_gpio_mux_ops` ثابتة.

**Return:** `clk_hw *` أو `ERR_PTR`.

---

### المجموعة 6: Platform Driver Probe Functions

**الغرض:** دي نقطة الدخول من الـ platform bus. الـ kernel بينادي الـ probe لما يلاقي device يطابق أحد الـ compatible strings.

---

#### `gpio_clk_driver_probe`

```c
static int gpio_clk_driver_probe(struct platform_device *pdev)
```

الـ probe بتاعت الـ `gpio-gate-clock` و `gpio-mux-clock`. بتقرأ الـ DT، بتحصل على الـ GPIO، وبتسجل الـ clock.

**Pseudocode:**
```
is_mux = of_device_is_compatible(node, "gpio-mux-clock")
num_parents = of_clk_get_parent_count(node)

if is_mux && num_parents != 2:
    return -EINVAL  // mux must have exactly 2 parents

gpio_name = is_mux ? "select" : "enable"
gpiod = devm_gpiod_get(dev, gpio_name, GPIOD_OUT_LOW)

if is_mux:
    hw = clk_hw_register_gpio_mux(dev, gpiod)
else:
    hw = clk_hw_register_gpio_gate(dev, num_parents, gpiod)

return devm_of_clk_add_hw_provider(dev, of_clk_hw_simple_get, hw)
```

| Parameter | الوصف |
|---|---|
| `pdev` | الـ platform device من الـ OF |

**Return:** `0` بالنجاح، أو كود خطأ سالب.

**Key details:**
- الـ `GPIOD_OUT_LOW` يضمن إن الـ GPIO بيبدأ low (clock مش شغال) بعد الـ probe مباشرة.
- الـ `devm_of_clk_add_hw_provider` بيسجل الـ clock provider في الـ OF clock lookup table.
- كل الـ resources (gpio, clock) بتتحرر أوتوماتيك عند الـ unbind بسبب الـ devm.

---

#### `clk_gated_fixed_probe`

```c
static int clk_gated_fixed_probe(struct platform_device *pdev)
```

الأكثر تعقيدًا — بتبني `clk_gated_fixed` struct كامل من الـ DT properties.

**Pseudocode:**
```
clk = devm_kzalloc(dev, sizeof(*clk))

// read rate from "clock-frequency" property (mandatory)
device_property_read_u32(dev, "clock-frequency", &rate)
clk->rate = rate

// read clock name (optional, falls back to node name)
ret = device_property_read_string(dev, "clock-output-names", &clk_name)
if ret: clk_name = fwnode_get_name(dev->fwnode)

// get optional regulator "vdd-supply" (ENODEV = not present, not error)
clk->supply = devm_regulator_get_optional(dev, "vdd")
if PTR_ERR(supply) == -ENODEV: clk->supply = NULL

// get optional GPIO "enable-gpios"
clk->clk_gpio.gpiod = devm_gpiod_get_optional(dev, "enable", GPIOD_OUT_LOW)

// choose ops based on GPIO sleep capability
if gpiod_cansleep(gpiod):
    ops = &clk_sleeping_gated_fixed_ops
else:
    ops = &clk_gated_fixed_ops

clk->clk_gpio.hw.init = CLK_HW_INIT_NO_PARENT(clk_name, ops, 0)

devm_clk_hw_register(dev, &clk->clk_gpio.hw)
devm_of_clk_add_hw_provider(dev, of_clk_hw_simple_get, &clk->clk_gpio.hw)
```

| Parameter | الوصف |
|---|---|
| `pdev` | الـ platform device |

**Return:** `0` بالنجاح، أو كود خطأ.

**Key details:**
- `CLK_HW_INIT_NO_PARENT` يعني الـ clock ده root — مالوش parent في الـ clock tree.
- الـ regulator optional — ممكن الـ clock تشتغل من غير supply (مثلاً لو الـ power دايمًا موجود).
- الـ GPIO كمان optional — ممكن يكون fixed always-on clock لو مفيش GPIO.
- الـ `gpiod_cansleep(NULL)` بترجع `false`، يعني لو مفيش GPIO الـ non-sleeping ops هي اللي بتتاخد.

---

### ملاحظة على الـ Ops Tables

```
clk_gpio_gate_ops:               clk_sleeping_gpio_gate_ops:
  .enable    = gate_enable          .prepare    = sleeping_gate_prepare
  .disable   = gate_disable         .unprepare  = sleeping_gate_unprepare
  .is_enabled = gate_is_enabled     .is_prepared = sleeping_gate_is_prepared

clk_gpio_mux_ops:
  .get_parent      = mux_get_parent
  .set_parent      = mux_set_parent
  .determine_rate  = __clk_mux_determine_rate  (CCF generic)

clk_gated_fixed_ops:             clk_sleeping_gated_fixed_ops:
  .prepare    = gated_fixed_prepare   .prepare    = sleeping_gated_fixed_prepare
  .unprepare  = gated_fixed_unprepare .unprepare  = sleeping_gated_fixed_unprepare
  .is_prepared = gated_fixed_is_prep  .is_prepared = sleeping_gpio_gate_is_prepared
  .enable     = gpio_gate_enable      .recalc_rate = gated_fixed_recalc_rate
  .disable    = gpio_gate_disable
  .is_enabled = gpio_gate_is_enabled
  .recalc_rate = gated_fixed_recalc_rate
```

الفرق الجوهري بين الـ `clk_gated_fixed_ops` والـ `clk_sleeping_gated_fixed_ops`:
- **Non-sleeping**: الـ regulator في `prepare`، الـ GPIO في `enable` (atomic).
- **Sleeping**: كل حاجة (regulator + GPIO) في `prepare`، مفيش `enable` على الإطلاق.

الـ `is_prepared` في الـ sleeping variant بتتحقق من الـ GPIO بس (مش الـ regulator) لأن تشغيل الـ GPIO هو آخر خطوة في الـ prepare، فلو الـ GPIO high يبقى كل حاجة اتعملت.
## Phase 5: دليل الـ Debugging الشامل

الـ driver ده (`clk-gpio.c`) بيتحكم في clock عن طريق GPIO line — إما gate (تشغيل/إيقاف) أو mux (اختيار بين parent clock اتنين). الـ debugging بتاعه بيجمع بين subsystem الـ clock framework والـ GPIO subsystem والـ regulator (في حالة `gated-fixed-clock`).

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ clock framework بيعمل entries تلقائيًا تحت `/sys/kernel/debug/clk/`:

```bash
# اعرض كل الـ clocks الموجودة في النظام
ls /sys/kernel/debug/clk/

# افترض إن اسم الـ clock هو "my-gpio-clk" (اسمه بييجي من DT node name)
CLK_NAME="my-gpio-clk"

# قراءة الـ rate الحالي
cat /sys/kernel/debug/clk/${CLK_NAME}/clk_rate

# قراءة حالة الـ enable (1 = enabled, 0 = disabled)
cat /sys/kernel/debug/clk/${CLK_NAME}/clk_enable_count

# قراءة حالة الـ prepare
cat /sys/kernel/debug/clk/${CLK_NAME}/clk_prepare_count

# قراءة الـ parent الحالي
cat /sys/kernel/debug/clk/${CLK_NAME}/clk_parent

# عرض شجرة الـ clocks كلها (مفيد جداً لشوف الـ parent chain)
cat /sys/kernel/debug/clk/clk_summary
```

**مثال على output من `clk_summary`:**
```
                                 enable  prepare  protect                                duty
   clock                          count    count    count        rate   accuracy phase  cycle
----------------------------------------------------------------------------------------
 osc24M                               3        3        0    24000000          0     0  50000
  my-gpio-clk                         1        1        0    24000000          0     0  50000
```

الـ `enable_count` لازم يتطابق مع حالة الـ GPIO الفعلية — لو الـ count = 1 بس الـ GPIO low، فيه مشكلة.

لفحص الـ GPIO state مباشرة من debugfs:
```bash
# شوف كل الـ GPIO lines وحالتها
cat /sys/kernel/debug/gpio
```

**مثال output:**
```
gpiochip0: GPIOs 0-31, parent: platform/gpio0, gpio0:
 gpio-5  (enable             |gpio-clk            ) out lo
```

الـ `out lo` معناه الـ GPIO low، يعني الـ clock disabled.

---

#### 2. مدخلات الـ sysfs

```bash
# فحص الـ platform device الخاص بالـ driver
ls /sys/bus/platform/devices/ | grep -E "gpio-clk|gated-fixed"

# مثال: لو الـ device اسمه gpio-clk.0
DEV="gpio-clk.0"

# فحص حالة الـ driver binding
cat /sys/bus/platform/devices/${DEV}/driver

# فحص الـ power state
cat /sys/bus/platform/devices/${DEV}/power/runtime_status

# لفحص الـ GPIO المرتبط بالـ clock
# الـ GPIO sysfs (طريقة قديمة للتحقق، مش موصى بيها في production)
ls /sys/class/gpio/

# فحص الـ regulator state (لـ gated-fixed-clock فقط)
ls /sys/class/regulator/
# دور على الـ regulator اللي اسمه "vdd" في الـ DT
cat /sys/class/regulator/regulator.X/state
cat /sys/class/regulator/regulator.X/microvolts
```

---

#### 3. استخدام الـ ftrace

```bash
# تفعيل الـ tracing
mount -t tracefs nodev /sys/kernel/tracing  # لو مش mounted

cd /sys/kernel/tracing

# فحص الـ events المتاحة للـ clk subsystem
ls events/clk/

# الـ events الأهم للـ clk-gpio:
#   clk_enable / clk_disable
#   clk_prepare / clk_unprepare
#   clk_set_parent
#   clk_enable_complete / clk_disable_complete

# تفعيل كل events الـ clock framework
echo 1 > events/clk/enable

# أو تفعيل events محددة
echo 1 > events/clk/clk_enable/enable
echo 1 > events/clk/clk_disable/enable
echo 1 > events/clk/clk_prepare/enable
echo 1 > events/clk/clk_unprepare/enable
echo 1 > events/clk/clk_set_parent/enable

# تفعيل الـ tracing
echo 1 > tracing_on

# شغّل الـ workload اللي عايز تتتبعه
# مثلاً: فعّل device بيستخدم الـ clock

# اقرأ النتايج
cat trace
```

**مثال output:**
```
     kworker/0:1-42    [000] ....   123.456789: clk_prepare: my-gpio-clk
     kworker/0:1-42    [000] ....   123.456800: clk_enable: my-gpio-clk
     kworker/0:1-42    [000] ....   456.123000: clk_disable: my-gpio-clk
     kworker/0:1-42    [000] ....   456.123010: clk_unprepare: my-gpio-clk
```

لو `clk_enable` اتبعت بـ `clk_disable` فوراً من غير شغل حقيقي، يبقى فيه consumer بيعمل enable/disable بسرعة زيادة.

لتتبع الـ GPIO events:
```bash
# فحص GPIO tracepoints
ls events/gpio/
echo 1 > events/gpio/gpio_value/enable
cat trace | grep "gpio-clk\|enable\|select"
```

---

#### 4. تفعيل الـ printk / dynamic debug

الـ `clk-gpio.c` نفسه مش بيعمل `dev_dbg()` كتير، بس الـ framework حواليه بيعمل كتير.

```bash
# تفعيل dynamic debug للـ clk core framework
echo "file drivers/clk/clk.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ gpio-clk driver نفسه (لو في messages)
echo "file drivers/clk/clk-gpio.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لـ GPIO subsystem
echo "file drivers/gpio/gpiolib.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ regulator (لـ gated-fixed-clock)
echo "file drivers/regulator/core.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل كل debug في الـ clk subsystem
echo "module clk +p" > /sys/kernel/debug/dynamic_debug/control

# رؤية الـ messages
dmesg -w | grep -E "clk|gpio|regulator"
```

لتفعيل من الـ bootloader (لمشاكل الـ early boot):
```
# في kernel cmdline
dyndbg="file drivers/clk/clk-gpio.c +p; file drivers/clk/clk.c +p"
```

---

#### 5. الـ Kernel Config Options للـ Debugging

| Option | الوصف |
|--------|-------|
| `CONFIG_CLK_DEBUG` | يفعّل الـ debugfs entries للـ clock framework |
| `CONFIG_GPIOLIB` | ضروري للـ driver يشتغل |
| `CONFIG_GPIO_SYSFS` | يكشف الـ GPIO عبر sysfs |
| `CONFIG_DEBUG_GPIO` | يضيف verbose logging للـ GPIO operations |
| `CONFIG_REGULATOR_DEBUG` | يفعّل debug للـ regulator (لـ gated-fixed-clock) |
| `CONFIG_COMMON_CLK` | الـ common clock framework نفسه |
| `CONFIG_FTRACE` | لازم لأي tracing |
| `CONFIG_TRACING_EVENTS_GPIO` | يفعّل GPIO tracepoints |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `dev_dbg` وأصحابه |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في الـ prepare_lock/enable_lock |

```bash
# تحقق إن الـ configs مفعّلة في kernel الحالي
zcat /proc/config.gz | grep -E "CONFIG_CLK_DEBUG|CONFIG_DEBUG_GPIO|CONFIG_REGULATOR_DEBUG"
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# clk_summary - أهم أداة للـ clock debugging
cat /sys/kernel/debug/clk/clk_summary | column -t

# clk_dump - JSON-like output لكل الـ clocks
cat /sys/kernel/debug/clk/clk_dump

# فحص الـ GPIO chip والـ lines
gpiodetect          # يعرض كل GPIO chips
gpioinfo gpiochip0  # يعرض كل lines في chip معين

# مثال output من gpioinfo:
# gpiochip0 - 32 lines:
#   line   5: "enable"       "gpio-clk"   output  active-high [used]

# قراءة قيمة GPIO بدون تغييرها
gpioget gpiochip0 5

# تفعيل/تعطيل الـ clock يدوياً للاختبار (خطر! بس مفيد في debugging)
# gpioset gpiochip0 5=1   # enable
# gpioset gpiochip0 5=0   # disable

# لمعرفة مين بيستخدم الـ clock حالياً
cat /sys/kernel/debug/clk/clk_dump | python3 -m json.tool | grep -A5 "my-gpio-clk"
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|----------------------|--------|-------|
| `Can't get 'enable' named GPIO property` | الـ DT مش فيه `enable-gpios` property | تحقق من الـ DT وتأكد إن الـ GPIO موجود وصح |
| `Can't get 'select' named GPIO property` | الـ DT مش فيه `select-gpios` في `gpio-mux-clock` | أضف الـ property في الـ DT |
| `mux-clock must have 2 parents` | الـ `gpio-mux-clock` محتاج بالظبط parent اتنين | راجع `clocks` property في الـ DT |
| `Failed to get clock-frequency` | `gated-fixed-clock` مش لاقي `clock-frequency` property | أضف الـ property للـ DT node |
| `Failed to get regulator` | الـ `vdd-supply` في الـ DT غلط أو الـ regulator مش probe | تحقق من الـ regulator driver وتسلسل الـ probe |
| `Failed to register clock` | `devm_clk_hw_register` فشل، غالباً duplicate name | تأكد إن اسم الـ clock unique في النظام |
| `Failed to register clock provider` | `devm_of_clk_add_hw_provider` فشل | تحقق إن الـ DT node صح ومش متسجل مرتين |
| `-EPROBE_DEFER` (implicit) | الـ GPIO أو الـ regulator لسه مش ready | انتظر، الـ kernel هيعيد الـ probe تلقائياً |
| `-EINVAL` من `gpio_clk_driver_probe` | عدد الـ parents في `gpio-mux-clock` مش 2 | صحح الـ DT |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
static int clk_gpio_gate_enable(struct clk_hw *hw)
{
    struct clk_gpio *clk = to_clk_gpio(hw);

    /* تحقق إن الـ gpiod مش NULL قبل الاستخدام */
    WARN_ON(!clk->gpiod);

    gpiod_set_value(clk->gpiod, 1);
    return 0;
}

static int gpio_clk_driver_probe(struct platform_device *pdev)
{
    /* بعد ما بنجيب الـ gpiod، نتحقق من حالته */
    gpiod = devm_gpiod_get(dev, gpio_name, GPIOD_OUT_LOW);
    if (IS_ERR(gpiod)) {
        /* هنا dev_err_probe بيطبع تلقائياً، بس نضيف dump للـ stack لو محتاجين */
        dump_stack(); /* في kernel debug builds بس */
        return dev_err_probe(...);
    }

    /* نتحقق إن الـ num_parents منطقي */
    WARN_ON(is_mux && num_parents != 2);
}

static int clk_gated_fixed_probe(struct platform_device *pdev)
{
    /* نتحقق إن الـ rate مش صفر */
    WARN_ON(rate == 0);

    /* بعد register، نتحقق */
    ret = devm_clk_hw_register(dev, &clk->clk_gpio.hw);
    WARN_ON(ret && ret != -EPROBE_DEFER);
}
```

أهم نقاط للـ `WARN_ON`:
- لو `gpiod` بقى `NULL` في وقت الـ enable (مفروض يتمسك من الـ probe)
- لو الـ `rate` في `clk_gated_fixed` = 0
- لو الـ `prepare_count` اكبر من `enable_count` بكتير (يدل على leak)

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware تتطابق مع حالة الـ Kernel

الـ `clk-gpio.c` بيتحكم في clock عن طريق GPIO output. التطابق المطلوب:

| حالة الـ Kernel | القيمة المتوقعة على الـ GPIO Pin |
|-----------------|----------------------------------|
| `clk_enable_count > 0` | الـ pin = HIGH (أو LOW لو active-low في DT) |
| `clk_enable_count == 0` | الـ pin = LOW |
| `clk_prepare_count > 0` (sleeping gate) | الـ pin = HIGH |
| `clk_gpio_mux` parent = 0 | الـ pin = LOW |
| `clk_gpio_mux` parent = 1 | الـ pin = HIGH |
| `gated-fixed` supply enabled | الـ regulator output = مضبوط |

للتحقق بالأوامر:
```bash
# قارن بين kernel state والـ GPIO الفعلي
CLK_ENABLE=$(cat /sys/kernel/debug/clk/my-gpio-clk/clk_enable_count)
GPIO_VAL=$(gpioget gpiochip0 5)

echo "Kernel enable_count: ${CLK_ENABLE}"
echo "GPIO actual value:   ${GPIO_VAL}"

# لو CLK_ENABLE > 0 بس GPIO_VAL = 0، فيه bug
[ "${CLK_ENABLE}" -gt 0 ] && [ "${GPIO_VAL}" -eq 0 ] && echo "MISMATCH DETECTED!"
```

---

#### 2. تقنيات الـ Register Dump

الـ GPIO registers بتختلف حسب الـ SoC. الطرق العامة:

```bash
# طريقة 1: devmem2 (لو متاح)
# مثال لـ GPIO controller على عنوان 0x4804C000 (OMAP style)
# قراءة الـ GPIO output register
devmem2 0x4804C13C   # GPIO_DATAOUT register
devmem2 0x4804C134   # GPIO_OE register (0=output, 1=input)

# طريقة 2: /dev/mem مع dd
# قراءة 4 bytes من عنوان GPIO register
dd if=/dev/mem bs=4 count=1 skip=$((0x4804C13C / 4)) 2>/dev/null | xxd

# طريقة 3: io utility (من package io أو busybox)
io -4 -r 0x4804C13C

# طريقة 4: من داخل الـ kernel (لو بتكتب debug code مؤقت)
# ioremap(gpio_base, size) ثم readl(base + offset)
```

لمعرفة عنوان الـ GPIO controller:
```bash
# من الـ DT compiled
cat /proc/device-tree/gpio@4804C000/reg | xxd

# أو من resources
cat /sys/bus/platform/devices/4804c000.gpio/resource
```

---

#### 3. نصائح الـ Logic Analyzer / Oscilloscope

**نقاط القياس:**
- الـ GPIO pin المتحكم في الـ clock (enable أو select)
- خرج الـ clock نفسه (لو ممكن الوصول ليه)
- الـ PMIC power rail (لـ gated-fixed-clock)

**ما تدور عليه:**

```
للـ gpio-gate-clock:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GPIO Enable Pin: _____|‾‾‾‾‾‾‾‾‾‾‾‾‾‾|_________
Clock Output:    _____|▓▓▓▓▓▓▓▓▓▓▓▓▓▓|_________
                      ↑               ↑
                 clk_enable()    clk_disable()

للـ gpio-mux-clock:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GPIO Select Pin: _____|‾‾‾‾‾‾‾‾‾‾‾‾‾‾
Parent 0 (24MHz): ▓▓▓▓|________________
Parent 1 (48MHz): ____|▓▓▓▓▓▓▓▓▓▓▓▓▓▓
                       ↑
                  clk_set_parent(1)
```

**أوقات مهمة تقيسها:**
- الـ glitch على الـ clock output لما بتعمل mux switching (المفروض يحصل في وقت الـ prepare مش الـ enable)
- الـ delay بين تفعيل الـ GPIO والـ clock يبدأ يشتغل فعلاً (بعض الـ oscillators محتاجين وقت startup)

**Trigger settings:**
- Trigger على الـ rising edge للـ GPIO enable pin
- قيس الـ delay لحد ما الـ clock يوصل للـ frequency المطلوبة
- لو الـ delay أكبر من 1ms، الـ driver محتاج `CLK_SET_RATE_UNGATE` أو sleep في الـ prepare

---

#### 4. مشاكل الـ Hardware الشائعة ← نمط الـ Kernel Log

| المشكلة | نمط الـ Log | التشخيص |
|---------|------------|---------|
| GPIO مش موجود فعلياً على الـ PCB | `Can't get 'enable' named GPIO property` + `-ENOENT` | تحقق من الـ schematic |
| GPIO controller مش probe اتجه | `-EPROBE_DEFER` بيتكرر | تحقق من الـ GPIO driver |
| GPIO active-low بس DT مش بيقوله | الـ clock يبان enabled بس هو معطل | أضف `GPIO_ACTIVE_LOW` في الـ DT |
| تداخل في الـ GPIO بين drivers تانيين | `gpio-X: Kernel is going to sleep in a non-sleeping context` | تحقق إن الـ GPIO مش shared |
| الـ oscillator محتاج وقت startup | جهاز يشتغل بشكل متقطع في البداية | استخدم sleeping gate أو أضف delay في consumer |
| regulator ما اتفعلش قبل الـ GPIO | جهاز يشتغل بدون power | تحقق من ترتيب الـ probe وإن `vdd-supply` موجود في DT |
| الـ mux بيقلب بين parents بدون سبب | messages عشوائية من الـ clock consumer | تحقق من الـ `determine_rate` logic في الـ consumer |

---

#### 5. Device Tree Debugging

```bash
# التحقق إن الـ DT compiled صح
# مثال DT صحيح لـ gpio-gate-clock:
cat << 'EOF'
clk_gpio_gate: gpio-gate-clock {
    compatible = "gpio-gate-clock";
    clocks = <&parent_clk>;
    #clock-cells = <0>;
    enable-gpios = <&gpio0 5 GPIO_ACTIVE_HIGH>;
};
EOF

# مثال DT لـ gpio-mux-clock:
cat << 'EOF'
clk_gpio_mux: gpio-mux-clock {
    compatible = "gpio-mux-clock";
    clocks = <&clk_24m>, <&clk_48m>;  /* لازم يكونوا 2 بالظبط */
    #clock-cells = <0>;
    select-gpios = <&gpio0 6 GPIO_ACTIVE_HIGH>;
};
EOF

# مثال DT لـ gated-fixed-clock:
cat << 'EOF'
clk_gated_fixed: gated-fixed-clock {
    compatible = "gated-fixed-clock";
    clock-frequency = <24000000>;
    clock-output-names = "my-fixed-clk";
    enable-gpios = <&gpio0 7 GPIO_ACTIVE_HIGH>;
    vdd-supply = <&reg_3v3>;  /* اختياري */
    #clock-cells = <0>;
};
EOF

# قراءة الـ DT الفعلي اللي اتلود
# بعد boot:
ls /proc/device-tree/
cat /proc/device-tree/gpio-gate-clock/compatible

# تحويل الـ DTB للـ DTS لمراجعته
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -A 20 "gpio-gate-clock\|gpio-mux-clock\|gated-fixed-clock"

# التحقق إن الـ clocks property بيشاور على الـ phandles الصح
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | grep -B5 -A15 "enable-gpios\|select-gpios"

# تحقق من الـ GPIO controller phandle
cat /proc/device-tree/gpio0/phandle | xxd

# تحقق إن الـ #clock-cells = <0>
cat /proc/device-tree/gpio-gate-clock/\#clock-cells | xxd
# المتوقع: 00 00 00 00
```

**أهم نقاط الـ DT تتحقق منها:**
1. `compatible` = `"gpio-gate-clock"` أو `"gpio-mux-clock"` أو `"gated-fixed-clock"` بالظبط
2. `gpio-mux-clock` لازم يكون فيه `clocks` property بـ phandle اتنين بالظبط
3. الـ GPIO active polarity (ACTIVE_HIGH/ACTIVE_LOW) لازم تتطابق مع الـ hardware
4. اسم الـ GPIO property: `enable-gpios` للـ gate، `select-gpios` للـ mux

---

### Practical Commands

#### مجموعة أوامر جاهزة للـ Copy-Paste

```bash
#!/bin/bash
# === clk-gpio Full Debug Script ===

CLK_NAME="${1:-my-gpio-clk}"  # مرر اسم الـ clock كـ argument
DEBUGFS="/sys/kernel/debug"

echo "=== Clock Summary ==="
cat ${DEBUGFS}/clk/clk_summary | grep -E "clock|${CLK_NAME}" | head -20

echo ""
echo "=== GPIO State ==="
cat ${DEBUGFS}/gpio | grep -E "gpio-clk|enable|select"

echo ""
echo "=== Clock Specific Debug ==="
if [ -d "${DEBUGFS}/clk/${CLK_NAME}" ]; then
    echo "Rate:          $(cat ${DEBUGFS}/clk/${CLK_NAME}/clk_rate)"
    echo "Enable count:  $(cat ${DEBUGFS}/clk/${CLK_NAME}/clk_enable_count)"
    echo "Prepare count: $(cat ${DEBUGFS}/clk/${CLK_NAME}/clk_prepare_count)"
    echo "Parent:        $(cat ${DEBUGFS}/clk/${CLK_NAME}/clk_parent 2>/dev/null || echo 'N/A')"
else
    echo "Clock ${CLK_NAME} not found in debugfs!"
    echo "Available clocks:"
    ls ${DEBUGFS}/clk/ | head -20
fi

echo ""
echo "=== Recent Kernel Messages ==="
dmesg | grep -E "gpio-clk|gpio_clk|gated-fixed|clk-gpio" | tail -20

echo ""
echo "=== Platform Devices ==="
ls /sys/bus/platform/devices/ | grep -E "gpio.clk|gated.fixed"
```

```bash
# تفعيل ftrace للـ clock events بسرعة
TRACING=/sys/kernel/tracing
echo 0 > ${TRACING}/tracing_on
echo "" > ${TRACING}/trace
echo "function_graph" > ${TRACING}/current_tracer
echo "clk_gpio_gate_enable" > ${TRACING}/set_graph_function
echo "clk_gpio_gate_disable" >> ${TRACING}/set_graph_function
echo "clk_gpio_mux_set_parent" >> ${TRACING}/set_graph_function
echo 1 > ${TRACING}/tracing_on
# ... شغّل workload ...
echo 0 > ${TRACING}/tracing_on
cat ${TRACING}/trace
```

```bash
# تحقق سريع من تطابق kernel state مع hardware
check_gpio_clk_consistency() {
    local CLK=$1
    local CHIP=$2
    local LINE=$3

    ENABLE_COUNT=$(cat /sys/kernel/debug/clk/${CLK}/clk_enable_count 2>/dev/null)
    GPIO_VAL=$(gpioget ${CHIP} ${LINE} 2>/dev/null)

    echo "Clock: ${CLK}"
    echo "  Kernel enable_count: ${ENABLE_COUNT}"
    echo "  GPIO actual value:   ${GPIO_VAL}"

    if [ "${ENABLE_COUNT:-0}" -gt 0 ] && [ "${GPIO_VAL:-0}" -eq 0 ]; then
        echo "  STATUS: *** MISMATCH *** (kernel thinks enabled, GPIO is low)"
    elif [ "${ENABLE_COUNT:-0}" -eq 0 ] && [ "${GPIO_VAL:-0}" -eq 1 ]; then
        echo "  STATUS: *** MISMATCH *** (kernel thinks disabled, GPIO is high)"
    else
        echo "  STATUS: OK (consistent)"
    fi
}

# استخدام:
check_gpio_clk_consistency "my-gpio-clk" "gpiochip0" 5
```

```bash
# تشخيص مشاكل الـ probe
dmesg | grep -E "gpio.clk|gated.fixed" | grep -E "error|fail|probe|defer|EPROBE"

# شوف الـ device tree nodes المرتبطة
find /proc/device-tree -name "compatible" -exec grep -l "gpio-gate-clock\|gpio-mux-clock\|gated-fixed-clock" {} \; 2>/dev/null | while read f; do
    dir=$(dirname $f)
    echo "Node: $dir"
    echo "  compatible: $(cat $f)"
    ls $dir | grep -E "gpios|clocks|clock-frequency"
done
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: بورد RK3562 — UART مش شغال بعد boot

#### العنوان
**الـ UART console مش بيظهر على industrial gateway بعد power-on**

#### السياق
بورد custom مبني على **RK3562** بيشتغل كـ industrial IoT gateway. الـ UART0 بيستخدم كـ debug console. الـ crystal oscillator الخاص بالـ UART clock controller متصل بـ GPIO enable pin — يعني في `gpio-gate-clock` node في الـ DT بيتحكم في تشغيل الـ oscillator.

#### المشكلة
بعد boot، الـ kernel messages بتوقف بعد early printk مباشرةً. الـ serial console مش بيشتغل خالص. الـ engineer بيحاول `cat /sys/kernel/debug/clk/clk_summary` فبيلاقي الـ clock اللي بيغذي الـ UART disabled.

#### التحليل
الـ DT node كان:
```dts
uart_clk_gate: uart-clk-gate {
    compatible = "gpio-gate-clock";
    clocks = <&osc24m>;
    #clock-cells = <0>;
    enable-gpios = <&gpio3 12 GPIO_ACTIVE_HIGH>;
};
```

في `gpio_clk_driver_probe` (السطر 195)، الكود بيعمل:
```c
gpiod = devm_gpiod_get(dev, gpio_name, GPIOD_OUT_LOW);
```

**المشكلة هنا:** الـ `GPIOD_OUT_LOW` بيعمل initialize للـ GPIO على LOW في اللحظة اللي بيتسجل فيها الـ clock. يعني الـ oscillator بيتقفل من أول وهو بيتسجل حتى لو محدش طلب منه يقفل.

بعدين في `clk_hw_register_gpio_gate` (السطر 175)، الكود بيشيك:
```c
if (gpiod_cansleep(gpiod))
    ops = &clk_sleeping_gpio_gate_ops;
else
    ops = &clk_gpio_gate_ops;
```

الـ GPIO ده on-SoC (مش I2C expander)، فـ `gpiod_cansleep` بيرجع false، يعني بيستخدم `clk_gpio_gate_ops` اللي فيه:
```c
static int clk_gpio_gate_enable(struct clk_hw *hw)
{
    gpiod_set_value(clk->gpiod, 1);  // non-sleeping set
    return 0;
}
```

المشكلة الحقيقية: الـ consumer (UART driver) بيعمل `clk_prepare_enable()` بس الـ framework بيبقى في state مش consistent لأن الـ GPIO بدأ LOW من غير ما يعمل proper disable sequence.

#### الحل
1. غير الـ DT ليبدأ الـ GPIO بـ HIGH (oscillator enabled من البداية):
```dts
enable-gpios = <&gpio3 12 GPIO_ACTIVE_HIGH>;
/* أو استخدم ACTIVE_LOW لو الـ enable logic معكوسة */
```

2. أو صلح الـ init state في الـ DT باستخدام `assigned-clocks`:
```dts
assigned-clocks = <&uart_clk_gate>;
assigned-clock-parents = <&osc24m>;
```

3. تأكد إن `clk_prepare_enable()` بيتعمل قبل ما الـ UART driver يبدأ يشتغل — وراجع الـ boot order في DT عبر `clocks` property.

#### الدرس المستفاد
**الـ `GPIOD_OUT_LOW` في `gpio_clk_driver_probe` بيعمل assert للـ gate على LOW فور التسجيل** — حتى لو الـ clock كان enabled قبل boot. لازم تفكر في الـ active logic (HIGH/LOW) وتختار `GPIO_ACTIVE_LOW` flag في DT لو الـ oscillator enable signal معكوس.

---

### السيناريو 2: Allwinner H616 — Android TV Box مفيش صوت

#### العنوان
**الـ HDMI audio مش شغال على TV box بسبب gpio-mux-clock misconfiguration**

#### السياق
TV box مبني على **Allwinner H616**. الـ audio subsystem بيستخدم `gpio-mux-clock` لاختيار بين مصدرين للـ clock: الـ internal PLL أو external crystal. الـ MUX بيتحكم فيه GPIO. في Android الـ HDMI audio track بيبدأ بس مفيش صوت.

#### المشكلة
الـ `clk_gpio_mux_get_parent` بيرجع قيمة غلط فالـ framework مش عارف أنهي parent هو الشغال. الـ `aplay -l` بيظهر الـ device موجود بس أي محاولة للتشغيل بتفشل بـ underrun.

#### التحليل
في الـ DT:
```dts
audio_mux: audio-mux-clock {
    compatible = "gpio-mux-clock";
    clocks = <&pll_audio>, <&osc24m>;
    #clock-cells = <0>;
    select-gpios = <&pio 7 5 GPIO_ACTIVE_HIGH>;
};
```

الـ `gpio_clk_driver_probe` (السطر 205) بيتأكد إن في parent اتنين بالظبط:
```c
if (is_mux && num_parents != 2) {
    dev_err(dev, "mux-clock must have 2 parents\n");
    return -EINVAL;
}
```

بعدين `clk_gpio_mux_get_parent` (السطر 119):
```c
static u8 clk_gpio_mux_get_parent(struct clk_hw *hw)
{
    struct clk_gpio *clk = to_clk_gpio(hw);
    return gpiod_get_value_cansleep(clk->gpiod);
    // 0 = clocks[0] = pll_audio
    // 1 = clocks[1] = osc24m
}
```

الهاردوير كان wire بشكل معكوس: لما GPIO=0 بيروح الـ osc24m (مش الـ pll_audio). الـ framework عنده impression إن parent[0] هو الـ pll_audio، بس الفعلي parent[0] على الهاردوير هو الـ osc24m. النتيجة: الـ audio clock بتشتغل على 24MHz بدل 22.5MHz اللازمة للـ HDMI audio.

#### الحل
صلح الترتيب في الـ DT:
```dts
/* رتب الـ parents بحيث index يطابق GPIO value */
clocks = <&osc24m>, <&pll_audio>;
/* GPIO=0 → osc24m (index 0), GPIO=1 → pll_audio (index 1) */
```

أو لو مش ممكن تغير الترتيب، استخدم `GPIO_ACTIVE_LOW` على الـ select-gpio:
```dts
select-gpios = <&pio 7 5 GPIO_ACTIVE_LOW>;
/* بيعكس القيمة: GPIO=0 (active low asserted) → index 1 = pll_audio */
```

للتحقق:
```bash
cat /sys/kernel/debug/clk/audio_mux_clock/clk_parent
# لازم يظهر pll_audio لما الصوت شغال
```

#### الدرس المستفاد
**الـ `gpio-mux-clock` بيربط GPIO value (0 أو 1) مباشرةً بـ index الـ parent في الـ DT** — ترتيب الـ `clocks` property في DT لازم يطابق الـ hardware wiring بالظبط. أي خطأ في الترتيب بيخلي الـ clock framework يختار parent غلط بدون أي error.

---

### السيناريو 3: STM32MP1 — I2C GPIO Expander يعمل kernel BUG

#### العنوان
**Kernel panic عند enable الـ SPI clock على industrial ECU بسبب sleeping GPIO في atomic context**

#### السياق
**STM32MP1**-based automotive ECU. الـ SPI flash clock controller متصل بـ GPIO expander على I2C bus (PCA9535). استخدموا `gpio-gate-clock` بـ GPIO من الـ I2C expander عشان يتحكموا في clock enable للـ SPI.

#### المشكلة
عند الـ boot، الـ kernel بيعمل BUG:
```
BUG: sleeping function called from invalid context
in_atomic(): 1, irqs_disabled(): 0
Call Trace:
 clk_gpio_gate_enable+0x18/0x30
 clk_core_enable+0xac/0x1c0
```

#### التحليل
الـ SPI driver بيعمل `clk_enable()` من context اللي فيه spinlock محجوز (atomic context). الـ framework بيكال `clk_gpio_gate_enable` (السطر 53):

```c
static int clk_gpio_gate_enable(struct clk_hw *hw)
{
    struct clk_gpio *clk = to_clk_gpio(hw);
    gpiod_set_value(clk->gpiod, 1);  // FORBIDDEN في sleeping GPIO!
    return 0;
}
```

المشكلة: `gpiod_set_value` (non-sleeping version) بتتعمل على GPIO اللي `gpiod_cansleep()` بترجع له `true` (لأنه على I2C expander). الكود في `clk_hw_register_gpio_gate` (السطر 175) كان المفروض يكتشف ده:

```c
if (gpiod_cansleep(gpiod))
    ops = &clk_sleeping_gpio_gate_ops;  // prepare/unprepare بدل enable/disable
else
    ops = &clk_gpio_gate_ops;
```

لكن هنا `clk_sleeping_gpio_gate_ops` بيستخدم `prepare/unprepare` بس — **مفيش `enable/disable`**. يعني لو الـ SPI driver بيعمل `clk_enable()` من atomic context، مفيش `.enable` op في الـ ops struct اللي تتعمل. المشكلة الأعمق: الـ I2C GPIO expander مش مناسب أصلاً لـ clock gate لازم يتعمل من atomic context.

#### الحل
الحل الصح معمارياً: استبدل الـ I2C GPIO expander بـ on-SoC GPIO:

```dts
/* قبل: GPIO على I2C expander */
enable-gpios = <&pca9535 4 GPIO_ACTIVE_HIGH>;

/* بعد: GPIO على السوك نفسه */
enable-gpios = <&gpioa 8 GPIO_ACTIVE_HIGH>;
```

لو مش ممكن تغير الهاردوير، استخدم `clk_prepare_enable()` بدل `clk_enable()` من thread context — وتأكد إن الـ SPI driver مش بيعمل clock ops من interrupt handler.

للتأكيد:
```bash
# شوف النوع
cat /sys/kernel/debug/gpio | grep -A2 "pca9535"
# لو sleeping = yes → مش صالح لـ atomic clock gate
```

#### الدرس المستفاد
**الـ `clk_sleeping_gpio_gate_ops` بيعمل gate في `prepare` step مش في `enable` step** — يعني الـ consumers لازم يستخدموا `clk_prepare_enable()` وليس `clk_enable()` فقط، وممنوع استخدام I2C/SPI GPIO expander كـ clock gate لو في أي consumer بيعمل enable من atomic context.

---

### السيناريو 4: i.MX8 — IoT Sensor بيستهلك تيار زيادة في sleep

#### العنوان
**الـ current consumption في deep sleep أعلى من المتوقع على IoT sensor node بـ i.MX8ULP**

#### السياق
IoT sensor device مبني على **i.MX8ULP** بيتشغل على بطارية. الـ design فيه `gated-fixed-clock` بيتحكم في تشغيل 32MHz oscillator خارجي لـ LoRa transceiver. الـ oscillator بيتغذى من regulator منفصل (LDO). المتوقع إن في sleep الـ oscillator وصاحبه الـ LDO يتقفلوا تماماً.

#### المشكلة
قياسات التيار بتظهر إن الـ LDO لسه شغال في sleep رغم إن الـ LoRa driver بيعمل `clk_disable_unprepare()`.

#### التحليل
الـ DT:
```dts
lora_clk: lora-clock {
    compatible = "gated-fixed-clock";
    clock-frequency = <32000000>;
    clock-output-names = "lora-clk";
    vdd-supply = <&ldo3_reg>;
    enable-gpios = <&gpio1 10 GPIO_ACTIVE_HIGH>;
};
```

الـ `clk_gated_fixed_probe` (السطر 356) بيعمل:
```c
clk->supply = devm_regulator_get_optional(dev, "vdd");
```

وبعدين بيشيك على الـ GPIO:
```c
if (gpiod_cansleep(clk->clk_gpio.gpiod))
    ops = &clk_sleeping_gated_fixed_ops;
else
    ops = &clk_gated_fixed_ops;
```

الـ GPIO ده on-SoC فـ `gpiod_cansleep` = false، فبيستخدم `clk_gated_fixed_ops`:

```c
static const struct clk_ops clk_gated_fixed_ops = {
    .prepare   = clk_gated_fixed_prepare,   // يشغل الـ regulator
    .unprepare = clk_gated_fixed_unprepare, // يقفل الـ regulator
    .enable    = clk_gpio_gate_enable,      // يرفع الـ GPIO
    .disable   = clk_gpio_gate_disable,     // يخفض الـ GPIO
    .recalc_rate = clk_gated_fixed_recalc_rate,
};
```

المشكلة: الـ LoRa driver كان بيعمل `clk_disable()` بس من غير `clk_unprepare()`. الـ GPIO بيتخفض (oscillator stops) بس `clk_gated_fixed_unprepare` مش بتتعمل — فالـ `regulator_disable()` مش بيتعمل. الـ LDO فاضل شغال.

#### الحل
صلح الـ LoRa driver ليعمل الـ full sequence:

```c
/* في الـ suspend callback */
clk_disable_unprepare(lora_clk);  // بيعمل disable ثم unprepare
/* مش بس */
clk_disable(lora_clk);            // ده بيسيب الـ regulator شغال!

/* في الـ resume callback */
clk_prepare_enable(lora_clk);     // prepare (regulator on) ثم enable (gpio high)
```

للتحقق:
```bash
# راقب الـ regulator state
cat /sys/kernel/debug/regulator/ldo3_reg/consumers
# المفروض يكون فاضي في sleep
```

#### الدرس المستفاد
**في `clk_gated_fixed_ops`، الـ regulator بيتحكم فيه من `prepare/unprepare` مش من `enable/disable`** — لازم دايماً تستخدم `clk_prepare_enable()` و`clk_disable_unprepare()` كـ pairs كاملة، وإلا الـ power domain هيفضل مشغول حتى لو الـ clock signal وقف.

---

### السيناريو 5: AM62x — Custom Board Bring-up، الـ clock مش بيتسجل

#### العنوان
**`devm_clk_hw_register` بيفشل بـ -ENOENT أثناء bring-up بورد جديد على AM62x**

#### السياق
فريق bring-up بيشتغل على custom board مبني على **TI AM62x**. البورد فيه `gpio-gate-clock` بيتحكم في 25MHz oscillator لـ Ethernet PHY. الـ engineer كاتب الـ DT وبيحاول يعمل boot لأول مرة.

#### المشكلة
في dmesg:
```
gpio-clk gpio-clk: Can't get 'enable' named GPIO property
gpio-clk: probe of gpio-clk failed with error -2
```

الـ `-2` هو `-ENOENT` — مش لاقي الـ GPIO.

#### التحليل
الـ `gpio_clk_driver_probe` (السطر 195) بيحدد اسم الـ GPIO property:
```c
gpio_name = is_mux ? "select" : "enable";
gpiod = devm_gpiod_get(dev, gpio_name, GPIOD_OUT_LOW);
if (IS_ERR(gpiod))
    return dev_err_probe(dev, PTR_ERR(gpiod),
                         "Can't get '%s' named GPIO property\n", gpio_name);
```

يعني الـ DT لازم يكون فيه property اسمها بالظبط `enable-gpios` (مش `gpio` ولا `gpios` ولا `enable-gpio`).

الـ DT اللي كتبه الـ engineer:
```dts
eth_clk_gate: eth-clk-gate {
    compatible = "gpio-gate-clock";
    clocks = <&osc25m>;
    #clock-cells = <0>;
    /* خطأ: الاسم غلط */
    gpio = <&gpio0 14 GPIO_ACTIVE_HIGH>;
};
```

الاسم الصح حسب الـ kernel GPIO naming convention هو `enable-gpios` (لأن `devm_gpiod_get(dev, "enable", ...)` بتدور على `enable-gpios` في DT).

#### الحل
صلح الـ DT:
```dts
eth_clk_gate: eth-clk-gate {
    compatible = "gpio-gate-clock";
    clocks = <&osc25m>;
    #clock-cells = <0>;
    /* صح: kernel بيضيف -gpios تلقائياً */
    enable-gpios = <&gpio0 14 GPIO_ACTIVE_HIGH>;
};
```

للـ `gpio-mux-clock` الاسم المطلوب هو `select-gpios`:
```dts
select-gpios = <&gpio0 15 GPIO_ACTIVE_HIGH>;
```

للتحقق قبل boot:
```bash
# تأكد من الـ DT compiled بشكل صح
dtc -I dtb -O dts /boot/am62x-custom.dtb | grep -A5 "gpio-gate-clock"

# أو من داخل kernel
cat /sys/firmware/devicetree/base/eth-clk-gate/enable-gpios
```

#### الدرس المستفاد
**الـ `devm_gpiod_get(dev, "enable", ...)` بتدور على property اسمها `enable-gpios` في الـ DT** — الـ GPIO subsystem بيضيف الـ `-gpios` suffix تلقائياً. الأسماء المطلوبة في هذا الـ driver هي `enable-gpios` للـ gate و`select-gpios` للـ mux، وأي اسم تاني بيدي `-ENOENT` على طول.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

| المقال | الوصف |
|--------|-------|
| [A common clock framework](https://lwn.net/Articles/472998/) | المقال الأساسي اللي شرح الـ Common Clock Framework لما اتضاف في kernel 3.4 سنة 2012 |
| [common clk framework (patch series)](https://lwn.net/Articles/486841/) | نقاش patch series الـ CCF الأولانية على الـ mailing list |
| [Documentation/clk.txt](https://lwn.net/Articles/489668/) | الـ documentation الرسمية اللي اتنشرت مع الـ CCF |
| [Binding and driver for gated-fixed-clocks](https://lwn.net/Articles/987538/) | المقال اللي غطى إضافة `gated-fixed-clock` لـ `clk-gpio.c` — ده أهم مرجع مباشر للـ driver |
| [The Common Clk Framework — Kernel Docs](https://static.lwn.net/kerneldoc/driver-api/clk.html) | الـ documentation الرسمية للـ CCF على static.lwn.net |
| [Subsystem drivers using GPIO](https://static.lwn.net/kerneldoc/driver-api/gpio/drivers-on-gpio.html) | بيشرح إزاي الـ subsystems زي الـ clk بتستخدم الـ GPIO API |

---

### الـ Kernel Documentation الرسمية

**الـ paths دي موجودة في شجرة الـ kernel:**

```
Documentation/driver-api/clk.rst          ← الـ CCF API كاملة
Documentation/driver-api/gpio/index.rst   ← الـ GPIO subsystem
Documentation/driver-api/gpio/consumer.rst ← إزاي تستهلك GPIO في driver
Documentation/driver-api/gpio/drivers-on-gpio.rst ← drivers بتستخدم GPIO
Documentation/devicetree/bindings/clock/gpio-gate-clock.yaml  ← DT binding للـ gate
Documentation/devicetree/bindings/clock/gpio-mux-clock.yaml   ← DT binding للـ mux
Documentation/devicetree/bindings/clock/gated-fixed-clock.yaml ← DT binding للـ gated-fixed
```

الـ binding القديمة بالـ `.txt` format كانت على:
- [`Documentation/devicetree/bindings/clock/gpio-gate-clock.txt`](https://mjmwired.net/kernel/Documentation/devicetree/bindings/clock/gpio-gate-clock.txt)

---

### الـ Kernel Commits المهمة

| الـ Commit | الوصف |
|-----------|-------|
| [clk: add gpio gated clock — Patchwork](https://patchwork.kernel.org/project/linux-omap/patch/1409919694-26941-1-git-send-email-jsarha@ti.com/) | الـ patch الأولانية اللي أدخلت `gpio-gate-clock` من Jyri Sarha (TI) سنة 2014 |
| [clk: add gpio controlled clock — alsa-devel Patchwork](https://patchwork.kernel.org/project/alsa-devel/patch/bb168f7d8ee4937adbd1f57dc4b1df9c40daf52c.1407332062.git.jsarha@ti.com/) | الـ patch series الأوسع اللي ضمّت الـ gpio clock في سياق الـ audio subsystem |
| [clk: clk-gpio: update documentation for gpio-gate clock](https://patchwork.kernel.org/project/linux-clk/patch/20240906082511.2963890-3-heiko@sntech.de/) | تحديث الـ documentation سنة 2024 من Heiko Stuebner |
| [PATCH v4 4/5: clk: clk-gpio: add driver for gated-fixed-clocks](https://www.spinics.net/lists/linux-clk/msg103842.html) | الـ patch اللي أضاف `gated-fixed-clock` لـ `clk-gpio.c` |

**الـ source على GitHub:**
- [`torvalds/linux: drivers/clk/clk-gpio.c`](https://github.com/torvalds/linux/blob/master/drivers/clk/clk-gpio.c)

---

### نقاشات الـ Mailing List

| المصدر | الوصف |
|--------|-------|
| [PATCH RFC v1: clk: add gpio controlled clock multiplexer](https://fa.linux.kernel.narkive.com/lprNrrhP/patch-rfc-v1-clk-add-gpio-controlled-clock-multiplexer) | النقاش الأول على الـ linux-kernel list حول الـ gpio-mux-clock |
| [PATCH: make gpio-gated clock support optional](https://lists.archive.carbon60.com/linux/kernel/3542721) | نقاش تحويل الـ driver لـ optional Kconfig |
| [Re: PATCH v3 2/5: clk-gpio documentation update — devicetree list](https://www.spinics.net/lists/devicetree/msg730203.html) | review discussion على الـ devicetree mailing list |
| [PATCH v2 1/1: clk: clk-gpio: add actual gated clock — u-boot list](https://lists.denx.de/pipermail/u-boot/2023-December/540743.html) | نقاش حول port مشابه في U-Boot |

---

### الكتب الموصى بيها

#### Linux Device Drivers, 3rd Edition (LDD3)
- **Chapter 1**: تعريف الـ kernel modules وكيفية عملها
- **Chapter 14**: الـ Linux Device Model (buses, devices, drivers) — أساسي لفهم `platform_driver`
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 18**: الـ Debugging — مفيد لفهم `dev_err_probe`
- **Chapter 17**: الـ Device Model وكيفية ربط الـ `platform_device` بالـ `platform_driver`

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **Chapter 15**: الـ Linux Clock and Timer Infrastructure
- بيشرح إزاي الـ embedded SoCs بتتعامل مع الـ clock trees وأهمية الـ GPIO في control الـ clocks

---

### مصادر eLinux.org

| المصدر | الوصف |
|--------|-------|
| [Generic Linux CLK Framework — CELF 2009 (PDF)](https://elinux.org/images/6/64/ELC_E_2009_Generic_Clock_Framework.pdf) | عرض تقديمي من مؤتمر ELC أوروبا 2009 عن الـ Generic Clock Framework |
| [Linux clock management framework — ELC 2007 (PDF)](https://elinux.org/images/a/a3/ELC_2007_Linux_clock_fmw.pdf) | العرض الأول من 2007 اللي رسم الأسس لإدارة الـ clocks |
| [Common Clock Framework BoF Notes (PDF)](https://elinux.org/images/b/b1/Common_Clock_Framework_(BoFs).pdf) | نقاشات الـ Birds of a Feather session حول الـ CCF |

---

### kernelnewbies.org

الـ CCF اتضاف في **Linux 3.4** — الصفحة المقابلة:
- [Linux 3.4 — kernelnewbies.org](https://kernelnewbies.org/Linux_3.4) ← بيذكر دمج الـ Common Clock Framework

للتغييرات الأحدث على `clk-gpio`:
- [Linux 6.14 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.14)
- [Linux 6.12 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.12)

---

### الـ Kernel Documentation Online

```
https://docs.kernel.org/driver-api/clk.html
https://docs.kernel.org/driver-api/gpio/index.html
https://docs.kernel.org/driver-api/gpio/consumer.html
https://docs.kernel.org/driver-api/gpio/drivers-on-gpio.html
```

**الـ Common Clock Framework API الرسمية:**
- [`kernel.org/doc/Documentation/clk.txt`](https://www.kernel.org/doc/Documentation/clk.txt)

---

### مصادر Bootlin

| المصدر | الوصف |
|--------|-------|
| [Common Clock Framework — ELCE 2013 (PDF)](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf) | شرح عملي شامل من Bootlin عن كيفية استخدام الـ CCF في embedded systems |

---

### Search Terms للبحث عن معلومات أكثر

```
# للبحث في الـ kernel source
git log --oneline -- drivers/clk/clk-gpio.c
git log --oneline -- Documentation/devicetree/bindings/clock/gpio-gate-clock*

# على الـ mailing lists
lore.kernel.org/linux-clk/?q=clk-gpio
lore.kernel.org/linux-clk/?q=gpio-gate-clock
lore.kernel.org/linux-clk/?q=gated-fixed-clock

# بحث عام
"gpio-gate-clock" linux kernel site:lore.kernel.org
"clk_gpio_gate" linux driver site:elixir.bootlin.com
"gated-fixed-clock" device tree binding linux
"clk_sleeping_gpio" linux kernel site:github.com

# على Elixir (Cross-Reference)
https://elixir.bootlin.com/linux/latest/source/drivers/clk/clk-gpio.c
https://elixir.bootlin.com/linux/latest/ident/clk_gpio_gate_ops
https://elixir.bootlin.com/linux/latest/ident/clk_gated_fixed_ops
```
## Phase 8: Writing simple module

### الفكرة

**الـ `clk_gpio_gate_enable`** هي الدالة اللي بتتنادى كل ما consumer طلب تفعيل clock مربوطة بـ GPIO. هنعمل kprobe عليها عشان نطبع اسم الـ clock وقيمة الـ GPIO اللي اتشغل.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * gpio_clk_probe_mod.c
 *
 * Attach a kprobe to clk_gpio_gate_enable() and log the clock name
 * plus the GPIO descriptor every time a GPIO-gated clock is enabled.
 */

/* --- Includes --- */
#include <linux/module.h>        /* MODULE_* macros, module_init/exit        */
#include <linux/kernel.h>        /* pr_info / pr_err                         */
#include <linux/kprobes.h>       /* struct kprobe, register_kprobe, ...      */
#include <linux/clk-provider.h>  /* struct clk_hw, clk_hw_get_name()         */
#include <linux/gpio/consumer.h> /* struct gpio_desc, gpiod_get_value()      */

/* ------------------------------------------------------------------ */
/* Re-declare the driver's internal struct so we can dereference it.  */
/* Must match drivers/clk/clk-gpio.c exactly.                         */
/* ------------------------------------------------------------------ */
struct clk_gpio {
    struct clk_hw    hw;        /* must be first — container_of relies on it */
    struct gpio_desc *gpiod;
};

#define to_clk_gpio(_hw) container_of(_hw, struct clk_gpio, hw)

/* ------------------------------------------------------------------ */
/* kprobe pre-handler: called just before clk_gpio_gate_enable runs   */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * clk_gpio_gate_enable(struct clk_hw *hw) — first argument.
     * On x86-64: first arg is in RDI. On ARM64: in X0.
     * kernel_regs_get_arg1() is a portable helper that fetches
     * the first function argument from the saved registers.
     */
#if defined(CONFIG_X86_64)
    struct clk_hw  *hw  = (struct clk_hw *)regs->di;
#elif defined(CONFIG_ARM64)
    struct clk_hw  *hw  = (struct clk_hw *)regs->regs[0];
#else
    /* Fallback: treat first saved reg as the argument */
    struct clk_hw  *hw  = (struct clk_hw *)regs->ARM_r0;
#endif

    struct clk_gpio *clk;
    const char      *name;
    int              gpio_val;

    if (!hw)
        return 0;

    clk      = to_clk_gpio(hw);
    name     = clk_hw_get_name(hw);      /* get registered clock name     */
    gpio_val = gpiod_get_value(clk->gpiod); /* current GPIO level (before set) */

    pr_info("gpio-clk-probe: enabling clock '%s', GPIO currently=%d\n",
            name ? name : "(unknown)", gpio_val);

    return 0; /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "clk_gpio_gate_enable", /* target function by name    */
    .pre_handler = handler_pre,            /* our callback               */
};

/* ------------------------------------------------------------------ */
/* Module init / exit                                                  */
/* ------------------------------------------------------------------ */
static int __init gpio_clk_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("gpio-clk-probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("gpio-clk-probe: kprobe planted on %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

static void __exit gpio_clk_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("gpio-clk-probe: kprobe removed\n");
}

module_init(gpio_clk_probe_init);
module_exit(gpio_clk_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe on clk_gpio_gate_enable to trace GPIO-gated clocks");
```

---

### Makefile للتجميع

```makefile
obj-m += gpio_clk_probe_mod.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
make
sudo insmod gpio_clk_probe_mod.ko
# trigger a clock enable from any driver using gpio-gate-clock
dmesg | grep gpio-clk-probe
sudo rmmod gpio_clk_probe_mod
```

---

### شرح كل جزء

#### الـ `#include`s

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيوفر `struct kprobe` وكل API الـ kprobe |
| `linux/clk-provider.h` | بيعرّف `struct clk_hw` و`clk_hw_get_name()` |
| `linux/gpio/consumer.h` | بيعرّف `struct gpio_desc` و`gpiod_get_value()` |

#### إعادة تعريف `struct clk_gpio`

الـ struct ده **static داخل الـ driver** — مش exported. علشان نقدر نوصل لـ `gpiod` من الـ kprobe، بنعيد تعريفه بنفس الترتيب تماماً عشان `container_of` يشتغل صح. لو الـ kernel اتغير وترتيب الـ fields اتغير، المفيش compile error لكن القراءة هتبقى غلط — لازم تتحقق.

#### الـ `handler_pre`

بيتنادى **قبل** تنفيذ `clk_gpio_gate_enable`. بياخد الـ argument الأول (الـ `struct clk_hw *`) من الـ registers بشكل architecture-specific، بعدين بيطبع اسم الـ clock والحالة الحالية للـ GPIO قبل ما تتغير — ده بيفيد في التشخيص.

#### `gpiod_get_value` في السياق ده

**الدالة دي safe** في context الـ kprobe لأن `clk_gpio_gate_enable` بتتنادى مع non-sleeping GPIO بس (الـ sleeping GPIO بتاخد المسار عن طريق `clk_sleeping_gpio_gate_prepare` مش `clk_gpio_gate_enable`). لو كنا hookna دالة sleeping، كان لازم نستخدم `gpiod_get_value_cansleep` من سياق يسمح بالـ sleep.

#### `module_init` و`module_exit`

- **init**: بيسجّل الـ kprobe بالاسم؛ الـ kernel بيحوّل الاسم لعنوان في الـ kallsyms.
- **exit**: **لازم** نعمل `unregister_kprobe` في الـ exit عشان نتجنب crash لو الـ module اتزيل والـ kprobe لسا مسجلة — الـ kernel هيكمل ينفّذ الـ handler في memory اتحررت.
