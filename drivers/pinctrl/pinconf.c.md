## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي بتنتمي ليه الملفات دي

الملفان `pinconf.c` و `pinconf.h` جزء من **PIN CONTROL SUBSYSTEM** — المنظومة المسؤولة عن إدارة الـ pins الجسدية في المعالجات المدمجة (SoCs). المسؤول عنها Linus Walleij، وبتتابعها القائمة `linux-gpio@vger.kernel.org`.

---

### القصة من الأول: ليه محتاجين حاجة زي دي أصلاً؟

تخيل عندك شريحة معالج (SoC) زي اللي بتلاقيها في الموبايل أو الـ Raspberry Pi. الشريحة دي فيها مئات من الـ **pins** — الأرجل الصغيرة اللي بتتوصل بيها مع العالم الخارجي. كل pin ممكن يشتغل بأكتر من طريقة:

- يبعت بيانات عبر I2C أو SPI أو UART
- يكون GPIO عادي (input أو output)
- يكون pin لجهاز صوت أو كاميرا

بس مش بس الوظيفة اللي بتتغير — **السلوك الكهربي** بيتغير برضو. كل pin ممكن تضبطه:

| الإعداد | المعنى |
|---|---|
| **pull-up** | اسحب الـ pin لمستوى الجهد العالي (VDD) عبر مقاومة داخلية |
| **pull-down** | اسحبه للأرضي (GND) |
| **drive strength** | قوة التيار اللي بيبعته |
| **open-drain** | نوع خاص من الإخراج للبصات المشتركة |
| **input debounce** | تصفية الاهتزازات في الإشارة |
| **schmitt-trigger** | تحسين الاستجابة للإشارات البطيئة |

ده اللي بيسموه **pin configuration** — إعداد الخصائص الكهربية للـ pin، مستقل تماماً عن اختيار وظيفته (اللي بتتعمل في الـ pinmux).

---

### تشبيه من الواقع

تخيل مكياج الوجه. عندك نفس الخد، بس ممكن تختار:
- هل تحطه مبلل ولا جاف؟
- هل تزيد تغطيته ولا تخليه خفيف؟
- هل تحطله تثبيت قوي ولا عادي؟

ده بالظبط الـ pin configuration — الـ pin موجود وشغال، بس أنت بتضبط **طريقة تصرفه الكهربي** مش وظيفته.

---

### الهدف من `pinconf.c` و `pinconf.h`

الملفان دول هما **الطبقة الوسيطة** (middleware layer) بين:

1. **الـ core** — اللي بيدير الـ pins ويعرف فين كل حاجة
2. **درايفر الهاردوير** — اللي بيعرف إزاي يكتب للـ registers الحقيقية

```
+---------------------------+
|   Device Driver           |  (e.g., I2C, SPI, GPIO consumer)
+---------------------------+
           |
           v
+---------------------------+
|   pinctrl core (core.c)   |  يستقبل الطلب ويوزعه
+---------------------------+
           |
           v
+---------------------------+
|  pinconf.c  ← أنت هنا    |  يترجم الـ settings للـ ops
+---------------------------+
           |
           v
+---------------------------+
|  HW Driver (e.g.,         |  يكتب للـ registers الفعلية
|  pinctrl-imx.c)           |
+---------------------------+
```

**الملف `pinconf.c` بيعمل 4 حاجات أساسية:**

1. **Validation** — يتأكد إن الـ map صح قبل ما يتطبق (`pinconf_validate_map`, `pinconf_check_ops`)
2. **Translation** — يحول اسم الـ pin من string لـ ID رقمي (`pinconf_map_to_setting`)
3. **Application** — يطبق الإعدادات فعلياً على الـ pin أو الـ group (`pinconf_apply_setting`)
4. **Debug** — يعرض الإعدادات في debugfs (`pinconf_init_device_debugfs`)

---

### القصة الكاملة: من الـ Device Tree للـ Register

لما بتشغل الـ kernel، هو بيقرأ الـ Device Tree أو board file ويلاقي حاجة زي:

```
&uart0 {
    pinctrl-0 = <&uart0_pins>;
};

uart0_pins: uart0-pins {
    pins = "GPIO_14", "GPIO_15";
    bias-pull-up;
    drive-strength = <4>;
};
```

**المسار كالآتي:**

1. الـ `pinctrl core` بيقرأ الـ map ويشوف إن فيه `PIN_MAP_TYPE_CONFIGS_PIN`
2. بيستدعي `pinconf_validate_map()` يتأكد إن الـ map صحيح
3. بيستدعي `pinconf_map_to_setting()` يحول اسم "GPIO_14" لـ pin number رقمي
4. لما الديفايس بيتفعّل بيستدعي `pinconf_apply_setting()`
5. اللي بيستدعي `ops->pin_config_set()` في درايفر الهاردوير الفعلي
6. الدرايفر يكتب للـ register المناسب ويخلي الـ pin يعمل بـ pull-up

---

### الفرق بين Pin و Group

الـ framework بيدعم نوعين من الإعداد:

| النوع | المعنى | متى تستخدمه |
|---|---|---|
| `PIN_MAP_TYPE_CONFIGS_PIN` | إعداد pin واحد | لما كل pin محتاج إعداد مختلف |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | إعداد مجموعة pins بإعداد واحد | لما كل pins في group بتاخد نفس الإعداد |

الـ group approach أسرع وأبسط — driver واحد بيكتب للكل دفعة واحدة.

---

### الـ `pinconf_ops` — العقد مع الهاردوير

الملف `include/linux/pinctrl/pinconf.h` بيعرّف الـ struct الأساسي:

```c
struct pinconf_ops {
    /* اقرأ إعداد pin معين */
    int (*pin_config_get)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *config);

    /* اكتب إعداد على pin معين */
    int (*pin_config_set)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *configs,
                          unsigned int num_configs);

    /* اقرأ إعداد group كامل */
    int (*pin_config_group_get)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *config);

    /* اكتب إعداد على group كامل */
    int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *configs,
                                unsigned int num_configs);

    /* debug hooks اختيارية */
    void (*pin_config_dbg_show)(...);
    void (*pin_config_group_dbg_show)(...);
    void (*pin_config_config_dbg_show)(...);
};
```

كل درايفر هاردوير لازم يملأ الـ struct ده ويسجله مع الـ core. الـ `pinconf.c` هو اللي بيستدعي الـ function pointers دي.

---

### الـ Config Value: إزاي بتعبر عن إعداد في `unsigned long`؟

الـ config بيتخزن في `unsigned long` واحد بيتقسم:

```
 31          16 15           0
+-------------+---------------+
|   param     |     value     |
+-------------+---------------+
```

- الـ **param** بيقول *إيه* اللي عايز تعمله (pull-up, drive-strength, ...)
- الـ **value** بيقول *قيمته* (مثلاً Ohms للـ pull أو mA للـ drive strength)

ده بيتعامل معاه في `include/linux/pinctrl/pinconf-generic.h` عبر macros زي `PIN_CONF_PACKED(param, value)`.

---

### الـ debugfs Support

لو الـ kernel اتبنى بـ `CONFIG_DEBUG_FS`، الملف بيضيف:

```
/sys/kernel/debug/pinctrl/<device>/
├── pinconf-pins    ← إعدادات كل pin
└── pinconf-groups  ← إعدادات كل group
```

ده بيخليك تعمل:
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-handles
```
وتشوف الإعدادات الحالية في الـ production system.

---

### الفرق بين `CONFIG_PINCONF` و `CONFIG_GENERIC_PINCONF`

| Config | المعنى |
|---|---|
| `CONFIG_PINCONF` | تفعيل الـ pin configuration بشكل عام |
| `CONFIG_GENERIC_PINCONF` | استخدام نظام الـ encoding الموحد (`pinconf-generic.c`) بدل ما كل driver يعمل encoding خاص بيه |

لو الـ driver استخدم `CONFIG_GENERIC_PINCONF`، الـ Device Tree parsing بتتم تلقائياً عبر `pinconf_generic_parse_dt_config()`.

---

### الملفات المكوّنة للـ Subsystem

#### Core Framework
| الملف | الدور |
|---|---|
| `drivers/pinctrl/core.c` | القلب — التسجيل، الـ state machine، الـ locking |
| `drivers/pinctrl/core.h` | الـ structs الداخلية (`pinctrl_dev`, `pin_desc`, `pinctrl_setting`) |
| `drivers/pinctrl/pinconf.c` | **طبقة الـ pin configuration** — موضوع دراستنا |
| `drivers/pinctrl/pinconf.h` | الـ internal API للـ pinconf |
| `drivers/pinctrl/pinconf-generic.c` | الـ generic encoding/decoding للـ configs |
| `drivers/pinctrl/devicetree.c` | قراءة الإعدادات من Device Tree |

#### Public Headers
| الملف | الدور |
|---|---|
| `include/linux/pinctrl/pinconf.h` | تعريف `pinconf_ops` — العقد مع الهاردوير |
| `include/linux/pinctrl/pinconf-generic.h` | الـ `pin_config_param` enum وكل القيم الممكنة |
| `include/linux/pinctrl/pinctrl.h` | الـ public API للـ consumers |
| `include/linux/pinctrl/machine.h` | تعريف الـ `pinctrl_map` — هيكل الـ mapping table |
| `include/linux/pinctrl/consumer.h` | الـ API اللي بتستخدمه الـ device drivers |
| `include/linux/pinctrl/pinctrl-state.h` | أسماء الـ states (`default`, `sleep`, `idle`) |

#### Hardware Drivers (أمثلة)
| الملف | الهاردوير |
|---|---|
| `drivers/pinctrl/freescale/pinctrl-imx.c` | NXP i.MX processors |
| `drivers/pinctrl/intel/pinctrl-intel.c` | Intel SoCs |
| `drivers/pinctrl/pinctrl-amd.c` | AMD platforms |
| `drivers/pinctrl/bcm/pinctrl-bcm2835.c` | Raspberry Pi (BCM2835) |
## Phase 2: شرح الـ Pin Configuration (pinconf) Framework

---

### المشكلة اللي بيحلها الـ pinconf

أي pin جسمانيًا على الـ SoC مش بس سلك — ده مجموعة دوائر إلكترونية كاملة جوا الـ IO cell. كل pin ممكن يتحكم فيه على مستوى كهربي دقيق جدًا:

- **Pull-up / Pull-down**: مقاومة داخلية بتشد الخط لـ VDD أو GND
- **Drive strength**: الـ current اللي الـ output buffer يقدر يحقنه في الخط (بالـ mA)
- **Slew rate**: سرعة الـ rising/falling edge (مهم جدًا في الـ EMI)
- **Schmitt trigger**: hysteresis عند الـ input لتحسين immunity للـ noise
- **Open-drain / Push-pull**: نوع الـ output driver
- **Debounce**: تجاهل الـ bouncing للسيجنال لفترة زمنية محددة
- وغيرها كتير

**الـ problem**: كل SoC بتعمل ده بطريقة مختلفة خالص. Qualcomm بتكتب register بـ format مختلف عن NXP اللي مختلف عن ST. لو كل driver اخترع طريقته الخاصة لتمرير هذه الـ configs، هيبقى في kernel code شبه مستحيل يتمنتن.

**التحدي التاني**: إزاي تكتب الـ pin configuration في الـ Device Tree بصورة معيارية وأي driver يفهمها؟ مش معقول تكتب `st-pullup = <1>` وبعدين `qcom,pull-up = <1>` لكل vendor.

---

### الحل اللي اتخده الـ kernel

الـ kernel فصل المسألة لطبقتين:

1. **Generic pinconf layer**: طبقة مجردة بتعرّف مفردات موحدة (`PIN_CONFIG_BIAS_PULL_UP`, `PIN_CONFIG_DRIVE_STRENGTH`, إلخ) وبتحوّل الـ Device Tree properties الاسمية دي لـ `unsigned long` values مضغوطة.

2. **Driver ops layer**: كل pin controller driver بيسجّل `struct pinconf_ops` فيها function pointers بتطبّق الـ configs الجينيرية دي على الـ hardware الحقيقية.

الـ core هو الوسيط: بيستقبل الـ config من فوق (Device Tree / board file) ويوجّهها للـ driver الصح.

---

### الـ Real-World Analogy — بالتفصيل

تخيل شركة إنشاءات بتبني مباني في دول مختلفة. الـ **مواصفات الكهرباء** موحدة: "مقياس كابل 2.5mm²، مفتاح 16A، أرضي مطلوب". دي الـ **Generic pinconf parameters** — لغة مشتركة مفهومة لكل مهندس.

لكن كل دولة عندها **كود كهرباء محلي مختلف**: في مصر بتربط بطريقة، في ألمانيا بطريقة ثانية. الـ **pin controller driver** هو المقاول المحلي اللي يعرف يطبّق المواصفات الموحدة دي على الواقع.

الـ **pinconf core** هو مكتب الاستشارات الهندسية اللي:
- بيستقبل الـ specs من المهندس المصمم (الـ Device Tree / board file)
- بيتحقق إن الـ specs صحيحة ومكتملة (`pinconf_check_ops`, `pinconf_validate_map`)
- بيوجّهها للمقاول المناسب (`pinconf_apply_setting`)
- بيحتفظ بأرشيف لكل الأعمال المنجزة (الـ debugfs)

والـ `PIN_CONF_PACKED` macro هي الـ **نموذج الموحد** للتواصل — رقم المواصفة في الـ 8 bits السفلية والقيمة في الـ 24 bits العليا.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      USERSPACE / BOARD                          │
│   Device Tree (bias-pull-up, drive-strength, etc.)              │
│   Board files (PIN_MAP_CONFIGS_PIN macros)                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │  DT properties / pinctrl_map[]
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              pinctrl CORE (drivers/pinctrl/core.c)              │
│  - pinctrl_get() / pinctrl_select_state()                       │
│  - manages pinctrl_map → pinctrl_setting translation            │
└──────────┬────────────────┬────────────────────────────────────-┘
           │                │
           ▼                ▼
┌──────────────────┐  ┌─────────────────────────────────────────┐
│  pinmux core     │  │        pinconf CORE                     │
│ (pinmux.c)       │  │   (pinconf.c + pinconf-generic.c)       │
│                  │  │                                         │
│ handles MUX_GROUP│  │  pinconf_validate_map()                 │
│ settings         │  │  pinconf_map_to_setting()               │
│                  │  │  pinconf_apply_setting()                │
│                  │  │  pinconf_generic_parse_dt_config()      │
└──────────────────┘  └──────────────────┬────────────────────-─┘
                                         │  calls ops->pin_config_set()
                                         │  or ops->pin_config_group_set()
                     ┌───────────────────┴──────────────────────┐
                     │         Pin Controller Drivers            │
                     │                                           │
                     │  ┌─────────────┐  ┌──────────────────┐   │
                     │  │ pinctrl-imx │  │ qcom-pinctrl     │   │
                     │  │  (NXP i.MX) │  │  (Qualcomm SoCs) │   │
                     │  └──────┬──────┘  └────────┬─────────┘   │
                     │         │                  │              │
                     └─────────┴──────────────────┴─────────────┘
                               │                  │
                    ┌──────────┴──┐        ┌──────┴──────────┐
                    │  IOMUXC    │        │  TLMM Registers  │
                    │  Registers  │        │  (Qualcomm HW)   │
                    └────────────┘        └─────────────────-┘
```

---

### الـ Core Abstraction: الـ Config كـ Packed `unsigned long`

ده أهم فكرة في الـ subsystem كله. كل pin configuration بتتمثل في **`unsigned long` واحد** مضغوط:

```
 63                    8  7               0
 ┌────────────────────────┬───────────────┐
 │      argument (24 bit) │  param (8 bit)│
 └────────────────────────┴───────────────┘
```

```c
/* Pack: param في الـ low byte, argument في الـ high 24 bits */
#define PIN_CONF_PACKED(p, a)  ((a << 8) | ((unsigned long) p & 0xffUL))

/* Unpack الـ param */
static inline enum pin_config_param pinconf_to_config_param(unsigned long config)
{
    return (enum pin_config_param)(config & 0xffUL);
}

/* Unpack الـ argument */
static inline u32 pinconf_to_config_argument(unsigned long config)
{
    return (u32)((config >> 8) & 0xffffffUL);
}
```

**مثال عملي**:
```c
/* pull-up بـ 10k ohm */
unsigned long cfg = PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 10000);
/* cfg = 0x00002706 */
/* param = 0x05 (PIN_CONFIG_BIAS_PULL_UP), argument = 10000 (0x2710) */

/* drive strength = 8mA */
unsigned long cfg2 = PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 8);
```

الـ driver بعدين بياخد الـ `unsigned long` ده ويفك تشفيره ويكتبه في الـ register المناسبة.

