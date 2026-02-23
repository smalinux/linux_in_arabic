## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

**`include/linux/of_gpio.h`** هو header بسيط جداً — بالكاد فيه سطور — لكنه يمثل نقطة وصل بين عالمين مهمين جداً في الـ Linux kernel: عالم **Open Firmware / Device Tree** وعالم **GPIO**.

---

### الـ Subsystem

الملف جزء من **GPIO SUBSYSTEM**، تحت إشراف:
- Linus Walleij
- Bartosz Golaszewski

**الـ mailing list:** `linux-gpio@vger.kernel.org`

---

### قصة المشكلة — ELI5

تخيل إنك بتبني جهاز embedded زي Raspberry Pi أو router. الجهاز ده فيه pins كتير — بعضها بيتحكم في LED، بعضها بيقرأ زرار، بعضها بيتكلم مع sensor. الـ pins دي اسمها **GPIO (General Purpose Input/Output)**.

دلوقتي السؤال: **الـ kernel بيعرف إيه الـ pin رقم كام مثلاً؟ وبيعرف إيه إنه active-low ولا active-high؟**

في الأيام القديمة — الـ kernel كان بيتعرف على التفاصيل دي من **hardcoded C code**. يعني كل بورد كان ليه `board_file.c` مكتوب فيه "الـ LED على GPIO 17". ده كان كارثة — أي تغيير في الـ hardware يحتاج تغيير في الـ kernel source وإعادة compile.

الحل كان **Device Tree (DT)** — ملف نص خارجي (`.dts`) بيوصف الـ hardware. بدل ما تـhardcode في C، تكتب:

```dts
leds {
    led-status {
        gpios = <&gpio1 17 GPIO_ACTIVE_LOW>;
    };
};
```

الـ kernel بيقرأ الملف ده عند الـ boot. لكن محتاج **bridge** بين الـ Device Tree parser وبين الـ GPIO API. ده بالظبط دور `of_gpio.h` — **OF** هنا اختصار **Open Firmware**، اللي هو الاسم التاريخي لنظام الـ Device Tree.

---

### الهدف من الملف

الملف بيعرّف دالة واحدة بس:

```c
int of_get_named_gpio(const struct device_node *np,
                      const char *list_name, int index);
```

**الـ `device_node`** ده هو representation الـ node بتاع الـ Device Tree في الـ memory — يعني لما الـ kernel يقرأ الـ DTS، كل node بيتحول لـ `struct device_node`.

**الوظيفة ببساطة:**
> إديني node من الـ Device Tree + اسم الـ property + رقم index → أنا هارجعلك رقم الـ GPIO.

مثال: driver بتاع LED يقول `of_get_named_gpio(np, "gpios", 0)` فيرجعله رقم GPIO 17 علشان يتعامل معاه.

---

### ليه الملف بسيط جداً؟

لأن الـ implementation الحقيقية انتقلت من هنا لـ `drivers/gpio/gpiolib-of.c`. الـ header ده أصبح **legacy thin wrapper** — يعني موجود للـ backward compatibility مع الـ drivers القديمة اللي لسه بتستخدمه.

الـ modern API بقى في `include/linux/gpio/consumer.h` مع `devm_gpiod_get()` اللي أذكى وأأمن.

---

### الـ CONFIG_OF_GPIO Guard

الملف بيستخدم compile-time guard:

```c
#ifdef CONFIG_OF_GPIO
    // الـ real implementation
    extern int of_get_named_gpio(...);
#else
    // stub بيرجع -ENOSYS
    static inline int of_get_named_gpio(...) { return -ENOSYS; }
#endif
```

ده يعني: لو الـ platform مش بتستخدم Device Tree (زي x86 القديم)، الـ drivers هتـlink تمام بس الـ function هترجع error بدل ما الـ build يفشل.

---

### الـ ASCII Diagram — كيف بتتفاعل الأجزاء

```
  Device Tree Source (.dts)
         │
         ▼
  OF/DT Parser (drivers/of/)
         │  device_node structs
         ▼
  of_get_named_gpio()          ← هنا include/linux/of_gpio.h
  [gpiolib-of.c implementation]
         │  GPIO number
         ▼
  GPIO Core API (gpiolib.c)
         │
         ▼
  Hardware GPIO Controller Driver
  (e.g., gpio-aspeed.c, gpio-pl061.c)
         │
         ▼
  Physical GPIO Pin (LED, Button, etc.)
```

---

### الملفات المرتبطة اللي المبرمج لازم يعرفها

| الملف | الدور |
|---|---|
| `include/linux/of_gpio.h` | الـ header موضوع شرحنا (legacy OF GPIO API) |
| `include/linux/gpio/consumer.h` | الـ modern GPIO consumer API (الأحدث والمفضل) |
| `include/linux/gpio/driver.h` | الـ API بتاع كتابة GPIO controller driver |
| `include/linux/of.h` | الـ Device Tree core API و `struct device_node` |
| `include/linux/gpio.h` | Legacy GPIO API (integer-based، قديم) |
| `drivers/gpio/gpiolib-of.c` | **الـ implementation الحقيقية** لكل دوال OF-GPIO |
| `drivers/gpio/gpiolib.c` | الـ GPIO subsystem core |
| `drivers/gpio/gpiolib-legacy.c` | الـ legacy integer-based GPIO API |
| `drivers/gpio/gpiolib-devres.c` | الـ device resource managed GPIO helpers |
| `Documentation/devicetree/bindings/gpio/` | الـ DT bindings specs للـ GPIO controllers |
| `Documentation/driver-api/gpio/` | الـ driver developer documentation |

---

### ملفات الـ Subsystem الأساسية

**Core:**
- `drivers/gpio/gpiolib.c` — قلب الـ GPIO subsystem
- `drivers/gpio/gpiolib-of.c` — OF/DT integration
- `drivers/gpio/gpiolib-acpi-core.c` — ACPI integration (لـ x86)
- `drivers/gpio/gpiolib-cdev.c` — character device interface (`/dev/gpiochipN`)
- `drivers/gpio/gpiolib-sysfs.c` — sysfs interface

**Headers:**
- `include/linux/gpio/consumer.h` — consumer API
- `include/linux/gpio/driver.h` — driver API
- `include/linux/gpio/machine.h` — board lookup tables
- `include/linux/of_gpio.h` — legacy OF helper

**Hardware Drivers (أمثلة):**
- `drivers/gpio/gpio-aspeed.c` — Aspeed BMC GPIO
- `drivers/gpio/gpio-pl061.c` — ARM PrimeCell GPIO
- `drivers/gpio/gpio-pxa.c` — Intel PXA GPIO

---

### خلاصة

**`of_gpio.h`** هو جسر بسيط (ولكن مهم تاريخياً) بين الـ Device Tree وبين الـ GPIO API. الـ driver القديم كان يقوله "روح في الـ Device Tree، جيبلي رقم الـ GPIO اللي اسمه كذا"، والـ function دي كانت بتعمل ده. الـ modern code بقى بيستخدم `devm_gpiod_get()` بدلاً منها، لكن الملف ده لسه موجود علشان مئات الـ drivers القديمة اللي بتستخدمه.
## Phase 2: شرح الـ OF-GPIO (Open Firmware GPIO) Framework

---

### المشكلة — ليه الـ OF-GPIO موجود أصلاً؟

في أي SoC بتشتغل عليه — i.MX6، AM335x، BCM2835 — فيه GPIO controller واحد أو أكتر. الـ driver بتاع الـ GPIO controller بيعرف يتحكم في الـ pins، لكن المشكلة هي: **إزاي الـ driver بتاع الـ consumer** (مثلاً driver بتاع شريحة SPI أو LED) يعرف يقول "أنا عايز GPIO رقم 5 من الـ controller ده"؟

**قبل الـ Device Tree:**
كل board كان عنده `board_init.c` مليان hardcoded numbers زي `gpio_request(GPIO_PD5, ...)` — ده كان كارثة لأن نفس الـ driver مش هيشتغل على board تاني من غير تعديل في الـ C code.

**بعد الـ Device Tree:**
الـ board description اتحولت لـ `.dts` files، بس محتاجين طريقة تربط بين الـ DT property زي `reset-gpios = <&gpio2 5 GPIO_ACTIVE_LOW>` والـ actual GPIO line في الـ kernel — ده بالظبط اللي بيعمله **OF-GPIO**.

**الـ OF-GPIO subsystem** هو الـ glue layer اللي بيترجم الـ phandle + specifier الموجودين في الـ Device Tree لـ GPIO descriptor (`struct gpio_desc`) قابل للاستخدام في أي driver.

---

### الحل — الـ approach بتاع الـ kernel

الـ kernel بيستخدم نظام **phandle + cells** في الـ Device Tree:

```dts
/* في ملف .dts */
gpio2: gpio@20a0000 {
    compatible = "fsl,imx6q-gpio";
    #gpio-cells = <2>;   /* <pin_number  flags> */
    ...
};

my_device: sensor@0 {
    reset-gpios = <&gpio2 5 GPIO_ACTIVE_LOW>;
    /*               ^      ^       ^
                  phandle  pin    polarity  */
};
```

الـ `of_get_named_gpio()` بتاخد الـ `device_node` بتاع `my_device` واسم الـ property (`"reset-gpios"`) وtraces الـ phandle لـ `gpio2`، بعدين بتترجم الـ specifier `<5 GPIO_ACTIVE_LOW>` لـ GPIO number عن طريق الـ `of_xlate` callback الموجود في الـ `gpio_chip`.

---

### الـ Big Picture Architecture

```
                    ┌─────────────────────────────────────────┐
                    │           Device Tree (.dts)             │
                    │  reset-gpios = <&gpio2 5 GPIO_ACTIVE_LOW>│
                    └───────────────┬─────────────────────────┘
                                    │ of_get_named_gpio()
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    OF-GPIO Layer (of_gpio.h)                   │
│                                                                │
│  • يفك ضغط الـ phandle من الـ DT property                    │
│  • يستدعي of_find_node_by_phandle() للوصول لـ gpio_chip       │
│  • يستدعي of_xlate() لتحويل (pin=5, flags=ACTIVE_LOW)         │
│    لـ GPIO number مطلق (global GPIO number)                   │
└───────────────────────────────────┬───────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│                    GPIO Subsystem (gpiolib)                    │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              struct gpio_chip  (gpio/driver.h)           │  │
│  │  .label          = "gpio2"                              │  │
│  │  .ngpio          = 32                                   │  │
│  │  .base           = 32  (لو dynamic: -1)                 │  │
│  │  .of_gpio_n_cells= 2                                    │  │
│  │  .of_xlate       = imx_gpio_of_xlate()  ◄── driver impl │  │
│  │  .get()  / .set() / .direction_input()  ◄── driver impl │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬───────────────────────────┘
                                    │
                                    ▼
┌───────────────────────────────────────────────────────────────┐
│              Hardware GPIO Controller (e.g. i.MX6 GPIO2)      │
│           Memory-mapped registers @ 0x020A_0000               │
│    DR (data)  |  GDIR (direction)  |  PSR (pad status)        │
└───────────────────────────────────────────────────────────────┘
```

**المستوى الأعلى: الـ consumers** (أي driver محتاج GPIO):
- `drivers/net/phy/...` → reset pin
- `drivers/spi/...` → chip select
- `drivers/leds/...` → LED pin

**المستوى الأدنى: الـ providers** (GPIO controller drivers):
- `drivers/gpio/gpio-mxc.c` → i.MX GPIO
- `drivers/gpio/gpio-pl061.c` → ARM PrimeCell GPIO
- `drivers/gpio/gpio-pca953x.c` → I2C GPIO expander

---

### التشبيه الحقيقي — نظام العناوين البريدية

تخيل إنك بتبعت جواب (GPIO request) لشخص في مدينة تانية.

| المفهوم الحقيقي | التشبيه |
|---|---|
| الـ `device_node` بتاع الـ consumer | المرسِل |
| الـ DT property `reset-gpios = <&gpio2 5 1>` | العنوان المكتوب على الجواب |
| الـ phandle `&gpio2` | اسم المدينة / المنطقة |
| الـ cells `<5 1>` | رقم الشارع والشقة |
| الـ `of_get_named_gpio()` | مكتب البريد اللي بيترجم العنوان لإحداثيات GPS |
| الـ `of_xlate()` | نظام ترقيم الشوارع المحلي لكل مدينة |
| الـ `gpio_chip` | العمارة نفسها |
| الـ `gpio_desc` اللي بترجع | المفتاح الفعلي للشقة |
| الـ `.get()` / `.set()` | فتح/قفل الباب |

