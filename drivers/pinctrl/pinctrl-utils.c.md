## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: PIN CONTROL SUBSYSTEM

الـ `pinctrl-utils.c` و `pinctrl-utils.h` جزء من **PIN CONTROL SUBSYSTEM** — المنظومة اللي بتتحكم في pins (أرجل الـ chips) في Linux kernel.

المشرف: **Linus Walleij** — والكود كله تحت `drivers/pinctrl/`.

---

### المشكلة اللي بيحلها الـ Subsystem ده

تخيل إن عندك SoC (System on a Chip) زي Raspberry Pi أو Jetson Nano — ده chip فيه مئات الـ pins الجسدية.

كل pin ممكن يشتغل بأكتر من طريقة:

```
Pin 42:
  ├── UART TX    (بيبعت data تسلسلي)
  ├── GPIO OUT   (إشارة رقمية عادية)
  ├── SPI CLK    (clock لبروتوكول SPI)
  └── PWM        (pulse width modulation)
```

**المشكلة:** إزاي تقول للـ chip "Pin 42 هيشتغل كـ UART TX" وكمان تضبط خصائصه زي pull-up/pull-down، drive strength، speed؟

ده اللي بيعمله الـ **pinctrl subsystem** — هو الوسيط المركزي اللي بيجمع كل إعدادات الـ pins في مكان واحد.

---

### القصة الكاملة: من Device Tree لـ Hardware

```
Device Tree (dts file)
    │
    │  &uart0 { pinctrl-0 = <&uart_pins>; }
    │  uart_pins: uart-pins { pins = "PA0","PA1"; function = "uart"; }
    │
    ▼
pinctrl core (core.c / devicetree.c)
    │  يقرأ الـ DT node ويحوله لـ pinctrl_map entries
    │
    ▼
pinctrl_map[] — جدول الإعدادات
    │  { device="uart0", group="uart-pins", function="uart" }
    │  { device="uart0", group="uart-pins", config=pull-up }
    │
    ▼
hardware driver (مثلاً pinctrl-tegra.c)
    │  يكتب الـ registers الفعلية في الـ SoC
    ▼
Hardware pins
```

---

### دور `pinctrl-utils.c` تحديداً

كل driver بيحتاج يبني الـ `pinctrl_map[]` ده أثناء الـ probe. المشكلة إن بناء الجدول ده كود متكرر في كل driver — تحجز ذاكرة، تحط entries، تدير الـ counters.

**`pinctrl-utils.c` هو مكتبة utilities مشتركة** بتوفر helper functions جاهزة يستخدمها أي driver بدل ما يعيد نفس الكود من الصفر.

فكره زي مكتبة أدوات مشتركة تستخدمها كل المشاريع بدل ما كل مشروع يعيد اختراع العجلة.

---

### الـ 5 Functions اللي فيها

#### 1. `pinctrl_utils_reserve_map` — احجز مكان في الجدول

```c
int pinctrl_utils_reserve_map(struct pinctrl_dev *pctldev,
        struct pinctrl_map **map,
        unsigned int *reserved_maps,
        unsigned int *num_maps,
        unsigned int reserve)
```

زي إنك بتحجز كراسي في قاعة قبل ما الناس تيجي. لو المكان المحجوز مش كفاية، بيعمل `krealloc_array` يوسّع الجدول.

#### 2. `pinctrl_utils_add_map_mux` — أضف إعداد mux

```c
int pinctrl_utils_add_map_mux(struct pinctrl_dev *pctldev,
        struct pinctrl_map **map, ...
        const char *group,
        const char *function)
```

بيضيف entry من نوع `PIN_MAP_TYPE_MUX_GROUP` — يعني "مجموعة الـ pins دي هتشتغل كـ function معينة (UART, SPI, إلخ)".

#### 3. `pinctrl_utils_add_map_configs` — أضف إعداد configuration

```c
int pinctrl_utils_add_map_configs(struct pinctrl_dev *pctldev,
        struct pinctrl_map **map, ...
        unsigned long *configs, unsigned int num_configs,
        enum pinctrl_map_type type)
```

بيضيف entry من نوع configs زي pull-up، drive strength، إلخ. لاحظ إنه بيعمل `kmemdup_array` للـ configs — يعني بيعمل نسخة خاصة منها في الذاكرة عشان ما تتعملش مشكلة لو الـ original اتحرر.

#### 4. `pinctrl_utils_add_config` — أضف config واحدة

```c
int pinctrl_utils_add_config(struct pinctrl_dev *pctldev,
        unsigned long **configs,
        unsigned int *num_configs,
        unsigned long config)
```

بيضيف قيمة config واحدة لمصفوفة الـ configs بـ `krealloc`. بتستخدمه لما بتبني الـ configs array قبل ما تمررها لـ `add_map_configs`.

#### 5. `pinctrl_utils_free_map` — حرر الذاكرة

```c
void pinctrl_utils_free_map(struct pinctrl_dev *pctldev,
        struct pinctrl_map *map,
        unsigned int num_maps)
```

بيحرر الذاكرة اللي اتحجزت — بيمشي على كل entry، ولو نوعها configs بيحرر الـ configs array الداخلية، وفي الآخر بيحرر الجدول كله.

---

### مثال حقيقي: driver بيستخدم الـ utils دي

في `drivers/pinctrl/pinctrl-tegra.c` (NVIDIA Tegra) الـ `.dt_node_to_map` callback بيعمل كده:

```c
// 1. احجز مكان للـ entries الجديدة
ret = pinctrl_utils_reserve_map(pctldev, &map,
                                &reserved_maps, num_maps, 2);

// 2. لو في function، ضيف mux entry
ret = pinctrl_utils_add_map_mux(pctldev, &map,
                                &reserved_maps, num_maps,
                                group, function);

// 3. لو في configs (pull, drive, ...)، ضيفها
ret = pinctrl_utils_add_map_configs(pctldev, &map,
                                    &reserved_maps, num_maps,
                                    group, configs, num_configs,
                                    PIN_MAP_TYPE_CONFIGS_GROUP);
```

---

### الـ Data Structures الأساسية

| Struct | ملف التعريف | الوظيفة |
|--------|-------------|---------|
| `struct pinctrl_map` | `include/linux/pinctrl/machine.h` | entry واحدة في جدول الإعدادات |
| `struct pinctrl_map_mux` | `include/linux/pinctrl/machine.h` | بيانات الـ mux (group + function) |
| `struct pinctrl_map_configs` | `include/linux/pinctrl/machine.h` | بيانات الـ configs (pin/group + array) |
| `struct pinctrl_dev` | `drivers/pinctrl/core.h` | الـ controller نفسه |
| `enum pinctrl_map_type` | `include/linux/pinctrl/machine.h` | نوع الـ entry (MUX، CONFIGS_PIN، CONFIGS_GROUP) |

---

### الملفات المكوّنة للـ Subsystem

#### الـ Core (القلب)

| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/core.c` | قلب الـ subsystem — registration، lookup، الـ state machine |
| `drivers/pinctrl/core.h` | تعريفات داخلية للـ core |
| `drivers/pinctrl/pinmux.c` | منطق الـ muxing (تحديد وظيفة الـ pin) |
| `drivers/pinctrl/pinconf.c` | منطق الـ configuration (pull-up, drive, ...) |
| `drivers/pinctrl/pinconf-generic.c` | implementation generic للـ configs |
| `drivers/pinctrl/devicetree.c` | تحويل الـ Device Tree nodes لـ pinctrl_map entries |
| `drivers/pinctrl/pinctrl-utils.c` | **helper functions مشتركة للـ drivers** |

#### الـ Headers العامة

| الملف | الدور |
|-------|-------|
| `include/linux/pinctrl/pinctrl.h` | الـ API الرئيسية + `pinctrl_ops` |
| `include/linux/pinctrl/machine.h` | `pinctrl_map` + أنواعها + macros |
| `include/linux/pinctrl/pinconf.h` | الـ API للـ pin configuration |
| `include/linux/pinctrl/pinmux.h` | الـ API للـ pin muxing |
| `include/linux/pinctrl/consumer.h` | الـ API للـ devices اللي بتستخدم الـ pinctrl |

#### Hardware Drivers (أمثلة)

| الملف | الـ Hardware |
|-------|-------------|
| `drivers/pinctrl/pinctrl-tegra.c` | NVIDIA Tegra |
| `drivers/pinctrl/intel/pinctrl-intel.c` | Intel SoCs |
| `drivers/pinctrl/freescale/pinctrl-imx.c` | NXP i.MX |
| `drivers/pinctrl/qcom/pinctrl-msm.c` | Qualcomm Snapdragon |
| `drivers/pinctrl/bcm/pinctrl-bcm2835.c` | Raspberry Pi |

---

### ملفات يهمك تعرفها كقارئ

- **`drivers/pinctrl/core.c`** — ابدأ منه لفهم الـ subsystem كله
- **`include/linux/pinctrl/machine.h`** — لفهم الـ `pinctrl_map` بالتفصيل
- **`drivers/pinctrl/devicetree.c`** — لفهم كيف الـ DT يتحول لـ map entries
- **`Documentation/driver-api/pin-control.rst`** — الـ documentation الرسمية
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة — ليه الـ Pinctrl موجود أصلاً؟

في أي SoC حديث (زي Tegra أو i.MX أو Rockchip)، الـ physical pins على الـ chip مش dedicated لـ function واحدة. كل pin ممكن يشتغل كـ UART TX، أو SPI CLK، أو GPIO، أو I2C SDA — حسب إيه اللي بتكتبه في registers الـ pin controller.

المشكلة اللي كانت موجودة قبل الـ pinctrl subsystem:
- كل driver كان بيلخبط في registers الـ pin mux بنفسه.
- مفيش coordination — driver A ممكن يعيد configure pin بيستخدمه driver B.
- الـ board-specific configuration كانت متفرقة في كل مكان.
- الـ Device Tree مكانش ليه طريقة موحدة يعبّر عن pin configurations.

**الـ pinctrl subsystem** جاء عشان يكون الـ single authority اللي بيتحكم في:
1. مين يملك الـ pin (ownership).
2. الـ pin مشغول بأي function (muxing).
3. الـ electrical properties للـ pin (configuration: pull-up, drive strength, etc.).

---

### الحل — المنهج اللي بياخده الـ kernel

الـ kernel قسّم الموضوع لثلاث طبقات:

| الطبقة | المسؤولية |
|--------|-----------|
| **pinctrl core** | Ownership، state machine، DT parsing coordination |
| **pinmux** | اختيار الـ function لكل pin/group |
| **pinconf** | ضبط الـ electrical parameters |

وكل SoC بيسجّل نفسه كـ **pin controller** يوفّر implementations لهذه الطبقات عبر vtable operations.

الـ **pinctrl-utils** تحديداً هو helper layer بيوفر utility functions للـ pin controller drivers أثناء بناء الـ `pinctrl_map` table من الـ Device Tree.

---

### الـ Real-World Analogy — لوحة توزيع الكهرباء

تخيل **لوحة توزيع كهرباء** (electrical panel) في مصنع:

| عنصر في المصنع | المقابل في الـ kernel |
|----------------|----------------------|
| **الأسلاك الفيزيائية** | الـ physical pins على الـ SoC |
| **القواطع (circuit breakers)** | الـ pin controller hardware |
| **مهندس الكهرباء** (اللي بيوصل السلك لأي مكان) | الـ pinctrl core |
| **خريطة توصيلات المصنع** (أي آلة متوصلة فين) | الـ `pinctrl_map` table |
| **نوع التوصيل** (380V/220V/control signal) | الـ `pinctrl_map_type` (MUX أو CONFIG) |
| **تعليمات التوصيل من المخطط** | الـ Device Tree nodes |
| **شركة المقاولات اللي بتنفذ التوصيلات** | الـ pin controller driver + pinctrl-utils |
| **وثيقة المواصفات** (مقاومة السلك، سُمكه) | الـ configs array (`unsigned long *configs`) |

الـ `pinctrl_utils_reserve_map()` = المهندس بيحجز مساحة في الخريطة قبل ما يبدأ يرسم.
الـ `pinctrl_utils_add_map_mux()` = بيكتب "السلك ده يروح للآلة دي" في الخريطة.
الـ `pinctrl_utils_add_map_configs()` = بيكتب "السلك ده بمقاومة كذا وسُمك كذا".
الـ `pinctrl_utils_free_map()` = بيحذف الخريطة المؤقتة بعد تسليمها للـ core.

---

### الـ Big Picture Architecture

```
                    ┌─────────────────────────────────────────────┐
                    │              User/Board Layer                │
                    │   Device Tree  /  Static pinctrl_map tables │
                    └────────────────────┬────────────────────────┘
                                         │
                    ┌────────────────────▼────────────────────────┐
                    │           pinctrl Core (core.c)             │
                    │  - pin ownership tracking                   │
                    │  - state machine (default/sleep/idle)       │
                    │  - map table registry & lookup              │
                    │  - DT binding glue                          │
                    └──────┬──────────────────┬───────────────────┘
                           │                  │
          ┌────────────────▼───┐    ┌─────────▼──────────────────┐
          │   pinmux layer     │    │     pinconf layer          │
          │  (pinmux.c)        │    │   (pinconf.c / generic)    │
          │  - function select │    │   - drive strength         │
          │  - group muxing    │    │   - pull up/down           │
          └────────┬───────────┘    └────────────┬───────────────┘
                   │                             │
          ┌────────▼─────────────────────────────▼───────────────┐
          │          Pin Controller Driver  (e.g., tegra-pinctrl) │
          │                                                        │
          │  ┌─────────────────────────────────────────────────┐  │
          │  │              pinctrl-utils helpers              │  │
          │  │  reserve_map / add_map_mux / add_map_configs    │  │
          │  │  add_config / free_map                          │  │
          │  └─────────────────────────────────────────────────┘  │
          │                                                        │
          │  pinctrl_ops.dt_node_to_map()  ──► builds pinctrl_map │
          │  pinmux_ops.set_mux()          ──► writes HW registers │
          │  pinconf_ops.pin_config_set()  ──► writes HW registers │
          └────────────────────────────────────────────────────────┘
                           │
          ┌────────────────▼───────────────────────────────────────┐
          │                SoC Hardware                            │
          │   Pin Mux Registers  │  Pin Config Registers           │
          └────────────────────────────────────────────────────────┘
