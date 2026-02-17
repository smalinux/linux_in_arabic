# شرح `pinconf.h` — `struct pinconf_ops` بالتفصيل

هذا الملف يُعرّف vtable واحدة: `struct pinconf_ops`، المسؤولة عن **قراءة وكتابة الخصائص الكهربائية** للـ pins.

---

## أولاً: `is_generic`

```c
#ifdef CONFIG_GENERIC_PINCONF
    bool is_generic;
#endif
```

الـ Linux kernel يوفر طريقتين لتنفيذ pin configuration:

**الطريقة المخصصة (custom):** الـ driver يُعرّف encoding خاص به للـ config values، ويفسرها بنفسه بالكامل.

**الطريقة العامة (generic):** الـ driver يضع `is_generic = true` ويستخدم الـ `pinconf-generic` framework الموجود في الـ kernel، الذي يُوفّر encoding جاهزاً لأغلب الخصائص الشائعة (pull-up، drive-strength، إلخ) عبر `PIN_CONFIG_*` macros في `pinconf-generic.h`.

في أغلب الـ modern drivers على Rockchip وAllwinner تجد `is_generic = true` لأنه يوفر وقتاً كبيراً ويُسهّل الـ Device Tree parsing تلقائياً.

---

## ثانياً: Get و Set — لـ Pin منفرد

### `pin_config_get(pctldev, pin, *config)`

يقرأ الـ configuration الحالية لـ pin واحد ويضعها في `*config`. القيمة المُرجَعة هي `unsigned long` تحمل الـ encoding الخاص بالـ driver (أو الـ generic encoding لو `is_generic = true`).

قواعد الـ return values محددة بدقة في التعليق:

- `-ENOTSUPP` → هذا النوع من الـ config غير مدعوم على هذا الـ controller أصلاً
- `-EINVAL` → مدعوم لكن غير مُفعَّل حالياً على هذا الـ pin

```c
static int foo_pin_config_get(struct pinctrl_dev *pctldev,
                              unsigned int pin,
                              unsigned long *config)
{
    unsigned int param = pinconf_to_config_param(*config);
    unsigned int val;

    switch (param) {
    case PIN_CONFIG_BIAS_PULL_UP:
        val = read_pull_reg(pin);
        *config = pinconf_to_config_packed(param, val);
        return 0;
    default:
        return -ENOTSUPP;
    }
}
```

### `pin_config_set(pctldev, pin, *configs, num_configs)`

الفرق المهم هنا عن `get`: الـ `set` يقبل **مصفوفة** من الـ configs دفعة واحدة (`num_configs` عنصر). هذا منطقي لأنك في الغالب تريد ضبط عدة خصائص معاً في نفس الوقت: pull-up + drive-strength + slew-rate مثلاً.

```c
static int foo_pin_config_set(struct pinctrl_dev *pctldev,
                              unsigned int pin,
                              unsigned long *configs,
                              unsigned int num_configs)
{
    for (int i = 0; i < num_configs; i++) {
        unsigned int param = pinconf_to_config_param(configs[i]);
        unsigned int arg   = pinconf_to_config_argument(configs[i]);

        switch (param) {
        case PIN_CONFIG_BIAS_PULL_UP:
            write_pull_reg(pin, arg);
            break;
        case PIN_CONFIG_DRIVE_STRENGTH:
            write_drive_reg(pin, arg);
            break;
        default:
            return -ENOTSUPP;
        }
    }
    return 0;
}
```

---

## ثالثاً: Get و Set — لـ Group كامل

### `pin_config_group_get(pctldev, selector, *config)`

### `pin_config_group_set(pctldev, selector, *configs, num_configs)`

نفس فكرة الـ per-pin callbacks لكن تعمل على **group كامل** (محدد بالـ `selector` وليس باسم الـ pin).

لماذا يوجد هذا؟ لأن بعض الـ hardware تملك registers مشتركة لمجموعة pins بالكامل. بدلاً من استدعاء `pin_config_set()` على كل pin بشكل منفرد (وهو ما سيفعله الـ core تلقائياً كـ fallback)، يمكن للـ driver كتابة register واحد يؤثر على الـ group كله دفعة واحدة — أكثر كفاءة وأحياناً ضروري للصحة الكهربائية.

