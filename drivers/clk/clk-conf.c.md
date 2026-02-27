## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الملف `clk-conf.c` جزء من **Common Clock Framework (CCF)** — الـ subsystem المسؤول عن إدارة كل الـ clocks في الـ Linux kernel. المشرفون عليه هم Michael Turquette و Stephen Boyd، والـ mailing list هو `linux-clk@vger.kernel.org`.

---

### القصة: ليه محتاج "إعداد افتراضي" للـ clocks؟

تخيل إنك بتشغّل جهاز embedded — مثلاً Samsung Exynos أو Raspberry Pi. الـ hardware فيه عشرات الـ clocks: clock لوحدة الـ CPU، clock للـ USB، clock للـ camera، إلخ. كل clock ممكن يتغذّى من مصادر مختلفة (**parents**)، وليه سرعة تشغيل (**rate**) محددة.

المشكلة: **مين اللي يحدد إعدادات الـ clocks دي قبل ما أي driver يبدأ شغله؟**

الحل الكلاسيكي كان: كل driver يضبط الـ clock بتاعه يدوياً في الكود — وده يعني إن الـ board configuration مدفون جوه الـ C code، وأي تغيير في الـ hardware يحتاج تعديل في الكود.

الحل الحديث: تكتب الإعدادات في الـ **Device Tree (DT)** — ملف وصفي للـ hardware — وتخلي الـ kernel يقرأها ويطبّقها أوتوماتيك. وده بالظبط اللي بيعمله `clk-conf.c`.

---

### الهدف من الملف

`clk-conf.c` مسؤول عن **تطبيق إعدادات الـ clocks المكتوبة في الـ Device Tree** على الـ hardware الفعلي وقت الـ boot أو وقت تسجيل الـ clock provider.

الإعدادات دي بتيجي من ثلاث properties في الـ DT:

| DT Property | المعنى |
|---|---|
| `assigned-clocks` | قائمة الـ clocks اللي هتتأثر بالإعدادات |
| `assigned-clock-parents` | مصدر كل clock منهم (الـ parent) |
| `assigned-clock-rates` | السرعة المطلوبة لكل clock (بالـ Hz، 32-bit) |
| `assigned-clock-rates-u64` | نفس الفكرة لكن 64-bit للـ clocks العالية السرعة |

---

### مثال حقيقي من الـ Device Tree

```dts
/* مثال: ضبط clock الـ UART على مصدر معين وسرعة 48 MHz */
uart0: serial@12000000 {
    assigned-clocks = <&cmu CLK_UART0>;
    assigned-clock-parents = <&cmu CLK_MOUT_UART0>;
    assigned-clock-rates = <48000000>;
};
```

لما الـ kernel يشوف الـ node ده، بيستدعي `of_clk_set_defaults()` اللي بتقرأ الـ properties دي وتطبّقها.

---

### الـ API الوحيد اللي بيعمله الملف

```c
int of_clk_set_defaults(struct device_node *node, bool clk_supplier);
```

- **`node`**: الـ device node في الـ DT اللي عايزين نطبّق إعداداته.
- **`clk_supplier`**: هل الـ node ده نفسه بيوفّر clocks لغيره؟ (مهم لتجنب الـ circular dependency).

الدالة دي بتنادي من داخلها على دالتين private:
1. `__set_clk_parents()` — تضبط الـ parent لكل clock.
2. `__set_clk_rates()` — تضبط الـ rate لكل clock.

---

### الـ `clk_supplier` flag — ليه مهم؟

لو عندي chip بيوفّر clocks لغيره (مثلاً PLL controller) وفي نفس الوقت بيحتاج يضبط clock بتاعه من نفسه — لو مفيش حماية، الكود هيحاول يجيب clock من provider مش اتسجّل لسه، وهيحصل deadlock أو error.

الحل: لو `clk_supplier = false` والـ node المطلوب هو نفس الـ node الحالي، الدالة بترجع فوراً بـ `0` من غير ما تعمل حاجة — علشان الـ clock provider نفسه لسه مش جاهز.

---

### مين بيستدعي `of_clk_set_defaults`؟

الدالة دي بتتنادى من أماكن كتير جداً في الـ kernel، لأن **أي device ممكن يعرّف إعدادات clock في الـ DT**:

| الملف | السياق |
|---|---|
| `drivers/clk/clk.c` | وقت تسجيل الـ clock provider |
| `drivers/base/platform.c` | وقت probe أي platform device |
| `drivers/i2c/i2c-core-base.c` | وقت probe أي I2C device |
| `drivers/spi/spi.c` | وقت probe أي SPI device |
| `drivers/amba/bus.c` | وقت probe أي AMBA device |
| `drivers/usb/common/ulpi.c` | وقت probe أي ULPI USB device |

يعني فعلياً: **أي device بيتعمله probe في الـ Linux، الـ kernel بيحاول يطبّق إعدادات الـ clocks بتاعته من الـ DT أوتوماتيك.**

---

### الصورة الكاملة: مكانه في الـ CCF

```
Device Tree (*.dts)
      │
      │  assigned-clocks / assigned-clock-parents / assigned-clock-rates
      │
      ▼
clk-conf.c  ←─── of_clk_set_defaults()
      │
      ├──► __set_clk_parents()  →  clk_set_parent()
      │
      └──► __set_clk_rates()   →  clk_set_rate()
                                        │
                                        ▼
                              clk.c  (Common Clock Framework core)
                                        │
                                        ▼
                              Hardware Clock Driver
                              (e.g. samsung/clk-exynos.c)
                                        │
                                        ▼
                              Physical PLL / MUX / Divider on Chip
```

---

### الملفات المرتبطة

**Core files للـ subsystem:**

| الملف | الدور |
|---|---|
| `drivers/clk/clk.c` | الـ CCF core — كل منطق الـ clocks |
| `drivers/clk/clk-conf.c` | تطبيق إعدادات الـ DT (هذا الملف) |
| `drivers/clk/clkdev.c` | ربط الـ clocks بالـ devices بالاسم |
| `drivers/clk/clk-devres.c` | managed (devm_) versions للـ clock APIs |

**Headers:**

| الملف | الدور |
|---|---|
| `include/linux/clk/clk-conf.h` | تعريف `of_clk_set_defaults()` |
| `include/linux/clk-provider.h` | APIs لصانعي الـ clock drivers |
| `include/linux/clk.h` | APIs لمستخدمي الـ clocks (device drivers) |
| `include/linux/of_clk.h` | APIs لربط الـ CCF مع الـ Device Tree |

**Hardware drivers (أمثلة):**

| الملف | الدور |
|---|---|
| `drivers/clk/samsung/` | clocks لشرائح Samsung Exynos |
| `drivers/clk/bcm/` | clocks لشرائح Broadcom |
| `drivers/clk/rockchip/` | clocks لشرائح Rockchip |
| `drivers/clk/meson/` | clocks لشرائح Amlogic Meson |

**Documentation:**

| الملف | الدور |
|---|---|
| `Documentation/devicetree/bindings/clock/` | DT bindings لكل clock controllers |
## Phase 2: شرح الـ Clock Configuration (clk-conf) Framework

---

### المشكلة — ليه الـ subsystem ده موجود؟

في أي SoC حديث (Qualcomm، Samsung Exynos، NXP i.MX، إلخ) فيه عشرات الـ clocks — كل واحد ممكن يتغذى من أكتر من source، وممكن يشتغل بأكتر من rate. لما بتيجي تشغّل kernel على بورد معينة، لازم كل clock يتسّط على:

1. **الـ parent** المناسب (يتغذى منين؟)
2. **الـ rate** المطلوبة (يشتغل بكام Hz؟)

**المشكلة الأصلية:** مين المسؤول إنه يعمل ده؟

- لو سبته للـ driver نفسه → كل driver لازم يعرف topology الـ clock tree كاملة ويعمل hardcode للقيم — ده كارثة لأن نفس الـ driver بيتشغل على boards مختلفة بـ clock configurations مختلفة.
- لو سبته للـ bootloader فقط → ممكن يكون الـ bootloader مش بيشيل كل الـ clocks.

**الحل الصح:** الـ configuration تيجي من الـ **Device Tree (DT)**، والـ kernel يقراها وينفذها أوتوماتيك وقت الـ probe.

---

### الحل — الـ kernel اتعمل إيه؟

الـ kernel عرّف ثلاث DT properties معيارية:

| Property | الغرض |
|---|---|
| `assigned-clocks` | قايمة الـ clocks اللي هتتغير |
| `assigned-clock-parents` | الـ parent لكل clock فيهم |
| `assigned-clock-rates` | الـ rate المطلوبة لكل clock (u32، بالـ Hz) |
| `assigned-clock-rates-u64` | نفس الفكرة بس u64 للـ rates العالية جداً |

الـ function `of_clk_set_defaults()` في `clk-conf.c` هي اللي بتقرا الـ properties دي من الـ DT node وبتطبقها على الـ clock tree فعلياً.

---

### مثال حقيقي

في Exynos5 SoC، الـ MMC controller محتاج clock بـ 200 MHz من PLL_DPLL. في الـ DT:

```dts
mmc0: mmc@12200000 {
    assigned-clocks = <&clock CLK_SCLK_MMC0>;
    assigned-clock-parents = <&clock CLK_MOUT_DPLL>;
    assigned-clock-rates = <200000000>;
};
```

لما kernel يعمل probe للـ `mmc0` node، بيستدعي `of_clk_set_defaults()` وده بيعمل:
1. يربط `CLK_SCLK_MMC0` بالـ parent `CLK_MOUT_DPLL`
2. يضبط rate الـ `CLK_SCLK_MMC0` على 200 MHz

الـ driver نفسه مش محتاج يعرف أي حاجة عن الـ clock tree.

---

### الـ Big Picture Architecture

```
Device Tree (.dts / .dtsi)
─────────────────────────────────────────────────────────────
 mmc0 node:
   assigned-clocks        = <&clock CLK_SCLK_MMC0>
   assigned-clock-parents = <&clock CLK_MOUT_DPLL>
   assigned-clock-rates   = <200000000>
         │
         │  parsed by OF subsystem
         ▼
┌─────────────────────────────────┐
│       of_clk_set_defaults()     │  <── clk-conf.c
│                                 │
│  __set_clk_parents()            │
│    of_parse_phandle_with_args() │──┐
│    of_clk_get_from_provider()   │  │  resolves phandle
│    clk_set_parent()             │  │  → struct clk *
│                                 │  │
│  __set_clk_rates()              │  │
│    of_property_read_u32_array() │  │
│    of_clk_get_from_provider()   │──┘
│    clk_set_rate()               │
└──────────────┬──────────────────┘
               │ calls into
               ▼
┌─────────────────────────────────┐
│     Common Clock Framework      │  <── drivers/clk/clk.c
│  (CCF — struct clk_core tree)   │
│                                 │
│  clk_set_parent() → ops->set_parent() ──► HW driver
│  clk_set_rate()   → ops->set_rate()   ──► HW driver
└─────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│   Platform Clock Provider       │  مثلاً samsung/clk-exynos5420.c
│   (struct clk_hw implementations│
│    for each physical clock)     │
└─────────────────────────────────┘
```

**الـ consumers** (اللي بتستخدم `of_clk_set_defaults`):
- الـ device driver core نفسه بيستدعيها أوتوماتيك في `of_clk_set_defaults()` من `drivers/base/platform.c` أثناء الـ probe.
- الـ clock provider نفسه ممكن يستدعيها لو كان `clk_supplier = true`.

**الـ providers** (اللي بتنفذ الـ clock hardware):
- Samsung clock drivers
- Qualcomm clock drivers
- NXP i.MX clock drivers
- إلخ — كلهم بيسجلوا نفسهم في الـ CCF provider list.

---

### الـ Analogy الحقيقي — توصيل كهرباء في مصنع

تخيل مصنع فيه ماكينات (devices) وشبكة كهرباء معقدة (clock tree).

| العالم الحقيقي | الـ Kernel |
|---|---|
| الـ **مخطط الكهربي** للمصنع | الـ **Device Tree** |
| كل **ماكينة** | كل **device node** في الـ DT |
| **مصادر الكهرباء** (generators) | الـ **PLLs / Oscillators** |
| **موزعات الكهرباء** (switchboards) | الـ **clock muxes** |
| **محولات الجهد** (transformers) | الـ **clock dividers/multipliers** |
| **السلك اللي بيوصل الماكينة بالموزع** | الـ `assigned-clock-parents` |
| **الجهد المطلوب على الماكينة** | الـ `assigned-clock-rates` |
| **مهندس الكهرباء** اللي بيقرا المخطط وينفذه | `of_clk_set_defaults()` |
| **شركة توزيع الكهرباء** (القواعد والـ API) | الـ **Common Clock Framework (CCF)** |
| **الكهربائي** اللي بيعمل الوصلات الفعلية | الـ **platform clock driver** |

