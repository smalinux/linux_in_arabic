## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Reset Controller Framework؟

تخيل إنك عندك جهاز كبير فيه تشيبات كتير — WiFi chip، USB controller، GPU، Ethernet MAC — كل واحدة منهم بتحتاج في وقت ما إنها "تتعمل reset"، يعني ترجع لحالتها الأولانية زي ما خرجت من المصنع. زي ما بتعمل restart للكمبيوتر لما بيتعلق.

المشكلة إن كل تشيبة بيتم reset-ها بطريقة مختلفة. في SoC (System on Chip) زي اللي بيشغل موبايلك، ممكن يكون فيه **مئات الـ reset lines** — كل line دي سلك بيتحكم فيه بـ bit واحد في register. لما بتكتب `1` في الـ bit ده بتحط التشيبة في reset (assert)، لما بتكتب `0` بتطلقها (deassert) وتبدأ تشتغل.

**الـ Reset Controller Framework** هو الطبقة اللي الكيرنل بيبنيها عشان يوحد الطريقة اللي بيه أي driver بيطلب يعمل reset لأي جهاز، بدون ما يفكر في التفاصيل الهاردوير.

---

### القصة: المشكلة اللي بيحلها

**قبل الـ Framework:**
كل driver كان بيعمل reset للجهاز بطريقته الخاصة. USB driver بيعرف عنوان الـ register اللي بيتحكم في reset الـ USB. Ethernet driver بيعرف register تانية. لو تغير الـ SoC، لازم تغير كل driver.

**بعد الـ Framework:**
- الـ **reset controller driver** (صاحب التشيبة اللي بتعمل الـ reset، مثلاً دراير الـ Clock/Reset Unit في SoC) بيسجل نفسه مع الـ framework، وبيقوله: "أنا عندي 200 reset line."
- الـ **consumer driver** (زي دراير الـ USB أو الـ Ethernet) بيقول: "عايز reset line رقم 42." من غير ما يعرف إيه الـ register.
- الـ **core.c** هو القلب اللي بيربط الاتنين ببعض.

```
+---------------------+
|  Consumer Driver    |  (USB, Ethernet, GPU driver...)
|  reset_control_get()|
+----------+----------+
           |
           v
+---------------------+
|   Reset Framework   |  <-- ده core.c بالظبط
|   (core.c)          |
+----------+----------+
           |
           v
+---------------------+
|  Reset Controller   |  (دراير الـ SoC اللي عنده الـ register)
|  Driver (.ops)      |
+---------------------+
           |
           v
+---------------------+
|  Hardware Register  |  (bit في CLK_RST_CONTROLLER)
+---------------------+
```

---

### ليه مفيد؟

| المشكلة | الحل في core.c |
|---|---|
| كل driver بيعمل reset بطريقة مختلفة | API موحد: `reset_control_reset()` |
| تشيبتين بتشاركا نفس reset line | Shared reset controls مع reference counting |
| Driver بينسى يعمل reset_control_put | devm_ functions بتتنظف أوتوماتيك |
| SoC مابدهوش reset hardware لبعض الأجهزة | GPIO fallback — بيعمل reset بـ GPIO عادي |
| جهاز محتاج أكتر من reset line | Array resets API |

---

### المفاهيم الأساسية في core.c

#### 1. الـ reset_controller_dev — صاحب الـ Reset Lines
ده الجهاز اللي عنده الـ reset lines فعلاً في الهاردوير. بيتسجل باستخدام `reset_controller_register()` وبيوفر الـ ops:

```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);   // pulse
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);  // keep in reset
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);// release
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);  // read state
};
```

#### 2. الـ reset_control — التذكرة اللي بيمسكها الـ Consumer
لما driver بيقول "عايز reset line رقم 5"، بيرجعله struct `reset_control`:

```c
struct reset_control {
    struct reset_controller_dev *rcdev;  // مين صاحب الـ line
    unsigned int id;                     // رقم الـ line
    bool shared;                         // هل متشارك مع حد تاني؟
    bool acquired;                       // هل محجوز exclusive؟
    atomic_t deassert_count;            // عدد من قالوا deassert (للـ shared)
    atomic_t triggered_count;           // عدد من عملوا reset (للـ shared)
    struct kref refcnt;                  // reference counting
};
```

#### 3. Exclusive vs Shared — مفهوم جوهري

**Exclusive reset:** زي مفتاح غرفة — واحد بس يقدر يستخدمه في نفس الوقت. لو USB driver حاجز الـ reset line، مش ممكن driver تاني يعمل بيها حاجة.

**Shared reset:** زي نور الأوضة اللي بيتحكم فيه أكتر من واحد. المنطق بالـ reference counting:
- كل `deassert` بيزود الـ counter — النور بيولع
- كل `assert` بيطفي counter — النور بيطفى بس لما العداد يوصل لصفر (كل الناس طفوه)

```
Driver A: deassert → counter = 1 → hardware deasserted
Driver B: deassert → counter = 2 → no-op (already deasserted)
Driver A: assert  → counter = 1 → no-op (B still wants it deasserted)
Driver B: assert  → counter = 0 → hardware asserted (NOW it's reset)
```

#### 4. الـ GPIO Fallback — لما مافيش Reset Controller

بعض الأجهزة في الـ Device Tree ما بتعرفش reset controller، بس بتعرف reset-gpios (سلك GPIO بيعمل reset لما يتحط على قيمة معينة). الـ framework في `__of_reset_control_get()` بيتشيك: لو مافيش `resets` property، بيدور على `reset-gpios` وبيعمل virtual reset controller عن طريق auxiliary device اسمه `reset-gpio`.

---

### الـ devm_ — الإدارة التلقائية للموارد

الـ `devm_` prefix معناها "device managed" — لما الـ driver بينفصل (detach)، الكيرنل أوتوماتيك بيعمل cleanup:

```c
// بدون devm: لازم تتذكر تعمل reset_control_put() في كل مسار خروج
struct reset_control *rstc = reset_control_get_exclusive(dev, "usb");
// ...
reset_control_put(rstc); // ممكن تنسى!

// مع devm: بيتنظف أوتوماتيك
struct reset_control *rstc = devm_reset_control_get_exclusive(dev, "usb");
// مش محتاج تعمل put — بيتعمل أوتوماتيك لما الـ device يتفصل
```

---

### سيناريو حقيقي: Ethernet Chip على SoC

```
Device Tree:
    ethernet0: ethernet@1c50000 {
        resets = <&ccu RST_BUS_EMAC0>;
        reset-names = "stmmaceth";
    };

    ccu: clock@3001000 {
        compatible = "allwinner,sun50i-h6-ccu";
        #reset-cells = <1>;
    };
```

1. الكيرنل يبوت، دراير الـ CCU (clock control unit) بيعمل `reset_controller_register()`.
2. الـ Ethernet driver يشتغل، بيقول `devm_reset_control_get_exclusive(dev, "stmmaceth")`.
3. الـ framework بيدور في الـ Device Tree: بيلاقي `resets = <&ccu RST_BUS_EMAC0>`.
4. بيدور على الـ CCU في `reset_controller_list` — بيلاقيه.
5. بيعمل `reset_control` object بـ ID = RST_BUS_EMAC0.
6. الـ Ethernet driver بيقول `reset_control_deassert(rstc)` — الـ framework بيكلم `ccu_ops->deassert(rcdev, RST_BUS_EMAC0)`.
7. الـ CCU driver بيكتب في الـ register الصح → الـ Ethernet chip بتشتغل.

---

### الملفات اللي بتكون المنظومة دي

#### الـ Core (القلب):
| الملف | الدور |
|---|---|
| `drivers/reset/core.c` | **الملف ده** — كل منطق الـ framework |
| `include/linux/reset.h` | Consumer API — الـ API اللي بيستخدمه الـ drivers |
| `include/linux/reset-controller.h` | Provider API — الـ structs اللي بيستخدمها الـ controller driver |

#### Hardware Drivers (أمثلة):
| الملف | الـ SoC |
|---|---|
| `drivers/reset/reset-simple.c` | Generic simple reset (register-based) |
| `drivers/reset/reset-gpio.c` | GPIO-based reset fallback |
| `drivers/reset/reset-imx7.c` | NXP i.MX7 SoC |
| `drivers/reset/reset-sunxi.c` | Allwinner SoCs |
| `drivers/reset/reset-k210.c` | Kendryte K210 RISC-V |
| `drivers/reset/reset-ti-sci.c` | Texas Instruments via firmware |
| `drivers/reset/reset-scmi.c` | ARM SCMI firmware interface |
| `drivers/reset/starfive/` | StarFive JH71x0 |
| `drivers/reset/tegra/` | NVIDIA Tegra |

#### Documentation:
| الملف | المحتوى |
|---|---|
| `Documentation/driver-api/reset.rst` | الـ API reference الرسمي |
| `Documentation/devicetree/bindings/reset/` | كيفية وصف الـ resets في Device Tree |
| `include/dt-bindings/reset/` | Constants للـ Device Tree |
## Phase 2: شرح الـ Reset Controller Framework

### المشكلة اللي الـ Subsystem بيحلها

في أي SoC حديث — خد مثلاً Allwinner H3 أو NXP i.MX8 — في عشرات الـ IP blocks جوه: USB controller، Ethernet MAC، display engine، audio DSP، وهكذا. كل واحد من دول بيحتاج **reset line** — خط هاردوير بيخلي الـ block يرجع لـ initial state.

المشكلة إن:

1. الـ reset lines مش موحدة — بعضها GPIO، بعضها بت في register في CRU (Clock/Reset Unit)، وبعضها PMIC output.
2. كل driver لو اتعامل مع الـ reset line بشكل مباشر، هيعمل coupling مع الهاردوير — مش portable.
3. نفس الـ reset line ممكن تكون **shared** بين أكتر من device (مثلاً USB PHY و USB controller بياخدوا نفس الـ reset).
4. لو اتنين drivers حاولوا يـ assert/deassert نفس الـ line في نفس الوقت بدون تنسيق، هيحصل corruption.

**الملخص:** لازم يكون في طبقة abstraction بين الـ driver اللي محتاج reset وبين الهاردوير اللي بينفذه.

---

### الحل — مدخل الـ Kernel

الـ kernel حل المشكلة بالـ **Reset Controller Framework** — طبقة وسط (middleware layer) بتوفر:

- **API موحد** لأي driver محتاج reset: `reset_control_assert()`, `reset_control_deassert()`, `reset_control_reset()`.
- **abstraction layer** يفصل الـ consumer (اللي محتاج reset) عن الـ provider (اللي بيتحكم في الهاردوير).
- **reference counting** للـ shared resets عشان محدش يـ assert لما حد تاني لسه شغال.
- **Device Tree integration** عشان كل device يعرف reset lines بتاعته من الـ DT بدون hardcoding.

---

### تشبيه من الواقع

تخيل **لوحة كهرباء** (distribution board) في مبنى:

| العنصر في التشبيه | المقابل في الـ Kernel |
|---|---|
| لوحة الكهرباء نفسها | `reset_controller_dev` (rcdev) |
| كل قاطع (circuit breaker) | reset line واحدة — بـ ID محدد |
| الكهربائي اللي بيشغل/يطفي القاطع | `reset_control_ops` (.assert / .deassert) |
| الشركة اللي بتستأجر الوحدة | driver (consumer) |
| بطاقة الاستئجار | `reset_control` handle |
| وحدة مشتركة بين شركتين | shared reset — `deassert_count` بيتحكم مين يقفل |
| مكتب إدارة المبنى | الـ framework core نفسه (`core.c`) |
| قائمة كل الوحدات المؤجَّرة | `reset_controller_list` + `reset_control_head` |

**التعمق في التشبيه:**
- لما شركة (driver) عايزة تشغل وحدتها، بتاخد **مفتاح** من مكتب الإدارة — ده `reset_control_get_exclusive()`.
- لو الوحدة **مشتركة** بين شركتين، المكتب بيتابع: لما الشركتين الاتنين قالوا "شغّل" → القاطع يتشغل. لو واحدة قالت "أوقف" بس التانية لسه شغالة → المكتب بيمنع إيقاف القاطع. ده الـ `deassert_count`.
- الـ "exclusive released" زي شركة بتاخد مفتاح احتياطي بس ماتستخدمشيش غير لما تطلب إذن (`reset_control_acquire()`).

---

### Big Picture Architecture

```
  ┌────────────────────────────────────────────────────────────────┐
  │                        Consumer Drivers                        │
  │   USB driver    Ethernet driver    Display driver    ...       │
  │      │                │                 │                      │
  │  reset_control_    reset_control_   reset_control_            │
  │  get_exclusive()   get_shared()     reset()                   │
  └──────────────────────────┬─────────────────────────────────────┘
                             │  Consumer API (include/linux/reset.h)
  ═══════════════════════════╪═══════════════════════════════════════
                             │
  ┌──────────────────────────▼─────────────────────────────────────┐
  │              Reset Controller Framework Core                   │
  │                   (drivers/reset/core.c)                       │
  │                                                                │
  │  ┌──────────────────┐   ┌───────────────────────────────────┐  │
  │  │ reset_controller │   │     reset_control (handle)        │  │
  │  │     _list        │   │  ┌──────────────────────────────┐ │  │
  │  │  (global list)   │   │  │ id, shared, acquired, kref   │ │  │
  │  └──────────────────┘   │  │ deassert_count (atomic)      │ │  │
  │                         │  │ triggered_count (atomic)     │ │  │
  │  DT lookup:             │  └──────────────────────────────┘ │  │
  │  __of_reset_control_get │  (one per consumer per line)      │  │
  │  → of_xlate()           └───────────────────────────────────┘  │
  │  → __reset_control_get_internal()                              │
  └──────────────────────────┬─────────────────────────────────────┘
                             │  Provider API (include/linux/reset-controller.h)
  ═══════════════════════════╪═══════════════════════════════════════
                             │
  ┌──────────────────────────▼─────────────────────────────────────┐
  │                    Reset Controller Drivers                    │
  │  (providers — بيسجلوا reset_controller_dev)                   │
  │                                                                │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
  │  │  Allwinner   │  │  NXP i.MX    │  │  reset-gpio driver   │ │
  │  │  CCU driver  │  │  CCM driver  │  │  (GPIO-based reset)  │ │
  │  │  (CRU regs)  │  │  (CRU regs)  │  │                      │ │
  │  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘ │
  └─────────┼────────────────┼────────────────────────┼────────────┘
            │                │                        │
  ┌─────────▼────────────────▼────────────────────────▼────────────┐
  │                     Hardware Layer                             │
  │   CRU Reset Registers     GPIO Lines     PMIC Reset Outputs   │
  └────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction

الفكرة المحورية في الـ framework هي **الفصل بين "من يطلب" و"من ينفذ"** عبر ثلاث structs أساسية:

#### 1. `struct reset_controller_dev` — الـ Provider

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;  /* vtable — الـ implementation */
    struct module *owner;
    struct list_head list;                /* entry في reset_controller_list */
    struct list_head reset_control_head;  /* قائمة الـ handles المفتوحة */
    struct device *dev;
    struct device_node *of_node;          /* ربط بالـ Device Tree */
    const struct of_phandle_args *of_args;/* للـ GPIO-based resets */
    int of_reset_n_cells;                 /* عدد خلايا الـ specifier في DT */
    int (*of_xlate)(...);                 /* ترجمة DT specifier → ID */
    unsigned int nr_resets;              /* عدد الـ reset lines */
};
```

كل driver بيوفر reset lines بيملأ الـ struct ده ويسجله بـ `reset_controller_register()`. الـ framework بيضيفه لـ `reset_controller_list` العالمية.

#### 2. `struct reset_control_ops` — الـ vtable

```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

الـ `id` هو رقم الـ reset line جوه الـ controller ده. الـ driver بيتحكم في الهاردوير بناءً عليه.

#### 3. `struct reset_control` — الـ Consumer Handle

```c
struct reset_control {
    struct reset_controller_dev *rcdev; /* pointer للـ provider */
    struct list_head list;              /* entry في rcdev->reset_control_head */
    unsigned int id;                    /* رقم الـ line في الـ rcdev */
    struct kref refcnt;                 /* reference counting */
    bool acquired;                      /* هل ممكن يستخدمه دلوقتي؟ */
    bool shared;                        /* shared أم exclusive؟ */
    bool array;                         /* هل ده array of controls؟ */
    atomic_t deassert_count;            /* للـ shared: كام consumer deasserted */
    atomic_t triggered_count;           /* للـ shared: اتعمل reset مرة؟ */
};
```

ده الـ **token** اللي الـ consumer بياخده ويتعامل بيه. الـ framework بيعمل caching للـ handles — لو اتنين consumers طلبوا نفس الـ shared line، بياخدوا نفس الـ object (بعد `kref_get`).

---

### العلاقة بين الـ Structs

```
  reset_controller_list (global)
        │
        ├──► rcdev_A ──── reset_control_head
        │         │              │
        │         │         ┌────▼──────────────────────────────┐
        │         │         │ rstc (id=0, shared=false, acq=true)│
        │         │         │ rstc (id=1, shared=true)           │◄─── Consumer 1
        │         │         │ rstc (id=1, shared=true, same obj) │◄─── Consumer 2
        │         │         └───────────────────────────────────-┘
        │         │              (نفس الـ object → kref=2)
        │         │
        │         └── ops → { .assert=cru_assert, .deassert=cru_deassert }
        │
        └──► rcdev_B (GPIO-based reset controller)
                  └── ops → { .reset=gpio_reset }
