## Phase 1: الصورة الكبيرة ببساطة

### ايه ده ببساطة؟

تخيل إن عندك لوحة مفاتيح (keyboard) فيها 100 زرار. كل زرار (pin) ممكن يشتغل بأكتر من طريقة واحدة:
- ممكن يبعت/يستقبل إشارة GPIO عادية.
- ممكن يكون جزء من UART (سيريال).
- ممكن يكون جزء من SPI أو I2C أو PWM.

مشكلة: الزرار دا واحد بس — مينفعش حاجتين تمسكوه في نفس الوقت.

**pinmux** هو المدير اللي بيقرر مين يستخدم إيه، وبيتأكد إن مفيش تعارض.

---

### الـ Subsystem في MAINTAINERS

الملفين دول جزء من:

```
PIN CONTROL SUBSYSTEM
Maintainer: Linus Walleij <linusw@kernel.org>
Mailing List: linux-gpio@vger.kernel.org
Files: drivers/pinctrl/  +  include/linux/pinctrl/
```

---

### ليه الملف ده موجود؟ (الهدف)

**`pinmux.c`** هو الـ core engine اللي بيدير عملية الـ multiplexing للـ pins على مستوى الـ kernel. هدفه:

1. **حجز الـ pin** لدريفر معين (function) وتسجيل الـ owner.
2. **منع التعارض** — لو pin اتحجز لـ SPI، UART مش هيقدر ياخده.
3. **تفعيل الـ mux setting** بالفعل على الهاردوير عن طريق استدعاء `ops->set_mux()`.
4. **دعم GPIO** كـ use-case خاص — لأن GPIO عادةً بيتعامل مع pin منفرد مش group.
5. **Generic helpers** — لو الدريفر مش عايز يكتب boilerplate، يستخدم `pinmux_generic_*`.

---

### التشبيه بالواقع

```
SoC عندك فيه Pin #5

┌─────────────────────────────────┐
│          Pin #5 (Physical)      │
│                                 │
│  Function A: UART_TX  ─────┐    │
│  Function B: SPI_CLK  ─────┤ MUX│──► Hardware Register
│  Function C: GPIO17   ─────┘    │
└─────────────────────────────────┘

pinmux.c بيقول:
  "اللي طلب UART_TX اتسجل كـ owner"
  "لو جه SPI_CLK يطلب نفس الـ pin → REJECTED"
```

---

### الـ Architecture: ازاي الكود متنظم؟

```
Consumer (device driver)
        │
        ▼
  pinctrl core (core.c)
        │
        ▼
  pinmux.c  ◄──────────────── pinmux.h (internal interface)
        │
        ▼
  pinmux_ops (set by hardware driver)
        │
        ▼
  Hardware Register (actual mux switch)
```

**ثلاث طبقات في الملف:**

| الطبقة | الوظيفة |
|---|---|
| Core API | `pin_request()`, `pin_free()` — إدارة الحجز |
| GPIO bridge | `pinmux_request_gpio()`, `pinmux_gpio_direction()` — ربط GPIO بالـ pinmux |
| Generic helpers | `pinmux_generic_add_function()` وأخواتها — لتسهيل كتابة الدريفرات |

---

### المفاهيم الأساسية

**`function`** — الوظيفة اللي الـ pin هيعملها (مثلاً: `"uart0"`, `"spi1"`, `"gpio"`).

**`group`** — مجموعة pins بتعمل الوظيفة دي مع بعض (مثلاً: SPI محتاج CLK + MOSI + MISO + CS = group).

**`mux_owner`** — اسم الدريفر اللي حاجز الـ pin كـ function.

**`gpio_owner`** — اسم الـ GPIO اللي حاجز الـ pin.

**`strict` mode** — لو `true`، الـ pin مش ممكن يتشارك بين GPIO وfunction تاني خالص.

**`mux_usecount`** — reference count — نفس الـ function ممكن تتطلب أكتر من مرة (مثلاً من أكتر من state).

---

### Flow تفعيل الـ Mux Setting

```c
// 1. Map string name → selector number
pinmux_func_name_to_selector(pctldev, "uart0")  // → func=2

// 2. احجز كل pin في الـ group واحد واحد
for each pin in group:
    pin_request(pctldev, pin, dev_name, NULL)
    // → يسجل mux_owner، يزود mux_usecount

// 3. سجّل الـ mux_setting في descriptor كل pin
desc->mux_setting = &setting->data.mux;

// 4. اطلب من الدريفر يكتب على الهاردوير
ops->set_mux(pctldev, func_selector, group_selector)
```

---

### لو حصل error؟ (Rollback)

```c
err_set_mux:
    // امسح mux_setting من كل pin
err_pin_request:
    // حرر كل الـ pins اللي اتحجزوا → pin_free()
```

الكود بيعمل atomic-style rollback — إما كله ينجح أو كله يرجع.

---

### الـ debugfs Interface

لو `CONFIG_DEBUG_FS` شغال، هتلاقي تحت `/sys/kernel/debug/pinctrl/<device>/`:

| الملف | بيعرض إيه |
|---|---|
| `pinmux-functions` | كل function واسمه والـ groups بتاعته |
| `pinmux-pins` | كل pin ومين حاجزه ولأيه وظيفة |
| `pinmux-select` | write-only: تقدر تعمل mux يدوي للتست |

---

### الفرق بين الملفات التلاتة

| الملف | الدور |
|---|---|
| `drivers/pinctrl/pinmux.c` | التنفيذ الفعلي لكل منطق الـ pinmux |
| `drivers/pinctrl/pinmux.h` | **internal** header — الـ interface بين `pinmux.c` وبقية الـ pinctrl core |
| `include/linux/pinctrl/pinmux.h` | **public** header — بيعرّف `struct pinmux_ops` اللي hardware drivers بتنفذها |

---

### الملفات المكونة للـ Subsystem (Core Files)

```
drivers/pinctrl/
├── core.c               ← نقطة الدخول الرئيسية للـ pinctrl subsystem
├── core.h               ← internal structs (pin_desc, pinctrl_dev...)
├── pinmux.c             ← ★ ملفنا: منطق الـ mux كامل
├── pinmux.h             ← internal interface للـ pinmux
├── pinconf.c            ← إدارة إعدادات الـ pin (pull-up, drive strength...)
├── pinconf.h            ← internal interface للـ pinconf
├── devicetree.c         ← ربط الـ pinctrl بالـ Device Tree

include/linux/pinctrl/
├── pinmux.h             ← ★ struct pinmux_ops (driver API)
├── pinctrl.h            ← pinctrl_desc, pinctrl_ops
├── pinconf.h            ← pinconf_ops
├── consumer.h           ← API اللي الـ consumer drivers بتستخدمها
├── machine.h            ← pinctrl_map (board-level mapping)
├── pinctrl-state.h      ← أسماء الـ states (default, sleep, idle...)
├── devinfo.h            ← ربط الـ pinctrl بالـ device lifecycle
```

---

### الملفات ذات الصلة المباشرة

- **`drivers/pinctrl/core.c`** — بيستدعي `pinmux_check_ops()`, `pinmux_enable_setting()` إلخ.
- **`drivers/pinctrl/core.h`** — بيعرّف `struct pin_desc` اللي فيها `mux_owner`, `mux_usecount`, `mux_setting`.
- **`include/linux/pinctrl/machine.h`** — بيعرّف `struct pinctrl_map` و`pinctrl_map_mux` اللي `pinmux_validate_map()` بتشتغل عليها.
- **أي hardware driver** زي `drivers/pinctrl/intel/pinctrl-intel.c` — بينفذ `struct pinmux_ops` ويملي `set_mux()`.
## Phase 2: شرح الـ pinmux Framework

### المشكلة اللي بيحلها

الـ SoC الواحد بيبقى فيه مئات الـ pins، كل pin ممكن يتوصل بأكتر من peripheral واحد (UART، SPI، GPIO، I2C، ...). الـ hardware بيختار مين بيشتغل على الـ pin ده عن طريق **multiplexer** داخلي. المشكلة:

- كل driver كان بيعمل الـ mux configuration بنفسه → **chaos و conflicts**.
- الـ GPIO subsystem محتاجة تعرف لو الـ pin اتاخد من peripheral تاني.
- محدش كان بيمنع driver من يسرق pin محجوز لحاجة تانية.

الـ **pinmux framework** جه يحل ده: **centralized ownership + arbitration** لكل pin.

---

### Big Picture: Architecture

```
  ┌──────────────────────────────────────────────────────────┐
  │                  Consumer Drivers                        │
  │         (UART, SPI, I2C, custom device drivers)         │
  └──────────────┬────────────────────────────┬─────────────┘
                 │ pinctrl_get/select_state    │ gpio_request
                 ▼                             ▼
  ┌──────────────────────────┐   ┌─────────────────────────┐
  │     pinctrl core         │   │     GPIO subsystem       │
  │  (drivers/pinctrl/core.c)│   │   (gpiolib / gpio-chip)  │
  └──────────┬───────────────┘   └────────────┬────────────┘
             │                                │
             │  pinmux_enable_setting()        │  pinmux_request_gpio()
             ▼                                ▼
  ┌──────────────────────────────────────────────────────────┐
  │              pinmux core  (pinmux.c)                     │
  │                                                          │
  │  • ownership tracking (pin_desc.mux_owner / gpio_owner)  │
  │  • conflict detection (strict mode / usecount)           │
  │  • map → setting resolution (func name → selector)       │
  │  • delegates actual HW writes to driver via pmxops       │
  └────────────────────────────┬─────────────────────────────┘
                               │ pmxops->set_mux()
                               │ pmxops->gpio_request_enable()
                               ▼
  ┌──────────────────────────────────────────────────────────┐
  │          SoC-specific pinctrl driver                     │
  │   (e.g. pinctrl-bcm2835.c / pinctrl-stm32.c / ...)      │
  │                                                          │
  │  • knows actual register addresses                       │
  │  • implements struct pinmux_ops callbacks                │
  │  • defines functions + groups for its hardware           │
  └──────────────────────────────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────┐
                    │  Physical Pins  │
                    │  (SoC pads)     │
                    └─────────────────┘
```

---

### تشبيه من الواقع

تخيل **فندق** فيه غرف (pins)، وكل غرفة ممكن تتأجر لـ شركات مختلفة (UART, SPI, GPIO...).

| مفهوم Kernel | مقابله في الفندق |
|---|---|
| `pin_desc` | سجل الغرفة |
| `mux_owner` | اسم الشركة اللي أجرت الغرفة |
| `gpio_owner` | الشخص اللي استخدم الغرفة كـ self-service |
| `pinmux core` | موظف الاستقبال (يحجز، يمنع التعارض) |
| `pmxops->set_mux()` | السباك اللي بيوصل الأنابيب فعلياً |
| `strict mode` | سياسة الفندق: غرفة واحدة = مستخدم واحد بالظبط |
| `mux_usecount` | عدد العقود على نفس الغرفة (shared non-strict) |

---

### Core Abstractions: ما بيملكه الـ Core vs ما بيفوضه

#### ما بيملكه pinmux core:

1. **Ownership tracking** — مين حاجز الـ pin ده؟
   - `pin_desc.mux_owner` → اسم الـ device اللي عمل `pinctrl_get()`
   - `pin_desc.gpio_owner` → اسم الـ GPIO اللي طلب الـ pin

2. **Conflict detection** — قبل أي حجز بيتأكد:
   - non-strict: نفس الـ owner ممكن يحجز نفس الـ group أكتر من مرة (usecount)
   - strict: غرفة واحدة = صاحب واحد بس، GPIO و mux function ما يجتمعوش

3. **Map resolution** — يحول اسم function نصي → selector رقم:
   ```c
   // "uart0" → selector=3, "spi1_grp" → group=7
   pinmux_func_name_to_selector(pctldev, "uart0");
   pinctrl_get_group_selector(pctldev, "spi1_grp");
   ```

4. **Settings lifecycle** — `pinmux_map_to_setting()` → `pinmux_enable_setting()` → `pinmux_disable_setting()`

#### ما بيفوضه للـ driver عبر `pinmux_ops`:

- الـ driver هو اللي يكتب الـ registers الفعلية
- يحدد الـ functions المتاحة وعلاقتها بالـ groups
- يقرر لو الـ pin يقدر يتاخد أصلاً (optional `request` callback)

---

### Struct Relationship Graph

