## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ GPIO؟

**GPIO** اختصار لـ General-Purpose Input/Output — يعني طرف كهربائي على الـ chip ممكن تستخدمه كـ input (تقرأ منه) أو كـ output (تكتب عليه). ده أبسط شكل ممكن للتواصل بين الـ processor والعالم الخارجي.

تخيل الـ processor عنده أصابع كهربائية — كل إصبع ممكن يحس بحاجة (input) أو يلمس حاجة (output). الـ GPIO هو الاصبع ده.

---

### القصة اللي الملف بيحلها

تخيل عندك **Raspberry Pi** أو أي **embedded board**. على اللوحة دي:
- LED محتاج يتشعل لما الـ system يبدأ
- زرار ضغط محتاج الـ kernel يسمع له
- sensor بيدي إشارة رقمية high/low

كل ده بيحصل عبر **GPIO pins**. المشكلة إن في آلاف الـ chips المختلفة من شركات مختلفة — كل واحدة عندها GPIO controller خاص بيها. كل driver كتبه بطريقته.

لما البورد يبدأ، الـ kernel محتاج يعرف:
- فين الـ GPIO controller ده في الـ hardware؟
- أنهو pin ده؟
- هل الإشارة active high ولا active low؟
- هل الـ pin connected لـ LED ولا motor ولا sensor؟

**السؤال الكبير:** إزاي الـ kernel يعرف كل ده من غير ما يحتاج source code مختلف لكل board؟

**الجواب:** الـ **Device Tree** — ملف وصفي بيوصف الـ hardware بشكل منفصل عن الـ driver code. والملف اللي بنشرحه ده هو القواعد اللي بتقول "إزاي تكتب وصف الـ GPIO في الـ Device Tree صح".

---

### الـ Subsystem اللي ينتمي له

الملف ده جزء من **GPIO subsystem** في Linux kernel، وتحديداً من **Device Tree bindings** بتاعته. ده مش kernel code — ده **documentation + specification** بتقول للناس إزاي يكتبوا الـ `.dts` files صح عشان الـ GPIO drivers تفهمها.

---

### الـ Device Tree — تشبيه بسيط

تخيل إن الـ Linux kernel زي موظف جديد في شركة. الموظف ده شاطر ويعرف يشتغل مع أي hardware، بس محتاج **ورقة تعريفية** بكل device في الشركة.

الـ **Device Tree** هي الورقة دي — بتقول للـ kernel: "عندك GPIO controller على العنوان الفلاني، عنده 32 pin، والـ LED الحمرا على pin رقم 5 وبتشتعل لما الإشارة low."

---

### إيه اللي بيحدده الملف ده؟

الملف `gpio.txt` بيحدد **اللغة المشتركة** بين:
1. **الـ hardware vendor** اللي بيكتب الـ Device Tree للـ board بتاعته
2. **الـ kernel driver** اللي بيقرأ الـ Device Tree ويشغّل الـ GPIO

#### 1. كيفية وصف استخدام GPIO لـ device معين (Consumer side)

لما device عايز يستخدم GPIO، بيقول كده في الـ DTS:

```dts
/* جهاز بيستخدم GPIO لـ enable وبيانات */
mydevice {
    /* اسم الـ property = [name]-gpios */
    enable-gpios = <&gpio1 18 GPIO_ACTIVE_HIGH>;
    data-gpios   = <&gpio1 12 0>,
                   <&gpio1 13 0>;
};
```

- **`&gpio1`** — phandle بيشير لـ GPIO controller
- **`18`** — رقم الـ pin جوه الـ controller
- **`GPIO_ACTIVE_HIGH`** — flag بيقول الإشارة active عند high voltage

#### 2. كيفية تعريف GPIO Controller نفسه (Provider side)

الـ controller نفسه بيتعرف كده:

```dts
gpio1: gpio-controller@1400 {
    compatible = "vendor,chip-gpio";  /* إيه الـ driver المناسب */
    reg = <0x1400 0x18>;              /* عنوان الـ register في الـ memory */
    gpio-controller;                  /* علامة إن ده controller */
    #gpio-cells = <2>;                /* كل GPIO بيحتاج 2 cells: رقم + flags */
    ngpios = <18>;                    /* عدد الـ GPIOs الفعلية */
    gpio-line-names = "MMC-CD", "LED R", ...; /* أسماء للـ lines */
};
```

#### 3. الـ GPIO Flags — بـ bitfield

```
Bit 0: polarity    → 0=active-high, 1=active-low
Bit 1: single-ended → 0=push-pull, 1=single-ended
Bit 2: drain type  → 0=open-source, 1=open-drain
Bit 3: persistence → 0=persistent during sleep, 1=transitory
Bit 4: pull-up     → 0=disabled, 1=enabled
Bit 5: pull-down   → 0=disabled, 1=enabled
```

#### 4. الـ GPIO Hogging — الاستحواذ التلقائي

**GPIO hog** معناها إن الـ GPIO controller لما يبدأ، يقوم وحده يحجز ويضبط بعض الـ GPIOs تلقائياً من غير ما أي driver تاني يطلب ده:

```dts
gpio-controller@1400 {
    gpio-controller;
    #gpio-cells = <2>;

    /* هذا الـ child node هو hog تلقائي */
    reset-line-hog {
        gpio-hog;
        gpios = <6 0>;   /* pin 6, active-high */
        output-low;      /* اضبطه كـ output بقيمة low فوراً */
        line-name = "system-reset";
    };
};
```

الـ use case: لما الـ board يبدأ، لازم تمسك reset line معينة في وضع معين قبل أي driver تاني يشتغل — الـ hog بيعمل ده تلقائياً.

#### 5. الربط مع الـ Pin Controller (gpio-ranges)

الـ GPIO pins الفيزيائية في الغالب مشتركة مع وظائف تانية (UART، SPI، I2C...). الـ **pin controller** هو اللي بيقرر الـ pin ده GPIO ولا بيشتغل كـ UART TX مثلاً.

الربط بيتعمل عبر `gpio-ranges`:

```dts
gpio-controller@1460 {
    gpio-controller;
    #gpio-cells = <2>;
    /* GPIO 0..9 على الـ controller ده = pins 20..29 على pinctrl1 */
    gpio-ranges = <&pinctrl1 0 20 10>;
};
```

```
GPIO Controller          Pin Controller (pinctrl1)
┌─────────────┐          ┌─────────────────────────┐
│ GPIO line 0 │◄────────►│ physical pin 20          │
│ GPIO line 1 │◄────────►│ physical pin 21          │
│ ...         │          │ ...                      │
│ GPIO line 9 │◄────────►│ physical pin 29          │
└─────────────┘          └─────────────────────────┘
```

---

### ليه الملف ده مهم؟

| المشكلة | الحل اللي الملف بيقدمه |
|---------|------------------------|
| كل vendor بيكتب DTS بطريقته | قواعد موحدة لكل الـ GPIO bindings |
| اسم الـ property مش واضح | اشتراط `[name]-gpios` كـ naming convention |
| اتجاه الإشارة مش معروف | flags cells معيارية |
| GPIOs محتاجة تتضبط عند البوت | آلية الـ GPIO hogging |
| الـ GPIO pins مرتبطة بـ pin mux | gpio-ranges property |

---

### ملفات مرتبطة لازم تعرفها

| الملف | الدور |
|-------|-------|
| `include/dt-bindings/gpio/gpio.h` | الـ macros الـ standard زي `GPIO_ACTIVE_HIGH`, `GPIO_OPEN_DRAIN` |
| `drivers/gpio/gpiolib.c` | الـ core library اللي بتقرأ الـ DT وتحول الـ bindings لـ GPIO descriptors |
| `drivers/gpio/gpiolib-of.c` | الكود الـ specific لـ parsing الـ Device Tree GPIO properties |
| `Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt` | وصف الـ pin controllers اللي بيتعاملوا مع الـ GPIO ranges |
| `Documentation/gpio/gpio.rst` | الـ GPIO subsystem documentation للـ driver writers |
| `Documentation/devicetree/bindings/gpio/*.yaml` | الـ bindings الـ specific لكل GPIO controller hardware |

---

### الـ GPIO Subsystem — ملفات الـ Core

```
الـ Device Tree Bindings (documentation/spec):
├── Documentation/devicetree/bindings/gpio/gpio.txt        ← الملف ده
├── Documentation/devicetree/bindings/gpio/*.yaml          ← bindings لكل controller
└── include/dt-bindings/gpio/gpio.h                        ← constants مشتركة

الـ Core Framework (kernel code):
├── drivers/gpio/gpiolib.c                  ← core: registration, consumer API
├── drivers/gpio/gpiolib-of.c               ← Device Tree parsing
├── drivers/gpio/gpiolib-sysfs.c            ← sysfs interface
├── drivers/gpio/gpiolib-cdev.c             ← character device interface
├── include/linux/gpio/driver.h             ← API for GPIO controller drivers
└── include/linux/gpio/consumer.h           ← API for GPIO consumers

الـ Hardware Drivers (examples):
├── drivers/gpio/gpio-pl061.c               ← ARM PrimeCell PL061
├── drivers/gpio/gpio-ath79.c               ← Atheros AR71xx/9xxx
├── drivers/gpio/gpio-nomadik.c             ← ST Nomadik
└── drivers/gpio/gpio-*.c                   ← مئات الـ drivers التانية
```

---

### الصورة الكاملة في نقطة واحدة

الملف `gpio.txt` هو **العقد** بين البورد الـ hardware ومنه والـ Linux kernel. بيقول: "لو عايز الـ kernel يفهم الـ GPIO على بورد بتاعك، اكتب الـ Device Tree بالطريقة دي." الـ kernel من جهته عنده `gpiolib-of.c` بيفهم الـ syntax ده ويحوله لـ GPIO descriptors جاهزة للاستخدام في أي driver.
## Phase 2: شرح الـ GPIO Device Tree Bindings Framework

---

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في أي SoC زي STM32 أو i.MX أو Raspberry Pi، في عندك عشرات أو مئات الـ GPIO lines. كل driver محتاج يعرف إزاي يوصل للـ GPIO الخاص بيه. المشكلة في الـ historical approach كانت إن الـ driver بيتضمن hardcoded numbers:

```c
/* الطريقة القديمة — worst practice */
gpio_request(42, "enable");        /* رقم 42 ده مين؟ بييجي منين؟ */
gpio_direction_output(42, 1);
```

ده بيخلق مشاكل حقيقية:
1. **Portability** — نفس الـ driver لو اتنقل على board تاني، الـ GPIO 42 بيبقى حاجة تانية خالص.
2. **Maintainability** — محدش عارف GPIO 42 ده بيتحكم في إيه من غير ما يرجع للـ schematic.
3. **Multiple instances** — لو في اتنين من نفس الـ IP block في الـ SoC، هتعمل إيه؟
4. **Polarity chaos** — اللي بيحدد active-high vs active-low هو الـ hardware design والـ PCB, مش الـ driver.

**الـ GPIO DT Bindings Framework** اتعمل عشان يفصل بين تلاتة حاجات:
- **Hardware description** — إيه الـ GPIO controllers الموجودة وفين؟
- **Board-level wiring** — إيه اللي متوصل بإيه على الـ PCB؟
- **Driver logic** — الـ consumer driver بيطلب GPIO بالاسم الوظيفي مش بالرقم.

---

### الحل — الـ approach بتاع الـ kernel

الـ kernel بيحل المشكلة دي بـ **three-layer separation**:

```
Layer 3: Consumer Driver
    "أنا محتاج GPIO اسمها enable-gpios"
         ↓
Layer 2: GPIO Core (gpiolib)
    بتترجم الـ DT phandle + specifier لـ gpio_desc
         ↓
Layer 1: GPIO Controller Driver (Provider)
    بيتكلم مع الـ hardware registers فعلاً
```

الـ Device Tree هو الـ glue بين الـ layers دي — بيحدد:
- إيه الـ GPIO controllers الموجودة (`gpio-controller` property)
- عندها كام line (`ngpio`, `#gpio-cells`)
- الـ consumer بيحتاج إيه بالظبط (`enable-gpios = <&gpio1 12 GPIO_ACTIVE_LOW>`)

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        USER SPACE                                    │
│          /dev/gpiochipN  ←  Character Device (libgpiod)             │
│          /sys/class/gpio ←  Legacy sysfs (deprecated)              │
└────────────────────────────┬────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│                     KERNEL SPACE                                     │
│                                                                      │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐    │
│  │   regulator  │   │   mmc/sdhci  │   │   spi / i2c / misc   │    │
│  │   driver     │   │   driver     │   │   consumer drivers   │    │
│  └──────┬───────┘   └──────┬───────┘   └──────────┬───────────┘    │
│         │                  │                       │                │
│         │   gpiod_get("enable-gpios")              │                │
│         └──────────────────┴───────────────────────┘                │
│                             │                                        │
│              ┌──────────────▼──────────────┐                        │
│              │         GPIO CORE           │                        │
│              │         (gpiolib)           │                        │
│              │                             │                        │
│              │  • DT lookup & parsing      │                        │
│              │  • gpio_desc management     │                        │
│              │  • polarity translation     │                        │
│              │  • request/free tracking    │                        │
│              │  • IRQ domain mapping       │                        │
│              └──────┬──────────────┬───────┘                        │
│                     │              │                                 │
│           ┌─────────▼──┐    ┌──────▼──────────┐                    │
│           │ gpio_chip  │    │   gpio_chip      │                    │
│           │  (Bank A)  │    │   (Bank B)       │                    │
│           │ SoC GPIO   │    │  I2C expander    │                    │
│           │  driver    │    │  (PCF8574)       │                    │
│           └─────────┬──┘    └──────┬───────────┘                    │
│                     │              │                                 │
└─────────────────────┼──────────────┼─────────────────────────────────┘
                      │              │
              ┌───────▼──┐    ┌──────▼──────┐
              │ Physical │    │  I2C Bus    │
              │  GPIO    │    │  Hardware   │
              │  Regs    │    │             │
              └──────────┘    └─────────────┘
```

---

### الـ Real-World Analogy — نظام الكهرباء في العمارة

تخيل عمارة سكنية كبيرة. الـ analogy دي متكاملة جداً:

| مفهوم في الـ analogy | المقابل في الـ kernel |
|----------------------|----------------------|
| **لوحة الكهرباء** في البدروم | `gpio_chip` — GPIO controller |
| **القاطع رقم 12** في اللوحة | GPIO line offset داخل الـ controller |
| **سلك التوصيل** من اللوحة للشقة | الـ trace على الـ PCB |
| **مفتاح الإضاءة** في الشقة | الـ consumer device |
| **دفتر العمارة** اللي بيوصف كل قاطع بيغذي إيه | الـ Device Tree |
| **الفني الكهربائي** اللي بيعرف يتكلم مع اللوحة | GPIO controller driver |
| **مكتب الإدارة** اللي بيوزع الطلبات | GPIO Core (gpiolib) |
| **العميل** اللي بيقول "أنا محتاج نور الصالة" | consumer driver |

الجزء المهم: الـ **consumer مش محتاج يعرف رقم القاطع** — هو بيقول للإدارة "أنا محتاج نور الصالة" وهي بترجع له الوصول الصح. لو اللوحة اتغيرت أو القاطع اتنقل، الـ دفتر (Device Tree) بيتحدث والـ consumer مش بيتأثر خالص.

**التفاصيل المعمقة للـ analogy:**

- **`#gpio-cells = <2>`** = كل قاطع في الدفتر محتاج معلومتين: رقم القاطع + ملاحظة (مثلاً: "اتجاه تركيب المفتاح مقلوب")
- **`GPIO_ACTIVE_LOW`** = المفتاح معكوس — لما بتحطه ON الضوء بيطفي (زي switches اللي التوصيل بيكون normally-closed)
- **`gpio-ranges`** = الدفتر بيقول "القاطعات 20 لـ 29 في اللوحة A بتتحكم في الغرف 0 لـ 9 في العمارة" — ده mapping بين pin controller و GPIO controller
- **GPIO hog** = قاطع محجوز ومضبوط من أول يوم — الفني الكهربائي بيشغله تلقائياً من غير ما يستنى حد يطلب

---

### الـ Core Abstraction — إيه الفكرة المحورية؟

الـ GPIO DT Bindings قائمة على فكرة **phandle + specifier**:

```
enable-gpios = <&gpio1  12  GPIO_ACTIVE_LOW>;
               ╔══════╗╔══╗╔═════════════╗
               ║phandle║  ║  consumer    ║
               ║ = ref ║  ║  flags       ║
               ║ to    ║  ╚═════════════╝
               ║gpio1  ║  ╔══╗
               ║ node  ║  ║12║ = local offset
               ╚══════╝  ╚══╝
                 cell[0]  cell[1]
```

