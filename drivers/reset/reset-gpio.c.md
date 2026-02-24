## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Reset Controller Framework؟

تخيل إن عندك جهاز كمبيوتر وفيه كارت شبكة، وعايز تعمله restart من غير ما تعيد تشغيل الجهاز كله. في العالم الحقيقي بتضغط زرار صغير أو بتقطع الكهرباء عنه ثانية وترجعها. في الـ SoC (System on Chip) — الشرائح اللي بتلاقيها في التليفونات والـ embedded systems — كل جزء صغير جوا الشريحة ممكن يكون محتاج "زرار reset" خاص بيه.

الـ **Reset Controller Framework** في لينكس هو الإطار اللي بيوحد إزاي drivers بتقول "احنا محتاجين نعمل reset لهذا الجهاز" وإزاي الـ reset providers بتنفذ ده.

---

### القصة: المشكلة اللي `reset-gpio.c` بتحلها

#### الموقف الكلاسيكي

عندك SoC فيه module داخلي — مثلاً USB controller أو Ethernet MAC. الـ SoC بيوفر reset register جوا نفسه، فالـ driver بيكتب على بت معين فيعمل reset للـ module ده. ده الحالة العادية، وكل صانع شريحة عنده driver خاص بيه زي `reset-imx7.c` أو `reset-sunxi.c`.

#### المشكلة

أحياناً الجهاز اللي عايز تعمله reset مش جزء من الـ SoC نفسه، بل **chip خارجي** موصل على البورد. مثلاً:

- تليفون فيه Ethernet chip خارجي زي RTL8211 موصل بـ GPIO pin رقم 42 على الـ SoC.
- لما البورد اتصممت، المهندس وصل الـ RESET# pin بتاع الـ RTL8211 على GPIO42.
- عشان تعمل reset للـ RTL8211، بس تشيل GPIO42 high لثانية وترجعها low.

السؤال: إزاي الـ Ethernet driver يعمل reset لنفسه بطريقة موحدة من غير ما يعرف إنه موصل على GPIO؟

الجواب: يطلب reset عن طريق Reset Controller API عادي، والـ kernel يحول الطلب ده لـ GPIO operation من تحت.

هنا يجي دور `reset-gpio.c`.

---

### إيه اللي بيعمله `reset-gpio.c`؟

**الـ `reset-gpio.c`** هو driver بسيط جداً بيلف GPIO pin واحد ويقدمه كـ **Reset Controller**. يعني:

- أي driver عايز يعمل reset لجهاز خارجي موصل على GPIO
- يطلب الـ reset عن طريق الـ Reset Controller API الموحد
- الـ `reset-gpio.c` يترجم الطلب ده لـ `gpiod_set_value()`

ببساطة: هو **وسيط** بين عالم الـ Reset Controller وعالم الـ GPIO.

---

### إزاي بيشتغل؟ (التدفق الكامل)

```
Device Tree:
  eth0: ethernet {
      reset-gpios = <&gpio 42 GPIO_ACTIVE_LOW>;
  };
         |
         v
reset/core.c يشوف إن الـ reset مش phandle عادي
بل reset-gpios، فيعمل auxiliary device تلقائياً
         |
         v
reset-gpio.c probe() يتسمى
يمسك الـ GPIO pin بـ devm_gpiod_get()
يسجل نفسه كـ reset_controller_dev
         |
         v
eth_driver يطلب:
  reset_control_assert(rst)   → gpiod_set_value(gpio, 1)
  reset_control_deassert(rst) → gpiod_set_value(gpio, 0)
  reset_control_status(rst)   → gpiod_get_value(gpio)
```

---

### الـ Auxiliary Bus: ليه مش Platform Device؟

الـ driver مش بيستخدم الـ platform bus العادي. بيستخدم **Auxiliary Bus** — وده bus داخلي في الكيرنل بيستخدم لما driver رئيسي محتاج يعمل sub-devices داخلية بدون ما يكون ليها نود في الـ Device Tree مباشرة.

الـ `reset/core.c` هو اللي بيعمل الـ auxiliary device برمجياً لما يشوف `reset-gpios` في الـ Device Tree، وبعدين الـ `reset-gpio.c` يبروب عليه.

---

### الـ Operations الثلاثة

| Operation | الوظيفة | GPIO Value |
|-----------|---------|------------|
| `assert` | يثبت الـ reset (الجهاز محجوز في reset) | HIGH (1) |
| `deassert` | يرفع الـ reset (الجهاز يشتغل) | LOW (0) |
| `status` | يقرأ الحالة الحالية | `gpiod_get_value()` |

ملحوظة: القيم دي بتتقلب لو الـ GPIO كان `ACTIVE_LOW` — الـ GPIO descriptor API بيتعامل مع ده تلقائياً.

---

### الـ Subsystem اللي ينتمي ليه

الملف ينتمي لـ subsystemين:
1. **RESET CONTROLLER FRAMEWORK** — الإطار الرئيسي
2. **GENERIC GPIO RESET DRIVER** — entry منفصل في MAINTAINERS

المشرف: Krzysztof Kozlowski `<krzk@kernel.org>`

---

### الملفات المرتبطة

#### الـ Core والـ Headers

| الملف | الدور |
|-------|-------|
| `drivers/reset/core.c` | الـ Reset Controller Framework نفسه، اللي بيعمل الـ auxiliary device لـ reset-gpio |
| `include/linux/reset-controller.h` | تعريف `struct reset_controller_dev` و`struct reset_control_ops` |
| `include/linux/reset.h` | الـ consumer API: `reset_control_assert/deassert/reset()` |
| `include/linux/gpio/consumer.h` | الـ GPIO descriptor API: `gpiod_set_value_cansleep()` وغيرها |
| `include/linux/auxiliary_bus.h` | تعريف `struct auxiliary_device` و`struct auxiliary_driver` |

#### Hardware Drivers أمثلة في نفس الـ directory

| الملف | الـ SoC/Platform |
|-------|-----------------|
| `drivers/reset/reset-simple.c` | Generic MMIO-based reset |
| `drivers/reset/reset-imx7.c` | NXP i.MX7 |
| `drivers/reset/reset-sunxi.c` | Allwinner SoCs |
| `drivers/reset/reset-zynqmp.c` | Xilinx ZynqMP |
| `drivers/reset/reset-gpio.c` | **أي GPIO pin على أي منصة** |

#### الـ Documentation

| الملف | المحتوى |
|-------|---------|
| `Documentation/driver-api/reset.rst` | الـ Reset Framework API الكامل |
| `Documentation/devicetree/bindings/reset/reset.txt` | كيف تكتب reset في Device Tree |

---

### ليه الملف مهم رغم صغر حجمه؟

الـ `reset-gpio.c` بالكاد 118 سطر، لكن أهميته إنه يخلي الـ Reset Framework **platform-agnostic** تماماً. أي chip خارجي على أي بورد بأي GPIO، يقدر يتعمل له reset بنفس الـ API اللي بيستخدمه كل driver تاني. ده مبدأ الـ **abstraction** في أجمل صوره.
## Phase 2: شرح الـ Reset Controller Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في أي SoC حقيقي زي Qualcomm SM8450 أو NXP i.MX8، فيه عشرات الـ peripherals — USB controller، Ethernet MAC، PCIe، DSP، كاميرا، إلخ. كل واحد منهم محتاج **reset line** — خط بيخليه يبدأ من حالة نظيفة أو يتعافى من حالة hang.

المشكلة إن الـ reset lines دي مش uniform:
- بعضها **register-mapped**: بتكتب bit في register في الـ SoC's Clock & Reset Controller (CRC) فيتعمل assert للـ reset.
- بعضها **GPIO-based**: بين بتعلي أو بتخفض GPIO line عشان تعمل reset لجهاز خارجي (مثلاً Ethernet PHY، Wi-Fi module، display bridge).
- بعضها **shared**: أكتر من device بيتشارك نفس الـ reset line.
- بعضها **self-deasserting**: بترسله pulse وهو بيطلع تاني لوحده.

من غير framework موحد، كل driver كان هيتعامل مع الـ reset بطريقته الخاصة — driver الـ USB هيكتب مباشرة في registers الـ CRC، driver الـ PHY هيتكلم مع الـ GPIO subsystem مباشرة، ومفيش abstraction. النتيجة: **code duplication، tight coupling، وصعوبة في الـ portability**.

---

### الحل — إيه الـ approach بتاع الـ kernel؟

الـ kernel حل المشكلة بإنه فصل:
1. **الـ consumer** — الـ driver اللي *محتاج* يعمل reset لـ device تاني (مثلاً driver الـ USB محتاج يعمل reset للـ USB PHY).
2. **الـ provider** — الـ driver اللي *يقدر ينفذ* الـ reset فعلاً (سواء كان GPIO أو register في CRC أو أي حاجة تانية).

الـ framework بيوفر:
- **`reset_controller_dev`**: الـ struct اللي بيمثل أي مصدر لـ reset lines — ممكن يكون CRC في الـ SoC أو GPIO واحد.
- **`reset_control_ops`**: vtable فيها function pointers للـ assert، deassert، وstatus.
- **`reset_control`**: handle بيمثل reset line واحد، بتاخده الـ consumer.

الـ framework بيعمل **lookup** بين الـ consumer اللي طلب reset وبين الـ provider المناسب عن طريق الـ Device Tree أو ACPI.

---

### تشبيه من الواقع — بالتفصيل

تخيل **لوحة توزيع كهرباء** في مبنى:

| عنصر الـ analogy | المقابل في الـ kernel |
|---|---|
| الـ circuit breaker الرئيسي (MCB) | الـ `reset_controller_dev` — هو المصدر اللي بيتحكم في الـ reset lines |
| كل fuse/breaker منفرد داخل اللوحة | كل `reset_control` — يمثل reset line واحد بـ ID معين |
| السلك اللي بين الـ breaker والجهاز | الـ mapping في الـ Device Tree (phandle) |
| الكهربائي اللي بيقفل/بيفتح الـ breaker | الـ provider driver — بيشيل الـ `reset_control_ops` |
| الـ tenant اللي طلب يقفل الكهرباء عن الأوضة | الـ consumer driver — بيعمل `reset_control_get` |
| شركة الكهرباء اللي بتنظم العملية دي | الـ Reset Controller Framework نفسه |

التفاصيل المهمة في التشبيه:
- الـ tenant (consumer) **مش محتاج يعرف** الـ breaker physical أو شغلانة الأسلاك — بس يطلب "قفّل الكهرباء عن الأوضة دي".
- الـ framework هو اللي بيعمل الـ lookup ويجيب الـ handle الصح.
- الـ MCB (provider) بيختلف من مبنى لمبنى — في مبنى قديم ممكن يكون fuse wire، في جديد ممكن يكون electronic breaker — بس الـ tenant بيتعامل بنفس الطريقة.
- لو الـ breaker shared بين أوض كتير، الـ framework بيتعامل مع الـ shared reset scenario.

---

### الـ Big Picture Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        Consumer Drivers                         │
  │  ┌─────────────┐   ┌──────────────┐   ┌──────────────────────┐ │
  │  │  USB Driver  │   │  PCIe Driver │   │  Ethernet PHY Driver │ │
  │  └──────┬──────┘   └──────┬───────┘   └──────────┬───────────┘ │
  └─────────┼─────────────────┼──────────────────────┼─────────────┘
            │ reset_control_get()                     │
            ▼                                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │              Reset Controller Framework (core)                  │
  │                  drivers/reset/core.c                           │
  │                                                                 │
  │   reset_control_assert()  ──►  ops->assert(rcdev, id)          │
  │   reset_control_deassert() ─►  ops->deassert(rcdev, id)        │
  │   reset_control_status()  ──►  ops->status(rcdev, id)          │
  │                                                                 │
  │   [ list of reset_controller_dev ]                              │
  └──────────────┬───────────────────────────────┬─────────────────┘
                 │                               │
                 ▼                               ▼
  ┌──────────────────────────┐    ┌─────────────────────────────────┐
  │  SoC CRC Provider        │    │  GPIO Reset Provider            │
  │  (e.g. clk-imx8mq.c)    │    │  (reset-gpio.c)  ◄── نحن هنا   │
  │                          │    │                                 │
  │  reset_controller_dev    │    │  reset_controller_dev           │
  │  nr_resets = 200         │    │  nr_resets = 1                  │
  │  ops->assert =           │    │  ops->assert =                  │
  │    write_register()      │    │    gpiod_set_value(1)           │
  │  ops->deassert =         │    │  ops->deassert =                │
  │    clear_register()      │    │    gpiod_set_value(0)           │
  └──────────────────────────┘    └────────────────┬────────────────┘
                                                   │
                                                   ▼
                                  ┌─────────────────────────────────┐
                                  │  GPIO Subsystem (gpiolib)       │
                                  │  gpiod_set_value_cansleep()     │
                                  └─────────────────────────────────┘
                                                   │
                                                   ▼
                                  ┌─────────────────────────────────┐
                                  │  Physical GPIO Pin              │
                                  │  (e.g. GPIO3_IO14 → RESET# pin │
                                  │   of Ethernet PHY RTL8211F)    │
                                  └─────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ Reset Controller Framework قائم على فكرة واحدة محورية:

**كل مصدر لـ reset lines هو `reset_controller_dev` بيوفر vtable من الـ ops.**

الـ abstraction بتقول: "مش مهم إزاي الـ reset بيتنفذ فيزيائياً — سواء بـ register write أو GPIO toggle أو I2C command — القصة كلها هي: assert، deassert، status."

