## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Pin Control Subsystem؟

تخيل إن عندك لوحة إلكترونية فيها معالج (CPU/SoC). الـ SoC ده فيه مئات الـ **pins** (أرجل) على الجنبين. كل pin ممكن يشتغل بأكتر من وظيفة — نفس الـ pin ممكن يبقى:
- خط UART (Serial communication)
- خط I2C (للتواصل مع الـ sensors)
- خط GPIO عادي (تشغيل LED مثلاً)
- خط SPI (للتواصل مع الـ Flash)

الـ **Pin Control Subsystem** هو الجهاز المسؤول عن إدارة قرار "مين شغّال على الـ pin ده؟ وبأي إعداد؟"

---

### المشكلة اللي بتحلّها

تصوّر إن عندك راوتر مبني على SoC. الـ SoC فيه 120 pin. كل chip له مصنّعه الخاص وطريقته في التحكم في الـ pins. من غير نظام موحّد، كل driver كان هيتعامل مع الـ hardware بشكل مختلف، وكان هيبقى فيه تعارض — مين ياخد الـ pin ده؟ هل شغّال UART ولّا GPIO؟ بيحصل تخبيط.

الـ **pinctrl subsystem** جاء عشان:
1. يوفّر **واجهة موحّدة** للتحكم في الـ pins على كل الـ hardware.
2. يمنع تعارض الـ pins بين الـ drivers.
3. يربط الـ GPIO controllers بالـ pin controllers.
4. يدعم **states** — مثلاً: pin في وضع "default" (شغال) أو "sleep" (موفر للطاقة).

---

### القصة كاملة (ELI5)

تخيّل مبنى كبير فيه غرف، وكل غرفة عندها باب بمفتاح واحد بس. أكتر من جهاز (UART, I2C, GPIO) عايزين يدخلوا نفس الغرفة (نفس الـ pin). الـ pinctrl هو الـ **حارس** اللي بيقرر مين يدخل امتى، وبيضمن إن محدش يدخل على بعضه.

لما device بيتشغّل، بيقول للـ pinctrl: "أنا عايز الـ pins دي في حالة default". الـ pinctrl بيراجع جدول الـ **mapping table** (اللي بييجي من Device Tree أو static config)، وبيشوف: الـ pin ده هيشتغل بإيه function، وبأي configuration (pull-up؟ drive strength؟).

---

### دور الملف `core.h`

الملف ده هو **الـ private header** للـ pinctrl core. مش للـ hardware drivers ولا للـ consumers، ده للـ core نفسه جوّاه. بيعرّف الـ data structures الداخلية اللي الـ core بيستخدمها لإدارة كل حاجة:

| Structure | الوظيفة |
|-----------|---------|
| `pinctrl_dev` | يمثّل جهاز pin controller مسجّل في النظام. كل SoC بيسجّل `pinctrl_dev` واحد أو أكتر. |
| `pinctrl` | الـ handle الخاص بكل device تعاملت مع pinctrl (ناتج `pinctrl_get()`). |
| `pinctrl_state` | حالة معيّنة (مثل "default" أو "sleep") لجهاز معيّن. |
| `pinctrl_setting` | إعداد واحد داخل state — إما mux (وظيفة) أو config (إعداد كهربي). |
| `pin_desc` | وصف كل pin منفرد — اسمه، مين بيستخدمه، الـ mux المختار. |
| `pinctrl_maps` | جزء من جدول الـ mapping (ربط device بـ pins بـ functions). |
| `group_desc` | وصف مجموعة pins (عند تفعيل `CONFIG_GENERIC_PINCTRL_GROUPS`). |

---

### العلاقة بين الـ structures

```
pinctrl_dev  ──owns──►  pin_desc[] (via radix tree)
     │                  group_desc[] (via radix tree, optional)
     │                  function_tree (via radix tree, optional)
     └──► gpio_ranges (list)

pinctrl (per consumer device)
     └──► pinctrl_state[]
               └──► pinctrl_setting[]
                         ├── type: MUX  → pinctrl_setting_mux {group, func}
                         └── type: CONFIG → pinctrl_setting_configs {pin/group, configs[]}

pinctrl_maps (global list)
     └──► pinctrl_map[] → connects: dev_name + state_name → pctldev + group + function
```

---

### الـ States والـ Hogging

**الـ hog** معناه إن الـ pin controller نفسه بيحجز بعض الـ pins لنفسه عند التسجيل — يعني قبل ما أي device تاني يشتغل. ده بيحصل لو الـ `dev_name` في الـ mapping هو نفسه اسم الـ pin controller.

```c
struct pinctrl_dev {
    struct pinctrl *p;             // نتيجة pinctrl_get() للـ controller نفسه
    struct pinctrl_state *hog_default;  // حالة "default" المحجوزة
    struct pinctrl_state *hog_sleep;    // حالة "sleep" المحجوزة
};
```

---

### الملفات المكوّنة للـ Subsystem

#### الـ Core (القلب):
- `drivers/pinctrl/core.c` — التطبيق الفعلي للـ core
- `drivers/pinctrl/core.h` — **الملف ده** — الـ private structs للـ core
- `drivers/pinctrl/pinmux.c` / `pinmux.h` — إدارة الـ mux functions
- `drivers/pinctrl/pinconf.c` / `pinconf.h` — إدارة الـ pin configurations
- `drivers/pinctrl/devicetree.c` / `devicetree.h` — parsing الـ Device Tree

#### الـ Public Headers (للـ drivers والـ consumers):
- `include/linux/pinctrl/pinctrl.h` — الـ API للـ pin controller drivers
- `include/linux/pinctrl/consumer.h` — الـ API للـ devices اللي بتستخدم pins
- `include/linux/pinctrl/machine.h` — الـ mapping table API
- `include/linux/pinctrl/pinmux.h` — الـ pinmux ops API
- `include/linux/pinctrl/pinconf.h` — الـ pinconf ops API
- `include/linux/pinctrl/pinconf-generic.h` — generic config parameters
- `include/linux/pinctrl/pinctrl-state.h` — أسماء الـ states المعيارية (default, sleep, ...)
- `include/linux/pinctrl/devinfo.h` — ربط الـ pinctrl بالـ device lifecycle

#### Hardware Drivers (أمثلة):
- `drivers/pinctrl/intel/pinctrl-intel.c` — Intel SoCs
- `drivers/pinctrl/freescale/pinctrl-imx.c` — NXP i.MX
- `drivers/pinctrl/qcom/pinctrl-msm.c` — Qualcomm Snapdragon
- `drivers/pinctrl/bcm/pinctrl-bcm2835.c` — Raspberry Pi (BCM2835)
- `drivers/pinctrl/samsung/pinctrl-samsung.c` — Samsung Exynos

---

### الملفات المهمة اللي المبرمج لازم يعرفها

| الملف | السبب |
|-------|-------|
| `drivers/pinctrl/core.c` | الـ implementation الكاملة للـ core — بيستخدم كل الـ structs المعرّفة هنا |
| `include/linux/pinctrl/consumer.h` | لو بتكتب device driver وعايز تطلب pins |
| `include/linux/pinctrl/pinctrl.h` | لو بتكتب pin controller driver جديد |
| `drivers/pinctrl/pinmux.h` | الـ internal API للـ mux بين الـ core والـ drivers |
| `drivers/pinctrl/devicetree.h` | فهم إزاي الـ DT بيتحوّل لـ mappings |
## Phase 2: شرح الـ Pinctrl Framework

### المشكلة اللي الـ Subsystem بيحلها

أي SoC حديث — خد مثلاً Raspberry Pi 4 أو STM32H7 — فيه مئات الـ pins على الـ chip. كل pin ممكن يشتغل بأكتر من وظيفة:

- نفس الـ pin ممكن يكون UART TX أو SPI MISO أو GPIO عادي أو I2C SDA.
- كمان لازم تقدر تبرمج خصائصه: pull-up/pull-down، drive strength، slew rate، open-drain.

من غير framework موحد، كل driver كان بيعمل اللي عايزه مباشرة — UART driver يكتب في registers معينة عشان يعمل mux، و SPI driver يكتب في registers تانية. النتيجة:
- **تعارض**: اتنين drivers يطالبوا نفس الـ pin في نفس الوقت.
- **عدم تنظيم**: مفيش مكان مركزي يقولك مين مالك الـ pin ده.
- **صعوبة الـ portability**: كل board لازم تفضل تعدل في كل driver على حدة.

---

### الحل اللي الـ Kernel اتبعه

الـ **pinctrl subsystem** هو الـ layer الوسيط اللي بيجمع كل حاجة متعلقة بالـ pins في مكان واحد. فكرته:

1. **Abstraction**: كل pin controller (الـ hardware IP اللي بيتحكم في الـ pins) بيتسجل كـ `pinctrl_dev` واحد.
2. **Mapping table**: في جدول مركزي (`pinctrl_maps`) بيقول "الـ UART device عايز الـ group اللي اسمه uart0_pins يشتغل بـ function اسمه uart".
3. **State machine**: كل device عنده states (default، sleep، idle)، والـ framework بيطبق الـ state المطلوبة بالـ mux + config settings المناسبة.
4. **Ownership tracking**: الـ `pin_desc` بيحتفظ بـ `mux_owner` عشان يمنع التعارض.

---

### التشبيه الحقيقي — لوحة توصيلات الكهربائي

تخيل لوحة كهرباء في عمارة كبيرة:

| مكوّن حقيقي | مقابله في الـ Kernel |
|---|---|
| اللوحة الكهربائية الرئيسية | `pinctrl_dev` — الـ pin controller نفسه |
| كل مقبس/نقطة توصيل | `pin_desc` — وصف الـ pin الواحد |
| مجموعة مقابس في غرفة واحدة | `pingroup` / `group_desc` — group of pins |
| وظيفة الغرفة (مطبخ/حمام/غرفة نوم) | `pinfunction` — الـ function (uart, spi, i2c) |
| خريطة التوصيلات للعمارة كلها | `pinctrl_maps` — mapping table |
| بطاقة "محجوز لـ X" على المقبس | `mux_owner` في `pin_desc` |
| شيت التشغيل (نهار/ليل/طوارئ) | `pinctrl_state` — default/sleep/idle |
| المهندس اللي بيطبق الشيت | الـ pinctrl core نفسه |
| كهربائي كل شقة | pin controller driver (مثلاً `pinctrl-bcm2835.c`) |

الفكرة الجوهرية: الكهربائي (الـ driver) بيعرف إزاي يوصل الأسلاك الفعلية، لكن المهندس المركزي (الـ core) هو اللي بيقرر مين يشتغل امتى وبيمنع التعارض.

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Consumer Drivers                          │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│   │  UART    │  │   SPI    │  │   I2C    │  │  GPIO subsys │   │
│   │  driver  │  │  driver  │  │  driver  │  │  (gpiolib)   │   │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│        │pinctrl_get()│              │                │           │
│        │pinctrl_     │              │                │           │
│        │lookup_state()             │           pinctrl_gpio_    │
│        │pinctrl_select_state()     │           request()        │
└────────┼─────────────┼─────────────┼────────────────┼───────────┘
         │             │             │                │
