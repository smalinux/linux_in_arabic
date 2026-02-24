## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **Reset Controller Framework** في Linux kernel، maintained بواسطة Philipp Zabel. الـ subsystem ده موجود كـ entry مستقل في MAINTAINERS باسم "RESET CONTROLLER FRAMEWORK".

---

### المشكلة اللي بيحلّها

تخيّل إنك بتبني لوحة إلكترونية فيها معالج رئيسي وحواليه 20 جهاز (USB controller، Ethernet MAC، GPU، display engine، إلخ). كل واحد من الأجهزة دي عنده **reset pin** — سلك فيزيائي لما تضبطه على "1" الجهاز يتجمّد ويتصفّر، ولما تضبطه على "0" يبدأ يشتغل.

في الـ SoC الحديثة (زي Qualcomm Snapdragon، NXP i.MX، Allwinner، إلخ)، مش كل جهاز عنده وحدة reset منفصلة — بدل كده في **وحدة مركزية واحدة** اسمها **Reset Controller** هي اللي بتتحكم في كل الـ reset signals دي من خلال registers. بتكتب bit معيّن في register معيّن فتضرب reset على USB مثلاً، أو تكتب في bit تاني تضرب reset على Ethernet.

**المشكلة:** كل driver محتاج يعمل reset لنفسه لما بيبدأ أو بيوقف. لو كل driver اشتغل مباشرة مع الـ hardware registers، هيحصل:
- **فوضى**: كل driver يعمل الكلام بطريقته الخاصة.
- **تعارض**: driver بيعمل assert على reset line وفي نفس الوقت driver تاني شايلها deasserted.
- **تكرار**: كل platform لازم تكتب نفس الكود من الأول.

**الحل:** طبقة abstraction اسمها **Reset Controller API** — بالظبط زي ما الـ clock framework بيعمل abstraction على الـ clock signals.

---

### القصة من الأول للآخر

```
┌─────────────────────────────────────────────────────────┐
│                       SoC                               │
│                                                         │
│  ┌──────────────────┐      reset line 0  ┌───────────┐ │
│  │                  │ ─────────────────► │  USB ctrl │ │
│  │  Reset           │      reset line 1  ├───────────┤ │
│  │  Controller      │ ─────────────────► │  Ethernet │ │
│  │  Hardware        │      reset line 2  ├───────────┤ │
│  │  (registers)     │ ─────────────────► │  GPU      │ │
│  │                  │      reset line N  ├───────────┤ │
│  └──────────────────┘ ─────────────────► │  ...      │ │
│           ▲                              └───────────┘ │
│           │                                             │
│  ┌────────┴────────────────────────────────────────┐   │
│  │           Reset Controller Framework             │   │
│  │  (drivers/reset/core.c)                         │   │
│  └──────────────────┬──────────────────────────────┘   │
│                     │                                   │
│        ┌────────────┼────────────┐                     │
│        ▼            ▼            ▼                     │
│  USB driver   ETH driver   GPU driver                  │
└─────────────────────────────────────────────────────────┘
```

**الـ framework** بيقسم المسؤولية لجزئين:

1. **Reset Controller Driver (Provider):** الـ driver اللي بيتعامل مع الـ hardware registers وبيعمل register للـ framework — بيقول "أنا عندي 32 reset line، اطلب منهم بالاسم أو بالرقم".

2. **Consumer Driver:** أي driver (USB، Ethernet، إلخ) محتاج يعمل reset لنفسه — بيطلب الـ reset control بالاسم من الـ device tree، ويعمل `assert` / `deassert` من غير ما يعرف إيه الـ hardware بالظبط.

---

### المفاهيم الأساسية

| المصطلح | المعنى |
|---|---|
| **Reset line** | السلك الفيزيائي اللي بيحمل إشارة الـ reset من الـ controller للـ peripheral |
| **Reset control** | الطريقة اللي بتتحكم فيها في سلك واحد أو أكثر — عادةً bit في register |
| **Reset controller** | الـ hardware unit اللي بتوفر مجموعة reset controls |
| **Reset consumer** | أي peripheral أو IC بيتأثر بإشارة reset |

---

### Shared vs Exclusive Resets

فكرة مهمة جداً في الـ framework:

- **Exclusive reset:** بتاخده لنفسك. لما تعمل `assert`، الـ line بتتضرب فوراً. زي غرفة فندق — أنت لوحدك فيها.

- **Shared reset:** أكتر من driver ممكن يشاركوا نفس الـ reset line (مثلاً: حاجتين بتعتمدوا على نفس power domain). الـ framework بيعمل **reference counting** — الـ line مش بترجع للـ assert غير لما آخر consumer يعمل assert. زي ضوء الأوضة المشتركة — بيفضل ولع طول ما فيه حد جوّا.

---

### أنواع العمليات

```c
// Driver يطلب reset control (exclusive) — devm تتولى release تلقائياً
rstc = devm_reset_control_get_exclusive(dev, "usb");

// يحط الجهاز في reset (يوقفه)
reset_control_assert(rstc);

// يطلعه من reset (يبدأ تشغيله)
reset_control_deassert(rstc);

// pulse واحدة للـ reset (self-deasserting)
reset_control_reset(rstc);

// يسأل: هل الجهاز في reset دلوقتي؟
reset_control_status(rstc);
```

---

### Optional Resets

نفس الـ peripheral ممكن يحتاج reset line في platform وفي platform تانية لا. الـ API بتدعم ده عن طريق `_optional_` variants — لو الـ reset مش معرّف في الـ device tree، بترجع `NULL` بدل error، وكل الـ functions بتتعامل مع `NULL` بهدوء من غير crash.

---

### Reset Control Arrays

أحياناً driver واحد عنده مجموعة reset lines محتاج يضربهم كلهم دفعة واحدة. الـ API بتديك `devm_reset_control_array_get()` اللي بترجع handle واحد تتعامل معاه كأنه reset واحد — الترتيب مش مضمون لكن المجموعة كلها بتتعالج.

---

### الـ Files المكوّنة للـ Subsystem

#### Core Framework
| الملف | الدور |
|---|---|
| `drivers/reset/core.c` | قلب الـ framework — كل الـ logic بتاع reference counting، registration، والـ consumer API |
| `include/linux/reset.h` | الـ consumer API — اللي بيستخدمه أي driver محتاج reset |
| `include/linux/reset-controller.h` | الـ provider API — اللي بيستخدمه driver الـ reset controller نفسه |

#### الـ Structs الأساسية (موجودة في `reset-controller.h`)
```c
/* الـ callbacks اللي لازم driver الـ controller يطبّقها */
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};

/* تمثيل الـ reset controller hardware في الـ kernel */
struct reset_controller_dev {
    const struct reset_control_ops *ops;  /* الـ callbacks */
    struct device *dev;
    struct device_node *of_node;          /* لربطه بالـ device tree */
    unsigned int nr_resets;               /* عدد الـ reset lines */
    /* ... */
};
```

#### Hardware Drivers (أمثلة من `drivers/reset/`)
| الملف | الـ Platform |
|---|---|
| `reset-imx7.c` | NXP i.MX7 SoC |
| `reset-k210.c` | Canaan K210 RISC-V SoC |
| `reset-bcm6345.c` | Broadcom BCM6345 |
| `reset-gpio.c` | Generic GPIO-based reset |
| `reset-ath79.c` | Qualcomm Atheros AR7xxx |
| `reset-aspeed.c` | ASPEED BMC SoCs |

#### Device Tree Bindings
- `Documentation/devicetree/bindings/reset/` — بتعرّف إزاي تكتب الـ reset controller في الـ device tree

#### الـ DT Reset Defines
- `include/dt-bindings/reset/` — constants للـ reset IDs بتستخدمها في الـ DTS files

---

### ملفات مرتبطة لازم تعرفها

- **`drivers/clk/`** — الـ clock framework مشابه جداً في التصميم، ممكن تتعلم منه patterns
- **`Documentation/devicetree/bindings/reset/`** — كيفية وصف الـ reset controllers في الـ DTS
- **`drivers/reset/core.c`** — الـ implementation الكاملة للـ API المذكورة في الـ doc
- **`include/linux/reset/`** — headers إضافية للـ reset framework extensions
## Phase 2: شرح الـ Reset Controller Framework

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في أي SoC حديث — خد مثلاً Allwinner H6 أو NXP i.MX8 أو Rockchip RK3588 — فيه عشرات الـ peripherals: USB controller، Ethernet MAC، Display Engine، Audio، وغيرهم. كل واحد فيهم محتاج **reset signal** فيزيائي — سلك بيجي من وحدة مركزية (غالباً CCU أو PRCM) وبيحط الـ peripheral في حالة reset أو بيطلعه منها.

المشكلة الكلاسيكية قبل الـ framework:

- كل driver كان بيعمل `ioremap` مباشرة على رجستر الـ CCU وبيعمل bitwise operations يدوياً.
- لو درايفرين اشتركوا في نفس الـ reset bit وعملوا assert/deassert بدون تنسيق → **race condition** وـ peripheral بيتعلق.
- لو platform تانية عندها نفس الـ IP block بس مع reset controller مختلف → driver كود متكرر ومختلف لكل platform.
- لا يوجد أي abstraction بين "مين محتاج الـ reset" و"مين يتحكم فيه".

**الخلاصة:** محتاجين طبقة وسيطة بتفصل الـ consumer (الـ driver اللي محتاج reset) عن الـ provider (الـ hardware اللي بيوفر الـ reset signal).

---

### الحل — الـ Kernel بيعمل إيه؟

الـ Reset Controller Framework بيوفر:

1. **Abstraction layer** بين الـ consumer drivers والـ reset hardware.
2. **Reference counting** للـ shared reset lines — ضمان إن الـ line مش هتتعمل assert طول ما حد لسه محتاجها deasserted.
3. **Device Tree integration** — الـ consumer بيطلب الـ reset بالاسم، والـ framework يترجمه لـ hardware ID تلقائياً عبر `of_xlate`.
4. **Resource management** عبر `devm_*` — reset_control_put بيتعمل automatically لما الـ driver بيتفك.
5. **Stub API** لما `CONFIG_RESET_CONTROLLER` مش enabled — الـ consumer code مش محتاج `#ifdef` في كل مكان.

---

### المقارنة بالـ Clock Framework

الـ documentation نفسه بيقول إن الـ consumer interface شبيه بالـ clock framework — وده مش مصادفة:

| Clock Framework | Reset Controller Framework |
|---|---|
| `clk_get()` | `reset_control_get_exclusive()` |
| `clk_enable()` | `reset_control_deassert()` |
| `clk_disable()` | `reset_control_assert()` |
| `devm_clk_get()` | `devm_reset_control_get_exclusive()` |
| reference counted enable | reference counted deassert (shared mode) |

> **ملاحظة مهمة:** الـ Clock Framework (`drivers/clk/`) subsystem منفصل، وبيتحكم في تشغيل وإيقاف وضبط تردد الـ clock signals. الـ reset و clock أغلب الوقت بيتحكم فيهم نفس الـ CCU hardware، بس الـ kernel بيفصلهم في frameworkين مستقلين.

---

### البنية العامة — Big Picture

```
+------------------------------------------------------------------+
|                    Consumer Drivers (Peripheral Drivers)         |
|   usb-dwc3.c    eth-stmmac.c    display-sun4i.c    sound/...    |
|       |               |                |                |        |
|       +-------+-------+----------------+----------------+        |
|               |                                                  |
|    reset_control_deassert(rstc)  /  reset_control_assert(rstc)  |
+---------------|--------------------------------------------------+
                |
                v
+------------------------------------------------------------------+
|              Reset Controller Core  (drivers/reset/core.c)      |
|                                                                  |
|   - Maintains list of all registered reset_controller_dev's     |
|   - Resolves DT phandles → rcdev + reset ID                     |
|   - Enforces exclusive vs shared semantics                       |
|   - Reference counts deassert for shared controls               |
|   - devm lifetime management                                     |
+---+-------------------------------+----------------------------+-+
    |                               |                            |
    v                               v                            v
+----------+               +-------------+             +----------+
| rcdev A  |               |   rcdev B   |             | rcdev C  |
| Allwinner|               | NXP IOMUXC  |             | GPIO-    |
|   CCU    |               |   Reset     |             | based    |
|          |               |             |             | Reset    |
| ops:     |               | ops:        |             |          |
| .assert  |               | .assert     |             | .assert  |
| .deassert|               | .deassert   |             | .deassert|
| .reset   |               | .reset      |             |          |
| .status  |               | .status     |             |          |
+----+-----+               +------+------+             +-----+----+
     |                            |                          |
     v                            v                          v
[CCU Registers]          [IOMUXC Registers]           [GPIO Pin]
bit 14 = USB reset       bit 3 = ENET reset           GPIO_PA5
bit 22 = EMAC reset      bit 7 = LCD reset
bit 31 = DE reset
     |                            |
     v                            v
[Physical Reset Lines to Peripherals on Die]
```

---

### التشبيه الحقيقي — لوحة كهرباء المبنى

تخيل إنك مسؤول عن مبنى إداري كبير (الـ SoC). في غرفة كهرباء مركزية (الـ Reset Controller HW) فيها **لوحة قواطع (circuit breakers)** — كل قاطع بيتحكم في طابق أو غرفة معينة.

| التشبيه | المفهوم الفعلي في الـ Kernel |
|---|---|
| لوحة القواطع المركزية | `reset_controller_dev` (الـ CCU أو PRCM) |
| رقم القاطع (مثلاً قاطع 14) | الـ reset line ID (`unsigned long id`) |
| السلك من القاطع للغرفة | الـ physical reset line على الـ die |
| الغرفة/الجهاز المتصل | الـ reset consumer (USB controller مثلاً) |
| مكتب الصيانة | الـ Reset Controller Core (`core.c`) |
| طلب الصيانة بالاسم ("غرفة 301") | `devm_reset_control_get_exclusive(dev, "usb")` |
| دفتر طلبات الصيانة | قائمة الـ `reset_control` المرتبطة بكل `rcdev` |
| قاعدة "لازم كل الناس توافق قبل تقفل الكهرباء" | الـ shared reset reference counting |
| موظف الصيانة اللي بيترجم "غرفة 301" لـ "قاطع 14" | `of_xlate` callback |

**التعمق في التشبيه:**

- لو **غرفتين** (دايرتين من الـ USB) شاركتا نفس القاطع → الصيانة (الـ core) بتحتفظ بعداد. لو طلب واحد يقفل الكهرباء، لكن التاني لسه محتاجها، القاطع مش هيتقفل — ده بالضبط الـ **shared reset** مع reference counting.
- لو **غرفة واحدة** طلبت قاطعها بشكل حصري → أي تاني يحاول يطلب نفس القاطع هياخد `-EBUSY` — ده الـ **exclusive reset**.
- لو القاطع "self-resetting" (بيرجع لوحده بعد ثانية) → ده الـ `reset_control_reset()` للـ self-deasserting resets.
- لو الغرفة مش موجودة في هذا المبنى (platform مش محتاجة الـ reset ده) → `devm_reset_control_get_optional_*()` بترجع NULL بدل error.

---

### الـ Core Abstraction — الفكرة المحورية

**الـ opaque `reset_control` handle** هو الـ abstraction الأساسي.

الـ consumer driver **مش عارف**:
- فين الـ registers
- إيه الـ bit number
- هل الـ reset active-high أو active-low
- هل فيه timing requirements

الـ consumer بس عنده handle وبيقول: "assert" أو "deassert" أو "pulse".

---

### بنية الـ Structs — كيف بيتربطوا ببعض

```
Device Tree:
  usb@01c19000 {
      resets = <&ccu RST_BUS_OTG>;
      reset-names = "usb";
  };

  ccu: clock-controller@01c20000 {
      #reset-cells = <1>;
  };

Consumer Driver (dwc3-of-simple.c):
  rstc = devm_reset_control_get_exclusive(dev, "usb");
  reset_control_deassert(rstc);

                    |
                    v

struct reset_control {            <-- opaque handle للـ consumer
    struct reset_controller_dev *rcdev;   -- يشاور على الـ CCU
    unsigned int id;              -- رقم الـ reset line (RST_BUS_OTG = 14)
    struct list_head list;        -- في قائمة rcdev->reset_control_head
    atomic_t deassert_count;      -- للـ shared resets
    atomic_t triggered_count;     -- للـ shared pulse resets
    bool shared;                  -- exclusive أو shared؟
    bool acquired;                -- للـ exclusive_released mode
};

                    |
                    | rcdev pointer
                    v

struct reset_controller_dev {     <-- يمثل الـ CCU hardware unit
    const struct reset_control_ops *ops;  -- callbacks للـ assert/deassert
    struct list_head list;        -- في global list of all rcdevs
    struct list_head reset_control_head; -- كل الـ reset_controls المطلوبة منه
    struct device *dev;           -- الـ CCU device نفسه
    struct device_node *of_node;  -- لحل الـ phandles من DT
    int of_reset_n_cells;         -- عدد cells في الـ specifier (عادة 1)
    int (*of_xlate)(...);         -- تحويل DT specifier → id
    unsigned int nr_resets;       -- عدد الـ reset lines المتاحة
};

                    |
                    | ops pointer
                    v

struct reset_control_ops {        <-- الـ callbacks اللي الـ CCU driver بيimplementها
    int (*reset)(rcdev, id);      -- pulse كامل (self-deasserting)
    int (*assert)(rcdev, id);     -- وضع الـ peripheral في reset
    int (*deassert)(rcdev, id);   -- إخراجه من reset
    int (*status)(rcdev, id);     -- استعلام عن الحالة
};
```