```

---

### منطق الـ Shared vs Exclusive بالتفصيل

#### Exclusive Reset

لما driver يطلب `reset_control_get_exclusive()`:
- لو حد تاني بيستخدم نفس الـ line بـ exclusive → `-EBUSY`.
- الـ `acquired = true` من أول ما يتاخد.
- `reset_control_assert()` بينفذ مباشرة بدون فحص counters.

#### Shared Reset

لما driver يطلب `reset_control_get_shared()`:
- أي عدد من الـ consumers يقدروا يشاركوا نفس الـ line.
- الـ framework بيتابع `deassert_count`:

```c
/* من reset_control_deassert() */
if (atomic_inc_return(&rstc->deassert_count) != 1)
    return 0;  /* مش أول واحد → مفيش حاجة تتعمل في الهاردوير */

/* لو أول واحد → نفّذ deassert فعلاً */
return rstc->rcdev->ops->deassert(rstc->rcdev, rstc->id);
```

```c
/* من reset_control_assert() */
if (atomic_dec_return(&rstc->deassert_count) != 0)
    return 0;  /* لسه في consumers شغالين → ماتعملش assert */

/* لو الأخير → نفّذ assert فعلاً */
return rstc->rcdev->ops->assert(rstc->rcdev, rstc->id);
```

**ده بالظبط زي mutex semaphore** — بس على مستوى الهاردوير.

#### Exclusive Released

ده نوع خاص: يتاخد exclusive بس `acquired = false` من البداية. لازم Consumer يعمل `reset_control_acquire()` أولاً قبل ما يستخدمه. الـ framework بيفحص إن محدش تاني acquired:

```c
/* من reset_control_acquire() */
list_for_each_entry(rc, &rstc->rcdev->reset_control_head, list) {
    if (rstc != rc && rstc->id == rc->id) {
        if (rc->acquired)
            return -EBUSY;  /* حد تاني بيستخدمه */
    }
}
rstc->acquired = true;
```

ده مفيد لما يكون في أكتر من subsystem بيتشاركوا نفس الـ reset لكن مش في نفس الوقت.

---

### مسار الـ Lookup من الـ Device Tree

لما driver بيعمل `devm_reset_control_get_exclusive(dev, "usb")`:

```
devm_reset_control_get_exclusive(dev, "usb")
        │
        ▼
__devm_reset_control_get(dev, "usb", 0, EXCLUSIVE)
        │
        ▼
__reset_control_get(dev, "usb", 0, EXCLUSIVE)
        │
        ├── dev->of_node موجود؟
        ▼
__of_reset_control_get(node, "usb", 0, EXCLUSIVE)
        │
        ├── 1. of_property_match_string(node, "reset-names", "usb") → index=2
        │
        ├── 2. of_parse_phandle_with_args(node, "resets", "#reset-cells", 2, &args)
        │       → args.np = &cru_node, args.args[0] = 42
        │
        ├── 3. __reset_find_rcdev(&args) → rcdev اللي of_node == cru_node
        │
        ├── 4. rcdev->of_xlate(rcdev, &args) → id = 42
        │
        └── 5. __reset_control_get_internal(rcdev, 42, EXCLUSIVE)
                    │
                    ├── بحث في rcdev->reset_control_head
                    │   لو موجود وشغال مع flags → error أو kref_get
                    │
                    └── إنشاء reset_control جديد + إضافته للـ list
```

---

### الـ GPIO Fallback

لو الـ device مش عنده `resets` property في الـ DT بس عنده `reset-gpios`، الـ framework بيعمل:

1. بيقرأ الـ GPIO phandle من `reset-gpios`.
2. بيخلق **auxiliary device** جديد بـ name = `"reset.gpio.N"`.
3. الـ `reset-gpio` driver بيـ probe الـ auxiliary device ده وبيسجل `reset_controller_dev` جديد بـ ops بتستخدم الـ GPIO.
4. الـ framework بيكمل normally بعد كده.

ده بيخلي الـ GPIO-based resets transparent للـ consumer — ما يعرفش فرق.

```
reset-gpios phandle
       │
       ▼
__reset_add_reset_gpio_device()
       │
       ├── gpio_device_find_by_fwnode()
       ├── fwnode_create_software_node()  ← software node بيحتوي GPIO info
       └── reset_add_gpio_aux_device()    ← auxiliary_device_add("reset.gpio.N")
                   │
                   ▼
           reset-gpio driver probe()
                   │
                   └── reset_controller_register(rcdev)  ← بيبقى عنده of_args بدل of_node
```

---

### الـ Array API

لو device محتاج يـ reset أكتر من line في نفس الوقت، الـ framework بيوفر `of_reset_control_array_get()` اللي بترجع `reset_control_array`:

```c
struct reset_control_array {
    struct reset_control base;      /* base.array = true → علامة إنه array */
    unsigned int num_rstcs;
    struct reset_control *rstc[];   /* flexible array of individual controls */
};
```

كل operation على الـ array بتتعمل على كل عنصر بالترتيب. في حالة الـ assert، لو واحدة فيهم فشلت، الـ framework بيـ rollback الباقيين:

```c
static int reset_control_array_assert(struct reset_control_array *resets)
{
    for (i = 0; i < resets->num_rstcs; i++) {
        ret = reset_control_assert(resets->rstc[i]);
        if (ret)
            goto err;  /* ← rollback */
    }
    return 0;
err:
    while (i--)
        reset_control_deassert(resets->rstc[i]);  /* ← undo ما اتعمل */
    return ret;
}
```

---

### الـ devres Integration

الـ framework متكامل مع نظام الـ **devres** (device resource management). معناه:

- `devm_reset_controller_register()` → بيعمل unregister أوتوماتيك لما الـ driver يتـ detach.
- `devm_reset_control_get_exclusive()` → بيعمل `reset_control_put()` أوتوماتيك.
- `devm_reset_control_get_exclusive_deasserted()` → بيـ deassert عند البداية وبيـ assert + put عند الـ detach.

الـ devres subsystem (مش شارحه هنا بالتفصيل) بيخزن destructors مع كل device وبيشغلهم لما الـ device يتشال — بيلغي الحاجة لـ cleanup code يدوي.

---

### ملخص: الـ Framework بيمتلك إيه وبيفوّض إيه؟

| الـ Framework بيمتلك | الـ Framework بيفوّض للـ Driver |
|---|---|
| قائمة الـ controllers العالمية | الـ register/bit manipulation الفعلي |
| الـ handle allocation والـ caching | ترجمة DT specifier لـ ID (`of_xlate`) |
| الـ reference counting (`kref`) | معرفة عدد الـ reset lines (`nr_resets`) |
| منطق الـ shared/exclusive | أي حاجة خاصة بالهاردوير |
| الـ deassert_count للـ shared resets | تنفيذ `.reset`, `.assert`, `.deassert`, `.status` |
| الـ GPIO fallback mechanism | — |
| الـ devres cleanup | — |
| الـ DT lookup (`of_parse_phandle_with_args`) | — |
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options

#### الـ Flags الأساسية (Bit Flags)

| Flag | Value | المعنى |
|------|-------|--------|
| `RESET_CONTROL_FLAGS_BIT_SHARED` | `BIT(0)` | الـ reset مشترك بين أكتر من consumer |
| `RESET_CONTROL_FLAGS_BIT_OPTIONAL` | `BIT(1)` | لو مش موجود في DT → رجّع NULL بدل error |
| `RESET_CONTROL_FLAGS_BIT_ACQUIRED` | `BIT(2)` | الـ exclusive reset متملّك فورًا بعد الـ get |
| `RESET_CONTROL_FLAGS_BIT_DEASSERTED` | `BIT(3)` | اعمل deassert تلقائي بعد الـ get |

#### الـ `enum reset_control_flags` — Cheatsheet

| القيمة | SHARED | OPTIONAL | ACQUIRED | DEASSERTED |
|--------|--------|----------|----------|------------|
| `RESET_CONTROL_EXCLUSIVE` | - | - | yes | - |
| `RESET_CONTROL_EXCLUSIVE_DEASSERTED` | - | - | yes | yes |
| `RESET_CONTROL_EXCLUSIVE_RELEASED` | - | - | - | - |
| `RESET_CONTROL_SHARED` | yes | - | - | - |
| `RESET_CONTROL_SHARED_DEASSERTED` | yes | - | - | yes |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE` | - | yes | yes | - |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE_DEASSERTED` | - | yes | yes | yes |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE_RELEASED` | - | yes | - | - |
| `RESET_CONTROL_OPTIONAL_SHARED` | yes | yes | - | - |
| `RESET_CONTROL_OPTIONAL_SHARED_DEASSERTED` | yes | yes | - | yes |

> **القاعدة الذهبية:** shared + acquired = invalid (WARN_ON في `__reset_control_get`).

#### الـ Config Options

| Kconfig | الأثر |
|---------|-------|
| `CONFIG_RESET_CONTROLLER` | يفعّل كل الـ framework — بدونه كل الـ APIs بترجع stubs |
| `CONFIG_RESET_GPIO` | يفعّل fallback لو الـ device ما عندوش `resets` property في DT، بيستخدم `reset-gpios` |
| `CONFIG_ACPI` | يفعّل مسار `_RST` method في `__device_reset` |

#### الـ Global State

| المتغير | النوع | الـ Lock | الغرض |
|---------|-------|---------|-------|
| `reset_controller_list` | `LIST_HEAD` | `reset_list_mutex` | قائمة كل الـ rcdev المسجّلين |
| `reset_gpio_lookup_list` | `LIST_HEAD` | `reset_gpio_lookup_mutex` | GPIO-based reset devices |
| `reset_gpio_ida` | `IDA` | `reset_gpio_lookup_mutex` | تخصيص IDs للـ GPIO aux devices |

---

### 1. الـ Structs المهمة

#### 1.1 `struct reset_control_ops`

**الغرض:** جدول الـ virtual functions اللي بيوفّرها driver الـ reset controller.

```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

| Field | الغرض | Optional؟ |
|-------|-------|----------|
| `reset` | pulse ذاتي الإلغاء (self-deasserting) | اختياري — بس لازم يكون موجود لـ `reset_control_reset` |
| `assert` | فرض حالة reset على الـ hardware | اختياري — بدونه exclusive assert بيرجع `-ENOTSUPP` |
| `deassert` | رفع الـ reset | اختياري — بدونه بيتفرض إن الـ line deasserted by default |
| `status` | قراءة حالة الـ reset line | اختياري — بدونه `reset_control_status` بيرجع `-ENOTSUPP` |

---

#### 1.2 `struct reset_controller_dev`

**الغرض:** يمثّل controller كامل بيدير N من reset lines. الـ driver بيملي الـ struct ده وبيسجّله في الـ framework.

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;      /* vtable */
    struct module *owner;                      /* منع unload طول ما في consumers */
    struct list_head list;                     /* ربط في reset_controller_list */
    struct list_head reset_control_head;       /* قائمة reset_control instances */
    struct device *dev;                        /* الـ device model backing */
    struct device_node *of_node;               /* DT node كـ phandle target */
    const struct of_phandle_args *of_args;     /* لـ GPIO-based controllers بدل of_node */
    int of_reset_n_cells;                      /* عدد cells في DT specifier */
    int (*of_xlate)(...);                      /* ترجمة DT args → reset line ID */
    unsigned int nr_resets;                    /* عدد reset lines */
};
```

**علاقاته:**
- `ops` → `reset_control_ops` (vtable)
- `list` → `reset_controller_list` (global list)
- `reset_control_head` → list of `reset_control` instances
- `dev` → kernel device model
- `of_node` / `of_args` → DT lookup (متحدش فيهم يتعمر في نفس الوقت)

---

#### 1.3 `struct reset_control`

**الغرض:** يمثّل handle لـ reset line واحدة من وجهة نظر consumer. كل consumer بياخد instance منفصل، لكن لو shared بيشاركوا نفس الـ object عبر `kref`.

```c
struct reset_control {
    struct reset_controller_dev *rcdev;  /* الـ controller اللي بيدير الـ line دي */
    struct list_head list;               /* ربط في rcdev->reset_control_head */
    unsigned int id;                     /* رقم الـ line داخل الـ controller */
    struct kref refcnt;                  /* عدد المستخدمين — عند 0 بيتحذف */
    bool acquired;                       /* exclusive: هل متملّك دلوقتي؟ */
    bool shared;                         /* true = shared mode */
    bool array;                          /* true = ده wrapper لـ array */
    atomic_t deassert_count;            /* shared: كم consumer عمل deassert */
    atomic_t triggered_count;           /* shared: اتعمل reset مرة واحدة بس */
};
```

**نقط مهمة:**
- `deassert_count`: عدّاد مشترك — الـ assert الفعلي بس لما يرجع لـ 0.
- `triggered_count`: للـ shared reset_control_reset — بيشتغل مرة واحدة بس في الـ lifetime.
- `acquired`: الـ mutex على مستوى الـ consumer — exclusive control.

---

#### 1.4 `struct reset_control_array`

**الغرض:** يغلّف مجموعة من `reset_control` handles في object واحد للتعامل معهم كـ unit.

```c
struct reset_control_array {
    struct reset_control base;            /* لازم يكون أول field — casting trick */
    unsigned int num_rstcs;
    struct reset_control *rstc[] __counted_by(num_rstcs); /* flexible array */
};
```

**الحيلة:** الـ `base.array = true` بيخلّي كل functions تشوف إن ده array وتتعامل معاه صح. الـ `container_of` في `rstc_to_array()` بيتحوّل من `base` للـ array بأمان.

---

#### 1.5 `struct reset_control_bulk_data`

**الغرض:** بيسهّل الـ API للـ consumers اللي عندهم أكتر من reset line.

```c
struct reset_control_bulk_data {
    const char          *id;    /* اسم الـ reset line زي ما في DT reset-names */
    struct reset_control *rstc; /* بيتملي بعد الـ get */
};
```

---

#### 1.6 `struct reset_gpio_lookup`

**الغرض:** يحفظ mapping بين GPIO phandle args وبين الـ auxiliary device اللي اتعمل عشانه. بيضمن إن نفس الـ GPIO ما يتسجّلش أكتر من مرة.

```c
struct reset_gpio_lookup {
    struct of_phandle_args of_args; /* الـ GPIO args كاملة */
    struct fwnode_handle *swnode;   /* software node للـ aux device */
    struct list_head list;          /* ربط في reset_gpio_lookup_list */
};
```

---

#### 1.7 `struct reset_control_bulk_devres`

**الغرض:** داخلي فقط — بيحفظ بيانات الـ bulk resources عشان `devres` يقدر يحررهم عند الـ driver detach.

```c
struct reset_control_bulk_devres {
    int num_rstcs;
    struct reset_control_bulk_data *rstcs;
};
```

---

### 2. مخطط العلاقات بين الـ Structs

```
reset_controller_list (global)
        │
        │  list_head
        ▼
┌─────────────────────────────────────┐
│      reset_controller_dev           │
│  ┌─────────────────────────────┐    │
│  │ ops → reset_control_ops     │    │
│  │       .reset()              │    │
│  │       .assert()             │    │
│  │       .deassert()           │    │
│  │       .status()             │    │
│  └─────────────────────────────┘    │
│  owner → struct module              │
│  dev   → struct device              │
│  of_node → device_node             │
│  of_args → of_phandle_args         │
│  reset_control_head ──────────────────────────────────────────┐
└─────────────────────────────────────┘                         │
                                                                 │ list_head
                                                                 ▼
                                              ┌────────────────────────────┐
                                              │    reset_control            │
                                              │  rcdev ──────────────────► rcdev │ (back pointer)
                                              │  id = 3                    │
                                              │  refcnt (kref)             │
                                              │  shared = false            │
                                              │  acquired = true           │
                                              │  deassert_count (atomic)   │
                                              │  triggered_count (atomic)  │
                                              └────────────────────────────┘
                                                         ▲
                                                         │ container_of(base)
                                              ┌────────────────────────────┐
                                              │  reset_control_array        │
                                              │  base (reset_control)       │
                                              │    .array = true            │
                                              │  num_rstcs = 3             │
                                              │  rstc[0] ──► reset_control │
                                              │  rstc[1] ──► reset_control │
                                              │  rstc[2] ──► reset_control │
                                              └────────────────────────────┘

reset_gpio_lookup_list (global)
        │
        ▼
┌─────────────────────────────┐
│   reset_gpio_lookup          │
│  of_args (phandle_args)      │
│  swnode ──► fwnode_handle    │
│              (software node) │
└─────────────────────────────┘
        │
        │ يؤدي لإنشاء
        ▼
┌─────────────────────────────┐
│   auxiliary_device           │
│   name = "reset.gpio"        │
│   → يتحوّل لـ rcdev جديد    │
└─────────────────────────────┘
```