**المهم في التشبيه ده:**
- كل مدينة (GPIO controller) بيكون ليها نظام ترقيم خاص بيها (`of_xlate`) — بعض الـ controllers بتستخدم `(instance, pin, flags)` كـ 3 cells، وبعضها `(pin, flags)` كـ 2 cells. مكتب البريد (OF-GPIO layer) بيتعامل مع التنوع ده بشكل transparent.
- الـ consumer مش محتاج يعرف أي حاجة عن نظام الترقيم الداخلي.

---

### الـ Core Abstraction — الفكرة المحورية

**الـ OF-GPIO subsystem بيقدم abstraction layer واحدة فقط:**

> تحويل **اسم property في DT** + **index** → **GPIO number** مقبول من الـ gpiolib.

الـ API بتاعه بسيط جداً:

```c
/*
 * np        : device_node بتاع الـ consumer (مش الـ GPIO controller)
 * propname  : اسم الـ property في الـ DTS مثلاً "reset-gpios"
 * index     : لو الـ property فيها أكتر من GPIO، نأخد الـ index ده
 *
 * returns   : GPIO number (global)، أو negative error
 */
int of_get_named_gpio(const struct device_node *np,
                      const char *propname, int index);
```

**مثال من driver حقيقي:**

```c
/* من driver بتاع PHY أو أي device بيحتاج reset */
static int my_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    int reset_gpio;

    /* بيقرأ من DTS: reset-gpios = <&gpio3 12 GPIO_ACTIVE_LOW> */
    reset_gpio = of_get_named_gpio(np, "reset-gpios", 0);
    if (reset_gpio < 0) {
        dev_err(&pdev->dev, "failed to get reset GPIO: %d\n", reset_gpio);
        return reset_gpio;
    }

    /* بعد كده بنستخدم gpiolib API العادية */
    gpio_request(reset_gpio, "my-reset");
    gpio_direction_output(reset_gpio, 0); /* assert reset */
}
```

> ملاحظة: الـ modern kernel بيفضل استخدام `devm_gpiod_get()` بدل `of_get_named_gpio()` مباشرة، لكن الأخيرة موجودة للـ legacy code وللفهم.

---

### الـ struct gpio_chip وعلاقته بـ OF-GPIO

الـ `struct gpio_chip` هو الـ central struct اللي بيربط الـ OF-GPIO بالـ hardware. فيه section مخصص لـ OF:

```
struct gpio_chip
┌────────────────────────────────────────────────┐
│ label          : "gpio2"                        │
│ ngpio          : 32                             │
│ base           : 32                             │
│ parent         : struct device *                │
│ fwnode         : struct fwnode_handle *  ◄──┐  │
│                                              │  │
│ .get()                                       │  │
│ .set()                                    linked │
│ .direction_input()                        to DT  │
│ .direction_output()                          │  │
│                                              │  │
│ [CONFIG_OF_GPIO section]                     │  │
│ of_gpio_n_cells : 2                          │  │
│ of_xlate        : imx_gpio_xlate() ◄─────────┘  │
│ of_node_instance_match : NULL (optional)        │
└────────────────────────────────────────────────┘
```

الـ `of_xlate` هو الـ callback اللي بيأخد `struct of_phandle_args`:

```c
struct of_phandle_args {
    struct device_node *np;      /* node بتاع الـ GPIO controller */
    int args_count;              /* عدد الـ cells = of_gpio_n_cells */
    uint32_t args[MAX_PHANDLE_ARGS]; /* القيم: {pin, flags} */
};
```

وبيرجع الـ GPIO offset داخل الـ chip:

```c
/* مثال implementation في gpio-mxc.c */
static int imx_gpio_of_xlate(struct gpio_chip *gc,
                              const struct of_phandle_args *gpiospec,
                              u32 *flags)
{
    /* gpiospec->args[0] = pin number */
    /* gpiospec->args[1] = flags (ACTIVE_LOW, etc.) */

    if (flags)
        *flags = gpiospec->args[1];

    return gpiospec->args[0]; /* offset داخل الـ chip */
}
```

---

### الـ Parsing Flow — خطوة بخطوة

```
of_get_named_gpio(np, "reset-gpios", 0)
        │
        ▼
of_parse_phandle_with_args(np, "reset-gpios", "#gpio-cells", 0, &args)
        │   • بيبحث في properties بتاع np
        │   • بيلاقي: reset-gpios = <&gpio2 5 1>
        │   • بيتبع الـ phandle &gpio2
        │   • بيقرأ #gpio-cells = 2
        │   • args.np = gpio2_node
        │   • args.args = {5, 1}
        │
        ▼
of_find_gpiochip_by_fwnode(args.np->fwnode)
        │   • بيبحث في قائمة الـ registered gpio_chips
        │   • بيلاقي gpio_chip بتاع gpio2
        │
        ▼
gc->of_xlate(gc, &args, &flags)
        │   • بيرجع offset = 5
        │
        ▼
gpiochip_offset_to_irq / gpio number = gc->base + offset
        │   = 32 + 5 = 37  (global GPIO number)
        │
        ▼
return 37
```

---

### ما الـ OF-GPIO بيملكه مقابل ما بيديه للـ drivers

| المسؤولية | الـ OF-GPIO Layer يملكها؟ | الـ GPIO Controller Driver يتولاها؟ |
|---|---|---|
| قراءة الـ DT property وتفكيك الـ phandle | نعم | لا |
| البحث عن الـ gpio_chip المناسب من الـ fwnode | نعم | لا |
| تحديد عدد الـ cells (`#gpio-cells`) | لا (الـ DT بيحدده) | الـ driver بيضبط `of_gpio_n_cells` |
| ترجمة الـ cells لـ offset (`of_xlate`) | لا | نعم، لازم يوفره |
| التحكم الفعلي في الـ hardware (get/set) | لا | نعم، لازم يوفره |
| إدارة الـ IRQ domain | لا | اختياري عبر `gpio_irq_chip` |
| الـ fallback لو `CONFIG_OF_GPIO` مش موجود | نعم (بترجع `-ENOSYS`) | لا |

---

### الـ Kconfig Guard — ليه مهم

```c
#ifdef CONFIG_OF_GPIO

extern int of_get_named_gpio(const struct device_node *np,
                             const char *list_name, int index);

#else

static inline int of_get_named_gpio(...)
{
    return -ENOSYS; /* Not supported */
}

#endif
```

ده pattern كلاسيكي في الـ kernel — الـ platforms اللي مش بتستخدم Device Tree (زي بعض الـ x86 أو الـ SPARC legacy) مش بيكون عندها `CONFIG_OF_GPIO`، فبدل ما الـ driver يفشل في الـ compile، الـ function بترجع `-ENOSYS` والـ driver ممكن يعمل fallback أو يكمل بدون الـ GPIO.

الـ drivers اللي بتستخدم الـ OF-GPIO بتكتب:

```c
gpio = of_get_named_gpio(np, "reset-gpios", 0);
if (gpio == -ENOSYS) {
    /* DT غير موجود أو OF-GPIO مش مفعّل، نكمل بدون GPIO */
    dev_warn(dev, "reset GPIO not available, skipping\n");
} else if (gpio < 0) {
    return gpio; /* error حقيقي */
}
```

---

### علاقة OF-GPIO بالـ Subsystems التانية

**الـ Device Tree (OF) subsystem** — `include/linux/of.h`:
اللي بيوفر الـ `device_node`، الـ `property`، وفنكشن زي `of_parse_phandle_with_args()`. الـ OF-GPIO بيبني فوقه مباشرة.

**الـ GPIO subsystem (gpiolib)** — `include/linux/gpio/driver.h`:
اللي بيوفر الـ `struct gpio_chip` وregistration API زي `gpiochip_add_data()`. الـ OF-GPIO بيستخدمه لما بيلاقي الـ chip ويعمل الترجمة.

**الـ Pin Control subsystem (pinctrl)**:
بعض الـ GPIO controllers هي نفسها pin controllers (مثلاً i.MX IOMUX). الـ `gpio_chip` ممكن يكون linked لـ `pinctrl_dev` عن طريق `add_pin_ranges()`.

الرسم ده بيوضح العلاقة:

```
[ Consumer Driver ]
       │
       │ of_get_named_gpio("reset-gpios", 0)
       ▼
[ OF-GPIO Layer ]  ←──── يعتمد على ────→  [ OF / DT Subsystem ]
       │                                    (of_parse_phandle_with_args)
       │ يستخدم gpio_chip
       ▼
[ gpiolib (GPIO Core) ]
       │
       │ يستدعي of_xlate() و get()/set()
       ▼
[ GPIO Controller Driver ]
       │
       ▼
[ Hardware Registers ]
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### ملاحظة أولية: حجم الـ File

الـ `of_gpio.h` نفسه صغير جداً — بيحتوي على function واحدة بس: `of_get_named_gpio()`. لكن الـ file ده بيكون **واجهة** لمنظومة أكبر بكتير، وبيعتمد على structs معرّفة في:

- `linux/of.h` — الـ Device Tree infrastructure
- `linux/gpio/driver.h` — الـ GPIO chip abstraction

الـ Phase ده هيغطي كل العلاقات دي بعمق.

---

### الـ Config Options والـ Flags — Cheatsheet

#### Config Options

| Option | المعنى | التأثير على `of_gpio.h` |
|---|---|---|
| `CONFIG_OF_GPIO` | تفعيل دعم GPIO عبر Device Tree | بيفعّل `of_get_named_gpio()` الحقيقية، أو بيرجّع `-ENOSYS` |
| `CONFIG_OF` | تفعيل Open Firmware / Device Tree | شرط أساسي لـ `CONFIG_OF_GPIO` |
| `CONFIG_OF_DYNAMIC` | دعم تعديل الـ Device Tree في runtime | بيفعّل reference counting لـ `device_node` |
| `CONFIG_OF_KOBJ` | دمج الـ Device Tree nodes مع الـ kobject system | بيضيف `kobject` جوه `device_node` |
| `CONFIG_GPIOLIB_IRQCHIP` | دعم IRQ controller جوه الـ gpio_chip | بيضيف `gpio_irq_chip` جوه `gpio_chip` |
| `CONFIG_IRQ_DOMAIN_HIERARCHY` | دعم hierarchical IRQ domains | بيضيف fields للـ parent domain في `gpio_irq_chip` |
| `CONFIG_GENERIC_MSI_IRQ` | دعم MSI interrupts | بيضيف `msiinfo` لـ `union gpio_irq_fwspec` |

#### الـ `device_node` Flags (`_flags` field)

| Flag | القيمة | المعنى |
|---|---|---|
| `OF_DYNAMIC` | `1` | الـ node والـ properties اتحجزت بـ `kmalloc` |
| `OF_DETACHED` | `2` | الـ node انفصلت عن الـ Device Tree |
| `OF_POPULATED` | `3` | الـ device اتعمل منها بالفعل |
| `OF_POPULATED_BUS` | `4` | اتعمل platform bus للـ children |
| `OF_OVERLAY` | `5` | اتحجزت لـ overlay |
| `OF_OVERLAY_FREE_CSET` | `6` | موجودة في overlay cset بيتحذف دلوقتي |

#### الـ GPIO Direction Constants

| Constant | القيمة | المعنى |
|---|---|---|
| `GPIO_LINE_DIRECTION_IN` | `1` | الـ pin شغّال input |
| `GPIO_LINE_DIRECTION_OUT` | `0` | الـ pin شغّال output |

---

### الـ Structs المهمة

#### 1. `struct device_node`
**الغرض:** تمثّل node واحدة في الـ Device Tree. كل جهاز أو controller في الـ DT عنده `device_node` خاص بيه.

```c
struct device_node {
    const char        *name;        /* اسم الـ node (e.g., "gpio") */
    phandle           phandle;      /* رقم فريد للـ node في الـ DT */
    const char        *full_name;   /* المسار الكامل (e.g., "/soc/gpio@0") */
    struct fwnode_handle fwnode;    /* ربط بالـ firmware node abstraction */