المهندس (clk-conf) بيقرا المخطط (DT) ويقول لشركة التوزيع (CCF): "الماكينة دي محتاجة تتوصل بالموزع ده وبجهد كذا". شركة التوزيع بتفهم القواعد وبتبعث الكهربائي (platform driver) يعمل الوصلة الفعلية في الـ hardware.

**المفاهيم الدقيقة في الـ analogy:**

- **`clk_supplier = true`**: الماكينة نفسها أحياناً بتكون موزع كهرباء لحاجات تانية — زي كارت network card ممكن يوفر clock لبقية الـ bus. في الحالة دي، المهندس بيكمل يشتغل حتى لو الماكينة هي نفسها المصدر.
- **`-EPROBE_DEFER`**: الماكينة التانية (الـ parent clock) لسه ما اتركبتش في المصنع — المهندس بيقول "أجي تاني لما تكون جاهزة".
- **`of_node_put()`**: بعد ما المهندس يقرا الرسمة، بيرجع الورقة لمكانها (reference counting للـ DT nodes).

---

### الـ Core Abstraction — الفكرة المحورية

**الـ clk-conf module هو translator بين الـ static DT configuration والـ dynamic clock tree.**

الفكرة المحورية هي: **الـ clock configuration ملكية الـ board/platform (DT), مش الـ driver**.

بدل ما كل driver يعمل:
```c
/* Bad old way — driver knows too much */
clk = clk_get(dev, "mmc_clk");
clk_set_parent(clk, pll_dpll_clk);  /* hardcoded! */
clk_set_rate(clk, 200000000);        /* hardcoded! */
```

الطريقة الجديدة:
```c
/* Driver does nothing — DT handles it */
/* of_clk_set_defaults() already ran before probe */
clk = devm_clk_get(dev, "mmc_clk");  /* just get it */
clk_prepare_enable(clk);              /* already configured */
```

الـ separation of concerns هنا حاد ونظيف:
- **الـ DT**: "أنا الـ board، ده الـ wiring"
- **clk-conf**: "أنا بقرا الـ wiring وأنفذه"
- **الـ driver**: "أنا مش عارف ومش مهتم بالـ wiring"

---

### الـ struct المحورية وعلاقتها ببعض

```
struct device_node (node في الـ DT)
│
│  "assigned-clocks" property
│  "assigned-clock-parents" property
│  "assigned-clock-rates" property
│
├── of_parse_phandle_with_args()
│       │
│       └──► struct of_phandle_args
│               │
│               ├── .np  (pointer لـ device_node التاني)
│               └── .args[] (الـ specifier — زي رقم الـ clock)
│
└── of_clk_get_from_provider(&clkspec)
        │
        └──► struct clk *  (consumer handle)
                │
                ├── clk_set_parent(clk, pclk)
                │       │
                │       └──► CCF ──► struct clk_core
                │                        └──► clk_ops->set_parent()
                │                                └──► HW register write
                │
                └── clk_set_rate(clk, rate)
                        │
                        └──► CCF ──► struct clk_core
                                         └──► clk_ops->set_rate()
                                                 └──► HW register write
```

**ملاحظة مهمة:** الـ `struct clk *` اللي بيرجعها `of_clk_get_from_provider()` هي consumer handle (per-user reference)، مش الـ `struct clk_core` الفعلي. الـ CCF بيحافظ على `clk_core` مركزياً وبيدي كل caller handle منفصل — زي الـ file descriptor pattern.

---

### الـ clk-conf يملك إيه مقابل إيه بيفوضه؟

| الـ clk-conf يملكه | بيفوضه لـ |
|---|---|
| قراية وتفسير الـ DT properties | **OF subsystem** — `of_parse_phandle_with_args()` |
| resolve الـ phandle لـ `struct clk *` | **CCF provider layer** — `of_clk_get_from_provider()` |
| منطق الـ ordering وإدارة الـ errors | نفسه |
| التعامل مع الـ `clk_supplier` edge case | نفسه |
| **الـ parent switch الفعلي** | **CCF core** — `clk_set_parent()` → `clk_ops->set_parent()` |
| **الـ rate change الفعلي** | **CCF core** — `clk_set_rate()` → `clk_ops->set_rate()` |
| كتابة الـ hardware registers | **Platform clock driver** |

---

### الـ clk_supplier Flag — التفصيلة الدقيقة

```c
if (clkspec.np == node && !clk_supplier) {
    of_node_put(clkspec.np);
    return 0;  /* bail out early */
}
```

لو الـ `assigned-clocks` أو `assigned-clock-parents` بيشير للـ node نفسها (الـ device هي نفسها clock provider)، وفي نفس الوقت `clk_supplier = false`، الـ function بتخرج فوراً.

**ليه؟** لأن لو الـ device نفسها هي اللي بتوفر الـ clock (supplier)، الـ clocks دي ممكن تكون لسه ما اتسجلتش في الـ CCF وقت الـ `of_clk_set_defaults()` بتتستدعى — فبدل ما نعمل race condition، بنتجاهل.

لما بتستدعي من الـ clock provider driver نفسه (بعد ما يسجّل نفسه)، بتبعت `clk_supplier = true` وبيكمل يشتغل عادي.

---

### الـ u64 rates — ليه؟

```c
count_64 = of_property_count_u64_elems(node, "assigned-clock-rates-u64");
if (count_64 > 0) {
    /* use 64-bit rates */
} else {
    /* fallback to 32-bit */
}
```

الـ `u32` بتعدي على 4 GHz كحد أقصى. في بعض الـ modern SoCs (مثلاً بعض الـ high-speed SerDes clocks)، الـ rates ممكن تتخطى ده. الـ `assigned-clock-rates-u64` property بتحل المشكلة دي مع backward compatibility كاملة.

---

### الـ Subsystems المرتبطة

1. **Common Clock Framework (CCF)** — `drivers/clk/clk.c`: الـ framework الرئيسي اللي بيدير الـ clock tree. الـ clk-conf بيتكلم معاه عن طريق `clk_set_parent()` و`clk_set_rate()`.

2. **OF (Open Firmware / Device Tree) subsystem** — `drivers/of/`: بيوفر كل الـ DT parsing APIs زي `of_parse_phandle_with_args()` و`of_property_read_u32_array()`.

3. **Device Core** — `drivers/base/`: بيستدعي `of_clk_set_defaults()` أوتوماتيك قبل الـ probe لكل device — الـ driver ما بيشوفهاش.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Config Options والـ Flags — Cheatsheet

#### Config Options اللي بتتحكم في الكود

| Config Option | التأثير |
|---|---|
| `CONFIG_OF` | لو مش موجود → `of_clk_set_defaults` بتبقى stub ترجع 0 دايمًا |
| `CONFIG_COMMON_CLK` | لو مش موجود → نفس الحكاية، الكود الحقيقي مش متضمّن |
| `CONFIG_OF_DYNAMIC` | لو موجود → `of_node_put` بتعمل reference counting حقيقي |
| `CONFIG_OF_KOBJ` | بيضيف `kobject` جوا `device_node` للـ sysfs integration |

الملف ده **مش بيتبنى خالص** غير لو الاتنين `CONFIG_OF` و `CONFIG_COMMON_CLK` موجودين مع بعض.

#### الـ Device Node Flags (`_flags` field)

| Flag | القيمة | المعنى |
|---|---|---|
| `OF_DYNAMIC` | 1 | الـ node اتعمل بـ `kmalloc`، ممكن يتحرر |
| `OF_DETACHED` | 2 | الـ node انفصل عن شجرة الـ DT |
| `OF_POPULATED` | 3 | device اتعمل للـ node ده |
| `OF_POPULATED_BUS` | 4 | platform bus اتعمل للـ children |
| `OF_OVERLAY` | 5 | الـ node اتعمل من overlay |
| `OF_OVERLAY_FREE_CSET` | 6 | في overlay cset جاري تحريره |

#### الـ CLK Notifier Event Types

| Macro | Bit | الوقت اللي بيتبعت فيه |
|---|---|---|
| `PRE_RATE_CHANGE` | BIT(0) | قبل ما الـ rate يتغير |
| `POST_RATE_CHANGE` | BIT(1) | بعد ما الـ rate يتغير بنجاح |
| `ABORT_RATE_CHANGE` | BIT(2) | لو التغيير فشل بعد الـ PRE |

---

### الـ Structs المهمة

#### 1. `struct device_node`

**الغرض:** بتمثل node واحدة في الـ Device Tree — يعني عقدة من شجرة الـ hardware description اللي بتتقرأ من الـ DTB (Device Tree Blob).

| Field | النوع | الشرح |
|---|---|---|
| `name` | `const char *` | اسم الـ node (مثلاً `"clock-controller"`) |
| `phandle` | `phandle` (u32) | معرّف فريد بيخلي nodes تانية تشاور عليه |
| `full_name` | `const char *` | المسار الكامل في الشجرة مثلاً `/soc/clk@12000000` |
| `fwnode` | `struct fwnode_handle` | الـ abstraction layer للـ firmware nodes |
| `properties` | `struct property *` | linked list بكل الـ properties |
| `deadprops` | `struct property *` | properties اتشالت (للـ overlay support) |
| `parent` | `struct device_node *` | الـ node الأب في الشجرة |
| `child` | `struct device_node *` | أول ابن |
| `sibling` | `struct device_node *` | الـ node الجنبها في نفس الـ level |
| `_flags` | `unsigned long` | الـ OF_* flags اللي فوق |
| `data` | `void *` | بيانات خاصة بالـ architecture |

**في clk-conf.c:** الـ `node` parameter في كل function هو pointer لـ `device_node` — ده اللي بتتقرأ منه `assigned-clocks` وغيرها.

---

#### 2. `struct of_phandle_args`

**الغرض:** بتحمل نتيجة parsing لـ phandle واحد من property زي `assigned-clocks` — يعني "انا عايز الـ clock رقم X من الـ provider Y".

| Field | النوع | الشرح |
|---|---|---|
| `np` | `struct device_node *` | الـ node اللي بتوفر الـ clock (الـ provider) |
| `args_count` | `int` | عدد الـ arguments اللي جت مع الـ phandle |
| `args[]` | `uint32_t[MAX_PHANDLE_ARGS]` | الـ arguments نفسها (زي clock index داخل الـ provider) |

**في clk-conf.c:** `of_parse_phandle_with_args` بتملي هذا الـ struct، وبعدين `of_clk_get_from_provider` بتاخده وترجع `struct clk *`.

**مهم جداً:** بعد ما تخلص من الـ `clkspec`، لازم تعمل `of_node_put(clkspec.np)` عشان تحرر الـ reference. الكود ده بيعمل ده صح في كل مكان.

---

#### 3. `struct clk` (opaque)

**الغرض:** الـ handle اللي الـ consumer بيشيل به reference على clock معينة. الـ struct نفسها internal في الـ CCF (Common Clock Framework) — الملف ده بيشوفها كـ opaque pointer بس.

الـ API المستخدم عليها في الملف ده:

| Function | العملية |
|---|---|
| `of_clk_get_from_provider(&clkspec)` | جيب الـ clock من الـ DT provider |
| `clk_set_parent(clk, pclk)` | غيّر الـ parent |
| `clk_set_rate(clk, rate)` | اضبط الـ frequency |
| `clk_get_rate(clk)` | اقرأ الـ frequency الحالية (للـ error logging) |
| `__clk_get_name(clk)` | اجيب اسم الـ clock (للـ logging) |
| `clk_put(clk)` | حرر الـ reference |

---

#### 4. `struct property`

**الغرض:** تمثل property واحدة داخل الـ `device_node` — زي `assigned-clocks = <&clk 0>`.

| Field | النوع | الشرح |
|---|---|---|
| `name` | `char *` | اسم الـ property |
| `length` | `int` | حجم الـ value بالـ bytes |
| `value` | `void *` | الـ raw data بالـ big-endian |
| `next` | `struct property *` | الـ property الجاية في الـ linked list |

**الملف ده** مش بيتعامل مع `struct property` مباشرة — بيستخدم الـ OF helper functions اللي بتعمل الـ parsing تلقائي.

---

### Struct Relationship Diagram

