## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem اللي ينتمي له الملف

**الـ `clk-bulk.c`** جزء من **Common CLK Framework** — الإطار المشترك لإدارة الـ clocks في Linux kernel. الـ maintainers هم Michael Turquette وStephen Boyd، والكود كله تحت `drivers/clk/`.

---

### الـ Clock في الـ Hardware — إيه ده أصلاً؟

تخيل إن كل جهاز إلكتروني جوه الـ SoC (زي USB controller، أو GPU، أو memory controller) محتاج **نبضة إيقاع ثابتة** عشان يشتغل — زي القلب بالظبط. الـ **clock** هو اللي بيدي الأجهزة دي الإيقاع ده.

في الـ SoC الواحد ممكن يكون في عشرات الـ clocks المختلفة — كل جهاز ليه clock (أو أكتر) خاص بيه. لما driver بيشتغل، لازم يـ:

1. **يجيب** الـ clock من النظام (`clk_get`)
2. **يحضره** (`clk_prepare`) — عملية ممكن تاخد وقت وبتنام
3. **يشغله** (`clk_enable`) — عملية سريعة مش بتنام
4. لما يخلص: **يطفيه** (`clk_disable`) ← ثم **يلغي تحضيره** (`clk_unprepare`) ← ثم **يرجعه** (`clk_put`)

---

### المشكلة اللي `clk-bulk.c` جاي يحلها

#### القصة:

تخيل بتكتب driver لـ display controller في موبايل. الـ display controller ده محتاج **5 clocks** عشان يشتغل:
- `pixel_clk` — للبيكسلات
- `axi_clk` — للـ bus
- `ahb_clk` — للـ config
- `dsi_clk` — للشاشة
- `hdmi_clk` — للـ HDMI

**بدون bulk:**

```c
/* كل clock لوحده — كود متكرر ومرهق */
pixel = clk_get(dev, "pixel");
if (IS_ERR(pixel)) { ret = PTR_ERR(pixel); goto err1; }

axi = clk_get(dev, "axi");
if (IS_ERR(axi)) { ret = PTR_ERR(axi); clk_put(pixel); goto err2; }

ahb = clk_get(dev, "ahb");
if (IS_ERR(ahb)) { ret = PTR_ERR(ahb); clk_put(axi); clk_put(pixel); goto err3; }

/* ... وهكذا لـ 5 clocks */
```

كل error path لازم تـ cleanup كل حاجة عملتها قبل كده — الكود بيبقى طويل، معرض للـ bugs، وكل driver بيعيد كتابة نفس المنطق.

**مع bulk:**

```c
/* كل الـ 5 clocks في سطر واحد */
static struct clk_bulk_data clks[] = {
    { .id = "pixel" },
    { .id = "axi"   },
    { .id = "ahb"   },
    { .id = "dsi"   },
    { .id = "hdmi"  },
};

ret = clk_bulk_get(dev, ARRAY_SIZE(clks), clks);
/* لو فشل أي clock، النظام بيـ cleanup الباقين تلقائياً */
```

هو ده الهدف: **API مريح يتعامل مع مجموعة clocks كوحدة واحدة** بدل التعامل مع كل واحد لوحده.

---

### إيه اللي بيعمله `clk-bulk.c` بالظبط؟

الملف ده بيوفر **طبقة convenience** فوق الـ single-clock API. كل Function فيه بتاخد array من الـ clocks وتعمل نفس العملية على كلهم:

| Function | العملية |
|---|---|
| `clk_bulk_get` | بيجيب مجموعة clocks بالاسم من الـ device |
| `clk_bulk_get_optional` | زي الأول بس مش بيعدّ غياب الـ clock كـ error |
| `clk_bulk_get_all` | بيجيب كل الـ clocks المذكورة في الـ device tree تلقائياً |
| `clk_bulk_put` | بيرجع الـ clocks (عكس get) |
| `clk_bulk_put_all` | بيرجع الـ clocks ويفرر الـ memory |
| `clk_bulk_prepare` | بيحضر كل الـ clocks (ممكن تنام) |
| `clk_bulk_unprepare` | عكس prepare |
| `clk_bulk_enable` | بيشغل كل الـ clocks (لا تنام) |
| `clk_bulk_disable` | بيطفي كل الـ clocks |

**نقطة مهمة في التصميم:** لو أي clock فشل أثناء الـ `get` أو الـ `enable` أو الـ `prepare`، الدالة بتعمل **rollback تلقائي** للي عملته قبل كده — مش محتاج تعمل cleanup يدوي.

---

### الـ Device Tree وربط الـ Clocks

الـ `clk-bulk.c` بيقرأ الـ clock names من الـ Device Tree. مثال:

```
/* في الـ device tree */
display_controller: display@12000000 {
    clocks = <&pixel_pll>, <&axi_clk>, <&ahb_clk>;
    clock-names = "pixel", "axi", "ahb";
};
```

الدالة `of_clk_bulk_get` بتقرأ `clock-names` وتجيب كل clock بالترتيب. `clk_bulk_get_all` بتجيب كلهم تلقائياً من غير ما تحدد العدد.

---

### الـ Two-Step Enable: ليه prepare ثم enable؟

الـ Linux CLK framework بيفصل بين خطوتين:

- **`prepare`**: ممكن تاخد وقت (بتنام، بتعمل PLL lock، ...) → تتعمل خارج الـ atomic context
- **`enable`**: لازم تكون سريعة جداً، مش بتنام → ممكن تتعمل في الـ atomic context (interrupt handler مثلاً)

```
clk_bulk_prepare(num, clks)   ← ممكن تنام، جهز الـ PLL
clk_bulk_enable(num, clks)    ← سريعة، فتح الـ gate

... الجهاز شغال ...

clk_bulk_disable(num, clks)   ← سريعة، سكر الـ gate
clk_bulk_unprepare(num, clks) ← ممكن تنام، ايقف الـ PLL
```

أو اختصاراً في الحالات العادية: `clk_bulk_prepare_enable` / `clk_bulk_disable_unprepare` (مشرحين في `clk.h`).

---

### الملفات المرتبطة اللي المفروض تعرفها

**الـ Core:**

| الملف | الدور |
|---|---|
| `drivers/clk/clk.c` | قلب الـ framework — كل الـ core logic للـ single clock |
| `drivers/clk/clk-bulk.c` | **الملف ده** — bulk operations |
| `drivers/clk/clk-devres.c` | نسخة `devm_` من الـ single clock APIs (auto-cleanup مع device lifetime) |

**الـ Headers:**

| الملف | الدور |
|---|---|
| `include/linux/clk.h` | الـ public API للـ clock consumers — فيه تعريف `struct clk_bulk_data` |
| `include/linux/clk-provider.h` | الـ API للي بيكتب clock drivers (providers مش consumers) |

**الـ Hardware Drivers (أمثلة):**

| الملف | الدور |
|---|---|
| `drivers/clk/clk-fixed-rate.c` | أبسط clock — تردد ثابت |
| `drivers/clk/clk-divider.c` | clock بيقسم تردد clock تاني |
| `drivers/clk/clk-mux.c` | clock بيختار من مصادر متعددة |
| `drivers/clk/clk-gate.c` | clock gate — بس يشغل/يطفي |
| `drivers/clk/imx/`, `drivers/clk/qcom/`, إلخ | drivers لـ SoCs محددة |

**الـ Device Tree Bindings:**

| الملف | الدور |
|---|---|
| `Documentation/devicetree/bindings/clock/` | وصف كيفية ربط الـ clocks في الـ device tree |
## Phase 2: شرح الـ CLK Bulk Framework

---

### المشكلة — ليه الـ subsystem ده موجود أصلاً؟

أي driver في الـ SoC بيحتاج يشغّل hardware block زي USB, UART, SPI، هيحتاج مش clock واحد — هيحتاج **عدة clocks في نفس الوقت**. مثلاً:

- `ahb_clk` — الـ AHB bus clock اللي بيخلي الـ peripheral يتكلم مع الـ CPU
- `per_clk` — الـ peripheral functional clock اللي بيشغّل اللوجيك الداخلي
- `phb_clk` — الـ PHY clock لو كان فيه USB PHY

الكود التقليدي من غير bulk بييجي كده:

```c
/* بدون bulk — كود مكرر وممل وعرضة للـ bugs */
dev->clk_ahb = clk_get(dev, "ahb");
if (IS_ERR(dev->clk_ahb)) { ret = PTR_ERR(dev->clk_ahb); goto err1; }

dev->clk_per = clk_get(dev, "per");
if (IS_ERR(dev->clk_per)) { ret = PTR_ERR(dev->clk_per); goto err2; }

dev->clk_phb = clk_get(dev, "phb");
if (IS_ERR(dev->clk_phb)) { ret = PTR_ERR(dev->clk_phb); goto err3; }

/* وبعدين تعمل clk_put لكل واحد لو حاجة فشلت */
err3: clk_put(dev->clk_per);
err2: clk_put(dev->clk_ahb);
err1: return ret;
```

المشاكل هنا:
- **Boilerplate** كتير — نفس الـ pattern بيتكرر في كل driver
- **Error handling** معقد — لازم تـ`put` كل clock اتعمله `get` بالترتيب العكسي
- **prepare/enable** لازم تعملها واحدة واحدة
- نسيان أي step = **resource leak** أو kernel panic

---

### الحل — النهج اللي اتخدم

الـ **CLK Bulk API** بيقدم abstraction بسيطة: بدل ما تتعامل مع كل clock على حدة، بتعمل **array من الـ `clk_bulk_data`** وبتمرره لـ API واحدة تعمل العملية دي على الكل.

```c
/* مع bulk — نظيف ومضمون */
static struct clk_bulk_data clks[] = {
    { .id = "ahb" },
    { .id = "per" },
    { .id = "phb" },
};

ret = clk_bulk_get(dev, ARRAY_SIZE(clks), clks);
if (ret) return ret;

ret = clk_bulk_prepare_enable(ARRAY_SIZE(clks), clks);
if (ret) goto err;
```

الـ framework بيضمن: لو `clk_get` نجحت لـ 2 من 3 clocks وفشلت في الـ 3، هو بنفسه بيعمل `put` للاتنين اللي نجحوا — **atomically من وجهة نظر الـ driver**.

---

### الـ Real-World Analogy — قاطع كهرباء المطبخ

تخيل إنك عامل upgrade للمطبخ وعندك:
- **الفرن** يحتاج 3 circuit breakers: واحد للـ heating element، واحد للـ timer، واحد للـ fan
- لو أي breaker مش شغال، الفرن مش هيشتغل خالص

**النهج القديم**: بتروح للـ panel، تشغّل كل breaker لوحده، وبتكتب ورقة تفكّرك ترجعهم كلهم لو حاجة غلط.

**الـ Bulk API**: زي **master switch للمطبخ كله** — بتدور بمفتاح واحد، ولو أي breaker فيه مشكلة، المفتاح ده بيرجّع كل حاجة لحالتها الأصلية تلقائياً.

الـ mapping على الـ kernel:
| الـ Analogy | الـ Kernel Concept |
|---|---|
| الـ breakers الـ 3 | الـ `clk_bulk_data[]` array |
| اسم كل breaker على الـ panel | الـ `.id` string في `clk_bulk_data` |
| الـ master switch | `clk_bulk_get()` / `clk_bulk_enable()` |
| الـ wiring diagram خلف الـ panel | الـ Device Tree `clock-names` property |
| الـ electrician | الـ CLK framework core (`drivers/clk/clk.c`) |
| لو breaker ما اتوجدش | `clk_bulk_get_optional()` — مش error، يكمّل |

---

### الـ Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Driver Layer                             │
│                                                                 │
│   ┌─────────────┐   ┌──────────────┐   ┌──────────────────┐   │
│   │  USB Driver │   │  SPI Driver  │   │   UART Driver    │   │
│   │             │   │              │   │                  │   │
│   │clk_bulk_get │   │clk_bulk_get  │   │clk_bulk_get_all  │   │
│   │clk_bulk_ena │   │clk_bulk_ena  │   │clk_bulk_enable   │   │
│   └──────┬──────┘   └──────┬───────┘   └────────┬─────────┘   │
│          │                 │                     │             │
└──────────┼─────────────────┼─────────────────────┼────────────-┘
           │                 │                     │
           ▼                 ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                   CLK Bulk API Layer                            │
