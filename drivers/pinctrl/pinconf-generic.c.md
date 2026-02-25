## Phase 1: الصورة الكبيرة ببساطة

### إيه هو الملف ده؟

`pinconf-generic.c` هو الـ **generic layer** اللي بيوفر لغة مشتركة لإعداد الـ pins في Linux. بدل ما كل driver يعمل parsing لـ Device Tree بنفسه ويخترع الكلام من الأول، الملف ده بيقول: "أنا عارف كل المصطلحات الـ standard، سلّمني الـ DT node وأنا أترجمها ليك لـ kernel config values."

### تشبيه من الواقع

تخيل إنك بتشتري مكيف، والمهندس بياخد spec sheet فيها: `voltage: 220V`, `current: 10A`, `frequency: 50Hz`. كل شركة ممكن تكتب الـ spec بأسلوبها، لكن في عالم الكهرباء في **IEC standard** بيحدد الكلام بالظبط.

`pinconf-generic.c` هو الـ IEC standard ده للـ pin configuration:
- `bias-pull-up` = اوصل مقاومة للـ VDD
- `drive-strength = 8` = الـ pin يدفع 8mA
- `input-schmitt-enable` = فعّل الـ Schmitt trigger

بدله كل driver يترجم الـ DT بنفسه، الـ framework بيترجم الـ strings دي لـ `unsigned long` packed values يعرفها الـ hardware driver.

### ليه موجود؟

قبل الـ generic framework، كل vendor driver (Qualcomm, NXP, Broadcom...) كان بيكتب نفس الكود 3 مرات:
1. يقرأ `bias-pull-up` من DT
2. يحوّله لـ internal enum
3. يكتبه لـ hardware register

ده **code duplication** بالآلاف من الأسطر. الملف ده حلّ المشكلة بـ single shared parser.

### الملفات المرتبطة

| الملف | الدور |
|---|---|
| `include/linux/pinctrl/pinconf-generic.h` | enum `pin_config_param` + struct definitions + inline pack/unpack helpers |
| `include/linux/pinctrl/pinconf.h` | `struct pinconf_ops` — الـ vtable اللي الـ drivers بتـ implement |
| `drivers/pinctrl/core.c` | الـ pinctrl core، يسجّل الـ devices ويدير الـ lifecycle |
| `drivers/pinctrl/pinconf.c` | الـ pinconf core — يـ dispatch الـ config calls للـ driver |
| `drivers/pinctrl/devicetree.c` | الـ DT map parsing العام |
| `drivers/pinctrl/pinctrl-utils.c` | helper functions لـ map reservation وإضافة entries |

### ملفات الـ subsystem

```
drivers/pinctrl/
├── core.c               ← قلب الـ pinctrl subsystem
├── core.h
├── pinconf.c            ← dispatcher للـ pin config operations
├── pinconf.h
├── pinconf-generic.c    ← *** هو ده ***
├── pinmux.c             ← الـ mux (function selection) layer
├── devicetree.c         ← DT integration عام
├── pinctrl-utils.c      ← map allocation helpers
├── pinctrl-utils.h
├── freescale/           ← vendor drivers (iMX, Vybrid...)
├── qualcomm/            ← vendor drivers (MSM, SC8180X...)
├── intel/               ← vendor drivers (Baytrail, Cannonlake...)
└── ...
```

---

## Phase 2: شرح الـ pinconf-generic Framework

### المشكلة اللي بيحلها

الـ Linux pinctrl subsystem بيدعم مئات الـ SoC controllers. كل controller عنده hardware registers مختلفة، لكن المفاهيم متشابهة: pull-up، drive strength، schmitt trigger. المشكلة في **3 مستويات**:

1. **DT parsing duplication**: كل driver يقرأ نفس الـ DT properties (`bias-pull-up`, `drive-strength`) بكود مختلف.
2. **debugfs fragmentation**: كل driver يطبع معلومات الـ config بصيغة مختلفة في `/sys/kernel/debug/pinctrl/`.
3. **API inconsistency**: مفيش تعريف موحّد لـ "ما هي الـ parameters المعيارية".

### مقاربة الحل

الملف بيعتمد على مبدأ **packed config encoding**:

```c
/* lower 8 bits = param type, upper 24 bits = value */
#define PIN_CONF_PACKED(p, a) ((a << 8) | ((unsigned long) p & 0xffUL))

/* encode */
config = pinconf_to_config_packed(PIN_CONFIG_BIAS_PULL_UP, 10000); // 10kΩ

/* decode */
param = pinconf_to_config_param(config);   // → PIN_CONFIG_BIAS_PULL_UP
value = pinconf_to_config_argument(config); // → 10000
```

كل config مجمّع في `unsigned long` واحد بيتنقل بين الـ layers.

### ASCII Architecture Diagram

```
Device Tree (.dts)
─────────────────
  pinctrl-0 {
    bias-pull-up = <10000>;
    drive-strength = <8>;
  };
         │
         ▼
┌─────────────────────────────────────────┐
│        pinconf-generic.c                │
│                                         │
│  dt_params[] ──► parse_dt_cfg()         │
│  (DT string → param enum)               │
│                                         │
│  pinconf_generic_dt_node_to_map()       │
│    └─► pinconf_generic_dt_subnode_to_map│
│          ├─► pinconf_generic_parse_dt_config()
│          └─► pinctrl_utils_add_map_configs()
│                                         │
│  [CONFIG_DEBUG_FS]                      │
│  conf_items[] ──► dump_one()            │
│  pinconf_generic_dump_pins()            │
└───────────────┬─────────────────────────┘
                │  pinctrl_map[]
                │  (packed unsigned long configs)
                ▼
┌─────────────────────────────────────────┐
│        pinconf.c  (dispatcher)          │
│  pin_config_set() / pin_config_get()    │
└───────────────┬─────────────────────────┘
                │  pinconf_ops vtable
                ▼
┌─────────────────────────────────────────┐
│   Vendor Driver (e.g. pinctrl-imx.c)   │
│   .pin_config_set()                     │
│   .pin_config_get()                     │
│   .is_generic = true                    │
└───────────────┬─────────────────────────┘
                │
                ▼
         Hardware Registers
```

### الـ Core Abstractions

#### 1. `dt_params[]` — جدول الترجمة DT → enum

```c
static const struct pinconf_generic_params dt_params[] = {
    /* DT property string   , kernel enum param         , default val */
    { "bias-pull-up",        PIN_CONFIG_BIAS_PULL_UP,    1 },
    { "drive-strength",      PIN_CONFIG_DRIVE_STRENGTH,  0 },
    { "input-schmitt-enable",PIN_CONFIG_INPUT_SCHMITT_ENABLE, 1 },
    /* 30+ entries total */
};
```

`struct pinconf_generic_params` بيربط الـ DT string بالـ kernel enum وبـ default value لو الـ property موجودة بدون قيمة.

#### 2. `parse_dt_cfg()` — الـ parser الأساسي

```c
static int parse_dt_cfg(struct device_node *np,
                        const struct pinconf_generic_params *params,
                        unsigned int count,
                        unsigned long *cfg,       /* output array */
                        unsigned int *ncfg)       /* entry counter */
{
    for (i = 0; i < count; i++) {
        /* try of_property_read_u32(), fallback to default_value */
        cfg[*ncfg] = pinconf_to_config_packed(par->param, val);
        (*ncfg)++;
    }
}
```

يمشي على كل known parameter، لو لقاه في الـ DT node يـ pack القيمة، لو ملقهوش يتجاوز.

#### 3. `conf_items[]` — جدول الـ debugfs display

```c
#ifdef CONFIG_DEBUG_FS
static const struct pin_config_item conf_items[] = {
    PCONFDUMP(PIN_CONFIG_BIAS_PULL_UP,    "input bias pull up", "ohms", true),
    PCONFDUMP(PIN_CONFIG_DRIVE_STRENGTH,  "output drive strength", "mA", true),
    /* ... */
};
```

بيستخدمه `pinconf_generic_dump_pins()` لطباعة إعدادات الـ pin في debugfs بصيغة human-readable.

#### 4. `pinconf_generic_parse_dt_pinmux()` — الـ pinmux packing

```c
/* pinmux property: bits[7:0] = mux function, bits[31:8] = pin ID */
pmux_t[i] = value & 0xff;          // mux selector
pid_t[i]  = (value >> 8) & 0xffffff; // pin identity
```

format خاص لـ packed pinmux entries في DT بيجمع الـ pin ID والـ mux function في `u32` واحد.

### ما يمتلكه vs ما يُفوّضه