    struct property   *properties;  /* linked list للـ properties */
    struct property   *deadprops;   /* properties اتشالت (للـ dynamic DT) */
    struct device_node *parent;     /* الـ parent node */
    struct device_node *child;      /* أول child */
    struct device_node *sibling;    /* الـ sibling التالي */

    unsigned long     _flags;       /* OF_DYNAMIC, OF_POPULATED, إلخ */
    void              *data;        /* بيانات خاصة بالـ platform */
};
```

**العلاقات:**
- بيحتوي على linked list من `struct property`
- بيتربط بـ `struct fwnode_handle` لتجريد الـ firmware layer
- الـ `of_get_named_gpio()` بتاخد pointer ليه كـ input

---

#### 2. `struct property`
**الغرض:** تمثّل property واحدة جوه الـ Device Tree node (زي `gpios = <&gpio0 5 0>`).

```c
struct property {
    char           *name;   /* اسم الـ property (e.g., "gpios", "reset-gpios") */
    int            length;  /* حجم الـ value بالـ bytes */
    void           *value;  /* قيمة الـ property (big-endian raw bytes) */
    struct property *next;  /* التالي في الـ linked list */
};
```

**العلاقات:**
- مرتبطة بـ `device_node` عبر `properties` pointer
- الـ `of_get_named_gpio()` بتبحث في الـ properties دي عن اسم الـ GPIO list

---

#### 3. `struct of_phandle_args`
**الغرض:** بتمثّل نتيجة parse لـ phandle مع arguments. ده اللي بيتبني من الـ DT property زي `gpios = <&gpio0 5 GPIO_ACTIVE_LOW>`.

```c
struct of_phandle_args {
    struct device_node *np;                    /* الـ node اللي الـ phandle بيشير ليها */
    int                args_count;             /* عدد الـ cells (عادةً 2) */
    uint32_t           args[MAX_PHANDLE_ARGS]; /* الـ cells نفسها (offset, flags) */
};
```

**العلاقات:**
- بتتعمل داخلياً في `of_get_named_gpio()` لما بتعمل parse للـ GPIO specifier
- الـ `gpio_chip.of_xlate()` callback بتاخدها كـ input لتحويل `args` لـ GPIO number حقيقي

---

#### 4. `struct gpio_chip`
**الغرض:** تجريد لـ GPIO controller كامل. كل driver بيكتب struct بيملأ الـ function pointers دي.

```c
struct gpio_chip {
    const char          *label;          /* اسم الـ controller */
    struct gpio_device  *gpiodev;        /* الـ internal state (opaque) */
    struct device       *parent;         /* الـ physical device */
    struct fwnode_handle *fwnode;        /* firmware node */

    /* --- Operations --- */
    int  (*request)(struct gpio_chip *gc, unsigned int offset);
    void (*free)(struct gpio_chip *gc, unsigned int offset);
    int  (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)(struct gpio_chip *gc, unsigned int offset);
    int  (*set)(struct gpio_chip *gc, unsigned int offset, int value);

    int  base;        /* أول GPIO number (deprecated: استخدم -1 للـ dynamic) */
    u16  ngpio;       /* عدد الـ GPIO lines */
    bool can_sleep;   /* true لو الـ I2C/SPI expander */

    /* CONFIG_OF_GPIO fields */
    unsigned int  of_gpio_n_cells;  /* عدد cells في الـ DT specifier */
    bool (*of_node_instance_match)(struct gpio_chip *gc, unsigned int i);
    int  (*of_xlate)(struct gpio_chip *gc,
                     const struct of_phandle_args *gpiospec, u32 *flags);

    /* CONFIG_GPIOLIB_IRQCHIP */
    struct gpio_irq_chip irq;
};
```

**العلاقات:**
- بيحتوي على `gpio_irq_chip` embedded
- بيتربط بـ `device_node` عبر `fwnode` pointer
- الـ `of_xlate` callback بتستخدم `of_phandle_args`

---

#### 5. `struct gpio_irq_chip`
**الغرض:** بتضيف IRQ controller functionality للـ `gpio_chip`. يعني كل GPIO line ممكن تكون interrupt source.

```c
struct gpio_irq_chip {
    struct irq_chip    *chip;           /* الـ IRQ chip operations */
    struct irq_domain  *domain;         /* mapping بين hwirq و Linux IRQ */
    irq_flow_handler_t  handler;        /* الـ IRQ handler (e.g., handle_edge_irq) */
    unsigned int        default_type;   /* IRQ_TYPE_EDGE_RISING, etc. */
    bool                threaded;       /* true لو I2C/SPI chip محتاج threaded IRQ */
    bool                initialized;    /* guard flag قبل ما يتعمل init */
    unsigned long      *valid_mask;     /* bitmask للـ GPIOs اللي ممكن تطلع interrupt */
    unsigned int        num_parents;    /* عدد الـ parent interrupt lines */
    unsigned int       *parents;        /* array بـ parent IRQ numbers */
};
```

**العلاقات:**
- embedded جوه `gpio_chip`
- بيتربط بـ `irq_domain` للـ Linux IRQ subsystem
- بيتربط بـ `irq_chip` للـ hardware operations

---

#### 6. `struct of_phandle_iterator`
**الغرض:** iterator بيتستخدم داخلياً لما بيعمل parse لـ list من phandles في property واحدة (زي لما عندك `gpios = <&gpio0 5 0>, <&gpio1 3 0>`).

```c
struct of_phandle_iterator {
    const char        *cells_name;   /* اسم الـ #cells property */
    int                cell_count;   /* عدد الـ cells لكل entry */
    const struct device_node *parent; /* الـ parent node */
    const __be32      *list_end;     /* نهاية الـ list */
    const __be32      *cur;          /* الموقع الحالي */
    phandle            phandle;      /* الـ phandle الحالي */
    struct device_node *node;        /* الـ node المفصودة */
};
```

---

### رسم العلاقات بين الـ Structs

```
Device Tree (binary blob in memory)
         │
         ▼
┌─────────────────────────────┐
│       device_node           │
│  name: "gpio@40021000"      │
│  full_name: "/soc/gpio@..."  │
│  fwnode ──────────────────────────────────────┐
│  properties ─────────────┐   │               │
│  parent ──► device_node  │   │               │
│  child  ──► device_node  │   │               │
└─────────────────────────────┘               │
                              │               │
                              ▼               ▼
                    ┌───────────────┐   ┌─────────────────┐
                    │   property    │   │  fwnode_handle  │
                    │  name:"gpios" │   │  (abstraction)  │
                    │  value: [raw] │   └────────┬────────┘
                    │  next ────────┤            │
                    └───────────────┘            │ container_of
                    ┌───────────────┐            │
                    │   property    │◄───────────┘
                    │  name:"#gpio  │
                    │   -cells"     │
                    └───────────────┘

of_get_named_gpio(np, "reset-gpios", 0)
         │
         │ يبحث في np->properties عن "reset-gpios"
         │ يبني of_phandle_args
         │
         ▼
┌──────────────────────────┐
│    of_phandle_args       │
│  np ──► device_node      │◄── الـ GPIO controller node
│  args_count = 2          │
│  args[0] = 5  (offset)   │
│  args[1] = 1  (flags)    │
└──────────┬───────────────┘
           │ يمرر لـ
           ▼
┌──────────────────────────────────────────────────┐
│                  gpio_chip                        │
│  label: "stm32-gpio"                             │
│  base: 80  (or dynamic)                          │
│  ngpio: 16                                        │
│  of_gpio_n_cells: 2                              │
│  gpiodev ──► gpio_device (opaque, internal)      │
│  fwnode  ──► fwnode_handle ──► device_node       │
│                                                   │
│  of_xlate() ─────────────────────────────────────┤
│    reads args[0]=offset, args[1]=flags           │
│    returns chip-relative GPIO number              │
│                                                   │
│  irq: gpio_irq_chip                              │
│    ├── chip ──► irq_chip                         │
│    ├── domain ──► irq_domain                     │
│    ├── parents[] ──► parent IRQ numbers           │
│    └── valid_mask ──► bitmask                    │
└──────────────────────────────────────────────────┘
```

---

### رسم الـ Lifecycle

#### أ. إنشاء الـ Device Tree Node وربطها بالـ GPIO Controller

```
Boot / DTB Load
      │
      ▼
┌──────────────────────────────────────────┐
│  of_flat_dt_to_node()                    │
│  يحوّل الـ DTB الـ binary لـ linked     │
│  tree من device_node structs في memory   │
└─────────────────┬────────────────────────┘
                  │
                  ▼
         device_node مُبني في memory
         flags = 0 (fresh)
                  │
                  ▼
┌──────────────────────────────────────────┐
│  Driver probe() يشتغل                   │
│  platform_get_resource() / devm_ioremap()│
│  يملأ gpio_chip fields                  │
└─────────────────┬────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────────┐
│  gpiochip_add_data()                     │
│  - يسجّل gpio_chip في gpiolib           │
│  - يعمل irq_domain لو irq.chip != NULL  │
│  - يربط gpio_chip->fwnode بالـ DT node  │
│  - يضرب OF_POPULATED flag               │
└─────────────────┬────────────────────────┘
                  │
                  ▼
         gpio_chip جاهز للاستخدام

```

#### ب. استخدام `of_get_named_gpio()` من Driver تاني

```
Consumer Driver probe()
      │
      ▼
of_get_named_gpio(dev->of_node, "reset-gpios", 0)
      │
      ├── of_parse_phandle_with_args()
      │       │
      │       ├── يبحث في properties عن "reset-gpios"
      │       ├── يقرأ الـ phandle → يجيب device_node للـ GPIO controller
      │       ├── يقرأ #gpio-cells من الـ controller node
      │       └── يبني of_phandle_args { np, args_count, args[] }
      │
      ├── of_find_gpio() → gpiochip_find_base_with_of()
      │       │
      │       └── يلف على كل gpio_chip مسجّل
      │           يتأكد إن gc->fwnode يطابق np->fwnode
      │
      └── gc->of_xlate(gc, &gpiospec, &flags)
              │
              └── يرجع GPIO number absolute (base + offset)

      │
      ▼
    GPIO number (integer) → يتستخدم مع gpiod_get_index() / gpio_request()

```

#### ج. Teardown

```
Consumer Driver remove()
      │
      ▼
gpio_free(gpio_num)
      │
      ▼
gpiochip_remove(gc)
      │
      ├── يشيل الـ irq_domain
      ├── يفكّ ربط الـ fwnode
      └── OF_POPULATED flag يتمسح

```

---

### رسم الـ Call Flow الكامل

```
Consumer Driver (e.g., reset controller):
  of_get_named_gpio(np, "reset-gpios", 0)
    │
    ▼
  of_parse_phandle_with_args(np, "reset-gpios", "#gpio-cells", 0, &args)
    │
    ├── of_find_node_by_phandle(phandle)
    │     └── يتنقل في شجرة الـ device_node
    │
    └── يملأ of_phandle_args:
          args.np        = &gpio_controller_node
          args.args_count = 2
          args.args[0]   = 5   ← offset
          args.args[1]   = 1   ← GPIO_ACTIVE_LOW
    │
    ▼
  gpiochip_find() → يجيب gpio_chip المناسب
    │
    ▼
  gc->of_xlate(gc, &args, &flags)
    │
    ├── يقرأ args.args[0] = 5 → offset
    ├── يقرأ args.args[1] = flags
    └── يرجع: gc->base + 5
    │
    ▼
  GPIO number = 85 (مثلاً لو base=80)
    │
    ▼
  gpio_request(85, "reset")
    │
    ▼
  gpio_direction_output(85, 0)  ← reset active
    │
    ▼
  gc->direction_output(gc, 5, 0)
    │
    ▼
  Hardware register write
```

---

### الـ `CONFIG_OF_GPIO` Guard — شرح مهم

الـ file بيستخدم compile-time guard بأسلوب نظيف:

```c
#ifdef CONFIG_OF_GPIO

/* التعريف الحقيقي — بيشتغل لما يكون في device tree support */
extern int of_get_named_gpio(const struct device_node *np,
                              const char *list_name, int index);

#else /* CONFIG_OF_GPIO */