```

**المستهلكون (Consumers):**
- أي driver بيطلب `pinctrl_get()` و `pinctrl_lookup_state()` و `pinctrl_select_state()` — يعني UART driver، SPI driver، إلخ.

**المزودون (Providers):**
- الـ SoC-specific pin controller drivers اللي بتسجّل نفسها بـ `pinctrl_register_and_init()`.

---

### الـ Core Abstraction — الـ `pinctrl_map`

الفكرة المركزية في الـ subsystem كلها هي الـ **mapping table** — جدول بيقول:

> "الـ device X، لما يكون في state Y، يحتاج الـ pin group Z يكون على الـ function F بـ configuration C."

```c
struct pinctrl_map {
    const char *dev_name;        /* مين الـ consumer؟ (e.g., "serial8250.0") */
    const char *name;            /* اسم الـ state (e.g., "default", "sleep") */
    enum pinctrl_map_type type;  /* MUX_GROUP أو CONFIGS_PIN أو CONFIGS_GROUP */
    const char *ctrl_dev_name;   /* اسم الـ pin controller */
    union {
        struct pinctrl_map_mux mux;        /* للـ mux entries */
        struct pinctrl_map_configs configs; /* للـ config entries */
    } data;
};
```

الـ **`pinctrl_map_type`** enum بيحدد نوع الـ entry:

| النوع | المعنى |
|-------|--------|
| `PIN_MAP_TYPE_MUX_GROUP` | وصّل الـ group لـ function معينة |
| `PIN_MAP_TYPE_CONFIGS_PIN` | اضبط electrical settings لـ pin واحد |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | اضبط electrical settings لـ group كامل |
| `PIN_MAP_TYPE_DUMMY_STATE` | state موجود بس مفيش operations |

---

### العلاقة بين الـ structs

```
pinctrl_map
├── dev_name ──────────────────► "spi@7000d400"  (consumer device)
├── name ──────────────────────► "default"        (state name)
├── type ──────────────────────► PIN_MAP_TYPE_MUX_GROUP
├── ctrl_dev_name ─────────────► "tegra-pinctrl"  (provider)
└── data.mux
    ├── group ─────────────────► "spi1_grp"       (pin group name)
    └── function ──────────────► "spi1"           (mux function)

pinctrl_map  (entry تاني في نفس الـ table)
├── dev_name ──────────────────► "spi@7000d400"
├── name ──────────────────────► "default"
├── type ──────────────────────► PIN_MAP_TYPE_CONFIGS_GROUP
├── ctrl_dev_name ─────────────► "tegra-pinctrl"
└── data.configs
    ├── group_or_pin ──────────► "spi1_grp"
    ├── configs ───────────────► [TEGRA_PIN_PULL_UP | ..., TEGRA_PIN_DRIVE_8MA]
    └── num_configs ───────────► 2
```

---

### الـ pinctrl-utils — وظيفتها بالضبط

الـ `pinctrl-utils.c` بيوفر **4 helper functions + 1 free function** مخصصين لاستخدامهم جوه الـ `dt_node_to_map()` callback بتاع الـ pin controller driver.

#### 1. `pinctrl_utils_reserve_map()`

```c
int pinctrl_utils_reserve_map(struct pinctrl_dev *pctldev,
        struct pinctrl_map **map, unsigned int *reserved_maps,
        unsigned int *num_maps, unsigned int reserve)
```

**المشكلة اللي بيحلها:** الـ driver بيمشي على الـ DT nodes واحدة واحدة، ومش عارف مسبقاً هيحتاج كام entry في الـ map. بدل ما يـ allocate كل مرة من الأول، بيستخدم **dynamic growing array** — يحجز مسبقاً `reserve` slots إضافية.

```
قبل الاستدعاء:
  map:           [entry0][entry1][------][------]
  reserved_maps: 4
  num_maps:      2

بعد reserve_map(... reserve=3):
  new_num = num_maps + reserve = 2 + 3 = 5
  old_num (4) < new_num (5) → krealloc

  map:           [entry0][entry1][------][------][------]
  reserved_maps: 5
  num_maps:      2  (لسه مش اتغير)
```

الـ `krealloc_array()` بيضمن إن الـ existing entries محافظة على بياناتها، والـ `memset()` بيصفّر الـ slots الجديدة.

#### 2. `pinctrl_utils_add_map_mux()`

```c
int pinctrl_utils_add_map_mux(struct pinctrl_dev *pctldev,
        struct pinctrl_map **map, unsigned int *reserved_maps,
        unsigned int *num_maps, const char *group,
        const char *function)
```

بيضيف entry من نوع `PIN_MAP_TYPE_MUX_GROUP` في الـ slot الجاي المتاح (`*num_maps`)، وبعدين يزوّد الـ counter.

```c
(*map)[*num_maps].type = PIN_MAP_TYPE_MUX_GROUP;
(*map)[*num_maps].data.mux.group = group;     /* string من الـ DT، مش copied */
(*map)[*num_maps].data.mux.function = function;
(*num_maps)++;
```

لاحظ: الـ strings مش بتتـ `kmemdup`، الـ driver بيمرر pointers لـ strings موجودة (عادةً من الـ DT node أو static arrays).

#### 3. `pinctrl_utils_add_map_configs()`

```c
int pinctrl_utils_add_map_configs(struct pinctrl_dev *pctldev,
        struct pinctrl_map **map, unsigned int *reserved_maps,
        unsigned int *num_maps, const char *group,
        unsigned long *configs, unsigned int num_configs,
        enum pinctrl_map_type type)
```

الفرق الجوهري هنا: الـ `configs` array **بتتـ duplicate** بـ `kmemdup_array()`. ليه؟ لأن الـ configs غالبًا بتتبنى على الـ stack أو في buffer مؤقت أثناء الـ DT parsing، فلازم نعمل نسخة منفصلة مملوكة للـ map.

```
configs (على الـ stack في الـ driver):
  [ TEGRA_PULL_UP_VALUE , TEGRA_DRIVE_8MA ]
        │
        └── kmemdup_array() ──►  heap allocation
                                 [ TEGRA_PULL_UP_VALUE , TEGRA_DRIVE_8MA ]
                                        ▲
                              map[n].data.configs.configs ─────────────┘
```

الـ `type` parameter بيكون إما `PIN_MAP_TYPE_CONFIGS_PIN` أو `PIN_MAP_TYPE_CONFIGS_GROUP`.

#### 4. `pinctrl_utils_add_config()`

```c
int pinctrl_utils_add_config(struct pinctrl_dev *pctldev,
        unsigned long **configs, unsigned int *num_configs,
        unsigned long config)
```

هذه الـ function مش بتشتغل على الـ map مباشرةً — هي بتبني الـ `configs` array نفسها value by value قبل ما تتمرر لـ `add_map_configs()`. بتستخدم `krealloc()` بعد كل إضافة — pattern شبه الـ `reserve_map`.

```
مثال على الاستخدام في dt_node_to_map:

unsigned long *configs = NULL;
unsigned int num_configs = 0;

// لكل property في الـ DT node:
pinctrl_utils_add_config(pctldev, &configs, &num_configs, TEGRA_PIN_PULL_UP);
pinctrl_utils_add_config(pctldev, pctldev, &configs, &num_configs, TEGRA_PIN_DRIVE_8MA);

// بعد الانتهاء:
pinctrl_utils_add_map_configs(..., configs, num_configs, PIN_MAP_TYPE_CONFIGS_GROUP);
// add_map_configs هتعمل kmemdup للـ configs → آمن تحرير configs الأصلي بعدين
```

#### 5. `pinctrl_utils_free_map()`

```c
void pinctrl_utils_free_map(struct pinctrl_dev *pctldev,
        struct pinctrl_map *map, unsigned int num_maps)
```

بتتحرك على الـ map وتحرر الـ `configs` arrays اللي اتعملت بـ `kmemdup_array()` في `add_map_configs()`، وبعدين تحرر الـ map نفسها. الـ mux entries مش محتاجة تحرير داخلي لأنها بس pointers لـ existing strings.

```c
for (i = 0; i < num_maps; i++) {
    switch (map[i].type) {
    case PIN_MAP_TYPE_CONFIGS_GROUP:
    case PIN_MAP_TYPE_CONFIGS_PIN:
        kfree(map[i].data.configs.configs); /* حرر الـ kmemdup'd array */
        break;
    default:
        break; /* PIN_MAP_TYPE_MUX_GROUP: مفيش heap data داخلي */
    }
}
kfree(map); /* حرر الـ map array نفسها */
```

---

### الـ Flow الكامل — من DT إلى Hardware

```
Device Tree
──────────
pinctrl@70000868 {
    spi1_default: spi1-default {
        pins = "spi1_mosi_pc0", "spi1_miso_pc1";
        function = "spi1";
        pull-up;
        drive-strength = <8>;
    };
};

        │
        ▼
dt_node_to_map()  (في الـ SoC driver)
        │
        ├── pinctrl_utils_reserve_map()    ← حجز slots
        │
        ├── pinctrl_utils_add_map_mux()    ← entry: group="spi1_grp" fn="spi1"
        │
        ├── pinctrl_utils_add_config() x2  ← بناء configs array
        │
        └── pinctrl_utils_add_map_configs() ← entry: configs=[PULL_UP, 8MA]
                │
                ▼
        pinctrl_map[]  (جاهزة)
                │
                ▼
        pinctrl core  ← بتسجّل الـ map في global list
                │
                ▼
        pinctrl_select_state("default")  ← لما الـ SPI driver يطلبها
                │
                ├── pinmux_ops.set_mux()     ← يكتب في registers
                └── pinconf_ops.pin_config_set() ← يكتب في registers
