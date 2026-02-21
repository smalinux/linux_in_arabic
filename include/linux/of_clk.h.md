## Phase 1: الصورة الكبيرة

# `of_clk.h`

> **PATH**: `include/linux/of_clk.h`
> **Subsystem**: Common CLK Framework (COMMON CLK FRAMEWORK)
> **الوظيفة الأساسية**: واجهة بين شجرة الأجهزة (Device Tree) ونظام إدارة الـ clock في Linux

---

### ما هو الـ Clock أصلاً؟

تخيل أن المعالج (CPU) هو قلب ينبض. كل جهاز داخل الـ SoC (كالـ USB أو الـ UART أو الـ I2C) يحتاج إلى "نبضة" تنظّم سرعته — هذه النبضة هي الـ **clock**.

بدون clock صحيح:
- الـ UART لن يتزامن مع الطرف الآخر فتضيع البيانات.
- الـ USB لن يعمل أبداً.
- وحدات الـ GPU ستعمل بسرعة خاطئة فتتعطل.

إذاً الـ **clock** هو إشارة تردّد منتظمة تتحكم في سرعة عمل كل وحدة في الـ hardware.

---

### المشكلة: كيف يعرف الـ Kernel من أين يأخذ الـ Clock؟

في الأنظمة المُدمجة (embedded systems)، الـ SoC يحتوي على عشرات الـ clock sources: PLL، crystal oscillators، dividers، muxes... إلخ.

كل board مختلف: Raspberry Pi لديها شجرة clock مختلفة عن BeagleBone أو Snapdragon.

الحل هو **Device Tree (DT)** — ملف نصي يصف الـ hardware للـ kernel، بما فيه:
- ما هي مصادر الـ clock المتاحة.
- أيها parent لأيها (شجرة هرمية).
- ماذا يستخدم كل device.

مثال في الـ Device Tree:
```
clks: clock-controller@1c20000 {
    compatible = "allwinner,sun4i-a10-ccu";
    #clock-cells = <1>;
    clocks = <&osc24M>, <&osc32k>;  /* مصادران للـ clock */
    clock-output-names = "pll-core", "pll-audio", ...;
};

uart0: serial@1c28000 {
    clocks = <&clks 16>;  /* يأخذ clock رقم 16 من الـ controller */
};
```

---

### دور `of_clk.h`: الجسر بين DT والـ Clock Framework

**الـ `of_clk.h`** هو header صغير جداً (33 سطراً) لكنه يُعرّف الـ API الذي يربط:

```
Device Tree (OF = Open Firmware)  <-->  Common CLK Framework
```

يُقدّم ثلاث وظائف فقط:

| الوظيفة | الغرض |
|---------|-------|
| `of_clk_get_parent_count(np)` | كم عدد الـ clock parents التي يحتاجها هذا الـ node في الـ DT؟ |
| `of_clk_get_parent_name(np, index)` | ما اسم الـ parent clock رقم N لهذا الـ node؟ |
| `of_clk_init(matches)` | امسح كل الـ DT وابدأ تشغيل كل clock providers المسجّلة |

---

### القصة الكاملة: ماذا يحدث عند boot؟

تخيل الـ kernel يبدأ التشغيل على SoC جديد:

```
1. الـ bootloader يُمرر Device Tree blob للـ kernel
         ↓
2. الـ kernel يقرأ الـ DT ويبني شجرة الـ device_node في الذاكرة
         ↓
3. of_clk_init() تُشغَّل مبكراً جداً (early boot)
   - تمسح كل nodes في الـ DT
   - تبحث عن nodes تطابق compatible strings مسجّلة بـ CLK_OF_DECLARE
   - تستدعي دالة الـ init الخاصة بكل clock controller
   - تحترم الترتيب: parent قبل child (تحل الـ dependencies أولاً)
         ↓
4. كل clock controller driver يُسجّل نفسه بـ of_clk_add_provider()
         ↓
5. لما device يريد clock، يستخدم of_clk_get_parent_name() ليعرف
   من أي provider يطلبه
         ↓
6. كل الـ devices تحصل على صحيح clocks وتبدأ العمل
```

---

### الـ Guard الذكي في الـ Header

```c
#if defined(CONFIG_COMMON_CLK) && defined(CONFIG_OF)
    /* الوظائف الحقيقية */
#else
    /* stubs فارغة */
#endif
```

هذا يعني:
- إذا الـ kernel مبني **بدون** دعم الـ Device Tree أو **بدون** الـ Common CLK Framework → تُعيد الوظائف قيم افتراضية آمنة (صفر أو NULL) بدون أي كود فعلي.
- هذا يسمح للكود العام أن يستدعي هذه الوظائف بأمان بغض النظر عن الـ configuration.

---

### ملفات يجب أن تعرفها

#### الملفات الأساسية للـ Subsystem

| الملف | الدور |
|-------|-------|
| `include/linux/of_clk.h` | **هذا الملف** — الـ OF/DT bridge API |
| `include/linux/clk.h` | الـ Consumer API — كيف يطلب driver الـ clock |
| `include/linux/clk-provider.h` | الـ Provider API — كيف يُسجّل driver clock جديد + `CLK_OF_DECLARE` |
| `include/linux/clkdev.h` | نظام البحث عن الـ clock بالاسم (lookup table) |
| `drivers/clk/clk.c` | **القلب** — تنفيذ كل وظائف `of_clk_*` + إدارة الـ clock tree |
| `drivers/clk/clkdev.c` | نظام الـ lookup |

#### مثال على Hardware Drivers

| الملف | الـ SoC |
|-------|---------|
| `drivers/clk/sunxi-ng/` | Allwinner SoCs (Raspberry Pi-like) |
| `drivers/clk/bcm/` | Broadcom SoCs |
| `drivers/clk/qcom/` | Qualcomm Snapdragon |
| `drivers/clk/samsung/` | Samsung Exynos |
| `drivers/clk/clk-fixed-rate.c` | Crystal oscillator ثابت التردد |

#### ملفات Device Tree Bindings

```
Documentation/devicetree/bindings/clock/
```

---

### كيف يستخدم الـ Driver هذا الـ Header؟

مثال driver يريد معرفة parents لـ clock controller:

```c
#include <linux/of_clk.h>

/* كم parent يوجد؟ */
unsigned int num_parents = of_clk_get_parent_count(np);

/* احصل على اسم كل parent */
for (int i = 0; i < num_parents; i++) {
    const char *name = of_clk_get_parent_name(np, i);
    /* استخدم الاسم لإيجاد الـ parent في الـ clock framework */
}
```

ومثال تسجيل clock provider جديد (يُشغَّل تلقائياً من `of_clk_init`):

```c
/* في ملف الـ driver */
CLK_OF_DECLARE(my_clk, "vendor,my-clock-ctrl", my_clk_init);
/*
 * هذا الـ macro يُسجّل: عندما تجد node بـ compatible = "vendor,my-clock-ctrl"
 * في الـ Device Tree، استدعِ my_clk_init(np)
 */
```

---

### الخلاصة

**الـ `of_clk.h`** هو header "جسر" صغير لكن حيوي. بدونه لن يستطيع أي clock driver أن يُهيّئ نفسه من الـ Device Tree، ولن تعمل أي وحدة hardware في الـ SoC بشكل صحيح. إنه الخيط الذي يربط الوصف النصي للـ hardware (Device Tree) بالكود الفعلي الذي يُدير الـ clocks في الـ kernel.
## Phase 2: شرح الـ OF Clock (of_clk) Framework

### المشكلة — ليش هذا الـ Subsystem موجود أصلاً؟

تخيّل عندك لوحة إلكترونية (SoC) فيها عشرات أو مئات الـ clocks — كل clock هو نبضة كهربائية بتحدّد سرعة شغل جزء من الـ hardware. بعض هذه الـ clocks مستقلة، وبعضها يعتمد على clock آخر (parent/child relationship). مثلاً:

```
Crystal Oscillator (24 MHz)
        │
        ▼
    PLL (1.2 GHz)
        │
   ┌────┴────┐
   ▼         ▼
CPU Clock  Bus Clock (divider /4 = 300 MHz)
(1.2 GHz)      │
               ▼
          UART Clock (divider /8 = 37.5 MHz)
```

المشكلة الأولى — **من يعرف هذه الـ clocks؟**

قديماً (قبل الـ Device Tree)، كان كل driver يكتب بـ hard-coded كيفية الحصول على الـ clock من الـ hardware. هذا يعني:
- لو غيّرت الـ SoC من Qualcomm إلى Samsung، يجب إعادة كتابة نصف الـ kernel
- مستحيل مشاركة الـ code بين platforms مختلفة

المشكلة الثانية — **كيف يعرف الـ kernel شجرة الـ clocks موجودة في الـ hardware فعلاً؟**

لا توجد طريقة موحّدة. كل board كانت لها initialization code خاصة مكتوبة بـ C داخل الـ kernel source.

---

### الحل — الـ Common Clock Framework + Device Tree

الـ kernel حلّ المشكلة بمحورين:

**المحور الأول: الـ Common Clock Framework (CCF)**
هو framework موحّد يوفّر API واحداً للتعامل مع كل الـ clocks بغض النظر عن الـ hardware. كل driver يطلب clock باسمه (`clk_get()`)، والـ framework يعرف أين يجده.

**المحور الثاني: الـ Device Tree (DT) + Open Firmware (OF)**
الـ Device Tree هو ملف وصف يصف الـ hardware كشجرة من الـ nodes. بدلاً من كتابة الـ hardware في الـ C code، تكتبه في ملف `.dts` مثل:

```dts
clocks {
    osc: oscillator {
        compatible = "fixed-clock";
        #clock-cells = <0>;
        clock-frequency = <24000000>;  /* 24 MHz */
    };

    pll: pll@10000 {
        compatible = "myvendor,pll";
        #clock-cells = <1>;
        clocks = <&osc>;   /* parent هو الـ oscillator */
        reg = <0x10000 0x100>;
    };
};

uart0: uart@20000 {
    compatible = "myvendor,uart";
    clocks = <&pll 2>;   /* استخدم output رقم 2 من الـ PLL */
};
```

**الـ of_clk** هو الجسر الذي يربط بين الـ Device Tree وبين الـ Common Clock Framework. مهمّته:
1. قراءة الـ clock nodes من الـ Device Tree
2. تسجيل الـ clock providers
3. حلّ العلاقات بين الـ clocks (من هو الـ parent لكل clock؟)

---

### تشبيه من الحياة الواقعية

تخيّل مدينة فيها شبكة كهرباء:

- **الـ Device Tree** = خريطة المدينة التي ترسم فيها مواقع المحطات والأسلاك والمنازل
- **الـ of_clk** = الشركة التي تقرأ الخريطة وتعرف من أين تبدأ بإيصال الكهرباء
- **الـ CCF** = نظام التوزيع الداخلي الذي يرسل الكهرباء لكل غرفة في كل منزل
- **الـ Driver** = الجهاز الكهربائي الذي يطلب الكهرباء فقط بدون أن يعرف من أين جاءت

---

### البنية الكبيرة — أين يقع الـ of_clk في الـ Kernel؟

```
┌─────────────────────────────────────────────────────────────┐
│                      User Space                             │
│              (لا يتعامل مع الـ clocks مباشرة)              │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                      Kernel Space                           │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Clock Consumer Drivers                  │  │
│  │   (UART, SPI, I2C, USB, GPU, etc.)                  │  │
│  │   يستخدمون: clk_get(), clk_enable(), clk_set_rate() │  │
│  └──────────────────────┬───────────────────────────────┘  │
│                         │                                    │
│  ┌──────────────────────▼───────────────────────────────┐  │
│  │         Common Clock Framework (CCF)                 │  │
│  │              drivers/clk/clk.c                       │  │
│  │  - إدارة شجرة الـ clocks في الذاكرة                  │  │
│  │  - تطبيق الـ enable/disable counting                  │  │
│  │  - حساب الـ rates                                     │  │
│  └──────┬───────────────────────────────────────────────┘  │
│         │                                                    │
│  ┌──────▼────────────────────────────────────────────────┐  │
│  │          OF Clock Bridge (of_clk)                     │  │
│  │    include/linux/of_clk.h + drivers/clk/clk.c        │  │
│  │                                                       │  │
│  │  of_clk_init()          ← يمشي على الـ DT            │  │
│  │  of_clk_get_parent_name() ← يقرأ اسم الـ parent      │  │
│  │  of_clk_get_parent_count() ← يعدّ الـ parents        │  │
│  └──────┬────────────────────────────────────────────────┘  │
│         │                                                    │
│  ┌──────▼────────────────────────────────────────────────┐  │
│  │          Device Tree / Open Firmware (OF)             │  │
│  │    include/linux/of.h                                 │  │
│  │                                                       │  │
│  │  struct device_node ← كل node في الـ DT              │  │
│  │  struct property    ← خصائص الـ node                  │  │
│  └──────┬────────────────────────────────────────────────┘  │
│         │                                                    │
│  ┌──────▼────────────────────────────────────────────────┐  │
│  │       Clock Provider Drivers (Hardware-specific)      │  │
│  │                                                       │  │
│  │  drivers/clk/clk-fixed-rate.c  (Fixed Oscillator)    │  │
│  │  drivers/clk/qcom/             (Qualcomm PLLs)        │  │
│  │  drivers/clk/samsung/          (Samsung PLLs)         │  │
│  │  drivers/clk/imx/              (i.MX PLLs)            │  │
│  │                                                       │  │
│  │  كل driver يسجّل نفسه عبر:                            │  │
│  │  CLK_OF_DECLARE() أو clk_hw_register()                │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                   Hardware (SoC)                            │
│          Crystal, PLLs, Dividers, Muxes, Gates              │
└─────────────────────────────────────────────────────────────┘
```

---

### الـ Structs الأساسية وكيف ترتبط ببعض