┌────────▼─────────────▼─────────────▼────────────────▼───────────┐
│                    pinctrl CORE                                   │
│                                                                   │
│  ┌─────────────────────┐    ┌──────────────────────────────────┐ │
│  │  pinctrl_maps (list)│    │  per-device pinctrl handle        │ │
│  │  ┌───────────────┐  │    │  ┌──────────────────────────────┐│ │
│  │  │ pinctrl_maps  │  │    │  │ struct pinctrl               ││ │
│  │  │  .maps[]      │  │    │  │   .states (list)             ││ │
│  │  │  pinctrl_map  │  │    │  │     ├─ "default"             ││ │
│  │  │   .dev_name   │  │    │  │     │    .settings (list)    ││ │
│  │  │   .name(state)│  │    │  │     │      ├─ mux setting    ││ │
│  │  │   .type       │  │    │  │     │      └─ config setting ││ │
│  │  │   .ctrl_dev   │  │    │  │     └─ "sleep"               ││ │
│  │  │   .data       │  │    │  └──────────────────────────────┘│ │
│  │  └───────────────┘  │    └──────────────────────────────────┘ │
│  └─────────────────────┘                                         │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐   │
│  │  pinctrl_dev (per controller)                              │   │
│  │   .pin_desc_tree (radix tree, pin# → pin_desc)            │   │
│  │   .pin_group_tree (radix tree, selector → group_desc)     │   │
│  │   .pin_function_tree (radix tree, selector → function)    │   │
│  │   .gpio_ranges (list of pinctrl_gpio_range)               │   │
│  │   .desc → pinctrl_desc                                    │   │
│  │              ├─ .pctlops  (pinctrl_ops vtable)            │   │
│  │              ├─ .pmxops   (pinmux_ops vtable)             │   │
│  │              └─ .confops  (pinconf_ops vtable)            │   │
│  └───────────────────────────────────────────────────────────┘   │
└──────────────────────────────────┬──────────────────────────────┘
                                   │ calls vtable ops
┌──────────────────────────────────▼──────────────────────────────┐
│              Pin Controller Drivers (Platform Specific)          │
│   ┌──────────────────┐   ┌──────────────────┐                   │
│   │ pinctrl-bcm2835  │   │ pinctrl-stm32    │                   │
│   │ (Raspberry Pi)   │   │ (STM32 SoCs)     │                   │
│   └──────────────────┘   └──────────────────┘                   │
│           │                       │                              │
│    writes to MMIO          writes to MMIO                        │
│    GPIO/FSEL registers     MODER/AFRL/PUPDR regs                 │
└─────────────────────────────────────────────────────────────────┘
                                   │
┌──────────────────────────────────▼──────────────────────────────┐
│                         Hardware                                 │
│   ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐     │
│   │PIN0│ │PIN1│ │PIN2│ │PIN3│ │PIN4│ │PIN5│ │PIN6│ │PIN7│ ... │
│   └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘     │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstractions — الأفكار المحورية

#### 1. الـ `pinctrl_dev` — قلب الـ Controller

```c
struct pinctrl_dev {
    struct list_head node;          // ربطه في القائمة العالمية للـ controllers
    const struct pinctrl_desc *desc; // الـ descriptor اللي الـ driver سجّله
    struct radix_tree_root pin_desc_tree;  // بحث سريع: pin number → pin_desc
    struct radix_tree_root pin_group_tree; // بحث سريع: selector → group
    struct radix_tree_root pin_function_tree; // بحث سريع: selector → function
    struct list_head gpio_ranges;    // الـ GPIO ranges اللي بيديرها الـ controller ده
    struct pinctrl *p;               // الـ pinctrl handle الخاص بالـ controller نفسه (للـ hog)
    struct pinctrl_state *hog_default; // الـ pins اللي الـ controller نفسه بيحجزها
    struct pinctrl_state *hog_sleep;
    struct mutex mutex;              // حماية العمليات المتزامنة
};
```

الـ **radix tree** — مش hash table عادية. ده tree بيستخدم الـ pin number كـ key مباشرةً، فالـ lookup بيبقى O(log n) في الأسوأ لكن في الغالب O(1) لو الـ pins قليلة. الـ `pin_desc_get()` inline function بتعمل ده ببساطة:

```c
static inline struct pin_desc *pin_desc_get(struct pinctrl_dev *pctldev,
                                            unsigned int pin)
{
    return radix_tree_lookup(&pctldev->pin_desc_tree, pin);
}
```

#### 2. الـ `pinctrl_desc` و الـ vtable — عقد الـ Driver

```c
struct pinctrl_desc {
    const char *name;
    const struct pinctrl_pin_desc *pins; // array بكل الـ pins
    unsigned int npins;
    const struct pinctrl_ops  *pctlops;  // grouping operations
    const struct pinmux_ops   *pmxops;   // mux operations
    const struct pinconf_ops  *confops;  // config operations (pull, drive, etc.)
    bool link_consumers; // انشئ device link مع كل consumer لضمان ترتيب الـ suspend/resume
};
```

الـ three vtables الثلاثة بتقسم المسؤوليات:
- **`pinctrl_ops`**: "إيه الـ groups الموجودة وإيه الـ pins في كل group؟"
- **`pinmux_ops`** (من `pinmux.h`): "اعمل mux لـ group معينة لـ function معينة"
- **`pinconf_ops`** (من `pinconf.h`): "برمج pull-up/drive strength/slew rate لـ pin معين"

#### 3. الـ `pinctrl` و `pinctrl_state` — نظام الـ States

```c
struct pinctrl {
    struct list_head node;      // في قائمة عالمية
    struct device *dev;         // الـ consumer device
    struct list_head states;    // قائمة بكل الـ states
    struct pinctrl_state *state; // الـ state الحالية
    struct list_head dt_maps;   // الـ maps اللي اتعملت من device tree
    struct kref users;          // reference counting
};

struct pinctrl_state {
    struct list_head node;
    const char *name;           // "default" أو "sleep" أو "idle" إلخ
    struct list_head settings;  // قائمة بالـ pinctrl_setting entries
};
```

الـ **`pinctrl_state`** مش بس اسم — هي قائمة من `pinctrl_setting` entries، كل entry بتقول:
- إما: "اعمل mux لـ group X علشان function Y" (نوع `PIN_MAP_TYPE_MUX_GROUP`)
- أو: "برمج configs معينة على pin/group Z" (نوع `PIN_MAP_TYPE_CONFIGS_PIN` أو `CONFIGS_GROUP`)

#### 4. الـ Mapping Table — الجسر بين الـ Board والـ Drivers

الـ **`pinctrl_map`** هو العقد الذي تكتبه في الـ board file أو device tree:

```c
struct pinctrl_map {
    const char *dev_name;      // "uart0" — الـ device اللي بتخدمه
    const char *name;          // "default" — اسم الـ state
    enum pinctrl_map_type type; // MUX_GROUP أو CONFIGS_PIN إلخ
    const char *ctrl_dev_name; // "pinctrl@7e200000" — الـ controller المسؤول
    union {
        struct pinctrl_map_mux mux;         // group + function names
        struct pinctrl_map_configs configs; // group_or_pin + configs array
    } data;
};
```

في الـ Device Tree الحديث، الـ `dt_node_to_map()` callback بتحول الـ DT nodes لـ `pinctrl_map` entries وتحطها في `pinctrl_maps` العالمية. الـ macro `for_each_pin_map` بيسمح للـ core يمشي على كل الـ maps:

```c
#define for_each_pin_map(_maps_node_, _map_)                               \
    list_for_each_entry(_maps_node_, &pinctrl_maps, node)                  \
        for (unsigned int __i = 0;                                          \
             __i < _maps_node_->num_maps && (_map_ = &_maps_node_->maps[__i]); \
             __i++)
```

#### 5. الـ `pin_desc` — سجل الـ Pin الواحد

```c
struct pin_desc {
    struct pinctrl_dev *pctldev; // أنا تابع لأنهي controller؟
    const char *name;            // "GPIO_0" أو "SPI_MOSI" إلخ
    void *drv_data;              // بيانات خاصة بالـ driver (الـ core ما بيلمسهاش)
#ifdef CONFIG_PINMUX
    unsigned int mux_usecount;   // كام حد طالبني؟ (مش boolean عشان نفس الـ pin ممكن يتطلب أكتر من مرة)
    const char *mux_owner;       // اسم الـ device اللي حاجزني دلوقتي
    const struct pinctrl_setting_mux *mux_setting; // الـ mux setting الحالية
    const char *gpio_owner;      // لو GPIO طالبني بـ pinctrl_gpio_request()
    struct mutex mux_lock;       // lock على مستوى الـ pin الواحد
#endif
};
```

الـ `mux_usecount` مش boolean — ده integer عشان ممكن mapping entries متعددة تطالب نفس الـ group/pin، وكل طلب بيزود الـ count. لما الـ count يرجع صفر، الـ pin بيتحرر.

---

### العلاقة بين الـ Structs — رسم توضيحي

```
pinctrl_dev
│
├─── desc ──────────────► pinctrl_desc
│                           ├─ pins[] ────► pinctrl_pin_desc[]
│                           │                 (number, name per pin)
│                           ├─ pctlops ──► pinctrl_ops vtable
│                           ├─ pmxops  ──► pinmux_ops vtable
│                           └─ confops ──► pinconf_ops vtable
│
├─── pin_desc_tree ─────► radix tree
│                           └─ [pin_number] ──► pin_desc
│                                               ├─ pctldev (back pointer)
│                                               ├─ mux_owner (string)
│                                               └─ mux_usecount
│
├─── pin_group_tree ────► radix tree
│                           └─ [selector] ──► group_desc
│                                             ├─ grp.name
│                                             └─ grp.pins[]
│
├─── pin_function_tree ─► radix tree
│                           └─ [selector] ──► pinfunction
│                                             ├─ name
│                                             └─ groups[]
│
├─── gpio_ranges ───────► list of pinctrl_gpio_range
│                           └─ { base, pin_base, npins, gc }
│
└─── p ─────────────────► pinctrl (self-hog handle)
                           └─ states
                               ├─ hog_default state
                               │    └─ settings list
                               │        └─ pinctrl_setting
                               │              ├─ type = MUX_GROUP
                               │              └─ data.mux = { group, func }
                               └─ hog_sleep state
```

---

### رحلة الـ Pin من الـ Device Tree للـ Hardware

```
Device Tree Node:
  uart0 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart0_pins_default>;
    pinctrl-1 = <&uart0_pins_sleep>;
  }

          │
          │ driver calls pinctrl_get(dev)
          ▼

pinctrl core يعمل struct pinctrl للـ UART device
          │
          │ يمشي على الـ DT nodes
          ▼

dt_node_to_map() في الـ controller driver
تحول الـ DT entries لـ pinctrl_map entries
وتضيفها لـ pinctrl_maps العالمية
          │
          │ driver calls pinctrl_lookup_state(p, "default")
          ▼

core يبني struct pinctrl_state بـ settings list
فيها: mux_setting { group=0, func=2 } + config_setting { pull=UP }
          │
          │ driver calls pinctrl_select_state(p, state)
          ▼

core يمشي على كل setting في الـ state:
  - للـ MUX: يتأكد مفيش owner تاني، يزود mux_usecount، يكال pmxops->set_mux()
  - للـ CONFIGS: يكال confops->pin_config_set()
          │
          │
          ▼

Controller driver يكتب في الـ MMIO registers:
  UART TX pin → Function 2 (UART) + Pull-UP + 8mA drive
```

---

### الـ GPIO Bridge — الربط مع الـ GPIO Subsystem

مفهوم مهم: الـ **GPIO subsystem** (الـ `gpiolib`) هو subsystem مستقل بيدير الـ GPIO lines. لكن على الـ hardware نفسه، الـ GPIO pins مبنية فوق نفس الـ physical pins اللي بيديرها الـ pinctrl.

الـ `pinctrl_gpio_range` هو الجسر:

```c
struct pinctrl_gpio_range {
    struct list_head node;
    const char *name;
    unsigned int id;
    unsigned int base;      // أول GPIO number في الـ range ده
    unsigned int pin_base;  // أول pin number في الـ pinctrl space المقابل له
    unsigned int npins;     // عدد الـ pins في الـ range
    unsigned int const *pins; // أو array صريح بالـ pin numbers
    struct gpio_chip *gc;   // pointer للـ gpio_chip المقابل
};
```

لما حد بيعمل `gpio_request()` على GPIO 45 مثلاً، الـ gpiolib بتكال `pinctrl_gpio_request()` في الـ pinctrl core، اللي بيدور على الـ range المناسب ويسجل الـ GPIO كـ `gpio_owner` في الـ `pin_desc`.

---

### إيه اللي الـ Core بيمتلكه مقابل إيه اللي بيفوّضه للـ Driver

| المسؤولية | الـ Core يمتلكه | الـ Driver ينفذه |
|---|---|---|
| تتبع مالكية الـ pin | `pin_desc.mux_owner` + `mux_usecount` | لا |
| الـ states وتطبيقها | `pinctrl_state`, `pinctrl_setting` | لا |
| الـ mapping table العالمية | `pinctrl_maps` list | يضيف entries بس |
| الـ radix trees للـ lookup | `pin_desc_tree`, `pin_group_tree` | لا |
| تحويل DT لـ maps | لا | `dt_node_to_map()` |
| كتابة الـ registers الفعلية | لا | `pmxops->set_mux()` |
| برمجة الـ pull/drive configs | لا | `confops->pin_config_set()` |
| عدد الـ groups والـ functions | اختياري (generic impl) | `pctlops->get_groups_count()` |
| الـ GPIO range mapping | يحتفظ بالـ list | Driver يضيف الـ range |
| الـ debugfs | ينشئ الـ root node | Driver يضيف `pin_dbg_show()` |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags, Enums, Config Options — Cheatsheet

#### الـ `enum pinctrl_map_type`

| القيمة | المعنى |
|---|---|
| `PIN_MAP_TYPE_INVALID` | entry غلط أو غير مهيأ |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمي — مش بيعمل أي حاجة، بس بيمنع error |
| `PIN_MAP_TYPE_MUX_GROUP` | بيحدد function لـ group من الـ pins |
| `PIN_MAP_TYPE_CONFIGS_PIN` | بيطبق config (pull, drive strength...) على pin واحد |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | بيطبق config على group كاملة |

#### الـ Config Options (Kconfig)

| الـ Option | الأثر |
|---|---|
| `CONFIG_PINCTRL` | تفعيل subsystem كله |
| `CONFIG_PINMUX` | دعم الـ mux — بيضيف fields في `pin_desc` |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | بيضيف `pin_group_tree` في `pinctrl_dev` |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | بيضيف `pin_function_tree` في `pinctrl_dev` |
| `CONFIG_GENERIC_PINCONF` | دعم generic pin configuration |
| `CONFIG_DEBUG_FS` | بيضيف `device_root` debugfs في `pinctrl_dev` |
| `CONFIG_OF` | دعم Device Tree parsing |

#### الـ `PINFUNCTION_FLAG_GPIO`

| الـ Flag | القيمة | الاستخدام |
|---|---|---|
| `PINFUNCTION_FLAG_GPIO` | `BIT(0)` | بيوضح إن الـ function دي هي GPIO function |

#### Macros — Cheatsheet

| الـ Macro | الغرض |
|---|---|
| `PINCTRL_PIN(a, b)` | تعريف pin descriptor باسم |
| `PINCTRL_PIN_ANON(a)` | pin descriptor بدون اسم |
| `PINCTRL_PINGROUP(name, pins, n)` | تعريف pin group |
| `PINCTRL_GROUP_DESC(name, pins, n, data)` | تعريف group_desc جاهز |
| `PINCTRL_PINFUNCTION(name, groups, n)` | تعريف pinfunction |
| `PINCTRL_GPIO_PINFUNCTION(name, groups, n)` | pinfunction مخصوص للـ GPIO |
| `PIN_MAP_DUMMY_STATE(dev, state)` | mapping entry وهمي |
| `PIN_MAP_MUX_GROUP(dev, state, pinctrl, grp, func)` | mapping entry لـ mux |
| `PIN_MAP_MUX_GROUP_HOG(dev, state, grp, func)` | mux mapping محجوز للـ driver نفسه |
| `PIN_MAP_CONFIGS_PIN(dev, state, pinctrl, pin, cfgs)` | config entry لـ pin |
| `PIN_MAP_CONFIGS_GROUP(dev, state, pinctrl, grp, cfgs)` | config entry لـ group |
| `for_each_pin_map(maps_node, map)` | iteration على كل entries في الـ global map list |

---

### كل الـ Structs المهمة

---

#### `struct pinctrl_dev`

**الغرض:** الـ object الرئيسي اللي بيمثل pin controller جوا الـ kernel — كل driver بيسجّل نفسه عن طريق هيكل زي ده.

| الـ Field | النوع | المعنى |
|---|---|---|
| `node` | `struct list_head` | ربطه في الـ global list `pinctrl_list` |
| `desc` | `const struct pinctrl_desc *` | الـ descriptor اللي فيه الـ ops والـ pins |
| `pin_desc_tree` | `struct radix_tree_root` | شجرة البحث السريع بـ pin number |
| `pin_group_tree` | `struct radix_tree_root` | (اختياري) شجرة الـ groups |
| `num_groups` | `unsigned int` | عدد الـ groups |
| `pin_function_tree` | `struct radix_tree_root` | (اختياري) شجرة الـ functions |
| `num_functions` | `unsigned int` | عدد الـ functions |
| `gpio_ranges` | `struct list_head` | قائمة نطاقات الـ GPIO المرتبطة |
| `dev` | `struct device *` | الـ device الفعلي في الـ kernel |
| `owner` | `struct module *` | الـ module المالك — للـ refcounting |
| `driver_data` | `void *` | بيانات خاصة بالـ driver |
| `p` | `struct pinctrl *` | نتيجة `pinctrl_get()` الخاصة بالـ controller نفسه (للـ hogging) |
| `hog_default` | `struct pinctrl_state *` | الـ default state المحجوز |
| `hog_sleep` | `struct pinctrl_state *` | الـ sleep state المحجوز |
| `mutex` | `struct mutex` | حماية كل العمليات على الـ controller ده |
| `device_root` | `struct dentry *` | debugfs entry (لو `CONFIG_DEBUG_FS`) |

**الـ connections:** يشاور على `pinctrl_desc` (الـ descriptor الثابت) وعلى `pinctrl` (الـ consumer handle اللي الـ driver بيعمله hog لنفسه).

---

#### `struct pinctrl_desc`

**الغرض:** الـ descriptor الثابت اللي بيقدمه الـ driver لحظة التسجيل — بيصف capabilities الـ controller.

| الـ Field | النوع | المعنى |
|---|---|---|
| `name` | `const char *` | اسم الـ controller |
| `pins` | `const struct pinctrl_pin_desc *` | مصفوفة وصف كل pin |
| `npins` | `unsigned int` | عدد الـ pins |
| `pctlops` | `const struct pinctrl_ops *` | vtable لعمليات الـ groups والـ DT parsing |
| `pmxops` | `const struct pinmux_ops *` | vtable لعمليات الـ mux |
| `confops` | `const struct pinconf_ops *` | vtable لعمليات الـ config |
| `owner` | `struct module *` | الـ module المالك |
| `num_custom_params` | `unsigned int` | عدد الـ custom config params |
| `custom_params` | `const struct pinconf_generic_params *` | الـ custom params |
| `custom_conf_items` | `const struct pin_config_item *` | معلومات العرض في debugfs |
| `link_consumers` | `bool` | إنشاء device link بين الـ controller وكل consumer |

---

#### `struct pinctrl_ops`

**الغرض:** الـ vtable الأساسي اللي بيتعامل مع الـ groups والـ Device Tree — الـ driver بيحط فيه function pointers.

| الـ Field | المعنى |
|---|---|
| `get_groups_count` | إرجاع عدد الـ groups |
| `get_group_name` | إرجاع اسم group معينة |
| `get_group_pins` | إرجاع مصفوفة الـ pins في group معينة |
| `pin_dbg_show` | (اختياري) عرض معلومات pin في debugfs |
| `dt_node_to_map` | تحويل DT node لـ mapping entries |
| `dt_free_map` | تحرير mapping entries اللي `dt_node_to_map` عملها |

---

#### `struct pinctrl`

**الغرض:** الـ handle الخاص بكل device consumer — لما device تعمل `pinctrl_get()` بترجع pointer على struct زي ده.

| الـ Field | النوع | المعنى |
|---|---|---|
| `node` | `struct list_head` | ربطه في global consumer list |
| `dev` | `struct device *` | الـ device المالك للـ handle ده |
| `states` | `struct list_head` | قائمة كل الـ states المتاحة |
| `state` | `struct pinctrl_state *` | الـ state الحالي المفعّل |
| `dt_maps` | `struct list_head` | الـ mappings اللي اتعملت من الـ DT ديناميكياً |
| `users` | `struct kref` | reference count — لما توصل لصفر بيتحرر |

---

#### `struct pinctrl_state`

**الغرض:** بيمثل state واحد (مثلاً "default" أو "sleep") لـ device معينة.

| الـ Field | النوع | المعنى |
|---|---|---|
| `node` | `struct list_head` | ربطه في قائمة `pinctrl->states` |
| `name` | `const char *` | اسم الـ state — "default", "sleep", إلخ |
| `settings` | `struct list_head` | قائمة الـ settings اللازم تطبيقها |

---

#### `struct pinctrl_setting`

**الغرض:** setting واحد داخل state — إما mux أو config لـ pin أو group.

| الـ Field | النوع | المعنى |
|---|---|---|
| `node` | `struct list_head` | ربطه في `pinctrl_state->settings` |
| `type` | `enum pinctrl_map_type` | نوع الـ setting |
| `pctldev` | `struct pinctrl_dev *` | الـ controller المسؤول عن التنفيذ |
| `dev_name` | `const char *` | اسم الـ device المالك |
| `data.mux` | `struct pinctrl_setting_mux` | لو النوع MUX_GROUP |
| `data.configs` | `struct pinctrl_setting_configs` | لو النوع CONFIGS_* |

---

#### `struct pinctrl_setting_mux`

**الغرض:** بيحدد الـ group والـ function اللازم إعداد الـ mux عليهم.

| الـ Field | المعنى |
|---|---|
| `group` | رقم الـ group selector |
| `func` | رقم الـ function selector |

---

#### `struct pinctrl_setting_configs`

**الغرض:** مصفوفة config values هتتكتب في الهاردوير.

| الـ Field | المعنى |
|---|---|
| `group_or_pin` | رقم الـ group أو الـ pin |
| `configs` | مصفوفة القيم (format محدد بالـ controller) |
| `num_configs` | عدد العناصر في المصفوفة |

---

#### `struct pin_desc`

**الغرض:** وصف كل pin بشكل داخلي في الـ core — بيتخزن في `pin_desc_tree`.

| الـ Field | النوع | المعنى |
|---|---|---|
| `pctldev` | `struct pinctrl_dev *` | الـ controller المالك |
| `name` | `const char *` | اسم الـ pin |
| `dynamic_name` | `bool` | هل الاسم اتخصص ديناميكياً |
| `drv_data` | `void *` | بيانات خاصة بالـ driver |
| `mux_usecount` | `unsigned int` | عدد مرات الـ claim (0 = حر) |
| `mux_owner` | `const char *` | اسم الـ device الحالك |
| `mux_setting` | `const struct pinctrl_setting_mux *` | آخر mux setting اتطبق |
| `gpio_owner` | `const char *` | اسم الـ GPIO المالك (لو `pinctrl_gpio_request` اتعملت) |
| `mux_lock` | `struct mutex` | mutex خاص بكل pin لحماية الـ mux state |

الـ fields من `mux_usecount` لـ `mux_lock` بتظهر بس لو `CONFIG_PINMUX` مفعّل.

---

#### `struct pinctrl_maps`

**الغرض:** wrapper بيحتوي على جزء من الـ global mapping table.

| الـ Field | المعنى |
|---|---|
| `node` | ربطه في global list `pinctrl_maps` |
| `maps` | مصفوفة `pinctrl_map` entries |
| `num_maps` | عدد الـ entries |

---

#### `struct pinctrl_map`

**الغرض:** entry واحد في الـ mapping table — بيربط device باسمها بـ state وـ function وـ controller.

| الـ Field | المعنى |
|---|---|
| `dev_name` | اسم الـ device المستهدفة |
| `name` | اسم الـ state |
| `type` | نوع الـ mapping |
| `ctrl_dev_name` | اسم الـ pin controller المسؤول |
| `data.mux` | بيانات الـ mux (لو MUX_GROUP) |
| `data.configs` | بيانات الـ config (لو CONFIGS_*) |

---

#### `struct group_desc`

**الغرض:** generic descriptor لـ pin group — بيستخدم مع `CONFIG_GENERIC_PINCTRL_GROUPS`.

| الـ Field | المعنى |
|---|---|
| `grp` | الـ `pingroup` (اسم + pins + عددهم) |
| `data` | بيانات خاصة بالـ driver |

---

#### `struct pingroup`

**الغرض:** الوصف الأساسي لـ pin group.

| الـ Field | المعنى |
|---|---|
| `name` | اسم الـ group |
| `pins` | مصفوفة أرقام الـ pins |
| `npins` | عدد الـ pins |

---

#### `struct pinctrl_gpio_range`

**الغرض:** بيعرّف نطاق GPIO مرتبط بـ pin controller معين.

| الـ Field | المعنى |
|---|---|
| `node` | ربطه في `pinctrl_dev->gpio_ranges` |
| `name` | اسم الـ range |
| `id` | رقم الـ chip |
| `base` | أول رقم GPIO في الـ range |
| `pin_base` | أول رقم pin في الـ range |
| `npins` | عدد الـ pins |
| `pins` | مصفوفة اختيارية بأرقام الـ pins |
| `gc` | pointer اختياري على `gpio_chip` |

---

#### `struct pinfunction`

**الغرض:** وصف function واحدة (مثلاً UART, SPI, GPIO...).

| الـ Field | المعنى |
|---|---|
| `name` | اسم الـ function |
| `groups` | مصفوفة أسماء الـ groups المدعومة |
| `ngroups` | عدد الـ groups |
| `flags` | `PINFUNCTION_FLAG_GPIO` لو هي GPIO function |

---

#### `struct pinctrl_pin_desc`

**الغرض:** وصف pin فيزيائي واحد — بيقدمه الـ driver لحظة التسجيل.

| الـ Field | المعنى |
|---|---|
| `number` | رقم الـ pin الفريد (global) |
| `name` | اسمه على الـ datasheet |
| `drv_data` | بيانات driver خاصة |

---

### رسم علاقات الـ Structs

```
                    ┌─────────────────────────────┐
                    │       pinctrl_desc           │  ← Driver يملأه ويسجله
                    │  name, pins[], npins         │
                    │  pctlops ──────────────────────→ pinctrl_ops (vtable)
                    │  pmxops  ──────────────────────→ pinmux_ops  (vtable)
                    │  confops ──────────────────────→ pinconf_ops (vtable)
                    └────────────┬────────────────┘
                                 │ desc pointer
                                 ▼
┌──────────────────────────────────────────────────────┐
│                   pinctrl_dev                        │  ← Core ينشئه
│  node ──→ [global pinctrl_list]                      │
│  desc ────────────────────────────────────────────▶  │
│  pin_desc_tree (radix) ──→ { pin_desc, pin_desc, … } │
│  pin_group_tree (radix) ──→ { group_desc, … }        │
│  pin_function_tree (radix) ──→ { function_desc, … }  │
│  gpio_ranges ──→ [pinctrl_gpio_range, …]             │
│  dev ──→ struct device                               │
│  p ───────────────────────────────────────────────▶ pinctrl (hog)
│  hog_default ─────────────────────────────────────▶ pinctrl_state
│  hog_sleep ───────────────────────────────────────▶ pinctrl_state
│  mutex  (per-controller lock)                        │
└──────────────────────────────────────────────────────┘

          pin_desc_tree يحتوي على:
          ┌────────────────────────┐
          │       pin_desc         │
          │  pctldev ─────────────▶ pinctrl_dev
          │  name                  │
          │  mux_usecount          │
          │  mux_owner             │
          │  mux_setting ─────────▶ pinctrl_setting_mux
          │  gpio_owner            │
          │  mux_lock              │
          └────────────────────────┘

Consumer side:
          ┌────────────────────────┐
          │       pinctrl          │  ← نتيجة pinctrl_get()
          │  node ──→ [global list]│
          │  dev ──→ consumer dev  │
          │  states ──→ list ──────────→ pinctrl_state ──→ pinctrl_state …
          │  state ───────────────────→ pinctrl_state (الحالي)
          │  dt_maps ──→ list ─────────→ pinctrl_maps
          │  users (kref)          │
          └────────────────────────┘

          pinctrl_state:
          ┌────────────────────────┐
          │    pinctrl_state       │
          │  name = "default"      │
          │  settings ──→ list ────────→ pinctrl_setting ──→ pinctrl_setting …
          └────────────────────────┘

          pinctrl_setting:
          ┌────────────────────────────────────┐
          │       pinctrl_setting               │
          │  type = MUX_GROUP                   │
          │  pctldev ──────────────────────────▶ pinctrl_dev
          │  data.mux = { group=2, func=5 }     │
          └────────────────────────────────────┘
          ┌────────────────────────────────────┐
          │       pinctrl_setting               │
          │  type = CONFIGS_PIN                 │
          │  pctldev ──────────────────────────▶ pinctrl_dev
          │  data.configs = { pin, *cfgs, n }   │
          └────────────────────────────────────┘

Global mapping table:
[pinctrl_maps] ──▶ [pinctrl_maps] ──▶ [pinctrl_maps]
      │                  │
      ▼                  ▼
  pinctrl_map[]      pinctrl_map[]
  (board/DT static   (DT dynamic
   entries)           entries via dt_maps)
```

---

### دورة حياة الـ Pin Controller

#### المرحلة 1: التسجيل (Registration)

```
Driver Probe
    │
    ├─ يملأ pinctrl_desc (name, pins, ops)
    │
    ├─ pinctrl_register_and_init(desc, dev, drvdata, &pctldev)
    │       │
    │       ├─ ينشئ struct pinctrl_dev
    │       ├─ يبني pin_desc_tree من desc->pins
    │       ├─ يحط في global pinctrl_list
    │       └─ يرجع pctldev (بس لسه مش enabled)
    │
    └─ pinctrl_enable(pctldev)
            │
            ├─ يعمل pinctrl_get() لنفسه (self-hogging)
            ├─ يبحث في pinctrl_maps عن entries بـ dev_name == pctldev->dev
            ├─ لو لاقى: يعمل hog_default و hog_sleep states
            └─ يطبق hog_default تلقائياً
```

#### المرحلة 2: الاستخدام من Consumer

```
Consumer Driver Probe
    │
    ├─ pinctrl_get(dev)
    │       │
    │       ├─ ينشئ struct pinctrl
    │       ├─ يبحث في pinctrl_maps عن كل entries للـ dev
    │       ├─ لكل entry: ينشئ pinctrl_state (لو مش موجودة)
    │       │              وينشئ pinctrl_setting يضيفه للـ state
    │       └─ يرجع handle
    │
    ├─ pinctrl_lookup_state(p, "default")
    │       └─ بيبحث في p->states بالاسم
    │
    └─ pinctrl_select_state(p, state)
            │
            ├─ لكل setting في state->settings:
            │       ├─ لو MUX_GROUP: ops->set_mux(pctldev, func, group)
            │       ├─ لو CONFIGS_PIN: ops->pin_config_set(...)
            │       └─ لو CONFIGS_GROUP: ops->pin_config_group_set(...)
            └─ يحدّث p->state
```

#### المرحلة 3: التحرير (Teardown)

```
Consumer Driver Remove
    │
    └─ pinctrl_put(p)
            │
            ├─ kref_put(&p->users, ...)
            └─ لو وصل لصفر:
                    ├─ يحرر كل pinctrl_state وـ pinctrl_setting
                    ├─ يحرر dt_maps
                    └─ يحرر struct pinctrl

Controller Driver Remove
    │
    └─ pinctrl_unregister(pctldev)
            │
            ├─ يلغي الـ hogged states
            ├─ يشيله من global list
            ├─ يمسح pin_desc_tree
            └─ يحرر struct pinctrl_dev
```

---

### Call Flow Diagrams

#### GPIO Request عبر Pinctrl

```
gpio_request(gpio_num)
    │
    └─ pinctrl_gpio_request(pctldev, pin)
            │
            ├─ pin_desc_get(pctldev, pin)   ← radix_tree_lookup
            │       └─ يرجع pin_desc
            │
            ├─ mutex_lock(&pin_desc->mux_lock)
            │
            ├─ لو mux_usecount > 0: error (pin محجوز)
            │
            ├─ ops->gpio_request_enable(pctldev, range, pin)
            │       └─ الـ driver يكتب في الـ register
            │
            ├─ pin_desc->gpio_owner = gpio_name
            └─ mutex_unlock(&pin_desc->mux_lock)
```

#### تطبيق Mux Setting

```
pinctrl_select_state(p, state)
    │
    ├─ list_for_each_entry(setting, &state->settings, node)
    │
    └─ [لكل setting من نوع MUX_GROUP]
            │
            ├─ pinmux_apply_mux_setting(pctldev, setting)
            │       │
            │       ├─ ops->get_group_pins(pctldev, group, &pins, &npins)
            │       │
            │       ├─ لكل pin في الـ group:
            │       │       ├─ pin_desc_get(pctldev, pin)
            │       │       ├─ mutex_lock(&desc->mux_lock)
            │       │       ├─ desc->mux_usecount++
            │       │       ├─ desc->mux_owner = dev_name
            │       │       └─ mutex_unlock(&desc->mux_lock)
            │       │
            │       └─ pmxops->set_mux(pctldev, func_selector, group_selector)
            │               └─ الـ driver يكتب في hardware register
            │
            └─ setting->pctldev->desc->pmxops->set_mux(...)
```

#### تطبيق Config Setting

```
pinctrl_select_state(p, state)
    │
    └─ [لكل setting من نوع CONFIGS_PIN]
            │
            └─ pinconf_apply_setting(setting)
                    │
                    └─ confops->pin_config_set(
                            pctldev,
                            setting->data.configs.group_or_pin,
                            setting->data.configs.configs,
                            setting->data.configs.num_configs
                       )
                            └─ الـ driver يفسر كل config value
                               ويكتبها في الـ register المناسب
```

#### البحث عن Pin Controller

```
get_pinctrl_dev_from_devname(dev_name)
    │
    └─ mutex_lock(&pinctrl_list_mutex)
       list_for_each_entry(pctldev, &pinctrl_list, node)
           │
           ├─ strcmp(dev_name(pctldev->dev), dev_name)
           └─ لو match: mutex_unlock() → return pctldev

get_pinctrl_dev_from_of_node(np)
    │
    └─ نفس الفكرة لكن بيقارن device_node pointer
```

#### الـ `pin_desc_get` (inline)

```c
static inline struct pin_desc *pin_desc_get(struct pinctrl_dev *pctldev,
                                            unsigned int pin)
{
    /* O(log n) lookup — radix tree indexed by pin number */
    return radix_tree_lookup(&pctldev->pin_desc_tree, pin);
}
```

---

### استراتيجية الـ Locking

#### الـ Locks الموجودة

| الـ Lock | النوع | يحمي إيه |
|---|---|---|
| `pinctrl_dev->mutex` | `struct mutex` | كل العمليات على controller معين — states, groups, functions |
| `pin_desc->mux_lock` | `struct mutex` | الـ mux state الخاص بكل pin منفرداً |
| `pinctrl_maps_mutex` | `struct mutex` (global) | الـ global mapping list `pinctrl_maps` |
| `pinctrl_list_mutex` | `struct mutex` (global) | الـ global controller list |

#### ترتيب الـ Locking (Lock Ordering)

الـ kernel بيلتزم بالترتيب التالي لتجنب الـ deadlock:

```
pinctrl_list_mutex        (الأعلى — global)
    │
    ▼
pinctrl_maps_mutex        (global — للـ mappings)
    │
    ▼
pinctrl_dev->mutex        (per-controller)
    │
    ▼
pin_desc->mux_lock        (per-pin — الأدنى)
```

يعني: لو محتاج تاخد أكتر من lock، لازم تاخدهم بالترتيب ده من فوق لتحت.

#### ملاحظات مهمة

- **الـ `pinctrl_find_gpio_range_from_pin_nolock`**: اسمه صريح — بيتّقال من context بتاخد `pinctrl_dev->mutex` بالفعل، فمبياخدش اللوك تاني لتجنب deadlock.
- **الـ `pin_desc->mux_lock`**: lock خاص بكل pin — ده بيسمح بـ concurrent access لـ pins مختلفة في نفس الوقت من غير ما controller lock الكبير يمنع ده.
- **الـ `kref` في `struct pinctrl`**: مش lock بالمعنى الكلاسيكي — بس بيضمن إن الـ struct مش بيتحرر وفيه حد لسه بيستخدمه.
- الـ `for_each_pin_map` macro بيفترض إن المستدعي شايل `pinctrl_maps_mutex` — لأنه بيعمل iteration على global list من غير ما ياخد اللوك جوا الـ macro نفسه.
## Phase 4: شرح الـ Functions

### ملخص سريع — Cheatsheet

#### Functions & APIs

| Function / Macro | Category | الغرض |
|---|---|---|
| `get_pinctrl_dev_from_devname()` | Lookup | جيب الـ `pinctrl_dev` من اسم الـ device |
| `get_pinctrl_dev_from_of_node()` | Lookup | جيب الـ `pinctrl_dev` من الـ DT node |
| `pin_get_from_name()` | Lookup | جيب رقم الـ pin من اسمه |
| `pin_get_name()` | Lookup | جيب اسم الـ pin من رقمه |
| `pin_desc_get()` | Lookup (inline) | جيب الـ `pin_desc` من الـ radix tree |
| `pinctrl_get_group_selector()` | Lookup | جيب الـ group selector من اسم الـ group |
| `pinctrl_find_gpio_range_from_pin_nolock()` | GPIO Range | ابحث عن الـ GPIO range اللي بيغطي الـ pin بدون lock |
| `pinctrl_force_sleep()` | State Control | اجبر الـ controller يدخل الـ sleep state |
| `pinctrl_force_default()` | State Control | اجبر الـ controller يرجع للـ default state |
| `pinctrl_generic_get_group_count()` | Generic Groups | عدد الـ groups المسجلة |
| `pinctrl_generic_get_group_name()` | Generic Groups | اسم الـ group بالـ selector بتاعه |
| `pinctrl_generic_get_group_pins()` | Generic Groups | الـ pins الموجودة في الـ group |
| `pinctrl_generic_get_group()` | Generic Groups | جيب الـ `group_desc` كاملاً |
| `pinctrl_generic_add_group()` | Generic Groups | سجّل group جديدة في الـ radix tree |
| `pinctrl_generic_remove_group()` | Generic Groups | احذف group من الـ radix tree |
| `for_each_pin_map()` | Macro / Iteration | iterate على كل entries في الـ global mapping table |

---

### المجموعة الأولى: Lookup Functions

الـ lookup functions دي هي الـ backbone اللي بتستخدمها الـ pinctrl core عشان تربط الـ names والـ IDs بالـ internal structures. كلها بتشتغل على الـ global lists أو الـ radix trees اللي موجودة في الـ `pinctrl_dev`.

---

#### `get_pinctrl_dev_from_devname`

```c
struct pinctrl_dev *get_pinctrl_dev_from_devname(const char *dev_name);
```

بتعمل iterate على الـ global `pinctrl_dev_list` وبتدور على الـ controller اللي اسم الـ device بتاعه يطابق `dev_name`. بتُستخدم لما الـ mapping table بتحدد الـ controller باسمه (مش بالـ DT node). بترجع `NULL` لو ماوجدتش — والـ caller لازم يتعامل مع ده.

**Parameters:**
- `dev_name` — الـ string اللي بيتطابق مع `dev_name(pctldev->dev)`

**Return:** pointer للـ `pinctrl_dev` أو `NULL`

**Key details:**
- بتاخد الـ `pinctrl_maps_mutex` internally أو بتعتمد على الـ caller إنه شايل الـ lock الصح — راجع السياق
- الـ caller context: الـ pinctrl core أثناء تطبيق الـ mapping table

---

#### `get_pinctrl_dev_from_of_node`

```c
struct pinctrl_dev *get_pinctrl_dev_from_of_node(struct device_node *np);
```

نفس الفكرة بس بتدور بالـ DT `device_node` بدل الاسم. بتُستخدم أثناء الـ DT parsing لما `dt_node_to_map` بتحتاج تعرف أي controller هو المسؤول.

**Parameters:**
- `np` — الـ `device_node` الخاص بالـ pin controller في الـ device tree

**Return:** pointer للـ `pinctrl_dev` أو `NULL`

**Key details:**
- بتقارن الـ `np` بـ `of_node` الخاص بـ `pctldev->dev`
- DT-only path، مش موجودة لو مفيش `CONFIG_OF`

---

#### `pin_get_from_name`

```c
int pin_get_from_name(struct pinctrl_dev *pctldev, const char *name);
```

بتعمل iterate على الـ `pin_desc_tree` (radix tree) عشان تلاقي الـ pin اللي اسمه `name` وترجع رقمه (pin number). الـ pin number ده بيتستخدم كـ key في الـ radix tree وكـ ID في كل العمليات اللي بعد كده.

**Parameters:**
- `pctldev` — الـ pin controller device
- `name` — اسم الـ pin زي "PA0" أو "gpio23"

**Return:** pin number (unsigned int cast لـ int) أو `-EINVAL` لو الاسم ماوجدش

**Key details:**
- الـ radix tree بيتعمل iterate بالكامل — مش O(1)
- بتاخد الـ `pctldev->mutex` — ممكن تنام

---

#### `pin_get_name`

```c
const char *pin_get_name(struct pinctrl_dev *pctldev, const unsigned int pin);
```

الـ reverse operation: بتاخد الـ pin number وبترجع الاسم. بتعمل `pin_desc_get()` على الـ radix tree وبترجع `desc->name`.

**Parameters:**
- `pctldev` — الـ pin controller device
- `pin` — رقم الـ pin

**Return:** string باسم الـ pin أو `NULL` لو الـ pin ماوجدش

**Key details:**
- الـ returned pointer صالح طول عمر الـ pin descriptor
- لا بتاخد lock — الـ caller مسؤول

---

#### `pin_desc_get` (inline)

```c
static inline struct pin_desc *pin_desc_get(struct pinctrl_dev *pctldev,
                                             unsigned int pin)
{
    return radix_tree_lookup(&pctldev->pin_desc_tree, pin);
}
```

الأسرع والأبسط — بتعمل O(log n) lookup في الـ radix tree جوه الـ `pctldev` وبترجع الـ `pin_desc` مباشرة.

**Parameters:**
- `pctldev` — الـ pin controller device
- `pin` — الـ pin number كـ radix tree key

**Return:** pointer لـ `pin_desc` أو `NULL`

**Key details:**
- بدون locking — الـ caller لازم يكون شايل `pctldev->mutex` أو في سياق آمن
- بيتستخدم في كل حتة جوه الـ pinmux والـ pinconf code

---

#### `pinctrl_get_group_selector`

```c
int pinctrl_get_group_selector(struct pinctrl_dev *pctldev,
                               const char *pin_group);
```

بتاخد اسم الـ group وبترجع الـ integer selector بتاعه. الـ selector ده بيتستخدم في كل الـ pinmux operations (`set_mux`, `get_group_pins`, إلخ).

**Parameters:**
- `pctldev` — الـ pin controller device
- `pin_group` — اسم الـ group زي "spi0_grp"

**Return:** selector >= 0 أو `-EINVAL`

**Key details:**
- بتستخدم `pctlops->get_groups_count()` و `pctlops->get_group_name()` للـ iteration
- بتاخد `pctldev->mutex`

**Pseudocode:**
```
for selector in 0..get_groups_count(pctldev):
    name = get_group_name(pctldev, selector)
    if name == pin_group:
        return selector
return -EINVAL
```

---

### المجموعة التانية: GPIO Range Functions

الـ GPIO range هو الـ bridge بين الـ GPIO subsystem والـ pinctrl subsystem. لما الـ GPIO core محتاج يحجز pin أو يعرف مين بيتحكم فيه، بتستخدم دول.

---

#### `pinctrl_find_gpio_range_from_pin_nolock`

```c
extern struct pinctrl_gpio_range *
pinctrl_find_gpio_range_from_pin_nolock(struct pinctrl_dev *pctldev,
                                        unsigned int pin);
```

بتعمل iterate على `pctldev->gpio_ranges` list وبتدور على الـ range اللي الـ pin number بيقع جواه. النسخة `_nolock` دي بتفترض إن الـ caller شايل الـ lock بالفعل.

**Parameters:**
- `pctldev` — الـ pin controller device
- `pin` — الـ pin number

**Return:** pointer لـ `pinctrl_gpio_range` أو `NULL`

**Key details:**
- **لا بتاخد lock** — الـ caller لازم يكون شايل `pctldev->mutex`
- الـ sister function `pinctrl_find_gpio_range_from_pin()` (معرفة في `pinctrl.h`) بتاخد الـ lock بنفسها
- بتُستخدم من داخل `pinctrl_gpio_request()` وأي path سريع محتاج performance

**Pseudocode:**
```
list_for_each_entry(range, &pctldev->gpio_ranges, node):
    if range uses pin array:
        check each pin in range->pins[]
    else:
        if pin >= range->pin_base && pin < range->pin_base + range->npins:
            return range
return NULL
```

---

### المجموعة التالتة: State Control Functions

دول بيتحكموا في الـ power states للـ pin controller كله. بيشتغلوا على مستوى الـ controller مش الـ device.

---

#### `pinctrl_force_sleep`

```c
extern int pinctrl_force_sleep(struct pinctrl_dev *pctldev);
```

بتجبر الـ pin controller يطبق الـ `hog_sleep` state — ده الـ state اللي الـ controller نفسه حجزه (hog) ليه لما بيدخل sleep. بيتستخدم في الـ suspend path.

**Parameters:**
- `pctldev` — الـ pin controller device

**Return:** 0 on success أو error code

**Key details:**
- بتشتغل على `pctldev->hog_sleep` — لو `NULL` بترجع 0 بهدوء
- بتستدعي `pinctrl_select_state()` internally
- الـ caller context: `pm_ops->suspend` أو الـ sleep framework

---

#### `pinctrl_force_default`

```c
extern int pinctrl_force_default(struct pinctrl_dev *pctldev);
```

نفس الـ pattern بس بترجع للـ `hog_default` state. بيتستخدم في الـ resume path أو لما محتاج ترجع الـ controller لحالته الاعتيادية.

**Parameters:**
- `pctldev` — الـ pin controller device

**Return:** 0 on success أو error code

**Key details:**
- بتشتغل على `pctldev->hog_default`
- كمان بتستدعي `pinctrl_select_state()` internally
- مهمة لـ controllers اللي بيعملوا self-hogging في الـ probe

---

### المجموعة الرابعة: Generic Group Functions

دي الـ generic implementation للـ group management. بتشتغل بس لو `CONFIG_GENERIC_PINCTRL_GROUPS` معمول enable. الفكرة إن مش كل driver محتاج يكتب الـ group management من الأول — يستخدم دي وخلاص.

الـ storage بيكون في `pctldev->pin_group_tree` (radix tree) وكل entry هي `struct group_desc`.

---

#### `pinctrl_generic_get_group_count`

```c
int pinctrl_generic_get_group_count(struct pinctrl_dev *pctldev);
```

بترجع `pctldev->num_groups` مباشرة. بيتستخدم كـ implementation لـ `pctlops->get_groups_count`.

**Parameters:**
- `pctldev` — الـ pin controller device

**Return:** عدد الـ groups المسجلة

**Key details:**
- بدون locking — الـ value بتتغير بس أثناء الـ probe/remove
- بتُستخدم في `pinctrl_get_group_selector()` iteration

---

#### `pinctrl_generic_get_group_name`

```c
const char *pinctrl_generic_get_group_name(struct pinctrl_dev *pctldev,
                                           unsigned int group_selector);
```

بتعمل `radix_tree_lookup` على `pin_group_tree` بالـ `group_selector` كـ key وبترجع `group_desc->grp.name`.

**Parameters:**
- `pctldev` — الـ pin controller device
- `group_selector` — الـ index الخاص بالـ group

**Return:** اسم الـ group أو `NULL`

**Key details:**
- بيتستخدم كـ implementation لـ `pctlops->get_group_name`
- بتاخد `pctldev->mutex`

---

#### `pinctrl_generic_get_group_pins`

```c
int pinctrl_generic_get_group_pins(struct pinctrl_dev *pctldev,
                                   unsigned int group_selector,
                                   const unsigned int **pins,
                                   unsigned int *npins);
```

بتجيب الـ `group_desc` من الـ radix tree وبتملي `*pins` و`*npins` بالـ pin array الخاص بالـ group.

**Parameters:**
- `pctldev` — الـ pin controller device
- `group_selector` — الـ group index
- `pins` — output: pointer لـ array الـ pins
- `npins` — output: عدد الـ pins في الـ group

**Return:** 0 on success أو `-EINVAL` لو الـ group ماوجدتش

**Key details:**
- الـ output pointers بيشاوروا على الـ data الموجودة في الـ `group_desc` — لا copy
- بيتستخدم كـ implementation لـ `pctlops->get_group_pins`

---

#### `pinctrl_generic_get_group`

```c
struct group_desc *pinctrl_generic_get_group(struct pinctrl_dev *pctldev,
                                             unsigned int group_selector);
```

بترجع الـ `group_desc` كاملاً بدل ما ترجع الـ pins بس. مفيدة لو الـ driver محتاج يوصل لـ `data` الخاص بيه اللي خزّنه في الـ descriptor.

**Parameters:**
- `pctldev` — الـ pin controller device
- `group_selector` — الـ group index

**Return:** pointer لـ `group_desc` أو `NULL`

---

#### `pinctrl_generic_add_group`

```c
int pinctrl_generic_add_group(struct pinctrl_dev *pctldev, const char *name,
                              const unsigned int *pins, int num_pins,
                              void *data);
```

بتعمل allocate `group_desc` جديدة وبتخزنها في `pin_group_tree` بالـ `num_groups` الحالي كـ key، وبعدين بتعمل increment لـ `num_groups`.

**Parameters:**
- `pctldev` — الـ pin controller device
- `name` — اسم الـ group (string يتنسخ internally)
- `pins` — array بأرقام الـ pins في الـ group
- `num_pins` — عدد الـ pins
- `data` — driver-specific data بتتخزن في `group_desc->data`

**Return:** الـ group selector الجديد (>= 0) أو error code

**Key details:**
- بيعمل `kmemdup` للـ pins array — الـ caller مش محتاج يكمل يشيلها
- بتاخد `pctldev->mutex`
- الـ caller context: أثناء الـ driver probe، عادةً من `dt_node_to_map`

**Pseudocode:**
```
lock(pctldev->mutex)
selector = pctldev->num_groups
group = kzalloc(sizeof(group_desc))
group->grp.name = kstrdup(name)
group->grp.pins = kmemdup(pins, ...)
group->grp.npins = num_pins
group->data = data
radix_tree_insert(&pctldev->pin_group_tree, selector, group)
pctldev->num_groups++
unlock(pctldev->mutex)
return selector
```

---

#### `pinctrl_generic_remove_group`

```c
int pinctrl_generic_remove_group(struct pinctrl_dev *pctldev,
                                 unsigned int group_selector);
```

بتشيل الـ `group_desc` من الـ radix tree وبتحرر الـ memory المرتبطة بيها.

**Parameters:**
- `pctldev` — الـ pin controller device
- `group_selector` — الـ selector الخاص بالـ group اللي هتتشال

**Return:** 0 on success أو `-EINVAL`

**Key details:**
- بتحرر الـ `grp.name` والـ `grp.pins` والـ `group_desc` نفسه
- **لا** بيعمل decrement لـ `num_groups` — الـ selectors مش بتتعاد استخدامها
- بتاخد `pctldev->mutex`
- الـ caller context: الـ driver remove أو الـ DT unmap path

---

### المجموعة الخامسة: Iteration Macro

---

#### `for_each_pin_map`

```c
#define for_each_pin_map(_maps_node_, _map_)                            \
    list_for_each_entry(_maps_node_, &pinctrl_maps, node)               \
        for (unsigned int __i = 0;                                      \
             __i < _maps_node_->num_maps && (_map_ = &_maps_node_->maps[__i]); \
             __i++)
```

Compound macro بتعمل nested iteration: الـ outer loop بيمشي على كل `pinctrl_maps` node في الـ global `pinctrl_maps` list، والـ inner loop بيمشي على كل `pinctrl_map` entry جوا الـ node دي.

**Variables:**
- `_maps_node_` — متغير من نوع `struct pinctrl_maps *` — الـ caller بيعرفه
- `_map_` — متغير من نوع `const struct pinctrl_map *` — بيتملي تلقائياً

**Key details:**
- الـ caller **لازم** يكون شايل `pinctrl_maps_mutex` قبل الـ macro
- الـ global `pinctrl_maps` و`pinctrl_maps_mutex` معرفين كـ `extern` في نفس الـ header
- البنية الـ nested دي بتمشي على كل الـ mappings بأقل overhead ممكن

**مثال استخدام:**
```c
/* داخل pinctrl core لما بيطبق الـ settings */
struct pinctrl_maps *maps_node;
const struct pinctrl_map *map;

mutex_lock(&pinctrl_maps_mutex);
for_each_pin_map(maps_node, map) {
    if (strcmp(map->dev_name, dev_name) == 0) {
        /* process this map entry */
    }
}
mutex_unlock(&pinctrl_maps_mutex);
```

---

### Global Variables المعرفة في الـ Header

| Variable | النوع | الغرض |
|---|---|---|
| `pinctrl_maps_mutex` | `struct mutex` | يحمي الـ `pinctrl_maps` global list من الـ concurrent access |
| `pinctrl_maps` | `struct list_head` | الـ global list لكل الـ mapping tables المسجلة في الـ system |

الاتنين معرفين `extern` هنا ومعمول initialize ليهم في `core.c`. أي code بيستخدم `for_each_pin_map` لازم يشيل الـ `pinctrl_maps_mutex` يدوياً.

---

### العلاقة بين الـ Structures والـ Functions

```
pinctrl_dev
 ├── pin_desc_tree (radix)  ←── pin_desc_get(), pin_get_from_name()
 ├── pin_group_tree (radix) ←── pinctrl_generic_*_group()
 ├── gpio_ranges (list)     ←── pinctrl_find_gpio_range_from_pin_nolock()
 ├── hog_default / hog_sleep ←── pinctrl_force_default/sleep()
 └── mutex                  ←── كل الـ functions بتاخده

pinctrl_maps (global list)
 └── pinctrl_maps_mutex     ←── for_each_pin_map()
```

الـ `pin_desc_tree` هو الـ single source of truth لكل الـ pins في الـ controller. كل العمليات التانية (groups, mux, config) بتبدأ بـ lookup فيه أو بـ group selector يتحول لـ pins منه.
## Phase 5: دليل الـ Debugging الشامل

### Software Level

#### 1. debugfs — المدخل الرئيسي للـ pinctrl

الـ `struct pinctrl_dev` بتحتفظ بـ `device_root` وهو الـ `dentry` بتاع الـ debugfs. ده بيتفعّل بس لو `CONFIG_DEBUG_FS=y`.

```bash
# شوف كل الـ pinctrl controllers المتسجلين
ls /sys/kernel/debug/pinctrl/

# مثال: SoC اسمه pinctrl-bcm2835
ls /sys/kernel/debug/pinctrl/pinctrl-bcm2835/
```

**المدخلات المهمة جوه كل controller:**

| المدخل | المحتوى |
|--------|---------|
| `pins` | كل الـ pins المتسجلين مع أسمائهم وأرقامهم |
| `pingroups` | الـ pin groups المعرّفة (من `pin_group_tree`) |
| `pinmux-pins` | حالة كل pin: `function`, `owner`, `usecount` |
| `pinmux-functions` | كل الـ functions المتاحة (من `pin_function_tree`) |
| `pinconf-pins` | الـ configs الحالية لكل pin |
| `pinconf-groups` | الـ configs على مستوى الـ group |
| `gpio-ranges` | الـ GPIO ranges المربوطة بالـ controller |

```bash
# اقرأ كل الـ pins
cat /sys/kernel/debug/pinctrl/pinctrl-bcm2835/pins

# شوف مين owns كل pin (من pin_desc->mux_owner)
cat /sys/kernel/debug/pinctrl/pinctrl-bcm2835/pinmux-pins

# تحقق من الـ gpio ranges (من gpio_ranges list_head)
cat /sys/kernel/debug/pinctrl/pinctrl-bcm2835/gpio-ranges
```

**مثال output حقيقي من `pinmux-pins`:**

```
pin 14 (GPIO14): uart0 (GPIO UNCLAIMED) function uart0 group uart0_grp
pin 15 (GPIO15): uart0 (GPIO UNCLAIMED) function uart0 group uart0_grp
pin 17 (GPIO17): UNCLAIMED
```

الـ `UNCLAIMED` يعني `mux_usecount == 0` و`mux_owner == NULL`.

---

#### 2. sysfs — المعلومات العامة

```bash
# شوف الـ pin controller devices كـ platform devices
ls /sys/bus/platform/devices/ | grep pinctrl

# أو عن طريق الـ class
ls /sys/class/pinctrl/ 2>/dev/null

# الـ device الحقيقي (الـ struct device داخل pinctrl_dev)
ls /sys/devices/platform/soc/pinctrl@7e200000/

# اقرأ الـ pinctrl state الحالية للـ consumer device
cat /sys/devices/platform/soc/3f201000.serial/pinctrl-names 2>/dev/null
```

**الـ pinctrl consumer عنده في sysfs:**

```bash
# شوف الـ states المتاحة لـ device معين
cat /sys/devices/platform/3f201000.serial/pinctrl-0  # default state
```

---

#### 3. ftrace — تتبع الـ pinctrl events

```bash
# اعرض كل الـ tracepoints المتاحة للـ pinctrl
grep pinctrl /sys/kernel/debug/tracing/available_events

# فعّل pinctrl tracepoints
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# فعّل events محددة
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_mux_setting/enable
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_setting_mux/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# افعل الـ operation اللي عايز تتتبعها (مثلاً set state)
echo 0 > /sys/kernel/debug/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

**الـ tracepoints المهمة:**

| Tracepoint | يتفعل عند |
|-----------|----------|
| `pinctrl_mux_setting` | تطبيق `pinctrl_setting_mux` — بيغير الـ function |
| `pinctrl_setting_mux` | اختيار group/function جديد |
| `pinctrl_setting_configs` | تغيير الـ configs (drive strength، pull) |
| `pinctrl_select_state` | تحول الـ device من state لـ state |
| `pinctrl_map_add` | إضافة entry للـ mapping table |

```bash
# استخدم function_graph لتتبع pinctrl_select_state
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo pinctrl_select_state > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. printk و dynamic debug

```bash
# فعّل dynamic debug لـ pinctrl core
echo "module pinctrl +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/pinctrl/core.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل لكل drivers الـ pinctrl
echo "file drivers/pinctrl/* +pmfl" > /sys/kernel/debug/dynamic_debug/control

# مثال: تتبع operations الـ pinmux
echo "file drivers/pinctrl/pinmux.c +p" > /sys/kernel/debug/dynamic_debug/control

# شوف اللي فعّلته
cat /sys/kernel/debug/dynamic_debug/control | grep pinctrl
```

**Kernel boot parameter لو محتاج debug من البداية:**

```bash
# في kernel command line
dyndbg="file drivers/pinctrl/core.c +p; file drivers/pinctrl/pinmux.c +p"
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_DEBUG_FS` | **أساسي** — بيفعّل `device_root` في `pinctrl_dev` |
| `CONFIG_PINCTRL` | الـ subsystem الأساسي |
| `CONFIG_PINMUX` | بيفعّل `mux_usecount`, `mux_owner`, `mux_setting` في `pin_desc` |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | بيفعّل `pin_group_tree` في `pinctrl_dev` |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | بيفعّل `pin_function_tree` في `pinctrl_dev` |
| `CONFIG_GENERIC_PINCONF` | بيفعّل `custom_params` في `pinctrl_desc` |
| `CONFIG_DEBUG_PINCTRL` | تفاصيل إضافية في الـ logs (لو متاح) |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في `pinctrl_dev->mutex` و `pin_desc->mux_lock` |
| `CONFIG_LOCKDEP` | تحليل الـ lock dependencies |
| `CONFIG_KASAN` | كشف memory corruption في الـ radix trees |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_(PINCTRL|PINMUX|DEBUG_FS|GENERIC_PIN)"
```

---

#### 6. Subsystem-Specific Tools

```bash
# pinctrl utility من package libgpiod / pinctrl-utils (على بعض distros)
# على Raspberry Pi مثلاً
pinctrl get 14          # شوف state الـ pin 14
pinctrl funcs 14        # شوف الـ functions المتاحة للـ pin 14

# lsgpio أو gpioinfo (من libgpiod) للـ GPIO ranges
gpioinfo | head -20

# شوف الـ device links (من link_consumers في pinctrl_desc)
ls -la /sys/devices/platform/3f201000.serial/supplier:*/
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|----------------------|--------|-------|
| `pin X is already requested` | `mux_usecount > 0` و`mux_owner` مختلف | شوف من owns الـ pin من `pinmux-pins` debugfs |
| `pin X already used by Y` | conflict في الـ mux_owner — `pin_desc->mux_owner` غير NULL | راجع الـ mapping table، حذف الـ conflict |
| `could not find a pinctrl device for pin group X` | `pin_group_tree` مش لاقي الـ group | تحقق من اسم الـ group في DT يطابق اللي في الـ driver |
| `pinctrl state X not found` | `pinctrl_state` بالاسم ده مش موجود في `states` list | تحقق من `pinctrl-names` في DT |
| `could not get pinctrl, deferring probe` | الـ pinctrl device لسه مش probe وا — `get_pinctrl_dev_from_of_node` return NULL | probe ordering issue — راجع الـ `defer_probe` |
| `pin X: invalid config Y` | الـ config value مش supported من الـ driver | راجع `pinconf_ops` في الـ driver |
| `group selector X out of range` | `group_selector >= num_groups` — `pin_group_tree` مش بتحتوي الـ selector | bug في الـ driver — راجع `pinctrl_generic_add_group` |
| `could not find function X` | `pin_function_tree` مش لاقي الـ function | تحقق من الـ DT `function` string يطابق الـ driver |
| `Failed to register pinctrl device` | فشل `pinctrl_register_and_init` | شوف `dmesg` للتفاصيل — غالباً memory أو DT issue |
| `request already granted for this GPIO` | `gpio_owner` في `pin_desc` غير NULL | conflict بين GPIO و pinmux — راجع الـ ownership |

---

#### 8. Strategic Points لـ dump_stack() و WARN_ON()

**الأماكن المهمة في الـ core:**

```c
/* في pinctrl_get() — لما بنحاول نحصل على pinctrl لـ device */
/* ضيف WARN_ON لو pinctrl->state != NULL وإحنا بنعمل get تاني */
WARN_ON(p->state != NULL);

/* في pin_request() — لما بيتعمل mux claim */
/* تحقق إن mux_usecount مش بيتجاوز الحد المتوقع */
WARN_ON(desc->mux_usecount > 10);  /* رقم كبير جداً = bug */

/* dump_stack() عند فشل pinctrl_select_state */
if (ret) {
    dev_err(dev, "failed to select state %s\n", state->name);
    dump_stack();  /* عشان تعرف مين طلب الـ state change */
}

/* في get_pinctrl_dev_from_devname() — لو مش لاقي الـ controller */
WARN_ON_ONCE(!pctldev);

/* تحقق من الـ mutex مش محصل lock مرتين */
WARN_ON(mutex_is_locked(&pctldev->mutex));  /* قبل lock جديد */
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware تطابق الـ Kernel State

الفكرة: اقرأ الـ actual register values وقارنها بالـ kernel state اللي شايفه في debugfs.

```bash
# الخطوة 1: شوف الـ kernel state
cat /sys/kernel/debug/pinctrl/pinctrl-bcm2835/pinmux-pins | grep "GPIO14"
# output: pin 14 (GPIO14): uart0 function uart0 group uart0_grp

# الخطوة 2: قرأ الـ register الحقيقي للتأكد
# GPFSEL1 register في BCM2835 يتحكم في GPIO 10-19
# GPIO14 bits [14:12] = 100 = ALT0 (UART TX)
devmem2 0x3F200004 w  # GPFSEL1
```

---

#### 2. Register Dump Techniques

```bash
# devmem2 — أسرع طريقة لقراءة register واحد
# تثبيت: apt install devmem2
devmem2 <physical_address> [b|h|w]  # byte, halfword, word

# مثال: BCM2835 GPIO Function Select Register 1
devmem2 0x3F200004 w
# output: Value at address 0x3F200004 (0x...) : 0x00024000
# bits [14:12] = 100 = ALT0 = UART TX

# /dev/mem — للـ raw access
dd if=/dev/mem bs=4 count=1 skip=$((0x3F200004/4)) 2>/dev/null | xxd

# io utility (من i2c-tools)
io -4 0x3F200004  # قراءة 32-bit

# لو بتشتغل على ARM وعندك iomem:
cat /proc/iomem | grep -i gpio  # شوف الـ base address
```

**Script يعمل dump لكل الـ GPIO function select registers:**

```bash
#!/bin/bash
BASE=0x3F200000
for i in 0 4 8 c 10 14; do
    addr=$((BASE + 0x$i))
    printf "GPFSEL%d (0x%08X): " $((0x$i/4)) $addr
    devmem2 $addr w 2>/dev/null | grep "Value"
done
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

**عند debug مشاكل الـ pinctrl، استخدم logic analyzer على:**

```
Signal                  | ما تدور عليه
------------------------|--------------------------------------------------
UART TX (مثلاً GPIO14) | لو مش شايف signal = pin مش configured صح
SPI CLK                 | glitch عند state transition = race condition
I2C SCL/SDA             | stuck low = pin مش released من pinmux بشكل صح
GPIO interrupt pin       | تحقق الـ pull-up/pull-down configured صح
```

**Oscilloscope tips:**

```
- قيس الـ voltage levels: 3.3V vs 1.8V = مهم للـ drive strength config
- شوف الـ rise time: بطيء جداً = drive strength ضعيف
- شوف الـ noise: كبير جداً = مفيش termination أو ground loop
- شوف عند system suspend/resume: الـ hog_sleep state بيتطبق صح؟
```

---

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | الـ Kernel Log Pattern | التفسير |
|----------------------|------------------------|---------|
| Pin مش configured | لا يوجد log — الـ device ببساطة مش بيشتغل | راجع `pinmux-pins` في debugfs |
| Voltage mismatch (3.3V vs 1.8V) | `I2C: i2c-0 timeout` | الـ signal مش بيوصل threshold |
| Multiple drivers claim نفس الـ pin | `pin X is already requested` | conflict في الـ mapping table |
| DT pinctrl node خاطئ | `could not find a pinctrl device for pin group` | اسم الـ group في DT غلط |
| Sleep state مش متعرف | `pinctrl state sleep not found` | ناقص entry في `pinctrl-names` |
| GPIO و pinmux conflict | `request already granted for this GPIO` | نفس الـ pin متطلوب من الاتنين |
| HOG pins مش بتتطبق | `pinctrl: failed to hog default state` | مشكلة في `hog_default` في `pinctrl_dev` |

---

#### 5. Device Tree Debugging

**تحقق إن الـ DT بيطابق الـ hardware:**

```bash
# شوف الـ compiled DT
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A 20 "pinctrl"

# أو استخدم fdtdump
fdtdump /boot/dtb/bcm2837-rpi-3-b.dtb 2>/dev/null | grep -A 10 "uart0_pins"

# تحقق من الـ pinctrl phandle references
cat /sys/firmware/devicetree/base/soc/serial@7e201000/pinctrl-0 | xxd
```

**مثال DT صح لـ UART:**

```dts
uart0_pins: uart0 {
    brcm,pins = <14 15>;          /* يطابق pin numbers في الـ driver */
    brcm,function = <4>;           /* ALT0 */
    brcm,pull = <0 0>;             /* no pull */
};

&uart0 {
    pinctrl-names = "default";    /* يطابق pinctrl_state->name */
    pinctrl-0 = <&uart0_pins>;
    status = "okay";
};
```

**تحقق من الـ DT parsing نجح:**

```bash
# لو الـ dt_node_to_map() اشتغل صح، هتلاقي entries في الـ mapping table
cat /sys/kernel/debug/pinctrl/pinctrl-bcm2835/pinmux-pins | grep uart

# تحقق من الـ dt_maps list في pinctrl struct (من kernel logs عند boot)
dmesg | grep -i pinctrl | head -30
```

---

### Practical Commands — جاهزة للنسخ

#### فحص سريع لحالة الـ pinctrl

```bash
#!/bin/bash
# pinctrl-debug.sh — فحص شامل وسريع

PCTRL_DIR="/sys/kernel/debug/pinctrl"

echo "=== Available Pin Controllers ==="
ls "$PCTRL_DIR/"

echo ""
echo "=== Picking first controller ==="
CTRL=$(ls "$PCTRL_DIR/" | head -1)
echo "Controller: $CTRL"

echo ""
echo "=== Pins ==="
cat "$PCTRL_DIR/$CTRL/pins"

echo ""
echo "=== Pin Groups ==="
cat "$PCTRL_DIR/$CTRL/pingroups" 2>/dev/null || echo "No generic groups"

echo ""
echo "=== Pinmux Status ==="
cat "$PCTRL_DIR/$CTRL/pinmux-pins" 2>/dev/null || echo "Pinmux not enabled"

echo ""
echo "=== GPIO Ranges ==="
cat "$PCTRL_DIR/$CTRL/gpio-ranges" 2>/dev/null || echo "No GPIO ranges"
```

#### تفعيل Tracing لـ pinctrl

```bash
#!/bin/bash
# enable-pinctrl-trace.sh

TRACE_DIR="/sys/kernel/debug/tracing"

# صفّر الـ trace buffer
echo > "$TRACE_DIR/trace"

# فعّل pinctrl events
echo 1 > "$TRACE_DIR/events/pinctrl/enable" 2>/dev/null || \
    echo "No pinctrl tracepoints — تحقق من CONFIG_PINCTRL"

# ابدأ التسجيل
echo 1 > "$TRACE_DIR/tracing_on"

echo "Tracing enabled. Perform your operation now."
echo "Press ENTER to stop and show results..."
read

echo 0 > "$TRACE_DIR/tracing_on"
cat "$TRACE_DIR/trace" | grep pinctrl
```

#### فحص pin conflict

```bash
#!/bin/bash
# find-pin-conflict.sh <pin_number>
PIN=$1
PCTRL_DIR="/sys/kernel/debug/pinctrl"

for ctrl in "$PCTRL_DIR"/*/; do
    name=$(basename "$ctrl")
    result=$(grep "pin $PIN " "$ctrl/pinmux-pins" 2>/dev/null)
    if [ -n "$result" ]; then
        echo "Controller: $name"
        echo "$result"
        # لو فيه اتنين owners = conflict
        owner_count=$(echo "$result" | grep -v UNCLAIMED | wc -l)
        [ "$owner_count" -gt 1 ] && echo "WARNING: Multiple owners detected!"
    fi
done
```

#### مقارنة الـ DT state مع الـ kernel state

```bash
#!/bin/bash
# compare-dt-kernel.sh

echo "=== DT pinctrl nodes ==="
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    grep -E "pinctrl-[0-9]|pinctrl-names" | head -20

echo ""
echo "=== Kernel registered states ==="
for ctrl in /sys/kernel/debug/pinctrl/*/; do
    echo "--- $(basename $ctrl) ---"
    cat "$ctrl/pinmux-functions" 2>/dev/null | head -10
done
```

**مثال output من `pinmux-pins` وإزاي تفسّره:**

```
pin 14 (GPIO14): uart0 (GPIO UNCLAIMED) function uart0 group uart0_grp
                 ^^^^^   ^^^^^^^^^^^^    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                 owner   gpio_owner=NULL  الـ function والـ group المتطبقين
                         (مش محجوز من gpio subsystem)

pin 17 (GPIO17): UNCLAIMED
                 ^^^^^^^^^
                 mux_usecount==0, mux_owner==NULL, gpio_owner==NULL
                 الـ pin حر تماماً

pin 8 (GPIO8): spi0 (GPIO UNCLAIMED) function spi0 group spi0_grp
               ^^^^
               مين hold الـ pin — الاسم جاي من pinctrl->dev->name
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: UART مش شغال على بورد RK3562 — Industrial Gateway

#### العنوان
**الـ UART2 مش بيشتغل بعد boot على industrial gateway بـ RK3562**

#### السياق
شركة بتعمل industrial gateway بـ RK3562 للاتصال بـ Modbus RTU عن طريق RS-485. البورد بيشتغل تمام في بداية المشروع، لكن بعد تحديث kernel من 6.1 لـ 6.6، الـ UART2 بطّل يشتغل خالص. الـ `/dev/ttyS2` موجود بس مفيش data بتيجي.

#### المشكلة
الـ driver بتاع RS-485 بيطلب الـ UART لكن بيفضل في state "idle" — مفيش مشكلة في الـ hardware، المشكلة في الـ pinctrl core مش بيطبّق الـ `default` state صح.

#### التحليل
الـ `pinctrl_dev` بتاع RK3562 عنده `pin_desc_tree` — radix tree فيه كل الـ pins. لما الـ UART driver بيعمل `pinctrl_get()` بيرجع `struct pinctrl` جديد. بعدين بيعمل `pinctrl_lookup_state()` بالاسم `"default"`. الـ `struct pinctrl_state` المطلوب فيه list من `struct pinctrl_setting`، كل setting فيها `type = PIN_MAP_TYPE_MUX_GROUP` وفيها `struct pinctrl_setting_mux` بـ `group` و`func`.

المشكلة: الـ `pinctrl_maps` list — المتغير العام في core.h:
```c
extern struct list_head pinctrl_maps;
```
الـ DT mapping للـ UART2 group مش موجود في الـ list دي. السبب إن الـ DT node بتاع `uart2` فيه `pinctrl-0` بس من غير `pinctrl-names` — فالـ core مش عارف يربطه بالاسم `"default"`.

لما بتعمل:
```bash
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinmux-pins
```
بتلاقي UART2 pins في state `"gpio"` مش `"uart"`.

#### الحل
في الـ DTS:

```c
/* قبل الإصلاح — المشكلة هنا */
&uart2 {
    status = "okay";
    pinctrl-0 = <&uart2m0_xfer>;
    /* pinctrl-names ناقص! */
};

/* بعد الإصلاح */
&uart2 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&uart2m0_xfer>;
};
```

بعد الإصلاح، الـ core بيعمل `pinctrl_register_mappings()` صح، وبيضيف `struct pinctrl_maps` للـ `pinctrl_maps` list. الـ `for_each_pin_map` macro بيلاقي الـ mapping وبيربطها بالـ `pinctrl_state` صح.

#### الدرس المستفاد
الـ `pinctrl-names` مش optional — من غيرها الـ core مش بيربط الـ `pinctrl-0` handle بأي state معروف. الـ `struct pinctrl_state` بتاعة الـ device بتتعمل بس لو الاسم موجود في الـ DT.

---

### السيناريو الثاني: SPI Flash مش بيشتغل على STM32MP1 — IoT Sensor

#### العنوان
**الـ SPI1 conflict مع GPIO على STM32MP1 في IoT sensor node**

#### السياق
بورد IoT بيقرأ بيانات من SPI NOR flash. أحياناً بعد resume من suspend، الـ flash بيبقى unresponsive. المهندس بيشك في الـ power management لكن المشكلة في الـ pin ownership conflict.

#### المشكلة
الـ GPIO driver طلب pin معين باستخدام `pinctrl_gpio_request()`. نفس الـ pin مطلوب من الـ SPI driver في الـ `default` state. الـ `mux_usecount` في `struct pin_desc` اتعمل corrupted بعد resume.

#### التحليل
الـ `struct pin_desc` في core.h عنده:

```c
#ifdef CONFIG_PINMUX
    unsigned int mux_usecount;
    const char *mux_owner;
    const struct pinctrl_setting_mux *mux_setting;
    const char *gpio_owner;
    struct mutex mux_lock;