```

---

### الـ Subsystem يملك إيه vs بيفوّض إيه للـ Drivers

| المسؤولية | الـ pinctrl core + utils يتحكم فيها | الـ Driver بيـ implement |
|-----------|-------------------------------------|--------------------------|
| Memory management للـ map table | `reserve_map`, `free_map` | — |
| بناء الـ map entries | `add_map_mux`, `add_map_configs` | — |
| تسجيل الـ map في global registry | `pinctrl_register_mappings()` | — |
| Consumer state lookup وتطبيقه | `pinctrl_select_state()` | — |
| تحديد الـ DT properties اللي تعني إيه | — | `dt_node_to_map()` |
| الكتابة الفعلية في registers | — | `pinmux_ops.set_mux()` |
| ضبط الـ electrical params في HW | — | `pinconf_ops.pin_config_set()` |
| تعريف الـ pin groups والـ functions | — | `pinctrl_ops.get_group_*()` |

---

### ملاحظة على الـ EXPORT_SYMBOL_GPL

كل الـ functions في `pinctrl-utils.c` معملّها `EXPORT_SYMBOL_GPL` — يعني متاحة بس للـ GPL modules. ده منطقي لأنها جزء من الـ kernel internal framework API، مش public userspace ABI.

---

### Subsystems تانية محتاج تعرفها للفهم الكامل

- **الـ Device Tree (OF subsystem)**: الـ `dt_node_to_map()` بتتعامل مع `struct device_node`، وبتستخدم `of_property_read_*()` functions لقراءة الـ DT properties. لازم تفهم الـ DT binding model عشان تفهم إزاي الـ driver بيترجم DT لـ map entries.
- **الـ GPIO subsystem**: الـ pinctrl بيتشابك معاه عبر `pinctrl_gpio_range` — بعض الـ pins بتشتغل كـ GPIO، وده بيتطلب coordination بين الـ subsystemين.
- **الـ regmap/platform bus**: الـ pin controller نفسه بيتسجّل كـ platform device وبيستخدم regmap أو direct MMIO للوصول للـ registers.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Flags, Enums, Config Options — Cheatsheet

#### `enum pinctrl_map_type` (من `include/linux/pinctrl/machine.h`)

| القيمة | المعنى |
|---|---|
| `PIN_MAP_TYPE_INVALID` | entry فاسد أو غير مهيأ |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمي لا يعمل على hardware حقيقي |
| `PIN_MAP_TYPE_MUX_GROUP` | entry لتحديد الـ mux function لمجموعة pins |
| `PIN_MAP_TYPE_CONFIGS_PIN` | entry لضبط config لـ pin واحد بالاسم |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | entry لضبط config لمجموعة pins بالاسم |

#### `PINFUNCTION_FLAG_GPIO` (من `include/linux/pinctrl/pinctrl.h`)

| الـ Flag | القيمة | المعنى |
|---|---|---|
| `PINFUNCTION_FLAG_GPIO` | `BIT(0)` | الـ function دي هي GPIO — مش function خاصة بـ peripheral |

#### Convenience Macros للـ Map Table

| الـ Macro | الاستخدام |
|---|---|
| `PIN_MAP_DUMMY_STATE(dev, state)` | يعمل entry فاضي لـ state |
| `PIN_MAP_MUX_GROUP(dev, state, pinctrl, grp, func)` | يربط group بـ mux function |
| `PIN_MAP_MUX_GROUP_DEFAULT(...)` | نفسه لكن الـ state هو `PINCTRL_STATE_DEFAULT` |
| `PIN_MAP_MUX_GROUP_HOG(...)` | الـ driver نفسه هو الـ consumer |
| `PIN_MAP_CONFIGS_PIN(dev, state, pinctrl, pin, cfgs)` | يضبط configs لـ pin واحد |
| `PIN_MAP_CONFIGS_GROUP(dev, state, pinctrl, grp, cfgs)` | يضبط configs لـ group |

---

### 1. الـ Structs المهمة

#### `struct pinctrl_map`

**الغرض:** الـ entry الأساسي في جدول الـ mapping — كل entry بيقول "الـ device ده في الـ state ده يستخدم الـ pin configuration دي".

| الـ Field | النوع | الشرح |
|---|---|---|
| `dev_name` | `const char *` | اسم الـ device اللي بيستخدم الـ mapping ده |
| `name` | `const char *` | اسم الـ state (مثلاً `"default"` أو `"sleep"`) |
| `type` | `enum pinctrl_map_type` | نوع الـ entry: mux ولا config |
| `ctrl_dev_name` | `const char *` | اسم الـ pin controller اللي بيتحكم في الـ pins دي |
| `data.mux` | `struct pinctrl_map_mux` | لو النوع `MUX_GROUP`: اسم الـ group والـ function |
| `data.configs` | `struct pinctrl_map_configs` | لو النوع `CONFIGS_*`: الـ configs array |

**كيف بيتربط:** المصفوفة بتاعت `pinctrl_map` بيتم تسجيلها في الـ core عن طريق `pinctrl_register_mappings()`. الـ `pinctrl-utils` بيساعد الـ driver في بناء الـ array دي ديناميكياً من DT.

---

#### `struct pinctrl_map_mux`

**الغرض:** بيحدد ربط الـ group بالـ mux function.

| الـ Field | النوع | الشرح |
|---|---|---|
| `group` | `const char *` | اسم الـ pin group — ممكن يبقى NULL ويستخدم أول group |
| `function` | `const char *` | اسم الـ mux function اللي المفروض يتفعّل |

**مثال:** لو عندك UART2 محتاج pins `{PA4, PA5}` — الـ group اسمه `"uart2_pins"` والـ function اسمه `"uart2"`.

---

#### `struct pinctrl_map_configs`

**الغرض:** بيحمل مصفوفة configs لـ pin أو group معين.

| الـ Field | النوع | الشرح |
|---|---|---|
| `group_or_pin` | `const char *` | اسم الـ pin أو الـ group اللي الـ configs دي بتنطبق عليه |
| `configs` | `unsigned long *` | pointer لمصفوفة الـ configs (كل قيمة بتجمع param + value) |
| `num_configs` | `unsigned int` | عدد العناصر في مصفوفة الـ configs |

**ملاحظة مهمة:** الـ `pinctrl_utils_add_map_configs()` بيعمل `kmemdup_array()` للـ configs — يعني الـ ownership بتاع المصفوفة الجديدة بيبقى عند الـ map entry، ولازم يتحرر بـ `pinctrl_utils_free_map()`.

---

#### `struct pinctrl_dev`

**الغرض:** الـ handle الأساسي للـ pin controller المسجّل — بيمثل instance واحد من الـ controller في الـ system.

الـ struct ده معرّف في `core.h` (internal) مش exposed للـ drivers مباشرةً. الـ `pinctrl-utils` بيستقبله كـ opaque pointer لعرض رسائل error بس.

---

#### `struct pinctrl_desc`

**الغرض:** الـ descriptor اللي الـ driver بيسجّله في الـ pinctrl core — بيوصف كل إمكانيات الـ controller.

| الـ Field | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | اسم الـ controller |
| `pins` | `const struct pinctrl_pin_desc *` | مصفوفة وصف كل الـ pins |
| `npins` | `unsigned int` | عدد الـ pins |
| `pctlops` | `const struct pinctrl_ops *` | عمليات الـ grouping وبناء الـ map من DT |
| `pmxops` | `const struct pinmux_ops *` | عمليات الـ mux |
| `confops` | `const struct pinconf_ops *` | عمليات الـ config |
| `owner` | `struct module *` | الـ module اللي بيملك الـ driver |
| `link_consumers` | `bool` | لو `true` يعمل device link مع الـ consumers |

---

#### `struct pinctrl_ops`

**الغرض:** الـ vtable اللي الـ driver بيوفر من خلاله عمليات الـ pin grouping وتحويل DT لـ map.

| الـ Function | الشرح |
|---|---|
| `get_groups_count()` | إجمالي عدد الـ groups |
| `get_group_name()` | اسم الـ group بالـ index |
| `get_group_pins()` | الـ pins اللي في group معين |
| `pin_dbg_show()` | debugfs hook اختياري |
| `dt_node_to_map()` | يحوّل DT node لـ `pinctrl_map[]` — هنا بيُستخدم `pinctrl-utils` |
| `dt_free_map()` | يحرر الـ map اللي عمله `dt_node_to_map` — هنا بيُستخدم `pinctrl_utils_free_map()` |

---

#### `struct pinctrl_pin_desc`

**الغرض:** وصف pin واحد.

| الـ Field | النوع | الشرح |
|---|---|---|
| `number` | `unsigned int` | رقم الـ pin الـ unique في الـ global space |
| `name` | `const char *` | اسم الـ pin (مثلاً `"PA4"`) |
| `drv_data` | `void *` | بيانات خاصة بالـ driver، الـ core ما بيلمسهاش |

---

#### `struct pingroup`

**الغرض:** بيجمع عدد من الـ pins تحت اسم واحد.

| الـ Field | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | اسم الـ group (مثلاً `"spi1_pins"`) |
| `pins` | `const unsigned int *` | مصفوفة أرقام الـ pins |
| `npins` | `size_t` | عدد الـ pins في المجموعة |

---

#### `struct pinfunction`

**الغرض:** بيصف function متاحة للـ mux (مثلاً SPI، I2C، UART).

| الـ Field | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | اسم الـ function |
| `groups` | `const char * const *` | أسماء الـ groups اللي تقدر تستخدم الـ function دي |
| `ngroups` | `size_t` | عدد الـ groups |
| `flags` | `unsigned long` | مثلاً `PINFUNCTION_FLAG_GPIO` |

---

### 2. مخطط علاقات الـ Structs

```
┌──────────────────────────────────────────────────────────┐
│                    struct pinctrl_desc                   │
│  name, pins[], npins                                     │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ pinctrl_ops │  │ pinmux_ops   │  │ pinconf_ops  │   │
│  │ (pctlops)   │  │ (pmxops)     │  │ (confops)    │   │
│  └──────┬──────┘  └──────────────┘  └──────────────┘   │
└─────────┼────────────────────────────────────────────────┘
          │ dt_node_to_map()
          │ dt_free_map()
          ▼
┌──────────────────────────────┐
│  struct pinctrl_map[]        │  ← مصفوفة ديناميكية
│  (مبنية بـ pinctrl-utils)    │
│                              │
│  ┌──────────────────────┐    │
│  │  entry[0]            │    │
│  │  dev_name, name      │    │
│  │  type = MUX_GROUP    │    │
│  │  data.mux ──────────►│──► struct pinctrl_map_mux
│  └──────────────────────┘    │    { group, function }
│  ┌──────────────────────┐    │
│  │  entry[1]            │    │
│  │  type = CONFIGS_PIN  │    │
│  │  data.configs ───────│──► struct pinctrl_map_configs
│  └──────────────────────┘    │    { group_or_pin,
└──────────────────────────────┘      configs[],
                                       num_configs }

struct pinctrl_desc
      │
      │ pins[]
      ▼
struct pinctrl_pin_desc[]    ← كل pin بـ number + name + drv_data

struct pingroup[]            ← groups من أرقام pins
      │
      └──► pins[]  (subset من pinctrl_pin_desc numbers)

struct pinfunction[]
      │
      └──► groups[]  (أسماء groups بتستخدم الـ function دي)
```

---

### 3. مخطط دورة الحياة (Lifecycle)

```
[Driver Init]
      │
      ▼
pinctrl_register_and_init(desc, dev, drv_data, &pctldev)
      │
      ▼
pinctrl_enable(pctldev)
      │
      ▼
[DT Parsing — per consumer device]
      │
      ▼
pctlops->dt_node_to_map(pctldev, np, &map, &num_maps)
      │
      ├─► pinctrl_utils_reserve_map()   ← يحجز/يوسّع المصفوفة
      │
      ├─► pinctrl_utils_add_map_mux()   ← يضيف MUX entry
      │   أو
      ├─► pinctrl_utils_add_map_configs() ← يضيف CONFIGS entry
      │       └─► kmemdup_array(configs)  ← copy الـ configs
      │
      ▼
[Map مكتملة — بتتسجل في الـ core]
      │
      ▼
[Runtime: consumer يطلب state]
      │
      ▼
pinctrl_select_state()
      │
      ├─► pmxops->set_mux()     ← لو MUX entry
      └─► confops->pin_config_set() ← لو CONFIGS entry

[Driver Remove / DT Free]
      │
      ▼
pctlops->dt_free_map(pctldev, map, num_maps)
      │
      └─► pinctrl_utils_free_map()
              │
              ├─► kfree(entry.data.configs.configs)  ← لكل CONFIGS entry
              └─► kfree(map)

[Driver Remove]
      │
      ▼
pinctrl_unregister(pctldev)
```

---

### 4. مخطط تدفق الاستدعاءات (Call Flow)

#### بناء الـ Map من DT

```
[DT binding: dt_node_to_map() في driver مثل Tegra/Rockchip]
  │
  ├─► pinctrl_utils_reserve_map(pctldev, &map, &reserved, &num, N)
  │       │
  │       └─► krealloc_array(*map, new_num, sizeof(pinctrl_map))
  │               └─► memset(new entries, 0)
  │               └─► *reserved_maps = new_num
  │
  ├─► [loop على كل DT property]
  │       │
  │       ├─► pinctrl_utils_add_config(pctldev, &configs, &num_cfg, val)
  │       │       └─► krealloc(*configs, new_size)
  │       │               └─► configs[old_num] = val
  │       │
  │       └─► pinctrl_utils_add_map_configs(pctldev, &map, &reserved,
  │               &num, group, configs, num_configs, type)
  │                   │
  │                   ├─► WARN_ON(*num_maps == *reserved_maps)
  │                   ├─► kmemdup_array(configs, num_configs)
  │                   └─► map[num_maps].data.configs = {copy}
  │                       map[num_maps].type = type
  │                       (*num_maps)++
  │
  └─► [أو لـ mux فقط]
          └─► pinctrl_utils_add_map_mux(pctldev, &map, &reserved,
                  &num, group, function)
                      ├─► map[num_maps].type = PIN_MAP_TYPE_MUX_GROUP
                      ├─► map[num_maps].data.mux.group = group
                      ├─► map[num_maps].data.mux.function = function
                      └─► (*num_maps)++
```

#### تحرير الـ Map

```
dt_free_map(pctldev, map, num_maps)
  │
  └─► pinctrl_utils_free_map(pctldev, map, num_maps)
          │
          └─► for i in 0..num_maps:
                  if map[i].type == CONFIGS_GROUP || CONFIGS_PIN:
                      kfree(map[i].data.configs.configs)
          └─► kfree(map)
```

---

### 5. استراتيجية الـ Locking

ملف `pinctrl-utils.c` نفسه **ما فيهوش أي lock** — وده مقصود لأن الـ functions دي utilities بيتم استدعاؤها من سياق واحد محدد.

#### مين اللي بيعمل الـ locking؟

| السياق | الـ Lock |
|---|---|
| `dt_node_to_map()` وكل الـ utils | بيُستدعى من `pinctrl_dt_to_map()` في الـ core اللي بيمسك `pctldev->mutex` قبل ما ييجي هنا |
| الـ `map` array اللي بتتبني | private للـ driver خلال الـ parsing — مش shared |
| `configs` array | بيتعمله `kmemdup_array` وبيبقى owned بالـ map entry — لا يتشارك |

#### ترتيب الـ Locks في الـ pinctrl core

```
pctldev->mutex          ← الـ lock الرئيسي على الـ controller
    └── pinctrl_maps_mutex  ← يحمي قائمة الـ maps المسجلة globally
```

**قاعدة:** الـ `pinctrl-utils` بيُستدعى وهو تحت `pctldev->mutex` — يعني أي وصول للـ controller state أو الـ map array آمن في السياق ده. الـ driver نفسه مش محتاج يعمل lock إضافي خلال `dt_node_to_map()`.

#### تفصيلة مهمة في الـ Memory Ownership

```
pinctrl_utils_add_map_configs()
  │
  ├─► configs (input) ← مملوكة للـ caller (ممكن stack أو heap)
  │
  └─► dup_configs = kmemdup_array(configs, ...)
          │
          └─► مملوكة بالكامل من map[i].data.configs.configs
                  └─► لازم تتحرر في pinctrl_utils_free_map()
```

يعني الـ caller حر يعمل `kfree` أو يسيب الـ original configs بعد الاستدعاء مباشرةً — الـ utils عمل نسخة خاصة.
## Phase 4: شرح الـ Functions

### ملخص الـ API — Cheatsheet

| Function | Category | الغرض |
|---|---|---|
| `pinctrl_utils_reserve_map` | Memory Management | تخصيص/توسيع array الـ map entries |
| `pinctrl_utils_add_map_mux` | Map Building | إضافة MUX entry للـ map |
| `pinctrl_utils_add_map_configs` | Map Building | إضافة config entry للـ map مع deep copy |
| `pinctrl_utils_add_config` | Config Helpers | إضافة config value واحدة لـ array |
| `pinctrl_utils_free_map` | Cleanup | تحرير الـ map وكل الـ configs المخصصة |

---

### الـ Data Structures المستخدمة

قبل ما نشرح الـ functions، لازم نفهم الـ structures الأساسية:

```c
/* نوع الـ map entry */
enum pinctrl_map_type {
    PIN_MAP_TYPE_INVALID,
    PIN_MAP_TYPE_DUMMY_STATE,
    PIN_MAP_TYPE_MUX_GROUP,      /* pinmux: ربط group بـ function */
    PIN_MAP_TYPE_CONFIGS_PIN,    /* config لـ pin واحد */
    PIN_MAP_TYPE_CONFIGS_GROUP,  /* config لـ group من pins */
};

/* الـ map entry الرئيسي */
struct pinctrl_map {
    const char *dev_name;        /* اسم الـ consumer device */
    const char *name;            /* اسم الـ state (مثلاً "default") */
    enum pinctrl_map_type type;
    const char *ctrl_dev_name;   /* اسم الـ pinctrl controller */
    union {
        struct pinctrl_map_mux mux;         /* لو type == MUX_GROUP */
        struct pinctrl_map_configs configs; /* لو type == CONFIGS_* */
    } data;
};

struct pinctrl_map_mux {
    const char *group;    /* اسم الـ pin group */
    const char *function; /* الـ mux function المطلوبة */
};

struct pinctrl_map_configs {
    const char *group_or_pin; /* اسم الـ pin أو الـ group */
    unsigned long *configs;   /* array من الـ config values */
    unsigned int num_configs; /* عدد الـ configs */
};
```

**السياق العام:** الـ drivers اللي بتـimport pinctrl-utils بتبني `pinctrl_map` array ديناميكياً أثناء `dt_node_to_map()` callback — وده اللي الـ utils دي بتسهله.

---

### Group 1: Memory Management

#### الغرض من المجموعة
الـ `pinctrl_map` array بتتبني تدريجياً أثناء parse الـ Device Tree. مش معروف مقدماً كام entry هتتضاف، فالـ driver محتاج يعمل `realloc` كل ما يلزمه مساحة أكتر. `pinctrl_utils_reserve_map` بتعمل ده بشكل آمن.

---

#### `pinctrl_utils_reserve_map`

```c
int pinctrl_utils_reserve_map(struct pinctrl_dev *pctldev,
        struct pinctrl_map **map, unsigned int *reserved_maps,
        unsigned int *num_maps, unsigned int reserve);
