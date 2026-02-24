## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Pin Control Subsystem؟

تخيل إن عندك لوحة Arduino أو Raspberry Pi — فيها أرجل (pins) كتير. كل رجل ممكن يشتغل بأكتر من طريقة: يبقى GPIO عادي، أو يبعت بيانات SPI، أو يشيل إشارة UART، أو يحمل ساعة I2C. المشكلة: **الـ pin الواحد مش ممكن يعمل الاتنين في نفس الوقت**.

الـ **pinctrl subsystem** هو المسؤول في Linux kernel عن:
1. تسجيل كل pin موجود في الـ SoC
2. تحديد أي وظيفة (function) بتشتغل على أي مجموعة pins (pinmux)
3. ضبط الإعدادات الفيزيائية للـ pin زي pull-up/pull-down والـ drive strength (pinconf)

---

### القصة كاملة — من الهاردوير للسوفتوير

**المشكلة اللي بيحلها:**

في أي SoC حديث (زي i.MX6 أو Raspberry Pi BCM2835 أو STM32)، الـ chip بيحتوي على مئات الـ pins. كل pin ممكن يكون متصل بـ 8 أو 16 وظيفة مختلفة داخل الـ chip، لكن طبعاً بيشتغل وظيفة واحدة بس في الوقت الواحد.

**مثال حقيقي:**

```
STM32 Pin PA9:
  - Function 0: GPIO output
  - Function 1: USART1_TX
  - Function 7: TIM1_CH2
  - Function 13: SAI1_SCK_B
```

لما الـ UART driver يشتغل، محتاج PA9 يبقى USART1_TX. لما مفيش حد بيستخدمه، يرجع GPIO. مين بيتحكم في ده؟ **الـ pinctrl subsystem**.

---

### دور الملف `pinctrl.h`

الملف ده هو **واجهة تسجيل الـ pin controller drivers** — يعني الكود اللي بيعرّف للـ kernel:
- "أنا pin controller، عندي X pin"
- "هنا قائمة الـ pins وأسمائهم وأرقامهم"
- "هنا العمليات اللي أقدر أعملها"

بالتالي هو **مش للمستخدم (consumer) وهو مش للـ hardware driver مباشرة** — هو نقطة التقاء بين الاتنين، بيحدد **الـ contract** اللي لازم أي pin controller driver يلتزم بيه.

---

### الـ Actors الرئيسية في الـ Subsystem

```
┌─────────────────────────────────────────────────────┐
│                  Linux Kernel                        │
│                                                      │
│  ┌──────────────┐     ┌───────────────────────────┐  │
│  │ Device Driver │     │   GPIO Subsystem          │  │
│  │  (consumer)   │     │  (gpio_chip)              │  │
│  └──────┬───────┘     └────────────┬──────────────┘  │
│         │                          │                  │
│         ▼                          ▼                  │
│  ┌─────────────────────────────────────────────────┐  │
│  │           pinctrl core  (drivers/pinctrl/core.c) │  │
│  │  - يحفظ كل الـ pin controllers المسجلين         │  │
│  │  - يربط الـ consumers بالـ states               │  │
│  │  - يحل تعارضات الـ ownership                   │  │
│  └────────────────────┬────────────────────────────┘  │
│                       │                               │
│         ┌─────────────┼────────────┐                  │
│         ▼             ▼            ▼                  │
│  ┌────────────┐ ┌──────────┐ ┌───────────┐           │
│  │ pinctrl_ops│ │pinmux_ops│ │pinconf_ops│           │
│  │ (grouping) │ │ (muxing) │ │  (config) │           │
│  └────────────┘ └──────────┘ └───────────┘           │
│         │                                             │
│         ▼                                             │
│  ┌─────────────────────────────────────────────────┐  │
│  │    Hardware Pin Controller Driver                │  │
│  │    (e.g., drivers/pinctrl/bcm/pinctrl-bcm2835.c)│  │
│  └─────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

### أهم الـ Structs في الملف

#### `struct pinctrl_pin_desc` — وصف الـ pin الواحد

```c
struct pinctrl_pin_desc {
    unsigned int number;  /* رقم فريد للـ pin في الـ global space */
    const char *name;     /* اسم بشري زي "GPIO0" أو "PA9" */
    void *drv_data;       /* بيانات خاصة بالـ driver، الـ core ميلمسهاش */
};
```

المستخدم بيعرّفها بالماكرو:
```c
static const struct pinctrl_pin_desc bcm2835_gpio_pins[] = {
    PINCTRL_PIN(0, "GPIO0"),
    PINCTRL_PIN(1, "GPIO1"),
    /* ... */
    PINCTRL_PIN(53, "GPIO53"),
};
```

#### `struct pingroup` — مجموعة pins بتشتغل مع بعض

```c
struct pingroup {
    const char *name;           /* اسم المجموعة زي "uart0_grp" */
    const unsigned int *pins;   /* مصفوفة أرقام الـ pins */
    size_t npins;               /* عددهم */
};
```

لأن في كتير من الـ functions محتاجة أكتر من pin واحد — زي SPI اللي محتاج MOSI + MISO + CLK + CS.

#### `struct pinfunction` — الوظيفة اللي المجموعة بتعملها

```c
struct pinfunction {
    const char *name;             /* "spi0", "uart1", "gpio" */
    const char * const *groups;   /* المجموعات اللي بتدعم الوظيفة دي */
    size_t ngroups;
    unsigned long flags;          /* PINFUNCTION_FLAG_GPIO لو ده GPIO */
};
```

#### `struct pinctrl_ops` — العمليات الأساسية

```c
struct pinctrl_ops {
    /* كام مجموعة عندك؟ */
    int (*get_groups_count)(struct pinctrl_dev *pctldev);

    /* ايه اسم المجموعة رقم N؟ */
    const char *(*get_group_name)(struct pinctrl_dev *pctldev,
                                  unsigned int selector);

    /* ايه الـ pins في المجموعة رقم N؟ */
    int (*get_group_pins)(struct pinctrl_dev *pctldev,
                          unsigned int selector,
                          const unsigned int **pins,
                          unsigned int *num_pins);

    /* لـ Device Tree: حوّل الـ node لـ mapping entries */
    int (*dt_node_to_map)(struct pinctrl_dev *pctldev,
                          struct device_node *np_config,
                          struct pinctrl_map **map,
                          unsigned int *num_maps);

    /* حرر الـ mapping entries اللي عملتها */
    void (*dt_free_map)(struct pinctrl_dev *pctldev,
                        struct pinctrl_map *map,
                        unsigned int num_maps);
};
```

#### `struct pinctrl_desc` — الـ descriptor الكامل للـ controller

```c
struct pinctrl_desc {
    const char *name;                       /* اسم الـ controller */
    const struct pinctrl_pin_desc *pins;    /* مصفوفة كل الـ pins */
    unsigned int npins;                     /* عددهم */
    const struct pinctrl_ops *pctlops;      /* عمليات التجميع (grouping) */
    const struct pinmux_ops *pmxops;        /* عمليات الـ muxing */
    const struct pinconf_ops *confops;      /* عمليات الـ configuration */
    struct module *owner;
    bool link_consumers; /* اعمل device link بين الـ controller والـ consumers */
};
```

ده هو اللي بيتبعت لـ `pinctrl_register_and_init()` عشان تسجّل نفسك.

#### `struct pinctrl_gpio_range` — تكامل الـ GPIO

```c
struct pinctrl_gpio_range {
    struct list_head node;
    const char *name;
    unsigned int base;      /* أول رقم GPIO في الـ range */
    unsigned int pin_base;  /* أول رقم pin مقابل */
    unsigned int npins;     /* عدد الـ pins */
    struct gpio_chip *gc;   /* الـ gpio_chip المقابل */
};
```

ده الجسر بين الـ pinctrl والـ GPIO subsystem — بيقول "الـ GPIO 100 → 163 بيقابل الـ pins 0 → 63 في الـ controller ده".

---

### دورة حياة الـ Pin Controller Driver

```
1. Driver probe()
       │
       ▼
2. عرّف مصفوفة pinctrl_pin_desc[]
   عرّف مصفوفة pingroup[]
   عرّف مصفوفة pinfunction[]
       │
       ▼
3. اعبي struct pinctrl_desc بالـ ops الثلاث
       │
       ▼
4. استدعي pinctrl_register_and_init()
   ثم pinctrl_enable()
       │
       ▼
5. الـ core يسجّله في قائمته الداخلية
       │
       ▼
6. لما device driver يطلب state معين:
   الـ core يبحث في الـ mapping table
   يطلب من الـ pinmux_ops يعمل set_mux()
   يطلب من الـ pinconf_ops يعمل pin_config_set()
```

---

### الـ GPIO Range Integration — القصة كاملة

**المشكلة:** الـ GPIO subsystem والـ pinctrl subsystem منفصلين في الـ kernel. لكن في الـ hardware، GPIO controller والـ pin controller غالباً نفس الـ IP block.

**الحل:** `pinctrl_gpio_range` — بيسمح للـ GPIO subsystem إنه لما حد يعمل `gpio_request(100)`، يكلم الـ pinctrl عشان يحجز الـ pin المقابل ويمنع أي driver تاني من استخدامه للـ muxing.

```c
/* مثال: ربط GPIO 0-53 بـ pins 0-53 في الـ BCM2835 */
static struct pinctrl_gpio_range bcm2835_pinctrl_gpio_range = {
    .name = "bcm2835_gpio",
    .npins = BCM2835_NUM_GPIOS,
};
/* ثم: */
pinctrl_add_gpio_range(pctldev, &bcm2835_pinctrl_gpio_range);
```

---

### الملفات المكوّنة للـ Subsystem

#### Core Headers
| الملف | الدور |
|-------|-------|
| `include/linux/pinctrl/pinctrl.h` | **الملف ده** — تسجيل الـ controllers، الـ pins، والـ groups |
| `include/linux/pinctrl/pinmux.h` | تعريف `pinmux_ops` — منطق الـ multiplexing |
| `include/linux/pinctrl/pinconf.h` | تعريف `pinconf_ops` — ضبط الخصائص الفيزيائية |
| `include/linux/pinctrl/pinconf-generic.h` | تعريفات standard للـ configs زي `PIN_CONFIG_BIAS_PULL_UP` |
| `include/linux/pinctrl/consumer.h` | الواجهة للـ device drivers اللي بتستهلك الـ pin states |
| `include/linux/pinctrl/machine.h` | الـ mapping table — ربط devices بـ states بـ functions |
| `include/linux/pinctrl/pinctrl-state.h` | أسماء الـ states: `"default"`, `"init"`, `"sleep"`, `"idle"` |
| `include/linux/pinctrl/devinfo.h` | info بتتحط في كل `struct device` لتتبع الـ pinctrl state |

#### Core Implementation
| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/core.c` | الـ core engine — كل منطق التسجيل والـ state management |
| `drivers/pinctrl/pinmux.c` | تنفيذ منطق الـ muxing وحل التعارضات |
| `drivers/pinctrl/pinconf.c` | تنفيذ منطق الـ pin configuration |
| `drivers/pinctrl/devicetree.c` | تحويل الـ DT nodes لـ mapping entries |

#### Hardware Drivers (أمثلة)
| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/bcm/pinctrl-bcm2835.c` | Raspberry Pi (BCM2835) |
| `drivers/pinctrl/freescale/pinctrl-imx.c` | NXP i.MX series |
| `drivers/pinctrl/intel/pinctrl-intel.c` | Intel Atom/Baytrail |
| `drivers/pinctrl/nomadik/` | ST-Ericsson (الـ hardware اللي الـ subsystem اتبنى عشانه) |

#### Documentation
| الملف | الدور |
|-------|-------|
| `Documentation/driver-api/pin-control.rst` | الـ reference الكامل للـ subsystem |
| `Documentation/devicetree/bindings/pinctrl/` | الـ DT bindings لكل controller |
## Phase 2: شرح الـ Pinctrl Framework

---

### المشكلة — ليه الـ Pinctrl موجود أصلاً؟

في أي SoC حديث — خد مثلاً STM32، RK3588، أو i.MX8 — عندك مئات الـ physical pins على الشريحة. كل pin ممكن يشتغل بأكتر من وظيفة (function):

- نفس الـ pin يبقى UART TX أو SPI MOSI أو GPIO عادي، حسب الـ mux register اللي تكتب فيه.
- نفس الـ pin محتاج تضبط له pull-up/pull-down، drive strength، slew rate، schmitt trigger.

**المشكلة الأساسية:** قبل الـ pinctrl subsystem، كل driver كان بيتعامل مع الـ pin registers بشكل مباشر — hard-coded addresses، magic values، وكل board support package بيعيد اختراع العجلة. النتيجة:

1. **تعارض**: درايفرين مختلفين ممكن يطلبوا نفس الـ pin لوظيفتين مختلفتين وما فيش حكم.
2. **تشتت**: كود الـ pin configuration موزع على آلاف الملفات في `arch/`.
3. **صعوبة الـ power management**: عند الدخول لـ suspend، لازم تحفظ state كل pin وترجعه — من غير abstraction ده صعب.
4. **تكرار**: كل SoC driver بيكتب نفس الـ boilerplate.

---

### الحل — المنهج اللي اتخده الـ Kernel

الـ **pinctrl subsystem** (دخل الـ kernel في v3.2 سنة 2012، بيد Linus Walleij) بيحل المشكلة عن طريق:

1. **Centralized ownership**: كل pin في النظام بيتسجل عند الـ pinctrl core، اللي بيمنع طلبين متعارضين.
2. **Three orthogonal services**: الـ subsystem بيقسم الشغل لـ 3 طبقات مستقلة:
   - **pinctrl** (التسجيل والـ grouping)
   - **pinmux** (اختيار الوظيفة — UART/SPI/GPIO)
   - **pinconf** (ضبط الخصائص — pull، drive strength)
3. **State machine**: بدل ما driver يبعث commands، بيعرّف **states** (default, sleep, idle) والـ core بيطبقها في الوقت المناسب.
4. **Device Tree integration**: الـ pin configuration بتتنقل من الكود للـ DTS، وتصبح board-specific لا driver-specific.

---

### التشبيه الواقعي — لوحة التوصيل في الاستوديو

تخيّل **استوديو تسجيل صوتي** فيه لوحة توصيل كبيرة (patch bay):

| مفهوم الاستوديو | المقابل في الـ Pinctrl |
|---|---|
| كل **socket** في الـ patch bay | كل **pin** في الـ SoC (`pinctrl_pin_desc`) |
| **كابل التوصيل** بين socketsاتنين | الـ **mux setting** — بيوصّل pin بوظيفة معينة |
| **مجموعة sockets** لآلة موسيقية واحدة | الـ **pin group** — pins بتشتغل مع بعض كـ UART مثلاً |
| **preset configuration** محفوظة للبرنامج | الـ **pinctrl state** (default/sleep) |
| **مهندس الصوت** اللي بيمنع توصيلين يتعارضوا | الـ **pinctrl core** — المحكم المركزي |
| **كتالوج الأجهزة** اللي الاستوديو بيدعمها | الـ **pinctrl_desc** — وصف كل الـ pins والـ operations |
| العازف اللي محتاج توصيلات معينة | الـ **consumer driver** (UART, SPI, I2C driver) |
| شركة صناعة الـ patch bay | الـ **pin controller driver** (الـ provider) |

التعمق في التشبيه:
- لما عازفان بيطلبوا نفس الـ socket → الـ core بيرفض ويرجع error (مثل `request()` callback في `pinmux_ops`).
- الـ preset configurations مش بتتطبق غير لما حد يطلبها (`pinctrl_select_state`) — مش automatic.
- الـ patch bay نفسها مش بتفهم الموسيقى — بس بتعرف "مين موصّل بمين"، زي الـ core بالظبط.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                      CONSUMER DRIVERS                           │
  │  uart_driver   spi_driver   i2c_driver   gpio_driver            │
  │       │             │            │             │                 │
  │       └─────────────┴────────────┴─────────────┘                │
  │                           │                                      │
  │                   pinctrl_get(dev)                               │
  │                   pinctrl_select_state(p, s)                     │
  └───────────────────────────┼─────────────────────────────────────┘
                              │  consumer.h API
  ┌───────────────────────────▼─────────────────────────────────────┐
  │                   PINCTRL CORE (kernel/pinctrl/)                 │
  │                                                                   │
  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐   │
  │  │  Pin Registry│  │ State Machine│  │  Mapping Table       │   │
  │  │  (radix tree)│  │ default/sleep│  │  (pinctrl_map[])     │   │
  │  │              │  │ idle/init    │  │  from DT or C code   │   │
  │  └──────────────┘  └──────────────┘  └─────────────────────┘   │
  │                                                                   │
  │         ┌────────────────────────────────────┐                  │
  │         │         pinctrl_desc               │                  │
  │         │  ┌──────────┐  ┌───────────────┐  │                  │
  │         │  │pinctrl_  │  │  pinmux_ops   │  │                  │
  │         │  │ops       │  │  set_mux()    │  │                  │
  │         │  │get_groups│  │  request()    │  │                  │
  │         │  └──────────┘  └───────────────┘  │                  │
  │         │  ┌────────────────────────────┐   │                  │
  │         │  │       pinconf_ops          │   │                  │
  │         │  │  pin_config_set()          │   │                  │
  │         │  └────────────────────────────┘   │                  │
  │         └────────────────────────────────────┘                  │
  └───────────────────────────┬─────────────────────────────────────┘
                              │  pinctrl_register_and_init()
  ┌───────────────────────────▼─────────────────────────────────────┐
  │              PIN CONTROLLER DRIVER (the provider)                │
  │                                                                   │
  │   e.g. pinctrl-stm32.c / pinctrl-rockchip.c / pinctrl-bcm2835  │
  │                                                                   │
  │   implements:  pinctrl_ops + pinmux_ops + pinconf_ops            │
  │   registers:   pinctrl_pin_desc[]  (all pins on SoC)            │
  │   owns:        register MMIO base address                        │
  └───────────────────────────┬─────────────────────────────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────────────┐
  │                       HARDWARE                                    │
  │   SoC Pin Mux Registers / Pad Config Registers / GPIO Bank Regs │
  └─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstractions — الأفكار المحورية

#### 1. الـ `pinctrl_pin_desc` — بطاقة هوية كل Pin

```c
struct pinctrl_pin_desc {
    unsigned int number;  /* رقم Pin عالمي فريد على مستوى الـ SoC */
    const char *name;     /* اسم قابل للقراءة مثل "PA0" أو "GPIO_23" */
    void *drv_data;       /* بيانات خاصة بالـ driver مثل register offset */
};
```

الـ `number` ده مش الـ GPIO number — ده رقم الـ pin في الـ pinctrl namespace. الـ GPIO subsystem عنده namespace مستقل (راجع `gpio_chip` و `gpio_desc` في الـ GPIO subsystem).

الـ `drv_data` بيخلي الـ pin controller driver يخزن أي بيانات يحتاجها بدون ما الـ core يتدخل — مثلاً register offset أو bitmask خاص بالـ SoC.

---

#### 2. الـ `pingroup` و `pinfunction` — الـ Logical Grouping

```c
struct pingroup {
    const char *name;           /* مثل "uart0_grp" */
    const unsigned int *pins;   /* مصفوفة أرقام الـ pins */
    size_t npins;
};

