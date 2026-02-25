## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ subsystem ده؟

**الـ `drivers/pinctrl/core.c`** هو **قلب نظام التحكم في الـ pins** (PIN CONTROL SUBSYSTEM) في Linux kernel. ده الـ maintainer بتاعه Linus Walleij وبيتعمل track تحت `linux-gpio@vger.kernel.org`.

---

### القصة من الأول: المشكلة اللي بيحلها

تخيل إن عندك SoC (System-on-Chip) زي Raspberry Pi أو أي ARM chip. الـ chip دي جوّاها مئات الـ **pins** — كل pin ممكن يتوصل بحاجة مختلفة:

- ممكن يبقى **GPIO** (يروح يجي إشارة رقمية).
- ممكن يبقى **UART TX/RX** (Serial).
- ممكن يبقى **SPI** أو **I2C**.
- ممكن يبقى **PWM**.
- ممكن يبقى **Ethernet**.

**المشكلة:** الـ SoC عنده عدد pins محدود، بس الـ functions أكتر من اللي هينفع يشغّلها في نفس الوقت. فمصنّع الـ chip بيعمل **multiplexing**: كل pin عنده أكتر من وظيفة، وانت بتختار كل واحد يعمل إيه عند boot.

**مثال حقيقي:** Pin رقم 42 في Raspberry Pi 4 ممكن يبقى:
- GPIO42
- UART2_TXD
- SD1_DATA2
- ARM_TRACEDATA_6

انت بتختار واحد بس — ده اللي بيتسمى **pinmux** (pin multiplexing).

وبعدين كمان، لو اخترت إنه يبقى GPIO، محتاج تحدد:
- pull-up؟ pull-down؟ floating؟
- drive strength كام mA؟
- open-drain؟

ده اللي بيتسمى **pinconf** (pin configuration).

---

### الـ core.c بالتحديد: دوره إيه؟

الـ `core.c` هو **السمسار** بين:

```
┌─────────────────────────────────────────────────────┐
│                  Device Drivers                     │
│         (SPI, I2C, UART, GPIO drivers...)           │
│    يطلبوا pins عن طريق: pinctrl_get() / devm_pinctrl_get()
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│              pinctrl CORE (core.c)                  │
│  • يحتفظ بـ global list لكل pin controllers       │
│  • يحتفظ بـ mapping table (مين يستخدم إيه)       │
│  • يدير states: default / sleep / init / idle      │
│  • يربط بين GPIO subsystem و pinctrl               │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│          Hardware Pin Controller Drivers            │
│  (مثلاً: pinctrl-bcm2835.c, pinctrl-imx.c ...)    │
│  بيسجّلوا نفسهم عن طريق: pinctrl_register()       │
└─────────────────────────────────────────────────────┘
```

---

### الـ Lifecycle: إزاي بيشتغل كل ده؟

**1. عند boot الـ kernel:**
   - كل driver للـ hardware pin controller (مثلاً للـ BCM2835) بيسجّل نفسه عن طريق `pinctrl_register()` أو `pinctrl_register_and_init()`.
   - ده بيعمل `struct pinctrl_dev` ويضيفه للـ global list `pinctrldev_list`.

**2. عند boot device معين (مثلاً UART driver):**
   - الـ device core بيعمل `devm_pinctrl_get(dev)` تلقائياً عند probe.
   - الـ core.c بيدور في الـ `pinctrl_maps` (mapping table) على الـ entries اللي بتخص الـ device ده.
   - بيبني `struct pinctrl` بكل الـ states الممكنة (default, sleep, init...).

**3. عند تفعيل state معين:**
   - الـ driver بيعمل `pinctrl_select_state(p, state)`.
   - الـ core.c بيطبّق الـ settings: أول pinmux، بعدين pinconf.
   - لو فيه state قديم، بيعمل disable ليه الأول.

**4. عند Power Management:**
   - عند suspend: `pinctrl_pm_select_sleep_state()` → pins بتاخد config تخلّيها safe في النوم.
   - عند resume: `pinctrl_pm_select_default_state()` → رجّعها للـ default.

---

### مفاهيم أساسية جوّا الـ core.c

| المفهوم | تعريفه |
|---|---|
| **`pinctrl_dev`** | الـ pin controller hardware نفسه (struct واحدة لكل chip) |
| **`pinctrl`** | handle بيمسكه كل device يستخدم pins |
| **`pinctrl_state`** | مجموعة settings بتتطبق مع بعض (مثلاً "default") |
| **`pinctrl_setting`** | setting واحدة: إما mux group أو pin config |
| **`pinctrl_map`** | الـ mapping table — "الـ device ده في الـ state ده يستخدم الـ function دي على الـ group ده" |
| **`pin_desc`** | descriptor لكل pin فيزيائي، محفوظ في radix tree |
| **`pinctrl_gpio_range`** | ربط بين GPIO numbers وpin numbers |

---

### الـ States الافتراضية

الـ kernel بيعرّف states معيارية في `pinctrl-state.h`:

```c
#define PINCTRL_STATE_DEFAULT  "default"  /* الحالة العادية */
#define PINCTRL_STATE_INIT     "init"     /* عند probe قبل ما يخلص */
#define PINCTRL_STATE_IDLE     "idle"     /* خامل */
#define PINCTRL_STATE_SLEEP    "sleep"    /* نائم */
```

**الـ hog:** لو الـ pin controller نفسه محتاج pins لنفسه (self-use)، بيعمل `pinctrl_claim_hogs()` — بيطبّق الـ "default" state على نفسه فور ما يتسجّل.

---

### الربط مع GPIO Subsystem

الـ `core.c` بيعمل جسر بين الـ GPIO world والـ pinctrl world:

```
GPIO Driver يطلب: gpio_request(42)
         ↓
pinctrl_gpio_request(gc, offset)
         ↓
core.c يترجم offset → pin number (عن طريق gpio_to_pin)
         ↓
pinmux_request_gpio(pctldev, range, pin, gpio)
         ↓
Hardware pin controller driver ينفّذ
```

الـ `pinctrl_gpio_range` هو الجسر: بيعرف إن GPIO numbers [X..Y] بتتطابق مع Pin numbers [A..B].

---

### الـ debugfs

الـ core.c بيفتح `/sys/kernel/debug/pinctrl/` ويضم:
- **`pinctrl-devices`** — كل pin controllers المسجّلين.
- **`pinctrl-maps`** — الـ mapping table كلها.
- **`pinctrl-handles`** — كل devices بتستخدم pinctrl وstates بتاعتها.
- Per-device: **`pins`**, **`pingroups`**, **`gpio-ranges`**.

---

### ملفات الـ Subsystem الأساسية

#### الـ Core (في `drivers/pinctrl/`)

| الملف | الدور |
|---|---|
| `core.c` | **القلب**: تسجيل controllers، إدارة states، GPIO bridge |
| `core.h` | الـ structs الداخلية: `pinctrl_dev`, `pinctrl`, `pin_desc`, `pinctrl_state`... |
| `pinmux.c` | تنفيذ الـ multiplexing (تحديد وظيفة الـ pin) |
| `pinmux.h` | واجهة الـ pinmux داخلياً |
| `pinconf.c` | تطبيق الـ pin configuration (pull, drive strength...) |
| `pinconf.h` | واجهة الـ pinconf داخلياً |
| `devicetree.c` | تحليل الـ device tree لبناء الـ mapping table |
| `devicetree.h` | واجهة الـ DT parsing |

#### الـ Headers (في `include/linux/pinctrl/`)

| الملف | الدور |
|---|---|
| `pinctrl.h` | الـ structs العامة: `pinctrl_desc`, `pinctrl_ops`, `pinctrl_pin_desc` |
| `consumer.h` | الـ API للـ drivers اللي تستهلك pins: `pinctrl_get`, `pinctrl_select_state`... |
| `machine.h` | الـ mapping table API: `pinctrl_map`, `pinctrl_register_mappings` |
| `pinconf.h` | الـ pin config API |
| `pinmux.h` | الـ pinmux API |
| `pinctrl-state.h` | أسماء الـ states المعيارية (default, sleep...) |
| `devinfo.h` | `dev_pin_info` — الـ pin info المحفوظة جوّا `struct device` |

#### أمثلة Hardware Drivers

| الملف | الـ Hardware |
|---|---|
| `bcm/pinctrl-bcm2835.c` | Raspberry Pi (BCM2835/2711) |
| `freescale/pinctrl-imx.c` | NXP i.MX series |
| `intel/pinctrl-intel.c` | Intel PCH/SoC |
| `pinctrl-amd.c` | AMD platforms |
| `samsung/pinctrl-samsung.c` | Samsung Exynos |
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

على أي SoC حديث (Qualcomm، NXP، STM32، Rockchip...)، كل pin فيزيائي على الـ chip ممكن يشتغل بأكتر من وظيفة. نفس الـ pin ممكن يكون:

- GPIO عادي (input/output)
- TX لـ UART
- SCL لـ I2C
- CLK لـ SPI
- أو حتى وظيفة custom للـ SoC زي camera interface

الـ hardware بيتحكم في ده عن طريق **مسجلات MUX** في الـ SoC، وكل pin كمان ممكن يتضبط: pull-up، pull-down، drive strength، slew rate، open-drain. الـ problem إن:

1. كل driver كان بيكتب كوده الخاص لضبط الـ pins — مفيش standardization.
2. لو driver تاني احتاج نفس الـ pin، مفيش coordination — ممكن يحصل تعارض.
3. الـ pin configuration كانت بتتحط hardcoded في الـ board files أو حتى داخل الـ drivers أنفسهم.
4. مع DeviceTree، محتاج طريقة موحدة تربط الـ device بالـ pin configuration بتاعته.

---

### الحل — النهج اللي اتخذه الـ Kernel

الـ **pinctrl subsystem** (اتضاف في Linux 3.2) بيوفر:

- **abstraction layer موحدة** بين الـ consumers (الـ drivers اللي محتاجة pins) والـ providers (الـ hardware pin controller driver).
- **نظام states**: كل device ممكن يكون ليه أكتر من حالة (`default`، `sleep`، `idle`، `init`) وتتغير بينهم أثناء runtime أو PM.
- **نظام maps**: جدول بيربط كل device بالـ pin groups والـ functions المطلوبة.
- **arbitration**: لو اتنين drivers طلبوا نفس الـ pin بـ function مختلفة، الـ core يرفض ويرجع error.

---

### تشبيه من الواقع — مكتب الحجز (Booking Office)

تخيل إن كل **pin فيزيائي** هو **قاعة اجتماعات** في مبنى (= الـ SoC). القاعة دي ممكن تتحجز لـ:
- اجتماع مجلس الإدارة (= UART)
- ورشة عمل (= SPI)
- استخدام حر (= GPIO)

الـ **pin controller driver** = مكتب الحجز (بيعرف كل القاعات وقواعدها).
الـ **pinctrl core** = نظام الحجز المركزي (بيتأكد مفيش تعارض وبيحتفظ بسجل لكل الحجوزات).
الـ **consumer driver** (UART، SPI...) = الجهة اللي بتطلب الحجز.

الـ **pinctrl_map** = نموذج الطلب (أنا "UART0"، محتاج "uart0_grp"، في state "default").
الـ **pinctrl_state** = تذكرة الحجز اللي بتتحفظ جنب الجهة الطالبة.
الـ **pinctrl_select_state()** = تفعيل الحجز فعلياً (بيبعت الأوامر للـ hardware).

لو الجهة راحت نامت (suspend)، الحجز بيتغير لـ "sleep state" — زي إنك بتقفل القاعة وتديها وظيفة تانية مؤقتاً.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Consumer Drivers Layer                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │  UART    │  │  SPI     │  │  I2C     │  │  GPIO driver │  │
│  │  driver  │  │  driver  │  │  driver  │  │  (gpiolib)   │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘  │
│       │              │              │                │          │
│       └──────────────┴──────────────┴────────────── ┤          │
│                                                      │          │
│              pinctrl_get() / pinctrl_select_state()  │          │
│              pinctrl_gpio_request() ─────────────────┘          │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                    pinctrl CORE  (core.c)                        │
│                                                                  │
│  ┌─────────────────────┐    ┌────────────────────────────────┐ │
│  │  Global Lists        │    │  State Machine                 │ │
│  │  pinctrl_list        │    │  pinctrl_get()                 │ │
│  │  pinctrldev_list     │    │  pinctrl_lookup_state()        │ │
│  │  pinctrl_maps        │    │  pinctrl_select_state()        │ │
│  └─────────────────────┘    │  pinctrl_commit_state()        │ │
│                              └────────────────────────────────┘ │
│  ┌─────────────────────┐    ┌────────────────────────────────┐ │
│  │  Map Table Engine   │    │  GPIO Bridge                   │ │
│  │  pinctrl_map        │    │  pinctrl_gpio_request()        │ │
│  │  add_setting()      │    │  pinctrl_gpio_direction_*()    │ │
│  │  create_pinctrl()   │    │  gpio_to_pin()                 │ │
│  └─────────────────────┘    └────────────────────────────────┘ │
│                                                                  │
│           ┌──────────────┬──────────────┐                       │
│           │   pinmux.c   │  pinconf.c   │  (sub-modules)        │
│           └──────┬───────┴──────┬───────┘                       │
└──────────────────┼──────────────┼────────────────────────────────┘
                   │              │
┌──────────────────▼──────────────▼────────────────────────────────┐
│              Pin Controller Driver (Provider)                     │
│                                                                   │
│  e.g. pinctrl-bcm2835.c / pinctrl-stm32.c / pinctrl-msm.c       │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  struct pinctrl_desc {                                     │  │
│  │    pins[]    ← array of all physical pins                  │  │
│  │    pctlops   ← groups/DT parsing ops                       │  │
│  │    pmxops    ← mux switching ops                           │  │
│  │    confops   ← pull/drive-strength ops                     │  │
│  │  }                                                         │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                    │
│                     Talks directly to HW registers                │
└───────────────────────────────────────────────────────────────────┘
                               │
                ┌──────────────▼─────────────┐
                │      Physical SoC Hardware  │
                │  MUX regs / PAD config regs │
                └────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ abstraction الأساسية في الـ pinctrl subsystem هي **الفصل بين ثلاث مفاهيم**:

| المفهوم | التمثيل في الكود | المعنى |
|--------|-----------------|--------|
| **Pin** | `struct pinctrl_pin_desc` + `struct pin_desc` | وحدة فيزيائية واحدة (pad/ball/finger على الـ SoC) |
| **Group** | `struct pingroup` / `struct group_desc` | مجموعة pins تشتغل كوحدة واحدة (e.g. UART0_TX + UART0_RX) |
| **Function** | `struct pinfunction` | الوظيفة المطلوبة للـ group (e.g. "uart0"، "spi1"، "gpio") |

الـ **State System** هو الطبقة اللي فوقيهم: كل device بتربط اسم state (string) بـ set من الـ settings (mux + config).

---

### شرح الـ Data Structures بالتفصيل

#### طبقة الـ Provider — ما بيسجله الـ driver

```c
/* الـ descriptor اللي الـ driver بيبعته للـ core */
struct pinctrl_desc {
    const char *name;                    /* اسم الـ controller */
    const struct pinctrl_pin_desc *pins; /* مصفوفة كل الـ pins */
    unsigned int npins;                  /* عددهم */
    const struct pinctrl_ops *pctlops;   /* vtable للـ groups */
    const struct pinmux_ops *pmxops;     /* vtable للـ muxing */
    const struct pinconf_ops *confops;   /* vtable للـ config */
    struct module *owner;
    bool link_consumers;                 /* device link للـ PM ordering */
};

/* وصف كل pin فيزيائي */
struct pinctrl_pin_desc {
    unsigned int number;  /* رقم فريد في الـ pin space */
    const char *name;     /* e.g. "GPIO_0", "PA1" */
    void *drv_data;       /* بيانات driver-specific */
};
```

الـ core بيأخد الـ `pinctrl_desc` ويعمل منه `pinctrl_dev` — وده هو الـ runtime object:

```c
struct pinctrl_dev {
    struct list_head node;              /* عقدة في pinctrldev_list العالمية */
    const struct pinctrl_desc *desc;   /* مؤشر للـ descriptor */

    /* كل pin بيتخزن في radix tree بـ pin number كـ key */
    struct radix_tree_root pin_desc_tree;

    /* الـ groups والـ functions في radix trees منفصلة */
    struct radix_tree_root pin_group_tree;
    struct radix_tree_root pin_function_tree;

    struct list_head gpio_ranges;       /* ربط مع الـ GPIO subsystem */
    struct device *dev;

    /* الـ hog: الـ controller نفسه بيحجز بعض الـ pins لنفسه */
    struct pinctrl *p;
    struct pinctrl_state *hog_default;
    struct pinctrl_state *hog_sleep;

    struct mutex mutex;
};
```

**الـ radix tree**: نوع من الـ prefix trees بيستخدمها الـ kernel لـ sparse index lookup بـ O(log n). هنا بتخزن الـ `pin_desc` objects بـ pin number كـ key — efficient لأن الـ pin space ممكن يكون sparse (مش كل الأرقام مستخدمة).

#### طبقة الـ Consumer — ما بيستخدمه الـ driver العادي

```c
/* الـ handle الرئيسي للـ consumer */
struct pinctrl {
    struct list_head node;        /* في pinctrl_list العالمية */
    struct device *dev;           /* الـ device المالك */
    struct list_head states;      /* قائمة الـ states المتاحة */
    struct pinctrl_state *state;  /* الـ state الحالية */
    struct list_head dt_maps;     /* maps اتجنبت من DT */
    struct kref users;            /* reference counting */
};

/* كل state ليها اسم وقائمة settings */
struct pinctrl_state {
    struct list_head node;
    const char *name;           /* "default", "sleep", "init", ... */
    struct list_head settings;  /* قائمة struct pinctrl_setting */
};

/* كل setting هي إما mux أو config */
struct pinctrl_setting {
    struct list_head node;
    enum pinctrl_map_type type;   /* MUX_GROUP أو CONFIGS_PIN أو CONFIGS_GROUP */
    struct pinctrl_dev *pctldev; /* الـ controller المسؤول */
    const char *dev_name;
    union {
        struct pinctrl_setting_mux mux;         /* group selector + func selector */
        struct pinctrl_setting_configs configs;  /* array of config params */
    } data;
};
```

#### طبقة الـ Mapping Table

الـ **mapping table** هي الجسر بين الـ consumer وتفاصيل الـ hardware. بتتسجل مرة واحدة (من DT أو من board file) وبتتخزن في الـ global list:

```c
/* entry واحدة في الـ mapping table */
struct pinctrl_map {
    const char *dev_name;       /* اسم الـ consumer device */
    const char *name;           /* اسم الـ state ("default", "sleep") */
    enum pinctrl_map_type type;
    const char *ctrl_dev_name;  /* اسم الـ pin controller */
    union {
        struct pinctrl_map_mux mux;         /* group name + function name */
        struct pinctrl_map_configs configs; /* pin/group name + config values */
    } data;
};
```

---

### علاقة الـ Structs ببعض — رسم تخطيطي

```
pinctrl_map[] (static/DT)          pinctrl_dev (runtime, per controller)
      │                                      │
      │ pinctrl_register_mappings()          │ pinctrl_register()
      ▼                                      ▼
pinctrl_maps list ─────────────► used during create_pinctrl()
      │
      ▼ for_each_pin_map()
      │ add_setting()
      │
      ▼
  struct pinctrl (per consumer device)
  ├── dev*
  ├── kref (users)
  └── states list
        ├── pinctrl_state ("default")
        │     └── settings list
        │           ├── pinctrl_setting (MUX_GROUP)
        │           │     └── data.mux = {group=2, func=1}
        │           └── pinctrl_setting (CONFIGS_PIN)
        │                 └── data.configs = {pin=5, configs=[PULL_UP]}
        └── pinctrl_state ("sleep")
              └── settings list
                    └── pinctrl_setting (MUX_GROUP)
                          └── data.mux = {group=2, func=0 (GPIO)}
```