```

**ما بتعمله:** بتتأكد إن الـ map array فيها مساحة كافية لـ `*num_maps + reserve` entries على الأقل. لو المساحة المتاحة كافية، بترجع فوراً بدون allocation. لو مش كافية، بتعمل `krealloc_array` وتـzero-initialize الـ entries الجديدة.

**المعاملات:**
- `pctldev` — الـ pinctrl device، بيتستخدم بس لطباعة رسائل الخطأ عبر `dev_err`
- `map` — مؤشر لمؤشر الـ map array (double pointer عشان الـ realloc ممكن يغير العنوان)
- `reserved_maps` — عدد الـ slots المخصصة حالياً في الـ array (capacity)
- `num_maps` — عدد الـ entries المستخدمة فعلاً (size)
- `reserve` — عدد الـ entries الإضافية المطلوبة

**القيمة المرجعة:** `0` عند النجاح، `-ENOMEM` لو فشل الـ `krealloc_array`.

**تفاصيل مهمة:**
- بتحسب `new_num = *num_maps + reserve` — مش `*reserved_maps + reserve`، يعني بتـbase الـ target على الـ actual usage مش الـ capacity الحالية.
- الـ `memset` بيـzero الـ entries الجديدة فقط (من `old_num` للـ `new_num`)، مش الـ array كلها.
- **لا يوجد locking** — الـ caller مسؤول عن الـ synchronization. في السياق الطبيعي، ده بيحصل في `probe` أو `dt_node_to_map` اللي مفيهاش race condition.
- بعد النجاح، `*reserved_maps` بيتحدّث لـ `new_num` لكن `*num_maps` ما بيتغيرش.

**من بيستدعيها:** الـ pinctrl drivers مباشرةً في تنفيذ `pinctrl_ops.dt_node_to_map`، قبل أي `add_map_*` call.

**Pseudocode:**
```
reserve_map(pctldev, map, reserved_maps, num_maps, reserve):
    new_num = *num_maps + reserve
    if *reserved_maps >= new_num:
        return 0  /* already enough space */

    new_map = krealloc_array(*map, new_num, sizeof(entry))
    if !new_map:
        return -ENOMEM

    zero(new_map[old_num .. new_num])
    *map = new_map
    *reserved_maps = new_num
    return 0
```

---

### Group 2: Map Building

#### الغرض من المجموعة
بعد ما الـ memory اتحجزت، الـ driver بيضيف entries للـ map. في pinctrl، في نوعين من الـ entries: **MUX entries** (بتربط pin group بـ peripheral function) و**Config entries** (بتضبط خصائص الـ pins زي pull-up/down، drive strength، إلخ).

---

#### `pinctrl_utils_add_map_mux`

```c
int pinctrl_utils_add_map_mux(struct pinctrl_dev *pctldev,
        struct pinctrl_map **map, unsigned int *reserved_maps,
        unsigned int *num_maps, const char *group,
        const char *function);
```

**ما بتعمله:** بتكتب `PIN_MAP_TYPE_MUX_GROUP` entry في الـ slot التالي المتاح في الـ map array. بتحدث `*num_maps` بعد كده. الـ strings مش بتتنسخ — بتتخزن بـ pointer مباشرةً (عادةً بتكون string literals أو DT property values لها lifetime أطول من الـ map).

**المعاملات:**
- `pctldev` — بس للـ `WARN_ON` context (مش بيتستخدم فعلياً في الكود)
- `map` — الـ map array
- `reserved_maps` — الـ capacity الحالية (للتحقق من عدم الـ overflow)
- `num_maps` — الـ index الحالي (بيتزود بعد الإضافة)
- `group` — اسم الـ pin group (مثلاً `"uart0_grp"`)
- `function` — اسم الـ mux function (مثلاً `"uart0"`)

**القيمة المرجعة:** `0` عند النجاح، `-ENOSPC` لو `*num_maps == *reserved_maps` (يعني الـ caller نسي يـcall `reserve_map` أولاً).

**تفاصيل مهمة:**
- `WARN_ON(*num_maps == *reserved_maps)` — ده defensive check، من المفروض الـ caller يضمن المساحة مسبقاً.
- **لا يوجد deep copy** للـ strings — الـ ownership بتاعت الـ strings بتفضل عند الـ caller.
- الـ `dev_name` و`ctrl_dev_name` مش بيتاتوا هنا — بيتملوا من الـ caller في حالات تانية أو بيتعملوا set من الـ pinctrl core لما الـ map يتسجل.

**مثال واقعي:**
```c
/* في dt_node_to_map لـ Tegra pinctrl */
ret = pinctrl_utils_reserve_map(pctldev, &map, &reserved_maps,
                                &num_maps, 2);
ret = pinctrl_utils_add_map_mux(pctldev, &map, &reserved_maps,
                                &num_maps, "sdmmc1_grp", "sdmmc1");
```

---

#### `pinctrl_utils_add_map_configs`

```c
int pinctrl_utils_add_map_configs(struct pinctrl_dev *pctldev,
        struct pinctrl_map **map, unsigned int *reserved_maps,
        unsigned int *num_maps, const char *group,
        unsigned long *configs, unsigned int num_configs,
        enum pinctrl_map_type type);
```

**ما بتعمله:** بتضيف config entry (إما `PIN_MAP_TYPE_CONFIGS_PIN` أو `PIN_MAP_TYPE_CONFIGS_GROUP`) للـ map. **بتعمل deep copy** للـ configs array عبر `kmemdup_array` — ده مهم لأن الـ caller's array ممكن تكون stack-allocated أو temp buffer.

**المعاملات:**
- `pctldev` — للـ WARN_ON
- `map`, `reserved_maps`, `num_maps` — نفس الـ pattern
- `group` — اسم الـ pin أو الـ group (الاسم مشترك للنوعين)
- `configs` — pointer لـ array من الـ config values (بيتعمل لها `kmemdup`)
- `num_configs` — حجم الـ configs array
- `type` — `PIN_MAP_TYPE_CONFIGS_PIN` أو `PIN_MAP_TYPE_CONFIGS_GROUP`

**القيمة المرجعة:** `0` عند النجاح، `-ENOSPC` لو المساحة مش كافية، `-ENOMEM` لو فشل الـ `kmemdup_array`.

**تفاصيل مهمة:**
- `kmemdup_array(configs, num_configs, sizeof(*dup_configs), GFP_KERNEL)` — بتعمل `num_configs * sizeof(unsigned long)` bytes مخصصة في الـ heap.
- الـ memory المخصصة هنا بتتحرر في `pinctrl_utils_free_map` — ده الـ ownership model.
- لو `num_configs == 0` فـ `kmemdup_array` ممكن يرجع non-NULL pointer صغير أو NULL حسب الـ implementation — الـ caller لازم يضمن `num_configs > 0`.
- الـ `group` string مش بتتنسخ، نفس `add_map_mux`.

**الفرق بين `CONFIGS_PIN` و`CONFIGS_GROUP`:**
- `CONFIGS_PIN` → بيـapply على pin واحد بالاسم
- `CONFIGS_GROUP` → بيـapply على كل الـ pins اللي في الـ group

**Pseudocode:**
```
add_map_configs(pctldev, map, reserved_maps, num_maps,
                group, configs, num_configs, type):
    if *num_maps == *reserved_maps:
        WARN_ON; return -ENOSPC

    dup = kmemdup_array(configs, num_configs, sizeof(ulong))
    if !dup: return -ENOMEM

    map[*num_maps].type = type
    map[*num_maps].data.configs.group_or_pin = group
    map[*num_maps].data.configs.configs = dup
    map[*num_maps].data.configs.num_configs = num_configs
    (*num_maps)++
    return 0
```

---

### Group 3: Config Helpers

#### الغرض من المجموعة
أثناء parsing الـ DT، الـ driver بيقرأ config properties واحدة واحدة وبيحتاج يبني array منها تدريجياً. `pinctrl_utils_add_config` بتوفر dynamic append لـ `unsigned long` array.

---

#### `pinctrl_utils_add_config`

```c
int pinctrl_utils_add_config(struct pinctrl_dev *pctldev,
        unsigned long **configs, unsigned int *num_configs,
        unsigned long config);
```

**ما بتعمله:** بتضيف `config` value واحدة لـ `*configs` array عبر `krealloc`. بتزود حجم الـ array بـ 1 في كل استدعاء. الناتج النهائي يتمرر لـ `pinctrl_utils_add_map_configs`.

**المعاملات:**
- `pctldev` — لـ `dev_err` لو فشل الـ krealloc
- `configs` — double pointer للـ array (بيتغير عنوانها بعد الـ realloc)
- `num_configs` — الحجم الحالي (بيتزود بعد الإضافة)
- `config` — الـ `unsigned long` config value الجديدة (عادةً encoded بـ `PIN_CONF_PACKED` أو macro خاص بالـ driver)

**القيمة المرجعة:** `0` عند النجاح، `-ENOMEM` لو فشل الـ `krealloc`.

**تفاصيل مهمة:**
- بتستخدم `krealloc` مش `krealloc_array` — لكن بتحسب الحجم يدوياً بـ `sizeof(*new_configs) * new_num`. ده آمن لأن `unsigned long` مفيش حاجة تغلط فيها.
- لو `*configs == NULL` و`*num_configs == 0`، الـ `krealloc(NULL, ...)` بتتصرف زي `kmalloc` — ده الـ initial call pattern.
- الـ caller مسؤول عن تحرير `*configs` لما يخلص (أو عبر `add_map_configs` اللي بتعمل kmemdup، وبعدين `free_map` اللي بتحرر الـ dup).

**مثال واقعي — pattern شائع في drivers:**
```c
unsigned long *configs = NULL;
unsigned int num_configs = 0;

/* بنقرأ كل property من DT وبنضيفها */
pinctrl_utils_add_config(pctldev, &configs, &num_configs,
                         PINCONF_PACK(PIN_CONFIG_BIAS_PULL_UP, 1));
pinctrl_utils_add_config(pctldev, &configs, &num_configs,
                         PINCONF_PACK(PIN_CONFIG_DRIVE_STRENGTH, 8));

/* بعدين بنضيف الـ array للـ map (بتعمل kmemdup داخلياً) */
pinctrl_utils_add_map_configs(pctldev, &map, &reserved_maps,
                              &num_maps, group_name,
                              configs, num_configs,
                              PIN_MAP_TYPE_CONFIGS_GROUP);

/* بنحرر الـ temp array */
kfree(configs);
```

---

### Group 4: Cleanup

#### الغرض من المجموعة
الـ `dt_node_to_map` callback بتبني map ديناميكياً، والـ pinctrl core بتاخدها وتـregister الـ entries. لما الـ driver يتفـunload أو يتعمل لها error recovery، لازم يتحرر كل الـ heap memory. `pinctrl_utils_free_map` بتعمل ده بشكل صحيح مراعيةً الـ ownership rules.

---

#### `pinctrl_utils_free_map`

```c
void pinctrl_utils_free_map(struct pinctrl_dev *pctldev,
        struct pinctrl_map *map, unsigned int num_maps);
```

**ما بتعمله:** بتـiterate على كل الـ map entries، وبتحرر الـ `configs` array في أي entry من نوع `CONFIGS_GROUP` أو `CONFIGS_PIN` (لأن دول اللي عندهم heap-allocated configs من `add_map_configs`). بعدين بتحرر الـ map array نفسها.

**المعاملات:**
- `pctldev` — مش بيتستخدم فعلياً في الكود الحالي (legacy parameter)
- `map` — الـ map array المراد تحريرها
- `num_maps` — عدد الـ entries الفعلية

**القيمة المرجعة:** لا يوجد (void).

**تفاصيل مهمة:**
- بتحرر فقط `map[i].data.configs.configs` — مش الـ strings (`group_or_pin`, `group`, `function`) لأن دول مش owned بالـ map.
- `PIN_MAP_TYPE_MUX_GROUP` و`PIN_MAP_TYPE_DUMMY_STATE` مش بيعملوا لهم `kfree` لأن مفيش heap allocation ليهم في الـ utils دي.
- **لازم** الـ caller يـcall دي على error path في `dt_node_to_map` قبل رجوع الـ error، وكمان في `dt_free_map` callback.
- **لا يوجد double-free protection** — الـ caller مسؤول إنه ما يـcall الـ function دي مرتين على نفس الـ map.

**Pseudocode:**
```
free_map(pctldev, map, num_maps):
    for i in 0..num_maps:
        if map[i].type == CONFIGS_GROUP or CONFIGS_PIN:
            kfree(map[i].data.configs.configs)
    kfree(map)
```

**مثال — تنفيذ `dt_free_map` في driver:**
```c
static void my_pinctrl_dt_free_map(struct pinctrl_dev *pctldev,
                                   struct pinctrl_map *map,
                                   unsigned int num_maps)
{
    /* free_map بتتكلف بتحرير الـ configs arrays والـ map نفسها */
    pinctrl_utils_free_map(pctldev, map, num_maps);
}
```

---

### الـ Lifecycle الكامل — كيف بتتعاون الـ Functions

```
dt_node_to_map() callback:
│
├─1─ pinctrl_utils_reserve_map()   ← حجز مساحة كافية
│
├─2─ pinctrl_utils_add_map_mux()   ← إضافة MUX entry (لو في function مطلوبة)
│     │
│     └─ (no heap alloc for configs)
│
├─3─ pinctrl_utils_add_config()    ← بناء configs array تدريجياً
│    pinctrl_utils_add_config()
│    pinctrl_utils_add_config()
│         │
│         └─ krealloc في كل مرة
│
├─4─ pinctrl_utils_add_map_configs() ← إضافة config entry مع kmemdup
│         │
│         └─ kmemdup_array → dup_configs (heap)
│
├─5─ kfree(configs)                ← تحرير الـ temp configs array
│
└─ return map, num_maps to pinctrl core
        │
        ▼
    pinctrl core يـregister الـ map
        │
        ▼ (عند unload أو error)
    dt_free_map() →
        pinctrl_utils_free_map()   ← تحرير dup_configs + map array