struct pinfunction {
    const char *name;           /* مثل "uart0" أو "spi1" */
    const char * const *groups; /* مجموعات الـ pins اللي ممكن تأدي الوظيفة دي */
    size_t ngroups;
    unsigned long flags;        /* PINFUNCTION_FLAG_GPIO لو GPIO function */
};
```

**لماذا منفصلتان؟** لأن نفس الـ function ممكن تتنفذ من خلال أكتر من مجموعة pins — مثلاً `uart0` ممكن يكون على `uart0_grp_a` (PA0,PA1) أو `uart0_grp_b` (PB4,PB5) — بيعطي flexibility في الـ PCB routing.

---

#### 3. الـ `pinctrl_desc` — الوصف الشامل للـ Pin Controller

```c
struct pinctrl_desc {
    const char *name;                        /* اسم الـ controller */
    const struct pinctrl_pin_desc *pins;     /* مصفوفة كل الـ pins */
    unsigned int npins;
    const struct pinctrl_ops *pctlops;       /* grouping operations */
    const struct pinmux_ops *pmxops;         /* mux operations */
    const struct pinconf_ops *confops;       /* config operations */
    struct module *owner;
    /* ... custom params for generic pinconf ... */
    bool link_consumers;  /* device link لضمان ترتيب الـ suspend/resume */
};
```

ده الـ "عقد" بين الـ driver والـ core. Driver بيملا الـ struct دي ويبعتها لـ `pinctrl_register_and_init()`.

الـ `link_consumers = true` مهم جداً على ARM SoCs — بيخلي الـ kernel يعرف إن الـ UART driver مثلاً يصحى **بعد** الـ pinctrl driver في الـ resume، لأن لو العكس حصل الـ pins هتفضل بدون config.

---

#### 4. الـ vtable الثلاثية — فصل المسؤوليات

**الـ `pinctrl_ops`** — الـ grouping layer:

```c
struct pinctrl_ops {
    /* كام group عندنا؟ */
    int (*get_groups_count)(struct pinctrl_dev *pctldev);

    /* اسم الـ group رقم selector ايه؟ */
    const char *(*get_group_name)(struct pinctrl_dev *pctldev,
                                  unsigned int selector);

    /* جيبلي الـ pins اللي في الـ group ده */
    int (*get_group_pins)(struct pinctrl_dev *pctldev,
                          unsigned int selector,
                          const unsigned int **pins,
                          unsigned int *num_pins);

    /* DT parsing: حوّل device tree node لـ pinctrl_map entries */
    int (*dt_node_to_map)(struct pinctrl_dev *pctldev,
                          struct device_node *np_config,
                          struct pinctrl_map **map,
                          unsigned int *num_maps);

    void (*dt_free_map)(struct pinctrl_dev *pctldev,
                        struct pinctrl_map *map,
                        unsigned int num_maps);
};
```

**الـ `pinmux_ops`** — الـ mux layer:

```c
struct pinmux_ops {
    /* هل الـ pin متاح للـ mux؟ (مش محجوز لحد تاني) */
    int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);
    int (*free)(struct pinctrl_dev *pctldev, unsigned int offset);

    /* كام function عندنا؟ وايه أسمائها؟ */
    int (*get_functions_count)(struct pinctrl_dev *pctldev);
    const char *(*get_function_name)(struct pinctrl_dev *pctldev,
                                     unsigned int selector);

    /* اعمل الـ mux فعلاً: اكتب في الـ register */
    int (*set_mux)(struct pinctrl_dev *pctldev,
                   unsigned int func_selector,
                   unsigned int group_selector);

    /* GPIO fast path: بدون الحاجة لـ function/group lookup */
    int (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                               struct pinctrl_gpio_range *range,
                               unsigned int offset);
    int (*gpio_set_direction)(struct pinctrl_dev *pctldev,
                              struct pinctrl_gpio_range *range,
                              unsigned int offset, bool input);

    bool strict; /* امنع استخدام نفس الـ pin كـ GPIO وfunction في نفس الوقت */
};
```

**الـ `pinconf_ops`** — الـ configuration layer:

```c
struct pinconf_ops {
    /* اقرأ/اكتب config لـ pin واحد */
    int (*pin_config_get)(struct pinctrl_dev *pctldev,
                          unsigned int pin, unsigned long *config);
    int (*pin_config_set)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *configs, unsigned int num_configs);

    /* اقرأ/اكتب config لـ group كامل دفعة واحدة */
    int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *configs,
                                unsigned int num_configs);
};
```

الـ `config` في الـ `pinconf_ops` هو `unsigned long` بيحتوي بالـ FIELD_PREP على parameter type في الـ high bits وقيمته في الـ low bits — ده الـ **generic pinconf encoding** (من `pinconf-generic.h`).

---

#### 5. الـ `pinctrl_map` — جدول الربط

```c
struct pinctrl_map {
    const char *dev_name;       /* اسم الـ consumer device */
    const char *name;           /* اسم الـ state: "default", "sleep", ... */
    enum pinctrl_map_type type; /* MUX_GROUP أو CONFIGS_PIN أو CONFIGS_GROUP */
    const char *ctrl_dev_name;  /* اسم الـ pin controller اللي بينفذ */
    union {
        struct pinctrl_map_mux mux;         /* group + function */
        struct pinctrl_map_configs configs;  /* pin/group + config array */
    } data;
};
```

ده الجسر بين الـ consumer driver والـ pin controller. ممكن يتبني من:
- **الـ DTS** عبر `dt_node_to_map()` (الأشيع على ARM boards)
- **كود C مباشر** عبر `pinctrl_register_mappings()` (للأنظمة اللي مش بتستخدم DT)

---

### علاقة الـ Structs ببعض — الرسم الكامل

```
pinctrl_desc
    │
    ├──► pinctrl_pin_desc[]   ◄── كل pin بـ number + name + drv_data
    │
    ├──► pinctrl_ops          ◄── get_groups_count / get_group_pins / dt_node_to_map
    │         │
    │         └── يرجع ──► pingroup[] (name + pins[] + npins)
    │
    ├──► pinmux_ops           ◄── request / set_mux / gpio_request_enable
    │         │
    │         └── يشتغل على ──► pinfunction[] (name + groups[] + flags)
    │
    └──► pinconf_ops          ◄── pin_config_set / pin_config_group_set
              │
              └── يشتغل على ──► unsigned long configs[] (encoded params)


pinctrl_map[]  ──► ربط: (dev_name + state_name) → (ctrl_dev + group + function/config)
      │
      ▼
pinctrl_core بيقرأ الـ map ويبني:
      │
      ▼
pinctrl (per-device handle)
      │
      ├──► pinctrl_state "default"  ──► list of settings
      ├──► pinctrl_state "sleep"    ──► list of settings
      └──► pinctrl_state "idle"     ──► list of settings
```

---

### الـ GPIO Range — الجسر مع الـ GPIO Subsystem

**الـ GPIO Subsystem** — نوضحه في سطر: هو الـ framework المسؤول عن قراءة وكتابة قيم الـ GPIO pins، وبيشتغل من خلال `gpio_chip`. مستقل عن الـ pinctrl لكن بحاجة يتنسق معاه.

```c
struct pinctrl_gpio_range {
    struct list_head node;  /* ضمن list في الـ pin controller */
    const char *name;
    unsigned int id;
    unsigned int base;      /* أول GPIO number في الـ GPIO namespace */
    unsigned int pin_base;  /* أول pin number في الـ pinctrl namespace */
    unsigned int npins;
    unsigned int const *pins; /* أو NULL لو sequential */
    struct gpio_chip *gc;     /* الـ gpio_chip المقابلة */
};
```

**السبب:** الـ GPIO number space والـ pinctrl pin number space مختلفان. مثلاً GPIO 32 ممكن يكون pin 100 في الـ pinctrl. الـ `pinctrl_gpio_range` هو الـ translation table.

لما `gpio_request()` بتتنادى على pin معين، الـ GPIO core بيبعت لـ `pinctrl_gpio_request()` اللي بيلاقي الـ range المناسب وبينادي `gpio_request_enable()` على الـ pin controller.

```
GPIO Request (GPIO 32)
      │
      ▼
pinctrl_gpio_request(gc, offset=0)
      │
      ▼  بحث في pinctrl_gpio_range list
      ▼
pinctrl_find_gpio_range_from_pin()  →  range: base=32, pin_base=100
      │
      ▼
pinmux_ops.gpio_request_enable(pctldev, range, offset=0)
      │
      ▼
Hardware: اكتب في register الـ mux إن pin 100 = GPIO mode
```

---

### الـ States System — ماشين الـ Power Management

```
                    ┌──────────┐
          probe ──► │ "init"   │ (optional, قبل probe)
                    └────┬─────┘
                         │ (بعد probe لو لسه "init")
                         ▼
                    ┌──────────┐ ◄── pm_runtime_resume / resume
      normal ──────►│"default" │
                    └────┬─────┘
                         │ pm_runtime_suspend / idle
                         ▼
                    ┌──────────┐
                    │  "idle"  │
                    └────┬─────┘
                         │ suspend
                         ▼
                    ┌──────────┐
                    │ "sleep"  │
                    └──────────┘
```

كل state بيتمثل في `pinctrl_state` struct اللي فيها list من الـ settings. كل setting هي إما mux setting أو config setting. لما بتنادي `pinctrl_select_state()` الـ core بيطبق كل الـ settings في الـ state دي بالترتيب.

---

### الـ Core يمتلك، الـ Driver ينفذ

| المسؤولية | الـ Core يمتلكها | الـ Driver ينفذها |
|---|---|---|
| منع تعارض استخدام الـ pins | نعم | لا |
| تخزين الـ mapping table | نعم | لا |
| بناء الـ state machine | نعم | لا |
| debugfs entries | نعم | callback اختياري |
| كتابة الـ mux register | لا | نعم (`set_mux`) |
| كتابة الـ config register | لا | نعم (`pin_config_set`) |
| DT parsing لـ vendor-specific nodes | لا | نعم (`dt_node_to_map`) |
| تحرير الـ map memory | لا | نعم (`dt_free_map`) |
| التحقق من توافر الـ pin | الـ core بيسأل | الـ driver بيجاوب (`request`) |
| تعريف الـ pin groups | لا | نعم (عبر `get_group_pins`) |

---

### مثال واقعي — UART على STM32

فرض عندنا STM32MP1، الـ UART4 بيستخدم PA11 (TX) و PA12 (RX):

```c
/* في pinctrl-stm32mp151.c */
static const unsigned int uart4_pins_a[] = { 11, 12 }; /* pin numbers */

static const struct pingroup stm32mp151_groups[] = {
    PINCTRL_PINGROUP("uart4grp_a", uart4_pins_a, ARRAY_SIZE(uart4_pins_a)),
    /* ... */
};

/* الـ function */
static const char * const uart4_groups[] = { "uart4grp_a", "uart4grp_b" };
static const struct pinfunction stm32mp151_functions[] = {
    PINCTRL_PINFUNCTION("uart4", uart4_groups, ARRAY_SIZE(uart4_groups)),
    /* ... */
};
```

في الـ DTS:
```dts
/* في stm32mp151.dtsi */
uart4 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart4_pins_a>;   /* default state */
    pinctrl-1 = <&uart4_sleep_pins_a>; /* sleep state */
};

&pinctrl {
    uart4_pins_a: uart4-0 {
        pins1 {
            pinmux = <STM32_PINMUX('A', 11, AF6)>; /* PA11 = UART4_TX, Alt Func 6 */
            bias-disable;
            drive-push-pull;
            slew-rate = <0>;
        };
        pins2 {
            pinmux = <STM32_PINMUX('A', 12, AF6)>; /* PA12 = UART4_RX */
            bias-disable;
        };
    };
};
```

لما الـ uart driver بيعمل `devm_pinctrl_get(dev)` ثم `pinctrl_select_state(p, default_state)`:
1. الـ core بيبحث في الـ mapping table عن `(dev="uart4", state="default")`.
2. بيلاقي الـ map entry اللي بتقول: استخدم group `uart4grp_a`، function `uart4`.
3. بينادي `pinmux_ops.set_mux(pctldev, func_sel, grp_sel)`.
4. الـ STM32 driver بيكتب في الـ MODER و AFRL registers لـ PA11 و PA12.
5. بينادي `pinconf_ops.pin_config_set` لكل pin بالـ configs (pull, drive, slew).

---

### الـ Registration Flow — من الـ Driver للـ Core

```
pin_controller_driver.probe()
          │
          ├── ملا pinctrl_desc بالـ ops والـ pins
          │
          ▼
pinctrl_register_and_init(pctldesc, dev, drv_data, &pctldev)
          │
          ├── core بيعمل alloc لـ pinctrl_dev
          ├── بيسجل كل pin في الـ radix tree
          ├── بيعمل validate على الـ ops
          └── بيرجع pctldev (opaque handle)
          │
          ▼
pinctrl_enable(pctldev)
          │
          ├── بيطبق الـ "hog" mappings
          │   (entries where dev_name == ctrl_dev_name — الـ controller بياخدها لنفسه)
          └── الـ controller جاهز للاستخدام