---

### 3. مخطط الـ Lifecycle

#### 3.1 Lifecycle الـ Controller (Provider Side)

```
Driver يبدأ probe()
        │
        ▼
يملي reset_controller_dev:
  .ops, .nr_resets, .of_node, .dev
        │
        ▼
reset_controller_register(rcdev)
  ├── يتحقق: of_node XOR of_args
  ├── لو مفيش of_xlate → يستخدم of_reset_simple_xlate
  ├── INIT_LIST_HEAD(&rcdev->reset_control_head)
  └── list_add(&rcdev->list, &reset_controller_list)
           [يحتاج reset_list_mutex]
        │
        ▼
    [Controller نشط — consumers يقدروا يطلبوه]
        │
        ▼
Driver يبدأ remove() أو devm cleanup
        │
        ▼
reset_controller_unregister(rcdev)
  └── list_del(&rcdev->list)
           [يحتاج reset_list_mutex]
        │
        ▼
    [Controller اختفى من القائمة — consumers جدد هيلاقوا EPROBE_DEFER]
```

#### 3.2 Lifecycle الـ Consumer (reset_control)

```
Consumer يطلب reset:
reset_control_get_exclusive(dev, "uart")
        │
        ▼
__reset_control_get(dev, id, 0, RESET_CONTROL_EXCLUSIVE)
        │
        ▼
__of_reset_control_get(node, "uart", 0, flags)
  ├── of_property_match_string(node, "reset-names", "uart") → index
  ├── of_parse_phandle_with_args(node, "resets", ...) → args
  │     لو فشل وCONFIG_RESET_GPIO:
  │       of_parse_phandle_with_args(node, "reset-gpios", ...) → args
  │       __reset_add_reset_gpio_device(&args)  [GPIO fallback]
  ├── mutex_lock(&reset_list_mutex)
  ├── __reset_find_rcdev(&args, gpio_fallback) → rcdev
  ├── rcdev->of_xlate(rcdev, &args) → rstc_id
  └── __reset_control_get_internal(rcdev, rstc_id, flags)
            ├── يبحث في rcdev->reset_control_head
            │     لو موجود ومشترك: kref_get → return existing
            ├── kzalloc(reset_control)
            ├── try_module_get(rcdev->owner)
            ├── rstc->rcdev = rcdev
            ├── list_add(&rstc->list, &rcdev->reset_control_head)
            ├── kref_init(&rstc->refcnt)
            ├── rstc->acquired = acquired
            ├── rstc->shared = shared
            └── get_device(rcdev->dev)
        │
        ▼
    [rstc متاح للاستخدام]
        │
        ├── reset_control_assert(rstc)
        ├── reset_control_deassert(rstc)
        ├── reset_control_reset(rstc)
        └── reset_control_status(rstc)
        │
        ▼
reset_control_put(rstc)
  ├── mutex_lock(&reset_list_mutex)
  └── __reset_control_put_internal(rstc)
          └── kref_put(&rstc->refcnt, __reset_control_release)
                    لو refcnt وصل 0:
                      ├── module_put(rcdev->owner)
                      ├── list_del(&rstc->list)
                      ├── put_device(rcdev->dev)
                      └── kfree(rstc)
```

#### 3.3 Lifecycle الـ GPIO Fallback (reset-gpio path)

```
__of_reset_control_get() ← "resets" property مش موجودة
        │
        ▼
of_parse_phandle_with_args(node, "reset-gpios", ...) → args
        │
        ▼
__reset_add_reset_gpio_device(&args)
  ├── [lockdep: reset_list_mutex يجب ما يكونش محجوز]
  ├── gpio_device_find_by_fwnode() ← يلاقي GPIO provider
  ├── guard(mutex)(&reset_gpio_lookup_mutex)
  ├── يبحث في reset_gpio_lookup_list → لو موجود: return 0
  ├── يبني property_entry[] (compatible, reset-gpios)
  ├── ida_alloc(&reset_gpio_ida) → id
  ├── kzalloc(reset_gpio_lookup)
  ├── fwnode_create_software_node(properties, NULL) → swnode
  ├── reset_add_gpio_aux_device(parent, swnode, id, &of_args)
  │     ├── kzalloc(auxiliary_device)
  │     ├── auxiliary_device_init(adev)
  │     └── __auxiliary_device_add(adev, "reset")
  │               → bind لـ reset-gpio driver
  │                   → probe() → reset_controller_register()
  └── list_add(&rgpio_dev->list, &reset_gpio_lookup_list)
        │
        ▼
[يرجع لـ __of_reset_control_get]
mutex_lock(&reset_list_mutex)
__reset_find_rcdev(&args, gpio_fallback=true)
  └── يبحث عن rcdev->of_args == args
```

---

### 4. مخططات Call Flow

#### 4.1 `reset_control_reset()` — Exclusive

```
consumer calls reset_control_reset(rstc)
  │
  ├── rstc == NULL → return 0  (optional reset)
  ├── IS_ERR(rstc) → return -EINVAL
  ├── rstc->array == true
  │     └── reset_control_array_reset(rstc_to_array(rstc))
  │               └── for each rstc[i]: reset_control_reset(rstc[i])
  │
  ├── !rcdev->ops->reset → return -ENOTSUPP
  │
  ├── rstc->shared == true
  │     ├── deassert_count != 0 → WARN, return -EINVAL
  │     ├── atomic_inc_return(&triggered_count) != 1 → return 0 (نو-أوب)
  │     └── [أول caller فعلًا بيعمل reset]
  │
  ├── rstc->shared == false
  │     └── !rstc->acquired → return -EPERM
  │
  └── rcdev->ops->reset(rcdev, rstc->id)
        │
        └── [hardware register write — في الـ driver]
              لو فشل وshared: atomic_dec(&triggered_count)
```

#### 4.2 `reset_control_assert()` / `reset_control_deassert()` — Shared Mode

```
=== assert (shared) ===
reset_control_assert(rstc)
  ├── triggered_count != 0 → WARN, return -EINVAL
  ├── deassert_count == 0 → WARN, return -EINVAL
  ├── atomic_dec_return(&deassert_count) != 0 → return 0 (لسه في users)
  ├── !ops->assert → return 0 (valid: line may stay deasserted)
  └── ops->assert(rcdev, id)

=== deassert (shared) ===
reset_control_deassert(rstc)
  ├── triggered_count != 0 → WARN, return -EINVAL
  ├── atomic_inc_return(&deassert_count) != 1 → return 0 (already deasserted)
  ├── !ops->deassert → return 0 (assume self-deasserting)
  └── ops->deassert(rcdev, id)
```

#### 4.3 `reset_control_acquire()` — Exclusive Released Pattern

```
consumer calls reset_control_acquire(rstc)
  ├── IS_ERR/NULL → return 0/-EINVAL
  ├── rstc->array → reset_control_array_acquire()
  │
  └── mutex_lock(&reset_list_mutex)
        ├── rstc->acquired == true → unlock, return 0
        ├── for each rc in rcdev->reset_control_head:
        │     لو rc->id == rstc->id && rc != rstc && rc->acquired
        │         → unlock, return -EBUSY
        ├── rstc->acquired = true
        └── mutex_unlock, return 0
```

#### 4.4 `__device_reset()` — Convenience Wrapper

```
device_reset(dev)
  └── __device_reset(dev, optional=false)
        │
        ├── [CONFIG_ACPI]
        │     ACPI_HANDLE(dev) موجود؟
        │       ├── !acpi_has_method(handle, "_RST") → return optional?0:-ENOENT
        │       └── acpi_evaluate_object(handle, "_RST") → return 0 / -EIO
        │
        ├── flags = RESET_CONTROL_EXCLUSIVE or OPTIONAL
        ├── __reset_control_get(dev, NULL, 0, flags) → rstc
        ├── reset_control_reset(rstc)
        └── reset_control_put(rstc)
```

#### 4.5 `devm` Pattern — Resource Management

```
devm_reset_control_get_exclusive(dev, "eth")
  └── __devm_reset_control_get(dev, "eth", 0, RESET_CONTROL_EXCLUSIVE)
        ├── devres_alloc(devm_reset_control_release, sizeof(ptr*))
        ├── __reset_control_get(...) → rstc
        ├── [لو DEASSERTED flag]: reset_control_deassert(rstc)
        ├── *ptr = rstc
        └── devres_add(dev, ptr)

=== عند driver detach ===
devm_reset_control_release(dev, res)
  └── reset_control_put(*(reset_control**)res)

=== لو DEASSERTED ===
devm_reset_control_release_deasserted(dev, res)
  ├── reset_control_assert(rstc)   ← أول حاجة
  └── reset_control_put(rstc)
```

---

### 5. استراتيجية الـ Locking

#### 5.1 الـ Locks الموجودة

| Lock | النوع | يحمي إيه |
|------|-------|---------|
| `reset_list_mutex` | `DEFINE_MUTEX` | `reset_controller_list` + `rcdev->reset_control_head` + `rstc->acquired` |
| `reset_gpio_lookup_mutex` | `DEFINE_MUTEX` | `reset_gpio_lookup_list` + `reset_gpio_ida` |

#### 5.2 ترتيب الـ Locking (Lock Ordering)

```
reset_gpio_lookup_mutex
        ↓
    [يُحرّر قبل ما نمسك التالي]
        ↓
reset_list_mutex
```

**مهم جداً:** `__reset_add_reset_gpio_device()` لازم يُستدعى **بدون** `reset_list_mutex` — لأن تسجيل الـ aux device ممكن يسبب bind فوري للـ driver اللي هيعمل `reset_controller_register()` واللي هياخد `reset_list_mutex` هو كمان. الـ `lockdep_assert_not_held(&reset_list_mutex)` موجود صراحةً عشان يضمن ده.

#### 5.3 تفصيل: مين بيحمي إيه

```
reset_list_mutex يحمي:
├── reset_controller_list (add/remove rcdev)
├── rcdev->reset_control_head (add/remove rstc instances)
├── rstc->acquired (read + write في reset_control_acquire/release)
└── kref operations على reset_control (kref_get, kref_put)

reset_gpio_lookup_mutex يحمي:
├── reset_gpio_lookup_list (add/search)
└── reset_gpio_ida (alloc/free)

atomic_t (lock-free):
├── rstc->deassert_count → يُعدّل بـ atomic_inc_return / atomic_dec_return
└── rstc->triggered_count → يُعدّل بـ atomic_inc_return / atomic_dec
    (الـ WARN checks على ده ممكن تكون غير دقيقة مع racing — documented limitation)
```

#### 5.4 نقطة دقيقة في الـ Shared Resets

الـ `deassert_count` و`triggered_count` هما `atomic_t` بدون mutex إضافي. ده معناه:
- الـ check-then-act مش fully atomic — ممكن يحصل race بين اتنين consumers.
- الـ kernel docs بتوضح إن الـ consumers المشتركين لازم يتنسقوا هم على مستوى أعلى.
- الـ WARN_ON في الحالات الغلط بتساعد detect المشاكل في development.

#### 5.5 مخطط تلخيصي للـ Locking

```
Consumer A                    Consumer B
     │                             │
     │ reset_control_acquire()     │
     ├─lock(reset_list_mutex)      │
     │ rstc->acquired = true       │ reset_control_acquire()
     ├─unlock(reset_list_mutex)    ├─lock(reset_list_mutex)
     │                             │ رشف rc->acquired == true → EBUSY
     │                             ├─unlock(reset_list_mutex)
     │ [يستخدم الـ reset]          │ [ينتظر أو يحاول بعدين]
     │
     │ reset_control_release()
     ├─lock(reset_list_mutex)
     │ rstc->acquired = false
     └─unlock(reset_list_mutex)
```
## Phase 4: شرح الـ Functions

---

### ملخص عام — Cheatsheet

#### Group 1: Registration / Unregistration

| Function | Exported | وصف مختصر |
|---|---|---|
| `reset_controller_register` | `EXPORT_SYMBOL_GPL` | يسجّل rcdev في قائمة global |
| `reset_controller_unregister` | `EXPORT_SYMBOL_GPL` | يحذف rcdev من القائمة |
| `devm_reset_controller_register` | `EXPORT_SYMBOL_GPL` | نسخة managed تستدعي unregister تلقائياً عند detach |

#### Group 2: Runtime Control (Single)

| Function | Exported | وصف مختصر |
|---|---|---|
| `reset_control_reset` | `EXPORT_SYMBOL_GPL` | pulse reset واحدة (self-deasserting) |
| `reset_control_rearm` | `EXPORT_SYMBOL_GPL` | يسمح لـ shared reset بإعادة التشغيل |
| `reset_control_assert` | `EXPORT_SYMBOL_GPL` | يثبّت خط الـ reset في حالة assert |
| `reset_control_deassert` | `EXPORT_SYMBOL_GPL` | يُحرّر خط الـ reset |
| `reset_control_status` | `EXPORT_SYMBOL_GPL` | يقرأ حالة خط الـ reset |

#### Group 3: Runtime Control (Bulk)

| Function | Exported | وصف مختصر |
|---|---|---|
| `reset_control_bulk_reset` | `EXPORT_SYMBOL_GPL` | reset على قائمة controls بالترتيب |
| `reset_control_bulk_assert` | `EXPORT_SYMBOL_GPL` | assert على القائمة بالترتيب مع rollback |
| `reset_control_bulk_deassert` | `EXPORT_SYMBOL_GPL` | deassert على القائمة عكسياً مع rollback |

#### Group 4: Acquire / Release

| Function | Exported | وصف مختصر |
|---|---|---|
| `reset_control_acquire` | `EXPORT_SYMBOL_GPL` | يحجز exclusive access |
| `reset_control_release` | `EXPORT_SYMBOL_GPL` | يُطلق exclusive access |
| `reset_control_bulk_acquire` | `EXPORT_SYMBOL_GPL` | نسخة bulk لـ acquire |
| `reset_control_bulk_release` | `EXPORT_SYMBOL_GPL` | نسخة bulk لـ release |

#### Group 5: Get / Put

| Function | Exported | وصف مختصر |
|---|---|---|
| `__reset_control_get` | `EXPORT_SYMBOL_GPL` | الـ backend الرئيسي للـ get |
| `__of_reset_control_get` | `EXPORT_SYMBOL_GPL` | get عبر OF/DT مع GPIO fallback |
| `reset_control_put` | `EXPORT_SYMBOL_GPL` | يُحرر reset_control |
| `__reset_control_bulk_get` | `EXPORT_SYMBOL_GPL` | bulk get |
| `reset_control_bulk_put` | `EXPORT_SYMBOL_GPL` | bulk put |
| `__devm_reset_control_get` | `EXPORT_SYMBOL_GPL` | managed get |
| `__devm_reset_control_bulk_get` | `EXPORT_SYMBOL_GPL` | managed bulk get |

#### Group 6: Array

| Function | Exported | وصف مختصر |
|---|---|---|
| `of_reset_control_array_get` | `EXPORT_SYMBOL_GPL` | يجيب array من كل resets في DT node |
| `devm_reset_control_array_get` | `EXPORT_SYMBOL_GPL` | نسخة managed للـ array get |
| `reset_control_get_count` | `EXPORT_SYMBOL_GPL` | عدد الـ resets المتاحة للـ device |

#### Group 7: Internal Helpers (static)

| Function | وصف مختصر |
|---|---|
| `rcdev_name` | يُرجع اسم الـ rcdev للـ logging |
| `of_reset_simple_xlate` | default xlate: 1:1 mapping |
| `__reset_control_get_internal` | ينشئ/يُعيد reset_control مع kref |
| `__reset_control_put_internal` | kref_put مع cleanup callback |
| `__reset_control_release` | cleanup callback: يحذف rstc |
| `reset_control_array_put` | يُحرر array وكل عناصرها |
| `rstc_to_array` | cast من reset_control لـ reset_control_array |
| `reset_control_is_array` | يتحقق إن كان rstc array |
| `__reset_find_rcdev` | يبحث عن rcdev في الـ global list |
| `__reset_add_reset_gpio_device` | ينشئ auxiliary device لـ GPIO reset |
| `reset_add_gpio_aux_device` | يسجّل الـ auxiliary_device نفسه |

---

### Group 1: تسجيل الـ Reset Controller

الـ reset framework يحتفظ بـ global linked list (`reset_controller_list`) محمية بـ `reset_list_mutex`. كل driver يسجّل `reset_controller_dev` واحد أو أكثر. الـ consumers بعدين يبحثوا في الـ list عشان يلاقوا الـ rcdev المناسب بناءً على الـ DT node.