#endif
```

اللي بيحصل:
1. قبل suspend، SPI بياخد الـ pin — `mux_usecount = 1`، `mux_owner = "spi1"`
2. أثناء suspend، الـ system بيعمل `pinctrl_force_sleep()` على الـ `pinctrl_dev`
3. بعد resume، SPI driver بيعمل `pinctrl_select_state()` للـ `default` state
4. في نفس الوقت، GPIO subsystem بيحاول يعمل `pinctrl_gpio_request()` لنفس الـ pin
5. الـ `mux_lock` mutex مش محتاط منه صح في سيناريو الـ resume race

نتيجة: `gpio_owner` و`mux_owner` بيبقوا set في نفس الوقت — والـ core مش بيسمح بده طبيعياً لكن الـ race condition بيعدّيه.

للتشخيص:
```bash
# قبل suspend
cat /sys/kernel/debug/pinctrl/pinctrl-soc:pin-controller@50002000/pinmux-pins | grep PA5

# بعد resume مباشرة
cat /sys/kernel/debug/pinctrl/pinctrl-soc:pin-controller@50002000/pinmux-pins | grep PA5
```

#### الحل
المشكلة في الـ DT — الـ GPIO hog بياخد الـ pin قبل الـ SPI:

```c
/* إصلاح الـ DTS — نشيل الـ GPIO hog من الـ SPI pins */
spi1_pins: spi1-pins {
    pins = "PA5", "PA6", "PA7";
    function = "spi1";
    /* لا hog هنا */
};

