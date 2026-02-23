## Phase 1: الصورة الكبيرة ببساطة

### ما هو هذا الملف؟

الـ `include/linux/of_gpio.h` هو **header** صغير جداً — لا يتجاوز 38 سطراً — لكنه يمثل جسراً بين عالمَين داخل الـ Linux kernel:

- عالم **Device Tree (OF = Open Firmware)**: الطريقة التي يصف بها الـ hardware نفسه للـ kernel عبر ملفات `.dts`.
- عالم **GPIO API**: الطريقة التي يتحكم بها الـ kernel في أطراف الـ GPIO (أطراف إدخال/إخراج رقمية عامة).

---

### القصة — لماذا هذا الجسر ضروري؟

تخيل أنك تبني جهازاً مدمجاً (embedded board) مثل Raspberry Pi. على هذا الجهاز عشرات أطراف الـ **GPIO** — بعضها يتحكم في LED، وبعضها يقرأ حالة زر ضغط.

الـ **bootloader** أو firmware يصف الـ hardware في ملف يسمى **Device Tree Blob (DTB)**:

```
// مثال من ملف .dts
leds {
    compatible = "gpio-leds";
    led-status {
        gpios = <&gpio1 17 GPIO_ACTIVE_LOW>;  /* المتحكم gpio1، الطرف رقم 17 */
    };
};
```

هنا `&gpio1` هو **phandle** — مؤشر إلى عقدة في شجرة الـ Device Tree. الرقم `17` هو رقم الطرف داخل ذلك المتحكم.

الآن السؤال: كيف يقرأ **driver** الـ LED هذه المعلومات ويحصل على descriptor يستطيع به التحكم الفعلي في الطرف؟

الجواب: عبر `of_get_named_gpio()` — الدالة الوحيدة التي يُصدّرها هذا الـ header.

---

### ما الذي يفعله `of_get_named_gpio()`؟

```c
/*
 * np        : عقدة الجهاز في Device Tree (مثل عقدة leds)
 * list_name : اسم الخاصية في الـ DTS (مثل "gpios" أو "enable-gpio")
 * index     : الرقم التسلسلي إذا كانت الخاصية تحتوي عدة GPIOs
 *
 * الإرجاع: رقم GPIO عالمي (global GPIO number) أو قيمة سالبة عند الخطأ
 */
extern int of_get_named_gpio(const struct device_node *np,
                             const char *list_name, int index);
```

الدالة تمشي في شجرة الـ Device Tree، تقرأ الـ phandle، تحوّله إلى رقم GPIO حقيقي يمكن استخدامه مع الـ GPIO API التقليدي.

---

### البنية الكاملة للنظام

```
Device Tree (.dts)
      |
      | (تحليل وقت الإقلاع)
      v
struct device_node   <-- تعريفه في include/linux/of.h
      |
      | of_get_named_gpio()
      v
رقم GPIO عالمي (integer)
      |
      | gpio_request() / gpio_direction_output() ...
      v
GPIO Hardware (pin على الـ SoC)
```

---

### الحالتان: مع أو بدون CONFIG_OF_GPIO

الـ header يستخدم **compile-time guard** لدعم الأنظمة التي لا تحتوي على Device Tree:

| الحالة | السلوك |
|---|---|
| `CONFIG_OF_GPIO=y` | الدالة الحقيقية تُربط من `drivers/gpio/gpiolib-of.c` |
| `CONFIG_OF_GPIO=n` | stub inline تُرجع `-ENOSYS` حتى لا يفشل الـ build |

هذا نمط شائع في الـ kernel: **graceful degradation** — الـ driver لا يعتمد بشكل صارم على الـ GPIO، فيُسمح له بالربط حتى على منصات بدون Device Tree.

---

### ملاحظة مهمة: هذا الـ API قديم (Legacy)

السطر 15 في الملف نفسه يقول:

```c
#include <linux/gpio.h>  /* FIXME: Shouldn't be here */
```

الـ `linux/gpio.h` نفسه يقول في أول سطوره:
> "This is the LEGACY GPIO bulk include file ... should not be included in new code."

الـ API الحديث هو `gpiod_get()` من `<linux/gpio/consumer.h>` — لكن `of_get_named_gpio()` لا تزال موجودة لدعم الـ drivers القديمة.

---

### الملفات ذات الصلة التي يجب معرفتها

| الملف | الدور |
|---|---|
| `include/linux/of_gpio.h` | **الـ header محل الدراسة** — الواجهة العامة |
| `drivers/gpio/gpiolib-of.c` | التنفيذ الفعلي لـ `of_get_named_gpio()` وكل منطق ربط OF بالـ GPIO |
| `drivers/gpio/gpiolib-of.h` | header داخلي لـ gpiolib-of.c |
| `drivers/gpio/gpiolib.c` | النواة الأساسية لنظام GPIO في الـ kernel |
| `include/linux/of.h` | تعريف `struct device_node` وكل دوال قراءة الـ Device Tree |
| `include/linux/gpio/driver.h` | واجهة كتابة **driver** لمتحكم GPIO (`struct gpio_chip`) |
| `include/linux/gpio/consumer.h` | الـ API الحديث للمستهلكين (بديل `of_get_named_gpio`) |
| `include/linux/gpio.h` | الـ API القديم (legacy) — مشمول هنا بالخطأ |

---

### الملفات المكوّنة للـ Subsystem (GPIO + OF)

**Core:**
- `drivers/gpio/gpiolib.c` — القلب النابض لنظام GPIO
- `drivers/gpio/gpiolib-of.c` — ربط Device Tree بالـ GPIO

**Headers:**
- `include/linux/of_gpio.h` — الـ header محل الدراسة
- `include/linux/gpio/driver.h` — واجهة الـ driver
- `include/linux/gpio/consumer.h` — واجهة المستهلك الحديثة
- `include/linux/of.h` — Device Tree types

**Hardware Drivers (أمثلة):**
- `drivers/gpio/gpio-mpc8xxx.c` — PowerPC
- `drivers/gpio/gpio-pxa.c` — Intel PXA
- `drivers/gpio/gpio-aspeed.c` — ASPEED BMC
- `drivers/gpio/gpio-dwapb.c` — DesignWare APB GPIO
## Phase 2: شرح الـ OF-GPIO Framework

---

### المشكلة — لماذا يوجد هذا الـ Subsystem؟

في الأنظمة المضمّنة المبنية على ARM، يحتوي الـ SoC على عشرات أو مئات من أطراف الـ GPIO موزّعة على عدة controllers. كل driver يحتاج GPIO يواجه سؤالاً أساسياً:

> **كيف يعرف الـ driver أي GPIO بالضبط يجب أن يستخدم؟**

قبل وجود Device Tree، كان الحل هو **hardcoding** أرقام الـ GPIO مباشرةً في كود الـ driver أو في بيانات الـ board file:

```c
/* الطريقة القديمة — hardcoded GPIO number في board file */
static struct foo_platform_data bar_pdata = {
    .reset_gpio = 42,   /* رقم عشوائي مرتبط بـ board معينة */
    .power_gpio = 43,
};
```

هذا النهج يكسر قاعدة أساسية في kernel: **driver يجب أن يصف السلوك، لا الهاردوير المحدد.** نفس الـ driver قد يعمل على boards مختلفة حيث نفس الوظيفة (reset, power, IRQ) متصلة بـ GPIO مختلف تماماً.

---

### الحل — ماذا يفعل الـ OF-GPIO Framework؟

**الـ OF** = **Open Firmware** — وهو المعيار الذي نشأ منه الـ Device Tree.

الـ OF-GPIO framework يوفر **طبقة ترجمة** بين:
- وصف الـ GPIO في الـ Device Tree (بيانات هاردوير، مستقلة عن الـ driver)
- الـ GPIO API العامة في الكيرنل (`gpiod_get`, `gpio_request`, ...إلخ)

بمعنى آخر: الـ framework يُجيب على السؤال — **"عند رؤية `reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>` في الـ DTS، كيف أحوّل هذا إلى gpio descriptor حقيقي يمكن استخدامه؟"**

الدالة المحورية التي يصدّرها هذا الهيدر هي:

```c
int of_get_named_gpio(const struct device_node *np,
                      const char *list_name, int index);
```

هذه الدالة:
1. تبحث عن property باسم `list_name` في الـ `device_node`
2. تحلّل الـ phandle والـ specifier داخل تلك الـ property
3. تترجمها إلى رقم GPIO عالمي (global GPIO number) داخل الـ kernel

---

### تشبيه حقيقي — نظام العنونة البريدية

تخيّل أن لديك شركة توصيل تعمل في مدن مختلفة. كل مدينة لها نظام ترقيم خاص بها للشوارع.

| عنصر التشبيه | المقابل في الكيرنل |
|---|---|
| رقم الشارع المحلي داخل المدينة | offset داخل `gpio_chip` (0..ngpio-1) |
| اسم المدينة | اسم الـ `gpio_chip` / `device_node` |
| العنوان الكامل = مدينة + شارع | Global GPIO number في الكيرنل |
| دليل العناوين البريدي | الـ DTS property (`reset-gpios = <&gpio1 5 ...>`) |
| موظف مكتب البريد الذي يترجم | `of_get_named_gpio()` + `of_xlate()` |
| الشركة المركزية التي تفهم كل المدن | GPIO subsystem (gpiolib) |

**التفاصيل:**
- الـ `<&gpio1>` في الـ DTS يشير إلى `device_node` خاص — هذا هو "اسم المدينة"
- الرقم `5` هو الـ offset داخل ذلك الـ chip — هذا هو "رقم الشارع المحلي"
- `GPIO_ACTIVE_LOW` هي flags تصف كيفية استخدام الطرف — هذا مثل "الطابق الثالث"
- `of_get_named_gpio()` تقرأ "العنوان" وتعيد رقماً عالمياً يفهمه كل الكيرنل

---

### المعمارية الكاملة — أين يقع OF-GPIO في الكيرنل؟

```
┌─────────────────────────────────────────────────────────┐
│                   Driver Layer                          │
│   (i2c-mux, regulators, leds, keys, spi-cs, ...etc)    │
│                                                         │
│   of_get_named_gpio(np, "reset-gpios", 0)               │
│   gpiod_get(dev, "reset", GPIOD_OUT_LOW)                │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│              OF-GPIO Bridge Layer                       │
│         (include/linux/of_gpio.h)                       │
│                                                         │
│  of_get_named_gpio()                                    │
│    └─► of_get_named_gpio_flags()                        │
│          └─► of_parse_phandle_with_args()   ← OF layer  │
│                └─► gc->of_xlate()           ← per-chip  │
└─────────────────────┬───────────────────────────────────┘
                      │
          ┌───────────┴────────────┐
          ▼                        ▼
┌──────────────────┐   ┌──────────────────────────────┐
│   OF / DT Layer  │   │       GPIO Subsystem          │
│  (linux/of.h)    │   │       (gpiolib)               │
│                  │   │                               │
│  device_node     │   │  gpio_chip  gpio_desc         │
│  property        │   │  gpio_device                  │
│  of_phandle_args │   │  gpiochip_add_data()          │
└──────────────────┘   └──────────────┬───────────────┘
                                       │
                    ┌──────────────────┴──────────────┐
                    ▼                                  ▼
         ┌──────────────────┐              ┌──────────────────┐
         │  SoC GPIO Driver │              │  I2C GPIO Expander│
         │  (e.g. pl061,    │              │  (e.g. pca953x,  │
         │   gpio-dwapb)    │              │   mcp23017)      │
         │                  │              │                  │
         │  struct gpio_chip│              │  struct gpio_chip│
         │  .of_xlate = ... │              │  .of_xlate = ... │
         │  .of_gpio_n_cells│              │  .can_sleep=true │
         └──────────────────┘              └──────────────────┘
```

---

### الـ Core Abstractions — الهياكل المحورية

#### 1. الـ `device_node` — عقدة الـ Device Tree

```c
/* من linux/of.h */
struct device_node {
    const char      *name;        /* اسم العقدة مثل "gpio@40021000" */
    phandle          phandle;     /* معرّف فريد للإشارة إليها من عقد أخرى */
    const char      *full_name;   /* المسار الكامل: /soc/gpio@40021000 */
    struct fwnode_handle fwnode;  /* جسر للـ generic firmware node API */

    struct property *properties;  /* قائمة مرتبطة بكل الـ properties */
    struct device_node *parent;   /* العقدة الأب في شجرة الـ DT */
    struct device_node *child;    /* أول عقدة ابن */
    struct device_node *sibling;  /* العقدة المجاورة على نفس المستوى */
};
```

**الـ `fwnode_handle`:** تجريد مشترك بين الـ OF (Device Tree) وACPI — يسمح للـ GPIO subsystem بالعمل على أي firmware interface دون تغيير الكود.

#### 2. الـ `property` — وصف خاصية واحدة

```c
struct property {
    char         *name;    /* مثل "reset-gpios" */
    int           length;  /* حجم البيانات بالبايت */
    void         *value;   /* بيانات خام big-endian */
    struct property *next; /* linked list من كل properties العقدة */
};
```

عندما تكتب في الـ DTS:
```dts
reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
```
يُخزَّن هذا كـ `property` باسم `"reset-gpios"` وقيمة raw = 3 خلايا 32-bit.