/* Stub — بيرجّع -ENOSYS عشان drivers ما تكسرش */
static inline int of_get_named_gpio(const struct device_node *np,
                                    const char *propname, int index)
{
    return -ENOSYS;  /* Not implemented */
}

#endif
```

**الفايدة:** drivers ممكن تكتب:
```c
gpio = of_get_named_gpio(np, "reset-gpios", 0);
if (gpio < 0) {
    /* fallback to platform data or skip */
}
```
من غير ما تعمل `#ifdef` في كل مكان.

---

### الـ of_xlate و #gpio-cells — علاقة حيوية

الـ Device Tree بيوصف GPIO كده:

```dts
/* الـ GPIO controller node */
gpio0: gpio@40021000 {
    compatible = "vendor,gpio";
    #gpio-cells = <2>;   /* عدد الـ args بعد الـ phandle */
    gpio-controller;
    ngpios = <16>;
};

/* الـ consumer node */
led-controller {
    reset-gpios = <&gpio0 5 GPIO_ACTIVE_LOW>;
    /*              ^     ^ ^
                  phandle offset flags */
};
```

الـ `of_gpio_n_cells = 2` في الـ `gpio_chip` لازم يتطابق مع `#gpio-cells = <2>` في الـ DT.

الـ `of_xlate` الـ default بتعمل كده:

```c
/* Default xlate في gpiolib/of.c */
int of_gpio_simple_xlate(struct gpio_chip *gc,
                         const struct of_phandle_args *gpiospec,
                         u32 *flags)
{
    if (flags)
        *flags = gpiospec->args[1];   /* GPIO_ACTIVE_LOW أو ACTIVE_HIGH */
    return gpiospec->args[0];         /* الـ offset مباشرةً */
}
```

---

### الـ Locking Strategy

#### 1. الـ `device_node` Reference Counting
الـ `device_node` مش protected بـ mutex عادي — بتستخدم **reference counting**:

| Function | Action |
|---|---|
| `of_node_get(node)` | يزوّد الـ refcount |
| `of_node_put(node)` | ينقّص ويحذف لو وصل صفر |
| `DEFINE_FREE(device_node, ...)` | cleanup بـ scope-based (C cleanup attribute) |

الـ `CONFIG_OF_DYNAMIC` هو اللي بيفعّل الـ real refcounting — من غيره الـ functions دي no-ops.

#### 2. الـ `gpio_chip` Locking

الـ `gpio_chip` محمي على مستويين:

| Lock | نوعه | يحمي إيه |
|---|---|---|
| `gpio_lock` (في `gpio_device`) | `spinlock_t` | قائمة الـ GPIO chips المسجّلة |
| `gc->bgpio_lock` | `spinlock_t` | الـ register access للـ basic GPIO |
| `gc->irq.lock_key` | `lock_class_key` | lockdep class للـ IRQ lock |
| `gc->irq.request_key` | `lock_class_key` | lockdep class للـ IRQ request |

#### 3. الـ `of_get_named_gpio()` — لا تحتاج lock

الدالة نفسها **read-only** — بتقرأ بس من الـ DT ومابتغيّرش state. الـ DT nodes محمية بـ `of_node_get/put` لو `CONFIG_OF_DYNAMIC`.

#### Lock Ordering

```
gpio_lock (spinlock)
    └─► bgpio_lock (spinlock) — لو بتكتب register
            └─► irq lock — من خلال irq_chip callbacks

NEVER: irq lock → gpio_lock (deadlock!)
```

#### 4. `initialized` flag في `gpio_irq_chip`

```c
bool initialized;  /* في gpio_irq_chip */
```

ده **guard flag** — الـ IRQ callbacks مش المفروض تتنادى قبل `gpiochip_irqchip_add()` تخلّص. الـ gpiolib بيتأكد منه قبل أي IRQ operation.

---

### ملخص العلاقات في جملة واحدة

**الـ `of_gpio.h` هي الـ bridge** اللي بيعمل translate من **اسم symbolic في الـ Device Tree** (زي `"reset-gpios"`) لـ **GPIO number حقيقي** يعرف الـ kernel يشتغل بيه، عبر `device_node` → `of_phandle_args` → `gpio_chip.of_xlate`.
## Phase 4: شرح الـ Functions

### جدول الـ API — Cheatsheet

| Function | Header | Config Guard | Return |
|---|---|---|---|
| `of_get_named_gpio()` | `linux/of_gpio.h` | `CONFIG_OF_GPIO` | GPIO number أو negative errno |

---

### تصنيف الـ Functions

الـ header ده بسيط جداً — بيعرّف **function واحدة بس** هي `of_get_named_gpio()`. الـ header كله هو abstraction layer صغيرة فوق الـ Device Tree GPIO lookup API. الفكرة: بدل ما الـ driver يشتغل مباشرة مع الـ DT phandle parsing، يستخدم الـ helper ده عشان يجيب رقم الـ GPIO من الـ DT property باسمها.

---

### Group: DT GPIO Lookup

الـ group ده مسؤول عن **ربط الـ Device Tree GPIO properties بالـ Linux GPIO numbering system**. الـ driver بيحدد اسم الـ property (زي `"reset-gpios"`) وعايز الـ GPIO رقم كام في النظام — الـ function دي بتعمل الـ resolution الكاملة من الـ phandle لحد رقم GPIO قابل للاستخدام مع الـ legacy `gpio_*` API.

---

### `of_get_named_gpio()`

```c
/* CONFIG_OF_GPIO enabled — real implementation (extern in of_gpio.c) */
extern int of_get_named_gpio(const struct device_node *np,
                             const char *list_name, int index);

/* CONFIG_OF_GPIO disabled — stub that always fails */
static inline int of_get_named_gpio(const struct device_node *np,
                                    const char *propname, int index)
{
    return -ENOSYS;
}
```

**الـ function بتعمل إيه:**
بتقرأ DT property باسمها من الـ `device_node`، بتعمل resolve للـ GPIO phandle اللي فيها، وبترجع الـ **global GPIO number** المقابل للـ GPIO المطلوب في الـ list. الـ implementation الحقيقية موجودة في `drivers/gpio/gpiolib-of.c` وبتستخدم `of_get_named_gpiod_flags()` تحت الغطاء.

الـ flow الداخلي بيمر على:
1. `of_parse_phandle_with_args()` — بتحوّل الـ property لـ `of_phandle_args`
2. `of_find_gpio()` — بتجيب الـ `gpio_chip` المسؤول
3. `gpio_chip->of_xlate()` — بتترجم الـ DT args لـ hwirq offset
4. `gpio_chip->base + offset` — بيطلع الـ global GPIO number

**Parameters:**

| Parameter | Type | Description |
|---|---|---|
| `np` | `const struct device_node *` | الـ device node اللي فيه الـ property، عادةً `pdev->dev.of_node` |
| `list_name` | `const char *` | اسم الـ DT property بدون suffix، مثلاً `"reset-gpios"` أو `"enable-gpios"` |
| `index` | `int` | رقم الـ GPIO داخل الـ list (zero-based)، لو الـ property فيها GPIO واحد بس يبقى `0` |

**Return Value:**
- **≥ 0**: رقم الـ GPIO العالمي (global GPIO number) — جاهز للاستخدام مع `gpio_request()` أو `gpio_direction_input()` إلخ.
- **`-ENOENT`**: الـ property مش موجودة في الـ DT.
- **`-EINVAL`**: الـ property موجودة بس الـ format غلط أو الـ index أكبر من عدد الـ GPIOs في الـ list.
- **`-ENOSYS`**: الـ kernel متبناش بـ `CONFIG_OF_GPIO` (بيرجع الـ stub).
- **`-EPROBE_DEFER`**: الـ GPIO controller اللي الـ phandle بيشاور عليه لسه ماتسجلش — الـ driver المستدعي لازم يرجع `EPROBE_DEFER` من `probe()`.
- **باقي الـ negative errno**: errors من الـ `gpio_chip->of_xlate()` أو الـ DT parsing.

**Key Details:**

- **Locking**: مفيش lock صريح في الـ wrapper نفسه، لكن الـ `of_parse_phandle_with_args()` بتاخد `of_mutex` داخلياً لما بتمشي في الـ DT tree.
- **Legacy API**: الـ function دي جزء من الـ **legacy GPIO API** (integer-based)، مش الـ descriptor-based الحديث. الـ kernel نفسه بيشجع استخدام `devm_gpiod_get()` بدلها في الكود الجديد، لكنها لسه موجودة لـ backward compatibility.
- **الـ `list_name` convention**: الـ kernel بيتوقع إن الـ DT property اسمها `<list_name>-gpios` أو `<list_name>-gpio`، يعني لو بعتله `"reset"` هيدور على `"reset-gpios"`. **ملحوظة مهمة**: الـ function الحديثة `of_get_named_gpiod_flags()` بتعمل الـ strip للـ suffix تلقائياً، لكن `of_get_named_gpio()` بتاخد الاسم كامل زي ما هو بما فيه الـ `-gpios`.
- **Error path**: لو الـ GPIO controller مش موجود وقت الـ probe، الـ function بترجع `EPROBE_DEFER` — ده مش error حقيقي، ده signal للـ driver core إنه يعيد الـ probe تاني.

**Who calls it:**
بيتكلمها الـ platform drivers في `probe()` function عشان تجيب أرقام GPIOs من الـ DT. مثال:

```c
static int my_driver_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    int gpio_num;

    /* get first GPIO from "reset-gpios" property */
    gpio_num = of_get_named_gpio(np, "reset-gpios", 0);
    if (gpio_num < 0) {
        if (gpio_num == -EPROBE_DEFER)
            return -EPROBE_DEFER; /* retry later */
        dev_err(&pdev->dev, "failed to get reset GPIO: %d\n", gpio_num);
        return gpio_num;
    }

    /* now use legacy API */
    return gpio_request(gpio_num, "my-reset");
}
```

---

### Pseudocode Flow (CONFIG_OF_GPIO enabled)

```
of_get_named_gpio(np, "reset-gpios", 0)
│
├─► of_get_named_gpiod_flags(np, "reset-gpios", 0, &flags)
│       │
│       ├─► of_parse_phandle_with_args(np, "reset-gpios",
│       │         "#gpio-cells", 0, &gpiospec)
│       │       └─► parses DT: finds phandle → gpio controller node
│       │                      reads args (pin number, flags)
│       │
│       ├─► of_find_gpio(np, "reset-gpios", 0, &flags)
│       │       └─► of_xlate(chip, &gpiospec, &flags)
│       │               └─► returns hwirq (offset within chip)
│       │
│       └─► returns gpio_desc*
│
└─► desc_to_gpio(desc)
        └─► returns chip->base + hwirq  (global GPIO number)
```

---

### ملاحظة على الـ Stub

لما `CONFIG_OF_GPIO` مش مفعّل، الـ `static inline` stub بترجع `-ENOSYS` على طول. ده بيخلي الـ drivers اللي بتستخدم `of_get_named_gpio()` تـ link بدون مشكلة حتى على platforms من غير OF GPIO support، طالما الـ driver بيـ handle الـ negative return بصورة صحيحة.

```c
/* stub — no DT GPIO support compiled in */
static inline int of_get_named_gpio(const struct device_node *np,
                                    const char *propname, int index)
{
    return -ENOSYS; /* always fail gracefully */
}
```
## Phase 5: دليل الـ Debugging الشامل

الـ `of_gpio.h` بيوفر الـ OF (Open Firmware / Device Tree) helpers للـ GPIO API. الـ debugging هنا بيتمحور حوالين ٣ محاور: **Device Tree parsing**، **GPIO subsystem state**، و**hardware line state**.

---

### Software Level

#### 1. الـ debugfs entries

الـ GPIO subsystem بيعرض معلوماتها الكاملة جوه `/sys/kernel/debug/gpio`:

```bash
# اقرأ حالة كل GPIO lines المسجلة في الكيرنل
cat /sys/kernel/debug/gpio
```

**مثال على الـ output:**

```
gpiochip0: GPIOs 0-31, parent: platform/gpio0, gpio0:
 gpio-5   (reset-gpio           ) out lo
 gpio-12  (enable-gpio          ) out hi
 gpio-17  (irq-gpio             ) in  hi IRQ
```