---

#### `reset_controller_register`

```c
int reset_controller_register(struct reset_controller_dev *rcdev);
```

بيضيف الـ `rcdev` للـ global list بعد ما يتحقق من الـ preconditions. لو مفيش `of_xlate` محددة، بيستخدم `of_reset_simple_xlate` كـ default ويضبط `of_reset_n_cells = 1`. يعمل `INIT_LIST_HEAD` على `reset_control_head` اللي هتتجمع فيها الـ controls المطلوبة من الـ rcdev ده.

**Parameters:**
- `rcdev`: مؤشر لـ `reset_controller_dev` مُهيَّأ من الـ driver — لازم يكون `ops` موجود، و`of_node` أو `of_args` بس مش الاتنين مع بعض.

**Return:** `0` نجاح، `-EINVAL` لو `of_node` و`of_args` موجودين مع بعض.

**Key details:**
- بياخد `reset_list_mutex` أثناء `list_add` — لازم يُستدعى من process context.
- مش بيعمل `get_device` — الـ driver مسؤول عن دورة الحياة.
- **Caller:** reset controller drivers من `probe()`.

---

#### `reset_controller_unregister`

```c
void reset_controller_unregister(struct reset_controller_dev *rcdev);
```

بيشيل الـ `rcdev` من الـ global list ببساطة. مش بيعمل أي cleanup للـ `reset_control` instances الموجودة — الـ consumers المفروض يكونوا عملوا `put` قبل الـ unregister.

**Parameters:**
- `rcdev`: نفس الـ pointer اللي اتسجّل بيه.

**Return:** void.

**Key details:**
- بياخد `reset_list_mutex`.
- **Caller:** driver `remove()` أو `devm_reset_controller_release`.

---

#### `devm_reset_controller_register`

```c
int devm_reset_controller_register(struct device *dev,
                                   struct reset_controller_dev *rcdev);
```

يعمل `devres_alloc` يخزن فيه pointer للـ rcdev، يستدعي `reset_controller_register`، ولو نجح يضيف الـ devres resource للـ device. عند driver detach الـ devres framework هيستدعي `devm_reset_controller_release` اللي هو wrapper لـ `reset_controller_unregister`.

**Parameters:**
- `dev`: الـ device المرتبط بالـ driver (يُستخدم لتسجيل الـ devres).
- `rcdev`: الـ reset controller device.

**Return:** `0` نجاح، `-ENOMEM` أو error من `reset_controller_register`.

**Caller:** `probe()` اللي عايزة managed cleanup تلقائي.

---

### Group 2: Runtime Control — Single Reset

الـ framework يفرّق بين نوعين أساسيين:
- **Exclusive**: واحد بس يقدر يتحكم، يحتاج `acquired = true`.
- **Shared**: متعدد consumers، الـ framework يحتفظ بـ `deassert_count` و`triggered_count` atomic لتنسيق الـ state.

---

#### `reset_control_reset`

```c
int reset_control_reset(struct reset_control *rstc);
```

بيبعت reset pulse واحدة (assert ثم deassert في نفس الـ operation) لـ self-deasserting resets. بالنسبة للـ shared controls، الـ pulse بتتنفذ مرة واحدة بس طول الـ lifetime — كل `triggered_count++` بعد الأولى return 0 بدون ما يتصل بالـ hardware.

**Parameters:**
- `rstc`: الـ reset control. لو `NULL` → return 0 (optional reset pattern).

**Return:** `0` نجاح، `-ENOTSUPP` لو الـ ops مش موجودة، `-EPERM` لو exclusive وmش acquired، `-EINVAL` لو IS_ERR.

**Key details:**
- لو `rstc->array` → يفوّض لـ `reset_control_array_reset`.
- للـ shared: يتحقق إن `deassert_count == 0` — ممنوع تخلط بين `reset()` و`assert/deassert` على نفس الـ shared line.
- لو الـ ops->reset فشل، بيعمل `atomic_dec` على `triggered_count` (rollback).
- **Caller:** consumer drivers في `probe()` أو runtime.

```
reset_control_reset(rstc):
    if NULL → return 0
    if array → delegate to array_reset
    if !ops->reset → -ENOTSUPP
    if shared:
        WARN_ON(deassert_count != 0)   // can't mix with assert/deassert
        if ++triggered_count != 1 → return 0  // already triggered
    else:
        if !acquired → -EPERM
    ret = ops->reset(rcdev, id)
    if shared && ret → atomic_dec(triggered_count)  // rollback
    return ret
```

---

#### `reset_control_rearm`

```c
int reset_control_rearm(struct reset_control *rstc);
```

يسمح لـ shared reset control إنه يتعمل trigger تاني. بيعمل `atomic_dec` على `triggered_count` عشان يرجع الـ counter لـ 0، فالـ `reset_control_reset` الجاية هتنفّذ الـ hardware reset فعلاً.

**Parameters:**
- `rstc`: الـ reset control. `NULL` → return 0.

**Return:** `0` نجاح، `-EINVAL` لو IS_ERR أو `deassert_count != 0`، `-EPERM` لو exclusive ومش acquired.

**Key details:**
- لو `triggered_count` وقع تحت 0 بعد الـ dec → `WARN_ON` (unbalanced call).
- لازم تكون calls متوازنة مع `reset_control_reset`.
- **Caller:** consumer يريد re-trigger لـ shared reset.

---

#### `reset_control_assert`

```c
int reset_control_assert(struct reset_control *rstc);
```

يثبّت خط الـ reset في حالة assert (الجهاز في reset). للـ exclusive بيستدعي `ops->assert` مباشرة. للـ shared يعمل `deassert_count--` وبيستدعي `ops->assert` بس لو الـ count وصل 0 (آخر consumer يعمل assert).

**Parameters:**
- `rstc`: الـ reset control.

**Return:** `0` نجاح أو no-op (shared مع consumers تانيين)، `-ENOTSUPP` لو exclusive وops مش موجودة، `-EPERM` لو exclusive ومش acquired، `-EINVAL` لو IS_ERR أو mixing مع `reset()`.

**Key details:**
- للـ shared: `triggered_count` لازم يكون 0 — ممنوع تخلط مع `reset()` API.
- للـ shared: لو `ops->assert` مش موجودة → return 0 بدون error (valid لـ self-deasserting).
- للـ exclusive: لو `ops->assert` مش موجودة → `-ENOTSUPP` (لأنه مش مضمون الـ assert state).
- **Locking:** لا يوجد lock داخلي — يعتمد على الـ atomic ops.

```
reset_control_assert(rstc):
    if array → delegate
    if shared:
        WARN_ON(triggered_count != 0)
        WARN_ON(deassert_count == 0)
        if --deassert_count != 0 → return 0  // others still deasserted
        if !ops->assert → return 0  // self-deasserting, OK
    else:
        if !ops->assert → -ENOTSUPP
        if !acquired → WARN + -EPERM
    return ops->assert(rcdev, id)
```

---

#### `reset_control_deassert`

```c
int reset_control_deassert(struct reset_control *rstc);
```

يُحرّر خط الـ reset (الجهاز يشتغل). للـ shared يعمل `deassert_count++` وبيستدعي `ops->deassert` بس لو الـ count وصل 1 (أول consumer يعمل deassert). لو `ops->deassert` مش موجودة → return 0 (self-deasserting reset بيكون deasserted by default).

**Parameters:**
- `rstc`: الـ reset control.

**Return:** `0` نجاح، `-EPERM` لو exclusive ومش acquired، `-EINVAL` لو IS_ERR أو mixing.

**Key details:**
- للـ shared: `triggered_count` لازم 0.
- الـ symmetry مع assert مهمة — كل deassert لازم يقابله assert.
- **Caller:** consumer في `probe()` بعد clock setup.

---

#### `reset_control_status`

```c
int reset_control_status(struct reset_control *rstc);
```

يقرأ حالة خط الـ reset من الـ hardware. بيفوّض لـ `ops->status`.

**Parameters:**
- `rstc`: الـ reset control. لا يقبل arrays.

**Return:** موجب لو asserted، 0 لو deasserted، `-ENOTSUPP` لو مفيش `ops->status`، `-EINVAL` لو array أو IS_ERR.

**Caller:** debugging أو drivers محتاجة تتحقق من الـ state قبل تشغيل الجهاز.

---

### Group 3: Bulk Runtime Control

الـ bulk APIs بتتعامل مع `reset_control_bulk_data[]` — كل عنصر فيه `id` (string) و`rstc` (pointer للـ control). الـ bulk functions بتوفّر **rollback تلقائي** لو فشل أي عنصر.

---

#### `reset_control_bulk_reset`

```c
int reset_control_bulk_reset(int num_rstcs,
                             struct reset_control_bulk_data *rstcs);
```

يعمل `reset_control_reset` على كل عنصر بالترتيب. لو فشل عنصر بيوقف ويرجع الـ error — **مفيش rollback** لأن `reset()` هو one-shot operation.

**Parameters:**
- `num_rstcs`: عدد العناصر.
- `rstcs`: مصفوفة الـ bulk data.

**Return:** `0` أو أول error.

---

#### `reset_control_bulk_assert`

```c
int reset_control_bulk_assert(int num_rstcs,
                              struct reset_control_bulk_data *rstcs);
```

يعمل assert بالترتيب. لو فشل عنصر i بيعمل deassert على كل العناصر 0..i-1 (rollback كامل).

**Return:** `0` أو error.

---

#### `reset_control_bulk_deassert`

```c
int reset_control_bulk_deassert(int num_rstcs,
                                struct reset_control_bulk_data *rstcs);
```

يعمل deassert **بالترتيب العكسي** (من `num_rstcs-1` لـ `0`). ده مهم لـ power sequencing. لو فشل عنصر i بيعمل assert على العناصر i+1..num_rstcs-1 (rollback).

**Key detail:** الترتيب العكسي مش صدفة — بيضمن إن آخر جهاز اتـ deassert هو أول جهاز يتـ assert في حالة الـ rollback.

---

### Group 4: Acquire / Release

الـ `acquire/release` mechanism موجود لـ use case محدد: exclusive reset يحتاج يتشارك بين multiple consumers بشكل **متسلسل** (مش concurrent). الـ consumer الأول يـ `get` الـ control كـ `exclusive_released` (يعني مش acquired افتراضياً)، وبعدين لو محتاج يتحكم يعمل `acquire`، وبعد ما يخلص `release` عشان consumer تاني يقدر يـ `acquire`.

---

#### `reset_control_acquire`

```c
int reset_control_acquire(struct reset_control *rstc);
```

يحجز exclusive access للـ reset control. يدور في `rcdev->reset_control_head` على أي `reset_control` تانية بنفس الـ id — لو لقى واحدة `acquired` يرجع `-EBUSY`.

**Parameters:**
- `rstc`: الـ reset control.

**Return:** `0` نجاح (أو لو already acquired)، `-EBUSY` لو consumer تاني حاجز، `-EINVAL` لو IS_ERR.

**Key details:**
- بياخد `reset_list_mutex` عشان يمشي على الـ list.
- للـ arrays: يفوّض لـ `reset_control_array_acquire` اللي يعمل rollback لو فشل.
- **Caller:** consumer قبل ما يعمل assert/deassert على released exclusive reset.

```
reset_control_acquire(rstc):
    mutex_lock(reset_list_mutex)
    if rstc->acquired → unlock, return 0
    for each rc in rcdev->reset_control_head:
        if rc != rstc && rc->id == rstc->id && rc->acquired:
            unlock, return -EBUSY
    rstc->acquired = true
    mutex_unlock
    return 0
```

---

#### `reset_control_release`

```c
void reset_control_release(struct reset_control *rstc);
```

يضبط `rstc->acquired = false` ببساطة. للـ arrays يفوّض لـ `reset_control_array_release`.

**Key details:** مش بياخد lock — الـ `acquired` flag single-writer في هذا السياق.

---

### Group 5: Get / Put — Lifecycle Management

هذا الـ group هو قلب الـ framework — بيدير إنشاء وتدمير الـ `reset_control` instances.

---

#### `__reset_control_get_internal`

```c
static struct reset_control *
__reset_control_get_internal(struct reset_controller_dev *rcdev,
                             unsigned int index,
                             enum reset_control_flags flags);
```

الـ function الجوهرية لإنشاء أو إعادة استخدام `reset_control`. بيمشي على `rcdev->reset_control_head`:
- لو لقى existing rstc بنفس الـ id:
  - لو الاتنين shared → `kref_get` ويرجع الموجود (sharing allowed).
  - لو أحدهم exclusive ومش released → `WARN + -EBUSY`.
  - لو exclusive released (`!acquired`) → يخرج من الـ loop وينشئ جديد (multiple exclusive released allowed).
- لو ملقاش → ينشئ rstc جديد بـ `kzalloc`، يعمل `try_module_get` على الـ owner، `get_device` على الـ rcdev->dev، و`kref_init`.

**Parameters:**
- `rcdev`: الـ controller.
- `index`: رقم الـ reset line (بعد الـ xlate).
- `flags`: SHARED و/أو ACQUIRED bits فقط (OPTIONAL وDEASSERTED متفلّترين قبل الاستدعاء).

**Return:** pointer لـ `reset_control` أو `ERR_PTR`.

**Key details:**
- `lockdep_assert_held(&reset_list_mutex)` — لازم يُستدعى تحت الـ lock.
- `try_module_get` يمنع unload الـ module أثناء الاستخدام.
- `get_device` يمنع device teardown.

---

#### `__reset_control_release` (kref callback)

```c
static void __reset_control_release(struct kref *kref);
```

الـ cleanup callback لـ `kref_put`. يعمل `module_put`، `list_del`، `put_device`، `kfree`.

**Key details:** `lockdep_assert_held(&reset_list_mutex)` — يُستدعى من داخل الـ lock دايماً.

---

#### `__reset_control_put_internal`

```c
static void __reset_control_put_internal(struct reset_control *rstc);
```

Wrapper بسيط يستدعي `kref_put` مع `__reset_control_release` كـ callback. يتجاهل `NULL` وـ `ERR_PTR`.

**Key details:** `lockdep_assert_held(&reset_list_mutex)`.

---

#### `__of_reset_control_get`

```c
struct reset_control *
__of_reset_control_get(struct device_node *node, const char *id,
                       int index, enum reset_control_flags flags);
```

الـ entry point الرئيسي للـ OF-based reset lookup. يمشي الخطوات:
1. لو `id` موجود: `of_property_match_string` على `reset-names` property.
2. `of_parse_phandle_with_args` على `resets` property.
3. لو فشلت (مفيش `resets` property) وـ `CONFIG_RESET_GPIO` enabled: يجرب `reset-gpios` property ويستدعي `__reset_add_reset_gpio_device`.
4. `mutex_lock` ثم `__reset_find_rcdev`.
5. لو ملقاش rcdev → `-EPROBE_DEFER` (يعني الـ controller لسه ما اتسجلش).
6. يتحقق من `args_count == of_reset_n_cells`.
7. يستدعي `rcdev->of_xlate` للحصول على `rstc_id` العددي.
8. يستدعي `__reset_control_get_internal`.

**Parameters:**
- `node`: الـ DT node للـ consumer.
- `id`: اسم الـ reset في `reset-names` أو `NULL` لو بتستخدم index.
- `index`: رقم الـ reset في الـ `resets` property.
- `flags`: مجموعة من `enum reset_control_flags`.

**Return:** `reset_control *` أو `ERR_PTR` أو `NULL` (لو optional ومش موجود).

**Key details:**
- `of_node_put(args.np)` يُستدعى دايماً في `out_put` (resource management).
- `lockdep_assert_not_held` في `__reset_add_reset_gpio_device` عشان الـ gpio device registration ممكن تـ trigger reset controller registration.
- **Caller:** `__reset_control_get` أو أي of_ wrapper.

```
__of_reset_control_get(node, id, index, flags):
    if id: index = of_property_match_string(node, "reset-names", id)
    ret = of_parse_phandle_with_args(node, "resets", ...)
    if ret && CONFIG_RESET_GPIO:
        ret = of_parse_phandle_with_args(node, "reset-gpios", ...)
        gpio_fallback = true
        __reset_add_reset_gpio_device(&args)
    mutex_lock(reset_list_mutex)
    rcdev = __reset_find_rcdev(&args, gpio_fallback)
    if !rcdev → -EPROBE_DEFER
    rstc_id = rcdev->of_xlate(rcdev, &args)
    rstc = __reset_control_get_internal(rcdev, rstc_id, flags)
    mutex_unlock
    of_node_put(args.np)
    return rstc
```

---

#### `__reset_control_get`

```c
struct reset_control *__reset_control_get(struct device *dev, const char *id,
                                          int index,
                                          enum reset_control_flags flags);
```