---

### الـ `enum pin_config_param` — قاموس الـ Pinconf

ده الـ vocabulary الموحد اللي الـ kernel عرّفه. كل قيمة بتمثل خاصية كهربية:

```c
enum pin_config_param {
    PIN_CONFIG_BIAS_BUS_HOLD,       /* bus keeper - يحافظ على آخر قيمة */
    PIN_CONFIG_BIAS_DISABLE,        /* يشيل أي pull */
    PIN_CONFIG_BIAS_HIGH_IMPEDANCE, /* tristate / floating */
    PIN_CONFIG_BIAS_PULL_DOWN,      /* pull-down للـ GND (argument = ohms) */
    PIN_CONFIG_BIAS_PULL_PIN_DEFAULT, /* hardware-defined pull */
    PIN_CONFIG_BIAS_PULL_UP,        /* pull-up للـ VDD (argument = ohms) */
    PIN_CONFIG_DRIVE_OPEN_DRAIN,    /* open-drain output */
    PIN_CONFIG_DRIVE_OPEN_SOURCE,   /* open-source/emitter output */
    PIN_CONFIG_DRIVE_PUSH_PULL,     /* push-pull (الـ default) */
    PIN_CONFIG_DRIVE_STRENGTH,      /* تيار الـ output بالـ mA */
    PIN_CONFIG_DRIVE_STRENGTH_UA,   /* تيار الـ output بالـ uA */
    PIN_CONFIG_INPUT_DEBOUNCE,      /* debounce time بالـ usec */
    PIN_CONFIG_INPUT_ENABLE,        /* تفعيل / تعطيل الـ input buffer */
    PIN_CONFIG_INPUT_SCHMITT,       /* schmitt trigger mode */
    PIN_CONFIG_INPUT_SCHMITT_ENABLE, /* تفعيل الـ schmitt */
    PIN_CONFIG_INPUT_SCHMITT_UV,    /* schmitt threshold بالـ uV */
    PIN_CONFIG_MODE_LOW_POWER,      /* low power mode */
    PIN_CONFIG_MODE_PWM,            /* PWM mode */
    PIN_CONFIG_LEVEL,               /* output level (0 أو 1) */
    PIN_CONFIG_OUTPUT_ENABLE,       /* تفعيل الـ output buffer */
    PIN_CONFIG_OUTPUT_IMPEDANCE_OHMS, /* impedance الخروج */
    PIN_CONFIG_PERSIST_STATE,       /* احتفظ بالـ config بعد reset */
    PIN_CONFIG_POWER_SOURCE,        /* اختيار مصدر الطاقة */
    PIN_CONFIG_SKEW_DELAY,          /* تأخير الـ clock */
    PIN_CONFIG_SKEW_DELAY_INPUT_PS, /* تأخير الـ input بالـ picoseconds */
    PIN_CONFIG_SKEW_DELAY_OUTPUT_PS, /* تأخير الـ output بالـ picoseconds */
    PIN_CONFIG_SLEEP_HARDWARE_STATE, /* حالة الـ sleep */
    PIN_CONFIG_SLEW_RATE,           /* معدل تغير الـ edge */
    PIN_CONFIG_END = 0x7F,          /* آخر قيمة standard */
    PIN_CONFIG_MAX = 0xFF,          /* الـ driver ممكن يضيف من 0x80 لـ 0xFF */
};
```

**لاحظ**: `PIN_CONFIG_END = 0x7F` يعني في مساحة من `0x80` لـ `0xFF` (128 قيمة) للـ vendor-specific configs. الـ driver بيحددها في `pctldev->desc->custom_params`.

---

### الـ `struct pinconf_ops` — عقد الـ Driver

ده الـ interface اللي كل driver لازم يوفّره:

```c
struct pinconf_ops {
#ifdef CONFIG_GENERIC_PINCONF
    bool is_generic;  /* يقول للـ core إن الـ driver بيستخدم الـ generic system */
#endif

    /* قراءة config من pin واحد */
    int (*pin_config_get)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *config);

    /* كتابة config على pin واحد - بياخد array من configs */
    int (*pin_config_set)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *configs,
                          unsigned int num_configs);

    /* قراءة config من group كامل من pins */
    int (*pin_config_group_get)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *config);

    /* كتابة config على group كامل */
    int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *configs,
                                unsigned int num_configs);

    /* debugfs hooks اختيارية */
    void (*pin_config_dbg_show)(struct pinctrl_dev *, struct seq_file *, unsigned int);
    void (*pin_config_group_dbg_show)(struct pinctrl_dev *, struct seq_file *, unsigned int);
    void (*pin_config_config_dbg_show)(struct pinctrl_dev *, struct seq_file *, unsigned long);
};
```

**مثال تطبيق driver (NXP i.MX style)**:
```c
static int imx_pinconf_set(struct pinctrl_dev *pctldev,
                           unsigned int pin,
                           unsigned long *configs,
                           unsigned int num_configs)
{
    int i;
    for (i = 0; i < num_configs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        u32 arg = pinconf_to_config_argument(configs[i]);

        switch (param) {
        case PIN_CONFIG_BIAS_PULL_UP:
            /* اكتب في الـ IOMUXC register */
            imx_write_reg(pin, PULL_UP_ENABLE | (arg ? PULL_VALUE(arg) : 0));
            break;
        case PIN_CONFIG_DRIVE_STRENGTH:
            imx_write_reg(pin, DSE_FIELD(arg));
            break;
        /* ... */
        }
    }
    return 0;
}
```

---

### الـ Data Flow: من الـ Device Tree للـ Hardware Register

#### الخطوة 1: الـ DT Node

```dts
/* Device Tree في الـ SoC .dtsi */
&pinctrl {
    uart0_pins: uart0-pins {
        pins = "PA0", "PA1";
        function = "uart0";
        bias-pull-up;              /* PIN_CONFIG_BIAS_PULL_UP */
        drive-strength = <8>;      /* PIN_CONFIG_DRIVE_STRENGTH, arg=8mA */
        slew-rate = <1>;           /* PIN_CONFIG_SLEW_RATE, arg=1 */
    };
};
```

#### الخطوة 2: الـ Generic Parser — `pinconf_generic_parse_dt_config()`

الكود بيمشي على الـ `dt_params[]` table:

```c
static const struct pinconf_generic_params dt_params[] = {
    { "bias-pull-up",    PIN_CONFIG_BIAS_PULL_UP,    1 },
    { "drive-strength",  PIN_CONFIG_DRIVE_STRENGTH,  0 },
    { "slew-rate",       PIN_CONFIG_SLEW_RATE,       0 },
    /* ... */
};
```

بيقرأ كل property من الـ DT ويحولها لـ packed `unsigned long`:
- `bias-pull-up` → `PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 1)` = `0x00000105`
- `drive-strength = <8>` → `PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 8)` = `0x00000909`
- `slew-rate = <1>` → `PIN_CONF_PACKED(PIN_CONFIG_SLEW_RATE, 1)` = `0x00000129`

#### الخطوة 3: الـ `pinctrl_map` و `pinctrl_setting`

```c
/* الـ map: static description من الـ DT/board */
struct pinctrl_map {
    const char *dev_name;       /* "2010000.uart" */
    const char *name;           /* "default" */
    enum pinctrl_map_type type; /* PIN_MAP_TYPE_CONFIGS_PIN */
    const char *ctrl_dev_name;  /* "pinctrl" */
    union {
        struct pinctrl_map_configs configs; /* يحتوي على الـ array */
    } data;
};

/* الـ setting: runtime resolved version */
struct pinctrl_setting {
    struct pinctrl_dev *pctldev;
    enum pinctrl_map_type type;
    struct {
        unsigned int group_or_pin; /* pin number بدل الاسم */
        unsigned long *configs;
        unsigned int num_configs;
    } data.configs;
};
```

الفرق بين الـ `map` والـ `setting`: الـ map بتستخدم **أسماء** للـ pins والـ groups. الـ `pinconf_map_to_setting()` بتحولها لـ **أرقام** runtime.

#### الخطوة 4: `pinconf_apply_setting()`

```c
int pinconf_apply_setting(const struct pinctrl_setting *setting)
{
    struct pinctrl_dev *pctldev = setting->pctldev;
    const struct pinconf_ops *ops = pctldev->desc->confops;

    switch (setting->type) {
    case PIN_MAP_TYPE_CONFIGS_PIN:
        /* يبعت الـ configs array للـ driver */
        return ops->pin_config_set(pctldev,
                                   setting->data.configs.group_or_pin,
                                   setting->data.configs.configs,
                                   setting->data.configs.num_configs);

    case PIN_MAP_TYPE_CONFIGS_GROUP:
        /* نفس الكلام بس للـ group كامل */
        return ops->pin_config_group_set(pctldev,
                                         setting->data.configs.group_or_pin,
                                         setting->data.configs.configs,
                                         setting->data.configs.num_configs);
    }
}
```

---

### العلاقة بين الـ Structs

```
struct pinctrl_desc
    │
    ├── .confops ──────────────► struct pinconf_ops
    │                                ├── pin_config_get()
    │                                ├── pin_config_set()
    │                                ├── pin_config_group_get()
    │                                └── pin_config_group_set()
    │
    ├── .custom_params ────────► struct pinconf_generic_params[]
    │                                ├── .property  "vendor-special-drive"
    │                                ├── .param     PIN_CONFIG_END + 1
    │                                └── .default_value
    │
    └── .custom_conf_items ────► struct pin_config_item[]
                                     (للـ debugfs display)


struct pinctrl_map
    ├── .dev_name   "uart@2010000"
    ├── .name       "default"
    ├── .type       PIN_MAP_TYPE_CONFIGS_PIN
    └── .data.configs
            ├── .group_or_pin  "PA0"  (string)
            ├── .configs       [0x105, 0x909, 0x129]
            └── .num_configs   3

           pinconf_map_to_setting()
                    │
                    ▼

struct pinctrl_setting
    ├── .pctldev    → struct pinctrl_dev*
    ├── .type       PIN_MAP_TYPE_CONFIGS_PIN
    └── .data.configs
            ├── .group_or_pin  42  (resolved pin number)
            ├── .configs       [0x105, 0x909, 0x129]
            └── .num_configs   3
```

---

### الـ Generic vs. Custom Parameters

الـ pinconf framework بيدعم نوعين من الـ parameters:

#### 1. Standard Generic Parameters
معرّفة في `pinconf-generic.h` وموثّقة في `dt_params[]`. أي driver بيقول `is_generic = true` هيتعامل معاها تلقائيًا.

#### 2. Vendor-Specific Parameters
```c
/* الـ driver بيسجّل custom params */
static const struct pinconf_generic_params imx_custom_params[] = {
    { "fsl,drive-open-drain", IMX_CONFIG_OPEN_DRAIN, 0 },
    { "fsl,hysteresis",       IMX_CONFIG_HYS, 0 },
};

static struct pinctrl_desc imx_pctrl_desc = {
    .custom_params     = imx_custom_params,
    .num_custom_params = ARRAY_SIZE(imx_custom_params),
    .custom_conf_items = imx_conf_items,  /* للـ debugfs */
};
```

**الـ core** في `pinconf_generic_parse_dt_config()` بيمشي على الـ standard params الأول، وبعدين على الـ custom params للـ driver. الكل بيتحول لنفس الـ packed format.

---

### الـ Debugfs Integration

الـ pinconf بيوفّر ملفين في `/sys/kernel/debug/pinctrl/<device>/`:

```
/sys/kernel/debug/pinctrl/
└── 2000000.pinctrl/
    ├── pinconf-pins       ← config لكل pin منفرد
    └── pinconf-groups     ← config لكل group
```

```bash
# مثال عملي على Raspberry Pi / i.MX board
cat /sys/kernel/debug/pinctrl/20e0000.iomuxc/pinconf-pins
# Pin config settings per pin
# Format: pin (name): configs
# pin 0 (MX6Q_PAD_SD2_CLK): input bias pull up (100000 ohms), output drive strength (40 mA)
# pin 1 (MX6Q_PAD_SD2_CMD): input bias pull up (100000 ohms)
```

الـ `pinconf_generic_dump_pins()` بتعمل loop على كل الـ standard params وبتقرأ كل واحدة من الـ hardware فعلًا، وبتطبعها بشكل human-readable:

```c
static const struct pin_config_item conf_items[] = {
    PCONFDUMP(PIN_CONFIG_BIAS_PULL_UP, "input bias pull up", "ohms", true),
    PCONFDUMP(PIN_CONFIG_DRIVE_STRENGTH, "output drive strength", "mA", true),
    /* ... */
};
```

---

### ما الـ pinconf بيملكه وما بيفوّضه

| المسؤولية | pinconf Core يملكها | Driver يقوم بيها |
|-----------|--------------------|--------------------|
| تعريف لغة الـ configs الموحدة (`enum pin_config_param`) | نعم | لا |
| parsing الـ Device Tree properties لـ packed values | نعم (generic) | ممكن يضيف custom props |
| التحقق من وجود الـ ops المطلوبة | نعم (`pinconf_check_ops`) | لا |
| تحويل الـ map من أسماء لأرقام | نعم (`pinconf_map_to_setting`) | لا |
| تطبيق الـ config فعلًا على الـ hardware | لا | نعم (`pin_config_set`) |
| معرفة تفاصيل الـ registers | لا | نعم (vendor-specific) |
| debugfs standard display | نعم | ممكن يضيف custom display |
| الـ custom vendor params | يستوعبها | يعرّفها |

---

### subsystems تانية مرتبطة

- **pinctrl core** (`core.c`): الـ framework الأب — بيدير الـ `pinctrl_dev`, الـ states, والـ maps. الـ pinconf هو sub-module جوّاه.
- **pinmux** (`pinmux.c`): أخ الـ pinconf — بيتعامل مع الـ `MUX_GROUP` settings (تحديد وظيفة الـ pin: UART أم SPI أم GPIO). الاتنين بيشتغلوا جنب بعض لأن الـ pin الواحد ليه mux function وليه electrical config.
- **Device Tree / OF**: الـ `pinconf_generic_parse_dt_config()` بيعتمد عليه لقراءة الـ properties من الـ DT nodes.
- **debugfs**: الـ `seq_file` interface بيوفّر الـ human-readable output للـ pin states.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Enums والـ Config Options — Cheatsheet

#### `enum pinctrl_map_type` — نوع الـ mapping entry

| القيمة | المعنى |
|--------|--------|
| `PIN_MAP_TYPE_INVALID` | قيمة غلط / placeholder |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمي بدون عمليات حقيقية |
| `PIN_MAP_TYPE_MUX_GROUP` | إعداد الـ mux لمجموعة pins |
| `PIN_MAP_TYPE_CONFIGS_PIN` | إعداد configuration لـ pin واحد |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | إعداد configuration لمجموعة pins |

#### `enum pin_config_param` — باراميترات الـ pin configuration