```
Device Tree (DTB in memory)
         │
         ▼
┌─────────────────────────────┐
│      device_node            │  ← الـ node اللي فيها:
│  ┌──────────────────────┐   │    assigned-clocks = <&clk1 0>, <&clk2 1>
│  │ properties (linked)  │   │    assigned-clock-parents = <&pclk 0>, <&pclk 1>
│  │  ┌────────────────┐  │   │    assigned-clock-rates = <24000000>
│  │  │ property:      │  │   │    assigned-clock-rates-u64 = <0 500000000>
│  │  │ "assigned-clks"│  │   │
│  │  └────────────────┘  │   │
│  │  ┌────────────────┐  │   │
│  │  │ property:      │  │   │
│  │  │ "assigned-rates│  │   │
│  │  └────────────────┘  │   │
│  └──────────────────────┘   │
│  parent ──► device_node     │
│  child  ──► device_node     │
│  sibling──► device_node     │
└─────────────────────────────┘
         │
         │  of_parse_phandle_with_args()
         ▼
┌─────────────────────────────┐
│     of_phandle_args         │
│  np ──────────────────────► │──► provider device_node
│  args_count = 1             │    (الـ clock controller)
│  args[0] = clock_index      │
└─────────────────────────────┘
         │
         │  of_clk_get_from_provider()
         ▼
┌─────────────────────────────┐
│       struct clk *          │  ← opaque handle
│  (managed by CCF)           │
│                             │
│  clk_set_parent(clk, pclk) │
│  clk_set_rate(clk, rate)   │
│  clk_put(clk)               │
└─────────────────────────────┘
```

---

### Lifecycle Diagram

#### دورة حياة إعداد الـ Clock من الـ DT

```
Boot / Driver Probe
        │
        ▼
of_clk_set_defaults(node, clk_supplier)
        │
        ├──► node == NULL ? → return 0 (no-op)
        │
        ▼
__set_clk_parents(node, clk_supplier)
        │
        ├── of_count_phandle_with_args("assigned-clock-parents")
        │       └── num_parents = N
        │
        ├── for index = 0..N-1:
        │       │
        │       ├── of_parse_phandle_with_args("assigned-clock-parents", index)
        │       │       └── fills: clkspec_parent
        │       │
        │       ├── [self-reference check] clkspec.np == node && !clk_supplier → return 0
        │       │
        │       ├── of_clk_get_from_provider(&clkspec_parent) → pclk
        │       │       └── IS_ERR? → return error (maybe -EPROBE_DEFER)
        │       │
        │       ├── of_parse_phandle_with_args("assigned-clocks", index)
        │       │       └── fills: clkspec_child
        │       │
        │       ├── of_clk_get_from_provider(&clkspec_child) → clk
        │       │       └── IS_ERR? → clk_put(pclk), return error
        │       │
        │       ├── clk_set_parent(clk, pclk)
        │       │       └── fail? → pr_err (but continue)
        │       │
        │       ├── clk_put(clk)
        │       └── clk_put(pclk)
        │
        ▼ (returns 0 or error)
__set_clk_rates(node, clk_supplier)
        │
        ├── of_property_count_u64_elems("assigned-clock-rates-u64") → count_64
        ├── of_property_count_u32_elems("assigned-clock-rates")     → count
        │
        ├── count_64 > 0 ?
        │       ├── YES: kcalloc u64 array → rates_64[]
        │       │         of_property_read_u64_array(...)
        │       └── NO:  kcalloc u32 array → rates[]
        │                 of_property_read_u32_array(...)
        │
        ├── [auto-freed via __free(kfree) on function exit]
        │
        ├── for index = 0..count-1:
        │       │
        │       ├── rate = rates_64[index] or rates[index]
        │       ├── rate == 0 ? → skip (zero means "don't change")
        │       │
        │       ├── of_parse_phandle_with_args("assigned-clocks", index)
        │       ├── [self-reference check]
        │       ├── of_clk_get_from_provider(&clkspec) → clk
        │       ├── clk_set_rate(clk, rate)
        │       │       └── fail? → pr_err + log current rate
        │       └── clk_put(clk)
        │
        └── return 0
```

---

### Call Flow Diagrams

#### Flow 1: تغيير الـ Parent

```
driver/platform_bus probe
  └─► of_clk_set_defaults(node, false)
        └─► __set_clk_parents(node, false)
              ├─► of_count_phandle_with_args(node, "assigned-clock-parents", "#clock-cells")
              │     └─► [OF core] reads property raw bytes, counts phandles
              │
              ├─► of_parse_phandle_with_args(node, "assigned-clock-parents", ..., index, &clkspec)
              │     └─► [OF core] resolves phandle → clkspec.np = provider_node
              │
              ├─► of_clk_get_from_provider(&clkspec)
              │     └─► [CCF] looks up clock in registered providers table
              │           └─► returns struct clk * (pclk)
              │
              ├─► of_parse_phandle_with_args(node, "assigned-clocks", ..., index, &clkspec)
              │     └─► [OF core] resolves the clock to be reparented
              │
              ├─► of_clk_get_from_provider(&clkspec)
              │     └─► [CCF] returns struct clk * (clk)
              │
              └─► clk_set_parent(clk, pclk)
                    └─► [CCF] clk_core_set_parent_nolock()
                          ├─► validates no circular dependency
                          ├─► calls clk_core->ops->set_parent()  ← hardware register write
                          └─► propagates rate recalculation up the tree
```

#### Flow 2: تغيير الـ Rate

```
of_clk_set_defaults(node, true)
  └─► __set_clk_rates(node, true)
        ├─► reads "assigned-clock-rates-u64" or "assigned-clock-rates"
        │     [u64 takes priority if exists]
        │
        ├─► for each (clock, rate) pair:
        │     ├─► of_parse_phandle_with_args → clkspec
        │     ├─► of_clk_get_from_provider → clk
        │     └─► clk_set_rate(clk, rate)
        │           └─► [CCF] clk_core_set_rate_nolock()
        │                 ├─► clk_calc_new_rates()  ← find best divider
        │                 ├─► clk_notify(PRE_RATE_CHANGE)
        │                 ├─► ops->set_rate()       ← hardware register write
        │                 └─► clk_notify(POST_RATE_CHANGE)
        │
        └─► [rates array auto-freed by __free(kfree) cleanup attribute]
```

#### Flow 3: الـ Self-Reference Guard

```
scenario: node X هو نفسه clock provider، وبيحاول يعمل reparent لنفسه

__set_clk_parents(node=X, clk_supplier=false)
  │
  ├─► parse "assigned-clock-parents"[0] → clkspec.np = X  (self reference!)
  │
  ├─► clkspec.np == node (X == X) && !clk_supplier (false)
  │         ↓
  │      TRUE → of_node_put(clkspec.np) → return 0
  │
  └─► [exits early, no parent change applied]
      [prevents deadlock: driver can't supply its own clock during probe]

─────────────────────────────────────────
scenario: clk_supplier=true (عارفين إن الـ node هو supplier)

__set_clk_parents(node=X, clk_supplier=true)
  │
  ├─► clkspec.np == node && !clk_supplier
  │         ↓
  │      FALSE (clk_supplier=true) → continues normally
  │
  └─► [sets parent as requested]
```

---

### الـ Locking Strategy

الملف `clk-conf.c` **نفسه مش بيمسك أي lock مباشرة**. الـ locking كله محصور في الطبقات اللي تحته:

#### طبقة الـ OF (Device Tree)

| اللي بيحصل | الـ Lock |
|---|---|
| `of_parse_phandle_with_args()` | بيقرأ الـ DT properties — protected بالـ `of_mutex` أو RCU حسب الـ config |
| `of_node_put()` / `of_node_get()` | reference counting بـ `atomic_t` — lock-free |

#### طبقة الـ CCF (Common Clock Framework)

| Operation | الـ Lock |
|---|---|
| `of_clk_get_from_provider()` | بيمسك `clk_provider_lock` (mutex) عشان يدور في قائمة الـ providers |
| `clk_set_parent()` | بيمسك `prepare_lock` (mutex) — serializes كل عمليات الـ clock tree |
| `clk_set_rate()` | بيمسك `prepare_lock` (mutex) — نفس الـ lock |
| `clk_put()` | بيمسك `prepare_lock` عشان يحرر الـ reference |

#### الـ Lock Ordering (من الخارج للداخل)

```
caller context (driver probe / platform init)
        │
        ▼
   [no lock held by clk-conf.c]
        │
        ▼
   of_clk_get_from_provider()
        └─► acquires: clk_provider_lock (mutex)
                └─► releases before returning struct clk *
        │
        ▼
   clk_set_parent() / clk_set_rate()
        └─► acquires: prepare_lock (mutex)
                └─► may acquire: enable_lock (spinlock) for hardware state
                        └─► calls ops->set_parent() / ops->set_rate()
                                └─► hardware register write
                └─► releases: enable_lock
                └─► releases: prepare_lock

Rule: prepare_lock > enable_lock (دايمًا prepare_lock الأول)
Rule: clk_provider_lock لازم يتحرر قبل ما تمسك prepare_lock
```

#### نقطة مهمة: الـ `-EPROBE_DEFER`

لو `of_clk_get_from_provider` رجعت `-EPROBE_DEFER`، ده معناه إن الـ clock provider لسه معملوش probe. الـ driver framework هيعيد probe الـ driver لاحقًا أوتوماتيك. الكود ده بيتعامل مع ده صح:

```c
if (IS_ERR(pclk)) {
    if (PTR_ERR(pclk) != -EPROBE_DEFER)
        pr_warn("...");   /* log only if NOT a defer */
    return PTR_ERR(pclk); /* propagate -EPROBE_DEFER silently */
}
```

ده pattern أساسي في الـ kernel — الـ `-EPROBE_DEFER` مش error حقيقي، هو "اسألني تاني بعدين".

---

### Memory Management — الـ `__free` Pattern

الكود بيستخدم الـ cleanup attribute الجديد (kernel 6.x):

```c
u64 *rates_64 __free(kfree) = NULL;
u32 *rates    __free(kfree) = NULL;
```

ده بيضمن إن الـ array اتعملها `kcalloc` هتتحرر تلقائيًا لما الـ function تخرج — سواء بـ `return` عادية أو بـ `return rc` بعد error. مفيش memory leak ممكن حتى لو الـ error path معقد.
## Phase 4: شرح الـ Functions

### ملخص الـ Functions — Cheatsheet

| Function | النوع | الغرض |
|---|---|---|
| `of_clk_set_defaults()` | Public API / `EXPORT_SYMBOL_GPL` | Entry point — يطبق كل الـ assigned-clock settings من DT |
| `__set_clk_parents()` | Static Internal | يقرأ `assigned-clock-parents` ويعمل `clk_set_parent()` |
| `__set_clk_rates()` | Static Internal | يقرأ `assigned-clock-rates` / `assigned-clock-rates-u64` ويعمل `clk_set_rate()` |

**الـ Kernel APIs المستخدمة داخل الملف:**

| API | من وين | الغرض |
|---|---|---|
| `of_count_phandle_with_args()` | `linux/of.h` | يعد عدد الـ phandles في property |
| `of_parse_phandle_with_args()` | `linux/of.h` | يـ parse phandle واحد من property |
| `of_clk_get_from_provider()` | `linux/clk.h` | يجيب `struct clk *` من الـ clk provider registry |
| `of_node_put()` | `linux/of.h` | يـ release reference على `device_node` |
| `clk_set_parent()` | `linux/clk.h` | يغير parent الـ clock |
| `clk_set_rate()` | `linux/clk.h` | يضبط الـ rate بالـ Hz |
| `clk_get_rate()` | `linux/clk.h` | يقرأ الـ rate الحالي (للـ error logging) |
| `clk_put()` | `linux/clk.h` | يـ release reference على `struct clk` |
| `__clk_get_name()` | `linux/clk-provider.h` | يجيب اسم الـ clock (للـ logging) |
| `of_property_count_u32_elems()` | `linux/of.h` | يعد العناصر في property من نوع u32 |
| `of_property_count_u64_elems()` | `linux/of.h` | يعد العناصر في property من نوع u64 |
| `of_property_read_u32_array()` | `linux/of.h` | يقرأ array من u32 من DT property |
| `of_property_read_u64_array()` | `linux/of.h` | يقرأ array من u64 من DT property |
| `kcalloc()` | `linux/slab.h` | يـ allocate zero-initialized array |

---

### المجموعة الوحيدة: DT Clock Configuration