**تفسير الـ output:**
- الاسم جنب الـ GPIO number = الـ `list_name` اللي اتبعت في `of_get_named_gpio()`
- `out lo` / `out hi` = direction + value
- `in hi IRQ` = input مربوط بـ interrupt

```bash
# شوف الـ GPIO chips المسجلة
ls /sys/kernel/debug/gpio*

# لو في kernel جديد بيستخدم gpio-cdev
cat /sys/kernel/debug/gpio | grep -A5 "gpiochip"
```

#### 2. الـ sysfs entries

```bash
# اعرف كل gpiochips موجودة
ls /sys/class/gpio/

# output:
# export  gpiochip0  gpiochip32  unexport

# اعرف معلومات gpiochip معين
cat /sys/class/gpio/gpiochip0/label     # اسم الـ chip
cat /sys/class/gpio/gpiochip0/base      # base GPIO number
cat /sys/class/gpio/gpiochip0/ngpio     # عدد الـ lines

# export GPIO يدوياً عشان تتعامل معاه
echo 17 > /sys/class/gpio/export
cat /sys/class/gpio/gpio17/direction    # in / out
cat /sys/class/gpio/gpio17/value        # 0 / 1

# الـ Device Tree path للـ GPIO chip
cat /sys/class/gpio/gpiochip0/device/of_node/compatible
```

**جدول الـ sysfs paths المهمة:**

| Path | المحتوى |
|------|---------|
| `/sys/class/gpio/gpiochipN/label` | اسم الـ controller |
| `/sys/class/gpio/gpiochipN/base` | رقم أول GPIO |
| `/sys/class/gpio/gpiochipN/ngpio` | عدد الـ lines |
| `/sys/class/gpio/gpioN/direction` | `in` أو `out` |
| `/sys/class/gpio/gpioN/value` | القيمة الحالية `0`/`1` |
| `/sys/class/gpio/gpioN/active_low` | inverted logic |
| `/sys/bus/platform/drivers/gpio*/` | الـ driver binding |

#### 3. الـ ftrace — tracepoints وإيفنتات مفيدة

```bash
# شوف الـ available events اللي علاقتها بالـ GPIO والـ OF
grep -r "gpio\|of_gpio" /sys/kernel/tracing/available_events

# فعّل tracing للـ GPIO requests
echo 1 > /sys/kernel/tracing/events/gpio/enable

# تابع parsing الـ Device Tree properties
echo 1 > /sys/kernel/tracing/events/of/enable

# ابدأ الـ tracing وشغّل الـ driver
echo 1 > /sys/kernel/tracing/tracing_on
modprobe <your_driver>   # أو insmod
echo 0 > /sys/kernel/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/tracing/trace
```

**filter على function معينة:**

```bash
# trace الـ of_get_named_gpio فقط
echo "of_get_named_gpio" > /sys/kernel/tracing/set_ftrace_filter
echo "function" > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on
```

#### 4. الـ printk والـ dynamic debug

```bash
# فعّل dynamic debug لـ of_gpio subsystem بالكامل
echo "file of_gpio.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لملفات الـ OF parsing كلها
echo "file drivers/of/*.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل debug للـ GPIO driver بتاعك (مثلاً gpio-pl061)
echo "file drivers/gpio/gpio-pl061.c +p" > /sys/kernel/debug/dynamic_debug/control

# شوف اللي اتفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep gpio

# أو عن طريق kernel cmdline
# dyndbg="file of_gpio.c +p"
```

**في الـ driver نفسه** — ضيف `dev_dbg` في الأماكن الحساسة:

```c
/* داخل probe() بعد of_get_named_gpio */
gpio_num = of_get_named_gpio(np, "reset-gpios", 0);
dev_dbg(dev, "of_get_named_gpio('reset-gpios', 0) = %d\n", gpio_num);
if (gpio_num < 0)
    dev_err(dev, "failed to get reset GPIO: %d\n", gpio_num);
```

#### 5. الـ Kernel Config options للـ Debugging

| Config | الغرض |
|--------|-------|
| `CONFIG_OF_GPIO` | لازم يكون `y` عشان `of_get_named_gpio` تشتغل |
| `CONFIG_GPIOLIB` | الـ GPIO subsystem الأساسي |
| `CONFIG_DEBUG_GPIO` | يفعّل debug messages في الـ GPIO core |
| `CONFIG_GPIO_SYSFS` | يعرض الـ GPIOs في sysfs |
| `CONFIG_OF_DYNAMIC` | يدعم تعديل الـ DT في runtime |
| `CONFIG_OF_UNITTEST` | unit tests للـ OF subsystem |
| `CONFIG_PROVE_LOCKING` | يكشف locking bugs في GPIO |
| `CONFIG_DEBUG_FS` | يعرض `/sys/kernel/debug/gpio` |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل dynamic debug system |
| `CONFIG_GPIOLIB_IRQCHIP` | debug للـ GPIO-based IRQs |

```bash
# تأكد إن الـ configs متفعلة
zcat /proc/config.gz | grep -E "CONFIG_(OF_GPIO|DEBUG_GPIO|GPIOLIB|GPIO_SYSFS)"
```

#### 6. أدوات خاصة بالـ Subsystem

**الـ `gpioinfo` و `gpioget` و `gpioset`** من `libgpiod`:

```bash
# اعرف كل GPIO chips والـ lines بتاعتها
gpioinfo

# اقرأ قيمة GPIO line معينة (chip0, line 17)
gpioget gpiochip0 17

# اكتب قيمة
gpioset gpiochip0 17=1

# تابع تغيير قيمة GPIO
gpiomon gpiochip0 17
```

**فحص الـ Device Tree node مباشرة:**

```bash
# اقرأ property اسمها reset-gpios من DT
cat /proc/device-tree/soc/mydevice@1000/reset-gpios | hexdump -C

# اعرف الـ phandle الخاص بالـ GPIO controller
cat /proc/device-tree/soc/mydevice@1000/reset-gpios | xxd
```

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ Kernel Log | السبب | الحل |
|---------------------|-------|------|
| `of_get_named_gpio: ret = -ENOENT` | الـ property مش موجودة في الـ DTS | تأكد من اسم الـ property في الـ DTS |
| `of_get_named_gpio: ret = -EPROBE_DEFER` | الـ GPIO controller لسه مش جاهز | الـ driver هيتجرب تاني، مش error حقيقي |
| `of_get_named_gpio: ret = -EINVAL` | الـ index أكبر من عدد الـ GPIOs في الـ property | خفّض الـ index أو صحح الـ DTS |
| `gpio: no more GPIOs available` | استنفدت كل الـ GPIO lines | راجع الـ `ngpio` في الـ chip |
| `gpiod_request: gpio-17 (reset) status -EBUSY` | الـ GPIO محجوز من driver تاني | اعرف مين بيستخدمه: `cat /sys/kernel/debug/gpio` |
| `gpio_request: chip not found` | الـ GPIO chip مش متسجل | الـ GPIO controller driver مش loaded |
| `OF: /soc/gpio: could not get #gpio-cells` | الـ DTS ناقصاها `#gpio-cells` | ضيف `#gpio-cells = <2>;` في الـ GPIO controller node |
| `-ENOSYS` returned | الـ kernel متبناش بـ `CONFIG_OF_GPIO` | ابني الـ kernel مع `CONFIG_OF_GPIO=y` |

#### 8. أماكن استراتيجية لـ `dump_stack()` و `WARN_ON()`

```c
/* في of_get_named_gpio — قبل return القيمة */
int gpio = of_get_named_gpio(np, propname, index);
WARN_ON(gpio == -EBUSY); /* GPIO متحجوز — مش متوقع هنا */

/* تأكد إن الـ node مش NULL قبل الاستخدام */
if (WARN_ON(!np)) {
    dump_stack(); /* عشان تعرف مين بعتك node فاضي */
    return -EINVAL;
}

/* في الـ probe() بعد الحصول على GPIO */
gpio = of_get_named_gpio(np, "enable-gpios", 0);
if (gpio < 0 && gpio != -EPROBE_DEFER) {
    WARN(1, "unexpected error %d getting enable GPIO\n", gpio);
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

```bash
# اقرأ القيمة من الـ kernel
gpioget gpiochip0 5

# قارنها بالقراءة المباشرة من الـ register
# (لازم تعرف base address الـ GPIO controller من الـ datasheet)
devmem2 0x40020014 w    # قرأ GPIO input data register

# تأكد إن الـ direction في الـ kernel بيطابق الـ hardware
cat /sys/class/gpio/gpio5/direction
```

#### 2. الـ Register Dump

```bash
# قرأ GPIO data register مباشرة (مثال: STM32 GPIOA)
# GPIOA base = 0x40020000
devmem2 0x40020000 w   # MODER   — direction لكل pin
devmem2 0x40020004 w   # OTYPER  — output type
devmem2 0x40020008 w   # OSPEEDR — speed
devmem2 0x4002000C w   # PUPDR   — pull-up/pull-down
devmem2 0x40020010 w   # IDR     — input data
devmem2 0x40020014 w   # ODR     — output data

# أو عن طريق /dev/mem مع io tool
io -4 -r 0x40020010    # قرأ 32-bit register
```

**مثال على تفسير نتيجة IDR:**

```
0x40020010 = 0x00000020  →  binary: 0000 0000 0010 0000
                          →  bit 5 = 1  →  GPIO5 = HIGH
```

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

- **Trigger point**: شيّل trigger على الـ falling/rising edge للـ GPIO line اللي بتدي مشكلة.
- **Timing**: قيس الـ delay بين `of_get_named_gpio()` وأول استخدام فعلي للـ pin — لو في تأخير في الـ `gpio_request()` ممكن تلاقي `EPROBE_DEFER`.
- **Voltage levels**: تأكد إن الـ GPIO line بتوصل الـ voltage الصح (3.3V أو 1.8V حسب الـ SoC).
- **Pull resistors**: لو الـ line بتطرطش (floating) وإنت مش مصرّح pull في الـ DTS، هتشوف noise على الـ analyzer.
- **Glitches**: لو الـ GPIO بيولع وبيطفي بسرعة، ابحث عن `gpio_set_value` بيتكال من interrupt context بدون `gpio_set_value_cansleep`.

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | Pattern في الـ Log | التفسير |
|---------|------------------|---------|
| الـ GPIO line فاضية (floating) | قيمة بتتغير من تلقاء نفسها في `gpiomon` | مفيش pull resistor — ضيفه في DTS أو hardware |
| الـ GPIO controller مش مشتغل | `gpio_request failed: -ENODEV` | تأكد إن الـ clock للـ GPIO controller متفعّل |
| تعارض في الـ pinmux | `pin X already requested by Y` | الـ pin متاخد من driver تاني عبر pinctrl |
| مشكلة في الـ voltage translator | القيمة صح في الـ kernel بس الـ hardware مش بيستجاوب | افحص الـ level shifter بالـ oscilloscope |
| الـ active_low مش مضبوط | السلوك معكوس | راجع الـ `GPIO_ACTIVE_LOW` flag في الـ DTS |

#### 5. الـ Device Tree Debugging

```bash
# تأكد إن الـ DT node موجود ومتحمّل
ls /proc/device-tree/soc/mydevice@1000/

# اقرأ الـ reset-gpios property
xxd /proc/device-tree/soc/mydevice@1000/reset-gpios
# output مثال:
# 00000000: 0000 0003 0000 0011 0000 0001  ....
# byte 0-3: phandle للـ GPIO controller (= 0x3)
# byte 4-7: GPIO number (= 17 = 0x11)
# byte 8-11: flags (= 1 = GPIO_ACTIVE_LOW)

# تأكد إن الـ GPIO controller node عنده #gpio-cells = <2>
cat /proc/device-tree/soc/gpio@40020000/#gpio-cells | xxd

# تحقق من الـ compatible string
cat /proc/device-tree/soc/gpio@40020000/compatible

# افحص الـ DT overlay لو في overlay محمّل
cat /sys/firmware/devicetree/base/__symbols__/gpio0
```

**أهم حاجة في الـ DTS** — تأكد من الـ format الصح:

```dts
/* GPIO controller node */
gpio0: gpio@40020000 {
    compatible = "vendor,gpio-ctrl";
    reg = <0x40020000 0x400>;
    #gpio-cells = <2>;   /* لازم تبقى 2: (gpio-number, flags) */
    gpio-controller;
};