/* الـ GPIO المحتاج pin تاني مش نفس الـ SPI */
&gpioa {
    cs-hog {
        gpio-hog;
        gpios = <8 GPIO_ACTIVE_LOW>; /* PA8 مش PA5 */
        output-low;
        line-name = "spi-cs";
    };
};
```

كمان ممكن نستخدم `pinctrl_force_default()` في الـ resume path:
```c
/* في الـ driver's resume callback */
ret = pinctrl_force_default(pctldev);
```

#### الدرس المستفاد
الـ `mux_usecount` و`gpio_owner` في `struct pin_desc` بيمنعوا الـ conflict لكن لازم الـ DT يكون صح من الأول. أي pin ممكن يبقى `mux_owner` أو `gpio_owner` — مش الاتنين مع بعض.

---

### السيناريو الثالث: I2C Hang على i.MX8 — Automotive ECU

#### العنوان
**الـ I2C3 بيـ hang على i.MX8MQ في automotive ECU — المشكلة في الـ pin config**

#### السياق
ECU بيتحكم في HVAC system باستخدام i.MX8MQ. الـ I2C3 بيكلم sensors بس بعد cold boot في درجة حرارة منخفضة، الـ bus بيـ hang. الـ hardware team قالوا الـ pull-up resistors صح. المشكلة في الـ drive strength والـ slew rate config.

#### المشكلة
الـ `struct pinctrl_setting_configs` للـ I2C3 pins مش بتطبّق الـ configs الصح — الـ `num_configs` بيتحسب غلط في الـ mapping table.

#### التحليل
الـ `struct pinctrl_setting` بتجمع الـ mux والـ configs:

```c
struct pinctrl_setting {
    struct list_head node;
    enum pinctrl_map_type type;        /* PIN_MAP_TYPE_CONFIGS_PIN */
    struct pinctrl_dev *pctldev;
    const char *dev_name;
    union {
        struct pinctrl_setting_mux mux;
        struct pinctrl_setting_configs configs;
    } data;
};
```

لما الـ core بيطبّق settings للـ I2C3، بيمشي على الـ `settings` list في `struct pinctrl_state`. كل setting من type `PIN_MAP_TYPE_CONFIGS_PIN` بتشتغل على `struct pinctrl_setting_configs`:

```c
struct pinctrl_setting_configs {
    unsigned int group_or_pin;  /* رقم الـ pin */
    unsigned long *configs;     /* array of config values */
    unsigned int num_configs;   /* المفروض 2: drive strength + slew rate */
};
```

المشكلة: المهندس استخدم `PIN_MAP_CONFIGS_PIN` macro بس وضع الـ configs array كـ local variable في function — فالـ pointer بيبقى dangling بعد ما الـ function تخلص.

كمان:
```bash
# نشوف الـ configs المطبّقة
cat /sys/kernel/debug/pinctrl/pinctrl-30330000.iomuxc/pinconf-pins | grep I2C3
```

#### الحل
في الـ board file أو الـ DTS:

```c
/* المشكلة — configs array محلية */
static int __init board_init(void)
{
    unsigned long bad_configs[] = {
        IMX_PAD_CTL_DSE_6,
        IMX_PAD_CTL_SPEED_MED,
    };
    /* bad_configs هتتشال من الـ stack! */
    PIN_MAP_CONFIGS_PIN("2-0048", "default", "30330000.iomuxc",
                        "I2C3_SCL", bad_configs); /* خطأ */
}