#### أولاً: الـ `device_node` — قلب الـ Device Tree

```c
struct device_node {
    const char *name;          /* اسم الـ node مثلاً "pll" */
    phandle phandle;           /* رقم فريد للإشارة لهذا الـ node من node آخر */
    const char *full_name;     /* المسار الكامل مثلاً "/clocks/pll@10000" */
    struct fwnode_handle fwnode; /* للربط مع الـ firmware framework */

    struct property *properties; /* قائمة خصائص الـ node */
    struct device_node *parent;  /* الـ node الأب في الشجرة */
    struct device_node *child;   /* أول child */
    struct device_node *sibling; /* الـ node الأخ */
};
```

الـ `phandle` هو رقم مثل ID في قاعدة بيانات — عندما يقول uart0:
```dts
clocks = <&pll 2>;
```
الـ `&pll` يُحوَّل إلى `phandle` رقمي، والـ of_clk يستخدمه ليجد الـ node المقابل.

#### ثانياً: الـ `property` — بيانات الـ node

```c
struct property {
    char  *name;       /* اسم الخاصية مثلاً "clock-frequency" */
    int   length;      /* حجم البيانات بالبايت */
    void  *value;      /* البيانات الفعلية (big-endian) */
    struct property *next; /* linked list من الخصائص */
};
```

مثال: في الـ DT عندنا:
```dts
clock-frequency = <24000000>;
```
يتحوّل لـ `property` اسمها `"clock-frequency"` وقيمتها `24000000` مخزّنة بـ big-endian.

#### ثالثاً: الـ `of_device_id` — جدول المطابقة

```c
struct of_device_id {
    char name[32];        /* اسم الـ node */
    char type[32];        /* نوع الجهاز */
    char compatible[128]; /* النص في compatible property مثلاً "myvendor,pll" */
    const void *data;     /* بيانات خاصة بالـ driver (مثلاً دالة initialization) */
};
```

الـ `of_clk_init()` يأخذ جدول من هذه الـ structs، ويمشي على كل الـ clock nodes في الـ DT، ويطابق كل node مع driver عنده نفس الـ `compatible` string.

---

### كيف تعمل الـ Functions الثلاث في `of_clk.h`؟

#### `of_clk_init(const struct of_device_id *matches)`

هذه أهم دالة. يتم استدعاؤها مبكراً جداً أثناء الـ boot:

```
boot
 │
 ├─ start_kernel()
 │       │
 │       └─ time_init()
 │               │
 │               └─ of_clk_init(NULL)  ← هنا يتم تسجيل كل الـ clocks
 │
 ├─ rest_init()
 │       │
 │       └─ ... (باقي الـ boot)
```

**ما الذي تفعله بالتفصيل؟**

1. تمشي على شجرة الـ Device Tree وتبحث عن كل node فيه `#clock-cells`
2. لكل node تجد له مطابقة في الـ `matches` table
3. تستدعي دالة الـ initialization الخاصة بالـ driver
4. الـ driver يسجّل الـ clock في الـ CCF

```
of_clk_init()
      │
      ├── for_each_matching_node_and_match(matches)
      │       │
      │       ├── node: oscillator (compatible = "fixed-clock")
      │       │       └── يستدعي fixed_clk_setup()
      │       │               └── clk_hw_register_fixed_rate() ← تسجّل في CCF
      │       │
      │       ├── node: pll@10000 (compatible = "myvendor,pll")
      │       │       └── يستدعي myvendor_pll_setup()
      │       │               └── clk_hw_register() ← تسجّل في CCF
      │       │
      │       └── ... باقي الـ clock nodes
      │
      └── [تنتهي] كل الـ clocks مسجّلة وجاهزة للاستخدام
```

#### `of_clk_get_parent_count(const struct device_node *np)`

تحسب كم parent clock لهذا الـ node. تقرأ الـ property `clocks` من الـ DT وتعدّ العناصر.

مثال:
```dts
pll: pll@10000 {
    clocks = <&osc>, <&ext_clk>;  /* parents: oscillator + external clock */
};
```
ترجع `2`.

لماذا نحتاج هذه المعلومة؟ لأن بعض الـ clocks لها أكثر من مصدر (mux clock)، والـ driver يحتاج يعرف كم خيار عنده ليبني الـ clock tree بشكل صحيح.

#### `of_clk_get_parent_name(const struct device_node *np, int index)`

ترجع اسم الـ parent رقم `index`. الاسم هذا هو الـ string الذي يستخدمه الـ CCF داخلياً لتتبّع العلاقات.

مثال:
```c
// لـ node عنده clocks = <&osc>, <&ext_clk>
name0 = of_clk_get_parent_name(np, 0); // يرجع "osc" (اسم الـ oscillator)
name1 = of_clk_get_parent_name(np, 1); // يرجع "ext_clk"
```

كيف تعمل داخلياً؟
1. تقرأ الـ property `clocks` من الـ DT
2. تجد الـ phandle رقم `index`
3. تذهب للـ node المقابل لهذا الـ phandle
4. تقرأ منه الـ `clock-output-names` property لتجد الاسم الحقيقي

---

### الشرط المزدوج: `CONFIG_COMMON_CLK && CONFIG_OF`

```c
#if defined(CONFIG_COMMON_CLK) && defined(CONFIG_OF)
// الـ real implementations
unsigned int of_clk_get_parent_count(...);
const char *of_clk_get_parent_name(...);
void of_clk_init(...);

#else
// stub implementations لا تفعل شيئاً
static inline unsigned int of_clk_get_parent_count(...) { return 0; }
static inline const char *of_clk_get_parent_name(...) { return NULL; }
static inline void of_clk_init(...) {}
#endif
```

**لماذا هذا الشرط مهم؟**

- `CONFIG_COMMON_CLK`: يمكن تعطيله على أنظمة قديمة جداً تستخدم clock API مختلف
- `CONFIG_OF`: يمكن تعطيله على أنظمة لا تستخدم Device Tree أصلاً (مثل بعض الأجهزة القديمة التي تعتمد على ACPI أو hard-coded configuration)

عندما يكون أحد الشرطين غير موجود، يتم استبدال كل الدوال بـ stubs فارغة، وبهذا الـ code الذي يستدعي هذه الدوال لا يحتاج أي `#ifdef` في كل مكان — يستدعيها وكأنها موجودة، والـ compiler يحذف الـ calls الفارغة.

---

### رحلة الـ Clock من الـ DTS إلى الـ Driver

```
.dts file
┌─────────────────────────────────────┐
│  pll: pll@10000 {                   │
│    compatible = "myvendor,pll";     │
│    #clock-cells = <1>;              │
│    clocks = <&osc>;                 │  الـ Device Tree source
│    clock-output-names = "pll-out"; │
│  };                                 │
└─────────────────────────────────────┘
              │
              │ dtc compiler
              ▼
.dtb file (binary blob يُحمَّل بواسطة bootloader)
              │
              │ kernel boot
              ▼
of_flat_dt_unflatten_tree()   ← يحوّل الـ binary لـ device_node structs في الذاكرة
              │
              ▼
┌─────────────────────────────────────┐
│  struct device_node {               │
│    name = "pll"                     │
│    full_name = "/clocks/pll@10000"  │
│    properties →                     │
│      ├── "compatible" = "myvendor,pll" │
│      ├── "#clock-cells" = 1         │  الـ Device Tree في الذاكرة
│      ├── "clocks" = phandle(osc)    │
│      └── "clock-output-names" = "pll-out" │
│  }                                  │
└─────────────────────────────────────┘
              │
              │ of_clk_init()
              ▼
matches table:
  { .compatible = "myvendor,pll", .data = myvendor_pll_setup }
              │
              │ match found!
              ▼
myvendor_pll_setup(device_node)
  │
  ├── of_clk_get_parent_count(np)  → 1
  ├── of_clk_get_parent_name(np, 0) → "osc"
  └── clk_hw_register(NULL, &pll_hw, &pll_init_data)
            │
            │ pll_init_data.parent_names = ["osc"]
            ▼
┌─────────────────────────────────────┐
│      Common Clock Framework         │
│                                     │
│  clk_core {                         │
│    name = "pll-out"                 │
│    parent = clk_core("osc")         │  الـ Clock Tree في الذاكرة
│    rate = 1200000000                │
│    ops = &myvendor_pll_ops          │
│  }                                  │
└─────────────────────────────────────┘
              │
              │ clk_get(dev, "myclk")
              ▼
┌─────────────────────────────────────┐
│  UART Driver                        │
│  struct clk *uart_clk               │  Consumer
│  clk_enable(uart_clk)               │
│  clk_set_rate(uart_clk, 115200*16)  │
└─────────────────────────────────────┘
```

---

### العلاقة بين الـ Structs — رسم بياني شامل

```
struct of_device_id          struct device_node
┌──────────────────┐        ┌──────────────────────────┐
│ compatible[128]  │        │ name                     │
│ name[32]         │        │ phandle  ◄───────────────┼── يُستخدم لربط الـ nodes
│ type[32]         │        │ full_name                │
│ data ────────────┼──┐     │ fwnode                   │
└──────────────────┘  │     │ properties ──────────────┼──┐
                      │     │ parent ──────────────────┼──┼──► device_node (parent)
يُمرَّر لـ             │     │ child ───────────────────┼──┼──► device_node (child)
of_clk_init()         │     │ sibling ─────────────────┼──┼──► device_node (next)
                      │     └──────────────────────────┘  │
                      │                                    │
                      │                                    ▼
                      │     struct property (linked list)
                      │     ┌──────────────────────────┐
                      │     │ name = "compatible"      │
                      │     │ value = "myvendor,pll"   │
                      │     │ next ────────────────────┼──► property {
                      │     └──────────────────────────┘       name = "clocks"
                      │                                         value = phandle
                      │                                         next → ...
                      │                                       }
                      │
                      ▼
         init function pointer
         مثلاً: myvendor_pll_setup(node)
                      │
                      ▼
         struct clk_hw          struct clk_init_data
         ┌────────────┐        ┌──────────────────────┐
         │ core ──────┼──────► │ name                 │
         │ clk        │        │ ops ─────────────────┼──► struct clk_ops {
         └────────────┘        │ parent_names[]       │      .enable
                               │ num_parents          │      .disable
                               │ flags                │      .set_rate
                               └──────────────────────┘      .recalc_rate
                                                              ...
                                                           }
```

---

### ملاحظة معمارية مهمة — الفصل بين الـ Layers

تصميم الـ of_clk يعكس مبدأ أساسياً في الـ kernel:

| Layer | المسؤولية | لا يعرف عن |
|-------|-----------|------------|
| Consumer Driver | يطلب clock باسم | كيف يعمل الـ hardware |
| CCF | يدير شجرة الـ clocks | الـ Device Tree |
| of_clk | يربط الـ DT بالـ CCF | تفاصيل الـ hardware |
| Clock Provider Driver | يتحكم في الـ hardware | بقية الـ system |
| Device Tree | يصف الـ hardware | الـ software |

هذا الفصل يعني أنك لو اشتريت SoC جديدة:
- تكتب فقط driver صغير للـ PLL الجديد
- تكتب `.dts` تصف الـ clocks
- كل الـ consumer drivers (UART, SPI, USB...) تشتغل بدون تعديل
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### 0. الـ Config Options والـ Flags — Cheatsheet

| الـ Config / Flag | المعنى | لو مش موجود؟ |
|---|---|---|
| `CONFIG_COMMON_CLK` | تفعيل نظام الـ clock الموحد في Linux | الـ API بتتحول لـ stubs فارغة |
| `CONFIG_OF` | تفعيل دعم الـ Device Tree | الـ API بتتحول لـ stubs فارغة |
| `CLK_IS_CRITICAL` | الـ clock ده لازم يشتغل دايمًا، مش مسموح إيقافه | — |
| `OF_POPULATED` | الـ node اتعمل له init قبل كده، متعملوش تاني | — |

| الـ DT Property | معناها |
|---|---|
| `#clock-cells` | عدد الأرقام اللي بتحدد الـ clock جوه الـ specifier |
| `clocks` | قائمة الـ parent clocks اللي الـ node بيستهلكها |
| `clock-names` | أسماء مقابلة لـ `clocks` |
| `clock-output-names` | أسماء الـ clocks اللي الـ provider بيوفرها |
| `clock-indices` | mapping بين index رقمي واسم في `clock-output-names` |
| `clock-critical` | قائمة indices للـ clocks الحيوية اللي مش هتتوقف |
| `assigned-clocks` | clocks محتاجين تعديل تلقائي عند boot |
| `assigned-clock-parents` | الـ parent المطلوب لكل clock في `assigned-clocks` |
| `assigned-clock-rates` | التردد المطلوب (u32 Hz) |
| `assigned-clock-rates-u64` | التردد المطلوب (u64 Hz — للترددات العالية) |
| `clock-ranges` | يسمح للـ child بيورث الـ clocks من الـ parent node |

---

### 1. الـ Structs المهمة

#### `struct device_node`
**الموجود في:** `include/linux/of.h`

ده الـ node الأساسي في شجرة الـ Device Tree. كل جهاز أو controller في الـ DT بيتمثل بـ `device_node` واحد.

| الـ Field | النوع | الوظيفة |
|---|---|---|
| `name` | `const char *` | اسم الـ node |
| `phandle` | `phandle` (u32) | رقم تعريفي فريد يُستخدم للإشارة للـ node من مكان تاني |
| `full_name` | `const char *` | المسار الكامل زي `/soc/clocks@1000` |
| `fwnode` | `struct fwnode_handle` | abstraction layer فوق الـ OF node |
| `properties` | `struct property *` | linked list بكل الـ properties اللي الـ node عنده |
| `parent` | `struct device_node *` | الـ parent في شجرة الـ DT |
| `child` | `struct device_node *` | أول child |
| `sibling` | `struct device_node *` | الـ node الجار في نفس المستوى |
| `_flags` | `unsigned long` | فلاجات مثل `OF_POPULATED` |
| `data` | `void *` | بيانات خاصة بالمنصة |