```
struct pinctrl_desc
  ├── .pins[]           → struct pinctrl_pin_desc[]
  ├── .pctlops          → struct pinctrl_ops        (groups: count/name/pins)
  ├── .pmxops           → struct pinmux_ops          (mux: set/request/gpio)
  └── .confops          → struct pinconf_ops         (config: pull/drive/...)

struct pinctrl_dev                        (runtime instance)
  ├── .desc             ──────────────────► struct pinctrl_desc (above)
  ├── .pin_desc_tree    → radix_tree { pin_number → struct pin_desc }
  ├── .pin_function_tree→ radix_tree { selector  → struct function_desc }
  ├── .pin_group_tree   → radix_tree { selector  → struct group_desc }
  ├── .gpio_ranges      → list of struct pinctrl_gpio_range
  └── .p                → struct pinctrl  (self-hog handle)

struct pin_desc                           (per physical pin)
  ├── .name
  ├── .mux_usecount     ← incremented per pin_request() call (non-GPIO)
  ├── .mux_owner        ← device name that called pinctrl_get()
  ├── .mux_setting      → struct pinctrl_setting_mux { .func, .group }
  ├── .gpio_owner       ← "gpiochip0:42" style string
  └── .mux_lock         ← per-pin mutex (fine-grained locking)

struct pinmux_ops                         (driver vtable)
  ├── .request(pctldev, pin)              ← optional: can driver give this pin?
  ├── .free(pctldev, pin)                 ← reverse of request
  ├── .get_functions_count(pctldev)       ← how many mux functions?
  ├── .get_function_name(pctldev, sel)    ← "uart0", "spi1", "gpio", ...
  ├── .get_function_groups(pctldev, sel)  ← which pin groups can carry func?
  ├── .function_is_gpio(pctldev, sel)     ← is this the GPIO function?
  ├── .set_mux(pctldev, func, group)      ← WRITE THE REGISTERS (mandatory)
  ├── .gpio_request_enable(...)           ← mux single pin to GPIO
  ├── .gpio_disable_free(...)             ← reverse
  ├── .gpio_set_direction(...)            ← input vs output for GPIO mux
  └── .strict                             ← bool: enforce exclusive ownership

struct pinfunction                        (generic function descriptor)
  ├── .name                               ← "uart0"
  ├── .groups[]                           ← { "uart0_grp_a", "uart0_grp_b" }
  ├── .ngroups
  └── .flags                              ← PINFUNCTION_FLAG_GPIO if GPIO func

struct function_desc                      (generic wrapper, radix tree node)
  ├── .func             ──────────────────► struct pinfunction
  └── .data                               ← driver private data

struct pinctrl_setting                    (one active mux or config entry)
  ├── .type             ← PIN_MAP_TYPE_MUX_GROUP
  ├── .pctldev
  ├── .dev_name
  └── .data.mux         → struct pinctrl_setting_mux { .func, .group }
```

---

### الـ Key Flows بالكود

#### 1. تفعيل Mux Setting (device يطلب "uart" state)

```
pinctrl_select_state()
  └─► pinmux_enable_setting(setting)
        ├── pctlops->get_group_pins()        // اعرف الـ pins في الـ group
        ├── pin_request() × N                // احجز كل pin (ownership check)
        │     ├── check mux_usecount / mux_owner / gpio_owner
        │     ├── ops->request(pctldev, pin) // optional driver check
        │     └── try_module_get()           // pin controller module refcount
        ├── desc->mux_setting = &setting.data.mux  // سجل الـ setting في كل pin
        └── ops->set_mux(pctldev, func, group)     // اكتب الـ HW registers
```

#### 2. GPIO Request على pin محجوز

```c
// في pinmux_can_be_used_for_gpio():
if (ops->strict && desc->mux_usecount && !func_is_gpio)
    return false;  // مش هينفع، pin محجوز لـ peripheral تاني

// في pin_request() لما gpio_range != NULL:
if ((gpio_range || ops->strict) && !gpio_ok && desc->gpio_owner)
    → "pin already requested by X" error
```

#### 3. Generic Functions Infrastructure

لما الـ driver يستخدم `CONFIG_GENERIC_PINMUX_FUNCTIONS`:

```c
// Driver في probe():
pinmux_generic_add_pinfunction(pctldev, &uart0_func, driver_data);
// Core بيحطها في pin_function_tree radix tree بـ selector رقم

// Core وقت enable:
pinmux_generic_get_function_name(pctldev, selector);  // lookup من الـ radix tree
pinmux_generic_get_function_groups(pctldev, selector, &groups, &ngroups);
```

---

### الـ strict Mode

```
strict = false (default):
  ┌─────┐     ┌─────┐
  │ mux │ +   │ GPIO│  ← مسموح يتعايشوا على نفس الـ pin
  └─────┘     └─────┘
  (غالباً الـ controller بيكون flexible)

strict = true:
  ┌─────────────────┐
  │  mux OR gpio    │  ← واحد بس، أي تعارض = -EBUSY
  │  (not both)     │
  └─────────────────┘
  + function_is_gpio() لازم يتعرف الـ GPIO function عشان الـ check يشتغل صح
```

---

### ملخص الـ Core vs Driver Boundary

| المسؤولية | pinmux core | SoC Driver |
|---|---|---|
| Ownership tracking | ✓ | |
| Conflict detection | ✓ | |
| Map → selector resolution | ✓ | |
| Functions/groups enumeration | delegates via ops | ✓ implements |
| Writing HW registers | | ✓ `set_mux()` |
| GPIO mux arbitration | ✓ checks | ✓ `gpio_request_enable()` |
| debugfs (functions/pins) | ✓ | optional `pin_dbg_show` |
| Module refcounting | ✓ `try_module_get` | |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — Flags, Enums, Config Options

#### `enum pinctrl_map_type`
| القيمة | المعنى |
|---|---|
| `PIN_MAP_TYPE_INVALID` | entry غلط |
| `PIN_MAP_TYPE_DUMMY_STATE` | state وهمي بدون أي إعداد فعلي |
| `PIN_MAP_TYPE_MUX_GROUP` | ضبط mux لمجموعة pins |
| `PIN_MAP_TYPE_CONFIGS_PIN` | ضبط config لـ pin واحد |
| `PIN_MAP_TYPE_CONFIGS_GROUP` | ضبط config لمجموعة |

#### `PINFUNCTION_FLAG_*`
| الـ Flag | القيمة | المعنى |
|---|---|---|
| `PINFUNCTION_FLAG_GPIO` | `BIT(0)` | الـ function ده هو GPIO — بيُستخدم مع `strict` mode |

#### `struct pinmux_ops` — الـ Callbacks الإلزامية vs الاختيارية
| الـ Callback | إلزامي؟ | المعنى |
|---|---|---|
| `get_functions_count` | **نعم** | عدد الـ functions المتاحة |
| `get_function_name` | **نعم** | اسم function بـ selector |
| `get_function_groups` | **نعم** | المجموعات اللي بيشتغل عليها الـ function |
| `set_mux` | **نعم** | برمجة الـ hardware فعلياً |
| `request` | لا | يأذن/يرفض طلب pin |
| `free` | لا | تحرير pin |
| `function_is_gpio` | لا | بيقول هل الـ function ده GPIO؟ |
| `gpio_request_enable` | لا | تفعيل GPIO على pin بعينه |
| `gpio_disable_free` | لا | إلغاء GPIO |
| `gpio_set_direction` | لا | input/output |
| `strict` (bool) | لا | منع الاستخدام المتزامن لـ GPIO + mux |

---

### 1. الـ Structs المهمة

#### `struct pinmux_ops`
**الغرض:** الـ vtable الخاص بالـ pin controller driver لتنفيذ عمليات الـ muxing.

```c
// include/linux/pinctrl/pinmux.h
struct pinmux_ops {
    int (*request)(struct pinctrl_dev *pctldev, unsigned int offset);
    int (*free)(struct pinctrl_dev *pctldev, unsigned int offset);
    int (*get_functions_count)(struct pinctrl_dev *pctldev);
    const char *(*get_function_name)(struct pinctrl_dev *pctldev,
                                     unsigned int selector);
    int (*get_function_groups)(struct pinctrl_dev *pctldev,
                               unsigned int selector,
                               const char * const **groups,
                               unsigned int *num_groups);
    bool (*function_is_gpio)(struct pinctrl_dev *pctldev,
                             unsigned int selector);
    int (*set_mux)(struct pinctrl_dev *pctldev,
                   unsigned int func_selector,
                   unsigned int group_selector);
    int (*gpio_request_enable)(struct pinctrl_dev *pctldev,
                               struct pinctrl_gpio_range *range,
                               unsigned int offset);
    void (*gpio_disable_free)(struct pinctrl_dev *pctldev,
                              struct pinctrl_gpio_range *range,
                              unsigned int offset);
    int (*gpio_set_direction)(struct pinctrl_dev *pctldev,
                              struct pinctrl_gpio_range *range,
                              unsigned int offset,
                              bool input);
    bool strict; // منع التعارض بين GPIO و mux
};
```

**الحقول المهمة:**
- `set_mux` — الأهم: بيبرمج الـ hardware register فعلياً.
- `strict` — لو `true`، pin مش ممكن يكون GPIO و mux في نفس الوقت خالص.
- `function_is_gpio` — بيُعرّف الـ core إن فيه function اسمه "gpio" عشان يتعامل معاه بشكل خاص في `strict` mode.

---

#### `struct pin_desc`
**الغرض:** يصف كل pin فيزيائي — مين بيمتلكه، ومتى.

```c
// drivers/pinctrl/core.h
struct pin_desc {
    struct pinctrl_dev *pctldev;     // الـ controller المسؤول عنه
    const char *name;                // اسم الـ pin
    bool dynamic_name;               // الاسم اتخصص ديناميكياً؟
    void *drv_data;                  // بيانات driver خاصة
#ifdef CONFIG_PINMUX
    unsigned int mux_usecount;       // كام مرة اتطلب بـ mux (ref count)
    const char *mux_owner;           // اسم الجهاز اللي طالبه بـ mux
    const struct pinctrl_setting_mux *mux_setting; // الـ mux setting الحالي
    const char *gpio_owner;          // اسم الـ GPIO اللي طالبه
    struct mutex mux_lock;           // حماية الـ mux fields
#endif
};
```

**الحقول المهمة:**
- `mux_usecount` — مش boolean، ref count عشان نفس الـ pin ممكن يتطلب من أكتر من mapping entry بنفس الـ owner.
- `mux_owner` vs `gpio_owner` — owner مختلف لكل نوع استخدام — في `strict` mode، الاتنين مينفعوش يتعمروا مع بعض.
- `mux_setting` — pointer للـ `pinctrl_setting_mux` الحالي، بيتحدد في `pinmux_enable_setting()`.
- `mux_lock` — mutex على مستوى الـ pin نفسه (مش global)، بيحمي الـ mux/gpio fields.

---

#### `struct pinctrl_setting_mux`
**الغرض:** الـ "عنوان" الكامل لإعداد الـ mux — function + group.

```c
// drivers/pinctrl/core.h
struct pinctrl_setting_mux {
    unsigned int group; // رقم مجموعة الـ pins (group selector)
    unsigned int func;  // رقم الـ function (function selector)
};
```

بسيطة جداً — بتحفظ الـ selectors اللي بيتحولوا لأرقام في `pinmux_map_to_setting()`.

---

#### `struct pinctrl_setting`
**الغرض:** وحدة إعداد واحدة — إما mux أو config — مرتبطة بـ state معين.

```c
// drivers/pinctrl/core.h
struct pinctrl_setting {
    struct list_head node;           // مربوط في قائمة settings الـ state
    enum pinctrl_map_type type;      // MUX_GROUP أو CONFIGS_PIN أو ...
    struct pinctrl_dev *pctldev;     // الـ controller المنفذ
    const char *dev_name;            // اسم الـ device صاحب الطلب
    union {
        struct pinctrl_setting_mux mux;         // لو type = MUX_GROUP
        struct pinctrl_setting_configs configs; // لو type = CONFIGS_*
    } data;
};
```

---

#### `struct pinctrl_map`
**الغرض:** البيانات الثابتة اللي بتحدد "إيه اللي المفروض يحصل" — بتيجي من board file أو device tree.

```c
// include/linux/pinctrl/machine.h
struct pinctrl_map {
    const char *dev_name;       // اسم الـ device اللي عايز الإعداد ده
    const char *name;           // اسم الـ state (default, sleep, ...)
    enum pinctrl_map_type type;
    const char *ctrl_dev_name;  // اسم الـ pin controller المنفذ
    union {
        struct pinctrl_map_mux mux;         // { group, function }
        struct pinctrl_map_configs configs; // { pin/group, configs[] }
    } data;
};
```

**الفرق بين `pinctrl_map` و `pinctrl_setting`:**
- `pinctrl_map` — ثابت، بيانات raw من board code.
- `pinctrl_setting` — runtime، بعد تحويل الأسماء لـ selectors.

---

#### `struct function_desc`
**الغرض:** الـ generic descriptor للـ function في `CONFIG_GENERIC_PINMUX_FUNCTIONS`.

```c
// drivers/pinctrl/pinmux.h
struct function_desc {
    const struct pinfunction *func; // الـ function نفسه (name + groups)
    void *data;                     // بيانات driver خاصة
};
```

---

#### `struct pinfunction`
**الغرض:** البيانات الأساسية لـ function — اسمه وإيه المجموعات اللي بيشتغل عليها.

```c
// include/linux/pinctrl/pinctrl.h
struct pinfunction {
    const char *name;              // اسم الـ function (مثلاً "uart0")
    const char * const *groups;   // أسماء المجموعات المتاحة
    size_t ngroups;               // عددهم
    unsigned long flags;          // PINFUNCTION_FLAG_GPIO أو 0
};
```

---

#### `struct pinctrl_dev`
**الغرض:** تمثيل الـ pin controller نفسه في الـ runtime — القلب اللي كل حاجة مربوطة بيه.

