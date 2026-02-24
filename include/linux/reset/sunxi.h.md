## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف ده جزء من **ARM/Allwinner sunXi SoC support** subsystem، اللي بيتولى دعم معالجات Allwinner (الصانع الصيني) في Linux kernel. الـ subsystem ده بيغطي كل حاجة من الـ clock management، الـ pinctrl، الـ SoC machine code، والـ reset controller.

---

### الصورة الكبيرة — ليه الـ Reset Controller موجود أصلاً؟

تخيل معايا إنك بتبني مدينة (الـ SoC = System on Chip). في المدينة دي فيه عشرات المباني: مبنى الـ USB، مبنى الـ Ethernet، مبنى الـ display engine، مبنى الـ camera، إلخ. كل مبنى ده عبارة عن **hardware block** جوه الـ chip.

لما بتشغّل المدينة أول مرة، أو لما مبنى معين واقف (hung) أو بيتصرف غلط، محتاج **تعمله reset** — يعني تقطع عنه الكهرباء وترجّعها تاني عشان يبدأ من الأول. الـ **Reset Controller** ده هو "لوحة المفاتيح المركزية" اللي بتتحكم في reset كل مبنى في المدينة. كل bit في register واحد = reset لـ hardware block واحد.

---

### المشكلة اللي بيحلها الملف ده

الـ **sun6i-A31** (Allwinner A31 SoC) عنده مشكلة خاصة:

- الـ **AHB1 bus** هو الـ bus الرئيسي اللي بتتصل بيه معظم الـ peripherals (زي الـ USB، GPIO، إلخ).
- الـ **AHB1 Reset Controller** — المسؤول عن reset الـ AHB1 devices — لازم يتشغّل **قبل** أي حاجة تاني خالص. يعني حتى قبل ما الـ kernel يعمل probe للـ devices من خلال الـ device model العادي.
- لو انتظرنا الـ normal device driver initialization، هنكون محتاجين نعمل reset لـ devices قبل ما الـ reset controller نفسه يكون جاهز — **مشكلة الـ chicken-and-egg الكلاسيكية**.

الحل هو تشغيل الـ reset controller ده **early** جداً — يعني تشغيله يدوياً وقت الـ machine init، قبل ما الـ device driver framework يبدأ شغله.

---

### دور الملف `include/linux/reset/sunxi.h`

الملف ده **header صغير جداً** — سطر واحد بس بيعمل declare لـ function واحدة:

```c
void __init sun6i_reset_init(void);
```

**الـ `__init`** معناها إن الـ function دي بتتشغل مرة واحدة بس وقت الـ boot، وبعدين الـ kernel ممكن يحذف الـ code بتاعها من الذاكرة عشان يوفر space.

الهدف من الـ header ده بسيط: **يفصل الـ interface عن الـ implementation**. الـ machine code في `arch/arm/mach-sunxi/sunxi.c` محتاج يعرف إن الـ function دي موجودة عشان يقدر يستدعيها، من غير ما يحتاج يعرف تفاصيل الـ implementation.

---

### الـ Story الكاملة — إيه اللي بيحصل وقت الـ Boot؟

```
Boot يبدأ
    │
    ▼
arch/arm/mach-sunxi/sunxi.c
    └── sun6i_timer_init()  ← machine init_time callback
            │
            ├── of_clk_init(NULL)       ← init clocks أولاً
            │
            ├── sun6i_reset_init()      ← ← ← هنا دور header ده
            │       │
            │       └── drivers/reset/reset-sunxi.c
            │               └── بيدور في Device Tree على
            │                   "allwinner,sun6i-a31-ahb1-reset"
            │                   وبيعمل register لـ reset controller
            │                   قبل ما أي driver تاني يبدأ
            │
            └── timer_probe()           ← بعدين الـ timers
```

يعني الـ header ده هو "الجسر" اللي بيسمح لـ machine code (arch layer) إنه يستدعي الـ reset driver code (drivers layer) في وقت مبكر جداً.

---

### ليه الـ AHB1 Reset محتاج Early Init؟

الـ **AHB1 bus** في sun6i-A31 بيضم devices زي:
- **USB OTG controller**
- **GPIO controller**
- **DMA controller**
- **و devices تانية كتير**

كل الـ devices دي محتاجة تتعمل deassert من الـ reset قبل ما يقدروا يشتغلوا. لو الـ reset controller نفسه اتعمل init في وقت متأخر، الـ devices دي هتفشل في الـ probe phase لأنها هتلاقي نفسها لسه في حالة reset.

---

### الفرق بين Early Reset وـ Normal Reset

| الـ Property | Early Reset (sunxi.h) | Normal Reset (device model) |
|---|---|---|
| **وقت التشغيل** | أثناء `init_time` callback | بعد `device_initcall` |
| **الطريقة** | يدوي عبر `sun6i_reset_init()` | تلقائي عبر `platform_driver` |
| **الـ DT matching** | `for_each_matching_node` | `of_match_table` |
| **الـ use case** | AHB1 reset على sun6i فقط | باقي الـ SoCs |

---

### الملفات المكوّنة للـ Subsystem

#### الـ Core Files

| الملف | الدور |
|---|---|
| `include/linux/reset/sunxi.h` | **الملف ده** — declares `sun6i_reset_init()` |
| `drivers/reset/reset-sunxi.c` | الـ implementation — early reset init للـ sun6i AHB1 |
| `drivers/reset/reset-simple.c` | الـ generic simple reset driver اللي sunxi بيستخدمه |
| `drivers/reset/core.c` | الـ core reset controller framework |

#### الـ Headers

| الملف | الدور |
|---|---|
| `include/linux/reset/reset-simple.h` | `struct reset_simple_data` و `reset_simple_ops` |
| `include/linux/reset-controller.h` | `struct reset_controller_dev` — الـ base struct |

#### الـ Machine & Architecture Files

| الملف | الدور |
|---|---|
| `arch/arm/mach-sunxi/sunxi.c` | machine init — بيستدعي `sun6i_reset_init()` |
| `arch/arm64/boot/dts/allwinner/` | الـ Device Tree للـ SoCs |

#### الـ Clock & Related

| الملف | الدور |
|---|---|
| `drivers/clk/sunxi-ng/` | الـ new-generation clock driver للـ Allwinner |
| `drivers/clk/sunxi/` | الـ older clock drivers |
| `drivers/pinctrl/sunxi/` | الـ GPIO/pinctrl للـ Allwinner |
| `drivers/soc/sunxi/` | الـ SoC-specific drivers |
## Phase 2: شرح الـ Reset Controller Framework

### المشكلة — ليه الـ subsystem ده موجود؟

في أي SoC حديث زي Allwinner A31 (sun6i)، في عشرات الـ peripherals: USB, EMAC, SPI, I2C, UART, GPU, وغيرهم. كل peripheral عنده **reset line** — سلك واحد بيتحكم فيه هاردوير بيحط الـ peripheral في حالة reset كاملة (registers بترجع لـ defaults، logic بتتصفر).

المشكلة الأساسية كانت:

1. **كل درايفر كان بيعمل reset بطريقته الخاصة** — بيعمل `ioremap` للرجستر، بيعمل `readl`، بيعمل `writel` بـ bit محدد، وبيكرر الكود في كل درايفر.
2. **الـ reset registers غالباً shared** — رجستر واحد فيه 32 bit، كل bit بيعمل reset لـ peripheral مختلف. لو درايفران عملوا read-modify-write في نفس الوقت من غير locking، حصل race condition وحد فيهم اتمسح reset bit التاني.
3. **Early boot problem** — بعض الـ resets لازم تتعمل قبل ما الـ device model يشتغل خالص، يعني قبل `probe()` وقبل `platform_device`. مثلاً على sun6i، الـ AHB1 bus reset لازم يتعمل early جداً وإلا الـ system مش بيبقى stable.
4. **لا يوجد تتبع لمن يملك reset line** — لو درايفران بيستخدموا نفس الـ shared reset، مين المسؤول؟ مين يعمل assert ومين يعمل deassert؟

---

### الحل — الـ Reset Controller Framework

**الـ Reset Controller subsystem** (اللي اتضاف الكيرنل سنة 2013 بشكل كبير مع Allwinner كأول use case) بيحل المشكلة بـ:

1. **فصل الـ provider عن الـ consumer** — الكود اللي بيعرف يتكلم مع الهاردوير (provider) منفصل تماماً عن الكود اللي عايز يعمل reset (consumer).
2. **Centralized locking** — الـ spinlock في الـ provider بيحمي الـ read-modify-write على مستوى واحد.
3. **Reference counting للـ shared resets** — الـ framework بيتتبع عدد الـ consumers اللي عملوا deassert، وبيعمل assert فعلي بس لما العدد يرجع للصفر.
4. **Early init path** — بيسمح بتسجيل reset controller في `__init` قبل الـ device model بالكامل.

---

### التشبيه الواقعي — لوحة الكهرباء (Circuit Breaker Panel)

تخيل لوحة كهرباء في مبنى:

| عنصر في التشبيه | المقابل في الكيرنل |
|---|---|
| لوحة الكهرباء نفسها | `struct reset_controller_dev` (rcdev) |
| كل circuit breaker داخل اللوحة | reset line واحدة (bit واحد في رجستر) |
| رقم الـ breaker | الـ `id` (unsigned long) اللي بيمثل رقم الـ bit |
| الكهربائي اللي بيتحكم في اللوحة | الـ provider driver (reset-sunxi.c / reset-simple.c) |
| الأجهزة اللي بتسحب كهرباء من الـ breaker | الـ consumer drivers (USB driver, SPI driver...) |
| استأذن من الكهربائي تفصل الكهرباء | `reset_control_get_exclusive()` |
| فصل الكهرباء | `reset_control_assert()` |
| وصل الكهرباء تاني | `reset_control_deassert()` |
| فصل وتوصيل في operation واحدة | `reset_control_reset()` |
| Breaker مشترك بين شقتين | shared reset control |
| الكهربائي بيحفظ إن الشقتين محتاجتين الكهرباء | deassert reference count |

**الأعمق من كده:** تخيل إن الـ breaker واحد بيوصل كهرباء لمطبخين (shared reset). لو الشقة A طلبت فصل الكهرباء، الكهربائي (الكيرنل) مش بيفصلها فعلاً لأن الشقة B لسه محتاجاها. بس لما الاتنين طلبوا الفصل، بيفصل. ده بالظبط الـ `deassert_count` في الـ framework.

---

### معمارية الـ Framework — الصورة الكبيرة