**العلاقة بالـ of_clk:** كل function في `of_clk.h` بتاخد `const struct device_node *np` — ده هو المدخل الرئيسي لأي عملية استفسار عن clock.

---

#### `struct of_device_id`
**الموجود في:** `include/linux/mod_devicetable.h`

ده جدول بيربط كل `compatible` string في الـ DT بـ driver أو function معين.

| الـ Field | الوظيفة |
|---|---|
| `compatible` | الـ string زي `"myvendor,clk-controller"` |
| `data` | pointer للـ init function الخاصة بالـ driver ده |

**الاستخدام في `of_clk_init()`:** الـ kernel بيمشي على كل node في الـ DT، لو الـ `compatible` اتطابق مع حاجة في الجدول ده، بيستدعي الـ `data` (وهو الـ init function).

---

#### `struct of_clk_provider`
**الموجود في:** `drivers/clk/clk.c` (internal، مش public header)

ده أهم struct في المنظومة دي. بيمثل **مزود الـ clock** — يعني أي controller سجّل نفسه كمصدر للـ clocks.

```c
struct of_clk_provider {
    struct list_head link;      /* رابط في القائمة العالمية of_clk_providers */
    struct device_node *node;   /* الـ DT node الخاص بالـ provider */
    struct clk *(*get)(struct of_phandle_args *clkspec, void *data);    /* legacy */
    struct clk_hw *(*get_hw)(struct of_phandle_args *clkspec, void *data); /* modern */
    void *data;                 /* بيانات خاصة بالـ driver (مصفوفة الـ clocks مثلًا) */
};
```

| الـ Field | الوظيفة |
|---|---|
| `link` | بيربط كل الـ providers في `LIST_HEAD(of_clk_providers)` العالمية |
| `node` | مين اللي بيوفر — الـ DT node |
| `get` | callback قديم يرجع `struct clk *` |
| `get_hw` | callback حديث يرجع `struct clk_hw *` (المفضّل) |
| `data` | السياق — غالبًا `clk_hw_onecell_data` أو struct خاص بالـ driver |

---

#### `struct of_phandle_args`
**الموجود في:** `include/linux/of.h`

لما الـ consumer بيقول في الـ DT: `clocks = <&pll0 2>;` — الـ kernel بيحول السطر ده لـ `of_phandle_args`.

```c
struct of_phandle_args {
    struct device_node *np;     /* الـ node اللي بيوفر الـ clock (pll0) */
    int args_count;             /* عدد الأرقام الإضافية (1 هنا) */
    uint32_t args[MAX_PHANDLE_ARGS]; /* الأرقام نفسها (2 هنا) */
};
```

ده بالضبط الـ "تذكرة" اللي بتتبعت للـ provider عشان يرجع الـ clock المناسب.

---

#### `struct clock_provider`
**الموجود في:** `drivers/clk/clk.c` (داخلي لـ `of_clk_init`)

struct مؤقت بيُستخدم داخل `of_clk_init()` فقط لتنظيم عملية التهيئة.

```c
struct clock_provider {
    void (*clk_init_cb)(struct device_node *); /* الـ init function */
    struct device_node *np;                     /* الـ DT node */
    struct list_head node;                      /* رابط في قائمة مؤقتة */
};
```

الفكرة: بدل ما تهيّئ كل الـ providers على طول، الـ kernel بيبني قائمة أول، بعدين بيهيّئ بالترتيب بناءً على جاهزية الـ parents.

---

#### `struct clk_onecell_data` و `struct clk_hw_onecell_data`
**الموجود في:** `include/linux/clk-provider.h`

لما عندك controller بيوفر أكتر من clock واحد، بتحط كلهم في الـ struct ده وتبعته كـ `data` للـ provider.

```c
/* القديم */
struct clk_onecell_data {
    struct clk **clks;      /* مصفوفة pointers للـ clocks */
    unsigned int clk_num;   /* عدد الـ clocks */
};

/* الحديث — يستخدم clk_hw مباشرة */
struct clk_hw_onecell_data {
    unsigned int num;
    struct clk_hw *hws[] __counted_by(num); /* flexible array */
};
```

---

### 2. علاقات الـ Structs — ASCII Diagram

```
        Device Tree (DT)
        ================
        device_node [pll-controller@1000]
          ├── property: compatible = "vendor,pll"
          ├── property: #clock-cells = <1>
          ├── property: clock-output-names = "pll_out0", "pll_out1"
          └── fwnode ──────────────────────────────────────────┐
                                                               │
        device_node [uart@2000]  (consumer)                    │
          ├── property: clocks = <&pll-controller 1>           │
          └── property: clock-names = "baud"                   │
                │                                              │
                │ of_parse_phandle_with_args()                 │
                ▼                                              │
        of_phandle_args                                        │
          ├── np ──────────────────────────────► device_node [pll-controller@1000]
          ├── args_count = 1                                   │
          └── args[0] = 1                                      │
                │                                              │
                │ of_clk_get_hw_from_clkspec()                 │
                ▼                                              │
        LIST: of_clk_providers (global)                        │
          └── of_clk_provider                                  │
                ├── link (list_head)                           │
                ├── node ──────────────────────────────────────┘
                ├── get_hw() ──► of_clk_hw_onecell_get()
                └── data ──────► clk_hw_onecell_data
                                  ├── num = 2
                                  ├── hws[0] ──► clk_hw (pll_out0)
                                  └── hws[1] ──► clk_hw (pll_out1)  ◄── هنا يرجع index 1
```

---

### 3. دورة حياة الـ Clock Provider — Lifecycle Diagram

```
BOOT TIME
=========

  [1] REGISTRATION — وقت الـ boot
  ─────────────────────────────────────────────────────────────
  Driver/init function
       │
       ├─► clk_hw_register(dev, hw)      ← تسجيل كل clock مع الـ framework
       │
       └─► of_clk_add_hw_provider(np, get_fn, data)
                │
                ├── kzalloc(of_clk_provider)
                ├── cp->node = of_node_get(np)   ← يزود الـ refcount
                ├── cp->get_hw = get_fn
                ├── cp->data = data
                ├── mutex_lock(&of_clk_mutex)
                ├── list_add(&cp->link, &of_clk_providers)  ← يضاف للقائمة
                ├── mutex_unlock(&of_clk_mutex)
                ├── clk_core_reparent_orphans()  ← يحل الـ clocks اليتيمة
                └── of_clk_set_defaults(np, true) ← يطبق assigned-clock-*


  [2] USAGE — وقت الاستخدام (consumer يطلب clock)
  ─────────────────────────────────────────────────────────────
  Consumer driver
       │
       └─► clk_get(dev, "baud")
                │
                └─► of_clk_get_hw(np, index, con_id)
                         │
                         ├── of_parse_clkspec()         ← يقرأ "clocks" property
                         │        └── of_parse_phandle_with_args()
                         │
                         └── of_clk_get_hw_from_clkspec(clkspec)
                                  │
                                  ├── mutex_lock(&of_clk_mutex)
                                  ├── يمشي على of_clk_providers
                                  │     └── يقارن provider->node == clkspec->np
                                  ├── __of_clk_get_hw_from_provider()
                                  │     └── provider->get_hw(clkspec, data)
                                  │            └── of_clk_hw_onecell_get()
                                  │                  └── hw_data->hws[args[0]]
                                  └── mutex_unlock(&of_clk_mutex)
                                       └── يرجع clk_hw *


  [3] TEARDOWN — وقت إزالة الـ driver
  ─────────────────────────────────────────────────────────────
  Driver remove / devm cleanup
       │
       └─► of_clk_del_provider(np)
                │
                ├── mutex_lock(&of_clk_mutex)
                ├── يدور على of_clk_providers لو cp->node == np
                │     ├── list_del(&cp->link)          ← يشيله من القائمة
                │     ├── fwnode_dev_initialized(false) ← يحدّث الـ state
                │     ├── of_node_put(cp->node)         ← ينقص الـ refcount
                │     └── kfree(cp)
                └── mutex_unlock(&of_clk_mutex)
```

---

### 4. تدفق `of_clk_init()` — Dependency-Aware Init

الـ function دي زي مدير مشروع ذكي — مش بتهيّئ بشكل عشوائي، بتشوف مين جاهز الأول.

```
of_clk_init(matches)
     │
     ├─[Phase 1: بناء القائمة]────────────────────────────────────────────
     │   for_each_matching_node_and_match(DT, matches):
     │       if node is available:
     │           malloc(clock_provider)
     │           cp->clk_init_cb = match->data   ← الـ init function
     │           cp->np = node
     │           list_add_tail → clk_provider_list
     │
     └─[Phase 2: التهيئة بالترتيب]──────────────────────────────────────
         while clk_provider_list not empty:
             for each provider in list:
                 if force OR parent_ready(provider->np):
                 │
                 ├── parent_ready(np):
                 │     for each parent clock in np->clocks:
                 │         of_clk_get(np, i) → لو -EPROBE_DEFER فمش جاهز
                 │         لو مفيش -EPROBE_DEFER → كل الـ parents جاهزين
                 │
                 ├── of_node_set_flag(np, OF_POPULATED)  ← منع التكرار
                 ├── clk_init_cb(np)                      ← تهيئة الـ driver
                 ├── of_clk_set_defaults(np, true)        ← assigned-clocks
                 └── list_del + kfree

             if لم يتم تهيئة أي provider في هذه الجولة:
                 force = true   ← تهيئة قسرية للبقية (ربما parent اختياري)
```

**تشبيه:** تخيّل إنك بتطبخ وجبة — المكرونة مش هتتعمل قبل الماء يغلي. الـ `of_clk_init` بيحترم التسلسل ده تلقائيًا.

---

### 5. تدفق الـ API الرئيسية — Call Flow Diagrams

#### `of_clk_get_parent_count(np)`
```
consumer driver / clock driver
    │
    └─► of_clk_get_parent_count(np)
             │
             └─► of_count_phandle_with_args(np, "clocks", "#clock-cells")
                      │
                      └── يعدّ عدد الـ entries في الـ "clocks" property
                          ويرجع العدد (أو 0 لو في error)
```

**مثال:** لو الـ DT فيه `clocks = <&pll0 1>, <&osc0>;` الناتج هيكون `2`.

---

#### `of_clk_get_parent_name(np, index)`
```
clock driver (يريد اسم parent عشان يسجّله)
    │
    └─► of_clk_get_parent_name(np, 1)  ← أريد اسم الـ parent رقم 1
             │
             ├─► of_parse_phandle_with_args(np, "clocks", "#clock-cells", 1, &clkspec)
             │        └── يحصل على: clkspec.np = &osc_node, clkspec.args[0] = X
             │
             ├── index = clkspec.args[0]   ← رقم الـ clock جوه الـ provider
             │
             ├── [اختياري] of_property_for_each_u32(clkspec.np, "clock-indices", pv)
             │        └── لو موجود: يحوّل الـ index لـ offset في clock-output-names
             │
             ├─► of_property_read_string_index(clkspec.np, "clock-output-names", index)
             │        ├── لو نجح: يرجع الاسم مباشرة ✓
             │        └── لو فشل:
             │               ├── of_clk_get_from_provider(&clkspec)
             │               │     └── يجيب الـ clk المسجّل ويسأله عن اسمه
             │               └── لو #clock-cells = 0: يستخدم node name كـ fallback
             │
             └── of_node_put(clkspec.np)  ← ينقص الـ refcount
                 └── يرجع const char * (الاسم)
```

---

#### `of_clk_init(matches)` — Full System Init
```
arch/arm/kernel/setup.c (أو early_initcall)
    │
    └─► of_clk_init(NULL)   ← NULL يعني: استخدم __clk_of_table المدمج
             │
             ├── matches = &__clk_of_table  (مبني وقت compile عبر CLK_OF_DECLARE)
             │
             ├── [بناء القائمة] for_each_matching_node_and_match
             │       └── لكل node متاح: kzalloc(clock_provider) + list_add_tail
             │
             └── [حلقة التهيئة التدريجية]
                     while list not empty:
                         for each provider:
                             parent_ready()? ──► YES:
                                 clk_init_cb(np)   ← مثلًا: mypll_of_clk_init(np)
                                     └── داخل الـ driver:
                                         ├── clk_hw_register(...)
                                         └── of_clk_add_hw_provider(np, get_fn, data)
                                               └── يُضاف لـ of_clk_providers
                             parent_ready()? ──► NO: يتأجل للجولة القادمة
```

---

### 6. استراتيجية الـ Locking

الجزء ده بسيط ومحكم في نفس الوقت.

```
                    ┌─────────────────────────────────────┐
                    │        of_clk_mutex  (Mutex)         │
                    │  يحمي: LIST of_clk_providers         │
                    └─────────────────────────────────────┘
                         ▲                    ▲
                         │                   │
              ┌──────────┴──┐        ┌───────┴───────────┐
              │  Writers    │        │    Readers         │
              │─────────────│        │────────────────────│
              │ add_provider│        │ get_hw_from_clkspec│
              │ del_provider│        │ of_clk_get_*       │
              └─────────────┘        └────────────────────┘
```

| الـ Lock | النوع | يحمي |
|---|---|---|
| `of_clk_mutex` | `DEFINE_MUTEX` | القائمة العالمية `of_clk_providers` — الإضافة والحذف والبحث |

**قواعد الـ Locking:**

