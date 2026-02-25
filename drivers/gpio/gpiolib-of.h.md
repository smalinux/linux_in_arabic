## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **GPIO subsystem** في Linux kernel، وبالتحديد من الـ **OF (Open Firmware / Device Tree) GPIO layer** — اللي بيربط بين الـ GPIO hardware وبين الـ Device Tree اللي بيوصف الـ hardware للـ kernel.

---

### الصورة الكبيرة — قبل أي كود

#### المشكلة اللي بيحلها

تخيل إنك بتشتري board جديدة — مثلاً Raspberry Pi أو BeagleBone — فيها chip عليها 40 pin. كل pin ممكن يبقى GPIO (General Purpose Input/Output): يعني ممكن تحطه high أو low، أو تقرأ منه.

السؤال هو: **إزاي الـ kernel يعرف إن الـ LED مربوط بـ GPIO 17؟ وإن الـ button مربوط بـ GPIO 18؟**

في الزمن القديم كان الـ code بيتكتب hardcoded جوه الـ kernel لكل board. ده كان كارثة لأن كل board محتاجة kernel مختلف.

الحل الحديث هو **Device Tree (DT)** — ملف نصي بيوصف الـ hardware بالكامل. بيقول مثلاً:

```
leds {
    led-status {
        gpios = <&gpio1 17 GPIO_ACTIVE_LOW>;
    };
};
```

يعني: "الـ LED ده مربوط بـ gpio1 controller، الـ pin رقم 17، وهو active low."

الـ **OF (Open Firmware)** هو الـ standard اللي بيحدد إزاي تكتب الـ Device Tree ده.

---

#### دور الـ `gpiolib-of.h`

الـ file ده هو **الـ internal header** اللي بيعرّف الـ interface بين جزئين:

1. **الـ `gpiolib.c`** — القلب الرئيسي للـ GPIO subsystem في الـ kernel.
2. **الـ `gpiolib-of.c`** — الجزء المسؤول عن قراءة وتفسير معلومات الـ GPIO من الـ Device Tree.

بمعنى آخر: الـ `gpiolib-of.h` بيقول للـ `gpiolib.c`:
> "أنا عندي وظائف بتقرأ الـ GPIO من الـ Device Tree، اطلبهم لما تحتاجهم."

---

### القصة كاملة — من الـ Device Tree للـ Hardware

**السيناريو:** driver بيطلب GPIO بالاسم "reset" لـ device معين.

```
                Driver يطلب gpiod_get(dev, "reset", ...)
                              |
                              v
               gpiolib.c يبحث عن الـ GPIO
                              |
                    هل الـ firmware type = OF?
                              |
                              v
              of_find_gpio() في gpiolib-of.c
                              |
                   يقرأ من Device Tree:
              reset-gpios = <&gpio2 5 GPIO_ACTIVE_HIGH>
                              |
                              v
              يرجع gpio_desc* للـ PIN رقم 5 في gpio2
                              |
                              v
                   Driver يستخدم الـ GPIO
```

---

### وظائف الـ `gpiolib-of.h`

| الوظيفة | الدور |
|---|---|
| `of_find_gpio()` | تبحث في الـ Device Tree عن GPIO بالاسم والـ index وترجع `gpio_desc*` |
| `of_gpiochip_add()` | لما GPIO controller جديد يتسجل، يربطه بالـ Device Tree node بتاعه |
| `of_gpiochip_remove()` | لما GPIO controller يتشال من النظام |
| `of_gpiochip_instance_match()` | بتتحقق إن الـ gpio_chip ده هو الـ instance رقم كذا في الـ DT |
| `of_gpio_count()` | بتعد كام GPIO اتعرف لـ device معين في الـ DT |
| `gpio_of_notifier` | notifier block لاستقبال events لما OF nodes تتضاف/تتشال (hotplug) |

---

### الـ `#ifdef CONFIG_OF_GPIO`

الـ file بيستخدم **compile-time guard** مهم:

- لو الـ kernel اتبنى **مع** دعم الـ Device Tree GPIO (`CONFIG_OF_GPIO=y`): الوظائف الحقيقية بتتحمل.
- لو اتبنى **بدون** دعم الـ Device Tree (مثلاً x86 server مش محتاج DT): كل الوظائف بتبقى **stubs** بترجع errors أو قيم فاضية — من غير ما تعمل compile error.

ده بيخلي الـ code النظيف يشتغل على **كل** الـ architectures.

---

### الفرق بين الـ public headers والـ internal header

| الـ Header | موقعه | مين يستخدمه |
|---|---|---|
| `include/linux/of_gpio.h` | public | أي driver خارجي |
| `include/linux/gpio/consumer.h` | public | أي driver يطلب GPIO |
| `drivers/gpio/gpiolib-of.h` | **internal** | `gpiolib.c` فقط داخل الـ subsystem |

الـ `gpiolib-of.h` ده **مش للـ drivers العادية** — ده للاستخدام الداخلي بين ملفات الـ gpiolib نفسها.

---

### الملفات المرتبطة — اللي لازم تعرفها

#### Core GPIO Subsystem

| الملف | الدور |
|---|---|
| `drivers/gpio/gpiolib.c` | القلب — كل منطق الـ GPIO العام |
| `drivers/gpio/gpiolib.h` | internal structs مثل `gpio_device`, `gpio_desc` |
| `drivers/gpio/gpiolib-of.c` | implementation الوظائف اللي header-ha هو `gpiolib-of.h` |
| `drivers/gpio/gpiolib-of.h` | **الملف ده** — declarations للـ OF layer |

#### الـ Firmware Abstraction Siblings

| الملف | الدور |
|---|---|
| `drivers/gpio/gpiolib-acpi.h` + `.c` | نفس الدور بس لـ ACPI (x86/ARM64 UEFI) |
| `drivers/gpio/gpiolib-swnode.h` + `.c` | نفس الدور بس لـ software nodes |

#### الـ Public Headers

| الملف | الدور |
|---|---|
| `include/linux/of_gpio.h` | public API للـ drivers القديمة |
| `include/linux/gpio/consumer.h` | الـ modern API للـ drivers |
| `include/linux/gpio/driver.h` | API لـ GPIO controller drivers |
| `include/linux/of.h` | الـ Device Tree core API |

#### الـ Utility Files

| الملف | الدور |
|---|---|
| `drivers/gpio/gpiolib-devres.c` | managed resources (devm_*) للـ GPIO |
| `drivers/gpio/gpiolib-cdev.c` | character device interface (`/dev/gpiochipN`) |
| `drivers/gpio/gpiolib-sysfs.c` | sysfs interface |
| `drivers/gpio/gpiolib-legacy.c` | الـ deprecated integer-based GPIO API |
## Phase 2: شرح الـ GPIO OF Framework

---

### المشكلة — ليه الـ subsystem ده موجود؟

في embedded Linux، أي driver محتاج يتحكم في GPIO لازم يعرف:
1. **أنهي GPIO controller** اللي موصل بيه الـ pin
2. **رقم الـ line** جوه الـ controller ده
3. **الـ flags** — هل active-low؟ open-drain؟ pull-up؟

قبل ما يتوحد الأمر، كل driver كان بيعمل اللي يناسبه: hardcoded numbers، platform data، أو magic macros. النتيجة؟ code مش محمول، و GPIO controller driver مش لازمه يتغير لو غيرت الـ board.

الـ **Device Tree** جه عشان يحل مشكلة الـ board description بشكل declarative — لكن ده خلق مشكلة تانية: **إزاي الـ kernel يحول نص زي `<&gpio1 5 GPIO_ACTIVE_LOW>` لـ `struct gpio_desc` فعلي؟**

ده بالظبط اللي بيحله الـ **GPIO OF Framework** (`gpiolib-of`).

---

### الحل — المنهج اللي اتبعه الـ kernel

الـ framework ده بيعمل **جسر** بين طريقتين لوصف الـ hardware:

- **الـ Device Tree side**: properties زي `reset-gpios = <&gpio2 17 1>;`
- **الـ gpiolib side**: `struct gpio_desc` اللي الـ drivers بتشتغل بيها

الجسر ده بيشتغل في اتجاهين:

| الاتجاه | العملية |
|---------|---------|
| **Chip registration** | لما GPIO controller بيعمل `gpiochip_add()` → `of_gpiochip_add()` يربط الـ chip بـ device node بتاعه في الـ DT |
| **GPIO lookup** | لما driver بيطلب GPIO باسمه → `of_find_gpio()` يبحث في الـ DT node ويرجع `gpio_desc*` |

---

### التشبيه الواقعي — مكتبة وطلبات الكتب

تخيل إن الـ kernel هو **مكتبة ضخمة** فيها آلاف الكتب (GPIOs).

- **الـ Device Tree** هو **كتالوج المكتبة** — فيه: "كتاب X موجود في رف GPIO2، خانة 17، وده كتاب نادر (active-low)."
- **الـ gpio_chip** هو **الرف الفعلي** — الموظف المسؤول عن رف معين.
- **الـ `#gpio-cells`** هو **نظام الترميز** — عدد الأرقام اللازمة لتحديد كتاب (رقمين: رقم الكتاب + الـ flags).
- **الـ `of_xlate`** هو **الموظف المترجم** — بيفهم الكود ويجيب الكتاب الصح من الرف.
- **الـ `of_find_gpio()`** هو **طلب الكتاب** — بتقوله: "عايز كتاب اسمه reset" فيدور في الكتالوج ويجيبهولك.
- **الـ `gpio_desc`** هو **الكتاب نفسه** اللي في إيدك بعد كده.
- **الـ GPIO hog** هو **كتاب محجوز دايمًا** من أول ما المكتبة تفتح — مش محتاج حد يطلبه.

**التعمق في التشبيه:**
- لما المكتبة بتتبني (boot time)، كل موظف (chip driver) بيسجل رفوفه (`of_gpiochip_add`) — بيقول: "أنا مسؤول عن الرف ده وعندي نظام ترميزه."
- لما driver يطلب `reset-gpios`، المكتبة تدور في الكتالوج (DT)، تلاقي phandle يشير لرف معين، تصحى الموظف المسؤول، يترجم الأرقام، ويديك الـ `gpio_desc`.
- لو في كتاب اتحجز بغلط (DTS خاطئ في الـ polarity)، عندنا **قسم التصحيح** (`of_gpio_flags_quirks`) اللي بيصحح الأخطاء التاريخية تلقائيًا.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────┐
  │                     CONSUMER DRIVERS                    │
  │   (reset-driver, regulator, SPI, MMC, LED trigger...)   │
  │                                                         │
  │     gpiod_get(dev, "reset", GPIOD_OUT_LOW)              │
  └────────────────────────┬────────────────────────────────┘
                           │
                           ▼
  ┌─────────────────────────────────────────────────────────┐
  │                  GPIOLIB CORE (gpiolib.c)               │
  │   gpiod_find_and_request()                              │
  │        │                                                │
  │        ├──► [Machine lookup table?]  ──► gpiod_machine  │
  │        │                                                │
  │        └──► [fwnode lookup]                             │
  │                  │                                      │
  │                  ▼                                      │
  │          gpiod_find_by_fwnode()                         │
  └────────────────────┬────────────────────────────────────┘
                       │ calls
                       ▼
  ┌─────────────────────────────────────────────────────────┐
  │             GPIO OF FRAMEWORK  (gpiolib-of.c/h)         │
  │                                                         │
  │   of_find_gpio(np, "reset", idx, &flags)                │
  │       │                                                 │
  │       ├─1─► try "reset-gpios" / "reset-gpio"           │
  │       │         of_get_named_gpiod_flags()              │
  │       │              │                                  │
  │       │              ├─► of_parse_phandle_with_args_map │
  │       │              │       (reads DT: &gpio2 17 1)    │
  │       │              │                                  │
  │       │              └─► of_find_gpio_device_by_xlate() │
  │       │                      │                          │
  │       │                      ▼                          │
  │       │              chip->of_xlate()  ◄──────────────┐ │
  │       │         (of_gpio_twocell_xlate /              │ │
  │       │          of_gpio_threecell_xlate /            │ │
  │       │          custom driver xlate)                 │ │
  │       │                                               │ │
  │       ├─2─► of_find_gpio_quirks[]                     │ │
  │       │       (rename table, polarity fixups)         │ │
  │       │                                               │ │
  │       └──► returns gpio_desc* + lookup flags          │ │
  │                                                       │ │
  │   of_gpiochip_add(chip) ──────────────────────────────┘ │
  │       ├─► assign of_xlate callback                       │
  │       ├─► of_gpiochip_add_pin_range() [pinctrl bridge]   │
  │       └─► of_gpiochip_scan_gpios()   [GPIO hogs]         │
  └───────────────────────────┬─────────────────────────────┘
                              │
             ┌────────────────┴──────────────────┐
             ▼                                   ▼
  ┌─────────────────────┐          ┌─────────────────────────┐
  │  gpio_chip / device │          │   Device Tree (DT)      │
  │  (Hardware Driver)  │          │                         │
  │                     │          │  gpio2: gpio@... {      │
  │  .of_xlate          │          │    #gpio-cells = <2>;   │
  │  .of_gpio_n_cells   │          │  };                     │
  │  .ngpio             │          │                         │
  │  .of_node_instance_ │          │  reset-gpios =          │
  │    match            │          │    <&gpio2 17 1>;       │
  └─────────────────────┘          └─────────────────────────┘
```

---

### الـ Core Abstractions

#### 1. الـ GPIO Specifier — لغة الـ DT لوصف GPIO

في الـ Device Tree، أي GPIO بيتكتب كـ **phandle + cells**:

```dts
/* consumer node */
reset-gpios = <&gpio2  17  1>;
/*             phandle  pin  flags */
/*             ^         ^    ^
/*         gpio_chip    line  OF_GPIO_ACTIVE_LOW */
```

الـ `#gpio-cells = <2>` في الـ gpio_chip node هو اللي بيحدد إن كل specifier هو cell واحد ولا اتنين ولا تلاتة.

الـ kernel بيحول ده لـ `struct of_phandle_args`:
```c
struct of_phandle_args {
    struct device_node *np;   /* يشير لـ gpio_chip node */
    int args_count;           /* = #gpio-cells = 2 */
    uint32_t args[MAX_PHANDLE_ARGS]; /* args[0]=17, args[1]=1 */
};
```

#### 2. الـ `of_xlate` callback — مترجم الـ specifier

كل `gpio_chip` لازم يقدر يحول `of_phandle_args` لـ GPIO line number. الـ framework بيوفر default implementations:

```c
/* الأكثر شيوعًا: 2 cells = [pin, flags] */
static int of_gpio_twocell_xlate(struct gpio_chip *gc,
                                 const struct of_phandle_args *gpiospec,
                                 u32 *flags)
{
    if (gpiospec->args[0] >= gc->ngpio)
        return -EINVAL;

    if (flags)
        *flags = gpiospec->args[1]; /* OF_GPIO_ACTIVE_LOW etc. */

    return gpiospec->args[0];       /* GPIO line number */
}

/* لما في أكتر من chip في نفس الـ DT node: 3 cells = [instance, pin, flags] */
static int of_gpio_threecell_xlate(struct gpio_chip *gc,
                                   const struct of_phandle_args *gpiospec,
                                   u32 *flags)
{
    /* args[0] = instance number -> of_node_instance_match() */
    /* args[1] = pin offset      */
    /* args[2] = flags           */
    if (!gc->of_node_instance_match(gc, gpiospec->args[0]))
        return -EINVAL;
    if (flags)
        *flags = gpiospec->args[2];
    return gpiospec->args[1];
}
```

