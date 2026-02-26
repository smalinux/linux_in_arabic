## Phase 1: الصورة الكبيرة ببساطة

### الـ Subsystem

الـ file ده جزء من **Common Clock Framework (CCK)** — الـ subsystem اللي بيتحكم في كل الـ clocks جوه الـ Linux kernel. بالتحديد، ده الـ "managed resources" layer فوق الـ clock API، اللي بيتعامل مع الـ **devres** (Device Resource Management).

---

### المشكلة اللي بيحلها الملف ده

#### القصة

تخيل إنك بتكتب driver لـ SoC زي Raspberry Pi أو أي ARM chip. الـ chip دي فيها clocks كتير — clock للـ USB، clock للـ SPI، clock للـ I2C، إلخ. لما الـ driver بيشتغل بيعمل:

```
1. جيب الـ clock → clk_get()
2. جهّزه       → clk_prepare()
3. شغّله        → clk_enable()
4. اشتغل...
5. وقّفه        → clk_disable()
6. افرد التجهيز → clk_unprepare()
7. حرّر الـ clock → clk_put()
```

المشكلة: لو حصل **error** في أي خطوة، لازم تتذكر تعمل cleanup لكل اللي عملته قبله. لو نسيت `clk_put()` مثلاً — **memory leak**. لو نسيت `clk_disable()` — الـ clock فاضل شغال بعد ما الـ device اتـ unbind — **power waste** وبعض الأحيان **hardware damage**.

#### الحل: الـ devres pattern

الـ **devres** (Device Resources) هو نظام في الـ kernel بيقول: "اربط الـ resource بالـ device نفسه، ولما الـ device يتحرر أو يفشل، الـ kernel يـ cleanup كل حاجة أوتوماتيك."

الـ file ده `clk-devres.c` بيطبّق الـ devres pattern على الـ clock API بالكامل — ده اللي معناه الـ prefix الـ `devm_` في كل الـ functions.

---

### الصورة الكبيرة بالتشبيه

تخيل إنك بتأجر أدوات من محل. عندك طريقتين:

**الطريقة القديمة (بدون devm):**
- تأخد المفتاح الإنجليزي، تشتغل، لازم تفتكر ترجعه بنفسك.
- لو نسيت — المحل خسران مفتاح.

**الطريقة الجديدة (مع devm):**
- تأخد المفتاح من "دلوقتي لحد ما تخرج من المحل".
- لما تخرج (device removed / driver unbind) — المحل بياخد مفتاحه أوتوماتيك حتى لو نسيت.

الـ `devres_alloc` + `devres_add` ده زي إنك بتسجّل في سجل المحل: "الأداة دي محجوزة لفلان، وفي حالة مسح المحجوزات اعمل كذا."

---

### ماذا يفعل الملف تحديداً

#### 1. إدارة Clock واحد

```c
// الـ state بيحتفظ بـ pointer للـ clock + دالة الـ cleanup
struct devm_clk_state {
    struct clk *clk;
    void (*exit)(struct clk *clk);  // يتنفذ لما الـ device يتحرر
};
```

الـ `__devm_clk_get()` هي الـ engine الأساسية — بتاخد:
- دالة `get` → عشان تجيب الـ clock
- دالة `init` → عشان تشغّله (prepare/enable)
- دالة `exit` → عشان تقفله لما الـ device يتحرر

الـ functions العامة المبنية عليها:

| Function | ماذا تفعل؟ | الـ exit callback |
|---|---|---|
| `devm_clk_get()` | جيب الـ clock بس | `clk_put` |
| `devm_clk_get_prepared()` | جيب + prepare | `clk_unprepare` |
| `devm_clk_get_enabled()` | جيب + prepare + enable | `clk_disable_unprepare` |
| `devm_clk_get_optional()` | جيب (مش لازم يكون موجود) | `clk_put` |
| `devm_clk_get_optional_prepared()` | optional + prepare | `clk_unprepare` |
| `devm_clk_get_optional_enabled()` | optional + prepare + enable | `clk_disable_unprepare` |
| `devm_clk_get_optional_enabled_with_rate()` | optional + set rate + enable | `clk_disable_unprepare` |

#### 2. إدارة Clocks كتير دفعة واحدة (Bulk)

```c
// بدل ما تجيب كل clock على حدة
struct clk_bulk_data clks[] = {
    { .id = "usb" },
    { .id = "spi" },
    { .id = "i2c" },
};
devm_clk_bulk_get(dev, ARRAY_SIZE(clks), clks);
// تلاتة clocks اتجابوا وهيتحرروا تلقائيًا مع الـ device
```

الـ bulk functions:

| Function | الوصف |
|---|---|
| `devm_clk_bulk_get()` | جيب عدد محدد من الـ clocks |
| `devm_clk_bulk_get_optional()` | نفس الكلام بس optional |
| `devm_clk_bulk_get_optional_enable()` | optional + enable |
| `devm_clk_bulk_get_all()` | جيب كل الـ clocks المرتبطة بالـ device |
| `devm_clk_bulk_get_all_enabled()` | جيب الكل + enable |

#### 3. الـ Cleanup التلقائي

```
Device removed / probe() failed
        ↓
kernel يمشي على قايمة الـ devres للـ device ده
        ↓
يلاقي clock state مسجّل
        ↓
يستدعي devm_clk_release()
        ↓
state->exit(clk)  → يوقف الـ clock
clk_put(clk)      → يحرر الـ reference
```

---

### الـ ASCII Diagram: تدفق الـ devm_clk_get_enabled

```
Driver probe()
    │
    ▼
devm_clk_get_enabled(dev, "uart")
    │
    ├─► devres_alloc(devm_clk_release, ...)  ← allocate state node
    │
    ├─► clk_get(dev, "uart")                 ← احصل على clock handle
    │
    ├─► clk_prepare_enable(clk)              ← prepare + enable
    │
    ├─► state = { clk, exit=clk_disable_unprepare }
    │
    └─► devres_add(dev, state)               ← سجّل في قايمة الـ device

         [Device works normally]

Device removed / driver unbind
    │
    ▼
devres cleanup loop
    │
    └─► devm_clk_release(dev, state)
            │
            ├─► clk_disable_unprepare(clk)   ← وقّف الـ clock
            └─► clk_put(clk)                  ← حرّر الـ reference
```

---

### ليه ده مهم؟

- **Safety**: مستحيل يحصل clock leak لو استخدمت الـ `devm_` variants.
- **Simplicity**: الـ driver كوده بيبقى أبسط بكتير — مش محتاج `remove()` handler معقد.
- **Correctness**: ترتيب الـ cleanup صح دايمًا — الـ kernel بيعكس ترتيب الـ registration.
- **Power Management**: الـ clocks بتتوقف صح لما الـ device يتحرر — مش هيبقى clocks شغالة من غير سبب.

---

### الملفات المرتبطة اللي المفروض تعرفها

#### Core Framework

| الملف | الدور |
|---|---|
| `drivers/clk/clk.c` | الـ core الأساسي للـ Common Clock Framework — كل logic الـ enable/disable/prepare |
| `drivers/clk/clk-bulk.c` | تنفيذ الـ bulk operations (`clk_bulk_get`, `clk_bulk_put`, ...) |
| `drivers/clk/clk-devres.c` | **الملف ده** — الـ managed/devm wrapper |

#### Headers

| الملف | الدور |
|---|---|
| `include/linux/clk.h` | الـ public API للـ clock consumers — كل الـ declarations |
| `include/linux/clk-provider.h` | API للـ clock providers (اللي بيسجّل الـ clocks) |
| `include/linux/device/devres.h` | الـ devres API (`devres_alloc`, `devres_add`, ...) |

#### Hardware Drivers (أمثلة)

| الملف | الدور |
|---|---|
| `drivers/clk/bcm/clk-bcm2835.c` | clock driver لـ Raspberry Pi |
| `drivers/clk/imx/` | clock drivers لـ NXP i.MX SoCs |
| `drivers/clk/at91/` | clock drivers لـ Microchip/Atmel SoCs |

---

### ملخص سريع

**الـ `clk-devres.c`** مش بيضيف logic جديد للـ clock framework — هو بس بيلف الـ existing clock API في الـ **devres pattern** عشان الـ driver writers يكتبوا كود أنظف وأأمن. فكّر فيه كـ "safety belt" — مش بيغير الـ hardware behavior، بس بيضمن إن الـ cleanup هيحصل صح دايمًا.
## Phase 2: شرح الـ Clock (CLK) + Devres Framework

### المشكلة اللي بيحلها الـ Subsystem ده

كل device في النظام المضمن محتاج **clock** عشان يشتغل — الـ UART محتاج baud clock، الـ SPI محتاج serial clock، الـ USB محتاج 48 MHz clock. من غير clock management مركزي، كل driver كان هيتكلم مع الـ hardware registers مباشرةً بنفسه — وده يعني:

- **resource leak**: الـ driver لو فشل في الـ probe أو الـ remove، الـ clock يفضل enabled.
- **كود متكرر**: كل driver يكتب نفس منطق الـ get/enable/disable/put.
- **عدم تنسيق**: لو اتنين شاركوا نفس الـ clock source وكل واحد shut it down بشكل مستقل — crash.
- **عدم تجريد**: كل SOC له registers مختلفة — الـ driver مش المفروض يعرف حاجة عن الـ hardware layout.

الـ **Common Clock Framework (CCF)** + **devres (Device Resource Management)** اتولدوا عشان يحلوا الاتنين مع بعض.

---

### الحل اللي بيعمله الـ Kernel

الـ kernel قسّم المشكلة لجزئين:

1. **الـ CLK Framework** (`CONFIG_COMMON_CLK`): بيوفر abstraction layer موحد للـ clocks — tree من الـ clock nodes، كل node عنده parent، rate، enable count. الـ driver بيطلب clock باسم منطقي (`"uart_clk"`)، والـ framework بيجيبه من الـ device tree أو الـ board file.

2. **الـ Devres Framework**: بيربط lifetime أي resource بـ `struct device`. لما الـ device يتـ unbind أو يفشل probe، الـ kernel بيـ cleanup كل الـ resources اللي اتسجلت معاه تلقائياً — بما فيها الـ clocks.

الـ `clk-devres.c` هو **الجسر بين الاتنين** — بيوفر `devm_clk_*` APIs اللي تجمع الاتنين في function واحدة.

---

### Real-World Analogy — مكتبة بها أمين مكتبة

تخيل مكتبة جامعية:

| المكتبة | الـ Kernel |
|---|---|
| **الكتب** | الـ clock sources (PLL، oscillator، divider) |
| **أمين المكتبة** | الـ CLK Framework |
| **بطاقة الاستعارة (library card)** | `struct clk *` |
| **سجل الاستعارات** | الـ enable reference count |
| **نظام الإرجاع التلقائي** | الـ devres cleanup |
| **الطالب** | الـ device driver |
| **انتهاء الفصل الدراسي** | driver unbind / device removal |

الطالب (driver) بيطلب كتاب (clock) من أمين المكتبة (CLK Framework) بالاسم — مش بالرف ورقم الصفحة. أمين المكتبة هو اللي يعرف الكتاب فين. لو الطالب نسي يرجع الكتاب لما الفصل خلص (driver unbind)، **نظام الإرجاع التلقائي (devres)** بياخده ويرده.

لكن الـ devres مش بس بيرجع الكتاب — هو بيعرف الخطوات بالضبط:
- لو الكتاب كان **enabled**: بيعمل `disable` + `unprepare` الأول.
- بعدين بيعمل `put` (يرد الـ reference).

ده المقابل الدقيق لـ `clk_disable_unprepare` → `clk_put`.

---

### Big Picture Architecture

```
+----------------------------------------------------------+
|                    Device Driver                         |
|   devm_clk_get_enabled(dev, "uart_clk")                  |
+---------------------------+------------------------------+
                            |
                            v
+----------------------------------------------------------+
|              clk-devres.c  (الجسر)                       |
|                                                          |
|  __devm_clk_get()                                        |
|    1. devres_alloc(devm_clk_release, ...)  ← allocate   |
|    2. get(dev, id)        ← clk_get()                   |
|    3. init(clk)           ← clk_prepare_enable()        |
|    4. devres_add(dev, state)              ← register     |
+----------+-------------------+-----------+--------------+
           |                   |           |
           v                   v           v
+------------------+  +--------------+  +----------------+
| CLK Framework    |  | devres core  |  | OF (DT) layer  |
| (clk_get,        |  | (kernel/     |  | of_clk_get_    |
|  clk_prepare,    |  |  devres.c)   |  | by_name()      |
|  clk_enable,     |  |              |  |                |
|  clk_put)        |  | devres_alloc |  | device tree    |
+--------+---------+  | devres_add   |  | clocks = <&>   |
         |            | devres_release+------------------+
         v            +--------------+
+----------------------------------------------------------+
|            Clock Tree (struct clk_core)                  |
|                                                          |
|   [xtal_24M]──►[PLL0]──►[AHB_DIV]──►[uart_clk]         |
|                    └────►[APB_DIV]──►[spi_clk]          |
|                                └────►[i2c_clk]          |
+----------------------------------------------------------+
         |
         v
+----------------------------------------------------------+
|      Hardware Clock Registers (SOC specific)             |
|   struct clk_hw_ops { .enable, .disable, .set_rate }    |
+----------------------------------------------------------+
```

---

### الـ Core Abstraction — الفكرة المحورية

الفكرة المحورية هي **تسجيل cleanup callback مع كل resource عند الحصول عليه**.

الـ `struct devm_clk_state` هو القلب:

```c
struct devm_clk_state {
    struct clk *clk;           /* الـ clock handle */
    void (*exit)(struct clk *clk); /* cleanup function: unprepare أو disable_unprepare */
};
```

الـ `exit` pointer هو اللي بيحدد **ما هو العكس الصحيح** للعملية اللي اتعملت:

```
get only          → exit = NULL          (بس clk_put)
get + prepare     → exit = clk_unprepare
get + enable      → exit = clk_disable_unprepare
```

الـ `devm_clk_release` بيُستدعى تلقائياً من الـ devres core عند device removal:

```c
static void devm_clk_release(struct device *dev, void *res)
{
    struct devm_clk_state *state = res;

    if (state->exit)
        state->exit(state->clk);  /* disable/unprepare لو لزم */

    clk_put(state->clk);          /* دايماً بيتعمل */
}
```

---

### تدفق الـ `__devm_clk_get` بالتفصيل

```c
static struct clk *__devm_clk_get(struct device *dev, const char *id,
                  struct clk *(*get)(struct device *, const char *),
                  int (*init)(struct clk *),
                  void (*exit)(struct clk *))
```

الـ function بتاخد 3 function pointers:
- `get`: إزاي تجيب الـ clock (`clk_get` أو `clk_get_optional`)
- `init`: إيه اللي تعمله عليه (`NULL`، `clk_prepare`، `clk_prepare_enable`)
- `exit`: عكس الـ init (`NULL`، `clk_unprepare`، `clk_disable_unprepare`)

```
devres_alloc()
     │
     ▼
  [state allocated in kernel heap, size = sizeof(devm_clk_state)]
     │
     ▼
get(dev, id)  ──────► clk_get()
                          │
                          ▼
                    [DT lookup: dev→of_node → "clocks" property → phandle]
                          │
                          ▼
                    [CLK framework returns struct clk*]
     │
     ▼ (if init != NULL)
init(clk)  ──────► clk_prepare_enable()
                          │
                          ▼
                    [2-phase: prepare (may sleep) → enable (atomic ok)]
     │
     ▼
state->clk = clk
state->exit = exit
devres_add(dev, state)
     │
     ▼
  [state linked into dev->devres_head list]
```

لو أي خطوة فشلت، الـ error path واضح:

```c
err_clk_init:
    clk_put(clk);      /* رد الـ reference */
err_clk_get:
    devres_free(state); /* حرر الـ state */
    return ERR_PTR(ret);
```

---

### الـ Bulk API — لما الـ Driver يحتاج أكتر من Clock

الـ `struct clk_bulk_data` هي array بسيطة من `{id, clk}` pairs:

```c
struct clk_bulk_data {
    const char  *id;   /* اسم الـ clock في الـ DT */
    struct clk  *clk;  /* بيتملاش من الـ framework */
};
```

الـ driver بيعرفها static:

```c
static struct clk_bulk_data my_clks[] = {
    { .id = "core" },
    { .id = "bus"  },
    { .id = "ref"  },
};
```

بعدين بياخدهم كلهم في call واحدة:

```c
ret = devm_clk_bulk_get_all_enabled(dev, &clks);
/* لو النتيجة > 0: عدد الـ clocks اللي اتجابت وفُعّلت */
```

الـ `struct clk_bulk_devres` بيحتفظ بالـ pointer للـ array والعدد:

```c
struct clk_bulk_devres {
    struct clk_bulk_data *clks;
    int num_clks;
};
```

والـ cleanup callback بيـ iterate عليهم كلهم:

```c
static void devm_clk_bulk_release_all_enable(struct device *dev, void *res)
{
    struct clk_bulk_devres *devres = res;

    clk_bulk_disable_unprepare(devres->num_clks, devres->clks);
    clk_bulk_put_all(devres->num_clks, devres->clks);
}
```

---

### العلاقة بين الـ Structs

```
struct device
    │
    └── devres_head (linked list)
            │
            ├── [devres node] ─► devm_clk_state
            │                        ├── clk ──────► struct clk
            │                        │                   │
            │                        └── exit()          └── struct clk_core
            │                                                    ├── name
            │                                                    ├── rate
            │                                                    ├── enable_count
            │                                                    ├── parent
            │                                                    └── ops (clk_ops)
            │
            └── [devres node] ─► clk_bulk_devres
                                     ├── num_clks = 3
                                     └── clks[] ──► [{id="core", clk=*},
                                                      {id="bus",  clk=*},
                                                      {id="ref",  clk=*}]
```

---

### الـ `devm_get_clk_from_child` — حالة خاصة

معظم الـ APIs بتجيب الـ clock من الـ `dev` نفسه. لكن أحياناً في SoC معقد، الـ clock محدد في **child DT node** مش في الـ device node نفسه.

```c
struct clk *devm_get_clk_from_child(struct device *dev,
                    struct device_node *np, const char *con_id)
```

هنا `np` هو الـ child node، و`dev` هو owner الـ resource (اللي هيتحذف عنده الـ clock).

مثال DT:

```dts
soc {
    my_device@1000 {
        sub_block {
            clocks = <&pll0 1>;
            clock-names = "ref";
        };
    };
};
```

الـ driver بياخد `np` للـ `sub_block` و`dev` للـ `my_device`:

```c
clk = devm_get_clk_from_child(dev, sub_node, "ref");
```

---

### ما الذي يملكه الـ Subsystem وما الذي يُفوِّضه

| المسؤولية | المالك |
|---|---|
| تسجيل cleanup مع الـ device | **clk-devres.c** |
| تخصيص `devm_clk_state` | **clk-devres.c** |
| ربط الـ `(dev, id)` بالـ clock hardware | **CLK Framework** (`clk_get`) |
| DT parsing لحل الـ phandle | **OF CLK layer** (`of_clk_get_by_name`) |
| إدارة الـ enable reference count | **CLK Framework** core |
| الـ hardware register write للـ enable/disable | **SOC-specific clk driver** (مثلاً `drivers/clk/clk-imx8m.c`) |
| تحرير الـ `devres` node من memory | **devres core** (`kernel/devres.c`) |

---

### Subsystems المرتبطة — مهمة تفهمها الأول

- **الـ devres (Device Resource Management)**: نظام عام في الـ kernel بيربط أي resource بـ `struct device`. أي `devm_*` function في الـ kernel بتعتمد عليه. يعرفه `kernel/devres.c`.
- **الـ Common Clock Framework (CCF)**: الـ framework اللي بيدير الـ clock tree كله — الـ `clk-devres.c` بيستخدم APIs منه زي `clk_get`، `clk_prepare`، `clk_enable`، `clk_put`.
- **الـ OF (Open Firmware / Device Tree)**: بيربط الاسم المنطقي للـ clock بالـ phandle في الـ DT. الـ `of_clk_get_by_name()` هي المدخل.

---

### مثال driver حقيقي — UART على i.MX8

```c
/* في probe */
static int imx_uart_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;

    /* get + prepare + enable في سطر واحد، cleanup تلقائي */
    sport->clk_per = devm_clk_get_enabled(dev, "per");
    if (IS_ERR(sport->clk_per))
        return PTR_ERR(sport->clk_per);

    sport->clk_ipg = devm_clk_get_enabled(dev, "ipg");
    if (IS_ERR(sport->clk_ipg))
        return PTR_ERR(sport->clk_ipg);

    /* من هنا، لو الـ probe فشل أو الـ device اتـ remove،
       الـ devres core هيعمل:
         clk_disable_unprepare(clk_ipg)
         clk_put(clk_ipg)
         clk_disable_unprepare(clk_per)
         clk_put(clk_per)
       بالترتيب العكسي تلقائياً */

    return 0;
}
/* مفيش imx_uart_remove محتاج يعمل clk_disable أو clk_put */
```

الـ DT المقابل:

```dts
uart1: serial@30860000 {
    clocks = <&clk IMX8MQ_CLK_UART1_ROOT>,
             <&clk IMX8MQ_CLK_UART1_ROOT>;
    clock-names = "ipg", "per";
};
```

---

### الـ Optional Clock — فلسفة المرونة

الـ `clk_get_optional` بترجع `NULL` بدل `-ENOENT` لو الـ clock مش موجود. ده مهم لـ drivers بتشتغل على أكتر من hardware variant — بعض الـ variants عندها clock معين وبعضها لا.

```c
/* لو مفيش "ref" clock في الـ DT، clk بيبقى NULL مش error */
clk = devm_clk_get_optional_enabled(dev, "ref");
if (IS_ERR(clk))
    return PTR_ERR(clk); /* error حقيقي */
/* clk == NULL يعني مفيش clock وده مقبول */
```

الـ CLK Framework نفسه بيتعامل مع `NULL clk` بشكل آمن في كل الـ `clk_*` operations — بيعمل no-op.

---

### الـ `devm_clk_get_optional_enabled_with_rate` — الأكثر تركيباً

```c
struct clk *devm_clk_get_optional_enabled_with_rate(struct device *dev,
                            const char *id,
                            unsigned long rate)
{
    struct clk *clk;
    int ret;

    /* الـ exit هنا clk_disable_unprepare بس — لأن الـ set_rate
       والـ prepare_enable بيتعملوا manually بعدين */
    clk = __devm_clk_get(dev, id, clk_get_optional, NULL,
                 clk_disable_unprepare);
    if (IS_ERR(clk))
        return ERR_CAST(clk);

    ret = clk_set_rate(clk, rate);   /* set rate الأول قبل enable */
    if (ret)
        goto out_put_clk;

    ret = clk_prepare_enable(clk);   /* بعدين enable */
    if (ret)
        goto out_put_clk;

    return clk;

out_put_clk:
    devm_clk_put(dev, clk);  /* يحذف الـ devres node يدوياً */
    return ERR_PTR(ret);
}
```

لازم `set_rate` يتعمل **قبل** `enable` لأن بعض الـ hardware بيرفض تغيير الـ rate وهو running.

---

### الـ `devm_clk_put` — الحذف اليدوي قبل الـ unbind

في الحالة الاستثنائية لما الـ driver يحتاج يحرر الـ clock مبكراً (مش بس عند الـ unbind):

```c
void devm_clk_put(struct device *dev, struct clk *clk)
{
    int ret;

    /* بيدور على الـ devres node اللي فيه الـ clk ده،
       بيشغّل الـ devm_clk_release callback، وبيحذف الـ node */
    ret = devres_release(dev, devm_clk_release, devm_clk_match, clk);

    WARN_ON(ret); /* لو مالقاش الـ clock = bug */
}
```

الـ `devm_clk_match` هو function صغيرة بتتحقق إن الـ `struct clk*` المخزون في الـ devres node هو نفس الـ pointer اللي طلبنا حذفه:

```c
static int devm_clk_match(struct device *dev, void *res, void *data)
{
    struct clk **c = res;
    /* res هو أول field في devm_clk_state وهو struct clk* */
    return *c == data;
}
```

**ملاحظة مهمة**: الـ `res` بيأشر على `devm_clk_state` مش على `struct clk*` مباشرة — لذلك الـ cast `struct clk **c = res` بيعمل لأن أول field في `devm_clk_state` هو `struct clk *clk`، وفي C الـ pointer لأول field = الـ pointer للـ struct نفسه.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

### جدول الـ Flags والـ Macros المهمة

| الـ Macro / الـ typedef | الغرض |
|---|---|
| `EXPORT_SYMBOL(sym)` | يجعل الـ symbol متاح لأي kernel module |
| `EXPORT_SYMBOL_GPL(sym)` | متاح فقط لـ GPL-licensed modules |
| `__must_check` | المترجم يحذر لو الـ return value اتجاهل |
| `IS_ERR(ptr)` | يتحقق لو الـ pointer فيه error code |
| `ERR_PTR(err)` | يحول error code لـ pointer |
| `PTR_ERR(ptr)` | يرجع الـ error code من داخل الـ pointer |
| `ERR_CAST(ptr)` | يكست الـ error pointer لنوع تاني |
| `GFP_KERNEL` | allocation flag — قابل للـ sleep، الـ use الاعتيادي في kernel context |
| `dr_release_t` | typedef لـ `void (*)(struct device *, void *)` — callback الـ cleanup |
| `dr_match_t` | typedef لـ `int (*)(struct device *, void *, void *)` — callback الـ matching |

---

### جدول الـ Config Options المؤثرة

| الـ Config | التأثير على الملف |
|---|---|
| `CONFIG_HAVE_CLK_PREPARE` | لو مش موجود، `clk_prepare` / `clk_unprepare` بيبقوا no-ops |
| `CONFIG_COMMON_CLK` | لو مش موجود، كل الـ clk API بيبقى stubs فارغة |

---

### الـ Structs المهمة

#### 1. `struct devm_clk_state`

```c
struct devm_clk_state {
    struct clk *clk;           /* pointer للـ clock المُدار */
    void (*exit)(struct clk *clk); /* callback بيتنفذ وقت الـ cleanup */
};
```

**الغرض:** حالة devres لـ clock واحد. بيتحجز مع `devres_alloc` وبيتسجل على الـ device. لما الـ device يتحرر، الـ kernel بيستدعي `devm_clk_release` اللي بيستخدم الـ `exit` callback الجواه.

| الـ Field | النوع | الشرح |
|---|---|---|
| `clk` | `struct clk *` | الـ clock handle اللي جه من `clk_get` |
| `exit` | function pointer | `clk_unprepare`، `clk_disable_unprepare`، أو `NULL` حسب الـ variant |

**الاتصال بالـ structs التانية:** بيحتوي على `struct clk *` اللي هو opaque handle من CLK framework.

---

#### 2. `struct clk_bulk_devres`

```c
struct clk_bulk_devres {
    struct clk_bulk_data *clks;  /* array الـ clock entries */
    int num_clks;                /* عدد الـ clocks في الـ array */
};
```

**الغرض:** حالة devres لمجموعة clocks. بيشاور على array من `clk_bulk_data` اللي بيحتفظ بها الـ driver في stack/heap خاص بيه.

| الـ Field | النوع | الشرح |
|---|---|---|
| `clks` | `struct clk_bulk_data *` | مؤشر على الـ array بتاع الـ caller |
| `num_clks` | `int` | عدد العناصر في الـ array |