```
┌─────────────────────────────────────────────────────────────────┐
│                        Consumer Drivers                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │USB Driver│  │SPI Driver│  │I2C Driver│  │ Other Driver │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘   │
│       │              │              │                │           │
│  reset_control_deassert()    reset_control_reset()             │
│       │              │              │                │           │
└───────┼──────────────┼──────────────┼────────────────┼──────────┘
        │              │              │                │
        ▼              ▼              ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Reset Controller Core                         │
│              (drivers/reset/core.c)                             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  struct reset_control (per consumer handle)             │   │
│  │  ├── *rcdev  ──────────────────────────────────────┐   │   │
│  │  ├── id (line number)                               │   │   │
│  │  ├── deassert_count                                 │   │   │
│  │  ├── acquired / shared flags                        │   │   │
│  │  └── list_head (linked to rcdev)                   │   │   │
│  └─────────────────────────────────────────────────────┘   │   │
│                                                             │   │
│  ┌─────────────────────────────────────────────────────┐   │   │
│  │  struct reset_controller_dev (rcdev)  ◄─────────────┘   │   │
│  │  ├── *ops ──────────────────────────────────────────┐   │   │
│  │  ├── nr_resets                                      │   │   │
│  │  ├── of_node                                        │   │   │
│  │  ├── of_xlate()                                     │   │   │
│  │  ├── reset_control_head (list of all handles)       │   │   │
│  │  └── list (global list of all rcdevs)               │   │   │
│  └─────────────────────────────────────────────────────┘   │   │
└─────────────────────────────────────────────────────────────────┘
                              │ ops->assert()
                              │ ops->deassert()
                              │ ops->reset()
                              │ ops->status()
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Provider Drivers                              │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  reset-sunxi.c (early init path)                        │  │
│  │  sun6i_reset_init() → sunxi_reset_init()               │  │
│  │  └── uses reset-simple ops                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  reset-simple.c (generic provider)                      │  │
│  │  reset_simple_ops: { assert, deassert, status }        │  │
│  │  struct reset_simple_data:                              │  │
│  │    ├── spinlock_t lock    ← protects read-modify-write  │  │
│  │    ├── void __iomem *membase  ← mapped registers       │  │
│  │    ├── rcdev              ← embedded rcdev struct       │  │
│  │    └── active_low         ← polarity (sun6i = true)    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│                              │                                  │
│                              ▼                                  │
│                   ┌──────────────────┐                         │
│                   │  MMIO Registers  │                         │
│                   │  (AHB1 Reset)    │                         │
│                   │  bit[N]=0 →      │                         │
│                   │  peripheral in   │                         │
│                   │  reset (sun6i)   │                         │
│                   └──────────────────┘                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Allwinner sun6i SoC Hardware                   │
│                                                                 │
│  AHB1 Reset Register (0x01C202C0):                             │
│  ┌────┬────┬────┬────┬─────┬────┬────┬────┐                   │
│  │b31 │... │b17 │b16 │ ... │ b6 │ b5 │ b0 │                   │
│  │USB3│    │SPI0│I2C0│     │DMA │USB1│... │                   │
│  └────┴────┴────┴────┴─────┴────┴────┴────┘                   │
│   0 = in reset (active_low=true)                               │
│   1 = running normal                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

### العلاقة بين الـ Structs

```
Device Tree:
  usb@01c13000 {
      resets = <&ahb_rst 17>;   /* phandle + reset ID */
  };

          │
          │ of_reset_control_get_exclusive()
          │
          ▼
┌─────────────────────────┐         ┌─────────────────────────┐
│   reset_control         │         │  reset_controller_dev   │
│  (consumer handle)      │────────►│  (rcdev)                │
│                         │  *rcdev │                         │
│  id = 17                │         │  ops = &reset_simple_ops│
│  deassert_count = 1     │         │  nr_resets = 32         │
│  shared = false         │         │  of_node = ahb_rst node │
│  list_head ─────────────┼────────►│  reset_control_head     │
└─────────────────────────┘         └────────────┬────────────┘
                                                 │
                                                 │ container_of
                                                 ▼
                                    ┌─────────────────────────┐
                                    │  reset_simple_data      │
                                    │                         │
                                    │  lock (spinlock)        │
                                    │  membase ───────────────┼──► MMIO
                                    │  active_low = true      │
                                    │  rcdev (embedded)       │
                                    └─────────────────────────┘
```

**الـ `rcdev` بيكون embedded داخل `reset_simple_data`** — يعني الـ provider بيعمل `container_of` عشان يوصل للـ data الكاملة من الـ ops callbacks.

---

### الـ Core Abstraction — الفكرة المحورية

**الـ core abstraction هي الـ `reset_control` handle** — دي pointer مش للهاردوير مباشرة، دي handle للعلاقة بين consumer معين وreset line معينة.

ده بيفصل تلات مستويات:

```
Consumer API              Framework Core           Provider Ops
─────────────             ──────────────           ────────────
reset_control_assert()  → checks ownership     →  ops->assert(rcdev, id)
                          checks shared flag
                          manages ref count
                          takes spinlock (provider side)
```

الـ consumer **مش بيعرف** إيه الـ register address، ولا إيه الـ bit number، ولا إيه الـ polarity. كل ده encapsulated في الـ provider.

---

### الـ Early Init Path — sun6i_reset_init()

ده الجزء الفريد اللي الـ header `sunxi.h` موجود عشانه. الـ function `sun6i_reset_init()` بتاخد الـ `__init` attribute — يعني بتتشغل في early boot، قبل الـ device model يشتغل.

```c
void __init sun6i_reset_init(void)
{
    struct device_node *np;

    /* بيمشي على كل node في الـ DT عنده compatible مطابق */
    for_each_matching_node(np, sunxi_early_reset_dt_ids)
        sunxi_reset_init(np);  /* بيعمل register للـ controller من غير platform_device */
}
```

**ليه ده مهم؟** لأن على sun6i، الـ AHB1 bus reset controller (**compatible: "allwinner,sun6i-a31-ahb1-reset"**) لازم يتسجل قبل أي driver تاني يقدر يعمل reset لأي peripheral على الـ AHB1 bus. لو اتأخر لـ `module_init` العادي، الـ USB أو الـ DMA هيفشل في الـ probe لأن الـ reset controller مش موجود لسه.

**الفرق بين المسارين:**

| المسار | متى؟ | الـ compatible |
|---|---|---|
| `sun6i_reset_init()` | Early `__init` | `allwinner,sun6i-a31-ahb1-reset` |
| `platform_driver` عادي | بعد device model | باقي الـ sun* resets |

---

### الـ active_low Polarity — تفصيلة مهمة

على sun6i (**Allwinner A31**)، الـ reset bits بتشتغل بـ **active_low = true**:

```
bit = 0  →  peripheral in RESET  (سلك الـ reset متفعّل)
bit = 1  →  peripheral running   (سلك الـ reset مش متفعّل)
```

يعني `assert` (تحط في reset) = **clear** الـ bit.
و`deassert` (تطلعه من reset) = **set** الـ bit.

ده عكس الـ convention المنطقي اللي بتفكر فيه. الـ `reset_simple_ops` بتتعامل مع الاتنين:

```c
/* من reset-simple.c — الـ logic بيعكس الـ bit لو active_low */
static int reset_simple_assert(struct reset_controller_dev *rcdev,
                               unsigned long id)
{
    /* active_low=true → assert يعني clear الـ bit */
    /* active_low=false → assert يعني set الـ bit */
}
```

---

### الـ Ownership Model — Exclusive vs. Shared

**Exclusive reset** (`RESET_CONTROL_EXCLUSIVE`):
- درايفر واحد بس يقدر يمسك الـ handle.
- لو درايفر تاني طلبه، الـ framework بيرجع `-EBUSY`.
- مناسب لـ peripheral مش بيتشاور مع حد.

**Shared reset** (`RESET_CONTROL_SHARED`):
- أكتر من درايفر يقدر يمسك handle على نفس الـ reset line.
- الـ framework بيعمل reference counting على الـ deassert.
- لو 3 drivers عملوا deassert، لازم الـ 3 يعملوا assert عشان الـ reset يترجع.
- مناسب لـ shared clock domain أو shared power domain.

---

### الـ Framework بيملك — والـ Drivers بيعملوا

| الـ Reset Controller Framework يملك | الـ Provider Driver بيعمل |
|---|---|
| Global list of all `rcdev`s | تسجيل `rcdev` عبر `reset_controller_register()` |
| Lookup عبر DT phandle | تحديد `of_xlate()` لتحويل DT specifier لـ ID |
| Exclusive/shared ownership tracking | تنفيذ `ops->assert()` و`ops->deassert()` |
| Reference counting للـ shared resets | الـ MMIO read-modify-write مع الـ spinlock |
| `devm_` cleanup على driver detach | تحديد `nr_resets` و`active_low` |
| Validation إن الـ consumer لا يتجاوز `nr_resets` | Early registration لو احتاج `__init` |

---

### Subsystems ذات صلة — لازم تعرفهم

- **الـ Device Tree / OF subsystem**: الـ framework بيعتمد عليه عشان يعمل lookup للـ reset controller من الـ DT phandle (`resets = <&ahb_rst 17>`). اعرف إزاي `of_phandle_args` بتشتغل.
- **الـ Clock Framework (clk)**: أقرب sibling للـ reset framework. نفس الفلسفة — provider/consumer separation. كتير من الـ peripherals محتاجة clock enable وreset deassert في نفس الوقت.
- **الـ devm (resource management)**: الـ `devm_reset_control_get_*()` functions بتضمن إن الـ `reset_control_put()` بيتعمل أوتوماتيك لما الـ driver يتسحب.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

> **ملاحظة مهمة:** الـ header الأصلي `include/linux/reset/sunxi.h` بسيط جداً — فيه declaration واحدة فقط هي `sun6i_reset_init()`. لكن عشان نفهم الصورة الكاملة، هنشرح النظام كله بما فيه implementation في `drivers/reset/reset-sunxi.c` والـ structs الجاية من `reset-simple.h` و `reset-controller.h`.

---

### الملف الأصلي: `include/linux/reset/sunxi.h`

```c
/* SPDX-License-Identifier: GPL-2.0 */
#ifndef __LINUX_RESET_SUNXI_H__
#define __LINUX_RESET_SUNXI_H__

void __init sun6i_reset_init(void);  /* Early init — called before driver model */