```

---

### جدول الـ Memory Ownership

| ما الذي يتم تخصيصه | أين يتم التخصيص | أين يتم التحرير |
|---|---|---|
| الـ `map` array نفسها | `reserve_map` (krealloc) | `free_map` (kfree(map)) |
| `dup_configs` في CONFIGS entries | `add_map_configs` (kmemdup_array) | `free_map` (kfree(configs)) |
| الـ temp `configs` array | `add_config` (krealloc) | الـ caller بعد `add_map_configs` |
| الـ strings (group, function, ...) | الـ caller (DT أو static) | الـ caller |
## Phase 5: دليل الـ Debugging الشامل

> **الـ scope:** ملف `pinctrl-utils.c` بيقدم utility functions اللي بتستخدمها drivers جوة الـ pinctrl subsystem عشان تبني الـ `pinctrl_map` table ديناميكيًا من الـ Device Tree. الـ debugging هنا بيشمل الـ map allocation وتسجيل الـ configs وربطها بالـ groups والـ functions.

---

### Software Level

#### 1. debugfs Entries

الـ pinctrl subsystem بيعرض معلومات غنية جدًا عبر **debugfs**. مفيش debugfs entries خاصة بـ `pinctrl-utils.c` نفسها لأنها مجرد helpers، لكن الـ maps اللي بتبنيها بتظهر هنا:

| المسار | المحتوى |
|--------|---------|
| `/sys/kernel/debug/pinctrl/` | directory فيها entry لكل pin controller مسجل |
| `/sys/kernel/debug/pinctrl/<ctrl-name>/pins` | كل الـ pins المسجلة مع أرقامها وأسمائها |
| `/sys/kernel/debug/pinctrl/<ctrl-name>/pingroups` | الـ pin groups وأي pins بتنتمي لكل group |
| `/sys/kernel/debug/pinctrl/<ctrl-name>/pinmux-functions` | الـ mux functions المتاحة |
| `/sys/kernel/debug/pinctrl/<ctrl-name>/pinmux-pins` | كل pin وإيه الـ function المحطوطة عليه دلوقتي |
| `/sys/kernel/debug/pinctrl/<ctrl-name>/pinconf-pins` | الـ config parameters لكل pin |
| `/sys/kernel/debug/pinctrl/<ctrl-name>/pinconf-groups` | الـ config parameters على مستوى الـ group |

```bash
# اقرأ كل الـ pin controllers المسجلة
ls /sys/kernel/debug/pinctrl/

# اعرض الـ groups وإيه الـ pins فيها
cat /sys/kernel/debug/pinctrl/pinctrl@30330000/pingroups

# اعرض mux state الحالية
cat /sys/kernel/debug/pinctrl/pinctrl@30330000/pinmux-pins

# اعرض الـ pin configs (pull, drive, etc.)
cat /sys/kernel/debug/pinctrl/pinctrl@30330000/pinconf-pins
```

**مثال على output من `pingroups`:**
```
registered pin groups:
group: uart0_grp
pin 12 (UART0_TX)
pin 13 (UART0_RX)

group: spi0_grp
pin 20 (SPI0_CLK)
pin 21 (SPI0_MOSI)
pin 22 (SPI0_MISO)
pin 23 (SPI0_CS)
```

عشان تتحقق إن `pinctrl_utils_add_map_mux` اشتغلت صح: اسم الـ group المحطوطة في الـ map لازم تطابق اسم group موجودة في `pingroups`.

---

#### 2. sysfs Entries

| المسار | المحتوى |
|--------|---------|
| `/sys/bus/platform/drivers/` | بتلاقي الـ pinctrl driver مسجل هنا |
| `/sys/devices/<dev>/pinctrl-names` | لو الـ driver عامل `devm_pinctrl_get` بيظهر الـ states |
| `/sys/class/gpio/` | الـ GPIO ranges المربوطة بالـ pin controller |

```bash
# شوف الـ pinctrl device
ls /sys/devices/platform/ | grep pinctrl

# تحقق من الـ driver binding
cat /sys/devices/platform/pinctrl@30330000/driver/module/version
```

---

#### 3. ftrace — Tracepoints والـ Events

الـ pinctrl subsystem عنده tracepoints مدمجة. الـ `pinctrl-utils.c` بالذات بيستخدم `krealloc_array` و`kmemdup_array`، لو فيه مشكلة في الـ memory allocation هتظهر هنا.

```bash
# فعّل الـ pinctrl events كلها
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# أو فعّل events محددة
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_dt_subnode_reg/enable
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_dt_node_found/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اعمل reload للـ driver أو boot
# ثم اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace

# وقف الـ trace
echo 0 > /sys/kernel/debug/tracing/tracing_on
```

عشان تـ trace مشاكل الـ map building على وجه التحديد:
```bash
# تابع كل kmalloc/krealloc calls من pinctrl driver
echo 'module pinctrl_utils' > /sys/kernel/debug/tracing/set_ftrace_filter 2>/dev/null || true

# استخدم function tracer
echo function > /sys/kernel/debug/tracing/current_tracer
echo pinctrl_utils_reserve_map > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. printk والـ Dynamic Debug

الـ `pinctrl-utils.c` بيستخدم `dev_err` في حالتين فقط: لما `krealloc_array` تفشل في `pinctrl_utils_reserve_map`، ولما `krealloc` تفشل في `pinctrl_utils_add_config`.

```bash
# فعّل dynamic debug على ملف pinctrl-utils.c بالكامل
echo 'file pinctrl-utils.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لـ pinctrl subsystem كله
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug للـ driver الخاص بالـ hardware
echo 'module <driver-name> +p' > /sys/kernel/debug/dynamic_debug/control

# اعرض الـ dynamic debug rules النشطة حاليًا
cat /sys/kernel/debug/dynamic_debug/control | grep pinctrl
```

لو عايز تضيف debug prints مؤقتة جوة driver بيستخدم الـ utils:
```c
/* في dt_node_to_map callback */
dev_dbg(pctldev->dev, "reserving %d maps for node %s\n",
        reserve, np->name);
```

```bash
# رفع مستوى kernel log عشان تشوف dev_dbg
echo 8 > /proc/sys/kernel/printk

# أو بالـ dmesg
dmesg -w | grep pinctrl
```

---

#### 5. Kernel Config Options للـ Debugging

| الـ Config | الوظيفة |
|-----------|---------|
| `CONFIG_PINCTRL` | لازم يكون enabled عشان الـ subsystem يشتغل |
| `CONFIG_DEBUG_PINCTRL` | بيفعّل verbose debugging في الـ pinctrl core |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | لازم لدعم generic group management |
| `CONFIG_GENERIC_PINCONF` | بيفعّل generic pin configuration |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | generic mux function support |
| `CONFIG_DEBUG_KERNEL` | prerequisite لمعظم debug options |
| `CONFIG_DYNAMIC_DEBUG` | بيفعّل `dev_dbg` و`pr_debug` |
| `CONFIG_KASAN` | بيكشف heap use-after-free في الـ krealloc calls |
| `CONFIG_DEBUG_SLAB` | بيكشف double-free في `pinctrl_utils_free_map` |
| `CONFIG_SLUB_DEBUG` | alternative لـ `CONFIG_DEBUG_SLAB` على SLUB |
| `CONFIG_DEBUG_OBJECTS` | بيراقب object lifecycles |
| `CONFIG_LOCKDEP` | بيكشف lock ordering issues في الـ pinctrl core |
| `CONFIG_PROVE_LOCKING` | extended lockdep |

```bash
# تحقق من الـ config الحالية
zcat /proc/config.gz | grep -E 'PINCTRL|GENERIC_PIN'

# أو
grep -E 'CONFIG_PINCTRL|CONFIG_GENERIC_PIN|CONFIG_DEBUG_PIN' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الأداة الرئيسية: `pinctrl` userspace tool**

```bash
# install
apt install pinctrl-tools   # Debian/Ubuntu
# أو من المصدر: tools/gpio/ في kernel tree

# اعرض الـ pin states
pinctrl list

# اعرض مين طالب كل pin
pinctrl status

# اعرض configs
pinctrl get <pin-number>
```

**استخدام `/sys/kernel/debug/pinctrl` مباشرة:**

```bash
# script بيطبع ملخص كامل للـ pinctrl state
for ctrl in /sys/kernel/debug/pinctrl/*/; do
    echo "=== Controller: $ctrl ==="
    echo "--- Groups ---"
    cat "${ctrl}pingroups" 2>/dev/null
    echo "--- Mux Pins ---"
    cat "${ctrl}pinmux-pins" 2>/dev/null
    echo "--- Pin Configs ---"
    cat "${ctrl}pinconf-pins" 2>/dev/null
    echo ""
done
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | السبب | الحل |
|---------------------|-------|------|
| `krealloc(map) failed` | الـ heap مفيش ذاكرة كافية لتوسيع الـ map array في `pinctrl_utils_reserve_map` | قلل الـ `reserve` size، افحص `CONFIG_KASAN` للـ heap corruption |
| `krealloc(configs) failed` | الـ heap مفيش ذاكرة لإضافة config entry في `pinctrl_utils_add_config` | نفس الحل، افحص الـ memory pressure |
| `WARN_ON(*num_maps == *reserved_maps)` | في `add_map_mux` أو `add_map_configs`، الـ driver طلب إضافة map entry بدون ما يعمل `reserve_map` الأول | الـ driver لازم يحسب الـ `reserve` صح قبل ما يبدأ يضيف entries |
| `pinctrl: unknown pin X` | الـ map بتشاور على pin number مش مسجل | تحقق إن الـ pin descriptor موجود في `pinctrl_desc.pins` |
| `pinctrl: group X not found` | الاسم في `pinctrl_map_mux.group` مش موجود | تحقق من `dt_node_to_map` إن الاسم بيتطابق مع الـ pingroup المسجلة |
| `could not find function X` | الـ mux function غير مسجلة | تحقق من `pinmux_ops` والـ function table |
| `pin already requested` | pin محاولة تحجزها مرتين | تحقق من `pinmux-pins` في debugfs لمين حاجزها |
| `-ENOSPC` returned | الـ `num_maps` وصل الـ `reserved_maps` قبل ما تكمل | زود الـ `reserve` في `pinctrl_utils_reserve_map` |
| `-ENOMEM` returned silently | الـ `kmemdup_array` فشلت في `add_map_configs` | زود الـ memory أو استخدم KASAN |

---

#### 8. أماكن استراتيجية لـ `dump_stack()` و`WARN_ON()`

الـ `pinctrl-utils.c` فعلًا بيستخدم `WARN_ON` في موضعين:

```c
/* في pinctrl_utils_add_map_mux — السطر 50 */
if (WARN_ON(*num_maps == *reserved_maps))
    return -ENOSPC;

/* في pinctrl_utils_add_map_configs — السطر 70 */
if (WARN_ON(*num_maps == *reserved_maps))
    return -ENOSPC;
```

لو عايز تضيف debugging إضافي جوة driver بيستخدم الـ utils:

```c
/* Strategic point 1: قبل reserve_map — تحقق من حساب الـ reserve */
static int my_dt_node_to_map(struct pinctrl_dev *pctldev,
                              struct device_node *np,
                              struct pinctrl_map **map,
                              unsigned int *num_maps)
{
    unsigned int reserved_maps = 0;
    int ret;

    /* حساب العدد الكلي للـ maps المطلوبة */
    unsigned int total = count_needed_maps(np);
    WARN_ON(total == 0);  /* لو 0 فيه مشكلة في الـ DT parsing */

    ret = pinctrl_utils_reserve_map(pctldev, map,
                                    &reserved_maps, num_maps, total);
    if (ret) {
        dump_stack();  /* Strategic: تفيد لو فيه memory pressure غريب */
        return ret;
    }
    /* ... */
}