```

الـ **hog mechanism** مهم: لو الـ pin controller نفسه محتاج pins معينة (مثلاً pins للـ SoC internal functions)، بيعرّف mappings بـ `dev_name == ctrl_dev_name`، والـ core بياخدها تلقائياً عند الـ `pinctrl_enable()` — بدون ما حد يطلبها.

---

### الـ `devm_` vs الـ Manual API

| الـ API | الاستخدام | تحرير الموارد |
|---|---|---|
| `pinctrl_register_and_init()` + `pinctrl_enable()` | الـ preferred modern API | يدوي عبر `pinctrl_unregister()` |
| `devm_pinctrl_register_and_init()` | الأفضل في معظم الحالات | تلقائي عند `device_unregister()` |
| `pinctrl_register()` (قديم) | legacy، لا تستخدمه | يدوي |

الـ `devm_` (device-managed) API بتربط الـ resource بعمر الـ device — لو الـ driver اتفصل، الـ pinctrl resource بيتحرر تلقائياً، وده بيمنع leaks في الـ error paths.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options — Cheatsheet

#### الـ `PINFUNCTION_FLAG_*` Flags

| Flag | Value | المعنى |
|------|-------|--------|
| `PINFUNCTION_FLAG_GPIO` | `BIT(0)` | الـ function دي بتمثل GPIO mode — بتُستخدم مع `PINCTRL_GPIO_PINFUNCTION()` |

#### الـ `enum pinctrl_map_type` (من `machine.h`)

| القيمة | المعنى |
|--------|--------|
| `PIN_MAP_TYPE_INVALID` | entry فاسد أو غير معرَّف |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمية — بتُستخدم لما الـ pinctrl مش موجود فعلاً |
| `PIN_MAP_TYPE_MUX_GROUP` | ربط group بـ mux function |
| `PIN_MAP_TYPE_CONFIGS_PIN` | تطبيق config (pull, drive strength...) على pin واحد |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | تطبيق config على group كاملة |

#### الـ Kconfig Options المؤثرة في الـ Header

| Option | التأثير |
|--------|---------|
| `CONFIG_PINCTRL` | يُفعّل كل الـ subsystem — بدونه كل الدوال بتبقى `inline` فاضية |
| `CONFIG_GENERIC_PINCONF` | يضيف `num_custom_params` و`custom_params` و`custom_conf_items` جوه `pinctrl_desc` — ويضيف `is_generic` جوه `pinconf_ops` |
| `CONFIG_OF` | يُفعّل `of_pinctrl_get()` لجلب الـ controller من device tree |

---

### 1. الـ Structs المهمة

#### `struct pinctrl_pin_desc`

**الغرض:** وصف pin واحد في الـ controller — زي ID card لكل pin.

```c
struct pinctrl_pin_desc {
    unsigned int number;   /* رقم Pin فريد على مستوى النظام كله */
    const char   *name;    /* اسم مقروء للإنسان زي "GPIO0" */
    void         *drv_data;/* بيانات خاصة بالـ driver — الـ core ميلمسهاش */
};
```

**الاتصال بـ structs أخرى:** مصفوفة منهم (`pins[]`) بتتخزن في `pinctrl_desc.pins`.

---

#### `struct pingroup`

**الغرض:** تجميع مجموعة pins تحت اسم واحد — الـ mux بيشتغل على groups مش على pins منفردة في الغالب.

```c
struct pingroup {
    const char         *name;  /* اسم الـ group زي "spi0_grp" */
    const unsigned int *pins;  /* مصفوفة أرقام الـ pins */
    size_t              npins; /* عدد الـ pins في المصفوفة */
};
```

**الاتصال:** الـ driver بيرجع الـ groups دي لما الـ core يستدعي `pctlops->get_group_pins()`.

---

#### `struct pinfunction`

**الغرض:** وصف function واحدة (زي UART أو SPI) — بتعدد الـ groups اللي ممكن تستخدمها.

```c
struct pinfunction {
    const char        *name;    /* اسم الـ function زي "uart0" */
    const char *const *groups;  /* مصفوفة أسماء الـ groups المتاحة */
    size_t             ngroups; /* عدد الـ groups */
    unsigned long      flags;   /* PINFUNCTION_FLAG_GPIO أو 0 */
};
```

**الاتصال:** الـ driver بيرجعها لما الـ core يستدعي `pmxops->get_function_name()` و`get_function_groups()`.

---

#### `struct pinctrl_gpio_range`

**الغرض:** ربط نطاق من GPIO numbers بالـ pin controller — الـ GPIO subsystem بيكلم الـ pinctrl عبر الـ ranges دي.

```c
struct pinctrl_gpio_range {
    struct list_head     node;     /* عقدة في linked list داخل الـ pctldev */
    const char          *name;     /* اسم الـ range */
    unsigned int         id;       /* ID للـ chip */
    unsigned int         base;     /* أول GPIO number في الـ range */
    unsigned int         pin_base; /* أول pin number لو pins == NULL */
    unsigned int         npins;    /* عدد الـ pins/GPIOs */
    unsigned int const  *pins;     /* مصفوفة صريحة لأرقام الـ pins أو NULL */
    struct gpio_chip    *gc;       /* pointer اختياري للـ gpio_chip */
};
```

**الاتصال:** بتتضاف لـ `pinctrl_dev` عبر `pinctrl_add_gpio_range()`. بتربط `gpio_chip` بـ `pinctrl_dev`.

---

#### `struct pinctrl_ops`

**الغرض:** vtable للعمليات الأساسية على الـ pin controller (groups + device tree).

| Function Pointer | الوظيفة | إجبارية؟ |
|-----------------|---------|----------|
| `get_groups_count` | عدد الـ groups الكلي | نعم |
| `get_group_name` | اسم group بـ selector | نعم |
| `get_group_pins` | مصفوفة الـ pins لـ group معينة | نعم |
| `pin_dbg_show` | طباعة debugfs لـ pin | لا |
| `dt_node_to_map` | تحويل device tree node لـ mapping entries | لا (مطلوب مع DT) |
| `dt_free_map` | تحرير الـ mappings اللي اتعملت بـ `dt_node_to_map` | لا (مطلوب مع DT) |

---

#### `struct pinmux_ops`

**الغرض:** vtable لعمليات الـ muxing — اختيار الـ function لكل group من الـ pins.

| Function Pointer | الوظيفة | إجبارية؟ |
|-----------------|---------|----------|
| `request` | تأكيد إن الـ pin متاح للـ mux | لا |
| `free` | تحرير الـ pin من الـ mux | لا |
| `get_functions_count` | عدد الـ functions | نعم |
| `get_function_name` | اسم function بـ selector | نعم |
| `get_function_groups` | الـ groups المرتبطة بـ function | نعم |
| `function_is_gpio` | هل الـ function دي GPIO? | لا |
| `set_mux` | تطبيق الـ mux فعلياً | نعم |
| `gpio_request_enable` | تفعيل GPIO على pin واحد | لا |
| `gpio_disable_free` | تعطيل GPIO | لا |
| `gpio_set_direction` | ضبط اتجاه GPIO (input/output) | لا |
| `strict` | منع تعارض GPIO مع functions أخرى | flag |

---

#### `struct pinconf_ops`

**الغرض:** vtable لعمليات الـ pin configuration (pull-up, drive strength, slew rate...).

| Function Pointer | الوظيفة |
|-----------------|---------|
| `pin_config_get` | قراءة config من pin واحد |
| `pin_config_set` | كتابة config على pin واحد |
| `pin_config_group_get` | قراءة config من group كاملة |
| `pin_config_group_set` | كتابة config على group كاملة |
| `pin_config_dbg_show` | debugfs لـ pin |
| `pin_config_group_dbg_show` | debugfs لـ group |
| `pin_config_config_dbg_show` | فك تشفير قيمة config وطباعتها |

---

#### `struct pinctrl_desc`

**الغرض:** الـ descriptor الرئيسي اللي الـ driver بيملاه وبيسلمه للـ core عند التسجيل — ده "الهوية الكاملة" للـ controller.

```c
struct pinctrl_desc {
    const char                      *name;         /* اسم الـ controller */
    const struct pinctrl_pin_desc   *pins;          /* مصفوفة وصف كل الـ pins */
    unsigned int                     npins;         /* عدد الـ pins */
    const struct pinctrl_ops        *pctlops;       /* vtable الـ groups */
    const struct pinmux_ops         *pmxops;        /* vtable الـ mux */
    const struct pinconf_ops        *confops;       /* vtable الـ config */
    struct module                   *owner;         /* THIS_MODULE للـ refcount */
    /* CONFIG_GENERIC_PINCONF فقط: */
    unsigned int                     num_custom_params;
    const struct pinconf_generic_params *custom_params;
    const struct pin_config_item    *custom_conf_items;

    bool                             link_consumers;/* device link مع الـ consumers */
};
```

**الاتصال:** بيتسلم لـ `pinctrl_register_and_init()` → الـ core بينشئ منه `pinctrl_dev`.

---

#### `struct pinctrl_map` (من `machine.h`)

**الغرض:** جدول الـ mapping — بيربط device معين بـ state معينة (مثلاً "default") بـ mux function أو config.

```c
struct pinctrl_map {
    const char             *dev_name;     /* اسم الـ device المستخدِم */
    const char             *name;         /* اسم الـ state زي "default" */
    enum pinctrl_map_type   type;         /* نوع الـ mapping */
    const char             *ctrl_dev_name;/* اسم الـ pin controller */
    union {
        struct pinctrl_map_mux     mux;    /* لو type == MUX_GROUP */
        struct pinctrl_map_configs configs;/* لو type == CONFIGS_* */
    } data;
};
```

---

### 2. مخطط علاقات الـ Structs

```
                         ┌─────────────────────────────────────────┐
                         │           struct pinctrl_desc            │
                         │  name, npins, link_consumers             │
                         │  pins[]──────────────────────────────────┼──► struct pinctrl_pin_desc[]
                         │  pctlops─────────────────────────────────┼──► struct pinctrl_ops
                         │  pmxops──────────────────────────────────┼──► struct pinmux_ops
                         │  confops─────────────────────────────────┼──► struct pinconf_ops
                         │  owner───────────────────────────────────┼──► struct module
                         └──────────────────┬──────────────────────-┘
                                            │ pinctrl_register_and_init()
                                            ▼
                         ┌─────────────────────────────────────────┐
                         │           struct pinctrl_dev             │
                         │  (opaque — internal to core)             │
                         │  gpio_ranges (list_head)─────────────────┼──► struct pinctrl_gpio_range
                         │                                          │         │
                         └──────────────────────────────────────────┘         ▼
                                                                       struct gpio_chip

  struct pingroup                        struct pinfunction
  ┌───────────────────┐                  ┌───────────────────────┐
  │ name              │                  │ name                  │
  │ pins[]───────────►│pin numbers       │ groups[]─────────────►│group names
  │ npins             │                  │ ngroups               │
  └───────────────────┘                  │ flags (GPIO?)         │
        ▲                                └───────────────────────┘
        │ get_group_pins()                       ▲
        │                                        │ get_function_groups()
        └──────────────────────────────────┐     │
                                   struct pinctrl_ops / pinmux_ops
                                   (function pointers, called by core)

  struct pinctrl_map
  ┌──────────────────────────┐
  │ dev_name                 │──► matches struct device.name
  │ name ("default", "sleep")│
  │ type (MUX / CONFIGS)     │
  │ ctrl_dev_name            │──► matches struct pinctrl_dev.name
  │ data.mux  ───────────────┼──► struct pinctrl_map_mux  { group, function }
  │ data.configs─────────────┼──► struct pinctrl_map_configs { pin/grp, configs[] }
  └──────────────────────────┘
```

---

### 3. مخطط دورة حياة الـ Pin Controller

```
[Driver Probe]
      │
      ▼
  ملء struct pinctrl_desc
  (pins[], pctlops, pmxops, confops, ...)
      │
      ▼
  pinctrl_register_and_init(desc, dev, drv_data, &pctldev)
      │
      ├─► الـ core بيعمل struct pinctrl_dev جوه
      ├─► بيسجل كل الـ pins في radix tree
      ├─► بيعمل debugfs entries
      └─► بيرجع pctldev (مش enabled لسه)
      │
      ▼
  pinctrl_enable(pctldev)
      │
      ├─► بيطبق الـ "hog" mappings (entries بتاعت الـ controller نفسه)
      └─► الـ controller جاهز للاستخدام
      │
      ▼
[Runtime — GPIO / Device Requests]
      │
      ├─► pinctrl_add_gpio_range() لربط GPIO ranges
      ├─► consumer بيطلب state → core بيبحث في pinctrl_map
      │       ├─► MUX: pmxops->set_mux(func, group)
      │       └─► CONFIG: confops->pin_config_set(pin, configs)
      │
      ▼
[Driver Remove]
      │
      ▼
  pinctrl_unregister(pctldev)
      │
      ├─► إزالة GPIO ranges
      ├─► تحرير الـ pins
      ├─► حذف debugfs entries
      └─► تحرير struct pinctrl_dev
```

---

### 4. مخططات الـ Call Flow

#### تسجيل الـ Controller

```
driver->probe()
  └─► pinctrl_register_and_init(desc, dev, drv_data, &pctldev)
        └─► pinctrl_init_controller()        [core/pinctrl.c]
              ├─► kzalloc(struct pinctrl_dev)
              ├─► pctldev->desc = desc
              ├─► pinctrl_register_pins()
              │     └─► for each pin: radix_tree_insert(pin_desc_tree)
              ├─► pinctrl_claim_hogs()        [بيطبق الـ HOG mappings]
              └─► يرجع pctldev

driver->probe()
  └─► pinctrl_enable(pctldev)
        └─► pinctrl_apply_state("default")
```

---

#### طلب GPIO من الـ pinctrl

```
gpio_request(gpio_num)
  └─► gpiochip_request_own_desc()
        └─► pinctrl_gpio_request(gpio)
              └─► pinctrl_find_gpio_range_from_pin(pctldev, pin)
                    └─► list_for_each_entry(range, &pctldev->gpio_ranges)
              └─► pinmux_request_gpio(pctldev, range, pin)
                    └─► pctldev->desc->pmxops->gpio_request_enable(pctldev, range, offset)
                          └─► [hardware register write]
```

---

#### تطبيق State (مثلاً "sleep")

```
device_pm_suspend()
  └─► pinctrl_pm_select_sleep_state(dev)
        └─► pinctrl_select_state(p, state)
              └─── for each setting in state:
                    ├─► [MUX] pinmux_enable_setting()
                    │     └─► pmxops->set_mux(pctldev, func_sel, grp_sel)
                    │           └─► [hardware MUX register]
                    └─► [CONFIG] pinconf_apply_setting()
                          └─► confops->pin_config_set(pctldev, pin, configs, n)
                                └─► [hardware config register]
```

---

#### Device Tree Parsing

```
pinctrl_dt_to_map(p)
  └─► for each "pinctrl-N" property in device node:
        └─► of_pinctrl_get(np) → pctldev
              └─── pctldev->desc->pctlops->dt_node_to_map(pctldev, np, &map, &num_maps)
                    └─► driver parses DT, fills struct pinctrl_map[]
              └─── pinctrl_register_map(map, num_maps)

[on unregister]
  └─► pctlops->dt_free_map(pctldev, map, num_maps)
```

---

### 5. استراتيجية الـ Locking

الـ header نفسه ما بيعرّفش locks صراحةً — ده مسؤولية الـ `pinctrl_dev` الداخلي في `core/pinctrl.c`. لكن من تحليل الـ API:

| المورد | الـ Lock المستخدَم | الوصف |
|--------|-------------------|-------|
| قائمة `gpio_ranges` | `pinctrl_dev->mutex` (mutex داخلي) | بيُأخذ في `pinctrl_add_gpio_range()` و`pinctrl_remove_gpio_range()` |
| الـ pin descriptor tree | `pinctrl_dev->mutex` | بيُأخذ عند البحث والتسجيل |
| الـ state switching | `pinctrl_dev->mutex` | بيمنع تغيير حالتين في نفس الوقت |
| الـ mapping table العالمي | `pinctrl_maps_mutex` (global) | يحمي القائمة العامة لكل الـ mappings |

#### ترتيب الـ Locking (Lock Ordering)

```
pinctrl_maps_mutex          (أعلى مستوى — global)
    └─► pinctrl_dev->mutex  (per-controller)
            └─► gpio_chip->bgpio_lock  (per-chip — لو GPIO)
```

> لا يجوز عكس الترتيب ده تحت أي ظرف — بيسبب deadlock.

#### ملاحظات عملية

- **الـ `pinctrl_ops` callbacks** بيتستدعوا وهو شايل الـ `pinctrl_dev->mutex` — يعني الـ driver ما يحاولش يأخذ نفس الـ lock من جوا الـ callback.
- **الـ `dt_node_to_map`** بيتستدعى من سياق probe (ممكن يتعمل allocation) — مسموح بـ `GFP_KERNEL`.
- **الـ `gpio_request_enable`** ممكن يتستدعى من سياق atomic لو الـ GPIO core طلب كده — الـ driver لازم يراعي ده.
## Phase 4: شرح الـ Functions

---

### ملخص كل الـ Functions والـ APIs — Cheatsheet

#### Registration & Lifecycle

| Function | النوع | الغرض |
|---|---|---|
| `pinctrl_register_and_init()` | extern int | تسجيل الـ controller + تهيئته بدون تفعيل |
| `pinctrl_enable()` | extern int | تفعيل الـ controller بعد التهيئة |
| `pinctrl_register()` | extern struct pinctrl_dev* | legacy registration (deprecated) |
| `pinctrl_unregister()` | extern void | إلغاء تسجيل الـ controller |
| `devm_pinctrl_register_and_init()` | extern int | نسخة devm من register_and_init |
| `devm_pinctrl_register()` | extern struct pinctrl_dev* | legacy devm registration (deprecated) |

#### GPIO Range Management

| Function | النوع | الغرض |
|---|---|---|
| `pinctrl_add_gpio_range()` | extern void | إضافة GPIO range واحد |
| `pinctrl_add_gpio_ranges()` | extern void | إضافة مصفوفة من الـ GPIO ranges |
| `pinctrl_remove_gpio_range()` | extern void | حذف GPIO range |
| `pinctrl_find_and_add_gpio_range()` | extern struct pinctrl_dev* | بحث عن controller وإضافة range |
| `pinctrl_find_gpio_range_from_pin()` | extern struct pinctrl_gpio_range* | إيجاد الـ range اللي بيغطي pin معين |

#### Query & Accessors

| Function | النوع | الغرض |
|---|---|---|
| `pinctrl_get_group_pins()` | extern int | استرجاع الـ pins الخاصة بـ group |
| `pinctrl_dev_get_name()` | extern const char* | اسم الـ pin controller |
| `pinctrl_dev_get_devname()` | extern const char* | اسم الـ underlying device |
| `pinctrl_dev_get_drvdata()` | extern void* | driver private data pointer |
| `of_pinctrl_get()` | extern struct pinctrl_dev* | lookup controller من device tree node |

#### Machine-level Mapping (من pinctrl/machine.h)

| Function | النوع | الغرض |
|---|---|---|
| `pinctrl_register_mappings()` | int | تسجيل mapping table statically |
| `devm_pinctrl_register_mappings()` | int | نسخة devm من register_mappings |
| `pinctrl_unregister_mappings()` | void | إلغاء تسجيل الـ mapping table |
| `pinctrl_provide_dummies()` | void | تفعيل dummy fallback لما pinctrl مش موجود |

#### vtable ops داخل الـ structs

| Callback | الـ struct | الغرض |
|---|---|---|
| `get_groups_count` | `pinctrl_ops` | عدد الـ groups |
| `get_group_name` | `pinctrl_ops` | اسم الـ group بالـ selector |
| `get_group_pins` | `pinctrl_ops` | مصفوفة الـ pins في group |
| `pin_dbg_show` | `pinctrl_ops` | debugfs hook لكل pin |
| `dt_node_to_map` | `pinctrl_ops` | تحويل DT node لـ mapping entries |
| `dt_free_map` | `pinctrl_ops` | تحرير الـ map entries المُولَّدة من DT |

---

### المجموعة الأولى: Registration & Lifecycle

هذه المجموعة هي نقطة الدخول لأي pin controller driver. الـ driver بيسجل نفسه مع الـ pinctrl core من خلالها، وبيحدد الـ `pinctrl_desc` اللي بيوصف كل الـ pins والـ ops الخاصة بيه. الـ two-step approach الجديد (register_and_init ثم enable) بيسمح بتأجيل التفعيل لحد ما الـ driver يجهز كل حاجة.

---

#### `pinctrl_register_and_init()`

```c
extern int pinctrl_register_and_init(const struct pinctrl_desc *pctldesc,
                                     struct device *dev,
                                     void *driver_data,
                                     struct pinctrl_dev **pctldev);