#endif
```

الـ header ده تعريف واحد بس — entry point للـ early boot reset controller الخاص بـ Allwinner sun6i (A31 SoC).

---

### 0. Flags, Enums, Config Options — Cheatsheet

#### `enum reset_control_flags` (من `include/linux/reset.h`)

| القيمة | الـ Bits المستخدمة | المعنى |
|---|---|---|
| `RESET_CONTROL_EXCLUSIVE` | `BIT(2)` | حجز حصري، مش هيتشارك مع حد |
| `RESET_CONTROL_EXCLUSIVE_DEASSERTED` | `BIT(2)\|BIT(3)` | حصري + deassert فوراً |
| `RESET_CONTROL_EXCLUSIVE_RELEASED` | `0` | حصري بس released — لازم acquire قبل الاستخدام |
| `RESET_CONTROL_SHARED` | `BIT(0)` | مشترك بين عدة consumers |
| `RESET_CONTROL_SHARED_DEASSERTED` | `BIT(0)\|BIT(3)` | مشترك + deassert فوراً |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE` | `BIT(1)\|BIT(2)` | optional — مش error لو مش موجود |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE_DEASSERTED` | `BIT(1)\|BIT(2)\|BIT(3)` | optional + حصري + deassert |
| `RESET_CONTROL_OPTIONAL_EXCLUSIVE_RELEASED` | `BIT(1)` | optional + released |
| `RESET_CONTROL_OPTIONAL_SHARED` | `BIT(0)\|BIT(1)` | optional + مشترك |
| `RESET_CONTROL_OPTIONAL_SHARED_DEASSERTED` | `BIT(0)\|BIT(1)\|BIT(3)` | optional + مشترك + deassert |

#### Bits الـ Flags

| الـ Bit | الثابت | المعنى |
|---|---|---|
| `BIT(0)` | `RESET_CONTROL_FLAGS_BIT_SHARED` | الـ reset مشترك |
| `BIT(1)` | `RESET_CONTROL_FLAGS_BIT_OPTIONAL` | مش إلزامي — error يتحول NULL |
| `BIT(2)` | `RESET_CONTROL_FLAGS_BIT_ACQUIRED` | الـ exclusive control متأخد فعلاً |
| `BIT(3)` | `RESET_CONTROL_FLAGS_BIT_DEASSERTED` | يعمل deassert تلقائي بعد get |

#### `active_low` في Sunxi — الفلسفة

| الـ Flag | assert كيف | deassert كيف |
|---|---|---|
| `active_low = true` (Sunxi default) | clear البت = 0 | set البت = 1 |
| `active_low = false` | set البت = 1 | clear البت = 0 |

> **Sunxi** بيستخدم `active_low = true` — يعني لما البت = 0 الـ peripheral في حالة reset، ولما = 1 شغال.

#### Config Options

| الـ Option | التأثير |
|---|---|
| `CONFIG_RESET_CONTROLLER` | لو disabled، كل الـ API بتبقى stubs ترجع 0 أو NULL |

---

### 1. الـ Structs المهمة

#### `struct reset_simple_data` (من `include/linux/reset/reset-simple.h`)

**الغرض:** الـ driver data الأساسية للـ simple reset controllers — Sunxi بيستخدمها directly.

```c
struct reset_simple_data {
    spinlock_t              lock;             /* يحمي read-modify-write على الـ registers */
    void __iomem           *membase;          /* base address للـ MMIO registers */
    struct reset_controller_dev rcdev;        /* embedded: تسجيل في الـ reset core */
    bool                    active_low;       /* true = clear bit to assert (Sunxi style) */
    bool                    status_active_low;/* true = bit reads 0 when asserted */
    unsigned int            reset_us;         /* minimum delay بين assert وdeassert */
};
```

| الحقل | النوع | الدور |
|---|---|---|
| `lock` | `spinlock_t` | يحمي الـ registers من race conditions |
| `membase` | `void __iomem *` | pointer للـ memory-mapped registers |
| `rcdev` | `reset_controller_dev` | embedded struct — بيتسجل في الـ reset subsystem |
| `active_low` | `bool` | logic polarity — Sunxi بيضبطها `true` |
| `status_active_low` | `bool` | polarity لقراءة الـ status |
| `reset_us` | `unsigned int` | minimum pulse width بالـ microseconds |

**الارتباط بالـ structs التانية:** الـ `rcdev` فيها embedded — يعني pointer على `reset_simple_data` من خلال `container_of(rcdev, struct reset_simple_data, rcdev)`.

---

#### `struct reset_controller_dev` (من `include/linux/reset-controller.h`)

**الغرض:** الـ entity الأساسية اللي بتمثل reset controller في kernel. أي controller لازم يسجل نفسه من خلالها.

```c
struct reset_controller_dev {
    const struct reset_control_ops *ops;      /* function pointers: reset/assert/deassert/status */
    struct module *owner;                     /* الـ kernel module اللي بيمتلك الـ driver */
    struct list_head list;                    /* linked list في الـ global reset controllers list */
    struct list_head reset_control_head;      /* list of all reset_control handles على الـ controller ده */
    struct device *dev;                       /* الـ device المرتبط (NULL في early init) */
    struct device_node *of_node;             /* الـ DT node عشان يربط الـ consumers بالـ controller */
    const struct of_phandle_args *of_args;   /* لو GPIO-based reset */
    int of_reset_n_cells;                    /* عدد cells في الـ reset specifier في DT */
    int (*of_xlate)(...);                    /* تحويل DT specifier لـ reset ID */
    unsigned int nr_resets;                  /* عدد الـ reset lines المتاحة */
};
```

| الحقل | الدور في Sunxi |
|---|---|
| `ops` | بيتضبط على `reset_simple_ops` |
| `owner` | `THIS_MODULE` |
| `nr_resets` | `size * 8` — يعني كل bit في الـ register space = reset line |
| `of_node` | الـ DT node لـ `allwinner,sun6i-a31-ahb1-reset` |
| `list` | بيتضاف على الـ global list في الـ reset core |

---

#### `struct reset_control_ops` (من `include/linux/reset-controller.h`)

**الغرض:** الـ vtable الخاصة بكل reset controller driver.

```c
struct reset_control_ops {
    int (*reset)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*assert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*deassert)(struct reset_controller_dev *rcdev, unsigned long id);
    int (*status)(struct reset_controller_dev *rcdev, unsigned long id);
};
```

الـ `reset_simple_ops` اللي Sunxi بيستخدمها — الـ implementation موجودة في `drivers/reset/reset-simple.c`:
- `assert`: بتعمل read-modify-write على الـ register — بتـ clear أو set البت حسب `active_low`
- `deassert`: عكس assert
- `reset`: assert ثم delay ثم deassert (لو `reset_us != 0`)
- `status`: بتقرأ البت وترجع الحالة

---

#### `struct reset_control_bulk_data` (من `include/linux/reset.h`)

**الغرض:** container بسيط لاستخدام الـ bulk API — لما device محتاجة أكتر من reset line.

```c
struct reset_control_bulk_data {
    const char        *id;    /* اسم الـ reset line زي "ahb" أو "apb" */
    struct reset_control *rstc; /* الـ handle الفعلي — بيتملى بعد الـ get */
};
```

---

### 2. رسم العلاقات بين الـ Structs

```
                    ┌─────────────────────────────────────┐
                    │        reset_simple_data             │
                    │  ┌──────────────────────────────┐   │
                    │  │   reset_controller_dev rcdev │   │
                    │  │                              │   │
                    │  │  ops ──────────────────────► reset_control_ops
                    │  │  of_node ──────────────────► device_node (DT)
                    │  │  list ──────────────────────► [global controller list]
                    │  │  reset_control_head ────────► [reset_control handles]
                    │  │  nr_resets = size * 8        │   │
                    │  └──────────────────────────────┘   │
                    │  lock      (spinlock_t)              │
                    │  membase ──────────────────────────► MMIO registers
                    │  active_low = true                   │
                    └─────────────────────────────────────┘
                              ▲
                              │ container_of(rcdev, ...)
                              │
                    ┌─────────────────────┐
                    │  reset_simple_ops   │
                    │  .assert()          │──► read reg → clear/set bit → write reg
                    │  .deassert()        │──► read reg → set/clear bit → write reg
                    │  .reset()           │──► assert → udelay → deassert
                    │  .status()          │──► read bit → return
                    └─────────────────────┘

                    Consumer side:
                    ┌─────────────────────────────┐
                    │  reset_control_bulk_data[]   │
                    │  id = "ahb"                  │
                    │  rstc ───────────────────────┼──► reset_control
                    └─────────────────────────────┘         │
                                                            ▼
                                                   reset_controller_dev
                                                   (via internal linkage)
```

---

### 3. رسم الـ Lifecycle

#### Early Boot Path (sun6i_reset_init)

```
Kernel boot (start_kernel)
        │
        ▼
machine_init() / early_init_dt_scan()
        │
        ▼
sun6i_reset_init()           ← called from arch/arm/mach-sunxi/ early
        │
        ├── for_each_matching_node(np, sunxi_early_reset_dt_ids)
        │           matches "allwinner,sun6i-a31-ahb1-reset"
        │
        ▼
sunxi_reset_init(np)
        │
        ├── kzalloc(reset_simple_data)     ← allocate driver data
        │
        ├── of_address_to_resource(np)     ← get MMIO range from DT
        │
        ├── request_mem_region()           ← reserve physical address range
        │
        ├── ioremap()                      ← map to virtual address
        │         data->membase = virtual address
        │
        ├── spin_lock_init(&data->lock)    ← init spinlock
        │
        ├── data->rcdev.ops = &reset_simple_ops
        ├── data->rcdev.nr_resets = size * 8
        ├── data->rcdev.of_node = np
        ├── data->active_low = true
        │
        ▼
reset_controller_register(&data->rcdev)
        │
        ├── adds rcdev to global reset_controller_list
        └── now consumers can get reset handles via DT phandle
```

#### Consumer Usage Lifecycle

```
Driver probe()
    │
    ▼
reset_control_get_exclusive(dev, "ahb")
    │
    ├── __reset_control_get()
    │     └── finds reset_controller_dev via of_node lookup
    │     └── allocates reset_control handle
    │     └── links it to rcdev->reset_control_head
    │
    ▼
reset_control_deassert(rstc)         ← take peripheral out of reset
    │
    └── rcdev->ops->deassert(rcdev, id)
          └── reset_simple_deassert()
                └── spin_lock() → read reg → set bit → write reg → spin_unlock()
    │
    ▼
[device is now running]
    │
    ▼
Driver remove()
    │
    ▼
reset_control_assert(rstc)           ← put back in reset
reset_control_put(rstc)              ← release handle
```

#### devm variant (auto-cleanup)

```
devm_reset_control_get_exclusive(dev, id)
    │
    └── allocates devres action
    └── on driver detach → automatically calls reset_control_put()
```

---

### 4. Call Flow Diagrams

#### `sun6i_reset_init()` — Early Registration

```
sun6i_reset_init()
  └── for_each_matching_node(np, sunxi_early_reset_dt_ids)
        └── sunxi_reset_init(np)
              ├── kzalloc(sizeof(reset_simple_data))
              ├── of_address_to_resource(np, 0, &res)
              │     └── reads "reg" property from DT node
              ├── request_mem_region(res.start, size, np->name)
              ├── ioremap(res.start, size)
              │     └── data->membase = virtual mapped address
              ├── spin_lock_init(&data->lock)
              ├── data->rcdev.ops = &reset_simple_ops
              ├── data->rcdev.nr_resets = size * 8
              │     └── e.g. 4 bytes = 32 reset lines
              ├── data->rcdev.of_node = np
              ├── data->active_low = true
              └── reset_controller_register(&data->rcdev)
                    └── list_add(&rcdev->list, &reset_controller_list)
                    └── reset_controller_add_lookup()
```

#### `reset_control_deassert()` — Actual Reset Release

```
reset_control_deassert(rstc)
  └── reset_control_ops->deassert(rcdev, id)
        └── reset_simple_deassert(rcdev, id)
              ├── data = container_of(rcdev, reset_simple_data, rcdev)
              ├── reg = id / BITS_PER_LONG     ← which register
              ├── bit = id % BITS_PER_LONG     ← which bit in register
              ├── spin_lock_irqsave(&data->lock, flags)
              ├── val = readl(data->membase + reg * 4)
              ├── if (active_low)
              │     val |= BIT(bit)            ← set = deassert (active_low)
              │   else
              │     val &= ~BIT(bit)           ← clear = deassert
              ├── writel(val, data->membase + reg * 4)
              └── spin_unlock_irqrestore(&data->lock, flags)
```

#### `reset_control_assert()` — Put in Reset

```
reset_control_assert(rstc)
  └── reset_simple_assert(rcdev, id)
        ├── spin_lock_irqsave(&data->lock, flags)
        ├── val = readl(membase + reg * 4)
        ├── if (active_low)
        │     val &= ~BIT(bit)    ← clear = assert (active_low)
        │   else
        │     val |= BIT(bit)     ← set = assert
        ├── writel(val, membase + reg * 4)
        └── spin_unlock_irqrestore(&data->lock, flags)
```

#### DT Consumer Resolution

```
device driver calls:
  reset_control_get_exclusive(dev, "ahb")
    └── __reset_control_get(dev, "ahb", 0, EXCLUSIVE)
          └── of_parse_phandle_with_args(dev->of_node, "resets", "#reset-cells")
                └── finds phandle → sunxi reset controller DT node
          └── finds reset_controller_dev with matching of_node
          └── calls rcdev->of_xlate(rcdev, &args)
                └── of_reset_simple_xlate() → returns args.args[0] as ID
          └── allocates struct reset_control
          └── links to rcdev->reset_control_head
          └── returns reset_control handle
```

---

### 5. Locking Strategy

الـ locking في النظام ده بسيط وواضح:

#### الـ `spinlock_t lock` في `reset_simple_data`

| ما بيحميه | السبب |
|---|---|
| الـ read-modify-write على الـ MMIO registers | لو فيه أكتر من consumer بيعمل assert/deassert في نفس الوقت على نفس الـ register، ممكن يحصل data corruption |
| الـ `membase` register access | الـ register واحد بيحمل bits لأكتر من reset line |

**Lock ordering:**
```
spin_lock_irqsave(&data->lock, flags)
    ← irqsave لأن الكود ده ممكن يتنادى من interrupt context
    ← بيعطل الـ interrupts محلياً عشان ما يحصلش preemption
    ← read → modify → write
