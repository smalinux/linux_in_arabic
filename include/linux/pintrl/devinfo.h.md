## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ `devinfo.h`** جزء من **PIN CONTROL SUBSYSTEM** في الـ Linux kernel — المسؤول بالكامل عن إدارة الـ pins الفيزيائية على الـ SoC (System on Chip). المسؤول عنه هو Linus Walleij وبيتتبع على قائمة `linux-gpio@vger.kernel.org`.

---

### القصة — ليه اصلاً في مشكلة؟

تخيل عندك شريحة (SoC) زي اللي في الموبايل — فيها مئات الـ pins الفيزيائية. كل pin ممكن يشتغل بأكتر من دور في نفس الوقت:
- **UART TX** (تبعت بيانات)
- **GPIO output** (يتحكم في LED)
- **SPI clock**
- أو حتى يبقى معطل خالص توفير طاقة

ده بيسمى **pin multiplexing (pinmux)**. الـ hardware عنده سجلات بتحدد كل pin بيعمل إيه في كل لحظة.

المشكلة إن كل **device driver** محتاج يقول للـ hardware "انت الـ pins دي خدها وشغلها بالشكل ده". لو كل driver عمل ده بنفسه على طول، هيبقى فيه فوضى وتعارض — درايفر بيغير إعداد pin درايفر تاني محتاجه.

**الحل:** kernel عمل **pin control subsystem** كوسيط مركزي يتحكم في الـ pins. كل device بدل ما يتكلم مع الـ hardware مباشرةً، بيتكلم مع الـ pinctrl subsystem اللي بينظم الموضوع كله.

---

### دور الـ `devinfo.h` تحديداً

الملف ده بيحل سؤال واحد: **إزاي الـ device core نفسه (مش الدرايفر) يتذكر إيه الـ pin states المرتبطة بكل device؟**

كل device في الـ Linux kernel ممثل بـ `struct device`. الـ `devinfo.h` بيضيف لكل device "جيب صغير" اسمه `struct dev_pin_info` جوا الـ `struct device` — الجيب ده بيحتفظ بـ:

| الـ field | بيمثل إيه |
|---|---|
| `p` | الـ pinctrl handle — الرابط الأساسي بين الـ device والـ pinctrl subsystem |
| `default_state` | الحالة العادية لما الـ device شغال بشكل طبيعي |
| `init_state` | حالة مؤقتة أثناء الـ probe قبل ما تبدأ تشتغل |
| `sleep_state` | حالة الـ pins لما الجهاز نايم (suspend) |
| `idle_state` | حالة الـ pins لما الجهاز في runtime idle |

---

### القصة كاملة — من probe لـ suspend

```
[Device يبدأ probe]
        |
        v
  pinctrl_bind_pins()        ← drivers/base/pinctrl.c
        |
        |-- يعمل alloc لـ dev->pins (struct dev_pin_info)
        |-- يجيب الـ pinctrl handle: dev->pins->p
        |-- يدور على "default" state
        |-- يدور على "init" state (لو موجود يبدأ بيه، لو لأ يبدأ بـ default)
        |-- يدور على "sleep" و "idle" states (لو CONFIG_PM مفعل)
        |
        v
  [Driver probe() بتشتغل — الـ pins جاهزة]
        |
        v
  [Runtime suspend]
        |-- pinctrl_pm_select_idle_state()
        |   يروح لـ dev->pins->idle_state
        v
  [System suspend]
        |-- pinctrl_pm_select_sleep_state()
        |   يروح لـ dev->pins->sleep_state
        v
  [Resume]
        |-- pinctrl_pm_select_default_state()
            يرجع لـ dev->pins->default_state
```

---

### ليه الملف ده في الـ device core مش في الـ pinctrl subsystem؟

ده القرار المعماري الأذكى هنا. الـ device core (`drivers/base/`) محتاج يعرف حاجة عن الـ pins لكل device عشان يعمل الـ bind/unbind automatically قبل وبعد الـ probe. بدل ما يـ include كل الـ pinctrl headers الضخمة جوا الـ device core، عملوا ملف صغير خفيف (`devinfo.h`) بس فيه اللي الـ device core محتاجه. ده بيحقق **separation of concerns**.

لو `CONFIG_PINCTRL` معطل خالص، الـ functions بتتحول لـ stubs فارغة والـ overhead = صفر.

---

### الـ Compilation Guard Pattern

```c
#ifdef CONFIG_PINCTRL

/* الكود الحقيقي لما الـ pinctrl موجود */
struct dev_pin_info { ... };
extern int pinctrl_init_done(struct device *dev);
static inline struct pinctrl *dev_pinctrl(struct device *dev) { ... }

#else

/* Stubs فارغة لما الـ pinctrl مش موجود */
static inline int pinctrl_init_done(struct device *dev) { return 0; }
static inline struct pinctrl *dev_pinctrl(struct device *dev) { return NULL; }

#endif
```

ده pattern كلاسيكي في الـ kernel — نفس الـ API بيشتغل سواء الـ feature مفعلة أو لا.

---

### الـ `pinctrl_init_done()` — وظيفتها الخاصة

الـ function دي بتُعلم الـ pinctrl subsystem إن الـ driver خلص الـ probe بنجاح. لو الـ device كان في `init_state` وخلص الـ probe، الـ kernel بيشيله لـ `default_state` automatically. ده بيحل مشكلة الـ drivers اللي محتاجة حالة خاصة أثناء الـ init عشان ماتحصلش glitches على الـ bus.

---

### ASCII — مكان الـ `devinfo.h` في الصورة الكبيرة

```
┌─────────────────────────────────────────────┐
│              struct device                  │
│  ┌─────────────────────────────────────┐   │
│  │  struct dev_pin_info  *pins         │   │  ← مُعرَّف في devinfo.h
│  │  ├── struct pinctrl  *p             │   │
│  │  ├── default_state                  │   │
│  │  ├── init_state                     │   │
│  │  ├── sleep_state  (CONFIG_PM)       │   │
│  │  └── idle_state   (CONFIG_PM)       │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
         │
         │  يتحكم فيه
         ▼
┌─────────────────────────┐
│  pinctrl subsystem core │  ← drivers/pinctrl/core.c
│  (pinctrl_bind_pins)    │  ← drivers/base/pinctrl.c
└─────────────────────────┘
         │
         │  يتكلم مع
         ▼
┌─────────────────────────┐
│  Hardware Pin Controller│  ← drivers/pinctrl/intel/*, freescale/*, ...
└─────────────────────────┘
```

---

### الملفات المرتبطة اللي المفروض تعرفها

#### Core Headers (include/linux/pinctrl/)

| الملف | الدور |
|---|---|
| `devinfo.h` | **الملف ده** — ربط الـ pin info بالـ device struct |
| `consumer.h` | الـ API اللي الـ drivers بتستخدمه للحصول على الـ pins |
| `pinctrl-state.h` | تعريف أسماء الـ states: `"default"`, `"init"`, `"sleep"`, `"idle"` |
| `pinctrl.h` | الـ API للـ pin controller drivers نفسها (الـ producers) |
| `pinmux.h` | الـ API الخاصة بالـ pin multiplexing |
| `pinconf.h` | الـ API الخاصة بإعدادات الـ pins (pull-up, drive strength...) |
| `machine.h` | الـ static pin mappings في الـ board files القديمة |

#### Implementation Files

| الملف | الدور |
|---|---|
| `drivers/base/pinctrl.c` | الكود اللي بيعمل `pinctrl_bind_pins()` قبل الـ probe |
| `drivers/pinctrl/core.c` | قلب الـ pinctrl subsystem |
| `drivers/pinctrl/devicetree.c` | قراءة الـ pin configs من الـ Device Tree |
| `include/linux/device.h` | فيه تعريف `struct device` واللي بيحتوي على `struct dev_pin_info *pins` |

#### Hardware Drivers (أمثلة)

| الملف | الهاردوير |
|---|---|
| `drivers/pinctrl/intel/pinctrl-intel.c` | Intel SoCs |
| `drivers/pinctrl/freescale/pinctrl-imx.c` | NXP i.MX SoCs |
| `drivers/pinctrl/bcm/pinctrl-bcm2835.c` | Raspberry Pi |
| `drivers/pinctrl/pinctrl-single.c` | Generic single-register controllers |
## Phase 2: شرح الـ Pinctrl (Pin Control) Framework

---

### المشكلة اللي الـ Subsystem بيحلها

في أي SoC (System-on-Chip) ARM، فيه مئات الـ pins الجسمانية على الـ chip. كل pin ممكن يشتغل بأكتر من وظيفة — نفس الـ pin ممكن يكون:
- **UART TX** لو بتشتخدمه مع serial port
- **GPIO output** لو بتتحكم فيه مباشرة
- **SPI MOSI** لو محتاج SPI bus
- **PWM output** لو محتاج PWM

ده بيسموه **pin multiplexing (مux)**. وبجانب الـ mux، كل pin ليه **electrical configuration** — pull-up، pull-down، drive strength، open-drain، وهكذا.

**المشكلة الأساسية** إن كل driver كان بيعمل اللي عايزه في registers الـ SoC مباشرة، من غير coordination. النتيجة:
- تضارب بين drivers على نفس الـ pin
- كود مكرر في كل driver
- مفيش مكان واحد يقولك "مين ماسك الـ pin ده دلوقتي"
- صعوبة التعامل مع power management (لازم تغير configuration الـ pins وقت الـ sleep)

---

### الحل — نهج الـ Kernel

الـ kernel حل المشكلة دي بـ **subsystem مركزي** اسمه **pinctrl** (pin control). الفكرة:

1. **مركزية التحكم**: driver واحد متخصص (الـ pinctrl driver الخاص بالـ SoC) هو اللي بيعرف registers الـ hardware ويعمل الـ configuration الفعلي.
2. **Abstraction طبقات**: الـ drivers التانية مش بتكلم الـ hardware مباشرة — بتطلب من الـ pinctrl subsystem.
3. **States بدل تعليمات**: بدل ما driver يقول "set bit X in register Y"، بيقول "اديني الـ pin في state اسمه `default`" — والـ pinctrl subsystem هو اللي يعرف إيه الـ default يعني إيه على الـ hardware الحالي.
4. **Integration مع Device Model**: كل `struct device` في الـ kernel ممكن يكون ليه `struct dev_pin_info` — ده اللي بيعمله الملف `devinfo.h`.

---

### التشبيه من الواقع — فندق وإدارة الغرف

تخيل فندق كبير:

| المفهوم الفعلي | التشبيه |
|---|---|
| الـ SoC pins | مفاتيح الكهرباء والإنترنت والتليفون في الغرفة |
| الـ pinctrl driver | إدارة الفندق (Front Desk) |
| الـ device driver | النزيل |
| الـ pinctrl subsystem | نظام الحجز المركزي |
| `struct dev_pin_info` | ورقة الـ check-in الخاصة بكل نزيل |
| `pinctrl_state` | "package" (غرفة عادية / suite / غرفة meeting) |
| `default` state | الغرفة وقت الإقامة العادية |
| `sleep` state | الغرفة وقت الـ do-not-disturb (بتوفر طاقة) |
| `init` state | الغرفة قبل ما النزيل ييجي (التجهيز) |
| `idle` state | الغرفة وقت ما النزيل بره وبيرجع قريب |

الـ Front Desk (pinctrl subsystem) هو اللي بيعرف مين في أي غرفة، ومين ممنوع يدخل غرفة تانية. النزيل (device driver) مش محتاج يعرف كهرباء الفندق شغالة إزاي — بس يقول "أنا عايز أقيم" وإدارة الفندق ترتب له كل حاجة.

---

### الـ Big Picture Architecture