---

### الـ Pin Descriptor — ما بيعيش في الـ Core

لكل pin فيزيائي، الـ core بيخلق `pin_desc` بعد التسجيل:

```c
struct pin_desc {
    struct pinctrl_dev *pctldev; /* مين يملكه */
    const char *name;            /* اسمه */
    bool dynamic_name;           /* اتولد تلقائياً "PIN5"؟ */
    void *drv_data;              /* driver-specific data */

    /* بيانات الـ mux (لو CONFIG_PINMUX مفعّل) */
    unsigned int mux_usecount;   /* عدد الـ consumers الحاليين */
    const char *mux_owner;       /* اسم الـ device اللي حاجز */
    const char *gpio_owner;      /* لو GPIO subsystem حاجزه */
    struct mutex mux_lock;       /* serialize access */
};
```

الـ `mux_usecount` هو الـ arbitration mechanism الأساسي. لو pin اتطلب من driver تاني والـ `mux_usecount > 0` وبـ function مختلفة، الـ core بيرفض.

---

### Flow الـ Registration — من الـ Driver للـ Hardware

```
Pin Controller Driver:
    1. يعبي struct pinctrl_desc
    2. يستدعي pinctrl_register_and_init()
             │
             ▼
    pinctrl_init_controller()
    ├── kzalloc(pinctrl_dev)
    ├── INIT_RADIX_TREE(pin_desc_tree)
    ├── pinctrl_check_ops() ← يتأكد pctlops موجودة
    ├── pinmux_check_ops() ← لو pmxops موجودة
    ├── pinconf_check_ops() ← لو confops موجودة
    └── pinctrl_register_pins()
            └── for each pin: pinctrl_register_one_pin()
                    └── kzalloc(pin_desc)
                        radix_tree_insert(pin_desc_tree, pin_number, pindesc)

    3. يستدعي pinctrl_enable()
             │
             ▼
    pinctrl_claim_hogs()  ← الـ controller يحجز الـ pins اللي بتاعته
    list_add_tail(pinctrldev_list)  ← يبقى visible للكل
    pinctrl_init_device_debugfs()  ← يعمل entries في /sys/kernel/debug/pinctrl/
```

---

### Flow الـ Consumer — كيف Driver يستخدم الـ Pinctrl

```c
/* في probe() */
struct pinctrl *p = devm_pinctrl_get(dev);
/*
 * devm_pinctrl_get()
 * └── pinctrl_get()
 *     └── create_pinctrl()
 *         ├── pinctrl_dt_to_map()     ← قراءة DT nodes
 *         └── for_each_pin_map():
 *             └── add_setting()
 *                 ├── find/create_state() بـ اسم الـ state
 *                 └── pinmux_map_to_setting() أو pinconf_map_to_setting()
 *                     (بيحول الأسماء لـ selectors أرقام)
 */

struct pinctrl_state *s = pinctrl_lookup_state(p, PINCTRL_STATE_DEFAULT);
pinctrl_select_state(p, s);
/*
 * pinctrl_select_state()
 * └── pinctrl_commit_state()
 *     ├── pinmux_disable_setting() ← يعطل الـ state القديمة
 *     ├── for each setting (pinmux first):
 *     │   └── pinmux_enable_setting()  ← يستدعي pmxops->set_mux()
 *     └── for each setting (pinconf after):
 *         └── pinconf_apply_setting() ← يستدعي confops->pin_config_set()
 */
```

---

### الـ GPIO Bridge — ربط gpiolib بالـ Pinctrl

ده subsystem تاني محتاج شرح مبدئي: **الـ GPIO subsystem** (gpiolib) هو الـ layer المسؤولة عن الـ GPIO API (`gpio_request`، `gpio_direction_input`...). في الـ ARM SoCs، الـ GPIO والـ pinmux موجودين في نفس الـ hardware block. الـ pinctrl core بيوفر bridge functions:

```c
/* الـ gpiolib بتستدعيها لما user يطلب GPIO */
int pinctrl_gpio_request(struct gpio_chip *gc, unsigned int offset)
{
    /* 1. البحث عن الـ gpio range المناسبة */
    pinctrl_get_device_gpio_range(gc, offset, &pctldev, &range);

    /* 2. تحويل GPIO number لـ pin number */
    pin = gpio_to_pin(range, gc, offset);

    /* 3. تأكيد إن الـ pin مش متحجز لـ function تانية */
    pinmux_request_gpio(pctldev, range, pin, gc->base + offset);
}
```

الـ **`pinctrl_gpio_range`** هو الجسر اللي بيربط الـ GPIO number space بالـ pin number space:

```c
struct pinctrl_gpio_range {
    unsigned int base;      /* أول GPIO number في الـ range */
    unsigned int pin_base;  /* أول pin number مقابله (لو linear) */
    unsigned int npins;
    unsigned int const *pins; /* أو array غير linear */
    struct gpio_chip *gc;
};
```

---

### الـ State Machine للـ PM

الـ pinctrl subsystem ملتحم مع الـ **Power Management subsystem**. الـ device core بيستدعي تلقائياً:

```
device probe:
    ├── pinctrl_bind_pins()    ← بيعمل pinctrl_get() وبيحجز كل الـ states
    └── pinctrl_init_done()   ← بعد probe ينهي، يتحول من "init" لـ "default"

device suspend:
    └── pinctrl_pm_select_sleep_state()  → pinctrl_select_state(sleep)

device resume:
    └── pinctrl_pm_select_default_state() → pinctrl_select_state(default)
```

الـ states المعرفة standard في `pinctrl-state.h`:
- `"default"` — الحالة الطبيعية للـ device
- `"init"` — حالة مؤقتة أثناء الـ probe
- `"sleep"` — الحالة أثناء الـ suspend
- `"idle"` — حالة وسطية (runtime PM idle)

---

### ما الـ Core يمتلكه مقابل ما يفوضه للـ Driver

| المسؤولية | الـ Pinctrl Core يمتلكها | يُفوِّضها للـ Driver |
|------------|--------------------------|----------------------|
| قائمة الـ controllers العالمية | `pinctrldev_list` ✓ | — |
| قائمة الـ consumer handles | `pinctrl_list` ✓ | — |
| الـ mapping table | `pinctrl_maps` ✓ | — |
| تخزين الـ pin descriptors | radix tree في `pinctrl_dev` ✓ | — |
| arbitration (مين يملك الـ pin؟) | `mux_usecount` + `mux_owner` ✓ | — |
| ربط الـ GPIO بالـ pinctrl | `pinctrl_gpio_request()` ✓ | — |
| تحويل الأسماء لـ selectors | `pinctrl_get_group_selector()` ✓ | — |
| **كتابة الـ MUX registers فعلياً** | — | `pmxops->set_mux()` ✓ |
| **كتابة الـ PAD config registers** | — | `confops->pin_config_set()` ✓ |
| **تعريف الـ pin groups** | — | `pctlops->get_group_pins()` ✓ |
| **تفسير الـ DT pin nodes** | — | `pctlops->dt_node_to_map()` ✓ |
| **بيانات الـ pins الخاصة بالـ SoC** | — | `pin->drv_data` ✓ |

---

### مثال واقعي — STM32 Pin Configuration في الـ DT

```dts
/* في device tree: ربط UART1 بالـ pins */
&usart1 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&usart1_pins_a>;   /* default state */
    pinctrl-1 = <&usart1_sleep_pins_a>; /* sleep state */
};

usart1_pins_a: usart1-0 {
    pins1 {
        pinmux = <STM32_PINMUX('A', 9, AF7)>;  /* PA9 = TX, AF7=USART1 */
        bias-disable;
        drive-push-pull;
        slew-rate = <0>;
    };
    pins2 {
        pinmux = <STM32_PINMUX('A', 10, AF7)>; /* PA10 = RX */
        bias-disable;
    };
};
```

الـ `dt_node_to_map()` في الـ STM32 driver بيقرأ الـ DT nodes دي ويحولها لـ `pinctrl_map[]` entries، الـ core بعدين بيسجلها ويبنيها في الـ `pinctrl_state` للـ UART device.

---

### الـ Hog Mechanism — الـ Controller يحجز لنفسه

ده concept مهم: الـ pin controller نفسه ممكن يحتاج pins (مثلاً نظام الـ GPIOs اللي موجودة على الـ PMIC). الـ core بعد التسجيل بيستدعي `pinctrl_claim_hogs()` — بيدور في الـ mapping table على الـ entries اللي الـ `dev_name == ctrl_dev_name` (يعني الـ consumer والـ controller نفس الـ device):

```c
static int pinctrl_claim_hogs(struct pinctrl_dev *pctldev)
{
    /* بيعمل pinctrl handle للـ controller device نفسه */
    pctldev->p = create_pinctrl(pctldev->dev, pctldev);

    /* بيطبق الـ default state فوراً */
    pctldev->hog_default = pinctrl_lookup_state(pctldev->p,
                                                PINCTRL_STATE_DEFAULT);
    pinctrl_select_state(pctldev->p, pctldev->hog_default);
    ...
}
```

---

### الـ Dummy State — Graceful Degradation

لو platform مفيهاش pinctrl driver (embedded systems بسيطة)، الـ drivers اللي بتستخدم pinctrl API هتفشل. الحل:

```c
void pinctrl_provide_dummies(void)
{
    pinctrl_dummy_state = true;
}
```

لو الـ flag ده اتعمل، `pinctrl_lookup_state()` بترجع dummy state فاضية بدل error — الـ driver يعمل `select_state()` بدون ما يكتب أي حاجة في الـ hardware.

---

### الـ debugfs Interface

الـ core بيعمل في `/sys/kernel/debug/pinctrl/` تلقائياً:

```
/sys/kernel/debug/pinctrl/
├── pinctrl-devices       ← قائمة كل الـ controllers المسجلة
├── pinctrl-maps          ← كل الـ mapping entries
├── pinctrl-handles       ← كل الـ consumers وحالتهم الحالية
└── <controller-name>/
    ├── pins              ← كل pin وحالته (مين حاجزه؟)
    ├── pingroups         ← كل group وأعضاؤه
    ├── gpio-ranges       ← الربط مع الـ GPIO subsystem
    ├── pinmux-functions  ← الـ functions المتاحة
    └── pinmux-pins       ← مين حاجز إيه
```

ده أداة debugging أساسية — لو pin مش شغال زي المتوقع، `cat pins` بيوريك على طول مين حاجزه وبـ إيه function.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Config Options — Cheatsheet

#### `enum pinctrl_map_type`

| القيمة | المعنى |
|--------|--------|
| `PIN_MAP_TYPE_INVALID` | نوع غلط، يُستخدم كـ sentinel |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمي لمنصات بلا pinctrl driver |
| `PIN_MAP_TYPE_MUX_GROUP` | ربط group بـ function معينة (pinmux) |
| `PIN_MAP_TYPE_CONFIGS_PIN` | تطبيق config على pin واحد (pull/drive/etc.) |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | تطبيق config على group كاملة |

#### Config Options مهمة

| الـ Kconfig | الأثر |
|-------------|-------|
| `CONFIG_PINMUX` | يُفعّل حقول `mux_usecount`, `mux_owner`, `mux_lock` في `pin_desc` |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | يُفعّل `pin_group_tree` و`num_groups` في `pinctrl_dev` |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | يُفعّل `pin_function_tree` و`num_functions` في `pinctrl_dev` |
| `CONFIG_GENERIC_PINCONF` | يُفعّل custom params في `pinctrl_desc` |
| `CONFIG_GPIOLIB` | يُفعّل منطق الربط بين GPIO وpin |
| `CONFIG_DEBUG_FS` | يُفعّل `device_root` وكل دوال الـ debugfs |
| `CONFIG_PM` | يُفعّل `pinctrl_pm_select_*_state()` |
| `CONFIG_OF` | يُفعّل `pinctrl_dt_to_map()` وتحليل device tree |

#### States المعرّفة (من `pinctrl-state.h`)

| الاسم | المعنى |
|-------|--------|
| `PINCTRL_STATE_DEFAULT` | `"default"` — الحالة العادية أثناء التشغيل |
| `PINCTRL_STATE_INIT` | `"init"` — أثناء probe قبل ما الـ driver يخلص |
| `PINCTRL_STATE_SLEEP` | `"sleep"` — عند دخول suspend |
| `PINCTRL_STATE_IDLE` | `"idle"` — عند idle runtime PM |

#### الـ Flag الوحيد لـ `pinfunction`

| الـ Flag | القيمة | المعنى |
|---------|--------|--------|
| `PINFUNCTION_FLAG_GPIO` | `BIT(0)` | الـ function دي هي GPIO (مش peripheral) |

---

### 1. الـ Structs المهمة

#### `struct pinctrl_dev`
**الغرض:** الـ struct الرئيسي اللي يمثّل controller الـ pins الفيزيائي (مثلاً: `pinctrl@fe000000` في device tree). بيتخزن في القائمة العالمية `pinctrldev_list`.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `node` | `list_head` | ربطه في `pinctrldev_list` العالمية |
| `desc` | `*pinctrl_desc` | الـ descriptor اللي بعته الـ driver (const) |
| `pin_desc_tree` | `radix_tree_root` | شجرة بتربط pin number → `pin_desc*` |
| `pin_group_tree` | `radix_tree_root` | شجرة بتربط selector index → `group_desc*` |
| `num_groups` | `unsigned int` | عدد groups المسجّلة |
| `pin_function_tree` | `radix_tree_root` | شجرة بتربط selector → function descriptor |
| `num_functions` | `unsigned int` | عدد functions المسجّلة |
| `gpio_ranges` | `list_head` | قائمة بـ `pinctrl_gpio_range` المرتبطة |
| `dev` | `*device` | الـ device object بتاع الـ controller |
| `owner` | `*module` | الموديول اللي سجّل الـ controller |
| `driver_data` | `void*` | بيانات خاصة بالـ driver |
| `p` | `*pinctrl` | الـ pinctrl handle للـ "hog" (الـ controller بيستخدم pins لنفسه) |
| `hog_default` | `*pinctrl_state` | الـ state الـ default للـ hog |
| `hog_sleep` | `*pinctrl_state` | الـ state الـ sleep للـ hog |
| `mutex` | `mutex` | يحمي عمليات الـ controller نفسه |
| `device_root` | `*dentry` | جذر مجلد debugfs للـ controller |

---

#### `struct pinctrl_desc`
**الغرض:** الـ descriptor الثابت اللي بيكتبه الـ driver ويبعته لـ `pinctrl_register()`. بيصف قدرات الـ controller.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `name` | `const char*` | اسم الـ controller (مطلوب) |
| `pins` | `*pinctrl_pin_desc[]` | array الـ pins الفيزيائية |
| `npins` | `unsigned int` | عدد الـ pins في الـ array |
| `pctlops` | `*pinctrl_ops` | عمليات تحكم الـ pins الأساسية |
| `pmxops` | `*pinmux_ops` | عمليات الـ pinmux (اختياري) |
| `confops` | `*pinconf_ops` | عمليات الـ pin config (اختياري) |
| `owner` | `*module` | `THIS_MODULE` |
| `num_custom_params` | `unsigned int` | عدد البارامترات الخاصة بالـ driver |
| `custom_params` | `*pinconf_generic_params` | البارامترات الخاصة |
| `custom_conf_items` | `*pin_config_item` | طريقة طباعتها في debugfs |
| `link_consumers` | `bool` | اعمل device link مع الـ consumers (للـ PM ordering) |

---

#### `struct pinctrl_ops`
**الغرض:** جدول العمليات اللي لازم الـ driver driver يوفّرها للـ core عشان يقدر يدير الـ groups والـ pins.

| الـ function pointer | متطلب؟ | الوظيفة |
|---------------------|--------|---------|
| `get_groups_count` | **مطلوب** | يرجع عدد الـ pin groups |
| `get_group_name` | **مطلوب** | يرجع اسم group بـ selector |
| `get_group_pins` | اختياري | يرجع الـ pins اللي في group معينة |
| `pin_dbg_show` | اختياري | يطبع معلومات خاصة بالـ driver في debugfs |
| `dt_node_to_map` | اختياري (DT only) | يحوّل DT node لـ mapping table entries |
| `dt_free_map` | اختياري (DT only) | يحرر الـ mapping entries اللي اتعملت |

---

#### `struct pinctrl`
**الغرض:** الـ handle اللي بيحصل عليه الـ consumer device لما بيطلب `pinctrl_get()`. بيخزن كل الـ states المتاحة للـ device ده. بيتخزن في `pinctrl_list` العالمية.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `node` | `list_head` | ربطه في `pinctrl_list` العالمية |
| `dev` | `*device` | الـ consumer device |
| `states` | `list_head` | قائمة بـ `pinctrl_state` objects |
| `state` | `*pinctrl_state` | الـ state الحالي المفعّل |
| `dt_maps` | `list_head` | mapping entries جت من device tree |
| `users` | `kref` | reference count (يدعم `pinctrl_get/put`) |

---

#### `struct pinctrl_state`
**الغرض:** يمثّل state واحد (مثلاً `"default"` أو `"sleep"`) لـ consumer device معين. كل state بيحتوي على قائمة من الـ settings اللي لازم تتطبّق على الـ hardware.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `node` | `list_head` | ربطه في قائمة `pinctrl->states` |
| `name` | `const char*` | اسم الـ state (e.g. `"default"`, `"sleep"`) |
| `settings` | `list_head` | قائمة بـ `pinctrl_setting` objects |

---

#### `struct pinctrl_setting`
**الغرض:** يمثّل setting واحد داخل state — إما mux group أو pin/group config. هو الوحدة الأصغر اللي بتتطبّق على الـ hardware.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `node` | `list_head` | ربطه في `pinctrl_state->settings` |
| `type` | `enum pinctrl_map_type` | نوع الـ setting (MUX أو CONFIG) |
| `pctldev` | `*pinctrl_dev` | الـ controller المسؤول عن تطبيق هذا الـ setting |
| `dev_name` | `const char*` | اسم الـ consumer device |
| `data.mux` | `pinctrl_setting_mux` | group selector + function selector (لـ MUX) |
| `data.configs` | `pinctrl_setting_configs` | pin/group + configs array (لـ CONFIG) |

---

#### `struct pin_desc`
**الغرض:** descriptor لكل pin فيزيائي في النظام. بيتخزن في `pinctrl_dev->pin_desc_tree` (radix tree). بيتعمل واحد لكل pin عند التسجيل.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `pctldev` | `*pinctrl_dev` | الـ controller اللي بيمتلك هذا الـ pin |
| `name` | `const char*` | اسم الـ pin (مثلاً `"GPIO0"`) |
| `dynamic_name` | `bool` | لو الاسم اتعمل بـ `kasprintf()` ومش من الـ driver |
| `drv_data` | `void*` | بيانات خاصة بالـ driver لهذا الـ pin |
| `mux_usecount` | `unsigned int` | عدد المرات اللي طُلب فيها هذا الـ pin للـ mux |
| `mux_owner` | `const char*` | اسم الـ device اللي بيمتلك هذا الـ pin حالياً |
| `mux_setting` | `*pinctrl_setting_mux` | الـ mux setting الحالي المطبّق |
| `gpio_owner` | `const char*` | اسم الـ GPIO اللي طلبه لو الـ pin في GPIO mode |
| `mux_lock` | `mutex` | يحمي عمليات الـ mux على هذا الـ pin |

