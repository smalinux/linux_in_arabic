## Phase 1: الصورة الكبيرة ببساطة

### ما هو الـ Reset Controller Framework؟

تخيل إنك عندك لوحة إلكترونية فيها عشرات الـ chips — USB controller، GPU، network card، display engine. كل chip دي لازم في بعض الأوقات تتعمل لها **reset** — يعني تقفل وتفتح من أول وجديد عشان تمسح حالتها القديمة وتبدأ نظيف.

الـ **reset** في الـ hardware بيتم عن طريق **reset line** — سلك حرفياً في الـ SoC بيتحكم في chip. لما تشيل السلك ده (assert) الـ chip بتتجمد، ولما تعيده (deassert) بتشتغل.

المشكلة: كل SoC بيعمل الموضوع ده بطريقته الخاصة — Qualcomm غير NXP غير Raspberry Pi غير Allwinner. لو كل driver كتب الـ reset logic بتاعه بنفسه كان هيبقى chaos.

**الحل:** الـ Reset Controller Framework — طبقة abstraction في الـ kernel بتوفر API موحدة للـ drivers عشان يعملوا reset للـ hardware بدون ما يعرفوا أي حاجة عن الـ SoC اللي شغالين عليه.

---

### القصة الكاملة

```
  ┌─────────────────────────────────────────────────────┐
  │                   Driver (Consumer)                  │
  │  USB driver / GPU driver / Network driver / etc.     │
  │                                                      │
  │  reset_control_get_exclusive(dev, "usb-rst")         │
  │  reset_control_deassert(rstc)  ← أنا عايز أشغل     │
  └──────────────────────┬──────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────┐
  │           Reset Controller Framework (Core)          │
  │          drivers/reset/core.c                        │
  │                                                      │
  │  - بيحفظ قائمة بكل الـ reset controllers            │
  │  - بيعمل reference counting للـ shared resets       │
  │  - بيترجم الـ device tree phandles لـ reset lines   │
  └──────────────────────┬──────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────┐
  │         Reset Controller Driver (Provider)           │
  │  مثلاً: drivers/reset/reset-imx7.c (NXP i.MX7)     │
  │                                                      │
  │  .assert   → اكتب في register معين في SoC           │
  │  .deassert → امسح bit في register                   │
  │  .reset    → assert + delay + deassert              │
  └──────────────────────┬──────────────────────────────┘
                         │
                         ▼
              ┌──────────────────┐
              │  الـ Hardware     │
              │  (Reset Register) │
              └──────────────────┘
```

---

### الـ File اللي إحنا بندرسه: `include/linux/reset.h`

الـ file ده هو **واجهة الـ consumer** — يعني الـ API اللي أي driver يستخدمها عشان يعمل reset لأجزاء الـ hardware. مش الـ API اللي بيكتبها صاحب الـ reset controller نفسه (دي في `reset-controller.h`).

#### ليه موجود؟

بدون الـ file ده، كل driver كان هيحتاج:
1. يعرف عنوان الـ register في الـ SoC
2. يعرف أي bit فيه بيتحكم في الـ reset
3. يكتب الـ timing logic بنفسه

بالـ file ده، الـ driver بيقول بس: "أنا عايز reset line اسمها `usb-rst`" — والـ framework يروح يجيب الـ reset controller الصح من الـ device tree.

---

### المفاهيم الأساسية في الـ File

#### 1. الـ Exclusive vs Shared Reset

| النوع | المعنى | متى تستخدمه |
|-------|---------|-------------|
| **exclusive** | واحد بس يمسك الـ reset | الـ driver الوحيد اللي بيتحكم في الـ chip |
| **shared** | أكتر من driver بيشاركوا نفس الـ reset line | chip واحدة بتخدم أكتر من subsystem |

مثال واقعي للـ shared: الـ USB PHY في بعض الـ SoCs بتخدم USB2 وUSB3 في نفس الوقت — الاتنين بيحتاجوا يـdeassert الـ reset قبل ما أي حد فيهم يشتغل. الـ framework بيعمل **reference counting**: الـ reset مش هيتـassert غير لما الاتنين يقولوا "أنا خلاص مش محتاجه".

#### 2. الـ Assert / Deassert / Reset

```
assert   → chip في حالة reset (مجمدة، مش شغالة)
deassert → chip خرجت من الـ reset (بدأت تشتغل)
reset    → assert + wait + deassert (دفعة واحدة للـ self-deasserting hardware)
```

#### 3. الـ Optional Reset

بعض الـ hardware مش لازم يعمل reset — أو الـ device tree مش محدد فيه reset line. الـ `optional` variants بترجع `NULL` بدل error — والـ driver بيعمل check وبيكمل بدون reset.

#### 4. الـ `devm_` prefix

ده الـ **resource-managed** variant — لما الـ driver بيتـunload أو يحصل error، الـ kernel بيعمل `reset_control_put()` تلقائياً. مفيش leak.

#### 5. الـ Bulk API

لو driver محتاج يعمل reset لـ 5 chips في نفس الوقت، مش هيتعامل معاهم واحد واحد. بيعمل array من `reset_control_bulk_data` وبيستدعي `reset_control_bulk_deassert()` مرة واحدة.

```c
/* مثال واقعي */
struct reset_control_bulk_data resets[] = {
    { .id = "axi" },
    { .id = "ahb" },
    { .id = "phy" },
};

devm_reset_control_bulk_get_exclusive(dev, ARRAY_SIZE(resets), resets);
reset_control_bulk_deassert(ARRAY_SIZE(resets), resets);
```

#### 6. الـ `CONFIG_RESET_CONTROLLER` Guard

الـ file فيه `#ifdef CONFIG_RESET_CONTROLLER` — لو الـ kernel اتـbuilt بدون الـ framework (embedded systems صغيرة مش محتاجاه)، الـ functions بتتحول لـ stubs بترجع 0 أو `NULL` بدل ما الـ build يفشل.

---

### الـ Flags System

```c
enum reset_control_flags {
    RESET_CONTROL_EXCLUSIVE,                    /* حجز حصري + acquired */
    RESET_CONTROL_EXCLUSIVE_DEASSERTED,         /* حصري + deassert فوراً */
    RESET_CONTROL_EXCLUSIVE_RELEASED,           /* حصري لكن released (lazily acquired) */
    RESET_CONTROL_SHARED,                       /* مشترك */
    RESET_CONTROL_SHARED_DEASSERTED,            /* مشترك + deassert */
    RESET_CONTROL_OPTIONAL_EXCLUSIVE,           /* اختياري + حصري */
    /* ... إلخ */
};
```

الـ `_RELEASED` variant مهم جداً: بيخليك تحجز الـ reset control بدون ما تـacquire حق الـ assert/deassert فوراً — محتاجه لما أكتر من driver بيتشاركوا نفس الـ reset لكن في أوقات مختلفة.

---

### الملفات المرتبطة اللي المفروض تعرفها

| الملف | الدور |
|-------|-------|
| `include/linux/reset.h` | **الـ file ده** — consumer API |
| `include/linux/reset-controller.h` | Provider API — اللي بيكتبها صاحب الـ reset controller driver |
| `drivers/reset/core.c` | قلب الـ framework — الـ implementation الحقيقية |
| `drivers/reset/reset-simple.c` | أبسط reset driver — بيتعامل مع single register |
| `drivers/reset/reset-gpio.c` | Reset عن طريق GPIO line |
| `drivers/reset/reset-imx7.c` | مثال على reset driver لـ NXP i.MX7 SoC |
| `drivers/reset/reset-qcom-aoss.c` | Qualcomm AOSS reset driver |
| `Documentation/driver-api/reset.rst` | الـ documentation الرسمية للـ framework |
| `Documentation/devicetree/bindings/reset/` | كيف بيتـعرّف الـ reset في الـ device tree |
| `include/dt-bindings/reset/` | Constants للـ device tree bindings |

---

### ملفات الـ Subsystem الكاملة

#### الـ Core
- `drivers/reset/core.c` — الـ implementation الحقيقية لكل الـ API

#### الـ Headers
- `include/linux/reset.h` — consumer API (الـ file ده)
- `include/linux/reset-controller.h` — provider API

#### الـ Hardware Drivers (أمثلة)
- `drivers/reset/reset-simple.c` — generic memory-mapped reset
- `drivers/reset/reset-gpio.c` — GPIO-based reset
- `drivers/reset/reset-scmi.c` — SCMI protocol reset (ARM)
- `drivers/reset/reset-raspberrypi.c` — Raspberry Pi
- `drivers/reset/reset-zynqmp.c` — Xilinx ZynqMP
- `drivers/reset/reset-ti-sci.c` — Texas Instruments SCI
- `drivers/reset/reset-qcom-aoss.c` / `reset-qcom-pdc.c` — Qualcomm
- `drivers/reset/reset-imx7.c` / `reset-imx-scu.c` — NXP i.MX
- `drivers/reset/reset-sunxi.c` — Allwinner (sunxi)
- `drivers/reset/reset-brcmstb.c` — Broadcom STB
- `drivers/reset/reset-aspeed.c` — ASPEED BMC SoCs
- `drivers/reset/reset-k210.c` / `reset-k230.c` — Kendryte RISC-V
- `drivers/reset/reset-th1520.c` — T-Head TH1520 RISC-V
- `drivers/reset/reset-uniphier.c` — Socionext UniPhier

#### الـ Device Tree Bindings
- `Documentation/devicetree/bindings/reset/`
- `include/dt-bindings/reset/`
## Phase 2: شرح الـ Reset Controller Framework

---

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

في أي SoC حديث (Qualcomm، Rockchip، NXP، إلخ) في مئات الـ IP blocks: USB controller، PCIe، HDMI، DSP، GPU. كل block منهم محتاج **reset line** — سلك واحد أو أكثر بيتحكم فيه عن طريق register في الـ reset controller الخاص بالـ SoC.

المشكلة القديمة كانت: كل driver بيكلم الـ reset registers **مباشرة** عن طريق عنوان ثابت hardcoded. النتيجة؟

- Driver الـ USB على Rockchip مختلف تماماً عن Driver الـ USB على NXP حتى لو الـ USB core نفسه واحد.
- مفيش API موحد — كل driver بيعمل `writel(0, RESET_REG)` بطريقته الخاصة.
- الـ Device Tree مش بيعرف يوصف العلاقة بين device والـ reset line بشكل عام.
- لو اتنين drivers محتاجين يشاركوا نفس الـ reset line، مفيش coordination.

---

### الحل — إيه اللي الـ kernel بيعمله؟

الـ kernel فصل المسألة لطبقتين:

| الطبقة | المسؤولية |
|--------|-----------|
| **Provider** (reset controller driver) | يعرف إزاي يكتب في registers الـ SoC المحدد عشان يعمل assert/deassert/reset |
| **Consumer** (device driver) | يطلب reset control باسم أو index ويستخدم API موحد بدون ما يعرف أي SoC |

الـ **Reset Controller Framework** هو الطبقة الوسيطة اللي بتربط الاتنين بعض، وبتجيب المعلومات دي من الـ Device Tree.

---

### التشبيه الواقعي — المولد الكهربائي والـ circuit breakers

تخيل لوحة كهربائية (distribution board) في عمارة:

```
┌─────────────────────────────────────────────────────┐
│              لوحة الكهرباء (SoC Reset Controller)   │
│                                                     │
│  [CB-0: USB]  [CB-1: PCIe]  [CB-2: GPU]  [CB-3: I2C]│
│    │               │            │             │      │
└────┼───────────────┼────────────┼─────────────┼──────┘
     │               │            │             │
   USB Driver     PCIe Driver  GPU Driver   I2C Driver
   (مستأجر A)    (مستأجر B)   (مستأجر C)  (مستأجر D)
```

الآن خلينا نعمل الـ mapping الكامل:

| عنصر في التشبيه | المقابل الحقيقي في الـ kernel |
|-----------------|-------------------------------|
| لوحة الكهرباء نفسها | `struct reset_controller_dev` — الـ provider |
| القاطع الكهربائي (circuit breaker) | `struct reset_control` — handle اللي الـ driver بيمسكه |
| رقم القاطع (CB-0, CB-1...) | `id` في الـ `reset_control` |
| مستأجر بيفتح/يقفل قاطعه | Driver بيعمل `deassert` / `assert` |
| مستأجر بيحجز قاطع لنفسه (exclusive) | `reset_control_get_exclusive()` |
| اتنين مستأجرين بيتشاركوا نفس القاطع | `reset_control_get_shared()` مع `deassert_count` |
| الكهربائي اللي بيفهم اللوحة | Provider driver (مثلاً `drivers/reset/reset-rk3xxx.c`) |
| عقد الإيجار (lease) | `devm_reset_control_get_*()` — بيتلغى automatically عند unload |
| الكهربائي بيترجم رقم الشقة لقاطع | `of_xlate()` callback |
| شقة مش محتاجة قاطع (optional) | `RESET_CONTROL_OPTIONAL_*` flags |

لما مستأجر B يقفل قاطع shared مع مستأجر C، الإضاءة متتطفيش لحد ما الاتنين يقفلوا — ده بالظبط الـ `deassert_count` logic في الـ shared reset.

---

### الـ Big Picture — مكان الـ subsystem في الـ kernel

```
┌─────────────────────────────────────────────────────────────┐
│                      Device Drivers (Consumers)             │
│   usb-dwc3.c    pcie-rockchip.c    gpu-lima.c    i2c-imx.c  │
│       │                │                │              │    │
│  reset_control_    reset_control_   reset_control_ reset_   │
│  get_exclusive()   get_shared()     get_exclusive() get_()  │
└───────────────────────────┬─────────────────────────────────┘
                            │  Consumer API  (reset.h)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Reset Controller Core  (drivers/reset/core.c)  │
│                                                             │
│  ┌──────────────────┐   ┌──────────────────────────────┐   │
│  │ reset_control    │   │ reset_controller_list        │   │
│  │ lookup via DT    │   │ (global list of all rcdevs)  │   │
│  │ of_xlate()       │   └──────────────────────────────┘   │
│  └──────────────────┘                                       │
│                                                             │
│  reset_control_assert()  ──► rcdev->ops->assert(rcdev, id) │
│  reset_control_deassert() ─► rcdev->ops->deassert(rcdev,id)│
│  reset_control_reset()  ──► rcdev->ops->reset(rcdev, id)   │
└───────────────────────────┬─────────────────────────────────┘
                            │  Provider API  (reset-controller.h)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Reset Controller Drivers (Providers)           │
│                                                             │
│  reset-rk3xxx.c      reset-imx7.c      reset-sunxi.c       │
│  (Rockchip)          (NXP i.MX7)       (Allwinner)         │
│                                                             │
│  كل واحد بيعمل register في reset_controller_list            │
│  وبيوفر struct reset_control_ops                            │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Hardware (SoC Registers)                  │
│                                                             │
│   CRU_SOFTRST0   CRU_SOFTRST1   CRU_SOFTRST2  ...          │
│   (bit per reset line)                                      │
└─────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — إيه الفكرة المحورية؟

الفكرة المحورية هي **الفصل بين "مين يحتاج reset" و"إزاي بيتعمل الـ reset"** عن طريق:

1. **`struct reset_control`** — handle مجرد (opaque) في يد الـ consumer. الـ consumer مش عارف فيه غير إنه موجود.
2. **`struct reset_controller_dev`** — الـ provider بيسجل نفسه بالـ ops وبالـ of_node.
3. **الـ Device Tree** بيوصف العلاقة: مين connected لمين.

---

### الـ Structs بالتفصيل وعلاقتها ببعض

#### `struct reset_control` — الـ handle عند الـ consumer

```c
/* defined in drivers/reset/core.c — مش public للـ consumer */
struct reset_control {
    struct reset_controller_dev *rcdev; /* pointer للـ provider */
    struct list_head list;              /* linked في rcdev->reset_control_head */
    unsigned int id;                    /* رقم الـ reset line داخل الـ rcdev */
    struct kref refcnt;                 /* reference counting */
    bool acquired;                      /* exclusive ومش released */
    bool shared;                        /* shared أم exclusive */
    bool array;                         /* هل ده array of resets */
    atomic_t deassert_count;            /* لحساب الـ shared deassert */
    atomic_t triggered_count;           /* لحساب الـ shared reset */
};
```

#### `struct reset_controller_dev` — الـ provider

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;    /* vtable: assert/deassert/reset/status */
    struct module *owner;
    struct list_head list;                  /* في reset_controller_list العالمية */
    struct list_head reset_control_head;    /* قائمة كل الـ handles المأخوذة منه */
    struct device *dev;
    struct device_node *of_node;            /* لربطه بالـ DT */
    const struct of_phandle_args *of_args;  /* للـ reset-gpios */
    int of_reset_n_cells;                   /* عدد cells في الـ specifier */
    int (*of_xlate)(...);                   /* ترجمة DT specifier → id */
    unsigned int nr_resets;                 /* عدد reset lines في هذا الـ controller */
};
```

