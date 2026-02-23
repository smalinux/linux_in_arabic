## Phase 1: الصورة الكبيرة ببساطة

### الـ GPIO — إيه هو أصلاً؟

تخيل إن عندك مفتاح كهرباء بسيط — تقدر تشغله أو تطفيه، وتقدر تحس إذا حد ضغط عليه. ده بالظبط الـ **GPIO** (General Purpose Input/Output): دبوس (pin) على الـ chip بتقدر تتحكم فيه بالـ software — تبعته high أو low، أو تقرأ منه القيمة.

الـ GPIO موجود في كل حتة: Arduino، Raspberry Pi، أي microcontroller، وكمان في الـ SoC اللي بيشغل موبايلك. بيستخدمه الـ driver عشان يتحكم في LED، يقرأ button، يلف relay، يتحكم في reset line لجهاز تاني، إلخ.

---

### القصة الكبيرة — من الفوضى للنظام

#### المشكلة الأصلية

في البداية، كل board في Linux كانت بتعمل GPIO بطريقتها الخاصة. كل platform كانت بتحدد أرقام للـ GPIOs زي:
- Board A: GPIO رقم 17 = الـ LED
- Board B: GPIO رقم 17 = حاجة تانية خالص

الـ driver لازم يعرف الأرقام دي hardcoded. ده كان كارثة لما جيت تحاول تشغل نفس الـ driver على boards مختلفة.

الحل الأول كان **GPIOLIB** — مكتبة مركزية في الـ kernel بتدير كل الـ GPIOs وبتديلهم **أرقام global** (global GPIO numberspace). الـ driver بيقول: "أنا عايز GPIO رقم 42" وبيشتغل.

#### المشكلة الثانية — الأرقام مجردة وبلا معنى

لكن مشكلة جديدة ظهرت: الأرقام دي ملهاش معنى semantic. إيه علاقة رقم 42 بإن ده "LED الأحمر"؟ لو بدلت الـ hardware بطريقة تغير الأرقام، كل الـ drivers هتتكسر.

الحل الحديث كان **GPIO descriptor API** — بدل ما تقول "GPIO 42"، بتقول "أنا device X، وعايز الـ GPIO اللي اسمه `enable` في الـ device tree". الـ kernel هو اللي يترجم.

---

### دور الـ `include/linux/gpio.h` — "الـ Legacy Wrapper"

الفايل ده مش فايل جديد ولا مهم للكود الجديد — هو بصراحة **deprecated**. الكومنت الأول فيه بيقول صراحة:

```c
/*
 * NOTE: This header *must not* be included.
 *
 * This is the LEGACY GPIO bulk include file, including legacy APIs.
 */
```

**الـ** `gpio.h` ده عبارة عن **compatibility shim** — طبقة توافق قديمة. وظيفته:

1. يوفر الـ API القديم `gpio_request()` / `gpio_free()` / `gpio_direction_input()` إلخ — اللي بتشتغل على أرقام وموش descriptors.
2. الـ functions دي جوّاها بتعمل `gpio_to_desc()` وبتفوّض للـ API الحديث (`gpiod_*`).
3. لو `CONFIG_GPIOLIB` مش موجود، كل الـ functions بترجع error أو بتعمل `WARN_ON`.

---

### التطور التاريخي — ثلاث طبقات

```
┌─────────────────────────────────────────────────────┐
│  LEGACY API  │  gpio_request(42, "led")             │ ← الكود القديم
│  gpio.h      │  gpio_direction_output(42, 1)        │   بيستخدم أرقام
└──────────────┴──────────────────────────────────────┘
        │  gpio_to_desc(42)  ← التحويل هنا
        ▼
┌─────────────────────────────────────────────────────┐
│  MODERN API  │  gpiod_get(dev, "enable", GPIOD_OUT) │ ← الكود الجديد
│  consumer.h  │  gpiod_direction_output(desc, 1)     │   بيستخدم descriptors
└──────────────┴──────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────┐
│  GPIOLIB     │  gpiolib.c / gpiolib-*.c             │ ← الـ Core
│  CORE        │  struct gpio_desc, gpio_chip         │
└──────────────┴──────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────┐
│  HW DRIVERS  │  gpio-pl061.c, gpio-zynq.c, ...      │ ← الـ Hardware
│  drivers/gpio│  كل chip بـ driver خاص              │
└──────────────┴──────────────────────────────────────┘
```

---

### الـ `CONFIG_GPIOLIB_LEGACY` — مفتاح الإيقاف

الـ legacy API موجود فقط لو `CONFIG_GPIOLIB_LEGACY=y`. الـ distributions الحديثة بدأت تشيله تدريجياً. الـ `gpio.h` نفسه ما بيعمل حاجة بدون الـ flag ده.

---

### مثال واقعي — Driver بسيط

```c
/* كود قديم (legacy) — لا تكتبه في code جديد */
ret = gpio_request(42, "my-led");       /* احجز GPIO رقم 42 */
gpio_direction_output(42, 0);           /* اتجاه output، قيمة ابتدائية 0 */
gpio_set_value(42, 1);                  /* شغل الـ LED */
gpio_free(42);                          /* حرر الـ GPIO */

/* كود حديث (descriptor-based) — الطريقة الصح */
struct gpio_desc *led;
led = gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);  /* من device tree */
gpiod_set_value(led, 1);               /* شغل الـ LED */
gpiod_put(led);                        /* حرر */
```

---

### الـ Flags القديمة

```c
#define GPIOF_IN            ((1 << 0))           /* input */
#define GPIOF_OUT_INIT_LOW  ((0 << 0) | (0 << 1)) /* output, initially low */
#define GPIOF_OUT_INIT_HIGH ((0 << 0) | (1 << 1)) /* output, initially high */
```

الـ flags دي بتتستخدم مع `gpio_request_one()` — اللي بتدمج الـ request والـ direction في call واحد.

---

### الملفات المكوِّنة للـ Subsystem

#### الـ Headers الأساسية

| الملف | الدور |
|-------|-------|
| `include/linux/gpio.h` | Legacy API — deprecated wrapper |
| `include/linux/gpio/consumer.h` | Modern descriptor-based API للـ consumers |
| `include/linux/gpio/driver.h` | API لكاتبي الـ GPIO chip drivers |
| `include/linux/gpio/machine.h` | GPIO lookup tables (board files) |
| `include/linux/gpio/property.h` | Software-defined GPIO properties |
| `include/linux/gpio/generic.h` | Generic MMIO GPIO chip helpers |
| `include/linux/of_gpio.h` | Device Tree / OF integration |

#### الـ Core Implementation

| الملف | الدور |
|-------|-------|
| `drivers/gpio/gpiolib.c` | الـ core الرئيسي — كل المنطق |
| `drivers/gpio/gpiolib-legacy.c` | Legacy API implementation |
| `drivers/gpio/gpiolib-devres.c` | devm_* resource management |
| `drivers/gpio/gpiolib-of.c` | Device Tree binding |
| `drivers/gpio/gpiolib-acpi-core.c` | ACPI binding |
| `drivers/gpio/gpiolib-cdev.c` | Character device (userspace ABI) |
| `drivers/gpio/gpiolib-sysfs.c` | Sysfs interface (legacy) |

#### أمثلة من الـ Hardware Drivers (أكتر من 200 driver)

| الملف | الـ Hardware |
|-------|-------------|
| `drivers/gpio/gpio-pl061.c` | ARM PrimeCell GPIO |
| `drivers/gpio/gpio-zynq.c` | Xilinx Zynq |
| `drivers/gpio/gpio-dwapb.c` | DesignWare APB GPIO |
| `drivers/gpio/gpio-pca953x.c` | NXP PCA953x I2C expander |
| `drivers/gpio/gpio-mcp23s08.c` | Microchip MCP23xxx |

#### الـ UAPI

| الملف | الدور |
|-------|-------|
| `include/uapi/linux/gpio.h` | Userspace ABI (character device) |
| `tools/gpio/` | CLI tools: gpio-event-mon, gpio-hammer |

---

### الخلاصة

**الـ** `include/linux/gpio.h` هو **أثر تاريخي** — موجود بس عشان الـ drivers القديمة ما تتكسرش. الـ kernel نفسه في الكومنت بيقول "لا تحتويه في كود جديد". كل وظائفه اتنقلت للـ descriptor API الحديث في `include/linux/gpio/consumer.h`.

الـ GPIO subsystem نفسه نظام ناضج وكامل بيدير مئات الـ chips المختلفة من خلال abstraction layer موحدة — من أبسط LED على Raspberry Pi لأكتر الأنظمة تعقيداً.
## Phase 2: شرح الـ GPIO Framework

### المشكلة اللي الـ GPIO Subsystem بيحلها

في أي SoC زي STM32 أو i.MX6 أو Raspberry Pi، عندك مئات الـ GPIO pins. المشكلة إن كل controller بيتحكم فيهم بطريقة مختلفة تماماً:

- الـ BCM2835 (Raspberry Pi) بيستخدم memory-mapped registers بسيطة، بدون sleep
- الـ MCP23017 (I2C expander) بيحتاج I2C transaction لكل قراءة/كتابة، يعني ممكن يـ sleep
- الـ Xilinx GPIO IP block جوا FPGA بيتحكم فيه بشكل مختلف كمان

لو كل driver محتاج يتعامل مباشرة مع الـ hardware-specific registers، هتلاقي:

1. كل consumer driver (زي driver الـ LED أو الـ button) محتاج يعرف هو شغال على أي controller بالظبط
2. كود متكرر في كل driver
3. مفيش إمكانية لـ runtime discovery — الـ pin number hardcoded في الكود
4. لو استبدلت الـ GPIO expander بيحتاج تعيد كتابة الـ consumer drivers

**الـ GPIO subsystem** جه عشان يعمل abstraction layer موحدة بين الـ hardware controllers والـ consumer code.

---

### الحل — إزاي الـ kernel بيتعامل مع الموضوع

الـ kernel قسّم الموضوع لجزأين واضحين:

**الجانب الأول — الـ Provider (Driver Side):**
كل GPIO controller بيسجّل نفسه عن طريق `struct gpio_chip`، وبيملا الـ function pointers اللي الـ framework يستخدمها.

**الجانب التاني — الـ Consumer (User Side):**
أي driver محتاج GPIO بيطلبه باسم منطقي من الـ firmware (Device Tree أو ACPI)، ويرجعله `struct gpio_desc *` — مش رقم raw.