---

#### `struct pinctrl_map`
**الغرض:** المدخل الأساسي في mapping table — بيربط device باسم state بـ controller وبيقول نوع العملية (MUX أو CONFIG). الـ board/machine بتوفّر array منها وتسجّلها بـ `pinctrl_register_mappings()`.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `dev_name` | `const char*` | اسم الـ consumer device |
| `name` | `const char*` | اسم الـ state (e.g. `"default"`) |
| `type` | `enum pinctrl_map_type` | نوع الـ mapping |
| `ctrl_dev_name` | `const char*` | اسم الـ pin controller المسؤول |
| `data.mux` | `pinctrl_map_mux` | اسم الـ group واسم الـ function |
| `data.configs` | `pinctrl_map_configs` | اسم الـ pin/group + configs array |

---

#### `struct pinctrl_maps`
**الغرض:** wrapper بسيط يحتوي على array من `pinctrl_map`. بيتخزن في القائمة العالمية `pinctrl_maps`. سبب وجوده: عشان يقدر يتسجّل أكتر من mapping table من أكتر من مكان.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `node` | `list_head` | ربطه في `pinctrl_maps` العالمية |
| `maps` | `*pinctrl_map` | pointer للـ array الأصلي (مش بيعمله copy) |
| `num_maps` | `unsigned int` | عدد entries في الـ array |

---

#### `struct pinctrl_gpio_range`
**الغرض:** بيعرّف نطاق GPIO numbers اللي بيديره الـ pin controller. بيتخزن في `pinctrl_dev->gpio_ranges`. بيسمح للـ GPIO subsystem إنها تتواصل مع الـ pinctrl subsystem.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `node` | `list_head` | ربطه في `pinctrl_dev->gpio_ranges` |
| `name` | `const char*` | اسم الـ GPIO chip |
| `id` | `unsigned int` | رقم تعريفي |
| `base` | `unsigned int` | أول GPIO number في النطاق (global space) |
| `pin_base` | `unsigned int` | أول pin number مقابله (لو linear) |
| `npins` | `unsigned int` | عدد الـ pins في النطاق |
| `pins` | `*unsigned int` | array من pin numbers (لو non-linear)، لو NULL يبقى linear |
| `gc` | `*gpio_chip` | pointer للـ GPIO chip المرتبطة |

---

#### `struct group_desc`
**الغرض:** generic descriptor لـ pin group (يُستخدم مع `CONFIG_GENERIC_PINCTRL_GROUPS`). بيتخزن في `pinctrl_dev->pin_group_tree`.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `grp` | `struct pingroup` | الاسم + array الـ pins + عددهم |
| `data` | `void*` | بيانات خاصة بالـ driver لهذا الـ group |

---

#### `struct pinctrl_pin_desc`
**الغرض:** الوصف الثابت لكل pin — بيكتبه الـ driver في array ثابتة ويبعتها في `pinctrl_desc`.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `number` | `unsigned int` | رقم الـ pin الفريد في مساحة الـ pins |
| `name` | `const char*` | اسم الـ pin (e.g. `"PA0"`) |
| `drv_data` | `void*` | بيانات خاصة بالـ driver |

---

### 2. مخطط علاقات الـ Structs (ASCII)

```
                    ┌─────────────────────────────────────────────┐
                    │          pinctrldev_list (global)           │
                    └──────────────┬──────────────────────────────┘
                                   │  list_head node
                                   ▼
                    ┌──────────────────────────────────────────────┐
                    │            struct pinctrl_dev                │
                    │                                              │
                    │  desc ──────────────► struct pinctrl_desc   │
                    │                         pins[]               │
                    │                         pctlops ──► pinctrl_ops
                    │                         pmxops  ──► pinmux_ops
                    │                         confops ──► pinconf_ops
                    │                                              │
                    │  pin_desc_tree                               │
                    │   [radix: num → pin_desc*]                   │
                    │        └──► struct pin_desc                  │
                    │               pctldev ◄────────────────────┘│
                    │                                              │
                    │  pin_group_tree                              │
                    │   [radix: sel → group_desc*]                 │
                    │        └──► struct group_desc                │
                    │               grp.pins[]                     │
                    │                                              │
                    │  gpio_ranges ──► struct pinctrl_gpio_range  │
                    │                    base, npins, pins[]       │
                    │                    gc ──► struct gpio_chip   │
                    │                                              │
                    │  p ──────────────► struct pinctrl (hog)     │
                    │  hog_default ────► struct pinctrl_state      │
                    │  hog_sleep ──────► struct pinctrl_state      │
                    └──────────────────────────────────────────────┘


                    ┌─────────────────────────────────────────────┐
                    │           pinctrl_list (global)             │
                    └──────────────┬──────────────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────────────────────┐
                    │              struct pinctrl                  │
                    │                                              │
                    │  dev ───────────────► struct device         │
                    │  state ─────────────► struct pinctrl_state  │◄─┐
                    │  states ─────────────────────────────────┐  │  │
                    │  users (kref)                             │  │  │
                    └───────────────────────────────────────────┼──┘  │
                                                                │     │
                            ┌───────────────────────────────────┘     │
                            ▼                                          │
                    ┌──────────────────────────────────────────────┐  │
                    │           struct pinctrl_state               │  │
                    │                                              │  │
                    │  name = "default" / "sleep" / ...           │──┘
                    │  settings ───────────────────────────────┐  │
                    └──────────────────────────────────────────┼──┘
                                                               │
                            ┌──────────────────────────────────┘
                            ▼
                    ┌──────────────────────────────────────────────┐
                    │           struct pinctrl_setting             │
                    │                                              │
                    │  type = MUX_GROUP / CONFIGS_PIN / ...       │
                    │  pctldev ──────────► struct pinctrl_dev     │
                    │  data.mux.group    = group selector         │
                    │  data.mux.func     = function selector      │
                    │  (أو)                                        │
                    │  data.configs.group_or_pin                   │
                    │  data.configs.configs[]                      │
                    └──────────────────────────────────────────────┘


                    ┌─────────────────────────────────────────────┐
                    │           pinctrl_maps (global)             │
                    └──────────────┬──────────────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────────────────────┐
                    │           struct pinctrl_maps                │
                    │                                              │
                    │  maps ──────────────► struct pinctrl_map[]  │
                    │                         dev_name            │
                    │                         name (state)        │
                    │                         type                │
                    │                         ctrl_dev_name       │
                    │                         data.mux / configs  │
                    │  num_maps                                    │
                    └──────────────────────────────────────────────┘
```

---

### 3. مخطط دورة الحياة

#### أ) تسجيل الـ Pin Controller (Driver side)

```
Driver probe()
    │
    ├─► تعريف pinctrl_desc (ثابت: pins[], pctlops, pmxops, confops)
    │
    ├─► pinctrl_register_and_init(desc, dev, drv_data, &pctldev)
    │       │
    │       └─► pinctrl_init_controller()
    │               ├─ kzalloc(pinctrl_dev)
    │               ├─ INIT_RADIX_TREE(pin_desc_tree)
    │               ├─ INIT_RADIX_TREE(pin_group_tree)
    │               ├─ INIT_RADIX_TREE(pin_function_tree)
    │               ├─ pinctrl_check_ops()       ← يتأكد pctlops سليمة
    │               ├─ pinmux_check_ops()         ← لو pmxops موجودة
    │               ├─ pinconf_check_ops()         ← لو confops موجودة
    │               └─ pinctrl_register_pins()
    │                       └─ لكل pin: kzalloc(pin_desc)
    │                                   radix_tree_insert(pin_desc_tree, number, desc)
    │
    ├─► [Driver يضيف groups, functions, gpio_ranges]
    │
    └─► pinctrl_enable(pctldev)
            ├─ pinctrl_claim_hogs()
            │       ├─ create_pinctrl(pctldev->dev, pctldev)
            │       │       └─ يمشي في pinctrl_maps ويبني states وsettings
            │       ├─ pinctrl_lookup_state("default") → hog_default
            │       ├─ pinctrl_select_state(hog_default)
            │       └─ pinctrl_lookup_state("sleep")  → hog_sleep
            ├─ list_add_tail(pctldev, pinctrldev_list)
            └─ pinctrl_init_device_debugfs()


                [pctldev جاهز وظاهر في pinctrldev_list]
```

#### ب) استخدام الـ Consumer Device

```
Consumer Driver probe()
    │
    ├─► devm_pinctrl_get(dev)
    │       │
    │       └─► pinctrl_get(dev)
    │               ├─ find_pinctrl(dev)  ← لو موجود من قبل، يزود kref
    │               └─ create_pinctrl(dev, NULL)
    │                       ├─ kzalloc(pinctrl)
    │                       ├─ pinctrl_dt_to_map()    ← يقرأ DT وينشئ maps
    │                       ├─ for_each_pin_map → add_setting()
    │                       │       ├─ find_state / create_state
    │                       │       ├─ kzalloc(pinctrl_setting)
    │                       │       ├─ pinmux_map_to_setting()   [MUX]
    │                       │       │   └─ يحوّل أسماء → selectors
    │                       │       └─ pinconf_map_to_setting()  [CONFIG]
    │                       ├─ kref_init(&p->users)
    │                       └─ list_add_tail(p, pinctrl_list)
    │
    ├─► pinctrl_lookup_state(p, "default")  → state handle
    │
    ├─► pinctrl_select_state(p, state)
    │       └─► pinctrl_commit_state()
    │               ├─ [disable مux settings القديمة]
    │               ├─ pass 1: كل MUX_GROUP → pinmux_enable_setting()
    │               │               └─ ops->set_mux(pctldev, func, group)
    │               └─ pass 2: كل CONFIG → pinconf_apply_setting()
    │                               └─ ops->pin_config_set()
    │
    ├─► pinctrl_init_done(dev)
    │       └─ لو state هو init → يغير لـ default
    │
    └─► [عند unload] devm_pinctrl_put() → pinctrl_put() → kref_put()
                                            └─► pinctrl_release()
                                                    └─► pinctrl_free()
                                                            ├─ list_del states وsettings
                                                            ├─ pinctrl_dt_free_maps()
                                                            └─ kfree(p)
```

#### ج) إلغاء تسجيل الـ Controller

```
Driver remove()
    │
    └─► pinctrl_unregister(pctldev)
            ├─ pinctrl_remove_device_debugfs()
            ├─ pinctrl_put(pctldev->p)         ← يحرر الـ hog
            ├─ list_del(pctldev, pinctrldev_list)
            ├─ pinmux_generic_free_functions()
            ├─ pinctrl_generic_free_groups()
            ├─ pinctrl_free_pindescs()          ← يحرر كل pin_desc من radix tree
            ├─ list_del gpio_ranges
            ├─ mutex_destroy(&pctldev->mutex)
            └─ kfree(pctldev)
```

---

### 4. مخططات Call Flow

#### أ) `pinctrl_gpio_request()` — طلب GPIO pin

```
gpiolib: gpio_request(chip, offset)
    │
    └─► pinctrl_gpio_request(gc, offset)
            │
            ├─► pinctrl_get_device_gpio_range(gc, offset, &pctldev, &range)
            │       │
            │       └── يمشي في pinctrldev_list
            │               └── يمشي في pctldev->gpio_ranges
            │                       └── pinctrl_match_gpio_range()
            │                               [gc->base + offset in [range->base, base+npins)]?
            │
            ├─► gpio_to_pin(range, gc, offset)
            │       لو range->pins: يرجع range->pins[pin_idx]
            │       لو لا:          يرجع range->pin_base + (gc->base + offset - range->base)
            │
            └─► pinmux_request_gpio(pctldev, range, pin, gpio)
                    └─► ops->gpio_request_enable() or ops->request()
                            └─► يكتب في الـ register
```

#### ب) `pinctrl_select_state()` — تطبيق state على الـ hardware

```
consumer: pinctrl_select_state(p, new_state)
    │
    ├─ لو p->state == new_state: return 0  (مفيش تغيير)
    │
    └─► pinctrl_commit_state(p, new_state)
            │
            ├─ [old_state موجود؟]
            │       └─► pinctrl_cond_disable_mux_setting(old_state, NULL)
            │               └── لكل MUX setting في old_state:
            │                       pinmux_disable_setting(setting)
            │                           └─► ops->free() / ops->disable()
            │
            ├─ p->state = NULL
            │
            ├─ [Pass 1: MUX first]
            │   └── لكل setting في new_state:
            │           لو MUX_GROUP:
            │               pinmux_enable_setting(setting)
            │                   └─► ops->set_mux(pctldev, func_sel, group_sel)
            │                           └─► يكتب في الـ register
            │           لو CONFIG: skip (ret=0)
            │
            ├─ [Pass 2: CONFIG after]
            │   └── لكل setting في new_state:
            │           لو CONFIG_PIN أو CONFIG_GROUP:
            │               pinconf_apply_setting(setting)
            │                   └─► ops->pin_config_set() or group_config_set()
            │                           └─► يكتب في الـ register
            │           لو MUX: skip (ret=0)
            │
            ├─ p->state = new_state
            │
            └─ [لو error في أي خطوة]
                    unapply_new_state: → pinctrl_cond_disable_mux_setting(new_state, failing_setting)
                    restore_old_state: → pinctrl_select_state(p, old_state)  [recursive لكن آمن]
```

#### ج) تسجيل pin controller — `pinctrl_register()`

```
driver: pinctrl_register(desc, dev, drv_data)
    │
    ├─► pinctrl_init_controller(desc, dev, drv_data)
    │       ├─ validate desc != NULL && desc->name != NULL
    │       ├─ kzalloc(pinctrl_dev)
    │       ├─ init radix trees + lists + mutex
    │       ├─ pinctrl_check_ops()
    │       │       [get_groups_count و get_group_name مطلوبين]
    │       ├─ pinmux_check_ops()     [لو pmxops != NULL]
    │       ├─ pinconf_check_ops()    [لو confops != NULL]
    │       └─ pinctrl_register_pins()
    │               └── لكل pin في desc->pins[]:
    │                       pinctrl_register_one_pin()
    │                           ├─ check مفيش duplicate
    │                           ├─ kzalloc(pin_desc)
    │                           ├─ mutex_init(&pindesc->mux_lock)
    │                           ├─ copy name (أو kasprintf "PIN%u")
    │                           └─ radix_tree_insert(pin_desc_tree, num, pindesc)
    │
    └─► pinctrl_enable(pctldev)
            ├─ pinctrl_claim_hogs()
            │       └─ create_pinctrl(dev, pctldev)   [hog: ctrl_dev_name == dev_name]
            │               └─ for_each_pin_map: add_setting() لكل map للـ device نفسه
            │       └─ lookup + select "default" state
            ├─ list_add_tail(&pctldev->node, &pinctrldev_list)
            └─ pinctrl_init_device_debugfs()
```

#### د) `add_setting()` — تحويل map إلى setting

```
create_pinctrl()
    └─► add_setting(p, pctldev, map)
            │
            ├─ find_state(p, map->name) → لو مش موجود: create_state()
            │
            ├─ لو type == DUMMY_STATE: return 0
            │
            ├─ kzalloc(pinctrl_setting)
            │
            ├─ setting->pctldev = get_pinctrl_dev_from_devname(map->ctrl_dev_name)
            │       لو NULL: return -EPROBE_DEFER (الـ controller لسه ما جاش)
            │
            ├─ switch(map->type):
            │       MUX_GROUP   → pinmux_map_to_setting(map, setting)
            │                       └─ يحوّل أسماء groups وfunctions لـ selectors
            │       CONFIGS_PIN │
            │       CONFIGS_GROUP → pinconf_map_to_setting(map, setting)
            │                           └─ يحوّل اسم pin/group لـ ID + ينسخ configs array
            │
            └─ list_add_tail(&setting->node, &state->settings)
```

---

### 5. استراتيجية الـ Locking

الـ subsystem بيستخدم ٤ locks مختلفة، كل واحدة بتحمي data مختلفة:

#### جدول الـ Locks

| الـ Lock | النوع | يحمي | مين يمسكه |
|---------|-------|------|-----------|
| `pinctrldev_list_mutex` | `DEFINE_MUTEX` (static) | قائمة `pinctrldev_list` العالمية | أي كود بيقرأ/يكتب في القائمة |
| `pinctrl_list_mutex` | `DEFINE_MUTEX` (static) | قائمة `pinctrl_list` العالمية + `pinctrl_free()` | `find_pinctrl`, `create_pinctrl`, `pinctrl_free` |
| `pinctrl_maps_mutex` | `DEFINE_MUTEX` (global, extern) | قائمة `pinctrl_maps` العالمية | `pinctrl_register_mappings`, `create_pinctrl` |
| `pctldev->mutex` | `mutex` (per-device) | عمليات على controller واحد (gpio_ranges, pin ops) | كل API بيتعامل مع controller بعينه |

#### ترتيب الـ Lock Ordering (مهم لتجنب deadlock)

```
لو محتاج أكتر من lock في نفس الوقت، الترتيب الصحيح هو:

pinctrldev_list_mutex
    └─► pctldev->mutex

pinctrl_list_mutex
    (مستقل — مش بيتداخل مع التانيين في نفس الوقت)

pinctrl_maps_mutex
    (مستقل — مش بيتداخل)
```

مثال في `pinctrl_ready_for_gpio_range()`:
```c
mutex_lock(&pinctrldev_list_mutex);          // 1️⃣ الأول
    list_for_each_entry(pctldev, ...) {
        mutex_lock(&pctldev->mutex);          // 2️⃣ التاني
        ...
        mutex_unlock(&pctldev->mutex);
    }
mutex_unlock(&pinctrldev_list_mutex);
```

#### الـ `mux_lock` في `pin_desc`

- **mutex per-pin** — يحمي `mux_usecount`, `mux_owner`, `mux_setting` لكل pin على حدة.
- بيتمسك داخل `pinmux_request_gpio()` و`pinmux_free_gpio()` وعمليات الـ mux.
- بيتمسك **بعد** `pctldev->mutex` لو الاتنين محتاجين.

#### نقطة مهمة: `READ_ONCE` في `pinctrl_commit_state()`

```c
struct pinctrl_state *old_state = READ_ONCE(p->state);
```

**الـ** `p->state` ممكن يتعدّل من thread تاني أو من interrupt context لما الـ state بيتعمله `select`. الـ `READ_ONCE` بيضمن إن الـ compiler مش بيعمل optimization تقرأ القيمة أكتر من مرة.

#### غياب الـ Lock عند القراءة في بعض الأماكن

- `pinctrl_find_gpio_range_from_pin_nolock()` — نسخة بدون lock، الـ caller المسؤول عن الـ lock.
- الـ debugfs functions بتمسك `pctldev->mutex` بس، لأن هي read-only وعلى controller واحد.
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Registration & Controller Lifecycle

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `pinctrl_register` | `(desc, dev, data) → pctldev*` | تسجيل controller (legacy) |
| `pinctrl_register_and_init` | `(desc, dev, data, &pctldev) → int` | تسجيل بدون enable (الأحدث) |
| `pinctrl_enable` | `(pctldev) → int` | enable بعد init |
| `pinctrl_unregister` | `(pctldev)` | إلغاء تسجيل controller |
| `devm_pinctrl_register` | `(dev, desc, data) → pctldev*` | resource-managed register |
| `devm_pinctrl_register_and_init` | `(dev, desc, data, &pctldev) → int` | resource-managed register+init |

#### Pin Descriptor Management