---

#### 3. `struct clk_bulk_data` (معرفة في `<linux/clk.h>`)

```c
struct clk_bulk_data {
    const char  *id;   /* اسم الـ clock consumer (مثلاً "ahb", "apb") */
    struct clk  *clk;  /* يتملي تلقائياً من clk_bulk_get */
};
```

**الغرض:** وحدة بيانات واحدة في الـ bulk API. الـ driver بيعمل array منها ويحط فيها الـ IDs، والـ API بيملي الـ `clk` pointers.

---

### مخطط العلاقات بين الـ Structs (ASCII)

```
struct device
  │
  └──[devres list]──────────────────────────────────┐
                                                    │
         ┌──────────────────┐      ┌────────────────┴──────────┐
         │ devm_clk_state   │      │   clk_bulk_devres         │
         │──────────────────│      │───────────────────────────│
         │ clk  ──────────► struct clk (opaque, CLK core)      │
         │ exit (fn ptr)    │      │ clks ──────────────────┐  │
         └──────────────────┘      │ num_clks               │  │
                                   └────────────────────────┼──┘
                                                            │
                                                            ▼
                                          struct clk_bulk_data[]
                                          ┌──────────────────────┐
                                          │ id:  "ahb"           │
                                          │ clk: ──► struct clk  │
                                          ├──────────────────────┤
                                          │ id:  "apb"           │
                                          │ clk: ──► struct clk  │
                                          └──────────────────────┘
```

**ملاحظة:** الـ `struct clk` نفسه هو **opaque type** — الـ CLK core بس يعرف شكله الداخلي. الـ devres structs بيشاوروا عليه بـ pointer بس.

---

### دورة الحياة — Single Clock (devm_clk_get)

```
driver calls devm_clk_get(dev, "myclk")
  │
  ▼
__devm_clk_get(dev, id, clk_get, NULL, NULL)
  │
  ├─► devres_alloc(devm_clk_release, sizeof(devm_clk_state), GFP_KERNEL)
  │     └─ [heap: devm_clk_state اتحجزت، لسه متسجلتش]
  │
  ├─► clk_get(dev, id)  ←── يجيب handle من CLK framework
  │     └─ لو فشل: devres_free(state) ← return ERR_PTR
  │
  ├─► init(clk)  [لو موجود: clk_prepare أو clk_prepare_enable]
  │     └─ لو فشل: clk_put(clk) → devres_free(state) ← return ERR_PTR
  │
  ├─► state->clk = clk
  ├─► state->exit = exit  [NULL / clk_unprepare / clk_disable_unprepare]
  │
  └─► devres_add(dev, state)
        └─ [state اتربطت بـ dev، هتتحرر automatically]
              │
              │    [normal usage by driver]
              │
              ▼
        device unbind / driver remove
              │
              └─► devm_clk_release(dev, state)
                    │
                    ├─► state->exit(state->clk)  [لو مش NULL]
                    │     مثلاً: clk_disable_unprepare(clk)
                    │
                    └─► clk_put(state->clk)
                          └─ [CLK framework بيحرر الـ handle]
```

---

### دورة الحياة — Bulk Clocks (devm_clk_bulk_get)

```
driver prepares:
  struct clk_bulk_data clks[] = {
      { .id = "ahb" },
      { .id = "apb" },
  };

devm_clk_bulk_get(dev, 2, clks)
  │
  ▼
__devm_clk_bulk_get(dev, 2, clks, optional=false)
  │
  ├─► devres_alloc(devm_clk_bulk_release, sizeof(clk_bulk_devres))
  │
  ├─► clk_bulk_get(dev, 2, clks)
  │     └─ يملي clks[0].clk و clks[1].clk
  │     └─ لو فشل: devres_free → return error
  │
  ├─► devres->clks    = clks
  ├─► devres->num_clks = 2
  │
  └─► devres_add(dev, devres)
              │
              ▼
        device unbind
              │
              └─► devm_clk_bulk_release(dev, devres)
                    │
                    └─► clk_bulk_put(2, clks)
```

---

### دورة الحياة — Bulk + Enable

```
devm_clk_bulk_get_all_enabled(dev, &clks)
  │
  ├─► devres_alloc(devm_clk_bulk_release_all_enable, ...)
  │
  ├─► clk_bulk_get_all(dev, &devres->clks)
  │     └─ بيعمل discover لكل الـ clocks في DT ويحجزها
  │     └─ لو ret <= 0: devres_free → return
  │
  ├─► *clks = devres->clks
  ├─► devres->num_clks = ret
  │
  ├─► clk_bulk_prepare_enable(num_clks, clks)
  │     └─ لو فشل: clk_bulk_put_all → devres_free → return error
  │
  └─► devres_add(dev, devres)
              │
              ▼
        device unbind
              │
              └─► devm_clk_bulk_release_all_enable(dev, devres)
                    │
                    ├─► clk_bulk_disable_unprepare(num, clks)
                    └─► clk_bulk_put_all(num, clks)
```

---

### مخطط الـ Call Flow — devm_clk_get_optional_enabled_with_rate

```
devm_clk_get_optional_enabled_with_rate(dev, id, rate)
  │
  ├─► __devm_clk_get(dev, id, clk_get_optional, NULL, clk_disable_unprepare)
  │     │
  │     │   [init=NULL → مفيش prepare هنا]
  │     │   [exit=clk_disable_unprepare → هيتنفذ وقت الـ release]
  │     │
  │     └─► return clk  (مسجلة في devres بـ exit فقط، مش enabled لسه)
  │
  ├─► clk_set_rate(clk, rate)
  │     └─ لو فشل: devm_clk_put(dev, clk) → return ERR_PTR
  │                  └─► devres_release → devm_clk_release → clk_put
  │
  ├─► clk_prepare_enable(clk)
  │     └─ لو فشل: devm_clk_put(dev, clk) → return ERR_PTR
  │
  └─► return clk  ← الـ driver بيستخدمه، الـ cleanup تلقائي وقت الـ unbind
```

---

### مخطط الـ Call Flow — devm_clk_put (manual release)

```
devm_clk_put(dev, clk)
  │
  └─► devres_release(dev, devm_clk_release, devm_clk_match, clk)
        │
        ├─► devm_clk_match(dev, res, clk)
        │     └─ بيمشي على الـ devres list ويدور على الـ entry اللي *res == clk
        │
        └─► devm_clk_release(dev, state)
              ├─► state->exit(state->clk)  [لو موجود]
              └─► clk_put(state->clk)
```

---

### مخطط الـ Call Flow — devm_get_clk_from_child

```
devm_get_clk_from_child(dev, np, con_id)
  │
  │   [np هو device_node تاني — مش الـ dev->of_node نفسه]
  │
  ├─► devres_alloc(devm_clk_release, sizeof(devm_clk_state))
  │
  ├─► of_clk_get_by_name(np, con_id)
  │     └─ يجيب الـ clock من DT node ابن مختلف
  │
  ├─► لو نجح:
  │     state->clk = clk
  │     state->exit = NULL   ← [مفيش exit callback!]
  │     devres_add(dev, state)
  │
  └─► لو فشل: devres_free(state)

الـ cleanup وقت unbind:
  devm_clk_release → clk_put(clk)  [بس، من غير exit]
```

---

### استراتيجية الـ Locking

الملف ده **لا يحتوي على أي lock صريح**. الـ locking بيحصل في المستويات التانية:

| الطبقة | الـ Lock | المسؤول |
|---|---|---|
| **devres layer** | `dev->mutex` (داخلي) | `devres_add` / `devres_release` بيمسكوه تلقائياً |
| **CLK framework** | `prepare_lock` (mutex) | `clk_prepare` / `clk_unprepare` بيمسكوه |
| **CLK framework** | `enable_lock` (spinlock) | `clk_enable` / `clk_disable` بيمسكوه |
| **CLK framework** | `clk_list_lock` | `clk_get` / `clk_put` لحماية الـ clock list |

**قاعدة الترتيب في الـ CLK framework:**
```
prepare_lock (mutex, sleepable)
  └─► enable_lock (spinlock, atomic)
```

لازم `clk_prepare` يتعمل **قبل** `clk_enable` دايماً — ده مش بس API contract، ده عشان الـ locks مختلفة ومش يجوز يتقلبوا.

**نتيجة مهمة للملف ده:** لأن `devres_alloc` بيستخدم `GFP_KERNEL` والـ `devres_add` بياخد `dev->mutex`، كل الـ `devm_clk_get*` functions **لازم تُستدعى من sleepable context فقط** — مينفعش من interrupt handlers أو atomic sections.

---

### جدول الـ API Variants

| الـ Function | الـ get backend | init callback | exit callback | optional? |
|---|---|---|---|---|
| `devm_clk_get` | `clk_get` | — | — | لا |
| `devm_clk_get_prepared` | `clk_get` | `clk_prepare` | `clk_unprepare` | لا |
| `devm_clk_get_enabled` | `clk_get` | `clk_prepare_enable` | `clk_disable_unprepare` | لا |
| `devm_clk_get_optional` | `clk_get_optional` | — | — | نعم |
| `devm_clk_get_optional_prepared` | `clk_get_optional` | `clk_prepare` | `clk_unprepare` | نعم |
| `devm_clk_get_optional_enabled` | `clk_get_optional` | `clk_prepare_enable` | `clk_disable_unprepare` | نعم |
| `devm_clk_get_optional_enabled_with_rate` | `clk_get_optional` | manual `clk_set_rate` + `clk_prepare_enable` | `clk_disable_unprepare` | نعم |

| الـ Bulk Function | الـ get backend | enable? | all? |
|---|---|---|---|
| `devm_clk_bulk_get` | `clk_bulk_get` | لا | لا |
| `devm_clk_bulk_get_optional` | `clk_bulk_get_optional` | لا | لا |
| `devm_clk_bulk_get_optional_enable` | `clk_bulk_get_optional` | نعم | لا |
| `devm_clk_bulk_get_all` | `clk_bulk_get_all` | لا | نعم |
| `devm_clk_bulk_get_all_enabled` | `clk_bulk_get_all` | نعم | نعم |
## Phase 4: شرح الـ Functions

---

### ملخص سريع — Cheatsheet

#### الـ Single Clock API

| Function | النوع | الـ init callback | الـ exit callback | Optional? |
|---|---|---|---|---|
| `devm_clk_get` | public | — | — | No |
| `devm_clk_get_prepared` | public | `clk_prepare` | `clk_unprepare` | No |
| `devm_clk_get_enabled` | public | `clk_prepare_enable` | `clk_disable_unprepare` | No |
| `devm_clk_get_optional` | public | — | — | Yes |
| `devm_clk_get_optional_prepared` | public | `clk_prepare` | `clk_unprepare` | Yes |
| `devm_clk_get_optional_enabled` | public | `clk_prepare_enable` | `clk_disable_unprepare` | Yes |
| `devm_clk_get_optional_enabled_with_rate` | public | manual | `clk_disable_unprepare` | Yes |
| `devm_get_clk_from_child` | public | — | — | No (OF node) |
| `devm_clk_put` | public | early release | — | — |
| `__devm_clk_get` | internal | configurable | configurable | configurable |

#### الـ Bulk Clock API

| Function | Optional? | Auto-Enable? | Auto-discover? |
|---|---|---|---|
| `devm_clk_bulk_get` | No | No | No |
| `devm_clk_bulk_get_optional` | Yes | No | No |
| `devm_clk_bulk_get_optional_enable` | Yes | Yes | No |
| `devm_clk_bulk_get_all` | — | No | Yes |
| `devm_clk_bulk_get_all_enabled` | — | Yes | Yes |

#### الـ Internal Helpers & Release Callbacks

| Function | الدور |
|---|---|
| `devm_clk_release` | devres cleanup لـ single clock |
| `devm_clk_bulk_release` | devres cleanup لـ bulk (get only) |
| `devm_clk_bulk_release_enable` | devres cleanup لـ bulk (get + enable) |
| `devm_clk_bulk_release_all` | devres cleanup لـ bulk_get_all |
| `devm_clk_bulk_release_all_enable` | devres cleanup لـ bulk_get_all + enable |
| `devm_clk_match` | devres match callback لـ devm_clk_put |
| `__devm_clk_bulk_get` | internal core لـ bulk get |
| `__devm_clk_bulk_get_enable` | internal core لـ bulk get+enable |

---

### المجموعة 1: الـ Internal Core — `__devm_clk_get` و `__devm_clk_bulk_get` و `__devm_clk_bulk_get_enable`

الثلاث دوال دي هي الـ backbone الحقيقي للملف كله. كل الـ public API هي مجرد wrappers بتمرر callbacks ليهم. الفكرة الأساسية: allocate devres → get clock → optional init → register devres. لو أي خطوة فشلت، الـ cleanup بيحصل manually قبل ما الـ devres framework يتدخل.

---

#### `__devm_clk_get`

```c
static struct clk *__devm_clk_get(struct device *dev, const char *id,
                                  struct clk *(*get)(struct device *dev, const char *id),
                                  int (*init)(struct clk *clk),
                                  void (*exit)(struct clk *clk))
```

**ما بتعمله:**
بتعمل allocate لـ `devm_clk_state` باستخدام `devres_alloc`، بعدين بتستدعي الـ `get` callback (ممكن يكون `clk_get` أو `clk_get_optional`) عشان تجيب الـ clock. لو في `init` callback (زي `clk_prepare` أو `clk_prepare_enable`)، بتشغله على الـ clock اللي اتجلب. في النهاية بتحفظ الـ `clk` والـ `exit` callback جوه الـ state وبتعمل `devres_add` عشان يتربط بالـ device lifecycle.