│              drivers/clk/clk-bulk.c                             │
│                                                                 │
│   clk_bulk_get()          clk_bulk_get_optional()              │
│   clk_bulk_get_all()      clk_bulk_put()                       │
│   clk_bulk_prepare()      clk_bulk_unprepare()                 │
│   clk_bulk_enable()       clk_bulk_disable()                   │
│                                                                 │
│   [iterates over clk_bulk_data[], calls single-clk APIs]       │
└────────────────────────────┬────────────────────────────────────┘
                             │ calls
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                   CLK Core API Layer                            │
│              include/linux/clk.h + drivers/clk/clk.c           │
│                                                                 │
│   clk_get()     clk_put()      clk_prepare()   clk_enable()   │
│   of_clk_get()  clk_unprepare() clk_disable()                  │
│                                                                 │
│   [manages clk_core tree, reference counting, topology]        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Clock Provider / Hardware Layer                    │
│                                                                 │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────────────────┐  │
│  │  PLL Driver  │  │  Gate Driver│  │  Divider Driver      │  │
│  │(clk-pll.c)   │  │(clk-gate.c) │  │(clk-divider.c)       │  │
│  └──────────────┘  └─────────────┘  └──────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Device Tree / ACPI                          │  │
│  │  clocks = <&pll1>, <&gate2>;                             │  │
│  │  clock-names = "ahb", "per";                             │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

### الـ Core Abstraction — الفكرة المحورية

الـ `clk_bulk_data` هي الـ **unit of abstraction** الأساسية:

```c
/* من include/linux/clk.h */
struct clk_bulk_data {
    const char  *id;   /* اسم الـ clock زي "ahb", "per" — بيتطابق مع clock-names في DT */
    struct clk  *clk;  /* pointer للـ clock object — الـ framework بيملّيها */
};
```

الـ driver بيملّي بس الـ `.id`، والـ framework بيملّي الـ `.clk` من الـ Device Tree.

```
clk_bulk_data[] array:
┌──────────────────────────────────────────┐
│ [0] .id  = "ahb"   →  .clk = <clk_ptr>  │
│ [1] .id  = "per"   →  .clk = <clk_ptr>  │
│ [2] .id  = "phb"   →  .clk = <clk_ptr>  │
└──────────────────────────────────────────┘
        ↑ Driver fills this    ↑ Framework fills this
```

---

### الـ Two-Phase Clock Model — prepare vs enable

قبل ما تفهم الـ bulk API، لازم تفهم إن الـ CLK framework (مش الـ bulk تحديداً) بيقسّم تشغيل الـ clock لـ **مرحلتين**:

> **الـ regulator subsystem**: نظام kernel آخر بيتحكم في الـ voltage rails — الـ clock والـ regulator بيشتغلوا جنب بعض في الـ power management.

**المرحلة الأولى — `prepare`**: ممكن تنام (sleepable context)
- بتعمل أي setup محتاج لوقت زي تشغيل PLL وانتظار lock
- بتزوّد الـ reference count على الـ `clk_core->prepare_count`
- **لازم** تتعمل قبل `enable` دايماً

**المرحلة التانية — `enable`**: لازم تكون atomic (non-sleepable)
- بس بتعمل hardware register write لفتح الـ gate
- بتزوّد الـ `clk_core->enable_count`
- **ممكن** تتعمل من interrupt context

```
Timeline:
                 prepare()          enable()          disable()      unprepare()
Driver call:        ↓                  ↓                  ↓               ↓
HW state:    [PLL warming up] → [clock gated open] → [gated closed] → [PLL off]
Context:        can sleep           atomic             atomic           can sleep
```

الـ bulk API بيعمل نفس الحاجة دي على array كاملة:

```c
/* prepare الكل، لو أي واحد فشل يـ unprepare اللي اتعملوا */
int clk_bulk_prepare(int num_clks, const struct clk_bulk_data *clks)
{
    for (i = 0; i < num_clks; i++) {
        ret = clk_prepare(clks[i].clk); /* single-clk API */
        if (ret) goto err;              /* rollback */
    }
err:
    clk_bulk_unprepare(i, clks);       /* undo i clocks فقط اللي نجحوا */
    return ret;
}
```

---

### الـ Error Handling Pattern — الـ Rollback الذكي

الـ bulk API بيستخدم pattern موحد في كل function: **forward loop + backward rollback**.

```c
/* clk_bulk_enable من clk-bulk.c */
int __must_check clk_bulk_enable(int num_clks, const struct clk_bulk_data *clks)
{
    int ret, i;

    for (i = 0; i < num_clks; i++) {
        ret = clk_enable(clks[i].clk); /* forward: enable one by one */
        if (ret) {
            pr_err("Failed to enable clk '%s': %d\n", clks[i].id, ret);
            goto err;
        }
    }
    return 0;

err:
    clk_bulk_disable(i, clks); /* backward: disable only those that succeeded */
    return ret;
}

/* clk_bulk_disable بيعمل reverse loop */
void clk_bulk_disable(int num_clks, const struct clk_bulk_data *clks)
{
    while (--num_clks >= 0)        /* من آخر واحد للأول */
        clk_disable(clks[num_clks].clk);
}
```

الـ reverse order في الـ disable مهم لأن الـ clock tree له **dependencies** — الـ child clock لازم يتوقف قبل الـ parent.

---

### الـ Device Tree Integration — من `clock-names` للـ `clk_bulk_data`

الـ `of_clk_bulk_get` بيربط اسم الـ clock في الـ DT بالـ `clk_bulk_data`:

```
Device Tree:
┌──────────────────────────────────┐
│ usb: usb@40080000 {              │
│   clocks = <&pll1 0>,            │  ← phandle للـ clock provider
│             <&gate2 5>;          │
│   clock-names = "ahb", "per";   │  ← الأسماء اللي بيدوّر عليها الـ driver
│ };                               │
└──────────────────────────────────┘
                 ↓
         of_clk_bulk_get()
         ↓                ↓
of_property_read_string_index()   of_clk_get()
     ↓                                 ↓
clks[i].id = "ahb"           clks[i].clk = <resolved clk ptr>
```

```c
static int of_clk_bulk_get(struct device_node *np, int num_clks,
                            struct clk_bulk_data *clks)
{
    for (i = 0; i < num_clks; i++) {
        /* بياخد الاسم من DT property "clock-names" */
        of_property_read_string_index(np, "clock-names", i, &clks[i].id);
        /* بيحوّل الـ phandle لـ struct clk pointer حقيقي */
        clks[i].clk = of_clk_get(np, i);
        if (IS_ERR(clks[i].clk)) {
            goto err; /* rollback كل اللي اتعمل */
        }
    }
}
```

---

### الـ `clk_bulk_get_all` — لما مش عارف عدد الـ clocks

لو الـ driver مش عارف عدد الـ clocks في الـ DT (generic driver):

```c
/* بيجيب عدد الـ clocks من DT أوتوماتيك */
int clk_bulk_get_all(struct device *dev, struct clk_bulk_data **clks)
{
    struct device_node *np = dev_of_node(dev); /* الـ DT node بتاع الـ device */

    if (!np) return 0;

    return of_clk_bulk_get_all(np, clks); /* بيـ allocate وبيملّي */
}

static int of_clk_bulk_get_all(struct device_node *np,
                                struct clk_bulk_data **clks)
{
    num_clks = of_clk_get_parent_count(np); /* عدد الـ clocks في الـ DT */
    clk_bulk = kmalloc_array(num_clks, sizeof(*clk_bulk), GFP_KERNEL); /* dynamic alloc */
    ret = of_clk_bulk_get(np, num_clks, clk_bulk);
    *clks = clk_bulk; /* رجّع الـ array للـ caller */
    return num_clks;
}
```

ده بيُستخدم مع `clk_bulk_put_all` اللي بتـ`kfree` الـ array كمان:

```c
void clk_bulk_put_all(int num_clks, struct clk_bulk_data *clks)
{
    if (IS_ERR_OR_NULL(clks)) return;
    clk_bulk_put(num_clks, clks);
    kfree(clks); /* مختلف عن clk_bulk_put العادي — بيـ free الـ array نفسه */
}
```

---

### الـ Optional Clocks — `clk_bulk_get_optional`

بعض الـ clocks ممكن تكون optional — موجودة على بعض الـ SoCs وغايبة على تانية:

```c
static int __clk_bulk_get(struct device *dev, int num_clks,
                           struct clk_bulk_data *clks, bool optional)
{
    for (i = 0; i < num_clks; i++) {
        clks[i].clk = clk_get(dev, clks[i].id);
        if (IS_ERR(clks[i].clk)) {
            ret = PTR_ERR(clks[i].clk);
            clks[i].clk = NULL;

            if (ret == -ENOENT && optional)
                continue; /* مش موجود؟ مشكلش، كمّل */

            goto err; /* لو مش optional، fail بجد */
        }
    }
}
```

الـ `optional` flag معناه: لو الـ clock مش معرّف في الـ DT (`-ENOENT`)، بنمشي ونسيبه `NULL` — الـ driver بعدين يـ check لو `clks[i].clk == NULL`.

---

### ما الـ Framework بيملكه vs ما بيفوّضه للـ Drivers

| الجانب | الـ CLK Bulk Framework بيملكه | الـ Driver بيملكه |
|--------|-------------------------------|-------------------|
| **الـ loop logic** | iteration على الـ array وعمل call واحد لكل عنصر | تعريف الـ array واختيار الأسماء |
| **الـ error rollback** | rollback تلقائي عند فشل أي clock | التعامل مع error code النهائي |
| **الـ DT parsing** | قراءة `clock-names` وربطها بالـ `clk*` | كتابة الـ DT بشكل صح |
| **الـ memory** | alloc وfree في `get_all/put_all` | alloc في الـ static array case |
| **الـ hardware access** | لا شيء | لا شيء — الـ clk core هو اللي بيكلم الـ HW |
| **الـ reference counting** | لا شيء — الـ clk.c بيعملها | لا شيء |
| **الـ prepare/enable order** | يـ prepare الكل ثم يطلب من الـ driver يـ enable | يستدعي `clk_bulk_prepare_enable` بالترتيب الصح |

---

### ملخص الـ API Map

```
clk_bulk_get()                ──→  clk_get()           [per clock]
clk_bulk_get_optional()       ──→  clk_get()           [skip -ENOENT]
clk_bulk_get_all()            ──→  of_clk_get()        [all DT clocks]
clk_bulk_put()                ──→  clk_put()           [reverse order]
clk_bulk_put_all()            ──→  clk_put() + kfree() [dynamic case]
clk_bulk_prepare()            ──→  clk_prepare()       [per clock]
clk_bulk_unprepare()          ──→  clk_unprepare()     [reverse order]
clk_bulk_enable()             ──→  clk_enable()        [per clock]
clk_bulk_disable()            ──→  clk_disable()       [reverse order]
```

كل function في الـ bulk API هي بالظبط **loop wrapper** على الـ single-clock equivalent — ما فيش magic، بس الـ discipline والـ error safety موحدين في مكان واحد.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الـ Flags والـ Config Options والـ Enums

| الرمز | النوع | القيمة | الوصف |
|-------|-------|--------|-------|
| `CONFIG_HAVE_CLK_PREPARE` | config | boolean | لو مفعّل، الـ prepare/unprepare API موجودة ومنفصلة عن الـ enable/disable |
| `CONFIG_COMMON_CLK` | config | boolean | لو مفعّل، يُفعَّل الـ Common Clock Framework وكل الـ notifier APIs |
| `CONFIG_HAVE_CLK` | config | boolean | لو مفعّل، الـ clk_get/clk_put APIs موجودة |
| `__must_check` | macro | compiler attr | يُجبر الـ caller يتحقق من الـ return value وإلا compiler warning |
| `GFP_KERNEL` | flag | بيت | flag لـ kmalloc: ممكن يـ sleep، يُستخدم في السياق العادي |
| `IS_ERR()` | macro | — | يتحقق إن الـ pointer فيه error code مخبّي |
| `PTR_ERR()` | macro | — | يستخرج الـ error code من الـ pointer |
| `IS_ERR_OR_NULL()` | macro | — | يتحقق إن الـ pointer إما NULL أو error |
| `EXPORT_SYMBOL_GPL` | macro | — | يصدّر الرمز بس للـ modules اللي GPL-licensed |
| `EXPORT_SYMBOL` | macro | — | يصدّر الرمز لأي module |

---

### الـ Structs المهمة

#### `struct clk_bulk_data`

**الغرض:** الوحدة الأساسية في الـ bulk clk API — كل عنصر فيها يمثّل ساعة واحدة (clock) باسمها ومرجعها.

```c
struct clk_bulk_data {
    const char  *id;   /* اسم الـ clock — يطابق "clock-names" في الـ DT */
    struct clk  *clk;  /* الـ pointer للـ clock object بعد ما يتجيب */
};
```

| الحقل | النوع | الدور |
|-------|-------|-------|
| `id` | `const char *` | اسم الـ clock المطلوب — بيتحدد مسبقًا من الـ driver أو بيتقرأ من الـ DT |
| `clk` | `struct clk *` | بيملاه الـ framework بعد `clk_get()` أو `of_clk_get()` ناجح |

**علاقته بالـ structs التانية:**
- `clk_bulk_data[]` → يحتوي على `struct clk *` × N
- الـ `struct clk *` → يشاور على الـ internal clock object الـ framework بيديره
- الـ driver بيعرّف الـ array ده مسبقًا بالـ IDs، وبعدين الـ API بتملا الـ `clk` fields