```

بتسجل الـ pin controller مع الـ pinctrl core وبتهيئه داخلياً، لكن من غير ما تفعّل الـ hog states أو تعمل pin requests. بتخزن الـ `pctldev` pointer في المتغير اللي بتشير إليه `**pctldev`. لازم تيجي بعدها `pinctrl_enable()` عشان يكتمل التسجيل ويبدأ الـ hogging.

**Parameters:**
- `pctldesc` — وصف الـ controller: اسمه، الـ pins، الـ ops vtables، الـ owner module
- `dev` — الـ `struct device` الخاص بالـ controller hardware
- `driver_data` — opaque pointer للـ driver private state، بترجعه `pinctrl_dev_get_drvdata()`
- `pctldev` — output parameter: العنوان اللي هيتحط فيه الـ `pinctrl_dev` الجديد

**Return:** `0` عند النجاح، negative errno عند الفشل (مثلاً `-ENOMEM` لو فيه مشكلة في الـ allocation).

**Key details:**
- مش بتعمل hog للـ states — ده بيحصل في `pinctrl_enable()`.
- آمنة للاستخدام في وقت الـ probe لو الـ driver محتاج يعمل setup بعد التسجيل وقبل الـ hogging.
- الـ `pinctrl_dev` الـ internal structure بتتعمل allocate وبتتربط بالـ `dev`.

**Caller context:** Driver probe function، process context فقط.

---

#### `pinctrl_enable()`

```c
extern int pinctrl_enable(struct pinctrl_dev *pctldev);
```

بتكمل عملية التسجيل اللي بدأتها `pinctrl_register_and_init()`. بتفعّل الـ controller وبتعمل hog للـ pin states اللي الـ controller بيحتاجها لنفسه (pin maps اللي `ctrl_dev_name == dev_name`).

**Parameters:**
- `pctldev` — الـ handle اللي رجعته `pinctrl_register_and_init()` في الـ output parameter

**Return:** `0` عند النجاح، negative errno لو الـ hogging فشل.

**Key details:**
- لو `pinctrl_register_and_init()` نجحت بس `pinctrl_enable()` فشلت، الـ driver لازم يعمل cleanup مناسب.
- الـ hogging معناه إن الـ controller بنفسه بيطلب بعض الـ pins لنفسه (مثلاً SMBus pins لـ I2C controller داخلي).

**Caller context:** Process context، بعد `pinctrl_register_and_init()` مباشرةً.

---

#### `pinctrl_register()` *(deprecated)*

```c
extern struct pinctrl_dev *pinctrl_register(const struct pinctrl_desc *pctldesc,
                                             struct device *dev,
                                             void *driver_data);
```

الـ legacy API القديم اللي بيجمع التسجيل والتفعيل في خطوة واحدة. مش موصى بيه في الكود الجديد — استخدم `pinctrl_register_and_init()` + `pinctrl_enable()` بدلاً منه.

**Parameters:** زي `pinctrl_register_and_init()` بالضبط، بس من غير output parameter.

**Return:** `struct pinctrl_dev *` عند النجاح، `ERR_PTR(errno)` عند الفشل — لازم تفحص بـ `IS_ERR()`.

**Key details:** بيعمل hog في نفس الـ call، مما بيعمل مشكلة لو الـ driver محتاج initialization بعد التسجيل.

---

#### `pinctrl_unregister()`

```c
extern void pinctrl_unregister(struct pinctrl_dev *pctldev);
```

بتعكس عملية `pinctrl_register()` أو `pinctrl_register_and_init()` + `pinctrl_enable()`. بتحرر كل الـ resources المرتبطة بالـ controller وبتشيله من الـ global list.

**Parameters:**
- `pctldev` — الـ handle الراجع من التسجيل

**Return:** `void`

**Key details:**
- بتعمل release للـ hog states أولاً.
- بتتأكد إن مفيش device تانية لسه بتستخدم pins من الـ controller ده.
- في حالة `devm_*`، ده بيتنادى أوتوماتيك عند الـ device removal.

**Caller context:** Driver remove/unbind function، process context.

---

#### `devm_pinctrl_register_and_init()`

```c
extern int devm_pinctrl_register_and_init(struct device *dev,
                                           const struct pinctrl_desc *pctldesc,
                                           void *driver_data,
                                           struct pinctrl_dev **pctldev);
```

نسخة resource-managed من `pinctrl_register_and_init()`. بتسجل cleanup action مع الـ devres framework، وبكده `pinctrl_unregister()` بتتنادى أوتوماتيك لما الـ `dev` بيتحرر.

**Parameters:** زي `pinctrl_register_and_init()` بالضبط — `dev` هنا بيعمل دور الـ devres owner.

**Return:** `0` أو negative errno.

**Key details:**
- الـ preferred method في الـ modern kernel drivers عشان بتتجنب الـ manual cleanup في error paths.
- لازم يتبعها `pinctrl_enable()` زي النسخة العادية.

---

#### `devm_pinctrl_register()` *(deprecated)*

```c
extern struct pinctrl_dev *devm_pinctrl_register(struct device *dev,
                                                  const struct pinctrl_desc *pctldesc,
                                                  void *driver_data);
```

الـ legacy devm version من `pinctrl_register()`. نفس التحذيرات بتنطبق عليها — استخدم `devm_pinctrl_register_and_init()` بدلاً منها.

---

### المجموعة التانية: GPIO Range Management

الـ pinctrl core محتاج يعرف أي pins بتقابل أي GPIO numbers عشان يقدر يعمل coordination بين الـ gpio subsystem والـ pinmux. الـ `pinctrl_gpio_range` بيعمل هذا الـ mapping.

```
GPIO number space:
  [base ... base+npins-1]
       |
       v (via pinctrl_gpio_range)
  [pin_base ... pin_base+npins-1]  في الـ pin number space
```

---

#### `pinctrl_add_gpio_range()`

```c
extern void pinctrl_add_gpio_range(struct pinctrl_dev *pctldev,
                                    struct pinctrl_gpio_range *range);
```

بتضيف `pinctrl_gpio_range` واحد لقايمة الـ GPIO ranges الخاصة بالـ pin controller. الـ core بيستخدم هذا الـ range لما GPIO driver بيطلب pin أو بيعمل mux له.

**Parameters:**
- `pctldev` — الـ pin controller المالك للـ range ده
- `range` — وصف الـ range: base GPIO، base pin، عدد الـ pins، وـ optional gpio_chip pointer

**Return:** `void`

**Key details:**
- الـ `range->node` بيتضاف على `pctldev->gpio_ranges` list.
- الـ range structure لازم تفضل valid طول ما الـ controller مسجل.
- بتاخد `pctldev->lock` داخلياً.

**Caller context:** عادةً في probe أو في `gpio_chip` registration callback، process context.

---

#### `pinctrl_add_gpio_ranges()`

```c
extern void pinctrl_add_gpio_ranges(struct pinctrl_dev *pctldev,
                                     struct pinctrl_gpio_range *ranges,
                                     unsigned int nranges);
```

بتضيف مصفوفة من الـ `pinctrl_gpio_range` دفعةً واحدة. مجرد loop بتنادي `pinctrl_add_gpio_range()` على كل عنصر.

**Parameters:**
- `pctldev` — الـ pin controller
- `ranges` — مصفوفة الـ range descriptors
- `nranges` — عدد العناصر في المصفوفة

**Return:** `void`

**Key details:** ملاءمة لما الـ controller بيدعم أكتر من GPIO bank في نفس الوقت.

---

#### `pinctrl_remove_gpio_range()`

```c
extern void pinctrl_remove_gpio_range(struct pinctrl_dev *pctldev,
                                       struct pinctrl_gpio_range *range);
```

بتشيل الـ range من قايمة الـ GPIO ranges في الـ controller. لازم تتنادى قبل تحرير الـ range structure لو كانت مضافة قبل كده.

**Parameters:**
- `pctldev` — الـ pin controller
- `range` — الـ range المراد إزالته

**Return:** `void`

**Key details:** بتاخد `pctldev->lock`، بتعمل `list_del()` للـ `range->node`.

---

#### `pinctrl_find_and_add_gpio_range()`

```c
extern struct pinctrl_dev *pinctrl_find_and_add_gpio_range(const char *devname,
                                                            struct pinctrl_gpio_range *range);
```

بتدور على الـ pin controller المسجل باسمه `devname` وبتضيف الـ range له. مفيدة لما GPIO driver محتاج يربط نفسه بالـ pin controller من غير ما يعرف الـ `pctldev` pointer مسبقاً.

**Parameters:**
- `devname` — اسم الـ device (كما في `dev_name()`) للـ pin controller
- `range` — الـ GPIO range المراد إضافته

**Return:** الـ `pinctrl_dev *` المُوجَد عند النجاح، `ERR_PTR(-EPROBE_DEFER)` لو الـ controller لسه مش مسجل — الـ caller يعمل deferred probe.

**Key details:**
- استخدام `-EPROBE_DEFER` هنا مهم جداً لحل الـ probe ordering issues بين GPIO و pinctrl drivers.
- الـ lookup بيتعمل على الـ global `pinctrldev_list`.

**Pseudocode:**
```c
// find_and_add_gpio_range pseudocode
pctldev = get_pinctrl_dev_from_devname(devname);
if (!pctldev)
    return ERR_PTR(-EPROBE_DEFER);
pinctrl_add_gpio_range(pctldev, range);
return pctldev;
```

---

#### `pinctrl_find_gpio_range_from_pin()`

```c
extern struct pinctrl_gpio_range *
pinctrl_find_gpio_range_from_pin(struct pinctrl_dev *pctldev,
                                  unsigned int pin);
```

بتبحث في قايمة الـ GPIO ranges عن الـ range اللي بيحتوي على الـ pin number المحدد.

**Parameters:**
- `pctldev` — الـ pin controller المراد البحث فيه
- `pin` — رقم الـ pin في الـ pin number space

**Return:** `struct pinctrl_gpio_range *` إذا وُجد، `NULL` إذا لم يكن الـ pin ضمن أي range.

**Key details:** بتعمل iterate على `pctldev->gpio_ranges` وبتقارن `pin_base` و `npins`. مهمة لعمليات الـ GPIO mux في الـ pinmux core.

---

### المجموعة التالتة: Query & Accessor Functions

هذه الـ functions بتوفر read-only access لمعلومات الـ pin controller بعد تسجيله. مفيدة للـ subsystems التانية والـ debugfs code.

---

#### `pinctrl_get_group_pins()`

```c
extern int pinctrl_get_group_pins(struct pinctrl_dev *pctldev,
                                   const char *pin_group,
                                   const unsigned int **pins,
                                   unsigned int *num_pins);
```

بترجع مصفوفة الـ pin numbers الخاصة بـ group معين باسمه. بتستخدم الـ `pinctrl_ops->get_group_pins` callback داخلياً بعد ما بتلاقي الـ group selector من اسمه.

**Parameters:**
- `pctldev` — الـ pin controller
- `pin_group` — اسم الـ group المطلوب
- `pins` — output: pointer لمصفوفة الـ pin numbers (مملوكة للـ driver، مش بتتحرر)
- `num_pins` — output: عدد الـ pins في المصفوفة

**Return:** `0` عند النجاح، `-EINVAL` لو الـ group مش موجود أو الـ ops مش موجودة.

**Key details:**
- الـ `*pins` بتشير لـ internal driver data — الـ caller مش مسؤول عن تحريرها.
- بتعمل lookup بـ name-to-selector أولاً باستخدام `pinctrl_get_group_selector()`.

---

#### `pinctrl_dev_get_name()`

```c
extern const char *pinctrl_dev_get_name(struct pinctrl_dev *pctldev);
```

بترجع اسم الـ pin controller descriptor (الـ `pinctrl_desc->name`).

**Parameters:**
- `pctldev` — الـ handle

**Return:** const string pointer — لا يُحرر.

**Key details:** مفيدة للـ logging والـ sysfs/debugfs display.

---

#### `pinctrl_dev_get_devname()`

```c
extern const char *pinctrl_dev_get_devname(struct pinctrl_dev *pctldev);
```

بترجع `dev_name()` للـ underlying `struct device` — يعني الاسم كما يظهر في `/sys/bus/platform/devices/`.

**Parameters:**
- `pctldev` — الـ handle

**Return:** const string pointer — لا يُحرر.

**Key details:** ده الاسم المستخدم في الـ `pinctrl_map->ctrl_dev_name` عشان ربط الـ mapping بالـ controller.

---

#### `pinctrl_dev_get_drvdata()`

```c
extern void *pinctrl_dev_get_drvdata(struct pinctrl_dev *pctldev);
```

بترجع الـ `driver_data` pointer اللي اتبعت في `pinctrl_register_and_init()`. هي الطريقة الوحيدة لأي callback في الـ `pinctrl_ops` / `pinmux_ops` / `pinconf_ops` عشان يوصل لـ driver private state.

**Parameters:**
- `pctldev` — الـ handle

**Return:** `void *` — الـ driver بيعمله cast لـ private struct.

**Key details:**

```c
/* مثال نموذجي داخل أي pinctrl callback */
static int mydrv_get_groups_count(struct pinctrl_dev *pctldev)
{
    struct mydrv_priv *priv = pinctrl_dev_get_drvdata(pctldev);
    return priv->ngroups;
}
```

---

#### `of_pinctrl_get()`

```c
/* متاحة فقط لو CONFIG_OF && CONFIG_PINCTRL */
extern struct pinctrl_dev *of_pinctrl_get(struct device_node *np);
```

بتعمل lookup للـ pin controller المرتبط بـ device tree node معين. بتبحث في الـ registered controllers عن اللي `pctldev->dev->of_node == np`.

**Parameters:**
- `np` — الـ `device_node` المراد البحث عن controller مرتبط بيه

**Return:** `struct pinctrl_dev *` لو وُجد، `NULL` لو لأ.

**Key details:**
- متاحة فقط لو `CONFIG_OF` و`CONFIG_PINCTRL` enabled، وإلا بترجع `NULL` always (inline stub).
- مستخدمة داخلياً من قِبل `pinctrl_get()` في `drivers/pinctrl/core.c` لما بيعمل resolve لـ `phandle` في device tree.

---

### المجموعة الرابعة: Machine-level Mapping Functions

هذه الـ functions بتسجل الـ mapping table — الجدول اللي بيربط كل device بـ pin states محددة على controller محدد. في أنظمة قديمة (non-DT)، الـ mapping بيتحدد statically في الـ board file.

---

#### `pinctrl_register_mappings()`

```c
int pinctrl_register_mappings(const struct pinctrl_map *map,
                               unsigned int num_maps);
```

بتسجل مصفوفة من `pinctrl_map` entries مع الـ pinctrl core. كل entry بتقول: "الـ device ده في الـ state دي يستخدم الـ group ده مع الـ function ده على الـ controller ده".

**Parameters:**
- `map` — مصفوفة static من `pinctrl_map` structures (لازم تفضل valid طول عمر الـ kernel)
- `num_maps` — عدد الـ entries في المصفوفة

**Return:** `0` عند النجاح، negative errno عند الفشل.

**Key details:**
- الـ core بيعمل copy للـ array pointer بس مش للـ data — الـ `map` لازم تكون static.
- بتُستخدم في `board_init()` أو `machine_init()` قبل الـ device registration.
- لو `CONFIG_PINCTRL` مش enabled، بترجع `0` automatically (no-op inline).

**مثال:**
```c
static struct pinctrl_map myboard_maps[] = {
    PIN_MAP_MUX_GROUP_DEFAULT("ssp0", "pinctrl.0", "ssp0_grp", "ssp0"),
    PIN_MAP_CONFIGS_PIN_DEFAULT("ssp0", "pinctrl.0", "PA0", uart_configs),
};

