## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

**الـ** `pm_clock.h` جزء من **Power Management Core** subsystem في Linux kernel، وبالتحديد من الـ **PM Clock framework** — وده موجود تحت:
- `MAINTAINERS` → `POWER MANAGEMENT CORE` → المسؤول: Rafael J. Wysocki
- الـ files بتاعته: `include/linux/pm_*.h` + `drivers/base/power/`

---

### القصة من البداية — ليه الـ PM Clock موجود أصلاً؟

تخيل عندك جهاز زي USB controller أو camera sensor جوه الـ SoC (System on Chip). الـ SoC دي فيها تقريباً عشرات أو مئات من الـ **clocks** — كل clock بيدي إشارة نبضات (pulse) لجزء معين من الـ chip عشان يشتغل.

المشكلة الكبيرة: لو الـ device مش شغال دلوقتي (مفيش data بيتنقل مثلاً)، الـ clock بتاعه لازم **يتقفل** عشان توفر طاقة. ولما الـ device يصحى تاني، الـ clock لازم **يتفتح** تاني قبل ما يشتغل.

ده بيحصل مئات المرات في الثانية الواحدة على أي هاتف أو laptop. لو كل driver driver كتب الكود ده لوحده، كنا هنشوف:
- كود متكرر في كل مكان
- bugs لأن حد نسي يقفل clock
- صعوبة في التوحيد والصيانة

**الحل:** الـ PM Clock framework — طبقة مركزية بتعمل إدارة الـ clocks وربطها بدورة حياة الـ device (suspend / resume) بشكل تلقائي.

---

### الـ `pm_clock.h` بيعمل إيه بالظبط؟

الملف ده هو **واجهة (API header)** للـ PM Clock framework. بيوفر:

#### 1. إدارة قائمة الـ clocks لكل device

كل device ممكن يحتاج أكتر من clock واحد. الـ framework بيخلي كل device عنده **linked list** من الـ clocks المرتبطة بيه:

```c
/* تهيئة قائمة الـ clocks لـ device معين */
extern void pm_clk_init(struct device *dev);

/* إنشاء البنية التحتية لإدارة الـ clocks */
extern int pm_clk_create(struct device *dev);

/* تحرير كل الـ clocks وحذف القائمة */
extern void pm_clk_destroy(struct device *dev);
```

#### 2. إضافة وحذف الـ clocks

```c
/* إضافة clock بالاسم (con_id = connection ID) */
extern int pm_clk_add(struct device *dev, const char *con_id);

/* إضافة clock بمؤشر مباشر */
extern int pm_clk_add_clk(struct device *dev, struct clk *clk);

/* إضافة كل الـ clocks المذكورة في Device Tree تلقائياً */
extern int of_pm_clk_add_clks(struct device *dev);

/* حذف clock معين */
extern void pm_clk_remove_clk(struct device *dev, struct clk *clk);
```

#### 3. الـ suspend / resume الفعلي

```c
/* قفل كل الـ clocks عند النوم */
extern int pm_clk_suspend(struct device *dev);

/* فتح كل الـ clocks عند الصحيان */
extern int pm_clk_resume(struct device *dev);
```

#### 4. تكامل مع Runtime PM

```c
/* Runtime PM callbacks جاهزة للاستخدام مباشرة */
#define USE_PM_CLK_RUNTIME_OPS \
    .runtime_suspend = pm_clk_runtime_suspend, \
    .runtime_resume  = pm_clk_runtime_resume,
```

الـ macro دي بتخلي الـ driver driver يكتب سطر واحد بدل ما يكتب الـ callbacks بنفسه.

#### 5. الـ Notifier للـ buses

```c
struct pm_clk_notifier_block {
    struct notifier_block nb;      /* الـ notifier العادي */
    struct dev_pm_domain *pm_domain; /* الـ power domain */
    char *con_ids[];               /* أسماء الـ clocks */
};

extern void pm_clk_add_notifier(const struct bus_type *bus,
                                struct pm_clk_notifier_block *clknb);
```

ده بيخلي كل device جديد بيتضاف على bus معينة يحصل له إضافة الـ clocks تلقائياً — من غير ما كل driver يعمل حاجة.

---

### القصة الكاملة: رحلة الـ clock مع الـ device

```
Device يتسجل في الـ kernel
         │
         ▼
pm_clk_create()  ← تجهيز قائمة الـ clocks
         │
         ▼
pm_clk_add() / of_pm_clk_add_clks()  ← إضافة الـ clocks المطلوبة
         │
         ▼
Device يشتغل عادي (clocks شغالة)
         │
    [مفيش نشاط]
         │
         ▼
Runtime PM يقرر: suspend
         │
         ▼
pm_clk_runtime_suspend()
    └─► pm_clk_suspend()  ← قفل كل الـ clocks
         │
    [device نايم، طاقة محفوظة]
         │
    [request جديد]
         │
         ▼
pm_clk_runtime_resume()
    └─► pm_clk_resume()  ← فتح كل الـ clocks
         │
         ▼
Device يشتغل تاني
```

---

### الـ Compile-time Guards

الملف بيستخدم ثلاث Kconfig guards:

| Guard | معناه |
|-------|--------|
| `CONFIG_PM` | الـ Power Management مفعّل |
| `CONFIG_PM_CLK` | الـ PM Clock framework مفعّل |
| `CONFIG_HAVE_CLK` | الـ hardware عنده clock framework |

لو أي واحد فيهم مش مفعّل، الـ functions بتتحول لـ stubs فارغة أو بترجع `-EINVAL` — من غير ما تأثر على الـ code اللي بيستخدمها.

---

### الملفات المكونة للـ Subsystem

#### Core Files

| الملف | الدور |
|--------|-------|
| `include/linux/pm_clock.h` | **الـ header — API الرئيسي** |
| `drivers/base/power/clock_ops.c` | التنفيذ الكامل للـ framework |
| `drivers/base/power/common.c` | الـ `pm_subsys_data` اللي بيحمل قائمة الـ clocks |
| `drivers/base/power/runtime.c` | الـ Runtime PM engine |
| `drivers/base/power/main.c` | الـ system sleep (suspend-to-RAM/disk) |

#### Headers المرتبطة

| الملف | الدور |
|--------|-------|
| `include/linux/pm.h` | التعريفات الأساسية للـ PM |
| `include/linux/pm_domain.h` | الـ Power Domains (`GENPD_FLAG_PM_CLK`) |
| `include/linux/pm_runtime.h` | الـ Runtime PM API |
| `include/linux/clk.h` | الـ Clock framework API |
| `include/linux/clkdev.h` | ربط الـ clocks بالـ devices |

#### Drivers بتستخدمه

| الملف | السبب |
|--------|-------|
| `drivers/pmdomain/core.c` | الـ Generic PM Domains بتستخدمه عند `GENPD_FLAG_PM_CLK` |
| `drivers/base/power/generic_ops.c` | الـ generic PM operations |

---

### ليه مهم؟

على هاتف Android أو أي SoC حديث:
- فيه **مئات الـ clocks**
- كل clock شغال بياكل طاقة
- الـ PM Clock framework بيضمن إن الـ clock بيتقفل تلقائياً لما الـ device مش محتاجه
- من غيره كل driver كان هيكتب نفس الكود، ومفيش ضمان إن الـ clock هيتقفل صح

الـ framework ده **أساسي لتوفير البطارية** في أي Linux system.
## Phase 2: شرح الـ PM Clock Framework

### المشكلة اللي بيحلها الـ Framework ده

في أي SoC حديث — خد مثلاً Allwinner H3 أو NXP i.MX8 — كل peripheral معاه clock أو أكتر مربوطة بيه.
الـ UART معاه `apb_clk`، الـ USB معاه `usb_clk`، الـ I2C معاه `i2c_clk`، وهكذا.

لما الـ driver بيعمل `runtime_suspend` للـ device، لازم يطفي الـ clock عشان يوفر power.
لما بيعمل `runtime_resume`، يفتحها تاني.

المشكلة إن كل driver كان بيعمل نفس الكود:

```c
/* كود متكرر في كل driver بيدعم clock gating */
static int my_driver_runtime_suspend(struct device *dev)
{
    struct my_priv *priv = dev_get_drvdata(dev);
    clk_disable_unprepare(priv->clk);   /* طفي الـ clock */
    return 0;
}

static int my_driver_runtime_resume(struct device *dev)
{
    struct my_priv *priv = priv = dev_get_drvdata(dev);
    return clk_prepare_enable(priv->clk); /* شغّل الـ clock */
}
```

ده boilerplate خالص. لو الـ device عنده 3 clocks؟ تكرار ثلاث مرات.
لو ال platform غير supported لـ `CONFIG_PM`؟ تحتاج `#ifdef`.
لو نسيت تطفي clock واحدة؟ power leak صامت مش هتلاقيه بسهولة.

الـ **PM Clock Framework** (`CONFIG_PM_CLK`) اتعمل عشان يمسح كل الـ boilerplate ده
ويديك generic callbacks جاهزة تربطها بالـ runtime PM بسطر واحد.

---

### الحل: Generic Clock Management على مستوى الـ Device

الفكرة بسيطة لكن قوية:

1. كل `device` عنده **قائمة** من الـ clocks المربوطة بيه (`clock_list` جوا `pm_subsys_data`).
2. الـ framework بيوفر `pm_clk_suspend()` و`pm_clk_resume()` كـ generic callbacks.
3. هم بيعملوا iterate على القايمة دي وبيطفوا أو يشغلوا كل clock تلقائياً.
4. الـ driver بس بيقولو "أنا عندي الـ clocks دي" وخلاص.

---

### Real-World Analogy: مدير فندق والغرف

تخيل فندق كبير فيه غرف كتير، كل غرفة فيها إضاءة وتكييف (= clocks).

- **الـ hotel manager** (= الـ PM Clock Framework) معاه **قايمة** بكل جهاز في كل غرفة.
- لما آخر نزيل يخرج (= `runtime_suspend`)، المدير بيطفي **كل حاجة** في الغرفة دي من قائمته — مش هيفضل ينتظر إن الـ driver يتذكر.
- لما نزيل جديد بييجي (= `runtime_resume`)، المدير يشغّل **كل حاجة** تاني.
- كل غرفة (= device) عندها **قائمة بالأجهزة بتاعتها** (= `clock_list`).
- الأودة بتعمل check-in/check-out (= `pm_clk_add` / `pm_clk_remove_clk`) على مستوى الـ connection ID (= `con_id`).
- مفيش سائس (= driver code) بيفضل يشغّل/يطفي كل جهاز بإيده — المدير بيعمل ده أوتوماتيك.

الـ mapping:

| الفندق | الـ Kernel |
|--------|-----------|
| غرفة | `struct device` |
| جهاز داخل الغرفة | `struct pm_clock_entry` |
| قايمة أجهزة الغرفة | `clock_list` داخل `pm_subsys_data` |
| تشغيل/إطفاء عند الـ check-in/out | `pm_clk_resume()` / `pm_clk_suspend()` |
| رقم تعريف الجهاز (كود الغرفة) | `con_id` (connection ID) |
| مدير الفندق | الـ PM Clock Framework نفسه |

---

### Big Picture Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Driver Layer                              │
│  my_driver_probe():                                          │
│    pm_clk_create(dev)        ← allocate pm_subsys_data      │
│    pm_clk_add(dev, "uart")   ← register clock by con_id     │
│    pm_clk_add(dev, "apb")    ← يمكن أكتر من clock           │
│                                                              │
│  dev_pm_ops = { USE_PM_CLK_RUNTIME_OPS }  ← macro magic     │
└─────────────────┬───────────────────────────────────────────┘
                  │ pm_clk_runtime_suspend/resume()
                  ▼
┌─────────────────────────────────────────────────────────────┐
│              PM Clock Framework (clock_ops.c)               │
│                                                             │
│  pm_clk_suspend(dev)                                        │
│    └─► iterate clock_list                                   │
│          └─► clk_disable() / clk_unprepare() per entry      │
│                                                             │
│  pm_clk_resume(dev)                                         │
│    └─► iterate clock_list                                   │
│          └─► clk_prepare() / clk_enable() per entry         │
└─────────────────┬───────────────────────────────────────────┘
                  │ clk_prepare_enable / clk_disable_unprepare
                  ▼
┌─────────────────────────────────────────────────────────────┐
│              Common Clock Framework (CCF)                   │
│  struct clk  ──►  struct clk_core  ──►  struct clk_ops      │
│                    (reference count,      (.enable,          │
│                     prepare count,         .disable,         │
│                     parent tree)           .prepare ...)     │
└─────────────────┬───────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────┐
│           Hardware Clock Controllers                        │
│  e.g. Allwinner CCU, NXP CCM, Qualcomm GCC                  │
│  (platform-specific register writes to gate/ungate clocks)  │
└─────────────────────────────────────────────────────────────┘
```

**ملاحظة مهمة**: الـ PM Clock Framework مش بيعرف أي حاجة عن الـ hardware.
هو بيكلم الـ **Common Clock Framework (CCF)** فقط من خلال `struct clk` API.
الـ CCF هو اللي بيعرف يتكلم مع الـ hardware.

---

### Core Data Structures

#### الـ `pm_clock_entry` — أساس كل حاجة

```c
/* داخل clock_ops.c — مش public */
struct pm_clock_entry {
    struct list_head node;          /* عقدة في قايمة clock_list */
    char *con_id;                   /* اسم الـ clock زي "uart", "apb" */
    struct clk *clk;                /* الـ handle الفعلي من CCF */
    enum pce_status status;         /* NONE / ACQUIRED / PREPARED / ENABLED / ERROR */
    bool enabled_when_prepared;     /* هل الـ clk كانت شغالة وقت الـ prepare؟ */
};
```

الـ `status` enum بيعكس lifecycle الـ clock:

```
PCE_STATUS_NONE
    │
    ▼  clk_get()
PCE_STATUS_ACQUIRED
    │
    ▼  clk_prepare()
PCE_STATUS_PREPARED
    │
    ▼  clk_enable()