spin_unlock_irqrestore(&data->lock, flags)
```

**ليه `irqsave` وليس `spin_lock` عادي؟**
- الـ reset controllers ممكن يتنادوا من interrupt handlers (مثلاً لما جهاز بيعمل timeout ومحتاج reset طارئ).
- `irqsave` بيحفظ interrupt state ويعطلها، فمافيش deadlock لو interrupt جات في النص.

#### الـ Consumer Side (reset_control)

الـ reset core نفسه (في `drivers/reset/core.c`) بيستخدم mutex لحماية الـ `reset_control_head` list من concurrent get/put — ده منفصل عن الـ spinlock الخاص بالـ register access.

```
reset core mutex
    └── protects: reset_controller_list, reset_control_head lists
    └── used during: get, put, register, unregister

reset_simple_data->lock (spinlock)
    └── protects: MMIO register read-modify-write
    └── used during: assert, deassert, status
```

**Lock ordering rule:** ما فيش تداخل بينهم — الـ mutex بيتأخد أولاً في الـ core، وبعدين الـ spinlock بيتأخد في الـ ops. مافيش حالة بيتأخد فيها الـ spinlock ثم الـ mutex.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions والـ APIs — Cheatsheet

الملف `include/linux/reset/sunxi.h` نفسه بسيط جداً — بيعرّف function واحدة بس، لكن الفهم الحقيقي بييجي من implementation الموجودة في `drivers/reset/reset-sunxi.c` اللي بيستخدم البنية دي مع الـ `reset-simple` framework.

#### Category 1: Early Init (Public API — الهيدر)

| Function | الملف | الغرض |
|---|---|---|
| `sun6i_reset_init()` | `reset/sunxi.h` (declared), `reset-sunxi.c` (defined) | Early boot initialization للـ AHB1 reset controller على sun6i (A31) |

#### Category 2: Internal / Static Helpers (في الـ .c)

| Function | الغرض |
|---|---|
| `sunxi_reset_init(np)` | الـ workhorse الفعلي — بيعمل map للـ registers وبيسجل الـ reset controller |

#### الـ Data Structures المستخدمة

| Struct | الملف | الدور |
|---|---|---|
| `reset_simple_data` | `reset/reset-simple.h` | بيحوي الـ `membase`, `lock`, `rcdev`, `active_low` |
| `reset_controller_dev` | `reset-controller.h` | الـ base struct للـ reset subsystem |
| `device_node` | `of.h` | بيمثل node من الـ Device Tree |
| `resource` | `ioport.h` | بيمثل memory region |

---

### Group 1: Early Initialization Functions

#### الغرض من المجموعة

الـ sun6i (Allwinner A31) محتاج الـ reset controller لـ AHB1 يتسجل **قبل** ما الـ driver model ياخد دوره في الـ boot sequence. ده لأن devices كتير على AHB1 bus محتاجة تخرج من الـ reset عشان الـ kernel يقدر يشوفها أصلاً. الحل هو early init يتم عن طريق `__init` function تتنادى بدري من الـ machine setup.

---

### `sun6i_reset_init` — Public API (المعلَن في الهيدر)

```c
void __init sun6i_reset_init(void);
```

**شرح:**
الـ function دي هي الـ entry point الوحيدة المُعلَنة في الهيدر. بتعمل iterate على كل الـ Device Tree nodes اللي بتتطابق مع الـ compatible string `"allwinner,sun6i-a31-ahb1-reset"` وبتنادي `sunxi_reset_init()` على كل واحدة. الـ `__init` annotation بتقول للـ kernel إن الكود ده ممكن يتحرر من الذاكرة بعد انتهاء الـ boot.

**Parameters:** لا يوجد — بتعتمد على الـ Device Tree لاكتشاف الـ hardware.

**Return value:** `void` — أي failure في الـ internal init بيتم تجاهله بـ silent fail (لأنها early init وماتوقعش exception handling).

**Key Details:**
- **Locking:** لا يوجد — الـ `__init` code بيشتغل single-threaded في وقت الـ boot.
- **Caller context:** بيتنادى من الـ machine-level setup code أو من `__initcall` قبل ما الـ driver core يبدأ.
- **Side effects:** بتسجل الـ reset controller في الـ global reset subsystem — بعدها أي driver يعمل `reset_control_get()` على device تحت AHB1 بيقدر يلاقيه.

**Pseudocode Flow:**
```
sun6i_reset_init():
    for each DT node matching "allwinner,sun6i-a31-ahb1-reset":
        sunxi_reset_init(node)
```

---

### `sunxi_reset_init` — Internal Worker (static, لا يُعلَن في الهيدر)

```c
static int sunxi_reset_init(struct device_node *np);
```

**شرح:**
ده الـ workhorse الفعلي. بيعمل allocate لـ `reset_simple_data`، بيحول الـ DT reg property لـ physical address عن طريق `of_address_to_resource()`، بيعمل `request_mem_region()` للـ I/O range، وبعدين بيعمل `ioremap()` عشان يقدر يكتب على الـ registers. أخيراً بيحوّل العدد الكلي للـ reset lines من حجم الـ register range (`size * 8` — كل bit هو reset line واحد) وبيسجل الـ controller مع الـ reset core.

**Parameters:**

| Parameter | النوع | الوصف |
|---|---|---|
| `np` | `struct device_node *` | الـ DT node الخاص بالـ reset controller — بيجيب منه الـ reg address وغيره |

**Return value:**
- `0` عند النجاح
- `-ENOMEM` لو `kzalloc()` فشلت
- `-EBUSY` لو `request_mem_region()` فشلت (المنطقة محجوزة من driver تاني)
- قيمة سالبة من `of_address_to_resource()` لو الـ DT node مش فيها `reg` property صح

**Key Details:**

**Locking:**
- `spin_lock_init(&data->lock)` — بيعمل init للـ spinlock اللي الـ `reset_simple_ops` بيستخدمه في كل assert/deassert operation عشان يحمي الـ read-modify-write sequence على الـ registers.

**Error Path:**
- الـ error path الوحيد هو `err_alloc` label — بيعمل `kfree(data)` فقط. لو الـ `ioremap` نجحت وبعدين `reset_controller_register()` فشلت، فيه memory leak محتمل للـ `ioremap()` — ده documented shortcoming في early init code.

**الـ `active_low = true` — نقطة محورية:**
الـ Allwinner SoCs بتستخدم **active-low reset bits** — يعني الـ bit بيتكلير (`0`) عشان يعمل assert للـ reset، وبيتسيت (`1`) عشان يعمل deassert. الـ `reset_simple_ops` بيحترم الـ flag ده في كل operation.

**الـ `nr_resets = size * 8`:**
الـ register range بيتحول لعدد الـ reset lines بضرب الحجم في 8 (كل byte = 8 bits = 8 reset lines). على A31 الـ AHB1 reset registers بتغطي ranges متعددة.

**Pseudocode Flow:**
```
sunxi_reset_init(np):
    data = kzalloc(sizeof(*data))          // allocate driver data
    if !data: return -ENOMEM

    ret = of_address_to_resource(np, 0, &res)  // parse "reg" from DT
    if ret: goto err_alloc

    size = resource_size(&res)

    if !request_mem_region(res.start, size, np->name):
        ret = -EBUSY
        goto err_alloc

    data->membase = ioremap(res.start, size)   // map registers
    if !data->membase:
        ret = -ENOMEM
        goto err_alloc

    spin_lock_init(&data->lock)

    data->rcdev.owner    = THIS_MODULE
    data->rcdev.nr_resets = size * 8           // 1 bit = 1 reset line
    data->rcdev.ops      = &reset_simple_ops   // assert/deassert via simple ops
    data->rcdev.of_node  = np
    data->active_low     = true                // Allwinner: clear bit = assert

    return reset_controller_register(&data->rcdev)

err_alloc:
    kfree(data)
    return ret
```

---

### Group 2: الـ reset_simple_ops — Runtime Operations

الـ `reset_simple_ops` مش معرفة في الهيدر موضوع الدراسة، لكنها محورية في فهم الـ system. الـ `sunxi_reset_init` بتربط الـ controller بيها عن طريق `data->rcdev.ops = &reset_simple_ops`.

```c
extern const struct reset_control_ops reset_simple_ops;
/* defined in drivers/reset/reset-simple.c */
```

**الـ ops الفعلية اللي بتتنفذ على كل reset line:**

| Op | السلوك على Allwinner (`active_low=true`) |
|---|---|
| `.assert` | بيعمل `readl()` ثم `writel()` مع **clear** الـ bit (spinlock-protected) |
| `.deassert` | بيعمل `readl()` ثم `writel()` مع **set** الـ bit (spinlock-protected) |
| `.reset` | assert + delay + deassert |
| `.status` | بيقرأ الـ bit — لو `0` يعني في reset (`status_active_low`) |

**الـ bit index** بيتحسب من الـ reset line ID:
```
reg_offset = (id / 32) * 4    /* كل register بيحوي 32 bit */
bit_mask   = BIT(id % 32)
```

---

### Group 3: الـ DT Matching Table

```c
static const struct of_device_id sunxi_early_reset_dt_ids[] __initconst = {
    { .compatible = "allwinner,sun6i-a31-ahb1-reset", },
    { /* sentinel */ },
};
```

**شرح:**
الـ table دي بتتعمل `__initconst` — يعني بتتحط في section خاصة بالـ init data اللي بتتحرر بعد الـ boot. الـ `for_each_matching_node()` macro بيعمل iterate على كل الـ DT tree ويرجع كل node بيتطابق compatible string بتاعتها مع أي entry في الـ table.

**ليه بس A31 في الـ early table؟**
الـ SoCs التانية اللي بتدعم `reset-simple` بتتسجل عادي عن طريق `platform_driver` في الـ driver model. الـ A31 محتاج early init لأن AHB1 reset controller بيتحكم في modules إن reset منهم هيمنع الـ kernel من الـ boot حتى.

---

### مخطط تدفق الـ Boot

```
Machine Init
    │
    ▼
sun6i_reset_init()                     ← called via early_initcall or machine setup
    │
    ├── for_each_matching_node(sunxi_early_reset_dt_ids)
    │       │
    │       └── sunxi_reset_init(np)
    │               │
    │               ├── kzalloc(reset_simple_data)
    │               ├── of_address_to_resource()   → physical address من DT
    │               ├── request_mem_region()        → reserve I/O range
    │               ├── ioremap()                   → virtual address للـ registers
    │               ├── spin_lock_init()
    │               ├── rcdev.nr_resets = size * 8
    │               ├── rcdev.ops = &reset_simple_ops
    │               └── reset_controller_register() → يُضاف للـ global list
    │
    ▼
Drivers boot → reset_control_get() → assert/deassert via reset_simple_ops
```

---

### ملاحظات على الـ API الموجودة في الهيدر

الهيدر `include/linux/reset/sunxi.h` نفسه minimalist عمداً:

```c
/* SPDX-License-Identifier: GPL-2.0 */
#ifndef __LINUX_RESET_SUNXI_H__
#define __LINUX_RESET_SUNXI_H__

void __init sun6i_reset_init(void);

#endif
```

**`__init` annotation:**
بيخلي الـ linker يحط الـ function في `.init.text` section. بعد انتهاء الـ init sequence، الـ kernel بيعمل free لكل الـ `.init.*` sections من الذاكرة. أي call لـ `sun6i_reset_init()` بعد كده = undefined behavior (crash محتمل).

**الـ include guard `__LINUX_RESET_SUNXI_H__`:**
Standard practice — بيمنع double-inclusion في حالة الـ headers المتداخلة.

---

### جدول مقارنة Early Init vs. Platform Driver Init

| البُعد | Early Init (`sun6i_reset_init`) | Platform Driver (`reset-simple`) |
|---|---|---|
| التوقيت | قبل `device_initcall` | عند `device_initcall` أو أحدث |
| الـ DT matching | يدوي عبر `for_each_matching_node` | تلقائي عبر `platform_driver.of_match_table` |
| الـ cleanup | لا يوجد (no `remove`) | `platform_driver.remove` |
| الـ memory lifetime | `__init` + يتحرر بعد boot | يبقى طول عمر الـ driver |
| الـ use case | AHB1 reset على A31 — critical early | باقي الـ controllers على SoCs أخرى |
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem اللي بنتعامل معاه هو **Allwinner (sun6i) Reset Controller** — درايفر بسيط بيتسجل early في الـ boot عن طريق `sun6i_reset_init()` قبل ما الـ device model يشتغل. الـ implementation مبنية على `reset_simple_ops` و `reset_simple_data`، والـ registers بتتعامل معاها بـ read-modify-write محمي بـ spinlock.

---

### Software Level

#### 1. debugfs entries

الـ reset framework بيكشف entries تحت `/sys/kernel/debug/reset/`:

```bash
# اعرض كل الـ reset controllers المسجلة
ls /sys/kernel/debug/reset/