```c
// drivers/pinctrl/core.h (مختصر)
struct pinctrl_dev {
    struct list_head node;              // في القائمة العالمية للـ controllers
    const struct pinctrl_desc *desc;   // الـ descriptor (vtables + pins[])
    struct radix_tree_root pin_desc_tree;      // pin number -> pin_desc
    struct radix_tree_root pin_function_tree;  // selector -> function_desc
    struct radix_tree_root pin_group_tree;     // selector -> group_desc
    unsigned int num_functions;
    unsigned int num_groups;
    struct list_head gpio_ranges;       // قائمة GPIO ranges
    struct device *dev;
    struct module *owner;              // لـ module refcount
    struct pinctrl *p;                 // الـ hog pinctrl handle
    struct pinctrl_state *hog_default;
    struct pinctrl_state *hog_sleep;
    struct mutex mutex;                // global lock للـ controller
};
```

---

#### `struct pinctrl_gpio_range`
**الغرض:** بيعرّف نطاق GPIO numbers اللي الـ pin controller بيتكلم عنها.

```c
// include/linux/pinctrl/pinctrl.h
struct pinctrl_gpio_range {
    struct list_head node;
    const char *name;          // اسم الـ GPIO chip
    unsigned int id;
    unsigned int base;         // أول رقم GPIO في النطاق
    unsigned int pin_base;     // أول رقم pin مقابل
    unsigned int npins;        // عدد الـ pins
    unsigned int const *pins;  // أو array صريح بدل pin_base
    struct gpio_chip *gc;      // الـ GPIO chip المرتبط
};
```

---

### 2. علاقات الـ Structs — ASCII Diagram

```
  Board/DT Code
  ┌─────────────────────────────────┐
  │  pinctrl_map[]                  │
  │  { dev_name, state_name,        │
  │    type=MUX_GROUP,              │
  │    data.mux = {group, func} }   │
  └────────────┬────────────────────┘
               │ pinctrl_register_mappings()
               ▼
  ┌──────────────────────┐
  │  pinctrl_maps        │  (قائمة عالمية)
  │  (list_head + maps[])│
  └──────────┬───────────┘
             │ pinmux_map_to_setting()
             ▼
  ┌───────────────────────────────┐
  │  pinctrl_setting              │
  │  { type, pctldev, dev_name,   │
  │    data.mux = {func, group} } │◄── selectors (أرقام)
  └──────┬────────────────────────┘
         │ مربوط في
         ▼
  ┌──────────────────┐
  │  pinctrl_state   │
  │  { name, list    │
  │    of settings } │
  └──────┬───────────┘
         │ مربوط في
         ▼
  ┌──────────────┐
  │  pinctrl     │
  │  { dev,      │
  │    states,   │
  │    state* }  │
  └──────────────┘

  ────────────────────────────────────────────────────────

  pinctrl_dev  (الـ controller)
  ┌──────────────────────────────────────────────────────┐
  │  desc ──────────────► pinctrl_desc                   │
  │                        { pins[], npins,              │
  │                          pctlops*, pmxops*,          │
  │                          confops* }                  │
  │                               │                      │
  │                               ├──► pinctrl_ops       │
  │                               │    (get_group_*)     │
  │                               │                      │
  │                               └──► pinmux_ops        │
  │                                    (set_mux, ...)    │
  │                                                      │
  │  pin_desc_tree ────► pin_desc (per pin)              │
  │                        { mux_usecount,               │
  │                          mux_owner,                  │
  │                          mux_setting*,               │
  │                          gpio_owner,                 │
  │                          mux_lock }                  │
  │                               │                      │
  │                               └──► pinctrl_setting_mux│
  │                                    { func, group }   │
  │                                                      │
  │  pin_function_tree ──► function_desc                 │
  │                          { func*, data }             │
  │                               │                      │
  │                               └──► pinfunction       │
  │                                    { name, groups[], │
  │                                      ngroups, flags }│
  │                                                      │
  │  gpio_ranges ──────► pinctrl_gpio_range              │
  │                        { base, npins, gc* }          │
  └──────────────────────────────────────────────────────┘
```

---

### 3. Lifecycle Diagrams

#### lifecycle الـ Pin (من التسجيل للتحرير)

```
  driver probe
      │
      ▼
  pinctrl_register_and_init()
      │
      ├──► pinmux_check_ops()   ← التحقق إن الـ ops الإلزامية موجودة
      │
      ├──► [alloc pin_desc per pin → pin_desc_tree]
      │
      └──► pinctrl_enable()
               │
               ▼
           [hog mappings: device requests its own pins]
               │
               ▼
           pinmux_map_to_setting()
               │  converts names → selectors
               ▼
           pinmux_enable_setting()
               │
               ├──► pin_request() × num_pins
               │         │
               │         ├─ check mux_owner / gpio_owner conflicts
               │         ├─ desc->mux_usecount++
               │         ├─ try_module_get()
               │         └─ ops->request() OR ops->gpio_request_enable()
               │
               ├──► desc->mux_setting = &setting->data.mux  (per pin)
               │
               └──► ops->set_mux(pctldev, func, group)
                         │
                         ▼
                    [hardware register written]

  ─────────── عند suspend أو driver remove ───────────

  pinmux_disable_setting()
      │
      ├──► check desc->mux_setting == &setting->data.mux
      └──► pin_free() × num_pins
               │
               ├─ desc->mux_usecount--
               ├─ ops->free() OR ops->gpio_disable_free()
               └─ module_put()
```

#### lifecycle طلب GPIO Pin

```
  gpio_request(gpio_num)
       │
       ▼
  pinmux_request_gpio(pctldev, range, pin, gpio)
       │
       ├──► kasprintf("rangename:gpio_num") → owner string
       │
       └──► pin_request(pctldev, pin, owner, range)
                 │
                 ├─ check: ops->strict && mux_usecount && !func_is_gpio → FAIL
                 ├─ check: gpio_owner already set → FAIL
                 ├─ desc->gpio_owner = owner
                 ├─ try_module_get()
                 └─ ops->gpio_request_enable(pctldev, range, pin)

  gpio_free(gpio_num)
       │
       ▼
  pinmux_free_gpio(pctldev, pin, range)
       │
       └──► pin_free(pctldev, pin, range)
                 │
                 ├─ owner = desc->gpio_owner
                 ├─ desc->gpio_owner = NULL
                 ├─ ops->gpio_disable_free()
                 ├─ module_put()
                 └─ return owner → kfree(owner)
```

---

### 4. Call Flow Diagrams

#### المسار الكامل من `pinctrl_select_state()` لـ hardware

```
  pinctrl_select_state(p, state)         [pinctrl core]
        │
        └──► للـ setting في state->settings:
                  │
                  ├─ type == MUX_GROUP
                  │       │
                  │       └──► pinmux_enable_setting(setting)
                  │                   │
                  │                   ├─ pctlops->get_group_pins()
                  │                   │       → pins[], num_pins
                  │                   │
                  │                   ├─ pin_request() لكل pin
                  │                   │       │
                  │                   │       └─ [check owners, usecount]
                  │                   │          ops->request(pctldev, pin)
                  │                   │
                  │                   ├─ desc->mux_setting = &setting.mux
                  │                   │
                  │                   └─ pmxops->set_mux(func, group)
                  │                             │
                  │                             └──► [write HW register]
                  │
                  └─ type == CONFIGS_*
                          │
                          └──► pinconf_apply_setting()  [مش هنا]
```

#### `pinmux_map_to_setting()` — تحويل map لـ setting

```
  pinmux_map_to_setting(map, setting)
        │
        ├─ pinmux_func_name_to_selector(pctldev, map->data.mux.function)
        │       │  يلف على كل function بـ get_function_name()
        │       └──► setting->data.mux.func = selector
        │
        ├─ pmxops->get_function_groups(pctldev, func_sel)
        │       └──► groups[], num_groups
        │
        ├─ لو map->data.mux.group محدد:
        │       match_string(groups, num_groups, group)
        │  لو مش محدد: groups[0]
        │
        └─ pinctrl_get_group_selector(pctldev, group_name)
                └──► setting->data.mux.group = selector
```

#### debugfs write path (تجربة يدوية من shell)

```
  echo "group_name func_name" > /sys/kernel/debug/.../pinmux-select
        │
        └──► pinmux_select_write()
                  │
                  ├─ parse: gname + fname من الـ input
                  ├─ pinmux_func_name_to_selector() → fsel
                  ├─ pmxops->get_function_groups() → groups[]
                  ├─ match_string(groups, num_groups, gname) → validate
                  ├─ pinctrl_get_group_selector(gname) → gsel
                  └─ pmxops->set_mux(pctldev, fsel, gsel)
```

---

### 5. Locking Strategy

الـ pinmux بيستخدم **مستويين من الـ locking**:

```
  المستوى الأول: pctldev->mutex
  ────────────────────────────
  - Global lock لكل الـ controller
  - بيتاخد في: debugfs functions (pinmux_functions_show, pinmux_pins_show)
  - بيحمي: قراءة pin_desc_tree و num_functions بشكل آمن
  - مش بيتاخد في: enable/disable setting (عشان ممكن تحصل من contexts تانية)

  المستوى التاني: desc->mux_lock  (per-pin mutex)
  ──────────────────────────────────────────────
  - Fine-grained lock على مستوى كل pin
  - بيتاخد في:
      ├─ pin_request()   → check + set mux_owner/gpio_owner/mux_usecount
      ├─ pin_free()      → clear owners
      ├─ pinmux_enable_setting() → set desc->mux_setting
      ├─ pinmux_disable_setting() → check + clear desc->mux_setting
      └─ pinmux_can_be_used_for_gpio() → check conflicts
  - بيستخدم guard(mutex) و scoped_guard() من kernel cleanup API
```

**ليه مستويين؟**

```
  ┌──────────────────────────────────────────────────┐
  │ لو استخدمنا pctldev->mutex بس:                  │
  │   → كل عملية pin request هتحجب الـ controller   │
  │     كله → bottleneck كبير في أنظمة كتير pins    │
  │                                                  │
  │ بالـ per-pin lock:                               │
  │   → pins مختلفة ممكن تتطلب في نفس الوقت         │
  │   → الـ global lock بس لقراءة البيانات الثابتة  │
  └──────────────────────────────────────────────────┘
```

**نمط الاستخدام في الكود:**

```c
// نمط scoped_guard — block محدد
scoped_guard(mutex, &desc->mux_lock) {
    desc->mux_owner = owner;
    desc->mux_usecount++;
}

// نمط guard — لحد نهاية الـ scope
guard(mutex)(&desc->mux_lock);
if (mux_setting && ops->function_is_gpio)
    func_is_gpio = ops->function_is_gpio(...);
// mutex يتحرر أوتوماتيك عند خروج من الـ scope
```

**تحذير لـ strict mode:**

```
  strict=true:
  ┌─────────────┬──────────────────┬────────────┐
  │ الحالة      │ gpio_owner محدد؟ │ النتيجة    │
  ├─────────────┼──────────────────┼────────────┤
  │ mux_request │ نعم              │ EBUSY      │
  │ gpio_request│ مع mux_usecount  │ EBUSY      │
  │             │ و func != gpio   │            │
  └─────────────┴──────────────────┴────────────┘

  strict=false:
  → GPIO و mux ممكن يتشاركوا نفس الـ pin
  → بس بيطلع warning في debugfs إنه unclaimed/shared
```

---

### ملخص العلاقات

```
  pinctrl_map (static, board)
       │
       │  pinmux_map_to_setting()
       ▼
  pinctrl_setting (runtime)
       │ stored in pinctrl_state->settings
       │
       │  pinmux_enable_setting()
       ▼
  pin_request() per pin
       │ updates pin_desc:
       │   mux_owner / mux_usecount / gpio_owner
       │   mux_setting → &setting.data.mux
       │
       │  pmxops->set_mux()
       ▼
  Hardware Register (pin muxed to function)
```
## Phase 4: شرح الـ Functions

---

### Cheatsheet — كل الـ Functions والـ API