#### 3. الـ `of_phandle_args` — نتيجة تحليل الـ phandle

```c
struct of_phandle_args {
    struct device_node *np;         /* العقدة التي يشير إليها الـ phandle */
    int                 args_count; /* عدد الـ cells بعد الـ phandle */
    uint32_t            args[MAX_PHANDLE_ARGS]; /* قيم الـ cells */
};
```

بعد تحليل `<&gpio1 5 GPIO_ACTIVE_LOW>`:
- `np` → يشير إلى عقدة `gpio1`
- `args_count` = 2
- `args[0]` = 5 (offset)
- `args[1]` = 1 (GPIO_ACTIVE_LOW)

#### 4. الـ `gpio_chip` — التجريد الأساسي للـ GPIO Controller

```c
struct gpio_chip {
    const char        *label;    /* اسم الـ controller */
    struct device     *parent;   /* الـ device المرتبط */
    struct fwnode_handle *fwnode;/* ربط مع firmware */

    /* Operations — يجب أن يُنفّذها كل driver */
    int  (*get_direction)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_input)(struct gpio_chip *gc, unsigned int offset);
    int  (*direction_output)(struct gpio_chip *gc, unsigned int offset, int value);
    int  (*get)(struct gpio_chip *gc, unsigned int offset);
    int  (*set)(struct gpio_chip *gc, unsigned int offset, int value);

    int   base;   /* أول GPIO number عالمي لهذا الـ chip (deprecated) */
    u16   ngpio;  /* عدد الأطراف في هذا الـ chip */

#ifdef CONFIG_OF_GPIO
    /* حقول خاصة بالـ OF-GPIO integration */
    unsigned int of_gpio_n_cells;  /* عدد cells في الـ specifier (عادةً 2) */

    /* callback لترجمة of_phandle_args → offset + flags */
    int (*of_xlate)(struct gpio_chip *gc,
                    const struct of_phandle_args *gpiospec,
                    u32 *flags);
#endif
};
```

---

### رسم العلاقات بين الـ Structs

```
DTS Source:
  reset-gpios = <&gpio1 5 1>;
         │
         │   of_get_named_gpio(np, "reset-gpios", 0)
         ▼
  ┌─────────────────────────────────┐
  │  struct property                │
  │  .name   = "reset-gpios"        │
  │  .value  = [phandle=gpio1][5][1]│
  └────────────────┬────────────────┘
                   │  of_parse_phandle_with_args()
                   ▼
  ┌─────────────────────────────────┐
  │  struct of_phandle_args         │
  │  .np        → device_node(gpio1)│
  │  .args_count = 2                │
  │  .args[0]   = 5  (offset)       │
  │  .args[1]   = 1  (ACTIVE_LOW)   │
  └────────────────┬────────────────┘
                   │  gc->of_xlate(gc, &gpiospec, &flags)
                   ▼
  ┌─────────────────────────────────┐
  │  struct gpio_chip (gpio1)       │
  │  .base   = 32                   │
  │  .ngpio  = 32                   │
  │  .of_xlate → of_gpio_simple_xlate│
  │                                 │
  │  returns: offset = 5            │
  │  flags:   GPIO_ACTIVE_LOW       │
  └────────────────┬────────────────┘
                   │  base + offset
                   ▼
         Global GPIO number = 37
                   │
                   ▼
  ┌─────────────────────────────────┐
  │  struct gpio_desc               │
  │  (داخل gpiolib)                  │
  │  → يمكن للـ driver استخدامه     │
  │    عبر gpiod_get_value()        │
  └─────────────────────────────────┘
```

---

### الـ Config Guard — `CONFIG_OF_GPIO`

الهيدر يستخدم compile-time guard ذكي:

```c
#ifdef CONFIG_OF_GPIO
/* الدالة الحقيقية — مُعرَّفة في drivers/gpio/gpiolib-of.c */
extern int of_get_named_gpio(const struct device_node *np,
                             const char *list_name, int index);
#else
/* stub يُعيد -ENOSYS لأجهزة بدون Device Tree */
static inline int of_get_named_gpio(const struct device_node *np,
                                    const char *propname, int index)
{
    return -ENOSYS;
}
#endif
```

**لماذا هذا مهم؟** يوجد platforms (مثل بعض x86 أو SPARC configurations) لا تستخدم Device Tree. الـ drivers التي تعتمد على GPIO لكنها مكتوبة بشكل portable يمكنها compile بدون OF دون تغيير الكود — ترى `ENOSYS` وتتعامل معها.

---

### ما يملكه الـ OF-GPIO Framework مقابل ما يُفوّضه للـ Drivers

| المسؤولية | OF-GPIO Framework | GPIO Driver (gpio_chip) |
|---|---|---|
| قراءة الـ property من الـ DT | **يملكها** — `of_parse_phandle_with_args()` | لا شأن له |
| إيجاد الـ `gpio_chip` المناسب | **يملكها** — يبحث عن chip مرتبط بـ `device_node` | لا شأن له |
| ترجمة cells → offset + flags | **يُفوّض** | `of_xlate()` callback |
| التحقق من صحة الـ offset | **يُفوّض** جزئياً | الـ driver يعرف `ngpio` |
| قراءة/كتابة قيمة الطرف | **لا يتدخل** | `get()` / `set()` |
| ضبط الاتجاه | **لا يتدخل** | `direction_input()` / `direction_output()` |
| إدارة الـ IRQ mapping | **لا يتدخل** (شأن `gpio_irq_chip`) | `to_irq()` |
| الـ clock/power management | **لا يتدخل** | `request()` / `free()` |

---

### مثال حقيقي — قراءة GPIO من Driver

```c
/* driver يريد GPIO المسمى "reset" من الـ DTS */
static int my_driver_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    int gpio_num;

    /* OF-GPIO يترجم "reset-gpios" من الـ DTS إلى رقم عالمي */
    gpio_num = of_get_named_gpio(np, "reset-gpios", 0);
    if (!gpio_is_valid(gpio_num)) {
        dev_err(&pdev->dev, "invalid reset GPIO\n");
        return -EINVAL;
    }

    /* الآن نستخدم الـ GPIO API العادية */
    return devm_gpio_request_one(&pdev->dev, gpio_num,
                                 GPIOF_OUT_INIT_HIGH, "my-reset");
}
```

والـ DTS المقابل:
```dts
my-device@0 {
    compatible = "vendor,my-device";
    reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>;
    /*              ▲      ▲        ▲
                  chip  offset   polarity  */
};
```

---

### الـ Subsystem الذي يجب فهمه أولاً

قبل الغوص في تفاصيل OF-GPIO، يجب معرفة:

- **GPIO Subsystem (gpiolib):** المنظومة الأساسية التي تدير `gpio_chip` و`gpio_desc` وتوفر الـ API للـ drivers — OF-GPIO هو مجرد **طبقة ترجمة فوقها**.
- **Device Tree / OF Layer:** آلية وصف الهاردوير في ملفات `.dts` — الـ OF-GPIO يعتمد عليها كلياً لقراءة بيانات الـ GPIO من شجرة الأجهزة.
- **pinctrl subsystem:** يُذكر لأن `gpio/driver.h` يتضمنه — يدير multiplexing الأطراف؛ في كثير من الأحيان يجب ضبط الـ pinmux قبل استخدام الـ GPIO.
## Phase 3: التفاصيل العميقة — الـ Structs والعلاقات

---

### الملف: `include/linux/of_gpio.h`

الـ header الخاص بـ **OF GPIO helpers** — وهو الجسر بين نظام **Device Tree (Open Firmware)** وطبقة **GPIO** في Linux kernel. يتيح للـ drivers استخراج أرقام الـ GPIO pins مباشرةً من خصائص الـ Device Tree node.

---

### 0. جدول الـ Config Options والـ Flags

#### Config Options

| Option | الأثر |
|---|---|
| `CONFIG_OF_GPIO` | يُفعّل دعم OF لـ GPIO؛ بدونه تُعيد الدالة `-ENOSYS` دائماً |
| `CONFIG_GPIOLIB_IRQCHIP` | يُدمج IRQ chip داخل gpiolib، يُفعّل حقل `irq` في `gpio_chip` |
| `CONFIG_OF_GPIO` (في `gpio_chip`) | يُفعّل حقول `of_gpio_n_cells` و`of_xlate` و`of_node_instance_match` |
| `CONFIG_IRQ_DOMAIN_HIERARCHY` | يُفعّل دعم الـ hierarchical interrupt domains في `gpio_irq_chip` |
| `CONFIG_GENERIC_MSI_IRQ` | يُضيف دعم MSI داخل `union gpio_irq_fwspec` |
| `CONFIG_OF_DYNAMIC` / `CONFIG_SPARC` | يُضيف `_flags` لـ `struct property` |
| `CONFIG_OF_KOBJ` | يُضيف `kobj` لـ `device_node` و`attr` لـ `property` |

#### GPIO Direction Constants

| Macro | القيمة | المعنى |
|---|---|---|
| `GPIO_LINE_DIRECTION_IN` | `1` | الـ pin مُهيَّأ كـ input |
| `GPIO_LINE_DIRECTION_OUT` | `0` | الـ pin مُهيَّأ كـ output |

#### قيم الـ Return من `of_get_named_gpio`

| الحالة | القيمة |
|---|---|
| نجاح | رقم الـ GPIO (غير سالب) |
| `CONFIG_OF_GPIO` غير مُفعَّل | `-ENOSYS` |
| الـ property غير موجودة | `-ENOENT` |
| الـ index خارج النطاق | `-EINVAL` |

---

### 1. الـ Structs المهمة

---

#### `struct device_node` — (من `include/linux/of.h`)

**الغرض:** تمثيل node واحدة في شجرة الـ Device Tree داخل الذاكرة.

| الحقل | النوع | الدور |
|---|---|---|
| `name` | `const char *` | اسم الـ node (مثلاً `gpio`) |
| `phandle` | `phandle` (u32) | معرّف فريد يُستخدم للإشارة إليها من nodes أخرى |
| `full_name` | `const char *` | المسار الكامل في الشجرة (مثلاً `/soc/gpio@4000`) |
| `fwnode` | `struct fwnode_handle` | واجهة موحّدة مع firmware subsystem |
| `properties` | `struct property *` | قائمة مرتبطة بكل الخصائص (مثل `gpios = <&gpio 10 0>`) |
| `deadprops` | `struct property *` | الخصائص المحذوفة ديناميكياً |
| `parent` | `struct device_node *` | الـ node الأب في الشجرة |
| `child` | `struct device_node *` | أول node ابن |
| `sibling` | `struct device_node *` | الـ node الأخ التالي |
| `_flags` | `unsigned long` | flags داخلية (initialized, detached...) |
| `data` | `void *` | بيانات خاصة بالمنصة |

**كيف تتصل:** `of_get_named_gpio` تأخذ مؤشراً إلى `device_node` وتقرأ خاصية `gpios` من `properties` لاستخراج phandle الـ GPIO controller ورقم الـ pin.

---

#### `struct property` — (من `include/linux/of.h`)

**الغرض:** تمثيل خاصية واحدة داخل `device_node` مثل `gpios = <&gpio 10 0>`.

| الحقل | النوع | الدور |
|---|---|---|
| `name` | `char *` | اسم الخاصية (مثلاً `"reset-gpios"`) |
| `length` | `int` | الحجم بالـ bytes |
| `value` | `void *` | القيمة الخام بصيغة big-endian |
| `next` | `struct property *` | الخاصية التالية في القائمة المرتبطة |

---

#### `struct of_phandle_args` — (من `include/linux/of.h`)

**الغرض:** حاوية مؤقتة لتخزين نتيجة تفسير مرجع phandle من الـ Device Tree مع وسائطه.

| الحقل | النوع | الدور |
|---|---|---|
| `np` | `struct device_node *` | مؤشر لـ node الـ GPIO controller |
| `args_count` | `int` | عدد الخلايا (عادةً 2: offset + flags) |
| `args[MAX_PHANDLE_ARGS]` | `uint32_t[]` | الوسائط: `args[0]`=offset, `args[1]`=flags |

**كيف تتصل:** الـ OF core يملأ هذه البنية عند تفسير `gpios = <&gpio 10 0>`، ثم يُمرَّر `args` إلى `gc->of_xlate`.

---

#### `struct gpio_chip` — (من `include/linux/gpio/driver.h`)

**الغرض:** التجريد الأساسي لـ **GPIO controller** — يُمثّل أي controller مهما كان نوعه (SOC، FPGA، I2C expander).