# اقرأ حالة reset line رقم 5 مثلاً (0 = deasserted, 1 = asserted)
cat /sys/kernel/debug/reset/allwinner\,sun6i-a31-ahb1-reset/5/status

# اعرض عدد مرات assert/deassert لكل line
cat /sys/kernel/debug/reset/allwinner\,sun6i-a31-ahb1-reset/5/assert_count
cat /sys/kernel/debug/reset/allwinner\,sun6i-a31-ahb1-reset/5/deassert_count
```

**تفسير الـ output:**
- **status = 1** مع `active_low = true` (حالة sun6i) يعني الـ bit محلوط (cleared) في الـ register ← الجهاز في حالة reset.
- **status = 0** يعني الـ bit مرفوع ← الجهاز شغال طبيعي.

#### 2. sysfs entries

الـ reset controller نفسه ما بيكشفش entries مباشرة في sysfs، لكن الـ devices اللي بتستخدم الـ reset بتظهر:

```bash
# اعرض كل الـ platform devices في الـ AHB1 bus على sun6i
ls /sys/bus/platform/devices/ | grep sun6i

# تحقق إن الـ driver اتعرف على الـ reset controller
cat /sys/bus/platform/drivers/reset-simple/*/uevent

# اعرض عدد الـ resets المتاحة في الـ controller
cat /sys/class/reset_controller/*/nr_resets
```

#### 3. ftrace — tracepoints/events

```bash
# فعّل tracing على reset controller operations
echo 1 > /sys/kernel/debug/tracing/events/reset/enable

# أو بشكل انتقائي
echo 1 > /sys/kernel/debug/tracing/events/reset/reset_control_assert/enable
echo 1 > /sys/kernel/debug/tracing/events/reset/reset_control_deassert/enable
echo 1 > /sys/kernel/debug/tracing/events/reset/reset_control_reset/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace

# تصفية على sun6i بس
echo 'name == "allwinner,sun6i-a31-ahb1-reset"' > \
    /sys/kernel/debug/tracing/events/reset/reset_control_assert/filter
```

**مثال output:**
```
     kworker/0:1-45    [000] ....  1234.567: reset_control_assert: \
         allwinner,sun6i-a31-ahb1-reset:5
     kworker/0:1-45    [000] ....  1234.600: reset_control_deassert: \
         allwinner,sun6i-a31-ahb1-reset:5
```
الـ timestamp بين assert وdeassert لازم يكون >= `reset_us` (في حالة sun6i عادةً 0 يعني no delay enforced).

#### 4. printk / dynamic debug

الدرايفر `reset-sunxi.c` ما عنده `dev_dbg` كتير لأنه early-init، لكن ممكن تفعّل dynamic debug للـ reset framework كله:

```bash
# فعّل كل debug messages في reset subsystem
echo 'file reset-sunxi.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file reset-simple.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file reset.c +p'        > /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل module
echo 'module reset-simple +p' > /sys/kernel/debug/dynamic_debug/control

# تحقق إيه اللي مفعّل
cat /sys/kernel/debug/dynamic_debug/control | grep reset
```

في الـ kernel command line للـ early boot:
```
dyndbg="file reset-sunxi.c +p; file reset-simple.c +p"
```

#### 5. Kernel config options للـ debugging

| Config | الوصف |
|--------|-------|
| `CONFIG_RESET_CONTROLLER` | لازم يكون مفعّل عشان الـ subsystem يشتغل أصلاً |
| `CONFIG_DEBUG_FS` | عشان `/sys/kernel/debug/reset/` تظهر |
| `CONFIG_DYNAMIC_DEBUG` | لتفعيل `dev_dbg` وقت التشغيل |
| `CONFIG_TRACING` | لـ ftrace reset events |
| `CONFIG_RESET_SUNXI` | الدرايفر نفسه — early init path |
| `CONFIG_DEBUG_SPINLOCK` | لاكتشاف spinlock bugs في read-modify-write |
| `CONFIG_PROVE_LOCKING` | lockdep للـ spinlock في `reset_simple_data.lock` |
| `CONFIG_DEBUG_ATOMIC_SLEEP` | لو حد استخدم الـ reset من context ممنوع |
| `CONFIG_KALLSYMS` | تحسين stack traces |
| `CONFIG_FRAME_POINTER` | stack traces أوضح |
| `CONFIG_EARLY_PRINTK` | مهم جداً لأن `sun6i_reset_init()` بتشتغل قبل serial driver |

#### 6. أدوات خاصة بالـ subsystem

الـ sun6i reset ما عنده devlink support لأنه مش network device. لكن في أدوات مفيدة:

```bash
# اعرض كل الـ reset controllers عبر sysfs
find /sys -name "nr_resets" -exec echo {} \; -exec cat {} \;

# تحقق إن الـ OF node اتعرف صح
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    grep -A5 "ahb1-reset"

# استخدم reset-simple driver info
cat /sys/bus/platform/devices/*/driver_override 2>/dev/null

# تحقق من الـ memory region اللي اتحجزت
cat /proc/iomem | grep -i "ahb1\|reset"
```

#### 7. رسائل الـ error الشائعة

| رسالة الـ kernel log | المعنى | الحل |
|----------------------|--------|-------|
| `of_address_to_resource failed` | الـ DT node ما عندوش `reg` property صح | تحقق من DT: لازم `reg = <base size>;` |
| `request_mem_region failed` | منطقة الـ memory محجوزة من درايفر تاني | ابحث عن تعارض في `/proc/iomem` |
| `ioremap failed` | فشل الـ virtual memory mapping | ذاكرة قليلة أو عنوان غلط في DT |
| `reset_controller_register failed` | فشل تسجيل الـ controller | تحقق من `nr_resets` وحالة الـ kernel |
| `reset control -EBUSY` | محاولة get exclusive reset مستخدمة بالفعل | consumer تاني شايل الـ reset — راجع `deassert_count` |
| `reset: -ENOTSUPP` | `CONFIG_RESET_CONTROLLER` مش مفعّل | أعد compile الـ kernel |
| `failed to get reset` في consumer driver | الـ reset line مش معرّفة في DT | أضف `resets = <&ahb1_rst N>;` في DT للـ device |

#### 8. نقاط `dump_stack()` / `WARN_ON()` الاستراتيجية

في `reset-sunxi.c` وفي `reset-simple.c`، النقاط المثالية:

```c
/* في sunxi_reset_init() — بعد of_address_to_resource */
if (ret) {
    pr_err("%s: failed to get resource: %d\n", np->name, ret);
    dump_stack();   /* نشوف مين استدى init في أي وقت */
    goto err_alloc;
}

/* في reset_simple_ops.assert/deassert — قبل writel */
WARN_ON(!spin_is_locked(&data->lock));  /* تحقق من الـ lock */

/* لو active_low جاي غلط */
WARN_ON(data->active_low && data->status_active_low);
/* sun6i: active_low=true, status_active_low=false — هو الصح */

/* في sun6i_reset_init() — تحقق إن for_each_matching_node لقى حاجة */
if (!found_any)
    WARN(1, "sun6i: no AHB1 reset controller found in DT!\n");
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ hardware تطابق الـ kernel

الـ sun6i AHB1 reset registers: كل bit يمثل reset line واحدة. `active_low = true` يعني:
- **bit = 1** → جهاز شغال (deasserted)
- **bit = 0** → جهاز في reset (asserted)

```bash
# احسب عنوان الـ register من DT
# sun6i A31: AHB1 reset @ 0x01C20240
RESET_BASE=0x01C20240

# اقرأ الـ register الأول (reset lines 0-31)
devmem2 0x01C20240 w

# اقرأ الـ register التاني (reset lines 32-63)
devmem2 0x01C20244 w

# قارن بـ kernel state:
cat /sys/kernel/debug/reset/allwinner\,sun6i-a31-ahb1-reset/0/status
```

**مثال للتفسير:**
```
devmem2 0x01C20240 w
Value at address 0x01C20240 (0xf0870d00): 0xBFFFFFF7
```
Binary: `1011 1111 1111 1111 1111 1111 1111 0111`
- Bit 3 = 0 → reset line 3 في حالة reset (asserted)
- Bit 30 = 0 → reset line 30 في حالة reset
- باقي الـ bits = 1 → deasserted (شغال)

#### 2. Register dump techniques

```bash
# باستخدام devmem2 (لازم مثبّت)
for i in 0 4 8 12 16; do
    addr=$(printf "0x%08X" $((0x01C20240 + i)))
    echo -n "Offset +$i: "
    devmem2 $addr w 2>/dev/null | grep "Value"
done

# باستخدام /dev/mem مباشرة (تحتاج CONFIG_DEVMEM=y)
dd if=/dev/mem bs=4 count=8 skip=$((0x01C20240/4)) 2>/dev/null | \
    xxd

# باستخدام io utility (من package i2c-tools)
io -4 -r 0x01C20240
io -4 -r 0x01C20244
```

**ملاحظة:** على kernels حديثة ممكن `CONFIG_STRICT_DEVMEM` يمنع القراءة — في الحالة دي استخدم الـ debugfs أو اعمل kernel module بسيط.

#### 3. Logic analyzer / oscilloscope tips

الـ reset lines في sun6i هي **GPIO-multiplexed** في بعض الأحيان، لكن معظمها داخلية:

- **الـ reset الخارجية** زي `nRST` على الـ SoC: قيس بـ oscilloscope على الـ pin المناسب.
- **الـ timing:** بعد assert، لازم تستنى minimum pulse width — sun6i المانيوال بيقول عادةً >= 1μs.
- **الـ glitch:** لو شايف reset متكرر غير متوقع، قيس على الـ power rails (VCC-IO) — الـ brownout بيعمل false resets.

```
        VDDIO
          |
     _____|_____
    |           |
    | Device    |--- nRST pin (scope here)
    |___________|
          |
         GND

Expected reset waveform:
    ___         ___________
       |_______|
       ^       ^
    assert  deassert
       |<----->|
      >= 1μs (check SoC TRM)
```

#### 4. مشاكل الـ hardware الشائعة وأنماطها في الـ kernel log

| المشكلة | النمط في الـ log | التشخيص |
|---------|-----------------|---------|
| الـ reset line مش واصلة فعلياً للجهاز | الـ device يفشل في الـ probe رغم deassert | scope على pin الـ reset الخاص بالـ chip |
| clock لازم يشتغل قبل الـ reset يتعمل | `timeout waiting for device` بعد deassert | تأكد إن الـ clock مفعّل قبل ما تعمل deassert |
| power sequencing غلط | `device not responding` فور الـ probe | راجع الـ TRM — بعض الأجهزة تحتاج VDD قبل reset |
| short على الـ reset line | الـ register بيتكتب بس الـ device مش بيرد | قيس resistance بين pin الـ reset والـ GND |
| خلل في الـ SoC AHB1 bus | kernel panic في early boot قبل ما console يشتغل | فعّل `earlyprintk` وراجع الـ UART output |

#### 5. Device Tree debugging