/* في board init */
pinctrl_register_mappings(myboard_maps, ARRAY_SIZE(myboard_maps));
```

---

#### `devm_pinctrl_register_mappings()`

```c
int devm_pinctrl_register_mappings(struct device *dev,
                                    const struct pinctrl_map *map,
                                    unsigned int num_maps);
```

نسخة resource-managed. بتسجل cleanup action مع `dev` بحيث `pinctrl_unregister_mappings()` بتتنادى أوتوماتيك لما `dev` يُحرر.

**Parameters:**
- `dev` — الـ owner device للـ devres action
- `map`, `num_maps` — زي `pinctrl_register_mappings()`

**Return:** `0` أو negative errno.

---

#### `pinctrl_unregister_mappings()`

```c
void pinctrl_unregister_mappings(const struct pinctrl_map *map);
```

بتشيل الـ mapping table المسجلة من الـ core. بتستخدم الـ pointer نفسه كـ key.

**Parameters:**
- `map` — نفس الـ pointer اللي اتبعت في `pinctrl_register_mappings()`

**Return:** `void`

---

#### `pinctrl_provide_dummies()`

```c
void pinctrl_provide_dummies(void);
```

بتفعّل "dummy mode" في الـ pinctrl core. في هذا الـ mode، لو device طلب pin state ومفيش controller موجود، الـ core بيرجع success بدلاً من error.

**Parameters:** لا يوجد

**Return:** `void`

**Key details:**
- مفيدة جداً في environments زي QEMU أو early bringup systems حيث الـ pinctrl hardware مش موجود فعلياً.
- بتُنادى عادةً في الـ `machine_init()` على hardware اللي مش بيدعم pinctrl.
- بعد تفعيلها، `pinctrl_get()` لا بيفشل حتى لو مفيش controller مسجل.

---

### المجموعة الخامسة: الـ vtable Callbacks في `pinctrl_ops`

الـ `pinctrl_ops` هو الـ vtable الأساسي اللي كل pin controller driver لازم (أو يُستحسن) يـ implement. بيعطي الـ core القدرة على التعامل مع مفهوم الـ groups.

---

#### `get_groups_count`

```c
int (*get_groups_count)(struct pinctrl_dev *pctldev);
```

بترجع العدد الكلي للـ pin groups المسجلة في الـ controller. الـ core بيستخدم هذا العدد كحد أقصى للـ `selector` في الـ calls التانية.

**Return:** عدد صحيح موجب >= 0، مفيش error return.

**Caller:** `pinctrl_core` أثناء الـ registration validation وفي sysfs/debugfs.

---

#### `get_group_name`

```c
const char *(*get_group_name)(struct pinctrl_dev *pctldev,
                               unsigned int selector);
```

بترجع اسم الـ group ذو الـ index `selector`. الـ core بيعمل iterate على كل الـ groups بـ `selector` من `0` لـ `get_groups_count()-1` لبناء الـ internal name-to-selector mapping.

**Parameters:**
- `selector` — index من `0` لـ `count-1`

**Return:** const string، لا يُحرر.

---

#### `get_group_pins`

```c
int (*get_group_pins)(struct pinctrl_dev *pctldev,
                      unsigned int selector,
                      const unsigned int **pins,
                      unsigned int *num_pins);
```

بترجع مصفوفة الـ pin numbers الخاصة بالـ group. الـ core بيستخدمها لما بيعمل mux لـ function على group معين — عشان يتأكد إن الـ pins مش في conflict مع حاجة تانية.

**Parameters:**
- `selector` — index الـ group
- `pins` — output: pointer لمصفوفة الـ pin numbers (owned by driver)
- `num_pins` — output: عدد الـ pins

**Return:** `0` عند النجاح، negative errno لو الـ selector غلط.

---

#### `pin_dbg_show`

```c
void (*pin_dbg_show)(struct pinctrl_dev *pctldev,
                     struct seq_file *s,
                     unsigned int offset);
```

optional callback للـ debugfs. لما المستخدم بيفتح `/sys/kernel/debug/pinctrl/<controller>/pins`، الـ core بينادي هذا الـ callback لكل pin عشان يطبع معلومات إضافية خاصة بالـ hardware (مثلاً: current mux state، drive strength، pull config).

**Parameters:**
- `s` — الـ `seq_file` للكتابة فيه باستخدام `seq_printf()`
- `offset` — رقم الـ pin

**Return:** `void`

---

#### `dt_node_to_map`

```c
int (*dt_node_to_map)(struct pinctrl_dev *pctldev,
                      struct device_node *np_config,
                      struct pinctrl_map **map,
                      unsigned int *num_maps);
```

أهم callback في الـ DT-based systems. بيحوّل device tree "pin configuration node" لمصفوفة من `pinctrl_map` entries. كل driver بيعمل parse لـ DT properties الخاصة بيه (زي `pinmux`، `bias-pull-up`، `drive-strength`) وبيولّد الـ map entries المناسبة.

**Parameters:**
- `np_config` — الـ DT node اللي بيحتوي على pin configuration (الـ child node في pinctrl DT binding)
- `map` — output: مصفوفة الـ `pinctrl_map` المُولَّدة (allocated بواسطة الـ driver)
- `num_maps` — output: عدد الـ entries

**Return:** `0` عند النجاح، negative errno لو الـ DT data غلطانة.

**Key details:**
- الـ driver لازم يعمل allocate الـ map array ومحتوياتها — الـ core مش بيعمل allocate.
- `dt_free_map` لازم تقابلها — الـ core بينادي `dt_free_map` لما بيخلص من الـ map.
- الـ call tree: `pinctrl_dt_to_map()` → بيعمل iterate على DT children → بينادي `dt_node_to_map` لكل child.

**Pseudocode مبسط:**
```c
// مثال مبسط لـ dt_node_to_map implementation
int mydrv_dt_node_to_map(struct pinctrl_dev *pctldev,
                          struct device_node *np,
                          struct pinctrl_map **map,
                          unsigned int *num_maps)
{
    // 1. count properties to determine number of maps needed
    // 2. allocate map array
    *map = devm_kcalloc(pctldev->dev, count, sizeof(**map), GFP_KERNEL);

    // 3. parse each pin/group property
    //    - PIN_MAP_TYPE_MUX_GROUP for mux settings
    //    - PIN_MAP_TYPE_CONFIGS_PIN/GROUP for electrical settings
    (*map)[i].type = PIN_MAP_TYPE_MUX_GROUP;
    (*map)[i].data.mux.group = group_name;
    (*map)[i].data.mux.function = func_name;

    *num_maps = count;
    return 0;
}
```

---

#### `dt_free_map`

```c
void (*dt_free_map)(struct pinctrl_dev *pctldev,
                    struct pinctrl_map *map,
                    unsigned int num_maps);
```

بتحرر الـ map array المُولَّدة من `dt_node_to_map`. الـ driver لازم يحرر أي dynamically allocated strings أو configs داخل الـ map entries، ثم يحرر الـ top-level array.

**Parameters:**
- `map` — الـ map array المراد تحريرها
- `num_maps` — عدد الـ entries

**Return:** `void`

**Key details:** لو الـ driver بيستخدم `devm_*` allocations في `dt_node_to_map`، ممكن تكون هذه الـ function جسم فاضي — الـ devm هيعمل الـ cleanup أوتوماتيك.

---

### ملاحظات على الـ Macros

#### `PINCTRL_PIN(a, b)` و `PINCTRL_PIN_ANON(a)`

```c
#define PINCTRL_PIN(a, b)   { .number = a, .name = b }
#define PINCTRL_PIN_ANON(a) { .number = a }
```

بيعملوا initialize لـ `pinctrl_pin_desc` بشكل مختصر. بيُستخدموا عادةً في تعريف الـ static pin table:

```c
static const struct pinctrl_pin_desc mydrv_pins[] = {
    PINCTRL_PIN(0, "PA0"),
    PINCTRL_PIN(1, "PA1"),
    PINCTRL_PIN_ANON(2),   /* pin بدون اسم */
};
```

#### `PINCTRL_PINGROUP(_name, _pins, _npins)`

```c
#define PINCTRL_PINGROUP(_name, _pins, _npins) \
(struct pingroup) { .name = _name, .pins = _pins, .npins = _npins }
```

بيعمل compound literal لـ `pingroup`. بيُستخدم في تعريف الـ groups table:

```c
static const struct pingroup mydrv_groups[] = {
    PINCTRL_PINGROUP("spi0_grp", spi0_pins, ARRAY_SIZE(spi0_pins)),
    PINCTRL_PINGROUP("i2c1_grp", i2c1_pins, ARRAY_SIZE(i2c1_pins)),
};
```

#### `PINCTRL_PINFUNCTION` و `PINCTRL_GPIO_PINFUNCTION`

```c
#define PINCTRL_PINFUNCTION(_name, _groups, _ngroups) \
(struct pinfunction) { .name = _name, .groups = _groups, .ngroups = _ngroups }

#define PINCTRL_GPIO_PINFUNCTION(_name, _groups, _ngroups) \
(struct pinfunction) { ..., .flags = PINFUNCTION_FLAG_GPIO }
```

الفرق الوحيد هو `PINFUNCTION_FLAG_GPIO` في الـ flags. الـ core بيستخدم هذا الـ flag مع الـ `strict` mode في `pinmux_ops` عشان يمنع التعارض بين GPIO access والـ mux functions التانية.

---

### تدفق الـ Registration الكامل — Pseudocode

```
Driver probe():
  1. allocate priv struct
  2. init hardware
  3. fill pinctrl_desc {
       .name = "mydrv-pinctrl",
       .pins = mydrv_pins,
       .npins = ARRAY_SIZE(mydrv_pins),
       .pctlops = &mydrv_pinctrl_ops,
       .pmxops  = &mydrv_pinmux_ops,
       .confops = &mydrv_pinconf_ops,
       .owner   = THIS_MODULE,
     }
  4. devm_pinctrl_register_and_init(dev, &desc, priv, &pctldev)
       → pinctrl core allocates pinctrl_dev
       → validates desc
       → registers pins in radix tree
       → registers in global pinctrldev_list
       → does NOT yet hog maps
  5. pinctrl_add_gpio_ranges(pctldev, ranges, nranges)
  6. pinctrl_enable(pctldev)
       → finds maps where ctrl_dev_name == dev_name(dev)
       → applies them (hog)
       → controller is now fully operational
```

---

### علاقات الـ Structs ببعض — ASCII Diagram

```
pinctrl_desc
  ├── pins[]          → pinctrl_pin_desc[]
  ├── pctlops         → pinctrl_ops vtable
  │     ├── get_groups_count
  │     ├── get_group_name
  │     ├── get_group_pins
  │     ├── pin_dbg_show
  │     ├── dt_node_to_map
  │     └── dt_free_map
  ├── pmxops          → pinmux_ops vtable
  └── confops         → pinconf_ops vtable

pinctrl_dev  (core-internal, allocated by core)
  ├── desc            → pinctrl_desc (above)
  ├── driver_data     → returned by pinctrl_dev_get_drvdata()
  ├── dev             → struct device
  └── gpio_ranges     → list of pinctrl_gpio_range

pinctrl_gpio_range
  ├── base            → first GPIO number
  ├── pin_base        → first pin number
  ├── npins           → count
  └── gc              → gpio_chip (optional)
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. Debugfs — الـ Entries والقراءة

الـ pinctrl subsystem بيعمل mount تلقائي تحت `/sys/kernel/debug/pinctrl/` لما يتفعل `CONFIG_DEBUG_FS`.

```
/sys/kernel/debug/pinctrl/
├── pinctrl-devices          ← قايمة كل الـ pin controllers المسجلين
├── pinctrl-handles          ← كل الـ handles الحالية (consumers + states)
├── pinctrl-maps             ← الـ mapping table كاملة
└── <controller-name>/
    ├── pins                 ← كل الـ pins مع أرقامهم وأسماءهم
    ├── pingroups            ← كل الـ pingroups وأعضاءهم
    ├── pinmux-functions     ← الـ functions المتاحة وارتباطها بالـ groups
    ├── pinmux-pins          ← الـ pin لكل function مضبوطة حالياً
    └── pinconf-pins         ← الـ config الحالية لكل pin (pull, drive, etc.)
```

**كيفية القراءة:**

```bash
# اعرض كل الـ pin controllers المسجلين في النظام
cat /sys/kernel/debug/pinctrl/pinctrl-devices

# اعرض الـ pingroups وأعضاءهم لـ controller معين
# غيّر "soc-pinctrl" باسم الـ controller على جهازك
cat /sys/kernel/debug/pinctrl/soc-pinctrl/pingroups

# مثال output:
# registered pin groups:
# group: uart0_grp
# pin 10 (UART0_TX)
# pin 11 (UART0_RX)

# اعرض الـ pin مضبوطة على أي function دلوقتي
cat /sys/kernel/debug/pinctrl/soc-pinctrl/pinmux-pins

# مثال output:
# pin 10 (UART0_TX): uart@12340000 (GPIO UNCLAIMED)
# pin 11 (UART0_RX): uart@12340000 (GPIO UNCLAIMED)

# اعرض الـ config لكل pin (pull-up/down, drive strength)
cat /sys/kernel/debug/pinctrl/soc-pinctrl/pinconf-pins

# اعرض كل الـ mapping entries المسجلة
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# اعرض كل الـ handles الحالية (device + state المختارة)
cat /sys/kernel/debug/pinctrl/pinctrl-handles
```

**مثال على output من `pinctrl-handles`:**

```
Requested pin control handlers:
device: uart@12340000 current state: default
  state: default
    type: MUX_GROUP controller soc-pinctrl group: uart0_grp function: uart0
    type: CONFIGS_GROUP controller soc-pinctrl group: uart0_grp
      config 00000001
```

---

#### 2. Sysfs — الـ Entries المفيدة

الـ pinctrl subsystem نفسه ما بيكشف entries كتير في `/sys/bus/`، لكن:

```
/sys/bus/platform/drivers/         ← هنا بتلاقي الـ pin controller driver
/sys/devices/<platform-bus>/<ctrl-dev>/
    ├── driver/                    ← link للـ driver
    ├── of_node -> ...             ← link للـ DT node
    └── pinctrl/                   ← (بعض الـ SoCs بتضيف entries هنا)
```

**للـ GPIO range integration:**

```bash
# اعرض GPIO chips وارتباطها بالـ pinctrl ranges
ls /sys/class/gpio/
cat /sys/class/gpio/gpiochipX/label
cat /sys/class/gpio/gpiochipX/base
cat /sys/class/gpio/gpiochipX/ngpio

# اعرض الـ device الـ consumer بتاعتها للـ pin
ls /sys/devices/platform/<device>/
```

---

#### 3. Ftrace — الـ Tracepoints والـ Events

الـ pinctrl subsystem بيوفر tracepoints مخصوصة تحت `pinctrl`:

```bash
# اعرض كل الـ events المتاحة للـ pinctrl
ls /sys/kernel/tracing/events/pinctrl/

# الـ events الأساسية:
# pinctrl_state_lookup   — بيتتبع طلب الـ state lookup
# pinctrl_state_select   — بيتتبع تطبيق الـ state
```

**تفعيل الـ tracing:**

```bash
# فعّل كل events الـ pinctrl
echo 1 > /sys/kernel/tracing/events/pinctrl/enable

# أو event بعينه
echo 1 > /sys/kernel/tracing/events/pinctrl/pinctrl_state_select/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل الـ device أو reproduce المشكلة هنا...

# وقف الـ trace واقرأ النتيجة
echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace

# مثال output:
# <...>-123 [000] .... 1234.567890: pinctrl_state_select: pinctrl_name="uart@12340000" state_name="default"
```

**لتتبع استدعاءات الـ functions داخلياً:**

```bash
# استخدام function tracer للـ pinctrl core
echo function > /sys/kernel/tracing/current_tracer
echo 'pinctrl_*' > /sys/kernel/tracing/set_ftrace_filter
echo 1 > /sys/kernel/tracing/tracing_on
# ... trigger the event ...
cat /sys/kernel/tracing/trace
```

---

#### 4. Printk والـ Dynamic Debug

**تفعيل dynamic debug للـ pinctrl:**

```bash
# فعّل كل رسائل الـ debug في ملفات الـ pinctrl core
echo 'file drivers/pinctrl/core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinconf.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/devicetree.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل رسائل الـ debug في الـ pinctrl module كله
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل لـ driver معين (مثلاً pinctrl-bcm2835)
echo 'module pinctrl_bcm2835 +p' > /sys/kernel/debug/dynamic_debug/control

# أضف timestamp وline number للرسائل
echo 'file drivers/pinctrl/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
```

**من kernel command line (لمشاكل boot-time):**