لو الـ driver محتاج منطق خاص — زي decode لـ multiplexed banks — بيكتب `of_xlate` خاص بيه وبيحطه في الـ `gpio_chip` struct قبل `gpiochip_add()`.

#### 3. الـ `of_gpio_flags` — الـ flags اللي بيطلع منها الـ lookup flags

```c
/* داخل gpiolib-of.c */
enum of_gpio_flags {
    OF_GPIO_ACTIVE_LOW    = 0x1,  /* السيجنال inverted */
    OF_GPIO_SINGLE_ENDED  = 0x2,  /* open-drain أو open-source */
    OF_GPIO_OPEN_DRAIN    = 0x4,  /* فعال بس مع SINGLE_ENDED */
    OF_GPIO_TRANSITORY    = 0x8,  /* GPIO مؤقت — مش لازم يتحفظ */
    OF_GPIO_PULL_UP       = 0x10,
    OF_GPIO_PULL_DOWN     = 0x20,
    OF_GPIO_PULL_DISABLE  = 0x40,
};
```

بعد الـ xlate، الـ `of_convert_gpio_flags()` بتحول الـ OF flags لـ generic `GPIO_*` flags اللي بيفهمها الـ gpiolib:

```c
static unsigned long of_convert_gpio_flags(enum of_gpio_flags flags)
{
    unsigned long lflags = GPIO_LOOKUP_FLAGS_DEFAULT;

    if (flags & OF_GPIO_ACTIVE_LOW)
        lflags |= GPIO_ACTIVE_LOW;
    if (flags & OF_GPIO_SINGLE_ENDED) {
        if (flags & OF_GPIO_OPEN_DRAIN)
            lflags |= GPIO_OPEN_DRAIN;
        else
            lflags |= GPIO_OPEN_SOURCE;
    }
    /* ... */
    return lflags;
}
```

---

### رسم تدفق `of_find_gpio()` خطوة بخطوة

```
of_find_gpio(np="eth@...", con_id="phy-reset", idx=0, *flags)
│
├─► loop: try "phy-reset-gpios", then "phy-reset-gpio"
│       of_get_named_gpiod_flags(np, "phy-reset-gpios", 0, &of_flags)
│           │
│           ├─► of_parse_phandle_with_args_map(np, "phy-reset-gpios",
│           │       "gpio", 0, &gpiospec)
│           │   → DT: phy-reset-gpios = <&gpio3 14 0>
│           │   → gpiospec.np = &gpio3_node
│           │   → gpiospec.args = [14, 0]
│           │
│           ├─► of_find_gpio_device_by_xlate(&gpiospec)
│           │   → بيدور على chip اللي device_node بتاعه = &gpio3_node
│           │   → ولازم chip->of_xlate(chip, &gpiospec, NULL) >= 0
│           │
│           ├─► of_xlate_and_get_gpiod_flags(chip, &gpiospec, &of_flags)
│           │   → chip->of_xlate(chip, &gpiospec, &raw_flags)
│           │   → returns line_number = 14
│           │   → gpiochip_get_desc(chip, 14) → gpio_desc*
│           │
│           └─► of_gpio_flags_quirks(np, "phy-reset-gpios", &of_flags, 0)
│               → check polarity fixup tables
│               → لـ fsl,imx6q-fec + "phy-reset-gpios":
│                  اقرأ "phy-reset-active-high" property
│                  وعدّل OF_GPIO_ACTIVE_LOW accordingly
│
├─► of_convert_gpio_flags(of_flags) → *flags = GPIO_ACTIVE_LOW or 0
│
└─► return gpio_desc*
```

---

### الـ GPIO Hog — GPIO بيحجز نفسه من الـ DT

الـ **GPIO hog** هو mechanism يخلي الـ firmware يطلب GPIO line ويضبط اتجاهه/قيمته من أول ما الـ gpio_chip يتسجل، من غير ما يكون في driver تاني يطلبه.

مثال DTS:

```dts
gpio2: gpio@020b4000 {
    #gpio-cells = <2>;
    /* ... */

    usb-hub-reset {
        gpio-hog;
        gpios = <14 0>;        /* line 14, active-high */
        output-low;
        line-name = "usb-hub-reset";
    };
};
```

لما `of_gpiochip_add()` بيتنادى، بيعمل `of_gpiochip_scan_gpios()` اللي بيدور على كل child node فيها `gpio-hog` property ويعمل `gpiod_hog()` عليها.

```c
static int of_gpiochip_scan_gpios(struct gpio_chip *chip)
{
    for_each_available_child_of_node_scoped(dev_of_node(&chip->gpiodev->dev), np) {
        if (!of_property_read_bool(np, "gpio-hog"))
            continue;
        ret = of_gpiochip_add_hog(chip, np);
    }
    return 0;
}
```

---

### الـ Quirks Layer — طبقة التوافق مع الـ Legacy DTS

ده من أكتر الحاجات اللي بتتميز بيها الـ gpiolib-of — عنده **ثلاث طبقات quirks**:

#### أ) إعادة التسمية (`of_find_gpio_rename`)

بعض الـ drivers القديمة كانت بتستخدم أسماء غير standard للـ GPIO properties:

```c
/* مثلاً: Himax LCD استخدم "gpios-reset" بدل "reset-gpios" */
{ "reset", "gpios-reset", "himax,hx8357" },

/* SPI GPIO driver استخدم "gpio-miso" بدل "miso-gpios" */
{ "miso", "gpio-miso", "spi-gpio" },
```

الـ framework بيجرب الاسم الجديد الصح الأول، ولو مش لاقيه بيجرب الـ legacy name بدل ما يفشل.

#### ب) تصحيح الـ polarity (`of_gpio_try_fixup_polarity`)

بعض الـ DTS القديمة كتبت الـ polarity غلط. الكود عنده table ثابتة:

```c
{ "cascoda,ca8210", "reset-gpio",  false }, /* active-low هو الصح */
{ "qi,lb60",        "rb-gpios",    true  }, /* active-high هو الصح */
```

#### ج) الـ polarity من property تانية (`of_gpio_set_polarity_by_property`)

بعض الـ bindings القديمة بتحدد الـ polarity في property منفصلة:

```dts
/* بدل ما تحط flags في الـ GPIO specifier */
phy-reset-gpios = <&gpio1 5 0>;
phy-reset-active-high;   /* property منفصلة بتحدد الـ polarity */
```

```c
{ "fsl,imx6q-fec", "phy-reset-gpios", "phy-reset-active-high" },
```

---

### الـ Pinctrl Bridge — `gpio-ranges`

ده مفهوم مهم جداً: في أغلب الـ SoCs، الـ GPIO lines هي نفسها الـ pins بتاعة الـ pinctrl subsystem.

> **الـ Pinctrl Subsystem**: بيتحكم في multiplexing الـ pins (GPIO vs UART vs SPI) وفي الـ pull resistors وغيرها. الـ GPIO subsystem بيحتاج يعرف أي GPIO line بيقابل أي pin رقم كام في الـ pinctrl.

```dts
gpio0: gpio@44e07000 {
    #gpio-cells = <2>;
    gpio-ranges = <&am33xx_pinmux 0 0 32>;
    /*              ^pinctrl_phandle  ^gpio_offset  ^pin_offset  ^count */
};
```

الـ `of_gpiochip_add_pin_range()` بيقرأ الـ `gpio-ranges` property ويعمل `gpiochip_add_pin_range()` عشان الـ gpiolib يعرف يتكلم مع الـ pinctrl لما تطلب `gpio_request()`.

---

### الـ Dynamic DT — `gpio_of_notifier`

لما `CONFIG_OF_DYNAMIC` شغال (بيسمح بتغيير الـ DT في runtime — مستخدم كتير في FPGA overlays وبعض الـ hypervisors):

```c
struct notifier_block gpio_of_notifier = {
    .notifier_call = of_gpio_notify,
};
```

الـ notifier ده بيتسجل على الـ OF reconfig chain. لما node فيها `gpio-hog` بتتضاف أو تتشال من الـ DT في runtime، الـ `of_gpio_notify()` بيعمل الحجز أو الإلغاء تلقائيًا.

---

### إيه اللي الـ Framework بيمتلكه vs. إيه اللي بيفوّضه للـ Drivers

| المسؤولية | مين بيعملها |
|-----------|-------------|
| قراءة الـ DT property وتحليل الـ phandle | **OF Framework** (`of_parse_phandle_with_args_map`) |
| ربط الـ gpio_chip بـ DT node بتاعه | **OF Framework** (`of_gpiochip_add`) |
| تحديد عدد الـ cells (`#gpio-cells`) | **GPIO Controller Driver** |
| تحويل args → line number | **Driver** (أو default `of_gpio_twocell_xlate`) |
| تحويل args → `of_gpio_flags` | **Driver** (جزء من الـ xlate) |
| تحويل `of_gpio_flags` → generic `GPIO_*` flags | **OF Framework** (`of_convert_gpio_flags`) |
| معالجة الـ legacy DTS quirks | **OF Framework** (quirk tables) |
| ربط GPIO lines بـ pinctrl pins | **OF Framework** (`of_gpiochip_add_pin_range`) |
| الحجز الفعلي للـ GPIO ووضع الـ direction | **gpiolib core** (`gpiod_hog`, `gpiod_direction_*`) |
| IRQ domain للـ GPIO-as-interrupt | **GPIO Controller Driver** + `gpio_irq_chip` |

---

### الـ Struct Relationships

```
struct gpio_chip
├── .of_xlate          ──────────────────► of_gpio_twocell_xlate()
│                                          of_gpio_threecell_xlate()
│                                          or custom driver function
│
├── .of_gpio_n_cells   ──── عدد cells في DT specifier (2 or 3)
│
├── .of_node_instance_match ── callback لما في أكتر من chip في node واحد
│
└── .gpiodev ─────────────► struct gpio_device
                                 └── .dev ──► struct device
                                                  └── .of_node ──► device_node
                                                                    │
                                                                    └── DT props:
                                                                        #gpio-cells
                                                                        gpio-ranges
                                                                        gpio-hog children

struct of_phandle_args         ← نتيجة parse الـ DT property
├── .np        ─────────────► device_node of gpio_chip
├── .args_count ────────────► = #gpio-cells value
└── .args[]    ─────────────► [pin_number, flags]
        │                     or [instance, pin, flags]
        │
        ▼ (passed to of_xlate)
        returns: line_number + *flags (as of_gpio_flags)
        │
        ▼ (gpiochip_get_desc)
struct gpio_desc   ◄──── الـ abstraction النهائي اللي يشتغل بيها الـ consumer
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options

#### `enum of_gpio_flags` — flags الـ GPIO في الـ Device Tree

| القيمة | الـ Hex | المعنى |
|--------|---------|--------|
| `OF_GPIO_ACTIVE_LOW` | `0x1` | الـ GPIO يشتغل لما يكون low (مقلوب) |
| `OF_GPIO_SINGLE_ENDED` | `0x2` | single-ended output — مع flag تاني بيتحدد نوعه |
| `OF_GPIO_OPEN_DRAIN` | `0x4` | open drain — مع `SINGLE_ENDED` يديك `GPIO_OPEN_DRAIN` |
| `OF_GPIO_TRANSITORY` | `0x8` | حالته بتتغير مؤقتاً (مثلاً reset pulse) |
| `OF_GPIO_PULL_UP` | `0x10` | pull-up مفعّل على الـ pin |
| `OF_GPIO_PULL_DOWN` | `0x20` | pull-down مفعّل على الـ pin |
| `OF_GPIO_PULL_DISABLE` | `0x40` | pull مقفول خالص |

> الـ flags دي بتيجي من الـ DTS وبتتحوّل لـ `GPIO_*` flags بالـ `of_convert_gpio_flags()`.

#### Mapping: `of_gpio_flags` → `gpio_lookup_flags`

| OF Flag | GPIO Flag الناتجة |
|---------|------------------|
| `OF_GPIO_ACTIVE_LOW` | `GPIO_ACTIVE_LOW` |
| `OF_GPIO_SINGLE_ENDED` + `OF_GPIO_OPEN_DRAIN` | `GPIO_OPEN_DRAIN` |
| `OF_GPIO_SINGLE_ENDED` بدون open drain | `GPIO_OPEN_SOURCE` |
| `OF_GPIO_TRANSITORY` | `GPIO_TRANSITORY` |
| `OF_GPIO_PULL_UP` | `GPIO_PULL_UP` |
| `OF_GPIO_PULL_DOWN` | `GPIO_PULL_DOWN` |
| `OF_GPIO_PULL_DISABLE` | `GPIO_PULL_DISABLE` |

#### Config Options المؤثرة في الملف

| Option | الأثر |
|--------|-------|
| `CONFIG_OF_GPIO` | بيفعّل كل الـ API — لو مش مفعّل كل الدوال بترجع stub |
| `CONFIG_OF_DYNAMIC` | بيفعّل الـ `gpio_of_notifier` ودعم hot-plug الـ hogs |
| `CONFIG_PINCTRL` | بيفعّل `of_gpiochip_add_pin_range()` |
| `CONFIG_SPI_MASTER` | بيفعّل quirks خاصة بالـ SPI chip select |
| `CONFIG_REGULATOR` | بيفعّل quirk الـ open-drain للـ `reg-fixed-voltage` |
| `CONFIG_LEDS_TRIGGER_GPIO` | بيفعّل `of_find_trigger_gpio()` |

---

### الـ Structs المهمة

#### 1. `struct notifier_block` — موجود في `include/linux/notifier.h`

**الغرض:** الـ building block لنظام الـ notification في الكيرنل. الـ `gpio_of_notifier` بيستخدمه لمتابعة تغييرات الـ Device Tree.

| الـ Field | النوع | الوصف |
|-----------|-------|-------|
| `notifier_call` | `notifier_fn_t` | الـ callback اللي بيتنادى لما يحصل event |
| `next` | `struct notifier_block __rcu *` | الـ linked list للـ notifiers (RCU-protected) |
| `priority` | `int` | أعلى رقم بيتنادى الأول |

**الاستخدام في الملف:**
```c
// gpiolib-of.c السطر 950
struct notifier_block gpio_of_notifier = {
    .notifier_call = of_gpio_notify,  /* يتنادى لما يتضاف/يتشال hog node */
};
```

#### 2. `struct of_phandle_args` — مؤقت داخل الدوال

**الغرض:** بيحمل نتيجة parsing الـ phandle من الـ DTS — يعني مين الـ GPIO controller وإيه الـ args.

| الـ Field | الوصف |
|-----------|-------|
| `np` | الـ `device_node` بتاع الـ GPIO controller |
| `args_count` | عدد الـ cells في الـ specifier |
| `args[]` | الـ cells نفسها (رقم الـ GPIO، الـ flags، وغيره) |

**مثال DTS:**
```
reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
/* np → gpio1's node, args[0]=5 (pin), args[1]=1 (active_low flag) */
```

#### 3. `struct gpio_chip` — معرَّف في `include/linux/gpio/driver.h`

**الغرض:** يمثّل GPIO controller واحد. الـ OF layer بيضيف عليه قدرات الـ Device Tree.

الـ fields المهمة **من منظور gpiolib-of**:

| الـ Field | الوصف |
|-----------|-------|
| `of_xlate` | دالة بتترجم `of_phandle_args` لـ GPIO number + flags |
| `of_gpio_n_cells` | عدد الـ cells في الـ DTS specifier (2 أو 3) |
| `of_node_instance_match` | للـ chips اللي فيها أكتر من instance في node واحد |
| `gpiodev` | pointer لـ `gpio_device` — البيانات الداخلية |
| `ngpio` | عدد الـ GPIOs في الـ chip |
| `offset` | أول رقم GPIO في الـ chip (لو في أكتر من chip في node) |

#### 4. `struct gpio_desc` — معرَّف في gpiolib داخلياً

**الغرض:** يمثّل GPIO line واحد. كل العمليات على GPIO بتتم من خلاله.

الـ fields المهمة **من منظور gpiolib-of**:

| الـ Field | الوصف |
|-----------|-------|
| `hog` | (في `CONFIG_OF_DYNAMIC`) pointer لـ `device_node` الـ hog |
| `flags` | بيتضمن `GPIOD_FLAG_IS_HOGGED` لو هو hog |

#### 5. `struct gpio_device` — wrapper للـ `gpio_chip`

**الغرض:** الـ reference-counted object اللي بيحتوي الـ `gpio_chip`. الـ OF code بيتعامل معاه عبر `gpio_device_find()` و `gpio_device_get_chip()`.

#### 6. `struct of_rename_gpio` — static داخل `of_find_gpio_rename()`

**الغرض:** جدول بيربط الـ legacy GPIO property names بالأسماء الجديدة الصح.

| الـ Field | الوصف |
|-----------|-------|
| `con_id` | الاسم الجديد الصح (مثلاً `"reset"`) |
| `legacy_id` | الاسم القديم في الـ DTS (مثلاً `"gpios-reset"`) |
| `compatible` | الـ compatible string للـ device اللي فيه الـ quirk |

#### 7. `of_find_gpio_quirk` — typedef للـ function pointer

```c
typedef struct gpio_desc *(*of_find_gpio_quirk)(
    struct device_node *np,
    const char *con_id,
    unsigned int idx,
    enum of_gpio_flags *of_flags
);
```

الـ array `of_find_gpio_quirks[]` بيحتوي قائمة من الـ fallback functions اللي بتتجرب واحدة واحدة لما ميلاقيش الـ GPIO باسمه الطبيعي.

---

### مخطط العلاقات بين الـ Structs

```
Device Tree (DTS)
      │
      │  of_parse_phandle_with_args_map()
      ▼