الـ **phandle** ده pointer في الـ DT — بيشير لـ node معينة. الـ **specifier** هو التفاصيل اللي بيفسرها الـ controller نفسه. ده مهم جداً: الـ kernel core مش بيفسر الـ specifier — بيديه للـ `of_xlate` callback الخاص بكل controller.

```c
/* في struct gpio_chip — الـ controller بيوفر الترجمة */
int (*of_xlate)(struct gpio_chip *gc,
                const struct of_phandle_args *gpiospec,
                u32 *flags);
/* gpiospec->args[0] = 12 (local offset) */
/* gpiospec->args[1] = GPIO_ACTIVE_LOW (flags) */
/* الـ function بترجع local offset وتحط الـ flags في *flags */
```

---

### الـ GPIO Specifier Flags بالتفصيل

الملف `include/dt-bindings/gpio/gpio.h` بيعرف الـ standard flags:

```c
/* Bit 0 — Polarity */
#define GPIO_ACTIVE_HIGH   0   /* logic 1 = voltage high */
#define GPIO_ACTIVE_LOW    1   /* logic 1 = voltage low  */

/* Bit 1 — Drive mode */
#define GPIO_PUSH_PULL     0   /* standard totem-pole output */
#define GPIO_SINGLE_ENDED  2   /* open drain or open source */

/* Bit 2 — Open drain vs open source */
#define GPIO_LINE_OPEN_SOURCE  0
#define GPIO_LINE_OPEN_DRAIN   4

/* Combiners للـ common patterns */
#define GPIO_OPEN_DRAIN   (GPIO_SINGLE_ENDED | GPIO_LINE_OPEN_DRAIN)   /* = 6 */
#define GPIO_OPEN_SOURCE  (GPIO_SINGLE_ENDED | GPIO_LINE_OPEN_SOURCE)  /* = 2 */

/* Bit 3 — Sleep behavior */
#define GPIO_PERSISTENT   0   /* يحافظ على الـ state في النوم */
#define GPIO_TRANSITORY   8   /* ممكن يتغير الـ state في النوم */

/* Bit 4, 5, 6 — Pull resistors */
#define GPIO_PULL_UP      16
#define GPIO_PULL_DOWN    32
#define GPIO_PULL_DISABLE 64
```

**مثال حقيقي:** reset line لـ ethernet chip على PCB:

```dts
/* الـ ethernet chip بيتعمله reset لما الـ signal بيوصل LOW */
/* لكن الـ board designer حط buffer معكوس في الـ circuit */
/* فالـ GPIO controller لازم يطلع HIGH عشان يعمل reset */
reset-gpios = <&gpio3 7 GPIO_ACTIVE_LOW>;
/*                        ↑
 * بيقول للـ kernel: "active state" للـ signal ده هو LOW على الـ GPIO pin
 * الـ kernel بيترجم gpiod_set_value(gpio, 1) → يطلع voltage low فعلاً
 */
```

---

### الـ GPIO Controller Node — التشريح الكامل

```dts
gpio-controller@00000000 {
    compatible = "foo";              /* بيحدد الـ driver */
    reg = <0x00000000 0x1000>;       /* عنوان الـ registers في الـ memory map */

    gpio-controller;                 /* ← property فاضي = flag */
                                     /* بيقول: الـ node دي GPIO provider */

    #gpio-cells = <2>;               /* ← كام cell في كل gpio specifier */
                                     /* المعيار = 2: [offset, flags] */

    ngpios = <18>;                   /* الـ hardware register 32 bit */
                                     /* بس 18 بس ليها pins فعلية */

    gpio-reserved-ranges = <0 4>,   /* pins 0..3 مش متاحة */
                           <12 2>;  /* pins 12..13 مش متاحة */

    gpio-line-names =
        "MMC-CD",    /* offset 0 — card detect للـ SD card */
        "MMC-WP",    /* offset 1 — write protect */
        "VDD eth",   /* offset 2 — power enable للـ ethernet */
        /* ... */
        "reset";     /* offset 17 */
};
```

**`gpio-controller`** property فاضي — الـ kernel بيستخدمه كـ marker. الماكرو `for_each_gpiochip_node` في `driver.h` بيبحث عنه:

```c
/* من driver.h */
#define for_each_gpiochip_node(dev, child)                    \
    device_for_each_child_node(dev, child)                    \
        for_each_if(fwnode_property_present(child, "gpio-controller"))
/* بيمشي على كل child nodes ويفلتر اللي عندها gpio-controller */
```

---

### الـ GPIO Hog Mechanism

الـ **GPIO hog** هو آلية بيطلب فيها الـ controller نفسه GPIO lines وبيضبطها تلقائياً وقت الـ probe — من غير ما يستنى أي consumer driver.

```dts
qe_pio_a: gpio-controller@1400 {
    compatible = "fsl,qe-pario-bank-a", "fsl,qe-pario-bank";
    reg = <0x1400 0x18>;
    gpio-controller;
    #gpio-cells = <2>;

    /* child node = hog definition */
    line_b-hog {
        gpio-hog;             /* ← marker: ده hog مش consumer */
        gpios = <6 0>;        /* GPIO offset 6, flags = 0 (active high) */
        output-low;           /* ← set as output, value = 0 */
        line-name = "foo-bar-gpio";
    };
};
```

**Use case حقيقي:** power sequencing — SoC لازم يطلع power enable على GPIO معينة قبل ما أي driver تاني يبدأ يشتغل. الـ hog بيضمن ده بدون ما تحتاج driver منفصل.

---

### الـ gpio-ranges — الربط مع الـ pinctrl Subsystem

**الـ pinctrl subsystem** (محتاج تفهمه كمان): بيتحكم في الـ pin muxing والـ configuration على مستوى الـ SoC. كل pin على الـ SoC ممكن يشتغل كـ GPIO أو UART أو SPI أو I2C. الـ pinctrl هو اللي بيقرر.

الـ `gpio-ranges` property بيعمل mapping صريح:

```
gpio-ranges = <&pinctrl1  0  20  10>;
               ╔═════════╗╔═╗╔══╗╔══╗
               ║phandle   ║  ║    ║    ║
               ║to        ║  ║    ║    ║
               ║pinctrl1  ║  ║    ║    ║
               ╚═════════╝  ║    ║    ║
                  GPIO offset╝    ║    ║
                          pin offset╝    ║
                                  count╝
```

يعني: الـ GPIO lines بتاعتي من 0 لـ 9 هي نفس الـ physical pins من 20 لـ 29 في الـ pinctrl1.

**ليه ده مهم؟**

لما driver يطلب `gpiod_get()` ويحتاج يعمل pin muxing للـ GPIO function، الـ GPIO core بيستخدم الـ mapping ده عشان يعرف يتكلم مع الـ pinctrl ويطلب منه يعمل mux الـ pin على الـ GPIO function.

```
ASCII: الـ mapping flow

Physical Pin 20 on SoC
        │
        ├──► pinctrl1 (pin space: 20)
        │         ↕ gpio-ranges mapping
        └──► gpio_chip (GPIO line: 0)

لما تعمل gpiod_get()، الـ gpiolib بيعرف:
GPIO line 0 → physical pin 20 → يروح يقول للـ pinctrl يعمل mux
```

---

### الـ struct gpio_chip — قلب الـ provider

الـ `struct gpio_chip` هو الـ abstraction الأساسي اللي أي GPIO controller driver لازم يملأه:

```c
struct gpio_chip {
    /* Identity */
    const char        *label;      /* اسم الـ controller */
    struct gpio_device *gpiodev;   /* internal state، الـ core بيملأه */
    struct device     *parent;     /* الـ physical device */
    struct fwnode_handle *fwnode;  /* link للـ DT node */

    /* Operations — الـ driver لازم يوفر دي */
    int  (*request)(struct gpio_chip *gc, unsigned int offset);
    void (*free)(struct gpio_chip *gc, unsigned int offset);
    int  (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)(struct gpio_chip *gc, unsigned int offset);
    int  (*set)(struct gpio_chip *gc, unsigned int offset, int value);

    /* Bulk operations — optional لكن مهمة للـ performance */
    int  (*get_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);
    int  (*set_multiple)(struct gpio_chip *gc, unsigned long *mask, unsigned long *bits);

    /* Pin configuration — بيتكلم مع الـ pinconf layer */
    int  (*set_config)(struct gpio_chip *gc, unsigned int offset, unsigned long config);

    /* IRQ support */
    int  (*to_irq)(struct gpio_chip *gc, unsigned int offset);

    /* DT translation callback */
    int  (*of_xlate)(struct gpio_chip *gc,
                     const struct of_phandle_args *gpiospec,
                     u32 *flags);

    /* Dimensions */
    int  base;    /* first GPIO number (deprecated: use -1 for dynamic) */
    u16  ngpio;   /* عدد الـ GPIO lines */

    /* Sleep capability flag */
    bool can_sleep; /* true لو I2C/SPI expander — operations بتنام */

    /* Optional IRQ chip integration */
    struct gpio_irq_chip irq;
};
```

**العلاقة بين الـ structs:**

```
┌─────────────────────────────────────────────────┐
│                 gpio_device                      │
│  (opaque internal struct — الـ core بيملأه)     │
│                                                  │
│   ┌──────────────────────────────────────────┐  │
│   │            gpio_chip                     │  │
│   │  (الـ driver بيملأه ويسجله)              │  │
│   │                                          │  │
│   │  ┌───────────────────────────────────┐  │  │
│   │  │        gpio_irq_chip              │  │  │
│   │  │  (optional — لو الـ GPIO lines    │  │  │
│   │  │   ممكن تولد IRQs)                 │  │  │
│   │  │                                   │  │  │
│   │  │  irq_chip* → irq_domain*          │  │  │
│   │  └───────────────────────────────────┘  │  │
│   └──────────────────────────────────────────┘  │
│                                                  │
│   gpio_desc[0..ngpio-1]                          │
│   (per-line state: requested?, label, flags)     │
└─────────────────────────────────────────────────┘
```

---

### الـ GPIO IRQ Integration

لما GPIO line ممكن تولد interrupt، الـ `struct gpio_irq_chip` بيربط الـ GPIO numbering بالـ IRQ domain:

```
GPIO Controller Hardware
    line 5 ← external signal يغير state
         │
         ▼
    gpio_irq_chip.parent_handler()    ← cascaded interrupt handler
         │
         ▼
    irq_domain translation
    (GPIO offset 5 → Linux IRQ number N)
         │
         ▼
    Registered handler للـ consumer
```

**الـ irq_domain subsystem** (مهم تعرفه): هو الـ framework اللي بيعمل mapping بين hardware interrupt numbers والـ Linux virtual IRQ numbers. كل GPIO controller بيعمل `irq_domain` خاص بيه.

في حالة الـ hierarchical interrupt support:

```c
/* الـ GPIO IRQ chip بيحتاج يترجم child hwirq → parent hwirq */
int (*child_to_parent_hwirq)(struct gpio_chip *gc,
                             unsigned int child_hwirq,   /* GPIO line number */
                             unsigned int child_type,    /* IRQ_TYPE_EDGE_* */
                             unsigned int *parent_hwirq, /* GIC interrupt number */
                             unsigned int *parent_type);
```

---

### ما الـ Subsystem بيملكه vs ما بيفوضه للـ Drivers

| المسؤولية | الـ GPIO Core يملكها | الـ Driver بيتكفل بيها |
|-----------|---------------------|----------------------|
| **DT parsing** | تحليل `gpios` property وإيجاد الـ controller | تفسير الـ specifier الخاص به عبر `of_xlate` |
| **Line tracking** | تتبع مين طالب كل line ومنع الـ conflicts | اختياري: `request`/`free` callbacks للـ hw-level setup |
| **Polarity translation** | تحويل logical value → physical value حسب الـ flags | لا — الـ core بيتكفل بيه كامل |
| **GPIO-to-IRQ** | lookup من `gpio_desc` للـ irq number | `to_irq` callback للـ mapping |
| **Pin muxing** | طلب الـ mux من الـ pinctrl عبر `gpio-ranges` | `add_pin_ranges` callback للـ complex mappings |
| **Bulk operations** | orchestration والـ fallback لـ single ops | `get_multiple`/`set_multiple` للـ hw optimization |
| **Sysfs/chardev** | exposing الـ GPIOs للـ userspace | لا — الـ core بيتكفل بيه |
| **Hardware register access** | لا — ده شغل الـ driver | `get`, `set`, `direction_*` callbacks |
| **IRQ chip wiring** | إنشاء الـ irq_domain وربطه | `gpio_irq_chip` configuration والـ handlers |

---

### Flow كامل: من الـ DT لـ Hardware

```
DT Source:
    enable-gpios = <&gpio1 12 GPIO_ACTIVE_LOW>;

                    │
    parse_time      │  of_find_node_by_phandle(&gpio1)
                    ▼
    GPIO Core:      struct of_phandle_args {
                        .np = gpio1_node
                        .args = [12, 1]   /* 1 = GPIO_ACTIVE_LOW */
                        .args_count = 2
                    }
                    │
    of_xlate()      │  controller's translation callback
                    ▼
                    offset = 12
                    flags = GPIO_ACTIVE_LOW

                    │
    gpio_desc       │  per-line descriptor
    allocation      ▼
                    struct gpio_desc {
                        struct gpio_device *gdev  → gpio1
                        unsigned int offset = 12
                        unsigned long flags = GPIOD_FLAGS_OUTPUT_LOW
                                            | GPIO_ACTIVE_LOW (inverted)
                    }

                    │
    Consumer        │  gpiod_set_value(desc, 1)  ← logical 1 = "assert"
    calls           ▼
                    GPIO Core sees GPIO_ACTIVE_LOW flag
                    → translates logical 1 to physical 0
                    → calls gc->set(gc, 12, 0)
                    │
    Driver          │  write 0 to hardware register bit 12
                    ▼
    Hardware:       GPIO pin 12 goes LOW ✓
```

---

### مثال متكامل — Raspberry Pi Style Board

```dts
/ {
    /* الـ SoC GPIO controller */
    gpio0: gpio@7e200000 {
        compatible = "brcm,bcm2835-gpio";
        reg = <0x7e200000 0xb4>;
        gpio-controller;
        #gpio-cells = <2>;
        ngpios = <54>;

        /* الـ pins دي shared مع pinctrl */
        gpio-ranges = <&pinctrl 0 0 54>;
    };

    /* الـ SD card controller — consumer */
    sdhost: mmc@7e202000 {
        compatible = "brcm,bcm2835-sdhost";
        reg = <0x7e202000 0x100>;

        /* Card detect على GPIO 47، active low على الـ hardware */
        cd-gpios = <&gpio0 47 GPIO_ACTIVE_LOW>;

        /* Write protect على GPIO 48، active high */
        wp-gpios = <&gpio0 48 GPIO_ACTIVE_HIGH>;
    };

    /* LED على الـ board */
    leds {
        compatible = "gpio-leds";

        status-led {
            /* GPIO 17، LED بيضوي لما الـ GPIO high */
            gpios = <&gpio0 17 GPIO_ACTIVE_HIGH>;
            default-state = "on";
        };
    };

    /* Power management — hog مثال */
    pwr-gpio-controller@7e200000 {
        /* ... */
        wifi-pwr-hog {
            gpio-hog;
            gpios = <35 0>;      /* GPIO 35 */
            output-high;         /* enable WiFi power at boot */
            line-name = "WiFi-PWR-EN";
        };
    };
};
```