| الباراميتر | المعنى | الـ argument |
|-----------|--------|-------------|
| `PIN_CONFIG_BIAS_DISABLE` | إيقاف كل الـ bias | يُتجاهل |
| `PIN_CONFIG_BIAS_HIGH_IMPEDANCE` | وضع high-Z / tristate | يُتجاهل |
| `PIN_CONFIG_BIAS_BUS_HOLD` | bus keeper — يحافظ على آخر قيمة | يُتجاهل |
| `PIN_CONFIG_BIAS_PULL_UP` | pull-up لـ VDD | قيمة بالأوم، أو 1 لتفعيل |
| `PIN_CONFIG_BIAS_PULL_DOWN` | pull-down لـ GND | قيمة بالأوم، أو 1 لتفعيل |
| `PIN_CONFIG_BIAS_PULL_PIN_DEFAULT` | pull حسب الـ hardware default | != 0 لتفعيل |
| `PIN_CONFIG_DRIVE_PUSH_PULL` | توصيل push-pull نشط | يُتجاهل |
| `PIN_CONFIG_DRIVE_OPEN_DRAIN` | open-drain output | يُتجاهل |
| `PIN_CONFIG_DRIVE_OPEN_SOURCE` | open-source (open-emitter) output | يُتجاهل |
| `PIN_CONFIG_DRIVE_STRENGTH` | تيار الـ sink/source | بالـ mA |
| `PIN_CONFIG_DRIVE_STRENGTH_UA` | تيار الـ sink/source | بالـ µA |
| `PIN_CONFIG_INPUT_ENABLE` | تفعيل الـ input buffer | 1=تفعيل، 0=إيقاف |
| `PIN_CONFIG_INPUT_DEBOUNCE` | debounce للإدخال | بالـ µs، 0=إيقاف |
| `PIN_CONFIG_INPUT_SCHMITT` | schmitt-trigger مع hysteresis مخصص | custom threshold |
| `PIN_CONFIG_INPUT_SCHMITT_ENABLE` | تفعيل/إيقاف schmitt-trigger | 1=تفعيل |
| `PIN_CONFIG_INPUT_SCHMITT_UV` | schmitt-trigger بالـ µV | بالـ µV |
| `PIN_CONFIG_OUTPUT_ENABLE` | تفعيل output buffer | 1=تفعيل |
| `PIN_CONFIG_OUTPUT_IMPEDANCE_OHMS` | impedance الـ output | بالأوم |
| `PIN_CONFIG_LEVEL` | تعيين/قراءة مستوى المنطق | 1=high، 0=low |
| `PIN_CONFIG_MODE_LOW_POWER` | وضع توفير الطاقة | 1=تفعيل |
| `PIN_CONFIG_MODE_PWM` | وضع PWM | custom |
| `PIN_CONFIG_SLEW_RATE` | معدل التغيير | custom |
| `PIN_CONFIG_SKEW_DELAY` | تأخير الـ clock skew / latch | custom |
| `PIN_CONFIG_SKEW_DELAY_INPUT_PS` | skew للـ input فقط | بالـ ps |
| `PIN_CONFIG_SKEW_DELAY_OUTPUT_PS` | latch delay للـ output فقط | بالـ ps |
| `PIN_CONFIG_POWER_SOURCE` | مصدر الجهد | custom |
| `PIN_CONFIG_PERSIST_STATE` | الاحتفاظ بالحالة عبر sleep/reset | يُتجاهل |
| `PIN_CONFIG_SLEEP_HARDWARE_STATE` | علامة أن الـ state للـ sleep | يُتجاهل |
| `PIN_CONFIG_END` | نهاية القيم القياسية | `= 0x7F` |
| `PIN_CONFIG_MAX` | أقصى قيمة ممكنة | `= 0xFF` |

#### تنسيق تعبئة الـ config — `PIN_CONF_PACKED`

```
unsigned long config:
┌────────────────────────────┬──────────────┐
│   argument [31:8] (24 bit) │ param [7:0]  │
└────────────────────────────┴──────────────┘

#define PIN_CONF_PACKED(p, a)  ((a << 8) | ((unsigned long)p & 0xffUL))
```

الـ lower 8 bits = الـ `pin_config_param`، والـ upper 24 bits = القيمة.

#### Kconfig options المتحكمة في الكود

| Option | الأثر |
|--------|-------|
| `CONFIG_PINCONF` | يفعّل كل الـ pinconfig core، بدونه كل الدوال stubs |
| `CONFIG_GENERIC_PINCONF` | يفعّل الـ generic config interface + الـ `is_generic` flag |
| `CONFIG_DEBUG_FS` | يفعّل debugfs entries للـ pins والـ groups |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | يفعّل `pin_group_tree` و `group_desc` |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | يفعّل `pin_function_tree` |
| `CONFIG_PINMUX` | يضيف fields الـ mux في `pin_desc` |
| `CONFIG_OF` | يفعّل DT parsing للـ configs |

---

### 1. الـ Structs المهمة

#### `struct pinconf_ops`
**الغرض:** جدول العمليات الذي ينفّذه الـ driver لإدارة الـ pin configuration. الـ core لا يعرف شيئاً عن الـ hardware — كل شيء يمر من هنا.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `is_generic` | `bool` | (فقط مع `CONFIG_GENERIC_PINCONF`) يخبر الـ core أن الـ driver يستخدم الـ generic interface |
| `pin_config_get` | function pointer | قراءة config لـ pin واحد، يرجع `unsigned long *config` |
| `pin_config_set` | function pointer | ضبط config لـ pin واحد، يأخذ array من configs |
| `pin_config_group_get` | function pointer | قراءة config لمجموعة بالكامل |
| `pin_config_group_set` | function pointer | ضبط config لمجموعة بالكامل |
| `pin_config_dbg_show` | function pointer | debugfs: عرض info لـ pin معين |
| `pin_config_group_dbg_show` | function pointer | debugfs: عرض info لـ group معين |
| `pin_config_config_dbg_show` | function pointer | debugfs: فك تشفير وعرض قيمة config واحدة |

الـ `pin_config_set` و `pin_config_group_set` **يجب** أن يكون واحد منهما موجوداً على الأقل — `pinconf_check_ops()` يتحقق من ذلك.

---

#### `struct pinctrl_dev`
**الغرض:** تمثيل الـ pin controller كـ device في الـ kernel. الكيان المركزي الذي يربط كل شيء.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `node` | `struct list_head` | ربط الـ controller في القائمة العالمية |
| `desc` | `const struct pinctrl_desc *` | الـ descriptor الذي قدّمه الـ driver — يحتوي `confops` |
| `pin_desc_tree` | `struct radix_tree_root` | lookup سريع لـ `pin_desc` بالـ pin number |
| `pin_group_tree` | `struct radix_tree_root` | (optional) شجرة الـ groups |
| `pin_function_tree` | `struct radix_tree_root` | (optional) شجرة الـ functions |
| `gpio_ranges` | `struct list_head` | نطاقات الـ GPIO المرتبطة |
| `dev` | `struct device *` | الـ device الأصلي |
| `owner` | `struct module *` | للـ refcounting |
| `driver_data` | `void *` | بيانات خاصة بالـ driver |
| `p` | `struct pinctrl *` | الـ pinctrl handle للـ hog |
| `hog_default` | `struct pinctrl_state *` | الـ state الافتراضي المحجوز |
| `hog_sleep` | `struct pinctrl_state *` | الـ sleep state المحجوز |
| `mutex` | `struct mutex` | الـ lock الرئيسي لحماية العمليات |
| `device_root` | `struct dentry *` | جذر الـ debugfs لهذا الـ controller |

---

#### `struct pinctrl_desc`
**الغرض:** الوصف الثابت للـ pin controller الذي يملأه الـ driver وقت التسجيل.

الحقل المهم هنا في سياق الـ pinconf:

```c
struct pinctrl_desc {
    const char *name;
    const struct pinctrl_pin_desc *pins;  /* array of pins */
    unsigned int npins;
    const struct pinctrl_ops *pctlops;    /* pin control ops */
    const struct pinmux_ops *pmxops;      /* pinmux ops */
    const struct pinconf_ops *confops;    /* pinconfig ops  ← المهم هنا */
    struct module *owner;
};
```

الـ `confops` هو الرابط بين الـ core وبين implementation الـ driver.

---

#### `struct pinctrl_map`
**الغرض:** مدخل في جدول الـ mapping — يصف "ماذا يريد هذا الـ device من الـ pin controller".

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `dev_name` | `const char *` | اسم الـ device صاحب الطلب |
| `name` | `const char *` | اسم الـ state مثل "default" أو "sleep" |
| `type` | `enum pinctrl_map_type` | نوع الـ mapping |
| `ctrl_dev_name` | `const char *` | اسم الـ pin controller المسؤول |
| `data.mux` | `struct pinctrl_map_mux` | بيانات الـ mux |
| `data.configs` | `struct pinctrl_map_configs` | بيانات الـ config |

---

#### `struct pinctrl_map_configs`
**الغرض:** البيانات الخاصة بـ mapping entry من نوع `CONFIGS_PIN` أو `CONFIGS_GROUP`.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `group_or_pin` | `const char *` | اسم الـ pin أو الـ group |
| `configs` | `unsigned long *` | array من القيم المعبأة بـ `PIN_CONF_PACKED` |
| `num_configs` | `unsigned int` | عدد العناصر في الـ array |

---

#### `struct pinctrl_setting`
**الغرض:** النسخة المُحوَّلة من `pinctrl_map` بعد الـ resolution — تحتوي على الـ IDs الرقمية بدل الأسماء.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `node` | `struct list_head` | ربط الـ setting في قائمة الـ state |
| `type` | `enum pinctrl_map_type` | النوع (configs_pin أو configs_group) |
| `pctldev` | `struct pinctrl_dev *` | الـ controller المسؤول عن تطبيق هذا الـ setting |
| `dev_name` | `const char *` | اسم الـ device صاحب الطلب |
| `data.configs` | `struct pinctrl_setting_configs` | البيانات المحوّلة |
| `data.mux` | `struct pinctrl_setting_mux` | بيانات الـ mux إذا كان النوع mux |

---

#### `struct pinctrl_setting_configs`
**الغرض:** نسخة resolved من `pinctrl_map_configs` — الأسماء أصبحت أرقام selectors.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `group_or_pin` | `unsigned int` | رقم الـ pin ID أو الـ group selector |
| `configs` | `unsigned long *` | نفس pointer الـ array من الـ map (لا يُنسخ) |
| `num_configs` | `unsigned int` | عدد العناصر |

---

#### `struct pinctrl`
**الغرض:** الـ handle الممنوح لأي device يطلب pinctrl عبر `pinctrl_get()`.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `node` | `struct list_head` | ربط في القائمة العالمية |
| `dev` | `struct device *` | الـ device المالك |
| `states` | `struct list_head` | قائمة الـ states المتاحة |
| `state` | `struct pinctrl_state *` | الـ state الحالي المُفعَّل |
| `dt_maps` | `struct list_head` | maps المحللة من DT |
| `users` | `struct kref` | reference count |

---

#### `struct pinctrl_state`
**الغرض:** يمثل حالة مُسمَّاة (مثل "default"، "sleep") — تحتوي على قائمة settings.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `node` | `struct list_head` | ربط في `pinctrl->states` |
| `name` | `const char *` | اسم الحالة |
| `settings` | `struct list_head` | قائمة `pinctrl_setting` |

---

#### `struct pin_desc`
**الغرض:** وصف كل pin فيزيائي — مخزّن في الـ radix tree داخل `pinctrl_dev`.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `pctldev` | `struct pinctrl_dev *` | الـ controller المالك |
| `name` | `const char *` | اسم الـ pin ("GPIO_0" مثلاً) |
| `dynamic_name` | `bool` | هل الاسم مخصّص ديناميكياً |
| `drv_data` | `void *` | بيانات خاصة بالـ driver |
| `mux_usecount` | `unsigned int` | (مع PINMUX) عداد الاستخدام |
| `mux_owner` | `const char *` | (مع PINMUX) اسم المالك الحالي |
| `mux_setting` | pointer | (مع PINMUX) الـ mux setting الحالي |
| `gpio_owner` | `const char *` | (مع PINMUX) اسم الـ GPIO المالك |
| `mux_lock` | `struct mutex` | (مع PINMUX) lock الـ mux |

---

#### `struct pin_config_item`
**الغرض:** تعريف بند في جدول عرض الـ debugfs — لكل `pin_config_param` اسم وتنسيق عرض.

| الحقل | النوع | الشرح |
|-------|-------|-------|
| `param` | `enum pin_config_param` | الباراميتر |
| `display` | `const char *` | الاسم القابل للقراءة |
| `format` | `const char *` | تنسيق الطباعة |
| `has_arg` | `bool` | هل يملك قيمة argument |
| `values` | `const char * const *` | أسماء القيم الاختيارية |
| `num_values` | `size_t` | عدد القيم |

---

### 2. مخطط علاقات الـ Structs

```
                    ┌─────────────────────────────────┐
                    │         pinctrl_dev              │
                    │  ┌──────────────────────────┐   │
                    │  │ desc: *pinctrl_desc       │   │
                    │  │   └──confops:*pinconf_ops │   │
                    │  └──────────────────────────┘   │
                    │  pin_desc_tree (radix)           │
                    │       │                          │
                    │       ▼                          │
                    │  ┌─────────┐                     │
                    │  │pin_desc │ × N                 │
                    │  └─────────┘                     │
                    │  mutex                           │
                    └────────┬────────────────────────┘
                             │ pctldev pointer
                             │
          ┌──────────────────▼──────────────────────┐
          │            pinctrl_setting               │
          │  type = CONFIGS_PIN | CONFIGS_GROUP      │
          │  data.configs:                           │
          │    ┌──────────────────────────────────┐  │
          │    │ pinctrl_setting_configs           │  │
          │    │  group_or_pin: unsigned int       │  │
          │    │  configs: *unsigned long ─────────┼──┼──► [packed config values]
          │    │  num_configs: unsigned int        │  │     PIN_CONF_PACKED(param, arg)
          │    └──────────────────────────────────┘  │
          └──────────────┬──────────────────────────┘
                         │ list_head node
                         ▼
          ┌──────────────────────────────┐
          │      pinctrl_state           │
          │  name: "default" / "sleep"   │
          │  settings: list_head ────────┼──► [list of pinctrl_setting]
          └──────────────┬───────────────┘
                         │ list_head node
                         ▼
          ┌──────────────────────────────┐
          │         pinctrl              │
          │  dev: *device                │
          │  states: list_head ──────────┼──► [list of pinctrl_state]
          │  state:  *pinctrl_state      │    (current active state)
          │  users:  kref                │
          └──────────────────────────────┘

          ┌──────────────────────────────┐
          │       pinctrl_map            │  ← static board/DT data
          │  dev_name                    │
          │  name (state name)           │
          │  type                        │
          │  data.configs:               │
          │    ┌─────────────────────┐   │
          │    │ pinctrl_map_configs │   │
          │    │  group_or_pin: char*│   │
          │    │  configs: *ulong    │   │
          │    └─────────────────────┘   │
          └──────────────────────────────┘
                         │
                         │  pinconf_map_to_setting()
                         │  (name → numeric ID)
                         ▼
                 pinctrl_setting  (resolved)
```

---

### 3. دورة الحياة — Lifecycle Diagram

```
 ┌─────────────────────────────────────────────────────┐
 │  1. REGISTRATION (وقت تسجيل الـ driver)              │
 │                                                     │
 │   driver fills pinctrl_desc {                       │
 │       .confops = &my_pinconf_ops                    │
 │   }                                                 │
 │        │                                            │
 │        ▼                                            │
 │   pinctrl_register(desc, dev, drvdata)              │
 │        │                                            │
 │        ├──► alloc pinctrl_dev                       │
 │        ├──► pinconf_check_ops()                     │
 │        │        checks: pin_config_set OR           │
 │        │                pin_config_group_set exists  │
 │        ├──► init pin_desc_tree (radix)              │
 │        ├──► register pins → pin_desc per pin        │
 │        └──► debugfs: pinconf_init_device_debugfs()  │
 └─────────────────────────────────────────────────────┘
                         │
                         ▼
 ┌─────────────────────────────────────────────────────┐
 │  2. MAP REGISTRATION (وقت boot / DT parsing)         │
 │                                                     │
 │   pinctrl_register_mappings(maps, num_maps)         │
 │        │                                            │
 │        ├──► alloc pinctrl_maps                      │
 │        └──► pinconf_validate_map() لكل entry        │
 │                 checks: group_or_pin != NULL         │
 │                         configs != NULL              │
 │                         num_configs > 0              │
 └─────────────────────────────────────────────────────┘
                         │
                         ▼
 ┌─────────────────────────────────────────────────────┐
 │  3. PINCTRL GET (device driver calls pinctrl_get)    │
 │                                                     │
 │   pinctrl_get(dev)                                  │
 │        │                                            │
 │        ├──► alloc pinctrl handle                    │
 │        ├──► for each matching map entry:            │
 │        │       pinconf_map_to_setting()             │
 │        │           resolves name → numeric pin ID   │
 │        │           copies configs pointer           │
 │        └──► returns pinctrl handle                  │
 └─────────────────────────────────────────────────────┘
                         │
                         ▼
 ┌─────────────────────────────────────────────────────┐
 │  4. STATE SELECT (pinctrl_select_state)              │
 │                                                     │
 │   pinctrl_select_state(p, state)                    │
 │        │                                            │
 │        └──► for each setting in state->settings:   │
 │                 pinconf_apply_setting(setting)      │
 │                     │                              │
 │                     ├─ CONFIGS_PIN:                │
 │                     │   ops->pin_config_set(...)   │
 │                     └─ CONFIGS_GROUP:              │
 │                         ops->pin_config_group_set  │
 └─────────────────────────────────────────────────────┘
                         │
                         ▼
 ┌─────────────────────────────────────────────────────┐
 │  5. TEARDOWN                                        │
 │                                                     │
 │   pinctrl_put(p)  →  kref_put                       │
 │        │                                            │
 │        └──► pinconf_free_setting()  (no-op currently)
 │                                                     │
 │   devm_pinctrl_put / device removal                 │
 │        │                                            │
 │        └──► pinctrl_unregister(pctldev)             │
 │                 remove debugfs entries              │
 │                 free pin_desc_tree                  │
 └─────────────────────────────────────────────────────┘
```

---

### 4. Call Flow Diagrams