الـ generic get اللي بيختار الـ backend المناسب. حالياً لو `dev->of_node` موجود → `__of_reset_control_get`، غير كده → `NULL` (optional) أو `-ENOENT`.

**Key details:** يتحقق من `WARN_ON(shared && acquired)` — ده combination غير منطقي.

---

#### `__reset_control_bulk_get`

```c
int __reset_control_bulk_get(struct device *dev, int num_rstcs,
                             struct reset_control_bulk_data *rstcs,
                             enum reset_control_flags flags);
```

يعمل `__reset_control_get` على كل عنصر. لو فشل عنصر i: يعمل `mutex_lock` ويعمل `__reset_control_put_internal` على 0..i-1 (rollback).

---

#### `reset_control_put`

```c
void reset_control_put(struct reset_control *rstc);
```

يُحرر `reset_control` واحد. للـ arrays → `reset_control_array_put`. غيره → `mutex_lock` + `__reset_control_put_internal`.

**Key details:** يتجاهل `NULL` وـ `ERR_PTR` بأمان.

---

#### `reset_control_bulk_put`

```c
void reset_control_bulk_put(int num_rstcs,
                            struct reset_control_bulk_data *rstcs);
```

يعمل `__reset_control_put_internal` على كل العناصر تحت `reset_list_mutex` واحد (أكفأ من استدعاء `reset_control_put` على كل عنصر لأن الـ lock يُأخذ مرة واحدة).

---

#### `__devm_reset_control_get`

```c
struct reset_control *
__devm_reset_control_get(struct device *dev, const char *id, int index,
                         enum reset_control_flags flags);
```

النسخة المُدارة (devres) من `__reset_control_get`. بيختار الـ release callback بناءً على `DEASSERTED` flag:
- لو `DEASSERTED`: بيستدعي `reset_control_deassert` فوراً، وعند الـ detach يستدعي `reset_control_assert` ثم `reset_control_put`.
- غيره: عند الـ detach يستدعي `reset_control_put` فقط.

**Key detail:** يشيل الـ `DEASSERTED` bit من الـ flags قبل ما يمرّرها لـ `__reset_control_get` لأن الـ internal functions مش بتعرف هذا الـ bit.

---

#### `__devm_reset_control_bulk_get`

```c
int __devm_reset_control_bulk_get(struct device *dev, int num_rstcs,
                                  struct reset_control_bulk_data *rstcs,
                                  enum reset_control_flags flags);
```

نفس منطق `__devm_reset_control_get` بس للـ bulk. الـ devres تخزّن `reset_control_bulk_devres` struct فيها `num_rstcs` والـ `rstcs` pointer.

---

### Group 6: Array APIs

الـ array APIs بتتعامل مع مجموعة resets كـ unit واحدة بدون اهتمام بالترتيب. الـ `reset_control_array` بتحتوي `reset_control base` عشان تبان كـ ordinary `reset_control` للـ API (اللي يتحقق من `rstc->array` flag قبل أي operation).

---

#### `of_reset_control_array_get`

```c
struct reset_control *
of_reset_control_array_get(struct device_node *np,
                           enum reset_control_flags flags);
```

يحسب عدد الـ resets في الـ DT node بـ `of_reset_control_get_count`، يعمل `kzalloc` للـ `reset_control_array` بحجم مناسب (يستخدم `struct_size` للـ flexible array)، ثم يعمل `__of_reset_control_get` على كل reset بالترتيب. لو فشل reset i بعمل `__reset_control_put_internal` على 0..i-1 (rollback).

**Parameters:**
- `np`: الـ DT node.
- `flags`: shared/exclusive/optional.

**Return:** `&resets->base` (مُعاد cast) أو `ERR_PTR` أو `NULL` (optional).

**Key detail:** `resets->base.array = true` هو الـ marker اللي بيخلي الـ API functions تعرف إنها تتعامل مع array.

---

#### `devm_reset_control_array_get`

```c
struct reset_control *
devm_reset_control_array_get(struct device *dev,
                             enum reset_control_flags flags);
```

نسخة managed من `of_reset_control_array_get`. الـ release callback هو `devm_reset_control_release` اللي يستدعي `reset_control_put` اللي بيكتشف الـ array ويستدعي `reset_control_array_put`.

---

#### `reset_control_get_count`

```c
int reset_control_get_count(struct device *dev);
```

يُرجع عدد الـ reset lines المحددة في `resets` property. مفيد للـ drivers اللي محتاجة تعرف عدد الـ resets قبل ما تعمل array get.

**Return:** عدد موجب أو `-ENOENT` لو مفيش `of_node` أو مفيش `resets` property.

---

### Group 7: GPIO Reset Backend

لما device tree node بتحتوي `reset-gpios` بدل `resets`، الـ framework بيخلق auxiliary device بـ name `reset.gpio.N` يتعامل معاه driver تاني (reset-gpio driver). الـ mechanism ده بيسمح بـ GPIO-based reset controllers بدون تعديل في consumer driver.

---

#### `__reset_add_reset_gpio_device`

```c
static int __reset_add_reset_gpio_device(const struct of_phandle_args *args);
```

ده الـ function الأكتر تعقيداً في الـ file. يعمل:
1. يتحقق إن `args_count == 2` (GPIO number + flags).
2. يبحث عن الـ GPIO device بـ `gpio_device_find_by_fwnode`.
3. تحت `reset_gpio_lookup_mutex`: يتحقق لو الـ combination ده موجود بالفعل (idempotent).
4. يعمل `fwnode_create_software_node` بـ properties تحتوي `compatible = "reset-gpio"` وـ `reset-gpios` property.
5. يخصص ID من `reset_gpio_ida`.
6. يسجّل `reset_gpio_lookup` struct في القائمة.
7. يستدعي `reset_add_gpio_aux_device`.

**Key details:**
- `lockdep_assert_not_held(&reset_list_mutex)` — ده مهم جداً عشان الـ auxiliary device registration ممكن تـ trigger `reset_controller_register` اللي محتاجة الـ lock ده.
- الـ `rgpio_dev` memory ومحتوياتها **persistent** — مش بتتحرر أبداً (subsystem data).
- يعمل `of_node_get` إضافية عشان يحتفظ بـ reference طول الـ lifetime.
- حالياً بيدعم `#gpio-cells=2` فقط.

```
__reset_add_reset_gpio_device(args):
    if args_count != 2 → -ENOENT
    lockdep_assert_not_held(reset_list_mutex)  // critical!
    gdev = gpio_device_find_by_fwnode(args->np)
    guard(mutex)(reset_gpio_lookup_mutex)
    // check if already registered
    for each rgpio_dev in reset_gpio_lookup_list:
        if of_phandle_args_equal → return 0
    // create software node with properties
    id = ida_alloc(reset_gpio_ida)
    rgpio_dev = kzalloc(...)
    rgpio_dev->swnode = fwnode_create_software_node(properties, NULL)
    reset_add_gpio_aux_device(parent, swnode, id, &args)
    list_add(rgpio_dev, reset_gpio_lookup_list)
```

---

#### `reset_add_gpio_aux_device`

```c
static int reset_add_gpio_aux_device(struct device *parent,
                                     struct fwnode_handle *swnode,
                                     int id, void *pdata);
```

ينشئ `auxiliary_device` بـ `name = "gpio"` (يظهر كـ `reset.gpio.N` في sysfs)، يضبط الـ `platform_data` و`swnode`، ثم يستدعي `auxiliary_device_init` ثم `__auxiliary_device_add` مع module name `"reset"`.

**Key details:** الـ `release` callback بيعمل `kfree(adev)` فقط — الـ swnode والـ lookup entry بتبقى.

---

#### `__reset_find_rcdev`

```c
static struct reset_controller_dev *
__reset_find_rcdev(const struct of_phandle_args *args, bool gpio_fallback);
```

يبحث في `reset_controller_list` عن الـ rcdev المطابق. لو `gpio_fallback = true` يقارن `rcdev->of_args` بـ `of_phandle_args_equal`، غيره يقارن `rcdev->of_node` بـ `args->np`.

**Key details:** `lockdep_assert_held(&reset_list_mutex)`.

---

#### `of_reset_simple_xlate`

```c
static int of_reset_simple_xlate(struct reset_controller_dev *rcdev,
                                 const struct of_phandle_args *reset_spec);
```

الـ default xlate function — بتتحقق إن `reset_spec->args[0] < rcdev->nr_resets` وترجعه كـ ID مباشرة. مناسبة لكل الـ controllers ذات 1:1 mapping.

**Return:** الـ reset line ID (0..nr_resets-1) أو `-EINVAL`.

---

#### `rcdev_name`

```c
static const char *rcdev_name(struct reset_controller_dev *rcdev);
```

Helper للـ logging — يرجع أفضل اسم متاح للـ rcdev: اسم الـ device، اسم الـ OF node، أو اسم الـ GPIO OF node.

---

### ملاحظات تصميمية مهمة

#### الـ Locking Strategy

```
reset_list_mutex:
    ├── يحمي reset_controller_list (global)
    ├── يحمي rcdev->reset_control_head (per-rcdev)
    └── يجب أن يُأخذ من process context

reset_gpio_lookup_mutex:
    └── يحمي reset_gpio_lookup_list فقط
        (منفصل عن reset_list_mutex عمداً)

atomic_t deassert_count / triggered_count:
    └── lock-free counters للـ shared resets
```

#### الـ Reference Counting

```
reset_control lifecycle:
    __reset_control_get_internal → kref_init(1) + get_device + try_module_get
    إضافة shared consumer     → kref_get
    reset_control_put         → kref_put
        └─ __reset_control_release → module_put + list_del + put_device + kfree
```

#### قاعدة الـ Exclusive vs Shared

| | Exclusive (acquired) | Exclusive (released) | Shared |
|---|---|---|---|
| عدد consumers | 1 | متعدد (متسلسل) | متعدد (متزامن) |
| reset() | ✓ | ✓ (بعد acquire) | ✓ (مرة واحدة) |
| assert/deassert | ✓ | ✓ (بعد acquire) | ✓ (بـ counter) |
| خلط reset() مع assert/deassert | ✗ | ✗ | ✗ |
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs entries

الـ Reset Controller framework مش بيعمل entries في debugfs بشكل افتراضي، بس ممكن تعمل واحدة في الـ controller driver نفسه. الطريقة العملية هي قراءة state عن طريق `/sys` أو عن طريق الـ `status` op مباشرة.

```bash
# mount debugfs لو مش موجود
mount -t debugfs none /sys/kernel/debug

# شوف كل الـ auxiliary devices اللي اتسجلت (reset-gpio)
ls /sys/kernel/debug/

# لو عندك controller بيعمل debugfs entry خاص بيه
ls /sys/kernel/debug/reset/
```

---

#### 2. sysfs entries

الـ reset framework بيظهر الـ controller device نفسه في sysfs عن طريق الـ `struct device` اللي بيتحط في `rcdev->dev`.

```bash
# شوف كل الـ reset controllers المسجلين كـ auxiliary devices
ls /sys/bus/auxiliary/devices/ | grep reset

# شوف الـ reset-gpio auxiliary devices
ls /sys/bus/auxiliary/devices/ | grep "reset.gpio"

# معلومات الـ device
cat /sys/bus/auxiliary/devices/reset.gpio.0/uevent

# الـ driver المرتبط
ls -la /sys/bus/auxiliary/devices/reset.gpio.0/driver

# الـ parent device
cat /sys/bus/auxiliary/devices/reset.gpio.0/../uevent
```

---

#### 3. ftrace — tracepoints وـ events

الـ reset subsystem مش عنده tracepoints جاهزة في core.c، بس ممكن تتبعه عن طريق function tracing.

```bash
# enable function tracing للـ reset core
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo function > /sys/kernel/debug/tracing/current_tracer

# فلتر على functions الـ reset فقط
echo 'reset_control_*' > /sys/kernel/debug/tracing/set_ftrace_filter
echo '__of_reset_control_get' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo '__reset_control_get_internal' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'reset_controller_register' >> /sys/kernel/debug/tracing/set_ftrace_filter

echo 1 > /sys/kernel/debug/tracing/tracing_on

# شغل الـ device الـ target
# بعدين اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace

# وقف
echo 0 > /sys/kernel/debug/tracing/tracing_on
echo nop > /sys/kernel/debug/tracing/current_tracer
```

**استخدم function_graph لو محتاج ترتيب الـ call stack:**

```bash
echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 'reset_control_reset' > /sys/kernel/debug/tracing/set_graph_function
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغل الـ device ...
cat /sys/kernel/debug/tracing/trace
```

**مثال output:**

```
# TASK-PID   CPU#  ||||  TIMESTAMP  FUNCTION
# | | | | | |
  probe-0    [000] ....  1.234567: reset_controller_register <-platform_driver_probe
  probe-0    [000] ....  1.234600: reset_control_deassert <-my_driver_probe
  probe-0    [000] ....  1.234610: __reset_control_get_internal <-__of_reset_control_get
```

---

#### 4. printk وـ dynamic debug

```bash
# تفعيل dynamic debug لكل الـ reset subsystem (لو في pr_debug في drivers)
echo 'module reset_controller +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل لملف معين
echo 'file drivers/reset/core.c +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل مع line numbers وـ function names
echo 'file drivers/reset/core.c +pflm' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ dynamic debug المفعلة
cat /sys/kernel/debug/dynamic_debug/control | grep reset

# تابع الـ kernel log
dmesg -w | grep -i reset
journalctl -k -f | grep -i reset
```

---

#### 5. Kernel config options للـ debugging

| Config Option | الغرض |
|---|---|
| `CONFIG_RESET_CONTROLLER` | لازم يكون enabled أصلاً |
| `CONFIG_RESET_GPIO` | يفعل دعم reset-gpio fallback |
| `CONFIG_DEBUG_KERNEL` | يفتح باب الـ debug options |
| `CONFIG_LOCKDEP` | يكتشف lockdep violations (موجودة في `lockdep_assert_held`) |
| `CONFIG_PROVE_LOCKING` | يفعل الـ lockdep proving |
| `CONFIG_DEBUG_MUTEXES` | يكتشف mutex misuse |
| `CONFIG_KREF_DEBUG` | يساعد في تتبع الـ kref leaks |
| `CONFIG_DEBUG_OBJECTS` | يكتشف use-after-free في الـ objects |
| `CONFIG_FTRACE` | يفعل function tracing |
| `CONFIG_DYNAMIC_DEBUG` | يفعل dynamic debug printing |
| `CONFIG_OF_DYNAMIC` | مهم لو بتعمل DT manipulation |
| `CONFIG_AUXILIARY_BUS` | لازم يكون enabled للـ reset-gpio devices |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_RESET|CONFIG_LOCKDEP|CONFIG_DEBUG_MUTEX'
# أو
grep -E 'CONFIG_RESET|CONFIG_LOCKDEP' /boot/config-$(uname -r)
```

---

#### 6. devlink وـ tools خاصة بالـ subsystem

الـ reset framework مش عنده devlink interface، بس في tools مفيدة:

```bash
# اعرض كل الـ auxiliary bus devices
ls -la /sys/bus/auxiliary/devices/

# اعرض الـ reset controls المطلوبة لـ device معين (عن طريق DT)
# أولاً، اعرف الـ device node
ls /sys/bus/platform/devices/

# اعرض الـ resets property في DT للـ device
cat /sys/firmware/devicetree/base/soc/my-device@1234/resets

# استخدم dtc لعرض الـ DT بالكامل
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | grep -A5 "resets"

# تحقق من الـ reset-names
find /sys/firmware/devicetree/base -name "reset-names" -exec xxd {} \;

# لو عندك reset-gpio
gpioinfo  # يعرض GPIO lines وحالتها
gpioget gpiochip0 5  # يقرأ GPIO معين (مثلاً GPIO5 هو الـ reset line)
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|---|---|---|
| `reset %s (ID: %u) is not acquired` | الـ driver بيحاول `assert`/`deassert` بدون ما يعمل `acquire` أول | استدعِ `reset_control_acquire()` قبل الاستخدام، أو استخدم `RESET_CONTROL_EXCLUSIVE` بدل `RELEASED` |
| `WARNING: ... IS_ERR(rstc)` | تم تمرير `ERR_PTR` لـ reset function | تحقق من قيمة الـ return من `reset_control_get_*()` |
| `WARNING: deassert_count != 0` | بيحاول `reset()` على shared reset وهو لسه deasserted | اتصل بـ `reset_control_assert()` أول قبل `reset()` |
| `WARNING: triggered_count != 0` | بيحاول `assert()` على shared reset بعد `reset()` بدون `rearm` | استخدم `reset_control_rearm()` أو استخدم `(de)assert` بدل `reset` |
| `WARNING: atomic_dec triggered_count < 0` | عدد `rearm` أكتر من `reset` | تأكد من symmetry — كل `reset_control_reset()` يقابله `reset_control_rearm()` |
| `-EBUSY` من `reset_control_acquire()` | controller تاني acquired نفس الـ reset line | انتظر لحد ما الـ consumer التاني يعمل `release` |
| `-EPERM` | exclusive reset مش acquired | نسيت `acquire()` أو استخدمت `RELEASED` variant |
| `-EPROBE_DEFER` | الـ reset controller device لسه ما اتسجلش | طبيعي أثناء الـ boot، الـ driver هيتجرب مرة تانية |
| `-ENOENT` | مش لاقي `resets` property في DT | تحقق من الـ DT binding — الـ device مش عنده reset line |
| `-EINVAL` في `of_reset_simple_xlate` | الـ reset index أكبر من `nr_resets` | تحقق من `#reset-cells` وعدد الـ resets في DT |
| `-ENOENT` في `__reset_add_reset_gpio_device` | `#gpio-cells != 2` | الـ GPIO controller مش بيدعم الـ format المطلوب |
| `reset-gpio code does not support GPIO flags` | الـ GPIO flags في DT أكتر من `GPIO_ACTIVE_LOW` | استخدم فقط flags 0 أو 1 في الـ DT |
| `-ENOTSUPP` من `reset_control_status` | الـ controller مش بينفذ `.status` op | الـ driver ناقص implementation أو الـ hardware مش بيدعمه |