| الـ Concern | المالك |
|---|---|
| تعريف الـ standard params (enum) | `pinconf-generic.h` |
| ترجمة DT strings → packed configs | **`pinconf-generic.c`** |
| بناء الـ `pinctrl_map[]` arrays | `pinctrl-utils.c` (مُفوَّض) |
| dispatch الـ config للـ driver | `pinconf.c` (مُفوَّض) |
| كتابة config للـ hardware | Vendor driver (مُفوَّض) |
| debugfs display للـ generic params | **`pinconf-generic.c`** |
| debugfs display للـ custom params | Vendor driver عبر `custom_conf_items` |
| DT node traversal | **`pinconf-generic.c`** (يمشي على subnodes) |

### الـ Extensibility Pattern

الـ drivers اللي تحتاج parameters غير standard بتضيف:

```c
/* in vendor driver descriptor */
.custom_params      = my_vendor_params,    /* extra DT properties */
.num_custom_params  = ARRAY_SIZE(my_vendor_params),
.custom_conf_items  = my_vendor_conf_items, /* debugfs display */
```

وتلقائياً `pinconf_generic_parse_dt_config()` و`pinconf_generic_dump_pins()` بيأخدوا الـ custom params في الاعتبار بدون تعديل في الـ framework نفسه.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### الـ Config Encoding — الـ Packed Format

كل configuration بتتحط في `unsigned long` واحدة:

```
 bit 31      bit 8  bit 7      bit 0
 +-----------+------+-----------+
 | argument  | ...  |   param   |
 | (24 bits) |      |  (8 bits) |
 +-----------+------+-----------+
```

**الـ Macros الجوهرية:**

| Macro / Inline | الوظيفة |
|---|---|
| `PIN_CONF_PACKED(p, a)` | يحزم param في bits[7:0] والـ argument في bits[31:8] |
| `pinconf_to_config_param(config)` | يسحب الـ param: `config & 0xFF` |
| `pinconf_to_config_argument(config)` | يسحب الـ arg: `(config >> 8) & 0xFFFFFF` |
| `pinconf_to_config_packed(param, arg)` | alias لـ `PIN_CONF_PACKED` |

---

### enum pin_config_param — جدول شامل

| القيمة | الـ DT Property | الوحدة | الـ Arg |
|---|---|---|---|
| `PIN_CONFIG_BIAS_BUS_HOLD` | `bias-bus-hold` | — | ignored |
| `PIN_CONFIG_BIAS_DISABLE` | `bias-disable` | — | ignored |
| `PIN_CONFIG_BIAS_HIGH_IMPEDANCE` | `bias-high-impedance` | — | ignored |
| `PIN_CONFIG_BIAS_PULL_DOWN` | `bias-pull-down` | Ω | !=0 → enable |
| `PIN_CONFIG_BIAS_PULL_PIN_DEFAULT` | `bias-pull-pin-default` | Ω | !=0 → enable |
| `PIN_CONFIG_BIAS_PULL_UP` | `bias-pull-up` | Ω | !=0 → enable |
| `PIN_CONFIG_DRIVE_OPEN_DRAIN` | `drive-open-drain` | — | ignored |
| `PIN_CONFIG_DRIVE_OPEN_SOURCE` | `drive-open-source` | — | ignored |
| `PIN_CONFIG_DRIVE_PUSH_PULL` | `drive-push-pull` | — | ignored |
| `PIN_CONFIG_DRIVE_STRENGTH` | `drive-strength` | mA | القيمة |
| `PIN_CONFIG_DRIVE_STRENGTH_UA` | `drive-strength-microamp` | uA | القيمة |
| `PIN_CONFIG_INPUT_DEBOUNCE` | `input-debounce` | usec | 0→off |
| `PIN_CONFIG_INPUT_ENABLE` | `input-enable` / `input-disable` | — | 1/0 |
| `PIN_CONFIG_INPUT_SCHMITT` | `input-schmitt` | custom | threshold |
| `PIN_CONFIG_INPUT_SCHMITT_ENABLE` | `input-schmitt-enable` / `...-disable` | — | 1/0 |
| `PIN_CONFIG_INPUT_SCHMITT_UV` | `input-schmitt-microvolts` | uV | القيمة |
| `PIN_CONFIG_MODE_LOW_POWER` | `low-power-enable` / `...-disable` | — | 1/0 |
| `PIN_CONFIG_MODE_PWM` | — | — | custom |
| `PIN_CONFIG_LEVEL` | `output-high` / `output-low` | — | 1/0 |
| `PIN_CONFIG_OUTPUT_ENABLE` | `output-enable` / `output-disable` | — | 1/0 |
| `PIN_CONFIG_OUTPUT_IMPEDANCE_OHMS` | `output-impedance-ohms` | Ω | القيمة |
| `PIN_CONFIG_PERSIST_STATE` | — | — | — |
| `PIN_CONFIG_POWER_SOURCE` | `power-source` | selector | القيمة |
| `PIN_CONFIG_SKEW_DELAY` | `skew-delay` | custom | القيمة |
| `PIN_CONFIG_SKEW_DELAY_INPUT_PS` | `skew-delay-input-ps` | ps | القيمة |
| `PIN_CONFIG_SKEW_DELAY_OUTPUT_PS` | `skew-delay-output-ps` | ps | القيمة |
| `PIN_CONFIG_SLEEP_HARDWARE_STATE` | `sleep-hardware-state` | — | ignored |
| `PIN_CONFIG_SLEW_RATE` | `slew-rate` | custom | القيمة |
| `PIN_CONFIG_END` | — | — | = 0x7F |
| `PIN_CONFIG_MAX` | — | — | = 0xFF (للـ custom params) |

---

### الـ Structs الأساسية

#### `struct pin_config_item` — للـ debugfs dump

```c
struct pin_config_item {
    const enum pin_config_param param;   /* رقم الـ config param */
    const char * const display;          /* اسم قابل للقراءة */
    const char * const format;           /* وحدة: "mA", "ohms", ... */
    bool has_arg;                        /* هل فيه قيمة رقمية؟ */
    const char * const *values;          /* مصفوفة أسماء (enum-like) */
    size_t num_values;                   /* عدد العناصر */
};
```

الـ Macro اللي بيبني entries بسرعة:
```c
#define PCONFDUMP(param, display, format, has_arg) \
    PCONFDUMP_WITH_VALUES(param, display, format, has_arg, NULL, 0)
```

#### `struct pinconf_generic_params` — للـ DT parsing

```c
struct pinconf_generic_params {
    const char * const property;   /* اسم الـ DT property مثلاً "bias-pull-up" */
    enum pin_config_param param;   /* الـ param المقابل */
    u32 default_value;             /* القيمة لما الـ property موجودة بدون value */
    const char * const *values;    /* للـ string-enum properties */
    size_t num_values;             /* عدد الـ strings */
};
```

#### `struct pinconf_ops` — واجهة الـ driver

```c
struct pinconf_ops {
    bool is_generic;                         /* علم: استخدم generic framework */
    int (*pin_config_get)(pctldev, pin, *config);
    int (*pin_config_set)(pctldev, pin, *configs, num_configs);
    int (*pin_config_group_get)(pctldev, selector, *config);
    int (*pin_config_group_set)(pctldev, selector, *configs, num_configs);
    void (*pin_config_dbg_show)(pctldev, seq_file, offset);
    void (*pin_config_group_dbg_show)(pctldev, seq_file, selector);
    void (*pin_config_config_dbg_show)(pctldev, seq_file, config);
};
```

#### `struct pinctrl_map` — وحدة الـ mapping

```c
struct pinctrl_map {
    const char *dev_name;       /* الجهاز المستخدم */
    const char *name;           /* اسم الـ state مثلاً "default" */
    enum pinctrl_map_type type; /* CONFIGS_PIN أو CONFIGS_GROUP أو MUX_GROUP */
    const char *ctrl_dev_name;  /* اسم الـ pin controller */
    union {
        struct pinctrl_map_mux mux;         /* للـ MUX_GROUP */
        struct pinctrl_map_configs configs;  /* للـ CONFIGS_* */
    } data;
};
```

#### `enum pinctrl_map_type`

| القيمة | المعنى |
|---|---|
| `PIN_MAP_TYPE_INVALID` | غير محدد، الـ parser يستنتج |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمي (placeholder) |
| `PIN_MAP_TYPE_MUX_GROUP` | تحديد الـ mux function لـ group |
| `PIN_MAP_TYPE_CONFIGS_PIN` | config لـ pin واحد |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | config لـ group من الـ pins |

---

### ASCII Struct Relationship Diagram

```
  Device Tree Node (np)
         |
         | of_property_read_u32 / fwnode_property_match
         v
  pinconf_generic_params[]   ──────────────────────────────┐
  (dt_params[] + custom_params[])                          |
         |                                                 |
         | parse_dt_cfg()                                  |
         v                                                 |
  unsigned long cfg[]  (packed: [arg:24|param:8])          |
         |                                                 |
         | kmemdup                                         |
         v                                                 |
  unsigned long *configs  ───────────────────────────────┐ |
         |                                               | |
         v                                               v v
  pinctrl_map_configs                           pin_config_item[]
  { .group_or_pin                               (conf_items[] للـ debugfs)
    .configs  ──► [packed_cfg, ...]
    .num_configs }
         |
         v
  pinctrl_map
  { .type = CONFIGS_PIN/GROUP
    .data.configs = ... }
         |
         v
  pinconf_ops.pin_config_set()  ◄── الـ driver ينفذ
```