#### 4.1 تطبيق الـ config على pin واحد

```
device driver calls pinctrl_select_state(p, "default")
  │
  └──► core iterates settings list
         │
         └──► pinconf_apply_setting(setting)   [pinconf.c:150]
                │
                ├── setting->type == PIN_MAP_TYPE_CONFIGS_PIN ?
                │       │
                │       └──► ops->pin_config_set(pctldev,
                │                               pin_id,
                │                               configs[],
                │                               num_configs)
                │                    │
                │                    └──► hardware register write
                │                         e.g. writel(val, reg_base + pin_offset)
                │
                └── setting->type == PIN_MAP_TYPE_CONFIGS_GROUP ?
                        │
                        └──► ops->pin_config_group_set(pctldev,
                                                       group_selector,
                                                       configs[],
                                                       num_configs)
                                     │
                                     └──► hardware: write same config
                                          to all pins in the group
```

#### 4.2 قراءة config لمجموعة عبر sysfs/debugfs

```
user reads /sys or debugfs "pinconf-groups"
  │
  └──► pinconf_groups_show(seq_file)   [pinconf.c:347]
         │
         ├── mutex_lock(&pctldev->mutex)
         ├── pctlops->get_groups_count(pctldev)
         │
         └──► for each group selector:
                pctlops->get_group_name(pctldev, selector)
                │
                └──► pinconf_dump_group(pctldev, s, selector, gname)
                       │
                       ├──► pinconf_generic_dump_pins()   [if GENERIC_PINCONF]
                       │       reads generic params and prints them
                       │
                       └──► ops->pin_config_group_dbg_show(pctldev, s, selector)
                               │
                               └──► driver formats and prints custom config
```

#### 4.3 `pin_config_group_get` — القراءة المباشرة

```
caller: pin_config_group_get("soc-pinctrl", "uart0-grp", &config)
  │
  ├──► get_pinctrl_dev_from_devname("soc-pinctrl")
  │       lookup in global pinctrl_dev list
  │
  ├──► mutex_lock(&pctldev->mutex)
  │
  ├──► pinctrl_get_group_selector(pctldev, "uart0-grp")
  │       returns integer selector (index)
  │
  └──► ops->pin_config_group_get(pctldev, selector, &config)
         │
         ├──── reads hardware register
         └──── encodes result: *config = PIN_CONF_PACKED(param, value)
```

#### 4.4 تحويل الـ Map إلى Setting

```
pinconf_map_to_setting(map, setting)   [pinconf.c:109]
  │
  ├── map->type == CONFIGS_PIN ?
  │       └──► pin_get_from_name(pctldev, map->data.configs.group_or_pin)
  │                   │
  │                   └──► radix_tree_lookup(pin_desc_tree, ...)
  │                         returns numeric pin ID
  │
  └── map->type == CONFIGS_GROUP ?
          └──► pinctrl_get_group_selector(pctldev, name)
                      │
  (in both cases)     └──► stores result in setting->data.configs.group_or_pin
                           copies configs pointer and num_configs
```

#### 4.5 التحقق من الـ ops عند التسجيل

```
pinctrl_register()
  │
  └──► pinconf_check_ops(pctldev)   [pinconf.c:27]
         │
         └── ops->pin_config_set == NULL
             AND ops->pin_config_group_set == NULL ?
                 │
                 YES → dev_err() + return -EINVAL
                 NO  → return 0  (OK)
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks المستخدمة

| الـ Lock | النوع | يحمي ماذا |
|---------|-------|----------|
| `pctldev->mutex` | `struct mutex` | كل عمليات الـ pin controller الواحد |
| `pinctrl_maps_mutex` | `struct mutex` (global) | القائمة العالمية لكل الـ maps |
| `pin_desc->mux_lock` | `struct mutex` | الـ mux state لكل pin على حدة |

#### أين يُستخدم كل lock في `pinconf.c`

**`pctldev->mutex`:**

```c
/* pin_config_group_get() */
mutex_lock(&pctldev->mutex);
    ops->pin_config_group_get(pctldev, selector, config);
mutex_unlock(&pctldev->mutex);

/* pinconf_pins_show() - debugfs */
mutex_lock(&pctldev->mutex);
    for each pin: pinconf_dump_pin(...)
mutex_unlock(&pctldev->mutex);
```

**`pinconf_apply_setting()`** — لا يأخذ الـ mutex مباشرة، لأن الـ caller (`pinctrl_select_state`) يمسك الـ lock قبل الاستدعاء.

#### ترتيب الـ Locking (Lock Ordering)

```
pinctrl_maps_mutex          ← الأعلى (global maps list)
      │
      ▼
pctldev->mutex              ← per-controller
      │
      ▼
pin_desc->mux_lock          ← per-pin (فقط للـ mux، ليس pinconf)
```

**القاعدة:** لا تأخذ `pinctrl_maps_mutex` وأنت ممسك `pctldev->mutex` — هذا يسبب deadlock.

#### ملاحظات مهمة على الـ Locking

- الـ `pin_config_get_for_pin()` **لا يأخذ الـ mutex** — المفروض الـ caller يكون ممسكه.
- الـ `pinconf_apply_setting()` **لا يأخذ الـ mutex** — الـ caller مسؤول.
- الـ `pin_config_group_get()` **يأخذ الـ mutex** لأنه public API يُستدعى من خارج الـ core.
- الـ debugfs callbacks (`pinconf_pins_show`, `pinconf_groups_show`) **يأخذان الـ mutex** لأنهما يُستدعيان من thread عشوائي.
- عمليات الـ `ops->pin_config_*` الفعلية تُنفَّذ **داخل** الـ mutex window — الـ driver لا يحتاج lock إضافي للوصول إلى الـ controller.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions (Cheatsheet)

#### Group 1: Validation & Ops Check

| Function | Signature Summary | الغرض |
|---|---|---|
| `pinconf_check_ops` | `(pctldev) → int` | يتحقق إن الـ driver عنده على الأقل `pin_config_set` أو `pin_config_group_set` |
| `pinconf_validate_map` | `(map, i) → int` | يتحقق إن الـ pinctrl map entry صحيحة وعندها pin/group وconfigs |

#### Group 2: Config Get

| Function | Signature Summary | الغرض |
|---|---|---|
| `pin_config_get_for_pin` | `(pctldev, pin, *config) → int` | يجيب config قيمة لـ pin معين بشكل مباشر عبر الـ driver ops |
| `pin_config_group_get` | `(dev_name, pin_group, *config) → int` | يجيب config لـ pin group كاملة باستخدام اسم الـ device والـ group |

#### Group 3: Map → Setting Lifecycle

| Function | Signature Summary | الغرض |
|---|---|---|
| `pinconf_map_to_setting` | `(map, setting) → int` | يحوّل الـ `pinctrl_map` entry لـ `pinctrl_setting` جاهز للتطبيق |
| `pinconf_free_setting` | `(setting) → void` | يحرر الموارد الخاصة بـ setting (حالياً no-op) |
| `pinconf_apply_setting` | `(setting) → int` | يطبّق الـ config الموجود في setting على الـ pin أو الـ group |

#### Group 4: Direct Config Set

| Function | Signature Summary | الغرض |
|---|---|---|
| `pinconf_set_config` | `(pctldev, pin, *configs, nconfigs) → int` | يطبّق configs مباشرة على pin بدون المرور بالـ setting lifecycle |

#### Group 5: DebugFS

| Function | Signature Summary | الغرض |
|---|---|---|
| `pinconf_show_config` | `(s, pctldev, *configs, num) → void` | static — يطبع raw config values في debugfs |
| `pinconf_show_map` | `(s, map) → void` | يطبع محتوى map entry في debugfs |
| `pinconf_show_setting` | `(s, setting) → void` | يطبع setting مع اسم الـ pin/group في debugfs |
| `pinconf_dump_pin` | `(pctldev, s, pin) → void` | static — يطبع config لـ pin واحد (generic + driver-specific) |
| `pinconf_dump_group` | `(pctldev, s, selector, gname) → void` | static — يطبع config لـ group (generic + driver-specific) |
| `pinconf_pins_show` | `(s, what) → int` | seq_file show callback لـ `pinconf-pins` debugfs file |
| `pinconf_groups_show` | `(s, what) → int` | seq_file show callback لـ `pinconf-groups` debugfs file |
| `pinconf_init_device_debugfs` | `(devroot, pctldev) → void` | ينشئ الـ debugfs files للـ device |

#### الـ `struct pinconf_ops` (Driver Interface)

| Member | النوع | الغرض |
|---|---|---|
| `is_generic` | `bool` | إشارة إن الـ driver يستخدم generic pinconf |
| `pin_config_get` | function pointer | اقرأ config لـ pin واحد |
| `pin_config_set` | function pointer | اكتب configs لـ pin واحد |
| `pin_config_group_get` | function pointer | اقرأ config لـ group كاملة |
| `pin_config_group_set` | function pointer | اكتب configs لـ group كاملة |
| `pin_config_dbg_show` | function pointer | debugfs display لـ pin |
| `pin_config_group_dbg_show` | function pointer | debugfs display لـ group |
| `pin_config_config_dbg_show` | function pointer | decode وعرض config value واحدة |

---

### Group 1: Validation & Ops Check

الـ functions دي بتتشغل وقت الـ registration والـ initialization عشان تتحقق إن الـ driver جاهز صح.

---

#### `pinconf_check_ops`

```c
int pinconf_check_ops(struct pinctrl_dev *pctldev);
```

بتتحقق إن الـ `pinconf_ops` struct اللي الـ driver سجّله فيه على الأقل واحدة من عمليتي الكتابة: إما `pin_config_set` أو `pin_config_group_set`. لو الاتنين مش موجودين، معناها الـ driver مش قادر يعمل أي config حقيقي وبيرجع error.

**Parameters:**
- `pctldev`: الـ `struct pinctrl_dev *` — الـ handle للـ pin controller المسجّل

**Return value:**
- `0` لو العمليات صحيحة
- `-EINVAL` لو الاتنين `pin_config_set` و`pin_config_group_set` مش موجودين

**Key details:**
- بيُستدعى من `pinctrl_register_one_dev()` في `core.c` وقت تسجيل الـ controller
- مفيش locking محتاج هنا — بيتشغل في initialization context قبل ما الـ device يبقى متاح

**Who calls it:** `pinctrl_register_one_dev()` في `drivers/pinctrl/core.c`

---

#### `pinconf_validate_map`

```c
int pinconf_validate_map(const struct pinctrl_map *map, int i);
```

بتتحقق إن map entry من نوع `PIN_MAP_TYPE_CONFIGS_*` صحيحة. بتشوف إن `group_or_pin` مش NULL وإن `configs` array موجود وعنده عدد صحيح.

**Parameters:**
- `map`: الـ `pinctrl_map` entry اللي هيتحقق منها
- `i`: index الـ map في الـ table — بيُستخدم فقط في رسائل الـ error

**Return value:**
- `0` لو صحيحة
- `-EINVAL` لو `group_or_pin` فارغ أو `num_configs` صفر أو `configs` pointer فارغ

**Key details:**
- بيُستدعى من `pinctrl_register_mappings()` في `core.c` على كل entry
- الـ `configs` pointer ومحتواه **لا يتحقق** من صحتهم هنا — ده مسؤولية الـ driver

**Who calls it:** `pinctrl_register_mappings()` في `core.c`

---

### Group 2: Config Get

الـ functions دي بتجيب config values من الـ hardware عبر الـ driver ops.

---

#### `pin_config_get_for_pin`

```c
int pin_config_get_for_pin(struct pinctrl_dev *pctldev, unsigned int pin,
                           unsigned long *config);
```

بتجيب config قيمة لـ pin محدد بشكل مباشر عن طريق استدعاء `ops->pin_config_get`. الـ `config` parameter بيعمل كـ in/out: الـ caller بيحط فيه الـ param type اللي عايزه وبيرجع بالقيمة.

**Parameters:**
- `pctldev`: الـ pin controller device
- `pin`: رقم الـ pin (global pin number في الـ controller)
- `config`: pointer لـ `unsigned long` — الـ format بتاعه هو `PIN_CONF_PACKED(param, value)` على طريقة generic pinconf

**Return value:**
- `0` + قيمة في `*config` عند النجاح
- `-ENOTSUPP` لو الـ driver مش بيدعم `pin_config_get`
- أي error code تاني من الـ driver نفسه

**Key details:**
- مفيش mutex هنا — المفروض الـ caller يمسك الـ `pctldev->mutex` لو محتاج
- مستخدمة من السياق الداخلي بعد ما الـ lock اتمسك فعلاً

**Who calls it:** `pinconf.c` internal callers، وكمان debugfs code في `core.c`

---

#### `pin_config_group_get`

```c
int pin_config_group_get(const char *dev_name, const char *pin_group,
                         unsigned long *config);
```

بتجيب config لـ pin group كاملة. بتدوّر على الـ controller بالاسم، بتقفل الـ mutex، بتحول اسم الـ group لـ selector، وبعدين بتستدعي `ops->pin_config_group_get`.

**Parameters:**
- `dev_name`: اسم الـ pin controller device (نفس اللي تم التسجيل بيه)
- `pin_group`: اسم الـ pin group
- `config`: نفس conventions الـ `pin_config_get_for_pin`

**Return value:**
- `0` + قيمة في `*config` عند النجاح
- `-EINVAL` لو الـ device مش موجود
- `-ENOTSUPP` لو `pin_config_group_get` مش موجودة في الـ driver
- error من `pinctrl_get_group_selector()` لو الـ group مش موجودة

**Key details:**
- **يمسك `pctldev->mutex`** — الوحيدة في الـ get functions اللي بتعمل كده صريح
- بيستخدم `get_pinctrl_dev_from_devname()` اللي بتدوّر في الـ global list
- الـ group selector هو index داخلي مش اسم — الـ name-to-selector translation بتحصل هنا

**Who calls it:** يمكن تُستدعى من driver-level code أو debugfs

**Pseudocode flow:**
```
pin_config_group_get(dev_name, pin_group, config):
    pctldev = get_pinctrl_dev_from_devname(dev_name)
    if !pctldev → return -EINVAL

    mutex_lock(pctldev->mutex)

    if !ops->pin_config_group_get → ret = -ENOTSUPP; goto unlock

    selector = pinctrl_get_group_selector(pctldev, pin_group)
    if selector < 0 → ret = selector; goto unlock

    ret = ops->pin_config_group_get(pctldev, selector, config)

unlock:
    mutex_unlock(pctldev->mutex)
    return ret
```

---

### Group 3: Map → Setting Lifecycle

الـ functions دي بتتعامل مع الـ pipeline من الـ static map table للـ runtime setting ولحد التطبيق على الـ hardware.

```
pinctrl_map (static table)
        │
        ▼
 pinconf_validate_map()      ← وقت تسجيل الـ mappings
        │
        ▼
 pinconf_map_to_setting()    ← وقت بناء الـ pinctrl state
        │
        ▼
 pinconf_apply_setting()     ← وقت activate الـ state
        │
        ▼
 pinconf_free_setting()      ← وقت تحرير الـ state (no-op حالياً)
```

---

#### `pinconf_map_to_setting`

```c
int pinconf_map_to_setting(const struct pinctrl_map *map,
                           struct pinctrl_setting *setting);
