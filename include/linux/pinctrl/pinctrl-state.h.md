## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem: PIN CONTROL SUBSYSTEM

الـ `pinctrl-state.h` جزء من **PIN CONTROL SUBSYSTEM** في Linux kernel، المسؤول عنه Linus Walleij، والـ mailing list بتاعته هو `linux-gpio@vger.kernel.org`.

---

### القصة: ليه أصلاً في حاجة اسمها Pin Control؟

تخيل عندك لوحة إلكترونية — SoC (System on Chip) زي اللي في موبايلك أو Raspberry Pi. الـ SoC ده فيه مئات الـ **pins** (أرجل) بتطلع منه. كل pin ممكن يشتغل بأكتر من طريقة:

- ممكن يكون **GPIO** (input/output عادي)
- ممكن يكون **UART TX** (بيبعت data)
- ممكن يكون **I2C SDA** (بيتكلم مع sensors)
- ممكن يكون **SPI MOSI** (بيتكلم مع flash memory)
- إلخ...

ده بيسموه **pin multiplexing** أو **pinmux**. الـ hardware بيديك نفس الـ pin تقدر تستخدمه بطرق مختلفة، بس لازم تقول للـ hardware "أنا عايز الـ pin ده يشتغل كـ UART مش GPIO".

مش بس كده — كل pin ممكن كمان تغير فيه الـ **electrical configuration**:
- pull-up / pull-down resistor
- drive strength (قوة الإشارة)
- open-drain / push-pull
- إلخ...

ده بيسموه **pin configuration** أو **pinconf**.

الـ **Pin Control Subsystem** هو الـ framework اللي بيدير كل ده في الـ kernel بطريقة موحدة لكل الـ hardware.

---

### الـ File: `pinctrl-state.h` — ببساطة شديدة

الملف ده صغير جداً — 4 `#define` بس. بس فكرته مهمة جداً.

تخيل الموبايل بتاعك:
- **لما بيكون شغال عادي** → الـ pins شغالة بكامل طاقتها (UART شغال، SPI شغال، إلخ)
- **لما بيكون في وضع idle** → بعض الـ pins بتتقفل لتوفير الطاقة
- **لما بينام (sleep)** → أغلب الـ pins بتتقفل تماماً عشان الطاقة توصل لأدنى مستوى
- **لما يصحى (probe)** → في حالة انتقالية قبل ما الـ driver يبدأ

الـ 4 macros دي بتعرّف **الأسماء القياسية** لهذه الحالات (states):

```c
#define PINCTRL_STATE_DEFAULT "default"  // شغال عادي
#define PINCTRL_STATE_INIT    "init"     // قبل probe الـ driver
#define PINCTRL_STATE_IDLE    "idle"     // خامل، بعض الـ clocks متوقفة
#define PINCTRL_STATE_SLEEP   "sleep"    // نايم، أقل استهلاك للطاقة
```

---

### ليه الـ Names دي مهمة؟

لأن الـ Device Tree (ملف وصف الـ hardware) بيستخدم نفس الأسماء دي. مثلاً:

```dts
/* في Device Tree بتاع أي board */
uart0 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart0_default_pins>;  /* الـ pins لما شغال */
    pinctrl-1 = <&uart0_sleep_pins>;    /* الـ pins لما نايم */
};
```

الـ kernel لما يقرأ الـ Device Tree ده، بيدور على state اسمها `"default"` وstate اسمها `"sleep"`. لو مفيش اتفاق على الأسماء دي، كل driver هيعمل أسماء مختلفة والنظام هيبقى فوضى.

الـ `pinctrl-state.h` هو **العقد المشترك** — constant strings معروفة للجميع: الـ device tree, الـ drivers, الـ power management code.

---

### الـ States التفصيلية

| State | الاسم | الوقت اللي بيُستخدم فيه | الـ PM callback |
|-------|-------|--------------------------|----------------|
| **DEFAULT** | `"default"` | تشغيل عادي، بعد الـ boot، بعد الـ resume | `.pm_runtime_resume()` / `.resume()` |
| **INIT** | `"init"` | قبل `probe()` لو الـ default سيعمل glitch | — (مؤقت) |
| **IDLE** | `"idle"` | خامل، clocks متوقفة، طاقة بعض الأجزاء ممكن شغالة | `.pm_runtime_suspend()` / `.pm_runtime_idle()` |
| **SLEEP** | `"sleep"` | نوم عميق، أقل استهلاك | `.suspend()` |

---

### القصة الكاملة: من البداية لنهاية

**السيناريو**: driver بتاع UART على SoC

1. **Boot time**: الـ kernel يقرأ الـ Device Tree، يلاقي `pinctrl-names = "default", "sleep"` في الـ UART node.
2. **قبل `probe()`**: الـ pinctrl core بيحط الـ pins في حالة `"init"` لو موجودة، أو `"default"` مباشرة.
3. **بعد `probe()`**: لو الـ pins لسه في `"init"` → بيتحولوا تلقائياً لـ `"default"`.
4. **لما الـ system يدخل sleep**: الـ PM code بيتصل بـ `pinctrl_pm_select_sleep_state()` ← بيدور على state اسمها `"sleep"` ← بيحط الـ pins في وضع استهلاك منخفض.
5. **لما يصحى**: بيتصل بـ `pinctrl_pm_select_default_state()` ← بيرجع الـ pins لـ `"default"`.

كل ده بيعتمد على الأسماء الثابتة في `pinctrl-state.h`.

---

### الـ INIT State: ليه موجودة؟

**المشكلة**: بعض الـ hardware لو حطيت الـ pins في `"default"` قبل الـ probe، الإشارات هتتغير فجأة وده ممكن يعمل **glitch** — نبضة كهربية غلط ممكن تفسد الـ device المتصل.

**الحل**: تعمل state اسمها `"init"` فيها الـ pins في وضع آمن (مثلاً input بدون pull)، الـ driver يعمل probe وهو عارف إن الـ pins في الـ init state، بعد ما الـ probe يخلص وتتأكد إن كل حاجة OK → الـ pins تتحول لـ `"default"`.

---

### الملفات المهمة اللي المفروض تعرفها

#### Core Framework
| الملف | الدور |
|-------|-------|
| `include/linux/pinctrl/pinctrl-state.h` | **الملف ده** — أسماء الـ states القياسية |
| `include/linux/pinctrl/consumer.h` | الـ API اللي الـ drivers بتستخدمها (`pinctrl_get`, `pinctrl_select_state`, إلخ) |
| `include/linux/pinctrl/pinctrl.h` | الـ API اللي الـ pin controller drivers بتسجل بيها نفسها |
| `include/linux/pinctrl/machine.h` | تعريف الـ pin mappings و hogs |
| `include/linux/pinctrl/pinmux.h` | الـ API الخاصة بالـ pin multiplexing |
| `include/linux/pinctrl/pinconf.h` | الـ API الخاصة بالـ pin configuration |
| `include/linux/pinctrl/pinconf-generic.h` | generic pin config parameters |

#### Core Implementation
| الملف | الدور |
|-------|-------|
| `drivers/pinctrl/core.c` | قلب الـ subsystem — بيدير الـ registration والـ state switching |
| `drivers/pinctrl/core.h` | الـ internal structures |
| `drivers/pinctrl/pinmux.c` | تنفيذ الـ muxing |
| `drivers/pinctrl/pinconf.c` | تنفيذ الـ pin configuration |
| `drivers/pinctrl/devicetree.c` | قراءة الـ pin states من الـ Device Tree |

#### Hardware Drivers (أمثلة)
| الملف | الـ Hardware |
|-------|-------------|
| `drivers/pinctrl/nomadik/` | ST-Ericsson Nomadik |
| `drivers/pinctrl/intel/` | Intel SoCs |
| `drivers/pinctrl/freescale/` | NXP/Freescale i.MX |
| `drivers/pinctrl/bcm/` | Broadcom |

#### Documentation
| الملف | المحتوى |
|-------|---------|
| `Documentation/driver-api/pin-control.rst` | الـ full documentation للـ subsystem |
## Phase 2: شرح الـ Pin Control (pinctrl) Framework

### المشكلة اللي بيحلها الـ subsystem ده

أي SoC حديث — خد مثلاً STM32 أو i.MX8 أو Raspberry Pi — فيه مئات الـ physical pins على الـ chip. كل pin ممكن يشتغل في أكتر من وظيفة (function) في نفس الوقت:

- نفس الـ pin ممكن يكون **UART TX** أو **SPI MOSI** أو **GPIO** — بس واحدة بس في الوقت.
- نفس الـ pin محتاج تحديد هل يشتغل بـ **pull-up** ولا **pull-down** ولا floating.
- لما الجهاز ييجي يدخل **sleep**، الـ pins لازم تتغير configuration علشان توفر طاقة.

قبل الـ pinctrl subsystem، كل driver كان بيكتب كود خاص بيه يتعامل مع الـ hardware registers اللي بتتحكم في الـ pin muxing والـ pin configuration. النتيجة:

- **Code duplication** هائل — كل driver يعيد اختراع العجلة.
- **Conflicts** — درايفرين ممكن يتنازعوا على نفس الـ pin بدون أي تنسيق.
- **Power management chaos** — مفيش طريقة موحدة تغير state الـ pins وقت الـ suspend.

---

### الحل اللي الـ kernel بياخده

الـ **pinctrl subsystem** هو layer وسطي (middleware) بين الـ hardware pin controller وبين كل driver محتاج pins. فكرته الأساسية:

1. **مركزية التحكم**: في pin controller driver واحد بس بيتكلم مع الـ hardware registers مباشرةً.
2. **State machine**: كل device عنده مجموعة "states" مسماة (`default`, `sleep`, `idle`, `init`)، وكل state بتحدد إيه configuration الـ pins في الحالة دي.
3. **Device Tree integration**: الـ configuration اتنقلت من الكود لـ DTS، فالـ driver بقى hardware-agnostic.

---

### Analogy — لوحة توصيل في استوديو تسجيل

تخيل استوديو تسجيل فيه:

| مكون الاستوديو | المقابل في الـ kernel |
|---|---|
| لوحة الـ patch panel (مئات الـ ports) | الـ SoC وكل الـ physical pins بتاعته |
| مهندس الصوت اللي بيوصل الكابلات | الـ **pin controller driver** |
| "preset" محفوظ للبرنامج التليفزيوني | **`PINCTRL_STATE_DEFAULT`** |
| "preset" وقت الفاصل (مفيش حاجة شغالة) | **`PINCTRL_STATE_IDLE`** |
| "preset" وقت ما الاستوديو مقفول بالليل | **`PINCTRL_STATE_SLEEP`** |
| "preset" وقت الإعداد قبل البث | **`PINCTRL_STATE_INIT`** |
| المخرج اللي بيقول "حول للبريك" | الـ driver بيعمل `pinctrl_pm_select_idle_state()` |
| القانون اللي بيمنع مهندسين يتنازعوا على port | الـ pinctrl core بيحمي من الـ conflicts |

المخرج (الـ driver) مش محتاج يعرف إزاي لوحة الـ patch بتتوصل — هو بس بيقول "حول للـ preset اسمه idle"، والمهندس (pin controller) هو اللي بيعرف تفاصيل الكابلات.

كل جزء في الـ analogy بيتمثل في kernel concept:
- الـ **patch panel** = `struct pinctrl_pin_desc[]` — قائمة كل الـ pins الموجودة.
- الـ **preset** = `struct pinctrl_state` — مجموعة settings محددة الاسم.
- الـ **مهندس الصوت** = `struct pinctrl_ops` + `struct pinmux_ops` + `struct pinconf_ops` — الـ vtables اللي بتنفذ الـ low-level operations.
- قانون منع التعارض = الـ pinctrl core اللي بيرفض يديك pin حد تاني بيستخدمه.

---

### Big Picture — أين يجلس الـ pinctrl في الـ kernel؟