---

### الـ Structs وعلاقتهم ببعض

```
  struct reset_gpio_priv          (الـ private state بتاع الـ driver ده)
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │   struct reset_controller_dev  rc                        │
  │   ┌────────────────────────────────────────────────────┐ │
  │   │  const struct reset_control_ops  *ops  ───────────►│─┼──► reset_gpio_ops
  │   │  struct module                   *owner            │ │      { .assert   = reset_gpio_assert   }
  │   │  struct list_head                list              │ │      { .deassert = reset_gpio_deassert }
  │   │  struct list_head                reset_control_head│ │      { .status   = reset_gpio_status   }
  │   │  struct device                   *dev              │ │
  │   │  struct device_node              *of_node          │ │
  │   │  const struct of_phandle_args    *of_args  ────────┼─┼──► platdata من الـ auxiliary bus
  │   │  int                             of_reset_n_cells  │ │    (فيها of_node + GPIO args)
  │   │  int (*of_xlate)(...)            ─────────────────►│─┼──► reset_gpio_of_xlate()
  │   │  unsigned int                    nr_resets = 1     │ │
  │   └────────────────────────────────────────────────────┘ │
  │                                                          │
  │   struct gpio_desc  *reset  ─────────────────────────────┼──► GPIO line فعلي
  │                                                          │    (output, active-high)
  └──────────────────────────────────────────────────────────┘
```

الـ `container_of` في الـ function `rc_to_reset_gpio` بتعمل الـ trick المعروفة: لما الـ framework بيجي يستدعي `ops->assert(rcdev, id)` وبيمرر pointer على الـ `rc` field، الـ driver بيرجع pointer على الـ `reset_gpio_priv` الكامل عن طريق حساب الـ offset.

```c
/* من rc (الـ embedded struct) نوصل للـ priv (الـ outer struct) */
static inline struct reset_gpio_priv *rc_to_reset_gpio(struct reset_controller_dev *rc)
{
    return container_of(rc, struct reset_gpio_priv, rc);
}
```

---

### الـ Auxiliary Bus — مفهوم مهم قبل ما نكمل

الـ **Auxiliary Bus** هو subsystem في الـ kernel بيسمح لـ driver يخلق "sub-devices" افتراضية أسفل نفسه، من غير ما يكون فيه hardware node مستقل في الـ Device Tree لكل واحد. بيُستخدم لما driver رئيسي (مثلاً reset controller في SoC) محتاج يخلق child device مخصوص لـ GPIO reset line معين.

الـ `reset-gpio.c` بيستخدم الـ auxiliary bus عشان:
- الـ parent driver (مثلاً driver بتاع SoC معين) بيخلق `auxiliary_device` بالـ platform data المناسبة (بما فيها `of_phandle_args` اللي بيحدد الـ GPIO).
- الـ `reset-gpio.c` بيعمل `probe` على الـ auxiliary device ده ويسجل نفسه كـ reset controller.

---

### الـ Device Tree — إزاي الربط بيحصل؟

```dts
/* الـ SoC Device Tree */
soc {
    /* الـ PHY بيعلن إن هو محتاج reset عن طريق GPIO */
    ethernet-phy@0 {
        compatible = "ethernet-phy-ieee802.3-c22";
        reset-gpios = <&gpio3 14 GPIO_ACTIVE_LOW>;
    };
};
```

الـ `reset-gpios` property في الـ DT بيقول: "الـ reset بتاعي هو GPIO رقم 14 في الـ gpio3 controller، وهو active-low."

الـ `of_phandle_args` struct بيشيل المعلومات دي:
- `of_node` → يشاور على الـ GPIO controller node في الـ DT
- `args[0]` → رقم الـ GPIO
- `args[1]` → الـ flags (active-low, active-high)

---

### تفاصيل الـ Probe Flow

```
reset_gpio_probe()
        │
        ├─ dev_get_platdata()   ── جيب الـ of_phandle_args من الـ auxiliary device
        │                          (فيها معلومات الـ GPIO من الـ Device Tree)
        │
        ├─ devm_kzalloc()       ── allocate الـ reset_gpio_priv
        │
        ├─ devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH)
        │                       ── اطلب الـ GPIO بالاسم "reset"
        │                          GPIOD_OUT_HIGH يعني: output، initial value = 1
        │                          (أي الـ device هيكون في reset من الأول)
        │
        ├─ priv->rc.ops = &reset_gpio_ops
        ├─ priv->rc.of_args = platdata   ── ربط الـ controller بالـ DT args
        ├─ priv->rc.of_reset_n_cells = 2 ── الـ specifier بيتكون من خليتين (GPIO# + flags)
        ├─ priv->rc.of_xlate = reset_gpio_of_xlate
        │                       ── بتحول الـ DT specifier لـ ID
        │                          هنا بترجع args[0] (رقم الـ GPIO) كـ ID
        ├─ priv->rc.nr_resets = 1        ── هو reset controller بيوفر reset واحد بس
        │
        └─ devm_reset_controller_register()
                                ── سجّل الـ controller في الـ framework
                                   من دلوقتي أي consumer يعمل reset_control_get()
                                   هيلاقي الـ controller ده
```

---

### الـ Assert و Deassert — المنطق الفيزيائي

```c
/* assert = put device in reset = GPIO HIGH (active-high) */
static int reset_gpio_assert(struct reset_controller_dev *rc, unsigned long id)
{
    struct reset_gpio_priv *priv = rc_to_reset_gpio(rc);
    return gpiod_set_value_cansleep(priv->reset, 1); /* logic 1 = IN RESET */
}

/* deassert = release device from reset = GPIO LOW */
static int reset_gpio_deassert(struct reset_controller_dev *rc, unsigned long id)
{
    struct reset_gpio_priv *priv = rc_to_reset_gpio(rc);
    return gpiod_set_value_cansleep(priv->reset, 0); /* logic 0 = RUNNING */
}
```

ملاحظة مهمة: الـ `gpiod_set_value_cansleep` مش `gpiod_set_value` العادية. الفرق:
- `gpiod_set_value()` — تُستخدم من context مش بيقدر ينام (interrupt handlers، atomic context)
- `gpiod_set_value_cansleep()` — تُستخدم لما الـ GPIO controller نفسه ممكن يحتاج يستنى (مثلاً GPIO expander على I2C — لازم يعمل I2C transaction، وده بيحتاج sleep)

الـ driver بيستخدم `_cansleep` عشان هو مش عارف الـ GPIO controller هيكون hardware GPIO مباشر أو GPIO expander على slow bus.

---

### الـ Active-High vs Active-Low — الـ GPIO Subsystem بيتكفل بيها

الـ `gpio_desc` بيشيل معلومات الـ polarity من الـ DT. لو الـ DT قال `GPIO_ACTIVE_LOW`، الـ GPIO subsystem بيعكس الـ value تلقائياً. يعني:

```
DT: reset-gpios = <&gpio3 14 GPIO_ACTIVE_LOW>;

gpiod_set_value(desc, 1)  →  الـ GPIO subsystem يشوف إن الـ line active-low
                          →  يحول 1 → 0 فيزيائياً على الـ pin
                          →  النتيجة: pin LOW = device IN RESET (active-low)
```

الـ reset-gpio driver **مش محتاج يعرف** الـ polarity — هو دايماً بيكتب 1 للـ assert و0 للـ deassert، والـ GPIO subsystem بيتعامل مع الـ polarity.

---

### إيه اللي الـ framework بيملكه vs إيه اللي بيفوّضه للـ driver

| المسؤولية | مين بيتعامل معاها |
|---|---|
| Lookup من الـ DT phandle لـ reset controller معين | **Framework** (core.c) |
| Reference counting على الـ shared reset lines | **Framework** |
| الـ API اللي الـ consumers بيشوفوها (`reset_control_assert` إلخ) | **Framework** |
| فعلاً عمل الـ assert / deassert على الـ hardware | **Driver** (reset-gpio.c) |
| ترجمة الـ DT specifier لـ ID عددي (`of_xlate`) | **Driver** |
| إدارة الـ GPIO resource (get/put) | **Driver** |
| الـ polarity handling (active-high vs active-low) | **GPIO Subsystem** (مش الـ driver ولا الـ framework) |
| الـ device lifecycle (probe/remove/devm cleanup) | **Driver** + devm infrastructure |

---

### ليه `nr_resets = 1` و `id` بيتتجاهل؟

الـ `reset-gpio.c` بيمثل حالة بسيطة جداً: **GPIO واحد = reset line واحد**. الـ `id` parameter في الـ ops functions بيتتجاهل لأن مفيش اختيار — عندنا reset واحد بس.

في المقابل، لو كان CRC بتاع SoC فيه 200 reset line، الـ `nr_resets = 200` والـ `id` بيحدد أنهي reset بالضبط (كل واحد بيتحكم بـ bit مختلفة في register مختلفة).

---

### ملخص تدفق الـ Data من الـ Consumer للـ Hardware

```
Consumer Driver
    │
    │  devm_reset_control_get(dev, "phy-reset")
    ▼
Reset Framework (core.c)
    │  يبحث في الـ DT عن "resets" property في الـ consumer node
    │  يجيب الـ phandle → يلاقي الـ reset_controller_dev المناسب
    │  يرجع struct reset_control
    ▼
Consumer يستدعي: reset_control_assert(rstc)
    │
    ▼
Framework يستدعي: rcdev->ops->assert(rcdev, id)
    │
    ▼
reset_gpio_assert(rc, id)
    │  rc_to_reset_gpio(rc)  ← container_of
    ▼
gpiod_set_value_cansleep(priv->reset, 1)
    │
    ▼
GPIO Subsystem
    │  يعمل polarity check → يحدد القيمة الفيزيائية
    ▼
GPIO Controller Driver
    │  يكتب في register الـ GPIO controller
    ▼
Physical GPIO Pin → RESET# line على الـ PCB → Device in RESET
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### مقدمة سريعة

الـ `reset-gpio.c` driver بسيط جداً — بيوفر **reset controller** واحد يتحكم في حالة reset لـ hardware عن طريق GPIO line واحد. بيشتغل فوق الـ **auxiliary bus**، ومش بيتسجل من DT مباشرة، لكن بياخد `of_phandle_args` من الـ parent driver كـ platform data.

---

### 0. الـ Flags, Enums, وخيارات الـ Config

#### `enum gpiod_flags` — بيتحدد وقت `devm_gpiod_get()`

| القيمة | الـ Value | المعنى |
|--------|-----------|--------|
| `GPIOD_ASIS` | `0` | متغيرش اتجاه الـ pin |
| `GPIOD_IN` | `BIT(0)` | input mode |
| `GPIOD_OUT_LOW` | `BIT(0)\|BIT(1)` | output، قيمة ابتدائية = 0 (deasserted لو active-high) |
| **`GPIOD_OUT_HIGH`** | `BIT(0)\|BIT(1)\|BIT(2)` | output، قيمة ابتدائية = 1 ← **ده اللي بيستخدمه الـ driver** |
| `GPIOD_OUT_LOW_OPEN_DRAIN` | `OUT_LOW \| BIT(3)` | open-drain low |
| `GPIOD_OUT_HIGH_OPEN_DRAIN` | `OUT_HIGH \| BIT(3)` | open-drain high |

> الـ driver بيطلب GPIO بـ `GPIOD_OUT_HIGH` يعني لحظة الـ probe الـ device بيكون في حالة **reset مفعّل** (سطر 1 = assert).

#### الـ `GPIOD_FLAGS_BIT_*` — البِتّات الداخلية

| الـ Bit | المعنى |
|--------|--------|
| `BIT(0)` `DIR_SET` | اتجاه الـ pin اتحدد |
| `BIT(1)` `DIR_OUT` | الـ pin output |
| `BIT(2)` `DIR_VAL` | القيمة الابتدائية = 1 |
| `BIT(3)` `OPEN_DRAIN` | open-drain mode |

#### الـ Config Options المهمة

| الـ Kconfig | الأثر |
|------------|-------|
| `CONFIG_RESET_CONTROLLER` | يفعّل الـ `devm_reset_controller_register()` — لو مش موجود الـ register بترجع 0 بدون ما تعمل حاجة |
| `CONFIG_GPIOLIB` | يفعّل كل دوال `gpiod_*` — لو مش موجود كل الدوال بترجع `-ENOSYS` |
| `CONFIG_OF` | لازم يكون موجود عشان الـ `of_phandle_args` و `of_node_put()` |

---

### 1. الـ Structs المهمة

#### `struct reset_gpio_priv`

الـ **private state** الوحيد للـ driver — بيعيش في الـ device-managed memory.

```c
struct reset_gpio_priv {
    struct reset_controller_dev rc;   /* embedded reset controller — لازم يكون الأول */
    struct gpio_desc *reset;          /* handle للـ GPIO line */
};
```

| الـ Field | النوع | الغرض |
|----------|-------|--------|
| `rc` | `struct reset_controller_dev` | الـ embedded controller — الـ reset core بيشوفه مباشرة |
| `reset` | `struct gpio_desc *` | opaque pointer للـ GPIO pin — الـ GPIO subsystem بيديره |

- الـ `rc` لازم يكون **أول field** عشان `container_of()` تشتغل صح في `rc_to_reset_gpio()`.
- الـ `gpio_desc` بيتحرر تلقائياً عند device teardown عن طريق `devm`.

---

#### `struct reset_controller_dev`

الـ **reset controller** المركزي اللي الـ reset core بيتعامل معاه.

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;      /* وظائف assert/deassert/status */
    struct module *owner;                      /* THIS_MODULE */
    struct list_head list;                     /* ربط في global list */
    struct list_head reset_control_head;       /* consumers بتوعه */
    struct device *dev;                        /* الـ device اللي بيملكه */
    struct device_node *of_node;               /* DT node للـ phandle matching */
    const struct of_phandle_args *of_args;     /* للـ GPIO-based controllers */
    int of_reset_n_cells;                      /* عدد cells في الـ specifier */
    int (*of_xlate)(...);                      /* ترجمة specifier → id */
    unsigned int nr_resets;                    /* عدد resets = 1 هنا */
};
```