| Function | Category | Exported? | Config Guard |
|---|---|---|---|
| `pinmux_check_ops` | Validation | No | `CONFIG_PINMUX` |
| `pinmux_validate_map` | Validation | No | `CONFIG_PINMUX` |
| `pinmux_can_be_used_for_gpio` | GPIO Mux | No | `CONFIG_PINMUX` |
| `pin_request` *(static)* | Core Internal | No | `CONFIG_PINMUX` |
| `pin_free` *(static)* | Core Internal | No | `CONFIG_PINMUX` |
| `pinmux_request_gpio` | GPIO Mux | No | `CONFIG_PINMUX` |
| `pinmux_free_gpio` | GPIO Mux | No | `CONFIG_PINMUX` |
| `pinmux_gpio_direction` | GPIO Mux | No | `CONFIG_PINMUX` |
| `pinmux_func_name_to_selector` *(static)* | Lookup | No | `CONFIG_PINMUX` |
| `pinmux_map_to_setting` | Setting Mgmt | No | `CONFIG_PINMUX` |
| `pinmux_free_setting` | Setting Mgmt | No | `CONFIG_PINMUX` |
| `pinmux_enable_setting` | Setting Mgmt | No | `CONFIG_PINMUX` |
| `pinmux_disable_setting` | Setting Mgmt | No | `CONFIG_PINMUX` |
| `pinmux_functions_show` *(static)* | DebugFS | No | `CONFIG_DEBUG_FS` |
| `pinmux_pins_show` *(static)* | DebugFS | No | `CONFIG_DEBUG_FS` |
| `pinmux_show_map` | DebugFS | No | `CONFIG_DEBUG_FS` |
| `pinmux_show_setting` | DebugFS | No | `CONFIG_DEBUG_FS` |
| `pinmux_select_show` *(static)* | DebugFS | No | `CONFIG_DEBUG_FS` |
| `pinmux_select_write` *(static)* | DebugFS | No | `CONFIG_DEBUG_FS` |
| `pinmux_init_device_debugfs` | DebugFS | No | `CONFIG_DEBUG_FS` |
| `pinmux_generic_get_function_count` | Generic API | `EXPORT_SYMBOL_GPL` | `CONFIG_GENERIC_PINMUX_FUNCTIONS` |
| `pinmux_generic_get_function_name` | Generic API | `EXPORT_SYMBOL_GPL` | `CONFIG_GENERIC_PINMUX_FUNCTIONS` |
| `pinmux_generic_get_function_groups` | Generic API | `EXPORT_SYMBOL_GPL` | `CONFIG_GENERIC_PINMUX_FUNCTIONS` |
| `pinmux_generic_get_function` | Generic API | `EXPORT_SYMBOL_GPL` | `CONFIG_GENERIC_PINMUX_FUNCTIONS` |
| `pinmux_generic_function_is_gpio` | Generic API | `EXPORT_SYMBOL_GPL` | `CONFIG_GENERIC_PINMUX_FUNCTIONS` |
| `pinmux_generic_add_function` | Generic API | `EXPORT_SYMBOL_GPL` | `CONFIG_GENERIC_PINMUX_FUNCTIONS` |
| `pinmux_generic_add_pinfunction` | Generic API | `EXPORT_SYMBOL_GPL` | `CONFIG_GENERIC_PINMUX_FUNCTIONS` |
| `pinmux_generic_remove_function` | Generic API | `EXPORT_SYMBOL_GPL` | `CONFIG_GENERIC_PINMUX_FUNCTIONS` |
| `pinmux_generic_free_functions` | Generic API | No | `CONFIG_GENERIC_PINMUX_FUNCTIONS` |

---

### Category 1: Validation — التحقق من صحة الـ ops والـ map

---

#### `pinmux_check_ops`

```c
int pinmux_check_ops(struct pinctrl_dev *pctldev);
```

**بتعمل إيه:** بتتحقق إن الـ `pinmux_ops` المسجلة في الـ `pctldev` فيها كل الـ callbacks الإلزامية، وإن كل function معرفة فيها اسم.

**Parameters:**
- `pctldev` — الـ pin controller device المراد تسجيله

**Return:** `0` لو كل حاجة تمام، `-EINVAL` لو:
- `pmxops` نفسها `NULL`
- أي من الـ callbacks الإلزامية (`get_functions_count`، `get_function_name`، `get_function_groups`، `set_mux`) مش موجودة
- أي function بترجع اسم `NULL`

**Locking:** لا يحتاج lock — بيُستدعى وقت registration قبل ما أي thread تاني يقدر يصل للـ device.

**Side effects:** يطبع `dev_err` لو في مشكلة.

**بيستدعيها:** `pinctrl_register` / `devm_pinctrl_register` في `core.c` وقت تسجيل الـ controller.

---

#### `pinmux_validate_map`

```c
int pinmux_validate_map(const struct pinctrl_map *map, int i);
```

**بتعمل إيه:** بتتحقق إن الـ `pinctrl_map` entry من نوع `PIN_MAP_TYPE_MUX_GROUP` عندها اسم function مش `NULL`.

**Parameters:**
- `map` — الـ map entry المراد التحقق منها
- `i` — الـ index بتاعها في الـ map table (للـ error message بس)

**Return:** `0` أو `-EINVAL`.

**Locking:** لا شيء.

**بيستدعيها:** `pinctrl_register_map` في `core.c`.

---

### Category 2: GPIO Mux — طلب وتحرير الـ pins للـ GPIO

---

#### `pinmux_can_be_used_for_gpio`

```c
bool pinmux_can_be_used_for_gpio(struct pinctrl_dev *pctldev, unsigned int pin);
```

**بتعمل إيه:** بتسأل: "هل الـ pin ده ممكن يتستخدم كـ GPIO دلوقتي؟". الـ controllers اللي مش `strict` بترجع `true` دايمًا. الـ `strict` controllers بتتحقق إن الـ pin مش مستخدم في function تانية مش GPIO، ومش عنده `gpio_owner` تاني.

**Parameters:**
- `pctldev` — الـ pin controller
- `pin` — رقم الـ pin في الـ global pin space

**Return:** `true` لو ممكن يتستخدم كـ GPIO، `false` لو محجوز لحاجة تانية.

**Locking:** `desc->mux_lock` (mutex) — بيستخدم `guard(mutex)` الـ scoped locking.

**Side effects:** لا شيء.

**بيستدعيها:** `gpiochip_add_pin_range` وأي code بيحتاج يتحقق قبل ما يطلب GPIO.

---

#### `pin_request` *(static)*

```c
static int pin_request(struct pinctrl_dev *pctldev,
                       int pin, const char *owner,
                       struct pinctrl_gpio_range *gpio_range);
```

**بتعمل إيه:** الـ core function لطلب pin واحد. لو `gpio_range != NULL` → طلب GPIO، غير كده → طلب mux function. بتتحقق من conflicts، بتزود الـ refcount على الـ module، وبتستدعي الـ hardware callback المناسب.

**Parameters:**
- `pctldev` — الـ controller
- `pin` — رقم الـ pin
- `owner` — string بيعرّف المالك (اسم الـ device أو GPIO)
- `gpio_range` — لو GPIO request، الـ range المناسبة؛ `NULL` لو mux request

**Return:** `0` لو نجح، أو كود خطأ سالب.

**Locking:** `desc->mux_lock` بـ `scoped_guard(mutex)`. الـ caller مش محتاج يمسك أي lock.

**Side effects:**
- بيزود `module refcount` للـ `pctldev->owner`
- بيسيت `desc->gpio_owner` أو `desc->mux_owner` + `desc->mux_usecount`
- بيستدعي `ops->gpio_request_enable` أو `ops->request` على الـ hardware

**Errors:**
- `-EINVAL` لو الـ pin مش مسجل، أو الـ module refcount فشل، أو الـ pin محجوز لحد تاني
- الـ error من الـ hardware callback لو فشل

**Pseudocode:**
```
pin_request(pctldev, pin, owner, gpio_range):
    desc = pin_desc_get(pin)           // get pin descriptor
    if !desc → return -EINVAL

    lock(desc->mux_lock):
        gpio_ok = check if current mux function is GPIO (or no mux set)

        // Conflict checks
        if mux request AND pin already owned by someone else AND not gpio_ok:
            error "already requested by <owner>"
            goto out

        if gpio request AND pin already has gpio_owner AND not gpio_ok:
            error "already requested by <gpio_owner>"
            goto out

        // Claim the pin
        if gpio_range:
            desc->gpio_owner = owner
        else:
            desc->mux_usecount++
            if usecount > 1: return 0   // already claimed, reuse

            desc->mux_owner = owner

    try_module_get(pctldev->owner)     // increment module refcount

    // Call hardware
    if gpio_range AND ops->gpio_request_enable:
        status = ops->gpio_request_enable(pctldev, gpio_range, pin)
    elif ops->request:
        status = ops->request(pctldev, pin)
    else:
        status = 0

    if status: rollback ownership + module_put()
    return status
```

**بيستدعيها:** `pinmux_request_gpio`، `pinmux_enable_setting`.

---

#### `pin_free` *(static)*

```c
static const char *pin_free(struct pinctrl_dev *pctldev, int pin,
                            struct pinctrl_gpio_range *gpio_range);
```

**بتعمل إيه:** عكس `pin_request`. بتحرر الـ pin، بتنقص الـ `mux_usecount`، وبتستدعي الـ hardware callback للتحرير. بترجع الـ `owner` string عشان الـ caller يعمل `kfree` عليه لو كان dynamically allocated.

**Parameters:**
- `pctldev` — الـ controller
- `pin` — رقم الـ pin
- `gpio_range` — لو تحرير GPIO؛ `NULL` لو تحرير mux

**Return:** الـ `owner` string السابقة (للـ `kfree` من الـ caller)، أو `NULL` لو حصل خطأ أو الـ `mux_usecount` لسه > 0 بعد الـ decrement.

**Locking:** `desc->mux_lock` بـ `scoped_guard(mutex)`.

**Side effects:**
- بتنقص `mux_usecount`، بتمسح `mux_owner`/`gpio_owner`/`mux_setting`
- بتستدعي `ops->gpio_disable_free` أو `ops->free` على الـ hardware
- بتعمل `module_put(pctldev->owner)`

**بيستدعيها:** `pinmux_free_gpio`، `pinmux_enable_setting` (في حالة الـ error rollback)، `pinmux_disable_setting`.

---

#### `pinmux_request_gpio`

```c
int pinmux_request_gpio(struct pinctrl_dev *pctldev,
                        struct pinctrl_gpio_range *range,
                        unsigned int pin, unsigned int gpio);
```

**بتعمل إيه:** واجهة عامة لطلب GPIO pin. بتعمل `kasprintf` لاسم الشكل `"<range_name>:<gpio_num>"` وبتستدعي `pin_request`.

**Parameters:**
- `pctldev` — الـ controller
- `range` — الـ GPIO range المناسبة
- `pin` — رقم الـ pin في الـ controller
- `gpio` — رقم الـ GPIO (للـ naming بس)

**Return:** `0` أو كود خطأ. لو فشل، بيعمل `kfree` للـ owner string تلقائيًا.

**Locking:** بيتم داخل `pin_request` → `desc->mux_lock`.

**Memory:** بيعمل `kasprintf` للـ owner؛ لو `pin_request` نجح، الـ ownership بتتنقل للـ `desc->gpio_owner` وبيتحرر وقت `pinmux_free_gpio`.

**بيستدعيها:** `pinctrl_gpio_request` في `core.c`.

---

#### `pinmux_free_gpio`

```c
void pinmux_free_gpio(struct pinctrl_dev *pctldev, unsigned int pin,
                      struct pinctrl_gpio_range *range);
```

**بتعمل إيه:** بتحرر الـ GPIO pin المطلوب سابقًا. بتستدعي `pin_free` وبتعمل `kfree` للـ owner string اللي كانت allocated بـ `kasprintf`.

**Parameters:**
- `pctldev`، `pin`، `range` — زي `pinmux_request_gpio`

**Return:** void.

**Locking:** داخل `pin_free` → `desc->mux_lock`.

**بيستدعيها:** `pinctrl_gpio_free` في `core.c`.

---

#### `pinmux_gpio_direction`

```c
int pinmux_gpio_direction(struct pinctrl_dev *pctldev,
                          struct pinctrl_gpio_range *range,
                          unsigned int pin, bool input);
```

**بتعمل إيه:** بتضبط اتجاه GPIO pin (input/output) عن طريق `ops->gpio_set_direction`. لو الـ callback مش موجود، بترجع `0` (افتراض إن الـ hardware مش محتاج تدخل).

**Parameters:**
- `input` — `true` = input، `false` = output

**Return:** `0` لو نجح أو الـ callback مش موجود، أو كود خطأ من الـ hardware.

**Locking:** لا شيء في الـ function نفسها.

**بيستدعيها:** `pinctrl_gpio_direction_input/output` في `core.c`.

---

### Category 3: Setting Management — إدارة الـ pinmux settings

---

#### `pinmux_func_name_to_selector` *(static)*

```c
static int pinmux_func_name_to_selector(struct pinctrl_dev *pctldev,
                                        const char *function);
```

**بتعمل إيه:** بتعمل linear search على كل الـ functions المسجلة في الـ controller وبترجع الـ selector (index) للـ function اللي اسمها مطابق.

**Parameters:**
- `function` — الاسم المطلوب

**Return:** الـ selector الـ `>= 0` لو لقته، `-EINVAL` لو مش موجود.

**Locking:** لا شيء — الـ caller مسئول.

**Complexity:** O(n) على عدد الـ functions.

**بيستدعيها:** `pinmux_map_to_setting`، `pinmux_select_write`، `pinmux_generic_add_pinfunction`.

---

#### `pinmux_map_to_setting`

```c
int pinmux_map_to_setting(const struct pinctrl_map *map,
                          struct pinctrl_setting *setting);
```

**بتعمل إيه:** بتحوّل `pinctrl_map` entry لـ `pinctrl_setting` جاهز للتفعيل. بتعمل:
1. Resolve اسم الـ function لـ selector → `setting->data.mux.func`
2. جيب الـ groups المرتبطة بالـ function
3. لو الـ map بيحدد group → تحقق وجودها، غير كده خد أول group
4. Resolve اسم الـ group لـ group selector → `setting->data.mux.group`

**Parameters:**
- `map` — الـ source map entry
- `setting` — الـ destination setting (لازم `setting->pctldev` يكون مضبوط)

**Return:** `0` أو كود خطأ (`-EINVAL` وغيره).

**Locking:** لا explicit locking.

**Side effects:** بتكتب في `setting->data.mux.func` و `setting->data.mux.group`.