| الحقل | النوع | الدور |
|---|---|---|
| `label` | `const char *` | اسم وصفي للـ controller |
| `gpiodev` | `struct gpio_device *` | الحالة الداخلية الـ opaque في gpiolib |
| `parent` | `struct device *` | الـ device الأب في device model |
| `fwnode` | `struct fwnode_handle *` | ربط الـ chip بـ firmware node |
| `owner` | `struct module *` | يمنع إزالة الـ module أثناء الاستخدام |
| `request` | `fn ptr` | hook لتفعيل الـ pin (clock، power) |
| `free` | `fn ptr` | hook لتعطيل الـ pin |
| `get_direction` | `fn ptr` | إرجاع اتجاه الـ pin (input/output) |
| `direction_input` | `fn ptr` | ضبط الـ pin كـ input |
| `direction_output` | `fn ptr` | ضبط الـ pin كـ output مع قيمة |
| `get` | `fn ptr` | قراءة قيمة الـ pin |
| `set` | `fn ptr` | كتابة قيمة الـ pin |
| `to_irq` | `fn ptr` | تحويل GPIO offset إلى Linux IRQ number |
| `base` | `int` | رقم أول GPIO في الـ global space (أو -1 ديناميكي) |
| `ngpio` | `u16` | عدد الـ pins التي يتحكم فيها |
| `can_sleep` | `bool` | `true` إذا كانت العمليات قد تنام (I2C/SPI) |
| `irq` | `struct gpio_irq_chip` | تكامل IRQ (بشرط `CONFIG_GPIOLIB_IRQCHIP`) |
| `of_gpio_n_cells` | `unsigned int` | عدد خلايا مُحدّد GPIO في Device Tree |
| `of_xlate` | `fn ptr` | تحويل `of_phandle_args` إلى chip-relative offset + flags |
| `of_node_instance_match` | `fn ptr` | لـ controllers متعددة في node واحدة |

---

#### `struct gpio_irq_chip` — (من `include/linux/gpio/driver.h`)

**الغرض:** دمج **IRQ controller** داخل `gpio_chip` — يجعل كل GPIO pin مصدراً محتملاً للـ interrupt.

| الحقل | النوع | الدور |
|---|---|---|
| `chip` | `struct irq_chip *` | الـ IRQ chip الفعلي |
| `domain` | `struct irq_domain *` | يربط hwirq بـ Linux IRQ numbers |
| `handler` | `irq_flow_handler_t` | معالج الـ IRQ (مثلاً `handle_edge_irq`) |
| `default_type` | `unsigned int` | نوع الـ trigger الافتراضي |
| `parent_handler` | `irq_flow_handler_t` | معالج الـ interrupt للـ parent |
| `num_parents` | `unsigned int` | عدد الـ parent interrupts |
| `parents` | `unsigned int *` | مصفوفة بأرقام الـ parent IRQs |
| `threaded` | `bool` | `true` إذا كان الـ handling في thread منفصل |
| `initialized` | `bool` | حماية من الاستخدام قبل التهيئة |
| `valid_mask` | `unsigned long *` | bitmask للـ pins الصالحة كـ interrupts |

---

### 2. مخططات علاقات الـ Structs

```
Device Tree (DTS)
      │
      │  gpios = <&gpio_ctrl 10 0>;
      │  reset-gpios = <&gpio_ctrl 5 GPIO_ACTIVE_LOW>;
      ▼
┌─────────────────────────────┐
│       device_node           │◄── np (الـ consumer device)
│  full_name: "/soc/led"      │
│  properties ─────────────► │property: name="reset-gpios"
│                             │          value=[phandle, 5, 1]
│  parent ──────────────────► │device_node: "/soc"
└─────────────────────────────┘
          │
          │ of_get_named_gpio() تبحث في properties
          │ وتحلّل الـ phandle
          ▼
┌─────────────────────────────┐
│     of_phandle_args         │  (مؤقت على الـ stack)
│  np ──────────────────────► │device_node: "/soc/gpio@4000"
│  args_count = 2             │
│  args[0] = 5  (offset)      │
│  args[1] = 1  (flags)       │
└─────────────────────────────┘
          │
          │ تُمرَّر إلى gc->of_xlate()
          ▼
┌─────────────────────────────────────────────┐
│              gpio_chip                       │
│  label = "gpio@4000"                        │
│  base = 32                                  │
│  ngpio = 32                                 │
│  of_gpio_n_cells = 2                        │
│  of_xlate ─────────────────► fn: يحوّل     │
│                                args → offset │
│  gpiodev ──────────────────► gpio_device   │
│  parent  ──────────────────► struct device  │
│  irq ──────────────────────► gpio_irq_chip  │
└─────────────────────────────────────────────┘
          │
          │ base + offset = GPIO number المطلق
          ▼
     GPIO number = 32 + 5 = 37
```

---

#### علاقة `gpio_chip` ↔ `gpio_irq_chip` ↔ `irq_domain`

```
┌──────────────────────────────────────────┐
│            gpio_chip                      │
│  irq:                                    │
│  ┌────────────────────────────────────┐  │
│  │        gpio_irq_chip               │  │
│  │  chip ──────────────► irq_chip     │  │
│  │  domain ────────────► irq_domain   │  │
│  │  handler = handle_edge_irq        │  │
│  │  parents[0] = 45 (parent IRQ)     │  │
│  │  valid_mask = 0xFFFFFFFF          │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
                    │
          irq_domain يربط:
          hwirq 5 ──► Linux IRQ 120
```

---

### 3. دورة حياة OF GPIO

```
[Boot / Driver Probe]
        │
        ▼
Driver يُهيّئ struct gpio_chip
  ├── يملأ: label, base=-1, ngpio=32
  ├── يسجّل callbacks: get, set, direction_*
  ├── يضبط: of_gpio_n_cells = 2
  └── يسجّل of_xlate callback
        │
        ▼
gpiochip_add_data(gc, driver_data)
  ├── gpiolib تختار base ديناميكياً
  ├── تُنشئ gpio_device داخلياً
  ├── تُنشئ irq_domain (إذا gpio_irq_chip مهيَّأ)
  └── تُسجَّل في قائمة gpio_devices العامة
        │
        ▼
[Consumer Driver Probe - يستخدم GPIO]
        │
        ▼
of_get_named_gpio(np, "reset-gpios", 0)
  ├── of_parse_phandle_with_args()
  │     يقرأ الـ property من device_node
  │     يملأ of_phandle_args
  ├── يبحث عن gpio_chip المرتبط بالـ phandle
  ├── يستدعي gc->of_xlate(gc, &args, &flags)
  └── يُعيد: gc->base + offset
        │
        ▼
[استخدام GPIO]
  gpio_request(gpio_num, "reset")
  gpio_direction_output(gpio_num, 1)
  gpio_set_value(gpio_num, 0)
        │
        ▼
[Driver Remove / Teardown]
  gpio_free(gpio_num)
  gpiochip_remove(gc)
    ├── إزالة irq_domain
    ├── إزالة gpio_device
    └── تحرير الـ base range
```

---

### 4. مخططات تدفق الاستدعاء (Call Flow)

#### `of_get_named_gpio` — التدفق الكامل

```
Consumer driver يستدعي:
of_get_named_gpio(np, "reset-gpios", 0)
  │
  ▼
of_get_named_gpio_flags()          [drivers/gpio/gpiolib-of.c]
  │
  ├── of_parse_phandle_with_args(np, "reset-gpios", "#gpio-cells", 0, &args)
  │     │
  │     ├── يقرأ np->properties ابحث عن "reset-gpios"
  │     ├── يُفسَّر phandle → device_node للـ GPIO controller
  │     ├── يقرأ "#gpio-cells" من controller node
  │     └── يملأ args.np, args.args_count, args.args[]
  │
  ├── gpiod_get_from_of_node()
  │     │
  │     ├── يبحث عن gpio_chip المُسجَّل لهذا device_node
  │     └── يستدعي gc->of_xlate(gc, &args, &flags)
  │           │
  │           └── يُعيد: offset داخل الـ chip
  │
  └── يُعيد: gc->base + offset   (رقم GPIO المطلق)
```

#### ضبط قيمة GPIO — تدفق من Consumer إلى Hardware

```
consumer يستدعي:
gpio_set_value(gpio_num, value)
  │
  ▼
gpiod_set_value(desc, value)      [gpiolib core]
  │
  ├── يتحقق من ownership وصحة الـ desc
  │
  ▼
gpiod_set_value_nocheck()
  │
  ▼
gc->set(gc, offset, value)        [driver callback]
  │
  ▼
writel(value, gpio_base + offset_reg)   [hardware register]
```

#### تحويل GPIO إلى IRQ (OF Context)

```
consumer يستدعي:
gpiod_to_irq(desc)
  │
  ▼
gc->to_irq(gc, offset)            [driver callback أو gpiolib default]
  │
  ▼
irq_find_mapping(gc->irq.domain, offset)
  │
  ├── irq_domain يبحث في جدول الـ mapping
  └── يُعيد: Linux IRQ number
```

---

### 5. استراتيجية الـ Locking

#### جدول الـ Locks وما تحميه

| الـ Lock | النوع | ما يحميه |
|---|---|---|
| `gpio_lock` (في gpiolib) | `spinlock_t` | قائمة `gpio_devices` العامة عند التسجيل/الإزالة |
| `gc->gpiodev->srcu` | `struct srcu_struct` | الوصول الـ RCU لقائمة الـ descriptors |
| `gpio_desc->request_lock` | ضمني في gpiolib | تجنّب double-request لنفس الـ pin |
| `irq_domain` locks | داخلية في IRQ subsystem | mapping table للـ hwirq → Linux IRQ |
| `lockdep: lock_key` | `struct lock_class_key` | تتبع نظام lockdep لـ IRQ locks |
| `lockdep: request_key` | `struct lock_class_key` | تتبع lockdep لـ IRQ request locks |

#### ترتيب الـ Locks (Lock Ordering)

```
[قاعدة: يجب الالتزام بهذا الترتيب لتجنّب deadlock]

1. gpio_lock (spinlock — قصير المدة فقط)
        │
        ▼
2. irq_domain_mutex (إذا احتجنا mapping)
        │
        ▼
3. chip-specific lock (إذا كان الـ driver يستخدم lock خاص)
        │
        ▼
4. hardware register access

[ملاحظة مهمة]
- إذا كان can_sleep=true (I2C/SPI expander):
  → لا يجوز استدعاء gpio_set_value من interrupt context
  → يجب استخدام threaded IRQ
  → gpiolib تتحقق من هذا وترفع WARN_ON
```

#### نمط الـ SRCU في gpiolib

```c
/* القراءة — RCU read lock */
srcu_read_lock(&gpio_devices_srcu);
    /* الوصول الآمن للـ gpio_device */
srcu_read_unlock(&gpio_devices_srcu);

/* الكتابة — عند التسجيل */
spin_lock_irqsave(&gpio_lock, flags);
    list_add(&gdev->list, &gpio_devices);
spin_unlock_irqrestore(&gpio_lock, flags);
synchronize_srcu(&gpio_devices_srcu);
```

---

### ملخص العلاقات

```
device_node ──properties──► property ("reset-gpios")
     │                              │
     │                         phandle → device_node (controller)
     │                              │
     └──────────────────────────────┤
                                    ▼
                             gpio_chip
                          ┌────────────────┐
                          │ of_xlate()     │
                          │ base, ngpio    │
                          │ get/set/dir    │
                          │ irq ──────────►gpio_irq_chip
                          │ gpiodev ──────►gpio_device (opaque)
                          │ parent ───────►struct device
                          └────────────────┘
                                    │
                      of_get_named_gpio()
                                    │
                                    ▼
                         GPIO number (integer)
                      ──► consumer driver يستخدمه
                          لطلب الـ pin وضبطه وقراءته
```
## Phase 4: شرح الـ Functions

### ملخص سريع — Cheatsheet

| الـ Function | الـ Header | الـ Config Guard | القيمة المُرجعة |
|---|---|---|---|
| `of_get_named_gpio()` | `linux/of_gpio.h` | `CONFIG_OF_GPIO` | GPIO number أو errno سالب |
| `of_get_named_gpio()` (stub) | `linux/of_gpio.h` | `!CONFIG_OF_GPIO` | `-ENOSYS` دائماً |

---

### تصنيف الـ Functions

الـ header يحتوي على **function واحدة فعلية** فقط، مع **stub** مقابلها لحالة غياب الـ `CONFIG_OF_GPIO`. هذا النمط شائع جداً في kernel لضمان أن الـ drivers تُكمّل الـ linking حتى بدون دعم الـ OF GPIO.

---

### المجموعة الوحيدة: Device Tree GPIO Lookup

هذه المجموعة مسؤولة عن **ترجمة مرجع GPIO من الـ Device Tree** إلى رقم GPIO صالح للاستخدام مع الـ legacy GPIO API. الـ function تقرأ الـ property المُسمّاة من الـ `device_node`، تحلّ الـ phandle المُشير إلى الـ GPIO controller، ثم تُرجع الـ GPIO number الـ global بعد دمج `gpio_chip->base` مع الـ offset المُعطى في الـ DT.

---

### `of_get_named_gpio` — النسخة الفعلية

```c
extern int of_get_named_gpio(const struct device_node *np,
                             const char *list_name, int index);
```

**ما تفعله:**
تقرأ الـ property المُسمّاة `list_name` من الـ `device_node`، تُحلّل الـ phandle داخلها عند الـ `index` المُحدد، ثم تُترجمه إلى رقم GPIO صالح عبر `of_get_named_gpiod_flags()` → `gpiod_to_gpio()` داخلياً. هي الطريقة الرسمية للحصول على GPIO من DT property مثل `reset-gpios = <&gpio1 5 GPIO_ACTIVE_LOW>`.

**المعاملات:**

| المعامل | النوع | الشرح |
|---|---|---|
| `np` | `const struct device_node *` | الـ node الذي يحتوي على الـ property، عادةً `pdev->dev.of_node` |
| `list_name` | `const char *` | اسم الـ DT property مثل `"reset-gpios"` أو `"enable-gpios"` |
| `index` | `int` | رقم الـ GPIO داخل الـ list (0-based)، للـ properties التي تحتوي أكثر من GPIO |