الـ framework في النص بيعمل:
- **Ownership tracking** — مين حاجز الـ pin ده
- **Active-low inversion** — بيعكس القيمة تلقائياً لو الـ pin active-low
- **IRQ mapping** — ربط الـ GPIO بـ Linux IRQ number
- **Pinctrl integration** — تنسيق مع الـ pin mux subsystem
- **Debugfs/sysfs exposure** — للـ debugging والـ userspace access

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        Consumer Drivers                          │
  │  LED driver   Button driver   SPI CS driver   Reset driver       │
  │       │             │               │               │            │
  │       └─────────────┴───────────────┴───────────────┘            │
  │                             │                                    │
  │              gpiod_get() / gpiod_set_value()                     │
  │              (linux/gpio/consumer.h)                             │
  └──────────────────────────┬──────────────────────────────────────┘
                             │
  ┌──────────────────────────▼──────────────────────────────────────┐
  │                      GPIOLIB Core                                │
  │                                                                   │
  │   ┌──────────────┐  ┌───────────────┐  ┌────────────────────┐   │
  │   │  gpio_desc   │  │  gpio_device  │  │  Lookup Tables     │   │
  │   │  (per-pin    │  │  (per-chip    │  │  (DT / ACPI /      │   │
  │   │   metadata)  │  │   state)      │  │   board files)     │   │
  │   └──────┬───────┘  └──────┬────────┘  └────────────────────┘   │
  │          └─────────────────┘                                     │
  │                   │                                              │
  │         Active-low inversion, ownership, IRQ mapping             │
  └──────────────────────────┬──────────────────────────────────────┘
                             │  calls function pointers
  ┌──────────────────────────▼──────────────────────────────────────┐
  │                      gpio_chip ops                               │
  │   .get() .set() .direction_input() .direction_output()           │
  │   .to_irq() .request() .free() .set_config()                     │
  └──────┬───────────────────┬───────────────────┬──────────────────┘
         │                   │                   │
  ┌──────▼──────┐   ┌────────▼───────┐  ┌───────▼──────────────┐
  │  SoC GPIO   │   │  I2C/SPI GPIO  │  │  FPGA GPIO           │
  │  Controller │   │  Expander      │  │  IP Block            │
  │  (MMIO)     │   │  (e.g. MCP23017│  │                      │
  └─────────────┘   └────────────────┘  └──────────────────────┘
         │
  ┌──────▼─────────────────────────────────────────────────────┐
  │               Pinctrl Subsystem (optional)                  │
  │   pin mux + pin config (pull-up/down, drive strength)       │
  └────────────────────────────────────────────────────────────┘
```

---

### الـ Legacy API مقابل الـ Modern Descriptor API

الـ `include/linux/gpio.h` نفسه موجود عشان **backward compatibility بس**، وفي أول السطر بيقول صراحة:

```c
/*
 * NOTE: This header *must not* be included.
 *
 * This is the LEGACY GPIO bulk include file...
 * should not be included in new code.
 */
```

#### الـ Legacy API (deprecated)

الفكرة القديمة كانت: كل GPIO في النظام عنده **رقم global unique** في **global GPIO numberspace**:

```c
/* Legacy — بيستخدم رقم global */
gpio_request(42, "my-led");          // احجز GPIO رقم 42
gpio_direction_output(42, 1);        // اتجاه output، قيمة 1
gpio_set_value(42, 0);               // اكتب 0
gpio_free(42);                       // حرر
```

المشكلة: الرقم 42 ده مرتبط بـ hardware معين. لو غيّرت الـ board أو أضفت controller تاني، الأرقام بتتبدل.

#### الـ Modern Descriptor API (الصح)

```c
/* Modern — بيستخدم اسم منطقي من الـ DT */
struct gpio_desc *led;

led = devm_gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
if (IS_ERR(led))
    return PTR_ERR(led);

gpiod_set_value(led, 1);   // active-high أو active-low تلقائياً
```

في الـ Device Tree:
```dts
my_device {
    led-gpios = <&gpio1 5 GPIO_ACTIVE_HIGH>;
};
```

الـ `devm_gpiod_get` بيدور على الـ GPIO المرتبط بالـ "led" property في الـ DT ويرجع `gpio_desc *`.

---

### التشابه من الواقع — مقارنة عميقة

تخيل نظام الفنادق الكبير:

**الفندق** = الـ SoC / embedded board
**الغرف** = الـ GPIO pins
**الاستقبال (GPIOLIB Core)** = اللي بيتحكم في كل حجوزات الغرف
**مفتاح الغرفة (gpio_desc*)** = handle مش رقم غرفة raw، بيحتوي على metadata
**النزيل (Consumer Driver)** = بيطلب غرفة باسم وظيفتها ("غرفة مطلة على البحر")، مش برقمها
**موظف الصيانة (gpio_chip ops)** = بيعرف بالظبط إزاي يفتح كل باب
**دفتر الحجوزات (gpio_device)** = بيتتبع مين حاجز إيه

لما النزيل (LED driver) بيقول "عايز غرفة باسم led-power"، الاستقبال (GPIOLIB) بيبحث في الـ DT، بيلاقي الغرفة دي هي GPIO5 في Controller 1، بيرجع مفتاح (gpio_desc). النزيل مش محتاج يعرف رقم الغرفة أو أي floor controller بيخدمها.

الـ active-low inversion زي "الباب ده بينفتح بالضغط مش بالشد" — الاستقبال بيعكس الـ logic تلقائياً ومحدش يعرف.

---

### الـ Core Abstraction — الفكرة المحورية

**الـ `struct gpio_desc`** هو الـ core abstraction.

مش رقم — ده **opaque handle** بيمثل GPIO line واحدة ومعاها كل الـ metadata بتاعتها. مش مكشوف للـ consumer، بس الـ framework يستخدمه داخلياً.

**الـ `struct gpio_chip`** هو الـ hardware abstraction interface — ده اللي الـ driver بيملاه.

```
gpio_desc  ←─────────────────────────────────────────────────────┐
(per-pin handle)                                                   │
    │                                                             │
    ├── flags  (ACTIVE_LOW, IS_OUT, IS_IRQ, etc.)                 │
    ├── label  (اسم الـ consumer)                                  │
    └── *gdev ──► gpio_device ──► gpio_chip ──► .get() / .set()  │
                  (per-chip state)   (hw ops)                      │
                                                                   │
الـ consumer شايف بس الـ gpio_desc*     ──────────────────────────┘
```

#### العلاقة بين الـ structs

```
┌─────────────────────────────────────────────────────────┐
│                     gpio_chip                            │
│  label: "gpio-bcm2835"                                   │
│  base: 0   ngpio: 54                                     │
│  .get()        → bcm2835_gpio_get()                      │
│  .set()        → bcm2835_gpio_set()                      │
│  .direction_input()  → bcm2835_gpio_dir_in()             │
│  .direction_output() → bcm2835_gpio_dir_out()            │
│  .to_irq()     → bcm2835_gpio_to_irq()                   │
│  .set_config() → bcm2835_gpio_set_config()               │
│  irq: struct gpio_irq_chip { ... }  ← IRQ integration    │
│  *gpiodev ──────────────────────────────────────┐        │
└─────────────────────────────────────────────────│────────┘
                                                  │
┌─────────────────────────────────────────────────▼────────┐
│                    gpio_device (opaque to drivers)        │
│  descs[0..53]: struct gpio_desc[]                         │
│    desc[5].flags = GPIO_V2_LINE_FLAG_ACTIVE_LOW           │
│    desc[5].label = "led-power"                            │
│    desc[5].gdev  = this                                   │
└──────────────────────────────────────────────────────────┘
```

---

### الـ gpio_chip بالتفصيل — إيه اللي الـ Driver بيملاه

```c
struct gpio_chip {
    const char *label;          /* "mcp23017" أو "gpio-bcm2835" */
    struct gpio_device *gpiodev;/* internal, الـ framework بيملاه */
    struct device *parent;      /* الـ I2C/SPI device مثلاً */

    /* ── function pointers اللي الـ driver لازم يملاها ── */
    int  (*request)(struct gpio_chip *gc, unsigned int offset);
    void (*free)   (struct gpio_chip *gc, unsigned int offset);

    /* Direction control */
    int (*get_direction)   (struct gpio_chip *gc, unsigned int offset);
    int (*direction_input) (struct gpio_chip *gc, unsigned int offset);
    int (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);

    /* Value read/write */
    int  (*get)         (struct gpio_chip *gc, unsigned int offset);
    int  (*get_multiple)(struct gpio_chip *gc, unsigned long *mask,
                         unsigned long *bits);
    int  (*set)         (struct gpio_chip *gc, unsigned int offset, int value);
    int  (*set_multiple)(struct gpio_chip *gc, unsigned long *mask,
                         unsigned long *bits);

    /* Config (pull-up/down, drive strength) */
    int (*set_config)(struct gpio_chip *gc, unsigned int offset,
                      unsigned long config);

    /* IRQ mapping */
    int (*to_irq)(struct gpio_chip *gc, unsigned int offset);

    int   base;     /* أول رقم GPIO في الـ global space (deprecated) */
    u16   ngpio;    /* عدد الـ GPIO pins في الـ controller ده */
    bool  can_sleep;/* true لو الـ get/set ممكن تنام (I2C/SPI expander) */

    /* IRQ chip integration (لو GPIOLIB_IRQCHIP enabled) */
    struct gpio_irq_chip irq;
};
```

#### مثال واقعي — MCP23017 (I2C GPIO Expander)

الـ MCP23017 عنده 16 GPIO pins، بيتحكم فيهم عبر I2C. الـ driver بتاعه بيملا الـ `gpio_chip` كالتالي:

```c
static int mcp23017_probe(struct i2c_client *client)
{
    struct mcp23017 *mcp;

    mcp->chip.label          = "mcp23017";
    mcp->chip.parent         = &client->dev;
    mcp->chip.ngpio          = 16;
    mcp->chip.base           = -1;          /* dynamic allocation */
    mcp->chip.can_sleep      = true;        /* I2C يحتاج sleep */
    mcp->chip.get            = mcp23017_gpio_get;
    mcp->chip.set            = mcp23017_gpio_set;
    mcp->chip.direction_input  = mcp23017_direction_input;
    mcp->chip.direction_output = mcp23017_direction_output;

    return gpiochip_add_data(&mcp->chip, mcp);
}
```

بعد الـ `gpiochip_add_data`، الـ GPIOLIB بقى يعرف الـ 16 pin دول وبيدير حجوزاتهم.

---

### الـ gpio_irq_chip — ربط GPIO بالـ IRQ Subsystem

ده subsystem مهم داخل الـ GPIO framework. الـ GPIO controller ممكن يكون هو نفسه interrupt controller.

**المشكلة:** pin رقم 5 في GPIO controller ممكن يطلع على IRQ line معينة في الـ GIC (Generic Interrupt Controller). محتاج translation layer.

**الحل:** `struct gpio_irq_chip` جوا الـ `gpio_chip`:

```c
struct gpio_irq_chip {
    struct irq_chip   *chip;          /* الـ irq_chip ops */
    struct irq_domain *domain;        /* translation: GPIO offset → Linux IRQ */
    struct irq_domain *parent_domain; /* للـ hierarchical IRQ */

    /* Translate GPIO hwirq → parent hwirq (للـ GIC مثلاً) */
    int (*child_to_parent_hwirq)(struct gpio_chip *gc,
                                  unsigned int child_hwirq,
                                  unsigned int child_type,
                                  unsigned int *parent_hwirq,
                                  unsigned int *parent_type);

    irq_flow_handler_t handler;       /* handle_level_irq أو handle_edge_irq */
    unsigned int       default_type;  /* IRQ_TYPE_EDGE_FALLING مثلاً */
    bool               threaded;      /* true لو I2C expander */
};
```

**الـ IRQ Domain:** ده concept من الـ IRQ subsystem. هو mapping table بين **hardware IRQ numbers** (اللي الـ hardware بيعرفها) و **Linux virtual IRQ numbers** (اللي الـ kernel بيستخدمها). محتاج تفهمه عشان تفهم إزاي GPIO-to-IRQ بيشتغل.

---

### الـ Active-Low Inversion — تفصيلة مهمة

لما بتكتب في الـ DT:
```dts
reset-gpios = <&gpio1 10 GPIO_ACTIVE_LOW>;
```

الـ GPIOLIB بيخزن الـ `ACTIVE_LOW` flag في الـ `gpio_desc`. لما Consumer بيعمل:

```c
gpiod_set_value(reset_gpio, 1);  /* "assert reset" */
```

الـ framework بيشوف الـ flag ويكتب على الـ hardware القيمة 0 (عكس 1). الـ consumer driver مش محتاج يعرف إيه polarity الـ hardware — بيفكر في logic values بس.

```
Consumer:  gpiod_set_value(desc, 1)
               │
               ▼
         [GPIOLIB Core]
         if (desc->flags & GPIO_ACTIVE_LOW)
             value = !value;   /* 1 → 0 */
               │
               ▼
         chip->set(gc, offset, 0)   /* Hardware sees 0 */
```

لو استخدمت الـ raw functions (`gpiod_set_raw_value`)، الـ inversion مش بتحصل.

---

### الـ devm_ Pattern — إزاي بيشتغل

الـ `devm_` prefix يعني **device-managed resource**. الـ GPIOLIB بيتسجل مع الـ devres infrastructure:

```c
/* الـ driver بيطلب GPIO */
led = devm_gpiod_get(&pdev->dev, "led", GPIOD_OUT_LOW);
```

لما الـ device بتتفصل (driver unbind)، الـ kernel تلقائياً بيـ call `gpiod_put(led)` ويحرر الـ GPIO. مش محتاج `remove()` callback يعمل cleanup.

ده بيتعمل عن طريق الـ **devres subsystem** — كل device عنده قايمة من الـ resources، ولما بيتـ detach يحررهم كلهم.

---

### الـ gpio_descs — Array Operations

لما عندك LEDs matrix أو data bus محتاج تكتب فيه 8 bits دفعة واحدة:

```c
struct gpio_descs *data_lines;

data_lines = devm_gpiod_get_array(&pdev->dev, "data", GPIOD_OUT_LOW);

/* كتابة 8 bits دفعة واحدة */
unsigned long values = 0b10110101;
gpiod_set_array_value(data_lines->ndescs,
                       data_lines->desc,
                       data_lines->info,
                       &values);
```

الـ `gpio_array` الـ opaque struct (`data_lines->info`) بيسمح للـ framework يعمل optimization لو الـ pins كلها في نفس الـ controller — بيكتبهم في register operation واحدة بدل N operations.

---

### إيه اللي الـ Framework بيملكه مقابل ما بيفوّضه للـ Driver

| المسؤولية | الـ GPIOLIB Core | الـ gpio_chip Driver |
|---|---|---|
| Ownership tracking (مين حاجز الـ pin) | **GPIOLIB** | — |
| Active-low inversion | **GPIOLIB** | — |
| IRQ domain management | **GPIOLIB** | يوفر `child_to_parent_hwirq` |
| Debugfs entries | **GPIOLIB** | ممكن يضيف `dbg_show` |
| sysfs export | **GPIOLIB** | — |
| DT/ACPI lookup | **GPIOLIB** | — |
| Hardware register access | — | **Driver** |
| Direction set/get | — | **Driver** |
| Value read/write | — | **Driver** |
| Sleep/can_sleep decision | — | **Driver** (`can_sleep` flag) |
| Pin config (pull-up, drive strength) | delegates via `set_config` | **Driver** implements |
| Pinctrl coordination | **GPIOLIB** calls `add_pin_ranges` | **Driver** implements |

---

### الـ Legacy vs Modern API — إزاي الـ Shim بيشتغل

الـ `gpio.h` اللي بنشتغل عليه ده بيعمل thin wrapper بين الـ legacy API والـ modern one:

```c
/* من gpio.h — legacy wrapper */
static inline int gpio_direction_input(unsigned gpio)
{
    /* بيحول الرقم لـ descriptor أولاً */
    return gpiod_direction_input(gpio_to_desc(gpio));
}

static inline void gpio_set_value(unsigned gpio, int value)
{
    /* gpio_to_desc بيبحث في الـ global numberspace */
    gpiod_set_raw_value(gpio_to_desc(gpio), value);
}
```

الـ `gpio_to_desc()` بيبحث في الـ global GPIO numberspace ويرجع الـ `gpio_desc*` المقابل. ده الجسر بين الـ عالمين.

لو `CONFIG_GPIOLIB` مش enabled، كل الفنكشنز بترجع `-ENOSYS` أو بتعمل `WARN_ON(1)` — compile-time safety.

---

### الـ Pinctrl Integration — علاقة مهمة

**الـ Pinctrl Subsystem** بيتحكم في الـ pin multiplexing — نفس الـ physical pin ممكن يكون GPIO أو UART TX أو SPI MOSI حسب الـ mux setting. ده subsystem تاني مختلف.

الـ GPIO framework بيتنسق معاه عبر:

```c
struct gpio_pin_range {
    struct list_head      node;      /* قايمة من الـ ranges */
    struct pinctrl_dev   *pctldev;   /* الـ pinctrl device */
    struct pinctrl_gpio_range range; /* الـ mapping: GPIO offsets → pin numbers */
};
```

لما بتعمل `gpiochip_add_pin_range()` في الـ driver، بتقول للـ GPIO framework: "الـ GPIO offset 0..15 في الـ chip دي بيتوافق مع الـ pins 100..115 في الـ pinctrl device". ده بيخلي الـ GPIOLIB يطلب الـ pin مع الـ pinctrl لما حد يـ request الـ GPIO.

---

### الـ cansleep Variants — تفصيلة الـ Context

```c
/* Non-sleeping context (ISR, spinlock held) */
int  gpiod_get_value(const struct gpio_desc *desc);
int  gpiod_set_value(struct gpio_desc *desc, int value);

/* Sleeping context allowed (process context only) */
int  gpiod_get_value_cansleep(const struct gpio_desc *desc);
int  gpiod_set_value_cansleep(struct gpio_desc *desc, int value);
```

لو الـ `gpio_chip.can_sleep = true` (I2C/SPI expander)، استخدام الـ non-sleep variants من ISR هيعمل `might_sleep()` WARN. الـ cansleep variants بتأكد إن الكود شغال في process context فقط.

---

### الـ CONFIG Guards — ليه التعقيد ده

في الـ header، كل حاجة محاطة بـ `#ifdef`:

```
CONFIG_GPIOLIB          → الـ gpiolib نفسه enabled
CONFIG_GPIOLIB_LEGACY   → الـ legacy numberspace API enabled
CONFIG_GPIOLIB_IRQCHIP  → GPIO-to-IRQ integration enabled
CONFIG_OF_GPIO          → Device Tree GPIO support
CONFIG_PINCTRL          → Pin control integration
CONFIG_ACPI             → ACPI GPIO support
CONFIG_GPIO_SYSFS       → sysfs export support
CONFIG_HTE              → Hardware Timestamp Engine support
```

لو `CONFIG_GPIOLIB` مش موجود (مثلاً في minimal embedded config)، كل الفنكشنز بتتحول لـ stubs بتعمل `return -ENOSYS` أو `WARN_ON(1)`. ده بيضمن إن الـ consumer drivers بتـ compile على طول بدون الـ gpiolib، وبتـ fail gracefully at runtime.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### الـ GPIOF_* flags (Legacy — `include/linux/gpio.h`)

| Flag | القيمة | المعنى |
|------|--------|--------|
| `GPIOF_IN` | `(1 << 0)` = `0x1` | اتجاه input |
| `GPIOF_OUT_INIT_LOW` | `(0<<0)\|(0<<1)` = `0x0` | output، القيمة الابتدائية low |
| `GPIOF_OUT_INIT_HIGH` | `(0<<0)\|(1<<1)` = `0x2` | output، القيمة الابتدائية high |

> دول الـ flags القديمة اللي بتتبعت مع `gpio_request_one()`.

---

#### الـ GPIOD_FLAGS_BIT_* (internal bits — `consumer.h`)

| Bit Macro | Bit Position | الوظيفة |
|-----------|-------------|---------|
| `GPIOD_FLAGS_BIT_DIR_SET` | BIT(0) | اتحدد اتجاه صراحةً |
| `GPIOD_FLAGS_BIT_DIR_OUT` | BIT(1) | الاتجاه output |
| `GPIOD_FLAGS_BIT_DIR_VAL` | BIT(2) | القيمة الابتدائية high |
| `GPIOD_FLAGS_BIT_OPEN_DRAIN` | BIT(3) | open-drain mode |
| `GPIOD_FLAGS_BIT_NONEXCLUSIVE` | BIT(4) | **deprecated** — مش استخدمه |

---

#### الـ `enum gpiod_flags` (Public API — `consumer.h`)

| Enum Value | القيمة الرقمية | الاستخدام |
|------------|--------------|-----------|
| `GPIOD_ASIS` | `0` | ماتغيرش أي حاجة |
| `GPIOD_IN` | `BIT(0)` | input |
| `GPIOD_OUT_LOW` | `BIT(0)\|BIT(1)` | output، يبدأ بـ low |
| `GPIOD_OUT_HIGH` | `BIT(0)\|BIT(1)\|BIT(2)` | output، يبدأ بـ high |
| `GPIOD_OUT_LOW_OPEN_DRAIN` | `GPIOD_OUT_LOW\|BIT(3)` | open-drain، low |
| `GPIOD_OUT_HIGH_OPEN_DRAIN` | `GPIOD_OUT_HIGH\|BIT(3)` | open-drain، high |

---

#### الـ ACPI GPIO Quirks (`consumer.h`)

| Macro | Bit | المعنى |
|-------|-----|--------|
| `ACPI_GPIO_QUIRK_NO_IO_RESTRICTION` | BIT(0) | تجاهل حقل `IoRestriction` في ACPI |
| `ACPI_GPIO_QUIRK_ONLY_GPIOIO` | BIT(1) | جيب بس نوع `GpioIo` من الـ _CRS |
| `ACPI_GPIO_QUIRK_ABSOLUTE_NUMBER` | BIT(2) | استخدم الـ pin كـ absolute GPIO number |

---

#### الـ Config Options المهمة

| Kconfig | الأثر |
|---------|-------|
| `CONFIG_GPIOLIB` | يفعّل كل الـ gpiolib core — بدونه كل الـ functions بترجع `-ENOSYS` أو `WARN_ON` |
| `CONFIG_GPIOLIB_LEGACY` | يفعّل الـ legacy number-space API في `gpio.h` |
| `CONFIG_GPIOLIB_IRQCHIP` | يدخل `gpio_irq_chip` جوه `gpio_chip` |
| `CONFIG_GPIO_SYSFS` | يفعّل `gpiod_export()` / `gpiod_unexport()` |
| `CONFIG_ACPI` | يفعّل `acpi_gpio_mapping` support |
| `CONFIG_OF_GPIO` | يفعّل Device Tree xlate داخل `gpio_chip` |
| `CONFIG_HTE` | Hardware Timestamp Engine — `gpiod_enable_hw_timestamp_ns()` |

---

### 1. الـ Structs المهمة

---

#### `struct gpio_desc` (opaque)

**الـ GPIO descriptor** — الـ handle الحديث لأي GPIO line.
الـ struct نفسه opaque (defined داخل `gpiolib.c`)، بس كل الـ API بيتعامل معاه عن طريق pointer.

| الحقل (داخلي) | النوع | الوظيفة |
|--------------|-------|---------|
| `gdev` | `*gpio_device` | الـ device اللي بيملك الـ descriptor |
| `flags` | `unsigned long` | الـ direction، active-low، open-drain، إلخ |
| `label` | `const char *` | اسم الـ consumer (للـ debugging) |
| `name` | `const char *` | اسم الـ GPIO line من الـ firmware |

**الـ connections:**
- مرتبط بـ `gpio_chip` عن طريق `gpio_device`
- الـ `gpio_to_desc(gpio)` بتحوّله من legacy number للـ descriptor
- الـ `desc_to_gpio(desc)` بتعمل العكس

---

#### `struct gpio_descs` (`consumer.h`)

مصفوفة من الـ descriptors بتتجمع مع بعض لتشغيل عدة GPIO lines في نفس الوقت (bulk ops).

```c
struct gpio_descs {
    struct gpio_array *info;   /* opaque — معلومات التحسين للـ bulk ops */
    unsigned int ndescs;       /* عدد الـ descriptors */
    struct gpio_desc *desc[];  /* flexible array من الـ descriptors */
};
```

| الحقل | الوظيفة |
|-------|---------|
| `info` | pointer لـ `gpio_array` اللي بتحتوي معلومات hardware-level تسرّع الـ bulk operations |
| `ndescs` | عدد الـ GPIO lines في المجموعة |
| `desc[]` | الـ flexible array — كل عنصر pointer لـ `gpio_desc` |

**مثال واقعي:** LED driver بيتحكم في 8 LEDs — بيجيبهم بـ `gpiod_get_array()` كـ `gpio_descs`، وبيشغلهم كلهم دفعة واحدة بـ `gpiod_set_array_value()`.

---

#### `struct gpio_chip` (`driver.h`)

**الواجهة الرئيسية للـ GPIO controller driver** — الـ struct اللي بيكتبه صاحب الـ driver.

```c
struct gpio_chip {
    const char          *label;       /* اسم الـ controller */
    struct gpio_device  *gpiodev;     /* internal state — readonly للـ driver */
    struct device       *parent;      /* الـ device الأب */
    struct fwnode_handle *fwnode;     /* firmware node */
    struct module       *owner;       /* THIS_MODULE */

    /* ops — الـ driver بيملأهم */
    int  (*request)(struct gpio_chip *gc, unsigned int offset);
    void (*free)(struct gpio_chip *gc, unsigned int offset);
    int  (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)(struct gpio_chip *gc, unsigned int offset);
    int  (*get_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int  (*set)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*set_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int  (*set_config)(struct gpio_chip *gc, unsigned int offset, unsigned long config);
    int  (*to_irq)(struct gpio_chip *gc, unsigned int offset);

    int   base;       /* أول GPIO number — deprecated، استخدم -1 */
    u16   ngpio;      /* عدد الـ GPIO lines */
    u16   offset;     /* offset جوه الـ device لو فيه أكتر من chip */
    const char *const *names; /* أسماء بديلة للـ lines */
    bool  can_sleep;  /* true لو الـ get/set ممكن تنام (I2C/SPI expanders) */

    struct gpio_irq_chip irq;         /* CONFIG_GPIOLIB_IRQCHIP */
    unsigned int of_gpio_n_cells;     /* CONFIG_OF_GPIO */
    int (*of_xlate)(...);             /* CONFIG_OF_GPIO */
};
```

**أهم الـ ops:**

| Callback | متى بيتاستدعى | Notes |
|----------|--------------|-------|
| `request` | لما consumer يعمل `gpiod_get()` | enable power/clock |
| `free` | لما consumer يعمل `gpiod_put()` | disable power/clock |
| `get_direction` | `gpiod_get_direction()` | يرجع 0=out، 1=in |
| `direction_input` | `gpiod_direction_input()` | يحول الـ pin لـ input |
| `direction_output` | `gpiod_direction_output()` | يحول لـ output + يحدد القيمة |
| `get` | `gpiod_get_value()` | يقرأ القيمة |
| `get_multiple` | `gpiod_get_array_value()` | bulk read |
| `set` | `gpiod_set_value()` | يكتب القيمة |
| `set_multiple` | `gpiod_set_array_value()` | bulk write |
| `set_config` | `gpiod_set_debounce()` و `gpiod_set_config()` | pinconf |
| `to_irq` | `gpiod_to_irq()` | حوّل GPIO offset لـ IRQ number |

---

#### `struct gpio_irq_chip` (`driver.h`)

بيدمج الـ IRQ controller جوه الـ GPIO chip.

```c
struct gpio_irq_chip {
    struct irq_chip        *chip;           /* الـ irq_chip ops */
    struct irq_domain      *domain;         /* mapping: hwirq <-> Linux IRQ */
    struct fwnode_handle   *fwnode;         /* للـ hierarchical IRQ domains */
    struct irq_domain      *parent_domain;  /* الـ parent لو hierarchical */

    irq_flow_handler_t      handler;        /* مثلاً handle_level_irq */
    unsigned int            default_type;   /* IRQ_TYPE_LEVEL_HIGH إلخ */
    irq_flow_handler_t      parent_handler; /* cascaded parent ISR */
    unsigned int            num_parents;
    unsigned int           *parents;        /* parent IRQ numbers */
    unsigned int           *map;            /* line -> parent IRQ */
    bool                    threaded;       /* threaded IRQs? */
    bool                    initialized;
    unsigned long          *valid_mask;     /* bitmap of IRQ-capable lines */

    /* callbacks */
    int  (*init_hw)(struct gpio_chip *gc);
    void (*init_valid_mask)(struct gpio_chip *gc, unsigned long *vm, unsigned int n);
    void (*irq_enable)(struct irq_data *data);
    void (*irq_disable)(struct irq_data *data);
    void (*irq_unmask)(struct irq_data *data);
    void (*irq_mask)(struct irq_data *data);
};
```

---

#### `struct acpi_gpio_params` (`consumer.h`)

بيوصف GPIO واحد داخل ACPI table.

```c
struct acpi_gpio_params {
    unsigned int   crs_entry_index; /* index في _CRS method */
    unsigned short line_index;      /* line داخل الـ GPIO resource */
    bool           active_low;      /* polarity */
};
```

---

#### `struct acpi_gpio_mapping` (`consumer.h`)

جدول mapping بين اسم function وـ GPIO resources في ACPI.

```c
struct acpi_gpio_mapping {
    const char                  *name;    /* مثلاً "reset-gpios" */
    const struct acpi_gpio_params *data;  /* array من الـ params */
    unsigned int                 size;    /* حجم الـ data array */
    unsigned int                 quirks;  /* ACPI_GPIO_QUIRK_* */
};
```

**مثال واقعي:**
```c
static const struct acpi_gpio_params reset_gpio = { 0, 0, false };
static const struct acpi_gpio_params irq_gpio   = { 1, 0, false };

static const struct acpi_gpio_mapping mychip_gpios[] = {
    { "reset-gpios", &reset_gpio, 1 },
    { "irq-gpios",   &irq_gpio,   1 },
    { }
};
/* في probe: */
acpi_dev_add_driver_gpios(adev, mychip_gpios);
```

---

### 2. رسومات العلاقات بين الـ Structs

```
┌─────────────────────────────────────────────────────────────────┐
│                     gpio_chip (driver side)                     │
│  label, ngpio, base, can_sleep                                  │
│  ops: request/free/get/set/direction_input/direction_output ... │
│  ┌─────────────────────┐   ┌──────────────────────────────┐    │
│  │  gpio_irq_chip      │   │  gpio_device * (gpiodev)     │    │
│  │  irq_chip *         │   │  (opaque — owned by gpiolib) │    │
│  │  irq_domain *       │   └──────────────┬───────────────┘    │
│  │  parent_domain *    │                  │                     │
│  └─────────────────────┘                  │                     │
└──────────────────────────────────────────-│─────────────────────┘
                                            │ contains array of
                                            ▼
                              ┌─────────────────────────┐
                              │  gpio_desc[]             │
                              │  (one per GPIO line)     │
                              │  flags, label, name      │
                              │  *gdev → gpio_device     │
                              └────────────┬────────────┘
                                           │  bundled into
                                           ▼
                              ┌──────────────────────────┐
                              │  gpio_descs              │
                              │  ndescs                  │
                              │  *info → gpio_array      │
                              │  *desc[] → gpio_desc[]   │
                              └──────────────────────────┘

ACPI Path:
┌──────────────────────────┐
│  acpi_gpio_mapping[]     │──→ acpi_gpio_params[]
│  name / data / quirks    │
└──────────────────────────┘
        │ acpi_dev_add_driver_gpios()
        ▼
┌──────────────────────────┐
│  acpi_device             │
│  (ACPI subsystem)        │
└──────────────────────────┘
        │ gpiod_get(dev, "reset-gpios", ...)
        ▼
┌──────────────────────────┐
│  gpio_desc *             │
└──────────────────────────┘

Legacy Path (gpio.h):
  gpio number (unsigned int)
        │ gpio_to_desc(gpio)
        ▼
  gpio_desc *
        │ gpiod_*() functions
        ▼
  hardware register
```

---

### 3. دورة الحياة — Lifecycle Diagrams

#### دورة حياة الـ GPIO Controller (Driver Side)

```
Driver probe()
    │
    ├── allocate gpio_chip
    │
    ├── fill gpio_chip fields:
    │     label, ngpio, base=-1, can_sleep
    │     ops: request, free, get, set, ...
    │
    ├── (optional) fill gpio_chip.irq:
    │     irq_chip, handler, default_type, parents, ...
    │
    ├── gpiochip_add_data(gc, driver_data)
    │     ├── allocate gpio_device
    │     ├── allocate gpio_desc[] array (ngpio entries)
    │     ├── assign dynamic base (if base == -1)
    │     ├── register irq_domain (if irq.chip set)
    │     └── register in sysfs / debugfs
    │
    │   [CONTROLLER IS LIVE — consumers can request GPIOs now]
    │
Driver remove()
    │
    └── gpiochip_remove(gc)
          ├── free gpio_desc[] array
          ├── unregister irq_domain
          ├── remove from sysfs / debugfs
          └── free gpio_device
```

---

#### دورة حياة الـ GPIO Line (Consumer Side)

```
Consumer driver probe()
    │
    ├── gpiod_get(dev, "reset", GPIOD_OUT_LOW)
    │     ├── lookup firmware node (DT / ACPI / board file)
    │     ├── call gpio_chip->request(gc, offset)   [optional]
    │     ├── set direction via gpio_chip->direction_output(gc, offset, 0)
    │     └── return gpio_desc *
    │
    ├── use GPIO:
    │     gpiod_set_value(desc, 1)   → gpio_chip->set(gc, offset, 1)
    │     gpiod_get_value(desc)      → gpio_chip->get(gc, offset)
    │     gpiod_to_irq(desc)         → gpio_chip->to_irq(gc, offset)
    │     gpiod_set_debounce(desc, 1000) → gpio_chip->set_config(...)
    │
Consumer driver remove()
    │
    └── gpiod_put(desc)
          ├── call gpio_chip->free(gc, offset)   [optional]
          └── mark gpio_desc as free
```

> لو استخدمت `devm_gpiod_get()` بدل `gpiod_get()` — الـ kernel بيعمل `gpiod_put()` أوتوماتيك لما الـ device يتفصل.

---

#### دورة حياة الـ Bulk GPIO (gpio_descs)

```
gpiod_get_array(dev, "leds", GPIOD_OUT_LOW)
    │
    ├── allocate gpio_descs (ndescs = N)
    ├── for each line: lookup + request + direction_output
    ├── build gpio_array (bulk optimization info)
    └── return gpio_descs *

use:
    gpiod_set_array_value_cansleep(descs->ndescs, descs->desc,
                                   descs->info, value_bitmap)
        └── gpio_chip->set_multiple() if available
            else loop: gpio_chip->set() per line

gpiod_put_array(descs)
    ├── for each line: gpio_chip->free()
    └── kfree(descs)
```

---

### 4. Call Flow Diagrams

#### `gpio_set_value(gpio, 1)` — Legacy Path

```
gpio_set_value(gpio, 1)           [gpio.h — inline]
    │
    └── gpiod_set_raw_value(gpio_to_desc(gpio), 1)
              │
              ├── gpio_to_desc(gpio)
              │     └── lookup gpio_desc in global gpio_device list
              │
              └── gpiod_set_raw_value(desc, 1)
                        │
                        ├── validate desc != NULL
                        ├── check !desc->can_sleep (would WARN in atomic context)
                        └── gpio_chip->set(gc, offset, 1)
                                  │
                                  └── write to hardware register
                                      (e.g., GPIODATA |= (1 << offset))
```

---

#### `gpiod_get(dev, "reset", GPIOD_OUT_LOW)` — Modern Path

```
gpiod_get(dev, "reset", GPIOD_OUT_LOW)
    │
    ├── gpiod_find_and_request()
    │     ├── [DT]   of_find_gpio()   → of_xlate() → gpio_desc
    │     ├── [ACPI] acpi_find_gpio() → acpi_gpio_mapping lookup → gpio_desc
    │     └── [board] gpiod_find()    → gpio_lookup_table → gpio_desc
    │
    ├── gpio_chip->request(gc, offset)    [optional — enable clk/power]
    │
    ├── apply GPIOD_OUT_LOW:
    │     gpio_chip->direction_output(gc, offset, 0)
    │
    └── return gpio_desc *
```

---

#### `gpiod_to_irq(desc)` — GPIO as Interrupt

```
gpiod_to_irq(desc)
    │
    └── gpio_chip->to_irq(gc, offset)
              │
              ├── [simple]  irq_find_mapping(gc->irq.domain, offset)
              │               └── return Linux IRQ number
              │
              └── [hierarchical]
                    child_to_parent_hwirq(gc, offset, ...)
                        └── irq_domain_translate → Linux IRQ number

then consumer:
    request_irq(irq, handler, IRQF_TRIGGER_RISING, "mydev", dev)
        └── irq_chip->irq_enable() / irq_unmask()
              └── set interrupt mask register in hardware
```

---

#### `gpiod_set_array_value_cansleep()` — Bulk Path

```
gpiod_set_array_value_cansleep(n, desc_array, array_info, bitmap)
    │
    ├── [fast path] array_info != NULL && chip supports set_multiple:
    │     gpio_chip->set_multiple(gc, mask, bits)
    │           └── single register write for all lines
    │
    └── [slow path] loop:
          for each bit set in bitmap:
              gpio_chip->set(gc, offset[i], value[i])
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks الموجودة في الـ GPIO Subsystem

| Lock | النوع | يحمي إيه | أين |
|------|-------|-----------|-----|
| `gpio_lock` | `spinlock` (داخل gpiolib.c) | الـ gpio_device list و gpio_desc flags | gpiolib core |
| `gc->irq.lock_key` | `lock_class_key` (lockdep) | الـ IRQ lock للـ irq_chip | gpio_irq_chip |
| `gc->irq.request_key` | `lock_class_key` (lockdep) | الـ IRQ request lock | gpio_irq_chip |
| `irq_domain->mutex` | `mutex` | الـ IRQ domain mappings | IRQ subsystem |

---

#### قواعد الـ Locking

**1. Atomic Context:**

لو `gpio_chip->can_sleep == false`:
- `gpio_get_value()` و `gpio_set_value()` آمنين في atomic context (ISR، spinlock held).
- **ممنوع** استخدام `gpio_get_value_cansleep()` في atomic context.

لو `gpio_chip->can_sleep == true` (I2C/SPI expanders):
- لازم تستخدم `_cansleep` variants دايماً.
- الـ IRQs بتكون **threaded** إجبارياً (`gc->irq.threaded = true`).

**2. IRQ Locking Order:**

```
gpio spinlock
    └── irq_chip lock (nested — محدد بـ lock_key للـ lockdep)
          └── hardware register access
```

> الـ `lock_class_key` بيخلي lockdep يعرف إن الـ nesting ده مقصود ومش deadlock.

**3. Descriptor Flags:**

الـ `gpio_desc->flags` بيتعمله set/clear بـ `spinlock` محمي — لأنه بيتقرأ ويتكتب من threads مختلفة (consumer thread + IRQ thread).

**4. devm و Lifetimes:**

- الـ `devm_gpiod_get()` بيربط الـ descriptor بـ device lifetime.
- الـ release بييجي عبر `devres` — مش لازم lock صريح، لأن الـ device teardown بيحصل في سياق محكوم.

---

#### مثال: I2C GPIO Expander (can_sleep = true)

```
Thread A: gpiod_set_value_cansleep(desc, 1)
    │
    └── might_sleep() assert — OK، ده thread context
        └── gpio_chip->set() → i2c_transfer() → [sleep OK]

ISR context: gpiod_set_value(desc, 1)   ← WRONG! WARN_ON triggered
    │
    └── gpio_chip->can_sleep == true → BUG
```

لو الـ GPIO expander بيدي IRQ — الـ IRQ handler لازم يكون threaded:
```c
gc->irq.threaded = true;
/* gpiolib automatically forces IRQF_ONESHOT on consumer's request_irq */
```
## Phase 4: شرح الـ Functions

> **ملاحظة معمارية مهمة:** الـ `linux/gpio.h` ده الـ **legacy GPIO API** اللي بيشتغل على الـ global GPIO numberspace (أرقام صحيحة زي `gpio=17`). الـ modern API بقت الـ descriptor-based اللي موجودة في `linux/gpio/consumer.h`. كل الـ functions اللي في `gpio.h` هي wrappers بتترجم من رقم GPIO لـ `gpio_desc*` عن طريق `gpio_to_desc()` وبعدين بتستدعي النظير الـ `gpiod_*` المقابل.

---

### جدول ملخص - Legacy GPIO API (`linux/gpio.h`)

| Function | النوع | الغرض |
|---|---|---|
| `gpio_is_valid()` | helper | يتحقق إن رقم GPIO صالح |
| `gpio_request()` | acquire | يحجز GPIO line |
| `gpio_request_one()` | acquire | يحجز GPIO ويضبط اتجاهه فورًا |
| `gpio_free()` | release | يحرر GPIO line |
| `devm_gpio_request_one()` | devres acquire | يحجز GPIO مع auto-cleanup |
| `gpio_direction_input()` | config | يضبط GPIO كـ input |
| `gpio_direction_output()` | config | يضبط GPIO كـ output بقيمة أولية |
| `gpio_get_value()` | I/O (atomic) | يقرأ قيمة GPIO من context غير نايم |
| `gpio_set_value()` | I/O (atomic) | يكتب قيمة GPIO من context غير نايم |
| `gpio_get_value_cansleep()` | I/O (sleepable) | يقرأ قيمة GPIO من sleepable context |
| `gpio_set_value_cansleep()` | I/O (sleepable) | يكتب قيمة GPIO من sleepable context |
| `gpio_to_irq()` | conversion | يحوّل رقم GPIO لـ IRQ number |

---

### جدول ملخص - Descriptor-based GPIO API (`linux/gpio/consumer.h`)

#### Acquisition / Release

| Function | الغرض |
|---|---|
| `gpiod_get()` | يجيب descriptor لـ GPIO مرتبط بـ device |
| `gpiod_get_index()` | يجيب descriptor برقم index في مجموعة GPIOs |
| `gpiod_get_optional()` | زي `gpiod_get` لكن بيرجع NULL لو مش موجود |
| `gpiod_get_array()` | يجيب array من الـ descriptors |
| `gpiod_put()` | يحرر descriptor |
| `gpiod_put_array()` | يحرر array من الـ descriptors |
| `devm_gpiod_get()` | نسخة devres من `gpiod_get` |
| `devm_gpiod_get_optional()` | نسخة devres اختيارية |
| `devm_gpiod_put()` | يحرر devres descriptor |
| `devm_gpiod_unhinge()` | ينفصل عن الـ devres management بدون تحرير |

#### Direction / Config

| Function | الغرض |
|---|---|
| `gpiod_direction_input()` | يضبط input (مع active-low awareness) |
| `gpiod_direction_output()` | يضبط output (مع active-low awareness) |
| `gpiod_direction_output_raw()` | يضبط output بقيمة physical بدون تطبيق active-low |
| `gpiod_get_direction()` | يقرأ الاتجاه الحالي |
| `gpiod_set_debounce()` | يضبط debounce time بـ microseconds |
| `gpiod_set_config()` | يضبط config عام (pinconf encoding) |
| `gpiod_toggle_active_low()` | يعكس الـ active-low polarity |

#### Value Get/Set

| Function | Context | Active-low aware? |
|---|---|---|
| `gpiod_get_value()` | atomic | نعم |
| `gpiod_set_value()` | atomic | نعم |
| `gpiod_get_raw_value()` | atomic | لا (physical) |
| `gpiod_set_raw_value()` | atomic | لا (physical) |
| `gpiod_get_value_cansleep()` | sleepable | نعم |
| `gpiod_set_value_cansleep()` | sleepable | نعم |
| `gpiod_get_raw_value_cansleep()` | sleepable | لا |
| `gpiod_set_raw_value_cansleep()` | sleepable | لا |
| `gpiod_get_array_value()` | atomic bulk | نعم |
| `gpiod_set_array_value()` | atomic bulk | نعم |
| `gpiod_set_array_value_cansleep()` | sleepable bulk | نعم |

#### Utility / Info

| Function | الغرض |
|---|---|
| `gpiod_to_irq()` | يحوّل descriptor لـ IRQ number |
| `gpiod_is_active_low()` | يتحقق من الـ polarity |
| `gpiod_cansleep()` | يتحقق لو الـ GPIO يحتاج sleeping context |
| `gpiod_is_shared()` | يتحقق لو الـ GPIO مشترك بين أكتر من consumer |
| `gpiod_is_equal()` | يقارن descriptorين |
| `gpiod_hwgpio()` | يرجع hardware GPIO number داخل الـ chip |
| `gpiod_set_consumer_name()` | يضبط اسم الـ consumer (للـ debugfs) |
| `gpio_to_desc()` | يحوّل رقم legacy لـ descriptor |
| `desc_to_gpio()` | يحوّل descriptor لرقم legacy |
| `gpiod_count()` | يرجع عدد الـ GPIOs المرتبطة بـ function |

#### Firmware / ACPI

| Function | الغرض |
|---|---|
| `fwnode_gpiod_get_index()` | يجيب GPIO من firmware node مباشرة |
| `devm_fwnode_gpiod_get_index()` | نسخة devres |
| `devm_fwnode_gpiod_get()` | wrapper بيجيب index=0 |
| `devm_fwnode_gpiod_get_optional()` | wrapper اختياري |
| `acpi_dev_add_driver_gpios()` | يضيف ACPI GPIO mapping للـ driver |
| `acpi_dev_remove_driver_gpios()` | يشيل الـ ACPI GPIO mapping |

#### Sysfs / Export

| Function | الغرض |
|---|---|
| `gpiod_export()` | يعمل export للـ GPIO في sysfs |
| `gpiod_export_link()` | يعمل symlink في sysfs |
| `gpiod_unexport()` | يشيل الـ GPIO من sysfs |

#### Hardware Timestamp

| Function | الغرض |
|---|---|
| `gpiod_enable_hw_timestamp_ns()` | يفعّل الـ hardware timestamping (يحتاج CONFIG_HTE) |
| `gpiod_disable_hw_timestamp_ns()` | يوقف الـ hardware timestamping |

---

### Category 1: Validation & Legacy Compatibility

#### `gpio_is_valid()`

```c
static inline bool gpio_is_valid(int number)
{
    return number >= 0; /* only non-negative numbers are valid */
}
```

الـ function دي بتتحقق إن رقم GPIO مش negative، لأن الـ legacy API بتستخدم أرقام صحيحة وبعض الـ platform code بتستخدم `-1` أو قيم سالبة للدلالة على "no GPIO". في الحالة اللي GPIOLIB مش موجودة، بترجع `false` دايمًا.

- **Parameters:** `number` — رقم GPIO المراد التحقق منه
- **Return:** `true` لو الرقم >= 0، `false` غير كده
- **Key detail:** مش بتتحقق من وجود الـ GPIO فعلًا في الـ hardware، بس من صحة الرقم كـ convention

---

#### `gpio_to_desc()` / `desc_to_gpio()`

```c
struct gpio_desc *gpio_to_desc(unsigned gpio);
int desc_to_gpio(const struct gpio_desc *desc);
```

**الـ `gpio_to_desc()`** هي حلقة الوصل بين الـ legacy numberspace والـ modern descriptor system. بتبحث في الـ registered `gpio_chip` controllers عن الـ chip اللي بتمتلك الرقم ده، وبترجع الـ `gpio_desc*` المقابل. لو GPIOLIB مش موجودة بترجع `NULL`.

**الـ `desc_to_gpio()`** بتعمل العكس، مفيدة في code بيتعامل مع الاتنين APIs.

- **Locking:** الـ `gpio_to_desc` بتحتاج `gpio_lock` داخليًا في الـ gpiolib
- **Caller context:** ممكن تتاعت من أي context

---

### Category 2: Legacy Request / Free

#### `gpio_request()`

```c
int gpio_request(unsigned gpio, const char *label);
```

بتحجز GPIO line عشان يبقى ملكك وتضمن مفيش حد تاني يستخدمه. الـ `label` بيظهر في `/sys/kernel/debug/gpio` وفي الـ kernel messages. الـ function دي مش inline لأنها بتستدعي كود من الـ gpiolib مباشرة.

- **Parameters:**
  - `gpio` — رقم الـ GPIO في الـ global numberspace
  - `label` — string وصفي للـ consumer (مش بيتحرر)
- **Return:** `0` على النجاح، أو error code سالب (مثلاً `-EBUSY` لو محجوز، `-EINVAL` لو رقم غلط)
- **Key detail:** بتستدعي `gpiod_request()` داخليًا بعد تحويل الرقم. الـ `might_sleep()` بيتعمل داخل الـ gpiolib لأن بعض الـ GPIO controllers بتحتاج I2C/SPI access
- **Caller context:** Process context فقط (ممكن تنام)

---

#### `gpio_free()`

```c
void gpio_free(unsigned gpio);
```

بتحرر GPIO line اللي اتحجز قبل كده بـ `gpio_request()`. لو GPIOLIB مش موجودة بتعمل `WARN_ON(1)` لأن الـ GPIO ما اتطلبش أصلًا.

- **Parameters:** `gpio` — رقم الـ GPIO
- **Return:** void
- **Key detail:** لازم تبقى ما استخدمتش الـ GPIO بعد الـ free. داخليًا بتستدعي `gpiod_free()`

---

#### `gpio_request_one()`

```c
int gpio_request_one(unsigned gpio, unsigned long flags, const char *label);
```

بتجمع الـ request والـ direction configuration في خطوة واحدة. الـ flags بتحدد الاتجاه والقيمة الابتدائية.

- **Parameters:**
  - `gpio` — رقم الـ GPIO
  - `flags` — combination من الـ `GPIOF_*` flags:
    - `GPIOF_IN` → input
    - `GPIOF_OUT_INIT_LOW` → output, initial value = 0
    - `GPIOF_OUT_INIT_HIGH` → output, initial value = 1
  - `label` — اسم الـ consumer
- **Return:** `0` على النجاح، error code على الفشل
- **Key detail:** لو الـ request نجح والـ direction config فشلت، بيعمل `gpio_free()` تلقائيًا

**Pseudocode flow:**
```
gpio_request_one(gpio, flags, label):
    ret = gpio_request(gpio, label)
    if ret: return ret

    if flags & GPIOF_IN:
        ret = gpio_direction_input(gpio)
    elif flags & GPIOF_OUT_INIT_HIGH:
        ret = gpio_direction_output(gpio, 1)
    else: /* GPIOF_OUT_INIT_LOW */
        ret = gpio_direction_output(gpio, 0)

    if ret:
        gpio_free(gpio)
    return ret
```

---

#### `devm_gpio_request_one()`

```c
int devm_gpio_request_one(struct device *dev, unsigned gpio,
                          unsigned long flags, const char *label);
```

نسخة الـ **device resource management** من `gpio_request_one()`. بتضيف الـ GPIO في الـ devres list الخاصة بالـ `dev`، وبالتالي لو الـ driver اتـ detach أو فيه error path، الـ GPIO بيتحرر تلقائيًا.

- **Parameters:**
  - `dev` — الـ device المالك للـ GPIO
  - `gpio`, `flags`, `label` — نفس `gpio_request_one()`
- **Return:** `0` أو error code
- **Key detail:** بتستخدم `devm_add_action_or_reset()` داخليًا. مش محتاج تستدعي `gpio_free()` يدويًا بعدين

---

### Category 3: Direction Configuration

#### `gpio_direction_input()`

```c
static inline int gpio_direction_input(unsigned gpio)
{
    return gpiod_direction_input(gpio_to_desc(gpio));
}
```

بتحول الـ GPIO لـ input mode وبتشيل أي output drive. الـ `gpiod_direction_input()` بتستدعي الـ `gpio_chip->direction_input()` callback في الـ hardware driver.

- **Parameters:** `gpio` — رقم الـ GPIO
- **Return:** `0` على النجاح، `-EINVAL` أو error من الـ hardware driver
- **Key detail:** الـ `gpiod_direction_input` بتراعي الـ active-low polarity. الـ legacy wrapper بتستخدم `gpio_to_desc()` اللي ممكن ترجع `NULL` لو الرقم مش موجود، وده بيودي لـ crash

---

#### `gpio_direction_output()`

```c
static inline int gpio_direction_output(unsigned gpio, int value)
{
    return gpiod_direction_output_raw(gpio_to_desc(gpio), value);
}
```

بتحول الـ GPIO لـ output وبتضبط القيمة الأولية في نفس الوقت (atomic). لاحظ إنها بتستدعي `gpiod_direction_output_raw` مش `gpiod_direction_output`، يعني بتبعت الـ `value` للـ hardware مباشرة بدون تطبيق الـ active-low transformation.

- **Parameters:**
  - `gpio` — رقم الـ GPIO
  - `value` — القيمة الأولية: 0 أو 1 (physical)
- **Return:** `0` أو error
- **Key detail:** الـ `_raw` variant ده intentional في الـ legacy API، عشان الكود القديم كان بيتعامل مع القيم الـ physical مباشرة

---

### Category 4: Value I/O — Atomic Context

#### `gpio_get_value()`

```c
static inline int gpio_get_value(unsigned gpio)
{
    return gpiod_get_raw_value(gpio_to_desc(gpio));
}
```

بتقرأ قيمة الـ GPIO من non-sleeping context زي interrupt handler أو tasklet. بتستخدم الـ `_raw` variant يعني مش بتطبق active-low inversion.

- **Parameters:** `gpio` — رقم الـ GPIO
- **Return:** `0` أو `1` (physical level)، أو قيمة سالبة لو في error (مع CONFIG_GPIOLIB الحديث)
- **Key detail:** لو الـ GPIO controller بتاعه يحتاج ينام (I2C-based expander)، الـ function دي ممنوع تتستدعي من interrupt context. استخدم `gpio_cansleep()` للتحقق

---

#### `gpio_set_value()`

```c
static inline void gpio_set_value(unsigned gpio, int value)
{
    gpiod_set_raw_value(gpio_to_desc(gpio), value);
}
```

بتكتب قيمة للـ GPIO من non-sleeping context. زي `gpio_get_value()`، بتستخدم الـ raw variant بدون active-low awareness.

- **Parameters:**
  - `gpio` — رقم الـ GPIO
  - `value` — `0` أو `1`
- **Return:** void
- **Key detail:** الـ `gpiod_set_raw_value()` داخليًا بتستخدم spinlock مش mutex، عشان كده آمنة من interrupt context لو الـ controller مش نايم

---

### Category 5: Value I/O — Sleepable Context

#### `gpio_get_value_cansleep()`

```c
static inline int gpio_get_value_cansleep(unsigned gpio)
{
    return gpiod_get_raw_value_cansleep(gpio_to_desc(gpio));
}
```

النسخة اللي ممكن تنام من قراءة الـ GPIO. للـ GPIO controllers اللي بتشتغل على I2C أو SPI، زي `pcf8574` أو `mcp23017`.

- **Parameters:** `gpio` — رقم الـ GPIO
- **Return:** `0` أو `1` (physical)
- **Key detail:** لازم تتاعت من process context بس. داخليًا بتستخدم mutex مش spinlock

---

#### `gpio_set_value_cansleep()`

```c
static inline void gpio_set_value_cansleep(unsigned gpio, int value)
{
    gpiod_set_raw_value_cansleep(gpio_to_desc(gpio), value);
}
```

نفس `gpio_set_value` بس للـ sleepable controllers.

- **Parameters:**
  - `gpio` — رقم الـ GPIO
  - `value` — `0` أو `1`
- **Return:** void
- **Caller context:** Process context فقط

---

### Category 6: IRQ Conversion

#### `gpio_to_irq()`

```c
static inline int gpio_to_irq(unsigned gpio)
{
    return gpiod_to_irq(gpio_to_desc(gpio));
}
```

بتحوّل رقم GPIO لـ Linux IRQ number ممكن تستخدمه مع `request_irq()`. الـ GPIO controller لازم يكون implement الـ `gpio_chip->to_irq()` callback، أو يكون مرتبط بـ `irq_domain`.

- **Parameters:** `gpio` — رقم الـ GPIO
- **Return:** IRQ number >= 0 على النجاح، `-EINVAL` لو الـ GPIO مش ممكن يكون interrupt source
- **Key detail:** الـ IRQ number اللي بترجعه بيتغير بين رجعات الـ boot. لازم تستدعيها في runtime مش تحرز الرقم في compile time

**مثال واقعي:**
```c
int irq = gpio_to_irq(17);
if (irq < 0) {
    dev_err(dev, "GPIO 17 has no IRQ\n");
    return irq;
}
ret = request_irq(irq, my_handler, IRQF_TRIGGER_RISING, "my-btn", priv);
```

---

### Category 7: Modern Descriptor Acquisition (Consumer API)

#### `gpiod_get()`

```c
struct gpio_desc *__must_check gpiod_get(struct device *dev,
                                         const char *con_id,
                                         enum gpiod_flags flags);
```

الـ function الرئيسية في الـ modern API لطلب GPIO. بتبحث عن GPIO مرتبط بالـ `dev` باسم الـ function المعطى `con_id`، عن طريق الـ firmware (DT/ACPI/board files). الـ `__must_check` لأن الـ return value لازم يتحقق منه دايمًا.

- **Parameters:**
  - `dev` — الـ consumer device
  - `con_id` — الاسم الوظيفي مثلاً `"reset"`, `"enable"`, `"cs"`
  - `flags` — `enum gpiod_flags`: `GPIOD_IN`, `GPIOD_OUT_LOW`, `GPIOD_OUT_HIGH`, `GPIOD_ASIS`
- **Return:** `gpio_desc*` على النجاح، `ERR_PTR(-ENOENT)` لو مش موجود، أو error آخر
- **Key detail:** في Device Tree، بتبحث عن property باسم `<con_id>-gpios`

**مثال واقعي (DT):**
```
/* في الـ .dts */
my-device {
    reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
    enable-gpios = <&gpio2 3 GPIO_ACTIVE_HIGH>;
};
```
```c
/* في الـ driver */
priv->reset_gpio = gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
if (IS_ERR(priv->reset_gpio))
    return PTR_ERR(priv->reset_gpio);
```

---

#### `gpiod_get_optional()`

```c
struct gpio_desc *__must_check gpiod_get_optional(struct device *dev,
                                                   const char *con_id,
                                                   enum gpiod_flags flags);
```

زي `gpiod_get()` بالظبط لكن لو الـ GPIO مش معرّف في الـ firmware، بترجع `NULL` بدل `ERR_PTR(-ENOENT)`. للـ GPIOs الاختيارية اللي الـ driver ممكن يشتغل بدونها.

- **Return:** `gpio_desc*`، أو `NULL` لو مش موجود، أو `ERR_PTR` لو في error حقيقي

**Pattern شائع:**
```c
priv->led = gpiod_get_optional(dev, "led", GPIOD_OUT_LOW);
if (IS_ERR(priv->led))
    return PTR_ERR(priv->led);
/* priv->led ممكن يكون NULL وده OK */
if (priv->led)
    gpiod_set_value(priv->led, 1);
```

---

#### `gpiod_get_index()`

```c
struct gpio_desc *__must_check gpiod_get_index(struct device *dev,
                                               const char *con_id,
                                               unsigned int idx,
                                               enum gpiod_flags flags);
```

لما يكون في أكتر من GPIO لنفس الـ function، زي `cs-gpios = <...>, <...>, <...>`. الـ `idx` بيحدد أنهي واحد تريده (0-indexed).

- **Parameters:**
  - `idx` — الـ index في الـ array
  - باقي الـ parameters زي `gpiod_get()`
- **Return:** نفس `gpiod_get()`

---

#### `gpiod_get_array()`

```c
struct gpio_descs *__must_check gpiod_get_array(struct device *dev,
                                                const char *con_id,
                                                enum gpiod_flags flags);
```

بتجيب كل الـ GPIOs المرتبطة بـ function معين دفعة واحدة في `struct gpio_descs`. مفيد جدًا للـ parallel GPIO operations اللي بتحتاج atomic set لعدة pins.

- **Return:** `gpio_descs*` (يحتوي على `ndescs` و `desc[]`)، أو `ERR_PTR`
- **Key detail:** الـ `gpio_descs.info` بيحتوي على معلومات الـ hardware batching اللي بتخلي `gpiod_set_array_value()` تشتغل atomically لو الـ chip بيدعم ده

---

#### `gpiod_put()` / `gpiod_put_array()`

```c
void gpiod_put(struct gpio_desc *desc);
void gpiod_put_array(struct gpio_descs *descs);
```

بيحرروا الـ descriptors المكتسبة بـ `gpiod_get()` و `gpiod_get_array()`. لازم يتاعتوا من process context.

- **Key detail:** `might_sleep()` بيتعمل داخليًا. لو استخدمت `devm_gpiod_get()` مش محتاج تستدعيهم يدويًا

---

### Category 8: Direction & Configuration (Descriptor API)

#### `gpiod_direction_input()`

```c
int gpiod_direction_input(struct gpio_desc *desc);
```

يضبط الـ GPIO كـ input. بيستدعي `chip->direction_input()` callback ويحفظ الاتجاه في الـ descriptor flags.

- **Return:** `0` أو error
- **Key detail:** بيعمل `WARN_ONCE` لو الـ GPIO مش في valid state. لو الـ pin مش configured كـ GPIO في الـ pinmux (لو في pinctrl)، ممكن يرجع error من الـ pinctrl subsystem

---

#### `gpiod_direction_output()` vs `gpiod_direction_output_raw()`

```c
int gpiod_direction_output(struct gpio_desc *desc, int value);
int gpiod_direction_output_raw(struct gpio_desc *desc, int value);
```

الفرق الجوهري بينهم:

| | `gpiod_direction_output` | `gpiod_direction_output_raw` |
|---|---|---|
| Active-low aware | نعم — بيعكس الـ value لو الـ flag set | لا — بيبعت الـ value as-is للـ hardware |
| متى تستخدمه | كود جديد دايمًا | legacy wrappers أو hardware-specific code |

**مثال:**
```c
/* GPIO مضبوط كـ active-low */
gpiod_direction_output(desc, 1);     /* يكتب 0 للـ hardware */
gpiod_direction_output_raw(desc, 1); /* يكتب 1 للـ hardware */
```

---

#### `gpiod_set_debounce()`

```c
int gpiod_set_debounce(struct gpio_desc *desc, unsigned int debounce);
```

يضبط الـ debounce time للـ GPIO inputs. يمنع الـ glitches عند الضغط على buttons مثلًا. الـ `debounce` بالـ microseconds.

- **Parameters:** `debounce` — وقت الـ debounce بالـ microseconds
- **Return:** `0` أو `-ENOTSUPP` لو الـ hardware مش بيدعم debounce
- **Key detail:** wrapper على `gpiod_set_config()` بـ `PIN_CONFIG_INPUT_DEBOUNCE` encoding

---

#### `gpiod_toggle_active_low()`

```c
void gpiod_toggle_active_low(struct gpio_desc *desc);
```

يعكس الـ `FLAG_ACTIVE_LOW` bit في الـ `gpio_desc`. مفيد لو الـ firmware معطاك GPIO بـ polarity غلطة.

- **Key detail:** بيأثر على كل الـ `gpiod_get/set_value()` calls اللاحقة. مش بيأثر على الـ `_raw` variants

---

### Category 9: Value Read / Write بالتفصيل

#### الفرق بين الـ variants

```
                    ┌─────────────────────────────────────────┐
                    │           GPIO Value API Tree           │
                    └────────────────┬────────────────────────┘
                                     │
               ┌─────────────────────┼─────────────────────┐
               │                     │                     │
          Atomic                  Sleepable              Bulk
    (no mutex, spinlock)      (may sleep, mutex)    (array ops)
               │                     │                     │
        ┌──────┴──────┐       ┌──────┴──────┐      ┌──────┴──────┐
        │   logical   │       │   logical   │      │  _array_    │
        │  (active-   │       │  (active-   │      │   value     │
        │   low aware)│       │   low aware)│      │  variants   │
        └──────┬──────┘       └──────┬──────┘      └─────────────┘
               │                     │
        ┌──────┴──────┐       ┌──────┴──────┐
        │     raw     │       │     raw     │
        │  (physical) │       │  (physical) │
        └─────────────┘       └─────────────┘
```

#### `gpiod_get_value()` / `gpiod_set_value()`

```c
int gpiod_get_value(const struct gpio_desc *desc);
int gpiod_set_value(struct gpio_desc *desc, int value);
```

القراءة/الكتابة الـ logical: بيراعوا الـ active-low flag. لو الـ GPIO مضبوط كـ active-low وبعتلك `1`، ده معناه "the signal is active" وبيكتب `0` على الـ hardware فعليًا.

- **Return لـ get:** `0` لو "inactive"، `1` لو "active" (logical)
- **Return لـ set:** `0` على النجاح، error لو في مشكلة
- **Caller context:** Atomic فقط (interrupt handlers, tasklets)

---

#### `gpiod_get_raw_value()` / `gpiod_set_raw_value()`

```c
int gpiod_get_raw_value(const struct gpio_desc *desc);
int gpiod_set_raw_value(struct gpio_desc *desc, int value);
```

القراءة/الكتابة الـ physical: بيتجاهلوا الـ active-low flag. الـ `1` دايمًا معناها voltage high على الـ pin.

- **متى تستخدمهم:** في الـ GPIO drivers نفسها، أو لما تعرف الـ polarity وتحكم فيها بنفسك

---

#### `gpiod_get_array_value()` / `gpiod_set_array_value()`

```c
int gpiod_get_array_value(unsigned int array_size,
                          struct gpio_desc **desc_array,
                          struct gpio_array *array_info,
                          unsigned long *value_bitmap);
int gpiod_set_array_value(unsigned int array_size,
                          struct gpio_desc **desc_array,
                          struct gpio_array *array_info,
                          unsigned long *value_bitmap);
```

للـ bulk GPIO operations. بتحاول تعمل الـ operation لكل الـ GPIOs atomically لو كانوا على نفس الـ `gpio_chip` وبيدعم الـ batch operations.

- **Parameters:**
  - `array_size` — عدد الـ GPIOs
  - `desc_array` — array من الـ descriptors
  - `array_info` — الـ opaque info جاية من `gpio_descs.info` (ممكن `NULL`)
  - `value_bitmap` — bitmap من القيم (bit N للـ GPIO index N)

**مثال واقعي (LED matrix):**
```c
struct gpio_descs *leds = gpiod_get_array(dev, "leds", GPIOD_OUT_LOW);
DECLARE_BITMAP(values, leds->ndescs);

/* اضبط كل الـ LEDs في خطوة واحدة */
bitmap_zero(values, leds->ndescs);
set_bit(2, values); /* فعّل LED رقم 2 بس */
gpiod_set_array_value_cansleep(leds->ndescs, leds->desc,
                                leds->info, values);
```

---

#### `gpiod_multi_set_value_cansleep()`

```c
static inline int gpiod_multi_set_value_cansleep(struct gpio_descs *descs,
                                                  unsigned long *value_bitmap)
```

wrapper مريح على `gpiod_set_array_value_cansleep()` بيتعامل مع `gpio_descs*` مباشرة بدل ما تفكك الـ struct بنفسك. لو `descs` هو `NULL` أو `ERR_PTR`، بيرجع `0` أو الـ error code بدون crash.

- **Key detail:** الـ `IS_ERR_OR_NULL(descs)` check بيخلي الكود اللي بيستخدم `gpiod_get_array_optional()` آمن بدون checks إضافية

---

### Category 10: Query Functions

#### `gpiod_to_irq()`

```c
int gpiod_to_irq(const struct gpio_desc *desc);
```

يحول الـ descriptor لـ Linux IRQ number. داخليًا بيستدعي `chip->to_irq()` callback أو بيستخدم الـ `irq_domain` المرتبط بالـ chip.

- **Return:** IRQ number >= 0، أو `-EINVAL` أو `-ENXIO`
- **Key detail:** الـ IRQ number مش ثابت، بيتغير مع كل boot في بعض الـ platforms

---

#### `gpiod_is_active_low()`

```c
int gpiod_is_active_low(const struct gpio_desc *desc);
```

يتحقق من الـ `FLAG_ACTIVE_LOW` في الـ descriptor. مفيد لو الـ code محتاج يعرف الـ polarity قبل ما يقرر هيستخدم `_raw` أو لا.

- **Return:** `1` لو active-low، `0` غير كده

---

#### `gpiod_cansleep()`

```c
int gpiod_cansleep(const struct gpio_desc *desc);
```

يتحقق من الـ `FLAG_CANSLEEP` في الـ descriptor. لازم تستدعيه قبل ما تستخدم `gpio_get/set_value()` من context حساس.

- **Return:** `1` لو الـ GPIO controller محتاج ينام، `0` لو آمن من interrupt context

**Pattern صحيح:**
```c
if (gpiod_cansleep(desc)) {
    /* لازم من process context */
    gpiod_set_value_cansleep(desc, 1);
} else {
    /* آمن من أي context */
    gpiod_set_value(desc, 1);
}
```

---

#### `gpiod_is_shared()`

```c
bool gpiod_is_shared(const struct gpio_desc *desc);
```

يتحقق من الـ `FLAG_SHARED` اللي بيدل إن الـ GPIO ممكن يتحجز من أكتر من consumer. ده كان موجود بالـ `GPIOF_SHARED` flag في الـ legacy API.

---

#### `gpiod_is_equal()`

```c
bool gpiod_is_equal(const struct gpio_desc *desc, const struct gpio_desc *other);
```

يقارن descriptorين هل بيشيروا لنفس الـ GPIO. مش بيقارن الـ pointers مباشرة لأن ممكن يكون في aliasing.

---

#### `gpiod_hwgpio()`

```c
int gpiod_hwgpio(const struct gpio_desc *desc);
```

يرجع الـ hardware-local GPIO number (offset داخل الـ `gpio_chip`). مفيد للـ debugging والـ low-level drivers.

- **Return:** non-negative offset، أو `-ENODEV`

---

#### `gpiod_count()`

```c
int gpiod_count(struct device *dev, const char *con_id);
```

يحسب كام GPIO معرّف لـ function معين في الـ firmware.

- **Return:** عدد الـ GPIOs >= 1، أو `-ENOENT` لو مفيش

---

### Category 11: Firmware / ACPI Integration

#### `fwnode_gpiod_get_index()`

```c
struct gpio_desc *fwnode_gpiod_get_index(struct fwnode_handle *fwnode,
                                          const char *con_id, int index,
                                          enum gpiod_flags flags,
                                          const char *label);
```

للـ drivers اللي بتشتغل مع الـ firmware nodes مباشرة (مش بـ `struct device`). مفيد في الـ subsystems زي USB أو PCI اللي بيجيبوا الـ GPIOs من الـ child nodes.

- **Parameters:**
  - `fwnode` — الـ firmware node (DT node أو ACPI device node)
  - `label` — اسم إضافي للـ debugging
- **Return:** `gpio_desc*` أو `ERR_PTR`

---

#### `devm_fwnode_gpiod_get_optional()`

```c
static inline struct gpio_desc *devm_fwnode_gpiod_get_optional(
    struct device *dev, struct fwnode_handle *fwnode,
    const char *con_id, enum gpiod_flags flags, const char *label)
```

Wrapper بيحول الـ `-ENOENT` من `devm_fwnode_gpiod_get_index()` لـ `NULL` بدل ما يرجعه كـ error.

- **Key detail:** بيفيد في الـ drivers اللي بتدعم boards قديمة وجديدة، والـ GPIO اختياري على بعض الـ boards

---

#### `acpi_dev_add_driver_gpios()`

```c
int acpi_dev_add_driver_gpios(struct acpi_device *adev,
                               const struct acpi_gpio_mapping *gpios);
```

في بعض الـ ACPI platforms، الـ GPIO names مش متعرفة في الـ ACPI tables بشكل صريح. الـ function دي بتضيف mapping يدوي بين أسماء GPIOs والـ resources في `_CRS`.

- **Parameters:**
  - `adev` — الـ ACPI device
  - `gpios` — array من الـ mappings (منتهية بـ `{}`)
- **Return:** `0` أو error

**مثال:**
```c
static const struct acpi_gpio_params reset_gpio = { 0, 0, false };
static const struct acpi_gpio_params irq_gpio   = { 1, 0, false };

static const struct acpi_gpio_mapping my_gpios[] = {
    { "reset-gpios", &reset_gpio, 1 },
    { "irq-gpios",   &irq_gpio,   1 },
    { }
};

/* في الـ probe */
acpi_dev_add_driver_gpios(ACPI_COMPANION(dev), my_gpios);
```

---

### Category 12: Sysfs Export

#### `gpiod_export()`

```c
int gpiod_export(struct gpio_desc *desc, bool direction_may_change);
```

يعمل export للـ GPIO في `/sys/class/gpio/gpioN/`. بيسمح لـ userspace بالتحكم في الـ GPIO. للـ debugging والـ prototyping بس، مش للـ production drivers.

- **Parameters:**
  - `direction_may_change` — لو `true`، userspace يقدر يغير الاتجاه
- **Return:** `0` أو error
- **Key detail:** يحتاج `CONFIG_GPIO_SYSFS`

---

#### `gpiod_export_link()`

```c
int gpiod_export_link(struct device *dev, const char *name,
                      struct gpio_desc *desc);
```

يعمل symlink في الـ sysfs directory بتاع الـ `dev` يشير للـ exported GPIO. بيسهّل الربط بين الـ GPIO وصاحبه من الـ device perspective.

---

#### `gpiod_unexport()`

```c
void gpiod_unexport(struct gpio_desc *desc);
```

يشيل الـ GPIO من الـ sysfs. لازم يتاعت قبل `gpiod_put()`.

---

### Category 13: Hardware Timestamping

#### `gpiod_enable_hw_timestamp_ns()` / `gpiod_disable_hw_timestamp_ns()`

```c
int gpiod_enable_hw_timestamp_ns(struct gpio_desc *desc, unsigned long flags);
int gpiod_disable_hw_timestamp_ns(struct gpio_desc *desc, unsigned long flags);
```

بيفعّلوا/بيوقفوا الـ **Hardware Timestamping Engine (HTE)** للـ GPIO. بدل ما الـ kernel يسجل الـ timestamp في الـ interrupt handler (اللي فيه latency)، الـ hardware نفسه بيسجل الـ exact timestamp بدقة nanosecond.

- **Parameters:** `flags` — حاليًا `0` (مش مستخدم)
- **Return:** `0`، أو `-ENOSYS` لو GPIOLIB أو HTE مش موجودين، أو `-EOPNOTSUPP` لو الـ chip مش بيدعم HTE
- **Key detail:** يحتاج `CONFIG_GPIOLIB && CONFIG_HTE`

**مثال واقعي (Tegra194 GPIO timestamp):**
```c
/* فعّل الـ HW timestamping على GPIO interrupt */
ret = gpiod_enable_hw_timestamp_ns(irq_gpio, 0);
if (ret)
    dev_warn(dev, "HW timestamp not supported, using SW\n");
```

---

### ملاحظات معمارية ختامية

#### Legacy vs Modern API

```
Legacy API (linux/gpio.h)          Modern API (linux/gpio/consumer.h)
─────────────────────────          ──────────────────────────────────
gpio_request(17, "reset")    →     gpiod_get(dev, "reset", GPIOD_OUT_HIGH)
gpio_direction_output(17,0)  →     (embedded in gpiod_get flags)
gpio_set_value(17, 1)        →     gpiod_set_value(desc, 1)
gpio_free(17)                →     gpiod_put(desc)
```

**ليه تتجنب الـ Legacy API:**
1. الـ global GPIO numbers مش portable بين الـ boards
2. مفيش active-low abstraction في الـ legacy get/set
3. الـ devres integration أضعف
4. GPIOLIB_LEGACY config قد يتشالها في المستقبل

**القاعدة الأساسية:** أي driver جديد لازم يستخدم `linux/gpio/consumer.h` فقط، والـ `linux/gpio.h` للـ legacy code اللي مستحيل تعيد كتابته.
## Phase 5: دليل الـ Debugging الشامل

> الـ `include/linux/gpio.h` هو الـ **legacy GPIO API** — بيعتمد على الـ global GPIO numberspace. الـ debugging بتاعته بيشمل مستويين: الـ gpiolib نفسه (اللي بيوفر الـ abstraction) وفوقيه الـ legacy wrappers اللي هي مجرد thin wrappers على الـ `gpiod_*` functions.

---

### Software Level

#### 1. Debugfs Entries

الـ **debugfs** بيوفر نظرة كاملة على حالة كل GPIO في النظام.

```bash
# تأكد إن debugfs متاح
mount | grep debugfs
# لو مش موجود:
mount -t debugfs none /sys/kernel/debug

# أهم مسار للـ GPIO
ls /sys/kernel/debug/gpio
cat /sys/kernel/debug/gpio
```

مثال على الـ output:

```
gpiochip0: GPIOs 0-31, parent: platform/fe200000.gpio, pinctrl-bcm2835:
 gpio-4   (                    |sysfs               ) in  hi IRQ ACTIVE_LOW
 gpio-17  (                    |led0                ) out lo
 gpio-27  (                    |button              ) in  hi IRQ
```

**تفسير الأعمدة:**

| العمود | المعنى |
|--------|--------|
| `gpio-N` | رقم الـ GPIO في الـ global numberspace |
| `label` | الـ label اللي اتحدد في `gpio_request(gpio, label)` |
| `in/out` | الاتجاه الحالي |
| `hi/lo` | القيمة الحالية |
| `IRQ` | مفعّل كـ interrupt source |
| `ACTIVE_LOW` | الـ polarity معكوسة |

```bash
# لو عايز تتابع تغيير قيمة GPIO معين في الوقت الحقيقي
watch -n 0.5 'grep "gpio-17" /sys/kernel/debug/gpio'
```

---

#### 2. Sysfs Entries

الـ **sysfs GPIO interface** (legacy أيضاً، بس مفيد للـ debugging):

```bash
# تصفح كل الـ gpiochips المسجلة
ls /sys/class/gpio/
# المخرج: gpiochip0  gpiochip32  export  unexport

# اعرف معلومات عن chip معين
cat /sys/class/gpio/gpiochip0/base   # أول رقم GPIO في الـ chip
cat /sys/class/gpio/gpiochip0/ngpio  # عدد الـ GPIOs
cat /sys/class/gpio/gpiochip0/label  # اسم الـ chip

# export GPIO رقم 17 للـ userspace
echo 17 > /sys/class/gpio/export

# اقرأ/اكتب بعد الـ export
cat /sys/class/gpio/gpio17/direction  # in أو out
cat /sys/class/gpio/gpio17/value      # 0 أو 1
echo out > /sys/class/gpio/gpio17/direction
echo 1   > /sys/class/gpio/gpio17/value

# تحرير الـ GPIO
echo 17 > /sys/class/gpio/unexport
```

> **ملاحظة مهمة:** لو الـ GPIO اتطلب بالفعل بـ `gpio_request()` من kernel driver، الـ export بيفشل بـ `EBUSY`.

---

#### 3. Ftrace — Tracepoints والـ Events

الـ **ftrace** بيساعد تتبع كل استدعاء لدوال الـ GPIO في الـ kernel.

```bash
# اعرف الـ GPIO tracepoints المتاحة
grep -r gpio /sys/kernel/tracing/available_events

# أهم الـ events:
# gpio:gpio_direction
# gpio:gpio_value

# تفعيل تتبع gpio_direction و gpio_value
cd /sys/kernel/tracing
echo 1 > tracing_on
echo 'gpio:gpio_direction gpio:gpio_value' > set_event

# أو تفعيل كل gpio events دفعة واحدة
echo 'gpio:*' > set_event

# تفعيل function tracing لدوال محددة
echo 'gpio_request gpio_free gpio_direction_input gpio_direction_output' > set_ftrace_filter
echo function > current_tracer

# اقرأ الـ trace
cat trace

# أو تابع في الوقت الحقيقي
cat trace_pipe
```

مثال على الـ ftrace output:

```
  kworker/0:1-45  [000]  123.456: gpio:gpio_direction: gpio=17 direction=out
  kworker/0:1-45  [000]  123.457: gpio:gpio_value: gpio=17 get=0 value=1
```

---

#### 4. Printk والـ Dynamic Debug

**تفعيل الـ dynamic debug** للـ gpiolib subsystem:

```bash
# تفعيل كل debug messages في gpiolib
echo 'file drivers/gpio/* +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file include/linux/gpio* +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug لملف محدد
echo 'file gpiolib.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# p = print, f = function name, l = line number, m = module, t = thread id

# تفعيل debug message معين برقم السطر
echo 'file gpiolib.c line 1234 +p' > /sys/kernel/debug/dynamic_debug/control

# عرض الـ dynamic debug controls الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep gpio

# إيقاف
echo 'file drivers/gpio/* -p' > /sys/kernel/debug/dynamic_debug/control
```

لرفع مستوى الـ **printk loglevel** مؤقتاً:

```bash
# اعرض المستوى الحالي
cat /proc/sys/kernel/printk   # مثال: 7 4 1 7

# اجعل كل الرسائل تظهر على الـ console
echo 8 > /proc/sys/kernel/printk

# تابع الـ kernel log
dmesg -w | grep -i gpio
journalctl -k -f | grep -i gpio
```

---

#### 5. Kernel Config Options للـ Debugging

```bash
# اعرض الـ config الحالي
zcat /proc/config.gz | grep -E 'GPIO|GPIOLIB' | sort
# أو
grep -E 'CONFIG_GPIO|CONFIG_GPIOLIB' /boot/config-$(uname -r)
```

| الـ Config Option | الوظيفة |
|-------------------|---------|
| `CONFIG_GPIOLIB` | تفعيل الـ gpiolib نفسه — أساسي |
| `CONFIG_GPIOLIB_LEGACY` | تفعيل الـ legacy API (`gpio_request`, `gpio_free`, إلخ) |
| `CONFIG_DEBUG_GPIO` | تفعيل الـ debug checks داخل الـ gpiolib (فحص الـ direction قبل القراءة/الكتابة) |
| `CONFIG_GPIO_SYSFS` | تفعيل الـ sysfs interface للـ GPIO |
| `CONFIG_GPIO_CDEV` | تفعيل الـ character device interface (الـ modern API) |
| `CONFIG_GPIOLIB_IRQCHIP` | دعم الـ GPIO-based IRQ chips |
| `CONFIG_PROVE_LOCKING` | اكتشاف deadlocks في الـ spinlocks/mutexes |
| `CONFIG_DEBUG_SPINLOCK` | فحص مشاكل الـ spinlock |
| `CONFIG_KASAN` | اكتشاف memory corruption |
| `CONFIG_LOCKDEP` | تتبع الـ lock ordering |

```bash
# تفعيل CONFIG_DEBUG_GPIO في kernel جديد
make menuconfig
# Device Drivers → GPIO Support → Debug GPIO calls
```

> **`CONFIG_DEBUG_GPIO`** مهم جداً — بيضيف checks زي: هل الـ GPIO اتطلب قبل الاستخدام؟ هل الاتجاه صح قبل القراءة/الكتابة؟

---

#### 6. Devlink وأدوات الـ Subsystem

```bash
# gpiodetect: اعرض كل الـ GPIO chips في النظام
gpiodetect
# المخرج: gpiochip0 [pinctrl-bcm2835] (54 lines)

# gpioinfo: عرض تفاصيل كل line في chip
gpioinfo gpiochip0
# أو كل الـ chips
gpioinfo

# gpioget: اقرأ قيمة GPIO
gpioget gpiochip0 17

# gpioset: اكتب قيمة GPIO
gpioset gpiochip0 17=1

# gpiomon: راقب تغييرات GPIO
gpiomon --num-events=10 gpiochip0 17

# تثبيت أدوات libgpiod
apt install gpiod       # Debian/Ubuntu
dnf install libgpiod    # Fedora
```

> هذه الأدوات بتتعامل مع الـ **character device API** (الـ `/dev/gpiochipN`) وليس الـ legacy sysfs، لكن بتعكس نفس الـ state الداخلي.

---

#### 7. جدول Error Messages الشائعة

| رسالة الـ Kernel Log | السبب | الحل |
|----------------------|-------|------|
| `gpio-17: requested but not yet used` | استدعاء `gpio_request()` بدون استخدام فعلي | تأكد إن الـ driver بيستخدم الـ GPIO |
| `gpio-N: can't use as IRQ` | الـ GPIO controller مش بيدعم IRQ على الـ line دي | استخدم GPIO تاني أو راجع الـ DT |
| `gpio-N: invalid direction "foo"` | قيمة غلط في sysfs `/direction` | استخدم `in` أو `out` بس |
| `request failed` + `-EBUSY` | الـ GPIO متطلوب من driver تاني | `cat /sys/kernel/debug/gpio` واعرف مين مسكه |
| `request failed` + `-EPROBE_DEFER` | الـ GPIO controller لسه ما اتسجلش | طبيعي في الـ boot، الـ driver هيحاول تاني |
| `gpio_free: WARN_ON(1)` | استدعاء `gpio_free()` بدون `gpio_request()` قبلها | ترتيب خاطئ في الـ driver |
| `not an output` | استدعاء `gpio_set_value()` على GPIO مش output | `gpio_direction_output()` الأول |
| `not an input` | استدعاء `gpio_get_value()` على GPIO مش input | `gpio_direction_input()` الأول |
| `-ENOSYS` from `gpio_request()` | الـ `CONFIG_GPIOLIB` مش مفعّل | فعّل الـ config option |
| `WARN_ON(1)` in `gpio_free` stub | الـ GPIOLIB مش مفعّل و`gpio_free()` اتستدعت | مشكلة في الـ build config |

---

#### 8. Strategic Points لـ dump_stack() و WARN_ON()

نقاط استراتيجية لإضافة checks في الـ kernel code:

```c
int gpio_request(unsigned gpio, const char *label)
{
    struct gpio_desc *desc;

    desc = gpio_to_desc(gpio);
    /* هنا: لو desc == NULL معناه الـ GPIO number غلط */
    if (WARN_ON(!desc))
        return -EINVAL;

    return gpiod_request(desc, label);
}

static inline void gpio_set_value(unsigned gpio, int value)
{
    struct gpio_desc *desc = gpio_to_desc(gpio);

    /* هنا: تأكد إن الـ GPIO request اتعمل */
    WARN_ON(!desc || !test_bit(FLAG_REQUESTED, &desc->flags));

    /* هنا: تأكد إن الاتجاه output */
    WARN_ON(!test_bit(FLAG_IS_OUT, &desc->flags));

    gpiod_set_raw_value(desc, value);
}
```

لإضافة `dump_stack()` في نقاط حرجة:

```c
/* في drivers/gpio/gpiolib.c — عند فشل الـ request */
int gpiod_request(struct gpio_desc *desc, const char *label)
{
    if (test_and_set_bit(FLAG_REQUESTED, &desc->flags)) {
        pr_err("GPIO %d already requested by '%s', new request from:\n",
               desc_to_gpio(desc), desc->label);
        dump_stack();  /* اعرف مين بيطلب GPIO مسكوك */
        return -EBUSY;
    }
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق من مطابقة الـ Hardware State للـ Kernel State

```bash
# اقرأ قيمة GPIO من الـ kernel
cat /sys/kernel/debug/gpio | grep "gpio-17"
# المخرج: gpio-17 (led0) out hi

# قارن بالقراءة المباشرة من الـ register
# (راجع datasheet للـ SoC الخاص بك لعنوان الـ GPIO data register)
```

الـ flow المثالي للتحقق:

```
Kernel State (debugfs)  →  sysfs value  →  /dev/mem register  →  Oscilloscope/Multimeter
      hi                →      1         →    bit set = 1       →    3.3V measured
```

---

#### 2. Register Dump Techniques

```bash
# تثبيت devmem2
apt install devmem2

# مثال: Raspberry Pi — BCM2835 GPIO registers
# Base address: 0x3F200000 (Pi 2/3) أو 0xFE200000 (Pi 4)
GPIO_BASE=0xFE200000

# GPFSEL1: Function Select Register 1 (GPIO 10-19)
devmem2 $((GPIO_BASE + 0x04)) w
# GPLEV0: Pin Level Register (GPIO 0-31)
devmem2 $((GPIO_BASE + 0x34)) w

# تفسير GPLEV0:
# Bit 17 = 1 → GPIO 17 = HIGH
# Bit 17 = 0 → GPIO 17 = LOW

# قراءة بـ /dev/mem مباشرة (بدون devmem2)
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 4096, offset=0xFE200000)
    mm.seek(0x34)  # GPLEV0
    val = struct.unpack('<I', mm.read(4))[0]
    print(f'GPLEV0 = 0x{val:08X}')
    print(f'GPIO17 = {(val >> 17) & 1}')
    mm.close()