1. **لا يوجد nested locks** — الـ `of_clk_mutex` بيُؤخذ وحده فقط.
2. **لا تستدعي callbacks وأنت ماسك الـ mutex** — الـ `get_hw()` callback بيُستدعى داخل الـ lock، لكن يجب ألا تستدعي هي بدورها `add_provider` أو `del_provider` (وإلا deadlock).
3. **الـ `of_clk_set_defaults()`** بيُستدعى خارج الـ lock بعد `add_provider` — لأنه بيحتاج يجيب clocks وده قد يحتاج الـ mutex تاني.
4. **الـ `clk_core_reparent_orphans()`** كمان بيُستدعى بعد إضافة الـ provider — له lockنفسه الخاص داخل الـ clk core (`clk_prepare_lock`).

**تشبيه:** الـ `of_clk_mutex` زي بواب لغرفة فيها سجل الـ providers. أي حد عايز يضيف أو يشيل أو يدور، لازم يستأذن من البواب أولًا. لكن لما الـ driver بيشغّل الـ clock فعليًا (set_rate, set_parent) ده بيتم عبر نظام تاني (clk prepare lock) في الـ clk core.

---

### ملخص العلاقات الكاملة

```
CLK_OF_DECLARE(mydrv, "vendor,mydrv", mydrv_init)
        │
        │  [compile time] يضيف entry في __clk_of_table section
        │
        ▼
of_clk_init()  [boot time]
        │
        ├── يقرأ DT ويبحث عن compatible matches
        ├── يبني قائمة مؤقتة (clock_provider list)
        └── يهيّئ بالترتيب حسب جاهزية الـ parents
                │
                ▼
        mydrv_init(device_node *np)  [driver's init]
                │
                ├── يقرأ الـ DT properties
                ├── clk_hw_register() لكل clock
                └── of_clk_add_hw_provider(np, of_clk_hw_onecell_get, hw_data)
                        │
                        ▼
                of_clk_provider  [مُضاف لـ of_clk_providers list]
                        │
                        │  [عند الاستخدام]
                        ▼
                consumer: clk_get(dev, "name")
                        │
                        └── of_clk_get_hw_from_clkspec()
                                └── يبحث في القائمة ويستدعي get_hw callback
                                        └── يرجع clk_hw * للـ consumer
```
## Phase 4: شرح الـ Functions

### جدول ملخص الـ Functions

| Function | Type | Purpose |
|----------|------|---------|
| `of_clk_get_parent_count()` | `EXPORT_SYMBOL_GPL` | يعدّ عدد الـ parent clocks المرتبطة بـ device node معيّن |
| `of_clk_get_parent_name()` | `EXPORT_SYMBOL_GPL` | يجيب باسم الـ parent clock في موضع معيّن من قائمة الـ clocks |
| `of_clk_init()` | `__init` (global) | يمسح الـ Device Tree ويُشغّل كل clock providers بالترتيب الصحيح |

---

### الفئة الأولى: الاستعلام عن الـ Parent Clocks

هذه المجموعة تجيب على سؤال واحد: **"من هو أبو الـ clock ده؟"** — الـ Device Tree يخزّن العلاقات بين الـ clocks بشكل شجرة، وهذه الـ functions تقرأ تلك العلاقات.

---

#### `of_clk_get_parent_count()`

```c
unsigned int of_clk_get_parent_count(const struct device_node *np);
```

**الـ function دي بتعمل إيه؟**

تخيّل إن عندك جهاز في الـ Device Tree وفي الـ `.dts` فيه property اسمها `clocks = <&clk_a>, <&clk_b>, <&clk_c>` — الـ function دي بتعدّ عدد العناصر في القائمة دي. بتستخدم داخليًا `of_count_phandle_with_args()` مع property اسمها `"clocks"` وخلية المواصفات `"#clock-cells"`.

لو الـ property مش موجودة أو فيها خطأ، بترجع صفر.

**الـ Parameters:**

| Parameter | النوع | الشرح |
|-----------|-------|-------|
| `np` | `const struct device_node *` | الـ node في الـ Device Tree اللي عايز تعدّ clocks الـ parent بتاعته |

**القيمة المُرجَعة:**

- عدد صحيح موجب = عدد الـ parent clocks
- `0` = مفيش parent clocks أو الـ property غلط

**تفاصيل مهمة:**

- **الـ locking:** مفيش locking صريح داخلها — اللي جوّاها `of_count_phandle_with_args()` بتعمل حمايتها بنفسها.
- **الاستخدام الشائع:** الـ clock drivers بتستخدمها عشان تعرف هتحجز قد إيه ذاكرة لقائمة الـ parents قبل ما تعمل `of_clk_get_parent_name()` في loop.

**مين بيستدعيها؟**

غالبًا الـ clock drivers في مرحلة الـ probe أو الـ initialization، وبعض الـ core clock framework code.

---

#### `of_clk_get_parent_name()`

```c
const char *of_clk_get_parent_name(const struct device_node *np, int index);
```

**الـ function دي بتعمل إيه؟**

دي أكتر function تعقيدًا في الـ header. تخيّلها زي موظف في بنك المعلومات: بتديه رقم (الـ `index`) وبتقوله "أنا عايز اسم الـ parent clock رقم كذا"، وهو بيبحث بطريقة ذكية متعددة الخطوات.

**خطوات العمل الداخلي:**

```
1. of_parse_phandle_with_args(np, "clocks", "#clock-cells", index, &clkspec)
        ↓
   بيجيب الـ phandle والـ args الخاصة بالـ parent المطلوب

2. لو فيه "clock-indices" property في الـ parent node:
        ↓
   بيحوّل الـ hardware index ده لـ array offset حقيقي

3. of_property_read_string_index(clkspec.np, "clock-output-names", index)
        ↓
   بيحاول يقرأ الاسم من "clock-output-names" مباشرة

4. لو فشل الخطوة 3:
        ↓
   بيحاول يجيب الـ clock من الـ framework ويسأله عن اسمه
   لو فشل ده كمان، يرجع اسم الـ node نفسه (فقط لو #clock-cells = 0)
```

**ليه كل التعقيد ده؟**

لأن الـ Device Tree مش دايمًا متسق. بعض الـ nodes بتعرّف اسم الـ clock في `"clock-output-names"` مباشرة، وبعضها بيستخدم `"clock-indices"` لعمل mapping، وبعضها مش بيعرّف اسم خالص وبتتكل على الـ framework. الـ function دي بتتعامل مع الحالات الثلاثة.

**الـ Parameters:**

| Parameter | النوع | الشرح |
|-----------|-------|-------|
| `np` | `const struct device_node *` | الـ device node اللي عايز تعرف parent clock بتاعته |
| `index` | `int` | رقم الـ parent المطلوب (ابتداءً من 0) |

**القيمة المُرجَعة:**

- `const char *` = مؤشر لاسم الـ clock (مملوك للـ DT، متمسكش فيه بعد ما تخلص)
- `NULL` = مش لاقي الـ parent أو فيه مشكلة

**تفاصيل مهمة:**

- **الـ `of_node_put()`:** الـ function دي بتعمل `of_node_put(clkspec.np)` قبل ما ترجع — يعني هي بتدير الـ reference counting بنفسها، مش المتصل.
- **الـ string المُرجَعة:** مؤشر للـ DT data في الذاكرة — ثابتة ومش محتاج تعمل لها `free`، بس متعدّلش فيها.
- **لو الـ clock مش registered:** بيرجع اسم الـ DT node كـ fallback بس فقط لو الـ `#clock-cells = 0` (يعني مش محتاج args إضافية).

**مين بيستدعيها؟**

- الـ clock drivers لما بتبني قائمة الـ parent names
- الـ `of_clk_parent_fill()` اللي بتستدعيها في loop
- الـ clock framework نفسه في `clk_core_get_parents_names_nolock()` لما بيطبع معلومات الـ debugfs

---

### الفئة الثانية: تهيئة نظام الـ Clocks من الـ Device Tree

---

#### `of_clk_init()`

```c
void __init of_clk_init(const struct of_device_id *matches);
```

**الـ function دي بتعمل إيه؟**

دي أهم function في الـ header. مهمتها بسيطة في الفكرة لكن متعقدة في التنفيذ: **تمشي على كل الـ Device Tree وتشغّل كل clock providers بالترتيب الصحيح** — يعني الأب قبل الابن.

تخيّل إن عندك عمارة بيوتها متوقفة على الكهرباء من الدور الأرضي. لازم تشغّل الدور الأرضي الأول، بعدين الدور التاني، بعدين الدور التالت. الـ function دي بتعمل نفس الكلام مع الـ clock tree.

**الـ `__init` marker:** الـ function دي مُعلَّمة بـ `__init` يعني الـ kernel بيحذف الكود بتاعها من الذاكرة بعد ما الـ boot يخلص — عشان توفير ذاكرة.

**خوارزمية العمل الداخلي (مُبسَّطة):**

```
1. لو matches == NULL:
       استخدم &__clk_of_table (الجدول الافتراضي اللي الـ CLK_OF_DECLARE بيملاه)

2. امشي على كل node في الـ DT يطابق matches:
       لو available: ضيفه في clk_provider_list

3. while (clk_provider_list مش فاضية):
       امشي على كل provider في القائمة:
           لو parent_ready(provider->np) أو force == true:
               - ضع OF_POPULATED flag على الـ node
               - استدعي clk_init_cb(np)      // الـ driver بتاعه
               - استدعي of_clk_set_defaults() // طبّق الـ assigned-clocks
               - احذفه من القائمة
               - is_init_done = true

       لو دورة كاملة ومش اتشغّل أي حاجة:
           force = true  // اشغّل الباقي حتى لو الأب مش جاهز
```

**ليه الـ `force` mode؟**

في بعض الأحيان الـ parent clock مش ضروري (optional)، أو في dependency مش ممكن تحلّها بالترتيب. الـ `force = true` بيضمن إن كل الـ providers هتتهيّأ في النهاية حتى لو الترتيب مثالي.

**الـ Parameters:**

| Parameter | النوع | الشرح |
|-----------|-------|-------|
| `matches` | `const struct of_device_id *` | مصفوفة من الـ compatible strings مع الـ init functions المقابلة، أو `NULL` لاستخدام الجدول الافتراضي |

**القيمة المُرجَعة:**

`void` — مفيش قيمة مُرجَعة. لو فيه مشكلة (مثلاً فشل الـ `kzalloc`) بترجع بصمت بعد تنظيف الـ list.

**تفاصيل مهمة:**

- **الـ `OF_POPULATED` flag:** بيتضرب على كل node يتهيّأ عشان الـ platform driver subsystem ميعمل probe ليه تاني. ده يمنع الـ double initialization.
- **الـ `of_clk_set_defaults()`:** بعد ما كل provider يتهيّأ، بيطبّق الـ `assigned-clocks` و`assigned-clock-rates` من الـ DT تلقائيًا.
- **إدارة الذاكرة:** كل `clock_provider` struct بيتعمله `kzalloc` ثم `kfree` بعد الاستخدام. لو `kzalloc` فشلت، بيعمل cleanup للـ list كلها ويرجع.
- **الاستدعاء:** بيتستدعى مرة واحدة في الـ boot sequence، من `start_kernel()` عن طريق `time_init()` أو الـ architecture-specific init code.

**مين بيستدعيها؟**

الـ architecture-specific boot code أو الـ `start_kernel()` chain. مستدعاها مرة واحدة بس في حياة الـ kernel.

---

### ملاحظة مهمة: الـ Stub Functions

الـ header بيوفّر نسخ فاضية (stubs) من الـ functions الثلاثة في حالة إن الـ kernel اتبنى من غير `CONFIG_COMMON_CLK` أو من غير `CONFIG_OF`:

```c
/* لو CONFIG_COMMON_CLK أو CONFIG_OF مش مفعّلين */
static inline unsigned int of_clk_get_parent_count(const struct device_node *np)
{
    return 0;  /* مفيش أبوين */
}

static inline const char *of_clk_get_parent_name(const struct device_node *np,
                                                  int index)
{
    return NULL;  /* مفيش اسم */
}

static inline void of_clk_init(const struct of_device_id *matches) {}
/* مفيش حاجة تتعمل */
```

ده الـ pattern المعيارية في الـ Linux kernel للـ optional features — بدل ما تملأ الكود بـ `#ifdef` في كل مكان، بتعمل inline stubs بترجع قيم آمنة، والـ compiler بيشيل الـ dead code تلقائيًا.
## Phase 5: دليل الـ Debugging الشامل

الـ `of_clk.h` هو واجهة صغيرة لكن حيوية — هي جسر بين الـ **Device Tree** والـ **Common Clock Framework (CCF)**. لما يفشل الـ clock في الـ boot أو يطلع جهاز ما بـ `EPROBE_DEFER`، هنا تبدأ التحقيق. دعنا نفصّل كل أداة تحتاجها.

---

### Software Level

#### 1. مدخلات الـ debugfs المتعلقة بالـ of_clk

الـ **CCF** يعرض شجرة الـ clocks كاملة في `/sys/kernel/debug/clk/`. كل clock له مجلد خاص فيه.

```bash
# اعرض شجرة الـ clocks كاملة
cat /sys/kernel/debug/clk/clk_summary

# أو بشكل مرتب أكثر
cat /sys/kernel/debug/clk/clk_dump
```

**تفسير الـ output:**

```
                                 enable  prepare  protect                                duty
   clock                          cnt     cnt      cnt        rate   accuracy phase  cycle
----------------------------------------------------------------------------------------
 osc24M                             1       1        0    24000000          0     0  50000
  pll-cpu                           1       1        0   816000000          0     0  50000
   cpu-clk                          1       1        0   816000000          0     0  50000
```

- **enable cnt**: كم مرة اتفتح الـ clock — لو `0` والجهاز ما شتغل، المشكلة هنا.
- **prepare cnt**: مرحلة الإعداد قبل التفعيل.
- **rate**: التردد الحالي بالـ Hz — لو `0` ما اتضبط.