**الـ Parameters:**
- `dev` — الـ device اللي الـ clock هيتربط بـ lifetime بتاعه
- `id` — الـ consumer ID اللي بيتبحث عنه في DT أو platform data
- `get` — function pointer لـ `clk_get` أو `clk_get_optional`
- `init` — optional callback بيشتغل بعد الـ get (prepare/enable)، `NULL` لو مش محتاج
- `exit` — opposite الـ `init`، بيتنفذ وقت الـ cleanup، `NULL` لو مش محتاج

**الـ Return:**
- `struct clk *` valid pointer لو نجح
- `ERR_PTR(-ENOMEM)` لو الـ devres allocation فشل
- `ERR_PTR(ret)` لو الـ `get` أو `init` فشلوا

**Key Details:**
- **Locking:** مفيش locking صريح هنا — الـ devres subsystem بيتولى الحماية، والـ clk framework عنده internal locking خاص بيه.
- **Error path:** لو `init` فشل، بيعمل `clk_put` على الـ clock اللي اتجلب قبل `devres_free`، عشان مافيش leak.
- **Context:** لازم يتنادى من sleepable context (الـ `clk_get` بنفسها ممكن تنام).

**Pseudocode flow:**
```
devres_alloc(devm_clk_release, sizeof(state))
    --> ENOMEM? return ERR_PTR

get(dev, id)
    --> IS_ERR? devres_free; return ERR_PTR

init(clk) [if not NULL]
    --> fail? clk_put; devres_free; return ERR_PTR

state->clk = clk
state->exit = exit
devres_add(dev, state)
return clk
```

**من بيناديها:** كل الـ public single-clock devm functions في الملف.

---

#### `__devm_clk_bulk_get`

```c
static int __devm_clk_bulk_get(struct device *dev, int num_clks,
                               struct clk_bulk_data *clks, bool optional)
```

**ما بتعمله:**
بتعمل allocate لـ `clk_bulk_devres`، بعدين بتستدعي `clk_bulk_get` أو `clk_bulk_get_optional` حسب الـ `optional` flag. لو نجح، بتحفظ الـ pointer وعدد الـ clocks وبتعمل `devres_add`. لو فشل، بتعمل `devres_free` فقط لأن الـ `clk_bulk_get` نفسها بتعمل cleanup للـ clocks اللي اتجلبت جزئياً.

**الـ Parameters:**
- `dev` — الـ consumer device
- `num_clks` — عدد الـ clocks في الـ `clks` array
- `clks` — array من `clk_bulk_data`، كل عنصر فيه `id` و `clk` field
- `optional` — لو `true`، الـ missing clocks مش error

**الـ Return:** `0` لو نجح، negative errno لو فشل.

**Key Details:**
- الـ `clks` array نفسها مش بتتcopy — الـ devres بس بيحفظ pointer ليها، فالـ caller مسؤول إن الـ array تفضل valid.
- الـ `devm_clk_bulk_release` هو الـ destructor المسجل.

**من بيناديها:** `devm_clk_bulk_get` و `devm_clk_bulk_get_optional`.

---

#### `__devm_clk_bulk_get_enable`

```c
static int __devm_clk_bulk_get_enable(struct device *dev, int num_clks,
                                      struct clk_bulk_data *clks, bool optional)
```

**ما بتعمله:**
زي `__devm_clk_bulk_get` بس بتضيف خطوة `clk_bulk_prepare_enable` بعد الـ get. الـ destructor المسجل هو `devm_clk_bulk_release_enable` اللي بيعمل `disable_unprepare` قبل الـ `put`.

**الـ Parameters:** نفس `__devm_clk_bulk_get`.

**الـ Return:** `0` لو نجح، negative errno لو أي خطوة فشلت.

**Key Details:**
- **Error path دقيق:** لو `clk_bulk_prepare_enable` فشل، بتعمل `clk_bulk_put` يدوياً على الـ clocks اللي اتجلبت قبل `devres_free`. الـ devres لسه مش registered فمفيش double-free.
- الـ destructor `devm_clk_bulk_release_enable` بيعمل `disable → unprepare → put` بالترتيب الصح.

**Pseudocode flow:**
```
devres_alloc(devm_clk_bulk_release_enable, ...)
    --> ENOMEM? return -ENOMEM

clk_bulk_get[_optional](dev, num_clks, clks)
    --> fail? devres_free; return ret

clk_bulk_prepare_enable(num_clks, clks)
    --> fail? clk_bulk_put(clks); devres_free; return ret

devres->clks = clks; devres->num_clks = num_clks
devres_add(dev, devres)
return 0
```

**من بيناديها:** `devm_clk_bulk_get_optional_enable`.

---

### المجموعة 2: الـ Release (Destructor) Callbacks

الـ devres framework بيستدعي الـ callbacks دي automatically لما الـ device بيتunbind أو بيتdestroy. الـ caller context دايماً هو devres teardown، بيحصل من `device_release_driver` أو `devm_clk_put`.

---

#### `devm_clk_release`

```c
static void devm_clk_release(struct device *dev, void *res)
```

**ما بتعمله:**
بتجيب الـ `devm_clk_state` من الـ `res` pointer، لو في `exit` callback (زي `clk_unprepare` أو `clk_disable_unprepare`) بتناديه أولاً، بعدين بتعمل `clk_put` على الـ clock.

**Key Details:**
- الترتيب مهم: exit (disable/unprepare) قبل put.
- `exit` ممكن تكون `NULL` لو الـ clock اتجلب بدون prepare/enable.

---

#### `devm_clk_bulk_release`

```c
static void devm_clk_bulk_release(struct device *dev, void *res)
```

**ما بتعمله:**
بتعمل `clk_bulk_put` على كل الـ clocks في الـ array. بس put — مفيش disable/unprepare. بتتنادى لو الـ clocks اتجلبت بدون enable (بس get).

---

#### `devm_clk_bulk_release_enable`

```c
static void devm_clk_bulk_release_enable(struct device *dev, void *res)
```

**ما بتعمله:**
بتعمل `clk_bulk_disable_unprepare` أولاً، بعدين `clk_bulk_put`. بتتنادى لو الـ clocks اتجلبت وعملت enable عبر `__devm_clk_bulk_get_enable`.

---

#### `devm_clk_bulk_release_all`

```c
static void devm_clk_bulk_release_all(struct device *dev, void *res)
```

**ما بتعمله:**
بتعمل `clk_bulk_put_all` — الفرق عن `clk_bulk_put` إن الـ `_all` variant بتعمل free للـ array نفسها اللي اتalloc جوه `clk_bulk_get_all`. بتتنادى لـ `devm_clk_bulk_get_all`.

---

#### `devm_clk_bulk_release_all_enable`

```c
static void devm_clk_bulk_release_all_enable(struct device *dev, void *res)
```

**ما بتعمله:**
`disable_unprepare` + `put_all`. بتتنادى لـ `devm_clk_bulk_get_all_enabled`.

---

### المجموعة 3: الـ Public Single Clock API

كل الدوال دي wrappers نظيفة فوق `__devm_clk_get`. الاختلاف الوحيد بينهم هو الـ `get`/`init`/`exit` callbacks اللي بيمروها.

---

#### `devm_clk_get`

```c
struct clk *devm_clk_get(struct device *dev, const char *id)
```

**ما بتعمله:**
بتجيب clock بـ `clk_get` من غير prepare أو enable. الـ clock بتترلخ automatically لما الـ device يتdestroy.

**الـ Parameters:**
- `dev` — الـ consumer device
- `id` — consumer ID (من DT `clock-names` أو `NULL` للوحيدة)

**الـ Return:** `struct clk *` أو `ERR_PTR`.

**Key Details:**
- `EXPORT_SYMBOL` (مش GPL) — متاحة للـ non-GPL modules.
- الـ clock مش prepared ولا enabled بعد الاستدعاء.

---

#### `devm_clk_get_prepared`

```c
struct clk *devm_clk_get_prepared(struct device *dev, const char *id)
```

**ما بتعمله:**
`devm_clk_get` + `clk_prepare`. بتجيب الـ clock وبتعمله prepare مباشرة. وقت الـ cleanup بتعمل `clk_unprepare` ثم `clk_put`. الـ driver لسه محتاج يعمل `clk_enable` بنفسه لو عايز يشغلها.

**الـ Return:** `struct clk *` prepared، أو `ERR_PTR`.

**Key Details:** `EXPORT_SYMBOL_GPL`.

---

#### `devm_clk_get_enabled`

```c
struct clk *devm_clk_get_enabled(struct device *dev, const char *id)
```

**ما بتعمله:**
`devm_clk_get` + `clk_prepare_enable`. بتجيب الـ clock وبتعمله prepare وenable في خطوة واحدة. وقت الـ cleanup بتعمل `clk_disable_unprepare` ثم `clk_put`.

**الـ Return:** `struct clk *` prepared+enabled، أو `ERR_PTR`.

**Key Details:** `EXPORT_SYMBOL_GPL`. الأكثر استخداماً في الـ simple drivers اللي مش محتاجة تتحكم في الـ enable يدوياً.

---

#### `devm_clk_get_optional`

```c
struct clk *devm_clk_get_optional(struct device *dev, const char *id)
```

**ما بتعمله:**
زي `devm_clk_get` بالظبط بس بتستخدم `clk_get_optional` تحت. الفرق: لو الـ clock مش موجودة في الـ DT، بترجع `NULL` بدل `ERR_PTR(-ENOENT)`. مفيدة للـ clocks الاختيارية في الـ hardware.

**الـ Return:** `struct clk *` أو `NULL` (clock غير موجودة) أو `ERR_PTR` (error حقيقي).

**Key Details:** `EXPORT_SYMBOL` (non-GPL).

---

#### `devm_clk_get_optional_prepared`

```c
struct clk *devm_clk_get_optional_prepared(struct device *dev, const char *id)
```

**ما بتعمله:**
`devm_clk_get_optional` + prepare. لو الـ clock مش موجودة بترجع `NULL` بدون error. لو موجودة بتعمل prepare وبتسجل `clk_unprepare` كـ cleanup.

**الـ Return:** `NULL` أو `struct clk *` prepared أو `ERR_PTR`.

**Key Details:** `EXPORT_SYMBOL_GPL`.

---

#### `devm_clk_get_optional_enabled`

```c
struct clk *devm_clk_get_optional_enabled(struct device *dev, const char *id)
```

**ما بتعمله:**
`devm_clk_get_optional` + prepare_enable. الأكثر convenience للـ optional clocks اللي عايزها تشتغل مباشرة لو موجودة.

**الـ Return:** `NULL` أو `struct clk *` enabled أو `ERR_PTR`.

**Key Details:** `EXPORT_SYMBOL_GPL`.

---

#### `devm_clk_get_optional_enabled_with_rate`

```c
struct clk *devm_clk_get_optional_enabled_with_rate(struct device *dev,
                                                     const char *id,
                                                     unsigned long rate)
```

**ما بتعمله:**
أكثر دالة مركبة في الملف. بتجيب الـ optional clock بدون init callback (الـ exit هو `clk_disable_unprepare` بس)، بعدين بتعمل `clk_set_rate` يدوياً، بعدين `clk_prepare_enable` يدوياً. السبب في الـ manual flow: لازم الـ rate يتset قبل الـ enable.

**الـ Parameters:**
- `dev` — الـ consumer device
- `id` — consumer clock ID
- `rate` — الـ rate المطلوب بالـ Hz

**الـ Return:** `NULL` لو مش موجودة، `struct clk *` enabled بالـ rate المطلوب، أو `ERR_PTR`.

**Key Details:**
- لو `clk_set_rate` أو `clk_prepare_enable` فشلوا، بتعمل `devm_clk_put` عشان تعمل cleanup للـ devres اللي اتسجل.
- الـ `__devm_clk_get` اتناداها بـ `NULL` init و `clk_disable_unprepare` exit — الـ enable بيحصل يدوياً بعدين.
- **لماذا مش بتستخدم `__devm_clk_get` callbacks؟** لأن الـ rate لازم يتset قبل الـ enable، والـ callback order في `__devm_clk_get` مش بيسمح بده.

**Pseudocode flow:**
```
__devm_clk_get(dev, id, clk_get_optional, NULL, clk_disable_unprepare)
    --> IS_ERR? return ERR_CAST

clk_set_rate(clk, rate)
    --> fail? devm_clk_put; return ERR_PTR

clk_prepare_enable(clk)
    --> fail? devm_clk_put; return ERR_PTR

return clk
```

**Key Details:** `EXPORT_SYMBOL_GPL`.

---

### المجموعة 4: الـ Public Bulk Clock API

الـ bulk API موجودة عشان الـ drivers اللي عندها multiple clocks — بدل ما يعمل `devm_clk_get` N مرات.

---

#### `devm_clk_bulk_get`

```c
int __must_check devm_clk_bulk_get(struct device *dev, int num_clks,
                                    struct clk_bulk_data *clks)
```

**ما بتعمله:**
بتجيب `num_clks` clock بناءً على الـ `id` fields في الـ `clks` array. كل الـ clocks لازم تكون موجودة وإلا بترجع error.

**الـ Parameters:**
- `dev` — الـ consumer device
- `num_clks` — عدد الـ clocks
- `clks` — array من `clk_bulk_data`، كل عنصر لازم فيه `id` متعبي، الـ `clk` field بتتملي تلقائياً

**الـ Return:** `0` أو negative errno.

**Key Details:** `__must_check` — لازم تتحقق من الـ return value. `EXPORT_SYMBOL_GPL`.

---

#### `devm_clk_bulk_get_optional`

```c
int __must_check devm_clk_bulk_get_optional(struct device *dev, int num_clks,
                                             struct clk_bulk_data *clks)
```

**ما بتعمله:**
زي `devm_clk_bulk_get` بس الـ missing clocks مش error — الـ `clk` field بييجي `NULL` ليها.

**الـ Return:** `0` لو كل حاجة تمام (حتى لو بعض الـ clocks `NULL`).