"

# io utility (من package iotools)
io -4 -r 0xFE200034   # قراءة 32-bit من GPLEV0
```

> **تحذير:** الـ `/dev/mem` access ممكن يكون معطّل بسبب `CONFIG_STRICT_DEVMEM`. في الحالة دي استخدم الـ debugfs بدل كده.

---

#### 3. Logic Analyzer وـ Oscilloscope Tips

**عند استخدام Logic Analyzer:**

```
1. ربط الـ probe على الـ GPIO pin مباشرة (مش على الـ resistor)
2. ضبط الـ voltage threshold:
   - 1.8V GPIO → threshold عند 0.9V
   - 3.3V GPIO → threshold عند 1.65V
   - 5V GPIO  → threshold عند 2.5V

3. Trigger conditions مفيدة:
   - Rising edge → لاكتشاف متى بيتفعّل الـ GPIO
   - Falling edge → لاكتشاف متى بيتعطّل
   - Pulse width < X → للكشف عن glitches

4. لتتبع I2C/SPI على GPIOs:
   - استخدم protocol decoder في Sigrok/PulseView
   - Channel 0 = SCL/CLK, Channel 1 = SDA/MOSI
```

**أوامر Sigrok (software logic analyzer):**

```bash
# تسجيل GPIO events بـ sigrok-cli
sigrok-cli --driver fx2lafw --config samplerate=1m \
           --channels D0=GPIO17,D1=GPIO27 \
           --samples 1000000 \
           --output-file gpio_capture.sr