---

#### 8. نقاط استراتيجية لـ dump_stack() وـ WARN_ON()

**في `reset_controller_register()` لتتبع من سجّل الـ controller:**

```c
int reset_controller_register(struct reset_controller_dev *rcdev)
{
    /* Add here to trace registration source */
    pr_debug("reset_controller_register: %s, nr_resets=%u\n",
             rcdev_name(rcdev), rcdev->nr_resets);
    dump_stack(); /* temporary — remove after debugging */
    ...
}
```

**في `__reset_control_get_internal()` لتتبع الـ consumers:**

```c
static struct reset_control *
__reset_control_get_internal(struct reset_controller_dev *rcdev,
                              unsigned int index, enum reset_control_flags flags)
{
    /* detect unexpected shared/exclusive conflicts */
    WARN_ON(!shared && list_empty(&rcdev->reset_control_head) == false);
    ...
}
```

**نقاط مهمة موجودة فعلاً في الكود (WARN_ON موجودة):**

```c
/* core.c line 334 — في reset_control_reset() */
if (WARN_ON(IS_ERR(rstc)))   /* rstc = ERR_PTR بيعدي للـ function */

/* core.c line 344 — shared reset مع deassert_count != 0 */
if (WARN_ON(atomic_read(&rstc->deassert_count) != 0))

/* core.c line 227 — في array_rearm */
if (WARN_ON(atomic_read(&rstc->deassert_count) != 0))

/* core.c line 479 — assert على exclusive غير acquired */
WARN(1, "reset %s (ID: %u) is not acquired\n", rcdev_name(rstc->rcdev), rstc->id);
```

**أضف WARN_ON في الـ controller driver نفسه:**

```c
/* في ops->reset() — تأكد إن الـ hardware استجاب */
static int my_reset_reset(struct reset_controller_dev *rcdev, unsigned long id)
{
    u32 val;
    writel(BIT(id), base + RESET_REG);
    udelay(1);
    val = readl(base + RESET_STATUS_REG);
    WARN_ON(val & BIT(id)); /* still in reset after deassert? */
    return 0;
}
```

---

### Hardware Level

#### 1. التحقق إن الـ hardware state يطابق الـ kernel state

الـ kernel يتتبع الـ reset state عبر `deassert_count` و`triggered_count` في `struct reset_control`، لكن الـ hardware state الفعلي يُقرأ من الـ controller registers.

```bash
# اقرأ الـ reset status عن طريق الـ .status op (لو الـ driver ينفذها)
# من داخل الـ driver (كـ debugfs entry مثلاً):
# reset_control_status(rstc) -- يرجع 1 لو asserted, 0 لو deasserted

# للـ GPIO-based resets — قرأ حالة الـ GPIO مباشرة
gpioinfo 2>/dev/null | grep -i reset

# تحقق من الـ GPIO value الحالي
cat /sys/class/gpio/gpio${N}/value   # 0 = deasserted (أو العكس حسب polarity)
cat /sys/class/gpio/gpio${N}/direction
```

**مقارنة:**

```
Kernel state (deassert_count):  1  → kernel thinks: deasserted
GPIO physical value:             0  → active-low GPIO → actually deasserted ✓

Kernel state (deassert_count):  0  → kernel thinks: asserted
GPIO physical value:             0  → active-low GPIO → actually deasserted ✗ MISMATCH!
```

---

#### 2. Register dump بـ devmem2 وـ /dev/mem

```bash
# تأكد إن CONFIG_DEVMEM=y أو استخدم devmem2
# اعرف عنوان الـ reset controller registers من DT أو datasheet

# مثال: SoC reset controller على عنوان 0xFE00_1000
devmem2 0xFE001000 w      # قرأ 32-bit word
devmem2 0xFE001004 w      # الـ register التاني

# أو عبر /dev/mem مع dd
dd if=/dev/mem bs=4 count=16 skip=$((0xFE001000/4)) 2>/dev/null | xxd

# io utility (من package ioport)
io -4 -r 0xFE001000

# لو MMIO
# استخدم /proc/iomem لتحديد العنوان
cat /proc/iomem | grep -i reset
```

**مثال output من `/proc/iomem`:**

```
fe001000-fe001fff : my-soc-reset-controller
  fe001000-fe00103f : reset-registers
```

---

#### 3. Logic Analyzer وـ Oscilloscope

**نقاط القياس:**

```
Reset Line (GPIO or dedicated pin):
  ┌─────────────────────────────────────────────┐
  │  Measure:                                   │
  │  1. Assert pulse width (min per datasheet)  │
  │  2. Deassert rise time                      │
  │  3. Setup time before first clock           │
  │  4. Active level: HIGH or LOW?              │
  └─────────────────────────────────────────────┘

Timing Diagram (active-low reset):
        assert          deassert
VCC ────┐               ┌────────────────
        │               │
GND     └───────────────┘
        │←  pulse_width →│
        │                │← setup_time →│clock_start
```

**نصائح عملية:**

- Channel 1: الـ reset line نفسها
- Channel 2: الـ first clock signal بعد الـ deassert
- Channel 3: الـ power rail لو في power sequencing
- اضبط trigger على falling edge للـ assert (active-low) أو rising edge للـ deassert
- قيس الـ pulse width وقارنه بالـ datasheet minimum

```bash
# ابعت trigger من الـ kernel لتزامن الـ scope
# أضف في الـ driver مؤقتاً:
gpio_set_value(debug_gpio, 1); /* scope trigger */
reset_control_assert(rstc);
udelay(100);
reset_control_deassert(rstc);
gpio_set_value(debug_gpio, 0); /* scope trigger off */
```

---

#### 4. مشاكل الـ hardware الشائعة وـ patterns الـ kernel log

| المشكلة | Pattern في الـ log | التشخيص |
|---|---|---|
| الـ reset line مش موصلة | Device probe fails, `-EPROBE_DEFER` مستمر | تحقق من continuity الـ PCB |
| polarity عكسية | الـ device مش بيعمل رغم deassert | قيس بالـ scope، راجع `GPIO_ACTIVE_LOW` في DT |
| pulse width قصير جداً | Device يعمل أحياناً ويفشل أحياناً (intermittent) | قيس بالـ scope، أضف `udelay()` في الـ driver |
| power sequencing خاطئ | Device يفشل رغم reset صحيح | تأكد إن الـ power rail مستقر قبل deassert |
| shared reset conflict | `WARN_ON` في الـ log: `deassert_count != 0` | راجع كل consumers للـ shared reset |
| GPIO مش configured | `gpioinfo` يظهر الـ GPIO غير مأخوذ | تحقق من DT `reset-gpios` property |
| controller driver ما اتسجلش | `-EPROBE_DEFER` في الـ consumer | تأكد من `reset_controller_register()` اتنادت |

---

#### 5. Device Tree debugging

```bash
# تحقق من الـ DT الفعلي المحمل في الـ kernel
ls /sys/firmware/devicetree/base/

# تحقق من الـ reset controller node
# مثال: CCF reset controller
find /sys/firmware/devicetree/base -name "#reset-cells" -exec sh -c 'echo "$1:"; xxd "$1"' _ {} \;

# تحقق من الـ resets property في الـ consumer device
find /sys/firmware/devicetree/base -name "resets" | while read f; do
    echo "=== $f ==="
    xxd "$f"
done

# تحقق من reset-names
find /sys/firmware/devicetree/base -name "reset-names" | while read f; do
    echo "=== $f ==="
    cat "$f" | tr '\0' '\n'
done

# compile الـ DT وشوفه بصيغة مقروءة
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -B5 -A10 "resets"

# تحقق من الـ phandle يشاور على الـ controller الصح
# في الـ consumer node:
#   resets = <&rst_ctrl 5>;  ← phandle يشاور على &rst_ctrl
# في الـ controller node:
#   rst_ctrl: reset-controller { #reset-cells = <1>; }
```

**مثال DT صحيح:**

```dts
/* Reset Controller */
rst: reset-controller@fe001000 {
    compatible = "myvendor,reset-ctrl";
    reg = <0xfe001000 0x1000>;
    #reset-cells = <1>;          /* index فقط */
};

/* Consumer Device */
my_device: ethernet@ff800000 {
    compatible = "myvendor,eth";
    reg = <0xff800000 0x1000>;
    resets = <&rst 3>;           /* reset line رقم 3 */
    reset-names = "eth-reset";   /* optional name */
};
```

**تحقق من الـ phandle بالأرقام:**

```bash
# اعرف الـ phandle value للـ controller
cat /sys/firmware/devicetree/base/soc/reset-controller@fe001000/phandle | xxd
# مثال output: 0x00000023  (phandle = 0x23 = 35)

# اعرف قيمة الـ resets property في الـ consumer
cat /sys/firmware/devicetree/base/soc/ethernet@ff800000/resets | xxd
# مثال output: 00 00 00 23 00 00 00 03
#              └── phandle=0x23 ──┘ └── index=3 ─┘
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

**1. اعرض كل الـ reset controllers المسجلة:**

```bash
# عبر auxiliary bus
ls -la /sys/bus/auxiliary/devices/ | grep reset
```

**مثال output:**

```
lrwxrwxrwx 1 root root 0 Feb 24 10:00 reset.gpio.0 -> ../../../devices/platform/gpio@a000/reset.gpio.0
lrwxrwxrwx 1 root root 0 Feb 24 10:00 reset.gpio.1 -> ../../../devices/platform/gpio@b000/reset.gpio.1
```

**2. تتبع reset operations بالـ ftrace:**

```bash
#!/bin/bash
# reset_trace.sh — trace all reset operations
TRACE=/sys/kernel/debug/tracing

echo 0 > $TRACE/tracing_on
echo > $TRACE/trace
echo function > $TRACE/current_tracer
echo 'reset_control_*' > $TRACE/set_ftrace_filter
echo '__of_reset_control_get' >> $TRACE/set_ftrace_filter
echo 'reset_controller_register' >> $TRACE/set_ftrace_filter
echo 1 > $TRACE/tracing_on
echo "Tracing reset operations... press Ctrl+C to stop"
cat $TRACE/trace_pipe
```

**3. تحقق من حالة الـ GPIO reset lines:**

```bash
# اعرض كل الـ GPIO lines واسمها
gpioinfo 2>/dev/null | grep -i -E "reset|rst"
```

**مثال output:**

```
gpiochip0 - 32 lines:
    line  5: "eth-reset"  output active-low [used]
    line 12: "usb-reset"  output active-low [used]
```

**4. اقرأ reset line state:**

```bash
# عن طريق sysfs (legacy)
RESET_GPIO=5
echo $RESET_GPIO > /sys/class/gpio/export 2>/dev/null
cat /sys/class/gpio/gpio${RESET_GPIO}/value
# 0 = deasserted (active-low)  1 = asserted (active-low)

# عن طريق gpioget
gpioget gpiochip0 5
```

**5. تحقق من الـ lockdep (لو CONFIG_LOCKDEP=y):**

```bash
# شغل الـ kernel مع lockdep وابحث عن reset mutex violations
dmesg | grep -A20 "possible deadlock"
dmesg | grep -A5 "reset_list_mutex"
```

**6. تحقق من الـ kref leaks:**

```bash
# لو CONFIG_DEBUG_OBJECTS=y
dmesg | grep -i "ODEBUG"
# أي reset_control object ما اتحرش بيه صح هيظهر هنا
```

**7. تحقق من الـ reset-gpio device الـ auxiliary:**

```bash
# شوف الـ software node الـ properties
cat /sys/bus/auxiliary/devices/reset.gpio.0/properties 2>/dev/null

# أو عبر الـ fwnode
find /sys/firmware -name "*.gpio*" 2>/dev/null
```

**8. محاكاة reset من userspace (للاختبار):**

```bash
# لو الـ reset line هي GPIO
# assert (active-low → set to 1 = assert)
gpioset gpiochip0 5=1
sleep 0.001  # 1ms pulse
# deassert
gpioset gpiochip0 5=0

# أو عبر sysfs (legacy)
echo out > /sys/class/gpio/gpio5/direction
echo 1 > /sys/class/gpio/gpio5/value   # assert
sleep 0.001
echo 0 > /sys/class/gpio/gpio5/value   # deassert
```

**9. تحقق من أن `nr_resets` مظبوط:**

```bash
# ابحث عن الـ controller في kernel messages
dmesg | grep -i "reset.*controller\|reset.*register"
# مثال:
# [    2.345678] my-reset-ctrl fe001000.reset-controller: registered 16 resets
```

**10. script شامل للـ diagnosis:**

```bash
#!/bin/bash
# reset_diag.sh — comprehensive reset framework diagnosis

echo "=== Reset Controller Devices ==="
ls /sys/bus/auxiliary/devices/ 2>/dev/null | grep reset || echo "None found"

echo ""
echo "=== GPIO Reset Lines ==="
gpioinfo 2>/dev/null | grep -i -E "reset|rst" || echo "gpioinfo not available"

echo ""
echo "=== Device Tree Reset Properties ==="
find /sys/firmware/devicetree/base -name "resets" 2>/dev/null | while read f; do
    dev=$(dirname $f | xargs basename)
    printf "%-30s: " "$dev"
    xxd "$f" 2>/dev/null | head -1
done

echo ""
echo "=== Recent Reset-related Kernel Messages ==="
dmesg | grep -i -E "reset.*controller|WARN.*reset|reset.*acquire|reset.*assert" | tail -20

echo ""
echo "=== Kernel Config ==="
zcat /proc/config.gz 2>/dev/null | grep -E 'CONFIG_RESET|CONFIG_LOCKDEP' || \
grep -E 'CONFIG_RESET|CONFIG_LOCKDEP' /boot/config-$(uname -r) 2>/dev/null
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Gateway صناعي على AM62x — الـ UART مش بيشتغل بعد bring-up

#### العنوان
**الـ UART driver بيرجع `-EPROBE_DEFER` بشكل لا نهائي على AM62x industrial gateway**

#### السياق
بتشتغل على bring-up لـ industrial gateway بيشغل TI AM62x، المنتج مفروض يوصّل PLC devices عبر RS-485. الـ board custom، الـ DT مكتوب من الصفر بناءً على reference design. بعد boot الـ serial console شغال، بس الـ RS-485 UART (الـ `uart3`) مش بيظهر في `/dev/` خالص.

#### المشكلة
الـ driver بيسجّل الـ error التالي في `dmesg` بشكل متكرر:

```
[    4.123] 20480000.serial: deferred probe pending
[    4.234] 20480000.serial: deferred probe pending
```

مفيش `-EPROBE_DEFER` صريح في log الـ driver نفسه، لكنه بيحصل بسبب الـ reset controller.

#### التحليل
الـ UART driver بيعمل `devm_reset_control_get_exclusive()` اللي بتكلّم:

```c
// reset.h → __devm_reset_control_get → __reset_control_get → __of_reset_control_get
struct reset_control *__of_reset_control_get(struct device_node *node,
        const char *id, int index, enum reset_control_flags flags)
{
    // بيعمل parse للـ phandle من DT
    ret = of_parse_phandle_with_args(node, "resets", "#reset-cells", index, &args);
    ...
    mutex_lock(&reset_list_mutex);
    rcdev = __reset_find_rcdev(&args, gpio_fallback);
    if (!rcdev) {
        rstc = ERR_PTR(-EPROBE_DEFER);  // ← المشكلة هنا
        goto out_unlock;
    }
```

الـ `__reset_find_rcdev` بتدور في `reset_controller_list` على `rcdev` اللي `of_node` بتاعه بيطابق الـ phandle في DT. لو مش لاقياه → بترجع `-EPROBE_DEFER`.