---

### Lifecycle + Call Flow Diagrams

#### 1. DT Boot-time Config Parsing

```
  kernel boot
      │
      ▼
  pinctrl_dt_to_map()                    [core.c]
      │
      ▼
  pinconf_generic_dt_node_to_map()       [pinconf-generic.c]
      │
      ├──► pinconf_generic_dt_subnode_to_map()  ← للـ root node
      │
      └──for_each_available_child──►
           pinconf_generic_dt_subnode_to_map()  ← لكل child node
               │
               ├── of_property_count_strings("pins" or "groups")
               ├── of_property_read_string("function")
               ├── pinconf_generic_parse_dt_config()
               │       │
               │       └── parse_dt_cfg(dt_params)
               │       └── parse_dt_cfg(custom_params)   [إن وُجد]
               │
               ├── pinctrl_utils_reserve_map()
               ├── pinctrl_utils_add_map_mux()            [إن كان function موجود]
               └── pinctrl_utils_add_map_configs()        [إن كانت configs موجودة]
```

#### 2. debugfs Read Flow

```
  cat /sys/kernel/debug/pinctrl/<dev>/pins
      │
      ▼
  pinconf_show_pins()                    [pinconf.c]
      │
      ▼
  pinconf_generic_dump_pins()            [pinconf-generic.c]
      │
      ├── check ops->is_generic
      ├── pinconf_generic_dump_one(conf_items)          ← standard params
      └── pinconf_generic_dump_one(custom_conf_items)   ← custom params
              │
              └── per-item:
                  pin_config_group_get() OR pin_config_get_for_pin()
                  → seq_printf(display + value + unit)
```

#### 3. pinmux Parsing (DT → arrays)

```
  DT: pinmux = <0x00010002 0x00020003>;
                 ^^^^^^^^  ^^
                 pid=1      mux=2
                 pid=2      mux=3
      │
      ▼
  pinconf_generic_parse_dt_pinmux()
      │
      ├── of_find_property("pinmux")
      ├── devm_kcalloc(pid_t[], pmux_t[])
      └── loop:
          value = prop[i]
          pmux_t[i] = value & 0xFF          ← bits[7:0]
          pid_t[i]  = (value >> 8) & 0xFFFFFF ← bits[31:8]
```

---

### Locking Strategy

الملف ده **بدون locks صريحة** — كل العمليات read-only على static tables أو allocation-then-return:

| العملية | الـ Locking |
|---|---|
| `dt_params[]` / `conf_items[]` | static const — no lock needed |
| `parse_dt_cfg()` | تشتغل خلال boot/probe فقط، single-threaded |
| `pinconf_generic_parse_dt_config()` | يعمل `kcalloc` + `kmemdup` — caller مسؤول عن الـ free |
| `pinconf_generic_dump_pins()` | debugfs context — يستدعي `pin_config_get` اللي بيأخذ driver lock |
| `pinconf_generic_dt_subnode_to_map()` | probe context — `pinctrl_utils_reserve_map` بيدير الـ map array |

---

## Phase 4: شرح الـ Functions

### API Summary Table

| الدالة | Export | الغرض |
|---|---|---|
| `pinconf_generic_dump_pins` | EXPORT (ifdef DEBUG_FS) | طباعة configs لـ pin/group في debugfs |
| `pinconf_generic_dump_config` | `EXPORT_SYMBOL_GPL` | طباعة config واحدة packed في debugfs |
| `pinconf_generic_parse_dt_pinmux` | `EXPORT_SYMBOL_GPL` | تحليل pinmux property من DT |
| `pinconf_generic_parse_dt_config` | `EXPORT_SYMBOL_GPL` | تحليل pinconf properties من DT node |
| `pinconf_generic_dt_subnode_to_map` | `EXPORT_SYMBOL_GPL` | تحويل DT subnode لـ pinctrl_map entries |
| `pinconf_generic_dt_node_to_map` | `EXPORT_SYMBOL_GPL` | entry point لـ DT → map كامل |
| `pinconf_generic_dt_free_map` | `EXPORT_SYMBOL_GPL` | تحرير الـ map array |
| `parse_dt_cfg` | static | helper داخلي لـ DT parsing |
| `pinconf_generic_dump_one` | static | helper داخلي للـ debugfs |

---

### Category 1: الـ debugfs Helpers

#### `pinconf_generic_dump_one`

```c
static void pinconf_generic_dump_one(struct pinctrl_dev *pctldev,
                                     struct seq_file *s, const char *gname,
                                     unsigned int pin,
                                     const struct pin_config_item *items,
                                     int nitems, int *print_sep)
```

دالة static داخلية بتلف على مصفوفة `pin_config_item` وتستعلم عن كل parameter من الـ hardware. بتستخدم `gname` للاستعلام عن group أو `pin` للاستعلام عن pin واحد. بتتجاهل الأخطاء العادية (`-EINVAL`, `-ENOTSUPP`) وتطبع بس القيم الموجودة فعلاً.

**Parameters:**
| المعامل | النوع | الوصف |
|---|---|---|
| `pctldev` | `struct pinctrl_dev *` | الـ pin controller device |
| `s` | `struct seq_file *` | الـ debugfs output stream |
| `gname` | `const char *` | اسم الـ group (أو NULL للـ pin mode) |
| `pin` | `unsigned int` | رقم الـ pin (مستخدم لو gname=NULL) |
| `items` | `const struct pin_config_item *` | مصفوفة الـ params للـ dump |
| `nitems` | `int` | عدد العناصر |
| `print_sep` | `int *` | state للـ comma separator (in/out) |

**Return:** void

**Locking:** بيستدعي `pin_config_group_get` / `pin_config_get_for_pin` اللي بيأخذوا الـ driver's own lock.

**Pseudocode:**
```
for each item in items[0..nitems-1]:
    config = pack(item.param, 0)
    if gname:
        ret = pin_config_group_get(dev_name, gname, &config)
    else:
        ret = pin_config_get_for_pin(pctldev, pin, &config)

    if ret == -EINVAL or -ENOTSUPP:
        continue          // param غير مدعوم أو مش موجود

    if ret != 0:
        print "ERROR READING CONFIG SETTING i"
        continue

    if print_sep: print ", "
    print_sep = 1
    print item.display

    if item.has_arg:
        val = extract_arg(config)
        if item.format:  print "(val unit)"
        else:            print "(0xval)"
        if item.values:  print "\"name\""
```

---

#### `pinconf_generic_dump_pins`

```c
void pinconf_generic_dump_pins(struct pinctrl_dev *pctldev,
                               struct seq_file *s,
                               const char *gname, unsigned int pin)
```

الـ entry point العام للـ debugfs pin/group dump. بتتأكد إن الـ driver مسجل كـ generic (`ops->is_generic`)، وبعدين تستدعي `pinconf_generic_dump_one` مرتين: مرة للـ standard params ومرة للـ custom params لو موجودين.

**Parameters:**
| المعامل | الوصف |
|---|---|
| `pctldev` | الـ pin controller |
| `s` | الـ seq_file output |
| `gname` | اسم الـ group أو NULL |
| `pin` | رقم الـ pin أو 0 |

**Return:** void

**Caller:** `pinconf_show_pins()` و `pinconf_show_pin_groups()` في `pinconf.c`.

**Guard:** `if (!ops->is_generic) return;` — بتخرج فوراً لو الـ driver مش generic.

---

#### `pinconf_generic_dump_config`

```c
void pinconf_generic_dump_config(struct pinctrl_dev *pctldev,
                                 struct seq_file *s, unsigned long config)
```

بتفك تشفير `unsigned long config` واحدة وتطبع اسمها وقيمتها في الـ seq_file. بتدور على `conf_items[]` الـ standard وكمان `custom_conf_items` لو موجودين. مستخدمة من `pin_config_config_dbg_show` callback.

**Parameters:**
| المعامل | الوصف |
|---|---|
| `pctldev` | الـ pin controller |
| `s` | الـ seq_file output |
| `config` | الـ packed config value |

**Return:** void — `EXPORT_SYMBOL_GPL`

**Locking:** لا يحتاج lock — بيقرأ فقط من static tables ومن `pctldev->desc`.

---

### Category 2: الـ DT Parsing Helpers

#### `parse_dt_cfg` (static)

```c
static int parse_dt_cfg(struct device_node *np,
                         const struct pinconf_generic_params *params,
                         unsigned int count, unsigned long *cfg,
                         unsigned int *ncfg)
```