```

بتحوّل الـ `pinctrl_map` entry لـ `pinctrl_setting` قابل للتطبيق. الـ map بيستخدم string names للـ pins/groups، لكن الـ setting بيستخدم numeric selectors — الـ function دي بتعمل الـ name-to-number resolution دي.

**Parameters:**
- `map`: الـ source map entry (read-only)
- `setting`: الـ destination setting اللي هيتملّى — `pctldev` فيه المفروض يكون اتحدد مسبقاً

**Return value:**
- `0` عند النجاح
- `< 0` لو الـ pin/group مش موجود (error من `pin_get_from_name` أو `pinctrl_get_group_selector`)

**Key details:**
- `setting->data.configs.configs` بيشاور على نفس الـ array الموجود في الـ `map` — مش بيعمل copy — يعني الـ map لازم يفضل موجود طول الـ lifetime
- `group_or_pin` في الـ setting بتتحوّل من string index لـ numeric: pin number في حالة `PIN_MAP_TYPE_CONFIGS_PIN`، أو group selector في حالة `PIN_MAP_TYPE_CONFIGS_GROUP`

**Who calls it:** `pinctrl_map_to_setting()` في `core.c` كجزء من بناء الـ `pinctrl_state`

---

#### `pinconf_free_setting`

```c
void pinconf_free_setting(const struct pinctrl_setting *setting);
```

المفروض تحرر الموارد الخاصة بالـ pinconf setting. حالياً هي **no-op تماماً** — جسم الـ function فارغ. ده منطقي لأن `pinconf_map_to_setting` مش بيعمل allocation — بس بيعمل reference للـ map data الموجودة.

**Parameters:**
- `setting`: الـ setting اللي المفروض يتحرر (ignored حالياً)

**Return value:** لا يوجد (void)

**Key details:**
- موجودة كـ API placeholder — لو في المستقبل `pinconf_map_to_setting` اتغيرت وعملت allocation، هيتحرر هنا
- الـ stub في `pinconf.h` لما `CONFIG_PINCONF` مش محدد بيعمل نفس الشيء (no-op)

**Who calls it:** `pinctrl_free_setting()` في `core.c`

---

#### `pinconf_apply_setting`

```c
int pinconf_apply_setting(const struct pinctrl_setting *setting);
```

الـ function الأساسية اللي بتطبّق الـ config على الـ hardware فعلياً. بتشوف نوع الـ setting (pin أو group) وبتستدعي الـ driver op المناسبة.

**Parameters:**
- `setting`: الـ setting المحتوية على `pctldev` والـ `configs` array والـ `num_configs` ورقم الـ pin/group

**Return value:**
- `0` عند النجاح
- `-EINVAL` لو `confops` مش موجود أو الـ op المطلوبة مش موجودة
- error من الـ driver op نفسها

**Key details:**
- **لا تمسك locks** — المفروض الـ caller يمسك الـ `pctldev->mutex`
- عند الـ failure، بيطبع `dev_err` مع رقم الـ pin/group
- الـ `configs` array بيتبعت للـ driver كما هو — الـ driver مسؤول عن parse كل entry

**Who calls it:** `pinctrl_commit_state()` في `core.c` أثناء تفعيل state

**Pseudocode flow:**
```
pinconf_apply_setting(setting):
    pctldev = setting->pctldev
    ops = pctldev->desc->confops

    if !ops → return -EINVAL

    switch setting->type:
        case PIN_MAP_TYPE_CONFIGS_PIN:
            if !ops->pin_config_set → return -EINVAL
            ret = ops->pin_config_set(pctldev,
                                      setting->data.configs.group_or_pin,
                                      setting->data.configs.configs,
                                      setting->data.configs.num_configs)
            if ret < 0 → dev_err + return ret

        case PIN_MAP_TYPE_CONFIGS_GROUP:
            if !ops->pin_config_group_set → return -EINVAL
            ret = ops->pin_config_group_set(...)
            if ret < 0 → dev_err + return ret

        default: return -EINVAL

    return 0
```

---

### Group 4: Direct Config Set

---

#### `pinconf_set_config`

```c
int pinconf_set_config(struct pinctrl_dev *pctldev, unsigned int pin,
                       unsigned long *configs, size_t nconfigs);
```

shortcut مباشر لتطبيق configs على pin بدون المرور بالـ map/setting pipeline. بيستدعي `ops->pin_config_set` مباشرة.

**Parameters:**
- `pctldev`: الـ pin controller
- `pin`: رقم الـ pin
- `configs`: array من config values
- `nconfigs`: عدد الـ configs في الـ array

**Return value:**
- `0` عند النجاح
- `-ENOTSUPP` لو `confops` أو `pin_config_set` مش موجود
- error من الـ driver

**Key details:**
- مش بيمسك أي lock — الـ caller مسؤول
- بيُستخدم في paths مختلفة عن الـ standard state switching — مثلاً من GPIO subsystem أو hog pins

**Who calls it:** `pinctrl_gpio_direction()` و`pinctrl_gpio_set_config()` عبر `core.c`

---

### Group 5: DebugFS Functions

كل الـ functions دي محاطة بـ `#ifdef CONFIG_DEBUG_FS` — مش موجودة في production builds بدون debugfs.

---

#### `pinconf_show_config` (static)

```c
static void pinconf_show_config(struct seq_file *s, struct pinctrl_dev *pctldev,
                                unsigned long *configs, unsigned int num_configs);
```

helper داخلي بيطبع كل config value في الـ array. لو الـ driver عنده `pin_config_config_dbg_show`، بيستخدمها عشان تعمل decode للقيمة لكلام مفهوم. لو لأ، بيطبعها كـ hex.

**Parameters:**
- `s`: الـ seq_file output
- `pctldev`: الـ controller (ممكن يكون NULL)
- `configs`: array الـ config values
- `num_configs`: العدد

**Key details:**
- الـ output format: `"config XXXXXXXX\n"` أو الـ decoded string من الـ driver
- static — مش exported، بس بيتستدعى من `pinconf_show_map` و`pinconf_show_setting`

---

#### `pinconf_show_map`

```c
void pinconf_show_map(struct seq_file *s, const struct pinctrl_map *map);
```

بتطبع map entry كاملة في debugfs — نوعها (pin/group)، اسم الـ target، وكل الـ configs. بيدوّر على الـ controller من اسمه في الـ map.

**Parameters:**
- `s`: seq_file output
- `map`: الـ map entry اللي هتتطبع

**Who calls it:** `pinctrl_maps_show()` في `core.c` لـ `pinctrl-maps` debugfs file

---

#### `pinconf_show_setting`

```c
void pinconf_show_setting(struct seq_file *s,
                          const struct pinctrl_setting *setting);
```

بتطبع setting محلولة في debugfs — اسم الـ pin أو الـ group الفعلي (مش مجرد رقم) وكل الـ configs. الـ pin/group names بتتجيب من الـ `pin_desc` أو من `pctlops->get_group_name`.

**Parameters:**
- `s`: seq_file output
- `setting`: الـ setting اللي هتتطبع

**Key details:**
- على عكس `pinconf_show_map`، هنا بيستخدم الـ numeric selector اللي اتحلّ مسبقاً ويرجعله الاسم — ده أدق وأمين للـ runtime state

**Who calls it:** debugfs state display في `core.c`

---

#### `pinconf_dump_pin` (static)

```c
static void pinconf_dump_pin(struct pinctrl_dev *pctldev,
                             struct seq_file *s, int pin);
```

بتطبع config لـ pin واحد في سياق الـ debugfs dump. بتستدعي `pinconf_generic_dump_pins` أولاً (لـ generic pinconf drivers) وبعدين `ops->pin_config_dbg_show` لو موجودة.

**Parameters:**
- `pctldev`: الـ controller
- `s`: seq_file output
- `pin`: رقم الـ pin

**Key details:**
- الـ dual call pattern (generic ثم driver-specific) بيسمح بـ layered output
- `pinconf_generic_dump_pins` no-op لو `CONFIG_GENERIC_PINCONF` مش محدد

---

#### `pinconf_dump_group` (static)

```c
static void pinconf_dump_group(struct pinctrl_dev *pctldev,
                               struct seq_file *s, unsigned int selector,
                               const char *gname);
```

نفس pattern الـ `pinconf_dump_pin` بس للـ groups — بتستدعي generic dump وبعدين driver-specific `pin_config_group_dbg_show`.

**Parameters:**
- `pctldev`: الـ controller
- `s`: seq_file output
- `selector`: الـ group selector (index)
- `gname`: اسم الـ group (للـ generic dump)

---

#### `pinconf_pins_show`

```c
static int pinconf_pins_show(struct seq_file *s, void *what);
```

الـ `show` callback الخاص بـ `pinconf-pins` debugfs file. بيمشي على كل الـ pins المسجلة في الـ controller descriptor ويطبع config كل واحد.

**Parameters:**
- `s`: الـ seq_file — `s->private` فيها `pctldev`
- `what`: مش مستخدم

**Return value:** دايماً `0`

**Key details:**
- **بيمسك `pctldev->mutex`** طول الـ iteration — ده ممكن يبقى مدة طويلة لو الـ controller عنده pins كتير
- بيعمل `pin_desc_get()` لكل pin ويتخطى الـ pins اللي مش موجودة descriptors
- مستخدم مع `DEFINE_SHOW_ATTRIBUTE(pinconf_pins)` اللي بتوّله تلقائياً لـ `pinconf_pins_fops`

---

#### `pinconf_groups_show`

```c
static int pinconf_groups_show(struct seq_file *s, void *what);
```

نفس الـ pattern بس للـ groups — بيمشي على كل الـ groups بالـ `get_groups_count` ويطبع config كل group.

**Parameters:**
- `s`: الـ seq_file — `s->private` فيها `pctldev`
- `what`: مش مستخدم

**Return value:** دايماً `0`

**Key details:**
- مش بيمسك `mutex` صريح — ده يمكن يكون مقصود لأن الـ group list بتتقرأ بس ومش بتتغير بعد الـ registration
- بيستخدم `pctlops->get_groups_count` وليس حاجة من `pinconf_ops` — لأن الـ group list هي مسؤولية الـ pinctrl_ops مش الـ pinconf_ops

---

#### `pinconf_init_device_debugfs`

```c
void pinconf_init_device_debugfs(struct dentry *devroot,
                                 struct pinctrl_dev *pctldev);
```

بتنشئ الاتنين debugfs files الخاصة بالـ pinconf لكل device: `pinconf-pins` و`pinconf-groups`. بيستخدم `debugfs_create_file` اللي بتتجاهل الـ errors (kernel idiom).

**Parameters:**
- `devroot`: الـ dentry directory الخاص بالـ device في debugfs
- `pctldev`: بيتحط كـ `private_data` في الـ seq_file عشان الـ show callbacks تلاقيه

**Key details:**
- مسموح read-only (`0444`) — مفيش write callbacks
- الـ `pctldev` pointer بيعيش طول عمر الـ device — مأمون الـ reference منه في callbacks

**Who calls it:** `pinctrl_init_device_debugfs()` في `core.c` وقت تسجيل الـ controller

---

### الـ `struct pinconf_ops` بالتفصيل

```c
/* include/linux/pinctrl/pinconf.h */
struct pinconf_ops {
#ifdef CONFIG_GENERIC_PINCONF
    bool is_generic;   /* set true if using generic pinconf params */
#endif

    /* Read current config of a single pin */
    int (*pin_config_get)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *config);

    /* Write configs array to a single pin */
    int (*pin_config_set)(struct pinctrl_dev *pctldev,
                          unsigned int pin,
                          unsigned long *configs,
                          unsigned int num_configs);

    /* Read config of a whole group (by selector) */
    int (*pin_config_group_get)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *config);

    /* Write configs array to a whole group */
    int (*pin_config_group_set)(struct pinctrl_dev *pctldev,
                                unsigned int selector,
                                unsigned long *configs,
                                unsigned int num_configs);

    /* Optional debugfs per-pin display */
    void (*pin_config_dbg_show)(struct pinctrl_dev *pctldev,
                                struct seq_file *s,
                                unsigned int offset);

    /* Optional debugfs per-group display */
    void (*pin_config_group_dbg_show)(struct pinctrl_dev *pctldev,
                                      struct seq_file *s,
                                      unsigned int selector);

    /* Optional: decode a single config value to human-readable text */
    void (*pin_config_config_dbg_show)(struct pinctrl_dev *pctldev,
                                       struct seq_file *s,
                                       unsigned long config);
};
```

**قواعد الـ ops:**

| Op | مطلوب؟ | ملاحظات |
|---|---|---|
| `pin_config_set` أو `pin_config_group_set` | واحدة على الأقل **إجبارية** | يتحقق منها `pinconf_check_ops` |
| `pin_config_get` | اختياري | لو مش موجود → `-ENOTSUPP` |
| `pin_config_group_get` | اختياري | لو مش موجود → `-ENOTSUPP` |
| `pin_config_group_set` | اختياري | لو مش موجود والـ setting نوعه group → `-EINVAL` |
| `*_dbg_show` | اختياري | debugfs فقط |
| `pin_config_config_dbg_show` | اختياري | لو مش موجود → raw hex في debugfs |

**الـ config value encoding:**

الـ `unsigned long config` بيستخدم encoding يجمع الـ param type والـ value في قيمة واحدة. في generic pinconf:
```
config = PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 1)
       = (PIN_CONFIG_BIAS_PULL_UP << 16) | 1
```
كل driver حر يستخدم encoding مختلف لو `is_generic = false`.

---

### تدفق الـ Data الكامل (End-to-End)

```
DT/board-file
    │  pinctrl_map[] static table
    │  { .name = "default",
    │    .type = PIN_MAP_TYPE_CONFIGS_PIN,
    │    .data.configs = { "UART0_TX", configs[], 2 } }
    │
    ▼
pinconf_validate_map()     ← يتحقق من صحة الـ entry
    │
    ▼
pinconf_map_to_setting()   ← يحوّل "UART0_TX" → pin number 42
    │                          يحفظ pointer للـ configs array
    ▼
pinctrl_state activated
    │
    ▼
pinconf_apply_setting()    ← يستدعي ops->pin_config_set(pctldev, 42,
    │                                                    configs, 2)
    ▼
driver hardware write      ← يعمل write للـ registers الفعلية
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — الـ Entries المهمة

الـ pinconf بيسجّل entries في debugfs تحت مسار الـ pinctrl device. الـ `pinconf_init_device_debugfs()` بتعمل ملفين:

| Entry | المسار الكامل | الوظيفة |
|---|---|---|
| `pinconf-pins` | `/sys/kernel/debug/pinctrl/<dev>/pinconf-pins` | إظهار config كل pin على حدة |
| `pinconf-groups` | `/sys/kernel/debug/pinctrl/<dev>/pinconf-groups` | إظهار config كل group |

```bash
# اعرف اسم الـ pinctrl device الموجودة
ls /sys/kernel/debug/pinctrl/

# اقرأ config كل pin
cat /sys/kernel/debug/pinctrl/pinctrl@fe000000/pinconf-pins

# اقرأ config كل group
cat /sys/kernel/debug/pinctrl/pinctrl@fe000000/pinconf-groups
```

**مثال output من `pinconf-pins`:**
```
Pin config settings per pin
Format: pin (name): configs
pin 0 (GPIO0): config bias-pull-up
pin 1 (GPIO1): config drive-strength:4
pin 2 (GPIO2): config 00400001
```

لو الـ driver مش بيـimplément `pin_config_config_dbg_show`، الـ config بيتطبع كـhex raw (`%08lx`). القيمة دي هي الـ `unsigned long config` اللي بتحتوي على `PIN_CONF_PACKED(param, arg)`.

**مسارات debugfs إضافية (مستوى pinctrl core):**

```bash
# شايف كل الـ mappings المسجّلة
cat /sys/kernel/debug/pinctrl/pinctrl@fe000000/pinmux-pins

# شايف كل الـ pin states الخاصة بـ device معينة
cat /sys/kernel/debug/pinctrl/pinctrl@fe000000/pinmux-functions
```

---

#### 2. sysfs — الـ Entries المهمة

الـ pinconf نفسه ما بيضيفش entries في sysfs مباشرةً، لكن في معلومات مفيدة:

```bash
# معرفة الـ pinctrl devices المسجّلة
ls /sys/bus/platform/drivers/pinctrl*/

# driver info للـ device
cat /sys/bus/platform/devices/<dev>/driver/module/version

# GPIO character device (لو الـ pin مربوط بـ GPIO)
gpioinfo gpiochip0

# لو الـ pin عنده regulator أو clock binding
cat /sys/class/gpio/export  # legacy sysfs GPIO
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# شوف كل الـ events المتاحة للـ pinctrl
ls /sys/kernel/debug/tracing/events/pinctrl/

# فعّل كل الـ pinctrl events
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# أو events محددة
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_setting_apply/enable

# ابدأ التتبع
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل العملية اللي بتحتاج debug فيها
echo 0 > /sys/kernel/debug/tracing/tracing_on

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace
```

**تتبع function calls مباشرة على `pinconf_apply_setting`:**

```bash
# فعّل function tracer
echo function > /sys/kernel/debug/tracing/current_tracer
echo pinconf_apply_setting > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
# شغّل الـ test ثم اقرأ
cat /sys/kernel/debug/tracing/trace
```

---

#### 4. printk والـ Dynamic Debug

الـ `pinconf.c` بيستخدم `pr_fmt` prefix هو `"pinconfig core: "`. يعني كل `pr_err` / `pr_dbg` بيبدأ بيه.

```bash
# فعّل dynamic debug لكل ملفات pinctrl
echo 'file drivers/pinctrl/* +p' > /sys/kernel/debug/dynamic_debug/control

# أو لـ pinconf.c بالتحديد
echo 'file drivers/pinctrl/pinconf.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع line numbers و function names
echo 'file drivers/pinctrl/pinconf.c +pflm' > /sys/kernel/debug/dynamic_debug/control