الـ `gpio-leds` driver بيعمل:
```c
/* في probe */
struct gpio_desc *led_gpio = devm_gpiod_get(&pdev->dev, NULL, GPIOD_OUT_HIGH);
/* الـ gpiolib بيدور على "gpios" property، بيلاقي &gpio0 17 GPIO_ACTIVE_HIGH */
/* بيرجع gpio_desc جاهز — الـ driver مش شايف أي hardware detail */
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### GPIO Specifier Flags (من `include/dt-bindings/gpio/gpio.h`)

| Macro | القيمة | الـ Bit | المعنى |
|---|---|---|---|
| `GPIO_ACTIVE_HIGH` | `0` | bit 0 = 0 | الـ signal فعّال لما يبقى HIGH |
| `GPIO_ACTIVE_LOW` | `1` | bit 0 = 1 | الـ signal فعّال لما يبقى LOW (inverted) |
| `GPIO_PUSH_PULL` | `0` | bit 1 = 0 | wiring عادي push-pull |
| `GPIO_SINGLE_ENDED` | `2` | bit 1 = 1 | single-ended wiring |
| `GPIO_LINE_OPEN_SOURCE` | `0` | bit 2 = 0 | open source (emitter) |
| `GPIO_LINE_OPEN_DRAIN` | `4` | bit 2 = 1 | open drain (collector) |
| `GPIO_OPEN_DRAIN` | `6` | bits 1+2 | single-ended + open drain (الأشيع) |
| `GPIO_OPEN_SOURCE` | `2` | bits 1+2 | single-ended + open source |
| `GPIO_PERSISTENT` | `0` | bit 3 = 0 | الـ output يتحافظ عليه في الـ sleep |
| `GPIO_TRANSITORY` | `8` | bit 3 = 1 | الـ output ممكن يتضيع في الـ sleep |
| `GPIO_PULL_UP` | `16` | bit 4 = 1 | تفعيل الـ pull-up resistor |
| `GPIO_PULL_DOWN` | `32` | bit 5 = 1 | تفعيل الـ pull-down resistor |
| `GPIO_PULL_DISABLE` | `64` | bit 6 = 1 | تعطيل الـ pull resistor |

#### GPIO Direction Constants (من `include/linux/gpio/driver.h`)

| Macro | القيمة | المعنى |
|---|---|---|
| `GPIO_LINE_DIRECTION_IN` | `1` | الـ line اتجاهها input |
| `GPIO_LINE_DIRECTION_OUT` | `0` | الـ line اتجاهها output |

#### DT Properties المهمة — Cheatsheet

| Property | المكان في الـ DT | إلزامي؟ | المعنى |
|---|---|---|---|
| `gpio-controller` | controller node | نعم | بيعلن إن الـ node ده GPIO controller |
| `#gpio-cells` | controller node | نعم | عدد الـ cells في الـ gpio-specifier (standard: 2) |
| `ngpios` | controller node | لا | عدد الـ GPIO lines الفعلية المستخدمة |
| `gpio-reserved-ranges` | controller node | لا | tuples تحدد ranges الـ GPIOs غير القابلة للاستخدام |
| `gpio-line-names` | controller node | لا | أسماء الـ GPIO lines بالترتيب من offset 0 |
| `gpio-ranges` | controller node | لا | ربط الـ GPIO lines بالـ pin controller |
| `gpio-ranges-group-names` | controller node | لا | أسماء pin groups بدل الـ offsets الرقمية |
| `gpio-hog` | child node | نعم (للـ hog) | بيعلن إن الـ child node ده GPIO hog |
| `gpios` | consumer / hog node | نعم | الـ phandle + specifier للـ GPIO |
| `[name]-gpios` | consumer node | لا | تسمية GPIO بحسب وظيفته في الـ device |
| `input` | hog node | أحدهم إلزامي | الـ hog بيتجه كـ input |
| `output-low` | hog node | أحدهم إلزامي | الـ hog بيتجه كـ output بقيمة LOW |
| `output-high` | hog node | أحدهم إلزامي | الـ hog بيتجه كـ output بقيمة HIGH |
| `line-name` | hog node | لا | اسم مخصص للـ GPIO hog |

---

### الـ Structs المهمة وعلاقاتها

#### 1. `struct gpio_chip` — العمود الفقري

**الغرض:** الـ struct الرئيسي اللي بيمثل أي GPIO controller في الـ kernel. الـ driver بيملّي الـ fields دي وبعدين يسجّلها عن طريق `gpiochip_add_data()`.

| Field | النوع | المعنى |
|---|---|---|
| `label` | `const char *` | اسم وصفي للـ chip (زي "gpio-bcm2835") |
| `gpiodev` | `struct gpio_device *` | الـ internal state — opaque، الـ gpiolib بيمتلكها |
| `parent` | `struct device *` | الـ parent device في الـ device model |
| `fwnode` | `struct fwnode_handle *` | ربط الـ chip بالـ DT node أو ACPI node |
| `owner` | `struct module *` | بيمنع تفكيك الـ module لو في GPIOs نشطة |
| `base` | `int` | أول رقم GPIO في الـ global space (سلبي = dynamic) |
| `ngpio` | `u16` | عدد الـ lines اللي الـ chip بيتحكم فيها |
| `offset` | `u16` | offset داخل الـ device لو في أكتر من chip في نفس الـ device |
| `names` | `const char *const *` | أسماء بديلة للـ GPIO lines (من `gpio-line-names` في الـ DT) |
| `can_sleep` | `bool` | لازم يتحدد لو الـ get/set بتنام (زي I2C expanders) |
| `irq` | `struct gpio_irq_chip` | الـ IRQ chip المدمج (لو `CONFIG_GPIOLIB_IRQCHIP` متفعّل) |
| `of_gpio_n_cells` | `unsigned int` | عدد الـ cells في الـ DT specifier (بيتطابق مع `#gpio-cells`) |
| `of_xlate` | `fn ptr` | بيترجم الـ DT specifier لـ chip-relative GPIO number + flags |
| `of_node_instance_match` | `fn ptr` | بيحدد أي instance لما يكون في أكتر من chip لنفس الـ DT node |
| `request` | `fn ptr` | activation hook لما يطلب حد الـ GPIO |
| `free` | `fn ptr` | deactivation hook لما يتحرر الـ GPIO |
| `get_direction` | `fn ptr` | بيرجع direction الـ line (0=out, 1=in) |
| `direction_input` | `fn ptr` | بيحوّل الـ line لـ input |
| `direction_output` | `fn ptr` | بيحوّل الـ line لـ output بقيمة معينة |
| `get` | `fn ptr` | بيقرأ قيمة الـ GPIO line |
| `set` | `fn ptr` | بيكتب قيمة على الـ GPIO line |
| `get_multiple` | `fn ptr` | بيقرأ قيم أكتر من line في عملية واحدة (كفاءة أعلى) |
| `set_multiple` | `fn ptr` | بيكتب قيم على أكتر من line في عملية واحدة |
| `set_config` | `fn ptr` | بيضبط الـ pin configuration (pull-up/down، drive strength) |
| `to_irq` | `fn ptr` | بيحوّل GPIO offset لـ Linux IRQ number |
| `add_pin_ranges` | `fn ptr` | بيضيف الـ pin ranges للـ pin controller يدوياً |
| `init_valid_mask` | `fn ptr` | بيحدد أنهي GPIOs صالحة للـ IRQ |

---

#### 2. `struct gpio_irq_chip` — الـ IRQ Integration

**الغرض:** بيدمج وظيفة الـ interrupt controller جوّا الـ GPIO chip. موجود كـ field داخل `struct gpio_chip`.

| Field | النوع | المعنى |
|---|---|---|
| `chip` | `struct irq_chip *` | الـ IRQ chip implementation من الـ driver |
| `domain` | `struct irq_domain *` | الـ IRQ domain اللي بيترجم بين GPIO hwirq و Linux IRQ |
| `fwnode` | `struct fwnode_handle *` | firmware node للـ hierarchical irqdomain |
| `parent_domain` | `struct irq_domain *` | الـ parent IRQ domain (للـ hierarchical IRQ) |
| `handler` | `irq_flow_handler_t` | الـ IRQ flow handler (زي `handle_simple_irq`) |
| `default_type` | `unsigned int` | نوع الـ triggering الافتراضي (edge/level) |
| `threaded` | `bool` | لو true، الـ IRQ handlers بتشتغل في threads منفصلة |
| `num_parents` | `unsigned int` | عدد الـ parent interrupts للـ chip |
| `parents` | `unsigned int *` | قائمة الـ Linux IRQ numbers للـ parent interrupts |
| `valid_mask` | `unsigned long *` | bitmask للـ GPIOs الصالحة للـ IRQ |
| `first` | `unsigned int` | أول IRQ (للـ static allocation) |
| `child_to_parent_hwirq` | `fn ptr` | ترجمة child hwirq لـ parent hwirq |
| `init_hw` | `fn ptr` | initialization للـ hardware قبل إضافة الـ IRQ chip |
| `irq_enable/disable/mask/unmask` | `fn ptrs` | callbacks قديمة محفوظة لـ wrapping |

---

#### 3. `struct gpio_device` — الـ Internal State (Opaque)

**الغرض:** الـ struct الداخلي اللي الـ gpiolib بيمتلكه. الـ driver مش بيتعامل معاه مباشرة. بيمثل الـ GPIO controller في الـ device model وبيحتوي على الـ descriptors والـ locks.

**الـ fields الأساسية (من الـ implementation):**
- قائمة الـ `gpio_desc` لكل line
- الـ `spinlock` الرئيسي للـ chip
- الـ `srcu` للـ notifiers
- الـ `device` المسجّل في `/sys/class/gpio/`

---

#### 4. `struct gpio_desc` — وصف الـ GPIO Line الواحدة

**الغرض:** بيمثل GPIO line واحدة (pin واحد). الـ consumer بيتعامل معاه عن طريق `gpiod_get()` وأخواتها.

**الـ fields الأساسية:**
- pointer للـ `gpio_device` اللي بيمتلكها
- الـ flags (direction، active-low، open-drain، إلخ)
- اسم الـ label اللي طلب بيها الـ consumer
- الـ offset جوّا الـ chip

---

#### 5. `union gpio_irq_fwspec`

**الغرض:** بيوحّد بين `struct irq_fwspec` العادي و `msi_alloc_info_t` للـ MSI interrupts في نفس الـ memory location.

```c
union gpio_irq_fwspec {
    struct irq_fwspec    fwspec;    /* standard IRQ firmware spec */
#ifdef CONFIG_GENERIC_MSI_IRQ
    msi_alloc_info_t     msiinfo;   /* MSI allocation info */
#endif
};
```

---

### رسوم العلاقات بين الـ Structs

```
DT Node (gpio-controller@XXXX)
        |
        | of_xlate() يترجم الـ specifier
        v
+-------------------------+
|     struct gpio_chip    |<--- الـ driver بيملّيه ويسجّله
|-------------------------|
| label                   |
| ngpio                   |
| base                    |
| of_gpio_n_cells         |-----> عدد cells في #gpio-cells
| fwnode ----------------+|-----> DT node
| parent ---------------+||-----> struct device (SoC/platform)
| owner                  |||
| get/set/direction...   |||
|                        |||
| gpiodev -------------->+||-----> struct gpio_device (opaque, gpiolib-owned)
|                         ||           |
| irq (embedded) ---------+|           +---> gpio_desc[0]
|   struct gpio_irq_chip   |           +---> gpio_desc[1]
|   - chip ----------------+-----> struct irq_chip     ...
|   - domain --------------+-----> struct irq_domain    +---> gpio_desc[N-1]
|   - parent_domain        |
+-------------------------+
```

```
Consumer DT Node
    enable-gpios = <&gpio1 12 GPIO_ACTIVE_LOW>;
         |
         | kernel parses phandle
         v
    gpio_chip (gpio1)
         |
         | of_xlate(offset=12, flags=GPIO_ACTIVE_LOW)
         v
    gpio_desc[12]   <--- gpiod_get() بترجع pointer ليه
         |
         | gpiod_set_value() / gpiod_get_value()
         v
    gpio_chip->set(gc, 12, value)  <--- hardware access
```

---

### كيف الـ DT Properties بتتحول لـ Kernel Objects

#### `#gpio-cells` و `of_gpio_n_cells`

```
DT:                              Kernel:
#gpio-cells = <2>;    ------>    gc->of_gpio_n_cells = 2
                                 الـ specifier بيبقى: [offset, flags]
```

#### `ngpios` و `gpio-reserved-ranges`

```
DT:                              Kernel:
ngpios = <18>;        ------>    gc->ngpio = 18
gpio-reserved-ranges =           gpiolib بيعمل mask على الـ
  <0 4>, <12 2>;                 descriptors التالية:
                                 0,1,2,3 و 12,13 → غير قابلة للطلب
```

#### `gpio-line-names`

```
DT:
gpio-line-names = "MMC-CD", "MMC-WP", ...;

                    ↓

Kernel:
gc->names = {"MMC-CD", "MMC-WP", ...}
gpio_desc[0]->name = "MMC-CD"
gpio_desc[1]->name = "MMC-WP"
...
```

#### `gpio-ranges` و Pin Controller

```
DT:
gpio-ranges = <&pinctrl1 0 20 10>;

                    ↓

Kernel: gpiochip_add_pin_range(gc, "pinctrl1", 0, 20, 10)
        يعني GPIO offset 0..9 ↔ pinctrl1 pin 20..29

        +-------------+        gpio-ranges        +-------------+
        | gpio_chip   |<--------------------------| pinctrl_dev |
        | offset 0..9 |    pinctrl_gpio_range      | pin 20..29  |
        +-------------+                           +-------------+
```

---

### دورة حياة الـ GPIO Controller — من الـ DT لـ Hardware

```
┌─────────────────────────────────────────────────────────────────┐
│                        CREATION                                  │
│                                                                  │
│  kernel boot / module load                                       │
│       │                                                          │
│       ▼                                                          │
│  platform_driver.probe()                                         │
│       │                                                          │
│       ▼                                                          │
│  Driver reads DT:                                                │
│    - of_property_read_u32("ngpios", &gc->ngpio)                 │
│    - of_property_read_string_array("gpio-line-names", ...)      │
│    - gc->of_gpio_n_cells = 2  (من #gpio-cells)                  │
│    - gc->fwnode = dev_fwnode(dev)                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       REGISTRATION                               │
│                                                                  │
│  gpiochip_add_data(gc, driver_data)                             │
│       │                                                          │
│       ├─► يخصص gpio_device + gpio_desc[ngpio]                  │
│       ├─► يسجّل الـ device في /sys/class/gpio/                 │
│       ├─► يضيف الـ pin ranges (من gpio-ranges في الـ DT)       │
│       ├─► يضيف الـ IRQ chip (لو gpio_irq_chip.chip != NULL)    │
│       └─► يعمل GPIO hogging (من child nodes في الـ DT)         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                          USAGE                                   │
│                                                                  │
│  Consumer driver:                                                │
│    desc = gpiod_get(dev, "enable", GPIOD_OUT_LOW)               │
│       │                                                          │
│       ├─► kernel يلاقي "enable-gpios" property في الـ DT       │
│       ├─► يفك الـ phandle ويطلع gpio_chip المناسب              │
│       ├─► يستدعي of_xlate() يترجم [offset, flags]              │
│       ├─► يطلب gpio_desc[offset] (gpio_chip->request)          │
│       └─► بيرجع gpio_desc pointer للـ consumer                  │
│                                                                  │
│    gpiod_set_value(desc, 1)                                      │
│       └─► gpio_chip->set(gc, offset, 1)  [hardware write]       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         TEARDOWN                                 │
│                                                                  │
│  Consumer:  gpiod_put(desc)  ──► gpio_chip->free(gc, offset)   │
│                                                                  │
│  Driver removal:                                                 │
│    gpiochip_remove(gc)                                           │
│       ├─► يتحقق إن مفيش GPIOs محجوزة (warning لو في)          │
│       ├─► يشيل الـ IRQ chip والـ IRQ domain                    │
│       ├─► يشيل الـ pin ranges                                  │
│       ├─► يمسح الـ gpio_device من الـ device model              │
│       └─► يحرر الـ gpio_desc array                              │
└─────────────────────────────────────────────────────────────────┘
```

---

### دورة حياة الـ GPIO Hog

```
gpiochip_add_data()
      │
      └─► gpiochip_of_hog_scan() تفحص كل child nodes في الـ DT
                │
                │  for each child with "gpio-hog" property:
                ▼
          gpiochip_of_hog()
                │
                ├─► يقرأ "gpios" property  →  يطلع offset + flags
                ├─► يقرأ "input" أو "output-low" أو "output-high"
                ├─► يقرأ "line-name" (أو يستخدم اسم الـ node)
                ├─► يستدعي gpiod_request() على الـ gpio_desc
                ├─► يضبط الـ direction (gpiod_direction_input/output)
                └─► الـ GPIO بيفضل محجوز طول ما الـ controller شغّال
```

---

### Call Flow — من Consumer Request لـ Hardware Access

```
Consumer Code                   gpiolib                    Driver
─────────────                   ───────                    ──────
gpiod_get(dev, "reset", ...)
      │
      ▼
of_find_gpio()
      │ يفحص "reset-gpios" في DT
      ▼
of_get_named_gpiod_flags()
      │ يفك الـ phandle
      ▼
of_xlate() ──────────────────────────────────────────► gc->of_xlate()
      │                                                      │
      │ offset, flags                                        │
      ◄──────────────────────────────────────────────────────┘
      │
      ▼
gpiod_request()
      │
      ▼
gc->request(gc, offset) ─────────────────────────────► driver->request()
                                                              (clock enable, etc.)
      │ desc ready
      ▼
gpiod_set_value(desc, 1)
      │
      ▼
gpiod_set_raw_value_cansleep()   [handles active-low inversion]
      │
      ▼
gc->set(gc, offset, value) ──────────────────────────► driver->set()
                                                         writel(val, reg)
```

---

### Call Flow — من DT GPIO Range لـ Pin Controller