/* الحل — configs array static */
static const unsigned long i2c3_scl_configs[] = {
    IMX_PAD_CTL_DSE_6,
    IMX_PAD_CTL_SPEED_MED,
    IMX_PAD_CTL_ODE,        /* open-drain للـ I2C */
};

/* أو أحسن — استخدم DTS مباشرة */
```

في DTS (الحل الأفضل):
```c
&i2c3 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c3>;
    status = "okay";
};

pinctrl_i2c3: i2c3grp {
    fsl,pins = <
        MX8MQ_IOMUXC_I2C3_SCL_I2C3_SCL  0x40000063
        MX8MQ_IOMUXC_I2C3_SDA_I2C3_SDA  0x40000063
    >;
    /* 0x40000063 = ODE + DSE6 + SPEED_MED */
};
```

#### الدرس المستفاد
الـ `configs` pointer في `struct pinctrl_setting_configs` لازم يشاور على memory تعيش طول عمر الـ driver. Static arrays في DTS أضمن من C code.

---

### السيناريو الرابع: HDMI مش بيشتغل على Allwinner H616 — Android TV Box

#### العنوان
**الـ HDMI output مفيش على Android TV Box بـ Allwinner H616 — missing pinctrl group**

#### السياق
منتج Android TV box بيستخدم Allwinner H616. الـ HDMI مشغّل على بورد reference لكن على الـ custom board مفيش output. الـ display subsystem بيشتغل، الـ framebuffer موجود، بس الشاشة سودا.

#### المشكلة
الـ HDMI DDC (I2C للـ EDID) مش شغال لأن الـ pinctrl group بتاعه مش اتضاف للـ `pin_group_tree` في الـ `pinctrl_dev`.

#### التحليل
الـ `pinctrl_dev` عنده (مع `CONFIG_GENERIC_PINCTRL_GROUPS`):

```c
struct pinctrl_dev {
    ...
#ifdef CONFIG_GENERIC_PINCTRL_GROUPS
    struct radix_tree_root pin_group_tree;
    unsigned int num_groups;
#endif
    ...
};
```

الـ `struct group_desc` المفروض يتسجّل للـ HDMI DDC group:

```c
struct group_desc {
    struct pingroup grp;  /* name + pins array */
    void *data;           /* driver-specific */
};
```

لما بنعمل:
```bash
cat /sys/kernel/debug/pinctrl/pio/pingroups
```

بنلاقي groups كتير لكن `hdmi_ddc` مش موجود.

السبب: في الـ Allwinner H616 pinctrl driver، الـ `pinctrl_generic_add_group()` اتنادى بس لـ groups المعرّفة في الـ static array. الـ HDMI DDC group اتنسي يتضاف. الـ `num_groups` بيكون أقل من المطلوب.

للتأكيد:
```bash
# نشوف عدد الـ groups
cat /sys/kernel/debug/pinctrl/pio/pingroups | grep "group:" | wc -l