# تحقق من الـ dynamic debug المفعّلة
cat /sys/kernel/debug/dynamic_debug/control | grep pinconf
```

**الرسائل اللي هتظهر بعد تفعيل dynamic debug:**
```
[  123.456] pinconfig core: cannot get pin configuration, .pin_config_get missing in driver
[  123.457] pinconfig core: cannot get configuration for pin group, missing group config get function in driver
```

**ضبط loglevel:**
```bash
# اعرض كل الـ kernel messages في real-time
dmesg -w | grep -i "pinconf\|pinctrl\|pinconfig"

# رفع loglevel لـ kernel console
echo 8 > /proc/sys/kernel/printk
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_PINCONF` | تفعيل subsystem الـ pinconf أصلاً |
| `CONFIG_DEBUG_FS` | لازم يكون مفعّل عشان `pinconf-pins` و`pinconf-groups` تظهر |
| `CONFIG_GENERIC_PINCONF` | generic pin config — بيوفر `pinconf_generic_dump_pins` |
| `CONFIG_GENERIC_PINCTRL` | generic pinctrl — بيوفر `pinctrl_generic_pins_function_dt_node_to_map` |
| `CONFIG_DEBUG_PINCTRL` | (لو موجود في distro) بيزيد verbosity في pinctrl core |
| `CONFIG_PROVE_LOCKING` | يكتشف mutex deadlocks في `pctldev->mutex` |
| `CONFIG_LOCKDEP` | lock dependency tracking |
| `CONFIG_DYNAMIC_DEBUG` | لازم يكون مفعّل عشان `+p` flags تشتغل |

```bash
# تحقق من الـ config في kernel جارٍ
grep -E "CONFIG_PINCONF|CONFIG_DEBUG_FS|CONFIG_GENERIC_PINCONF|CONFIG_DYNAMIC_DEBUG" /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**أداة `pinctrl` في userspace (من package `libgpiod` أو kernel tools):**

```bash
# لو مفيش أداة جاهزة، استخدم debugfs مباشرة
# أو استخدم script بسيط يقرأ كل الـ entries

for dev in /sys/kernel/debug/pinctrl/*/; do
    echo "=== $dev ==="
    echo "-- pinconf-pins --"
    cat "${dev}pinconf-pins" 2>/dev/null
    echo "-- pinconf-groups --"
    cat "${dev}pinconf-groups" 2>/dev/null
done
```

**استخدام `gpio-tools` لاختبار الـ pin state:**

```bash
# اقرأ حالة pin معين
gpioget gpiochip0 5

# اكتب على pin
gpioset gpiochip0 5=1
```

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|---|---|---|
| `pinconf has to be able to set a pins config` | الـ driver سجّل `confops` لكن لا `pin_config_set` ولا `pin_config_group_set` موجودة | أضف واحدة على الأقل من الدالتين في `pinconf_ops` |
| `failed to register map %s (%d): no group/pin given` | الـ `pinctrl_map` اللي اتسجلت فيها `data.configs.group_or_pin == NULL` | تحقق من صحة الـ pinctrl mapping table في الـ driver أو DT |
| `failed to register map %s (%d): no configs given` | الـ `num_configs == 0` أو `configs == NULL` | تأكد إن الـ config array مش فاضية وبتتمرر صح |
| `could not map pin config for "%s"` | الـ pin name مش موجود في الـ pinctrl descriptor | تحقق من إن الاسم في الـ map مطابق لـ `pin_desc->name` |
| `could not map group config for "%s"` | الـ group name مش موجود | تحقق من أسماء الـ groups في `pctlops->get_group_name` |
| `missing confops` | الـ `pctldev->desc->confops == NULL` وقت `pinconf_apply_setting` | الـ driver لازم يسجّل `confops` في `pinctrl_desc` |
| `missing pin_config_set op` | الـ setting من نوع `CONFIGS_PIN` لكن `ops->pin_config_set == NULL` | نفّذ `pin_config_set` في الـ driver |
| `missing pin_config_group_set op` | نفس الفكرة لـ group setting | نفّذ `pin_config_group_set` |
| `pin_config_set op failed for pin %d` | الـ driver رجّع error من `pin_config_set` | راجع الـ driver implementation وتحقق من الـ hardware register |
| `pin_config_group_set op failed for group %d` | نفس الفكرة لـ group | راجع الـ driver implementation |
| `cannot get pin configuration, .pin_config_get missing in driver` | طلب `pin_config_get` والـ driver مش بيدعمه | إما نفّذ الـ callback أو تجنب استدعاء `pin_config_get_for_pin` |

---

#### 8. نقاط استراتيجية لـ `dump_stack()` و`WARN_ON()`

```c
int pinconf_apply_setting(const struct pinctrl_setting *setting)
{
    struct pinctrl_dev *pctldev = setting->pctldev;
    const struct pinconf_ops *ops = pctldev->desc->confops;
    int ret;

    /* نقطة 1: تحقق إن الـ ops مش NULL قبل الاستخدام */
    if (WARN_ON(!ops)) {
        dump_stack();  /* من هنا اتسمّيت الدالة؟ */
        return -EINVAL;
    }

    switch (setting->type) {
    case PIN_MAP_TYPE_CONFIGS_PIN:
        /* نقطة 2: الـ pin number معقول؟ */
        WARN_ON(setting->data.configs.group_or_pin >= pctldev->desc->npins);

        ret = ops->pin_config_set(pctldev,
                setting->data.configs.group_or_pin,
                setting->data.configs.configs,
                setting->data.configs.num_configs);

        /* نقطة 3: الـ driver رجّع error غريب */
        if (ret < 0 && ret != -ENOTSUPP)
            WARN_ON(1);  /* يطبع stack trace + الـ error code */
        break;
    }
    return ret;
}
```

```c
int pinconf_map_to_setting(const struct pinctrl_map *map,
                           struct pinctrl_setting *setting)
{
    /* نقطة 4: لو الـ pctldev NULL دا bug خطير */
    WARN_ON(!setting->pctldev);

    /* نقطة 5: الـ config array فاضية؟ */
    WARN_ON(!map->data.configs.num_configs);
    ...
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware بتطابق حالة الـ Kernel

الـ pinconf بيكتب configs في registers الـ hardware عبر `pin_config_set`. التحقق بيتم بـ:

```bash
# 1. اقرأ من debugfs — بتعكس اللي الـ kernel يعتقده
cat /sys/kernel/debug/pinctrl/<dev>/pinconf-pins

# 2. قارن بالـ register المادي (شوف العنوان من datasheet)
# مثال: pinctrl base address = 0xfe000000، pin0 config register offset = 0x000
devmem2 0xfe000000 w

# 3. لو الـ driver بيستخدم regmap، ممكن تقرأ عبر
cat /sys/kernel/debug/regmap/<dev>/registers
```

**مقارنة منطقية:**

```
Kernel يقول: pin 0 = bias-pull-up
Register يقول: bit[8] = 1 (pull-up enable) ✓
```

---

#### 2. Register Dump Techniques

```bash
# devmem2 — أقرأ register واحد
devmem2 <physical_address> [b|h|w]
# b = byte, h = halfword (16-bit), w = word (32-bit)

# مثال: اقرأ 4 bytes من عنوان pinctrl
devmem2 0xfe600000 w

# اكتب على register (احترس!)
devmem2 0xfe600000 w 0x00000100

# /dev/mem — قراءة كبيرة
dd if=/dev/mem bs=4 count=256 skip=$((0xfe600000/4)) 2>/dev/null | xxd

# io command (من package ioport)
io -4 -r 0xfe600000
```

**Script يقرأ نطاق كامل من registers:**

```bash
#!/bin/bash
BASE=0xfe600000
COUNT=32
echo "Dumping $COUNT registers from $BASE"
for i in $(seq 0 4 $((COUNT*4-4))); do
    addr=$(printf "0x%08x" $((BASE + i)))
    val=$(devmem2 $addr w 2>/dev/null | grep "Value at" | awk '{print $NF}')
    echo "$addr: $val"
done
```

---

#### 3. Logic Analyzer / Oscilloscope

**متى تستخدم:**
- لو الـ kernel بيقول إن الـ config اتطبق لكن الـ hardware مش بيتصرف صح
- مشاكل الـ drive strength (الـ signal ضعيف)
- مشاكل الـ slew rate
- الـ pull-up/pull-down مش شغال صح

**نقاط القياس:**

```
Pin ──────────────────────────────────── Oscilloscope CH1
             │
             ├─ قيس الـ voltage level (high/low threshold صح؟)
             ├─ قيس الـ rise/fall time (= slew rate)
             └─ قيس الـ current (= drive strength)

VDD ────── Pull Resistor ───── Pin
                               │
                          Multimeter (measure pull voltage)
```

**Tips عملية:**
- لو الـ `bias-pull-up` اتحدد بس الـ pin بيقرأ LOW، شوف:
  - الـ pull resistor قيمته كبير جداً؟
  - في load خارجي بيسحب للـ GND؟
- لو الـ `drive-strength` منخفض والـ signal بيتشوّه في edges:
  - زود الـ drive strength في الـ DT config
  - قيس الـ capacitance على الـ trace

---

#### 4. Hardware Issues الشائعة وأنماطها في الـ Kernel Log

| المشكلة | الـ Pattern في dmesg | التفسير |
|---|---|---|
| Pin مش موجود في descriptor | `could not map pin config for "GPIO5"` | الاسم في الـ DT مختلف عن الـ pin descriptor |
| Register access خاطئ | `pin_config_set op failed for pin 3` + kernel oops | عنوان غلط أو الـ clock على الـ pinctrl controller مش شغال |
| Contention على الـ pin | `pin already requested` | جهازان بيطلبوا نفس الـ pin |
| Driver مش بيدعم config | `cannot get pin configuration, .pin_config_get missing` | الـ driver read-only أو مش محتاج get |
| Power domain إيقاف | `pin_config_set op failed` بعد runtime PM | الـ pinctrl controller دخل power-off state |

```bash
# تابع الـ log في real-time أثناء boot أو driver probe
dmesg -w | grep -E "pinctrl|pinconf|pinconfig|GPIO"

# ابحث عن errors تاريخية
journalctl -k | grep -i "pinctrl\|pinconf"
```

---

#### 5. Device Tree Debugging

الـ `pinconf_generic_parse_dt_config()` بتقرأ الـ properties من الـ DT وتحوّلها لـ config values.

**التحقق إن الـ DT بيطابق الـ Hardware:**

```bash
# اقرأ الـ compiled DT من الـ kernel
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "pinctrl"

# أو استخدم fdtdump
fdtdump /sys/firmware/fdt | less

# قارن properties DT مع register values
# مثال: لو DT قال bias-pull-up، bit معين في الـ register لازم = 1
```

**Properties شائعة في DT للـ pinconf وكيف تتحقق منها:**

```dts
/* مثال DT node */
pinctrl_uart0: uart0grp {
    fsl,pins = <
        MX8MN_IOMUXC_UART1_RXD  0x140  /* pull-up + 4mA */
        MX8MN_IOMUXC_UART1_TXD  0x140
    >;
};
```

```bash
# تحقق من الـ property اللي اتحولت
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pinconf-pins | grep -A2 "uart"
# لازم تشوف: config bias-pull-up drive-strength:4

# لو مش ظاهر، فعّل dynamic debug وشوف parse errors
echo 'file drivers/pinctrl/pinconf-generic.c +p' > /sys/kernel/debug/dynamic_debug/control
```

---

### Practical Commands

#### مجموعة الأوامر الكاملة — جاهزة للـ Copy

```bash
#!/bin/bash
# ============================================================
# pinconf full debug script
# ============================================================

PINCTRL_DBG="/sys/kernel/debug/pinctrl"
DYNDEBUG="/sys/kernel/debug/dynamic_debug/control"
TRACING="/sys/kernel/debug/tracing"

echo "=== [1] Listing pinctrl devices ==="
ls "$PINCTRL_DBG"/

echo ""
echo "=== [2] Reading pinconf-pins for all devices ==="
for dev in "$PINCTRL_DBG"/*/; do
    echo "--- $(basename $dev) ---"
    cat "${dev}pinconf-pins" 2>/dev/null || echo "(not available)"
done

echo ""
echo "=== [3] Reading pinconf-groups for all devices ==="
for dev in "$PINCTRL_DBG"/*/; do
    echo "--- $(basename $dev) ---"
    cat "${dev}pinconf-groups" 2>/dev/null || echo "(not available)"
done

echo ""
echo "=== [4] Enabling dynamic debug for pinctrl ==="
echo 'file drivers/pinctrl/* +pflm' > "$DYNDEBUG"
echo "Dynamic debug enabled. Run your test now and check dmesg."

echo ""
echo "=== [5] Enabling ftrace for pinctrl events ==="
if [ -d "$TRACING/events/pinctrl" ]; then
    echo 1 > "$TRACING/events/pinctrl/enable"
    echo 1 > "$TRACING/tracing_on"
    echo "ftrace enabled. Run your test, then:"
    echo "  cat $TRACING/trace"
else
    echo "(pinctrl tracepoints not available in this kernel)"
fi

echo ""
echo "=== [6] Checking kernel config ==="
for opt in CONFIG_PINCONF CONFIG_DEBUG_FS CONFIG_GENERIC_PINCONF \
           CONFIG_DYNAMIC_DEBUG CONFIG_PROVE_LOCKING; do
    val=$(grep "^$opt" /boot/config-$(uname -r) 2>/dev/null)
    echo "  $opt: ${val:-NOT SET}"
done
```

---

#### قراءة وتفسير الـ Output

**Output من `pinconf-pins`:**

```
Pin config settings per pin
Format: pin (name): configs
pin 0 (UART1_RXD): config bias-pull-up
pin 1 (UART1_TXD): config drive-strength:4
pin 2 (I2C1_SCL): config 00400001
pin 3 (I2C1_SDA): config bias-disable
```

**التفسير:**
- `bias-pull-up` — الـ driver نفّذ `pin_config_config_dbg_show` وفسّر الـ config
- `drive-strength:4` — 4mA drive strength، decoded بواسطة generic pinconf
- `00400001` — raw hex config، الـ driver مش بيوفر decoder؛ المعنى من الـ datasheet
- `bias-disable` — مفيش pull-up ولا pull-down

**فك تشفير الـ raw config `00400001`:**

```c
/* PIN_CONF_PACKED(param, arg) — param في low bits، arg في high bits */
/* 0x00400001:
 *   param = 0x01 = PIN_CONFIG_BIAS_DISABLE
 *   arg   = 0x0040 = 64 (قيمة إضافية)
 */
#define PIN_CONF_UNPACK_PARAM(conf)  ((conf) & 0xffUL)
#define PIN_CONF_UNPACK_ARG(conf)    ((conf) >> 8)
```

```bash
# فك تشفير raw config في bash
conf=0x00400001
param=$((conf & 0xff))
arg=$((conf >> 8))
echo "param=$param arg=$arg"
# param=1 arg=64
```

---

#### تتبع `pinconf_apply_setting` من البداية للنهاية

```bash
# 1. فعّل function_graph tracer
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo pinconf_apply_setting > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 2. trigger الـ apply (rebind driver مثلاً)
echo "driver_name" > /sys/bus/platform/drivers/pinctrl-foo/unbind
echo "driver_name" > /sys/bus/platform/drivers/pinctrl-foo/bind

# 3. اقرأ النتيجة
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

**مثال output:**

```
 # DURATION    FUNCTION CALLS
 # |           |   |   |   |
   4.521 us    |  pinconf_apply_setting() {
   1.200 us    |    ops->pin_config_set();
               |  }
```

لو الـ `ops->pin_config_set()` بتاخد وقت طويل جداً، ممكن في مشكلة في الـ register access أو الـ clock.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — UART بيفشل عند boot

#### العنوان
**الـ RS485 UART مش شغال على Industrial Gateway بعد تغيير kernel config**

#### السياق
شركة بتعمل industrial gateway بيستخدم RK3562، الـ gateway بيتكلم مع PLCs عن طريق RS485 على UART2. الـ product كان شغال تمام، لكن بعد upgrade للـ kernel من 6.1 لـ 6.6 وإعادة build، الـ UART2 وقف عن الشغل خالص.

#### المشكلة
الـ boot log بيظهر:

```
pinconfig core: pinconf has to be able to set a pins config
pinctrl-rk3562: probe failed -22
```

الـ UART2 device مش بيظهر في `/dev/ttyS*` خالص.

#### التحليل
الـ error جاي من `pinconf_check_ops()` في `pinconf.c`:

```c
int pinconf_check_ops(struct pinctrl_dev *pctldev)
{
    const struct pinconf_ops *ops = pctldev->desc->confops;

    /* We have to be able to config the pins in SOME way */
    if (!ops->pin_config_set && !ops->pin_config_group_set) {
        dev_err(pctldev->dev,
            "pinconf has to be able to set a pins config\n");
        return -EINVAL;
    }
    return 0;
}
```

