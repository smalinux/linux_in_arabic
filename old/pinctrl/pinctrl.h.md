# شرح ملف `pinctrl.h` - Linux Kernel

## ما هو الـ Pin Control Subsystem؟

هو نظام فرعي في Linux kernel مسؤول عن إدارة الـ **pins** (أطراف الشرائح الإلكترونية) في المعالجات المدمجة، يتحكم في:

- **Pin Multiplexing (pinmux):** تحديد وظيفة كل pin (UART, SPI, GPIO...إلخ)
- **Pin Configuration:** ضبط خصائص الـ pin (pull-up/pull-down, drive strength...إلخ)

---

## الهياكل الأساسية (Core Structures)

**`struct pinctrl_pin_desc`** — يصف pin واحد:

- رقمه الفريد `number` في الـ global pin space
- اسمه `name`
- بيانات خاصة بالـ driver في `drv_data`

**`struct pingroup`** — مجموعة pins تعمل معاً (مثلاً pins الـ SPI كلها في group واحد)

**`struct pinfunction`** — وظيفة معينة (مثل `uart0`) وترتبط بـ groups معينة. لو كانت وظيفتها GPIO تُضاف flag الـ `PINFUNCTION_FLAG_GPIO`

**`struct pinctrl_gpio_range`** — يربط نطاق من الـ GPIO numbers بالـ pin controller المسؤول عنها

---

## الـ Operations (vtables)

**`struct pinctrl_ops`** — العمليات الأساسية التي يجب أن يُنفّذها الـ driver:

- `get_groups_count` / `get_group_name` / `get_group_pins` → للتعامل مع الـ groups
- `dt_node_to_map` / `dt_free_map` → لقراءة الـ **Device Tree** وتحويله إلى **mapping table**

---

## الـ Descriptor الرئيسي

**`struct pinctrl_desc`** — هو "بطاقة تعريف" الـ pin controller كاملاً، يحتوي على:

- مصفوفة كل الـ pins المدعومة
- مؤشرات للـ `pctlops`, `pmxops`, `confops` (الـ vtables الثلاث)
- خيار `link_consumers` لضمان ترتيب صحيح في الـ suspend/resume

---

## دورة حياة الـ Driver

```
pinctrl_register_and_init()  →  pinctrl_enable()  →  pinctrl_unregister()
```

أو النسخة الـ **managed**:

```
devm_pinctrl_register_and_init()  ← تُحرر تلقائياً مع الـ device
```

---

## الخلاصة

هذا الملف هو الـ **interface** الذي يتعامل معه مطور الـ BSP لكتابة driver لأي pin controller (مثل Rockchip، Allwinner، STM32...)، حيث يُسجّل الـ driver نفسه مع الـ subsystem عبر `pinctrl_desc` ويُنفّذ الـ ops المطلوبة.

# Linux Kernel `pinctrl.h` — شرح تفصيلي

---

## 1. ما المشكلة التي يحلّها هذا الـ Subsystem؟

في أي SoC (مثل RK3562 أو STM32MP1)، كل **physical pin** على الشريحة يمكن أن يؤدي أكثر من وظيفة واحدة. مثلاً: **PIN_A5** يمكن أن يكون:

- `UART2_TX`
- `SPI0_CLK`
- `GPIO3_A5`
- `PWM1`

مهمة الـ **Pin Control Subsystem** هي:

1. تسجيل كل الـ pins المتاحة
2. تعريف الـ **functions** و**groups** المرتبطة بها
3. السماح لأي driver بطلب تفعيل وظيفة معينة على مجموعة من الـ pins

---

## 2. الصورة الكبيرة — Big Picture