# نشوف الـ pins المتاحة
cat /sys/kernel/debug/pinctrl/pio/pins | grep -i hdmi
```

الـ HDMI DDC pins موجودة في `pin_desc_tree` بس مش مجمّعة في group.

#### الحل
في الـ Allwinner H616 pinctrl driver:

```c
/* في pinctrl-sun50i-h616.c */
static const unsigned int hdmi_ddc_pins[] = {
    SUNXI_PIN(SUNXI_PA, 0),   /* HDMI-DDC-SCL */
    SUNXI_PIN(SUNXI_PA, 1),   /* HDMI-DDC-SDA */
};

/* إضافة الـ group الناقص */
static const struct sunxi_pinctrl_group sun50i_h616_groups[] = {
    ...
    SUNXI_PIN_GROUP("hdmi_ddc", hdmi_ddc_pins),  /* السطر الناقص */
    ...
};
```

بعد الإضافة، `pinctrl_generic_add_group()` بتضيف `group_desc` للـ `pin_group_tree` وبتزوّد `num_groups`. الـ `pinctrl_get_group_selector()` بيقدر يلاقي `"hdmi_ddc"` ويرجع الـ selector الصح.

في DTS:
```c
&hdmi {
    pinctrl-names = "default";
    pinctrl-0 = <&hdmi_ddc_pins_a>;
    status = "okay";
};
```

#### الدرس المستفاد
الـ `pin_group_tree` في `pinctrl_dev` لازم يشمل كل الـ groups حتى لو الـ feature مش هتستخدم في كل الـ boards. Missing group بيخلي `pinctrl_get_group_selector()` يفشل بـ `-EINVAL` وده بيمنع الـ mux setting من الاتطبيق.

---

### السيناريو الخامس: GPIO Conflict بعد Board Bring-up على AM62x — Custom Board

#### العنوان
**الـ SPI CS مش بيشتغل على AM62x custom board — pin hogging conflict**

#### السياق
فريق hardware بيعمل bring-up لبورد جديد بـ TI AM62x. الـ SPI0 بيتحكم في ADC خارجي. المهندس شغّل الـ SPI driver لكن `chip_select` مش بيشتغل — الـ CS line مش بتتحرك.

#### المشكلة
الـ pinctrl driver نفسه بيعمل **hog** للـ CS pin في الـ `default` state عشان يبقى في output-high. ده بيتعارض مع الـ SPI driver اللي محتاج يتحكم في نفس الـ pin عن طريق GPIO.

#### التحليل
الـ `pinctrl_dev` فيه:

```c
struct pinctrl_dev {
    ...
    struct pinctrl *p;             /* result of pinctrl_get() for this device */
    struct pinctrl_state *hog_default;  /* hogged pins in default state */
    struct pinctrl_state *hog_sleep;    /* hogged pins in sleep state */
    ...
};
```

لما الـ pinctrl controller driver بيتسجّل، الـ core بيشوف لو في maps بتاعت نفس الـ controller device في الـ `pinctrl_maps` list. ده الـ hogging mechanism. لو الـ map entry's `dev_name` == `ctrl_dev_name`، الـ pin بيتعمله hog.

```c
/* في الـ DTS — المشكلة */
&main_pmx0 {
    spi0_cs_hog: spi0-cs-hog {
        pinctrl-single,pins = <
            AM62X_IOPAD(0x01c4, PIN_OUTPUT_PULLUP, 0)  /* SPI0_CS0 */
        >;
    };
    /* الـ hog بيحجز الـ pin لـ pinctrl controller نفسه */
};