```
gpiochip_add_data()
      │
      ├─► gc->add_pin_ranges(gc)      [driver-specific, optional]
      │         │
      │         └─► gpiochip_add_pin_range(gc, pinctrl_name, gpio_off, pin_off, npins)
      │                   │
      │                   └─► pinctrl_find_and_add_gpio_range()
      │                             │
      │                             └─► pinctrl_dev gets gpio_range registered
      │
      └─► OR: gpiolib يقرأ "gpio-ranges" من الـ DT تلقائياً
                    │
                    └─► of_gpiochip_add_pin_range()
                              │
                              ├─► numerical ranges: gpiochip_add_pin_range()
                              └─► named groups: pinctrl_get_group_pins()
                                               + gpiochip_add_pin_range()
```

---

### استراتيجية الـ Locking

#### الـ Locks الموجودة وما تحميه

| Lock | النوع | ما يحميه |
|---|---|---|
| `gpio_device->srcu` | `struct srcu_struct` | الـ notifier list لأحداث الـ GPIO lines |
| Internal `spinlock` في `gpio_device` | `spinlock_t` | الـ gpio_desc flags (direction، value، request state) |
| `gpio_irq_chip.lock_key` | `struct lock_class_key` | lockdep class للـ IRQ lock |
| `gpio_irq_chip.request_key` | `struct lock_class_key` | lockdep class للـ IRQ request |
| `irq_domain` internal lock | `mutex` | الـ IRQ mapping table |

#### قواعد الـ Locking

**الـ spinlock** بيُستخدم في:
- تغيير الـ gpio_desc flags (request/free، direction، value)
- السياقات اللي مش ممكن تنام (interrupt context)

**الـ mutex** بيُستخدم في:
- عمليات الـ pin range (add/remove)
- الـ IRQ domain mappings
- أي عملية ممكن تنام

#### ترتيب الـ Locks (Lock Ordering)

```
pinctrl_dev->mutex
      │
      └─► gpio_device spinlock
                │
                └─► irq_desc lock    ← الأعمق، لازم يتحصل آخر
```

> لو الـ `gc->can_sleep = true` (زي I2C expander)، الـ `get/set` callbacks بتنام، فبالتالي:
> - مش ممكن تتنادى من interrupt context
> - الـ IRQ handling لازم يبقى threaded (`gpio_irq_chip.threaded = true`)
> - الـ gpiolib بيتحقق من ده وبيطبع warning لو اتخالف

#### لما الـ `gpio-ranges` موجودة في الـ DT

الـ pin controller مش ممكن يتاخد out of mux بدون الـ GPIO subsystem يعرف — الـ lock ordering بيبقى:
```
pinctrl subsystem lock  →  gpio subsystem lock
```
الـ kernel بيطبّق ده عن طريق lockdep annotations في الـ `pinctrl_gpio_request()`.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ DT Properties

| Property | Node | الوصف |
|---|---|---|
| `gpio-controller` | GPIO controller | بتعلن إن الـ node ده gpio controller |
| `#gpio-cells` | GPIO controller | عدد الـ cells في الـ gpio-specifier |
| `ngpios` | GPIO controller | عدد الـ GPIO lines الفعلية المستخدمة |
| `gpio-reserved-ranges` | GPIO controller | ranges من الـ GPIOs المحجوزة وغير قابلة للاستخدام |
| `gpio-line-names` | GPIO controller | أسماء نصية لكل GPIO line |
| `gpio-ranges` | GPIO controller | ربط GPIO offsets بـ pin controller pins |
| `gpio-ranges-group-names` | GPIO controller | أسماء pin groups بدل الـ numerical ranges |
| `gpio-hog` | child of controller | يعلن إن الـ child node ده GPIO hog |
| `gpios` | hog child / consumer | GPIO specifier (id + flags) |
| `input` | hog child | اتجاه الـ GPIO: input |
| `output-low` | hog child | اتجاه output، قيمة low |
| `output-high` | hog child | اتجاه output، قيمة high |
| `line-name` | hog child | label اختياري للـ GPIO المحجوز |
| `[name-]gpios` | consumer device | الـ GPIO property في الـ consumer |

#### الـ DT Macros (include/dt-bindings/gpio/gpio.h)

| Macro | Value | المعنى |
|---|---|---|
| `GPIO_ACTIVE_HIGH` | 0 | الـ signal active عند high level |
| `GPIO_ACTIVE_LOW` | 1 | الـ signal active عند low level |
| `GPIO_PUSH_PULL` | 0 | wiring push-pull |
| `GPIO_SINGLE_ENDED` | 2 | single-ended wiring |
| `GPIO_OPEN_DRAIN` | 4 | open-drain (مع GPIO_SINGLE_ENDED) |
| `GPIO_TRANSITORY` | 8 | الـ output ممكن يتخسش في sleep |
| `GPIO_PULL_UP` | 16 | تفعيل pull-up |
| `GPIO_PULL_DOWN` | 32 | تفعيل pull-down |

#### الـ Kernel API Functions

| Function | الموقع | الغرض |
|---|---|---|
| `of_get_named_gpio()` | `gpiolib-of.c` | جيب GPIO number باسم الـ property — **deprecated** |
| `of_get_named_gpiod_flags()` | `gpiolib-of.c` | جيب `gpio_desc` + flags من DT |
| `of_gpio_named_count()` | `gpiolib-of.c` | عد الـ GPIOs في property معينة |
| `of_gpio_count()` | `gpiolib-of.c` | عد الـ GPIOs مع handling للـ SPI quirks |
| `gpiod_get()` | `gpiolib.c` | جيب GPIO descriptor من consumer device |
| `devm_gpiod_get()` | `gpiolib.c` | نفس `gpiod_get` بس مع devres cleanup |
| `gpiod_get_index()` | `gpiolib.c` | جيب GPIO descriptor برقم index |
| `devm_gpiod_get_index()` | `gpiolib.c` | نفس `gpiod_get_index` مع devres |
| `gpiod_get_optional()` | `gpiolib.c` | زي `gpiod_get` بس مش error لو مش موجود |
| `devm_gpiod_get_optional()` | `gpiolib.c` | devres نسخة من `gpiod_get_optional` |
| `of_xlate_and_get_gpiod_flags()` | `gpiolib-of.c` | تحويل الـ phandle args لـ `gpio_desc` |
| `of_find_gpio_device_by_xlate()` | `gpiolib-of.c` | البحث عن الـ `gpio_device` المناسب |
| `of_gpio_flags_quirks()` | `gpiolib-of.c` | تطبيق الـ legacy polarity quirks |

---

### Category 1: الـ DT Binding Properties والـ Specifier

الـ binding يبدأ بتعريف الـ **gpio-controller** node في الـ DTS، وده اللي بيحدد كل حاجة تانية. الـ consumer device مش بتتعامل مع الـ controller مباشرة — بتستخدم الـ phandle في الـ `[name-]gpios` property.

#### `#gpio-cells`

```c
/* DTS property — integer */
#gpio-cells = <2>;
```

بتحدد عدد الـ u32 cells اللي بتكون الـ gpio-specifier. القيمة الأكثر شيوعاً هي `<2>` (pin offset + flags). الـ kernel بيستخدمها في `of_parse_phandle_with_args_map()` عشان يعرف يفسر الـ phandle args.

**Best practice:** استخدم `<2>` دايماً — الـ cell التانية للـ flags.

#### `gpio-controller`

```c
/* DTS property — empty boolean */
gpio-controller;
```

الـ property دي empty (presence-only). وجودها بيخلي الـ OF layer يتعامل مع الـ node ده كـ GPIO provider. الـ kernel driver لازم يسجل `struct gpio_chip` بحيث يربطه بالـ device node ده.

#### `ngpios`

```c
/* DTS property — u32 */
ngpios = <18>;
```

بتخبر الـ driver بعدد الـ GPIO lines الفعلية لما الـ hardware register بيكون أكبر من الـ pins الفعلية. مثلاً، register 32-bit بس 18 pin فقط موجودة فعلياً. الـ driver بياخد الـ value دي عن طريق `of_property_read_u32(np, "ngpios", &ngpios)`.

#### `gpio-reserved-ranges`

```c
/* DTS property — array of <start size> tuples */
gpio-reserved-ranges = <0 4>, <12 2>;
```

لما الـ GPIOs المتاحة مش contiguous من offset 0، الـ property دي بتحدد الـ ranges المحجوزة. الـ kernel بيستخدمها مع `of_gpiochip_add()` لعمل mark على الـ lines دي كـ invalid.

#### `gpio-line-names`

```c
/* DTS property — string array */
gpio-line-names = "MMC-CD", "MMC-WP", "VDD eth", ..., "reset";
```

مصفوفة أسماء للـ GPIO lines، الـ index بيوافق الـ GPIO offset. الـ kernel بيعرضها في `/sys/kernel/debug/gpio` وبيستخدمها الـ userspace tools زي `gpioinfo`. الاسم المناسب بيوصف الوظيفة مش الـ pin name.

---

### Category 2: الـ GPIO Specifier (الـ Flags Bits)

الـ **gpio-specifier** هو القيم اللي بتيجي بعد الـ phandle في الـ `gpios` property. الـ cell التانية (flags) بتتكون من bitfield موثق في الـ DT binding spec:

```
enable-gpios = <&gpio1 18 GPIO_ACTIVE_LOW>;
/*                       ^^                   offset */
/*                           ^^^^^^^^^^^^^^   flags  */
```

| Bit | 0 | 1 |
|---|---|---|
| Bit 0 (polarity) | Active High | Active Low |
| Bit 1 (wiring type) | Push-Pull | Single-Ended |
| Bit 2 (open type) | Open Source | Open Drain |
| Bit 3 (transitory) | Maintain during sleep | يتخسش في sleep |
| Bit 4 (pull-up) | لا pull-up | Enable pull-up |
| Bit 5 (pull-down) | لا pull-down | Enable pull-down |

الـ kernel بيعمل map للـ flags دي لـ `enum of_gpio_flags` في `gpiolib-of.c`:

```c
enum of_gpio_flags {
    OF_GPIO_ACTIVE_LOW    = 0x1,
    OF_GPIO_SINGLE_ENDED  = 0x2,
    OF_GPIO_OPEN_DRAIN    = 0x4,
    OF_GPIO_TRANSITORY    = 0x8,
    OF_GPIO_PULL_UP       = 0x10,
    OF_GPIO_PULL_DOWN     = 0x20,
    OF_GPIO_PULL_DISABLE  = 0x40,
};
```

---

### Category 3: Parsing Functions — جيب الـ GPIO من DT

#### `of_get_named_gpiod_flags()` — الـ Core Function

```c
static struct gpio_desc *of_get_named_gpiod_flags(
    const struct device_node *np,
    const char *propname,
    int index,
    enum of_gpio_flags *flags);
```

دي الـ function الأساسية اللي كل حاجة تانية بتبني عليها. بتعمل parse للـ `[name-]gpios` property وبترجع `gpio_desc` جاهز للاستخدام مع الـ GPIO descriptor API.

**Parameters:**
- `np` — الـ device node للـ consumer (مش الـ controller)
- `propname` — اسم الـ property، مثلاً `"enable-gpios"` أو `"reset-gpios"`
- `index` — رقم الـ GPIO في حال الـ property فيها أكتر من واحد (zero-based)
- `flags` — pointer لـ `enum of_gpio_flags`، بيتملأ بالـ flags لو مش NULL

**Return:**
- `gpio_desc *` valid لو نجح
- `ERR_PTR(-EPROBE_DEFER)` لو الـ GPIO controller لسه مش registered
- `ERR_PTR(-EINVAL)` لو الـ specifier size غلط

**Pseudocode:**
```
of_get_named_gpiod_flags(np, propname, index, flags):
    1. of_parse_phandle_with_args_map(np, propname, "gpio", index, &gpiospec)
       → يفسر الـ phandle ويملأ gpiospec.np و gpiospec.args[]
    2. of_find_gpio_device_by_xlate(&gpiospec)
       → يلاقي gpio_device المناسب عن طريق مقارنة device node + xlate
    3. of_xlate_and_get_gpiod_flags(chip, &gpiospec, flags)
       → يستدعي chip->of_xlate() → يرجع gpio_desc
    4. of_gpio_flags_quirks(np, propname, flags, index)
       → يعدل الـ flags لو في legacy quirks
    5. of_node_put(gpiospec.np)   ← reference counting cleanup
    6. return desc
```

**Locking:** بيحتاج يحط الـ gpio_device reference (`__free(gpio_device_put)`) عشان thread-safe.

---

#### `of_get_named_gpio()` — الـ Legacy API (Deprecated)

```c
int of_get_named_gpio(const struct device_node *np,
                      const char *propname,
                      int index);
```

**مهم جداً:** الـ function دي **deprecated** رسمياً. موجودة للـ backward compatibility بس. الكود الجديد لازم يستخدم `gpiod_get()` أو `devm_gpiod_get()`.

بتعمل نفس اللي بتعمله `of_get_named_gpiod_flags()` بس بترجع الـ GPIO number الـ integer (legacy GPIO numbering system) بدل الـ `gpio_desc`.

**Parameters:** نفس الـ parameters بس من غير `flags`.

**Return:** GPIO number (>= 0) لو نجح، أو negative errno لو فشل.

**Caller context:** Driver probe functions — process context فقط.

```c
/* مثال على الاستخدام — deprecated style */
int gpio = of_get_named_gpio(np, "reset-gpios", 0);
if (gpio < 0)
    return gpio;
gpio_request(gpio, "reset");
gpio_direction_output(gpio, 0);
```

---

#### `of_gpio_named_count()` — عد الـ GPIOs

```c
static int of_gpio_named_count(const struct device_node *np,
                               const char *propname);
```

بتعد عدد الـ GPIO specifiers في property معينة. بتدي `of_count_phandle_with_args()` باسم `"#gpio-cells"` كـ cells property.

**Return:**
- عدد الـ GPIOs (بيشمل الـ empty/null specifiers)
- `EINVAL-` لو الـ property malformed
- `ENOENT-` لو الـ property مش موجودة أصلاً

**مثال:** لو `data-gpios = <&gpio1 12 0>, <&gpio1 13 0>` ترجع 2.

---

#### `of_gpio_count()` — عد الـ GPIOs مع SPI Quirk

```c
int of_gpio_count(const struct fwnode_handle *fwnode, const char *con_id);
```

بتعد الـ GPIOs بنفس منطق `of_gpio_named_count()` بس مع handling خاص للـ SPI legacy devices. بعض الـ Freescale وPPC SPI controllers بتستخدم `"gpios"` بدل `"cs-gpios"` للـ chip selects.

**Parameters:**
- `fwnode` — firmware node (بيتحول لـ `device_node` جوه)
- `con_id` — اسم الـ function، مثلاً `"cs"` للـ chip select

**الـ SPI Quirk:** لو `con_id == "cs"` والـ compatible هو `"fsl,spi"` أو `"aeroflexgaisler,spictrl"` أو `"ibm,ppc4xx-spi"`، بتعد `"gpios"` بدل `"cs-gpios"`.

---

### Category 4: Lookup والـ Translation Functions

#### `of_find_gpio_device_by_xlate()`

```c
static struct gpio_device *
of_find_gpio_device_by_xlate(const struct of_phandle_args *gpiospec);
```

بتدور على الـ `gpio_device` اللي الـ `np` بتاعه بيوافق الـ `gpiospec->np` وعنده `of_xlate` callback يقبل الـ specifier ده. بتستخدم `gpio_device_find()` مع custom matcher.

**Return:** pointer لـ `gpio_device` (مع reference) أو NULL لو مش موجود — الـ caller لازم يعمل `gpio_device_put()`.

---

#### `of_xlate_and_get_gpiod_flags()`

```c
static struct gpio_desc *of_xlate_and_get_gpiod_flags(
    struct gpio_chip *chip,
    struct of_phandle_args *gpiospec,
    enum of_gpio_flags *flags);
```

بتستدعي `chip->of_xlate()` (اللي بيوفره الـ controller driver) لتحويل الـ raw specifier args لـ GPIO offset داخل الـ chip، وبعدين بتجيب الـ `gpio_desc` المناسب عن طريق `gpiochip_get_desc()`.

**Validation:** لو `chip->of_gpio_n_cells != gpiospec->args_count` ترجع `ERR_PTR(-EINVAL)` — ده بيحمي من mismatch بين الـ `#gpio-cells` في DT والـ driver.

---

### Category 5: الـ Polarity Quirks Functions

#### `of_gpio_flags_quirks()`

```c
static void of_gpio_flags_quirks(const struct device_node *np,
                                 const char *propname,
                                 enum of_gpio_flags *flags,
                                 int index);
```