#### `struct reset_control_ops` — الـ vtable

```c
struct reset_control_ops {
    /* pulse assert ثم deassert تلقائياً — لـ self-deasserting hardware */
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);

    /* assert فقط — يحط الـ IP block في حالة reset */
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);

    /* deassert فقط — يخرج الـ IP block من حالة reset */
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);

    /* اقرأ الحالة الحالية للـ reset line */
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

#### العلاقة بين الـ structs

```
reset_controller_list (global)
    │
    ├──► reset_controller_dev (Rockchip CRU)
    │       ├── ops ──► reset_control_ops { .assert=rk_assert, .deassert=rk_deassert }
    │       ├── nr_resets = 400
    │       ├── of_node ──► /clock-controller@ff760000
    │       └── reset_control_head
    │               ├──► reset_control { id=8,  shared=0, acquired=1 }  ← USB driver
    │               └──► reset_control { id=10, shared=1 }              ← PCIe driver (x2)
    │
    └──► reset_controller_dev (i.MX7 SRC)
            ├── ops ──► reset_control_ops { .reset=imx7_reset }
            ├── nr_resets = 16
            └── reset_control_head
                    └──► reset_control { id=2, shared=0, acquired=1 }   ← MIPI driver
```

---

### الـ Flags — matrix كامل

الـ `enum reset_control_flags` مبنية على أربع bits:

```
Bit 0: SHARED       — مشترك بين أكثر من consumer
Bit 1: OPTIONAL     — مش لازم يكون موجود في الـ DT
Bit 2: ACQUIRED     — exclusive ومحجوز فوراً
Bit 3: DEASSERTED   — بعد الحصول عليه ابدأ deassert مباشرة
```

| Flag | SHARED | OPTIONAL | ACQUIRED | DEASSERTED | الاستخدام |
|------|--------|----------|----------|------------|-----------|
| `EXCLUSIVE` | 0 | 0 | 1 | 0 | الأكثر شيوعاً — driver واحد فقط |
| `EXCLUSIVE_DEASSERTED` | 0 | 0 | 1 | 1 | احجز وابدأ تشغيل مباشرة |
| `EXCLUSIVE_RELEASED` | 0 | 0 | 0 | 0 | احجز لكن مش acquired — لازم `acquire()` بعدين |
| `SHARED` | 1 | 0 | 0 | 0 | اتنين drivers بيشاركوا reset line |
| `SHARED_DEASSERTED` | 1 | 0 | 0 | 1 | shared وابدأ deassert |
| `OPTIONAL_EXCLUSIVE` | 0 | 1 | 1 | 0 | ممكن مش موجود في الـ DT |
| `OPTIONAL_SHARED` | 1 | 1 | 0 | 0 | shared وممكن مش موجود |

---

### الـ Exclusive vs Shared — الفرق العملي

#### Exclusive Reset

```c
/* USB driver — ملكية كاملة */
rstc = reset_control_get_exclusive(dev, "usb");
/* لو driver تاني حاول ياخد نفس الـ reset → -EBUSY */

reset_control_assert(rstc);    /* حط USB في reset */
mdelay(1);
reset_control_deassert(rstc);  /* شغّل USB */
```

#### Shared Reset — الـ deassert_count logic

```c
/* Driver A */                     /* Driver B */
rstc_a = get_shared(dev, "bus");   rstc_b = get_shared(dev, "bus");
reset_control_deassert(rstc_a);    /* deassert_count = 1 */
                                   reset_control_deassert(rstc_b);  /* count = 2 */
reset_control_assert(rstc_a);      /* count = 1, مش بيعمل assert فعلي */
                                   reset_control_assert(rstc_b);    /* count = 0 → assert فعلي */
```

الـ core بيحتفظ بـ `deassert_count` كـ atomic، وبيعمل assert فعلي بس لما تيجي صفر.

#### Exclusive Released — الـ acquire/release pattern

```c
/* لما اتنين drivers بيتشاركوا حق التحكم في أوقات مختلفة */
rstc = reset_control_get_exclusive_released(dev, "phy");
/* الـ reset موجود لكن مش acquired حد تاني ممكن يـ acquire */

reset_control_acquire(rstc);   /* حجز حصري مؤقت */
reset_control_reset(rstc);     /* استخدام */
reset_control_release(rstc);   /* تحرير للاستخدام التاني */
```

---

### إزاي الـ Device Tree بيربط كل حاجة

مثال Rockchip RK3399:

```dts
/* Provider — الـ reset controller */
cru: clock-controller@ff760000 {
    compatible = "rockchip,rk3399-cru";
    reg = <0x0 0xff760000 0x0 0x1000>;
    #reset-cells = <1>;   /* of_reset_n_cells = 1 */
};

/* Consumer — USB controller */
usb_host0_ehci: usb@fe380000 {
    compatible = "generic-ehci";
    reg = <0x0 0xfe380000 0x0 0x10000>;
    resets = <&cru SRST_USBHOST0_AHB>;  /* phandle + id */
    reset-names = "ahb";                 /* اسم لـ reset_control_get_exclusive(dev, "ahb") */
};
```

**مسار الـ lookup بالخطوات:**

```
reset_control_get_exclusive(dev, "ahb")
    │
    ▼
__reset_control_get(dev, "ahb", 0, EXCLUSIVE)
    │
    ▼
of_parse_phandle_with_args(dev->of_node, "resets", "#reset-cells", ...)
    │  → يجيب: phandle = &cru, args[0] = SRST_USBHOST0_AHB (= 96)
    ▼
__of_reset_control_get(cru_node, NULL, 0, EXCLUSIVE)
    │
    ▼
/* بحث في reset_controller_list عن rcdev بـ of_node = &cru */
    │
    ▼
rcdev->of_xlate(rcdev, &reset_spec)
    │  → يرجع id = 96
    ▼
alloc reset_control { rcdev=cru_rcdev, id=96, shared=0 }
```

---

### الـ Consumer API — الطبقات

#### 1. Single Reset — الأبسط

```c
/* get + use + put يدوياً */
struct reset_control *rstc;
rstc = reset_control_get_exclusive(dev, "core");
reset_control_assert(rstc);
udelay(10);
reset_control_deassert(rstc);
reset_control_put(rstc);  /* لازم manual put */
```

#### 2. Devres (devm) — الأكثر استخداماً

```c
/* الـ put بيتعمل automatically لما الـ driver يتـ unbind */
rstc = devm_reset_control_get_exclusive(dev, "core");
if (IS_ERR(rstc))
    return PTR_ERR(rstc);
reset_control_deassert(rstc);
/* مش محتاج put */
```

#### 3. Bulk — لما الـ IP محتاج أكثر من reset line

```c
/* مثلاً: GPU محتاج reset للـ core، الـ memory interface، والـ shader */
static const struct reset_control_bulk_data gpu_resets[] = {
    { .id = "core" },
    { .id = "mem"  },
    { .id = "shader" },
};

/* copy لـ local array عشان kernel بيكتب فيه */
struct reset_control_bulk_data resets[ARRAY_SIZE(gpu_resets)];
memcpy(resets, gpu_resets, sizeof(resets));

ret = devm_reset_control_bulk_get_exclusive(dev, ARRAY_SIZE(resets), resets);
/* الآن resets[0].rstc, resets[1].rstc, resets[2].rstc كلهم populated */

reset_control_bulk_assert(ARRAY_SIZE(resets), resets);
udelay(10);
reset_control_bulk_deassert(ARRAY_SIZE(resets), resets);
```

#### 4. Array — لما الـ DT بيحدد list مش بأسماء

```c
/* بيجمع كل الـ resets في الـ DT node في handle واحد */
rstc = devm_reset_control_array_get_exclusive(dev);
reset_control_reset(rstc);  /* بيعمل reset لكل الـ lines دفعة واحدة */
```

---

### `reset_control_reset()` vs `assert/deassert` — الفرق الجوهري

```
reset_control_reset()           reset_control_assert() + deassert()
────────────────────────        ─────────────────────────────────────
لـ self-deasserting hardware    لـ hardware بيحتاج تحكم يدوي
(hardware بيعمل deassert        (driver مسؤول عن توقيت الـ deassert)
 بعد pulse تلقائياً)

مش مسموح على shared controls   مسموح على shared مع deassert_count

مثال: بعض الـ GPIO resets      مثال: USB PHY، PCIe، HDMI
```

---

### الـ Optional Controls

لما device مش موجودة في الـ DT (مثلاً feature غير موجودة في هذا الـ board):

```c
/* بدون optional: */
rstc = reset_control_get_exclusive(dev, "usb3");
if (IS_ERR(rstc))
    return PTR_ERR(rstc);  /* بيفشل لو مش موجود في DT */

/* مع optional: */
rstc = reset_control_get_optional_exclusive(dev, "usb3");
if (IS_ERR(rstc))
    return PTR_ERR(rstc);  /* بيفشل بس لو في error حقيقي */
if (rstc == NULL)
    ; /* مش موجود في DT — تمام، نكمل بدونه */
```

---

### إيه اللي الـ framework بيملكه مقابل اللي بيفوضه للـ drivers

| الـ Framework يملك | الـ Driver يقرر |
|---------------------|-----------------|
| قائمة كل الـ reset controllers | إزاي بيكتب في الـ hardware registers |
| الـ lookup من DT لـ rcdev | الـ `of_xlate` لترجمة specifier لـ id |
| الـ reference counting للـ handles | عدد الـ reset lines (`nr_resets`) |
| الـ shared deassert_count logic | هل الـ hardware self-deasserting أم لا |
| الـ exclusive/acquired enforcement | هل الـ `status()` متاح |
| الـ devm cleanup | الـ `of_reset_n_cells` المناسب للـ SoC |
| الـ array و bulk helpers | — |

---

### مثال Provider Driver — Rockchip بشكل مبسط

```c
/* drivers/clk/rockchip/clk.c (مبسط) */

static int rk3399_reset_assert(struct reset_controller_dev *rcdev,
                                unsigned long id)
{
    struct rockchip_clk_provider *ctx = /* container_of */ ...;
    /* id يحدد الـ register والـ bit */
    unsigned int reg  = id / 16;  /* كل register فيه 16 reset */
    unsigned int bit  = id % 16;

    /* Rockchip بتستخدم write-mask: upper 16 bits هي mask */
    writel(BIT(bit) | (BIT(bit) << 16),
           ctx->reg_base + RK3399_SOFTRST_CON(reg));
    return 0;
}

static int rk3399_reset_deassert(struct reset_controller_dev *rcdev,
                                  unsigned long id)
{
    /* نفس المنطق لكن بيكتب 0 في الـ bit مع mask */
    writel(BIT(bit) << 16,
           ctx->reg_base + RK3399_SOFTRST_CON(reg));
    return 0;
}

static const struct reset_control_ops rk3399_reset_ops = {
    .assert   = rk3399_reset_assert,
    .deassert = rk3399_reset_deassert,
};

/* في init: */
ctx->reset.ops         = &rk3399_reset_ops;
ctx->reset.of_node     = np;
ctx->reset.nr_resets   = 408;
ctx->reset.of_reset_n_cells = 1;
/* of_xlate مش محتاجه — الـ default of_reset_simple_xlate كافي */
reset_controller_register(&ctx->reset);
```

---

### subsystem تانية محتاج تفهمها معاه

- **Clock Framework (CCF)**: الـ reset controller على أغلب الـ SoCs جزء من نفس الـ Clock and Reset Unit (CRU). الـ clock subsystem بيوفر `clk_prepare_enable()` بينما الـ reset subsystem بيوفر `reset_control_deassert()`. الاتنين بالعادة بيتعملوا تسجيل من نفس الـ driver. اعرف الـ CCF قبل ما تكمل.
- **Device Tree / OF subsystem**: الـ framework يعتمد بشكل أساسي على `of_parse_phandle_with_args()` عشان يربط consumer بـ provider. لازم تفهم الـ DT phandle mechanism.
- **devres (devm)**: الـ `devm_*` variants بتعتمد على الـ managed resource system. الـ `devm_reset_controller_register()` بيضيف cleanup action تلقائية.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. Flags، Enums، وخيارات الـ Config — Cheatsheet

#### الـ Config Option

| Option | المعنى |
|--------|--------|
| `CONFIG_RESET_CONTROLLER` | يفعّل subsystem الـ reset كامل — لو مش موجود كل الـ APIs بتبقى stubs بترجع 0 أو `NULL` |

---

#### الـ Bit Flags — `RESET_CONTROL_FLAGS_BIT_*`

| Flag | Value | المعنى |
|------|-------|--------|
| `RESET_CONTROL_FLAGS_BIT_SHARED` | `BIT(0)` | الـ reset line مشتركة بين أكتر من consumer |
| `RESET_CONTROL_FLAGS_BIT_OPTIONAL` | `BIT(1)` | لو الـ reset مش موجود في DT، إرجع `NULL` بدل error |
| `RESET_CONTROL_FLAGS_BIT_ACQUIRED` | `BIT(2)` | الـ exclusive control اتاخد فعلاً (acquired state) |
| `RESET_CONTROL_FLAGS_BIT_DEASSERTED` | `BIT(3)` | deassert تلقائي بعد الـ get |

---

#### الـ Enum `reset_control_flags` — كل القيم والتركيبة

| Enum Value | SHARED | OPTIONAL | ACQUIRED | DEASSERTED | الاستخدام |
|-----------|--------|----------|----------|-----------|-----------|
| `RESET_CONTROL_EXCLUSIVE` | 0 | 0 | 1 | 0 | أشيع حالة — exclusive مباشرة |
| `RESET_CONTROL_EXCLUSIVE_DEASSERTED` | 0 | 0 | 1 | 1 | exclusive + deassert تلقائي |
| `RESET_CONTROL_EXCLUSIVE_RELEASED` | 0 | 0 | 0 | 0 | exclusive بس "released" — لازم acquire يدوي |
| `RESET_CONTROL_SHARED` | 1 | 0 | 0 | 0 | shared بين drivers |
| `RESET_CONTROL_SHARED_DEASSERTED` | 1 | 0 | 0 | 1 | shared + deassert تلقائي |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE` | 0 | 1 | 1 | 0 | exclusive، لو مش موجود → `NULL` |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE_DEASSERTED` | 0 | 1 | 1 | 1 | optional exclusive + deassert |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE_RELEASED` | 0 | 1 | 0 | 0 | optional released exclusive |
| `RESET_CONTROL_OPTIONAL_SHARED` | 1 | 1 | 0 | 0 | shared optional |
| `RESET_CONTROL_OPTIONAL_SHARED_DEASSERTED` | 1 | 1 | 0 | 1 | shared optional + deassert |

> **القاعدة:** القيم دي **مش قابلة للـ OR** مع بعض — كل قيمة enum هي combination جاهزة من الـ bits.

---

### 1. الـ Structs المهمة

#### 1.1 `struct reset_control_ops` — واجهة الـ Hardware Driver

**الغرض:** الـ vtable اللي كل reset controller driver بيملّيه — بتعرّف إيه اللي الـ hardware يقدر يعمله.