**القيمة المُرجعة:**

- **قيمة موجبة أو صفر**: رقم الـ GPIO الـ global (`gpio_chip->base + offset`) جاهز للاستخدام مع `gpio_request()` أو `gpio_direction_input()` إلخ.
- **قيمة سالبة**: error code:
  - `-ENOENT`: الـ property غير موجودة في الـ node.
  - `-EINVAL`: الـ property موجودة لكن الـ format خاطئ أو الـ index خارج النطاق.
  - `-EPROBE_DEFER`: الـ GPIO controller لم يُسجَّل بعد — يجب إعادة المحاولة لاحقاً.
  - `-ENODEV`: الـ GPIO chip غير موجود.

**تفاصيل مهمة:**

- تُستدعى في **سياق الـ probe** فقط، وهو سياق يُسمح فيه بالـ sleeping لأن بعض مسارات الـ resolution تحتاج mutex.
- داخلياً تستدعي `of_get_named_gpiod_flags()` ثم `desc_to_gpio()` لتحويل الـ `gpio_desc` إلى رقم integer — هذا الـ bridge هو السبب في التعليق `/* FIXME: Shouldn't be here */` على `#include <linux/gpio.h>`؛ الـ OF layer يجب أن لا يعتمد على الـ legacy integer GPIO API.
- لا تحجز (request) الـ GPIO — المُستدعي مسؤول عن `gpio_request()` أو `devm_gpio_request()` لاحقاً.
- تدعم الـ `index` لأن DT property واحدة قد تُعرّف أكثر من GPIO في قائمة.
- الـ `-EPROBE_DEFER` يجب أن يُعاد propagation بشكل صحيح من الـ probe function وإلا يختل ترتيب التهيئة.

**من يستدعيها:**

الـ platform drivers في مرحلة الـ `probe`، مثال:

```c
static int my_driver_probe(struct platform_device *pdev)
{
    int reset_gpio;

    /* Read index 0 of "reset-gpios" property from DT */
    reset_gpio = of_get_named_gpio(pdev->dev.of_node, "reset-gpios", 0);
    if (reset_gpio < 0) {
        if (reset_gpio == -EPROBE_DEFER)
            return reset_gpio; /* retry later */
        dev_err(&pdev->dev, "failed to get reset gpio: %d\n", reset_gpio);
        return reset_gpio;
    }

    /* Now request and use the GPIO */
    return devm_gpio_request_one(&pdev->dev, reset_gpio,
                                 GPIOF_OUT_INIT_HIGH, "my-reset");
}
```

---

### `of_get_named_gpio` — الـ Stub (بدون CONFIG_OF_GPIO)

```c
static inline int of_get_named_gpio(const struct device_node *np,
                                    const char *propname, int index)
{
    return -ENOSYS;
}
```

**ما تفعله:**
نسخة فارغة تُرجع `-ENOSYS` دائماً. تُوجد لضمان أن الـ drivers التي لا تعتمد **صراحةً** على GPIO-OF support تُكمّل الـ compilation والـ linking بدون أخطاء حتى على منصات لا تدعم `CONFIG_OF_GPIO`.

**تفاصيل مهمة:**

- الـ `static inline` يضمن zero overhead — الـ compiler يُزيل الاستدعاء كاملاً إذا كان الـ driver يتحقق صحيح من الـ return value.
- الـ driver الصحيح يجب أن يتعامل مع `-ENOSYS` كحالة "GPIO غير متاح" وليس كـ fatal error.
- المعاملات متطابقة مع النسخة الحقيقية لضمان تبادلية الـ API عبر الـ Kconfig options.

---

### تدفق العمل الداخلي — Pseudocode Flow

```
of_get_named_gpio(np, list_name, index)
│
├── of_get_named_gpiod_flags(np, list_name, index, &flags)
│   ├── of_parse_phandle_with_args(np, list_name, "#gpio-cells", index, &args)
│   │   └── يُحلّل الـ phandle ويستخرج args[] من الـ DT property
│   ├── يبحث عن gpio_chip المرتبط بالـ args.np
│   │   └── إذا لم يجد → -EPROBE_DEFER
│   └── يُنشئ gpio_desc من (gpio_chip, offset=args.args[0])
│
└── desc_to_gpio(desc)
    └── يُرجع gpio_chip->base + hwirq  ← الرقم النهائي
```

---

### ملاحظة معمارية

الـ header `of_gpio.h` هو **bridge layer** بين:

- **الـ OF/DT subsystem** (`linux/of.h`) الذي يعمل بـ `device_node` و `property`.
- **الـ legacy integer GPIO API** (`linux/gpio.h`) الذي يعمل بأرقام GPIO الـ global.

الاتجاه الحديث في الـ kernel هو استخدام `devm_gpiod_get()` / `gpiod_get()` مباشرةً مع الـ descriptor-based API بدلاً من `of_get_named_gpio()`. هذه الأخيرة باقية للـ legacy drivers التي تعتمد على الـ integer GPIO numbers.
## Phase 5: دليل الـ Debugging الشامل

يغطي هذا الدليل تقنيات debugging كاملة لـ subsystem الـ `of_gpio` — وهو الطبقة التي تربط بين الـ **Device Tree** وواجهة الـ **GPIO** في الـ Linux kernel، محورها الرئيسي دالة `of_get_named_gpio()` والـ `struct device_node` المرتبطة بها.

---

### Software Level

#### 1. مدخلات الـ debugfs

الـ **debugfs** يكشف حالة الـ GPIO والـ Device Tree داخلياً:

```bash
# Mount debugfs إن لم يكن مُوصَّلاً
mount -t debugfs none /sys/kernel/debug

# عرض كل GPIO chips المسجّلة مع تفاصيل كل line
cat /sys/kernel/debug/gpio

# عرض شجرة الـ Device Tree كاملة بصيغة مقروءة
cat /sys/kernel/debug/of_component_probe   # إن وُجد
ls /sys/kernel/debug/pinctrl/              # لكل pinctrl controller
cat /sys/kernel/debug/pinctrl/<controller>/pins
cat /sys/kernel/debug/pinctrl/<controller>/pingroups
cat /sys/kernel/debug/pinctrl/<controller>/pinmux-pins
```

**تفسير output الـ `/sys/kernel/debug/gpio`:**

```
gpiochip0: GPIOs 0-31, parent: platform/gpio0, gpio-controller:
 gpio-5  (                    ) in  hi    IRQ
 gpio-12 (reset               ) out lo
 gpio-18 (spi-cs              ) out hi
```

- **in/out**: اتجاه الـ GPIO
- **hi/lo**: القيمة الحالية
- **IRQ**: مُستخدم كـ interrupt

```bash
# قراءة GPIO chip محددة مع كل consumers
cat /sys/kernel/debug/gpio | grep -A5 "gpiochip0"

# فحص الـ DT nodes التي طلبت GPIO بعينه
grep -r "gpios" /sys/firmware/devicetree/base/ 2>/dev/null | head -20
```

#### 2. مدخلات الـ sysfs

```bash
# قائمة كل GPIO controllers
ls /sys/bus/platform/drivers/gpio*/
ls /sys/class/gpio/

# export GPIO يدوياً للاختبار (GPIO رقم 18 مثلاً)
echo 18 > /sys/class/gpio/export
cat /sys/class/gpio/gpio18/direction
cat /sys/class/gpio/gpio18/value
cat /sys/class/gpio/gpio18/edge

# عرض الـ device node المرتبط بالـ GPIO chip
ls -la /sys/class/gpio/gpiochip0/device/of_node
cat /sys/class/gpio/gpiochip0/device/of_node/compatible

# فحص الـ properties من الـ DT عبر sysfs
ls /sys/firmware/devicetree/base/
# تصفح node معين
ls /sys/firmware/devicetree/base/leds/
xxd /sys/firmware/devicetree/base/leds/gpios   # قراءة الـ phandle + args
```

**جدول مسارات الـ sysfs الجوهرية:**

| المسار | ما يكشفه |
|--------|----------|
| `/sys/class/gpio/gpiochipN/base` | رقم أول GPIO في الـ chip |
| `/sys/class/gpio/gpiochipN/ngpio` | عدد الـ GPIOs |
| `/sys/class/gpio/gpiochipN/label` | اسم الـ chip |
| `/sys/class/gpio/gpioN/direction` | `in` أو `out` |
| `/sys/class/gpio/gpioN/value` | `0` أو `1` |
| `/sys/firmware/devicetree/base/` | شجرة الـ DT كاملة |

#### 3. الـ ftrace — تتبع مسار `of_get_named_gpio()`

```bash
# تفعيل tracing للـ GPIO و OF subsystems
cd /sys/kernel/debug/tracing

# عرض الـ events المتاحة
grep -i gpio available_events
grep -i "of\b"  available_events

# تفعيل events الـ GPIO
echo 1 > events/gpio/enable

# تتبع function calls مباشرة
echo function > current_tracer
echo "of_get_named_gpio" > set_ftrace_filter
echo "of_get_named_gpio_flags" >> set_ftrace_filter
echo "gpiod_get_index" >> set_ftrace_filter
echo "of_parse_phandle_with_args" >> set_ftrace_filter
echo 1 > tracing_on
# شغّل الـ driver أو أعد تحميله
cat trace
echo 0 > tracing_on
echo nop > current_tracer
```

**مثال على output متوقع:**

```
      modprobe-1234  [000] ....  12.345678: of_get_named_gpio <-mydriver_probe
      modprobe-1234  [000] ....  12.345679: of_parse_phandle_with_args <-of_get_named_gpio
      modprobe-1234  [000] ....  12.345680: of_xlate_and_get_gpiod_flags <-of_get_named_gpio
```

```bash
# function_graph tracer لرؤية التسلسل الكامل
echo function_graph > current_tracer
echo "of_get_named_gpio" > set_graph_function
echo 1 > tracing_on
# trigger the driver load
cat trace | head -50
```

#### 4. تفعيل الـ printk والـ dynamic debug

```bash
# تفعيل debug messages لـ GPIO subsystem بأكمله
echo "file drivers/gpio/*.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file include/linux/of_gpio.h +p" > /sys/kernel/debug/dynamic_debug/control

# تفعيل debug للـ OF/DT subsystem
echo "file drivers/of/*.c +p" > /sys/kernel/debug/dynamic_debug/control
echo "file lib/devres.c +p" >> /sys/kernel/debug/dynamic_debug/control

# رؤية كل الـ dynamic debug flags النشطة
cat /sys/kernel/debug/dynamic_debug/control | grep gpio

# رفع مستوى الـ loglevel مؤقتاً
echo 8 > /proc/sys/kernel/printk

# تفعيل debug من سطر kernel boot
# في /etc/default/grub:
# GRUB_CMDLINE_LINUX="... dyndbg='file of_gpio.c +p; file gpiolib-of.c +p'"
```

**إضافة `pr_debug()` في kernel module مخصص:**

```c
#define pr_fmt(fmt) "mydrv: " fmt
#define DEBUG  /* يجب قبل أي include لتفعيل pr_debug */

#include <linux/of_gpio.h>

static int mydrv_probe(struct platform_device *pdev)
{
    int gpio;
    gpio = of_get_named_gpio(pdev->dev.of_node, "reset-gpios", 0);
    pr_debug("of_get_named_gpio returned: %d\n", gpio);  /* يظهر مع dyndbg */
    if (gpio < 0) {
        dev_err(&pdev->dev, "GPIO request failed: %d\n", gpio);
        return gpio;
    }
    return 0;
}
```

#### 5. خيارات الـ Kernel Config للـ Debugging

| الخيار | الوظيفة |
|--------|----------|
| `CONFIG_OF_GPIO` | تفعيل دعم GPIO من DT (شرط أساسي) |
| `CONFIG_GPIOLIB` | مكتبة GPIO الأساسية |
| `CONFIG_DEBUG_GPIO` | رسائل debug مفصّلة لكل عمليات GPIO |
| `CONFIG_GPIO_SYSFS` | كشف GPIO عبر sysfs للاختبار اليدوي |
| `CONFIG_OF_DYNAMIC` | دعم تعديل DT ديناميكياً (overlays) |
| `CONFIG_OF_OVERLAY` | دعم DT overlays للاختبار |
| `CONFIG_PROVE_LOCKING` | كشف deadlocks في GPIO locks |
| `CONFIG_DEBUG_SPINLOCK` | كشف مشاكل spinlock في GPIO IRQ |
| `CONFIG_DYNAMIC_DEBUG` | تفعيل `pr_debug()` ديناميكياً بلا recompile |
| `CONFIG_FTRACE` | تتبع function calls |
| `CONFIG_FUNCTION_TRACER` | تتبع كل دوال الكيرنل |
| `CONFIG_OF_UNITTEST` | اختبارات وحدة لـ OF/DT subsystem |

```bash
# التحقق من الخيارات المُفعّلة في الـ kernel الحالي
zcat /proc/config.gz | grep -E "CONFIG_(OF_GPIO|DEBUG_GPIO|GPIO_SYSFS|OF_OVERLAY)"
# أو
grep -E "CONFIG_(OF_GPIO|DEBUG_GPIO)" /boot/config-$(uname -r)
```