**Pseudocode:**
```
pinmux_map_to_setting(map, setting):
    fsel = pinmux_func_name_to_selector(map->data.mux.function)
    if fsel < 0: return error

    setting->data.mux.func = fsel

    get_function_groups(fsel) → groups[], num_groups
    if num_groups == 0: return -EINVAL

    if map->data.mux.group specified:
        verify group in groups[]
        group = map->data.mux.group
    else:
        group = groups[0]   // default: first group

    gsel = pinctrl_get_group_selector(group)
    setting->data.mux.group = gsel
    return 0
```

**بيستدعيها:** `pinctrl_apply_setting` path في `core.c`.

---

#### `pinmux_free_setting`

```c
void pinmux_free_setting(const struct pinctrl_setting *setting);
```

**بتعمل إيه:** حاليًا **مفيهاش implementation** (empty function). موجودة كـ placeholder للـ symmetry مع `pinmux_enable_setting`.

**بيستدعيها:** `pinctrl_free_setting` في `core.c`.

---

#### `pinmux_enable_setting`

```c
int pinmux_enable_setting(const struct pinctrl_setting *setting);
```

**بتعمل إيه:** الـ function الأساسية لتفعيل mux setting كاملة. بتمر على كل الـ pins في الـ group، بتطلبهم واحد واحد، بتسجل الـ `mux_setting` في كل `pin_desc`، وأخيرًا بتستدعي `ops->set_mux` على الـ hardware. لو حصل error في أي خطوة، بتعمل rollback كامل.

**Parameters:**
- `setting` — الـ setting المراد تفعيله (بيحتوي على `func` و `group` selectors)

**Return:** `0` أو كود خطأ.

**Locking:**
- `desc->mux_lock` (per-pin mutex) أثناء كتابة `mux_setting`
- الـ caller في `core.c` يمسك `pctldev->mutex` عادةً

**Side effects:**
- بتستدعي `pin_request` لكل pin في الـ group
- بتكتب `desc->mux_setting` لكل pin
- بتستدعي `ops->set_mux` → hardware register write
- لو فشل: بتعمل rollback لكل الـ pins عبر `pin_free`

**Pseudocode:**
```
pinmux_enable_setting(setting):
    get_group_pins(setting->group) → pins[], num_pins

    // Phase 1: Request all pins
    for i in 0..num_pins:
        ret = pin_request(pins[i], setting->dev_name, NULL)
        if ret: goto err_pin_request   // rollback pins[0..i-1]

    // Phase 2: Record mux_setting in each pin desc
    for i in 0..num_pins:
        lock(desc->mux_lock):
            desc->mux_setting = &setting->data.mux

    // Phase 3: Apply to hardware
    ret = ops->set_mux(func, group)
    if ret: goto err_set_mux          // clear mux_setting + free all pins

    return 0

err_set_mux:
    for each pin: clear desc->mux_setting
err_pin_request:
    for each claimed pin: pin_free(pin)
    return ret
```

**بيستدعيها:** `pinctrl_commit_state` في `core.c`.

---

#### `pinmux_disable_setting`

```c
void pinmux_disable_setting(const struct pinctrl_setting *setting);
```

**بتعمل إيه:** عكس `pinmux_enable_setting`. بتمر على كل الـ pins في الـ group، وبتتحقق إن كل pin لسه فعلًا معاه نفس الـ `mux_setting` (مش اتغير من setting تانية)، وبعدين بتستدعي `pin_free` عليه.

**Parameters:**
- `setting` — الـ setting المراد تعطيله

**Return:** void.

**Locking:** `desc->mux_lock` للـ comparison، بعدين `pin_free` بيمسك الـ lock تاني للتحرير.

**Side effects:**
- بتستدعي `pin_free` لكل pin متحقق
- لو الـ pin تغيرت الـ setting بتاعته → `dev_warn` وبتتجاهل التحرير

**بيستدعيها:** `pinctrl_select_state` / state transitions في `core.c`.

---

### Category 4: DebugFS — واجهات الـ debugging

---

#### `pinmux_functions_show` *(static)*

```c
static int pinmux_functions_show(struct seq_file *s, void *what);
```

**بتعمل إيه:** بتطبع كل الـ functions المسجلة في الـ controller مع الـ groups المرتبطة بكل function.

**Output format:**
```
function 0: uart0, groups = [ uart0_grp uart0_flow_grp ]
function 1: spi0, groups = [ spi0_grp ]
```

**Locking:** `pctldev->mutex` (يمسكه ويحرره بنفسه).

**بيستدعيها:** kernel debugfs عند قراءة `/sys/kernel/debug/pinctrl/<dev>/pinmux-functions`.

---

#### `pinmux_pins_show` *(static)*

```c
static int pinmux_pins_show(struct seq_file *s, void *what);
```

**بتعمل إيه:** بتطبع حالة كل pin في الـ controller: مين بيملكه (`mux_owner`/`gpio_owner`)، هل هو hog، وإيه الـ function والـ group المضبوطة عليه.

**Locking:** `pctldev->mutex` + `desc->mux_lock` (per-pin).

**بيستدعيها:** قراءة `/sys/kernel/debug/pinctrl/<dev>/pinmux-pins`.

---

#### `pinmux_show_map`

```c
void pinmux_show_map(struct seq_file *s, const struct pinctrl_map *map);
```

**بتعمل إيه:** بتطبع محتوى `pinctrl_map` entry (group + function names).

**Locking:** لا شيء.

**بيستدعيها:** `pinctrl_maps_show` في `core.c`.

---

#### `pinmux_show_setting`

```c
void pinmux_show_setting(struct seq_file *s,
                         const struct pinctrl_setting *setting);
```

**بتعمل إيه:** بتطبع الـ setting الـ resolved (group name + number، function name + number).

**Locking:** لا شيء.

**بيستدعيها:** `pinctrl_show_setting` في `core.c`.

---

#### `pinmux_select_show` *(static)*

```c
static int pinmux_select_show(struct seq_file *s, void *unused);
```

**بتعمل إيه:** دايمًا بترجع `-EPERM`. الـ file ده write-only من ناحية القراءة.

---

#### `pinmux_select_write` *(static)*

```c
static ssize_t pinmux_select_write(struct file *file, const char __user *user_buf,
                                   size_t len, loff_t *ppos);
```

**بتعمل إيه:** بتسمح للـ developer يضبط mux يدويًا من debugfs. الـ format: `"<group_name> <function_name>\n"`. بتعمل parse، بتبحث، وبتستدعي `ops->set_mux` مباشرة.

**Parameters:**
- `user_buf` — buffer من الـ user space بالـ format المذكور
- `len` — الحجم

**Return:** `len` لو نجح، أو كود خطأ.

**Locking:** لا explicit locking — بتستخدم `pinmux_func_name_to_selector` و `pinctrl_get_group_selector` اللي مش thread-safe بدون lock خارجي.

**Side effects:** بتستدعي `ops->set_mux` مباشرة → hardware change. **خطيرة في production**.

**Pseudocode:**
```
pinmux_select_write(user_buf, len):
    buf = memdup_user_nul(user_buf, len)
    gname = strstrip(buf)           // first token = group name
    split at first space → fname    // second token = function name

    fsel = pinmux_func_name_to_selector(fname)
    get_function_groups(fsel) → groups[]
    match gname in groups[]
    gsel = pinctrl_get_group_selector(gname)
    ops->set_mux(fsel, gsel)        // direct hardware write
    kfree(buf)
    return len
```

**بيستدعيها:** كتابة على `/sys/kernel/debug/pinctrl/<dev>/pinmux-select`.

---

#### `pinmux_init_device_debugfs`

```c
void pinmux_init_device_debugfs(struct dentry *devroot,
                                struct pinctrl_dev *pctldev);
```

**بتعمل إيه:** بتنشئ الـ 3 debugfs files التالية تحت `devroot`:
- `pinmux-functions` (0444 — read only)
- `pinmux-pins` (0444 — read only)
- `pinmux-select` (0200 — write only)

**Locking:** لا شيء.

**بيستدعيها:** `pinctrl_init_device_debugfs` في `core.c`.

---

### Category 5: Generic Pinmux API — الـ `CONFIG_GENERIC_PINMUX_FUNCTIONS`

الـ functions دي بتوفر implementation جاهزة للـ `pinmux_ops` callbacks للـ drivers اللي بتستخدم `struct function_desc` + `radix_tree`.

```c
/* Generic function descriptor — used internally */
struct function_desc {
    const struct pinfunction *func;  /* name + groups + flags */
    void *data;                      /* driver-private data */
};
```

---

#### `pinmux_generic_get_function_count`

```c
int pinmux_generic_get_function_count(struct pinctrl_dev *pctldev);
```

**بتعمل إيه:** بترجع `pctldev->num_functions` مباشرة.

**Return:** عدد الـ functions المسجلة.

**Locking:** لا — الـ caller (pinmux core) بيمسك `pctldev->mutex`.

**Export:** `EXPORT_SYMBOL_GPL`.

---

#### `pinmux_generic_get_function_name`

```c
const char *pinmux_generic_get_function_name(struct pinctrl_dev *pctldev,
                                              unsigned int selector);
```

**بتعمل إيه:** بتعمل `radix_tree_lookup` بالـ selector وبترجع `function->func->name`.

**Return:** اسم الـ function أو `NULL` لو مش موجود.

**Locking:** الـ `radix_tree_lookup` بحاجة لـ RCU أو mutex من الـ caller.

**Export:** `EXPORT_SYMBOL_GPL`.

---

#### `pinmux_generic_get_function_groups`

```c
int pinmux_generic_get_function_groups(struct pinctrl_dev *pctldev,
                                       unsigned int selector,
                                       const char * const **groups,
                                       unsigned int * const ngroups);
```

**بتعمل إيه:** بتعمل `radix_tree_lookup` وبتعبي `*groups` و `*ngroups` من الـ `function->func`.

**Parameters:**
- `groups` — output: مصفوفة أسماء الـ groups
- `ngroups` — output: عدد الـ groups

**Return:** `0` أو `-EINVAL` لو الـ selector مش موجود.

**Export:** `EXPORT_SYMBOL_GPL`.

---

#### `pinmux_generic_get_function`

```c
const struct function_desc *pinmux_generic_get_function(
    struct pinctrl_dev *pctldev, unsigned int selector);
```

**بتعمل إيه:** بتعمل `radix_tree_lookup` وبترجع الـ `function_desc` كاملة (للـ driver اللي محتاج `->data`).

**Return:** pointer للـ `function_desc` أو `NULL`.

**Export:** `EXPORT_SYMBOL_GPL`.

---

#### `pinmux_generic_function_is_gpio`

```c
bool pinmux_generic_function_is_gpio(struct pinctrl_dev *pctldev,
                                     unsigned int selector);
```

**بتعمل إيه:** بتتحقق إن الـ function عندها `PINFUNCTION_FLAG_GPIO` في الـ `func->flags`.

**Return:** `true` لو GPIO function، `false` لو لأ أو مش موجودة.

**Export:** `EXPORT_SYMBOL_GPL`.

**ملاحظة:** الـ `pin_request` و `pinmux_can_be_used_for_gpio` بيستخدموا الـ callback `ops->function_is_gpio` اللي ممكن تشاور على الـ function دي.

---

#### `pinmux_generic_add_function`

```c
int pinmux_generic_add_function(struct pinctrl_dev *pctldev,
                                const char *name,
                                const char * const *groups,
                                const unsigned int ngroups,
                                void *data);
```

**بتعمل إيه:** wrapper بسيط بيبني `struct pinfunction` بـ `PINCTRL_PINFUNCTION(name, groups, ngroups)` وبيستدعي `pinmux_generic_add_pinfunction`.

**Return:** الـ selector الجديد (`>= 0`) أو كود خطأ.

**Export:** `EXPORT_SYMBOL_GPL`.

---

#### `pinmux_generic_add_pinfunction`

```c
int pinmux_generic_add_pinfunction(struct pinctrl_dev *pctldev,
                                   const struct pinfunction *func,
                                   void *data);
```

**بتعمل إيه:** الـ core function لإضافة function جديدة. لو الاسم موجود بالفعل → بترجع الـ selector الحالي. غير كده → بتخصص `function_desc` بـ `devm_kzalloc`، بتعمل `devm_kmemdup_const` للـ `pinfunction`، وبتسجلها في الـ `radix_tree`.

**Return:** الـ selector (index) الجديد أو `-ENOMEM`/error من الـ radix_tree.

**Memory:** كل الـ allocations بـ `devm_*` — بيتحرروا تلقائيًا لما الـ device يترجستر.

**Locking:** لا explicit locking — الـ caller مسئول.

**Pseudocode:**
```
pinmux_generic_add_pinfunction(pctldev, func, data):
    selector = pinmux_func_name_to_selector(func->name)
    if selector >= 0: return selector   // already exists

    selector = pctldev->num_functions

    function = devm_kzalloc(function_desc)
    function->func = devm_kmemdup_const(func)
    function->data = data

    radix_tree_insert(pin_function_tree, selector, function)
    pctldev->num_functions++
    return selector
```

**Export:** `EXPORT_SYMBOL_GPL`.

---

#### `pinmux_generic_remove_function`

```c
int pinmux_generic_remove_function(struct pinctrl_dev *pctldev,
                                   unsigned int selector);
```

**بتعمل إيه:** بتمسح function من الـ `radix_tree` وبتعمل `devm_kfree` للـ `function_desc`. **لا تحرر الـ `func` نفسها** لأنها `devm_kmemdup_const`.