القلب الفعلي للـ DT config parsing. بيلف على مصفوفة `params` ولكل entry بيحاول يقرأ الـ DT property المقابلة. لو الـ property string-enum بيستخدم `fwnode_property_match_property_string`، غير كده `of_property_read_u32`. لو الـ property غايبة بيحط `default_value`. النتيجة بتتحط في `cfg[*ncfg]` وبيزود العداد.

**Parameters:**
| المعامل | الوصف |
|---|---|
| `np` | الـ DT node |
| `params` | مصفوفة الـ param descriptors |
| `count` | عدد الـ params |
| `cfg` | مصفوفة الـ output (مخصصة مسبقاً) |
| `ncfg` | عداد الـ output entries (in/out) |

**Return:** `0` للنجاح، `-ENOENT` لو `fwnode_property_match` رجع `-ENOENT`.

**Locking:** لا يحتاج lock — probe/init context.

**Pseudocode:**
```
for i in 0..count-1:
    par = &params[i]

    if par->values != NULL:
        ret = fwnode_property_match_property_string(np, par->property,
                                                    par->values, par->num_values)
        if ret == -ENOENT: return -ENOENT
        if ret >= 0: val = ret; ret = 0
    else:
        ret = of_property_read_u32(np, par->property, &val)

    if ret == -EINVAL:  // property غير موجودة
        continue

    if ret != 0:        // property موجودة بدون value → default
        val = par->default_value

    cfg[*ncfg] = pack(par->param, val)
    (*ncfg)++

return 0
```

---

#### `pinconf_generic_parse_dt_pinmux`

```c
int pinconf_generic_parse_dt_pinmux(struct device_node *np,
                                     struct device *dev,
                                     unsigned int **pid,
                                     unsigned int **pmux,
                                     unsigned int *npins)
```

بتقرأ `pinmux` property من الـ DT وتفكها لمصفوفتين: `pid[]` للـ pin IDs و`pmux[]` للـ mux values. الـ encoding: كل u32 بيحتوي على `mux[7:0]` و `pin_id[31:8]`. بتخصص الـ arrays باستخدام `devm_kcalloc` فالـ cleanup تلقائي.

**Parameters:**
| المعامل | الوصف |
|---|---|
| `np` | الـ DT node |
| `dev` | الـ device للـ devm allocation |
| `pid` | output: مصفوفة pin IDs |
| `pmux` | output: مصفوفة mux values |
| `npins` | output: عدد الـ pins |

**Return:** `0` للنجاح، `-ENOENT` لو `pinmux` property غايبة، `-EINVAL` لو parameters خاطئة، `-ENOMEM` لو allocation فشل.

**Error Handling:** على الـ allocation error بيعمل `devm_kfree` للمصفوفتين قبل الرجوع.

**Caller:** driver-specific probe functions اللي بتدعم pinmux encoding.

**Encoding:**
```
u32 value = pinmux[i]
pmux_t[i] = value & 0xFF          // mux selector
pid_t[i]  = (value >> 8) & 0xFFFFFF  // pin number
```

---

#### `pinconf_generic_parse_dt_config`

```c
int pinconf_generic_parse_dt_config(struct device_node *np,
                                     struct pinctrl_dev *pctldev,
                                     unsigned long **configs,
                                     unsigned int *nconfigs)
```

بتحول DT properties لـ `unsigned long` config array جاهز للاستخدام في `pinctrl_map`. بتخصص buffer مؤقت بحجم `ARRAY_SIZE(dt_params) + num_custom_params`، بتملاه عبر `parse_dt_cfg`، وبعدين بتعمل `kmemdup` بالحجم الفعلي بس. الـ caller مسؤول عن `kfree` للـ `*configs`.

**Parameters:**
| المعامل | الوصف |
|---|---|
| `np` | الـ DT node |
| `pctldev` | الـ pin controller (ممكن NULL لو standard params بس) |
| `configs` | output: مصفوفة الـ packed configs |
| `nconfigs` | output: عدد الـ configs |

**Return:** `0` للنجاح، `-EINVAL` لو `np=NULL`، `-ENOMEM` لو allocation فشل.

**لو `ncfg == 0`:** بيرجع `*configs=NULL, *nconfigs=0` مع `0` — مش error.

**Locking:** لا يحتاج lock — probe context فقط.

**Pseudocode:**
```
if !np: return -EINVAL

max_cfg = ARRAY_SIZE(dt_params)
if pctldev: max_cfg += pctldev->desc->num_custom_params

cfg = kcalloc(max_cfg, sizeof(unsigned long))
if !cfg: return -ENOMEM

parse_dt_cfg(np, dt_params, ..., cfg, &ncfg)

if pctldev && custom_params:
    parse_dt_cfg(np, custom_params, ..., cfg, &ncfg)

if ncfg == 0:
    *configs = NULL; *nconfigs = 0
    goto out

*configs = kmemdup(cfg, ncfg * sizeof(unsigned long))
*nconfigs = ncfg

out:
    kfree(cfg)
    return ret
```

---

### Category 3: الـ DT → Map Conversion

#### `pinconf_generic_dt_subnode_to_map`

```c
int pinconf_generic_dt_subnode_to_map(struct pinctrl_dev *pctldev,
                                       struct device_node *np,
                                       struct pinctrl_map **map,
                                       unsigned int *reserved_maps,
                                       unsigned int *num_maps,
                                       enum pinctrl_map_type type)
```

بتحول DT node واحد لـ `pinctrl_map` entries. بتدور على `pins` أو `groups` property، بتقرأ `function` الاختيارية، بتعمل parse للـ configs، وبتضيف مدخل لكل pin/group. الـ map array بتتوسع ديناميكياً عبر `pinctrl_utils_reserve_map`.

**Parameters:**
| المعامل | الوصف |
|---|---|
| `pctldev` | الـ pin controller |
| `np` | الـ DT subnode |
| `map` | مصفوفة الـ map (in/out) |
| `reserved_maps` | عدد الـ slots المحجوزة (in/out) |
| `num_maps` | عدد الـ entries الفعلية (in/out) |
| `type` | نوع الـ map أو `PIN_MAP_TYPE_INVALID` للاستنتاج التلقائي |

**Return:** `0` للنجاح، أو error code سالب.

**لو مافيش `pins` ولا `groups`:** بيرجع `0` بهدوء — الـ node ممكن يحتوي بس على child nodes.

**Locking:** لا يحتاج lock مباشر — `pinctrl_utils_reserve_map` بيدير الـ array realloc.

**Caller:** `pinconf_generic_dt_node_to_map`.

**Logic Flow:**
```
count_pins = of_property_count_strings(np, "pins")
if count_pins < 0:
    count_groups = of_property_count_strings(np, "groups")
    if count_groups < 0: return 0   // مش pinmux node
    type = CONFIGS_GROUP (لو INVALID)
    target = "groups"
else:
    type = CONFIGS_PIN (لو INVALID)
    target = "pins"

read optional "function" string

parse_dt_config → configs[], num_configs

reserve = (function ? 1 : 0) + (num_configs ? 1 : 0)
reserve *= strings_count

reserve_map(reserve)

for each group/pin in target:
    if function: add_map_mux(group, function)
    if configs:  add_map_configs(group, configs, num_configs, type)

kfree(configs)
```

---

#### `pinconf_generic_dt_node_to_map`

```c
int pinconf_generic_dt_node_to_map(struct pinctrl_dev *pctldev,
                                    struct device_node *np_config,
                                    struct pinctrl_map **map,
                                    unsigned int *num_maps,
                                    enum pinctrl_map_type type)
```

الـ entry point الرئيسي اللي بيستخدمه الـ driver في `pinctrl_ops.dt_node_to_map`. بتبدأ بـ map فاضية، بتعالج الـ root node نفسه، وبعدين بتلف على كل child nodes المتاحة.

**Parameters:**
| المعامل | الوصف |
|---|---|
| `pctldev` | الـ pin controller |
| `np_config` | الـ DT config node |
| `map` | output: مصفوفة الـ map المخصصة |
| `num_maps` | output: عدد الـ entries |
| `type` | نوع الـ map الإجباري أو `PIN_MAP_TYPE_INVALID` |

**Return:** `0` للنجاح، أو error code سالب مع cleanup تلقائي للـ map.

**على الـ error:** بيستدعي `pinctrl_utils_free_map` لتحرير كل حاجة.

**Variants (inline في الـ header):**

| الدالة | الـ type المُمرر |
|---|---|
| `pinconf_generic_dt_node_to_map_group` | `PIN_MAP_TYPE_CONFIGS_GROUP` |
| `pinconf_generic_dt_node_to_map_pin` | `PIN_MAP_TYPE_CONFIGS_PIN` |
| `pinconf_generic_dt_node_to_map_all` | `PIN_MAP_TYPE_INVALID` (auto-detect) |