┌─────────────────────┐
│  of_phandle_args    │
│  ┌───────────────┐  │
│  │ np ────────────┼──────────────────────────┐
│  │ args_count    │  │                         │
│  │ args[0..N]    │  │                         ▼
│  └───────────────┘  │            ┌────────────────────┐
└─────────────────────┘            │   device_node      │
                                   │   (GPIO controller)│
      │                            └────────────────────┘
      │  of_find_gpio_device_by_xlate()          │
      │  → gpio_device_find()                    │
      ▼                                          │
┌─────────────────┐                              │
│  gpio_device    │◄─────────────────────────────┘
│  (ref-counted)  │   device_match_of_node()
│  ┌───────────┐  │
│  │ dev       │  │
│  └───────────┘  │
└────────┬────────┘
         │  gpio_device_get_chip()
         ▼
┌──────────────────────────┐
│      gpio_chip           │
│  ┌──────────────────┐    │
│  │ of_xlate ────────┼────┼──► of_gpio_twocell_xlate()
│  │ of_gpio_n_cells  │    │    of_gpio_threecell_xlate()
│  │ of_node_instance │    │
│  │ _match           │    │
│  │ ngpio            │    │
│  │ offset           │    │
│  │ gpiodev ─────────┼────┼──► gpio_device (back-pointer)
│  └──────────────────┘    │
└──────────────────────────┘
         │  gpiochip_get_desc(chip, pin_number)
         ▼
┌──────────────────────────┐
│      gpio_desc           │
│  ┌──────────────────┐    │
│  │ flags            │    │   ← GPIOD_FLAG_IS_HOGGED
│  │ hog (OF_DYNAMIC) │────┼──► device_node (hog node)
│  └──────────────────┘    │
└──────────────────────────┘
```

---

### مخطط دورة حياة الـ GPIO Chip في الـ OF Layer

```
┌─────────────────────────────────────────────────────────┐
│                    DRIVER PROBE                          │
│  (مثلاً: pl061_probe() في drivers/gpio/gpio-pl061.c)    │
└───────────────────┬─────────────────────────────────────┘
                    │
                    │  gpiochip_add_data()
                    ▼
┌─────────────────────────────────────────────────────────┐
│               gpiolib core                               │
│  بيعمل gpio_device, بيسجّل الـ chip                    │
└───────────────────┬─────────────────────────────────────┘
                    │
                    │  of_gpiochip_add(chip)   ← نقطة دخول gpiolib-of
                    ▼
         ┌──────────────────────┐
         │  هل chip->of_xlate   │
         │  = NULL?             │
         └──────┬───────────────┘
                │
      ┌─────────┴──────────┐
      │ YES                │ NO (driver وفّر xlate خاصته)
      ▼                    ▼
┌───────────────┐    ┌────────────────┐
│n_cells == 3?  │    │  استخدم xlate  │
└──────┬────────┘    │  الـ driver    │
       │             └───────┬────────┘
  ┌────┴────┐                │
  │YES  │NO │                │
  ▼     ▼   │                │
three two   │                │
cell  cell  │                │
xlate xlate │                │
  └────┬────┘                │
       └──────────┬──────────┘
                  ▼
    of_gpiochip_add_pin_range(chip)
    (ربط الـ GPIOs بالـ pinctrl ranges)
                  │
                  ▼
         of_node_get(np)
         (زيادة refcount الـ device_node)
                  │
                  ▼
    of_gpiochip_scan_gpios(chip)
    (scan أولاد الـ node عشان gpio-hog)
                  │
         ┌────────┴────────┐
         │ كل child node   │
         │ عنده gpio-hog?  │
         └────────┬────────┘
                  │ YES
                  ▼
    of_gpiochip_add_hog(chip, hog_node)
         loop على كل GPIO في الـ hog
         │
         ├─► of_parse_own_gpio() → gpio_desc
         ├─► gpiod_hog(desc, name, lflags, dflags)
         └─► WRITE_ONCE(desc->hog, hog_node)  [OF_DYNAMIC فقط]
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│                  CHIP READY                              │
│  الـ GPIO chip متسجّل ومربوط بالـ DT ومتعمل له hogs    │
└─────────────────────────────────────────────────────────┘


══════════ TEARDOWN ══════════

    of_gpiochip_remove(chip)
         │
         ▼
    of_node_put(np)
    (تخفيض refcount — لو وصل صفر يتشال)
```

---

### مخطط تدفق `of_find_gpio()` — GPIO Lookup من الـ DT

```
Consumer driver يطلب GPIO:
  gpiod_get(dev, "reset", GPIOD_OUT_LOW)
         │
         ▼ (عبر gpiolib core → OF path)
  of_find_gpio(np, "reset", 0, &flags)
         │
         │  Step 1: جرب الأسماء الطبيعية
         ├─► for_each_gpio_property_name(propname, "reset"):
         │     try "reset-gpios"    → of_get_named_gpiod_flags()
         │     try "reset-gpio"     → of_get_named_gpiod_flags()
         │         │
         │         ▼
         │   of_parse_phandle_with_args_map(np, propname, "gpio", ...)
         │         │
         │         ▼
         │   of_find_gpio_device_by_xlate(&gpiospec)
         │   → gpio_device_find() يدور على كل الـ chips
         │       → of_gpiochip_match_node_and_xlate()
         │           بيتأكد: نفس الـ node + of_xlate بترجع >= 0
         │         │
         │         ▼
         │   of_xlate_and_get_gpiod_flags(chip, &gpiospec, &flags)
         │   → chip->of_xlate(chip, gpiospec, flags)
         │       → يرجع pin number
         │   → gpiochip_get_desc(chip, pin_number)
         │         │
         │         ▼
         │   of_gpio_flags_quirks(np, propname, &flags, idx)
         │       ├─ of_gpio_try_fixup_polarity()   ← static polarity fixes
         │       └─ of_gpio_set_polarity_by_property() ← property-based fixes
         │
         │  Step 2: لو ما لقاش، جرب الـ quirks
         └─► for each quirk in of_find_gpio_quirks[]:
               ├─ of_find_gpio_rename()    ← legacy property names
               ├─ of_find_mt2701_gpio()    ← MT2701 specific [optional]
               └─ of_find_trigger_gpio()   ← "trigger-sources" special case
                        │
                        ▼
               of_get_named_gpiod_flags(np, legacy_id, idx, &flags)
                        │
                        ▼
                  gpio_desc  ← النتيجة النهائية
         │
         ▼
  of_convert_gpio_flags(of_flags)
  → يحوّل OF flags لـ GPIO_* flags
         │
         ▼
  يرجع gpio_desc للـ consumer
```

---

### مخطط تدفق الـ HOG في `CONFIG_OF_DYNAMIC`

```
OF notifier chain يستقبل تغيير
         │
         ▼
  of_gpio_notify(nb, action, rd)
         │
    ┌────┴─────────────────────────┐
    │ OF_RECONFIG_CHANGE_ADD       │
    │                              │
    │  هل rd->dn عنده "gpio-hog"? │
    │  ─── YES ───────────────────►│
    │                              │
    │  of_node_test_and_set_flag() │
    │  (بيتأكد مش متعمل قبل كده) │
    │         │                    │
    │         ▼                    │
    │  of_find_gpio_device_by_node │
    │  (من parent node)            │
    │         │                    │
    │         ▼                    │
    │  of_gpiochip_add_hog()       │
    │  يضيف الـ hog descriptors    │
    └──────────────────────────────┘
    │
    │ OF_RECONFIG_CHANGE_REMOVE
    │
    │  هل الـ node متعمل له flag؟
    │  ─── YES ───────────────────►
    │                              │
    │  of_find_gpio_device_by_node │
    │         │                    │
    │         ▼                    │
    │  of_gpiochip_remove_hog()    │
    │  for_each_gpio_desc_with_flag│
    │    (GPIOD_FLAG_IS_HOGGED)    │
    │    لو desc->hog == hog_node  │
    │      gpiochip_free_own_desc()│
    │  of_node_clear_flag()        │
    └──────────────────────────────┘
```

---

### مخطط تدفق `of_gpiochip_add_pin_range()` — ربط GPIO بالـ Pinctrl

```
of_gpiochip_add_pin_range(chip)
         │
         │  loop على كل "gpio-ranges" entries في الـ DTS
         ▼
  of_parse_phandle_with_fixed_args(np, "gpio-ranges",
                                   n_cells+1, index, &pinspec)
         │
    ┌────┴──────────────────────┐
    │ 2-cell chip               │ 3-cell chip
    │ args: [gpio_off, pin, cnt]│ args: [instance, gpio_off, pin, cnt]
    └────┬──────────────────────┘
         │
         ▼
  of_pinctrl_get(pinspec.np) → pctldev
         │
    ┌────┴──────────────────────────────┐
    │ count > 0: linear range           │ count == 0: named group range
    │                                   │
    ▼                                   ▼
gpiochip_add_pin_range(chip,      gpiochip_add_pingroup_range(chip,
  pctldev_name,                     pctldev,
  gpio_offset,                      gpio_offset,
  pin_base,                         group_name)
  count)
```

---

### استراتيجية الـ Locking

الملف ده **لا يمسك locks بنفسه** — الـ locking بيتعمل في طبقات تانية:

| السيناريو | الـ Lock المسؤول | أين |
|-----------|----------------|------|
| البحث عن GPIO chip | `gpio_devices_lock` (أو RCU في بعض الإصدارات) | `gpio_device_find()` في gpiolib.c |
| قراءة/كتابة `desc->hog` | `WRITE_ONCE`/`READ_ONCE` | atomic — no lock needed بسبب الـ memory model |
| الـ notifier chain | الـ OF notifier rwsem | kernel OF subsystem |
| الـ `of_node` refcount | atomic operations | OF core |
| الـ pin range operations | `gpio_chip`'s internal lock | pinctrl/gpiolib core |

**ملاحظة مهمة:** الـ `of_node_test_and_set_flag()` في `of_gpio_notify()` بتشتغل كـ atomic operation عشان تمنع race condition لو نفس الـ hog node اتضاف مرتين في نفس الوقت.

**lock ordering** (المهم تتبعه عشان ما يحصلش deadlock):
```
OF core locks
    └─► gpio_device refcount (atomic, no blocking)
            └─► gpio_chip internal operations
                    └─► pinctrl locks (لو في pin ranges)
```

---

### ملخص الـ API العام (المعرَّف في الـ Header)

| الدالة | الغرض | متى تُستخدم |
|--------|--------|-------------|
| `of_find_gpio()` | الدالة الرئيسية للـ GPIO lookup من DT | الـ gpiolib core بيناديها |
| `of_gpiochip_add()` | تسجيل chip مع الـ OF layer | الـ `gpiochip_add_data()` بيناديها |
| `of_gpiochip_remove()` | إلغاء التسجيل وتحرير الـ node ref | الـ `gpiochip_remove()` بيناديها |
| `of_gpiochip_instance_match()` | تأكيد الـ instance لـ multi-chip nodes | الـ xlate logic |
| `of_gpio_count()` | عدّ الـ GPIOs في property معيّنة | الـ consumer API |
| `gpio_of_notifier` | متغير عام للـ notifier block | بيتسجّل في OF notifier chain |
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions — Cheatsheet

#### تصنيف الـ APIs الرئيسية

| Function | Category | Exported? | Context |
|---|---|---|---|
| `of_find_gpio()` | Lookup | No (internal) | Process |
| `of_gpio_count()` | Lookup | No (internal) | Process |
| `of_gpiochip_add()` | Registration | No (internal) | Process |
| `of_gpiochip_remove()` | Cleanup | No (internal) | Process |
| `of_gpiochip_instance_match()` | Helper | No (internal) | Any |
| `of_get_named_gpio()` | Lookup (deprecated) | `EXPORT_SYMBOL_GPL` | Process |
| `of_get_named_gpiod_flags()` | Lookup (static) | No | Process |
| `of_gpio_named_count()` | Count helper (static) | No | Process |
| `of_gpio_spi_cs_get_count()` | Quirk count (static) | No | Process |
| `of_gpiochip_add_pin_range()` | Registration helper | No | Process |
| `of_gpiochip_scan_gpios()` | Hog scanner | No | Process |
| `of_gpiochip_add_hog()` | Hog registration | No | Process |
| `of_parse_own_gpio()` | Hog parser | No | Process |
| `of_gpio_twocell_xlate()` | Xlate callback | No | Process |
| `of_gpio_threecell_xlate()` | Xlate callback | No | Process |
| `of_convert_gpio_flags()` | Flag converter | No | Any |
| `of_gpio_flags_quirks()` | Quirk handler | No | Process |
| `of_gpio_try_fixup_polarity()` | Quirk: static polarity | No | Process |
| `of_gpio_set_polarity_by_property()` | Quirk: DT property | No | Process |
| `of_gpio_quirk_polarity()` | Quirk: low-level | No | Process |
| `of_find_gpio_rename()` | Legacy name lookup | No | Process |
| `of_find_mt2701_gpio()` | Vendor quirk | No | Process |
| `of_find_trigger_gpio()` | LED trigger GPIO | No | Process |
| `of_gpio_notify()` | OF notifier callback | No | Kernel thread |
| `of_gpiochip_remove_hog()` | Hog removal | No | Process |
| `of_find_gpio_device_by_node()` | Device finder | No | Process |
| `of_find_gpio_device_by_xlate()` | Device finder | No | Process |
| `of_gpiochip_match_node()` | Match callback | No | Any |
| `of_gpiochip_match_node_and_xlate()` | Match callback | No | Any |

---

### Group 1: Lookup & Count Functions

**الـ group ده مسؤول عن البحث عن GPIO descriptors من الـ Device Tree وعد الـ GPIOs المتاحة لكل consumer device.**

---

#### `of_gpio_count()`

```c
int of_gpio_count(const struct fwnode_handle *fwnode, const char *con_id)
```

**الـ function دي بتحسب عدد الـ GPIOs المعرّفة في الـ DT property المرتبطة بالـ `con_id`.** بتمر على أشكال الـ property names الممكنة (مثلاً `foo-gpios` و`foo-gpio`) عن طريق `for_each_gpio_property_name`. قبل كده بتعمل special-case للـ SPI chip selects عن طريق `of_gpio_spi_cs_get_count`.

| Parameter | الوصف |
|---|---|
| `fwnode` | الـ firmware node للـ consumer device — بيتحول لـ `device_node` داخلياً بـ `to_of_node()` |
| `con_id` | اسم الـ GPIO function، مثل `"reset"` أو `"cs"` |

**Return:** عدد الـ GPIOs (>= 0)، أو `-ENOENT` لو مفيش property، أو `-EINVAL` لو الـ property format غلط.

**Key details:** الـ function دي هي الـ entry point من الـ `gpiolib` core للـ OF backend لما بيحتاج يعرف count. مش بتأخذ لوك.

---

#### `of_gpio_named_count()` (static)

```c
static int of_gpio_named_count(const struct device_node *np,
                               const char *propname)