---

### الـ DT Translation Flow — من الاسم للـ Bit

ده من أهم الأجزاء — إزاي الـ framework بيربط اسم (`"usb"`) بـ bit معين في register:

```
Step 1: Consumer calls devm_reset_control_get_exclusive(dev, "usb")
        |
        v
Step 2: Core reads DT property "reset-names" → finds index of "usb" = 0
        reads "resets" property at index 0 → phandle = &ccu, args = [14]
        |
        v
Step 3: Core searches global rcdev list for rcdev with of_node == &ccu
        |
        v
Step 4: Core calls rcdev->of_xlate(rcdev, &phandle_args)
        Default of_reset_simple_xlate() just returns args.args[0] = 14
        |
        v
Step 5: Core creates/finds struct reset_control {rcdev=ccu_rcdev, id=14}
        Returns opaque pointer to consumer
        |
        v
Step 6: Consumer calls reset_control_deassert(rstc)
        Core calls rstc->rcdev->ops->deassert(rcdev, 14)
        |
        v
Step 7: CCU driver deassert() clears bit 14 in BUS_SOFT_RST_REG0
```

---

### Shared vs Exclusive — التفصيل الكامل

#### Exclusive Reset

```c
/* Driver A يطلب exclusive */
rstc = devm_reset_control_get_exclusive(dev, "enet");
/* لو Driver B حاول نفس الشيء → -EBUSY */

reset_control_assert(rstc);    /* فوري، بدون reference counting */
reset_control_deassert(rstc);  /* فوري */
```

#### Shared Reset (مع Reference Counting)

```c
/* Driver A */
rstcA = reset_control_get_shared(devA, "shared-bus");
reset_control_deassert(rstcA);  /* deassert_count = 1 → فعلاً deassert */

/* Driver B (نفس الـ reset line) */
rstcB = reset_control_get_shared(devB, "shared-bus");
reset_control_deassert(rstcB);  /* deassert_count = 2 → لا حاجة جديدة */

/* Driver A انتهى */
reset_control_assert(rstcA);    /* deassert_count = 1 → لا assert فعلي، B لسه محتاجه */

/* Driver B انتهى */
reset_control_assert(rstcB);    /* deassert_count = 0 → assert فعلي على الـ hardware */
```

#### Exclusive Released — للـ Runtime PM

ده mode متخصص للـ drivers اللي بتستخدم runtime PM وبتحتاج تشارك الـ reset handle مع الـ PM callbacks:

```c
/* الـ driver بيحجز الـ handle بدون استخدامه */
rstc = devm_reset_control_get_exclusive_released(dev, "core");

/* قبل الاستخدام لازم acquire */
reset_control_acquire(rstc);
reset_control_assert(rstc);
reset_control_deassert(rstc);
reset_control_release(rstc);  /* يسيبه لحد تاني يستخدمه */
```

---

### الـ Reset Control Array

لما driver محتاج يعمل reset لمجموعة lines دفعة واحدة بدون order محدد:

```c
/* في DT:
   resets = <&ccu RST_A>, <&ccu RST_B>, <&ccu RST_C>;
*/

/* الـ framework بيرجع handle واحد بيغلف الثلاثة */
rstc = devm_reset_control_array_get_exclusive(dev);
reset_control_assert(rstc);    /* بيعمل assert على الثلاثة */
reset_control_deassert(rstc);  /* بيعمل deassert على الثلاثة */
```

---

### Self-Deasserting Resets والـ Rearm

بعض الـ hardware عنده reset line بتعمل pulse تلقائي وترجع — مش محتاج driver يعمل assert ثم deassert يدوياً:

```c
/* pulse واحد بس */
reset_control_reset(rstc);

/*
 * لو shared self-deasserting reset:
 * أول caller بس اللي يعمل pulse فعلي.
 * الباقين لازم يعملوا rearm الأول عشان يسمحوا لـ pulse جديدة.
 */
reset_control_rearm(rstc);
```

---

### مين بيملك إيه — Core vs Driver

| المسؤولية | الـ Reset Core (`core.c`) | الـ Controller Driver (مثلاً `clk-sunxi-ng.c`) |
|---|---|---|
| حل الـ DT phandle | نعم — بيقرأ "resets" و"reset-names" | لا |
| إنشاء `reset_control` struct | نعم | لا |
| Reference counting | نعم | لا |
| Exclusive/Shared enforcement | نعم | لا |
| devm cleanup | نعم | لا |
| كتابة/قراءة الـ registers | لا | نعم — عبر `ops->assert/deassert` |
| تعريف عدد الـ reset lines | لا | نعم — `nr_resets` |
| ترجمة DT specifier لـ ID | لا (بيستدعي `of_xlate`) | نعم — `of_xlate` implementation |
| تسجيل نفسه | لا (بيستقبل) | نعم — `reset_controller_register()` |

---

### مثال كامل — Reset Controller Driver (Provider Side)

```c
/* مثال مبسط لـ CCU reset driver على Allwinner */

static int sunxi_reset_assert(struct reset_controller_dev *rcdev,
                              unsigned long id)
{
    struct sunxi_ccu *ccu = rcdev_to_sunxi_ccu(rcdev);
    u32 reg = BUS_SOFT_RST_REG(id / 32);   /* أيه الـ register */
    u32 bit = BIT(id % 32);                 /* أيه الـ bit */
    u32 val;

    val = readl(ccu->base + reg);
    val &= ~bit;                            /* clear = assert reset */
    writel(val, ccu->base + reg);
    return 0;
}

static int sunxi_reset_deassert(struct reset_controller_dev *rcdev,
                                unsigned long id)
{
    struct sunxi_ccu *ccu = rcdev_to_sunxi_ccu(rcdev);
    u32 reg = BUS_SOFT_RST_REG(id / 32);
    u32 bit = BIT(id % 32);
    u32 val;

    val = readl(ccu->base + reg);
    val |= bit;                             /* set = deassert reset */
    writel(val, ccu->base + reg);
    return 0;
}

static const struct reset_control_ops sunxi_reset_ops = {
    .assert   = sunxi_reset_assert,
    .deassert = sunxi_reset_deassert,
    /* .reset و .status اختياريين */
};

static int sunxi_ccu_probe(struct platform_device *pdev)
{
    struct sunxi_ccu *ccu;
    /* ... */

    ccu->rcdev.ops           = &sunxi_reset_ops;
    ccu->rcdev.owner         = THIS_MODULE;
    ccu->rcdev.dev           = &pdev->dev;
    ccu->rcdev.of_node       = pdev->dev.of_node;
    ccu->rcdev.nr_resets     = CLK_NUMBER_OF_RESETS;
    ccu->rcdev.of_reset_n_cells = 1;
    /* of_xlate = NULL → يستخدم of_reset_simple_xlate تلقائياً */

    return devm_reset_controller_register(&pdev->dev, &ccu->rcdev);
}
```

---

### مثال كامل — Consumer Driver (Consumer Side)

```c
/* مثال: USB DWC3 driver على platform بتستخدم CCU */

static int dwc3_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct reset_control *rstc;
    int ret;

    /* الـ framework بيحل الـ DT phandle تلقائياً */
    rstc = devm_reset_control_get_exclusive(dev, "usb");
    if (IS_ERR(rstc))
        return PTR_ERR(rstc);  /* -ENOENT لو مش في DT، أو -EBUSY لو محجوز */

    /* إخراج الـ USB controller من الـ reset */
    ret = reset_control_deassert(rstc);
    if (ret)
        return ret;

    /* ... باقي الـ probe ... */

    /* devm هيعمل reset_control_put() تلقائياً عند driver detach */
    /* لو محتاج تعيد assert يدوياً عند remove: */
    /* reset_control_assert(rstc); */
    return 0;
}
```

---

### الـ Optional Resets — التعامل مع الـ Platform Differences

نفس الـ IP block (مثلاً Ethernet MAC) ممكن على platform A يحتاج reset line وعلى platform B لا:

```c
/* بدل ما تعمل #ifdef PLATFORM_A */
rstc = devm_reset_control_get_optional_exclusive(dev, "mac");
/*
 * لو "resets" مش موجود في DT → rstc = NULL
 * لو موجود → rstc = valid handle
 * الـ reset_control_*() functions بتقبل NULL وبترجع 0 بهدوء
 */

reset_control_deassert(rstc);  /* آمن حتى لو rstc == NULL */
```

---

### موقع الـ Subsystem في الـ Kernel

```
kernel/
└── drivers/
    └── reset/
        ├── core.c              ← قلب الـ framework
        ├── reset-simple.c      ← generic driver لـ memory-mapped reset controllers
        ├── Kconfig
        ├── Makefile
        └── (platform-specific drivers مثل)
            ├── reset-sunxi.c   (Allwinner — أحياناً مدمج في drivers/clk/sunxi-ng/)
            ├── reset-imx7.c    (NXP i.MX7)
            ├── reset-berlin.c  (Marvell Berlin)
            └── reset-ti-syscon.c (TI via syscon)

include/linux/
├── reset.h              ← consumer API
└── reset-controller.h   ← provider/driver API
```

الـ `CONFIG_RESET_CONTROLLER` هو الـ Kconfig option اللي بيفعّل الـ framework — لو معمول `n`، كل الـ functions بتبقى inline stubs بترجع 0 أو NULL، فالـ consumer drivers مش محتاجة `#ifdef`.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### Flags, Enums, Config Options — Cheatsheet

#### `RESET_CONTROL_FLAGS_BIT_*` — الـ Bit Flags الأساسية

| Flag | Bit | المعنى |
|------|-----|--------|
| `RESET_CONTROL_FLAGS_BIT_SHARED` | `BIT(0)` | الـ reset مش exclusive — ممكن يتشارك |
| `RESET_CONTROL_FLAGS_BIT_OPTIONAL` | `BIT(1)` | مش لازم يكون موجود في الـ DT |
| `RESET_CONTROL_FLAGS_BIT_ACQUIRED` | `BIT(2)` | الـ exclusive control اتأخد (مش released) |
| `RESET_CONTROL_FLAGS_BIT_DEASSERTED` | `BIT(3)` | يعمل deassert تلقائي بعد الـ get |

#### `enum reset_control_flags` — القيم الجاهزة للاستخدام

| القيمة | Bits فعّالة | الاستخدام |
|--------|------------|-----------|
| `RESET_CONTROL_EXCLUSIVE` | ACQUIRED | exclusive + مأخوذ — الحالة الأشيع |
| `RESET_CONTROL_EXCLUSIVE_DEASSERTED` | ACQUIRED + DEASSERTED | exclusive + deassert تلقائي عند الـ get |
| `RESET_CONTROL_EXCLUSIVE_RELEASED` | 0 | exclusive بس محتاج acquire صريح |
| `RESET_CONTROL_SHARED` | SHARED | shared بدون deassert تلقائي |
| `RESET_CONTROL_SHARED_DEASSERTED` | SHARED + DEASSERTED | shared + deassert تلقائي |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE` | OPTIONAL + ACQUIRED | exclusive اختياري |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE_DEASSERTED` | OPTIONAL + ACQUIRED + DEASSERTED | exclusive اختياري + deassert تلقائي |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE_RELEASED` | OPTIONAL | exclusive اختياري + released |
| `RESET_CONTROL_OPTIONAL_SHARED` | OPTIONAL + SHARED | shared اختياري |
| `RESET_CONTROL_OPTIONAL_SHARED_DEASSERTED` | OPTIONAL + SHARED + DEASSERTED | shared اختياري + deassert تلقائي |

#### Config Options

| Option | الأثر |
|--------|-------|
| `CONFIG_RESET_CONTROLLER=y` | يفعّل كل الـ API الحقيقية |
| `CONFIG_RESET_CONTROLLER=n` | كل الدوال بتبقى stubs بترجع 0 أو NULL — بدون ifdefs في الكود |

---

### الـ Structs المهمة

#### 1. `struct reset_control_ops`

**الغرض:** الـ vtable اللي بيعرّف قدرات الـ reset controller — بيجوز كل الـ callbacks تبقى NULL.

```c
/* in: include/linux/reset-controller.h */
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    /* pulse ذاتي — self-deasserting reset */
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    /* يثبّت الـ reset line في الحالة asserted */
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    /* يرفع الـ reset line */
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
    /* يقرأ الحالة الحالية للـ line */
};
```

| Field | النوع | المعنى |
|-------|-------|--------|
| `reset` | function ptr | pulse ذاتي على الـ line — لازم الـ HW يعمل deassert لوحده |
| `assert` | function ptr | يضغط الـ reset (يحطه في reset state) |
| `deassert` | function ptr | يرفع الـ reset (يخليه يشتغل) |
| `status` | function ptr | يرجع > 0 لو asserted، 0 لو لا، سالب لو مش supported |

**كلهم optional** — لو `assert` مش موجود والـ reset exclusive، `reset_control_assert()` بترجع `-ENOTSUPP`.

---

#### 2. `struct reset_controller_dev`

**الغرض:** يمثّل وحدة الـ reset controller في الـ hardware — الـ driver بيملاه ويسجّله.

```c
/* in: include/linux/reset-controller.h */
struct reset_controller_dev {
    const struct reset_control_ops *ops;     /* الـ vtable */
    struct module *owner;                    /* الـ module اللي سجّله */
    struct list_head list;                   /* entry في reset_controller_list العالمية */
    struct list_head reset_control_head;     /* قائمة الـ reset_control اللي اتطلبت */
    struct device *dev;                      /* الـ device في نموذج الـ driver */
    struct device_node *of_node;             /* الـ DT node كـ phandle target */
    const struct of_phandle_args *of_args;   /* لـ GPIO-based controllers */
    int of_reset_n_cells;                    /* عدد الخلايا في الـ specifier */
    int (*of_xlate)(...);                    /* تحويل الـ DT specifier لـ id */
    unsigned int nr_resets;                  /* عدد الـ reset lines */
};
```

| Field | المعنى العملي |
|-------|--------------|
| `ops` | بوّابة الـ hardware — لازم يتعبّى قبل التسجيل |
| `list` | الـ core بيضيفه في `reset_controller_list` عند التسجيل |
| `reset_control_head` | كل الـ consumers اللي اتعلقوا بيه موجودين هنا |
| `of_node` | الـ consumers بيلاقوا الـ controller بالـ phandle عليه |
| `of_args` | بديل لـ `of_node` في حالة الـ GPIO controllers |
| `of_xlate` | لو `NULL` → يستخدم `of_reset_simple_xlate` (mapping 1:1) |
| `nr_resets` | الـ `of_reset_simple_xlate` بيتحقق إن الـ id أقل من الرقم ده |

---

#### 3. `struct reset_control`

**الغرض:** handle يمثّل consumer's reference على reset line معينة — بيعيش في الـ kernel heap، مش في الـ hardware.

```c
/* in: drivers/reset/core.c — internal struct */
struct reset_control {
    struct reset_controller_dev *rcdev;  /* الـ controller اللي بيتحكم فيه */
    struct list_head list;               /* entry في rcdev->reset_control_head */
    unsigned int id;                     /* رقم الـ reset line في الـ controller */
    struct kref refcnt;                  /* عدد المرات اللي اتطلب فيها نفس الـ control */
    bool acquired;                       /* exclusive + جاهز للاستخدام */
    bool shared;                         /* shared (1) أو exclusive (0) */
    bool array;                          /* ده wrapper لـ array ولا لا */
    atomic_t deassert_count;             /* عدد الـ consumers اللي عملوا deassert */
    atomic_t triggered_count;           /* للـ shared pulse: 0 أو 1 */
};
```

| Field | المعنى التفصيلي |
|-------|----------------|
| `refcnt` | لو كذا driver طلب نفس الـ (rcdev, id, shared) → نفس الـ object، الـ kref بيزيد |
| `acquired` | الـ exclusive released resets محتاجة `reset_control_acquire()` تخليها true |
| `deassert_count` | **الـ shared logic:** أول deassert (counter يبقى 1) → HW call. أول assert يرجعه لـ 0 → HW call |
| `triggered_count` | **الـ shared pulse logic:** أول reset() → HW call + counter=1. التاني → skip. `rearm()` يرجعه لـ 0 |

---

#### 4. `struct reset_control_array`

**الغرض:** wrapper يخلي الـ consumer يتعامل مع مجموعة reset lines كأنها واحدة.

```c
/* in: drivers/reset/core.c */
struct reset_control_array {
    struct reset_control base;           /* لازم يكون أول field — casting بالـ container_of */
    unsigned int num_rstcs;
    struct reset_control *rstc[] __counted_by(num_rstcs);  /* flexible array */
};
```

الـ `base.array = true` بتخلي كل الدوال تعمل dispatch لـ `reset_control_array_*` بدل ما تتكلم مع الـ HW مباشرة.

---

#### 5. `struct reset_control_bulk_data`

**الغرض:** بيانات الـ consumer لمجموعة resets — consumer بيعلنها static ويملاها.

```c
/* in: include/linux/reset.h */
struct reset_control_bulk_data {
    const char          *id;    /* اسم الـ reset line في الـ DT */
    struct reset_control *rstc; /* الـ core بيملأ ده بعد الـ get */
};
```

---

#### 6. `struct reset_gpio_lookup`

**الغرض:** ربط بين GPIO phandle والـ reset controller المؤقت المربوط بيه — internal للـ core.

```c
struct reset_gpio_lookup {
    struct of_phandle_args of_args;    /* الـ GPIO spec من الـ DT */
    struct fwnode_handle *swnode;      /* software node للـ GPIO provider */
    struct list_head list;             /* entry في reset_gpio_lookup_list */
};
```

---

### Struct Relationship Diagram

```
  ┌─────────────────────────────────────────────────────────────┐
  │              reset_controller_list (global)                 │
  │   ┌──────────────────┐    ┌──────────────────┐             │
  │   │ rcdev A          │    │ rcdev B          │   ...        │
  │   │ (reset_controller│◄──►│ (reset_controller│             │
  │   │  _dev)           │    │  _dev)           │             │
  └───┤                  │    │                  │─────────────┘
      │  .ops ──────────►│    │  .ops ──────────►│
      │  (reset_control  │    │  (reset_control  │
      │   _ops)          │    │   _ops)          │
      │                  │    └──────────────────┘
      │  .reset_control_ │
      │   head ──────────┤
      │                  │    ┌──────────────────┐
      └──────────────────┘    │ reset_control X  │
                              │ .rcdev ──────────┼──► rcdev A
                              │ .id = 3          │
                              │ .shared = false  │
                              │ .acquired = true │
                              │ .deassert_count  │
                              │ .triggered_count │
                              └──────────────────┘
                                      ▲ list ▼
                              ┌──────────────────┐
                              │ reset_control Y  │
                              │ .rcdev ──────────┼──► rcdev A
                              │ .id = 3          │
                              │ .shared = true   │
                              └──────────────────┘

  ┌──────────────────────────────────────────┐
  │ reset_control_array                      │
  │  .base (reset_control, .array=true)      │
  │  .num_rstcs = 3                          │
  │  .rstc[0] ────────────────────────────► reset_control X
  │  .rstc[1] ────────────────────────────► reset_control Y
  │  .rstc[2] ────────────────────────────► reset_control Z
  └──────────────────────────────────────────┘

  ┌──────────────────────────────────────────┐
  │ reset_control_bulk_data[]  (consumer)    │
  │  [0] .id = "usb"                         │
  │      .rstc ──────────────────────────── ► reset_control (USB)
  │  [1] .id = "phy"                         │
  │      .rstc ──────────────────────────── ► reset_control (PHY)
  └──────────────────────────────────────────┘