الـ file ده بيعمل حاجة واحدة محددة: يقرأ الـ Device Tree properties الخاصة بالـ `assigned-clocks` ويطبقها على الـ clock tree وقت الـ probe. الفكرة إن الـ SoC vendor أو الـ board file بيحدد في الـ DT مين هو الـ parent المفضل لكل clock ومين هي الـ rate المطلوبة، والـ kernel بيطبق ده أوتوماتيك من غير ما كل driver يكتب كود initialization خاص بيه.

الـ flow العام:

```
of_clk_set_defaults(node, clk_supplier)
    ├── __set_clk_parents(node, clk_supplier)
    │       └── for each index in assigned-clock-parents:
    │               parse parent phandle → of_clk_get_from_provider()
    │               parse clock phandle  → of_clk_get_from_provider()
    │               clk_set_parent(clk, pclk)
    │               clk_put(clk); clk_put(pclk)
    │
    └── __set_clk_rates(node, clk_supplier)
            ├── read all rates (u64 preferred over u32)
            └── for each index where rate != 0:
                    parse clock phandle → of_clk_get_from_provider()
                    clk_set_rate(clk, rate)
                    clk_put(clk)
```

---

### `of_clk_set_defaults()`

```c
int of_clk_set_defaults(struct device_node *node, bool clk_supplier);
```

**الـ function دي هي الـ public API الوحيدة في الـ file.** بتشتغل كـ orchestrator — تستدعي `__set_clk_parents()` الأول عشان تضبط الـ topology، وبعدين `__set_clk_rates()` عشان تضبط الـ frequencies. الترتيب مهم: لازم تعرف الـ parent الصح الأول قبل ما تضبط الـ rate لأن الـ rate calculation بتعتمد على الـ parent.

**الـ Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `node` | `struct device_node *` | الـ DT node اللي فيه الـ `assigned-clocks` properties. لو `NULL` الـ function ترجع 0 فوراً |
| `clk_supplier` | `bool` | لو `true`، الـ function بتطبق الـ settings حتى لو الـ node نفسه هو الـ clock provider. لو `false` وشافت إن الـ clock منسوب لنفس الـ node، بتوقف وترجع 0 |

**الـ Return Value:**
- `0` — نجاح أو مفيش settings.
- `negative errno` — فشل، أهمهم `-EPROBE_DEFER` لو الـ clock provider لسه ما اتسجلش (الـ driver محتاج يتأجل).

**Key Details:**
- الـ function دي مش بتـ acquire أي lock بنفسها، الـ locking بيحصل داخل `clk_set_parent()` و`clk_set_rate()` اللي بيمسكوا الـ `prepare_lock` (mutex) الداخلي للـ CCF.
- بيتم استدعاؤها من `of_clk_init()` لما الـ clk provider بيتسجل، ومن `driver_probe_device()` في الـ driver core قبل ما يبدأ الـ probe.
- **الـ `clk_supplier` flag** بيحل مشكلة الـ chicken-and-egg: لو الـ node هو نفسه بيوفر الـ clock اللي محتاج يتعدل، يبقى لازم يتسجل الأول كـ provider ثم نطبق عليه الـ settings — ده بيتعمل بـ `clk_supplier = true`.

---

### `__set_clk_parents()`

```c
static int __set_clk_parents(struct device_node *node, bool clk_supplier);
```

**الـ function دي بتـ iterate على كل element في الـ `assigned-clock-parents` property** وبتـ pair كل parent مع الـ clock المقابل من `assigned-clocks`، وبتستدعي `clk_set_parent()`. الـ pairing بيعتمد على الـ index — العنصر رقم N في `assigned-clock-parents` هو الـ parent المطلوب للعنصر رقم N في `assigned-clocks`.

**الـ Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `node` | `struct device_node *` | الـ DT node صاحب الـ properties |
| `clk_supplier` | `bool` | نفس المعنى اللي في `of_clk_set_defaults()` |

**الـ Return Value:**
- `0` — نجاح أو مفيش parents محددة.
- `negative errno` — فشل الـ parse أو الـ `clk_set_parent()`.

**Pseudocode Flow:**

```c
num_parents = of_count_phandle_with_args(node, "assigned-clock-parents", "#clock-cells");

// لو -EINVAL يبقى في خطأ في الـ DT نفسه
if (num_parents == -EINVAL) pr_err(...);

for (index = 0; index < num_parents; index++) {
    // parse الـ parent phandle
    rc = of_parse_phandle_with_args(node, "assigned-clock-parents", "#clock-cells", index, &clkspec);
    if (rc == -ENOENT) continue;  // null phandle → skip
    if (rc < 0)        return rc;

    // لو الـ parent منسوب لنفس الـ node ومش clk_supplier → bail out
    if (clkspec.np == node && !clk_supplier) { of_node_put(...); return 0; }

    pclk = of_clk_get_from_provider(&clkspec);
    of_node_put(clkspec.np);  // release الـ ref على الـ parent node
    if (IS_ERR(pclk)) return PTR_ERR(pclk);

    // parse الـ clock اللي هيتغير parent-ه
    rc = of_parse_phandle_with_args(node, "assigned-clocks", "#clock-cells", index, &clkspec);
    if (rc < 0) goto err;
    if (clkspec.np == node && !clk_supplier) { of_node_put(...); rc = 0; goto err; }

    clk = of_clk_get_from_provider(&clkspec);
    of_node_put(clkspec.np);
    if (IS_ERR(clk)) { rc = PTR_ERR(clk); goto err; }

    rc = clk_set_parent(clk, pclk);
    clk_put(clk);
    clk_put(pclk);
}
return 0;

err:
    clk_put(pclk);  // pclk ممكن تكون اتحجزت لو فشلنا بعد الـ parse
    return rc;
```

**Key Details:**
- **Null phandles** في الـ DT (زي `<0>`) بترجع `-ENOENT` وبيتعملها `continue` مش `return`، عشان ممكن تسكيب slot معين.
- لو `of_clk_get_from_provider()` رجعت `-EPROBE_DEFER` الـ function دي بترجعها للـ caller فوراً من غير warning، عشان ده سيناريو طبيعي وقت الـ boot.
- الـ `of_node_put()` بيتعمل دايماً بعد الـ `of_parse_phandle_with_args()` مباشرة عشان يمنع الـ reference leak على الـ device_node.
- **Error path:** لو حصل error بعد ما اتحجز `pclk` بس قبل `clk_put(pclk)`، الـ `goto err` بيضمن الـ release.

---

### `__set_clk_rates()`

```c
static int __set_clk_rates(struct device_node *node, bool clk_supplier);
```

**الـ function دي بتقرأ الـ rates من الـ DT وبتطبقها على الـ clocks المقابلة.** بتدعم نوعين من الـ properties: `assigned-clock-rates` (u32، بالـ Hz، max ~4.29 GHz) و`assigned-clock-rates-u64` (u64، بيغطي كل الـ modern high-frequency cases). الـ u64 property ليها أولوية لو الاتنين موجودين.

**الـ Parameters:**

| Parameter | النوع | الشرح |
|---|---|---|
| `node` | `struct device_node *` | الـ DT node صاحب الـ properties |
| `clk_supplier` | `bool` | نفس المعنى |

**الـ Return Value:**
- `0` — نجاح أو مفيش rates محددة.
- `negative errno` — فشل الـ allocation أو الـ parse أو error في `clk_set_rate()`.

**Pseudocode Flow:**

```c
count    = of_property_count_u32_elems(node, "assigned-clock-rates");
count_64 = of_property_count_u64_elems(node, "assigned-clock-rates-u64");

if (count_64 > 0) {
    // u64 ليها أولوية
    count = count_64;
    rates_64 = kcalloc(count, sizeof(u64), GFP_KERNEL);  // __free(kfree) → auto-freed
    rc = of_property_read_u64_array(node, "assigned-clock-rates-u64", rates_64, count);
} else if (count > 0) {
    rates = kcalloc(count, sizeof(u32), GFP_KERNEL);  // __free(kfree) → auto-freed
    rc = of_property_read_u32_array(node, "assigned-clock-rates", rates, count);
} else {
    return 0;  // مفيش rates خالص
}

if (rc) return rc;

for (index = 0; index < count; index++) {
    rate = rates_64 ? rates_64[index] : rates[index];

    if (rate == 0) continue;  // 0 = "don't change this clock"

    rc = of_parse_phandle_with_args(node, "assigned-clocks", "#clock-cells", index, &clkspec);
    if (rc == -ENOENT) continue;
    if (rc < 0)        return rc;

    if (clkspec.np == node && !clk_supplier) { of_node_put(...); return 0; }

    clk = of_clk_get_from_provider(&clkspec);
    of_node_put(clkspec.np);
    if (IS_ERR(clk)) return PTR_ERR(clk);

    rc = clk_set_rate(clk, rate);
    if (rc < 0)
        pr_err("... current rate: %lu\n", clk_get_rate(clk));  // للـ debug
    clk_put(clk);
}
return 0;
```

**Key Details:**
- **الـ `__free(kfree)` attribute** هو الـ kernel cleanup mechanism الجديد (من kernel 6.x) اللي بيـ auto-free الـ pointer لما يخرج من الـ scope، سواء بـ `return` عادي أو بـ error path — من غير ما تحتاج `kfree()` explicit.
- **Rate = 0 في الـ DT** معناه "متغيرش الـ clock ده"، بيتعمله `continue` مش error.
- لو `clk_set_rate()` فشلت، الـ function **بتعمل `pr_err` بس مش بترجع error**، كده بتكمل على الـ clocks التانية. ده قرار design conscious — فشل ضبط rate واحدة ما يمنعش الباقي. لو مهم للـ system الـ driver نفسه يتحقق من الـ rate بعد الـ probe.
- الـ `clk_get_rate()` في الـ error message بتجيب الـ rate الفعلية اللي الـ hardware شتغل عليها للـ comparison مع الـ requested rate.
- **لا locks صريحة** في الـ function: الـ `clk_set_rate()` بيأخذ الـ `prepare_lock` داخلياً في الـ CCF، فمفيش حاجة يعملها الـ caller.

---

### نقطة تصميم مهمة: الـ `clk_supplier` Guard

الـ guard pattern ده بيتكرر في الاتنين `__set_clk_parents` و`__set_clk_rates`:

```c
if (clkspec.np == node && !clk_supplier) {
    of_node_put(clkspec.np);
    return 0;  // أو rc = 0; goto err;
}
```

**المشكلة اللي بيحلها:** لو driver هو نفسه الـ clock provider للـ clocks اللي عايزين نعدلها، وأحنا لسه في مرحلة الـ `of_clk_init()` (قبل ما يتسجل كـ provider)، الـ `of_clk_get_from_provider()` هتفشل بـ `-EPROBE_DEFER` لأن الـ provider مش موجود في الـ registry لسه.

**الحل:** لما بنستدعي من `of_clk_init()` بنبعت `clk_supplier = false`، وده بيخلي الـ function تبوق لما تلاقي إن الـ clock منسوب لنفس الـ node اللي بيتعمل فيه init — وده **طبيعي ومقصود**، لأن الـ clock provider هيطبق الـ settings بنفسه بعد ما يتسجل. لما بيبعت `clk_supplier = true` يبقى الـ provider نفسه هو اللي بيطلب التطبيق بعد ما اتسجل فعلاً.
## Phase 5: دليل الـ Debugging الشامل

الـ `clk-conf.c` مسؤول عن قراءة الـ Device Tree وتطبيق إعدادات الـ clocks (parent + rate) على الأجهزة أثناء الـ probe. الـ debugging هنا يتمركز حول: هل اتقرأت الـ DT properties صح؟ هل اتطبقت الـ parent/rate صح؟ وليه بيفشل الـ probe.

---

### Software Level

#### 1. debugfs Entries

**الـ clock framework** بيوفر debugfs غني جداً تحت `/sys/kernel/debug/clk/`:

```
/sys/kernel/debug/clk/
├── clk_summary          # ملخص شامل لكل الـ clocks (hierarchy + rate + enable_count)
├── clk_dump             # نفس المعلومات بصيغة JSON
├── clk_orphan_summary   # clocks مش لاقية parent
└── <clock_name>/
    ├── clk_rate         # الـ rate الحالية بالـ Hz
    ├── clk_parent       # اسم الـ parent الحالي
    ├── clk_enable_count # عدد مرات الـ enable
    ├── clk_prepare_count
    ├── clk_flags        # CLK_SET_RATE_PARENT وغيرها
    └── clk_possible_parents  # كل الـ parents الممكنة
```