نفس قواعد الـ return codes (`-ENOTSUPP` و`-EINVAL`) تنطبق هنا.

---

## رابعاً: Debugfs Callbacks الثلاثة

الثلاثة اختيارية تماماً، غرضها فقط إظهار معلومات مفيدة في `/sys/kernel/debug/pinctrl/`.

### `pin_config_dbg_show(pctldev, s, offset)`

يُطبع معلومات config لـ pin واحد في الـ debugfs. الـ `s` هو `seq_file` للكتابة فيه، و`offset` هو رقم الـ pin. يظهر في ملف `pinconf-pins`.

### `pin_config_group_dbg_show(pctldev, s, selector)`

نفس الفكرة لكن لـ group كامل. يظهر في ملف `pinconf-groups`.

### `pin_config_config_dbg_show(pctldev, s, config)`

هذا مختلف عن الاثنين السابقين: يأخذ قيمة config خام (`unsigned long`) ويُفسّرها بشكل بشري مقروء. مفيد جداً في الـ debugging لأن الـ config value عادةً مجرد رقم مشفّر غير مفهوم بدون تفسير.

مثال على ما قد يُطبعه:

```
config: 0x00040001 → BIAS_PULL_UP, drive-strength=4mA
```

---

## ملخص — متى تُنفّذ ماذا؟

|الـ Callback|إلزامي؟|ملاحظة|
|---|---|---|
|`pin_config_get`|شبه إلزامي|الـ core يحتاجه للتحقق والـ debugfs|
|`pin_config_set`|إلزامي|القلب الفعلي للـ pinconf|
|`pin_config_group_get`|اختياري|الـ core يستخدم per-pin كـ fallback|
|`pin_config_group_set`|اختياري|لكنه مُفضّل لو الـ HW يدعمه|
|`pin_config_dbg_show`|اختياري|للـ debugging فقط|
|`pin_config_group_dbg_show`|اختياري|للـ debugging فقط|
|`pin_config_config_dbg_show`|اختياري|مفيد جداً لتفسير الـ raw values|

الفرق الجوهري بين هذا الملف و`pinmux.h`: الـ pinmux يتعامل مع **توجيه الإشارات** (أي وظيفة تخرج من الـ pin)، أما الـ pinconf فيتعامل مع **الخصائص الكهربائية** للـ pin بغض النظر عن وظيفته.

# شرح عميق: `pinconf.h` — الـ Internal Interface للـ Pin Configuration

> هذا الملف هو **الواجهة الداخلية** بين الـ pinctrl core وجزء الـ pin configuration. مش للـ drivers العادية — ده للـ kernel internals بس.

---

## أولاً: إيه دور الملف ده؟

```
┌─────────────────────────────────────────────────────┐
│                  Consumer Driver                     │
│         (uart, spi, i2c, ...)                        │
└──────────────────────┬──────────────────────────────┘
                       │  pinctrl_select_state()
                       ▼
┌─────────────────────────────────────────────────────┐
│               Pinctrl Core                           │
│           drivers/pinctrl/core.c                     │
│                                                      │
│   بيستخدم functions من pinconf.h                    │
│   عشان يطبّق الـ config settings                    │
└──────────────────────┬──────────────────────────────┘
                       │  pinconf_apply_setting()
                       ▼
┌─────────────────────────────────────────────────────┐
│              SoC Pinctrl Driver                      │
│   (rockchip, stm32, ...)                             │
│   بينفّذ pinconf_ops->pin_config_set()              │
└─────────────────────────────────────────────────────┘
```

الملف ده هو **الجسر** بين الـ core والـ SoC driver في جزء الـ configuration.

---

## ثانياً: الـ `#ifdef CONFIG_PINCONF` — ليه؟

الكود مقسّم بـ `#ifdef` لسبب مهم جداً:

```c
#ifdef CONFIG_PINCONF
    // الكود الحقيقي
    int pinconf_check_ops(...);
    int pinconf_apply_setting(...);
#else
    // Stubs فاضية بترجع 0
    static inline int pinconf_check_ops(...) { return 0; }
    static inline int pinconf_apply_setting(...) { return 0; }
#endif
```

### ليه الـ stubs دي مهمة؟

```
لو CONFIG_PINCONF=n  (الـ SoC مش محتاج pin config)
    ↓
الـ compiler بيشيل كل الكود ده من الـ binary
    ↓
مفيش overhead خالص
    ↓
بس الـ core code بيفضل يكمّل بدون ما يـ crash
```

ده نمط شائع جداً في الـ kernel اسمه **"compile-time feature toggling"**.

---

## ثالثاً: شرح كل function بالتفصيل

### 🔵 `pinconf_check_ops()`

```c
int pinconf_check_ops(struct pinctrl_dev *pctldev);
```

**بتعمل إيه؟** لما SoC driver بيسجّل نفسه، الـ core بيعمل sanity check — "هل الـ `pinconf_ops` اللي بعتلي صح؟"

```
pinctrl_register()
    │
    ▼
pinconf_check_ops()  ← بيتأكد إن:
    ├── لو `is_generic = true` → لازم `pin_config_set` موجود
    ├── لو `pin_config_get` موجود → `pin_config_set` لازم يبقى موجود
    └── مفيش تناقض في الـ ops
```

---

### 🔵 `pinconf_validate_map()`

```c
int pinconf_validate_map(const struct pinctrl_map *map, int i);
```

**بتعمل إيه؟** لما board/DT بيعرّف mapping، الـ core بيتحقق إن كل entry فيها valid.

```
pinctrl_register_mappings()
    │
    ▼
for each map entry:
    pinconf_validate_map(map, i)
        ├── النوع PIN_MAP_TYPE_CONFIGS_PIN أو CONFIGS_GROUP؟
        ├── فيه configs array؟
        └── الـ configs مش NULL؟
```

**مثال على map entry للـ configs:**

```c
{
    .dev_name  = "ff000000.uart",
    .name      = PINCTRL_STATE_DEFAULT,
    .type      = PIN_MAP_TYPE_CONFIGS_PIN,   // ← ده اللي بيتـ validate
    .ctrl_dev_name = "pinctrl",
    .data.configs = {
        .group_or_pin = "uart2_rx",
        .configs      = my_configs_array,
        .num_configs  = 2,
    }
}
```

---

### 🔵 `pinconf_map_to_setting()`

```c
int pinconf_map_to_setting(const struct pinctrl_map *map,
                           struct pinctrl_setting *setting);
```

**ده من أهم الـ functions!**

الـ `pinctrl_map` هو التعريف الـ static (زي DT أو board file). الـ `pinctrl_setting` هو الـ runtime representation اللي الـ core بيشتغل بيه.

```
pinctrl_map (static definition)
    │
    │  pinconf_map_to_setting()
    ▼
pinctrl_setting (runtime object)
    ├── .type    = PIN_MAP_TYPE_CONFIGS_PIN
    ├── .pctldev = pointer to SoC driver
    └── .data.configs:
            ├── .group_or_pin = pin number (resolved)
            ├── .configs      = unsigned long[]
            └── .num_configs  = N
```

بمعنى: بيحوّل الاسم "uart2_rx" لـ pin number حقيقي.

---

### 🔵 `pinconf_free_setting()`

```c
void pinconf_free_setting(const struct pinctrl_setting *setting);
```

بسيطة — بتحرّر الـ `configs` array اللي اتعمل له `kmalloc` في `pinconf_map_to_setting()`.

```
pinctrl_setting
    └── data.configs.configs  ← kfree() ده
```

---

### 🔵 `pinconf_apply_setting()` — ⭐ الأهم

```c
int pinconf_apply_setting(const struct pinctrl_setting *setting);
```

**ده اللي بيحصل فعلاً على الـ hardware!**