| Function | الغرض |
|---|---|
| `pinctrl_register_pins` | تسجيل مصفوفة من pins |
| `pinctrl_register_one_pin` | تسجيل pin واحد في radix tree |
| `pinctrl_free_pindescs` | حذف pin descriptors |
| `pin_get_from_name` | بحث عن pin number من اسمه |
| `pin_get_name` | بحث عن اسم pin من رقمه |
| `pin_desc_get` | lookup مباشر في radix tree (inline) |

#### GPIO Range Management

| Function | الغرض |
|---|---|
| `pinctrl_add_gpio_range` | إضافة GPIO range لـ controller |
| `pinctrl_add_gpio_ranges` | إضافة مصفوفة ranges |
| `pinctrl_remove_gpio_range` | حذف GPIO range |
| `pinctrl_find_and_add_gpio_range` | بحث عن controller وإضافة range |
| `pinctrl_find_gpio_range_from_pin` | إيجاد range من pin number |
| `pinctrl_find_gpio_range_from_pin_nolock` | نفسه بدون lock |
| `gpio_to_pin` | تحويل GPIO offset → pin number |
| `pinctrl_match_gpio_range` | مطابقة GPIO مع range |

#### GPIO Operations (gpiolib interface)

| Function | الغرض |
|---|---|
| `pinctrl_gpio_can_use_line` | هل الـ GPIO line متاحة للاستخدام? |
| `pinctrl_gpio_request` | طلب pin كـ GPIO |
| `pinctrl_gpio_free` | تحرير pin من GPIO |
| `pinctrl_gpio_direction_input` | ضبط اتجاه GPIO كـ input |
| `pinctrl_gpio_direction_output` | ضبط اتجاه GPIO كـ output |
| `pinctrl_gpio_set_config` | ضبط pin config من GPIO driver |

#### Generic Group Management (CONFIG_GENERIC_PINCTRL_GROUPS)

| Function | الغرض |
|---|---|
| `pinctrl_generic_get_group_count` | عدد الـ groups المسجلة |
| `pinctrl_generic_get_group_name` | اسم group من selector |
| `pinctrl_generic_get_group_pins` | pins الخاصة بـ group |
| `pinctrl_generic_get_group` | إرجاع `group_desc` كامل |
| `pinctrl_generic_add_group` | إضافة group جديد |
| `pinctrl_generic_remove_group` | حذف group برقمه |
| `pinctrl_generic_free_groups` | حذف كل الـ groups |
| `pinctrl_get_group_selector` | إيجاد selector number من اسم group |
| `pinctrl_get_group_pins` | public API للحصول على pins |

#### Consumer Handle API

| Function | الغرض |
|---|---|
| `pinctrl_get` | الحصول على handle لـ device |
| `pinctrl_put` | تحرير handle (kref-based) |
| `pinctrl_lookup_state` | البحث عن state بالاسم |
| `pinctrl_select_state` | تفعيل state على الـ HW |
| `devm_pinctrl_get` | resource-managed pinctrl_get |
| `devm_pinctrl_put` | resource-managed pinctrl_put |

#### Mapping Table

| Function | الغرض |
|---|---|
| `pinctrl_register_mappings` | تسجيل جدول mappings |
| `pinctrl_unregister_mappings` | إلغاء تسجيل جدول mappings |
| `devm_pinctrl_register_mappings` | resource-managed register |

#### Power Management

| Function | الغرض |
|---|---|
| `pinctrl_force_sleep` | إجبار controller على sleep state |
| `pinctrl_force_default` | إجبار controller على default state |
| `pinctrl_init_done` | الإعلام عن انتهاء probe |
| `pinctrl_select_default_state` | تفعيل default state |
| `pinctrl_pm_select_default_state` | PM wrapper → default |
| `pinctrl_pm_select_sleep_state` | PM wrapper → sleep |
| `pinctrl_pm_select_idle_state` | PM wrapper → idle |
| `pinctrl_pm_select_init_state` | PM wrapper → init |

#### Helpers & Misc

| Function | الغرض |
|---|---|
| `pinctrl_provide_dummies` | تفعيل dummy state mode للـ platforms بدون driver |
| `pinctrl_dev_get_name` | إرجاع اسم الـ controller |
| `pinctrl_dev_get_devname` | إرجاع اسم الـ device |
| `pinctrl_dev_get_drvdata` | إرجاع driver_data |
| `get_pinctrl_dev_from_devname` | lookup pctldev من اسم device |
| `get_pinctrl_dev_from_of_node` | lookup pctldev من OF node |

---

### Group 1: Controller Registration & Lifecycle

هذه المجموعة هي نقطة دخول أي pin controller driver. وظيفتها إنشاء الـ `struct pinctrl_dev`، التحقق من صحة الـ ops، تسجيل الـ pins، وربط الـ hogs، ثم إضافة الـ controller للقائمة العالمية.

#### `pinctrl_init_controller`

```c
static struct pinctrl_dev *
pinctrl_init_controller(const struct pinctrl_desc *pctldesc,
                        struct device *dev, void *driver_data)
```

تُنشئ هذه الدالة الـ `struct pinctrl_dev` وتملأ حقوله الأساسية. تقوم بتهيئة الـ `radix_tree_root` للـ pin descriptors وللـ groups وللـ functions (حسب الـ Kconfig). ثم تتحقق من أن الـ ops المطلوبة موجودة وتسجل جميع الـ pins المذكورة في الـ descriptor.

- **`pctldesc`**: الـ descriptor الذي يقدمه الـ driver، يحتوي على الاسم ومصفوفة الـ pins والـ ops
- **`dev`**: الـ parent device
- **`driver_data`**: بيانات خاصة بالـ driver، تُخزن في `pctldev->driver_data`
- **Return**: `pctldev*` أو `ERR_PTR` عند الفشل
- **Key details**: لا تُضيف الـ controller للقائمة العالمية — ذلك يحدث في `pinctrl_enable()`. عند الفشل تُحرر الـ pindescs المسجلة جزئيًا ثم تفعل `kfree(pctldev)`.

---

#### `pinctrl_enable`

```c
int pinctrl_enable(struct pinctrl_dev *pctldev)
```

بعد `pinctrl_register_and_init()` ينتهي الـ driver من إعداد نفسه ثم يستدعي هذه الدالة. تقوم بثلاثة أشياء: استدعاء `pinctrl_claim_hogs()` لتفعيل الـ hog states، إضافة الـ `pctldev` لـ `pinctrldev_list` تحت الـ `pinctrldev_list_mutex`، ثم إنشاء الـ debugfs entries.

- **`pctldev`**: الـ controller المراد تفعيله
- **Return**: 0 عند النجاح، أو error من `pinctrl_claim_hogs()`
- **Locking**: يُمسك `pinctrldev_list_mutex` عند الإضافة للقائمة

---

#### `pinctrl_register`

```c
struct pinctrl_dev *pinctrl_register(const struct pinctrl_desc *pctldesc,
                                     struct device *dev, void *driver_data)
```

الـ legacy API. تجمع `pinctrl_init_controller()` + `pinctrl_enable()` في استدعاء واحد. المشكلة المعروفة بها: الـ driver لا يملك `pctldev` handle قبل استدعاء `dt_node_to_map()`، مما قد يسبب مشاكل إذا احتاج الـ driver للـ handle داخل الـ callback. لذا يُفضَّل استخدام `pinctrl_register_and_init()` بدلها.

- **Return**: `pctldev*` أو `ERR_PTR(-errno)` عند الفشل
- **Caller**: pin controller drivers في `probe()`

---

#### `pinctrl_register_and_init`

```c
int pinctrl_register_and_init(const struct pinctrl_desc *pctldesc,
                               struct device *dev, void *driver_data,
                               struct pinctrl_dev **pctldev)
```

الحل للمشكلة في `pinctrl_register()`. تضع الـ `pctldev` في `*pctldev` قبل أي callback، ثم يكمل الـ driver إعداداته، ويستدعي `pinctrl_enable()` عندما يكون جاهزًا.

- **`*pctldev`**: output parameter — يحصل عليه الـ driver قبل أي استدعاء لـ `pinctrl_enable()`
- **Return**: 0 أو `-ENOMEM` / `-EINVAL`
- **Important**: لا تُفعّل الـ hogs ولا تضيف للقائمة العالمية — هذا يتم في `pinctrl_enable()`

---

#### `pinctrl_unregister`

```c
void pinctrl_unregister(struct pinctrl_dev *pctldev)
```

عكس `pinctrl_register()`. تُحرر كل موارد الـ controller بالترتيب الصحيح: تحذف debugfs، تُحرر الـ `hog` pinctrl handle بـ `pinctrl_put()`، تُزيل الـ controller من `pinctrldev_list`، تحذف الـ functions والـ groups والـ pin descriptors، ثم تُزيل GPIO ranges وتُدمر الـ mutex.

- **Locking**: يأخذ `pinctrldev_list_mutex` و `pctldev->mutex` معًا — ترتيب ثابت لتجنب deadlock
- **Caller**: driver's `remove()` أو devm cleanup

---

#### `pinctrl_claim_hogs` (static)

```c
static int pinctrl_claim_hogs(struct pinctrl_dev *pctldev)
```

**الـ pin hogging** هو مفهوم مهم: الـ controller نفسه يطلب pins خاصة به (يكون consumer لنفسه). هذه الدالة تستدعي `create_pinctrl()` مع `pctldev->dev` كـ consumer وتبحث عن الـ `default` و `sleep` states في الـ mapping table، ثم تفعّل الـ default state مباشرة.

```
pinctrl_claim_hogs()
  └─ create_pinctrl(dev=pctldev->dev, pctldev=pctldev)
       └─ iterates pinctrl_maps for maps where dev_name == ctrl_dev_name
  └─ pinctrl_lookup_state(p, "default") → hog_default
  └─ pinctrl_select_state(p, hog_default)
  └─ pinctrl_lookup_state(p, "sleep") → hog_sleep
```

- إذا لم توجد أي hog maps، يُرجع -ENODEV ويُعامله كنجاح (no hogs found)
- `pctldev->p` يحتفظ بالـ handle الداخلي

---

#### `devm_pinctrl_register` و `devm_pinctrl_register_and_init`

```c
struct pinctrl_dev *devm_pinctrl_register(struct device *dev,
                                          const struct pinctrl_desc *pctldesc,
                                          void *driver_data)

int devm_pinctrl_register_and_init(struct device *dev,
                                   const struct pinctrl_desc *pctldesc,
                                   void *driver_data,
                                   struct pinctrl_dev **pctldev)
```

Wrappers حول النسخ العادية تستخدم `devm_add_action_or_reset()` لتسجيل `pinctrl_unregister()` كـ devm cleanup action. عند unbound الـ device يُنظف تلقائيًا.

---

### Group 2: Pin Descriptor Management

الـ pin descriptors هي السجل الداخلي لكل pin. تُخزن في `radix_tree_root` داخل الـ `pctldev`، وكل descriptor هو `struct pin_desc`.

#### `pinctrl_register_one_pin` (static)

```c
static int pinctrl_register_one_pin(struct pinctrl_dev *pctldev,
                                    const struct pinctrl_pin_desc *pin)
```

تُسجل pin واحد في الـ radix tree. تخصص `struct pin_desc`، تضبط صاحبها (`pctldev`)، تُهيئ `mux_lock` إذا كانت CONFIG_PINMUX مفعلة، وتُعالج حالة الـ pin بدون اسم بتوليد اسم ديناميكي "PIN%u" مع تعيين `dynamic_name = true` لتحريره لاحقًا.

- **Locking**: لا يأخذ lock — المُستدعي مسؤول (يُستدعى فقط من `pinctrl_register_pins()` أثناء init)
- **Error path**: إذا فشل `radix_tree_insert` يُحرر الـ pindesc والاسم الديناميكي

---

#### `pinctrl_register_pins` (static)

```c
static int pinctrl_register_pins(struct pinctrl_dev *pctldev,
                                 const struct pinctrl_pin_desc *pins,
                                 unsigned int num_descs)
```

Loop بسيط يستدعي `pinctrl_register_one_pin()` لكل pin. إذا فشل أي pin يتوقف فورًا ويُرجع الخطأ — الـ partial cleanup يتم من المستدعي عبر `pinctrl_free_pindescs()`.

---

#### `pinctrl_free_pindescs` (static)

```c
static void pinctrl_free_pindescs(struct pinctrl_dev *pctldev,
                                  const struct pinctrl_pin_desc *pins,
                                  unsigned int num_pins)
```

تحذف pin descriptors من الـ radix tree وتُحرر الذاكرة. تتحقق من `dynamic_name` لتحرير الاسم المُولَّد ديناميكيًا. تعمل على مصفوفة محدودة بـ `num_pins` وليس كل الـ tree.

---

#### `pin_get_from_name`

```c
int pin_get_from_name(struct pinctrl_dev *pctldev, const char *name)
```

تبحث خطيًا في مصفوفة `pctldev->desc->pins` وتقارن الاسم مع `desc->name` من الـ radix tree. تعيد رقم الـ pin أو `-EINVAL`. استخدام الـ `desc->pins` array للتكرار بدلًا من الـ radix tree مباشرة يعالج الـ sparse pin space.

- **Return**: pin number أو `-EINVAL`
- **Complexity**: O(n) حيث n عدد الـ pins

---

#### `pin_get_name`

```c
const char *pin_get_name(struct pinctrl_dev *pctldev, const unsigned int pin)
```

Lookup مباشر في الـ radix tree بـ O(log n)، يُرجع `desc->name`. يطبع `dev_err` إذا لم يجد الـ pin.

---

### Group 3: GPIO Range Management

ربط الـ GPIO subsystem بالـ pin controller يتم عبر `struct pinctrl_gpio_range`. كل range تعبّر عن نطاق من GPIO numbers مرتبط بنطاق من pin numbers.

```
GPIO space:  [base  ... base+npins-1]
                       ↕ mapping
Pin space:   [pin_base ... pin_base+npins-1]  (linear)
         OR  pins[]  (sparse/custom mapping)
```

#### `gpio_to_pin` (static inline)

```c
static inline int gpio_to_pin(struct pinctrl_gpio_range *range,
                               struct gpio_chip *gc, unsigned int offset)
```

تحوّل GPIO offset إلى pin number. إذا كانت `range->pins` موجودة تستخدمها مباشرة كـ lookup table، وإلا تحسب بالصيغة الخطية: `pin_base + (gc->base + offset - range->base)`.

---

#### `pinctrl_match_gpio_range` (static)

```c
static struct pinctrl_gpio_range *
pinctrl_match_gpio_range(struct pinctrl_dev *pctldev,
                          struct gpio_chip *gc, unsigned int offset)
```

تبحث في `pctldev->gpio_ranges` عن range تحتوي `gc->base + offset`. تأخذ `pctldev->mutex` أثناء البحث لحماية الـ list.

---

#### `pinctrl_get_device_gpio_range` (static)

```c
static int pinctrl_get_device_gpio_range(struct gpio_chip *gc,
                                          unsigned int offset,
                                          struct pinctrl_dev **outdev,
                                          struct pinctrl_gpio_range **outrange)
```

تبحث في **كل** الـ controllers في `pinctrldev_list` عن أي range تحتوي الـ GPIO المطلوب. هذه هي نقطة البحث العالمية لربط GPIO بـ pin controller.

- **Return**: 0 ونتائج في `*outdev` و `*outrange`، أو `-EPROBE_DEFER` إذا لم يجد (قد يكون الـ controller لم يُسجَّل بعد)
- **Locking**: يأخذ `pinctrldev_list_mutex` ثم `pctldev->mutex` (عبر `pinctrl_match_gpio_range`)

---

#### `pinctrl_ready_for_gpio_range` (static, CONFIG_GPIOLIB)

```c
static bool pinctrl_ready_for_gpio_range(struct gpio_chip *gc,
                                          unsigned int offset)
```

تُستخدم كـ complement لـ `pinctrl_get_device_gpio_range()`. إذا لم يجد الأخير range، تتحقق هذه إذا كان أي controller يملك range تتداخل مع الـ GPIO chip. إذا نعم → الـ controller جاهز وهذا الـ pin ليس له backend (ليس خطأ). إذا لا → الـ controller ربما لم يُسجَّل بعد.

---

#### `pinctrl_add_gpio_range`

```c
void pinctrl_add_gpio_range(struct pinctrl_dev *pctldev,
                             struct pinctrl_gpio_range *range)
```

**DEPRECATED في الـ DT world** — يُفضل استخدام OF GPIO binding مباشرة. تضيف range لـ `pctldev->gpio_ranges` تحت `pctldev->mutex`.

---

#### `pinctrl_find_and_add_gpio_range`

```c
struct pinctrl_dev *pinctrl_find_and_add_gpio_range(const char *devname,
                                                     struct pinctrl_gpio_range *range)
```

تجمع البحث والإضافة. إذا لم يجد الـ controller يُرجع `ERR_PTR(-EPROBE_DEFER)` — يُستخدم من GPIO drivers التي تحتاج ربط نفسها بـ pin controller.

---

#### `pinctrl_find_gpio_range_from_pin_nolock`

```c
struct pinctrl_gpio_range *
pinctrl_find_gpio_range_from_pin_nolock(struct pinctrl_dev *pctldev,
                                         unsigned int pin)
```

البحث العكسي: من pin number إلى GPIO range. تبحث في sparse pins أو linear ranges. **بدون lock** — يجب أن يكون المستدعي يحمل `pctldev->mutex`. تُستخدم من pinmux/pinconf داخليًا.

---

### Group 4: GPIO Operations (gpiolib Interface)

هذه الدوال هي الواجهة بين الـ gpiolib والـ pinctrl subsystem. تُستدعى فقط من GPIO drivers وليس مباشرة من platform code.

#### `pinctrl_gpio_request`

```c
int pinctrl_gpio_request(struct gpio_chip *gc, unsigned int offset)
```

يُستدعى من `gpiochip->request()`. يبحث عن الـ pin controller المسؤول عن هذا الـ GPIO، يحوّل الـ offset لـ pin number، ثم يستدعي `pinmux_request_gpio()` لتسجيل الملكية.

```
pinctrl_gpio_request(gc, offset)
  ├─ pinctrl_get_device_gpio_range() → pctldev, range
  │    └─ if -EPROBE_DEFER:
  │         pinctrl_ready_for_gpio_range() → true? return 0 (no backend, OK)
  ├─ mutex_lock(pctldev->mutex)
  ├─ gpio_to_pin(range, gc, offset) → pin
  ├─ pinmux_request_gpio(pctldev, range, pin, gpio_num)
  └─ mutex_unlock
```

- **Return**: 0 أو `-EINVAL` إذا كان الـ pin مأخوذًا بـ mux آخر

---

#### `pinctrl_gpio_free`

```c
void pinctrl_gpio_free(struct gpio_chip *gc, unsigned int offset)
```

عكس `pinctrl_gpio_request()`. يستدعي `pinmux_free_gpio()` لمسح `pin_desc->gpio_owner`.

---

#### `pinctrl_gpio_direction` (static)

```c
static int pinctrl_gpio_direction(struct gpio_chip *gc, unsigned int offset,
                                   bool input)
```

Helper مشترك بين `pinctrl_gpio_direction_input()` و `pinctrl_gpio_direction_output()`. يفوّض لـ `pinmux_gpio_direction()`.

---

#### `pinctrl_gpio_set_config`

```c
int pinctrl_gpio_set_config(struct gpio_chip *gc, unsigned int offset,
                             unsigned long config)
```

