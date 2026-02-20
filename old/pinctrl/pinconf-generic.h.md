# شرح `pinconf-generic.h` — الـ Generic Pin Configuration Framework

هذا الملف هو **قلب نظام الـ pin configuration العام** في Linux. يُوفّر encoding موحّد وجاهز لكل الخصائص الكهربائية الشائعة، بدلاً من أن يخترع كل driver encoding خاص به.

---

## أولاً: الـ Config Value Encoding — كيف يُخزَّن كل شيء في `unsigned long`؟

هذه أذكى فكرة في الملف. كل config كاملة (نوع + قيمة) تُخزَّن في **`unsigned long` واحد**:

```
 unsigned long (32-bit على الأقل):
 ┌──────────────────────────────────┬─────────────────┐
 │         argument (24 bit)        │   param (8 bit) │
 │         bits [31:8]              │   bits [7:0]    │
 └──────────────────────────────────┴─────────────────┘

مثال: PIN_CONFIG_DRIVE_STRENGTH بقيمة 8mA
  param    = PIN_CONFIG_DRIVE_STRENGTH = 9  → 0x09
  argument = 8 (mA)                         → 0x08
  المحزوم  = (8 << 8) | 9              = 0x0809

مثال: PIN_CONFIG_BIAS_PULL_UP مُفعَّل
  param    = PIN_CONFIG_BIAS_PULL_UP = 5    → 0x05
  argument = 1 (enabled)                    → 0x01
  المحزوم  = (1 << 8) | 5              = 0x0105
```

### الدوال المساعدة

```c
// فكّ الـ param من الـ config المحزوم
static inline enum pin_config_param pinconf_to_config_param(unsigned long config)
{
    return (enum pin_config_param)(config & 0xffUL);  // أخذ الـ 8 bits السفلى
}

// فكّ الـ argument من الـ config المحزوم
static inline u32 pinconf_to_config_argument(unsigned long config)
{
    return (u32)((config >> 8) & 0xffffffUL);  // أخذ الـ 24 bits العليا
}

// تجميع param + argument في قيمة واحدة
static inline unsigned long pinconf_to_config_packed(enum pin_config_param param,
                                                     u32 argument)
{
    return PIN_CONF_PACKED(param, argument);  // = (argument << 8) | param
}
```

**استخدام عملي في driver:**

```c
static int foo_pin_config_set(struct pinctrl_dev *pctldev,
                              unsigned int pin,
                              unsigned long *configs,
                              unsigned int num_configs)
{
    for (int i = 0; i < num_configs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        u32 arg                     = pinconf_to_config_argument(configs[i]);

        switch (param) {
        case PIN_CONFIG_DRIVE_STRENGTH:
            set_drive_strength_reg(pin, arg);  // arg بالـ mA
            break;
        case PIN_CONFIG_BIAS_PULL_UP:
            set_pull_up_reg(pin, arg != 0);
            break;
        }
    }
}
```

---

## ثانياً: `enum pin_config_param` — شرح كل قيمة

### مجموعة BIAS — الـ Biasing الكهربائي

```
   VDD
    │
   [R]  ← pull-up resistor
    │
────┴──── pin ────┬──── إلى الدائرة
                  │
                 [R]  ← pull-down resistor
                  │
                 GND
```

|الـ Param|الوصف|الـ Argument|
|---|---|---|
|`BIAS_DISABLE`|يُلغي كل biasing (لا pull-up ولا pull-down)|يُتجاهل|
|`BIAS_HIGH_IMPEDANCE`|يضع الـ pin في حالة Hi-Z أو tristate — الـ pin منفصل كهربائياً|يُتجاهل|
|`BIAS_PULL_UP`|يربط الـ pin بـ VDD عبر resistor داخلي|قيمة المقاومة بالـ Ohm أو 1 للتفعيل|
|`BIAS_PULL_DOWN`|يربط الـ pin بـ GND عبر resistor داخلي|قيمة المقاومة بالـ Ohm أو 1 للتفعيل|
|`BIAS_BUS_HOLD`|الـ pin يحتفظ بآخر قيمة على الـ bus (weak latch)|يُتجاهل|
|`BIAS_PULL_PIN_DEFAULT`|الـ hardware نفسه يقرر اتجاه الـ pull بناءً على الـ mux function الحالية|0 = تجاهل، غير 0 = فعّل|