```

---

### Lifecycle Diagrams

#### Controller Driver Lifecycle

```
  ┌─────────────────────────────────────────────────────────────┐
  │  Reset Controller Driver                                    │
  │                                                             │
  │  probe()                                                    │
  │    │                                                        │
  │    ├─ alloc reset_controller_dev                            │
  │    ├─ fill .ops (assert/deassert/reset/status)              │
  │    ├─ set .nr_resets                                        │
  │    ├─ set .of_node = dev->of_node                           │
  │    │                                                        │
  │    └─ devm_reset_controller_register(dev, rcdev)            │
  │           │                                                 │
  │           ├─ devres_alloc() → allocate cleanup entry        │
  │           ├─ reset_controller_register(rcdev)               │
  │           │     ├─ validate: of_node XOR of_args            │
  │           │     ├─ if !of_xlate → set of_reset_simple_xlate │
  │           │     ├─ INIT_LIST_HEAD(&rcdev->reset_control_head)│
  │           │     ├─ mutex_lock(&reset_list_mutex)            │
  │           │     ├─ list_add(&rcdev->list, &reset_controller │
  │           │     │           _list)                          │
  │           │     └─ mutex_unlock(...)                        │
  │           └─ devres_add(dev, cleanup_entry)                 │
  │                                                             │
  │  [driver running — consumers يطلبوا reset controls]        │
  │                                                             │
  │  remove() / driver detach                                   │
  │    └─ devres_release() → reset_controller_unregister()      │
  │           ├─ mutex_lock(&reset_list_mutex)                  │
  │           ├─ list_del(&rcdev->list)                         │
  │           └─ mutex_unlock(...)                              │
  └─────────────────────────────────────────────────────────────┘
```

#### Consumer reset_control Lifecycle

```
  Consumer Driver probe()
  │
  ├─ devm_reset_control_get_exclusive(dev, "usb")
  │      └─ __devm_reset_control_get(dev, "usb", 0, EXCLUSIVE)
  │             └─ __reset_control_get(dev, "usb", 0, flags)
  │                    └─ of_reset_control_get_from_lookup()
  │                           ├─ بيدوّر في reset_controller_list
  │                           ├─ بيستخدم of_xlate يحوّل DT args → id
  │                           ├─ بيدوّر في rcdev->reset_control_head:
  │                           │   لو لاقى نفس (rcdev, id, shared type):
  │                           │     ├─ shared → kref_get() يرجع نفس الـ object
  │                           │     └─ exclusive → -EBUSY
  │                           └─ لو مش موجود: kzalloc() → rstc جديد
  │                                  ├─ .rcdev = rcdev
  │                                  ├─ .id = id
  │                                  ├─ .shared = false
  │                                  ├─ .acquired = true
  │                                  └─ list_add في rcdev->reset_control_head
  │
  ├─ [استخدام]
  │   ├─ reset_control_assert(rstc)    → rcdev->ops->assert(rcdev, id)
  │   ├─ reset_control_deassert(rstc)  → rcdev->ops->deassert(rcdev, id)
  │   ├─ reset_control_reset(rstc)     → rcdev->ops->reset(rcdev, id)
  │   └─ reset_control_status(rstc)    → rcdev->ops->status(rcdev, id)
  │
  └─ remove() / driver detach
       └─ devres_release() → reset_control_put(rstc)
              └─ kref_put(&rstc->refcnt, reset_control_release_mem)
                     └─ [لو refcnt وصل 0]
                            ├─ mutex_lock(&reset_list_mutex)
                            ├─ list_del(&rstc->list)
                            ├─ mutex_unlock(...)
                            └─ kfree(rstc)
```

---

### Call Flow Diagrams

#### `reset_control_assert()` — Exclusive

```
consumer calls reset_control_assert(rstc)
  │
  ├─ rstc == NULL? → return 0  (optional reset)
  ├─ IS_ERR(rstc)? → return -EINVAL
  ├─ rstc->array? → reset_control_array_assert() → loop على كل element
  │
  ├─ rstc->shared == true?
  │     ├─ triggered_count != 0? → WARN + -EINVAL
  │     ├─ atomic_dec_return(&deassert_count) != 0? → return 0  (لسه في consumers)
  │     └─ ops->assert exists? لو لا → return 0  (valid option للـ shared)
  │
  └─ rstc->shared == false (exclusive)?
        ├─ ops->assert exists? لو لا → return -ENOTSUPP
        ├─ rstc->acquired == false? → WARN + return -EPERM
        └─ rstc->rcdev->ops->assert(rcdev, rstc->id)
                └─ [HW driver] يكتب في الـ register بتاع الـ reset
```

#### `reset_control_deassert()` — Shared (مثال: USB PHY مشترك بين core وhost)

```
consumer A calls reset_control_deassert(rstc_shared)
  │
  ├─ shared == true
  ├─ triggered_count != 0? → WARN (مش المفروض تعمل deassert لو في pulse جاي)
  ├─ atomic_inc_return(&deassert_count) == 1?
  │     └─ YES → أول deassert → ops->deassert(rcdev, id) → HW يرفع الـ line
  │
consumer B calls reset_control_deassert(rstc_shared)
  ├─ atomic_inc_return(&deassert_count) == 2?
  │     └─ NO → return 0  (مفيش HW call تاني)
  │
consumer A calls reset_control_assert(rstc_shared)
  ├─ atomic_dec_return(&deassert_count) == 1?
  │     └─ YES (لسه B موجود) → return 0  (مفيش HW call)
  │
consumer B calls reset_control_assert(rstc_shared)
  ├─ atomic_dec_return(&deassert_count) == 0?
  │     └─ YES → آخر consumer → ops->assert(rcdev, id) → HW يضغط الـ line
```

#### `reset_control_reset()` — Shared Pulse (مثال: USB hub reset line)

```
consumer A calls reset_control_reset(rstc_shared)
  │
  ├─ deassert_count != 0? → WARN (لازم يكون asserted أولاً)
  ├─ atomic_inc_return(&triggered_count) == 1?
  │     └─ YES → ops->reset(rcdev, id) → HW pulse
  │              لو HW error → atomic_dec(&triggered_count)
  │
consumer B calls reset_control_reset(rstc_shared)
  ├─ atomic_inc_return(&triggered_count) == 2?
  │     └─ YES → return 0  (no-op — pulse اتعمل خلاص)
  │
consumer A calls reset_control_rearm(rstc_shared)
  ├─ atomic_dec_return(&triggered_count) → الـ counter بيقل
  │
consumer B calls reset_control_rearm(rstc_shared)
  ├─ atomic_dec_return(&triggered_count) → يوصل 0
  │
  └─ دلوقتي الـ consumer التالي يقدر يعمل reset_control_reset() تاني
```

#### Controller Registration Flow

```
reset_controller_register(rcdev)
  │
  ├─ of_node && of_args معاً؟ → -EINVAL  (مش ينفع الاتنين)
  ├─ !of_xlate? → set of_reset_n_cells=1, of_xlate=of_reset_simple_xlate
  ├─ INIT_LIST_HEAD(&rcdev->reset_control_head)
  ├─ mutex_lock(&reset_list_mutex)
  ├─ list_add(&rcdev->list, &reset_controller_list)
  └─ mutex_unlock(&reset_list_mutex)
```

#### Consumer Lookup — من الـ DT للـ HW

```
devm_reset_control_get_exclusive(dev, "phy")
  │
  └─ __devm_reset_control_get(dev, "phy", 0, EXCLUSIVE)
       └─ __reset_control_get(dev, ...)
            └─ __of_reset_control_get(dev->of_node, "phy", 0, flags)
                 │
                 ├─ of_property_read_string_index() → بيلاقي "phy" في resets-names[]
                 ├─ of_parse_phandle_with_args() → بيجيب (phandle, args)
                 │   مثلاً: <&rst_ctrl 5>  →  rcdev = rst_ctrl, args[0] = 5
                 │
                 ├─ mutex_lock(&reset_list_mutex)
                 ├─ بيدوّر في reset_controller_list على الـ rcdev اللي of_node == phandle
                 ├─ rcdev->of_xlate(rcdev, &reset_spec) → id = 5
                 │
                 ├─ بيدوّر في rcdev->reset_control_head:
                 │   لو لاقى rstc بنفس الـ id:
                 │     ├─ نفس النوع (shared) → kref_get()، return نفس الـ rstc
                 │     └─ مختلف (exclusive موجود) → -EBUSY
                 │
                 ├─ مش موجود → kzalloc(sizeof(*rstc))
                 │   rstc->rcdev = rcdev
                 │   rstc->id = 5
                 │   rstc->shared = false
                 │   rstc->acquired = true
                 │   kref_init(&rstc->refcnt)
                 │   list_add(&rstc->list, &rcdev->reset_control_head)
                 │
                 └─ mutex_unlock(&reset_list_mutex)
                      return rstc
```

---

### Locking Strategy

#### الـ Locks الموجودة

| Lock | النوع | بيحمي إيه |
|------|-------|-----------|
| `reset_list_mutex` | `DEFINE_MUTEX` | `reset_controller_list` + `rcdev->reset_control_head` + `rstc->acquired` |
| `reset_gpio_lookup_mutex` | `DEFINE_MUTEX` | `reset_gpio_lookup_list` |
| `rstc->deassert_count` | `atomic_t` | عداد الـ shared deassert — بدون lock |
| `rstc->triggered_count` | `atomic_t` | عداد الـ shared pulse — بدون lock |

#### Lock Ordering

```
reset_list_mutex
  └─ يتأخد لما بنعدّل على:
       ├─ reset_controller_list  (register/unregister)
       ├─ rcdev->reset_control_head  (get/put reset_control)
       └─ rstc->acquired  (acquire/release)

  لا يوجد nested locks → مفيش خطر deadlock
```

#### ليه الـ `deassert_count` و `triggered_count` atomic وبدون mutex؟

لأنها بتتعدّل بالـ `atomic_inc_return()` و `atomic_dec_return()` اللي هي read-modify-write atomic في الـ CPU — كافية لأن الـ logic بتعتمد على قيمة الـ counter بعد العملية مباشرة (`!= 0`, `== 1`, `< 0`) مش على تسلسل عمليات متعددة. الـ `reset_list_mutex` بيحمي فقط اللي محتاج consistency أكبر (linked lists + the `acquired` bool).

#### مثال على Lock Ordering في `reset_control_acquire()`

```c
reset_control_acquire(rstc)
  │
  ├─ mutex_lock(&reset_list_mutex)       /* يمنع race مع get/put تانية */
  │
  ├─ rstc->acquired? → unlock, return 0  /* already acquired */
  │
  ├─ list_for_each_entry(rc, &rcdev->reset_control_head):
  │     لو rc != rstc && rc->id == rstc->id && rc->acquired:
  │         mutex_unlock(...)
  │         return -EBUSY               /* في حد تاني شايله */
  │
  ├─ rstc->acquired = true
  │
  └─ mutex_unlock(&reset_list_mutex)
```

مفيش حاجة تانية بتأخد `reset_list_mutex` وبعدين تأخد lock تاني → مفيش deadlock بالـ design.

---

### ملخص: مين بيستخدم مين

```
Consumer Driver
  │  يطلب handle عبر devm_reset_control_get_*()
  │
  ▼
reset_control  ──(.rcdev)──►  reset_controller_dev  ──(.ops)──►  reset_control_ops
  │                                │                                    │
  │ .id                      .reset_control_head                  .assert()
  │ .shared                  .of_node                             .deassert()
  │ .acquired                .nr_resets                           .reset()
  │ .deassert_count          .of_xlate()                          .status()
  │ .triggered_count              │
  │                               ▼
  │                         Hardware Registers
  │
  └─── [لو .array=true] ───► reset_control_array
                                    │
                              .rstc[0..n] ──► reset_control instances
```
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Consumer API — Single Reset

| Function | النوع | الوصف المختصر |
|---|---|---|
| `reset_control_get_exclusive()` | inline wrapper | جيب reset خاص بيك لوحدك |
| `reset_control_get_shared()` | inline wrapper | جيب reset مشترك مع تانيين |
| `reset_control_get_optional_exclusive()` | inline wrapper | زي exclusive بس مش لازم يبقى موجود |
| `reset_control_get_exclusive_released()` | inline wrapper | exclusive بس released، محتاج acquire قبل الاستخدام |
| `devm_reset_control_get_exclusive()` | inline wrapper | managed نسخة من exclusive get |
| `devm_reset_control_get_exclusive_deasserted()` | inline wrapper | managed + deassert تلقائي |
| `devm_reset_control_get_shared()` | inline wrapper | managed نسخة من shared get |
| `devm_reset_control_get_shared_deasserted()` | inline wrapper | managed shared + deassert تلقائي |
| `reset_control_assert()` | exported | ابعت إشارة reset (active) |
| `reset_control_deassert()` | exported | شيل إشارة reset (خلي الـ peripheral يشتغل) |
| `reset_control_reset()` | exported | pulse كامل على self-deasserting resets |
| `reset_control_rearm()` | exported | اسمح للـ shared reset يتعمل trigger تاني |
| `reset_control_status()` | exported | اسأل هل الـ reset line asserted دلوقتي؟ |
| `reset_control_acquire()` | exported | اكتسب exclusive ownership مؤقت |
| `reset_control_release()` | exported | اتخلى عن الـ exclusive ownership |
| `reset_control_put()` | exported | رجّع الـ reset control handle |
| `device_reset()` | inline wrapper | reset كامل للـ device في سطر واحد |
| `device_reset_optional()` | inline wrapper | زي device_reset بس مش هيفشل لو مش موجود |
| `reset_control_get_count()` | exported | عد الـ resets المتاحة للـ device |

#### Consumer API — Bulk (Multiple Resets)

| Function | الوصف |
|---|---|
| `reset_control_bulk_get_exclusive()` | جيب array من exclusive resets |
| `reset_control_bulk_get_shared()` | جيب array من shared resets |
| `reset_control_bulk_assert()` | assert على الكل بالترتيب |
| `reset_control_bulk_deassert()` | deassert على الكل بالعكس |
| `reset_control_bulk_reset()` | pulse على الكل بالترتيب |
| `reset_control_bulk_acquire()` | acquire على الكل |
| `reset_control_bulk_release()` | release على الكل |
| `reset_control_bulk_put()` | رجّع الكل |
| `devm_reset_control_bulk_get_exclusive()` | managed bulk exclusive |
| `devm_reset_control_bulk_get_shared()` | managed bulk shared |

#### Consumer API — Arrays

| Function | الوصف |
|---|---|
| `devm_reset_control_array_get_exclusive()` | كل الـ resets في الـ DT كـ array |
| `devm_reset_control_array_get_shared()` | array shared |
| `devm_reset_control_array_get_optional_exclusive()` | optional array exclusive |
| `of_reset_control_array_get()` | array من device_node مباشرة |
| `of_reset_control_array_get_exclusive()` | wrapper exclusive |

#### Provider (Controller Driver) API

| Function | الوصف |
|---|---|
| `reset_controller_register()` | سجّل الـ reset controller في الـ framework |
| `reset_controller_unregister()` | ألغي التسجيل |
| `devm_reset_controller_register()` | managed تسجيل |
| `of_reset_simple_xlate()` | ترجمة بسيطة من DT specifier لـ ID |

#### Internal Helpers (لا تُستخدم مباشرة)

| Function | الوصف |
|---|---|
| `__reset_control_get()` | core get من device |
| `__of_reset_control_get()` | core get من device_node |
| `__reset_control_get_internal()` | allocate/lookup reset_control struct |
| `__reset_control_put_internal()` | kref_put داخلي |
| `__device_reset()` | implementation حقيقي لـ device_reset |
| `__devm_reset_control_get()` | implementation حقيقي لـ devm get |

---

### Group 1: Provider Registration — تسجيل الـ Controller

الـ reset controller driver بيملا `struct reset_controller_dev` وبيسجّلها. الـ framework بيحتفظ بـ global list محمية بـ `reset_list_mutex`. كل الـ consumers اللي بيعملوا `get` قبل ما الـ controller يتسجّل بيحصلوا `EPROBE_DEFER` تلقائي.

---

#### `of_reset_simple_xlate`

```c
static int of_reset_simple_xlate(struct reset_controller_dev *rcdev,
                                 const struct of_phandle_args *reset_spec);