```
  ┌─────────────────────────────────────────────────────────┐
  │                   User Space                            │
  └────────────────────────┬────────────────────────────────┘
                           │
  ┌────────────────────────▼────────────────────────────────┐
  │              Kernel Subsystems (Consumers)              │
  │                                                         │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │
  │  │  I2C     │  │  SPI     │  │  UART    │  │  GPIO  │  │
  │  │  driver  │  │  driver  │  │  driver  │  │ subsys │  │
  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───┬────┘  │
  │       │              │              │             │       │
  │       └──────────────┴──────────────┴─────────────┘      │
  │                            │                             │
  │              pinctrl_get() / pinctrl_select_state()      │
  │                            │                             │
  │  ┌─────────────────────────▼───────────────────────────┐ │
  │  │              pinctrl CORE                           │ │
  │  │                                                     │ │
  │  │  ┌──────────────┐    ┌───────────────────────────┐  │ │
  │  │  │ pinctrl_get  │    │   State Machine           │  │ │
  │  │  │ (per-device  │    │   "default" / "sleep" /   │  │ │
  │  │  │  handle)     │    │   "idle"   / "init"       │  │ │
  │  │  └──────────────┘    └───────────────────────────┘  │ │
  │  │                                                     │ │
  │  │  ┌──────────────────────────────────────────────┐   │ │
  │  │  │  Mapping Table  (from Device Tree / board)   │   │ │
  │  │  │  device X, state "default" → group A → func B│   │ │
  │  │  └──────────────────────────────────────────────┘   │ │
  │  └─────────────────────────┬───────────────────────────┘ │
  │                            │                             │
  │         pinctrl_ops / pinmux_ops / pinconf_ops           │
  │                            │                             │
  │  ┌─────────────────────────▼───────────────────────────┐ │
  │  │         Pin Controller Driver  (Provider)           │ │
  │  │   e.g. pinctrl-stm32.c / pinctrl-bcm2835.c         │ │
  │  │                                                     │ │
  │  │   struct pinctrl_desc  ──►  pinctrl_register()      │ │
  │  └─────────────────────────┬───────────────────────────┘ │
  └────────────────────────────┼────────────────────────────┘
                               │  MMIO registers
  ┌────────────────────────────▼────────────────────────────┐
  │              SoC Hardware — Pin Controller IP            │
  │    (MUX registers, CONF registers, IOCFG registers)      │
  └─────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الفكرة المحورية في الـ pinctrl هي: **تسمية الـ states بدل hardcode الـ configuration**.

الـ `pinctrl-state.h` هو أبسط ملف في الـ subsystem كله — أربع `#define` بس:

```c
#define PINCTRL_STATE_DEFAULT "default"   /* جاهز للشغل الطبيعي */
#define PINCTRL_STATE_INIT    "init"      /* قبل الـ probe — تجنب glitches */
#define PINCTRL_STATE_IDLE    "idle"      /* runtime suspend — توفير جزئي للطاقة */
#define PINCTRL_STATE_SLEEP   "sleep"     /* full suspend — أقل استهلاك ممكن */
```

الأهمية مش في الـ strings نفسها — الأهمية إن دول **contracts** متفق عليها بين:
- الـ **Device Tree** اللي بيعرّف الـ pin configuration لكل state.
- الـ **driver core** اللي بيطبق الـ state تلقائياً في الوقت المناسب.
- الـ **device drivers** اللي ممكن تطلب state معين يدوياً.

---

### كيف الـ states بتتربط ببعض — رحلة الـ device

```
  Power On
     │
     ▼
  ┌──────────────────────────────────────────┐
  │  pinctrl_init_done() من الـ driver core  │
  │  لو في "init" state → يطبقها            │
  │  لو مفيش → يطبق "default" مباشرةً       │
  └───────────────────┬──────────────────────┘
                      │
                      ▼
              ┌───────────────┐
              │  driver probe()│
              └───────┬───────┘
                      │  لو الـ pins لسه في "init"
                      │  → الـ core يحولها لـ "default"
                      ▼
              ┌───────────────┐
              │   "default"   │  ◄─── الشغل الطبيعي
              └───────┬───────┘
                      │
            ┌─────────┴────────────┐
            │                      │
            ▼                      ▼
     ┌─────────────┐        ┌─────────────┐
     │   "idle"    │        │   "sleep"   │
     │ pm_runtime  │        │  .suspend() │
     │  _suspend() │        └─────────────┘
     └─────────────┘
```

---

### الـ Structs الأساسية وعلاقتها ببعض

```
  struct device
      └── struct dev_pin_info  *pins          (في devinfo.h)
              │
              ├── struct pinctrl       *p             ← الـ handle الخاص بالـ device
              ├── struct pinctrl_state *default_state ← pointer للـ "default"
              ├── struct pinctrl_state *init_state    ← pointer للـ "init"
              ├── struct pinctrl_state *sleep_state   ← pointer للـ "sleep" (CONFIG_PM)
              └── struct pinctrl_state *idle_state    ← pointer للـ "idle"  (CONFIG_PM)


  struct pinctrl  (opaque للـ consumers)
      ├── struct device *dev
      ├── struct list_head states      ← list of pinctrl_state
      └── struct pinctrl_state *state  ← الـ current active state


  struct pinctrl_state
      ├── const char *name             ← "default" / "sleep" / ...
      └── struct list_head settings    ← list of pinctrl_setting
                                           (كل setting = pin group + function/config)


  struct pinctrl_desc                  (الـ provider بيسجل ده)
      ├── const char              *name
      ├── pinctrl_pin_desc        *pins     ← array of all pins
      ├── struct pinctrl_ops      *pctlops  ← grouping
      ├── struct pinmux_ops       *pmxops   ← mux control
      └── struct pinconf_ops      *confops  ← electrical config
```

---

### الـ State Lifecycle في الكود الحقيقي

#### Consumer (driver عادي):
```c
/* الطريقة المختصرة — devm تضمن cleanup تلقائي */
struct pinctrl *pctl = devm_pinctrl_get_select_default(&pdev->dev);
if (IS_ERR(pctl))
    return PTR_ERR(pctl);

/* لو محتاج تتحكم يدوياً في كل state */
struct pinctrl *p = devm_pinctrl_get(&pdev->dev);
struct pinctrl_state *sleep_st = pinctrl_lookup_state(p, PINCTRL_STATE_SLEEP);

/* وقت الـ suspend */
pinctrl_select_state(p, sleep_st);

/* وقت الـ resume */
pinctrl_pm_select_default_state(&pdev->dev);
```

#### Provider (pin controller driver):
```c
static const struct pinctrl_pin_desc myboard_pins[] = {
    PINCTRL_PIN(0, "PA0"),
    PINCTRL_PIN(1, "PA1"),
    /* ... */
};

static const struct pinctrl_desc myboard_desc = {
    .name    = "myboard-pinctrl",
    .pins    = myboard_pins,
    .npins   = ARRAY_SIZE(myboard_pins),
    .pctlops = &myboard_pctlops,
    .pmxops  = &myboard_pmxops,
    .confops = &myboard_confops,
};

/* في الـ probe */
pinctrl_register_and_init(&myboard_desc, dev, drvdata, &pctldev);
pinctrl_enable(pctldev);
```

---

### الـ init State — ليه موجود أصلاً؟

ده concept ذكي جداً. الـ driver core بيطبق الـ `default` state تلقائياً **قبل** ما يستدعي `probe()`. ده كويس في الغالب، بس في بعض الأجهزة:

- تطبيق الـ `default` state بيسبب **glitch** على الـ pin (voltage spike قصير).
- لو الجهاز (مثلاً display panel) حساس، الـ glitch ده ممكن يعطله.

الحل: Driver بيعرّف `"init"` state في الـ DTS. الـ core بيطبق الـ `init` بدل `default` قبل الـ probe. بعد الـ probe لو الـ state لسه `init`، الـ core بيحوله لـ `default` — بس بعد ما الـ driver عمل initialize صحيح للـ hardware.

---

### ما الذي يمتلكه الـ pinctrl Core مقابل ما يفوّضه للـ Drivers؟

| المسؤولية | الـ pinctrl Core | الـ Pin Controller Driver |
|---|---|---|
| تسجيل الـ pins والـ groups | يستقبل ويخزن | يوفر عبر `pinctrl_desc` |
| تحليل الـ Device Tree | `dt_node_to_map()` callback | الـ driver ينفذ الـ callback |
| إدارة الـ states والـ transitions | نعم — بالكامل | لا |
| منع التعارض على الـ pins | نعم — بالكامل | لا |
| كتابة الـ MUX registers | لا | نعم — عبر `pinmux_ops` |
| ضبط الـ electrical config | لا | نعم — عبر `pinconf_ops` |
| ربط الـ state بالـ PM lifecycle | نعم — تلقائياً عبر driver core | لا |
| debugfs interface | نعم — `/sys/kernel/debug/pinctrl/` | يوفر `pin_dbg_show` اختياري |

---

### علاقة الـ pinctrl بالـ GPIO Subsystem

الـ **GPIO subsystem** محتاج يعرف: هل الـ pin ده حر ولا الـ pinctrl شايله؟

الـ pinctrl core بيوفر:
```c
bool pinctrl_gpio_can_use_line(struct gpio_chip *gc, unsigned int offset);
int  pinctrl_gpio_request(struct gpio_chip *gc, unsigned int offset);
void pinctrl_gpio_free(struct gpio_chip *gc, unsigned int offset);
int  pinctrl_gpio_direction_input(struct gpio_chip *gc, unsigned int offset);
int  pinctrl_gpio_direction_output(struct gpio_chip *gc, unsigned int offset);
```

الـ GPIO subsystem بيستدعي الـ functions دي قبل ما يتعامل مع الـ pin، علشان الـ pinctrl core يأكد إن مفيش تعارض ويعمل الـ mux switch المطلوب.

---

### الـ subsystem ده بيعتمد على إيه؟

- **Device Tree (OF)**: علشان يقرأ الـ pin configurations — لازم تعرف الـ OF/DT infrastructure.
- **Device Model (driver core)**: الـ `struct device` هو اللي بيحمل الـ `dev_pin_info` — الـ pinctrl متكامل جوه الـ driver model نفسه.
- **Power Management (PM)**: الـ `pinctrl_pm_select_*_state()` functions بتتكامل مع الـ PM callbacks — لازم تعرف الـ runtime PM lifecycle.
- **GPIO subsystem**: علاقة bidirectional — كل منهم محتاج يعرف حالة الـ pins عند التاني.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### ملخص الملف

الـ `pinctrl-state.h` ملف بسيط جداً — بيعرّف بس **4 string constants** بتمثل الـ standard pin control states اللي Linux بيعرفها. مفيش structs، مفيش enums، مفيش config options — بس macros. الأهمية مش في التعقيد، الأهمية في إن ده **اللغة المشتركة** بين كل الـ drivers والـ device tree والـ pinctrl core.

---

### 0. الـ State Constants — Cheatsheet

| Macro | القيمة (String) | الوقت اللي بيُستخدم فيه | مثال |
|---|---|---|---|
| `PINCTRL_STATE_DEFAULT` | `"default"` | الحالة الطبيعية — الـ device شغال | بعد `probe()` أو بعد `resume()` |
| `PINCTRL_STATE_INIT` | `"init"` | قبل `probe()` — لو الـ default هيعمل glitch | UART بيتكلم مع bootloader |
| `PINCTRL_STATE_IDLE` | `"idle"` | الـ system مش نايم بس مش شغال بكامل طاقته | بعد `pm_runtime_suspend()` |
| `PINCTRL_STATE_SLEEP` | `"sleep"` | أعمق حالة نوم | من `suspend()` |

#### متى يُستخدم كل state؟

```
BOOT
 │
 ├─► "init"     ← لو موجود في DT → بيتطبق قبل probe()
 │
 ├─► probe()
 │
 ├─► "default"  ← بيتطبق تلقائي بعد probe() (أو لو "init" مش موجود)
 │
 │   [Device running normally]
 │
 ├─► pm_runtime_suspend() / pm_runtime_idle()
 │       └─► "idle"
 │
 ├─► pm_runtime_resume() / resume()
 │       └─► "default"
 │
 └─► suspend()
         └─► "sleep"
```

---

### 1. الـ Structs المرتبطة (من consumer.h)

الملف نفسه مفيش فيه structs، لكن الـ `consumer.h` اللي بيعمل `#include` ليه بيعرّض الـ opaque types دي:

#### `struct pinctrl`

- **الغرض**: handle بيمثل إن device معين اتطلب pinctrl resources. private للـ core — الـ driver بيشوفه كـ cookie بس.
- **الـ driver بياخده من**: `pinctrl_get(dev)` أو `devm_pinctrl_get(dev)`
- **بيتحط فيه**: `pinctrl_state` pointers كتير (واحد لكل state اسمها في DT)

#### `struct pinctrl_state`

- **الغرض**: بيمثل state واحدة (زي `"default"` أو `"sleep"`) — بتحتوي على مجموعة pin settings.
- **الـ driver بياخده من**: `pinctrl_lookup_state(p, PINCTRL_STATE_DEFAULT)`
- **بيتفعّل بـ**: `pinctrl_select_state(p, s)`

---

### 2. رسم العلاقات بين الـ Structs

```
struct device
    │
    │  pinctrl_get(dev)
    ▼
struct pinctrl  ◄──────────────────────────────────┐
    │                                               │
    │  pinctrl_lookup_state(p, "default")           │
    │  pinctrl_lookup_state(p, "sleep")             │
    │  pinctrl_lookup_state(p, "idle")              │
    │  pinctrl_lookup_state(p, "init")              │
    ▼                                               │
struct pinctrl_state ["default"] ─────────────────►│
struct pinctrl_state ["sleep"]   ─────────────────►│
struct pinctrl_state ["idle"]    ─────────────────►│
struct pinctrl_state ["init"]    ─────────────────►│
                                                    │
    pinctrl_put(p) ─────────────────────────────────┘
    (يحرر كل الـ states)
```

الـ `PINCTRL_STATE_*` macros بتُستخدم كـ keys للـ lookup — يعني:

```c
s = pinctrl_lookup_state(p, PINCTRL_STATE_SLEEP);
//                            ^^^^^^^^^^^^^^^^^^^^^^
//                            ده بس string "sleep"
//                            بيتطابق مع اسم الـ state في DT
```