```
dyndbg="file drivers/pinctrl/core.c +p; file drivers/pinctrl/devicetree.c +p"
```

**رفع مستوى الـ loglevel مؤقتاً:**

```bash
# اعرض الـ console loglevel الحالية
cat /proc/sys/kernel/printk

# ارفعها لـ 8 (DEBUG) مؤقتاً
echo 8 > /proc/sys/kernel/printk

# أو باستخدام dmesg
dmesg -n 8
```

---

#### 5. Kernel Config Options للـ Debugging

| **Option** | **وظيفتها** |
|---|---|
| `CONFIG_PINCTRL` | تفعيل الـ subsystem نفسه |
| `CONFIG_DEBUG_FS` | لازم يكون مفعّل عشان debugfs يشتغل |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | دعم الـ generic group management |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | دعم الـ generic function management |
| `CONFIG_GENERIC_PINCONF` | دعم الـ generic pin configuration |
| `CONFIG_DEBUG_PINCTRL` | يفعّل traces وchecks إضافية داخل الـ core |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في الـ spinlocks والـ mutexes |
| `CONFIG_LOCKDEP` | يتتبع الـ locking order |
| `CONFIG_KASAN` | يكشف memory corruption في الـ pinctrl structs |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug()` runtime |
| `CONFIG_KALLSYMS` | يظهر أسماء الـ symbols في stack traces |
| `CONFIG_STACKTRACE` | يسمح بـ `dump_stack()` |
| `CONFIG_OF` | لازم لدعم الـ Device Tree |
| `CONFIG_OF_DYNAMIC` | لو بتعمل overlay DT أثناء الـ runtime |

```bash
# تحقق من الـ config الحالية لجهازك
zcat /proc/config.gz | grep -E 'PINCTRL|PINMUX|PINCONF' | sort
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الأداة الرئيسية: `pinctrl` (من حزمة `libgpiod` أو `pinctrl-tools`):**

```bash
# على أنظمة تدعمها (بعض distros)
pinctrl list        # اعرض كل الـ pins
pinctrl get 10      # اعرض config الـ pin رقم 10
pinctrl set 10 op   # اضبط pin 10 كـ output (للاختبار)
```

**باستخدام `gpioinfo` لرؤية الارتباط مع pinctrl:**

```bash
# يُظهر كل GPIO lines وحالتها
gpioinfo

# مثال output:
# gpiochip0 - 32 lines:
#   line  0: "GPIO0" unused input active-high
#   line  10: "UART0_TX" "uart0" output active-high [used]
```

**فحص الـ device link بين pinctrl والـ consumer:**

```bash
# لو link_consumers=true في pinctrl_desc
ls -la /sys/devices/platform/<consumer-device>/supplier\:*/
```

---

#### 7. جدول رسائل الأخطاء الشائعة

| **رسالة الخطأ** | **المعنى** | **الحل** |
|---|---|---|
| `could not get pinctrl` | الـ device مش لاقي الـ controller | تحقق من `pinctrl-0` property في الـ DT |
| `could not find single map entry for device` | مفيش entry في الـ mapping table | أضف `pinctrl-0`, `pinctrl-names` للـ DT node |
| `could not find function for map entry` | اسم الـ function في الـ map مش موجود في الـ driver | تحقق من اسم الـ function في `.pmxops` |
| `could not find group for function` | اسم الـ group في الـ map مش موجود | تحقق من `pinctrl_ops.get_group_name()` |
| `pin X already requested` | pin بيتحاول يتحجز مرتين | لازم `pinctrl_gpio_free()` قبل re-request |
| `could not get state default` | الـ state الافتراضية مش محددة | أضف `pinctrl-names = "default"` في الـ DT |
| `pin controller already registered` | الـ controller اتسجل مرتين | تحقق من الـ probe يتنفذ مرتين |
| `failed to register pinctrl driver` | مشكلة في `pinctrl_register_and_init()` | تحقق من الـ `pinctrl_desc` fields كاملة |
| `could not map group for function` | الـ group والـ function مش متوافقين | راجع `get_group_pins()` implementation |
| `-EPROBE_DEFER` | الـ pinctrl controller لسه ما probed | طبيعي في الـ boot، الـ driver هيحاول تاني |
| `GPIO X is already used` | pin محجوز كـ GPIO وحد تاني بيطلبه | تحقق من الـ multiplexing conflicts |

---

#### 8. نقاط وضع `dump_stack()` و`WARN_ON()`

**أماكن استراتيجية في الـ driver code:**

```c
/* في pinctrl_ops.dt_node_to_map() — لو الـ parsing بيفشل */
static int my_dt_node_to_map(struct pinctrl_dev *pctldev,
                              struct device_node *np_config,
                              struct pinctrl_map **map,
                              unsigned int *num_maps)
{
    /* تحقق من الـ DT node قبل parsing */
    WARN_ON(!np_config);

    ret = parse_pin_config(np_config, ...);
    if (ret < 0) {
        dev_err(pctldev->dev, "failed to parse node %pOF\n", np_config);
        dump_stack(); /* يساعد تتعرف من استدعى الـ function */
        return ret;
    }
}

/* في pinctrl_ops.get_group_pins() — لو selector خارج النطاق */
static int my_get_group_pins(struct pinctrl_dev *pctldev,
                              unsigned int selector,
                              const unsigned int **pins,
                              unsigned int *num_pins)
{
    WARN_ON(selector >= priv->num_groups); /* كشف out-of-bounds */
    ...
}

/* في register callback — تحقق من الـ init */
static int my_pinctrl_probe(struct platform_device *pdev)
{
    ...
    ret = pinctrl_register_and_init(&my_desc, &pdev->dev, priv, &pctldev);
    WARN_ON(ret); /* هيطبع callstack لو فشل */
    ...
}
```

---

### Hardware Level

---

#### 1. التحقق من تطابق حالة الـ Hardware مع الـ Kernel

الهدف هو التأكد إن الـ pin فعلاً اتضبط في الـ hardware زي ما الـ kernel فكر.

```bash
# الخطوة 1: اعرف الـ physical address للـ pinctrl register block من الـ DT
cat /sys/kernel/debug/pinctrl/soc-pinctrl/pins
# أو
cat /proc/device-tree/<pinctrl-node>/reg | xxd

# الخطوة 2: قارن القيمة الـ hardware بالـ software state
# من debugfs (software view):
cat /sys/kernel/debug/pinctrl/soc-pinctrl/pinmux-pins
# ومن الـ register مباشرة (hardware view):
devmem2 <physical_address> w
```

---

#### 2. Register Dump Techniques

```bash
# devmem2 — أسهل أداة
# اقرأ register بـ 32-bit من address 0x12340000
devmem2 0x12340000 w

# اقرأ 4 registers متتالية (يدوياً)
devmem2 0x12340000 w
devmem2 0x12340004 w
devmem2 0x12340008 w
devmem2 0x1234000C w

# /dev/mem + dd (لو devmem2 مش متاح)
dd if=/dev/mem bs=4 count=1 skip=$((0x12340000/4)) 2>/dev/null | xxd

# io utility (من package ioport أو similar)
io -4 -r 0x12340000

# داخل الـ kernel driver — dump تلقائي عند الـ probe
static void my_dump_registers(struct my_pinctrl_priv *priv)
{
    int i;
    for (i = 0; i < priv->num_pins; i++) {
        dev_dbg(priv->dev, "pin%d reg: 0x%08x\n",
                i, readl(priv->base + i * 4));
    }
}
```

**ملاحظة:** `/dev/mem` بيحتاج `CONFIG_DEVMEM=y` و `CONFIG_STRICT_DEVMEM=n` للـ non-whitelisted regions.

---

#### 3. Logic Analyzer / Oscilloscope Tips

| **السيناريو** | **الأداة** | **الإعداد** |
|---|---|---|
| الـ pin مش بيشتغل كـ UART | Logic Analyzer | Decode UART على الـ pin، تحقق من الـ signal |
| مشكلة pull-up/pull-down | Oscilloscope | قيس الـ voltage مع وبدون load، يكون 3.3V أو 1.8V |
| الـ SPI clock مش صح | Logic Analyzer | Decode SPI، تحقق من CPOL/CPHA |
| الـ pin floating | Oscilloscope | تحقق من الـ voltage مش بين 0.8V و2V (undefined zone) |
| الـ drive strength ضعيف | Oscilloscope | الـ rise time بطيء (RC effect) |

**نقاط مهمة:**
- ابدأ دائماً بالتأكد إن الـ pin مش في **tri-state** (floating) قبل أي measurement.
- لو الـ pin بيشتغل كـ GPIO output والـ oscilloscope بيقول 0V ثابت، يمكن الـ pinmux لسه على function تانية.
- استخدم **pull-up resistor خارجي** (10kΩ) كـ sanity check لو مش عارف حالة الـ pin.

---

#### 4. Common Hardware Issues وـ Kernel Log Patterns

| **المشكلة الـ Hardware** | **Pattern في الـ Kernel Log** | **السبب** |
|---|---|---|
| Pin conflict بين وحدتين | `pin X already requested by Y, cannot claim for Z` | اتنين devices بيحاولوا يستخدموا نفس الـ pin |
| Voltage مش صح | Device يشتغل غلط بدون error واضح | الـ IO voltage مش متوافق مع الـ signal level |
| الـ pull resistor مش مضبوط | Random glitches أو stuck states | الـ pinconf مش بيضبط pull بشكل صح |
| الـ drive strength ضعيف | SPI/I2C يفشل عند frequencies عالية | الـ slew rate مش كافي |
| الـ pin مش موصل للـ peripheral | Peripheral driver timeout | Hardware routing issue |
| الـ clock للـ pinctrl controller وقف | `could not get pinctrl` عند boot | Clock gating قبل الـ pinctrl init |

---

#### 5. Device Tree Debugging

**خطوات التحقق من صحة الـ DT:**

```bash
# الخطوة 1: اقرأ الـ DT المحمّل فعلاً في الـ kernel
ls /proc/device-tree/
# أو بشكل أوضح
dtc -I fs /proc/device-tree 2>/dev/null | grep -A 20 'pinctrl'

# الخطوة 2: تحقق من الـ phandles والـ references
cat /proc/device-tree/<device-node>/pinctrl-0 | xxd
# يجب أن يُرجع الـ phandle لـ pinctrl node

# الخطوة 3: تحقق من الـ pinctrl-names
cat /proc/device-tree/<device-node>/pinctrl-names
# يجب أن يكون: default\0 (مع null terminator)

# الخطوة 4: تحقق من الـ pinctrl node نفسه
ls /proc/device-tree/soc/pinctrl@12340000/
cat /proc/device-tree/soc/pinctrl@12340000/compatible

# الخطوة 5: تحقق من الـ reg property
cat /proc/device-tree/soc/pinctrl@12340000/reg | xxd
# يجب أن يُطابق الـ datasheet base address
```

**مقارنة الـ DT source بالـ loaded DT:**

```bash
# احفظ الـ DT المحمّل في ملف واقارنه بالـ source
dtc -I fs -O dts /proc/device-tree > /tmp/loaded.dts 2>/dev/null

# ابحث عن الـ pinctrl configuration node
grep -A 30 'pinctrl' /tmp/loaded.dts | head -60
```

**الأخطاء الشائعة في الـ DT:**

```dts
/* غلط — بينسى pinctrl-names */
uart0 {
    pinctrl-0 = <&uart0_pins>;
    /* لازم يضيف: pinctrl-names = "default"; */
};

/* غلط — phandle بيشير لـ node غلط */
uart0 {
    pinctrl-0 = <&wrong_node>;  /* لازم يكون &uart0_pins */
    pinctrl-names = "default";
};

/* صح */
uart0 {
    pinctrl-0 = <&uart0_pins>;
    pinctrl-names = "default";
};

&pinctrl {
    uart0_pins: uart0-pins {
        pins = "UART0_TX", "UART0_RX";
        function = "uart0";
    };
};
```

---

### Practical Commands

---

#### 1. أوامر جاهزة لكل تقنية

**فحص شامل سريع للـ pinctrl:**

```bash
#!/bin/sh
# pinctrl-debug-dump.sh — script للـ dump الشامل

DEBUGFS=/sys/kernel/debug/pinctrl
CTRL=$(ls $DEBUGFS | grep -v 'pinctrl-' | head -1)

echo "=== Registered Controllers ==="
cat $DEBUGFS/pinctrl-devices

echo ""
echo "=== Active Handles (consumers + states) ==="
cat $DEBUGFS/pinctrl-handles

echo ""
echo "=== Mapping Table ==="
cat $DEBUGFS/pinctrl-maps

echo ""
echo "=== Pin Groups for: $CTRL ==="
cat $DEBUGFS/$CTRL/pingroups

echo ""
echo "=== Mux Functions ==="
cat $DEBUGFS/$CTRL/pinmux-functions

echo ""
echo "=== Current Pin Assignments ==="
cat $DEBUGFS/$CTRL/pinmux-pins

echo ""
echo "=== Pin Configurations ==="
cat $DEBUGFS/$CTRL/pinconf-pins
```

```bash
# تشغيل الـ script
chmod +x pinctrl-debug-dump.sh
./pinctrl-debug-dump.sh 2>&1 | tee /tmp/pinctrl-dump.txt
```

**تفعيل dynamic debug وجمع الـ logs:**

```bash
# فعّل debug logging للـ pinctrl core كاملاً
for f in core.c pinmux.c pinconf.c devicetree.c; do
    echo "file drivers/pinctrl/$f +pflmt" > /sys/kernel/debug/dynamic_debug/control
done

# ارفع الـ dmesg level
dmesg -n 8

# احفظ الـ kernel messages الحالية
dmesg --clear

# أعد تشغيل الـ device أو trigger الـ probe
echo "your-driver" > /sys/bus/platform/drivers/your-driver/unbind
echo "your-driver" > /sys/bus/platform/drivers/your-driver/bind

# اجمع الـ output
dmesg | grep -E 'pinctrl|pinmux|pinconf' > /tmp/pinctrl-logs.txt
cat /tmp/pinctrl-logs.txt
```

**تفعيل ftrace:**

```bash
# تفعيل tracing للـ pinctrl state selection
mount -t tracefs nodev /sys/kernel/tracing 2>/dev/null || true

echo nop > /sys/kernel/tracing/current_tracer
echo > /sys/kernel/tracing/trace
echo 1 > /sys/kernel/tracing/events/pinctrl/enable
echo 1 > /sys/kernel/tracing/tracing_on

# trigger the operation you want to trace...
# مثلاً: echo driver > /sys/bus/.../unbind && echo driver > .../bind

echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace
```

**فحص GPIO range integration:**

```bash
# اعرف الـ base pin لكل GPIO chip وارتباطه بالـ pinctrl
for chip in /sys/class/gpio/gpiochip*; do
    echo "=== $(cat $chip/label) ==="
    echo "  base: $(cat $chip/base)"
    echo "  ngpio: $(cat $chip/ngpio)"
done

# فحص conflict معين — هل pin 10 محجوز؟
cat /sys/kernel/debug/pinctrl/soc-pinctrl/pinmux-pins | grep "pin 10"
```

**قراءة الـ register مباشرة:**

```bash
# بعد ما تعرف الـ base address من DT أو datasheet
# مثلاً base = 0x12340000, الـ pin MUX register على offset 0x28
PINMUX_REG=0x12340028
devmem2 $PINMUX_REG w

# output مثلاً:
# /dev/mem opened.
# Memory mapped at address 0xb6f6e000.
# Value at address 0x12340028 (0xb6f6e028): 0x00000002
# القيمة 0x2 تعني الـ function رقم 2 مضبوطة على هذا الـ pin
```

---

#### 2. تفسير الـ Output

**مثال على `pinmux-pins` وتفسيره:**

```
pin 10 (UART0_TX): uart@12340000 (GPIO UNCLAIMED)
│      │            │               │
│      │            │               └─ مفيش GPIO consumer حاجز الـ pin
│      │            └─── الـ device الـ consumer الحالي
│      └──────────────── اسم الـ pin
└─────────────────────── رقم الـ pin
```

**مثال على `pingroups` وتفسيره:**

```
registered pin groups:
group: uart0_grp
pin 10 (UART0_TX)
pin 11 (UART0_RX)
group: spi0_grp
pin 20 (SPI0_CLK)
pin 21 (SPI0_MOSI)
pin 22 (SPI0_MISO)
pin 23 (SPI0_CS0)
```

**مثال على error في الـ dmesg وتفسيره:**

```
[    2.345678] pinctrl-foo 12340000.pinctrl: pin 10 already requested by spi@56780000; cannot claim for uart@12340000
                │                             │                  │                         │
                │                             │                  │                         └─ الـ device اللي بيطلبه
                │                             │                  └─── اللي حاجزه من قبل
                │                             └─── الـ pin المتعارض
                └─────────────────────────────── الـ pin controller
```

**الحل:** راجع الـ DT أو الـ mapping table، وتأكد إن كل pin مش مستخدم في أكتر من function واحدة في نفس الوقت.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: RK3562 — Industrial Gateway — UART3 مش شغال بعد bring-up

#### العنوان
**الـ UART3 على RK3562 بيرفض يشتغل رغم إن الـ device tree صح**