```
+------------------------------------------------------------------+
|                        Linux Kernel                              |
|                                                                  |
|   +----------------+    +----------------+    +--------------+  |
|   |  Device Driver  |    |  Device Driver  |    | GPIO Driver  |  |
|   |   (e.g. UART)   |    |   (e.g. SPI)    |    | (gpiolib)    |  |
|   +-------+--------+    +-------+--------+    +------+-------+  |
|           |                     |                     |          |
|           | pinctrl_get()       | pinctrl_get()       | pinctrl_ |
|           | pinctrl_select_     | pinctrl_select_     | gpio_*() |
|           | state()             | state()             |          |
|           v                     v                     v          |
|   +----------------------------------------------------------+   |
|   |              pinctrl CORE (drivers/pinctrl/core.c)       |   |
|   |                                                          |   |
|   |  - pin ownership tracking                               |   |
|   |  - state machine (default/init/sleep/idle)              |   |
|   |  - mux conflict detection                               |   |
|   |  - integration with struct device (dev_pin_info)        |   |
|   +---------------------------+------------------------------+   |
|                               |                                  |
|          pinctrl_ops / pinmux_ops / pinconf_ops                  |
|                               |                                  |
|          +--------------------+--------------------+             |
|          |                                         |             |
|   +------+----------+                   +----------+------+      |
|   | SoC pinctrl drv |                   | SoC pinctrl drv |      |
|   | (e.g. IMX8 MXC) |                   | (e.g. BCM2835)  |      |
|   +------+----------+                   +----------+------+      |
|          |                                         |             |
+----------|------------------------------------------|-----------+
           |                                         |
    +------+------+                          +-------+-----+
    |  SoC Pins   |                          |  SoC Pins   |
    | (hardware)  |                          | (hardware)  |
    +-------------+                          +-------------+
```

---

### الـ Core Abstraction

**الفكرة المحورية** في الـ pinctrl subsystem هي الـ **named state machine**.

بدل ما تتعامل مع الـ pins كـ registers وبتات، الـ subsystem بيعرف abstraction:

```
device --has--> pinctrl handle (struct pinctrl)
                    |
                    +--has-many--> pinctrl states (struct pinctrl_state)
                                        |
                                        +-- "default" state
                                        |       |
                                        |       +-- pin group A: mux=UART, pull=none
                                        |       +-- pin group B: mux=GPIO, pull-up
                                        |
                                        +-- "sleep" state
                                        |       |
                                        |       +-- pin group A: mux=GPIO, pull-down (save power)
                                        |
                                        +-- "init" state
                                                |
                                                +-- pin group A: high-Z (avoid glitch)
```

الـ device driver بيقول "انقل الجهاز لـ state اسمها `sleep`" — والـ pinctrl core بيعمل كل التفاصيل.

---

### `struct dev_pin_info` — الـ Integration بين الـ Device Model والـ Pinctrl

ده القلب اللي الملف `devinfo.h` بيعرفه:

```c
struct dev_pin_info {
    struct pinctrl *p;                  /* handle شامل للـ device كله */
    struct pinctrl_state *default_state; /* state الشغل العادي */
    struct pinctrl_state *init_state;    /* state قبل الـ probe */
#ifdef CONFIG_PM
    struct pinctrl_state *sleep_state;   /* state وقت الـ system suspend */
    struct pinctrl_state *idle_state;    /* state وقت الـ runtime suspend */
#endif
};
```

**وين بتتحط؟** جوه `struct device` نفسه:

```c
/* include/linux/device.h */
struct device {
    ...
#ifdef CONFIG_PINCTRL
    struct dev_pin_info *pins;   /* ← هنا */
#endif
    ...
};
```

يعني كل `struct device` في الـ kernel — سواء كان I2C device، SPI device، platform device — ممكن يكون ليه pin state machine خاصة بيه، من غير ما الـ driver يحتاج يعمل أي حاجة إضافية لو الـ Device Tree مضبوط صح.

---

### تدفق الحياة الكاملة لـ Device مع Pinctrl

```
Device Tree / ACPI
    |
    | pinctrl-0 = <&uart_pins_default>
    | pinctrl-names = "default", "sleep"
    |
    v
+------------------+
| really_probe()   |  (drivers/base/dd.c)
| in device core   |
+--------+---------+
         |
         | pinctrl_bind_pins(dev)
         |   - pinctrl_get(dev)        → يجيب الـ handle
         |   - lookup "init" state     → لو موجود، يطبقه الأول
         |   - lookup "default" state  → يحفظه في dev->pins->default_state
         |   - lookup "sleep" state    → يحفظه في dev->pins->sleep_state
         |   - lookup "idle" state     → يحفظه في dev->pins->idle_state
         v
+------------------+
| driver->probe()  |  ← لما الـ driver بدأ، الـ pins في "init" أو "default"
+--------+---------+
         |
         | pinctrl_init_done(dev)
         |   - لو الـ pins لسه في "init"، ينقلهم لـ "default" تلقائياً
         v
+------------------+
| Device Running   |  ← "default" state نشط
+--------+---------+
         |
    [System suspend]
         |
         | pinctrl_pm_select_sleep_state(dev)
         v
+------------------+
| Sleep State      |  ← "sleep" state نشط (power saving)
+------------------+
```

---

### الفرق بين الـ States الأربعة — جدول مقارنة

| State | متى بيتطبق | الهدف |
|---|---|---|
| `init` | قبل `probe()` | تجنب glitches على الـ pins قبل ما الـ driver يتحكم |
| `default` | بعد `probe()` / بعد resume | الـ pins جاهزة للشغل الكامل |
| `idle` | `runtime_suspend()` | نص نوم — توفير طاقة مع إمكانية الرجوع سريع |
| `sleep` | `system suspend()` | نوم كامل — أقصى توفير للطاقة |

---

### اللي الـ Pinctrl Subsystem بيمتلكه مقابل اللي بيفوضه للـ Drivers

| المسؤولية | المالك |
|---|---|
| تعريف الـ states ("default"، "sleep"...) | pinctrl core |
| تتبع مين ماسك أنهي pin | pinctrl core |
| كشف تضارب الـ mux بين devices | pinctrl core |
| Integration مع `struct device` و Power Management | pinctrl core |
| الـ API اللي drivers بتستخدمه | pinctrl core (consumer.h) |
| كتابة الـ registers الفعلية على الـ SoC | SoC-specific pinctrl driver |
| معرفة pin groups وأسمائها | SoC-specific pinctrl driver |
| تحديد الـ functions المتاحة لكل group | SoC-specific pinctrl driver |
| الـ electrical config (pull، drive strength...) | SoC-specific pinctrl driver |

---

### الـ `pinctrl_init_done()` — لماذا موجودة؟

```c
extern int pinctrl_init_done(struct device *dev);
```

بعض الـ devices بيتضرروا لو الـ pins اتحولت لـ "default" قبل ما الـ driver يكون جاهز (على سبيل المثال: UART بيبدأ يستقبل بيانات قبل ما يكون مهيأ). عشان كده الـ device core بيضع الـ pins في `init` state الأول، وبعد ما الـ `probe()` ينتهي بنجاح بيستدعي `pinctrl_init_done()` اللي بيشوف:

- لو فيه `init_state` وفيه `default_state` → ينقل الـ pins من `init` لـ `default`
- لو مفيش `init_state` → مش بيعمل حاجة (الـ pins بالفعل في `default`)

ده بيضمن انتقال آمن وبدون glitches.

---

### الـ `dev_pinctrl()` — Helper Function

```c
static inline struct pinctrl *dev_pinctrl(struct device *dev)
{
    if (!dev->pins)  /* لو الـ device مش عنده pin info خالص */
        return NULL;
    return dev->pins->p;  /* يرجع الـ handle */
}
```

دي convenience function بسيطة — بدل ما تكتب `dev->pins->p` في كل مكان، بتستخدم `dev_pinctrl(dev)`. مهمتها الأساسية إنها بتتعامل مع الحالة اللي `dev->pins` يكون `NULL` (يعني الـ device مش محتاج pinctrl).

---

### Stub Pattern — الـ `#ifdef CONFIG_PINCTRL`

الملف بيستخدم pattern شائع جداً في الـ kernel:

```c
#ifdef CONFIG_PINCTRL
/* الـ implementation الحقيقي */
extern int pinctrl_init_done(struct device *dev);
static inline struct pinctrl *dev_pinctrl(struct device *dev) { ... }
#else
/* Stubs فارغة لو الـ pinctrl مش موجود في الـ config */
static inline int pinctrl_init_done(struct device *dev) { return 0; }
static inline struct pinctrl *dev_pinctrl(struct device *dev) { return NULL; }
#endif
```

**الهدف**: drivers تقدر تستخدم الـ pinctrl API من غير ما تحتاج تكتب `#ifdef` بنفسها — الـ header هو اللي بيعمل الـ abstraction. لو الـ kernel اتبنى من غير `CONFIG_PINCTRL`، كل الـ calls بتروح لـ stubs وبترجع قيم آمنة.

---

### ملاحظة على الـ Subsystems المرتبطة

- **GPIO subsystem (gpiolib)**: قبل ما driver يطلب GPIO، الـ pinctrl بيتأكد إن الـ pin ده مش محجوز لـ function تانية — ده بيحصل عن طريق `pinctrl_gpio_request()`.
- **Power Management (PM)**: الـ `CONFIG_PM` guard جوه `dev_pin_info` بيضيف `sleep_state` و `idle_state` بس لما الـ kernel بيدعم power management — ده بيوفر memory على الأنظمة اللي مش محتاجة ده.
- **Device Tree**: الـ pin states بتتعرف في Device Tree عن طريق `pinctrl-0`, `pinctrl-1`, ... و `pinctrl-names` — الـ pinctrl core بيقرأهم وقت الـ `pinctrl_bind_pins()`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Cheatsheet — الـ Config Options والـ State Strings

#### Config Options

| Option | الأثر |
|---|---|
| `CONFIG_PINCTRL` | لو مفعّل، الـ `dev_pin_info` بتتضاف للـ `device` والـ API الحقيقية بتتفعّل |
| `CONFIG_PM` | لو مفعّل، الـ `sleep_state` والـ `idle_state` بيتضافوا للـ `dev_pin_info` |

#### Standard Pin State Strings (من `pinctrl-state.h`)

| Macro | String Value | الاستخدام |
|---|---|---|
| `PINCTRL_STATE_DEFAULT` | `"default"` | الـ state الطبيعي وقت الشغل |
| `PINCTRL_STATE_INIT` | `"init"` | قبل الـ `probe()` عشان تتجنب glitch في الـ pins |
| `PINCTRL_STATE_IDLE` | `"idle"` | runtime suspend — الـ system شغال بس مش busy |
| `PINCTRL_STATE_SLEEP` | `"sleep"` | full system suspend — أعمق حالة نوم |

---

### 1. الـ Structs المهمة

#### `struct dev_pin_info`

**الغرض:** Container صغير بيتحط داخل كل `struct device` عشان يربط الـ device بالـ pinctrl handle بتاعها وبكل الـ states المعروفة.

| Field | النوع | الوصف |
|---|---|---|
| `p` | `struct pinctrl *` | الـ handle الأساسي اللي بييجي من `pinctrl_get()` أو `devm_pinctrl_get()` — هو الـ "تذكرة" اللي بنتكلم بيها مع الـ pinctrl core |
| `default_state` | `struct pinctrl_state *` | الـ state اللي بيتفعّل وقت الشغل الطبيعي |
| `init_state` | `struct pinctrl_state *` | الـ state اللي بيتفعّل قبل الـ `probe()` لو الـ driver محتاج يتحكم في الـ pins قبل التهيئة |
| `sleep_state` | `struct pinctrl_state *` | (يتضاف بس لو `CONFIG_PM`) الـ state وقت الـ full suspend |
| `idle_state` | `struct pinctrl_state *` | (يتضاف بس لو `CONFIG_PM`) الـ state وقت الـ runtime idle |