**الفرق بين `BIAS_HIGH_IMPEDANCE` و `BIAS_DISABLE`:**

- `BIAS_DISABLE`: يُلغي الـ pull-up أو pull-down فقط، الـ pin لا يزال متصلاً بالدائرة
- `BIAS_HIGH_IMPEDANCE`: يفصل الـ pin كهربائياً من الـ output buffer تماماً

---

### مجموعة DRIVE — طريقة قيادة الـ Output

```
  Push-Pull:          Open-Drain:        Open-Source:
  VDD                 VDD                VDD
   │                   │                  │
  [P-MOS]         external R            [P-MOS]
   │                   │                  │
 ──┤── output      ────┤── output      ────┤── output
   │                   │                  │
  [N-MOS]           [N-MOS]           (مفتوح - لا يسحب)
   │                   │
  GND                 GND
```

|الـ Param|الوصف|الـ Argument|
|---|---|---|
|`DRIVE_PUSH_PULL`|الوضع الافتراضي: transistoران يدفعان HIGH أو LOW بنشاط|يُتجاهل|
|`DRIVE_OPEN_DRAIN`|يسحب LOW فقط، ترك HIGH لـ external resistor (I2C مثلاً)|يُتجاهل|
|`DRIVE_OPEN_SOURCE`|يدفع HIGH فقط، ترك LOW للـ external (نادر الاستخدام)|يُتجاهل|
|`DRIVE_STRENGTH`|الحد الأقصى للتيار الذي يمكن أن يصدر أو يستقبله الـ pin|بالـ mA|
|`DRIVE_STRENGTH_UA`|نفسه لكن بدقة أعلى|بالـ µA|
|`OUTPUT_IMPEDANCE_OHMS`|ضبط مقاومة الـ output buffer|بالـ Ohm|

---

### مجموعة INPUT — إعدادات الـ Input

|الـ Param|الوصف|الـ Argument|
|---|---|---|
|`INPUT_ENABLE`|تفعيل أو تعطيل الـ input buffer|1 = فعّل، 0 = عطّل|
|`INPUT_DEBOUNCE`|ينتظر استقرار الإشارة قبل قراءتها (مفيد للـ buttons)|زمن الـ debounce بالـ µsec، 0 = إيقاف|
|`INPUT_SCHMITT_ENABLE`|تفعيل Schmitt Trigger على الـ input (hysteresis لمقاومة الضوضاء)|1 = فعّل، 0 = عطّل|
|`INPUT_SCHMITT`|Schmitt Trigger بعتبة قابلة للضبط|قيمة مخصصة|
|`INPUT_SCHMITT_UV`|Schmitt Trigger بعتبة بالـ µV|بالـ µV|

**Schmitt Trigger موضّح:**

```
بدون Schmitt:          مع Schmitt:
     Vin                    Vin
      │                      │
   VT─┼──────              VT+─┼───────  threshold للـ HIGH
      │    └──output           │    └─
      │                    VT-─┼───────  threshold للـ LOW
                               │    ┌─
                                    └──output

→ يمنع الـ flickering عند الـ noise حول الـ threshold
```

---

### مجموعة MODE والمتفرقات

|الـ Param|الوصف|الـ Argument|
|---|---|---|
|`MODE_LOW_POWER`|وضع استهلاك طاقة منخفض|1 = فعّل، 0 = أوقف|
|`MODE_PWM`|تهيئة الـ pin لوضع PWM|مخصص|
|`LEVEL`|ضبط الـ pin كـ output وتحديد قيمته (HIGH أو LOW)|1 = HIGH، 0 = LOW|
|`OUTPUT_ENABLE`|تفعيل الـ output buffer دون تحديد قيمة|1 = فعّل، 0 = عطّل|
|`PERSIST_STATE`|الاحتفاظ بالـ config عبر sleep أو controller reset|يُتجاهل|
|`POWER_SOURCE`|اختيار مصدر الطاقة للـ pin لو متعدد|مخصص|
|`SLEEP_HARDWARE_STATE`|يشير لأن هذا الـ state متعلق بالـ sleep|يُتجاهل|