```

بتترجم الـ specifier الجاي من الـ device tree لـ numeric ID بسيط. الـ default translation لما الـ driver ميحددش `of_xlate` custom. بتشيك إن الـ index مش أكبر من `nr_resets`.

| Parameter | الشرح |
|---|---|
| `rcdev` | الـ reset controller اللي بيعمل الترجمة |
| `reset_spec` | الـ phandle args من DT، `args[0]` هو الـ index |

**Return:** الـ index نفسه لو صح، `-EINVAL` لو تجاوز `nr_resets`.

**Key details:** بتتسمّى من جوه `__of_reset_control_get()` تحت `reset_list_mutex`. لو الـ driver محتاج mapping معقد (مش linear) يوفر `of_xlate` خاص بيه.

---

#### `reset_controller_register`

```c
int reset_controller_register(struct reset_controller_dev *rcdev);
```

بتضيف الـ rcdev لـ global `reset_controller_list`. لو الـ driver ما حددش `of_xlate`، الـ function بتحط `of_reset_simple_xlate` وبتضبط `of_reset_n_cells = 1`.

| Parameter | الشرح |
|---|---|
| `rcdev` | الـ struct المهيّأ من الـ driver، لازم يكون فيه `ops` و `nr_resets` و `of_node` أو `of_args` |

**Return:** `0` للنجاح، `-EINVAL` لو الـ rcdev عنده `of_node` و `of_args` في نفس الوقت (متناقضان).

**Key details:** بتاخد `reset_list_mutex`، process context فقط. بتعمل `INIT_LIST_HEAD` على `reset_control_head`. أي `get` requests كانت pending بتُحل تلقائي بعد التسجيل.

**Pseudocode:**
```c
// reset_controller_register
if (rcdev->of_node && rcdev->of_args)
    return -EINVAL;           // تعارض في الـ DT source
if (!rcdev->of_xlate)
    set default xlate + n_cells=1;
INIT_LIST_HEAD(&rcdev->reset_control_head);
mutex_lock(&reset_list_mutex);
list_add(rcdev, &reset_controller_list);
mutex_unlock(&reset_list_mutex);
return 0;
```

---

#### `reset_controller_unregister`

```c
void reset_controller_unregister(struct reset_controller_dev *rcdev);
```

بتشيل الـ rcdev من الـ global list. لو في consumers لسه بيمسكوا handles، هيحصل use-after-free لو الـ driver اتـ unload — الـ module reference count المحمي بـ `try_module_get` في `__reset_control_get_internal` بيمنع ده عادةً.

**Key details:** بتاخد `reset_list_mutex`. الـ driver يضمن إن محدش بيستخدم الـ rcdev قبل الاستدعاء.

---

#### `devm_reset_controller_register`

```c
int devm_reset_controller_register(struct device *dev,
                                   struct reset_controller_dev *rcdev);
```

نسخة managed من `reset_controller_register()`. بتخزّن pointer للـ rcdev في devres وبتعمل `reset_controller_unregister()` تلقائي لما الـ device يتفصل.

| Parameter | الشرح |
|---|---|
| `dev` | الـ device اللي هتتربط فيه الـ devres cleanup |
| `rcdev` | الـ reset controller |

**Return:** `0` للنجاح، `-ENOMEM` لو فشل allocate الـ devres، أو error من `reset_controller_register()`.

**Best practice:** استخدمها دايماً في الـ probe function بدل `reset_controller_register()` عشان ما تحتاجش remove function منفصلة.

---

### Group 2: Consumer Acquisition — جيب الـ Handle

الفكرة الأساسية: كل الـ public `get` functions هي wrappers حول `__reset_control_get()` أو `__of_reset_control_get()` اللي بيديهم `enum reset_control_flags` مختلف. الـ flags هي combination من:
- `BIT_SHARED` — shared vs exclusive
- `BIT_ACQUIRED` — acquired immediately vs released
- `BIT_OPTIONAL` — مش لازم يبقى موجود
- `BIT_DEASSERTED` — deassert بعد الـ get تلقائي

---

#### `__of_reset_control_get` (Core Internal)

```c
struct reset_control *
__of_reset_control_get(struct device_node *node, const char *id,
                       int index, enum reset_control_flags flags);
```

الـ core function اللي بتعمل كل الشغل الحقيقي للـ lookup. بتدور على الـ `resets` property في الـ DT باستخدام الـ name أو الـ index، بتعمل phandle parse، بتدور على الـ rcdev المناسب، وبتستدعي `__reset_control_get_internal()`. لو مش لاقية `resets` property وكان `CONFIG_RESET_GPIO` مفعّل، بتحاول تبص على `reset-gpios` property.

**Pseudocode:**
```c
// __of_reset_control_get
if (id)
    index = of_property_match_string(node, "reset-names", id);
ret = of_parse_phandle_with_args(node, "resets", "#reset-cells", index, &args);
if (ret && CONFIG_RESET_GPIO)
    // fallback: parse "reset-gpios" and create aux GPIO device
    gpio_fallback = true;
mutex_lock(&reset_list_mutex);
rcdev = __reset_find_rcdev(&args, gpio_fallback);
if (!rcdev)
    rstc = ERR_PTR(-EPROBE_DEFER);   // controller لسه متسجلش
else
    rstc_id = rcdev->of_xlate(rcdev, &args);
    rstc = __reset_control_get_internal(rcdev, rstc_id, flags);
mutex_unlock(&reset_list_mutex);
return rstc;
```

**Key details:** `-EPROBE_DEFER` بيرجع لو الـ controller متسجلش بعد. الـ `reset_list_mutex` بيحمي الـ lookup والـ internal get في نفس الوقت.

---

#### `__reset_control_get_internal` (Core Internal)

```c
static struct reset_control *
__reset_control_get_internal(struct reset_controller_dev *rcdev,
                             unsigned int index,
                             enum reset_control_flags flags);
```

بتعمل allocate أو reuse لـ `struct reset_control`. لو في control بنفس الـ id موجود بالفعل وكان shared وإحنا طالبينه shared، بتعمل `kref_get()` وبترجعه. لو exclusive، بتعمل allocate جديد.

**Key details:** لازم تتسمى تحت `reset_list_mutex`. بتعمل `try_module_get()` على الـ owner module، لو فشلت بترجع `-ENODEV`. **لا تستدعيها مباشرة.**

---

#### `reset_control_get_exclusive`

```c
static inline struct reset_control *
__must_check reset_control_get_exclusive(struct device *dev, const char *id);
```

بتجيب exclusive handle للـ reset المحدد بالاسم. لو حد تاني مسكه قبل كده بـ exclusive، بترجع `-EBUSY`.

| Parameter | الشرح |
|---|---|
| `dev` | الـ device اللي عايز الـ reset |
| `id` | اسم الـ reset زي ما هو في `reset-names` في الـ DT، أو NULL لأول reset |

**Return:** `struct reset_control *` أو `IS_ERR()`. مش بترجع NULL أبداً (غير optional).

**Caller must:** يستدعي `reset_control_put()` لما يخلص، أو يستخدم `devm_` variant.

---

#### `reset_control_get_shared`

```c
static inline struct reset_control *
reset_control_get_shared(struct device *dev, const char *id);
```

بتجيب shared handle. ممكن أكتر من driver يمسك نفس الـ reset. الـ assert/deassert بيشتغل بـ reference counting.

**Key rule:** على shared reset، **ممنوع** تستدعي `reset_control_reset()`. لازم تستخدم `assert/deassert` فقط. وكمان `assert` من غير `deassert` أولاً ممنوع.

---

#### `reset_control_get_exclusive_released`

```c
static inline struct reset_control *
__must_check reset_control_get_exclusive_released(struct device *dev, const char *id);
```

بتجيب exclusive handle بس بـ state = released (مش acquired). ده بيسمح لأكتر من consumer يتشارك نفس الـ reset line بشكل controlled عن طريق `acquire/release` explicit.

**Use case:** driver A و driver B محتاجين نفس الـ reset في أوقات مختلفة زي power domain resets في SoCs.

---

#### `devm_reset_control_get_exclusive`

```c
static inline struct reset_control *
__must_check devm_reset_control_get_exclusive(struct device *dev, const char *id);
```

Managed wrapper. بيضيف devres callback تستدعي `reset_control_put()` تلقائياً لما الـ device يتـ detach.

**Always prefer** الـ `devm_` variants في probe functions.

---

#### `devm_reset_control_get_exclusive_deasserted`

```c
static inline struct reset_control * __must_check
devm_reset_control_get_exclusive_deasserted(struct device *dev, const char *id);
```

بتجيب الـ reset وبتعمل `deassert` فيه في نفس الوقت. لما الـ device يتـ detach، بيتعمل `assert` وبعدين `put` تلقائياً. مثالي لـ peripherals اللي محتاجة تبقى out of reset طول فترة الـ probe.

**Implementation note:** داخلياً، الـ `devm_reset_control_release_deasserted()` هي الـ devres callback اللي بتعمل assert + put معاً.

---

#### `__devm_reset_control_get` (Core Internal)

```c
struct reset_control *
__devm_reset_control_get(struct device *dev, const char *id, int index,
                         enum reset_control_flags flags);
```

الـ implementation الحقيقي لكل `devm_reset_control_get_*` functions. بتشوف لو الـ `DEASSERTED` bit متحط، بتختار الـ devres destructor المناسب (`devm_reset_control_release_deasserted` بدل `devm_reset_control_release`).

---

### Group 3: Runtime Control — التحكم الفعلي

---

#### `reset_control_assert`

```c
int reset_control_assert(struct reset_control *rstc);
```

بتحط الـ peripheral في حالة reset (assert = reset active). السلوك بيختلف حسب نوع الـ reset:
- **Exclusive:** بتستدعي `ops->assert()` مباشرة. لو `ops->assert` مش موجود، بترجع `-ENOTSUPP`.
- **Shared:** بتـ decrement الـ `deassert_count`. لو وصل لصفر، بتستدعي `ops->assert()`. لو الـ count لسه > 0، بترجع 0 من غير ما تعمل حاجة فعلية.

| Parameter | الشرح |
|---|---|
| `rstc` | الـ handle، لو NULL (optional) بترجع 0 |

**Return:** `0` للنجاح، `-EINVAL` لو `IS_ERR(rstc)` أو لو shared و `triggered_count != 0`، `-EPERM` لو exclusive وغير acquired.

**Critical:** على shared reset، ممنوع تستدعي assert من غير deassert أولاً (`WARN_ON` بيتطلع). مش مضمون إن الـ line اتـ assert فعلاً لو في consumers تانيين.

**Pseudocode:**
```c
// reset_control_assert
if (!rstc) return 0;           // optional
if (rstc->array) return array_assert();
if (rstc->shared) {
    WARN_ON(triggered_count != 0);    // ممنوع مع reset()
    WARN_ON(deassert_count == 0);     // ممنوع assert من غير deassert
    if (atomic_dec_return(&deassert_count) != 0)
        return 0;              // في حد تاني لسه deasserted
    if (!ops->assert) return 0;       // shared: OK مش لازم assert
} else {
    if (!ops->assert) return -ENOTSUPP;
    if (!acquired) return -EPERM;
}
return ops->assert(rcdev, id);
```

---

#### `reset_control_deassert`

```c
int reset_control_deassert(struct reset_control *rstc);
```

بتخرج الـ peripheral من حالة reset. بعد ما بتنجح، الـ peripheral بيبدأ تشغيله.
- **Exclusive:** بتستدعي `ops->deassert()`. لو مش موجود، بترجع 0 (assuming self-deassert).
- **Shared:** بتـ increment الـ `deassert_count`. لو بقى 1 (أول واحد)، بتستدعي `ops->deassert()`.

**Return:** `0` للنجاح، `-EPERM` لو exclusive وغير acquired، `-EINVAL` لو triggered_count != 0.

**Key detail:** لو `ops->deassert` مش موجود، الـ framework بيافترض إن الـ reset self-deasserting وبيرجع 0 من غير ما يعمل حاجة. ده الـ behavior الصح لـ pulse-only controllers.

**Pseudocode:**
```c
// reset_control_deassert
if (!rstc) return 0;
if (rstc->array) return array_deassert();
if (rstc->shared) {
    WARN_ON(triggered_count != 0);
    if (atomic_inc_return(&deassert_count) != 1)
        return 0;              // مش أول واحد، مش محتاج يعمل حاجة
} else {
    if (!acquired) return -EPERM;
}
if (!ops->deassert) return 0; // self-deasserting assumed
return ops->deassert(rcdev, id);
```

---

#### `reset_control_reset`

```c
int reset_control_reset(struct reset_control *rstc);
```

بتعمل pulse كامل (assert + deassert) على الـ reset line. مخصصة لـ self-deasserting resets اللي ما بتدعمش assert/deassert منفصلين. على shared resets، **أول استدعاء بس** اللي بيعمل pulse فعلي، الباقي no-op لحد ما يتعمل `reset_control_rearm()`.

**Return:** `0` للنجاح، `-ENOTSUPP` لو `ops->reset` مش موجود، `-EPERM` لو exclusive وغير acquired، `-EINVAL` لو shared و `deassert_count != 0`.

**Key detail:** على shared، بتـ increment `triggered_count`. لو > 1، بترجع 0 من غير ما تعمل pulse. لو الـ `ops->reset` فشل، بتـ decrement `triggered_count` تاني.

**Pseudocode:**
```c
// reset_control_reset
if (!rstc) return 0;
if (array) return array_reset();
if (!ops->reset) return -ENOTSUPP;
if (rstc->shared) {
    WARN_ON(deassert_count != 0);       // مش compatible مع assert/deassert
    if (atomic_inc_return(&triggered_count) != 1)
        return 0;                        // مش أول caller
} else {
    if (!acquired) return -EPERM;
}
ret = ops->reset(rcdev, id);
if (shared && ret)
    atomic_dec(&triggered_count);       // rollback لو فشل