# تحليل الـ capture
pulseview gpio_capture.sr
```

**Oscilloscope tips:**

- لقياس الـ rise/fall time: ضبط timebase على 10-100ns
- لاكتشاف voltage drop: راقب الـ voltage أثناء تحميل عالي على الـ GPIO
- الـ GPIO output current عادةً 4-16mA max — فوق كده بيحصل voltage sag

---

#### 4. Common Hardware Issues وـ Kernel Log Patterns

| المشكلة الهاردوير | الـ Pattern في الـ Kernel Log | التشخيص والحل |
|-------------------|------------------------------|---------------|
| GPIO pin شورت على GND | `gpio-N: value read back unexpected` | قس الـ resistance بين الـ pin وـ GND (يكون > 10KΩ) |
| Pull-up/Pull-down مش صح | الـ GPIO بيقلقل بدون سبب (IRQ floods) | `dmesg \| grep "irq N: nobody cared"` — ضيف pull-up |
| الـ GPIO Controller مش بيتعرف | `gpio-N: chip not found` | تأكد من الـ power domain للـ GPIO block |
| Voltage mismatch (3.3V GPIO على 5V signal) | لا log — بس الـ GPIO بيحترق | افحص الـ voltage levels قبل الربط |
| GPIO pin مستخدم كـ alternate function | `gpio_request: GPIO N is already in use` | راجع الـ pinmux في الـ DT |
| Clock للـ GPIO controller مش مفعّل | `gpio gpiochip0: Failed to get clock` | فعّل الـ clock في الـ DT أو الـ CCF config |
| الـ GPIO لا يستجيب | Pin stuck → فحص بـ `devmem2` يرجع نفس القيمة دايماً | ممكن الـ pin محروق أو في reset state |

---

#### 5. Device Tree Debugging

**التحقق إن الـ DT مطابق للـ Hardware:**

```bash
# اعرض الـ compiled DT (DTB) المستخدم حالياً
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 10 "gpio"