```

**Thin wrapper حوالين `of_count_phandle_with_args()` بيستخدم `"#gpio-cells"` كـ cells property.** بتعد كل الـ GPIO specifiers في الـ property حتى لو فيها empty specifiers (zero phandles).

| Parameter | الوصف |
|---|---|
| `np` | الـ device node |
| `propname` | اسم الـ property الكامل، مثل `"reset-gpios"` |

**Return:** عدد الـ entries، أو errno سالب.

---

#### `of_gpio_spi_cs_get_count()` (static)

```c
static int of_gpio_spi_cs_get_count(const struct device_node *np,
                                    const char *con_id)
```

**بتتعامل مع quirk في SPI controllers قديمة (Freescale، IBM PPC4xx، Aeroflex Gaisler) اللي بتستخدم `"gpios"` بدل `"cs-gpios"`.** لو الـ `con_id` مش `"cs"` أو الـ device مش compatible، بترجع 0 على طول.

**Return:** عدد الـ GPIOs من property `"gpios"` لو الشروط اتحققت، أو 0.

---

#### `of_find_gpio()` (exported in header, used by core)

```c
struct gpio_desc *of_find_gpio(struct device_node *np, const char *con_id,
                               unsigned int idx, unsigned long *flags)
```

**دي الـ main entry point للـ GPIO lookup من الـ OF backend.** بتحاول تلاقي الـ GPIO descriptor بالترتيب التالي:
1. الـ property `foo-gpios` أو `foo-gpio` بشكل صريح.
2. لو فشلت، بتمر على array من الـ quirk handlers: `of_find_gpio_rename`، `of_find_mt2701_gpio`، `of_find_trigger_gpio`.

بعد ما تلاقي الـ descriptor، بتحوّل الـ OF-specific flags لـ generic `GPIO_*` flags عن طريق `of_convert_gpio_flags`.

| Parameter | الوصف |
|---|---|
| `np` | الـ device node للـ consumer |
| `con_id` | اسم الـ GPIO function (مثل `"reset"`)، ممكن يكون NULL |
| `idx` | الـ index في الـ property (لو فيه أكتر من GPIO) |
| `flags` | pointer لـ `unsigned long` — الـ function بتكتب فيها الـ `GPIO_*` flags المحوّلة |

**Return:** `struct gpio_desc *` صالح، أو `ERR_PTR(-ENOENT)` لو مش موجود، أو `ERR_PTR(-EPROBE_DEFER)` لو الـ GPIO controller لسه ما probe-اش.

**Pseudocode:**
```
for each propname in {"con_id-gpios", "con_id-gpio"}:
    desc = of_get_named_gpiod_flags(np, propname, idx, &of_flags)
    if desc found: break

if not found:
    for each quirk in of_find_gpio_quirks[]:
        desc = quirk(np, con_id, idx, &of_flags)
        if desc found: break

if IS_ERR(desc): return desc
*flags = of_convert_gpio_flags(of_flags)
return desc
```

---

#### `of_get_named_gpiod_flags()` (static)

```c
static struct gpio_desc *of_get_named_gpiod_flags(const struct device_node *np,
        const char *propname, int index, enum of_gpio_flags *flags)
```

**دي اللب الحقيقي للـ OF GPIO lookup — بتعمل parse للـ phandle، بتدور على الـ gpio_device المناسب، وبتعمل xlate.** الخطوات بالترتيب:
1. `of_parse_phandle_with_args_map()` — بتفسر الـ phandle وبتحوّل الـ args عبر `gpio-map` لو موجودة.
2. `of_find_gpio_device_by_xlate()` — بتدور على الـ `gpio_device` اللي بيقبل الـ gpiospec ده.
3. `of_xlate_and_get_gpiod_flags()` — بتترجم الـ specifier لـ GPIO number وبتجيب الـ descriptor.
4. `of_gpio_flags_quirks()` — بتعدّل الـ flags لو في quirks معروفة.

**Return:** `gpio_desc *` أو `ERR_PTR`. لو الـ gpio_device مش لقيها: `ERR_PTR(-EPROBE_DEFER)`.

**Key details:** بتستخدم `__free(gpio_device_put)` cleanup attribute لضمان release الـ reference، هذا يعني الكود modern C cleanup-based. بتعمل `of_node_put` على الـ gpiospec.np قبل الخروج في كل الـ paths.

---

#### `of_get_named_gpio()` (DEPRECATED, exported)

```c
int of_get_named_gpio(const struct device_node *np, const char *propname,
                      int index)
```

**Deprecated wrapper بيرجع GPIO number الـ legacy (integer) بدل `gpio_desc *`.** بتستدعي `of_get_named_gpiod_flags()` وبتحوّل النتيجة بـ `desc_to_gpio()`.

**Return:** GPIO number >= 0، أو errno سالب.

**ملحوظة:** الكود الجديد ممنوع يستخدم الـ function دي — بتتوجد بس للـ backward compatibility مع drivers قديمة.

---

### Group 2: Registration Functions

**الـ group ده مسؤول عن ربط الـ `gpio_chip` بالـ Device Tree وإعداده لما بييجي الـ probe.**

---

#### `of_gpiochip_add()`

```c
int of_gpiochip_add(struct gpio_chip *chip)
```

**دي الـ function الرئيسية اللي بتسمّي الـ gpio_chip بالـ OF subsystem.** بتمر بالخطوات دي:
1. لو الـ `chip->of_xlate` مش متحطة، بتحط default واحدة: `of_gpio_twocell_xlate` لو `of_gpio_n_cells == 2`، أو `of_gpio_threecell_xlate` لو `== 3` (بشرط وجود `of_node_instance_match`).
2. بتتحقق إن `of_gpio_n_cells` مش أكبر من `MAX_PHANDLE_ARGS`.
3. بتستدعي `of_gpiochip_add_pin_range()` لربط الـ GPIO lines بالـ pinctrl ranges.
4. بتعمل `of_node_get()` على الـ device node لحجز reference.
5. بتستدعي `of_gpiochip_scan_gpios()` لتسجيل الـ GPIO hogs.

| Parameter | الوصف |
|---|---|
| `chip` | الـ `gpio_chip` اللي بيتسجّل — لازم `chip->gpiodev->dev` يكون معاه `of_node` |

**Return:** 0 لو نجح، `-EINVAL` لو الـ config غلط، أو errno من الـ helpers.

**Key details:** بتُستدعى من `gpiochip_add_data()` في الـ core. لو الـ device node مش موجود (pure ACPI device مثلاً) بترجع 0 على طول من غير ما تعمل حاجة.

**Pseudocode:**
```
np = dev_of_node(&chip->gpiodev->dev)
if !np: return 0

if !chip->of_xlate:
    if n_cells == 3:
        if !of_node_instance_match: return -EINVAL
        chip->of_xlate = of_gpio_threecell_xlate
    else:
        chip->of_gpio_n_cells = 2
        chip->of_xlate = of_gpio_twocell_xlate

if n_cells > MAX_PHANDLE_ARGS: return -EINVAL

ret = of_gpiochip_add_pin_range(chip)
if ret: return ret

of_node_get(np)
ret = of_gpiochip_scan_gpios(chip)
if ret: of_node_put(np)
return ret
```

---

#### `of_gpiochip_add_pin_range()` (static, CONFIG_PINCTRL)

```c
static int of_gpiochip_add_pin_range(struct gpio_chip *chip)
```

**بتقرأ الـ `gpio-ranges` property من الـ DT وبتربط كل range من الـ GPIO lines بـ pinctrl lines مقابلة.** بتدعم كل من الـ linear ranges (بـ count > 0) والـ named group ranges (بـ count == 0 مع `gpio-ranges-group-names`).

**Key details:**
- لو الـ pinctrl device مش موجود بعد: ترجع `-EPROBE_DEFER`.
- في حالة الـ 3-cell chips، بتتحقق من `of_node_instance_match` عشان تتجاهل ranges مش خاصة بالـ chip instance ده.
- بتعمل trim للـ range لو بتتداخل مع حدود الـ chip.

**Return:** 0، أو `-EPROBE_DEFER`، أو errno من helpers.

---

### Group 3: Cleanup Functions

---

#### `of_gpiochip_remove()`

```c
void of_gpiochip_remove(struct gpio_chip *chip)
```

**بتعمل `of_node_put()` على الـ device node اللي اتحجز في `of_gpiochip_add()`.** بسيطة جداً — بس مهمة لتفادي memory leak في الـ `device_node` reference count.

**يُستدعى من:** `gpiochip_remove()` في الـ core مباشرة.

---

### Group 4: GPIO Hog Functions

**الـ GPIO hog هو GPIO بيتحجز ويتضبط من الـ kernel نفسه أثناء الـ boot بناءً على الـ DT، من غير ما أي driver user-space يطلبه.**

---

#### `of_gpiochip_scan_gpios()` (static)

```c
static int of_gpiochip_scan_gpios(struct gpio_chip *chip)
```

**بتمشي على كل الـ child nodes للـ gpio_chip device node وبتدور على أي node عندها property `gpio-hog`.** لكل node زي دي، بتستدعي `of_gpiochip_add_hog()` وبتعمل `of_node_set_flag(np, OF_POPULATED)` بعد النجاح.

**يُستدعى من:** `of_gpiochip_add()` — بس مرة واحدة في الـ probe.

---

#### `of_gpiochip_add_hog()` (static)

```c
static int of_gpiochip_add_hog(struct gpio_chip *chip, struct device_node *hog)
```

**بتمشي على كل الـ GPIO entries في الـ hog node (0, 1, 2...) وبتسجّل كل واحدة عن طريق `gpiod_hog()`.** بتفضل تتكرر بـ `i++` لحد ما `of_parse_own_gpio()` يرجع error (يعني خلصنا الـ entries).

| Parameter | الوصف |
|---|---|
| `chip` | الـ gpio_chip الأب |
| `hog` | الـ device node للـ hog (child node فيها property `gpio-hog`) |

**Return:** 0 لو نجح، أو negative errno من `gpiod_hog`.

**Key details:** لو `CONFIG_OF_DYNAMIC` مفعَّل، بتكتب `desc->hog = hog` بـ `WRITE_ONCE` عشان الـ dynamic removal يعرف يشيل الـ hog الصح بـ RCU safety.

---

#### `of_parse_own_gpio()` (static)

```c
static struct gpio_desc *of_parse_own_gpio(struct device_node *np,
                                           struct gpio_chip *chip,
                                           unsigned int idx, const char **name,
                                           unsigned long *lflags,
                                           enum gpiod_flags *dflags)
```

**بتقرأ الـ GPIO specifier رقم `idx` من الـ `gpios` property في الـ hog node وبتحوّله لـ descriptor مع كل الـ flags المطلوبة.**

الـ function دي بتقرأ `#gpio-cells` من الـ parent chip node عشان تعرف هتقرأ كام cell من الـ `gpios` property، وبعدين بتبني `of_phandle_args` يدوياً وبتعمله xlate.

بعد الـ xlate، بتحدد الـ `dflags` من properties:
- `input` → `GPIOD_IN`
- `output-low` → `GPIOD_OUT_LOW`
- `output-high` → `GPIOD_OUT_HIGH`
- لو مفيش property من دول: تحذير وترجع `-EINVAL`

**Return:** `gpio_desc *`، أو `ERR_PTR(-EINVAL)` لو مفيش `of_node` أو مفيش hogging state.

---

#### `of_gpiochip_remove_hog()` (static, CONFIG_OF_DYNAMIC)

```c
static void of_gpiochip_remove_hog(struct gpio_chip *chip,
                                   struct device_node *hog)
```

**بتمشي على كل الـ descriptors الـ hogged في الـ chip وبتشيل اللي ارتبطت بالـ hog node المحددة.** بتستخدم `for_each_gpio_desc_with_flag(chip, desc, GPIOD_FLAG_IS_HOGGED)` وبتقارن `desc->hog` بـ `READ_ONCE`.

**يُستدعى من:** `of_gpio_notify()` في حالة `OF_RECONFIG_CHANGE_REMOVE`.

---

### Group 5: Xlate (Translation) Functions

**الـ xlate functions هي callbacks بتترجم الـ GPIO specifier من الـ DT (اللي ممكن يكون 2 أو 3 cells) لـ GPIO number داخل الـ chip.**

---

#### `of_gpio_twocell_xlate()` (static)

```c
static int of_gpio_twocell_xlate(struct gpio_chip *gc,
                                 const struct of_phandle_args *gpiospec,
                                 u32 *flags)
```

**الـ default xlate لأغلبية الـ GPIO chips: `cell[0]` هو الـ GPIO number، و`cell[1]` هو الـ flags.** بتتحقق إن `of_gpio_n_cells == 2` وإن الـ `args[0]` أقل من `gc->ngpio`.

```c
// DT: foo-gpios = <&gpio0 5 GPIO_ACTIVE_LOW>;
// args[0] = 5  (GPIO number)
// args[1] = 1  (OF_GPIO_ACTIVE_LOW)
```

**Return:** GPIO number (`args[0]`) لو نجحت، أو `-EINVAL`.

---

#### `of_gpio_threecell_xlate()` (static)

```c
static int of_gpio_threecell_xlate(struct gpio_chip *gc,
                                   const struct of_phandle_args *gpiospec,
                                   u32 *flags)
```

**للـ chips اللي بتكون فيها أكتر من instance من نفس الـ chip type على نفس الـ DT node: `cell[0]` هو الـ instance number، `cell[1]` هو الـ GPIO offset، `cell[2]` هو الـ flags.**

بتستخدم `gc->of_node_instance_match(gc, args[0])` عشان تتحقق إن الـ request دي للـ instance ده وبالتالي ترفض لو مش هو.

```c
// DT: foo-gpios = <&gpio 1 3 GPIO_ACTIVE_HIGH>;
// args[0] = 1  (instance index)
// args[1] = 3  (GPIO offset within instance)
// args[2] = 0  (flags)
```