PCE_STATUS_ENABLED
```

لو أي خطوة فشلت → `PCE_STATUS_ERROR`، والـ framework بيعمل skip لها في المرات الجاية.

#### الـ `pm_subsys_data` — بيت الـ clocks في الـ device

الـ framework مش بيخزن البيانات جوا الـ driver struct — بيخزنها في `dev->power.subsys_data`:

```c
/* من include/linux/pm.h و drivers/base/power/power.h */
struct pm_subsys_data {
    spinlock_t lock;
    unsigned int refcount;
    unsigned int clock_op_might_sleep;  /* عدد الـ clocks اللي clk_enable بتاعتها بتنام */
    struct mutex clock_mutex;           /* mutex لـ sleepable clock ops */
    struct list_head clock_list;        /* قايمة pm_clock_entry */
    struct pm_domain_data *domain_data;
};
```

**لماذا يوجد spinlock و mutex معاً؟**
- `pm_clk_suspend/resume` ممكن يتنادوا من atomic context (interrupt handler).
- لو كل الـ clocks سريعة (non-sleeping)، يستخدم الـ spinlock.
- لو في clock واحدة على الأقل `clk_enable` بتاعتها بتعمل sleep (زي bعض PLLs)، لازم يستخدم الـ mutex.
- الـ `clock_op_might_sleep` counter بيتتبع كام clock من النوع ده موجودة.

#### الـ `pm_clk_notifier_block` — ربط الـ Framework بالـ Bus

```c
struct pm_clk_notifier_block {
    struct notifier_block nb;       /* الـ notifier standard hook */
    struct dev_pm_domain *pm_domain;/* الـ PM domain المراد ربطه */
    char *con_ids[];                /* flexible array: أسماء الـ clocks */
};
```

ده بيُستخدم مع `pm_clk_add_notifier()` عشان تقول:
"على أي device اتضاف على الـ bus دي، ضيف الـ clocks دي أوتوماتيك وربّطه بالـ PM domain ده."

مش محتاج تعدل كل driver على حدة — بتسجّل مرة واحدة على مستوى الـ bus.

---

### الـ API كاملة مع الـ Use Cases

#### 1. إنشاء وتدمير الـ Clock List

```c
/* يحجز pm_subsys_data ويبدأ clock_list */
int pm_clk_create(struct device *dev);

/* نسخة managed — بتتنظف أوتوماتيك لما الـ device بتتحذف */
int devm_pm_clk_create(struct device *dev);

/* يحرر كل الـ clock entries و pm_subsys_data */
void pm_clk_destroy(struct device *dev);
```

#### 2. إضافة الـ Clocks

```c
/* يضيف clock بالاسم (con_id) — الـ framework يعمل clk_get بنفسه */
int pm_clk_add(struct device *dev, const char *con_id);

/* يضيف clock عندك handle ليها بالفعل */
int pm_clk_add_clk(struct device *dev, struct clk *clk);

/* يقرأ كل الـ clocks من device tree تلقائياً */
int of_pm_clk_add_clks(struct device *dev);
```

الـ `of_pm_clk_add_clks()` بيعمل parse لـ `clocks` property في الـ DTS ويضيفهم كلهم.
مثال DTS:

```dts
uart0: serial@1c28000 {
    clocks = <&ccu CLK_BUS_UART0>, <&ccu CLK_UART0>;
    clock-names = "bus", "mod";
};
```

هنا `of_pm_clk_add_clks()` هيضيف `CLK_BUS_UART0` و `CLK_UART0` للقايمة.

#### 3. الـ Suspend/Resume Callbacks

```c
/* بيطفي كل الـ clocks في القايمة */
int pm_clk_suspend(struct device *dev);

/* بيشغل كل الـ clocks في القايمة */
int pm_clk_resume(struct device *dev);

/* runtime PM variants — بيعملوا نفس الحاجة لكن للـ RPM */
int pm_clk_runtime_suspend(struct device *dev);
int pm_clk_runtime_resume(struct device *dev);
```

#### 4. الـ Convenience Macro

```c
#define USE_PM_CLK_RUNTIME_OPS \
    .runtime_suspend = pm_clk_runtime_suspend, \
    .runtime_resume  = pm_clk_runtime_resume,
```

الاستخدام في الـ driver:

```c
static const struct dev_pm_ops my_pm_ops = {
    USE_PM_CLK_RUNTIME_OPS        /* ← سطر واحد بدل دالتين */
    .suspend = my_suspend,
    .resume  = my_resume,
};
```

---

### مثال كامل: Driver على Allwinner SoC

```c
static int sunxi_uart_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    int ret;

    /* 1. ابدأ الـ clock management */
    ret = devm_pm_clk_create(dev);
    if (ret)
        return ret;

    /* 2. أضف كل الـ clocks من الـ DTS أوتوماتيك */
    ret = of_pm_clk_add_clks(dev);
    if (ret < 0)
        return ret;

    /* enable_irq, request resources, etc. */
    pm_runtime_enable(dev);
    return 0;
}

static const struct dev_pm_ops sunxi_uart_pm_ops = {
    USE_PM_CLK_RUNTIME_OPS   /* الـ framework بيتولى كل حاجة */
};

static struct platform_driver sunxi_uart_driver = {
    .probe  = sunxi_uart_probe,
    .driver = {
        .name = "sunxi-uart",
        .pm   = &sunxi_uart_pm_ops,
    },
};
```

**النتيجة**: لما الـ UART مش مستخدم، الـ runtime PM بيعمل `runtime_suspend`
← الـ framework يطفي `CLK_BUS_UART0` و `CLK_UART0` ← توفير power حقيقي
بدون ولا سطر clock management في الـ driver.

---

### ماذا يمتلك الـ Framework وماذا يفوّض؟

| المسؤولية | من يتولاها؟ |
|-----------|-------------|
| تتبع قايمة الـ clocks لكل device | **PM Clock Framework** |
| إطفاء/تشغيل الـ clocks عند الـ suspend/resume | **PM Clock Framework** |
| تحديد ترتيب إطفاء الـ clocks | **PM Clock Framework** (LIFO — عكس ترتيب الإضافة) |
| معرفة الـ clock topology (parent/child) | **CCF** (Common Clock Framework) |
| الـ register writes الفعلية على الـ hardware | **Platform clock driver** (e.g. `clk-sunxi-ccu.c`) |
| تحديد أي clocks تُضاف للـ device | **الـ driver أو الـ bus notifier** |
| إدارة الـ PM domain للـ device | **PM Domain Framework** (subsystem منفصل) |
| قرار متى يتعمل suspend/resume | **Runtime PM Framework** |

---

### علاقة الـ Framework بمفاهيم أخرى

**الـ Common Clock Framework (CCF)**: هو الـ abstraction layer للـ clocks في الـ kernel.
بيوفر `struct clk` كـ handle معزول عن الـ hardware. الـ PM Clock Framework بيكلمه
فقط من خلال `clk_prepare_enable()` و `clk_disable_unprepare()`.

**الـ Runtime PM**: هو الـ subsystem اللي بيقرر متى يتنادى `runtime_suspend/resume`.
الـ PM Clock Framework مجرد **implementation** لـ callbacks الـ Runtime PM.

**الـ PM Domain**: بيجمع devices تحت power domain واحد.
الـ `pm_clk_notifier_block` بيربط الاتنين — ممكن تقول "كل device على الـ bus ده
يتضاف تحت الـ PM domain ده وتتضاف ليه الـ clocks دي أوتوماتيك."

```
Runtime PM ──calls──► pm_clk_runtime_suspend()
                              │
                              ▼
                   PM Clock Framework
                   (iterates clock_list)
                              │
                              ▼
                   CCF: clk_disable_unprepare()
                              │
                              ▼
                   Platform Clock Driver
                   (writes to HW registers)
```
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Flags والـ Enums والـ Config Options

#### Config Options (Kconfig)

| Option | المعنى |
|--------|--------|
| `CONFIG_PM` | تفعيل runtime PM بشكل عام — بدونه `pm_clk_runtime_suspend/resume` مش موجودين |
| `CONFIG_PM_CLK` | تفعيل إدارة الـ clocks عبر PM — الجزء الأساسي للـ `pm_clock.h` |
| `CONFIG_HAVE_CLK` | الـ platform عندها clock framework — بدونه `pm_clk_add_notifier` بتبقى stub |
| `CONFIG_PM_GENERIC_DOMAINS` | تفعيل الـ generic PM domains (genpd) — لو موجود، `pm_subsys_data` بتحتوي على `domain_data` |

#### الـ Macro الرئيسي

| Macro | القيمة لما `CONFIG_PM` موجود | القيمة لما مش موجود |
|-------|------------------------------|----------------------|
| `USE_PM_CLK_RUNTIME_OPS` | `.runtime_suspend = pm_clk_runtime_suspend, .runtime_resume = pm_clk_runtime_resume,` | فاضي تماماً |

---

#### enum `pce_status` (في `clock_ops.c`)

الـ state machine لكل clock entry:

| القيمة | الرقم | المعنى |
|--------|-------|--------|
| `PCE_STATUS_NONE` | 0 | الـ entry اتعمل بس مفيش حاجة اتعملت للـ clock |
| `PCE_STATUS_ACQUIRED` | 1 | `clk_get()` نجح، ودي حالة الـ clocks اللي `enabled_when_prepared` |
| `PCE_STATUS_PREPARED` | 2 | `clk_prepare()` نجح (الـ clock جاهز بس موقفش) |
| `PCE_STATUS_ENABLED` | 3 | `clk_enable()` نجح — الـ clock شغال دلوقتي |
| `PCE_STATUS_ERROR` | 4 | فشل في الـ acquire أو الـ prepare |

---

### 1. الـ Structs الأساسية

---

#### `struct pm_clk_notifier_block`

**الغرض:** ربط الـ notifier chain بالـ PM domain وقائمة الـ clock connection IDs، عشان لما device تتضاف أو تتشال من bus معين، تتعمل/تتمسح الـ clocks بتاعتها أوتوماتيك.

```c
struct pm_clk_notifier_block {
    struct notifier_block nb;    /* embedded notifier — يتحط في bus notifier chain */
    struct dev_pm_domain *pm_domain; /* الـ PM domain اللي هيتعين للـ device */
    char *con_ids[];             /* flexible array — أسماء الـ clocks (con_id strings) */
};
```

| Field | النوع | الدور |
|-------|-------|-------|
| `nb` | `struct notifier_block` | الـ base notifier — بيتعين فيه `notifier_call = pm_clk_notify` |
| `pm_domain` | `struct dev_pm_domain *` | الـ domain اللي بيتعين لـ `dev->pm_domain` لما device تتضاف |
| `con_ids[]` | `char *[]` | flexible array من الـ connection ID strings — آخر عنصر `NULL` |

**ارتباطها بالـ structs التانية:**
- بتحتوي على `struct notifier_block` بالـ embedding — بيتعمله `container_of` في `pm_clk_notify()`
- بتشير لـ `struct dev_pm_domain` اللي بيتعين لـ `dev->pm_domain`

---

#### `struct pm_clock_entry` (في `clock_ops.c` — internal)

**الغرض:** تمثيل كل clock واحد مربوط بـ device في إطار الـ PM. القائمة منهم بتتحفظ في `psd->clock_list`.

```c
struct pm_clock_entry {
    struct list_head node;           /* ربط في clock_list الخاص بالـ psd */
    char *con_id;                    /* اسم الـ connection ID (ممكن NULL) */
    struct clk *clk;                 /* pointer على الـ clock الفعلي */
    enum pce_status status;          /* الحالة الحالية للـ clock */
    bool enabled_when_prepared;      /* لو true: clk_prepare == clk_enable */
};
```

| Field | الدور |
|-------|-------|
| `node` | بيربطه في `psd->clock_list` بـ `list_head` |
| `con_id` | اسم الـ clock المطلوب من الـ framework (مثلاً `"ahb"`, `"apb"`) — `kstrdup`'d |
| `clk` | الـ pointer الفعلي بعد `clk_get()` |
| `status` | الـ state machine اللي بيحكم إيه الـ operations المسموحة |
| `enabled_when_prepared` | لو الـ clock بيتفعل أوتوماتيك لما بيتـ prepare — يعني `clk_prepare` == `clk_enable` معاً |

**تأثير `enabled_when_prepared`:** لما يبقى `true`، بيزود `psd->clock_op_might_sleep` بـ 1، عشان الـ locking strategy تعرف إنها ممكن تحتاج mutex مش بس spinlock.

---

#### `struct pm_subsys_data` (في `linux/pm.h`)

**الغرض:** البيانات اللي subsystem بيملكها في `dev->power.subsys_data`. لما `CONFIG_PM_CLK` موجود، بتحتوي على قائمة الـ clock entries والـ locking data.

```c
struct pm_subsys_data {
    spinlock_t lock;                  /* يحمي الـ clock_list في atomic context */
    unsigned int refcount;            /* reference count — بيتحكم فيه dev_pm_get/put_subsys_data */
#ifdef CONFIG_PM_CLK
    unsigned int clock_op_might_sleep; /* عدد الـ clocks اللي enabled_when_prepared */
    struct mutex clock_mutex;          /* يحمي الـ clock_list في sleepable context */
    struct list_head clock_list;       /* قائمة pm_clock_entry nodes */
#endif
#ifdef CONFIG_PM_GENERIC_DOMAINS
    struct pm_domain_data *domain_data;
#endif
};
```

| Field | الدور |
|-------|-------|
| `lock` | spinlock — الـ fast path لو مفيش clocks تحتاج sleep |
| `refcount` | كم حد عامل `get` للـ subsys_data — بيتحرر لما يوصل صفر |
| `clock_op_might_sleep` | counter — لو > 0 يبقى لازم mutex مش spinlock في `pm_clk_suspend/resume` |
| `clock_mutex` | mutex — الـ slow path لما الـ clock ops ممكن تـ sleep |
| `clock_list` | `list_head` — head للـ linked list من `pm_clock_entry` nodes |

---

#### `struct notifier_block` (في `linux/notifier.h`)

```c
struct notifier_block {
    notifier_fn_t notifier_call;       /* الـ callback function */
    struct notifier_block __rcu *next; /* الـ next في الـ chain */
    int priority;                      /* أولوية التنفيذ — أكبر = أسبق */
};
```

الـ `pm_clk_notifier_block` بتـ embed الـ `notifier_block` كأول field عشان `container_of` يشتغل صح.

---

#### `struct dev_pm_domain` (في `linux/pm.h`)

```c
struct dev_pm_domain {
    struct dev_pm_ops ops;          /* الـ PM callbacks: suspend, resume, ... */
    int (*start)(struct device *);
    void (*detach)(struct device *, bool power_off);
    int (*activate)(struct device *);
    void (*sync)(struct device *);
    void (*dismiss)(struct device *);
};
```

الـ `pm_clk_notifier_block` بيشير عليها عشان يعينها لـ `dev->pm_domain` لما device تتضاف للـ bus.

---

### 2. رسم العلاقات بين الـ Structs

```
bus_type
   │
   │  (notifier chain)
   ▼