# أو
cat /sys/firmware/devicetree/base/soc/gpio@fe200000/compatible
cat /sys/firmware/devicetree/base/soc/gpio@fe200000/reg

# اعرض كل الـ GPIO controllers المسجلة
ls /sys/bus/platform/drivers/gpio-*
ls /sys/class/gpio/

# تحقق من الـ pinctrl state
cat /sys/kernel/debug/pinctrl/pinctrl-handles
cat /sys/kernel/debug/pinctrl/fe200000.gpio/gpio-ranges

# اعرض الـ GPIO mappings من الـ DT
cat /sys/kernel/debug/gpio | head -50
```

**مثال DT لـ GPIO صح:**

```dts
/* في الـ Device Tree Source */
leds {
    compatible = "gpio-leds";
    status = "okay";

    led0: led-0 {
        label = "green-led";
        gpios = <&gpio 17 GPIO_ACTIVE_HIGH>;
        /* gpio: phandle للـ GPIO controller */
        /* 17: رقم الـ GPIO */
        /* GPIO_ACTIVE_HIGH: الـ polarity */
    };
};
```

```bash
# تحقق إن الـ phandle صح
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null \
    | grep -B5 -A15 "gpio-leds"

# قارن الـ reg address في الـ DT مع الـ datasheet
# مثلاً: reg = <0x0 0xfe200000 0x0 0xb4>;
# → base address 0xfe200000، size 0xb4 bytes
```

**أخطاء DT شائعة:**

```
OF: /leds/led-0: could not get #gpio-cells for /soc/gpio@fe200000
→ الـ #gpio-cells property ناقصة في الـ GPIO controller node