/* Strategic point 2: بعد free_map — تحقق من consistency */
void my_dt_free_map(struct pinctrl_dev *pctldev,
                    struct pinctrl_map *map, unsigned int num_maps)
{
    WARN_ON(!map && num_maps > 0);  /* inconsistency detector */
    pinctrl_utils_free_map(pctldev, map, num_maps);
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

الـ pinctrl-utils بيبني الـ map, لكن الـ hardware state بيتطبق فعلًا لما الـ driver يعمل `pinctrl_select_state`. عشان تتحقق:

```
Kernel Map (pinctrl_map)
        ↓
pinctrl_select_state()
        ↓
pmxops->set_mux() + confops->pin_config_set()
        ↓
Hardware Registers (Physical Pins)
```

```bash
# خطوة 1: تحقق من الـ kernel state
cat /sys/kernel/debug/pinctrl/<ctrl>/pinmux-pins
# Expected output:
# pin 12 (UART0_TX): uart0 GPIO UNCLAIMED

# خطوة 2: قارنه بالـ datasheet register
# على Raspberry Pi مثلاً:
# GPFSEL0 register - كل 3 bits بتحدد function
devmem2 0x3F200000 w   # GPIO Function Select 0
```

#### 2. Register Dump Techniques

```bash
# طريقة 1: devmem2 (يحتاج install)
apt install devmem2
# اقرأ الـ pinmux register على عنوان محدد من الـ datasheet
devmem2 0x30330000 w    # read 32-bit word
devmem2 0x30330000 b    # read byte
devmem2 0x30330004 w    # التالي

# طريقة 2: /dev/mem (kernel لازم compile بـ CONFIG_DEVMEM=y)
dd if=/dev/mem bs=4 count=1 skip=$((0x30330000/4)) 2>/dev/null | xxd

# طريقة 3: io utility
io -4 0x30330000     # read 32-bit
io -4 -r 0x30330000 0x100   # dump 256 bytes

# طريقة 4: من جوة kernel driver (أفضل وأأمن)
# استخدم devm_ioremap ثم readl/writel
void dump_pinmux_regs(struct platform_device *pdev)
{
    void __iomem *base = devm_platform_ioremap_resource(pdev, 0);
    int i;
    for (i = 0; i < 0x100; i += 4)
        dev_info(&pdev->dev, "reg[0x%x] = 0x%08x\n",
                 i, readl(base + i));
}
```

#### 3. Logic Analyzer / Oscilloscope

لما الـ pinctrl map اتبنت صح وإنت لسه مش شايف الـ hardware يشتغل:

```
تحقق من:
┌─────────────────────────────────────────────────────────────┐
│  1. الـ pin function محطوطة صح (MUX select bits)           │
│  2. الـ pull-up/pull-down configured                        │
│  3. الـ drive strength مناسب للـ bus speed                  │
│  4. الـ voltage level (1.8V vs 3.3V IO)                     │
│  5. الـ slew rate                                           │
└─────────────────────────────────────────────────────────────┘
```

```bash
# Tips للـ logic analyzer:
# - راقب الـ pin على scope بعد ما الـ kernel driver يعمل probe
# - لو الـ pin مش بيتحرك: غالبًا الـ mux set_mux لم يُستدعى
# - لو بتشوف glitch وقت الـ probe: الـ config لم يُطبق بالتسلسل الصح

# في الـ kernel، الـ pinctrl_select_state بيستدعي:
# 1. pinmux_enable_setting → set_mux
# 2. pinconf_apply_setting → pin_config_set / pin_config_group_set
# تحقق إن الـ sequence صح في الـ oscilloscope
```

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| المشكلة | نمط الـ Kernel Log | التشخيص |
|---------|-------------------|---------|
| الـ pin مش بيتعرفش عليه | `pinctrl: unknown pin <N>` | الـ pin number في الـ DT أكبر من الـ npins في الـ pinctrl_desc |
| الـ mux مش بيتطبقش | `pinctrl: could not request pin <N>` | الـ pin مش موجود في الـ group المطلوبة |
| تعارض بين دريفرين | `pin already requested by <dev>` | دريفرين بيطلبوا نفس الـ pin |
| الـ function غلط | `could not find function <name>` | اسم الـ function في الـ DT مش موجود في `pinmux_ops` |
| الـ config مش بيتطبقش | `pinctrl: could not apply group pins` | `confops->pin_config_group_set` بيرجع error |
| الـ hardware مش بيستجيبش | لا errors لكن الـ device مش شغال | الـ register base address غلط أو الـ clock مش enabled |

#### 5. Device Tree Debugging

الـ `pinctrl-utils.c` بيُستخدم أساسًا من `dt_node_to_map`. عشان تتحقق إن الـ DT صح:

```bash
# اعرض الـ DT compiled (DTB) كـ DTS
dtc -I dtb -O dts /proc/device-tree > /tmp/current_dt.dts 2>/dev/null

# ابحث عن pinctrl nodes
grep -A 20 "pinctrl" /tmp/current_dt.dts | head -60

# تحقق من الـ pin groups المعرفة
grep -E "pinctrl-[0-9]|pinctrl-names" /tmp/current_dt.dts | head -20

# تحقق إن الـ device بيشاور على الـ pinctrl بشكل صح
cat /proc/device-tree/<device-node>/pinctrl-names 2>/dev/null
cat /proc/device-tree/<device-node>/pinctrl-0 2>/dev/null | xxd
```

**مثال على DT node صح:**
```dts
/* الـ pinctrl controller */
pinctrl: pinctrl@30330000 {
    compatible = "vendor,pinctrl";
    reg = <0x30330000 0x10000>;

    uart0_pins: uart0-grp {
        pins = "UART0_TX", "UART0_RX";
        function = "uart0";
        drive-strength = <4>;
        bias-pull-up;
    };
};

/* الـ consumer device */
uart0: serial@30890000 {
    compatible = "vendor,uart";
    reg = <0x30890000 0x1000>;
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart0_pins>;     /* default state */
    pinctrl-1 = <&uart0_sleep_pins>; /* sleep state */
};
```

```bash
# تحقق من الـ phandle references
grep -r "uart0_pins\|pinctrl-0" /proc/device-tree/ 2>/dev/null

# تحقق من فشل dt_node_to_map بالـ dynamic debug
echo 'file <driver>.c +p' > /sys/kernel/debug/dynamic_debug/control
dmesg | grep -E 'pinctrl|dt_node_to_map'
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للـ Copy-Paste

**1. فحص شامل للـ pinctrl state:**
```bash
#!/bin/bash
# pinctrl-debug.sh — comprehensive pinctrl state dump

echo "======================================="
echo " Registered Pin Controllers"
echo "======================================="
ls /sys/kernel/debug/pinctrl/

for CTRL in /sys/kernel/debug/pinctrl/*/; do
    NAME=$(basename "$CTRL")
    echo ""
    echo "======================================="
    echo " Controller: $NAME"
    echo "======================================="

    echo "--- Pin List ---"
    cat "${CTRL}pins" 2>/dev/null | head -30

    echo "--- Pin Groups ---"
    cat "${CTRL}pingroups" 2>/dev/null

    echo "--- Mux Functions ---"
    cat "${CTRL}pinmux-functions" 2>/dev/null

    echo "--- Mux Pin State ---"
    cat "${CTRL}pinmux-pins" 2>/dev/null | grep -v UNCLAIMED | head -20

    echo "--- Pin Configs ---"
    cat "${CTRL}pinconf-pins" 2>/dev/null | head -30
done
```

**2. تفعيل الـ ftrace للـ pinctrl:**
```bash
#!/bin/bash
# enable pinctrl tracing

cd /sys/kernel/debug/tracing

# reset
echo 0 > tracing_on
echo > trace

# enable pinctrl events
echo 1 > events/pinctrl/enable 2>/dev/null || \
    echo "No pinctrl tracepoints available"

# set function filter
echo pinctrl_utils_reserve_map > set_ftrace_filter 2>/dev/null
echo pinctrl_utils_add_map_mux >> set_ftrace_filter 2>/dev/null
echo pinctrl_utils_add_map_configs >> set_ftrace_filter 2>/dev/null
echo pinctrl_utils_add_config >> set_ftrace_filter 2>/dev/null
echo pinctrl_utils_free_map >> set_ftrace_filter 2>/dev/null

# use function tracer
echo function > current_tracer

# start
echo 1 > tracing_on
echo "Tracing started. Reload your driver now..."
echo "Then run: cat /sys/kernel/debug/tracing/trace"
```

**3. تشخيص مشكلة memory في الـ map building:**
```bash
# فحص الـ slab allocator
cat /proc/slabinfo | grep -i "size-\|kmalloc"

# فحص الـ memory pressure
free -m
cat /proc/meminfo | grep -E 'MemFree|Slab|KernelStack'

# تفعيل SLUB debugging
echo 1 > /sys/kernel/slab/kmalloc-64/trace 2>/dev/null
# أو عند الـ boot: slub_debug=FPZU kernel parameter
```

**4. تحقق من الـ DT parsing:**
```bash
# فعّل verbose DT parsing
echo 1 > /sys/module/of/parameters/of_debug 2>/dev/null

# أو عند الـ boot
# kernel cmdline: of_debug=1

# ثم اقرأ الـ dmesg
dmesg | grep -E 'OF:|pinctrl|dt_node_to_map' | head -50
```

**5. مثال على output حقيقي وتفسيره:**
```bash
$ cat /sys/kernel/debug/pinctrl/30330000.pinctrl/pinmux-pins
```
```
Pinmux settings per pin
Format: pin (name): mux_owner gpio_owner hog?
pin 0 (GPIO0): (MUX UNCLAIMED) (GPIO UNCLAIMED)
pin 1 (GPIO1): (MUX UNCLAIMED) (GPIO UNCLAIMED)
pin 12 (UART0_TX): uart0 (GPIO UNCLAIMED)
pin 13 (UART0_RX): uart0 (GPIO UNCLAIMED)
pin 20 (SPI0_CLK): (MUX UNCLAIMED) (GPIO UNCLAIMED)
```

**التفسير:**
- **`UNCLAIMED`** → الـ pin مش محجوز، الـ pinctrl_utils_add_map_mux لم تُستدعى، أو الـ `pinctrl_select_state` لم ينجح.
- **`uart0`** → الـ UART driver حجز الـ pin بنجاح عبر الـ map اللي `pinctrl-utils` بنتها.
- الـ `hog?` بيظهر لو الـ pin controller نفسه هو المالك (self-hogging).

```bash
# لو pin ظهر UNCLAIMED رغم إن المفروض يكون معرّف في DT:
# 1. تحقق من الـ map
dmesg | grep "pinctrl.*map\|dt_node_to_map"

# 2. تحقق من الـ state selection
dmesg | grep "pinctrl.*select\|pinctrl.*state"

# 3. تحقق من الـ WARN_ON في pinctrl-utils
dmesg | grep "WARNING\|WARN_ON" | grep -i pinctrl
```

**6. فحص الـ configs على pin محدد:**
```bash
$ cat /sys/kernel/debug/pinctrl/30330000.pinctrl/pinconf-pins | grep -A 5 "pin 12"
```
```
pin 12 (UART0_TX):
  drive-strength: 4 mA
  bias-pull-up
  input-disable
  output-enable
```

**التفسير:** الـ configs دي اتبنت عبر `pinctrl_utils_add_map_configs` وتخزنت كـ `pinctrl_map_configs.configs` array، وتطبقت عبر `confops->pin_config_set`.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — كرنل panic وقت probe بسبب ENOMEM في `pinctrl_utils_reserve_map`

#### العنوان
**Memory exhaustion أثناء DT parsing لـ pinctrl driver مكتوب حديثاً على RK3562**

#### السياق
بتشتغل على industrial gateway بيستخدم **Rockchip RK3562**، الـ gateway ده بيتحكم في 3 أجهزة UART و2 SPI و1 I2C. كتبت pinctrl driver جديد للـ SoC ده وبدأت تعمل bring-up. الـ board بتعمل boot لكن الـ kernel بيطبع error وبيوقف قبل ما أي peripheral يشتغل.

#### المشكلة
الـ dmesg بيظهر:

```
[    1.432] rk3562-pinctrl pinctrl: krealloc(map) failed
[    1.433] rk3562-pinctrl pinctrl: Error parsing pin config
[    1.434] rk3562-pinctrl: probe of pinctrl failed with error -12
```

الـ `-12` هو `ENOMEM`. الـ peripherals كلها مش شغالة لأن الـ pinctrl driver فشل.

#### التحليل
الخطأ بييجي من `pinctrl_utils_reserve_map` في السطر 33:

```c
new_map = krealloc_array(*map, new_num, sizeof(*new_map), GFP_KERNEL);
if (!new_map) {
    dev_err(pctldev->dev, "krealloc(map) failed\n");
    return -ENOMEM;
}
```

الـ driver بتاعك في `dt_node_to_map()` بتعمل `reserve_map` لكل pin على حدة داخل loop:

```c
/* BAD: بيعمل reserve لكل pin لوحده */
for_each_child_of_node(np, child) {
    ret = pinctrl_utils_reserve_map(pctldev, &map,
                &reserved_maps, &num_maps, 1); /* reserve=1 فقط */
    /* ... */
}
```

المشكلة إن `krealloc_array` بتتكلم مع الـ allocator كتير جداً — كل iteration بتعمل `krealloc` من جديد. لو الـ DT فيه 200 pin node، الـ fragmentation بيخلي الـ kernel يفشل في إيجاد block متراص حتى لو في memory كافية.

الـ flow في `reserve_map`:
```
old_num = *reserved_maps   (مثلاً 150)
new_num = *num_maps + reserve = 150 + 1 = 151

old_num (150) < new_num (151) → بيدخل الـ krealloc
```

كل iteration بتزود الـ buffer بـ 1 فقط، يعني 200 `krealloc` call متتالية.

#### الحل
قبل ما تدخل الـ loop، احسب العدد الكلي للـ maps المطلوبة وعمل reserve واحد:

```c
/* GOOD: احسب الكل الأول */
total = 0;
for_each_child_of_node(np, child) {
    /* كل child ممكن يحتاج mux entry + configs entry */
    total += 2;
}

ret = pinctrl_utils_reserve_map(pctldev, &map,
            &reserved_maps, &num_maps, total);
if (ret)
    return ret;

/* دلوقتي ادخل الـ loop بأمان */
for_each_child_of_node(np, child) {
    ret = pinctrl_utils_add_map_mux(pctldev, &map,
                &reserved_maps, &num_maps, group, func);
    /* ... */
}
```

الـ `reserve_map` بتشيك في بداية الدالة:
```c
if (old_num >= new_num)
    return 0;  /* مفيش حاجة تتعمل — الـ buffer كافي */
```

فلو عملت reserve بـ `total` مرة واحدة، كل الـ calls الجاية بترجع 0 على طول بدون `krealloc`.

#### الدرس المستفاد
**الـ `reserve` parameter في `pinctrl_utils_reserve_map` ليس "reserve one more" — هو "ensure at least this many free slots".** استخدمه مرة واحدة قبل الـ loop بالعدد الكلي المتوقع، مش جوه الـ loop.

---

### السيناريو 2: Android TV Box على Allwinner H616 — UART1 بيطلع garbage بسبب config leak

#### العنوان
**Memory leak في `pinctrl_utils_add_map_configs` بيسبب corruption في UART1 على Allwinner H616**

#### السياق
بتعمل Android TV box على **Allwinner H616**. الـ UART1 مستخدم للـ Bluetooth module. الـ box بيشتغل تمام لكن بعد كذا hour، الـ Bluetooth بيوقف وأحياناً الـ kernel بيطبع `BUG: memory leak detected`.

#### المشكلة
الـ driver عنده خطأ في error path: لما `pinctrl_utils_add_map_configs` بتنجح، الـ configs array بتتعمل `kmemdup_array` جواها:

```c
dup_configs = kmemdup_array(configs, num_configs,
                sizeof(*dup_configs), GFP_KERNEL);
```

لكن لما بيحصل error بعد كده في نفس الـ `dt_node_to_map()` call، الـ driver بيعمل cleanup ناقص:

```c
/* BAD: الـ driver بيعمل kfree للـ map array بس */
err_exit:
    kfree(map);  /* هنا ناقص! */
    return ret;
```

المشكلة إن `dup_configs` اللي اتعملت جوه `pinctrl_utils_add_map_configs` مش بتتحرر. الـ `pinctrl_utils_free_map` موجودة عشان تعمل ده:

```c
void pinctrl_utils_free_map(struct pinctrl_dev *pctldev,
          struct pinctrl_map *map, unsigned int num_maps)
{
    for (i = 0; i < num_maps; i++) {
        switch (map[i].type) {
        case PIN_MAP_TYPE_CONFIGS_GROUP:
        case PIN_MAP_TYPE_CONFIGS_PIN:
            kfree(map[i].data.configs.configs); /* ده المهم */
            break;
        }
    }
    kfree(map);
}
```

#### التحليل
الـ flow بيكون:

```
dt_node_to_map() {
    pinctrl_utils_reserve_map(...)        → allocates map array
    pinctrl_utils_add_map_mux(...)        → fills map[0]
    pinctrl_utils_add_map_configs(...)    → kmemdup_array → fills map[1]
    /* حصل error هنا */
    goto err_exit;

err_exit:
    kfree(map);  ← بتحرر map array بس!
                   map[1].data.configs.configs لسه allocated
}
```

الـ `dup_configs` pointer موجود جوه `map[1].data.configs.configs` لكن اتعمل `kfree` للـ array نفسها من غير ما يتعمل kfree للـ configs pointer الجوه.

#### الحل
استبدل كل error paths في الـ driver بـ `pinctrl_utils_free_map`:

```c
err_exit:
    /* GOOD: بتحرر الـ configs arrays الجوه قبل الـ map array */
    pinctrl_utils_free_map(pctldev, map, num_maps);
    return ret;
```

وتأكد إن الـ `dt_free_map` callback في الـ `pinctrl_ops` بتستخدم نفس الدالة:

```c
static void h616_dt_free_map(struct pinctrl_dev *pctldev,
                             struct pinctrl_map *map,
                             unsigned int num_maps)
{
    pinctrl_utils_free_map(pctldev, map, num_maps);
}
```

#### الدرس المستفاد
**أي `pinctrl_map` entry من نوع `CONFIGS_PIN` أو `CONFIGS_GROUP` عنده nested allocation داخلية.** الـ `pinctrl_utils_free_map` هي الطريقة الوحيدة الصح لتحرير الـ map لأنها بتعرف تمشي على الـ types وتحرر الـ `configs` array الجوه.

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — SPI مش شغال بسبب `WARN_ON` في `add_map_mux`

#### العنوان
**SPI2 لا يستجيب على STM32MP1 بسبب logic error في `reserved_maps` tracking**

#### السياق
بتعمل IoT sensor board على **STM32MP157**. الـ board عنده SPI2 متصل بـ accelerometer. الـ kernel بيبدأ، الـ SPI driver بيعمل probe، لكن الـ accelerometer مش بيرد وكل read بترجع `-ENODEV`.

#### المشكلة
الـ dmesg بيظهر:

```
WARNING: CPU: 0 PID: 1 at drivers/pinctrl/pinctrl-utils.c:50
pinctrl_utils_add_map_mux+0x28/0x60
```

ده الـ `WARN_ON` في السطر 50:

```c
int pinctrl_utils_add_map_mux(...)
{
    if (WARN_ON(*num_maps == *reserved_maps))
        return -ENOSPC;
    /* ... */
}
```

يعني الـ driver حاول يضيف map entry لكن مفيش space reserved.

#### التحليل
الـ driver بتاع STM32MP1 عنده bug في حساب الـ `reserve`:

```c
static int stm32_dt_node_to_map(struct pinctrl_dev *pctldev,
                                struct device_node *np, ...)
{
    /* خاطئ: بيحسب فقط عدد الـ config properties */
    ret = pinctrl_utils_reserve_map(pctldev, &map,
                &reserved_maps, &num_maps,
                num_configs);  /* ← ناسي إن في mux entry كمان! */