```bash
# اعرض clock معين بالاسم
ls /sys/kernel/debug/clk/
cat /sys/kernel/debug/clk/<clock-name>/clk_rate
cat /sys/kernel/debug/clk/<clock-name>/clk_enable_count
cat /sys/kernel/debug/clk/<clock-name>/clk_prepare_count
cat /sys/kernel/debug/clk/<clock-name>/clk_flags
```

**الـ flags المهمة:**

| Flag | المعنى |
|------|---------|
| `CLK_IS_ROOT` | مفيش أب |
| `CLK_IGNORE_UNUSED` | ما يتوقف حتى لو محدش يستخدمه |
| `CLK_IS_CRITICAL` | لازم يشتغل دايماً |

---

#### 2. مدخلات الـ sysfs

```bash
# اعرض معلومات الـ clock provider من الـ Device Tree
ls /sys/bus/platform/devices/
cat /sys/bus/platform/devices/<device-name>/of_node/clock-names 2>/dev/null
cat /sys/bus/platform/devices/<device-name>/of_node/clocks 2>/dev/null

# تحقق من الـ device node في الـ DT المحمّل
ls /sys/firmware/devicetree/base/
# مثال: ابحث عن clock-controller
find /sys/firmware/devicetree/base/ -name "compatible" -exec grep -l "clock" {} \;
```

---

#### 3. استخدام الـ ftrace لتتبع دوال الـ of_clk

الـ **ftrace** هو أقوى أداة لترى بالضبط إيه اللي بيحصل وقت الـ boot أو التشغيل.

```bash
# فعّل الـ tracing
mount -t tracefs tracefs /sys/kernel/tracing

# تتبع دوال الـ of_clk تحديداً
echo 'of_clk_*' > /sys/kernel/tracing/set_ftrace_filter
echo function > /sys/kernel/tracing/current_tracer
echo 1 > /sys/kernel/tracing/tracing_on

# شغّل الجهاز / أعد التشغيل / افعل ما تريد تتبعه
# ثم اقرأ النتيجة
cat /sys/kernel/tracing/trace

# تتبع events الـ clk الجاهزة
echo 1 > /sys/kernel/tracing/events/clk/enable
echo 1 > /sys/kernel/tracing/events/clk/clk_set_rate
echo 1 > /sys/kernel/tracing/events/clk/clk_enable
echo 1 > /sys/kernel/tracing/events/clk/clk_disable
echo 1 > /sys/kernel/tracing/events/clk/clk_prepare
echo 1 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace
```

**مثال output مفيد:**

```
kworker/0:1-45    [000] ....   1.234567: clk_enable: uart0_clk
kworker/0:1-45    [000] ....   1.234600: clk_set_rate: uart0_clk 48000000
```

لو شفت `clk_enable` بدون `clk_prepare` قبلها — مشكلة في الـ driver.

---

#### 4. الـ printk والـ dynamic debug

```bash
# فعّل dynamic debug لكل ملفات الـ of_clk و clk-core
echo 'file drivers/clk/clk.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/clk/of.c +p' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/clk/clk-conf.c +p' > /sys/kernel/debug/dynamic_debug/control

# أو فعّل كل الـ clk subsystem
echo 'module clk +p' > /sys/kernel/debug/dynamic_debug/control

# في وقت الـ boot من kernel cmdline
# أضف في /etc/default/grub أو bootloader:
# dyndbg="file drivers/clk/* +p"
```

---

#### 5. خيارات الـ Kernel Config للـ debugging

| Config | الوظيفة |
|--------|---------|
| `CONFIG_COMMON_CLK` | لازم مفعّل عشان of_clk يشتغل أصلاً |
| `CONFIG_OF` | لازم مفعّل للـ Device Tree support |
| `CONFIG_CLK_DEBUG` | يفعّل مجلدات الـ debugfs للـ clocks |
| `CONFIG_DEBUG_KERNEL` | أساسي لكل debugging |
| `CONFIG_DYNAMIC_DEBUG` | يفعّل الـ pr_debug() messages |
| `CONFIG_OF_DEBUG` | يطبع معلومات إضافية عن الـ DT parsing |
| `CONFIG_CLK_KUNIT_TEST` | unit tests للـ CCF |
| `CONFIG_PROVE_LOCKING` | يكتشف deadlocks في الـ clock locks |
| `CONFIG_DEBUG_SHIRQ` | يساعد في debug الـ shared IRQs المرتبطة بالـ clocks |

```bash
# تحقق من الـ config الحالي
zcat /proc/config.gz | grep -E 'CONFIG_(COMMON_CLK|OF|CLK_DEBUG|DYNAMIC_DEBUG)'
```

---

#### 6. أدوات الـ subsystem

```bash
# أداة clk_summary مباشرة
cat /sys/kernel/debug/clk/clk_summary | grep -E "rate|enable"

# استخدم devlink لو الـ platform يدعمه
devlink dev show

# أداة لقراءة الـ DT المحمّل
dtc -I fs /sys/firmware/devicetree/base/ -O dts 2>/dev/null | grep -A 10 "clock-controller"

# أداة fdtget للقراءة المباشرة من DTB
fdtget /boot/dtb-$(uname -r) /clocks compatible
fdtget /boot/dtb-$(uname -r) /clocks clock-frequency
```

---

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الـ Log | المعنى | الحل |
|--------------|--------|------|
| `of_clk_get_parent_name: index out of range` | الـ `index` المطلوب أكبر من عدد الـ `clocks` في الـ DT | تحقق من عدد الـ entries في `clocks` property في الـ DTS |
| `of_clk_init: no matches found` | ما في `of_device_id` يتطابق مع الـ DT nodes | تحقق من الـ `compatible` string في الـ DTS مقابل الـ driver |
| `failed to get clock: -EPROBE_DEFER` | الـ clock provider لسه ما اتسجّلش | الـ probe order خاطئ — الـ consumer يشتغل قبل الـ provider |
| `failed to get clock: -ENOENT` | اسم الـ clock غير موجود | تحقق من `clock-names` في DTS يتطابق مع الـ driver |
| `clk: Disabling unused clocks` | الـ kernel يوقف clocks غير مستخدمة | لو جهاز بيحتاج الـ clock، تأكد من `clk_prepare_enable()` |
| `WARNING: CPU: 0 PID: 1 at drivers/clk/clk.c:...` | خلل في الـ CCF state machine | في الغالب `clk_disable` بدون `clk_enable` مسبق |
| `Unable to parse clock-names` | خطأ في صياغة الـ DTS | افحص الـ DTS syntax بـ `dtc -I dtb -O dts` |
| `of_clk_get_parent_count: missing clocks property` | الـ node ما فيهوش `clocks` property | أضف `clocks` property للـ DT node |

---

#### 8. أماكن وضع الـ dump_stack و WARN_ON

لو بتكتب driver أو بتـ debug مشكلة، ضع الـ checks في هذه النقاط:

```c
/* في of_clk_init() — تحقق إن الـ matches مش NULL */
void of_clk_init(const struct of_device_id *matches)
{
    WARN_ON(!matches); /* لو NULL، مشكلة في الـ caller */
    /* ... */
}

/* في أي driver يستخدم of_clk_get_parent_name() */
const char *name = of_clk_get_parent_name(np, i);
if (!name) {
    /* هنا dump_stack() مفيد لمعرفة من استدعى هذا */
    dump_stack();
    dev_err(dev, "parent clock %d not found in DT\n", i);
    return -EINVAL;
}

/* تحقق من عدد الـ parents قبل الحلقة */
unsigned int count = of_clk_get_parent_count(np);
WARN_ON(count == 0); /* provider بدون parents؟ مشكلة */
```

---

### Hardware Level

#### 1. التحقق من أن الـ Hardware State يطابق الـ Kernel State

الفكرة: الـ kernel يعتقد إن الـ clock شغّال، لكن هل هو فعلاً شغّال على الـ silicon؟

```bash
# قارن rate الـ kernel مع القياس الفعلي
cat /sys/kernel/debug/clk/<clock-name>/clk_rate
# القيمة دي لازم تتطابق مع قراءة الـ oscilloscope على الـ pin

# تحقق من enable count
cat /sys/kernel/debug/clk/<clock-name>/clk_enable_count
# لو 0 والجهاز ما شتغلش، الـ clock ما اتفتحش
```

---

#### 2. تقنيات الـ Register Dump

معظم الـ SoCs عندها clock controller بعنوان محدد في الـ memory map.

```bash
# اقرأ register من الـ clock controller (مثال: عنوان وهمي)
# أولاً اعرف العنوان من DTS أو datasheet
devmem2 0x01C20000 w   # قرأ word من عنوان CCU على Allwinner A20

# أو استخدم /dev/mem مع dd
dd if=/dev/mem bs=4 count=1 skip=$((0x01C20000/4)) 2>/dev/null | xxd

# لو devmem2 مش موجود، ثبّته
apt-get install devmem2
# أو استخدم busybox
busybox devmem 0x01C20000

# اقرأ مجموعة registers (مثلاً 256 byte من الـ CCU)
devmem2 0x01C20000 b  # byte
devmem2 0x01C20050 w  # word (4 bytes) - عادةً register الـ PLL
```

**تفسير قيمة الـ register:**

```
PLL_CPU_CTRL_REG = 0x82001800
Bit 31 = 1 → PLL مفعّل (PLL_ENABLE)
Bit 28 = 0 → مش locked بعد؟ (PLL_LOCK)
Bits 8-15 = 0x18 = 24 → N factor
```

---

#### 3. نصائح الـ Logic Analyzer / Oscilloscope

```
الهدف: قياس الـ clock signal مباشرة على الـ PCB

1. ابحث في الـ schematic عن test points للـ clock lines:
   - MCLK، SCLK، CLK_OUT، REFCLK، OSC_OUT

2. على الـ oscilloscope:
   - ضع الـ probe على الـ test point
   - اختار: DC coupling لو الـ frequency عالية
   - اضبط timebase حسب التردد المتوقع:
     * 24 MHz OSC → 40ns/div
     * 1 GHz PLL → 1ns/div
   - قيس: Frequency, Duty Cycle, Rise Time, Voltage Level

3. على الـ Logic Analyzer:
   - فيد أكثر للـ clock gating والـ enable/disable
   - اضبط threshold voltage حسب الـ IO voltage (1.8V أو 3.3V)
   - ابحث عن glitches في وقت تغيير الـ rate

4. مقارنة النتيجة:
   - kernel يقول rate = 24000000 Hz
   - oscilloscope يقيس 24.001 MHz → طبيعي (دقة الـ oscillator)
   - oscilloscope يقيس 0 Hz → الـ clock مش شغّال رغم ما يقوله الـ kernel
```

---

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Log

| المشكلة الـ Hardware | ما تراه في الـ dmesg | الحل |
|--------------------|----------------------|------|
| الـ oscillator مش شغّال | `clk: Couldn't get clock, -ENODEV` أو الجهاز ما يبوت | افحص توصيل الـ crystal، تحقق من الـ power supply |
| الـ PLL ما اتـ lock | `pll timeout waiting for lock` | تحقق من الـ input frequency، قد يكون الـ N/M factors خاطئة |
| clock gating خاطئ | `device not responding` + `clk_enable_count = 0` | الـ gate register ما اتكتبش صح، افحص التوصيل |
| voltage غير مناسب للـ clock speed | `random crashes at high frequency` | الـ SoC يحتاج voltage أعلى للـ PLL speed المطلوبة |
| noise على الـ clock line | `bit errors` في الـ UART/SPI/I2C | أضف capacitor على الـ clock line، تحقق من الـ PCB layout |
| الـ clock source خاطئ | `wrong baud rate`، `incorrect timing` | الـ DTS يحدد clock source خاطئ، افحص الـ datasheet |

---

#### 5. الـ Device Tree Debugging

الـ **DTS** هو ما يقرأه `of_clk_get_parent_name()` و`of_clk_get_parent_count()` — لو الـ DTS خاطئ، كل شيء بعده خاطئ.

```bash
# اقرأ الـ DT المحمّل حالياً (كما يراه الـ kernel)
dtc -I fs /sys/firmware/devicetree/base/ -O dts -o /tmp/current.dts 2>/dev/null
cat /tmp/current.dts | grep -B 5 -A 20 "clock-controller"

# قارن مع الـ DTB على القرص
dtc -I dtb -O dts /boot/dtbs/$(uname -r)/your-board.dtb -o /tmp/disk.dts 2>/dev/null

# تحقق من clock properties لـ device معين
# مثال: uart0 يجب أن يكون له clocks property
grep -A 5 "uart0" /tmp/current.dts

# تحقق من of_clk_get_parent_count نتيجته عبر:
# عدد entries في clocks property
fdtget -l /boot/dtbs/your-board.dtb /clocks
fdtget /boot/dtbs/your-board.dtb /uart0 clock-names
fdtget /boot/dtbs/your-board.dtb /uart0 clocks

# تحقق إن الـ phandle مترابط صح
# في DTS:
# clocks = <&ccu CLK_UART0>;  ← &ccu هو phandle لـ clock controller
# الـ CLK_UART0 هو index داخل الـ provider
```

**مثال DTS صح للـ of_clk:**

```dts
/* الـ clock provider */
ccu: clock-controller@1c20000 {
    compatible = "allwinner,sun7i-a20-ccu";
    reg = <0x01c20000 0x400>;
    clocks = <&osc24M>, <&osc32k>;       /* parents */
    clock-names = "hosc", "losc";
    #clock-cells = <1>;                   /* مطلوب للـ of_clk */
    #reset-cells = <1>;
};

/* الـ clock consumer */
uart0: serial@1c28000 {
    compatible = "snps,dw-apb-uart";
    reg = <0x01c28000 0x400>;
    clocks = <&ccu CLK_BUS_UART0>, <&ccu CLK_UART0>;  /* clock IDs */
    clock-names = "bus", "baudclk";      /* of_clk_get_parent_name يقرأ هذا */
};
```

---