gpio-N: (null): not an output
→ الـ DT بيحدد GPIO_ACTIVE_HIGH لكن الـ pin مش configured كـ output

Failed to request GPIO: -517 (EPROBE_DEFER)
→ الـ GPIO controller node لسه ما اتبروب — تأكد من الـ boot order في الـ DT
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

```bash
#!/bin/bash
# gpio-debug.sh — سكريبت شامل لـ debugging الـ GPIO

GPIO_NUM=${1:-17}  # رقم الـ GPIO (default: 17)

echo "=== GPIO $GPIO_NUM Debug Report ==="

echo "--- Kernel GPIO State ---"
grep "gpio-$GPIO_NUM" /sys/kernel/debug/gpio 2>/dev/null || \
    cat /sys/kernel/debug/gpio | grep "gpio-$GPIO_NUM"

echo "--- GPIO Chip Info ---"
for chip in /sys/class/gpio/gpiochip*; do
    base=$(cat $chip/base)
    ngpio=$(cat $chip/ngpio)
    label=$(cat $chip/label)
    end=$((base + ngpio - 1))
    if [ $GPIO_NUM -ge $base ] && [ $GPIO_NUM -le $end ]; then
        echo "GPIO $GPIO_NUM is in: $label (GPIOs $base-$end)"
        echo "Chip path: $chip"
    fi
done

echo "--- Export & Read ---"
echo $GPIO_NUM > /sys/class/gpio/export 2>/dev/null && {
    echo "Direction: $(cat /sys/class/gpio/gpio$GPIO_NUM/direction)"
    echo "Value:     $(cat /sys/class/gpio/gpio$GPIO_NUM/value)"
    echo "Edge:      $(cat /sys/class/gpio/gpio$GPIO_NUM/edge 2>/dev/null)"
    echo $GPIO_NUM > /sys/class/gpio/unexport
} || echo "GPIO $GPIO_NUM already exported or in use by kernel driver"

echo "--- IRQ Mapping ---"
cat /proc/interrupts | grep -i "gpio-$GPIO_NUM" 2>/dev/null || \
    echo "No IRQ mapped for GPIO $GPIO_NUM"

echo "--- Recent Kernel Messages ---"
dmesg | grep -i "gpio" | grep "$GPIO_NUM" | tail -20

echo "--- gpioinfo (libgpiod) ---"
gpioinfo 2>/dev/null | grep -A2 "line $((GPIO_NUM % 32))" | head -5
```

```bash
# تفعيل كامل لـ GPIO tracing
GPIO_TRACE_START() {
    cd /sys/kernel/tracing
    echo 0 > tracing_on
    echo > trace
    echo 'gpio:*' > set_event
    echo function > current_tracer
    echo 'gpio_*' > set_ftrace_filter
    echo 1 > tracing_on
    echo "GPIO tracing started — read from: /sys/kernel/tracing/trace_pipe"
}

GPIO_TRACE_STOP() {
    echo 0 > /sys/kernel/tracing/tracing_on
    cp /sys/kernel/tracing/trace /tmp/gpio-trace-$(date +%Y%m%d-%H%M%S).txt
    echo "Trace saved."
}
```

```bash
# تشخيص سريع للـ GPIO بدون export
# (بيستخدم gpioget من libgpiod)
for i in $(seq 0 53); do
    val=$(gpioget gpiochip0 $i 2>/dev/null)
    echo "GPIO $i = $val"
done

# مراقبة GPIO لمدة 10 ثواني
timeout 10 gpiomon --format="%e %o %S" gpiochip0 17 27

# قراءة الـ pinctrl debug info
cat /sys/kernel/debug/pinctrl/pinctrl-maps | grep -A5 "gpio-17"
```