#### 6. أدوات الـ devlink والـ GPIO-specific

```bash
# gpiodetect: عرض كل GPIO chips
gpiodetect          # من حزمة libgpiod-tools

# gpioinfo: تفاصيل كل line في chip معينة
gpioinfo gpiochip0

# gpioget: قراءة قيمة GPIO
gpioget gpiochip0 5

# gpiomon: مراقبة تغيرات GPIO
gpiomon --num-events=10 gpiochip0 5

# فحص DT من userspace
dtc -I fs /sys/firmware/devicetree/base > /tmp/current.dts 2>/dev/null
grep -A5 "reset-gpios\|gpios" /tmp/current.dts

# fdt_dump بديل
fdtdump /sys/firmware/fdt 2>/dev/null | grep -A3 "gpio"
```

#### 7. جدول رسائل الخطأ الشائعة

| رسالة الخطأ | المعنى | الحل |
|-------------|--------|-------|
| `of_get_named_gpio: invalid GPIO` | رقم GPIO خارج النطاق أو سالب | تحقق من DT property وقيمة args |
| `GPIO: pin X is not set as output` | محاولة كتابة على GPIO مُضبوط كـ input | استدعِ `gpio_direction_output()` أولاً |
| `can't request region for resource` | تعارض على عنوان الـ GPIO controller | تحقق من عدم تعارض `iomem` |
| `-EPROBE_DEFER` من `of_get_named_gpio` | الـ GPIO chip لم يُسجَّل بعد | طبيعي، الـ kernel سيعيد المحاولة تلقائياً |
| `-ENOENT` | الـ property غير موجودة في DT | راجع اسم الـ property في DTS |
| `-EINVAL` | صيغة الـ property غير صحيحة (cells خاطئة) | تحقق من `#gpio-cells` في الـ GPIO controller node |
| `gpiochip_add: GPIOs X..Y already in use` | تعارض بين drivers | راجع من يملك نطاق GPIO هذا |
| `of_parse_phandle_with_args: phandle not found` | الـ phandle في DT يشير إلى node غير موجود | تحقق من صحة الـ phandle |
| `irq: type not supported` | نوع الـ IRQ trigger غير مدعوم من الـ chip | راجع `irq_set_irq_type()` والـ driver |
| `Unable to request GPIO` | الـ GPIO مُحجوز بالفعل من driver آخر | استخدم `gpioinfo` لمعرفة المالك |

#### 8. نقاط استراتيجية لـ `dump_stack()` و`WARN_ON()`

```c
#include <linux/of_gpio.h>
#include <linux/bug.h>

static int mydrv_get_gpio(struct device_node *np, const char *prop)
{
    int gpio;

    /* نقطة 1: تحقق من صحة الـ device_node قبل أي عملية */
    if (WARN_ON(!np)) {
        pr_err("NULL device_node passed\n");
        return -EINVAL;
    }

    gpio = of_get_named_gpio(np, prop, 0);

    /* نقطة 2: رصد EPROBE_DEFER لتحليل ترتيب التهيئة */
    if (gpio == -EPROBE_DEFER) {
        pr_debug("GPIO provider not ready yet for '%s'\n", prop);
        return gpio;   /* لا dump_stack هنا — هذا سلوك طبيعي */
    }

    /* نقطة 3: dump stack عند فشل غير متوقع */
    if (WARN_ON(gpio < 0 && gpio != -ENOENT)) {
        dump_stack();  /* يكشف من استدعى هذه الدالة */
        return gpio;
    }

    /* نقطة 4: WARN إذا طُلب GPIO خارج نطاق الـ chip */
    WARN_ON(gpio > 1024);  /* حد تقريبي — يكشف قيم مشبوهة */

    return gpio;
}
```

**أماكن إضافة `WARN_ON()` في الـ gpiolib-of.c:**

```c
/* عند xlate: تحقق من تطابق عدد الـ args مع #gpio-cells */
WARN_ON(gpiospec->args_count != gc->of_gpio_n_cells);

/* عند تسجيل chip جديدة: تحقق من عدم تداخل النطاقات */
WARN_ON(gpio_is_valid(gc->base) &&
        gpiochip_find_base(gc->ngpio) != gc->base);
```

---

### Hardware Level

#### 1. التحقق من تطابق الحالة بين الـ Hardware والـ Kernel

```bash
# 1. قراءة قيمة GPIO من الـ kernel
cat /sys/class/gpio/gpio18/value    # ما يراه الـ kernel

# 2. تصدير GPIO وقياسه بـ multimeter في نفس الوقت
echo 18 > /sys/class/gpio/export
echo out > /sys/class/gpio/gpio18/direction
echo 1 > /sys/class/gpio/gpio18/value
# قِس الجهد على الـ pin — يجب أن يكون VCC (3.3V أو 5V حسب الـ SoC)
echo 0 > /sys/class/gpio/gpio18/value
# الجهد يجب أن يصبح 0V (أو قريب منه)

# 3. التحقق من الـ pull-up/pull-down
# إذا كان GPIO كـ input وقيمته غير ثابتة، قد يكون floating
cat /sys/class/gpio/gpio18/value    # قرأ عدة مرات، يجب أن يستقر
```

#### 2. تقنيات Register Dump

```bash
# قراءة سجل GPIO controller مباشرة (مثال لـ BCM2835 - Raspberry Pi)
# عنوان GPIO base = 0x3F200000 لـ RPi 2/3

# باستخدام devmem2
devmem2 0x3F200034 w   # GPLEV0 - قيم GPIOs 0-31
devmem2 0x3F200038 w   # GPLEV1 - قيم GPIOs 32-53

# قراءة GPFSEL (Function Select) لـ GPIOs 0-9
devmem2 0x3F200000 w   # GPFSEL0

# باستخدام /dev/mem (يحتاج CONFIG_DEVMEM=y)
python3 -c "
import mmap, struct
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), 4096, offset=0x3F200000)
    val = struct.unpack('<I', m[0x34:0x38])[0]
    print(f'GPLEV0 = 0x{val:08X}')
    for i in range(32):
        print(f'GPIO{i} = {(val >> i) & 1}')
"

# باستخدام io utility (من package io أو memtool)
io -4 0x3F200034   # قراءة 4 bytes

# فحص الـ iomem المُخصص للـ GPIO controller
cat /proc/iomem | grep -i gpio
```

**مثال لهيكل سجلات GPIO نموذجي (ARM SoC):**

```
عنوان قاعدة GPIO = 0xXXXX0000
+-----------+--------+---------------------------+
| Offset    | الاسم  | الوظيفة                   |
+-----------+--------+---------------------------+
| 0x000     | GPDIR  | Data Direction Register    |
| 0x004     | GPDAT  | Data Register (R/W)        |
| 0x008     | GPEN   | Output Enable              |
| 0x00C     | GPINTCFG| Interrupt Config          |
| 0x010     | GPINTSTAT| Interrupt Status         |
+-----------+--------+---------------------------+
```

#### 3. نصائح الـ Logic Analyzer والـ Oscilloscope

**إعداد الـ Logic Analyzer لتحليل GPIO:**

```
Channel Setup:
- CH0: GPIO pin المُختبَر
- CH1: CLK signal (إن وُجد)
- CH2: Enable/CS signal
- Trigger: Rising/Falling edge على CH0

معدل الأخذ المناسب:
- GPIO عادي: 1MHz يكفي
- GPIO كـ SPI CS: 10-100MHz
- GPIO كـ I2C: 400kHz - 1MHz

Decode Options:
- إذا GPIO مرتبط بـ SPI: فعّل SPI decoder
- إذا GPIO مرتبط بـ I2C: فعّل I2C decoder
```

**نقاط قياس حرجة:**

```
SoC GPIO Pin ──────────────── Target Device
        |                           |
        +── 10KΩ Pull-up ── VCC     |
        |                           |
      [CH0 Logic Analyzer]          |
        |                           |
        +──────────────────────────>|
             قِس هنا بالـ probe
```

```bash
# مزامنة kernel event مع قياس الـ oscilloscope
# أضف toggle في الكود:
echo 1 > /sys/class/gpio/gpio18/value  # rising edge
cat /sys/class/gpio/gpio18/value        # تأكيد
echo 0 > /sys/class/gpio/gpio18/value  # falling edge
# على الـ oscilloscope: يجب رؤية نبضة واضحة
```

#### 4. مشاكل الـ Hardware الشائعة وأنماطها في الـ Kernel Log

| مشكلة الـ Hardware | نمط في dmesg | التحقق |
|-------------------|--------------|--------|
| GPIO floating (غير موصول بـ pull) | قيمة تتقلب عشوائياً | `gpiomon gpiochip0 N` — يرى toggles مستمرة |
| جهد خاطئ (5V على SoC 3.3V) | `GPIO: timeout waiting for GPIO to settle` | قِس بـ multimeter على الـ GPIO pad |
| تعارض PIN مع pinctrl | `pin X is already requested` | `cat /sys/kernel/debug/pinctrl/*/pins` |
| GPIO controller لم يُهيَّأ | `gpiochip_add: base 0 is invalid` | تحقق من ترتيب تهيئة الـ drivers |
| مشكلة power domain | GPIO يعمل أحياناً فقط | تحقق من `CONFIG_PM` وتسلسل الـ power |
| GPIO مُتحكَّم به من firmware | `can't map GPIO to IRQ` | تحقق من ACPI/UEFI GPIO ownership |

**أنماط dmesg المشبوهة:**

```bash
# ابحث عن رسائل GPIO في dmesg
dmesg | grep -i "gpio\|of_gpio\|gpiochip" | grep -E "error|fail|invalid|warn"

# أنماط خطيرة
dmesg | grep "gpio.*-EPROBE_DEFER"   # driver ينتظر
dmesg | grep "GPIO.*busy"            # تعارض
dmesg | grep "gpiolib: detected"     # اكتشاف تلقائي
```

#### 5. تصحيح الـ Device Tree والتحقق من تطابقه مع الـ Hardware

```bash
# 1. استخراج الـ DT المُحمَّل حالياً
dtc -I fs -O dts /sys/firmware/devicetree/base -o /tmp/live.dts 2>/dev/null

# 2. البحث عن كل GPIO references
grep -n "gpios\|gpio-controller\|gpio-cells" /tmp/live.dts

# 3. مثال نموذجي لـ node صحيح:
cat << 'EOF'
/* DTS صحيح */
gpio0: gpio@3f200000 {
    compatible = "brcm,bcm2835-gpio";
    reg = <0x3f200000 0xb4>;
    gpio-controller;
    #gpio-cells = <2>;        /* يجب أن يتطابق مع args في of_get_named_gpio */
    gpio-ranges = <&pmx 0 0 54>;
};

mydevice {
    compatible = "vendor,mydev";
    reset-gpios = <&gpio0 18 GPIO_ACTIVE_LOW>;
    /*              ^      ^  ^
                    phandle pin flags  */
};
EOF

# 4. التحقق من تطابق #gpio-cells مع عدد الـ args
# of_get_named_gpio(np, "reset-gpios", 0) يتوقع args[0]=pin, args[1]=flags
# إذا #gpio-cells = <2> فهذا صحيح
xxd /sys/firmware/devicetree/base/gpio@3f200000/\#gpio-cells

# 5. DT Overlay للاختبار بدون reboot
cat > /tmp/test_gpio.dts << 'EOF'
/dts-v1/;
/plugin/;
/ {
    fragment@0 {
        target-path = "/";
        __overlay__ {
            test_gpio_node {
                compatible = "test,gpio-debug";
                test-gpios = <&gpio0 20 0>;
                status = "okay";
            };
        };
    };
};
EOF
dtc -I dts -O dtb /tmp/test_gpio.dts -o /tmp/test_gpio.dtbo
# تحميل الـ overlay (يحتاج CONFIG_OF_OVERLAY + configfs)
mkdir -p /sys/kernel/config/device-tree/overlays/test
cp /tmp/test_gpio.dtbo /sys/kernel/config/device-tree/overlays/test/dtbo

# 6. التحقق من أن الـ OF node مُرتبط بالـ device الصحيح
ls -la /sys/bus/platform/devices/*/of_node
readlink /sys/bus/platform/devices/mydevice/of_node
# يجب أن يشير إلى /sys/firmware/devicetree/base/mydevice
```

---

### Practical Commands

#### أوامر جاهزة للنسخ والتشغيل

**أولاً: فحص شامل سريع لـ GPIO + OF:**

```bash
#!/bin/bash
# gpio_of_debug.sh — فحص شامل لـ of_gpio subsystem

echo "=== GPIO Chips ==="
gpiodetect 2>/dev/null || ls /sys/class/gpio/gpiochip*/label

echo "=== GPIO Details ==="
gpioinfo 2>/dev/null || cat /sys/kernel/debug/gpio

echo "=== DT GPIO References ==="
dtc -I fs /sys/firmware/devicetree/base -o - 2>/dev/null | \
    grep -A2 "gpios\s*="

echo "=== Active GPIO Requests ==="
cat /sys/kernel/debug/gpio | grep -v "^$\|^gpio"

echo "=== Pinctrl Status ==="
for f in /sys/kernel/debug/pinctrl/*/pinmux-pins; do
    echo "--- $f ---"
    cat "$f" | grep "gpio\|GPIO"
done

echo "=== Kernel Config Check ==="
zcat /proc/config.gz 2>/dev/null | grep -E "CONFIG_(OF_GPIO|DEBUG_GPIO|GPIO_SYSFS)="
```