atomic_notifier_head
   │
   └──► notifier_block ◄──── (embedded in)
                                    │
                         pm_clk_notifier_block
                          ├── nb  (notifier_block)
                          ├── pm_domain ──────────────────► dev_pm_domain
                          │                                    └── dev_pm_ops
                          └── con_ids[] ─► ["ahb", "apb", NULL]
                                                │
                                       (used in pm_clk_notify)
                                                │
                                                ▼
                                          struct device
                                           ├── power
                                           │    └── subsys_data ──► pm_subsys_data
                                           │                          ├── lock (spinlock)
                                           │                          ├── clock_mutex
                                           │                          ├── clock_op_might_sleep
                                           │                          └── clock_list
                                           │                               │
                                           │                    ┌──────────┴──────────┐
                                           │                    │                     │
                                           │             pm_clock_entry       pm_clock_entry
                                           │              ├── node              ├── node
                                           │              ├── con_id            ├── con_id
                                           │              ├── clk ──► struct clk (opaque)
                                           │              ├── status            ├── status
                                           │              └── enabled_when_prepared
                                           │
                                           └── pm_domain ──────────────────► dev_pm_domain
```

---

### 3. دورة الحياة (Lifecycle Diagram)

#### مرحلة الـ Initialization والتسجيل

```
Driver/Platform Code
        │
        ├─ 1. pm_clk_create(dev)
        │       └── dev_pm_get_subsys_data(dev)
        │               └── kzalloc(pm_subsys_data)
        │               └── dev->power.subsys_data = psd
        │               └── psd->refcount++
        │
        ├─ 2. pm_clk_init(dev)          [اختياري — لو psd موجود مسبقاً]
        │       └── INIT_LIST_HEAD(&psd->clock_list)
        │       └── mutex_init(&psd->clock_mutex)
        │       └── psd->clock_op_might_sleep = 0
        │
        ├─ 3a. pm_clk_add(dev, "ahb")   [إضافة بالاسم]
        │       └── __pm_clk_add(dev, "ahb", NULL)
        │               └── kzalloc(pm_clock_entry)
        │               └── ce->con_id = kstrdup("ahb")
        │               └── pm_clk_acquire(dev, ce)
        │               │       └── clk_get(dev, "ahb")  →  ce->clk
        │               │       └── clk_prepare(ce->clk)  →  PCE_STATUS_PREPARED
        │               └── list_add_tail(&ce->node, &psd->clock_list)
        │
        ├─ 3b. pm_clk_add_clk(dev, clk) [إضافة بـ pointer]
        │       └── __pm_clk_add(dev, NULL, clk)
        │               └── ce->clk = clk  (مش بيعمل clk_get)
        │               └── pm_clk_acquire(dev, ce)
        │               └── list_add_tail(...)
        │
        └─ 3c. of_pm_clk_add_clks(dev)  [من device tree]
                └── of_clk_get_parent_count(dev->of_node)
                └── for each clock:
                        └── of_clk_get(dev->of_node, i)
                        └── pm_clk_add_clk(dev, clks[i])
```

#### مرحلة الـ Runtime (suspend/resume)

```
Runtime PM Core
      │
      ├─ SUSPEND PATH:
      │    pm_clk_runtime_suspend(dev)
      │         └── pm_generic_runtime_suspend(dev)   [driver callbacks first]
      │         └── pm_clk_suspend(dev)
      │               └── pm_clk_op_lock(psd, &flags)
      │               │       [clock_op_might_sleep == 0] → spin_lock_irqsave
      │               │       [clock_op_might_sleep > 0]  → mutex_lock
      │               └── list_for_each_entry_reverse(ce, &clock_list):
      │               │       [PCE_STATUS_ENABLED + enabled_when_prepared]:
      │               │           clk_disable_unprepare → PCE_STATUS_ACQUIRED
      │               │       [PCE_STATUS_ENABLED]:
      │               │           clk_disable          → PCE_STATUS_PREPARED
      │               └── pm_clk_op_unlock(psd, &flags)
      │
      └─ RESUME PATH:
           pm_clk_runtime_resume(dev)
                └── pm_clk_resume(dev)
                │     └── pm_clk_op_lock(psd, &flags)
                │     └── list_for_each_entry(ce, &clock_list):
                │     │       __pm_clk_enable(dev, ce)
                │     │           [PCE_STATUS_ACQUIRED] → clk_prepare_enable → ENABLED
                │     │           [PCE_STATUS_PREPARED] → clk_enable        → ENABLED
                │     └── pm_clk_op_unlock(psd, &flags)
                └── pm_generic_runtime_resume(dev)    [driver callbacks after]
```

#### مرحلة الـ Teardown

```
Driver Remove / pm_clk_destroy(dev)
        │
        ├── dev_to_psd(dev)  →  psd
        ├── pm_clk_list_lock(psd)
        ├── list_for_each_entry_safe_reverse → list_move all entries to local list
        ├── psd->clock_op_might_sleep = 0
        ├── pm_clk_list_unlock(psd)
        ├── dev_pm_put_subsys_data(dev)   [psd->refcount-- → free if 0]
        └── for each ce in local list:
                __pm_clk_remove(ce)
                    [PCE_STATUS_ENABLED]  → clk_disable
                    [PCE_STATUS_PREPARED] → clk_unprepare
                    [PCE_STATUS_ACQUIRED] → (skip unprepare)
                    [any]                 → clk_put(ce->clk)
                    kfree(ce->con_id)
                    kfree(ce)
```

---

### 4. Call Flow Diagrams

#### pm_clk_add → الاستخدام في PM

```
driver calls pm_clk_add(dev, "ahb")
  → __pm_clk_add(dev, "ahb", NULL)
    → kzalloc(pm_clock_entry)
    → pm_clk_acquire(dev, ce)
      → clk_get(dev, "ahb")              [clock framework lookup]
        → clk_is_enabled_when_prepared?
          YES → ce->status = PCE_STATUS_ACQUIRED
                ce->enabled_when_prepared = true
                psd->clock_op_might_sleep++
          NO  → clk_prepare(ce->clk)
                → ce->status = PCE_STATUS_PREPARED
    → pm_clk_list_lock(psd)             [mutex + spinlock]
    → list_add_tail(&ce->node, &psd->clock_list)
    → pm_clk_list_unlock(psd)
```

#### Bus Notifier Flow

```
platform bus registers pm_clk_add_notifier(bus, clknb)
  → blocking_notifier_chain_register(bus->p->bus_notifier, &clknb->nb)

device added to bus
  → bus_notify(BUS_NOTIFY_ADD_DEVICE, dev)
    → pm_clk_notify(&clknb->nb, BUS_NOTIFY_ADD_DEVICE, dev)
      → clknb = container_of(nb, struct pm_clk_notifier_block, nb)
      → dev->pm_domain == NULL?  YES →
          pm_clk_create(dev)
          dev_pm_domain_set(dev, clknb->pm_domain)
          for each con_id in clknb->con_ids[]:
              pm_clk_add(dev, con_id)

device removed from bus
  → bus_notify(BUS_NOTIFY_DEL_DEVICE, dev)
    → pm_clk_notify(...)
      → dev->pm_domain == clknb->pm_domain?  YES →
          dev_pm_domain_set(dev, NULL)
          pm_clk_destroy(dev)