```c
/* من include/linux/reset-controller.h */
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

| Field | النوع | المعنى |
|-------|-------|--------|
| `reset` | function pointer | pulse كامل على الـ reset line — self-deasserting |
| `assert` | function pointer | يحط الـ hardware في reset (active low عادةً) |
| `deassert` | function pointer | يطلّع الـ hardware من reset |
| `status` | function pointer | يسأل: هل الـ line في reset دلوقتي؟ |

> لو الـ hardware ما بيدعمش `assert`/`deassert` منفصلين، الـ driver بيستخدم `reset` فقط.

---

#### 1.2 `struct reset_controller_dev` — الـ Controller نفسه

**الغرض:** يمثّل جهاز reset controller — يعني الـ IP block اللي بيتحكم في عدة reset lines في الـ SoC.

```c
/* من include/linux/reset-controller.h */
struct reset_controller_dev {
    const struct reset_control_ops *ops;   /* vtable — ops الـ hardware */
    struct module *owner;                  /* الـ module اللي سجّل الـ controller */
    struct list_head list;                 /* node في reset_controller_list العالمية */
    struct list_head reset_control_head;   /* قائمة الـ reset_control المفتوحة من consumer */
    struct device *dev;                    /* الـ device المقابل في driver model */
    struct device_node *of_node;           /* DT node كـ phandle target */
    const struct of_phandle_args *of_args; /* لـ GPIO-based resets */
    int of_reset_n_cells;                  /* عدد cells في الـ DT specifier */
    int (*of_xlate)(...);                  /* ترجمة DT specifier → reset ID */
    unsigned int nr_resets;                /* عدد الـ reset lines في الـ controller */
};
```

| Field | الأهمية |
|-------|--------|
| `ops` | بوابة الـ hardware — كل عملية بتمشي من هنا |
| `list` | مسجّل في `reset_controller_list` العالمية المحمية بـ `reset_list_mutex` |
| `reset_control_head` | كل consumer حيجيب `reset_control` بيتضاف لهنا |
| `of_node` / `of_args` | واحد بس منهم يكون موجود — `of_node` للـ normal، `of_args` للـ GPIO |
| `of_xlate` | لو مش محدد، الـ core بيستخدم `of_reset_simple_xlate` اللي بترجع `args[0]` مباشرة |
| `nr_resets` | حد أقصى لـ ID — `of_reset_simple_xlate` بتتأكد إن `args[0] < nr_resets` |

---

#### 1.3 `struct reset_control` — رابط الـ Consumer بالـ Controller

**الغرض:** handle اللي الـ consumer driver بيمسكه — يمثّل علاقة consumer واحد بـ reset line واحدة.

```c
/* من drivers/reset/core.c — سطر 51 */
struct reset_control {
    struct reset_controller_dev *rcdev;  /* الـ controller اللي المسيول عن الـ line دي */
    struct list_head list;               /* node في rcdev->reset_control_head */
    unsigned int id;                     /* رقم الـ reset line داخل الـ controller */
    struct kref refcnt;                  /* reference count — عدد getters */
    bool acquired;                       /* exclusive control: هل اتاخد؟ */
    bool shared;                         /* هل ده shared control؟ */
    bool array;                          /* هل ده في الحقيقة reset_control_array؟ */
    atomic_t deassert_count;             /* عدد deasserts — للـ shared لضمان symmetry */
    atomic_t triggered_count;            /* عدد resets — للـ shared (0 أو 1) */
};
```

| Field | المعنى |
|-------|--------|
| `rcdev` | pointer للـ controller — طريق الوصول للـ ops الفعلية |
| `id` | الرقم الداخلي للـ reset line بعد ترجمة الـ DT specifier |
| `refcnt` | `kref` — لو وصل لصفر، الـ object اتحذف |
| `acquired` | خاص بـ exclusive released controls — `reset_control_acquire()` بيحطها `true` |
| `shared` | يغيّر سلوك `assert`/`deassert` — بيعتمد على `deassert_count` |
| `deassert_count` | atomic counter — deassert فعلي بيحصل لما count يوصل صفر |
| `triggered_count` | يمنع `reset()` المتكرر على الـ shared controls |

---

#### 1.4 `struct reset_control_array` — لما الـ Device محتاج Reset Lines متعددة

**الغرض:** wrapper على `reset_control` العادي بس يحتوي array من الـ controls — بيتعامل معاه كـ control واحد.

```c
/* من drivers/reset/core.c — سطر 69 */
struct reset_control_array {
    struct reset_control base;           /* MUST يكون أول field — الـ cast بيعتمد عليه */
    unsigned int num_rstcs;              /* عدد الـ controls في الـ array */
    struct reset_control *rstc[] __counted_by(num_rstcs); /* flexible array */
};
```

**ازاي الـ casting بيشتغل:**
```c
/* rstc_to_array في core.c */
static inline struct reset_control_array *
rstc_to_array(struct reset_control *rstc) {
    return container_of(rstc, struct reset_control_array, base);
}
/* لو rstc->array == true، الـ core بيعمل cast للـ array */
```

---

#### 1.5 `struct reset_control_bulk_data` — للـ Consumer اللي محتاج Bulk API

**الغرض:** array element بسيطة — الـ consumer بيحدد الـ name، والـ framework بيملّي الـ pointer.

```c
/* من include/linux/reset.h */
struct reset_control_bulk_data {
    const char        *id;    /* اسم الـ reset line زي ما في DT */
    struct reset_control *rstc; /* الـ core بيملّيه بعد الـ get */
};
```

**مثال استخدام:**
```c
static const struct reset_control_bulk_data my_resets[] = {
    { .id = "axi" },
    { .id = "apb" },
    { .id = "core" },
};
/* بعدين: */
devm_reset_control_bulk_get_exclusive(dev, ARRAY_SIZE(my_resets), my_resets);
/* الـ framework بيملّي my_resets[i].rstc لكل entry */
```

---

#### 1.6 `struct reset_gpio_lookup` — Bridge بين GPIO وـ Reset Framework

**الغرض:** بيربط الـ GPIO-based reset controllers بالـ reset subsystem — ad-hoc created entries.

```c
/* من drivers/reset/core.c */
struct reset_gpio_lookup {
    struct of_phandle_args of_args;   /* phandle + GPIO number من DT */
    struct fwnode_handle *swnode;     /* software node للـ GPIO provider */
    struct list_head list;            /* node في reset_gpio_lookup_list */
};
```

محمي بـ `reset_gpio_lookup_mutex` منفصل عن `reset_list_mutex`.

---

### 2. العلاقات بين الـ Structs — ASCII Diagram

```
                    ┌─────────────────────────────────────────┐
                    │          reset_controller_list           │  ← global list
                    │          (reset_list_mutex)              │
                    └───────────────┬─────────────────────────┘
                                    │ list_head
                                    ▼
              ┌─────────────────────────────────────────────┐
              │         reset_controller_dev (rcdev)         │
              │  ┌─────────────────────────────────────┐    │
              │  │ ops ──────────────────────────────► │reset_control_ops│
              │  │ dev ──────────────────────────────► struct device    │
              │  │ of_node ──────────────────────────► device_node      │
              │  │ reset_control_head ◄──────────────┐ │    │
              │  │ nr_resets = N                      │ │    │
              │  └────────────────────────────────────┼─┘    │
              └───────────────────────────────────────┼──────┘
                                                      │ list_head (per control)
                         ┌────────────────────────────┘
                         │
              ┌──────────▼───────────────────────────────┐
              │          reset_control (rstc)             │
              │  rcdev ──────────────────────────────────►│ (back to rcdev)
              │  id = 3          (reset line index)       │
              │  shared = false                           │
              │  acquired = true                          │
              │  refcnt = 1                               │
              │  deassert_count = 1                       │
              └──────────────────────────────────────────┘
                         ▲
                         │ (لو array=true)
              ┌──────────┴───────────────────────────────┐
              │       reset_control_array                 │
              │  base (reset_control) ─────────────────► │
              │  num_rstcs = 3                            │
              │  rstc[0] ──────────────────────────────► reset_control A
              │  rstc[1] ──────────────────────────────► reset_control B
              │  rstc[2] ──────────────────────────────► reset_control C
              └───────────────────────────────────────────┘

Consumer side:
              ┌───────────────────────────────────────────┐
              │      reset_control_bulk_data[]             │
              │  [0] id="axi"  rstc ──────────────────►  reset_control
              │  [1] id="apb"  rstc ──────────────────►  reset_control
              │  [2] id="core" rstc ──────────────────►  reset_control
              └───────────────────────────────────────────┘
```

---

### 3. Lifecycle Diagrams

#### 3.1 Lifecycle الـ `reset_controller_dev` (Provider Side)

```
Driver Probe
    │
    ▼
allocate rcdev (devm_kzalloc)
    │
    ▼
set rcdev->ops = &my_reset_ops
set rcdev->dev = dev
set rcdev->of_node = dev->of_node
set rcdev->nr_resets = MY_NR_RESETS
    │
    ▼
devm_reset_controller_register(dev, rcdev)
    │
    ├─► reset_controller_register(rcdev)
    │       ├─► INIT_LIST_HEAD(&rcdev->reset_control_head)
    │       ├─► mutex_lock(&reset_list_mutex)
    │       ├─► list_add(&rcdev->list, &reset_controller_list)
    │       └─► mutex_unlock(&reset_list_mutex)
    │
    ▼
[rcdev مسجّل — جاهز يخدم consumers]
    │
    ▼   (on driver detach — devm cleanup)
devm_reset_controller_release()
    │
    └─► reset_controller_unregister(rcdev)
            ├─► mutex_lock(&reset_list_mutex)
            ├─► list_del(&rcdev->list)
            └─► mutex_unlock(&reset_list_mutex)
```

#### 3.2 Lifecycle الـ `reset_control` (Consumer Side — Exclusive)

```
Consumer Driver Probe
    │
    ▼
devm_reset_control_get_exclusive(dev, "core")
    │
    └─► __devm_reset_control_get(dev, "core", 0, EXCLUSIVE)
            │
            └─► __reset_control_get(dev, "core", 0, EXCLUSIVE)
                    │
                    ├─► of_parse_clkspec → get DT phandle
                    ├─► find rcdev in reset_controller_list
                    ├─► rcdev->of_xlate(rcdev, spec) → id
                    ├─► look for existing rstc with same id in rcdev->reset_control_head
                    │       ├─► [found, shared=false] → return -EBUSY
                    │       └─► [not found] → kzalloc new reset_control
                    │                           set rstc->rcdev = rcdev
                    │                           set rstc->id = id
                    │                           set rstc->shared = false
                    │                           set rstc->acquired = true
                    │                           kref_init(&rstc->refcnt)
                    │                           list_add to rcdev->reset_control_head
                    └─► return rstc
    │
    ▼
[rstc جاهز للاستخدام]
    │
    ├─► reset_control_assert(rstc)
    │       └─► rcdev->ops->assert(rcdev, rstc->id)
    │
    ├─► reset_control_deassert(rstc)
    │       └─► rcdev->ops->deassert(rcdev, rstc->id)
    │
    ├─► reset_control_reset(rstc)  [pulse]
    │       └─► rcdev->ops->reset(rcdev, rstc->id)
    │
    ▼   (on driver detach — devm cleanup)
reset_control_put(rstc)
    │
    └─► kref_put(&rstc->refcnt, reset_control_release)
            └─► [refcnt == 0]
                    ├─► list_del(&rstc->list)
                    └─► kfree(rstc)
```

#### 3.3 Lifecycle الـ `reset_control` — Exclusive Released (acquire/release pattern)

```
get_exclusive_released(dev, "usb")
    │
    └─► rstc->acquired = false  ← الفرق الوحيد
    │
    ▼
[rstc موجود بس مش acquired — محدش تاني يقدر يعمل exclusive get]
    │
    ▼
reset_control_acquire(rstc)
    │
    └─► rstc->acquired = true
    │
    ▼
reset_control_assert / deassert / reset
    │
    ▼
reset_control_release(rstc)
    │
    └─► rstc->acquired = false
    │
    ▼   [driver تاني ممكن يعمل acquire دلوقتي]
```

---

### 4. Call Flow Diagrams

#### 4.1 Flow: `reset_control_deassert()` على Shared Control

```
consumer A calls reset_control_deassert(rstc_A)
    │
    └─► core: rstc->shared == true
            │
            └─► atomic_inc(&rstc->deassert_count)
                    │
                    └─► [was 0 → now 1] → actually call hardware
                            │
                            └─► rcdev->ops->deassert(rcdev, rstc->id)
                                    │
                                    └─► [driver writes HW register]

consumer B calls reset_control_deassert(rstc_B)   [same line]
    │
    └─► atomic_inc(&rstc->deassert_count)
            │
            └─► [was 1 → now 2] → already deasserted, skip HW call

consumer A calls reset_control_assert(rstc_A)
    │
    └─► atomic_dec_return(&rstc->deassert_count)
            │
            └─► [result = 1] → still consumers → skip HW assert

consumer B calls reset_control_assert(rstc_B)
    │
    └─► atomic_dec_return(&rstc->deassert_count)
            │
            └─► [result = 0] → last consumer → actually assert
                    │
                    └─► rcdev->ops->assert(rcdev, rstc->id)
```

#### 4.2 Flow: الـ get من DT — Lookup Chain كاملة

```
devm_reset_control_get_exclusive(dev, "axi")
    │
    └─► __devm_reset_control_get(dev, "axi", 0, EXCLUSIVE)
            │
            └─► __reset_control_get(dev, "axi", 0, EXCLUSIVE)
                    │
                    ├─► [device has OF node?]
                    │       YES → __of_reset_control_get(node, "axi", 0, EXCLUSIVE)
                    │               │
                    │               ├─► of_parse_phandle_with_args(node, "resets",
                    │               │       "#reset-cells", index, &args)
                    │               │
                    │               ├─► find rcdev: scan reset_controller_list
                    │               │       compare rcdev->of_node == args.np
                    │               │
                    │               └─► __reset_control_get_internal(rcdev, id, flags)
                    │
                    └─► [device is ACPI?]
                            → acpi_reset_control_get path (similar)
```

#### 4.3 Flow: `reset_control_reset()` على Array

```
reset_control_reset(rstc)  [rstc->array == true]
    │
    └─► rstc_to_array(rstc) → resets (reset_control_array*)
            │
            └─► reset_control_array_reset(resets)
                    │
                    ├─► for i in 0..num_rstcs:
                    │       reset_control_reset(resets->rstc[i])
                    │           │
                    │           └─► [rstc->array == false]
                    │                   └─► rcdev->ops->reset(rcdev, id)
                    │
                    └─► [all succeeded → return 0]
                        [any failed → return error, partial state possible]
```

#### 4.4 Flow: `device_reset()` — الـ High-Level Helper

```
device_reset(dev)
    │
    └─► __device_reset(dev, optional=false)
            │
            └─► rstc = __reset_control_get(dev, NULL, 0, EXCLUSIVE)
                    │
                    ├─► [error] → return error
                    └─► [ok]
                            │
                            └─► reset_control_reset(rstc)
                                    │
                                    └─► rcdev->ops->reset(rcdev, id)
                                            │
                                            ├─► [ok] → reset_control_put(rstc)
                                            │               → return 0
                                            └─► [error] → reset_control_put(rstc)
                                                            → return error
```

---

### 5. استراتيجية الـ Locking

#### الـ Locks الموجودة

| Lock | النوع | اللي بيحميه |
|------|-------|------------|
| `reset_list_mutex` | `struct mutex` | `reset_controller_list` العالمية — add/del الـ rcdev |
| `reset_gpio_lookup_mutex` | `struct mutex` | `reset_gpio_lookup_list` — GPIO lookup entries |
| `rstc->deassert_count` | `atomic_t` | عداد الـ deassert للـ shared controls — lockless |
| `rstc->triggered_count` | `atomic_t` | عداد الـ reset للـ shared controls — lockless |
| `rstc->refcnt` | `struct kref` | reference counting — atomic internally |

#### Lock Ordering

```
reset_list_mutex          ← المستوى الأعلى
    │
    └─► يُمسك عند:
        - reset_controller_register()   [write]
        - reset_controller_unregister() [write]
        - __reset_control_get()         [read — scan القائمة]
        - reset_control_put()           [write — del من القائمة]

reset_gpio_lookup_mutex   ← مستقل تماماً — لا يتداخل مع reset_list_mutex
    │
    └─► يُمسك عند:
        - إضافة/حذف reset_gpio_lookup entries