---

### 3. دورة الحياة الكاملة — Lifecycle Diagram

```
[Device Tree / ACPI]
    pinctrl-names = "default", "init", "sleep", "idle";
    pinctrl-0 = <&uart_pins_default>;
    pinctrl-1 = <&uart_pins_init>;
    pinctrl-2 = <&uart_pins_sleep>;
    pinctrl-3 = <&uart_pins_idle>;
         │
         │  (kernel parses DT at boot)
         ▼
[pinctrl core] يبني internal state objects
         │
         │  devm_pinctrl_get(dev)   ← في probe()
         ▼
    struct pinctrl *p  (handle للـ device)
         │
         │  pinctrl_lookup_state(p, "default")
         ▼
    struct pinctrl_state *s_default
         │
         │  pinctrl_select_state(p, s_default)
         ▼
    [Pins configured: mux + config applied to hardware]
         │
    ┌────┴─────────────────────────────────┐
    │                                      │
    │  pm_runtime_suspend()                │
    │      pinctrl_pm_select_idle_state()  │
    │           └─► "idle" state           │
    │                                      │
    │  suspend()                           │
    │      pinctrl_pm_select_sleep_state() │
    │           └─► "sleep" state          │
    │                                      │
    │  resume() / pm_runtime_resume()      │
    │      pinctrl_pm_select_default_state()│
    │           └─► "default" state        │
    └──────────────────────────────────────┘
         │
         │  device removed / driver unloaded
         │  devm_pinctrl_put(p)   ← تلقائي مع devm
         ▼
    [Resources freed, pins released]
```

---

### 4. Call Flow Diagrams

#### الـ init → default transition (الأهم)

```
kernel boot
    │
    ├─► pinctrl core يشوف "init" state موجود في DT؟
    │       ├─► نعم → يطبق "init" قبل ما يكمل
    │       └─► لأ  → يطبق "default" مباشرة
    │
    ├─► driver->probe() بيتنادى
    │
    └─► بعد probe() بيرجع:
            ├─► لو الـ state لسه "init" → core يحوّله تلقائياً لـ "default"
            └─► لو الـ driver غيّره في probe() → يفضل زي ما هو
```

#### الـ PM state transitions

```
pm_runtime_suspend(dev)
    │
    └─► driver's .runtime_suspend()
            │
            └─► pinctrl_pm_select_idle_state(dev)
                    │
                    └─► pinctrl_lookup_state(p, "idle")   [= PINCTRL_STATE_IDLE]
                            │
                            └─► pinctrl_select_state(p, s)
                                    │
                                    └─► pinctrl core applies mux/config to hw regs
```

```
suspend(dev)
    │
    └─► driver's .suspend()
            │
            └─► pinctrl_pm_select_sleep_state(dev)
                    │
                    └─► pinctrl_lookup_state(p, "sleep")  [= PINCTRL_STATE_SLEEP]
                            │
                            └─► pinctrl_select_state(p, s)
                                    │
                                    └─► pins set to low-power configuration
```

#### `pinctrl_get_select()` helper — الأكثر استخداماً في probe()

```c
devm_pinctrl_get_select(dev, PINCTRL_STATE_DEFAULT)
    │
    ├─► devm_pinctrl_get(dev)
    │       └─► يبحث عن pinctrl data في device's DT node
    │           يخزن handle في devres list
    │
    ├─► pinctrl_lookup_state(p, "default")
    │       └─► يبحث في قائمة الـ states اللي اتبنت من DT
    │
    └─► pinctrl_select_state(p, s)
            └─► يطبق كل الـ pin settings على الـ hardware
```

---

### 5. الـ Locking Strategy

الـ `pinctrl-state.h` نفسه مفيش فيه locks — هو بس string definitions. لكن الـ subsystem اللي بيستخدمه بيعتمد على:

| Lock | بيحمي إيه | مين بيمسكه |
|---|---|---|
| `pinctrl->mutex` (في core) | الـ state transitions، منع race بين probe وـ PM | `pinctrl_select_state()` |
| `pinctrl_maps_mutex` | قائمة الـ pin map entries | عند register/unregister |
| الـ `device` lock (الـ PM framework) | إن الـ PM callbacks مش بتتنادى بالتوازي | PM core نفسه |

#### مهم جداً — لا تعمل state transition من ISR

الـ `pinctrl_select_state()` ممكن تاخد sleep (بيمسك mutex)، يعني:
- ✓ مسموح من process context (probe, suspend, resume)
- ✗ ممنوع من interrupt context أو atomic context

---

### ملاحظة عملية: الـ "init" state — ليه موجود؟

بعض الـ drivers زي UART أو SPI لما الـ pins بتتحول لـ "default" بتعمل **glitch** — نبضة خاطئة — لأن الـ mux بيتغير وهو الـ bus شغال. الـ "init" state بيخلي الـ pins في وضع آمن (مثلاً: Hi-Z أو pulled) لحد ما الـ driver يجهز الـ hardware controller، وبعدين الـ driver بنفسه يطبق "default" في الوقت الصح.

```c
/* مثال: driver محتاج يتحكم في timing */
static int my_uart_probe(struct platform_device *pdev)
{
    struct pinctrl *pinctrl;
    struct pinctrl_state *state;

    /* get handle - pins still in "init" state */
    pinctrl = devm_pinctrl_get(&pdev->dev);

    /* configure hardware first, THEN switch pins */
    my_uart_hw_init(pdev);

    /* now safe to switch to default */
    state = pinctrl_lookup_state(pinctrl, PINCTRL_STATE_DEFAULT);
    pinctrl_select_state(pinctrl, state);

    return 0;
}
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

الـ `pinctrl-state.h` مش فيه functions — هو بيعرّف بس **4 macros** بتمثل أسماء الـ standard states اللي بيستخدمها الـ pinctrl subsystem. الـ consumer API في `consumer.h` هو اللي بيستخدم الأسماء دي.

#### جدول الـ State Macros

| Macro | Value | متى بيُستخدم |
|---|---|---|
| `PINCTRL_STATE_DEFAULT` | `"default"` | عند الـ resume، بعد الـ probe، الحالة الطبيعية للـ device |
| `PINCTRL_STATE_INIT` | `"init"` | قبل الـ probe لو الـ default هيعمل glitch على الـ pins |
| `PINCTRL_STATE_IDLE` | `"idle"` | عند الـ `pm_runtime_suspend()` أو `pm_runtime_idle()` |
| `PINCTRL_STATE_SLEEP` | `"sleep"` | عند الـ `suspend()` — أعمق حالة نوم |

#### جدول الـ Consumer API Functions (من `consumer.h`)

| Function | النوع | الغرض |
|---|---|---|
| `pinctrl_get()` | acquire | بياخد handle على الـ pinctrl للـ device |
| `pinctrl_put()` | release | بيحرر الـ handle |
| `devm_pinctrl_get()` | acquire (managed) | زي `pinctrl_get` بس بيتحرر أوتوماتيك مع الـ device |
| `devm_pinctrl_put()` | release (managed) | بيحرر الـ managed handle قبل الـ device removal |
| `pinctrl_lookup_state()` | lookup | بيجيب pointer للـ state باسمها |
| `pinctrl_select_state()` | activate | بيفعّل state معينة على الـ pinctrl |
| `pinctrl_select_default_state()` | helper | بيفعّل الـ `"default"` state مباشرة |
| `pinctrl_get_select()` | combo | get + lookup + select في call واحدة |
| `pinctrl_get_select_default()` | combo | `pinctrl_get_select` مع `"default"` |
| `devm_pinctrl_get_select()` | combo (managed) | devm version من `pinctrl_get_select` |
| `devm_pinctrl_get_select_default()` | combo (managed) | devm version مع `"default"` |
| `pinctrl_pm_select_default_state()` | PM helper | بيفعّل `"default"` من سياق الـ PM |
| `pinctrl_pm_select_init_state()` | PM helper | بيفعّل `"init"` من سياق الـ PM |
| `pinctrl_pm_select_sleep_state()` | PM helper | بيفعّل `"sleep"` من سياق الـ PM |
| `pinctrl_pm_select_idle_state()` | PM helper | بيفعّل `"idle"` من سياق الـ PM |
| `pinctrl_gpio_can_use_line()` | GPIO bridge | بيتحقق لو الـ pin مش محجوز من الـ pinctrl |
| `pinctrl_gpio_request()` | GPIO bridge | بيطلب pin من الـ pinctrl للـ GPIO use |
| `pinctrl_gpio_free()` | GPIO bridge | بيحرر الـ pin من الـ GPIO use |
| `pinctrl_gpio_direction_input()` | GPIO bridge | بيبعت direction change للـ pinctrl |
| `pinctrl_gpio_direction_output()` | GPIO bridge | بيبعت direction change للـ pinctrl |
| `pinctrl_gpio_set_config()` | GPIO bridge | بيبعت pin config (pull, drive, etc.) للـ pinctrl |

---

### Group 1: State Name Macros

الـ macros دي هي **string constants** بتمثل الأسماء المعيارية للـ pin states. مش كود قابل للتنفيذ — هي contracts بين الـ driver والـ Device Tree أو ACPI. الـ DT property `pinctrl-names` بتحتوي نفس الأسماء دي، والـ kernel بيعمل lookup بيها.

#### `PINCTRL_STATE_DEFAULT`

```c
#define PINCTRL_STATE_DEFAULT "default"
```

**الـ `"default"` state** هي الـ state الإجبارية لأي device بيستخدم pinctrl. الـ kernel بيفعّلها أوتوماتيك قبل ما `probe()` تتنفذ (إلا لو في `"init"` state معرّفة). بيرجعلها كمان في `pm_runtime_resume()` و `resume()`.

- **المعيار**: أي driver عنده pinctrl لازم يكون عنده `"default"` state على الأقل.
- **الـ DT**: `pinctrl-names = "default";` مع `pinctrl-0 = <&my_pins_default>;`

---

#### `PINCTRL_STATE_INIT`

```c
#define PINCTRL_STATE_INIT "init"
```

الـ **`"init"` state** بتُستخدم لما الـ `"default"` state هتعمل **glitch** على الـ pins قبل ما الـ driver يتهيأ. مثال: SPI bus لو اتنقل للـ default قبل الـ probe، ممكن يبعت data خاطئة على الـ slave.

- الـ kernel بيفعّلها قبل `probe()` لو موجودة.
- لو الـ pins فضلت في `"init"` بعد ما `probe()` خلصت، الـ kernel بيحوّلهم أوتوماتيك لـ `"default"`.
- **Use case**: UART TX pin — تفضل high-Z في الـ init عشان ما يبعتش garbage.

---

#### `PINCTRL_STATE_IDLE`

```c
#define PINCTRL_STATE_IDLE "idle"
```

الـ **`"idle"` state** بتمثل حالة وسطى — السيستم مش شغال بكامل طاقته بس مش نايم. الـ clocks ممكن تكون متقفّلة لكن الـ power موجودة. بيُستخدم من `pm_runtime_suspend()` أو `pm_runtime_idle()`.

- مثال: I2C bus في `"idle"` ممكن يحوّل الـ SDA و SCL لـ GPIO input عشان يوفر power.
- اختياري — مش كل الـ devices بتحتاجه.

---

#### `PINCTRL_STATE_SLEEP`

```c
#define PINCTRL_STATE_SLEEP "sleep"
```

الـ **`"sleep"` state** هي أعمق حالة نوم. الـ device مش شغال خالص والـ pins لازم تكون في حالة آمنة (مثلاً pulled down أو high-Z) عشان ما يحصلش leakage current. بيُستخدم من `suspend()`.

- مثال: SDIO pins في الـ sleep ممكن يتحولوا لـ GPIO output low.
- لازم يكون مقابله `"default"` في الـ resume path.

---

### Group 2: Core Acquire/Release Functions

#### `pinctrl_get()`

```c
struct pinctrl * __must_check pinctrl_get(struct device *dev);
```

بيعمل **acquire** على الـ pinctrl handle للـ device. بيمشي على الـ pin mappings المسجّلة في الـ subsystem ويبني `struct pinctrl` فيها كل الـ states المتاحة للـ device ده. الـ handle ده هو entry point لكل عمليات الـ pinctrl بعد كده.

- **`dev`**: الـ device اللي عايزين نتحكم في pins بتاعته.
- **Return**: pointer لـ `struct pinctrl` أو `ERR_PTR` لو فشل (مثلاً `-EPROBE_DEFER` لو الـ pinctrl driver لسه مش موجود).
- **لازم تتحقق** من الـ return بـ `IS_ERR()`.
- **Locking**: الـ function بتاخد internal mutex داخل الـ pinctrl core.
- **Who calls it**: driver's `probe()` — أو `devm_pinctrl_get()` كـ wrapper.

---

#### `pinctrl_put()`

```c
void pinctrl_put(struct pinctrl *p);
```

بيحرر الـ `struct pinctrl` ويرجّع كل الـ resources اللي اتحجزت. بيعمل deactivate للـ state الحالية وبيرجع الـ pins لحالتها الـ default في الـ pinctrl core.

- **`p`**: الـ handle اللي رجع من `pinctrl_get()`.
- **Return**: void.
- **Who calls it**: driver's `remove()` أو error path في `probe()`.

---

#### `devm_pinctrl_get()`

```c
struct pinctrl * __must_check devm_pinctrl_get(struct device *dev);
```

نفس `pinctrl_get()` بالظبط لكن بيسجّل cleanup action مع الـ **devres** framework. لما الـ device بيتشال أو الـ driver بيفشل، الـ kernel بيعمل `pinctrl_put()` أوتوماتيك.

- **الفرق الوحيد**: الـ cleanup أوتوماتيك — مش محتاج تكتب `pinctrl_put()` في الـ remove.
- **الاستخدام المفضّل** في كل الـ drivers الحديثة.

---

#### `devm_pinctrl_put()`

```c
void devm_pinctrl_put(struct pinctrl *p);
```

بيحرر الـ managed handle قبل الـ device removal. نادراً بيُستخدم إلا لو الـ driver عايز يعمل cleanup مبكر (مثلاً في error path بعد `devm_pinctrl_get()`).

---

### Group 3: State Lookup & Selection

#### `pinctrl_lookup_state()`

```c
struct pinctrl_state * __must_check pinctrl_lookup_state(struct pinctrl *p,
                                                         const char *name);