**Return:** `args[1]` لو نجحت، أو `-EINVAL`.

---

#### `of_xlate_and_get_gpiod_flags()` (static)

```c
static struct gpio_desc *of_xlate_and_get_gpiod_flags(struct gpio_chip *chip,
                                struct of_phandle_args *gpiospec,
                                enum of_gpio_flags *flags)
```

**Wrapper بيستدعي `chip->of_xlate()` ثم بيحوّل الـ GPIO number اللي رجع لـ `gpio_desc *` بـ `gpiochip_get_desc()`.** بتتحقق إن `args_count == of_gpio_n_cells` قبل الاستدعاء.

**Return:** `gpio_desc *`، أو `ERR_PTR(-EINVAL)` لو الـ cells count مش مطابق.

---

### Group 6: Flag Conversion & Quirk Functions

**الـ group ده بيتعامل مع ترجمة وتصحيح الـ GPIO flags القادمة من الـ DT — لأن الـ DT bindings القديمة فيها inconsistencies تاريخية كتير.**

---

#### `of_convert_gpio_flags()` (static)

```c
static unsigned long of_convert_gpio_flags(enum of_gpio_flags flags)
```

**بتحوّل الـ `of_gpio_flags` enum (اللي جاي من الـ DT xlate) لـ `GPIO_*` bitmask المستخدم في الـ generic GPIO API.**

| OF Flag | GPIO Flag |
|---|---|
| `OF_GPIO_ACTIVE_LOW` | `GPIO_ACTIVE_LOW` |
| `OF_GPIO_SINGLE_ENDED` + `OF_GPIO_OPEN_DRAIN` | `GPIO_OPEN_DRAIN` |
| `OF_GPIO_SINGLE_ENDED` (بدون `OPEN_DRAIN`) | `GPIO_OPEN_SOURCE` |
| `OF_GPIO_TRANSITORY` | `GPIO_TRANSITORY` |
| `OF_GPIO_PULL_UP` | `GPIO_PULL_UP` |
| `OF_GPIO_PULL_DOWN` | `GPIO_PULL_DOWN` |
| `OF_GPIO_PULL_DISABLE` | `GPIO_PULL_DISABLE` |

**Return:** الـ `unsigned long` flags الجاهزة لـ `gpiod_get()` وأصحابها.

---

#### `of_gpio_flags_quirks()` (static)

```c
static void of_gpio_flags_quirks(const struct device_node *np,
                                 const char *propname,
                                 enum of_gpio_flags *flags,
                                 int index)
```

**الـ orchestrator للـ flag quirks — بتستدعي الـ quirk handlers بالترتيب.** بتعمل 5 أشياء:
1. `of_gpio_try_fixup_polarity()` — static polarity overrides معروفة.
2. `of_gpio_set_polarity_by_property()` — polarity من DT property تانية.
3. Legacy open-drain لـ `reg-fixed-voltage`.
4. SPI CS active-high handling بناءً على `spi-cs-high` property في الـ child node.
5. `snps,reset-active-low` للـ stmmac.

**يُستدعى من:** `of_get_named_gpiod_flags()` بس لما الـ `flags` مش NULL.

---

#### `of_gpio_quirk_polarity()` (static)

```c
static void of_gpio_quirk_polarity(const struct device_node *np,
                                   bool active_high,
                                   enum of_gpio_flags *flags)
```

**Low-level polarity enforcer — بتضرب/تشيل `OF_GPIO_ACTIVE_LOW` بناءً على الـ `active_high` المطلوب.** لو في conflict (مثلاً الـ DT قال active-low بس الـ quirk يقول active-high)، بتطبع warning وبتغلب على الـ DT.

---

#### `of_gpio_try_fixup_polarity()` (static)

```c
static void of_gpio_try_fixup_polarity(const struct device_node *np,
                                       const char *propname,
                                       enum of_gpio_flags *flags)
```

**بتحتوي على static lookup table من الـ DT bindings المعروفة إن فيها polarity غلط.** بتعمل match على (compatible, propname) وبتطبق الـ polarity الصح. الـ entries الحالية تشمل:
- Himax HX8357/HX8369 `gpios-reset`
- Ingenic JZ4780 `rb-gpios`
- Cascoda CA8210 `reset-gpio`
- Lantiq PCI `gpio-reset`
- Samsung S5M8767 DVS/DS GPIOs

---

#### `of_gpio_set_polarity_by_property()` (static)

```c
static void of_gpio_set_polarity_by_property(const struct device_node *np,
                                             const char *propname,
                                             enum of_gpio_flags *flags)
```

**بتشوف لو في property تانية في الـ DT بتحدد الـ polarity بدل الـ flags في الـ gpio specifier نفسه.** مثلاً `phy-reset-active-high` في `fsl,imx6q-fec` بتحدد polarity الـ `phy-reset-gpios`. الـ Atmel HSMCI عندها special case لأن الـ compatible في الـ parent node والـ GPIO property في الـ child.

---

### Group 7: Legacy Name Lookup (Quirks)

---

#### `of_find_gpio_rename()` (static)

```c
static struct gpio_desc *of_find_gpio_rename(struct device_node *np,
                                             const char *con_id,
                                             unsigned int idx,
                                             enum of_gpio_flags *of_flags)
```

**بتتعامل مع الـ drivers اللي كانت بتستخدم أسماء GPIO properties مختلفة عن الـ standard `foo-gpios` convention.** بتحتوي على lookup table من `(con_id, legacy_id, compatible)` triples وبتجرب الـ legacy name لما الـ standard name مش موجود.

أمثلة من الـ table:

| con_id | legacy_id | compatible |
|---|---|---|
| `"reset"` | `"gpios-reset"` | `"himax,hx8357"` |
| `"cs"` | `"gpios"` | `"fsl,spi"` |
| `"miso"` | `"gpio-miso"` | `"spi-gpio"` |
| `"fcs,int_n"` | NULL (same) | `"fcs,fusb302"` |

**Return:** `gpio_desc *` لو لقت legacy name، أو `ERR_PTR(-ENOENT)`.

---

#### `of_find_trigger_gpio()` (static)

```c
static struct gpio_desc *of_find_trigger_gpio(struct device_node *np,
                                              const char *con_id,
                                              unsigned int idx,
                                              enum of_gpio_flags *of_flags)
```

**Special case للـ LED trigger sources — الـ property `trigger-sources` ممكن تشاور على أي نوع من الـ phandles مش بس GPIOs.** بتسمح بالـ lookup من property مش بتنتهي بـ `-gpios`.

**Active فقط لما:** `CONFIG_LEDS_TRIGGER_GPIO` مفعَّل، والـ `con_id == "trigger-sources"`.

---

#### `of_find_mt2701_gpio()` (static, CONFIG_SND_SOC_MT2701_CS42448)

```c
static struct gpio_desc *of_find_mt2701_gpio(struct device_node *np,
                                             const char *con_id,
                                             unsigned int idx,
                                             enum of_gpio_flags *of_flags)
```

**Vendor-specific quirk للـ MediaTek MT2701 مع CS42448 codec — بتحوّل الـ `i2s1-in-sel[0/1]` GPIOs لـ DT property names مختلفة.** بتعمل manual mapping بين الـ index وبين الـ legacy property name.

---

### Group 8: Dynamic OF (CONFIG_OF_DYNAMIC)

**الـ group ده خاص بالـ systems اللي بتدعم Dynamic Device Tree — اللي فيها ممكن يتضاف أو يتشال nodes من الـ DT وهو الـ system شغّال.**

---

#### `of_gpio_notify()` (static)

```c
static int of_gpio_notify(struct notifier_block *nb, unsigned long action,
                          void *arg)
```

**الـ notifier callback المسجّل في `gpio_of_notifier` — بتستجيب لأحداث الـ OF reconfig (إضافة/إزالة DT nodes).** بتدعم بس إضافة وإزالة `gpio-hog` nodes الكاملة.

```
OF_RECONFIG_CHANGE_ADD:
    if !gpio-hog property: return NOTIFY_DONE
    if already populated: return NOTIFY_DONE
    find gpio_device by parent node
    call of_gpiochip_add_hog()

OF_RECONFIG_CHANGE_REMOVE:
    if not populated: return NOTIFY_DONE
    find gpio_device by parent node
    call of_gpiochip_remove_hog()
    clear OF_POPULATED flag
```

**Key details:** بيستخدم `of_node_test_and_set_flag(OF_POPULATED)` atomic لمنع الـ double-add. بيعمل `gpio_device_put` auto-cleanup بـ `__free()` attribute. لو فشل الـ add: بتمسح الـ `OF_POPULATED` flag وبترجع `notifier_from_errno(ret)`.

---

#### `gpio_of_notifier` (extern)

```c
struct notifier_block gpio_of_notifier = {
    .notifier_call = of_gpio_notify,
};
```

**الـ notifier block المُصدَّر في الـ header اللي بيربط الـ GPIO subsystem بأحداث الـ OF reconfig chain.** بيتسجّل في مكان تاني في الـ gpiolib core ليستقبل أحداث الـ DT overlays.

---

### Group 9: Device Finder Helpers

---

#### `of_find_gpio_device_by_xlate()` (static)

```c
static struct gpio_device *
of_find_gpio_device_by_xlate(const struct of_phandle_args *gpiospec)
```

**بتدور على الـ `gpio_device` اللي بيمتلك الـ chip المناسب لتفسير الـ `gpiospec` المعطى.** بتستخدم `gpio_device_find()` مع callback `of_gpiochip_match_node_and_xlate`.

**الـ match callback** بتتحقق من شرطين معاً:
1. الـ device node بتاع الـ chip مطابق للـ phandle في الـ spec.
2. الـ `chip->of_xlate()` بيقبل الـ spec (يرجع >= 0).

**Return:** `gpio_device *` مع incremented reference count، أو NULL.

---

#### `of_find_gpio_device_by_node()` (static, CONFIG_OF_DYNAMIC)

```c
static struct gpio_device *of_find_gpio_device_by_node(struct device_node *np)
```

**بتدور على الـ `gpio_device` اللي بيطابق الـ node المعطى بالضبط.** بتستخدم `of_gpiochip_match_node()` كـ callback لـ `gpio_device_find()`.

---

#### `of_gpiochip_instance_match()`

```c
bool of_gpiochip_instance_match(struct gpio_chip *gc, unsigned int index)
```

**Thin wrapper على `gc->of_node_instance_match()`.** لو الـ callback مش محطوط: ترجع `false` على طول. الـ function دي مُعلَنة في الـ header عشان بتُستخدم من الـ gpiolib core.

| Parameter | الوصف |
|---|---|
| `gc` | الـ gpio_chip |
| `index` | الـ instance index اللي هيتحقق منه |

**Return:** `true` لو الـ chip هو الـ instance رقم `index`، أو `false`.

---

### ملخص تدفق الـ GPIO Lookup من الـ DT

```
gpiod_get(dev, con_id, flags)
    └─► gpiod_find_and_request()
            └─► of_find_gpio(np, con_id, idx, &lflags)
                    ├─► for propname in {"con_id-gpios", "con_id-gpio"}:
                    │       └─► of_get_named_gpiod_flags(np, propname, idx, &of_flags)
                    │               ├─► of_parse_phandle_with_args_map()   [parse DT phandle]
                    │               ├─► of_find_gpio_device_by_xlate()     [find gpio_device]
                    │               ├─► of_xlate_and_get_gpiod_flags()     [translate to desc]
                    │               └─► of_gpio_flags_quirks()             [fix-up flags]
                    │
                    └─► if not found: run quirks[]
                            ├─► of_find_gpio_rename()    [legacy names]
                            ├─► of_find_mt2701_gpio()    [vendor quirk]
                            └─► of_find_trigger_gpio()   [LED triggers]
                                        │
                                        ▼
                            of_convert_gpio_flags()  [OF flags → GPIO_* bitmask]
                                        │
                                        ▼
                            gpio_desc* + flags  →  back to gpiod_get()
```
## Phase 5: دليل الـ Debugging الشامل

الـ `gpiolib-of.h` هو الـ interface بين الـ GPIO subsystem وبين الـ Device Tree (OF = Open Firmware). الـ debugging هنا بيتركز على ثلاث محاور: صح تحويل الـ DT nodes لـ gpio descriptors، صح تسجيل الـ gpiochip، وصح تشغيل الـ `gpio_of_notifier`.

---

### Software Level

#### 1. debugfs entries

الـ GPIO subsystem بيعمل expose كامل عبر `/sys/kernel/debug/gpio`:

```bash
# اقرأ كل الـ gpiochips المسجلة مع تفاصيل كل line
cat /sys/kernel/debug/gpio
```

**مثال على الـ output:**
```
gpiochip0: GPIOs 0-31, parent: platform/fe200000.gpio, pinctrl-bcm2835:
 gpio-2   (                    |scl             ) in  hi    IRQ ACTIVE LOW
 gpio-3   (                    |sda             ) in  hi    IRQ ACTIVE LOW
 gpio-17  (                    |led0            ) out lo
```

كل سطر بيظهر: رقم الـ GPIO، الـ consumer اللي استخدمه، الاتجاه (in/out)، القيمة (hi/lo)، وإذا كان على IRQ.

```bash
# تفاصيل الـ IRQ domain المرتبط بالـ gpiochip
cat /sys/kernel/debug/irq/domains/gpio
```

---

#### 2. sysfs entries

```bash
# كل الـ gpiochips المتاحة
ls /sys/bus/gpio/devices/

# معلومات gpiochip محدد (عدد الـ lines، الاسم، الـ label)
cat /sys/class/gpio/gpiochip0/label
cat /sys/class/gpio/gpiochip0/ngpio
cat /sys/class/gpio/gpiochip0/base

# تحقق إن الـ chip مربوط بـ OF node
ls -la /sys/class/gpio/gpiochip0/of_node
# المفروض يطلع symlink زي:
# of_node -> ../../../firmware/devicetree/base/soc/gpio@fe200000

# export GPIO يدوي للاختبار
echo 17 > /sys/class/gpio/export
cat /sys/class/gpio/gpio17/direction
cat /sys/class/gpio/gpio17/value
echo 17 > /sys/class/gpio/unexport
```

---

#### 3. ftrace — tracepoints وأحداث الـ GPIO

```bash
# شوف كل الـ trace events المتعلقة بالـ GPIO
ls /sys/kernel/debug/tracing/events/gpio/

# فعّل كل أحداث الـ GPIO
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable

# أو اختار حدث محدد: gpio_value يُطلق عند كل get/set
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_direction/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

**مثال على الـ output:**
```
      systemd-1     [000] ....  1234.567890: gpio_value: 17 get 1
      kworker/0:1   [000] ....  1234.568001: gpio_direction: 17 in
```

لتتبع `of_find_gpio` و `of_gpiochip_add` اللي مش ليهم tracepoints جاهزة، استخدم الـ function tracer:

```bash
echo function > /sys/kernel/debug/tracing/current_tracer
echo of_find_gpio > /sys/kernel/debug/tracing/set_ftrace_filter
echo of_gpiochip_add >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

---

#### 4. printk / dynamic debug

```bash
# فعّل كل رسائل الـ debug في الـ gpio subsystem ديناميكيًا
echo 'module gpiolib +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/gpiolib-of.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل ملفات الـ gpio
echo 'file drivers/gpio/* +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ rules المفعّلة
grep gpio /sys/kernel/debug/dynamic_debug/control

# تفعيل أثناء الـ boot عبر kernel cmdline
# في /boot/grub/grub.cfg أو الـ bootloader:
# dyndbg="file drivers/gpio/gpiolib-of.c +p"
```