| الـ Field | القيمة في الـ driver | الغرض |
|----------|---------------------|--------|
| `ops` | `&reset_gpio_ops` | بوينتر للـ callbacks |
| `owner` | `THIS_MODULE` | منع unload وقت الاستخدام |
| `dev` | `&adev->dev` | ربط بالـ auxiliary device |
| `of_args` | `platdata` | الـ phandle args من الـ parent |
| `of_node` | من `of_args->np` | الـ DT node للـ matching |
| `of_reset_n_cells` | `2` | cells في الـ reset specifier |
| `of_xlate` | `reset_gpio_of_xlate` | ترجع `args[0]` |
| `nr_resets` | `1` | controller واحد بـ reset واحد |

---

#### `struct reset_control_ops`

الـ **vtable** — الـ callbacks اللي الـ reset core بيستدعيها.

```c
static const struct reset_control_ops reset_gpio_ops = {
    .assert   = reset_gpio_assert,    /* GPIO → HIGH (1) */
    .deassert = reset_gpio_deassert,  /* GPIO → LOW  (0) */
    .status   = reset_gpio_status,    /* قراءة قيمة الـ GPIO */
};
```

| الـ Callback | الـ GPIO Value | المعنى الفيزيائي |
|-------------|---------------|-----------------|
| `.assert` | `1` (HIGH) | الـ device في حالة reset |
| `.deassert` | `0` (LOW) | الـ device شغّال عادي |
| `.status` | قراءة | `1` = في reset، `0` = شغال |

> ملاحظة: الـ `.reset` (self-deasserting) **مش موجود** هنا — الـ driver بيدعم manual assert/deassert فقط.

---

#### `struct auxiliary_device`

الـ bus device اللي الـ driver بيتربط بيه.

```c
struct auxiliary_device {
    struct device dev;      /* embedded kernel device */
    const char *name;       /* "gpio" → match name = "reset.gpio" */
    u32 id;
    struct { ... } sysfs;
};
```

الـ driver بيوصله عن طريق `adev` في `probe()` — بياخد منه `dev` و `platdata`.

---

#### `struct auxiliary_driver`

الـ **driver registration object**.

```c
static struct auxiliary_driver reset_gpio_driver = {
    .probe    = reset_gpio_probe,
    .id_table = reset_gpio_ids,       /* matches "reset.gpio" */
    .driver   = {
        .name = "reset-gpio",
        .suppress_bind_attrs = true,  /* مش بيعمل sysfs bind/unbind */
    },
};
```

---

#### `struct of_phandle_args`

بيجي كـ platform data من الـ parent driver — مش بيتعرّف في الـ file ده لكنه محوري.

```c
/* من include/linux/of.h */
struct of_phandle_args {
    struct device_node *np;   /* الـ DT node للـ reset provider */
    int args_count;
    uint32_t args[MAX_PHANDLE_ARGS];  /* args[0] = reset line id */
};
```

الـ driver بيحطه في `rc.of_args` عشان الـ reset core يعرف يعمل phandle matching.

---

### 2. مخطط علاقات الـ Structs

```
┌─────────────────────────────────────────────┐
│            struct reset_gpio_priv            │
│  ┌────────────────────────────────────────┐  │
│  │      struct reset_controller_dev rc   │  │
│  │  ops ──────────────────────────────►  │  │
│  │  dev ──────────────────────────────►  │  │
│  │  of_args ──────────────────────────►  │  │
│  │  of_node ──────────────────────────►  │  │
│  │  of_xlate() = reset_gpio_of_xlate    │  │
│  │  nr_resets  = 1                      │  │
│  └────────────────────────────────────────┘  │
│  reset ─────────────────────────────────►    │
└─────────────────────────────────────────────┘
        │ops                │reset          │dev/of_args
        ▼                   ▼               ▼
┌──────────────────┐  ┌──────────┐  ┌──────────────────────┐
│reset_control_ops │  │gpio_desc │  │  auxiliary_device    │
│ .assert()        │  │(opaque)  │  │  ┌────────────────┐  │
│ .deassert()      │  │GPIO HW   │  │  │  struct device │  │
│ .status()        │  └──────────┘  │  └────────────────┘  │
└──────────────────┘                │  platdata ──────────► │
                                    │  of_phandle_args      │
                                    └──────────────────────┘

         reset core (kernel)
              │
              │  devm_reset_controller_register()
              ▼
     ┌─────────────────────┐
     │  global rcdev list  │◄── rc.list (list_head)
     └─────────────────────┘
```

---

### 3. مخطط دورة الحياة (Lifecycle)

```
MODULE LOAD
    │
    ▼
module_auxiliary_driver(reset_gpio_driver)
    │
    ├─► auxiliary_driver_register()
    │       └─► bus يسجّل الـ driver
    │
    ▼
DEVICE MATCH  (parent يسجّل auxiliary_device باسم "reset.gpio")
    │
    ▼
reset_gpio_probe(adev, id)
    │
    ├─ [1] تحقق من platdata → -EINVAL لو مفيش
    │
    ├─ [2] devm_kzalloc() → alloc reset_gpio_priv
    │
    ├─ [3] auxiliary_set_drvdata(adev, &priv->rc)
    │       └─ يخزن &rc في dev->driver_data
    │
    ├─ [4] devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH)
    │       └─ يطلب GPIO، يحطه HIGH (assert reset)
    │
    ├─ [5] تعبئة priv->rc (ops, owner, dev, of_args, ...)
    │
    ├─ [6] devm_add_action_or_reset(dev, reset_gpio_of_node_put, of_node)
    │       └─ يضمن of_node_put() عند الـ teardown
    │
    └─ [7] devm_reset_controller_register(dev, &priv->rc)
            └─ يضيف rc في global list → controller جاهز للاستخدام
    │
    ▼
RUNTIME
    │  consumer calls reset_control_assert()
    │      → reset_gpio_assert() → gpiod_set_value_cansleep(reset, 1)
    │
    │  consumer calls reset_control_deassert()
    │      → reset_gpio_deassert() → gpiod_set_value_cansleep(reset, 0)
    │
    │  consumer calls reset_control_status()
    │      → reset_gpio_status() → gpiod_get_value_cansleep(reset)
    │
    ▼
TEARDOWN  (device removed أو module unload)
    │
    ├─ devm يشغّل cleanup بالعكس:
    │   ├─ reset_controller_unregister(&priv->rc)  ← devm action
    │   ├─ reset_gpio_of_node_put(of_node)          ← devm action
    │   ├─ gpiod_put(priv->reset)                   ← devm action
    │   └─ kfree(priv)                              ← devm_kzalloc
    │
    ▼
MODULE UNLOAD
    │
    └─ auxiliary_driver_unregister()
```

---

### 4. مخططات تدفق الاستدعاء (Call Flow)

#### أ. تدفق الـ Assert (وضع الـ device في reset)

```
consumer driver
    │
    └─► reset_control_assert(rstc)
            │  [reset core validates rstc]
            │
            └─► rcdev->ops->assert(rcdev, id)
                    │  [= reset_gpio_assert()]
                    │
                    └─► rc_to_reset_gpio(rc)
                            │  [container_of → priv]
                            │
                            └─► gpiod_set_value_cansleep(priv->reset, 1)
                                    │  [GPIO subsystem]
                                    │
                                    └─► GPIO hardware register → PIN = HIGH
                                                              → Device in RESET
```

#### ب. تدفق الـ Deassert (تشغيل الـ device)

```
consumer driver
    │
    └─► reset_control_deassert(rstc)
            │
            └─► rcdev->ops->deassert(rcdev, id)
                    │  [= reset_gpio_deassert()]
                    │
                    └─► gpiod_set_value_cansleep(priv->reset, 0)
                                    │
                                    └─► GPIO hardware register → PIN = LOW
                                                              → Device RUNNING
```

#### ج. تدفق الـ Status Query

```
consumer driver
    │
    └─► reset_control_status(rstc)
            │
            └─► rcdev->ops->status(rcdev, id)
                    │  [= reset_gpio_status()]
                    │
                    └─► gpiod_get_value_cansleep(priv->reset)
                                    │
                                    └─► قراءة GPIO register
                                        returns: 1 = in reset
                                                 0 = running
```

#### د. تدفق الـ of_xlate (ترجمة DT specifier)

```
reset core يحاول يعمل match لـ phandle
    │
    └─► rcdev->of_xlate(rcdev, reset_spec)
            │  [= reset_gpio_of_xlate()]
            │
            └─► return reset_spec->args[0]
                    └─ يرجع الـ reset line id من الـ DT args
                       (عادةً 0 لأن nr_resets = 1)
```

#### هـ. تدفق الـ Probe الكامل مع error handling

```
reset_gpio_probe(adev, id)
    │
    ├─ !platdata ? ──────────────────────────────► return -EINVAL
    │
    ├─ devm_kzalloc() failed ? ──────────────────► return -ENOMEM
    │
    ├─ devm_gpiod_get() failed ?
    │       └─ IS_ERR(priv->reset) ─────────────► return dev_err_probe(...)
    │                                               [يطبع error ويرجع errno]
    │
    ├─ devm_add_action_or_reset() failed ? ──────► return ret
    │       [devm تعمل cleanup للي اتعمل قبل كده]
    │
    └─ devm_reset_controller_register() failed ? ► return ret
            └─ نجح → 0 → controller مسجّل
```

---

### 5. استراتيجية الـ Locking

#### الـ driver ده **مفيش فيه locks صريحة** — ليه؟

لأن الـ driver اتعمل بشكل deliberate ليكون **stateless من ناحية الـ locking**:

| الطبقة | اللي بيعمل الـ locking | الآلية |
|--------|----------------------|--------|
| **Reset core** | بيحمي الـ `rcdev->list` و `reset_control_head` | internal spinlock في `drivers/reset/core.c` |
| **GPIO subsystem** | بيحمي الـ `gpio_desc` | per-descriptor lock في `gpio/gpiolib.c` |
| **devm framework** | بيحمي الـ cleanup actions list | `device->devres_lock` (spinlock) |
| **Auxiliary bus** | بيحمي الـ device lifecycle | `device->mutex` في الـ driver model |

#### تفاصيل الـ sleeping context

الـ driver بيستخدم `gpiod_set_value_cansleep()` و `gpiod_get_value_cansleep()` بدلاً من النسخ العادية — ده مهم:

```
gpiod_set_value()        ← atomic context only (لو GPIO controller fast)
gpiod_set_value_cansleep() ← يقدر يـsleep (لو GPIO عبر I2C/SPI مثلاً)
```

الـ reset ops بتاعته **مش بتتحاسب على أي lock** — الـ caller (reset core) هو المسؤول عن serialization لو محتاج. الـ GPIO descriptor نفسه protected من جوه.

#### ترتيب الـ Locks (Lock Ordering) في السياق الأشمل

```
reset_core_lock  (في core.c)
    └─► GPIO subsystem lock  (في gpiolib.c)
            └─► hardware I/O
```

> مفيش deadlock risk هنا لأن الـ driver نفسه مش بيمسك أي lock — كل الـ locking في الطبقات التحتية.

---

### ملخص العلاقات بالشكل الكامل