**علاقته بالـ structs التانية:**
- **الـ `struct device`** بيحتوي على pointer اسمه `pins` من نوع `struct dev_pin_info *` — يعني كل device في النظام ممكن يبقى عنده pinctrl info.
- كل field من الـ states بيبص على `struct pinctrl_state` اللي هي opaque للـ consumer وبيديرها الـ pinctrl core.
- الـ `struct pinctrl *p` هو الـ handle اللي بيربط الـ device بالـ pinctrl core بالكامل.

---

#### `struct pinctrl` (مُعرّفة في الـ core — مُعلنة هنا بس)

مش معرّفة في الملف ده، بس مهمة نعرفها. هي الـ cookie الـ private اللي:
- بتتعمل في `pinctrl_get(dev)`
- بتحتوي داخليًا على list من الـ `pinctrl_state` objects
- مرتبطة بـ `struct pinctrl_dev` اللي بيمثل الـ controller الفيزيائي

---

#### `struct pinctrl_state` (مُعرّفة في الـ core — مُعلنة هنا بس)

كمان opaque للـ consumer. بتحتوي:
- اسم الـ state (string زي `"default"`)
- list من الـ `pinctrl_setting` اللي بتحدد الـ mux وإعدادات كل pin في الـ state ده

---

### 2. مخطط علاقات الـ Structs (ASCII)

```
struct device
┌─────────────────────────────────────┐
│  ...                                │
│  struct dev_pin_info  *pins ────────┼──────────────────────────┐
│  ...                                │                          │
└─────────────────────────────────────┘                          │
                                                                 ▼
                                              struct dev_pin_info
                                        ┌──────────────────────────────────┐
                                        │  struct pinctrl        *p        │──────┐
                                        │  struct pinctrl_state  *default_state │──┐│
                                        │  struct pinctrl_state  *init_state    │  ││
                                        │  struct pinctrl_state  *sleep_state   │  ││  (CONFIG_PM)
                                        │  struct pinctrl_state  *idle_state    │  ││  (CONFIG_PM)
                                        └──────────────────────────────────┘  ││
                                                                               ││
                        ┌──────────────────────────────────────────────────────┘│
                        │                                                        │
                        ▼                                                        │
              struct pinctrl                                                     │
        ┌─────────────────────────┐                                              │
        │  struct device   *dev   │                                              │
        │  list_head  states ─────┼──┐                                           │
        │  ...                    │  │                                           │
        └─────────────────────────┘  │                                           │
                                     │                                           │
                    ┌────────────────┘     ┌────────────────────────────────────┘
                    │                      │
                    ▼                      ▼
          struct pinctrl_state     struct pinctrl_state
        ┌──────────────────────┐  ┌──────────────────────┐
        │  char  *name         │  │  char  *name         │
        │  list settings       │  │  list settings       │
        └──────────────────────┘  └──────────────────────┘
```

---

### 3. مخطط الـ Lifecycle

```
══════════════════════════════════════════════════════════════════
                    LIFECYCLE: dev_pin_info
══════════════════════════════════════════════════════════════════

[1] CREATION — عند تسجيل الـ device في device core
    ─────────────────────────────────────────────
    really_probe(dev)
      └─► pinctrl_bind_pins(dev)
            ├─► devm_pinctrl_get(dev)
            │     └─► pinctrl_get()  ──creates──► struct pinctrl *p
            │
            ├─► pinctrl_lookup_state(p, "init")    ──► init_state   (أو NULL)
            ├─► pinctrl_lookup_state(p, "default") ──► default_state
            ├─► pinctrl_lookup_state(p, "sleep")   ──► sleep_state  (CONFIG_PM)
            ├─► pinctrl_lookup_state(p, "idle")    ──► idle_state   (CONFIG_PM)
            │
            └─► كل ده بيتحط في dev->pins (struct dev_pin_info)


[2] INIT STATE — قبل الـ probe()
    ─────────────────────────────
    لو init_state موجود:
      pinctrl_select_state(p, init_state)
        └─► الـ pins بتتضبط للـ init config


[3] PROBE — أثناء driver probe()
    ──────────────────────────────
    driver probe() بيشتغل...
    بعد الـ probe:
      pinctrl_init_done(dev)
        └─► لو الـ pins لسّه على init_state:
              pinctrl_select_state(p, default_state)
                └─► الانتقال للـ default state


[4] RUNTIME NORMAL — الشغل الطبيعي
    ──────────────────────────────────
    الـ pins على default_state
    dev_pinctrl(dev) ──► بيرجع dev->pins->p للـ driver لو محتاج


[5] POWER MANAGEMENT — أثناء الـ suspend/resume
    ──────────────────────────────────────────────
    pm_runtime_suspend()  ──► pinctrl_pm_select_idle_state(dev)
                                └─► select idle_state
    suspend()             ──► pinctrl_pm_select_sleep_state(dev)
                                └─► select sleep_state
    resume() / pm_runtime_resume() ──► pinctrl_pm_select_default_state(dev)
                                         └─► رجوع للـ default_state


[6] TEARDOWN — عند إزالة الـ device
    ──────────────────────────────────
    device_release_driver(dev)
      └─► devm cleanup يعمل devm_pinctrl_put(p)
            └─► pinctrl_put(p)
                  └─► الـ struct pinctrl اتحرر + dev->pins = NULL
══════════════════════════════════════════════════════════════════
```

---

### 4. مخطط الـ Call Flow

#### 4.1 تهيئة الـ device (bind pins)

```
device_add(dev)
  └─► really_probe(dev)
        └─► pinctrl_bind_pins(dev)   [drivers/base/pinctrl.c]
              │
              ├─► kzalloc(dev_pin_info)        // حجز الـ container
              ├─► devm_pinctrl_get(dev)
              │     └─► pinctrl_get(dev)
              │           └─► pinctrl core: يدور على الـ pinctrl_dev المناسب
              │                 └─► create struct pinctrl + ربطه بالـ dev
              │
              ├─► pinctrl_lookup_state(p, PINCTRL_STATE_INIT)
              │     └─► بيدور في p->states list على state اسمه "init"
              │
              ├─► pinctrl_lookup_state(p, PINCTRL_STATE_DEFAULT)
              │
              ├─► [CONFIG_PM] pinctrl_lookup_state(p, PINCTRL_STATE_SLEEP)
              ├─► [CONFIG_PM] pinctrl_lookup_state(p, PINCTRL_STATE_IDLE)
              │
              ├─► dev->pins = الـ dev_pin_info الجديدة
              │
              └─► لو init_state موجود:
                    pinctrl_select_state(p, init_state)
                      └─► pinctrl core: يطبّق كل الـ settings في الـ state
                            └─► ops->set_mux() / ops->pin_config_set()
                                  └─► hardware registers
```

#### 4.2 انتهاء الـ probe

```
driver->probe(dev) returns 0
  └─► pinctrl_init_done(dev)   [drivers/base/pinctrl.c]
        │
        ├─► لو dev->pins->init_state == NULL  ──► return 0 (مفيش حاجة)
        │
        ├─► لو current state != init_state  ──► return 0 (الـ driver غيّر بنفسه)
        │
        └─► pinctrl_select_state(p, default_state)
              └─► تطبيق الـ default config على الـ hardware
```

#### 4.3 الـ dev_pinctrl() inline

```
caller calls dev_pinctrl(dev)
  │
  ├─► dev->pins == NULL ?  ──► return NULL
  │
  └─► return dev->pins->p   // الـ pinctrl handle
```

---

### 5. استراتيجية الـ Locking

الملف ده بنفسه **ما فيهوش أي lock** — لأنه مجرد header بيعرّف struct وinline functions. لكن فهم الـ locking مهم لأي حد بيتعامل مع الـ dev_pin_info:

| المورد | الـ Lock اللي بيحميه | ملاحظات |
|---|---|---|
| `dev->pins` (الـ pointer نفسه) | `device_lock(dev)` | بيتحط مرة واحدة في `pinctrl_bind_pins()` وما بيتغيرش |
| محتوى `struct pinctrl` | `pinctrl_maps_mutex` (في الـ core) | بيتحكم في الـ maps والـ states lists |
| تغيير الـ active state | لا يوجد lock صريح في الـ API | الـ caller مسؤول إنه يضمن serialization |
| الـ PM transitions | `dev->power.lock` (pm_runtime lock) | بيحمي الـ state أثناء الـ suspend/resume |

**ترتيب الـ Locks (لو احتجت أكتر من lock):**
```
device_lock(dev)
  └─► (داخل الـ pinctrl core) pinctrl_maps_mutex
```
دايمًا `device_lock` الأول عشان تتجنب deadlock.

**ملاحظة مهمة:** الـ `devm_pinctrl_get()` بيربط التحرير التلقائي بـ `struct device` — يعني في الـ teardown مش محتاج تتذكر تعمل `pinctrl_put()` يدوي.
## Phase 4: شرح الـ Functions

### ملخص سريع — Cheatsheet

| Function / Macro | النوع | الغرض |
|---|---|---|
| `dev_pinctrl(dev)` | `static inline` | بترجع الـ `pinctrl` handle الخاص بالـ device |
| `pinctrl_init_done(dev)` | `extern` | بتخبر الـ pinctrl core إن الـ probe انتهى وممكن نطبق الـ default state |

---

### التصنيفات

```
┌─────────────────────────────────────────┐
│          devinfo.h Functions            │
├──────────────────┬──────────────────────┤
│  Accessor        │  dev_pinctrl()       │
├──────────────────┼──────────────────────┤
│  Lifecycle       │  pinctrl_init_done() │
└──────────────────┴──────────────────────┘
```

---

### الـ struct المركزي: `dev_pin_info`

```c
struct dev_pin_info {
    struct pinctrl       *p;             /* handle رئيسي للـ pinctrl */
    struct pinctrl_state *default_state; /* الـ "default" state */
    struct pinctrl_state *init_state;    /* الـ "init" state عند الـ probe */
#ifdef CONFIG_PM
    struct pinctrl_state *sleep_state;   /* الـ "sleep" state عند الـ suspend */
    struct pinctrl_state *idle_state;    /* الـ "idle" state عند الـ runtime suspend */
#endif
};
```

الـ `dev_pin_info` بتتخزن جوه `struct device` تحت الحقل `dev->pins`.
الـ device core نفسه بيكون هو الـ **consumer** اللي بيطلب الـ states دي من الـ pinctrl subsystem.
الـ struct ده بيعمل abstraction للـ pin states الأربعة الأساسية: default, init, sleep, idle.

---

### Category 1: Accessor

#### `dev_pinctrl`

```c
static inline struct pinctrl *dev_pinctrl(struct device *dev)
{
    if (!dev->pins)
        return NULL;
    return dev->pins->p;
}
```

**الـ function بتعمل إيه:**
بترجع الـ `pinctrl` handle المرتبط بالـ device، وده هو الـ opaque cookie اللي بيتعامل بيه أي consumer مع الـ pinctrl core.
لو الـ device مش عنده `pins` (يعني CONFIG_PINCTRL مش موجود أو الـ device مش معاه pin configuration في الـ DT)، بترجع `NULL` بأمان من غير ما تكسر حاجة.

**الـ Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ device المراد الاستعلام عنه |

**الـ Return Value:**
- `struct pinctrl *` — الـ handle لو موجود.
- `NULL` — لو `dev->pins == NULL` (الـ device مش عنده pin info).