---

#### `struct clk` (opaque — معرّف في داخل الـ core)

**الغرض:** يمثّل clock consumer handle — هو اللي بيتبادله الـ driver مع الـ framework.

| الحقل | ملاحظة |
|-------|--------|
| داخلي | الـ struct مش مكشوف للـ drivers — بس الـ pointer بيتتداول |
| مرتبط بـ `clk_core` | الـ core الداخلي اللي فيه الـ refcount والـ rate والـ ops |

---

#### `struct device_node` (من `linux/of.h`)

**الغرض:** يمثّل node في شجرة الـ Device Tree.

| الحقل المستخدم هنا | الدور |
|--------------------|-------|
| `properties` | فيها `clock-names` و `clocks` |
| — | بيتمشى لـ `of_property_read_string_index()` و`of_clk_get()` |

---

#### `struct device` (من `linux/device.h`)

**الغرض:** يمثّل الـ device اللي عايز الـ clocks.

| الحقل المستخدم | الدور |
|----------------|-------|
| `of_node` | الـ DT node المرتبط بالـ device — بيتجيب عبر `dev_of_node(dev)` |

---

### مخطط علاقات الـ Structs

```
Driver Code
    │
    │  defines statically:
    ▼
┌─────────────────────────────┐
│  clk_bulk_data[]            │  ← array بعرّفه الـ driver
│  ┌─────────────────────┐    │
│  │ [0] id = "core"     │    │
│  │     clk = NULL ──────────────┐
│  ├─────────────────────┤    │  │
│  │ [1] id = "bus"      │    │  │
│  │     clk = NULL ──────────────┼──┐
│  ├─────────────────────┤    │  │  │
│  │ [2] id = "ahb"      │    │  │  │
│  │     clk = NULL ──────────────┼──┼──┐
│  └─────────────────────┘    │  │  │  │
└─────────────────────────────┘  │  │  │
                                  │  │  │
         after clk_bulk_get()     │  │  │
                                  ▼  ▼  ▼
                            ┌──────────────────┐
                            │  struct clk *    │  ← opaque handles
                            │  (from clk_get)  │
                            └────────┬─────────┘
                                     │
                                     ▼
                            ┌──────────────────┐
                            │  clk_core        │  ← internal framework object
                            │  .name           │
                            │  .rate           │
                            │  .enable_count   │
                            │  .prepare_count  │
                            │  .ops ───────────────► clk_ops { .enable, .disable ... }
                            └──────────────────┘


┌──────────────┐        dev_of_node()       ┌──────────────────┐
│ struct device│ ───────────────────────►   │ struct device_node│
│   .of_node   │                            │  "clock-names"   │
└──────────────┘                            │  "clocks" phandle│
                                            └──────────────────┘
                                                    │
                                         of_clk_get(np, i)
                                                    │
                                                    ▼
                                            struct clk *
```

---

### مخطط دورة حياة الـ Clock (Lifecycle)

#### المسار الأساسي — device-based (clk_bulk_get)

```
┌─────────────────────────────────────────────────────────────────┐
│                     CREATION / ACQUISITION                      │
│                                                                 │
│  Driver defines:  struct clk_bulk_data clks[] = {              │
│                       { .id = "core" }, { .id = "bus" }, ...   │
│                   };                                            │
│                                                                 │
│  clk_bulk_get(dev, num_clks, clks)                             │
│    └─► __clk_bulk_get(..., optional=false)                     │
│           └─► for each i: clk_get(dev, clks[i].id)            │
│                   └─► fills clks[i].clk                        │
│                   └─► on error: goto err → clk_bulk_put(i,clks)│
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        PREPARATION                              │
│  (only if CONFIG_HAVE_CLK_PREPARE)                              │
│                                                                 │
│  clk_bulk_prepare(num_clks, clks)                               │
│    └─► for each i: clk_prepare(clks[i].clk)                    │
│             └─► may sleep — allocates HW resources              │
│             └─► on error: clk_bulk_unprepare(i, clks)          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         ENABLING                                │
│                                                                 │
│  clk_bulk_enable(num_clks, clks)                                │
│    └─► for each i: clk_enable(clks[i].clk)                     │
│             └─► atomic-safe, gates the clock ON                 │
│             └─► on error: clk_bulk_disable(i, clks)            │
└─────────────────────────────────────────────────────────────────┘
                              │
                        [DEVICE IN USE]
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        DISABLING                                │
│                                                                 │
│  clk_bulk_disable(num_clks, clks)                               │
│    └─► for each i (reverse): clk_disable(clks[i].clk)          │
│             └─► atomic-safe, gates the clock OFF               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       UNPREPARING                               │
│  (only if CONFIG_HAVE_CLK_PREPARE)                              │
│                                                                 │
│  clk_bulk_unprepare(num_clks, clks)                             │
│    └─► for each i (reverse): clk_unprepare(clks[i].clk)        │
│             └─► may sleep                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        TEARDOWN                                 │
│                                                                 │
│  clk_bulk_put(num_clks, clks)                                   │
│    └─► for each i (reverse): clk_put(clks[i].clk)              │
│             └─► decrements refcount, clks[i].clk = NULL        │
└─────────────────────────────────────────────────────────────────┘
```

#### المسار البديل — DT-based مع dynamic allocation (clk_bulk_get_all)

```
clk_bulk_get_all(dev, &clks)
  └─► dev_of_node(dev)  ──►  struct device_node *np
  └─► of_clk_bulk_get_all(np, &clks)
        └─► of_clk_get_parent_count(np)  ──►  num_clks
        └─► kmalloc_array(num_clks, sizeof(clk_bulk_data))
        └─► of_clk_bulk_get(np, num_clks, clk_bulk)
              └─► for each i:
                    of_property_read_string_index(np, "clock-names", i, &clks[i].id)
                    of_clk_get(np, i)  ──►  clks[i].clk
        └─► returns num_clks (positive = success)

[teardown]
clk_bulk_put_all(num_clks, clks)
  └─► clk_bulk_put(num_clks, clks)
  └─► kfree(clks)   ← مهم: بس هنا بيحرر الـ array نفسه
```

---

### مخطط تدفق الـ Calls (Call Flow)

#### `clk_bulk_get` — المسار الكامل

```
driver calls clk_bulk_get(dev, N, clks[])
  │
  └─► __clk_bulk_get(dev, N, clks, optional=false)
        │
        ├─[init]─► for i=0..N-1: clks[i].clk = NULL
        │
        └─[loop]─► for i=0..N-1:
                     clk_get(dev, clks[i].id)
                       │
                       ├─[ok]──► clks[i].clk = returned clk*
                       │
                       └─[err]─► ret = PTR_ERR(clks[i].clk)
                                 clks[i].clk = NULL
                                 if (ret == -ENOENT && optional) continue
                                 dev_err_probe(dev, ret, ...)
                                 goto err:
                                   clk_bulk_put(i, clks)
                                     └─► for j=i-1..0: clk_put(clks[j].clk)
                                   return ret
```

#### `clk_bulk_enable` — مع error rollback

```
driver calls clk_bulk_enable(N, clks[])
  │
  └─► for i=0..N-1:
        clk_enable(clks[i].clk)
          │
          ├─[ok]──► continue
          │
          └─[err]─► pr_err("Failed to enable clk '%s'", clks[i].id)
                    goto err:
                      clk_bulk_disable(i, clks)
                        └─► for j=i-1..0: clk_disable(clks[j].clk)
                      return ret
```

#### `clk_bulk_prepare` — مع error rollback

```
driver calls clk_bulk_prepare(N, clks[])
  │
  └─► for i=0..N-1:
        clk_prepare(clks[i].clk)    ← may sleep
          │
          ├─[ok]──► continue
          │
          └─[err]─► pr_err("Failed to prepare clk '%s'", clks[i].id)
                    goto err:
                      clk_bulk_unprepare(i, clks)
                        └─► for j=i-1..0: clk_unprepare(clks[j].clk)
                      return ret
```

#### `clk_bulk_put` و `clk_bulk_put_all`

```
clk_bulk_put(N, clks[])
  └─► while (--N >= 0):
        clk_put(clks[N].clk)
        clks[N].clk = NULL

clk_bulk_put_all(N, clks[])
  ├─► if IS_ERR_OR_NULL(clks): return   ← guard
  ├─► clk_bulk_put(N, clks)
  └─► kfree(clks)                        ← يحرر الـ array نفسه
```

---

### استراتيجية الـ Locking

الـ `clk-bulk.c` بذاته **لا يمتلك أي lock** — هو wrapper محض فوق الـ single-clock APIs.

| الطبقة | الـ Lock | ما بيحميه |
|--------|---------|-----------|
| `clk_bulk_*` (هذا الملف) | لا يوجد | wrapper فقط |
| `clk_enable/disable` | `clk_enable_lock` (spinlock) | الـ enable_count و HW register access |
| `clk_prepare/unprepare` | `prepare_lock` (mutex) | الـ prepare_count — مسموح يـ sleep |
| `clk_get/put` | internal clk_core lock | الـ refcount والـ lookup |

**قاعدة الترتيب (Lock Ordering):**

```
prepare_lock (mutex)  ──►  clk_enable_lock (spinlock)

يعني: لو محتاج الاتنين، اقفل prepare_lock الأول
      وده بالضبط هو ترتيب clk_prepare_enable()
```

**نقاط مهمة:**

- الـ `clk_bulk_enable/disable` → **atomic-safe** → ممكن تتنادى من interrupt context
- الـ `clk_bulk_prepare/unprepare` → **قد تـ sleep** → ممنوع في interrupt context
- الـ `clk_bulk_get/put` → **قد تـ sleep** → ممنوع في interrupt context
- الـ `clk_bulk_put_all` بتتحقق من `IS_ERR_OR_NULL` قبل ما تـ lock أي حاجة — defensive check

**نمط الـ rollback عند الـ Error:**

الـ bulk API بتـ rollback بالـ reverse order عشان تضمن إن الـ clocks اللي اتجابت بس هي اللي بتتحرر:

```
للـ get:  put clocks 0..i-1    (اللي اتجابوا قبل الفشل)
للـ enable: disable clocks 0..i-1
للـ prepare: unprepare clocks 0..i-1
```

ده بيضمن إنك مش بتـ put/disable clock مش محجوز/معمول له enable.
## Phase 4: شرح الـ Functions

---

### ملخص الـ Functions — Cheatsheet

#### Group 1: Acquisition (Get)

| Function | Export | Optional | Source |
|---|---|---|---|
| `__clk_bulk_get()` | static | flag param | clk-bulk.c |
| `clk_bulk_get()` | `EXPORT_SYMBOL` | No | clk-bulk.c |
| `clk_bulk_get_optional()` | `EXPORT_SYMBOL_GPL` | Yes | clk-bulk.c |
| `clk_bulk_get_all()` | `EXPORT_SYMBOL` | — | clk-bulk.c |
| `of_clk_bulk_get()` | static | No | clk-bulk.c |
| `of_clk_bulk_get_all()` | static | — | clk-bulk.c |

#### Group 2: Release (Put)

| Function | Export | Frees array? |
|---|---|---|
| `clk_bulk_put()` | `EXPORT_SYMBOL_GPL` | No |
| `clk_bulk_put_all()` | `EXPORT_SYMBOL` | Yes (`kfree`) |

#### Group 3: Prepare / Unprepare (sleepable)

| Function | Export | Guard |
|---|---|---|
| `clk_bulk_prepare()` | `EXPORT_SYMBOL_GPL` | `CONFIG_HAVE_CLK_PREPARE` |
| `clk_bulk_unprepare()` | `EXPORT_SYMBOL_GPL` | `CONFIG_HAVE_CLK_PREPARE` |

#### Group 4: Enable / Disable (atomic-safe)

| Function | Export | May sleep? |
|---|---|---|
| `clk_bulk_enable()` | `EXPORT_SYMBOL_GPL` | No |
| `clk_bulk_disable()` | `EXPORT_SYMBOL_GPL` | No |

---

### البنية الأساسية: `struct clk_bulk_data`

```c
/* include/linux/clk.h */
struct clk_bulk_data {
    const char  *id;   /* consumer clock name, e.g. "apb", "ahb" */
    struct clk  *clk;  /* pointer filled by bulk get, NULL until then */
};
```

الـ driver بيعرّف array من الـ `clk_bulk_data`، كل عنصر فيها بيحدد `id` بالاسم المكتوب في الـ DTS تحت `clock-names`. الـ bulk API بتملأ حقل `clk` تلقائيًا.

---

### Group 1: Acquisition — جلب الـ Clocks

الهدف من الـ group ده هو الحصول على references لكل الـ clocks اللي الـ driver محتاجها دفعة واحدة، مع rollback تلقائي لو أي clock فشل.

---

#### `of_clk_bulk_get` (static)

```c
static int __must_check of_clk_bulk_get(struct device_node *np, int num_clks,
                                        struct clk_bulk_data *clks);
```