```
pinconf_apply_setting(setting)
    │
    ├── لو النوع PIN_MAP_TYPE_CONFIGS_PIN:
    │       pinconf_set_config(pctldev, pin_number, configs, nconfigs)
    │           │
    │           └── pctldev->desc->confops->pin_config_set()
    │                   │
    │                   └── SoC driver بيكتب في الـ registers ✓
    │
    └── لو النوع PIN_MAP_TYPE_CONFIGS_GROUP:
            pinconf_ops->pin_config_group_set()
                │
                └── بيعمل config لكل pins في الـ group دفعة واحدة
```

---

### 🔵 `pinconf_set_config()`

```c
int pinconf_set_config(struct pinctrl_dev *pctldev, unsigned int pin,
                       unsigned long *configs, size_t nconfigs);
```

Helper مباشر — بيكلّم `pin_config_set` على pin محدد بـ number مش اسم.

---

### 🔵 `pin_config_get_for_pin()` و `pin_config_group_get()`

```c
int pin_config_get_for_pin(struct pinctrl_dev *pctldev, unsigned int pin,
                           unsigned long *config);

int pin_config_group_get(const char *dev_name, const char *pin_group,
                         unsigned long *config);
```

للـ **قراءة** — إيه الـ config الحالية للـ pin أو الـ group؟ بيُستخدم في الـ debugfs عشان يعرض الإعدادات الحالية.

---

## رابعاً: الـ `#ifdef CONFIG_DEBUG_FS` section

```c
#if defined(CONFIG_PINCONF) && defined(CONFIG_DEBUG_FS)
void pinconf_show_map(struct seq_file *s, const struct pinctrl_map *map);
void pinconf_show_setting(struct seq_file *s, const struct pinctrl_setting *setting);
void pinconf_init_device_debugfs(struct dentry *devroot, struct pinctrl_dev *pctldev);
#endif
```

```
/sys/kernel/debug/pinctrl/<controller>/
    │
    ├── pinconf-pins    ← pinconf_show_setting() لكل pin
    └── pinconf-groups  ← pinconf_show_setting() لكل group
```

الـ `pinconf_init_device_debugfs()` بتتسمّى مرة واحدة وقت تسجيل الـ controller وبتعمل الـ files دي.

---

## خامساً: الـ Generic Pinconf — الجزء الأكثر أهمية عملياً

```c
#if defined(CONFIG_GENERIC_PINCONF) && defined(CONFIG_OF)
int pinconf_generic_parse_dt_config(...);
int pinconf_generic_parse_dt_pinmux(...);
#endif
```

### `pinconf_generic_parse_dt_config()` — ⭐⭐ الأهم عملياً

```c
int pinconf_generic_parse_dt_config(struct device_node *np,
                                    struct pinctrl_dev *pctldev,
                                    unsigned long **configs,
                                    unsigned int *nconfigs);
```

**ده بيقرأ الـ DT node وبيحوّله لـ `configs` array.**

```
DT node:
    bias-pull-up;
    drive-strength = <8>;
    input-schmitt-enable;
        │
        ▼
pinconf_generic_parse_dt_config()
        │
        ▼
configs[] = {
    PIN_CONF_PACKED(PIN_CONFIG_BIAS_PULL_UP, 1),
    PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 8),
    PIN_CONF_PACKED(PIN_CONFIG_INPUT_SCHMITT_ENABLE, 1),
}
nconfigs = 3
```

**الـ encoding بتاع كل config value:**

```
unsigned long config:
┌─────────────────────┬──────────────────┐
│  bits [31:16]       │  bits [15:0]     │
│  argument (value)   │  param (type)    │
└─────────────────────┴──────────────────┘

مثال: drive-strength = <8>
PIN_CONF_PACKED(PIN_CONFIG_DRIVE_STRENGTH, 8)
= (8 << 16) | PIN_CONFIG_DRIVE_STRENGTH
```

الـ SoC driver بيفكّها بكده:

```c
param = pinconf_to_config_param(config);   // bits[15:0]
arg   = pinconf_to_config_argument(config); // bits[31:16]
```

---

### `pinconf_generic_parse_dt_pinmux()` — للـ flat DT style