```bash
# اقرأ الـ summary الكامل — أهم أمر
cat /sys/kernel/debug/clk/clk_summary

# شوف clock معين بالاسم (مثلاً: pll0)
cat /sys/kernel/debug/clk/pll0/clk_rate
cat /sys/kernel/debug/clk/pll0/clk_parent
cat /sys/kernel/debug/clk/pll0/clk_possible_parents

# شوف الـ orphan clocks — دي بتظهر لو الـ parent مش موجود
cat /sys/kernel/debug/clk/clk_orphan_summary
```

**كيف تقرأ الـ clk_summary:**
```
                                 enable  prepare  protect                               duty
   clock                          count    count    count        rate   accuracy phase  cycle
----------------------------------------------------------------------------------------------------
 osc_clk                              1        1        0    24000000          0     0  50000
    pll0                              1        1        0   600000000          0     0  50000
       uart_clk                       1        1        0    50000000          0     0  50000
```
الـ `enable_count = 0` مع device شغال = مشكلة. الـ `rate = 0` = الـ clock مش اتضبط.

---

#### 2. sysfs Entries

```
/sys/bus/platform/devices/<dev>/
├── of_node -> /sys/firmware/devicetree/base/<path>   # DT node
└── driver/                                            # هل اتعمله bind؟

/sys/firmware/devicetree/base/<node-path>/
├── assigned-clocks          # binary phandle data
├── assigned-clock-parents   # binary phandle data
└── assigned-clock-rates     # binary u32 array
```

```bash
# تحقق إن الـ DT properties موجودة على الـ node
ls /sys/firmware/devicetree/base/soc/uart@1234/

# اقرأ الـ assigned-clock-rates كـ hex
xxd /sys/firmware/devicetree/base/soc/uart@1234/assigned-clock-rates

# تحقق من ربط الـ driver
ls -la /sys/bus/platform/devices/1234.uart/driver
```

---

#### 3. ftrace

الـ tracepoints المهمة في clock subsystem:

```bash
# فعّل الـ clock tracepoints
echo 1 > /sys/kernel/debug/tracing/events/clk/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_parent
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate_complete
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_parent_complete

# أو فعّل كل أحداث الـ clk دفعة واحدة
echo 1 > /sys/kernel/debug/tracing/events/clk/enable

# شغّل الـ tracing
echo 1 > /sys/kernel/debug/tracing/tracing_on

# trigger الـ probe (مثلاً rebind الـ driver)
echo '1234.uart' > /sys/bus/platform/drivers/my-driver/unbind
echo '1234.uart' > /sys/bus/platform/drivers/my-driver/bind

# اقرأ النتيجة
cat /sys/kernel/debug/tracing/trace
```

**مثال على الـ output:**
```
          <idle>-0   [000] ....  123.456: clk_set_parent: clock=uart_clk parent=pll0
          <idle>-0   [000] ....  123.457: clk_set_parent_complete: clock=uart_clk parent=pll0
          <idle>-0   [000] ....  123.458: clk_set_rate: clock=uart_clk rate=50000000
```

للـ function tracing على `of_clk_set_defaults`:

```bash
echo 'of_clk_set_defaults' > /sys/kernel/debug/tracing/set_ftrace_filter
echo '__set_clk_parents' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo '__set_clk_rates' >> /sys/kernel/debug/tracing/set_ftrace_filter
echo 'function' > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace
```

---

#### 4. printk / Dynamic Debug

الـ `clk-conf.c` بيستخدم `pr_err` و`pr_warn` مباشرة، فالرسائل بتظهر في الـ kernel log بدون تفعيل إضافي.

لتفعيل **dynamic debug** على مستوى الـ module:

```bash
# فعّل كل الـ debug messages في clk subsystem
echo 'module clk-conf +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/clk/clk-conf.c +p' > /sys/kernel/debug/dynamic_debug/control

# فعّل مع line numbers و function names
echo 'file drivers/clk/clk-conf.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# فعّل الـ debug على كل الـ clk framework
echo 'module clk +p' > /sys/kernel/debug/dynamic_debug/control

# تابع الـ kernel messages
dmesg -w | grep -E 'clk:|clock'
# أو
journalctl -k -f | grep -i clk
```

---

#### 5. Kernel Config Options للـ Debugging

| Config Option | الوظيفة |
|---|---|
| `CONFIG_COMMON_CLK` | لازم يكون enabled — بدونه الكود ده stub |
| `CONFIG_CLK_DEBUG` | يفعّل الـ debugfs entries تحت `/sys/kernel/debug/clk/` |
| `CONFIG_DEBUG_KERNEL` | base للـ debugging |
| `CONFIG_OF` | لازم enabled علشان الـ DT parsing يشتغل |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل الـ dynamic debug messages |
| `CONFIG_PROVE_LOCKING` | يكشف deadlocks في الـ clock mutex |
| `CONFIG_DEBUG_OBJECTS` | يكشف use-after-free في الـ clk objects |
| `CONFIG_CLK_ZYNQ` / platform clk | لو بتتعامل مع platform معين |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(COMMON_CLK|CLK_DEBUG|OF|DYNAMIC_DEBUG)'
# أو
grep -E 'CONFIG_(COMMON_CLK|CLK_DEBUG|OF)' /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

**الـ clk_summary** هو الأداة الأقوى:

```bash
# الملخص الكامل مع الـ hierarchy
cat /sys/kernel/debug/clk/clk_summary | column -t

# ابحث عن clock معين
cat /sys/kernel/debug/clk/clk_summary | grep -A2 "uart"

# الـ JSON dump — مفيد للـ scripting
cat /sys/kernel/debug/clk/clk_dump | python3 -m json.tool | grep -A5 "uart"

# أداة clk_summary الخاصة في بعض الـ SoCs
# Qualcomm:
cat /sys/kernel/debug/clk/gcc_blsp1_uart2_apps_clk/clk_rate

# أداة لمقارنة الـ requested vs actual rate
cat /sys/kernel/debug/clk/<clock_name>/clk_rate
```

**devlink:** مش relevant لهذا الـ subsystem مباشرة — `clk-conf.c` هو pure clock framework.

---

#### 7. جدول رسائل الـ Error الشائعة

| رسالة الـ Kernel Log | المعنى | الإصلاح |
|---|---|---|
| `clk: invalid value of clock-parents property at /soc/uart@1234` | الـ `assigned-clock-parents` property فيها قيمة غلط (مش phandle صحيح) | راجع الـ DTS — تأكد أن الـ phandle بيشاور على node بيحتوي `#clock-cells` |
| `clk: couldn't get parent clock 0 for /soc/uart@1234` | `of_clk_get_from_provider` فشل — الـ parent clock مش متسجل | الـ parent clock driver مش اتعمله probe بعد — غالباً مشكلة ordering |
| `clk: couldn't get assigned clock 0 for /soc/uart@1234` | نفس المشكلة لكن على الـ child clock | تأكد أن الـ `assigned-clocks` bيشاور صح |
| `clk: failed to reparent <clk> to <parent>: -22` | `clk_set_parent` رجع `-EINVAL` | الـ parent مش في قائمة الـ possible parents |
| `clk: failed to reparent <clk> to <parent>: -16` | `clk_set_parent` رجع `-EBUSY` | الـ clock مستخدم حالياً وبيرفض الـ reparent |
| `clk: couldn't set <clk> clk rate to 50000000 (-22), current rate: 24000000` | `clk_set_rate` فشل | الـ rate مش supported، أو خارج الـ range المسموح |
| `clk: couldn't set <clk> clk rate to 50000000 (-16), current rate: 24000000` | الـ rate exclusive locked | في driver تاني قافل الـ rate |
| صمت تام — لا parent ولا rate | `num_parents <= 0` أو `count <= 0` | الـ DT properties غير موجودة أصلاً — تحقق بـ `dtc -I fs /sys/firmware/devicetree/base` |
| `-EPROBE_DEFER` (مش رسالة ظاهرة) | الـ clock provider لسه مش جاهز | طبيعي — الـ kernel هيعيد الـ probe، لو استمرت راجع الـ boot order |

---

#### 8. نقاط استراتيجية لـ dump_stack() و WARN_ON()

```c
/* في __set_clk_parents — بعد فشل clk_set_parent */
rc = clk_set_parent(clk, pclk);
if (rc < 0) {
    pr_err("clk: failed to reparent %s to %s: %d\n",
           __clk_get_name(clk), __clk_get_name(pclk), rc);
    /* هنا ضيف WARN_ON لو عايز stack trace */
    WARN_ON(rc == -EINVAL); /* parent غير صالح — يستحق تحقيق */
    dump_stack();           /* للـ development فقط، شيله في production */
}

/* في __set_clk_rates — بعد فشل clk_set_rate */
rc = clk_set_rate(clk, rate);
if (rc < 0) {
    pr_err("clk: couldn't set %s clk rate to %lu (%d), current rate: %lu\n",
           __clk_get_name(clk), rate, rc, clk_get_rate(clk));
    WARN_ON(rc == -EBUSY); /* يكشف من القافل الـ rate */
}

/* نقطة مهمة: قبل of_clk_get_from_provider */
/* لو IS_ERR(pclk) && PTR_ERR(pclk) != -EPROBE_DEFER */
/* ده يعني provider مسجّل بس الـ clkspec غلط */
WARN_ON(!IS_ERR(pclk) == false && PTR_ERR(pclk) != -EPROBE_DEFER);
```

---

### Hardware Level

#### 1. التحقق من تطابق الـ Hardware مع الـ Kernel State

```bash
# قارن الـ rate المتوقعة من الـ DT مع الـ actual hardware rate
CLOCK_NAME="uart_clk"
EXPECTED_RATE=$(grep -r "assigned-clock-rates" /sys/firmware/devicetree/base/soc/ 2>/dev/null | head -1)
ACTUAL_RATE=$(cat /sys/kernel/debug/clk/${CLOCK_NAME}/clk_rate 2>/dev/null)
echo "Expected: ${EXPECTED_RATE}, Actual: ${ACTUAL_RATE}"

# تحقق أن الـ parent الحالي هو اللي في الـ DT
cat /sys/kernel/debug/clk/${CLOCK_NAME}/clk_parent

# تحقق من الـ enable count — لو 0 والـ device شغال فيه مشكلة
cat /sys/kernel/debug/clk/${CLOCK_NAME}/clk_enable_count
```

**مثال عملي على Raspberry Pi:**
```bash
# تحقق من uart clock
cat /sys/kernel/debug/clk/uart0_clk/clk_rate   # يجب يكون 48000000
cat /sys/kernel/debug/clk/uart0_clk/clk_parent  # يجب يكون "pllc_per"
```

---

#### 2. Register Dump

الـ `clk-conf.c` نفسه ما بيتعامل مع registers مباشرة — ده شغل الـ clock provider driver. لكن ممكن تتحقق يدوياً:

```bash
# devmem2 — اقرأ register الـ clock control بالـ physical address
# (الـ address بييجي من datasheet الـ SoC)
devmem2 0xF8000100 w    # مثال: Zynq UART clock register

# لو devmem2 مش متاح، استخدم /dev/mem
dd if=/dev/mem bs=4 count=1 skip=$((0xF8000100/4)) 2>/dev/null | xxd

# أو استخدم io utility
io -4 0xF8000100

# على ARM SoCs، الـ CCF registers غالباً في CMU/CRU
# مثال على Rockchip RK3399:
devmem2 0xFF760000 w  # CRU base
```

**تحقق من الـ PLL lock bit:**
```bash
# لو الـ PLL مش locked، الـ rate هتبقى غلط
# الـ lock bit عادةً في register الـ PLL status
devmem2 0xFF760008 w  # PLL status register (مثال)
# bit معين = 1 → PLL locked
```

---

#### 3. Logic Analyzer / Oscilloscope Tips

| ما تقيسه | الـ Test Point | القيمة المتوقعة |
|---|---|---|
| الـ UART clock output | TX pin في idle (high) — قيس الـ baud rate | يجب يتطابق مع الـ configured rate |
| الـ PLL output | clock test pad على الـ SoC (لو متاح) | المتوقع من الـ DT |
| الـ parent clock | XTAL pins | عادةً 24MHz أو 25MHz أو 48MHz |

```
Oscilloscope Setup للـ clock debugging:
- Sample rate: 10× الـ clock المتوقع على الأقل
- Trigger: Rising edge على الـ clock signal
- قيس: Frequency, Duty Cycle, Jitter
- لو الـ frequency غلط بنسبة ثابتة (مثلاً نص القيمة): الـ divider غلط
- لو في jitter كبير: الـ PLL مش locked أو في noise في الـ power supply
```