```bash
# تحقق إن الـ DT node موجود وصح
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    grep -B2 -A10 "ahb1-reset"

# التوقعات الصحيحة لـ sun6i AHB1 reset:
# compatible = "allwinner,sun6i-a31-ahb1-reset"
# reg = <0x01c20240 0xc>;  ← 3 registers = 24 reset lines (حسب الـ SoC)
# #reset-cells = <1>;

# تحقق إن الـ consumer device عندها الـ resets property
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    grep -B5 -A15 "mmc0\|emac\|usb"  | grep -A3 "resets"

# تحقق إن phandle الـ reset controller صح
# الـ consumer لازم يشاور على نفس node
cat /sys/firmware/devicetree/base/soc/reset-controller@1c20240/compatible
```

**DT مثال صح:**
```dts
ahb1_rst: reset@1c20240 {
    compatible = "allwinner,sun6i-a31-ahb1-reset";
    reg = <0x01c20240 0xc>;
    #reset-cells = <1>;
};

/* consumer */
mmc0: mmc@1c0f000 {
    ...
    resets = <&ahb1_rst 8>;  /* reset line 8 = MMC0 */
};
```

لو `#reset-cells` مش موجود أو = 0، الـ kernel هيطلع:
```
mmc0: failed to get reset: -EINVAL
```

---

### Practical Commands

#### مجموعة الأوامر الكاملة — جاهزة للنسخ

```bash
#!/bin/bash
# === sun6i Reset Controller Debug Script ===

RESET_BASE=0x01C20240  # sun6i A31 AHB1 reset base

echo "=== 1. Reset Controllers في الـ System ==="
find /sys -name "nr_resets" 2>/dev/null | while read f; do
    echo "$f: $(cat $f)"
done

echo ""
echo "=== 2. قراءة الـ Registers مباشرة ==="
for offset in 0 4 8; do
    addr=$(printf "0x%08X" $((RESET_BASE + offset)))
    echo -n "Register @ $addr: "
    devmem2 $addr w 2>/dev/null | grep "Value" || echo "N/A (جرب عشان تشغّل root)"
done

echo ""
echo "=== 3. حالة كل Reset Lines في الـ Kernel ==="
RCDEV_PATH="/sys/kernel/debug/reset"
if [ -d "$RCDEV_PATH" ]; then
    for ctrl in "$RCDEV_PATH"/*/; do
        echo "Controller: $(basename $ctrl)"
        for line in "$ctrl"*/; do
            id=$(basename $line)
            status=$(cat "$line/status" 2>/dev/null)
            echo "  Line $id: status=$status"
        done
    done
else
    echo "debugfs مش متاح — فعّل CONFIG_DEBUG_FS"
fi

echo ""
echo "=== 4. تفعيل ftrace لـ Reset Events ==="
TRACE=/sys/kernel/debug/tracing
echo 0 > $TRACE/tracing_on
echo > $TRACE/trace
echo 1 > $TRACE/events/reset/enable 2>/dev/null || \
    echo "reset tracepoints غير متاحة"
echo 1 > $TRACE/tracing_on
echo "Trace مفعّل — شغّل الجهاز اللي بتـ debug عليه، وبعدين:"
echo "  cat $TRACE/trace"

echo ""
echo "=== 5. dynamic debug للـ reset subsystem ==="
DD=/sys/kernel/debug/dynamic_debug/control
echo 'file reset-sunxi.c +p' > $DD 2>/dev/null && \
    echo "reset-sunxi.c debug: ON" || \
    echo "dynamic_debug غير متاح"
echo 'file reset-simple.c +p' > $DD 2>/dev/null
echo 'file reset.c +p' > $DD 2>/dev/null

echo ""
echo "=== 6. تحقق من DT ==="
dtc -I fs /sys/firmware/devicetree/base 2>/dev/null | \
    grep -A8 "ahb1-reset" | head -20 || \
    echo "dtc غير متاح — install device-tree-compiler"

echo ""
echo "=== 7. /proc/iomem للتحقق من Memory Mapping ==="
grep -i "reset\|ahb1\|01c202" /proc/iomem || echo "لا توجد entries مطابقة"

echo ""
echo "=== 8. kernel log للـ Reset Controller ==="
dmesg | grep -i "reset\|sun6i\|ahb1" | tail -30
```

#### تفسير مثال output كامل

```
=== 2. قراءة الـ Registers مباشرة ===
Register @ 0x01C20240: Value at address 0x01C20240: 0xBFFFFFF7

# Binary: 1011 1111 1111 1111 1111 1111 1111 0111
# active_low=true في sun6i:
#   bit=0 → reset asserted (جهاز في reset)
#   bit=1 → reset deasserted (جهاز شغال)
#
# Bit 3  = 0 → Reset line 3  asserted  ← هنا مشكلة لو المفروض الجهاز يشتغل
# Bit 30 = 0 → Reset line 30 asserted  ← تحقق ايه الجهاز على line 30
# باقي bits = 1 → OK
```

```bash
# لو لقيت line عليها reset غير متوقع — فك الـ reset يدوياً للاختبار
# line 3 = bit 3 في الـ register الأول
# active_low → اكتب 1 في الـ bit عشان تعمل deassert

OLD=$(devmem2 0x01C20240 w | grep Value | awk '{print $NF}')
NEW=$(printf "0x%08X" $(( OLD | (1 << 3) )))
devmem2 0x01C20240 w $NEW
echo "كتبنا $NEW في الـ register — تحقق إن الجهاز اشتغل"
```

```bash
# تحقق من reset بعد system reboot — هل الـ early init اشتغل؟
dmesg | grep -E "sun6i|ahb1|reset.*init|reset.*register"
# النتيجة المتوقعة:
# [    0.000123] reset-simple: registered allwinner,sun6i-a31-ahb1-reset
# لو مفيش → الـ DT compatible string غلط أو الـ driver مش compiled
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Android TV Box بـ Allwinner H616 — الـ USB مش شغال من البداية

#### العنوان
**الـ USB controller مش بيتعرف** على بورد Android TV box بـ Allwinner H616

#### السياق
شركة بتعمل Android TV box رخيصة، استخدمت SoC جديد من Allwinner (H616). البورد بتبوت بس الـ USB ports فاضية — مفيش keyboard، مفيش USB storage. المنتج المفروض يتشحن الأسبوع الجاي.

#### المشكلة
الـ USB controller محتاج reset قبل ما يتهيأ. الـ reset controller للـ AHB1 bus مش اتسجل وقت boot لأن الـ DT entry غلط أو ناقص، فالـ USB driver بيفضل يرجع `-EPROBE_DEFER` للأبد.

```bash
# الـ dmesg بيظهر
[    2.341] usb_phy_generic: probe of usb_phy failed with error -517
[    2.342] dwc2 5100000.usb: probe deferred
```

#### التحليل

الـ boot flow على sun6i/sun8i بيمر بالترتيب ده:

```
arch/arm/mach-sunxi/sunxi.c
  └── sun6i_timer_init()          ← init_time hook
        ├── of_clk_init(NULL)
        ├── sun6i_reset_init()    ← من sunxi.h
        └── timer_probe()
```

الـ `sun6i_reset_init()` في `reset-sunxi.c` بتعمل:

```c
void __init sun6i_reset_init(void)
{
    struct device_node *np;

    /* بتدور على كل node متوافق مع allwinner,sun6i-a31-ahb1-reset */
    for_each_matching_node(np, sunxi_early_reset_dt_ids)
        sunxi_reset_init(np);   /* بتسجل reset controller مبكراً */
}
```

لو الـ DT مش فيه node بالـ compatible الصح، الـ `for_each_matching_node` مش هيلاقي حاجة، والـ reset controller مش هيتسجل أبداً. كل driver تاني بعد كده بيطلب reset وبيلاقي ما فيش controller → `-EPROBE_DEFER` أو `-ENODEV`.

#### الحل

**1. تأكد من الـ DT node:**

```dts
/* arch/arm64/boot/dts/allwinner/sun50i-h616.dtsi */
ahb1_rst: reset@1c202c0 {
    compatible = "allwinner,sun6i-a31-ahb1-reset";
    reg = <0x01c202c0 0xc>;
    #reset-cells = <1>;
};
```

**2. تأكد إن الـ board DT بيستخدمه:**

```dts
&usb_otg {
    resets = <&ahb1_rst 0>;
    reset-names = "ahb";
    status = "okay";
};
```

**3. تحقق بـ debugfs:**

```bash
ls /sys/bus/platform/drivers/reset-simple/
cat /sys/kernel/debug/reset/allwinner*
```

#### الدرس المستفاد
الـ `sun6i_reset_init()` بتشتغل early جداً — قبل الـ device model. لو الـ compatible string في الـ DT غلط بحرف واحد، الـ `for_each_matching_node` مش هيلاقي الـ node وكل الـ peripherals اللي على الـ AHB1 bus هتفشل بصمت.

---

### السيناريو 2: بورد Bring-Up جديد بـ Allwinner A31 — الـ kernel يـ panic وقت boot

#### العنوان
**Kernel panic** في `sun6i_timer_init` على custom board بـ A31

#### السياق
مهندس بيعمل bring-up لـ custom industrial gateway بـ Allwinner A31 (sun6i). الـ kernel compile تمام، بس البورد بتـ panic قبل ما تظهر حاجة على الـ serial console.

#### المشكلة
الـ `sun6i_timer_init` بتستدعي `sun6i_reset_init()` اللي بتستدعي `sunxi_reset_init(np)` اللي بتعمل `ioremap`. لو الـ reg في الـ DT بيشاور على address غلط أو مش موجود في الـ address space، الـ `of_address_to_resource` بيرجع error وبيعمل `kfree` وبيرجع — بس الـ timer probe بعد كده بيفضل يحاول يـ access الـ reset controller اللي مش موجود.

```c
static int sunxi_reset_init(struct device_node *np)
{
    ...
    ret = of_address_to_resource(np, 0, &res);
    if (ret)
        goto err_alloc;   /* بيـ kfree ويرجع ret */
    ...
    if (!request_mem_region(res.start, size, np->name)) {
        ret = -EBUSY;
        goto err_alloc;   /* نفس الموضوع */
    }
    ...
}
```

الـ `sun6i_reset_init()` مش بتتحقق من الـ return value:

```c
void __init sun6i_reset_init(void)
{
    struct device_node *np;

    for_each_matching_node(np, sunxi_early_reset_dt_ids)
        sunxi_reset_init(np);   /* الـ return value اتجاهل! */
}
```

لو الـ timer driver نفسه احتاج reset → NULL pointer dereference → panic.

#### التحليل

```
sun6i_timer_init()
  ├── of_clk_init()         ✓
  ├── sun6i_reset_init()    ← بتفشل بصمت
  └── timer_probe()
        └── timer_driver_probe()
              └── reset_control_get()  ← returns ERR_PTR
                    └── *reset → deref → PANIC
```

#### الحل

**1. تحقق من الـ DT address:**

```bash
# من U-Boot أو early serial
md.l 0x01c202c0 4   # شوف لو الـ registers موجودة
```

**2. صحح الـ DT:**

```dts
ahb1_rst: reset@1c202c0 {
    compatible = "allwinner,sun6i-a31-ahb1-reset";
    reg = <0x01c202c0 0xc>;   /* تأكد من الـ size صح */
    #reset-cells = <1>;
};
```

**3. لو عايز تـ debug early panic، فعّل:**

```
CONFIG_EARLY_PRINTK=y
CONFIG_DEBUG_LL=y
earlycon=uart8250,mmio32,0x01c28000
```

#### الدرس المستفاد
الـ `__init` functions على الـ Sunxi platform بتشتغل في وقت حرج جداً. أي فشل فيها بيتاهل بصمت لأن الـ error handling في `sun6i_reset_init()` خارجي — المهندس لازم يتأكد من صحة الـ DT addresses من U-Boot قبل ما يـ boot الـ kernel.

---

### السيناريو 3: مقارنة مع RK3562 — ليه الـ approach مختلف؟

#### العنوان
مهندس جاي من **Rockchip RK3562** بيـ port kernel لـ Allwinner A31 وبيتعجب ليه الـ reset مش شغال بـ driver عادي

#### السياق
مهندس embedded اشتغل على RK3562 (Rockchip) اللي فيه reset controller بيتسجل عادي كـ platform driver. اتنقل لـ project تاني بـ Allwinner A31 ولاحظ إن الـ reset controller بيتسجل وقت `init_time` مش وقت `module_init`.

#### المشكلة
على RK3562، الـ reset controller بيتسجل متأخر نسبياً في الـ boot. على A31، الـ AHB1 reset controller لازم يكون موجود قبل الـ timer probe لأن الـ timer نفسه على الـ AHB1 bus ومحتاج reset.

```
RK3562 boot order:
  early_initcall → subsys_initcall → device_initcall (reset driver هنا)