```

بيدوّر على الـ state بالاسم داخل الـ `struct pinctrl` ويرجع pointer ليها. الأسماء هي نفسها الـ macros المعرّفة في `pinctrl-state.h` (أو أي اسم custom).

- **`p`**: الـ pinctrl handle.
- **`name`**: اسم الـ state — مثلاً `PINCTRL_STATE_DEFAULT` أو `"my-custom-state"`.
- **Return**: pointer لـ `struct pinctrl_state` أو `ERR_PTR(-ENODEV)` لو الاسم مش موجود.
- **Key detail**: الـ `struct pinctrl_state` هو opaque cookie — ما تحاولش تقرأ حاجة منه مباشرة.

---

#### `pinctrl_select_state()`

```c
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *s);
```

الـ **core activation function** — بيفعّل الـ state المحددة على الـ hardware. بيعمل:
1. deactivate للـ state الحالية.
2. تطبيق كل الـ pin muxing والـ pin configurations اللي في الـ state الجديدة.
3. تحديث الـ `p->state` لتشاور على الـ new state.

- **`p`**: الـ pinctrl handle.
- **`s`**: الـ state pointer اللي رجع من `pinctrl_lookup_state()`.
- **Return**: `0` لو نجح، أو negative error code.
- **Locking**: بياخد `pinctrl_maps_mutex` وممكن يبعت calls للـ pinctrl driver اللي ممكن تاخد locks تانية.
- **Who calls it**: الـ driver مباشرة أو من خلال الـ PM helpers.

---

#### `pinctrl_select_default_state()`

```c
int pinctrl_select_default_state(struct device *dev);
```

shortcut بيعمل lookup وselect للـ `"default"` state. بيستخدمه الـ kernel core بعد `probe()` لو الـ driver استخدم `"init"` state وفضلت الـ pins فيها.

- **`dev`**: الـ device مباشرة بدون الحاجة للـ handle.
- **Return**: `0` لو نجح أو ما لقاش state (في الحالة دي بيرجع 0 عادةً).

---

### Group 4: Combo Helper Functions (inline)

الـ functions دي `static inline` في الـ header — هدفها تقليل الـ boilerplate في الـ drivers.

#### `pinctrl_get_select()`

```c
static inline struct pinctrl * __must_check
pinctrl_get_select(struct device *dev, const char *name);
```

بيعمل الـ 3 خطوات الأساسية في call واحدة: `pinctrl_get()` + `pinctrl_lookup_state()` + `pinctrl_select_state()`. لو أي خطوة فشلت بيعمل cleanup للخطوات اللي قبلها ويرجع `ERR_PTR`.

```c
/* Pseudocode flow */
p = pinctrl_get(dev);
if (IS_ERR(p)) return p;

s = pinctrl_lookup_state(p, name);
if (IS_ERR(s)) {
    pinctrl_put(p);          // cleanup
    return ERR_CAST(s);
}

ret = pinctrl_select_state(p, s);
if (ret < 0) {
    pinctrl_put(p);          // cleanup
    return ERR_PTR(ret);
}

return p;  // caller مسؤول عن pinctrl_put
```

- **Return**: الـ `struct pinctrl *` handle لو نجح (الـ caller مسؤول عن `pinctrl_put()`).
- **من يستخدمه**: drivers بسيطة في `probe()` — غير الـ devm version.

---

#### `pinctrl_get_select_default()`

```c
static inline struct pinctrl * __must_check
pinctrl_get_select_default(struct device *dev);
```

نفس `pinctrl_get_select()` بس hardcoded على `PINCTRL_STATE_DEFAULT`.

```c
return pinctrl_get_select(dev, PINCTRL_STATE_DEFAULT);
```

---

#### `devm_pinctrl_get_select()`

```c
static inline struct pinctrl * __must_check
devm_pinctrl_get_select(struct device *dev, const char *name);
```

نفس `pinctrl_get_select()` لكن بيستخدم `devm_pinctrl_get()` بدل `pinctrl_get()`. الـ cleanup أوتوماتيك مع الـ device lifecycle. الـ preferred pattern في الـ modern drivers.

```c
/* Pseudocode flow */
p = devm_pinctrl_get(dev);   // managed
if (IS_ERR(p)) return p;

s = pinctrl_lookup_state(p, name);
if (IS_ERR(s)) {
    devm_pinctrl_put(p);     // explicit early cleanup
    return ERR_CAST(s);
}

ret = pinctrl_select_state(p, s);
if (ret < 0) {
    devm_pinctrl_put(p);
    return ERR_PTR(ret);
}

return p;  // cleanup مع device removal أوتوماتيك
```

---

#### `devm_pinctrl_get_select_default()`

```c
static inline struct pinctrl * __must_check
devm_pinctrl_get_select_default(struct device *dev);
```

الأبسط والأكثر استخداماً في الـ modern drivers — devm + default state في سطر واحد.

```c
return devm_pinctrl_get_select(dev, PINCTRL_STATE_DEFAULT);
```

---

### Group 5: PM (Power Management) Helpers

الـ functions دي موجودة تحت `CONFIG_PM` — بتربط بين الـ standard PM callbacks والـ pinctrl states. لو `CONFIG_PM` مش مفعّل، بيتحولوا لـ stubs بترجع `0`.

#### `pinctrl_pm_select_default_state()`

```c
int pinctrl_pm_select_default_state(struct device *dev);
```

بيفعّل `PINCTRL_STATE_DEFAULT` من سياق الـ PM. بيُستخدم في `resume()` و `pm_runtime_resume()`.

- **`dev`**: الـ device.
- **Return**: `0` لو نجح، أو error code.
- **Context**: sleep context مسموح بيه — بيتنادى من الـ PM callbacks.

---

#### `pinctrl_pm_select_init_state()`

```c
int pinctrl_pm_select_init_state(struct device *dev);
```

بيفعّل `PINCTRL_STATE_INIT`. نادر الاستخدام في الـ PM context — معظم الـ drivers بتستخدمه فقط في `probe()`.

---

#### `pinctrl_pm_select_sleep_state()`

```c
int pinctrl_pm_select_sleep_state(struct device *dev);
```

بيفعّل `PINCTRL_STATE_SLEEP` من `suspend()`. بيحوّل الـ pins لأقل استهلاك طاقة ممكن.

- **Context**: يُنادى من `suspend()` أو `pm_runtime_suspend()`.
- **مهم**: لو فشل لازم الـ driver يعالج الـ error — فشله ممكن يمنع الـ system من الدخول لـ sleep.

---

#### `pinctrl_pm_select_idle_state()`

```c
int pinctrl_pm_select_idle_state(struct device *dev);
```

بيفعّل `PINCTRL_STATE_IDLE` من `pm_runtime_idle()` أو `pm_runtime_suspend()`. الـ pins في حالة وسطى — ممكن يوفر power بدون أعمق sleep.

---

### Group 6: GPIO Bridge Functions

الـ functions دي واجهة بين الـ **GPIO subsystem** والـ **pinctrl subsystem**. الـ gpio core بتناديها تلقائياً — الـ device drivers النهائية مش بتناديها مباشرة.

#### `pinctrl_gpio_can_use_line()`

```c
bool pinctrl_gpio_can_use_line(struct gpio_chip *gc, unsigned int offset);
```

بيتحقق إن الـ pin مش محجوز بالفعل من الـ pinctrl لـ function تانية (مثلاً UART أو SPI). الـ gpio core بتناديه قبل ما تسمح للـ userspace أو الـ driver بحجز الـ GPIO.

- **`gc`**: الـ GPIO chip.
- **`offset`**: رقم الـ GPIO line داخل الـ chip.
- **Return**: `true` لو الـ line متاحة للـ GPIO use.

---

#### `pinctrl_gpio_request()`

```c
int pinctrl_gpio_request(struct gpio_chip *gc, unsigned int offset);
```

بيبلّغ الـ pinctrl subsystem إن الـ GPIO subsystem بياخد ownership على الـ pin. ممكن يعمل **mux** للـ pin لـ GPIO function لو الـ pinctrl driver supportها.

- **Return**: `0` أو negative error.
- **Who calls it**: `gpiochip_request_own_desc()` أو `gpio_request()` — مش الـ application driver.

---

#### `pinctrl_gpio_free()`

```c
void pinctrl_gpio_free(struct gpio_chip *gc, unsigned int offset);
```

عكس `pinctrl_gpio_request()` — بيرجع الـ pin للـ pinctrl وبيشيل الـ GPIO ownership.

---

#### `pinctrl_gpio_direction_input()` / `pinctrl_gpio_direction_output()`

```c
int pinctrl_gpio_direction_input(struct gpio_chip *gc, unsigned int offset);
int pinctrl_gpio_direction_output(struct gpio_chip *gc, unsigned int offset);
```

بيبعتوا **direction change notification** للـ pinctrl driver. بعض الـ hardware بيحتاج تغيير الـ pin configuration لما الـ direction بيتغير (مثلاً تغيير الـ drive strength أو الـ pull).

- **Who calls them**: `gpio_direction_input()` / `gpio_direction_output()` في الـ GPIO core.

---

#### `pinctrl_gpio_set_config()`

```c
int pinctrl_gpio_set_config(struct gpio_chip *gc, unsigned int offset,
                            unsigned long config);
```

بيمرّر **pin configuration** (encoded في `unsigned long` بـ `PIN_CONF_PACKED` format) من الـ GPIO subsystem للـ pinctrl driver. بيُستخدم لضبط الـ pull-up/pull-down، الـ drive strength، إلخ.

- **`config`**: القيمة المشفّرة بـ `pinconf_to_config_packed(param, arg)`.

---

### نمط الاستخدام الكامل في Driver

```c
/* probe() — الطريقة الحديثة */
static int my_driver_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct pinctrl *pctl;

    /* get + select "default" في سطر واحد مع devm */
    pctl = devm_pinctrl_get_select_default(dev);
    if (IS_ERR(pctl)) {
        /* -EPROBE_DEFER لو pinctrl driver لسه مش جاهز */
        return PTR_ERR(pctl);
    }

    /* باقي الـ probe ... */
    return 0;
}

/* suspend() */
static int my_driver_suspend(struct device *dev)
{
    /* حوّل الـ pins لأقل power */
    return pinctrl_pm_select_sleep_state(dev);
}

/* resume() */
static int my_driver_resume(struct device *dev)
{
    /* رجّع الـ pins للعمل الطبيعي */
    return pinctrl_pm_select_default_state(dev);
}
```

---

### علاقة الـ State Macros بالـ Device Tree

```
/* DTS */
&uart0 {
    pinctrl-names = "default", "sleep";   /* ترتيب الأسماء مهم */
    pinctrl-0 = <&uart0_default_pins>;    /* index 0 = "default" */
    pinctrl-1 = <&uart0_sleep_pins>;      /* index 1 = "sleep" */
};

/* الـ kernel بيعمل: */
/* pinctrl_lookup_state(p, "default") → pinctrl-0 */
/* pinctrl_lookup_state(p, "sleep")   → pinctrl-1 */
```

الأسماء في `pinctrl-names` لازم تتطابق بالظبط مع الـ string values للـ macros. الـ index في `pinctrl-names` بيقابل رقم `pinctrl-N`.
## Phase 5: دليل الـ Debugging الشامل

الـ `pinctrl-state.h` بيعرّف الـ **standard pin control state names** اللي بيستخدمها الـ pinctrl subsystem عشان يحدد حالة الـ pins في كل مرحلة من مراحل lifecycle الـ device — `default`, `init`, `idle`, `sleep`. الـ debugging هنا بيتمحور حول التحقق إن الـ pin state بيتغير صح في كل transition.

---

### Software Level

#### 1. debugfs Entries

**الـ** debugfs للـ pinctrl موجود في `/sys/kernel/debug/pinctrl/`:

```bash
# اعرض كل الـ pinctrl controllers المتاحة
ls /sys/kernel/debug/pinctrl/