```

#### الـ Shared Controls وـ Lockless Atomics

**للـ `deassert_count`:**
- `atomic_inc()` عند كل `deassert` — بدون lock
- `atomic_dec_return()` عند كل `assert` — لو النتيجة 0 يبعت للـ HW
- ده safe لأن الـ HW state consistent مع الـ count

**للـ `triggered_count`:**
- بيمنع إن الـ shared control يتعمل `reset()` لما حد تاني `deassert`-ه

#### ملاحظات مهمة على الـ Locking

1. **الـ Consumer-side operations** (`assert`/`deassert`/`reset`) **بدون mutex** — بيعتمدوا على الـ atomic counters للـ shared وعلى الـ `acquired` flag للـ exclusive.
2. **الـ `reset_list_mutex`** بيتمسك فقط وقت register/unregister وgetالـ control — مش وقت الـ actual reset operations — عشان يقلل الـ contention.
3. **الـ `kref`** بيضمن إن الـ `reset_control` object ما يتحذفش وفيه references — `reset_control_put()` هو الـ endpoint الوحيد للـ free.
4. **الـ drivers** المفروض **ما يمسكوش locks عند استدعاء الـ ops callbacks** — الـ controller driver نفسه مسؤول عن protect الـ hardware register access (spin_lock أو mutex حسب context).
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet لكل الـ APIs

#### الـ Runtime Control Functions

| Function | Signature (مختصرة) | الغرض |
|---|---|---|
| `reset_control_reset` | `int (rstc)` | assert ثم deassert فوراً |
| `reset_control_rearm` | `int (rstc)` | إعادة تمكين triggered reset لـ shared |
| `reset_control_assert` | `int (rstc)` | وضع الـ hardware في حالة reset |
| `reset_control_deassert` | `int (rstc)` | إخراج الـ hardware من حالة reset |
| `reset_control_status` | `int (rstc)` | قراءة حالة الـ reset line |
| `reset_control_acquire` | `int (rstc)` | الاستيلاء على exclusive released control |
| `reset_control_release` | `void (rstc)` | تحرير exclusive released control |

#### الـ Bulk Runtime Functions

| Function | الغرض |
|---|---|
| `reset_control_bulk_reset` | reset لعدة controls دفعة واحدة |
| `reset_control_bulk_assert` | assert لعدة controls |
| `reset_control_bulk_deassert` | deassert لعدة controls |
| `reset_control_bulk_acquire` | acquire لعدة exclusive released controls |
| `reset_control_bulk_release` | release لعدة controls |

#### الـ Get/Put (Lifecycle) Functions

| Function | الغرض | مع devm؟ |
|---|---|---|
| `reset_control_get_exclusive` | exclusive handle بالاسم | لا |
| `reset_control_get_exclusive_released` | exclusive released بالاسم | لا |
| `reset_control_get_shared` | shared handle بالاسم | لا |
| `reset_control_get_optional_exclusive` | optional exclusive | لا |
| `reset_control_get_optional_shared` | optional shared | لا |
| `reset_control_put` | تحرير handle | لا |
| `devm_reset_control_get_exclusive` | exclusive + auto-put | نعم |
| `devm_reset_control_get_exclusive_deasserted` | exclusive + deassert عند الحصول | نعم |
| `devm_reset_control_get_shared` | shared + auto-put | نعم |
| `devm_reset_control_get_shared_deasserted` | shared + deassert | نعم |
| `devm_reset_control_get_optional_exclusive` | optional exclusive + auto-put | نعم |
| `devm_reset_control_get_optional_shared` | optional shared + auto-put | نعم |
| `device_reset` | reset الـ device مباشرة | — |
| `device_reset_optional` | نفس السابق بدون error لو مش موجود | — |

#### الـ OF (Device Tree) Get Functions

| Function | الغرض |
|---|---|
| `of_reset_control_get_exclusive` | exclusive من device_node بالاسم |
| `of_reset_control_get_optional_exclusive` | optional exclusive من device_node |
| `of_reset_control_get_shared` | shared من device_node بالاسم |
| `of_reset_control_get_exclusive_by_index` | exclusive من device_node بالـ index |
| `of_reset_control_get_shared_by_index` | shared من device_node بالـ index |

#### الـ Array / Bulk Get Functions

| Function | الغرض |
|---|---|
| `devm_reset_control_array_get_exclusive` | كل الـ resets في الـ DT كـ array exclusive |
| `devm_reset_control_array_get_shared` | كل الـ resets كـ array shared |
| `of_reset_control_array_get_exclusive` | array exclusive من device_node |
| `reset_control_get_count` | عدد الـ reset lines المتاحة للـ device |

---

### Group 1: الـ Core Runtime Operations

دي الـ functions اللي بتتعامل مع الـ reset line مباشرةً بعد ما تحصل على الـ handle. كل عملية reset في الـ kernel بتمر من هنا.

---

#### `reset_control_reset`

```c
int reset_control_reset(struct reset_control *rstc);
```

**الـ function دي بتعمل assert ثم deassert للـ reset line في operation واحدة.** مفيدة لما تحتاج تعمل pulse للـ hardware لإعادة تشغيله من غير ما تحتاج تتحكم في التوقيت. **مش مسموح** باستخدامها مع **shared** reset controls لأن الـ shared logic بتعتمد على deassert count ولو جاء reset في النص هيكسر الـ counting.

- **`rstc`**: الـ handle المستلم من أي `reset_control_get_*` function.
- **Return**: `0` عند النجاح، أو negative errno (`-EPERM` لو shared، `-EBUSY` لو في حالة غلط).
- **Key details**: مش thread-safe بشكل ضمني — الـ reset core نفسه بيحمي الـ hardware state بـ internal locking. لو `rstc == NULL` (optional مش موجود) بيرجع `0` مباشرة.
- **Caller context**: يُستخدم من driver probe أو runtime PM callbacks.

---

#### `reset_control_rearm`

```c
int reset_control_rearm(struct reset_control *rstc);
```

**بتعيد تمكين الـ triggered reset على الـ shared control بعد ما اتعمل له deassert.** الـ triggered reset هو نوع خاص من الـ resets بيحتاج إعادة arming صريح بعد كل استخدام. بتوجد بشكل أساسي في بعض الـ SoCs اللي فيها hardware reset mechanism غير عادي.

- **`rstc`**: الـ reset control handle.
- **Return**: `0` أو negative errno.
- **Key details**: مش موجودة في كل الـ reset controllers — بتعتمد على وجود `.reset` op في الـ `reset_control_ops`. لو `rstc == NULL` بيرجع `0`.
- **Caller context**: بعد `reset_control_deassert` مباشرة في triggered reset contexts.

---

#### `reset_control_assert`

```c
int reset_control_assert(struct reset_control *rstc);
```

**بتوضع الـ reset line في الـ asserted state (يعني الـ hardware يدخل في حالة reset).** في الـ **exclusive** mode، بتعمل assert مباشرة. في الـ **shared** mode، الـ reset core بيديك assert فعلي بس لو الـ `deassert_count` وصل لـ صفر، يعني كل اللي عملوا deassert عملوا assert.

- **`rstc`**: الـ reset control handle.
- **Return**: `0` أو negative errno. في الـ shared mode ممكن يرجع `0` بدون ما يعمل assert فعلي (لأن في حد تاني لسه محتاج الـ deassert).
- **Key details**: على الـ shared control، لازم يتسبق بـ `reset_control_deassert` مسبق وإلا بيرجع `-EPERM`. الـ locking داخلي في الـ reset core.
- **Caller context**: `probe` failure path، `remove`، أو suspend.

---

#### `reset_control_deassert`

```c
int reset_control_deassert(struct reset_control *rstc);
```

**بتخرج الـ hardware من حالة reset (deasserted state).** في الـ **shared** mode، بتزود الـ `deassert_count` بواحد، والـ reset ما بيتشالش فعلاً إلا لما أول `deassert_count` يعدي من صفر لواحد. الـ consumers التانيين اللي شاركوا الـ reset line محتاجين هم كمان يعملوا deassert قبل ما الـ hardware يشتغل.

- **`rstc`**: الـ reset control handle.
- **Return**: `0` أو negative errno.
- **Key details**: الـ pair مع `reset_control_assert`. في حالة `rstc == NULL`، بيرجع `0` فوراً. الـ `deassert_count` محمي بـ spinlock داخلي في الـ reset core.
- **Caller context**: `probe` بعد ما تجهز الـ clocks، أو resume.

---

#### `reset_control_status`

```c
int reset_control_status(struct reset_control *rstc);
```

**بتقرأ الحالة الحالية للـ reset line من الـ hardware register مباشرة.** القيمة بترجع `1` لو الـ reset asserted، `0` لو deasserted. مش كل الـ reset controllers بيدعموا الـ status read.

- **`rstc`**: الـ reset control handle.
- **Return**: `1` (asserted)، `0` (deasserted)، أو negative errno (`-ENOSYS` لو الـ controller مش بيدعم الـ status op).
- **Key details**: read-only، مش بيغير الـ state. ممكن يبقى slow لو الـ reset controller بيتحكم فيه عن طريق slow bus (I2C/SPI-based PMIC).
- **Caller context**: diagnostic، أو للـ drivers اللي محتاجة تتأكد من الـ initial state.

---

#### `reset_control_acquire`

```c
int reset_control_acquire(struct reset_control *rstc);
```

**بتاخد الـ ownership الفعلي لـ exclusive released control.** الـ `RESET_CONTROL_EXCLUSIVE_RELEASED` controls بيتاخدوا من البداية released (مش acquired)، ومش ممكن تستخدمهم قبل ما تعمل acquire صريح. ده بيتيح لـ multiple drivers إنهم يتشاركوا reset line واحدة مع exclusive access بالتناوب.

- **`rstc`**: exclusive released handle.
- **Return**: `0` أو `-EBUSY` لو حد تاني acquire في نفس الوقت.
- **Key details**: بيحتاج `reset_control_release` بعدين. مش مناسب للـ shared resets. الـ acquire/release sequence هو manual locking على مستوى الـ reset line.
- **Caller context**: من driver اللي عاوز يستخدم الـ reset line مؤقتاً.

**Pseudocode flow:**
```
reset_control_acquire(rstc):
    if rstc->acquired:
        return -EBUSY
    rstc->acquired = true
    return 0
```

---

#### `reset_control_release`

```c
void reset_control_release(struct reset_control *rstc);
```

**بتحرر الـ ownership اللي اتاخد بـ `reset_control_acquire`.** بعدها أي driver تاني ممكن يعمل acquire على نفس الـ reset line.

- **`rstc`**: exclusive released handle.
- **Return**: void.
- **Key details**: لازم تتاخد بعد كل `reset_control_acquire` ناجح. مش بتعمل assert أو deassert تلقائي — الـ driver مسؤول عن ترتيب الـ state.
- **Caller context**: بعد انتهاء الاستخدام المؤقت للـ reset line.

---

### Group 2: الـ Bulk Runtime Operations

الـ functions دي بتشتغل على array من الـ `reset_control_bulk_data` وبتعمل نفس العمليات على كل الـ controls بالترتيب. مفيدة جداً للـ devices اللي عندها multiple reset lines (زي الـ PCIe، USB، Display subsystems).

---

#### `reset_control_bulk_reset`

```c
int reset_control_bulk_reset(int num_rstcs, struct reset_control_bulk_data *rstcs);
```

**بتعمل `reset_control_reset` على كل الـ controls في الـ array بالترتيب.** لو أي control فشل، بتوقف وترجع الـ error.

- **`num_rstcs`**: عدد الـ controls في الـ array.
- **`rstcs`**: pointer لـ array من `reset_control_bulk_data`.
- **Return**: `0` أو أول negative errno من أول control فشل.

---

#### `reset_control_bulk_assert` / `reset_control_bulk_deassert`

```c
int reset_control_bulk_assert(int num_rstcs, struct reset_control_bulk_data *rstcs);
int reset_control_bulk_deassert(int num_rstcs, struct reset_control_bulk_data *rstcs);
```

**بتعمل assert أو deassert على كل الـ controls.** الـ `bulk_assert` بيعمل assert بالـ reverse order عشان يحترم الـ power sequencing (الـ last asserted هو الـ first deasserted).

- **Return**: `0` أو negative errno مع rollback للـ controls اللي نجحت.
- **Key details**: الـ `bulk_deassert` لو فشلت في الوسط بترجع error بعد ما تعمل re-assert للـ controls اللي نجحت (error path cleanup).

---

#### `reset_control_bulk_acquire` / `reset_control_bulk_release`

```c
int reset_control_bulk_acquire(int num_rstcs, struct reset_control_bulk_data *rstcs);
void reset_control_bulk_release(int num_rstcs, struct reset_control_bulk_data *rstcs);
```

**bulk variants للـ acquire/release على exclusive released controls.**

---

### Group 3: الـ Core Get Functions (Low-level Internal)

الـ functions دي هي اللي بتعمل الـ actual lookup في الـ reset controller framework. كل الـ high-level `get_*` wrappers بترجع ليهم.

---

#### `__reset_control_get`

```c
struct reset_control *__reset_control_get(struct device *dev, const char *id,
                                          int index, enum reset_control_flags flags);
```

**الـ function الأساسية للـ lookup من device.** بتدور على الـ reset controller المرتبط بالـ device عن طريق الـ device tree أو platform data. بترجع `reset_control` handle جاهز للاستخدام.

- **`dev`**: الـ consumer device.
- **`id`**: اسم الـ reset line زي `"rst"` أو `"phy-reset"` — لو `NULL` بيستخدم index.
- **`index`**: الـ index في الـ `resets` property في الـ DT. يُستخدم لو `id == NULL`.
- **`flags`**: `enum reset_control_flags` يحدد exclusive/shared/optional/deasserted.
- **Return**: valid `reset_control*` أو `ERR_PTR(-errno)`. لو optional وما وجودش، ترجع `NULL`.
- **Key details**: الـ function دي protected بـ `mutex` داخلي. لو exclusive وفي حد تاني عامل get عليه، ترجع `ERR_PTR(-EBUSY)`.
- **Caller**: كل الـ `reset_control_get_*` wrappers.

**Pseudocode flow:**
```
__reset_control_get(dev, id, index, flags):
    optional = flags & BIT_OPTIONAL
    shared   = flags & BIT_SHARED

    rstc = of_reset_control_get_from_provider(dev->of_node, id, index)
    if !rstc:
        return optional ? NULL : ERR_PTR(-ENOENT)

    if !shared && rstc->exclusive_count > 0:
        return ERR_PTR(-EBUSY)

    if shared:
        rstc->shared_count++
    else:
        rstc->exclusive_count = 1

    if flags & BIT_DEASSERTED:
        reset_control_deassert(rstc)

    return rstc
```

---

#### `__of_reset_control_get`

```c
struct reset_control *__of_reset_control_get(struct device_node *node,
                                              const char *id, int index,
                                              enum reset_control_flags flags);
```

**نفس `__reset_control_get` لكن بتاخد `device_node` بدل `device`.** بتُستخدم في حالات زي الـ power domains وغيره اللي ما عندهاش `struct device` كامل.

- **`node`**: الـ device tree node.
- **`id`**, **`index`**, **`flags`**: نفس السابق.
- **Return**: `reset_control*` أو `ERR_PTR`.

---

#### `reset_control_put`

```c
void reset_control_put(struct reset_control *rstc);
```

**بتحرر الـ reset control handle وبتقلل الـ reference count.** لو الـ count وصل صفر، الـ core ممكن يحرر الـ resources الداخلية.

- **`rstc`**: الـ handle المراد تحريره. لو `NULL`، بترجع بدون عمل حاجة.
- **Return**: void.
- **Key details**: لازم تتسمى بعد `reset_control_get_*` مباشرةً. مش بتعمل assert تلقائي — الـ driver مسؤول عن الـ state قبل `put`.
- **Caller**: `remove` callback أو error path في `probe`.

---

#### `__reset_control_bulk_get`

```c
int __reset_control_bulk_get(struct device *dev, int num_rstcs,
                             struct reset_control_bulk_data *rstcs,
                             enum reset_control_flags flags);