يُمكّن GPIO drivers من تعديل pin configuration (مثل debounce). يُحوّل الـ config لمصفوفة ويُمررها لـ `pinconf_set_config()`. مثال: driver يريد ضبط debounce timer.

```c
/* Example: GPIO driver sets 100ms debounce */
unsigned long cfg = PIN_CONF_PACKED(PIN_CONFIG_INPUT_DEBOUNCE, 100);
pinctrl_gpio_set_config(gc, offset, cfg);
```

---

#### `pinctrl_gpio_can_use_line`

```c
bool pinctrl_gpio_can_use_line(struct gpio_chip *gc, unsigned int offset)
```

تتحقق إذا كان الـ GPIO line غير محجوز بـ mux آخر. تستدعي `pinmux_can_be_used_for_gpio()`. إذا لم يجد controller (لا يوجد backend) يُرجع `true` افتراضيًا.

---

### Group 5: Generic Group Management

هذه المجموعة متاحة فقط مع `CONFIG_GENERIC_PINCTRL_GROUPS`. تُوفر تنفيذًا جاهزًا لـ `struct pinctrl_ops` بدلًا من أن يكتب كل driver تنفيذه الخاص، مع تخزين الـ groups في `radix_tree`.

#### `pinctrl_generic_add_group`

```c
int pinctrl_generic_add_group(struct pinctrl_dev *pctldev, const char *name,
                               const unsigned int *pins, int num_pins,
                               void *data)
```

تضيف group جديد للـ radix tree. تتحقق أولًا إذا كان الاسم موجودًا (idempotent — إذا موجود تُرجع selector الموجود). تستخدم `devm_kzalloc()` لأن الـ groups يجب أن تبقى طالما الـ device موجود.

- **`data`**: بيانات خاصة بالـ driver (مثل register offsets)
- **Return**: selector number (index في radix tree) أو error
- **Note**: **المستدعي مسؤول عن الـ locking** — الدالة لا تأخذ lock

```
pinctrl_generic_add_group()
  ├─ pinctrl_generic_group_name_to_selector() → check duplicate
  ├─ selector = pctldev->num_groups
  ├─ devm_kzalloc(group_desc)
  ├─ *group = PINCTRL_GROUP_DESC(name, pins, num_pins, data)
  ├─ radix_tree_insert(&pin_group_tree, selector, group)
  └─ num_groups++
```

---

#### `pinctrl_generic_remove_group`

```c
int pinctrl_generic_remove_group(struct pinctrl_dev *pctldev,
                                  unsigned int selector)
```

تحذف group من الـ radix tree وتُحرر الذاكرة بـ `devm_kfree()`. تُقلل `num_groups`.

- **Note**: الـ selector numbers لا تُعاد استخدامها — sparse tree

---

#### `pinctrl_generic_free_groups` (static)

```c
static void pinctrl_generic_free_groups(struct pinctrl_dev *pctldev)
```

تحذف **كل** الـ entries من الـ radix tree بـ `radix_tree_for_each_slot`. لا تُحرر الـ group_desc ذاتها لأنها مُخصصة بـ `devm_kzalloc()` وستُحرر تلقائيًا عند unregister الـ device.

---

#### `pinctrl_get_group_selector`

```c
int pinctrl_get_group_selector(struct pinctrl_dev *pctldev,
                                const char *pin_group)
```

تبحث خطيًا عبر الـ `pctlops->get_group_name()` callback عن group بالاسم وتُرجع رقمه (selector). هذه دالة public تستخدمها الـ pinmux و pinconf داخليًا.

- **Return**: selector ≥ 0 أو `-EINVAL`

---

### Group 6: Consumer Handle API

هذه هي الـ API الأساسية التي يستخدمها أي driver (consumer) لطلب والتبديل بين الـ pin states.

#### `create_pinctrl` (static)

```c
static struct pinctrl *create_pinctrl(struct device *dev,
                                       struct pinctrl_dev *pctldev)
```

الدالة الجوهرية للـ consumer side. تُنشئ `struct pinctrl` وتملأه بالـ states من مصدرين: الـ Device Tree (عبر `pinctrl_dt_to_map()`) والـ static mapping table. ثم تضيف الـ handle للقائمة العالمية `pinctrl_list`.

```
create_pinctrl(dev, pctldev)
  ├─ kzalloc(struct pinctrl)
  ├─ pinctrl_dt_to_map(p, pctldev)    ← parse DT pinctrl nodes
  ├─ mutex_lock(pinctrl_maps_mutex)
  ├─ for_each_pin_map(maps_node, map):
  │    if map->dev_name != dev_name: skip
  │    if pctldev && map->ctrl_dev_name != pctldev->dev: skip (hog filter)
  │    add_setting(p, pctldev, map)
  │      ├─ find_state() or create_state()
  │      └─ kzalloc(pinctrl_setting) + pinmux_map_to_setting() / pinconf_map_to_setting()
  ├─ mutex_unlock(pinctrl_maps_mutex)
  ├─ kref_init(&p->users)
  └─ list_add_tail → pinctrl_list
```

- **`pctldev` غير NULL**: معناها نحن نبني hog (الـ controller يطلب pins لنفسه)، وفي هذه الحالة نتجاهل الـ maps الخاصة بـ controllers أخرى
- **EPROBE_DEFER handling**: إذا وجد map لكن الـ controller غير موجود بعد → defer فوري

---

#### `pinctrl_get`

```c
struct pinctrl *pinctrl_get(struct device *dev)
```

الـ public API للـ consumers. تبحث أولًا إذا كان handle موجودًا لهذا الـ device (عبر `find_pinctrl()`)، وإذا نعم تزيد الـ `kref` وتُرجع نفس الـ handle — لا تُنشئ نسخة جديدة. إذا لا، تستدعي `create_pinctrl(dev, NULL)`.

- **Return**: `struct pinctrl*` أو `ERR_PTR`
- **Refcounting**: `kref` يسمح لأكثر من مستدعٍ بالحصول على نفس الـ handle

---

#### `pinctrl_put`

```c
void pinctrl_put(struct pinctrl *p)
```

تُقلل الـ kref. إذا وصل لصفر تستدعي `pinctrl_release()` → `pinctrl_free()` التي تُحرر كل states وكل settings وتحذف الـ DT maps وتُزيل الـ handle من `pinctrl_list`.

---

#### `pinctrl_lookup_state`

```c
struct pinctrl_state *pinctrl_lookup_state(struct pinctrl *p,
                                            const char *name)
```

تبحث في `p->states` عن state بالاسم. إذا لم تجد وكان `pinctrl_dummy_state == true`، تُنشئ dummy state فارغة بدلًا من الفشل — مفيد للـ platforms التي تشغّل shared drivers بدون pin controller.

---

#### `pinctrl_select_state`

```c
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state)
```

الـ entry point لتفعيل state. تتحقق أولًا إذا كانت الـ state المطلوبة هي نفس الحالية (no-op). ثم تستدعي `pinctrl_commit_state()`.

---

#### `pinctrl_commit_state` (static)

```c
static int pinctrl_commit_state(struct pinctrl *p, struct pinctrl_state *state)
```

الدالة الأكثر تعقيدًا في الـ consumer side. تُطبق الـ state الجديدة على الـ hardware بطريقة atomic قدر الإمكان مع rollback.

```
pinctrl_commit_state(p, new_state)
  ├─ old_state = p->state
  ├─ if old_state: pinctrl_cond_disable_mux_setting(old_state, NULL)
  │     → disable all mux settings in old state
  ├─ p->state = NULL   ← atomic clear before applying
  │
  ├─ Pass 1 — MUX settings:
  │    for each setting in new_state:
  │      if MUX_GROUP: pinmux_enable_setting(setting)
  │      if FAIL: goto unapply_new_state
  │      pinctrl_link_add() ← device link consumer→controller
  │
  ├─ Pass 2 — CONFIG settings:
  │    for each setting in new_state:
  │      if CONFIGS_*: pinconf_apply_setting(setting)
  │      if FAIL: goto unapply_mux_setting
  │
  ├─ p->state = state  ← success
  │
  ├─ unapply_mux_setting:
  │    pinctrl_cond_disable_mux_setting(state, NULL)
  │    → restore_old_state
  │
  └─ unapply_new_state / restore_old_state:
       if old_state: pinctrl_select_state(p, old_state)  ← recursive, but p->state is NULL so no loop
```

**نقاط مهمة:**
- الـ MUX settings تُطبق أولًا ثم الـ CONFIG settings — هذا الترتيب مقصود
- `p->state = NULL` قبل التطبيق يمنع loop لا نهائي في حالة rollback
- الـ `device_link_add()` يربط الـ consumer بالـ controller لضمان صحة suspend/resume order
- لا يمكن "unmux" pin بشكل كامل، لكن يمكن `pinmux_disable_setting` لمسح الملكية في SW

---

#### `pinctrl_free` (static)

```c
static void pinctrl_free(struct pinctrl *p, bool inlist)
```

تُحرر كل resources الـ handle. تمر على كل states وكل settings، تستدعي `pinctrl_free_setting()` على كل منها (الذي يُعطّل أي mux نشط إذا كانت `state == p->state`). ثم تُحرر الـ DT maps، وتُزيل من القائمة إذا `inlist == true`.

- **Locking**: يأخذ `pinctrl_list_mutex` لأنه قد يُعدّل `pinctrl_list`

---

#### `devm_pinctrl_get` و `devm_pinctrl_put`

```c
struct pinctrl *devm_pinctrl_get(struct device *dev)
void devm_pinctrl_put(struct pinctrl *p)
```

**الـ`devm_pinctrl_get`** تُضيف `devm_pinctrl_release` كـ cleanup action بعد `pinctrl_get()`. **الـ`devm_pinctrl_put`** تُلغي الـ action مبكرًا بـ `devm_release_action()`. مناسبة لـ drivers التي تريد RAII-style resource management.

---

### Group 7: Mapping Table Management

الـ mapping table هي الرابط بين devices وpin controllers وstates. تُعرَّف statically (أو من DT) وتُسجَّل مرة واحدة.

#### `pinctrl_register_mappings`

```c
int pinctrl_register_mappings(const struct pinctrl_map *maps,
                               unsigned int num_maps)
```

تُسجل مصفوفة mappings. تتحقق أولًا من صحة كل entry: وجود `dev_name` واسم للـ state، وجود `ctrl_dev_name` للأنواع غير dummy، وصحة البيانات الخاصة بكل نوع (عبر `pinmux_validate_map()` و `pinconf_validate_map()`). ثم تُنشئ `struct pinctrl_maps` وتُضيفها لـ `pinctrl_maps` list.

- **Important**: الـ maps array يجب أن **لا تكون** `__initdata` لأن الـ core يحتفظ بـ pointer إليها
- **Locking**: يأخذ `pinctrl_maps_mutex` عند الإضافة

---

#### `pinctrl_unregister_mappings`

```c
void pinctrl_unregister_mappings(const struct pinctrl_map *map)
```

تبحث في `pinctrl_maps` عن الـ node الذي يحتوي هذا الـ map pointer وتحذفه. البحث بالـ pointer مباشرة (pointer equality) وليس بالقيمة.

---

#### `devm_pinctrl_register_mappings`

```c
int devm_pinctrl_register_mappings(struct device *dev,
                                    const struct pinctrl_map *maps,
                                    unsigned int num_maps)
```

Resource-managed version. مفيد لـ drivers التي تُنشئ mappings ديناميكيًا في `probe()`.

---

### Group 8: Power Management

#### `pinctrl_force_sleep` و `pinctrl_force_default`

```c
int pinctrl_force_sleep(struct pinctrl_dev *pctldev)
int pinctrl_force_default(struct pinctrl_dev *pctldev)
```

يُستدعيان لإجبار الـ controller على state محدد من خارج الـ consumer API العادي. يعملان مباشرة على `pctldev->p` (الـ hog handle) و `pctldev->hog_sleep` / `pctldev->hog_default`.

- **Use case**: power management drivers تريد وضع كل الـ pins في safe state قبل suspend
- تتحقق من `IS_ERR` للـ handle والـ state قبل الاستدعاء

---

#### `pinctrl_init_done`

```c
int pinctrl_init_done(struct device *dev)
```

تُستدعى بعد انتهاء `probe()`. تفحص إذا كانت الـ device لا تزال في `init` state، وإذا نعم تُبدّلها للـ `default` state. السبب: بعض devices تحتاج configuration خاصة أثناء الـ init (مثل pull-ups مختلفة) ثم configuration أخرى في الـ runtime.

- تُهمل بصمت إذا لم يكن للـ device `dev->pins`
- تُهمل إذا لم يكن للـ device `init_state` أو `default_state`
- تُهمل إذا كان الـ device لا يزال ليس في `init_state`

---

#### `pinctrl_pm_select_*_state`

```c
int pinctrl_pm_select_default_state(struct device *dev)
int pinctrl_pm_select_sleep_state(struct device *dev)
int pinctrl_pm_select_idle_state(struct device *dev)
int pinctrl_pm_select_init_state(struct device *dev)
```

أربع wrappers حول `pinctrl_select_bound_state()`. مُعرّفة تحت `CONFIG_PM`. كلها تتحقق من `dev->pins` أولًا وتُرجع 0 بصمت إذا لم يكن للـ device pin info.

**الـ `pinctrl_select_bound_state` (static):**
```c
static int pinctrl_select_bound_state(struct device *dev,
                                       struct pinctrl_state *state)
```
تتحقق من `IS_ERR(state)` (لو لم تُعثر على الـ state أثناء init، ستكون ERR_PTR) وتُرجع 0 بصمت — هذا تصميم مقصود لتجنب فشل الـ PM operations بسبب غياب optional states.

---

### Group 9: Accessors & Lookup Helpers

#### `pinctrl_provide_dummies`

```c
void pinctrl_provide_dummies(void)
```

تُعين `pinctrl_dummy_state = true`. يُستدعى من platform init على أنظمة تُشغّل drivers تستخدم pinctrl API لكن بدون أي pin controller فعلي. بعد استدعائها، `pinctrl_lookup_state()` لن تفشل بـ `-ENODEV` بل ستُنشئ dummy states فارغة.

---

#### `pinctrl_dev_get_name` / `pinctrl_dev_get_devname` / `pinctrl_dev_get_drvdata`

```c
const char *pinctrl_dev_get_name(struct pinctrl_dev *pctldev)
const char *pinctrl_dev_get_devname(struct pinctrl_dev *pctldev)
void *pinctrl_dev_get_drvdata(struct pinctrl_dev *pctldev)
```

Accessors بسيطة مُعرَّفة كـ `EXPORT_SYMBOL_GPL`. `get_name` تُرجع `pctldev->desc->name` (اسم الـ controller). `get_devname` تُرجع `dev_name(pctldev->dev)` (اسم الـ device في الـ bus). `get_drvdata` تُرجع `pctldev->driver_data`.

---

#### `get_pinctrl_dev_from_devname`

```c
struct pinctrl_dev *get_pinctrl_dev_from_devname(const char *devname)
```

Lookup بالاسم في `pinctrldev_list`. تُستخدم من `add_setting()` و `pinctrl_find_and_add_gpio_range()` لإيجاد controller بناءً على اسم device من الـ mapping table.

- **Locking**: يأخذ ويُحرر `pinctrldev_list_mutex`
- **Return**: `pctldev*` أو NULL

---

#### `get_pinctrl_dev_from_of_node`

```c
struct pinctrl_dev *get_pinctrl_dev_from_of_node(struct device_node *np)
```

مثل السابقة لكن تستخدم `device_match_of_node()` للمطابقة. تُستخدم من الـ DT parsing code.

---

### Group 10: DebugFS

كل دوال الـ debugfs مشروطة بـ `CONFIG_DEBUG_FS`.

#### `pinctrl_init_debugfs` (static)

```c
static void pinctrl_init_debugfs(void)
```

تُنشئ `/sys/kernel/debug/pinctrl/` ثلاثة ملفات:
- `pinctrl-devices`: قائمة بكل الـ controllers وإذا كانوا يدعمون pinmux وpinconf
- `pinctrl-maps`: كل الـ mapping table entries
- `pinctrl-handles`: كل الـ consumer handles وstates وsettings النشطة

---

#### `pinctrl_init_device_debugfs` (static)

```c
static void pinctrl_init_device_debugfs(struct pinctrl_dev *pctldev)
```

تُنشئ directory خاصة بكل controller تحت `/sys/kernel/debug/pinctrl/`. اسم الـ directory = `devname-ctrlname` إذا كانا مختلفين، وإلا `devname` فقط. تُنشئ:
- `pins`: كل الـ pins مع أسمائها وارتباطها بـ GPIO إذا وجد
- `pingroups`: كل الـ groups ومحتوياتها من pins
- `gpio-ranges`: الـ GPIO ranges المسجلة
- ملفات إضافية من pinmux وpinconf إذا كانا مدعومَين

---

### Group 11: Subsystem Init

#### `pinctrl_init`

```c
static int __init pinctrl_init(void)
```

دالة الـ init الخاصة بالـ subsystem كله. تُشغَّل بـ `core_initcall` (مرحلة مبكرة جدًا من الـ boot) لأن الكثير من الـ drivers تحتاج pinctrl أثناء تسجيلها.

```c
core_initcall(pinctrl_init);
```

- تفعل فقط `pinctrl_init_debugfs()` — الـ subsystem نفسه لا يحتاج تهيئة معقدة
- الـ lists والـ mutexes مُهيأة statically بـ `LIST_HEAD` و `DEFINE_MUTEX`

---

### ملاحظات على الـ Locking Model

```
pinctrldev_list_mutex  → يحمي pinctrldev_list (global list of controllers)
pinctrl_list_mutex     → يحمي pinctrl_list (global list of consumer handles)
pinctrl_maps_mutex     → يحمي pinctrl_maps (global mapping table)
pctldev->mutex         → يحمي per-controller data (gpio_ranges, pin states)
pin_desc->mux_lock     → يحمي per-pin mux ownership (CONFIG_PINMUX)
```

**ترتيب الـ locking (لتجنب deadlock):**

```
pinctrldev_list_mutex
  └─ pctldev->mutex
       └─ (لا lock داخلي آخر)

pinctrl_list_mutex
  └─ (لا lock داخلي)

pinctrl_maps_mutex
  └─ pctldev lookup (بدون lock على pctldev أثناء البحث)
```

في `pinctrl_unregister()` يُمسك `pinctrldev_list_mutex` ثم `pctldev->mutex` بهذا الترتيب دائمًا — يجب احترام هذا الترتيب في أي كود يأخذ كلا الـ locks.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — الـ Entries والقراءة

الـ pinctrl core بيعمل directory كامل في debugfs عند `pinctrl_init_debugfs()` وكمان directory خاص لكل controller عند `pinctrl_init_device_debugfs()`.

**الـ global entries تحت `/sys/kernel/debug/pinctrl/`:**

| Entry | Handler | الوصف |
|---|---|---|
| `pinctrl-devices` | `pinctrl_devices_show` | اسم كل controller + هل بيدعم pinmux وpinconf |
| `pinctrl-maps` | `pinctrl_maps_show` | كل الـ mapping table entries المسجلة |
| `pinctrl-handles` | `pinctrl_show` | كل الـ consumers وstate الحالية لكل device |

**الـ per-controller entries تحت `/sys/kernel/debug/pinctrl/<controller-name>/`:**