**Caller:** الـ driver في `pinctrl_ops->dt_node_to_map`.

---

#### `pinconf_generic_dt_free_map`

```c
void pinconf_generic_dt_free_map(struct pinctrl_dev *pctldev,
                                  struct pinctrl_map *map,
                                  unsigned int num_maps)
```

wrapper مباشر فوق `pinctrl_utils_free_map`. بتتوافق مع `pinctrl_ops->dt_free_map` callback interface. تحرر الـ map array والـ configs الداخلية.

**Parameters:**
| المعامل | الوصف |
|---|---|
| `pctldev` | الـ pin controller (للـ logging) |
| `map` | الـ map array المراد تحريره |
| `num_maps` | عدد الـ entries |

**Return:** void — `EXPORT_SYMBOL_GPL`

**Caller:** pinctrl core عبر `pinctrl_ops->dt_free_map`.
## Phase 5: دليل الـ Debugging الشامل

### 5.1 debugfs Entries

الـ kernel بيعرض معلومات الـ pinconf عبر `debugfs` طالما `CONFIG_DEBUG_FS=y`.

**المسارات الأساسية:**

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/pinctrl/` | directory لكل pinctrl device |
| `/sys/kernel/debug/pinctrl/<dev>/pins` | قايمة الـ pins وأرقامها |
| `/sys/kernel/debug/pinctrl/<dev>/pinconf-pins` | الـ config الحالي لكل pin |
| `/sys/kernel/debug/pinctrl/<dev>/pinconf-groups` | الـ config لكل group |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-functions` | الـ functions المتاحة |
| `/sys/kernel/debug/pinctrl/<dev>/pinmux-pins` | مين محتاج مين |

**إزاي البيانات بتتجمع:**

`pinconf_generic_dump_pins()` هي الـ function اللي بتُملي الـ `seq_file` — بتلف على `conf_items[]` وبتقرأ كل parameter عبر `pin_config_get_for_pin()` أو `pin_config_group_get()`. لو الـ driver مش بيدعم parameter معين بيرجع `-EINVAL` أو `-ENOTSUPP` وبيتخطاه.

```bash
# اقرأ config كل الـ pins
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinconf-pins

# مثال output:
# pin 42 (gpio1-b2): input bias pull up (47000 ohms), input schmitt enabled
# pin 43 (gpio1-b3): output drive push pull, output drive strength (8 mA)
```

```bash
# اقرأ config الـ groups
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinconf-groups

# output:
# group: i2c2 (2 pins):
#   input bias pull up (47000 ohms)
```

**الدالة المسؤولة في الكود:**

```c
// pinconf-generic.c:123
void pinconf_generic_dump_pins(struct pinctrl_dev *pctldev, struct seq_file *s,
                               const char *gname, unsigned int pin)
{
    const struct pinconf_ops *ops = pctldev->desc->confops;
    int print_sep = 0;

    if (!ops->is_generic)  /* يتجاهل non-generic drivers */
        return;

    /* يطبع generic params أولاً */
    pinconf_generic_dump_one(pctldev, s, gname, pin, conf_items,
                             ARRAY_SIZE(conf_items), &print_sep);
    /* ثم custom params للـ driver */
    if (pctldev->desc->num_custom_params &&
        pctldev->desc->custom_conf_items)
        pinconf_generic_dump_one(pctldev, s, gname, pin,
                                 pctldev->desc->custom_conf_items,
                                 pctldev->desc->num_custom_params,
                                 &print_sep);
}
```

---

### 5.2 Sysfs Entries

الـ pinctrl مش ليه sysfs interface مباشر لأسباب أمنية وتصميمية — الـ kernel طلع قرار بإن debugfs هو الأنسب. لكن بعض الـ drivers بيضيفوا:

```bash
# GPIO sysfs (deprecated but still common)
ls /sys/class/gpio/
echo 42 > /sys/class/gpio/export
cat /sys/class/gpio/gpio42/direction
cat /sys/class/gpio/gpio42/value
```

---

### 5.3 ftrace Usage

**تتبع `parse_dt_cfg()` و `pinconf_generic_parse_dt_config()`:**

```bash
# تفعيل tracing للـ pinctrl functions
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo function > /sys/kernel/debug/tracing/current_tracer

# فلترة الـ functions المهمة
echo "pinconf_generic_parse_dt_config" > /sys/kernel/debug/tracing/set_ftrace_filter
echo "parse_dt_cfg" >> /sys/kernel/debug/tracing/set_ftrace_filter
echo "pinconf_generic_dt_subnode_to_map" >> /sys/kernel/debug/tracing/set_ftrace_filter

echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل probe أو أعد تحميل driver
modprobe pinctrl-rockchip

cat /sys/kernel/debug/tracing/trace
```

**مثال output:**

```
# tracer: function
#
            kworker/0:1-45    [000] ....   23.456789: pinconf_generic_parse_dt_config <-pinctrl_dt_to_map
            kworker/0:1-45    [000] ....   23.456791: parse_dt_cfg <-pinconf_generic_parse_dt_config
            kworker/0:1-45    [000] ....   23.456793: pinconf_generic_dt_subnode_to_map <-pinconf_generic_dt_node_to_map
```

**تتبع كل الـ pinctrl subsystem:**

```bash
echo "pinconf_generic_*" > /sys/kernel/debug/tracing/set_ftrace_filter
echo "pin_config_*" >> /sys/kernel/debug/tracing/set_ftrace_filter
```

**Function graph لمعرفة الـ call chain:**

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo "pinconf_generic_parse_dt_config" > /sys/kernel/debug/tracing/set_graph_function
cat /sys/kernel/debug/tracing/trace
```

```
 # tracer: function_graph
 #
 # CPU  DURATION                  FUNCTION CALLS
 # |     |   |                     |   |   |   |
  0)               |  pinconf_generic_parse_dt_config() {
  0)               |    parse_dt_cfg() {
  0)   0.234 us    |      of_property_read_u32();
  0)   0.891 us    |    }
  0)   1.200 us    |  }
```

---

### 5.4 Dynamic Debug

الـ `pr_debug()` في `parse_dt_cfg()` بتطبع لما تلاقي property في الـ DT:

```c
// pinconf-generic.c:254
pr_debug("found %s with value %u\n", par->property, val);
```

**التفعيل:**

```bash
# تفعيل dynamic debug للملف ده
echo "file pinconf-generic.c +p" > /sys/kernel/debug/dynamic_debug/control

# أو للـ module كله
echo "module pinctrl_rockchip +p" > /sys/kernel/debug/dynamic_debug/control

# تأكيد التفعيل
cat /sys/kernel/debug/dynamic_debug/control | grep pinconf-generic

# الـ output في dmesg:
dmesg | grep "generic pinconfig"
# [    2.345678] generic pinconfig core: found bias-pull-up with value 1
# [    2.345680] generic pinconfig core: found drive-strength with value 8
# [    2.345682] generic pinconfig core: found slew-rate with value 0
```

---

### 5.5 CONFIG_DEBUG_* Options المهمة

| Option | الأثر |
|--------|-------|
| `CONFIG_DEBUG_FS` | يفعّل `/sys/kernel/debug/pinctrl/` |
| `CONFIG_PINCTRL_ROCKCHIP` | driver خاص بـ RK3562 |
| `CONFIG_DEBUG_PINCTRL` | يضيف verbose logging في بعض الـ drivers |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug()` runtime |
| `CONFIG_FUNCTION_TRACER` | يفعّل ftrace |
| `CONFIG_KALLSYMS` | يظهر أسماء الـ functions في trace |
| `CONFIG_OF_DYNAMIC` | debugging لـ device tree |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "DEBUG_FS|PINCTRL|DYNAMIC_DEBUG"
```

---

### 5.6 جدول رسائل الـ Errors

| الرسالة | الملف/السطر | السبب | الحل |
|---------|------------|-------|------|
| `"Missing pinmux property"` | `pinconf-generic.c:286` | الـ DT node مش فيه `pinmux` property | ضيف `pinmux = <...>` للـ node |
| `"parameters error"` | `pinconf-generic.c:292` | `pid`/`pmux`/`npins` == NULL | تحقق من الـ caller |
| `"kalloc memory fail"` | `pinconf-generic.c:299` | OOM عند `devm_kcalloc` | زود الـ memory أو فحص الـ memory pressure |
| `"get pinmux value fail"` | `pinconf-generic.c:306` | `of_property_read_u32_index` فشل | تحقق من الـ DT syntax |
| `"could not parse property function"` | `pinconf-generic.c:422` | قيمة غير صحيحة لـ `function` | فحص الـ DT node |
| `"could not parse node property"` | `pinconf-generic.c:430` | فشل `pinconf_generic_parse_dt_config` | فحص الـ properties في DT |
| `"ERROR READING CONFIG SETTING %d"` | `pinconf-generic.c:85` | `pin_config_get_for_pin` رجع error غير متوقع | فحص `pinconf_ops->pin_config_get` |

---

### 5.7 Hardware-Level Debugging

**قراءة الـ registers مباشرة:**

```bash
# على RK3562: قراءة GRF_GPIO1B_IOMUX_H (عنوان افتراضي)
# devmem2 tool
devmem2 0xFD5F8000 w   # RK3562 IOMUX base