**Logic Analyzer للـ I2C/SPI clock:**
```
Protocol: I2C
Clock: SCL pin
Data: SDA pin
قارن الـ SCL frequency مع assigned-clock-rates / divider المتوقع
```

---

#### 4. Hardware Issues → Kernel Log Patterns

| المشكلة الـ Hardware | Pattern في الـ Kernel Log |
|---|---|
| الـ PLL مش locked (power/stability issue) | `clk: couldn't set <clk> clk rate to X` + rate مختلفة تماماً |
| الـ crystal oscillator مش شغال | كل الـ clocks بـ rate = 0 أو بـ ENODEV |
| مشكلة في الـ voltage regulator للـ PLL | `clk: failed to reparent` + `-EIO` |
| الـ clock enable signal مش واصل (GPIO/regulator) | `clk: couldn't get parent clock` بعد timeout |
| الـ reference clock مقطوع أو بـ frequency غلط | الـ rates الفعلية = كسور من القيم المتوقعة |

---

#### 5. Device Tree Debugging

```bash
# dump الـ DT الكامل من الـ running kernel
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | grep -A 20 "uart@1234"

# تحقق من الـ assigned-clocks properties تحديداً
dtc -I fs /sys/firmware/devicetree/base -O dts 2>/dev/null | \
    grep -B2 -A5 "assigned-clock"

# اقرأ الـ phandle values (binary) وترجمها
# phandle هو 4 bytes big-endian integer
hexdump -C /sys/firmware/devicetree/base/soc/uart@1234/assigned-clock-parents

# ابحث عن الـ node اللي بيحتوي الـ phandle ده
PHANDLE=0x00000015  # مثال
grep -r "phandle" /sys/firmware/devicetree/base/ | grep "${PHANDLE}"

# تحقق أن الـ clock provider node عنده #clock-cells
cat /sys/firmware/devicetree/base/soc/clk-controller@ff760000/\#clock-cells

# تحقق من الـ compatible string للـ clock provider
cat /sys/firmware/devicetree/base/soc/clk-controller@ff760000/compatible
```

**أخطاء DT الشائعة في `clk-conf.c`:**

```dts
/* خطأ شائع 1: عدد الـ assigned-clocks != عدد الـ assigned-clock-rates */
uart0: uart@1234 {
    assigned-clocks = <&cru CLK_UART0>, <&cru CLK_UART1>;  /* 2 clocks */
    assigned-clock-rates = <50000000>;                       /* بس 1 rate — غلط! */
};

/* خطأ شائع 2: الـ node نفسه supplier وconsumer بدون clk_supplier=true */
cru: clock-controller@ff760000 {
    assigned-clocks = <&cru ARMCLK>;      /* بيشاور على نفسه */
    assigned-clock-rates = <600000000>;
    /* لازم of_clk_set_defaults(node, true) — مش false */
};

/* الصح */
uart0: uart@1234 {
    assigned-clocks = <&cru CLK_UART0>;
    assigned-clock-parents = <&cru PLL_GPLL>;
    assigned-clock-rates = <50000000>;
};
```

---

### Practical Commands

#### أوامر جاهزة للنسخ

**1. فحص سريع لكل الـ clocks:**
```bash
#!/bin/bash
# quick-clk-check.sh — فحص سريع لحالة الـ clocks

echo "=== Clock Summary ==="
cat /sys/kernel/debug/clk/clk_summary | head -50

echo ""
echo "=== Orphan Clocks (مشكلة!) ==="
cat /sys/kernel/debug/clk/clk_orphan_summary

echo ""
echo "=== Clocks with rate=0 ==="
cat /sys/kernel/debug/clk/clk_summary | awk '$6 == 0 {print $0}'
```

**2. تتبع probe مع clock info:**
```bash
# فعّل الـ tracing قبل الـ probe
echo 1 > /sys/kernel/debug/tracing/events/clk/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# أعد تحميل الـ driver
DEVICE="1234.uart"
echo $DEVICE > /sys/bus/platform/drivers/serial8250/unbind 2>/dev/null
echo $DEVICE > /sys/bus/platform/drivers/serial8250/bind 2>/dev/null

# اقرأ النتيجة
echo 0 > /sys/kernel/debug/tracing/tracing_on
cat /sys/kernel/debug/tracing/trace | grep -E "clk_set_(rate|parent)"
```

**3. مقارنة DT مع الـ actual state:**
```bash
#!/bin/bash
# verify-clk-conf.sh — تحقق أن clk-conf.c طبّق الـ DT صح

DEVICE_NODE="/sys/firmware/devicetree/base/soc/uart@1234"
CLK_NAME="uart_clk"

# الـ expected rate من الـ DT
EXPECTED=$(od -An -tu4 ${DEVICE_NODE}/assigned-clock-rates 2>/dev/null | tr -d ' ')
echo "Expected rate from DT: ${EXPECTED} Hz"

# الـ actual rate من الـ kernel
ACTUAL=$(cat /sys/kernel/debug/clk/${CLK_NAME}/clk_rate 2>/dev/null)
echo "Actual rate in kernel: ${ACTUAL} Hz"

if [ "$EXPECTED" = "$ACTUAL" ]; then
    echo "OK: rates match"
else
    echo "MISMATCH! DT=${EXPECTED}, Actual=${ACTUAL}"
fi
```

**4. تفعيل dynamic debug وتتبع:**
```bash
# فعّل debug لـ clk-conf.c
echo 'file drivers/clk/clk-conf.c +pflmt' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ messages
dmesg -c  # clear
echo '1234.uart' > /sys/bus/platform/drivers/my_driver/unbind
echo '1234.uart' > /sys/bus/platform/drivers/my_driver/bind
dmesg | grep -E 'clk:|clk-conf'
```

**5. dump كامل لـ clock لـ device معين:**
```bash
#!/bin/bash
# dump-device-clks.sh
DEVICE_PATH="/sys/firmware/devicetree/base/soc/uart@1234"

echo "=== assigned-clocks (raw phandles) ==="
xxd ${DEVICE_PATH}/assigned-clocks 2>/dev/null || echo "not found"

echo ""
echo "=== assigned-clock-parents ==="
xxd ${DEVICE_PATH}/assigned-clock-parents 2>/dev/null || echo "not found"

echo ""
echo "=== assigned-clock-rates (Hz) ==="
od -An -tu4 ${DEVICE_PATH}/assigned-clock-rates 2>/dev/null || echo "not found"

echo ""
echo "=== assigned-clock-rates-u64 (Hz, 64-bit) ==="
od -An -tu8 ${DEVICE_PATH}/assigned-clock-rates-u64 2>/dev/null || echo "not found"
```

---

**مثال على الـ output وتفسيره:**

```
$ cat /sys/kernel/debug/clk/clk_summary | grep uart
                        enable  prepare  protect             duty
   clock                 count    count    count    rate     cycle
   uart0_clk                 1        1        0    48000000  50000
```

| الحقل | القيمة | التفسير |
|---|---|---|
| `enable_count` | 1 | الـ clock enabled — طبيعي لو الـ device شغال |
| `prepare_count` | 1 | الـ clock prepared — لازم يساوي enable_count |
| `protect_count` | 0 | مفيش exclusive rate lock — طبيعي |
| `rate` | 48000000 | 48MHz — طابق مع الـ DT assigned-clock-rates |

لو `rate = 24000000` والـ DT بيطلب `48000000` → الـ `clk_set_rate` فشل أو الـ `assigned-clock-rates` property مش موجودة. راجع `dmesg | grep "clk: couldn't set"`.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Industrial Gateway على RK3562 — UART لا يشتغل بالسرعة الصح

#### العنوان
**assigned-clock-rates** مش بتتطبق على UART في gateway صناعي

#### السياق
شركة بتعمل **industrial gateway** على بورد مبنية على **RK3562**. الـ gateway بتستخدم UART2 للتواصل مع Modbus RTU بـ 921600 baud. البورد اشتغلت على kernel قديم، وبعد upgrade للـ kernel الجديد، الـ UART بدأت تيجي frames مشوهة وتحصل communication errors.

#### المشكلة
الـ UART clock مش بتتضبط على القيمة المطلوبة. الـ DT فيها:

```dts
&uart2 {
    assigned-clocks = <&cru SCLK_UART2>;
    assigned-clock-rates = <921600>;
    status = "okay";
};
```

الـ baud rate محتاج clock مضبوط على **24MHz** مش 921600 مباشرة، لأن الـ UART driver بيعمل division داخلي.

#### التحليل
الكود في `__set_clk_rates()` بيقرأ القيمة من **`assigned-clock-rates`** كـ u32:

```c
count = of_property_count_u32_elems(node, "assigned-clock-rates");
/* ... */
rc = of_property_read_u32_array(node, "assigned-clock-rates", rates, count);
```

ثم بيعمل `clk_set_rate(clk, rate)` بالقيمة دي مباشرة. المشكلة مش في `clk-conf.c` نفسه — الكود بيمرر 921600 Hz لـ `clk_set_rate()` وده صح من الناحية التقنية. لكن الـ CCU على RK3562 مش بيقدر يولد 921600 Hz بالظبط، فبيعمل **rounding** لأقرب قيمة ممكنة وبيطبعها في kernel log:

```
clk: couldn't set SCLK_UART2 clk rate to 921600 (0), current rate: 24000000
```

الـ engineer فهم غلط — هو شايف الـ "current rate: 24000000" وفاكر إن الـ clock اتضبطت، بس الحقيقة إن الـ driver نفسه محتاج source clock = 24MHz وهو بيعمل divide بـ 26 عشان يوصل لـ 921600 baud. لما حط 921600 في `assigned-clock-rates`، الـ source clock اتغيرت وبقى الـ division table مش صح.

#### الحل
تصحيح قيمة الـ `assigned-clock-rates` لتكون الـ source clock المطلوبة:

```dts
&uart2 {
    assigned-clocks = <&cru SCLK_UART2>;
    assigned-clock-rates = <24000000>;  /* 24 MHz source clock */
    status = "okay";
};
```

وللتأكد من التطبيق الفعلي:

```bash
# تحقق من الـ rate اللي اتطبق فعلاً
cat /sys/kernel/debug/clk/sclk_uart2/clk_rate

# شوف الـ kernel log أثناء probe
dmesg | grep "clk:"

# تحقق من الـ baud error
cat /sys/class/tty/ttyS2/../../uevent
```

#### الدرس المستفاد
**`assigned-clock-rates`** بتضبط الـ **source clock** للبيريفيرال، مش الـ baud rate أو الـ final frequency. لازم تفهم الـ clock tree كامل قبل ما تحط أي قيمة — الـ `clk-conf.c` بيمرر القيمة حرفياً لـ `clk_set_rate()` من غير أي interpretation.

---

### السيناريو الثاني: Android TV Box على Allwinner H616 — HDMI مش بتطلع صورة

#### العنوان
**assigned-clock-parents** غلط بيخلي HDMI PLL مش بتتقفل

#### السياق
منتج **Android TV Box** على **Allwinner H616**. الـ display pipeline محتاجة HDMI-TX clock تيجي من **PLL_VIDEO0** لدعم 4K@30fps. البورد بتبوت بس الشاشة بتفضل سودة. الـ HDMI port مش بتطلع صورة خالص.

#### المشكلة
الـ DT فيها:

```dts
&de {
    assigned-clocks = <&ccu CLK_DE>, <&ccu CLK_HDMI>;
    assigned-clock-parents = <&ccu CLK_PLL_PERIPH0>, <&ccu CLK_PLL_VIDEO1>;
    assigned-clock-rates = <300000000>, <594000000>;
};
```

الـ engineer حط `CLK_PLL_VIDEO1` بدل `CLK_PLL_VIDEO0` كـ parent للـ HDMI clock، والـ VIDEO1 مش بتدعم 594MHz اللي محتاجها TMDS clock لـ 4K@30.

#### التحليل
الكود في `__set_clk_parents()` بيمشي على كل entry في `assigned-clock-parents`:

```c
num_parents = of_count_phandle_with_args(node, "assigned-clock-parents",
                                          "#clock-cells");
for (index = 0; index < num_parents; index++) {
    rc = of_parse_phandle_with_args(node, "assigned-clock-parents",
                        "#clock-cells", index, &clkspec);
    /* ... */
    pclk = of_clk_get_from_provider(&clkspec);
    /* ... */
    clk = of_clk_get_from_provider(&clkspec);  /* from assigned-clocks */
    /* ... */
    rc = clk_set_parent(clk, pclk);
    if (rc < 0)
        pr_err("clk: failed to reparent %s to %s: %d\n",
               __clk_get_name(clk), __clk_get_name(pclk), rc);
```