| Entry | Handler | الوصف |
|---|---|---|
| `pins` | `pinctrl_pins_show` | كل الـ pins المسجلة مع أسمائها وربطها بـ GPIO |
| `pingroups` | `pinctrl_groups_show` | كل الـ groups والـ pins اللي فيهم |
| `gpio-ranges` | `pinctrl_gpioranges_show` | الـ GPIO ranges المربوطة بالـ controller |
| `pinmux-pins` | من `pinmux.c` | الـ pins اللي اتمسكت من مين |
| `pinmux-functions` | من `pinmux.c` | الـ functions المتاحة |
| `pinconf-pins` | من `pinconf.c` | config الـ pins |
| `pinconf-groups` | من `pinconf.c` | config الـ groups |

```bash
# اقرأ كل الـ controllers الموجودة
cat /sys/kernel/debug/pinctrl/pinctrl-devices

# شوف كل الـ maps المسجلة
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# شوف state كل device حالياً
cat /sys/kernel/debug/pinctrl/pinctrl-handles

# افرض اسم الـ controller هو "fe200000.gpio" — اقرأ كل الـ pins
cat /sys/kernel/debug/pinctrl/fe200000.gpio/pins

# شوف الـ groups
cat /sys/kernel/debug/pinctrl/fe200000.gpio/pingroups

# شوف الـ GPIO ranges
cat /sys/kernel/debug/pinctrl/fe200000.gpio/gpio-ranges

# شوف مين ماسك الـ pins (مهمة جداً لحل تعارض الـ mux)
cat /sys/kernel/debug/pinctrl/fe200000.gpio/pinmux-pins
```

**مثال output من `pins`:**
```
registered pins: 54
pin 0 (GPIO0) 0:gpiochip0
pin 1 (GPIO1) 1:gpiochip0
pin 2 (GPIO2) 0:?
...
```
الـ `0:?` معناها الـ pin مش مربوط بأي GPIO chip حالياً.

**مثال output من `pinmux-pins`:**
```
pin 4 (GPIO4): UNCLAIMED
pin 5 (GPIO5): device fe300000.i2c function i2c group i2c1-pins
pin 6 (GPIO6): GPIO UNCLAIMED
```

---

#### 2. sysfs Entries

الـ pinctrl مش عنده sysfs interface خاص بيه بشكل مباشر، لكن الـ consumer devices بيظهروا فيه:

```bash
# شوف pinctrl info لأي device (لو الـ kernel support devinfo)
ls /sys/bus/platform/devices/<device>/
# هتلاقي: driver, uevent, etc.

# الـ device_link اللي بيتعمل لما pinctrl.link_consumers = true
ls /sys/bus/platform/devices/<consumer>/consumer:<pinctrl-dev>/

# شوف كل devices اللي بيستخدموا pinctrl عبر device links
ls /sys/kernel/debug/devices_deferred   # الـ deferred probe بسبب pinctrl
```

---

#### 3. ftrace — Tracepoints والـ Events

```bash
# شوف الـ available events المتعلقة بـ pinctrl
ls /sys/kernel/debug/tracing/events/ | grep pin

# enable كل pinctrl events
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# أو بشكل انتقائي — لو الـ tracepoints موجودة
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_select_state/enable

# enable الـ function tracing لأهم functions في core.c
echo pinctrl_select_state >> /sys/kernel/debug/tracing/set_ftrace_filter
echo pinctrl_commit_state >> /sys/kernel/debug/tracing/set_ftrace_filter
echo pinctrl_get >> /sys/kernel/debug/tracing/set_ftrace_filter
echo add_setting >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'function' > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل الـ operation اللي عايز تتبعها، بعدين
cat /sys/kernel/debug/tracing/trace

# disable
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

**trace مفيد لتتبع transition بين الـ states:**
```bash
echo pinctrl_commit_state >> /sys/kernel/debug/tracing/set_ftrace_filter
echo pinmux_enable_setting >> /sys/kernel/debug/tracing/set_ftrace_filter
echo pinconf_apply_setting >> /sys/kernel/debug/tracing/set_ftrace_filter
```

---

#### 4. printk و Dynamic Debug

الـ `core.c` بيستخدم `pr_fmt` prefix هو `"pinctrl core: "` — يعني كل `pr_debug` بيطلع بيه.

```bash
# enable dynamic debug لكل drivers/pinctrl/core.c
echo 'file drivers/pinctrl/core.c +p' > /sys/kernel/debug/dynamic_debug/control

# enable لكل الـ pinctrl subsystem
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control

# enable كل الـ pinctrl files
echo 'file drivers/pinctrl/* +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug rules الحالية
cat /sys/kernel/debug/dynamic_debug/control | grep pinctrl

# disable
echo 'file drivers/pinctrl/core.c -p' > /sys/kernel/debug/dynamic_debug/control
```

**أو في kernel cmdline:**
```
dyndbg="file drivers/pinctrl/core.c +p"
```

**الـ pr_debug المهمة في core.c:**

```c
pr_debug("registered pin %d (%s) on %s\n", ...);          // عند تسجيل كل pin
pr_debug("add %u pinctrl maps\n", num_maps);               // عند إضافة maps
dev_dbg(dev, "try to register %d pins ...\n", ...);        // قبل تسجيل الـ pins
dev_dbg(pctldev->dev, "no hogs found\n");                  // لما مفيش pin-hog
dev_dbg(pdev, "obtain a copy of previously claimed ...");  // pinctrl_get() مكررة
```

لو عايز ترفع مستوى الـ verbosity أكتر، في kernel cmdline:
```
loglevel=8
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوصف |
|---|---|
| `CONFIG_DEBUG_FS` | **أساسي** — لازم يكون enabled عشان أي debugfs entry تشتغل |
| `CONFIG_PINCTRL` | تفعيل الـ subsystem أصلاً |
| `CONFIG_PINMUX` | تفعيل pinmux support |
| `CONFIG_PINCONF` | تفعيل pin configuration support |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | تفعيل الـ generic group management (radix tree) |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | تفعيل الـ generic function management |
| `CONFIG_GENERIC_PINCONF` | تفعيل الـ generic pinconf parameters |
| `CONFIG_DEBUG_PINCTRL` | (لو موجود في config الـ board) extra verbosity |
| `CONFIG_GPIOLIB` | لربط الـ GPIO ranges بالـ pins في debugfs |
| `CONFIG_OF` | لـ DT-based pin mapping عبر `pinctrl_dt_to_map()` |
| `CONFIG_DYNAMIC_DEBUG` | لتفعيل `dev_dbg`/`pr_debug` runtime |
| `CONFIG_PROVE_LOCKING` | لكشف deadlock في الـ mutexes (pinctrl_list_mutex, إلخ) |
| `CONFIG_LOCKDEP` | تتبع lock ordering violations |
| `CONFIG_DEBUG_OBJECTS` | كشف use-after-free في الـ objects |

```bash
# تحقق من الـ config في runtime
zcat /proc/config.gz | grep -E "CONFIG_(PINCTRL|PINMUX|PINCONF|DEBUG_FS|GENERIC_PIN)"
```

---

#### 6. أدوات خاصة بالـ Subsystem

مفيش devlink هنا، لكن في أدوات مخصصة:

```bash
# pinctrl userspace tool (من pinctrl-utils أو بعض distros)
# على بعض الأنظمة يكون اسمه pinctrl
which pinctrl 2>/dev/null

# على Raspberry Pi مثلاً:
raspi-gpio get 17         # شوف state الـ GPIO/pin
raspi-gpio funcs          # شوف الـ alt functions

# على systemd-based systems
# بعض الـ DTs بيكشفوا pinctrl info عبر
udevadm info /sys/bus/platform/devices/<pinctrl-dev>

# لفحص الـ device tree الـ compiled
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "pinctrl"

# أو
cat /sys/firmware/devicetree/base/*/pinctrl*/compatible 2>/dev/null
```

---

#### 7. جدول Error Messages الشائعة

| رسالة الـ kernel log | السبب | الحل |
|---|---|---|
| `pinctrl core: pin N already registered` | نفس الـ pin number اتسجل مرتين في `pinctrl_register_one_pin()` | افحص `npins` والـ `pins[]` array في الـ driver — فيه تكرار في الأرقام |
| `pinctrl ops lacks necessary functions` | الـ `pinctrl_ops` مش فيها `get_groups_count` أو `get_group_name` | اكمّل تعريف `pinctrl_ops` في الـ driver |
| `does not have pin group <name>` | الـ state بيطلب group مش موجودة | افحص الـ DT أو الـ mapping table — اسم الـ group غلط |
| `error during pin registration` | `radix_tree_insert()` فشل أو الـ pin number > max | افحص الـ pin numbers في الـ descriptor |
| `unknown pinctrl device <name> in map entry, deferring probe` | الـ pin controller اللي الـ map بتشير إليه لسه ما probe-ش | ده normal مع `-EPROBE_DEFER`، لكن لو بيتكرر = مشكلة في probe order |
| `error claiming hogs: <errno>` | `pinctrl_claim_hogs()` فشل — الـ controller نفسه مش قادر يحجز pins الحالية | افحص الـ DT hog entries في node الـ pinctrl controller نفسه |
| `failed to select default state` | `pinctrl_select_state()` فشل على الـ default state | افحص إن الـ mux settings مش متعارضة مع pin آخر |
| `Error applying setting, reverse things back` | `pinmux_enable_setting()` فشل أثناء `pinctrl_commit_state()` | الـ pin محجوز من device تاني، أو الـ hardware مش بيقبل الـ function |
| `failed to register map <name> (N): no device given` | الـ map entry مش فيها `dev_name` | صحح الـ `pinctrl_map` table |
| `failed to create debugfs directory for <dev>` | debugfs مش mounted أو مشكلة في memory | mount debugfs أو افحص الـ dmesg للـ OOM errors |
| `pinctrl core: failed to determine debugfs dir name` | `devm_kasprintf()` فشل | مشكلة memory عند init |
| `failed to activate default pinctrl state` | `pinctrl_init_done()` — الـ transition من init لـ default state فشل | افحص الـ pinctrl-0 و pinctrl-names في DT |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

الأماكن الأنسب لإضافة debug assertions:

```c
/* في pinctrl_commit_state() — لو الـ state switch فشل */
if (ret < 0) {
    dev_err(p->dev, "Error applying setting, reverse things back\n");
    /* هنا تقدر تضيف: */
    dump_stack();   /* لو عايز تعرف مين استدعى select_state */
}

/* في add_setting() — لو الـ pctldev مش موجود */
if (!setting->pctldev) {
    /* WARN_ON مناسب هنا لأنه حالة غير متوقعة */
    WARN_ON(!pinctrl_dummy_state);
}

/* في pinctrl_register_one_pin() — double registration */
if (pindesc) {
    dev_err(pctldev->dev, "pin %d already registered\n", pin->number);
    WARN_ON(1);  /* يكشف الـ caller stack */
}

/* في pinctrl_get() */
if (WARN_ON(!dev))   /* موجود أصلاً في الكود */
    return ERR_PTR(-EINVAL);

/* في gpio_to_pin() — للتحقق من صحة الـ range */
WARN_ON(gc->base + offset < range->base);
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State موافق الـ Kernel State

بعد تطبيق أي pinctrl state، لازم تتحقق من:

```bash
# 1. شوف الـ kernel يعتقد إيه
cat /sys/kernel/debug/pinctrl/<controller>/pinmux-pins
# مثال output:
# pin 5 (SDA1): device fe804000.i2c function i2c1 group i2c1_pins

# 2. قارن بـ register الـ hardware الفعلي
# على Raspberry Pi (BCM2835) — GPFSEL registers
# لكل 10 pins في register واحد، 3 bits لكل pin
# GPFSEL0 @ 0xFE200000 — pins 0-9
# bit 15-18 = pin 5 function (3 bits)
devmem2 0xfe200000 w   # اقرأ GPFSEL0

# فسر النتيجة: ALT0=0b100, ALT1=0b101, INPUT=0b000, OUTPUT=0b001
```

---

#### 2. Register Dump Techniques

```bash
# devmem2 — الأبسط (لازم يتنصب أو يتبنى)
devmem2 <phys_addr> w     # read 32-bit word
devmem2 <phys_addr> w <value>  # write

# /dev/mem مباشرة (يحتاج CONFIG_DEVMEM=y و root)
dd if=/dev/mem bs=4 count=1 skip=$((0xfe200000/4)) 2>/dev/null | xxd

# io utility (من iotools package)
io -4 -r 0xfe200000      # read 4 bytes

# من داخل kernel module — الأسلم:
void __iomem *base = ioremap(PINCTRL_BASE, SZ_4K);
pr_info("GPFSEL0 = 0x%08x\n", readl(base + GPFSEL0_OFFSET));
iounmap(base);

# dump كامل للـ pinctrl registers عبر debugfs (لو الـ driver دعم pin_dbg_show)
cat /sys/kernel/debug/pinctrl/<controller>/pins
```

**مثال — فحص BCM2835 GPFSEL للـ pins 0-9:**
```bash
GPFSEL0=$(devmem2 0xfe200000 w | grep "Value at" | awk '{print $NF}')
echo "GPFSEL0 = $GPFSEL0"
# كل 3 bits تمثل pin function:
# 000=INPUT, 001=OUTPUT, 100=ALT0, 101=ALT1, 110=ALT2, 111=ALT3, 011=ALT4, 010=ALT5
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

| الحالة | الأداة | الإجراء |
|---|---|---|
| التحقق من I2C mux صح | Logic Analyzer | ضع probe على SCL/SDA — شوف إن الـ signal بيظهر بعد `pinctrl_select_state()` |
| فحص pull-up/pull-down | Multimeter | قس الـ voltage على الـ pin idle — يكشف لو الـ config اتطبق |
| فحص drive strength | Oscilloscope | قس slew rate الـ signal — يكشف مشاكل الـ `PIN_CONFIG_DRIVE_STRENGTH` |
| تحقق من ALT function | Logic Analyzer | شوف الـ signal pattern على الـ pin — SPI clock pattern مختلف عن UART |
| كشف pin contention | Oscilloscope | شوف رنين أو voltage غريب — دليل على إن اتنين بيحركوا نفس الـ pin |

```
Pin Contention Pattern على الـ Oscilloscope:
─────────────────────────────────────────────
     ___     ___
    |   |   |   |        ← signal طبيعي
____|   |___|   |____

    _________
   /         \
__/           \__        ← ringing / contention — مشكلة!
```

---

#### 4. Hardware Issues الشائعة وأنماطها في الـ Kernel Log

| المشكلة الهاردوير | النمط في dmesg | التحقق |
|---|---|---|
| **Pin مش متوصل** | device probe ينجح بس الـ peripheral مش بيشتغل | `devmem2` على GPFSEL — هل الـ function اتطبق فعلاً؟ |
| **Pull resistor غلط** | الـ I2C بيعطي `NACK` أو الـ UART بيديه garbage | قس الـ pull voltage بـ multimeter |
| **Drive strength ضعيف** | الـ SPI بيشتغل ببطء أو بيفشل عند high speed | Oscilloscope على SCK — شوف الـ slew rate |
| **Pin محجوز من bootloader** | `pin already in use` أو `pin N already registered` | افحص الـ bootloader pinmux config (U-Boot: `gpio status`) |
| **تعارض pins بين drivers** | `Error applying setting` + `gpio request failed` | `cat /sys/kernel/debug/pinctrl/*/pinmux-pins` |
| **DT mismatch مع hardware** | device probe يفشل بـ `-ENODEV` على `pinctrl_lookup_state` | قارن الـ DT compiled مع الـ schematic |

---

#### 5. Device Tree Debugging

```bash
# 1. اقرأ الـ DT المحمل فعلاً في الـ system
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null > /tmp/current.dts

# 2. ابحث عن pinctrl nodes
grep -n "pinctrl" /tmp/current.dts | head -30

# 3. تحقق من الـ pinctrl-0 و pinctrl-names لأي device
grep -A20 "i2c@" /tmp/current.dts | grep -E "pinctrl|status"

# 4. افحص إن الـ phandle في pinctrl-0 بيشير لـ node صح
grep -n "pinctrl-0" /tmp/current.dts

# 5. تحقق من الـ compatible strings
cat /sys/firmware/devicetree/base/soc/gpio@*/compatible 2>/dev/null
```

**مثال DT صح لـ I2C مع pinctrl:**
```dts
i2c1: i2c@fe804000 {
    compatible = "brcm,bcm2835-i2c";
    pinctrl-names = "default";
    pinctrl-0 = <&i2c1_pins>;   /* phandle يشير لـ pinctrl node */
    status = "okay";
};

&gpio {
    i2c1_pins: i2c1 {
        brcm,pins = <2 3>;
        brcm,function = <BCM2835_FSEL_ALT0>;
    };
};
```

**أخطاء DT شائعة:**

```bash
# خطأ 1: pinctrl-names مش موجودة أو غلطت الاسم
# dmesg سيطلع: "failed to lookup the default state"

# خطأ 2: phandle خاطئ في pinctrl-0
# dmesg سيطلع: "unknown pinctrl device ... in map entry, deferring probe"

# خطأ 3: pin group اسمه مختلف عن اللي الـ driver بيطلبه
# dmesg سيطلع: "does not have pin group <name>"

# فحص الـ applied DT overlays (Raspberry Pi مثلاً)
vcdbg log assert 2>/dev/null | grep overlay
cat /proc/device-tree/chosen/bootargs | tr ' ' '\n' | grep dtoverlay
```

---

### Practical Commands

#### أهم الأوامر الجاهزة للنسخ

**فحص شامل — شغّلهم بالترتيب ده:**

```bash
#!/bin/bash
# === pinctrl debug script ===

echo "=== Registered Controllers ==="
cat /sys/kernel/debug/pinctrl/pinctrl-devices

echo ""
echo "=== Active Pin Mappings ==="
cat /sys/kernel/debug/pinctrl/pinctrl-maps

echo ""
echo "=== Current States per Consumer ==="
cat /sys/kernel/debug/pinctrl/pinctrl-handles

echo ""
echo "=== Per-Controller Details ==="
for ctrl_dir in /sys/kernel/debug/pinctrl/*/; do
    ctrl=$(basename "$ctrl_dir")
    # تخطى الـ global files
    [ -d "$ctrl_dir" ] || continue
    [ "$ctrl" = "pinctrl-devices" ] && continue
    [ "$ctrl" = "pinctrl-maps" ] && continue
    [ "$ctrl" = "pinctrl-handles" ] && continue

    echo ""
    echo "--- Controller: $ctrl ---"
    echo "[pins]"
    cat "${ctrl_dir}/pins" 2>/dev/null
    echo ""
    echo "[pingroups]"
    cat "${ctrl_dir}/pingroups" 2>/dev/null
    echo ""
    echo "[gpio-ranges]"
    cat "${ctrl_dir}/gpio-ranges" 2>/dev/null
    echo ""
    if [ -f "${ctrl_dir}/pinmux-pins" ]; then
        echo "[pinmux-pins]"
        cat "${ctrl_dir}/pinmux-pins"
    fi
done
```

---

**تفعيل dynamic debug وجمع logs:**
```bash
# enable
echo 'file drivers/pinctrl/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinmux.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شغّل الـ device probe
echo <dev_name> > /sys/bus/platform/drivers/<driver>/bind

# اجمع الـ logs
dmesg | grep "pinctrl"

# disable
echo 'file drivers/pinctrl/core.c -p' > /sys/kernel/debug/dynamic_debug/control
```

---

**كشف تعارض الـ pins (pin contention):**
```bash
# شوف الـ pins اللي محجوزة من أكتر من حاجة
cat /sys/kernel/debug/pinctrl/*/pinmux-pins 2>/dev/null | grep -v UNCLAIMED

# output مثال:
# pin 2 (SDA1): device fe804000.i2c function i2c1 group i2c1_pins
# pin 3 (SCL1): device fe804000.i2c function i2c1 group i2c1_pins
# pin 14 (TXD0): device fe201000.serial function uart0 group uart0_pins
```