بتجيب `num_clks` clock من الـ device tree node مباشرةً باستخدام `of_clk_get()`. بتقرأ اسم كل clock من الـ property `clock-names` باستخدام `of_property_read_string_index()` وبتحط الاسم في `clks[i].id`. لو أي clock فشل بترجع للـ `err` label وبتعمل `clk_bulk_put()` على الـ clocks اللي اتجابت بالفعل.

**Parameters:**
- `np` — الـ `device_node` اللي فيه الـ `clocks` و `clock-names` properties
- `num_clks` — عدد الـ clocks المطلوبة
- `clks` — pointer لـ array من `clk_bulk_data` مخصصة مسبقًا

**Return:** `0` on success، negative errno on failure

**Key details:**
- بتـ zero-initialize جميع الـ entries أول الأمر قبل الـ loop — ده ضروري عشان الـ `clk_bulk_put` في الـ error path يشتغل صح ومش يـ call `clk_put(NULL)` على entries لسه ما اتملتش
- مش exported — للاستخدام الداخلي فقط من `of_clk_bulk_get_all`

**Pseudocode:**
```
zero all clks[].id and clks[].clk
for i in 0..num_clks:
    read clock-names[i] → clks[i].id
    clks[i].clk = of_clk_get(np, i)
    if ERR:
        clks[i].clk = NULL
        goto err → clk_bulk_put(i, clks)
        return ret
return 0
```

---

#### `of_clk_bulk_get_all` (static)

```c
static int __must_check of_clk_bulk_get_all(struct device_node *np,
                                            struct clk_bulk_data **clks);
```

بتحدد عدد الـ parent clocks في الـ node باستخدام `of_clk_get_parent_count()`، بتخصص array ديناميكية بـ `kmalloc_array()`، وبتستدعي `of_clk_bulk_get()`. لو فيه error بتـ `kfree` الـ array وبترجع الـ error. لو نجحت بتحط الـ pointer في `*clks` وبترجع عدد الـ clocks.

**Parameters:**
- `np` — الـ device node
- `clks` — pointer-to-pointer، بتملأه بـ pointer للـ array المخصصة

**Return:** عدد الـ clocks on success، `0` لو مفيش clocks، negative errno on failure

**Key details:**
- الـ caller مسؤول عن تحرير الـ array بـ `clk_bulk_put_all()` لأنها بتعمل `kfree` على الـ array نفسها
- `kmalloc_array` بتستخدم `GFP_KERNEL` — مينفعش يُستدعى من interrupt context

---

#### `__clk_bulk_get` (static)

```c
static int __clk_bulk_get(struct device *dev, int num_clks,
                          struct clk_bulk_data *clks, bool optional);
```

ده الـ core implementation للـ device-based bulk get. بتستدعي `clk_get(dev, clks[i].id)` لكل entry. لو `optional=true` والـ clock مش موجود (`-ENOENT`) بتتجاوز الـ entry وتفضل في `NULL` وبتكمل. لو `optional=false` أو الـ error مش `-ENOENT` بتـ bail out وبتعمل `clk_bulk_put()` على الـ clocks اللي اتجابت.

**Parameters:**
- `dev` — الـ consumer device (بتستخدمه `clk_get` لـ lookup)
- `num_clks` — عدد الـ clocks
- `clks` — الـ array، الـ `id` field لازم يكون مملوء مسبقًا
- `optional` — `true`: ignore `-ENOENT`؛ `false`: treat as fatal error

**Return:** `0` on success، negative errno on failure

**Key details:**
- بيستخدم `dev_err_probe()` بدل `dev_err()` عشان يدعم الـ deferred probe pattern بشكل نظيف
- الـ error path بيعمل `clk_bulk_put(i, clks)` — بس للـ entries من `0` لـ `i-1`، مش الكل

**Pseudocode:**
```
zero all clks[].clk
for i in 0..num_clks:
    clks[i].clk = clk_get(dev, clks[i].id)
    if ERR:
        if ret == -ENOENT && optional:
            continue   /* leave clks[i].clk = NULL */
        dev_err_probe(...)
        goto err → clk_bulk_put(i, clks)
        return ret
return 0
```

---

#### `clk_bulk_get`

```c
int __must_check clk_bulk_get(struct device *dev, int num_clks,
                              struct clk_bulk_data *clks);
```

الـ public API للحصول على عدد محدد من الـ clocks بناءً على الـ `id` fields المملوءة مسبقًا في الـ array. بتستدعي `__clk_bulk_get(..., false)` يعني أي clock ناقص هيكون fatal error.

**Parameters:**
- `dev` — الـ consumer device
- `num_clks` — عدد الـ entries في الـ array
- `clks` — array من `clk_bulk_data` مع الـ `id` fields مملوءة

**Return:** `0` on success، negative errno on failure

**Export:** `EXPORT_SYMBOL` — متاحة للـ modules سواء GPL أو non-GPL

**Who calls it:** driver probe functions — مينفعش من interrupt context

---

#### `clk_bulk_get_optional`

```c
int __must_check clk_bulk_get_optional(struct device *dev, int num_clks,
                                       struct clk_bulk_data *clks);
```

نفس `clk_bulk_get` بالضبط لكن بـ `optional=true`، يعني لو أي clock مش موجود في الـ DTS (`-ENOENT`) بتكمل وتسيب `clks[i].clk = NULL` بدل ما تفشل. مفيدة للـ clocks الاختيارية اللي الـ hardware ممكن يشتغل من غيرها.

**Return:** `0` on success حتى لو بعض الـ clocks `NULL`

**Export:** `EXPORT_SYMBOL_GPL`

---

#### `clk_bulk_get_all`

```c
int __must_check clk_bulk_get_all(struct device *dev,
                                  struct clk_bulk_data **clks);
```

بتجيب كل الـ clocks المعرّفة في الـ DTS للـ device دفعة واحدة بدون ما الـ driver يحتاج يعرف عددهم مسبقًا. بتجيب الـ `device_node` من الـ device بـ `dev_of_node()` وبتستدعي `of_clk_bulk_get_all()`. لو الـ device ملوش DTS node بترجع `0`.

**Parameters:**
- `dev` — الـ consumer device
- `clks` — pointer-to-pointer، بيتملأ بـ array مخصصة ديناميكيًا

**Return:** عدد الـ clocks on success، `0` لو مفيش DTS node أو مفيش clocks، negative errno on failure

**Key details:**
- الـ caller لازم يستدعي `clk_bulk_put_all()` مش `clk_bulk_put()` عشان يتحرر الـ array نفسها

**Export:** `EXPORT_SYMBOL`

---

### Group 2: Release (Put) — تحرير الـ Clocks

---

#### `clk_bulk_put`

```c
void clk_bulk_put(int num_clks, struct clk_bulk_data *clks);
```

بتحرر references الـ clocks بترتيب عكسي (من `num_clks-1` لـ `0`) بـ `clk_put()` وبتزيّر الـ `clk` pointer بعد كل تحرير. الترتيب العكسي مهم لأن بعض الـ clock trees بتكون فيها dependencies — تحرير العكس بيتجنب الـ use-after-free في الـ framework.

**Parameters:**
- `num_clks` — عدد الـ entries في الـ array
- `clks` — الـ array (مش بتحررها، بس الـ references اللي جوّاها)

**Key details:**
- بتـ null-ify الـ pointer بعد `clk_put()` — defense against double-free
- بتُستدعى من الـ error paths في كل الـ `get` functions
- مش بتعمل `kfree` للـ array نفسها — الـ caller مسؤول

**Export:** `EXPORT_SYMBOL_GPL`

---

#### `clk_bulk_put_all`

```c
void clk_bulk_put_all(int num_clks, struct clk_bulk_data *clks);
```

للـ arrays المخصصة ديناميكيًا (من `clk_bulk_get_all()`). بتتحقق من `IS_ERR_OR_NULL(clks)` أولًا ثم بتستدعي `clk_bulk_put()` ثم `kfree(clks)`. الـ `IS_ERR_OR_NULL` check بتحمي من الحالات اللي `clk_bulk_get_all()` رجعت `0` clocks وسابت الـ pointer بـ NULL.

**Parameters:**
- `num_clks` — العدد (ممكن يكون `0`)
- `clks` — الـ array المخصصة ديناميكيًا أو `NULL`

**Key details:**
- الـ pair الصحيح: `clk_bulk_get_all()` ↔ `clk_bulk_put_all()`
- الـ pair الصحيح: `clk_bulk_get()` ↔ `clk_bulk_put()`

**Export:** `EXPORT_SYMBOL`

---

### Group 3: Prepare / Unprepare — التجهيز قبل التشغيل

الـ prepare/unprepare API موجودة بس لو `CONFIG_HAVE_CLK_PREPARE` محددة. الـ prepare step بتعمل الـ work اللي ممكن يـ sleep (مثل power domain enable، PLL lock wait). لازم يتعمل قبل `clk_bulk_enable()`.

---

#### `clk_bulk_prepare`

```c
int __must_check clk_bulk_prepare(int num_clks,
                                  const struct clk_bulk_data *clks);
```

بتعمل `clk_prepare()` لكل clock بالترتيب من `0` لـ `num_clks-1`. لو أي clock فشل بتعمل `clk_bulk_unprepare(i, clks)` على الـ clocks اللي اتجهزت وبترجع الـ error.

**Parameters:**
- `num_clks` — عدد الـ clocks
- `clks` — الـ array (const لأن الـ prepare مش بيغير الـ struct)

**Return:** `0` on success، negative errno on failure

**Key details:**
- **May sleep** — ممنوع استخدامها من interrupt context أو atomic context
- الـ error rollback بيعمل `clk_bulk_unprepare` بس على الـ clocks اللي اتجهزت فعلًا
- لازم `clk_bulk_enable()` يُستدعى بعدها

**Export:** `EXPORT_SYMBOL_GPL`

**Pseudocode:**
```
for i in 0..num_clks:
    ret = clk_prepare(clks[i].clk)
    if ret:
        pr_err(...)
        clk_bulk_unprepare(i, clks)  /* rollback */
        return ret
return 0
```

---

#### `clk_bulk_unprepare`

```c
void clk_bulk_unprepare(int num_clks, const struct clk_bulk_data *clks);
```

بتعمل `clk_unprepare()` على كل الـ clocks بالترتيب العكسي. لازم تُستدعى **بعد** `clk_bulk_disable()` مش قبلها.

**Parameters:**
- `num_clks` — عدد الـ clocks
- `clks` — الـ array

**Key details:**
- **May sleep**
- الترتيب العكسي مهم لتجنب dependency issues في الـ clock tree
- مفيش return value — مش المفروض تفشل

**Export:** `EXPORT_SYMBOL_GPL`

---

### Group 4: Enable / Disable — التشغيل والإيقاف

الـ enable/disable API atomic-safe، يعني **مش بتـ sleep**. اللي بيـ sleep هو الـ prepare/unprepare. الـ enable بس بيشغّل الـ gate (hardware bit). لازم الـ clock يكون prepared قبل enable.

---

#### `clk_bulk_enable`

```c
int __must_check clk_bulk_enable(int num_clks, const struct clk_bulk_data *clks);
```

بتعمل `clk_enable()` لكل clock من `0` لـ `num_clks-1`. لو أي clock فشل بتعمل `clk_bulk_disable(i, clks)` على الـ clocks اللي اتفعّلت وبترجع الـ error.

**Parameters:**
- `num_clks` — عدد الـ clocks
- `clks` — الـ array (const)

**Return:** `0` on success، negative errno on failure

**Key details:**
- **Must not sleep** — ممكن تُستدعى من atomic context (بعد prepare)
- لازم `clk_bulk_prepare()` يُستدعى قبلها
- **لازم** يُستدعى `clk_bulk_disable()` قبل `clk_bulk_unprepare()` عند الـ cleanup

**Export:** `EXPORT_SYMBOL_GPL`

**Pseudocode:**
```
for i in 0..num_clks:
    ret = clk_enable(clks[i].clk)
    if ret:
        pr_err(...)
        clk_bulk_disable(i, clks)  /* rollback */
        return ret
return 0
```

---

#### `clk_bulk_disable`

```c
void clk_bulk_disable(int num_clks, const struct clk_bulk_data *clks);
```

بتعمل `clk_disable()` على كل الـ clocks بالترتيب العكسي. لازم تُستدعى **قبل** `clk_bulk_unprepare()`.

**Parameters:**
- `num_clks` — عدد الـ clocks
- `clks` — الـ array

**Key details:**
- **Must not sleep**
- الـ `clk_disable()` بتـ decrement الـ enable reference count — مش بتوقف الـ clock إلا لما العداد يوصل صفر

**Export:** `EXPORT_SYMBOL_GPL`

---

### الـ Lifecycle الكامل — ASCII Diagram