```

#### Full Runtime Suspend Call Chain

```
RPM core calls .runtime_suspend(dev)          [= pm_clk_runtime_suspend]
  → pm_generic_runtime_suspend(dev)
    → driver->pm->runtime_suspend(dev)        [driver's own callback]
  → pm_clk_suspend(dev)
    → dev_to_psd(dev)                         [dev->power.subsys_data]
    → pm_clk_op_lock(psd, &flags)
      → clock_op_might_sleep == 0?
        YES → spin_lock_irqsave(&psd->lock)   [atomic-safe path]
        NO  → mutex_lock(&psd->clock_mutex)   [sleepable path]
    → list_for_each_entry_reverse(ce, clock_list)
      → ce->status == PCE_STATUS_ENABLED?
        + enabled_when_prepared → clk_disable_unprepare(ce->clk)
        else                    → clk_disable(ce->clk)
    → pm_clk_op_unlock(psd, &flags)
```

#### الـ Clock State Machine

```
          clk_get()
[NONE] ──────────────► [ACQUIRED]
   │     (success,        │   │
   │   enabled_when_      │   │ clk_prepare_enable()
   │    prepared=true)    │   │ (on resume)
   │                      │   ▼
   │     clk_get() +   [ENABLED] ◄──── clk_enable()
   │     clk_prepare()    │              (from PREPARED)
   │     (success)        │
   ▼                      │ clk_disable_unprepare()
[PREPARED] ◄─────────────┘ (on suspend)
   │
   │ clk_enable()
   ▼
[ENABLED]
   │
   │ clk_disable()
   ▼
[PREPARED]

[ERROR]: clk_get() أو clk_prepare() فشل — الـ entry موجود بس مش بيتعمل فيه حاجة
```

---

### 5. استراتيجية الـ Locking

الـ `pm_clock.h` وتطبيقه بيستخدم **two-level locking scheme** متطور جداً عشان يخدم كل من الـ atomic context والـ sleepable context.

---

#### الـ Locks الموجودة في `pm_subsys_data`

| Lock | النوع | بيحمي إيه |
|------|-------|-----------|
| `psd->lock` | `spinlock_t` | الـ `clock_list` في atomic context — بيتاخد دايماً مع IRQs disabled |
| `psd->clock_mutex` | `struct mutex` | الـ `clock_list` في sleepable context — لما `clock_op_might_sleep > 0` |

---

#### السيناريوهات الأربعة

**1. تعديل الـ `clock_list` (إضافة/حذف entries) — `pm_clk_list_lock`:**
```
mutex_lock(&psd->clock_mutex)   ← أول
spin_lock_irq(&psd->lock)       ← تاني
   [تعديل clock_list]
spin_unlock_irq(&psd->lock)
mutex_unlock(&psd->clock_mutex)
```
ده بيضمن حماية كاملة ضد أي concurrent access سواء atomic أو sleepable.

**2. تشغيل/إيقاف الـ clocks (suspend/resume) — `pm_clk_op_lock`:**

- لو `clock_op_might_sleep == 0`:
  ```
  spin_lock_irqsave(&psd->lock)   ← آمن في أي context
  ```
- لو `clock_op_might_sleep > 0`:
  ```
  [atomic context] → ERROR! يرجع -EPERM
  [process context] → mutex_lock(&psd->clock_mutex)
  ```

**3. ليه الاتنين مع بعض في `pm_clk_list_lock`?**

عشان `pm_clk_suspend/resume` ممكن يشتغلوا في atomic context (في الـ spinlock path)، ولازم التعديل على الـ list يكون محمي من كل الاتجاهين. لو اخدنا بس الـ mutex في التعديل، ممكن يحصل race مع الـ spinlock path في suspend/resume.

---

#### ترتيب الـ Lock Ordering (لمنع deadlock)

```
دايماً الترتيب:
  clock_mutex  →  lock (spinlock)

الـ unlock بالعكس:
  lock (spinlock)  →  clock_mutex
```

**لا يجوز أبداً:** اخد الـ `lock` spinlock الأول وبعدين تحاول تاخد الـ `clock_mutex` — ده deadlock مضمون.

---

#### جدول ملخص: مين يحمي إيه

| Operation | Lock المستخدم | Context |
|-----------|---------------|---------|
| `pm_clk_add` / `pm_clk_remove_clk` | mutex + spinlock (كلاهم) | process only |
| `pm_clk_suspend` / `pm_clk_resume` (no sleep clocks) | spinlock فقط | any (atomic OK) |
| `pm_clk_suspend` / `pm_clk_resume` (with sleep clocks) | mutex فقط | process only |
| `pm_clk_destroy` | mutex + spinlock (كلاهم) | process only |
| قراءة `clock_op_might_sleep` | spinlock (atomic read under lock) | any |
| تعديل `clock_op_might_sleep` | mutex + spinlock | process only |
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### Group 1: Lifecycle / Setup

| Function | Kconfig Guard | الهدف |
|---|---|---|
| `pm_clk_init(dev)` | `CONFIG_PM_CLK` | initialize الـ clock_list والـ mutex بدون alloc |
| `pm_clk_create(dev)` | `CONFIG_PM_CLK` | allocate وربط `pm_subsys_data` بالـ device |
| `devm_pm_clk_create(dev)` | `CONFIG_PM_CLK` | نفس `pm_clk_create` لكن managed — auto cleanup عند device release |
| `pm_clk_destroy(dev)` | `CONFIG_PM_CLK` | remove كل الـ clock entries وحرّر `pm_subsys_data` |

#### Group 2: Clock Registration

| Function | Kconfig Guard | الهدف |
|---|---|---|
| `pm_clk_add(dev, con_id)` | `CONFIG_PM_CLK` | أضف clock بـ connection ID من الـ clk framework |
| `pm_clk_add_clk(dev, clk)` | `CONFIG_PM_CLK` | أضف `struct clk *` جاهز مسبقاً |
| `of_pm_clk_add_clks(dev)` | `CONFIG_PM_CLK` | أضف كل الـ clocks من الـ DT node تلقائياً |
| `pm_clk_remove_clk(dev, clk)` | `CONFIG_PM_CLK` | أزل clock واحد من القائمة |

#### Group 3: Runtime Suspend / Resume

| Function | Kconfig Guard | الهدف |
|---|---|---|
| `pm_clk_suspend(dev)` | `CONFIG_PM_CLK` | أوقف كل الـ clocks (عند system/runtime suspend) |
| `pm_clk_resume(dev)` | `CONFIG_PM_CLK` | أشغّل كل الـ clocks (عند system/runtime resume) |
| `pm_clk_runtime_suspend(dev)` | `CONFIG_PM` | `pm_generic_runtime_suspend` + `pm_clk_suspend` |
| `pm_clk_runtime_resume(dev)` | `CONFIG_PM` | `pm_clk_resume` + `pm_generic_runtime_resume` |

#### Group 4: Notifier / Bus Integration

| Function | Kconfig Guard | الهدف |
|---|---|---|
| `pm_clk_add_notifier(bus, clknb)` | `CONFIG_HAVE_CLK` | سجّل bus notifier يعمل auto clock management |

#### Group 5: Query Helper

| Function | Kconfig Guard | الهدف |
|---|---|---|
| `pm_clk_no_clocks(dev)` | `CONFIG_PM_CLK` (inline) | اسأل: هل القائمة فاضية؟ |

#### Macro

| Macro | الهدف |
|---|---|
| `USE_PM_CLK_RUNTIME_OPS` | ملء `dev_pm_ops` بـ `runtime_suspend/resume` callbacks مباشرة |

---

### الـ Data Structures الجوهرية

قبل الدخول في الـ functions لازم تعرف البنيات الأساسية اللي بيشتغل عليها الـ subsystem.

**`struct pm_clock_entry`** — داخلية، مش exposed في الـ header:
```c
struct pm_clock_entry {
    struct list_head node;           /* linked في psd->clock_list */
    char            *con_id;         /* اسم الـ clock connection */
    struct clk      *clk;            /* الـ clock handle نفسه */
    enum pce_status  status;         /* NONE / ACQUIRED / PREPARED / ENABLED / ERROR */
    bool             enabled_when_prepared; /* clk يعمل prepare وenable في نفس الوقت؟ */
};
```

**`enum pce_status`** — state machine لكل clock entry:
```
PCE_STATUS_NONE
    → clk_get()        → PCE_STATUS_ACQUIRED  (إذا clk_is_enabled_when_prepared)
    → clk_prepare()    → PCE_STATUS_PREPARED
    → clk_enable()     → PCE_STATUS_ENABLED
    → error            → PCE_STATUS_ERROR
```

**`struct pm_clk_notifier_block`** — exposed في الـ header:
```c
struct pm_clk_notifier_block {
    struct notifier_block  nb;        /* embedded notifier — يُمرّر لـ bus_register_notifier */
    struct dev_pm_domain  *pm_domain; /* الـ PM domain اللي هيتعين للـ device */
    char                  *con_ids[]; /* flexible array — أسماء الـ clocks المطلوبة */
};
```

---

### Group 1: Lifecycle / Setup

الهدف من المجموعة دي هو إعداد وتنظيف بنية البيانات الخاصة بـ PM clocks لكل device. الـ `pm_subsys_data` هي الحاوية الرئيسية اللي بتخزن فيها `clock_list` والـ `clock_mutex` والـ `clock_op_might_sleep` counter.

---

#### `pm_clk_init`

```c
void pm_clk_init(struct device *dev);
```

بتـ initialize الـ `clock_list` والـ `clock_mutex` داخل `dev->power.subsys_data` إذا كانت موجودة بالفعل. مش بتعمل allocation — بتفترض إن `pm_subsys_data` اتعملت قبل كده. بتصفّر `clock_op_might_sleep` counter.

**Parameters:**
- `dev` — الـ device المطلوب تهيئة الـ clock list بتاعته.

**Return:** void

**Key details:**
- بتعمل `INIT_LIST_HEAD` و `mutex_init` — safe تتنادى من أي context.
- لو `dev->power.subsys_data == NULL`، بتعمل no-op بدون error.
- عادةً بيتنادى منها الـ bus/platform driver في الـ probe أو من `pm_clk_create` indirectly.

**Who calls it:** الـ `pm_domain` implementations أو platform drivers اللي بتعمل subsys_data manually.

---

#### `pm_clk_create`

```c
int pm_clk_create(struct device *dev);
```

بتـ allocate وتـ attach `pm_subsys_data` للـ device عن طريق `dev_pm_get_subsys_data()`. الـ `pm_clk_init` بتتعمل ضمنياً من `dev_pm_get_subsys_data` عند أول استدعاء. مش بتعمل clock entries — بس بتجهّز الـ container.

**Parameters:**
- `dev` — الـ target device.

**Return:** `0` عند النجاح، أو error code سالب (عادةً `-ENOMEM`).

**Key details:**
- **Reference counted** — كل `pm_clk_create` بتزوّد reference count. لازم تقابلها `pm_clk_destroy` لتحريرها.
- آمنة تتنادى أكتر من مرة على نفس الـ device (reference counting).
- **Locking:** داخلياً `dev_pm_get_subsys_data` بتاخد `dev->power.lock` spinlock.

**Who calls it:** الـ `pm_clk_notify` عند `BUS_NOTIFY_ADD_DEVICE`، أو الـ driver مباشرة في الـ probe.

---

#### `devm_pm_clk_create`

```c
int devm_pm_clk_create(struct device *dev);
```

**Managed** version من `pm_clk_create`. بتسجّل `pm_clk_destroy` كـ devres action عن طريق `devm_add_action_or_reset`، فبتتنادى تلقائياً عند `device_unregister` أو فشل الـ probe.

```
devm_pm_clk_create(dev)
    └── pm_clk_create(dev)           /* allocate + ref++ */
    └── devm_add_action_or_reset(dev, pm_clk_destroy_action, dev)
              /* إذا فشلت → pm_clk_destroy(dev) فوراً */
```

**Parameters:**
- `dev` — الـ target device.

**Return:** `0` أو error code سالب.

**Key details:**
- لو `devm_add_action_or_reset` فشلت، بتـ rollback بـ `pm_clk_destroy` تلقائياً — لا resource leak.
- الـ driver مش محتاج يعمل `pm_clk_destroy` في الـ remove إذا استخدم النسخة دي.

**Who calls it:** Modern platform/SoC drivers في الـ probe.

---

#### `pm_clk_destroy`

```c
void pm_clk_destroy(struct device *dev);
```

بتحذف كل الـ `pm_clock_entry` objects من الـ `clock_list`، وبتعمل لكل واحد منهم `clk_disable` + `clk_unprepare` + `clk_put` حسب الـ status. بعدين بتحرر `pm_subsys_data` بـ `dev_pm_put_subsys_data`.

**Pseudocode flow:**
```
pm_clk_destroy(dev):
    psd = dev_to_psd(dev)
    if !psd → return

    INIT_LIST_HEAD(&list)            /* قائمة مؤقتة */
    pm_clk_list_lock(psd)           /* mutex + spinlock */
        /* انقل كل entries من clock_list → list */
        list_for_each_entry_safe_reverse(ce, c, &psd->clock_list, node)
            list_move(ce, list)
        psd->clock_op_might_sleep = 0
    pm_clk_list_unlock(psd)

    dev_pm_put_subsys_data(dev)      /* حرّر/انقص الـ ref */

    /* أدمر كل entry بعد تحرير الـ lock */
    list_for_each_entry_safe_reverse(ce, c, &list, node)
        list_del(ce)
        __pm_clk_remove(ce)          /* disable + unprepare + put + kfree */
```

**Parameters:**
- `dev` — الـ device المراد تنظيف الـ clocks بتاعته.

**Return:** void

**Key details:**
- الـ `__pm_clk_remove` بتتنادى **خارج الـ lock** لتفادي sleeping while holding spinlock (الـ `clk_unprepare` ممكن تـ sleep).
- الـ list entries بتتنقل لـ local list أثناء الـ lock، وبعدين الـ cleanup يحصل free من أي lock.
- الترتيب `_reverse` مهم — يضمن disable بعكس ترتيب الـ enable.

**Who calls it:** الـ driver remove، الـ `pm_clk_destroy_action` (devres)، `pm_clk_notify` عند `BUS_NOTIFY_DEL_DEVICE`.

---

### Group 2: Clock Registration

الهدف هو بناء قائمة الـ clocks اللي الـ PM subsystem هيتحكم فيها. كل clock بيتحول لـ `pm_clock_entry` وبيتضاف لـ `psd->clock_list`.

---

#### `pm_clk_add`

```c
int pm_clk_add(struct device *dev, const char *con_id);
```

بتضيف clock جديد للـ PM clock list بناءً على الـ connection ID النصي. بتستدعي داخلياً `__pm_clk_add(dev, con_id, NULL)` اللي بتعمل `clk_get(dev, con_id)` لجلب الـ handle.

**Parameters:**
- `dev` — الـ device صاحب الـ clock.
- `con_id` — اسم الـ clock connection (ممكن `NULL` للـ default clock).

**Return:** `0` أو error code سالب (`-ENOMEM`, `-EINVAL`).

**Key details:**
- بعد `clk_get` ناجح، بتعمل `clk_prepare` إذا `!clk_is_enabled_when_prepared`.
- بتزوّد `psd->clock_op_might_sleep` بـ 1 لو الـ clock يعمل prepare+enable في نفس الوقت (يعني الـ suspend/resume هيحتاج mutex مش spinlock).
- الـ `pm_clk_list_lock` = `mutex_lock` + `spin_lock_irq` في نفس الوقت.

**Who calls it:** Platform drivers، `pm_clk_notify`.

---

#### `pm_clk_add_clk`

```c
int pm_clk_add_clk(struct device *dev, struct clk *clk);
```

نفس `pm_clk_add` لكن بتقبل `struct clk *` مباشرة بدل الـ con_id. **الـ subsystem بيستلم ملكية الـ clk pointer** — الـ caller ميعملش `clk_put` بعدها.

**Parameters:**
- `dev` — الـ device.
- `clk` — pointer للـ clock المراد إضافته. لو `IS_ERR(clk)` → ترجع `-ENOENT`.

**Return:** `0` أو error code سالب.

**Key details:**
- الـ `ce->con_id` بيفضل `NULL` — مش محتاج `clk_get` لأن الـ handle موجود.
- عند الـ error، الـ caller مش مسؤول عن الـ clk — الـ function بتـ return فقط، ما بتعملش `clk_put`.
- **مهم:** لو نجحت، الـ caller **لا يجوز** يعمل `clk_put(clk)`.

**Who calls it:** `of_pm_clk_add_clks`، drivers عندها clock handles جاهزة.

---

#### `of_pm_clk_add_clks`

```c
int of_pm_clk_add_clks(struct device *dev);
```

بتقرأ كل الـ clocks من الـ `clocks` property في الـ DT node للـ device وبتضيفهم كلهم للـ PM clock list. اتصميمها للـ platform devices اللي بتوصف كل الـ clocks بتاعتها في الـ DT.

**Pseudocode flow:**
```
of_pm_clk_add_clks(dev):
    count = of_clk_get_parent_count(dev->of_node)
    if count <= 0 → return -ENODEV

    clks = kcalloc(count, sizeof(*clks))
    for i in 0..count:
        clks[i] = of_clk_get(dev->of_node, i)
        if IS_ERR(clks[i]) → goto error

        ret = pm_clk_add_clk(dev, clks[i])
        if ret:
            clk_put(clks[i])
            goto error

    kfree(clks)
    return i  /* عدد الـ clocks المضافة */

error:
    while i--:
        pm_clk_remove_clk(dev, clks[i])  /* rollback */
    kfree(clks)
    return ret
```

**Parameters:**
- `dev` — الـ device اللي لازم يكون عنده `dev->of_node` صالح.

**Return:** عدد الـ clocks المضافة بنجاح (양수)، أو error code سالب.

**Key details:**
- **Atomic rollback** — لو أي clock فشل، كل اللي اتضاف قبله بيتشال بـ `pm_clk_remove_clk`.
- الـ `clks[]` array مؤقتة — بس لتتبع ترتيب الـ rollback.
- يتطلب `CONFIG_PM_CLK` + `CONFIG_OF`.

**Who calls it:** SoC platform drivers، generic PM domain drivers.

---

#### `pm_clk_remove_clk`

```c
void pm_clk_remove_clk(struct device *dev, struct clk *clk);
```

بتشيل clock entry معين (بـ pointer match) من الـ clock list وبتعمله cleanup كامل (disable → unprepare → put).

**Parameters:**
- `dev` — الـ device.
- `clk` — الـ clock pointer المراد إزالته.

**Return:** void — لو مش موجود، no-op.

**Key details:**
- البحث بيتم بـ pointer comparison `clk == ce->clk`.
- الـ `list_del` وتعديل `clock_op_might_sleep` بيحصلوا تحت الـ lock.
- الـ `__pm_clk_remove` (اللي فيها `clk_disable` + `clk_unprepare` + `clk_put`) بتتنادى **بعد** تحرير الـ lock — لأنها ممكن تـ sleep.
- بتنقص `psd->clock_op_might_sleep` لو الـ clock كان من النوع `enabled_when_prepared`.

**Who calls it:** `of_pm_clk_add_clks` عند الـ rollback، drivers مباشرة.

---

### Group 3: Runtime Suspend / Resume

دي أهم مجموعة — هي الـ callbacks الفعلية اللي بيشتغل بيها الـ PM subsystem.

---

#### الـ Locking Model في الـ Suspend/Resume

الـ subsystem عنده **locking scheme** مدروس لأن الـ suspend/resume ممكن يتنادوا من atomic context:

```
┌─────────────────────────────────────────────────────┐
│ clock_op_might_sleep == 0                           │
│   → كل الـ clocks normal (prepare/enable منفصلين)  │
│   → استخدم spin_lock_irqsave فقط                   │
├─────────────────────────────────────────────────────┤
│ clock_op_might_sleep > 0                            │
│   → في clocks بتعمل prepare+enable معاً (تنام)     │
│   → استخدم mutex فقط                               │
│   → يمنع الاستدعاء من atomic context               │
└─────────────────────────────────────────────────────┘
```

الـ `pm_clk_op_lock` بتختار تلقائياً بين الـ spinlock والـ mutex حسب القيمة دي.

---

#### `pm_clk_suspend`

```c
int pm_clk_suspend(struct device *dev);
```

بتوقف كل الـ clocks في قائمة الـ device بترتيب عكسي. بتعمل `clk_disable_unprepare` للـ clocks من النوع `enabled_when_prepared` أو `clk_disable` فقط للنوع التاني.

**Parameters:**
- `dev` — الـ device المراد تعليق الـ clocks بتاعته.

**Return:** `0` عند النجاح، `-EPERM` لو نادينا من atomic context وفيه clocks تحتاج sleep.

**Pseudocode flow:**
```
pm_clk_suspend(dev):
    psd = dev_to_psd(dev)
    if !psd → return 0

    ret = pm_clk_op_lock(psd, &flags, __func__)
    if ret → return ret

    list_for_each_entry_reverse(ce, &psd->clock_list, node):
        if ce->status == PCE_STATUS_ENABLED:
            if ce->enabled_when_prepared:
                clk_disable_unprepare(ce->clk)
                ce->status = PCE_STATUS_ACQUIRED
            else:
                clk_disable(ce->clk)
                ce->status = PCE_STATUS_PREPARED

    pm_clk_op_unlock(psd, &flags)
    return 0
```

**Key details:**
- الترتيب العكسي (`_reverse`) مهم لضمان صحة الـ clock dependencies.
- الـ clocks اللي `status != PCE_STATUS_ENABLED` يتم تجاهلها.
- لا تُعاقب الـ caller لو الـ clock كانت معطلة أصلاً.

**Who calls it:** `pm_clk_runtime_suspend`، الـ `dev_pm_ops.suspend` في الـ system sleep.

---

#### `pm_clk_resume`

```c
int pm_clk_resume(struct device *dev);
```

بتشغّل كل الـ clocks في القائمة بالترتيب الطبيعي عن طريق `__pm_clk_enable` لكل entry.

**Parameters:**
- `dev` — الـ device.

**Return:** `0` أو `-EPERM` (atomic context مع sleeping clocks).

**Key details:**
- `__pm_clk_enable` بتختار `clk_prepare_enable` أو `clk_enable` فقط حسب الـ status الحالي.
- الأخطاء في enable بتتـ log بـ `dev_err` لكن ما بتوقفش الـ loop — بتكمّل باقي الـ clocks.
- الترتيب طبيعي (من الأقدم للأحدث) — عكس الـ suspend.

**Who calls it:** `pm_clk_runtime_resume`، الـ `dev_pm_ops.resume`.

---

#### `pm_clk_runtime_suspend`

```c
int pm_clk_runtime_suspend(struct device *dev);
```

الـ complete runtime suspend callback. بتستدعي `pm_generic_runtime_suspend` أولاً (اللي بتنادي الـ driver's `->runtime_suspend` callback)، وبعدين بتوقف الـ clocks. لو `pm_clk_suspend` فشلت، بتعمل rollback بـ `pm_generic_runtime_resume`.

```
pm_clk_runtime_suspend(dev):
    ret = pm_generic_runtime_suspend(dev)   /* driver callback */
    if ret → return ret

    ret = pm_clk_suspend(dev)               /* اطفي الـ clocks */
    if ret:
        pm_generic_runtime_resume(dev)      /* rollback */
        return ret

    return 0
```

**Return:** `0` أو error code.

**Key details:**
- ترتيب العمليات: driver suspend **أولاً** ثم clock disable — لأن الـ driver محتاج الـ clocks شغالة أثناء الـ suspend sequence.
- يُستخدم عبر الـ macro `USE_PM_CLK_RUNTIME_OPS`.

**Who calls it:** الـ RPM core عند `runtime_suspend`.

---

#### `pm_clk_runtime_resume`

```c
int pm_clk_runtime_resume(struct device *dev);
```

عكس `pm_clk_runtime_suspend`. بتشغّل الـ clocks أولاً ثم بتنادي الـ driver resume callback.

```
pm_clk_runtime_resume(dev):
    ret = pm_clk_resume(dev)                /* شغّل الـ clocks */
    if ret → return ret

    return pm_generic_runtime_resume(dev)   /* driver callback */
```

**Return:** `0` أو error code.

**Key details:**
- الـ clocks لازم تكون شغالة **قبل** ما الـ driver يحاول يوصل للـ hardware registers.
- لو `pm_clk_resume` فشلت، الـ driver callback ما بتتنادّيش.

**Who calls it:** الـ RPM core عند `runtime_resume`.

---

### Group 4: Bus Notifier Integration

---

#### `pm_clk_add_notifier`

```c
void pm_clk_add_notifier(const struct bus_type *bus,
                          struct pm_clk_notifier_block *clknb);
```

بتسجّل `pm_clk_notify` كـ bus notifier على الـ `bus` المحدد. لما أي device يتضاف أو يتشال من الـ bus، الـ notifier بيشتغل ويعمل الـ clock management تلقائياً.

**Parameters:**
- `bus` — نوع الـ bus (مثلاً `platform_bus_type`).
- `clknb` — الـ notifier block مع بيانات الـ pm_domain والـ con_ids. الـ `nb.notifier_call` بيتكتب تلقائياً — ما تحتاجش تملأه.

**Return:** void

**Key details:**
- بتكتب `clknb->nb.notifier_call = pm_clk_notify` قبل الـ register.
- الـ `clknb->pm_domain` و `clknb->con_ids` لازم يكونوا معبّيين مسبقاً.
- بعد التسجيل، أي `BUS_NOTIFY_ADD_DEVICE` event هيشغّل `pm_clk_create` + `pm_clk_add` + `dev_pm_domain_set` تلقائياً للـ devices الجديدة.
- **`CONFIG_HAVE_CLK`** guard — لو الـ platform ما عندهاش clock framework، الـ function تبقى no-op.

**Who calls it:** SoC init code، platform bus setup code، عادةً من `__init` functions.

---

#### `pm_clk_notify` (static)

```c
static int pm_clk_notify(struct notifier_block *nb,
                          unsigned long action, void *data);
```

الـ callback الفعلي. مش exposed في الـ header لكن بيشتغل على events من الـ bus.

**عند `CONFIG_PM_CLK`:**

| Event | الفعل |
|---|---|
| `BUS_NOTIFY_ADD_DEVICE` | `pm_clk_create` + `dev_pm_domain_set` + `pm_clk_add` لكل con_id |
| `BUS_NOTIFY_DEL_DEVICE` | `dev_pm_domain_set(NULL)` + `pm_clk_destroy` |

**عند `!CONFIG_PM_CLK`** (fallback):

| Event | الفعل |
|---|---|
| `BUS_NOTIFY_BIND_DRIVER` | `clk_prepare_enable` لكل con_id (force on) |
| `BUS_NOTIFY_DRIVER_NOT_BOUND` / `BUS_NOTIFY_UNBOUND_DRIVER` | `clk_disable_unprepare` لكل con_id |

**Key details:**
- الـ `container_of(nb, struct pm_clk_notifier_block, nb)` بيوصل للـ con_ids والـ pm_domain.
- لو `dev->pm_domain` معبّي بقيمة تانية → تجاهل الـ ADD event.
- لو `dev->pm_domain != clknb->pm_domain` → تجاهل الـ DEL event.

---

### Group 5: Query Helper

---

#### `pm_clk_no_clocks`

```c
static inline bool pm_clk_no_clocks(struct device *dev);
```

بتسأل: هل الـ device مش عنده أي clocks مسجّلة في الـ PM clock list؟

**Parameters:**
- `dev` — الـ device.

**Return:**
- `true` — القائمة فاضية أو `subsys_data == NULL` أو `dev == NULL`.
- `false` — في على الأقل clock واحد في القائمة.

**Key details:**
- inline — zero overhead.
- بتتحقق من `dev && dev->power.subsys_data` أولاً لتفادي null dereference.
- عند `!CONFIG_PM_CLK` → دايماً ترجع `true` (trivial stub).
- مش بتاخد lock — النتيجة ممكن تتغير بعدها فوراً، لكن كافية لـ fast-path checks.

**Who calls it:** PM domain code، drivers قبل ما يحاولوا يعملوا suspend.

---

### الـ Macro: `USE_PM_CLK_RUNTIME_OPS`

```c
#ifdef CONFIG_PM
#define USE_PM_CLK_RUNTIME_OPS \
    .runtime_suspend = pm_clk_runtime_suspend, \
    .runtime_resume  = pm_clk_runtime_resume,
#else
#define USE_PM_CLK_RUNTIME_OPS
#endif
```

**الاستخدام:**
```c
static const struct dev_pm_ops my_driver_pm_ops = {
    USE_PM_CLK_RUNTIME_OPS
    /* ممكن تضيف system sleep callbacks هنا */
};
```

ده اختصار إنك تملا الـ `dev_pm_ops` بالـ runtime callbacks الجاهزة. عند `!CONFIG_PM` ينحذف تلقائياً. بيغني عن كتابة wrapper يدوي في كل driver.

---

### الـ Internal Locking: تفاصيل `pm_clk_op_lock`

```c
static int pm_clk_op_lock(struct pm_subsys_data *psd,
                           unsigned long *flags, const char *fn)
```

ده أعقد جزء في الـ subsystem — بيضمن إن الـ suspend/resume آمنة في أي context:

```
pm_clk_op_lock:
try_again:
    spin_lock_irqsave(&psd->lock, *flags)

    if clock_op_might_sleep == 0:
        /* كل الـ clocks عادية → الـ spinlock كافي */
        return 0  /* نفضل hold الـ spinlock */

    /* في clocks تنام */
    if atomic_context:
        spin_unlock_irqrestore(...)
        might_sleep()  /* kernel warning */
        return -EPERM

    /* نحوّل للـ mutex */
    spin_unlock_irqrestore(...)
    mutex_lock(&psd->clock_mutex)

    if likely(clock_op_might_sleep):
        return 0  /* نفضل hold الـ mutex */

    /* القيمة اتغيرت بين الـ spin unlock والـ mutex lock! */
    mutex_unlock(...)
    goto try_again  /* ابدأ من الأول */
```

الـ `try_again` loop ضرورية لأن `clock_op_might_sleep` ممكن يتغير بين تحرير الـ spinlock وأخذ الـ mutex — race condition محتملة بتتعالج بشكل صحيح.
## Phase 5: دليل الـ Debugging الشامل

الـ subsystem ده هو **PM Clock** (`pm_clock.h`) — المسؤول عن إدارة الـ clocks الخاصة بالـ devices أثناء دورات الـ runtime PM (suspend/resume). الـ debugging هنا بيشمل مستويين: الـ software (kernel PM + clk framework) والـ hardware (oscilloscope + register dumps).

---

### Software Level

#### 1. Debugfs Entries

الـ PM clock subsystem مش بيسجّل entries خاصة بيه في debugfs بشكل مباشر، لكن بيعتمد على entries الـ PM العامة والـ clk framework.

| Entry | المسار | الغرض |
|-------|--------|--------|
| runtime PM status | `/sys/kernel/debug/pm_genpd/` | حالة الـ power domain لكل device |
| clk tree | `/sys/kernel/debug/clk/` | شجرة الـ clocks الكاملة + enable count |
| clk summary | `/sys/kernel/debug/clk/clk_summary` | ملخص كل الـ clocks مع rates و enable counts |
| rpm stats | `/sys/kernel/debug/pm_debug/` | إحصائيات الـ runtime PM |

```bash
# اقرا شجرة الـ clocks الكاملة
cat /sys/kernel/debug/clk/clk_summary

# دور على clock معين (مثلاً uart_clk)
grep uart /sys/kernel/debug/clk/clk_summary

# شوف حالة power domain معين
ls /sys/kernel/debug/pm_genpd/
cat /sys/kernel/debug/pm_genpd/<domain_name>/summary
```

**Output مثال من clk_summary:**
```
                                 enable  prepare  protect                                duty
   clock                          cnt    cnt      cnt        rate   accuracy phase  cycle
---------------------------------------------------------------------------------------------
 osc_24m                            3      3        0    24000000          0     0  50000
    uart_clk                        1      1        0   115200000          0     0  50000
```
الـ `enable cnt = 0` بيعني الـ clock متفعلش رغم إنه مضاف للـ device — دي مشكلة شائعة.

---

#### 2. Sysfs Entries

| Entry | الغرض |
|-------|--------|
| `/sys/bus/<bus>/devices/<dev>/power/runtime_status` | حالة الـ runtime PM (active/suspended/suspending) |
| `/sys/bus/<bus>/devices/<dev>/power/runtime_enabled` | هل الـ runtime PM مفعّل |
| `/sys/bus/<bus>/devices/<dev>/power/control` | `auto` أو `on` |
| `/sys/bus/<bus>/devices/<dev>/power/runtime_suspended_time` | وقت الـ suspend بالـ ms |
| `/sys/bus/<bus>/devices/<dev>/power/runtime_active_time` | وقت الـ active بالـ ms |

```bash
# مثال عملي لـ device على i2c bus
DEV="0-0050"  # عنوان الـ device
BUS="i2c"

cat /sys/bus/$BUS/devices/$DEV/power/runtime_status
cat /sys/bus/$BUS/devices/$DEV/power/runtime_enabled
cat /sys/bus/$BUS/devices/$DEV/power/control

# لو عايز تعمل force active للـ device (للـ debugging بس)
echo on > /sys/bus/$BUS/devices/$DEV/power/control
```

---

#### 3. Ftrace — Tracepoints والـ Events

الـ PM clock بيستخدم tracepoints الـ clk framework والـ runtime PM.

```bash
# فعّل الـ tracing
mount -t tracefs nodev /sys/kernel/tracing

# Events الـ clk framework
echo 1 > /sys/kernel/tracing/events/clk/clk_enable/enable
echo 1 > /sys/kernel/tracing/events/clk/clk_disable/enable
echo 1 > /sys/kernel/tracing/events/clk/clk_prepare/enable
echo 1 > /sys/kernel/tracing/events/clk/clk_unprepare/enable
echo 1 > /sys/kernel/tracing/events/clk/clk_set_rate/enable

# Events الـ runtime PM
echo 1 > /sys/kernel/tracing/events/rpm/rpm_suspend/enable
echo 1 > /sys/kernel/tracing/events/rpm/rpm_resume/enable
echo 1 > /sys/kernel/tracing/events/rpm/rpm_idle/enable

# فلتر على device معين
echo 'name == "my_device"' > /sys/kernel/tracing/events/rpm/rpm_suspend/filter

# شغّل الـ tracer
echo 1 > /sys/kernel/tracing/tracing_on
# ... اعمل العملية اللي بتحتاج تتريسها ...
cat /sys/kernel/tracing/trace
echo 0 > /sys/kernel/tracing/tracing_on
```

**مثال output:**
```
  kworker/0:1-45    [000] d..1   12.345678: clk_enable: uart_clk
  kworker/0:1-45    [000] d..1   12.345700: rpm_resume: device=serial0 flags=0x0
```

---

#### 4. Printk والـ Dynamic Debug

```bash
# فعّل dynamic debug لملف pm_clock.c بالكامل
echo 'file drivers/base/power/clock_ops.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل الـ pm debug messages
echo 'module pm_clk +p' > /sys/kernel/debug/dynamic_debug/control

# شوف الـ debug messages الفعّالة حالياً
grep pm_clock /sys/kernel/debug/dynamic_debug/control

# فعّل الـ kernel PM debug level
echo 1 > /sys/power/pm_debug_messages

# تابع الـ messages
dmesg -w | grep -E "(pm_clk|clk_|rpm_)"
```

---

#### 5. Kernel Config Options للـ Debugging

| Option | الغرض |
|--------|--------|
| `CONFIG_PM_CLK` | يفعّل الـ pm_clock subsystem نفسه |
| `CONFIG_PM_DEBUG` | يفعّل PM debug messages العامة |
| `CONFIG_PM_ADVANCED_DEBUG` | يضيف sysfs attributes إضافية للـ PM |
| `CONFIG_PM_SLEEP_DEBUG` | debug لـ sleep states |
| `CONFIG_DEBUG_DRIVER` | debug عام للـ driver model |
| `CONFIG_HAVE_CLK` | يفعّل دعم pm_clk_add_notifier |
| `CONFIG_CLK_DEBUG` | debug للـ clk framework |
| `CONFIG_COMMON_CLK_DEBUG` | debugfs entries للـ common clk framework |
| `CONFIG_PM_TRACE` | tracing للـ PM events |

```bash
# تحقق من الـ config الحالية
zcat /proc/config.gz | grep -E "CONFIG_(PM_CLK|PM_DEBUG|CLK_DEBUG|COMMON_CLK)"

# أو
grep -E "PM_CLK|PM_DEBUG|CLK_DEBUG" /boot/config-$(uname -r)
```

---

#### 6. أدوات خاصة بالـ Subsystem

```bash
# راجع كل الـ clocks المرتبطة بـ device معين عن طريق clk_summary
grep -A5 "device_name" /sys/kernel/debug/clk/clk_summary

# استخدم pm-utils لمراقبة الـ PM state
pm-is-supported --suspend

# راجع الـ OPP (Operating Performance Points) المرتبطة بالـ device
cat /sys/bus/platform/devices/<dev>/opp_table

# راجع الـ genpd (Generic Power Domain) اللي الـ device منتمي ليه
cat /sys/kernel/debug/pm_genpd/*/summary | grep <device_name>

# devlink لبعض الـ devices (network)
devlink dev info
```

---

#### 7. جدول Common Error Messages

| رسالة الـ Kernel Log | المعنى | الحل |
|---------------------|--------|------|
| `pm_clk_create: Failed to create PM clock list` | فشل تخصيص الذاكرة لقائمة الـ clocks | راجع حالة الـ memory، تأكد من إن `pm_clk_init` اتّنادى أولاً |
| `pm_clk_add: Failed to get clock 'xxx'` | الـ clock اللي بالـ con_id مش موجود في الـ DT أو مش registered | راجع الـ DT، تأكد من وجود الـ clock في `clocks` property |
| `pm_clk_suspend: Failed, clock not disabled` | clock لسه شغّال وقت الـ suspend | تحقق من إن مفيش consumer تاني بيستخدم الـ clock |
| `Unable to get clk xxx` | الـ clock provider مش loaded | تأكد من ترتيب الـ probe وإن الـ clock driver اتحمل قبل الـ consumer |
| `pm_clk_runtime_suspend returned -EBUSY` | الـ device لسه busy | راجع الـ usage count بـ `runtime_active_kids` في sysfs |
| `-EINVAL` من `pm_clk_add` | `CONFIG_PM_CLK` مش مفعّل | فعّل الـ config وأعد build الـ kernel |
| `clock_list is empty` لكن الـ device شغّال | `pm_clk_no_clocks` بترجع true خطأ | تأكد من إن `pm_clk_add` اتّنادى بنجاح |
| `devm_pm_clk_create: -ENOMEM` | نفد الذاكرة | راجع الـ memory pressure وقت الـ probe |

---

#### 8. Strategic Points لـ dump_stack() و WARN_ON()

```c
/* في pm_clk_suspend - تحقق إن الـ clock list مش فاضية بشكل غير متوقع */
int pm_clk_suspend(struct device *dev)
{
    struct pm_subsys_data *psd = dev_to_psd(dev);

    WARN_ON(!psd);  /* يجب إن psd يكون موجود */

    if (pm_clk_no_clocks(dev)) {
        /* دي حالة طبيعية، بس لو توقعت إن في clocks فهنا ممكن تحط dump */
        dev_dbg(dev, "no clocks to suspend\n");
        return 0;
    }
    /* ... */
}

/* في pm_clk_add - تحقق من الـ con_id قبل الإضافة */
int pm_clk_add(struct device *dev, const char *con_id)
{
    WARN_ON(!dev);          /* device مش NULL */
    WARN_ON(!con_id);       /* con_id مش NULL */

    if (!dev->power.subsys_data) {
        dump_stack();       /* كشف مين اتّنادى قبل pm_clk_create */
        return -EINVAL;
    }
    /* ... */
}

/* في pm_clk_runtime_resume - تحقق من الـ state بعد الـ resume */
int pm_clk_runtime_resume(struct device *dev)
{
    int ret = pm_clk_resume(dev);
    WARN_ON(ret && ret != -ENODEV);  /* أي error غير متوقع */
    return ret;
}
```

---

### Hardware Level

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

```bash
# خطوة 1: شوف الـ clock enable count في kernel
cat /sys/kernel/debug/clk/uart_clk/clk_enable_count

# خطوة 2: قارن بالـ register الفعلي في الـ clock controller
# (بافتراض إن SoC بيستخدم sunxi أو مشابه)
# قرا الـ register بـ devmem2
devmem2 0x01C20000 w   # مثال لـ CCU base على Allwinner

# خطوة 3: قارن الـ clock rate المحسوبة بالـ measured
cat /sys/kernel/debug/clk/uart_clk/clk_rate
# قيس الـ rate الفعلية بـ oscilloscope أو frequency counter
```

---

#### 2. Register Dump Techniques

```bash
# باستخدام devmem2 (أحسن أداة للـ bare metal)
# ثبّت الأداة
apt-get install devmem2   # أو build from source

# اقرا register واحد (32-bit)
devmem2 0x01C20060 w

# اقرا range من الـ registers
for offset in $(seq 0 4 64); do
    addr=$(printf "0x%08x" $((0x01C20060 + offset)))
    echo -n "$addr: "
    devmem2 $addr w 2>/dev/null | grep "Read at"
done

# باستخدام /dev/mem مباشرة (للقراءة فقط)
dd if=/dev/mem bs=4 count=16 skip=$((0x01C20060 / 4)) 2>/dev/null | xxd

# باستخدام io utility (من package devmem)
io -4 -r 0x01C20060
```

**تفسير النتيجة:**
```
Read at address 0x01C20060 (0xB6F47060): 0x00030001
                                           ^^^^ ^^^^
                                           |    |-- bit 0: clock enable (1=enabled)
                                           |------- bits 16-17: clock source select
```

---

#### 3. Logic Analyzer والـ Oscilloscope

**نقاط القياس المهمة:**

```
SoC
 │
 ├─── CLK_OUT ──────────────────► [Oscilloscope CH1]
 │                                   قيس: التردد، الـ duty cycle
 │
 ├─── CLK_ENABLE (GPIO) ────────► [Logic Analyzer CH1]
 │                                   لاحظ: timing مع الـ suspend/resume
 │
 └─── POWER_GOOD ───────────────► [Logic Analyzer CH2]
                                     تأكد: الـ power يستقر قبل enable الـ clock
```

**إعدادات Logic Analyzer:**
- Sample rate: 10x الـ clock frequency على الأقل
- Trigger: rising edge على CLK_ENABLE
- Capture window: 10ms قبل وبعد الـ trigger
- Protocol decode: I2C/SPI لو بتدبّق على clock controller يتحكم فيه بـ serial

**ما تدور عليه:**
- الـ glitch على الـ CLK_OUT وقت الـ enable/disable
- الـ runt pulses اللي ممكن تسبب حسابات خاطئة في الـ PLL
- الـ clock doesn't start within expected time بعد الـ enable

---

#### 4. Common Hardware Issues وأنماط الـ Kernel Log

| المشكلة | نمط الـ Log | التشخيص |
|---------|------------|----------|
| Clock controller مش power-on | `clk_enable failed: -ENODEV` | تحقق من الـ power rail بـ multimeter |
| PLL lock timeout | `pll: timeout waiting for lock` | راجع الـ reference clock frequency |
| Clock glitch وقت الـ resume | device يشتغل غلط بعد resume | قيّس الـ clock output بـ oscilloscope |
| Shared clock بين devices | `clk_disable: still in use` | راجع الـ enable_count في debugfs |
| Wrong clock parent | device يشتغل بـ rate غلطة | راجع `clk_parent` في debugfs |
| Missing clock in DT | `pm_clk_add: -ENOENT` | راجع الـ DT node للـ device |

**مثال sequence خاطئ في الـ log:**
```
[  12.345] pm_clk_runtime_resume: resuming device serial0
[  12.346] clk_enable: uart_clk (enable_count: 0 -> 1)
[  12.347] uart: timeout waiting for TX ready        ← clock لسه متستقرش
[  12.350] uart: FIFO error detected                 ← نتيجة الـ glitch
```

---

#### 5. Device Tree Debugging

```bash
# تحقق من الـ compiled DT
dtc -I fs /proc/device-tree 2>/dev/null | grep -A20 "uart@"

# قارن مع الـ DT source
# المهم: كل device بيستخدم pm_clock لازم يكون عنده:
# 1. clocks property
# 2. clock-names property (بيطابق الـ con_id في pm_clk_add)

# مثال DT صح:
# uart0: serial@01c28000 {
#     compatible = "allwinner,sun8i-h3-uart";
#     clocks = <&ccu CLK_BUS_UART0>, <&ccu CLK_UART0>;
#     clock-names = "bus", "mod";    ← هيتطابق مع pm_clk_add(dev, "bus")
# };

# تحقق من الـ DT الـ compiled فعلاً
cat /proc/device-tree/soc/serial@01c28000/clock-names | tr '\0' '\n'

# راجع الـ DT overlay لو موجود
ls /sys/kernel/config/device-tree/overlays/

# تأكد من الـ clocks property بيشاور على provider موجود
cat /proc/device-tree/soc/serial@01c28000/clocks | xxd
# قارن الـ phandles مع
cat /proc/device-tree/soc/clock-controller@01c20000/phandle | xxd
```

**أشيع مشكلة في DT:** اسم الـ `clock-names` مش بيطابق الـ `con_id` اللي بيتمرر في `pm_clk_add`:

```c
/* في الـ driver */
pm_clk_add(dev, "bus");   /* لازم يكون في DT: clock-names = "bus", ...; */

/* لو DT فيه: clock-names = "ahb", ...; */
/* هتيجي رسالة: pm_clk_add: Failed to get clock 'bus' */
```

---

### Practical Commands

#### الـ Commands جاهزة للـ Copy

```bash
#!/bin/bash
# pm_clock_debug.sh — سكريبت شامل لـ debugging الـ PM clock subsystem

DEVICE="${1:-}"  # اسم الـ device كـ argument اختياري

echo "=== PM Clock Debug Script ==="
echo "Date: $(date)"
echo "Kernel: $(uname -r)"
echo ""

# 1. حالة الـ clk framework
echo "--- CLK Summary ---"
cat /sys/kernel/debug/clk/clk_summary 2>/dev/null || echo "debugfs not mounted"
echo ""

# 2. لو في device محدد
if [ -n "$DEVICE" ]; then
    echo "--- Device PM Status: $DEVICE ---"
    for f in runtime_status runtime_enabled control \
              runtime_suspended_time runtime_active_time \
              runtime_usage runtime_active_kids; do
        val=$(cat /sys/bus/*/devices/$DEVICE/power/$f 2>/dev/null)
        [ -n "$val" ] && echo "$f: $val"
    done
    echo ""
fi

# 3. Power domains
echo "--- Power Domains ---"
cat /sys/kernel/debug/pm_genpd/*/summary 2>/dev/null | head -50
echo ""

# 4. فعّل الـ tracing
echo "--- Enabling PM+CLK Tracing ---"
TRACEFS=/sys/kernel/tracing
echo 0 > $TRACEFS/tracing_on
echo "" > $TRACEFS/trace

for event in clk/clk_enable clk/clk_disable clk/clk_prepare \
             clk/clk_unprepare rpm/rpm_suspend rpm/rpm_resume; do
    echo 1 > $TRACEFS/events/$event/enable 2>/dev/null && \
        echo "Enabled: $event"
done

echo 1 > $TRACEFS/tracing_on
echo "Tracing enabled. Run your test, then:"
echo "  cat $TRACEFS/trace"
echo "  echo 0 > $TRACEFS/tracing_on"
```

```bash
# تشغيل السكريبت
chmod +x pm_clock_debug.sh
./pm_clock_debug.sh "0-0050"
```

---

```bash
# اقرا الـ trace بعد ما تعمل الـ test
cat /sys/kernel/tracing/trace | grep -E "(clk_enable|clk_disable|rpm_)" | head -50
```

**تفسير الـ Output:**

```
# FIELD:    TIMESTAMP  EVENT          DATA
kworker/0:1  12.345   clk_enable:    uart_clk        ← clock اتّفعل
  kworker/0  12.346   rpm_resume:    device=serial0   ← device اتّشغل
  kworker/0  15.001   rpm_suspend:   device=serial0   ← device اتـsuspend
kworker/0:1  15.002   clk_disable:   uart_clk        ← clock اتـ disable

# لو شوفت:
# rpm_suspend بدون clk_disable ← مشكلة في pm_clk_runtime_suspend
# clk_enable بدون rpm_resume   ← الـ clock اتّفعل من مصدر تاني
```

```bash
# تحقق من الـ clock references بعد الـ test
grep uart /sys/kernel/debug/clk/clk_summary
# لو enable_count > 0 بعد الـ suspend ← leak في الـ clk_enable
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: Android TV Box على Allwinner H616 — الـ HDMI بيتجمد بعد resume

#### العنوان
HDMI output يتجمد بعد wake-up من suspend على TV box بـ Allwinner H616

#### السياق
شركة بتصنع Android TV boxes بتستخدم Allwinner H616. المنتج بيعمل كويس وهو شغال، لكن لما المستخدم يضغط power على الريموت وتتجمد الشاشة بعد الـ resume، المشكلة بتظهر بشكل عشوائي — أحيانًا الصورة ترجع، وأحيانًا تفضل سودة.

#### المشكلة
الـ HDMI controller driver بيعتمد على clock اسمه `tcon-tv` وclock تاني اسمه `hdmi`. لما الجهاز بيدخل suspend، الـ clocks بتتقفل. لكن عند الـ resume، في بعض الحالات الـ clock مش بيترجع صح لأن الـ driver مش بيتعامل مع `pm_clk_resume()` صح — بيرجع `0` (success) من غير ما يتحقق إن الـ clock اتفتح فعلًا.

#### التحليل

الـ header بيعرف:

```c
#ifdef CONFIG_PM_CLK
extern int pm_clk_resume(struct device *dev);
extern int pm_clk_suspend(struct device *dev);
#else
#define pm_clk_suspend  NULL
#define pm_clk_resume   NULL
#endif
```

لما `CONFIG_PM_CLK` مش متفعل في الـ kernel config بتاع الـ vendor، الـ `pm_clk_suspend` و`pm_clk_resume` بيتحولوا لـ `NULL`. الـ pm_domain بيحاول يستدعيهم — لو الـ ops pointer هو `NULL` والـ framework مش بيتحقق، الـ clocks مش بتتوقف ولا بترجع بشكل منظم.

حتى لو `CONFIG_PM_CLK` متفعل، المشكلة التانية: الـ driver بيستخدم `pm_clk_add()` بس بيبعت `con_id` غلط:

```c
/* driver code — wrong con_id string */
pm_clk_add(dev, "hdmi-clk");   /* DT uses "hdmi" not "hdmi-clk" */
```

النتيجة: `pm_clk_add()` بيرجع `0` حتى لو الـ clock مش اتلقاش، لأن الـ lookup بيسجل placeholder وبيفشل lazy، مش فوري.

#### الحل

**1. تحقق من الـ kernel config:**

```bash
grep CONFIG_PM_CLK /path/to/.config
# يجب يكون:
# CONFIG_PM_CLK=y
```

**2. صحّح الـ con_id في الـ driver:**

```c
/* correct: match DT clock-names exactly */
ret = pm_clk_add(dev, "hdmi");
if (ret) {
    dev_err(dev, "failed to add hdmi clock to pm: %d\n", ret);
    return ret;
}
```

**3. استخدم `devm_pm_clk_create()` بدل `pm_clk_create()` عشان تضمن الـ cleanup:**

```c
ret = devm_pm_clk_create(dev);
if (ret)
    return ret;
```

**4. في الـ `dev_pm_ops`، استخدم الماكرو الجاهز:**

```c
static const struct dev_pm_ops hdmi_pm_ops = {
    USE_PM_CLK_RUNTIME_OPS   /* expands to runtime_suspend/resume */
    .suspend = pm_clk_suspend,
    .resume  = pm_clk_resume,
};
```

#### الدرس المستفاد
الـ `con_id` في `pm_clk_add()` لازم يتطابق حرف بحرف مع `clock-names` في الـ Device Tree. والـ `CONFIG_PM_CLK` لازم يكون enabled وإلا كل الـ pm_clk API بيتحول stub ولا بيعمل حاجة.

---

### السيناريو الثاني: Industrial Gateway على RK3562 — الـ UART بيوقع بعد runtime suspend

#### العنوان
UART على RK3562 بيبعت garbage data أو بيوقع بعد idle timeout في industrial gateway

#### السياق
gateway صناعي بيستخدم RK3562، بيتواصل مع sensors عبر RS-485 (UART2). الـ runtime PM متفعل عشان يوفر power. بعد فترة idle، الـ UART بيدخل runtime suspend. لما يجي request جديد من الـ sensor، أحيانًا الـ UART بيبعت garbage أو الـ read بيفشل بـ `-EIO`.

#### المشكلة
الـ driver بيستخدم `USE_PM_CLK_RUNTIME_OPS` كـ shortcut:

```c
static const struct dev_pm_ops uart_pm_ops = {
    USE_PM_CLK_RUNTIME_OPS
};
```

الماكرو بيتوسع لـ:

```c
.runtime_suspend = pm_clk_runtime_suspend,
.runtime_resume  = pm_clk_runtime_resume,
```

المشكلة: الـ driver مش بيحفظ حالة الـ UART registers قبل ما الـ clock يتقفل في `pm_clk_runtime_suspend`. لما الـ clock يرجع في `pm_clk_runtime_resume`، الـ UART controller بيرجع لـ reset state وبيفقد الـ baud rate وباقي إعدادات الـ port.

#### التحليل

`pm_clk_runtime_suspend()` بيعمل حاجة واحدة بس: بيقفل كل الـ clocks المسجلة في `dev->power.subsys_data->clock_list`. ما بيعملش أي save/restore للـ registers.

```c
/* pm_clock.h exposes: */
extern int pm_clk_runtime_suspend(struct device *dev);
extern int pm_clk_runtime_resume(struct device *dev);
```

الـ driver المفروض يضيف حفظ/استعادة الـ register state حواليهم:

#### الحل

```c
static int my_uart_runtime_suspend(struct device *dev)
{
    struct uart_priv *priv = dev_get_drvdata(dev);

    /* save UART config before clocks die */
    priv->saved_lcr = readl(priv->base + UART_LCR);
    priv->saved_mcr = readl(priv->base + UART_MCR);
    priv->saved_div = readl(priv->base + UART_DIV);

    return pm_clk_runtime_suspend(dev);  /* now safe to kill clocks */
}

static int my_uart_runtime_resume(struct device *dev)
{
    struct uart_priv *priv = dev_get_drvdata(dev);
    int ret;

    ret = pm_clk_runtime_resume(dev);    /* restore clocks first */
    if (ret)
        return ret;

    /* restore UART config after clocks are back */
    writel(priv->saved_lcr, priv->base + UART_LCR);
    writel(priv->saved_mcr, priv->base + UART_MCR);
    writel(priv->saved_div, priv->base + UART_DIV);

    return 0;
}

static const struct dev_pm_ops uart_pm_ops = {
    .runtime_suspend = my_uart_runtime_suspend,
    .runtime_resume  = my_uart_runtime_resume,
};
```

**للتحقق من الـ runtime PM events:**

```bash
cat /sys/bus/platform/devices/fe670000.serial/power/runtime_status
cat /sys/bus/platform/devices/fe670000.serial/power/runtime_suspended_time
```

#### الدرس المستفاد
الـ `USE_PM_CLK_RUNTIME_OPS` ماكرو مناسب بس للـ devices اللي مش محتاجة register save/restore. الـ UART وSPI وI2C كلهم بيحتاجوا wrapper functions تعمل save/restore قبل وبعد `pm_clk_runtime_suspend/resume`.

---

### السيناريو الثالث: IoT Sensor Board على STM32MP1 — memory leak في bring-up

#### العنوان
Memory leak واضح في kernel أثناء module reload على STM32MP1 IoT board

#### السياق
مهندس بيعمل bring-up لـ IoT sensor board بـ STM32MP157. الـ sensor driver بيتحمل كـ kernel module. أثناء testing، المهندس لاحظ إن كل مرة بيعمل `rmmod` و`insmod` للـ driver، الـ kernel memory بتزيد — leak واضح.

#### المشكلة
الـ driver بيستدعي `pm_clk_create()` في `probe()` لكن بيستدعي `pm_clk_destroy()` في المكان الغلط، أو مش بيستدعيها خالص في بعض الـ error paths.

#### التحليل

```c
/* pm_clock.h */
extern int pm_clk_create(struct device *dev);
extern void pm_clk_destroy(struct device *dev);
extern int devm_pm_clk_create(struct device *dev);
```

الـ `pm_clk_create()` بيخصص `dev->power.subsys_data` والـ `clock_list`. لو الـ `probe()` فشل بعد الـ `pm_clk_create()` ومش في error handling يستدعي `pm_clk_destroy()`، الـ memory بيتخسر.

الـ driver الغلط:

```c
static int sensor_probe(struct platform_device *pdev)
{
    int ret;

    ret = pm_clk_create(&pdev->dev);   /* allocates subsys_data */
    if (ret)
        return ret;

    ret = pm_clk_add(&pdev->dev, "sensor-clk");
    if (ret)
        return ret;   /* BUG: forgot pm_clk_destroy() on error path */

    /* ... more init ... */
    return 0;
}

static int sensor_remove(struct platform_device *pdev)
{
    pm_clk_destroy(&pdev->dev);
    return 0;
}
```

#### الحل

**الحل الأمثل: استخدم `devm_pm_clk_create()` بدلًا من `pm_clk_create()`:**

```c
static int sensor_probe(struct platform_device *pdev)
{
    int ret;

    /*
     * devm_pm_clk_create() ties the lifecycle of pm_clk data
     * to the device — auto-destroyed on remove/probe-failure.
     */
    ret = devm_pm_clk_create(&pdev->dev);
    if (ret)
        return ret;

    ret = pm_clk_add(&pdev->dev, "sensor-clk");
    if (ret)
        return ret;   /* safe: devm will clean up pm_clk data */

    return 0;
}

/* no need to call pm_clk_destroy() in remove() when using devm variant */
```

**للتحقق من الـ leak:**

```bash
# قبل وبعد module reload
cat /proc/meminfo | grep MemFree
cat /sys/kernel/debug/kmemleak   # إذا CONFIG_DEBUG_KMEMLEAK=y
```

#### الدرس المستفاد
دايمًا استخدم `devm_pm_clk_create()` بدل `pm_clk_create()` عشان الـ kernel يتكفل بالـ cleanup تلقائيًا في كل الـ error paths وعند الـ remove. الـ `devm_*` variants موجودة بالضبط عشان تتجنب ده النوع من الـ bugs.

---

### السيناريو الرابع: Automotive ECU على i.MX8 — الـ SPI flash مش بيتعرف عليه بعد cold boot

#### العنوان
SPI NOR flash مش بيتعرف عليه في أول boot على i.MX8-based automotive ECU

#### السياق
ECU سيارة بيستخدم i.MX8QM، فيه SPI NOR flash متوصل بـ ECSPI3 بيخزن الـ calibration data. المشكلة بتحصل بس في أول boot بعد power-on كامل (cold boot)، لكن بعد restart بيشتغل تمام.

#### المشكلة
الـ SPI driver بيستخدم `pm_clk_no_clocks()` للتحقق إن الـ clock list مش فاضية قبل ما يبدأ الـ transfer:

```c
/* inside spi_transfer_one() */
if (pm_clk_no_clocks(dev)) {
    dev_err(dev, "no clocks available, cannot transfer\n");
    return -ENODEV;
}
```

لكن المشكلة: الـ `pm_clk_no_clocks()` بترجع `true` حتى لو الـ clock list بيتملى حاليًا (race condition في الـ probe).

#### التحليل

```c
/* pm_clock.h */
static inline bool pm_clk_no_clocks(struct device *dev)
{
    return dev && dev->power.subsys_data
        && list_empty(&dev->power.subsys_data->clock_list);
}
```

الـ function دي بتتحقق من `clock_list` في اللحظة دي. المشكلة إن:

1. الـ `pm_clk_create()` اتنادى.
2. الـ `pm_clk_add()` لـ `"ecspi-clk"` لسه ما اتنادتش (أو بتنادى async).
3. في النفس اللحظة، الـ SPI framework حاول يعمل transfer → `pm_clk_no_clocks()` بترجع `true` → فشل.

في cold boot، الـ probe sequence بطيء أكتر بسبب الـ clock framework initialization order.

#### الحل

**تأكد إن كل الـ clocks اتضافت قبل ما الـ device يبقى active:**

```c
static int ecspi_probe(struct platform_device *pdev)
{
    int ret;

    ret = devm_pm_clk_create(&pdev->dev);
    if (ret)
        return ret;

    /* add ALL clocks before registering the SPI controller */
    ret = pm_clk_add(&pdev->dev, "ipg");
    if (ret)
        return dev_err_probe(&pdev->dev, ret, "failed to add ipg clock\n");

    ret = pm_clk_add(&pdev->dev, "per");
    if (ret)
        return dev_err_probe(&pdev->dev, ret, "failed to add per clock\n");

    /* NOW register the SPI controller — clocks are ready */
    return spi_register_controller(ctlr);
}
```

**تحقق بـ debugfs:**

```bash
# تحقق من clock_list للـ device
cat /sys/kernel/debug/clk/ecspi3_ipg/clk_enable_count
cat /sys/kernel/debug/clk/ecspi3_per/clk_enable_count
```

**أو استخدم `of_pm_clk_add_clks()` لإضافة كل الـ clocks من الـ DT دفعة واحدة:**

```c
/* adds all clocks listed in DT "clocks" property at once */
ret = of_pm_clk_add_clks(&pdev->dev);
if (ret && ret != -ENODEV)
    return ret;
```

#### الدرس المستفاد
`pm_clk_no_clocks()` snapshot check — مش guarantee. لازم تضيف كل الـ clocks لـ`clock_list` قبل ما تسجل الـ device مع أي framework تاني. الـ `of_pm_clk_add_clks()` أسرع وأأمن لأنها بتعمل الـ loop الداخلي.

---

### السيناريو الخامس: Custom Board Bring-up على AM62x — bus driver بيديه الـ clock لكل الـ children

#### العنوان
إضافة pm_clk_notifier لـ platform bus على AM62x لإدارة clocks الـ child devices تلقائيًا

#### السياق
مهندس BSP بيعمل bring-up لبورد custom بـ AM62x (TI Sitara). في الـ SoC ده، في IP blocks كتير (I2C, UART, SPI) بيشاركوا clock domain واحد. الـ PM team عايزة solution generalized بدل ما كل driver يعمل `pm_clk_add()` بنفسه.

#### المشكلة
كل driver بيضطر يعمل نفس الـ boilerplate:

```c
devm_pm_clk_create(dev);
pm_clk_add(dev, "fck");
pm_clk_add(dev, "ick");
```

لو driver جديد اتضاف ونسي الـ boilerplate ده، الـ power management بيبوظ. المهندس عايز الـ bus layer يعمل ده تلقائيًا لكل child device.

#### التحليل

الـ header بيوفر `pm_clk_add_notifier()` بالضبط لده:

```c
/* pm_clock.h */
#ifdef CONFIG_HAVE_CLK
extern void pm_clk_add_notifier(const struct bus_type *bus,
                struct pm_clk_notifier_block *clknb);
#endif
```

والـ `struct pm_clk_notifier_block`:

```c
struct pm_clk_notifier_block {
    struct notifier_block nb;      /* linked into bus notifier chain */
    struct dev_pm_domain *pm_domain; /* optional pm_domain to assign */
    char *con_ids[];               /* flexible array of clock con_ids */
};
```

الـ mechanism: لما device يتضاف للـ bus، الـ notifier بيتنادى، وبيضيف الـ clocks المحددة في `con_ids[]` لكل device تلقائيًا.

#### الحل

**في الـ bus/platform init code:**

```c
#include <linux/pm_clock.h>

/* define the notifier block with clock names for AM62x peripherals */
static struct {
    struct pm_clk_notifier_block clknb;
    const char *ids[3];   /* must account for flexible array */
} am62x_clknb = {
    .clknb = {
        .nb = {
            .notifier_call = /* filled by pm_clk_add_notifier internally */,
        },
        .pm_domain = &am62x_pm_domain,  /* optional */
        .con_ids = { "fck", "ick", NULL },
    },
};

static int am62x_bus_init(void)
{
    /*
     * Register notifier — every device added to platform_bus_type
     * will automatically get "fck" and "ick" added to its pm clock list.
     */
    pm_clk_add_notifier(&platform_bus_type, &am62x_clknb.clknb);
    return 0;
}
postcore_initcall(am62x_bus_init);
```

**للتحقق إن الـ notifier شتغل:**

```bash
# بعد boot، تحقق من أي device على platform bus
ls /sys/bus/platform/devices/

# تحقق من power subsys_data (indirect — عبر runtime PM activity)
cat /sys/bus/platform/devices/2000000.i2c/power/runtime_status

# enable runtime PM verbose logging
echo 1 > /sys/module/pm_runtime/parameters/debug
dmesg | grep -i "pm_clk"
```

**نقطة مهمة: الـ `con_ids` flexible array**

```c
struct pm_clk_notifier_block {
    struct notifier_block nb;
    struct dev_pm_domain *pm_domain;
    char *con_ids[];    /* flexible array — MUST be last member */
};
```

الـ allocation لازم تحسب الـ array كاملة:

```c
/* allocate for 3 con_ids (2 names + NULL terminator) */
struct pm_clk_notifier_block *clknb;
clknb = kzalloc(sizeof(*clknb) + 3 * sizeof(char *), GFP_KERNEL);
clknb->con_ids[0] = "fck";
clknb->con_ids[1] = "ick";
clknb->con_ids[2] = NULL;
```

#### الدرس المستفاد
الـ `pm_clk_add_notifier()` مع `pm_clk_notifier_block` هي الطريقة الصح لأتمتة إدارة الـ clocks على مستوى الـ bus كله، بدل تكرار الـ boilerplate في كل driver. الـ `con_ids[]` flexible array لازم يتنتهي بـ `NULL` وحجمه يتحسب صح في الـ allocation.
## Phase 7: مصادر ومراجع

### مقالات LWN.net

دي أهم المقالات اللي بتتكلم عن الـ runtime PM وإدارة الـ clocks في الـ kernel:

| المقال | الأهمية |
|--------|---------|
| [PM / Runtime: Generic clock manipulation routines for runtime PM (v3)](https://lwn.net/Articles/440445/) | **الأهم** — المقال الأصلي اللي قدّم الـ `pm_clk_*` API من Rafael J. Wysocki |
| [PM / Domains: Support for generic I/O PM domains](https://lwn.net/Articles/449302/) | ربط الـ `pm_clock` بنظام الـ power domains |
| [PM: Suspend/resume and runtime PM for clock sources/clock event devices in PM domains](https://lwn.net/Articles/509887/) | تفاصيل تعامل الـ clock devices مع PM domains |
| [PM: Introduce core framework for run-time PM of I/O devices (rev. 17)](https://lwn.net/Articles/347574/) | الأساس النظري للـ runtime PM اللي بيشتغل فوقيه الـ pm_clock |
| [A common clock framework](https://lwn.net/Articles/472998/) | شرح الـ Common Clock Framework (CCF) اللي بيتعامل معاه `pm_clk_add_clk()` |
| [Active state management of power domains](https://lwn.net/Articles/744047/) | التطور الأحدث للـ genpd وعلاقته بالـ clocks |
| [Linux power management: The documentation I wanted to read](https://lwn.net/Articles/505683/) | نظرة شاملة على PM infrastructure في الـ kernel |

---

### التوثيق الرسمي في الـ Kernel

**الـ `Documentation/` paths المهمة:**

```
Documentation/power/runtime_pm.rst       ← الـ runtime PM framework بالكامل
Documentation/driver-api/pm/devices.rst  ← device PM basics
Documentation/driver-api/clk.rst         ← Common Clock Framework API
```

**روابط مباشرة على docs.kernel.org:**

- [Runtime Power Management Framework for I/O Devices](https://docs.kernel.org/power/runtime_pm.html) — بيشرح الـ callbacks زي `runtime_suspend` و `runtime_resume` اللي `USE_PM_CLK_RUNTIME_OPS` بتسندّهم
- [Device Power Management Basics](https://docs.kernel.org/driver-api/pm/devices.html) — شرح `dev->power` والـ `subsys_data` اللي بتستخدمها `pm_clk_no_clocks()`
- [The Common Clk Framework](https://docs.kernel.org/driver-api/clk.html) — بيشرح الـ `struct clk` اللي `pm_clk_add_clk()` بتاخده

**الـ Kconfig:**
- [CONFIG_PM_CLK — Linux Kernel Driver DataBase](https://cateee.net/lkddb/web-lkddb/PM_CLK.html) — بيوضح الـ dependencies والـ platforms اللي بتفعّل الـ option دي

---

### الـ Source Files المهمة في الـ Kernel

```
include/linux/pm_clock.h          ← الـ header الرئيسي (الملف اللي بندرسه)
drivers/base/power/clock_ops.c    ← الـ implementation الكاملة للـ pm_clk_*
drivers/base/power/domain.c       ← الـ genpd اللي بيستخدم pm_clock
include/linux/pm_domain.h         ← struct dev_pm_domain المرتبط بـ pm_clk_notifier_block
```

---

### Commits مهمة في الـ Git History

دي الـ commits الأساسية اللي شكّلت الـ pm_clock subsystem:

| الوصف | المصدر |
|-------|--------|
| الـ commit الأولاني لـ `clock_ops.c` — إدخال `pm_runtime_clk_add()` | [LWN patch thread](https://lwn.net/Articles/440445/) |
| إضافة `pm_clk_add_clk()` — دعم إضافة `struct clk *` مباشرةً | [SysTutorials kernel reference](https://www.systutorials.com/linux-kernel-pm-clk-fix-clock-error-check-in-__pm_clk_add/) |
| `of_pm_clk_add_clks()` — دعم Device Tree لجلب الـ clocks تلقائيًا | مرتبط بتطور الـ DT binding في الـ ARM subsystem |

للبحث في تاريخ الـ commits مباشرة:
```bash
git log --oneline -- drivers/base/power/clock_ops.c
git log --oneline -- include/linux/pm_clock.h
```

أو باستخدام:
- [Linux Commits Search (Typesense)](https://linux-commits-search.typesense.org/) — ابحث عن `pm_clk` أو `clock_ops`
- [GitHub torvalds/linux commits](https://github.com/torvalds/linux/commits/master/) — فلتر على الملف المطلوب

---

### نقاشات Mailing List

- [Re: PATCH v10 — amba: Don't unprepare clocks if driver wants IRQ safe runtime PM](https://yhbt.net/lore/all/3262760.d60MN56YDG@vostro.rjw.lan/) — نقاش عملي لـ Rafael Wysocki عن تفاصيل الـ pm_clock مع الـ AMBA bus
- [Patchwork: clk: Add support for runtime PM](https://patchwork.kernel.org/project/linux-clk/patch/1503302703-13801-2-git-send-email-m.szyprowski@samsung.com/) — نقاش عن دمج الـ runtime PM مع الـ clock core
- [Patchwork: imx8 new clock binding for better pm support](https://patchwork.kernel.org/project/linux-arm-kernel/cover/1568081408-26800-1-git-send-email-aisheng.dong@nxp.com/) — مثال حقيقي على استخدام `of_pm_clk_add_clks()` في الـ NXP i.MX8

---

### كتب مُوصى بيها

#### Linux Device Drivers (LDD3)
- **المؤلفون:** Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- الفصل المتعلق: **Chapter 14 — The Linux Device Model**
- بيشرح الـ `struct device`، الـ `bus_type`، وبنية الـ power management اللي بتعتمد عليها `pm_clk_add_notifier()`
- متاح مجانًا: [lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development (Robert Love)
- **الطبعة الثالثة** — Addison-Wesley
- الفصل المتعلق: **Chapter 14 — The Block I/O Layer** + **Chapter 16 — Modules**
- للفهم العام لبنية الـ kernel، والـ subsystems، وتقنيات الـ runtime hooks

#### Embedded Linux Primer (Christopher Hallinan)
- **الطبعة الثانية** — Prentice Hall
- الفصل المتعلق: **Chapter 15 — Embedded Linux Power Management**
- بيتكلم عن إدارة الـ clocks في الـ embedded platforms، وعلاقة الـ `pm_clk_*` بتقليل الاستهلاك على الـ SoCs

---

### مصادر eLinux.org

- [Power Management — eLinux.org](https://elinux.org/Power_Management) — نقطة انطلاق شاملة للـ embedded Linux PM
- [Power Management Framework — eLinux.org](https://elinux.org/Power_Management_Framework) — بيتكلم عن الـ clock gating وعلاقته بالـ `pm_clk_suspend/resume`
- [Power Management Definition Of Terms — eLinux.org](https://elinux.org/Power_Management_Definition_Of_Terms) — مسرد مصطلحات مفيد
- [Power Management Presentations — eLinux.org](https://elinux.org/Power_Management_Presentations) — slides من مؤتمرات LinuxCon وELC عن الـ clock framework والـ PM

---

### مصادر KernelNewbies.org

- [KernelGlossary — kernelnewbies.org](https://kernelnewbies.org/KernelGlossary) — تعريفات للمصطلحات زي runtime PM، clock gating، power domain
- [LinuxChanges — kernelnewbies.org](https://kernelnewbies.org/LinuxChanges) — متابعة التغييرات في الـ pm_clock عبر الـ kernel versions
- [Linux_2_6_32 — kernelnewbies.org](https://kernelnewbies.org/Linux_2_6_32) — الـ version اللي شهد تطورات كبيرة في الـ runtime PM

---

### مصطلحات للبحث

لو عايز تلاقي معلومات أكتر، استخدم الـ search terms دي:

```
pm_clk_add runtime PM clock gating Linux kernel
pm_clock_ops clock_list subsys_data kernel
dev_pm_domain clock management embedded Linux
CONFIG_PM_CLK Kconfig drivers/base/power/clock_ops.c
of_pm_clk_add_clks device tree clock binding
pm_clk_notifier_block bus notifier power management
devm_pm_clk_create managed resource clock suspend
Rafael Wysocki runtime PM clock manipulation patch
```

**روابط بحث مباشرة:**
- [LWN.net search: pm_clk](https://lwn.net/Search/DoSearch?words=pm_clk) — آخر المقالات المتعلقة
- [lore.kernel.org search: pm_clock](https://lore.kernel.org/search/?q=pm_clk&a=s) — أرشيف الـ mailing lists الرسمي
- [Elixir Cross Referencer: pm_clock.h](https://elixir.bootlin.com/linux/latest/source/include/linux/pm_clock.h) — لمتابعة الـ callers والـ users للـ API في كل الـ kernel
## Phase 8: Writing simple module

### الفكرة

**`pm_clk_add`** هي function بتضيف clock لجهاز معين عن طريق الـ PM clock list بتاعته. هنعمل **kprobe** عليها عشان نشوف إيه الـ device اللي بيضيف clock وإيه الـ connection ID بتاعه — ده مفيد جداً في debugging مشاكل الـ power management على embedded systems.

---

### الـ Module الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_pm_clk_add.c
 * Hooks pm_clk_add() to log every device that adds a PM clock entry.
 */

#include <linux/module.h>      /* MODULE_LICENSE, module_init/exit */
#include <linux/kernel.h>      /* pr_info */
#include <linux/kprobes.h>     /* kprobe API */
#include <linux/device.h>      /* struct device, dev_name() */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on pm_clk_add — log PM clock additions per device");

/*
 * pre_handler runs just before pm_clk_add() executes.
 * At this point registers still hold the original arguments:
 *   arg0 (RDI on x86_64) = struct device *dev
 *   arg1 (RSI on x86_64) = const char *con_id
 */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /* Retrieve the two arguments from the call registers */
    struct device *dev   = (struct device *)regs->di;  /* first arg */
    const char    *con   = (const char    *)regs->si;  /* second arg */

    if (!dev)
        return 0;

    /* Print device name and clock connection-id (con_id may be NULL) */
    pr_info("pm_clk_add: dev=%s  con_id=%s\n",
            dev_name(dev),
            con ? con : "(null)");

    return 0; /* 0 = let the original function continue normally */
}

/* kprobe descriptor — we attach to pm_clk_add by symbol name */
static struct kprobe kp = {
    .symbol_name = "pm_clk_add",
    .pre_handler = handler_pre,
};

static int __init pmclk_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("pm_clk_add kprobe: register failed, ret=%d\n", ret);
        return ret;
    }

    pr_info("pm_clk_add kprobe: planted at %p\n", kp.addr);
    return 0;
}

static void __exit pmclk_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("pm_clk_add kprobe: removed\n");
}

module_init(pmclk_probe_init);
module_exit(pmclk_probe_exit);
```

---

### الـ Makefile

```makefile
obj-m += kprobe_pm_clk_add.o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

---

### شرح كل جزء

#### الـ Includes

| Header | السبب |
|---|---|
| `linux/module.h` | الـ macros الأساسية لأي kernel module |
| `linux/kernel.h` | `pr_info` / `pr_err` للـ logging |
| `linux/kprobes.h` | تعريف `struct kprobe` وكل الـ API بتاعته |
| `linux/device.h` | تعريف `struct device` و `dev_name()` |

---

#### الـ `handler_pre` — الـ Callback

الـ pre_handler بيتنادى قبل ما `pm_clk_add` تنفذ، يعني الـ arguments لسه موجودين في الـ registers. بنقرأ `regs->di` و `regs->si` عشان ده الـ calling convention على x86\_64 (RDI = arg1, RSI = arg2).

بنطبع اسم الـ device باستخدام `dev_name()` عشان الـ `dev->name` مش دايماً مباشرة قابل للقراءة، والـ `con_id` ممكن يكون NULL لو الـ driver مش بيحدد اسم معين للـ clock.

---

#### الـ `struct kprobe kp`

بنحدد الـ symbol بالاسم `"pm_clk_add"` وده أسهل وأأمن من تحديد عنوان ثابت — الـ kernel بيحسب العنوان وقت التسجيل. الـ `pre_handler` هو الـ callback اللي هيتنادى.

---

#### الـ `module_init` و `module_exit`

**`register_kprobe`** بتزرع الـ breakpoint في الذاكرة وقت الـ load، ولو فشلت بنرجع الـ error بدل ما الـ module يفضل شغال بدون hook. **`unregister_kprobe`** في الـ exit ضروري جداً عشان لو مسحناش الـ kprobe، الـ kernel هيفضل يجري الـ callback على function اتمسحت من الذاكرة وده kernel panic فوري.

---

### تشغيل وتجربة

```bash
# بناء الـ module
make

# تحميله
sudo insmod kprobe_pm_clk_add.ko

# مراقبة الـ output (في terminal تاني أو نفس الـ terminal)
sudo dmesg -w | grep pm_clk_add

# لتشغيل device يستخدم pm_clk_add — مثلاً على Raspberry Pi / OMAP:
# echo suspend > /sys/bus/platform/devices/<device>/power/control
# أو أضف driver يحتاج PM clocks

# إزالة الـ module
sudo rmmod kprobe_pm_clk_add
```

**مثال على الـ output المتوقع:**

```
[  123.456789] pm_clk_add kprobe: planted at ffffffffc0123456
[  124.001234] pm_clk_add: dev=48004000.i2c  con_id=fck
[  124.002345] pm_clk_add: dev=48004000.i2c  con_id=ick
[  130.999000] pm_clk_add kprobe: removed
```

ده بيوضح إن الـ I2C controller بيضيف clock اسمه `fck` (functional clock) و `ick` (interface clock) — نفس اللي بتشوفه في الـ device tree على OMAP SoCs.