**Key Details:**
- الـ function دي `static inline` فمفيش call overhead.
- مفيش locking — الـ caller المفروض يضمن إن `dev->pins` مش بيتعمله free في نفس الوقت.
- الـ stub الموجودة لو `!CONFIG_PINCTRL` بترجع `NULL` دايمًا.

**من بيستدعيها:**
الـ device core وأي driver بيحتاج يتحقق إن الـ pinctrl handle موجود قبل ما يعمل عليه operations.

---

### Category 2: Lifecycle

#### `pinctrl_init_done`

```c
extern int pinctrl_init_done(struct device *dev);
/* stub when !CONFIG_PINCTRL */
static inline int pinctrl_init_done(struct device *dev)
{
    return 0;
}
```

**الـ function بتعمل إيه:**
بتُعلم الـ pinctrl core إن الـ device أتمت الـ probe وإن الـ `init` state لو كانت متوفرة لازم يتم التبديل منها للـ `default` state.
الـ design pattern هنا: الـ "init" state بتُستخدم أثناء الـ probe بس (مثلًا لتهيئة power sequencing معينة)، وبعد ما الـ probe خلص نرجع للـ "default" state الطبيعية.

**الـ Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `dev` | `struct device *` | الـ device اللي خلص الـ probe بتاعته |

**الـ Return Value:**
- `0` — تمام، أو مفيش `init_state` أصلًا.
- قيمة سالبة — error من `pinctrl_select_state` لو التبديل للـ default state فشل.

**Key Details:**
- الـ function دي بتتاكل من `drivers/pinctrl/devicetree.c` أو `core.c` — الـ implementation مش في الـ header.
- لو `!CONFIG_PINCTRL`، الـ stub بترجع `0` بدون أي عمل.
- **Error path:** لو رجعت قيمة سالبة، الـ device core ممكن يفشل الـ probe بالكامل — الـ caller لازم يتعامل مع الـ error بجدية.
- **Caller context:** بيُستدعى من `really_probe()` في `drivers/base/dd.c` بعد ما الـ driver's `.probe()` callback ينجح.

**Pseudocode Flow:**

```
pinctrl_init_done(dev):
    pins = dev->pins
    if !pins || !pins->init_state:
        return 0                        // no init state, nothing to do

    if pins->p current state == init_state:
        select default_state            // switch away from init
        if error:
            return error
    return 0
```

---

### ملاحظات مهمة على الـ Dual-Implementation Pattern

الـ header بيستخدم نمط `#ifdef CONFIG_PINCTRL` لكل function:

```c
#ifdef CONFIG_PINCTRL
    extern int pinctrl_init_done(struct device *dev);
    static inline struct pinctrl *dev_pinctrl(struct device *dev) { ... real impl ... }
#else
    /* Stubs */
    static inline int pinctrl_init_done(struct device *dev) { return 0; }
    static inline struct pinctrl *dev_pinctrl(struct device *dev) { return NULL; }
#endif
```

الـ pattern ده بيضمن إن الـ device core يشتغل بدون أي تعديل حتى لو الـ pinctrl subsystem مش موجود في الـ kernel config — كل الـ calls بتبقى no-ops أو stub returns.

---

### الـ States الأربعة — دورة حياة الـ Device

```
Device Lifecycle:
─────────────────────────────────────────────────────────►
   probe starts       probe done       suspend       resume
       │                  │               │             │
       ▼                  ▼               ▼             ▼
  [init_state]    [default_state]   [sleep_state]  [default_state]
  (optional)      (pinctrl_init_done)  (CONFIG_PM)   (CONFIG_PM)
                                    or [idle_state]
                                    (runtime suspend)
```

الـ `dev_pin_info` بتحتفظ بالـ pointers للـ states دي، والـ device core بيتنقل بينهم تلقائيًا خلال دورة حياة الـ device.
## Phase 5: دليل الـ Debugging الشامل

الـ `pinctrl/devinfo.h` بيعرّف `struct dev_pin_info` اللي بتربط كل device بـ pinctrl handle بتاعها وبـ states مختلفة (default، init، sleep، idle). الـ debugging هنا بيتمحور حول: هل الـ device اتربطت بصح بالـ pinctrl subsystem؟ وهل الـ state switching بيحصل صح في كل مرحلة (probe، suspend، resume)؟

---

### Software Level

#### 1. debugfs Entries

الـ pinctrl subsystem بيعمل tree كامل جوه debugfs:

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# اطبع كل الـ pinctrl controllers المتاحة
ls /sys/kernel/debug/pinctrl/

# مثال: controller اسمه "soc-pinctrl"
ls /sys/kernel/debug/pinctrl/soc-pinctrl/
# هتلاقي: pingroups  pinmux-functions  pinmux-pins  pinconf-pins  gpio-ranges
```

| الملف | ما بيعرضه | ازاي تفسره |
|-------|-----------|------------|
| `pinmux-pins` | كل pin وهو معمول mux لأنهو | لو pin مش معمول mux → الـ device مش شغالة |
| `pinmux-functions` | الـ functions المتاحة لكل pin group | تأكد الـ function اللي الـ DT بيطلبه موجود |
| `pingroups` | الـ groups وأعضاءها | تحقق من الـ group membership |
| `pinconf-pins` | الـ pull-up/down، drive strength | قارن بالـ DT |
| `gpio-ranges` | mapping بين GPIO numbers وpin numbers | مهم لـ GPIO/pinctrl conflict |

```bash
# شوف إيه الـ state الحالية لكل device
cat /sys/kernel/debug/pinctrl/soc-pinctrl/pinmux-pins

# مثال output:
# pin 42 (PA10): device soc:uart0, function uart0, group uart0_grp
# pin 43 (PA11): device soc:uart0, function uart0, group uart0_grp
```

لو شفت pin مش معموله مux لأي device → يعني `pinctrl_select_state()` فشلت أو `dev_pin_info` مش initialized صح.

#### 2. sysfs Entries

الـ sysfs مش بيعرض pinctrl details بشكل مباشر، لكن ممكن تتحقق من الـ device state:

```bash
# شوف الـ power state للـ device (بيأثر على pin state selection)
cat /sys/bus/platform/devices/soc:uart0/power/runtime_status
# output: active | suspended | error

# تأكد الـ pinctrl handle موجود (لو الـ device بتستخدم devm_pinctrl_get)
ls /sys/bus/platform/devices/soc:uart0/

# الـ driver binding
cat /sys/bus/platform/devices/soc:uart0/driver_override
```

#### 3. ftrace — Tracepoints وEvents

```bash
# فعّل الـ pinctrl tracepoints
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/enable

# أو فعّل events محددة
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_state_init/enable
echo 1 > /sys/kernel/debug/tracing/events/pinctrl/pinctrl_state_select/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# probe الـ driver أو trigger الـ suspend/resume
# مثلاً: modprobe spi-driver

# اقرأ الـ trace
cat /sys/kernel/debug/tracing/trace

# مثال output:
# kworker/0:2-45    [000] ....  123.456: pinctrl_state_select: p=soc:spi0 state=default
```

لو مش شايف `pinctrl_state_select` event → الـ `pinctrl_select_state()` ما اتكلمتش أصلاً.

لتتبع الـ `dev_pin_info` initialization بالذات، استخدم function tracer:

```bash
echo function > /sys/kernel/debug/tracing/current_tracer
echo pinctrl_init_done > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ثم trigger الـ driver probe
cat /sys/kernel/debug/tracing/trace
```

#### 4. printk / Dynamic Debug

```bash
# فعّل dynamic debug للـ pinctrl subsystem كله
echo "file drivers/pinctrl/*.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file include/linux/pinctrl/*.h +p" > /sys/kernel/debug/dynamic_debug/control

# أو لـ file معين (الـ core اللي بيعمل dev_pin_info)
echo "file drivers/pinctrl/core.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
# p = printk, f = function name, l = line number, m = module, t = timestamp

# شوف الـ messages
dmesg | grep -i pinctrl

# لو عايز تفعّل من kernel cmdline (في bootloader)
# أضف: dyndbg="file drivers/pinctrl/core.c +p"
```

#### 5. Kernel Config Options للـ Debugging

```
CONFIG_PINCTRL=y                    # الـ subsystem الأساسي — لازم يكون y
CONFIG_DEBUG_PINCTRL=y              # verbose logging للـ pinctrl core
CONFIG_PINCTRL_SINGLE=y             # لـ simple pinctrl drivers
CONFIG_PM=y                         # لازم لـ sleep_state/idle_state في dev_pin_info
CONFIG_PM_DEBUG=y                   # debug messages للـ power management
CONFIG_PM_SLEEP_DEBUG=y             # debug الـ suspend/resume state transitions
CONFIG_DEBUG_DEVRES=y               # يتتبع devm_pinctrl_get allocations
CONFIG_PROVE_LOCKING=y              # يكشف locking issues في pinctrl
CONFIG_DEBUG_ATOMIC_SLEEP=y         # يكشف لو pinctrl_select_state اتكلمت في atomic context
CONFIG_FTRACE=y                     # لازم للـ function tracing
CONFIG_TRACING=y                    # لازم للـ event tracing
```

#### 6. أدوات خاصة بالـ Subsystem

**أداة `pinctrl` في userspace (لو متاحة):**

```bash
# بعض الـ distros فيها pinctrl tool
pinctrl list          # اطبع كل الـ pins
pinctrl get 42        # اطبع config الـ pin رقم 42
```

**استخدام sysfs GPIO لتأكيد الـ pin ownership:**

```bash
# تحقق من أن الـ pin مش متأخوذ من GPIO subsystem غلط
cat /sys/kernel/debug/gpio
# لو pin ظاهر هنا وفي pinmux في نفس الوقت → conflict
```

**الـ `pinctrl_init_done()` function:**

الـ function دي بتتكلم من device core بعد ما الـ driver probe يخلص، لو رجعت error → الـ device مش هتشتغل. تقدر تضيف `WARN_ON` قبلها:

```c
/* في drivers/base/dd.c أو driver نفسه */
ret = pinctrl_init_done(dev);
WARN_ON(ret);  /* يطبع stack trace لو فشلت */
```

#### 7. جدول الـ Error Messages الشائعة

| رسالة الـ Kernel | المعنى | الحل |
|-----------------|--------|-------|
| `pinctrl_get() failed for device soc:uart0` | مش لاقي pinctrl handle | تأكد `CONFIG_PINCTRL=y` و DT صح |
| `could not get default pinstate` | مش لاقي state اسمها "default" في DT | أضف `pinctrl-0` و `pinctrl-names = "default"` في DT |
| `could not get init pinstate` | مش لاقي state "init" | إما طبيعي أو ناقص في DT |
| `could not get sleep pinstate` | مش لاقي state "sleep" | أضف `pinctrl-names = "default", "sleep"` |
| `pin XX already requested` | تعارض — pin اتطلب من device تاني | شوف `pinmux-pins` في debugfs وحل الـ overlap |
| `pin-controller soc-pinctrl status -EPROBE_DEFER` | الـ pinctrl controller لسه مش probe | الـ device هتعمل probe تاني، انتظر أو تأكد الـ controller probe أول |
| `ops for pin XX missing in driver` | الـ pinctrl driver مش بيدعم العملية دي | راجع الـ `pinctrl_ops` في الـ driver |
| `pinctrl_select_state() failed` | فشل في switch الـ state | تحقق من الـ pin config validity والـ log |
| `could not get default pinstate for device` | `dev->pins` كان NULL وقت الطلب | الـ `dev_pin_info` مش initialized، راجع device core init order |

#### 8. Strategic Points لـ dump_stack() / WARN_ON()

الأماكن الاستراتيجية في `drivers/pinctrl/core.c` و `drivers/base/dd.c`:

```c
/* 1. في pinctrl_bind_pins() — أول ما بتتربط الـ device بالـ pinctrl */
int pinctrl_bind_pins(struct device *dev)
{
    dev->pins = devm_kzalloc(dev, sizeof(*(dev->pins)), GFP_KERNEL);
    WARN_ON(!dev->pins);  /* هنا لو ما اتعملتش alloc */

    dev->pins->p = devm_pinctrl_get(dev);
    if (IS_ERR(dev->pins->p)) {
        /* هنا debug: إيه الـ error بالظبط */
        dev_err(dev, "pinctrl_get failed: %ld\n", PTR_ERR(dev->pins->p));
        dump_stack();  /* مين استدى؟ */
    }
}