لو `clk_set_parent()` نجح (رجع 0)، مفيش error في الـ log. الـ parent اتضبط على VIDEO1 بنجاح من ناحية الـ mux، لكن لما `__set_clk_rates()` اشتغل وحاول يضبط 594MHz على VIDEO1، الـ PLL مش قادر يوصل للقيمة دي.

الـ kernel log بيطلع:

```
clk: couldn't set clk-hdmi clk rate to 594000000 (-22), current rate: 297000000
```

الـ `-22` ده EINVAL — الـ PLL VIDEO1 عنده max frequency 297MHz.

#### الحل
تصحيح الـ parent:

```dts
&de {
    assigned-clocks = <&ccu CLK_DE>, <&ccu CLK_HDMI>;
    assigned-clock-parents = <&ccu CLK_PLL_PERIPH0>, <&ccu CLK_PLL_VIDEO0>; /* fix */
    assigned-clock-rates = <300000000>, <594000000>;
};
```

للـ debug:

```bash
# شوف الـ clock tree كامل
cat /sys/kernel/debug/clk/clk_summary | grep -i "video\|hdmi"

# تحقق من الـ parent الحالي
cat /sys/kernel/debug/clk/clk-hdmi/clk_parent

# لو مش شايف clk_summary، جرب
ls /sys/kernel/debug/clk/
```

#### الدرس المستفاد
الـ `clk-conf.c` بيعمل `clk_set_parent()` الأول، ثم `clk_set_rate()` بعدين — الترتيب ده مقصود لأن الـ parent بيتحدد قبل ما تحاول تضبط الـ rate. لو الـ parent غلط، الـ rate هتفشل بصمت (الـ log بيقول "couldn't set rate" بس مش واضح ليه). لازم تتحقق من قدرات كل PLL قبل ما تختار الـ parent.

---

### السيناريو الثالث: IoT Sensor على STM32MP1 — probe_defer لا نهائي

#### العنوان
جهاز لا يكمل boot بسبب `-EPROBE_DEFER` دايم في `of_clk_set_defaults()`

#### السياق
**IoT sensor node** بيشتغل على **STM32MP1**. الجهاز بيجمع بيانات من SPI sensor وبيبعتها عبر LPUART. أثناء bring-up، الـ kernel بيقف في boot ولا بيكمل — systemd مش بتشتغل والـ console بيطلع بس:

```
[    2.341] platform spi0: deferred probe pending
[    4.341] platform spi0: deferred probe pending
...
```

#### المشكلة
الـ DT فيه:

```dts
&spi0 {
    assigned-clocks = <&rcc SPI0_K>;
    assigned-clock-parents = <&rcc PLL4_Q>;
    assigned-clock-rates = <50000000>;
    status = "okay";
};

&rcc {
    /* PLL4 config */
};
```

الـ RCC clock driver لسه مش اتسجّل وقت ما الـ spi0 بيحاول probe. يعني `of_clk_get_from_provider()` بترجع **`-EPROBE_DEFER`**.

#### التحليل
في `__set_clk_parents()`:

```c
pclk = of_clk_get_from_provider(&clkspec);
of_node_put(clkspec.np);
if (IS_ERR(pclk)) {
    if (PTR_ERR(pclk) != -EPROBE_DEFER)
        pr_warn("clk: couldn't get parent clock %d for %pOF\n",
                index, node);
    return PTR_ERR(pclk);  /* يرجع -EPROBE_DEFER للـ caller */
}
```

الكود بيرجع `-EPROBE_DEFER` بصمت (من غير pr_warn عشان هو check على -EPROBE_DEFER). الـ `of_clk_set_defaults()` بترجع الـ error ده لـ `clk_init_cb()` أو لـ bus infrastructure، اللي بيعمل re-queue للـ device probe.

المشكلة الأعمق: الـ RCC driver نفسه فيه مشكلة initialization — الـ probe بتاعته بيفشل بسبب missing regulator، ومفيش حد لاحظ لأن الـ error صغير في الـ log.

#### الحل
**خطوة 1** — تحديد سبب الـ defer الحقيقي:

```bash
# شوف كل الـ deferred probes
cat /sys/kernel/debug/devices_deferred

# شوف الـ rcc driver اشتغل ولا لأ
dmesg | grep -i "rcc\|stm32mp1-rcc"

# تحقق من الـ clock providers المسجلة
cat /sys/kernel/debug/clk/clk_orphan_summary
```

**خطوة 2** — الـ RCC driver فاشل بسبب vdd-core regulator مش موجود، ضيف الـ regulator في DT:

```dts
&vdd_core {
    regulator-min-microvolt = <1200000>;
    regulator-max-microvolt = <1350000>;
    regulator-always-on;
};
```

**خطوة 3** — تأكيد إن `of_clk_set_defaults()` بقت تشتغل:

```bash
dmesg | grep "clk:" | grep -v "EPROBE_DEFER"
```

#### الدرس المستفاد
الـ `clk-conf.c` بيتعامل مع `-EPROBE_DEFER` كـ expected case وبيرجعه بصمت — ده intentional design. المشكلة دايماً في الـ provider اللي مش register. لازم تبدأ الـ debug من الـ clock provider مش من الـ consumer.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — CAN Bus Timing خاطئ

#### العنوان
**assigned-clock-rates-u64** مطلوبة لـ CAN FD clock عالي الدقة على i.MX8QM

#### السياق
**Automotive ECU** على **i.MX8QM** محتاجة CAN FD تشتغل بـ clock بالظبط **80,000,000 Hz** (80 MHz) عشان تحقق الـ timing requirements لـ CAN FD بـ 8 Mbit/s data rate. الـ engineer لاقى إن الـ CAN timing calculations بتطلع خاطئة بفارق صغير بيسبب frame errors على الـ CAN bus في بعض الأوقات.

#### المشكلة
الكود القديم كان بيستخدم:

```dts
&can0 {
    assigned-clocks = <&clk IMX8QM_CAN0_CLK>;
    assigned-clock-rates = <80000000>;
};
```

الـ u32 بيعمل كويس هنا (80MHz < 4GHz)، بس لما الـ engineer حاول يرفع لـ **200,000,000 Hz** للـ next-gen hardware، الـ u32 كفاية لأن 200MHz أقل من 4.29GHz. بس في بعض الـ SoC الأحدث، الـ PLL frequencies بتتعدى 4GHz في الـ internal calculations.

#### التحليل
الكود في `__set_clk_rates()` بيدعم الـ u64 صراحةً:

```c
count_64 = of_property_count_u64_elems(node, "assigned-clock-rates-u64");
if (count_64 > 0) {
    count = count_64;
    rates_64 = kcalloc(count, sizeof(*rates_64), GFP_KERNEL);
    /* ... */
    rc = of_property_read_u64_array(node,
                                    "assigned-clock-rates-u64",
                                    rates_64, count);
} else if (count > 0) {
    rates = kcalloc(count, sizeof(*rates), GFP_KERNEL);
    /* ... */
    rc = of_property_read_u32_array(node, "assigned-clock-rates",
                                    rates, count);
}
```

الـ priority واضحة: لو في `assigned-clock-rates-u64`، يستخدمها وبيتجاهل `assigned-clock-rates`. ده بيخلي الـ migration سهلة.

للـ i.MX8QM المحتاج accuracy عالية في CAN FD timing:

```dts
&can0 {
    assigned-clocks = <&clk IMX8QM_CAN0_CLK>;
    /* استخدام u64 لضمان precision في الأجهزة المستقبلية */
    assigned-clock-rates-u64 = /bits/ 64 <80000000>;
};
```

للتحقق من الـ rate الفعلي والـ error margin:

```bash
# تحقق من الـ clock rate الفعلي
cat /sys/kernel/debug/clk/can0_clk/clk_rate

# احسب الـ error percentage
# لو المطلوب 80000000 والفعلي 79999872 مثلاً
python3 -c "print(abs(80000000-79999872)/80000000*100, '%')"

# شوف الـ CAN bit timing
ip -details link show can0
```

#### الدرس المستفاد
**`assigned-clock-rates-u64`** مش مجرد للقيم فوق 4GHz — ده property جديد بيوفر precision أفضل وبيوضح الـ intent في الـ DT. في الـ automotive context، حتى فرق 128Hz في الـ source clock ممكن يسبب CAN timing violations. الـ `clk-conf.c` بيعطي الـ priority للـ u64 property لو موجودة.

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — I2C لا تشتغل بعد Power Management

#### العنوان
`clk_supplier = false` بيخلي I2C clock ما بتتضبطش على بورد جديدة AM62x

#### السياق
فريق bring-up بيشتغل على بورد custom مبنية على **TI AM62x**. الـ I2C0 بتتواصل مع PMIC. بعد إضافة **power management** وعمل `pm_runtime_enable()` في الـ I2C driver، الـ I2C بدأت تفشل بعد أول suspend/resume cycle. الـ clock بتتضبط صح في الأول، بس بعد resume مش بتتضبط.

#### المشكلة
الـ engineer لاقى في الـ I2C driver إن `of_clk_set_defaults()` بيتكلم مرتين: مرة في `probe()` ومرة في `resume()`. في الـ resume، الـ flag `clk_supplier` بيتبعت كـ `true` بدل `false`.

الـ DT:

```dts
&i2c0 {
    assigned-clocks = <&k3_clks 41 0>;
    assigned-clock-parents = <&k3_clks 41 2>;
    assigned-clock-rates = <400000>;
    status = "okay";

    pmic@48 {
        compatible = "ti,tps65219";
        reg = <0x48>;
    };
};
```

#### التحليل
في `__set_clk_parents()`:

```c
if (clkspec.np == node && !clk_supplier) {
    of_node_put(clkspec.np);
    return 0;  /* يخرج فوراً بدون ضبط أي حاجة */
}
```

وفي `__set_clk_rates()`:

```c
if (clkspec.np == node && !clk_supplier) {
    of_node_put(clkspec.np);
    return 0;  /* نفس السلوك */
}
```

الـ logic دي موجودة عشان لو الـ node نفسه هو supplier للـ clock، لازم `clk_supplier = true` عشان يسمح بالضبط. الـ I2C node على AM62x بيحتوي على clock index بيبص على `k3_clks` مش على نفسه، يعني `clkspec.np != node` وبالتالي الـ check ده مش هو المشكلة هنا.

المشكلة الفعلية: في `resume()`, الـ driver بيعمل:

```c
/* في I2C driver resume */
of_clk_set_defaults(dev->of_node, true);  /* خطأ! clk_supplier=true */
```

لما `clk_supplier = true`، الـ function بتكمل حتى لو `clkspec.np == node`. لو الـ clock provider مش جاهز بعد resume، `of_clk_get_from_provider()` بتفشل.

الصح في الـ probe:

```c
/* of_clk_set_defaults() بتتكلم من bus infrastructure بـ clk_supplier=false */
of_clk_set_defaults(np, false);
```

#### الحل
**خطوة 1** — تحديد أين بيتكلم `of_clk_set_defaults()` في الـ resume:

```bash
# شوف الـ kernel log أثناء resume
dmesg | grep -i "clk:\|i2c" | tail -50

# فحص الـ clock state بعد resume
cat /sys/kernel/debug/clk/i2c0_fclk/clk_enable_count
```

**خطوة 2** — في الـ driver، حذف استدعاء `of_clk_set_defaults()` من `resume()` لأن الـ bus framework بيعملها تلقائياً أثناء `probe()`. أو لو محتاجه في resume، يستخدم `clk_supplier = false`:

```c
static int am62x_i2c_resume(struct device *dev)
{
    /* الـ bus framework بيضبط الـ clocks في probe - مش محتاج تعيد هنا */
    /* لو لازم، استخدم false مش true */
    // of_clk_set_defaults(dev->of_node, false);  /* لو ضرورة */
    return am62x_i2c_init(dev);
}
```

**خطوة 3** — تأكيد:

```bash
# suspend ثم resume وشوف الـ I2C
echo mem > /sys/power/state
# بعد الاستيقاظ:
i2cdetect -y 0
dmesg | grep -i "clk:\|i2c" | grep -v "EPROBE"
```

#### الدرس المستفاد
الـ `clk_supplier` flag في `of_clk_set_defaults()` مش مجرد تفصيلة — ده بيتحكم في سلوك الـ function كاملاً. الـ bus framework (مثل platform bus) بيبعته بـ `false` في الـ probe عشان يمنع race conditions مع الـ clock providers. لو بتكلم `of_clk_set_defaults()` يدوياً في الـ resume، لازم تفهم الـ semantics دي وتختار الـ flag الصح.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتشرح الـ Common Clock Framework من الأساس لحد التفاصيل:

| المقال | الأهمية |
|--------|---------|
| [A common clock framework](https://lwn.net/Articles/472998/) | المقال الأصلي اللي شرح الـ CCF لأول مرة — لازم تقراه |
| [common clk framework (patch discussion)](https://lwn.net/Articles/486841/) | نقاش تقني على الـ patches الأولى للـ framework |
| [Add a generic struct clk](https://lwn.net/Articles/460193/) | المقترح الأولي لتوحيد الـ `struct clk` قبل ما الـ CCF يتبنى |
| [Create common DPLL/clock configuration API](https://lwn.net/Articles/920248/) | تطوير أحدث — DPLL API فوق الـ CCF |
| [driver/clk/clk-si5338](https://lwn.net/Articles/698134/) | مثال واقعي لـ driver بيستخدم الـ CCF |

---

### التوثيق الرسمي للـ Kernel

**الـ** `Documentation/` **paths** المباشرة:

```
Documentation/driver-api/clk.rst          — الـ CCF API الكاملة
Documentation/devicetree/bindings/clock/  — bindings الـ DT للكلوك
include/linux/clk.h                       — consumer API
include/linux/clk-provider.h              — provider API
drivers/clk/clk.c                         — core implementation
drivers/clk/clk-conf.c                    — الملف اللي بندرسه
```

الـ online version:
- [The Common Clk Framework — kernel.org docs](https://docs.kernel.org/driver-api/clk.html)
- [The Common Clk Framework — static.lwn.net](https://static.lwn.net/kerneldoc/driver-api/clk.html)
- [Common Clock Framework — kernel.org (clk.txt)](https://www.kernel.org/doc/Documentation/clk.txt)

---

### Kernel Commits المهمة

**الـ commit** اللي أدخل `clk-conf.c` و`of_clk_set_defaults()` و`assigned-clocks` support:

- **Commit**: `86be408bfbd8` — *"clk: Support for clock parents and rates assigned from device tree"*
  - الـ author: Sylwester Nawrocki (Samsung)
  - الـ patch على Patchwork: [PATCH/RFC,V8,1/1](https://patchwork.kernel.org/patch/4377571/)
  - نقاش الـ patch التفصيلي: [patchwork.kernel.org](https://patchwork.kernel.org/project/spi-devel-general/patch/1403105372-30403-2-git-send-email-s.nawrocki@samsung.com/)

**الـ patch** بيوضح إن الـ helpers دي بتتاكل من:
1. الـ bus code (platform, I2C, SPI) — قبل الـ driver probing
2. الـ clock core — بعد تسجيل الـ clock provider

**أهم commit للـ CCF الأصلي**:
```bash
# تقدر تشوف تاريخ الـ CCF من هنا
git log --oneline drivers/clk/clk.c | tail -20
git log --oneline drivers/clk/clk-conf.c
```

---

### نقاشات الـ Mailing List

- **LKML patch v7**: [clk: introduce the common clock framework](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00026.html) — النقاش الأصلي لإدخال الـ CCF
- **Device Tree spec**: [Add assigned-clocks description](https://www.spinics.net/lists/devicetree-spec/msg01040.html) — توثيق الـ DT bindings
- **Patchwork**: [assigned-clock-parents support](https://patchwork.kernel.org/patch/4377571/) — الـ patch اللي أدخل `clk-conf.c`

---

### كتب مُوصى بيها

#### Linux Device Drivers 3rd Edition (LDD3)
- **Chapter 14**: The Linux Device Model — فاهم إزاي الـ device tree بيشتغل مع الـ drivers
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)
- **Chapter 17**: Devices and Modules — الـ bus/device/driver model
- الـ CCF بيبني فوق نفس الـ model ده

#### Embedded Linux Primer — Christopher Hallinan
- **Chapter 15**: Debugging the Linux Kernel — بيتكلم عن الـ clock debugging
- **Chapter 7**: Bootloader Fundamentals — clock init في الـ early boot

#### Professional Linux Kernel Architecture — Wolfgang Mauerer
- فصل الـ device drivers — شرح عميق للـ subsystem model

---

### مصادر eLinux.org

- [Generic Linux CLK Framework - CELF 2009 (PDF)](https://elinux.org/images/6/64/ELC_E_2009_Generic_Clock_Framework.pdf) — presentation قديم لكنه مفيد لفهم الـ motivation
- [Linux clock management framework (PDF)](https://elinux.org/images/a/a3/ELC_2007_Linux_clock_fmw.pdf) — الجذور التاريخية
- [Common Clock Framework BoFs (PDF)](https://elinux.org/images/b/b1/Common_Clock_Framework_(BoFs).pdf) — Birds of a Feather session

---

### مصادر kernelnewbies.org

الـ CCF بيتذكر في changelogs كتير:
- [Linux 6.7 Changelog](https://kernelnewbies.org/Linux_6.7) — آخر تحديثات الـ CCF
- [Linux 5.9 Changelog](https://kernelnewbies.org/Linux_5.9) — تغييرات مهمة في الـ clock subsystem
- [Linux 6.12 Changelog](https://kernelnewbies.org/Linux_6.12)
- [Linux 6.13 Changelog](https://kernelnewbies.org/Linux_6.13)

---

### Bootlin (Free Electrons) Presentations

من أحسن المصادر العملية:
- [Common Clock Framework: How To Use It — ELCE 2013 (PDF)](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf)
- [Common Clock Framework — ELC 2013 (PDF)](https://bootlin.com/pub/conferences/2013/elc/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf)
- شرح Gregory Clement بتفصيل إزاي تكتب clock driver وإزاي الـ DT bindings بتشتغل

---

### مصادر تانية مفيدة

- **STM32MPU Wiki**: [Clock overview](https://wiki.st.com/stm32mpu/wiki/Clock_overview) — مثال واقعي لـ SoC بيستخدم الـ CCF
- **Linux KDB**: [CONFIG_COMMON_CLK](https://cateee.net/lkddb/web-lkddb/COMMON_CLK.html) — الـ Kconfig entry
- **mindlinux blog**: [Common Clock Framework: How To Use It](https://mindlinux.wordpress.com/2013/10/24/common-clock-framework-how-to-use-it-gregory-clement-free-electrons/) — ملخص الـ presentation

---

### Search Terms للبحث أكتر

لو حابب تعمق أكتر، استخدم الـ search terms دي:

```
linux kernel CCF common clock framework
of_clk_set_defaults assigned-clocks device tree
clk_set_parent clk_set_rate kernel
linux clock provider consumer API
CONFIG_COMMON_CLK kernel
assigned-clock-parents DT binding
struct clk_ops linux kernel
clk-conf.c samsung nawrocki
EPROBE_DEFER clock framework
of_clk_get_from_provider kernel
```

---

### ملخص المصادر الأساسية

```
أهم 3 حاجات تبدأ بيهم:
1. https://lwn.net/Articles/472998/          ← CCF overview
2. https://docs.kernel.org/driver-api/clk.html ← official docs
3. https://patchwork.kernel.org/patch/4377571/ ← الـ patch الأصلي لـ clk-conf.c
```
## Phase 8: Writing simple module

### الفكرة

الـ `of_clk_set_defaults()` هي الدالة الوحيدة المـ exported في الملف — بتقرأ الـ device tree وتضبط الـ clock parents والـ rates. هنستخدم **kprobe** عشان نخطف كل مرة بيتنادى عليها ونطبع اسم الـ node اللي بيتضبط وهو supplier ولا لأ.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on of_clk_set_defaults()
 * Prints the device-tree node name and clk_supplier flag
 * every time clock defaults are applied from DT.
 */

/* --- Includes --- */
#include <linux/module.h>       /* module_init/exit, MODULE_* macros        */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ...      */
#include <linux/of.h>           /* struct device_node, of_node_full_name()  */
#include <linux/printk.h>       /* pr_info / pr_err                         */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe demo: trace of_clk_set_defaults() calls");

/* ---------------------------------------------------------------
 * pre_handler
 * Called just BEFORE of_clk_set_defaults() executes.
 *
 * Prototype of the hooked function:
 *   int of_clk_set_defaults(struct device_node *node, bool clk_supplier);
 *
 * On x86-64 the arguments live in:
 *   regs->di  → node          (1st arg)
 *   regs->si  → clk_supplier  (2nd arg)
 * --------------------------------------------------------------- */
static int clk_conf_pre_handler(struct kprobe *p, struct pt_regs *regs)
{
    /* Cast register values back to the real argument types */
    struct device_node *node =
        (struct device_node *)regs->di;          /* 1st arg */
    bool clk_supplier = (bool)regs->si;          /* 2nd arg */

    if (node)
        pr_info("clk-conf kprobe: of_clk_set_defaults() called\n"
                "  node      = %s\n"
                "  supplier  = %s\n",
                node->full_name ? node->full_name : "(null)",
                clk_supplier ? "true" : "false");
    else
        pr_info("clk-conf kprobe: of_clk_set_defaults(NULL)\n");

    return 0; /* 0 = continue normal execution */
}

/* ---------------------------------------------------------------
 * kprobe descriptor
 * symbol_name → kernel resolves the address at registration time.
 * --------------------------------------------------------------- */
static struct kprobe clk_conf_kp = {
    .symbol_name  = "of_clk_set_defaults",
    .pre_handler  = clk_conf_pre_handler,
};

/* ---------------------------------------------------------------
 * module_init
 * --------------------------------------------------------------- */
static int __init clk_conf_probe_init(void)
{
    int ret;

    ret = register_kprobe(&clk_conf_kp);
    if (ret < 0) {
        pr_err("clk-conf kprobe: register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("clk-conf kprobe: planted at %s (%px)\n",
            clk_conf_kp.symbol_name,
            clk_conf_kp.addr);       /* addr filled in by the kernel */
    return 0;
}

/* ---------------------------------------------------------------
 * module_exit
 * --------------------------------------------------------------- */
static void __exit clk_conf_probe_exit(void)
{
    unregister_kprobe(&clk_conf_kp);
    pr_info("clk-conf kprobe: removed\n");
}

module_init(clk_conf_probe_init);
module_exit(clk_conf_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

| Header | ليه موجود |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل دوال التسجيل |
| `linux/of.h` | تعريف `struct device_node` عشان نعمل cast صح للـ pointer |
| `linux/printk.h` | `pr_info` / `pr_err` |

---

#### الـ `pre_handler`

الـ handler بيتنادى عليه **قبل** ما الدالة الأصلية تتنفذ، فبنقدر نشوف الـ arguments بدون ما نأثر على السلوك.
الـ `regs->di` و`regs->si` هما الـ registers اللي بتحمل أول وتاني argument على x86-64 حسب الـ System V ABI — لازم نعمل cast يدوي لأن الـ kprobe بيعطينا `pt_regs` خام.

---

#### الـ `struct kprobe`

الـ `symbol_name` بيخلي الـ kernel يحل عنوان `of_clk_set_defaults` وقت التسجيل بنفسه بدل ما نحسبه يدوي.
بعد `register_kprobe()` الـ `addr` field بيتملى بالعنوان الفعلي في الـ kernel image.

---

#### الـ `module_init`

`register_kprobe()` بتحجز الـ breakpoint في الـ kernel text — لو فشل (مثلاً الدالة مش exported أو الـ kprobes معطل في الـ config) بنرجع الـ error فوراً بدل ما يتحمل الـ module بـ hook معطوب.

---

#### الـ `module_exit`

**لازم** ننادي `unregister_kprobe()` في الـ exit — لو ما عملناش كده والـ module اتـ unload، الـ breakpoint هيفضل في الـ kernel text وأول ما يتنادى على `of_clk_set_defaults` هيـ crash النظام لأن الـ handler address بقى invalid.

---

### Makefile للتجربة

```makefile
obj-m += clk_conf_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وتتبع النتيجة

```bash
# تحميل الـ module
sudo insmod clk_conf_probe.ko

# مشاهدة الـ log (لو في driver بيتحمل وبيستدعي of_clk_set_defaults)
sudo dmesg | grep "clk-conf kprobe"

# تفريغ الـ module
sudo rmmod clk_conf_probe
```

مثال على الـ output المتوقع:

```
clk-conf kprobe: planted at of_clk_set_defaults (ffffffffc012ab40)
clk-conf kprobe: of_clk_set_defaults() called
  node      = /soc/clock-controller@1c20000
  supplier  = true
clk-conf kprobe: removed
```