```
auxiliary_bus
    │
    │  matches "reset.gpio"
    ▼
reset_gpio_driver (auxiliary_driver)
    │
    │  .probe()
    ▼
reset_gpio_priv  ◄──────────────────────────── devm memory (auto-freed)
  │
  ├── rc (reset_controller_dev)
  │     ├── ops → reset_gpio_ops (assert/deassert/status)
  │     ├── dev → auxiliary_device.dev
  │     ├── of_args → of_phandle_args (from platdata)
  │     ├── of_node → DT node (ref-counted, put on teardown)
  │     ├── of_xlate → reset_gpio_of_xlate
  │     ├── nr_resets = 1
  │     └── list ─────────────────────────────► global rcdev_list (reset core)
  │
  └── reset (gpio_desc*)
        └──────────────────────────────────────► GPIO hardware PIN
                                                  HIGH = device in reset
                                                  LOW  = device running
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### جميع الـ Functions في الملف

| Function | النوع | الغرض |
|---|---|---|
| `rc_to_reset_gpio` | Helper / Accessor | تحويل `reset_controller_dev *` لـ `reset_gpio_priv *` |
| `reset_gpio_assert` | Reset Control Op | تفعيل الـ reset (يرفع الـ GPIO) |
| `reset_gpio_deassert` | Reset Control Op | إلغاء الـ reset (يخفض الـ GPIO) |
| `reset_gpio_status` | Reset Control Op | قراءة حالة الـ reset line |
| `reset_gpio_of_xlate` | OF Translation | ترجمة الـ phandle args لـ reset ID |
| `reset_gpio_of_node_put` | Cleanup Callback | تحرير الـ `of_node` reference عند الـ device removal |
| `reset_gpio_probe` | Driver Probe | تهيئة الـ driver وتسجيل الـ reset controller |

#### الـ External APIs المستخدمة

| API | Header | الغرض |
|---|---|---|
| `container_of` | `<linux/kernel.h>` | الوصول لـ enclosing struct من member pointer |
| `gpiod_set_value_cansleep` | `<linux/gpio/consumer.h>` | كتابة قيمة على GPIO (sleeping-safe) |
| `gpiod_get_value_cansleep` | `<linux/gpio/consumer.h>` | قراءة قيمة GPIO (sleeping-safe) |
| `devm_gpiod_get` | `<linux/gpio/consumer.h>` | الحصول على GPIO descriptor مع devm |
| `dev_get_platdata` | `<linux/device.h>` | قراءة platform data من device |
| `devm_kzalloc` | `<linux/device.h>` | تخصيص ذاكرة مع devm |
| `auxiliary_set_drvdata` | `<linux/auxiliary_bus.h>` | حفظ driver data على الـ auxiliary device |
| `devm_add_action_or_reset` | `<linux/device.h>` | تسجيل cleanup callback مع devm |
| `of_node_put` | `<linux/of.h>` | تحرير reference count على الـ device tree node |
| `devm_reset_controller_register` | `<linux/reset-controller.h>` | تسجيل الـ reset controller مع devm |
| `dev_err_probe` | `<linux/device.h>` | طباعة error وإرجاع error code |

---

### المجموعة 1: Accessor / Helper Functions

هذه المجموعة تحتوي على functions مساعدة تُستخدم داخلياً للتنقل بين الـ structs. وظيفتها تحويل الـ pointer من الـ abstract layer (`reset_controller_dev`) إلى الـ private driver struct الذي يحتوي على كل البيانات الفعلية.

---

#### `rc_to_reset_gpio`

```c
static inline struct reset_gpio_priv
*rc_to_reset_gpio(struct reset_controller_dev *rc)
{
    return container_of(rc, struct reset_gpio_priv, rc);
}
```

**الوظيفة:** الـ reset framework يعمل مع `reset_controller_dev *` فقط. هذه الـ function تحوّل الـ pointer ده لـ `reset_gpio_priv *` الذي يحتوي على الـ `gpio_desc *` الفعلي. تستخدم `container_of` اللي بتحسب offset الـ `rc` member داخل الـ struct وترجع الـ pointer للـ struct الأكبر.

**Parameters:**
- `rc` — pointer لـ `reset_controller_dev` المدمج داخل `reset_gpio_priv`

**Return Value:** pointer لـ `reset_gpio_priv` المحتوي على الـ `rc` المعطى.

**Key Details:** الـ `inline` يمنع overhead الـ function call. لا يوجد locking — مجرد pointer arithmetic آمنة طالما الـ struct حي في الذاكرة. لا error path لأن `container_of` لا تفشل إذا كان الـ pointer صحيحاً.

**Who Calls It:** جميع الـ reset control ops (`assert`, `deassert`, `status`) تستدعيها كأول خطوة.

---

### المجموعة 2: Reset Control Operations (Ops)

هذه المجموعة هي قلب الـ driver. الـ reset framework يستدعيها عبر الـ `reset_control_ops` vtable. كل function تأخذ الـ `reset_controller_dev *` وتتحول عبر `rc_to_reset_gpio` للوصول للـ GPIO descriptor.

الـ `id` parameter موجود في الـ signature لأن الـ framework يدعم controllers تتحكم في أكثر من reset line، لكن هذا الـ driver يدعم line واحدة فقط (`nr_resets = 1`)، فـ `id` يُتجاهل.

---

#### `reset_gpio_assert`

```c
static int reset_gpio_assert(struct reset_controller_dev *rc, unsigned long id)
{
    struct reset_gpio_priv *priv = rc_to_reset_gpio(rc);
    return gpiod_set_value_cansleep(priv->reset, 1);
}
```

**الوظيفة:** تضع الجهاز في حالة reset عن طريق رفع الـ GPIO line للقيمة logical-1. الـ GPIO descriptor يعالج الـ active-low polarity تلقائياً، يعني لو الـ pin معرّف في الـ DT كـ `active-low`، الـ GPIOLIB ستعكس القيمة وتنزّل الـ pin فيزيائياً.

**Parameters:**
- `rc` — pointer للـ reset controller الخاص بهذا الـ driver
- `id` — رقم الـ reset line (مهمل هنا، دايماً 0)

**Return Value:** `0` عند النجاح، قيمة سالبة من الـ GPIO driver عند الفشل.

**Key Details:** استخدام `gpiod_set_value_cansleep` بدل `gpiod_set_value` يعني الـ function مقدرة تُستدعى من sleeping context (process context) وقد تُحدث sleep داخلياً لو الـ GPIO controller يحتاج I2C أو SPI للوصول. لا يمكن استدعاؤها من atomic context أو interrupt handler.

**Who Calls It:** الـ reset framework core عبر `reset_control_assert()` من أي driver يملك handle على هذا الـ reset line.

---

#### `reset_gpio_deassert`

```c
static int reset_gpio_deassert(struct reset_controller_dev *rc,
                               unsigned long id)
{
    struct reset_gpio_priv *priv = rc_to_reset_gpio(rc);
    return gpiod_set_value_cansleep(priv->reset, 0);
}
```

**الوظيفة:** تُخرج الجهاز من حالة reset بوضع الـ GPIO على logical-0. مكمّلة `reset_gpio_assert` تماماً — نفس الـ polarity handling ينطبق.

**Parameters:**
- `rc` — pointer للـ reset controller
- `id` — رقم الـ reset line (مهمل)

**Return Value:** `0` عند النجاح، قيمة سالبة عند الفشل.

**Key Details:** عند استدعاء `reset_gpio_probe`، الـ GPIO يُهيَّأ بـ `GPIOD_OUT_HIGH`، يعني الجهاز يبدأ في حالة reset (asserted). الـ deassert هو الخطوة التالية اللي يجب أن تحدث لتشغيل الجهاز.

**Who Calls It:** الـ reset framework core عبر `reset_control_deassert()` أو `reset_control_reset()`.

---

#### `reset_gpio_status`

```c
static int reset_gpio_status(struct reset_controller_dev *rc, unsigned long id)
{
    struct reset_gpio_priv *priv = rc_to_reset_gpio(rc);
    return gpiod_get_value_cansleep(priv->reset);
}
```

**الوظيفة:** تقرأ الحالة الحالية للـ reset line. ترجع القيمة logical للـ GPIO — 1 يعني الجهاز في reset، 0 يعني خارج reset. الـ GPIOLIB تتولى الـ polarity inversion تلقائياً.

**Parameters:**
- `rc` — pointer للـ reset controller
- `id` — رقم الـ reset line (مهمل)

**Return Value:** `1` لو الـ reset مُفعَّل (asserted)، `0` لو مُلغى (deasserted)، قيمة سالبة عند خطأ في القراءة.

**Key Details:** الـ `cansleep` مهم هنا أيضاً لأن GPIO controllers الموجودة على I2C/SPI buses تحتاج sleeping context للقراءة. الـ reset framework يستخدم هذه الـ function لتنفيذ `reset_control_status()`.

**Who Calls It:** الـ reset framework core عبر `reset_control_status()` من أي consumer يريد التحقق من حالة الـ reset line.

---

### المجموعة 3: OF (Device Tree) Translation

الـ reset framework يستخدم الـ phandle args من الـ DT لتحديد الـ reset line المطلوبة. هذه الـ function تترجم الـ args لـ ID رقمي يُمرر للـ ops.

---

#### `reset_gpio_of_xlate`

```c
static int reset_gpio_of_xlate(struct reset_controller_dev *rcdev,
                               const struct of_phandle_args *reset_spec)
{
    return reset_spec->args[0];
}
```

**الوظيفة:** تترجم الـ `of_phandle_args` القادمة من الـ device tree لـ reset ID رقمي. في هذا الـ driver، الـ ID هو مجرد `args[0]` مباشرة. الـ translation بسيطة لأن الـ driver يدعم reset واحد فقط وقيمة الـ arg لا تُستخدم فعلاً في الـ ops.

**Parameters:**
- `rcdev` — pointer للـ reset controller (غير مستخدم هنا)
- `reset_spec` — الـ phandle args من الـ DT، تحتوي على `args[]` و `args_count`

**Return Value:** قيمة `args[0]` كـ reset ID، أو قيمة سالبة لو كان هناك خطأ في الـ args count (لكن هنا لا يوجد validation — يُرجع مباشرة).

**Key Details:** الـ `of_reset_n_cells = 2` يُحدد أن الـ phandle في الـ DT يأخذ خليتين (cell)، لكن التعليق في الـ source يوضح أن هذا فقط "لمطابقة الـ GPIO specifier" وليس له استخدام حقيقي. الـ `args[0]` هو GPIO number والـ `args[1]` هو الـ flags في الـ GPIO specifier.

**Who Calls It:** الـ reset framework core عند resolve الـ phandle من الـ DT للحصول على reset ID.

---

### المجموعة 4: Cleanup / Teardown Functions

هذه الـ function مسؤولة عن تحرير الموارد عند إزالة الـ device. تُسجَّل كـ devm action لضمان تنفيذها تلقائياً.

---

#### `reset_gpio_of_node_put`

```c
static void reset_gpio_of_node_put(void *data)
{
    of_node_put(data);
}
```

**الوظيفة:** تُحرر الـ reference count على الـ device tree node (`struct device_node`). الـ `of_args->np` (الـ `of_node` في الـ `reset_controller_dev`) يُمسك reference على الـ DT node — هذه الـ function تُعيد هذا الـ reference عند تفكيك الـ driver.

**Parameters:**
- `data` — pointer لـ `struct device_node *` يُمرر عبر `devm_add_action_or_reset` كـ opaque `void *`

**Return Value:** لا يوجد (void).

**Key Details:** الـ `of_node_put` تُنقص الـ refcount للـ DT node. لو الـ refcount وصل صفر، تُحرر الذاكرة. الـ wrapping في هذه الـ function ضروري لأن `devm_add_action_or_reset` تقبل `void (*)(void *)` signature فقط. في حالة فشل الـ probe، الـ devm framework يستدعيها تلقائياً لتنظيف ما تم allocate-ه.

**Who Calls It:** يُستدعى تلقائياً بواسطة الـ devm framework عند device removal أو عند فشل الـ probe بعد تسجيله.

---

### المجموعة 5: Driver Lifecycle — Probe

هذه هي الـ function الأساسية التي تُهيئ كل شيء. لأن الـ driver يعمل على الـ auxiliary bus، يتلقى `auxiliary_device *` بدلاً من `platform_device *`.

---

#### `reset_gpio_probe`

```c
static int reset_gpio_probe(struct auxiliary_device *adev,
                            const struct auxiliary_device_id *id)
```

**الوظيفة:** تُهيئ الـ `reset_gpio_priv`، تحصل على الـ GPIO descriptor، تملأ الـ `reset_controller_dev`، وتسجّله مع الـ reset framework. كل التخصيصات تتم عبر devm لضمان cleanup تلقائي عند الـ removal.

**Parameters:**
- `adev` — الـ auxiliary device المرتبط بهذا الـ driver، يحمل `platform_data` هو `of_phandle_args *`
- `id` — الـ device ID المطابق من `reset_gpio_ids` (غير مستخدم داخلياً)

**Return Value:** `0` عند نجاح التسجيل، قيم سالبة عند الفشل: `-EINVAL` لو لا يوجد platform data، `-ENOMEM` لو فشل الـ allocation، قيمة من `PTR_ERR` لو فشل GPIO acquire، قيمة من `devm_add_action_or_reset` أو `devm_reset_controller_register` لو فشل التسجيل.

**Key Details:**
- لا يوجد explicit locking — الـ probe يعمل في process context وكل devm allocations آمنة.
- الـ GPIO يُهيَّأ كـ `GPIOD_OUT_HIGH` مما يعني الجهاز يبدأ في reset state — هذا السلوك المطلوب لمعظم الـ hardware.
- `auxiliary_set_drvdata(adev, &priv->rc)` يحفظ pointer للـ `rc` وليس للـ `priv` مباشرة — الـ consumers يمكنهم الوصول لـ `priv` عبر `rc_to_reset_gpio`.
- `of_reset_n_cells = 2` و `of_xlate = reset_gpio_of_xlate` يُعرّفان كيف يُفسَّر الـ phandle من الـ DT.
- `nr_resets = 1` يُخبر الـ framework أن هذا الـ controller يدعم reset line واحدة فقط.

**Who Calls It:** الـ auxiliary bus core عند match بين device و driver.

**Pseudocode Flow:**

```
reset_gpio_probe(adev, id):
    dev = &adev->dev

    // 1. التحقق من platform data
    platdata = dev_get_platdata(dev)
    if !platdata → return -EINVAL

    // 2. تخصيص private state
    priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL)
    if !priv → return -ENOMEM

    // 3. حفظ driver data
    auxiliary_set_drvdata(adev, &priv->rc)

    // 4. الحصول على GPIO (يبدأ HIGH = reset asserted)
    priv->reset = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH)
    if IS_ERR(priv->reset) → return dev_err_probe(...)

    // 5. ملء reset_controller_dev
    priv->rc.ops    = &reset_gpio_ops
    priv->rc.owner  = THIS_MODULE
    priv->rc.dev    = dev
    priv->rc.of_args = platdata

    // 6. تسجيل cleanup للـ DT node reference
    ret = devm_add_action_or_reset(dev, reset_gpio_of_node_put,
                                   priv->rc.of_node)
    if ret → return ret

    // 7. ضبط OF translation
    priv->rc.of_reset_n_cells = 2
    priv->rc.of_xlate = reset_gpio_of_xlate
    priv->rc.nr_resets = 1

    // 8. تسجيل الـ reset controller
    return devm_reset_controller_register(dev, &priv->rc)