/* 2. في pinctrl_select_state() — كل ما بيتغير الـ state */
int pinctrl_select_state(struct pinctrl *p, struct pinctrl_state *state)
{
    WARN_ON(in_atomic());  /* ميتكلمش من atomic context */
    /* ... */
}

/* 3. في pinctrl_init_done() — بعد الـ probe */
int pinctrl_init_done(struct device *dev)
{
    if (!dev->pins)
        return 0;
    if (dev->pins->init_state == dev->pins->p->state) {
        /* لسه في init state بعد probe — هنا WARN */
        WARN(1, "device %s still in init state after probe\n",
             dev_name(dev));
    }
}
```

---

### Hardware Level

#### 1. التحقق من أن الـ Hardware State بيطابق الـ Kernel State

المبدأ: الـ `dev_pin_info` بتخزن الـ state اللي الـ kernel فاكره، لكن لازم تتأكد الـ hardware فعلاً كده:

```bash
# الخطوة 1: اعرف الـ state اللي الـ kernel شايفه
cat /sys/kernel/debug/pinctrl/soc-pinctrl/pinmux-pins
# مثال: pin 10 (PB2): device soc:i2c1, function i2c1, group i2c1_grp

# الخطوة 2: اعرف الـ register value المتوقع لـ mux هذا
# من الـ datasheet: I2C1 function على PB2 = مثلاً FUNC_SEL bits = 0b010

# الخطوة 3: قرأ الـ register الفعلي بـ devmem2
devmem2 0x01C20858 w   # عنوان register الـ PB مثلاً
```

#### 2. Register Dump Techniques

```bash
# تثبيت devmem2 لو مش موجود
apt install devmem2
# أو
busybox devmem 0x ADDRESS w

# قراءة register واحد (32-bit)
devmem2 0x01C20800 w
# output: Value at address 0x01C20800 (0xb6f0d000): 0x77000000

# قراءة range كامل للـ pinctrl bank
for addr in $(seq 0x01C20800 4 0x01C20820); do
    printf "0x%08x: " $addr
    devmem2 $addr w 2>/dev/null | grep "Value" | awk '{print $NF}'
done

# باستخدام /dev/mem مباشرة (مع xxd)
dd if=/dev/mem bs=4 count=16 skip=$((0x01C20800/4)) 2>/dev/null | xxd

# باستخدام io utility (من package ioport)
io -4 -r 0x01C20800
```

**تفسير الـ output:**

```
Register PB_CFG0 @ 0x01C20800:
Bits [2:0]  = PB0 function select
Bits [6:4]  = PB1 function select
Bits [10:8] = PB2 function select  ← I2C1_SDA لازم = 010
Bits [14:12]= PB3 function select  ← I2C1_SCK لازم = 010
```

لو الـ kernel بيقول I2C1 على PB2/PB3 لكن الـ register بيقول function = 000 (GPIO) → الـ `pinctrl_select_state()` ما شتغلتش صح أو الـ hardware reset الـ register.

#### 3. Logic Analyzer / Oscilloscope Tips

**للـ I2C pins مثلاً:**

```
Logic Analyzer Setup:
─────────────────────────────────────────
Channel 0 → SDA pin
Channel 1 → SCL pin
Sample Rate → 4x أكتر من الـ I2C frequency
Trigger → Rising edge على SCL

المتوقع لو pinctrl صح:
  - Pull-up موجود (line بترجع HIGH)
  - الـ drive strength صح (edges مش slow جداً)
  - مفيش glitches وقت الـ state transition
```

**مشاكل شائعة وعلاماتها على الـ scope:**

```
مشكلة 1: line عالقة LOW
  → pin لسه مضبوط كـ GPIO output LOW
  → الـ pinctrl state selection فشلت

مشكلة 2: signal ضعيف / slow edges
  → drive strength مش مظبوط في pinctrl config
  → راجع: pinconf-pins في debugfs

مشكلة 3: noise على الـ line وقت suspend
  → الـ sleep_state مش معمولها config صح
  → لازم تبقى configured كـ input مع pull
```

**Oscilloscope عند الـ suspend/resume:**

```
Trigger على GPIO توثق لحظة الـ suspend:
  - لازم ترى transition من active signal → idle level
  - الوقت من start suspend لحد pin بيتغير = وقت pinctrl_pm_select_sleep_state()
  - لو الـ pin بقى floating → idle_state أو sleep_state مش معمولها
```

#### 4. Common Hardware Issues وـ Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log | التفسير |
|---------------------|--------------------------|---------|
| Pin مش شايل pull-up/pull-down | `i2c i2c-1: timeout waiting for bus ready` | الـ line مش بترجع HIGH — راجع pinconf |
| تعارض مع GPIO | `gpiod_request: pin X is already in use` | نفس الـ pin متطلوب من GPIO وpinctrl |
| Drive strength منخفض | `spi_transfer timed out` أو data corruption | الـ edge rate ضعيف — زود drive strength |
| Pin مش معمول mux | `uart-pl011: unable to read` أو no data | الـ RX/TX مش switched لـ UART function |
| Hardware reset الـ pinctrl registers | مشاكل بعد suspend/resume | الـ bootloader أو PMU بيعمل reset للـ bank |
| Open-drain مش مضبوط | `i2c: bus arbitration lost` continuously | SCL/SDA مش open-drain في الـ pinconf |

#### 5. Device Tree Debugging

```bash
# الخطوة 1: اطبع الـ DT المحمّل فعلاً (مش الـ source)
cat /sys/firmware/devicetree/base/soc/uart@1c28000/pinctrl-0
# لو فيه output → الـ DT صح

# الخطوة 2: تحقق من الـ pinctrl-names
cat /sys/firmware/devicetree/base/soc/uart@1c28000/pinctrl-names
# المتوقع: default\0  (null-terminated)

# الخطوة 3: fdtdump لمقارنة الـ DT source بالمحمّل
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -A 20 "uart@1c28000"

# الخطوة 4: تحقق من الـ phandle references
# الـ pinctrl-0 بيشير لـ phandle، تأكد الـ phandle موجود
grep -r "pinctrl-0" /sys/firmware/devicetree/base/ 2>/dev/null
```

**مثال DT صحيح مقابل خاطئ:**

```dts
/* صح: كل الـ states معرفة */
&uart0 {
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&uart0_default>;   /* للـ default_state */
    pinctrl-1 = <&uart0_sleep>;     /* للـ sleep_state */
};

/* خطأ شائع: ناقص sleep state والـ kernel بيطلبه */
&uart0 {
    pinctrl-names = "default";      /* بس default */
    pinctrl-0 = <&uart0_default>;
    /* لو CONFIG_PM=y والـ driver بيحاول pinctrl_pm_select_sleep_state() */
    /* هيطلع: could not get sleep pinstate */
};
```

```bash
# تحقق من الـ phandle target
# اعرف رقم الـ phandle من pinctrl-0
hexdump -C /sys/firmware/devicetree/base/soc/uart@1c28000/pinctrl-0
# ابحث عن الـ node اللي عنده الـ phandle ده
find /sys/firmware/devicetree/base -name "phandle" | xargs -I{} sh -c \
  'val=$(hexdump -e "1/4 \"%u\"" {}); echo "$val: $(dirname {})"' 2>/dev/null
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

**1. الفحص الأولي السريع:**

```bash
#!/bin/bash
# pinctrl-quick-debug.sh — فحص سريع لحالة الـ pinctrl

DEVICE="${1:-soc:uart0}"

echo "=== pinctrl Controllers ==="
ls /sys/kernel/debug/pinctrl/

echo -e "\n=== pinmux-pins ==="
for ctrl in /sys/kernel/debug/pinctrl/*/; do
    echo "--- $(basename $ctrl) ---"
    cat "$ctrl/pinmux-pins" | grep "$DEVICE" || echo "(no pins for $DEVICE)"
done

echo -e "\n=== Device Power State ==="
cat /sys/bus/platform/devices/$DEVICE/power/runtime_status 2>/dev/null || echo "N/A"

echo -e "\n=== Recent pinctrl dmesg ==="
dmesg | grep -i pinctrl | tail -20
```

**2. تفعيل الـ Tracing الكامل:**

```bash
# فعّل كل pinctrl events
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo "" > trace
echo "pinctrl:*" > set_event
echo 1 > tracing_on

# اعمل action (مثلاً: suspend/resume)
systemctl suspend &
sleep 5

# اقرأ
cat trace | grep -E "pinctrl|$DEVICE"
echo 0 > tracing_on
```

**3. قراءة وتفسير الـ pinconf:**

```bash
# اطبع الـ pin configuration كاملة
cat /sys/kernel/debug/pinctrl/soc-pinctrl/pinconf-pins

# مثال output مشروح:
# pin 10 (PB2):
#   drive strength: 10 mA       ← قوة التيار
#   bias: pull-up (100 KOhm)    ← pull-up مفعّل
#   input enable: enabled       ← ممكن يقرأ
#   output enable: disabled     ← مش بيكتب
```

**4. اختبار state switching يدوياً:**

```bash
# ده مثال لو بتكتب driver أو تعمل test module
# استخدم debugfs لإجبار state change (لو الـ driver بيدعمه):
echo "sleep" > /sys/kernel/debug/pinctrl/soc-pinctrl/pinmux-select
# (مش كل الـ drivers بيدعموا ده، لكن بعض SoCs بيحطوه)
```

**5. تحقق من أن `dev_pin_info` initialized بشكل صح:**

```bash
# الـ dev->pins بيتعمل في pinctrl_bind_pins() اللي بتتكلم من really_probe()
# لو فيه مشكلة هتشوفها هنا:
dmesg | grep -E "pinctrl|pins" | grep -E "error|failed|defer|EPROBE"

# مثال output مشكلة:
# [    2.345] platform soc:uart0: Failed to create pinctrl for device; -517
# -517 = -EPROBE_DEFER → الـ pinctrl controller لسه مش جاهز
```

**6. تحقق من الـ sleep/idle state transitions:**

```bash
# شغّل dynamic debug للـ PM pinctrl
echo "func pinctrl_pm_select_sleep_state +p" > /sys/kernel/debug/dynamic_debug/control
echo "func pinctrl_pm_select_idle_state +p" > /sys/kernel/debug/dynamic_debug/control
echo "func pinctrl_pm_select_default_state +p" > /sys/kernel/debug/dynamic_debug/control

# عمل suspend/resume
rtcwake -s 10 -m mem

# شوف اللي حصل
dmesg | grep -i "pinctrl_pm"
# المتوقع:
# [  120.000] pinctrl_pm_select_sleep_state: device soc:uart0 → sleep
# [  130.000] pinctrl_pm_select_default_state: device soc:uart0 → default
```

**7. Register dump script:**