```

**بتعمل `__reset_control_get` على كل عنصر في الـ `rstcs` array بالترتيب.** لو أي واحد فشل، بتعمل `reset_control_put` على كل اللي نجحوا قبله (complete rollback).

- **`num_rstcs`**: حجم الـ array.
- **`rstcs`**: array من `reset_control_bulk_data` — الـ `id` لازم يكون معبأ قبل الاستدعاء.
- **`flags`**: نفس `__reset_control_get`.
- **Return**: `0` أو negative errno مع complete rollback.

---

#### `reset_control_bulk_put`

```c
void reset_control_bulk_put(int num_rstcs, struct reset_control_bulk_data *rstcs);
```

**بتعمل `reset_control_put` على كل الـ controls في الـ array.**

---

### Group 4: الـ `devm_*` Resource-Managed Functions

الـ functions دي بتستخدم الـ devres framework عشان تعمل automatic cleanup لما الـ driver بيتفصل. مش محتاج تعمل `reset_control_put` يدوياً.

---

#### `__devm_reset_control_get`

```c
struct reset_control *__devm_reset_control_get(struct device *dev,
                                                const char *id, int index,
                                                enum reset_control_flags flags);
```

**بتعمل `__reset_control_get` وبتسجل cleanup callback في الـ devres list بتاع الـ device.** لما الـ device بيتفصل، الـ kernel بيعمل `reset_control_put` أوتوماتيك. الـ variants اللي فيها `_DEASSERTED` بتسجل assert callback كمان.

- **`dev`**, **`id`**, **`index`**, **`flags`**: نفس `__reset_control_get`.
- **Return**: valid handle أو `ERR_PTR`.
- **Key details**: الـ devres callback بيتسجل بـ `devm_add_action_or_reset`. لو الـ flags فيها `DEASSERTED`، الـ devm cleanup هيعمل `reset_control_assert` + `reset_control_put` عند الـ detach.
- **Caller**: `probe` callback بشكل حصري تقريباً.

---

#### `__devm_reset_control_bulk_get`

```c
int __devm_reset_control_bulk_get(struct device *dev, int num_rstcs,
                                  struct reset_control_bulk_data *rstcs,
                                  enum reset_control_flags flags);
```

**الـ devm variant للـ bulk get.** بتسجل `reset_control_bulk_put` كـ devres cleanup.

---

### Group 5: الـ High-Level Wrappers (Exclusive / Shared / Optional)

الـ functions دي مش بتضيف logic جديدة — كلها thin inlines بتوصل لـ `__reset_control_get` أو `__devm_reset_control_get` بـ flags مختلفة. الجدول ده بيوضح التحويل:

| Wrapper | Internal Call | Flags |
|---|---|---|
| `reset_control_get_exclusive` | `__reset_control_get` | `EXCLUSIVE` |
| `reset_control_get_exclusive_released` | `__reset_control_get` | `EXCLUSIVE_RELEASED` |
| `reset_control_get_shared` | `__reset_control_get` | `SHARED` |
| `reset_control_get_optional_exclusive` | `__reset_control_get` | `OPTIONAL_EXCLUSIVE` |
| `reset_control_get_optional_shared` | `__reset_control_get` | `OPTIONAL_SHARED` |
| `devm_reset_control_get_exclusive` | `__devm_reset_control_get` | `EXCLUSIVE` |
| `devm_reset_control_get_exclusive_deasserted` | `__devm_reset_control_get` | `EXCLUSIVE_DEASSERTED` |
| `devm_reset_control_get_shared` | `__devm_reset_control_get` | `SHARED` |
| `devm_reset_control_get_shared_deasserted` | `__devm_reset_control_get` | `SHARED_DEASSERTED` |
| `devm_reset_control_get_optional_exclusive` | `__devm_reset_control_get` | `OPTIONAL_EXCLUSIVE` |
| `devm_reset_control_get_optional_shared` | `__devm_reset_control_get` | `OPTIONAL_SHARED` |
| `of_reset_control_get_exclusive` | `__of_reset_control_get` | `EXCLUSIVE` |
| `of_reset_control_get_optional_exclusive` | `__of_reset_control_get` | `OPTIONAL_EXCLUSIVE` |
| `of_reset_control_get_shared` | `__of_reset_control_get` | `SHARED` |
| `of_reset_control_get_exclusive_by_index` | `__of_reset_control_get` | `EXCLUSIVE` + index |
| `of_reset_control_get_shared_by_index` | `__of_reset_control_get` | `SHARED` + index |

---

### Group 6: الـ Device Reset Helpers

---

#### `device_reset`

```c
static inline int __must_check device_reset(struct device *dev);
```

**أبسط طريقة لـ reset أي device — بتعمل get للـ reset الأول، reset، put.** الـ `__must_check` بتجبرك تتعامل مع الـ return value. داخلياً بتستدعي `__device_reset(dev, false)`.

- **`dev`**: الـ device المراد reset.
- **Return**: `0` أو negative errno (بما فيها `-ENOTSUPP` لو الـ CONFIG_RESET_CONTROLLER مش مفعل ومش optional).
- **Caller context**: early `probe`، power management، أو firmware loading.

---

#### `device_reset_optional`

```c
static inline int device_reset_optional(struct device *dev);
```

**زي `device_reset` لكن لو مفيش reset controller معرف، بترجع `0` بدل error.** مفيدة للـ devices اللي الـ reset فيها optional بطبيعتها.

- **Return**: `0` دايماً لو مفيش reset (optional)، أو `0` لو نجح الـ reset، أو negative errno لو فشل الـ reset.

---

#### `__device_reset`

```c
int __device_reset(struct device *dev, bool optional);
```

**الـ implementation الفعلية للـ device_reset.** بتعمل:
1. `reset_control_get` بالـ flags المناسبة.
2. `reset_control_reset`.
3. `reset_control_put`.

مش مفروض تتسمى مباشرة من الـ drivers — استخدم `device_reset` أو `device_reset_optional`.

---

### Group 7: الـ Array Get Functions

---

#### `devm_reset_control_array_get`

```c
struct reset_control *devm_reset_control_array_get(struct device *dev,
                                                    enum reset_control_flags flags);
```

**بتجيب كل الـ reset lines الموجودة في الـ `resets` property في الـ DT وبتلفها في handle واحد (array reset control).** مفيدة لما الـ device عنده reset lines كتير وعاوز تتعامل معاهم ككتلة واحدة.

- **`dev`**: الـ consumer device.
- **`flags`**: exclusive أو shared أو optional variants.
- **Return**: valid `reset_control*` (اللي بيمثل الـ array) أو `ERR_PTR`.
- **Key details**: الـ handle ده ممكن يتستخدم مع كل الـ runtime functions العادية (assert/deassert/reset) وكلها هتشتغل على كل الـ controls في الـ array. الـ devm cleanup هيعمل `reset_control_put` على الـ array handle.
- **Caller**: `probe` callback للـ devices اللي عندها multiple resets.

الـ wrappers بتاعتها:

| Wrapper | Flags |
|---|---|
| `devm_reset_control_array_get_exclusive` | `EXCLUSIVE` |
| `devm_reset_control_array_get_exclusive_released` | `EXCLUSIVE_RELEASED` |
| `devm_reset_control_array_get_shared` | `SHARED` |
| `devm_reset_control_array_get_optional_exclusive` | `OPTIONAL_EXCLUSIVE` |
| `devm_reset_control_array_get_optional_shared` | `OPTIONAL_SHARED` |

---

#### `of_reset_control_array_get`

```c
struct reset_control *of_reset_control_array_get(struct device_node *np,
                                                  enum reset_control_flags flags);
```

**نفس `devm_reset_control_array_get` لكن بتاخد `device_node` بدل `device` وبدون devm management.**

- **Return**: `reset_control*` أو `ERR_PTR`. الـ cleanup يدوي بـ `reset_control_put`.

---

#### `reset_control_get_count`

```c
int reset_control_get_count(struct device *dev);
```

**بترجع عدد الـ reset lines المرتبطة بالـ device في الـ device tree.** مفيدة لما الـ driver محتاج يعرف قبل ما يعمل allocation للـ bulk array.

- **`dev`**: الـ consumer device.
- **Return**: عدد الـ reset lines (`>= 0`) أو `-ENOENT` لو مفيش reset lines، أو `-ENOENT` لو `CONFIG_RESET_CONTROLLER` مش مفعل.

---

### Group 8: الـ Legacy / Transitional Wrappers

الـ functions دي موجودة للـ backward compatibility فقط وهيتاشالوا في المستقبل. مش تستخدمهم في كود جديد.

| Legacy Function | الـ Replacement الصح |
|---|---|
| `of_reset_control_get(node, id)` | `of_reset_control_get_exclusive(node, id)` |
| `of_reset_control_get_by_index(node, idx)` | `of_reset_control_get_exclusive_by_index(node, idx)` |
| `devm_reset_control_get(dev, id)` | `devm_reset_control_get_exclusive(dev, id)` |
| `devm_reset_control_get_optional(dev, id)` | `devm_reset_control_get_optional_exclusive(dev, id)` |
| `devm_reset_control_get_by_index(dev, idx)` | `devm_reset_control_get_exclusive_by_index(dev, idx)` |

---

### الـ `CONFIG_RESET_CONTROLLER` Stubs

لو الـ kernel اتبنى بدون `CONFIG_RESET_CONTROLLER`، كل الـ functions بترجع:
- `0` للـ optional variants
- `NULL` للـ optional struct pointers
- `ERR_PTR(-ENOTSUPP)` أو `-EOPNOTSUPP` للـ non-optional variants

ده بيخلي الـ driver code يشتغل بدون تغيير على kernels مش عندها reset support، بشرط إنه يتعامل صح مع الـ optional variants.

---

### مثال عملي: Driver Probe

```c
/* مثال لـ driver بيستخدم devm exclusive reset */
static int my_driver_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct reset_control *rst;
    int ret;

    /* الـ devm هيعمل assert + put تلقائي عند detach */
    rst = devm_reset_control_get_exclusive(dev, "core");
    if (IS_ERR(rst))
        return PTR_ERR(rst);

    /* إخراج الـ hardware من الـ reset */
    ret = reset_control_deassert(rst);
    if (ret)
        return ret;

    /* ... init the hardware ... */
    return 0;
}

/* مثال bulk devm لـ device عنده 3 reset lines */
static const char * const rst_names[] = { "axi", "ahb", "phy" };

static int my_complex_probe(struct platform_device *pdev)
{
    struct reset_control_bulk_data rsts[3];
    int i, ret;

    /* تعبئة الأسماء */
    for (i = 0; i < ARRAY_SIZE(rsts); i++)
        rsts[i].id = rst_names[i];

    ret = devm_reset_control_bulk_get_exclusive(&pdev->dev, 3, rsts);
    if (ret)
        return ret;

    return reset_control_bulk_deassert(3, rsts);
}
```
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

#### 1. debugfs — المدخلات المتاحة وطريقة قراءتها

الـ **Reset Controller framework** مش بيعمل debugfs entries افتراضية كتير، بس في طريقة تشوف الـ controllers المسجّلة عن طريق `/sys` (شوف القسم اللي بعده). لو الـ driver بيستخدم `debugfs` يدويًا، هتلاقيه تحت:

```bash
# افتراضي — تأكد إن debugfs متماونت
mount | grep debugfs
# لو مش موجود
mount -t debugfs none /sys/kernel/debug

# شوف كل حاجة تحت reset لو الـ driver عامل entries
ls /sys/kernel/debug/reset* 2>/dev/null || echo "No debugfs entries for reset"

# بعض SOC drivers بتعمل entries زي:
ls /sys/kernel/debug/clk/      # clk + reset غالبًا متجمعين في نفس driver
```

> الـ `reset_controller_list` هي linked list داخلية محمية بـ `reset_list_mutex`. مفيش طريقة مباشرة تقراها من userspace إلا عن طريق sysfs أو من خلال الـ driver نفسه لو عمل debugfs.

---

#### 2. sysfs — المدخلات المهمة

```bash
# كل device عندها reset controls بتظهر تحت
ls /sys/bus/platform/devices/<device-name>/

# معرفة الـ reset controls المربوطة بجهاز معين
cat /sys/bus/platform/devices/<device>/driver_override

# الـ reset controller نفسه بيظهر كـ device
ls /sys/bus/platform/devices/ | grep reset

# مثال عملي — Raspberry Pi / Allwinner / iMX
ls /sys/devices/platform/soc/*.reset-controller/

# فحص عدد الـ resets المتاحة من controller معين
# (يتعمل من خلال reset_control_get_count)
# لو الـ driver expose ده في sysfs attribute
cat /sys/devices/platform/<rcdev>/nr_resets 2>/dev/null
```

---

#### 3. ftrace — الـ Tracepoints والـ Events

```bash
# تفعيل tracing على كل دوال الـ reset subsystem
cd /sys/kernel/debug/tracing

# شوف الـ available events
grep -r "reset" available_events

# تفعيل function tracing للـ reset core
echo 'reset_control_reset' >> set_ftrace_filter
echo 'reset_control_assert' >> set_ftrace_filter
echo 'reset_control_deassert' >> set_ftrace_filter
echo '__reset_control_get' >> set_ftrace_filter
echo 'reset_control_put' >> set_ftrace_filter
echo function > current_tracer
echo 1 > tracing_on

# تشغيل الكود اللي بيعمل reset
cat trace | grep reset

# أو استخدم function_graph لتتبع call stack كامل
echo function_graph > current_tracer
echo 'reset_control_*' > set_graph_function
echo 1 > tracing_on
cat trace
```

مثال لـ output متوقع:

```
# tracer: function
#
    kworker/0:1-42    [000] ....   123.456789: reset_control_assert <-my_driver_suspend
    kworker/0:1-42    [000] ....   123.456800: reset_control_deassert <-my_driver_resume
```

---

#### 4. printk والـ Dynamic Debug

```bash
# تفعيل dynamic debug لكل الـ reset framework
echo 'file drivers/reset/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# تفعيل لكل الـ reset drivers دفعة واحدة
echo 'file drivers/reset/* +p' > /sys/kernel/debug/dynamic_debug/control

# تفعيل لـ driver معين بالاسم
echo 'file drivers/reset/reset-imx7.c +p' > /sys/kernel/debug/dynamic_debug/control

# قراءة الـ log
dmesg | grep -i reset
journalctl -k | grep -i "reset"

# رفع loglevel مؤقتًا
echo 8 > /proc/sys/kernel/printk
```

لو محتاج تضيف `pr_debug()` في الكود:

```c
/* في drivers/reset/core.c — في reset_control_assert() مثلًا */
pr_debug("reset_control_assert: rcdev=%s id=%u\n",
         rcdev_name(rstc->rcdev), rstc->id);
```

---

#### 5. Kernel Config Options للـ Debugging

| Config | الوصف |
|--------|-------|
| `CONFIG_RESET_CONTROLLER` | يفعّل الـ reset framework أصلًا |
| `CONFIG_DEBUG_KERNEL` | يفعّل كل خيارات الـ debug العامة |
| `CONFIG_DYNAMIC_DEBUG` | يتيح `+p` على `pr_debug()` |
| `CONFIG_FTRACE` | يفعّل function tracer |
| `CONFIG_FUNCTION_TRACER` | تتبع كل function calls |
| `CONFIG_FUNCTION_GRAPH_TRACER` | رسم call graph |
| `CONFIG_LOCKDEP` | كشف deadlocks في الـ mutex اللي بيحمي `reset_list_mutex` |
| `CONFIG_PROVE_LOCKING` | تحقق من correctness الـ locking |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | كشف نوم في atomic context (مهم لأن `reset_control_assert` ممكن يأخد mutex) |
| `CONFIG_KASAN` | كشف memory corruption في الـ reset_control structs |
| `CONFIG_DEBUG_OBJECTS` | تتبع lifetime الـ objects |

```bash
# فحص الـ config الحالي
zcat /proc/config.gz | grep -E "CONFIG_RESET|CONFIG_DEBUG_KERNEL|CONFIG_DYNAMIC_DEBUG"
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# أداة رئيسية: lsreset (مش موجودة في mainline، بس ممكن تعملها بنفسك)
# البديل: استخدم /proc/device-tree وقارن بالـ sysfs

# فحص الـ reset lines المعرّفة في الـ DT لجهاز معين
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "resets"

# رؤية كل platform devices وربطها بالـ reset controllers
ls -la /sys/bus/platform/drivers/

# لو الـ SoC بيستخدم clock+reset متجمعين (زي Rockchip CRU)
ls /sys/kernel/debug/clk/

# فحص GPIO-based reset controllers
ls /sys/kernel/debug/gpio