---

**تتبع pinctrl state transitions في real-time:**
```bash
# setup ftrace
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo 'pinctrl_select_state' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'pinctrl_commit_state' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'pinmux_enable_setting' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# (اعمل اللي عايز تتتبعه هنا)
# مثلاً: echo mem > /sys/power/state  ← هيعمل sleep/wake transitions

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace | grep -E "pinctrl|pinmux"

# cleanup
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

---

**فحص deferred probe بسبب pinctrl:**
```bash
# الـ devices اللي عملت defer
cat /sys/kernel/debug/devices_deferred 2>/dev/null

# أو من dmesg
dmesg | grep -i "defer\|EPROBE_DEFER\|pinctrl"

# output نموذجي:
# fe804000.i2c: probe deferral - wait for pinctrl device fe200000.gpio
```

---

**تفسير output `pinctrl-handles` بشكل عملي:**

```bash
cat /sys/kernel/debug/pinctrl/pinctrl-handles
```

```
# Output:
Requested pin control handlers their pinmux maps:
device fe804000.i2c current state: default
  state: default
    type: MUX_GROUP controller fe200000.gpio group: i2c1_pins (2) function: i2c1 (1)

device fe201000.serial current state: default
  state: default
    type: MUX_GROUP controller fe200000.gpio group: uart0_pins (8) function: uart0 (4)
  state: sleep
    type: MUX_GROUP controller fe200000.gpio group: uart0_sleep_pins (9) function: uart0_sleep (5)
```

**التفسير:**
- كل device عنده state حالية (current state)
- كل state عندها settings — كل setting بتقول: `controller` + `group` + `function`
- الـ numbers بين `()` هي الـ selectors في الـ radix tree
- لو `current state: none` = الـ device لسه ما اختارش state أو فيه error

---

**dump الـ DT pinctrl nodes بشكل مقروء:**
```bash
# الطريقة الأسرع
find /sys/firmware/devicetree/base -name "pinctrl*" -o -name "pins" 2>/dev/null \
  | while read f; do
    echo "=== $f ==="; xxd "$f" 2>/dev/null || cat "$f" 2>/dev/null
  done

# أو باستخدام dtc
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null \
  | awk '/pinctrl/,/^[[:space:]]*\}/' | head -100
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART مش شغال على بورد RK3562 — مشكلة pinctrl hog

#### العنوان
**الـ UART2 مش بيظهر على industrial gateway مبني على RK3562**

#### السياق
شركة بتصنع industrial gateway للتحكم في المعدات عبر Modbus RTU. البورد مبنية على SoC **RK3562**، والـ UART2 المفروض يشتغل كـ RS-485. بعد flashing الـ kernel الجديد، الـ `/dev/ttyS2` مش بيظهر، والـ driver بيرجع error أثناء الـ probe.

#### المشكلة
```
[    2.134] serial 2-0062.2: failed to activate default pinctrl state
[    2.135] serial 2-0062.2: probe failed: -22
```
الـ UART driver فشل في تفعيل الـ `default` pinctrl state.

#### التحليل

الـ flow بيبدأ من `pinctrl_init_done()` اللي بتُستدعى بعد الـ probe:

```c
/* core.c, line 1584 */
int pinctrl_init_done(struct device *dev)
{
    struct dev_pin_info *pins = dev->pins;
    /* ... */
    ret = pinctrl_select_state(pins->p, pins->default_state);
    if (ret)
        dev_err(dev, "failed to activate default pinctrl state\n");
    return ret;
}
```

`pinctrl_select_state()` بتكال `pinctrl_commit_state()`:

```c
/* core.c, line 1279 */
static int pinctrl_commit_state(struct pinctrl *p, struct pinctrl_state *state)
{
    /* ... */
    list_for_each_entry(setting, &state->settings, node) {
        switch (setting->type) {
        case PIN_MAP_TYPE_MUX_GROUP:
            ret = pinmux_enable_setting(setting);  /* <-- فشل هنا */
            break;
        /* ... */
        }
        if (ret < 0)
            goto unapply_new_state;
    }
    /* ... */
}
```

الـ `pinmux_enable_setting()` رجّع `-EINVAL` لأن pin group اسمها `"uart2-xfer"` مش موجود في الـ controller. السبب: في الـ DT الجديد، اسم الـ group اتغير من `"uart2-xfer"` لـ `"uart2m1-xfer"` عشان يدعم multiple mux options، لكن الـ `pinctrl-0` في node الـ UART لسه بيشير للـ state القديمة.

الـ `pinctrl_get_group_selector()` في الـ core (line 736) بيعمل string match:

```c
int pinctrl_get_group_selector(struct pinctrl_dev *pctldev,
                               const char *pin_group)
{
    /* ... */
    while (group_selector < ngroups) {
        const char *gname = pctlops->get_group_name(pctldev, group_selector);
        if (gname && !strcmp(gname, pin_group))  /* string compare صارم */
            return group_selector;
        group_selector++;
    }
    dev_err(pctldev->dev, "does not have pin group %s\n", pin_group);
    return -EINVAL;
}
```

#### الحل

**1. تحقق من أسماء الـ groups الموجودة:**
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pingroups | grep uart2
```

**2. صحح الـ DTS:**
```dts
/* قبل */
&uart2 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart2_xfer>;  /* اسم قديم */
    status = "okay";
};

/* بعد */
&uart2 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart2m1_xfer>;  /* اسم صح */
    status = "okay";
};
```

**3. تحقق من الـ state بعد الإصلاح:**
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-handles
# يجب تشوف: device: 2-0062.2  current state: default
```

#### الدرس المستفاد
الـ `pinctrl_get_group_selector()` بيعمل **exact string match** — أي فرق ولو حرف واحد في اسم الـ group في DT بيسبب `-EINVAL` وفشل الـ probe. لازم دايمًا تعمل `grep` في الـ debugfs قبل ما تكتب الـ DT overlays.

---

### السيناريو 2: GPIO مش بيشتغل كـ output على Allwinner H616 — مشكلة GPIO range

#### العنوان
**الـ GPIO للـ LED مش بيتحكم فيه على Android TV box مبني على Allwinner H616**

#### السياق
Android TV box بيستخدم **Allwinner H616** SoC. الـ LED الأخضر مربوط على `PH10` (GPIO bank H, pin 10). الـ BSP driver بيحاول يعمل `gpio_direction_output()` ويفضل فاشل بـ `-EPROBE_DEFER`.

#### المشكلة
```
[    3.891] leds-gpio leds: error -517 getting GPIO
```
الـ `-517` هو `-EPROBE_DEFER`، يعني الـ pinctrl device لسه ما اتسجلش.

#### التحليل

الـ GPIO subsystem بيكال `pinctrl_gpio_request()` (line 800):

```c
int pinctrl_gpio_request(struct gpio_chip *gc, unsigned int offset)
{
    struct pinctrl_gpio_range *range;
    struct pinctrl_dev *pctldev;
    int ret, pin;

    ret = pinctrl_get_device_gpio_range(gc, offset, &pctldev, &range);
    if (ret) {
        /* ret هنا = -EPROBE_DEFER */
        if (pinctrl_ready_for_gpio_range(gc, offset))
            ret = 0;
        return ret;
    }
    /* ... */
}
```

`pinctrl_get_device_gpio_range()` بتلف على كل الـ `pinctrldev_list` وبتدور على range تغطي الـ GPIO offset:

```c
static int pinctrl_get_device_gpio_range(struct gpio_chip *gc,
                                         unsigned int offset,
                                         struct pinctrl_dev **outdev,
                                         struct pinctrl_gpio_range **outrange)
{
    /* ... */
    list_for_each_entry(pctldev, &pinctrldev_list, node) {
        struct pinctrl_gpio_range *range;
        range = pinctrl_match_gpio_range(pctldev, gc, offset);
        if (range) {
            *outdev = pctldev;
            *outrange = range;
            /* ... */
            return 0;
        }
    }
    return -EPROBE_DEFER;  /* ما لقاش range = defer */
}
```

الـ `pinctrl_match_gpio_range()` (line 304) بتتحقق:
```c
if ((gc->base + offset) >= range->base &&
    (gc->base + offset) < range->base + range->npins)
```

المشكلة: الـ Allwinner H616 pinctrl driver بيسجل الـ GPIO ranges في `probe()` باستخدام `pinctrl_add_gpio_range()`، لكنه لسه ما اتسجلش في الـ `pinctrldev_list` لأن `pinctrl_enable()` ما اتكلاش بعد. الترتيب الصح:

```
pinctrl_init_controller()  → ينشئ pctldev لكن ما يضيفوش للـ list
pinctrl_enable()           → يضيف pctldev للـ pinctrldev_list ← لازم يكون قبل أي GPIO request
```

#### الحل

**1. تحقق من حالة الـ pinctrl devices:**
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-devices
# لو H616 pinctrl مش موجود = driver ما اتسجلش
```

**2. تحقق من الـ GPIO ranges:**
```bash
cat /sys/kernel/debug/pinctrl/pio/gpio-ranges
```

**3. الإصلاح في الـ driver:** لو الـ driver بيستخدم `pinctrl_register_and_init()` ولسه ما كالش `pinctrl_enable()`، الـ LED driver هيحصل `-EPROBE_DEFER`. الحل إنك تتأكد إن الـ pinctrl driver بيكمل الـ `pinctrl_enable()` قبل أي consumer:

```c
/* في pinctrl driver probe() */
ret = pinctrl_register_and_init(&h616_pinctrl_desc, dev, priv, &pctldev);
if (ret) return ret;

/* لازم تكال قبل return */
ret = pinctrl_enable(pctldev);
if (ret) return ret;
```

**4. التحقق من device ordering في DT:**
```dts
/* pio (pinctrl) لازم يبقى قبل leds في init order */
pio: pinctrl@300b000 {
    compatible = "allwinner,sun50i-h616-pinctrl";
    /* ... */
};

leds {
    compatible = "gpio-leds";
    /* ... */
};
```

#### الدرس المستفاد
الـ `pinctrl_get_device_gpio_range()` بترجع `-EPROBE_DEFER` لو الـ pinctrl controller لسه ما اتضافش للـ `pinctrldev_list`. ده بيحصل لو `pinctrl_enable()` ما اتكلاتش. الـ `pinctrl_register_and_init()` + `pinctrl_enable()` pattern موجود بالضبط عشان تحل الـ chicken-and-egg problem، لكن لازم تكمل الـ sequence.

---

### السيناريو 3: SPI بيتعارض مع GPIO على STM32MP1 — مشكلة mux_usecount

#### العنوان
**الـ SPI flash مش بيشتغل بعد ما GPIO driver طلب نفس الـ pins على STM32MP1**

#### السياق
Industrial IoT sensor مبني على **STM32MP157**. الـ SPI5 مستخدم لقراءة external flash. في بعض الأحيان، kernel module تاني بيحاول يستخدم `PA8` كـ GPIO لـ chip-select software. بعد تحميل الـ module، الـ SPI بيبدأ يفشل بشكل عشوائي.

#### المشكلة
```
[   45.123] spi-stm32 spi5: failed to transfer one message from queue
[   45.124] spi-stm32 spi5: Error applying setting, reverse things back
```

#### التحليل

لما GPIO driver طلب `PA8`، الـ core كال `pinctrl_gpio_request()` اللي كال `pinmux_request_gpio()`. ده بيتعارض مع الـ SPI5 اللي بالفعل حاجز نفس الـ pin.

الـ `pinctrl_gpio_can_use_line()` (line 763) المفروض يمنع ده:

```c
bool pinctrl_gpio_can_use_line(struct gpio_chip *gc, unsigned int offset)
{
    /* ... */
    if (pinctrl_get_device_gpio_range(gc, offset, &pctldev, &range))
        return true;  /* لو ما فيش range = افتراض ممكن */

    mutex_lock(&pctldev->mutex);
    pin = gpio_to_pin(range, gc, offset);
    result = pinmux_can_be_used_for_gpio(pctldev, pin);
    mutex_unlock(&pctldev->mutex);
    return result;
}
```

الـ `pinmux_can_be_used_for_gpio()` بيتحقق من `mux_usecount` في `struct pin_desc` (defined in core.h):

```c
struct pin_desc {
    /* ... */
#ifdef CONFIG_PINMUX
    unsigned int mux_usecount;   /* لو > 0 = pin محجوز */
    const char *mux_owner;
    const struct pinctrl_setting_mux *mux_setting;
    const char *gpio_owner;
    struct mutex mux_lock;
#endif
};
```

المشكلة: الـ GPIO driver ما كالش `pinctrl_gpio_can_use_line()` قبل الـ request، أو الـ GPIO chip مش مربوط بـ pinctrl range في الـ DT بشكل صحيح، فرجع `true` من أول شرط.

**التتبع الكامل:**
```
gpio_request(PA8)
  └─> pinctrl_gpio_request()          [core.c:800]
        └─> pinctrl_get_device_gpio_range()  [core.c:387]
              └─> لو ما لقاش range → pinctrl_ready_for_gpio_range() → true
                  يعني: ما حصلش check للـ mux_usecount
```

#### الحل

**1. تحقق من الـ pin مين بيستخدمه:**
```bash
cat /sys/kernel/debug/pinctrl/soc:pinctrl@50002000/pins | grep PA8
# يجب تشوف: pin 8 (PA8) ... function: spi5
```

**2. تحقق من الـ gpio-ranges في DT:**
```bash
cat /sys/kernel/debug/pinctrl/soc:pinctrl@50002000/gpio-ranges
```

**3. الحل في DTS — لا تستخدم نفس الـ pin لغرضين:**
```dts
/* غلط: PA8 محجوز لـ SPI5 وكمان في gpio-hog */
&spi5 {
    pinctrl-0 = <&spi5_pins>;  /* PA8 = CLK */
    status = "okay";
};

/* صح: استخدم pin تاني للـ CS */
spi5_cs: spi5-cs {
    compatible = "gpio-leds";
    /* استخدم PB15 مثلاً */
};
```

**4. إذا لازم GPIO على نفس الـ bank، تأكد من الـ pinctrl-0:**
```dts
&gpioa {
    gpio-ranges = <&pinctrl 0 0 16>;
    /* ده بيضمن إن pinctrl_get_device_gpio_range بتلاقي range */
    /* وبالتالي pinmux_can_be_used_for_gpio بيتعمل check صح */
};
```

#### الدرس المستفاد
الـ `pinctrl_gpio_can_use_line()` بتعمل check للـ `mux_usecount` بس لو الـ GPIO chip عنده **gpio-ranges** صحيح في DT ومربوط بالـ pinctrl device. لو الـ range مش موجودة، الـ function بترجع `true` بالافتراض. دي حالة تسبب silent conflict بين الـ SPI pinmux والـ GPIO request.

---

### السيناريو 4: الـ HDMI مش بيشتغل بعد الـ suspend/resume على i.MX8MQ

#### العنوان
**الـ HDMI بيختفي بعد الـ suspend/resume على automotive infotainment مبني على i.MX8MQ**

#### السياق
Automotive ECU للـ infotainment مبني على **i.MX8MQ**. البورد عندها HDMI output للشاشة الأمامية. بعد الـ system suspend (S2R)، الـ HDMI بيظهر على شاشة سودة ومش بيرجع حتى بعد الـ resume.

#### المشكلة
في الـ kernel log بعد الـ resume:
```
[  120.44] hdmi-audio-codec: failed to activate pinctrl state sleep
[  120.45] imx-hdmi 32c00000.hdmi: pinctrl sleep state failed
```

ولما يعمل resume:
```
[  121.10] imx-hdmi 32c00000.hdmi: failed to activate default pinctrl state
```

#### التحليل

الـ power management في pinctrl core بتعتمد على 4 functions (lines 1642-1685):
- `pinctrl_pm_select_default_state()` → للـ resume
- `pinctrl_pm_select_sleep_state()` → للـ suspend
- `pinctrl_pm_select_init_state()`
- `pinctrl_pm_select_idle_state()`

كلهم بيكالوا `pinctrl_select_bound_state()` (line 1608):

```c
static int pinctrl_select_bound_state(struct device *dev,
                                      struct pinctrl_state *state)
{
    struct dev_pin_info *pins = dev->pins;
    int ret;

    if (IS_ERR(state))
        return 0; /* No such state — ده مش error */
    ret = pinctrl_select_state(pins->p, state);
    if (ret)
        dev_err(dev, "failed to activate pinctrl state %s\n",
                state->name);
    return ret;
}
```

اللي كاله هو `pinctrl_select_state()` → `pinctrl_commit_state()`. في `pinctrl_commit_state()`:

```c
static int pinctrl_commit_state(struct pinctrl *p, struct pinctrl_state *state)
{
    struct pinctrl_state *old_state = READ_ONCE(p->state);

    if (old_state) {
        /* بيعطل الـ mux settings القديمة */
        pinctrl_cond_disable_mux_setting(old_state, NULL);
    }

    p->state = NULL;  /* لو حصل error هنا، الـ state = NULL */

    list_for_each_entry(setting, &state->settings, node) {
        switch (setting->type) {
        case PIN_MAP_TYPE_MUX_GROUP:
            ret = pinmux_enable_setting(setting);
            break;
        /* ... */
        }
        if (ret < 0)
            goto unapply_new_state;
    }
    /* ... */
unapply_new_state:
    /* ... */
    if (old_state)
        pinctrl_select_state(p, old_state);  /* محاولة رجوع */
    return ret;
}
```

المشكلة: الـ i.MX8MQ HDMI pinctrl driver ما عنده `"sleep"` state في الـ pin descriptor table. لما الـ suspend بيكال `pinctrl_pm_select_sleep_state()`، الـ `dev->pins->sleep_state` بيكون `ERR_PTR(-ENODEV)` فبترجع 0 (success). لكن الـ hardware الـ HDMI PHY بيدخل في state غريبة لأن الـ pins ما اتحطتش في sleep mode.

لما بيجي الـ resume، `pinmux_enable_setting()` فاشل لأن الـ HDMI PHY لسه في undetermined state.

#### الحل

**1. أضف `"sleep"` state في DTS:**
```dts
&hdmi {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&pinctrl_hdmi_default>;
    pinctrl-1 = <&pinctrl_hdmi_sleep>;  /* sleep state */
    status = "okay";
};

&iomuxc {
    pinctrl_hdmi_default: hdmigrp {
        fsl,pins = <
            MX8MQ_IOMUXC_HDMI_DDC_SCL_HDMI_TX_DDC_SCL  0x400001c3
            MX8MQ_IOMUXC_HDMI_DDC_SDA_HDMI_TX_DDC_SDA  0x400001c3
        >;
    };

    pinctrl_hdmi_sleep: hdmigrp-sleep {
        fsl,pins = <
            MX8MQ_IOMUXC_HDMI_DDC_SCL_GPIO3_IO24  0x100  /* GPIO mode for sleep */
            MX8MQ_IOMUXC_HDMI_DDC_SDA_GPIO3_IO25  0x100
        >;
    };
};
```

**2. تحقق من الـ states المتاحة:**
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-handles | grep hdmi
# يجب تشوف: states: default  sleep
```

**3. تحقق من الـ suspend/resume sequence:**
```bash
# قبل suspend
echo mem > /sys/power/state &
# راقب الـ dmesg
dmesg | grep pinctrl | tail -20
```