**ثانياً: تتبع `of_get_named_gpio()` بالكامل:**

```bash
#!/bin/bash
# trace_of_gpio.sh

TRACEDIR=/sys/kernel/debug/tracing

# إيقاف أي tracing سابق
echo 0 > $TRACEDIR/tracing_on
echo nop > $TRACEDIR/current_tracer
echo "" > $TRACEDIR/set_ftrace_filter

# إعداد الـ function graph tracer
echo function_graph > $TRACEDIR/current_tracer
cat >> $TRACEDIR/set_graph_function << 'FUNCS'
of_get_named_gpio
of_parse_phandle_with_args
of_xlate_and_get_gpiod_flags
gpiod_get_index
FUNCS

echo 1 > $TRACEDIR/tracing_on
echo "Loading module (replace with your module name)..."
modprobe mydriver 2>&1
sleep 2
echo 0 > $TRACEDIR/tracing_on

echo "=== Trace Output ==="
cat $TRACEDIR/trace

# تنظيف
echo nop > $TRACEDIR/current_tracer
```

**ثالثاً: اختبار GPIO بعينه من الـ DT:**

```bash
#!/bin/bash
# test_gpio_from_dt.sh GPIO_NUMBER

GPIO=${1:-18}
echo "Testing GPIO $GPIO"

# Export
echo $GPIO > /sys/class/gpio/export 2>/dev/null || true

# Direction test
echo "=== Setting as output ==="
echo out > /sys/class/gpio/gpio${GPIO}/direction
echo 1 > /sys/class/gpio/gpio${GPIO}/value
sleep 0.1
VAL=$(cat /sys/class/gpio/gpio${GPIO}/value)
echo "Written 1, Read: $VAL"

echo 0 > /sys/class/gpio/gpio${GPIO}/value
sleep 0.1
VAL=$(cat /sys/class/gpio/gpio${GPIO}/value)
echo "Written 0, Read: $VAL"

# Input test
echo "=== Setting as input ==="
echo in > /sys/class/gpio/gpio${GPIO}/direction
echo "Current value: $(cat /sys/class/gpio/gpio${GPIO}/value)"

# Cleanup
echo $GPIO > /sys/class/gpio/unexport
```

**رابعاً: فحص DT لـ GPIO property محددة:**

```bash
#!/bin/bash
# check_dt_gpio_prop.sh <node_path> <property_name>
# مثال: ./check_dt_gpio_prop.sh /leds/led0 gpios

NODE_PATH=${1:-/}
PROP=${2:-gpios}
DT_BASE=/sys/firmware/devicetree/base

FULL_PATH="${DT_BASE}${NODE_PATH}"

if [ ! -d "$FULL_PATH" ]; then
    echo "Node not found: $FULL_PATH"
    exit 1
fi

PROP_FILE="${FULL_PATH}/${PROP}"
if [ ! -f "$PROP_FILE" ]; then
    echo "Property '$PROP' not found in $NODE_PATH"
    echo "Available properties:"
    ls "$FULL_PATH"
    exit 1
fi

echo "=== Raw bytes of '$PROP' ==="
xxd "$PROP_FILE"

echo "=== Interpreted as phandle + args ==="
python3 -c "
import struct, sys
data = open('$PROP_FILE', 'rb').read()
words = struct.unpack('>%dI' % (len(data)//4), data)
print(f'phandle: 0x{words[0]:08X} ({words[0]})')
for i, a in enumerate(words[1:], 1):
    print(f'arg[{i-1}]: {a} (0x{a:08X})')
"
```

**خامساً: قراءة registers الـ GPIO controller مباشرة:**

```bash
#!/bin/bash
# dump_gpio_regs.sh <base_address> <size>
# مثال: ./dump_gpio_regs.sh 0x3F200000 0xB4

BASE=${1:-0x3F200000}
SIZE=${2:-0xB4}

if ! command -v devmem2 &>/dev/null; then
    echo "devmem2 not found, trying /dev/mem..."
    python3 -c "
import mmap, struct, sys
base = $BASE
size = $SIZE
with open('/dev/mem', 'rb') as f:
    m = mmap.mmap(f.fileno(), (size+4095)&~4095, offset=base&~4095)
    off = base & 4095
    for i in range(0, size, 4):
        val = struct.unpack('<I', m[off+i:off+i+4])[0]
        print(f'0x{base+i:08X}: 0x{val:08X}  {val:032b}')
"
    exit 0
fi

echo "=== GPIO Register Dump @ $BASE ==="
ADDR=$BASE
while [ $((ADDR - BASE)) -lt $SIZE ]; do
    VAL=$(devmem2 $ADDR w 2>/dev/null | grep "Value at" | awk '{print $NF}')
    printf "0x%08X: %s\n" $ADDR "$VAL"
    ADDR=$((ADDR + 4))
done
```

#### تفسير النتائج المتوقعة

**`gpioinfo` output طبيعي:**
```
gpiochip0 - 54 lines:
        line   0:      unnamed       unused   input  active-high
        line  18:  "reset-gpio"      "mydrv"  output active-low  [used]
        line  25:     "led-red"      "leds"   output active-high [used]
```
- **unused**: GPIO حر، لا driver يمتلكه
- **used**: مُحجوز من driver (الاسم بين `""`)
- **active-low**: الـ kernel يعكس المنطق تلقائياً

**`/sys/kernel/debug/gpio` output مع مشكلة:**
```
gpio-18  (reset               ) out hi    -- تحذير: DT يقول active-low لكن قيمة hi تعني فعلياً LOW
```

**`dmesg` عند نجاح تهيئة GPIO من DT:**
```
[    2.345678] mydriver: GPIO 18 (reset) obtained from DT property 'reset-gpios'
[    2.345679] mydriver: GPIO direction set to output, initial value: 0
```
## Phase 6: سيناريوهات من الحياة العملية

---

### السيناريو 1: بوابة صناعية على RK3562 — GPIO لا يستجيب عند التشغيل

#### العنوان
**الـ** `of_get_named_gpio` يُرجع `-ENOSYS` على بوابة صناعية مبنية على RK3562

#### السياق
شركة تبني **industrial gateway** تعمل بنظام Linux مخصص على **Rockchip RK3562**.
المنتج يتحكم في relay خارجي عبر GPIO لفتح/غلق دوائر كهربائية.
المبرمج يستخدم driver مكتوب منذ سنتين على منصة مختلفة.

#### المشكلة
عند boot، الـ driver يستدعي `of_get_named_gpio()` ويحصل على `-ENOSYS` دائماً، مما يجعل الـ relay يبقى مغلقاً ولا يمكن التحكم فيه.

#### التحليل
بالنظر في `of_gpio.h`:

```c
#ifdef CONFIG_OF_GPIO

extern int of_get_named_gpio(const struct device_node *np,
                             const char *list_name, int index);

#else /* CONFIG_OF_GPIO */

#include <linux/errno.h>

/* Drivers may not strictly depend on the GPIO support, so let them link. */
static inline int of_get_named_gpio(const struct device_node *np,
                                   const char *propname, int index)
{
    return -ENOSYS;  /* <-- هذا ما يحدث! */
}

#endif /* CONFIG_OF_GPIO */
```

الملف يحتوي على **compile-time guard**: إذا لم يكن `CONFIG_OF_GPIO` مُفعَّلاً في الـ kernel config، فإن `of_get_named_gpio` ترجع `-ENOSYS` دائماً بغض النظر عن أي شيء في الـ Device Tree.

التحقق من الـ config:

```bash
# على الجهاز
zcat /proc/config.gz | grep OF_GPIO
# النتيجة: CONFIG_OF_GPIO is not set  <-- المشكلة هنا
```

#### الحل

```bash
# في menuconfig
make menuconfig
# Device Drivers → GPIO Support → GPIO lib → OF GPIO

# أو مباشرة في .config
echo "CONFIG_OF_GPIO=y" >> arch/arm64/configs/rk3562_gateway_defconfig

# إعادة البناء
make -j$(nproc) Image dtbs modules
```

في الـ DT يجب التأكد من وجود الـ property بالاسم الصحيح:

```c
/* في driver */
gpio_num = of_get_named_gpio(np, "relay-gpios", 0);
```

```dts
/* في .dts */
my-relay-controller {
    compatible = "vendor,relay-ctrl";
    relay-gpios = <&gpio2 5 GPIO_ACTIVE_HIGH>;
};
```

#### الدرس المستفاد
**الـ** `of_gpio.h` يُعرِّف نسختين من `of_get_named_gpio` — واحدة حقيقية وأخرى stub تُرجع `-ENOSYS`. النسخة المُستخدمة تعتمد فقط على `CONFIG_OF_GPIO` وقت الـ compile. دائماً تحقق من الـ kernel config قبل افتراض أن الـ DT هو المشكلة.

---

### السيناريو 2: Android TV Box على Allwinner H616 — HDMI CEC GPIO يعمل أحياناً فقط

#### العنوان
GPIO الخاص بـ HDMI CEC يُحضر قيماً غير صحيحة بسبب خطأ في `index` بـ `of_get_named_gpio`

#### السياق
منتج **Android TV box** مبني على **Allwinner H616**.
الـ HDMI CEC يحتاج GPIO للتحكم، والـ DT يُعرِّف قائمة تحتوي على GPIO واحد فقط.
المطور الجديد يُعدِّل الـ driver ليضيف دعم HDMI second port مستقبلاً ويغيِّر بعض القيم.

#### المشكلة
CEC يتوقف عن العمل بشكل عشوائي. أحياناً يعمل بعد reboot وأحياناً لا.

#### التحليل
`of_get_named_gpio` signature:

```c
extern int of_get_named_gpio(const struct device_node *np,
                             const char *list_name, int index);
```

**الـ** `index` يُحدد أي GPIO تأخذ من القائمة (zero-based).

المطور كتب:

```c
/* خطأ: index=1 لكن القائمة تحتوي فقط GPIO واحد */
int cec_gpio = of_get_named_gpio(np, "hdmi-cec-gpios", 1);
```

لكن الـ DT:

```dts
hdmi_cec: cec-controller {
    compatible = "allwinner,h616-cec";
    hdmi-cec-gpios = <&pio 7 4 GPIO_ACTIVE_LOW>;  /* GPIO واحد فقط */
};
```

عندما يكون الـ `index` خارج نطاق القائمة، `of_get_named_gpio` تستدعي داخلياً `of_get_named_gpio_flags` التي تُرجع `-ENOENT` أو قيمة غير صحيحة، مما يؤدي لاستخدام رقم GPIO عشوائي أو خاطئ.

التشخيص:

```bash
# على الجهاز
cat /sys/kernel/debug/gpio | grep cec
# لا يظهر شيء — دليل على فشل التسجيل

dmesg | grep -i cec
# [ 3.421] cec-controller: failed to get cec gpio: -22
```

#### الحل

```c
/* الصحيح: index=0 للـ GPIO الأول والوحيد */
int cec_gpio = of_get_named_gpio(np, "hdmi-cec-gpios", 0);

if (!gpio_is_valid(cec_gpio)) {
    dev_err(dev, "failed to get cec gpio: %d\n", cec_gpio);
    return cec_gpio;
}
```

للتوسع المستقبلي، يجب تعريف قائمة في الـ DT:

```dts
/* دعم منفذين */
hdmi-cec-gpios = <&pio 7 4 GPIO_ACTIVE_LOW>,
                 <&pio 7 5 GPIO_ACTIVE_LOW>;
```

#### الدرس المستفاد
**الـ** `index` في `of_get_named_gpio` هو zero-based ويجب أن يتطابق مع عدد العناصر في الـ DT property. خطأ بسيط في الـ index يُعطي GPIO رقم خاطئ أو error، والـ GPIO الخاطئ قد يعمل أحياناً بالصدفة إذا تطابق مع GPIO آخر نشط.

---

### السيناريو 3: IoT Sensor Node على STM32MP1 — Driver لا يُترجَم لـ non-OF platform

#### العنوان
Driver مكتوب لـ OF يفشل عند نقله لـ STM32MP1 bare-metal build بدون Device Tree

#### السياق
فريق يطور **IoT sensor node** على **STM32MP157** يستخدمه في بيئتين:
1. Linux مع Device Tree (بيئة التطوير)
2. بيئة bare-metal مخصصة بدون OF (الإنتاج المبسَّط)

الـ driver يستخدم `of_get_named_gpio` مباشرة بدون أي fallback.

#### المشكلة
عند build الـ production kernel بدون `CONFIG_OF_GPIO`، الـ driver يُترجَم لكنه لا يعمل — كل GPIO calls تُرجع `-ENOSYS` ولا يوجد أي error واضح في الـ log.

#### التحليل
الملف `of_gpio.h` يوفر الـ stub تلقائياً:

```c
#else /* CONFIG_OF_GPIO */

#include <linux/errno.h>

/* Drivers may not strictly depend on the GPIO support, so let them link. */
static inline int of_get_named_gpio(const struct device_node *np,
                                   const char *propname, int index)
{
    return -ENOSYS;  /* stub يمنع linker error لكن لا يعمل */
}

#endif /* CONFIG_OF_GPIO */
```