A31 boot order:
  arch_initcall → .init_time → sun6i_timer_init → sun6i_reset_init
                                                    ↑ قبل كل initcalls
```

الـ file `sunxi.h` فيه declaration واحدة بس:

```c
void __init sun6i_reset_init(void);
```

الـ `__init` هنا مهم — الكود بيتحط في `.init.text` section اللي بتتحرر بعد الـ boot. ده بيعني مستحيل تستدعيها بعد كده.

#### التحليل

```c
/* من sunxi.c */
static void __init sun6i_timer_init(void)
{
    of_clk_init(NULL);

    /* لازم RESET_CONTROLLER مفعّل في الـ config */
    if (IS_ENABLED(CONFIG_RESET_CONTROLLER))
        sun6i_reset_init();    /* sunxi.h */

    timer_probe();   /* الـ timer محتاج الـ reset controller يكون جاهز */
}
```

الـ `IS_ENABLED(CONFIG_RESET_CONTROLLER)` guard مهم — لو حد عطّل الـ option ده في الـ config، الـ reset مش هيتسجل وكل حاجة على الـ AHB1 هتفشل.

#### الحل

**تأكد من الـ kernel config:**

```bash
grep RESET_CONTROLLER /boot/config-$(uname -r)
# لازم يكون:
# CONFIG_RESET_CONTROLLER=y
```

**لو بتبني kernel:**

```bash
# في .config أو defconfig
CONFIG_RESET_CONTROLLER=y
CONFIG_RESET_SIMPLE=y
```

**للمقارنة — على RK3562 الـ reset تاني:**

```bash
# RK3562: بيظهر كـ platform device
ls /sys/bus/platform/devices/ | grep reset

# A31: بيظهر في reset debugfs مبكراً
cat /sys/kernel/debug/reset/
```

#### الدرس المستفاد
الـ architecture الخاصة بـ Allwinner sun6i تستلزم early reset registration لأن الـ AHB1 bus نفسه بيحتاج reset قبل الـ timer. الـ header `sunxi.h` بيعبّر عن ده بوضوح بـ `__init` annotation — الـ function دي مش عادية وتوقيتها مش قابل للتغيير.

---

### السيناريو 4: IoT Sensor Board بـ Allwinner V3s — الـ SPI مش بيشتغل

#### العنوان
**SPI flash** مش بيتعرف على IoT sensor board بـ Allwinner V3s (sun8i)

#### السياق
شركة بتصنع IoT sensor node صغير بـ Allwinner V3s. الـ board عندها SPI NOR flash بـ 16MB للـ firmware. بعد migration من kernel 4.x لـ 6.x، الـ flash بطلت تتعرف.

#### المشكلة
في kernel الجديد، الـ SPI driver بيطلب reset control صح. الـ DT القديم مش فيه `resets` property للـ SPI node. بالإضافة، الـ `sun6i_timer_init` بتُستدعى لـ sun8i كمان:

```c
/* من sunxi.c */
DT_MACHINE_START(SUN8I_DT, "Allwinner sun8i Family")
    .init_time = sun6i_timer_init,   /* نفس الـ function! */
    .dt_compat = sun8i_board_dt_compat,
MACHINE_END
```

فالـ `sun6i_reset_init()` بتشتغل على V3s — بس لو الـ DT مش فيه node بـ compatible `allwinner,sun6i-a31-ahb1-reset`، الـ reset controller مش هيتسجل.

#### التحليل

```
sun8i (V3s) boot:
  sun6i_timer_init()
    └── sun6i_reset_init()
          └── for_each_matching_node(sunxi_early_reset_dt_ids)
                → compatible: "allwinner,sun6i-a31-ahb1-reset"
                → لو مش موجود في DT → مفيش iteration → مفيش registration

  SPI driver probe:
    └── reset_control_get("spi")
          → -ENODEV   ← لأن الـ controller مش موجود
```

```bash
# الـ dmesg:
[    1.823] spi-sun6i 1c69000.spi: Failed to get reset control
[    1.824] spi-sun6i 1c69000.spi: probe of 1c69000.spi failed with error -19
```

#### الحل

**1. أضف الـ reset node للـ V3s DTS:**

```dts
/* arch/arm/boot/dts/sun8i-v3s.dtsi */
ccu: clock@1c20000 {
    compatible = "allwinner,sun8i-v3s-ccu";
    ...
};

/* الـ AHB reset لازم يكون present */
ahb_rst: reset@1c202c0 {
    compatible = "allwinner,sun6i-a31-ahb1-reset";
    reg = <0x01c202c0 0xc>;
    #reset-cells = <1>;
};
```

**2. اربط الـ SPI بالـ reset:**

```dts
&spi0 {
    resets = <&ahb_rst 20>;   /* SPI0 reset bit */
    status = "okay";

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <50000000>;
    };
};
```

**3. تحقق بعد الـ boot:**

```bash
# لو الـ reset controller اتسجل صح
cat /sys/kernel/debug/reset/reset20
# يرجع: 0 (deasserted) أو 1 (asserted)
```

#### الدرس المستفاد
الـ `sun6i_reset_init()` بتشتغل على sun8i كمان (مش بس sun6i) — الاسم ممكن يكون مضلل. أي board بيستخدم `sun6i_timer_init` كـ `.init_time` محتاج الـ DT node الصح للـ AHB1 reset، وإلا كل الـ peripherals على الـ bus دي هتفشل بشكل غامض.

---

### السيناريو 5: Automotive ECU بـ Allwinner A31 — HDMI مش بيشتغل بعد suspend/resume

#### العنوان
**HDMI display** بيختفي بعد system suspend على automotive ECU بـ A31

#### السياق
شركة automotive بتبني ECU display unit بـ Allwinner A31 لشاشة dashboard. الـ system بيعمل suspend/resume لتوفير الطاقة. بعد الـ resume، الـ HDMI بيرجع بس الشاشة سودا — الـ display controller مش شغال.

#### المشكلة
الـ HDMI display controller محتاج reset assertion/deassertion بعد الـ resume. لكن الـ reset controller المسجّل بـ `sun6i_reset_init()` هو early reset controller — الـ `reset_simple_ops` بتعمل:

- **assert**: بتـ clear الـ bit (لأن `active_low = true`)
- **deassert**: بتـ set الـ bit

بعد الـ resume، الـ HDMI driver بيطلب reset cycle بس في حالات معينة الـ spinlock في `reset_simple_data` ممكن يكون في state غريب لو الـ resume path مش handling الـ state صح.

```c
/* من reset-sunxi.c */
spin_lock_init(&data->lock);   /* بيتعمل مرة واحدة وقت الـ init */

data->active_low = true;   /* الـ HDMI reset bit اتعكس */
```

#### التحليل

```
suspend path:
  HDMI driver → reset_control_assert()
              → reset_simple_set_support() → spin_lock → clear bit → spin_unlock

resume path:
  HDMI driver → reset_control_deassert()
              → reset_simple_set_support() → spin_lock
              ↑ لو في race condition مع driver تاني بيعمل reset في نفس الوقت
              → ممكن يبقى الـ bit value غلط
```

الـ `nr_resets = size * 8` معناها كل bit في الـ register range هو reset line مستقلة. لو الـ size غلط في الـ DT:

```c
size = resource_size(&res);
data->rcdev.nr_resets = size * 8;   /* لو size = 4 → 32 reset lines */
```

HDMI reset bit ممكن يكون خارج الـ range المتوقع.

#### الحل

**1. تحقق من الـ reset bit number:**

```bash
# شوف الـ HDMI reset في DT
grep -r "hdmi" arch/arm/boot/dts/sun6i-a31*.dts* | grep reset
```

**2. تأكد من الـ reg size في DT:**

```dts
ahb1_rst: reset@1c202c0 {
    compatible = "allwinner,sun6i-a31-ahb1-reset";
    reg = <0x01c202c0 0xc>;   /* 0xc = 12 bytes = 96 reset lines */
    #reset-cells = <1>;
};
```

**3. اختبر الـ reset cycle يدوياً بعد الـ resume:**

```bash
# trigger manual reset test
echo 1 > /sys/kernel/debug/reset/reset_N   # N = HDMI bit
echo 0 > /sys/kernel/debug/reset/reset_N
# شوف لو الـ HDMI رجع
```

**4. لو المشكلة في الـ resume ordering، أضف في الـ HDMI driver:**

```c
static int hdmi_resume(struct device *dev)
{
    /* force reset cycle */
    reset_control_assert(hdmi->rst);
    udelay(10);
    reset_control_deassert(hdmi->rst);
    /* reinit display controller */
    return hdmi_hw_init(hdmi);
}
```

#### الدرس المستفاد
الـ early reset controller المسجّل بـ `sun6i_reset_init()` بيظل active طول حياة الـ system — هو مش بيتحرر أو بيتـ suspend. المشاكل الظاهرة بعد suspend/resume غالباً بترجع لـ ordering في الـ driver نفسه مش في الـ reset controller. الـ `active_low = true` على الـ Allwinner AHB1 reset معناه إن الـ bit الـ set = deasserted (شغال)، وده بيعكس الـ intuition العادي — لازم تراجع الـ datasheet قبل ما تـ debug.
## Phase 7: مصادر ومراجع

### الـ Source File

الـ header المدروس هو `include/linux/reset/sunxi.h` — بسيط جداً، بس بيمثل نقطة دخول مهمة:

```c
/* SPDX-License-Identifier: GPL-2.0 */
#ifndef __LINUX_RESET_SUNXI_H__
#define __LINUX_RESET_SUNXI_H__

void __init sun6i_reset_init(void);