#### السياق
شركة بتعمل industrial gateway بالـ RK3562. الـ gateway بيتكلم مع PLC عبر RS-485 على UART3. البورد اتبنت، kernel اتبوت، بس `/dev/ttyS3` مش ظاهر خالص.

#### المشكلة
الـ engineer فتح `dmesg` لقى:
```
pinctrl-rockchip: pin 77 already requested by gpio-keys; cannot claim for uart3
```
الـ UART3 مش قادر يأخذ الـ pins لأن GPIO driver سبقه.

#### التحليل
الـ pinctrl subsystem بيشتغل كالآتي: لما UART3 driver بيعمل probe، بيطلب الـ pinctrl state بتاعته. الـ core بيمشي على الـ `pinctrl_desc` اللي سجّله الـ RK3562 driver، وبيدور على الـ `pingroup` المرتبط بـ "uart3-xfer":

```c
/* في rockchip pinctrl driver */
static const struct pinctrl_ops rk3562_pctrl_ops = {
    .get_groups_count = rockchip_get_groups_count,
    .get_group_name   = rockchip_get_group_name,
    .get_group_pins   = rockchip_get_group_pins,  /* بيرجع pins[] للـ group */
    .dt_node_to_map   = rockchip_dt_node_to_map,
    .dt_free_map      = rockchip_dt_free_map,
};
```

الـ `get_group_pins` بيرجع array فيها `pins[0] = 77`. الـ core بعدين بيحاول يعمل `request` على pin 77 عبر `pinmux_ops->request`، بس الـ pin مسجّل قبل كده عند gpio-keys، فبيرجع `-EBUSY` وبيطبع الـ error.

المشكلة الأصل: في الـ device tree، `gpio-keys` node بياخد `rockchip,pins = <2 RK_PA5 ...>` وده نفس pin 77.

#### الحل

**خطوة 1:** تأكيد من debugfs:
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pins | grep "pin 77"
# Output: pin 77 (GPIO2_A5) owned by gpio-keys
```

**خطوة 2:** تصليح الـ Device Tree — إزالة التعارض:
```dts
/* قبل التصليح — خطأ */
gpio_keys: gpio-keys {
    compatible = "gpio-keys";
    button-0 {
        gpios = <&gpio2 RK_PA5 GPIO_ACTIVE_LOW>;  /* pin 77 — conflict! */
    };
};

uart3: serial@fe690000 {
    pinctrl-0 = <&uart3m0_xfer>;  /* كمان بيستخدم pin 77 */
    status = "okay";
};

/* بعد التصليح */
gpio_keys: gpio-keys {
    compatible = "gpio-keys";
    button-0 {
        gpios = <&gpio2 RK_PA4 GPIO_ACTIVE_LOW>;  /* غيّر لـ pin 76 */
    };
};
```

**خطوة 3:** verify:
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinmux-pins | grep uart3
# pin 77 (GPIO2_A5): uart3 GPIO UNCLAIMED
```

#### الدرس المستفاد
الـ `pinmux_ops->request` callback هو الـ gatekeeper — بيمنع أي تعارض على مستوى الـ pin. لازم تراجع `/sys/kernel/debug/pinctrl/` قبل أي bring-up جديد عشان تعرف مين ساكن على أنهي pin.

---

### السيناريو الثاني: STM32MP1 — IoT Sensor Board — SPI2 بيتعطل بعد suspend/resume

#### العنوان
**الـ SPI2 بيوقف عن الشغل بعد أول دورة suspend على STM32MP1**

#### السياق
IoT sensor بيشتغل على STM32MP157. فيه pressure sensor بيتكلم عبر SPI2. في الـ production، البورد بتعمل suspend بعد فترة idle. بعد أول resume، الـ SPI transactions بتفشل بـ `-ETIMEDOUT`.

#### المشكلة
بعد resume، الـ SPI pins بترجع لـ default state (GPIO input) بدل ما ترجع لـ SPI function. الـ DMA transfers بتفشل لأن الـ MOSI/MISO مش متوصلين للـ SPI controller.

#### التحليل
المشكلة في الـ `link_consumers` field في `struct pinctrl_desc`:

```c
struct pinctrl_desc {
    const char *name;
    const struct pinctrl_pin_desc *pins;
    unsigned int npins;
    const struct pinctrl_ops *pctlops;
    const struct pinmux_ops *pmxops;
    const struct pinconf_ops *confops;
    struct module *owner;
    bool link_consumers;  /* <-- ده اللي المشكلة فيه */
};
```

لما `link_consumers = false` (الـ default)، مفيش **device link** بين الـ pinctrl device والـ SPI consumer. يعني الـ PM core ممكن يعمل suspend للـ pinctrl controller قبل ما SPI driver يخلص الـ resume sequence بتاعه. النتيجة: الـ pinctrl بيـ restore الـ default state بعدين SPI driver بيتأخر يطلب الـ "default" pinctrl state تاني.

الـ flow الغلط:
```
Resume event
    └─> STM32 pinctrl resumes first  ← correct
    └─> SPI driver resumes           ← but pinctrl already reset to "sleep" state
        └─> pinctrl_select_state("default") ← too late, race condition
```

#### الحل

**خطوة 1:** تفعيل `link_consumers` في الـ STM32 pinctrl driver:

```c
/* drivers/pinctrl/stm32/pinctrl-stm32.c */
static int stm32_pctrl_probe(struct platform_device *pdev)
{
    struct stm32_pinctrl *pctl;
    /* ... */

    pctl->pctl_desc.name          = dev_name(&pdev->dev);
    pctl->pctl_desc.pins          = pctl->pins;
    pctl->pctl_desc.npins         = pctl->npins;
    pctl->pctl_desc.pctlops       = &stm32_pctrl_ops;
    pctl->pctl_desc.pmxops        = &stm32_pmx_ops;
    pctl->pctl_desc.confops       = &stm32_pconf_ops;
    pctl->pctl_desc.owner         = THIS_MODULE;
    pctl->pctl_desc.link_consumers = true;  /* أضف السطر ده */

    return devm_pinctrl_register_and_init(&pdev->dev,
                                          &pctl->pctl_desc,
                                          pctl,
                                          &pctl->pctl_dev);
}
```

**خطوة 2:** verify الـ device link:
```bash
ls /sys/bus/platform/devices/spi@4000b000/consumer:*/
# يظهر link للـ pinctrl device
```

**خطوة 3:** test suspend/resume cycle:
```bash
echo mem > /sys/power/state
# بعد wake-up:
spidev_test -D /dev/spidev1.0 -p "hello" -v
# يجي OK بدون timeout
```

#### الدرس المستفاد
الـ `link_consumers` في `struct pinctrl_desc` مش مجرد field زيادة — هو بيحكم ترتيب الـ PM. في أي منتج فيه suspend/resume، خليه `true` عشان تضمن إن الـ pinctrl يصحى قبل الـ consumers بتاعته.

---

### السيناريو الثالث: i.MX8MP — Android TV Box — HDMI audio مش بيشتغل

#### العنوان
**الـ HDMI audio silent على i.MX8MP TV Box رغم إن video شغال**

#### السياق
Android TV box بيشغل Android 13 على i.MX8MP. الـ HDMI video شغال تمام، بس الـ audio على HDMI صامت. الـ customer بيشتكي إن الـ TV مش بيعزف صوت.

#### المشكلة
الـ I2S pins المربوطة بالـ HDMI audio bridge مش مـ configured صح. الـ `pinctrl-names` في الـ device tree عندها `"default"` و`"sleep"` بس مفيش `"init"` state للـ I2S clock pins.

#### التحليل
الـ i.MX8MP pinctrl driver بيسجّل `struct pinctrl_desc` بـ `dt_node_to_map`:

```c
static const struct pinctrl_ops imx_pctrl_ops = {
    .get_groups_count = imx_get_groups_count,
    .get_group_name   = imx_get_group_name,
    .get_group_pins   = imx_get_group_pins,
    .pin_dbg_show     = imx_pin_dbg_show,
    .dt_node_to_map   = imx_dt_node_to_map,   /* بيحوّل DT nodes لـ map entries */
    .dt_free_map      = imx_dt_free_map,
};
```

لما الـ `dt_node_to_map` بتشتغل على الـ I2S audio node، بتعمل `pinctrl_map` entries للـ "default" state. بس الـ I2S3_TXD pin (اللي بيبعت audio data للـ bridge) اتعمله `IOMUXC_SAI3_TXD__SAI3_TX_DATA0` مع **drive strength 0** بدل ما يكون X4 أو X6.

نتيجة: الـ signal level بيوصل للـ HDMI bridge ضعيف جداً فالـ bridge بيـ ignore الـ I2S data.

التحقق:
```bash
# على Android shell
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pins | grep SAI3_TXD
# pin 88 (SAI3_TXD): drive-strength=0 (should be 4 or 6)
```

#### الحل

**خطوة 1:** تصليح الـ DT للـ I2S pins:

```dts
/* في imx8mp-tv-box.dts */
&iomuxc {
    pinctrl_sai3: sai3grp {
        fsl,pins = <
            /* قبل: drive strength R0/6 = ~43 ohm — ضعيف */
            MX8MP_IOMUXC_SAI3_TXD__AUDIOMIX_SAI3_TX_DATA00  0x94

            /* بعد: drive strength R0/2 = ~130 ohm مع slew rate fast */
            MX8MP_IOMUXC_SAI3_TXD__AUDIOMIX_SAI3_TX_DATA00  0xd6
        >;
    };
};
```

**خطوة 2:** تأكيد من الـ debugfs إن الـ `pingroup` بياخد الـ pins الصح:

```bash
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pingroups | grep -A5 sai3grp
# group: sai3grp
#   pin 85 (SAI3_RXD)
#   pin 86 (SAI3_RXC)
#   pin 87 (SAI3_RXFS)
#   pin 88 (SAI3_TXD)    <-- الـ pin المشكلة
```

**خطوة 3:** verify الـ `get_group_pins` بيرجع الصح عبر:
```bash
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pinmux-functions | grep sai3
```

#### الدرس المستفاد
الـ `pingroup` struct والـ `get_group_pins` callback مش بس بيحددوا أنهي pins — الـ config value المرفق معاهم (drive strength, slew rate) بيأثر على الـ signal integrity. Audio signals حساسة جداً للـ drive strength؛ دايماً اتحقق من الـ electrical specs في datasheet.

---

### السيناريو الرابع: AM62x — Automotive ECU — I2C بيـ hang بعد GPIO export

#### العنوان
**الـ I2C3 على AM62x ECU بيـ lock-up بعد ما userspace يعمل GPIO export للـ SCL pin**

#### السياق
Automotive ECU بيشتغل على TI AM62x. الـ I2C3 بيتحكم في ADAS sensor. فريق الـ BSP عمل script بيعمل export لـ GPIO لـ testing، غلطانين عملوا export لـ GPIO اللي هو نفسه SCL على I2C3.

#### المشكلة
بعد `echo 74 > /sys/class/gpio/export`، الـ I2C3 controller اتـ freeze. الـ kernel log بيقول:
```
i2c_designware 20520000.i2c: controller timed out
```

#### التحليل
الـ `pinmux_ops->strict` flag في AM62x pinctrl driver مش مفعّل:

```c
static const struct pinmux_ops am62x_pmx_ops = {
    .request             = am62x_pmx_request,
    .free                = am62x_pmx_free,
    .get_functions_count = am62x_pmx_get_fns_count,
    .get_function_name   = am62x_pmx_get_fn_name,
    .get_function_groups = am62x_pmx_get_fn_groups,
    .set_mux             = am62x_pmx_set_mux,
    .gpio_request_enable = am62x_pmx_gpio_request_enable,
    .gpio_disable_free   = am62x_pmx_gpio_disable_free,
    .gpio_set_direction  = am62x_pmx_gpio_set_direction,
    .strict              = false,  /* <-- المشكلة هنا */
};
```

لما `strict = false`، الـ subsystem بيسمح بـ concurrent ownership — الـ GPIO subsystem بياخد الـ pin للـ GPIO function، والـ I2C driver عنده الـ pin كـ I2C function في نفس الوقت. النتيجة: الـ IOMUX register اتغيّر لـ GPIO input، فالـ SCL line بقت floating وسط transaction، والـ I2C controller hang في wait state.

الـ `pinctrl_gpio_range` المرتبطة بالـ GPIO controller عندها `base` و`pin_base` بيوصل للـ pin 74. الـ `pinctrl_add_gpio_range` اتستدعت بدون ما تتحقق إن الـ pin مشغول.

#### الحل

**خطوة 1:** تفعيل `strict` في الـ AM62x driver:

```c
static const struct pinmux_ops am62x_pmx_ops = {
    /* ... باقي الـ callbacks ... */
    .strict = true,  /* يمنع GPIO export لأي pin مشغول بـ function */
};
```

**خطوة 2:** إضافة protection في الـ production script:

```bash
#!/bin/bash
# قبل أي GPIO export، تحقق إن الـ pin مش مـ muxed لـ function
PIN=74
OWNER=$(cat /sys/kernel/debug/pinctrl/pinctrl-single/pinmux-pins 2>/dev/null | \
        grep "pin ${PIN}" | awk '{print $NF}')

if [ "$OWNER" != "UNCLAIMED" ] && [ -n "$OWNER" ]; then
    echo "ERROR: pin ${PIN} owned by ${OWNER}, cannot export as GPIO"
    exit 1
fi
echo $PIN > /sys/class/gpio/export
```

**خطوة 3:** لو محتاج I2C و GPIO يتشاركوا نفس الـ physical pin (مش recommended)، استخدم `pinctrl_find_gpio_range_from_pin` لتتحقق:

```c
struct pinctrl_gpio_range *range;
range = pinctrl_find_gpio_range_from_pin(pctldev, 74);
if (range)
    dev_warn(dev, "pin 74 already in GPIO range %s\n", range->name);
```

#### الدرس المستفاد
الـ `strict` flag في `pinmux_ops` هو صمام أمان بين الـ GPIO subsystem والـ peripheral functions. في الـ automotive وأي safety-critical system، لازم يكون `true` دايماً. الـ `pinctrl_find_gpio_range_from_pin` أداة قوية للـ runtime validation.

---

### السيناريو الخامس: Allwinner H616 — Custom Board Bring-Up — pinctrl driver مش بيتسجل

#### العنوان
**الـ custom pinctrl driver للـ H616 بيـ crash في boot بسبب `devm_pinctrl_register_and_init` sequence غلط**

#### السياق
فريق بيعمل bring-up لبورد custom على Allwinner H616 (بيستخدم في Android TV boxes). الـ engineer كتب pinctrl driver جديد بيـ extend الـ sunxi driver. البورد بتـ crash بـ kernel panic في أول boot.

#### المشكلة
الـ kernel panic:
```
BUG: kernel NULL pointer dereference at 0000000000000008
PC is at pinctrl_enable+0x14/0x60
```

#### التحليل
الـ engineer استخدم الـ API القديم `pinctrl_register` بدل الـ modern two-step API:

```c
/* كود الـ engineer — غلط */
static int h616_pinctrl_probe(struct platform_device *pdev)
{
    struct h616_pinctrl *pctl;
    struct pinctrl_dev *pctldev;

    /* ... init pctl ... */

    /* استخدم الـ deprecated API */
    pctldev = pinctrl_register(&pctl->desc, &pdev->dev, pctl);
    if (IS_ERR(pctldev))
        return PTR_ERR(pctldev);

    /* حاول يعمل pinctrl_enable بعدين — بس الـ pctldev مش initialized كامل */
    return pinctrl_enable(pctldev);  /* <-- CRASH هنا */
}
```

المشكلة: `pinctrl_register` (الـ deprecated) بيعمل register وبيعمل enable في نفس الوقت داخلياً. لو الـ engineer بعدين استدعى `pinctrl_enable` تاني، الـ internal state بيتـ corrupt.

الصح: استخدام الـ two-step API اللي موجود في الـ header:

```c
/* من pinctrl.h — الـ API الصح */
extern int pinctrl_register_and_init(const struct pinctrl_desc *pctldesc,
                                     struct device *dev,
                                     void *driver_data,
                                     struct pinctrl_dev **pctldev);
extern int pinctrl_enable(struct pinctrl_dev *pctldev);
```

الـ two-step design موجود عشان بعض الـ drivers محتاجين يعملوا additional setup (زي تسجيل GPIO ranges عبر `pinctrl_add_gpio_range`) بين الـ register والـ enable.

#### الحل

**الكود الصح:**

```c
static int h616_pinctrl_probe(struct platform_device *pdev)
{
    struct h616_pinctrl *pctl;
    struct pinctrl_dev *pctldev;
    int ret;

    pctl = devm_kzalloc(&pdev->dev, sizeof(*pctl), GFP_KERNEL);
    if (!pctl)
        return -ENOMEM;

    /* Step 1: register only — no enable yet */
    ret = devm_pinctrl_register_and_init(&pdev->dev,
                                         &pctl->desc,
                                         pctl,
                                         &pctldev);
    if (ret)
        return ret;

    /* Step 2: add GPIO ranges BEFORE enable */
    pinctrl_add_gpio_ranges(pctldev,
                            h616_gpio_ranges,
                            ARRAY_SIZE(h616_gpio_ranges));

    /* Step 3: now enable — subsystem is fully ready */
    return pinctrl_enable(pctldev);
}
```