**Key Details:** `EXPORT_SYMBOL_GPL`.

---

#### `devm_clk_bulk_get_optional_enable`

```c
int __must_check devm_clk_bulk_get_optional_enable(struct device *dev, int num_clks,
                                                    struct clk_bulk_data *clks)
```

**ما بتعمله:**
`devm_clk_bulk_get_optional` + `clk_bulk_prepare_enable`. بتجيب الـ clocks الموجودة وبتشغلها، والـ cleanup بيعمل `disable_unprepare` ثم `put`.

**الـ Return:** `0` لو نجح.

**Key Details:** `EXPORT_SYMBOL_GPL`.

---

#### `devm_clk_bulk_get_all`

```c
int __must_check devm_clk_bulk_get_all(struct device *dev,
                                        struct clk_bulk_data **clks)
```

**ما بتعمله:**
بتجيب كل الـ clocks المتاحة للـ device بدون ما الـ driver يحدد الـ names. بتستخدم `clk_bulk_get_all` اللي بتعمل allocate للـ `clks` array داخلياً وبترجع عدد الـ clocks. الـ `*clks` pointer بيتعبى بالـ allocated array.

**الـ Parameters:**
- `dev` — الـ consumer device
- `clks` — pointer-to-pointer، هيتعبى بالـ allocated array

**الـ Return:** عدد الـ clocks (positive) أو `0` لو مافيش أو negative error.

**Key Details:**
- الـ array نفسها اتعملت allocate من جوه `clk_bulk_get_all`، فالـ cleanup عبر `clk_bulk_put_all` اللي بيعمل `kfree` عليها.
- `EXPORT_SYMBOL_GPL`.

**Pseudocode flow:**
```
devres_alloc(devm_clk_bulk_release_all, ...)
    --> ENOMEM? return -ENOMEM

clk_bulk_get_all(dev, &devres->clks)
    --> ret > 0?
        *clks = devres->clks
        devres->num_clks = ret
        devres_add(dev, devres)
    --> ret <= 0?
        devres_free(devres)

return ret
```

---

#### `devm_clk_bulk_get_all_enabled`

```c
int __must_check devm_clk_bulk_get_all_enabled(struct device *dev,
                                                struct clk_bulk_data **clks)
```

**ما بتعمله:**
`devm_clk_bulk_get_all` + `clk_bulk_prepare_enable`. بتجيب كل الـ clocks وبتشغلها. الـ cleanup بيعمل `disable_unprepare` ثم `put_all` (مع الـ kfree للـ array).

**الـ Return:** عدد الـ clocks enabled (positive) أو `0` أو negative error.

**Key Details:**
- الـ devres بيتضاف بس لو الـ prepare_enable نجح — لو فشل بيعمل `clk_bulk_put_all` و `devres_free` يدوياً.
- الفرق الدقيق عن `devm_clk_bulk_get_all`: الـ return value لما كل حاجة تمام هو `devres->num_clks` مش `0`.
- `EXPORT_SYMBOL_GPL`.

---

### المجموعة 5: الـ Match و Manual Release

---

#### `devm_clk_match`

```c
static int devm_clk_match(struct device *dev, void *res, void *data)
```

**ما بتعمله:**
الـ match callback اللي بتستخدمها `devres_release` عشان تلاقي الـ devres entry الصح. بتقارن الـ `struct clk *` المحفوظ في الـ devres مع الـ `data` pointer (اللي هو الـ `clk` اللي بنعمله `devm_clk_put`).

**الـ Parameters:**
- `dev` — الـ device (مش بيتستخدم مباشرة)
- `res` — الـ devres resource block (هو `struct devm_clk_state *`)
- `data` — الـ `struct clk *` المطلوب تطابقه

**الـ Return:** `1` لو match، `0` لو لا.

**Key Details:**
- بتعمل `WARN_ON` لو الـ res pointer نفسه أو الـ clock pointer جوه NULL — هي defensive check لأن الوضع ده theoretically مينفعش يحصل لو الـ devres اتسجل صح.
- `res` في الحقيقة pointer لـ `struct clk **`، يعني pointer لـ field أول جوه الـ `devm_clk_state`.

---

#### `devm_clk_put`

```c
void devm_clk_put(struct device *dev, struct clk *clk)
```

**ما بتعمله:**
بتعمل early release للـ clock قبل ما الـ device يتdestroy. بتستخدم `devres_release` مع `devm_clk_release` و `devm_clk_match` عشان تلاقي الـ devres entry الصح وتطلقه فوراً.

**الـ Parameters:**
- `dev` — الـ device اللي الـ clock متربطة بيه
- `clk` — الـ clock اللي عايزين نطلقها

**الـ Return:** void، بس بيعمل `WARN_ON` لو `devres_release` فشل (يعني الـ clock مش موجودة في الـ devres list).

**Key Details:**
- `devres_release` بتعمل match → release → free للـ devres block في خطوة واحدة.
- لو الـ `exit` callback موجود (زي `clk_disable_unprepare`)، بيتنفذ هنا.
- `EXPORT_SYMBOL` (non-GPL).
- **use case:** الـ driver عايز يرجع الـ clock في الـ `probe` error path بشكل explicit، أو في الـ `remove` قبل الـ device teardown الطبيعي.

---

### المجموعة 6: الـ OF (Device Tree) Helper

---

#### `devm_get_clk_from_child`

```c
struct clk *devm_get_clk_from_child(struct device *dev,
                                     struct device_node *np, const char *con_id)
```

**ما بتعمله:**
بتجيب clock من child device node في الـ Device Tree (مش من الـ device نفسه). بتستخدم `of_clk_get_by_name` بدل `clk_get`. الـ lifetime بتاع الـ clock بتترابط بالـ `dev` مش بالـ `np`.

**الـ Parameters:**
- `dev` — الـ device اللي الـ devres هيتربط بـ lifetime بتاعه
- `np` — الـ child `device_node` في الـ DT اللي فيه الـ clock definition
- `con_id` — اسم الـ clock في الـ child node (`clock-names`)، أو `NULL` للأول

**الـ Return:** `struct clk *` أو `ERR_PTR`.

**Key Details:**
- مفيش `init`/`exit` callbacks — الـ cleanup هو `clk_put` فقط عبر `devm_clk_release` (بس الـ exit هيكون `NULL`).
- الـ pattern ده شائع في الـ MFD drivers اللي عندها child nodes بيعرفوا clocks لأجهزة sub-peripheral.
- `EXPORT_SYMBOL` (non-GPL).

---

### الـ Data Structures

#### `struct devm_clk_state`

```c
struct devm_clk_state {
    struct clk *clk;            /* the managed clock handle */
    void (*exit)(struct clk *clk); /* cleanup callback (unprepare/disable) */
};
```

الـ devres payload للـ single clock. بيتعمل allocate بـ `devres_alloc` وبيتحفظ جوه الـ devres list للـ device. الـ `exit` pointer هو المفتاح اللي بيخلي نفس الـ destructor (`devm_clk_release`) يتعامل مع حالات مختلفة (get-only، get+prepare، get+prepare+enable).

#### `struct clk_bulk_devres`

```c
struct clk_bulk_devres {
    struct clk_bulk_data *clks; /* pointer to the bulk data array */
    int num_clks;               /* number of clocks in the array */
};
```

الـ devres payload للـ bulk clocks. بيحتفظ بـ pointer للـ array (مش copy) والعدد. الـ destructor المسجل (واحد من الأربعة bulk release functions) بيستخدم الـ num_clks عشان يعرف كام clock يعمل put ليها.
## Phase 5: دليل الـ Debugging الشامل

---

### Software Level

---

#### 1. مدخلات الـ debugfs المتعلقة بالـ clk-devres

الـ **debugfs** للـ clock subsystem موجود في `/sys/kernel/debug/clk/`.

```bash
# mount debugfs لو مش mounted
mount -t debugfs none /sys/kernel/debug

# شوف شجرة الـ clocks الكاملة
cat /sys/kernel/debug/clk/clk_summary

# شوف clock معين باسمه
cat /sys/kernel/debug/clk/<clock_name>/clk_rate
cat /sys/kernel/debug/clk/<clock_name>/clk_enable_count
cat /sys/kernel/debug/clk/<clock_name>/clk_prepare_count
cat /sys/kernel/debug/clk/<clock_name>/clk_flags
```

| المدخل | المعنى |
|--------|--------|
| `clk_rate` | التردد الحالي بالـ Hz |
| `clk_enable_count` | عدد مرات `clk_enable` بدون `clk_disable` مقابلها |
| `clk_prepare_count` | عدد مرات `clk_prepare` بدون `clk_unprepare` |
| `clk_flags` | flags الـ clock زي `CLK_IS_ROOT`, `CLK_SET_RATE_GATE` |
| `clk_summary` | جدول شامل لكل الـ clocks في النظام |

```bash
# مثال عملي: تتبع clock اسمه "pll0"
grep -A 5 "pll0" /sys/kernel/debug/clk/clk_summary
# Output مثال:
#                                  enable  prepare  protect                                duty  hardware
#   clock                          count   count    count        rate   accuracy phase  cycle    enable
# ----------------------------------------------------------------------------------------
#   pll0                               1       1        0   800000000          0     0  50000         Y
```

**الـ devres** نفسها مش ليها debugfs entries مباشرة — الـ debugging بيتم عبر `clk_summary` + `devres` dump.

```bash
# شوف devres المسجلة للـ device
cat /sys/kernel/debug/devices_deferred   # للـ deferred probe issues
```

---

#### 2. مدخلات الـ sysfs المتعلقة بالـ clk-devres

```bash
# الـ clock info عبر sysfs مش متاحة مباشرة في كل kernel
# لكن الـ device نفسها ممكن تتعامل معاها كده:

# شوف الـ device المرتبطة بالـ clock
ls /sys/bus/platform/devices/<device_name>/

# الـ power domain وحالة الـ runtime pm
cat /sys/bus/platform/devices/<device_name>/power/runtime_status
cat /sys/bus/platform/devices/<device_name>/power/runtime_active_time

# لو الـ driver مش probe بسبب missing clock
cat /sys/bus/platform/devices/<device_name>/driver  # empty = probe فشل
```

---

#### 3. الـ ftrace — tracepoints وإزاي تفعلها

الـ **tracepoints** الخاصة بالـ clock subsystem:

```bash
# شوف الـ events المتاحة
ls /sys/kernel/debug/tracing/events/clk/

# Events المهمة لـ clk-devres:
# clk_get, clk_put, clk_prepare, clk_unprepare, clk_enable, clk_disable
# clk_set_rate, clk_bulk_prepare, clk_bulk_enable

# فعّل كل events الـ clk
echo 1 > /sys/kernel/debug/tracing/events/clk/enable

# أو فعّل event معين بس
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_prepare/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_enable/enable
echo 1 > /sys/kernel/debug/tracing/events/clk/clk_set_rate/enable

# ابدأ الـ trace
echo 1 > /sys/kernel/debug/tracing/tracing_on

# شوف الـ output
cat /sys/kernel/debug/tracing/trace

# أو بشكل live
cat /sys/kernel/debug/tracing/trace_pipe
```

**مثال على output:**

```
# tracer: nop
#
          <idle>-0     [000]  1234.567890: clk_prepare: pll0
          <idle>-0     [000]  1234.567891: clk_enable:  pll0
    my_driver-123  [001]  1234.568000: clk_set_rate: pll0 rate=400000000
```

**لتتبع مشكلة `devm_clk_get` بالذات:**

```bash
# فعّل function tracer على الـ functions المحددة
echo __devm_clk_get > /sys/kernel/debug/tracing/set_ftrace_filter
echo devm_clk_release >> /sys/kernel/debug/tracing/set_ftrace_filter
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

---

#### 4. الـ printk والـ dynamic debug

```bash
# فعّل dynamic debug للـ clock subsystem كله
echo "file drivers/clk/clk-devres.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file drivers/clk/*.c +p" > /sys/kernel/debug/dynamic_debug/control

# فعّل بـ module name (لو الـ clock مبني كـ module)
echo "module clk_devres +p" > /sys/kernel/debug/dynamic_debug/control

# شوف الـ active dynamic debug entries
cat /sys/kernel/debug/dynamic_debug/control | grep clk

# رفع مستوى الـ kernel log لشوف كل الـ debug messages
echo 8 > /proc/sys/kernel/printk
# أو
dmesg -n 8
```

**لتتبع مشاكل الـ devres release:**

```bash
# فعّل debug للـ devres infrastructure نفسها
echo "file lib/devres.c +p" > /sys/kernel/debug/dynamic_debug/control
```

---

#### 5. الـ Kernel Config options للـ Debugging

| CONFIG | الوظيفة |
|--------|---------|
| `CONFIG_DEBUG_KERNEL` | يفعّل كل options الـ debugging الأساسية |
| `CONFIG_COMMON_CLK` | لازم يكون enabled لأي clock framework debugging |
| `CONFIG_CLK_DEBUG` | يضيف مدخلات debugfs للـ clock framework |
| `CONFIG_DEBUG_DEVRES` | يطبع devres alloc/free operations في الـ dmesg |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل `pr_debug()` و `dev_dbg()` بشكل selective |
| `CONFIG_TRACING` | البنية التحتية للـ ftrace |
| `CONFIG_DEBUG_OBJECTS` | يتتبع الـ objects زي struct clk لو اتحررت غلط |
| `CONFIG_KASAN` | يكشف memory corruption في الـ devres allocations |
| `CONFIG_LOCKDEP` | يكشف deadlocks في mutex الـ clock framework |
| `CONFIG_PROVE_LOCKING` | يثبت صحة استخدام الـ locks |
| `CONFIG_DEBUG_SLAB` / `CONFIG_SLUB_DEBUG` | يكشف use-after-free في الـ devres state |

```bash
# تحقق من الـ config الحالية
zcat /proc/config.gz | grep -E "CONFIG_(CLK_DEBUG|DEBUG_DEVRES|DYNAMIC_DEBUG)"
```

---

#### 6. أدوات subsystem-specific

**أداة `clk_summary`:**

```bash
# جدول كامل لكل الـ clocks مع الـ parents والـ rates
cat /sys/kernel/debug/clk/clk_summary