```

---

### ملاحظة على بنية الـ Driver

الـ driver لا يملك `remove` function صريحة لأن كل شيء مُدار عبر **devm**:

```
devm_kzalloc          → تحرير التاني
devm_gpiod_get        → gpiod_put تلقائياً
devm_add_action_or_reset → reset_gpio_of_node_put تلقائياً
devm_reset_controller_register → unregister تلقائياً
```

الـ `suppress_bind_attrs = true` في الـ `driver` struct يمنع ظهور الـ `bind`/`unbind` sysfs attributes، وهذا منطقي لأن الـ driver لا يُدعم إعادة binding يدوياً في هذا السياق.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs entries

**الـ** `reset-gpio` driver بيشتغل فوق **reset controller subsystem** وكمان فوق **GPIO subsystem**، الاتنين عندهم entries في debugfs.

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# شوف كل الـ reset controllers المسجلين
ls /sys/kernel/debug/reset/

# اقرأ state الـ reset lines
cat /sys/kernel/debug/reset/<device-name>/reset_count

# GPIO debugfs — بيوري كل الـ GPIO descriptors المطلوبة
cat /sys/kernel/debug/gpio
```

مثال على output من `cat /sys/kernel/debug/gpio`:

```
gpiochip0: GPIOs 0-31, parent: platform/gpio-controller, gpio-controller:
 gpio-5  (                    |reset               ) out hi ACTIVE LOW
```

الـ `reset` هو الـ consumer name اللي اتحط في `devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH)` — لو مش شايفه فالـ GPIO مش اتطلبش صح.

```bash
# شوف الـ auxiliary bus devices
ls /sys/bus/auxiliary/devices/

# ابحث عن الـ reset.gpio device بالاسم
ls /sys/bus/auxiliary/devices/ | grep reset
```

---

#### 2. sysfs entries

| المسار | المحتوى |
|--------|---------|
| `/sys/bus/auxiliary/devices/reset.gpio.<N>/` | الـ device directory |
| `/sys/bus/auxiliary/devices/reset.gpio.<N>/driver` | symlink للـ driver |
| `/sys/bus/auxiliary/drivers/reset-gpio/` | الـ driver directory |
| `/sys/class/gpio/` | الـ GPIO pins المـ export (لو اتعمل export يدوي) |

```bash
# تأكد إن الـ driver bind صح
ls -la /sys/bus/auxiliary/devices/reset.gpio.0/driver
# المفروض يكون symlink لـ /sys/bus/auxiliary/drivers/reset-gpio

# شوف platform data الـ device
cat /sys/bus/auxiliary/devices/reset.gpio.0/modalias
```

---

#### 3. ftrace — tracepoints وـ events

**الـ** `reset-gpio` بيستخدم GPIO functions وـ reset controller core، فيه tracepoints مفيدة في الاتنين.

```bash
# فعّل tracing على GPIO subsystem
cd /sys/kernel/debug/tracing

# شوف الـ events المتاحة
grep -r "gpio\|reset" available_events

# فعّل GPIO set events
echo 1 > events/gpio/enable

# أو بالاختيار للـ events المهمة
echo 1 > events/gpio/gpio_value/enable

# trace الـ auxiliary bus probe
echo 1 > events/auxiliary_bus/enable

# ابدأ الـ trace
echo 1 > tracing_on
# ... شغّل/اقلع الـ device ...
cat trace
echo 0 > tracing_on
```

**ftrace بـ function tracing للـ reset-gpio functions:**

```bash
cd /sys/kernel/debug/tracing
echo function > current_tracer
echo 'reset_gpio_*' > set_ftrace_filter
echo 'gpiod_set_value_cansleep' >> set_ftrace_filter
echo 'gpiod_get_value_cansleep' >> set_ftrace_filter
echo 1 > tracing_on
cat trace
```

مثال output:

```
  kworker/0:1-45    [000] ..... 1234.567890: reset_gpio_assert <-reset_control_assert
  kworker/0:1-45    [000] ..... 1234.567891: gpiod_set_value_cansleep <-reset_gpio_assert
```

---

#### 4. printk وـ dynamic debug

**الـ** driver بيستخدم `dev_err_probe` بس — مفيش verbose logging افتراضي. تقدر تفعّل dynamic debug على GPIO وـ reset core:

```bash
# فعّل debug messages لـ reset controller core
echo 'module reset_controller +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug messages لـ GPIO core
echo 'module gpiolib +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لـ auxiliary bus
echo 'module auxiliary +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل كل debug messages في driver معين بـ file
echo 'file drivers/reset/reset-gpio.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
```

الـ flags في `+pflmt`:
- `p` = print the message
- `f` = include function name
- `l` = include line number
- `m` = include module name
- `t` = include thread id

```bash
# شوف الـ dynamic debug controls الفعّالة دلوقتي
grep reset /sys/kernel/debug/dynamic_debug/control
```

---

#### 5. Kernel config options للـ debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_DEBUG_GPIO` | يفعّل verbose GPIO debugging في kernel log |
| `CONFIG_GPIOLIB` | لازم يكون enabled عشان الـ driver يشتغل |
| `CONFIG_RESET_CONTROLLER` | لازم enabled — بدونه الـ driver stub |
| `CONFIG_GPIO_SYSFS` | يعمل expose للـ GPIO عبر sysfs |
| `CONFIG_DEBUG_FS` | يفعّل debugfs |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug` / `dev_dbg` runtime |
| `CONFIG_AUXILIARY_BUS` | لازم enabled للـ auxiliary_device |
| `CONFIG_KALLSYMS` | يفيد في قراءة stack traces |
| `CONFIG_DEBUG_KERNEL` | يفعّل مجموعة من debug options |
| `CONFIG_PROVE_LOCKING` | يكتشف lockdep violations |

```bash
# تحقق من الـ configs في kernel الشغال
zcat /proc/config.gz | grep -E 'CONFIG_DEBUG_GPIO|CONFIG_RESET_CONTROLLER|CONFIG_GPIOLIB|CONFIG_AUXILIARY'
```

---

#### 6. devlink وـ tools خاصة بالـ subsystem

```bash
# gpioinfo — يوري كل الـ GPIO chips والـ lines (من libgpiod)
gpioinfo

# gpioget — اقرأ قيمة GPIO بدون driver
gpioget gpiochip0 5

# gpioset — اكتب قيمة GPIO يدوياً (اتحرس — ممكن يكسر device في reset)
gpioset gpiochip0 5=1   # assert reset
gpioset gpiochip0 5=0   # deassert reset

# lsgpio — نسخة أبسط
lsgpio

# شوف الـ reset controllers المسجلة عبر /proc
cat /proc/reset_controllers 2>/dev/null || echo "مش موجود — شوف debugfs"
```

---

#### 7. جدول رسائل الـ errors الشائعة

| رسالة في الـ kernel log | المعنى | الحل |
|------------------------|--------|-------|
| `Could not get reset gpios` | `devm_gpiod_get` فشل | تحقق من الـ DT: `reset-gpios` موجود وصح؟ GPIO pin مش مستخدم؟ |
| `probe with driver reset-gpio failed with error -EINVAL` | `platdata` بـ NULL — الـ auxiliary device مش بيحمل `of_phandle_args` | الـ parent driver مش بيسيب الـ platform data صح |
| `probe with driver reset-gpio failed with error -ENOMEM` | `devm_kzalloc` فشل | نادر — الـ system low memory |
| `probe with driver reset-gpio failed with error -EPROBE_DEFER` | GPIO controller مش جاهز | انتظر — الـ kernel هيعيد الـ probe لوحده |
| `gpiod_set_value_cansleep: gpio-X (reset) is not an output` | الـ GPIO pin مش output | تحقق من الـ pinctrl configuration في DT |
| `gpio-X (reset): Error in cansleep check` | مشكلة في الـ GPIO controller | تحقق من GPIO controller driver |
| `reset-gpio reset.gpio.0: devm_add_action_or_reset failed` | فشل تسجيل cleanup action لـ `of_node` | نادر — مشكلة memory أو device lifecycle |

---

#### 8. أماكن استراتيجية لـ `dump_stack()` وـ `WARN_ON()`

```c
static int reset_gpio_probe(struct auxiliary_device *adev,
                            const struct auxiliary_device_id *id)
{
    struct of_phandle_args *platdata = dev_get_platdata(dev);

    /* WARN لو platdata فاضي — المفروض Parent يملاه */
    WARN_ON(!platdata);

    priv->reset = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
    if (IS_ERR(priv->reset)) {
        /* dump_stack هنا يوري من استدعى الـ probe */
        dump_stack();
        return dev_err_probe(...);
    }
    ...
}

static int reset_gpio_assert(struct reset_controller_dev *rc, unsigned long id)
{
    struct reset_gpio_priv *priv = rc_to_reset_gpio(rc);

    /* WARN لو priv->reset NULL بشكل غير متوقع */
    WARN_ON(!priv->reset);

    return gpiod_set_value_cansleep(priv->reset, 1);
}
```

---

### Hardware Level

#### 1. التحقق إن الـ hardware state يطابق الـ kernel state

الـ driver بيتحكم في reset عبر GPIO — القيمة الـ logical `1` في `reset_gpio_assert` تعني "assert reset" لكن القيمة الـ physical على الـ pin بتعتمد على الـ `active-low` flag في الـ DT.

```bash
# اقرأ القيمة الـ logical (بعد active-low compensation)
cat /sys/kernel/debug/gpio | grep reset
# مثال: gpio-5 (reset) out hi ACTIVE LOW
# "hi" = logical high = assert reset = physical LOW على الـ pin

# تحقق من الـ reset status عبر kernel API
# (لو عندك test module)
# reset_control_status() بيرجع نتيجة gpiod_get_value_cansleep
```

المطابقة:

| `gpiod_set_value_cansleep` | ACTIVE LOW في DT | القيمة الـ physical على الـ pin |
|---------------------------|------------------|-------------------------------|
| `1` (assert) | نعم | LOW (0V) |
| `1` (assert) | لا | HIGH (3.3V/1.8V) |
| `0` (deassert) | نعم | HIGH (3.3V/1.8V) |
| `0` (deassert) | لا | LOW (0V) |

---

#### 2. Register dump techniques

الـ `reset-gpio` driver بيتحكم في reset عبر GPIO controller — مش بيكلم hardware registers مباشرة. الـ registers الحقيقية هي في GPIO controller.

```bash
# عرّف base address للـ GPIO controller من DT
cat /proc/device-tree/gpio-controller@<addr>/reg | xxd

# استخدم devmem2 تقرأ GPIO data register
devmem2 0x<GPIO_BASE + DATA_OFFSET> b

# مثال على Raspberry Pi (BCM2835) - GPIO level register
devmem2 0x3F200034 w   # GPLEV0 — يوري state كل GPIO 0-31

# مثال على i.MX — GPIO data register
devmem2 0x30200000 w   # GPIO1 DR register

# io utility (من package i2c-tools أو مستقل)
io -4 0x<GPIO_BASE>
```

---

#### 3. Logic Analyzer / Oscilloscope

**الاتصال:**

```
Device Under Test (DUT)
        │
   [RESET PIN] ──────────────────── Logic Analyzer CH0
        │                                    │
       GND ─────────────────────────────── GND
```

**الإعدادات المقترحة:**

- **Sample rate**: 10 MHz أو أكتر (كافي للـ GPIO reset pulses)
- **Trigger**: falling edge على الـ reset pin (لو active-low)
- **Protocol decoder**: مش لازم — الـ reset signal بسيط
- **Voltage threshold**: 1.65V لـ 3.3V signals، 0.9V لـ 1.8V

**ما تبصله:**

1. تأكد إن الـ reset pulse عنده الـ width المطلوب للـ device (مثلاً: ≥ 100ns لبعض الـ ICs)
2. لو الـ reset line بيبقى LOW طول الوقت → الـ driver مش بيعمل deassert صح
3. لو الـ reset line ما بتتحركش → GPIO controller issue أو pinmux مش صح

---

#### 4. Common hardware issues → kernel log patterns

| المشكلة | الـ kernel log pattern | السبب |
|---------|----------------------|-------|
| الـ GPIO pin مش configured | `Could not get reset gpios` + `-ENOENT` | `reset-gpios` مش موجود في DT |
| الـ GPIO pin في use | `Could not get reset gpios` + `-EBUSY` | driver تاني طالب نفس الـ pin |
| الـ GPIO controller مش جاهز | `Could not get reset gpios` + `-EPROBE_DEFER` | GPIO driver لسه مش loaded |
| خطأ polarity | Device مش بيرجع من reset | `active-low` مش صح في DT |
| الـ reset line مش موصول | Device بيتهنج | تحقق بـ oscilloscope |
| voltage mismatch | GPIO read دايماً 0 أو 1 | 3.3V GPIO على 1.8V device |

---

#### 5. Device Tree debugging

الـ DT entry المتوقع للـ `reset-gpio` driver (بيتكون من parent driver زي `reset-simple`):

```dts
/* parent node — بيعرّف الـ reset GPIOs */
some_device: some-device {
    compatible = "vendor,some-device";
    resets = <&reset_ctrl 0>;
};