الـ output هيظهر في `dmesg`:

```
[    2.345678] gpiolib-of: of_gpiochip_add: registered chip gpiochip0 with 54 lines
[    2.345700] gpiolib-of: of_find_gpio: found gpio 17 flags=0x0 for con_id=led
```

---

#### 5. Kernel config options للـ debugging

| **CONFIG** | الغرض |
|---|---|
| `CONFIG_OF_GPIO` | لازم يكون مفعّل حتى يشتغل الكود أصلاً |
| `CONFIG_DEBUG_GPIO` | يفعّل extra validation وـ verbose logging في الـ gpio core |
| `CONFIG_GPIO_SYSFS` | يعرض الـ GPIO عبر sysfs لـ userspace debugging |
| `CONFIG_GPIOLIB_IRQCHIP` | debugging لـ gpiochips اللي بتعمل IRQ controllers |
| `CONFIG_PROVE_LOCKING` | يكشف الـ locking bugs في الـ notifier chain |
| `CONFIG_LOCKDEP` | تتبع هرمية الـ locks في `gpio_of_notifier` |
| `CONFIG_OF_DYNAMIC` | يفعّل الـ notifier لأحداث الـ DT hotplug |
| `CONFIG_DEBUG_FS` | ضروري لكل الـ debugfs entries |
| `CONFIG_TRACING` | ضروري للـ ftrace |
| `CONFIG_GPIO_CDEV` | الـ character device interface للاختبار من userspace |

---

#### 6. أدوات خاصة بالـ subsystem

**الـ gpiod tools (من حزمة `gpiod`):**

```bash
# اعرض كل الـ gpiochips والـ lines
gpiodetect
# output: gpiochip0 [pinctrl-bcm2835] (54 lines)

# معلومات تفصيلية لكل line
gpioinfo gpiochip0

# اقرأ قيمة GPIO
gpioget gpiochip0 17

# اضبط قيمة GPIO
gpioset gpiochip0 17=1

# مراقبة أحداث GPIO (edges)
gpiomon gpiochip0 17
```

**الـ OF/DT inspection:**

```bash
# تحقق من الـ device tree المحمّل في الـ kernel
dtc -I fs /proc/device-tree > /tmp/current.dts 2>/dev/null
grep -A5 "gpio" /tmp/current.dts

# شوف الـ phandles للـ gpio controllers
grep -r "gpio-controller" /proc/device-tree/ 2>/dev/null
```

---

#### 7. رسائل الأخطاء الشائعة

| **رسالة في dmesg** | المعنى | الحل |
|---|---|---|
| `gpio: no more GPIOs, increase CONFIG_ARCH_NR_GPIO` | نفدت الـ gpio numbers | زود `CONFIG_ARCH_NR_GPIO` في الـ kernel config |
| `of_get_named_gpiod_flags: can't parse gpios property` | الـ DT property اسمها غلط أو ناقصة | تحقق من اسم الـ property ووجود `-gpios` suffix |
| `GPIO chip gpiochip0: of_gpiochip_add failed` | فشل تسجيل الـ chip مع OF | تحقق من وجود `gpio-controller;` في الـ DT node |
| `can't claim GPIO 17: it's already in use` | الـ GPIO محجوز من kernel component تاني | استخدم `cat /sys/kernel/debug/gpio` لتحديد من يحتجزه |
| `gpio-N: invalid GPIO (can't request)` | الرقم خارج النطاق أو الـ chip مش registered | تأكد أن `of_gpiochip_add` تمّ بنجاح |
| `fwnode is not associated with a GPIO controller` | الـ fwnode اللي اتبعت من الـ DT مش لـ gpiochip | تحقق أن الـ node عنده `gpio-controller;` و`#gpio-cells` |
| `GPIO lookup failed: see diagnostics above` | `of_find_gpio` رجعت `ERR_PTR` | فعّل `CONFIG_DEBUG_GPIO` وشوف التفاصيل |
| `OF: overlay: WARNING: memory leak will occur if overlay removed` | الـ `gpio_of_notifier` مش بيهنّدل الـ remove صح | مشكلة في الـ overlay lifecycle |

---

#### 8. مواضع `dump_stack()` و`WARN_ON()`

```c
/* في of_find_gpio — عند رجوع NULL descriptor */
if (!desc) {
    WARN_ON(!desc); /* يطبع backtrace + يظهر في dmesg */
    return ERR_PTR(-ENOENT);
}

/* في of_gpiochip_add — تحقق من صحة الـ chip قبل التسجيل */
WARN_ON(!gc->of_node && !gc->parent);

/* عند حدوث mismatch بين الـ DT وعدد الـ lines */
if (gc->ngpio != of_gpio_count(fwnode, NULL)) {
    dev_warn(&gc->gpiodev->dev,
        "DT says %d GPIOs but chip reports %d\n",
        of_gpio_count(fwnode, NULL), gc->ngpio);
    dump_stack(); /* لتتبع من أين جاء الـ registration */
}

/* في الـ notifier callback — تحقق من سلامة البيانات */
WARN_ON(action != OF_RECONFIG_ADD_PROPERTY &&
        action != OF_RECONFIG_REMOVE_PROPERTY);
```

---

### Hardware Level

#### 1. التحقق أن حالة الـ hardware بتطابق الـ kernel state

```bash
# قارن بين ما يقوله الـ kernel وما يقوله الـ hardware
# الـ kernel يقول:
cat /sys/kernel/debug/gpio | grep "gpio-17"
# gpio-17  (led0) out hi

# الـ hardware: قس بالـ multimeter على الـ pin المقابل
# لو الـ kernel قال "out hi" والـ pin عنده 0V = مشكلة في الـ driver أو الـ pinmux
```

**تحقق من الـ pinmux** (مهم جداً لأن GPIO ممكن يكون assigned لـ function تانية):

```bash
# شوف الـ pinmux status
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep "gpio 17"

# لو الـ pin مش في GPIO mode = الـ pinmux مش صح
# الـ of_gpiochip_add بيطلب الـ pin من الـ pinctrl subsystem
```

---

#### 2. Register dump

```bash
# استخدم devmem2 لقراءة الـ GPIO controller registers مباشرة
# مثال: BCM2835 GPIO base address = 0xFE200000
# قراءة GPLEV0 (GPIO Pin Level 0) — بيحتوي على قيم GPIO 0-31
sudo devmem2 0xFE200034 w
# output: 0xFE200034: 0x00020000  ->  bit 17 = 1 = GPIO17 = HIGH

# قراءة GPFSEL1 (Function Select 1) — GPIO 10-19
sudo devmem2 0xFE200004 w
# كل 3 bits بتحدد function: 000=input, 001=output, 100-111=alt functions

# باستخدام /dev/mem مباشرة من C
# أو باستخدام io utility:
sudo io -4 0xFE200034
```

**حساب الـ register offset:**

```
GPIO Function Select: base + 0x00 (GPFSEL0), +0x04 (GPFSEL1), ...
GPIO Output Set:      base + 0x1C (GPSET0)
GPIO Output Clear:    base + 0x28 (GPCLR0)
GPIO Level:           base + 0x34 (GPLEV0)
GPIO Event Detect:    base + 0x40 (GPEDS0)
```

---

#### 3. Logic Analyzer / Oscilloscope

```
مشاكل شائعة تتحتاج logic analyzer:

1. الـ GPIO بيتغير بسرعة أكبر من اللي الـ kernel بيقول
   -> سبب: الـ interrupt handler بيعمل toggle سريع
   -> شيل: channel على الـ pin + trigger على edge

2. الـ GPIO مش بيتغير خالص رغم الـ driver call
   -> سبب: الـ pinmux غلط أو الـ GPIO line مش connected
   -> شيل: probe قبل الـ level shifter وبعده

3. Glitches عند الـ request/release
   -> سبب: الـ of_gpiochip_add بيضبط default direction
   -> شيل: trigger على first edge بعد الـ driver probe

نصايح عملية:
- استخدم channel digital واحد لكل GPIO
- فعّل timestamps بدقة nanoseconds
- قارن مع الـ ftrace output (نفس التوقيت تقريباً)
- الـ pull-up/pull-down في الـ DT (gpio-pull-up, gpio-pull-down)
  ممكن تأثر على القراءة الأولى بعد الـ probe
```

---

#### 4. مشاكل الـ hardware الشائعة وأنماطها في الـ kernel log

| **المشكلة** | **Pattern في الـ kernel log** | **التشخيص** |
|---|---|---|
| الـ GPIO controller مش موجود فعلياً | `of_gpiochip_add: no such device` | الـ DT address غلط، تحقق من `reg` property |
| مشكلة في الـ power rail للـ GPIO bank | `gpio request timeout` + pin stuck | قس الـ VDD للـ GPIO controller |
| الـ pin مربوط لـ function تانية | `pin X is already used by function Y` | راجع الـ pinmux configuration في الـ DT |
| الـ IRQ line للـ GPIO controller غلط | `failed to request IRQ N` | تحقق من `interrupts` property في الـ DT |
| مشكلة في الـ clock للـ GPIO controller | `clk_enable failed` | تأكد أن الـ clock tree صح في الـ DT |
| الـ GPIO line مش موجودة فعلياً في الـ chip | `gpio-N: invalid GPIO offset` | تحقق من `#gpio-cells` وعدد الـ lines |

---

#### 5. Device Tree Debugging

```bash
# قارن الـ DT المُجمَّع (compiled) بالـ hardware فعلاً
# الخطوة 1: استخرج الـ DT المحمّل حالياً
dtc -I fs -O dts /proc/device-tree -o /tmp/live.dts 2>/dev/null

# الخطوة 2: ابحث عن الـ gpio controllers
grep -A 20 "gpio-controller" /tmp/live.dts | head -60

# الخطوة 3: تحقق من صحة الـ gpio specifiers
# كل gpio consumer لازم يطابق #gpio-cells في الـ controller
# مثال: #gpio-cells = <2>  ->  لازم تكتب: <&gpio 17 GPIO_ACTIVE_HIGH>
grep -B2 "gpios = " /tmp/live.dts | head -30

# الخطوة 4: تحقق من الـ phandles
# of_find_gpio بتتبع الـ phandle لأول gpio-controller
grep -n "phandle" /tmp/live.dts | grep gpio
```

**أكثر مشاكل الـ DT شيوعاً مع `gpiolib-of.h`:**

```dts
/* خطأ شائع 1: نسيان gpio-controller */
my_gpio: gpio@12340000 {
    compatible = "vendor,my-gpio";
    reg = <0x12340000 0x1000>;
    /* gpio-controller;  <- منسي! */
    #gpio-cells = <2>;
};

/* خطأ شائع 2: #gpio-cells غلط */
/* لو الـ driver بيتوقع cell واحدة بس الـ DT بيديله اتنين */
leds {
    gpios = <&my_gpio 17 0>;  /* اتنين cells لكن #gpio-cells = <1> */
};

/* خطأ شائع 3: اسم property غلط */
/* الـ of_find_gpio بتدور على "${con_id}-gpios" */
leds {
    led-gpio = <&gpio 17 0>;   /* خطأ */
    led-gpios = <&gpio 17 0>;  /* صح */
};
```

```bash
# تحقق باستخدام of_gpio_count بشكل غير مباشر:
# لو كانت القيمة 0 = الـ property مش موجودة أو اسمها غلط
# شوف الـ dmesg لرسائل من of_gpio_count
dmesg | grep "gpio" | grep -E "count|property|cells"
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

```bash
# ===== تشخيص سريع شامل =====
echo "=== GPIO chips ===" && cat /sys/kernel/debug/gpio
echo "=== OF node links ===" && ls -la /sys/class/gpio/gpiochip*/of_node 2>/dev/null
echo "=== DT gpio-controllers ===" && grep -r "gpio-controller" /proc/device-tree/ 2>/dev/null | head -20

# ===== فعّل debugging للـ gpio subsystem =====
echo 'file drivers/gpio/gpiolib-of.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/gpio/gpiolib.c +pflmt' >> /sys/kernel/debug/dynamic_debug/control
dmesg -w &  # راقب الـ output في الـ background

# ===== ftrace لتتبع استدعاءات of_find_gpio =====
echo function > /sys/kernel/debug/tracing/current_tracer
echo of_find_gpio > /sys/kernel/debug/tracing/set_ftrace_filter
echo of_gpiochip_add >> /sys/kernel/debug/tracing/set_ftrace_filter
echo of_gpiochip_remove >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe

# ===== تنظيف الـ ftrace =====
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
echo > /sys/kernel/debug/tracing/set_ftrace_filter

# ===== تحقق من الـ notifier chain =====
# gpio_of_notifier بيتسجل في OF reconfig chain
# لو فعّلت CONFIG_OF_DYNAMIC:
dmesg | grep -i "OF: reconfig"
```

**تشخيص مشكلة `of_find_gpio` بالتفصيل:**

```bash
# سيناريو: driver بيشتكي إن GPIO مش موجود
# الخطوة 1: تأكد إن الـ chip متسجّل
cat /sys/kernel/debug/gpio | grep "gpiochip"

# الخطوة 2: تأكد إن الـ DT property موجودة بالاسم الصح
# لو con_id = "reset" بتدور على "reset-gpios"
grep -r "reset-gpios" /proc/device-tree/ 2>/dev/null

# الخطوة 3: تحقق من العدد
# of_gpio_count بترجع عدد الـ GPIOs في الـ property
# لو رجعت 0 = مفيش property
ls /proc/device-tree/soc/mydevice@xxx/ 2>/dev/null | grep gpio

# الخطوة 4: شوف الـ phandle يشاور فين
hexdump -C /proc/device-tree/soc/mydevice@xxx/reset-gpios 2>/dev/null
# أول 4 bytes = الـ phandle (big-endian) للـ gpio controller

# ===== devmem2 لقراءة GPIO registers =====
# Raspberry Pi 4 مثال: BCM2711 GPIO base
sudo devmem2 0xFE200034 w  # GPLEV0: قيم GPIO 0-31
sudo devmem2 0xFE200038 w  # GPLEV1: قيم GPIO 32-57
sudo devmem2 0xFE200004 w  # GPFSEL1: function select GPIO 10-19
```

**تفسير output الـ `cat /sys/kernel/debug/gpio`:**

```
gpiochip0: GPIOs 0-53, parent: platform/fe200000.gpio, pinctrl-bcm2835:
 gpio-2   (                    |i2c1_sda        ) in  hi    IRQ
 gpio-17  (                    |led0            ) out lo
 gpio-22  (                    |                ) in  lo
           ^                    ^                 ^    ^     ^
           رقم الـ GPIO          الـ consumer       dir  val   وجود IRQ

- in/out: اتحدد عبر of_find_gpio -> gpiod_configure_flags
- hi/lo: القيمة الفعلية بعد الـ active-high/low polarity
- IRQ: الـ line عندها interrupt مفعّل
- consumer فاضي = الـ GPIO محجوز بس مش مستخدم أو kernel internal
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — الـ I2C GPIO Expander مش بيتعرف

#### العنوان
فشل `of_gpiochip_add()` على GPIO expander متصل بـ I2C في gateway صناعي

#### السياق
شركة بتبني industrial gateway بيشتغل على **RK3562** (Rockchip). الجهاز فيه PCA9555 GPIO expander متصل على I2C bus، وبيتحكم في relay outputs وdigital inputs. الـ kernel driver تم تفعيله، الـ Device Tree node موجود، بس الـ GPIOs مش ظاهرة في `/sys/class/gpio`.