# فلترة clock معين
grep -B2 -A5 "uart_clk" /sys/kernel/debug/clk/clk_summary
```

**أداة `devres` dump:**

```bash
# لو CONFIG_DEBUG_DEVRES=y، الـ kernel يطبع في dmesg عند probe/remove
# مثال:
dmesg | grep -i "devres\|devm_clk"
```

**أداة `clk-provider` inspection:**

```bash
# شوف الـ parents لكل clock
cat /sys/kernel/debug/clk/<clk_name>/clk_parent

# شوف الـ children
ls /sys/kernel/debug/clk/<clk_name>/
```

**`devmem2` لقراءة CCF (Common Clock Framework) registers:**

```bash
# بعد معرفة عنوان الـ register من الـ datasheet
devmem2 0xFE000000 w   # اقرأ register الـ clock control
```

---

#### 7. رسالة الخطأ → المعنى → الحل

| رسالة الـ kernel log | المعنى | الحل |
|----------------------|--------|------|
| `devm_clk_get: failed to get clock 'uart_clk'` | الـ clock مش موجود في الـ DT أو الـ clock provider مش registered | تأكد من الـ DT node وإن الـ clock provider probe بنجاح |
| `WARN_ON(!c \|\| !*c)` في `devm_clk_match` | الـ `devm_clk_put` اتعمل على pointer null أو invalid | تحقق إن الـ `clk` اللي بتعمله `put` هو نفس اللي جاب من `devm_clk_get` |
| `WARN_ON(ret)` في `devm_clk_put` | `devres_release` فشل يلاقي الـ resource | الـ clock مش متسجل في الـ devres — ممكن اتعمله `put` مرتين أو اتأخذ بـ non-devm `clk_get` |
| `-ENOENT` من `clk_get` | الـ clock مش موجود وrequest بالاسم | افحص `clocks` property في الـ DT |
| `-EPROBE_DEFER` من `clk_get` | الـ clock provider لسه مش probe | انتظر أو تحقق من ترتيب الـ initcalls |
| `-ENOMEM` من `devres_alloc` | ماكوش ذاكرة كافية لـ devres state | نادر في الـ production — افحص memory pressure |
| `clk_prepare_enable failed` | الـ hardware مش شغال أو الـ parent clock مش enabled | فعّل الـ parent clock أولاً، افحص power domains |
| `clk_set_rate: rate not supported` | الـ rate المطلوب مش ضمن حدود الـ clock | افحص `clk_round_rate` قبل `clk_set_rate` |
| `disable_unused_clocks: clock X` | kernel بيوقف clock مش بيتستخدم | لو الـ driver محتاجه، تأكد إنه enabled وعنده `prepare_count > 0` |

---

#### 8. أماكن استراتيجية لـ `dump_stack()` و`WARN_ON()`

الـ `clk-devres.c` فعلاً بيستخدم `WARN_ON` في مكانين مهمين:

```c
/* في devm_clk_match — يحمي من null pointer */
static int devm_clk_match(struct device *dev, void *res, void *data)
{
    struct clk **c = res;
    if (!c || !*c) {
        WARN_ON(!c || !*c); /* <-- هنا: لو res فيه مشكلة */
        return 0;
    }
    return *c == data;
}

/* في devm_clk_put — يحمي من double-put أو put على clock غير مسجل */
void devm_clk_put(struct device *dev, struct clk *clk)
{
    int ret;
    ret = devres_release(dev, devm_clk_release, devm_clk_match, clk);
    WARN_ON(ret); /* <-- هنا: لو مالقاش الـ resource */
}
```

**أماكن إضافية مفيد تحط فيها debugging لو بتعدل الـ kernel:**

```c
/* في __devm_clk_get — بعد فشل init */
err_clk_init:
    dump_stack(); /* اطبع الـ call stack لمعرفة مين طلب الـ clock */
    clk_put(clk);

/* في devm_clk_release — عند الـ cleanup */
static void devm_clk_release(struct device *dev, void *res)
{
    struct devm_clk_state *state = res;
    dev_dbg(dev, "releasing clock, exit=%ps\n", state->exit); /* اطبع الـ exit function */
    if (state->exit)
        state->exit(state->clk);
    clk_put(state->clk);
}
```

---

### Hardware Level

---

#### 1. التحقق إن الـ Hardware State يطابق الـ Kernel State

```bash
# الـ kernel يقول إن pll0 enabled بـ rate 800MHz
cat /sys/kernel/debug/clk/pll0/clk_rate          # 800000000
cat /sys/kernel/debug/clk/pll0/clk_enable_count  # يجب أن يكون >= 1

# تحقق من الـ hardware فعلاً
# اقرأ register الـ PLL lock bit من الـ SoC memory map
devmem2 0xFE001000 w   # عنوان مثال — ارجع للـ datasheet

# تحقق من power domain
cat /sys/kernel/debug/pm_genpd/pm_genpd_summary
```

#### 2. تقنيات الـ Register Dump

```bash
# باستخدام devmem2 (محتاج تثبيته أو build من source)
devmem2 <phys_addr> [b|h|w]   # b=byte, h=halfword, w=word

# مثال: قراءة clock control register
devmem2 0x10000000 w

# باستخدام /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((0x10000000/4)) 2>/dev/null | xxd

# باستخدام io utility (من package ioport أو busybox)
io -4 -r 0x10000000

# قراءة range كامل من registers
for addr in $(seq 0x10000000 4 0x10000100); do
    printf "0x%08x: " $addr
    devmem2 $addr w 2>/dev/null | grep "Read at"
done
```

**ملاحظة:** `/dev/mem` محتاج `CONFIG_DEVMEM=y` وممكن يكون مقيد بـ `CONFIG_STRICT_DEVMEM`.

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

- **تحقق من الـ clock output:** ضع probe على الـ clock pin في الـ PCB — التردد لازم يطابق `clk_rate`.
- **تحقق من الـ enable signal:** بعض الـ SoCs ليها clock enable GPIO — راقبه مع `clk_enable_count`.
- **راقب الـ startup sequence:** الـ clock لازم يتحول لـ stable قبل ما الـ device تبدأ تشتغل — فيه `clk_prepare` عشان ده.
- **قيس الـ jitter:** clock بـ jitter عالية ممكن تسبب مشاكل intermittent حتى لو الـ kernel يقول إنه enabled.
- **تحقق من الـ parent clock أولاً:** لو الـ child clock غلط، الـ parent ممكن يكون المشكلة.

```
Oscilloscope Connections:
  ┌──────────────┐
  │   SoC        │
  │  CLK_OUT ────┼──── CH1 (scope)   ← قيس الـ frequency
  │  CLK_EN  ────┼──── CH2 (scope)   ← راقب الـ enable signal
  └──────────────┘
```

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ kernel log

| مشكلة الـ Hardware | النمط في الـ kernel log | التشخيص |
|--------------------|------------------------|---------|
| الـ clock oscillator مش شغال | `clk_prepare_enable failed` ثم device timeout | راجع الـ crystal/oscillator بالـ oscilloscope |
| power domain مش enabled | `-ENODEV` أو hang عند `clk_enable` | افحص voltage rails بالـ multimeter |
| الـ clock rate مش stable | errors متقطعة في الـ device driver | قيس الـ jitter بالـ scope |
| الـ parent clock خاطئ | `clk_set_rate: invalid rate` | تحقق من الـ PLL configuration registers |
| clock glitch عند switch | device reset مفاجئ عند `clk_set_rate` | استخدم `clk_set_rate_range` أو تحقق من الـ reparenting |

#### 5. الـ Device Tree Debugging

```bash
# تحقق إن الـ DT محمّل صح
cat /proc/device-tree/<node_path>/clocks | xxd
# أو
fdtdump /boot/dtb/<board>.dtb | grep -A 10 "uart0"

# شوف الـ DT compiled اللي الـ kernel بيستخدمه فعلاً
ls /sys/firmware/devicetree/base/
cat /sys/firmware/devicetree/base/<node>/clock-names

# تحقق من الـ clock phandle
# في الـ DT:
#   clocks = <&pll0 0>;   ← phandle للـ pll0، index 0
# في الـ kernel:
grep -r "pll0" /sys/firmware/devicetree/base/ 2>/dev/null

# تحقق من الـ clock-names property
cat /sys/firmware/devicetree/base/<device_node>/clock-names
# لازم يطابق الـ id اللي بتبعته لـ devm_clk_get(dev, "uart_clk")
```

**مشكلة شائعة:** الـ `con_id` في `devm_clk_get(dev, "uart_clk")` لازم يطابق بالظبط الـ string في `clock-names` في الـ DT.

```dts
/* صح */
uart0: serial@10000000 {
    clocks = <&clkc 17>;
    clock-names = "uart_clk";   /* ← لازم يطابق الـ id في devm_clk_get */
};
```

```bash
# تحقق من الـ of_clk_get_by_name للـ child nodes (devm_get_clk_from_child)
cat /sys/firmware/devicetree/base/<parent>/<child>/clock-names
```

---

### Practical Commands

---

#### مجموعة أوامر جاهزة للنسخ

**1. فحص سريع لحالة الـ clocks:**

```bash
#!/bin/bash
# clk_check.sh - فحص سريع لحالة الـ clock subsystem

echo "=== Clock Summary ==="
cat /sys/kernel/debug/clk/clk_summary | column -t

echo ""
echo "=== Enabled Clocks Only ==="
awk 'NR==1 || $3 > 0' /sys/kernel/debug/clk/clk_summary
```

**2. تفعيل trace كامل لـ clock events:**

```bash
#!/bin/bash
# clk_trace.sh

# صفّر الـ trace buffer
echo "" > /sys/kernel/debug/tracing/trace

# فعّل كل events الـ clock
echo 1 > /sys/kernel/debug/tracing/events/clk/enable

# شغّل عملية الـ probe
echo 1 > /sys/kernel/debug/tracing/tracing_on
modprobe my_driver   # أو echo device_name > /sys/bus/platform/drivers/my_driver/bind
sleep 2
echo 0 > /sys/kernel/debug/tracing/tracing_on

# اعرض النتائج
cat /sys/kernel/debug/tracing/trace | grep -v "^#"
```

**3. مراقبة مشكلة `EPROBE_DEFER`:**

```bash
# شوف الـ devices اللي في deferred probe
cat /sys/kernel/debug/devices_deferred

# مثال output:
# my_uart.0: Driver my_driver requests probe deferral
# Reason: devm_clk_get failed: -517 (EPROBE_DEFER)
```

**4. تفعيل `DEBUG_DEVRES` بدون recompile (dynamic debug):**

```bash
# شوف كل devres operations
echo "file lib/devres.c +pflmt" > /sys/kernel/debug/dynamic_debug/control
dmesg -w | grep devres &

# بعدين trigger الـ probe
echo my_device > /sys/bus/platform/drivers/my_driver/bind
```

**5. فحص مشكلة `devm_clk_put` double-free:**

```bash
# فعّل KASAN (محتاج kernel مبني بـ CONFIG_KASAN=y)
# الـ output هيظهر في dmesg تلقائياً لو فيه مشكلة

dmesg | grep -A 20 "BUG: KASAN"
# لو لقيت:
# BUG: KASAN: use-after-free in devm_clk_match
# Read of size 8 at addr ffff888...
# ← ده معناه double-put أو put على freed clock
```

**6. تتبع `devm_clk_get_optional_enabled_with_rate`:**

```bash
# فعّل trace على الـ functions المحددة في clk-devres.c
echo devm_clk_get_optional_enabled_with_rate > /sys/kernel/debug/tracing/set_ftrace_filter
echo __devm_clk_get >> /sys/kernel/debug/tracing/set_ftrace_filter
echo clk_set_rate >> /sys/kernel/debug/tracing/set_ftrace_filter
echo clk_prepare_enable >> /sys/kernel/debug/tracing/set_ftrace_filter
echo devm_clk_put >> /sys/kernel/debug/tracing/set_ftrace_filter

echo function_graph > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
# ... trigger the driver probe ...
cat /sys/kernel/debug/tracing/trace
```

**مثال output للـ function_graph:**

```
 1)               |  devm_clk_get_optional_enabled_with_rate() {
 1)               |    __devm_clk_get() {
 1)   0.523 us    |      devres_alloc();
 1)   1.234 us    |      clk_get_optional();
 1)               |    }
 1)   2.100 us    |    clk_set_rate();       /* rate=400000000 */
 1)   0.876 us    |    clk_prepare_enable();
 1)               |  }
```

**7. فحص bulk clocks:**

```bash
# لو بتستخدم devm_clk_bulk_get_all
# شوف عدد الـ clocks اللي اتجابت
dmesg | grep "clk_bulk"

# بعد تفعيل dynamic debug:
echo "file drivers/clk/clk-devres.c +p" > /sys/kernel/debug/dynamic_debug/control
dmesg -w
```

**8. فحص الـ clock consumer من الـ DT:**

```bash
# اطبع كل الـ clock consumers لـ device معينة
python3 - <<'EOF'
import os, struct

dt_base = "/sys/firmware/devicetree/base"
device = "serial@10000000"  # غير ده لاسم device الـ

node = os.path.join(dt_base, device)
for prop in ["clock-names", "clocks"]:
    path = os.path.join(node, prop)
    if os.path.exists(path):
        with open(path, "rb") as f:
            data = f.read()
        print(f"{prop}: {data}")
EOF
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: UART مش بيشتغل على بورد RK3562 — Industrial Gateway