```
┌─────────────────────────────────────────────────────────────┐
│                     Linux Kernel                            │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │  UART    │   │   SPI    │   │   I2C    │   │  GPIO   │ │
│  │  Driver  │   │  Driver  │   │  Driver  │   │  Driver │ │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬────┘ │
│       │              │              │               │       │
│       └──────────────┴──────────────┴───────────────┘       │
│                              │                               │
│                   ┌──────────▼──────────┐                   │
│                   │  pinctrl subsystem  │  ← هذا الملف      │
│                   │  (pinctrl core)     │                   │
│                   └──────────┬──────────┘                   │
│                              │                               │
│                   ┌──────────▼──────────┐                   │
│                   │  pinctrl_desc       │  ← يسجّل نفسه     │
│                   │  (your BSP driver)  │                   │
│                   └──────────┬──────────┘                   │
│                              │                               │
│                   ┌──────────▼──────────┐                   │
│                   │   Physical SoC      │                   │
│                   │   Pins/Pads         │                   │
│                   └─────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. الـ Structs — شرح تفصيلي ومترابط

### 3.1 `struct pinctrl_pin_desc` — وحدة البناء الأساسية

هذا هو **أصغر وحدة** في النظام. كل pin في الـ SoC له سجل واحد من هذا النوع.

```c
struct pinctrl_pin_desc {
    unsigned int number;   // رقم فريد عالمي للـ pin
    const char  *name;     // اسم بشري مقروء مثل "GPIO3_A5"
    void        *drv_data; // بيانات خاصة بالـ driver (اختياري)
};
```

**مثال عملي لـ RK3562:**

```c
static const struct pinctrl_pin_desc rk3562_pins[] = {
    PINCTRL_PIN(0,   "GPIO0_A0"),
    PINCTRL_PIN(1,   "GPIO0_A1"),
    PINCTRL_PIN(2,   "GPIO0_A2"),
    // ... مئات من الـ pins
    PINCTRL_PIN(127, "GPIO3_D7"),
};
```

```
pinctrl_pin_desc Array:
┌─────┬─────────────┬──────────┐
│ [0] │ number: 0   │ "GPIO0_A0"│
├─────┼─────────────┼──────────┤
│ [1] │ number: 1   │ "GPIO0_A1"│
├─────┼─────────────┼──────────┤
│ [2] │ number: 2   │ "GPIO0_A2"│
├─────┼─────────────┼──────────┤
│ ... │ ...         │ ...      │
├─────┼─────────────┼──────────┤
│[127]│ number: 127 │"GPIO3_D7"│
└─────┴─────────────┴──────────┘
```

---

### 3.2 `struct pingroup` — تجميع الـ Pins في مجموعات

الـ **group** هو مجموعة pins تعمل معاً لأداء وظيفة واحدة. مثلاً SPI يحتاج 4 pins تعمل معاً.

```c
struct pingroup {
    const char         *name;   // اسم المجموعة مثل "spi0_grp"
    const unsigned int *pins;   // مصفوفة أرقام الـ pins
    size_t              npins;  // عدد الـ pins في المجموعة
};
```

**مثال:**

```c
static const unsigned int spi0_pins[]  = { 10, 11, 12, 13 };
static const unsigned int uart2_pins[] = { 20, 21 };
static const unsigned int i2c1_pins[]  = { 30, 31 };

static const struct pingroup rk3562_groups[] = {
    PINCTRL_PINGROUP("spi0_grp",  spi0_pins,  ARRAY_SIZE(spi0_pins)),
    PINCTRL_PINGROUP("uart2_grp", uart2_pins, ARRAY_SIZE(uart2_pins)),
    PINCTRL_PINGROUP("i2c1_grp",  i2c1_pins,  ARRAY_SIZE(i2c1_pins)),
};
```

```
pingroup "spi0_grp":
┌─────────────────────────────────────┐
│ name: "spi0_grp"                    │
│ npins: 4                            │
│ pins: ──────────────────────────┐   │
└─────────────────────────────────┼───┘
                                  ▼
                    ┌────┬────┬────┬────┐
                    │ 10 │ 11 │ 12 │ 13 │
                    └────┴────┴────┴────┘
                      ▲    ▲    ▲    ▲
                      │    │    │    │
                  CLK MOSI MISO  CS
             (كل رقم يشير إلى pinctrl_pin_desc في المصفوفة الرئيسية)
```

---

### 3.3 `struct pinfunction` — الوظيفة الكاملة

الـ **function** هي الوظيفة المنطقية (مثل `spi0`)، وترتبط بـ **group واحد أو أكثر** (لأن نفس الوظيفة قد تكون على pins مختلفة — مثل UART على GPIO bank مختلف).

```c
struct pinfunction {
    const char        *name;    // اسم الوظيفة مثل "spi0"
    const char *const *groups;  // مصفوفة أسماء الـ groups المدعومة
    size_t             ngroups; // عددها
    unsigned long      flags;   // PINFUNCTION_FLAG_GPIO إذا كانت GPIO
};
```

**مثال:**

```c
static const char *const spi0_groups[]  = { "spi0_grp" };
static const char *const uart2_groups[] = { "uart2_grp_a", "uart2_grp_b" };