السبب: الـ DT بيشاور على `prcm` node كـ reset controller، بس الـ driver بتاع الـ `ti,sci-reset` لسه مش probe اتمسجّلش عبر `reset_controller_register()` اللي بتعمل:

```c
int reset_controller_register(struct reset_controller_dev *rcdev)
{
    ...
    mutex_lock(&reset_list_mutex);
    list_add(&rcdev->list, &reset_controller_list);  // هنا بس بيبقى visible
    mutex_unlock(&reset_list_mutex);
    return 0;
}
```

#### الحل

**خطوة 1** — تحقق إن الـ reset controller node موجود صح في DT:

```bash
# على الـ target
cat /proc/device-tree/bus@f0000/prcm@4000/compatible
# المفروض ترجع: ti,am62-sci-reset أو مشابه
```

**خطوة 2** — تأكد إن الـ module اتحمّل:

```bash
lsmod | grep ti_sci
# لو مش موجود:
modprobe ti-sci-pm-domains
```

**خطوة 3** — لو الـ DT خاطئ، صلّح الـ phandle:

```dts
/* غلط — بيشاور على node مش موجود */
uart3: serial@20480000 {
    resets = <&wrong_prcm 42>;
};

/* صح */
uart3: serial@20480000 {
    compatible = "ti,am64-uart";
    reg = <0x20480000 0x100>;
    clocks = <&k3_clks 146 0>;
    resets = <&k3_reset 146 1>;  /* phandle صح + reset ID */
    reset-names = "uart";
};
```

#### الدرس المستفاد
الـ `-EPROBE_DEFER` من `__of_reset_control_get` مش دايمًا bug في الـ driver نفسه — غالبًا الـ reset controller provider لسه مش probe. ابدأ تحقيقك من `reset_controller_list` مش من الـ consumer driver.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ USB بيتعطل بعد suspend/resume

#### العنوان
**الـ USB ports بتوقف بعد أول suspend/resume cycle على H616 TV box**

#### السياق
بتشتغل على BSP لـ Android TV box بيستخدم Allwinner H616. الـ box بيدعم 4K streaming. بعد الشاشة تتحول لـ sleep mode وترجع، الـ USB keyboard والـ USB hub بيبطلوا يشتغلوا. الـ reboot بيحل المشكلة.

#### المشكلة
في `dmesg` بعد resume:

```
[  142.881] usb usb1-port2: cannot reset (err = -110)
[  142.882] usb 1-2: USB disconnect, device number 3
```

الـ error `-ETIMEDOUT` بييجي من الـ USB host controller reset operation. الـ engineer الأول ظن المشكلة في الـ PHY driver، بس المشكلة الحقيقية في الـ reset framework.

#### التحليل
الـ USB host driver على H616 بيستخدم reset shared لأن الـ USB controller والـ USB PHY بيشاركوا نفس الـ reset line في CRU (Clock Reset Unit):

```c
/* في الـ USB host driver */
rstc_host = reset_control_get_shared(dev, "usb-host");
rstc_phy  = reset_control_get_shared(dev, "usb-phy");
```

المشكلة في تسلسل suspend: الـ PHY driver بيعمل `reset_control_assert()` قبل الـ host driver يخلص تنظيفه. بس الـ framework بيتعامل مع الـ shared reset كالآتي في `reset_control_assert`:

```c
int reset_control_assert(struct reset_control *rstc)
{
    if (rstc->shared) {
        if (WARN_ON(atomic_read(&rstc->triggered_count) != 0))
            return -EINVAL;

        if (WARN_ON(atomic_read(&rstc->deassert_count) == 0))
            return -EINVAL;

        // بيـdeassert_count-- — لو وصل 0 بس بيعمل assert فعلي
        if (atomic_dec_return(&rstc->deassert_count) != 0)
            return 0;
        ...
    }
    return rstc->rcdev->ops->assert(rstc->rcdev, rstc->id);
}
```

المشكلة: الـ PHY driver عمل deassert مرة واحدة بس، والـ host driver عمل deassert مرتين (bug). لما الـ PHY assert → `deassert_count` بيوصل 1 مش 0 → الـ reset مش بيتعمل. لما الـ host بعدين يحاول assert → `deassert_count` بيبقى 0 → بيعمل assert فعلي في وقت غلط.

**أداة التشخيص:**

```bash
# شوف قيم atomic counters لو عندك debugfs
cat /sys/kernel/debug/reset/*/deassert_count 2>/dev/null

# أو فضّل kernel tracing
echo 1 > /sys/kernel/debug/tracing/events/reset/enable
echo 'reset_control_assert' > /sys/kernel/debug/tracing/set_ftrace_filter
```

#### الحل

في الـ USB host driver، لازم كل `deassert` يقابله `assert` واحدة بالظبط:

```c
static int h616_usb_probe(struct platform_device *pdev)
{
    priv->rst = devm_reset_control_get_shared(&pdev->dev, "usb-host");
    if (IS_ERR(priv->rst))
        return PTR_ERR(priv->rst);

    /* deassert مرة واحدة بس */
    ret = reset_control_deassert(priv->rst);
    if (ret)
        return ret;
    ...
}

static int h616_usb_suspend(struct device *dev)
{
    /* assert مرة واحدة تطابق الـ deassert */
    reset_control_assert(priv->rst);
    return 0;
}

static int h616_usb_resume(struct device *dev)
{
    /* deassert تاني عند resume */
    return reset_control_deassert(priv->rst);
}
```

#### الدرس المستفاد
الـ shared reset controls بتعتمد على `deassert_count` atomic counter — أي خلل في التوازن بين `assert`/`deassert` calls بيعطل الـ reset logic. دايمًا اعمل trace لقيمة الـ `deassert_count` عند تحقيق مشاكل suspend/resume مع shared resets.

---

### السيناريو 3: i.MX8M Plus — كاميرا ISP بتيجي `-EBUSY` عند تحميل الـ driver

#### العنوان
**الـ ISP camera driver بيفشل بـ `-EBUSY` على i.MX8M Plus عند استخدام exclusive reset**

#### السياق
بتشتغل على camera subsystem لـ embedded vision board بيستخدم i.MX8M Plus. الـ system بيشغّل Linux مع V4L2 framework. الـ ISP driver الجديد بتكتبه بيستخدم `reset_control_get_exclusive()` لإعادة تهيئة الـ ISP block عند load. بيجيك `-EBUSY` عند `insmod`.

#### المشكلة
```
[  45.112] nxp-isp 32e00000.isp: Failed to get reset control: -16
# -16 = EBUSY
```

#### التحليل
الـ `-EBUSY` بيرجع من `__reset_control_get_internal`:

```c
static struct reset_control *
__reset_control_get_internal(struct reset_controller_dev *rcdev,
                             unsigned int index, enum reset_control_flags flags)
{
    bool shared = flags & RESET_CONTROL_FLAGS_BIT_SHARED;
    bool acquired = flags & RESET_CONTROL_FLAGS_BIT_ACQUIRED;

    list_for_each_entry(rstc, &rcdev->reset_control_head, list) {
        if (rstc->id == index) {
            /* لو في rstc موجود بالفعل على نفس الـ ID */
            if (!rstc->shared && !shared && !acquired)
                break; /* exclusive released — مسموح */

            if (WARN_ON(!rstc->shared || !shared))
                return ERR_PTR(-EBUSY);  /* ← هنا بتيجي المشكلة */
            ...
        }
    }
```

السبب: driver تاني (مثلًا `imx8m-media-blk-ctrl`) مسجّل قبل كده على نفس الـ reset line (ISP reset، index 5 مثلًا) كـ exclusive acquired. لما الـ ISP driver بتاعك بيحاول يجيبه exclusive برضو → `EBUSY`.

**اتحقق مين بيمسك الـ reset:**

```bash
# شوف كل reset controls المسجّلة
ls /sys/kernel/debug/reset/
# أو
grep -r "isp" /sys/kernel/debug/clk/ 2>/dev/null

# تتبّع من أخد الـ reset
echo 'function' > /sys/kernel/debug/tracing/current_tracer
echo '__reset_control_get_internal' > /sys/kernel/debug/tracing/set_ftrace_filter
echo 1 > /sys/kernel/debug/tracing/tracing_on
# reload الـ driver وشوف الـ trace
```

#### الحل
**خيار 1** — لو الـ media block controller هو اللي المفروض يتحكم:

```c
/* في الـ ISP driver، استخدم shared بدل exclusive */
priv->rst = devm_reset_control_get_shared(&pdev->dev, "isp");
```

**خيار 2** — لو محتاج exclusive وعندك تسلسل probe محدد، استخدم `exclusive_released`:

```c
/* الـ media block controller بيمسك الـ reset كـ released */
priv->rst = devm_reset_control_get_exclusive_released(&pdev->dev, "isp");

/* لما تحتاج تعمل reset فعلي */
ret = reset_control_acquire(priv->rst);
if (ret)
    return ret;

reset_control_reset(priv->rst);
reset_control_release(priv->rst);
```

الـ `reset_control_acquire` بتتحقق إن مفيش حد تاني acquired:

```c
int reset_control_acquire(struct reset_control *rstc)
{
    ...
    list_for_each_entry(rc, &rstc->rcdev->reset_control_head, list) {
        if (rstc != rc && rstc->id == rc->id) {
            if (rc->acquired) {
                mutex_unlock(&reset_list_mutex);
                return -EBUSY;  /* حد تاني شايله */
            }
        }
    }
    rstc->acquired = true;
    ...
}
```

#### الدرس المستفاد
الـ `-EBUSY` من reset framework معناه exclusive conflict — دايمًا افتش مين driver تاني بيمسك نفس الـ reset ID قبل ما تحاول تاخده. الحل مش دايمًا تغيير لـ shared — أحيانًا الـ `exclusive_released` pattern هو الصح لو كل driver محتاج يلتزم بـ acquire/release protocol.

---

### السيناريو 4: STM32MP1 — الـ SPI flash مش بيتعرف عليه بعد warm reset

#### العنوان
**الـ SPI NOR flash بييجي unresponsive بعد warm reboot على STM32MP1 IoT sensor board**

#### السياق
عندك IoT sensor بيشتغل على STM32MP157، بيجمع بيانات كل ثانية ويكتبها على SPI NOR flash (W25Q128). بعد كل warm reboot (مش power cycle) الـ `mtd` devices بتختفي ومش بيقدر يكتب. الـ cold boot شغال تمام.

#### المشكلة
```bash
# بعد warm reboot
ls /dev/mtd*
# لا نتيجة
dmesg | grep spi
# [  2.456] spi-nor spi0.0: unrecognized JEDEC id bytes: ff ff ff
```

الـ flash بيرجع `0xFF` — يعني مش responsive — يعني الـ SPI controller نفسه أو الـ flash لسه في حالة غير متوقعة.

#### التحليل
المشكلة في إن الـ SPI controller driver بيستخدم `reset_control_reset()` لكن الـ reset line على STM32MP1 هو **level-triggered self-deasserting** (بيعمل pulse تلقائي). الـ ops بتاعة الـ rcdev بتوفر `reset` بس، مش `assert`/`deassert` منفصلين.

لما الـ driver بيعمل `devm_reset_control_get_exclusive` + بيعمل `reset_control_deassert` على warm boot:

```c
int reset_control_deassert(struct reset_control *rstc)
{
    ...
    if (!rstc->rcdev->ops->deassert)
        return 0;  /* ← بترجع 0 بدون عمل حاجة */
    ...
}
```

يعني الـ deassert operation بتنجح بدون ما تعمل حاجة فعلية لأن الـ rcdev مش بيعمّل `deassert` op. المشكلة الحقيقية إن على warm boot الـ SPI controller نفسه لم يأخذ reset كافي لأن:

1. الـ `bootloader` مش بيعمل reset للـ SPI controller
2. الـ driver بيعمل `reset_control_deassert` اللي بيرجع 0 على طول بدون فعل حاجة
3. الـ SPI flash بيلاقي controller في حالة غريبة → `0xFF` responses

الصح كان المفروض يستخدم `reset_control_reset()`:

```c
int reset_control_reset(struct reset_control *rstc)
{
    ...
    if (!rstc->rcdev->ops->reset)
        return -ENOTSUPP;  /* لو مفيش reset op → error واضح */

    if (rstc->shared) {
        if (atomic_inc_return(&rstc->triggered_count) != 1)
            return 0;
    } else {
        if (!rstc->acquired)
            return -EPERM;
    }

    ret = rstc->rcdev->ops->reset(rstc->rcdev, rstc->id);
    ...
}
```

#### الحل

**الـ driver code:**

```c
static int stm32_spi_probe(struct platform_device *pdev)
{
    spi->rst = devm_reset_control_get_exclusive(&pdev->dev, "spi");
    if (IS_ERR(spi->rst))
        return PTR_ERR(spi->rst);

    /* استخدم reset_control_reset مش deassert منفصل
     * لأن STM32 reset lines self-deasserting */
    ret = reset_control_reset(spi->rst);
    if (ret) {
        dev_err(&pdev->dev, "Failed to reset SPI: %d\n", ret);
        return ret;
    }

    /* انتظر الـ controller يكمّل initialization */
    udelay(10);
    ...
}
```

**تحقق من الـ DT:**

```dts
spi0: spi@44004000 {
    compatible = "st,stm32mp1-spi";
    reg = <0x44004000 0x400>;
    resets = <&rcc SPI1_R>;   /* self-deasserting reset */
    reset-names = "spi";
    ...
};
```

**إثبات الحل:**

```bash
# تأكد إن reset بيحصل عند probe
echo 1 > /sys/kernel/debug/tracing/events/reset/reset_control_reset/enable
rmmod spi-stm32 && modprobe spi-stm32
cat /sys/kernel/debug/tracing/trace | grep spi
```

#### الدرس المستفاد
لازم تفرّق بين نوعين reset: **self-deasserting** (بس `reset` op موجود) و**level-triggered** (عندك `assert`/`deassert` منفصلين). على STM32MP1 الـ RCC resets self-deasserting → استخدم `reset_control_reset()`. لو استخدمت `deassert` على self-deasserting reset → بيرجع 0 بدون فعل حاجة وبتتوه في debugging.

---

### السيناريو 5: RK3562 — الـ HDMI مش بيشتغل بعد إضافة driver جديد للـ VOP

#### العنوان
**الـ HDMI output اختفى بعد دمج driver جديد للـ VOP على RK3562 display board**

#### السياق
بتشتغل على display subsystem لـ custom board بيستخدم RK3562 (مناسب لـ digital signage). الـ board عنده HDMI output عبر الـ built-in HDMI TX. بعد دمج driver جديد للـ VOP2 (Video Output Processor) من مطوّر تاني، الـ HDMI بطّل يشتغل خالص رغم إن الـ DT مش اتغيّر.

#### المشكلة
```bash
dmesg | grep -E "(hdmi|vop|reset)"
[  3.221] rockchip-vop2 fe040000.vop: failed to deassert reset: -EPERM
[  3.222] rockchip-vop2 fe040000.vop: failed to initialize vop2
[  3.223] rockchip-hdmi fde80000.hdmi: no encoder found, waiting for VOP
```

#### التحليل
الـ `-EPERM` بييجي من `reset_control_deassert`:

```c
int reset_control_deassert(struct reset_control *rstc)
{
    ...
    } else {
        /* exclusive reset */
        if (!rstc->acquired) {
            WARN(1, "reset %s (ID: %u) is not acquired\n",
                 rcdev_name(rstc->rcdev), rstc->id);
            return -EPERM;  /* ← هنا */
        }
    }
    ...
}
```

السبب: الـ driver الجديد بيستخدم `reset_control_get_exclusive_released()` بدل `reset_control_get_exclusive()`. الفرق:

| Function | `acquired` field | قادر يعمل assert/deassert مباشرة؟ |
|---|---|---|
| `get_exclusive()` | `true` | نعم |
| `get_exclusive_released()` | `false` | لا — لازم `acquire()` الأول |

```c
// في __reset_control_get_internal:
rstc->acquired = acquired;  // acquired bit من الـ flags

// RESET_CONTROL_EXCLUSIVE      → BIT_ACQUIRED = 1 → acquired = true
// RESET_CONTROL_EXCLUSIVE_RELEASED → 0 → acquired = false
```

الـ driver الجديد كان بيستخدم:

```c
/* في الـ VOP2 driver الجديد — غلط للـ use case ده */
priv->rst_axi = reset_control_get_exclusive_released(dev, "axi");
reset_control_deassert(priv->rst_axi);  /* → -EPERM لأن acquired=false */
```

الـ `get_exclusive_released` مصمم للـ use case اللي فيه أكتر من consumer عايزين يتشاركوا exclusive access بالتناوب (acquire/use/release protocol) — مش للاستخدام المباشر.

**تتبع المشكلة:**

```bash
# شوف الـ WARN في dmesg
dmesg | grep "is not acquired"
# هيطلع: reset fe040000.vop:resets (ID: 2) is not acquired

# اتحقق من الـ reset ID في DT
fdtget /boot/dtb/rk3562.dtb /vop resets
# مثلًا: <&cru SRST_VOP_AXI> <&cru SRST_VOP_CORE>
```