reset_ctrl: reset-controller {
    compatible = "gpio-reset";          /* أو أي driver بيعمل create للـ auxiliary device */
    reset-gpios = <&gpio0 5 GPIO_ACTIVE_LOW>;
    #reset-cells = <1>;
};
```

```bash
# تحقق من الـ DT parsed بالفعل
ls /proc/device-tree/

# اقرأ الـ reset-gpios property
cat /proc/device-tree/<node>/reset-gpios | xxd
# bytes: [phandle 4B][GPIO number 4B][flags 4B]
# flags: 1 = GPIO_ACTIVE_LOW

# تحقق من الـ of_node اللي الـ reset controller بيستخدمه
cat /sys/bus/auxiliary/devices/reset.gpio.0/of_node 2>/dev/null || \
    ls -la /sys/bus/auxiliary/devices/reset.gpio.0/

# استخدم dtc لـ decode الـ DT
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "reset-gpios"

# تحقق من pinctrl — GPIO pin لازم يكون configured كـ GPIO مش كـ function تانية
cat /sys/kernel/debug/pinctrl/*/pins | grep -i "gpio5\|pin 5"
cat /sys/kernel/debug/pinctrl/*/pinmux-pins | grep -i "gpio5\|pin 5"
```

---

### Practical Commands

#### شوف state الـ driver كامل

```bash
#!/bin/bash
# reset-gpio-debug.sh — شوف كل حاجة متعلقة بـ reset-gpio driver

echo "=== Auxiliary Devices ==="
ls /sys/bus/auxiliary/devices/ | grep reset

echo ""
echo "=== Driver Binding ==="
ls -la /sys/bus/auxiliary/devices/reset.gpio.*/driver 2>/dev/null

echo ""
echo "=== GPIO State ==="
cat /sys/kernel/debug/gpio | grep -A2 -B2 reset

echo ""
echo "=== Kernel Messages ==="
dmesg | grep -i "reset-gpio\|reset\.gpio\|gpiod.*reset" | tail -20

echo ""
echo "=== Module Info ==="
lsmod | grep reset
modinfo reset_gpio 2>/dev/null | grep -E "filename|description|author"
```

#### فعّل dynamic debug وشوف الـ probe

```bash
# فعّل debug قبل لود الـ module
echo 'file drivers/reset/reset-gpio.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# لو الـ driver محتاج reload
modprobe -r reset_gpio && modprobe reset_gpio

# شوف الـ messages
dmesg | tail -30
```

#### اختبر الـ reset يدوياً عبر libgpiod

```bash
# اعرف رقم الـ GPIO chip والـ line
gpioinfo | grep -B1 reset

# مثال: gpiochip0, line 5
# Assert reset (active-low = set physical LOW)
gpioset --mode=time --sec=1 gpiochip0 5=0

# Deassert reset
gpioset gpiochip0 5=1

# اقرأ الحالة الحالية
gpioget gpiochip0 5
```

#### تتبع الـ assert/deassert calls بـ ftrace

```bash
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo '' > trace
echo function > current_tracer
echo 'reset_gpio_assert
reset_gpio_deassert
reset_gpio_status
gpiod_set_value_cansleep
gpiod_get_value_cansleep' > set_ftrace_filter
echo 1 > tracing_on

# شغّل الـ device اللي بيستخدم الـ reset
# مثلاً: modprobe لـ consumer driver

cat trace | head -50
echo 0 > tracing_on
echo nop > current_tracer
```

**مثال output وتفسيره:**

```
# TASK-PID  CPU#  TIMESTAMP   FUNCTION
  eth0-up-99  [001] 5678.123: reset_gpio_assert <-reset_control_assert
  eth0-up-99  [001] 5678.124: gpiod_set_value_cansleep <-reset_gpio_assert
  eth0-up-99  [001] 5678.224: reset_gpio_deassert <-reset_control_deassert
  eth0-up-99  [001] 5678.225: gpiod_set_value_cansleep <-reset_gpio_deassert
```

الـ `100ms` بين assert وـ deassert هو الـ reset pulse width — لو مش كافي للـ device هتلاقيه بيتهنج.

#### تحقق من الـ pinmux

```bash
# تأكد إن الـ pin مش مخصص لـ function تانية
cat /sys/kernel/debug/pinctrl/*/pinmux-pins 2>/dev/null | grep -i "gpio\|reset" | head -20

# لو الـ pin مش configured كـ GPIO هتلاقي:
# "pin 5 (gpio5): UNCLAIMED"  ← مشكلة
# "pin 5 (gpio5): device gpio-controller function gpio group gpio5-grp" ← صح
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ RS485 Transceiver مش بيتعرف

#### العنوان
الـ RS485 transceiver على industrial gateway مبني على RK3562 مش بيتفعل بعد boot.

#### السياق
شركة بتصنع industrial IoT gateway بيشتغل على RK3562. الـ gateway ده بيتواصل مع sensors صناعية عن طريق RS485. الـ transceiver chip (مثلاً MAX3485) محتاجة GPIO معينة عشان تخرج من الـ reset state وتبدأ تشتغل. الـ firmware engineer كتب الـ DTS بتاعته وشغّل الـ board، لكن الـ RS485 مش شايل أي بيانات.

#### المشكلة
الـ RS485 transceiver واقف في الـ reset — الـ GPIO المسؤولة عن الـ reset مش بتتحرك من HIGH لـ LOW. التطبيق مش بيتلقى بيانات خالص.

#### التحليل
الـ `reset-gpio.c` بيتعامل مع الـ reset عن طريق `auxiliary_device`. الـ probe بتيجي من `reset_gpio_probe`:

```c
priv->reset = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
```

**الـ GPIO بتتاخد بـ `GPIOD_OUT_HIGH`** — يعني لحظة الـ probe، الـ line بتتحط HIGH، وده معناه الـ chip لسه في reset.

بعدين لما device تاني في الـ DT يعمل `reset_control_deassert()`، الكود بيتنفذ:

```c
static int reset_gpio_deassert(struct reset_controller_dev *rc,
                               unsigned long id)
{
    struct reset_gpio_priv *priv = rc_to_reset_gpio(rc);
    return gpiod_set_value_cansleep(priv->reset, 0); /* LOW = deassert */
}
```

المشكلة إن الـ UART driver مش بيكال `reset_control_deassert()` أصلاً، أو إن الـ DT مش فيه `resets` property صح.

**فحص الـ DT:**
```dts
/* غلط — missing resets property في الـ UART node */
&uart3 {
    status = "okay";
    /* resets = <&rs485_reset 0>;  <-- مش موجودة */
};
```

```bash
# تأكيد المشكلة
cat /sys/kernel/debug/gpio | grep reset
# الـ GPIO هتبان HIGH طول الوقت
```

#### الحل
**أولاً:** تعريف الـ reset controller في الـ DT:

```dts
rs485_reset: reset-controller {
    compatible = "reset-gpio";
    reset-gpios = <&gpio3 RK_PB5 GPIO_ACTIVE_HIGH>;
    #reset-cells = <1>;
};

&uart3 {
    status = "okay";
    resets = <&rs485_reset 0>;
    reset-names = "rs485-phy";
};
```

**ثانياً:** لو الـ UART driver مش بيدعم reset control، تضيف manual deassert في الـ board init أو تستخدم `reset-gpios` مباشرة في الـ transceiver node.

```bash
# تحقق إن الـ auxiliary device اتسجل
ls /sys/bus/auxiliary/devices/ | grep reset
# شوف الـ GPIO قيمتها بعد الـ deassert
gpioget gpiochip3 13
```

#### الدرس المستفاد
الـ `GPIOD_OUT_HIGH` في `reset_gpio_probe` بيضمن إن الـ chip تفضل في reset أثناء الـ boot. **لازم الـ consumer driver يعمل `reset_control_deassert()` صراحة**، وإلا الـ peripheral هيفضل مجمد. افتكر دايماً تتحقق من الـ `resets` property في الـ DT للـ consumer node.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ HDMI مش بيظهر صورة

#### العنوان
Android TV box مبني على Allwinner H616، الـ HDMI مش بيشتغل بعد kernel upgrade.

#### السياق
منتج TV box بيستخدم Allwinner H616. الـ HDMI output كان شغال تمام على kernel قديم. بعد upgrade لـ kernel أحدث، الشاشة اتبقت سودا وما فيش signal على الـ HDMI port. الـ team بدأ يحقق.

#### المشكلة
الـ HDMI PHY chip خارجية (مثلاً IT66121) محتاجة GPIO reset. بعد الـ kernel upgrade، الـ `reset-gpio` driver اتغير من device مباشر لـ `auxiliary_device`، والـ DT القديم بقى incompatible.

#### التحليل
الـ `reset_gpio_probe` بيشتغل على `auxiliary_device`:

```c
static int reset_gpio_probe(struct auxiliary_device *adev,
                            const struct auxiliary_device_id *id)
{
    struct device *dev = &adev->dev;
    struct of_phandle_args *platdata = dev_get_platdata(dev);

    if (!platdata)         /* لو platdata فاضية، رجع -EINVAL */
        return -EINVAL;
    ...
}
```

الـ `platdata` بتيجي من الـ parent driver اللي بيعمل `auxiliary_device_add()`. لو الـ parent driver (reset core) اتغير طريقة تسجيله للـ `of_phandle_args`، الـ probe هيفشل بـ `-EINVAL` بصمت.

```bash
# شوف الـ dmesg
dmesg | grep -i "reset\|hdmi\|gpio"
# هتلاقي:
# reset reset.gpio.0: Could not get reset gpios
# أو
# auxiliary reset.gpio.0: probe failed with error -22
```

الـ `reset_gpio_of_xlate` بسيطة جداً:
```c
static int reset_gpio_of_xlate(struct reset_controller_dev *rcdev,
                               const struct of_phandle_args *reset_spec)
{
    return reset_spec->args[0]; /* بترجع أول argument فقط */
}
```

لو الـ HDMI driver بيطلب reset بـ args غلط، الـ xlate هترجع قيمة غلط أو out of range لأن `nr_resets = 1`.

#### الحل
**تحقق من الـ DT syntax الجديدة:**

```dts
/* صح للـ kernel الجديد */
hdmi_phy_reset: hdmi-phy-reset {
    compatible = "reset-gpio";
    reset-gpios = <&pio 7 GPIO_ACTIVE_LOW>;  /* active-low مهم */
    #reset-cells = <1>;
};

&hdmi {
    resets = <&hdmi_phy_reset 0>;
    reset-names = "phy";
    status = "okay";
};
```

```bash
# اتحقق من الـ active polarity
cat /sys/kernel/debug/gpio | grep hdmi

# اعمل manual reset للتجربة
echo 1 > /sys/class/gpio/gpio227/value
sleep 0.1
echo 0 > /sys/class/gpio/gpio227/value

# شوف status الـ reset controller
cat /sys/bus/auxiliary/devices/reset.gpio.0/modalias
```

**لو الـ GPIO active-low غلط في الـ DT**، الـ `gpiod_set_value_cansleep(priv->reset, 1)` في `reset_gpio_assert` هيعمل عكس اللي المفروض يعمله، والـ chip هتفضل deasserted وهي المفروض تكون asserted.

#### الدرس المستفاد
الـ `GPIO_ACTIVE_LOW` / `GPIO_ACTIVE_HIGH` في الـ DT بيأثر على الـ logical value اللي الـ `gpiod_set_value_cansleep` بتشتغل بيه. الـ driver مش بيكتب physical values — بيكتب **logical values**، والـ GPIO subsystem بيعكسها لو محتاج. دايماً تتحقق من datasheet الـ chip عشان تعرف الـ reset polarity الصح.

---

### السيناريو 3: IoT Sensor Board على STM32MP1 — الـ SPI Flash مش بيتعرف بعد sleep

#### العنوان
الـ SPI NOR flash على STM32MP1-based IoT sensor مش بيرجع يشتغل بعد suspend/resume.

#### السياق
IoT sensor بيشتغل على STM32MP1، فيه SPI NOR flash (W25Q64) للـ data logging. الـ board مصممة تدخل في deep sleep كل فترة عشان توفر طاقة. المشكلة ظهرت بعد أسبوعين من التشغيل الميداني — الـ flash بيبقى unreachable بعد أول wake-up.

#### المشكلة
الـ W25Q64 فيه reset pin محتاج يتعمله deassert بعد الـ power-on أو بعد الخروج من sleep. الـ reset GPIO مش بيتعمله deassert في الـ resume path.

#### التحليل
الـ `reset_gpio_probe` بيسجل الـ controller مرة واحدة وبس. الـ `devm_*` resources مربوطة بالـ device lifetime مش بالـ suspend/resume cycle.

لما الـ system يعمل suspend:
1. الـ GPIO controller ممكن يفقد state
2. الـ reset GPIO ترجع لـ default (HIGH في معظم الحالات = chip في reset)
3. لما يحصل resume، محدش بيعمل `deassert` تاني

```c
/* الـ probe بيحط الـ GPIO HIGH أول ما يشتغل */
priv->reset = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);