/* ده بيعمل hog_default state للـ pinctrl_dev */
```

نتيجة: لما الـ SPI driver بيحاول يعمل `pinctrl_gpio_request()` للـ CS pin، `pin_desc->gpio_owner` بيلاقي `mux_owner` مش NULL (الـ pinctrl controller حاجز الـ pin). الـ core بيرفض الـ request.

```bash
# للتشخيص
cat /sys/kernel/debug/pinctrl/main_pmx0/pinmux-pins | grep 0x01c4
# هيظهر: pin 73 (SPI0_CS0): UNCLAIMED أو hogged by pinctrl
```

للتأكيد من الـ hog:
```bash
cat /sys/kernel/debug/pinctrl/main_pmx0/pinmux-select
```

#### الحل
إزالة الـ hog من الـ pinctrl node وتحريك الـ config للـ SPI node نفسه:

```c
/* إزالة الـ hog — مش محتاجينه */
&main_pmx0 {
    spi0_pins_default: spi0-default-pins {
        pinctrl-single,pins = <
            AM62X_IOPAD(0x01b4, PIN_INPUT, 0)          /* SPI0_CLK */
            AM62X_IOPAD(0x01b8, PIN_OUTPUT, 0)          /* SPI0_D0 = MOSI */
            AM62X_IOPAD(0x01bc, PIN_INPUT, 0)           /* SPI0_D1 = MISO */
            AM62X_IOPAD(0x01c4, PIN_OUTPUT_PULLUP, 0)  /* SPI0_CS0 */
        >;
    };
};

/* الـ SPI node بياخد الـ pin في default state بتاعه */
&spi0 {
    pinctrl-names = "default";
    pinctrl-0 = <&spi0_pins_default>;
    status = "okay";

    adc@0 {
        compatible = "ti,ads8688";
        reg = <0>;
        spi-max-frequency = <8000000>;
    };
};
```

لو المهندس عايز الـ CS يبدأ high قبل الـ SPI driver يشتغل، الحل الصح هو `gpio-hog` في الـ GPIO node مش في الـ pinctrl node:

```c
&main_gpio0 {
    spi0-cs-hog {
        gpio-hog;
        gpios = <73 GPIO_ACTIVE_LOW>;  /* pin 73 = SPI0_CS0 */
        output-high;
        line-name = "spi0-cs-hog";
    };
};
```

#### الدرس المستفاد
الـ `hog_default` في `pinctrl_dev` مخصوص لـ pins اللي الـ pinctrl controller نفسه محتاجها — مش لـ client devices. لو عملت hog لـ pin المفروض يبقى تحت تحكم driver تاني، الـ `mux_owner` في `pin_desc` هيتعبّيه بالـ controller device وده بيمنع أي client تاني من استخدامه.
## Phase 7: مصادر ومراجع

### مصادر رسمية — Kernel Documentation

| المصدر | الرابط |
|--------|--------|
| **Official pinctrl Documentation** (kernel.org) | [docs.kernel.org/driver-api/pin-control.html](https://docs.kernel.org/driver-api/pin-control.html) |
| **pinctrl.txt** (v4.14 stable API reference) | [kernel.org/doc/html/v4.14/driver-api/pinctl.html](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html) |
| **Documentation/pinctrl.txt** (Android kernel mirror) | [android.googlesource.com](https://android.googlesource.com/kernel/common/+/bcmdhd-3.10/Documentation/pinctrl.txt) |

الـ `Documentation/pinctrl.txt` هو نقطة البداية الصح لأي حد بدأ يفهم الـ subsystem ده — بيشرح الـ `pinctrl_dev`، الـ `pin_desc`، الـ mapping table، والـ state machine كلها.

---

### مقالات LWN.net

دي أهم المقالات على [lwn.net](https://lwn.net) اللي بتغطي الـ pinctrl subsystem من الأول:

| المقالة | الرابط |
|---------|--------|
| **The pin control subsystem** — المقالة التأسيسية اللي شرحت الـ subsystem لما اتضافت | [lwn.net/Articles/468759](https://lwn.net/Articles/468759/) |
| **pin controller subsystem v7** — نسخة الـ RFC اللي دخلت الـ mainline | [lwn.net/Articles/459190](https://lwn.net/Articles/459190/) |
| **Documentation/pinctrl.txt** — النسخة الأولى من الـ documentation | [lwn.net/Articles/465077](https://lwn.net/Articles/465077/) |
| **pinctrl: add a pin config interface** — إضافة الـ `pinconf` ops | [lwn.net/Articles/471826](https://lwn.net/Articles/471826/) |
| **pinctrl: Add generic pinctrl-simple driver** — الـ `pinctrl-single` العام لـ OMAP2+ | [lwn.net/Articles/496075](https://lwn.net/Articles/496075/) |
| **pinctrl: introduce GPIO pin function category** — تطور العلاقة بين pinctrl وـ GPIO | [lwn.net/Articles/1031226](https://lwn.net/Articles/1031226/) |
| **pinctrl: Add new pinctrl/GPIO driver** — مثال عملي لـ driver جديد | [lwn.net/Articles/803863](https://lwn.net/Articles/803863/) |

**ابدأ بـ `468759` و`459190`** — بيديك سياق تاريخي مهم عن ليه الـ subsystem ده اتعمل من الأول.

---

### مستودع Linus Walleij الرسمي

الـ maintainer الأصلي بيحتفظ بـ development tree منفصلة:

```
pub/scm/linux/kernel/git/linusw/linux-pinctrl
https://kernel.googlesource.com/pub/scm/linux/kernel/git/linusw/linux-pinctrl/
```

الـ patches بتمشي من هنا لـ `linux-next` وبعدين للـ mainline عن طريق Linus Torvalds.

---

### مقالات elinux.org

| المصدر | الرابط |
|--------|--------|
| **Pin Control & GPIO update** (PDF عرض Linus Walleij) | [elinux.org/images/c/cd/Pincontrol-gpio-update.pdf](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) |
| **EBC Device Trees** — أمثلة عملية على `pinctrl-single` في device tree | [elinux.org/EBC_Device_Trees](https://elinux.org/EBC_Device_Trees) |
| **Tests: i2c-demux-pinctrl** | [elinux.org/Tests:i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) |

الـ PDF من Linus Walleij نفسه ده مهم جداً — بيشرح العلاقة بين pinctrl وـ GPIO وازاي الـ `gpio_ranges` اتضافت.

---

### kernelnewbies.org — تتبع التغييرات عبر الإصدارات

الـ pinctrl بيتذكر في كل إصدار تقريباً. الإصدارات دي بها تغييرات مهمة:

| الإصدار | الرابط |
|---------|--------|
| **Linux 6.1** — تغييرات في الـ generic groups API | [kernelnewbies.org/Linux_6.1](https://kernelnewbies.org/Linux_6.1) |
| **Linux 6.8** — تحسينات pinctrl و GPIO | [kernelnewbies.org/Linux_6.8](https://kernelnewbies.org/Linux_6.8) |
| **Linux 6.11** — drivers جديدة وتغييرات في الـ core | [kernelnewbies.org/Linux_6.11](https://kernelnewbies.org/Linux_6.11) |
| **Linux 6.13** | [kernelnewbies.org/Linux_6.13](https://kernelnewbies.org/Linux_6.13) |
| **Linux 6.17** | [kernelnewbies.org/Linux_6.17](https://kernelnewbies.org/Linux_6.17) |

---

### embedded.com

| المقالة | الرابط |
|---------|--------|
| **Linux device driver development: The pin control subsystem** — مقالة شاملة للـ embedded developers | [embedded.com/linux-device-driver-development-the-pin-control-subsystem](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/) |

---

### Commits مهمة في الـ Kernel

الـ commits دي مرجعية لفهم تطور `core.h` وهياكل الـ `pinctrl_dev` و`pin_desc`:

```bash
# إيجاد أول commit للـ pinctrl subsystem
git log --oneline --follow drivers/pinctrl/core.c | tail -20