**التحقق من الـ registration:**

```bash
# بعد boot ناجح
ls /sys/bus/platform/drivers/h616-pinctrl/
cat /sys/kernel/debug/pinctrl/pinctrl-h616/pins | head -20

# تأكيد إن الـ GPIO ranges اتضافت
cat /sys/kernel/debug/pinctrl/pinctrl-h616/gpio-ranges
# Output:
# GPIO range 0: name "h616-pio" base 0 npins 288 pin base 0
```

**استخدام `of_pinctrl_get` للـ debug:**

```c
/* في debug code أو في consumer driver للتحقق */
#if IS_ENABLED(CONFIG_OF) && IS_ENABLED(CONFIG_PINCTRL)
struct pinctrl_dev *pdev_check;
pdev_check = of_pinctrl_get(dev->of_node);
if (!pdev_check)
    dev_err(dev, "pinctrl device not found for this node!\n");
#endif
```

#### الدرس المستفاد
الـ header `pinctrl.h` نفسه بيقول صراحة: `"Please use pinctrl_register_and_init() and pinctrl_enable() instead"`. الـ two-step API مش مجرد style — هو بيضمن إن الـ GPIO ranges والـ mappings كلها تتسجل قبل ما الـ subsystem يبدأ يـ service الـ consumer requests. أي new driver لازم يستخدم `devm_pinctrl_register_and_init` + `pinctrl_enable` وميقربش من الـ deprecated `pinctrl_register`.
## Phase 7: مصادر ومراجع

### توثيق Kernel الرسمي

| المصدر | الرابط |
|--------|--------|
| **PINCTRL subsystem** — التوثيق الرسمي الكامل | [docs.kernel.org/driver-api/pin-control.html](https://docs.kernel.org/driver-api/pin-control.html) |
| **Documentation/pinctrl.txt** — النسخة القديمة من التوثيق | [kernel.org/doc/Documentation/pinctrl.txt](https://www.kernel.org/doc/Documentation/pinctrl.txt) |
| **pinctrl.h** — الـ header الرئيسي للـ subsystem | `include/linux/pinctrl/pinctrl.h` |
| **pinmux.h** — واجهة الـ pin multiplexing | `include/linux/pinctrl/pinmux.h` |
| **pinconf.h** — واجهة الـ pin configuration | `include/linux/pinctrl/pinconf.h` |
| **consumer.h** — واجهة المستهلك (الـ drivers) | `include/linux/pinctrl/consumer.h` |
| **machine.h** — الـ mapping tables | `include/linux/pinctrl/machine.h` |
| **kernel source tree** — الـ pinctrl subsystem | `drivers/pinctrl/` |

---

### مقالات LWN.net

دي أهم المقالات اللي غطّت نشأة وتطور الـ pinctrl subsystem:

#### المقالات الأساسية

**1. The pin control subsystem**
الـ overview الرئيسي للـ subsystem — شرح المكونات الثلاثة (pinctrl / pinmux / pinconf) وليه اتعمل أصلاً.
[https://lwn.net/Articles/468759/](https://lwn.net/Articles/468759/)

**2. Documentation/pinctrl.txt**
أول ما اتنشر الـ documentation الرسمي على LWN — بيشرح الـ API والـ concepts الأساسية.
[https://lwn.net/Articles/465077/](https://lwn.net/Articles/465077/)

**3. drivers: create a pin control subsystem**
الـ patch series الأولانية اللي أنشأت الـ subsystem من الأساس.
[https://lwn.net/Articles/463335/](https://lwn.net/Articles/463335/)

**4. pin controller subsystem v7**
الـ revision السابعة من الـ patchset — قبل ما يتقبل في الـ mainline.
[https://lwn.net/Articles/459190/](https://lwn.net/Articles/459190/)

**5. pinctrl: add a generic pin config interface**
إضافة الـ generic configuration interface — بدل ما كل driver يعمل الـ config بطريقته.
[https://lwn.net/Articles/468770/](https://lwn.net/Articles/468770/)

**6. pinctrl: add a pin config interface**
الـ patch الأولاني لإضافة الـ pin config API.
[https://lwn.net/Articles/471826/](https://lwn.net/Articles/471826/)

**7. drivers: create a pinmux subsystem v3**
الـ patch series اللي أضافت الـ pinmux كـ component منفصل.
[https://lwn.net/Articles/447394/](https://lwn.net/Articles/447394/)

#### مقالات التطوير اللاحق

**8. drivers: pinctrl sleep and idle states in the core**
إضافة الـ sleep و idle states لدعم الـ power management.
[https://lwn.net/Articles/552972/](https://lwn.net/Articles/552972/)

**9. drivers/pinctrl: Add the concept of an "init" state**
إضافة الـ `init` state — يتطبق قبل `probe` ثم ينتقل للـ `default`.
[https://lwn.net/Articles/615322/](https://lwn.net/Articles/615322/)

**10. pinctrl: introduce the concept of a GPIO pin function category**
إضافة الـ `PINFUNCTION_FLAG_GPIO` و `struct pinfunction` — موجودين في الـ header الحالي.
[https://lwn.net/Articles/1031226/](https://lwn.net/Articles/1031226/)

**11. gpio: add pinctrl based generic gpio driver**
دمج الـ GPIO والـ pinctrl في driver واحد generic.
[https://lwn.net/Articles/946636/](https://lwn.net/Articles/946636/)

---

### Kernel Git Tree للـ pinctrl

الـ subsystem بيتطور في tree منفصلة عند Linus Walleij:

```
git://git.kernel.org/pub/scm/linux/kernel/git/linusw/linux-pinctrl.git
```

Mirror على Google:
[https://kernel.googlesource.com/pub/scm/linux/kernel/git/linusw/linux-pinctrl/](https://kernel.googlesource.com/pub/scm/linux/kernel/git/linusw/linux-pinctrl/)

#### Commits مهمة في تاريخ الـ subsystem

| التغيير | البحث في git log |
|---------|-----------------|
| الـ commit الأول للـ subsystem | `git log --all --oneline -- drivers/pinctrl/core.c` |
| إضافة `pinctrl_register_and_init()` | `git log --all --grep="pinctrl_register_and_init"` |
| إضافة `link_consumers` لـ `pinctrl_desc` | `git log --all --grep="link_consumers"` |
| إضافة `struct pinfunction` | `git log --all --grep="pinfunction"` |

---

### Mailing List

النقاشات الرئيسية للـ subsystem على:

- **lore.kernel.org** (الأرشيف الرسمي):
  [https://lore.kernel.org/linux-gpio/](https://lore.kernel.org/linux-gpio/)

- نقاش إضافة الـ GPIO pin function category:
  [https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10587.html](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10587.html)

- نقاشات Linus Walleij على lkml.kernel.org:
  [https://lkml.kernel.org/lkml/CACRpkdYqoU0foW1izcZapbgUm+NO4quz9AZZBmts4841BRcigA@mail.gmail.com/](https://lkml.kernel.org/lkml/CACRpkdYqoU0foW1izcZapbgUm+NO4quz9AZZBmts4841BRcigA@mail.gmail.com/)

---

### kernelnewbies.org

الـ kernel releases اللي فيها تغييرات مهمة في الـ pinctrl:

| الـ Release | التغييرات |
|------------|-----------|
| **Linux 6.8** | إضافة drivers للـ Qualcomm X1E80100 وSM8650 وSM4450 | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) |
| **Linux 6.11** | تغييرات في الـ pinctrl core | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| **Linux 6.13** | دعم Versal وKendryte K230 وExynos990 | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) |
| **Linux 6.15** | دعم SG2042 وAmlogic وAllwinner A523 | [kernelnewbies.org/Linux_6.15](https://kernelnewbies.org/Linux_6.15) |
| **كل التغييرات** | صفحة LinuxChanges الشاملة | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) |

---

### elinux.org

- **Pin Control & GPIO Update** — عرض تقديمي من ELC بيشرح العلاقة بين pinctrl و GPIO:
  [https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf)

- **EBC Device Trees** — أمثلة عملية على استخدام `pinctrl-single` في Device Tree للـ BeagleBone:
  [https://elinux.org/EBC_Device_Trees](https://elinux.org/EBC_Device_Trees)

- **Tests: i2c-demux-pinctrl** — اختبار الـ `i2c-demux-pinctrl` driver:
  [https://www.elinux.org/Tests:i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl)

---

### مصادر خارجية إضافية

- **PINCTRL subsystem — kernel.org v4.14**:
  [https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html)

- **Linux device driver development: The pin control subsystem** (embedded.com):
  [https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/)

- **Pinctrl overview — STM32MPU Wiki** (مثال تطبيقي على STM32):
  [https://wiki.st.com/stm32mpu/wiki/Pinctrl_overview](https://wiki.st.com/stm32mpu/wiki/Pinctrl_overview)

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)

الكتاب مش بيغطي الـ pinctrl subsystem مباشرة (لأنه اتكتب قبل ما يتضاف للـ kernel)، بس الفصول دي مهمة كـ background:

- **Chapter 1** — Introduction to Device Drivers
- **Chapter 14** — The Linux Device Model (`struct device`، الـ `kobject`، الـ `sysfs`)
- **Chapter 9** — Communicating with Hardware (الـ I/O registers والـ memory-mapped I/O)

#### Linux Kernel Development — Robert Love (3rd Edition)

- **Chapter 17** — Devices and Modules (الـ device model، الـ bus/driver/device)
- **Chapter 12** — Memory Management (لفهم `devm_*` allocation)
- **Chapter 5** — System Calls (لفهم kernel/userspace boundary)

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- **Chapter 16** — Kernel Initialization (بيشرح إزاي الـ pinctrl بيتهيأ وقت الـ boot)
- **Chapter 15** — Bootloaders (العلاقة بين الـ bootloader والـ pin setup)
- **Chapter 14** — Board Support Packages (الـ BSP والـ pin multiplexing الخاص بكل SoC)

---

### Search Terms للبحث عن معلومات أكتر

```
linux kernel pinctrl subsystem
pinctrl_register_and_init driver example
pinctrl_ops dt_node_to_map device tree
linux pin multiplexing embedded
pinctrl_gpio_range gpio chip integration
pinctrl_desc pinmux_ops pinconf_ops
devm_pinctrl_register_and_init
linux pinctrl consumer API
pinctrl states default sleep idle
linux pinctrl debugfs
```

---

### ملخص سريع للمراجع الأهم

```
أهم 5 مراجع تبدأ بيهم:

1. docs.kernel.org/driver-api/pin-control.html   ← التوثيق الرسمي الكامل
2. lwn.net/Articles/468759/                       ← أحسن overview للـ subsystem
3. lwn.net/Articles/465077/                       ← الـ documentation الأصلي
4. drivers/pinctrl/core.c                         ← الـ implementation الفعلي
5. lore.kernel.org/linux-gpio/                    ← مناقشات الـ maintainers
```
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على الـ `pinctrl_register_and_init()` — دي الـ function الأساسية اللي بيتسجل فيها أي pin controller جديد في الـ kernel. كل مرة أي driver بيسجل نفسه كـ pin controller، الـ probe بتتفعل وبتطبع اسم الـ controller وعدد الـ pins بتاعته.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * pinctrl_register_and_init kprobe monitor
 *
 * Hooks into pinctrl_register_and_init() and logs every pin controller
 * registration: controller name + pin count.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/pinctrl/pinctrl.h> /* struct pinctrl_desc */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on pinctrl_register_and_init — log every pin controller registration");

/* -----------------------------------------------------------------------
 * pre_handler: called just before pinctrl_register_and_init() executes.
 *
 * Signature of the hooked function:
 *   int pinctrl_register_and_init(const struct pinctrl_desc *pctldesc,
 *                                  struct device *dev,
 *                                  void *driver_data,
 *                                  struct pinctrl_dev **pctldev);
 *
 * On x86-64:
 *   regs->di = arg0 → pointer to struct pinctrl_desc
 *   regs->si = arg1 → pointer to struct device  (unused here)
 * ----------------------------------------------------------------------- */
static int pinctrl_reg_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Cast the first argument register (RDI on x86-64) to the descriptor
     * pointer. We read it before the function body runs, so the descriptor
     * is always valid at this point.
     */
    const struct pinctrl_desc *desc =
            (const struct pinctrl_desc *)regs->di;

    if (!desc)          /* safety: NULL guard */
        return 0;

    pr_info("pinctrl_probe: registering controller \"%s\", pins=%u\n",
            desc->name  ? desc->name  : "(unnamed)",
            desc->npins);

    return 0; /* 0 = let the original function continue normally */
}

/* -----------------------------------------------------------------------
 * kprobe descriptor
 * ----------------------------------------------------------------------- */
static struct kprobe kp = {
    .symbol_name = "pinctrl_register_and_init", /* kernel symbol to hook   */
    .pre_handler = pinctrl_reg_pre,             /* called before the func  */
};

/* -----------------------------------------------------------------------
 * module_init
 * ----------------------------------------------------------------------- */
static int __init pinctrl_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("pinctrl_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("pinctrl_probe: kprobe planted on %s @ %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* -----------------------------------------------------------------------
 * module_exit
 * ----------------------------------------------------------------------- */
static void __exit pinctrl_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("pinctrl_probe: kprobe removed\n");
}

module_init(pinctrl_probe_init);
module_exit(pinctrl_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | **MODULE_LICENSE** والـ `module_init`/`module_exit` macros |
| `linux/kernel.h` | الـ `pr_info` / `pr_err` logging |
| `linux/kprobes.h` | الـ `struct kprobe` وكل الـ API بتاعتها |
| `linux/pinctrl/pinctrl.h` | الـ `struct pinctrl_desc` عشان نقرأ منها اسم الـ controller وعدد الـ pins |

الـ `linux/pinctrl/pinctrl.h` ضروري هنا لأنه بيعرّف الـ `struct pinctrl_desc` اللي بنعمل cast ليها من الـ register argument.

---

#### الـ `pre_handler` — `pinctrl_reg_pre`

```c
static int pinctrl_reg_pre(struct kprobe *p, struct pt_regs *regs)
```

الـ kprobe بتستدعي الـ handler ده قبل ما تنفذ أول instruction في الـ function المسجلة. الـ `struct pt_regs` بيحتوي على قيم الـ CPU registers في لحظة الاستدعاء، فعلى x86-64 بنلاقي الـ argument الأول في `regs->di` (اللي هو RDI register وفق الـ System V ABI). بنعمل cast لـ `const struct pinctrl_desc *` ونطبع الاسم وعدد الـ pins.

الـ `return 0` مهم — أي قيمة تانية بتخلي الـ kprobe تحجب تنفيذ الـ function الأصلية، وده مش اللي بنعمله هنا.

---

#### الـ `struct kprobe kp`

```c
static struct kprobe kp = {
    .symbol_name = "pinctrl_register_and_init",
    .pre_handler = pinctrl_reg_pre,
};
```

الـ `.symbol_name` بيخلي الـ kprobe subsystem يحل الـ symbol وياخد عنوانه من الـ kernel symbol table تلقائياً بدون ما نحتاج نكتب عنوان hardcoded. الـ `.pre_handler` هو الـ callback اللي بتتشغل قبل الـ function.

---

#### الـ `module_init` — `pinctrl_probe_init`

الـ `register_kprobe` بتزرع breakpoint في الـ kernel text عند عنوان الـ symbol. لو فشلت (مثلاً الـ CONFIG_KPROBES مش موجود، أو الـ symbol مش exportable) بترجع error سالب وبنرجعه مباشرة عشان الـ insmod يفشل بشكل واضح. لو نجحت بنطبع العنوان الفعلي `kp.addr` للتأكيد.

---

#### الـ `module_exit` — `pinctrl_probe_exit`

الـ `unregister_kprobe` ضروري في الـ exit عشان يشيل الـ breakpoint من الـ kernel text. لو ما شلناهوش والـ module اتحذف، الـ handler function هتبقى في ذاكرة مش موجودة وهيحصل kernel panic في أول مرة يتسجل فيها pin controller.

---

### كيفية البناء والتشغيل

**Makefile بسيط:**

```makefile
obj-m := pinctrl_probe.o

KDIR  ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

**تحميل الـ module ومتابعة الـ output:**

```bash
sudo insmod pinctrl_probe.ko
sudo dmesg | grep pinctrl_probe
# مثال output:
# [  42.123456] pinctrl_probe: kprobe planted on pinctrl_register_and_init @ ffffffffc0a12340
# [  42.789012] pinctrl_probe: registering controller "gpio0", pins=64
# [  43.001234] pinctrl_probe: registering controller "soc-pinctrl", pins=128

sudo rmmod pinctrl_probe
```

> **ملاحظة:** الـ kprobe هتشتغل بس وقت registering الـ controllers، وده بيحصل عادةً أثناء الـ boot أو لما driver جديد بيتحمّل. ممكن تشوف output فوري لو في hot-pluggable pin controller أو لو حملت driver بيسجل controller.