/* Device node بيستخدم الـ GPIO */
mydevice@1000 {
    compatible = "vendor,mydevice";
    reg = <0x1000 0x100>;
    reset-gpios = <&gpio0 17 GPIO_ACTIVE_LOW>;
    /*              ↑       ↑       ↑
                  phandle  number  flags */
};
```

**لو `of_get_named_gpio` رجعت -EINVAL:**

```bash
# تحقق إن #gpio-cells في الـ controller = 2
# وإن الـ property اسمها صح (بتنتهي بـ -gpios أو -gpio)
dtc -I fs /proc/device-tree 2>/dev/null | grep -A3 "mydevice"
```

---

### Practical Commands — جاهز للـ Copy-Paste

#### فحص حالة الـ GPIO بالكامل

```bash
#!/bin/bash
# gpio_debug.sh — فحص شامل للـ GPIO state

echo "=== GPIO Chips ==="
gpioinfo 2>/dev/null || cat /sys/kernel/debug/gpio

echo ""
echo "=== GPIO sysfs ==="
for chip in /sys/class/gpio/gpiochip*; do
    echo "Chip: $(basename $chip)"
    echo "  Label: $(cat $chip/label)"
    echo "  Base:  $(cat $chip/base)"
    echo "  NGpio: $(cat $chip/ngpio)"
done

echo ""
echo "=== Exported GPIOs ==="
for gpio in /sys/class/gpio/gpio*; do
    [ -d "$gpio" ] || continue
    n=$(basename $gpio | sed 's/gpio//')
    dir=$(cat $gpio/direction 2>/dev/null)
    val=$(cat $gpio/value 2>/dev/null)
    echo "  GPIO$n: direction=$dir value=$val"
done
```

#### تفعيل debug وتشغيل الـ driver

```bash
# خطوة 1: فعّل dynamic debug
echo "file of_gpio.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/gpio/*.c +p" > /sys/kernel/debug/dynamic_debug/control

# خطوة 2: فعّل ftrace
echo nop > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/events/gpio/enable
echo 1 > /sys/kernel/tracing/tracing_on

# خطوة 3: حمّل الـ driver
modprobe my_gpio_driver

# خطوة 4: اقرأ النتيجة
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace | grep -i gpio

# خطوة 5: شوف الـ kernel messages
dmesg | tail -50 | grep -iE "gpio|of_gpio|gpiochip"
```

#### فحص الـ EPROBE_DEFER

```bash
# شوف الـ drivers اللي في حالة deferred probe
cat /sys/kernel/debug/devices_deferred

# أو
dmesg | grep "EPROBE_DEFER\|probe deferred"
```

#### مراقبة تغيير GPIO line في real-time

```bash
# باستخدام gpiomon (libgpiod)
gpiomon --num-events=10 --rising-edge --falling-edge gpiochip0 17

# أو export وقرأ في loop
echo 17 > /sys/class/gpio/export
while true; do
    val=$(cat /sys/class/gpio/gpio17/value)
    echo "$(date +%s.%N): gpio17=$val"
    sleep 0.1
done
```

**تفسير الـ output:**

```
1708690000.123456789: gpio17=0
1708690000.234567890: gpio17=1    ← rising edge هنا
1708690000.345678901: gpio17=1
```

#### Register dump سريع

```bash
# بدل devmem2 لو مش موجود
python3 -c "
import mmap, struct, os
GPIO_BASE = 0x40020000
GPIO_SIZE = 0x400
f = os.open('/dev/mem', os.O_RDONLY | os.O_SYNC)
mem = mmap.mmap(f, GPIO_SIZE, mmap.MAP_SHARED, mmap.PROT_READ, offset=GPIO_BASE)
idr = struct.unpack('<I', mem[0x10:0x14])[0]
print(f'IDR = 0x{idr:08X}  (binary: {idr:032b})')
mem.close()
os.close(f)
"
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — الـ `reset-gpios` مش شغال

#### العنوان
**الـ GPIO reset مش بيتفعل في industrial gateway بيستخدم RK3562**

#### السياق
شركة بتصمم industrial gateway بيشتغل على RK3562 SoC. الـ board بيحمل chip خارجية (RS-485 transceiver) محتاجة GPIO line للـ hardware reset. الـ DT بيحدد `reset-gpios` property، والـ driver بيستخدم `of_get_named_gpio()`.

#### المشكلة
الـ driver بيطلع `probe()` بنجاح، لكن الـ RS-485 chip مش بتتعرف — الـ transceiver مش بيشتغل خالص. الـ engineer يشوف الكود يلاقي:

```c
gpio_num = of_get_named_gpio(np, "reset-gpios", 0);
if (gpio_num < 0) {
    dev_err(dev, "failed to get reset gpio\n");
    return gpio_num;
}
```

الـ log مش بيطلع أي error، يعني الـ function رجعت قيمة >= 0، بس الـ reset مش بيحصل.

#### التحليل
الـ `of_get_named_gpio()` في `of_gpio.h` بتاخد:
- `np` — الـ `device_node` للـ device
- `list_name` — اسم الـ property (`"reset-gpios"`)
- `index` — الـ index في الـ list

الـ function بترجع رقم GPIO **global** من الـ GPIO space. المشكلة مش في الـ API نفسه — المشكلة إن الـ engineer استخدم الرقم الناتج كـ "active high" من غير ما يشيك على الـ flags:

```c
/* WRONG: ignores active-low flag from DT */
gpio_direction_output(gpio_num, 1); /* should be 0 for active-low reset */
```

الـ DT عنده:
```dts
reset-gpios = <&gpio2 5 GPIO_ACTIVE_LOW>;
```

الـ `of_get_named_gpio()` بترجع بس رقم الـ GPIO من غير ما تحمل معاها الـ polarity. الـ `GPIO_ACTIVE_LOW` flag بتتضيع لو المبرمج مش بيستخدم `gpiod` API الحديث.

#### الحل

**Option 1 — استخدام الـ `gpiod` API الحديث بدل `of_get_named_gpio()`:**

```c
#include <linux/gpio/consumer.h>

/* In probe(): */
struct gpio_desc *reset_gpio;

reset_gpio = devm_gpiod_get(dev, "reset", GPIOD_OUT_LOW);
if (IS_ERR(reset_gpio)) {
    dev_err(dev, "failed to get reset gpio: %ld\n", PTR_ERR(reset_gpio));
    return PTR_ERR(reset_gpio);
}

/* Assert reset (handles active-low automatically) */
gpiod_set_value_cansleep(reset_gpio, 1);
msleep(10);
gpiod_set_value_cansleep(reset_gpio, 0);
```

**Option 2 — لو لازم تفضل مع `of_get_named_gpio()` لأسباب legacy:**

```c
enum of_gpio_flags flags;
int gpio_num;

/* Use the extended form to get flags */
gpio_num = of_get_named_gpio_flags(np, "reset-gpios", 0, &flags);
if (!gpio_is_valid(gpio_num))
    return -EINVAL;

/* Check polarity manually */
int assert_val = (flags & OF_GPIO_ACTIVE_LOW) ? 0 : 1;
gpio_direction_output(gpio_num, assert_val); /* assert reset */
msleep(10);
gpio_direction_output(gpio_num, !assert_val); /* deassert reset */
```

**Debug commands:**

```bash
# تشوف القيمة الحالية للـ GPIO
gpioinfo gpiochip2 | grep -i "line 5"

# تقرا الـ DT property مباشرة
cat /proc/device-tree/rs485/reset-gpios | xxd
```

#### الدرس المستفاد
**الـ `of_get_named_gpio()` بترجع رقم بس — مش بتحمل الـ polarity.** أي كود جديد لازم يستخدم `gpiod` API اللي بيتعامل مع الـ `GPIO_ACTIVE_LOW` flag أوتوماتيك. الـ `of_gpio.h` نفسه فيه comment يقول `FIXME: Shouldn't be here` — ده مؤشر إن الـ API ده legacy.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — الـ `CONFIG_OF_GPIO` مش enabled

#### العنوان
**الـ driver بيرجع `-ENOSYS` على Allwinner H616 Android TV box**

#### السياق
فريق بيعمل port لـ Android على TV box بيستخدم Allwinner H616. الـ box عنده HDMI HPD (Hot Plug Detect) GPIO. الـ driver بيستخدم `of_get_named_gpio()` وبيفاجأ بـ `-ENOSYS` في كل مرة.

#### المشكلة
الـ kernel configuration مش عندها `CONFIG_OF_GPIO=y`. الـ `of_gpio.h` عنده:

```c
#ifdef CONFIG_OF_GPIO

extern int of_get_named_gpio(const struct device_node *np,
                             const char *list_name, int index);

#else /* CONFIG_OF_GPIO */

#include <linux/errno.h>

static inline int of_get_named_gpio(const struct device_node *np,
                                   const char *propname, int index)
{
    return -ENOSYS;  /* <-- هنا المشكلة */
}

#endif /* CONFIG_OF_GPIO */
```

الـ driver بيعمل:
```c
gpio = of_get_named_gpio(np, "hpd-gpios", 0);
if (gpio < 0) {
    dev_warn(dev, "no HPD gpio, ret=%d\n", gpio);
    return 0; /* silently ignores! */
}
```

الـ `-ENOSYS` بيعمل early return وما فيش HDMI HPD detection خالص.

#### التحليل
الـ `of_gpio.h` بيوفر stub إن `CONFIG_OF_GPIO` مش موجود — بالذات علشان drivers مش لازم تـ depend على GPIO support. الـ comment في الـ header بيقول بصراحة:

```c
/* Drivers may not strictly depend on the GPIO support, so let them link. */
```

يعني الـ design decision: الـ header بيخلي الـ compilation تعدي، لكن الـ runtime بيرجع `-ENOSYS`. المشكلة في الـ Kconfig مش في الـ كود نفسه.

#### الحل

**خطوة 1 — تشيك الـ Kconfig:**

```bash
grep -r "CONFIG_OF_GPIO" /workspace/external/linux/arch/arm64/configs/
# أو
zcat /proc/config.gz | grep OF_GPIO
```

**خطوة 2 — تضيف الـ config:**

```kconfig
# في allwinner_h616_defconfig
CONFIG_GPIOLIB=y
CONFIG_OF_GPIO=y
CONFIG_GPIO_SUNXI=y
```

**خطوة 3 — تأكد الـ DT صح:**

```dts
/* sun50i-h616.dtsi */
&hdmi {
    hpd-gpios = <&pio 7 4 GPIO_ACTIVE_HIGH>; /* PH4 */
    status = "okay";
};
```

**خطوة 4 — Debug:**

```bash
# بعد الـ fix، تشوف إن الـ gpio ظهر
ls /sys/class/gpio/
dmesg | grep -i "hpd"
```

#### الدرس المستفاد
الـ `#else` branch في `of_gpio.h` موجود بالذات علشان compilers يعدوا — مش علشان الـ feature تشتغل. لو الـ driver بيستخدم `of_get_named_gpio()` لازم `CONFIG_OF_GPIO=y` في الـ defconfig. الفرق بين "يـcompile" و"يشتغل" مهم جداً هنا.

---

### السيناريو الثالث: IoT Sensor على STM32MP1 — الـ `of_get_named_gpio()` بترجع قيمة غلط

#### العنوان
**الـ SPI chip select GPIO مش صح على STM32MP1 IoT sensor board**

#### السياق
board صغيرة IoT بتستخدم STM32MP1 مع SPI pressure sensor (LPS22HB). الـ CS line بتتحكم فيها GPIO بدل hardware CS. الـ driver بيستخدم `of_get_named_gpio()`.

#### المشكلة
الـ driver probe بيحصل، لكن الـ SPI transactions مش بتوصل للـ sensor صح. الـ logic analyzer بيورّي إن الـ CS line مش بتتفعل.

الـ DT:
```dts
&spi1 {
    pressure@0 {
        compatible = "st,lps22hb";
        reg = <0>;
        cs-gpios = <&gpioa 4 GPIO_ACTIVE_LOW>,
                   <&gpiob 1 GPIO_ACTIVE_LOW>; /* هنا في 2 entries */
    };
};
```