#### العنوان
**الـ UART driver بيفشل في الـ probe بسبب clock leak من session سابقة**

#### السياق
شركة بتبني industrial gateway على RK3562. الـ product بيستخدم UART2 للتواصل مع Modbus RTU devices. البورد اتعمل لها bringup جديد، والـ kernel بيتعمله upgrade من 5.15 لـ 6.6.

#### المشكلة
بعد الـ upgrade، الـ UART driver بدأ يطلع kernel panic عند الـ suspend/resume:

```
kernel BUG at drivers/clk/clk.c:1234!
clk_prepare_lock: already prepared
```

الـ engineer لقى إن في بعض الأحيان الـ UART driver بيتعمله probe مرتين بسبب deferred probe، وبعدين الـ devres مش بيتعمله cleanup صح.

#### التحليل
الـ driver كان بيستخدم `devm_clk_get_enabled` اللي بتعمل `clk_prepare_enable` جوا `__devm_clk_get`:

```c
struct clk *devm_clk_get_enabled(struct device *dev, const char *id)
{
    return __devm_clk_get(dev, id, clk_get,
                          clk_prepare_enable, clk_disable_unprepare);
}
```

لما الـ probe بيفشل في خطوة تانية بعد الـ `devm_clk_get_enabled`، الـ devres framework بيشيل كل الـ resources تلقائي — بما فيهم الـ `devm_clk_state` اللي `exit` فيه هو `clk_disable_unprepare`.

المشكلة الحقيقية: الـ driver كان بيعمل `clk_prepare_enable` يدوي تاني مرة في كود قديم ورثه الـ engineer، وده خلى الـ prepare count يعدي الـ enable count.

```c
static void devm_clk_release(struct device *dev, void *res)
{
    struct devm_clk_state *state = res;

    if (state->exit)
        state->exit(state->clk);  /* بيعمل clk_disable_unprepare */

    clk_put(state->clk);
}
```

الـ devres لما بيتعمل له release، بيستدعي `clk_disable_unprepare` مرة، لكن الـ driver اليدوي كان عمل enable مرتين فبقى فيه prepare count = 1 وما اتعملش unprepare.

#### الحل
شيل أي `clk_prepare_enable` يدوي من الـ probe، وخلي `devm_clk_get_enabled` هي المسؤولة الوحيدة:

```c
/* قبل الإصلاح — خطأ */
priv->uart_clk = devm_clk_get_enabled(dev, "uart");
clk_prepare_enable(priv->uart_clk);  /* مرة تانية! */

/* بعد الإصلاح */
priv->uart_clk = devm_clk_get_enabled(dev, "uart");
if (IS_ERR(priv->uart_clk))
    return PTR_ERR(priv->uart_clk);
/* خلاص، الـ devres هيعمل disable+unprepare+put تلقائي */
```

للتحقق:
```bash
cat /sys/kernel/debug/clk/uart2/clk_enable_count
cat /sys/kernel/debug/clk/uart2/clk_prepare_count
```

#### الدرس المستفاد
`devm_clk_get_enabled` بتعمل prepare+enable وبتسجل `clk_disable_unprepare` كـ exit callback. أي `clk_prepare_enable` يدوي تاني هو bug مؤكد. الـ devres مش بيحمي من double-enable، هو بس بيضمن الـ cleanup عند الـ device removal.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI مش بيظهر

#### العنوان
**الـ HDMI driver بيعمل probe ناجح لكن الشاشة مش بتتعرف — bulk clock لم يتفعل**

#### السياق
TV box product على Allwinner H616 بيشغّل Android 12. الـ HDMI controller محتاج 4 clocks: `ahb`, `mmc`, `tmds`, `cec`. الـ engineer استخدم `devm_clk_bulk_get` وبعدين نسي يعمل enable.

#### المشكلة
الـ probe بيرجع 0 (success) والـ dmesg مفيش فيه errors، لكن الـ HDMI مش بيشتغل خالص. الشاشة مش بتتعرف حتى لو الكابل صح.

#### التحليل
```c
/* من الـ driver */
static struct clk_bulk_data hdmi_clks[] = {
    { .id = "ahb"  },
    { .id = "mmc"  },
    { .id = "tmds" },
    { .id = "cec"  },
};

static int hdmi_probe(struct platform_device *pdev)
{
    /* ده بيعمل get بس — مش enable */
    ret = devm_clk_bulk_get(dev, ARRAY_SIZE(hdmi_clks), hdmi_clks);
    if (ret)
        return ret;
    /* نسي clk_bulk_prepare_enable! */
    ...
}
```

الـ `devm_clk_bulk_get` جوا `__devm_clk_bulk_get`:

```c
static int __devm_clk_bulk_get(struct device *dev, int num_clks,
                               struct clk_bulk_data *clks, bool optional)
{
    ...
    if (optional)
        ret = clk_bulk_get_optional(dev, num_clks, clks);
    else
        ret = clk_bulk_get(dev, num_clks, clks);
    if (!ret) {
        devres->clks = clks;
        devres->num_clks = num_clks;
        devres_add(dev, devres);
    }
    ...
}
```

الـ release callback المسجل هو `devm_clk_bulk_release` اللي بيعمل `clk_bulk_put` بس — مش `disable_unprepare`. يعني الـ clocks اتحجزت بس ما اتفعّلتش.

#### الحل
استبدل `devm_clk_bulk_get` بـ `devm_clk_bulk_get_all` أو استخدم `clk_bulk_prepare_enable` بعدها:

```c
/* الحل 1: استخدم الـ variant اللي بتعمل enable تلقائي */
static struct clk_bulk_data *hdmi_clks;

ret = devm_clk_bulk_get_all_enabled(dev, &hdmi_clks);
if (ret < 0)
    return ret;
/* ret = عدد الـ clocks، والـ devres هيعمل disable+unprepare+put_all */
```

```c
/* الحل 2: لو محتاج تتحكم في الـ enable timing */
ret = devm_clk_bulk_get(dev, ARRAY_SIZE(hdmi_clks), hdmi_clks);
if (ret)
    return ret;

ret = clk_bulk_prepare_enable(ARRAY_SIZE(hdmi_clks), hdmi_clks);
if (ret)
    return ret;
/* لازم تعمل disable_unprepare يدوي في remove! */
```

```bash
# للتشخيص
cat /sys/kernel/debug/clk/hdmi-ahb/clk_enable_count
# لو 0 مع إن الـ probe نجح — ده هو الـ bug
```

#### الدرس المستفاد
`devm_clk_bulk_get` بتعمل **get فقط** — مش prepare ولا enable. لو الـ hardware محتاج الـ clock يكون running من أول الـ probe، استخدم `devm_clk_bulk_get_all_enabled` أو `devm_clk_bulk_get_optional_enable` حسب الحاجة.

---

### السيناريو 3: IoT Sensor على STM32MP1 — Memory Leak في الـ Probe Error Path

#### العنوان
**الـ SPI sensor driver بيعمل memory leak لما الـ DT مش كامل**

#### السياق
شركة بتعمل IoT edge device على STM32MP1. الـ sensor driver بيتعامل مع SPI accelerometer. في بعض البوردات الـ `spi-clk` مش موجود في الـ DT لأنه optional على hardware revision معينة.

#### المشكلة
بعد آلاف الـ probe/remove cycles في stress test، الـ system بيلاقي memory exhaustion. الـ `kmemleak` بيرجع:

```
unreferenced object 0xc3a45800 (size 32):
  comm "kworker", pid 45
  backtrace:
    devres_alloc
    __devm_clk_get
    devm_clk_get
    spi_sensor_probe
```

#### التحليل
الـ driver كان بيستخدم `devm_clk_get` مع clock اسمه `spi-clk` وهو absent من الـ DT:

```c
priv->clk = devm_clk_get(dev, "spi-clk");
if (IS_ERR(priv->clk)) {
    /* الـ engineer كتب ده ظناً إنه كافي */
    dev_warn(dev, "no spi-clk, using default\n");
    priv->clk = NULL;
    /* لكن الـ state اللي اتعمله alloc في __devm_clk_get اتحرر؟ */
}
```

لما ننظر لـ `__devm_clk_get`:

```c
static struct clk *__devm_clk_get(...)
{
    state = devres_alloc(devm_clk_release, sizeof(*state), GFP_KERNEL);
    if (!state)
        return ERR_PTR(-ENOMEM);

    clk = get(dev, id);
    if (IS_ERR(clk)) {
        ret = PTR_ERR(clk);
        goto err_clk_get;  /* بيروح هنا */
    }
    ...

err_clk_get:
    devres_free(state);  /* ✓ الـ state بيتحرر صح */
    return ERR_PTR(ret);
}
```

الـ `state` بيتحرر صح في error path. إذن الـ leak مش من هنا مباشرة. الـ engineer لقى إن الـ bug الحقيقي كان في كود لاحق: بعد ما الـ probe نجح، الـ driver كان بيعمل `devm_clk_get` تاني في `resume` callback (خطأ معماري) — والـ `state` ده ما اتحررش لأن الـ devres قرنه بالـ device مرتين.

#### الحل
استخدم `devm_clk_get_optional` بدل `devm_clk_get` للـ clocks الـ optional، وازيل أي استدعاء لـ `devm_clk_*` من خارج الـ probe:

```c
/* صح — devm_clk_get_optional بترجع NULL لو مش موجود */
priv->clk = devm_clk_get_optional(dev, "spi-clk");
if (IS_ERR(priv->clk))
    return PTR_ERR(priv->clk);

/* priv->clk ممكن يكون NULL — لازم تتعامل معاه */
if (priv->clk) {
    ret = clk_prepare_enable(priv->clk);
    if (ret)
        return ret;
}
```

```bash
# للتشخيص بـ kmemleak
echo scan > /sys/kernel/debug/kmemleak
cat /sys/kernel/debug/kmemleak
```

#### الدرس المستفاد
`devm_clk_get` فاشل لو الـ clock مش موجود في الـ DT ورجع error. استخدم `devm_clk_get_optional` للـ optional clocks. وأهم من كده: **لا تستدعي `devm_clk_*` إلا من جوا `probe`** — كل الـ devm resources مربوطة بالـ device lifetime مش بالـ suspend/resume cycle.

---

### السيناريو 4: Automotive ECU على i.MX8 — Clock Rate خاطئ بيخرب الـ CAN Bus

#### العنوان
**الـ CAN controller بيشتغل بـ baud rate خاطئ بسبب `devm_clk_get_optional_enabled_with_rate` مش بيشتغل زي المتوقع**

#### السياق
automotive ECU على i.MX8MP بيستخدم FlexCAN controller للتواصل مع الـ vehicle network. الـ baud rate المطلوب 500 kbps. الـ engineer استخدم `devm_clk_get_optional_enabled_with_rate` لتبسيط الـ clock setup.

#### المشكلة
الـ CAN frames بتيجي corrupted والـ bus errors بتزيد. الـ oscilloscope بيكشف إن الـ clock frequency غلط — بدل 80 MHz بيديها 24 MHz (الـ default).

#### التحليل
ننظر لـ `devm_clk_get_optional_enabled_with_rate`:

```c
struct clk *devm_clk_get_optional_enabled_with_rate(struct device *dev,
                                                    const char *id,
                                                    unsigned long rate)
{
    struct clk *clk;
    int ret;

    /* لاحظ: بتعمل get مع exit=clk_disable_unprepare بس مش init */
    clk = __devm_clk_get(dev, id, clk_get_optional, NULL,
                         clk_disable_unprepare);
    if (IS_ERR(clk))
        return ERR_CAST(clk);

    ret = clk_set_rate(clk, rate);  /* بتحاول تضبط الـ rate */
    if (ret)
        goto out_put_clk;

    ret = clk_prepare_enable(clk);
    if (ret)
        goto out_put_clk;

    return clk;

out_put_clk:
    devm_clk_put(dev, clk);
    return ERR_PTR(ret);
}
```

المشكلة: `clk_set_rate` على i.MX8 للـ FlexCAN clock كان بيرجع 0 (success) لكن الـ rate الحقيقي ما اتغيرش لأن الـ clock هو `CLK_SET_RATE_GATE` — يعني لازم يكون disabled الأول عشان تغير الـ rate. الـ clock كان enabled من boot، والـ `clk_set_rate` نجح "formally" لكن ما عملش حاجة.

#### الحل
افحص الـ rate فعلاً بعد الـ set:

```c
priv->can_clk = devm_clk_get_optional_enabled_with_rate(dev, "can", 80000000UL);
if (IS_ERR(priv->can_clk))
    return PTR_ERR(priv->can_clk);

/* تحقق من الـ rate الفعلي */
actual_rate = clk_get_rate(priv->can_clk);
if (actual_rate != 80000000UL) {
    dev_err(dev, "CAN clock rate mismatch: got %lu, want 80MHz\n",
            actual_rate);
    return -EINVAL;
}
```

أو استخدم approach مختلف للـ CLK_SET_RATE_GATE clocks:

```c
/* 1. get بدون enable */
priv->can_clk = devm_clk_get(dev, "can");

/* 2. اضبط الـ rate وهو disabled */
ret = clk_set_rate(priv->can_clk, 80000000UL);

/* 3. enable بعدين */
ret = clk_prepare_enable(priv->can_clk);
```

```bash
# للتشخيص
cat /sys/kernel/debug/clk/flexcan1_root/clk_rate
# لازم يطلع 80000000
```

#### الدرس المستفاد
`devm_clk_get_optional_enabled_with_rate` بتعمل `set_rate` قبل الـ enable — ده صح لـ normal clocks. لكن لـ `CLK_SET_RATE_GATE` clocks، الـ set_rate ممكن يرجع 0 بدون ما يعمل حاجة. دايماً افحص `clk_get_rate` بعد الـ set للـ safety-critical applications.

---

### السيناريو 5: Custom Board Bring-up على AM62x — `devm_get_clk_from_child` بيفشل بعد DT Restructure