```bash
#!/bin/bash
# dump-pinctrl-regs.sh BASE_ADDR COUNT
# مثال: ./dump-pinctrl-regs.sh 0x01C20800 16

BASE=$1
COUNT=${2:-8}

echo "=== Pin Control Register Dump ==="
echo "Base: $BASE, Count: $COUNT registers"
echo ""
printf "%-12s %-12s %s\n" "Address" "Value" "Bits[31:0]"
printf "%-12s %-12s %s\n" "-------" "-----" "---------"

for i in $(seq 0 $((COUNT-1))); do
    ADDR=$(printf "0x%08x" $(( BASE + i*4 )))
    VAL=$(devmem2 $ADDR w 2>/dev/null | grep "Value" | awk '{print $NF}')
    BITS=$(python3 -c "v=int('$VAL',16); print(format(v,'032b'))" 2>/dev/null)
    printf "%-12s %-12s %s\n" "$ADDR" "$VAL" "$BITS"
done
```

**مثال output وتفسيره:**

```
=== Pin Control Register Dump ===
Base: 0x01C20800, Count: 4 registers

Address      Value        Bits[31:0]
-------      -----        ---------
0x01c20800   0x77222277   01110111001000100010001001110111
0x01c20804   0x00000033   00000000000000000000000000110011
0x01c20808   0x00000000   00000000000000000000000000000000
0x01c2080c   0x00000000   00000000000000000000000000000000

تفسير 0x01c20800 = 0x77222277:
  Bits[2:0]   = 111 = 7 → reserved/input
  Bits[6:4]   = 111 = 7 → reserved/input
  Bits[10:8]  = 010 = 2 → I2C1_SDA ✓ (صح)
  Bits[14:12] = 010 = 2 → I2C1_SCK ✓ (صح)
  Bits[18:16] = 111 = 7 → reserved/input
  Bits[22:20] = 111 = 7 → reserved/input
  Bits[26:24] = 111 = 7 → reserved/input
  Bits[30:28] = 111 = 7 → reserved/input
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — الـ UART مش شغال بعد الـ probe

#### العنوان
الـ UART driver بيفشل silently على industrial gateway بسبب غياب الـ `default` pinctrl state

#### السياق
شركة بتبني industrial IoT gateway على SoC **RK3562**. الـ board بتحتوي على UART2 موصل بـ RS-485 transceiver للتواصل مع PLCs. الـ BSP مبني على kernel 6.1. بعد flash الـ firmware الجديد، الـ gateway مش بتستقبل أي data من الـ PLCs، لكن الـ device بيظهر في `/dev/ttyS2` بشكل طبيعي.

#### المشكلة
الـ UART driver بيعمل `probe()` بنجاح، لكن الـ pins الفيزيائية لـ UART2 (TX/RX) فاضلة على وضع GPIO مش على UART function. النتيجة: الـ RS-485 transceiver مش بيشوف أي signal.

#### التحليل
الـ device core بتستدعي `pinctrl_bind_pins()` أثناء الـ probe. الـ function دي بتبحث في `dev->pins` اللي نوعه `struct dev_pin_info` المعرف في `devinfo.h`:

```c
struct dev_pin_info {
    struct pinctrl *p;               /* handle رئيسي للـ pinctrl */
    struct pinctrl_state *default_state; /* الـ state اللي بيتطبق تلقائيًا */
    struct pinctrl_state *init_state;
#ifdef CONFIG_PM
    struct pinctrl_state *sleep_state;
    struct pinctrl_state *idle_state;
#endif
};
```

الـ `default_state` بيتملى لما الـ device core يلاقي state اسمه `"default"` (القيمة `PINCTRL_STATE_DEFAULT` المعرفة في `pinctrl-state.h`). لو مش موجود في الـ DT، الـ pointer يفضل `NULL` والـ pins مش بتتحول.

الـ `dev_pinctrl()` inline function اللي في `devinfo.h`:

```c
static inline struct pinctrl *dev_pinctrl(struct device *dev)
{
    if (!dev->pins)   /* لو مفيش pins info خالص */
        return NULL;
    return dev->pins->p;  /* بترجع الـ handle */
}
```

الـ function دي بترجع الـ handle لكن مش بتتحقق إن الـ `default_state` اتطبق فعلاً. الـ `pinctrl_init_done()` المعرف في نفس الملف بيعمل transition من `init_state` إلى `default_state` بس بعد الـ probe — لو الـ `default_state` نفسه مش معرف في الـ DTS أصلاً، مش هيحصل أي حاجة.

فحص الـ DTS القديم:
```
/* DTS غلط — مفيش pinctrl-names */
&uart2 {
    status = "okay";
};
```

#### الحل
تعريف الـ pinctrl states في الـ DTS بشكل صريح:

```dts
&uart2 {
    status = "okay";
    pinctrl-names = "default";          /* اسم الـ state لازم يطابق PINCTRL_STATE_DEFAULT */
    pinctrl-0 = <&uart2m1_xfer>;        /* المجموعة المقابلة في pinctrl node */
};
```

التحقق من إن الـ pins اتحولت فعلاً:
```bash
# عرض الـ state الحالي للـ pins
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinmux-pins | grep uart2

# عرض الـ dev_pin_info للـ device
cat /sys/kernel/debug/pinctrl/pinctrl-rockchip-pinctrl/pinmux-functions
```

#### الدرس المستفاد
الـ `default_state` في `struct dev_pin_info` مش بيتملى تلقائيًا — محتاج تعريف صريح في الـ DTS بـ `pinctrl-names = "default"`. غيابه بيخلي الـ pins على حالها من الـ reset state (عادة GPIO) من غير أي error message.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — الـ HDMI بيشتغل بعد suspend ثم بيقطع

#### العنوان
الـ HDMI بيفقد الـ signal بعد كل resume بسبب غياب `sleep_state` في الـ pinctrl config

#### السياق
منتج Android TV box مبني على **Allwinner H616**. الـ box بتشتغل تمام لما بتشغل فيديو، لكن لما الشاشة تعمل suspend (بعد 30 ثانية inactivity) وترجع، الـ HDMI بيظهر no signal لمدة 3-5 ثواني أو بيفضل مقطوع خالص لحد ما المستخدم يعمل reboot.

#### المشكلة
أثناء الـ suspend، الـ HDMI pins بتدخل state غلط — الـ pull-up/pull-down settings مش مناسبة للـ DDC lines (I2C للـ EDID)، فالـ sink device (التلفزيون) بيعتبر الـ connection اتقطعت.

#### التحليل
الـ `struct dev_pin_info` في `devinfo.h` بيحتوي على:

```c
#ifdef CONFIG_PM
    struct pinctrl_state *sleep_state;   /* بيتفعل عند الـ suspend */
    struct pinctrl_state *idle_state;    /* بيتفعل عند الـ runtime suspend */
#endif
```

الـ fields دي موجودة **فقط لو** `CONFIG_PM` متفعلة. في الـ Android kernel عادة متفعلة. الـ `pinctrl_pm_select_sleep_state()` المعرف في `consumer.h` بتبحث عن `sleep_state` في الـ `dev->pins`:

```c
/* من consumer.h — الـ stub لو CONFIG_PINCTRL مش موجود */
static inline int pinctrl_pm_select_sleep_state(struct device *dev)
{
    return 0;  /* مفيش حاجة بتحصل */
}
```

لما `CONFIG_PINCTRL` موجودة، الـ implementation الحقيقية بتجيب `dev->pins->sleep_state` — لو `NULL` (لأنه مش معرف في DTS)، الـ pins بتفضل على الـ `default` state طول الـ suspend وده بيخلي الـ DDC lines تسحب current وتبعث سيجنال للتلفزيون إن الـ HDMI مقطوع.

فحص:
```bash
# هل الـ sleep state معرف؟
cat /sys/kernel/debug/pinctrl/*/pinmux-pins 2>/dev/null | grep hdmi

# الـ sleep/wake transitions
dmesg | grep -i "pinctrl\|hdmi\|suspend"
```

#### الحل
إضافة `sleep` state في الـ DTS مع تعطيل الـ pull على DDC lines أثناء الـ suspend:

```dts
&hdmi {
    status = "okay";
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&hdmi_ddc_pins>;       /* default: pull-up على DDC */
    pinctrl-1 = <&hdmi_ddc_sleep_pins>; /* sleep: high-Z أو weak pull */
};

/* في الـ pinctrl node */
hdmi_ddc_sleep_pins: hdmi-ddc-sleep {
    pins = "PI12", "PI13";              /* SCL, SDA للـ DDC */
    function = "hdmi";
    bias-disable;                        /* high-Z أثناء الـ sleep */
};
```

#### الدرس المستفاد
الـ `sleep_state` في `struct dev_pin_info` موجود خلف `#ifdef CONFIG_PM` — ده مش مجرد تفصيلة، ده بيعني إن لازم تضيف `"sleep"` لـ `pinctrl-names` في الـ DTS لأي device بيعمل power management. فقدانه بيسبب مشاكل hardware خفية مش kernel crashes.

---

### السيناريو الثاني: Custom Board Bring-up على STM32MP157 — الـ SPI مش بيشتغل أصلاً

#### العنوان
الـ SPI driver بيرجع `-EPROBE_DEFER` بشكل لا نهائي أثناء bring-up على STM32MP157

#### السياق
فريق bring-up على custom board مبنية على **STM32MP157** (dual Cortex-A7). الـ board بتحتوي على SPI3 موصل بـ flash memory خارجية للـ configuration data. الـ driver بيرجع `-EPROBE_DEFER` في الـ dmesg بشكل متكرر ومش بيكمل الـ probe أبدًا.

#### المشكلة
الـ pinctrl driver نفسه لسه مش اتعمله probe (بيجي late في الـ boot sequence)، فالـ device core بيحاول يعمل bind للـ pins ومش بيلاقي الـ pinctrl available.

#### التحليل
الـ `pinctrl_init_done()` المعرف في `devinfo.h`:

```c
extern int pinctrl_init_done(struct device *dev);
```

الـ function دي (implemented في `core.c`) بتتحقق من إن الـ init pinctrl state اتطبق وتعمل transition للـ default. بتتسمى من الـ device core بعد الـ `probe()`. لكن المشكلة بتحصل **قبل** الـ probe — في `pinctrl_bind_pins()`:

```
device_add()
  └─ pinctrl_bind_pins(dev)
       └─ pinctrl_get(dev)        ← بيحاول يجيب الـ pinctrl handle
            └─ لو الـ pinctrl driver مش موجود لسه
                 └─ بيرجع -EPROBE_DEFER
```

الـ `dev->pins` بيفضل `NULL` والـ `dev_pinctrl()` بيرجع `NULL`:

```c
static inline struct pinctrl *dev_pinctrl(struct device *dev)
{
    if (!dev->pins)
        return NULL;   /* ده اللي بيحصل — مفيش pins binding حصل */
    return dev->pins->p;
}
```

الـ `-EPROBE_DEFER` صح تمامًا — المفروض الـ SPI driver يحاول تاني لما الـ pinctrl يبقى جاهز — لكن المشكلة إن الـ pinctrl driver عنده bug وهو نفسه مش بيكمل probe.

فحص:
```bash
# شوف كل الـ deferred devices
cat /sys/kernel/debug/devices_deferred

# تأكد إن الـ pinctrl نفسه probe
dmesg | grep "pinctrl-stm32"
ls /sys/bus/platform/drivers/pinctrl-stm32/
```

#### الحل
المشكلة الأصلية كانت في الـ DTS — الـ pinctrl node عنده `status = "disabled"` بالغلط:

```dts
/* قبل الإصلاح */
&pinctrl {
    status = "disabled";   /* !! غلط */
};

/* بعد الإصلاح */
&pinctrl {
    status = "okay";

    spi3_pins_default: spi3-default {
        pins1 {
            pinmux = <STM32_PINMUX('C', 10, AF6)>,   /* SCK */
                     <STM32_PINMUX('C', 11, AF6)>,   /* MISO */
                     <STM32_PINMUX('C', 12, AF6)>;   /* MOSI */
            bias-disable;
            drive-push-pull;
            slew-rate = <1>;
        };
    };
};
```

#### الدرس المستفاد
الـ `-EPROBE_DEFER` من `pinctrl_bind_pins()` مش دايمًا يعني مشكلة في الـ consumer device. لازم تتحقق من الـ pinctrl driver نفسه أولاً. الـ `dev->pins` بيفضل `NULL` لحد ما الـ binding ينجح — والـ `dev_pinctrl()` الـ return value الـ `NULL` ده مش بالضرورة bug، ممكن يكون symptom لـ dependency غير محلولة.

---

### السيناريو الرابع: IoT Sensor على AM62x — الـ I2C بياخد current زيادة في الـ sleep

#### العنوان
الـ battery-powered IoT sensor بياخد 15mA في الـ deep sleep بدل 50µA بسبب الـ I2C pins

#### السياق
product IoT sensor مبني على **Texas Instruments AM62x** (Cortex-A53). الـ device المفروض يشتغل على بطارية 3 سنين بـ wake interval كل 5 دقايق. القياسات بتظهر إن الـ current في الـ sleep mode 300 ضعف المطلوب.

#### المشكلة
الـ I2C2 pins (موصلين بـ environmental sensors) فاضلين على `default` state أثناء الـ deep sleep — الـ pull-up resistors الداخلية شاغلين والـ external pull-ups (4.7kΩ) بتسحب current طول الوقت.

#### التحليل
الـ kernel بيستدعي `pinctrl_pm_select_sleep_state()` عند الـ system suspend. الـ function دي بتبص على `dev->pins->sleep_state`:

```c
/* من struct dev_pin_info في devinfo.h */
struct dev_pin_info {
    struct pinctrl *p;
    struct pinctrl_state *default_state;
    struct pinctrl_state *init_state;
#ifdef CONFIG_PM
    struct pinctrl_state *sleep_state;   /* ← ده اللي بيتبص فيه */
    struct pinctrl_state *idle_state;
#endif
};
```

لو الـ `sleep_state` هو `NULL` (مش معرف في DTS)، الـ function بترجع بدون عمل أي حاجة. النتيجة: الـ I2C pins فاضلة بـ pull-up enabled، والـ 3.3V بيتدفع من خلال الـ 4.7kΩ pull-ups:

```
I = V/R = 3.3V / 4.7kΩ ≈ 0.7mA لكل line
2 lines (SDA + SCL) × 0.7mA = 1.4mA
مع كل الـ sensors والـ peripherals التانية → 15mA إجمالي
```

فحص الـ current:
```bash
# لو عندك INA219 أو similar
# أو بـ regulator debug
cat /sys/kernel/debug/regulator/regulator_summary

# شوف الـ pin states قبل الـ suspend
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep i2c2
```

#### الحل
إضافة `sleep` state بيعمل disable للـ pull-ups:

```dts
&i2c2 {
    status = "okay";
    pinctrl-names = "default", "sleep";
    pinctrl-0 = <&i2c2_pins_default>;
    pinctrl-1 = <&i2c2_pins_sleep>;
};

/* في pinctrl node الخاص بـ AM62x */
i2c2_pins_sleep: i2c2-sleep {
    pinmux = <K3_PINMUX(0x0104, PIN_INPUT, 7)>,  /* GPIO mode */
             <K3_PINMUX(0x0108, PIN_INPUT, 7)>;
    bias-disable;   /* بيعطل الـ internal pull-up/down */
};
```

بعد الإصلاح، الـ kernel بيملأ `sleep_state` في `struct dev_pin_info` عند الـ boot، وعند الـ suspend `pinctrl_pm_select_sleep_state()` بتطبق الـ state ده وبتقلل الـ current لـ < 100µA.

#### الدرس المستفاد
الـ `sleep_state` و`idle_state` في `struct dev_pin_info` مش luxury — في الأنظمة المعتمدة على البطارية هما حياة أو موت. كل I2C/SPI/UART device محتاج explicit `sleep` pinctrl state يعطل الـ pull resistors لتوفير الـ power.

---

### السيناريو الخامس: Automotive ECU على i.MX8M Plus — الـ CAN bus بياخد وقت طويل للـ init

#### العنوان
الـ CAN controller على i.MX8M Plus بياخد 2 ثانية زيادة في الـ boot بسبب مشكلة في الـ `init_state` transition

#### السياق
automotive ECU مبني على **NXP i.MX8M Plus**. الـ ECU بيتحكم في body electronics وبيتكلم مع CAN bus. المطلب الـ functional safety إن الـ CAN interface يكون جاهز في أقل من 500ms من الـ power-on. القياسات بتظهر إن الـ `flexcan` driver بياخد 2.3 ثانية.

#### المشكلة
الـ CAN pins بتتحول من `init` state (بيمنع glitches) للـ `default` state بشكل متأخر جدًا — الـ `pinctrl_init_done()` مش بيتسمى في الوقت المناسب.

#### التحليل
الـ `devinfo.h` بيعرف `init_state`:

```c
struct dev_pin_info {
    struct pinctrl *p;
    struct pinctrl_state *default_state;
    struct pinctrl_state *init_state;   /* ← الـ state اللي بيتطبق قبل الـ probe */
    /* ... */
};
```

الـ flow المقصود:
```
device_add()
  └─ pinctrl_bind_pins()
       └─ لو init_state موجود → طبقه أولاً
            └─ driver probe()
                 └─ pinctrl_init_done(dev)   ← لازم الـ driver يسميها
                      └─ لو الـ pins لسه على init → حولهم لـ default
```

الـ `pinctrl_init_done()` المعلن في `devinfo.h`:

```c
extern int pinctrl_init_done(struct device *dev);
```

الـ `flexcan` driver الـ custom المعدل بيعمل defer لـ `pinctrl_init_done()` لحد ما يعمل CAN bus synchronization — ده بيأخر الـ transition من `init` → `default`، والـ CAN transceiver مش بيشتغل بكفاءة على الـ `init` state (اللي مصمم يمنع الـ glitches مش يشغل الـ normal traffic).

فحص:
```bash
# timing من الـ boot
dmesg -T | grep -E "flexcan|pinctrl|can0"

# الـ current pin state
cat /sys/kernel/debug/pinctrl/30330000.iomuxc/pinmux-pins | grep -i can

# هل init_state لسه active؟
cat /sys/kernel/debug/pinctrl/*/pinmux-pins
```

كود المشكلة في الـ driver المعدل:

```c
static int flexcan_probe(struct platform_device *pdev)
{
    /* ... setup ... */

    /* المشكلة: الـ driver بيعمل can sync قبل pinctrl_init_done */
    ret = flexcan_can_sync(priv);  /* ده بياخد وقت طويل */
    if (ret)
        return ret;

    /* pinctrl_init_done متأخر جدًا */
    pinctrl_init_done(&pdev->dev);
    return 0;
}
```

#### الحل
تقديم استدعاء `pinctrl_init_done()` قبل الـ CAN sync:

```c
static int flexcan_probe(struct platform_device *pdev)
{
    /* ... basic setup ... */

    /*
     * انقل الـ pins من init → default أولاً
     * عشان الـ transceiver يكون جاهز قبل الـ CAN sync
     */
    ret = pinctrl_init_done(&pdev->dev);
    if (ret)
        return ret;

    /* دلوقتي ابدأ الـ CAN bus sync */
    ret = flexcan_can_sync(priv);
    if (ret)
        return ret;

    return 0;
}
```

وفي الـ DTS:

```dts
&flexcan1 {
    status = "okay";
    pinctrl-names = "default", "init";
    pinctrl-0 = <&flexcan1_pins>;         /* default: normal operation */
    pinctrl-1 = <&flexcan1_init_pins>;    /* init: glitch-free during probe */
};

flexcan1_init_pins: flexcan1-init {
    /* نفس الـ mux لكن بـ pull-down على TX عشان يمنع spurious frames */
    fsl,pins = <MX8MP_IOMUXC_SAI5_RXD1__CAN1_TX  0x154>,
               <MX8MP_IOMUXC_SAI5_RXD2__CAN1_RX  0x154>;
};
```

#### الدرس المستفاد
**الـ `init_state`** في `struct dev_pin_info` مصمم لحالة واحدة محددة: لما تطبيق الـ `default` state مباشرة سيسبب glitch على الـ bus. الـ driver **مسئول** إنه يستدعي `pinctrl_init_done()` بنفسه في الوقت المناسب من الـ probe — مش تلقائي. تأخيره بيعني تأخير الـ CAN/UART/SPI من الاشتغال بشكل كامل. في سياق automotive، ده بيخل بمتطلبات الـ boot timing الـ safety-critical.
## Phase 7: مصادر ومراجع

### مصادر الـ kernel الرسمية