```c
int pinconf_generic_parse_dt_pinmux(struct device_node *np, struct device *dev,
                                    unsigned int **pid, unsigned int **pmux,
                                    unsigned int *npins);
```

بيُستخدم مع الـ DT style اللي بيحدد mux و config في نفس الـ node:

```dts
/* Flat style (بعض الـ SoCs زي Allwinner) */
pinctrl-single,pins = <PIN_PA0  MUX_UART>;
```

بيرجع:

- `pid[]` = array of pin IDs
- `pmux[]` = array of mux values لكل pin

---

### `pinctrl_generic_pins_function_dt_node_to_map()`

```c
int pinctrl_generic_pins_function_dt_node_to_map(struct pinctrl_dev *pctldev,
                                                  struct device_node *np,
                                                  struct pinctrl_map **maps,
                                                  unsigned int *num_maps);
```

ده الـ **all-in-one** parser — بيعمل كل حاجة في خطوة واحدة:

```
DT node
    │
    ▼
pinctrl_generic_pins_function_dt_node_to_map()
    ├── بيقرأ الـ mux info   → PIN_MAP_TYPE_MUX_GROUP entries
    ├── بيقرأ الـ config info → PIN_MAP_TYPE_CONFIGS_PIN entries
    └── بيرجع maps[] جاهزة للـ core
```

معظم الـ modern SoC drivers بتستخدمه مباشرةً في الـ `pinctrl_ops.dt_node_to_map`:

```c
static const struct pinctrl_ops my_pctrl_ops = {
    .dt_node_to_map = pinctrl_generic_pins_function_dt_node_to_map, // جاهز!
    .dt_free_map    = pinconf_generic_dt_free_map,
    ...
};
```

---

## سادساً: الصورة الكاملة — كيف تتكامل مع بعض

```
Boot time:
──────────
DT node (uart2m0-pins)
    │
    ▼
pinctrl_ops.dt_node_to_map()
    ├── pinconf_generic_parse_dt_config()   ← config properties
    └── ينشئ pinctrl_map[] entries

    │
    ▼
pinconf_validate_map()   ← sanity check
    │
    ▼
pinconf_map_to_setting() ← يحوّل الاسم لـ pin number

Runtime (عند probe أو state change):
─────────────────────────────────────
pinctrl_select_state("default")
    │
    ▼
pinconf_apply_setting()
    │
    ▼
pinconf_ops->pin_config_set()
    │
    ▼
✓ Register على الـ hardware اتكتب
```

---

## سابعاً: ليه فيه `pinconf_check_ops` بيرجع 0 في الـ stub؟

```c
// لو CONFIG_PINCONF=n
static inline int pinconf_check_ops(struct pinctrl_dev *pctldev)
{
    return 0;  // ← دايماً success
}
```

لأن لو مفيش PINCONF support، مفيش حاجة تتفحص أصلاً — الـ core بيعمل كأنه كل حاجة تمام وبيكمل.

الـ `pinconf_set_config` بس اللي بيرجع `-ENOTSUPP` في الـ stub:

```c
static inline int pinconf_set_config(...) { return -ENOTSUPP; }
```

لأنه لو حد حاول فعلاً يعمل config وـ PINCONF مش enabled — لازم يعرف إن الـ operation مش supported، مش إنها "نجحت".

---

## ملخص سريع

|Function|بتتسمّى امتى|بتعمل إيه|
|---|---|---|
|`pinconf_check_ops`|وقت register الـ driver|تتأكد الـ ops صح|
|`pinconf_validate_map`|وقت register الـ mappings|تتأكد كل entry صح|
|`pinconf_map_to_setting`|وقت `pinctrl_get()`|تحوّل map لـ runtime setting|
|`pinconf_free_setting`|وقت `pinctrl_put()`|تحرّر الـ memory|
|`pinconf_apply_setting`|وقت `pinctrl_select_state()`|تكتب على الـ hardware ✓|
|`pinconf_generic_parse_dt_config`|في `dt_node_to_map`|تقرأ DT config properties|