return ret;
```

---

#### `reset_control_rearm`

```c
int reset_control_rearm(struct reset_control *rstc);
```

بتسمح لـ shared reset يتعمل trigger تاني. على shared resets اللي استخدمت `reset_control_reset()`، الـ pulse بيتعمل مرة واحدة بس لكل "جيل". `rearm` بيدّي إشارة إن الـ consumer خلص من الـ reset ومش محتاجه، فبيعمل `dec` على `triggered_count` عشان الـ next caller يقدر يعمل pulse تاني.

| Parameter | الشرح |
|---|---|
| `rstc` | لو NULL (optional) بترجع 0 |

**Return:** `0` للنجاح، `-EINVAL` لو `deassert_count != 0`، `-EPERM` لو exclusive وغير acquired.

**Caller balance:** كل `reset_control_reset()` لازم يقابله `reset_control_rearm()` على shared resets.

---

#### `reset_control_status`

```c
int reset_control_status(struct reset_control *rstc);
```

بتسأل الـ hardware عن الحالة الفعلية للـ reset line. مش كل الـ controllers بيدعموها.

**Return:** قيمة موجبة لو الـ line asserted، `0` لو deasserted أو `rstc` = NULL، `-ENOTSUPP` لو `ops->status` مش موجود، `-EINVAL` لو array handle.

**Note:** الوحيدة اللي **مش بتشتغل** على array handles. محتاج single reset handle.

---

### Group 4: Acquire/Release — التحكم في الملكية

الـ acquire/release pattern بيسمح لـ "exclusive released" resets تتشارك بين أكتر من consumer بشكل serialized. الأيديولوجيا: بدل ما كل consumer يعمل exclusive get (اللي بتفشل لو حد تاني مسكها)، بيعملوا كلهم exclusive-released get، وبعدين كل واحد يعمل acquire/release لما محتاج.

---

#### `reset_control_acquire`

```c
int reset_control_acquire(struct reset_control *rstc);
```

بتكتسب الـ exclusive ownership المؤقت للـ reset. بتبص على كل الـ reset controls بنفس الـ ID على الـ rcdev، لو حد فيهم acquired ترجع `-EBUSY`.

**Return:** `0` للنجاح أو لو مخصوص exclusive غير released، `-EBUSY` لو حد تاني acquired، `-EINVAL` لو IS_ERR.

**Locking:** بتاخد `reset_list_mutex` وبتعمل linear search على `reset_control_head`.

**Pseudocode:**
```c
// reset_control_acquire
mutex_lock(&reset_list_mutex);
if (rstc->acquired) { unlock; return 0; }  // already acquired
list_for_each(rc, &rcdev->reset_control_head)
    if (rc->id == rstc->id && rc->acquired)
        { unlock; return -EBUSY; }
rstc->acquired = true;
mutex_unlock(&reset_list_mutex);
return 0;
```

---

#### `reset_control_release`

```c
void reset_control_release(struct reset_control *rstc);
```

بترجع الـ exclusive ownership. بتضبط `rstc->acquired = false`، اللي بيخلي caller تاني يقدر يعمل acquire.

**No return value.** لو NULL أو IS_ERR، بتتجاهل.

---

### Group 5: Bulk Operations — عمليات الـ Bulk

الـ Bulk API بيوفر convenience لما الـ driver عنده أكتر من reset line. الـ struct المستخدم هو `reset_control_bulk_data` اللي فيه `id` (اسم الـ reset) و `rstc` (الـ handle اللي بيتملى تلقائياً).

---

#### `reset_control_bulk_assert`

```c
int reset_control_bulk_assert(int num_rstcs,
                              struct reset_control_bulk_data *rstcs);
```

بتعمل assert بالترتيب من 0 لـ `num_rstcs-1`. **لو أي reset فشل، بتعمل rollback** وتعمل deassert على اللي اتعمل assertion بالفعل. ده بيضمن atomicity على مستوى الـ software.

---

#### `reset_control_bulk_deassert`

```c
int reset_control_bulk_deassert(int num_rstcs,
                                struct reset_control_bulk_data *rstcs);
```

**ملحوظة مهمة:** بتعمل deassert بالـ **عكسي** (من `num_rstcs-1` لـ 0). لو أي واحد فشل، بتعمل rollback وتعمل assert على اللي اتعمل deassert. الترتيب العكسي للـ deassert مقابل الترتيب الطبيعي للـ assert هو convention شائع في embedded systems.

---

#### `reset_control_bulk_reset`

```c
int reset_control_bulk_reset(int num_rstcs,
                             struct reset_control_bulk_data *rstcs);
```

بتعمل pulse على كل الـ resets بالترتيب. **مش بتعمل rollback** لو واحد فشل — بتوقف وبترجع الـ error.

---

### Group 6: Array Operations — عمليات الـ Array

الـ array handle هو opaque `struct reset_control *` اللي جوّاه `struct reset_control_array`. الـ `rstc->array = true` flag هو اللي بيميّزه. كل runtime functions زي assert/deassert/reset بتشوف الـ flag ده وبتعمل dispatch للـ array variant.

---

#### `of_reset_control_array_get`

```c
struct reset_control *
of_reset_control_array_get(struct device_node *np,
                           enum reset_control_flags flags);
```

بتعد كل الـ resets في الـ `resets` property باستخدام `of_reset_control_get_count()` وبتعمل allocate `reset_control_array` بـ flexible array member. بتعمل loop وتجيب كل reset بـ index. لو أي واحد فشل، بتعمل cleanup للاللي اتجابوا.

**Return:** `struct reset_control *` يشير لـ `reset_control_array.base`، أو `IS_ERR()`.

**Pseudocode:**
```c
// of_reset_control_array_get
num = of_reset_control_get_count(np);
resets = kzalloc(struct_size(resets, rstc, num));
resets->num_rstcs = num;
for (i = 0; i < num; i++)
    resets->rstc[i] = __of_reset_control_get(np, NULL, i, flags);
resets->base.array = true;
return &resets->base;
```

---

#### `devm_reset_control_array_get`

```c
struct reset_control *
devm_reset_control_array_get(struct device *dev,
                             enum reset_control_flags flags);
```

نسخة managed من `of_reset_control_array_get()`. بتعمل `reset_control_put()` تلقائياً على الـ array كله عند الـ detach. الـ `reset_control_put()` بتـ detect إن الـ handle array وبتستدعي `reset_control_array_put()` اللي بتحرر كل element.

---

#### `reset_control_get_count` / `of_reset_control_get_count`

```c
int reset_control_get_count(struct device *dev);
static int of_reset_control_get_count(struct device_node *node);
```

بيعدوا عدد الـ resets المتاحة عن طريق `of_count_phandle_with_args()` على الـ `resets` property. مفيد قبل ما تعمل array get أو loop manual.

**Return:** عدد موجب، `-ENOENT` لو مفيش resets، `-EINVAL` لو `node` = NULL.

---

### Group 7: One-shot Device Reset — الأسهل للاستخدام

---

#### `device_reset` / `device_reset_optional`

```c
static inline int __must_check device_reset(struct device *dev);
static inline int device_reset_optional(struct device *dev);
```

اللي وراهم `__device_reset(dev, optional)`. بيعملوا exclusive get + reset + put في خطوة واحدة. مثالي للـ simple devices اللي عندها reset واحد ومحتاجة reset في الـ probe.

**الفرق:** `device_reset()` بترجع error لو مفيش reset controller. `device_reset_optional()` بترجع 0.

**ACPI support:** لو الـ device عنده ACPI handle، بتحاول تستدعي `_RST` method أولاً قبل ما تبص على الـ DT.

**Pseudocode:**
```c
// __device_reset
#ifdef CONFIG_ACPI
if (acpi_handle && acpi_has_method(handle, "_RST"))
    return acpi_evaluate_object(handle, "_RST");
#endif
rstc = __reset_control_get(dev, NULL, 0, flags);
ret = reset_control_reset(rstc);
reset_control_put(rstc);
return ret;
```

---

### Group 8: Internal Lifecycle — الـ Reference Counting

الـ `struct reset_control` بيستخدم `struct kref` للـ reference counting:

```
get_exclusive()  → kzalloc جديد + kref_init(1) + try_module_get()
get_shared()     → لو موجود: kref_get() | مش موجود: kzalloc + kref_init(1)
put()            → kref_put() → لو وصل 0: module_put() + list_del() + kfree()
```

الـ `reset_list_mutex` بيحمي كل الـ list operations. الـ `kref` بيضمن إن الـ `struct reset_control` مش بيتحرر وفي حد لسه مسكه.

---

### Group 9: الـ Structs الأساسية

#### `struct reset_control` (Internal, `drivers/reset/core.c`)

```c
struct reset_control {
    struct reset_controller_dev *rcdev;  /* الـ controller اللي بيمتلكه */
    struct list_head list;               /* في reset_control_head */
    unsigned int id;                     /* رقم الـ line */
    struct kref refcnt;                  /* reference count */
    bool acquired;                       /* exclusive ownership */
    bool shared;                         /* shared vs exclusive */
    bool array;                          /* array handle؟ */
    atomic_t deassert_count;            /* shared: عدد المعمولين deassert */
    atomic_t triggered_count;           /* shared: عدد المعمولين reset() */
};
```

#### `struct reset_controller_dev` (`include/linux/reset-controller.h`)

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops; /* callbacks: reset/assert/deassert/status */
    struct module *owner;                /* لمنع unload مبكر */
    struct list_head list;               /* في reset_controller_list */
    struct list_head reset_control_head; /* الـ consumers اللي جابوا handles */
    struct device *dev;                  /* الـ device model entity */
    struct device_node *of_node;         /* DT node كـ phandle target */
    const struct of_phandle_args *of_args; /* لـ GPIO-based controllers */
    int of_reset_n_cells;               /* عدد الـ cells في الـ DT specifier */
    int (*of_xlate)(...);               /* ترجمة DT specifier → ID */
    unsigned int nr_resets;             /* إجمالي عدد الـ reset lines */
};
```

#### `struct reset_control_ops` (`include/linux/reset-controller.h`)

```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

كل الـ callbacks اختيارية. الـ framework بيتعامل مع absence كل منها بشكل مختلف:
- **`reset` مش موجود:** `reset_control_reset()` بترجع `-ENOTSUPP`.
- **`assert` مش موجود:** exclusive assert بترجع `-ENOTSUPP`؛ shared assert بترجع 0 (acceptable).
- **`deassert` مش موجود:** بيترجع 0 (يُفترض self-deasserting).
- **`status` مش موجود:** `reset_control_status()` بترجع `-ENOTSUPP`.

---

### مثال عملي — Driver يستخدم الـ API

#### Consumer Driver (Single Exclusive Reset)

```c
/* في probe() */
struct reset_control *rstc;

/* جيب الـ reset وخليه out-of-reset في سطر واحد */
rstc = devm_reset_control_get_exclusive_deasserted(dev, "usb");
if (IS_ERR(rstc))
    return PTR_ERR(rstc);
/* الـ peripheral اشتغل، وعند detach هيتعمل assert + put تلقائي */
```

#### Consumer Driver (Shared Reset — Multi-driver scenario)

```c
/* في probe() — كل driver بيعمل ده */
struct reset_control *rstc;
rstc = devm_reset_control_get_shared(dev, "bus");
if (IS_ERR(rstc))
    return PTR_ERR(rstc);

reset_control_deassert(rstc);  /* deassert_count++ */

/* في remove() */
reset_control_assert(rstc);    /* deassert_count-- → لو وصل 0، assert فعلي */
```

#### Provider Driver (Controller)

```c
static int my_reset_assert(struct reset_controller_dev *rcdev,
                           unsigned long id)
{
    struct my_reset_priv *priv = container_of(rcdev, struct my_reset_priv, rcdev);
    /* اكتب في الـ register */
    writel(BIT(id), priv->base + RESET_ASSERT_REG);
    return 0;
}

static const struct reset_control_ops my_reset_ops = {
    .assert   = my_reset_assert,
    .deassert = my_reset_deassert,
    .status   = my_reset_status,
};

static int my_reset_probe(struct platform_device *pdev)
{
    struct my_reset_priv *priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL);
    priv->rcdev.ops       = &my_reset_ops;
    priv->rcdev.owner     = THIS_MODULE;
    priv->rcdev.of_node   = pdev->dev.of_node;
    priv->rcdev.nr_resets = 64;
    /* of_xlate مش محتاجها — of_reset_simple_xlate كافية */
    return devm_reset_controller_register(&pdev->dev, &priv->rcdev);
}
```

---

### جدول الـ Locking

| Context | الـ Lock |
|---|---|
| `reset_controller_register/unregister` | `reset_list_mutex` |
| `reset_control_acquire/release` | `reset_list_mutex` |
| `reset_control_put` | `reset_list_mutex` |
| `__reset_control_get_internal` | `reset_list_mutex` (caller) |
| `__reset_control_put_internal` | `reset_list_mutex` (caller) |
| `deassert_count` read/write | `atomic_t` (lock-free) |
| `triggered_count` read/write | `atomic_t` (lock-free) |
| `reset_gpio_lookup_list` | `reset_gpio_lookup_mutex` |

كل الـ runtime operations (assert/deassert/reset/status) **ما بتاخدش mutex** — بتاخد `atomic` operations للـ counters وبتستدعي الـ ops مباشرة. ده بيخلي الـ hot path خفيف.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs

الـ reset controller framework مش بيسجّل entries كتير في debugfs بشكل افتراضي، بس ممكن تلاقي بعض المعلومات عن طريق:

```bash
# اتأكد إن debugfs متماونت
mount | grep debugfs
# لو مش متماونت
mount -t debugfs none /sys/kernel/debug

# شوف الـ reset controllers اللي اتسجلت
ls /sys/kernel/debug/reset/ 2>/dev/null || echo "No reset debugfs entries"

# لو الـ platform بيدعم reset debugfs (بعض الـ SoCs زي Rockchip بتضيف entries)
cat /sys/kernel/debug/reset/<controller_name>/reset_status 2>/dev/null
```

> **ملاحظة:** الـ reset framework نفسه معندوش dedicated debugfs directory في mainline kernel — الـ debugging بيتم عن طريق sysfs + tracing + printk.

---

#### 2. مدخلات الـ sysfs

```bash
# اعرض كل الـ reset controllers المسجّلين في النظام
ls /sys/bus/platform/drivers/ | grep reset

# للوصول لـ reset controller device معيّن
ls /sys/bus/platform/devices/<reset-ctrl-device>/

# شوف الـ driver اللي اتعيّن للـ device
cat /sys/bus/platform/devices/<reset-ctrl-device>/driver

# اعرف الـ device tree compatible string
cat /sys/bus/platform/devices/<reset-ctrl-device>/of_node/compatible

# شوف الـ power state (مهم — الـ reset controller لازم يكون powered)
cat /sys/bus/platform/devices/<reset-ctrl-device>/power/runtime_status

# مثال عملي على Raspberry Pi
ls /sys/bus/platform/devices/ | grep reset
# Output مثلا: fe200000.gpio (بيحتوي reset lines)
```

---

#### 3. الـ ftrace — Tracepoints وأحداث الـ Reset

الـ reset framework بيستخدم tracepoints معرّفة في `include/trace/events/reset.h`:

```bash
# اعرض كل الـ tracepoints المتعلقة بالـ reset
ls /sys/kernel/tracing/events/reset/

# الـ events الأساسية:
# reset_control_reset      — لما بيتعمل reset pulse
# reset_control_assert     — لما بيتعمل assert
# reset_control_deassert   — لما بيتعمل deassert
# reset_control_status     — لما بيتسأل عن الـ status

# فعّل كل أحداث الـ reset
echo 1 > /sys/kernel/tracing/events/reset/enable

# أو فعّل event معيّن
echo 1 > /sys/kernel/tracing/events/reset/reset_control_assert/enable
echo 1 > /sys/kernel/tracing/events/reset/reset_control_deassert/enable

# ابدأ الـ tracing
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل الـ driver أو افعل اللي يستدعي الـ reset
modprobe <your_driver>

# اقرأ النتايج
cat /sys/kernel/tracing/trace

# مثال على output متوقع
# <driver>-123 [001] .... reset_control_assert: id=5 triggered
# <driver>-123 [001] .... reset_control_deassert: id=5 triggered

# صفّر بعد الانتهاء
echo 0 > /sys/kernel/tracing/tracing_on
echo > /sys/kernel/tracing/trace
```

**استخدام function_graph لتتبّع دوال الـ reset:**

```bash
echo function_graph > /sys/kernel/tracing/current_tracer
echo 'reset_control_assert
reset_control_deassert
reset_control_reset
reset_control_status' > /sys/kernel/tracing/set_graph_function
echo 1 > /sys/kernel/tracing/tracing_on
# ... شغّل الـ driver ...
cat /sys/kernel/tracing/trace
```

---

#### 4. الـ printk والـ Dynamic Debug

```bash
# فعّل الـ dynamic debug لكل دوال الـ reset core
echo 'file drivers/reset/core.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لـ driver معيّن بيستخدم reset API
echo 'file drivers/clk/clk-bcm2835.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل debug لكل الـ reset subsystem (core + كل الـ drivers)
echo 'module reset* +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages في dmesg
dmesg -w | grep -i reset