/* بس مفيش كود suspend/resume في الـ driver */
/* الـ consumer لازم يعمل deassert بنفسه بعد resume */
```

الـ `reset_gpio_ops` مفيهاش `.reset` callback (بس `.assert` و `.deassert` و `.status`):
```c
static const struct reset_control_ops reset_gpio_ops = {
    .assert   = reset_gpio_assert,
    .deassert = reset_gpio_deassert,
    .status   = reset_gpio_status,
    /* .reset = NULL -- مش موجود */
};
```

يعني الـ SPI flash driver لازم يدير الـ assert/deassert بنفسه في الـ pm_ops بتاعته.

```bash
# اتحقق من الـ GPIO state بعد resume
cat /sys/kernel/debug/gpio | grep flash-reset
# لو بان HIGH، الـ chip لسه في reset

# تتبع الـ suspend/resume
echo 1 > /sys/module/gpio/parameters/debug
dmesg | grep -A5 "resume"
```

#### الحل
في الـ SPI flash driver أو board file، تضيف explicit deassert في الـ resume callback:

```c
/* في الـ SPI flash driver */
static int w25q_resume(struct device *dev)
{
    struct w25q_priv *priv = dev_get_drvdata(dev);

    /* deassert reset بعد wake-up */
    reset_control_deassert(priv->rst);
    /* انتظر minimum reset pulse time من datasheet */
    usleep_range(100, 200);

    return 0;
}
```

أو في الـ DT تستخدم `assigned-resets` لو الـ framework بيدعمه:

```dts
&spi1 {
    flash@0 {
        compatible = "winbond,w25q64";
        reg = <0>;
        spi-max-frequency = <50000000>;
        resets = <&flash_reset 0>;
        reset-names = "flash";
    };
};
```

#### الدرس المستفاد
الـ `reset-gpio` driver نفسه مالوش suspend/resume logic — ده مقصود. المسؤولية على الـ **consumer driver** إنه يعمل `reset_control_deassert()` في الـ resume path. في الأنظمة اللي بتعمل power gating، لازم تفكر في كل GPIO state بعد كل wake-up cycle.

---

### السيناريو 4: Automotive ECU على i.MX8 — الـ CAN Transceiver بيعمل glitch أثناء الـ boot

#### العنوان
CAN bus transceiver على automotive ECU (i.MX8) بيبعت spurious frames أثناء الـ kernel boot.

#### السياق
ECU مخصصة للسيارات مبنية على i.MX8MQ. في الـ boot sequence، الـ CAN transceiver (TJA1050) بيبعت frames غلط على الـ CAN bus قبل ما الـ software يكون جاهز. ده بيسبب مشاكل في الـ automotive network لأن الـ ECUs التانية بتستجيب للـ frames دي.

#### المشكلة
الـ TJA1050 بيخرج من reset مبكر جداً — قبل ما الـ CAN controller نفسه يكون initialized. الـ reset GPIO بيتحرر (deassert) في وقت غلط.

#### التحليل
الـ `reset_gpio_probe` بيحط الـ GPIO HIGH عند الـ probe:

```c
priv->reset = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
```

ده صح — الـ transceiver في reset. المشكلة في الـ **timing** لما يحصل deassert.

لو الـ CAN controller driver (مثلاً `flexcan`) بيعمل `reset_control_deassert()` في أول الـ probe بتاعته بدل ما ينتظر الـ network interface يتفتح:

```c
/* في flexcan driver — المشكلة هنا */
static int flexcan_probe(struct platform_device *pdev)
{
    priv->reset = devm_reset_control_get(&pdev->dev, "transceiver");
    reset_control_deassert(priv->reset); /* مبكر أوي! */
    /* ... بعدين بيعمل register_netdev() */
}
```

الـ `reset_gpio_deassert` بتشتغل فوراً:
```c
static int reset_gpio_deassert(struct reset_controller_dev *rc,
                               unsigned long id)
{
    struct reset_gpio_priv *priv = rc_to_reset_gpio(rc);
    return gpiod_set_value_cansleep(priv->reset, 0); /* LOW = transceiver active */
}
```

مفيش delay، مفيش sequencing — الـ chip بتصحى وتبدأ على طول.

```bash
# شوف الـ probe order
dmesg | grep -E "(flexcan|reset.gpio|can[0-9])" | head -20

# راقب الـ CAN bus أثناء boot
candump -t d can0 &
reboot
# هتشوف frames غلط في أول ثواني
```

#### الحل
الـ deassert لازم يحصل بعد ما الـ CAN controller يكون جاهز — يعني في `ndo_open` مش في `probe`:

```c
/* الحل الصح في الـ flexcan/CAN driver */
static int flexcan_open(struct net_device *dev)
{
    struct flexcan_priv *priv = netdev_priv(dev);

    /* assert الأول عشان تضمن clean state */
    reset_control_assert(priv->reset);
    usleep_range(10, 20);  /* minimum reset time من datasheet */

    /* بعدين deassert */
    reset_control_deassert(priv->reset);
    usleep_range(100, 200); /* stabilization time */

    /* دلوقتي ابدأ الـ CAN controller */
    flexcan_chip_enable(priv);
    ...
}