التعليق في الكود `"Drivers may not strictly depend on the GPIO support, so let them link."` يشرح الهدف: السماح بالـ compilation بدون OF دون كسر الـ linker. لكن الـ driver لم يتعامل مع `-ENOSYS` كـ non-fatal error.

```c
/* driver الأصلي — لا يتعامل مع ENOSYS */
static int sensor_probe(struct platform_device *pdev)
{
    int gpio = of_get_named_gpio(pdev->dev.of_node, "sensor-gpios", 0);

    /* لا يوجد فحص لـ ENOSYS */
    if (gpio < 0) {
        dev_err(&pdev->dev, "gpio error\n");
        return gpio;  /* يُرجع -ENOSYS ويفشل probe */
    }
    ...
}
```

#### الحل

```c
static int sensor_probe(struct platform_device *pdev)
{
    int gpio;

#ifdef CONFIG_OF_GPIO
    /* مسار OF: استخدم DT */
    if (pdev->dev.of_node) {
        gpio = of_get_named_gpio(pdev->dev.of_node, "sensor-gpios", 0);
        if (!gpio_is_valid(gpio)) {
            dev_err(&pdev->dev, "invalid gpio from DT: %d\n", gpio);
            return -EINVAL;
        }
    }
#else
    /* مسار non-OF: استخدم platform_data أو hardcoded للـ production */
    struct sensor_platform_data *pdata = dev_get_platdata(&pdev->dev);
    if (!pdata) {
        dev_err(&pdev->dev, "no platform data, no OF — cannot get gpio\n");
        return -ENODEV;
    }
    gpio = pdata->gpio_num;
#endif
    ...
}
```

#### الدرس المستفاد
**الـ** stub في `of_gpio.h` مُصمَّم لمنع linker errors وليس لإسقاط الـ GPIO silently. Driver صحيح يجب أن يفحص `-ENOSYS` بشكل صريح أو يستخدم `#ifdef CONFIG_OF_GPIO` لتوفير مسار بديل على platforms بدون OF.

---

### السيناريو 4: Automotive ECU على i.MX8MP — تعارض أسماء GPIO properties في DT

#### العنوان
**الـ** `of_get_named_gpio` يُرجع `-EINVAL` على ECU مبني على i.MX8MP بسبب اسم property خاطئ

#### السياق
شركة automotive تبني **ECU (Engine Control Unit)** على **NXP i.MX8M Plus**.
الـ ECU يتحكم في عدة actuators عبر GPIO.
مهندس جديد يُعدِّل الـ DT ويُغيِّر أسماء الـ properties لتتوافق مع standard معين.

#### المشكلة
بعد تعديل الـ DT، بعض الـ actuators توقفت عن العمل. الـ dmesg يُظهر:

```
[  4.123] imx8-ecu: gpio request failed: -22
[  4.124] imx8-ecu: probe failed
```

#### التحليل
`of_get_named_gpio` تبحث عن property باسم محدد:

```c
extern int of_get_named_gpio(const struct device_node *np,
                             const char *list_name,  /* اسم الـ property */
                             int index);
```

الـ driver يستدعي:

```c
/* driver يبحث عن "actuator-gpios" */
int act_gpio = of_get_named_gpio(np, "actuator-gpios", 0);
```

لكن المهندس غيَّر اسم الـ property في الـ DT:

```dts
/* قبل التعديل — يعمل */
ecu-actuator {
    compatible = "vendor,imx8-actuator";
    actuator-gpios = <&gpio4 12 GPIO_ACTIVE_HIGH>;
};

/* بعد التعديل — لا يعمل */
ecu-actuator {
    compatible = "vendor,imx8-actuator";
    output-gpios = <&gpio4 12 GPIO_ACTIVE_HIGH>;  /* اسم مختلف! */
};
```

عندما لا تجد `of_get_named_gpio` الـ property بالاسم المطلوب، تُرجع `-ENOENT` أو `-EINVAL`. الـ `-22` في dmesg هو `EINVAL`.

أداة التشخيص:

```bash
# فحص الـ DT على الجهاز
cat /proc/device-tree/ecu-actuator/output-gpios | xxd
# تجد الـ property لكن بالاسم الخاطئ

# أو استخدام dtc
dtc -I fs /proc/device-tree 2>/dev/null | grep -A5 "ecu-actuator"
```

#### الحل

خياران:

**الخيار 1**: إعادة الاسم الأصلي في الـ DT:

```dts
ecu-actuator {
    compatible = "vendor,imx8-actuator";
    actuator-gpios = <&gpio4 12 GPIO_ACTIVE_HIGH>;
};
```

**الخيار 2**: تعديل الـ driver ليقبل كلا الاسمين (للتوافقية):

```c
/* محاولة الاسم الجديد أولاً، ثم القديم */
int act_gpio = of_get_named_gpio(np, "output-gpios", 0);
if (!gpio_is_valid(act_gpio)) {
    /* fallback للاسم القديم */
    act_gpio = of_get_named_gpio(np, "actuator-gpios", 0);
    if (!gpio_is_valid(act_gpio)) {
        dev_err(dev, "no valid gpio found: %d\n", act_gpio);
        return -EINVAL;
    }
    dev_warn(dev, "deprecated 'actuator-gpios', use 'output-gpios'\n");
}
```

#### الدرس المستفاد
**الـ** `list_name` في `of_get_named_gpio` يجب أن يتطابق حرفياً مع اسم الـ property في الـ DT. لا توجد أي magic lookup أو aliases. تغيير اسم الـ property في الـ DT دون تغيير الـ driver يكسر الـ GPIO lookup بصمت تقريباً. في بيئات automotive حيث الـ DT والـ driver يُطوَّران بشكل منفصل، يجب توثيق أسماء الـ properties وتثبيتها في binding documentation.

---

### السيناريو 5: Custom Board Bring-up على AM62x — تشخيص GPIO عبر فهم بنية الـ header

#### العنوان
مهندس bring-up يستخدم فهم `of_gpio.h` لتشخيص لماذا GPIO لـ SPI CS لا يظهر في sysfs على **TI AM62x**

#### السياق
فريق يقوم بـ **custom board bring-up** على **Texas Instruments AM62x** (BeaglePlay-class SoC).
اللوحة تحتوي على SPI flash خارجي، والـ chip select يُدار عبر GPIO.
المهندس يحاول التحقق من أن الـ GPIO يعمل في مرحلة مبكرة قبل كتابة الـ SPI driver الكامل.

#### المشكلة
`of_get_named_gpio` تُرجع قيمة سالبة في early bring-up. المهندس لا يعرف إن كانت المشكلة في الـ DT، الـ config، أم الـ hardware.

#### التحليل
المهندس يبدأ بقراءة `of_gpio.h` ليفهم الـ flow:

```c
/* of_gpio.h — الـ header بالكامل */
#ifdef CONFIG_OF_GPIO
    /* النسخة الحقيقية — تحتاج OF GPIO subsystem */
    extern int of_get_named_gpio(...);
#else
    /* stub — يُرجع -ENOSYS دائماً */
    static inline int of_get_named_gpio(...) { return -ENOSYS; }
#endif
```

الـ header يُضمِّن أيضاً:
- `<linux/gpio/driver.h>` — بنى الـ GPIO chip
- `<linux/gpio.h>` — الـ API الأساسي
- `<linux/of.h>` — `struct device_node` وtools الـ DT

منهجية التشخيص المنظَّمة:

```bash
# الخطوة 1: تحقق من CONFIG_OF_GPIO
zcat /proc/config.gz | grep "OF_GPIO\|GPIOLIB"
# يجب أن تكون: CONFIG_OF_GPIO=y و CONFIG_GPIOLIB=y

# الخطوة 2: تحقق من وجود الـ DT node
ls /proc/device-tree/ | grep spi
cat /proc/device-tree/spi@0/cs-gpios | xxd
# إذا لم يوجد الملف: الـ property مفقودة من الـ DT

# الخطوة 3: تحقق من الـ GPIO controller
cat /sys/kernel/debug/gpio
# ابحث عن gpio chip الخاص بـ AM62x

# الخطوة 4: اختبر الـ GPIO مباشرة
gpioget $(gpiodetect | grep am62 | head -1 | awk '{print $1}') 15
```

الـ DT الصحيح:

```dts
/* في am62x-custom-board.dts */
&spi0 {
    status = "okay";
    cs-gpios = <&main_gpio0 15 GPIO_ACTIVE_LOW>;

    flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-max-frequency = <25000000>;
    };
};
```

الـ driver test snippet:

```c
/* test probe للـ bring-up المبكر */
static int spi_cs_test_probe(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    int cs_gpio;
    int ret;

    cs_gpio = of_get_named_gpio(np, "cs-gpios", 0);

    /* تشخيص مفصَّل */
    if (cs_gpio == -ENOSYS) {
        dev_err(&pdev->dev, "CONFIG_OF_GPIO not enabled!\n");
        return -ENOSYS;
    }
    if (cs_gpio == -ENOENT) {
        dev_err(&pdev->dev, "cs-gpios property missing from DT\n");
        return -ENOENT;
    }
    if (!gpio_is_valid(cs_gpio)) {
        dev_err(&pdev->dev, "invalid gpio number: %d\n", cs_gpio);
        return -EINVAL;
    }

    ret = gpio_request(cs_gpio, "spi-cs-test");
    if (ret) {
        dev_err(&pdev->dev, "gpio_request failed: %d\n", ret);
        return ret;
    }

    /* toggle الـ CS لاختبار hardware */
    gpio_direction_output(cs_gpio, 1);
    mdelay(10);
    gpio_set_value(cs_gpio, 0);
    mdelay(10);
    gpio_set_value(cs_gpio, 1);

    dev_info(&pdev->dev, "SPI CS GPIO %d works OK\n", cs_gpio);
    gpio_free(cs_gpio);
    return 0;
}
```

#### الدرس المستفاد
**الـ** `of_gpio.h` رغم بساطته الشديدة (39 سطراً فقط) يُخفي منطقاً مهماً: تقسيم compile-time بين نسخة حقيقية ونسخة stub. فهم هذا التقسيم يُوجِّه المهندس مباشرة لأول ما يجب فحصه. في bring-up، قراءة الـ header قبل كتابة الكود توفر ساعات من التشخيص العشوائي. كما أن التعامل مع كل قيمة error مُحتملة (`-ENOSYS`، `-ENOENT`، `-EINVAL`) بشكل منفصل يُسرِّع تحديد جذر المشكلة.
## Phase 7: مصادر ومراجع

---

### مقدمة

الملف `include/linux/of_gpio.h` هو واجهة برمجية خفيفة تربط بين **Open Firmware / Device Tree** وإطار عمل **GPIOlib** في نواة Linux. يوفر دالة `of_get_named_gpio()` للحصول على رقم GPIO من خاصية Device Tree بالاسم. المراجع أدناه تغطي السياق التاريخي، الواجهة الحديثة البديلة، وتوثيق Device Tree bindings.

---

### مقالات LWN.net