# على STM32MP1:
devmem2 0x50002000 w   # GPIOA MODER register
```

**oscilloscope / logic analyzer:**

- اتحقق من الـ pull resistor بـ multimeter: القياس بين الـ pin وVCC/GND
- Drive strength: قيس الـ rise time بالـ oscilloscope — كلما زاد الـ strength كلما قل الـ rise time

**GPIO toggle test:**

```bash
# export pin واختبار الـ output
echo 42 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio42/direction
echo 1 > /sys/class/gpio/gpio42/value
# قيس الـ voltage على الـ pin
echo 0 > /sys/class/gpio/gpio42/value
```

---

### 5.8 DT Debugging

**فحص الـ DT المحمل فعلاً:**

```bash
# شوف الـ pinctrl nodes
ls /proc/device-tree/pinctrl*/

# اقرأ property معينة
cat /proc/device-tree/pinctrl@fdc60000/i2c2-pins/bias-pull-up
# output: binary (big-endian u32)
hexdump -C /proc/device-tree/pinctrl@fdc60000/i2c2-pins/drive-strength

# dtc لتحويل الـ binary لـ readable
dtc -I fs /proc/device-tree/ 2>/dev/null | grep -A 10 "i2c2-pins"
```

**تحقق من الـ phandles:**

```bash
# شوف الـ pinctrl mappings
cat /sys/kernel/debug/pinctrl/pinctrl@fdc60000/pinmux-pins | grep i2c
```

**معرفة إيه الـ state الحالي:**

```bash
# شوف كل active states
cat /sys/kernel/debug/pinctrl/pinctrl@fdc60000/pinmux-functions
```

---

### 5.9 Shell Commands جاهزة للنسخ

```bash
#!/bin/bash
# === pinconf-generic.c Debug Toolkit ===

PCTLDEV=$(ls /sys/kernel/debug/pinctrl/ | head -1)
echo "=== Using: $PCTLDEV ==="

# 1. قايمة كل الـ pins
echo "--- All Pins ---"
cat /sys/kernel/debug/pinctrl/$PCTLDEV/pins

# 2. Config كل pin
echo "--- Pin Configs ---"
cat /sys/kernel/debug/pinctrl/$PCTLDEV/pinconf-pins

# 3. Config الـ groups
echo "--- Group Configs ---"
cat /sys/kernel/debug/pinctrl/$PCTLDEV/pinconf-groups

# 4. الـ functions المتاحة
echo "--- Functions ---"
cat /sys/kernel/debug/pinctrl/$PCTLDEV/pinmux-functions

# 5. تفعيل dynamic debug
echo "file pinconf-generic.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "Dynamic debug enabled for pinconf-generic.c"

# 6. dmesg للـ pin messages
dmesg | grep -i "pinconf\|pinctrl\|generic pinconfig" | tail -20
```

**مثال output كامل:**

```
=== Using: pinctrl@fdc60000 ===
--- All Pins ---
pin 0 (gpio0-a0)
pin 1 (gpio0-a1)
...
pin 42 (gpio1-b2)
--- Pin Configs ---
pin 0 (gpio0-a0): input bias disabled
pin 42 (gpio1-b2): input bias pull up (47000 ohms), slew rate (0)
--- Group Configs ---
group: i2c2 (2 pins):
  input bias pull up (47000 ohms)
  output drive strength (8 mA)
--- Functions ---
function: i2c2, groups = [ i2c2 ]
function: spi0, groups = [ spi0 ]
```

---

## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1 — RK3562: I2C بيتهنج عند Boot

**العنوان:** I2C2 على RK3562 مش شغال وكل الـ transactions بتفشل

**السياق:**
بورد مخصص بـ RK3562 SoC، الـ I2C2 موصل بـ EEPROM. الـ `i2c_transfer()` بترجع `-ETIMEDOUT` دايماً.

**المشكلة:**
الـ DT فيه:
```dts
&i2c2 {
    pinctrl-names = "default";
    pinctrl-0 = <&i2c2m0_xfer>;
};

i2c2m0_xfer: i2c2m0-xfer {
    rockchip,pins = <1 RK_PB2 1 &pcfg_pull_up_drv_level_0>;
};
```
لكن الـ SDA line دايماً low.

**التحليل عبر الكود:**

1. عند boot، الـ kernel بيستدعي `pinconf_generic_dt_node_to_map()` على الـ node ده.
2. الدالة بتستدعي `pinconf_generic_dt_subnode_to_map()` اللي بتقرأ الـ `pins` property.
3. `pinconf_generic_parse_dt_config()` بتلف على `dt_params[]` — بتدور على `bias-pull-up`, `drive-strength` إلخ.
4. `parse_dt_cfg()` بتقرأ `of_property_read_u32()` لكل property — هنا `bias-pull-up` موجودة.
5. الـ config بيتحول لـ `pinconf_to_config_packed(PIN_CONFIG_BIAS_PULL_UP, 1)`.
6. الـ Rockchip driver بيطبق القيمة في الـ GRF register.

**الخطأ:** الـ DT عامل pull-up لكن الـ PCB فيه pull-down خارجي قيمته 1kΩ قوي جداً، بيغلب الـ internal pull-up.

**التحقق:**

```bash
# تحقق من الـ config الفعلي
cat /sys/kernel/debug/pinctrl/pinctrl@fdc60000/pinconf-pins | grep "gpio1-b2"
# output: pin 42 (gpio1-b2): input bias pull up (47000 ohms)

# 47kΩ internal vs 1kΩ external → SDA بتظل low
dmesg | grep "generic pinconfig"
# [2.34] generic pinconfig core: found bias-pull-up with value 1
```

**الحل:**
شيل الـ pull-down الخارجي أو استبدله بـ 4.7kΩ pull-up، أو عدّل الـ DT:

```dts
i2c2m0_xfer: i2c2m0-xfer {
    rockchip,pins = <1 RK_PB2 1 &pcfg_pull_none>;
    /* اعتمد على الـ external pull-up فقط */
};
```

**الدرس المستفاد:**
`pinconf_generic_parse_dt_config()` بتطبق الـ config بأمانة بس مش بتعرف حاجة عن الـ external components — لازم تتحقق من الـ schematic مع الـ debugfs output.

---

### السيناريو 2 — STM32MP1: GPIO Interrupt مش بيشتغل

**العنوان:** PMIC interrupt على STM32MP1 مش واصل للـ kernel

**السياق:**
نظام مبني على STM32MP157، الـ STPMIC1 موصل بـ GPIO PI8. الـ IRQ handler مش بيتستدعى أبداً.

**المشكلة:**

```dts
/* DT خاطئ */
&pinctrl {
    pmic_pins: pmic-pins {
        pins = "PI8";
        bias-disable;          /* مفيش pull-up! */
        drive-push-pull;       /* خطأ للـ open-drain PMIC output */
    };
};
```

**التحليل عبر الكود:**

1. `pinconf_generic_dt_subnode_to_map()` بتشوف `pins = "PI8"` → type = `PIN_MAP_TYPE_CONFIGS_PIN`.
2. `pinconf_generic_parse_dt_config()` بتلف على `dt_params[]`:
   - بتلاقي `"bias-disable"` → `PIN_CONFIG_BIAS_DISABLE` بـ `default_value=0`
   - بتلاقي `"drive-push-pull"` → `PIN_CONFIG_DRIVE_PUSH_PULL` بـ `default_value=0`
3. `parse_dt_cfg()` بتعمل `pinconf_to_config_packed()` لكل منهم.
4. الـ STM32 driver بيطبق: no pull + push-pull mode.
5. الـ PMIC بيشيل الـ line وما فيش حاجة ترفعها → الـ MCU بيشوف voltage floating.

**التحقق:**

```bash
cat /sys/kernel/debug/pinctrl/soc:pinctrl/pinconf-pins | grep "pi8"
# pin 136 (PI8): input bias disabled, output drive push pull