الـ كود:
```c
/* BUG: always uses index 0, should use index based on configuration */
gpio = of_get_named_gpio(np, "cs-gpios", 0);
```

#### التحليل
الـ `of_get_named_gpio()` بتاخد `index` parameter:

```c
extern int of_get_named_gpio(const struct device_node *np,
                             const char *list_name, int index);
```

الـ `list_name` هنا `"cs-gpios"` وفيها list من الـ GPIOs. الـ index 0 بيرجع `gpioa 4` — الصح. لكن المشكلة إن الـ developer اتفرد بـ index 0 من غير ما يشيك على الـ board configuration.

بعد review للـ schematic اتضح إن الـ production board الجديدة بتستخدم الـ second GPIO (index 1 = `gpiob 1`)، لكن الكود hardcoded على index 0.

#### الحل

```c
/* Read which CS line to use from DT property */
u32 cs_index = 0;
of_property_read_u32(np, "cs-index", &cs_index); /* optional property */

gpio = of_get_named_gpio(np, "cs-gpios", cs_index);
if (!gpio_is_valid(gpio)) {
    dev_err(dev, "invalid cs gpio at index %u\n", cs_index);
    return -EINVAL;
}
```

**DT update:**
```dts
pressure@0 {
    compatible = "st,lps22hb";
    reg = <0>;
    cs-gpios = <&gpioa 4 GPIO_ACTIVE_LOW>,
               <&gpiob 1 GPIO_ACTIVE_LOW>;
    cs-index = <1>; /* use second CS line on this board */
};
```

**Debug:**
```bash
# تتأكد من الـ GPIO numbers المحددين
cat /sys/kernel/debug/gpio | grep -A2 "gpioa\|gpiob"

# تشوف الـ DT property
fdtget /boot/stm32mp157c-dk2.dtb /soc/spi@4000b000/pressure@0 cs-gpios
```

#### الدرس المستفاد
الـ `index` parameter في `of_get_named_gpio()` قوي جداً — بيخليك تعمل multi-device configurations في نفس الـ DT node. لازم تاخد الـ index من الـ DT نفسه ومتـhardcode-هوش في الكود، خصوصاً لو البورد ممكن تتغير بين revisions.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — الـ driver بيطلع error في kernel بدون `CONFIG_OF`

#### العنوان
**الـ ECU driver يطلع compile error على i.MX8 automotive platform**

#### السياق
شركة automotive بتعمل ECU (Engine Control Unit) على i.MX8QM. الـ kernel configuration stripped-down جداً لمتطلبات AUTOSAR — وبعض الـ OF support متعملش disable.

#### المشكلة
الـ driver كود:

```c
#include <linux/of_gpio.h>

static int ecu_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    int ignition_gpio;

    ignition_gpio = of_get_named_gpio(np, "ignition-detect-gpios", 0);
    ...
}
```

الـ compile بيطلع:
```
error: implicit declaration of function 'of_get_named_gpio'
```

#### التحليل
الـ `of_gpio.h` بيوضح المشكلة:

```c
#ifdef CONFIG_OF_GPIO

extern int of_get_named_gpio(const struct device_node *np,
                             const char *list_name, int index);

#else /* CONFIG_OF_GPIO */

static inline int of_get_named_gpio(...)
{
    return -ENOSYS;
}

#endif /* CONFIG_OF_GPIO */
```

لو `CONFIG_OF_GPIO` مش موجود، الـ stub الـ `static inline` موجود. لكن المشكلة إن الـ engineer نسي يـinclude الـ header خالص، أو الـ `CONFIG_OF` نفسه disabled مما بيخلي الـ `device_node` struct مش متعرفش.

الـ `of_gpio.h` بيحتاج:
```c
#include <linux/of.h>  /* for struct device_node */
```

والـ `of.h` نفسه guarded بـ:
```c
/* من of.h */
struct device_node { ... };
```

لو الـ `CONFIG_OF` غير موجود، الـ `device_node` struct مش متعرفش في بعض الـ kernel versions.

#### الحل

**Option 1 — تأكد Kconfig dependencies:**

```kconfig
# Kconfig للـ ECU driver
config ECU_DRIVER
    tristate "Automotive ECU Driver"
    depends on OF && OF_GPIO
    help
      Driver for automotive ECU on i.MX8QM.
```

**Option 2 — استخدام platform data بديل عن DT للـ automotive:**

```c
/* ecu_platform.h */
struct ecu_platform_data {
    int ignition_gpio;
    bool ignition_active_low;
};

/* في probe() */
#ifdef CONFIG_OF
if (pdev->dev.of_node) {
    gpio = of_get_named_gpio(np, "ignition-detect-gpios", 0);
} else
#endif
{
    struct ecu_platform_data *pdata = dev_get_platdata(&pdev->dev);
    if (pdata)
        gpio = pdata->ignition_gpio;
}
```

**Debug commands:**

```bash
# تشوف الـ kernel config
zcat /proc/config.gz | grep -E "CONFIG_OF|CONFIG_OF_GPIO"
# Output expected:
# CONFIG_OF=y
# CONFIG_OF_GPIO=y
```

#### الدرس المستفاد
الـ `of_gpio.h` الـ guard `#ifdef CONFIG_OF_GPIO` بيوفر stub بس مش بيحل مشكلة missing `struct device_node`. في الـ automotive kernels المقلصة، لازم `CONFIG_OF=y` و`CONFIG_OF_GPIO=y` يكونوا enabled مع بعض. الـ `Kconfig` الصريح في الـ driver أهم من أي workaround.

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — تـdebug الـ `of_get_named_gpio()` بيرجع `-ENOENT`

#### العنوان
**الـ `-ENOENT` من `of_get_named_gpio()` أثناء bring-up بورد AM62x جديدة**

#### السياق
engineer بيعمل bring-up لبورد AM62x (Texas Instruments) جديدة. الـ board عندها I2C temperature sensor (TMP117) بيحتاج ALERT GPIO. الـ driver complains بـ `-ENOENT` أثناء الـ probe.

#### المشكلة
الـ driver:

```c
alert_gpio = of_get_named_gpio(np, "alert-gpios", 0);
if (alert_gpio == -ENOENT) {
    dev_info(dev, "no alert gpio defined, polling mode\n");
    priv->use_polling = true;
} else if (alert_gpio < 0) {
    dev_err(dev, "alert gpio error: %d\n", alert_gpio);
    return alert_gpio;
}
```

الـ code بيتعامل مع `-ENOENT` كـ "optional GPIO" ويمشي في polling mode. لكن الـ engineer متأكد إن الـ DT عنده `alert-gpios` property.

#### التحليل
الـ `of_get_named_gpio()` internally بتستخدم `of_find_property()` على الـ `device_node`. لو الـ property موجودة بس الـ phandle غلط، بترجع `-ENOENT`.

**الأسباب الممكنة:**

1. **Typo في اسم الـ property:**
```dts
/* WRONG */
alert-gpio = <&gpio0 15 GPIO_ACTIVE_LOW>; /* missing 's' */

/* CORRECT */
alert-gpios = <&gpio0 15 GPIO_ACTIVE_LOW>;
```

2. **الـ `gpio0` label مش موجود في الـ DT:**
```dts
/* WRONG: gpio controller not defined */
alert-gpios = <&gpio0 15 GPIO_ACTIVE_LOW>;

/* لكن في am62x.dtsi مفيش &gpio0، الصح هو: */
alert-gpios = <&main_gpio0 15 GPIO_ACTIVE_LOW>;
```

3. **الـ `device_node` الغلط بيتبعت:**
```c
/* WRONG: using parent node instead of device node */
struct device_node *np = pdev->dev.of_node->parent;
alert_gpio = of_get_named_gpio(np, "alert-gpios", 0);
```

#### الحل

**Step 1 — تتأكد من الـ DT property:**

```bash
# تقرا الـ compiled DTB
fdtget /boot/k3-am625-sk.dtb /bus@f0000/i2c@20000000/tmp117@48 alert-gpios
# Output: 0x0000001d 0x0000000f 0x00000001
# يعني: phandle=0x1d, pin=15, flags=GPIO_ACTIVE_LOW

# تتأكد إن الـ phandle صح
fdtget /boot/k3-am625-sk.dtb /bus@f0000/i2c@20000000/tmp117@48 --type x alert-gpios
```

**Step 2 — من الـ live system:**

```bash
# تشوف كل GPIOs المتاحة
gpioinfo

# تقرا الـ device node properties
ls /proc/device-tree/bus@f0000/i2c@20000000/tmp117@48/
cat /proc/device-tree/bus@f0000/i2c@20000000/tmp117@48/alert-gpios | xxd

# تشوف الـ driver log
dmesg | grep -i "tmp117\|alert"
```

**Step 3 — الـ DTS الصح للـ AM62x:**

```dts
/* k3-am625-custom.dts */
&main_i2c0 {
    status = "okay";
    clock-frequency = <400000>;

    tmp117: temperature@48 {
        compatible = "ti,tmp117";
        reg = <0x48>;
        alert-gpios = <&main_gpio0 15 GPIO_ACTIVE_LOW>; /* main_gpio0 مش gpio0 */
        #io-channel-cells = <0>;
    };
};
```

**Step 4 — Defensive code في الـ driver:**

```c
alert_gpio = of_get_named_gpio(np, "alert-gpios", 0);
if (alert_gpio == -ENOENT || alert_gpio == -ENODATA) {
    dev_info(dev, "alert-gpios not in DT, using polling\n");
    priv->use_polling = true;
} else if (!gpio_is_valid(alert_gpio)) {
    dev_err(dev, "invalid alert gpio: %d\n", alert_gpio);
    return -EINVAL;
} else {
    /* GPIO valid, use interrupt mode */
    priv->alert_gpio = alert_gpio;
    priv->use_polling = false;
}
```

#### الدرس المستفاد
في الـ bring-up، `-ENOENT` من `of_get_named_gpio()` مش دايماً يعني "الـ property مش موجودة" — ممكن يعني "الـ GPIO controller المشار إليه مش موجود أو اسمه غلط". أهم خطوات الـ debug: تتأكد من **اسم الـ GPIO controller** في الـ SoC DTSi الأصلي (زي `main_gpio0` في AM62x)، وتستخدم `fdtget` و`gpioinfo` علشان تتتبع المشكلة للـ source.
## Phase 7: مصادر ومراجع

### مقدمة

الـ `of_gpio.h` هو header صغير بس مهم — بيربط بين الـ Device Tree والـ GPIO subsystem. المصادر الجاية هتساعدك تفهم السياق الكامل من legacy integer API لحد modern descriptor-based interface.

---

### LWN.net — المقالات الأساسية