**Return:** `0` أو `-ENOENT`.

**Locking:** الـ docstring صريح: "caller must take care of locking".

**Export:** `EXPORT_SYMBOL_GPL`.

---

#### `pinmux_generic_free_functions`

```c
void pinmux_generic_free_functions(struct pinctrl_dev *pctldev);
```

**بتعمل إيه:** بتمسح كل الـ entries من الـ `pin_function_tree` بـ `radix_tree_for_each_slot` وبتعمل reset للـ `num_functions = 0`. **لا تعمل `devm_kfree`** لأن الـ devm بيتكلف بيها.

**Locking:** الـ caller مسئول.

**بيستدعيها:** `pinctrl_unregister` في `core.c`.

---

### ملاحظات معمارية مهمة

| جانب | التفصيل |
|---|---|
| **Strict mode** | الـ `ops->strict = true` بيمنع مشاركة الـ pin بين mux owner وgpio owner. الـ non-strict controllers بيسمحوا بالـ overlap. |
| **`mux_usecount`** | بيسمح لنفس الـ owner يطلب نفس الـ pin أكتر من مرة (reference counting). |
| **`devm_*` في subsystem core** | الكود نفسه بيعلق على الـ FIXME — ده technical debt معروف في الـ pinctrl subsystem. |
| **Radix Tree للـ functions** | الـ `pin_function_tree` بيستخدم `radix_tree` بـ selector (integer index) كـ key للـ O(log n) lookup. |
| **`PINFUNCTION_FLAG_GPIO`** | الـ GPIO function بيتعرف بـ flag في `pinfunction.flags`، مش بـ convention في الاسم. |
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem ده هو **pinmux core** — المسؤول عن إنك تعرف أي pin بيتحول لأي function (UART، SPI، GPIO، إلخ). الـ bugs بتاعته صعبة لأنها بتظهر على شكل device مش شغال، مش error صريح.

---

### 1. debugfs — أول حاجة تفتحها

الـ kernel بيـregister ملفات تلاتة تحت `CONFIG_DEBUG_FS` (كود `pinmux_init_device_debugfs`):

```
/sys/kernel/debug/pinctrl/<controller-name>/
├── pinmux-functions    ← كل function وأي groups بتستخدمها
├── pinmux-pins         ← كل pin ومين owner بتاعه دلوقتي
└── pinmux-select       ← write-only: تقدر تعمل set_mux يدوي
```

#### عرض كل الـ functions المتاحة

```bash
cat /sys/kernel/debug/pinctrl/$(ls /sys/kernel/debug/pinctrl/ | head -1)/pinmux-functions
```

**مثال output:**
```
function 0: gpio, groups = [ gpio0 gpio1 gpio2 ]
function 1: uart0, groups = [ uart0_grp ]
function 2: spi0, groups = [ spi0_grp spi0_cs_grp ]
```

#### عرض حالة كل pin

```bash
cat /sys/kernel/debug/pinctrl/$(ls /sys/kernel/debug/pinctrl/ | head -1)/pinmux-pins
```

**مثال output (strict controller):**
```
Pinmux settings per pin
Format: pin (name): mux_owner|gpio_owner (strict) hog?
pin 0 (PA0): device serial0 (HOG) function uart0 group uart0_grp
pin 1 (PA1): device serial0 function uart0 group uart0_grp
pin 2 (PA2): GPIO gpiochip0:2
pin 3 (PA3): UNCLAIMED
```

**مثال output (non-strict controller):**
```
Format: pin (name): mux_owner gpio_owner hog?
pin 0 (PA0): serial0 (GPIO UNCLAIMED) function uart0 group uart0_grp
pin 4 (PA4): (MUX UNCLAIMED) gpiochip0:4
```

#### تغيير الـ mux يدوياً عبر pinmux-select

```bash
# format: "<group_name> <function_name>"
echo "uart0_grp uart0" > /sys/kernel/debug/pinctrl/<ctrl>/pinmux-select
```

> الملف write-only — القراءة منه بترجع `-EPERM` عن قصد (السطر 712 في الكود).

---

### 2. sysfs — فحص سريع

```bash
# لست كل الـ pin controllers المتسجلة
ls /sys/bus/platform/drivers/pinctrl/

# فحص الـ device state
cat /sys/class/gpio/export   # لو الـ pin اتطلب كـ GPIO
```

---

### 3. ftrace / Tracepoints

الـ pinmux core نفسه ما عندوش tracepoints مخصصة، بس تقدر تـtrace الـ function calls:

```bash
# تفعيل function tracer على دوال الـ pinmux
echo 'pinmux_enable_setting' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 'pin_request' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'pinmux_request_gpio' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ device / GPIO request اللي بيـfail
# بعدين:
cat /sys/kernel/debug/tracing/trace
```

**مثال output:**
```
# tracer: function
#
   kworker/0:1-45    [000] .... 1234.567890: pin_request <-pinmux_enable_setting
   kworker/0:1-45    [000] .... 1234.567891: pinmux_enable_setting <-pinctrl_commit_state
```

#### Tracepoints مفيدة من pinctrl core

```bash
ls /sys/kernel/debug/tracing/events/pinctrl/
# مثال:
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable
```

---

### 4. printk / Dynamic Debug

الكود بيستخدم `dev_dbg` و`dev_err` — الـ `dev_dbg` مش ظاهرة بـdefault.

#### تفعيل dynamic debug لـ pinmux

```bash
# تفعيل كل رسايل debug في pinmux.c
echo 'file pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو بالـ module كله
echo 'module pinctrl +p' > /sys/kernel/debug/dynamic_debug/control
```

**بعد التفعيل هتشوف في dmesg:**
```
[ 12.345678] pinmux core: request pin 5 (PA5) for spi0
[ 12.345690] pinmux core: pin PA5 already requested by serial0; cannot claim for spi0
```

#### تفعيل early boot (من kernel cmdline)

```
dyndbg="file pinmux.c +p"
```

---

### 5. CONFIG_DEBUG_* — خيارات الـ Kernel

| Config | الأثر |
|--------|-------|
| `CONFIG_DEBUG_FS` | يفعّل ملفات `pinmux-functions`, `pinmux-pins`, `pinmux-select` |
| `CONFIG_PINMUX` | الـ subsystem نفسه — بدونه كل الدوال stubs فارغة |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | يفعّل `pinmux_generic_*` API والـ radix tree |
| `CONFIG_DYNAMIC_DEBUG` | يتيح `dev_dbg` التفعيل runtime |
| `CONFIG_PROVE_LOCKING` | يكتشف deadlocks في `desc->mux_lock` |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | يكتشف لو `mutex_lock` اتنادى في atomic context |

---

### 6. جدول رسايل الـ Error الشائعة

| الرسالة | الدالة | السبب | الحل |
|---------|--------|-------|-------|
| `pinmux ops lacks necessary functions` | `pinmux_check_ops` | driver ناقص `get_functions_count` أو `set_mux` | اكمّل الـ `pinmux_ops` struct |
| `pin %d is not registered so it cannot be requested` | `pin_request` | الـ pin_desc مش موجود | تأكد إن الـ pin اترجستر في `pinctrl_register_pins` |
| `pin %s already requested by %s; cannot claim for %s` | `pin_request` | conflict بين owner حالي وطالب جديد | افحص مين بيحتفظ بالـ pin عبر `pinmux-pins` |
| `invalid function %s in map table` | `pinmux_map_to_setting` / `pinmux_select_write` | اسم الـ function غلط أو مش معرّف | قارن بـoutput الـ `pinmux-functions` |
| `function %s can't be selected on any group` | `pinmux_map_to_setting` | الـ function عنده `ngroups = 0` | راجع driver الـ `get_function_groups` |
| `invalid group "%s" for function "%s"` | `pinmux_map_to_setting` | الـ group مش في قائمة الـ function | تأكد من اسم الـ group في الـ DT أو pinctrl map |
| `could not get pins for group %s` | `pinmux_enable_setting` | `get_group_pins` فشلت | **debug data فقط** — يـwarn مش يـfail |
| `could not request pin %d (%s) from group %s on device %s` | `pinmux_enable_setting` | فشل `pin_request` لأحد الـ pins في الـ group | راجع الـ pin ownership |
| `pin is not registered so it cannot be freed` | `pin_free` | محاولة free لـpin مش موجود | bug في الـ driver — balance الـ request/free |
| `does not support mux function` | `pinmux_map_to_setting` | `pmxops` بـ NULL | الـ driver ما implmentش الـ pinmux ops |
| `could not increase module refcount for pin %d` | `pin_request` | الـ module بدأ يـunload | race condition — راجع lifecycle الـ module |
| `%s could not find function%i` | `pinmux_generic_get_function_groups` | selector خارج نطاق الـ radix tree | bug في الـ driver أو index غلط |

---

### 7. WARN_ON / dump_stack في الكود

#### WARN_ON في pin_free (السطر 252)

```c
if (WARN_ON(!desc->mux_usecount))
    return NULL;
```

**ده بيظهر لو:**
- الـ driver عمل `pin_free` أكتر من ما عمل `pin_request`
- unbalanced call flow

**الـ output في dmesg:**
```
WARNING: CPU: 0 PID: 1234 at drivers/pinctrl/pinmux.c:252 pin_free+0x...
Modules linked in: ...
Call Trace:
  pin_free+0x...
  pinmux_free_gpio+0x...
  gpiod_free+0x...
```

**الحل:** تتبع الـ call chain وتتأكد إن كل `pinmux_request_gpio` عنده `pinmux_free_gpio` مقابلها.

---

### 8. Hardware State vs Kernel State

ده أخطر نوع bug: الـ kernel عارف إن الـ pin لـfunction معينة، بس الـ hardware register مختلف.

#### السبب الشائع:
- bootloader عمل mux setting معين ولو الـ kernel ما فعّلش `set_mux` تاني، الـ hardware بيفضل على الـ bootloader setting
- الـ kernel state بيتحدث في `pinmux_enable_setting` عند استدعاء `ops->set_mux` — لو الدالة دي ما اتنادتش، في فرق

#### Verify بالـ register dump (مثال ARM SoC)

```bash
# لو عندك devmem
devmem2 0x10A00000 w  # عنوان الـ IOMUX register

# أو عبر /dev/mem
python3 -c "
import mmap, struct
with open('/dev/mem','rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0x10A00000)
    val = struct.unpack('<I', m.read(4))[0]
    print(hex(val))
"
```

**قارن القيمة بالـ datasheet** — الـ bits المسؤولة عن mux selection لازم تطابق الـ function اللي مضبوط في `pinmux-pins`.

#### Kernel state صح لكن HW غلط — السيناريو:

```
pinmux-pins يقول: pin 5 → uart0  ✓
dmesg يقول: set_mux called OK   ✓
المنفذ UART مش شغال              ✗
```

**الخطوة:** dump الـ register الفعلي وشوف لو الـ bits اتكتبت صح.

---

### 9. Device Tree Debugging

```bash
# تحقق إن الـ DT node اتفسر صح
cat /proc/device-tree/soc/pinctrl@10A00000/uart0_pins/function
# المفروض يطبع: "uart0"

cat /proc/device-tree/soc/pinctrl@10A00000/uart0_pins/groups
# المفروض يطبع: "uart0_grp"

# فحص الـ pinctrl-0 في الـ consumer device
cat /proc/device-tree/soc/serial@10000000/pinctrl-names
# المفروض: "default"
```

**لو الـ node مش موجود أو القيم غلط** — الـ `pinmux_validate_map` هتـfail وهتشوف:
```
pinctrl core: failed to register map uart0 (0): no function given
```

#### تحقق من الـ state الـ active

```bash
cat /sys/kernel/debug/pinctrl/<ctrl>/pinctrl-handles
# بيـlist كل الـ pinctrl handles الـ active وstates بتاعتها
```

---

### 10. أوامر تشخيص شاملة (Cheat Sheet)

```bash
CTRL=$(ls /sys/kernel/debug/pinctrl/ | head -1)
BASE="/sys/kernel/debug/pinctrl/$CTRL"

# 1. شوف كل الـ functions
cat "$BASE/pinmux-functions"

# 2. شوف حالة كل pin
cat "$BASE/pinmux-pins"

# 3. شوف الـ pin groups
cat "$BASE/pinctrl-devices"
cat "$BASE/pingroups"

# 4. شوف الـ active pinctrl consumers
cat "$BASE/pinctrl-handles"

# 5. فعّل dynamic debug
echo 'file pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file pinctrl-core.c +p' >> /sys/kernel/debug/dynamic_debug/control

# 6. اعمل mux يدوي للاختبار
echo "uart0_grp uart0" > "$BASE/pinmux-select"

# 7. تتبع requests في real-time
echo 'pin_request' > /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace_pipe &

# 8. شوف الـ kernel log بـtimestamp
dmesg -T | grep -i "pinmux\|pinctrl\|pin.*request\|already requested"
```

---

### 11. سيناريو عملي: GPIO Request فشل

**الخطأ في dmesg:**
```
gpio gpiochip0: (sysfs): gpiod_request: pin 10 already requested by serial0; cannot claim for gpiochip0:10
```

**خطوات التشخيص:**