### Practical Commands

#### مجموعة أوامر جاهزة للنسخ

```bash
#!/bin/bash
# === of_clk Debug Script ===

echo "=== Clock Tree Summary ==="
cat /sys/kernel/debug/clk/clk_summary 2>/dev/null || echo "debugfs not mounted"

echo ""
echo "=== Mounting debugfs if needed ==="
mountpoint -q /sys/kernel/debug || mount -t debugfs debugfs /sys/kernel/debug
mountpoint -q /sys/kernel/tracing || mount -t tracefs tracefs /sys/kernel/tracing

echo ""
echo "=== All Clock Names ==="
ls /sys/kernel/debug/clk/ 2>/dev/null

echo ""
echo "=== Clocks with enable_count > 0 (active clocks) ==="
for clk in /sys/kernel/debug/clk/*/; do
    count=$(cat "$clk/clk_enable_count" 2>/dev/null)
    if [ "$count" -gt 0 ] 2>/dev/null; then
        rate=$(cat "$clk/clk_rate" 2>/dev/null)
        echo "$(basename $clk): enabled=$count, rate=${rate}Hz"
    fi
done

echo ""
echo "=== Device Tree Clock Info ==="
dtc -I fs /sys/firmware/devicetree/base/ -O dts 2>/dev/null | \
    grep -E "(clock-controller|clock-names|#clock-cells|compatible)" | \
    head -30
```

```bash
# فعّل ftrace للـ clk events
echo 0 > /sys/kernel/tracing/tracing_on
echo > /sys/kernel/tracing/trace
echo 1 > /sys/kernel/tracing/events/clk/enable
echo 1 > /sys/kernel/tracing/events/clk/clk_enable
echo 1 > /sys/kernel/tracing/events/clk/clk_disable
echo 1 > /sys/kernel/tracing/events/clk/clk_set_rate
echo 1 > /sys/kernel/tracing/events/clk/clk_prepare
echo 1 > /sys/kernel/tracing/tracing_on

# افعل العملية اللي تريد تتبعها (مثلاً load driver)
modprobe your_driver

echo 0 > /sys/kernel/tracing/tracing_on
cat /sys/kernel/tracing/trace | grep -E "(clk_enable|clk_set_rate)"
```

```bash
# تحقق من clock معين بالتفصيل
CLK_NAME="uart0_clk"   # غيّر هذا للـ clock اللي تبحث عنه

echo "=== Debug: $CLK_NAME ==="
echo -n "Rate: "; cat /sys/kernel/debug/clk/$CLK_NAME/clk_rate
echo -n "Enable count: "; cat /sys/kernel/debug/clk/$CLK_NAME/clk_enable_count
echo -n "Prepare count: "; cat /sys/kernel/debug/clk/$CLK_NAME/clk_prepare_count
echo -n "Flags: "; cat /sys/kernel/debug/clk/$CLK_NAME/clk_flags
echo -n "Phase: "; cat /sys/kernel/debug/clk/$CLK_NAME/clk_phase
echo -n "Accuracy: "; cat /sys/kernel/debug/clk/$CLK_NAME/clk_accuracy
```

```bash
# dynamic debug لمتابعة of_clk في الـ kernel log
echo 'file drivers/clk/clk.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
echo 'file drivers/clk/of.c +pflmt' > /sys/kernel/debug/dynamic_debug/control
# ثم راقب الـ log
dmesg -w | grep -i "clk\|clock\|of_clk"
```

```bash
# تحقق من الـ DT nodes التي لها clock-controller
find /sys/firmware/devicetree/base/ -name "compatible" | while read f; do
    if grep -q "clock" "$f" 2>/dev/null; then
        dir=$(dirname "$f")
        echo "=== $(basename $dir) ==="
        cat "$f" 2>/dev/null | tr '\0' ' '
        echo ""
        # اعرض clock-cells لو موجود
        [ -f "$dir/#clock-cells" ] && echo "  #clock-cells: $(cat $dir/'#clock-cells' | xxd)"
    fi
done
```

#### مثال output وكيف تفسّره

**output من `clk_summary`:**

```
                                 enable  prepare  protect                                duty
   clock                          cnt     cnt      cnt        rate   accuracy phase  cycle
----------------------------------------------------------------------------------------
 osc24M                             2       2        0    24000000          0     0  50000
  pll-cpu                           1       1        0   816000000          0     0  50000
   cpu-clk                          1       1        0   816000000          0     0  50000
  pll-periph                        3       3        0   600000000          0     0  50000
   ahb1-clk                         3       3        0   150000000          0     0  50000
    uart0-clk                       1       1        0    24000000          0     0  50000
    uart0-clk-gate                  0       0        0    24000000          0     0  50000
```

**التفسير:**
- الـ `uart0-clk-gate` عنده `enable_count = 0` — يعني الـ driver ما فتحشي الـ gate.
- الـ `uart0-clk` نفسه شغّال بـ 24 MHz — الـ source صح.
- المشكلة في الـ gate — الـ driver لازم يستدعي `clk_prepare_enable(uart0_gate_clk)`.

**output من dmesg عند مشكلة of_clk:**

```
[    2.345678] uart@1c28000: Failed to get clock 'baudclk': -2
[    2.345700] uart@1c28000: probe failed: -2
```

- الـ `-2` هو `ENOENT` — الـ clock `baudclk` غير موجود في الـ DT.
- الحل: أضف `clock-names = "bus", "baudclk"` و`clocks = <&ccu CLK_BUS_UART0>, <&ccu CLK_UART0>` في الـ DTS.
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو الأول: بورد RK3562 — UART ما بيشتغل بعد boot

#### العنوان
Industrial gateway على RK3562 — الـ UART صامت تماماً بعد تشغيل الجهاز

#### السياق
شركة تبني industrial gateway للتحكم في مصانع. الـ SoC هو **RK3562** من Rockchip. الـ gateway لازم يتكلم مع أجهزة Modbus عن طريق UART. المنتج اشتغل تمام على kernel قديم، لكن بعد upgrade لـ kernel جديد مع تفعيل `CONFIG_OF` — الـ UART وقف تماماً.

#### المشكلة
الـ UART driver بيتحمل، لكن الـ clock بتاعته `uart_clk` بيرجع `NULL` من `clk_get()`. النتيجة؟ الـ driver مش قادر يضبط الـ baud rate، فالـ UART صامت.

#### التحليل
الـ problem جذورها في `of_clk.h`. لما الـ kernel بيعمل boot، بيشتغل **`of_clk_init()`**:

```c
// من of_clk.h — الدالة دي بتشغّل كل clock providers في DT
void of_clk_init(const struct of_device_id *matches);
```

الدالة دي بتمشي على كل nodes في الـ Device Tree اللي عندها `compatible` موجود في قائمة `matches`، وبتسجّل كل clock provider. لو الـ clock provider بتاع UART مش اتسجّل صح، الـ `clk_get()` هيرجع `NULL`.

السبب الفعلي: الـ DT بتاع البورد الجديد فيه node للـ CRU (Clock Reset Unit) لكن الـ `compatible` string اتغيرت بين الـ kernel versions:

```
// DT قديم (شغال)
cru: cru@ff000000 {
    compatible = "rockchip,rk3562-cru";
    ...
};

// DT جديد (مكسور) — اسم خاطئ
cru: cru@ff000000 {
    compatible = "rockchip,rk3562-cru-v2"; /* ← هذا غلط */
    ...
};
```

لما `of_clk_init()` بتشوف الـ `matches` table، مش بتلاقي match للـ `"rockchip,rk3562-cru-v2"`، فالـ CRU مش بيتسجّلش كـ clock provider، وبالتالي `of_clk_get_parent_name()` مش بيرجع أي parent clock للـ UART.

#### الحل

**خطوة 1:** تشخيص المشكلة عن طريق kernel logs:

```bash
dmesg | grep -i "clk\|clock\|cru"
# هتشوف: "clk: Couldn't get clock 'uart_clk'"
```

**خطوة 2:** فحص الـ DT المُحمَّل فعلاً:

```bash
cat /sys/firmware/devicetree/base/cru@ff000000/compatible
# هيطبع: rockchip,rk3562-cru-v2
```

**خطوة 3:** فحص الـ driver بيدعم إيه:

```bash
grep -r "rk3562-cru" /sys/bus/platform/drivers/
# أو في kernel source:
grep "rk3562" drivers/clk/rockchip/clk-rk3562.c
```

**خطوة 4:** تصحيح الـ DTS:

```c
/* تصحيح في arch/arm64/boot/dts/rockchip/rk3562.dtsi */
cru: cru@ff000000 {
    compatible = "rockchip,rk3562-cru"; /* ← الاسم الصحيح */
    reg = <0x0 0xff000000 0x0 0x1000>;
    rockchip,grf = <&grf>;
    #clock-cells = <1>;
    #reset-cells = <1>;
};
```

#### الدرس المستفاد
**`of_clk_init()`** بتعتمد كلياً على الـ `compatible` strings. لو الـ DT node اسمه غلط بمقدار حرف واحد، الـ clock provider كله مش بيتسجّل، وكل الـ peripherals اللي بتعتمد عليه بتصحى بدون clock. دايماً verify الـ `compatible` strings من الـ driver source مباشرة، مش من documentation قديمة.

---

### السيناريو الثاني: STM32MP1 — I2C بيـ probe لكن بيفشل بعدين

#### العنوان
IoT sensor node على STM32MP1 — الـ I2C driver بيـ probe ناجح لكن الـ transfers بتفشل

#### السياق
شركة بتبني IoT sensor nodes صغيرة تعمل على **STM32MP1** من ST. الـ node بتقيس temperature وpressure عن طريق I2C. البورد custom-designed، والـ kernel compiled مع `CONFIG_COMMON_CLK=y` و`CONFIG_OF=y`.

#### المشكلة
الـ I2C device بيـ probe بنجاح (يعني الـ driver اشتغل)، لكن أي محاولة قراءة أو كتابة على الـ I2C bus بتطلع error:

```
i2c i2c-1: Transfer failed: -ETIMEDOUT
```

#### التحليل
الـ engineer بدأ يفحص الـ clock setup. الـ I2C driver بيستخدم:

```c
// الـ I2C driver بيطلب parent clock من DT
parent_count = of_clk_get_parent_count(np);
// ثم بياخد اسم الـ parent
parent_name = of_clk_get_parent_name(np, 0);
```

الـ `of_clk_get_parent_count()` بترجع عدد الـ clocks في `clocks` property في الـ DT node. الـ `of_clk_get_parent_name()` بترجع الـ `clock-names` property بالـ index المطلوب.

الـ DT بتاع الـ I2C على هذا الشكل:

```
i2c1: i2c@40012000 {
    compatible = "st,stm32mp15-i2c";
    reg = <0x40012000 0x400>;
    clocks = <&rcc I2C1_K>; /* ← clock واحدة */
    clock-names = "i2c_ker_ck";
    ...
};
```

الـ `of_clk_get_parent_count()` رجعت `1` — صح. الـ `of_clk_get_parent_name(np, 0)` رجعت `"i2c_ker_ck"` — صح.

المشكلة الفعلية: الـ clock نفسها موجودة لكن الـ RCC (Reset and Clock Controller) مش بيديها الـ frequency الصح. الـ I2C core clock المتوقع 100MHz، لكن الـ RCC مضبوطة على 24MHz فقط (خطأ في DT).

```bash
# فحص الـ clock frequency الفعلية
cat /sys/kernel/debug/clk/i2c_ker_ck/clk_rate
# النتيجة: 24000000  ← 24MHz بدل 100MHz
```

الـ I2C hardware بيحسب الـ timing registers بناءً على الـ expected clock، فلما الـ clock مختلفة، الـ timing كله خاطئ والـ transfers بتـ timeout.

#### الحل

**خطوة 1:** تتبع سلسلة الـ clocks:

```bash
# شوف الـ clock tree كامل
cat /sys/kernel/debug/clk/clk_summary | grep -A5 "i2c"
```

**خطوة 2:** فحص الـ PLL configuration في DT:

```c
/* المشكلة في arch/arm64/boot/dts/st/stm32mp151.dtsi */
/* الـ PLL4 مضبوطة غلط */
pll4: st,pll@3 {
    cfg = <1 49 0 6 7 PQR(1,0,0)>; /* ← حساب خاطئ */
};
```

**خطوة 3:** تصحيح الـ PLL ليدي 100MHz لـ I2C:

```c
/* التصحيح الصح */
pll4: st,pll@3 {
    cfg = <3 99 5 5 4 PQR(1,1,1)>; /* يدي 200MHz، نقسمه /2 = 100MHz */
};
```

#### الدرس المستفاد
**`of_clk_get_parent_name()`** بترجعلك اسم الـ clock من الـ DT، لكنها **مش بتتحقق** من الـ frequency. ممكن الـ clock موجودة وشغالة بـ frequency غلط تماماً. دايماً استخدم `/sys/kernel/debug/clk/clk_summary` للتحقق من الـ actual rate بعد الـ boot.

---

### السيناريو الثالث: i.MX8 — Android TV box مش بيعرض على HDMI

#### العنوان
Android TV box على i.MX8MQ — الشاشة ما بتشتغلش بعد تغيير الـ BSP

#### السياق
شركة صينية بتصنع **Android TV box** بـ **i.MX8MQ** من NXP. البورد بتوصل بالتلفزيون عن طريق HDMI. المنتج اشتغل على Android 10 مع kernel 4.19. بعد upgrade لـ Android 12 مع kernel 5.15 — الـ HDMI مات تماماً.

#### المشكلة
الـ HDMI port مش بيشتغل. الـ dmesg بيقول:

```
imx8mq-disp-hdmi: Failed to get clock 'hdmi_ref_266m'
```

#### التحليل
الـ HDMI controller على i.MX8MQ بيحتاج عدة clocks بالترتيب الصحيح. الـ driver بيستخدم:

```c
/* الـ HDMI driver بيعدّ الـ clocks في الـ DT node */
count = of_clk_get_parent_count(np);
for (i = 0; i < count; i++) {
    name = of_clk_get_parent_name(np, i);
    /* يسجّل كل clock باسمها */
}
```

الـ engineer فتح الـ DTS القديم والجديد وقارن:

```
/* DT قديم — kernel 4.19 — شغّال */
hdmi: hdmi@32c00000 {
    clocks = <&clk IMX8MQ_CLK_HDMI_APB>,
             <&clk IMX8MQ_CLK_HDMI_REF_266M>,
             <&clk IMX8MQ_CLK_HDMI_PHY>;
    clock-names = "hdmi_apb",
                  "hdmi_ref_266m",
                  "hdmi_phy";
};

/* DT جديد — kernel 5.15 — مكسور */
hdmi: hdmi@32c00000 {
    clocks = <&clk IMX8MQ_CLK_HDMI_APB>,
             <&clk IMX8MQ_CLK_HDMI_PHY>;  /* ← الـ REF_266M اتحذف بالغلط! */
    clock-names = "hdmi_apb",
                  "hdmi_phy";
};
```

لما الـ driver طلب `of_clk_get_parent_name(np, 1)` — رجعت `"hdmi_phy"` بدل `"hdmi_ref_266m"`. الـ `of_clk_get_parent_count()` رجعت `2` بدل `3`. الـ driver حاول يبحث عن `"hdmi_ref_266m"` فما لقيهاش.

الـ `of_clk_get_parent_name()` بتشتغل كالآتي: بتفتش في الـ `clock-names` property بالـ index المطلوب. لو الـ index موجود، بترجع الاسم. لو مش موجود (لأن الـ list أقصر)، بترجع `NULL`.

#### الحل

**خطوة 1:** مقارنة سريعة:

```bash
# على البورد
cat /sys/firmware/devicetree/base/hdmi@32c00000/clock-names | tr '\0' '\n'
# النتيجة: hdmi_apb  hdmi_phy  ← ناقص clock
```

**خطوة 2:** مقارنة مع الـ driver المتوقع:

```bash
grep "clock-names\|clk_get" drivers/gpu/drm/imx/hdmi/imx8mq-hdmi.c
```

**خطوة 3:** إضافة الـ clock الناقصة في DTS:

```c
/* arch/arm64/boot/dts/freescale/imx8mq.dtsi */
hdmi: hdmi@32c00000 {
    clocks = <&clk IMX8MQ_CLK_HDMI_APB>,
             <&clk IMX8MQ_CLK_HDMI_REF_266M>, /* ← أُعيد */
             <&clk IMX8MQ_CLK_HDMI_PHY>;
    clock-names = "hdmi_apb",
                  "hdmi_ref_266m",             /* ← أُعيد */
                  "hdmi_phy";
};
```

#### الدرس المستفاد
**`of_clk_get_parent_count()`** بتعدّ الـ clock entries في DT، والـ `of_clk_get_parent_name()` بتجيب الاسم بالـ index. لو واحدة من الـ clocks اتحذفت من الـ DT — الـ driver مش هيعرف، هيقول فقط "ما لقيتش الـ clock دي". في الـ board bring-up والـ BSP upgrades: دايماً قارن الـ `clock-names` في الـ DT مع الـ `clk_get(dev, "name")` calls في الـ driver.

---

### السيناريو الرابع: AM62x — Custom board bring-up، SPI لا يستجيب

#### العنوان
Custom industrial board على AM62x — الـ SPI controller عنده probe ناجح لكن الـ bus صامت

#### السياق
شركة embedded بتعمل custom board بـ **AM62x** من Texas Instruments لـ automotive ECU. الـ ECU بيتكلم مع external ADC عن طريق SPI. فريق الـ BSP بدأ الـ bring-up من صفر من DTSI الرسمي لـ AM62x.

#### المشكلة
الـ SPI driver بيـ probe بنجاح، لكن oscilloscope على الـ SPI CLK pin يقرأ صفر — مفيش إشارة خالص.

#### التحليل
الـ engineer اشتبه في الـ clocks لأن الـ SPI protocol يعتمد كلياً على الـ clock. بدأ يفحص:

```bash
# خطوة 1: فحص هل of_clk_init اشتغلت صح
dmesg | grep -E "clk:|of_clk"
# نتيجة مريبة: مفيش رسالة "Initialized X clocks" من TI CLK driver
```

الـ `of_clk_init()` في kernel initialization path بتُستدعى هكذا:

```c
/* في init/main.c أو arch setup */
of_clk_init(NULL); /* NULL يعني استخدم built-in table */
```

لما بيتمرر `NULL`، الـ kernel بيستخدم `__clk_of_table` — وهي قائمة تتكوّن من كل الـ clock drivers اللي اتسجّلوا بـ `CLK_OF_DECLARE`. المشكلة: الـ clock driver بتاع AM62x (`ti-sci-clk`) مش مـ compile في الـ kernel!

```bash
# فحص الـ kernel config
grep "TI_SCI_CLK\|TI_CLK" /proc/config.gz | zcat
# النتيجة: CONFIG_TI_SCI_CLK is not set  ← هنا المشكلة!
```

يعني `of_clk_init()` اتنادت صح، لكن الـ clock provider نفسه مش موجود في الـ kernel لأن الـ Kconfig مش مفعّلة. النتيجة؟ الـ SPI clock request فشل بصمت، والـ SPI controller شتغل على الـ default (صفر أو undefined rate).

#### الحل

**خطوة 1:** تأكيد الـ Kconfig المطلوبة:

```bash
# لـ AM62x، لازم:
grep -r "am62\|ti-sci" drivers/clk/Kconfig
# هتلاقي: config TI_SCI_CLK
#   depends on TI_SCI_PROTOCOL
```

**خطوة 2:** تفعيل الـ config في `.config`:

```
CONFIG_TI_SCI_PROTOCOL=y
CONFIG_TI_SCI_CLK=y
CONFIG_COMMON_CLK_TI_SCI=y
```

**خطوة 3:** إعادة compile وتحقق:

```bash
make menuconfig  # فعّل TI SCI Clock driver
make -j8 Image dtbs
# بعد boot:
dmesg | grep "TI SCI"
# هتشوف: "ti_sci_clk: Probing clocks for AM62x"
```

**خطوة 4:** تأكيد نهائي:

```bash
cat /sys/kernel/debug/clk/clk_summary | grep "spi\|mcspi"
# هتشوف الـ SPI clock بـ rate صحيحة مثلاً 48000000
```

#### الدرس المستفاد
**`of_clk_init()`** بتسجّل بس الـ clock providers اللي موجودة فعلاً في الـ kernel. لو الـ driver مش مـ compile (Kconfig مش مفعّلة)، الـ `of_clk_init()` هتمشي وما تعملش حاجة — ومفيش error واضح. في الـ custom board bring-up: دايماً ابدأ بـ `clk_summary` في debugfs وتأكد ان كل الـ SoC clocks موجودة قبل ما تبدأ تفحص الـ peripheral drivers.

---

### السيناريو الخامس: Allwinner H616 — USB مش بيشوف devices

#### العنوان
Android TV box على Allwinner H616 — الـ USB ports مش بتشوف أي device

#### السياق
منتج Android TV box رخيص الثمن بـ **Allwinner H616**. الـ box بيستخدم **USB 3.0** للـ external storage و**USB 2.0** للـ keyboard/mouse. المنتج كان شغال على BSP قديم من Allwinner. بعد تبني mainline kernel — الـ USB مات.

#### المشكلة
الـ `lsusb` مش بيطلع أي devices، حتى لو connected. الـ USB host controller الـ probe ناجح، لكن مفيش activity.

#### التحليل
الـ USB controller على H616 بيحتاج **USB PHY clock** وهو `usb_phy_clk`. الـ engineer فتح الـ DT:

```
/* DT بتاع USB في allwinner H616 */
usb_host0_ehci: usb@5101000 {
    compatible = "generic-ehci";
    reg = <0x05101000 0x100>;
    clocks = <&ccu CLK_BUS_OHCI0>,
             <&ccu CLK_USB_PHY0>;
    clock-names = "bus", "phy";
    phys = <&usbphy 0>;
    ...
};
```

ثم فحص عدد الـ parent clocks:

```bash
# في shell على الجهاز
python3 -c "
import os
path = '/sys/firmware/devicetree/base/usb@5101000/clock-names'
with open(path, 'rb') as f:
    data = f.read()
names = data.split(b'\x00')
print(f'Clock count: {len([n for n in names if n])}')
print(f'Names: {[n.decode() for n in names if n]}')
"
# النتيجة:
# Clock count: 2
# Names: ['bus', 'phy']
```

يبدو صح — عندنا clock اسمها `phy`. لكن:

```bash
cat /sys/kernel/debug/clk/clk_summary | grep "usb\|phy"
# النتيجة: CLK_USB_PHY0 مش موجودة في الـ summary!
```

الـ problem: الـ mainline kernel الـ CCU driver للـ H616 ناقص الـ `CLK_USB_PHY0` definition! الـ `of_clk_get_parent_name(np, 1)` بترجع `"phy"` كاسم، لكن لما الـ driver يحاول يعمل `clk_get(dev, "phy")`، بيلاقي إن الـ clock provider مسجّلها لكن الـ clock نفسها غير معرّفة في الـ H616 CCU driver.

```bash
# فحص الـ CCU driver المتاح
grep "USB_PHY\|usb_phy" drivers/clk/sunxi-ng/ccu-h616.c
# نتيجة ممكنة: غياب تعريف CLK_USB_PHY0
```

الـ flow كالآتي:
1. `of_clk_init()` ← تسجّل الـ H616 CCU كـ clock provider
2. الـ USB driver ← يطلب `of_clk_get_parent_name(np, 1)` ← يرجع `"phy"`
3. الـ USB driver ← يطلب `clk_get(dev, "phy")` ← يبحث في الـ CCU
4. الـ CCU ← ما عنده `CLK_USB_PHY0` معرّف ← يرجع `ERR_PTR(-ENOENT)`
5. الـ USB controller ← يشتغل بدون PHY clock ← PHY ميت ← USB ميت

#### الحل

**خطوة 1:** تأكيد المشكلة بـ strace أو ftrace:

```bash
# تفعيل clock tracing
echo 1 > /sys/kernel/debug/tracing/events/clk/enable
dmesg | grep "clk.*phy\|usb.*clk"
```

**خطوة 2:** إضافة الـ missing clock في CCU driver:

```c
/* drivers/clk/sunxi-ng/ccu-h616.c */
/* إضافة definition للـ USB PHY clocks */
static SUNXI_CCU_GATE(clk_usb_phy0, "usb-phy0", "osc24M",
                      0x0cc, BIT(8), 0);

static SUNXI_CCU_GATE(clk_usb_phy1, "usb-phy1", "osc24M",
                      0x0cc, BIT(9), 0);
```

**خطوة 3:** تسجيلهم في الـ clk_table:

```c
/* في قائمة h616_clks */
[CLK_USB_PHY0] = &clk_usb_phy0.common.hw,
[CLK_USB_PHY1] = &clk_usb_phy1.common.hw,
```

**خطوة 4:** إضافة الـ ID في الـ header:

```c
/* include/dt-bindings/clock/sun50i-h616-ccu.h */
#define CLK_USB_PHY0    100
#define CLK_USB_PHY1    101
```

**خطوة 5:** بعد البناء والـ boot:

```bash
cat /sys/kernel/debug/clk/clk_summary | grep usb
# هتشوف الـ USB PHY clocks بـ rate = 24000000
lsusb
# الـ USB devices ظهرت!
```

#### الدرس المستفاد
الـ `of_clk_get_parent_name()` و`of_clk_get_parent_count()` بيشتغلوا على الـ **DT text** فقط — بيقرأوا الأسماء من `clock-names` property. لكن الأسماء دي مجرد strings، لازم يكون في clock provider بيعرف ويقدم الـ clock دي فعلاً. في mainline kernel bring-up لـ SoCs جديدة زي **Allwinner H616**: كثير من الـ clocks موجودة في الـ DT لكن ناقصة في الـ CCU driver. دايماً قارن بين `clock-names` في الـ DT والـ clocks المعرّفة فعلاً في الـ CCU/CRU driver بتاع الـ SoC.
## Phase 7: مصادر ومراجع

---

### مقالات LWN.net

**الـ LWN.net** هو المرجع الأول لأي حاجة تتعلق بـ Linux kernel internals. الروابط دي حقيقية وجاية من نتائج البحث:

| العنوان | الوصف |
|---|---|
| [A common clock framework](https://lwn.net/Articles/472998/) | المقالة الأصلية اللي شرحت تصميم الـ Common Clock Framework لأول مرة |
| [common clk framework](https://lwn.net/Articles/486841/) | نقاش تقني عميق حول آلية عمل الـ framework وتفاصيل الـ struct clk |
| [Documentation/clk.txt](https://lwn.net/Articles/489668/) | ملف التوثيق الرسمي للـ clock framework بشكل مباشر |
| [Add a generic struct clk](https://lwn.net/Articles/460193/) | الـ patch الأولي اللي اقترح توحيد الـ struct clk عبر كل الـ platforms |
| [clk: dt: bindings for mux, divider & gate clocks](https://lwn.net/Articles/555242/) | توثيق الـ Device Tree bindings للـ clock types الأساسية (mux, divider, gate) |
| [Add device tree and clock drivers for SM8250 SoC](https://lwn.net/Articles/809919/) | مثال حي على كيفية دمج الـ of_clk مع clock driver لـ SoC حقيقي |
| [Basic clock and device tree support for StarFive JH7110](https://lwn.net/Articles/923946/) | مثال أحدث يوضح of_clk_init وof_clk_get_parent_name في سياق RISC-V |

---

### التوثيق الرسمي للـ Kernel

**الـ kernel documentation** هو الحقيقة الوحيدة — كل حاجة تانية ممكن تتقادم:

#### الملفات الرسمية في kernel.org

```
Documentation/driver-api/clk.rst       ← الدليل الشامل للـ Common Clock Framework
Documentation/devicetree/bindings/clock/ ← كل الـ DT bindings للساعات
include/linux/of_clk.h                  ← الملف اللي بندرسه
include/linux/clk-provider.h           ← الـ API الخاص بـ clock providers
drivers/clk/clk.c                       ← التنفيذ الفعلي للـ framework
drivers/clk/clk-of.c                   ← منطق الـ of_clk_* functions
```

#### الروابط الإلكترونية للتوثيق

- [The Common Clk Framework — kernel docs](https://docs.kernel.org/driver-api/clk.html) — الصفحة الرسمية الكاملة
- [Common Clock Framework (kernel.org raw)](https://www.kernel.org/doc/Documentation/clk.txt) — النسخة النصية القديمة لا تزال مفيدة

---

### Kernel Commits المهمة

**الـ commits** دي هي اللي شكّلت الـ `of_clk.h` كما هو موجود دلوقتي:

- **[torvalds/linux — drivers/clk/clk.c](https://github.com/torvalds/linux/blob/master/drivers/clk/clk.c)**
  تاريخ التعديلات على الملف الرئيسي للـ framework.

- **[torvalds/linux — include/linux/clk.h](https://github.com/torvalds/linux/blob/master/include/linux/clk.h)**
  الـ header الرئيسي للـ clock API.

- **[linux-commits-search (Typesense)](https://linux-commits-search.typesense.org/)**
  ابحث عن `of_clk_init` أو `of_clk_get_parent_name` هنا لتتبع تاريخ التعديلات.

- **[LKML: PATCH v7 — introduce the common clock framework](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00026.html)**
  الـ patch الأصلي سنة 2012 اللي أدخل الـ framework — التاريخ الحقيقي للموضوع.

---

### نقاشات Mailing List

- **[LKML v7 patch — Common Clock Framework (2012)](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00026.html)**
  النقاش الأصلي بين Mike Turquette والـ maintainers لما تم اقتراح الـ framework.
  اقرأه عشان تفهم ليه التصميم اتخذ الشكل ده.

---

### مصادر إضافية

#### Bootlin / Free Electrons

- **[Common Clock Framework: How to Use It — ELCE 2013 (PDF)](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf)**
  عرض تقديمي من مؤتمر Embedded Linux Conference Europe — عملي جداً وبيشرح of_clk_init خطوة بخطوة.

#### kernelnewbies.org

- **[Linux_6.7 — kernelnewbies](https://kernelnewbies.org/Linux_6.7)**
  ملخص التغييرات في الـ Common CLK Framework في kernel 6.7.
- **[Linux_6.12 — kernelnewbies](https://kernelnewbies.org/Linux_6.12)**
  آخر التحديثات المتعلقة بالـ clock subsystem.

#### elinux.org

- **[Timing API Specification — elinux.org](https://elinux.org/Timing_API_Specification)**
  متطلبات الـ clock accessibility أثناء boot وتفاصيل الـ timestamp APIs في الـ embedded Linux.
- **[High Resolution Timers — elinux.org](https://elinux.org/High_Resolution_Timers)**
  علاقة الـ clock sources بالـ high resolution timers — سياق مفيد لفهم أهمية الـ clock framework.
- **[Boot-up Time Definition Of Terms — elinux.org](https://elinux.org/Boot-up_Time_Definition_Of_Terms)**
  مصطلحات مهمة تخص توقيت الـ boot وعلاقته بـ of_clk_init.

---

### الكتب الموصى بها

#### Linux Device Drivers 3rd Edition (LDD3)

**الكتاب المجاني المتاح online:**

- **الفصل ده مباشر:** Chapter 14 — The Linux Device Model
  بيشرح كيف الـ Device Tree وبنية الـ device nodes بتتربط بالـ drivers.
- **رابط مجاني:** [https://lwn.net/Kernel/LDD3/](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love (3rd Edition)

- **الفصل المفيد:** Chapter 17 — Devices and Modules
  بيشرح كيف الـ subsystems بتشتغل وعلاقتها بالـ bus/device model.
- **الفصل المكمّل:** Chapter 14 — The Block I/O Layer
  سياق لفهم إزاي الـ kernel بيدير الـ resources المشتركة.

#### Embedded Linux Primer — Christopher Hallinan (2nd Edition)

- **الفصل المباشر:** Chapter 15 — Embedded Linux BSP (Board Support Package)
  هنا بيتكلم عن الـ clock initialization في سياق embedded boards — of_clk_init هو اللي بيشغّل الـ BSP clocks.
- **الفصل المكمّل:** Chapter 8 — Device Tree
  شرح عملي للـ Device Tree وإزاي الـ clock-names وclock-cells بتتعرّف.

#### Professional Linux Kernel Architecture — Wolfgang Mauerer

- مفيد لفهم الـ kernel internal structures المرتبطة بالـ clock framework.

---

### مصطلحات البحث (Search Terms)

لو عايز تعرف أكتر، دول أفضل الـ search terms:

```
# عام
linux kernel common clock framework
of_clk_init device tree clock provider
of_clk_get_parent_name clock-names binding
CONFIG_COMMON_CLK CONFIG_OF

# للـ Device Tree bindings
devicetree clock-cells clock-names linux
of_clk_add_provider clock provider registration

# للـ internals
clk_of_match drivers/clk/clk-of.c
of_clk_hw_simple_get clock lookup
clk_ops enable disable rate set

# للـ debugging
clk_summary debugfs /sys/kernel/debug/clk
clk_dump linux clock tree

# للـ مقارنة platforms
ARM clock controller DT binding
Qualcomm gcc-sm8250 of_clk
Raspberry Pi BCM2835 clock driver
```

---

### ملخص سريع للمصادر حسب الهدف

| الهدف | المصدر |
|---|---|
| فهم التصميم من الأساس | [LWN: A common clock framework](https://lwn.net/Articles/472998/) |
| API reference كامل | [kernel docs: clk.html](https://docs.kernel.org/driver-api/clk.html) |
| مثال عملي للـ DT bindings | [ELCE 2013 PDF](https://bootlin.com/pub/conferences/2013/elce/common-clock-framework-how-to-use-it/common-clock-framework-how-to-use-it.pdf) |
| تاريخ الـ patches | [LKML v7 patch](https://lkml.iu.edu/hypermail/linux/kernel/1203.2/00026.html) |
| embedded context | Embedded Linux Primer Ch. 15 |
| آخر التحديثات | [kernelnewbies Linux_6.12](https://kernelnewbies.org/Linux_6.12) |
## Phase 8: Writing simple module

### الفكرة: نراقب `of_clk_get_parent_name`

الملف `of_clk.h` يُصدِّر ثلاث functions رئيسية تتعلق بـ **Device Tree clock management**. أكثرها إثارةً للمراقبة هي `of_clk_get_parent_name` — لأنها تُستدعى كلما أراد أي driver معرفة اسم الـ parent clock من الـ Device Tree. فكرها زي بندر يسأل: "مين أبو هذه الساعة؟" وهي ترجع الاسم.

سنستخدم **kprobe** لنلتصق بهذه الدالة ونطبع اسم الـ node والـ index المطلوب في كل استدعاء.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * of_clk_parent_probe.c
 *
 * kprobe على of_clk_get_parent_name
 * يطبع اسم الـ device_node والـ index في كل استدعاء
 */

/* -------- الـ includes -------- */
#include <linux/module.h>       /* module_init / module_exit / MODULE_* */
#include <linux/kprobes.h>      /* struct kprobe, register_kprobe, ... */
#include <linux/kernel.h>       /* pr_info */
#include <linux/of.h>           /* struct device_node */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Arabic Kernel Student");
MODULE_DESCRIPTION("kprobe على of_clk_get_parent_name لمراقبة طلبات الـ parent clock");

/* -------- الـ pre-handler (يُنفَّذ قبل الدالة الأصلية) -------- */
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * أول argument للدالة = struct device_node *np
     * ثاني argument       = int index
     *
     * على x86-64: الـ arguments تجي في:
     *   regs->di  → أول arg (np)
     *   regs->si  → ثاني arg (index)
     *
     * على ARM64: الـ arguments تجي في:
     *   regs->regs[0] → أول arg (np)
     *   regs->regs[1] → ثاني arg (index)
     */
#ifdef CONFIG_X86_64
    struct device_node *np = (struct device_node *)regs->di;
    int index              = (int)regs->si;
#elif defined(CONFIG_ARM64)
    struct device_node *np = (struct device_node *)regs->regs[0];
    int index              = (int)regs->regs[1];
#else
    /* معمارية ثانية — نتجاهل */
    return 0;
#endif

    /* نتحقق إن الـ pointer مش NULL قبل ما نقرأ منه */
    if (np && np->full_name)
        pr_info("[of_clk_probe] of_clk_get_parent_name called: node='%s', index=%d\n",
                np->full_name, index);
    else
        pr_info("[of_clk_probe] of_clk_get_parent_name called: node=NULL, index=%d\n",
                index);

    return 0; /* 0 = تابع التنفيذ الطبيعي */
}

/* -------- تعريف الـ kprobe -------- */
static struct kprobe kp = {
    .symbol_name = "of_clk_get_parent_name", /* اسم الدالة اللي نريد نراقبها */
    .pre_handler = handler_pre,              /* الـ callback قبل التنفيذ */
};

/* -------- module_init -------- */
static int __init of_clk_probe_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[of_clk_probe] فشل register_kprobe: %d\n", ret);
        return ret;
    }

    pr_info("[of_clk_probe] kprobe مسجّل على العنوان: %px\n", kp.addr);
    return 0;
}

/* -------- module_exit -------- */
static void __exit of_clk_probe_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[of_clk_probe] kprobe أُزيل، الوداع!\n");
}

module_init(of_clk_probe_init);
module_exit(of_clk_probe_exit);
```

---

### شرح كل قسم

#### الـ `#include`s

| Header | سبب وجوده |
|---|---|
| `<linux/module.h>` | أساس كل module — بدونه ما تشتغل الـ `module_init` ولا الـ `MODULE_*` |
| `<linux/kprobes.h>` | يجيب `struct kprobe` وكل API الـ kprobe |
| `<linux/kernel.h>` | يجيب `pr_info` و `pr_err` |
| `<linux/of.h>` | يعرِّف `struct device_node` حتى نقدر نقرأ `full_name` منه |

---

#### الـ `handler_pre` — قلب الموضوع

الـ **pre-handler** هو الكود اللي يشتغل مباشرةً قبل ما الـ kernel ينفِّذ `of_clk_get_parent_name`. تخيَّله زي حارس يقف عند باب المطبخ ويسجِّل كل طلب قبل ما يوصل للطباخ.

**لماذا `regs->di` و `regs->si`؟**
على x86-64، الـ calling convention يحط أول argument في register الـ `RDI` وثاني argument في `RSI`. الـ `struct pt_regs` يحفظ حالة الـ CPU عند نقطة الـ probe، فنقرأ منه مباشرة.

**لماذا التحقق من NULL؟**
في عالم الـ kernel، أي pointer ممكن يكون NULL أو invalid — التحقق يحمينا من الـ kernel panic.

---

#### `struct kprobe kp`

```c
static struct kprobe kp = {
    .symbol_name = "of_clk_get_parent_name",
    .pre_handler = handler_pre,
};
```

الـ `symbol_name` يخلي الـ kernel يبحث عن عنوان الدالة في الـ **kallsyms table** تلقائياً. ما نحتاج نكتب عنواناً يدوياً — الـ kernel يحله بنفسه وقت `register_kprobe`.

---

#### `module_init` — التسجيل

`register_kprobe` تضع **breakpoint** (INT3 على x86 أو BRK على ARM64) عند أول instruction من الدالة. كلما استدعاها أي كود في الـ kernel، الـ CPU يقف، يستدعي الـ `pre_handler`، ثم يرجع للتنفيذ الطبيعي. إن فشل التسجيل (مثلاً الدالة مش exported أو محمية) نرجع الـ error ولا نكمل.

---

#### `module_exit` — إلغاء التسجيل

`unregister_kprobe` تُزيل الـ breakpoint وتُعيد الـ instruction الأصلية. هذا ضروري جداً: لو تركنا الـ kprobe بدون إزالة وأُزيل الـ module، الـ CPU سيصطدم بـ breakpoint يشير على كود غير موجود — وهذا kernel panic مضمون.

---

### كيف تبنيه وتشغّله

```bash
# Makefile بسيط
# ضع هذا في ملف Makefile بجانب of_clk_parent_probe.c

obj-m += of_clk_parent_probe.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

```bash
# بناء وتحميل
make
sudo insmod of_clk_parent_probe.ko

# مشاهدة الـ output
sudo dmesg -w | grep of_clk_probe

# إزالة الـ module
sudo rmmod of_clk_parent_probe
```

---

### مثال على الـ output المتوقع

```
[of_clk_probe] kprobe مسجّل على العنوان: ffffffffc0a12340
[of_clk_probe] of_clk_get_parent_name called: node='/soc/clock-controller@12345678', index=0
[of_clk_probe] of_clk_get_parent_name called: node='/soc/clock-controller@12345678', index=1
[of_clk_probe] of_clk_get_parent_name called: node='/clocks/pll0', index=0
[of_clk_probe] kprobe أُزيل، الوداع!
```

كل سطر يقول: "فلان driver يسأل عن اسم الـ parent clock رقم N للـ node الفلاني." هكذا تقدر ترى أي device يجلب معلومات الـ clock من الـ Device Tree في الوقت الفعلي.