```
          ┌──────────────────────────────────────────────────────┐
          │               clk-bulk lifecycle                     │
          └──────────────────────────────────────────────────────┘

  [DTS node]                  [Driver Probe]             [Driver Remove]
       │                            │                           │
       ▼                            ▼                           ▼
 of_clk_bulk_get()        clk_bulk_get()              clk_bulk_disable()
       │                 clk_bulk_get_all()                     │
       │                 clk_bulk_get_optional()    clk_bulk_unprepare()
       │                            │                           │
       │                  clk_bulk_prepare()         clk_bulk_put()
       │                            │              clk_bulk_put_all()
       │                  clk_bulk_enable()
       │                            │
       │                    [Device Running]
       │                            │
       │                  clk_bulk_disable()  ◄── suspend / pm ops
       │                  clk_bulk_unprepare()
       │                            │
       └────────────────────────────┘


  ┌───────────────┬──────────────────┬───────────────────┐
  │ Step          │ May Sleep?       │ Function          │
  ├───────────────┼──────────────────┼───────────────────┤
  │ Get           │ Yes (GFP_KERNEL) │ clk_bulk_get*     │
  │ Prepare       │ Yes              │ clk_bulk_prepare  │
  │ Enable        │ No               │ clk_bulk_enable   │
  │ Disable       │ No               │ clk_bulk_disable  │
  │ Unprepare     │ Yes              │ clk_bulk_unprepare│
  │ Put           │ No               │ clk_bulk_put*     │
  └───────────────┴──────────────────┴───────────────────┘
```

---

### الـ Error Handling Pattern — Rollback Symmetry

الـ file بيتّبع نمط ثابت في كل function بتمر على array:

```c
/* Generic pattern used by prepare, enable, get */
for (i = 0; i < num_clks; i++) {
    ret = op(clks[i]);
    if (ret) {
        /* rollback only the successful ones: 0..i-1 */
        reverse_op(i, clks);
        return ret;
    }
}
```

ده بيضمن إن لو عندك 5 clocks وفشل الـ 3rd، الـ cleanup بيتعمل بس على الـ 2 الأوليين اللي نجحوا. مش بيعمل cleanup على الـ entries اللي لم تُعالج بعد.

---

### ملاحظات على الـ Export Symbols

| Function | Symbol Type | المعنى |
|---|---|---|
| `clk_bulk_get` | `EXPORT_SYMBOL` | متاحة لأي module |
| `clk_bulk_get_all` | `EXPORT_SYMBOL` | متاحة لأي module |
| `clk_bulk_put_all` | `EXPORT_SYMBOL` | متاحة لأي module |
| `clk_bulk_get_optional` | `EXPORT_SYMBOL_GPL` | GPL modules فقط |
| `clk_bulk_put` | `EXPORT_SYMBOL_GPL` | GPL modules فقط |
| `clk_bulk_prepare` | `EXPORT_SYMBOL_GPL` | GPL modules فقط |
| `clk_bulk_unprepare` | `EXPORT_SYMBOL_GPL` | GPL modules فقط |
| `clk_bulk_enable` | `EXPORT_SYMBOL_GPL` | GPL modules فقط |
| `clk_bulk_disable` | `EXPORT_SYMBOL_GPL` | GPL modules فقط |
## Phase 5: دليل الـ Debugging الشامل

الـ `clk-bulk.c` بتوفر واجهة bulk لإدارة مجموعة clocks مع بعض — get/put/prepare/enable/disable. الـ debugging هنا بيتمحور حول: إيه الـ clocks اللي اتجبت؟ هل اتعملها prepare/enable صح؟ وإيه اللي حصل لو فيه error في النص.

---

### Software Level

#### 1. debugfs

الـ Common Clock Framework بيعمل tree كامل في debugfs. المسار الافتراضي:

```
/sys/kernel/debug/clk/
```

كل clock عندها directory خاص بيها فيه:
- `clk_rate` — التردد الحالي
- `clk_enable_count` — عدد مرات enable
- `clk_prepare_count` — عدد مرات prepare
- `clk_flags` — الـ flags الحالية (CLK_IS_CRITICAL، إلخ)
- `clk_parent` — اسم الـ parent clock

```bash
# اقرأ الـ tree كلها
cat /sys/kernel/debug/clk/clk_summary

# clock معينة باسمها (مثلاً apb_clk)
cat /sys/kernel/debug/clk/apb_clk/clk_rate
cat /sys/kernel/debug/clk/apb_clk/clk_enable_count
cat /sys/kernel/debug/clk/apb_clk/clk_prepare_count
cat /sys/kernel/debug/clk/apb_clk/clk_flags
```

مثال output من `clk_summary`:

```
                                 enable  prepare  protect                                duty
   clock                          count    count    count        rate   accuracy phase  cycle
---------------------------------------------------------------------------------------------
 xtal_24m                             1        1        0    24000000          0     0  50000
  pll_sys                             2        2        0   600000000          0     0  50000
   apb_clk                            1        1        0    75000000          0     0  50000
```

لو `enable_count` > 0 بس الـ hardware مش شغال — ده مؤشر bug.

**الـ `clk_possible_parents`** بتديك كل الـ parents المحتملة:

```bash
cat /sys/kernel/debug/clk/apb_clk/clk_possible_parents
```

#### 2. sysfs

الـ CCF مش بيعمل entries كتير في sysfs مباشرةً، بس الـ devices اللي بتستخدم clocks ليها:

```bash
# الـ device links (supplier/consumer)
ls /sys/bus/platform/devices/<device>/clocks/

# من خلال power domain
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
```

#### 3. ftrace — Tracepoints

الـ CCF عنده tracepoints جاهزة في `include/trace/events/clk.h`:

```bash
# اعرض كل events المتاحة
ls /sys/kernel/debug/tracing/events/clk/

# enable الأحداث المهمة لـ bulk operations
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_prepare/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_unprepare/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_enable/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_disable/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... شغّل الـ driver
cat /sys/kernel/debug/tracing/trace
```

مثال output:

```
           <...>-123   [001] ....   45.123: clk_prepare: apb_clk
           <...>-123   [001] ....   45.124: clk_enable: apb_clk
           <...>-123   [001] ....   46.001: clk_disable: apb_clk
           <...>-123   [001] ....   46.002: clk_unprepare: apb_clk
```

#### 4. printk / Dynamic Debug

الرسائل المدمجة في `clk-bulk.c` نفسها:

```c
pr_err("%pOF: Failed to get clk index: %d ret: %d\n", np, i, ret);
pr_err("Failed to prepare clk '%s': %d\n", clks[i].id, ret);
pr_err("Failed to enable clk '%s': %d\n", clks[i].id, ret);
```

لتفعيل **dynamic debug** لكل رسائل الـ clk subsystem:

```bash
# كل رسائل clk-bulk.c
echo "file drivers/clk/clk-bulk.c +p" > /sys/kernel/debug/dynamic_debug/control

# كل رسائل الـ clk subsystem
echo "file drivers/clk/*.c +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل dev_err_probe بشكل عام (مفيد لـ -EPROBE_DEFER)
echo "module <module_name> +p" > /sys/kernel/debug/dynamic_debug/control
```

لو `dev_err_probe` بتطبع `-EPROBE_DEFER` — ده طبيعي، يعني clock provider لسه ما اتسجلش.

#### 5. Kernel Config Options للـ Debugging

| Config | الوظيفة |
|--------|----------|
| `CONFIG_COMMON_CLK` | الـ framework الأساسي — لازم يكون enabled |
| `CONFIG_CLK_DEBUG` | يعمل debugfs entries لكل clock |
| `CONFIG_DEBUG_KERNEL` | يفعّل assertions عامة في الكيرنل |
| `CONFIG_PROVE_LOCKING` | يكشف lockdep violations في prepare_lock |
| `CONFIG_DEBUG_OBJECTS` | يتتبع صحة الكائنات (clk structs) |
| `CONFIG_KALLSYMS` | يظهر أسماء الـ symbols في stack traces |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug` / `dev_dbg` بشكل dynamic |
| `CONFIG_OF` | لازم لـ `of_clk_bulk_get` |

```bash
# تأكد إن debugfs متوصلة
grep CONFIG_CLK_DEBUG /boot/config-$(uname -r)
grep CONFIG_COMMON_CLK /boot/config-$(uname -r)
```

#### 6. أدوات خاصة بالـ Subsystem

```bash
# clk_summary — أهم أداة
cat /sys/kernel/debug/clk/clk_summary

# clk_dump — JSON-like output
cat /sys/kernel/debug/clk/clk_dump

# فلترة على clock معينة
grep -A5 "apb_clk" /sys/kernel/debug/clk/clk_summary

# عد الـ clocks اللي enable_count > 0
awk '$3 > 0' /sys/kernel/debug/clk/clk_summary
```

لمعرفة الـ clocks اللي بيستخدمها device معين عبر OF:

```bash
# قرأ clock-names و clocks من DT
cat /proc/device-tree/<path-to-node>/clock-names
# أو
fdtget /boot/dtb.dtb /path/to/node clock-names
```

#### 7. جدول رسائل الـ Error الشائعة

| الرسالة | المعنى | الحل |
|---------|--------|-------|
| `Failed to get clk index: N ret: -ENOENT` | الـ clock مش موجود في DT أو ما اتسجلش | تأكد من `clock-names` في DTS و وجود الـ provider |
| `Failed to get clk index: N ret: -EPROBE_DEFER` | الـ clock provider لسه ما اتسجلش | ده طبيعي — الكيرنل هيعيد الـ probe تاني |
| `Failed to get clk '%s'` | `clk_get()` فشلت للـ device | تأكد من اسم الـ clock في `clock-names` |
| `Failed to prepare clk '%s': N` | `clk_prepare()` فشلت | غالباً hardware init error أو parent مش متاح |
| `Failed to enable clk '%s': N` | `clk_enable()` فشلت | الـ clock مش prepared، أو hardware gate issue |
| `%pOF: Failed to get clk index: %d ret: -EINVAL` | رقم الـ clock index غلط في DT | راجع عدد الـ clocks في `clocks = <...>` في DTS |

#### 8. نقاط استراتيجية لـ dump_stack() / WARN_ON()

```c
/* تحقق إن الـ bulk array مش NULL قبل الاستخدام */
WARN_ON(!clks);
WARN_ON(num_clks <= 0);

/* بعد clk_bulk_get — تأكد إن كل clk اتجب */
for (i = 0; i < num_clks; i++)
    WARN_ON(IS_ERR_OR_NULL(clks[i].clk));

/* لو حصل double-put (bug شائع) */
WARN_ON(clks[i].clk == NULL); /* قبل clk_put */

/* في enable path لو مش prepared */
/* dump_stack() هنا بيفيد لمعرفة من فين اتنادى */
if (ret) {
    dump_stack();
    goto err;
}
```

---

### Hardware Level

#### 1. التحقق إن حالة الـ Hardware مطابقة للـ Kernel State

الـ clk-bulk بيشتغل فوق الـ CCF اللي بيتعامل مع registers. للتحقق:

```bash
# قارن clk_enable_count مع حالة الـ gate register في Hardware
cat /sys/kernel/debug/clk/apb_clk/clk_enable_count
# لو القيمة > 0 يبقى الـ gate register لازم يكون = 1 (enabled)

# قارن clk_rate مع ما تحسبه يدوياً من PLL registers
cat /sys/kernel/debug/clk/pll_sys/clk_rate
```

#### 2. Register Dump Techniques

لقراءة الـ clock controller registers مباشرة:

```bash
# باستخدام devmem2 (لو مثبّت)
devmem2 0xFE010000 w   # مثال: عنوان CCU register

# باستخدام /dev/mem
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 0x1000, offset=0xFE010000)
    val = struct.unpack('<I', m.read(4))[0]
    print(hex(val))
"

# باستخدام io tool (من busybox أو package منفصل)
io -4 -r 0xFE010000

# قراءة range من registers
for addr in $(seq 0xFE010000 4 0xFE010100); do
    printf "0x%08x: " $addr
    devmem2 $addr w 2>/dev/null | grep -o "0x[0-9A-Fa-f]*$"