```bash
# 1. افحص مين بيمسك الـ pin
CTRL=$(ls /sys/kernel/debug/pinctrl/ | head -1)
grep "pin 10" /sys/kernel/debug/pinctrl/$CTRL/pinmux-pins

# output المتوقع:
# pin 10 (PA10): device serial0 function uart0 group uart0_grp

# 2. هل الـ function فعلاً GPIO؟
grep "function_is_gpio" /sys/kernel/debug/pinctrl/$CTRL/pinmux-functions
# لو strict=true والـ function مش gpio → pin_request هترفض

# 3. فحص الـ DT: هل الـ serial0 بيحجز الـ pin بشكل صح؟
cat /proc/device-tree/soc/serial@.../pinctrl-0

# 4. الحل: أزل الـ pin من uart0_grp في الـ DT أو استخدم pin تاني للـ GPIO
```

**الكود اللي بيتسبب في الـ error (السطر 154):**
```c
dev_err(pctldev->dev,
    "pin %s already requested by %s; cannot claim for %s\n",
    desc->name, desc->mux_owner, owner);
```

**الشرط:** `ops->strict == true` أو مفيش `gpio_range` + الـ `mux_usecount > 0` + owner مختلف.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: RK3562 — UART2 مش شغال بعد boot

#### السياق
بورد مبني على **Rockchip RK3562**، بيستخدم `UART2` كـ debug console. المطور عمل custom DTS وبعد الـ boot مفيش output على serial.

#### المشكلة
الـ UART2 مش بيشتغل رغم إن الـ driver موجود وبيـ probe بنجاح.

#### التحليل (trace خلال pinmux.c)

1. الـ DTS فيه `pinctrl-0` بيشاور على state اسمه `"uart2-default"`.
2. الـ core بيستدعي `pinmux_map_to_setting()` → بتشيل الـ `map->data.mux.function` (اسمه `"uart2"`) وتديه لـ `pinmux_func_name_to_selector()`.
3. `pinmux_func_name_to_selector()` بتعمل loop على كل الـ functions عن طريق `ops->get_function_name()` وبتقارن الأسماء.
4. **السبب**: المطور سمى الـ function في الـ pinctrl driver `"UART2"` بحروف كبيرة، بس في الـ map مكتوب `"uart2"` — الـ `strcmp` case-sensitive فرجع `-EINVAL`.
5. الـ log بيظهر:

```
pinmux core: invalid function uart2 in map table
```

#### الحل

```c
/* في rockchip pinctrl driver */
static const char *rk3562_pmx_get_func_name(struct pinctrl_dev *pctldev,
                                             unsigned selector)
{
    /* تأكد إن الاسم lowercase ومتطابق مع الـ DTS */
    return rk3562_functions[selector].name; /* "uart2" مش "UART2" */
}
```

تأكد من تطابق الأسماء بين `pinmux_ops->get_function_name()` والـ `pinctrl-map` في الـ DTS.

#### الدرس المستفاد
**`pinmux_func_name_to_selector()`** بتعمل `strcmp` عادي case-sensitive. أي mismatch في الاسم بين الـ driver والـ DTS بيخلي الـ mux يفشل صامت من وجهة نظر المستخدم.

---

### السيناريو 2: STM32MP1 — SPI1 و GPIO يتعارضوا في runtime

#### السياق
على **STM32MP157**، الـ SPI1 بيستخدم pins `PA4-PA7`. تطبيق userspace بيحاول يفتح `/sys/class/gpio/gpio4` في نفس الوقت اللي الـ SPI driver شغال.

#### المشكلة
الـ GPIO export بيفشل بـ error، أو أسوأ: الـ SPI بيتعطل فجأة.

#### التحليل (trace خلال pinmux.c)

1. عند load الـ SPI driver: `pinmux_enable_setting()` → بتستدعي `pin_request()` على كل pin في group `"spi1-grp"`.
2. في `pin_request()`:
   - بتشيل `desc->mux_usecount++`
   - `desc->mux_owner = "spi1"`
   - بتستدعي `ops->request(pctldev, pin)` أو `ops->gpio_request_enable()`
3. لما userspace يعمل GPIO export: `pinmux_request_gpio()` → `pin_request(..., gpio_range)` → بتيجي على الـ check:

```c
if ((gpio_range || ops->strict) && !gpio_ok && desc->gpio_owner) {
    dev_err(..., "pin %s already requested by %s; cannot claim for %s\n", ...);
    goto out;
}
```

4. لو `ops->strict = true` في الـ STM32 pinctrl driver → `pinmux_can_be_used_for_gpio()` بترجع `false` لأن:

```c
if (ops->strict && desc->mux_usecount && !func_is_gpio)
    return false;
```

#### الحل

```
/* في DTS: اعمل pinctrl state منفصل */
&spi1 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&spi1_pins>;
    pinctrl-1 = <&spi1_sleep_pins>; /* release pins عند suspend */
};
```

مش تحاول تشارك نفس الـ pin بين SPI function وGPIO وانت `strict = true`. استخدم `pinctrl-select` في debugfs لو بتعمل تجارب.

#### الدرس المستفاد
الـ **`strict` flag** في `pinmux_ops` بيمنع أي مشاركة بين `mux_owner` و`gpio_owner` على نفس الـ pin. الـ STM32 بيستخدمه لحماية integrity الـ hardware.

---

### السيناريو 3: i.MX8MQ — I2C3 بيـ probe لكن الـ device مش بيرد

#### السياق
على **NXP i.MX8MQ**، I2C3 متصل بـ PMIC. الـ driver بيـ probe، الـ bus بيتعرف، لكن كل transaction بيرجع `-ETIMEDOUT`.

#### المشكلة
الـ pins مش متـ mux صح للـ I2C function رغم إن الـ DTS صح ظاهريًا.

#### التحليل (trace خلال pinmux.c)

1. `pinmux_map_to_setting()` بتنجح: اتحول `"i2c3"` لـ selector صح.
2. `pinmux_enable_setting()` → بتستدعي `pctlops->get_group_pins()` → جيبت pins `[72, 73]` (SCL, SDA).
3. Loop في `pin_request()` للـ pin 72 → نجح. للـ pin 73 → **فشل** لأن:

```c
if ((!gpio_range || ops->strict) && !gpio_ok &&
    desc->mux_usecount && strcmp(desc->mux_owner, owner)) {
    dev_err(..., "pin %s already requested by %s; cannot claim for %s\n", ...);
    goto out;
}
```

الـ pin 73 كان **مطلوب قبل كدا** من driver تاني (مثلاً `pwm` driver اتـ load قبل الـ I2C).

4. الـ rollback في `err_pin_request`:

```c
while (--i >= 0)
    pin_free(pctldev, pins[i], NULL);
```

بيعمل free للـ pin 72 بس، والـ mux register في الـ hardware فضل على القيمة الافتراضية (GPIO mode).

#### الحل

تأكد من **ترتيب الـ drivers** في الـ DTS. الـ PWM اللي بياخد pin 73 لازم يـ release الـ pin قبل الـ I2C أو تغير الـ pinmux group. استخدم debugfs:

```bash
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pinmux-pins
# شوف مين مـ claiming pin 73
```

#### الدرس المستفاد
**`pinmux_enable_setting()`** بتعمل atomic rollback لو أي pin في الـ group فشل. الـ hardware بيفضل unchanged. لازم تتأكد إن مفيش driver بياخد pins بشكل مخفي قبل driver تاني.

---

### السيناريو 4: AM62x — USB0 بيتحول لـ GPIO بغلطة في runtime

#### السياق
على **TI AM62x** (BeaglePlay board)، الـ USB0 شغال تمام. بعد تشغيل script بيعمل GPIO toggling على port معين، الـ USB بييجي disconnected.

#### المشكلة
سكريبت الـ GPIO بيعمل `export` لـ GPIO line بيشارك نفس pin الـ USB VBUS enable.

#### التحليل (trace خلال pinmux.c)

1. الـ USB driver اتـ load: `pinmux_enable_setting()` → `pin_request()` على group `"usb0-pins"`.
   - `desc->mux_usecount = 1`, `desc->mux_owner = "usb0"`
2. السكريبت عمل `echo 42 > /sys/class/gpio/export`:
   - `pinmux_request_gpio()` → `pin_request(..., gpio_range)`.
3. الـ AM62x pinctrl driver **مش strict** (`ops->strict = false`) → الـ check ده بيعدي:

```c
if (ops->strict && desc->mux_usecount && !func_is_gpio)
    return false; /* مش بيتنفذ لأن strict=false */
```

4. `desc->gpio_owner = "GPIO:42"` اتكتب بنجاح.
5. `ops->gpio_request_enable()` اتنفذ → غير الـ mux register للـ GPIO mode في الـ hardware.
6. الـ VBUS enable signal وقف → USB disconnected.

#### الحل

لازم TI driver يضبط `ops->strict = true` أو يعمل `function_is_gpio()` callback صح. أو على مستوى الـ DTS:

```dts
/* في am62x-beagleplay.dts */
&usb0 {
    /* pin 42 ميعملوش export في userspace */
    /* استخدم gpio-hog بدل userspace export */
    pinctrl-0 = <&usb0_pins>;
};
```

استخدام **gpio-hog** بدل userspace export بيعمل `pin_request()` من kernel space وبيتحكم في الـ ownership بشكل أفضل.

#### الدرس المستفاد
الـ **`strict` flag** في `pinmux_ops` هو الفاصل بين "نظام محمي" و"نظام ممكن يتعطل بغلطة من userspace". بدونه، `gpio_owner` ممكن يتكتب فوق `mux_owner` ويغير الـ hardware state من تحت الـ driver الأصلي.

---

### السيناريو 5: Allwinner H616 — HDMI مش بيظهر صورة بعد kernel update

#### السياق
على **Allwinner H616** (Orange Pi Zero 2)، HDMI كان شغال. بعد update لـ kernel أحدث، الشاشة فضت سودا رغم إن `modetest` بيشوف الـ connector.

#### المشكلة
الـ kernel الجديد فيه pinctrl driver محدث أضاف function جديدة، وغير الـ selector index للـ HDMI function.

#### التحليل (trace خلال pinmux.c)

1. `pinmux_check_ops()` اتـ call عند register الـ driver → نجح (كل الـ ops موجودة).
2. `pinmux_validate_map()` نجحت: الـ map فيها `function = "hdmi"`.
3. `pinmux_func_name_to_selector()` → loop على functions:

```c
while (selector < nfuncs) {
    const char *fname = ops->get_function_name(pctldev, selector);
    if (fname && !strcmp(function, fname))
        return selector;
    selector++;
}
```

4. في الـ kernel القديم: `"hdmi"` كان selector `5`. في الجديد: أضافوا function جديدة في البداية فـ `"hdmi"` بقى selector `6`.
5. الـ **radix tree** في `pinmux_generic_get_function()` بيرجع الـ function الصح بالاسم — **مفيش مشكلة هنا**.
6. المشكلة الحقيقية: الـ DTS القديم كان بيستخدم `pinmux_select` في debugfs بـ hardcoded selector number من script قديم:

```bash
echo "hdmi-grp hdmi" > /sys/kernel/debug/pinctrl/.../pinmux-select
```

الـ `pinmux_select_write()` بتستدعي `pinmux_func_name_to_selector()` → بتيجي بالـ selector الصح → بتستدعي `pmxops->set_mux()`. **الـ script ده كان OK**.

7. المشكلة الفعلية: `pinmux_enable_setting()` نجحت في إن الـ pins اتطلبت، لكن `ops->set_mux()` في الـ H616 driver فيه bug: بتقرأ الـ selector وتحسب الـ register offset غلط بعد إضافة الـ function الجديدة — **bug في الـ driver نفسه مش في الـ core**.

8. التأكيد عن طريق debugfs:

```bash
cat /sys/kernel/debug/pinctrl/300b000.pinctrl/pinmux-pins
# Pin 120 (PH0): UNCLAIMED  ← المفروض يكون hdmi
```

#### الحل

```c
/* في sunxi pinctrl driver — set_mux fix */
static int sunxi_pmx_set_mux(struct pinctrl_dev *pctldev,
                              unsigned function, unsigned group)
{
    /* استخدم function name مش selector index لحساب الـ register value */
    const char *fname = ops->get_function_name(pctldev, function);
    u32 val = sunxi_get_mux_val_by_name(fname); /* lookup by name */
    /* ... */
}
```

**وفي الـ debugfs للتشخيص**:

```bash
# تحقق إن الـ pins اتطلبت
cat /sys/kernel/debug/pinctrl/300b000.pinctrl/pinmux-functions | grep hdmi

# جرب set manual
echo "hdmi-grp hdmi" > /sys/kernel/debug/pinctrl/300b000.pinctrl/pinmux-select
```

#### الدرس المستفاد
الـ **`pinmux_generic_add_function()`** بتخزن الـ functions في **radix tree** بـ sequential selector. الـ core بيستخدم الأسماء (strings) للـ lookup، مش hardcoded indices. لو الـ driver نفسه بيستخدم الـ selector كـ magic number في `set_mux()` بدل ما يعمل name-based lookup، أي إضافة function جديدة بتكسر كل الـ functions اللي بعدها.
## Phase 7: مصادر ومراجع

### توثيق رسمي — kernel.org

- **PINCTRL subsystem — driver API docs (v4.14)**
  `https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html`
  الصفحة الرئيسية للـ subsystem: pinmux، pinconf، pin grouping، GPIO integration.