#### الدرس المستفاد
الـ `pinctrl_pm_select_sleep_state()` بترجع 0 بصمت لو الـ `sleep` state مش معرّفة (الـ `IS_ERR(state)` check في `pinctrl_select_bound_state`). ده معناه إن الـ pins بتفضل في `default` state أثناء الـ suspend، وده ممكن يسبب مشاكل في الـ hardware. دايمًا اعرّف `"sleep"` state صريح حتى لو كان identical للـ `default`.

---

### السيناريو 5: الـ I2C sensor مش بيتعرف عليه على AM62x — مشكلة pinctrl_dummy_state

#### العنوان
**الـ I2C temperature sensor مش بيتعرف عليه على IoT gateway مبني على AM62x (TI)**

#### السياق
Industrial IoT gateway مبني على **TI AM62x** SoC. في أثناء early bring-up، الـ I2C2 driver بيـ probe بنجاح لكن الـ sensor على العنوان `0x48` مش بيتعرف عليه. الـ `i2cdetect` بيرجع شبكة فاضية.

#### المشكلة
```
[    1.445] i2c i2c-2: Failed to set up pinctrl state 'default'
[    1.446] i2c i2c-2: Error -19 when setting pinctrl state 'default'
```

الـ `-19` هو `-ENODEV`.

#### التحليل

في `pinctrl_lookup_state()` (line 1231):

```c
struct pinctrl_state *pinctrl_lookup_state(struct pinctrl *p,
                                           const char *name)
{
    struct pinctrl_state *state;

    state = find_state(p, name);
    if (!state) {
        if (pinctrl_dummy_state) {
            /* create dummy state */
            dev_dbg(p->dev, "using pinctrl dummy state (%s)\n", name);
            state = create_state(p, name);
        } else
            state = ERR_PTR(-ENODEV);  /* ← الـ error هنا */
    }
    return state;
}
```

لو `pinctrl_dummy_state == false` (الـ default)، والـ `"default"` state مش موجودة في الـ mapping، بيرجع `ERR_PTR(-ENODEV)`.

المشكلة تتبعها: الـ `create_pinctrl()` (line 1052) بيبني الـ states من خلال:

```c
ret = pinctrl_dt_to_map(p, pctldev);
```

`pinctrl_dt_to_map()` بتقرأ `pinctrl-0`, `pinctrl-1`... من الـ DT node. في حالتنا، الـ engineer نسي يضيف `pinctrl-names` و `pinctrl-0` لـ I2C2 node في الـ DT:

```dts
/* غلط - بدون pinctrl */
&i2c2 {
    clock-frequency = <400000>;
    status = "okay";

    tmp102@48 {
        compatible = "ti,tmp102";
        reg = <0x48>;
    };
};
```

لما `create_pinctrl()` بتحاول تبني الـ states من الـ DT، ما بتلاقيش أي mapping، فبترجع `p` فاضية من states. بعدين لما `pinctrl_lookup_state(p, "default")` اتكلت، `find_state()` ما لاقتش state باسم `"default"` في `p->states`، و `pinctrl_dummy_state` = false، فرجعت `-ENODEV`.

**لو `pinctrl_provide_dummies()` اتكلت (line 69):**

```c
void pinctrl_provide_dummies(void)
{
    pinctrl_dummy_state = true;
}
```

كانت `pinctrl_lookup_state()` ستنشئ dummy state فارغة وتتجاهل المشكلة. بعض الـ platforms بتكال ده في الـ board file لتجنب مشاكل الـ bring-up — لكن ده مش الحل الصح على المدى البعيد.

#### الحل

**الحل الصح: أضف pinctrl في DT:**
```dts
/* صح */
&i2c2 {
    pinctrl-names = "default";
    pinctrl-0 = <&i2c2_pins_default>;
    clock-frequency = <400000>;
    status = "okay";

    tmp102@48 {
        compatible = "ti,tmp102";
        reg = <0x48>;
    };
};

&main_pmx0 {
    i2c2_pins_default: i2c2-default-pins {
        pinctrl-single,pins = <
            AM62X_IOPAD(0x0b0, PIN_INPUT_PULLUP, 1)  /* I2C2_SCL */
            AM62X_IOPAD(0x0b4, PIN_INPUT_PULLUP, 1)  /* I2C2_SDA */
        >;
    };
};
```

**تحقق من أن الـ pins اتسجلوا:**
```bash
# شوف الـ pins المتاحة
cat /sys/kernel/debug/pinctrl/pinctrl-single/pins | grep -A2 "b0\|b4"

# شوف الـ groups
cat /sys/kernel/debug/pinctrl/pinctrl-single/pingroups | grep i2c2

# شوف الـ maps
cat /sys/kernel/debug/pinctrl/pinctrl-maps | grep i2c2
```

**لو في early bring-up وعايز تتجاهل الـ pinctrl مؤقتًا:**
```c
/* في board file أو early init */
pinctrl_provide_dummies();  /* تسمح بـ dummy states */
```

**لكن تأكد من الـ physical pins:**
```bash
# تحقق من أن الـ I2C lines مش floating
# لو الـ mux مش ضبطان، الـ I2C هيفضل مش شغال حتى بعد dummy state
i2cdetect -y 2
```

#### الدرس المستفاد
الـ `pinctrl_lookup_state()` بتبحث عن state اسمها `"default"` في قايمة الـ states اللي اتبنت من الـ DT. لو الـ DT node مالوش `pinctrl-names`/`pinctrl-0`، القايمة فاضية وبيجي `-ENODEV`. الـ `pinctrl_provide_dummies()` حل مؤقت لأغراض الـ bring-up بس — استخدامه في production بيعني إن الـ pins ممكن تفضل على الـ reset state (غالبًا GPIO) وده ممكن يمنع الـ I2C من الشغل تمامًا على مستوى الـ hardware.
## Phase 7: مصادر ومراجع

### مصادر رسمية — Kernel Documentation

| المصدر | الرابط |
|--------|--------|
| **Official pinctrl docs** (kernel.org) | [Documentation/pinctrl.txt](https://www.kernel.org/doc/Documentation/pinctrl.txt) |
| **Kernel Driver API — PINCTRL subsystem** | [kernel.org/doc/html/v4.14/driver-api/pinctl.html](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html) |
| **Source tree** | `drivers/pinctrl/core.c` — الـ entry point الرئيسي للـ subsystem |
| **Headers** | `include/linux/pinctrl/pinctrl.h`, `include/linux/pinctrl/consumer.h`, `include/linux/pinctrl/machine.h` |

مسارات الـ `Documentation/` المهمة داخل الـ kernel source tree:

```
Documentation/driver-api/pinctl.rst        ← الوثيقة الرئيسية
Documentation/devicetree/bindings/pinctrl/ ← DT bindings لكل SoC
```

---

### LWN.net — أهم المقالات

دي أهم المقالات اللي غطّت نشأة وتطور الـ pinctrl subsystem:

| المقال | الأهمية |
|--------|---------|
| [pin controller subsystem v7](https://lwn.net/Articles/459190/) | أول patch series رسمية من Linus Walleij قدّمت الـ subsystem |
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | الـ patch اللي اتقبلت فعلاً في الـ mainline |
| [drivers: create a pin control subsystem v8](https://lwn.net/Articles/460768/) | النسخة المُعدَّلة قبل الـ merge |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | الوثيقة الرسمية الأولى للـ subsystem |
| [The pin control subsystem](https://lwn.net/Articles/468759/) | مقالة شرح مفصّلة للـ concepts |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/) | إضافة الـ `pinconf` layer |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | تطوير الـ config interface |
| [pinctrl: add a generic pin config interface (v2)](https://lwn.net/Articles/467269/) | نسخة أخرى من الـ pinconf proposal |
| [drivers: pinctrl sleep and idle states in the core](https://lwn.net/Articles/552972/) | إضافة `sleep` و`idle` states |
| [drivers: pinctrl — "init" state concept](https://lwn.net/Articles/615322/) | الـ `init` state اللي بتتطبّق قبل `probe` |
| [pinctrl: introduce GPIO pin function category](https://lwn.net/Articles/1031226/) | ربط الـ pinctrl بالـ GPIO API |

---

### Kernel Commits — أهم التغييرات التاريخية

**الـ pinctrl subsystem اتقبل في Linux 3.2 (December 2011).** أهم الـ commits:

```bash
# أول commit للـ subsystem
git log --oneline --all -- drivers/pinctrl/core.c | tail -5

# البحث في lore.kernel.org عن تاريخ الـ patches
# https://lore.kernel.org/linux-gpio/
```

| الحدث | الإصدار |
|-------|---------|
| إضافة الـ pinctrl core | Linux 3.2 |
| إضافة الـ pinconf (pin configuration) | Linux 3.3 |
| إضافة الـ `sleep` و `idle` states | Linux 3.11 |
| إضافة الـ `init` state | Linux 3.18 |
| دمج الـ GPIO و pinctrl APIs | Linux 4.5+ |

---

### Mailing Lists — قوائم البريد الإلكتروني

الـ pinctrl patches والنقاشات بتعدي على قائمتين أساسيتين:

| القائمة | الرابط |
|---------|--------|
| **linux-gpio** (pinctrl + GPIO) | [lore.kernel.org/linux-gpio/](https://lore.kernel.org/linux-gpio/) |
| **LKML** (العام) | [lkml.org](https://lkml.org/) |
| **Spinics archive** | [spinics.net/lists/kernel/](https://www.spinics.net/lists/kernel/) |

للبحث عن نقاش معيّن:
```bash
# مثال: البحث عن pinctrl_register في lore
https://lore.kernel.org/linux-gpio/?q=pinctrl_register
```

---

### eLinux.org — مصادر Embedded

| الصفحة | المحتوى |
|--------|---------|
| [EBC Device Trees](https://elinux.org/EBC_Device_Trees) | أمثلة عملية لـ `pinctrl-single` على BeagleBone |
| [Tests: i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) | اختبار الـ `i2c-demux-pinctrl` driver |
| [eLinux Pin Control PDF](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) | عرض تقديمي عن تحديثات الـ pinctrl/GPIO |

---

### KernelNewbies.org — متابعة التغييرات بالإصدار

صفحات الـ changelogs بتحتوي تفاصيل تغييرات الـ pinctrl في كل إصدار:

| الإصدار | الرابط |
|---------|--------|
| Linux 6.1 | [kernelnewbies.org/Linux_6.1](https://kernelnewbies.org/Linux_6.1) |
| Linux 6.8 | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) |
| Linux 6.11 | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| Linux 6.13 | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) |
| Linux 6.14 | [kernelnewbies.org/Linux_6.14](https://kernelnewbies.org/Linux_6.14) |
| Linux 6.17 | [kernelnewbies.org/Linux_6.17](https://kernelnewbies.org/Linux_6.17) |
| All Changes | [kernelnewbies.org/LinuxChanges](https://kernelnewbies.org/LinuxChanges) |

---

### كتب موصى بيها

#### Linux Device Drivers (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم**: Chapter 9 — Communicating with Hardware (GPIO, registers)
- **ملاحظة**: الكتاب قديم (2005) ومش بيغطي الـ pinctrl، لكنه أساس لفهم الـ driver model
- [متاح مجاناً](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل المهم**: Chapter 17 — Devices and Modules
- بيشرح الـ device model اللي الـ pinctrl مبني عليه

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل المهم**: Chapter 15 — Debugging Embedded Linux Applications
- بيغطي الـ pin muxing في سياق الـ embedded SoCs

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds
- أحدث كتاب عملي بيغطي الـ pinctrl في سياق Yocto و Buildroot

---

### المقال الخارجي المميّز

| المصدر | الرابط |
|--------|--------|
| Embedded.com — The pin control subsystem | [embedded.com/linux-device-driver-development-the-pin-control-subsystem/](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) |

الـ article ده بيشرح الـ subsystem بشكل عملي من منظور الـ driver developer.

---

### كلمات البحث — Search Terms

للبحث عن معلومات إضافية استخدم:

```
# بحث عام
"linux pinctrl subsystem"
"pinctrl_register_and_init"
"pinctrl_dev pinctrl_desc"
"struct pinctrl_ops"
"pin multiplexing linux kernel"

# بحث في lore.kernel.org
https://lore.kernel.org/linux-gpio/?q=pinctrl+core

# بحث في elixir (cross-reference)
https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl/core.c

# بحث في kernel git
git log --oneline -- drivers/pinctrl/core.c
git log --oneline -- include/linux/pinctrl/
```

---

### Elixir Cross-Reference — أداة التصفح المباشر

الأداة الأهم لقراءة الـ source code مع الـ cross-references:

| الرابط | الوصف |
|--------|-------|
| [elixir.bootlin.com — core.c](https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl/core.c) | الـ `core.c` مع كل الـ references |
| [elixir.bootlin.com — pinctrl.h](https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/pinctrl.h) | الـ main header |
| [elixir.bootlin.com — consumer.h](https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/consumer.h) | الـ consumer API |
## Phase 8: Writing simple module

### الفكرة

**`pinctrl_select_state()`** هي أنسب function نعمل عليها kprobe — بتتنادى كل ما device بتغير الـ pin state بتاعتها (مثلاً من `init` لـ `default` أو من `default` لـ `sleep`)، وبتعدّي معاها الـ `pinctrl` handle والـ `pinctrl_state` handle اللي فيهم اسم الـ device واسم الـ state الجديدة.

---

### الـ Module

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe on pinctrl_select_state()
 * Logs: device name + new pin state name every time a driver switches states.
 */

/* --- Includes --- */
#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit         */
#include <linux/kprobes.h>     /* kprobe, register_kprobe, unregister_kprobe */
#include <linux/pinctrl/consumer.h>  /* struct pinctrl, struct pinctrl_state    */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe on pinctrl_select_state to trace pin state transitions");

/*
 * pinctrl و pinctrl_state هما structs خاصة بالـ core وmعرفوش في headers عامة,
 * بس بنقدر نوصل للمعلومات اللي محتاجينها من خلال الـ fields الأولى في الـ structs.
 *
 * struct pinctrl  { struct device *dev; ... }    -- dev في البداية
 * struct pinctrl_state { const char *name; ... } -- name في البداية
 *
 * بنستخدم forward declarations بدل include للـ internal headers.
 */

/* Forward declarations matching core.h internal layout */
struct pinctrl {
    struct device      *dev;     /* owning device */
    /* rest of fields not needed */
};

struct pinctrl_state {
    const char         *name;    /* state name: "default", "sleep", etc. */
    /* rest of fields not needed */
};

/* --- kprobe callback --- */

/*
 * pre_handler: بيتنادى قبل تنفيذ الـ function المستهدفة مباشرة.
 * regs->di = أول argument (p)، regs->si = تاني argument (state) على x86_64.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Extract the two arguments from registers (x86_64 calling convention) */
    struct pinctrl       *pctl  = (struct pinctrl *)regs->di;
    struct pinctrl_state *state = (struct pinctrl_state *)regs->si;

    /* Guard against NULL — defensive programming inside kernel probes */
    if (!pctl || !state)
        return 0;

    pr_info("pinctrl_select_state: dev=%s -> state=%s\n",
            dev_name(pctl->dev),   /* e.g. "2000000.uart"            */
            state->name);          /* e.g. "default", "sleep", "init" */

    return 0; /* 0 = let the original function run normally */
}

/* --- kprobe struct --- */

/*
 * الـ kprobe struct بيحدد الـ symbol اللي هنحط عليه الـ probe،
 * والـ pre_handler اللي بيتنادى قبل كل call.
 */
static struct kprobe kp = {
    .symbol_name = "pinctrl_select_state",
    .pre_handler = handler_pre,
};

/* --- Module init --- */

/*
 * بنسجّل الـ kprobe هنا عشان الـ kernel يعرف يعترض الـ calls
 * ويحوّلها للـ handler بتاعنا قبل ما ينفذ الـ original function.
 */
static int __init pinctrl_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe registered on pinctrl_select_state @ %p\n", kp.addr);
    return 0;
}

/* --- Module exit --- */

/*
 * الـ unregister ضروري عشان لو نزلنا الـ module من غير ما نشيل الـ kprobe،
 * الـ kernel هيفضل يجري الـ handler على function اتشالت من الذاكرة = kernel panic.
 */
static void __exit pinctrl_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe unregistered from pinctrl_select_state\n");
}

module_init(pinctrl_probe_init);
module_exit(pinctrl_probe_exit);
```

---

### Makefile

```makefile
obj-m += pinctrl_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### شرح كل قسم

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | لازم لأي kernel module — بيعرّف `module_init`, `module_exit`, الـ macros دي |
| `linux/kprobes.h` | بيعرّف `struct kprobe`, `register_kprobe`, `unregister_kprobe`, و`pt_regs` |
| `linux/pinctrl/consumer.h` | بيعرّف `struct pinctrl` و `struct pinctrl_state` كـ forward declarations |

**الـ** `pinctrl` و `pinctrl_state` هما structs داخليين في الـ core، بس بيعرفوا الـ `dev` و `name` كـ أول field — فبنعمل re-declaration محلي بس بالـ fields اللي محتاجينهم.

---

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

- **`struct pt_regs *regs`**: بيحتوي على الـ CPU registers وقت الـ intercept. على x86_64 بيبقى:
  - `regs->di` = أول argument = `struct pinctrl *p`
  - `regs->si` = تاني argument = `struct pinctrl_state *state`
- **`dev_name(pctl->dev)`**: بيرجع اسم الـ device زي `"2000000.uart"` أو `"soc:pinctrl"`.
- **`state->name`**: بيرجع اسم الـ state زي `"default"`, `"sleep"`, `"init"`.
- **`return 0`**: معناها "الـ original function تكمل طبيعي" — لو رجعنا 1 هيتخطى الـ function كلها.

**الـ** NULL check مهم عشان الـ handler بيتنادى في أي وقت والـ race conditions ممكنة.

---

#### الـ `struct kprobe kp`

```c
.symbol_name = "pinctrl_select_state",
```

الـ kernel بيدور على الـ symbol ده في الـ kallsyms table وبيحط breakpoint افتراضي عليه. لو الـ function اتضمنت `inline` أو اتزالت بالـ LTO مش هتشتغل — بس `pinctrl_select_state` مضمون موجود كـ `EXPORT_SYMBOL_GPL`.

---

#### الـ `module_init` / `module_exit`

- **`register_kprobe(&kp)`**: بيعمل patch في الـ kernel code عشان يحط الـ breakpoint — لو فشل (مثلاً الـ symbol مش موجود أو الـ kernel مش supportive) بيرجع error.
- **`unregister_kprobe(&kp)`**: **إجباري** في الـ exit عشان يشيل الـ patch. لو المودول اتنزل من غيره، الـ handler pointer هيبقى dangling pointer وأي call لـ `pinctrl_select_state` هتعمل crash.

---

### تجربة الـ Module

```bash
# بناء المودول
make

# تحميل المودول
sudo insmod pinctrl_probe.ko

# مشاهدة الـ output — بيظهر كل ما device تغير state
sudo dmesg -w | grep "pinctrl_select_state"

# مثال للـ output المتوقع:
# [  42.123456] kprobe registered on pinctrl_select_state @ ffffffffc0123456
# [  43.789012] pinctrl_select_state: dev=2000000.uart -> state=default
# [  44.001234] pinctrl_select_state: dev=soc:pinctrl@0 -> state=sleep

# إزالة المودول
sudo rmmod pinctrl_probe
```

---

### ملاحظة على ARM64

على **ARM64** الـ calling convention بيختلف — الـ arguments بتتمرر في:
- `regs->regs[0]` = أول argument
- `regs->regs[1]` = تاني argument

فلو هتشغّل المودول على ARM هتبدّل `regs->di` / `regs->si` بـ `regs->regs[0]` / `regs->regs[1]`.