# استخدام libgpiod لـ GPIO reset يدوي (للاختبار فقط)
gpioset gpiochip0 5=0   # assert reset (active low)
sleep 0.01
gpioset gpiochip0 5=1   # deassert reset
```

---

#### 7. جدول رسائل الـ Errors الشائعة

| رسالة الـ Error | المعنى | الحل |
|----------------|--------|------|
| `reset_control_get: failed to get reset 'xxx'` | الاسم المطلوب مش موجود في الـ DT | تأكد إن `resets` و`reset-names` في الـ DTS صح |
| `-EBUSY: reset control already in use` | عمل `get_exclusive` أكتر من مرة على نفس الـ reset line | استخدم `get_shared` لو أكتر من driver بيشاركوا نفس الـ reset |
| `-ENOTSUPP` | `CONFIG_RESET_CONTROLLER` مش مفعّل | فعّل الـ config وأعد البناء |
| `-ENOENT` | الـ reset controller اللي بيشاور عليه الـ DT مش مسجّل | الـ driver الخاص بالـ controller لسا مش probe |
| `-EINVAL` | الـ index أو الـ id في الـ DT غلط | راجع `#reset-cells` في الـ DTS |
| `reset_control_assert called on shared reset` | خطأ في الاستخدام — shared reset يحتاج deassert أولًا | ابدأ بـ `deassert` قبل `assert` في shared mode |
| `reset_control_reset called on shared reset` | `reset_control_reset` ممنوع على shared resets | استخدم `assert` + `deassert` يدويًا |
| `reset: -ETIMEDOUT` | الـ hardware مأخدتش تخرج من الـ reset في الوقت المحدد | مشكلة hardware أو clock مش شغال للـ peripheral |
| `reset: failed to lookup reset` | الـ phandle في الـ DT بيشاور على node مش موجود | تحقق من صحة الـ phandle في الـ DTS |
| `-EACCES: reset control not acquired` | بتحاول تعمل `assert/deassert` على `EXCLUSIVE_RELEASED` قبل `acquire` | اعمل `reset_control_acquire()` الأول |

---

#### 8. أماكن استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في __reset_control_get() — لو رجعت NULL أو ERR لـ non-optional */
static struct reset_control *__reset_control_get(struct device *dev,
    const char *id, int index, enum reset_control_flags flags)
{
    struct reset_control *rstc;

    rstc = __reset_control_get_internal(dev, id, index, flags);

    /* نقطة استراتيجية: الـ non-optional مش المفروض ترجع error */
    if (!(flags & RESET_CONTROL_FLAGS_BIT_OPTIONAL))
        WARN_ON(IS_ERR_OR_NULL(rstc));

    return rstc;
}

/* في reset_control_assert() — كشف double-assert */
int reset_control_assert(struct reset_control *rstc)
{
    /* تحذير لو شخص بيعمل assert على shared قبل ما يعمل deassert */
    WARN_ON(rstc->shared &&
            atomic_read(&rstc->deassert_count) == 0);
    /* ... */
}

/* في reset_control_deassert() — كشف imbalance */
int reset_control_deassert(struct reset_control *rstc)
{
    /* dump stack لو الـ deassert_count وصل لـ overflow */
    if (WARN_ON(atomic_read(&rstc->deassert_count) < 0)) {
        dump_stack();
        return -EINVAL;
    }
    /* ... */
}