done
```

**مهم:** عنوان الـ CCU بيختلف حسب الـ SoC — راجع الـ datasheet أو الـ DTS.

#### 3. Logic Analyzer / Oscilloscope Tips

لو الـ clock مش بتشتغل:

- **Oscilloscope**: قيس على الـ clock pin المفروض يكون فيه output — تأكد من التردد والـ amplitude.
- **Logic Analyzer**: صوّر الـ I2C/SPI bus اللي بيتكلم مع الـ clock generator (لو external).
- للـ on-chip clocks: استخدم الـ test points لو الـ board بتوفرها، أو راقب سلوك الـ peripheral اللي بتاخد الـ clock.

نقاط مهمة:
- تردد مختلف عن المتوقع → مشكلة في PLL dividers.
- لا توجد إشارة خالص → الـ gate مش بيتفتح (enable_count = 0 أو hardware bug).
- إشارة بتيجي وترجع → toggling بسبب repeated probe failures.

#### 4. Hardware Issues → Kernel Log Patterns

| المشكلة في الـ Hardware | الـ Pattern في dmesg |
|------------------------|----------------------|
| الـ clock provider مش شغال (power domain مغلق) | `Failed to get clk index: N ret: -EPROBE_DEFER` (متكرر بلا نهاية) |
| الـ PLL مش بيقدر يتفتح | `Failed to enable clk 'pll': -EIO` |
| الـ gate register read-only أو protected | `Failed to prepare clk: -EPERM` |
| clock oscillator مش موجود فعلياً | `Failed to get clk index: 0 ret: -ENOENT` |
| power supply للـ clock domain منخفض | clock rate متذبذب أو `clk_enable` بيفشل عشوائياً |

```bash
# شوف كل رسائل الـ clk في boot
dmesg | grep -i "clk\|clock" | head -50

# رسائل الـ EPROBE_DEFER تحديداً
dmesg | grep -i "probe_defer\|EPROBE_DEFER"

# رسائل الـ Failed
dmesg | grep "Failed to.*clk"
```

#### 5. Device Tree Debugging

```bash
# تأكد إن الـ DT اتحمل صح
ls /proc/device-tree/

# اقرأ clock-names لـ node معين
cat /proc/device-tree/soc/uart@fe010000/clock-names
# أو باستخدام fdtget
fdtget /boot/dtb.dtb /soc/uart@fe010000 clock-names
fdtget /boot/dtb.dtb /soc/uart@fe010000 clocks

# تأكد إن الـ phandle بيشاور على provider صح
# الـ clocks property بتحتوي phandle + specifier
# مثال: clocks = <&ccu CLK_APB>, <&ccu CLK_AHB>

# اعرض الـ DT كـ text
dtc -I fs /proc/device-tree 2>/dev/null | grep -A20 "uart@"
```

**تحقق إن عدد الـ entries في `clock-names` = عدد الـ entries في `clocks`:**

```bash
# count clock-names
fdtget /boot/dtb.dtb /soc/uart@fe010000 clock-names | wc -w

# count clocks (كل specifier عادةً بيتكون من phandle + 1 cell)
fdtget /boot/dtb.dtb /soc/uart@fe010000 clocks | wc -w
```

لو فيه mismatch → `of_clk_bulk_get` هيفشل بـ `-EINVAL`.

---

### Practical Commands

#### مجموعة commands جاهزة للنسخ

```bash
# === 1. فحص عام للـ clock tree ===
cat /sys/kernel/debug/clk/clk_summary | column -t

# === 2. فحص clock معينة (غيّر apb_clk لاسم الـ clock بتاعتك) ===
CLK=apb_clk
echo "Rate:    $(cat /sys/kernel/debug/clk/$CLK/clk_rate)"
echo "Enable:  $(cat /sys/kernel/debug/clk/$CLK/clk_enable_count)"
echo "Prepare: $(cat /sys/kernel/debug/clk/$CLK/clk_prepare_count)"
echo "Flags:   $(cat /sys/kernel/debug/clk/$CLK/clk_flags)"
echo "Parent:  $(cat /sys/kernel/debug/clk/$CLK/clk_parent)"

# === 3. enable كل events الـ clk في ftrace ===
cd /sys/kernel/debug/tracing
echo 0 > tracing_on
echo > trace
echo 1 > events/clk/enable
echo 1 > tracing_on
# شغّل الـ driver أو الـ module
echo 0 > tracing_on
cat trace | grep -E "clk_prepare|clk_enable|clk_disable|clk_unprepare"

# === 4. dynamic debug لـ clk-bulk ===
echo "file drivers/clk/clk-bulk.c +pflm" > /sys/kernel/debug/dynamic_debug/control
# الـ flags: p=print, f=filename, l=line, m=module

# === 5. فحص dmesg للأخطاء ===
dmesg --level=err,warn | grep -i "clk\|clock" | tail -20

# === 6. فحص الـ EPROBE_DEFER ===
dmesg | grep -c "EPROBE_DEFER"   # عدد المرات
# لو كبيرة → provider مش بيتسجل أبداً

# === 7. شوف الـ consumers لكل clock ===
grep -r "" /sys/kernel/debug/clk/*/clk_prepare_count 2>/dev/null | \
  awk -F: '$2 > 0 {print $1, $2}' | sed 's|/sys/kernel/debug/clk/||'

# === 8. DT فحص clock-names ===
for node in $(find /proc/device-tree -name "clock-names" 2>/dev/null); do
    echo "=== $node ==="
    cat "$node" | tr '\0' '\n'
done

# === 9. كل الـ clocks اللي enable_count > 0 ===
awk 'NR>2 && $3 > 0 {print $1, "enable_count=" $3, "rate=" $5}' \
  /sys/kernel/debug/clk/clk_summary

# === 10. تفعيل كل الـ clk tracepoints دفعة واحدة ===
find /sys/kernel/debug/tracing/events/clk -name enable | \
  xargs -I{} sh -c 'echo 1 > {}'
```

#### مثال تفسير output

```
# output من clk_summary لو bulk_get فشلت:
                             enable  prepare  protect
   clock                      count    count    count        rate
-----------------------------------------------------------------
 xtal_24m                         1        1        0    24000000
  pll_sys                         0        0        0           0   <-- مش enabled
   apb_clk                        0        0        0           0   <-- تبع
```

لو `pll_sys` فيها `enable_count = 0` بعد ما الـ driver اشتغل → يعني `clk_bulk_enable` فشلت أو ما اتنادتش. راجع الـ error code من `clk_bulk_enable` في الـ driver.

```
# ftrace output لـ bulk_prepare ناجحة على 3 clocks:
<driver>-456  ....  clk_prepare: pll_sys
<driver>-456  ....  clk_prepare: apb_clk
<driver>-456  ....  clk_prepare: ahb_clk
<driver>-456  ....  clk_enable: pll_sys
<driver>-456  ....  clk_enable: apb_clk
<driver>-456  ....  clk_enable: ahb_clk
```

الترتيب في `clk_bulk_prepare` بيكون index 0 → N (forward)، بس في `clk_bulk_unprepare` بيكون N → 0 (reverse) — ده by design في الكود لضمان الـ cleanup الصحيح.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: Industrial Gateway على RK3562 — Driver بيفشل في probe بسبب missing clock-names

#### العنوان
**الـ SPI driver بيرجع `-EPROBE_DEFER` بشكل لا نهائي على gateway صناعي**

#### السياق
شركة بتبني industrial gateway بتستخدم RK3562 لجمع بيانات من حساسات عبر SPI. الـ driver المسؤول عن الـ SPI controller بيستخدم `clk_bulk_get()` لجلب 3 clocks: `spi`, `apb_pclk`, `spi_clk`. الـ product في مرحلة bring-up وفي أول boot الـ driver مش بيعدي probe خالص.

#### المشكلة
الـ kernel log بيظهر:

```
spi-rockchip fe610000.spi: Failed to get clk 'apb_pclk'
spi-rockchip fe610000.spi: error -ENOENT: Failed to get clk 'apb_pclk'
```

الـ driver عالق في `-ENOENT` بدل `-EPROBE_DEFER`، يعني الـ clock مش موجود أصلاً في الـ DT.

#### التحليل
الـ `clk_bulk_get()` في `clk-bulk.c` بتنادي `__clk_bulk_get()` مع `optional = false`:

```c
/* clk-bulk.c السطر 114 */
int __must_check clk_bulk_get(struct device *dev, int num_clks,
                              struct clk_bulk_data *clks)
{
    return __clk_bulk_get(dev, num_clks, clks, false);
}
```

جوه `__clk_bulk_get()`:

```c
/* السطر 92-103 */
clks[i].clk = clk_get(dev, clks[i].id);
if (IS_ERR(clks[i].clk)) {
    ret = PTR_ERR(clks[i].clk);
    clks[i].clk = NULL;

    if (ret == -ENOENT && optional)
        continue;   /* مش هيتنفذ لأن optional=false */

    dev_err_probe(dev, ret,
                  "Failed to get clk '%s'\n",
                  clks[i].id);
    goto err;       /* بيروح هنا مباشرة */
}
```

لما `optional = false` والـ `ret == -ENOENT`، الـ code بيطلع بـ error فوراً. الـ `dev_err_probe` بيطبع الـ error لأن `-ENOENT` مش `-EPROBE_DEFER`، فمحدش بيعيد المحاولة.

في الـ `err` label:
```c
/* السطر 108-111 */
err:
    clk_bulk_put(i, clks);  /* بيرجع كل الـ clocks اللي اتجمعت قبل */
    return ret;             /* بيرجع -ENOENT */
```

`clk_bulk_put()` بيلف بـ decrement loop:
```c
/* السطر 72-78 */
void clk_bulk_put(int num_clks, struct clk_bulk_data *clks)
{
    while (--num_clks >= 0) {
        clk_put(clks[num_clks].clk);
        clks[num_clks].clk = NULL;
    }
}
```

فالـ clock الأولانية اللي اتجمعت (`spi`) اترجعت تمام.

#### الحل
**تحقق من الـ DTS:**

```bash
grep -A 20 "fe610000" arch/arm64/boot/dts/rockchip/rk3562.dtsi
```

الـ node كان:
```dts
/* خطأ — clock-names ناقصة */
&spi0 {
    clocks = <&cru CLK_SPI0>, <&cru PCLK_SPI0>;
    /* clock-names مش موجودة! */
};
```

**الصح:**
```dts
&spi0 {
    clocks = <&cru CLK_SPI0>, <&cru PCLK_SPI0>, <&cru CLK_SPI0_SAMPLE>;
    clock-names = "spi", "apb_pclk", "spi_clk";
};
```

في `of_clk_bulk_get()` لو الـ `clock-names` property مش موجودة، الـ `of_property_read_string_index()` بترجع error والـ `id` بيبقى NULL، فالـ `clk_get(dev, NULL)` بيرجع `-ENOENT`.

**تأكيد الحل:**
```bash
cat /sys/kernel/debug/clk/clk_summary | grep spi
```

#### الدرس المستفاد
الـ `clk_bulk_get()` مش optional — أي clock مش موجود بيوقف الـ probe كاملاً. لو الـ clock اختياري استخدم `clk_bulk_get_optional()`. دايماً تأكد إن عدد الـ entries في `clocks` و `clock-names` متطابق في الـ DTS.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI بيتقفل بعد suspend/resume

#### العنوان
**الـ HDMI controller بيوقف عن الشغل بعد أول suspend على H616 TV box**

#### السياق
TV box بيستخدم Allwinner H616 مع Android. الـ HDMI driver بيستخدم `clk_bulk_prepare()` و `clk_bulk_enable()` لتشغيل 4 clocks: `tmds`, `pll`, `bus`, `mod`. بعد أول sleep/wake cycle، الشاشة بتتقفل.

#### المشكلة
الـ kernel log بعد resume بيظهر:

```
sunxi-hdmi display-engine: Failed to prepare clk 'pll': -5
```

الـ `-5` هو `-EIO` — الـ PLL مش بيتقفل.

#### التحليل
الـ `clk_bulk_prepare()` بيلف على الـ clocks بالترتيب:

```c
/* clk-bulk.c السطر 176-197 */
int __must_check clk_bulk_prepare(int num_clks,
                                  const struct clk_bulk_data *clks)
{
    int ret;
    int i;

    for (i = 0; i < num_clks; i++) {
        ret = clk_prepare(clks[i].clk);
        if (ret) {
            pr_err("Failed to prepare clk '%s': %d\n",
                    clks[i].id, ret);
            goto err;
        }
    }

    return 0;

err:
    clk_bulk_unprepare(i, clks);  /* بيعمل rollback للـ clocks اللي اتعملتلها prepare */
    return ret;
}
```

الـ `clk_bulk_unprepare()` بيعمل rollback بالعكس:

```c
/* السطر 161-165 */
void clk_bulk_unprepare(int num_clks, const struct clk_bulk_data *clks)
{
    while (--num_clks >= 0)
        clk_unprepare(clks[num_clks].clk);
}
```

المشكلة: الـ driver في الـ suspend handler بينادي `clk_bulk_disable()` بس مش `clk_bulk_unprepare()`. الـ `clk_bulk_disable()` بيعمل gate للـ clocks:

```c
/* السطر 211-216 */
void clk_bulk_disable(int num_clks, const struct clk_bulk_data *clks)
{
    while (--num_clks >= 0)
        clk_disable(clks[num_clks].clk);
}
```

لكن الـ PLL clock لما بيتعمله `clk_disable` من غير `clk_unprepare`، الـ reference count بتاعه في prepared state بيفضل موجود. في الـ H616، الـ PLL بيحتاج reset كامل بعد الـ deep sleep، والـ driver مش بيعمله `clk_unprepare` فالـ PLL بيحاول يرجع من state غلط.

#### الحل
تصحيح الـ suspend/resume sequence في الـ HDMI driver:

```c
/* في الـ suspend handler */
static int sunxi_hdmi_suspend(struct device *dev)
{
    struct sunxi_hdmi *hdmi = dev_get_drvdata(dev);

    clk_bulk_disable(HDMI_NUM_CLKS, hdmi->clks);
    clk_bulk_unprepare(HDMI_NUM_CLKS, hdmi->clks); /* ده اللي كان ناقص */
    return 0;
}