| العنوان | الرابط | الأهمية |
|---------|--------|---------|
| **GPIO in the kernel: an introduction** | [lwn.net/Articles/532714](https://lwn.net/Articles/532714/) | مقدمة شاملة لـ GPIO API في النواة، يتناول `of_get_named_gpio` والواجهة التقليدية |
| **GPIO in the kernel: future directions** | [lwn.net/Articles/533632](https://lwn.net/Articles/533632/) | يشرح التحول من الواجهة القائمة على الأرقام (التي تعتمد عليها `of_get_named_gpio`) إلى descriptor-based API |
| **New descriptor-based GPIO interface** | [lwn.net/Articles/565662](https://lwn.net/Articles/565662/) | يعرض `gpiod_get()` كبديل حديث لـ `of_get_named_gpio()` |
| **gpiolib: introduce descriptor-based GPIO interface** | [lwn.net/Articles/528226](https://lwn.net/Articles/528226/) | الـ patch الأصلي الذي أدخل الواجهة الجديدة وجعل الواجهة القديمة مرشّحة للإهمال |
| **gpio: introduce descriptor-based interface** | [lwn.net/Articles/531848](https://lwn.net/Articles/531848/) | نقاش أعمق حول تصميم descriptor interface وعلاقتها بـ Device Tree |
| **Device trees II: The harder parts** | [lwn.net/Articles/573409](https://lwn.net/Articles/573409/) | يشرح كيف يُحدَّد GPIO في Device Tree، بما في ذلك `#gpio-cells` وتفسير قيم `gpios = <&gpio1 5 0>` |
| **Documentation: gpiolib: document new interface** | [lwn.net/Articles/574055](https://lwn.net/Articles/574055/) | توثيق رسمي مُقدَّم كـ patch للنواة يوضح الفرق بين الواجهتين |
| **gpio: Document GPIO hogging mechanism** | [lwn.net/Articles/641117](https://lwn.net/Articles/641117/) | يتناول آلية **GPIO hog** المرتبطة بـ Device Tree وكيف تعمل مع `gpio-controller` |

---

### توثيق النواة الرسمي

```
Documentation/driver-api/gpio/index.rst          ← نقطة البداية لكل وثائق GPIO
Documentation/driver-api/gpio/board.rst          ← GPIO Mappings عبر Device Tree / ACPI
Documentation/driver-api/gpio/consumer.rst       ← كيفية استخدام GPIO من driver (gpiod_get)
Documentation/driver-api/gpio/driver.rst         ← كتابة gpio-controller driver
Documentation/devicetree/bindings/gpio/gpio.txt  ← Device Tree bindings رسمية لـ GPIO
```

الروابط الإلكترونية:

- [GPIO Mappings — kernel.org](https://www.kernel.org/doc/html/latest/driver-api/gpio/board.html)
- [GPIO Descriptor Consumer Interface — kernel.org](https://docs.kernel.org/driver-api/gpio/consumer.html)
- [General Purpose Input/Output (GPIO) — kernel.org](https://docs.kernel.org/driver-api/gpio/index.html)
- [devicetree/bindings/gpio/gpio.txt — kernel.org](https://www.kernel.org/doc/Documentation/devicetree/bindings/gpio/gpio.txt)

---

### Commits ذات صلة في kernel git

يمكن البحث عن هذه الـ commits عبر [git.kernel.org](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git):

| الحدث | الوصف |
|-------|-------|
| إدخال `of_get_named_gpio` | Anton Vorontsov, MontaVista Software, 2008 — الـ commit الأصلي لـ `of_gpio.c` |
| إضافة `CONFIG_OF_GPIO` guard | تقسيم الملف إلى مسارَي compile-time لدعم أنظمة بدون Device Tree |
| إدخال `gpiod_get()` | Alexandre Courbot — بديل descriptor-based لـ `of_get_named_gpio` |
| إهمال `of_get_named_gpio` رسمياً | إضافة تعليق `DEPRECATED` في التوثيق (2024) |

للبحث المباشر:
```bash
git log --oneline --all -- drivers/gpio/gpiolib-of.c
git log --oneline --all -- include/linux/of_gpio.h
git log --grep="of_get_named_gpio" --oneline
```

---

### نقاشات Mailing List

- **linux-gpio@vger.kernel.org** — القائمة الرسمية لكل مناقشات GPIO في النواة
- [gpio: legacy: mark old interfaces as deprecated — patchew.org](https://patchew.org/linux/20240104182034.61892-1-brgl@bgdev.pl/) — الـ patch الذي أعلن رسمياً إهمال `of_get_named_gpio` وأخواتها
- [GPIO marking individual pins not available in device tree — linuxppc-dev](https://linuxppc-dev.ozlabs.narkive.com/qp4tnrzh/gpio-marking-individual-pins-not-available-in-device-tree) — نقاش تقني حول تمييز الـ pins في Device Tree

للاشتراك في الأرشيف:
```
https://lore.kernel.org/linux-gpio/
```

---

### kernelnewbies.org

| الرابط | المحتوى |
|--------|---------|
| [Using gpio from device tree on platform devices](https://lists.kernelnewbies.org/pipermail/kernelnewbies/2015-August/014911.html) | نقاش عملي حول استخدام GPIO من Device Tree في platform driver، يُوضح متى تُستخدم `of_get_named_gpio` vs `gpiod_get` |
| [devm_gpiod_get usage to get the gpio num](http://lists.kernelnewbies.org/pipermail/kernelnewbies/2016-August/016707.html) | سؤال وجواب حول `devm_gpiod_get` كبديل managed لـ `of_get_named_gpio` |
| [Linux_2_6_25 — kernelnewbies.org](https://kernelnewbies.org/Linux_2_6_25) | الإصدار الذي شهد تحسينات مبكرة لـ `gpiolib` |

---

### elinux.org

| الرابط | المحتوى |
|--------|---------|
| [Device Tree Reference — elinux.org](https://elinux.org/Device_Tree_Reference) | مرجع شامل لبنية Device Tree، يشمل `#gpio-cells` وخصائص `gpios` |
| [Device Tree Mysteries — elinux.org](https://elinux.org/Device_Tree_Mysteries) | يشرح `#gpio-cells` وكيف تُفسَّر قيم `gpios = <&controller pin flags>` |
| [EBC Device Trees — elinux.org](https://elinux.org/EBC_Device_Trees) | أمثلة عملية على BeagleBone لتحديد GPIO في Device Tree overlays |

---

### كتب مُوصى بها

#### Linux Device Drivers (LDD3)
- **المؤلفون**: Jonathan Corbet, Alessandro Rubini, Greg Kroah-Hartman
- **الفصل ذو الصلة**: Chapter 1 (Device Driver Concepts) — الخلفية العامة
- **ملاحظة**: LDD3 يعود لعصر 2.6.10 ولا يتناول `of_gpio` مباشرةً، لكنه ضروري لفهم البنية العامة لـ drivers
- [متاح مجاناً على lwn.net/Kernel/LDD3](https://lwn.net/Kernel/LDD3/)

#### Linux Kernel Development — Robert Love
- **الطبعة الثالثة**، الفصل 17 (Device Model)
- يبني الأساس لفهم `struct device_node` و Device Model الذي يعتمد عليه `of_gpio`

#### Embedded Linux Primer — Christopher Hallinan
- **الفصل 15**: Device Tree و Open Firmware
- يشرح بناء Device Tree nodes وخصائص `gpio-controller` و `#gpio-cells`
- ضروري لفهم كيف تُقرأ `gpios = <&gpio0 5 GPIO_ACTIVE_LOW>` بواسطة `of_get_named_gpio`

#### Linux Device Driver Development — John Madieu (Packt, 2nd Ed. 2022)
- **الفصل 15**: GPIO Controller and Consumer Drivers
- يغطي `of_get_named_gpio` و `gpiod_get` بأمثلة حديثة

---

### مصادر إضافية

- [Linux device driver development: GPIO interface and device tree — embedded.com](https://www.embedded.com/linux-device-driver-development-the-gpio-interface-and-device-tree/)
- [GPIO device tree configuration — STM32MPU wiki](https://wiki.st.com/stm32mpu/wiki/GPIO_device_tree_configuration)
- [Stop using /sys/class/gpio — The Good Penguin](https://www.thegoodpenguin.co.uk/blog/stop-using-sys-class-gpio-its-deprecated/)
- [GPIO Programming: Exploring libgpiod — ICS](https://www.ics.com/blog/gpio-programming-exploring-libgpiod-library)

---

### مصطلحات البحث

للبحث عن معلومات إضافية، استخدم هذه الـ search terms:

```
of_get_named_gpio deprecated alternative
of_gpio.c kernel source
gpiod_get device tree consumer
linux gpio descriptor interface
#gpio-cells device tree binding
gpios property device tree
CONFIG_OF_GPIO kernel config
gpio-controller device tree node
of_get_gpio_flags kernel
gpiolib of gpio open firmware
linux-gpio mailing list vger
```

```bash
# البحث في مصدر النواة
grep -r "of_get_named_gpio" drivers/ --include="*.c" | head -20
grep -r "gpiod_get" drivers/ --include="*.c" | head -20
```
## Phase 8: Writing simple module

### الهدف

**الـ** `of_get_named_gpio` هي الدالة الوحيدة المُصدَّرة في `of_gpio.h`، وهي المدخل الرئيسي لأي driver يريد الحصول على رقم GPIO من Device Tree. سنضع عليها **kprobe** لنطبع اسم الـ node ومعامل `list_name` في كل مرة يُستدعى فيها.

---

### الكود الكامل

```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kprobe_of_gpio.c
 * Hook of_get_named_gpio() to trace DT GPIO lookups.
 */

#include <linux/kernel.h>   /* pr_info, pr_err */
#include <linux/module.h>   /* MODULE_* macros, module_init/exit */
#include <linux/kprobes.h>  /* kprobe, register_kprobe, ... */
#include <linux/of.h>       /* struct device_node, of_node_full_name() */

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Kernel Student");
MODULE_DESCRIPTION("Trace of_get_named_gpio() calls via kprobe");

/* ---------------------------------------------------------------
 * kprobe entry handler — fires BEFORE of_get_named_gpio executes
 * Prototype:
 *   int of_get_named_gpio(const struct device_node *np,
 *                         const char *list_name, int index);
 * ---------------------------------------------------------------*/
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    /*
     * Argument passing follows the platform ABI:
     *   arg0 (regs->di on x86-64 / regs->regs[0] on arm64) = np
     *   arg1                                                  = list_name
     *   arg2                                                  = index
     *
     * kprobes provides the helper regs_get_kernel_argument()
     * which is portable across architectures.
     */
    const struct device_node *np =
        (const struct device_node *)regs_get_kernel_argument(regs, 0);
    const char *list_name =
        (const char *)regs_get_kernel_argument(regs, 1);
    int index = (int)(long)regs_get_kernel_argument(regs, 2);

    /* of_node_full_name() returns the full DT path — safe even if np is NULL */
    pr_info("[kprobe_of_gpio] node=%-32s  prop=%-24s  idx=%d\n",
            np ? of_node_full_name(np) : "<null>",
            list_name ? list_name : "<null>",
            index);

    return 0; /* 0 = continue normal execution */
}

/* ---------------------------------------------------------------
 * kprobe descriptor — one per hooked symbol
 * ---------------------------------------------------------------*/
static struct kprobe kp = {
    .symbol_name = "of_get_named_gpio", /* kernel symbol to hook */
    .pre_handler  = handler_pre,        /* called before the function */
};

/* ---------------------------------------------------------------
 * module_init — register the kprobe
 * ---------------------------------------------------------------*/
static int __init kprobe_of_gpio_init(void)
{
    int ret;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("[kprobe_of_gpio] register_kprobe failed: %d\n", ret);
        return ret;
    }

    pr_info("[kprobe_of_gpio] planted at %pS\n", kp.addr);
    return 0;
}

/* ---------------------------------------------------------------
 * module_exit — unregister the kprobe
 * ---------------------------------------------------------------*/
static void __exit kprobe_of_gpio_exit(void)
{
    unregister_kprobe(&kp);
    pr_info("[kprobe_of_gpio] removed\n");
}

module_init(kprobe_of_gpio_init);
module_exit(kprobe_of_gpio_exit);
```

---

### شرح كل قسم

#### الـ includes

| Header | السبب |
|---|---|
| `<linux/kprobes.h>` | يحتوي `struct kprobe`، `register_kprobe`، `regs_get_kernel_argument` |
| `<linux/of.h>` | يعرّف `struct device_node` ودالة `of_node_full_name()` |
| `<linux/module.h>` | ماكروهات `MODULE_LICENSE` و`module_init`/`module_exit` |
| `<linux/kernel.h>` | `pr_info` / `pr_err` |

#### الـ `handler_pre`

**الـ** `pre_handler` يُستدعى قبل تنفيذ الدالة المُستهدفة مباشرةً، فتكون معاملاتها لا تزال سليمة في السجلات. نستخدم `regs_get_kernel_argument()` بدلاً من الوصول المباشر لـ`regs->di` لأنه portable بين x86-64 وARM64 وغيرها.

نطبع:
- المسار الكامل لـ device node (`of_node_full_name`) — يُظهر مكان الـ node في شجرة DT.
- اسم الـ property (`list_name`) — مثلاً `"led-gpios"` أو `"reset-gpios"`.
- الـ`index` — لأن property واحدة قد تحتوي أكثر من GPIO.

#### الـ `struct kprobe`

**الـ** `symbol_name` يُحدد الدالة المُستهدفة باسمها في جدول رموز الكيرنل (`/proc/kallsyms`). الكيرنل يحوّل الاسم إلى عنوان أثناء `register_kprobe`.

#### الـ `module_init`

`register_kprobe` تضع نقطة اختراق (breakpoint) على أول تعليمة في `of_get_named_gpio`. إذا فشلت (مثلاً لأن الرمز غير موجود أو محمي)، نُعيد الخطأ ولا يُحمَّل الموديول.

#### الـ `module_exit`

`unregister_kprobe` تُزيل الـ breakpoint وتنتظر انتهاء أي استدعاء جارٍ قبل الإزالة — ضروري لتجنب الـ use-after-free لو كانت الدالة تُستدعى أثناء إلغاء التسجيل.

---

### Makefile بسيط

```makefile
obj-m += kprobe_of_gpio.o

KDIR ?= /lib/modules/$(shell uname -r)/build

all:
	make -C $(KDIR) M=$(PWD) modules

clean:
	make -C $(KDIR) M=$(PWD) clean
```

---

### تشغيل وملاحظة المخرجات

```bash
# بناء الموديول
make

# تحميله
sudo insmod kprobe_of_gpio.ko

# مشاهدة السجل في الوقت الفعلي
sudo dmesg -w | grep kprobe_of_gpio

# مثال على مخرجات حقيقية أثناء تشغيل GPIO driver:
# [kprobe_of_gpio] node=/soc/gpio@fe200000          prop=led-gpios           idx=0
# [kprobe_of_gpio] node=/soc/gpio@fe200000          prop=reset-gpios         idx=0

# إزالة الموديول
sudo rmmod kprobe_of_gpio
```

---

### لماذا `of_get_named_gpio` تحديداً؟

- هي **نقطة الدخول الوحيدة** في `of_gpio.h`، كل driver يحتاج GPIO من DT يمر بها.
- آمنة للـ kprobe لأنها ليست داخل مسار الـ kprobe نفسه (لا recursion).
- تعطي معلومات حقيقية وذات قيمة: من يطلب GPIO؟ وأي property؟