| المصدر | الوصف |
|--------|-------|
| [`include/linux/pinctrl/devinfo.h`](https://github.com/torvalds/linux/blob/master/include/linux/pinctrl/devinfo.h) | الملف الأصلي — `struct dev_pin_info` وكل الـ stubs |
| [`drivers/pinctrl/core.c`](https://github.com/torvalds/linux/blob/master/drivers/pinctrl/core.c) | الـ core implementation — بيستخدم `devinfo.h` مباشرة |
| [`Documentation/driver-api/pin-control.rst`](https://docs.kernel.org/driver-api/pin-control.html) | التوثيق الرسمي الكامل لـ pinctrl subsystem |
| [`www.kernel.org/doc/Documentation/pinctrl.txt`](https://www.kernel.org/doc/Documentation/pinctrl.txt) | النسخة النصية القديمة من التوثيق |
| [Bootlin Elixir — devinfo.h](https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/devinfo.h) | Cross-reference تفاعلي للكود |

---

### مقالات LWN.net

دي أهم المقالات اللي بتغطي الـ pinctrl subsystem وتطوره:

#### الأساس — إنشاء الـ subsystem

- [**Documentation/pinctrl.txt**](https://lwn.net/Articles/465077/) — أول نسخة من التوثيق الرسمي اللي بتشرح المفاهيم الأساسية للـ subsystem
- [**pin controller subsystem v7**](https://lwn.net/Articles/459190/) — مرحلة متقدمة من تطوير الـ subsystem قبل الـ merge
- [**drivers: create a pin control subsystem**](https://lwn.net/Articles/463335/) — الـ patch اللي أنشأ الـ subsystem فعلاً
- [**drivers: create a pin control subsystem v8**](https://lwn.net/Articles/460768/) — الـ patch النهائي قبل الـ merge في الـ mainline
- [**drivers: create a pinmux subsystem v2**](https://lwn.net/Articles/442315/) — من بداية الفكرة كـ pinmux-only
- [**drivers: create a pinmux subsystem v3**](https://lwn.net/Articles/447394/) — تطور الفكرة نحو subsystem أشمل

#### الـ pin states و`devinfo.h` تحديداً

- [**drivers: pinctrl sleep and idle states in the core**](https://lwn.net/Articles/552972/) — الـ patch اللي أضاف `sleep_state` و`idle_state` لـ `struct dev_pin_info` — مباشرة متعلق بالملف ده
- [**drivers/pinctrl: Add the concept of an "init" state**](https://lwn.net/Articles/615322/) — الـ patch اللي أضاف `init_state` لـ `struct dev_pin_info`

#### الـ pin config interface

- [**pinctrl: add a generic pin config interface**](https://lwn.net/Articles/468770/) — تصميم الـ generic config API
- [**pinctrl: add a pin config interface**](https://lwn.net/Articles/471826/) — تفاصيل الـ implementation
- [**pinctrl: add a generic pin config interface (earlier)**](https://lwn.net/Articles/467269/) — نسخة مبكرة من نفس الـ patch
- [**pinctrl/pinconfig: add debug interface**](https://lwn.net/Articles/545790/) — إضافة debugfs للـ pin config

---

### مناقشات الـ mailing list والـ commits المهمة

الـ commits دي هي اللي شكّلت `devinfo.h` بشكله الحالي:

```
# الـ commit اللي أنشأ struct dev_pin_info الأصلي
# Author: Linus Walleij <linus.walleij@linaro.org>
# Copyright (C) 2012 ST-Ericsson SA

# الـ commit اللي أضاف sleep/idle states (CONFIG_PM)
# → راجع: https://lwn.net/Articles/552972/

# الـ commit اللي أضاف init_state
# → راجع: https://lwn.net/Articles/615322/
```

**الـ embedded.com article** بيشرح الـ subsystem من منظور driver developer:
- [Linux device driver development: The pin control subsystem](https://www.embedded.com/linux-device-driver-development-the-pin-control-subsystem/)

**الـ STM32 wiki** — مثال عملي كامل:
- [Pinctrl overview — stm32mpu](https://wiki.st.com/stm32mpu/wiki/Pinctrl_overview)

---

### kernelnewbies.org

الموقع بيوثّق التغييرات في كل إصدار kernel. ابحث عن pinctrl في:

| الصفحة | ما تلاقيه |
|--------|-----------|
| [Linux_6.11](https://kernelnewbies.org/Linux_6.11) | تغييرات الـ pinctrl drivers في 6.11 |
| [Linux_6.13](https://kernelnewbies.org/Linux_6.13) | pinctrl updates في 6.13 |
| [Linux_6.14](https://kernelnewbies.org/Linux_6.14) | pinctrl updates في 6.14 |
| [LinuxChanges](https://kernelnewbies.org/LinuxChanges) | تاريخ كامل للتغييرات |

---

### elinux.org

| الصفحة | ما تلاقيه |
|--------|-----------|
| [EBC Device Trees](https://elinux.org/EBC_Device_Trees) | أمثلة عملية لـ pinctrl في device tree |
| [Tests:i2c-demux-pinctrl](https://www.elinux.org/Tests:i2c-demux-pinctrl) | اختبار الـ i2c-demux-pinctrl driver |
| [Pin Control GPIO Update (PDF)](https://elinux.org/images/c/cd/Pincontrol-gpio-update.pdf) | presentation من مؤتمر ELC عن pinctrl/GPIO |
| [Linux Kernel Resources](https://elinux.org/Linux_Kernel_Resources) | روابط مصادر kernel عامة |

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 1**: نظرة عامة على بنية الـ kernel وطبقات الـ drivers
- **الفصل 14 (The Linux Device Model)**: شرح `struct device` و`dev->pins` اللي بيحمل `dev_pin_info`
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Ed.)
- **الفصل 14**: The Block I/O Layer — بيفهّمك تصميم الـ subsystems
- **الفصل 17**: Devices and Modules — كيف بيتعامل الـ kernel مع الـ devices
- الـ power management chapters — ضروري لفهم `sleep_state` و`idle_state`

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 15**: Kernel Initialization — بيشرح الـ device probe sequence اللي فيها `pinctrl_init_done()` بيتكلم
- **الفصل 16**: Using a Root File System — أمثلة على embedded hardware وكيف بيشتغل الـ pinctrl

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- **Chapter 6**: Device Drivers — تفاصيل عميقة عن device model
- مفيد لفهم العلاقة بين `struct device` والـ subsystems المختلفة

---

### توثيق الـ kernel الرسمي — مسارات `Documentation/`

```
Documentation/driver-api/pin-control.rst   ← التوثيق الأساسي الكامل
Documentation/devicetree/bindings/pinctrl/ ← DT bindings للـ pinctrl
Documentation/power/runtime_pm.rst         ← شرح runtime PM (idle_state)
Documentation/driver-api/pm/devices.rst    ← شرح sleep states (sleep_state)
```

---

### search terms للبحث عن معلومات أكتر

```
# للبحث في LKML (mailing list أرشيف)
site:lore.kernel.org "dev_pin_info"
site:lore.kernel.org "pinctrl_init_done"
site:lore.kernel.org "pinctrl sleep state"
site:lore.kernel.org "pinctrl idle state"

# للبحث العام
linux kernel "struct dev_pin_info" explanation
linux pinctrl "init_state" probe sequence
linux pinctrl "sleep_state" suspend runtime_pm
"CONFIG_PINCTRL" "dev->pins" device core integration
pinctrl_bind_pins device probe pinctrl
linux kernel pinctrl consumer API tutorial
```

---

### ملخص سريع للمراجع الأهم

| الأولوية | المرجع | السبب |
|---------|--------|-------|
| 1 | [docs.kernel.org/driver-api/pin-control.html](https://docs.kernel.org/driver-api/pin-control.html) | المرجع الرسمي الشامل |
| 2 | [LWN: sleep and idle states](https://lwn.net/Articles/552972/) | الـ patch الأساسي لـ `devinfo.h` |
| 3 | [LWN: init state](https://lwn.net/Articles/615322/) | الـ patch اللي أضاف `init_state` |
| 4 | [LWN: pinctrl subsystem creation](https://lwn.net/Articles/463335/) | الأصل التاريخي للـ subsystem |
| 5 | [Bootlin Elixir](https://elixir.bootlin.com/linux/latest/source/include/linux/pinctrl/devinfo.h) | cross-reference تفاعلي للكود |
| 6 | LDD3 Chapter 14 | فهم device model |
## Phase 8: Writing simple module

### الفكرة

الـ function اللي هنعمل لها hook هي **`pinctrl_select_state`** — دي exported function من الـ pinctrl subsystem بتتحوّط كل مرة device بيغيّر حالة الـ pins بتاعته (مثلاً من `default` لـ `sleep` وقت الـ suspend). ده بيخلّيها نقطة مراقبة مثالية لأي حركة في الـ pin states على مستوى الـ system كله.

هنستخدم **kprobe** لأن `pinctrl_select_state` function عادية مش tracepoint ومش notifier.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on pinctrl_select_state()
 * Logs every pin-state transition on the system.
 */

/* --- Includes --- */
#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>      /* kprobe struct, register/unregister */
#include <linux/device.h>       /* struct device, dev_name() */
#include <linux/pinctrl/consumer.h> /* pinctrl_select_state signature */

/*
 * نحتاج kprobes.h عشان نعرّف الـ kprobe وندير callbacks.
 * نحتاج device.h عشان نقدر نطبع اسم الـ device من الـ args.
 * نحتاج consumer.h عشان نعرف signature الـ function اللي بنعمل hook عليها.
 */

/* --- Pre-handler callback --- */
/*
 * الـ pre_handler بيتشغّل لحظة وصول CPU لأول instruction
 * في pinctrl_select_state قبل ما تنفّذ أي حاجة.
 * الـ regs بتحتوي على registers الحالية — منها args الـ function.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64: first arg (struct pinctrl *p)  → regs->di
     *            second arg (struct pinctrl_state *s) → regs->si
     *
     * pinctrl is an opaque cookie — we can't safely dereference it
     * from a kprobe without internal headers, so we just print the
     * pointer values as addresses for correlation with ftrace/perf.
     */
    struct pinctrl       *pinctrl_handle = (struct pinctrl *)regs->di;
    struct pinctrl_state *state          = (struct pinctrl_state *)regs->si;

    pr_info("pinctrl_probe: select_state called | "
            "pinctrl=%px  state=%px\n",
            pinctrl_handle, state);

    return 0; /* 0 = continue normal execution */
}

/*
 * الـ post_handler بيتشغّل بعد ما الـ function تنفّذ وترجع.
 * هنا بنطبع إن الـ function خلصت بنجاح عشان نقدر نتابع
 * أي state transition اتمّت فعلاً.
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    pr_info("pinctrl_probe: select_state returned | ax(ret)=%ld\n",
            (long)regs->ax);
}

/* --- kprobe struct --- */
/*
 * الـ kprobe struct بتربط الاسم بالـ callbacks.
 * بنستخدم symbol_name بدل عنوان ثابت عشان يشتغل مع KASLR.
 */
static struct kprobe kp = {
    .symbol_name = "pinctrl_select_state",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* --- module_init --- */
/*
 * بنسجّل الـ kprobe هنا — من اللحظة دي أي call لـ pinctrl_select_state
 * هيمرّ على callbacks بتاعتنا.
 */
static int __init pinctrl_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("pinctrl_probe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("pinctrl_probe: hooked pinctrl_select_state at %px\n",
            kp.addr);
    return 0;
}

/* --- module_exit --- */
/*
 * لازم نشيل الـ kprobe في exit عشان منسيبش dangling hook
 * يودّي لـ kernel crash لو الـ module اتحط من الذاكرة.
 */
static void __exit pinctrl_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("pinctrl_probe: unhooked pinctrl_select_state\n");
}

module_init(pinctrl_probe_init);
module_exit(pinctrl_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe demo: trace every pinctrl_select_state() call");
```

---

### شرح كل جزء

#### الـ Includes

| Header | ليه موجود |
|--------|-----------|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل API الخاص بيه |
| `linux/device.h` | تعريف `struct device` — لو احتجنا نطبع اسم device لاحقاً |
| `linux/pinctrl/consumer.h` | توثيق signature الـ function اللي بنعمل لها probe |

**الـ `consumer.h`** مش ضروري technically لكنه مفيد كـ documentation لأنه بيحدد إن `pinctrl_select_state` بتاخد `(struct pinctrl *, struct pinctrl_state *)`.

---

#### الـ `handler_pre`

بيتشغّل قبل تنفيذ الـ function. بنقرأ الـ args من الـ registers مباشرة (على x86-64: `rdi` = arg1, `rsi` = arg2). بنطبع عناوين الـ handles كـ pointers عشان نقدر نربطهم بـ tools زي `ftrace` أو `perf`.

---

#### الـ `handler_post`

بيتشغّل بعد ما الـ function ترجع. بنطبع قيمة `rax` اللي هي الـ return value — لو رجعت 0 معناه الـ state transition نجحت، لو رجعت negative فيه error.

---

#### الـ `kprobe` struct

**`symbol_name`** بدل عنوان ثابت عشان الـ kernel بيفعّل KASLR (عناوين بتتغير مع كل boot)، فالـ kprobe subsystem هو اللي بيحلّ الاسم لعنوان صح وقت الـ register.

---

#### الـ `module_init` / `module_exit`

`register_kprobe` بتحط **breakpoint** (على x86: `int3`) على أول byte من `pinctrl_select_state`. لو فشل التسجيل (مثلاً الـ function مش exported أو محميّة بـ `nokprobe_inline`) بنرجع error ومبنكملش. في الـ exit بنعمل `unregister_kprobe` بشكل إلزامي عشان الكيرنل يشيل الـ breakpoint ويرجع الـ original bytes ويضمن إن مفيش CPU بيرن في كود اتحذف.

---

### طريقة الاختبار

```bash
# Build
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# Load
sudo insmod pinctrl_probe.ko

# Trigger a state change (e.g. suspend/resume or load a driver)
sudo sh -c "echo mem > /sys/power/state"

# Watch the log
sudo dmesg | grep pinctrl_probe

# Unload
sudo rmmod pinctrl_probe
```

### Makefile

```makefile
obj-m += pinctrl_probe.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```