/* في الـ resume handler */
static int sunxi_hdmi_resume(struct device *dev)
{
    struct sunxi_hdmi *hdmi = dev_get_drvdata(dev);
    int ret;

    ret = clk_bulk_prepare(HDMI_NUM_CLKS, hdmi->clks);
    if (ret)
        return ret;

    ret = clk_bulk_enable(HDMI_NUM_CLKS, hdmi->clks);
    if (ret) {
        clk_bulk_unprepare(HDMI_NUM_CLKS, hdmi->clks);
        return ret;
    }

    return 0;
}
```

**debug command:**
```bash
# تحقق من prepare count قبل وبعد suspend
cat /sys/kernel/debug/clk/pll_hdmi/clk_prepare_count
```

#### الدرس المستفاد
الـ `clk_bulk_disable` و `clk_bulk_unprepare` ليسوا interchangeable. الـ comment في الكود صريح: "clk_bulk_disable must be called before clk_bulk_unprepare". الـ suspend يحتاج الاتنين بالترتيب الصح، والـ resume يحتاج prepare قبل enable.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — Memory leak في error path

#### العنوان
**الـ I2C multi-sensor driver بيسبب memory leak عند فشل initialization على STM32MP151**

#### السياق
IoT edge node بيستخدم STM32MP151 لقراءة بيانات من 8 حساسات عبر I2C. الـ driver بيستخدم `clk_bulk_get_all()` لجلب كل الـ clocks بشكل ديناميكي من الـ DT بدون تحديد الأعداد مسبقاً. بعد أسبوعين من الـ production، الـ system بدأ يشتكي من نقص memory.

#### المشكلة
كل مرة الـ driver بيفشل في initialize حساس معين، الـ memory بتزيد بمقدار ثابت. الـ `kmemleak` report بيظهر:

```
unreferenced object 0xc3a20000 (size 64):
  comm "modprobe", pid 412
  backtrace:
    kmalloc_array+0x...
    of_clk_bulk_get_all+0x...
    clk_bulk_get_all+0x...
    my_sensor_probe+0x...
```

#### التحليل
الـ `clk_bulk_get_all()` بينادي `of_clk_bulk_get_all()`:

```c
/* clk-bulk.c السطر 139-149 */
int __must_check clk_bulk_get_all(struct device *dev,
                                  struct clk_bulk_data **clks)
{
    struct device_node *np = dev_of_node(dev);

    if (!np)
        return 0;

    return of_clk_bulk_get_all(np, clks);
}
```

الـ `of_clk_bulk_get_all()` بتعمل allocate للـ array:

```c
/* السطر 46-70 */
static int __must_check of_clk_bulk_get_all(struct device_node *np,
                                            struct clk_bulk_data **clks)
{
    struct clk_bulk_data *clk_bulk;
    int num_clks;
    int ret;

    num_clks = of_clk_get_parent_count(np);
    if (!num_clks)
        return 0;

    clk_bulk = kmalloc_array(num_clks, sizeof(*clk_bulk), GFP_KERNEL);
    if (!clk_bulk)
        return -ENOMEM;

    ret = of_clk_bulk_get(np, num_clks, clk_bulk);
    if (ret) {
        kfree(clk_bulk);  /* هنا بيتحرر لو فشل of_clk_bulk_get */
        return ret;
    }

    *clks = clk_bulk;  /* بيعطي الـ pointer للـ caller */
    return num_clks;
}
```

الـ function نفسها صح — بتحرر الـ memory لو `of_clk_bulk_get` فشل. المشكلة في الـ driver نفسه:

```c
/* الكود الغلط في الـ driver */
static int my_sensor_probe(struct platform_device *pdev)
{
    struct clk_bulk_data *clks;
    int num_clks;

    num_clks = clk_bulk_get_all(&pdev->dev, &clks);
    if (num_clks < 0)
        return num_clks;

    /* ... initialization code ... */

    if (some_init_failed) {
        clk_bulk_put(num_clks, clks);  /* بيرجع الـ clocks بس مش الـ array */
        return -EIO;
        /* clks نفسها لسه allocated! */
    }
}
```

الـ driver بينادي `clk_bulk_put()` بس مش `clk_bulk_put_all()`. الفرق في:

```c
/* السطر 128-136 */
void clk_bulk_put_all(int num_clks, struct clk_bulk_data *clks)
{
    if (IS_ERR_OR_NULL(clks))
        return;

    clk_bulk_put(num_clks, clks);

    kfree(clks);  /* ده اللي بيحرر الـ array نفسها */
}
```

#### الحل
تغيير الـ driver لاستخدام `clk_bulk_put_all()`:

```c
static int my_sensor_probe(struct platform_device *pdev)
{
    struct clk_bulk_data *clks;
    int num_clks;

    num_clks = clk_bulk_get_all(&pdev->dev, &clks);
    if (num_clks < 0)
        return num_clks;

    if (some_init_failed) {
        clk_bulk_put_all(num_clks, clks); /* يحرر الـ clocks والـ array */
        return -EIO;
    }
}
```

**تأكيد:**
```bash
# شغّل kmemleak scan
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

#### الدرس المستفاد
القاعدة ثابتة: `clk_bulk_get_all()` pair بتاعتها `clk_bulk_put_all()` — اللي بتعمل `kfree` للـ array. `clk_bulk_put()` بس بترجع الـ clocks من غير ما تحرر الـ array الديناميكية. استخدم `clk_bulk_get_all` / `clk_bulk_put_all` دايماً مع بعض.

---

### السيناريو 4: Automotive ECU على i.MX8QM — Race condition في multi-core boot

#### العنوان
**الـ CAN controller بيتعطل عشوائياً أثناء الـ boot على i.MX8QM ECU**

#### السياق
ECU للسيارات بيستخدم i.MX8QM مع RTOS على Cortex-M4 وLinux على Cortex-A53. الـ CAN driver في Linux بيستخدم `clk_bulk_enable()` لتشغيل 3 clocks: `ipg`, `per`, `scfw`. في بعض الـ boots (تقريباً 1 من كل 20)، الـ CAN driver بيفشل.

#### المشكلة
```
flexcan 5a8d0000.can: Failed to enable clk 'scfw': -110
```

الـ `-110` هو `-ETIMEDOUT` — الـ clock مش بيتشغل في الوقت المطلوب.

#### التحليل
الـ `clk_bulk_enable()` بيلف بالترتيب:

```c
/* clk-bulk.c السطر 227-248 */
int __must_check clk_bulk_enable(int num_clks, const struct clk_bulk_data *clks)
{
    int ret;
    int i;

    for (i = 0; i < num_clks; i++) {
        ret = clk_enable(clks[i].clk);
        if (ret) {
            pr_err("Failed to enable clk '%s': %d\n",
                    clks[i].id, ret);
            goto err;
        }
    }

    return 0;

err:
    clk_bulk_disable(i, clks);  /* rollback للـ clocks اللي اتشغلت */
    return ret;
}
```

لما `clk_enable("scfw")` بيفشل بـ `-ETIMEDOUT`، الـ `clk_bulk_disable()` بيعمل rollback للـ `ipg` و `per`. لكن المشكلة الأعمق: الـ `scfw` clock بيتحكم فيه الـ System Controller Firmware اللي شغال على Cortex-M0. لما الـ M4 بيكون لسه بيعمل boot ومشغول الـ SCU interface، الـ Linux request بياخد timeout.

المشكلة مش في `clk-bulk.c` نفسها — الـ rollback صح. المشكلة إن الـ driver مش بيتعامل مع `-EPROBE_DEFER` من الـ SCFW clock controller.

الـ flow هو:
1. `clk_bulk_enable("ipg")` — نجح
2. `clk_bulk_enable("per")` — نجح
3. `clk_bulk_enable("scfw")` — `-ETIMEDOUT` لأن SCU مشغول
4. `clk_bulk_disable(2, clks)` — بيعمل disable لـ `per` وبعدين `ipg` (بالعكس)

الـ rollback صح، لكن الـ driver بيرجع error نهائي بدل ما ينتظر.

#### الحل
تعديل الـ SCFW clock driver ليرجع `-EPROBE_DEFER` بدل `-ETIMEDOUT` لما الـ SCU مشغول، وتعديل الـ CAN driver ليتعامل مع الـ defer:

```c
/* في الـ CAN driver probe */
static int flexcan_probe(struct platform_device *pdev)
{
    int ret;

    ret = clk_bulk_prepare_enable(FLEXCAN_NUM_CLKS, priv->clks);
    if (ret == -EPROBE_DEFER)
        return -EPROBE_DEFER;  /* الـ kernel هيعيد المحاولة */
    if (ret)
        return ret;
    /* ... */
}
```

وفي الـ SCFW clock driver:
```c
/* لو SCU مشغول، رجّع EPROBE_DEFER مش ETIMEDOUT */
if (scfw_busy)
    return -EPROBE_DEFER;
```

**debug:**
```bash
# تحقق من الـ boot timing
dmesg | grep -E "flexcan|scfw|SCFW" | head -20
# شوف الـ deferred probes
cat /sys/kernel/debug/devices_deferred
```

#### الدرس المستفاد
الـ `clk_bulk_enable()` بيعمل rollback صح عند أي error، لكن الـ error code مهم جداً. الـ clock controllers في systems زي i.MX8 اللي بيعتمد على external firmware (SCFW) لازم يرجعوا `-EPROBE_DEFER` لما الـ firmware مش جاهز، مش `-ETIMEDOUT`، عشان الـ probe mechanism يعمل retry تلقائي.

---

### السيناريو 5: Custom Board Bring-up على AM62x — Driver بيشتغل على Mainline بس مش على BSP Kernel

#### العنوان
**الـ USB driver بياخد clocks تمام على mainline kernel بس بيفشل على BSP مع `-ENOENT`**

#### السياق
مهندس بيعمل bring-up لـ custom industrial board مبنية على TI AM62x. الـ USB driver شغال تمام على mainline kernel 6.6، لكن لما انتقل لـ TI BSP kernel 5.10، الـ driver بدأ يفشل في probe. الـ driver بيستخدم `clk_bulk_get_optional()`.

#### المشكلة
```
dwc3 31000000.usb: Failed to get clk 'ref': -2
```

الـ `-2` هو `-ENOENT`. بس الـ DTS نفسه وبما إن `clk_bulk_get_optional()` المفروض يتجاهل `ENOENT`!

#### التحليل
الـ `clk_bulk_get_optional()` بتعمل:

```c
/* clk-bulk.c السطر 121-126 */
int __must_check clk_bulk_get_optional(struct device *dev, int num_clks,
                                       struct clk_bulk_data *clks)
{
    return __clk_bulk_get(dev, num_clks, clks, true);
}
```

في `__clk_bulk_get()` مع `optional = true`:

```c
/* السطر 90-104 */
for (i = 0; i < num_clks; i++) {
    clks[i].clk = clk_get(dev, clks[i].id);
    if (IS_ERR(clks[i].clk)) {
        ret = PTR_ERR(clks[i].clk);
        clks[i].clk = NULL;

        if (ret == -ENOENT && optional)
            continue;   /* المفروض يعدي هنا ويكمل */

        dev_err_probe(dev, ret, "Failed to get clk '%s'\n", clks[i].id);
        goto err;
    }
}
```

المفروض لو `optional = true` والـ clock مش موجود، بيكمل. بس الـ log بيقول إنه اتطبع! ده معناه إن `ret` مش `-ENOENT` — يعني الـ `continue` مش بيتنفذ.

التحقيق: في BSP kernel 5.10، الـ `clk_get()` لما الـ clock غير موجود بيرجع `-EINVAL` مش `-ENOENT`، لأن الـ BSP clock driver القديم كان بيعمل mapping مختلف. الـ check في `__clk_bulk_get()` هو:

```c
if (ret == -ENOENT && optional)
    continue;
```

الـ check بيتحقق من `-ENOENT` بالضبط، فـ `-EINVAL` بيعدي من غير `continue` وبيروح `err`.

**تأكيد التشخيص:**
```bash
# على BSP kernel
dmesg | grep "Failed to get clk 'ref'"
# شوف الـ error number بالضبط
strace -e trace=ioctl modprobe dwc3 2>&1 | grep EINVAL

# تحقق من الـ clock في debugfs
ls /sys/kernel/debug/clk/ | grep ref
cat /sys/kernel/debug/clk/ref/clk_enable_count
```

#### الحل