/* في reset_control_put() — كشف use-after-free */
void reset_control_put(struct reset_control *rstc)
{
    WARN_ON(IS_ERR_OR_NULL(rstc));
    /* ... */
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware متطابقة مع حالة الـ Kernel

```bash
# الطريقة الأساسية: قارن قيمة الـ register مع الـ deassert_count

# مثال على SoC زي iMX8 أو Rockchip:
# 1. اقرأ الـ register اليدوي (شوف القسم التالي)
# 2. قارنه مع الـ kernel state

# من داخل kernel (في driver للاختبار):
int status = reset_control_status(rstc);
pr_info("reset line status: %d (0=asserted, 1=deasserted)\n", status);

# الـ status() callback بيرجع القيمة الحقيقية من الـ hardware register
```

---

#### 2. Register Dump — قراءة الـ Hardware Registers

```bash
# أداة devmem2 (لازم تكون مثبّتة)
# مثال: Rockchip RK3399 CRU reset register عند 0xFF760000
devmem2 0xFF760000 w    # قرأ word من الـ reset register base
devmem2 0xFF760400 w    # SOFTRST_CON0

# أداة /dev/mem مباشرة بـ Python
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    mm = mmap.mmap(f.fileno(), 4096, offset=0xFF760000)
    val = struct.unpack('<I', mm.read(4))[0]
    print(f'Reset register: 0x{val:08X}')
    mm.close()
"

# أداة io (من package io-compat)
io -4 0xFF760400    # قرأ 32-bit register

# Allwinner H3 — Reset Register لـ USB
devmem2 0x01C202D0 w

# iMX6 — SRC (System Reset Controller)
devmem2 0x020D8000 w   # SRC_SCR
devmem2 0x020D8008 w   # SRC_SBMR1

# تفسير القيمة:
# لو bit=1 → deasserted (device شغال)
# لو bit=0 → asserted (device في reset)
# (بيفرق من SoC لـ SoC — شوف الـ datasheet)
```

---

#### 3. Logic Analyzer / Oscilloscope

```
نقاط القياس المهمة:
┌─────────────────────────────────────────────────┐
│  SoC                    Device                  │
│  ┌──────┐  RESET_N     ┌──────────┐             │
│  │Reset │──────────────│ RESET_N  │             │
│  │Ctrl  │  (active low)│ Pin      │             │
│  └──────┘              └──────────┘             │
│                                                 │
│  قس هنا ↑                                       │
└─────────────────────────────────────────────────┘

نصائح للـ Logic Analyzer:
- Sample rate: 10x فوق أسرع تردد متوقع للـ reset pulse
- الـ reset pulse المعتادة: 1µs → 100ms حسب الـ device
- اتحقق من:
  * عرض الـ pulse (بعض devices بتحتاج minimum assert time)
  * الـ polarity (active low vs active high)
  * الـ glitches خلال الـ power-up sequence

نصائح للـ Oscilloscope:
- استخدم 10:1 probe لتقليل الـ capacitive loading
- قس الـ rise/fall time — لازم يكون أسرع من 1µs في معظم الحالات
- راقب voltage level: 1.8V أو 3.3V حسب الـ I/O standard
- اعمل trigger على الـ falling edge (لـ active-low reset)
```

---

#### 4. Common Hardware Issues وأنماط الـ Kernel Log

| المشكلة | نمط الـ Log | الأسباب المحتملة |
|---------|------------|-----------------|
| Reset line عالقة asserted | `device timeout waiting for reset` | GPIO مش موصّل / pull-up ناقص / hardware damage |
| Reset pulse قصيرة جدًا | `device not responding after reset` | Minimum assert time مش اتحقق |
| Power sequencing خاطئ | `reset: -ETIMEDOUT` بعد power-on | لازم power يستقر قبل deassert الـ reset |
| Shared reset مش متزامن | `deassert_count underflow` + `WARN_ON` | Driver تاني عمل `assert` من غير coordination |
| Clock مش شغال قبل الـ reset | Device مش بيشتغل بعد deassert | لازم تفعّل الـ clock قبل deassert الـ reset |
| Voltage غلط على RESET_N | Glitches في بداية التشغيل | فحص الـ power supply بـ oscilloscope |

```bash
# فلترة رسائل الـ reset في الـ kernel log
dmesg | grep -iE "reset|rstc|rcdev|ETIMEDOUT" | head -50

# تتبع الـ probe sequence
dmesg | grep -E "probe|reset" | grep -v "grep"
```

---

#### 5. Device Tree Debugging

```bash
# تحقق إن الـ DT تحمّل صح
dtc -I fs -O dts /proc/device-tree 2>/dev/null | grep -B5 -A10 "resets"

# فحص phandle يشاور على reset controller صح
# مثال DTS صح:
# uart0: serial@ff000000 {
#     resets = <&cru SRST_UART0>;     ← phandle صح
#     reset-names = "uart";           ← اسم يطابق ما يطلبه الـ driver
# };

# تحقق من الـ reset controller node نفسه
dtc -I fs -O dts /proc/device-tree 2>/dev/null | grep -B2 -A20 "reset-controller"

# فحص #reset-cells
cat /proc/device-tree/<reset-controller-node>/#reset-cells | xxd
# لازم يطابق عدد الـ args في الـ consumer (عادةً 1)

# تحقق من of_xlate
# لو #reset-cells = <1> ← consumer يكتب <&rcdev RESET_ID>
# لو #reset-cells = <2> ← consumer يكتب <&rcdev MODULE RESET_BIT>

# أداة fdtget للاستعلام السريع
fdtget /boot/dtb/<your>.dtb /soc/uart@ff000000 resets

# مقارنة الـ DTS مع الـ kernel state
ls /sys/firmware/devicetree/base/
```

---

### Practical Commands — أوامر جاهزة للنسخ

#### فحص سريع للـ Reset Subsystem

```bash
#!/bin/bash
# quick-reset-debug.sh — فحص سريع لحالة الـ reset subsystem

echo "=== Reset Controllers في الـ sysfs ==="
find /sys/devices -name "*.reset-controller" -o -name "*-reset" 2>/dev/null | head -20

echo ""
echo "=== Device Tree Reset Nodes ==="
dtc -I fs -O dts /proc/device-tree 2>/dev/null | \
    grep -E "(reset-controller|resets =|reset-names)" | head -30

echo ""
echo "=== Kernel Log للـ Reset ==="
dmesg | grep -iE "\breset\b" | tail -20

echo ""
echo "=== Config الـ Reset ==="
zcat /proc/config.gz 2>/dev/null | grep -E "CONFIG_RESET" || \
    cat /boot/config-$(uname -r) 2>/dev/null | grep -E "CONFIG_RESET"
```

#### تفعيل الـ Dynamic Debug للـ Reset Framework

```bash
# تفعيل كل الـ debug messages في reset/core.c
echo 'file drivers/reset/core.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# التحقق من التفعيل
grep "reset/core" /sys/kernel/debug/dynamic_debug/control

# تعطيل
echo 'file drivers/reset/core.c -p' > /sys/kernel/debug/dynamic_debug/control
```

#### تتبع reset_control_assert/deassert بـ ftrace

```bash
#!/bin/bash
# ftrace-reset.sh

TRACE=/sys/kernel/debug/tracing

echo 0 > $TRACE/tracing_on
echo > $TRACE/trace

# اختار function_graph للتفاصيل
echo function_graph > $TRACE/current_tracer

# فلتر على دوال الـ reset فقط
echo 'reset_control_assert
reset_control_deassert
reset_control_reset
reset_control_acquire
reset_control_release
__reset_control_get
reset_control_put' > $TRACE/set_graph_function

echo 1 > $TRACE/tracing_on
echo "Tracing enabled — run your test now, then press Enter"
read
echo 0 > $TRACE/tracing_on

cat $TRACE/trace | grep -v "^#" | head -100
```

**مثال لـ output متوقع:**

```
 1)               |  reset_control_deassert() {
 1)               |    __reset_control_deassert() {
 1)   0.312 us    |      atomic_add_return();
 1)   1.204 us    |      rockchip_reset_deassert();  /* actual HW write */
 1)   2.891 us    |    }
 1)   3.102 us    |  }
```

#### فحص حالة reset_control بـ gdb على kernel live

```bash
# باستخدام crash tool أو kgdb
# ابحث عن الـ reset_controller_list
crash> list reset_controller_list -s reset_controller_dev.dev,nr_resets,ops

# أو بـ /proc/kallsyms + devmem2
grep "reset_controller_list" /proc/kallsyms
# ثم اقرأ العنوان بـ devmem2 (على نظام بدون KASLR)
```

#### فحص register يدوي لـ SoC شهير

```bash
# Rockchip RK3399 — SOFTRST_CON registers (CRU base = 0xFF760000)
for i in $(seq 0 21); do
    offset=$(printf "%X" $((i * 4)))
    addr=$(printf "0x%X" $((0xFF760400 + i * 4)))
    val=$(devmem2 $addr w 2>/dev/null | grep "Read at" | awk '{print $NF}')
    echo "SOFTRST_CON$i ($addr): $val"
done

# iMX8MQ — SRC registers (base = 0x30390000)
devmem2 0x30390000 w   # SRC_SCR
devmem2 0x30390004 w   # SRC_A53RCR0
devmem2 0x30390008 w   # SRC_A53RCR1

# Allwinner H6 — Reset registers
devmem2 0x03001000 w   # MBUS Reset
devmem2 0x0300168C w   # USB Reset
```

#### اختبار reset يدوي من userspace (للاختبار فقط)

```bash
# باستخدام debugfs lو الـ driver فتح interface
# أو عن طريق GPIO مباشرة لـ GPIO-based reset controllers

# مثال: assert reset عن طريق GPIO
echo 5 > /sys/class/gpio/export       # export GPIO 5
echo out > /sys/class/gpio/gpio5/direction
echo 0 > /sys/class/gpio/gpio5/value  # assert (active low)
sleep 0.01                             # 10ms reset pulse
echo 1 > /sys/class/gpio/gpio5/value  # deassert
```

---

> **ملاحظة مهمة:** الـ `reset_control_reset()` على shared reset ممنوع ومبيشتغلش — لو شايف `-EBUSY` أو `-EINVAL` على shared reset، اتأكد إنك بتستخدم `assert`/`deassert` يدويًا وليس `reset` مباشرة. وكمان، الـ `deassert_count` في `struct reset_control` هو atomic counter — لو وصل صفر مع وجود `assert` إضافية، هيتعمل real hardware assert.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: RK3562 — Industrial Gateway — الـ UART مش شغال بعد الـ probe

#### العنوان
الـ UART driver بيفشل على RK3562 بسبب missing reset في الـ DT

#### السياق
شركة بتعمل industrial gateway بالـ RK3562. الجهاز بيشغّل Linux مع custom BSP. الـ UART2 مربوط بـ RS-485 transceiver. بعد كل reboot، الـ UART بيجي dead — مفيش output، مفيش interrupt، حتى loopback test بيفشل.

#### المشكلة
الـ driver بيكمل الـ probe بدون error، بس الـ UART hardware نفسه واقف في reset state. الـ /dev/ttyS2 موجود بس أي write عليه بيختفي في الفراغ.

#### التحليل
الـ engineer بيفتح driver الـ UART لـ Rockchip ولاقى:

```c
/* في drivers/tty/serial/8250/8250_dw.c */
static int dw8250_probe(struct platform_device *pdev)
{
    struct dw8250_data *data;
    ...
    /* driver بيستخدم reset_control_get_optional_exclusive */
    data->rst = devm_reset_control_get_optional_exclusive(&pdev->dev, NULL);
    if (IS_ERR(data->rst))
        return PTR_ERR(data->rst);

    reset_control_deassert(data->rst);  /* لو rst = NULL مش هيحصل حاجة */
    ...
}
```

**الـ reset_control_get_optional_exclusive** بيكال في النهاية:

```c
/* reset.h — السطر 427 */
static inline struct reset_control *reset_control_get_optional_exclusive(
                    struct device *dev, const char *id)
{
    return __reset_control_get(dev, id, 0, RESET_CONTROL_OPTIONAL_EXCLUSIVE);
}
```

الـ flag هنا هو `RESET_CONTROL_OPTIONAL_EXCLUSIVE` اللي قيمته:

```c
/* reset.h — السطر 58 */
RESET_CONTROL_OPTIONAL_EXCLUSIVE = RESET_CONTROL_FLAGS_BIT_OPTIONAL |
                                   RESET_CONTROL_FLAGS_BIT_ACQUIRED,
```

لأن الـ `OPTIONAL` flag موجود، لو الـ DT مش فيه `resets` property، الدالة بترجع `NULL` بدل error. الـ driver بيشوف `NULL` ومبيعملش `deassert`، والـ hardware فاضل في reset.

الـ DT الموجود:

```dts
/* rk3562-gateway.dts — WRONG */
&uart2 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&uart2m1_xfer>;
    /* resets property مش موجود خالص! */
};
```

لكن الـ RK3562 CRU فيه reset line للـ UART. السبب إن الـ board bringup engineer نسي يضيفها.

#### الحل

**الخطوة 1:** تأكد من الـ reset ID في CRU documentation أو kernel source:

```bash
grep -r "SRST_UART2" arch/arm64/boot/dts/rockchip/rk3562.dtsi
```

**الخطوة 2:** صلّح الـ DT:

```dts
/* rk3562-gateway.dts — FIXED */
#include <dt-bindings/reset/rockchip,rk3562-cru.h>

&uart2 {
    status = "okay";
    pinctrl-names = "default";
    pinctrl-0 = <&uart2m1_xfer>;
    resets = <&cru SRST_UART2>;       /* أضف reset line */
    reset-names = "uart";
};
```

**الخطوة 3:** تحقق بعد الـ boot:

```bash
# شوف لو الـ reset controller بيشوفه
cat /sys/kernel/debug/reset/uart2/

# اختبر الـ UART
echo "test" > /dev/ttyS2
stty -F /dev/ttyS2 115200 raw && cat /dev/ttyS2 &
echo "hello" > /dev/ttyS2
```

#### الدرس المستفاد
الـ `_optional_` variants في الـ reset API (زي `reset_control_get_optional_exclusive`) صمّمت تمنع الـ driver من الـ fail لو مفيش reset في الـ DT — بس ده بالضبط ممكن يخبّي مشكلة hardware حقيقية. دايمًا زوّد الـ resets property في الـ DT حتى لو الـ driver مش بيرفض بدونها.

---

### السيناريو 2: STM32MP1 — IoT Sensor Hub — الـ SPI hanging بعد error recovery

#### العنوان
الـ shared reset على STM32MP1 بيخلي الـ SPI يتعلق بعد sensor fault

#### السياق
product بيجمع data من 4 SPI sensors على STM32MP1. الـ SPI1 bus مشترك بين chip وبين peripheral DMA engine. كل sensor بيعمل fault بيتعمل له software reset. المشكلة: بعد الـ reset، أحيانًا الـ SPI bus بيتعلق تمامًا ومحتاج hard reboot.

#### المشكلة
اتنين drivers — الـ SPI master driver والـ DMA engine driver — كلاهما بيعمل `reset_control_assert` على نفس الـ reset line بدون تنسيق. الـ shared reset semantics بتتكسر.

#### التحليل
الـ SPI driver بيستخدم exclusive reset:

```c
/* drivers/spi/spi-stm32.c — simplified */
static int stm32_spi_probe(struct platform_device *pdev)
{
    spi->rst = devm_reset_control_get_exclusive(&pdev->dev, NULL);
    ...
}

static void stm32_spi_handle_err(struct spi_master *master)
{
    /* assert ثم deassert لعمل reset */
    reset_control_assert(spi->rst);
    udelay(2);
    reset_control_deassert(spi->rst);
}
```

الـ DMA driver كمان بيعمل نفس الحاجة على نفس الـ line. الـ exclusive ownership بيعني إن أي حد يحاول يأخد الـ handle تاني هياخد `-EBUSY`.

لكن المشكلة الأعمق: لو الـ DMA driver عمل `assert` وبعدين الـ SPI driver عمل `deassert` من غير ما يعرف — الـ hardware اترسّت في الوقت الغلط.

الحل الصح هو الـ **shared reset**. الـ reset.h بيشرح السلوك:

```c
/*
 * reset.h — السطر 382-389:
 * When a reset-control is shared, the reset-core will keep track of a
 * deassert_count and only (re-)assert the reset after reset_control_assert
 * has been called as many times as reset_control_deassert was called.
 */
static inline struct reset_control *reset_control_get_shared(
                    struct device *dev, const char *id)
{
    return __reset_control_get(dev, id, 0, RESET_CONTROL_SHARED);
}
```

الـ `deassert_count` internal بيضمن إن الـ reset مش هيتعمل assert غصب عن اللي عنده reference.

#### الحل

**صلّح كلا الـ drivers يستخدموا shared reset:**

```c
/* في كلا الـ SPI و DMA drivers */
rstc = devm_reset_control_get_shared(&pdev->dev, "bus-reset");
if (IS_ERR(rstc))
    return PTR_ERR(rstc);
```

**الـ DT:**

```dts
/* stm32mp15-iot-hub.dts */
&spi1 {
    resets = <&rcc SPI1_R>;
    reset-names = "bus-reset";
    ...
};

/* DMA node يشير لنفس الـ reset */
&dma1 {
    resets = <&rcc SPI1_R>;
    reset-names = "bus-reset";
    ...
};
```

**تحقق من الـ deassert_count:**

```bash
# راقب الـ reset state
cat /sys/kernel/debug/reset/spi1/

# simulate fault ثم شوف لو الـ bus شغال
echo 1 > /sys/kernel/debug/reset/spi1/reset
sleep 1
spi-test -D /dev/spidev0.0 -p "hello"
```

#### الدرس المستفاد
الـ `RESET_CONTROL_SHARED` مش مجرد convenience — هو ضروري لأي reset line مشتركة بين أكتر من driver. الـ `deassert_count` بيحمي من race conditions. استخدام `EXCLUSIVE` على reset مشترك بيتسبب في `-EBUSY` أو أسوأ: reset غير متوقع.

---

### السيناريو 3: i.MX8MQ — Android TV Box — الـ HDMI مش بيظهر على بعض الـ TVs

#### العنوان
الـ HDMI controller على i.MX8MQ بيحتاج explicit reset sequence مش مجرد deassert

#### السياق
Android TV box بالـ i.MX8MQ. 95% من الـ TVs شغالة كويس، بس بعض الـ TVs القديمة مبتشتغلش. الـ HDMI signal موجود (بيظهر على oscilloscope) بس الـ TV مش بتعمل lock.

#### المشكلة
الـ HDMI PHY محتاج full pulse reset — يعني assert ثم deassert — مش بس deassert من initial state. بعض الـ TVs حساسة لـ timing الـ PHY initialization. الـ driver بيستخدم `reset_control_deassert` بس بدون assert أول.

#### التحليل
الـ driver الحالي:

```c
/* drivers/gpu/drm/imx/imx-hdmi.c — simplified buggy version */
static int imx_hdmi_bind(struct device *dev, ...)
{
    hdmi->reset = devm_reset_control_get_exclusive(dev, "hdmi");
    if (IS_ERR(hdmi->reset))
        return PTR_ERR(hdmi->reset);

    /* بس deassert بدون assert الأول! */
    reset_control_deassert(hdmi->reset);
    ...
}
```

المشكلة إن لو الـ PHY كان في unknown state من boot (مش assertsed بالضرورة)، الـ `deassert` مش بيعمل الـ clean reset pulse اللي الـ PHY محتاجه.

الـ API الصح هو `reset_control_reset`:

```c
/* reset.h — السطر 73 */
int reset_control_reset(struct reset_control *rstc);
/*
 * بيعمل assert ثم deassert automatically —
 * يعني guaranteed clean pulse
 * مش مسموح على shared resets
 */
```

أو يعملها manually:

```c
reset_control_assert(hdmi->reset);
usleep_range(10, 20);   /* minimum assert time حسب datasheet */
reset_control_deassert(hdmi->reset);
usleep_range(100, 200); /* stabilization time */
```

لو الـ engineer مش عارف لو الـ reset ده shared ولا لا، يستخدم `reset_control_get_count`:

```c
/* reset.h — السطر 108 */
int reset_control_get_count(struct device *dev);
```

ولو محتاج يعمل reset على array من الـ resets:

```c
/* reset.h — السطر 81 */
int reset_control_bulk_reset(int num_rstcs,
                             struct reset_control_bulk_data *rstcs);
```

#### الحل

```c
/* imx-hdmi.c — FIXED probe sequence */
static int imx_hdmi_bind(struct device *dev, ...)
{
    hdmi->reset = devm_reset_control_get_exclusive(dev, "hdmi");
    if (IS_ERR(hdmi->reset))
        return PTR_ERR(hdmi->reset);

    /* clean pulse reset للـ PHY */
    ret = reset_control_reset(hdmi->reset);
    if (ret) {
        dev_err(dev, "failed to reset HDMI PHY: %d\n", ret);
        return ret;
    }

    /* انتظر الـ PHY stabilization */
    usleep_range(500, 1000);
    ...
}
```

**DT verification:**

```dts
/* imx8mq-tv-box.dts */
&hdmi {
    resets = <&src IMX8MQ_RESET_HDMI_PHY>;
    reset-names = "hdmi";
};
```

```bash
# debug: شوف الـ reset state
cat /sys/kernel/debug/reset/hdmi_phy/

# force reset من userspace للتجربة
echo 1 > /sys/class/drm/card0/device/reset  # لو موجود
```

#### الدرس المستفاد
`reset_control_deassert` وحدها مش دايمًا كفاية. الـ PHYs والـ hardware blocks الحساسة محتاجة clean reset pulse. استخدم `reset_control_reset` لضمان assert→deassert cycle — وراجع الـ datasheet عشان تعرف الـ minimum assert time.

---

### السيناريو 4: AM62x — Automotive ECU — الـ driver بيفشل بسبب `-EBUSY` على reset exclusive

#### العنوان
الـ CAN controller driver على AM62x بيرجع `-EBUSY` عند الـ probe بسبب exclusive reset conflict

#### السياق
Automotive ECU بالـ AM62x (TI Sitara). الـ system بيشغّل 2 instances من CAN controller (MCAN0 و MCAN1). الـ MCAN0 شغال تمام، بس الـ MCAN1 بيفشل في الـ probe بـ `-EBUSY`. الـ dmesg بيقول:

```
mcan 20738000.can: Failed to get reset: -16
```

`-16` هو `-EBUSY`.

#### المشكلة
الـ DTB فيه error: الـ MCAN0 والـ MCAN1 عندهم نفس الـ reset specifier في الـ DT بدل ما كل واحد يبقى له reset منفصل. الـ MCAN0 بياخد الـ exclusive lock على الـ reset، وبعدين الـ MCAN1 بيلاقيه مأخوذ.

#### التحليل
الـ driver بيستخدم:

```c
/* drivers/net/can/m_can/m_can_platform.c */
static int m_can_plat_probe(struct platform_device *pdev)
{
    mcan_class->reset = devm_reset_control_get_exclusive(&pdev->dev, NULL);
    if (IS_ERR(mcan_class->reset)) {
        if (PTR_ERR(mcan_class->reset) == -EPROBE_DEFER)
            return -EPROBE_DEFER;
        /* غيره: treat as optional */
        mcan_class->reset = NULL;
    }
    ...
}
```

لكن الـ error هنا `-EBUSY` مش `-EPROBE_DEFER`، فالـ driver بيتجاهله ويكمل بـ `reset = NULL`.

لو الـ AM62x DT:

```dts
/* am62x-ecu.dts — WRONG */
&mcan0 {
    resets = <&k3_reset 0x40>;   /* MCAN reset */
    status = "okay";
};

&mcan1 {
    resets = <&k3_reset 0x40>;   /* نفس الـ reset! خطأ */
    status = "okay";
};
```

الـ `reset_control_get_exclusive` بيكال من الـ `__reset_control_get`:

```c
/* reset.h — السطر 89 */
struct reset_control *__reset_control_get(struct device *dev, const char *id,
                      int index, enum reset_control_flags flags);
```

الـ flag هنا `RESET_CONTROL_EXCLUSIVE` = `RESET_CONTROL_FLAGS_BIT_ACQUIRED`. الـ reset core بيشوف إن الـ reset ده عنده بالفعل exclusive owner (الـ MCAN0)، فبيرجع `-EBUSY`.

للتشخيص:

```bash
# شوف مين ماسك الـ reset
cat /sys/kernel/debug/reset/k3_reset_0x40/

# أو من kernel log
dmesg | grep -i "reset\|mcan\|ebusy"
```

#### الحل

**الحل الصح:** كل MCAN instance محتاج reset line مستقلة في الـ DT:

```dts
/* am62x-ecu.dts — FIXED */
&mcan0 {
    resets = <&k3_reset 0x40>;   /* MCAN0 reset */
    reset-names = "mcan";
    status = "okay";
};

&mcan1 {
    resets = <&k3_reset 0x41>;   /* MCAN1 reset — مختلف! */
    reset-names = "mcan";
    status = "okay";
};
```

**لو فعلًا الـ hardware بيشارك نفس الـ reset line** (بعض SoCs كده)، استخدم shared:

```c
/* في الـ driver */
mcan_class->reset = devm_reset_control_get_shared(&pdev->dev, "mcan");
```

```dts
/* وفي الـ DT نفس الـ specifier على الاتنين — shared مقبول */
&mcan0 {
    resets = <&k3_reset 0x40>;
};
&mcan1 {
    resets = <&k3_reset 0x40>;  /* shared — OK لو driver بيستخدم get_shared */
};
```

#### الدرس المستفاد
الـ `-EBUSY` من الـ reset API معناها exclusive conflict — مش hardware fault. دايمًا تحقق إن كل instance عنده reset line مستقلة في الـ DT. لو الـ hardware فعلًا بيشارك reset line، الـ driver لازم يستخدم `reset_control_get_shared` مش `get_exclusive`.

---

### السيناريو 5: Allwinner H616 — Custom Board Bring-up — USB مش بيشتغل وكل الـ resets Optional

#### العنوان
الـ USB3 controller على Allwinner H616 فاشل في الـ bring-up بسبب optional resets مخبّية الـ مشكلة

#### السياق
Team بيعمل custom board بالـ Allwinner H616 (Orange Pi). الـ USB3 port مش بيشتغل. الـ kernel بيشغّل بدون errors في الـ dmesg. الـ lsusb مش بيشوف devices. الـ team فاكر إن المشكلة hardware.

#### المشكلة
الـ USB3 driver بيستخدم `devm_reset_control_get_optional_exclusive` لكل الـ resets. الـ DT مفيهوش أي `resets` properties للـ USB3 node. كل الـ reset handles بتبقى `NULL`. الـ USB3 PHY فاضل في reset state، والـ driver مش بيعرف.

#### التحليل
Driver الـ USB3 (dwc3):

```c
/* drivers/usb/dwc3/dwc3-of-simple.c — simplified */
static int dwc3_of_simple_probe(struct platform_device *pdev)
{
    struct dwc3_of_simple *simple;
    ...
    /* كل الـ resets optional */
    ret = devm_reset_control_bulk_get_optional_exclusive(dev,
                                    simple->num_resets,
                                    simple->resets);
    if (ret)
        return ret;

    /* لو كلهم NULL مش هيحصل حاجة */
    reset_control_bulk_deassert(simple->num_resets, simple->resets);
    ...
}
```

الـ `reset_control_bulk_deassert`:

```c
/* reset.h — السطر 83 */
int reset_control_bulk_deassert(int num_rstcs,
                                struct reset_control_bulk_data *rstcs);
```

الـ bulk API بيلف على كل element في الـ array. لو `rstc` = `NULL`، الـ reset core بيتجاوزه. النتيجة: الـ USB3 PHY والـ controller فاضلين في reset والـ driver مش عارف.

للكشف، الـ engineer يستخدم:

```bash
# شوف عدد الـ resets اللي الـ device بيشوفها
cat /sys/kernel/debug/reset/

# شوف الـ USB3 node في الـ DT
dtc -I fs /sys/firmware/devicetree/base | grep -A 20 "usb@"

# أو من userspace
ls /sys/bus/platform/devices/5200000.usb/
```

الـ `reset_control_get_count` ممكن يساعد في الـ debug:

```c
/* reset.h — السطر 108 */
int reset_control_get_count(struct device *dev);
/* بيرجع عدد الـ resets في الـ DT لهذا الـ device */
```

لو بيرجع 0 أو `-ENOENT`، الـ DT فعلًا مش فيه resets.

الـ DT الناقص:

```dts
/* h616-custom.dts — INCOMPLETE */
&usb3_0 {
    status = "okay";
    /* مفيش resets! */
};
```

الـ H616 DT الصح (من Orange Pi kernel):

```dts
/* sun50i-h616.dtsi — reference */
usb3_0: usb@5200000 {
    compatible = "allwinner,sun50i-h616-dwc3";
    resets = <&ccu RST_BUS_USB3>,
             <&ccu RST_BUS_USB3_PHY0>;
    reset-names = "dwc3", "phy";
    ...
};
```

#### الحل

**الخطوة 1:** أضف الـ resets في الـ custom DT:

```dts
/* h616-custom.dts — FIXED */
#include <dt-bindings/reset/sun50i-h616-ccu.h>

&usb3_0 {
    status = "okay";
    resets = <&ccu RST_BUS_USB3>,
             <&ccu RST_BUS_USB3_PHY0>;
    reset-names = "dwc3", "phy";
};
```

**الخطوة 2:** تحقق من الـ CCU (Clock Control Unit) driver موجود:

```bash
ls /sys/kernel/debug/clk/ | grep usb
ls /sys/kernel/debug/reset/ | grep usb
```

**الخطوة 3:** بعد الـ fix، تحقق:

```bash
dmesg | grep -i "usb\|dwc3\|reset"
lsusb
ls /sys/bus/usb/devices/
```

**الخطوة 4:** لو محتاج تتحقق من الـ reset state يدويًا أثناء الـ debug:

```bash
# force deassert للـ USB3 reset (لو الـ debugfs بيسمح)
echo 0 > /sys/kernel/debug/reset/usb3/assert
```

#### الدرس المستفاد
الـ `_optional_` variants في الـ reset API صمّمت للـ drivers اللي ممكن تشتغل على أكتر من SoC — بعضها عنده reset وبعضها لأ. على custom board bring-up، دايمًا افحص `reset_control_get_count` أو الـ debugfs عشان تتأكد إن الـ DT فيه الـ resets الصح. الـ silence مش دايمًا success.
## Phase 7: مصادر ومراجع

### توثيق الـ Kernel الرسمي

| المصدر | الوصف |
|--------|-------|
| [`Documentation/driver-api/reset.rst`](https://www.kernel.org/doc/Documentation/driver-api/reset.rst) | التوثيق الرسمي لـ Reset Controller API — نقطة البداية الإلزامية |
| [`Documentation/devicetree/bindings/reset/reset.txt`](https://www.kernel.org/doc/Documentation/devicetree/bindings/reset/reset.txt) | Device Tree bindings للـ reset consumers والـ providers |
| [Reset controller API — kernel.org](https://docs.kernel.org/driver-api/reset.html) | النسخة الـ HTML من توثيق الـ API الكامل |
| [Reset controller API — v5.18](https://www.kernel.org/doc/html/v5.18/driver-api/reset.html) | نسخة مرجعية تاريخية من التوثيق |

### مقالات LWN.net

**الـ** LWN.net هو المرجع الأهم لمتابعة تطور الـ kernel patches والـ subsystem discussions.

| المقال | الأهمية |
|--------|---------|
| [Reset: Add generic GPIO reset driver](https://lwn.net/Articles/585145/) | أول driver عام يستخدم الـ reset framework عبر GPIO — مثال حي على الـ provider implementation |
| [Add Arria10 System Manager Reset Controller](https://lwn.net/Articles/714705/) | إضافة reset controller لـ Altera Arria10 SoC — يوضح كيف تُكتب الـ provider drivers |
| [clk: en7523: reset-controller support for EN7523 SoC](https://lwn.net/Articles/1039543/) | مثال حديث لـ SoC reset controller متكامل مع الـ clock subsystem |
| [PCI: Expose and manage PCI device reset](https://lwn.net/Articles/864911/) | إدارة الـ reset في سياق PCI/PCIe — يوضح تعقيدات الـ multi-reset scenarios |
| [PCI: Expose and manage PCI device reset (discussion)](https://lwn.net/Articles/866578/) | نقاش المجتمع حول الـ sysfs interface لعمليات الـ reset |

### Patchwork — تاريخ تطور الـ API

الـ patchwork يحتفظ بتاريخ الـ patches من لحظة إرسالها حتى الـ merge.

| الـ Patch | الأهمية |
|-----------|---------|
| [v4: Reset: Add reset controller API — Philipp Zabel, 2013](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/) | **الـ patch الأصلي** اللي أدخل الـ reset controller framework للـ kernel — إلزامي للفهم الجذري |
| [v4: Reset: Add GPIO support to reset controller framework](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1397463709-19405-2-git-send-email-p.zabel@pengutronix.de/) | إضافة الـ GPIO reset driver للـ framework — 2014 |
| [GIT PULL: Reset controller changes for v3.19](https://patchwork.ozlabs.org/project/linux-imx/patch/1417424646.4624.2.camel@pengutronix.de/) | pull request تاريخي يوثق المرحلة الأولى من نضوج الـ subsystem |

### Lore.kernel.org — نقاشات الـ Mailing List

الـ **lore.kernel.org** هو الأرشيف الرسمي لكل نقاشات الـ mailing lists.

| النقاش | الأهمية |
|--------|---------|
| [GIT PULL: Reset controller fixes for v6.2 — Philipp Zabel](https://lore.kernel.org/all/20230104095855.3809733-1-p.zabel@pengutronix.de/) | مثال على دورة حياة الـ fixes في subsystem ناضج |
| [Why do we need reset_control_get_optional()?](https://groups.google.com/g/fa.linux.kernel/c/9jG5RFPcEZU) | نقاش مهم يشرح الـ rationale وراء الـ optional variant |
| [ARM: imx: drop devlinks to reset-controller node](https://lore.kernel.org/linux-arm-kernel/4746f394f06d0a52ee2bfd26d6f216dfd4485231.camel@pengutronix.de/) | مثال على تطور الـ DT bindings مع الوقت |
| [dt-bindings: reset: Document BCM7216 RESCAL reset controller](https://lore.kernel.org/linux-arm-kernel/20200103190429.1847-2-f.fainelli@gmail.com/) | نموذج لكتابة الـ DT bindings documentation |
| [imx_dsp_rproc: Use reset controller API to control the DSP](https://www.mail-archive.com/linux-kernel@vger.kernel.org/msg2584130.html) | مثال عملي على استخدام الـ API مع الـ remoteproc subsystem |

### الكود المصدري المرجعي في الـ Kernel

```
include/linux/reset.h          ← Consumer API (الملف محل الدراسة)
include/linux/reset-controller.h ← Provider API
drivers/reset/core.c           ← تنفيذ الـ core logic
drivers/reset/reset-gpio.c     ← مثال بسيط على provider
drivers/reset/Kconfig          ← قائمة الـ drivers المتاحة
```

الـ `drivers/reset/` يحتوي على أكثر من 60 driver — كل منها مثال حي على implementation مختلفة.

### كتب مرجعية

#### Linux Device Drivers, 3rd Edition (LDD3)
- **الـ** LDD3 متاح مجاناً على [lwn.net/Kernel/LDD2](https://lwn.net/Kernel/LDD2/)
- الـ reset framework جاء بعد LDD3، لكن الفصول دي أساسية لفهم السياق:
  - **Chapter 14: The Linux Device Model** — بيشرح الـ `struct device` والـ device tree integration
  - **Chapter 9: Communicating with Hardware** — الأساس لفهم كيف الـ driver بيتحكم في الـ hardware

#### Linux Kernel Development, 3rd Edition — Robert Love
- **Chapter 17: Devices and Modules** — بيشرح الـ device model اللي الـ reset framework بيبني عليه
- **Chapter 11: Timers and Time Management** — مهم لفهم الـ timing بعد الـ assert/deassert

#### Embedded Linux Primer, 2nd Edition — Christopher Hallinan
- **Chapter 15: Kernel Initialization** — بيشرح كيف الـ reset signals بتُستخدم في مرحلة الـ boot
- **Chapter 16: Using the Kernel's Device Model** — السياق الكامل للـ platform drivers

#### Mastering Embedded Linux Programming — Frank Vasquez & Chris Simmonds
- الطبعة الثالثة بتغطي الـ device tree و platform drivers بشكل عملي
- الأنسب للي بيكتب reset driver من الصفر لـ custom SoC

### مصطلحات البحث

لو عايز تكمل البحث في الموضوع ده، استخدم الـ search terms دي:

```
# للبحث في الـ kernel source
git log --oneline --all -- drivers/reset/
git log --oneline --all -- include/linux/reset.h

# للبحث في الـ mailing lists على lore.kernel.org
reset-controller@vger.kernel.org
linux-arm-kernel@lists.infradead.org

# Google search terms
linux kernel "reset_control_get_exclusive" site:lore.kernel.org
linux "reset controller" "shared reset" OR "exclusive reset" site:lwn.net
linux kernel "devm_reset_control" driver example
linux "reset_control_bulk" patch
```

### ملخص المراجع حسب الأولوية

| الأولوية | المرجع | السبب |
|----------|--------|-------|
| **إلزامي** | `docs.kernel.org/driver-api/reset.html` | التوثيق الرسمي الكامل للـ API |
| **إلزامي** | Patchwork: original reset controller patch | بيفهمك الـ design decisions من الأساس |
| **مهم جداً** | LWN: GPIO reset driver article | أبسط مثال كامل على provider + consumer |
| **مهم** | `drivers/reset/core.c` (kernel source) | تنفيذ الـ shared/exclusive logic فعلياً |
| **للعمق** | LDD3 Chapters 9 & 14 | الأساس النظري للـ device model |
| **للعمق** | Linux Kernel Development — Robert Love | السياق الكامل للـ kernel architecture |
## Phase 8: Writing simple module

### الفكرة

هنعمل kprobe على الدالة `reset_control_assert` — دالة exported من reset framework بتتحكم في تأكيد حالة الـ reset على hardware peripherals. كل مرة أي driver يعمل assert على reset line، الـ kprobe handler بتاعتنا بيطبع معلومات عن الـ caller والـ `reset_control` pointer. ده مفيد جداً لمعرفة أي device بيتعمله reset ومتى.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on reset_control_assert() — traces every hardware reset assertion
 * from any driver using the Linux reset framework.
 *
 * Tested against: linux/include/linux/reset.h
 */

#include <linux/module.h>       /* module_init / module_exit / MODULE_* macros */
#include <linux/kernel.h>       /* pr_info / pr_err */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, etc. */
#include <linux/reset.h>        /* struct reset_control — the probed function's arg type */
#include <linux/device.h>       /* struct device — to print device name if reachable */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc-probe");
MODULE_DESCRIPTION("Trace reset_control_assert() calls via kprobe");

/* ------------------------------------------------------------------ */
/* Pre-handler: runs just before reset_control_assert() executes       */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64 the first argument is in rdi.
     * On ARM64 it is in x0.
     * kprobes gives us pt_regs so we read it portably via regs_get_kernel_argument().
     */
    struct reset_control *rstc =
        (struct reset_control *)regs_get_kernel_argument(regs, 0);

    /*
     * We intentionally do NOT dereference rstc internals here because
     * reset_control is an opaque struct (defined in drivers/reset/core.c,
     * not in the public header).  Printing its pointer value is safe and
     * already useful for cross-referencing with /proc/kallsyms or ftrace.
     */
    pr_info("reset_probe: reset_control_assert called | rstc=%px | caller=%pS\n",
            rstc,                          /* opaque pointer — safe to print */
            (void *)regs_get_register(regs, offsetof(struct pt_regs, ARCH_RET_REG)));
            /* ARCH_RET_REG gives return address / link register on most arches */

    /* Simpler cross-arch alternative for the caller address: */
    pr_info("reset_probe: caller IP = %pS\n", (void *)regs->ip); /* x86 */

    return 0; /* 0 = let the real function run normally */
}

/* ------------------------------------------------------------------ */
/* Post-handler: runs after reset_control_assert() returns             */
/* ------------------------------------------------------------------ */
static void handler_post(struct kprobe *p, struct pt_regs *regs,
                         unsigned long flags)
{
    /*
     * regs->ax on x86-64 holds the return value after the function ran.
     * A non-zero value means the assert failed.
     */
    long retval = regs_return_value(regs);   /* portable helper */

    if (retval)
        pr_info("reset_probe: reset_control_assert FAILED ret=%ld\n", retval);
    else
        pr_info("reset_probe: reset_control_assert SUCCESS\n");
}

/* ------------------------------------------------------------------ */
/* kprobe descriptor                                                   */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name    = "reset_control_assert", /* function to intercept */
    .pre_handler    = handler_pre,
    .post_handler   = handler_post,
};

/* ------------------------------------------------------------------ */
/* init / exit                                                         */
/* ------------------------------------------------------------------ */
static int __init reset_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("reset_probe: register_kprobe failed, ret=%d\n", ret);
        pr_err("reset_probe: is CONFIG_KPROBES=y and the symbol exported?\n");
        return ret;
    }

    pr_info("reset_probe: planted kprobe at %px (%s)\n",
            kp.addr, kp.symbol_name);
    return 0;
}

static void __exit reset_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("reset_probe: kprobe removed from %px\n", kp.addr);
}

module_init(reset_probe_init);
module_exit(reset_probe_exit);
```

---

### Makefile

```makefile
obj-m += reset_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
# تجميع وتحميل الـ module
make
sudo insmod reset_probe.ko
sudo dmesg | tail -20        # مشاهدة الـ output
sudo rmmod reset_probe
```

---

### شرح كل قسم

#### الـ Includes

| Header | السبب |
|--------|-------|
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل الـ API الخاصة بالـ kprobes — لازم لأي module بيستخدم هذه التقنية |
| `linux/reset.h` | بيعرّف `struct reset_control` اللي هي argument الدالة المستهدفة، حتى لو هنستخدمها كـ opaque pointer بس |
| `linux/device.h` | موجود للمرجعية لأن الـ reset API مرتبطة ارتباطاً وثيقاً بـ `struct device` |
| `linux/module.h` | بيعرّف الـ macros الأساسية `module_init`, `module_exit`, `MODULE_LICENSE` إلخ — لا يتخطى في أي module |

#### الـ `handler_pre`

**الـ `regs_get_kernel_argument(regs, 0)`** بيجيب أول argument للدالة (الـ `rstc` pointer) بطريقة portable عبر الـ architectures المختلفة بدل ما نكتب `regs->di` مباشرة وننحبس على x86 بس. بنطبعه كـ pointer `%px` علشان نعرف إيه الـ reset control اللي اتعمله assert.

**الـ `%pS` format specifier** بيحوّل عنوان الـ IP لاسم رمزي من kernel symbol table — يطلع مثلاً `clk_disable+0x3c/0x80 [clk_sunxi]` بدل مجرد عنوان hex. ده بيساعد في معرفة مين بيعمل reset بالضبط بدون أي أدوات خارجية.

#### الـ `handler_post`

**الـ `regs_return_value(regs)`** بيقرأ قيمة الـ return من الـ register الصحيح على كل architecture. لو الـ assert فشل (يرجع `-EINVAL` أو `-EIO` مثلاً)، بنطبع رسالة تنبيه — مفيد لاكتشاف hardware مشكلة فيها أو misconfigured reset lines.

#### الـ `struct kprobe kp`

الـ `.symbol_name` بيخلي الـ kernel يحل العنوان تلقائياً عند الـ registration بدل ما نحدده hardcoded. ده أآمن وأسهل لأن الـ KASLR بيحرك العناوين عند كل boot.

#### الـ `module_init` / `module_exit`

**الـ `register_kprobe`** بتزرع breakpoint في الـ kernel code عند entry point الدالة. لو فشلت (symbol مش موجود، أو الـ function inlined، أو `CONFIG_KPROBES` مش موجود) بنرجع error ومنحملش.

**الـ `unregister_kprobe` في الـ exit** ضروري جداً — لو تركنا الـ breakpoint من غير تسجيل، الـ kernel هيكمل يستدعي handler على memory اتحررت وده بيودي لـ kernel panic أو memory corruption. الـ cleanup هنا مش اختياري.

---

### مثال على الـ Output

```
[ 1234.567890] reset_probe: planted kprobe at ffffffffc0ab1234 (reset_control_assert)
[ 1235.100001] reset_probe: reset_control_assert called | rstc=ffff888012345678 | ...
[ 1235.100003] reset_probe: caller IP = phy_init+0x44/0xb0 [phy_generic]
[ 1235.100010] reset_probe: reset_control_assert SUCCESS
[ 1236.200001] reset_probe: reset_control_assert called | rstc=ffff888087654321 | ...
[ 1236.200003] reset_probe: caller IP = dwmac4_core_init+0x1c/0x70 [stmmac]
[ 1236.200010] reset_probe: reset_control_assert FAILED ret=-5
```

الـ output ده بيكشف مباشرة إن `dwmac4` (Ethernet MAC driver) حاول يعمل reset على hardware لكن فشل برمز `-EIO`، وده مش واضح في الـ normal kernel logs.