#### المشكلة
بعد boot، الأمر ده بيطلع فاضي:
```bash
ls /sys/class/gpio/
# gpiochip0  gpiochip32  — مفيش gpiochip للـ expander
```
والـ dmesg بيطلع:
```
pca953x 1-0027: of_gpiochip_add failed: -22
```

#### التحليل
الكود في `gpiolib-of.h` بيعرّف:
```c
int of_gpiochip_add(struct gpio_chip *gc);
```
الفانكشن دي بتتنفذ لما الـ driver يعمل `gpiochip_add_data()` وبيحاول يربط الـ `gpio_chip` بالـ Device Tree node الخاص بيه.

الـ `-22` ده `-EINVAL`، يعني في حاجة في الـ DT node غلط خلّت `of_gpiochip_add()` تفشل. الفانكشن بتعمل validation على:
1. وجود `#gpio-cells` property في الـ node
2. صحة قيمة `gpio-ranges` لو موجودة
3. matching بين عدد الـ cells والـ driver المتوقع

بنشوف الـ DT:
```dts
pca9555: gpio@27 {
    compatible = "nxp,pca9555";
    reg = <0x27>;
    gpio-controller;
    /* مفيش #gpio-cells ! */
};
```

المشكلة واضحة: `#gpio-cells` مش موجودة. الـ `of_gpiochip_add()` بتحاول تقرأ الـ property دي عشان تعرف كل GPIO reference كام cell بياخد في الـ DT، لو مش موجودة بترجع `-EINVAL`.

#### الحل
```dts
pca9555: gpio@27 {
    compatible = "nxp,pca9555";
    reg = <0x27>;
    gpio-controller;
    #gpio-cells = <2>;  /* flag + pin number */
    interrupt-parent = <&gpio1>;
    interrupts = <RK_PA3 IRQ_TYPE_LEVEL_LOW>;
};
```
بعدين في consumer node:
```dts
relay-gpios = <&pca9555 0 GPIO_ACTIVE_HIGH>;
```

#### الدرس المستفاد
الـ `of_gpiochip_add()` بتتحقق من `#gpio-cells` قبل ما تكمل registration. أي GPIO controller node في الـ DT **لازم** يحتوي على `#gpio-cells` حتى لو الـ driver نفسه بيحدد `ngpio`. الـ `-EINVAL` من `of_gpiochip_add` في الـ dmesg دايما يبدأ التحقيق بالـ DT node مش بالـ driver code.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — GPIO مش شغال بدون `CONFIG_OF_GPIO`

#### العنوان
الـ `of_find_gpio()` بيرجع `-ENOENT` دايماً رغم وجود الـ DT node الصح

#### السياق
مطور بيعمل custom Android TV box على **Allwinner H616**. الـ IR receiver متوصل على GPIO وبحاجة يتعرف عليه من الـ DT. الـ driver بيستخدم `gpiod_get()` اللي بتتصل داخلياً بـ `of_find_gpio()`. المشكلة إن الكود بيرجع `-ENOENT` دايماً.

#### المشكلة
```bash
dmesg | grep ir
# sunxi-ir: error -2 getting gpio
# -2 == -ENOENT
```
الـ DT node صح تماماً:
```dts
ir_receiver: ir@1c21800 {
    compatible = "allwinner,sun50i-h616-ir";
    reg = <0x01c21800 0x400>;
    ir-gpios = <&pio 1 0 GPIO_ACTIVE_HIGH>;
};
```

#### التحليل
في `gpiolib-of.h` الجزء ده هو المفتاح:

```c
#ifdef CONFIG_OF_GPIO
struct gpio_desc *of_find_gpio(struct device_node *np,
                               const char *con_id,
                               unsigned int idx,
                               unsigned long *lookupflags);
/* ... */
#else
static inline struct gpio_desc *of_find_gpio(struct device_node *np,
                                             const char *con_id,
                                             unsigned int idx,
                                             unsigned long *lookupflags)
{
    return ERR_PTR(-ENOENT);  /* دايما بيرجع error لو CONFIG_OF_GPIO مش معمول */
}
#endif /* CONFIG_OF_GPIO */
```

لو `CONFIG_OF_GPIO` مش متفعل في الـ `.config`، الـ `of_find_gpio()` بتتحول لـ stub فارغة ترجع `-ENOENT` فوراً من غير ما تبص على الـ DT. ده هو بالظبط اللي بيحصل.

نتحقق:
```bash
grep CONFIG_OF_GPIO /boot/.config
# CONFIG_OF_GPIO is not set
```

#### الحل
في `menuconfig`:
```
Device Drivers → GPIO Support → GPIO lib → OF GPIO
[*] GPIO lib OF support (CONFIG_OF_GPIO)
```
أو مباشرة في الـ `.config`:
```bash
echo "CONFIG_OF_GPIO=y" >> arch/arm64/configs/sun50i_h616_defconfig
make ARCH=arm64 sun50i_h616_defconfig
make ARCH=arm64 Image
```

#### الدرس المستفاد
الـ `#else` branch في `gpiolib-of.h` موجود عشان platforms من غير OpenFirmware/DeviceTree. على أي embedded Linux بيستخدم DT، `CONFIG_OF_GPIO` **لازم** يكون مفعّل. الـ stub functions موجودة فقط عشان الكود يكمل compile من غير errors على platforms تانية، مش عشان توفر functionality حقيقية.

---

### السيناريو الثالث: IoT Sensor Board على STM32MP1 — `gpio_of_notifier` وتسريب GPIO بعد device removal

#### العنوان
GPIO resources مش بتتحرر صح لما يتعمل `rmmod` للـ driver على STM32MP1

#### السياق
فريق بيبني IoT sensor node على **STM32MP157C**. الجهاز بيشتغل على Linux مع loadable kernel modules. الـ sensor driver بيستخدم GPIOs من DT. المشكلة إن بعد `rmmod` للـ driver وعمل `insmod` تاني، الـ kernel بيطلع warning:
```
gpio gpiochip2: (spi2): gpio-5 (sensor-irq) status already claimed
```

#### المشكلة
الـ GPIO مش بيتحرر بشكل صح لما الـ device بيتشال من النظام. الـ notifier mechanism المسؤول عن cleanup مش بيتنفذ.

#### التحليل
في `gpiolib-of.h` في السطر ده:
```c
extern struct notifier_block gpio_of_notifier;
```

الـ `gpio_of_notifier` ده **`notifier_block`** بيتسجل مع الـ OF (Open Firmware) subsystem عشان يتلقى events لما devices بتتضاف أو بتتشال. لما device بيتشال، الـ notifier بيتنفذ عشان يعمل cleanup للـ GPIO mappings المرتبطة بالـ `device_node`.

المشكلة: الـ driver كان بيعمل `gpiod_get()` بس مش بيعمل `gpiod_put()` في الـ `remove()` function:

```c
static int sensor_probe(struct platform_device *pdev)
{
    priv->irq_gpio = gpiod_get(&pdev->dev, "sensor-irq", GPIOD_IN);
    /* ... */
}

static int sensor_remove(struct platform_device *pdev)
{
    /* مفيش gpiod_put()! ده هو البق */
    return 0;
}
```

لما الـ device بيتشال، الـ `gpio_of_notifier` بيتنفذ ويحاول يعمل cleanup، بس لأن الـ descriptor لسه claimed من غير ما يتحرر، بيحصل conflict في الـ probe التاني.

#### الحل
```c
static int sensor_remove(struct platform_device *pdev)
{
    struct sensor_priv *priv = platform_get_drvdata(pdev);

    /* تحرير الـ GPIO descriptor صح */
    if (priv->irq_gpio)
        gpiod_put(priv->irq_gpio);

    return 0;
}
```
أو الأحسن: استخدام `devm_gpiod_get()` عشان الـ kernel يعمل cleanup أوتوماتيك:
```c
static int sensor_probe(struct platform_device *pdev)
{
    /* devm_ بتعمل cleanup أوتوماتيك لما device يتشال */
    priv->irq_gpio = devm_gpiod_get(&pdev->dev, "sensor-irq", GPIOD_IN);
}
```

#### الدرس المستفاد
الـ `gpio_of_notifier` المُعلَن في `gpiolib-of.h` بيعمل housekeeping على مستوى الـ OF subsystem، بس مش بيعوّض عن الـ driver اللي مش بيعمل `gpiod_put()` بشكل صح. استخدام `devm_gpiod_get()` بدل `gpiod_get()` بيضمن إن الـ GPIO بيتحرر أوتوماتيك مع دورة حياة الـ device.

---

### السيناريو الرابع: Custom Board Bring-up على i.MX8MM — `of_gpiochip_instance_match()` وتعدد GPIO controllers متطابقة الاسم

#### العنوان
خطأ في GPIO lookup لما فيه اتنين `gpio-hog` nodes بنفس الاسم على i.MX8MM

#### السياق
مهندس hardware بيعمل bring-up لـ custom carrier board على **NXP i.MX8M Mini**. الـ board فيها اتنين SPI flash chips، كل واحد محتاج chip-select GPIO. الـ DT فيه اتنين `gpio-hog` بنفس الـ compatible. بعد boot، واحد من الـ flash chips مش بيتعرف عليه.

#### المشكلة
```bash
dmesg | grep spi
# spi-nor spi0.0: w25q128: detected
# spi-nor spi0.1: unrecognized JEDEC id bytes: ff ff ff
```
الـ chip التاني بيقرأ `0xFF` يعني مش بيتكلم، غالباً الـ CS GPIO مش بيتفعّل.

#### التحليل
في `gpiolib-of.h`:
```c
bool of_gpiochip_instance_match(struct gpio_chip *gc, unsigned int index);
```

الفانكشن دي بتتحقق من رقم الـ instance لـ GPIO chip معينة عشان يميّز بين controllers متعددة من نفس النوع. لما الـ GPIO subsystem بيحاول يحل GPIO reference من الـ DT، بيمر على الـ registered `gpio_chip`s ويستخدم `of_gpiochip_instance_match()` يتأكد إنه بيكلم الـ controller الصح مش أي controller.

المشكلة في الـ DT:
```dts
/* Controller الأول */
gpio_cs0: cs-gpio-0 {
    gpio-hog;
    gpios = <&gpio1 10 GPIO_ACTIVE_LOW>;
    output-high;
    line-name = "spi-cs0";
};

/* Controller التاني — نفس format لكن بـ index غلط */
gpio_cs1: cs-gpio-1 {
    gpio-hog;
    gpios = <&gpio1 10 GPIO_ACTIVE_LOW>;  /* نفس GPIO بالغلط! */
    output-high;
    line-name = "spi-cs1";
};
```

الـ `gpios = <&gpio1 10 ...>` في الاتنين بيشير لنفس الـ GPIO pin! لازم يكون:
```dts
gpios = <&gpio1 11 GPIO_ACTIVE_LOW>;  /* pin مختلف للـ CS1 */
```

لو الـ controllers كانوا منفصلين، `of_gpiochip_instance_match()` كانت هتتحقق من الـ `index` عشان تفرق بينهم. بس هنا المشكلة في الـ DT نفسه.

#### الحل
تصحيح الـ DT:
```dts
&gpio1 {
    spi-cs0-hog {
        gpio-hog;
        gpios = <RK_PB2 GPIO_ACTIVE_LOW>;  /* pin 10 */
        output-high;
        line-name = "spi0-cs";
    };

    spi-cs1-hog {
        gpio-hog;
        gpios = <RK_PB3 GPIO_ACTIVE_LOW>;  /* pin 11 — مختلف */
        output-high;
        line-name = "spi1-cs";
    };
};
```

التحقق بعد التصحيح:
```bash
gpioinfo | grep spi
# line  10: "spi0-cs"        output active-high [used]
# line  11: "spi1-cs"        output active-high [used]
```

#### الدرس المستفاد
الـ `of_gpiochip_instance_match()` موجودة عشان تتعامل مع سيناريو فيه اتنين controllers من نفس النوع. لكنها مش بتحل مشكلة تكرار الـ GPIO pin number في الـ DT. لما يفضل SPI device بيقرأ `0xFF`، أول خطوة هي التحقق من الـ CS GPIO بـ `gpioinfo` قبل ما تبص في الـ driver code.

---

### السيناريو الخامس: Automotive ECU على AM62x — `of_gpio_count()` وعدد GPIOs الديناميكي للـ CAN transceivers

#### العنوان
الـ driver بيفشل يحدد عدد CAN transceiver enable pins الصح على TI AM62x

#### السياق
فريق بيطور automotive ECU على **TI AM62x** (Sitara). الـ ECU فيه variable عدد من CAN transceivers (2 أو 4 حسب variant الـ PCB). الـ driver المفروض يقرأ عدد الـ `enable-gpios` من الـ DT ديناميكياً. بس الـ driver دايماً بيبلّغ عن 0 GPIOs.

#### المشكلة
```bash
dmesg | grep can-transceiver
# can-transceiver: found 0 enable GPIOs, expected 2 or 4
```
الـ CAN transceivers مش بتتفعّل، والـ CAN bus مش شغال.

#### التحليل
في `gpiolib-of.h`:
```c
int of_gpio_count(const struct fwnode_handle *fwnode, const char *con_id);
```

الفانكشن دي بتحسب عدد الـ GPIOs المعرّفة بـ `con_id` في الـ `fwnode`. الـ driver بيستخدمها:

```c
static int can_transceiver_probe(struct platform_device *pdev)
{
    struct fwnode_handle *fwnode = dev_fwnode(&pdev->dev);
    int n_gpios;

    /* بتحسب عدد الـ "enable-gpios" في الـ DT node */
    n_gpios = of_gpio_count(fwnode, "enable");
    if (n_gpios <= 0) {
        dev_err(&pdev->dev, "found %d enable GPIOs\n", n_gpios);
        return -EINVAL;
    }
}
```

الـ `of_gpio_count()` بتدور على property اسمها `enable-gpios` (بتضيف `-gpios` suffix للـ `con_id` تلقائياً). نبص على الـ DT:

```dts
can_transceiver: can-transceiver {
    compatible = "vendor,can-transceiver";
    /* المبرمج كتب الاسم غلط! */
    enable-gpio = <&gpio0 5 GPIO_ACTIVE_HIGH>,   /* مش "enable-gpios" */
                  <&gpio0 6 GPIO_ACTIVE_HIGH>;
};
```

المشكلة إن الاسم `enable-gpio` (بدون `s`) مش هو اللي بتدور عليه `of_gpio_count()`. الـ convention في Linux GPIO هو دايماً `*-gpios` (جمع).

لو `CONFIG_OF_GPIO` كانت مش متفعلة، الـ `of_gpio_count()` كانت هترجع `0` مباشرة من الـ stub:
```c
static inline int of_gpio_count(const struct fwnode_handle *fwnode,
                                const char *con_id)
{
    return 0;  /* الـ stub دايما بيرجع 0 */
}
```
بس هنا المشكلة في اسم الـ property مش في الـ config.

#### الحل
تصحيح الـ DT:
```dts
can_transceiver: can-transceiver {
    compatible = "vendor,can-transceiver";
    /* الاسم الصح: enable-gpios (بالـ s في الآخر) */
    enable-gpios = <&gpio0 5 GPIO_ACTIVE_HIGH>,
                   <&gpio0 6 GPIO_ACTIVE_HIGH>;
};
```