**الحل الأول**: backport fix من mainline — تغيير الـ BSP clock driver ليرجع `-ENOENT` بدل `-EINVAL` لما الـ clock مش موجود في الـ DT.

**الحل الثاني**: إضافة الـ clock للـ DTS صح:

```dts
/* custom board DTS */
&dwc3_0 {
    clocks = <&k3_clks 151 0>,   /* ref */
             <&k3_clks 151 2>,   /* suspend */
             <&k3_clks 151 1>;   /* bus */
    clock-names = "ref", "suspend", "bus";
    assigned-clocks = <&k3_clks 151 0>;
    assigned-clock-rates = <19200000>;
};
```

**الحل الثالث**: لو الـ clock اختياري فعلاً، في بعض الـ BSP kernels القديمة الحل هو check صريح:

```c
/* في الـ driver نفسه قبل clk_bulk_get_optional */
if (!of_property_match_string(dev->of_node, "clock-names", "ref"))
    /* ref موجود في DT */
```

**تأكيد:**
```bash
# بعد الـ fix
modprobe dwc3
dmesg | tail -20
lsusb  # المفروض يشوف الـ USB devices
```

#### الدرس المستفاد
الـ `clk_bulk_get_optional()` مش "magic" — هو بس بيتجاهل `-ENOENT` بالضبط. لو الـ BSP clock driver بيرجع error code مختلف لنفس الـ condition، الـ optional check هيفشل. عند الانتقال بين kernel versions أو من mainline لـ BSP، دايماً تأكد من الـ error codes اللي بيرجعها الـ clock framework، مش بس من وجود الـ function نفسها.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ LWN.net** هو أهم مصدر لتوثيق تطور الـ Linux kernel — الروابط دي بتغطي نشأة الـ Common Clock Framework والـ bulk API اللي `clk-bulk.c` جزء منه.

| المقال | الوصف |
|--------|--------|
| [A common clock framework](https://lwn.net/Articles/472998/) | المقال الأصلي اللي وصف الـ framework لما Mike Turquette قدّمه — نقطة البداية لفهم ليه `clk_bulk` موجود |
| [common clk framework (discussion)](https://lwn.net/Articles/486841/) | نقاش تاني على LWN بيغطي تفاصيل implementation وقت الـ merge |
| [Documentation/clk.txt (LWN mirror)](https://lwn.net/Articles/489668/) | نسخة من الـ documentation الرسمي عشان تشوف التطور التاريخي للـ API |

---

### التوثيق الرسمي للـ Kernel

**الـ kernel documentation** المباشر هو المرجع الأدق دايمًا:

- **`Documentation/driver-api/clk.rst`** — الشرح الكامل للـ Common CLK Framework، بيشرح `struct clk_hw`، الـ ops، وكيفية استخدام الـ bulk API.
  - Online: [docs.kernel.org/driver-api/clk.html](https://docs.kernel.org/driver-api/clk.html)
  - قديم (txt): [kernel.org/doc/Documentation/clk.txt](https://www.kernel.org/doc/Documentation/clk.txt)

- **`include/linux/clk.h`** — الـ header الرئيسي اللي بيعرّف `clk_bulk_data`، `clk_bulk_get`، `clk_bulk_enable`، إلخ — القراءة الإجبارية قبل أي استخدام.

- **`include/linux/clk-provider.h`** — الـ header الخاص بكُتّاب الـ drivers اللي بيعملوا clock nodes جديدة (مش consumers).

- **`drivers/clk/clk.c`** — الـ implementation الرئيسي للـ framework اللي `clk-bulk.c` بيستدعي functions منه زي `clk_get`، `clk_enable`، إلخ.

---

### Commits المهمة

**الـ patch الأصلي** اللي أضاف `of_clk_bulk_get` و`clk-bulk.c` من NXP:

- [V4: clk: bulk: add of_clk_bulk_get() — Patchwork](https://patchwork.kernel.org/project/linux-clk/patch/1506415441-4435-1-git-send-email-aisheng.dong@nxp.com/)
  - المؤلف: Dong Aisheng (NXP)، سنة 2017
  - بيشرح السبب: كانوا محتاجين `bulk` API تشتغل من `device_node` مباشرةً من غير `struct device`

```bash
# عشان تتتبع تاريخ الملف من جوه الـ repo
git log --follow --oneline drivers/clk/clk-bulk.c
```

---

### نقاشات Mailing List

- **[LKML / linux-clk] clk: bulk: export of_clk_bulk_get_all** — نقاش على spinics:
  [spinics.net/lists/kernel/msg4186831.html](https://www.spinics.net/lists/kernel/msg4186831.html)
  بيناقش تصدير `of_clk_bulk_get_all` للاستخدام خارج `clk-bulk.c`

- **lore.kernel.org** — الأرشيف الرسمي لكل patches الـ linux-clk:
  [lore.kernel.org/linux-clk](https://lore.kernel.org/linux-clk/)
  ابحث بـ: `clk_bulk` أو `clk-bulk.c`

---

### eLinux.org

- [Linux clock management framework (ELC 2007 PDF)](https://elinux.org/images/a/a3/ELC_2007_Linux_clock_fmw.pdf)
  — عرض قديم بس مهم يوضّح المشكلة اللي الـ Common CLK Framework جه يحلها

- [Power Management Specification — eLinux.org](https://elinux.org/Power_Management_Specification)
  — بيربط الـ clock gating بالـ power management، وده context مهم لفهم ليه `clk_bulk_disable` ضروري قبل `clk_bulk_unprepare`

---

### kernelnewbies.org

**الـ kernelnewbies** بيوثّق التغييرات في كل إصدار — ابحث في الصفحات دي عن bulk clock changes:

- [Linux_6.7 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.7) — فيه تحديثات على الـ clk subsystem
- [Linux_5.9 — kernelnewbies.org](https://kernelnewbies.org/Linux_5.9) — يحتوي على تغييرات في الـ clock bulk API
- [Linux_6.12 — kernelnewbies.org](https://kernelnewbies.org/Linux_6.12) — آخر تحديثات مهمة في الـ clk framework

---

### كتب مقترحة

| الكتاب | الفصل / الموضوع |
|--------|-----------------|
| **Linux Device Drivers, 3rd Ed. (LDD3)** — Rubini, Corbet, Kroah-Hartman | Chapter 14: The Linux Device Model — بيشرح `struct device`، `device_node`، والـ resource management اللي `clk_bulk_get` بيعتمد عليه |
| **Linux Kernel Development — Robert Love** | Chapter 17: Devices and Modules — context الـ driver model |
| **Embedded Linux Primer — Christopher Hallinan** | Chapter 15: Debugging Embedded Linux Applications + الفصول عن power management في embedded SoCs — مثالي لأن `clk-bulk.c` غالبًا بيُستخدم في ARM SoC drivers |
| **Professional Linux Kernel Architecture — Wolfgang Mauerer** | الفصل الخاص بـ power management وإدارة الـ resources |

---

### مصطلحات البحث

لو عايز تعمق أكتر، استخدم المصطلحات دي:

```
clk_bulk_get linux kernel
devm_clk_bulk_get_all device tree
linux common clock framework CONFIG_COMMON_CLK
clk_prepare_enable vs clk_bulk_prepare_enable
of_clk_get device_node clock-names
linux clock gating power management driver
```

---

### ملخص سريع للمصادر

```
الأهم أولًا:
1. docs.kernel.org/driver-api/clk.html        ← الـ reference الرسمي
2. lwn.net/Articles/472998/                   ← أصل الـ framework
3. include/linux/clk.h                        ← كل الـ API تعريفاتها هنا
4. patchwork.kernel.org (NXP patch 2017)      ← سبب وجود clk-bulk.c
5. lore.kernel.org/linux-clk/                 ← نقاشات الـ mailing list
```
## Phase 8: Writing simple module

### الفكرة

**`clk_bulk_enable`** هي من أكتر الفانكشنز إثارة في الملف — بتعمل enable لمجموعة clocks دفعة واحدة. هنعمل kprobe عليها علشان نشوف كل مرة بيتعمل فيها enable لـ bulk clocks ونطبع عدد الـ clocks اللي اتعملت enable.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_clk_bulk.c
 * Hook clk_bulk_enable() to log every bulk clock enable call.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* kprobe API */
#include <linux/clk.h>         /* struct clk_bulk_data */

/* ───── kprobe callback ───── */

/*
 * pre_handler fires just BEFORE clk_bulk_enable() executes.
 * pt_regs holds the CPU registers at the probe site:
 *   - regs->di = first argument  (num_clks, int)
 *   - regs->si = second argument (clks pointer, struct clk_bulk_data *)
 * This is x86-64 calling convention (System V AMD64 ABI).
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    int num_clks = (int)regs->di;
    struct clk_bulk_data *clks = (struct clk_bulk_data *)regs->si;
    int i;

    pr_info("[clk_bulk_probe] clk_bulk_enable called: num_clks=%d\n",
            num_clks);

    /* Print the ID of each clock being enabled — safe read-only access */
    for (i = 0; i < num_clks && clks; i++) {
        if (clks[i].id)
            pr_info("[clk_bulk_probe]   clk[%d] = '%s'\n", i, clks[i].id);
        else
            pr_info("[clk_bulk_probe]   clk[%d] = (no id)\n", i);
    }

    return 0; /* 0 = let the original function continue */
}

/* ───── kprobe struct ───── */

/*
 * symbol_name tells the kernel which function to intercept.
 * pre_handler is our callback that runs before the real function.
 */
static struct kprobe kp = {
    .symbol_name = "clk_bulk_enable",
    .pre_handler = handler_pre,
};

/* ───── module init ───── */

static int __init clk_bulk_probe_init(void)
{
    int ret;

    /*
     * register_kprobe() looks up the symbol, patches the first byte
     * of clk_bulk_enable with a breakpoint (int3 on x86), and wires
     * our handler into the breakpoint path.
     */
    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[clk_bulk_probe] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[clk_bulk_probe] planted on clk_bulk_enable @ %p\n", kp.addr);
    return 0;
}

/* ───── module exit ───── */

static void __exit clk_bulk_probe_exit(void)
{
    /*
     * unregister_kprobe() removes the int3 patch and restores the
     * original bytes — leaving clk_bulk_enable clean after rmmod.
     * Without this the kernel would jump to freed module memory → crash.
     */
    unregister_kprobe(&kp);
    pr_info("[clk_bulk_probe] removed from clk_bulk_enable\n");
}

module_init(clk_bulk_probe_init);
module_exit(clk_bulk_probe_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("kernel-doc-example");
MODULE_DESCRIPTION("kprobe on clk_bulk_enable to log bulk clock enable calls");
```

---

### شرح كل جزء

#### الـ Includes

| Header | ليه موجود |
|--------|-----------|
| `linux/module.h` | الـ macros الأساسية زي `module_init` و `MODULE_LICENSE` |
| `linux/kernel.h` | `pr_info` و `pr_err` |
| `linux/kprobes.h` | `struct kprobe` و `register_kprobe` / `unregister_kprobe` |
| `linux/clk.h` | `struct clk_bulk_data` علشان نقرأ الـ `id` من الـ array |

---

#### الـ `handler_pre` — الـ callback

**الـ `pt_regs *regs`** هو snapshot للـ CPU registers لحظة ضرب الـ kprobe — على x86-64 الأرغيومنت الأول يجي في `rdi` والتاني في `rsi`، فمن غير ما نلمس الـ stack بنقدر نقرأ الـ arguments مباشرة.

بنعمل loop على الـ `clk_bulk_data` array ونطبع الـ `id` لكل clock — الـ `id` ده string اتحط فيها في `__clk_bulk_get` من اسم الـ clock في الـ device tree.

---

#### الـ `struct kprobe`

الـ `symbol_name = "clk_bulk_enable"` بيقول للكيرنل هوك على الفانكشن دي بالاسم — الكيرنل بيحوّل الاسم لـ address وقت الـ `register_kprobe`. محتاجين `EXPORT_SYMBOL_GPL` موجود على الفانكشن (وفعلاً موجود في `clk-bulk.c`) علشان الـ kprobe يلاقي الـ symbol.

---

#### الـ `module_init` / `module_exit`

**`register_kprobe`** بيحط `int3` (breakpoint) في أول الفانكشن — الكيرنل يشغّل هاندلرنا لما أي حد يكال `clk_bulk_enable`. **`unregister_kprobe`** في الـ exit ضروري علشان يشيل الـ breakpoint ويرجّع البايت الأصلي قبل ما الموديول يتحذف من الميموري، لو متعملتش ده الكيرنل هيعمل crash لما يحاول ينفذ الكود اللي اتمسح.

---

### Makefile للتجربة

```makefile
obj-m += kprobe_clk_bulk.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

```bash
# تحميل الموديول
sudo insmod kprobe_clk_bulk.ko

# متابعة اللوج
sudo dmesg -w | grep clk_bulk_probe

# تفريغ الموديول
sudo rmmod kprobe_clk_bulk
```