static const struct pinfunction rk3562_functions[] = {
    PINCTRL_PINFUNCTION("spi0",  spi0_groups,  1),
    PINCTRL_PINFUNCTION("uart2", uart2_groups, 2), // وظيفة واحدة، مجموعتان!
    PINCTRL_GPIO_PINFUNCTION("gpio", gpio_groups, N),
};
```

```
Function "uart2" يدعم مجموعتين:
┌──────────────────────────────────────────────┐
│ pinfunction: "uart2"                         │
│ ngroups: 2                                   │
│ groups: ─────────────────────┐               │
└──────────────────────────────┼───────────────┘
                               ▼
                ┌──────────────┴──────────────┐
                ▼                             ▼
       "uart2_grp_a"                 "uart2_grp_b"
     ┌─────────────┐               ┌─────────────┐
     │ pins: 20,21 │               │ pins: 50,51 │
     │ (GPIO0 bank)│               │ (GPIO1 bank)│
     └─────────────┘               └─────────────┘
```

---

### 3.4 `struct pinctrl_gpio_range` — ربط GPIO بالـ Pin Controller

هذه الـ struct تربط نطاق من أرقام الـ **GPIO Linux namespace** بالـ pins الفعلية في الـ pin controller.

```c
struct pinctrl_gpio_range {
    struct list_head   node;      // linked list داخلي
    const char        *name;      // اسم الـ range
    unsigned int       id;        // معرّف
    unsigned int       base;      // أول رقم GPIO في Linux
    unsigned int       pin_base;  // أول رقم pin مقابل له
    unsigned int       npins;     // عدد الـ pins في الـ range
    unsigned int const*pins;      // أو مصفوفة صريحة بدلاً من pin_base
    struct gpio_chip  *gc;        // الـ gpio_chip المرتبط
};
```

```
GPIO Linux Namespace        Pin Controller Namespace
─────────────────────       ──────────────────────────
  gpio base=32         ──►   pin_base=0
  gpio 32              ──►   pin 0   "GPIO0_A0"
  gpio 33              ──►   pin 1   "GPIO0_A1"
  gpio 34              ──►   pin 2   "GPIO0_A2"
  ...                         ...
  gpio 32+31           ──►   pin 31  "GPIO0_D7"
  (npins = 32)
```

---

### 3.5 `struct pinctrl_ops` — العمليات الأساسية (vtable #1)

هذه الـ vtable هي "العقد" بين الـ core و driver الخاص بك. يجب أن تُنفّذها.

```c
struct pinctrl_ops {
    // كم عدد الـ groups المسجّلة؟
    int (*get_groups_count)(struct pinctrl_dev *pctldev);

    // ما اسم الـ group رقم selector؟
    const char *(*get_group_name)(struct pinctrl_dev *pctldev,
                                  unsigned int selector);

    // ما هي الـ pins في الـ group رقم selector؟
    int (*get_group_pins)(struct pinctrl_dev *pctldev,
                          unsigned int selector,
                          const unsigned int **pins,
                          unsigned int *num_pins);

    // طباعة معلومات pin في debugfs (اختياري)
    void (*pin_dbg_show)(struct pinctrl_dev *pctldev,
                         struct seq_file *s,
                         unsigned int offset);

    // تحويل Device Tree node إلى mapping table
    int (*dt_node_to_map)(struct pinctrl_dev *pctldev,
                          struct device_node *np_config,
                          struct pinctrl_map **map,
                          unsigned int *num_maps);

    // تحرير الـ mapping table
    void (*dt_free_map)(struct pinctrl_dev *pctldev,
                        struct pinctrl_map *map,
                        unsigned int num_maps);
};
```

**كيف يستخدمها الـ core:**

```
pinctrl core يريد معرفة pins الـ group "spi0_grp":