# المشكلة واضحة: no pull-up, push-pull بدل open-drain input
```

**الحل:**

```dts
pmic_pins: pmic-pins {
    pins = "PI8";
    bias-pull-up;          /* pull-up للـ open-drain output */
    input-enable;          /* configure كـ input */
    /* لا drive-push-pull */
};
```

**الدرس المستفاد:**
`dt_params[]` في `pinconf-generic.c` بيقبل `bias-disable` و `drive-push-pull` في نفس الوقت بدون اعتراض — الـ validation الدراماتيكية مسؤولية الـ driver الـ hardware-specific. استخدم debugfs لتأكيد الـ config الفعلي قبل ما تصيح على الـ interrupt subsystem.

---

### السيناريو 3 — i.MX8: SPI Data Corruption بسرعات عالية

**العنوان:** SPI على i.MX8MQ بيدي data corruption عند 50MHz

**السياق:**
NXP i.MX8MQ مع SPI flash موصل بـ ECSPI1. الـ flash شغالة تمام بـ 25MHz لكن بتفشل بـ 50MHz.

**المشكلة:**

```dts
ecspi1_grp: ecspi1-grp {
    fsl,pins = <
        MX8MQ_IOMUXC_ECSPI1_SCLK  0x82   /* drive-strength missing! */
        MX8MQ_IOMUXC_ECSPI1_MOSI  0x82
        MX8MQ_IOMUXC_ECSPI1_MISO  0x82
        MX8MQ_IOMUXC_ECSPI1_SS0   0x82
    >;
};
```

**التحليل عبر الكود:**

على i.MX8، الـ driver بيستخدم `custom_params` بدلاً من `dt_params` العامة. الـ `pinconf_generic_parse_dt_config()` في `pinconf-generic.c` بتشوف الـ driver-specific params:

```c
/* pinconf-generic.c:355 */
if (pctldev && pctldev->desc->num_custom_params &&
    pctldev->desc->custom_params) {
    ret = parse_dt_cfg(np, pctldev->desc->custom_params,
                       pctldev->desc->num_custom_params, cfg, &ncfg);
}
```

الـ i.MX driver بيعرض `fsl,drive-strength` كـ custom param. لكن الـ raw value `0x82` بيطبق drive strength منخفض.

**التحقق:**

```bash
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pinconf-pins | grep "ecspi"
# pin 12 (ECSPI1_SCLK): output drive strength (2 mA)
# Drive strength 2mA + 50MHz + PCB capacitance = signal integrity failure
```

**الحل:**

```dts
ecspi1_grp: ecspi1-grp {
    fsl,pins = <
        MX8MQ_IOMUXC_ECSPI1_SCLK  0x86   /* drive strength أعلى */
        MX8MQ_IOMUXC_ECSPI1_MOSI  0x86
        MX8MQ_IOMUXC_ECSPI1_MISO  0x82   /* MISO: input لا يحتاج drive strength */
        MX8MQ_IOMUXC_ECSPI1_SS0   0x86
    >;
};
```

أو بالـ generic way:

```dts
ecspi1_sclk: ecspi1-sclk {
    pins = "ECSPI1_SCLK";
    drive-strength = <6>;   /* drive-strength في mA أو حسب الـ SoC */
    slew-rate = <0>;        /* fast slew */
};
```

**الدرس المستفاد:**
`pinconf_generic_dump_config()` (المصدر: `pinconf-generic.c:144`) بتعرض القيمة الـ raw. قارنها بالـ datasheet لتعرف الـ drive strength الفعلي. الـ custom_conf_items بتتعرض في debugfs مع الـ generic ones.

---

### السيناريو 4 — AM62x: UART Noise عند Sleep/Wake

**العنوان:** UART على TI AM62x بتستقبل garbage bytes بعد wake من suspend

**السياق:**
AM62x في product بيدخل deep sleep، الـ UART2 موصل بـ Bluetooth module. بعد wake-up الـ Bluetooth driver بيشتكي من framing errors.

**المشكلة:**

```dts
uart2_sleep_pins: uart2-sleep-pins {
    pinctrl-single,pins = <
        AM62X_IOPAD(0x0054, PIN_INPUT, 7)  /* RX: GPIO mode في sleep */
        AM62X_IOPAD(0x0058, PIN_OUTPUT_PULLDOWN, 7)  /* TX: pulled down */
    >;
    /* مفيش sleep-hardware-state! */
};
```

**التحليل عبر الكود:**

`dt_params[]` في `pinconf-generic.c:173` فيه:

```c
{ "sleep-hardware-state", PIN_CONFIG_SLEEP_HARDWARE_STATE, 0 },
```

ده بيقول للـ hardware يحتفظ بالـ state في الـ sleep. لما بيغيب من الـ DT، الـ pins بتتحرر وبتدخل floating state.

`parse_dt_cfg()` بتلف على كل الـ `dt_params` وبتدور على `"sleep-hardware-state"` في الـ DT node:

```c
/* pinconf-generic.c:243 */
ret = of_property_read_u32(np, par->property, &val);
if (ret == -EINVAL)
    continue;  /* property غير موجودة → skip */
```

مش موجودة → بتاخد الـ default → مش بيتبعت للـ driver → الـ hardware ما بيعملش latch.

**التحقق:**

```bash
# تفعيل dynamic debug
echo "file pinconf-generic.c +p" > /sys/kernel/debug/dynamic_debug/control

# عمل suspend ثم wake
echo mem > /sys/power/state

dmesg | grep "sleep-hardware-state"
# لا يوجد output → property لم تُقرأ
```

**الحل:**

```dts
uart2_sleep_pins: uart2-sleep-pins {
    pinctrl-single,pins = <
        AM62X_IOPAD(0x0054, PIN_INPUT_PULLUP, 7)
        AM62X_IOPAD(0x0058, PIN_OUTPUT_PULLUP, 7)
    >;
    sleep-hardware-state;  /* يأمر الـ hardware يحتفظ بالـ state */
};
```

**الدرس المستفاد:**
`sleep-hardware-state` في `dt_params[]` موجود بـ `default_value=0` — لازم تضيف الـ property صراحةً. غيابها مش error بل silent skip داخل `parse_dt_cfg()`.

---

### السيناريو 5 — Allwinner H616: HDMI Audio مش شغال

**العنوان:** HDMI audio على Orange Pi Zero 2 (H616) مش بيشتغل

**السياق:**
H616 SoC، الـ HDMI audio path يمر عبر I2S3. الـ `aplay` بيشتغل بدون error لكن مافيش صوت.

**المشكلة:**

```dts
i2s3_pins: i2s3-pins {
    pins = "PH4", "PH5", "PH6", "PH7";
    function = "i2s3";
    bias-pull-down;  /* خطأ! I2S محتاج pull-none أو pull-up */
    drive-strength = <40>;  /* 40mA! عالي جداً */
};
```

**التحليل عبر الكود:**

`pinconf_generic_dt_subnode_to_map()` في `pinconf-generic.c:388`:

1. بتلاقي `pins = "PH4"...` → `strings_count = 4`
2. بتلاقي `function = "i2s3"` → `reserve++`
3. `pinconf_generic_parse_dt_config()` بتقرأ:
   - `bias-pull-down` → `PIN_CONFIG_BIAS_PULL_DOWN` بـ default value `1`
   - `drive-strength = <40>` → `PIN_CONFIG_DRIVE_STRENGTH` بـ `val=40`

4. `parse_dt_cfg()` بتعمل:
```c
cfg[*ncfg] = pinconf_to_config_packed(PIN_CONFIG_BIAS_PULL_DOWN, 1);
(*ncfg)++;
cfg[*ncfg] = pinconf_to_config_packed(PIN_CONFIG_DRIVE_STRENGTH, 40);
(*ncfg)++;
```

5. الـ Allwinner driver بيطبق pull-down على I2S data lines → الـ signal ما بتطلعش صح.

**التحقق:**

```bash
cat /sys/kernel/debug/pinctrl/pio/pinconf-groups | grep -A3 "i2s3"
# group: i2s3 (4 pins):
#   input bias pull down (1), output drive strength (40 mA)