---

### مجموعة SKEW و SLEW

هذه مهمة في الأنظمة عالية السرعة:

```
SLEW_RATE — سرعة تغيير المستوى:
  Fast slew:  │‾‾│  حواف حادة → ضوضاء EMI أعلى
  Slow slew:  ╱‾‾╲  حواف ناعمة → أقل ضوضاء لكن أبطأ

SKEW_DELAY — تأخير الإشارة:
  Clock ──────┬────────────────
              │  delay
  Data  ──────┴──── [inverters] ──── output
  (يضبط عدد الـ double inverters في المسار)
```

|الـ Param|الوصف|الـ Argument|
|---|---|---|
|`SLEW_RATE`|معدل تغيير الـ output voltage|مخصص|
|`SKEW_DELAY`|تأخير موحّد للـ input والـ output|مخصص|
|`SKEW_DELAY_INPUT_PS`|تأخير الـ input فقط|بالـ picosecond|
|`SKEW_DELAY_OUTPUT_PS`|تأخير الـ output فقط|بالـ picosecond|

---

### حدود الـ Enum

```c
PIN_CONFIG_END = 0x7F  // آخر قيمة قياسية
PIN_CONFIG_MAX = 0xFF  // الحد الأقصى للـ encoding

// لو تريد custom params خاصة بـ driver:
#define MY_CUSTOM_PARAM  (PIN_CONFIG_END + 1)  // = 0x80
#define MY_CUSTOM_PARAM2 (PIN_CONFIG_END + 2)  // = 0x81
// يمكنك حتى 0xFF = 128 custom param
```

---

## ثالثاً: الـ Structs المساعدة للـ Debugfs

### `struct pin_config_item`

تُستخدم لتوصيف كيفية طباعة config معينة في الـ debugfs:

```c
struct pin_config_item {
    const enum pin_config_param param;   // أي param يصف؟
    const char * const display;          // اسم للعرض
    const char * const format;           // صيغة العرض
    bool has_arg;                        // هل له argument يُعرض؟
    const char * const *values;          // قائمة قيم نصية (اختياري)
    size_t num_values;
};

// مثال:
PCONFDUMP(PIN_CONFIG_BIAS_PULL_UP, "input bias pull up", "Ohms", true)
// → يُطبع في debugfs: "input bias pull up (10000 Ohms)"

PCONFDUMP(PIN_CONFIG_DRIVE_PUSH_PULL, "output drive push pull", NULL, false)
// → يُطبع في debugfs: "output drive push pull"
```

### `struct pinconf_generic_params`

تربط **property name في الـ Device Tree** بـ `pin_config_param` مع قيمة افتراضية:

```c
struct pinconf_generic_params {
    const char * const property;     // اسم الـ DT property
    enum pin_config_param param;     // الـ param المقابل
    u32 default_value;               // القيمة لو لم يُحدَّد argument
    const char * const *values;      // قائمة قيم نصية (اختياري)
    size_t num_values;
};

// مثال داخلي في الـ kernel:
{ "bias-pull-up",    PIN_CONFIG_BIAS_PULL_UP,    1 }
{ "drive-strength",  PIN_CONFIG_DRIVE_STRENGTH,  0 }
{ "input-schmitt-enable", PIN_CONFIG_INPUT_SCHMITT_ENABLE, 1 }
```

---

## رابعاً: الـ DT Parsing Functions — السحر الحقيقي

هذه الدوال هي سبب وجود هذا الملف أصلاً. تُحوّل الـ Device Tree تلقائياً إلى `pinctrl_map` entries.

### `pinconf_generic_dt_node_to_map()` و variants