# مشاهدة تاريخ core.h
git log --oneline --follow drivers/pinctrl/core.h

# تغييرات مرتبطة بـ CONFIG_GENERIC_PINCTRL_GROUPS
git log --oneline --all-match --grep="GENERIC_PINCTRL_GROUPS" -- drivers/pinctrl/
```

أهم commit هو اللي أضاف `CONFIG_GENERIC_PINCTRL_GROUPS` وحول الـ `group_desc` من كل driver لـ generic API في الـ core.

---

### مناقشات Mailing List

| المناقشة | الرابط |
|----------|--------|
| **PATCH v5: introduce GPIO pin function category** | [mail-archive.com](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg10587.html) |
| **PATCH v7: pinctrl support for AAEON UP board FPGA** | [mail-archive.com](https://www.mail-archive.com/linux-hardening@vger.kernel.org/msg09695.html) |
| **lkml.org — أرشيف كامل للـ pinctrl patches** | [lkml.org](https://lkml.org/) — ابحث بـ `pinctrl` |
| **lore.kernel.org** — الأرشيف الرسمي الحديث | [lore.kernel.org/linux-gpio/](https://lore.kernel.org/linux-gpio/) |

الـ mailing list بتاع الـ pinctrl هو `linux-gpio@vger.kernel.org` — نفس القائمة بتاعة GPIO.

---

### كتب مرجعية

#### Linux Device Drivers (LDD3) — Rubini, Corbet, Kroah-Hartman
- الكتاب مش بيغطي pinctrl بشكل مباشر (قديم قبل الـ subsystem)، لكن:
  - **Chapter 9** — Communicating with Hardware (فهم الـ MMIO والـ platform devices)
  - **Chapter 14** — The Linux Device Model (فهم `struct device` اللي بتستخدمها `pinctrl_dev`)
- متاح مجاناً: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 17** — Devices and Modules: بيشرح الـ device model الكامل
- **Chapter 11** — Timers and Time Management: مش مرتبط مباشراً لكن مهم لفهم الـ kernel internals
- الـ `radix_tree` المستخدمة في `pin_desc_tree` متشرحة في سياق الـ data structures

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **Chapter 15** — Kernel Initialization: بيشرح bootstrap لـ platform devices
- **Chapter 16** — Kernel Porting Guide: بيتكلم عن الـ board-level pin setup اللي pinctrl جه يحله

#### Linux Device Driver Development — John Madieu (2nd Edition, Packt)
- الكتاب ده الأحدث والأشمل للـ pinctrl — بيغطي:
  - تسجيل الـ `pinctrl_dev`
  - كتابة الـ `pinctrl_ops`, `pinmux_ops`, `pinconf_ops`
  - الـ Device Tree bindings للـ pinctrl

---

### مسارات الـ Source Code المهمة

```
drivers/pinctrl/core.c          # تنفيذ الـ core API
drivers/pinctrl/core.h          # الـ private structs (الملف ده)
drivers/pinctrl/pinmux.c        # تنفيذ الـ mux operations
drivers/pinctrl/pinconf.c       # تنفيذ الـ config operations
drivers/pinctrl/pinctrl-utils.c # helper functions
drivers/pinctrl/devicetree.c    # DT parsing للـ pinctrl maps
include/linux/pinctrl/pinctrl.h # الـ public API للـ drivers
include/linux/pinctrl/machine.h # الـ pinctrl_map و mapping macros
include/linux/pinctrl/consumer.h # الـ API للـ consumer drivers
include/linux/pinctrl/pinmux.h  # الـ pinmux_ops interface
include/linux/pinctrl/pinconf.h # الـ pinconf_ops interface
```

---

### Search Terms للبحث عن معلومات أكثر

```
linux kernel pinctrl subsystem internals
pinctrl_dev pin_desc radix tree
pinctrl mapping table device tree
linux pinmux ops group selector
pinctrl gpio ranges overlap
CONFIG_GENERIC_PINCTRL_GROUPS kernel
pinctrl state machine default sleep idle
linus walleij pinctrl maintainer
linux pinctrl consumer API devm_pinctrl_get
pinctrl hog device tree self
```

للبحث في LKML مباشرة:
```
https://lore.kernel.org/linux-gpio/?q=pinctrl+core
```
## Phase 8: Writing simple module

### الفكرة

**`pin_get_from_name()`** هي function مصدّرة في pinctrl subsystem بتاخد اسم pin وترجع رقمه (ID) من الـ `pinctrl_dev`. هنعمل **kprobe** عليها عشان نشوف في runtime أي device بتدور على أي pin بالاسم، ده مفيد جداً لتتبع مشاكل الـ pin configuration على embedded boards.

---

### الـ Module كامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on pin_get_from_name() — trace pinctrl pin lookup by name
 *
 * Works on any kernel with CONFIG_PINCTRL=y and CONFIG_KPROBES=y.
 * Tested pattern: whenever a driver calls pin_get_from_name(), we log
 * the controller device name and the pin name being searched.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit        */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe, ...     */
#include <linux/printk.h>      /* pr_info                                 */
#include <linux/device.h>      /* struct device, dev_name()               */

/*
 * Forward-declare enough of pinctrl_dev so we can read its ->dev pointer.
 * We only need the first field layout up to ->dev; using an opaque
 * partial struct avoids pulling in the private core.h header.
 *
 * Layout from drivers/pinctrl/core.h:
 *   struct pinctrl_dev {
 *       struct list_head  node;      // 2 pointers
 *       const struct pinctrl_desc *desc;
 *       struct radix_tree_root pin_desc_tree;
 *       ...
 *       struct device *dev;          // NOT at offset 0
 *   };
 *
 * To stay safe we just use the real exported helpers — pin_get_from_name is
 * the probe target, and the args are passed in registers (x86-64 SysV ABI:
 * rdi = pctldev, rsi = name).
 */

/* --- pre-handler: fires just before pin_get_from_name executes --- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
	/*
	 * x86-64: first arg (pctldev) is in regs->di,
	 *         second arg (name)   is in regs->si.
	 * ARM64:  first arg in regs->regs[0], second in regs->regs[1].
	 * We cast to void * to avoid dereferencing the opaque struct.
	 */
#ifdef CONFIG_X86_64
	void        *pctldev = (void *)regs->di;
	const char  *pin_name = (const char *)regs->si;
#elif defined(CONFIG_ARM64)
	void        *pctldev = (void *)regs->regs[0];
	const char  *pin_name = (const char *)regs->regs[1];
#else
	void        *pctldev  = NULL;
	const char  *pin_name = "(arch unsupported)";
#endif

	/*
	 * struct pinctrl_dev starts with list_head (16 bytes on 64-bit),
	 * then a pointer to pinctrl_desc (8 bytes), then radix_tree_root
	 * (8 bytes), so ->dev is NOT trivially at a fixed small offset.
	 *
	 * Safest approach: just print the pctldev address + pin name.
	 * A production module would import core.h or use debugfs instead.
	 */
	pr_info("pinctrl_probe: pin_get_from_name() called — "
		"pctldev=%p  pin_name=\"%s\"\n",
		pctldev,
		pin_name ? pin_name : "(null)");

	return 0; /* 0 = continue normal execution */
}

/* --- post-handler: fires after pin_get_from_name returns --- */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
			 unsigned long flags)
{
	/*
	 * Return value: x86-64 → rax, ARM64 → regs[0] after the call.
	 * A negative value means "pin not found".
	 */
#ifdef CONFIG_X86_64
	long ret = (long)regs->ax;
#elif defined(CONFIG_ARM64)
	long ret = (long)regs->regs[0];
#else
	long ret = -ENOSYS;
#endif

	if (ret < 0)
		pr_info("pinctrl_probe: pin NOT found (err=%ld)\n", ret);
	else
		pr_info("pinctrl_probe: pin found → pin_id=%ld\n", ret);
}

/* --- kprobe descriptor --- */
static struct kprobe kp = {
	.symbol_name    = "pin_get_from_name", /* symbol to probe              */
	.pre_handler    = handler_pre,         /* called before the function   */
	.post_handler   = handler_post,        /* called after it returns      */
};

/* ------------------------------------------------------------------ */
/*                          module_init                                 */
/* ------------------------------------------------------------------ */
static int __init pinctrl_probe_init(void)
{
	int ret;

	/* register_kprobe patches the first byte of the target symbol
	 * with a breakpoint instruction and stores our handlers.        */
	ret = register_kprobe(&kp);
	if (ret < 0) {
		pr_err("pinctrl_probe: register_kprobe failed: %d\n", ret);
		return ret;
	}

	pr_info("pinctrl_probe: kprobe planted at %s (%p)\n",
		kp.symbol_name, kp.addr);
	return 0;
}

/* ------------------------------------------------------------------ */
/*                          module_exit                                 */
/* ------------------------------------------------------------------ */
static void __exit pinctrl_probe_exit(void)
{
	/* unregister_kprobe restores the original instruction byte and
	 * ensures no handler is running before freeing resources.       */
	unregister_kprobe(&kp);
	pr_info("pinctrl_probe: kprobe removed from %s\n", kp.symbol_name);
}

module_init(pinctrl_probe_init);
module_exit(pinctrl_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on pin_get_from_name() to trace pinctrl pin lookups");
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

### شرح كل جزء

#### الـ includes

| Header | ليه موجود |
|---|---|
| `linux/module.h` | `module_init` / `module_exit` / `MODULE_LICENSE` |
| `linux/kprobes.h` | `struct kprobe` و `register_kprobe` |
| `linux/printk.h` | `pr_info` / `pr_err` |
| `linux/device.h` | `struct device` لو احتجنا `dev_name()` لاحقاً |

الـ includes دي بتجيب كل الـ kernel APIs اللي المودول محتاجها بدون ما نجيب private headers زي `core.h` اللي مش مفروض تعتمد عليها من خارج الـ subsystem.

#### الـ `handler_pre`

الـ pre-handler بيتشغل **قبل** ما الـ CPU ينفذ أول instruction في `pin_get_from_name`. بنقرأ الـ arguments من الـ registers مباشرةً (calling convention بتحدد مكانهم)، وبنطبع عنوان الـ `pinctrl_dev` واسم الـ pin اللي الكود بيدور عليه — ده بيخلي المطوّر يشوف في `dmesg` كل مرة driver بيسأل عن pin بالاسم.

#### الـ `handler_post`

الـ post-handler بيتشغل **بعد** ما الـ function ترجع وقبل ما المتصل يشوف النتيجة. بنقرأ الـ return value من register الـ return (rax/x0)؛ لو سالب معناه الـ pin مش موجود، ولو موجب هو رقم الـ pin ID — المعلومة دي مهمة لـ debugging مشاكل الـ pin mapping في early boot.

#### الـ `struct kprobe`

الـ `.symbol_name` بيقول للـ kernel على أي function نحط الـ breakpoint بالاسم بدل العنوان الـ hardcoded، وده بيخلي المودول portable عبر kernel versions مختلفة طول ما الـ symbol موجود.

#### الـ `module_init`

`register_kprobe` بتعمل **runtime patching**: بتبدل أول byte من الـ function بـ breakpoint instruction، وبتخزن الـ handlers. لو الـ symbol مش موجود أو الـ CONFIG_KPROBES مش مفعّل، الـ registration بترجع error ونطلع clean.

#### الـ `module_exit`

`unregister_kprobe` ضرورية عشان ترجع الـ original instruction وتضمن إن محدش لسه شغال في الـ handler وقت الـ cleanup — من غيرها هيبقى عندنا use-after-free في أي call بتيجي بعد rmmod.

---

### تشغيل وقراءة النتائج

```bash
# بناء المودول
make

# تحميله
sudo insmod pinctrl_probe.ko

# شوف الـ output (أي pin lookup هيظهر هنا)
sudo dmesg | grep pinctrl_probe

# مثال output على board فيها pinctrl نشط:
# [  42.123456] pinctrl_probe: kprobe planted at pin_get_from_name (ffffffffc0123456)
# [  42.234567] pinctrl_probe: pin_get_from_name() called — pctldev=ffff888... pin_name="uart0_tx"
# [  42.234580] pinctrl_probe: pin found → pin_id=17

# إزالة المودول
sudo rmmod pinctrl_probe
sudo dmesg | tail -3
# [  99.000001] pinctrl_probe: kprobe removed from pin_get_from_name
```

---

### ليه `pin_get_from_name` تحديداً؟

الـ `pinctrl_force_sleep` و `pinctrl_force_default` ممكن تسبب side effects لو اتستدعوا من probe. في المقابل، **`pin_get_from_name`** هي read-only lookup: بتمشي على الـ `pin_desc_tree` وترجع ID — مفيش state يتغير، فالـ kprobe عليها آمن تماماً ومش هيأثر على الـ system.