```bash
# التحقق من الـ kernel config المهم للـ GPIO
check_gpio_config() {
    CFG=/boot/config-$(uname -r)
    [ -f /proc/config.gz ] && CFG=<(zcat /proc/config.gz)

    for opt in GPIOLIB GPIOLIB_LEGACY DEBUG_GPIO GPIO_SYSFS GPIO_CDEV GPIOLIB_IRQCHIP; do
        val=$(grep "CONFIG_$opt=" $CFG 2>/dev/null | cut -d= -f2)
        printf "CONFIG_%-25s = %s\n" "$opt" "${val:-NOT SET}"
    done
}
check_gpio_config
```

**تفسير مثال على output الـ `/sys/kernel/debug/gpio`:**

```
gpiochip0: GPIOs 0-53, parent: platform/fe200000.gpio, pinctrl-bcm2835:
 gpio-2   (                    |                    ) in  hi              ← GPIO 2: input، قيمته HIGH، مش متطلوب
 gpio-4   (                    |?                   ) in  hi ACTIVE_LOW   ← ? = لا label، ACTIVE_LOW معناه HIGH logic = LOW physical
 gpio-17  (                    |green-led           ) out lo              ← GPIO 17: output، قيمته LOW، اسمه "green-led"
 gpio-27  (                    |button              ) in  hi IRQ ACTIVE_LOW ← GPIO 27: input، IRQ مفعّل، ACTIVE_LOW
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — درايفر قديم بيستخدم `gpio_request` وبيتعطل على kernel جديد

#### العنوان
**ميجريشن درايفر legacy GPIO على gateway صناعي بـ RK3562**

#### السياق
شركة بتعمل industrial gateway بيشغّل Modbus RTU على UART مع RS-485 direction control عن طريق GPIO. الـ gateway بيشتغل على RK3562 مع kernel 6.6. الدرايفر الأصلي اتكتب سنة 2018 لـ kernel 4.14 وبيستخدم الـ legacy API من `<linux/gpio.h>` مباشرةً.

#### المشكلة
عند compile الـ driver على kernel 6.6:
```
warning: #warning This header should not be included
error: implicit declaration of function 'gpio_request'
```
الـ build بيفشل لأن `CONFIG_GPIOLIB_LEGACY` مش enabled في kernel config الجديد، فـ`gpio_request` مش موجودة خالص.

#### التحليل
الكود الأصلي:
```c
#include <linux/gpio.h>   /* legacy include — يجب تجنبه */