# المشكلة: pull-down على I2S lines بتشوّه الـ signal
```

**الحل:**

```dts
i2s3_pins: i2s3-pins {
    pins = "PH4", "PH5", "PH6", "PH7";
    function = "i2s3";
    bias-disable;           /* لا pull resistors */
    drive-strength = <10>;  /* قيمة معقولة */
};
```

**الدرس المستفاد:**
`pinconf_generic_dt_subnode_to_map()` بتقبل أي combination من الـ configs بدون semantic validation — `bias-pull-down` مع I2S signals صحيح syntactically لكن خاطئ electrically. دايماً اقرأ الـ debugfs output وقارن بالـ datasheet.

---

## Phase 7: مصادر ومراجع

### 7.1 مقالات LWN.net

| العنوان | الرابط |
|---------|--------|
| pinctrl: add a generic pin config interface | [lwn.net/Articles/467269](https://lwn.net/Articles/467269/) |
| pinctrl: common handling of generic pinconfig props in dt | [lwn.net/Articles/553754](https://lwn.net/Articles/553754/) |
| pinctrl/pinconfig: add debug interface | [lwn.net/Articles/545790](https://lwn.net/Articles/545790/) |
| pinctrl: add a pin config interface | [lwn.net/Articles/471826](https://lwn.net/Articles/471826/) |
| The pin control subsystem | [lwn.net/Articles/468759](https://lwn.net/Articles/468759/) |
| Documentation/pinctrl.txt (original) | [lwn.net/Articles/465077](https://lwn.net/Articles/465077/) |

### 7.2 Kernel Documentation الرسمية

| الملف | المسار في الـ kernel |
|-------|---------------------|
| Pin control subsystem | `Documentation/driver-api/pin-control.rst` |
| DT bindings | `Documentation/devicetree/bindings/pinctrl/pinctrl-bindings.txt` |
| الـ online version | [docs.kernel.org/driver-api/pin-control.html](https://docs.kernel.org/driver-api/pin-control.html) |
| v4.13 version | [kernel.org/doc/html/v4.13/driver-api/pinctl.html](https://www.kernel.org/doc/html/v4.13/driver-api/pinctl.html) |

### 7.3 Source Files الأساسية في الـ Kernel

```
drivers/pinctrl/pinconf-generic.c   ← الملف ده
drivers/pinctrl/pinconf.c           ← core pinconf
drivers/pinctrl/pinctrl-utils.c     ← helper functions
drivers/pinctrl/core.c              ← pinctrl core
include/linux/pinctrl/pinconf-generic.h  ← API definitions
include/linux/pinctrl/pinconf.h          ← pinconf_ops struct
```

### 7.4 Driver Examples للدراسة

```
drivers/pinctrl/rockchip/pinctrl-rockchip.c   ← RK3562
drivers/pinctrl/stm32/pinctrl-stm32.c         ← STM32MP1
drivers/pinctrl/freescale/pinctrl-imx.c       ← i.MX8
drivers/pinctrl/ti/pinctrl-single.c           ← AM62x
drivers/pinctrl/sunxi/pinctrl-sunxi.c         ← H616
```

### 7.5 مراجع eLinux.org

- Pin Control and GPIO Update: [pdf4pro.com/view/pin-control-and-gpio-update-elinux-org](https://pdf4pro.com/view/pin-control-and-gpio-update-elinux-org-2dea87.pdf)
- Pinctrl overview (STM32): [wiki.st.com/stm32mpu/wiki/Pinctrl_overview](https://wiki.st.com/stm32mpu/wiki/Pinctrl_overview)
- Linux Device Tree Pinctrl Tutorial: [blog.modest-destiny.com/posts/linux-device-tree-pinctrl-tutorial](https://blog.modest-destiny.com/posts/linux-device-tree-pinctrl-tutorial/)

### 7.6 كتب

| الكتاب | الفصل ذو الصلة |
|--------|----------------|
| **Linux Device Drivers (LDD3)** — Corbet, Rubini, Kroah-Hartman | Chapter 14: The Linux Device Model |
| **Linux Kernel Development** — Robert Love | Chapter 17: Devices and Modules |
| **Embedded Linux Primer** — Christopher Hallinan | Chapter 15: Embedded Linux BSP |
| **Linux Device Drivers Development** — John Madieu | Chapter 14: Pin Control and GPIO Subsystem |
| **Mastering Embedded Linux Programming** — Frank Vasquez & Chris Simmonds | Chapter 6: Selecting a Build System |

### 7.7 Search Terms مفيدة

```
pinconf_generic_parse_dt_config site:lkml.org
pinconf-generic.c explanation site:kernelnewbies.org
"PIN_CONFIG_BIAS_PULL_UP" device tree example
pinctrl debugfs pinconf-pins explained
"pinconf_to_config_packed" kernel API
rockchip pinctrl pinconf-generic example
"pinconf_generic_dt_node_to_map" driver implementation
```

---

## Phase 8: Writing simple module

الـ module ده بيستخدم **kprobe** على `pinconf_generic_parse_dt_config` — الدالة اللي بتحول الـ DT properties لـ pin config values. بنطبع معلومات عن كل استدعاء.

```c
// kprobe_pinconf.c
// Intercepts pinconf_generic_parse_dt_config() to log DT config parsing

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/of.h>
#include <linux/pinctrl/pinctrl.h>

/* اسم الدالة اللي هنعمل probe عليها */
#define TARGET_FUNC "pinconf_generic_parse_dt_config"

/*
 * الـ kprobe struct بيحدد الـ function اللي هنراقبها.
 * بنستخدم symbol_name بدل address عشان أسهل.
 */
static struct kprobe kp = {
    .symbol_name = TARGET_FUNC,
};

/*
 * pre_handler: بيتشغل قبل ما الدالة الأصلية تبدأ.
 * الـ regs بتحتوي على الـ arguments: RDI=np, RSI=pctldev, RDX=configs, RCX=nconfigs (x86_64)
 * أو على ARM64: x0=np, x1=pctldev, x2=configs, x3=nconfigs
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* الـ argument الأول هو struct device_node *np */
#ifdef CONFIG_X86_64
    struct device_node *np = (struct device_node *)regs->di;
#elif defined(CONFIG_ARM64)
    struct device_node *np = (struct device_node *)regs->regs[0];
#else
    struct device_node *np = NULL;
#endif

    /* طباعة اسم الـ DT node اللي بيتفارس */
    if (np)
        pr_info("[kprobe_pinconf] parsing DT node: %pOF\n", np);
    else
        pr_info("[kprobe_pinconf] parsing unknown DT node\n");

    return 0; /* صفر = استمر في تنفيذ الدالة الأصلية */
}

/*
 * post_handler: بيتشغل بعد ما الدالة الأصلية تخلص.
 * بنستخدمه للتأكيد إن الـ parse اتم.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    pr_info("[kprobe_pinconf] pinconf_generic_parse_dt_config() returned\n");
}

/*
 * module_init: بيسجل الـ kprobe عند تحميل الـ module.
 * لو فشل التسجيل (الدالة مش موجودة أو mismatch)، بنرجع error.
 */
static int __init kprobe_pinconf_init(void)
{
    int ret;

    kp.pre_handler  = handler_pre;
    kp.post_handler = handler_post;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[kprobe_pinconf] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[kprobe_pinconf] planted kprobe on %s at %p\n",
            TARGET_FUNC, kp.addr);
    return 0;
}

/*
 * module_exit: بيشيل الـ kprobe بأمان عند إزالة الـ module.
 * لازم نعمل unregister قبل ما الـ module يتفرغ من الـ memory.
 */
static void __exit kprobe_pinconf_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[kprobe_pinconf] kprobe removed from %s\n", TARGET_FUNC);
}

module_init(kprobe_pinconf_init);
module_exit(kprobe_pinconf_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc example");
MODULE_DESCRIPTION("kprobe on pinconf_generic_parse_dt_config");
```

**Makefile:**

```makefile
# Makefile لبناء الـ module
obj-m += kprobe_pinconf.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

### شرح كل جزء

| الجزء | الشرح |
|-------|-------|
| `struct kprobe kp` | الـ struct اللي بتعرّف الـ probe — بيحدد الـ function بالاسم عشان الـ kernel يحل الـ address وقت التسجيل |
| `handler_pre` | بيتنفذ قبل الـ target function — بيجيب الـ `device_node *np` من الـ registers ويطبع اسم الـ DT node اللي بيتفارس |
| `handler_post` | بيتنفذ بعد ما الدالة ترجع — مفيد لو عايز تطبع الـ return value أو تقيس الـ latency |
| `register_kprobe()` | بيسجل الـ probe في الـ kernel — لو الـ symbol مش موجود أو الـ kernel compiled without `CONFIG_KPROBES` بيرجع error |
| `unregister_kprobe()` | ضروري في `module_exit` عشان تمنع crash لما الـ module يتشال والـ probe لسه active |
| `MODULE_LICENSE("GPL")` | لازم GPL عشان تقدر تستخدم `register_kprobe()` — الـ symbol ده exported لـ GPL modules فقط |

**تشغيل وعرض النتائج:**

```bash
# بناء وتحميل
make
sudo insmod kprobe_pinconf.ko

# شوف الـ output (بيظهر عند boot أو عند rebind لـ driver)
sudo dmesg | grep kprobe_pinconf
# [   12.345] kprobe_pinconf: planted kprobe on pinconf_generic_parse_dt_config at ffffffffc0a12340
# [   13.001] kprobe_pinconf: parsing DT node: /soc/pinctrl@fdc60000/i2c2-pins
# [   13.002] kprobe_pinconf: pinconf_generic_parse_dt_config() returned
# [   13.005] kprobe_pinconf: parsing DT node: /soc/pinctrl@fdc60000/uart2-pins
# [   13.006] kprobe_pinconf: pinconf_generic_parse_dt_config() returned

# إزالة الـ module
sudo rmmod kprobe_pinconf
```