واجهة تجميعية بتطبق كل الـ legacy polarity overrides بالترتيب:
1. `of_gpio_try_fixup_polarity()` — static fixes للـ DTS اللي فيها polarity غلط
2. `of_gpio_set_polarity_by_property()` — polarity محددة بـ DT property تانية
3. Legacy open drain لـ `reg-fixed-voltage`
4. SPI chip select polarity من `spi-cs-high` property

---

#### `of_gpio_quirk_polarity()`

```c
static void of_gpio_quirk_polarity(const struct device_node *np,
                                   bool active_high,
                                   enum of_gpio_flags *flags);
```

بتعدل الـ `OF_GPIO_ACTIVE_LOW` bit في الـ flags حسب الـ `active_high` parameter. لو في تعارض (مثلاً الـ DT قال active_high بس الـ flag قال active_low) بتطبع warning وبتعمل override.

---

#### `of_gpio_try_fixup_polarity()`

```c
static void of_gpio_try_fixup_polarity(const struct device_node *np,
                                       const char *propname,
                                       enum of_gpio_flags *flags);
```

بتشيك الـ device node ضد جدول ثابت من الـ compatible strings والـ property names اللي معروف إن فيها polarity غلطة في DTS قديمة. مثلاً:
- `himax,hx8357` + `gpios-reset` → force active low
- `qi,lb60` + `rb-gpios` → force active high

ده **compile-time conditional** — كل entry محمية بـ `#if IS_ENABLED(...)`.

---

#### `of_gpio_set_polarity_by_property()`

```c
static void of_gpio_set_polarity_by_property(const struct device_node *np,
                                              const char *propname,
                                              enum of_gpio_flags *flags);
```

بتشيك لو في property تانية في الـ DT بتحدد الـ polarity. مثلاً، الـ `fsl,imx6q-fec` بتستخدم `phy-reset-active-high` property لتحديد polarity الـ `phy-reset-gpios`. لو الـ property موجودة → active high، لو مش موجودة → active low.

---

#### `of_convert_gpio_flags()`

```c
static unsigned long of_convert_gpio_flags(enum of_gpio_flags flags);
```

بتحول الـ `enum of_gpio_flags` (OF-specific) لـ `unsigned long` bitmask من الـ `GPIO_*` flags (generic GPIO consumer flags). ده الـ bridge بين الـ OF layer والـ GPIO descriptor API.

| OF Flag | GPIO Flag |
|---|---|
| `OF_GPIO_ACTIVE_LOW` | `GPIO_ACTIVE_LOW` |
| `OF_GPIO_SINGLE_ENDED + OF_GPIO_OPEN_DRAIN` | `GPIO_OPEN_DRAIN` |
| `OF_GPIO_SINGLE_ENDED` (بدون open drain) | `GPIO_OPEN_SOURCE` |
| `OF_GPIO_TRANSITORY` | `GPIO_TRANSITORY` |
| `OF_GPIO_PULL_UP` | `GPIO_PULL_UP` |
| `OF_GPIO_PULL_DOWN` | `GPIO_PULL_DOWN` |

---

### Category 6: الـ Modern Consumer API (gpiod)

دي الـ functions الصح للاستخدام في الكود الجديد. بتقرأ الـ DT automatically عبر `gpiod_find_and_request()` اللي بتستدعي الـ OF layer.

#### `gpiod_get()` / `devm_gpiod_get()`

```c
struct gpio_desc *gpiod_get(struct device *dev,
                            const char *con_id,
                            enum gpiod_flags flags);

struct gpio_desc *devm_gpiod_get(struct device *dev,
                                 const char *con_id,
                                 enum gpiod_flags flags);
```

الـ main entry point لأي driver يريد GPIO من DT. بتبحث عن property اسمها `[con_id-]gpios` في الـ device node وبترجع `gpio_desc` جاهز.

**Parameters:**
- `dev` — الـ consumer device (بتاع الـ driver)
- `con_id` — اسم الـ function، مثلاً `"reset"` يبحث عن `reset-gpios`، أو NULL يبحث عن `gpios`
- `flags` — الحالة المطلوبة: `GPIOD_IN`، `GPIOD_OUT_HIGH`، `GPIOD_OUT_LOW`، `GPIOD_ASIS`

**Return:** `gpio_desc *` أو `ERR_PTR(errno)`. الـ `devm_` نسخة بتعمل release تلقائي لما الـ device بيتنفصل.

**Locking:** آمن للاستخدام في process context (probe).

```c
/* مثال صح — modern style */
struct gpio_desc *rst_gpio;

rst_gpio = devm_gpiod_get(&pdev->dev, "reset", GPIOD_OUT_HIGH);
if (IS_ERR(rst_gpio))
    return PTR_ERR(rst_gpio);

/* DTS المناسب: reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>; */
gpiod_set_value_cansleep(rst_gpio, 1); /* assert reset */
```

---

#### `gpiod_get_optional()` / `devm_gpiod_get_optional()`

```c
struct gpio_desc *gpiod_get_optional(struct device *dev,
                                     const char *con_id,
                                     enum gpiod_flags flags);
```

زي `gpiod_get()` بالظبط بس لو الـ property مش موجودة في DT بترجع NULL بدل error. مفيدة للـ GPIOs الاختيارية.

```c
/* لو الـ GPIO مش موجود في DT → rst_gpio = NULL → مش error */
rst_gpio = devm_gpiod_get_optional(&pdev->dev, "reset", GPIOD_OUT_HIGH);
if (IS_ERR(rst_gpio))
    return PTR_ERR(rst_gpio);
if (rst_gpio)
    gpiod_set_value_cansleep(rst_gpio, 1);
```

---

#### `gpiod_get_index()` / `devm_gpiod_get_index()`

```c
struct gpio_desc *gpiod_get_index(struct device *dev,
                                  const char *con_id,
                                  unsigned int idx,
                                  enum gpiod_flags flags);
```

نفس `gpiod_get()` بس مع index للـ arrays. مثلاً لو عندك `data-gpios = <&gpio1 12 0>, <&gpio1 13 0>` تقدر تجيب كل واحد بـ index.

**Parameters (الإضافي):**
- `idx` — رقم الـ GPIO في الـ property array (zero-based)

---

### Category 7: الـ gpio-ranges Binding Functions

#### الـ `gpio-ranges` Property

```
gpio-ranges = <&pinctrl1 gpio_offset pinctrl_offset count>;
```

الـ property دي بتعمل map بين الـ GPIO controller offsets والـ pin controller pins. الـ kernel بيستخدمها في `of_gpiochip_add_pin_range()` لتسجيل الـ ranges عبر `gpiochip_add_pin_range()`.

**Format التفصيلي:**
```
gpio-ranges = <[pinctrl phandle] [gpio_base] [pinctrl_base] [count]>;
```

- `gpio_base` — أول GPIO offset داخل الـ gpio controller
- `pinctrl_base` — أول pin number داخل الـ pin controller
- `count` — عدد الـ pins في الـ range

**مثال:**
```
gpio-ranges = <&foo 0 20 10>, <&bar 10 50 20>;
/* GPIO 0-9  → pinctrl foo pins 20-29 */
/* GPIO 10-29 → pinctrl bar pins 50-69 */
```

#### `gpio-ranges-group-names`

بدل الـ numerical ranges، ممكن تستخدم pin group names لما `pinctrl_base = 0` و`count = 0`:

```
gpio-ranges = <&pinctrl2 10 0 0>;
gpio-ranges-group-names = "foo";
/* GPIO 10+ → pin group "foo" في pinctrl2 */
```

الـ kernel بيجيب عدد الـ pins من الـ pin group definition نفسها مش من الـ DT.

---

### Category 8: الـ GPIO Hogging

#### الـ gpio-hog Mechanism

الـ **GPIO hogging** بيخلي الـ GPIO controller driver يطلب ويضبط GPIOs تلقائياً أثناء الـ probe — من غير ما أي consumer driver يطلبهم.

```dts
qe_pio_a: gpio-controller@1400 {
    gpio-controller;
    #gpio-cells = <2>;

    line_b-hog {          /* child node */
        gpio-hog;         /* علامة إن ده hog */
        gpios = <6 0>;    /* GPIO index 6, active high */
        output-low;       /* اتجاه: output، قيمة: low */
        line-name = "foo-bar-gpio";  /* label اختياري */
    };
};
```

**الـ Kernel Processing:**
الـ function `of_gpiochip_scan_gpios()` بتعدي على كل child nodes للـ gpio controller. لو لقت `gpio-hog` property، بتستدعي `of_gpiochip_add_hog()` اللي بتعمل:
1. `gpiod_get()` للـ GPIO
2. بتقرأ `input` / `output-low` / `output-high` بالترتيب ده (أول match بياخد)
3. بتضبط الـ direction والـ value تلقائياً
4. بتعمل label للـ line باسم `line-name` أو الـ node name

**الـ GPIO المحجوز مش ممكن أي driver تاني يطلبه** — بيبقى locked للـ hog للأبد.

---

### ملاحظات مهمة للـ Developer

1. **الـ `of_get_named_gpio()` deprecated** — استخدم `devm_gpiod_get()` في أي كود جديد.

2. **الـ `EPROBE_DEFER`:** لو الـ GPIO controller لسه مش probe، الـ kernel بيرجع `-EPROBE_DEFER` وبيعيد probe الـ consumer driver لاحقاً. ده سلوك صح ولازم تتعامل معاه.

3. **الـ Polarity:** الـ kernel بيعكس الـ GPIO value تلقائياً لو الـ flag `GPIO_ACTIVE_LOW` — استخدم `gpiod_set_value()` دايماً مع القيمة الـ logical (0/1) مش الـ physical.

4. **الـ `#gpio-cells` mismatch:** لو الـ DT فيه `#gpio-cells = <2>` بس الـ driver بيتوقع خلية واحدة، الـ `of_xlate_and_get_gpiod_flags()` هترجع `-EINVAL` فوراً.

5. **الـ `gpio-line-names`:** مش required بس highly recommended للـ debugging والـ userspace tools.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. debugfs — الـ Entries المهمة

الـ **debugfs** بيوفر نظرة مباشرة على حالة الـ GPIO subsystem في الـ kernel. Mount أول:

```bash
mount -t debugfs none /sys/kernel/debug
```

| Entry | المسار | الغرض |
|---|---|---|
| `gpio` | `/sys/kernel/debug/gpio` | قائمة بكل الـ GPIO chips والـ lines |
| `pinctrl` | `/sys/kernel/debug/pinctrl/` | حالة الـ pin mux وربطها بالـ GPIO ranges |
| `gpio-aggregator` | `/sys/kernel/debug/gpio-aggregator` | لو الـ aggregator driver محمّل |

```bash
# اقرأ حالة كل الـ GPIOs المسجلة في الـ kernel
cat /sys/kernel/debug/gpio
```

مثال للـ output:

```
gpiochip0: GPIOs 0-31, parent: platform/1400000.gpio, gpio-controller@1400000:
 gpio-6   (                    |foo-bar-gpio        ) out lo
 gpio-18  (                    |enable              ) out hi ACTIVE LOW

gpiochip1: GPIOs 32-63, parent: platform/1460000.gpio, gpio-controller@1460000:
 gpio-50  (                    |MMC-CD              ) in  hi IRQ
```

**التفسير:**
- `out lo` / `out hi` = الاتجاه والقيمة الحالية
- `ACTIVE LOW` = البين معرّف في الـ DT بـ `GPIO_ACTIVE_LOW`
- `in hi IRQ` = مدخل، مستوى عالي، مربوط بـ interrupt

```bash
# pinctrl: تأكد إن الـ GPIO ranges صح mapped
ls /sys/kernel/debug/pinctrl/
cat /sys/kernel/debug/pinctrl/<pinctrl-name>/gpio-ranges
```

---

#### 2. sysfs — الـ Entries المهمة

الـ **sysfs** GPIO interface القديم (legacy، deprecated في kernel حديث) وكمان الـ gpiochip interface:

```bash
# قائمة الـ GPIO chips
ls /sys/class/gpio/

# export GPIO يدوياً (legacy interface)
echo 18 > /sys/class/gpio/export
cat /sys/class/gpio/gpio18/direction
cat /sys/class/gpio/gpio18/value
cat /sys/class/gpio/gpio18/active_low

# معلومات الـ gpiochip نفسه
cat /sys/class/gpio/gpiochip0/label
cat /sys/class/gpio/gpiochip0/ngpio
cat /sys/class/gpio/gpiochip0/base
```

| Sysfs Path | المحتوى |
|---|---|
| `/sys/class/gpio/gpiochipN/label` | اسم الـ controller (من `gpio-line-names` أو الـ driver) |
| `/sys/class/gpio/gpiochipN/ngpio` | عدد الـ GPIO lines |
| `/sys/class/gpio/gpiochipN/base` | الـ global GPIO number الأول |
| `/sys/class/gpio/gpioN/direction` | `in` أو `out` |
| `/sys/class/gpio/gpioN/value` | `0` أو `1` |
| `/sys/class/gpio/gpioN/active_low` | `0` أو `1` |

**أداة gpioinfo (libgpiod — الطريقة الحديثة):**

```bash
# الأفضل: استخدم libgpiod بدل legacy sysfs
gpioinfo gpiochip0
gpiodetect
gpioget gpiochip0 18
gpiomon --num-events=5 gpiochip0 18
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

**الـ GPIO subsystem** عنده tracepoints مدمجة:

```bash
# شوف الـ GPIO events المتاحة
ls /sys/kernel/debug/tracing/events/gpio/

# الـ events الموجودة عادةً:
# gpio_direction   — لما يتغير اتجاه البين
# gpio_value       — لما تتغير قيمة البين
```

```bash
# فعّل الـ GPIO tracing كله
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe
```

```bash
# trace تغيير قيمة GPIO محدد فقط (باستخدام filter)
echo 'gpio=18' > /sys/kernel/debug/tracing/events/gpio/gpio_value/filter
echo 1 > /sys/kernel/debug/tracing/events/gpio/gpio_value/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

مثال للـ output:

```
     kworker/0:1-42    [000] .... 123.456789: gpio_value: 18 set value 1
          process-99   [000] .... 123.457001: gpio_direction: 18 set direction output
```

**trace الـ DT parsing** عند الـ boot:

```bash
# فعّل function tracing للـ of_get_named_gpio
echo 'of_get_named_gpio*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
dmesg | tail -20
cat /sys/kernel/debug/tracing/trace
```

---

#### 4. printk و dynamic debug

**فعّل dynamic debug للـ GPIO subsystem:**

```bash
# debug messages للـ gpiolib كله
echo 'file gpiolib.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file gpiolib-of.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file of.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو عن طريق module name
echo 'module gpio_generic +p' > /sys/kernel/debug/dynamic_debug/control

# اقرأ الـ dmesg في نفس الوقت
dmesg -w
```

**لو محتاج debug من الـ boot الأول** (قبل ما يبدأ الـ userspace)، ضيف في الـ kernel command line:

```
dyndbg="file gpiolib-of.c +p; file gpiolib.c +p"
```

أو في `/etc/modprobe.d/gpio-debug.conf`:

```
options <gpio-driver-module> dyndbg=+p
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|---|---|
| `CONFIG_DEBUG_GPIO` | يفعّل extra validation في الـ GPIO core — يكشف double-request وغلطات الـ direction |
| `CONFIG_GPIO_SYSFS` | يفعّل الـ sysfs interface للـ manual testing |
| `CONFIG_GPIOLIB_FASTPATH_LIMIT` | يتحكم في الـ fast path threshold |
| `CONFIG_DEBUG_FS` | لازم يكون مفعّل عشان debugfs يشتغل |
| `CONFIG_TRACING` | أساسي عشان ftrace يشتغل |
| `CONFIG_GPIO_AGGREGATOR` | لو بتستخدم gpio-aggregator |
| `CONFIG_PINCTRL` | لازم مفعّل لو بتستخدم `gpio-ranges` |
| `CONFIG_OF_GPIO` | الـ DT GPIO parsing — لازم مفعّل في أي SoC بيستخدم DT |
| `CONFIG_KALLSYMS` | يساعد في قراءة stack traces |
| `CONFIG_PROVE_LOCKING` | يكشف locking bugs في الـ GPIO callbacks |

```bash
# تأكد من الـ config المفعّل
zcat /proc/config.gz | grep -E 'GPIO|PINCTRL' | grep -v '^#'
```

---

#### 6. أدوات خاصة بالـ Subsystem

**libgpiod — الأداة الرسمية الحديثة:**

```bash
# اكتشف كل الـ chips
gpiodetect

# معلومات تفصيلية عن chip معين
gpioinfo gpiochip0

# اقرأ قيمة
gpioget gpiochip0 18

# اكتب قيمة
gpioset gpiochip0 18=1