    /* بيضيف mux entry */
    ret = pinctrl_utils_add_map_mux(pctldev, &map,
                &reserved_maps, &num_maps, group, func);
    /* WARN_ON هنا! لأن num_maps == reserved_maps */

    /* بيضيف configs entry */
    ret = pinctrl_utils_add_map_configs(...);
}
```

لما الـ DT node عنده config واحد فقط:
- `reserve = num_configs = 1`
- بعد `add_map_mux`: `num_maps = 1`, `reserved_maps = 1`
- لما بيحاول `add_map_configs`: `num_maps == reserved_maps` → `WARN_ON`

الـ `add_map_mux` بترجع `-ENOSPC` فالـ map للـ SPI2 group مش بتتكتب كامل، وبالتالي الـ SPI2 مش بيحصله pinmux صح.

#### الحل
حساب الـ `reserve` صح: كل pin group محتاج **1 mux entry + عدد configs entries**:

```c
/* كل group: 1 للـ mux + 1 للـ configs (لو موجودة) */
unsigned int maps_per_pin = 1; /* للـ mux دايماً */
if (has_configs)
    maps_per_pin += 1;

ret = pinctrl_utils_reserve_map(pctldev, &map,
            &reserved_maps, &num_maps,
            num_groups * maps_per_pin);
```

أو الأبسط، ابعت reserve بـ 2 per group دايماً:

```c
ret = pinctrl_utils_reserve_map(pctldev, &map,
            &reserved_maps, &num_maps,
            num_groups * 2);
```

الـ `reserve_map` مش بتضيف entries فارغة، هي بس بتضمن إن في space كافي.

#### التحقق بعد الإصلاح

```bash
# تحقق إن الـ SPI2 group اتعمله pinmux صح
cat /sys/kernel/debug/pinctrl/soc:pin-controller/pinmux-groups | grep spi2

# تحقق إن الـ configs اتطبقت
cat /sys/kernel/debug/pinctrl/soc:pin-controller/pinconf-groups | grep spi2
```

#### الدرس المستفاد
**الـ `WARN_ON(*num_maps == *reserved_maps)` في `add_map_mux` و `add_map_configs` هو safety check — مش error handling.** لما تشوفه، معناه إن الـ `reserve` الأول كان ناقص. احسب كل أنواع الـ entries (mux + configs) قبل أول `reserve_map` call.

---

### السيناريو 4: Automotive ECU على i.MX8MP — CAN bus silent بسبب غلط في `pinctrl_map_type`

#### العنوان
**CAN1 على i.MX8MP صامت تماماً بعد إضافة low-power state في الـ DT**

#### السياق
بتعمل automotive ECU على **NXP i.MX8MP**. الـ ECU بيستخدم CAN1 للتواصل مع الـ vehicle network. أضفت **sleep state** للـ pinctrl في الـ DT عشان توفر power لما الـ ECU يدخل low-power mode. بعد التغيير، الـ CAN1 مش بيشتغل حتى في الـ normal state.

#### المشكلة
الـ CAN controller بيعمل probe بنجاح، لكن مفيش frames بتتبعت أو بتتستقبل. لا error في الـ dmesg.

#### التحليل
الـ DT الجديد:

```dts
can1_sleep_pins: can1-sleep-pins {
    fsl,pins = <
        MX8MP_IOMUXC_SAI5_RXD3__GPIO3_IO16   0x100  /* CAN_TX as GPIO */
        MX8MP_IOMUXC_SAI5_MCLK__GPIO3_IO25   0x100  /* CAN_RX as GPIO */
    >;
};
```

الـ driver بتاعك في `dt_node_to_map` بيضيف الـ sleep state configs:

```c
ret = pinctrl_utils_add_map_configs(pctldev, &map,
            &reserved_maps, &num_maps,
            group_name, configs, num_configs,
            PIN_MAP_TYPE_CONFIGS_GROUP);  /* ← صح */
```

لكن المشكلة في الـ mux للـ sleep state — الـ driver بيضيف mux entry بيشير لـ function غلط:

```c
/* BUG: بيضيف mux entry للـ sleep state بنفس function الـ default */
ret = pinctrl_utils_add_map_mux(pctldev, &map,
            &reserved_maps, &num_maps,
            group_name,
            "can1");  /* ← المفروض يكون "gpio" للـ sleep state */
```

لما الـ kernel يقرأ الـ map، بيلاقي إن الـ sleep state group اسمه `can1_sleep` لكن الـ function المطلوب `can1`. الـ pinctrl subsystem بيقرر إن الـ default state والـ sleep state بيستخدموا نفس الـ mux، فبيفضل الـ pin على الـ CAN function حتى لو المفروض يرجع.

الـ `pinctrl_map` entry المتأثرة:

```c
map[2].type = PIN_MAP_TYPE_MUX_GROUP;
map[2].data.mux.group    = "can1_sleep_pins";
map[2].data.mux.function = "can1";  /* ← غلط، المفروض "gpio" */
```

#### الحل
صحح الـ function name في parsing الـ sleep state:

```c
/* في الـ dt_node_to_map، امشي على الـ state name */
if (strcmp(state_name, "sleep") == 0 &&
    of_property_read_bool(np, "gpio-pins")) {
    func_name = "gpio";
} else {
    func_name = of_find_function_name(pctldev, np);
}

ret = pinctrl_utils_add_map_mux(pctldev, &map,
            &reserved_maps, &num_maps,
            group_name, func_name);
```

أو الأصح: خلي الـ DT يحدد الـ function صراحة وأوعى تخمن.

للتحقق:

```bash
# شوف الـ mapping الفعلي للـ can1
cat /sys/kernel/debug/pinctrl/30890000.can/pinmux-functions

# شوف الـ active state دلوقتي
cat /sys/kernel/debug/pinctrl/30890000.can/pinctrl-handles
```

#### الدرس المستفاد
**الـ `pinctrl_utils_add_map_mux` بتأخذ `function` كـ string — والـ kernel مش بيتحقق منه وقت `add_map_mux`, بيتحقق منه بس وقت `pinctrl_select_state`.** لما بتضيف multiple states، كل state محتاجة الـ function الصح الخاص بيها.

---

### السيناريو 5: Custom Board Bring-up على AM62x — I2C لا يظهر في الـ bus scan بسبب `pinctrl_utils_add_config` leak

#### العنوان
**I2C2 غائب من `/dev/i2c-*` على TI AM62x بسبب incomplete config building**

#### السياق
بتعمل bring-up لـ custom board على **Texas Instruments AM62x** (BeagleBone-class SoC). الـ board عنده I2C2 متصل بـ EEPROM وـ temperature sensor. الـ kernel بيبدأ لكن `i2c-detect -l` مش بتظهر الـ I2C2 bus.

#### المشكلة
الـ dmesg:

```
[    2.110] pinctrl-single 4084000.pinctrl: krealloc(configs) failed
[    2.111] 20020000.i2c: probe deferred
```

الـ I2C2 controller بيستنى الـ pinctrl ينجح عشان يكمل probe.

#### التحليل
الـ custom pinctrl driver بيبني الـ configs dynamically باستخدام `pinctrl_utils_add_config`:

```c
static int am62x_parse_pin_configs(struct pinctrl_dev *pctldev,
                                   struct device_node *np,
                                   unsigned long **configs,
                                   unsigned int *num_configs)
{
    unsigned long config;

    /* بيضيف pull-up config */
    config = AM62X_PIN_CONFIG_PULL_UP;
    ret = pinctrl_utils_add_config(pctldev, configs,
                num_configs, config);
    if (ret)
        goto err;

    /* بيضيف drive strength config */
    config = AM62X_PIN_CONFIG_DS_4MA;
    ret = pinctrl_utils_add_config(pctldev, configs,
                num_configs, config);
    if (ret)
        goto err;

    return 0;

err:
    /* BUG: لو فشل الـ add الثاني، الـ array الأول لسه allocated */
    return ret;  /* ← مش بيعمل kfree للـ *configs! */
}
```

الـ `pinctrl_utils_add_config` بتعمل `krealloc` في كل مرة:

```c
new_configs = krealloc(*configs, sizeof(*new_configs) * new_num,
                       GFP_KERNEL);
if (!new_configs) {
    dev_err(pctldev->dev, "krealloc(configs) failed\n");
    return -ENOMEM;  /* الـ *configs القديمة لسه valid! */
}
```

لما فشل الـ `krealloc` للـ config التانية، الـ `*configs` pointer لسه بيشاور على الـ array الأولى المخصصة. الـ caller بيمسحها بـ error path لكن من غير `kfree`:

```c
/* في الـ dt_node_to_map */
ret = am62x_parse_pin_configs(pctldev, np, &configs, &num_configs);
if (ret) {
    /* BUG: configs array مش بتتحرر */
    pinctrl_utils_free_map(pctldev, map, num_maps);
    return ret;
}
```

#### الحل
الخطأين اتنين محتاجين إصلاح:

**أولاً: في `am62x_parse_pin_configs`**

```c
err:
    /* حرر الـ partial configs array */
    kfree(*configs);
    *configs = NULL;
    *num_configs = 0;
    return ret;
```

**ثانياً: في `dt_node_to_map` بعد استخدام `add_map_configs`**

بعد ما `pinctrl_utils_add_map_configs` تنجح، هي بتعمل `kmemdup_array` للـ configs:

```c
dup_configs = kmemdup_array(configs, num_configs,
                sizeof(*dup_configs), GFP_KERNEL);
```

يعني الـ original `configs` array اللي عملتها محتاج يتحرر بعد الـ `add_map_configs` call:

```c
ret = pinctrl_utils_add_map_configs(pctldev, &map,
            &reserved_maps, &num_maps,
            group, configs, num_configs,
            PIN_MAP_TYPE_CONFIGS_GROUP);

/* الـ add_map_configs عمل copy — حرر الـ original */
kfree(configs);
configs = NULL;
num_configs = 0;

if (ret)
    goto err;
```

#### التحقق

```bash
# بعد الإصلاح والـ boot
i2cdetect -l
# المفروض يظهر: i2c-2    i2c    20020000.i2c    I2C adapter

# تحقق من الـ pinctrl state
cat /sys/kernel/debug/pinctrl/4084000.pinctrl/pinconf-groups | grep i2c2
```

#### الدرس المستفاد
**الـ `pinctrl_utils_add_config` بتملك الـ `*configs` pointer وبتعمله `krealloc` — لو فشلت، الـ pointer القديم لسه valid ومحتاج `kfree` من الـ caller.** كمان، بعد ما تعدي الـ configs لـ `pinctrl_utils_add_map_configs`، هي بتعمل copy جوهها (`kmemdup_array`) — يعني أنت مسؤول تحرر الـ original buffer.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| [`Documentation/driver-api/pinctl.rst`](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html) | التوثيق الرسمي الكامل لـ pinctrl subsystem — يشرح الـ pin registration وال pin groups والـ pin config |
| [`Documentation/pinctrl.txt`](https://www.kernel.org/doc/Documentation/pinctrl.txt) | النسخة الأصلية من التوثيق (kernel قديم) — مرجع مهم لفهم التصميم الأولي |
| `Documentation/devicetree/bindings/pinctrl/` | الـ DT bindings لكل SoC — أمثلة عملية على تعريف الـ pin groups في device tree |

---

### مقالات LWN.net

دي أهم المقالات اللي غطت تطوير الـ pinctrl subsystem من الأساس:

#### المقالات التأسيسية

- **[drivers: create a pin control subsystem](https://lwn.net/Articles/463335/)**
  الـ patch series الأولى من Linus Walleij اللي أنشأت الـ subsystem — لازم تقرأها عشان تفهم الـ design decisions الأصلية.

- **[pin controller subsystem v7](https://lwn.net/Articles/459190/)**
  النسخة السابعة من الـ patch series — توضح تطور الـ design قبل الـ merge.

- **[drivers: create a pin control subsystem v8](https://lwn.net/Articles/460768/)**
  النسخة الثامنة — ما قبل الـ merge مباشرة.

- **[drivers: create a pinmux subsystem v2](https://lwn.net/Articles/442315/)**
  الجذور الأولى للـ subsystem لما كانت اسمها pinmux بس — تاريخ مهم.

#### الـ pin config Interface

- **[The pin control subsystem](https://lwn.net/Articles/468759/)**
  مقالة تقنية تشرح الفكرة الكاملة وراء الـ subsystem — مرجع ممتاز للمبتدئين.

- **[pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/)**
  إضافة الـ generic pin config (biasing, driving modes) — الـ interface اللي بتستخدمه `pinctrl_utils_add_map_configs`.

- **[pinctrl: add a pin config interface](https://lwn.net/Articles/471826/)**
  تفاصيل أعمق للـ pin config API.

- **[pinctrl: add a generic pin config interface (v2)](https://lwn.net/Articles/467269/)**
  نسخة سابقة توضح التطور.

#### الـ Power Management States

- **[drivers: pinctrl sleep and idle states in the core](https://lwn.net/Articles/552972/)**
  إضافة الـ sleep/idle states للـ pinctrl core — مهمة لفهم إزاي الـ pin maps بتتغير مع power management.

- **[drivers/pinctrl: Add the concept of an "init" state](https://lwn.net/Articles/615322/)**
  الـ "init" state اللي بتتضبط قبل `probe()` — فاهم الـ pin maps بشكل أعمق.

#### الـ Documentation

- **[Documentation/pinctrl.txt (LWN mirror)](https://lwn.net/Articles/465077/)**
  مرجع سريع للـ documentation النصي الرسمي.

---

### الـ Kernel Commits المهمة

#### نشأة `pinctrl-utils.c`

الملف اتكتب في 2013 بواسطة **Laxman Dewangan** من NVIDIA عشان يحل مشكلة تكرار الكود في drivers مختلفة زي Tegra. الـ commit message الأصلي كان عن إضافة utility functions مشتركة بدل ما كل driver يعمل نسخته الخاصة من بناء الـ `pinctrl_map` arrays.

#### تحديث `krealloc_array`

- **[pinctrl: use krealloc_array()](https://lore.kernel.org/mm-commits/20201215030408.46a0I49Em%25akpm@linux-foundation.org/)**
  الـ commit اللي استبدل `krealloc()` بـ `krealloc_array()` في `pinctrl_utils_reserve_map` — تحسين أمان الذاكرة ضد integer overflow.

---

### نقاشات الـ Mailing List

- **[PATCH V2 0/3: pinctrl: add pincontrol driver for palmas device](https://linux.kernel.narkive.com/zzzi2I3f/patch-v2-0-3-pinctrl-add-pincontrol-driver-for-palmas-device)**
  مثال عملي على driver استخدم الـ `pinctrl-utils` API من الأول.

- **[LKML: pinctrl: add generic functions + pins mapper](https://lkml.org/lkml/2026/1/19/800)**
  نقاش حديث عن إضافة generic mapping functions للـ subsystem.

- **[pinctrl: Add generic pinctrl-simple driver for omap2+](https://lwn.net/Articles/496075/)**
  مثال على driver بيستخدم `pinctrl_utils_add_map_mux` في device tree parsing.

- **[i2c: Add generic I2C multiplexer using pinctrl API](https://lwn.net/Articles/498071/)**
  حالة استخدام مثيرة للاهتمام — الـ pinctrl بيتحكم في الـ I2C bus multiplexing.

---

### مواقع المجتمع

#### KernelNewbies.org

الصفحات دي بتوثق التغييرات اللي اتعملت على الـ pinctrl في كل إصدار:

- **[Linux_6.11 - Kernel Newbies](https://kernelnewbies.org/Linux_6.11)** — دعم i.MX91 وi.MX95 وNuvoton MA35D1
- **[Linux_6.13 - Kernel Newbies](https://kernelnewbies.org/Linux_6.13)** — Versal platform وCanaan K230 وExynos990
- **[Linux_6.17 - Kernel Newbies](https://kernelnewbies.org/Linux_6.17)** — Eswin EIC7700 وRaspberry Pi RP1 pinmux
- **[Linux_6.8 - Kernel Newbies](https://kernelnewbies.org/Linux_6.8)** — X1E80100 وQualcomm SM8650 TLMM

#### eLinux.org

- **[Pin Control and GPIO update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf)**
  عرض تقديمي من مؤتمر Embedded Linux — يشرح العلاقة بين الـ pinctrl وGPIO subsystems.

- **[EBC Device Trees](https://elinux.org/EBC_Device_Trees)**
  أمثلة عملية على `pinctrl-single,pins` في device tree — مفيد للمبتدئين.

- **[Tests: i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl)**
  اختبارات الـ i2c-demux-pinctrl driver — يوضح سيناريو استخدام متقدم للـ pin maps.

---

### مقالات تقنية خارجية

- **[Linux device driver development: The pin control subsystem — Embedded.com](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/)**
  مقالة شاملة من كتاب "Linux Device Drivers Development" بتشرح كل الـ API بأمثلة عملية.

- **[Pin Control and GPIO Subsystem — O'Reilly](https://www.oreilly.com/library/view/linux-device-drivers/9781785280009/a71178ac-47f2-49cb-a5ef-655b814e7699.xhtml)**
  فصل كامل مخصص للـ pinctrl وGPIO من كتاب "Linux Device Drivers Development" بقلم John Madieu.

- **[Pinctrl overview — STM32MPU Wiki](https://wiki.st.com/stm32mpu/wiki/Pinctrl_overview)**
  مثال عملي حقيقي على implementation كاملة للـ pinctrl على SoC — ST Microelectronics.

---

### كتب موصى بها

| الكتاب | المؤلف | الأهمية |
|--------|--------|---------|
| **Linux Device Drivers (LDD3)** | Corbet, Rubini, Kroah-Hartman | الأساس — فصول الـ hardware access والـ memory management مهمة لفهم `krealloc`/`kfree` |
| **Linux Kernel Development (3rd Ed.)** | Robert Love | فصول الـ memory allocation وkernel data structures — يشرح `GFP_KERNEL` وlocking |
| **Linux Device Drivers Development** | John Madieu | **الأهم** — فصل كامل عن pinctrl مع أمثلة device tree حديثة |
| **Embedded Linux Primer (2nd Ed.)** | Christopher Hallinan | فصل الـ Board Support Package — يشرح إزاي الـ pinctrl بيتكامل مع الـ BSP |
| **Mastering Embedded Linux Programming** | Frank Vasquez & Chris Simmonds | الإصدار الثالث — يشمل device tree وpinctrl في context الـ Yocto/Buildroot |

---

### مسارات البحث

لو حابب تعمق أكتر، استخدم الـ search terms دي:

```
# البحث في LWN
site:lwn.net "pinctrl_map" OR "pin control" subsystem

# البحث في lore.kernel.org (mailing list رسمي)
site:lore.kernel.org pinctrl-utils

# البحث في git log
git log --all --oneline -- drivers/pinctrl/pinctrl-utils.c

# البحث في الـ kernel source
git log v3.9..v3.10 --oneline -- drivers/pinctrl/

# Elixir Cross-referencer
https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl/pinctrl-utils.c
```

#### الـ Elixir Cross-Referencer

**[elixir.bootlin.com — pinctrl-utils.c](https://elixir.bootlin.com/linux/latest/source/drivers/pinctrl/pinctrl-utils.c)**
أداة لا غنى عنها — بتخليك تشوف كل الـ callers لكل function في `pinctrl-utils.c` عبر كل تاريخ الـ kernel.

---

### ملخص المراجع الأهم

```
للمبتدئين:
  1. LWN: "The pin control subsystem" → https://lwn.net/Articles/468759/
  2. Embedded.com article → practical examples
  3. Documentation/driver-api/pinctl.rst → official reference

للمتقدمين:
  1. LWN patch series v1-v8 → design decisions
  2. lore.kernel.org → mailing list history
  3. git log -- drivers/pinctrl/ → commit archaeology

للـ driver writers:
  1. pinctrl-utils.h header → API surface
  2. drivers/pinctrl/pinctrl-tegra.c → reference implementation
  3. Documentation/devicetree/bindings/pinctrl/ → DT bindings
```
## Phase 8: Writing simple module

### الفكرة

هنعمل **kprobe** على الفانكشن `pinctrl_utils_add_map_mux` — دي فانكشن exported من `pinctrl-utils.c` بتتسمى من أي pinctrl driver لما بيسجّل MUX mapping جديد.
ده بيحصل كتير وقت boot لما الـ pinctrl drivers بتـparse الـ device tree، فالـ kprobe هيعدّي على events حقيقية وهيطبع اسم الـ group والـ function اللي اتسجّلوا.

---

### الـ Module كامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_pinmux.c
 * Hook pinctrl_utils_add_map_mux() to log every MUX mapping registration.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe */
#include <linux/printk.h>      /* pr_info */

/*
 * pinctrl_utils_add_map_mux signature:
 *   int pinctrl_utils_add_map_mux(struct pinctrl_dev *pctldev,
 *                                  struct pinctrl_map **map,
 *                                  unsigned int *reserved_maps,
 *                                  unsigned int *num_maps,
 *                                  const char *group,
 *                                  const char *function);
 *
 * We read 'group' and 'function' from pt_regs — architecture-specific:
 *   x86-64: rdi=pctldev, rsi=map, rdx=reserved_maps, rcx=num_maps,
 *           r8=group, r9=function
 *   arm64:  x0=pctldev, x1=map, x2=reserved_maps, x3=num_maps,
 *           x4=group, x5=function
 */

static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    const char *group;
    const char *function;

#if defined(CONFIG_X86_64)
    /* 5th arg → r8, 6th arg → r9 */
    group    = (const char *)regs->r8;
    function = (const char *)regs->r9;
#elif defined(CONFIG_ARM64)
    /* 5th arg → x4, 6th arg → x5 */
    group    = (const char *)regs->regs[4];
    function = (const char *)regs->regs[5];
#else
    /* fallback: just log the hit without args */
    group    = "<unknown>";
    function = "<unknown>";
#endif

    /* Print the group/function being registered as a MUX map entry */
    pr_info("pinmux_kprobe: add_map_mux → group='%s'  function='%s'\n",
            group ? group : "(null)",
            function ? function : "(null)");

    return 0; /* 0 = continue normal execution */
}

/* kprobe descriptor — we only use pre_handler (before the function runs) */
static struct kprobe kp = {
    .symbol_name = "pinctrl_utils_add_map_mux",
    .pre_handler = handler_pre,
};

static int __init pinmux_kprobe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("pinmux_kprobe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("pinmux_kprobe: hooked pinctrl_utils_add_map_mux @ %p\n",
            kp.addr);
    return 0;
}

static void __exit pinmux_kprobe_exit(void)
{
    /* Must unregister before module unloads — otherwise kernel will call
     * handler_pre after the code page is freed → instant panic */
    unregister_kprobe(&kp);
    pr_info("pinmux_kprobe: unhooked\n");
}

module_init(pinmux_kprobe_init);
module_exit(pinmux_kprobe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Example Author");
MODULE_DESCRIPTION("kprobe on pinctrl_utils_add_map_mux to log MUX registrations");
```

---

### Makefile

```makefile
obj-m += kprobe_pinmux.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/module.h` | الـ macros الأساسية زي `module_init`, `MODULE_LICENSE` |
| `linux/kprobes.h` | **kprobe** API — `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/printk.h` | `pr_info` / `pr_err` للطباعة في kernel log |

**الـ** `linux/pinctrl/pinctrl.h` **مش محتاجينه** هنا لأننا مش بنستدعي الفانكشن مباشرة، بس بنـhook عليها من بره.

---

#### ليه `pinctrl_utils_add_map_mux`؟

**الـ** `pinctrl_utils_add_map_mux` هي أكتر فانكشن في الملف بتتسمى من كل الـ drivers لما بتـparse الـ device tree، فهي نقطة مراقبة ممتازة لشوف إيه الـ MUX configurations اللي بتتسجّل على الـ system.
**كمان** هي `EXPORT_SYMBOL_GPL` يعني اسمها موجود في الـ kallsyms وبيقدر الـ kprobe يلاقيها بـ `symbol_name`.

---

#### الـ `handler_pre` والـ `pt_regs`

**الـ** `pt_regs` هو struct بيحفظ حالة الـ CPU registers لحظة الـ probe hit — بنقرأ منه الـ arguments حسب الـ calling convention بتاع كل architecture.
**بنقرأ** الـ `group` و`function` اللي بتتحول لـ MUX entry عشان نطبعهم، وده بيكشف exactly إيه الـ pingroups اللي الـ driver بيسجّلها.

---

#### الـ `return 0` في `handler_pre`

**الـ** `0` بيقول للـ kprobe framework "تابع تنفيذ الفانكشن الأصلية normally".
**لو رجّعنا** قيمة تانية، الـ kernel ممكن يسكيب الفانكشن — ده مش حنا عايزينه هنا، بس بيُستخدم في الـ kretprobe بشكل مختلف.

---

#### الـ `module_exit` و `unregister_kprobe`

**لو** ما عملناش `unregister_kprobe` قبل ما الـ module يتـunload، الـ kernel هيفضل يـjump لـ `handler_pre` في memory page اتـfreed → **kernel panic** مباشرة.
**الـ** `unregister_kprobe` بتنتظر كمان إن مفيش CPU تانية شغالة في الـ handler دلوقتي قبل ما تـreturn — thread-safe تماماً.

---

### تشغيل وتتبع النتيجة

```bash
# Build
make

# Insert — الـ hook بيتسجّل
sudo insmod kprobe_pinmux.ko

# شوف اللي اتطبع (أغلبه بيحصل وقت boot، ممكن تـtrigger بـ bind/unbind)
sudo dmesg | grep pinmux_kprobe

# Remove
sudo rmmod kprobe_pinmux
sudo dmesg | tail -3
```

مثال على الـ output:

```
[    2.341] pinmux_kprobe: hooked pinctrl_utils_add_map_mux @ ffffffffc03a1120
[    2.412] pinmux_kprobe: add_map_mux → group='uart0_grp'  function='uart0'
[    2.413] pinmux_kprobe: add_map_mux → group='i2c1_grp'   function='i2c1'
[    2.415] pinmux_kprobe: add_map_mux → group='spi0_grp'   function='spi0'
```

كل سطر بيمثّل **pinmux entry** واحدة اتسجّلت، يعني قدرنا نشوف كل الـ mux decisions بتاعة الـ board من غير ما نعدّل سطر واحد في أي driver.