static int flexcan_close(struct net_device *dev)
{
    struct flexcan_priv *priv = netdev_priv(dev);
    flexcan_chip_disable(priv);

    /* ارجع assert عشان الـ transceiver يفضل هادي */
    reset_control_assert(priv->reset);
    ...
}
```

#### الدرس المستفاد
الـ `reset-gpio` driver بينفذ الـ assert/deassert فوراً بدون أي built-in timing. في الـ automotive contexts، **الـ reset sequencing والـ timing حرجين جداً**. الـ consumer driver هو المسؤول الوحيد عن الـ sequence الصح. استخدم `reset_control_assert()` ثم delay ثم `reset_control_deassert()` مع وجود timing margins من الـ datasheet.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ Ethernet PHY مش بيظهر في `ip link`

#### العنوان
أثناء bring-up لـ custom board على AM62x، الـ Ethernet PHY (DP83867) مش بيظهر ومش بيحصل autonegotiation.

#### السياق
فريق hardware عامل custom carrier board على AM62x (TI). الـ board فيها DP83867 Gigabit PHY. الـ BSP جاهز من TI، بس الـ firmware engineer محتاج يكيّفه للـ custom board. بعد boot، `ip link show eth0` بيظهر الـ interface بس `NO-CARRIER` وما فيش link.

#### المشكلة
الـ PHY reset GPIO غلط في الـ DT، والـ `reset-gpio` driver بيتسجل بس مش بيوصل للـ GPIO الصح.

#### التحليل
خطوة خطوة في الـ `reset_gpio_probe`:

```c
static int reset_gpio_probe(struct auxiliary_device *adev,
                            const struct auxiliary_device_id *id)
{
    struct device *dev = &adev->dev;
    struct of_phandle_args *platdata = dev_get_platdata(dev);
    struct reset_gpio_priv *priv;
    int ret;

    if (!platdata)
        return -EINVAL;

    priv = devm_kzalloc(dev, sizeof(*priv), GFP_KERNEL);
    if (!priv)
        return -ENOMEM;

    auxiliary_set_drvdata(adev, &priv->rc);

    /* هنا بياخد الـ GPIO */
    priv->reset = devm_gpiod_get(dev, "reset", GPIOD_OUT_HIGH);
    if (IS_ERR(priv->reset))
        return dev_err_probe(dev, PTR_ERR(priv->reset),
                             "Could not get reset gpios\n");
```

الـ `devm_gpiod_get(dev, "reset", ...)` بيدور على property اسمها `reset-gpios` في الـ DT node المرتبطة بالـ `auxiliary_device`. الـ lookup بيمشي عن طريق `platdata->of_node`.

**المشكلة:** الـ engineer كتب في الـ DT:

```dts
/* غلط */
phy_reset: ethernet-phy-reset {
    compatible = "reset-gpio";
    phy-reset-gpios = <&gpio0 14 GPIO_ACTIVE_LOW>;  /* الاسم غلط! */
    #reset-cells = <1>;
};
```

بدل:
```dts
/* صح */
phy_reset: ethernet-phy-reset {
    compatible = "reset-gpio";
    reset-gpios = <&gpio0 14 GPIO_ACTIVE_LOW>;  /* اسم صح */
    #reset-cells = <1>;
};
```

الـ `devm_gpiod_get(dev, "reset", ...)` بيدور على `reset-gpios` بالظبط (الـ naming convention: `con_id` + `-gpios`). أي اسم تاني هيرجع `-ENOENT`.

```bash
# الـ error في dmesg
dmesg | grep "reset gpios"
# [    3.421] reset reset.gpio.0: Could not get reset gpios: -ENOENT

# تأكد من الـ GPIO property في الـ DT compiled
fdtdump /sys/firmware/fdt | grep -A3 "ethernet-phy-reset"

# أو
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -B2 -A5 "reset-gpio"
```

**بعد تصحيح الـ DT:**

```bash
# تحقق إن الـ reset controller اتسجل
ls /sys/bus/auxiliary/devices/
# المفروض يظهر: reset.gpio.0

# شوف الـ PHY reset GPIO
cat /sys/kernel/debug/gpio | grep phy

# اتحقق من الـ reset status
cat /sys/class/reset_control/*/status 2>/dev/null
```

```bash
# بعد تصحيح الـ DT وإعادة boot
ip link show eth0
# المفروض يظهر: LOWER_UP وفيه carrier
ethtool eth0 | grep -i speed
```

**لو الـ GPIO active polarity غلط برضو:**

```bash
# شوف الـ PHY ID عن طريق MDIO
cat /sys/bus/mdio_bus/*/phy_id
# لو ظهر 0xffffffff، الـ PHY مش responsive = لسه في reset
```

في الحالة دي، الـ `GPIO_ACTIVE_LOW` معناها إن الـ assert = physical HIGH، والـ deassert = physical LOW. لو الـ PHY reset pin active-high (يعني HIGH = reset)، لازم تغير لـ `GPIO_ACTIVE_HIGH`:

```dts
reset-gpios = <&gpio0 14 GPIO_ACTIVE_HIGH>;
```

ولأن `reset_gpio_assert` بتكتب logical 1:
```c
return gpiod_set_value_cansleep(priv->reset, 1); /* logical assert */
```

والـ GPIO subsystem بيترجمها لـ physical value بناءً على الـ polarity في الـ DT.

#### الدرس المستفاد
الـ GPIO consumer name في `devm_gpiod_get(dev, "reset", ...)` **لازم يتطابق بالظبط** مع الـ property name في الـ DT بصيغة `<name>-gpios`. أي خطأ إملائي بيرجع `-ENOENT` وبيفشل الـ probe. كمان، افهم الفرق بين الـ logical polarity (اللي الـ driver بيشتغل بيه) والـ physical polarity (اللي بيتحدد من الـ DT) — الـ GPIO subsystem هو اللي بيعمل الـ translation، مش الـ driver.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ LWN.net** هو أهم مصدر لمتابعة تطور الـ Linux kernel، وفيما يلي أبرز المقالات ذات الصلة بـ reset-gpio driver والـ subsystems اللي بيعتمد عليها:

| المقال | الوصف |
|--------|-------|
| [reset: Add generic GPIO reset driver](https://lwn.net/Articles/585145/) | أول patch أضاف الـ generic GPIO reset driver للـ kernel — نقطة البداية الحرفية للـ `reset-gpio.c` |
| [Managing multifunction devices with the auxiliary bus](https://lwn.net/Articles/840416/) | شرح معمق للـ auxiliary bus اللي بيعتمد عليه الـ driver الحالي — مدمج في kernel 5.11 |
| [GPIO in the kernel: an introduction](https://lwn.net/Articles/532714/) | مقدمة شاملة للـ GPIO subsystem في الـ kernel: الـ `gpiod_*` API والـ descriptor-based model |
| [Introduction of STM32MP13 RCC driver (Reset Clock Controller)](https://lwn.net/Articles/888111/) | مثال حقيقي على تطبيق reset controller في SoC متكامل |
| [Expose and manage PCI device reset](https://lwn.net/Articles/857548/) | نقاش مهم حول إدارة الـ reset في الـ kernel بشكل عام |

---

### التوثيق الرسمي للـ Kernel

**الـ Documentation/** هي المرجع الأساسي للـ APIs والـ subsystems:

#### Reset Subsystem
- **[Reset controller API — docs.kernel.org](https://docs.kernel.org/driver-api/reset.html)**
  الـ API الكاملة: `reset_controller_dev`، `reset_control_ops`، وكيفية تسجيل الـ controller.

- **[Documentation/devicetree/bindings/reset — GitHub](https://github.com/torvalds/linux/tree/master/Documentation/devicetree/bindings/reset)**
  الـ Device Tree bindings لكل reset controllers في الـ kernel.

- **[include/linux/reset-controller.h — GitHub](https://github.com/torvalds/linux/blob/master/include/linux/reset-controller.h)**
  تعريف `struct reset_controller_dev` و`struct reset_control_ops` — القلب النابض للـ subsystem.

#### GPIO Subsystem
- **[General Purpose Input/Output (GPIO) — docs.kernel.org](https://docs.kernel.org/driver-api/gpio/index.html)**
  النقطة الرئيسية لكل توثيق الـ GPIO subsystem.

- **[GPIO Driver Interface — docs.kernel.org](https://docs.kernel.org/driver-api/gpio/driver.html)**
  كيف يكتب المبرمج الـ GPIO driver ويستخدم الـ `gpiod_*` descriptor API.

- **[GPIO Mappings — docs.kernel.org](https://docs.kernel.org/driver-api/gpio/board.html)**
  كيف يُعرَّف الـ GPIO في Device Tree وكيف يُربط بالـ driver.

- **[Subsystem drivers using GPIO — docs.kernel.org](https://docs.kernel.org/driver-api/gpio/drivers-on-gpio.html)**
  قائمة الـ subsystems اللي تعتمد على GPIO كـ backend — ومنها الـ reset subsystem.

#### Auxiliary Bus
- **[Auxiliary Bus — docs.kernel.org](https://docs.kernel.org/driver-api/auxiliary_bus.html)**
  التوثيق الرسمي للـ `auxiliary_device` و`auxiliary_driver` اللي بيستخدمهم `reset-gpio.c`.

---

### نقاشات الـ Mailing List والـ Patches

هنا تقدر تتبع التطور الكامل للـ driver من أول patch لحد آخر تعديل:

| الـ Patch / النقاش | الوصف |
|---------------------|-------|
| [v8: reset: Add driver for gpio-controlled reset pins — Patchwork](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20130716015038.GD28375@S2101-09.ap.freescale.net/) | النسخة الثامنة من الـ patch الأصلي اللي أضاف الـ GPIO reset driver |
| [v4,4/6: reset: Instantiate reset GPIO controller for shared reset-gpios — Patchwork](https://patchwork.kernel.org/project/linux-pm/patch/20240123141311.220505-5-krzysztof.kozlowski@linaro.org/) | إضافة دعم الـ shared reset-gpios — عمل Krzysztof Kozlowski |
| [v6,0/6: reset: gpio: ASoC: shared GPIO resets — Patchwork](https://patchwork.ozlabs.org/project/devicetree-bindings/cover/20240129115216.96479-1-krzysztof.kozlowski@linaro.org/) | مجموعة patches لدعم الـ shared GPIO resets مع الـ ASoC |
| [LKML: reset: gpio: convert the driver to using the auxiliary bus — v7](https://lkml.org/lkml/2025/11/20/1002) | تحويل الـ driver للـ auxiliary bus — Bartosz Golaszewski |
| [LKML: reset: gpio: convert the driver to using the auxiliary bus — v3](https://lkml.org/lkml/2025/10/29/1101) | نسخة سابقة من نفس العمل لمقارنة التطور |
| [LKML: Krzysztof Kozlowski: shared reset-gpios — v5](https://lkml.org/lkml/2024/1/24/220) | نقاش تفصيلي حول آلية الـ instantiation للـ reset GPIO controller |

---

### قواعد بيانات Commits

- **[Linux commits search — Typesense](https://linux-commits-search.typesense.org/)**
  ابحث بـ `reset-gpio` أو `reset_gpio` لتتبع كل commit في تاريخ الـ driver.

- **[torvalds/linux — GitHub: drivers/reset/reset-gpio.c](https://github.com/torvalds/linux/blob/master/drivers/reset/reset-gpio.c)**
  الملف الحالي مع كل الـ history على GitHub — اضغط "History" لترى كل الـ commits.

---

### Kernelnewbies.org

- **[Linux_6.11 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.11)**
  يتضمن ملخص التغييرات في الـ reset subsystem في kernel 6.11.

- **[Linux_4.3 — kernelnewbies.org](https://kernelnewbies.org/Linux_4.3)**
  الـ release اللي شاف تطورات مهمة في الـ reset subsystem المبكر.

- **[LinuxChanges — kernelnewbies.org](https://kernelnewbies.org/LinuxChanges)**
  نقطة انطلاق لتتبع أي تغيير في أي subsystem عبر الإصدارات.

- **[Documents — kernelnewbies.org](https://kernelnewbies.org/Documents)**
  مجموعة وثائق تعليمية لكتابة الـ kernel drivers من الصفر.

---

### eLinux.org

- **[EBC gpio Polling and Interrupts — elinux.org](https://elinux.org/EBC_gpio_Polling_and_Interrupts)**
  أمثلة عملية للتعامل مع GPIO في Embedded Linux — مفيد لفهم السياق العملي.

- **[Leapster Explorer: GPIO subsystem — elinux.org](https://elinux.org/Leapster_Explorer:_GPIO_subsystem)**
  دراسة حالة على الـ GPIO subsystem في جهاز embedded حقيقي.

---

### كتب مرجعية

#### Linux Device Drivers 3rd Edition (LDD3)
- **الفصل 1**: Introduction to Device Drivers — فهم الـ driver model الأساسي
- **الفصل 14**: The Linux Device Model — الـ `kobject`، الـ `bus`، والـ `device` — أساس الـ auxiliary bus
- متاح مجاناً: [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **الفصل 13**: The Virtual Filesystem — فهم الـ abstraction layers
- **الفصل 17**: Devices and Modules — كيف يُسجَّل الـ driver ويُربط بالـ device
- مرجع ممتاز لفهم `module_auxiliary_driver()` macro والـ lifecycle الكامل

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
- **الفصل 9**: File System و Devices — الـ GPIO والـ reset في السياق الـ embedded
- **الفصل 15**: Debugging Embedded Linux — أدوات debug الـ GPIO في الـ target hardware

---

### مصطلحات البحث

استخدم هذه المصطلحات في Google أو [elixir.bootlin.com](https://elixir.bootlin.com/linux/latest/source) للعثور على مزيد من المعلومات:

```
reset_controller_dev linux kernel
devm_reset_controller_register
gpiod_set_value_cansleep reset
auxiliary_bus reset driver linux
reset-gpios device tree binding
reset_control_ops assert deassert
module_auxiliary_driver linux kernel
of_phandle_args reset controller
```

---

### روابط سريعة للمصادر الأولية

| المصدر | الرابط |
|--------|--------|
| الـ source file الرسمي | `drivers/reset/reset-gpio.c` في kernel tree |
| الـ Device Tree binding | `Documentation/devicetree/bindings/reset/gpio-reset.yaml` |
| الـ Reset API header | `include/linux/reset-controller.h` |
| الـ GPIO consumer header | `include/linux/gpio/consumer.h` |
| الـ Auxiliary bus header | `include/linux/auxiliary_bus.h` |
| MAINTAINERS | ابحث عن `RESET CONTROLLER FRAMEWORK` في `MAINTAINERS` |
## Phase 8: Writing simple module

### الفكرة

**`reset_gpio_probe`** هي أكتر function مثيرة للاهتمام في الملف — هي اللي بتسجّل الـ reset controller وبتربط الـ GPIO بيه. هنعمل **kprobe** عليها عشان نعرف كل مرة بيتـ probe فيها device جديد: اسمه، والـ GPIO descriptor اللي اتخصصله.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_reset_gpio.c
 *
 * Hooks reset_gpio_probe() to log every auxiliary device
 * that gets a GPIO-based reset controller registered for it.
 */

/* ---- includes ---- */
#include <linux/module.h>       /* MODULE_* macros, module_init/exit       */
#include <linux/kernel.h>       /* pr_info(), KERN_INFO                    */
#include <linux/kprobes.h>      /* struct kprobe, register/unregister_kprobe */
#include <linux/auxiliary_bus.h>/* struct auxiliary_device, dev_name()     */
#include <linux/device.h>       /* struct device                           */

/*
 * الـ kprobe بيتشغّل عند دخول reset_gpio_probe() — قبل ما تنفّذ أي سطر.
 * بنستخدم الـ pre_handler عشان نمسك الـ arguments من الـ registers مباشرة.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64:  arg0 → rdi (adev), arg1 → rsi (id)
     * On ARM64:   arg0 → x0  (adev), arg1 → x1  (id)
     * كلاهم بيتمثّلوا في regs->di / regs->regs[0] حسب المعمارية.
     * بنستخدم الـ macro المناسب عن طريق struct pt_regs مباشرة.
     */
#if defined(CONFIG_X86_64)
    struct auxiliary_device *adev =
            (struct auxiliary_device *)regs->di;
#elif defined(CONFIG_ARM64)
    struct auxiliary_device *adev =
            (struct auxiliary_device *)regs->regs[0];
#else
    /* fallback: على معماريات تانية مش بندعمها في المثال ده */
    pr_info("reset_gpio_probe: unsupported arch — skipping\n");
    return 0;
#endif

    /*
     * dev_name() بترجع الاسم المسجّل للـ device في sysfs.
     * مثلاً: "reset.gpio.0"  أو "reset.gpio.1"
     * adev->id هو الـ instance number اللي الـ auxiliary_bus بيديه.
     */
    pr_info("[kprobe] reset_gpio_probe called: device='%s' id=%u\n",
            dev_name(&adev->dev),
            adev->id);

    return 0; /* zero = لا تغيير في تدفق التنفيذ */
}

/*
 * الـ post_handler بيتشغّل بعد ما reset_gpio_probe خلصت.
 * بنستخدمه عشان نعرف هل الـ probe نجحت ولا لأ
 * (القيمة راجعة بتكون في regs->ax على x86-64 / regs->regs[0] على ARM64).
 */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
#if defined(CONFIG_X86_64)
    long ret = (long)regs->ax;
#elif defined(CONFIG_ARM64)
    long ret = (long)regs->regs[0];
#else
    long ret = 0;
#endif

    if (ret == 0)
        pr_info("[kprobe] reset_gpio_probe → SUCCESS\n");
    else
        pr_info("[kprobe] reset_gpio_probe → FAILED (ret=%ld)\n", ret);
}

/* تعريف الـ kprobe نفسه — symbol_name هو اسم الـ function اللي هنـ hook عليها */
static struct kprobe kp = {
    .symbol_name = "reset_gpio_probe",
    .pre_handler  = handler_pre,
    .post_handler = handler_post,
};

/* ---- module_init ---- */
static int __init kprobe_reset_gpio_init(void)
{
    int ret;

    /*
     * register_kprobe() بتبحث عن الـ symbol في الـ kernel symbol table
     * وبتزرع breakpoint (int3 على x86 / BRK على ARM64) عند بداية الـ function.
     * لو الـ symbol مش exported أو مش موجود بترجع error.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[kprobe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[kprobe] planted on reset_gpio_probe at %p\n", kp.addr);
    return 0;
}

/* ---- module_exit ---- */
static void __exit kprobe_reset_gpio_exit(void)
{
    /*
     * unregister_kprobe() ضروري جداً في الـ exit عشان تشيل الـ breakpoint
     * وترجع الـ bytes الأصلية — لو مرجعتش هيحصل kernel panic أو corruption
     * أول ما الـ handler page يتـ unmap وحد يحاول يـ probe مكانه.
     */
    unregister_kprobe(&kp);
    pr_info("[kprobe] removed from reset_gpio_probe\n");
}

module_init(kprobe_reset_gpio_init);
module_exit(kprobe_reset_gpio_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Learner");
MODULE_DESCRIPTION("kprobe on reset_gpio_probe to trace GPIO reset controller registration");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|--------|--------|
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل الـ API بتاعته |
| `linux/auxiliary_bus.h` | بنحتاجه عشان نـ cast الـ argument لـ `struct auxiliary_device` |
| `linux/device.h` | بنستخدم `dev_name()` عشان نجيب اسم الـ device |
| `linux/module.h` | الـ macros الأساسية لأي module |

---

#### الـ `handler_pre` — ليه؟

**الـ pre_handler** بيتشغّل قبل الـ function المـ probed — ده معناه إن الـ arguments لسه موجودة في الـ CPU registers. بنعمل cast للـ register الأول (`rdi` على x86-64) لـ `struct auxiliary_device *` عشان نجيب اسم الـ device واللي وصله من الـ auxiliary bus.

---

#### الـ `handler_post` — ليه؟

**الـ post_handler** بيتشغّل بعد ما الـ function رجعت — بنقرأ قيمة الـ return من register الـ return value عشان نعرف هل الـ probe نجحت (0) أو فشلت (error code سالب).

---

#### الـ `struct kprobe kp`

الـ `symbol_name` هو اسم الـ kernel function اللي هنزرع فيها الـ breakpoint. الـ kernel بيحوّله لـ address وقت الـ registration. الـ `pre_handler` و`post_handler` هم الـ callbacks اللي بيتنفذوا قبل وبعد الـ function.

---

#### الـ `module_init` / `module_exit`

- **`register_kprobe`** في الـ init بتزرع الـ breakpoint في الـ kernel وبتربط الـ handlers بيه.
- **`unregister_kprobe`** في الـ exit **إجباري** — بيشيل الـ breakpoint ويرجع الـ instructions الأصلية. لو حذفته والـ module اتـ unload، أول ما الـ CPU يوصل للـ breakpoint address هيـ crash لأن الـ handler page اتشالت.

---

### طريقة التشغيل

```bash
# بناء الـ module
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules

# تحميل الـ module
sudo insmod kprobe_reset_gpio.ko

# مراقبة الـ output (بيظهر لما device جديد يتـ probe)
sudo dmesg -w | grep kprobe

# إزالة الـ module
sudo rmmod kprobe_reset_gpio
```

---

### مثال على الـ Output المتوقع

```
[kprobe] planted on reset_gpio_probe at ffffffffc0a12340
[kprobe] reset_gpio_probe called: device='reset.gpio.0' id=0
[kprobe] reset_gpio_probe → SUCCESS
[kprobe] removed from reset_gpio_probe
```

---

### ملف `Kbuild` المطلوب

```makefile
obj-m += kprobe_reset_gpio.o
```