# راقب التغييرات
gpiomon --num-events=10 --rising-edge gpiochip0 18

# find GPIO باسمه (من gpio-line-names في DT)
gpioinfo | grep "MMC-CD"
```

**gpio-mockup للـ unit testing:**

```bash
# حمّل الـ mock GPIO controller (للـ testing بدون hardware)
modprobe gpio-mockup gpio_mockup_ranges=0,8
ls /sys/class/gpio/  # هتلاقي gpiochipN جديد
cat /sys/kernel/debug/gpio
```

**Device Tree overlay لـ runtime testing:**

```bash
# تطبيق DT overlay يحتوي GPIO node جديد
dtoverlay <overlay-name>
# راقب الـ dmesg
dmesg | tail -20
```

---

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `GPIO-X: requested but not freed` أو `gpio_request: gpio-X is already requested` | الـ GPIO اتطلب مرتين بدون تحرير | تأكد إن الـ driver بيعمل `gpio_free()` أو `devm_gpio_free()` صح |
| `gpio-X (label): direction failed` | الـ hardware رفض تغيير الاتجاه | الـ pin ممكن يكون مقيّد بالـ pin controller، راجع `pinctrl` config |
| `of_get_named_gpio: can't parse gpios property` | خطأ في صياغة الـ DT property | تأكد من `#gpio-cells` وعدد الخلايا في الـ specifier |
| `GPIO chip: -EPROBE_DEFER` | الـ GPIO controller لسه مش متسجل لما جاء consumer يطلبه | ده طبيعي لو الـ ordering مش صح — الـ driver المستهلك هيعيد الـ probe |
| `gpiod_get: no GPIO consumer label found` | الـ DT property اسمها غلط أو مش موجودة | تأكد من الاسم `[name]-gpios` في الـ DT |
| `invalid GPIO X` | الـ GPIO number خارج الـ range | راجع `ngpios` و `gpio-reserved-ranges` في الـ DT |
| `gpio-ranges: failed to add pin range` | مشكلة في الـ mapping بين GPIO controller والـ pin controller | تأكد إن الـ phandle في `gpio-ranges` صح وإن الـ pinctrl driver محمّل |
| `Duplicate GPIO line name found` | اسمين متشابهين في `gpio-line-names` | خلّي كل اسم unique أو استخدم string فاضية `""` للـ lines غير المستخدمة |
| `GPIO hog: could not request GPIO` | الـ GPIO المطلوب في الـ hog node مش available | تأكد من الـ offset في `gpios` property صح وإنه مش reserved |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

ضيف في كود الـ driver في الأماكن دي لو بتحقق مشكلة:

```c
/* في دالة .probe() بعد gpiod_get */
struct gpio_desc *desc = devm_gpiod_get(dev, "enable", GPIOD_OUT_LOW);
if (IS_ERR(desc)) {
    dev_err(dev, "Failed to get enable GPIO: %ld\n", PTR_ERR(desc));
    /* dump_stack هنا يكشف من طلب الـ probe */
    dump_stack();
    return PTR_ERR(desc);
}

/* تأكد إن الـ descriptor مش NULL قبل الاستخدام */
WARN_ON(!desc);

/* في الـ IRQ handler لو الـ GPIO state مش متوقع */
int val = gpiod_get_value(desc);
WARN_ON(val < 0); /* قيمة سالبة = error */

/* في gpio_chip.direction_output callback */
int my_gpio_direction_output(struct gpio_chip *gc, unsigned int offset, int value)
{
    WARN_ON(offset >= gc->ngpio); /* offset خارج النطاق */
    /* ... */
}
```

---

### Hardware Level

---

#### 1. التحقق من حالة الـ Hardware مقارنةً بحالة الـ Kernel

```bash
# قارن قيمة الـ GPIO في الـ kernel مع قراءة فعلية من الـ hardware
cat /sys/kernel/debug/gpio | grep "gpio-18"
# gpio-18  (enable) out hi ACTIVE LOW

# معناه: الـ kernel قال output high، لكن لأنه ACTIVE LOW
# القيمة الفعلية على الـ pin = LOW (0V)

# تأكد بـ multimeter أو oscilloscope إن الـ pin فعلاً بـ 0V
```

**قارن الـ DT مع الـ hardware manual:**

```bash
# اقرأ الـ DT المفعّل حالياً
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 10 "gpio-controller"

# أو
cat /sys/firmware/devicetree/base/soc/gpio@1400/ngpios | xxd
```

---

#### 2. Register Dump Techniques

**باستخدام `devmem2`:**

```bash
# قرأة سجل الـ GPIO data register (مثال Freescale QE GPIO @ 0x1400)
devmem2 0x1400 w    # قرأة 32-bit word
devmem2 0x1404 w    # الـ direction register

# كتابة: set GPIO pin 6 high
devmem2 0x1400 w 0x02000000
```

**باستخدام `/dev/mem` مباشرة (احذر):**

```bash
# تأكد إن CONFIG_STRICT_DEVMEM=n أو استخدم /dev/mem مع cap
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 4096, offset=0x1400000)
    val = struct.unpack('<I', mm[:4])[0]
    print(f'GPIO data reg: 0x{val:08X}')
    mm.close()
"
```

**باستخدام `io` utility:**

```bash
# متاح في بعض الـ embedded distributions
io -4 -r 0x01400000    # قراءة 32-bit من عنوان الـ GPIO base
```

**مقارنة الـ register مع الـ DT:**

```bash
# الـ DT بيقول reg = <0x1400 0x18>؟
# يعني الـ base address = 0x1400، size = 0x18 (24 bytes)
# قرأ كل الـ registers الموجودة
for offset in 0 4 8 c 10 14 18; do
    devmem2 $((0x1400 + 0x$offset)) w
done
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**لـ GPIO toggling عالي التردد:**

```
- ربط probe الـ logic analyzer على الـ GPIO pin المشبوه
- شيل edge trigger على rising/falling
- قارن الـ timestamp مع الـ kernel log (loglevel=8)
- لو في jitter: فحص الـ interrupt latency
```

**لـ I2C/SPI GPIO expanders:**

```
- capture الـ I2C bus (SCL/SDA) مع الـ logic analyzer
- ابحث عن register writes إلى الـ GPIO expander address
- قارن مع ما الـ kernel يعتقده (debugfs gpio)
```

**لـ GPIO hog debugging:**

```
- الـ GPIO hogging بيحصل في probe time، قبل userspace
- ضيف scope probe قبل ما تبوت وشوف الـ initial state
- قارن مع output-low / output-high في الـ DT
```

**معلومة مهمة:** لو الـ `ACTIVE LOW` flag مضبوط في الـ DT والـ kernel بيقول `out hi`، القيمة الفعلية على الـ pin هي **0V** — الـ oscilloscope لازم يثبت ده.

---

#### 4. Hardware Issues الشائعة وأنماطها في الـ Kernel Log

| المشكلة | النمط في الـ dmesg |
|---|---|
| الـ GPIO controller مش موجود على الـ bus | `platform gpio@1400: probe failed: -ENXIO` |
| الـ GPIO pin مش موصول فعلاً (floating) | قراءات عشوائية — `gpio_value` trace بيقلب بدون سبب |
| الـ voltage level غلط (3.3V vs 1.8V) | `gpio_direction_output: failed` لو الـ IO cell مش بيقبل |
| الـ pin controller مش حارر الـ pin كـ GPIO | `pin X already requested` من الـ pinctrl subsystem |
| الـ pull-up/pull-down configuration غلط | الـ input بيقرأ دايماً 0 أو دايماً 1 |
| ngpios في الـ DT أصغر من الـ real hardware | `invalid GPIO X` لما تطلب pin برة النطاق |
| gpio-reserved-ranges غلط | الـ driver بيحاول يستخدم reserved pin، بتلاقي `GPIO-X is not usable` |

---

#### 5. Device Tree Debugging

**تحقق إن الـ DT المحمّل يطابق الـ hardware:**

```bash
# قرأة الـ compiled DT المحمّل حالياً (live DTB)
dtc -I fs -O dts /sys/firmware/devicetree/base 2>/dev/null > /tmp/live.dts

# دور على gpio-controller nodes
grep -A 20 "gpio-controller" /tmp/live.dts

# تأكد من #gpio-cells
grep "#gpio-cells" /tmp/live.dts

# تأكد من ngpios
grep "ngpios" /tmp/live.dts

# تأكد من gpio-reserved-ranges
grep "gpio-reserved-ranges" /tmp/live.dts
```

**تحقق من الـ gpio-line-names:**

```bash
# الأسماء المعرّفة في DT
cat /sys/kernel/debug/gpio | grep -v "^$"
# كل line عندها اسم؟ ولا فاضية؟

# باستخدام libgpiod
gpioinfo gpiochip0 | grep -v "unnamed"
```

**تحقق من الـ gpio-ranges (ربط GPIO بالـ pin controller):**

```bash
# شوف الـ GPIO ranges المسجلة
cat /sys/kernel/debug/pinctrl/<pinctrl>/gpio-ranges

# مثال للـ output:
# GPIO ranges handled:
#  0: 1400.gpio GPIOS [0 - 17] PINS [20 - 37]
```

**لو الـ DT property غلط** — الـ of_get_named_gpio هترجع error:

```bash
# فعّل dynamic debug لـ DT GPIO parsing
echo 'file gpiolib-of.c +p' > /sys/kernel/debug/dynamic_debug/control
dmesg -w &
# شغّل الـ driver من جديد
echo "<driver-name>" > /sys/bus/platform/drivers/<driver>/unbind
echo "<device-name>" > /sys/bus/platform/drivers/<driver>/bind
```

---

### Practical Commands

---

#### مجموعة أوامر جاهزة للنسخ

**1. نظرة عامة سريعة على الـ GPIO system:**

```bash
cat /sys/kernel/debug/gpio
```

Output مثال:
```
gpiochip0: GPIOs 0-17, parent: platform/1400.gpio, gpio@1400:
 gpio-6   (foo-bar-gpio       ) out lo
 gpio-12  (data0              ) out hi
 gpio-13  (data1              ) out lo
 gpio-18  [skipped - reserved]
```

**2. فحص كل الـ GPIO chips المتاحة:**

```bash
gpiodetect && echo "---" && gpioinfo
```

**3. تفعيل كامل لـ GPIO debugging:**

```bash
# script جاهز
#!/bin/bash
mount -t debugfs none /sys/kernel/debug 2>/dev/null
echo 'file gpiolib.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file gpiolib-of.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 1 > /sys/kernel/debug/tracing/events/gpio/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on
echo "GPIO debugging enabled. Run: cat /sys/kernel/debug/tracing/trace_pipe"
```

**4. مراقبة GPIO في real-time:**

```bash
# الطريقة الحديثة (libgpiod)
gpiomon --num-events=20 --format="%e %o %s %n" gpiochip0 18

# Output مثال:
# 1 18 1709123456 123456789   (rising edge, gpio 18, timestamp)
```

**5. تتبع من يطلب GPIO معين:**

```bash
# فعّل tracing على gpiod_request
echo 'gpiod_request' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل الـ driver
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | head -50
```

**6. تحقق من الـ DT gpio specifier:**

```bash
# استخراج الـ gpio specifier من الـ live DT
fdtget /sys/firmware/fdt /soc/mydevice@0 enable-gpios
# Output: 18 0  (gpio 18, active high)
# أو
# Output: 18 1  (gpio 18, active low)
```

**7. اختبار الـ gpio-reserved-ranges:**

```bash
# حاول تصدّر GPIO محجوز — المفروض يفشل
echo 5 > /sys/class/gpio/export
# Expected: -bash: echo: write error: Device or resource busy
dmesg | tail -3
# gpio-5: reserved by device tree
```

**8. مقارنة شاملة DT vs hardware:**

```bash
#!/bin/bash
echo "=== Live DT GPIO Controllers ==="
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    grep -B2 -A15 "gpio-controller;"

echo "=== Kernel GPIO State ==="
cat /sys/kernel/debug/gpio

echo "=== Pin Controller GPIO Ranges ==="
for f in /sys/kernel/debug/pinctrl/*/gpio-ranges; do
    echo "--- $f ---"
    cat "$f" 2>/dev/null
done
```

**9. إعادة تشغيل driver مع كامل الـ debug:**

```bash
DRIVER="gpio-pl061"  # غيّره لاسم الـ driver بتاعك
DEVICE="1c02000.gpio"

echo 'file gpiolib*.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo "$DEVICE" > /sys/bus/platform/drivers/$DRIVER/unbind 2>/dev/null
echo "$DEVICE" > /sys/bus/platform/drivers/$DRIVER/bind
dmesg | tail -30
```

**10. فحص الـ interrupt mapping للـ GPIO:**

```bash
# شوف الـ GPIOs المربوطة بـ interrupts
cat /proc/interrupts | grep gpio
# أو
cat /sys/kernel/debug/gpio | grep IRQ
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — GPIO اتجاهه اتعكس

#### العنوان
**LED الـ status بيضيء دايمًا وملوش علاقة بالـ firmware state**

#### السياق
بتشتغل على industrial gateway بيستخدم **RK3562** — الـ gateway ده بيتحكم في sensors صناعية وبيبعت بياناتها للـ cloud. في الـ board، فيه status LED أحمر المفروض يضيء لما يكون في error، وLED أخضر يضيء لما يكون كل حاجة تمام. الـ hardware team قالوا إن الـ LEDs مربوطة على GPIO2_A3 وGPIO2_A4.

#### المشكلة
بعد boot، الـ LED الأحمر بيضيء دايمًا حتى لما مافيش أي error. الـ firmware بيطلب `gpiod_set_value(led_err, 0)` بس الـ LED مش بيطفى.

#### التحليل
الأول، نشوف الـ DTS:

```dts
/* DTS الأصلي — فيه خطأ */
leds {
    compatible = "gpio-leds";

    error-led {
        /* GPIO2_A3 = bank 2, pin 3 */
        gpios = <&gpio2 3 GPIO_ACTIVE_HIGH>;
        default-state = "off";
    };
};
```

الـ binding بيقول إن الـ `gpio-specifier` الـ flag cell بتحدد الـ polarity. هنا استخدمنا `GPIO_ACTIVE_HIGH` يعني 0. لكن لما راحنا نشوف الـ schematic، لقينا إن الـ LED مربوط بين الـ GPIO pin والـ VCC — يعني الـ LED **active-low** من ناحية الـ GPIO controller.

الـ gpio.txt بيقول صراحة:

> *"Bit 0: 0 means active high, 1 means active low"*

يعني لما بتكتب `GPIO_ACTIVE_HIGH`، الـ kernel بيعتبر إن pull-up = logic 1 = LED on. لكن لأن الـ hardware عكسي، الـ LED بيضيء لما الـ GPIO = 0 (اللي الـ kernel شايفه "off").

```
GPIO pin ──────────────── Anode(LED) ──── Cathode ──── VCC
             (active-low wiring)
```

الـ kernel لما بيعمل `gpiod_set_value(led, 0)` — بيدي signal low — اللي بيحول الـ current ويضيء الـ LED رغم إننا قلنا "off".

#### الحل

```dts
/* DTS المصحح */
leds {
    compatible = "gpio-leds";

    error-led {
        /* GPIO_ACTIVE_LOW = 1 in flags cell */
        gpios = <&gpio2 3 GPIO_ACTIVE_LOW>;
        default-state = "off";
    };

    ok-led {
        gpios = <&gpio2 4 GPIO_ACTIVE_LOW>;
        default-state = "on";
    };
};
```

أو بالـ raw value بدل الـ macro:

```dts
/* equivalent: GPIO_ACTIVE_LOW = BIT(0) = 1 */
gpios = <&gpio2 3 1>;
```

للـ verification من الـ userspace:

```bash
# قرأ الـ line info
gpioinfo gpiochip2 | grep -E "line 3|line 4"