static int rs485_probe(struct platform_device *pdev)
{
    int gpio_num = of_get_named_gpio(pdev->dev.of_node, "de-gpios", 0);
    if (!gpio_is_valid(gpio_num))   /* gpio_is_valid: only valid under CONFIG_GPIOLIB_LEGACY */
        return -EINVAL;

    gpio_request(gpio_num, "rs485-de");         /* legacy: maps to global GPIO number */
    gpio_direction_output(gpio_num, 0);          /* internally calls gpio_to_desc() then gpiod_direction_output_raw() */
}
```

الـ flow داخل `gpio.h` لما `CONFIG_GPIOLIB` موجود بس `CONFIG_GPIOLIB_LEGACY` مش موجود:
```
gpio_direction_output()  →  NOT DEFINED (whole block under #ifdef CONFIG_GPIOLIB_LEGACY)
gpio_request()           →  NOT DEFINED
gpio_is_valid()          →  NOT DEFINED
```

الـ `gpio_direction_output` اللي كانت موجودة كانت بتعمل:
```c
static inline int gpio_direction_output(unsigned gpio, int value)
{
    /* يحوّل الرقم العالمي لـ descriptor */
    return gpiod_direction_output_raw(gpio_to_desc(gpio), value);
}
```
لكن ده كله محصور جوه `#ifdef CONFIG_GPIOLIB_LEGACY` اللي مش موجود.

#### الحل
**الخطوة 1:** استبدال الـ include والـ API بالكامل:

```c
/* قبل */
#include <linux/gpio.h>
#include <linux/of_gpio.h>

/* بعد */
#include <linux/gpio/consumer.h>
```

**الخطوة 2:** تعديل الكود:

```c
static struct gpio_desc *rs485_de_gpio;

static int rs485_probe(struct platform_device *pdev)
{
    /* gpiod_get يستخدم DT property "de-gpios" automatically */
    rs485_de_gpio = devm_gpiod_get(&pdev->dev, "de", GPIOD_OUT_LOW);
    if (IS_ERR(rs485_de_gpio))
        return PTR_ERR(rs485_de_gpio);

    return 0;
}

static void rs485_set_direction(int transmit)
{
    /* لا حاجة لـ gpio_direction_output — الـ direction اتحدد في probe */
    gpiod_set_value(rs485_de_gpio, transmit);
}
```

**الخطوة 3:** Device Tree صح:
```dts
&uart2 {
    status = "okay";
    de-gpios = <&gpio3 RK_PA5 GPIO_ACTIVE_HIGH>;
};
```

#### الدرس المستفاد
الـ `<linux/gpio.h>` وكل اللي جوا من `gpio_request`, `gpio_direction_output`, `gpio_is_valid` ده **legacy API** محصور جوه `#ifdef CONFIG_GPIOLIB_LEGACY`. أي كود جديد لازم يستخدم `<linux/gpio/consumer.h>` مع `devm_gpiod_get()` مباشرة. الـ `devm_` prefix بيضمن إن الـ GPIO هيتحرر أوتوماتيك عند driver detach.

---

### السيناريو الثاني: Android TV Box بـ Allwinner H616 — active-low reset GPIO بيتصرف عكس المتوقع

#### العنوان
**reset GPIO للـ HDMI transmitter على TV box بـ H616 بيعمل عكس المتوقع**

#### السياق
TV box بيشتغل على Allwinner H616 مع Android 12. الـ HDMI transmitter (IT66121) محتاج reset نشط على low. الدرايفر القديم اتبورت من مشروع تاني وبيستخدم `gpio_set_value` من `gpio.h`.

#### المشكلة
الـ HDMI مش بيشتغل. الـ IT66121 بيظهر في I2C scan بس بيرجع قراءات غلط. الـ oscilloscope بيوري إن الـ reset line ماشيه HIGH طول الوقت حتى لما الكود بيكتب 0.

#### التحليل
الكود المشكلة:
```c
#include <linux/gpio.h>

/* reset: active-low، الـ 0 = reset active، الـ 1 = normal */
gpio_set_value(reset_gpio, 0);  /* المفروض يعمل reset */
msleep(10);
gpio_set_value(reset_gpio, 1);  /* release reset */
```

الـ `gpio_set_value` في `gpio.h` بتستدعي:
```c
static inline void gpio_set_value(unsigned gpio, int value)
{
    gpiod_set_raw_value(gpio_to_desc(gpio), value);  /* RAW — لا يراعي active-low polarity */
}
```

الكلمة المفتاحية هنا: **`gpiod_set_raw_value`** — ده بيكتب القيمة مباشرة على الـ hardware بدون ما يراعي `GPIO_ACTIVE_LOW` flag الموجود في الـ DT.

الـ DT كان:
```dts
reset-gpios = <&pio 2 5 GPIO_ACTIVE_LOW>;
```

يعني الـ GPIO فيزيائياً مش مقلوب في الـ hardware، والـ GPIO_ACTIVE_LOW معناه إن الـ software لازم يعكس القيمة. لكن `gpiod_set_raw_value` بتتجاهل ده.

#### الحل
**الحل الصح:** استخدام `gpiod_set_value` (مش raw) عبر الـ new API:

```c
#include <linux/gpio/consumer.h>

static struct gpio_desc *reset_desc;

static int it66121_probe(struct i2c_client *client)
{
    /* devm_gpiod_get تقرأ GPIO_ACTIVE_LOW من DT وتخزنه في الـ descriptor */
    reset_desc = devm_gpiod_get(&client->dev, "reset", GPIOD_OUT_HIGH);
    /* GPIOD_OUT_HIGH = logical high = physical low (بسبب ACTIVE_LOW) = reset active */
    if (IS_ERR(reset_desc))
        return PTR_ERR(reset_desc);

    return 0;
}

static void it66121_reset(void)
{
    gpiod_set_value(reset_desc, 1);   /* logical 1 = assert reset (physical 0 بسبب ACTIVE_LOW) */
    msleep(10);
    gpiod_set_value(reset_desc, 0);   /* logical 0 = release reset (physical 1) */
}
```

**تأكيد الحل عبر sysfs:**
```bash
# قبل الحل — القراءة بتوري الـ physical value
cat /sys/kernel/debug/gpio | grep it66121-reset

# بعد الحل — gpiod_set_value بيراعي polarity
gpioget gpiochip2 5   # يرجع 1 لما reset مش active
```

#### الدرس المستفاد
**الفرق الجوهري** بين `gpio_set_value` (legacy، يستدعي `gpiod_set_raw_value`) و `gpiod_set_value` (new API، يراعي `GPIO_ACTIVE_LOW`). استخدام `gpio.h` مع `gpio_set_value` على GPIO معرّف بـ `GPIO_ACTIVE_LOW` في DT ينتج سلوك معكوس تماماً. القاعدة: لو في DT فيه `GPIO_ACTIVE_LOW`، لازم تستخدم الـ descriptor API مع `gpiod_set_value`.

---

### السيناريو الثالث: IoT Sensor Board بـ STM32MP1 — `gpio_to_irq` بيرجع error غير متوقع

#### العنوان
**interrupt من SPI accelerometer على STM32MP1 مش بيشتغل بسبب غلط في GPIO-to-IRQ mapping**

#### السياق
board بيشتغل كـ IoT sensor node، بيحمل MEMS accelerometer (LIS2DH12) على SPI. الـ data-ready interrupt موصّل على GPIO. الكود بيستخدم `gpio_to_irq()` من الـ legacy `gpio.h`.

#### المشكلة
```
lis2dh12: failed to get irq: -22
```
الـ driver مش بيسجل الـ interrupt. الـ GPIO سليم، موصّل صح، بس `gpio_to_irq` بيرجع `-EINVAL`.

#### التحليل
الكود المشكلة:
```c
#include <linux/gpio.h>

static int lis2dh12_probe(struct spi_device *spi)
{
    int gpio_num = of_get_named_gpio(spi->dev.of_node, "int1-gpios", 0);
    int irq;

    gpio_request(gpio_num, "lis2dh12-int1");
    gpio_direction_input(gpio_num);

    irq = gpio_to_irq(gpio_num);  /* المشكلة هنا */
    if (irq < 0)
        return irq;   /* بيرجع -EINVAL */
}
```

الـ `gpio_to_irq` في `gpio.h`:
```c
static inline int gpio_to_irq(unsigned gpio)
{
    return gpiod_to_irq(gpio_to_desc(gpio));  /* بيحوّل الرقم لـ descriptor الأول */
}
```

الـ `gpio_to_desc(gpio_num)` بيرجع `NULL` لأن الـ GPIO numberspace على STM32MP1 مش متطابق مع الـ number اللي رجعه `of_get_named_gpio`. في بعض الـ kernels، الـ legacy numberspace بيبدأ من رقم مختلف حسب ترتيب register الـ GPIO banks.

تتبع المشكلة:
```
of_get_named_gpio()  →  يرجع 85 (global GPIO number)
gpio_to_desc(85)     →  يبحث في legacy GPIO map → NULL (لم يُسجَّل بعد بالترتيب الصح)
gpiod_to_irq(NULL)   →  returns -EINVAL
```

#### الحل
التحويل الكامل لـ descriptor API:

```c
#include <linux/gpio/consumer.h>
#include <linux/interrupt.h>

static struct gpio_desc *int1_desc;
static int irq_num;

static int lis2dh12_probe(struct spi_device *spi)
{
    /* gpiod_get تجيب الـ descriptor مباشرة من DT بدون global numberspace */
    int1_desc = devm_gpiod_get(&spi->dev, "int1", GPIOD_IN);
    if (IS_ERR(int1_desc)) {
        dev_err(&spi->dev, "failed to get int1 gpio: %ld\n", PTR_ERR(int1_desc));
        return PTR_ERR(int1_desc);
    }

    /* gpiod_to_irq تشتغل مع الـ descriptor مباشرة */
    irq_num = gpiod_to_irq(int1_desc);
    if (irq_num < 0) {
        dev_err(&spi->dev, "failed to get irq: %d\n", irq_num);
        return irq_num;
    }

    return devm_request_irq(&spi->dev, irq_num, lis2dh12_isr,
                            IRQF_TRIGGER_RISING, "lis2dh12", spi);
}
```

**DT:**
```dts
&spi2 {
    lis2dh12@0 {
        compatible = "st,lis2dh12";
        reg = <0>;
        int1-gpios = <&gpiof 5 GPIO_ACTIVE_HIGH>;
    };
};
```

**Debug:**
```bash
# تحقق إن الـ GPIO descriptor اتسجل صح
cat /sys/kernel/debug/gpio | grep lis2dh12

# تحقق من الـ IRQ mapping
cat /proc/interrupts | grep lis2dh12
```

#### الدرس المستفاد
الـ `gpio_to_irq` في `gpio.h` بتعتمد على `gpio_to_desc()` اللي بتحوّل رقم عالمي لـ descriptor. لو الـ GPIO numberspace مش متطابق (وده شائع جداً على STM32MP1 وRK3xxx)، النتيجة `NULL` و`-EINVAL`. الـ `gpiod_to_irq` من `consumer.h` بتشتغل مع الـ descriptor مباشرة بدون اعتماد على الـ legacy numberspace.

---

### السيناريو الرابع: Automotive ECU بـ i.MX8 — `devm_gpio_request_one` بيفشل بعد kernel upgrade

#### العنوان
**CAN bus transceiver enable GPIO على ECU بـ i.MX8QM بيفشل مع kernel 6.1**

#### السياق
ECU خاص بـ ADAS (Advanced Driver Assistance System) يشتغل على i.MX8QM. الـ CAN transceiver (TJA1043) محتاج GPIO لتفعيل وضع التشغيل العادي بدل الـ standby. الكود اتكتب على kernel 5.4 وبيستخدم `devm_gpio_request_one`.

#### المشكلة
بعد upgrade لـ kernel 6.1:
```
tja1043: devm_gpio_request_one failed: -524
```
الـ -524 = `-EPROBE_DEFER`، يعني الـ GPIO controller لسه متحملش.

#### التحليل
الكود القديم:
```c
#include <linux/gpio.h>

static int tja1043_probe(struct spi_device *spi)
{
    int en_gpio = of_get_named_gpio(spi->dev.of_node, "enable-gpios", 0);

    /* devm_gpio_request_one: موجودة فقط لما CONFIG_GPIOLIB_LEGACY enabled */
    ret = devm_gpio_request_one(&spi->dev, en_gpio,
                                GPIOF_OUT_INIT_HIGH, "tja1043-en");
    if (ret)
        return ret;  /* بيرجع -EPROBE_DEFER */
}
```

الـ `devm_gpio_request_one` في `gpio.h` معرّفة فقط تحت `#ifdef CONFIG_GPIOLIB_LEGACY`:
```c
int devm_gpio_request_one(struct device *dev, unsigned gpio,
                          unsigned long flags, const char *label);
```

في kernel 6.1، الـ `CONFIG_GPIOLIB_LEGACY` اتشال من imx8 defconfig. الـ function موجودة لكن الـ `gpio_request` داخليا بتعتمد على legacy infrastructure. المشكلة إن `of_get_named_gpio` بيرجع رقم تاني من الرقم المتوقع، والـ `gpio_to_desc` بتفشل تحوّله في وقت الـ probe المبكر.

**الـ EPROBE_DEFER flow:**
```
devm_gpio_request_one(gpio=45)
  → gpio_request(45)
    → gpiochip_find_base() → GPIO bank لسه في probe queue
    → -EPROBE_DEFER
```

المشكلة إن الـ legacy path مش بيتعامل صح مع الـ deferred probe.

#### الحل
**المنهج الأمثل:** استخدام `devm_gpiod_get` اللي بيتعامل مع `EPROBE_DEFER` تلقائياً:

```c
#include <linux/gpio/consumer.h>

static int tja1043_probe(struct spi_device *spi)
{
    struct gpio_desc *en_gpio;

    /*
     * devm_gpiod_get: تدعم EPROBE_DEFER بشكل صح
     * GPIOD_OUT_HIGH يعوّض عن GPIOF_OUT_INIT_HIGH
     */
    en_gpio = devm_gpiod_get(&spi->dev, "enable", GPIOD_OUT_HIGH);
    if (IS_ERR(en_gpio)) {
        ret = PTR_ERR(en_gpio);
        if (ret != -EPROBE_DEFER)
            dev_err(&spi->dev, "failed to get enable gpio: %d\n", ret);
        return ret;  /* -EPROBE_DEFER يتعامل معه kernel framework تلقائياً */
    }

    /* GPIO enabled، CAN transceiver في وضع normal */
    dev_info(&spi->dev, "TJA1043 enabled\n");
    return 0;
}
```

**الجدول المقابل بين الـ flags القديمة والجديدة:**

| Legacy (gpio.h) | New (consumer.h) |
|---|---|
| `GPIOF_IN` | `GPIOD_IN` |
| `GPIOF_OUT_INIT_LOW` | `GPIOD_OUT_LOW` |
| `GPIOF_OUT_INIT_HIGH` | `GPIOD_OUT_HIGH` |

#### الدرس المستفاد
الـ `devm_gpio_request_one` من `gpio.h` معرّفة جوه `CONFIG_GPIOLIB_LEGACY` فقط، وبتشتغل على global GPIO numbers. الـ `devm_gpiod_get` من `consumer.h` أفضل لأنها بتتعامل مع `EPROBE_DEFER` بشكل صح، ومش بتحتاج legacy config. في الـ automotive context، الـ EPROBE_DEFER مهم جداً لأن الـ probe order ممكن يتغير.

---

### السيناريو الخامس: Custom Board Bring-up بـ AM62x — `gpio_is_valid` بترجع `true` لأرقام غلط

#### العنوان
**bring-up board بـ AM62x — `gpio_is_valid` بترجع صح لـ GPIO مش موجود فيزيائياً**

#### السياق
مهندس بيعمل bring-up لـ custom board بـ Texas Instruments AM62x. الـ board عندها SPI flash موصّل بـ CS software-controlled عن طريق GPIO. الكود القديم بيستخدم `gpio_is_valid` كـ sanity check قبل ما يبدأ يستخدم الـ GPIO.

#### المشكلة
الكود بيمشي بدون errors، بس الـ SPI flash مش بيستجيب. الهاردوير verification بيوري إن الـ CS line ساكنة. الـ `gpio_is_valid` بترجع `true` حتى لو الرقم كان خطأ.

#### التحليل
الكود المشكلة:
```c
#include <linux/gpio.h>

#define SPI_CS_GPIO  150  /* المهندس كتب الرقم غلط — المفروض 105 */

static int spiflash_probe(struct platform_device *pdev)
{
    if (!gpio_is_valid(SPI_CS_GPIO)) {
        dev_err(&pdev->dev, "invalid GPIO\n");
        return -EINVAL;
    }
    /* gpio_is_valid رجعت true، فالكود اكمل */
    gpio_request(SPI_CS_GPIO, "spi-cs");
    gpio_direction_output(SPI_CS_GPIO, 1);
}
```

الـ `gpio_is_valid` في `gpio.h`:
```c
static inline bool gpio_is_valid(int number)
{
    /* only non-negative numbers are valid */
    return number >= 0;
}
```

المشكلة واضحة: **`gpio_is_valid` بتتحقق بس إن الرقم >= 0**. مش بتتحقق إن الرقم ده موجود فعلاً في أي GPIO controller. الرقم 150 قانوني رياضياً، فبترجع `true`، لكن فيزيائياً الـ AM62x ما عندوش GPIO رقم 150 في الـ bank ده.

بعدين `gpio_request(150)` ممكن تنجح بردو لو كان فيه GPIO مسجّل بالرقم ده في GPIO expander تاني موصّل على I2C — وده ما حدش منتبهله.

**المشكلة الحقيقية في الـ flow:**
```
gpio_is_valid(150) → true  (>= 0)
gpio_request(150)  → success (رقم موجود في GPIO expander مش في الـ SoC)
gpio_direction_output(150, 1) → success (بيتحكم في GPIO خطأ!)
SPI transaction  → CS ساكنة (الـ GPIO الصح 105 لسه input)
```

#### الحل
**الحل الصح:** التخلص من `gpio_is_valid` تماماً واستخدام `devm_gpiod_get` اللي بتتحقق من الـ DT مباشرة:

```c
#include <linux/gpio/consumer.h>

static struct gpio_desc *cs_gpio;

static int spiflash_probe(struct platform_device *pdev)
{
    /*
     * devm_gpiod_get: تجيب الـ GPIO من DT property "cs-gpios"
     * لو الـ property مش موجودة أو الـ GPIO مش موجود → IS_ERR
     * لا اعتماد على أرقام hard-coded
     */
    cs_gpio = devm_gpiod_get(&pdev->dev, "cs", GPIOD_OUT_HIGH);
    if (IS_ERR(cs_gpio)) {
        dev_err(&pdev->dev, "cannot get cs gpio: %ld\n", PTR_ERR(cs_gpio));
        return PTR_ERR(cs_gpio);
    }

    return 0;
}
```

**الـ DT الصح:**
```dts
&spi0 {
    spiflash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        cs-gpios = <&main_gpio0 105 GPIO_ACTIVE_LOW>;  /* الرقم الصح 105 */
    };
};
```

**Debug للتأكد:**
```bash
# شوف كل الـ GPIOs المستخدمة وأسمائها
cat /sys/kernel/debug/gpio

# تحقق إن الـ GPIO 105 هو المستخدم فعلاً
gpioinfo gpiochip0 | grep -i "105\|cs\|spi"

# اختبار manual
gpioset gpiochip0 105=0   # assert CS
gpioset gpiochip0 105=1   # deassert CS
```

#### الدرس المستفاد
**`gpio_is_valid` من `gpio.h` ما بتتحققش إن الـ GPIO موجود فيزيائياً** — بتتحقق بس إن الرقم non-negative. ده المصيدة الأكبر في الـ legacy API. الـ descriptor API مع DT بتضمن إن الـ GPIO اللي بتشتغل عليه هو بالظبط اللي في الـ device tree، مش رقم hard-coded ممكن يشير لـ GPIO خطأ في expander أو bank تاني.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

**الـ LWN.net** مصدر أساسي لفهم تطور الـ GPIO subsystem في الـ Linux kernel.

| المقالة | الوصف |
|---------|-------|
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة عن الـ GPIO mechanism منذ kernel 2.6.21 — نقطة بداية ممتازة |
| [GPIO in the kernel: future directions](https://lwn.net/Articles/533632/) | مناقشة التوجهات المستقبلية وسبب الانتقال من الـ integer-based interface |
| [gpio: introduce descriptor-based interface](https://lwn.net/Articles/531848/) | أول patch series قدّم الـ `gpiod_*` API الحديث |
| [gpiolib: introduce descriptor-based GPIO interface](https://lwn.net/Articles/528226/) | النسخة الأولى من الـ descriptor-based interface بقلم Alexandre Courbot |
| [New descriptor-based GPIO interface](https://lwn.net/Articles/565662/) | تغطية اعتماد الـ API الجديد رسميًا في الـ kernel |
| [Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/) | توثيق الـ gpiod interface الجديد وإزالة الـ legacy docs |

---

### التوثيق الرسمي في الـ kernel

**الـ Documentation/** في الـ kernel tree هو المرجع الرسمي الأول:

```bash
# الملفات الرئيسية في kernel source tree
Documentation/driver-api/gpio/intro.rst        # مدخل للـ GPIO subsystem
Documentation/driver-api/gpio/consumer.rst     # كيف يستخدم الـ driver الـ GPIO
Documentation/driver-api/gpio/driver.rst       # كيف تكتب GPIO chip driver
Documentation/driver-api/gpio/board.rst        # GPIO mappings (DT, ACPI, platform)
Documentation/driver-api/gpio/legacy.rst       # الـ legacy integer interface (deprecated)
Documentation/admin-guide/gpio/              # GPIO من منظور الـ userspace
```

روابط HTML الرسمية:

- [General Purpose Input/Output (GPIO) — kernel docs](https://docs.kernel.org/driver-api/gpio/index.html)
- [GPIO Descriptor Consumer Interface](https://docs.kernel.org/driver-api/gpio/consumer.html)
- [GPIO Driver Interface](https://static.lwn.net/kerneldoc/driver-api/gpio/driver.html)
- [GPIO Mappings (board.rst)](https://docs.kernel.org/driver-api/gpio/board.html)
- [Legacy GPIO Interfaces](https://www.kernel.org/doc/html/v5.0/driver-api/gpio/legacy.html)

---

### الملفات الرئيسية في الـ source tree

الملفات التي يجب قراءتها لفهم الـ GPIO subsystem كاملًا:

```
include/linux/gpio.h           ← الملف الحالي (legacy — لا تستخدمه)
include/linux/gpio/consumer.h  ← الـ API الحديث للـ consumer drivers
include/linux/gpio/driver.h    ← الـ API لكتابة GPIO chip drivers
drivers/gpio/gpiolib.c         ← الـ core implementation
drivers/gpio/gpiolib-legacy.c  ← wrapper للـ legacy functions
drivers/gpio/gpiolib-of.c      ← Device Tree integration
drivers/gpio/gpiolib-acpi.c    ← ACPI integration
```

---

### Kernel Commits المهمة

**الـ commits** التي غيّرت مسار الـ GPIO subsystem:

| الحدث | التفاصيل |
|-------|----------|
| GPIO initial introduction | Kernel 2.6.21 — أول إضافة للـ GPIO subsystem |
| Descriptor-based API | قدّمها Alexandre Courbot — انتقال من `int gpio` لـ `struct gpio_desc *` |
| `GPIOLIB_LEGACY` Kconfig | عزل الـ legacy code خلف config option لتسهيل الإزالة لاحقًا |
| Legacy removal patches | [patch series لإزالة gpio_export](https://patchwork.kernel.org/project/linux-media/patch/20230207142952.51844-6-andriy.shevchenko@linux.intel.com/) |

للبحث في تاريخ الملف مباشرة:

```bash
git log --oneline -- include/linux/gpio.h
git log --oneline -- drivers/gpio/gpiolib-legacy.c
git log --oneline -- include/linux/gpio/consumer.h
```

---

### نقاشات Mailing List

- [LKML.org — linux-gpio archive](https://lkml.org/) — ابحث عن `gpio-legacy` أو `gpiod migration`
- [vger.kernel.org — linux-gpio list](https://subspace.kernel.org/vger.kernel.org.html) — القائمة الرسمية للـ GPIO subsystem
- [PATCH v2: gpiolib: remove legacy gpio_export](https://patchew.org/linux/20211109100207.2474024-1-arnd@kernel.org/20211109100207.2474024-7-arnd@kernel.org/) — مثال على جهود الإزالة

---

### Kernelnewbies.org

- [Linux 2.6.21 — GPIO introduction](https://kernelnewbies.org/Linux_2_6_21) — أول ظهور للـ GPIO في الـ kernel
- [Linux 2.6.25 — GPIO enhancements](https://kernelnewbies.org/Linux_2_6_25) — تحسينات مبكرة
- [LinuxChanges — تتبع كل kernel release](https://kernelnewbies.org/LinuxChanges) — ابحث عن "gpio" في كل إصدار
- [Linux 6.13 — GPIO changes](https://kernelnewbies.org/Linux_6.13) — أحدث التغييرات
- [Mailing list: devm_gpiod_get usage](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html) — مثال عملي على استخدام الـ descriptor API
- [Mailing list: GPIO from device tree](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html) — كيف تربط GPIO بالـ device tree

---

### eLinux.org

- [New GPIO Interface for User Space (ELCE 2017 slides)](https://elinux.org/images/7/74/Elce2017_new_GPIO_interface.pdf) — شرح الـ character device interface الجديد
- [Introduction to pin muxing and GPIO control under Linux (ELC 2021)](https://elinux.org/images/a/a7/ELC-2021_Introduction_to_pin_muxing_and_GPIO_control_under_Linux.pdf) — شرح شامل للـ pinmux وعلاقته بالـ GPIO
- [EBC GPIO via mmap](https://elinux.org/EBC_GPIO_via_mmap) — مثال عملي على الـ memory-mapped GPIO access
- [EBC gpio Polling and Interrupts](https://elinux.org/EBC_gpio_Polling_and_Interrupts) — sysfs interface والـ interrupts

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 14** — The Linux Device Model
- مجاني online: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- ملاحظة: الكتاب قديم (2.6.x) — لا يغطي الـ descriptor-based API، لكنه يشرح الأساسيات جيدًا

#### Linux Kernel Development — Robert Love (3rd Ed.)
- **الفصل 17** — Devices and Modules
- يشرح الـ device model الذي يقوم عليه الـ GPIO subsystem

#### Embedded Linux Primer — Christopher Hallinan (2nd Ed.)
- **الفصل 15** — Embedded Linux Application Development
- يغطي استخدام الـ GPIO من الـ userspace وكيف يتفاعل مع الـ kernel drivers

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds
- أحدث كتاب يغطي الـ `libgpiod` والـ character device interface الحديث
- يُنصح به على LDD3 لمن يعمل على أنظمة حديثة

---

### Search Terms للبحث عن مزيد من المعلومات

```
# للبحث في kernel source
git log --all --grep="gpiolib" --oneline
git log --all --grep="gpio_to_desc" --oneline
git log --all --grep="GPIOLIB_LEGACY" --oneline

# للبحث في LKML
"linux-gpio" site:lore.kernel.org
"gpio descriptor migration" site:lore.kernel.org
"GPIOLIB_LEGACY removal" site:lore.kernel.org

# Google/DuckDuckGo
"linux kernel gpio descriptor vs legacy"
"gpiod_get vs gpio_request"
"linux gpio numberspace deprecated"
"linux gpio char device /dev/gpiochipN"
"libgpiod userspace gpio"
```

---

### ملخص سريع للمراجع حسب الأولوية

| الأولوية | المرجع | لماذا؟ |
|----------|--------|--------|
| 1 | [kernel docs — GPIO index](https://docs.kernel.org/driver-api/gpio/index.html) | المصدر الرسمي والمحدّث دائمًا |
| 2 | [LWN: GPIO introduction](https://lwn.net/Articles/532714/) | يشرح "لماذا" تغيّر الـ API |
| 3 | [LWN: New descriptor interface](https://lwn.net/Articles/565662/) | يشرح الـ API الحديث بعمق |
| 4 | [legacy.rst](https://www.kernel.org/doc/html/v5.0/driver-api/gpio/legacy.html) | لفهم الكود القديم قبل migration |
| 5 | [ELCE 2017 slides](https://elinux.org/images/7/74/Elce2017_new_GPIO_interface.pdf) | شرح عملي للـ char device interface |
## Phase 8: Writing simple module

### الفكرة

هنعمل module بيستخدم **kprobe** عشان يعترض الـ `gpio_request` — الـ function الـ legacy اللي بتطلب GPIO pin باسم معين. ده مفيد جداً لو عايز تعرف مين من الـ drivers بيطلب GPIO numbers وبأي labels في الـ runtime.

الـ `gpio_request` exported وموجودة في الـ gpiolib core، وبتاخد رقم الـ GPIO والـ label. الـ kprobe هيتقفل قبل ما الـ function تتنفذ ونطبع الـ arguments.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * gpio_request_probe.c
 *
 * Intercept legacy gpio_request() calls using kprobe
 * and log which GPIO number + label is being requested.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */

/* Pre-handler: called just before gpio_request() executes.
 * On x86_64:
 *   regs->di = first arg  (unsigned gpio)
 *   regs->si = second arg (const char *label)
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    unsigned int gpio = (unsigned int)regs->di;
    const char   *label = (const char *)regs->si;

    /* label could be NULL, guard with a ternary */
    pr_info("gpio_request intercepted: gpio=%u label=\"%s\"\n",
            gpio, label ? label : "(null)");

    return 0; /* 0 = let the real function continue */
}

/* Optional post-handler: called after gpio_request() returns.
 * regs->ax holds the return value on x86_64.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    int ret = (int)regs->ax;

    pr_info("gpio_request returned: %d (%s)\n",
            ret, ret == 0 ? "success" : "failed");
}

/* kprobe descriptor — symbol_name ties it to the target function */
static struct kprobe kp = {
    .symbol_name = "gpio_request",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

static int __init gpio_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("gpio_request kprobe planted at %p\n", kp.addr);
    return 0;
}

static void __exit gpio_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("gpio_request kprobe removed\n");
}

module_init(gpio_probe_init);
module_exit(gpio_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Doc Project");
MODULE_DESCRIPTION("kprobe on legacy gpio_request() to trace GPIO allocation");
```

---

### Makefile

```makefile
obj-m += gpio_request_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|--------|-------|
| `<linux/module.h>` | الـ macros الأساسية لأي module: `module_init`, `MODULE_LICENSE`, إلخ |
| `<linux/kernel.h>` | الـ `pr_info` / `pr_err` للـ logging |
| `<linux/kprobes.h>` | الـ `struct kprobe` وكل الـ API بتاعتها |

---

#### الـ `handler_pre`

ده الـ callback اللي بيتشغل **قبل** ما `gpio_request` تتنفذ فعلاً. بنقرأ الـ arguments مباشرة من الـ `pt_regs` — على الـ x86_64 الـ `di` هو أول argument والـ `si` هو التاني — وبنطبعهم. لو الـ `label` جت `NULL` بنحطها `"(null)"` عشان منحصلش kernel panic من الـ `%s`.

---

#### الـ `handler_post`

بيتشغل **بعد** ما الـ function ترجع. الـ return value موجود في `regs->ax` على الـ x86_64، ونطبعه مع تفسير بشري. ده مفيد نعرف منه لو الـ GPIO كان متطلوب قبل كده أو مفيش driver بيديره.

---

#### الـ `struct kprobe`

الحقل `symbol_name = "gpio_request"` بيخلي الـ kernel يحل الـ address من الـ kallsyms تلقائياً. ملناش داعي نحط الـ address يدوي.

---

#### الـ `module_init` و `module_exit`

- **`register_kprobe`**: بتزرع الـ probe في الـ kernel — بتحل الـ symbol، بتحفظ الـ instruction الأصلية، وبتحطه breakpoint.
- **`unregister_kprobe`**: لازم تتعمل في الـ `exit` عشان تشيل الـ breakpoint وتمسح الـ handler قبل ما الـ module يتشال من الذاكرة — لو متعملتيش هيحصل crash لو الـ function اتطلبت بعد الـ unload.

---

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميله
sudo insmod gpio_request_probe.ko

# شوف الـ output في الـ kernel log
sudo dmesg | grep "gpio_request"

# لو عندك LED على GPIO معروف، حمّل أي driver بيستخدم gpio_request
# مثلاً على Raspberry Pi:
echo 17 > /sys/class/gpio/export   # بيعمل gpio_request تحت الغطا

# إزالة الـ module
sudo rmmod gpio_request_probe

# تأكيد
sudo dmesg | tail -5
```

**مثال output متوقع:**

```
[  123.456] gpio_request intercepted: gpio=17 label="sysfs"
[  123.457] gpio_request returned: 0 (success)
[  124.001] gpio_request intercepted: gpio=4 label="leds-gpio"
[  124.002] gpio_request returned: 0 (success)
```

---

### ملاحظة مهمة

الـ `gpio_request` هو الـ legacy API المبني على الـ global GPIO numberspace. على الـ kernels الحديثة (5.x+) معظم الـ drivers بقت بتستخدم `gpiod_get` بدله. لو عايز تـ trace الـ API الجديدة، استبدل `symbol_name` بـ `"gpiod_get"` وعدّل الـ `pre_handler` يقرأ الـ `con_id` (الـ `si`) والـ `flags` (الـ `dx`).