التحقق:
```bash
# قبل التصحيح
grep -r "enable-gpio" /sys/firmware/devicetree/base/can-transceiver/
# بعد التصحيح
grep -r "enable-gpios" /sys/firmware/devicetree/base/can-transceiver/
# المفروض تطلع القيم

# تأكيد شغل الـ CAN
ip link show can0
# can0: <NOARP,UP,LOWER_UP,ECHO> mtu 16 ...
```

إضافة debug في الـ driver لتتبع القيم:
```c
dev_dbg(&pdev->dev, "of_gpio_count returned: %d\n",
        of_gpio_count(fwnode, "enable"));
```

#### الدرس المستفاد
الـ `of_gpio_count()` بتدور بالظبط على property اسمها `<con_id>-gpios`. أي typo في اسم الـ property في الـ DT بيخلّيها ترجع `0` بدون أي error message واضحة. القاعدة الثابتة في Linux GPIO DT bindings: الـ property دايماً `*-gpios` (جمع، بالـ `s`)، حتى لو فيه GPIO واحد بس. الـ `of_gpio_count()` returning `0` مش معناها `CONFIG_OF_GPIO` مش متفعل — اتحقق من اسم الـ property الأول.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

دي أهم المقالات اللي بتشرح تطور الـ GPIO subsystem والعلاقته بالـ Device Tree في الـ Linux kernel:

| المقال | الأهمية |
|--------|---------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | المدخل الأساسي لفهم الـ GPIO API في الـ kernel |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | بيشرح التوجه نحو الـ descriptor-based interface وازاي الـ OF هيتكامل معاه |
| [gpiolib: introduce descriptor-based GPIO interface](https://lwn.net/Articles/528226/) | الـ patch الأول اللي قدم `gpio_desc` — أساس `of_find_gpio()` |
| [New descriptor-based GPIO interface](https://lwn.net/Articles/565662/) | شرح `gpiod_get()` والـ lookup من الـ device tree |
| [Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/) | توثيق رسمي للـ gpiod API الجديد |
| [Device trees II: The harder parts](https://lwn.net/Articles/573409/) | بيغوص في تفاصيل الـ device tree اللي بتأثر على الـ GPIO lookup |
| [gpio: Document GPIO hogging mechanism](https://lwn.net/Articles/641117/) | الـ hogging اللي بيستخدم `of_gpiochip_add()` |
| [gpiolib: Add GPIO name support](https://lwn.net/Articles/651500/) | إضافة `line-name` من الـ DT bindings |
| [gpio: improve support for shared GPIOs](https://lwn.net/Articles/1039548/) | تحسينات حديثة على الـ GPIO sharing عبر الـ OF |

---

### التوثيق الرسمي للـ Kernel

الملفات دي موجودة في الـ kernel source وبتوضح الـ subsystem من زوايا مختلفة:

#### Driver API Documentation

```
Documentation/driver-api/gpio/index.rst       ← الفهرس الرئيسي
Documentation/driver-api/gpio/intro.rst       ← مقدمة الـ GPIO subsystem
Documentation/driver-api/gpio/consumer.rst    ← كيفية استهلاك GPIO من driver
Documentation/driver-api/gpio/driver.rst      ← كيفية تسجيل gpio_chip
Documentation/driver-api/gpio/board.rst       ← GPIO mappings (DT, ACPI, platform data)
Documentation/driver-api/gpio/using-gpio.rst  ← استخدام GPIO من userspace
```

**الأهم للموضوع ده:** الملف `board.rst` بيشرح بالظبط إزاي الـ `of_find_gpio()` بتلاقي الـ GPIO من الـ device tree node باستخدام الـ `<function>-gpios` properties.

#### Device Tree Bindings

```
Documentation/devicetree/bindings/gpio/gpio.txt       ← الـ binding الأساسي
Documentation/devicetree/bindings/gpio/gpio-consumer.yaml
Documentation/devicetree/bindings/gpio/gpio-controller.yaml
```

الملف `gpio.txt` بيحدد:
- الـ `#gpio-cells` property
- الـ `<name>-gpios` naming convention
- الـ `gpio-ranges` للربط مع الـ pinctrl

#### Online Kernel Docs

- [GPIO Mappings (board.html)](https://www.kernel.org/doc/html/latest/driver-api/gpio/board.html)
- [GPIO Consumer Interface](https://docs.kernel.org/driver-api/gpio/consumer.html)
- [General Purpose I/O (index)](https://www.kernel.org/doc/html/latest/driver-api/gpio/index.html)

---

### الـ Source Code المرجعي

الملفات دي بتكوّن الـ OF GPIO subsystem كاملًا:

```
drivers/gpio/gpiolib-of.h    ← الـ header موضوع البحث
drivers/gpio/gpiolib-of.c    ← الـ implementation الكاملة
drivers/gpio/gpiolib.c       ← الـ core gpiolib
drivers/gpio/gpiolib.h       ← internal core header
include/linux/gpio/driver.h  ← gpio_chip definition
include/linux/of_gpio.h      ← legacy OF GPIO API
```

**على GitHub:**
- [gpiolib-of.c على torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpiolib-of.c)
- [gpiolib.c على torvalds/linux](https://github.com/torvalds/linux/blob/master/drivers/gpio/gpiolib.c)

---

### Mailing List Discussions

| الرابط | الموضوع |
|--------|---------|
| [GPIO marking pins in DT (linuxppc-dev)](https://linuxppc-dev.ozlabs.narkive.com/qp4tnrzh/gpio-marking-individual-pins-not-available-in-device-tree) | نقاش مهم عن تمييز الـ GPIO pins في الـ DT |
| [Using gpio from device tree on platform devices (kernelnewbies)](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html) | سؤال عملي عن `of_find_gpio()` على platform devices |
| [devm_gpiod_get usage to get gpio num (kernelnewbies)](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html) | ربط `devm_gpiod_get()` بالـ DT descriptor |

---

### موارد eLinux.org

| الصفحة | المحتوى |
|--------|---------|
| [Device Tree Reference](https://elinux.org/Device_Tree_Reference) | المرجع الشامل للـ DT syntax بما فيه الـ GPIO properties |
| [Device Tree Mysteries](https://elinux.org/Device_Tree_Mysteries) | شرح `#gpio-cells` وتفسير الـ phandle |
| [EBC Device Trees](https://elinux.org/EBC_Device_Trees) | أمثلة عملية على BeagleBone بالـ GPIO في الـ DT |

---

### الكتب المرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 1** — مقدمة الـ kernel modules
- **الفصل 14** — الـ device model وعلاقته بالـ firmware
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Ed.)
- **الفصل 17** — Devices and Modules
- بيشرح الـ platform devices وكيف الـ DT بيولد platform_device

#### Embedded Linux Primer — Christopher Hallinan (2nd Ed.)
- **الفصل 16** — Configuring the Linux Kernel
- **الفصل 14** — بيتكلم عن الـ device tree بشكل عملي لـ embedded targets

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **الفصل 6** — Device Drivers
- تغطية عميقة لـ bus/device model اللي الـ `of_gpiochip_add()` بيعتمد عليه

---

### Kernel Newbies — إصدارات مهمة

| الإصدار | التغيير |
|---------|---------|
| [Linux 6.2](https://kernelnewbies.org/Linux_6.2) | إضافة software node support لـ gpiolib |
| [Linux 6.6](https://kernelnewbies.org/Linux_6.6) | GPIO recovery support في pinctrl |
| [Linux 6.11](https://kernelnewbies.org/Linux_6.11) | GPIO PWM driver جديد |
| [Linux 6.13](https://kernelnewbies.org/Linux_6.13) | Aspeed G7 GPIO support |
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | تتبع كل تغييرات الـ GPIO عبر الإصدارات |

---

### search terms للبحث عن مزيد من المعلومات

لو عايز تعمق أكتر، الكلمات دي هتجيب نتايج دقيقة:

```
"gpiolib-of"                          ← الملف نفسه
"of_find_gpio"                        ← الدالة الرئيسية
"of_gpiochip_add"                     ← تسجيل الـ chip مع الـ OF
"gpio_of_notifier"                    ← الـ notifier block
"CONFIG_OF_GPIO"                      ← الـ Kconfig option
"gpio-cells device tree"              ← الـ DT binding
"gpiod_get device tree lookup"        ← الـ consumer API
"of_parse_phandle_with_args gpio"     ← دالة الـ phandle parsing
"gpio fwnode lookup"                  ← الـ firmware node abstraction
"gpiolib descriptor interface"        ← الـ gpiod API
site:lore.kernel.org gpio of_find_gpio
site:lore.kernel.org gpiolib-of
```
## Phase 8: Writing simple module

### الفكرة

**`of_gpiochip_add()`** هي الدالة اللي بتتحاكل لما أي GPIO chip بتتسجل في النظام وبيكون ليها device tree node. هي exported وبتتاكل في كل boot أو لما kernel module بيحمّل GPIO driver — ده بيخليها نقطة hook مثالية عشان نشوف أي GPIO chip بتتضاف للسيستم.

هنستخدم **kprobe** عشان نعمل hook على `of_gpiochip_add` ونطبع اسم الـ chip وعدد الـ GPIO lines بتاعتها.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_of_gpiochip_add.c
 * Hooks of_gpiochip_add() to log every GPIO chip registered via device tree.
 */

/* kprobes API — struct kprobe, register_kprobe, etc. */
#include <linux/kprobes.h>

/* pr_info(), pr_err() macros */
#include <linux/kernel.h>

/* module_init, module_exit, MODULE_LICENSE, etc. */
#include <linux/module.h>

/* struct gpio_chip definition */
#include <linux/gpio/driver.h>

/* struct device_node, of_node_full_name() */
#include <linux/of.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("GPIO Observer");
MODULE_DESCRIPTION("kprobe on of_gpiochip_add to log DT-based GPIO chips");

/*
 * pre_handler — بيتنادى قبل ما of_gpiochip_add تشتغل.
 * regs بيحتوي على registers في وقت الـ probe.
 * الـ argument الأول (RDI على x86-64) هو pointer لـ struct gpio_chip.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * على x86-64: الـ argument الأول بيتبعت في RDI.
     * على ARM64: بيتبعت في x0.
     * regs_get_kernel_argument() بتجيب الـ argument رقم N بشكل portable.
     */
    struct gpio_chip *chip =
        (struct gpio_chip *)regs_get_kernel_argument(regs, 0);

    if (!chip)
        return 0;

    /*
     * chip->label: اسم الـ GPIO controller (مثلاً "gpio-bcm2835").
     * chip->ngpio: عدد الـ GPIO lines اللي بيوفرها الـ chip.
     * chip->gpiodev->dev: الـ device المرتبط بيه، بنطبع منه الـ DT node path.
     */
    pr_info("[of_gpiochip_add] chip='%s' ngpio=%u node='%s'\n",
            chip->label ? chip->label : "(null)",
            chip->ngpio,
            chip->gpiodev ? dev_name(&chip->gpiodev->dev) : "(no dev)");

    /* إرجاع 0 يعني "كمّل تنفيذ الدالة الأصلية بشكل طبيعي" */
    return 0;
}

/*
 * struct kprobe — بتعرّف نقطة الـ hook.
 * symbol_name: اسم الدالة المستهدفة كـ string.
 * pre_handler: الـ callback اللي بيتنادى قبل تنفيذ أول instruction في الدالة.
 */
static struct kprobe kp = {
    .symbol_name = "of_gpiochip_add",
    .pre_handler = handler_pre,
};

static int __init kprobe_gpiochip_init(void)
{
    int ret;

    /*
     * register_kprobe: بتسجّل الـ hook في الكيرنل.
     * لو الدالة مش موجودة أو مش قابلة للـ probe، بترجع error.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[of_gpiochip_add kprobe] register failed: %d\n", ret);
        return ret;
    }

    pr_info("[of_gpiochip_add kprobe] planted at %p\n", kp.addr);
    return 0;
}

static void __exit kprobe_gpiochip_exit(void)
{
    /*
     * unregister_kprobe: ضروري جداً في exit عشان تشيل الـ hook.
     * لو ما اتعملتش، أي GPIO chip تتضاف بعد unload الـ module هتعمل crash
     * لأن pre_handler هيتنادى وكود الـ module مش موجود في الذاكرة.
     */
    unregister_kprobe(&kp);
    pr_info("[of_gpiochip_add kprobe] removed\n");
}

module_init(kprobe_gpiochip_init);
module_exit(kprobe_gpiochip_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|--------|
| `<linux/kprobes.h>` | بيعرّف `struct kprobe`، `register_kprobe()`، `regs_get_kernel_argument()` |
| `<linux/kernel.h>` | الـ `pr_info()` و `pr_err()` |
| `<linux/module.h>` | الـ `module_init()` / `module_exit()` و macros الـ metadata |
| `<linux/gpio/driver.h>` | الـ `struct gpio_chip` اللي بنقرأ منها `label` و `ngpio` |
| `<linux/of.h>` | الـ device tree helpers زي `of_node_full_name()` |

---

#### الـ `pre_handler`

**الـ `regs_get_kernel_argument(regs, 0)`** بتجيب الـ argument الأول بشكل portable على أي architecture (x86-64, ARM64, RISC-V). ده أحسن من قراءة `regs->di` مباشرة لأنه بيتغير من architecture لأخرى.

**الـ `chip->label`** بيطبع اسم الـ GPIO controller زي `"gpio-bcm2835"` أو `"stm32-gpio"` — ده بيوضح أي driver بيضيف الـ chip دي.

**الـ return 0** في `pre_handler` معناه "كمّل تنفيذ الدالة الأصلية" — لو رجعنا غير صفر، الكيرنل هيتجاهل الدالة الأصلية، وده مش اللي بنعمله هنا.

---

#### الـ `struct kprobe`

**الـ `.symbol_name`** بيخلي الكيرنل يحلّ العنوان تلقائياً من الـ kallsyms بدل ما إحنا نوفر عنوان hardcoded.

**الـ `.pre_handler`** هو الـ callback الوحيد المطلوب — بيتنادى قبل أول instruction في الدالة المستهدفة، يعني الـ arguments لسه ما اتغيرتش.

---

#### الـ `module_init` و `module_exit`

**الـ `register_kprobe`** في `init` بتغرس breakpoint في كود `of_gpiochip_add` في الذاكرة. لو الدالة مش موجودة (مثلاً `CONFIG_OF_GPIO` مش enabled)، بترجع `-ENOENT` وإحنا بنتعامل معاها بـ error print ورجوع من `init`.

**الـ `unregister_kprobe`** في `exit` ضرورية لتجنب use-after-free: لو شيلنا الـ module وسابنا الـ hook، الكيرنل هيحاول يرجع لكود مش موجود في الذاكرة وهيعمل kernel panic.

---

### كيفية التجربة

```bash
# بناء الـ module
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod kprobe_of_gpiochip_add.ko

# مشاهدة الـ log (لو في GPIO driver بيتحمّل أو موجود أصلاً)
sudo dmesg | grep "of_gpiochip_add"

# إجبار إعادة تحميل GPIO driver لتفعيل الـ hook
sudo modprobe -r gpio_mockup && sudo modprobe gpio_mockup

# إزالة الـ module
sudo rmmod kprobe_of_gpiochip_add
```

**ملاحظة:** `of_gpiochip_add` بتتنادى فقط لما GPIO chip عندها device tree node. على أجهزة بدون DT (زي بعض أجهزة x86)، هتحتاج تستخدم `gpio_mockup` أو تشتغل على Raspberry Pi / BeagleBone عشان تشوف النتيجة.