# اختبار مباشر
gpioset gpiochip2 3=0   # المفروض يطفى الـ LED
gpioset gpiochip2 3=1   # المفروض يضيء
```

#### الدرس المستفاد
الـ `GPIO_ACTIVE_LOW` flag في الـ DTS مش مجرد تعليق توضيحي — الـ kernel بيعكس الـ logic تلقائيًا. لازم دايمًا تتأكد من الـ schematic وتحدد الـ polarity من ناحية الـ GPIO controller، مش من ناحية الـ device.

---

### السيناريو 2: Android TV Box على Allwinner H616 — ngpios غلط بيعمل panic

#### العنوان
**`kernel BUG at drivers/gpio/gpiolib.c` عند probe الـ Wi-Fi module**

#### السياق
بتعمل bring-up لـ Android TV box رخيصة بتستخدم **Allwinner H616**. الـ Wi-Fi chip هو XR829 مربوط بـ SDIO، ومحتاج GPIO للـ reset والـ power enable. الـ SoC ده فيه GPIO controller واحد فيه 4 banks (A/C/F/G/H) والـ driver بيعمل enumerate ليهم كـ single controller.

#### المشكلة

```
[    3.421] sunxi-pinctrl 300b000.pinctrl: initialized OK
[    3.589] BUG: KASAN: out-of-bounds in gpiochip_add_data_with_key+0x...
[    3.590] gpio gpiochip0: requested GPIO 220 is out of range [0, 192)
```

الـ panic بييجي لما الـ Wi-Fi driver بيطلب GPIO 220 للـ reset.

#### التحليل

الـ H616 GPIO controller عنده:
- Port A: 16 pins (0..15)
- Port C: 17 pins (96..112)
- Port F: 7 pins (160..166)
- Port G: 14 pins (192..205)
- Port H: 19 pins (224..242)

الـ DTS الأصلي كان:

```dts
/* DTS غلط */
pio: pinctrl@300b000 {
    compatible = "allwinner,sun50i-h616-pinctrl";
    reg = <0x0300b000 0x400>;
    gpio-controller;
    #gpio-cells = <3>;
    ngpios = <192>;  /* ← المشكلة هنا */
    /* ... */
};
```

الـ gpio.txt بيقول:

> *"setting 'ngpios = <18>;' informs the driver that only the first 18 GPIOs, at local offset 0 .. 17, are in use"*

الـ `ngpios = <192>` بيقول للـ driver إن آخر GPIO offset هو 191. لما الـ Wi-Fi driver بيطلب GPIO 220 (Port H pin 0 = offset 224)، الـ kernel بيرفض لأنه خارج الـ range المعلنة.

المشكلة الأصلية: الـ engineer حسب عدد الـ GPIOs في أول 3 ports بس (A+C+F = تقريبًا 192) ونسي الـ G و H.

الحل الصح: مش كل ports متصلة، فـ ports معينة فيها gaps. الـ binding المناسب هو `gpio-reserved-ranges` مش تقليل `ngpios`.

#### الحل

```dts
pio: pinctrl@300b000 {
    compatible = "allwinner,sun50i-h616-pinctrl";
    reg = <0x0300b000 0x400>;
    gpio-controller;
    #gpio-cells = <3>;

    /* الـ total range يغطي كل الـ ports */
    ngpios = <256>;

    /*
     * gpio-reserved-ranges: exclude gaps between ports
     * Port A ends at 15, Port C starts at 96 → gap 16..95
     * Port C ends at 112, Port F starts at 160 → gap 113..159
     * Port F ends at 166, Port G starts at 192 → gap 167..191
     * Port G ends at 205, Port H starts at 224 → gap 206..223
     */
    gpio-reserved-ranges =
        <16 80>,    /* gap A→C */
        <113 47>,   /* gap C→F */
        <167 25>,   /* gap F→G */
        <206 18>;   /* gap G→H */
};

/* Wi-Fi reset على Port H pin 0 = offset 224 */
wifi_pwrseq: pwrseq {
    compatible = "mmc-pwrseq-simple";
    reset-gpios = <&pio 7 0 GPIO_ACTIVE_LOW>; /* PH0 */
};
```

```bash
# تحقق بعد fix
cat /sys/kernel/debug/gpio | grep -A5 "gpiochip0"
gpioinfo | grep "line 224"
```

#### الدرس المستفاد
الـ `ngpios` بيحدد الـ upper bound للـ valid GPIO numbers. لو عندك SoC بـ non-contiguous GPIO banks، استخدم `ngpios` بأكبر offset + 1، وحدد الـ gaps بـ `gpio-reserved-ranges` بدل ما تقلل `ngpios` وتحجب ports كاملة.

---

### السيناريو 3: IoT Sensor على STM32MP1 — GPIO hog بيمنع driver من الشغل

#### العنوان
**SPI sensor مش بيشتغل رغم إن الـ wiring صح**

#### السياق
بتبني IoT sensor node بيستخدم **STM32MP157C** مربوط بـ BME280 sensor عن طريق SPI. الـ sensor محتاج chip-select (CS) وpin إضافي للـ power enable. الـ board engineer قرر يستخدم **GPIO hog** عشان يضمن إن الـ power enable يتفعل تلقائيًا عند الـ boot قبل ما أي driver يحاول يكلم الـ sensor.

#### المشكلة

```
[    4.102] spi-stm32 44004000.spi: spi_device spi0.0 probe failed
[    4.103] bme280: Failed to read chip ID, ret=-EIO
```

الـ SPI communication فاشلة تمامًا.

#### التحليل

الـ DTS كان فيه:

```dts
&gpioe {
    gpio-controller;
    #gpio-cells = <2>;

    /* hog لتفعيل power */
    sensor-power-hog {
        gpio-hog;
        gpios = <5 GPIO_ACTIVE_HIGH>;  /* PE5 = power enable */
        output-high;
        line-name = "sensor-pwr-en";
    };

    /* hog للـ CS — وده هو المشكلة */
    sensor-cs-hog {
        gpio-hog;
        gpios = <4 GPIO_ACTIVE_LOW>;   /* PE4 = SPI CS */
        output-high;                    /* CS idle high = deasserted */
        line-name = "sensor-cs";
    };
};

&spi1 {
    bme280@0 {
        compatible = "bosch,bme280";
        reg = <0>;
        spi-max-frequency = <10000000>;
        cs-gpios = <&gpioe 4 GPIO_ACTIVE_LOW>;
    };
};
```

الـ gpio.txt بيقول إن الـ GPIO hog بيعمل **automatic GPIO request** جزء من الـ gpio-controller probe. يعني لما الـ gpio-controller اتعمل probe، هو طلب PE4 لنفسه كـ hog.

لما الـ SPI driver حاول يطلب نفس الـ GPIO (PE4) من خلال `cs-gpios`، لقى إن الـ GPIO اتحجز بالفعل من الـ hog. الـ `gpio_request()` بترجع `-EBUSY`.

الـ gpio.txt بيوضح إن الـ hog بيحجز الـ GPIO نهائيًا ضمن الـ controller — مش بيتحرر.

#### الحل

الحل إن الـ CS مفيش سبب تعمله hog — الـ SPI controller هو اللي لازم يتحكم فيه. شيل الـ CS hog وسيب power enable hog بس:

```dts
&gpioe {
    gpio-controller;
    #gpio-cells = <2>;

    /* هوج للـ power enable بس — مش للـ CS */
    sensor-power-hog {
        gpio-hog;
        gpios = <5 GPIO_ACTIVE_HIGH>;
        output-high;
        line-name = "sensor-pwr-en";
    };
    /* sensor-cs-hog اتشال — الـ SPI controller يتحكم في CS */
};

&spi1 {
    pinctrl-names = "default";
    pinctrl-0 = <&spi1_pins>;
    cs-gpios = <&gpioe 4 GPIO_ACTIVE_LOW>;

    bme280@0 {
        compatible = "bosch,bme280";
        reg = <0>;
        spi-max-frequency = <10000000>;
    };
};
```

للـ debug:

```bash
# شوف إيه الـ GPIOs المحجوزة كـ hogs
cat /sys/kernel/debug/gpio | grep hog

# أو بـ gpioinfo
gpioinfo gpiochip4 | grep "sensor"
```

#### الدرس المستفاد
الـ GPIO hog مناسب للـ signals اللي مش بيتحكم فيها أي driver — زي الـ power rails أو الـ hardware enable pins. لو الـ GPIO لازم يتحكم فيه driver زي SPI أو I2C، متعملهوش hog أبدًا لأن الـ hog بيحجزه للأبد.

---

### السيناريو 4: Automotive ECU على i.MX8MP — gpio-ranges خاطئة بتعطل pinmux

#### العنوان
**CAN bus مش بيشتغل رغم صح الـ pinctrl configuration**

#### السياق
بتعمل bring-up لـ automotive ECU بيستخدم **i.MX8M Plus** في نظام ADAS (Advanced Driver Assistance). الـ ECU محتاج CAN FD للتواصل مع الـ sensors. الـ CAN controller محتاج GPIO للـ transceiver enable، والـ pin ده (SAI1_RXD0 = GPIO4_IO2) محتاج pinmux من الـ pinctrl.

#### المشكلة

```
[    5.234] flexcan 308c0000.can: unable to get irq
[    5.235] imx_pinctrl 30330000.pinctrl: pin SAI1_RXD0 already used by sai1
```

الـ pinctrl بيرفض يعمل mux للـ pin للـ GPIO function.

#### التحليل

الـ DTS كان فيه:

```dts
&iomuxc {
    pinctrl_can_en: can_en_grp {
        fsl,pins = <
            /* SAI1_RXD0 → GPIO4 IO2 */
            MX8MP_IOMUXC_SAI1_RXD0__GPIO4_IO02  0x140
        >;
    };
};

&gpio4 {
    gpio-controller;
    #gpio-cells = <2>;
    /* gpio-ranges خاطئة */
    gpio-ranges = <&iomuxc 0 84 32>;  /* ← offset 84 في الـ pinctrl */
};
```

الـ gpio.txt بيشرح:

> *"The format is: `<[pin controller phandle], [GPIO controller offset], [pin controller offset], [number of pins]>`"*

المشكلة: الـ `gpio-ranges = <&iomuxc 0 84 32>` بتقول إن GPIO4 pin 0 يقابل pinctrl pin 84. لكن فعليًا الـ SAI1_RXD0 موجود في pinctrl offset مختلف.

لما الـ GPIO driver بيطلب pin عن طريق `pinctrl_gpio_request()` بيحسب الـ pinctrl offset غلط، فالـ pinctrl بيرفض لأن الـ pin محجوز بالفعل من SAI driver بأوفست مختلف.

للتحقق:

```bash
# شوف الـ actual pin offsets في الـ i.MX8MP pinctrl
cat /sys/kernel/debug/pinctrl/30330000.pinctrl/pins | grep SAI1_RXD0
# Output: pin 110 (SAI1_RXD0)
```

يعني الـ offset الصح هو 110، مش 84.

#### الحل

```dts
&gpio4 {
    gpio-controller;
    #gpio-cells = <2>;
    /*
     * gpio-ranges صحيحة:
     * GPIO4 pin 0 يقابل iomuxc pin 106
     * (i.MX8MP: GPIO4 starts at pinctrl offset 106)
     */
    gpio-ranges = <&iomuxc 0 106 32>;
};
```

بعد الـ fix، الـ pinctrl يعرف يعمل mux صح:

```bash
# تحقق من الـ mapping
cat /sys/kernel/debug/pinctrl/30330000.pinctrl/gpio-ranges

# اختبر الـ CAN transceiver enable
gpioget gpiochip3 2

# شوف الـ pin state
cat /sys/kernel/debug/pinctrl/30330000.pinctrl/pingroups | grep can
```

#### الدرس المستفاد
الـ `gpio-ranges` مش مجرد documentation — الـ kernel بيستخدمها لحساب الـ pinctrl offset لما بيطلب GPIO pin. offset غلط بيوصل لـ pin conflict أو silent malfunction. دايمًا تحقق من الـ SoC Reference Manual وقارن بـ `debugfs/pinctrl/pins`.

---

### السيناريو 5: Custom Board على AM62x — gpio-line-names بتسبب confusion في production

#### العنوان
**Script الـ factory test بيفصل الـ wrong relay في production line**

#### السياق
بتعمل custom board على **TI AM625** لـ industrial control panel. الـ board فيها 8 relays مربوطة على GPIO1 pins 0..7. في الـ factory، فيه Python script بيستخدم `libgpiod` لاختبار كل relay عن طريق الـ line name.

#### المشكلة

```python
# factory_test.py
import gpiod

chip = gpiod.Chip("gpiochip1")
relay3 = chip.find_line("RELAY-3")
relay3.request(consumer="factory-test", type=gpiod.LINE_REQ_DIR_OUT)
relay3.set_value(1)  # المفروض يفعل relay 3
```

الـ script بيشتغل بدون errors، لكن في الـ production line لقوا إن RELAY-4 هو اللي بيتفعل مش RELAY-3.

#### التحليل

الـ DTS كان:

```dts
&gpio1 {
    gpio-controller;
    #gpio-cells = <2>;
    ngpios = <92>;

    gpio-line-names =
        "RELAY-1",    /* line 0 */
        "RELAY-2",    /* line 1 */
        "RELAY-3",    /* line 2 */
        "RELAY-4",    /* line 3 */
        "RELAY-5",    /* line 4 */
        "RELAY-6",    /* line 5 */
        "RELAY-7",    /* line 6 */
        "RELAY-8";    /* line 7 */
};
```

الـ gpio.txt بيقول:

> *"The names are assigned starting from line offset 0, from left to right, from the passed array."*

يعني "RELAY-1" هو line 0، "RELAY-3" هو line 2.

المشكلة: الـ hardware engineer وصل الـ relays على الـ board بدءًا من pin 1 مش pin 0. يعني:
- GPIO1[0] = مش موصل
- GPIO1[1] = RELAY-1 الفعلي
- GPIO1[2] = RELAY-2 الفعلي
- GPIO1[3] = RELAY-3 الفعلي

لما الـ script طلب `RELAY-3` (اللي الـ DTS حطه على line 2)، فعليًا فعّل GPIO1[2] اللي هو RELAY-2 الفعلي على الـ hardware. كل اسم متأخر بـ offset 1.

#### الحل

الأول، تحقق من الـ actual wiring بـ oscilloscope أو بـ visual inspection، بعدين صحح الـ DTS:

```dts
&gpio1 {
    gpio-controller;
    #gpio-cells = <2>;
    ngpios = <92>;

    gpio-line-names =
        "",           /* line 0: unconnected — use blank string */
        "RELAY-1",    /* line 1: first relay */
        "RELAY-2",    /* line 2 */
        "RELAY-3",    /* line 3 */
        "RELAY-4",    /* line 4 */
        "RELAY-5",    /* line 5 */
        "RELAY-6",    /* line 6 */
        "RELAY-7",    /* line 7 */
        "RELAY-8";    /* line 8 */
};
```

الـ gpio.txt نفسه بيقول:

> *"rather use the '' (blank string) if the use of the GPIO line is undefined in your design"*

للـ verification قبل production:

```bash
# تحقق من الـ line names
gpioinfo gpiochip1 | head -20

# اختبر كل relay وشوف اللي بيتفعل فعليًا
for i in $(seq 1 8); do
    echo "Testing RELAY-$i"
    gpioset --mode=time --sec=1 gpiochip1 $(gpioinfo gpiochip1 | grep "RELAY-$i" | awk '{print NR-1}')=1
    read -p "Press enter after verifying..."
done
```

للـ Python script الصح:

```python
import gpiod

chip = gpiod.Chip("gpiochip1")
lines = chip.get_all_lines()

# print all line names for verification
for line in lines:
    if line.name():
        print(f"Line {line.offset()}: {line.name()}")