# أو عبر kernel log level
echo 8 > /proc/sys/kernel/printk   # فعّل DEBUG messages
dmesg --level=debug | grep reset
```

**عند الـ boot — kernel command line:**

```bash
# في /etc/default/grub أضف:
GRUB_CMDLINE_LINUX="dyndbg='file drivers/reset/core.c +p'"
# أو
GRUB_CMDLINE_LINUX="reset.dyndbg=+p"
```

---

#### 5. خيارات الـ Kernel Config للـ Debugging

| Config Option | الوصف | القيمة المقترحة |
|---|---|---|
| `CONFIG_RESET_CONTROLLER` | تفعيل الـ reset framework أصلاً | `y` |
| `CONFIG_DEBUG_KERNEL` | يفتح باب كل خيارات الـ debug | `y` |
| `CONFIG_DYNAMIC_DEBUG` | dynamic debug لـ pr_debug() | `y` |
| `CONFIG_TRACING` | تفعيل الـ ftrace infrastructure | `y` |
| `CONFIG_FUNCTION_TRACER` | تتبع استدعاءات الدوال | `y` |
| `CONFIG_STACK_TRACER` | stack tracing مع ftrace | `y` |
| `CONFIG_LOCKDEP` | كشف مشاكل الـ locking (مهم لـ shared resets) | `y` |
| `CONFIG_PROVE_LOCKING` | التحقق الصارم من الـ locks | `y` (dev only) |
| `CONFIG_DEBUG_OBJECTS` | كشف use-after-free في الـ objects | `y` |
| `CONFIG_KASAN` | كشف memory errors | `y` (dev only) |
| `CONFIG_PM_DEBUG` | debug الـ power management + reset | `y` |
| `CONFIG_OF_DYNAMIC` | dynamic device tree — مهم لـ DT debugging | `y` |

```bash
# تحقق من الـ config الحالية
zcat /proc/config.gz | grep -E 'RESET|DEBUG_KERNEL|DYNAMIC_DEBUG|TRACING'
# أو
grep -E 'CONFIG_RESET|CONFIG_DEBUG' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**استخدام `devlink` (لـ network devices فقط، مش مباشر للـ reset):**

```bash
# الـ reset controller مش بيتعامل مع devlink مباشرة
# بس لو الـ network device بيستخدم reset:
devlink dev show
devlink dev reload <device>   # بيستدعي reset داخلياً
```

**أداة `reset-gpio` للـ debugging عبر GPIO:**

```bash
# لو الـ reset controller مبني على GPIO (gpio-reset driver)
gpioinfo | grep -i reset
gpioset <chip> <line>=1    # assert reset
sleep 0.1
gpioset <chip> <line>=0    # deassert reset
```

**`of_node_get` inspection عبر سكريبت:**

```bash
# اعرض كل الـ resets المعرّفة في DT لـ device معيّن
cat /sys/bus/platform/devices/<device>/of_node/resets 2>/dev/null
# أو عبر dtc
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -A3 'resets'
```

---

#### 7. جدول رسائل الـ Error الشايعة

| رسالة الـ Kernel | المعنى | الحل |
|---|---|---|
| `reset controller not found` | مفيش reset controller اتسجّل للـ device | تأكد إن الـ reset controller driver اتلود قبل الـ consumer driver |
| `reset: failed to get reset` | `devm_reset_control_get()` فشلت | تحقق من الـ DT: هل `resets` property موجودة وصح؟ |
| `-EPROBE_DEFER` | الـ reset controller لسه مش جاهز وقت probe | طبيعي — الـ kernel هيعيد الـ probe تلقائياً، بس لو استمر راجع ترتيب الـ initcalls |
| `-EINVAL` في `reset_control_assert` | الـ reset control مش exclusive أو النوع غلط | تأكد إنك طلبته بـ `get_exclusive()` مش `get_shared()` |
| `reset_control_assert called more times than reset_control_deassert` | unbalanced calls في shared reset | راجع الـ refcount — كل `deassert` لازم يتقابل بـ `assert` |
| `WARNING: reset id N is already in use` | محاولة طلب exclusive reset وهو محجوز | تحقق إن مفيش driver تاني بيستخدمه |
| `reset: cannot reset a shared reset control` | استدعاء `reset_control_reset()` على shared reset بعد أول trigger | لازم تستدعي `reset_control_rearm()` الأول |
| `of_reset_simple_xlate: invalid reset id` | الـ reset ID في DT أكبر من `nr_resets` | تحقق من `reset_controller_dev.nr_resets` في الـ controller driver |
| `reset: no reset specified` | الـ device مش عنده `resets` property في DT | أضف الـ property أو استخدم `get_optional` |

---

#### 8. نقاط استراتيجية لـ `dump_stack()` و `WARN_ON()`

**في الـ controller driver (مثال):**

```c
static int my_reset_assert(struct reset_controller_dev *rcdev,
                            unsigned long id)
{
    struct my_reset_priv *priv = container_of(rcdev, struct my_reset_priv, rcdev);

    /* تحقق إن الـ id منطقي */
    if (WARN_ON(id >= priv->rcdev.nr_resets)) {
        dump_stack();
        return -EINVAL;
    }

    /* تحقق إن الـ hardware accessible */
    if (WARN_ON(IS_ERR_OR_NULL(priv->base))) {
        pr_err("reset: base address is invalid\n");
        return -EIO;
    }

    pr_debug("reset: asserting reset line %lu\n", id);
    /* ... الـ implementation ... */
    return 0;
}
```

**في الـ consumer driver:**

```c
static int my_device_probe(struct platform_device *pdev)
{
    struct reset_control *rst;

    rst = devm_reset_control_get_exclusive(&pdev->dev, "main");
    if (IS_ERR(rst)) {
        /* EPROBE_DEFER = طبيعي، غيره = مشكلة */
        if (PTR_ERR(rst) != -EPROBE_DEFER)
            WARN(1, "failed to get reset: %ld\n", PTR_ERR(rst));
        return PTR_ERR(rst);
    }

    /* قبل ما تشغّل الـ hardware تأكد من الـ reset state */
    WARN_ON(reset_control_status(rst) <= 0);  /* لازم يكون deasserted */
}
```

---

### Hardware Level

---

#### 1. التحقق إن الـ hardware state بيطابق الـ kernel state

```bash
# 1. اقرأ الـ status عبر API
# في kernel code:
int status = reset_control_status(rst);
pr_info("reset line status: %s\n", status ? "asserted" : "deasserted");

# 2. قارن بقراءة الـ register مباشرة (devmem2)
# مثال: Allwinner A64 reset register على 0x01C202C0
devmem2 0x01C202C0 w
# لو bit معيّن = 0 → asserted, 1 → deasserted (على معظم الـ SoCs)

# 3. استخدم /proc/iomem تتعرف على عنوان الـ reset controller
grep -i reset /proc/iomem
```

---

#### 2. تقنيات قراءة الـ Registers

```bash
# تأكد من تثبيت devmem2
apt-get install devmem2   # Debian/Ubuntu

# اقرأ reset register (32-bit)
devmem2 <physical_address> w

# مثال عملي — Raspberry Pi 4 (BCM2711) PM_WDOG reset register
devmem2 0xFE100024 w

# استخدام /dev/mem مباشرة عبر Python
python3 -c "
import mmap, os
fd = os.open('/dev/mem', os.O_RDONLY)
with mmap.mmap(fd, 4096, offset=0xFE100000) as m:
    val = int.from_bytes(m[0x24:0x28], 'little')
    print(f'Reset reg: 0x{val:08X}')
os.close(fd)
"

# باستخدام io utility (من package 'io' أو busybox)
io -4 -r <address>

# ملاحظة: لو الـ CONFIG_STRICT_DEVMEM مفعّل هتحتاج kernel module
# أو تعطّله مؤقتاً عبر /proc/sys/kernel/perf_event_paranoid
```

**kernel module لقراءة الـ registers في الـ runtime:**

```c
#include <linux/io.h>

/* في debug code داخل الـ driver */
void my_dump_reset_regs(struct my_reset_priv *priv)
{
    u32 val;
    int i;

    for (i = 0; i < 4; i++) {
        val = readl(priv->base + i * 4);
        pr_info("reset reg[%d] @ offset 0x%02x = 0x%08x\n",
                i, i * 4, val);
    }
}
```

---

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**Logic Analyzer:**

```
Setup:
- اربط الـ probe على الـ reset line بين الـ controller والـ peripheral
- اضبط الـ trigger على falling edge (معظم الـ reset lines active-low)
- Sample rate: minimum 10x أسرع من أضيق pulse متوقع
  - مثال: لو الـ min reset pulse = 1ms → sample rate >= 10kHz
- ابحث عن:
  ┌─────────────────┐
  │   Normal (high) │    Reset Pulse    │  Back to normal
  ─┘                 └───────────────────┘
                    ←── pulse width ──→
                    (verify against datasheet min)
```

**Oscilloscope:**

```
Channel setup:
- AC coupling لو بتشوف noise على الـ reset line
- DC coupling للقراءة الدقيقة للـ voltage levels
- تحقق:
  - Voh (logic high) = VDD أو قريب منه
  - Vol (logic low) = 0V أو قريب منه
  - Rise/fall time مناسب للـ peripheral specs
  - مفيش glitches (spikes) أثناء الـ operation
```

---

#### 4. مشاكل الـ Hardware الشايعة وأنماط الـ Kernel Log

| المشكلة | نمط الـ kernel log | السبب | الحل |
|---|---|---|---|
| الـ peripheral مش بييجي من الـ reset | `timeout waiting for reset` | pulse width أقصر من الـ minimum المطلوب | زوّد `reset-duration-us` في DT |
| الـ reset line دايماً asserted | `device not responding after deassert` | مشكلة في الـ hardware أو الـ register غلط | افحص الـ active-low/high polarity |
| glitch على الـ reset line أثناء الـ boot | peripheral يعمل reset فجأة | power supply noise | أضف capacitor أو راجع الـ sequencing |
| الـ reset controller مش بيستجيب | `reset: timed out` | الـ clock للـ reset controller مش شغّال | تأكد إن الـ clock source enabled |
| shared reset بيتـassert من غير قصد | device stops working randomly | unbalanced assert/deassert calls | راجع الـ refcount بـ LOCKDEP |

---

#### 5. تـDebugging الـ Device Tree

```bash
# 1. تحقق إن الـ DT اتحمّل صح
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null > /tmp/current.dts
grep -A5 'reset' /tmp/current.dts

# 2. تحقق من الـ reset controller node
grep -A10 'reset-controller' /tmp/current.dts

# مثال على DT صح:
# reset_ctrl: reset-controller@1c00000 {
#     compatible = "allwinner,sun50i-a64-reset";
#     reg = <0x01c00000 0x1000>;
#     #reset-cells = <1>;
# };
#
# my_device: device@1c10000 {
#     resets = <&reset_ctrl 42>;
#     reset-names = "main";
# };

# 3. تحقق إن الـ compatible string في الـ DT بيطابق الـ driver
grep 'compatible' /sys/bus/platform/devices/<reset-ctrl>/of_node/

# 4. تحقق من الـ #reset-cells
cat /sys/firmware/devicetree/base/<reset-ctrl-node>/\#reset-cells | xxd
# لازم يكون 1 (أو أكتر حسب الـ controller)

# 5. تحقق إن الـ reg property صح
cat /sys/firmware/devicetree/base/<reset-ctrl-node>/reg | xxd
# compare مع الـ datasheet address

# 6. فعّل DT debugging في الـ boot
# أضف في kernel cmdline:
# of_devlink=hooks  أو  earlycon + initcall_debug
```

**مقارنة الـ DT الـ expected بالـ actual:**

```bash
# compile the intended DTS
dtc -I dts -O dtb my_board.dts -o /tmp/intended.dtb
# dump the running DT
cp /sys/firmware/fdt /tmp/running.dtb
# قارن
fdtdump /tmp/intended.dtb > /tmp/intended.dts
fdtdump /tmp/running.dtb  > /tmp/running.dts
diff /tmp/intended.dts /tmp/running.dts | grep -A3 'reset'
```

---

### Practical Commands

---

#### جميع الأوامر جاهزة للنسخ

**سيناريو 1: الـ driver بيفشل في الـ probe بسبب reset**

```bash
# 1. شوف الـ error
dmesg | grep -i "reset\|EPROBE_DEFER\|probe" | tail -20

# 2. تحقق إن الـ reset controller driver اتلود
lsmod | grep reset
ls /sys/bus/platform/drivers/ | grep reset

# 3. تحقق من الـ device tree
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | \
    grep -B2 -A8 'reset-controller'

# 4. فعّل dynamic debug وحاول reload
echo 'file drivers/reset/core.c +p' > /sys/kernel/debug/dynamic_debug/control
modprobe -r <your_driver>
modprobe <your_driver>
dmesg | tail -30
```

**سيناريو 2: التحقق من الـ reset sequence**

```bash
# فعّل tracing
echo nop > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/events/reset/enable
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل الـ operation اللي بتحتاج تتبّعها
echo 1 > /sys/bus/platform/devices/<device>/driver_override
echo <device> > /sys/bus/platform/drivers/<driver>/bind

# اقرأ النتايج
cat /sys/kernel/tracing/trace | grep -E 'reset_control'

# مثال على output مفسَّر:
# my_driver-456  [002]  reset_control_assert: id=3
#   → الـ driver بيعمل assert لـ reset line رقم 3
# my_driver-456  [002]  reset_control_deassert: id=3
#   → بعد كده بيعمل deassert — الـ sequence صح

echo 0 > /sys/kernel/tracing/tracing_on
```

**سيناريو 3: كشف unbalanced reset calls**

```bash
# فعّل LOCKDEP + PROVE_LOCKING في الـ config (يستلزم recompile)
# بدون recompile، راجع الـ refcount يدوياً عبر:
echo 'file drivers/reset/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# +p = print, +f = function, +l = line, +m = module, +t = thread

dmesg -w &
modprobe <your_driver>
# ابحث عن:
# reset_control_deassert: deassert_count going from 0 to 1
# reset_control_assert:   deassert_count going from 1 to 0
# اي assert بدون deassert مقابل = bug
```

**سيناريو 4: التحقق من حالة الـ reset line على الـ hardware**

```bash
# اعرف عنوان الـ reset controller register من DT
ADDR=$(cat /sys/firmware/devicetree/base/<ctrl-node>/reg | \
       python3 -c "import sys; d=sys.stdin.buffer.read(); print(hex(int.from_bytes(d[:4],'big')))")
echo "Reset controller base: $ADDR"

# اقرأ الـ register
devmem2 $ADDR w
# مثال output:
# Value at address 0x01C202C0: 0xFFFFF3FF
# Bit 10 و 11 = 0 → reset lines 10,11 assertted
# باقي الـ bits = 1 → deasserted
```

**سيناريو 5: reset controller driver لا يتعرّف عليه**

```bash
# تحقق من الـ compatible string في kernel
grep -r '"your,compatible-string"' /lib/modules/$(uname -r)/source/drivers/reset/
# أو في الـ source tree
grep -r 'your,compatible' /workspace/external/linux/drivers/reset/

# تحقق من الـ MODULE_DEVICE_TABLE
modinfo <reset_driver>.ko | grep alias

# تحقق من uevent
cat /sys/bus/platform/devices/<device>/uevent
# لازم DRIVER= يظهر لو اتبند صح
```

**سيناريو 6: مراقبة الـ reset activity في الـ production (overhead منخفض)**

```bash
# استخدم perf بدل ftrace
perf probe -m drivers/reset/core -a reset_control_assert
perf probe -m drivers/reset/core -a reset_control_deassert
perf record -e probe:reset_control_assert -e probe:reset_control_deassert \
    -g -- sleep 10
perf report
# ظهّر الـ call stack لكل reset operation
```

**سيناريو 7: debugging الـ Device Tree mismatch**

```bash
# اعرض كل الـ devices اللي عندها resets property في الـ DT
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | \
    awk '/resets =/{found=1} found{print; if(/;/) {found=0}}'

# تحقق إن كل consumer device اتلقى reset controller بتاعه
for dev in /sys/bus/platform/devices/*/; do
    if [ -f "$dev/of_node/resets" ]; then
        echo "Device: $(basename $dev)"
        echo "  Driver: $(readlink $dev/driver 2>/dev/null | xargs basename)"
        xxd "$dev/of_node/resets"
    fi
done
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — الـ UART مش بيشتغل بعد الـ suspend

#### العنوان
الـ UART controller بيفضل في حالة reset بعد ما الـ system يرجع من الـ suspend على gateway صناعي

#### السياق
بتشتغل على industrial IoT gateway بيستخدم **RK3562 SoC**، الـ gateway ده بيتكلم مع sensors بالـ UART. المنتج بيدخل في الـ suspend بعد فترة خمول، وبعد ما يصحى، الـ UART بيبقى ميت تمامًا — مفيش data بتعدّي خالص.

#### المشكلة
الـ UART driver بيعمل `reset_control_assert()` في الـ suspend handler بس مش بيعمل `reset_control_deassert()` صح في الـ resume. الـ peripheral فاضل في حالة reset.

#### التحليل
الـ API documentation بتقول صريح:

> *"Exclusive resets guarantee direct control — an assert causes the reset line to be asserted immediately"*

الـ driver بياخد الـ reset control بـ `devm_reset_control_get_exclusive()`. في الـ suspend:

```c
/* suspend handler — يعمل assert للـ UART reset */
reset_control_assert(uart_rst);
```

في الـ resume، الـ developer نسي إنه لازم يعمل deassert قبل ما يحاول يكتب في الـ UART registers:

```c
/* resume handler — مكتوبة غلط */
clk_enable(uart_clk);
uart_configure(dev);  /* بيكتب في registers والـ peripheral لسه في reset! */
```

الـ `reset_control_status()` لو الـ controller بيدعمها كانت ممكن تكشف الموضوع فورًا.

#### الحل

**في الـ DT** — `rk3562.dtsi`:
```dts
uart2: serial@feb50000 {
    compatible = "rockchip,rk3562-uart";
    resets = <&cru SRST_UART2>;
    reset-names = "uart";
};
```

**في الـ driver**:
```c
static int rk_uart_resume(struct device *dev)
{
    struct rk_uart_port *port = dev_get_drvdata(dev);

    clk_enable(port->clk);

    /* الأهم: deassert الـ reset الأول قبل أي حاجة تانية */
    reset_control_deassert(port->rst);

    uart_configure(port);  /* دلوقتي الـ peripheral جاهز */
    return 0;
}
```

**للـ debug السريع**:
```bash
# شوف حالة الـ reset lines عبر debugfs
cat /sys/kernel/debug/reset/reset_controls