#endif
```

الـ function دي بتتستخدم جوا machine initialization لـ Allwinner A31 (sun6i) — SoC بيحتاج reset controller خاص يتشغل بدري قبل ما drivers العادية تبدأ.

---

### مقالات LWN.net

| المقال | الصلة بالموضوع |
|--------|---------------|
| [reset: Add generic GPIO reset driver](https://lwn.net/Articles/585145/) | تقديم generic GPIO reset لـ reset controller framework — نفس الـ framework اللي بيستخدمه sunxi |
| [ARM: sunxi: Introduce Allwinner A23 (sun8i) support](https://lwn.net/Articles/602527/) | يشرح إزاي sun8i ورثت reset controller support من sun6i |
| [ARM: sunxi: Introduce Allwinner H3 support](https://lwn.net/Articles/644670/) | يضيف compatible جديد لـ H3 reset controller فوق نفس البنية |
| [Initial Allwinner H6 support](https://lwn.net/Articles/743378/) | تطور reset controller في جيل جديد من sunxi SoCs |
| [Introduction of STM32MP13 RCC (Reset Clock Controller)](https://lwn.net/Articles/888111/) | مثال مقارن: reset + clock controller مدمجين زي ما بيحصل في sunxi AHB1 |
| [Add Mobileye EyeQ system controller support (clk, reset, pinctrl)](https://lwn.net/Articles/979170/) | pattern مشابه: reset ضمن system controller شامل |
| [Allwinner sun6i message box support](https://lwn.net/Articles/809324/) | يستخدم `devm_reset_control_get_exclusive` كـ consumer لـ sunxi reset |

---

### توثيق الـ Kernel الرسمي

**الـ Reset Controller API:**
```
Documentation/driver-api/reset.rst
```
- المرجع الرئيسي لكل consumer و provider APIs
- موجود online: [Reset controller API — kernel.org](https://docs.kernel.org/driver-api/reset.html)

**الـ Device Tree Bindings:**
```
Documentation/devicetree/bindings/reset/allwinner,sun6i-a31-clock-reset.yaml
```
- بيعرّف الـ compatible string: `allwinner,sun6i-a31-ahb1-reset`
- مثال DTS:
  ```
  resets = <&ahb1_rst 21>;
  ```

**توثيق ARM Allwinner SoCs:**
```
Documentation/arm/sunxi.rst
```
- موجود online: [ARM Allwinner SoCs — kernel.org](https://www.kernel.org/doc/html/v5.6/arm/sunxi.html)

---

### Patchwork — أهم الـ Patches

| الـ Patch | الأهمية |
|-----------|---------|
| [reset: Add reset controller API — v4,2/8](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1361878774-6382-3-git-send-email-p.zabel@pengutronix.de/) | Philipp Zabel يُقدّم الـ framework الأصلي |
| [ARM: sunxi: allow building without reset controller](https://patchwork.kernel.org/project/linux-arm-kernel/patch/4307148.6KkdViVLZk@wuerfel/) | Arnd Bergmann يجعل `sun6i_reset_init()` optional بـ `IS_ENABLED(CONFIG_RESET_CONTROLLER)` |
| [reset: sunxi: Add Allwinner H3 bus resets — v4,4/6](https://patchwork.ozlabs.org/patch/536747/) | توسعة sunxi reset لجيل H3 |
| [power: reset: Add Allwinner A31 reset code — 2/5](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1398265476-29373-3-git-send-email-maxime.ripard@free-electrons.com/) | Maxime Ripard يُضيف الـ sun6i reboot driver |

---

### الـ Source Code المرجعي في الـ Kernel

```
drivers/reset/reset-sunxi.c        ← التنفيذ الفعلي للـ provider
include/linux/reset/sunxi.h        ← الـ header المدروس (sun6i_reset_init)
include/linux/reset.h              ← consumer API العام
drivers/reset/core.c               ← قلب الـ reset controller framework
```

**الـ GitHub (torvalds/linux):**
- [drivers/reset/reset-sunxi.c](https://github.com/torvalds/linux/blob/master/drivers/reset/reset-sunxi.c)

---

### نقاشات Mailing List

| المصدر | الموضوع |
|--------|---------|
| [linux-sunxi Google Groups](https://groups.google.com/g/linux-sunxi) | مجتمع Allwinner الرئيسي — نقاشات reset وكل حاجة sunxi |
| [linux-sunxi.org Mailing list](https://linux-sunxi.org/Mailing_list) | دليل القوائم الرسمية للـ community |
| [LKML: Arnd Bergmann — optional reset build](https://patchwork.kernel.org/project/linux-arm-kernel/patch/4307148.6KkdViVLZk@wuerfel/) | النقاش اللي أثبت ليه `sun6i_reset_init` موجودة في header منفصل |

---

### eLinux.org

| الصفحة | المحتوى |
|--------|---------|
| [Hack A10 devices — eLinux.org](https://elinux.org/Hack_A10_devices) | أدوات sunxi الأولى وتاريخ التطوير |
| [Device Tree Information — eLinux.org](https://www.elinux.org/Device_Tree_Information) | أساسيات DT اللي بيعتمد عليها reset controller |
| [Mainlining out-of-tree SoCs (PDF)](https://elinux.org/images/3/31/Ripard-mainlining-out-of-tree-socs.pdf) | Maxime Ripard بيشرح تجربة إدخال sunxi للـ mainline |

---

### كتب مقترحة

| الكتاب | الفصل / القسم المقترح |
|--------|-----------------------|
| **Linux Device Drivers, 3rd Ed. (LDD3)** — Corbet, Rubini, Kroah-Hartman | Chapter 14: The Linux Device Model — بيفهم hierarchy اللي reset controller بيشتغل فيها |
| **Linux Kernel Development** — Robert Love | Chapter 17: Devices and Modules — platform_driver و init sequences |
| **Embedded Linux Primer, 2nd Ed.** — Christopher Hallinan | Chapter 16: Kernel Initialization — يربط `__init` و machine setup بالـ reset init |
| **Professional Linux Kernel Architecture** — Wolfgang Mauerer | Chapter 6: Device Drivers — driver model كامل |

---

### مصطلحات البحث (Search Terms)

لو حابب تعمق أكتر، ابحث بـ:

```
linux reset controller framework tutorial
sunxi reset controller device tree
allwinner sun6i a31 reset ahb1
reset_controller_register linux kernel
devm_reset_control_get_exclusive example
__init linux kernel machine initialization
CONFIG_RESET_CONTROLLER kconfig
reset_control_assert deassert linux
philipp zabel reset controller maintainer
```

---

### مراجع سريعة — LKML و Lore

- [lore.kernel.org — reset controller patches](https://lore.kernel.org/linux-arm-kernel/?q=sunxi+reset+controller)
- [LKML: Philipp Zabel — Reset Controller Framework MAINTAINERS](https://lkml.org/lkml/2025/8/14/951)
- [Linux Kernel Driver DataBase — CONFIG_POWER_RESET_SUN6I](https://cateee.net/lkddb/web-lkddb/POWER_RESET_SUN6I.html)
- [allwinner,sun6i-a31-clock-reset.yaml binding](https://mjmwired.net/kernel/Documentation/devicetree/bindings/reset/allwinner,sun6i-a31-clock-reset.yaml)
## Phase 8: Writing simple module

### الفكرة

الملف `include/linux/reset/sunxi.h` بيعرّف function واحدة بس هي `sun6i_reset_init()` — دي early-init function بتتشغل قبل ما أي driver عادي يتسجّل، وبتمشي على device tree nodes وبتسجّل reset controllers لـ Allwinner sun6i SoCs.

بما إن الـ function دي `__init` (بتتنفذ مرة واحدة وقت boot)، مش مناسب نستخدمها مع kprobe بعد ما الـ kernel اتحمل. البديل الأذكى هو نعمل **kprobe على `reset_controller_register`** — دي الـ exported function اللي `sun6i_reset_init` بتستدعيها، وبتتسجّل كل مرة أي reset controller بيتضاف للنظام. كده نشوف في الـ log كل reset controller بيتسجّل مع معلومات مفيدة عنه.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on reset_controller_register()
 * Prints info about every reset controller registered in the system.
 * Useful for tracing sunxi (and any other) reset controller registration.
 */

#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/kprobes.h>
#include <linux/reset-controller.h>   /* struct reset_controller_dev */
#include <linux/of.h>                 /* struct device_node, of_node_full_name() */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("Trace reset_controller_register() calls via kprobe");

/* ---------- kprobe handler ---------- */

/*
 * pre_handler يتنادى مباشرةً قبل ما الـ CPU ينفّذ أول instruction
 * في reset_controller_register(). الـ argument الأول للـ function
 * موجود في regs->di على x86-64 (أو r0 على ARM).
 * kprobes بتجيب الـ args من الـ pt_regs اللي حفظها قبل الـ hook.
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * نجيب الـ argument الأول (rcdev) من الـ registers.
     * الـ macro دا portable — بيختار الـ register الصح حسب الـ arch.
     */
    struct reset_controller_dev *rcdev =
        (struct reset_controller_dev *)regs_get_kernel_argument(regs, 0);

    if (!rcdev)
        return 0;

    pr_info("reset_controller_register: nr_resets=%u, of_node=%s, ops=%ps\n",
            rcdev->nr_resets,
            rcdev->of_node ? of_node_full_name(rcdev->of_node) : "(none)",
            rcdev->ops);

    return 0; /* لازم يرجع 0 دايماً في pre_handler */
}

/* ---------- kprobe struct ---------- */

/*
 * الـ struct دا بيحدد:
 *   - symbol_name: اسم الـ function اللي هنعمل عليها probe
 *   - pre_handler: الـ callback اللي بيتنادى قبل الـ function
 * مش بنحتاج post_handler هنا لأن بياناتنا موجودة في الـ args.
 */
static struct kprobe kp = {
    .symbol_name = "reset_controller_register",
    .pre_handler = handler_pre,
};

/* ---------- init / exit ---------- */

static int __init sunxi_trace_init(void)
{
    int ret;

    /*
     * register_kprobe() بتحط software breakpoint على أول byte
     * من الـ function المحددة. لو الـ function مش موجودة أو
     * مش مسموح probe عليها، بترجع error.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("sunxi_trace: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("sunxi_trace: kprobe planted on reset_controller_register at %p\n",
            kp.addr);
    return 0;
}

static void __exit sunxi_trace_exit(void)
{
    /*
     * unregister_kprobe() ضروري في exit عشان تشيل الـ breakpoint
     * من الـ kernel text وترجع الـ original bytes.
     * لو منعملش unregister، الـ handler pointer بيبقى dangling
     * بعد ما الـ module يتـ unload وده بيودّي kernel panic.
     */
    unregister_kprobe(&kp);
    pr_info("sunxi_trace: kprobe removed\n");
}

module_init(sunxi_trace_init);
module_exit(sunxi_trace_exit);
```

---

### Makefile

```makefile
obj-m += sunxi_trace.o

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
| `linux/kprobes.h` | بيعرّف `struct kprobe` وكل الـ API الخاصة بيها |
| `linux/reset-controller.h` | بيعرّف `struct reset_controller_dev` اللي هنقرأ منها الـ data |
| `linux/of.h` | بيوفر `of_node_full_name()` عشان نطبع اسم الـ DT node بشكل مقروء |

#### `regs_get_kernel_argument(regs, 0)`

**الـ macro دا** portable abstraction فوق الـ calling convention — على x86-64 بيقرأ `rdi`، وعلى ARM64 بيقرأ `x0`، بدون ما نكتب كود arch-specific. ده بيجيب الـ argument الأول للـ function اللي عملنا عليها probe وهو `rcdev`.

#### `pr_info()` في الـ handler

بنطبع ثلاث معلومات مهمة:
- **`nr_resets`**: عدد الـ reset lines اللي الـ controller بيوفرها — بيخلينا نعرف حجم الـ controller.
- **`of_node`**: اسم الـ device tree node المرتبط بالـ controller — بيحدد الـ hardware المقصود.
- **`ops`**: عنوان الـ ops struct بيتطبع كاسم function (`%ps`) عشان نعرف إيه الـ driver اللي سجّل.

#### ليه `return 0` في `pre_handler`؟

الـ kprobes API بتتوقع `pre_handler` يرجع 0 دايماً. أي قيمة تانية ممكن تعمل undefined behavior في بعض الـ kernel versions.

#### ليه `unregister_kprobe` في `module_exit`؟

لو ما أزلناش الـ probe، الـ kernel هيفضل يستدعي `handler_pre` اللي عنوانه اتحذف مع الـ module — ده بيعمل **use-after-free** وكراش مضمون. الـ unregister بيرجع أصل الـ instruction الأصلية ويزيل الـ hook بشكل آمن.

---

### تشغيل وتجربة

```bash
# Build
make

# Load
sudo insmod sunxi_trace.ko

# شوف أي reset controller بيتسجّل (مثلاً عند تحميل driver جديد)
sudo dmesg | grep sunxi_trace

# Unload
sudo rmmod sunxi_trace

# تأكيد إن الـ probe اتشال
sudo dmesg | tail -5
```

### مثال على الـ output المتوقع

```
[  142.331204] sunxi_trace: kprobe planted on reset_controller_register at ffffffffc0a12340
[  142.889017] reset_controller_register: nr_resets=128, of_node=/soc/reset-controller@1c202c0, ops=reset_simple_ops
```

الـ line دي بتأكد إن الـ sun6i reset controller اتسجّل بـ 128 reset line (128 = 16 bytes × 8 bits) من الـ DT node الصح، وإن الـ ops المستخدمة هي `reset_simple_ops`.