# مثال على output:
# pinctrl-0  pinctrl-maps  pinctrl-handles

# اعرض كل الـ pin groups المسجلة
cat /sys/kernel/debug/pinctrl/<controller-name>/pingroups

# اعرض كل الـ pin functions
cat /sys/kernel/debug/pinctrl/<controller-name>/pins

# اعرض الـ pinmux المفعّل حالياً لكل pin
cat /sys/kernel/debug/pinctrl/<controller-name>/pinmux-pins

# اعرض الـ pinmux functions
cat /sys/kernel/debug/pinctrl/<controller-name>/pinmux-functions

# اعرض كل الـ pin control mappings (دي الأهم للـ state debugging)
cat /sys/kernel/debug/pinctrl/pinctrl-maps

# اعرض الـ handles (كل device وحالته الحالية)
cat /sys/kernel/debug/pinctrl/pinctrl-handles
```

**تفسير الـ output من `pinctrl-handles`:**

```
device: 1c28000.serial
state: default          <-- الـ state الحالي للـ device
  type: MUX_GROUP ...
  type: CONFIGS_GROUP ...

device: 1c28000.serial
state: sleep
  ...
```

لو شايف `state: init` بعد الـ probe ده يعني الـ probe فشل أو ما اتكملش صح.

#### 2. sysfs Entries

الـ pinctrl مش بيعمل expose كبير في sysfs، لكن:

```bash
# تحقق من الـ runtime PM state اللي بيحدد أي pinctrl state يتفعّل
cat /sys/bus/platform/devices/<device>/power/runtime_status
# output: active | suspended | error

# تحقق من الـ power/wakeup
cat /sys/bus/platform/devices/<device>/power/wakeup

# تحقق من الـ gpio direction و value اللي بيتأثر بالـ pinctrl state
cat /sys/class/gpio/gpio<N>/direction
cat /sys/class/gpio/gpio<N>/value
```

#### 3. ftrace — Tracepoints

```bash
# فعّل الـ tracing للـ pinctrl
cd /sys/kernel/debug/tracing

# اعرض كل الـ pinctrl events المتاحة
ls events/pinctrl/

# الـ events الأساسية:
# pinctrl_select_state  -- أهم event: بيتتبع كل transition بين states

# فعّل event معين
echo 1 > events/pinctrl/pinctrl_select_state/enable

# ابدأ الـ tracing
echo 1 > tracing_on

# اعمل الـ operation اللي عايز تتبعها (مثلاً: suspend/resume)
echo mem > /sys/power/state  # أو أي driver operation

# اقرأ النتيجة
cat trace
```

**مثال على output:**

```
kworker/0:2-45    [000] ....  1234.5678: pinctrl_select_state: device 1c28000.serial group default
```

لو الـ state اللي بتتوقعه (مثلاً `sleep`) ما ظهرش في الـ trace، معناه الـ driver ما بيعمل `pinctrl_select_state()` صح في الـ suspend path.

```bash
# تابع الـ pin config changes
echo 1 > events/pinctrl/pinctrl_setting_mux/enable
echo 1 > events/pinctrl/pinctrl_setting_configs/enable

# فعّل filter على device معين
echo 'name == "1c28000.serial"' > events/pinctrl/pinctrl_select_state/filter
```

#### 4. printk / Dynamic Debug

```bash
# فعّل الـ dynamic debug لكل الـ pinctrl subsystem
echo 'module pinctrl_core +p' > /sys/kernel/debug/dynamic_debug/control
echo 'module pinctrl_utils +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل لملف معين
echo 'file core.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file pinmux.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file pinconf.c +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ state selection messages
dmesg | grep -i pinctrl
dmesg | grep -E "(default|init|idle|sleep)" | grep -i pin

# فعّل الـ kernel boot parameter لـ debug level
# أضف في cmdline:
# pinctrl.debug=1
```

**في الـ driver نفسه لو محتاج تضيف debug:**

```c
/* strategic printk في الـ probe */
dev_dbg(dev, "pinctrl state transition: %s -> %s\n",
        current_state_name, PINCTRL_STATE_DEFAULT);
```

#### 5. Kernel Config Options

| Config Option | الغرض |
|---|---|
| `CONFIG_PINCTRL` | الـ pinctrl subsystem الأساسي |
| `CONFIG_DEBUG_PINCTRL` | يفعّل verbose logging في الـ pinctrl core |
| `CONFIG_PINMUX` | دعم الـ pin multiplexing |
| `CONFIG_PINCONF` | دعم الـ pin configuration |
| `CONFIG_GENERIC_PINCTRL_GROUPS` | generic group support |
| `CONFIG_GENERIC_PINMUX_FUNCTIONS` | generic function support |
| `CONFIG_DEBUG_FS` | مطلوب لـ debugfs entries |
| `CONFIG_TRACING` | مطلوب للـ ftrace events |
| `CONFIG_PM_DEBUG` | debug لـ PM transitions اللي بتؤثر على pin states |
| `CONFIG_PM_SLEEP_DEBUG` | debug لـ suspend/resume pin state changes |
| `CONFIG_PROVE_LOCKING` | يكتشف deadlocks في الـ pinctrl locks |

```bash
# تحقق من الـ config في kernel حالياً
zcat /proc/config.gz | grep -E "CONFIG_(PINCTRL|PINMUX|PINCONF|DEBUG_PINCTRL)"
# أو
grep -E "CONFIG_(PINCTRL|PINMUX|PINCONF)" /boot/config-$(uname -r)
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# pinctrl tool من الـ userspace (لو متاح)
# بعض distros بتيجي مع busybox أو buildroot tools

# استخدم /sys/kernel/debug/pinctrl كـ primary tool
# اعمل script لمتابعة state changes

#!/bin/bash
# مراقبة الـ pinctrl state للـ device
DEVICE="1c28000.serial"
watch -n 1 'grep -A5 "'$DEVICE'" /sys/kernel/debug/pinctrl/pinctrl-handles'

# تحقق من الـ GPIO التابعة للـ pinctrl
gpioinfo   # من package gpiod
gpiodetect
gpiomon <chip> <line>  # راقب تغيير الـ pin value
```

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ Kernel Log | المعنى | الحل |
|---|---|---|
| `could not get pinctrl default` | الـ DT ما فيهوش `pinctrl-0` أو `pinctrl-names = "default"` | أضف الـ pinctrl bindings في الـ DT |
| `could not set pins to default state` | الـ state موجود في DT لكن الـ pinctrl driver فشل في تطبيقه | تحقق من الـ pin conflicts وvalidity الـ group |
| `pin X already requested` | pin بتحاول تحجزه driver تاني | ابحث عن conflict في الـ DT أو الـ driver |
| `could not get pinctrl sleep` | `pinctrl-names` ما فيهاش `"sleep"` أو مش متطابقة مع `pinctrl-N` | اتأكد إن ترتيب `pinctrl-names` متطابق مع `pinctrl-0`, `pinctrl-1`, ... |
| `could not get pinctrl init` | نفس المشكلة للـ `init` state | أضف `init` state في DT أو استخدم `devm_pinctrl_get_select_default` |
| `pin in wrong state` | الـ hardware state ما بيطابقش الـ expected state | راجع الـ register dump للـ pinctrl controller |
| `maps not found for device` | الـ device ما عندوش pinctrl map في الـ DT | أضف `pinctrl-0` و `pinctrl-names` في الـ device node |
| `pinctrl_select_state: failed` | فشل عام في الـ state transition | فعّل `CONFIG_DEBUG_PINCTRL` وشوف الـ detailed error |
| `pin already in use` | تعارض بين deviceين على نفس الـ pin | راجع الـ DT وتأكد كل pin بيتخدم لـ device واحد بس |

#### 8. Strategic Points لـ dump_stack() و WARN_ON()

```c
/* في الـ driver اللي بيستخدم pinctrl states */

static int mydrv_probe(struct platform_device *pdev)
{
    struct mydrv *drv = ...;

    drv->pinctrl = devm_pinctrl_get(&pdev->dev);
    if (IS_ERR(drv->pinctrl)) {
        /* WARN هنا مفيد لأن probe failure مش دايماً obvious */
        dev_warn(&pdev->dev, "no pinctrl handle\n");
        /* مش WARN_ON لأن بعض devices مش محتاجة pinctrl */
    }

    drv->pins_default = pinctrl_lookup_state(drv->pinctrl,
                                              PINCTRL_STATE_DEFAULT);
    /* WARN_ON لو الـ default state مش موجود — ده unexpected */
    if (WARN_ON(IS_ERR(drv->pins_default)))
        return PTR_ERR(drv->pins_default);

    return 0;
}

static int mydrv_suspend(struct device *dev)
{
    struct mydrv *drv = dev_get_drvdata(dev);
    int ret;

    if (!IS_ERR_OR_NULL(drv->pins_sleep)) {
        ret = pinctrl_select_state(drv->pinctrl, drv->pins_sleep);
        /* WARN_ON هنا لأن فشل الـ sleep transition خطير */
        if (WARN_ON(ret))
            dev_err(dev, "failed to go to sleep state: %d\n", ret);
    }
    return ret;
}