# تأكد إن الـ driver اتسجل صح
ls /sys/bus/platform/drivers/rockchip-uart/
```

#### الدرس المستفاد
في الـ exclusive reset model، أي `assert` بيأثر فورًا وبيفضل معمول حتى تعمل `deassert` صريح. الـ suspend/resume sequence لازم تكون متطابقة تمامًا: assert في suspend ↔ deassert في resume، وده لازم يتعمل **قبل** أي access للـ registers.

---

### السيناريو 2: Android TV Box على Allwinner H616 — الـ HDMI مش بيطلع صورة بعد الـ boot

#### العنوان
الـ HDMI output فاشل على TV box لأن الـ shared reset control متعمله assert من driver تاني

#### السياق
بتعمل bring-up لـ Android TV box بيستخدم **Allwinner H616**. الـ HDMI مش بيطلع صورة خالص من أول الـ boot، بالرغم من إن الـ display pipeline كلها شغالة ظاهريًا.

#### المشكلة
الـ H616 فيه reset line واحدة مشتركة بين الـ HDMI controller والـ DE (Display Engine). التراكيب بتستخدم `devm_reset_control_get_shared()` بس في driver واحد بس — الـ HDMI driver — وفي نفس الوقت الـ DE driver بياخده exclusive وبيعمله assert في الـ probe بتاعته.

#### التحليل
الـ documentation بتوضح:

> *"Shared resets: only the first deassert increments the count to one... the last assert which decrements back to zero has physical effect"*

الـ DE driver:
```c
/* de_driver.c — غلط: بياخد shared reset كـ exclusive */
rst = devm_reset_control_get_exclusive(dev, "de-hdmi-rst");
reset_control_assert(rst);  /* ده بيعمل assert فعلي فوري */
```

الـ HDMI driver:
```c
/* hdmi_driver.c */
rst = devm_reset_control_get_shared(dev, "de-hdmi-rst");
reset_control_deassert(rst);  /* الـ refcount بيبقى 1 */
```

لكن لأن الـ DE driver عمل `exclusive assert`، الـ physical line فاضلت asserted بغض النظر عن الـ shared deassert من الـ HDMI driver.

#### الحل

**تصليح الـ DT**:
```dts
/* sun50i-h616.dtsi */
de2: display-engine@1000000 {
    resets = <&ccu RST_BUS_DE>,
             <&ccu RST_BUS_HDMI>;   /* reset منفصل للـ DE */
    reset-names = "de", "hdmi-shared";
};

hdmi: hdmi@6000000 {
    resets = <&ccu RST_BUS_HDMI>;
    reset-names = "hdmi";
};
```

**تصليح الـ driver**:
```c
/* de_driver.c — الإصلاح */
/* بدل exclusive، استخدم shared لو الـ reset line مشتركة */
rst = devm_reset_control_get_shared(dev, "hdmi-shared");
reset_control_deassert(rst);  /* مش بيعمل assert خالص */
```

**debug commands**:
```bash
# فحص الـ reset controller entries في الـ DT
dtc -I fs /sys/firmware/devicetree/base | grep -A5 "resets"

# تتبع الـ reset calls في الـ kernel log
dmesg | grep -i "reset"
```

#### الدرس المستفاد
الـ shared vs exclusive reset مش مجرد API choice — ده contract. لو reset line مشتركة بين peripherals، **كل** الـ consumers لازم يستخدموا `get_shared()`. واحد بس يستخدم `get_exclusive()` وكده يقدر يكسر الـ pipeline كلها.

---

### السيناريو 3: Automotive ECU على i.MX8 — الـ SPI flash مش بيتعرف بعد hardware reset

#### العنوان
الـ SPI NOR flash مش بيتعرف على i.MX8-based ECU بعد power cycle سريع

#### السياق
بتبني **automotive ECU** بيستخدم **i.MX8QM SoC** متصل بـ SPI NOR flash لتخزين الـ firmware. في الظروف العادية كل حاجة شغالة. بس لما الـ ECU بيتعمله power cycle سريع جدًا (أقل من 100ms)، الـ SPI flash مش بيتعرف وبتاخد error في الـ boot.

#### المشكلة
الـ SPI controller driver بيستخدم `reset_control_reset()` لعمل pulse على الـ reset line، بس الـ flash IC محتاج وقت أطول بعد الـ reset pulse عشان يرجع تاني. الـ driver مش بيستنى خالص.

#### التحليل
الـ documentation بتشرح `reset_control_reset()`:

> *"trigger a reset pulse on a self-deasserting reset control"*

الـ pulse بيتعمل وبيخلص بسرعة، بس الـ flash IC (مثلًا W25Q128) محتاج `tRST = 30µs` minimum بعد الـ reset pulse قبل ما يبدأ يقبل SPI commands.

```c
/* imx8_spi_driver.c — الكود الغلط */
static int imx8_spi_probe(struct platform_device *pdev)
{
    rst = devm_reset_control_get_exclusive(dev, "spi-rst");

    /* بيعمل pulse بس مش بيستنى */
    reset_control_reset(rst);

    /* مباشرة بيحاول يتكلم مع الـ flash — ده بيفشل */
    spi_flash_identify(dev);
}
```

#### الحل

**الإصلاح في الـ driver** — نستخدم assert/deassert بدل pulse ونضيف delay:

```c
/* imx8_spi_driver.c — الكود الصح */
static int imx8_spi_probe(struct platform_device *pdev)
{
    rst = devm_reset_control_get_exclusive(dev, "spi-rst");

    /* manual assert/deassert مع delay كافي */
    reset_control_assert(rst);
    udelay(10);               /* hold time للـ reset */
    reset_control_deassert(rst);
    udelay(50);               /* tRST: وقت الـ flash عشان يصحى */

    /* دلوقتي الـ flash جاهز */
    spi_flash_identify(dev);
}
```

**أو عن طريق الـ DT** لو الـ reset controller بيدعم reset duration:
```dts
/* imx8qm.dtsi */
spi0: spi@5a000000 {
    compatible = "fsl,imx8qm-lpspi";
    resets = <&rst IMX8QM_SPI0_RST>;
    reset-names = "spi";
    reset-duration-us = <50>;  /* لو الـ controller بيدعم */
};
```

**debug**:
```bash
# فحص الـ SPI flash detection
dmesg | grep -i "spi\|flash\|w25q"

# تحقق من الـ reset timing عبر oscilloscope على الـ RESET# pin
# أو استخدم logic analyzer على الـ SPI lines
```

#### الدرس المستفاد
الـ `reset_control_reset()` بيعمل pulse بس مش بيضيف delay. الـ external ICs زي الـ flash chips محتاجة وقت recovery بعد الـ reset. دايمًا راجع الـ datasheet للـ IC عشان تعرف `tRST` وتضيف `udelay()` مناسب.

---

### السيناريو 4: IoT Sensor Board على STM32MP1 — الـ I2C bus فاشل بشكل عشوائي

#### العنوان
الـ I2C bus بيـhang بشكل عشوائي على STM32MP1-based IoT sensor board

#### السياق
بتبني **IoT sensor board** بيستخدم **STM32MP157** متصل بعدة sensors بالـ I2C. بعد تشغيل طويل (ساعات)، الـ I2C bus بيـhang وبتاخد timeout errors، وما عندكش طريقة تعيد الـ bus للحياة غير إنك تعمل full system reboot.

#### المشكلة
الـ STM32MP1 I2C controller بيدعم soft reset عبر الـ reset controller، بس الـ driver مش بيستخدم الـ reset API صح لعمل recovery لما الـ bus يـhang. كمان الـ driver بياخد الـ reset كـ optional بس مش بيتعامل معاه صح لما بيبقى NULL.

#### التحليل
الـ documentation بتقول:

> *"Passing a NULL pointer to the reset_control functions causes them to return quietly without an error"*

الـ driver الأصلي:
```c
/* stm32_i2c.c */
static int stm32_i2c_probe(struct platform_device *pdev)
{
    /* بياخد الـ reset كـ optional */
    priv->rst = devm_reset_control_get_optional_exclusive(dev, "i2c");
    /* لو NULL، كل calls بعدين هترجع بهدوء من غير effect */
}

static int stm32_i2c_recover_bus(struct i2c_adapter *adap)
{
    /* محاولة recovery بدون reset — بتفشل */
    stm32_i2c_clrset_bits(priv, STM32MP_I2C_CR1, 0, I2C_CR1_PE);
    udelay(2);
    stm32_i2c_clrset_bits(priv, STM32MP_I2C_CR1, I2C_CR1_PE, 0);
    return 0;  /* بيرجع success حتى لو الـ bus لسه hang */
}
```

الـ proper recovery المحتاجة:
```c
/* الـ reset controller بيديك فرصة تعمل hard reset للـ I2C controller */
static int stm32_i2c_recover_bus(struct i2c_adapter *adap)
{
    struct stm32_i2c_dev *priv = i2c_get_adapdata(adap);

    dev_warn(priv->dev, "I2C bus hung, attempting recovery via reset\n");

    /* assert → deassert = hard reset للـ I2C controller */
    reset_control_assert(priv->rst);
    udelay(2);
    reset_control_deassert(priv->rst);

    /* إعادة initialize الـ controller */
    return stm32_i2c_hw_config(priv);
}
```

#### الحل

**تصليح الـ DT**:
```dts
/* stm32mp15xx.dtsi */
i2c1: i2c@40012000 {
    compatible = "st,stm32mp15-i2c";
    resets = <&rcc I2C1_R>;
    reset-names = "i2c";  /* لازم موجود عشان الـ recovery يشتغل */
};
```

**تصليح الـ driver**:
```c
static int stm32_i2c_probe(struct platform_device *pdev)
{
    /* نستخدم exclusive مش optional لأننا محتاجينه للـ recovery */
    priv->rst = devm_reset_control_get_exclusive(dev, "i2c");
    if (IS_ERR(priv->rst)) {
        dev_err(dev, "failed to get reset: %ld\n", PTR_ERR(priv->rst));
        return PTR_ERR(priv->rst);
    }
}
```

**debug commands**:
```bash
# راقب الـ I2C errors في real-time
watch -n1 'cat /sys/bus/i2c/devices/i2c-1/statistics/*'

# فحص الـ reset controller في debugfs
cat /sys/kernel/debug/reset/reset_controls | grep i2c

# تشغيل الـ recovery يدويًا لو الـ driver بيدعمه
echo "recovery" > /sys/bus/i2c/devices/i2c-1/recovery
```

#### الدرس المستفاد
الـ `get_optional_*` مناسبة لـ peripherals اللي فعلًا ممكن تشتغل بدون reset. لو الـ reset محتاج للـ fault recovery، استخدم `get_exclusive()` وتعامل مع الـ error صح. الـ silent NULL handling مفيدة بس ممكن تخبي مشاكل في الـ initialization.

---

### السيناريو 5: Custom Board Bring-up على AM62x — الـ USB مش بيشتغل والـ reset ما اتسجلش

#### العنوان
الـ USB controller مش بيشتغل خالص على custom AM62x board لأن الـ reset controller driver ما اتسجلش قبل الـ USB driver

#### السياق
بتعمل **custom board bring-up** بيستخدم **TI AM625 SoC** (AM62x family). الـ board عندها USB Type-C port محتاج للـ product. بعد كتابة الـ DT وتشغيل الـ kernel، الـ USB driver بيـprobe بس الـ USB مش بيشتغل وبتلاقي error في الـ kernel log.

#### المشكلة
الـ USB driver بيطلب reset control بعد ما الـ reset controller driver يسجل نفسه. بس بسبب الـ probe order، الـ USB driver بيـprobe **قبل** الـ reset controller driver. الـ `devm_reset_control_get_exclusive()` بترجع `-EPROBE_DEFER` بس الـ driver مش بيتعامل معاه صح.

#### التحليل
الـ documentation بتقول:

> *"Drivers fill a struct reset_controller_dev and register it with reset_controller_register() in their probe function"*

الـ reset controller لازم يـprobe الأول. الـ kernel infrastructure بيرجع `-EPROBE_DEFER` لما الـ reset controller لسه مش موجود، وده معناه "حاول تاني بعدين".

الـ USB driver المكسور:
```c
/* am62x_usb.c — الكود الغلط */
static int am62x_usb_probe(struct platform_device *pdev)
{
    priv->rst = devm_reset_control_get_exclusive(dev, "usb-rst");
    if (IS_ERR(priv->rst)) {
        /* بيعامله كـ fatal error بدل ما يتعامل مع EPROBE_DEFER */
        dev_err(dev, "failed to get reset\n");
        return -EINVAL;  /* غلط! المفروض يرجع PTR_ERR() */
    }
}
```

الـ kernel log:
```
[    2.341] am62x-usb fe400000.usb: failed to get reset
[    2.342] am62x-usb: probe of fe400000.usb failed with error -22
```

#### الحل

**تصليح الـ driver عشان يتعامل مع `-EPROBE_DEFER`**:
```c
/* am62x_usb.c — الكود الصح */
static int am62x_usb_probe(struct platform_device *pdev)
{
    priv->rst = devm_reset_control_get_exclusive(dev, "usb-rst");
    if (IS_ERR(priv->rst)) {
        /* رجّع الـ error كما هو — الـ kernel هيعمل retry تلقائي */
        return dev_err_probe(dev, PTR_ERR(priv->rst),
                             "failed to get USB reset\n");
    }

    reset_control_deassert(priv->rst);
    /* باقي الـ initialization */
}
```

**تصليح الـ DT عشان تضمن الـ probe order**:
```dts
/* k3-am625.dtsi */
psc: power-sleep-controller@400000 {
    compatible = "ti,am625-psc";
    /* الـ reset controller لازم يتعرف كـ dependency */
    #reset-cells = <2>;
};

usbss0: usb@f900000 {
    compatible = "ti,am62-usb";
    resets = <&psc AM625_PSC_USB0>;
    reset-names = "usb-rst";
    /* الـ EPROBE_DEFER هيتعامل معاه تلقائي */
};
```

**التحقق من الـ probe order**:
```bash
# شوف ترتيب الـ probe في الـ kernel log
dmesg | grep -E "probe|psc|usb" | head -30

# تأكد إن الـ reset controller اتسجل
ls /sys/bus/platform/drivers/ti-psc/

# فحص الـ deferred probes
cat /sys/kernel/debug/devices_deferred