- **Documentation/pinctrl.txt — kernel.org**
  `https://www.kernel.org/doc/Documentation/pinctrl.txt`
  النص الأصلي للـ design doc اللي كتبه **Linus Walleij** — شرح `pinmux_ops`، `set_mux`، `gpio_request_enable`.

- **devicetree/bindings/pinctrl/pinmux-node.yaml**
  `https://mjmwired.net/kernel/Documentation/devicetree/bindings/pinctrl/pinmux-node.yaml`
  مواصفات Device Tree لـ pinmux nodes — `function`، `groups`، `pins`.

---

### مقالات LWN.net

| العنوان | الرابط | الأهمية |
|---|---|---|
| **The pin control subsystem** | [lwn.net/Articles/468759](https://lwn.net/Articles/468759/) | نظرة معمارية شاملة — أساسي |
| **Documentation/pinctrl.txt** (early RFC) | [lwn.net/Articles/465077](https://lwn.net/Articles/465077/) | أول نسخة من الـ doc — تاريخي |
| **pin controller subsystem v7** | [lwn.net/Articles/459190](https://lwn.net/Articles/459190/) | مرحلة الـ RFC قبل merge في mainline |
| **pinctrl: add a generic pin config interface** | [lwn.net/Articles/468770](https://lwn.net/Articles/468770/) | إضافة `pinconf` بجانب `pinmux` |
| **pinctrl: add a pin config interface** | [lwn.net/Articles/471826](https://lwn.net/Articles/471826/) | تفاصيل تطبيق `pinctrl_ops` |
| **pinctrl: introduce GPIO pin function category** | [lwn.net/Articles/1031226](https://lwn.net/Articles/1031226/) | `PINFUNCTION_FLAG_GPIO` و `function_is_gpio` — مباشر للكود |
| **pinctrl: Add generic pinctrl-simple driver** | [lwn.net/Articles/496075](https://lwn.net/Articles/496075/) | مثال عملي — `CONFIG_PINCTRL_SINGLE` |
| **pinctrl: Add new pinctrl/GPIO driver** | [lwn.net/Articles/803863](https://lwn.net/Articles/803863/) | نمط كتابة driver حديث |

---

### eLinux.org

- **Pin Control & GPIO update (PDF)**
  `https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf`
  عرض تقديمي — تكامل `pinctrl` مع `gpio subsystem`، `strict mode`، `gpio_owner`.

- **EBC Device Trees**
  `https://elinux.org/EBC_Device_Trees`
  أمثلة DTS عملية على `pinctrl-single` و pin group configuration.

- **Tests: i2c-demux-pinctrl**
  `https://www.elinux.org/Tests:i2c-demux-pinctrl`
  اختبار `pinmux_request_gpio` / `pinmux_free_gpio` في سياق `i2c-demux`.

---

### kernelnewbies.org — تتبع التغييرات

- **Linux_6.11** — `https://kernelnewbies.org/Linux_6.11`
- **Linux_6.14** — `https://kernelnewbies.org/Linux_6.14`
- **Linux_6.15** — `https://kernelnewbies.org/Linux_6.15`
- **Linux_6.17** — `https://kernelnewbies.org/Linux_6.17`
- **Linux_6.18** — `https://kernelnewbies.org/Linux_6.18`

كل صفحة بتحتوي على قسم **pinctrl** بيسرد الـ drivers الجديدة والـ API changes لكل kernel release.

---

### Source Files — الكود المدروس

| الملف | الوصف |
|---|---|
| `drivers/pinctrl/pinmux.c` | core implementation — `pin_request`، `pin_free`، `pinmux_enable_setting`، `pinmux_generic_*` |
| `drivers/pinctrl/pinmux.h` | internal interface — `function_desc`، stub inlines لـ `CONFIG_PINMUX=n` |

#### دوال رئيسية في `pinmux.c`

```c
/* التحقق من صحة pinmux_ops قبل تسجيل الـ controller */
int pinmux_check_ops(struct pinctrl_dev *pctldev);

/* طلب pin واحد للـ GPIO أو function */
static int pin_request(struct pinctrl_dev *pctldev, int pin,
                       const char *owner, struct pinctrl_gpio_range *gpio_range);

/* تفعيل mux setting كامل (function + group) */
int pinmux_enable_setting(const struct pinctrl_setting *setting);

/* GENERIC_PINMUX_FUNCTIONS — إدارة functions بـ radix_tree */
int pinmux_generic_add_pinfunction(struct pinctrl_dev *pctldev,
                                   const struct pinfunction *func, void *data);
```

---

### Commits ذات صلة — git.kernel.org

- **أول commit للـ pinmux subsystem (v3.0 era)**
  Author: Linus Walleij `<linus.walleij@linaro.org>`
  Tree: `git://git.kernel.org/pub/scm/linux/kernel/git/linusw/linux-pinctrl.git`

- **إضافة `function_is_gpio` و `PINFUNCTION_FLAG_GPIO`**
  مرجع: [lwn.net/Articles/1031226](https://lwn.net/Articles/1031226/) — يتعلق بـ `pinmux_can_be_used_for_gpio()` و `pinmux_generic_function_is_gpio()`.

- **Merge tag pinctrl-v4.1**
  `https://kernel.googlesource.com/pub/scm/linux/kernel/git/torvalds/linux/+/07e492eb8921a8aa53fd2bf637bee3da94cc03fe`

- **GitHub mirror للكود الحالي**
  `https://github.com/torvalds/linux/blob/master/drivers/pinctrl/pinmux.c`
  `https://github.com/torvalds/linux/blob/master/drivers/pinctrl/pinmux.h`

---

### كتب مرجعية

| الكتاب | الصلة بالموضوع |
|---|---|
| **Linux Device Drivers (LDD3)** — Corbet, Rubini, Kroah-Hartman | Chapter على GPIO و character devices — قاعدة لفهم `pin_request` ownership model |
| **Linux Kernel Development** — Robert Love (3rd ed.) | فصول kernel data structures: `radix_tree`، `mutex`، `module_get` — مستخدمة مباشرة في `pinmux.c` |
| **Embedded Linux Primer** — Christopher Hallinan (2nd ed.) | Chapter 17: BSP و pin muxing على ARM platforms — سياق عملي لـ `pinmux_map_to_setting` |
| **Mastering Embedded Linux Programming** — Frank Vasquez & Chris Simmonds (3rd ed.) | Device Tree و pinctrl integration — `pinctrl-names`، `pinctrl-0` في DTS |

---

### Search Terms مفيدة

```
pinmux_ops set_mux kernel
pinctrl_setting mux group function kernel
pin_request mux_owner gpio_owner kernel
pinmux_enable_setting pin_free kernel
CONFIG_GENERIC_PINMUX_FUNCTIONS
CONFIG_PINMUX strict mode
function_desc radix_tree pinctrl
pinmux_can_be_used_for_gpio strict
PINFUNCTION_FLAG_GPIO function_is_gpio
debugfs pinmux-functions pinmux-pins pinmux-select
```

---

### مواد إضافية

- **embedded.com — Linux pin control subsystem**
  `https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/`
  شرح عملي لـ `pinmux_ops` callbacks مع أمثلة driver حقيقية.

- **Linaro git — linux-pinctrl (Linus Walleij)**
  `https://git.linaro.org/people/linus.walleij/linux-pinctrl.git`
  الـ development tree — للـ patches قبل mainline merge.

- **Android kernel MSM — pinmux.c (مقارنة)**
  `https://android.googlesource.com/kernel/msm/+/refs/heads/android-msm-crosshatch-4.9-pie-qpr2/drivers/pinctrl/pinmux.c`
  نسخة قديمة مفيدة لرؤية تطور الـ API.
## Phase 8: Writing simple module

### الفكرة العامة

هنعمل kernel module بيعمل **kprobe** على الدالة `pinmux_enable_setting` — وهي الدالة اللي بتتفعّل كل ما device بدأ يستخدم mux setting جديد على مجموعة pins. الـ kprobe هيطبع اسم الـ device اللي طلب الـ mux وأرقام الـ func وال group اللي اتختارت.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0-only
/*
 * kprobe_pinmux_enable.c
 * Hook pinmux_enable_setting() to trace mux activations.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/pinctrl/pinctrl.h>  /* struct pinctrl_dev, pinctrl_dev_get_name */

/* Forward-declare the internal struct we need — mirror from pinctrl core */
struct pinctrl_setting_mux {
    unsigned int group;
    unsigned int func;
};

/* pinctrl_setting is an internal struct; we only need the mux sub-field.
 * We redefine a minimal shadow to avoid pulling in private headers. */
struct pinctrl_setting_shadow {
    struct pinctrl_dev  *pctldev;   /* offset 0 — pointer to pin controller  */
    unsigned int         type;      /* setting type (mux = 0)                */
    const char          *dev_name;  /* device that owns this setting         */
    struct pinctrl_setting_mux data_mux; /* func + group selectors           */
};

/* ------------------------------------------------------------------ */
/* kprobe handler — called just before pinmux_enable_setting() runs    */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument lands in rdi.
     * On arm64 it is in x0.
     * We cast it to our shadow struct to read the fields we care about.
     */
#if defined(CONFIG_X86_64)
    const struct pinctrl_setting_shadow *setting =
            (const struct pinctrl_setting_shadow *)regs->di;
#elif defined(CONFIG_ARM64)
    const struct pinctrl_setting_shadow *setting =
            (const struct pinctrl_setting_shadow *)regs->regs[0];
#else
    /* Generic fallback — may not work on all arches */
    const struct pinctrl_setting_shadow *setting = NULL;
#endif

    if (!setting || !setting->pctldev)
        return 0;   /* safety: skip if pointer looks bad */

    pr_info("[pinmux_probe] enable_setting: dev=%s ctrl=%s func=%u group=%u\n",
            setting->dev_name   ? setting->dev_name   : "(null)",
            pinctrl_dev_get_name(setting->pctldev),
            setting->data_mux.func,
            setting->data_mux.group);

    return 0; /* 0 = continue normal execution */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                    */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "pinmux_enable_setting", /* function to hook         */
    .pre_handler = handler_pre,             /* called before the func   */
};

/* ------------------------------------------------------------------ */
/* module_init                                                          */
/* ------------------------------------------------------------------ */
static int __init pinmux_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[pinmux_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[pinmux_probe] kprobe planted on %s at %p\n",
            kp.symbol_name, kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                          */
/* ------------------------------------------------------------------ */
static void __exit pinmux_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[pinmux_probe] kprobe removed from %s\n", kp.symbol_name);
}

module_init(pinmux_probe_init);
module_exit(pinmux_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc session");
MODULE_DESCRIPTION("kprobe on pinmux_enable_setting — traces mux activations");
```

---

### شرح كل جزء

#### `struct pinctrl_setting_shadow`
بما إن `struct pinctrl_setting` مش exported للـ modules، عملنا **shadow struct** بنفس ترتيب الـ fields اللي محتاجينها (`pctldev`, `dev_name`, `data.mux.func`, `data.mux.group`) عشان نقدر نقرا البيانات من الـ pointer اللي بييجي argument.

#### `handler_pre`
ده الـ **pre-handler** — بيتشغّل قبل ما `pinmux_enable_setting` تبدأ. بيسحب الـ argument الأول من الـ registers (رجع لـ calling convention لكل architecture)، وبعدين بيطبع:
- اسم الـ device اللي طلب الـ mux (`dev_name`)
- اسم الـ pin controller (`pinctrl_dev_get_name`)
- رقم الـ function selector (`func`)
- رقم الـ group selector (`group`)

#### `struct kprobe kp`
بيحدد الـ **symbol** اللي هنحطّ عليه الـ probe (`pinmux_enable_setting`) والـ handler اللي هيتشغّل. مش محتاجين `post_handler` هنا لأن البيانات كلها متاحة في الـ arguments قبل الدخول.

#### `pinmux_probe_init`
بيعمل **register_kprobe** لتركيب الـ hook في الـ kernel. لو فشل (مثلاً الـ symbol مش موجود أو `CONFIG_KPROBES` مش مفعّل) بيرجع الـ error code.

#### `pinmux_probe_exit`
بيعمل **unregister_kprobe** عشان يشيل الـ hook بنظافة لما الـ module يتشال من الـ kernel.

---

### Makefile مبسّط

```makefile
obj-m += kprobe_pinmux_enable.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### إزاي تجرّبه

```bash
# تحميل الـ module
sudo insmod kprobe_pinmux_enable.ko

# شوف الـ output في dmesg
dmesg | grep pinmux_probe

# تفريغ أي device يعمل pinmux (مثلاً USB أو GPIO)
# وهتشوف سطر زي:
# [pinmux_probe] enable_setting: dev=soc:pinctrl ctrl=pinctrl-bcm2835 func=3 group=7

# إزالة الـ module
sudo rmmod kprobe_pinmux_enable
```

---

### ليه `pinmux_enable_setting` تحديداً؟

لأنها **النقطة المركزية** في الـ pinmux subsystem — كل request لـ mux setting جديد (من device driver أو GPIO subsystem) لازم يمرّ بيها. الـ hook عليها بيديك رؤية كاملة لكل عملية mux activation في الـ system بدون ما تعدّل أي سطر في الـ kernel نفسه.