```c
// للـ group configs:
pinconf_generic_dt_node_to_map_group(pctldev, np, map, num_maps)
// → ينتج PIN_MAP_TYPE_CONFIGS_GROUP entries

// للـ pin configs:
pinconf_generic_dt_node_to_map_pin(pctldev, np, map, num_maps)
// → ينتج PIN_MAP_TYPE_CONFIGS_PIN entries

// يستنتج النوع تلقائياً من الـ DT properties:
pinconf_generic_dt_node_to_map_all(pctldev, np, map, num_maps)
// → يمرر PIN_MAP_TYPE_INVALID ويترك الـ parser يقرر
```

### كيف يعمل هذا في الواقع؟

```dts
/* Device Tree: */
&pinctrl {
    uart0_pins: uart0-pins {
        pins = "GPIO0_A0", "GPIO0_A1";
        bias-pull-up;
        drive-strength = <4>;
        slew-rate = <1>;
    };
};
```

```
pinconf_generic_dt_node_to_map_pin() تقرأ هذا وتُنتج:

configs[0] = PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 1)    = 0x0105
configs[1] = PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 4)  = 0x0409
configs[2] = PIN_CONF_PACKED(PIN_CONFIG_SLEW_RATE, 1)       = 0x011B

pinctrl_map entry:
{
    type = PIN_MAP_TYPE_CONFIGS_PIN,
    data.configs.pin_name = "GPIO0_A0",
    data.configs.configs  = [0x0105, 0x0409, 0x011B],
    data.configs.num_configs = 3
}
```

### `pinconf_generic_dt_free_map()`

تُحرر الـ memory المُخصصة من `dt_node_to_map()`. يجب استدعاؤها في `dt_free_map()` الخاصة بك.

---

## كيف تستخدم كل هذا في BSP Driver

```c
// 1. في pinctrl_desc تُعلن أنك generic:
static struct pinconf_ops foo_pinconf_ops = {
    .is_generic         = true,       // ← هذا يفتح الـ generic framework
    .pin_config_get     = foo_pin_config_get,
    .pin_config_set     = foo_pin_config_set,
};

// 2. في dt_node_to_map تستخدم الـ generic parser مباشرة:
static int foo_dt_node_to_map(struct pinctrl_dev *pctldev,
                              struct device_node *np,
                              struct pinctrl_map **map,
                              unsigned int *num_maps)
{
    // بدل كتابة parser خاص بك، استخدم الجاهز:
    return pinconf_generic_dt_node_to_map_all(pctldev, np, map, num_maps);
}

static void foo_dt_free_map(struct pinctrl_dev *pctldev,
                            struct pinctrl_map *map,
                            unsigned int num_maps)
{
    pinconf_generic_dt_free_map(pctldev, map, num_maps);
}

// 3. في pin_config_set تفك الـ packed values بالدوال الجاهزة:
static int foo_pin_config_set(struct pinctrl_dev *pctldev,
                              unsigned int pin,
                              unsigned long *configs,
                              unsigned int num_configs)
{
    for (int i = 0; i < num_configs; i++) {
        enum pin_config_param param = pinconf_to_config_param(configs[i]);
        u32 arg                     = pinconf_to_config_argument(configs[i]);

        switch (param) {
        case PIN_CONFIG_BIAS_PULL_UP:
            writel(arg ? BIT(pin) : 0, PULL_UP_REG);
            break;
        case PIN_CONFIG_DRIVE_STRENGTH:
            writel(arg / 2, DRIVE_REG + pin * 4);  // مثلاً: 4mA → 2
            break;
        default:
            return -ENOTSUPP;
        }
    }
    return 0;
}
```

---

## ملخص — لماذا Generic؟

```
بدون generic framework:             مع generic framework:
───────────────────────             ──────────────────────
Driver A encoding خاص               encoding موحّد للجميع
Driver B encoding مختلف             DT parser جاهز
Driver C encoding ثالث              debugfs تلقائي
كل driver يكتب DT parser            custom params ممكنة
كل driver يكتب debugfs code         (PIN_CONFIG_END+1 فأعلى)
```

الـ `is_generic = true` يوفر عليك كتابة مئات الأسطر من الـ boilerplate code، ويجعل الـ Device Tree الخاص بك متوافقاً مع الـ standard properties التي يفهمها كل مطور Linux.