#### العنوان
**الـ display subsystem driver مش بيلاقي الـ clock بعد restructure للـ Device Tree**

#### السياق
custom industrial HMI board على TI AM62x. الـ display controller driver بيجيب pixel clock من child node اسمه `display-timings`. بعد restructure للـ DT عشان يتوافق مع upstream schema، الـ driver بدأ يفشل.

#### المشكلة
```
display-drm: failed to get pixel clock from display-timings: -ENOENT
```

الـ probe بيفشل، والشاشة مش بتشتغل.

#### التحليل
الـ driver كان بيستخدم `devm_get_clk_from_child`:

```c
struct clk *devm_get_clk_from_child(struct device *dev,
                                    struct device_node *np,
                                    const char *con_id)
{
    struct devm_clk_state *state;
    struct clk *clk;

    state = devres_alloc(devm_clk_release, sizeof(*state), GFP_KERNEL);
    if (!state)
        return ERR_PTR(-ENOMEM);

    clk = of_clk_get_by_name(np, con_id);  /* بيدور في الـ child node بالاسم */
    if (!IS_ERR(clk)) {
        state->clk = clk;
        devres_add(dev, state);
    } else {
        devres_free(state);
    }

    return clk;
}
```

`of_clk_get_by_name` بيدور على `clocks` property في الـ `np` node المحدد باسمه في `clock-names`. بعد الـ DT restructure، الـ engineer حرك الـ `clocks` property من `display-timings` node لـ parent node:

```dts
/* قبل الـ restructure — DT قديم */
display-timings {
    clocks = <&k3_clks 235 0>;
    clock-names = "pixel";
    ...
};

/* بعد الـ restructure — DT جديد */
display@fe010000 {
    clocks = <&k3_clks 235 0>;
    clock-names = "pixel";
    display-timings { ... };  /* مفيهاش clocks */
};
```

الـ driver لسه بيمرر `display-timings` node لـ `devm_get_clk_from_child` فبيدور في الـ child اللي ما فيهوش `clocks` property.

#### الحل
لو الـ clock انتقل للـ parent، استخدم `devm_clk_get` العادية بدل `devm_get_clk_from_child`:

```c
/* قبل الإصلاح */
np = of_get_child_by_name(dev->of_node, "display-timings");
priv->pclk = devm_get_clk_from_child(dev, np, "pixel");

/* بعد الإصلاح — لما الـ clock على الـ parent node */
priv->pclk = devm_clk_get(dev, "pixel");
if (IS_ERR(priv->pclk))
    return PTR_ERR(priv->pclk);
```

لو الـ use case لازم يبقى child node، رجّع الـ `clocks` property للـ child في الـ DT:

```dts
display-timings {
    clocks = <&k3_clks 235 0>;
    clock-names = "pixel";
};
```

```bash
# للتشخيص
fdtdump /boot/dtb/am625-custom-hmi.dtb | grep -A5 "display-timings"
# افتش على وجود "clocks" في الـ node الصح
```

#### الدرس المستفاد
`devm_get_clk_from_child` تحديداً بتستخدم `of_clk_get_by_name` على الـ **child node** اللي بتمرره — مش على الـ device node. لو الـ DT schema اتغير وحرك الـ `clocks` property، لازم تغير الـ driver كمان. وبشكل عام، `devm_get_clk_from_child` مفيدة بس لما الـ hardware model يفرض فعلاً إن الـ clock تتعرّف من subnode مختلف.
## Phase 7: مصادر ومراجع

### توثيق الكيرنل الرسمي

| المصدر | الرابط |
|--------|--------|
| **The Common Clk Framework** — توثيق رسمي للـ CCF | [docs.kernel.org/driver-api/clk.html](https://static.lwn.net/kerneldoc/driver-api/clk.html) |
| **Devres - Managed Device Resource** — شرح كامل لآلية الـ devres | [docs.kernel.org/driver-api/driver-model/devres.html](https://docs.kernel.org/driver-api/driver-model/devres.html) |
| **Clk API (KUnit)** — توثيق الـ API للـ clk testing | [docs.kernel.org/dev-tools/kunit/api/clk.html](https://docs.kernel.org/dev-tools/kunit/api/clk.html) |

**الـ** `Documentation/` paths في الكيرنل source:

```
Documentation/driver-api/clk.rst          ← الـ Common Clock Framework
Documentation/driver-api/driver-model/devres.rst  ← الـ devres infrastructure
Documentation/devicetree/bindings/clock/  ← الـ DT bindings للـ clocks
```

---

### مقالات LWN.net

| المقال | الأهمية |
|--------|---------|
| [A common clock framework](https://lwn.net/Articles/472998/) | مقال أساسي — بيشرح المشكلة اللي حلّها الـ CCF وازاي اتصمّم |
| [common clk framework](https://lwn.net/Articles/486841/) | نسخة أحدث من المقال الأول — بتتكلم عن الـ implementation النهائي |
| [Dynamic clock devices](https://lwn.net/Articles/413332/) | خلفية عن الـ dynamic clock management قبل الـ CCF |

---

### Kernel Commits المهمة

**الـ** `devm_clk_get()` — الـ commit الأصلي اللي أدخل الـ managed clock API:
- الـ patch series اللي أدخلت `devm_clk_get_prepared()` و`devm_clk_get_enabled()`:
  - [`[PATCH v8 01/16] clk: generalize devm_clk_get() a bit`](https://www.spinics.net/lists/linux-clk/msg67675.html)
  - Commit: `935ec78de50e21ac516b3d24d978281511b8417a` — *"clk: Provide new devm_clk helpers for prepared and enabled clocks"* (Uwe Kleine-König)

**الـ** `devm_clk_get_optional_enabled_with_rate()` — أُضيفت لاحقاً لتبسيط drivers اللي محتاجة set_rate قبل enable.

---

### نقاشات Mailing List

| النقاش | الرابط |
|--------|--------|
| `of_clk_get()` vs `devm_clk_get()` — متى تستخدم أيهما | [linux-arm-kernel 2013](http://lists.infradead.org/pipermail/linux-arm-kernel/2013-February/149651.html) |
| نقاش تفصيلي على narkive | [linux-arm-kernel infradead](https://linux-arm-kernel.infradead.narkive.com/1OsLSE6w/of-clk-get-devm-clk-get) |

---

### kernelnewbies.org

- [LinuxTimekeeping](https://kernelnewbies.org/LinuxTimekeeping) — شرح timekeeping في Linux بشكل عام، مفيد كخلفية قبل الدخول في الـ clock framework
- [Linux_6.13](https://kernelnewbies.org/Linux_6.13) — مثال على إضافات حديثة لـ clock controllers في الكيرنل

---

### elinux.org

| الصفحة | المحتوى |
|--------|---------|
| [Linux clock management framework (PDF)](https://elinux.org/images/a/a3/ELC_2007_Linux_clock_fmw.pdf) | presentation من ELC 2007 — خلفية تاريخية عن الـ clock management قبل الـ CCF |
| [Kernel Timer Systems](https://elinux.org/index.php/Kernel_Timer_Systems) | نظرة عامة على أنظمة الـ timing في Linux |
| [High Resolution Timers](https://elinux.org/High_Resolution_Timers) | الـ hrtimer infrastructure — مكمّل لفهم الـ clock sources |

---

### مصادر إضافية

**الـ** Bootlin Slides (مرجع ممتاز للـ embedded developers):
- [Common Clock Framework: How to use it — ELCE 2013](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf)

**الـ** devres internals:
- [The Right Way: Managed Resource Allocation in Linux Device Drivers](http://www.haifux.org/lectures/323/haifux-devres.pdf) — Eli Billauer — شرح عميق لآلية الـ devres

---

### كتب مُوصى بها

| الكتاب | الفصل ذو الصلة |
|--------|---------------|
| **Linux Device Drivers (LDD3)** — Corbet, Rubini, Kroah-Hartman | Chapter 14: The Linux Device Model — بيشرح الـ device/driver lifecycle الذي يعتمد عليه الـ devres |
| **Linux Kernel Development** — Robert Love | Chapter 17: Devices and Modules — سياق عام للـ driver infrastructure |
| **Embedded Linux Primer** — Christopher Hallinan | Chapter 15: Embedded Linux Kernel Resources — بيغطي الـ clock management في embedded SoCs |
| **Mastering Embedded Linux Programming** — Frank Vasquez & Chris Simmonds | Chapter 9: Interfacing with Device Drivers — أمثلة عملية على `devm_clk_get()` في drivers حقيقية |

---

### مصطلحات البحث

لو عايز تلاقي مزيد من المعلومات ابحث عن:

```
linux kernel devm_clk_get internals
linux common clock framework CCF tutorial
linux devres managed resources driver
clk_bulk_get linux driver example
linux clock provider consumer DT bindings
linux clk_prepare_enable devm
linux drivers/clk/ subsystem
```
## Phase 8: Writing simple module

### الفكرة

**`devm_clk_get`** هي الدالة الأكثر استخداماً في الملف — كل driver محتاج clock بيناديها. هنعمل **kprobe** عليها عشان نشوف أي device بيطلب clock وبأي `id`.

---

### الـ Module كامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe on devm_clk_get — prints device name + clock id on every call
 */
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/kprobes.h>
#include <linux/device.h>
#include <linux/clk.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("kprobe on devm_clk_get to trace clock requests");

/* ------------------------------------------------------------------ */
/*  kprobe handler — called just before devm_clk_get executes          */
/* ------------------------------------------------------------------ */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * On x86-64: first arg (dev)  -> rdi
     *            second arg (id)  -> rsi
     * On arm64:  first arg (dev)  -> x0
     *            second arg (id)  -> x1
     */
#ifdef CONFIG_X86_64
    struct device *dev = (struct device *)regs->di;
    const char    *id  = (const char *)regs->si;
#elif defined(CONFIG_ARM64)
    struct device *dev = (struct device *)regs->regs[0];
    const char    *id  = (const char *)regs->regs[1];
#else
    /* fallback — may not compile on other arches without adjustment */
    struct device *dev = NULL;
    const char    *id  = NULL;
#endif

    /* dev_name() returns the kobject name — safe to call here */
    pr_info("devm_clk_get: device='%s'  clock_id='%s'\n",
            dev ? dev_name(dev) : "(null)",
            id  ? id            : "(null)");

    return 0; /* 0 = let the real function run normally */
}

/* ------------------------------------------------------------------ */
/*  kprobe struct                                                       */
/* ------------------------------------------------------------------ */
static struct kprobe kp = {
    .symbol_name = "devm_clk_get", /* الدالة اللي هنعمل probe عليها */
    .pre_handler = handler_pre,
};

/* ------------------------------------------------------------------ */
/*  module_init                                                         */
/* ------------------------------------------------------------------ */
static int __init clk_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("kprobe planted on devm_clk_get at %p\n", kp.addr);
    return 0;
}

/* ------------------------------------------------------------------ */
/*  module_exit                                                         */
/* ------------------------------------------------------------------ */
static void __exit clk_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("kprobe on devm_clk_get removed\n");
}

module_init(clk_probe_init);
module_exit(clk_probe_exit);
```

---

### شرح كل جزء

#### الـ Includes

```c
#include <linux/kprobes.h>
#include <linux/device.h>
#include <linux/clk.h>
```

الـ `kprobes.h` بيجيب `struct kprobe` ودوال `register_kprobe` / `unregister_kprobe`. الـ `device.h` محتاجينه عشان `dev_name()` تشتغل صح من غير undefined behavior.

---

#### الـ `handler_pre` — قلب الـ module

```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
```

الـ `pt_regs` هو snapshot للـ CPU registers لحظة ما الـ kprobe اتفعّل. بنقرأ منه `di/si` على x86-64 أو `regs[0]/regs[1]` على arm64 عشان نجيب الـ arguments اللي اتبعتت لـ `devm_clk_get`. الـ return value صفر يعني "استمر في التنفيذ الطبيعي".

---

#### الـ `struct kprobe`

```c
static struct kprobe kp = {
    .symbol_name = "devm_clk_get",
    .pre_handler = handler_pre,
};
```

**`symbol_name`** بيخلي الـ kernel يحل العنوان تلقائياً من الـ kallsyms بدل ما نحدد عنوان صريح — أسهل وأكثر portability. الـ `pre_handler` بيتنادى *قبل* ما الدالة الأصلية تبدأ، وده المكان الصح لو عايزين نشوف الـ arguments.

---

#### الـ `module_init` / `module_exit`

`register_kprobe` بتزرع breakpoint في الـ kernel text segment على عنوان `devm_clk_get`. لو فشل (مثلاً الدالة inline أو محمية) بيرجع error سالب ونطبعه. الـ `unregister_kprobe` في الـ exit **إجبارية** — من غيرها الـ breakpoint بيفضل في الـ kernel بعد الـ rmmod وبيسبب kernel panic أو undefined behavior لأول حاجة تنادي الدالة.

---

### الـ Makefile لتجربة الـ module

```makefile
obj-m += clk_devm_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل
make
sudo insmod clk_devm_probe.ko

# مراقبة الـ output
sudo dmesg -wH | grep devm_clk_get

# إزالة
sudo rmmod clk_devm_probe
```

---

### مثال على الـ output المتوقع

```
[  +0.000123] kprobe planted on devm_clk_get at ffffffffc0a12340
[  +2.341000] devm_clk_get: device='1c28000.mmc'   clock_id='core'
[  +2.341050] devm_clk_get: device='1c28000.mmc'   clock_id='ahb'
[  +3.100200] devm_clk_get: device='4830000.i2c'   clock_id='fck'
```

كل سطر بيوريك اسم الـ device اللي طلب الـ clock واسم الـ clock اللي طلبه — معلومة مفيدة جداً في debugging مشاكل الـ clock tree أو في فهم تسلسل الـ driver initialization.