core                          your driver
 │                                │
 │── get_groups_count() ─────────►│ returns 10
 │                                │
 │── get_group_name(0) ──────────►│ returns "spi0_grp"
 │                                │
 │── get_group_pins(0) ──────────►│ returns [10,11,12,13], num=4
 │                                │
```

---

### 3.6 `struct pinctrl_desc` — الـ Descriptor الرئيسي (قلب النظام)

هذه الـ struct هي **كل شيء** عن الـ pin controller. تُسجّلها مرة واحدة عند الـ probe.

```c
struct pinctrl_desc {
    const char                    *name;        // اسم الـ controller
    const struct pinctrl_pin_desc *pins;        // مصفوفة كل الـ pins
    unsigned int                   npins;       // عددها
    const struct pinctrl_ops      *pctlops;     // vtable الأساسية
    const struct pinmux_ops       *pmxops;      // vtable الـ mux
    const struct pinconf_ops      *confops;     // vtable الـ config
    struct module                 *owner;       // THIS_MODULE
    // إذا كان CONFIG_GENERIC_PINCONF مفعّلاً:
    unsigned int                   num_custom_params;
    const struct pinconf_generic_params *custom_params;
    const struct pin_config_item        *custom_conf_items;
    bool                           link_consumers; // device link للـ suspend/resume
};
```

---

## 4. كيف تترابط كل الـ Structs معاً؟

```
┌─────────────────────────────────────────────────────────────────────┐
│                        pinctrl_desc                                 │
│  name: "rk3562-pinctrl"                                             │
│                                                                     │
│  pins ──────────────────────────────────────────────────────────┐   │
│  npins: 128                                                     │   │
│                                                                 │   │
│  pctlops ──────────────┐                                        │   │
│  pmxops  ──────────┐   │                                        │   │
│  confops ──────┐   │   │                                        │   │
└────────────────┼───┼───┼────────────────────────────────────────┘   │
                 │   │   │                                        │
                 │   │   ▼                                        ▼
                 │   │  pinctrl_ops              pinctrl_pin_desc[]
                 │   │  ├─ get_groups_count      ┌──────────────────┐
                 │   │  ├─ get_group_name        │[0]  "GPIO0_A0"   │
                 │   │  ├─ get_group_pins ──►    │[1]  "GPIO0_A1"   │
                 │   │  ├─ dt_node_to_map        │...               │
                 │   │  └─ dt_free_map           │[127]"GPIO3_D7"   │
                 │   │                           └──────────────────┘
                 │   ▼
                 │  pinmux_ops
                 │  ├─ get_functions_count
                 │  ├─ get_function_name ──► pinfunction[]
                 │  ├─ get_function_groups     ┌──────────────────┐
                 │  ├─ set_mux ──────────►     │[0] "spi0"        │
                 │  └─ gpio_set_direction      │    groups:[grp0] │
                 │                             │[1] "uart2"       │
                 │                             │    groups:[grp1] │
                 │                             └──────────────────┘
                 │                                      │
                 │                                      ▼
                 │                             pingroup[]
                 │                             ┌──────────────────┐
                 │                             │[0] "spi0_grp"    │
                 │                             │    pins:[10,11,  │
                 │                             │         12,13]   │
                 │                             │[1] "uart2_grp"   │
                 │                             │    pins:[20,21]  │
                 │                             └──────────────────┘
                 │
                 ▼
                pinconf_ops
                ├─ pin_config_get
                ├─ pin_config_set ──► يضبط pull-up/down, drive strength
                └─ pin_config_group_set
```

---

## 5. دورة حياة الـ Driver — Lifecycle

```
SoC Boot
   │
   ▼
platform_driver probe() يُستدعى
   │
   ├──► بناء pinctrl_pin_desc[]  (كل pins الـ SoC)
   ├──► بناء pingroup[]          (groups المنطقية)
   ├──► بناء pinfunction[]       (الوظائف المتاحة)
   ├──► تعريف pinctrl_ops        (get_groups, dt_node_to_map...)
   ├──► تعريف pinmux_ops         (set_mux...)
   ├──► تعريف pinconf_ops        (pin_config_set...)
   │
   ├──► تعبئة pinctrl_desc بكل ما سبق
   │
   ├──► استدعاء pinctrl_register_and_init()
   │         │
   │         └──► الـ core يُسجّل الـ controller
   │              ويُنشئ debugfs entries
   │              ويقرأ الـ Device Tree
   │
   └──► استدعاء pinctrl_enable()
             │
             └──► الـ controller جاهز لاستقبال الطلبات

   │
   ▼