# لو الـ USB اشتغل بعد الـ retry
dmesg | grep "usb.*registered\|xhci"
```

**تحقق من الـ reset controller registration**:
```c
/* ti_psc_driver.c — الـ reset controller driver */
static int ti_psc_probe(struct platform_device *pdev)
{
    struct reset_controller_dev *rcdev = &psc->rcdev;

    rcdev->ops = &ti_psc_reset_ops;
    rcdev->owner = THIS_MODULE;
    rcdev->nr_resets = AM625_PSC_MAX;
    rcdev->of_node = pdev->dev.of_node;

    /* التسجيل ده اللي بيخلي الـ USB driver يـprobe بعدين */
    return devm_reset_controller_register(&pdev->dev, rcdev);
}
```

#### الدرس المستفاد
الـ `-EPROBE_DEFER` مش error — ده mechanism. أي driver بيطلب reset control لازم يرجع `PTR_ERR()` كما هو (أو يستخدم `dev_err_probe()`)، وده بيخلي الـ kernel يعيد الـ probe تلقائي لما الـ reset controller يـregister نفسه. ما تعملش `return -EINVAL` hardcoded وإلا هتقفل الـ deferred probe mechanism.
## Phase 7: مصادر ومراجع

### مصادر رسمية — التوثيق الرسمي للـ Kernel

| المصدر | الرابط |
|--------|--------|
| **Reset controller API** — التوثيق الرسمي الكامل | [docs.kernel.org/driver-api/reset.html](https://docs.kernel.org/driver-api/reset.html) |
| **reset.rst** — الملف الأصلي في شجرة الـ kernel | [kernel.org/doc/Documentation/driver-api/reset.rst](https://www.kernel.org/doc/Documentation/driver-api/reset.rst) |
| **reset-controller.h** — تعريف `reset_controller_dev` و `reset_control_ops` | [`include/linux/reset-controller.h`](https://github.com/torvalds/linux/blob/master/include/linux/reset-controller.h) |
| **reset.h** — واجهة الـ consumer API | [`include/linux/reset.h`](https://github.com/torvalds/linux/blob/master/include/linux/reset.h) |
| **core.c** — تنفيذ الـ subsystem الأساسي | [`drivers/reset/core.c`](https://github.com/torvalds/linux/blob/master/drivers/reset/core.c) |
| **Devres** — managed device resources (استُخدم مع `devm_*`) | [static.lwn.net — Devres docs](https://static.lwn.net/kerneldoc/driver-api/driver-model/devres.html) |

---

### مقالات LWN.net

الـ **LWN.net** هو المرجع الأهم لمتابعة تطور الـ kernel — دي أبرز المقالات المتعلقة بالـ reset controller subsystem:

#### 1. إضافة GPIO Reset Driver العام
> **"reset: Add generic GPIO reset driver"**

- **الرابط:** [lwn.net/Articles/585145/](https://lwn.net/Articles/585145/)
- **الأهمية:** أُضيف دعم الـ GPIO كـ reset controller — مثال حقيقي لتنفيذ driver بسيط يستخدم الـ framework.

#### 2. إضافة Sparx5 Switch Reset Driver
> **"Adding the Sparx5 Switch Reset Driver"**

- **الرابط:** [lwn.net/Articles/852989/](https://lwn.net/Articles/852989/)
- **الأهمية:** مثال عملي لكتابة reset controller driver لـ SoC حقيقي (Microchip Sparx5) — الـ driver بيحمي busses تانية من الـ reset عن طريق bitmask.

#### 3. إضافة دعم Reset Controller لـ EN7523 SoC
> **"clk: en7523: reset-controller support for EN7523 SoC"**

- **الرابط:** [lwn.net/Articles/1039543/](https://lwn.net/Articles/1039543/)
- **الأهمية:** بيوضح إزاي الـ clock و reset controllers ممكن يتشاركوا كود في نفس الـ driver.

#### 4. إضافة Mobileye EyeQ System Controller
> **"Add Mobileye EyeQ system controller support (clk, reset, pinctrl)"**

- **الرابط:** [lwn.net/Articles/979170/](https://lwn.net/Articles/979170/)
- **الأهمية:** مثال متقدم على تكامل الـ reset مع الـ clock framework و pinctrl في نفس الـ system controller.

#### 5. إضافة STM32MP13 RCC Driver
> **"Introduction of STM32MP13 RCC Driver (Reset Clock Controller)"**

- **الرابط:** [lwn.net/Articles/888111/](https://lwn.net/Articles/888111/)
- **الأهمية:** STM32 بيجمع بين الـ Reset و Clock في وحدة واحدة اسمها RCC — نموذج شائع جداً في الـ embedded SoCs.

#### 6. إضافة Arria10 System Manager Reset Controller
> **"Add Arria10 System Manager Reset Controller"**

- **الرابط:** [lwn.net/Articles/714705/](https://lwn.net/Articles/714705/)
- **الأهمية:** بيوضح إزاي الـ System Manager في Intel/Altera Arria10 بيتحكم في resets متعددة.

---

### Patches ومناقشات الـ Mailing List

دي أهم الـ patches التاريخية اللي شكّلت الـ subsystem:

#### Patch الأصلي — إضافة Reset Controller API
> **"[v4,2/8] reset: Add reset controller API"** — Philipp Zabel

- **الرابط:** [patchwork.kernel.org — v4,2/8](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/)
- **الأهمية:** الـ patch الأصلي اللي أدخل الـ subsystem للـ kernel — بيتضمن الـ DT binding والـ API الأساسي.

#### GIT PULL الأولى للـ Reset Controller API
> **"[GIT PULL v2] Reset controller API"** — Philipp Zabel

- **الرابط:** [lore.kernel.org — GIT PULL v2](https://lore.kernel.org/linux-arm-kernel/1365683829.4388.52.camel@pizza.hi.pengutronix.de/)
- **الأهمية:** أول مرة بيتعمل pull request رسمي للـ subsystem من الـ maintainer.

#### Patch إضافة GPIO Support
> **"[v4,2/2] reset: Add GPIO support to reset controller framework"** — Philipp Zabel

- **الرابط:** [patchwork.kernel.org — GPIO support](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1397463709-19405-2-git-send-email-p.zabel@pengutronix.de/)
- **الأهمية:** بيضيف `reset-gpio` driver عشان أي GPIO pin يقدر يشتغل كـ reset line.

#### Patch التحويل لـ exclusive/shared API الصريح
> **"reset: Ensure drivers are explicit when requesting reset lines"**

- **الرابط:** [lore.kernel.org — explicit exclusive resets](https://lore.kernel.org/patchwork/patch/811829/)
- **الأهمية:** الـ patch series اللي حوّل كل الـ drivers القديمة من الـ deprecated `reset_control_get()` للـ `get_exclusive()` و `get_shared()` الصريحة — تقريباً 100+ driver اتغيروا.

#### GIT PULL Reset Controller Fixes — v6.2
> **"[GIT PULL] Reset controller fixes for v6.2"** — Philipp Zabel

- **الرابط:** [lore.kernel.org — v6.2 fixes](https://lore.kernel.org/all/20230104095855.3809733-1-p.zabel@pengutronix.de/)
- **الأهمية:** بيوضح إزاي الـ subsystem لسه بيتطور ويتصحح.

---

### مصادر الـ Source Code المباشرة

```
drivers/reset/          ← كل الـ reset controller drivers
├── core.c              ← قلب الـ subsystem
├── reset-simple.c      ← أبسط reset controller driver
├── gpio-reset.c        ← GPIO-based reset driver
├── Kconfig
└── Makefile

include/linux/
├── reset.h             ← consumer API
└── reset-controller.h  ← provider/driver API

Documentation/driver-api/reset.rst    ← التوثيق الرسمي
Documentation/devicetree/bindings/reset/  ← DT bindings
```

---

### كتب مرجعية

#### Linux Device Drivers (LDD3) — Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
الـ **LDD3** هو المرجع الكلاسيكي لكتابة الـ kernel drivers — الفصول الأهم للـ reset controller:

| الفصل | الموضوع |
|-------|---------|
| Chapter 3 | Char Drivers — بيشرح الـ device model الأساسي |
| Chapter 14 | The Linux Device Model — الـ `kobject`، الـ `bus`، الـ `device` |
| Chapter 1 | Introduction — بيشرح فلسفة الـ kernel subsystems |

- **الكتاب مجاناً:** [lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
الـ **Robert Love** بيشرح الـ kernel internals بشكل عملي — أهم الفصول للـ reset:

| الفصل | الموضوع |
|-------|---------|
| Chapter 17 | Devices and Modules |
| Chapter 18 | Debugging |
| Chapter 11 | Timers and Time Management — مهم لفهم timing في الـ reset pulses |

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)
الأهم للعمل على الـ embedded SoCs:

| الفصل | الموضوع |
|-------|---------|
| Chapter 16 | Kernel Initialization |
| Chapter 11 | BusyBox |
| Chapter 5 | Kernel Initialization — بيشرح فلسفة الـ peripheral initialization |

---

### بحث في الـ Kernel Source

للبحث في الـ kernel source على الـ Bootlin Elixir cross-referencer:

- **Reset Controller Core:** [elixir.bootlin.com — reset/core.c](https://elixir.bootlin.com/linux/latest/source/drivers/reset/core.c)
- **Reset Consumer Header:** [elixir.bootlin.com — reset.h](https://elixir.bootlin.com/linux/latest/source/include/linux/reset.h)
- **Reset Controller Header:** [elixir.bootlin.com — reset-controller.h](https://elixir.bootlin.com/linux/latest/source/include/linux/reset-controller.h)
- **DT Bindings للـ reset:** [elixir.bootlin.com — DT reset bindings](https://elixir.bootlin.com/linux/latest/source/Documentation/devicetree/bindings/reset)

---

### مصادر إضافية مفيدة

#### STM32 Reset Overview — ST Microelectronics Wiki
- **الرابط:** [wiki.st.com/stm32mpu/wiki/Reset_overview](https://wiki.st.com/stm32mpu/wiki/Reset_overview)
- **الأهمية:** بيشرح كيف بتُدار الـ resets في STM32 SoCs — مثال عملي ممتاز.

#### CONFIG_RESET_SIMPLE — Linux Kernel Driver Database
- **الرابط:** [cateee.net — RESET_SIMPLE](https://cateee.net/lkddb/web-lkddb/RESET_SIMPLE.html)
- **الأهمية:** قائمة بكل الـ SoCs اللي بتستخدم الـ simple reset controller driver.

#### Bootlin Kernel Training Materials
- **الرابط:** [bootlin.com/training/kernel/](https://bootlin.com/training/kernel/)
- **الأهمية:** المواد التدريبية مجانية — بتغطي كتابة الـ device drivers بالكامل مع أمثلة عملية.

#### Mail Archive — linux-kernel مناقشة optional resets
- **الرابط:** [mail-archive.com — Why reset_control_get_optional?](https://groups.google.com/g/fa.linux.kernel/c/9jG5RFPcEZU)
- **الأهمية:** نقاش مهم عن فلسفة الـ optional resets وامتى بنحتاجها.

---

### كلمات البحث الموصى بها

لو عايز تبحث عن معلومات أكتر، استخدم الـ search terms دي:

```
# على Google / DuckDuckGo
linux kernel reset controller subsystem
philipp zabel reset controller pengutronix
devm_reset_control_get_exclusive kernel
reset_control_ops linux driver implementation
linux reset controller device tree binding
shared exclusive reset linux kernel

# على lore.kernel.org
reset: core
reset controller maintainer
reset: add

# على GitHub / Elixir
drivers/reset/core.c
include/linux/reset-controller.h
reset_controller_dev
reset_control_ops

# على LWN.net
reset controller
reset-gpio driver
SoC reset controller
```

---

### ملخص المراجع الأساسية

| الأولوية | المرجع | الاستخدام |
|----------|--------|-----------|
| **أولى** | [docs.kernel.org/driver-api/reset.html](https://docs.kernel.org/driver-api/reset.html) | API reference رسمي |
| **أولى** | [elixir.bootlin.com — reset/core.c](https://elixir.bootlin.com/linux/latest/source/drivers/reset/core.c) | قراءة الـ source |
| **ثانية** | [lwn.net/Articles/852989/](https://lwn.net/Articles/852989/) | مثال reset driver حقيقي |
| **ثانية** | [lwn.net/Articles/585145/](https://lwn.net/Articles/585145/) | GPIO reset driver |
| **ثالثة** | [patchwork.kernel.org — original API](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/) | تاريخ الـ subsystem |
| **ثالثة** | [lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/) | كتاب LDD3 مجاناً |
| **إضافي** | [bootlin.com/training/kernel/](https://bootlin.com/training/kernel/) | تدريب عملي |
## Phase 8: Writing simple module

### الفكرة: Hook الـ `reset_control_assert`

**الـ function اللي هنعمل عليها hook** هي `reset_control_assert` — اللي بتبعت signal الـ reset لـ peripheral. ده مكان مثالي للمراقبة لأن كل driver بيعمل reset لجهاز بيمر منها، وبالتالي نقدر نشوف مين بيعمل reset لإيه.

بنستخدم **kprobe** لأن `reset_control_assert` مش tracepoint جاهز، وهي exported وغير-inlined، فالـ kprobe بيقدر يضرب عليها بسهولة.

---

### الكود الكامل للـ module

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * reset_assert_probe.c
 * Hooks reset_control_assert() via kprobe to log every reset assertion event.
 */

/* --- Includes --- */
#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kprobes.h>     /* struct kprobe, register_kprobe     */
#include <linux/kernel.h>      /* pr_info                             */
#include <linux/reset.h>       /* struct reset_control layout hints   */

/*
 * نحتاج noinline + الـ struct الداخلي عشان نقدر نقرأ حاجات من الـ rstc.
 * الـ reset_control struct مش exposed في الـ headers العامة،
 * فبنعتمد على الـ pointer نفسه فقط كـ opaque handle.
 */

/* --- kprobe handler (pre: runs before the real function) --- */
/*
 * الـ pre_handler بيتشغل قبل ما reset_control_assert تتنفذ،
 * فبنقدر نقرأ argument الأول (rstc) من الـ registers قبل ما الـ stack يتغير.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * الـ argument الأول بيجي في:
     *   rdi  على x86_64
     *   r0   على ARM32
     *   x0   على ARM64
     * الـ macro regs_get_kernel_argument() بيجيب الـ Nth argument
     * بصورة portable عبر architectures.
     */
    unsigned long rstc_addr = regs_get_kernel_argument(regs, 0);

    /* نطبع عنوان الـ reset_control struct كـ pointer */
    pr_info("reset_probe: reset_control_assert() called, rstc=%px, cpu=%d, comm=%s\n",
            (void *)rstc_addr,
            smp_processor_id(),    /* رقم الـ CPU اللي شغال عليه */
            current->comm);        /* اسم الـ process اللي طلب الـ reset */

    /*
     * إرجاع 0 يعني "كمّل تنفيذ الـ function الأصلية كالعادة".
     * لو رجعنا 1 كنا هنتجاوزها — مش اللي عايزينه هنا.
     */
    return 0;
}

/* --- تعريف الـ kprobe struct --- */
/*
 * بنحدد اسم الـ symbol (الـ function) اللي عايزين نضرب عليه،
 * والـ kernel بيحوّل الاسم لعنوان تلقائيًا وقت الـ registration.
 */
static struct kprobe kp = {
    .symbol_name = "reset_control_assert",  /* الـ function اللي هنـhook */
    .pre_handler  = handler_pre,            /* callback قبل التنفيذ       */
};

/* --- module_init --- */
/*
 * بنسجّل الـ kprobe هنا؛ لو فشل (مثلاً الـ symbol مش موجود أو
 * CONFIG_KPROBES=n) بنرجع error ونوقف load الـ module.
 */
static int __init reset_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("reset_probe: register_kprobe failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("reset_probe: kprobe planted at %px (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

/* --- module_exit --- */
/*
 * لازم نشيل الـ kprobe في الـ exit عشان منسيبش breakpoint يشتغل
 * بعد ما الـ module اتنزل من الـ memory — ده هيعمل kernel panic.
 */
static void __exit reset_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("reset_probe: kprobe removed from %px\n", kp.addr);
}

module_init(reset_probe_init);
module_exit(reset_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe hook on reset_control_assert() to trace peripheral resets");
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | `module_init`, `module_exit`, `MODULE_LICENSE` — أساسيات أي module |
| `linux/kprobes.h` | الـ API بتاع kprobe: `struct kprobe`, `register_kprobe`, `unregister_kprobe` |
| `linux/kernel.h` | `pr_info`, `pr_err` للـ logging |
| `linux/reset.h` | عشان نعرف إن `reset_control_assert` موجودة وما هو توقيعها |

---

#### الـ `handler_pre` — قلب الـ module

الـ `pre_handler` بيتشغل مباشرةً قبل دخول `reset_control_assert`، فالـ CPU registers لسه شايلة الـ arguments الأصلية. بنستخدم `regs_get_kernel_argument(regs, 0)` عشان نجيب الـ `rstc` pointer بصورة portable على x86_64 وARM64 بدون ما نكتب كود architecture-specific.

**الـ `current->comm`** بيدينا اسم الـ task اللي طلب الـ reset (مثلاً `eth0`, `mmc0`, `i2c-1`)، وده مفيد جداً في debug — بتعرف مين اتعمله reset ومين اللي عمله.

---

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "reset_control_assert",
    .pre_handler  = handler_pre,
};
```

**الـ `.symbol_name`** بيخلي الـ kernel يحل العنوان من الـ kallsyms تلقائيًا وقت الـ registration — مش محتاج نكتب عنوان hardcoded. لو الـ function اتعملت inline في kernel build معين، الـ registration هترجع error وهنعرف فورًا.

---

#### الـ `module_init` و `module_exit`

- **`register_kprobe`** بتزرع software breakpoint (INT3 على x86 أو BRK على ARM) في بداية الـ function المستهدفة. لو فشلت بنرجع الـ error مباشرةً عشان المستخدم يعرف المشكلة.
- **`unregister_kprobe`** في الـ exit واجبة وليست اختيارية — لو نسيناها والـ module اتنزل، الـ breakpoint handler بقى يشاور على كود اتمسح من الـ memory، والنتيجة `kernel panic`.

---

### طريقة التجربة

```bash
# بناء الـ module (Makefile بسيط في نفس الـ directory)
make -C /lib/modules/$(uname -r)/build M=$PWD modules

# تحميله
sudo insmod reset_assert_probe.ko

# مراقبة الـ log
sudo dmesg -w | grep reset_probe

# مثال على output عند reboot لـ USB controller:
# reset_probe: reset_control_assert() called, rstc=ffff888003a1c080, cpu=2, comm=kworker/2:1

# إزالة الـ module
sudo rmmod reset_assert_probe
```

---

### ملاحظة على الـ portability

لو الـ kernel compiled بـ `CONFIG_RESET_CONTROLLER=n`، الـ `reset_control_assert` مش هتبقى موجودة في الـ symbol table وبالتالي `register_kprobe` هترجع `-ENOENT`. الـ module بيتعامل مع ده gracefully عبر check الـ return value.