/* نقطة استراتيجية: تحقق إن الـ init state اتبدّل لـ default بعد probe */
static void mydrv_verify_state_transition(struct mydrv *drv)
{
    /* لو الـ pins لسه في init بعد probe → مشكلة */
    if (drv->pinctrl_state == drv->pins_init) {
        WARN(1, "pins still in init state after probe!\n");
        dump_stack();
    }
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State بيطابق الـ Kernel State

الـ **PINCTRL_STATE_DEFAULT** المفروض يعني:
- الـ pins configured كـ alternate function (مش GPIO)
- الـ pull-up/pull-down configured صح
- الـ drive strength على القيمة الصح

```bash
# خطوة 1: اعرف الـ pin number من الـ debugfs
cat /sys/kernel/debug/pinctrl/<controller>/pinmux-pins | grep <device-name>
# output مثال: pin 42 (PA10): 1c28000.serial function uart group uart0-pins

# خطوة 2: اقرأ الـ registers للـ pin دي
# (العنوان بيجي من الـ SoC datasheet)
# مثلاً على Allwinner A64:
# PA10 config register at 0x01c20828
devmem2 0x01c20828 w
```

#### 2. Register Dump Techniques

```bash
# devmem2 — أبسط طريقة
# اقرأ word من عنوان معين
devmem2 <physical_address> w

# مثال: اقرأ الـ UART0 pin config register على Allwinner
devmem2 0x01c20824 w   # PA8-PA15 config

# /dev/mem مع dd (لو devmem2 مش متاح)
dd if=/dev/mem bs=4 count=1 skip=$((0x01c20824 / 4)) 2>/dev/null | xxd

# io tool (من package i2c-tools أو مستقل)
io -4 -r 0x01c20824

# اقرأ range كاملة من registers
for addr in $(seq 0x01c20800 4 0x01c2087c); do
    printf "0x%08x: " $addr
    devmem2 $addr w 2>/dev/null | grep "Read at"
done

# مقارنة الـ register value بالـ expected
# مثال: لو PA10 المفروض تكون UART function (value=2)
CURRENT=$(devmem2 0x01c20828 w 2>/dev/null | grep "Read" | awk '{print $NF}')
BITS=$(( (CURRENT >> 8) & 0xF ))  # bits 11:8 للـ PA10
echo "PA10 function: $BITS (expected 2 for UART)"
```

#### 3. Logic Analyzer / Oscilloscope Tips

**للـ state transitions:**

- **PINCTRL_STATE_DEFAULT**: شوف الـ signal على الـ pin — المفروض يكون نشط (UART TX/RX traffic, SPI clk, etc.)
- **PINCTRL_STATE_IDLE**: الـ signal المفروض يكون هادي — مش static high/low بس مفيش activity
- **PINCTRL_STATE_SLEEP**: الـ pin المفروض يكون في حالة defined (pulled high/low حسب الـ DT config) — ومش floating
- **PINCTRL_STATE_INIT**: نفس الـ sleep أو default حسب الـ hardware requirements

**إعدادات عملية للـ logic analyzer:**

```
- Sample rate: 10x أعلى من أعلى frequency على الـ bus
- Trigger: على الـ first valid transition بعد power-on
- Channels: الـ pin اللي بتختبرها + power-enable signal كـ reference
- Protocol decoder: فعّل الـ UART/SPI/I2C decoder حسب الـ function
```

**علامات المشاكل على الـ oscilloscope:**

| Observation | المشكلة المحتملة |
|---|---|
| Pin floating في الـ sleep state | الـ pull-up/down مش configured في `sleep` state |
| Glitch عند الـ transition لـ default | الـ `init` state مش configured — استخدم `PINCTRL_STATE_INIT` |
| Signal مش بييجي بعد resume | الـ driver ما بيرجعش لـ `default` في `.pm_runtime_resume()` |
| Constant high/low مش expected | الـ pin stuck في `idle` state ومش بيرجع لـ `default` |

#### 4. Common Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log | التفسير |
|---|---|---|
| Pin floating في الـ sleep | `irq X: nobody cared` أو spurious interrupts بعد suspend | الـ pin مش مـ pulled صح في `sleep` state |
| Power leakage | current consumption عالي في الـ suspend | pins في `default` state بدل `sleep` — الـ `suspend()` ما استدعاش `pinctrl_select_state` |
| Glitch عند الـ boot | device probe failure intermittent | محتاج `PINCTRL_STATE_INIT` عشان الـ pin يكون stable قبل الـ probe |
| Communication failure بعد resume | `uart-pl011: ... rx error` أو I2C NAK | الـ resume ما رجعش لـ `default` state |
| GPIO conflict | `gpio-keys: error -16` أو `-EBUSY` | pin محجوز من الـ pinctrl وفي نفس الوقت بيتطلب كـ GPIO |

#### 5. Device Tree Debugging

```bash
# تحقق من الـ DT المحمّل فعلاً (مش الـ source)
# الـ DT المحمّل موجود في /proc/device-tree أو /sys/firmware/devicetree/base/

# ابحث عن device معين
find /proc/device-tree -name "*.compatible" -exec grep -l "pl011\|uart" {} \;

# اقرأ الـ pinctrl properties للـ device
# مثال لـ uart0
cat /proc/device-tree/soc/serial@1c28000/pinctrl-names
# المفروض يطبع: default (أو default\0sleep\0 لو في أكتر من state)

# تحقق من عدد الـ pinctrl states المعرّفة
ls /proc/device-tree/soc/serial@1c28000/ | grep pinctrl

# decode الـ phandles
# pinctrl-0 بيحتوي على phandle بيشاور لـ pin group
xxd /proc/device-tree/soc/serial@1c28000/pinctrl-0

# استخدم dtc لـ decompile الـ DTB
dtc -I dtb -O dts /boot/dtb/$(uname -r)/your-board.dtb 2>/dev/null | \
    grep -A 10 "serial@1c28000"
```

**مثال DT صح:**

```dts
/* في الـ device node */
&uart0 {
    pinctrl-names = "default", "sleep";  /* الترتيب مهم */
    pinctrl-0 = <&uart0_pins_default>;   /* يطابق "default" */
    pinctrl-1 = <&uart0_pins_sleep>;     /* يطابق "sleep" */
    status = "okay";
};

/* في الـ pinctrl node */
uart0_pins_default: uart0-default {
    pins = "PA4", "PA5";
    function = "uart0";
    drive-strength = <20>;
};

uart0_pins_sleep: uart0-sleep {
    pins = "PA4", "PA5";
    function = "gpio_in";
    bias-pull-down;
};
```

**أخطاء DT شائعة:**

```bash
# الخطأ: pinctrl-names فيها 3 entries لكن pinctrl-0,1,2 مش كلها موجودة
# الكشف:
python3 -c "
import os
base = '/proc/device-tree/soc/serial@1c28000'
names_file = os.path.join(base, 'pinctrl-names')
if os.path.exists(names_file):
    with open(names_file, 'rb') as f:
        names = f.read().decode().strip('\x00').split('\x00')
    print(f'pinctrl-names count: {len(names)}: {names}')
    for i, name in enumerate(names):
        p = os.path.join(base, f'pinctrl-{i}')
        exists = os.path.exists(p)
        print(f'pinctrl-{i} ({name}): {\"OK\" if exists else \"MISSING\"}')"
```

---

### Practical Commands

#### جاهز للنسخ — Software Debugging

```bash
#!/bin/bash
# === pinctrl-state Full Debug Script ===
# استخدم: bash pinctrl-debug.sh <device-name>
# مثال:   bash pinctrl-debug.sh "1c28000.serial"

DEVICE="${1:-1c28000.serial}"
PINCTRL_DEBUG_BASE="/sys/kernel/debug/pinctrl"

echo "=== 1. All pinctrl controllers ==="
ls "$PINCTRL_DEBUG_BASE/"

echo ""
echo "=== 2. pinctrl maps (all state bindings) ==="
cat "$PINCTRL_DEBUG_BASE/pinctrl-maps"

echo ""
echo "=== 3. Current state for device: $DEVICE ==="
grep -A 20 "device: $DEVICE" "$PINCTRL_DEBUG_BASE/pinctrl-handles" 2>/dev/null || \
    echo "Device not found in pinctrl-handles"

echo ""
echo "=== 4. Active pinmux assignments ==="
for ctrl in "$PINCTRL_DEBUG_BASE"/*/; do
    CTRLNAME=$(basename "$ctrl")
    echo "--- Controller: $CTRLNAME ---"
    cat "$ctrl/pinmux-pins" 2>/dev/null | grep -v "^pin " | head -20
done

echo ""
echo "=== 5. Pin configurations ==="
for ctrl in "$PINCTRL_DEBUG_BASE"/*/; do
    CTRLNAME=$(basename "$ctrl")
    echo "--- Controller: $CTRLNAME ---"
    cat "$ctrl/pinconf-pins" 2>/dev/null | head -20
done

echo ""
echo "=== 6. Runtime PM state ==="
find /sys/bus/platform/devices/ -name "runtime_status" 2>/dev/null | \
    xargs -I{} sh -c 'echo -n "{}: "; cat {}'
```

```bash
# فعّل ftrace لتتبع كل state transitions
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo > /sys/kernel/debug/tracing/trace
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغّل الـ test (مثلاً suspend/resume)
# ...

echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep -E "(select_state|default|sleep|idle|init)"
```

```bash
# فعّل dynamic debug للـ pinctrl core
echo 'module pinctrl_core +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/pinctrl/pinmux.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ debug output
dmesg -w | grep -E "pinctrl|pin-state|select_state"
```

```bash
# تحقق من الـ kernel config للـ pinctrl debugging
zcat /proc/config.gz 2>/dev/null | grep -E \
    "CONFIG_(PINCTRL|PINMUX|PINCONF|DEBUG_PINCTRL|PM_DEBUG|PM_SLEEP_DEBUG|DEBUG_FS|TRACING)"
```

#### جاهز للنسخ — Hardware Debugging

```bash
# اقرأ الـ pin config registers (مثال: Allwinner A64 UART0 على PA4/PA5)
# PA0-PA7 function select register = 0x01c20800
# كل pin بتاخد 4 bits
devmem2 0x01c20800 w   # PA0-PA7 function config

# حساب الـ offset للـ pin:
# PA4 = bits 19:16, PA5 = bits 23:20
# لو القيمة = 0x22?????
# PA4 function = 2 = UART0_TX ✓
# PA5 function = 2 = UART0_RX ✓

# script لقراءة وتفسير register
PA_CONFIG=$(devmem2 0x01c20800 w 2>/dev/null | grep "Read at" | awk '{print $NF}')
PA4_FUNC=$(( (PA_CONFIG >> 16) & 0xF ))
PA5_FUNC=$(( (PA_CONFIG >> 20) & 0xF ))
echo "PA4 function = $PA4_FUNC (2=UART0_TX, 0=GPIO_IN, 1=GPIO_OUT)"
echo "PA5 function = $PA5_FUNC (2=UART0_RX, 0=GPIO_IN, 1=GPIO_OUT)"
```

```bash
# تحقق من الـ DT compiled bindings
dtc -I dtb -O dts /sys/firmware/fdt 2>/dev/null | \
    grep -B5 -A15 "pinctrl-names"

# أو لو عندك الـ DTB file
dtc -I dtb -O dts /boot/dtbs/$(uname -r)/*.dtb 2>/dev/null | \
    grep -B2 -A10 'pinctrl-names = "default"'
```

```bash
# مقارنة شاملة: DT state definitions vs hardware reality
echo "=== Expected from DT ==="
grep -A5 "device: 1c28000.serial" /sys/kernel/debug/pinctrl/pinctrl-handles

echo "=== Actual hardware pin functions ==="
# اقرأ الـ pin config register وقارنه بالـ expected
# (العناوين من الـ SoC datasheet)

echo "=== Current runtime PM state ==="
cat /sys/bus/platform/devices/1c28000.serial/power/runtime_status
```

```bash
# مراقبة state transitions في real-time
watch -d -n 0.5 'grep -A8 "device: 1c28000.serial" \
    /sys/kernel/debug/pinctrl/pinctrl-handles 2>/dev/null'
```
## Phase 6: سيناريوهات من الحياة العملية

---

### سيناريو 1: Industrial Gateway على RK3562 — UART بيتقطع بعد الـ suspend

#### العنوان
UART console بيتوقف بعد الرجوع من الـ sleep على gateway صناعي

#### السياق
شركة بتعمل **industrial gateway** بيستخدم RK3562، الـ gateway بيتوصل بـ MODBUS devices عن طريق RS-485. الـ gateway بيدخل **system suspend** كل ساعة لتوفير الطاقة، وبعد الـ resume الـ UART بيرجع بس البيانات الجاية من الـ RS-485 بتتقطع.

#### المشكلة
الـ UART driver بيشتغل تمام بعد الـ resume، بس الـ pins مش بترجع لحالة الـ drive strength والـ pull-up الصح. النتيجة: signal integrity ضعيفة وبيانات corrupt.

#### التحليل
الـ driver بيعمل `pinctrl_pm_select_sleep_state()` في `suspend()` — ده بيحط الـ pins في state اسمها `"sleep"` — اللي بتتعرف من:

```c
/* pinctrl-state.h */
#define PINCTRL_STATE_SLEEP "sleep"
```

في الـ `resume()` المفروض يرجع لـ `"default"`:

```c
#define PINCTRL_STATE_DEFAULT "default"
```

بس المشكلة إن الـ Device Tree مش فيه `pinctrl-1` (اللي هو الـ sleep state) مضبوط صح:

```dts
/* غلط — مفيش pinctrl-1 */
&uart2 {
    pinctrl-names = "default";
    pinctrl-0 = <&uart2_pins>;
};
```

فلما الـ kernel بيحاول يعمل `pinctrl_select_state()` بالـ string `"sleep"`، مش بيلاقيه، فالـ pins بتفضل في حالة undefined بعد الـ resume.

#### الحل

**1. صلح الـ DT:**

```dts
&uart2 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart2_pins_active>;
    pinctrl-1 = <&uart2_pins_sleep>;
};
```

```dts
uart2_pins_active: uart2-active {
    rockchip,pins =
        <1 RK_PB2 2 &pcfg_pull_up>,   /* TX */
        <1 RK_PB3 2 &pcfg_pull_up>;   /* RX */
};

uart2_pins_sleep: uart2-sleep {
    rockchip,pins =
        <1 RK_PB2 0 &pcfg_pull_none>, /* GPIO input — no drive */
        <1 RK_PB3 0 &pcfg_pull_none>;
};
```

**2. تحقق بعد الـ resume:**

```bash
# شوف الـ state الحالية
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip/pinmux-pins | grep uart2
```

#### الدرس المستفاد
الـ `PINCTRL_STATE_SLEEP` مش magic — لو مش معرّف في الـ DT كـ `pinctrl-names = "...", "sleep"` الـ kernel مش بيطبقه، والـ pins بتفضل في حالة غير محددة بعد الـ resume.

---

### سيناريو 2: Android TV Box على Allwinner H616 — HDMI بيطفي بعد idle

#### العنوان
شاشة HDMI بتسود بعد دقيقتين من الـ inactivity على Android TV box

#### السياق
منتج **Android TV box** بيستخدم Allwinner H616. المستخدمين بيشتكوا إن الشاشة بتسود فجأة حتى لو في video شغال، وبترجع بس لما يحركوا الريموت.

#### المشكلة
الـ HDMI PHY driver بيستخدم `pm_runtime_idle()` بشكل عدواني، وبيحط الـ pins في state `"idle"` اللي بتوقف الـ TMDS clock lines.

#### التحليل
الـ `pinctrl-state.h` بيعرّف:

```c
#define PINCTRL_STATE_IDLE "idle"
```

الـ HDMI driver عنده:

```c
static int sun50i_hdmi_runtime_idle(struct device *dev)
{
    /* بيحط الـ TMDS pins في idle — بيوقف الـ clock */
    return pinctrl_pm_select_idle_state(dev);
}
```

الـ DT عنده `pinctrl-2` للـ idle بيعمل الـ TMDS pins كـ GPIO input:

```dts
hdmi_pins_idle: hdmi-idle {
    pins = "PH4", "PH5", "PH6", "PH7"; /* TMDS D+/D- */
    function = "gpio_in";
    bias-disable;
};
```

ده بيوقف الـ HDMI signal تماماً — الشاشة بتسود.

#### الحل

**Option A:** شيل الـ idle state من الـ DT خالص لو الـ HDMI مش المفروض يوقف:

```dts
/* قبل */
pinctrl-names = "default", "sleep", "idle";

/* بعد */
pinctrl-names = "default", "sleep";
```

**Option B:** لو عايز توفر طاقة بدون قطع الشاشة، استخدم state خفيف:

```dts
hdmi_pins_idle: hdmi-idle {
    pins = "PH4", "PH5", "PH6", "PH7";
    function = "hdmi";         /* فضل على الـ HDMI function */
    drive-strength = <10>;     /* قلل الـ drive بس متقطعش */
};
```

**Debug:**

```bash
# شوف لما بيتغير الـ state
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable
cat /sys/kernel/debug/tracing/trace_pipe | grep hdmi
```

#### الدرس المستفاد
الـ `PINCTRL_STATE_IDLE` مش دايمًا آمن لكل peripheral — الـ HDMI بيحتاج الـ clock lines active طول الوقت، وحط الـ pins في idle ممكن يقطع الـ video signal.

---

### سيناريو 3: IoT Sensor على STM32MP1 — SPI بيعمل glitch في الـ probe

#### العنوان
SPI sensor readings غلط في أول 50ms بعد الـ boot على IoT node

#### السياق
**IoT sensor node** بيستخدم STM32MP157 مع IMU sensor على SPI. في أول 50ms بعد الـ boot، الـ sensor readings فيها glitches — بعدين بتستقر. المنتج بيستخدم الـ readings دي لـ calibration.

#### المشكلة
الـ SPI pins بتتحول لـ `"default"` قبل ما الـ sensor يكمل تسلسل الـ initialization الداخلي، ده بيسبب الـ CS line يتحرك بشكل غير متوقع.

#### التحليل
الـ `pinctrl-state.h` بيعرّف:

```c
#define PINCTRL_STATE_INIT  "init"
#define PINCTRL_STATE_DEFAULT "default"
```

الـ pinctrl subsystem بيعمل كالتالي:
1. لو في state اسمها `"init"` → يطبقها **قبل** `probe()`
2. لو الـ driver خلص الـ `probe()` والـ pins لسه في `"init"` → يحولهم **تلقائيًا** لـ `"default"`

الـ SPI driver مش عارف يـ delay الـ transition دي، لأن الـ kernel بيعملها automatically.

المشكلة: الـ DT مفيهوش `"init"` state، فالـ `"default"` بيتطبق مباشرة، والـ CS line بيتحرك وهو الـ sensor لسه بيـ boot.

```dts
/* مفيش init state */
&spi1 {
    pinctrl-names = "default";
    pinctrl-0 = <&spi1_pins>;
};
```

#### الحل

**1. أضف `"init"` state يخلي الـ CS high طول فترة الـ sensor boot:**

```dts
&spi1 {
    pinctrl-names = "init", "default", "sleep";
    pinctrl-0 = <&spi1_pins_init>;
    pinctrl-1 = <&spi1_pins_active>;
    pinctrl-2 = <&spi1_pins_sleep>;
};
```

```dts
spi1_pins_init: spi1-init {
    /* CS مرفوع كـ GPIO — مش SPI function — عشان ما يحصلش glitch */
    pins1 {
        pinmux = <STM32_PINMUX('A', 4, GPIO)>; /* CS */
        output-high;
        bias-disable;
    };
    pins2 {
        pinmux = <STM32_PINMUX('A', 5, AF5)>, /* CLK */
                 <STM32_PINMUX('A', 6, AF5)>, /* MISO */
                 <STM32_PINMUX('A', 7, AF5)>; /* MOSI */
    };
};
```

**2. في الـ driver، انتظر حتى الـ sensor يكمل init قبل ما يخلي الـ pins تتحول:**

```c
static int imu_probe(struct spi_device *spi)
{
    /* الـ pins لسه في "init" — CS high */
    msleep(50); /* انتظر الـ sensor boot */

    /* بعد الـ probe، الـ kernel هيحولهم لـ "default" تلقائيًا */
    return imu_init_registers(spi);
}
```

#### الدرس المستفاد
الـ `PINCTRL_STATE_INIT` موجود بالظبط لحالات زي دي — لو الـ peripheral محتاج وقت قبل ما الـ pins تبدأ تشتغل كـ SPI/I2C/etc، عرّف `"init"` state بيحمي الـ signal طول فترة الـ driver probe.

---

### سيناريو 4: Automotive ECU على i.MX8QM — CAN bus اتحرق بسبب الـ sleep state

#### العنوان
CAN transceiver اتحرق على automotive ECU بعد الدخول في deep sleep

#### السياق
**Automotive ECU** بيستخدم NXP i.MX8QM مع CAN bus لـ vehicle network. في بعض السيارات، الـ ECU بيدخل **deep sleep** لما الـ ignition يتفل. بعد شوية، الـ CAN transceiver (TJA1044) اتحرق.

#### المشكلة
لما الـ system بيدخل sleep، الـ CAN TX pin مش بيتحول لـ high-impedance — بيفضل driven low. ده بيسبب الـ transceiver يحاول يـ drive الـ CAN bus low طول الوقت، وبيتحرق من الـ current.

#### التحليل
الـ `pinctrl-state.h` بيعرّف:

```c
#define PINCTRL_STATE_SLEEP "sleep"
```

الـ CAN driver بيعمل:

```c
static int flexcan_suspend(struct device *dev)
{
    pinctrl_pm_select_sleep_state(dev);
    /* ... */
}
```

بس الـ sleep state في الـ DT كانت:

```dts
/* غلط — TX لسه driven */
can1_pins_sleep: can1-sleep {
    fsl,pins = <
        MX8QM_UART0_RXD_ADMA_FLEXCAN0_TX  0x00000021  /* TX driven low! */
        MX8QM_UART0_TXD_ADMA_FLEXCAN0_RX  0x00000021
    >;
};
```

الـ config value `0x21` بيعني output-low — ده اتعملش قصداً بس حصل لأن copy-paste من الـ UART config.

#### الحل

**1. صلح الـ sleep state — TX لازم يبقى high-impedance:**

```dts
can1_pins_sleep: can1-sleep {
    fsl,pins = <
        /* TX: input, no pull — high-Z, مش بيـ drive الـ bus */
        MX8QM_UART0_RXD_ADMA_FLEXCAN0_TX  0x00000060
        /* RX: input with pull-up */
        MX8QM_UART0_TXD_ADMA_FLEXCAN0_RX  0x00000021
    >;
};
```

**2. verify قبل الـ production:**

```bash
# تأكد إن الـ TX pin مش driven في الـ sleep state
i2cget -y 0 0x48 0x00  # لو في GPIO expander بتراقب الـ CAN bus

# أو استخدم oscilloscope على CAN_TX أثناء الـ suspend
echo mem > /sys/power/state
# قيس الـ voltage على CAN_TX — المفروض يكون recessive (3.5V) مش dominant (0V)
```

**3. أضف protection في الـ driver:**

```c
static int flexcan_suspend(struct device *dev)
{
    struct flexcan_priv *priv = dev_get_drvdata(dev);

    /* أوقف الـ transceiver أولاً قبل تغيير الـ pins */
    gpiod_set_value(priv->transceiver_en, 0);
    udelay(10);

    return pinctrl_pm_select_sleep_state(dev);
}
```

#### الدرس المستفاد
الـ `PINCTRL_STATE_SLEEP` في الأنظمة الـ safety-critical زي الـ automotive لازم يتـ review بعناية شديدة — pin configuration غلط في الـ sleep state ممكن يحرق hardware، مش بس يسبب software bug.

---

### سيناريو 5: Custom Board Bring-up على AM62x — I2C مش بيشتغل من أول boot

#### العنوان
I2C bus مش بيـ respond في أول تشغيل للـ board — بس بيشتغل بعد reboot

#### السياق
فريق hardware بيعمل **custom board bring-up** على TI AM62x SoC. الـ board عليها PMIC على I2C0 والـ PMIC بيـ control الـ core voltage. في أول boot، الـ kernel بيـ panic لأنه مش قادر يوصل للـ PMIC — بس بعد hard reboot بيشتغل.

#### المشكلة
الـ bootloader (U-Boot) بيسيب الـ I2C pins في state غير متوقعة — drive strength عالي و pull-down enabled. أول probe للـ I2C driver، الـ pins بتكون في حالة غلط، فالـ bus بيبدو busy (SDA low).

#### التحليل
الـ kernel بيطبق `PINCTRL_STATE_DEFAULT` قبل الـ `probe()`:

```c
/* من pinctrl-state.h */
#define PINCTRL_STATE_DEFAULT "default"
```

بس المشكلة إن الـ "default" state في الـ DT بتفترض إن الـ pin state الأولي neutral، وده مش صح لأن U-Boot سيّب الـ pins بـ pull-down.

الـ I2C bus بيعتبر busy لو SDA أو SCL كانوا low — وده بيحصل لأن الـ pull-down من U-Boot أقوى من الـ pull-up في الـ default state لفترة قصيرة.

لو في `"init"` state، كان ممكن يتعامل مع الموضوع ده:

```c
/* pinctrl subsystem order:
 * 1. apply "init" state (لو موجود)
 * 2. call probe()
 * 3. لو pins لسه في "init" → switch to "default"
 */
```

#### الحل

**1. عرّف `"init"` state بتـ reset الـ pins بشكل صريح:**

```dts
&i2c0 {
    pinctrl-names = "init", "default";
    pinctrl-0 = <&i2c0_pins_init>;
    pinctrl-1 = <&i2c0_pins_active>;
};
```

```dts
i2c0_pins_init: i2c0-init {
    pinctrl-single,pins = <
        /* GPIO output-high مؤقتاً — بيضمن SDA/SCL high قبل الـ I2C init */
        AM62X_IOPAD(0x0260, PIN_OUTPUT_PULLUP, 7) /* I2C0_SDA as GPIO */
        AM62X_IOPAD(0x0264, PIN_OUTPUT_PULLUP, 7) /* I2C0_SCL as GPIO */
    >;
};

i2c0_pins_active: i2c0-active {
    pinctrl-single,pins = <
        AM62X_IOPAD(0x0260, PIN_INPUT_PULLUP, 0) /* I2C0_SDA */
        AM62X_IOPAD(0x0264, PIN_INPUT_PULLUP, 0) /* I2C0_SCL */
    >;
};
```

**2. في الـ I2C driver، ضيف I2C bus recovery قبل الاستخدام:**

```c
static int am65_i2c_probe(struct platform_device *pdev)
{
    struct i2c_adapter *adap;

    /* الـ pins دلوقتي في "init" — SDA/SCL high كـ GPIO */
    /* عمل 9 clock pulses لـ reset أي stuck slave */
    am65_i2c_recover_bus(pdev);

    /* بعد الـ probe، الـ kernel هيحول لـ "default" (I2C function) */
    return am65_i2c_init(pdev);
}
```

**3. debug الـ pin state قبل وبعد:**

```bash
# شوف الـ pin state الحالية
cat /sys/kernel/debug/pinctrl/pinctrl-single/pinmux-pins | grep i2c0

# اتحقق إن الـ bus مش busy
i2cdetect -y 0
# لو بيقول "Bus busy" → الـ SDA لسه low

# force الـ pin لـ GPIO وارفع SDA يدوياً للـ debug
gpioset 0 5=1  # SDA
gpioset 0 6=1  # SCL
i2cdetect -y 0  # جرب تاني
```

#### الدرس المستفاد
الـ `PINCTRL_STATE_INIT` مفيدة جداً في الـ board bring-up — بتديك فرصة تـ reset الـ pins لحالة معروفة قبل ما الـ driver يبدأ، خصوصاً لو الـ bootloader بيسيب الـ hardware في حالة غير محددة. من غير `"init"` state، الـ `"default"` بيتطبق على pins في حالة مجهولة.
## Phase 7: مصادر ومراجع

### مقدمة

الـ `pinctrl-state.h` جزء من subsystem أكبر بكثير — الـ **pin control subsystem** اللي بيتحكم في كل حاجة من pin muxing لـ pin configuration في Linux kernel. المصادر دي هتساعدك تفهم الـ big picture وتتعمق في التفاصيل.

---

### توثيق الـ Kernel الرسمي

| المصدر | الوصف |
|--------|--------|
| [`Documentation/pinctrl.txt`](https://www.kernel.org/doc/Documentation/pinctrl.txt) | التوثيق الرسمي الأصلي، مكتوب بواسطة Linus Walleij، بيشرح كل جزء في الـ subsystem |
| [`Documentation/driver-api/pinctl.rst`](https://www.kernel.org/doc/html/v4.14/driver-api/pinctl.html) | النسخة الحديثة من التوثيق (HTML) على kernel.org |

---

### مقالات LWN.net

دي أهم المقالات اللي بتغطي نشأة وتطور الـ pinctrl subsystem:

#### الأساسيات والنشأة

- **[pin controller subsystem v7](https://lwn.net/Articles/459190/)** — أول patch series رسمية قدّمها Linus Walleij لإدخال الـ subsystem في الـ kernel. نقطة البداية الحقيقية.

- **[Documentation/pinctrl.txt](https://lwn.net/Articles/465077/)** — الـ patch اللي أضافت التوثيق الرسمي مع قبول الـ subsystem في الـ mainline kernel.

- **[pinctrl: add a generic pin config interface](https://lwn.net/Articles/468770/)** — إضافة الـ generic pin configuration API (biasing، drive strength، إلخ) اللي `pinctrl-state.h` بيعتمد عليها.

#### تطور الـ States

- **[drivers: pinctrl sleep and idle states in the core](https://lwn.net/Articles/552972/)** — الـ patch اللي أضافت `PINCTRL_STATE_SLEEP` و`PINCTRL_STATE_IDLE` للـ core. ده مباشرةً مرتبط بـ `pinctrl-state.h`.

- **[drivers/pinctrl: Add the concept of an "init" state](https://lwn.net/Articles/615322/)** — الـ patch اللي أضافت `PINCTRL_STATE_INIT`. بتشرح ليه المطورين احتاجوا state جديدة قبل `probe()` عشان يتجنبوا الـ glitches.

#### تطورات حديثة

- **[pinctrl: introduce the concept of a GPIO pin function category](https://lwn.net/Articles/1031226/)** — مقترح حديث لتصنيف pins المـ mux-ed كـ GPIO.

- **[gpio: add pinctrl based generic gpio driver](https://lwn.net/Articles/946636/)** — إضافة generic GPIO driver مبني على الـ pinctrl framework.

- **[Add pinctrl support for the AAEON UP board FPGA](https://lwn.net/Articles/1033269/)** — مثال حديث على إضافة pinctrl support لـ hardware جديد.

---

### مناقشات الـ Mailing List

الـ **linux-gpio mailing list** هو المكان الرئيسي لنقاشات الـ pinctrl (مش LKML مباشرةً):

- **Lore Kernel Archive للـ pinctrl**: [`https://lore.kernel.org/linux-gpio/`](https://lore.kernel.org/linux-gpio/) — ابحث عن `pinctrl-state` أو `PINCTRL_STATE_SLEEP`

- **مراسلة Linus Walleij**: [`https://lore.kernel.org/all/CACRpkdb0mLzZMyMejMYTFvcsPjX8sADbkrekU7AFXbKc-MJttA@mail.gmail.com/`](https://lore.kernel.org/all/CACRpkdb0mLzZMyMejMYTFvcsPjX8sADbkrekU7AFXbKc-MJttA@mail.gmail.com/) — مثال على أسلوب review الـ pinctrl patches.

- **LKML Archive**: [`https://lkml.org/`](https://lkml.org/) — ابحث عن `"pinctrl-state"` للـ threads المتعلقة بتعريف الـ states.

---

### Commits مهمة في الـ Kernel

الـ commits دي علامات في تاريخ `pinctrl-state.h`:

| الـ Commit | الوصف |
|-----------|--------|
| `2744e8af` | إدخال الـ pinctrl subsystem الأساسي مع `PINCTRL_STATE_DEFAULT` |
| إضافة sleep/idle | الـ patch series على [LWN 552972](https://lwn.net/Articles/552972/) بتحتوي على الـ commit hash |
| إضافة init state | الـ patch series على [LWN 615322](https://lwn.net/Articles/615322/) |

للبحث عن الـ commits مباشرةً:
```bash
git log --oneline -- include/linux/pinctrl/pinctrl-state.h
git log --all --grep="PINCTRL_STATE_SLEEP" --oneline
```

---

### كود المصدر المهم في الـ Kernel

```
include/linux/pinctrl/pinctrl-state.h   ← الملف ده نفسه (State name macros)
include/linux/pinctrl/consumer.h        ← API للـ drivers اللي بتستخدم الـ states
include/linux/pinctrl/pinctrl.h         ← Core pinctrl structures
drivers/pinctrl/core.c                  ← التطبيق الفعلي للـ state machine
drivers/base/pinctrl.c                  ← ربط الـ states بـ device lifecycle
```

على GitHub:
- [`include/linux/pinctrl/pinctrl.h`](https://github.com/torvalds/linux/blob/master/include/linux/pinctrl/pinctrl.h)
- [`drivers/pinctrl/core.c`](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/core.c)

---

### مصادر eLinux.org

- **[Tests: i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl)** — توثيق عملي لـ driver بيستخدم الـ pinctrl states لتبديل الـ I2C bus بين pins مختلفة — مثال حي على `PINCTRL_STATE_DEFAULT`.

- **[EBC Device Trees](https://elinux.org/EBC_Device_Trees)** — أمثلة عملية على استخدام `pinctrl-0`, `pinctrl-names = "default"` في Device Tree للـ BeagleBone — بتوضح ازاي الـ `PINCTRL_STATE_DEFAULT` بيتحدد في الـ DT.

- **[Pin Control & GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf)** — presentation من ELC بتشرح العلاقة بين الـ pinctrl subsystem والـ GPIO subsystem.

---

### Kernelnewbies.org

الـ site ده بيوثّق كل تغيير في كل إصدار kernel. ابحث في الصفحات دي عن `pinctrl`:

- [Linux 6.11 Changes](https://kernelnewbies.org/Linux_6.11) — تغييرات الـ pinctrl في kernel 6.11
- [Linux 6.8 Changes](https://kernelnewbies.org/Linux_6.8) — إضافات drivers جديدة لـ pinctrl
- [Linux 6.1 Changes](https://kernelnewbies.org/Linux_6.1) — تحديثات الـ subsystem
- [LinuxChanges (All versions)](https://kernelnewbies.org/LinuxChanges) — الفهرس الكامل لكل تغييرات الـ kernel

---

### كتب مقترحة

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل المهم**: Chapter 1 (Device Model) — يفهّمك الـ `struct device` اللي الـ pinctrl بيتعامل معاها
- **متاح مجاناً**: [`https://lwn.net/Kernel/LDD3/`](https://lwn.net/Kernel/LDD3/)
- **ملاحظة**: LDD3 قديم (2005) ومش بيتكلم عن الـ pinctrl subsystem مباشرةً لأنه ظهر بعده، لكن أساسيات الـ kernel driver model لازالت سارية.

#### Linux Kernel Development, 3rd Edition (Robert Love)
- **الفصل المهم**: Chapter 14 (The Block I/O Layer) و Chapter 17 (Devices and Modules) — بيشرح الـ device model الأساسي
- **الأهمية لـ pinctrl**: فهم الـ `pm_runtime` و `suspend/resume` اللي `PINCTRL_STATE_SLEEP` و`PINCTRL_STATE_IDLE` مبنيين عليهم

#### Embedded Linux Primer, 2nd Edition (Christopher Hallinan)
- **الفصل المهم**: Chapter 15 (Kernel Initialization) وأي chapter عن Device Tree
- **الأهمية**: بيشرح ازاي الـ embedded systems بتحتاج pin multiplexing وده بالظبط الـ use case الأساسي لـ `pinctrl-state.h`

#### Professional Linux Kernel Architecture (Wolfgang Mauerer)
- بيتكلم بتفصيل عن الـ power management states اللي `PINCTRL_STATE_IDLE` و`PINCTRL_STATE_SLEEP` جزء منها

---

### مقال embedded.com

- **[Linux device driver development: The pin control subsystem](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/)** — شرح عملي ممتاز بأمثلة كود حقيقية، مناسب جداً للمبتدئين في الـ pinctrl.

---

### مصطلحات البحث

لو عايز تدور على معلومات أكتر، استخدم الـ search terms دي:

```
# على Google / DuckDuckGo
pinctrl_select_state linux kernel
pinctrl-state.h PINCTRL_STATE_DEFAULT
linux pinctrl "pm_runtime_suspend" "idle state"
linux pinctrl "init state" probe glitch
linux device tree "pinctrl-names" "default" "sleep"
PINCTRL_STATE_SLEEP power management pins

# على lore.kernel.org
https://lore.kernel.org/linux-gpio/?q=pinctrl-state

# على Elixir Cross-Referencer (مصدر ممتاز!)
https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/pinctrl-state.h
```

---

### الـ Elixir Cross-Referencer

أداة لازم تعرفها: [Elixir Bootlin](https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/pinctrl-state.h) — بتعرض الكود مع كل الـ references والـ callers مباشرةً في الـ browser، مفيد جداً لفهم مين بيستخدم `PINCTRL_STATE_DEFAULT` في الـ kernel.
## Phase 8: Writing simple module

### الفكرة

**`pinctrl_select_state()`** هي الـ function الأهم في الـ pinctrl subsystem — كل device بيغير state (default / sleep / idle) بيمر بيها. هنعمل **kprobe** عليها عشان نشوف مين بيغير الـ pin state ومتى.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on pinctrl_select_state()
 * Logs every pin-state transition: who called it and which state.
 */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/device.h>

/* pinctrl internal struct — we only need the name field.
 * We mirror just what we need; full definition is in drivers/pinctrl/core.h */
struct pinctrl_state {
	struct list_head node;
	const char *name;   /* "default" / "sleep" / "idle" / "init" */
	/* ... rest of fields we don't touch */
};

/* ------------------------------------------------------------------ */
/*  Pre-handler: runs BEFORE pinctrl_select_state() executes           */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
	/*
	 * pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *s)
	 *
	 * On x86-64:  rdi = first arg (pinctrl*), rsi = second arg (pinctrl_state*)
	 * On ARM64:   x0  = first arg,            x1  = second arg
	 */
#if defined(CONFIG_X86_64)
	struct pinctrl_state *state = (struct pinctrl_state *)regs->si;
#elif defined(CONFIG_ARM64)
	struct pinctrl_state *state = (struct pinctrl_state *)regs->regs[1];
#else
	/* fallback — skip logging on unsupported arch */
	return 0;
#endif

	/* guard against NULL before dereferencing */
	if (!state || !state->name)
		return 0;

	pr_info("pinctrl_probe: state transition → \"%s\"  [caller: %pS]\n",
		state->name,
		(void *)regs->ip); /* instruction pointer = caller address */

	return 0; /* 0 = let the real function run normally */
}

/* ------------------------------------------------------------------ */
/*  kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
	.symbol_name = "pinctrl_select_state", /* function to hook        */
	.pre_handler = handler_pre,            /* our callback            */
};

/* ------------------------------------------------------------------ */
/*  init / exit                                                        */
/* ------------------------------------------------------------------ */
static int __init pinctrl_probe_init(void)
{
	int ret;

	ret = register_kprobe(&kp);
	if (ret < 0) {
		pr_err("pinctrl_probe: register_kprobe failed: %d\n", ret);
		return ret;
	}

	pr_info("pinctrl_probe: hooked pinctrl_select_state() at %p\n",
		kp.addr);
	return 0;
}

static void __exit pinctrl_probe_exit(void)
{
	unregister_kprobe(&kp);
	pr_info("pinctrl_probe: kprobe removed\n");
}

module_init(pinctrl_probe_init);
module_exit(pinctrl_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on pinctrl_select_state — log all pin-state transitions");
```

---

### Makefile

```makefile
obj-m += pinctrl_probe.o

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
|---|---|
| `linux/kprobes.h` | يعرّف `struct kprobe` و`register_kprobe` |
| `linux/module.h` | ماكروهات `module_init` / `module_exit` / `MODULE_*` |
| `linux/kernel.h` | `pr_info` / `pr_err` |
| `linux/device.h` | تعريف `struct device` المستخدم في الـ pinctrl API |

#### تعريف `struct pinctrl_state` يدويًا

**الـ** `pinctrl_state` definition موجودة في `drivers/pinctrl/core.h` اللي مش exported للـ modules. بنعرّف نسخة مختصرة بس بنوصل للـ `name` field بس — ده آمن لأن `name` أول حقل بعد `node` وما بنلمسش الباقي.

#### الـ Pre-handler

الـ callback بتشتغل **قبل** ما الدالة الأصلية تتنفذ. بناخد الـ argument التاني (`pinctrl_state*`) من الـ registers حسب الـ ABI:
- **x86-64**: `regs->si` = `rsi` = الـ argument التاني.
- **ARM64**: `regs->regs[1]` = `x1` = الـ argument التاني.

`%pS` في `pr_info` بتطبع اسم الـ function + offset من الـ `ip` — بنعرف مين استدى `pinctrl_select_state`.

#### الـ `kprobe` struct

`symbol_name` بيخلي الـ kernel يحل عنوان الدالة تلقائيًا من الـ kallsyms بدل ما نحدد عنوان يدوي — أنظف وأمن.

#### الـ `module_exit` و `unregister_kprobe`

لازم نشيل الـ kprobe في الـ exit عشان لو ما عملناش ده، الـ handler هيفضل بيشتغل حتى بعد ما الـ module اتشال من الذاكرة، وده بيعمل kernel panic لأن الـ IP هيبقى invalid.

---

### تشغيل وملاحظة الـ Output

```bash
# بناء وتحميل
make
sudo insmod pinctrl_probe.ko

# شوف الـ log في real-time
sudo dmesg -w | grep pinctrl_probe

# مثال على output
# [  42.317] pinctrl_probe: hooked pinctrl_select_state() at ffffffffc0a3e120
# [  43.891] pinctrl_probe: state transition → "default"  [caller: really_probe+0x1a4/0x3c0]
# [  98.002] pinctrl_probe: state transition → "sleep"    [caller: pinctrl_pm_select_sleep_state+0x38/0x60]

# تفريغ الـ module
sudo rmmod pinctrl_probe
```

**الـ** `really_probe` في الـ output ده هو الـ driver core اللي بيعمل `PINCTRL_STATE_DEFAULT` أول ما أي device بيتـ probe — بيوضح إن كل device جديد بيعمل state transition تلقائيًا.