UART driver يطلب "uart2" function:
   │
   ├──► pinctrl_get() + pinctrl_lookup_state() + pinctrl_select_state()
   │
   └──► الـ core يستدعي set_mux() في pinmux_ops
            └──► driver يكتب في registers الـ SoC
                      └──► PIN_20 و PIN_21 يصبحان UART2_TX و UART2_RX
```

---

## 6. Device Tree → Driver — كيف يُترجم الـ DT؟

```dts
// في الـ Device Tree:
&pinctrl {
    uart2_pins_default: uart2-pins {
        pinmux = <PINMUX_GPIO20_FUNC_UART2_TX>,
                 <PINMUX_GPIO21_FUNC_UART2_RX>;
        bias-disable;
        drive-strength = <4>;
    };
};
```

```
dt_node_to_map() يستقبل هذا الـ node ويُنشئ:

pinctrl_map[] = [
  { type: PIN_MAP_TYPE_MUX_GROUP,
    data.mux.function = "uart2",
    data.mux.group    = "uart2_grp" },

  { type: PIN_MAP_TYPE_CONFIGS_PIN,
    data.configs.pin_name = "GPIO0_A4",
    data.configs.configs  = [bias-disable, drive-4mA] },
]
```

---

## 7. الفرق بين الـ Registration Functions

```
pinctrl_register()               ← قديم، لا تستخدمه
      │
      └── يُرجع pinctrl_dev* أو NULL عند الخطأ

pinctrl_register_and_init()      ← الحديث المفضّل
      │                             يُرجع error code
      └── + pinctrl_enable()    ← خطوتان منفصلتان للمرونة
                                    (يمكن تأخير الـ enable)

devm_pinctrl_register_and_init() ← الأفضل للـ platform drivers
      │
      └── تُحرر تلقائياً عند device_remove()
          بدون الحاجة لـ pinctrl_unregister() يدوياً
```

---

## 8. ملخص العلاقات — Summary Diagram

```
                      ┌─────────────────────┐
                      │    pinctrl_desc      │  ← تُسجّل مرة واحدة
                      │   (البطاقة الكاملة)  │    عند probe()
                      └──────────┬──────────┘
                                 │ يحتوي على
              ┌──────────────────┼───────────────────┐
              │                  │                   │
              ▼                  ▼                   ▼
    pinctrl_pin_desc[]      pinctrl_ops          pinmux_ops
    ─────────────────       ───────────          ──────────
    كل pins الـ SoC         get_groups           get_functions
    (المصدر الأول)          dt_node_to_map       set_mux
          │                      │               gpio_set_direction
          │               يُعيد  │
          ▼               ───────▼────────
       pin numbers         pingroup[]         pinfunction[]
       ───────────         ──────────         ─────────────
       0..127              "spi0_grp"         "spi0"
                           "uart2_grp"        "uart2"
                           "i2c1_grp"         "gpio"
                                │                  │
                                └──────────────────┘
                                  يرتبطان عبر
                                  أسماء الـ groups
                                  (string matching)

    pinctrl_gpio_range[]
    ────────────────────
    GPIO 32..63 → pins 0..31
    GPIO 64..95 → pins 32..63
    يُضاف عبر pinctrl_add_gpio_range()
```

---

## 9. نقاط مهمة للـ BSP Developer

| الموضوع            | التفاصيل                                             |
| ------------------ | ---------------------------------------------------- |
| **إلزامي**         | `pctlops` مطلوب دائماً                               |
| **اختياري**        | `pmxops` فقط لو تدعم pinmux                          |
| **اختياري**        | `confops` فقط لو تدعم pin config                     |
| **dt_node_to_map** | إذا كنت تدعم Device Tree (وأنت تدعمه دائماً في 2024) |
| **link_consumers** | فعّله لو عندك مشاكل في ترتيب الـ suspend/resume      |
| **devm_**          | استخدم دائماً النسخة الـ managed                     |