الـ kernel الجديد غيّر الـ `CONFIG_PINCONF` من `y` لـ `m` في `.config`، لكن الـ RK3562 pinctrl driver اتبنى كـ built-in `y`. الـ `confops` struct في الـ driver بقت NULL pointer بسبب mismatch في الـ Kconfig dependencies، فالشرط `!ops->pin_config_set && !ops->pin_config_group_set` اتحقق وإرجع `-EINVAL`.

#### الحل
**أولاً: تحقق من الـ Kconfig:**
```bash
grep -E "PINCONF|PINCTRL_RK" /path/to/.config
```

**ثانياً: Fix الـ .config:**
```bash
# في menuconfig
CONFIG_PINCONF=y          # مش m
CONFIG_PINCTRL_ROCKCHIP=y
```

**ثالثاً: تحقق بعد rebuild:**
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rk3562/pinconf-pins | grep uart2
```

الـ `pinconf_check_ops()` بتتأكد إن الـ driver يعرف يعمل set لـ config على الأقل بطريقة واحدة — إما `pin_config_set` أو `pin_config_group_set` — وجود أي واحدة منهم كافي.

#### الدرس المستفاد
أي تغيير في `CONFIG_PINCONF` بيأثر على كل الـ pinctrl drivers في الـ SoC. لازم `pinconf_check_ops()` تنجح أثناء الـ probe — لو فشلت، الـ pinctrl device مش بيتسجل خالص وكل الـ peripherals المعتمدة عليه بتموت.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — HDMI No Signal

#### العنوان
**الـ HDMI مفيش signal بعد تطبيق custom pinctrl map غلط**

#### السياق
فريق بيعمل Android TV box بيستخدم Allwinner H616. الـ HDMI كان شغال في المرحلة الأولى، لكن بعد إضافة custom board support وكتابة machine-level pinctrl maps يدوياً، الـ HDMI اختفى.

#### المشكلة
الـ dmesg بيظهر:

```
pinconfig core: failed to register map hdmi-pins (2): no group/pin given
pinctrl core: failed to register map
sunxi-pinctrl: apply setting failed
```

الشاشة مفيش عليها أي حاجة.

#### التحليل
الـ error جاي من `pinconf_validate_map()`:

```c
int pinconf_validate_map(const struct pinctrl_map *map, int i)
{
    if (!map->data.configs.group_or_pin) {
        pr_err("failed to register map %s (%d): no group/pin given\n",
               map->name, i);
        return -EINVAL;
    }

    if (!map->data.configs.num_configs ||
            !map->data.configs.configs) {
        pr_err("failed to register map %s (%d): no configs given\n",
               map->name, i);
        return -EINVAL;
    }

    return 0;
}
```

الـ engineer كتب الـ `pinctrl_map` array بس نسي يحدد الـ `group_or_pin` field في المدخلة رقم 2 (index = 2):

```c
/* الكود الغلط */
static const struct pinctrl_map h616_hdmi_map[] = {
    PIN_MAP_MUX_GROUP("hdmi", PINCTRL_STATE_DEFAULT, "sunxi-pinctrl", "hdmi-group", "hdmi"),
    PIN_MAP_CONFIGS_GROUP("hdmi", PINCTRL_STATE_DEFAULT, "sunxi-pinctrl", "hdmi-group", hdmi_configs),
    /* entry رقم 2: group_or_pin = NULL بالغلط */
    PIN_MAP_CONFIGS_GROUP("hdmi", PINCTRL_STATE_DEFAULT, "sunxi-pinctrl", NULL, extra_configs),
};
```

#### الحل
```c
/* الكود الصح */
static const struct pinctrl_map h616_hdmi_map[] = {
    PIN_MAP_MUX_GROUP("hdmi", PINCTRL_STATE_DEFAULT, "sunxi-pinctrl",
                      "hdmi-group", "hdmi"),
    PIN_MAP_CONFIGS_GROUP("hdmi", PINCTRL_STATE_DEFAULT, "sunxi-pinctrl",
                          "hdmi-group", hdmi_configs),
    PIN_MAP_CONFIGS_GROUP("hdmi", PINCTRL_STATE_DEFAULT, "sunxi-pinctrl",
                          "hdmi-group", extra_configs), /* Fix: اضفنا الاسم */
};
```

**للتحقق:**
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-maps | grep hdmi
```

#### الدرس المستفاد
الـ `pinconf_validate_map()` بتفحص **كل entry** في الـ map array بالـ index بتاعه. الـ error message بيحتوي على الـ index الغلط مباشرةً — لازم تبص عليه وتعدّ entries الـ array من الصفر.

---

### السيناريو الثالث: IoT Sensor Board على STM32MP1 — I2C جهازين بيتعارضوا

#### العنوان
**الـ pull-up resistor configuration بيتكتب غلط على I2C pins في STM32MP1**

#### السياق
بورد IoT sensor بيضم STM32MP1 مع حساسات متعددة على I2C1 وI2C2. الـ I2C1 شغال، لكن الـ I2C2 بيطلع timeout في كل read محاولة.

#### المشكلة
الـ oscilloscope بيظهر إن SDA على I2C2 مش بترجع high بعد كل transaction. الـ dmesg مفيش فيه errors واضحة. الـ pins بيبان إنهم configured بس الـ pull-up مش شغالة.

#### التحليل
الـ engineer استخدم `pinconf_apply_setting()` لتطبيق الـ configs. الكود:

```c
int pinconf_apply_setting(const struct pinctrl_setting *setting)
{
    struct pinctrl_dev *pctldev = setting->pctldev;
    const struct pinconf_ops *ops = pctldev->desc->confops;
    int ret;

    if (!ops) {
        dev_err(pctldev->dev, "missing confops\n");
        return -EINVAL;
    }

    switch (setting->type) {
    case PIN_MAP_TYPE_CONFIGS_PIN:
        if (!ops->pin_config_set) {
            dev_err(pctldev->dev, "missing pin_config_set op\n");
            return -EINVAL;
        }
        ret = ops->pin_config_set(pctldev,
                setting->data.configs.group_or_pin,
                setting->data.configs.configs,
                setting->data.configs.num_configs);
        if (ret < 0) {
            dev_err(pctldev->dev,
                "pin_config_set op failed for pin %d\n",
                setting->data.configs.group_or_pin);
            return ret;
        }
        break;
    ...
    }
    return 0;
}
```

**المشكلة الحقيقية:** الـ Device Tree node للـ I2C2 pins كان بيستخدم `PIN_MAP_TYPE_CONFIGS_GROUP` بدل `PIN_MAP_TYPE_CONFIGS_PIN`. الـ STM32 pinctrl driver بيـimplements الـ `pin_config_group_set` بطريقة بتطبق الـ config على الـ group metadata بس، مش على كل pin منفرداً. الـ `PIN_CONF_PACK(PIN_CONFIG_BIAS_PULL_UP, 1)` اتطبق على الـ group selector مش على الـ physical pins.

```dts
/* الغلط */
&i2c2_pins {
    pinctrl-single,pins = <
        STM32_PINMUX('H', 4, AF4)  /* SCL */
        STM32_PINMUX('H', 5, AF4)  /* SDA */
    >;
    bias-pull-up;  /* بيتحول لـ GROUP config مش PIN config */
};
```

#### الحل
```dts
/* الصح: نعرّف كل pin على حدة */
&i2c2_scl_pins {
    pins = "PH4";
    function = "i2c2";
    bias-pull-up;
    drive-open-drain;
    slew-rate = <0>;
};

&i2c2_sda_pins {
    pins = "PH5";
    function = "i2c2";
    bias-pull-up;
    drive-open-drain;
    slew-rate = <0>;
};
```

**للتحقق:**
```bash
# شوف الـ config على كل pin منفرداً
cat /sys/kernel/debug/pinctrl/soc:pin-controller/pinconf-pins | grep -A2 "PH4\|PH5"
```

#### الدرس المستفاد
الـ `pinconf_apply_setting()` بتفرّق بين `PIN_MAP_TYPE_CONFIGS_PIN` و`PIN_MAP_TYPE_CONFIGS_GROUP` وبتوجه الـ call لـ callback مختلف في كل حالة. لازم تفهم إيه اللي الـ driver بتاعك بيعمله في كل callback — بعض الـ drivers بيعمل الـ group config بطريقة مختلفة عن الـ per-pin config.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — SPI بيرفض يشتغل بعد suspend/resume

#### العنوان
**الـ SPI config بيتمسح بعد system suspend على i.MX8**

#### السياق
automotive ECU بيستخدم i.MX8QM كـ main processor. بعد كل دورة suspend/resume، الـ SPI المتصل بـ external EEPROM بيوقف عن الاستجابة. الـ ECU بيحتاج يعمل hard reset عشان يشتغل تاني.

#### المشكلة
```
spi-imx: transfer timed out
imx-pinctrl: failed to apply SPI0 state: -22
```

#### التحليل
الـ problem في الـ flow بتاع `pinconf_map_to_setting()`:

```c
int pinconf_map_to_setting(const struct pinctrl_map *map,
                          struct pinctrl_setting *setting)
{
    struct pinctrl_dev *pctldev = setting->pctldev;
    int pin;

    switch (setting->type) {
    case PIN_MAP_TYPE_CONFIGS_PIN:
        pin = pin_get_from_name(pctldev,
                                map->data.configs.group_or_pin);
        if (pin < 0) {
            dev_err(pctldev->dev, "could not map pin config for \"%s\"",
                    map->data.configs.group_or_pin);
            return pin;
        }
        setting->data.configs.group_or_pin = pin;
        break;
    case PIN_MAP_TYPE_CONFIGS_GROUP:
        pin = pinctrl_get_group_selector(pctldev,
                                 map->data.configs.group_or_pin);
        if (pin < 0) {
            dev_err(pctldev->dev, "could not map group config for \"%s\"",
                    map->data.configs.group_or_pin);
            return pin;
        }
        setting->data.configs.group_or_pin = pin;
        break;
    default:
        return -EINVAL;
    }

    setting->data.configs.num_configs = map->data.configs.num_configs;
    setting->data.configs.configs = map->data.configs.configs;  /* pointer copy فقط! */

    return 0;
}
```

الـ `setting->data.configs.configs` هو **pointer** للـ configs array الأصلية في الـ `pinctrl_map`. بعد الـ suspend، الـ power domain للـ i.MX8 pinctrl بيتعمله reset، وعملية الـ resume بتحاول تعيد تطبيق الـ settings. في الـ custom board code، الـ `pinctrl_map` configs array اتعرّفت كـ stack-allocated variable في function اتنهت من زمان، فالـ pointer بقى dangling.

#### الحل
```c
/* الكود الغلط — configs على الـ stack */
static int board_spi_init(void)
{
    unsigned long spi_configs[] = {
        PIN_CONF_PACK(PIN_CONFIG_DRIVE_STRENGTH, 6),
        PIN_CONF_PACK(PIN_CONFIG_SLEW_RATE, 0),
    };
    /* spi_configs بيتمسح لما الـ function تخلص */
    pinctrl_register_mappings(spi_map, ARRAY_SIZE(spi_map));
}

/* الكود الصح — configs كـ static */
static const unsigned long spi_configs[] = {
    PIN_CONF_PACK(PIN_CONFIG_DRIVE_STRENGTH, 6),
    PIN_CONF_PACK(PIN_CONFIG_SLEW_RATE, 0),
};
```

**للتحقق إن الـ configs بتتطبق صح بعد resume:**
```bash
# قبل suspend
cat /sys/kernel/debug/pinctrl/*/pinconf-pins > /tmp/before.txt
# بعد resume
cat /sys/kernel/debug/pinctrl/*/pinconf-pins > /tmp/after.txt
diff /tmp/before.txt /tmp/after.txt
```

#### الدرس المستفاد
الـ `pinconf_map_to_setting()` بتعمل **shallow copy** — بتنسخ الـ pointer للـ configs مش الـ data نفسها. لازم الـ configs arrays تكون `static` أو globally allocated وتفضل موجودة طول عمر الـ driver. الـ suspend/resume بيكشف الـ dangling pointers اللي بتتستخبى في الـ normal operation.

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — الـ debugfs بيفيد في تشخيص GPIO مش شغال

#### العنوان
**استخدام `pinconf-pins` debugfs لتشخيص GPIO output مش شغال على AM62x بورد جديد**

#### السياق
فريق bring-up بيشغّل بورد custom على TI AM62x للمرة الأولى. الـ LED المتصل بـ GPIO2_IO15 مش بيشتغل رغم إن الكود بيكتب عليه. مفيش errors في الـ dmesg.

#### المشكلة
```bash
# محاولة تشغيل الـ LED
echo 1 > /sys/class/gpio/gpio47/value
# مفيش حاجة بتحصل — اللـ LED مش بيضيء
```

#### التحليل
الأدوات الموجودة في `pinconf.c` للـ debugfs:

```c
/* pinconf_dump_pin بتجمع المعلومات من مصدرين */
static void pinconf_dump_pin(struct pinctrl_dev *pctldev,
                             struct seq_file *s, int pin)
{
    const struct pinconf_ops *ops = pctldev->desc->confops;

    /* generic config dump — drive strength, pull, etc. */
    pinconf_generic_dump_pins(pctldev, s, NULL, pin);

    /* driver-specific dump */
    if (ops && ops->pin_config_dbg_show)
        ops->pin_config_dbg_show(pctldev, s, pin);
}
```

وكذلك `pinconf_show_setting()` بتظهر الـ active settings:

```c
void pinconf_show_setting(struct seq_file *s,
                          const struct pinctrl_setting *setting)
{
    struct pinctrl_dev *pctldev = setting->pctldev;
    const struct pinctrl_ops *pctlops = pctldev->desc->pctlops;
    struct pin_desc *desc;

    switch (setting->type) {
    case PIN_MAP_TYPE_CONFIGS_PIN:
        desc = pin_desc_get(setting->pctldev,
                            setting->data.configs.group_or_pin);
        seq_printf(s, "pin %s (%d)", desc->name,
                   setting->data.configs.group_or_pin);
        break;
    ...
    }
    pinconf_show_config(s, pctldev, setting->data.configs.configs,
                        setting->data.configs.num_configs);
}
```

**خطوات التشخيص الفعلية:**

```bash
# الخطوة 1: شوف الـ pinctrl devices المتاحة
ls /sys/kernel/debug/pinctrl/

# الخطوة 2: شوف كل pins وconfigs الحالية
cat /sys/kernel/debug/pinctrl/4084000.pinctrl/pinconf-pins | grep -A3 "gpio2-15\|GPIO2_IO15"

# الخطوة 3: شوف الـ active pinctrl state
cat /sys/kernel/debug/pinctrl/pinctrl-handles | grep -A10 "gpio2_15\|leds"
```

الـ output كشف:

```
pin 47 (GPIO2_IO15): config 00000001  <- bias-disable فقط، مفيش output-enable
```

بينما الـ DT كان:

```dts
/* الغلط — مفيش output-enable */
&main_pmx0 {
    led_pins: led-pins {
        pinctrl-single,pins = <
            AM62X_IOPAD(0x01d4, PIN_INPUT, 7) /* GPIO mode لكن INPUT! */
        >;
    };
};
```

#### الحل
```dts
/* الصح */
&main_pmx0 {
    led_pins: led-pins {
        pinctrl-single,pins = <
            AM62X_IOPAD(0x01d4, PIN_OUTPUT, 7) /* GPIO OUTPUT */
        >;
    };
};
```

**بعد إصلاح الـ DT والـ rebuild:**

```bash
# تحقق إن الـ config اتطبق
cat /sys/kernel/debug/pinctrl/4084000.pinctrl/pinconf-pins | grep "GPIO2_IO15"
# المفروض يظهر: output-enable

# اختبر الـ LED
echo 1 > /sys/class/gpio/gpio47/value  # اللـ LED هيشتغل
```

الـ `pinconf_init_device_debugfs()` في `pinconf.c` بتسجل الـ files دي تلقائياً:

```c
void pinconf_init_device_debugfs(struct dentry *devroot,
                         struct pinctrl_dev *pctldev)
{
    debugfs_create_file("pinconf-pins", 0444,
                        devroot, pctldev, &pinconf_pins_fops);
    debugfs_create_file("pinconf-groups", 0444,
                        devroot, pctldev, &pinconf_groups_fops);
}
```