#### الحل

**الإصلاح في الـ driver:**

```c
static int rk3562_vop2_probe(struct platform_device *pdev)
{
    struct reset_control_bulk_data resets[] = {
        { .id = "axi" },
        { .id = "ahb" },
        { .id = "dclk" },
    };

    /* استخدم exclusive عادي — مش released */
    ret = devm_reset_control_bulk_get_exclusive(dev,
                ARRAY_SIZE(resets), resets);
    if (ret) {
        dev_err(dev, "failed to get resets: %d\n", ret);
        return ret;
    }

    /* deassert بالعكسي — من الآخر للأول */
    ret = reset_control_bulk_deassert(ARRAY_SIZE(resets), resets);
    if (ret) {
        dev_err(dev, "failed to deassert resets: %d\n", ret);
        return ret;
    }

    ...

    /* عند remove: assert بالترتيب */
    reset_control_bulk_assert(ARRAY_SIZE(resets), resets);
}
```

لاحظ إن `reset_control_bulk_deassert` بتعمل deassert بالعكسي (من الآخر للأول) والـ `reset_control_bulk_assert` بتعمل assert بالترتيب الطبيعي — وده مقصود لضمان تسلسل صحيح:

```c
int reset_control_bulk_deassert(int num_rstcs,
                                struct reset_control_bulk_data *rstcs)
{
    /* يبدأ من آخر element */
    for (i = num_rstcs - 1; i >= 0; i--) {
        ret = reset_control_deassert(rstcs[i].rstc);
        ...
    }
}
```

**التحقق بعد الإصلاح:**

```bash
dmesg | grep -E "(vop|hdmi)"
# المفروض:
# [  3.221] rockchip-vop2 fe040000.vop: initialized successfully
# [  3.450] rockchip-hdmi fde80000.hdmi: registered HDMI encoder

# تأكد إن HDMI شغال
cat /sys/class/drm/card0-HDMI-A-1/status
# connected
```

#### الدرس المستفاد
**الـ `get_exclusive_released` ليس بديلًا عن `get_exclusive`** — هو pattern متخصص لـ cooperative exclusive access بين أكتر من driver. لو driver واحد بس بيستخدم الـ reset وعايز يعمل deassert مباشرة → استخدم `get_exclusive()`. الـ `WARN` message اللي بتطلع ("is not acquired") في `reset_control_assert`/`deassert` هي clue واضح جدًا تدلّك على المشكلة.
## Phase 7: مصادر ومراجع

### المصادر الرسمية للـ Kernel

#### التوثيق الرسمي

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| **Reset controller API** (docs.kernel.org) | [docs.kernel.org/driver-api/reset.html](https://docs.kernel.org/driver-api/reset.html) | التوثيق الرسمي الكامل للـ API |
| **Reset controller API** (kernel.org RST) | [kernel.org/doc/Documentation/driver-api/reset.rst](https://www.kernel.org/doc/Documentation/driver-api/reset.rst) | المصدر الخام للتوثيق |
| **reset.h header** (GitHub) | [include/linux/reset.h](https://github.com/torvalds/linux/blob/master/include/linux/reset.h) | واجهة الـ consumer API |

#### مسارات داخل الـ kernel source tree

```
Documentation/driver-api/reset.rst    ← التوثيق الرسمي للـ API
drivers/reset/core.c                  ← التنفيذ المركزي للـ framework
drivers/reset/reset-gpio.c            ← مثال: GPIO-based reset controller
include/linux/reset.h                 ← consumer API
include/linux/reset-controller.h      ← provider API
```

---

### مقالات LWN.net

| المقال | الرابط | الأهمية |
|--------|--------|---------|
| **reset: Add generic GPIO reset driver** | [lwn.net/Articles/585145](https://lwn.net/Articles/585145/) | أول patch لـ GPIO reset driver — مثال عملي على كيفية بناء provider |
| **clk: en7523: reset-controller support** | [lwn.net/Articles/1039543](https://lwn.net/Articles/1039543/) | مثال حديث على دمج reset controller مع clock controller في SoC |
| **Add Arria10 System Manager Reset Controller** | [lwn.net/Articles/714705](https://lwn.net/Articles/714705/) | مثال على reset controller لـ Altera/Intel SoC |
| **Expose and manage PCI device reset** | [lwn.net/Articles/862313](https://lwn.net/Articles/862313/) | reset في سياق PCI وعلاقته بالـ framework |

---

### Patchwork — الـ Patches الأصلية

الـ **reset controller framework** أضافه Philipp Zabel من Pengutronix سنة 2013:

| الـ Patch | الرابط | الوصف |
|-----------|--------|-------|
| `[v4,2/8] reset: Add reset controller API` | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/) | الـ patch الأصلي للـ framework — نقطة البداية التاريخية |
| `[v4,2/2] reset: Add GPIO support` | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1397463709-19405-2-git-send-email-p.zabel@pengutronix.de/) | إضافة دعم GPIO للـ framework |
| `[v8] reset: Add driver for gpio-controlled reset pins` | [patchwork.kernel.org](https://patchwork.kernel.org/project/linux-arm-kernel/patch/20130716015038.GD28375@S2101-09.ap.freescale.net/) | الـ GPIO reset driver الكامل |
| `[v1,1/5] reset: devm_reset_control_array_get()` | [patchwork.kernel.org](https://patchwork.kernel.org/project/alsa-devel/patch/20210302112123.24161-2-digetx@gmail.com/) | إضافة array reset في released state |
| `[PATCH v4 2/4] reset: Add APIs to manage array of resets` | [lkml.iu.edu](https://lkml.iu.edu/hypermail/linux/kernel/1705.2/04766.html) | الـ API للـ reset arrays |

---

### نقاشات الـ Mailing List

| النقاش | الرابط | الموضوع |
|--------|--------|---------|
| **[GIT PULL] Reset controller fixes for v6.2** | [lore.kernel.org](https://lore.kernel.org/all/20230104095855.3809733-1-p.zabel@pengutronix.de/) | pull request من المشرف الرسمي |
| **Why do we need reset_control_get_optional()?** | [groups.google.com](https://groups.google.com/g/fa.linux.kernel/c/9jG5RFPcEZU) | نقاش عميق حول الـ optional resets ولماذا تختلف عن الـ mandatory |
| **PATCH: MAINTAINERS: Add reset controller framework entry** | [lists.gt.net](https://lists.gt.net/linux/kernel/1931867) | إضافة الـ framework رسمياً لقائمة المشرفين |
| **Re: imx_dsp_rproc: Use reset controller API** | [mail-archive.com](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2584130.html) | مثال على استخدام الـ API في driver حقيقي |
| **Bartosz Golaszewski: reset: handle removing supplier before consumers** | [lkml.org](https://lkml.org/lkml/2026/2/23/484) | نقاش حديث (2026) حول إدارة lifetime الـ reset controllers |

---

### توثيق خاص بالـ Vendors

| المصدر | الرابط | الجهة |
|--------|--------|-------|
| **ZynqMP Linux Reset-controller Driver** | [xilinx-wiki.atlassian.net](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842455/ZynqMP+Linux+Reset-controller+Driver) | Xilinx/AMD — مثال على تطبيق المفهوم في SoC احترافي |
| **Reset overview - STM32MPU** | [wiki.st.com/stm32mpu](https://wiki.st.com/stm32mpu/wiki/Reset_overview) | STMicroelectronics — شرح مفصل للـ reset في STM32 ecosystem |

---

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصول ذات الصلة**:
  - Chapter 1: An Introduction to Device Drivers — فهم بنية الـ kernel frameworks
  - Chapter 14: The Linux Device Model — فهم `struct device` والـ kref وdependency tracking
- **ملاحظة**: الكتاب قديم (2005) ولا يغطي reset framework مباشرة، لكن الفهم الأساسي للـ device model لازم
- **رابط مجاني**: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development, 3rd Edition
- **المؤلف**: Robert Love
- **الفصول ذات الصلة**:
  - Chapter 17: Devices and Modules — كيف يشتغل الـ driver model
  - Chapter 20: Patches, Hacking, and the Community — فهم الـ patch workflow
- **الناشر**: Addison-Wesley, 2010

#### Embedded Linux Primer, 2nd Edition
- **المؤلف**: Christopher Hallinan
- **الفصول ذات الصلة**:
  - Chapter 16: Kernel Debugging Techniques — debugging الـ reset issues
  - أي فصل يتكلم عن Device Tree وـ SoC initialization
- **الناشر**: Prentice Hall, 2010

#### Professional Linux Kernel Architecture
- **المؤلف**: Wolfgang Mauerer
- **الفصول ذات الصلة**:
  - Chapter 6: Device Drivers — بنية الـ frameworks بشكل عام

---

### مصادر KernelNewbies و eLinux

| المصدر | الرابط | الوصف |
|--------|--------|-------|
| **KernelNewbies: KernelGlossary** | [kernelnewbies.org/KernelGlossary](https://kernelnewbies.org/KernelGlossary) | مرجع للمصطلحات العامة في الـ kernel |
| **KernelNewbies: FirstKernelPatch** | [kernelnewbies.org/FirstKernelPatch](https://kernelnewbies.org/FirstKernelPatch) | للمبتدئين اللي عايزين يساهموا في الـ reset subsystem |
| **KernelNewbies: KernelHacking** | [kernelnewbies.org/KernelHacking](https://kernelnewbies.org/KernelHacking) | نقطة بداية لفهم بنية الـ kernel |
| **eLinux: Kernel Development** | [elinux.org/Kernel_Development](https://elinux.org/Kernel_Development) | مصادر تطوير الـ kernel للـ embedded |

---

### Search Terms للبحث عن معلومات إضافية

للبحث في **LKML / lore.kernel.org**:
```
reset controller framework linux kernel
devm_reset_control_get exclusive shared
reset_control_assert deassert
drivers/reset/core.c
reset-controller device tree binding
```

للبحث في **Google / DuckDuckGo**:
```
linux kernel reset controller subsystem tutorial
reset_control_get_exclusive vs shared
linux reset controller provider driver example
of_reset_control_get device tree
reset controller array bulk API
```

للبحث في **Elixir Cross-Referencer** (elixir.bootlin.com):
```
# لمشاهدة كل مكان بيُستخدم فيه الـ API
reset_control_get_exclusive
reset_control_assert
reset_controller_dev
devm_reset_control_get
```

---

### ملخص سريع لأهم 5 مصادر

| الأولوية | المصدر | السبب |
|----------|--------|-------|
| **1** | [docs.kernel.org/driver-api/reset.html](https://docs.kernel.org/driver-api/reset.html) | التوثيق الرسمي — ابدأ منه |
| **2** | [Patch الأصلي لـ Philipp Zabel](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/) | التاريخ الكامل للـ framework وسبب وجوده |
| **3** | [lwn.net/Articles/585145](https://lwn.net/Articles/585145/) | مثال GPIO reset driver من LWN |
| **4** | [نقاش reset_control_get_optional](https://groups.google.com/g/fa.linux.kernel/c/9jG5RFPcEZU) | فهم الـ design decisions |
| **5** | [include/linux/reset.h على GitHub](https://github.com/torvalds/linux/blob/master/include/linux/reset.h) | الـ API الكامل بالتعليقات الرسمية |
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على `reset_controller_register` — الـ function الأساسية اللي أي reset controller driver بيسجل نفسه بيها في الـ kernel. كل مرة driver جديد (زي reset controller لـ SoC) بيشتغل ويسجل نفسه، الـ kprobe بتاعتنا بتتحرك وتطبع اسم الـ controller وعدد الـ reset lines اللي بيديها.

ده مفيد جداً لأنك تعرف في real-time أي reset controllers موجودين في النظام وإمتى بيتسجلوا.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on reset_controller_register()
 * Prints info about every reset controller as it registers.
 */

#include <linux/module.h>       /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>       /* pr_info */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe */
#include <linux/reset-controller.h> /* struct reset_controller_dev */
#include <linux/device.h>       /* dev_name */

/* ------------------------------------------------------------------ */
/* kprobe pre-handler: runs BEFORE reset_controller_register() body   */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64: first argument lives in RDI.
     * On ARM64 : first argument lives in X0.
     * regs_get_kernel_argument() abstracts this for us.
     */
    struct reset_controller_dev *rcdev =
        (struct reset_controller_dev *)regs_get_kernel_argument(regs, 0);

    /* rcdev->dev may be NULL for some GPIO-based controllers */
    const char *name = "(unknown)";
    if (rcdev->dev)
        name = dev_name(rcdev->dev);
    else if (rcdev->of_node)
        name = rcdev->of_node->full_name;

    pr_info("[reset_probe] registering controller: name=%s  nr_resets=%u  ops=%ps\n",
            name,
            rcdev->nr_resets,
            rcdev->ops);

    return 0; /* 0 = let the original function continue normally */
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "reset_controller_register", /* function to hook */
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/* module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init reset_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[reset_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[reset_probe] kprobe planted on reset_controller_register @ %p\n",
            kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/* module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit reset_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[reset_probe] kprobe removed\n");
}

module_init(reset_probe_init);
module_exit(reset_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on reset_controller_register to trace reset controllers");
```

---

### Makefile لبناء الـ module

```makefile
obj-m += reset_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل
make
sudo insmod reset_probe.ko
dmesg | grep reset_probe

# إزالة
sudo rmmod reset_probe
```

---

### شرح كل جزء

#### الـ includes

| Header | السبب |
|---|---|
| `linux/module.h` | `module_init / module_exit / MODULE_*` macros |
| `linux/kernel.h` | `pr_info / pr_err` |
| `linux/kprobes.h` | `struct kprobe` وكل API الـ kprobes |
| `linux/reset-controller.h` | تعريف `struct reset_controller_dev` اللي هنستخدمه داخل الـ handler |
| `linux/device.h` | `dev_name()` عشان نطبع اسم الـ device |

الـ includes دي هي بالظبط اللي بيحتاجها أي كود بيتعامل مع reset controllers، نفس اللي في `core.c` نفسه.

#### الـ `handler_pre`

**الـ `struct pt_regs *regs`** بيحتوي على حالة الـ CPU registers لحظة الـ probe. بنستخدم `regs_get_kernel_argument(regs, 0)` عشان ناخد الـ argument الأول (`rcdev`) بطريقة portable على x86 وARM على حد سواء بدل ما نعمل cast يدوي لـ `rdi` أو `x0`.

بعد ما جبنا الـ `rcdev`، بنطبع:
- **اسم الـ controller**: من `rcdev->dev` أو `rcdev->of_node` لأن بعض الـ controllers (زي GPIO-based) ممكن يبقى عندهم node بس من غير `device`.
- **`nr_resets`**: عدد الـ reset lines اللي الـ controller ده بيتحكم فيها — معلومة مهمة تفيدك تعرف حجم الـ SoC subsystem.
- **`ops`**: بنطبع pointer الـ ops struct مع `%ps` عشان الـ kernel يطبع اسم الـ symbol (زي `imx_reset_ops`) بدل مجرد address، ده بيوضح مين الـ driver وراه.

الـ function بترجع `0` يعني "استمر في تنفيذ الـ function الأصلية" — ما بنعملش intercepting، بس observing.

#### الـ `struct kprobe kp`

**`.symbol_name`** هو اسم الـ function اللي هنزرع فيها الـ probe. الـ kernel بيعمل lookup في الـ symbol table ويجيب الـ address وقت الـ `register_kprobe`. اخترنا `reset_controller_register` لأنها `EXPORT_SYMBOL_GPL` — يعني موجودة في الـ symbol table ومتاحة للـ modules.

#### الـ `module_init`

بنسجل الـ kprobe وبنتأكد إن الـ registration نجحت. لو فشلت (مثلاً الـ kernel مبيعرفش الـ symbol أو CONFIG_KPROBES معمولهاش enable) بنرجع error فوراً ومش بنكمل.

#### الـ `module_exit`

**لازم** نعمل `unregister_kprobe` في الـ exit لأن الـ kprobe بتحتفظ بـ patch في الـ kernel code (بتحل محل أول byte من الـ instruction بـ breakpoint). لو الـ module اتحذف من غير ما ننزع الـ probe، أي call لـ `reset_controller_register` بعد كده هيـ trigger exception من غير handler، وده kernel panic مضمون.

---

### مثال على الـ output

```
[reset_probe] kprobe planted on reset_controller_register @ ffffffffc0123456
[reset_probe] registering controller: name=soc:reset-controller  nr_resets=64  ops=imx_reset_ops
[reset_probe] registering controller: name=fc000000.reset  nr_resets=8   ops=sunxi_reset_ops
```

كل سطر بيمثل driver جديد سجّل نفسه — ممكن تشوف ده وقت الـ boot أو وقت hotplug لـ module.