| المقال | الأهمية |
|--------|---------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة للـ GPIO API في الكيرنل، بتشرح الـ legacy integer interface وإزاي الـ device tree بيحدد الـ GPIO |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | بيشرح ليه الـ integer-based API بقت مشكلة وإيه الاتجاه نحو descriptor-based API |
| [gpiolib: introduce descriptor-based GPIO interface](https://lwn.net/Articles/528226/) | أول patch series قدم الـ `gpiod_*` API كبديل للـ `of_get_named_gpio` |
| [gpio: introduce descriptor-based interface](https://lwn.net/Articles/531848/) | تفاصيل implementation الـ descriptor interface الجديدة |
| [New descriptor-based GPIO interface](https://lwn.net/Articles/565662/) | مقال كامل بيوضح الـ new API بأمثلة عملية وإزاي تعمل migration |
| [Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/) | توثيق الـ new interface رسميًا في الكيرنل |
| [gpiolib: Add GPIO name support](https://lwn.net/Articles/651500/) | إضافة دعم الـ GPIO names من الـ device tree — مرتبط مباشرة بالـ `of_get_named_gpio` |
| [gpio: add GPIO hogging mechanism](https://lwn.net/Articles/626140/) | الـ hogging mechanism بتتفعّل من `of_gpiochip_add()` اللي بيستخدم `of_gpio` |

---

### التوثيق الرسمي في الكيرنل

**الـ paths دي موجودة في source tree:**

```
Documentation/driver-api/gpio/
├── intro.rst          # مقدمة للـ GPIO subsystem
├── board.rst          # إزاي تعمل GPIO mappings من device tree
├── consumer.rst       # الـ consumer API (gpiod_get, gpiod_set_value, ...)
├── driver.rst         # إزاي تكتب GPIO controller driver
└── legacy.rst         # الـ legacy integer-based API (of_get_named_gpio هنا)

Documentation/devicetree/bindings/gpio/
└── gpio.txt           # الـ DT binding spec للـ GPIO controllers والـ consumers
```

**روابط online:**

- [GPIO Driver API — kernel.org](https://www.kernel.org/doc/html/latest/driver-api/gpio/index.html)
- [GPIO Mappings (board.rst)](https://www.kernel.org/doc/html/latest/driver-api/gpio/board.html) — بيشرح `<function>-gpios` property في الـ DT وإزاي `gpiod_get()` بيقراها
- [GPIO devicetree bindings](https://www.kernel.org/doc/Documentation/devicetree/bindings/gpio/gpio.txt) — الـ spec الرسمي لـ `gpio-controller`, `#gpio-cells`, وتنسيق الـ GPIO specifier

---

### Commits مهمة في تاريخ `of_gpio`

ابحث عن الـ commits دي في `git log --follow` على `include/linux/of_gpio.h` أو `drivers/gpio/gpiolib-of.c`:

```bash
# شوف تاريخ التعديلات على الملف
git log --oneline include/linux/of_gpio.h

# شوف commit اللي أضاف of_get_named_gpio في الأول
git log --oneline --diff-filter=A -- drivers/of/gpio.c
```

**Commits تاريخية بارزة:**

| الحدث | الوصف |
|-------|-------|
| **v2.6.25** | أول إضافة للـ `gpiolib` في الكيرنل — [kernelnewbies Linux_2_6_25](https://kernelnewbies.org/Linux_2_6_25) |
| **v3.13** | إدخال الـ descriptor-based interface (`gpiod_*`) كبديل للـ `of_get_named_gpio` |
| **v5.x+** | وضع `of_get_named_gpio` رسميًا كـ deprecated في favor of `gpiod_get()` |

---

### نقاشات الـ Mailing List

- [GPIO marking individual pins in device tree — linuxppc-dev](https://linuxppc-dev.ozlabs.narkive.com/qp4tnrzh/gpio-marking-individual-pins-not-available-in-device-tree) — نقاش عن تحديد الـ GPIO pins في الـ DT
- [Using gpio from device tree on platform devices — kernelnewbies list](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html) — نقاش عملي عن الفرق بين الـ legacy `of_get_named_gpio` والـ descriptor API الجديدة
- [devm_gpiod_get usage to get gpio num — kernelnewbies list](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html) — إزاي تستخدم `devm_gpiod_get` بدل `of_get_named_gpio`

---

### eLinux.org — مراجع عملية

- [Device Tree Reference — elinux.org](https://elinux.org/Device_Tree_Reference) — مرجع شامل للـ DT، فيه section للـ GPIO و `#gpio-cells`
- [Device Tree Mysteries — elinux.org](https://elinux.org/Device_Tree_Mysteries) — بيشرح `#gpio-cells` وصيغة الـ GPIO specifier في الـ DT
- [EBC Device Trees — elinux.org](https://elinux.org/EBC_Device_Trees) — أمثلة عملية على BeagleBone لضبط GPIO في الـ DT

---

### KernelNewbies.org

- [Linux_2_6_21 — أول إضافة GPIO API](https://kernelnewbies.org/Linux_2_6_21) — الكيرنل ده شاف أول GPIO abstraction
- [Linux_2_6_25 — إضافة gpiolib](https://kernelnewbies.org/Linux_2_6_25) — تم إضافة `gpiolib` directory والـ GPIO expander drivers
- [Linux_6.2 Changes](https://kernelnewbies.org/Linux_6.2) — تحديثات GPIO في الإصدارات الحديثة
- [LinuxChanges](https://kernelnewbies.org/LinuxChanges) — تتبع كل التغييرات في كل إصدار كيرنل

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)

- **الفصل 12: I/O Memory and Ports** — بيشرح الـ hardware access primitives
- الكتاب متاح مجانًا: [free-electrons.com/doc/books/ldd3.pdf](https://lwn.net/Kernel/LDD3/)
- ملاحظة: LDD3 قديم (kernel 2.6)، مش بيغطي الـ `gpiod_*` API ولا الـ DT integration الحديث

#### Linux Kernel Development — Robert Love (3rd Edition)

- **الفصل 17: Devices and Modules** — بيشرح الـ driver model وكيف الـ subsystems بتتكامل
- مهم لفهم الـ `CONFIG_OF_GPIO` guards وطريقة compile-time conditionality

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- **الفصل 15: Embedded Bootloaders** — بيشرح الـ device tree وطريقة تعريف الـ GPIO في الـ DTS files
- أفضل كتاب للـ embedded context اللي `of_gpio.h` موجود فيه

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds (3rd Edition)

- **الفصل 11: Interfacing with Device Drivers** — بيغطي الـ gpiod API الحديث مع أمثلة DT

---

### مصادر إضافية مفيدة

- [Linux GPIO Interface & Device Tree — embedded.com](https://www.embedded.com/linux-device-driver-development-the-gpio-interface-and-device-tree/) — مقال عملي بيربط الـ GPIO API بالـ device tree مع code examples
- [Stop using /sys/class/gpio — thegoodpenguin.co.uk](https://www.thegoodpenguin.co.uk/blog/stop-using-sys-class-gpio-its-deprecated/) — بيشرح ليه الـ legacy interfaces بقت deprecated
- [GPIO device tree configuration — STM32MPU wiki](https://wiki.st.com/stm32mpu/wiki/GPIO_device_tree_configuration) — مثال حقيقي لـ vendor بيستخدم `of_gpio` على STM32

---

### Search Terms للبحث عن معلومات أكتر

```
# بحث عام
"of_get_named_gpio" site:lore.kernel.org
"of_gpio" deprecated gpiod_get migration
linux kernel gpio descriptor migration guide

# في الكود
git log --oneline drivers/gpio/gpiolib-of.c
git log --grep="of_get_named_gpio" --oneline
grep -r "of_get_named_gpio" drivers/ --include="*.c" | head -20

# في الـ documentation
grep -r "of_get_named_gpio" Documentation/
```

---

### ملخص سريع للـ Migration Path

```
Legacy (ما تزالش شغالة):               Modern (المُوصى بيها):
─────────────────────────────────      ──────────────────────────────────
of_get_named_gpio()           ──►      gpiod_get() / devm_gpiod_get()
gpio_request() + gpio_direction_*  ──► gpiod_direction_input/output()
gpio_get_value()              ──►      gpiod_get_value()
gpio_set_value()              ──►      gpiod_set_value()
```

**الـ `of_get_named_gpio`** لسه شغال بس officially deprecated — أي كود جديد لازم يستخدم الـ descriptor API.
## Phase 8: Writing simple module

### الفكرة

الـ function الأكتر إثارة في `of_gpio.h` هي `of_get_named_gpio()` — بتجيب رقم GPIO من device tree بناءً على اسم property وindex. هنحط عليها **kprobe** عشان نشوف كل مرة بيطلبها driver: اسم الـ node، اسم الـ property، والـ index.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0+
/*
 * kprobe_of_gpio.c
 *
 * Hooks of_get_named_gpio() and logs every call:
 *   - device_node full_name
 *   - GPIO property name (list_name)
 *   - index requested
 */

/* ---- includes ---- */
#include <linux/module.h>       /* module_init/exit, MODULE_* macros */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/of.h>           /* struct device_node */
#include <linux/printk.h>       /* pr_info */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe on of_get_named_gpio to trace GPIO DT lookups");

/* ---- pre-handler: runs just before of_get_named_gpio executes ---- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * of_get_named_gpio signature:
     *   int of_get_named_gpio(const struct device_node *np,
     *                         const char *list_name, int index)
     *
     * On x86-64: rdi=np, rsi=list_name, rdx=index
     * On ARM64 : x0=np,  x1=list_name,  x2=index
     */

#if defined(CONFIG_X86_64)
    const struct device_node *np   = (void *)regs->di;
    const char               *name = (void *)regs->si;
    int                       idx  = (int)   regs->dx;
#elif defined(CONFIG_ARM64)
    const struct device_node *np   = (void *)regs->regs[0];
    const char               *name = (void *)regs->regs[1];
    int                       idx  = (int)   regs->regs[2];
#else
    /* fallback: just report that the probe fired */
    pr_info("of_gpio_probe: of_get_named_gpio called (arch not decoded)\n");
    return 0;
#endif

    /* guard against NULL pointers before dereferencing */
    pr_info("of_gpio_probe: node=\"%s\"  prop=\"%s\"  index=%d\n",
            (np && np->full_name) ? np->full_name : "(null)",
            name ? name : "(null)",
            idx);

    return 0; /* 0 = continue normal execution */
}

/* ---- kprobe struct ---- */
static struct kprobe kp = {
    .symbol_name = "of_get_named_gpio",  /* function to hook by name */
    .pre_handler = handler_pre,           /* called before the function */
};

/* ---- module_init ---- */
static int __init of_gpio_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("of_gpio_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("of_gpio_probe: kprobe planted on %s @ %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ---- module_exit ---- */
static void __exit of_gpio_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("of_gpio_probe: kprobe removed\n");
}

module_init(of_gpio_probe_init);
module_exit(of_gpio_probe_exit);
```

---

### Makefile

```makefile
obj-m += kprobe_of_gpio.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

| Header | ليه موجود |
|--------|-----------|
| `linux/module.h` | ماكروهات `module_init/exit` و `MODULE_LICENSE` |
| `linux/kprobes.h` | تعريف `struct kprobe` ودوال `register/unregister_kprobe` |
| `linux/of.h` | تعريف `struct device_node` عشان نـaccess الـ `full_name` |
| `linux/printk.h` | `pr_info / pr_err` |

---

#### الـ `handler_pre`

**الـ pre-handler** بيتشغل فور ما الـ CPU يوصل لأول instruction في `of_get_named_gpio` قبل ما تتنفذ أي حاجة منها.
بنقرأ الـ arguments من الـ registers مباشرةً لأن الـ kprobe بيشتغل على مستوى machine code، مش على مستوى C function call — فالـ ABI بيحدد إن الـ args الأولى بتتمرر في `rdi/rsi/rdx` على x86-64 وفي `x0/x1/x2` على ARM64.

---

#### الـ `struct kprobe`

الـ `symbol_name` بيخلي الـ kernel يبحث عن العنوان تلقائياً من الـ kallsyms وقت الـ `register_kprobe` — مش محتاجين نحدد عنوان hardcoded.
الـ `pre_handler` هو الـ callback اللي هيتشغل قبل كل call للدالة.

---

#### الـ `module_init`

بيسجّل الـ kprobe ويطبع عنوانه الفعلي بعد التسجيل — لو فشل (مثلاً الدالة مش exported أو الـ CONFIG_KPROBES مش مفعّل) بيرجع الـ error ويخلي الـ insmod يفشل بدل ما يظل silent.

---

#### الـ `module_exit`

**لازم** نـunregister الـ kprobe في الـ exit عشان نزيل الـ int3 breakpoint اللي زرعه الـ kernel في كود `of_get_named_gpio` — لو فضلناه من غير module يـcrash الـ system فور ما يتصل بيها أي driver.

---

### تجربة عملية

```bash
# بناء وتحميل الـ module
make
sudo insmod kprobe_of_gpio.ko

# شاهد الـ output في real-time
sudo dmesg -w | grep of_gpio_probe

# مثال على output لما driver بيطلب GPIO
# of_gpio_probe: node="/soc/gpio@ff780000"  prop="reset-gpios"  index=0
# of_gpio_probe: node="/soc/i2c@ff110000/sensor@48"  prop="irq-gpios"  index=0

# إزالة الـ module
sudo rmmod kprobe_of_gpio
```

---

### ملاحظات أمان

- الكود بيتحقق من `NULL` قبل ما يـdereference `np->full_name` أو `name` عشان callback بيشتغل في kernel context وأي page fault هيـoops الـ system.
- الـ `return 0` من الـ `pre_handler` ضروري — أي قيمة تانية بتبوس الـ execution path وممكن تـskip الدالة الأصلية.