#### الدرس المستفاد
الـ debugfs interface في `pinconf.c` — خصوصاً `pinconf-pins` و`pinconf-groups` — هو **أول حاجة** لازم تبص فيها في أي bring-up. بيفضح الفرق بين اللي إنت فاكر إنه بيتطبق واللي بيتطبق فعلاً. الـ `PIN_INPUT` بدل `PIN_OUTPUT` خطأ صامت تماماً لولا الـ debugfs.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأول لمتابعة تطور الـ pinctrl/pinconf subsystem — كل المقالات دي بتوثّق المراحل الحقيقية للتطوير:

| المقال | الوصف |
|--------|-------|
| [The pin control subsystem](https://lwn.net/Articles/468759/) | Overview شامل كتبه Jonathan Corbet عام 2011 — نقطة البداية لأي حد يريد يفهم الـ subsystem |
| [drivers: create a pin control subsystem](https://lwn.net/Articles/463335/) | الـ patch series الأصلية اللي أنشأت الـ pinctrl subsystem بالكامل |
| [pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/) | إضافة الـ per-pin و per-group pin config interface — الأساس اللي بنيت عليه `pinconf.c` |
| [pinctrl: add a pin config interface](https://lwn.net/Articles/471826/) | تفاصيل تقنية لإضافة الـ `pinconf_ops` interface |
| [pinctrl/pinconfig: add debug interface](https://lwn.net/Articles/545790/) | إضافة الـ debugfs interface اللي موجودة في `CONFIG_DEBUG_FS` block في الكود |
| [pinctrl: common handling of generic pinconfig props in dt](https://lwn.net/Articles/553754/) | دعم الـ Device Tree لخصائص الـ pinconf — أساس `pinconf_generic_parse_dt_config` |
| [Documentation/pinctrl.txt](https://lwn.net/Articles/465077/) | النسخة الأولى من التوثيق الرسمي للـ kernel |
| [drivers: create a pin control subsystem v8](https://lwn.net/Articles/460768/) | الإصدار الثامن من الـ patch — يوضح التطور التدريجي قبل الـ merge |

---

### التوثيق الرسمي للـ kernel

**الـ Documentation/** الرسمية هي المرجع الأكثر دقة وتحديثاً:

```
Documentation/driver-api/pin-control.rst
```

- **Online (latest):** [PINCTRL subsystem — kernel.org](https://www.kernel.org/doc/html/latest/driver-api/pinctl.html)
- **Online (v4.14):** [PINCTRL — kernel.org v4.14](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html)
- **Raw text (قديم):** [Documentation/pinctrl.txt](https://www.kernel.org/doc/Documentation/pinctrl.txt)

الـ header files الرسمية في الـ kernel source:

```
include/linux/pinctrl/pinconf.h       ← struct pinconf_ops + pin_config_get/set API
include/linux/pinctrl/pinconf-generic.h ← enum pin_config_param + generic configs
include/linux/pinctrl/pinctrl.h       ← struct pinctrl_dev + pinctrl_ops
include/linux/pinctrl/machine.h       ← struct pinctrl_map
```

---

### ملفات الـ source الأساسية

الملفات دي هي قلب الـ pinconf subsystem:

```
drivers/pinctrl/pinconf.c      ← core implementation (موضوع الدراسة)
drivers/pinctrl/pinconf.h      ← internal interface
drivers/pinctrl/pinconf-generic.c ← generic pin config + DT parsing
drivers/pinctrl/core.c         ← pinctrl core
drivers/pinctrl/core.h         ← internal core structs
```

---

### Commits مهمة في الـ git history

الـ commits دي قدّمت أو غيّرت الـ pinconf بشكل جوهري — يمكن تتبّعها عبر:

```bash
# تاريخ الـ pinconf.c من البداية
git log --oneline drivers/pinctrl/pinconf.c

# الـ commit الأصلي للـ subsystem (kernel 3.1)
git log --oneline --all | grep -i "pin control\|pinctrl\|pinconf" | tail -20

# مشاهدة الـ patch الأصلي
git show $(git log --oneline drivers/pinctrl/pinconf.c | tail -1 | awk '{print $1}')
```

على **GitHub/Torvalds:**

- [pinconf.c على GitHub](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/pinconf.c)
- [pinconf-generic.h على GitHub](https://github.com/torvalds/linux/blob/master/include/linux/pinctrl/pinconf-generic.h)
- [pinconf.h (include) على GitHub](https://github.com/torvalds/linux/blob/master/include/linux/pinctrl/pinconf.h)

---

### نقاشات الـ mailing list

**الـ linux-gpio mailing list** هو القناة الرسمية لمتابعة تطوير الـ pinctrl:

- [Patchwork: linux-gpio project](https://patchwork.kernel.org/project/linux-gpio/list/)
- [linux-kernel@vger.kernel.org أرشيف](https://www.mail-archive.com/linux-kernel@vger.kernel.org/)
- [LKML: Linus Walleij git pull for v6.19](https://lkml.org/lkml/2025/12/8/736) — مثال على الـ pull requests الدورية للـ subsystem

للبحث في الأرشيف:
```
site:lkml.org "pinconf" OR "pinconf_ops" OR "pin_config_set"
site:patchwork.kernel.org/project/linux-gpio pinconf
```

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **متاح مجاناً:** [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)
- **الفصول ذات الصلة:**
  - Chapter 9: Communicating with Hardware — فهم الـ I/O registers والـ memory-mapped I/O اللي الـ pinctrl drivers بتتعامل معاها
  - Chapter 14: The Linux Device Model — فهم `struct device` والـ driver model اللي الـ `pctldev->dev` جزء منه
- **ملاحظة:** الكتاب قديم (2005) ومش فيه chapter عن pinctrl لأن الـ subsystem اتضاف بعده، بس الأساسيات لازمة

#### Linux Kernel Development (Robert Love) — 3rd Edition
- **الفصول ذات الصلة:**
  - Chapter 17: Devices and Modules — فهم الـ device model و `struct device`
  - Chapter 5: System Calls — فهم الـ kernel/userspace interface (مفيد لفهم debugfs)
  - Chapter 15: The Process Address Space — مهم لفهم الـ memory management في الـ drivers

#### Embedded Linux Primer (Christopher Hallinan) — 2nd Edition
- **الفصل الأهم:** Chapter 14: Kernel Debugging Techniques — يغطي الـ debugfs اللي الـ `pinconf_init_device_debugfs` بتستخدمه
- **كمان:** Chapter 16: Porting Linux — فيه أمثلة عملية لـ BSP development وهو السياق الأشمل للـ pinctrl

#### Linux Driver Development for Embedded Processors (Alberto Liberal de los Ríos)
- الأحدث والأكثر شمولاً للـ embedded pinctrl — فيه chapters كاملة عن الـ pinctrl subsystem مع أمثلة على Raspberry Pi و i.MX

---

### kernelnewbies.org

صفحات الـ kernel releases بتوثّق كل إضافة جديدة للـ pinctrl في كل إصدار:

- [Linux_6.11 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.11)
- [Linux_6.13 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.13)
- [Linux_6.17 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.17)
- [LinuxChanges — كل الإصدارات](https://kernelnewbies.org/LinuxChanges)

للبحث عن pinconf تحديداً:
```
site:kernelnewbies.org pinconf OR "pin config" OR pinctrl
```

---

### elinux.org

- [Pin Control GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) — presentation من ELC عن العلاقة بين الـ pinctrl والـ GPIO subsystems
- [EBC Device Trees](https://elinux.org/EBC_Device_Trees) — أمثلة عملية على pinctrl configuration في Device Tree
- [Tests: i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) — مثال على استخدام الـ pinctrl مع الـ I2C multiplexing

---

### مقالات Embedded.com

- [Linux device driver development: The pin control subsystem](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) — شرح عملي للـ pinctrl من منظور developer

---

### مصطلحات البحث

للبحث عن معلومات إضافية، استخدم الـ search terms دي:

```
# للـ kernel documentation
pinctrl subsystem linux kernel documentation
pinconf_ops linux kernel driver
pin_config_set linux kernel
pinconf_apply_setting linux
pinconf_generic_parse_dt_config device tree

# للـ mailing lists
linux-gpio pinconf patch 2024
pinctrl driver implementation tutorial
pinconf_ops pin_config_set implementation example

# للـ hardware-specific examples
imx6 pinctrl pinconf implementation
rockchip pinctrl driver pinconf
sunxi pinctrl pinconf

# لـ debugging
pinconf debugfs linux kernel
/sys/kernel/debug/pinctrl pinconf-pins
debugfs pinconf-groups linux
```

---

### ملخص المراجع الأساسية

```
الأهم للبدء:
├── LWN: The pin control subsystem     → https://lwn.net/Articles/468759/
├── LWN: add a generic pin config      → https://lwn.net/Articles/468770/
├── Kernel docs (RST)                  → Documentation/driver-api/pin-control.rst
└── kernel.org HTML                    → https://www.kernel.org/doc/html/latest/driver-api/pinctl.html

للتعمق في الكود:
├── pinconf.c                          → drivers/pinctrl/pinconf.c
├── pinconf-generic.c                  → drivers/pinctrl/pinconf-generic.c
└── pinconf-generic.h                  → include/linux/pinctrl/pinconf-generic.h

للـ debugging:
├── LWN: add debug interface           → https://lwn.net/Articles/545790/
└── debugfs path                       → /sys/kernel/debug/pinctrl/<dev>/pinconf-pins
```
## Phase 8: Writing simple module

### الفكرة

**`pinconf_apply_setting`** هي الدالة اللي بتتنفذ فعلياً لما الـ kernel يطبّق إعدادات pin — زي pull-up/pull-down أو drive strength — على hardware. دي نقطة مثيرة للاهتمام لأن كل مرة بيتطبق فيها config على pin أو group بتعدّي من هنا. هنستخدم **kprobe** نعلّق عليها ونطبع اسم الـ controller ورقم الـ pin ونوع الـعملية.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe on pinconf_apply_setting()
 *
 * Every time the pinctrl core applies a pin/group config we intercept
 * the call and log: controller name, pin-or-group index, setting type,
 * and number of config words being applied.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/pinctrl/pinctrl.h>   /* struct pinctrl_dev, pinctrl_setting */
#include <linux/pinctrl/machine.h>   /* PIN_MAP_TYPE_CONFIGS_PIN/GROUP */

/* ------------------------------------------------------------------ */
/* Internal layout we need — mirrors kernel's core.h just enough.     */
/* We only read fields; we never write.                                */
/* ------------------------------------------------------------------ */

/*
 * pinctrl_setting_configs: the data inside a PIN_MAP_TYPE_CONFIGS_* setting.
 * Defined in <linux/pinctrl/machine.h> but reproduced here for clarity.
 */
struct my_pinctrl_setting_configs {
    unsigned int  group_or_pin;   /* pin number OR group selector */
    unsigned long *configs;       /* array of packed config values */
    unsigned int  num_configs;
};

/*
 * We only need the fields we'll actually read from pinctrl_setting.
 * The real struct lives in drivers/pinctrl/core.h (not exported).
 * Using the same memory layout is safe because we only READ the struct
 * through the pointer the kernel passes us.
 */
struct my_pinctrl_setting {
    /* list_head comes first — skip it (two pointers) */
    void *list_next;
    void *list_prev;

    enum pinctrl_map_type type;          /* PIN_MAP_TYPE_CONFIGS_PIN / _GROUP */
    struct pinctrl_dev   *pctldev;       /* owning controller */

    /* union: for CONFIGS_* the first member is the configs struct */
    struct my_pinctrl_setting_configs data;
};

/* ------------------------------------------------------------------ */
/* kprobe pre-handler                                                  */
/* ------------------------------------------------------------------ */

static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * First argument of pinconf_apply_setting(const struct pinctrl_setting *)
     * On x86-64: rdi holds arg0.
     * On arm64 : x0  holds arg0.
     * kprobes gives us pt_regs so we use regs_get_kernel_argument() which
     * is arch-agnostic since kernel 5.8.
     */
    const struct my_pinctrl_setting *setting =
        (const struct my_pinctrl_setting *)regs_get_kernel_argument(regs, 0);

    if (!setting || !setting->pctldev)
        return 0;   /* nothing useful to print */

    /*
     * pctldev->desc->name is the controller's human-readable name
     * (e.g. "gpio0", "soc-pinctrl").
     * We access it through the public pinctrl_dev pointer; desc and name
     * are at stable, well-known offsets.
     */
    const char *ctrl_name = dev_name(setting->pctldev->dev);

    if (setting->type == PIN_MAP_TYPE_CONFIGS_PIN) {
        pr_info("apply_setting: ctrl=[%s] type=PIN pin=%u ncfg=%u\n",
                ctrl_name,
                setting->data.group_or_pin,
                setting->data.num_configs);
    } else if (setting->type == PIN_MAP_TYPE_CONFIGS_GROUP) {
        pr_info("apply_setting: ctrl=[%s] type=GROUP grp=%u ncfg=%u\n",
                ctrl_name,
                setting->data.group_or_pin,
                setting->data.num_configs);
    }

    return 0;   /* 0 = let the original function run normally */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */

static struct kprobe kp = {
    .symbol_name = "pinconf_apply_setting",  /* function to hook */
    .pre_handler = handler_pre,              /* called before the function */
};

/* ------------------------------------------------------------------ */
/* module_init / module_exit                                           */
/* ------------------------------------------------------------------ */

static int __init pinconf_probe_init(void)
{
    int ret;

    /*
     * register_kprobe() finds the kernel symbol by name, patches a
     * breakpoint instruction at its entry, and registers our handler.
     * Returns 0 on success or a negative errno.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe planted on pinconf_apply_setting @ %px\n", kp.addr);
    return 0;
}

static void __exit pinconf_probe_exit(void)
{
    /*
     * unregister_kprobe() removes the breakpoint and waits for any
     * in-flight handler to finish before returning.  Must be called
     * in exit to avoid a dangling handler after the module is gone.
     */
    unregister_kprobe(&kp);
    pr_info("kprobe removed from pinconf_apply_setting\n");
}

module_init(pinconf_probe_init);
module_exit(pinconf_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on pinconf_apply_setting to trace pin config application");
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
#include <linux/pinctrl/pinctrl.h>
#include <linux/pinctrl/machine.h>
```

الـ `kprobes.h` بيجيب تعريف `struct kprobe` والـ API بتاعتها. الـ `pinctrl.h` و`machine.h` بيجيبوا تعريف `struct pinctrl_dev` وثوابت `PIN_MAP_TYPE_*` اللي محتاجينها نفهم نوع الـ setting.

#### الـ Shadow Structs

```c
struct my_pinctrl_setting { ... };
```

الـ `struct pinctrl_setting` الحقيقية موجودة في `drivers/pinctrl/core.h` اللي مش exported. بما إننا بنقرأ بس (مش بنكتب)، بنعمل نسخة بنفس ترتيب الـ fields عشان نوصل للبيانات اللي محتاجينها من الـ pointer اللي بييجي في الـ argument.

#### الـ `handler_pre`

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

الـ `pt_regs` بيحمل state الـ CPU لحظة الـ breakpoint. استخدمنا `regs_get_kernel_argument(regs, 0)` عشان دي دالة portable على x86-64 وarm64 بدل ما نـhard-code اسم register. بنطبع اسم الـ controller ورقم الـ pin أو الـ group وعدد الـ configs — معلومات كافية تفيد في debugging.

#### الـ `register_kprobe` في الـ init

الـ `register_kprobe` بتـpatch breakpoint في أول الدالة المستهدفة وقت الـ runtime. لو فشلت (مثلاً الـ symbol مش موجود أو الـ kernel compiled بـ `CONFIG_KPROBES=n`) بترجع error وبنحجب loading المودول.

#### الـ `unregister_kprobe` في الـ exit

لو خرج المودول من الـ kernel من غير ما يشيل الـ kprobe، أي استدعاء لـ `pinconf_apply_setting` هيحاول يجري handler موجود في memory اتحررت — kernel panic مضمون. الـ `unregister_kprobe` بتشيل الـ breakpoint وبتضمن إن مفيش handler شغال قبل ما تكمل.

---

### Makefile

```makefile
obj-m += pinconf_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

### تشغيل وتجربة

```bash
# بناء المودول
make

# تحميله
sudo insmod pinconf_probe.ko

# شوف الـ log — اي pin config بيتطبق هيظهر هنا
sudo dmesg -w | grep apply_setting

# تفريغه
sudo rmmod pinconf_probe
```

مثال واقعي للـ output لما جهاز بـ pinctrl يشتغل:

```
[  42.311234] kprobe planted on pinconf_apply_setting @ ffffffffc0a1b340
[  42.450011] apply_setting: ctrl=[soc-pinctrl] type=PIN  pin=17 ncfg=2
[  42.450098] apply_setting: ctrl=[soc-pinctrl] type=GROUP grp=3  ncfg=1
```