```

#### الدرس المستفاد
الـ `gpio-line-names` بتبدأ من offset 0 دايمًا. لو الـ first GPIO pin مش موصل أو reserved، لازم تحط string فاضي `""` في البداية عشان تخلي الـ indices صح. أي غلطة في الـ naming بتأثر على أي code بيحدد الـ GPIO باسمه — سواء scripts أو userspace apps أو kernel consumers.
## Phase 7: مصادر ومراجع

---

### مصادر رسمية — توثيق الـ Kernel

| المصدر | الوصف |
|--------|-------|
| [`Documentation/devicetree/bindings/gpio/gpio.txt`](https://www.kernel.org/doc/Documentation/devicetree/bindings/gpio/gpio.txt) | الملف الرئيسي لـ bindings الـ GPIO في الـ device tree — المصدر الأساسي |
| [`Documentation/driver-api/gpio/`](https://www.kernel.org/doc/html/latest/driver-api/gpio/index.html) | دليل الـ GPIO API الكامل بكل أجزاءه |
| [`Documentation/driver-api/gpio/board.html`](https://www.kernel.org/doc/html/latest/driver-api/gpio/board.html) | شرح تفصيلي لـ GPIO Mappings في الـ device tree والـ platform data |
| [`Documentation/driver-api/gpio/consumer.html`](https://docs.kernel.org/driver-api/gpio/consumer.html) | الـ GPIO Descriptor Consumer Interface — كيف يستخدم الـ driver الـ GPIO |
| [`Documentation/driver-api/gpio/driver.html`](https://static.lwn.net/kerneldoc/driver-api/gpio/driver.html) | كيفية كتابة الـ GPIO controller driver |
| [`Documentation/driver-api/gpio/legacy.html`](https://static.lwn.net/kerneldoc/driver-api/gpio/legacy.html) | الواجهة القديمة القائمة على الأرقام — legacy integer-based interface |
| [`include/dt-bindings/gpio/gpio.h`](https://github.com/torvalds/linux/blob/master/include/dt-bindings/gpio/gpio.h) | الـ macros الرسمية: `GPIO_ACTIVE_HIGH`, `GPIO_ACTIVE_LOW`, `GPIO_OPEN_DRAIN`, إلخ |
| [`Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt`](https://www.kernel.org/doc/Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt) | الـ bindings الخاصة بالـ pin controller المرتبطة بـ `gpio-ranges` |

---

### مقالات LWN.net

الـ LWN هو المصدر الأهم لفهم تطور الـ GPIO subsystem في الـ kernel.

#### سلسلة الـ GPIO الأساسية

- **[GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/)**
  مقدمة شاملة لنظام الـ GPIO في الـ kernel — نقطة البداية المثالية لأي مطور.

- **[GPIO in the kernel: future directions](https://lwn.net/Articles/533632/)**
  تناقش الاتجاهات المستقبلية للـ GPIO API وسبب الانتقال من الـ integer-based إلى الـ descriptor-based.

#### الـ Descriptor-based Interface (gpiod)

- **[gpio: introduce descriptor-based interface](https://lwn.net/Articles/531848/)**
  الـ patch الأصلي الذي قدّم الـ `struct gpio_desc *` كبديل للأرقام العشوائية.

- **[gpiolib: introduce descriptor-based GPIO interface](https://lwn.net/Articles/528226/)**
  النسخة الأولى من الـ patch series الذي أعاد تصميم الـ gpiolib من الأساس.

- **[New descriptor-based GPIO interface](https://lwn.net/Articles/565662/)**
  ملخص التغييرات النهائية قبل الدمج في الـ mainline kernel.

- **[Documentation: gpiolib: document new interface](https://lwn.net/Articles/574055/)**
  الـ patch الخاص بتوثيق الواجهة الجديدة رسمياً في `Documentation/`.

#### الـ GPIO Hogging

- **[gpio: Document GPIO hogging mechanism](https://lwn.net/Articles/641117/)**
  تقديم آلية الـ GPIO hog في الـ device tree — تفسر الـ `gpio-hog` property الموثقة في `gpio.txt`.

#### الـ Device Tree العام

- **[Device trees II: The harder parts](https://lwn.net/Articles/573409/)**
  يشرح الجوانب المعقدة في الـ device tree بما فيها الـ GPIO ranges والـ pinctrl interaction.

- **[An alternative device-tree source language](https://lwn.net/Articles/730217/)**
  نقاش حول تطوير صياغة الـ DTS.

---

### نقاشات الـ Mailing List

- **[Using gpio from device tree on platform devices](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html)**
  نقاش عملي على قائمة kernelnewbies حول استخدام الـ GPIO من الـ device tree في الـ platform devices.

- **[gpio-mcp23s08 driver with multiple chips](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-June/016408.html)**
  مثال واقعي على مشاكل الـ GPIO controller driver مع الـ device tree bindings.

- **[devm_gpiod_get usage to get the gpio num](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html)**
  نقاش حول استخدام `devm_gpiod_get()` مع الـ device tree.

---

### Commits مهمة في الـ Kernel

| الميزة | الـ Commit / الوصف |
|--------|-------------------|
| **gpio-ranges** | إضافة الـ `gpio-ranges` property لربط الـ GPIO بالـ pin controller — قُدِّمت مع دمج gpiolib وpinctrl subsystems |
| **gpio-hog** | إضافة آلية الـ GPIO hogging التلقائية — موثقة في [lwn.net/Articles/641117](https://lwn.net/Articles/641117/) |
| **gpiod descriptor API** | الـ patch series في kernel 3.13 — راجع [lwn.net/Articles/531848](https://lwn.net/Articles/531848/) |
| **gpio-line-names** | إضافة تسمية الـ GPIO lines في الـ device tree لدعم `libgpiod` |
| **gpio-reserved-ranges** | إضافة property لتحديد الـ GPIO lines غير الصالحة للاستخدام |

للبحث عن commits بعينها:
```bash
git log --oneline --all -- Documentation/devicetree/bindings/gpio/gpio.txt
git log --oneline --grep="gpio-hog" --grep="gpio-ranges" --all-match
```

---

### مصادر eLinux.org

- **[Device Tree Reference](https://elinux.org/Device_Tree_Reference)**
  مرجع شامل للـ device tree يغطي الـ GPIO وgpios property والـ pinctrl interaction.

- **[Device Tree Mysteries](https://elinux.org/Device_Tree_Mysteries)**
  شرح تفصيلي لخصائص الـ device tree الغامضة بما فيها `#gpio-cells` وصياغة الـ gpio-specifier.

- **[EBC Device Trees](https://elinux.org/EBC_Device_Trees)**
  دليل عملي على BeagleBone لاستخدام الـ device tree overlays مع الـ GPIO وضبط الـ pull-up/pull-down.

---

### مصادر kernelnewbies.org

- **[LinuxChanges](https://kernelnewbies.org/LinuxChanges)**
  تتبع تغييرات الـ GPIO عبر إصدارات الـ kernel المختلفة.

- **[Linux_6.13 Changes](https://kernelnewbies.org/Linux_6.13)**
  آخر التغييرات في الـ GPIO subsystem في الإصدارات الحديثة.

- **[Linux_6.2 Changes](https://kernelnewbies.org/Linux_6.2)**
  تغييرات الـ GPIO في kernel 6.2 بما فيها إضافات الـ GPIO drivers الجديدة.

---

### توثيق خارجي مفيد

- **[GPIO device tree configuration — STM32MPU](https://wiki.st.com/stm32mpu/wiki/GPIO_device_tree_configuration)**
  مثال واقعي على STM32 يوضح استخدام كامل الـ GPIO bindings مع `gpio-ranges` و`gpio-hog` في منتج حقيقي.

- **[GPIO.txt — Android Kernel (Tegra)](https://android.googlesource.com/kernel/tegra/+/d543254f2888fef055e84715238009fa2500f501/Documentation/devicetree/bindings/gpio/gpio.txt)**
  نسخة الـ file في NVIDIA Tegra kernel — مثال على vendor extensions فوق الـ upstream bindings.

---

### كتب مرجعية

#### Linux Device Drivers (LDD3)
**الكتاب**: *Linux Device Drivers, 3rd Edition* — Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman

الـ LDD3 متاح مجاناً على: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

| الفصل | الصلة بالموضوع |
|-------|----------------|
| Chapter 1: An Introduction to Device Drivers | أساسيات الـ driver model |
| Chapter 14: The Linux Device Model | الـ kobject، الـ bus، الـ device — أساس فهم الـ GPIO subsystem |

> **ملاحظة**: الـ LDD3 صدر عام 2005 وبالتالي لا يغطي الـ descriptor-based GPIO API (gpiod) الذي ظهر في kernel 3.13. استخدمه للمبادئ العامة وليس الـ GPIO API الحديث.

#### Linux Kernel Development (Robert Love)
**الكتاب**: *Linux Kernel Development, 3rd Edition* — Robert Love (2010)

| الفصل | الصلة بالموضوع |
|-------|----------------|
| Chapter 14: The Block I/O Layer | نماذج الـ subsystems المشابهة |
| Chapter 17: Devices and Modules | كيفية تسجيل الـ devices وعلاقتها بالـ device tree |

#### Embedded Linux Primer (Christopher Hallinan)
**الكتاب**: *Embedded Linux Primer, 2nd Edition* — Christopher Hallinan

| الفصل | الصلة بالموضوع |
|-------|----------------|
| Chapter 16: Kernel Initialization | كيف يُعالَج الـ device tree عند الـ boot |
| Chapter 17: Linux BSP | الـ board support وعلاقة الـ GPIO بالـ hardware platform |

#### Mastering Embedded Linux Programming
**الكتاب**: *Mastering Embedded Linux Programming, 3rd Edition* — Frank Vasquez, Chris Simmonds

يغطي الـ `libgpiod` والتفاعل مع الـ device tree GPIO bindings من منظور المبرمج.

---

### مصطلحات البحث

للعثور على معلومات إضافية استخدم هذه المصطلحات:

```
gpio-controller device tree binding linux kernel
gpiod_get descriptor interface linux
gpio-ranges pinctrl linux device tree
gpio-hog auto-request device tree
#gpio-cells specifier linux dts
gpio-reserved-ranges ngpios linux
GPIO_ACTIVE_LOW GPIO_OPEN_DRAIN dt-bindings
gpio-line-names libgpiod linux
devm_gpiod_get linux driver example
gpio polarity active-low device tree best practice
```

---

### روابط سريعة — مرجع دائم

```
Official kernel docs:  https://www.kernel.org/doc/html/latest/driver-api/gpio/
gpio.txt source:       Documentation/devicetree/bindings/gpio/gpio.txt
dt-bindings header:    include/dt-bindings/gpio/gpio.h
gpio core:             drivers/gpio/gpiolib.c
gpio OF support:       drivers/gpio/gpiolib-of.c
gpio sysfs:            drivers/gpio/gpiolib-sysfs.c
```
## Phase 8: Writing simple module

### الفكرة

الـ `gpio.txt` بتوضح إن كل GPIO controller في الـ Device Tree لازم يتسجل في الـ kernel عبر `gpiochip_add_data()` — دي النقطة اللي بيتحول فيها الـ DT binding من مجرد نص لـ hardware حقيقي شغال. هنعمل kprobe على `gpiochip_add_data` عشان نتلصص على كل GPIO controller بيتسجل — نطبع اسمه، عدد الـ lines، والـ base number.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * gpio_dt_probe_spy.c
 *
 * kprobe on gpiochip_add_data() to observe GPIO controllers
 * being registered — the exact moment a DT gpio-controller node
 * becomes a live kernel object.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* kprobe API */
#include <linux/gpio/driver.h> /* struct gpio_chip */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("GPIO DT Spy");
MODULE_DESCRIPTION("kprobe on gpiochip_add_data: watch DT gpio-controller nodes come alive");

/* ------------------------------------------------------------------ */
/* kprobe handler — called just before gpiochip_add_data() executes   */
/* ------------------------------------------------------------------ */

/*
 * gpiochip_add_data signature (kernel >= 5.x):
 *   int gpiochip_add_data(struct gpio_chip *gc, void *data)
 *
 * pt_regs on x86-64:
 *   regs->di = first arg  → struct gpio_chip *gc
 *   regs->si = second arg → void *data  (not used here)
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Cast first argument to gpio_chip pointer */
    struct gpio_chip *gc = (struct gpio_chip *)regs->di;

    if (!gc)
        return 0;

    /*
     * gc->label  — the name set by the driver (matches DT compatible or
     *              explicit label), same string you'd see in /sys/class/gpio/
     * gc->ngpio  — number of GPIO lines this chip exposes (maps to
     *              "ngpios" DT property or hardware default)
     * gc->base   — starting GPIO number in the global numberspace
     *              (-1 means kernel picks dynamically)
     * gc->of_node — the DT node; if non-NULL this chip came from DT
     */
    pr_info("[gpio_spy] gpiochip_add_data called:\n");
    pr_info("[gpio_spy]   label    = %s\n",
            gc->label ? gc->label : "(null)");
    pr_info("[gpio_spy]   ngpio    = %u  (DT: #gpio-cells lines)\n",
            gc->ngpio);
    pr_info("[gpio_spy]   base     = %d  (-1 → dynamic assignment)\n",
            gc->base);
    pr_info("[gpio_spy]   from DT  = %s\n",
            gc->of_node ? of_node_full_name(gc->of_node) : "no DT node");

    return 0; /* 0 = let the real function continue */
}

/* ------------------------------------------------------------------ */
/* kprobe struct                                                        */
/* ------------------------------------------------------------------ */

static struct kprobe kp = {
    .symbol_name = "gpiochip_add_data", /* function to intercept */
    .pre_handler = handler_pre,          /* our callback          */
};

/* ------------------------------------------------------------------ */
/* init / exit                                                          */
/* ------------------------------------------------------------------ */

static int __init gpio_spy_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[gpio_spy] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[gpio_spy] kprobe planted on gpiochip_add_data @ %p\n",
            kp.addr);
    return 0;
}

static void __exit gpio_spy_exit(void)
{
    /*
     * Unregister MUST happen in exit — otherwise the handler pointer
     * points to unloaded code and the next call to gpiochip_add_data
     * causes a kernel panic (bad IP).
     */
    unregister_kprobe(&kp);
    pr_info("[gpio_spy] kprobe removed, module unloaded cleanly\n");
}

module_init(gpio_spy_init);
module_exit(gpio_spy_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | ماكروهات `module_init`, `module_exit`, `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info`, `pr_err` |
| `linux/kprobes.h` | الـ `struct kprobe` وكل الـ API |
| `linux/gpio/driver.h` | **`struct gpio_chip`** — الـ struct الأساسي لكل GPIO controller |

الـ `gpio/driver.h` مهم لأن فيه تعريف `gpio_chip` اللي بيجمع كل المعلومات اللي DT binding بيحولها لـ kernel objects، زي `ngpio` و `label` و `of_node`.

---

#### الـ `handler_pre` — قلب الموضوع

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

**الـ kprobe** بيوقف التنفيذ لحظة ما قبل ما `gpiochip_add_data` تشتغل، وبيديك `pt_regs` فيه الـ CPU registers وقت الاستدعاء. على x86-64 بيتبع **System V AMD64 ABI**، يعني:

- `regs->di` = الـ argument الأول = `struct gpio_chip *gc`

الـ `gc->of_node` هو الرابط المباشر بين الكود والـ DT — لو موجود معناها الـ controller جه من DT node زي اللي اتوصف في `gpio.txt`.

---

#### ليه `gpiochip_add_data` تحديداً؟

لأنها **النقطة الوحيدة** اللي فيها الـ DT binding بيتجسد — لما الـ driver بيعمل probe على الـ platform device المرتبط بالـ DT node اللي فيه `gpio-controller;`، بيبني `gpio_chip` ويسجلها بـ `gpiochip_add_data`. قبل الاستدعاء ده: لا `/sys/class/gpio/gpiochipN`، لا `gpiolib` entries، لا `gpio-line-names` متاحة للـ userspace.

---

#### الـ `module_init` و `module_exit`

```c
ret = register_kprobe(&kp);
```

**الـ `register_kprobe`** بيعمل:
1. يبحث عن عنوان `gpiochip_add_data` في الـ kernel symbol table.
2. يحفظ الـ instruction الأصلية ويحطّ مكانها breakpoint (int3 على x86).
3. لما الـ breakpoint يضرب، بيشغل `pre_handler` وبعدين يرجع الـ instruction الأصلية.

```c
unregister_kprobe(&kp);
```

**لازم** يتعمل في الـ `exit` — لو المودول اتفكّ من الذاكرة وفضل الـ breakpoint موجود، الـ kernel هيحاول يشغل `handler_pre` اللي عنوانه بقى garbage، والنتيجة **kernel panic** فوري.

---

### Makefile للبناء

```makefile
obj-m += gpio_dt_probe_spy.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وملاحظة النتيجة

```bash
# بناء المودول
make

# تحميل
sudo insmod gpio_dt_probe_spy.ko

# لو عندك GPIO controller بيتسجل (مثلاً على Raspberry Pi أو QEMU virt):
# شوف الـ output في kernel log
sudo dmesg | grep gpio_spy

# تفريغ
sudo rmmod gpio_dt_probe_spy
sudo dmesg | tail -3
```

**مثال output متوقع على Raspberry Pi:**

```
[gpio_spy] kprobe planted on gpiochip_add_data @ ffffffffc0123456
[gpio_spy] gpiochip_add_data called:
[gpio_spy]   label    = pinctrl-bcm2835
[gpio_spy]   ngpio    = 54  (DT: #gpio-cells lines)
[gpio_spy]   base     = -1  (-1 → dynamic assignment)
[gpio_spy]   from DT  = /soc/gpio@7e200000
```

الـ `/soc/gpio@7e200000` ده بالظبط الـ DT node اللي فيه `gpio-controller;` و `#gpio-cells = <2>;` — اللي اتكلمنا عنه في `gpio.txt`.

---

### ربط المودول بالـ DT Binding

| مفهوم في `gpio.txt` | مقابله في الكود |
|---|---|
| `gpio-controller;` property | وجود `gc->of_node` non-NULL |
| `#gpio-cells = <2>` | عدد cells محفوظ في driver، مش في `gpio_chip` مباشرة |
| `ngpios = <18>` | `gc->ngpio == 18` |
| `gpio-line-names` | متاحة بعد ما `gpiochip_add_data` تنجح |
| `gpio-ranges` | بتتحل بعد التسجيل عبر `gpiochip_add_pin_range()` |
| base GPIO number | `gc->base` (-1 لو الـ driver سابه للـ kernel) |
